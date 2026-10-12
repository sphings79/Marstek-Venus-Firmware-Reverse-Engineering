# BMS-FW V115 (Venus A) — Vollanalyse

> 🇬🇧 English version: [VNSA_BMS_FW_v115_Full_Analysis.md](VNSA_BMS_FW_v115_Full_Analysis.md)

**Stand:** 2026-10-11.

**Vorgänger/Referenz:** Venus A BMS 110 (`VNSA_BMS_FW_v110_Vollanalyse_de.md`), Venus A BMS 110.5
(`VNSA_BMS_FW_v1105_Vollanalyse_de.md`), Venus D BMS 118 (`VNSD_BMS_FW_v118_Vollanalyse_de.md`) · **Diff-Tabelle:**
[VNSA_BMS_Diff_v110_v115_de.md](VNSA_BMS_Diff_v110_v115_de.md) (mit Spalte 110.5) · **Dekompilat/Listing:**
`Dekompilate/VNSA_BMS_115/` (nur lokal, nicht im Repo) (727 Funktionen, `listing/` mit Flash-Disassembly +
`globals_and_registers.txt`) · **Begleitdokument:** `Control_FW/VNSA_Control_FW_v151_Vollanalyse_de.md`.

## 0. Zusammenfassung

- Image-Basis **0x08002800**, Image-Größe 0x1A800 Byte (108.544 B), MCU **Nations N32G45x** (Cortex-M4F), Keil ARMCC,
  FreeRTOS, letter-shell (Benutzer `BLUE`) — dieselbe Plattform wie A 110/110.5. Versionsstring `Version: 115.0`,
  Build-Datum `Sep  1 2026 11:14:53`. Der Dateinamensstamm `VEPRO_BMS` ist nur ein Name: der Code ist die Codebasis des
  Venus-A-BMS.
- **Codefamilie (geprüft, nicht angenommen):** Die Pointer/BL-maskierte Segmentabdeckung des 115-Images beträgt 82,4 %
  gegen A 110.5, 81,4 % gegen A 110, 65,7 % gegen D 118 und nur 12,6 % gegen E3 BMS 115; der String-Bestand ist bis auf
  Build-/Versionsstrings gleich A 110/110.5. 115 ist also **kein** E3- oder D-artiger Zweig.
- **115 führt beide A-Zweige zusammen:** Die Beta-Logik von A 110.5 (PV-Rettung über CAN `0x03410110`, 120-s-STOP bei
  Zell-UVP Stufe 2, 10,0-A-Ladebegrenzung unter 35,00 V, ±70-A-Komparatorfenster, Schwellen 2800 mV für „leer“ und
  UVP 2A, Niedrigspannungs-Balancer mit Min-Zelle < 2800 mV, SELF-Flag-Sperre nur über 39,00 V) ist wieder enthalten,
  Teile von A 110/109 bleiben (keine `g_chargeStatusCode` 2/3/4, feste Ladespannungsgrenze). Die **Systemstromgrenzen
  liegen wieder bei 50,0 A** (110: 40,0 A), die Ladespannungsgrenze ist fest **48,0 V** (CAN-Klemmung 46,8 V
  unverändert).
- **Im Code gefundene Changelog-Punkte** (Remark des OTA-Eintrags, übersetzt): (1) Packwechsel bei SOC-Differenz
  zwischen Packs über 20 → `PackSelect_SocImbalanceSwitch`; (2) dynamische Korrektur der Lade-/Entladeziele →
  `PackSelect_ChargeStepTarget` (95-/100-%-Regeln) und Neuberechnung der Schwellen in den Step-Rotationen;
  (3) Lade-/Entlade-Endkorrektur braucht drei Zellen an der Schwelle → Zellzähler aus `APP_Get_Pack_Summary_Data`,
  genutzt von `Charge_SOC_Integration`/`Discharge_SOC_Integration`; (4) verbesserte Lade-/Entlade-MOS-Befreiung →
  `OwnPack_MosRecovery_SM` (neuer Ausgang PE7, CAN `0x1805AA01`).
- **Nicht im Changelog, aber neu:** RTC-Weckzeitgeber der MCU vor dem STOP-Modus (LSI, nominal 6 h) mit automatischer
  Rückkehr in STOP nach 50 Schutz-Ticks; Pack-Unterspannungs-STOP unter 36,00 V; Schutzzähler werden bei einem
  AFE-Kommunikationsfehler gelöscht; zwei neue eingehende CAN-IDs (`0x01210103`, `0x04410110`); SOC-Übergabe-
  Zustandsautomaten, gekoppelt an ein EMS-Modus-Byte; Konfigurationsblock auf 0xD8 Byte vergrößert mit Versionskennung
  `'s'`; SOC-Kalibrierlog auf 5 Slots reduziert.
- **Pack-Kapazität:** `CAN_TX_PerPack_10Msgs` sendet Packs 1..6 (`pack_id != 0 && pack_id < 7`), **13 Zellen** pro Pack —
  unverändert.
- **Alle 727 Funktionen benannt und typisiert**, 0 `FUN_*`, alle Prüf-Gates bestanden (§2.3). Tags: `NAME_VERIFIED` 727,
  `NAME_LOW_CONFIDENCE` 51.

## 1. Binär-Fingerabdruck

| Datei | Version | Größe | SHA-256 |
|---|---|---|---|
| `115_bms_VNSA-0_VEPRO_BMS_V115_APP_20260901_ota_174540.bin` | 115.0 | 108.544 B (0x1A800) | `d44833daebebf49350deeffbb725b15b80babd33f35d69e94a004005c8cc8758` |

Hersteller-Changelog: „VNSA115 Official Edition“; OTA-Remark (übersetzt): „Offizielle Version; 1. Packwechsel bei einer
SOC-Differenz zwischen Packs größer 20 ergänzt, um eine zu große SOC-Differenz zu vermeiden; 2. dynamische Korrektur der
Lade-/Entlade-Zielwerte ergänzt, damit alte Werte in Sonderfällen den Packwechsel nicht beeinflussen; 3. Empfindlichkeit
der Lade-/Entlade-Endkorrektur gesenkt, jetzt müssen drei Zellspannungen die Schwelle erreichen; 4. Lade-/Entlade-
MOS-Befreiungsmechanismus optimiert, um Extremsituationen zu vermeiden“. OTA-Eintrag: Edition 4563, CRC 30760, nicht
erzwungen. Image-Versionskennung bei `0x08010000`: **`BMS2`** (unverändert).

Der Dateinamensstamm `VEPRO_BMS` (A 110: `VABMS`, A 110.5: `VA50A`) lässt eine geplante „Venus E Pro" (App-Typcodes `VEPRO`/`VEPRO2`)
vermuten, die den Venus-A-BMS-Code nutzt; das ist eine Annahme aus dem Namen, die Firmware selbst bestätigt sie nicht
(Details: [VNSA_BMS_FW_v115_Offene_Punkte_de.md](VNSA_BMS_FW_v115_Offene_Punkte_de.md) §0).

## 2. MCU, Speicherkarte, Bearbeitungsstand

### 2.1 MCU

Nations N32G45x wie bei 110/110.5/D (Vektortabelle mit 102 Einträgen, UID bei `0x1FFFF7F0`, `SystemInit` `0x08011768`).
Neu genutzte Peripherie: **RTC** (`0x40002800`: WRP, CTRL, INITSTS, WKUPT), **Backup-Register** (`0x40006C00`,
BKP_DAT1/2), PWR-DBKP und RCC-LSI/RTC-Bits über Bit-Band-Aliase; der RTC-Interrupt (Vektor IRQ3 = `RTC_IRQn`, A 110:
Default-Handler) wird jetzt genutzt.

**Nachweis der Image-Basis:** Vektortabelle Wort 0 = SP `0x2000C9A8`, Wort 1 = Reset `0x08002A6D`. Bei Dateioffset 0x26C
liegt der Keil-`Reset_Handler` (`ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`), dessen Literale `0x08011769`
(SystemInit) und `0x08002999` (`__main`) nur bei Basis 0x08002800 ins Image zeigen; die Keil-Regionstabelle bei
`0x0801CA30` enthält dazu passende Lade-/Ausführungsadressen.

### 2.2 Speicherkarte

| Bereich | Inhalt |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (nicht Teil des Images); Boot-/OTA-Seite `g_bootConfigPage` bei `0x08002000` |
| `0x08002800`–`0x0801CFFF` | Anwendungsimage; Vektortabelle bei `0x08002800` |
| `0x08010000` | Image-Versionskennung `g_imageVersionTag` = `'BMS2'` |
| `0x0801BB6C` | `g_crc8Poly07Table` (CRC-8, Polynom 0x07, SH36735-Frames) |
| `0x0801CA30` | Keil-Regionstabelle (2 Regionen), `0x0801CA50` `shellCommandTable` (52 × 16 B), `0x0801CD90` `.data`-Ladeabbild |
| `0x0801E000` | OTA-Staging-Kopie `g_otaStagingImage` (Kennungswort `0x0802B800` muss `'BMS2'` sein) |
| `0x20002800`–`0x20002BDF` | `.data` (0x3E0 B; 110: 0x398 B, 110.5: 0x3A0 B) |
| `0x20002BE0`–`0x2000C9A7` | `.bss` (0x9DC8 B); FreeRTOS-`ucHeap` bei `0x20004718` (bereits 8-Byte-ausgerichtet, kein Füllwort) |
| `0x2000C9A8` | `__initial_sp` (110: `0x2000C958`) |
| I2C-EEPROM | Konfigurationsblock bei 600 (**0xD8 B, Kennung `'s'`**), SOC-Kalibrierlog **0x440, 5 × 20 B**, Master-Marker `0x01F6`, Zähler `0x0254`, Ereignislog `0x2000`/`0x3000`, Fehlersperre `0xFC00`, Vollladungs-Marker `0xFC02` |

Die RAM-Namen wurden über den Abgleich der Literal-Pool-Referenzen gepaarten Codes gegen 110 portiert (und gegen 110.5
geprüft); das Layout weicht von 110 stückweise um 1–0x50 Byte ab (neue Variablen, entfernter Relay-Comm-Unterzustand/
-Timer). Eine reine Adressportierung wäre an mehreren Stellen falsch gewesen (z. B. liegt `g_pack0SlotId` bei
`0x200037DE`, die zwölf Übertemperatur-Zähler sind um 0xC verschoben).

### 2.3 Bearbeitungsstand in Ghidra

- Programm `/VNSA/BMS/BMS_115_VNSA-0_VEPRO_BMS_V115_APP_20260901_ota_174540.bin`: **727 Funktionen, alle benannt**, jede
  Funktion mit benutzerdefiniertem Prototyp und Plate-Kommentar, 0 `FUN_*`, 0 Duplikate.
- Portierung von 110: Segmentabgleich (374 Segmente, 98 nach Zusammenführung gleich großer Lücken), 661 Funktionspaare
  über Adressen (636 maskiert identisch inkl. Lokalvariablen) + 30 über Befehlsähnlichkeit (monotone Reihenfolge) +
  6 von Hand verschoben/neu angelegt (`CAN_TX_PF2_Capacity`, `PackHandoff_Tick`, `PackSelect_ChargeStepTarget`,
  `APP_Get_Pack_Summary_Data`, `SoftFloat_Round_To_Even`, `SoftFloat_Pack_Float_Result`). 30 neue Funktionen aus dem
  Code analysiert und benannt (RTC/BKP-Treiber nach der N32-Standardbibliothek benannt; `SoftFloat_Int_To_Double`/
  `_Double_To_Float` und `HAL_RCC_Get_Flag_Status` byte-identisch zu E3 115 bzw. D 118 und wie dort benannt).
- Strukturreparaturen: Literal-Pools hinter nicht zurückkehrenden/Tail-Aufrufen, drei Funktionen, die eine folgende neue
  Funktion verschluckt hatten, aufgeteilt, Scheinfunktion im Shell-Benutzerstring entfernt, Strings und Tabellen am
  Image-Ende neu typisiert, zwei falsche Analysator-Referenzen gelöscht. `SOC_Select_Threshold_2420_2750` wieder in
  **`SOC_Select_Threshold_2420_2800`** umbenannt (Code = 110.5).
- Gates (`vna115_bms_gates.py`, dieselben Prüfungen wie für 110/110.5): 0 undefinierte Bytes, 0 verwaiste Befehle,
  0 Flüsse in Daten, 0 Funktionszeiger auf Nicht-Code, 0 Nur-Literal-Funktionen, 0 als Code dekodierte Literal-Pools,
  0 Aufrufe in Funktionsrümpfe, 0 unbenannte referenzierte RAM-Globals, 0 Default-Prototypen, Tasknamen =
  `xTaskCreate`-Namensstrings, No-Return-Flags plausibel, 0 doppelte Labels; ein 1-Zeichen-String gemeldet = leeres
  Shell-Passwort (wie 110/110.5). Dekompilat-Export: 727 Funktionen, **0 Fehler**.

## 3. FreeRTOS-Tasks

`RTOS_CreateAllTasks` (`0x0800F9FC`) legt dieselben 10 Tasks wie 110 an (Namen, Stacks, Prioritäten unverändert):

| Task | Einsprung | Stack (Worte) | Prio |
|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800FC88` | 0x80 | 5 |
| `RTOS_vTaskKeyScan` | `0x0800FC68` | 0x80 | 9 |
| `RTOS_vTaskKA495XX` (SH36735-Worker) | `0x0800FC50` | 0x80 | 8 |
| `RTOS_vTaskCanRec` | `0x0800FBF0` | 0x180 | 4 |
| `RTOS_vTaskCanSend` | `0x0800FC00` | 0x80 | 3 |
| `RTOS_vTaskRS485Comm` | `0x0800FCC2` | 0x180 | 1 |
| `RTOS_vTaskSocAlgorithm` | `0x0800FCD0` | 0x100 | 7 |
| `RTOS_vTaskProtectCheck` | `0x0800FCAC` | 0x100 | 6 |
| `RTOS_vTaskControlLogic` | `0x0800FC0A` | 0x80 | 2 |
| `RTOS_vTaskDataSave` | `0x0800FC24` | 0x80 | 0 |

Der IWDG wird weiterhin nur bedient, von der Anwendung nie konfiguriert oder gestartet.

## 4. Schnittstellen

### 4.1 CAN

Frame-Satz und Layouts wie bei 110.5 (`VNSA_BMS_FW_v1105_Vollanalyse_de.md` §4.1), mit diesen Unterschieden:

**Ausgehend**

| ID | Funktion | A 115 |
|---|---|---|
| `0x1802AA01` | `CAN_TX_PF2_Capacity` `0x080084EC` | Byte 7 = **Packauswahl-Quittung** (`g_packSelectAckReport`: 2, sobald die PB13-Quittung gesehen wurde, nach dem Senden gelöscht); 110/110.5: Relay-Comm-Unterzustand 1/2/3 |
| `0x1803AA01` | `CAN_TX_PF3_ChargeDischargeLimits` `0x080085D4` | Ladespannungsklemmung 46,8 V; Ladestrom = Systemgrenze **≤ 50,0 A**, **10,0 A, solange die eigene Packspannung unter 35,00 V liegt** (wie 110.5), 25,0 A bei Packfehler; Entladestrom ≤ 50,0 A |
| `0x1804AA01` | `CAN_TX_PF4_ProtectWarnings` `0x0800876C` | Byte 4 = **0** (110: `g_chargeStatusCode`, 110.5: 2/3/4) |
| `0x1805AA01` | `APP_IdleKeepaliveBroadcast_Tick` `0x0800F144` | **neu**, DLC 1: `g_mosRecoveryReport` (0 Ruhe, 1 MOS-Befreiung läuft, 2 fertig); aus dem Leerlauf-Tick gesendet, sobald 50 ms vergangen sind (auf Nicht-Master-Packs danach bei jedem Aufruf, weil der Timer nur im Master-Pfad zurückgesetzt wird) |
| `0x1820..0x1841AA00+n` | `CAN_TX_PerPack_10Msgs` `0x08008DE8` | Packs 1..6 (`pack_id != 0 && pack_id < 7`); 0x1821 Ladestrom **10,0 A für jedes Pack, solange die eigene Packspannung unter 35,00 V liegt** (wie 110.5); 0x1823 FW-Kennung des eigenen Packs **0x73 = 115** |
| `0x1851/0x1852AA00+n` | `CAN_TX_PerPack_SocDiag` | unverändert |

**Eingehend** (`CAN_RX_Handler` `0x08008BD4`)

| ID | A 115 |
|---|---|
| `0x03410110` (PV-Rettung des Wechselrichters) | `g_pvRescueRequestFlag` = (Byte 1 > 8) **oder (Byte 6 == 1)**; 110.5: nur Byte 1 > 8; 110: ignoriert |
| `0x1801FFAA` (Master-Broadcast) | wie bisher, zusätzlich **Byte 3 → `g_emsHeartbeatModeByte`**; Wert 2 schaltet die SOC-Übergabe-Zustandsautomaten und die Relay-Comm-Sperre frei (§6) |
| `0x01210103` | **neu:** `g_can01210103Flag` = oberes Nibble von Byte 0 != 0 (hält die Relay-Comm-Sperre gesetzt); Sender nicht identifiziert |
| `0x04410110` | **neu:** `g_can04410110Ratio` = int16(Bytes 4–5) × 10000,0 / uint16(Bytes 6–7) als float; in diesem Image ohne Leser; Sender nicht identifiziert |

Systembefehls-Opcodes (`0x100101AA`, inkl. 0x0A–0x0C), die Heartbeat-Antwort auf `0x100501AA`, der gestaffelte
Slave-Statusbericht und der `0xA5A5`-Echotest sind unverändert (Code wie 110.5).

Das A-Control 151 füllt in `0x1801FFAA` nur die Bytes 0–2 (`Grid_BuildStatusFlagsReg`); Byte 3 bleibt 0, mit Control
151 werden die Modus-2-Pfade aus §6 also nicht aktiviert (aus dem Code; nicht auf dem Bus beobachtet).

### 4.2 RS485

Unverändert bis auf das Versionsbyte **`'s'` (0x73)** in Opcode 5 und in `RS485_Pack_Telemetry_Response` (110: `'n'`).
Code sonst identisch mit 110.5.

### 4.3 Debug-Shell (letter-shell)

Unverändert: Benutzer `BLUE`, 52 Tabelleneinträge, 39 Befehle. Das Login-Banner zeigt `Version: 115.0` und das
Build-Datum.

### 4.4 EEPROM-Belegung (I2C)

| Adresse | Änderung in 115 |
|---|---|
| 600 (0x258) Konfigurationsblock | **0xD8 Byte** (110: 0xD4), Versionskennung **`'s'`** (110: `'n'`), neues Feld `g_cfgPackUvShutdownTimeout` bei Offset 0xD4 (Default 120). Die Prüfsumme (Offset 0xD0) wird über die Offsets 0..0xD3 gebildet, also **über ihren eigenen vorherigen Wert**, und deckt das neue Feld nicht ab (§13). |
| `0x0440`… SOC-Kalibrierlog | **5 Slots × 20 B ab 0x440** (110: 50 Slots ab 0x400). Der Formatierungsmarker bei `0x024E` (`'f'`) wird nicht zurückgesetzt, ein von einer älteren Version formatiertes EEPROM wird also nicht neu formatiert; die neuen Slots beginnen bei altem Slot 3 + 4 Byte. |
| `0x01F6`, `0x0254`, `0x2000`/`0x3000`, `0xFC00`, `0xFC02` | unverändert |

## 5. Zellüberwachung — SH36735

Unverändert: SPI1, CRC-8-Polynom 0x07, **13 Zellen** (`INIT_SH36735_Subsystem` `0x0800E530`, nur die Versionskonstante
auf 0x73 geändert), Balancing-Regeln, Nullpunktkalibrierung („SELF“-Flag). Neu: Während eines
AFE-Kommunikationsfehlers (`g_protState4` Bit 4) werden die Zell-, Strom- und Temperatur-Schutzzähler gelöscht und nicht
ausgewertet.

## 6. Packverwaltung

- **SOC-Ungleichgewichts-Wechsel** (`PackSelect_SocImbalanceSwitch` `0x08007B98`, aufgerufen in Packauswahl-Modus 0): Ist
  der höchste minus der niedrigste SOC der Online-Packs **≥ 20 %**, wird nach einer leistungsabhängigen Zeit ein Wechsel
  angefordert (|I| × V des aktiven Packs > 1500 W: 1200 Ticks, > 1000 W: 1800, > 500 W: 2400, sonst 3000): beim Laden
  (> 1,0 A) zum Pack mit dem niedrigsten SOC, beim Entladen (< −1,0 A) zum Pack mit dem höchsten SOC.
- **Dynamische Ziele:** `PackSelect_ChargeStepTarget` liefert 100 %, wenn alle Online-Packs ≥ 95 % sind, 95 %, wenn alle
  ≥ 90 % sind und das Pack unter 95 % liegt, sonst die bisherige 10-%-Stufe. `PackSelect_Charge_StepRotate` /
  `_Discharge_StepRotate` berechnen eine gespeicherte Schwelle neu, die mehr als 10 % vom SOC des aktiven Packs abweicht.
- **SOC-Übergabe-Zustandsautomaten** (`PackHandoff_Charge_SocSwitch_SM` `0x08006B5C`,
  `PackHandoff_Discharge_SocSwitch_SM` `0x08006D0C`): von `PackHandoff_Tick` statt `PackHandoff_StateMachine` genutzt,
  wenn `g_emsHeartbeatModeByte == 2` und das PV-Rettungsflag gesetzt ist; sie schalten die Lade- und Entladefreigaben von
  altem und neuem Pack in einer von der Stromrichtung abhängigen Reihenfolge (Laden: altes Entladen aus → neues Laden
  ein → altes Laden aus → neues Entladen ein; Entladen: altes Laden aus → neues Entladen ein → altes Entladen aus → neues
  Laden ein), jeder Schritt wartet auf den MOS-Status, und überspringen das Warten auf den Lade-MOS, wenn das neue Pack
  > 98 % hat.
- Freigabemasken: `PackSelect_ChargeEnable_Maintain`, `_MosMismatch_Fallback` und `_Next_AboveSocThreshold` setzen die
  Lade-/Entladefreigabemasken vor einer Neuwahl auf das aktive Pack zurück; das aktive Pack wird wieder in die
  Entladefreigabemaske aufgenommen, wenn kein Schutz es sperrt und sein SOC über 100 % − Entladegrenze liegt.
- Relay-Comm-Zustandsautomat: Der Unterzustand (1/2/3) und sein 300-Tick-Befehlstimer entfallen; die neue
  `g_relayCommEscalationInhibit` (PV-Rettung in Modus 2, Flag von CAN `0x01210103` oder |Systemstrom| ≥ 1,1 A)
  unterdrückt die Eskalationen nach 300/600 Ticks in die Fehlermodi 3/2 und das zugehörige LED-Muster.
- **MOS-Befreiung** (`OwnPack_MosRecovery_SM` `0x08005BA4`, alle 100 ms von `BMS_Role_Dispatcher` getaktet): Findet die
  Master-Variante des MOSFET-Treibers den Ladefreigabe-Ausgang PC5 auf Low, obwohl er an sein sollte, steuert sie den
  neuen Ausgang **PE7** (Schalter 7), bis die eigene Packspannung über 8,49 V liegt, wartet 20 Ticks und schaltet dann PE7
  aus und PC5 ein; der Fortschritt wird in `0x1805AA01` gesendet. Wiederholungs-/Fehlerzustände (mit `g_protState1`
  Bit 2) existieren, kein Pfad in diesem Image setzt sie. Die Board-Funktion von PE7 ist aus der Firmware nicht
  erkennbar.
- `Pack_LowVoltage_Balancer` und SELF-Flag-Sperre wie 110.5 (Kriterium Min-Zelle < 2800 mV; Relais ohne SELF-Flag nur
  über 39,00 V zwangsweise aus).

## 7. SOC/SOH

Tabellen unverändert (byte-identisch). Änderungen: Leer-Schwelle **2800 mV** bei ≥ 0 °C
(`SOC_Select_Threshold_2420_2800` `0x08014624`, wie 110.5; 110: 2750 mV); die Lade-End-Interpolation in
`Charge_SOC_Integration` beginnt, sobald **3 Zellen ≥ 3500 mV** sind (110: Max-Zelle ≥ 3500 mV), die
Entlade-End-Kompensation in `Discharge_SOC_Integration`, sobald **3 Zellen ≤ 2900 mV** sind (110: Min-Zelle ≤ 2900 mV);
die Zähler stammen aus `APP_Get_Pack_Summary_Data` (Bytes 14/15). Das SOC-Kalibrierlog hat 5 Slots (§4.4).

## 8. Lade-/Entladegrenzen

| Punkt | Venus A 115 | Venus A 110 | Venus A 110.5 |
|---|---|---|---|
| System-Ladestromgrenze (`g_sysChargeCurrentLimit_x10`, Modbus 32106) | **50,0 A** | 40,0 A | 50,0 A |
| System-Ladestrom außerhalb Master-Laufzustand 2 | 50,0 A, 0 solange `g_socStateFlags` Bit 5 gesetzt ist | 0 | 50,0 A (0 mit Statuscode 4, solange Bit 5 gesetzt ist) |
| System-Entladestromgrenze (Modbus 32107) | **50,0 A** | 40,0 A | 50,0 A |
| Ladespannungsgrenze (`g_sysChargeVoltLimit_x10`, Modbus 35110) | **48,0 V fest**, CAN-Klemmung 46,8 V | 46,8 V fest | Packspannung + 2,0 V, max. 48,0 V, CAN-Klemmung 46,8 V |
| CAN-Ladestrom unter 35,00 V eigener Packspannung (0x1803 und 0x1821) | **10,0 A** | keine Begrenzung | 10,0 A |
| Max. Ladestrom pro Pack (`Max_Charge_Current_Calc` `0x08014A08`) | Tabelle, max. 50,0 A; Derating „nie voll geladen“ | gleich | gleich |
| Entladespannungsgrenze | 39,0 V | 39,0 V | 39,0 V |

Mit dem Control werden die Systemgrenzen zu Leistung: 50,0 A bei der Nennspannung 13S (41,6 V) sind ~2080 W, über der
Grenze des A-Micro, die Batteriegrenze begrenzt die AC-Leistung also normalerweise nicht (Schlussfolgerung, nicht
gemessen).

## 9. Schutzfunktionen

Konfigurations-Defaults unverändert (neues Feld: `g_cfgPackUvShutdownTimeout` = 120). Code:

- `Cell_OVP_UVP_TempCompensated_Protection` (`0x0800DBB0`): wie 110.5 (UVP Stufe 2, Stufe A 2800 mV und Stufe B 2500 mV
  setzen beide Protect1 Bit 5 und den SOC auf 0 %; Sensorfehler-Auslösung nur über 39,00 V); **neu:** alle Zähler werden
  gelöscht, solange der AFE-Kommunikationsfehler gesetzt ist. Derselbe Rücksetzblock ist in `Current_Protection_Checker`
  und `Voltage_Temp_Protection_Checker` ergänzt.
- `BMS_ProtectionAndCommTick` (`0x0800A18C`): wie 110.5 — bei Zell-UVP Stufe 2 **120 s ohne `g_pvRescueRequestFlag` →
  STOP** (`System_Reset_And_Halt_WithCurrentCheck`); eigene MOSFETs während UVP2 gesperrt; `g_standaloneModeFlag`
  geschrieben.
- **Neu `Pack_UnderVoltageShutdown_Check`** (`0x0800D590`): eigene Packspannung **< 36,00 V** für
  `g_cfgPackUvShutdownTimeout × 10` Ticks (Default 1200) → `g_protState4` Bit 5 und, sofern das PV-Rettungsflag nicht
  gesetzt ist, STOP; Freigabe über 37,00 V. Nicht in die Protect-/Warning-Worte abgebildet (intern; über den Snapshot als
  Ereignis 1 protokolliert).
- `HW_OvercurrentAndSensorProtection` (`0x0800BC70`): Komparatorfenster **±70,0 A** (wie 110.5); **neu:** Prüfungen der
  NTC-Anzahl und der MOS-Temperaturbereiche unter 20,00 V Packspannung übersprungen.
- **Neuer Ausgang PD8** (`Pack_LowVoltageOutput_PD8_Update`): High, solange die eigene Packspannung unter 20,00 V liegt.

## 10. STOP-Modus und RTC-Wecken (neu)

`System_Reset_And_Halt_WithCurrentCheck` (`0x08004B1C`) ruft jetzt vor dem STOP `RTC_ArmWakeupTimer_Before_Stop`
(`0x0800FD1C`) auf:

1. PWR-Takt und Schreibzugriff auf die Backup-Domäne ein, LSI ein (warten auf LSIRD), RTC-Taktquelle = LSI, RTC-Takt
   ein, warten auf Synchronisation.
2. Nur wenn `BKP_DAT1 != 0x5460`: Weckzeitgeber aus, `WKUPSEL = 4` (ck_spre), `WKUPT = 0x5460` (21600), Marker
   `BKP_DAT1 = 0x5460`. 21601 Perioden = **6,0 h bei nominal 1 Hz**; der RTC-Vorteiler wird nicht geschrieben
   (Reset-Wert), mit dem unkalibrierten LSI (~40 kHz typisch) hängt die reale Periode also von der LSI-Frequenz ab (etwa
   4,9 h bei genau 40 kHz mit dem Default-Vorteiler /128/256) — aus Code und Referenzhandbuch abgeleitet, nicht gemessen.
3. EXTI-Leitung 20 steigende Flanke + NVIC IRQ3 (`INIT_RtcWakeupExti_And_NVIC`), Weck-Interrupt frei, Backup-Domäne
   gesperrt, Weckzeitgeber ein; dann STOP, danach System-Reset.

Beim Wecken löscht `RTC_IRQHandler` (`0x0800F974`) die Flags, schreibt **`BKP_DAT2 = 1`** und stoppt den RTC-Takt. Beim
Start kopiert `BKP_ReadAndClearRtcWakeFlag` BKP_DAT2 nach `g_rtcWakeupFlag` und löscht es; `GPIO_Init_All_Pins`
initialisiert dann PC5/PB0/PB2/PB5 neu, und `RTC_WakeCycle_ReturnToStopTick` schickt das BMS **nach 50 Schutz-Ticks
zurück in STOP** (das Flag wird erst beim nächsten Start gelöscht). Ein schlafen gelegtes BMS (UVP2 ohne PV-Rettung, Pack
< 36,00 V, 600 s offline, Systembefehl 2) wacht also periodisch kurz auf, statt bis zu einem externen Wecken zu
schlafen. Welche externen Ereignisse den Zyklus beenden (z. B. Ladegerät, CAN-Aktivität), ist in diesem Pfad nicht
erkennbar; das Weckflag selbst wird während des Laufs nie gelöscht (offener Punkt).

## 11. OTA/Update

Unverändert: CAN-`BMS_OTA_Flash_Writer` (`0x0800E0EC`, Opcode 0xCE) und `RS485_OTA_Flash_Handler` (`0x0800D870`),
Staging-Kopie `0x0801E000`, CRC16, Kennung `'BMS2'`. Ein 115-Image wird von einem A-BMS mit 109/110/110.5 angenommen und
umgekehrt (gleiche Kennung); D-Images (`'BMS1'`) werden abgewiesen. **Nach dem Update wird der Konfigurationsblock mit
Defaults neu geladen**, weil sich die Versionskennung (`'n'` → `'s'`) und die Blocklänge (0xD4 → 0xD8) ändern.

## 12. Sicherheitshinweise

Unverändert gegenüber 110.5 (Shell am RS485-Debugpfad, nicht authentifizierte CAN-Befehle, IWDG nicht gestartet). Neue
nicht authentifizierte CAN-Eingänge: `0x01210103` (Relay-Comm-Sperre), `0x04410110` (Wert ohne Leser) und das erweiterte
`0x03410110` (hält das BMS in UVP2 und im neuen Pack-Unterspannungspfad wach).

## 13. Offene Punkte

Ausführliche Ausarbeitung der Punkte 1–3, 6 (Rechnung, Sender/Empfänger, Pins, Beweisstufen):
[VNSA_BMS_FW_v115_Offene_Punkte_de.md](VNSA_BMS_FW_v115_Offene_Punkte_de.md).

1. **Konfigurations-Prüfsumme:** `APP_Config_Checksum_And_Save` (`0x0800B7E4`) und `BMS_Config_Params_Load_Or_Default`
   (`0x0800A840`) bilden die Einerkomplement-Bytesumme über die Offsets 0..0xD3, die das Prüfsummenwort bei 0xD0 selbst
   enthalten. Ein gespeicherter Block besteht die Prüfung nur, wenn die Bytesumme des neuen und des vorherigen
   Prüfsummenworts zufällig gleich ist; sonst werden beim nächsten Start die Defaults geladen (und gespeichert), geänderte
   Parameter blieben also nicht erhalten. Aus dem Code abgeleitet, nicht am Gerät getestet.
2. Sender und Bedeutung von `0x01210103` und `0x04410110`; Bedeutung von Byte 6 in `0x03410110`; wer
   `g_emsHeartbeatModeByte == 2` sendet (Control 151 sendet 0).
3. Board-Funktion der neuen Ausgänge PE7 (MOS-Befreiung) und PD8 (< 20,00 V) und ihre Pin-Modus-Einrichtung (nur die
   strukturbasierten GPIO-Inits sind ungeklärt); Bedeutung der 8,49-V-Bedingung in der Befreiung.
4. Reale RTC-Weckperiode (LSI) und das Feldverhalten des periodischen Weck-/STOP-Zyklus; was ihn beendet.
5. SOC-Kalibrierung nach einem Update: Das alte Log (50 Slots ab 0x400) wird mit dem neuen 5-Slot-Layout ab 0x440
   gelesen; ob beim ersten Start ein gültiger Datensatz gefunden wird, ist nicht getestet.
6. Empfänger von `0x1805AA01` in A-Control/-Micro: im Control 151 kein Handler gefunden (Suche nach der ID im
   Dekompilat-Export).
