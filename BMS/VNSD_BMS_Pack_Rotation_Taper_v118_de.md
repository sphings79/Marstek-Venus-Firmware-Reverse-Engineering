# Venus D — BMS 118: Pack-Rotation, Untergrenze und Lade-Taper

> 🇬🇧 English version: [VNSD_BMS_Pack_Rotation_Taper_v118_de.md](VNSD_BMS_Pack_Rotation_Taper_v118_de.md)

> **Hinweis (2026-09-30):** Flash-Adressen in diesem Dokument stammen aus der alten Analyse mit falscher Basisadresse (0x2800 zu niedrig). Ladegrenzen und Stromtabelle mit korrekten Adressen: [VNSD_Ladegrenzen_Ladeleistung_de.md](VNSD_Ladegrenzen_Ladeleistung_de.md), Gesamtanalyse: [VNSD_BMS_FW_v118_Vollanalyse_de.md](VNSD_BMS_FW_v118_Vollanalyse_de.md).

**Stand:** 2026-09-24
**Gilt für:** Venus D (VNSD-0), BMS-Firmware 118 (`BMS_118_20260119100535e43806957.bin`).
Das Venus-A-BMS hat laut Byte-Suche dieselbe 10-%-Zielfunktion, die Entlade- und
Untergrenzen-Logik aber nur hier — Details in `Methodik_und_Meta/Geraetevergleich_VNSD_VNSA_VNSE3_de.md` §3f.
**Quellen:** Ghidra-Projekt `/VNSD/BMS/BMS_118_…`, Dekompilat-Export `Dekompilate/VNSD_BMS_118/` (lokal, nicht im Repo),
Felddaten aus Home Assistant (Carport-Gerät, 7 Packs à 2,56 kWh) vom 19.–22.09.2026.

Vorgänger dieser Analyse: `VNSD_BMS_FW_Analyse_v118_de.md` §11 (Limit-Aggregation, Handoff). Was dort
offen war, ist hier geklärt oder korrigiert (§10).

---

## 1. Kernaussagen

1. **Es lädt und entlädt immer genau ein Pack.** Die Auswahl trifft das Master-BMS, nicht
   der Control und nicht der Wechselrichter.
2. **Laden:** Ziel ist das nächste Zehnerraster über dem aktuellen SOC (57 % → 60 %, 60 % → 70 %).
   Ist es erreicht, übernimmt der Pack mit dem **niedrigsten** SOC. Bei Gleichstand gewinnt der
   niedrigere Index.
3. **Entladen:** Die Schwelle liegt 10 Prozentpunkte unter dem SOC **zum Zeitpunkt der letzten
   Neuberechnung**. Sie ist kein Zehnerraster und nie kleiner als `100 − DoD` (Standard 10 %).
   Unterschreitet der Pack sie, übernimmt der **vollste** Pack.
4. **Ziel und Schwelle veralten.** Neu berechnet werden sie nur beim Pack-Wechsel oder wenn der
   aktive Pack sein Ziel erreicht und trotzdem der extremste bleibt. Endet eine Ladung, bevor
   gewechselt wurde, gelten die alten Werte weiter. Genau das erklärt die Entladung
   von 28,7 auf 12,0 % ohne Zwischenstopp am 20.09. (§5).
5. **Taper ist Konstantstrom je SOC-Zone,** nicht Konstantleistung. Gemessen am 22.09.:
   90 → 95 % ≈ 23,2 A, 95 → 100 % ≈ 9,1 A. Die Tabelle selbst liegt nicht im Image (§8).
6. Unter 11 % SOC wechselt die Logik in eigene Modi. Ist dann kein Pack mehr voller als der
   aktive, geht das BMS in den Pack-Modus **Shutdown** (§7).

---

## 2. Zeitbasis

Die Master-Logik läuft in einer eigenen FreeRTOS-Task (Code ab `0x0800C24C`, von Ghidra nicht
als Funktion erkannt, per Rohdekodierung der BL-Befehle ermittelt):

```
loop:
    FUN_0800932c();
    FUN_080087ac();
    BMS_Role_Dispatcher();        // → Master_MultiPack_Tick → Pack-Auswahl
    vTaskDelay(100);
```

SysTick wird in `System_Init_Main` mit `SystemCoreClock / 1000 − 1` geladen, also 1 kHz.
`LIB_vTaskDelayWithAssert` ist ein unverändertes `vTaskDelay` (Ticks, keine ms-Umrechnung).

**Ein Tick der Pack-Logik ≈ 100 ms** (plus Laufzeit der drei Aufrufe, also eher etwas mehr).
Alle Zählerwerte unten sind in diese Einheit umgerechnet.

---

## 3. Datenlage

### 3.1 Pack-Array

Basis `0x200041B6`, Stride `0x60`, Index 0 = eigener Pack. Die Feldnamen stammen aus
`BMS_Debug_PerPack_Detail_Print` (Klartext-Ausgabe im Image):

| Offset | Feld | Einheit |
|---|---|---|
| +0x00 | Addr | — |
| +0x01 | Online | 0/1 |
| +0x0E | Bat Soc | 0,1 % |
| +0x10 | Bit 5 = Chg MOS, Bit 4 = Dsg MOS | — |
| +0x48 | Min Cell | mV |
| +0x4C | Bat Curr | 0,1 A, i16, **Laden positiv** |
| +0x50 / +0x52 | max. Lade- / Entladestrom | 0,1 A |
| +0x56 / +0x58 | Protect1 / Protect2 | Bitmaske |

### 3.2 SRAM-Variablen

| Adresse | Bedeutung |
|---|---|
| `0x20002A0C` | Master-Laufzustand (0 → 1 → 2) |
| `0x20002A11` | aktiver Pack |
| `0x20002A12` | Ziel-Pack des Handoffs |
| `0x20002A17` | **Soll-Maske Chg-MOS** (welche Packs laden dürfen) |
| `0x20002A18` | **Soll-Maske Dsg-MOS** (welche Packs entladen dürfen) |
| `0x20002A14` | Ausschlussmaske: offline oder Protect1 `0x1F0` / Protect2 `0x9000` |
| `0x20002A1E` | Wiederherstellungsladung aktiv (§7.4), `+1` Maske, `+2` Min-Zellspannung |
| `0x20002A22` | **Ladeziel** in % (Zehnerraster) |
| `0x20002A23` | **Entladeschwelle** in % |
| `0x20002A24` | Auswahlmodus 0–3 (§7) |
| `0x20002A44` | Handoff angefordert |
| `0x20002A54` / `0x20002A56` | Zähler Lade- / Entladewechsel |
| `0x20002A58` | Kopie des DoD, um Änderungen zu erkennen |
| `0x200028CA` | Pack-Modus (0 Idle, 1 Transitional, 2 Active, 3 Shutdown, 4 Error) |
| `0x200028DB` | Anzahl weiterer Packs (Packs − 1) |
| `0x200028DC` | Online-Maske |
| `0x200028E0` | **DoD** in %, Standard `0x5A` = 90 |

`0x200028E0` ist die Entladetiefe. Es ist kein „SOC-Balance-Parameter“, wie es in
`VNSD_BMS_FW_Analyse_v118_de.md` §4.8/§11.5 hieß. Belege:

- `CAN_Load_Or_Init_Config_Value` liest den Wert aus dem Flash, Standard 90.
- `CAN_TX_PF2_Capacity` sendet ihn an den Control.
- Der Control gibt ihn als `pgn_1802_info.dischrg_dod` aus.

Gesetzt wird er über CAN (`0x100201AA`, außerdem Byte 13 im Frame `0x1801FFAA`) und RS485.

Die Backup-Reserve des Control (EEPROM `0x201`, App-Einstellung 30–88 %, Standard
Reserve 12 %) ist ein **anderer** Parameter. Siehe
`Modbus_RS485_TCP/VNSD_Register_Persistenz_RAM_vs_EEPROM_v150_de.md`, Nachtrag 2026-08-22.

---

## 4. Aufrufkette

```
Master_MultiPack_Tick
 └─ Master_RunState_Dispatcher                      (0x20002A0C)
     0: FUN_080040e8                                  → 1
     1: Master_RunState_InitSelect                    → 2
          ├─ Protect1-Bit-0-Pack vorhanden? → Wiederherstellung anfordern (§7.4)
          └─ sonst PackSelect_Startup_Closest50       (§7.5)
     2: 0x20002A1E == 0 ?
          ├─ PackHandoff_Tick
          │    ├─ PackSelect_ModeDispatch             (Modus 0–3, §5–§7)
          │    ├─ PackHandoff_StateMachine            (§6)
          │    ├─ PackSelect_MosMismatch_Fallback     (§7.6)
          │    └─ PackSelect_ChargeEnable_Maintain    (§7.6)
          ├─ PackSelect_Tick                          (Fehlerpfad, §7.6)
          └─ PackRecovery_Request_Check
        sonst PackRecovery_LowCell_Charge_SM          (§7.4)
```

---

## 5. Modus 0 — normale Rotation

`PackSelect_ModeDispatch` ruft im Modus 0 in jedem Tick beide Funktionen auf, die fürs Laden
und die fürs Entladen. Welche greift, entscheidet das Vorzeichen des Pack-Stroms.

### 5.1 Laden — `PackSelect_Charge_StepRotate` (0x08004CD0)

```c
if (soc% < ziel || strom < 1,1 A)          // Ziel nicht erreicht oder lädt nicht
    zaehler = 0;
else if (++zaehler > 50 || soc > 99,9 %) {  // > 5 s am Ziel mit Ladestrom
    kandidat = Pack mit dem niedrigsten SOC (online, strikt kleiner → niedrigerer Index gewinnt);
    if (kandidat == aktiv) {                 // bleibt der leerste
        ziel     = PackSelect_ChargeStepTarget(aktiv);
        schwelle = PackSelect_DischargeStepThreshold(aktiv);
    } else
        Handoff zu kandidat;
}
```

`PackSelect_ChargeStepTarget` (0x08006CFC): `((soc% + 10) / 10) × 10`, höchstens 100.
`soc%` ist dabei der ganzzahlige Prozentwert (57,6 % → 57).

**Korrektur zur Kurzfassung vom 24.09.:** Der Zähler läuft nur, solange der Pack mit
**mindestens** 1,1 A lädt, nicht bei weniger als 1,1 A. Reißt die Ladung am Ziel innerhalb
der 5 s ab, wird nicht gewechselt.

An 90 % passiert in der Auswahl nichts Besonderes. Die Stufen bei 90 und 95 % im Feld
stammen aus der Stromtabelle (§8).

### 5.2 Entladen — `PackSelect_Discharge_StepRotate` (0x08003BDC)

```c
if (dod geändert) zaehler läuft sofort;
if ((soc% > schwelle || strom > −1,1 A) && dod unverändert)
    zaehler = 0;
else if (++zaehler > 50) {                  // > 5 s an/unter der Schwelle mit Entladestrom
    kandidat = Pack mit dem höchsten SOC;
    if (kandidat == aktiv) {
        ziel     = PackSelect_ChargeStepTarget(aktiv);
        schwelle = PackSelect_DischargeStepThreshold(aktiv);
    } else
        Handoff zu kandidat;
}
```

`PackSelect_DischargeStepThreshold` (0x08006BBC), mit `L = 100 − DoD`:

| Lage | Ergebnis |
|---|---|
| `soc% − 10 ≥ L` | `soc% − 10` |
| `soc% − 10 < L < soc%` | `L` |
| `soc% ≤ L` | `L − 1` |

Gewechselt wird, sobald `soc% ≤ schwelle`, also bei einer Schwelle von 20 ab 20,9 %.

### 5.3 Veraltetes Ziel und veraltete Schwelle — Feldbeleg

Beide Werte werden **nur** an drei Stellen neu gesetzt: am Ende eines Handoffs (§6, Schritt 5),
im Startpfad (§7.5) und in den beiden `kandidat == aktiv`-Zweigen oben. Die HA-Historie
(Omnibattery-SOC je Pack) zeigt genau dieses Verhalten:

| Zeit | Beobachtung | Erklärung aus dem Code |
|---|---|---|
| 19.09. 02:00–02:21 | P1 entlädt 30,6 → **20,9 %**, dann übernimmt ein anderer Pack | P1 bei ~30 % aktiviert → Schwelle 20 → Wechsel bei `soc% ≤ 20` |
| 19.09. 13:53–14:02 | P1 lädt 19,6 → 30,1 %, dann endet die Ladung | Ziel 20; bei 20,x ist P1 noch der leerste → Ziel 30, **Schwelle = Clamp(20) = 10**. Bei 30,1 % reißt der Strom ab, bevor 5 s um sind → **kein Wechsel**, P1 bleibt aktiv |
| 20.09. 23:05–23:42 | P1 entlädt **28,7 → 12,0 %** ohne Halt, P3/P4/P6 stehen bei 28,9 % | Schwelle ist noch 10 → Wechsel erst bei `soc% ≤ 10`. Bei 12,0 % hat Omnibattery abgeschaltet |
| 21.09. 07:37–08:32 | P1 lädt 12,0 → **30,1 %**, obwohl P2 mit 12,0 % gleich leer und P5/P7 mit ~18 % leerer sind | Ziel ist noch **30** vom 19.09. Erst bei 30,1 % wird gewechselt, zu P2 (12,1 %) |
| 21.09. 08:32–08:50 | P2 12 → 20, P7 17,7 → 20, P5 18,6 → 20, dann P2 20,7 → 30 | reguläre Rotation, Gleichstand bei 20,1 % → niedrigerer Index (P2) |

Die Zwangsentladung per RS485 am 20.09. spielt dabei keine Rolle. Das BMS kennt den Modus des
Control nicht, es sieht nur Strom und SOC.

**Was ohne Omnibattery passiert wäre:** P1 wäre bis 10,9 % gelaufen, dann hätte der vollste
Pack (P3, 28,9 %) übernommen. Unter 11 % hätte P1 nie mehr per Modus 0 übernommen (§7).

---

## 6. Handoff — `PackHandoff_StateMachine` (0x08003F40)

Die Sprungtabelle ist inzwischen aufgelöst, alle Schritte sind belegt:

| Schritt | Aktion | weiter, wenn | Dauer |
|---|---|---|---|
| 0 | Minimum-Bildung der Limits einschalten (`0x20002A0E = 0`) | 100 Ticks | ≈ 10 s |
| 1 | Chg-MOS-Soll leeren (`0x20002A17 = 0`) | kein Pack lädt mehr | MOS-Schaltzeit |
| 2 | Dsg-MOS-Soll `|= neu` — **alter und neuer Pack entladeseitig parallel** | Dsg-MOS des neuen Packs an | |
| 3 | Dsg-MOS-Soll `= neu` | genau ein Dsg-MOS an, und zwar der neue | |
| 4 | Chg-MOS-Soll `= neu` | neuer Pack lädt oder SOC > 98 % | |
| 5 | aktiv = neu; Modus, Ladeziel und Entladeschwelle neu berechnen | sofort | |

**Korrektur zu `VNSD_BMS_FW_Analyse_v118_de.md` §11.4:** `0x20002A18` ist die Soll-Maske der
**Entlade**-MOSFETs, nicht eine allgemeine Enable-Maske. Das Make-before-break betrifft nur die
Entladeseite, geladen wird während des Handoffs nie über zwei Packs.

Das deckt sich mit den gemessenen 15–30 s bei 0 W zwischen zwei Packs: 10 s Wartezeit plus
MOSFET-Schaltvorgänge plus Nachregelung des Wechselrichters.

---

## 7. Sonderfälle

### 7.1 Modusauswahl — `PackSelect_Mode_BySoc` (0x08006B34)

Früherer Ghidra-Name `OTA_Validate_FW_Version`, falsch. Die Funktion wird nur beim Aktivieren
eines Packs aufgerufen (Handoff-Schritt 5, Startpfad, Wiederherstellung):

| SOC des neuen Packs | Modus |
|---|---|
| `soc% < 9` und `0x20002A75 == 0` | 2 (SOC wird als Startwert gemerkt) |
| `soc% < 9` und `0x20002A75 != 0` | 3 |
| `9 ≤ soc% < 11` | 2 |
| `soc% ≥ 100` | 1 |
| sonst | 0 |

`0x20002A75` setzt nur `CAN_CMD_04_Handler` (Unterbefehl 4, ID `0x100401AA`). Von wem dieser
Befehl kommt, ist offen (§10).

### 7.2 Modus 2 — niedriger SOC (`PackSelect_Mode2_LowSoc`, 0x08003AF4)

- Lädt der Pack (Flag `0x20002A1D` Bit 0 gesetzt und ≥ 0,5 A): Ladeziel 9 bzw. 10 %,
  Entladeschwelle 20 %, danach normale Rotation (§5). Der Pack wird also sofort gegen den
  leersten getauscht.
- Sonst: nach 600 Ticks (≈ 60 s) **oder** 1,0 % SOC-Abfall seit Aktivierung wechselt er zum
  vollsten Pack, aber nur, wenn dieser über 10,0 % steht.

### 7.3 Modus 3 — leer (`PackSelect_Mode3_Empty`, 0x08003844)

- Entladen (< −1,0 A) 100 Ticks (≈ 10 s) lang, **oder** Leerlauf (|I| < 1,0 A) 1200 Ticks
  (≈ 120 s) lang → Wechsel zum vollsten Pack.
- **Ist der aktive Pack selbst der vollste, setzt die Funktion den Pack-Modus
  `0x200028CA = 3` (Shutdown).** Das ist das Verhalten des Geräts ganz unten.
- Laden (≥ 1,0 A) setzt die Zähler zurück und bei `0x20002A6D == 0` den Modus auf 0.

### 7.4 Wiederherstellungsladung — `PackRecovery_LowCell_Charge_SM` (0x0800DF0C)

Früherer Ghidra-Name `BMS_OTA_Upgrade_StateMachine`, falsch. Die Funktion hat nichts mit OTA
zu tun. Sie läuft statt der normalen Rotation, sobald ein Pack Protect1 Bit 0 meldet
(`Pack_Get_Protect1Bit0_Mask`, gesetzt über `PackRecovery_Request_Check`):

1. Alle Dsg-MOS aus.
2. Aus den betroffenen Packs den mit der **niedrigsten Min-Zellspannung** wählen und
   exklusiv laden (Chg-MOS an, Dsg-MOS aus).
3. Ab 30 Ticks mit ≥ 1,1 A zusätzlich Dsg-MOS freigeben.
4. Fertig, wenn `Min Cell > 3100 mV` **und** SOC > 14 % (im Schritt 4 > 13 %) **und** Bit 0
   gelöscht ist. Dann folgt der nächste betroffene Pack.
5. Sind alle durch: den Pack **am nächsten an 50 %** aktivieren, Modus/Ziel/Schwelle neu
   berechnen, zurück zur normalen Rotation.
6. 300 s ohne Ladestrom (< 1,1 A) oder 60 s ohne Umschaltung → Pack-Modus 3 (Shutdown).

**Protect1 Bit 0 ist Zell-Unterspannung.** `Cell_OVP_UVP_TempCompensated_Protection` setzt es
bei `Min Cell < 2500 mV` (kalt 2300 mV) und zwingt dabei den SOC des Packs auf 0; Bit 1 ist
Überspannung und zwingt den SOC auf 100. Details in `VNSD_BMS_FW_Analyse_v118_de.md` §4.6. In
`VNSD_BMS_FW_Analyse_v117.7_de.md` §5.2 waren Bit 0 und Bit 1 vertauscht, dort ist das korrigiert.

Dieselbe Funktion liegt bytegleich auch in 117.7 (`0x0800df0c`), ebenso `PackSelect_Mode_BySoc`
(`0x08006acc`).

### 7.5 Startpfad — `PackSelect_Startup_Closest50` (0x08004148)

Damit ist der offene Punkt `FUN_08004148` aus §11.7 erledigt:

1. warten, bis kein Pack mehr lädt;
2. unter den Online-Packs mit SOC > 20 % den mit dem kleinsten Abstand zu 50 % wählen, sonst
   Pack 0; nur dessen Dsg-MOS freigeben;
3. sobald er entlädt: bei SOC > 98 % fertig, **ohne** Ladefreigabe (ein fast voller Pack soll
   nicht geladen werden). Sonst Chg-MOS freigeben und warten, bis er lädt;
4. Timeout 600 Ticks (≈ 60 s) → Pack-Modus 4 (Error).

Die > 98-%-Prüfung ist also kein Kalibrierpfad, sondern verhindert die Ladefreigabe für einen
vollen Pack beim Start.

### 7.6 Fehler- und Rückfallpfade

| Funktion | Auslöser | Wirkung |
|---|---|---|
| `PackSelect_ChargeEnable_Maintain` (0x0800352C) | aktiver Pack voll (`0x20002A6F` oder 100,0 %) | Chg-MOS-Soll löschen; bei Soll/Ist-Abweichung > 600 Ticks → Pack-Modus 4 |
| `PackSelect_MosMismatch_Fallback` (0x08003708) | Soll/Ist-Maske der MOS weicht > 300 Ticks ab | anderen Pack mit `soc% > L` als Ziel; nach 3 Versuchen Pack-Modus 4 |
| `PackFault_Build_ExcludeMask` (0x08004924) | offline oder Protect1 `0x1F0` / Protect2 `0x9000` | Ausschlussmaske `0x20002A14` |
| `PackSelect_Next_AboveSocThreshold` (0x080049D8) | Ausschlussmaske ≠ 0 | letzter fehlerfreier Pack mit `soc% > L` wird Ziel |
| `PackSelect_FaultPath_AllOff` (0x080034AC) | Protect1 `0xE0C` / Protect2 `0x2200` | alle MOS-Soll-Masken 0; ist der Fehler weg → Pack-Modus 4 |
| `Master_Check_AllPacksOnline` (0x0800364C) | Online-Anzahl ≠ Pack-Anzahl | `0x20002A0F = 1` (geht in Protect2 Bit 15 ein) |

---

### 7.7 Eigene MOSFETs, Vorladen und Start

**Start — `Master_Startup_WaitForEnable` (0x080040E8, Master-Laufzustand 0):**

- Lädt noch ein Pack, wird nur die Chg-MOS-Soll-Maske gelöscht.
- Sonst gibt der Master die **Entlade-MOSFETs aller Packs** frei (`0x20002A18` = alle Bits) und
  wartet, bis `0x20002A72` = 1 ist und kein Error-Lock (`0x20002A71`) besteht.
- `0x20002A72` setzt nur der CAN-Frame `0x1801FFAA` mit Byte 11 = 1 (`CAN_RX_Handler`). Der
  Master startet die Pack-Auswahl also erst nach einer Freigabe von außen. Bis dahin hängen alle
  Packs entladeseitig parallel.

**Treiber — `OwnPack_MOSFET_Driver` (0x080043C4), läuft in jedem Master-Tick:**

| `HAL_GPIO_Control_Switch` | Pin | Funktion |
|---|---|---|
| 1 / 2 | GPIOC Pin 5 an / aus | **Lade-MOSFET** (Pack-Struct +0x10 Bit 5) |
| 3 / 4 | GPIOB Pin 0 an / aus | **Entlade-MOSFET** (+0x10 Bit 4) |
| 5 / 6 | GPIOB Pin 2 an / aus | **Vorlade-Schalter** |

- Das eigene Bit ist `1 << (Adresse − 1)`.
- Lade-MOSFET an: eigenes Bit in `0x20002A17` oder Override `0x20002A1A` = 1, und kein
  ladesperrendes Schutzbit.
- Entlade-MOSFET an: eigenes Bit in `0x20002A18` oder Override `0x20002A19` = 1, und kein
  entladesperrendes Schutzbit.
- Ist der Entlade-MOSFET noch aus, läuft zuerst das Vorladen.
- Bei gesperrten Schutzbits: alle drei aus.
- Die Overrides setzt `CAN_Periodic_Alarm_Broadcast` in seiner Startphase: erst zehnmal im
  500-ms-Takt `CAN_PackCmd_Build_Type6`, danach 300-mal im 1-s-Takt.

**Vorladen — `OwnPack_Precharge_SM` (0x080046B8):**

1. Vorlade-Schalter an; `ADC_Read_Precharge_Voltage` (ADC Kanal 1, Teiler 2,1 × 3,3 V, Bereich
   0–69 V) messen.
2. Fertig, wenn der Messwert ≤ 10 V ist **oder** Messwert + Pack-Spannung > 44 V. Dann
   Vorlade-Schalter aus, Entlade-MOSFET an. Sinkt der Messwert nicht (< 1 V pro Durchlauf,
   dreimal) oder dauert es länger als 1 s: Vorlade-Schalter aus, 1 s Pause, neuer Versuch.
3. Nach 2 Wiederholungen: **C1 Bit 2 setzen → Protect1 Bit 9** (in 117.7 §5.2 als
   „Kurzschluss HW“ geführt). Ein Pack, der den Bus nicht vorladen kann, meldet also denselben
   Fehler wie ein Kurzschluss.
4. Liegt GPIOC Pin 1 als Eingang auf High, bricht die Sequenz sofort ab und schaltet den
   Entlade-MOSFET direkt ein.

Was der ADC-Kanal physikalisch misst (vermutlich die Spannung über dem offenen
Entlade-Pfad), ist nicht belegt.

## 8. Lade-Taper

### 8.1 Code

`Max_Charge_Current_Calc` (0x0800FE60) nimmt je Pack SOC-Zone × Temperaturzeile aus drei
Tabellen: Temperatur-Index `0x0801B97E`, SOC-Zonen-Schwellen `0x0801B9A2`, Strommatrix
`0x0801B9BC`. Ergebnis = Eintrag × 0,5 A (Eintrag in % von 50 A). Bei SOC = 100 % → 0 A.

**Alle drei Tabellen liegen hinter dem Image-Ende `0x08019FFF`** und sind aus dem OTA-Image
nicht lesbar. `Master_Build_ChargeCurrentLimit` (0x080093FC) nimmt davon den aktiven Pack
(im Handoff das Minimum aus beiden), höchstens 50 A. Die Ladeschlussspannung ist fest 57,6 V.
Eine Zellspannung geht nirgends ein.

### 8.2 Feld — 22.09.2026, P1, max. Zelltemperatur 33–34 °C

HA-Sensoren `batterieleistung_bms` (= 32100 × 32101) und `bms_batteriespannung` (32100):

| SOC-Bereich | Leistung BMS-seitig | Spannung | Strom | Dauer |
|---|---|---|---|---|
| < 90 % | ~2310 W an der Batterie (AC-Grenze 2500 W) | — | < 50 A, **nicht** BMS-begrenzt | — |
| 90,2 → 95,1 % | 1273–1301 W | 54,8–55,3 V | **≈ 23,2 A konstant** | 6 min 26 s |
| 95,1 → 97,2 % (→ 100) | 497–511 W | 54,6–56,8 V | **≈ 9,1 A konstant** | 7 min 43 s |

Die Leistung steigt mit der Spannung, der Strom bleibt gleich. Die Stufen sind also
Konstantstrom-Zonen der Tabelle. Bei 97,2 % springt der SOC in 30 s auf 100 %
(Vollerkennung bei 56,8 V), danach 15–30 s Pause und der nächste Pack beginnt mit ~1280 W.

Rückrechnung auf Matrix-Einträge (0,5-A-Raster): **46 oder 47** (23,0 / 23,5 A) und
**18 oder 19** (9,0 / 9,5 A). Welcher, lässt sich aus dem Ist-Strom nicht entscheiden, weil der
Wechselrichter knapp unter dem Limit regelt. Das ist eine Messung, kein Tabellenwert.

**Zuverlässiger Weg ohne Flash-Dump:** Register **32106** (`bms_charge_current_limit`,
0,1 A, SRAM `0x20014FA0`) mitloggen. Es ist genau das Ergebnis von
`Master_Build_ChargeCurrentLimit` und liefert je SOC-Zone und Temperatur den Tabellenwert.
In der HA-Integration ist es derzeit **kein** Sensor.

### 8.3 Korrektur

In `VNSD_BMS_FW_Analyse_v118_de.md` §11 stand bis 2026-09-24 ein Plateau von 142–150 W oberhalb 90 %
mit einer Rückrechnung auf einen Matrixeintrag von 5. **Dieses Plateau war ein Effekt der
Omnibattery-Integration, nicht der BMS-Firmware.** Die Passage ist entfernt.

---

## 9. Ghidra-Umbenennungen (2026-09-24)

| Adresse | vorher | nachher |
|---|---|---|
| `0x08006b34` | `OTA_Validate_FW_Version` | `PackSelect_Mode_BySoc` |
| `0x08006cfc` | `APP_Calc_Pack_SOC_Percent_Rounded` | `PackSelect_ChargeStepTarget` |
| `0x08006bbc` | `PackSelect_SOCBalance_Clamp` | `PackSelect_DischargeStepThreshold` |
| `0x0800df0c` | `BMS_OTA_Upgrade_StateMachine` | `PackRecovery_LowCell_Charge_SM` |
| `0x08006d34` | `APP_Get_Packs_Charging_Bitmask` | `Pack_Get_Protect1Bit0_Mask` |
| `0x08006c6c` | `APP_Get_Packs_Charging_Bitmask_2` | `Pack_Get_ChgMosOn_Mask` |
| `0x08006cb4` | `APP_Get_Packs_Discharging_Bitmask` | `Pack_Get_DsgMosOn_Mask` |
| `0x08004cd0` | `FUN_08004cd0` | `PackSelect_Charge_StepRotate` |
| `0x08003bdc` | `FUN_08003bdc` | `PackSelect_Discharge_StepRotate` |
| `0x0800368c` | `FUN_0800368c` | `PackSelect_ModeDispatch` |
| `0x080039b0` | `FUN_080039b0` | `PackSelect_Mode1_Full` |
| `0x08003af4` | `FUN_08003af4` | `PackSelect_Mode2_LowSoc` |
| `0x08003844` | `FUN_08003844` | `PackSelect_Mode3_Empty` |
| `0x08010070` | `FUN_08010070` | `PackSelect_HighestSoc_Online` |
| `0x080100c0` | `FUN_080100c0` | `PackSelect_LowestSoc_Online` |
| `0x08004148` | `FUN_08004148` | `PackSelect_Startup_Closest50` |
| `0x08004394` | `FUN_08004394` | `Master_RunState_InitSelect` |
| `0x08004dc4` | `FUN_08004dc4` | `PackRecovery_Request_Check` |
| `0x08003708` | `FUN_08003708` | `PackSelect_MosMismatch_Fallback` |
| `0x0800352c` | `FUN_0800352c` | `PackSelect_ChargeEnable_Maintain` |
| `0x080034ac` | `FUN_080034ac` | `PackSelect_FaultPath_AllOff` |
| `0x08004924` | `FUN_08004924` | `PackFault_Build_ExcludeMask` |
| `0x0800364c` | `FUN_0800364c` | `Master_Check_AllPacksOnline` |
| `0x08004368` | `FUN_08004368` | `PackHandoff_Tick` |
| `0x08003634` | `FUN_08003634` | `LIB_Popcount` |
| `0x080040e8` | `FUN_080040e8` | `Master_Startup_WaitForEnable` |
| `0x080043c4` | `FUN_080043c4` | `OwnPack_MOSFET_Driver` |
| `0x080046b8` | `FUN_080046b8` | `OwnPack_Precharge_SM` |
| `0x08010110` | `FUN_08010110` | `ADC_Read_Precharge_Voltage` |

Dekompilat-Export `Dekompilate/VNSD_BMS_118/` danach neu erzeugt. Die Fehlnamen im 117.7-Programm sind
im Nachtrag unten mit korrigiert.

Nachtrag am selben Tag:

| Programm | Adresse | vorher | nachher |
|---|---|---|---|
| 118 | `0x08010658` | `SOC_Update_Accumulator` | `prvAddCurrentTaskToDelayedList` (FreeRTOS, Aufrufer `vTaskDelay`/`vTaskPlaceOnEventList*`) |
| 117.7 | `0x08006acc` | `OTA_Validate_FW_Version` | `PackSelect_Mode_BySoc` |
| 117.7 | `0x0800df0c` | `BMS_OTA_Upgrade_StateMachine` | `PackRecovery_LowCell_Charge_SM` |
| 117.7 | `0x080105e8` | `SOC_Update_Accumulator` | `prvAddCurrentTaskToDelayedList` |
| 117.7 | `0x08006cc8` | `APP_Get_Packs_Charging_Bitmask` | `Pack_Get_Protect1Bit0_Mask` |

Die Funktionstabelle in `VNSD_BMS_FW_Analyse_v117.7_de.md` ist angepasst.

---

## 10. Offene Punkte

- **Modus 1** (`PackSelect_Mode1_Full`, voller Pack aktiviert): wertet `0x20002A1D` Bit 0/1 aus.
  Bit 0 ohne Bit 1 → sofort zum leersten Pack. Bit 0 und Bit 1 → nach 300 s Entladen oder
  60 s Leerlauf. Bit 0 aus → bleibt. `0x20002A1D` kommt als Byte 12 aus dem CAN-Frame
  `0x1801FFAA`, dessen ID auch im Control-Image steht (Tabelle `0x08056A94`). Welche
  Control-Funktion ihn sendet und was die Bits bedeuten, ist offen.
- **`0x20002A75`** (Modus 3 statt 2 unter 9 %): Sender von CAN `0x100401AA` Unterbefehl 4 offen.
- **Sender von `0x1801FFAA`** (Freigabe Byte 11, Flags Byte 12, DoD Byte 13): statisch nicht
  gefunden. Stand der Suche vom 2026-09-24:
  - Die ID steht als Konstante **nur im Control**, in allen vier D-Versionen (147, 149, 149.2,
    150). Sie liegt in einer 7er-Tabelle direkt hinter den Konstanten (v150: `0x08056A94`:
    `0x1801FFAA, 0x100101AA, 0x100201AA, 0x100301AA, 0x100401AA, 0x100601AA, 0x180701AA`).
  - Auf die Tabelle zeigt nichts: kein Literal, kein PC-relatives `ADR`/`LDR`, kein Zeiger im
    entpackten `.data` (`Scripts/marstek_descriptor_unpack.py` (lokal, nicht im Repo)). Sie ist entweder toter Code oder
    wird über einen Weg angesprochen, den die statische Analyse nicht sieht.
  - Micro 116 enthält keine ID dieses Schemas und baut sie auch nicht zur Laufzeit:
    `CAN_BuildArbID` (`prio<<24 | b2<<16 | b1<<8 | b0`) wird nur aus `CAN_TX_SendMessage` mit der
    Segmentnummer als Priorität aufgerufen, `0x18` entsteht dort nicht.
  - BMS 118 empfängt sie nur (`CAN_RX_Handler`: `0x180102AA + 0xFD00`). Seine Sende-IDs kommen
    alle aus `CAN_Build_Arbitration_ID`, deren Low-Byte der Funktionscode ist; `0xAA` kann dort
    nicht entstehen.
  - Einzige andere Verwendung im Control: `FUN_0801D4E4` sendet `0x100101AA` (CMD 01) und pulst
    danach GPIOB Pin 10. Die Funktion hängt an einem Tasten-Handler (`FUN_080025A4`, Eingang
    GPIOB Pin 11), ist also eine manuelle Weck- oder Aktivierungssequenz. Den Freigabe-Frame
    sendet sie nicht.
  - Klären ließe sich das nur am Bus: CAN-Mitschnitt beim Start des Geräts, auf Frames mit
    ID `0x1801FFAA` achten.
- **Taper-Tabelle:** Werte nur über Register 32106 oder einen SWD-Dump
  (`Methodik_und_Meta/VNSD_SWD_JTAG_Dump_Plan_de.md`).
- **Verhalten unter 12 %:** Die Vorhersage aus §7 ist nicht am Gerät geprüft. Testvorschlag:
  Zwangsentladung per RS485 ohne Omnibattery bis unter 11 % am aktiven Pack laufen lassen,
  dabei SOC je Pack, 34x04 (MOS-Status) und 32111 (`work_bat_idx`) im 10-s-Raster loggen.
  Erwartung: Wechsel bei `soc% ≤ 10` zum vollsten Pack. Unter 9 % Modus 2/3, Shutdown erst,
  wenn kein Pack mehr voller ist.
