# Solace Health Clinics – Evidenz-Review

**Begleitdokument zum Notebook `Evidence_Review.ipynb`.** Jede hier genannte Zahl wird im Notebook von einer Zelle erzeugt. Die Teile folgen der Reihenfolge der Checkpoints.

---

## 1. Daten-Audit (Checkpoint 1)

Der Datensatz `solace_visits.csv` hat 1.209 Besuche (eine Zeile pro Besuch), 14 Spalten und vier Kliniken. **Zuerst wurde geprüft, dann zusammengefasst.**

### 1.1 Ausreißerregel (vor der Anwendung festgelegt)

> Ein Wert ist markiert, wenn er größer ist als **Q3 + 1,5 × IQR** oder kleiner als **Q1 − 1,5 × IQR** (IQR = Q3 − Q1). Markierte Werte werden **nicht gelöscht**, sondern angesehen und eingeordnet.

| Variable | Fallzahl markiert | Einordnung |
|---|---|---|
| `actual_duration_min` | 17 | **14 wirken wie Fehler** (das 5,7- bis 13,3-Fache der geplanten Zeit, z. B. 30 Min. geplant, 399 gedauert); 3 sind harmlos (~1,3×). Klare Lücke dazwischen. |
| `visit_cost` | 15 | **Echte Extremwerte** (ca. 464 bis 1.209 Dollar), verteilt auf mehrere Terminarten: der lange Schwanz einer rechtsschiefen Verteilung. |
| `wait_time_min` | 32 | **Gewöhnlicher oberer Rand**: Wartezeiten sind rechtsschief, Maximum ~70 Minuten, plausibel. |

(Fallzahlen auf dem bereinigten Datensatz mit 1.200 Zeilen. Auf den Rohdaten sind es 17 / 15 / 34.)

### 1.2 Was geändert wurde, und was bewusst nicht

| Problem | Befund | Entscheidung | Verworfene Alternative |
|---|---|---|---|
| **Fehlende Zufriedenheit** | 123 von 1.209 (10,2 %). Bei **Urgent 31,5 %**, sonst ~5 %. Nicht durch Gruppengröße (Specialist am kleinsten, niedrigste Rate) oder Klinik (9,4–11,0 %) erklärbar. | Lücken **markiert**, Rücklaufquote (Urgent: 68,5 %) neben jedem Durchschnitt berichtet. | Löschen (Urgent unterrepräsentiert) · mit Mittelwert auffüllen (stellt keinen wahren Wert wieder her, wiederholt die Verzerrung). |
| **Doppelte Datensätze** | 18 Zeilen (9 Paare) stimmen in **jeder** Spalte überein. „Gleicher Patient + gleiches Datum“ würde 58 markieren: 40 davon sind echte Zweitbesuche. | Nur die 9 exakten Kopien entfernt (1.209 → 1.200). | Alle 58 entfernen (zerstört 40 echte Besuche). |
| **Spaltentyp `zip_code`** | Als Zahl gespeichert, in allen Zeilen fehlt die führende Null (`8540` statt `08540`). | Als Text mit 5 Stellen (sicher: keine PLZ länger als 5 Stellen). | Unverändert lassen (Verknüpfungen würden still fehlschlagen). |
| **Kategorienspalten** | 4 / 4 / 4 / 7 Werte, keine Leerzeichen- oder Schreibfehler. | **Keine Änderung** (aktiv geprüft). | – |
| **Ausreißer** | siehe 1.1 | **Markiert** (14 Dauern zur Prüfung), nichts gelöscht. | Löschen. |

### 1.3 Welches Problem hätte eine Schlussfolgerung verändert?

- **Fehlende Werte** verändern die Aussage über die **Zufriedenheit**: Der Urgent-Durchschnitt (4,06) stützt sich nur auf 68,5 % der Urgent-Besuche. Ohne Hinweis würde er als vollständig gelesen.
- **Doppelte Datensätze** verändern keine Schlussfolgerung dieses Projekts (z. B. Mittelwert Besuchskosten 190,73 → 190,83; Unterschied Riverside–Downtown 8,54 → 8,58 Minuten). Zählungen wären um 9 Besuche aufgebläht.
- **Der `zip_code`-Typ** ändert keine der hier berechneten Zahlen, weil die Spalte nirgends verwendet wird.

### 1.4 Zusammenfassung von `visit_cost`

| Minimum | Q1 | Median | Q3 | Maximum | Mittelwert |
|---|---|---|---|---|---|
| 70,86 | 130,13 | 157,28 | 263,73 | 1.208,54 | 190,83 |

Der Abstand Median → Q3 (~106,5) ist rund viermal so groß wie Q1 → Median (~27,2), das Maximum liegt beim Achtfachen des Medians, der Mittelwert über dem Median (Skewness ≈ 2,5): Die Verteilung ist **rechtsschief**, ein langer Schwanz teurer Besuche (9 Besuche über 500 Dollar).

**Einzelwert für die Folie: der Median (~157 Dollar)**, zusammen mit der Frage, die er beantwortet („Was kostet ein gewöhnlicher Besuch?“). Der **verworfene Mittelwert** beantwortet: „Wie hoch sind die Gesamtkosten geteilt durch die Zahl der Besuche?“ Nur Mittelwert × Anzahl ergibt die tatsächliche Summe. Für ein Quartalsbudget wäre er richtig, der Median läge dort ~34 Dollar pro Besuch zu niedrig.

### 1.5 Zusammenhänge

- **Zwei Zahlenspalten** (Wartezeit, Zufriedenheit): r ≈ **−0,39** (n = 1.077 Besuche mit Antwort). Wer länger wartet, ist tendenziell unzufriedener, **mäßig stark** (~15 % der Unterschiede in der Zufriedenheit gehen mit der Wartezeit einher). Vorbehalt: Die Lücken liegen vor allem bei Urgent.
- **Zwei Kategorien** (Terminart, No-Show): Chi-Quadrat ≈ 13,3 (3 Freiheitsgrade), p ≈ 0,004, Cramérs V ≈ **0,105**. Statistisch auffällig, aber **schwach**: Bei Urgent bleiben ~17 % dem Termin fern, sonst 7–10 %. Ein kleiner p-Wert sagt nicht, dass der Zusammenhang stark ist.

---

## 2. Einseitiger Test: das Serviceversprechen in Downtown (Checkpoint 2)

H₀: Die wahre mittlere Wartezeit in Downtown beträgt 15 Minuten oder mehr. H₁: Sie beträgt weniger als 15 Minuten. (n = 430, Mittelwert 12,18 Minuten, t = −7,20, df = 429, p einseitig ≈ 1,3 · 10⁻¹².)

1. **Was behaupten die Hypothesen, welche schützt der Test?** Die Nullhypothese sagt: Das Versprechen wird *nicht* gehalten (15 Minuten oder länger). Die Alternative sagt: Es wird gehalten (weniger als 15). Der Test **schützt die Nullhypothese**: Sie gilt, bis die Daten nur noch schwer anders zu erklären sind. Ein nicht signifikantes Ergebnis hieße deshalb nicht „das Versprechen wird gebrochen“, sondern „die Daten reichen nicht, um es zu belegen“.
2. **Was bedeutet α = 0,05, was änderte sich bei α = 0,01?** Operations nimmt in Kauf, in höchstens 5 % der Fälle zu behaupten „das Versprechen wird gehalten“, obwohl es nicht stimmt. Die Schwelle wird vor dem Blick auf die Daten festgelegt. Bei α = 0,01 wäre die Hürde strenger: Man sagt nur bei sehr starker Evidenz „belegt“, übersieht dafür häufiger echte Verbesserungen. Hier ändert sich nichts, weil der p-Wert weit unter beiden Schwellen liegt.
3. **Welche Verteilung, was bewirkt df = 429?** Die **t-Verteilung**, nicht die Normalverteilung: Die Streuung der Wartezeiten war nicht vorher bekannt, sie wurde aus denselben 430 Besuchen geschätzt. Diese zusätzliche Unsicherheit bildet die t-Verteilung mit etwas breiteren Rändern ab. Die Freiheitsgrade (430 − 1 = 429) bestimmen, wie breit die Ränder sind. Bei so vielen Besuchen ist sie von der Normalverteilung kaum zu unterscheiden.
4. **Warum ein Rand, und wann wäre das Halbieren falsch?** Die Alternative lautet „kleiner als 15“, nur das ist für die Frage (Versprechen eingehalten?) interessant, und die Richtung stand vor den Daten fest. Ein zweiseitiger p-Wert (≈ 2,7 · 10⁻¹²) würde auch Abweichungen nach oben als Evidenz zählen und an der Frage vorbeigehen. Die Division durch 2 wäre **stark falsch** gewesen, wenn der Stichprobenmittelwert **über** 15 gelegen hätte: Dann wäre der richtige einseitige p-Wert nahe 1.
5. **Was bedeutet p = 0,0000000000013, was sagt man Operations?** *Wenn die wahre mittlere Wartezeit tatsächlich 15 Minuten oder mehr betrüge, wäre es extrem unwahrscheinlich (etwa 1 zu einer Billion), in 430 Besuchen einen Durchschnitt zu sehen, der so weit darunter liegt wie 12,2 Minuten.* Schlussfolgerung: **Die Daten dieses Quartals stützen das Versprechen** für den Durchschnitt in Downtown (rund 12,2 Minuten, Obergrenze ~12,8). Der p-Wert sagt **nicht**, wie groß die Verbesserung ist, und **nicht**, dass jeder Patient unter 15 Minuten wartet: Rund 28 % warten länger.

---

## 3. Zweiseitiger Vergleich: Riverside gegen Downtown (Checkpoint 3)

H₀: Beide Standorte haben dieselbe wahre mittlere Wartezeit. H₁: Sie unterscheiden sich. (Riverside n = 290, Mittelwert 20,72 Minuten; Downtown n = 430, Mittelwert 12,18; Unterschied 8,54 Minuten; Welch-t = 13,51; p ≈ 1,6 · 10⁻³⁶; 95-%-Intervall 7,30 bis 9,78 Minuten; Cohens d ≈ 1,03.)

1. **Hypothesen, warum zweiseitig?** Die Zahl der Seiten folgt der **Frage**, nicht einer Vermutung. Regional Operations würde bei **jedem** Ergebnis handeln: Wäre Riverside *schneller*, fiele die Personalentscheidung ebenfalls anders aus. Beide Richtungen sind bedeutsam. In Checkpoint 2 zählte dagegen nur eine Richtung (Versprechen eingehalten oder nicht).
2. **Welcher Test, was ändert `equal_var=False`?** Ein **Zwei-Stichproben-t-Test nach Welch**. Der ältere Test nimmt gleiche Streuung an und rechnet mit einer gemeinsamen. Welch lässt jede Gruppe ihre eigene Streuung behalten und passt die Freiheitsgrade an (≈ 604). Das ist der sichere Standard, weil die Gruppen **ungleich groß** sind (290 gegen 430): Streuen sie unterschiedlich, kann der klassische Test zu optimistische oder zu vorsichtige Ergebnisse liefern. Sind die Streuungen gleich (8,45 und 8,12), kostet Welch praktisch nichts.
3. **Typ-I- und Typ-II-Fehler.** *Typ I:* Wir schließen „Riverside wartet länger“, obwohl die Standorte gleich sind: Zwei Mitarbeitende werden **unnötig versetzt**, sie fehlen anderswo, Kosten und Unruhe ohne Nutzen. *Typ II:* Wir schließen „kein Unterschied“, obwohl Riverside wirklich länger wartet: Die Patienten dort warten weiter länger, die Hilfe kommt nicht. α begrenzt den **Typ-I-Fehler**. Über den Typ-II-Fehler enthält die Ausgabe **keine Information**.
4. **p ≈ 10⁻³⁶ und was man Operations zeigt.** Der p-Wert belegt, dass ein Unterschied dieser Größe bei gleicher wahrer Wartezeit praktisch nicht durch Zufall entstünde: Die Behauptung „nur Rauschen“ ist widerlegt. Er sagt **nicht**, wie groß der Unterschied ist, und nicht, ob er relevant ist. Gezeigt wird das **Konfidenzintervall**: *Riverside wartet durchschnittlich etwa 7,3 bis 9,8 Minuten länger als Downtown.* Damit lässt sich Personal planen. Cohens d (große Wirkung) ordnet die Größe im Verhältnis zur Streuung ein; der p-Wert gehört nicht auf die Folie.

---

## 4. Design-Review: der SMS-Erinnerungsplan (Checkpoint 4)

### 4.1 Die sechs Annahmen, einzeln geprüft

| # | Annahme | Erfüllt? | Stelle im Plan, die sie verletzt |
|---|---|---|---|
| 1 | Zufällige Zuteilung | **Nein** | „Riverside wurde ausgewählt, weil sich der Site Manager freiwillig gemeldet hat und weil die No-Show-Rate dort am höchsten ist“ (Auswahl nach Freiwilligkeit und Bedarf, nicht Zufall) |
| 2 | Unabhängigkeit der Einheiten | **Nein** | „Riverside erhält im gesamten März die neue SMS-Erinnerung. Downtown behält den aktuellen Prozess“ (Zuteilung je Standort: eine Einheit pro Arm, Besuche teilen Personal und Abläufe) |
| 3 | Keine Interferenz zwischen den Armen | **Nein** | „Der Manager von Downtown … begann, seinen eigenen Patienten Erinnerungs-E-Mails zu senden“ (Kontrolle bekommt eine Erinnerung); „… begann Patienten ohne hinterlegte Mobilnummer telefonisch anzurufen“ (Treatment ist nicht mehr nur eine SMS) |
| 4 | Gleichzeitigkeit | **Nein** | „Vergleiche die No-Show-Rate von Riverside im März mit der Rate von Downtown im Februar“ |
| 5 | Vorab deklarierte primäre Kennzahl, gleich gemessen | **Nein** | „Wir sehen uns No-Shows an und außerdem Wartezeiten, Zufriedenheit und Umsatz pro Besuch und berichten, was sich davon bewegt“ · „weil die Februar-Zahlen … bereits bereinigt und geladen wurden“ (Februar und März unterschiedlich aufbereitet) |
| 6 | Ausreichende Power | **Nein** | „Riverside hat ungefähr 290 Besuche pro Quartal, Downtown ungefähr 430 … erwartet … um etwa 2 Prozentpunkte“ (März allein: ca. ein Drittel davon) |

**Rechenbeispiel zu Annahme 6:** Für 2 Prozentpunkte weniger (bei rund 11 % Grundrate, 80 % Power) bräuchte man etwa **3.500 Besuche je Arm**. Mit rund 100 (Riverside) und 140 (Downtown) Besuchen im März läge die Chance, einen echten Effekt dieser Größe zu finden, bei etwa 7 %.

*Randnotiz:* In den Daten dieses Quartals hat Riverside **nicht** die höchste No-Show-Rate (9,4 %, hinter Eastview 12,7 % und Downtown 11,6 %). Die Begründung der Auswahl sollte der Plan belegen.

### 4.2 Urteil in einem Satz

> **Der Plan sollte so nicht durchgeführt werden, weil Riverside freiwillig und wegen hohen Bedarfs ausgewählt und mit Downtown in einem anderen Monat verglichen wird: Jeder Unterschied wäre nicht von Standort, Jahreszeit und Auswahl zu trennen, und das lässt sich durch Umschreiben nicht beheben.**

**Die entscheidende Verletzung** ist Annahme 1 (zufällige Zuteilung) in Verbindung mit „ein Standort gegen einen Standort“: Sie steckt in der **Struktur des Vergleichs**. Selbst mit korrigierter Kennzahl, gleichem Monat und gleicher Messung bliebe ein Unterschied zwischen *Riverside* und *Downtown*, den man nicht von der SMS trennen kann. Durch **Umschreiben reparierbar** wären dagegen die Annahmen 4, 5 und 6 (und 3 mit klaren Regeln für das Front-Desk-Team).

**Wo hätte ein Ergebnis signifikant werden können, ohne etwas zu bedeuten?** Beim Vergleich „Riverside im März gegen Downtown im Februar“: Ein Unterschied hätte jeden Grund haben können (Standort, Monat, Freiwilligkeit, andere Aufbereitung), und bei vier berichteten Kennzahlen wäre mit Glück irgendeine „signifikant“ gewesen.

### 4.3 Redesign: eine Version, die es wert wäre

**Kurz:** Jeden Termin zufällig der SMS-Gruppe oder der Gruppe ohne Erinnerung zuteilen, über alle vier Standorte, im selben Zeitraum, mit **einer** vorab festgelegten Kennzahl (No-Show-Rate pro Termin) und einer vorher berechneten Stichprobe (~3.500 Termine je Gruppe, bei ~1.200 Besuchen pro Quartal also länger als ein Quartal, oder einen größeren Mindesteffekt festlegen). Das Front-Desk-Team schickt keine zusätzlichen Erinnerungen; Abweichungen werden protokolliert. Feste Laufzeit, kein Abbruch, wenn es gut aussieht.

**Falls das nicht möglich ist** (Termine lassen sich nicht randomisieren): Standorte **zufällig und zeitlich versetzt** umstellen; wenn auch das nicht geht, den Vergleich offen als **Beobachtungsstudie** durchführen (Vorher-Nachher mit Vergleichsstandorten) und das Ergebnis nur als Hinweis berichten, nicht als Nachweis einer Wirkung.

---

## Was diese Evidenz nicht belegen kann

- Dass jeder Patient in Downtown unter 15 Minuten wartet (rund 28 % warten länger).
- **Warum** Riverside länger wartet: Der Test zeigt einen Unterschied, keine Ursache.
- Ob eine SMS-Erinnerung No-Shows senkt: Dafür gibt es in diesem Datensatz keine Daten, und der vorgeschlagene Plan würde es auch nicht klären.
- Zufriedenheit für alle Patienten: Die Werte fehlen vor allem bei Urgent.
