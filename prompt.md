# Prompt-Versionen

Der Prompt wurde in vier Schritten entwickelt. Jede Version wurde gegen
denselben Goldstandard aus 40 E-Mails gemessen, bevor die nächste entstand.

**Alte Versionen wurden nie überschrieben.** Eine Version ist ein Beleg
dafür, was tatsächlich gelaufen ist — nicht meine heutige Meinung darüber.
Deshalb steht hier auch v4, obwohl sie verworfen wurde.

| Version | Änderung | Anlass | Kategorie korrekt |
|---|---|---|---|
| v1 | Vier Kategorien, Dringlichkeit, JSON | erster Entwurf | Grundlinie |
| v2 | Abgrenzungsregeln, Konfidenzwert | v1 ordnete Erstattungsforderungen dem Versand zu | — |
| v3 | Verbot von Markdown-Codeblöcken, Ausgabeblock ans Ende | 20 von 40 Zeilen blieben leer | **92,5 %** |
| v4 | Regel 1 als "Unternehmensfehler" formuliert | drei Fehler in dieselbe Richtung | 87,5 % — verworfen |

Getestet wurde durchgehend mit `claude-sonnet-5`, Effort Level `low`,
`max_tokens` 1000. Das Modell wurde zwischen den Versionen nicht gewechselt,
damit der Unterschied allein am Prompt liegt.

---

## v1 — erster Entwurf

Absichtlich knapp gehalten: vier Kategorienamen, keine Abgrenzungsregeln,
keine Definition von "dringend". Diese Version ist die Vergleichsgrundlage.

```text
Du sortierst eingehende Kunden-E-Mails eines Online-Händlers vor.

Ordne jede E-Mail einer dieser Kategorien zu:
Lieferung, Rechnung, Reklamation, Sonstiges.

Gib außerdem an, ob die E-Mail dringend ist.

Antworte nur mit JSON:
{"kategorie": "...", "dringend": "ja" oder "nein"}
```

**Beobachtung:** Bei den eindeutigen E-Mails lag v1 bereits richtig. Er
scheiterte an den Grenzfällen — dort, wo die Zuordnung nicht aus dem Text
folgt, sondern aus einer betrieblichen Entscheidung. Beispiel: Express-Versand
bezahlt, Paket kam zu spät, der Kunde fordert die Versandkosten zurück.
v1 sagte "Lieferung", mein Goldstandard sagt "Reklamation".

**Daraus die Einsicht, die den ganzen Prompt trägt:** Ein Prompt macht das
Modell nicht klüger. Er teilt ihm Entscheidungen mit, die es aus den Daten
gar nicht ableiten kann.

---

## v2 — Abgrenzungsregeln und Konfidenz

Die Regeln stammen aus meinen eigenen Notizen beim Labeln: überall dort, wo
ich selbst zögern musste, habe ich festgehalten, wie ich entschieden habe
und warum. Diese Notizen wurden fast wörtlich zu den Abgrenzungsregeln.

Neu außerdem: ein Konfidenzwert, damit unsichere Fälle später an einen
Menschen ausgesteuert werden können.

```text
Du sortierst eingehende Kunden-E-Mails eines Online-Händlers vor.
Du beantwortest die Anfragen nicht — du ordnest sie nur ein.

KATEGORIEN — wähle genau eine:

Lieferung   — Versandstatus, Verzögerung, Sendungsverfolgung, Lieferadresse,
              Liefertermine, allgemeine Fragen zum Versand.
Rechnung    — Rechnungen, Zahlungen, Abbuchungen, Mahnungen, Gutschriften,
              Rechnungs- und Steuerdaten, Zahlungsarten.
Reklamation — Beanstandungen zur Ware oder zu einer bezahlten Leistung:
              defekt, beschädigt, falsch, unvollständig, Gewährleistung,
              Reparatur.
Sonstiges   — alles andere: Produktberatung, Bewerbungen, Kooperationen,
              Datenschutz, Newsletter, Werbung, allgemeine Fragen.

ABGRENZUNGSREGELN — sie haben Vorrang vor der Kategorienliste:

1. Beanstandung mit Geldforderung → Reklamation.
   Reine Auskunft zu einer Rechnung oder Zahlung → Rechnung.
   Beispiel: Express-Versand bezahlt, Paket kam zu spät, Kunde fordert die
   Versandkosten zurück → Reklamation, nicht Lieferung.

2. Ware beschädigt, defekt, falsch oder unvollständig geliefert →
   Reklamation, auch wenn der Schaden beim Transport entstanden ist.

3. Widerruf ohne Mangel → Sonstiges. Der Widerruf ist ein gesetzliches
   Rücktrittsrecht, die Ware kann einwandfrei sein.
   Rückgabe wegen eines Mangels → Reklamation.

4. Hinweis auf eine mögliche Gefahr für Personen (Überhitzung, Brandgeruch,
   Stromschlag) → immer Reklamation.

DRINGLICHKEIT — "ja" nur, wenn mindestens eines zutrifft:

- eine gesetzliche Frist läuft (Datenschutzanfrage nach DSGVO, Widerruf,
  Mahnung)
- Gefahr für Personen
- Geld ist bereits falsch abgeflossen (Doppelabbuchung, zu hoher Betrag)
- der Kunde schreibt nachweislich zum wiederholten Mal, oder es werden
  rechtliche Schritte angedroht
- eine Änderung ist nur bis zu einem bestimmten Zeitpunkt möglich
  (z. B. Adressänderung vor dem Versand)

Sonst "nein".

Ein verärgerter oder fordernder Ton allein macht eine E-Mail nicht dringend.
Werbung und Bewerbungen sind nie dringend.

AUSGABE — antworte ausschließlich mit diesem JSON, ohne Text davor oder danach:

{
  "kategorie": "Lieferung" | "Rechnung" | "Reklamation" | "Sonstiges",
  "dringend": "ja" | "nein",
  "konfidenz": <Zahl zwischen 0 und 1, mit Punkt als Dezimaltrennzeichen>
}

Setze "konfidenz" unter 0.7, wenn die E-Mail zu zwei Kategorien passt oder
zu wenig Information enthält. Rate nicht — eine niedrige Konfidenz ist eine
gültige und erwünschte Antwort.
```

Zwei Formulierungen darin sind bewusst als **Verbote** geschrieben:

- *Ein verärgerter Ton allein macht eine E-Mail nicht dringend.*
- *Werbung und Bewerbungen sind nie dringend.*

Verbote wirken hier stärker als Gebote. Ohne sie stuft das Modell alles als
dringend ein, was scharf formuliert ist — genau der Fehler, den ich beim
ersten eigenen Durchgang selbst gemacht habe.

**Problem in v2:** Bei ungefähr jeder zweiten Antwort verpackte das Modell
sein JSON in einen Markdown-Codeblock. Make konnte das nicht lesen und
schrieb leere Werte in die Tabelle — ohne Fehlermeldung. Der Durchlauf
meldete 81 von 81 Operationen erfolgreich, aber nur 20 von 40 Zeilen waren
gefüllt.

---

## v3 — Formatvorgaben, aktuell im Einsatz

Zwei Änderungen gegenüber v2, beide am Ausgabeblock:

1. Der Block `AUSGABE` steht jetzt **am Ende** des Prompts. Anweisungen am
   Schluss werden zuverlässiger befolgt als solche, die mitten im Text
   stehen.
2. Die Formatvorgaben sind als ausdrückliche Verbotsliste formuliert.

Der übrige Text ist identisch mit v2. Hier nur der geänderte Schluss:

```text
KONFIDENZ

Setze "konfidenz" unter 0.7, wenn die E-Mail zu zwei Kategorien passt oder
zu wenig Information enthält. Rate nicht — eine niedrige Konfidenz ist eine
gültige und erwünschte Antwort.

AUSGABE

Antworte mit genau diesem JSON-Objekt und mit nichts anderem:

{
  "kategorie": "Lieferung" | "Rechnung" | "Reklamation" | "Sonstiges",
  "dringend": "ja" | "nein",
  "konfidenz": <Zahl zwischen 0 und 1, mit Punkt als Dezimaltrennzeichen>
}

Formatvorgaben, die ausnahmslos gelten:
- Keine Code-Blöcke, keine Markdown-Formatierung.
- Keine Backticks und kein "json" vor oder nach dem Objekt.
- Kein erklärender Text davor oder danach.
- Das erste Zeichen deiner Antwort ist {, das letzte ist }.
```

**Ergebnis:** 40 von 40 Zeilen gefüllt. Kategorie korrekt in 92,5 % der
Fälle, Dringlichkeit in 95 %.

**Anmerkung:** Eine Formatvorgabe im Prompt ist eine Bitte. Zuverlässiger
wäre eine strukturierte Ausgabe (Output Format) auf API-Ebene — dann ist die
Abweichung technisch unmöglich statt unerwünscht. Das steht auf der Liste
der nächsten Schritte.

---

## v4 — verworfen

Alle drei verbleibenden Kategoriefehler zeigten in dieselbe Richtung:
Reklamationen wurden als Lieferung oder Rechnung eingeordnet. Die
gemeinsame Ursache: Regel 1 knüpfte an eine Geldforderung an, obwohl das
eigentliche Unterscheidungsmerkmal ein anderes ist — ob der Kunde einen
**Fehler des Unternehmens** beanstandet.

v4 ist identisch mit v3, nur Regel 1 wurde ersetzt durch:

```text
1. Beanstandet der Kunde einen Fehler des Unternehmens — eine zugesagte,
   aber nicht erbrachte Leistung, eine falsche Abrechnung, eine verlorene
   Sendung, eine unberechtigte Mahnung — dann ist es eine Reklamation,
   unabhängig davon, ob Geld gefordert wird und welcher Abteilung das
   Thema zugehört.
   Stellt er dagegen nur eine Frage oder bittet um eine Handlung
   ("Rechnung zusenden", "Zahlungsart ändern"), ist es Rechnung
   beziehungsweise Lieferung.
```

**Ergebnis: 87,5 % statt 92,5 %.**

Die drei Zielfehler (Mail 9, 16, 20) waren behoben. Dafür entstanden vier
neue (Mail 4, 12, 13, 17). Das Modell hatte die neue Regel korrekt
angewendet — mein Goldstandard zieht die Grenze zwischen Rechnung und
Reklamation aber nicht einheitlich. Zwei fast identische Fälle, beide eine
unberechtigte Mahnung, hatte ich unterschiedlich gelabelt:

| Mail | Sachverhalt | mein Label |
|---|---|---|
| 13 | Mahnung erhalten, obwohl bezahlt | Rechnung |
| 20 | Mahnung über die Skonto-Differenz, obwohl korrekt gezahlt | Reklamation |

Solange die Regel vage war, traf das Modell diese Inkonsistenz zufällig mal
so und mal so. Eine klare Regel machte das Modell konsistent — und legte
damit offen, dass der Maßstab es nicht ist.

**Entscheidung:** zurück zu v3. Der Rückbau wurde erneut gemessen und
bestätigte 92,5 %.

**Erkenntnis:** Das Modell kann nicht konsistenter sein als der
Goldstandard, an dem es gemessen wird. Diese Messung hat nicht das Modell
bewertet, sondern das Messinstrument.

Die Überarbeitung des Goldstandards steht auf der Liste der nächsten
Schritte. Sie wurde bewusst nicht mehr im laufenden Projekt gemacht: eine
neue Grenzziehung hätte bedeutet, alle 40 E-Mails neu zu labeln, und die
alten Messwerte wären nicht mehr vergleichbar gewesen.
