# BMS-FW V110.5 (Venus A) — Vollanalyse

> 🇬🇧 English version: [VNSA_BMS_FW_v1105_Full_Analysis.md](VNSA_BMS_FW_v1105_Full_Analysis.md)

**Stand:** 2026-10-02.

**Referenz-Image:** Venus D BMS 118 (`VNSD_BMS_FW_v118_Vollanalyse_de.md`) ·
**Dekompilat/Listing:** `Dekompilate/VNSA_BMS_1105/` (nur lokal, nicht im Repo) (698 Funktionen, `listing/` mit
Flash-Disassembly + `globals_and_registers.txt`) · **Gegenstück:** `Control_FW/VNSA_Control_FW_v1509_Vollanalyse_de.md`.

## 0. Zusammenfassung

- Image-Basis **0x08002800**, Image-Größe 0x19800 Byte (104.448 B), MCU **Nations N32G45x** (Cortex-M4F) — gleiche MCU,
  Toolchain (Keil ARMCC), RTOS (FreeRTOS, `heap_4`, Heap `0x4FF0` B) und Shell (letter-shell, Default-User `BLUE`) wie
  beim Venus-D-BMS. Versions-String `Version: 110.5`, Build-Datum `May 11 2026 09:47:24`.
- **Codebasis:** Das A-BMS ist ein Geschwister-Build der D-BMS-Codebasis. 77 % der Image-Bytes liegen in Segmenten, die
  mit D 118 identisch sind (Pointer/BL maskiert); 506 der 711 D-Funktionen kommen byte-identisch vor, 57 mit Änderungen,
  148 D-Funktionen fehlen (vor allem der KA495XX-AFE-Treiber und der zweite RS485-Bus). Das A-Image bringt ~130 eigene
  Funktionen mit (SH36735-AFE-Treiber, Event-Log, RTC-/Zähler-/Balancing-Shell-Befehle, Master-Marker, LED-Muster).
- **Batterie-Frontend:** Der A nutzt einen **SH36735**-AFE (Teilename aus den Strings `SH36735`, `SH36735_PRT`; Hersteller
  nicht verifiziert) statt des KA495XX des D, mit **13 Zellen pro Pack** (Default; 4..12 über AFE-Register 0x43
  konfigurierbar, alles andere fällt auf 13 zurück). Der D hat 16 Zellen. Der FreeRTOS-Task behält den D-Namen
  `RTOS_vTaskKA495XX`.
- **Spannungsbezogene Grenzen auf 13S LFP skaliert:** Entladespannungsgrenze 48,0 V → **39,0 V**, CAN-Ladespannungs-Clamp
  60,9 V → **46,8 V**, Ladespannungsgrenze fest 57,6 V → **Packspannung + 2,0 V, max. 48,0 V**, Kapazitätsfaktor im
  CAN-Kapazitäts-Frame 512 → **416** (13 × 32 statt 16 × 32). Tabellen auf Zellebene (OCV/SOC-Kurve, NTC-Kurven, Lade-
  und Entlade-Derating-Matrizen, Achsen, SOH-Kurve) sind **byte-identisch** zu D 118.
- **Changelog-Punkte im Code gefunden:**
  1. „Tiefentladungs-Rettung per PV-Start zusammen mit dem Wechselrichter“: Solange Zellunterspannung Stufe 2 aktiv ist,
     wartet das BMS jetzt bis zu **120 s** auf einen CAN-Frame **`0x03410110`** vom Wechselrichter (Nutzdaten-Byte 1 > 8 =
     `g_pvRescueRequestFlag`); ohne ihn geht das BMS in den STOP-Modus (`System_Reset_And_Halt_WithCurrentCheck`).
     Zusätzlich wird die CAN-Ladestromgrenze auf **10,0 A gezwungen, solange die Packspannung unter 35,00 V liegt**, die
     UVP-Stufe-2-Schwelle Stufe A beträgt 2800 mV (D: 3000 mV) und die Min-Zellen-Sensorfehlerauslösung ist nur oberhalb
     von 39,00 V aktiv.
  2. „Optimierung der Zwangsladelogik“: nicht sicher zugeordnet; Kandidaten sind die 10-A-Ladebegrenzung bei Unterspannung
     und das neue „noch nie voll geladen“-Derating in `Max_Charge_Current_Calc` (siehe §8).
- **Alle 698 Funktionen benannt und typisiert**, 0 `FUN_*`, alle Prüf-Gates bestanden (§2.3). Tags: `NAME_VERIFIED` 698,
  `NAME_LOW_CONFIDENCE` 49.

## 1. Binär-Fingerabdruck

| Datei | Version | Größe | SHA-256 |
|---|---|---|---|
| `1105_bms_VNSA-0_VA50A_APP_V1105_ota_144658.bin` | 110.5 (Beta) | 104.448 B (0x19800) | `c642c7ea1e84dc8d5848f86fb97f69baaa7e9c4f4ee41473bf972a836a0dfc3d` |

Hersteller-Changelog (übersetzt): „Nur zusammen mit Wechselrichter-Firmware 119.3 oder neuer verwenden. 1. Rettung der
Batterie bei extremer Unterspannung per PV-Start zusammen mit dem Wechselrichter. 2. Optimierung der Zwangsladelogik.“
Image-Versions-Tag bei `0x08010000`: **`BMS2`** (D: `BMS1`).

## 2. MCU, Speicherkarte, Bearbeitungsstand

### 2.1 MCU: Nations N32G45x (Cortex-M4F)

Gleiche Belege wie beim D-BMS: Vektortabelle mit 102 Einträgen (16 + 86 IRQs), UID-Lesezugriff bei `0x1FFFF7F0`
(`HAL_ReadUniqueDeviceID`), N32-RCC/PWR-Layout in `SystemInit` (setzt `SCB_VTOR = 0x08002800`). Register- und IRQ-Namen
aus der N32G457-SVD. Genutzte Peripherie: SPI1 (AFE, Chip-Select PA4, Polling), CAN1, USART1 (RS485, 9600 Baud),
ADC, TIM2/TIM3, ein per GPIOD gebitbangter serieller Bus zur RTC (Code identisch mit D, dort `KA495XX_SPI_*` genannt; da der
A keinen KA495XX hat, ist das sehr wahrscheinlich ein separater RTC-Chip), I2C-EEPROM.

### 2.2 Speicherkarte

| Bereich | Inhalt |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (nicht Teil des Images) |
| `0x08002000` | Boot/OTA-Handshake-Seite `g_bootConfigPage` (im Bootloader-Bereich) |
| `0x08002800`–`0x0801BFFF` | Applikations-Image; Vektortabelle bei `0x08002800` |
| `0x08010000` | Image-Versions-Tag `g_imageVersionTag` = `'BMS2'` |
| `0x0801AB10` | `g_crc8Poly07Table` (CRC-8, Polynom 0x07, SH36735-Frames) |
| `0x0801B9D4` | Keil-Regionstabelle (2 Regionen), `0x0801B9F4` `shellCommandTable` (52 × 16 B), `0x0801BD34` `.data`-Ladeabbild |
| `0x0801E000` | OTA-Staging-Kopie `g_otaStagingImage` (Tag-Wort bei `0x0802B800` muss `'BMS2'` sein) |
| `0x20002800`–`0x20002B9F` | `.data` (0x3A0 B, im Image LZ-komprimiert; D: 0x3C4 B) |
| `0x20002BA0`–`0x2000C95F` | `.bss` (0x9DC0 B; D: 0x9FDC B); FreeRTOS-`ucHeap` bei `0x200046D8`, Keil-`malloc`-Heap `0x20009964`–`0x2000A95F` |
| `0x2000C960` | `__initial_sp` (Stack-Obergrenze) |
| I2C-EEPROM | Konfigurationsblock bei 600, Master-Marker `0x01F6`, Zähler `0x0254`, Event-Log `0x2000`/`0x3000`, Vollladungs-Marker `0xFC02` (§4.4) |

Das RAM-Layout weicht von D 118 ab (abschnittsweise Verschiebungen um 4–2.584 Byte); alle RAM-Namen wurden deshalb über
die Literal-Pool-Referenzen gepaarter Funktionen übertragen, nicht über die Adresse.

### 2.3 Bearbeitungsstand in Ghidra

- Programm `/VNSA/BMS/BMS_1105_VNSA-0_VA50A_APP_V1105_ota_144658.bin`: **698 Funktionen, alle benannt**, jede Funktion hat
  einen benutzerdefinierten Prototyp (Task-Einstiege `void f(void *pvParameters)`), Plate-Kommentare, 0 `FUN_*`,
  0 Duplikate.
- Gates (`vna_bms_gates.py`, eine aufgeteilte Fassung von `marstek_gates_micro.py` plus zwei Zusatzprüfungen):
  0 undefinierte Bytes, 0 verwaiste Instruktionen, 0 Flüsse in Daten, 0 Funktionszeiger auf Nicht-Code, 0 reine
  Literal-Funktionen, **0 als Code dekodierte Literal-Pools**, **0 Aufrufe mitten in Funktionsrümpfe**, 0 unbenannte
  referenzierte RAM-Globals, 0 Default-Prototypen, Task-Namen = `xTaskCreate`-Namensstrings, No-Return-Flags plausibel.
  Dekompilat-Export: 698 Funktionen, **0 Fehler**.
- Portierung: 563 D↔A-Funktionspaare (506 identisch, 57 geändert) haben Namen, Prototypen, Plates und lokale Variablen
  übernommen; jede der 57 geänderten Funktionen wurde gegen den D-Code geprüft (52 behalten den D-Namen mit A-spezifischem
  Plate, 5 umbenannt). Die ~130 A-eigenen Funktionen und 145 A-eigenen RAM-Globals wurden aus dem Code benannt.
- Korrekturen gegenüber einer reinen Portierung: Literal-Pools hinter nicht zurückkehrenden Aufrufen, die auch das
  Referenzprogramm als Code dekodiert (6 Stellen, D 118 hat denselben Fehler), eine 2-Byte-Verschiebung des System-SOC-
  Blocks (`g_sysMaxCellVoltage`… bei `0x20004068`), D-Namen im AFE-RAM-Bereich, die zum alten KA495XX-Treiber gehören.

## 3. FreeRTOS-Tasks

`RTOS_CreateAllTasks` (`0x0800EA34`) legt dieselben 10 Tasks an wie beim D (Namen, Stacks und Prioritäten identisch):

| Task | Einstieg | Stack (Worte) | Prio | Rumpf beim A |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800ECC0` | 0x80 | 5 | LED-Muster (`LED_StatusPatternUpdate`, PB5) + Leerlauf-Relaistakt |
| `RTOS_vTaskKeyScan` | `0x0800ECA0` | 0x80 | 9 | Pack-Auswahl-Relais-Zustandsautomaten (kein Tastenscan) |
| `RTOS_vTaskKA495XX` | `0x0800EC88` | 0x80 | 8 | **SH36735**-Worker alle 5 ms + `PerPack_Struct_Builder` |
| `RTOS_vTaskCanRec` | `0x0800EC28` | 0x180 | 4 | CAN-Empfangsverarbeitung |
| `RTOS_vTaskCanSend` | `0x0800EC38` | 0x80 | 3 | CAN-Sendequeue |
| `RTOS_vTaskRS485Comm` | `0x0800ECFA` | 0x180 | 1 | RS485 (ein Bus) |
| `RTOS_vTaskSocAlgorithm` | `0x0800ED08` | 0x100 | 7 | SOC/SOH |
| `RTOS_vTaskProtectCheck` | `0x0800ECE4` | 0x100 | 6 | Schutzfunktionen |
| `RTOS_vTaskControlLogic` | `0x0800EC42` | 0x80 | 2 | Aggregation, Schutz-/Kommunikations-Tick |
| `RTOS_vTaskDataSave` | `0x0800EC5C` | 0x80 | 0 | IWDG füttern, RTC lesen, Bildschirm-Refresh |

Der unabhängige Watchdog wird nur gefüttert (`HAL_IWDG_Refresh`); anders als beim D (`System_Init_Main`: Prescaler 6,
Reload 0x30D, Start) konfiguriert oder startet die A-Applikation ihn nie.

## 4. Schnittstellen

### 4.1 CAN (interner Bus zu Control, Wechselrichter und anderen Packs)

29-Bit-Identifier, gleiches internes Layout wie beim D. Empfangene Frames sind N32-`CanRxMessage`-Strukturen; „Byte n“
bedeutet unten Nutzdaten-Byte n (Strukturoffset 0x0B + n).

**Ausgehend** (`CAN_TX_Orchestrator` `0x08007ACC`, Antwort auf `0x1801nnAA`):

| ID | Funktion | Inhalt (A) | Unterschied zum D |
|---|---|---|---|
| `0x1801AA01` | `CAN_TX_PF1_PackMeasurements` | Packspannung (10 mV), Gesamtstrom (0,1 A), max. NTC, mittlerer SOC | — |
| `0x1802AA01` | `CAN_TX_PF2_Capacity` | Gesamtkapazität = Packs × Kapazität × **416** / 10, Packanzahl, Online-Maske, aktives Pack, Entlade-%, Relaiszustand | Faktor 512 beim D |
| `0x1803AA01` | `CAN_TX_PF3_ChargeDischargeLimits` | Ladespannungsgrenze (Clamp **46,8 V**), Ladestrom (**10,0 A bei Packspannung < 35,00 V**, 25,0 A bei Pack-Fehler), Entladestrom, Statusbits | Clamp 60,9 V, keine Unterspannungs-Begrenzung beim D |
| `0x1804AA01` | `CAN_TX_PF4_ProtectWarnings` | Protect-Wort 1/2, Statuscode `g_chargeStatusCode`, online, Error-Lock | D sendet Statuscode 0 |
| `0x1820..0x1823AA00+n`, `0x1830..0x1833AA00+n`, `0x1840/0x1841AA00+n` | `CAN_TX_PerPack_10Msgs` | pro Pack n = 1..6 (ab 1 gezählt: die Grenze lautet `pack_id != 0 && pack_id < 7`, die Datensätze werden mit `pack_id - 1` indiziert, also sechs Packs): Übersicht, Grenzen, Min/Max, Protect + FW-Tag; Zellspannungen (16 Plätze, 13 belegt); NTC-/MOS-Temperaturen | gleiche Frame-Menge |
| `0x1851AA00+n`, `0x1852AA00+n` | `CAN_TX_PerPack_SocDiag` | **nur A**: 7 Worte aus `0x20004020`–`0x2000402E` (Schreiber nicht gefunden) | — |
| `0xA5A5` | `CAN_TX_KeepaliveA5A5` | **nur A**: 1-Byte-Frame (Daten 1) aus dem 500-ms-Leerlauf-Tick, wenn freigegeben; auch für den RS485-Echotest | — |

Auf `0x100501AA` antwortet ein Pack mit `CAN_TX_OwnPackReport` (PF1–PF4, wenn online/Master, + eigene Per-Pack-Frames +
SOC-Diag-Frames); beim D setzte diese ID nur ein CAN-Neustart-Flag. Der Slave-Statusbericht aus 11 Frames
(`CAN_TX_BMS_Status_Reporter`) wird jetzt nach einer slot-abhängigen Verzögerung ((Slot − 1) × 15 ms,
`CAN_StatusReply_StaggerTick`) statt sofort gesendet.

**Eingehend** (`CAN_RX_Handler` `0x08008264`, `APP_CAN_RX_Task_Process` → `CAN_RX_System_Command_Handler`):

| ID | Bedeutung (A) | D |
|---|---|---|
| `0x1801nnAA` (nn = 1..6) | Statusanfrage für Pack nn → `CAN_TX_Orchestrator(nn)` | gleich |
| `0x1801FFAA` | Master-Heartbeat an alle Packs: Master-Enable (Byte 0 == 1), EMS-Flags (Byte 1), Entlade-% (Byte 2), Watchdog-Reset → `CAN_TX_Orchestrator(0xFF)` | gleich |
| `0x100101AA` | Systembefehl, Opcode = Byte 0 (Tabelle unten) | Teilmenge |
| `0x100201AA` | SOC-Reset auf Byte 0 % (`CAN_CMD_02_Handler`, Event 3) | gleiche ID |
| `0x100301AA` | Entladegrenze % (`g_dischargeLimitPct`) + EEPROM-Schreiben | gleich |
| `0x100401AA` | `CAN_CMD_04_Handler` | gleich |
| `0x100501AA` | zweiter Heartbeat: gleiche Felder wie `0x1801FFAA`, Antwort ist aber `CAN_TX_OwnPackReport` (nur eigenes Pack) | nur ein CAN-Neustart-Flag (Byte 0 == 1) |
| **`0x03410110`** | **nur A:** `g_pvRescueRequestFlag = (Byte 1 > 8)` — Anforderung des Wechselrichters, die das BMS bei Tiefentladung wach hält | — |
| `0xA5A5` | **nur A:** Echotest-Antwort | — |
| Präfix 3 (`CAN_RX_Command_Dispatcher`) | Slot-Befehle: 1 Status, 2 Pack-Modus, 3 Handshake, 4 Status/Alarm, 5 Werksreset, 0x0A Notify, 0xC4 Aktivierung, 0xCE OTA-Schreiben | gleiche Menge |

**Systembefehl-Opcodes `0x100101AA`** (`CAN_CMD_01_Handler` / `CAN_RX_System_Command_Handler`):

| Opcode | Aktion | D 118 |
|---|---|---|
| 1 | Adressanfrage (anstehender Systembefehl 1) | ja |
| 2 | Werksreset + Neustart (`APP_Factory_Reset_And_Reboot`) | ja |
| 3 | Relaisbefehl | ja |
| 4 | Fehlermodus 4 | ja |
| 7 | Pack-Übergabe-Anforderung | ja |
| 8 | **Entsperren**: Error-Lock löschen (EEPROM-Marker `0xFC00`) | ja |
| 9 | **Sperren**: Error-Lock setzen | ja |
| 0x0A | **CAN neu starten** (`CAN1_Reinit_OrSystemReset`) | **neu** |
| 0x0B | **Master werden**: Master-Marker → EEPROM, Event 2, Werksreset/Neustart, Slot 1 | **neu** |
| 0x0C | **Slave werden**: Master-Marker löschen, Slot 0xFF, Systemreset | **neu** |

Einen „Encode“-Opcode gibt es im BMS nicht. `CAN_CMD_Pack_Activation` kennt zusätzlich Opcode 5 (Werksreset) und
Opcode 8 (**RTC stellen** aus einem gepackten 32-Bit-Datum: Jahr 6 Bit, Monat 4, Tag 5, Stunde 5, Minute 6, Sekunde 6).

### 4.2 RS485

Nur **ein** RS485-Bus auf USART1 (9600 Baud, PA9/PA10, kein Driver-Enable-Pin); der D betreibt zwei (USART2 + UART4,
DMA). Der IRQ-Handler ist `USART1_IRQHandler` (Vektor IRQ37). Frames werden über einen 3.645-µs-Pausentimer abgegrenzt
und beginnen mit `'s'`; der Dispatcher (`RS485_Dispatcher` `0x08011004`) behält die D-Opcodes und ergänzt:

| Opcode | Verhalten beim A |
|---|---|
| `0x03` | **neu:** Balancing ein/aus (manueller Modus, alle Zellen) |
| `0x05` | antwortet immer mit Versionsbyte `0x6E`; der D-Handshake (`A1 B1` → Pending-Flag) entfällt |
| `0x10` | **neu:** Zellspannungs-Stream ein/aus (alle 100 Abfragen ein `0x19`-Frame) |
| `0x19` | Zellspannungen: 13 Zellen (0x1A Byte) statt 16 |
| `0x2F` | **neu:** CAN-Echotest (Frame `0xA5A5`) |
| `0x00` | Statusantwort ohne das rohe KA495XX-Fehlerwort |

`RS485_Pack_Telemetry_Response` meldet das Konfig-Versionsbyte `'n'` (0x6E; D 118: `'v'`). Der Dispatcher enthält noch
einen vom D übernommenen Test auf Kanal 3 (UART4), für den es im A-Image keinen Kontext gibt (toter Pfad).

### 4.3 Debug-Shell (letter-shell)

User `BLUE` („default user“), 52 Tabelleneinträge: 10 Tastenbelegungen, `help`, `clear` und **39 Befehle** (D: 25). Alle
23 übrigen D-Befehle existieren mit gleichem Namen (`bat_data`, `set_soc`, `set_dod`, `checkout_rs485`, `force`,
`set_work_pack`, `state_printf`, `sox_data`, `wake`, `inq_uid`, `uid_printf`, `parall`, `self_flag`, `reset`, `can_send`,
`update_printf`, `event`, `prt`, `task_size`, `factory`, `lock_set`, `volt_diff`, `set_flg`). Neu beim A:

| Befehl | Handler | Funktion |
|---|---|---|
| `TOL_V` | `CLI_Cmd_PrintTolVoltage` | ADC-Kanal 0x0B → V (/4095 × 3,3 / 0,0208) |
| `BAL_ON` / `BAL_OFF` | `CLI_Cmd_BalanceOn` / `CLI_Cmd_BalanceOff` | manuelles Balancing aller Zellen / aus |
| `EVEN_BALL` / `ODD_BALL` | `CLI_Cmd_BalanceEvenCellsOn` / `CLI_Cmd_BalanceOddCellsOn` | gerade / ungerade Zellen balancieren |
| `SH36735` | `CLI_Cmd_PrintSH36735Raw` | AFE-Rohwerte (VADC, CADC, CCUR, VBAT1/2, Balance-Index, SELF-Flag) |
| `ctl_prt` | `CLI_Cmd_PrintLccAndLoadVoltage` | LCC-Pin (PC2) und Lastspannung |
| `cnt_read` / `cnt_write` | `CLI_Cmd_CounterRead` / `CLI_Cmd_CounterWrite` | EEPROM-Zähler bei `0x0254` |
| `addr_save` / `addr_reset` | `CLI_Cmd_AddrSave` / `CLI_Cmd_AddrReset` | Master-Marker speichern / löschen |
| `log_prt`, `log_read`, `log_write` | `CLI_Cmd_LogPrintIndex`, `CLI_Cmd_LogRead`, `CLI_Cmd_LogWrite` | Event-Log |
| `rtc_read` / `rtc_write` | `CLI_Cmd_RtcRead` / `CLI_Cmd_RtcWrite` | externe RTC |

Nach dem Login gibt die Shell ein ASCII-Banner mit Build-Datum und Version aus (`CLI_LoginUser`).

### 4.4 EEPROM-Belegung (I2C)

| Adresse | Inhalt |
|---|---|
| 600 (0x258), 0xD4 B | Konfigurationsblock (`BMS_Config_Params_Load_Or_Default`), Versions-Tag **`'n'`** (D: `'v'`), Einerkomplement-Prüfsumme |
| `0x01F6` | **nur A:** Master-Marker `{00 AA FF 55}` (Komplementprüfung); beim Boot gelesen → `g_pack0SlotId = 1`, sonst `0xFF` |
| `0x0254` | **nur A:** Zähler `{v, ~v}` (Shell `cnt_read`/`cnt_write`) |
| `0x2000`… | **nur A:** Event-Log-Index, 50 × 8 B `{gepackte Zeit u32, Event u16, Datensatzadresse u16}` |
| `0x3000`–`0x3CE3` | **nur A:** Event-Datensätze, je 0x42 B: Zeit, SOC, Strom, max./min. Zelle, max./min. NTC, MOS-Temperaturen, 16 Zellplätze, 4 NTC, Protect-Worte 1/2 |
| `0xFC00` | Error-Lock-Marker (`0x86` gesperrt / `0xFF` frei), wie beim D |
| `0xFC02` | **A:** Vollladungs-Marker `0xA5` (bei der ersten Voll-Erkennung geschrieben). Der D schreibt hier einen 10-Byte-RTC-Stempel |

Event-IDs: 0 SOC-Kalibrierpunkt (voll/leer), 1 Schutzzustand geändert, 2 Master geworden, 3 SOC-Reset per CAN,
4 Sprung der Anzeige-SOC > 5 %.

## 5. Zellüberwachung — SH36735

- SPI1, Polling, Chip-Select PA4 (`SH36735_SpiTransfer` → `HAL_SPI_TransmitReceive_Polling`). Lese-Frame
  `{02, reg, len}` → Antwort `{FF, 02, reg, len, Daten…, crc}`; Schreib-Frame `{01, reg, wert, crc}` → Quittung `… A5`.
  CRC-8 Polynom **0x07** (`SH36735_Crc8`, `g_crc8Poly07Table`; D-KA495XX: Polynom 0xD5). 5 Wiederholungen pro Transfer;
  dauerhafter Fehler setzt `g_protState4` Bit 4 (neues Protect2-Bit `0x400`) und, entprellt, `g_sh36735StatusFlags` Bit 7.
- Genutzte Register: Konfiguration 0x40–0x54 (jede Sekunde geschrieben und geprüft, 0x51–0x54 fest `96 76 76 9E`),
  Balance-Maske 0x55–0x57, Flags 0x58/0x59/0x5B/0x5C, NTC 0x5D (4 Kanäle), interne Temperatur 0x65, „VADC“ 0x67,
  Zellspannungen 0x69 (13 × 16 Bit), Strom „CADC“ 0x91, Packspannung 0x93, zweite Spannung 0x95, Zähler 0x97.
- Zellenzahl: Konfigurationsabbild Register 0x43, untere 5 Bit; Default `0x6D` → **13 Zellen** (`INIT_SH36735_Subsystem`).
- Skalierung: Zelle mV = roh × 5000 / 32000; Pack mV = roh × 50000 / 12800; Strom = Mittel über 4 Messungen mit 100 mA
  Totband; NTC über den Widerstand und die D-NTC-Tabellen. Automatische Nullpunktkalibrierung der Stromkanäle
  (`SH36735_ZeroOffsetCalibration`, „E2ssCurrOffset1/2“).
- Takt: Worker alle 5 ms, Messzyklus ~70 ms, langsamer Pfad 1 s (`SH36735_TaskStateMachine`).
- **Balancing** (`SH36735_UpdateBalancingMask`): automatisch nur bei Zellspreizung ≥ 20 mV und Packstrom ≥ 1000 mA; eine
  Zelle wird balanciert, wenn sie ≥ 3000 mV und ≥ 50 mV (unter 3400 mV) bzw. ≥ 20 mV über der kleinsten Zelle liegt, nach
  29 aufeinanderfolgenden Durchläufen. D (KA495XX): Spreizung ≥ 30 mV, Strom ≥ 1,001 A, Einschaltschwelle 3501 mV.
  Manuelle Modi über Shell (`BAL_ON`, `EVEN_BALL`, `ODD_BALL`) und RS485-Opcode 3.

## 6. Pack-Verwaltung

Gleiches Master/Slave-Modell und gleiche Pack-Auswahl-Familie wie beim D. Unterschiede: alle Auswahlen
(`PackSelect_Mode3_Empty`, `_Discharge_StepRotate`, `_Charge_StepRotate`, `_HighestSoc_Online`) überspringen Packs in
`g_packExcludeMask`; die Master-Rolle lässt sich per CAN-Opcode 0x0B/0x0C dauerhaft festlegen (EEPROM-Master-Marker);
`Pack_LowVoltage_Balancer` holt ein Pack zusätzlich in die Rotation, wenn seine kleinste Zelle unter 2800 mV liegt.
MOSFET-Gates (`OwnPack_MOSFET_Driver`, `_MasterVariant`) nutzen das SH36735-Kommunikationsfehler-Bit statt des
KA495XX-Watchdogs und sind gesperrt, bis die Nullpunktkalibrierung fertig ist („SELF“-Flag, Master-Variante nur über
39,00 V).

## 7. SOC/SOH

Identischer Algorithmus und identische Tabellen (OCV/SOC-Kurve, Coulomb-Zählung, SOH-Kurve `SOH_LookupByCycleCount` über
`g_ocpDelayVsCurrentTable` — eine vom D geerbte Fehlbenennung). Unterschiede: Leer-Schwelle 2800 mV bei ≥ 0 °C (D: 2750 mV;
2420 mV unter 0 °C unverändert), Voll-Schwelle 3700/3600 mV unverändert; Voll-/Leer-Erkennungen schreiben Event 0; die erste
Voll-Erkennung schreibt den EEPROM-Marker `0xFC02 = 0xA5`; `SOC_OcvBoundsRecalibrateOnFlag` und die RTC-Stempel-Schreibvorgänge
von D 118 fehlen; die Aktualisierung der Anzeige-SOC protokolliert Event 4 bei Sprüngen > 5 %.

## 8. Lade-/Entladegrenzen

| Punkt | Venus A 110.5 | Venus D 118 |
|---|---|---|
| Max. Ladestrom | 50,0 A (Tabelle, identische Matrix) | 50,0 A |
| Ladestrom, solange noch nie voll geladen (`g_neverFullChargedFlag`) | 50,0 A unter 45 °C, darüber 25,0 A − 1,67 A/°C | — |
| System-Ladestrom außerhalb des Master-Run-Zustands | 50,0 A, 0 solange das Nullpunktkalibrier-Flag gesetzt ist | 0 |
| Ladespannungsgrenze | Packspannung + 2,0 V, max. 48,0 V; CAN-Clamp 46,8 V | 57,6 V fest; CAN-Clamp 60,9 V |
| CAN-Ladestrom bei Packspannung < 35,00 V | 10,0 A | keine Begrenzung |
| Entladespannungsgrenze | 39,0 V | 48,0 V |
| Entladestrom | 50,0 A Deckel, identische Spannungs-/Temperatur-Matrix | gleich |
| Konfig: Kurzschluss-Stufe-1-Timeout | 5000 ms (beide Reset-Pfade) | 20000 ms |

## 9. Schutzfunktionen

Konfigurations-Defaults identisch zum D (Zell-OVP 3600/3650/3700 mV, UVP 2800/2700/2600 mV, Freigabe 3350/3150 mV,
OC 55 A/SC 60/100 A, Temperaturschwellen), außer dem Kurzschluss-Stufe-1-Timeout (5 s). Code-Unterschiede:

- `Cell_OVP_UVP_TempCompensated_Protection`: UVP Stufe 2, Stufe A (≥ 0 °C) 3000 → **2800 mV**; eine Auslösung setzt
  `g_protState1` Bit 5, löscht die Auslösezähler und setzt den SOC-Schätzer auf 0 %. Min-Zellen-Sensorfehlerauslösung
  (< 1500 mV) nur solange die Packspannung über 39,00 V liegt.
- **Tiefentladungsbehandlung** (`BMS_ProtectionAndCommTick`, 1-s-Tick): D — bei aktiver UVP Stufe 2 werden die eigenen
  MOSFETs nur unter 2,0 A freigegeben, und ein alleinstehendes Pack geht nach 200 s in Fehlermodus 3. A — zählt
  `g_uvpLvl2ShutdownTimerS`; nach **120 s ohne `g_pvRescueRequestFlag`** (CAN `0x03410110`, Byte 1 > 8) ruft das BMS
  `System_Reset_And_Halt_WithCurrentCheck` auf (STOP-Modus, danach Systemreset); die D-Strom-/Alleinstehend-Logik entfällt.
- `HW_OvercurrentAndSensorProtection`: Kreuzprüffenster des Komparators ±3,0 A → ±70,0 A.
- Protect2-Wort: neue Bits `0x400` (SH36735-Kommunikationsfehler) und `0x2000` (durch einen Adress-Reset-Befehl
  verriegelt); Masken für Fehlerscan / Alles-Aus auf `0x2E00` erweitert.
- `System_Reset_And_Halt_WithCurrentCheck`: kein EEPROM-Fehlerstempel vor dem STOP-Modus (der D schreibt einen RTC-Stempel
  nach `0xFC02`, wenn |P| > 5 kW).

## 10. OTA/Update

Gleiche Pfade über CAN (`BMS_OTA_Flash_Writer`, Opcode 0xCE) und RS485 (`RS485_OTA_Flash_Handler`), Staging-Kopie bei
`0x0801E000`, CRC16-Prüfung, Bootloader-Handshake-Magics `0x11223344`/`0x55667788`. **Das Staging-Image muss `'BMS2'`
tragen** (D: `'BMS1'`), D-BMS-Images werden vom A-BMS also abgelehnt und umgekehrt. Neu: `OTA_EnsureBootConfigPage`
repariert die Bootloader-Seite `0x08002000` (Zustandswort 3) beim Start, wenn sie inkonsistent ist.

## 11. Sicherheitshinweise

- Shell-Login-User `BLUE` und der volle Befehlssatz (inkl. `factory`, `lock_set`, `addr_save`/`addr_reset`, `rtc_write`,
  `log_write`) sind wie beim D über den RS485-Debugpfad erreichbar.
- CAN-Befehle ohne Authentifizierung können die Master-Rolle dauerhaft umschalten (0x0B/0x0C), den Error-Lock setzen/
  löschen (8/9), einen Werksreset auslösen (2, Slot-Befehl 5) und die RTC stellen.
- Der IWDG wird von der Applikation nie gestartet (nur gefüttert); ein Hänger wird nicht abgefangen, sofern ihn nicht
  der Bootloader startet.

## 12. Zusammenspiel mit dem Venus-A-Wechselrichter (Micro 1211) und Control 1509

- Der Wechselrichter muss innerhalb von 120 s nach einem Zell-UVP-Stufe-2-Ereignis CAN **`0x03410110`** mit einem
  Nutzdaten-Byte 1 größer als 8 senden, sonst schläft das BMS ein; das ist die BMS-Seite des „PV-Rettungsstarts“ (Changelog: Wechselrichter ≥ 119.3).
- Während der Rettung meldet das BMS nur 10,0 A Ladestrom (`0x1803AA01`), solange die Packspannung unter 35,00 V liegt,
  und eine Ladespannungsobergrenze von 46,8 V.
- Control/EMS: gleiches Anfrage-/Heartbeat-Protokoll; der Heartbeat `0x100501AA` löst jetzt einen vollständigen Bericht des
  eigenen Packs aus; die Systembefehle 0x0A–0x0C sind neu.

## 13. Offene Punkte

1. Genaue Bedeutung der Nutzdaten von `0x03410110` (Byte 1 > 8 — PV-Spannung, Zustandscode?) — braucht die Analyse des
   A-Micro.
2. Schreiber der Worte `0x20004020`–`0x2000402E`, die in `0x1851/0x1852AA00+n` gesendet werden (kein Code in diesem Image
   schreibt sie).
3. Bedeutung der SH36735-Konfigurationsregister 0x41–0x54, 0x95, 0x97 und des VADC-Kanals.
4. Polarität des „SELF“-Flags (`g_socStateFlags` Bit 5): die Kalibrierung setzt es nach Abschluss und die MOSFET-Gates
   brauchen es, aber `Master_Build_ChargeCurrentLimit` setzt die Ladegrenze auf 0, solange es gesetzt ist (außerhalb des
   Master-Run-Zustands).
5. „Optimierung der Zwangsladelogik“ aus dem Changelog: keiner konkreten Codeänderung zugeordnet.
6. Unterwegs gefundene Probleme im Referenzprogramm (D 118, dort nicht geändert): `APP_SystemFaultHalt` ist das C-`main()`;
   `g_ocpDelayVsCurrentTable` ist die SOH-Kurve; die Bitbang-Funktionen `KA495XX_SPI_*`/`RTC_*` sprechen sehr
   wahrscheinlich einen separaten RTC-Chip an, nicht den AFE; sechs Literal-Pools hinter nicht zurückkehrenden Aufrufen
   werden als Code dekodiert.
