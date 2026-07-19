# Kitzrettung Thüringen – Bericht

Dokumentation zu `bericht_kitzrettung.qmd` und `kitzretter.yml`. Beschreibt, was der Bericht zeigt, wie jede Zahl berechnet wird und warum die Grafiken so aufgebaut sind, wie sie sind.

## 1. Grundprinzip

Der Bericht liest alle Daten aus einer einzigen Datei, `kitzretter.yml`. Jeder **Kitzretter** ist ein Drohnenteam (meist selbst ein Verein/eine Jägerschaft), jeder **Einsatz** ist ein einzelner Rettungseinsatz mit Datum, Dauer und optionalen Zusatzangaben.

```yaml
kitzretter:
  - name: "Drohnenteam Kyffhäuser"
    verein: "Jägerschaft Kyffhäuser"
    einsaetze:
      - datum: 2026-04-20
        dauer_h: 4.0
        flaeche_ha: 40
        kitze_gerettet: 5
        auftraggeber: "Landwirtschaftsbetrieb Seehausen"
```

- **`name`** = das Drohnenteam selbst (die eigentliche Analyseeinheit)
- **`verein`** = Zugehörigkeit des Teams
- **`auftraggeber`** = wer den Einsatz gebucht hat (Landwirt, Jägerschaft) — wird aktuell nur dokumentiert, fließt in keine Berechnung ein
- **`flaeche_ha`** = wird aktuell nicht ausgewertet, ist für spätere Kennzahlen (z. B. Hektar/Stunde) vorbereitet
- Das **Jahr** einer Einsatzzeile wird nicht separat gepflegt, sondern immer aus `datum` abgeleitet

## 2. Berichtsjahr

```r
berichtsjahr <- daten$berichtsjahr %||% max(alle_einsaetze$jahr)
```

Standardmäßig nimmt der Bericht automatisch das **jüngste Jahr**, für das Einsätze erfasst sind. Optional lässt sich `berichtsjahr:` in der YAML fest setzen (z. B. um rückwirkend einen alten Jahrgang neu zu rendern).

Alle Abschnitte, die explizit "`Jahr`" im Titel tragen (Gesamtergebnis, Einzelanalyse, Grafiken), beziehen sich **nur auf das Berichtsjahr** — nicht auf die gesamte Historie.

## 3. Kennzahlen je Kitzretter (Berichtsjahr)

Pro Team, gefiltert auf `jahr == berichtsjahr`:

| Kennzahl | Formel |
|---|---|
| Einsätze | Anzahl Einsatzzeilen des Teams im Berichtsjahr |
| Gesamtstunden | Summe `dauer_h` über alle Einsätze des Teams |
| Ø Std/Einsatz | Gesamtstunden ÷ Einsätze (je Team) |
| Kitze gerettet | Summe `kitze_gerettet` über alle Einsätze des Teams |
| Kitze/h (Effizienz) | Kitze gerettet ÷ Gesamtstunden (je Team) |
| Gegenwert (€) | Summe (`dauer_h` × Mindestlohn des jeweiligen Einsatzjahres) — siehe Abschnitt 5 |

## 4. Gesamtergebnis (Berichtsjahr, alle Teams zusammen)

Wichtig: **Ø Stunden je Einsatz (Thüringen)** ist **nicht** der Durchschnitt der Team-Durchschnitte, sondern:

```
Ø Stunden je Einsatz (Thüringen) = Summe aller Stunden ÷ Summe aller Einsätze
```

Das ist bewusst so gewählt: Der Durchschnitt der Einzeldurchschnitte würde ein Team mit wenigen, langen Einsätzen genauso stark gewichten wie ein Team mit vielen, kurzen Einsätzen — das verzerrt bei unterschiedlicher Einsatzzahl je Team. Die gewichtete Variante (Gesamtstunden/Gesamteinsätze) bildet die tatsächliche Thüringen-weite Realität ab.

`Ø Gesamtstunden je Kitzretter` ist dagegen absichtlich der einfache Mittelwert der Team-Gesamtstunden (nicht gewichtet), weil hier die Frage lautet "wie viel leistet ein Team im Schnitt" — nicht "wie lang dauert ein Einsatz im Schnitt".

## 5. Wirtschaftlicher Gegenwert (Mindestlohn-Basis)

```r
gegenwert = dauer_h × mindestlohn_jahr
```

`mindestlohn_jahr` wird **pro Einsatz aus dem Jahr des jeweiligen Einsatzdatums** nachgeschlagen, nicht pauschal mit dem aktuellen Satz berechnet — sonst würden 2025er-Einsätze mit dem höheren 2026er-Satz aufgebläht.

Aktuelle Tabelle (in `bericht_kitzrettung.qmd`, Chunk `setup`):

| Jahr | Mindestlohn (€/h) |
|---|---|
| 2023 | 12,00 |
| 2024 | 12,41 |
| 2025 | 12,82 |
| 2026 | 13,90 |
| 2027 | 14,60 |

**Bei neuer gesetzlicher Erhöhung**: Zeile in `mindestlohn_tabelle` ergänzen. Für nicht gelistete Jahre greift automatisch der zuletzt bekannte Satz als Näherung (`get_mindestlohn()`-Fallback) — sollte aber bei Bedarf manuell nachgepflegt werden.

**Einordnung**: Der Mindestlohn ist eine **konservative Untergrenze**, keine realistische Bezahlung für Drohnenpilotage (Ausrüstung, Ausbildung, Verantwortung bleiben unberücksichtigt). Der Bericht benennt das explizit im Fließtext, um die Zahl nicht als "das ist der reale Wert der Arbeit" misszuverstehen.

## 6. Mehrjahresübersicht

Eine Zeile je Jahr, aggregiert über **alle** Teams, die in diesem Jahr mindestens einen Einsatz hatten:

- **Aktive Teams** = Anzahl unterschiedlicher Teamnamen mit ≥1 Einsatz im Jahr
- **Gegenwert** je Jahr nutzt den für dieses Jahr gültigen Mindestlohn (nicht den des Berichtsjahres)
- **Wirtschaftliche Gesamtleistung seit Erfassungsbeginn** = Summe über alle Jahre und Teams hinweg

### Team-Fluktuation

Wird automatisch aus den Daten abgeleitet, keine manuelle Pflege nötig:

```r
status = "nicht mehr aktiv"       wenn letztes_Einsatzjahr < Berichtsjahr
status = "neu seit diesem Jahr"   wenn erstes_Einsatzjahr == Berichtsjahr
status = "aktiv"                  sonst
```

Ein Team verschwindet aus dem Bericht einfach dadurch, dass in `kitzretter.yml` keine neuen Einsätze mehr für dieses Jahr eingetragen werden — es muss nicht gelöscht oder als "inaktiv" markiert werden. Die Historie bleibt erhalten (fließt weiter in die Mehrjahresübersicht und in Team-Boxplots ein), taucht aber nicht mehr in den Berichtsjahr-spezifischen Abschnitten auf.

## 7. Grafiken

**Gesamtstunden je Kitzretter** und **Ø Stunden je Einsatz je Kitzretter**: Lollipop-Charts (Segment + Punkt statt Balken), Farbe kontinuierlich nach Wert abgestuft (`scale_color_gradient`). Bei der Einsatzdauer zusätzlich eine gestrichelte Linie für den Thüringen-Durchschnitt (gewichtet, siehe Abschnitt 4).

**Boxplot je Kitzretter**:
- **Box** = gesamter erfasster Zeitraum des Teams (alle Jahre, nicht nur Berichtsjahr)
- **Punkte** = nur die Einsätze aus dem Berichtsjahr
- Zweck: zeigt die langfristige Streuung eines Teams UND ordnet ein, wo die aktuellen Einsätze innerhalb dieser Historie liegen
- Nur Teams, die im Berichtsjahr aktiv sind, werden gezeigt (`kitzretter_df$name`-Filter)

**Boxplot je Jahr — "Konsistenz der Einsatzdauer über die Jahre"** (in der Mehrjahresübersicht):
- Eine Box pro Jahr, über alle Teams hinweg, unabhängig vom Berichtsjahr
- Wird die Box über die Jahre schmaler, deutet das auf konsistentere/eingespieltere Einsatzdauer hin — allerdings erst mit mehreren Jahrgängen wirklich aussagekräftig, mit nur 2 Jahren noch früh

**Wichtig: Konsistenz ≠ Effizienz.** Der Titel heißt bewusst "Konsistenz", nicht "Effizienz" — beide Begriffe werden leicht verwechselt, meinen aber Unterschiedliches:
- **Konsistenz** = wie vorhersehbar/gleichmäßig die Einsatzdauer ist (schmale Box = wenig Streuung). Sagt nichts darüber, ob die Einsätze *gut* laufen.
- **Effizienz** = Output pro Input, z. B. Kitze gerettet pro Stunde oder Fläche pro Stunde. Sagt, ob die Arbeit *wirksam* ist.
- Ein Team kann konsistent **und** ineffizient sein (immer verlässlich 4 Stunden für eine Fläche, die ein anderes Team in 2 Stunden schafft). Umgekehrt kann ein Team stark schwanken, aber im Schnitt sehr effizient sein.
- Der Bericht bildet **beide Dimensionen getrennt** ab: Konsistenz über den Jahres-Boxplot (Streuung der Dauer), Effizienz über die Kennzahl **Kitze/h** (`kitze_gerettet / dauer_h`) — als eigene Spalte in Gesamtergebnis, Einzelanalyse und Mehrjahresübersicht, plus eigene Grafik "Effizienz: Kitze gerettet je Stunde" je Kitzretter.

**Boxplot-Konventionen** (gilt für beide Boxplots):
- Box = mittlere 50 % der Werte (1. bis 3. Quartil, Q1–Q3)
- Strich in der Box = Median
- IQR (Interquartilsabstand) = Q3 − Q1 = Boxbreite
- Whisker = Werte außerhalb der Box bis maximal 1,5 × IQR
- Punkte außerhalb der Whisker wären statistische Ausreißer (aktuell keine vorhanden)
- **Einschränkung bei kleinen Fallzahlen**: Bei 2–4 Einsätzen je Team sind Quartile statistisch wenig belastbar — die Box ist dann eher eine grobe Orientierung als eine robuste Kennzahl. Wird mit mehr Einsätzen und mehr Jahren automatisch aussagekräftiger.

`set.seed()` vor jedem Boxplot-Chunk sorgt dafür, dass die (zufällig gestreuten) Punktpositionen bei jedem Rendern gleich bleiben — sonst sähe der Chart bei jedem `quarto render` leicht anders aus.

**Gesamtstunden & Gegenwert je Jahr**: Balkendiagramm, ein Balken pro Jahr, mit Stunden- und Euro-Wert direkt als Label im Balken.

## 8. Design/Technik

- **Farbschema**: Waldgrün (`#1f3d2c` bis `#a9c2ac`) für Stunden-Kennzahlen, Bernstein/Sandgold (`#7a4f1c` bis `#e6c789`) für Einsatzdauer-Kennzahlen — durchgängig als Farbverlauf statt Flatfarbe
- **Kopfbereich**: farbig unterlegte Box (`\colorbox`) mit Titel, Untertitel, Erfasser (`daten$erstellt`), Stand (`daten$erstellt_am`), Berichtsjahr — ersetzt den Standard-Pandoc-Titel (`\renewcommand{\maketitle}{}`)
- **Tabellen**: Zebrastreifen (`\rowcolors`, `xcolor[table]`) in dezentem Salbeigrau; zu breite Tabellen werden automatisch auf Textbreite runterskaliert (`\resizebox`), schmale Tabellen bleiben in normaler Schriftgröße (`\ifdim`-Bedingung), damit keine Tabelle künstlich aufgeblasen wird
- **Grafik-Device**: `quartz_pdf` (macOS-nativ), damit Unicode-Zeichen wie € in den Charts korrekt dargestellt werden — `cairo_pdf` würde X11/XQuartz voraussetzen, das auf dem Zielsystem nicht installiert ist
- Alle Geldbeträge: `format(..., big.mark = ".", decimal.mark = ",")` für deutsche Zahlschreibweise (1.234,00 statt 1,234.00)

## 9. Bekannte Grenzen / offene Punkte

- `flaeche_ha` und `auftraggeber` werden erfasst, aber nicht ausgewertet — Potenzial für spätere Kennzahlen (z. B. Hektar/Stunde, Top-Auftraggeber)
- **Kitze/h (Effizienz) ist gewichtsblind je Einsatz**: Die Kennzahl summiert über alle Einsätze eines Teams/Jahres (Gesamt-Kitze ÷ Gesamt-Stunden), rechnet also nicht Einsatz für Einsatz einzeln. Bei sehr ungleich langen Einsätzen kann ein einzelner besonders erfolgreicher (oder erfolgloser) Einsatz die Zahl stark verschieben — bei kleinen Fallzahlen (2–4 Einsätze) mit Vorsicht lesen, aus demselben Grund wie bei den Boxplots
- Mindestlohn-Tabelle muss manuell gepflegt werden, wenn ein neues Jahr hinzukommt und noch keine gesetzliche Festlegung existiert (Fallback nutzt sonst den letzten bekannten Satz)
- Boxplots bei sehr kleinen Fallzahlen (2–3 Einsätze) sind statistisch mit Vorsicht zu lesen
