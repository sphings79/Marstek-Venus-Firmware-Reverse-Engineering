# BMS-FW V110 (Venus A) — Vollanalyse

> 🇬🇧 English version: [VNSA_BMS_FW_v110_Full_Analysis.md](VNSA_BMS_FW_v110_Full_Analysis.md)

**Stand:** 2026-10-11.

**Vorgänger/Referenz:** Venus A BMS 110.5 (`VNSA_BMS_FW_v1105_Vollanalyse_de.md`), Venus D BMS 118
(`VNSD_BMS_FW_v118_Vollanalyse_de.md`) · **Diff-Tabelle:** [VNSA_BMS_Diff_v1105_v110_de.md](VNSA_BMS_Diff_v1105_v110_de.md) ·
**Dekompilat/Listing:** `Dekompilate/VNSA_BMS_110/` (nur lokal, nicht im Repo) (697 Funktionen, `listing/` mit
Flash-Disassembly + `globals_and_registers.txt`) · **Gegenstück:** `Control_FW/VNSA_Control_FW_v151_Vollanalyse_de.md`.

## 0. Zusammenfassung

- Image-Basis **0x08002800**, Image-Größe 0x19800 Bytes (104.448 B), MCU **Nations N32G45x** (Cortex-M4F), Keil ARMCC,
  FreeRTOS, letter-shell (Benutzer `BLUE`) — dieselbe Plattform wie A 110.5. Versionsstring `Version: 110.0`,
  Build-Datum `May 20 2026 13:59:48`.
- **Codebasis: A 109, nicht A 110.5.** Das Image ist byte-identisch mit A BMS 109 (Build 26.12.2025) bis auf die
  System-Stromgrenze, die Versionskonstanten und den Build-/Versionsstring (334 abweichende Bytes). Die Änderungen der
  Beta 1105 („110.5", Build 11.05.2026) sind **nicht** enthalten. Nach Build-Datum und OTA-Edition (3430 gegen 3316) ist
  110 das neuere Release; inhaltlich ist es ein paralleler Zweig (109 + 40 A). Details: [Diff-Tabelle](VNSA_BMS_Diff_v1105_v110_de.md).
- **Changelog „maximaler Lade-/Entladestrom auf 40 A geändert":** `Master_Build_ChargeCurrentLimit` und
  `Master_Build_DischargeCurrentLimit` begrenzen die **System**grenzen auf **400 (40,0 A)** statt 500. Das sind die in
  CAN `0x1803AA01` gesendeten Werte, die das Control als Modbus 32106/32107 (Aliase 35111/35112) bereitstellt. Die
  Pack-Grenzen (Lade-/Entlade-Drosselungstabellen, Tabellenmaximum 50,0 A) sind unverändert.
- **Verhalten gegenüber 1105 (was ein Update 1105 → 110 ändert):** kein PV-Rettungs-Handshake (CAN `0x03410110` wird
  ignoriert, kein Schlafen nach 120 s in Zell-UVP Stufe 2; stattdessen die Venus-D-Logik: eigene MOSFETs nur unter
  2,0 A, alleinstehendes Pack Fehlermodus 3 nach 200 s), keine 10,0-A-Ladebegrenzung unter 35,00 V,
  Ladespannungsgrenze fest 46,8 V, UVP Stufe 2 Stufe A 3000 mV nur als Warnung, Überstrom-Komparatorprüfung ±3,0 A,
  Leer-Schwelle 2750 mV, gemeldete BMS-Version **1100**.
- **Pack-Kapazität:** `CAN_TX_PerPack_10Msgs` sendet Packs 1..6 (`pack_id != 0 && pack_id < 7`), **13 Zellen** je Pack.
- **Alle 697 Funktionen benannt und typisiert**, 0 `FUN_*`, alle Prüf-Gates bestanden (§2.3). Tags: `NAME_VERIFIED` 697,
  `NAME_LOW_CONFIDENCE` 49.

## 1. Binär-Fingerabdruck

| Datei | Version | Größe | SHA-256 |
|---|---|---|---|
| `110_bms_VNSA-0_VABMS_APP_110_ota_163748.bin` | 110.0 | 104.448 B (0x19800) | `f4ee89c8837f1178efe085aae107f733483055bc2d91e57d510e354167283559` |

Hersteller-Changelog (übersetzt): „Maximaler Lade-/Entladestrom auf 40 A geändert." OTA-Eintrag: Edition 3430,
CRC 13436, nicht erzwungen. Image-Versionskennung bei `0x08010000`: **`BMS2`** (unverändert). Zum Vergleich: A 109
(`109_bms_VNSA-0_20251226103854b52c66481.bin`) Changelog „max. angeforderter Lade-/Entladestrom auf 50 A geändert".

## 2. MCU, Speicherkarte, Bearbeitungsstand

### 2.1 MCU

Nations N32G45x wie bei 1105/D (Vektortabelle mit 102 Einträgen, UID bei `0x1FFFF7F0`, `SystemInit` `0x080105D4` setzt
`SCB_VTOR = 0x08002800`). Register- und IRQ-Namen aus der N32G457-SVD (von 1105 übernommen).

**Nachweis der Image-Basis:** Vektortabelle Wort 0 = SP `0x2000C958`, Wort 1 = Reset `0x08002A6D`. Datei-Offset 0x26C
enthält den Keil-`Reset_Handler` (`ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`), dessen Literale `0x080107D5`
(SystemInit) und `0x08002999` (`__main`) nur bei Basis 0x08002800 in das Image zeigen; die Keil-Regionstabelle bei
`0x0801B860` enthält dazu passende Lade-/Ausführungsadressen.

### 2.2 Speicherkarte

| Bereich | Inhalt |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (nicht Teil des Images) |
| `0x08002800`–`0x0801BFFF` | Anwendungs-Image; Vektortabelle bei `0x08002800` |
| `0x08010000` | Image-Versionskennung `g_imageVersionTag` = `'BMS2'` (feste Adresse; danach `CAN_TX_BMS_Status_Reporter`) |
| `0x0801A99C` | `g_crc8Poly07Table` (CRC-8, Polynom 0x07, SH36735-Frames) |
| `0x0801B860` | Keil-Regionstabelle (2 Regionen), `0x0801B880` `shellCommandTable` (52 × 16 B), `0x0801BBC0` Ladeabbild von `.data` |
| `0x0801E000` | OTA-Staging-Kopie `g_otaStagingImage` (Kennungswort `0x0802B800` muss `'BMS2'` sein) |
| `0x20002800`–`0x20002B97` | `.data` (0x398 B; 1105: 0x3A0 B) |
| `0x20002B98`–`0x2000C957` | `.bss` (0x9DC0 B); FreeRTOS-`ucHeap` bei `0x200046D0`, Keil-Heap `0x2000995C`–`0x2000A957` |
| `0x2000C958` | `__initial_sp` (Stack-Ende; 1105: `0x2000C960`) |

Das RAM-Layout entspricht 1105 ohne die beiden nur in 1105 vorhandenen Variablen `g_uvpLvl2ShutdownTimerS` und
`g_pvRescueRequestFlag` (die Adressen dahinter sind um bis zu 8 Bytes verschoben, z. B. `ucHeap` 0x200046D8 →
0x200046D0). Die RAM-Namen wurden über die Ausrichtung der Literal-Pool-Referenzen gepaarter Funktionen übertragen
(693 Namen) plus Interpolation (2).

### 2.3 Bearbeitungsstand in Ghidra

- Programm `/VNSA/BMS/BMS_110_VNSA-0_VABMS_APP_110_ota_163748.bin`: **697 Funktionen, alle benannt**, jede Funktion hat
  einen benutzerdefinierten Prototyp, Plate-Kommentare, 0 `FUN_*`, 0 Duplikate.
- Übertragung von 1105: Offline-Segmentabgleich (Pointer/BL maskiert, 350 Segmente, 92,6 % der Bytes; erweitert auf 29
  gleich große Bereiche), Funktionslayout und Markup kopiert, 697 Funktionspaare (680 maskiert identisch inkl. lokaler
  Variablen), 7 kleine Funktionen vom Linker verschoben (z. B. `SysTick_Handler`, `HAL_NoOpStub`, `CAN_EchoTest_Poll`),
  4 Paare von Hand gesetzt (`OwnPack_MOSFET_Driver_MasterVariant`, `Pack_LowVoltage_Balancer`, `CAN_RX_Handler`,
  `Master_Build_ChargeCurrentLimit`). Peripherie-/Systemblöcke über die Adresse kopiert. Flow- und
  Fall-through-Overrides (17) sowie Float/Double-Pooltypen übertragen; ARM-Konstantenreferenzen je Funktion neu erzeugt.
- Geänderte Funktionen gegen 1105 und D 118 geprüft; Plates neu geschrieben; `SOC_Select_Threshold_2420_2800`
  umbenannt in `SOC_Select_Threshold_2420_2750` (Code = D), RAM `0x20002858` heißt `g_protCommTickCounter200`
  (D-Name und -Verwendung; 1105 nutzt die Adresse als `g_standaloneModeFlag`).
- Gates (`vna110_bms_gates.py`, dieselben Prüfungen wie bei 1105): 0 undefinierte Bytes, 0 verwaiste Befehle, 0 Flüsse
  in Daten, 0 Funktionszeiger auf Nicht-Code, 0 reine Literal-Funktionen, 0 als Code dekodierte Literal-Pools,
  0 Aufrufe in Funktionsrümpfe, 0 unbenannte referenzierte RAM-Globals, 0 Standard-Prototypen, Tasknamen =
  `xTaskCreate`-Namensstrings, No-Return-Flags plausibel, 0 doppelte Labels. Dekompilat-Export: 697 Funktionen,
  **0 Fehler**.

## 3. FreeRTOS-Tasks

`RTOS_CreateAllTasks` (`0x0800E8FC`) legt dieselben 10 Tasks an wie 1105/D (Namen, Stacks, Prioritäten unverändert):

| Task | Einsprung | Stack (Worte) | Prio |
|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800EB88` | 0x80 | 5 |
| `RTOS_vTaskKeyScan` | `0x0800EB68` | 0x80 | 9 |
| `RTOS_vTaskKA495XX` (SH36735-Worker) | `0x0800EB50` | 0x80 | 8 |
| `RTOS_vTaskCanRec` | `0x0800EAF0` | 0x180 | 4 |
| `RTOS_vTaskCanSend` | `0x0800EB00` | 0x80 | 3 |
| `RTOS_vTaskRS485Comm` | `0x0800EBC2` | 0x180 | 1 |
| `RTOS_vTaskSocAlgorithm` | `0x0800EBD0` | 0x100 | 7 |
| `RTOS_vTaskProtectCheck` | `0x0800EBAC` | 0x100 | 6 |
| `RTOS_vTaskControlLogic` | `0x0800EB0A` | 0x80 | 2 |
| `RTOS_vTaskDataSave` | `0x0800EB24` | 0x80 | 0 |

Wie bei 1105 wird der IWDG von der Anwendung nur bedient, nie konfiguriert oder gestartet.

## 4. Schnittstellen

### 4.1 CAN

Frame-Satz, IDs und Aufbau wie bei 1105 (siehe `VNSA_BMS_FW_v1105_Vollanalyse_de.md` §4.1). Unterschiede in 110:

| ID | 110 |
|---|---|
| `0x1803AA01` (`CAN_TX_PF3_ChargeDischargeLimits` `0x08007C1C`) | Ladespannungsgrenze **46,8 V** (fest, zugleich PF3-Begrenzung), Ladestrom = Systemgrenze **≤ 40,0 A** (25,0 A bei Pack-Fehler, **keine 10,0-A-Begrenzung** unter 35,00 V), Entladestrom **≤ 40,0 A**, Statusbits unverändert |
| `0x1804AA01` (`CAN_TX_PF4_ProtectWarnings`) | Byte 4 `g_chargeStatusCode` = nur das beim Start aus EEPROM 0xFC02 gelesene Byte (0xA5 nach der ersten Vollladung); 1105 sendet zusätzlich 2/3/4 |
| `0x1820..0x1841AA00+n` (`CAN_TX_PerPack_10Msgs` `0x08008338`) | Packs 1..6 (`pack_id != 0 && pack_id < 7`); 0x1821 enthält den eigenen max. Ladestrom des Packs (kein 10,0-A-Überschreiben); 0x1823 FW-Kennung **1100** |
| `0x03410110` (PV-Rettungsanforderung des Wechselrichters) | **nicht behandelt** (1105: `g_pvRescueRequestFlag`) |
| `0x100201AA` | SOC-Reset über `CAN_RX_Handler` → `CAN_CMD_02_Handler` (1105 dekodiert die ID zusätzlich in `CAN_RX_System_Command_Handler`) |

Die Systembefehle auf `0x100101AA` (inkl. 0x0A CAN-Neustart, 0x0B Master werden, 0x0C Slave werden), die Antwort auf
den Heartbeat `0x100501AA` (`CAN_TX_OwnPackReport`), der zeitversetzte Slave-Statusbericht und die
`0xA5A5`-Keepalive-/Echo-Frames sind unverändert.

### 4.2 RS485

Unverändert gegenüber 1105 (ein Bus an USART1, 9600 Baud, `RS485_Dispatcher` `0x08010EB4`, Versionsbyte `'n'` = 0x6E,
13 Zellspannungen in Opcode 0x19).

### 4.3 Debug-Shell (letter-shell)

Unverändert: Benutzer `BLUE`, 52 Tabelleneinträge (10 Tastenbelegungen, `help`, `clear`, 39 Befehle). Das Login-Banner
zeigt `Version: 110.0` und das Build-Datum.

### 4.4 EEPROM-Belegung (I2C)

Unverändert gegenüber 1105: Konfigurationsblock bei 600 mit Versionskennung **`'n'`** (A 109: `'m'`; bei abweichender
Kennung lädt `BMS_Config_Params_Load_Or_Default` die Standardwerte), Master-Marker `0x01F6`, Zähler `0x0254`,
Ereignisprotokoll `0x2000`/`0x3000`, Fehlersperre `0xFC00`, Vollladungs-Marker `0xFC02`.

## 5. Zellüberwachung — SH36735

Unverändert gegenüber 1105: SH36735-AFE an SPI1, CRC-8-Polynom 0x07, **13 Zellen** (Konfigurationsregister 0x43,
`INIT_SH36735_Subsystem` `0x0800D680`), Balancing-Regeln, Nullpunktkalibrierung („SELF"-Flag).

## 6. Pack-Verwaltung

Master/Slave-Modell, Auswahlverfahren, Ausschlussmaske, dauerhafter Master-Marker wie bei 1105. Unterschiede:
`Pack_LowVoltage_Balancer` (`0x08007EAC`) nimmt nur Packs mit SOC < 8 % oder Protect1 Bit0 in die Rotation (kein
Kriterium „minimale Zelle < 2800 mV"); `OwnPack_MOSFET_Driver_MasterVariant` (`0x08007150`) hält die Relais bei nicht
gesetztem SELF-Flag bei jeder Packspannung aus (1105: nur oberhalb 39,00 V).

## 7. SOC/SOH

Wie bei 1105, außer der Leer-Schwelle: `SOC_Select_Threshold_2420_2750` (`0x08013474`) liefert bei ≥ 0 °C **2750 mV**
(unter 0 °C 2420 mV), den Venus-D-Wert (1105: 2800 mV). Die erste Vollerkennung schreibt weiterhin den EEPROM-Marker
`0xFC02 = 0xA5`; die Drosselung „nie voll geladen" ist vorhanden (auch in A 109).

## 8. Lade-/Entladegrenzen

| Größe | Venus A 110 | Venus A 110.5 | Venus D 118 |
|---|---|---|---|
| Grenze System-Ladestrom (`g_sysChargeCurrentLimit_x10`, Modbus 32106) | **40,0 A** | 50,0 A | 50,0 A |
| System-Ladestrom außerhalb Master-Laufzustand 2 | 0 | 50,0 A (0 bei gesetztem SELF-Flag) | 0 |
| Grenze System-Entladestrom (Modbus 32107) | **40,0 A** | 50,0 A | 50,0 A |
| Max. Ladestrom je Pack (`Max_Charge_Current_Calc` `0x08013858`) | Tabelle, max. 50,0 A; solange nie voll geladen 50,0 A unter 45 °C, darüber 25,0 A − 1,67 A/°C | gleich | nur Tabelle |
| Entladestrom je Pack (`Max_Discharge_Current_Calc` `0x080139AC`) | Tabelle, max. 50,0 A | gleich | gleich |
| Ladespannungsgrenze (Modbus 35110) | **fest 46,8 V** | Packspannung + 2,0 V, max. 48,0 V, CAN-Begrenzung 46,8 V | 57,6 V, CAN-Begrenzung 60,9 V |
| CAN-Ladestrom bei eigener Packspannung unter 35,00 V | keine Begrenzung | 10,0 A | keine Begrenzung |
| Entladespannungsgrenze | 39,0 V | 39,0 V | 48,0 V |

Die Systemgrenze ist die Grenze des aktiven Packs, begrenzt auf 40,0 A (`Master_Build_ChargeCurrentLimit`
`0x0800BC20`, `Master_Build_DischargeCurrentLimit` `0x0800BC90`); das Control rechnet sie in Leistung um
(Ladebegrenzung ≈ U × I).

## 9. Schutzfunktionen

Konfigurations-Standardwerte unverändert. Code gegenüber 1105:

- `Cell_OVP_UVP_TempCompensated_Protection` (`0x0800CD68`): UVP Stufe 2, Stufe A (≥ 0 °C) **3000 mV**, eine Auslösung
  setzt nur `g_protState5` Bit2 (Warnung); Stufe B (2500 mV) setzt Protect1 Bit5, löscht die Zähler und setzt den
  SOC-Schätzer auf 0 %. Sensorfehler minimale Zelle (< 1500 mV) bei jeder Packspannung. (= Verhalten Venus D.)
- `BMS_ProtectionAndCommTick` (`0x0800963C`): solange Zell-UVP Stufe 2 aktiv ist, werden die eigenen MOSFETs nur unter
  2,0 A Packstrom freigegeben und nur solange `g_protCommFlagA` (PB1-Ausgang low und PE9-Eingang high) gelöscht ist;
  ein alleinstehendes Pack (Slot 0xFF, oder Slot 1 als einziges Pack) geht nach 200 s in Fehlermodus 3. Kein Warten von
  120 s auf eine Anforderung des Wechselrichters und kein STOP-Modus aus diesem Pfad (1105:
  `System_Reset_And_Halt_WithCurrentCheck` nach 120 s ohne `0x03410110`).
- `HW_OvercurrentAndSensorProtection` (`0x0800B09C`): Prüffenster des Komparators **±3,0 A** (1105: ±70,0 A).
- Unverändert: Protect2-Bits `0x400`/`0x2000`, Fehlermasken, `System_Reset_And_Halt_WithCurrentCheck`.

## 10. OTA/Update

Unverändert: CAN `BMS_OTA_Flash_Writer` (`0x0800D23C`, Opcode 0xCE) und `RS485_OTA_Flash_Handler` (`0x0800CA28`),
Staging-Kopie `0x0801E000`, CRC16, Kennung `'BMS2'`, Reparatur der Boot-Seite. Ein 110-Image wird von einem A-BMS mit
109 oder 1105 angenommen und umgekehrt (gleiche Kennung); D-Images (`'BMS1'`) werden abgelehnt.

## 11. Sicherheitshinweise

Unverändert gegenüber 1105 (Shell-Befehle auf dem RS485-Debugpfad, nicht authentifizierte CAN-Befehle für Master-Rolle,
Fehlersperre, Werksreset und RTC, IWDG nicht gestartet).

## 12. Zusammenspiel mit dem Venus-A-Wechselrichter (Micro 1211) und Control 151

- Der 1105-Changelog verlangte Wechselrichter-Firmware ≥ 119.3 für den PV-Rettungsstart. BMS 110 hat dafür **keine**
  BMS-Seite: Frames `0x03410110` vom Wechselrichter werden ignoriert, und das BMS durchläuft in UVP Stufe 2 nicht die
  120-s-/STOP-Sequenz. Ob die Rettung des A-Micro 1211 mit BMS 110 noch funktioniert, hängt vom Micro-Verhalten ab
  (nicht getestet).
- Control: Systemgrenzen ≤ 40,0 A (Modbus 32106/32107), konstante Ladespannungsgrenze 46,8 V (35110), BMS-Version 1100
  (30204/34010/37012).

## 13. Offene Punkte

1. Warum der Hersteller 110 auf Basis von 109 statt 1105 veröffentlicht hat, lässt sich aus dem Code nicht ableiten. Ob
   eine spätere Version beide Zweige zusammenführt, ist mit dem nächsten Image zu prüfen.
2. Tatsächliches Verhalten der 40-A-Grenze im Feld (CAN-/Modbus-Werte mit BMS 110) nicht gemessen.
3. Die offenen Punkte 2–4 der 1105-Analyse (Schreiber der in `0x1851/0x1852` gesendeten Worte, SH36735-Register,
   Polarität des SELF-Flags) gelten unverändert (gleicher Code).
4. Von 1105 übernommene, vermutlich falsche Benennung, aus Gründen der Versionskonsistenz beibehalten:
   `g_cliInsertCharEscSeqPtr` (`0x20002B34`) enthält den Zeiger auf das Login-Banner, den `CLI_LoginUser` verwendet.
