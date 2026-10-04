# BMS-FW v117.7 (Beta, Venus D) — Vollanalyse

> 🇬🇧 English version: [VNSD_BMS_FW_v1177_Full_Analysis.md](VNSD_BMS_FW_v1177_Full_Analysis.md)

**Stand:** 2026-09-30.

**Englisch:** `VNSD_BMS_FW_v1177_Vollanalyse_de.md` (wird nach dieser Fassung erstellt) ·
**Diff zum Nachfolger v118:** `VNSD_BMS_Diff_v1177_v118_de.md` ·
**Vollanalyse v118:** `VNSD_BMS_FW_v118_Vollanalyse_de.md` (wird parallel von anderer Stelle erstellt —
hier nur verlinkt, nicht editiert) ·
**Verwandt:** `VNSD_Ladegrenzen_Ladeleistung_de.md` (Ladegrenzen BMS/Control/Micro, verifiziert) ·
**Alte Analyse (Hintergrund, Basisadresse falsch — s. §13):** `VNSD_BMS_FW_Analyse_v117.7_de.md` ·
**Dekompilat/Listing:** `Dekompilate/VNSD_BMS_1177/` (lokal, nicht im Repo) (710 Funktionen, `listing/` mit Flash-Rohdisassemblat
+ `globals_and_registers.txt`)

## 0. Kurzfassung

BMS-Firmware **v117.7**, als **Beta** gekennzeichnet (Changelog `CHANGELOG.de.md` (lokal, nicht im Repo): "v117.7 (Beta) —
2026-08-12", Einträge "OTA-Test" / "7"). Basisadresse **`0x08002800`** (Bootloader darunter),
Image-Größe `0x1A000` B (106 496 B), MCU **Nations N32G45x** (Cortex-M4F), Keil ARMCC, FreeRTOS
(`heap_4`, 10 Tasks), letter-shell (36 Kommandos, Standardnutzer `BLUE`), FatFs, KA495xx-Zellmonitor-AFE
über SPI mit CRC-8.

**Verhältnis zu v118:** 117.7 ist im Firmware-Archiv der **einzige vorhandene Vorgänger** von v118 —
eine Beta von der (nicht archivierten) stabilen Version v117. Der Code ist in weiten Teilen mit v118
identisch: von 711 (118) bzw. 710 (117.7) Funktionen sind **622 byte-für-byte identisch** (nach
Adressnormierung) und **77 weitere** unterscheiden sich nur durch Relozierung/Decompiler-Artefakte
(keine Verhaltensänderung). Nur **11 Funktionen plus 1 komplett neue Funktion** (`SOC_
OcvBoundsRecalibrateOnFlag`, nur in 118) enthalten reale Änderungen — Details und vollständige
Fundstellenliste in `VNSD_BMS_Diff_v1177_v118_de.md`. **Diese Vollanalyse beschreibt ausschließlich den
117.7-Code** mit seinen eigenen Adressen; wo der Inhalt mit 118 identisch ist, wird das vermerkt, aber
stets mit der 117.7-Adresse belegt.

Die wichtigsten Punkte, in denen 117.7 sich von 118 unterscheidet (Details: Diff-Dokument):

- `PackSelect_DischargeStepThreshold` (`0x08009354`) leitet die Entlade-Rotationsschwelle aus einer
  **festen** Stufentabelle ab und ignoriert den bereits vorhandenen konfigurierbaren Entladelimit-
  Prozentsatz `g_dischargeLimitPct` komplett — in 118 wird diese Formel durch eine vom Limit abhängige
  ersetzt.
- Der periodische CAN-Heartbeat-Frame (`CAN_RX_Handler`, `0x080081dc`) aktualisiert
  `g_dischargeLimitPct` **nicht** (nur das dedizierte CAN-Kommando `0x100301aa` tut das).
- Keine Funktion `SOC_OcvBoundsRecalibrateOnFlag`; das dafür in 118 neu konsumierte Flag
  `g_socOcvRecalibrateRequest` (`0x200029ca`) existiert in 117.7 bereits im RAM, wird aber ebenfalls
  von keiner Funktion beschrieben — ungenutzt in beiden Versionen.
- Config-Versions-Tag `g_cfgVersionTag` = `'u'` (0x75, entspricht "117"), nicht `'v'`.
- Gemeldeter Pack-FW-Versionswert `g_pack0FwVersionTag` = `0x499` (1177 dezimal, Beta-Zählweise
  "117.7 × 10"), nicht `118`.
- Kein Fehler-Zeitstempel-Log bei `System_Reset_And_Halt_WithCurrentCheck` (`0x08004998`) — die
  Leistungsprüfung vor dem Reset fehlt.
- `PackMode_Shutdown_Handler` (`0x0800f324`) setzt `g_pendingActivationFlag` sofort beim Eintritt in
  die Shutdown-Sequenz, nicht erst nach den 300 Ticks Wartezeit.
- RAM: `.bss` `0x20002bc4`–`0x2000cb8f` (40 908 B, **16 B kleiner** als 118), FreeRTOS-Heap **`0x4FF8`**
  (8 B größer als 118s `0x4FF0`), Initial-SP `0x2000CB90` (118: `0x2000CBA0`, exakt +16 B).

## 0a. Prüfliste gegenüber v118

| # | Frage | Antwort 117.7 | Beleg |
|---|---|---|---|
| 1 | CAN-Protokoll wie in 118? | Kommandosatz/Message-IDs identisch. Einzige Abweichung: der periodische 0xfd00-Heartbeat aktualisiert `g_dischargeLimitPct` **nicht** (dieser Pfad existiert erst ab 118). | §4.1, Diff §2.1 |
| 2 | RS485 wie in 118? | Dispatcher-Struktur/Opcodes identisch; Versions-Antwortbyte `0x75` statt `0x76`. | §4.2, Diff §2.6 |
| 3 | Lade-/Entladegrenzen wie in 118? | Harte Grenzen (50 A, 57,6 V/48,0 V, Temperatur×SOC-Matrix) identisch. Die Rotationsschwellen-Formel in `PackSelect_DischargeStepThreshold` unterscheidet sich (feste Tabelle statt limit-abhängiger Formel). | §8, Diff §2.1 |
| 4 | AFE/KA495xx-Treiber wie in 118? | Identisch (nur Adressverschiebung). | §5 |
| 5 | OTA-/Bootloader-Handshake wie in 118? | Identisch: Basis `0x08002800`, Tag `'BMS1'` @ `0x08010000`, Staging @ `0x0801E000`, Tag-Prüfung @ `0x0802B800`. | §10 |
| 6 | Shell-Kommandos wie in 118? | Identisch (36 Einträge, Nutzer `BLUE`). | §11 |
| 7 | RAM-Layout wie in 118? | `.bss` 16 B kleiner, Heap 8 B größer, sechs Pack-/Rotations-Globals ab `0x20002a58` liegen 2 B "früher" als in 118 (kein `g_dischargeStepRotateLimitCache`). | §2, Diff §2.5 |

## 1. Binary-Fingerprint

| Merkmal | Wert |
|---|---|
| Datei | `1177_bms_VNSD-0_20251010135647565eb2036.bin` |
| Größe | 106 496 Byte (`0x1A000`) |
| SHA-256 | `69b18aa42831536f01546bbe707e36ee65b6faa84e6cf4e92f9fbfe237f67c77` (per `shasum -a 256` verifiziert) |
| Link-Adresse | `0x08002800`–`0x0801C7FF` (Bootloader unterhalb `0x08002800`) |
| MCU | Nations N32G45x (Cortex-M4F, Thumb-2); ADC1/ADC2 @ `0x40020800`/`0x40020C00`, UID @ `0x1FFFF7F0`, 102 Vektoren (86 IRQ + 16) |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW-Segment) |
| Bibliotheken | FreeRTOS (`heap_4`), letter-shell, FatFs |
| Initial-SP | `0x2000CB90` (aus Byte 0–3 des Images) |
| Reset-Vektor | `0x08002A6D` (Thumb) → `Reset_Handler` (`0x08002a6c`) |
| Versionskonstante | `'BMS1'` (`0x31534D42`) @ `0x08010000` |
| OTA-Staging | Kopie @ `0x0801E000` (außerhalb des laufenden Images), Tag-Prüfung @ `0x0802B800` (`g_otaStagingVersionTag`) |
| Herstell-Changelog | v117.7 (Beta) — 2026-08-12: "OTA-Test", "7" (Issue #19) |

**Scatter-Load:** `.data` (`0x08002800`+Offset, LZ77-komprimiert, 0x3C4 B) → `0x20002800`; `.bss`
(`__scatterload_zeroinit`) → `0x20002bc4`–`0x2000cb8f` (40 908 B); FreeRTOS-Heap `ucHeap` @ `0x200050d8`,
Größe `0x4FF8` (20 472 B). Initial-SP `0x2000CB90` liegt exakt am Ende dieses Speicherbereichs.

## 2. MCU / Speicherkarte / Ghidra-Status

**Prüfstand** (`Dekompilate/VNSD_BMS_1177/_index.md`):

| Prüfung | Ergebnis |
|---|---|
| Funktionen | **710**, davon 710 benannt, 0 `FUN_*` |
| Ghidra-Symbole | 3761 |
| Dekompilation fehlgeschlagen | 0 |
| Vergleich zu 118 | 711 Funktionen (118) = 710 (117.7) + 1 neue (`SOC_OcvBoundsRecalibrateOnFlag`) |

Alle Verifikations-Gates (vollständige Benennung, keine fehlgeschlagenen Dekompilationen, keine
verwaisten `FUN_*`-Reste) sind bestanden — entsprechend dem für v118 etablierten Prüfschema. Die
Benennung wurde **aus v118 abgeleitet**: 622 der 710 Funktionen sind byte-identisch zu ihrem
v118-Gegenstück (adressnormiert) und übernehmen dessen bereits verifizierten Namen unverändert; die
übrigen 88 wurden einzeln anhand des tatsächlichen 117.7-Codes neu geprüft (s. Diff-Dokument, §1).

**RAM-/Peripherie-Blöcke** (`listing/globals_and_registers.txt`):

| Block | Bereich | Größe |
|---|---|---|
| `.data` | `0x20002800`–`0x20002bc3` | `0x3C4` B (964 B) |
| `.bss` | `0x20002bc4`–`0x2000cb8f` | `0x9fcc` B (40 908 B) — **16 B kleiner als 118** (`0x9fdc`) |
| FreeRTOS-Heap (`ucHeap`) | `0x200050d8`ff. | `0x4FF8` B (20 472 B) — **8 B größer als 118** (`0x4FF0`) |
| `flash_upper` (Reserve/OTA-Staging) | `0x0801c800`–`0x0803ffff` | enthält `g_otaStagingVersionTag` @ `0x0802b800` |
| `sysmem_info` | `0x1ffff000`–`0x1ffff7ff` | UID @ `0x1FFFF7F0`, Alt-UID @ `0x1FFFF270`, Fabrikwort @ `0x1FFFF298` |

Initial-SP `0x2000CB90` = Ende von `.bss` + 1, exakt konsistent mit der Scatter-Load-Rechnung.

## 3. FreeRTOS-Tasks

Aus `RTOS_CreateAllTasks` (`0x0800e750`, 212 B) — zehn direkte `xTaskCreate()`-Aufrufe:

| Task | Entry-Adresse | Stack (Wörter) | Priorität | Handle |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800e9c5` (Thumb) | `0x80` | 5 | `0x20002b08` |
| `RTOS_vTaskKeyScan` | `0x0800e9ab` | `0x80` | 9 | `0x20002b0c` |
| `RTOS_vTaskKA495XX` | `0x0800e99d` | `0x80` | 8 | `g_ka495xxTaskHandle` |
| `RTOS_vTaskCanRec` | `0x0800e945` | `0x180` | 4 | `0x20002b14` |
| `RTOS_vTaskCanSend` | `0x0800e955` | `0x80` | 3 | `0x20002b18` |
| `RTOS_vTaskRS485Comm` | `0x0800e9fb` | `0x180` | 1 | `0x20002b1c` |
| `RTOS_vTaskSocAlgorithm` | `0x0800ea09` | `0x100` | 7 | `0x20002b20` |
| `RTOS_vTaskProtectCheck` | `0x0800e9e5` | `0x100` | 6 | `0x20002b24` |
| `RTOS_vTaskControlLogic` | `0x0800e95f` | `0x80` | 2 | `0x20002b28` |
| `RTOS_vTaskDataSave` | `0x0800e979` | `0x80` | 0 | `0x20002b2c` |

Byte-für-byte identisch zu 118 bis auf die zehn relozierten Einstiegsadressen (verifiziert, s.
Diff-Dokument §1). `RTOS_vTaskKA495XX` ruft `KA495XX_TaskStateMachine` (`0x0800b1e0`) alle 5 Ticks auf.
`RTOS_vTaskSocAlgorithm` treibt `SOC_AlgorithmOrchestrator` (§7). IDLE-Task und Scheduler-Start folgen
dem Standard-FreeRTOS-Muster (`vTaskStartScheduler` `0x080192dc` → `xPortStartScheduler` `0x08019530`).

## 4. Schnittstellen

### 4.1 CAN

`CAN1_Init_And_CreateQueues` (`0x08003b88`, 284 B) initialisiert bxCAN (reines CAN1, kein I2C-Anteil
trotz früherer Fehlbenennung, s. Namens-Notiz in der Naming-Evidenz). `CAN_RX_Handler` (`0x080081dc`,
104 B) dekodiert die erweiterte Arbitration-ID:

| Kommando/Bereich | Bedeutung |
|---|---|
| `0x100101aa` | → `CAN_CMD_01_Handler` |
| `0x100201aa` | → `CAN_CMD_02_Handler` |
| `0x100301aa` | Entladelimit-Prozent setzen: `g_dischargeLimitPct = payload[0]`, persistiert via `APP_WriteFlash_DischargeLimitPct` |
| `0x100401aa` | → `CAN_CMD_04_Handler` |
| `0x100501aa` | Payload `0x01` → `g_canRxRestartFlag_a70 = 1` |
| `0x180101aa`/`0x180102aa` | → `CAN_TX_Orchestrator` (Pack-Abfrage, Broadcast/gezielt) |
| Klasse `0xfd00` (Heartbeat/Master-Enable) | `g_masterEnableReceivedFlag` (bei Payload-Byte `0x01`), `g_lastCanCmd04PayloadFlags`, `g_packOnlineWatchdogTimerMs=0`, `CAN_TX_Orchestrator(0xff)`. **Aktualisiert `g_dischargeLimitPct` in 117.7 nicht** (erst ab 118, s. Diff §2.1). |

`CAN_RX_Command_Dispatcher` (`0x0800f0bc`, 324 B) behandelt einen zweiten ID-Bereich (Opcode im
untersten Byte, u. a. `0xCE` = Fern-Reset/OTA-nah, `0x01`–`0x0A` Status-/Modus-Kommandos).
`CAN_CMD_Pack_Activation` (`0x0800df04`, 396 B) behandelt Pack-Aktivierung/-Deaktivierung; Opcode `0xCE`
mit Sub-ID 0 armiert die OTA-Übernahme (`g_otaFinalizeRequestedFlag`-artige Flags), Sub-ID 4 routet zu
`CAN_RxSlavePackTelemetry_Unpack` (`0x0800db88`, 311 B, Pack-Status-Bitfelder). Antworten laufen über
`CAN_TX_Orchestrator`, `CAN_TX_PF2_Capacity` (`0x08007bd8`, Kapazität) und `CAN_TX_PF3_
ChargeDischargeLimits` (`0x08007c74`, 318 B, Lade-/Entladelimit-Broadcast, s. §8).

### 4.2 RS485

`RS485_Dispatcher` (`0x08010004`, 717 B) — Opcode-basiertes Protokoll auf USART2
(`0x40004400` → `0x20002BC4`-Ringpuffer, `RS485_USART2_Init_DMA_RingBuffer`, `0x0800518c`). Notierte
Opcode-Befunde: `0x29`/`0x2e`/`0x30` kopieren große Pack-Struct-Blöcke (`0x60`/`0x54`/`0x2c` B) unter
angehobener Basispriorität (interrupt-sensitive Bulk-Transfers); `0x34` pulst das Relais-Pin (GPIOB
`0x1000`) für 80 ms; `0x2` löst dieselbe DSB/AIRCR-Reset-Sequenz wie `HAL_NVIC_SystemReset` aus.
Versions-Antwortbyte im Puffer = `0x75` (117, s. §0). `RS485_Register_Read_Handler` (`0x08009f78`,
1438 B) und `RS485_Register_Write_Handler` (`0x0800a524`, 758 B) sind gruppen-ID-basierte
Config-Parameter-Zugriffe (Gruppen 1:1 zwischen Lese-/Schreibpfad); nicht zugeordnete Gruppen-IDs
fallen auf `RS485_RegisterWrite_DefaultGroup` (`0x0800a62a`) zurück. `RS485_Pack_Telemetry_Response`
(`0x080123a4`, 120 B, Opcode `0x2e`) liefert den Pack-Telemetrie-Record inkl. Versionsbyte `'u'` an
Offset `0x48`. `RS485_OTA_Flash_Handler` (`0x0800c9d8`, 728 B) — s. §10.

### 4.3 Debug-Shell (letter-shell)

`shellAdd` (`0x08013e10`) registriert die letter-shell-Instanz; `CLI_ShellInit` (`0x0801474c`, 60 B)
meldet den Standardnutzer **`BLUE`** an (`CLI_LoginUser`, `0x08014c60`) und bestätigt eine
Kommandotabelle mit **36 Einträgen** (Naming-Evidenz, `shellAdd`-Notiz: "Confirms the shell command
table has 36 entries"). `CLI_ValidateLoginPassword` (`0x08013e48`, 52 B) prüft das Passwort — laut
Diff-Artefakt im Literal-Pool ist das hinterlegte Passwort leer (nicht in dieser Sitzung erneut
dekodiert, s. §12). `CLI_ProcessCommandLine`/`CLI_RunExtendedCommand`/`CLI_ResolveVariableReference`
implementieren letter-shell-typische Kommandoverarbeitung inkl. `$variable`-Auflösung und
Tab-Completion. CAN-/RS485-nahe Test-Kommandos existieren (`CLI_Cmd_SendTestCanFrame`,
`RS485_SetDebugCheckoutFlag`, `RS485_SetDebugCmdPendingFlag`).

### 4.4 I2C-EEPROM

`I2C_EEPROM_Write_WithMutex` (`0x08008c18`, 84 B, bitgebanged I2C, kein Hardware-I2C-Peripherie-Zugriff)
ist der zentrale Schreibpfad, u. a. für SOC-Kalibrierung (§7) und (ab 118) Fehler-/Ereignis-Zeitstempel
auf Adresse `0xfc02` (10 B, RTC-Datum/Zeit aus `RTC_Read_DateTime`, `0x080076ce`). `SOC_
EepromLoadCalibration` (`0x08015e9c`, 204 B) und `SOC_SaveCalibrationIfChanged`/`SOC_
ReadAndValidateFlashCalibSlot` verwalten einen wear-leveled Log-Bereich mit 50 Slots à 20 B
(1000 B gesamt) für die SOC-Kalibrierdaten.

## 5. KA495xx-Zellmonitor (AFE)

16 Zellen, 5 NTC-Kanäle, SPI mit CRC-8 (Polynom `0xD5`, Tabelle `g_ka495xxCrc8Table`, Prüffunktion
`KA495XX_Crc8` @ `0x08016a58`, 16 B). Treiber-Cluster (alle 117.7-Adressen): `INIT_KA495XX_SPI_GPIO`
(`0x08007960`), `KA495XX_SPI_Read_All_Registers` (`0x0800798c`), `KA495XX_SPI_Write_7Registers`
(`0x080079b8`), `KA495XX_SPI_Read_Register`/`_Write_Register_2` (`0x08007abc`/`0x08007aec`),
`KA495XX_TaskStateMachine` (`0x0800b1e0`, 156 B, Poll-Zustandsautomat der RTOS-Task), `KA495XX_
VerifyConfigRegisterTable` (`0x080166c4`, 60 B), `KA495XX_CellBalanceChannelConfig` (`0x08016974`,
126 B — liefert 0 zurück, wenn die konfigurierte Zellenzahl außerhalb 4–17 liegt).

Zellausgleich (`KA495XX_CellBalanceDecision`): Start bei Zellspannung ≥ 3501 mV **und**
(Zelle − Minimum) ≥ 30 mV; Stopp bei < 3500 mV **und** Differenz < 20 mV; globaler Abbruch bei
(Max − Min) < 30 mV oder Strom < 1001 (roh). `KA495XX_ReadRegister` prüft eine Checksumme über
`HAL_ShortBusyWaitLoop`, deren genauer Algorithmus im Dekompilat nicht rekonstruierbar ist (Offener
Punkt, §12). Alle KA495xx-Funktionen sind zwischen 117.7 und 118 **identisch** bis auf Adressverschiebung
(Diff-Dokument, Tabelle §1).

## 6. Pack-Management

**Rotation/Auswahl:** `PackSelect_ModeBySoc` (`0x080092cc`, 124 B) wählt je nach SOC-Zustand
(voll/normal/niedrig/leer) einen der Zweige `PackSelect_Mode1_Full` (`0x08006174`), `_Mode2_LowSoc`
(`0x080062b8`), `_Mode3_Empty` (`0x08006008`). `PackSelect_Discharge_StepRotate` (`0x080063a0`, 91 B)
und `PackSelect_DischargeStepThreshold` (`0x08009354`, 26 B, Formel s. §8/Diff §2.1) steuern den
Wechsel des aktiven Entlade-Packs anhand einer gleitenden SOC-Schwelle; nach ~50 Ticks unterhalb der
Schwelle wird das Pack mit der höchsten Spannung neu ermittelt (`PackSelect_Next_
AboveSocThreshold`, `0x08007174`) und ggf. übergeben (`PackSelect_CommitPendingRotation`,
`0x0800bca8`). `PackHandoff_StateMachine` (`0x080066dc`, 354 B) setzt die Lade-MOSFET-Maske auf 0 oder
exakt ein Bit — **physisch lädt immer nur ein Pack** (s. §8). `OwnPack_MOSFET_Driver`/`
_MasterVariant` (`0x08006b60`/`0x08007220`) schalten den Lade-MOSFET entsprechend dieser Maske.

**Recovery/Zustand:** `PackRecovery_LowCell_Charge_SM` (`0x0801070c`, 513 B) behandelt Packs mit
kritisch niedriger Zellspannung in einem eigenen Lade-Zustandsautomaten. `PackMode_Dispatcher`
(`0x08009778`, 138 B) routet zwischen Aktivierungs-, Übergangs- (`PackMode_Transitional_Handler`) und
Shutdown-Zuständen (`PackMode_Shutdown_Handler`, `0x0800f324`, 81 B — dreistufig: GPIOB-Pin/Zähler
zurücksetzen → 300 Ticks warten (setzt danach in 118, aber bereits in 117.7 **sofort** beim Eintritt
`g_pendingActivationFlag`) → periodischer Off-Broadcast → `System_Reset_And_Halt_
WithCurrentCheck`).

**Kommunikation/Relais:** `PackSelect_RelayCommStateMachine` (`0x0800b298`, 514 B) und
`PackSelect_TriggerWakePulse` (`0x080155a4`) steuern eine RS485-/Relais-Handshake-Leitung zwischen
Packs; genaue Protokollsemantik nicht vollständig rekonstruiert (Offener Punkt, §12).
`BMS_AggregateSystemPackStats` (`0x0800b834`, 526 B) räumt Protect-/Status-Felder ausgefallener Packs
auf. `PerPack_Struct_Builder` (`0x0800c770`, 580 B) baut den pro-Pack-Telemetrie-Record (Zellspannungen,
NTC-Extrema, SOC, Strom, MOS-Temperaturen, Relaisstatus, Zyklenzahl, Protect-/Warnwörter,
Versionsfeld `g_pack0FwVersionTag = 0x499` = 1177, s. §0).

## 7. SOC / SOH

`SOC_AlgorithmOrchestrator` (`0x08015d20`, 157 Instruktionen/346 B) läuft in `RTOS_vTaskSocAlgorithm`
und orchestriert je Tick: `SOC_State_Detect` (`0x080121d8`) → Modusermittlung (1=Laden, 2=Entladen,
3=Leerlauf) → `Coulomb_Counting` (`0x08011b38`, 334 B) → `SOC_Smoothing` (`0x08011878`) →
`Charge_SOC_Integration`/`Discharge_SOC_Integration` (`0x080119a8`/`0x08011df8`) je nach Modus.
Im Leerlaufmodus (3): SOH-/Zyklenzahl-Update aus dem Coulomb-Durchsatz (`SOH_From_Throughput`
`0x08012ab4`, `CycleCount_From_Throughput` `0x08012274`) plus ein **1-Stunden-Leerlauf-Timer**
(`g_socIdleTimeAccumMs`), der nach Ablauf — abhängig von Lade-/Entladerichtung vor dem Leerlauf — eine
OCV-Fenster-Prüfung auslöst und ggf. `NTC_ConvertAndUpdatePackTempExtremes` aufruft. **Das ist der
tatsächlich wirksame OCV-Rekalibrierungspfad** in 117.7 — die in 118 neu hinzukommende, separat
aufgerufene Funktion `SOC_OcvBoundsRecalibrateOnFlag` existiert in 117.7 nicht und ist laut
Naming-Evidenz für 118 ohnehin unerreichbarer Code (kein Schreiber für das benötigte Flag,
s. Diff-Dokument §2.3).

`SOC_CoulombCountTick` (`0x08016598`, 284 B) läuft im Leerlaufzweig mit einem 0,4-pro-Aufruf-
Downsampling-Muster. Kalibrierdaten werden über `SOC_SaveCalibrationToFlash` (`0x08015fdc`),
`SOC_SaveCalibrationIfChanged` (`0x08016490`) und `SOC_EepromLoadCalibration`/`SOC_
ReadAndValidateFlashCalibSlot` in einem 50-Slot-Wechsel-Log persistiert (§4.4); nur die ersten vier
32-Bit-Worte jedes Slots werden beim Laden gegen `0xf4241` (1 000 000, Fixpunkt-Obergrenze) geprüft —
das fünfte Wort wird ungeprüft übernommen (möglicher Validierungs-Gap, Offener Punkt §12).
`SOC_UpdateDisplayValueAndFullFlag` (`0x08016ac0`) erzwingt den angezeigten SOC auf 100,0 % (1000),
wenn das Vollladungs-Latch gesetzt ist und nicht entladen wird.

## 8. Lade-/Entladegrenzen

Deckt sich inhaltlich vollständig mit dem verifizierten Befund in `VNSD_Ladegrenzen_Ladeleistung_de.md` (dort mit v118-Adressen); hier die entsprechenden **117.7-Adressen**:

- `Max_Charge_Current_Calc` (`0x080125f0`, 84 Instr.) berechnet den erlaubten Ladestrom des aktiven
  Packs aus Temperatur (Achse `g_tempAxisTable` @ `0x0801B846`, 18 Stützstellen) und SOC (Achse @
  `0x0801B86A`) über eine Prozent-von-50-A-Matrix (`g_chargeCurrentDerateMatrix` @ `0x0801B884`,
  18×12×`uint16`); Ergebnis in 0,1 A (100 % = 500 = 50,0 A).
- `Max_Discharge_Current_Calc` (`0x080126e8`, 93 Instr.) nutzt dieselbe Temperaturachse
  (`0x0801B846`) sowie eine Entladestrom-Matrix @ `0x0801BA34` und zwei Spannungs-Bracket-Index-Tabellen
  @ `0x0801BA48`/`0x0801BA3E`.
- `Master_Build_ChargeCurrentLimit` (`0x0800bb90`, 174 B) begrenzt den Wert des aktiven Packs
  nochmals auf **500 = 50,0 A** und setzt die Ladeschlussspannung fest auf `g_sysChargeVoltLimit_x10`
  (`0x20004a16`) = `0x240` = 576 → **57,6 V**. `Master_Build_DischargeCurrentLimit` (`0x0800bc58`,
  64 B) setzt entsprechend `g_sysDischargeVoltLimit_x10` (`0x20004a1a`) = `0x1E0` = 480 → **48,0 V**.
  **Klarstellung** (per Brief, korrigiert gegenüber älteren Notizen): Beide Globals sind **Spannungen**
  (57,6 V/48,0 V), keine Ströme.
- `PackHandoff_StateMachine` (`0x080066dc`) erzwingt, dass die Lade-MOSFET-Maske nie mehr als ein Bit
  gesetzt hat → **physisch lädt immer nur ein Pack**, unabhängig von der Packanzahl.
- `CAN_TX_PF3_ChargeDischargeLimits` (`0x08007c74`, 318 B) sendet Strom-/Spannungslimit per CAN.

Ergebnis (unverändert zu 118): maximal 50,0 A / 57,6 V pro aktivem Pack, keine Summierung über Packs;
zusammen mit den Micro-/Control-Grenzen (2500 W) ergibt sich die in `VNSD_Ladegrenzen_Ladeleistung_de.md` beschriebene Gesamtgrenze. Die **einzige** 117.7-spezifische Abweichung betrifft nicht
diese Grenzwerte selbst, sondern die Rotationsschwellen-Formel in `PackSelect_
DischargeStepThreshold` (§6, Diff §2.1): In 117.7 hängt diese Formel **nicht** von
`g_dischargeLimitPct` ab, sondern nutzt eine feste 10-%-Stufentabelle mit Minimalwerten 9/10.

## 9. Schutz

`Cell_OVP_UVP_TempCompensated_Protection` (`0x0800cd18`, 441 Instr.) und `Voltage_Temp_
Protection_Checker` (`0x0800c0b8`, 705 Instr., trotz Namens nur Temperaturprotektion) bilden den
Kern der Spannungs-/Temperaturüberwachung. `Current_Protection_Checker` (`0x0800ac20`, 544 B)
implementiert zweistufigen OCP für Lade- und Entladerichtung plus einen fest verdrahteten
kurzschlussartigen Trip. `HW_OvercurrentAndSensorProtection` (`0x0800aff4`, 366 B) deckt zusätzlich
NTC-/MOS-Temperatur-Plausibilität ab. Bitmasken-Aufbau: `Protect1_Bitmask_RawBuilder` (`0x0800da80`)
baut die rohe Maske, `Protect1_Bitmask_BuilderWithDebounce` (`0x0800da30`) legt Entprellung darüber;
`Protect2_Bitmask_Builder` (`0x0800db30`) und `Warning_Bitmask_Builder` (`0x0800d980`) ergänzen
Protect2-/Warnwort. `BMS_ProtectionMasterTick` (`0x0800bac0`, 116 B) ist der periodische
Top-Level-Dispatcher (aus `RTOS_vTaskProtectCheck`). `AFE_CommLine_FaultChecker` (`0x0800bcbc`) und
`HW_RelayFaultChecker` (`0x0800bd48`) prüfen AFE-Kommunikationsleitung bzw. Relais-Hardware-Fehler.
`Cell_Balancing_PreconditionManager` (`0x0800f3e0`, 342 B) gated den Balancer anhand von
Temperatur-/Spannungsschwellen (kein OTA-Bezug trotz älterer Fehlbenennung). Alle genannten Funktionen
sind zwischen 117.7 und 118 **identisch** — vollständiger C-Vergleich (nicht nur Listing-Diff) zeigt
keine geänderten Schwellwerte (Diff-Dokument, §3).

## 10. OTA / Bootloader

Image-Basis `0x08002800`, Versions-Tag `'BMS1'` (`0x31534D42`) @ `0x08010000`, Staging-Kopie des neuen
Images @ `0x0801E000` (außerhalb des laufenden App-Bereichs `0x08002800`–`0x0801C7FF`), Tag-Prüfung der
Staging-Kopie @ `0x0802B800` (`g_otaStagingVersionTag`). `BMS_OTA_Flash_Writer` (`0x0800d1f8`, 584 B)
verarbeitet Sub-Opcodes (0=Start/1=Anhängen/2=Block-Commit) aus dem CAN-Transport
(`CAN_CMD_Pack_Activation` Opcode `0xCE`); `BMS_OTA_Flash_Writer_FinalizeTail` (`0x0800d440`, 142 B)
schließt den Schreibvorgang ab und ruft `OTA_Set_Reboot_Params` (`0x080115a8`) — den "an den
Bootloader übergeben"-Handoff — sowohl von hier als auch von `RS485_OTA_Flash_Handler` (`0x0800c9d8`,
728 B, RS485-Transportweg) aus auf. `LIB_CRC16_Modbus_Calc` prüft OTA-Flash-Blöcke,
`Flash_AddressToSectorIndex` bildet Adressen auf Sektorindizes ab. `HAL_Flash_Write_Config_Block`
(`0x0800ff98`, 74 B) schreibt in den gemeinsamen Boot-/App-Auswahlbereich `g_bootConfigPage`
(`0x08002000`, unterhalb des App-Images). Alle genannten OTA-Funktionen sind zwischen 117.7 und 118
identisch bis auf Adressverschiebung.

## 11. Sicherheit

- **Shell-Login:** Standardnutzer `BLUE`, Passwortprüfung über `CLI_ValidateLoginPassword`
  (`0x08013e48`); laut Diff-Artefakt im Literal-Pool ist das hinterlegte Passwort leer — das
  Passwort-Byte selbst wurde in dieser Sitzung nicht erneut aus dem Rohbinary dekodiert (Offener Punkt).
- **Config-Integrität:** `g_cfgVersionTag` (`'u'` = 117) wird beim Boot gegen den erwarteten Wert
  geprüft (`BMS_Config_Params_Load_Or_Default`, `0x08009c60`); bei Mismatch Factory-Reset des
  Parameterblocks. `LIB_Checksum_Ones_Complement` (`0x0800b18c`) sichert den Config-Block zusätzlich
  per Einerkomplement-Checksumme.
- **Fehlerverriegelung:** `APP_SetErrorLockFlag`/`APP_CheckErrorLockStatus` (`0x0800c028`/`0x0800c068`)
  implementieren ein persistentes Fehler-Lock (kein AFE-Registerzugriff trotz älterer
  Fehlzuordnung), geprüft beim Power-On (`APP_CheckErrorLockStatus`, einziger Aufrufer die
  Init-Routine). `APP_EnsureFactoryMarkerByte` (`0x0800af40`) verwaltet ein Werksmarker-Byte, dessen
  genauer Zweck nicht abschließend geklärt ist.
- **OTA-Authentifizierung:** Keine kryptographische Signaturprüfung im dekompilierten Code gefunden;
  die einzige Integritätsprüfung ist die CRC16-Blockprüfung (`LIB_CRC16_Modbus_Calc`) plus der
  Versions-Tag-Vergleich (`'BMS1'`). Kein Hinweis auf Verschlüsselung des OTA-Images.
- **I2C-EEPROM:** Bitgebangte Software-Implementierung ohne Hardware-I2C-Peripherie; kein
  Zugriffsschutz jenseits der Mutex-Serialisierung (`I2C_EEPROM_Write_WithMutex`).

## 12. Offene Punkte

- **KA495XX-Checksummenalgorithmus** (`KA495XX_ReadRegister` → `HAL_ShortBusyWaitLoop`): Die
  eigentliche Byte-Checksummenberechnung ist im Dekompilat nicht sichtbar (vermutlich
  Decompiler-Artefakt bei Registerverfolgung); Algorithmus (XOR/Summe) nicht rekonstruiert.
- **KA495XX-Registerschleifen** (`KA495XX_VerifyConfigRegisterTable`, `KA495XX_
  CheckFaultRegisters`): Schleifen-Dekompilation zeigt verlorene Registerverfolgung
  (`extraout_r2`/`r3`); Abbruchbedingung aus dem umgebenden Muster erschlossen, nicht direkt gelesen.
- **Shell-Passwort:** Nicht erneut aus dem Rohbinary dekodiert; frühere Analyse notierte ein leeres
  Passwort anhand eines Literal-Pool-Kommentars.
- **`PackSelect_RelayCommStateMachine`/`PackSelect_TriggerWakePulse`:** Exakte
  Handshake-Protokollsemantik der Relaisleitung zwischen Packs nicht vollständig rekonstruiert.
- **SOC-Kalibrier-Validierung:** `SOC_ReadAndValidateFlashCalibSlot` prüft nur die ersten vier von
  fünf 32-Bit-Worten jedes Slots gegen die Obergrenze `0xf4241`; das fünfte Wort (vermutlich
  `TarSoc`) wird ungeprüft übernommen — möglicher, aber nicht bestätigter Validierungs-Gap.
- **`g_socOcvRecalibrateRequest`:** In 117.7 vorhanden, aber ohne jeden Leser oder Schreiber im
  gesamten Image — Zweck und vorgesehener externer Trigger unklar (s. auch Diff-Dokument §4).
- **BMS_InitDefaultProtectionConfig:** Mehrere Schwellwerte (`0x20004104`–`0x20004112`) ließen sich
  aus dieser Sitzung heraus keiner konkreten physikalischen Bedeutung zuordnen (vermutlich weitere
  Temperaturschutzschwellen).

## 13. Korrekturen gegenüber der alten Analyse (`VNSD_BMS_FW_Analyse_v117.7_de.md`)

Die alte Analyse ging von einer **falschen Basisadresse `0x08000000`** aus (statt korrekt
`0x08002800`) — jede dort genannte Flash-Adresse ist um `0x2800` zu niedrig. Konkrete Korrekturen:

| Punkt | Alte Analyse | Diese Analyse |
|---|---|---|
| Basisadresse | `0x08000000` | `0x08002800` (Bootloader-Offset) |
| Flash-Bereich | `0x08000000–0x08019FFF` | `0x08002800–0x0801C7FF` |
| Initial-SP | `0x2000CB90` (zufällig korrekt, da RAM-Adressen von der Flash-Basis-Verschiebung nicht betroffen sind) | `0x2000CB90` (bestätigt) |
| Reset-Handler | `0x08002A6D` | `0x08002A6D` (Vektor-Eintrag, Thumb-Bit gesetzt) → `Reset_Handler` @ `0x08002a6c` |
| Funktionen | 552 / 552 (100 %) | **710** / 710 (100 %) — deutlich mehr Funktionen durch bessere Auto-Analyse/Namensgebung, nicht durch andere Firmware |
| MCU | Nicht explizit als N32G45x identifiziert; Peripherie-Adressen wurden ohne offizielles SVD interpretiert | **Nations N32G45x**, bestätigt über ADC1/ADC2-Basisadressen, UID-Adresse und Vektortabellengröße (s. §1) |
| Versions-Tippfehler | Datei/Titel nannte ursprünglich fälschlich "v177.7" (im Dokument selbst bereits am 2026-07-09 korrigiert) | Version **117.7** bestätigt über `g_pack0FwVersionTag = 0x499` (1177 dezimal) |
| Ladegrenzen | In der alten Analyse nicht mit Ghidra-verifizierten Adressen belegt | Vollständig verifiziert und mit `VNSD_Ladegrenzen_Ladeleistung_de.md` (118-Adressen) sowie den 117.7-Adressen in §8 dieses Dokuments abgeglichen |
| "148-W-Plateau" | In verwandten Pack-Rotations-/SOH-Dokumenten früher als BMS-Verhalten beschrieben | **Zurückgezogen** — der Effekt stammte von der Omnibattery-Integration, nicht vom BMS (s. Projekt-Memo); wird in diesem Dokument nicht wiederholt |

Alle sonstigen inhaltlichen Befunde der alten Analyse (Task-Liste, KA495xx als Zellmonitor-IC,
CAN+RS485-Kommunikationswege) wurden gegen den frischen 117.7-Export erneut verifiziert und in den
§§3–11 mit korrekten Adressen dargestellt.
