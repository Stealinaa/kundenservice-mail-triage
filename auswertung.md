# Auswertung

Gemessen mit Prompt v3 gegen 40 E-Mails, die ich vorher selbst von Hand
gelabelt habe. Die Auswertung läuft vollständig über Formeln in derselben
Google-Tabelle, in der auch die Rohdaten liegen — kein separates Werkzeug,
keine manuelle Zählung.

## Kennzahlen

| Kennzahl | Wert |
|---|---|
| Kategorie korrekt | 37 von 40 = **92,5 %** |
| Dringlichkeit korrekt | 38 von 40 = **95,0 %** |
| Konfidenz unter 0,7 (an den Menschen) | 1 von 40 |
| Kategorie korrekt unter den sicheren Fällen | 36 von 39 = 92,31 % |

Die letzte Zeile ist der eigentliche Befund und wird unten erklärt.

## Wie gemessen wurde

Der Goldstandard steht in den Spalten `kategorie` und `dringend`, die
Antworten des Modells in `ki_kategorie`, `ki_dringend` und `ki_konfidenz`.
Verglichen wird zeilenweise mit Tabellenformeln, zum Beispiel:

```
Trefferquote Kategorie
=SUMPRODUCT(--(D2:D41=K2:K41))/40

Trefferquote unter den sicheren Fällen
=SUMPRODUCT((N2:N41>=0,7)*(D2:D41=K2:K41))/SUMPRODUCT(--(N2:N41>=0,7))
```

**Ein Zwischenschritt war nötig:** Die API liefert den Konfidenzwert mit
Punkt als Dezimaltrennzeichen (`0.85`). Die Tabelle erwartet in deutscher
Spracheinstellung ein Komma und legt den Wert deshalb als Text ab — jeder
Vergleich mit 0,7 wäre stillschweigend falsch ausgefallen. Statt die
Rohdaten zu ändern, steht die Umwandlung in einer eigenen Spalte:

```
=IFERROR(VALUE(SUBSTITUTE(M2;".";","));VALUE(M2))
```

So überlebt die Aufbereitung jeden neuen Durchlauf. Rohdaten bleiben roh,
die bereinigte Fassung liegt daneben.

## Schwellenwert: was er kostet und was er bringt

| Schwelle | E-Mails an den Menschen | Fehler, die durchgehen |
|---|---|---|
| 0,70 | 1 von 40 | 3 |
| 0,80 | 3 von 40 | 2 |
| 0,86 | 13 von 40 | 0 |
| 0,90 | 18 von 40 | 0 |
| 0,96 | 35 von 40 | 0 |

Oberhalb von 0,86 wächst nur noch die Arbeit: alle Fehler sind bereits
gefangen, die Zahl der Prüffälle steigt aber von 13 auf 35.

**Die Wahl der Schwelle ist eine Entscheidung über Fehlerkosten, keine
technische Einstellung.** Drei falsch abgelegte E-Mails gegen ein Drittel
des Postfachs zurück in Handarbeit — das gehört in den Fachbereich, nicht
in die Technik.

## Die drei Kategoriefehler

| Mail | Betreff | Goldstandard | Modell | Konfidenz |
|---|---|---|---|---|
| 9 | Paket nie angekommen — Frist abgelaufen | Reklamation | Lieferung | 0,85 |
| 16 | Gutschrift nicht eingegangen | Reklamation | Rechnung | 0,75 |
| 20 | Skonto abgezogen — trotzdem Mahnung | Reklamation | Rechnung | 0,85 |

Alle drei zeigen in dieselbe Richtung: eine Reklamation wurde als Lieferung
oder Rechnung eingeordnet, nie umgekehrt. Der Fehler ist also systematisch
und nicht zufällig.

Der Versuch, ihn durch eine schärfere Regel zu beheben, ist dokumentiert in
[`../prompt.md`](../prompt.md) unter v4 — und wurde verworfen.

## Befund 1: Der Konfidenzwert trennt nicht

Bei einer Schwelle von 0,7 ging genau **eine** E-Mail an den Menschen:
Mail 26 mit einer Konfidenz von 0,65. Diese E-Mail war **korrekt**
klassifiziert.

Alle drei echten Fehler kamen mit Konfidenzen zwischen 0,75 und 0,85,
also deutlich oberhalb der Schwelle.

Der Unterschied zwischen 92,50 % (alle E-Mails) und 92,31 % (nur die
sicheren) zeigt dasselbe: Das Aussteuern nach Konfidenz verbessert die
Qualität der automatisch abgelegten Post praktisch nicht.

**Das Modell ist genauso überzeugt, wenn es sich irrt.** Der Konfidenzwert
ist eine Selbsteinschätzung, keine Wahrscheinlichkeit für Richtigkeit.

Konsequenz für den Betrieb: Nicht nach Unsicherheit aussteuern, sondern nach
Fehlerkosten. Alles, was das Modell als dringende Reklamation einstuft,
sollte grundsätzlich ein Mensch sehen — dort ist ein Fehler teuer,
unabhängig davon, wie sicher sich das Modell fühlt.

## Befund 2: "Erfolgreich" heißt nicht "richtig"

Ein früherer Durchlauf (Prompt v2) meldete **81 von 81 Operationen
erfolgreich**. In der Tabelle waren aber nur **20 von 40 Zeilen** gefüllt.

Ursache: Das Modell verpackte sein JSON bei etwa jeder zweiten Antwort in
einen Markdown-Codeblock. Make konnte die Antwort dann nicht auswerten,
übergab leere Werte an die Tabelle und meldete den Schreibvorgang trotzdem
als erfolgreich — die Zelle wurde ja korrekt beschrieben, nur eben mit
nichts.

Gefunden wurde der Fehler nicht im Protokoll, sondern durch Abgleich: Ich
habe gezählt, wie viele Zeilen gefüllt sind, und die Zahl stimmte nicht mit
der Zahl der verarbeiteten E-Mails überein. Erst danach führte das Protokoll
zur Ursache.

Behoben in v3 durch ausdrückliche Formatvorgaben. Danach: 40 von 40 Zeilen.

**Merksatz: nicht den Status prüfen, sondern das Ergebnis.**

## Befund 3: Das Modell kann nicht konsistenter sein als der Maßstab

Siehe v4 in [`../prompt.md`](../prompt.md). Kurzfassung: Eine eindeutig
formulierte Regel behob die drei Zielfehler und erzeugte vier neue, weil
mein eigener Goldstandard die Grenze zwischen Rechnung und Reklamation
uneinheitlich zieht.

Zwei fast identische Fälle — beide eine unberechtigte Mahnung — hatte ich
unterschiedlich gelabelt (Mail 13: Rechnung, Mail 20: Reklamation).

Die Messung hat hier nicht das Modell bewertet, sondern den Maßstab.

## Kosten

| | |
|---|---|
| Ein Durchlauf über 40 E-Mails | ca. 0,045 USD |
| Hochgerechnet | ca. 1,10 USD je 1000 E-Mails |
| Gesamte Entwicklung (alle Versionen, Fehlersuche, fünf Durchläufe) | **0,31 USD** |
| Laufzeit eines Durchlaufs | 1 Minute für 40 E-Mails |

Die Angaben betreffen nur die API. Ein Durchlauf verbraucht zusätzlich 81
Make-Operationen (eine zum Lesen der Tabelle, je eine pro E-Mail für Modell
und Rückschreiben). Der kostenlose Make-Tarif mit 1000 Operationen im Monat
deckt damit etwa 330 E-Mails ab.

## Was diese Zahlen nicht sagen

- **40 E-Mails sind eine kleine Stichprobe.** Ein Fehler mehr oder weniger
  verschiebt die Trefferquote um 2,5 Prozentpunkte. Für das Erkennen von
  Fehlertypen reicht das, für belastbare Prozentwerte nicht.
- **Es gibt keinen Hold-out-Datensatz.** Der Prompt wurde an denselben
  E-Mails verbessert, an denen er gemessen wird. Das grenzt an Overfitting.
  Zehn E-Mails hätten von Anfang an zurückgelegt gehört.
- **Gemessen wurde nur das Sortieren**, nicht die Bearbeitung der Anfrage.
- **Die Vergleichszeit von 27,6 Sekunden** stammt aus einer Tabelle, in der
  Betreff und Text sofort sichtbar waren. Im realen Postfach wäre der
  manuelle Aufwand höher — die Zahl ist also konservativ.
