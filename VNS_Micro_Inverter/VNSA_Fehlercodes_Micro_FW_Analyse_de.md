# Venus A — Fehler- und Warncodes der Wechselrichter-Firmware

> 🇬🇧 English version: [VNSA_Error_Codes_Micro_FW_Analysis.md](VNSA_Error_Codes_Micro_FW_Analysis.md)

> **Korrektur (2026-10-02):** Diese Analyse entstand auf einem Image, das an der falschen Basisadresse importiert war. Die
> vollständige Neuanalyse auf der korrekten Basis (0x08006000) steht in [VNSA_Micro_FW_v1211_Vollanalyse_de.md](VNSA_Micro_FW_v1211_Vollanalyse_de.md) §6.
> Die Menge der vorhandenen Bits ist bestätigt (err1 0–4, 7, 20, 21; err2 5, 6, 7, 8, 9, 11, 13; warn wird nie gesetzt), mehrere
> Bedeutungen ändern sich aber: err1 Bit 0/1 = Netz-Über-/Unterspannung, Bit 2/3 = Netz-Über-/Unterfrequenz, err1 Bit 20 =
> BMS-Kommunikation verloren, Bit 21 = BMS-Fehlercode >= 4, err2 Bit 6 = Zwischenkreis > 530 V (570 V in einem LLC-Modus), err2 Bit 7 =
> Abschaltung durch die Zustandsmaschine (Grund 5), Bit 9 = ungültiger Zustandsindex, Bit 11 = Zeitüberschreitung beim
> Zustandswechsel, Bit 13 = Relaisprüfung fehlgeschlagen. Funktionsnamen und Adressen in diesem Dokument beziehen sich auf das
> alte Programm mit falscher Basis.

**Gerät:** Marstek Venus A (VNSA-0), Micro/Inverter-MCU (`VA_inv_app_*`)
**Stand:** 21. September 2026
**Anlass:** Anwendermeldung aus dem Forum — `fault_status_2_low` (36103) sprang mehrfach kurz
auf Werte um 2330–2380, dazu `mppt_error`/`mppt_warning` ungleich 0. Keine Debug-Logs verfügbar.
**Status:** Fehlerwort-Block lokalisiert, Modbus-Zuordnung für Control v150 bewiesen, **alle**
Setzstellen abschließend enumeriert. **Der Anlassfall ist aufgeklärt:** kein Gerätefehler, sondern
verrutschte Modbus-Antworten nach einem Lese-Timeout — der Ausschluss gelang über die Bitliste
(Abschnitt 9).
**Werkzeuge:** Ghidra (Neuimport + Lückendisassembly), Python-Rohscanner über die Images.

---

## 1. Ausgangslage

`a.yaml` der HA-Integration führt für `fault_status*` dieselben `bit_descriptions` wie `d.yaml`
(„Hardware protection latch A/B/C", „Limit exceeded for more than 3000 cycles"). Die stammen aus
`VNSD_Fehlercodes_Micro_FW_Analyse_de.md`, also aus der **Venus-D**-Micro-Firmware v116. Ob sie für den A
gelten, war offen — `Methodik_und_Meta/Geraetevergleich_VNSD_VNSA_VNSE3_de.md` behandelt nur den
Control, nicht den Wechselrichter.

**Sie gelten nicht.** Das ist jetzt belegt, nicht mehr nur vermutet.

## 2. Die A-Wechselrichter-Firmware ist eine andere Codebasis

Blockvergleich über alle 24-Byte-Fenster (Schrittweite 4) der Images:

| Vergleich | gemeinsame Blöcke | Anteil |
|---|---|---|
| D-Micro 116 ↔ **A-Micro 1211** | 103 | **0,4 %** |
| D-Micro 116 ↔ D-Micro 115 | 17.582 | 62,9 % |
| A-Micro 1211 ↔ A-Micro 119 | 19.192 | 56,4 % |

Innerhalb einer Familie teilen sich zwei Versionen rund 60 % der Blöcke, über die Familien hinweg
praktisch nichts. Zusätzlich geprüft: die Befehlsfolge von `HW_Protection_Latch_To_ErrorBits`
(D-Micro, `0a 79 42 f0 02 02 0a 71 …`) kommt in **keinem** der drei A-Images vor.

Das ist der Gegensatz zum Control, der zwischen D und A funktionsweise byte-identisch ist
(`Methodik_und_Meta/Geraetevergleich_VNSD_VNSA_VNSE3_de.md`, Abschnitt 1). Der Control erzeugt aber keine Fehlercodes,
er reicht sie durch — die Semantik kommt vom Wechselrichter, und der ist beim A ein anderes Gerät.

**Die Venus E3 ist der Gegenfall:** ihr Wechselrichter (`inv_app_0119`) teilt 43,2 % der Blöcke mit
dem D-Image, hat denselben Reset-Handler `0x080042AD`, erreicht den Fehlerblock ebenfalls über einen
Literal-Pool und trägt bitgenau dieselbe Belegung samt Event-Codes. Für die E3 gelten die D-Texte
also — nur für den A nicht. Details in `VNSD_Fehlercodes_Micro_FW_Analyse_de.md`, Abschnitt 10.

### Binary-Fingerprint der A-Micro-Images

| Version | Datei | Größe | Flash | Initial SP | Reset Handler | MD5 |
|---|---|---|---|---|---|---|
| **1211** | `1211_micro_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin` | 146.432 B | `0x08000000–0x08023BFF` | `0x20002400` | `0x08006435` | `c1d5ff1c…` |
| 1193 | `1193_micro_VNSA-0_VA_inv_app_1193_0512_ota_092355.bin` | 146.432 B | dito | dito | dito | `78683f90…` |
| 119 | `119_micro_VNSA-0_VA_inv_app_0119_0326_ota_144725.bin` | 146.432 B | dito | dito | dito | `7e4cca4a…` |

Feste OTA-Slot-Größe wie bei D-Micro und BMS. **Offen:** der initiale SP steht auf `0x20002400`,
die Globals liegen aber bei `0x2010xxxx`/`0x2012xxxx`. Entweder zwei RAM-Bänke oder ein anderer
Controllerhersteller als beim D — nicht weiterverfolgt, für die Fehlercodes ohne Belang.

## 3. Ghidra-Setup

Ein Rohbinary lässt sich über ReVas `import-file` nicht importieren (Batch-Import erkennt kein
Format). Stattdessen per Skript: `ProgramDB` mit `ARM:LE:32:Cortex`/`default` anlegen,
`createInitializedBlock("flash", 0x08000000, …)` aus der Datei, dazu ein uninitialisierter
`sram`-Block ab `0x20000000` (0x8000) — letzterer nur, damit Ghidra für die Globals Referenzen
und `DAT_`-Symbole anlegt, was der D-Micro-Import nicht hat.

Danach wie beim D: Auto-Analyse, Vektortabelle als Funktionseinstiege, dann
Lücken-Disassembly im Thumb-Modus in mehreren Durchläufen.

| | nach Auto-Analyse | nach Lückenlauf |
|---|---|---|
| disassembliert | 27,6 % (40.410 B) | **83,4 %** (122.070 B) |
| Funktionen | 117 | **294** |
| in Funktionen | — | 58,1 % |

Programm im Projekt: `/VNSA/Micro/Micro_VNSA_1211_VA_inv_app_1211.bin`.

Die bekannte Kehrseite des pauschalen Laufs (Literal-Pools als Code) gilt auch hier; die
ausgewerteten Stellen wurden gegen die Rohbytes verifiziert.

## 4. Wo die Fehlerwörter liegen

Drei `u32` ab SRAM **`0x20101650`**:

| Offset | Adresse | Inhalt |
|---|---|---|
| +0x00 | `0x20101650` | err1 |
| +0x04 | `0x20101654` | err2 |
| +0x08 | `0x20101658` | warn |

Die Reihenfolge im Speicher ist dieselbe wie beim D (`0x200019F4`/`F8`/`FC` = err1/err2/war1).

## 5. Beweis der Modbus-Zuordnung

`FUN_08002a80` (Telemetrie-Builder, gerufen aus `FUN_08013290`) füllt einen Puffer ab
`0x2012046c` und schickt **48 Byte** per `FUN_080145b0(handle, 0x30, &DAT_2012046c)`:

```
_DAT_20120470 = _DAT_20101658;   // Payload +0x04  <- warn
_DAT_20120474 = _DAT_20101650;   // Payload +0x08  <- err1
_DAT_20120478 = _DAT_20101654;   // Payload +0x0C  <- err2
```

Dass dieser Payload 1:1 das Control-Struct `0x20014E9C` ist, zeigt der Feldvergleich — nicht nur
die drei Fehlerwörter, sondern der gesamte Rest:

| Payload | Typ | Control-Struct (aus `Modbus_RS485_TCP/VNSD_Descriptor_Offene_Register_Ghidra_Befund_de.md`, §2a) |
|---|---|---|
| +0x04 / +0x08 / +0x0C | u32 | +0x04 `warn_code`, +0x08 `error_code1`, +0x0C `error_code2` |
| +0x14 … +0x27 | durchgehend u16 | +0x14 `off_grid_volt` … +0x26 `min_power` |
| +0x28, +0x2C | u32 | +0x28 `chrg_energy`, +0x2C `dischrg_energy` |

Die u16/u32-Grenzen fallen exakt zusammen. Damit gilt:

| SRAM | Control-Struct | Modbus |
|---|---|---|
| `0x20101658` warn | +0x04 | **36000 / 36001** |
| `0x20101650` err1 | +0x08 | **36100 / 36101** (Alias 37013) |
| `0x20101654` err2 | +0x0C | **36102 / 36103** (Alias 37015) |

Die Descriptor-Tabelle des A-Control ist an diesen Stellen identisch zum D — geprüft durch
Entpacken (`Scripts/marstek_descriptor_unpack.py` (lokal, nicht im Repo)) von A-v150 (`0x20014EA0/A4/A8`, wie D-v150)
und A-1509 (`0x200176A0/A4/A8`, nur Layout-Drift). 246 Einträge in beiden Fällen.

**Unabhängige Gegenprobe am Gerät:** der meldende Anwender berichtet 36102 (High-Word) = 0 und
36000/36001 = 0. Beides ist strukturell zwingend — für err2 existiert keine einzige Setzstelle
oberhalb Bit 13, und für `warn` existiert **überhaupt keine** Setzstelle. Die Zuordnung ist damit
auch von der Geräteseite bestätigt.

## 6. Der Mechanismus

Identisch zum D, nur mit anderem Event-Logger und anderen Codes:

```
  <Bedingung>            ; meist: Messwert gegen Grenzwert, dann Zykluszähler gegen Trip-Zeit
  movs r0,#<Event-Code>  ; ggf. r1..r3 = Messwerte oder Kanalindex
  bl   0x080103e0        ; Event-Logger
  movw r0,#0x1650
  movt r0,#0x2010
  ldrb r1,[r0,#<Byte-Offset>]
  orr  r1,r1,#<Bit>      ; and/bic bei Entwarnung
  strb r1,[r0,#<Byte-Offset>]
```

### Warum die Bitliste vollständig ist

Die Basisadresse wird **ausschließlich** per `movw`/`movt`-Paar geladen — es gibt im gesamten
Image kein Literal-Pool-Wort mit dem Wert `0x2010164c`/`50`/`54`/`58`/`5c`. Ein Rohscan über die
Bytes nach `movw rD,#0x1650` + `movt rD,#0x2010` findet für alle 13 möglichen Zielregister
**26 Stellen in 1211** — exakt die 26, die auch Ghidra kennt. Zusätzlich wurde das Idiom
`ldrb/orr|bic/strb` roh über das ganze Image gesucht (55 Treffer, alle Basisregister): darunter
keine weitere Stelle mit Offset 0–11 auf diesen Block.

Die Aufzählung ist damit nicht „alles, was Ghidra gefunden hat", sondern abschließend.

Zwei Feinheiten, die beim Nachbauen Treffer kosten:

- Das abschließende `strb` fehlt an einigen Stellen, weil der Compiler stattdessen auf einen
  gemeinsamen Speicherbefehl springt (`orr r1,r1,#0x1 ; b …`). Wer `strb` erzwingt, verliert unter
  anderem err1 Bits 0–2.
- Eine einzige Stelle lädt die Basis **nach** dem `orr` (err1 Bit 21 setzen, `0x08002314`:
  `orr r0,r0,#0x20 ; movw r1,#0x1650 ; movt r1,#0x2010 ; strb r0,[r1,#0x2]`). Die findet der
  Rohscanner nicht; sie kam aus Ghidra. Erkennungsmerkmal: ein Bit taucht in der Scannerausgabe
  nur als `clear` auf.

## 7. Die Bits

Alle drei Archivversionen (0119, 1193, 1211) haben **dieselbe Bitbelegung** — nur die Adressen der
Setzstellen und zwei Event-Codes wandern (0119: 0x42/0x47 statt 0x43/0x48).

### err2 — 36102 / 36103

| Bit | Event | Auslöser | Konfidenz |
|---|---|---|---|
| **5** | 0x43 | Schutzabschaltung; setzt zusätzlich die Flags `0x201004dd` und `0x20120511`, loggt drei Messwerte mit | mittel |
| **6** | 0x35 | Messwert **> 568,0** (`0x440E0000`, float) — Zwischenkreis-/DC-Überspannung; zwei Setzstellen (`0x08004db6`, `0x08004e1a`) | mittel |
| **7** | 0x15 | Zustandsautomat, Kanalparameter 5; im Alternativzweig wird stattdessen Peripherie `0x40009444` Bit 6 gesetzt | niedrig |
| **8** | 0x31 / 0x32 *(nach* dem Setzen geloggt) | **Zwei Integratoren, siehe unten** | **hoch** |
| **9** | 0x29 | Vergleiche gegen 100,0 und ±500,0 (float) | niedrig |
| **11** | 0x36 | **Sammelbit „Grenzwert zu lange überschritten"**, 4 Setzstellen, siehe unten | hoch |
| **13** | 0x48 | Zykluszähler ≥ 20001 (`0x4E21`), loggt `[0x2010056c+0xC]` | mittel |

Bits 0–4, 10, 12, 14–31: **keine Setzstelle**. err2 kann rechnerisch nur Werte aus
{32, 64, 128, 256, 512, 2048, 8192} zusammensetzen.

#### Bit 8 — der Überlast-Integrator (`FUN_080149a8`, 0x08014a7e / 0x08014b12)

Ereignis 0x31: ein Akkumulator `_DAT_20101578` wird je Durchlauf verändert, abhängig vom Messwert
`_DAT_20121b8c` (Strom):

| Messwert | Schritt |
|---|---|
| > 7,85 | +60 |
| > 7,2 | +10 |
| > 6,6 | +1 |
| > 6,54 | −1 |
| > 3,27 | −240 |
| ≤ 3,27 | −1800 |

Bei < 0 wird auf 0 geklemmt, bei **> 600.000** wird err2 Bit 8 gesetzt und Event 0x31 mit
Strom, Spannung und Zählerstand geloggt. Das ist ein i²t-artiges Überlastintegral: kurze Spitzen
werden verkraftet, Dauerlast knapp über der Nennleistung läuft nach einer Weile auf.
(Nebenbefund: bei > 60.000 wird `DAT_201233a5` gesetzt — offenbar eine Vorwarnschwelle.)

Ereignis 0x32: nur in den Betriebsarten `0x18`/`0x04`. Liegt `_DAT_20121b94` **unter 207,0**,
läuft ein Zähler bis **201** hoch, dann wird dasselbe Bit 8 gesetzt. Netzunterspannung.

Bit 8 heißt also nicht „ein bestimmter Fehler", sondern **„Betriebsgrenze über längere Zeit
verletzt"** — Überlast oder Unterspannung.

#### Bit 11 — Sammelbit mit vier Kanälen

Alle vier Setzstellen haben dieselbe Form: eine Bedingung, ein Zykluszähler, eine Schwelle.

| Adresse | Schwelle | Event-Parameter (r1, r2) |
|---|---|---|
| `0x08007912` | 101 (`0x65`) | 0x0B, 0x0D |
| `0x0800926e` | 5001 (`0x1389`) | 0x03, 0x05 |
| `0x0800947a` | 10001 (`0x2711`) | 0x0B, 0x07 |
| `0x08009684` | 300.001 / 6.000.001 | Kanalbyte aus `[r5]` |

Strukturell dasselbe wie err2 Bit 11 beim D — nur zählt der D 3000 Zyklen, der A 101 / 5001 /
10001 / 300000. **Der Text „Limit exceeded for more than 3000 cycles" ist für den A also
zahlenmäßig falsch**, das Prinzip stimmt. Welcher Kanal ausgelöst hat, steht wie beim D nur im
Event-Log (Parameter zu Event 0x36), nicht im Modbus-Register.

### err1 — 36100 / 36101

Bits 0–4 und 7 werden alle in `FUN_0800e4a0` gesetzt. Die Funktion ist der **Netzschutz**: sie
vergleicht `_DAT_20121b44` gegen 47,0 / 53,0 / 57,0 / 63,0 (Frequenzfenster 50 und 60 Hz) und
`_DAT_20100640` gegen 170,0 (Spannung), betreibt ein ±2,0-Hz-Fenster um `_DAT_20100690` mit
2000-Zyklen-Zähler (Event 0x30) und arbeitet sonst durchgehend mit Grenzwerten aus einem
Konfigurationsstruct plus Auslösezähler.

| Bit | Event | Auslöser | Konfidenz |
|---|---|---|---|
| **0** | 0x2c | Frequenz **über** Grenzwert `_DAT_20120410` länger als der Zähler in `[r7+0x0c]` | mittel |
| **1** | 0x2c | Frequenz **unter** Grenzwert `_DAT_2012040c` länger als `[r7+0x1c]` | mittel |
| **2** | 0x2a | Grenzwertverletzung, Zähler gegen `[r7+0x24]`, loggt zwei Messwerte | mittel |
| **3** | 0x2a | dito, Zähler gegen `[r7+0x2c]` | mittel |
| **4** | 0x2d | `|_DAT_20100570|` **> 430,0** (`0x43D70000`) | mittel |
| **7** | 0x2e | Zähler ≥ 4, loggt drei Messwerte | niedrig |
| **20** | 0x1a | gesetzt `0x08002358`, gelöscht `0x08002344` (`and #0xef`) — beides in `FUN_080022a0`, der Kommunikationsüberwachung | mittel |
| **21** | 0x0c beim Löschen | gesetzt `0x08002314` (`orr #0x20`), gelöscht `0x080022e4` (`bic #0x20`), dieselbe Funktion | mittel |

Bits 5, 6, 8–19, 22–31: keine Setzstelle.

Die Parallele zum D ist auffällig: dort sind err1 Bit 20 = CAN-Kommunikationsfehler und Bit 21 =
BMS meldet Fehler. Beim A liegen dieselben zwei Bits in einer Kommunikations-Überwachungsfunktion.
**Nicht verifiziert** — die Vermutung ist naheliegend, aber ein Gerätetest fehlt.

### warn — 36000 / 36001

**Keine einzige Setzstelle im gesamten Image.** Das Wort wird gefüllt, übertragen und ausgegeben,
aber nie beschrieben. Beim A ist `warn_code` damit dauerhaft 0 — anders als beim D, wo es
zumindest theoretisch beschreibbar wäre.

## 8. Quittierung

Zwei Stellen löschen das **untere Halbwort** von err2 und lassen das obere stehen:

```
0x0800961c   ldrh r1,[r0,#0x6] ; lsls r1,r1,#0x10 ; str r1,[r0,#0x4]   ; err2 = (err2 & 0xFFFF0000)
0x08005630   movw r6,#0xffff   ; bics r3,r6       ; str r3,[r2,#0x4]   ; err2 &= ~0xFFFF
```

Da oberhalb Bit 13 ohnehin nichts gesetzt wird, ist „unteres Halbwort löschen" beim A faktisch
„err2 komplett löschen".

> Diese Stellen sahen zunächst nach der Erklärung für das Rückfallen der gemeldeten Werte auf 0
> nach 7–8 Sekunden aus. Das war eine Fehlspur — dort lag nie ein Fehler an, und der Rücksprung
> auf 0 war schlicht der nächste Lesevorgang ohne Antwortversatz (Abschnitt 9.3). Der
> Quittierungspfad selbst ist davon unberührt und gilt weiter.

## 9. Die gemeldeten Werte waren gar keine Fehlercodes

**Aufgelöst.** 36103 hat nie einen Fehler angezeigt — die Integration hat verrutschte
Modbus-Antworten gelesen. Der Weg dahin steht in 9.1 bis 9.4, weil die Firmwareanalyse den
entscheidenden Ausschluss geliefert hat.

Aus der Anwendermeldung (36103, jeweils 7–8 s anstehend):

| Wert | Hex | Bits |
|---|---|---|
| 2328 | `0x0918` | 3, 4, 8, 11 |
| 2337 | `0x0921` | 0, 5, 8, 11 |
| 2349 | `0x092D` | 0, 2, 3, 5, 8, 11 |
| 2350 | `0x092E` | 1, 2, 3, 5, 8, 11 |
| 2380 | `0x094C` | 2, 3, 6, 8, 11 |

**Bits 0–4 haben in err2 keine Setzstelle** — in keiner der drei archivierten Versionen. Sie
existieren nur in err1. err2 kann diese Werte nicht annehmen.

Genau dieser Ausschluss war der Hebel: solange man die Bits für echt hält, sucht man nach einem
Gerätefehler. Sobald die Firmware sagt, dass es diese Bits nicht gibt, ist klar, dass der Wert
nicht aus err2 stammt — und die Suche verlagert sich auf den Weg dorthin.

### 9.1 Die Firmware des Geräts ist bekannt — und liegt im Archiv

Der kombinierte Versionssensor des Geräts zeigt `V150.9.1193.0.1105`. Aufgelöst (zur Formatierung
siehe Abschnitt 10, Punkt 5):

| Komponente | Wert | Archiv |
|---|---|---|
| Control/EMS | 150.9 | `1509_control_VNSA-0_ems_app_1509_0908_093804.bin` ✔ |
| Wechselrichter | **119.3** | `1193_micro_VNSA-0_VA_inv_app_1193_0512_092355.bin` ✔ |
| MPPT | 0 | nicht vorhanden |
| BMS | 110.5 | `1105_bms_VNSA-0_VA50A_APP_V1105_ota_144658.bin` ✔ |

**Die naheliegende Erklärung „Firmware neuer als das Archiv" ist damit erledigt.** Das Gerät fährt
1193, und dessen Bitbelegung ist hier vollständig ausgewertet — identisch zu 0119 und 1211.

Damit war auch der zweite Verdacht hinfällig, der **Control 1509** könne das Struct anders füllen.
Er war nicht unbegründet — 1509 hat den Bereich umgebaut, `work_mode` (30010) liegt in v150 noch
auf dem eigenständigen Global `0x20000144`, in 1509 dagegen auf `0x200176D4`, also **innerhalb**
des Structs bei +0x38. Die Fehlerfelder selbst sind aber geschlossen um +0x2800 gewandert
(36000/36100/36102 → `0x200176A0/A4/A8`, 30000–30007 und 32300/32302 mit demselben Delta), und
der Debug-Log löst den Fall ohne diese Hypothese.

### 9.2 Der Debug-Log

Der meldende Anwender hat einen HA-Debug-Log vom 17. bis 21. September 2026 geliefert
(**nicht im Projekt abgelegt** — personenbezogene Netz- und Verbrauchsdaten). Ausgewertet wurde er
auf drei Fragen: Zeitkorrelation, Wertebereich, Verbindungszustand.

**Zeitkorrelation.** Jedem der fünf Ereignisse geht 3–4 Sekunden vorher ein gescheiterter
Lesezugriff auf **Register 36100** voraus — das erste Register genau dieses Blocks. Vier von fünf
haben zusätzlich einen pymodbus-Transaktions-ID-Versatz:

```
2026-09-20 22:21:27.296  Failed to read register 36100 (0x8D04) after 1 attempt(s)
2026-09-20 22:21:28.401  ERROR: request ask for transaction_id=1487 but got id=1486, Skipping.
2026-09-20 22:21:31      fault_status_2_low = 2350
```

| Ereignis | TID-Versatz | Lesefehler 36100 |
|---|---|---|
| 2328 @ 21.09. 11:35:35 | −3 s | −3 s |
| 2380 @ 21.09. 13:50:53 | – | −3 s |
| 2350 @ 20.09. 22:21:31 | −3 s | −3 s |
| 2349 @ 20.09. 22:06:11 | −3 s | −3 s |
| 2337 @ 20.09. 20:21:34 | −3 s | −4 s |

TID-Versätze treten im Log 1507-mal in 104,8 h auf, also einer alle 250 s. Die Zufallswahrschein-
lichkeit für einen Treffer im ±15-s-Fenster liegt bei 11,6 %; vier von fünf Treffern, und jedes
Mal bei exakt −3 s statt gestreut, schließen Zufall aus.

**Wertebereich.** Im Debug-Fenster (21.09., 21:23–21:28) liest `ac_voltage` (32200) Werte von
**2322 bis 2333**. Die fünf „Fehlerwerte" 2328/2337/2349/2350/2380 liegen in genau diesem Band,
**2328 ist buchstäblich einer der gemessenen `ac_voltage`-Werte**. Kein anderes gepolltes Register
liegt in diesem Bereich.

**Verbindungszustand.** ~13.000 Integrations-Warnungen und 130–1200 Fehler pro Tag; 1209-mal
„Modbus client not connected, attempting reconnect before write to register 42021". Eine
0-Einspeisungs-Automation schreibt `set_discharge_power` (42021) im Sekundentakt, während parallel
gepollt wird. Der Modbus-Stack des Geräts kommt nicht hinterher — bekanntes Verhalten, siehe die
`VNSD_FW_Bug_*`-Dokumente unter `Modbus_RS485_TCP/`.

### 9.3 Der Ablauf

1. Der Blockleser für 36100–36103 läuft in einen Timeout; die Antwort des Geräts bleibt unterwegs.
2. Die Integration fällt auf **Einzellesungen** der vier Register zurück.
3. Die verspätete Antwort trifft ein, die Zuordnung Anfrage→Antwort ist um eine Position versetzt.
   Weil `ac_voltage` und `fault_status_2_low` beide Ein-Register-Lesungen sind, passt die Länge;
   nur die Transaktions-ID würde es auffangen — und genau die schlägt fehl.
4. `fault_status_2_low` übernimmt den Wert einer fremden Einzellesung.
5. Der nächste saubere Zyklus 7–8 s später stellt 0 wieder her. Das sah nach Selbstquittierung aus,
   war aber nur der erste Lesevorgang ohne Versatz.

`coordinator.py` kennt die Gefahr bereits — vor dem Fallback läuft `_async_recover_half_open` mit
dem Kommentar „every fallback read below starts on the same suspect socket". Das greift nur bei
halboffenen Sockets, nicht bei einem, in dem noch eine gültige, verspätete Antwort steckt.

### 9.4 Die MPPT-Werte

Vermutlich derselbe Mechanismus, **nicht belegt** — zu den Zeitpunkten (20./21.09. nachmittags)
lief noch kein Debug-Log. Dafür spricht: `mppt_version` steht auf 0, das Gerät hat gar keine
MPPT-Stufe, die etwas melden könnte; und `set_discharge_power` (42021) wird von derselben
Automation laufend mit Werten beschrieben, die abends bei 700–770 W liegen — mittags sind
1098/1100/1367 W genau der erwartete Bereich.

### 9.5 Was das für die Bitlisten heißt

Die Deutung aus Abschnitt 7 bleibt gültig, sie war hier nur nicht anwendbar: es lag kein Fehler an.
Der Wert dieser Analyse liegt in diesem Fall im **Ausschluss** — ohne die vollständige Bitliste
hätte man die Werte weiter als Hardware-Schutzauslösungen gedeutet und am falschen Ende gesucht.

## 10. Folgen für die Integration

Stand 21.09.2026 sind die Punkte 1, 2, 5 und 7 im Projekt *Marstek Modbus Suite* umgesetzt;
3 und 4 sind reine Feststellungen, 6 ist offen.

1. **`a.yaml` darf die D-Bittexte nicht führen.** *(umgesetzt)* „Hardware protection latch A/B/C" existiert im
   A-Code nicht, und die 3000 Zyklen sind beim A 101/5001/10001/300000. Entweder die
   `bit_descriptions` für `fault_status*` in `a.yaml` entfernen oder durch die Tabellen aus
   Abschnitt 7 ersetzen.
2. **`warn_code` (36000/36001) ist beim A tot.** *(umgesetzt: kein Bit gelistet)* Kann als Entität deaktiviert bleiben.
3. **Das High-Word von err1 und err2 ist beim A fast leer** — err2 komplett, err1 nur Bits 20/21.
4. Die Wortreihenfolge (36102 = High, 36103 = Low) gilt beim A wie beim D; die Descriptor-Tabelle
   und der FC03-Serializer sind identisch.
5. **Der kombinierte Versionssensor formatierte nur den EMS-Teil.** *(umgesetzt)* `MarstekVersionSensor.
   calculate_value` (`sensor.py`) wendet die Zehntel-Regel „vierstellig → `x // 10` . `x % 10`"
   ausschließlich auf `ems` an; `vms`, `mppt` und `bms` werden roh angehängt. Marstek nummeriert
   aber **alle** Komponenten gleich, wie das Firmware-Archiv zeigt:

   | Komponente | Archivierte Werte | gemeint |
   |---|---|---|
   | Control | 148, **1487**, 149, 150, **1508**, **1509** | 148, 148.7, 149, 150, 150.8, 150.9 |
   | Wechselrichter (A) | 119, **1193**, **1211** | 119, 119.3, 121.1 |
   | BMS (A) | 109, **1105** | 109, 110.5 |
   | Wechselrichter (D) | 115, 116 | unverändert |
   | BMS (D) | **1177**, 118 | 117.7, 118 |

   Deshalb steht auf dem gemeldeten Gerät `V150.9.1193.0.1105` statt „150.9 / 119.3 / – / 110.5".
   Zwei Folgefehler: die Punkte trennen nicht mehr eindeutig (`150.9.119.3.110.5` wäre nicht besser
   lesbar, hier braucht es ein anderes Trennzeichen), und der MPPT-Teil steht beim A immer auf 0,
   weil das Gerät keine MPPT-Stufe hat — der Modus `ems_vms_mppt_bms` produziert dort eine
   sinnlose Null. Gehört ins Modbus-Suite-Projekt, hier nur festgehalten.
6. **Nach einem Lese-Timeout hart neu verbinden** *(offen)*, nicht nur auf halboffen prüfen. Sonst läuft der
   gesamte Einzellese-Fallback auf einem Socket, in dem noch eine verspätete Antwort steckt, und
   liefert Werte fremder Register (Abschnitt 9.3). Das ist die Ursache des Anlassfalls.
7. **Plausibilitätsprüfung auf den Fehlerregistern.** *(als Kommentar in `a.yaml` hinterlegt, Prüfung selbst offen)* Aus dieser Analyse ist bekannt, welche Bits
   es überhaupt gibt — beim A `err1 = 0–4, 7, 20, 21` und `err2 = 5, 6, 7, 8, 9, 11, 13`, beim D
   die Liste aus `VNSD_Fehlercodes_Micro_FW_Analyse_de.md`. Ein Wert mit anderen Bits ist per Definition
   eine Fehllesung und sollte verworfen statt angezeigt werden. Im Anlassfall hätte diese eine
   Prüfung alle fünf Falschmeldungen abgefangen.

## 11. Offen

1. **Control 1509 analysieren** — nicht mehr dringend, seit der Anlassfall aufgeklärt ist, aber
   offen: welche Funktion das Struct `0x2017769C` aus dem 48-Byte-Payload des Wechselrichters
   füllt und ob sie dieselben Offsets liest wie der v150-Control. Die Descriptor-Zeiger sind
   geschlossen um +0x2800 gewandert, `work_mode` ist aber ins Struct gerutscht (Abschnitt 9.1) —
   die Zuordnung „36102/36103 = err2" ist deshalb streng genommen nur für **v150** bewiesen
   (Abschnitt 5). Das Image liegt im Archiv, ist aber noch nicht in Ghidra.
2. **`mppt_error` / `mppt_warning` (37023/37024)** — der Anlassfall ist mit hoher Wahrscheinlichkeit
   dieselbe Fehllesung (Abschnitt 9.4), die Semantik der Register bleibt davon unabhängig offen.
   Die Descriptor-Tabelle des A führt sie
   wie beim D auf Inverter-Struct +0x02 und +0x06 (`0x2001776E`/`0x20017772` bei 1509), gefüllt
   über `Modbus_StoreRegisterSlot` aus den CAN-Slots. Eine Bit-zu-Text-Tabelle existiert in keiner
   Firmware, für kein Modell. Ergänzender Befund aus den D-Logs:
   `Scan_Logs/battery_disconnect_and_then_power_cycle_without_battery.csv` (lokal, Rohdaten) zeigt 37024 in einem
   einzelnen Sample auf **1345 = 0x0541** bei sonst durchgehend 0 und allen Fehlerregistern auf 0
   — dieselbe Wertefamilie wie die 1367 des A-Anwenders. Ein kurzer Ausschlag von
   `mppt_warning` tritt also auch am D auf, ohne begleitenden Fehler.
   **Neu dazu:** auf dem gemeldeten Gerät steht `mppt_version` (30205) auf **0** — es gibt dort
   keine MPPT-Stufe. Trotzdem liefern 37023/37024 von Null verschiedene Werte. Was die CAN-Slots
   beim A ohne MPPT-Controller in den Inverter-Struct schreiben, ist offen; ein stehengebliebener
   oder fremdbelegter Slot ist die naheliegendste Vermutung.
3. **Event-Codes** — der Logger `0x080103e0` schreibt Code plus bis zu drei Parameter in einen
   Ringpuffer. Wo der landet und ob er auslesbar ist, wurde nicht verfolgt. Das wäre der Weg zu
   den Kanalindizes hinter err2 Bit 11.
4. **Fehlender Flash-Tail.** Der Debug-Print bei `0x080169c8` liest die drei Wörter per
   `ldm.w r0,{r1,r2,r3}` und lädt einen Formatstring von `0x08024419` — das liegt hinter dem
   Image-Ende `0x08023BFF`. Wie beim D fehlt der Tail, es gibt also keine Klartextnamen aus der
   Firmware.
5. **Internes Fehlerwort `0x201005b0`.** `FUN_08020360` setzt dort systematisch die Bits 0–15 und
   20–25 (22 Setzstellen), `FUN_08014c68` löscht es. Es wird **nicht** in die Telemetrie kopiert
   und ist über Modbus nicht sichtbar. Vermutlich die Schutzabschaltungs-Erfassung der
   Leistungsstufe. Ungeprüft, aber der naheliegendste nächste Brocken, falls doch noch ein
   Fehlerwort mit dicht belegten unteren Bits gebraucht wird.

---

## Anhang: Werkzeuge

Der Rohscanner, der die Bitlisten ohne Ghidra aus einem Image zieht, steht als
`Scripts/vnsa_micro_error_bits.py` (lokal, nicht im Repo). Er sucht `movw`/`movt`-Paare auf eine gegebene Basisadresse,
dekodiert das folgende `ldrb` + `orr|bic|and #imm`, löst den Thumb-`ModImm`-Direktwert auf und
ordnet Byte-Offset plus Maske einer Wortnummer zu; `bic`/`and` werden als Entwarnung gewertet.
Der Event-Code wird über den nächstgelegenen vorangehenden `bl` gesucht, damit für Stellen ohne
Logger-Aufruf nichts Falsches gemeldet wird.

```
python3 Scripts/vnsa_micro_error_bits.py <firmware.bin>              # Bitliste
python3 Scripts/vnsa_micro_error_bits.py <firmware.bin> --find-bases # Basisadresse suchen
```

`--find-bases` meldet in allen drei A-Images `0x20101650` (16 Stellen) und `0x201005b0`
(12 Stellen, das interne Wort aus Abschnitt 11.5). Auf die D-Micro-Images angewandt liefert er
**0 Treffer** — die nutzen das Idiom nicht, was den Befund aus Abschnitt 2 ein weiteres Mal
bestätigt.
