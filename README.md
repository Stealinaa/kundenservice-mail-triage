# KI-gestützte E-Mail-Triage im Kundenservice

Eingehende Kundenanfragen werden automatisch nach **Kategorie**
(Lieferung · Rechnung · Reklamation · Sonstiges) und **Dringlichkeit**
vorsortiert. Umgesetzt mit der Claude API und Make, ohne eine Zeile Code.

> Eigenprojekt. Die 40 Testanfragen sind erfunden, aber realistisch
> nachgebaut und mit einem KI-Assistenten erzeugt. Es werden keine echten
> Kundendaten verarbeitet.

---

## Das Problem

Kundenanfragen laufen in einem gemeinsamen Postfach auf. Jede muss gelesen
und dem richtigen Bereich zugeordnet werden, bevor sie überhaupt bearbeitet
wird.

Ich habe gemessen, wie lange das dauert: **27,6 Sekunden pro E-Mail**
(20 E-Mails, mit der Stoppuhr, vor jedem Automatisierungsversuch).
Hochgerechnet sind das 46 Minuten je 100 E-Mails.

Wichtiger als die Zeit ist aber etwas anderes: **nichts hat eine Priorität.**
Eine Reklamation über ein überhitzendes Ladegerät liegt in derselben Liste
wie die Frage nach den Öffnungszeiten.

## Die Lösung

```
Google Sheets (eingehende E-Mails)
        ↓
Claude API  —  Kategorie · Dringlichkeit · Konfidenz
        ↓
Google Sheets (Ergebnis zurück in die Zeile)
```

Zwei Entwurfsentscheidungen tragen das Ganze:

**Das System beantwortet nichts, es sortiert nur vor.** Eine falsche
Einsortierung ist mit einem Klick korrigiert. Eine falsch verschickte
Antwort nicht.

**Das System darf "ich weiß es nicht" sagen.** Jede Antwort enthält einen
Konfidenzwert. Unsichere Fälle können an einen Menschen gehen, statt
automatisch abgelegt zu werden. Wie gut das funktioniert, steht weiter
unten — die Antwort war überraschend.

## Ergebnisse

Gemessen gegen 40 Anfragen, die ich vorher selbst von Hand gelabelt habe.

| | Wert |
|---|---|
| Kategorie korrekt | **92,5 %** (37 von 40) |
| Dringlichkeit korrekt | **95,0 %** (38 von 40) |
| Manuelle Sortierzeit vorher | 27,6 s je E-Mail |
| Kosten | ca. 1,10 USD je 1000 E-Mails |
| Laufzeit | 1 Minute für 40 E-Mails |

Die gesamte Entwicklung — alle Experimente, vier Prompt-Versionen,
Fehlersuche und fünf Durchläufe — hat 0,31 USD an API-Kosten verursacht.

### Wie viel Arbeit beim Menschen bleibt

Das hängt davon ab, ab welcher Konfidenz automatisch abgelegt wird:

| Schwelle | E-Mails an den Menschen | Fehler, die durchgehen |
|---|---|---|
| 0,70 | 1 von 40 | 3 |
| 0,80 | 3 von 40 | 3 |
| 0,86 | 10 von 40 | 0 |
| 0,90 | 10 von 40 | 0 |
| 0,96 | 37 von 40 | 0 |

Oberhalb von 0,86 gewinnt man nichts mehr: alle Fehler sind gefangen, aber
die Arbeit für den Menschen wächst weiter.

**Die Wahl der Schwelle ist keine technische Einstellung, sondern eine
Entscheidung über die Kosten eines Fehlers.** Drei falsch abgelegte E-Mails
gegen ein Drittel des Postfachs zurück in Handarbeit — das muss der
Fachbereich entscheiden, nicht die Technik.

## Drei Erkenntnisse

### 1. Der Konfidenzwert sagt nicht, ob die Antwort stimmt

Bei einer Schwelle von 0,7 ging genau **eine** E-Mail an den Menschen — und
die war korrekt klassifiziert. Alle drei echten Fehler kamen mit einer
Konfidenz von 0,75 bis 0,85, also deutlich über der Schwelle.

Das Modell ist genauso überzeugt, wenn es sich irrt. Der Konfidenzwert ist
eine Selbsteinschätzung, keine Wahrscheinlichkeit.

### 2. "Erfolgreich" heißt nicht "richtig"

Ein Durchlauf meldete 81 von 81 Operationen erfolgreich — aber nur die
Hälfte der Zeilen war gefüllt. Die Ursache: das Modell verpackte sein JSON
bei etwa jeder zweiten Antwort in einen Markdown-Codeblock. Make konnte das
nicht lesen und schrieb leere Werte in die Tabelle. Kein Fehler, keine
rote Meldung — nur leere Zellen.

Gefunden habe ich es nicht im Log, sondern beim Abgleich: ich habe gezählt,
wie viele Zeilen gefüllt sind, und die Zahl stimmte nicht. Behoben in
Prompt v3 durch ein ausdrückliches Verbot von Codeblöcken.

**Merksatz: nicht den Status prüfen, sondern das Ergebnis.**

### 3. Das Modell kann nicht konsistenter sein als der Goldstandard

Alle drei Kategoriefehler zeigten in dieselbe Richtung: Reklamationen
wurden als Lieferung oder Rechnung eingeordnet. Ich habe die betreffende
Regel deshalb geschärft (Version v4).

Ergebnis: die drei Fehler waren weg — dafür entstanden vier neue. Das Modell
wandte die neue Regel sauber an, aber mein eigener Goldstandard zog die
Grenze zwischen Rechnung und Reklamation uneinheitlich. Zwei fast
identische Fälle (unberechtigte Mahnung, Mail 13 und Mail 20) hatte ich
unterschiedlich gelabelt.

v4 wurde verworfen und dokumentiert. Die Messung hat hier nicht das Modell
bewertet, sondern den Maßstab.

## Prompt-Versionen

| Version | Änderung | Anlass | Kategorie korrekt |
|---|---|---|---|
| v1 | Vier Kategorien, Dringlichkeit, JSON | erster Entwurf | Grundlinie |
| v2 | Abgrenzungsregeln, Konfidenzwert | v1 ordnete Erstattungsforderungen dem Versand zu | — |
| v3 | Verbot von Markdown-Codeblöcken, Ausgabeblock ans Ende | 20 von 40 Zeilen blieben leer | **92,5 %** |
| v4 | Regel 1 als "Unternehmensfehler" formuliert | drei Fehler in dieselbe Richtung | 87,5 % — **verworfen** |

Die Volltexte stehen in [`prompt.md`](prompt.md). Alte Versionen wurden nie
überschrieben: jede Version ist ein Beleg dafür, was tatsächlich gelaufen
ist, und nicht meine heutige Meinung darüber.

## Grenzen

- **40 E-Mails sind wenig.** Genug, um Fehlertypen zu erkennen, zu wenig für
  belastbare Prozentwerte. Ein Fehler mehr oder weniger verschiebt das
  Ergebnis um 2,5 Prozentpunkte.
- **Kein Hold-out-Datensatz.** Ich habe den Prompt an denselben 40 E-Mails
  verbessert, an denen ich ihn messe. Das grenzt an Overfitting.
- **Der Goldstandard ist an einer Stelle widersprüchlich** (siehe Erkenntnis 3).
- **Gemessen wurde nur das Sortieren**, nicht die inhaltliche Bearbeitung
  der Anfrage. Die dauert unverändert gleich lang.
- **Anhänge werden nicht ausgewertet**, nur Betreff und Text.
- **Die Kostenangabe betrifft nur die API.** Der kostenlose Make-Tarif deckt
  etwa 330 E-Mails pro Monat ab (drei Operationen je E-Mail); darüber
  hinaus fallen Plattformkosten an.

## Nächste Schritte

1. Goldstandard überarbeiten: die Grenze zwischen Rechnung und Reklamation
   eindeutig definieren und alle 40 E-Mails danach neu labeln.
2. Zehn E-Mails als Hold-out zurücklegen und den Prompt nie an ihnen ändern.
3. Anbindung an ein echtes Postfach mit Vergabe von Gmail-Labels.
4. Statt der Konfidenzschwelle nach Fehlerkosten aussteuern: alles, was das
   Modell als dringende Reklamation einstuft, geht grundsätzlich an einen
   Menschen.
5. Strukturierte Ausgabe (Output Format) statt einer Formatvorgabe im
   Prompt — eine Vorgabe ist eine Bitte, ein Schema ist eine Einschränkung.

## Aufbau

| Pfad | Inhalt |
|---|---|
| `data/emails.csv` | 40 Testanfragen |
| `data/goldstandard.csv` | meine Labels, die Antworten des Modells, Konfidenz |
| `prompt.md` | alle Prompt-Versionen mit Begründung |
| `results/auswertung.md` | Trefferquoten, Schwellenwert-Tabelle, Fehleranalyse |
| `screenshots/` | Make-Szenario, Playground, Auswertung |

## Werkzeuge

Claude API (`claude-sonnet-5`) · Make · Google Sheets · Anthropic Console
Playground · GitHub

Der API-Schlüssel liegt in den Credentials von Make und nicht im Szenario.
Jede Entscheidung des Systems wird protokolliert — ohne Protokoll lässt sich
weder die Qualität messen noch ein Einzelfall nachvollziehen.

## Rolle

Konzeption, Kategorienschema, Abgrenzungsregeln, Goldstandard-Labeling,
Prompt-Entwicklung über vier Versionen, Aufbau des Make-Szenarios,
Fehlersuche und Auswertung: alles selbst. Als Werkzeug habe ich dabei einen
KI-Assistenten genutzt, auch für die sprachliche Fassung dieses Textes.
Die inhaltlichen Entscheidungen — Kategorien, Schwellenwert, was als richtig
gilt — habe ich getroffen und kann sie begründen.
