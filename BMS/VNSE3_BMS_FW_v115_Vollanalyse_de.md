# BMS-FW V115 (Venus E 3.0) — Vollanalyse

> 🇬🇧 English version: [VNSE3_BMS_FW_v115_Full_Analysis.md](VNSE3_BMS_FW_v115_Full_Analysis.md)

**Stand:** 2026-10-02.

**Referenz-Images:** Venus D BMS 118 (`VNSD_BMS_FW_v118_Vollanalyse_de.md`), Venus A BMS 110.5
(`VNSA_BMS_FW_v1105_Vollanalyse_de.md`) · **Dekompilat/Listing:** `Dekompilate/VNSE3_BMS_115/` (nur lokal, nicht im
Repo) (653 Funktionen, `listing/` mit Flash-Disassembly + `globals_and_registers.txt`) · **Gegenstück:**
`Control_FW/VNSE3_Control_FW_v151_Vollanalyse_de.md`.

## 0. Zusammenfassung

- Image-Basis **0x08002800**, Image-Größe 0x14800 Byte (83.968 B), MCU **Nations N32G45x** (Cortex-M4F) — gleiche MCU,
  Toolchain (Keil ARMCC), RTOS (FreeRTOS, hier mit `configASSERT`-Strings und nur 5 Prioritäten) und Shell (letter-shell,
  User `BLUE`, leeres Passwort) wie beim Venus-D/A-BMS. Versions-String `Version: 115.0`, Build-Datum `Jun 29 2026 10:55:09`.
- **Codebasis:** Das E3-BMS ist eine **eigene Codebasis**, kein Geschwister-Build des D-BMS. Nur 17,7 % der Image-Bytes
  liegen in Segmenten, die mit D 118 identisch sind (Pointer/BL maskiert; A 1105: 16,9 %): C-Runtime, printf/dtoa,
  NTC-Tabellen, Teile von FreeRTOS. Ein Fuzzy-Funktionsmatcher ordnete 222 der 653 Funktionen D 118 zu (6 weitere nur
  mit A 1105: RTC-Helfer, Shell-Write-Callback, Passwortprüfung); alle wurden gegen den E3-Code geprüft. Die übrigen ~430
  Funktionen wurden anhand des E3-Codes benannt.
- **Batterie-Frontend:** **KA495XX** (gleiche Treiberfamilie wie D, SPI1 mit CRC-8 0xD5), **16 Zellen**, 4 Zell-NTCs,
  dazu ein MOS-NTC und ein Umgebungs-NTC am MCU-ADC.
- **Ein Pack.** Der Eigner gibt an (Hardware-Fakt, nicht aus der Firmware abgeleitet), dass der Venus E 3.0 genau ein
  eingebautes Pack hat und nicht erweitert werden kann. Die Firmware passt dazu: Der CAN-Kapazitäts-Frame `0x1802AA01` ist
  **konstant** (1 Pack, Online-Maske 1, 5120 Wh), der Pro-Pack-Burst wird nur für Pack 1 gesendet, und die Reste einer
  Multi-Pack-/Slave-Bus-Logik sind unerreichbar (§6). Der Pro-Pack-Sender behält die D-Grenze `pack_id != 0 && pack_id < 10`.
- **Anderer CAN-Protokollsatz:** Neben den Extended-Frames im D-Stil (`0x18xxAAyy`, `0x100x01AA`) beantwortet das E3-BMS
  eine **11-Bit-Anfrage `0x305` im Pylontech-Stil** mit `0x351/0x355/0x356/0x359/0x35C`, `0x360–0x363` (16 Zellspannungen)
  und `0x36A/0x36B` (Temperaturen), nimmt MOSFET-Befehle von `0xCF`, eine Abschaltung von `0x300` und einen Stromwert von
  der Extended-ID `0x04410110` entgegen. Es gibt **keine** Pack-Kodierung, keine Master/Slave-Rolle, keine Fehlersperre
  und kein RS485-Inter-Pack-Protokoll.
- **Sechs Batterie-Parametersätze („Profile“ 0..5):** SOH-Kurve, OCV-Tabelle, Kapazität-über-Temperatur-Tabelle sowie die
  Lade- und Entlade-Strommatrizen werden pro Profil ausgewählt (CAN `0x100601AA`, Shell `set_table`) und im
  EEPROM-Konfigurationsblock abgelegt (§7).
- **Changelog-Punkte im Code gefunden** (Herstellertext: „1. Optimised the start-up check when SOC reads 100 % or
  0 %; 2. Deep-discharge recovery together with the inverter; 3. Fixed a bug that could prevent charging and discharging
  after a full charge“):
  1. `SOC_AlgorithmEntry`: Beim Start wird ein gespeicherter SOC von 100 % bei max. Zelle < 3400 mV zu 99 %, ein
     gespeicherter SOC von 0 % bei min. Zelle > 2900 mV zu 1 % (nur bei gültigen AFE-Daten).
  2. `APP_ShutdownWake_Step` (1 s): Solange die Zellunterspannungs-Auslösung aktiv ist, setzt ein Ladestrom > 2,0 A einen
     200-s-Timer zurück und hebt die Unterspannungs-Entladesperre auf (beide MOSFETs dürfen leiten); ohne Ladestrom geht
     das BMS nach 200 s in den STOP-Modus (§9).
  3. Nicht sicher zugeordnet (Kandidaten: Full-Latch-Behandlung in der MOSFET-Steuerung, die „Report-Clamp“ in §5).
- **Alle 653 Funktionen benannt und typisiert**, 0 `FUN_*`, alle Prüf-Gates bestanden (§2.3). Tags: `NAME_VERIFIED` 653,
  `NAME_LOW_CONFIDENCE` 10.

## 1. Binär-Fingerabdruck

| Datei | Version | Größe | SHA-256 |
|---|---|---|---|
| `115_bms_VNSE3-0_Ac3.0BmsAPP_V115_ota_110451.bin` | 115 | 83.968 B (0x14800) | `cb50cc792e247d490abfe8cbc45d611dae49e7b8a8e615000cc5962ece9af4be` |

Image-Versions-Tag bei `0x08010000`: **`BMS1`** — derselbe Tag wie beim D-BMS (A: `BMS2`). Der CAN-OTA-Writer prüft
ebenfalls `BMS1` bei `0x0802B800` (§11).

## 2. MCU, Speicherkarte, Bearbeitungsstand

### 2.1 MCU: Nations N32G45x (Cortex-M4F)

Gleiche Belege wie beim D-BMS: Vektortabelle mit 102 Einträgen, N32-RCC/PWR-Layout in `SystemInit` (`SCB_VTOR = 0x08002800`),
Register- und IRQ-Namen aus dem N32G457-SVD. Verwendete Peripherie: SPI1 (KA495XX, Chip-Select PA4, Ready PA6), CAN1
(gesamter Verkehr), CAN2 (nur RX0-Interrupt, der einen Frame zwischenspeichert, nichts wertet ihn aus), USART1 (115200 8N1,
letter-shell-Konsole und Ymodem-Update), ADC1 mit DMA (Kanal 1, Kanal 2 = MOS-NTC, Kanal 9 = Umgebungs-NTC), TIM2/TIM3
(One-Shot-Callbacks), I2C-EEPROM (Bit-Banging auf PB10/PB11), GPIOD-Bit-Banging-Seriellbus zur RTC
(`KA495XX_SPI_*`/`RTC_*`-Funktionen, Code identisch zu D), IWDG (Prescaler 6, Reload 0x30D, gestartet in `System_Init_Main`).

### 2.2 Speicherkarte

| Bereich | Inhalt |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (nicht Teil des Images); Boot-Request-Seite `0x08002000` (`OTA_Set_Reboot_Params`) |
| `0x08002800`–`0x08016FFF` | Anwendungs-Image; Vektortabelle bei `0x08002800` |
| `0x08010000` | Image-Versions-Tag `'BMS1'` |
| `0x08013BB0`–`0x080145D7` | RCC-Tabellen, NTC-Kurven A/B (byte-identisch zu D) |
| `0x080145D8`–`0x080161B3` | E3-Tabellen: RTC-Tage pro Monat, `g_shellCrcSelfTestTable` (erwartete KA495XX-Registerwerte), Kanalmasken, CRC-8-Tabelle, NTC-Kurven C (−30…120 °C) und D (−40…100 °C), fünf Parametersätze (SOH, OCV, Lade- und Entlade-Matrizen mit Achsen) |
| `0x080169BC` | Keil-Regiontabelle (2 Regionen), `0x080169DC` `shellCommandTable` (39 × 16 B), `0x08016C4C` `.data`-Ladeimage |
| `0x0801E000` | OTA-Staging-Bereich (CAN-OTA und Ymodem); Tag-Wort bei `0x0802B800` |
| `0x20002800`–`0x20002BF7` | `.data` (0x3F8 B, im Image LZ-komprimiert; D: 0x3C4 B) |
| `0x20002BF8`–`0x2000C69F` | `.bss` (0x9AA8 B); Pro-Pack-Record-Array 20 × 0x5A B bei `0x20003144`, Konfigurationsblock `0x2000384C` (0xA8 B), SOC-Zustandsblock `0x2000397C` |
| `0x2000C6A0` | `__initial_sp` (Stack-Ende) |

Das RAM-Layout unterscheidet sich von D und A; RAM-Namen wurden nur dort von D übernommen, wo der ausgerichtete Code
dasselbe Objekt verwendet.

### 2.3 Bearbeitungsstand in Ghidra

- Programm `/VNSE3/BMS/BMS_115_VNSE3-0_Ac3.0BmsAPP_V115_ota_110451.bin`: **653 Funktionen, alle benannt**, jede Funktion hat
  einen benutzerdefinierten Prototyp (Task-Einsprünge `void f(void *pvParameters)`), Plate-Kommentare, 0 `FUN_*`, 0 Duplikate;
  613 RAM-Globals benannt und typisiert (Arrays/Strukturen für Zellspannungen, Pro-Pack-Records, Konfiguration, SOC-Block,
  Profiltabellen, FreeRTOS-Listen, Shell-Instanz).
- Gates (`ve3_bms_gates.py`, das aufgeteilte BMS-Gate-Set aus A 1105): 0 undefinierte Bytes, 0 verwaiste Instruktionen, 0
  Flüsse in Daten, 0 Funktionszeiger auf Nicht-Code, 0 reine Literal-Funktionen, 0 als Code dekodierte Literal-Pools, 0
  Aufrufe in Funktionskörper, 0 unbenannte referenzierte RAM-Globals, 0 Default-Prototypen, Task-Namen =
  `xTaskCreate`-Namensstrings, No-Return-Flags plausibel. Dekompilat-Export: 653 Funktionen, **0 Fehler**.
- Strukturelle Korrekturen: 39 per Tail-Call angesprungene Funktionen aus ihren Aufrufern herausgetrennt (der Compiler
  beendet viele Funktionen mit `b.w` zur nächsten), Literal-Pools hinter nicht zurückkehrenden Reset-Aufrufen als Daten
  typisiert (6 Stellen), der leere Shell-Passwort-String `""` des Users `BLUE` (Pointer `0x080161CD`) ist Daten, keine
  Funktion, die E3-Datentabellen typisiert (§2.2).

## 3. FreeRTOS-Tasks

`RTOS_CreateAllTasks` (`0x08009990`) erzeugt neun Tasks (`configMAX_PRIORITIES` = 5; D: zehn Tasks, 10 Prioritäten):

| Task | Einsprung | Stack (Wörter) | Prio | E3-Rumpf |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x08009BAC` | 0x200 | 4 | Tasten-Halte-LED + toter Slave-Poll (`APP_RS485_SlavePoll_Step`) |
| `RTOS_vTaskKeyScan` | `0x08009B9C` | 0x200 | 4 | Power-Taste PB13 (`KEY_ScanPowerButton`) + Power-Zustandsautomat |
| `RTOS_vTaskKA495XX` | `0x08009B90` | 0x200 | 4 | KA495XX-Zustandsautomat alle 5 ms |
| `RTOS_vTaskINVCanComm` | `0x08009B7C` | 0x200 | 4 | CAN-RX-Queue → Dispatcher, 5 ms (D: CanRec + CanSend) |
| `RTOS_vTaskRS485Comm` | `0x08009BCC` | 0x200 | 4 | letter-shell auf USART1, oder Ymodem-Empfänger nach `update` |
| `RTOS_vTaskSocAlgorithm` | `0x08009BF4` | 0x280 | 3 | SOC/SOH, 100 Ticks |
| `RTOS_vTaskProtectCheck` | `0x08009BBC` | 0x100 | 4 | Schutz + MOSFET-Steuerung, 100 Ticks |
| `RTOS_vTaskControlLogic` | `0x08009B3C` | 0x100 | 4 | Aggregation, Abschalt-/Wake-Logik, 1 s |
| `RTOS_vTaskDataSave` | `0x08009B64` | 0x100 | 0 | 1-s-Tick, Konsolen-Statusausgabe, IWDG-Fütterung |

## 4. Schnittstellen

### 4.1 CAN (nur CAN1)

Accept-All-Filter, RX-Queue 50 × 20 B; es gibt keine TX-Queue: `CAN_TX_SendExtFrame` / `CAN_TX_SendStdFrame` schreiben
direkt in eine freie Mailbox (ein Frame wird stillschweigend verworfen, wenn alle drei belegt sind). Eingehende Frames sind
N32-`CanRxMessage`-Strukturen, „Byte n“ = Struct-Offset 0x0B + n. `CAN_CommLossWatchdog`: CAN1-Re-Init nach 12000 Leerlauf-
Polls, System-Reset nach 20 Re-Inits.

**Extended-IDs, eingehend** (`CAN_RX_Handler` `0x08005878`, `CAN_RX_AuxCommand_Handler` `0x08008988`):

| ID | E3-Aktion | D 118 |
|---|---|---|
| `0x180101AA`…`0x180106AA`, `0x1801FFAA` | Statusanfrage → `CAN_TX_Orchestrator`: PF1–PF4 + Pro-Pack-Burst von Pack 1 | `0x1801nnAA` pro Pack, `0x1801FFAA` = Master-Heartbeat mit Enable/Entlade-% |
| `0x180107AA` | Selbsttest, Antwort mit gleicher ID, 1 Byte: 1 ok, 2 EEPROM, 3 Flash, 4 AFE-Fehler | — |
| `0x100101AA` | Systembefehl, Opcode = Byte 0 (Tabelle unten) | Systembefehl |
| `0x100201AA` | SOC auf Byte 0 % setzen (`CAN_CMD_02_Handler`), Echo | SOC-Reset |
| `0x100601AA` | **Batterieprofil wählen** 0..5 (Byte 0), Konfiguration ins EEPROM 0x258 speichern, Tabellen neu laden, Echo | — |
| `0x100301AA`, `0x100401AA`, `0x100501AA` | nicht behandelt | Entladegrenze / Befehl 04 / CAN-Neustart-Flag |
| Klasse 3 (`(id >> 12) & 0xF == 3`), Low-Byte `0x08` | RTC aus einer 32-Bit-Epoche setzen (+8 h) | — |
| Klasse 3, Low-Byte `0xCC` | RTC-Datum/Uhrzeit setzen (zwei Frames) | — |
| Klasse 3, Low-Byte `0xCE` | CAN-OTA-Blockwriter (`BMS_OTA_Flash_Writer`) | gleich |
| Klasse 3, andere Low-Bytes (1–5, 0x0A, 0xC4 …) | ignoriert | Status, Pack-Modus, Handshake, Werksreset, Notify, Aktivierung |
| `0x04410110` | Strom-/Verhältniswert vom Wechselrichter (VNS) → Float `0x2000299C`, vom Float-SOC-Integrator verwendet | — (A: `0x03410110` PV-Rescue-Anforderung) |

**Systembefehl-Opcodes `0x100101AA`** (`CAN_RX_System_Command_Handler`):

| Opcode | E3 | D 118 / A 1105 |
|---|---|---|
| 1 | **Abschaltanforderung** (`g_shutdownRequestFlag`): ControlLogic protokolliert Event 2, speichert die RTC-Epoche im EEPROM `0xFC08`, STOP-Modus mit Tasten-Wake, Reset | Adressanfrage |
| 2 | **Werksreset**: `BMS_Config_Params_Factory_Reset`, Echo, 500 ms, Power-Zustand 4 → Neustart | Werksreset + Neustart |
| 4 | Neustart (Power-Zustand 4) | Fehlermodus 4 |
| 0x0A | `CAN1_Reinit_OrSystemReset(0)` | — (A: CAN neu starten) |
| 0xA0 | Wake-/Re-Init-Anforderung: Peripherie neu initialisieren, IWDG-Reload 0x61A, Schlaf → der IWDG setzt den MCU zurück; direkt nach einem IWDG-Reset ignoriert | — |
| 3, 5–9, 0x0B, 0x0C | ignoriert (kein Relais, kein Lock/Unlock, kein Master/Slave) | Relais, Handoff, Unlock/Lock (8/9), A: Master/Slave (0x0B/0x0C) |

Das Opcode-1-Verhalten von D/A (Adressanfrage; auf Nicht-Master-Packs Teil der Pack-Kodierungssequenz) existiert nicht:
Beim E3 schaltet Opcode 1 die Batterie ab.

**Extended-IDs, ausgehend** (alle `0x18xxAAyy`, DLC 8, Little Endian):

| ID | Builder | Inhalt (E3) | Unterschied zu D |
|---|---|---|---|
| `0x1801AA01` | `CAN_TX_Build_PF1_PackMeasurements` | max. Packspannung, summierter Strom, max. NTC, **Float-SOC** × 10 | D: mittlerer SOC der Packs |
| `0x1802AA01` | `CAN_TX_Build_PF2_Capacity` | **konstant** `00 14 01 01 00 01 5A` + Statusbyte: 5120 Wh, 1 Pack, Online-Maske 0x0001, aktives Pack 1, Entladegrenze 90 %; Byte 7 = Tasten-/Abschaltzustand | aus Pack-Anzahl/Kapazität berechnet |
| `0x1803AA01` | `CAN_TX_Build_PF3_ChargeDischargeLimits` | Ladespannungsgrenze 57,6 V, Ladestrom, Entladestrom, Byte 6 Flags (Bit0 Laden, Bit1 Entladen erlaubt), Byte 7 = 0 | D Byte 7: Balancer-/Aggregations-Flags; D begrenzt bei 60,9 V |
| `0x1804AA01` | `CAN_TX_Build_PF4_ProtectWarnings` | Protect-Wort 1/2; **Byte 4 = 0xAA nach einem IWDG-Reset**, Bytes 5–7 = 0 | D Byte 6 = Online-Flags, Byte 7 = Fehlersperre |
| `0x1820…0x1823AA00+n` | `CAN_TX_PerPack_10Msgs` | Spannung, Strom, SOC, Zyklen / Grenzen, MOS-Bits / max-min-Zelle + Temperaturen / Protect 1/2, Alarmwort, **Versionsbyte 115 (0x73)** | gleicher Frame-Satz |
| `0x1830…0x1833AA00+n` | gleich | 16 Zellspannungen | gleich |
| `0x1840/0x1841AA00+n` | gleich | 4 Zell-NTCs / Mittelwert, Umgebungs- und MOS-NTC, **Byte 6 = Batterieprofil, Byte 7 = MOS-Aus-Bits** | D: Bytes 6–7 = 0 |
| `0x1852/0x1853AA00+n` | gleich | **nur E3:** SOH × 10 und die Leerschwelle 2800 mV / Zähler für geladene und entladene Energie | A: `0x1851/0x1852` mit anderem Inhalt |

**Standard-11-Bit-IDs** (`CAN_RX_StdFrame_Handler` `0x08008AE4`):

| ID | Richtung | Inhalt |
|---|---|---|
| `0x305` | ein | Anfrage → `CAN_TX_InverterProtocol_AllFrames` sendet die folgenden Frames im Abstand von 5 ms |
| `0x351` | aus | Ladespannungsgrenze (0,1 V) = **57,0 V + Pack-Anzahl × 0,1 V** (57,1 V bei einem Pack, Clamp 58,9 V), Ladestromgrenze, Entladestromgrenze, Zyklenzahl |
| `0x355` | aus | SOC × 10 (Float-SOC), SOH, Kapazität in Wh (5120), Byte 6 = 115 (Version) |
| `0x356` | aus | Packspannung (10 mV), Strom (0,1 A), max./min. Zelltemperatur |
| `0x359` | aus | Protect-Wort 1, Protect-Wort 2, Alarmwort |
| `0x35C` | aus | Byte 0 Flags Laden/Entladen erlaubt, Byte 2 Tasten-Halte-/Abschalt-Flag |
| `0x360`–`0x363` | aus | 16 Zellspannungen (mV) |
| `0x36A`, `0x36B` | aus | 4 Zell-NTCs / Umgebungs- und MOS-NTC |
| `0xCF` | ein | MOSFET-Befehl: Byte 0 ≠ 0 aktiviert die Steuerung durch Byte 1 (1 = Entlade-MOS ein, 2/3 = Lade-MOS ein) |
| `0x300` | ein | Byte 0 == 1 → Abschaltanforderung (gleicher Pfad wie Opcode 1) |

Es gibt keine periodische Übertragung; alles wird auf Anfrage gesendet.

### 4.2 USART1: Konsole und Ymodem (kein RS485-Inter-Pack-Protokoll)

`RTOS_vTaskRS485Comm` bedient die letter-shell (115200 8N1). Der Shell-Befehl `update` schaltet den Task auf einen
Ymodem-Empfänger um (`OTA_Ymodem_Receive`: erstes Paket Name + Größe, Limit 1 MB, Daten in den Staging-Bereich `0x0801E000`,
endet in `OTA_Set_Reboot_Params` und einem Reset). Das D-Frame-Protokoll (Startbyte `'s'`, XOR-Prüfsumme, Opcodes 0x00–0x34)
ist weiterhin als Sender einkompiliert (`RS485_Send_Packet_Bus1`, `RS485_Build_And_Send_Frame`), hat aber keinen
funktionierenden Transport: Der Ringpuffer-Treiber kennt nur Port 0, der Sender nutzt Port 1, USART2/UART4 werden nur
de-initialisiert.

### 4.3 Debug-Shell (letter-shell)

User `BLUE` (leeres Passwort), 39 Tabelleneinträge: 10 Tastenbelegungen, `help`, `clear` und **27 Befehle** — keiner der
D/A-Pack-Befehle (`bat_data`, `inq_uid`, `set_work_pack`, `factory`, `lock_set`, …):

| Befehl | Handler | Funktion |
|---|---|---|
| `reset` | `CLI_Cmd_Reset` | NVIC-System-Reset |
| `sox` | `CLI_Cmd_Sox` | Ausgabe des SOC/SOH-Blocks |
| `set_soc` | `SOC_SetSocPercent` | SOC setzen |
| `cyc_printf` | `CLI_Cmd_CycPrintf` | periodische Statusausgabe ein/aus |
| `protect` | `CLI_Cmd_Protect` | aktive Schutzfunktionen als Text |
| `update` | `CLI_Cmd_Update` | Ymodem-Firmware-Update |
| `rtc_read` / `rtc_write` | `CLI_Cmd_RtcRead` / `CLI_Cmd_RtcWrite` | RTC |
| `chg_sop` / `dsg_sop` | `CLI_Cmd_ChgSop` / `CLI_Cmd_DsgSop` | Strommatrix-Abfragen |
| `self_check` | `CLI_Cmd_SelfCheck` | wie CAN `0x180107AA` |
| `restore` | `CLI_Cmd_Restore` | Werks-Schutzblock + SOC-Kalibrierlog löschen, Reset |
| `set_table` | `CLI_Cmd_SetTable` | Batterieprofil 0..5 |
| `chg_mos` / `dsg_mos` | `CLI_Cmd_ChgMos` / `CLI_Cmd_DsgMos` | Lade-/Entlade-MOS zwangsweise aus |
| `can_send` / `can_err` / `ban_can` | `CLI_Cmd_CanSend` / `CLI_Cmd_CanErr` / `CLI_Cmd_BanCan` | Testframe `0x112233`, Ausgabe des CAN-Fehlerflags, Slave-Poll-Sperre |
| `set_pro_val` | `CLI_Cmd_SetProVal` | Schutzwerte bearbeiten (nur RAM) |
| `reson` | `CLI_Cmd_ResetReason` | Reset-Ursache (IWDG, WWDG, Soft, POR, Hard, Sleep) |
| `log_prt` / `log_write` / `log_read` / `log_clearAll` | `EventLog_*` | Event-Log |
| `shell_100ms_tick`, `chgDsgIdleTime` | `CLI_Cmd_Shell100msTick`, `CLI_Cmd_ChgDsgIdleTime` | Debug-Werte (Tabellenbeschreibung fälschlich „log_ClearAll“) |

### 4.4 EEPROM-Layout (I2C, Bit-Banging)

| Adresse | Inhalt |
|---|---|
| `0x0000` | Zustandsblock 0x50 B (bei Sleep-/Wake-Pfaden geschrieben); Byte 0 auch vom EEPROM-Selbsttest verwendet |
| `0x012C` | Hilfsblock 0x1C B (Defaults 4000/100/120, kein Leser gefunden) |
| `0x0258` (600), 0xA8 B | Konfigurationsblock: Magic `0x62`, `'s'`, Zellanzahl 16, Profilindex, Schutzwerte mit Verzögerungen, Einerkomplement-Prüfsumme (`BMS_Config_Params_Load_Or_Default`) |
| `0x0400` + 0x14 × i, i = 0..49 | SOC-Kalibrierungsdatensätze (Real-/Anzeige-/Max-/Min-SOC, Durchsatz) — gleiche Slots wie D |
| `0x1000` + 8 × i, i = 0..49 | Event-Log-Index (gepackte Zeit, Event-ID, Datensatzadresse) |
| `0x2000`–`0x2CE3` | Event-Datensätze 0x42 B (Zeit, SOC, Strom, max./min. Zelle, NTCs, 16 Zellen, Protect-Wörter) |
| `0xFC00` / `0xFC04` | Zähler für geladene / entladene Energie (bei D ist `0xFC00` der Fehlersperren-Marker) |
| `0xFC08` | RTC-Epoche der letzten Abschaltung |
| `0xFC0C` | Float-SOC × 10 für den nächsten Start |

Event-IDs 0..10 werden akzeptiert (A: < 6); verwendet: 1 Schutzänderung / Tasten-Abschaltung, 2 CAN-Abschaltung, 3 SOC-Abfall auf
0 %, 4 SOC-Sprung > 2 %.

## 5. Zellüberwachung — KA495XX

- SPI1, gepollt; Lese-Frame `E1, reg<<1, CRC` → `hi, lo, CRC`, Schreib-Frame `E0, reg<<1, hi, lo, CRC`; CRC-8-Polynom
  0xD5 (wie D). Shadow-Register-Cache mit 96 Einträgen; die Konfiguration wird in jedem Zyklus neu geprüft (Register 7, 0x0A,
  0x12, 0x18) und gegen die Tabelle `g_shellCrcSelfTestTable` mit 27 Einträgen (D-Name beibehalten; es ist die Tabelle der
  erwarteten Registerwerte). Die Tabelle entspricht D 118 bis auf **Register 0x0E: 0x0400 (D: 0x0B00)**;
  `KA495XX_RegisterInit` schreibt denselben Wert.
- Zellanzahl aus der Konfiguration (Default 16, `g_afeCellChannelMaskTable` 4..17 Zellen, identisch zu D: bei 16 Zellen
  wird AFE-Kanal 14 übersprungen). Zell-mV = Code × 5000 >> 14. 4 Zell-NTCs (NTC-Kanalmaske 0x1F).
- Messzyklus bei jedem 5-ms-Task-Tick: 6 ADC-Runden, Wert = (Summe − Max − Min) >> 2; Strom mit Gain-Faktor
  (begrenzt auf 0,5..2,0) und einer festen Korrektur **1,08** (D: 0,88).
- **Balancing** (`KA495XX_CellBalanceDecision`): nur wenn die Zellspreizung ≥ 30 mV und der Strom > 1,0 A; eine Zelle wird
  bei ≥ 3501 mV und ≥ 30 mV über dem Minimum eingeschaltet, unterhalb von 3500 mV oder bei < 20 mV über dem Minimum
  ausgeschaltet; abwechselnd ungerade/gerade Kanäle. Schwellen identisch zu D.
- **Report-Clamp** (`BMS_CellVoltageReportClamp` `0x0800A1E0`, nur E3): eine einmalige Prüfung nach dem Start (Zustand 0xFF):
  Liegt die Packspannung bei ≥ 51,80 V oder eine Zelle bei ≥ 3700 mV für etwa 1000 Zyklen, werden die veröffentlichten
  Zellspannungen eingefroren und nach weiteren 300 Zyklen alle veröffentlichten Zellen auf **4000 mV** gesetzt; sie wird
  freigegeben, wenn die Bedingung 20000 Zyklen lang falsch war. Ist die Bedingung zuerst falsch (600 Zyklen), endet die
  Prüfung endgültig. Der Schutz verwendet die Rohwerte. Die Absicht (dem Wechselrichter „voll“ signalisieren?) ist nicht
  dokumentiert.
- Im Image existiert keine Heizungssteuerung.

## 6. Pack-Management — einzelnes Pack

- `g_pack0SlotId` (`0x20003144`) wird beim Start auf 0xFF gesetzt, und nur die unerreichbaren Adresszustände würden es ändern.
- `APP_PowerAndAddressStateMachine` (`0x080064D0`, KeyScan-Task): Die Zustände 1 und 2 (Slave-ID-Bestätigung / Zuweisung mit
  PB12-Pulsen und Frames 0x27/0x25 an bis zu 20 Slaves) werden **nie betreten** — geschrieben werden nur 0, 3 (Tasten-Abschaltung)
  und 4 (Neustart). `APP_RS485_SlavePoll_Step` (LED-Task) und `BMS_AggregatePackData` (Schleife über `g_slavePackCount` mit
  Online-Maske) laufen, aber die Slave-Anzahl bleibt 0, sodass nur Record 0 (eigenes Pack) beiträgt.
- Pro-Pack-Record-Array: 20 Records × 0x5A B (`0x20003144`–`0x2000384B`, D: Records mit 0x60 Byte). Record 0 wird von
  `BMS_UpdatePublishedMeasurements` gefüllt; die Records 1..19 werden nur vom toten Aggregationspfad gelesen.
- Keine Fehlersperre, kein Relais-/Vorlade-Handshake zwischen Packs, kein Handoff, keine Pack-Auswahl.

## 7. SOC/SOH

- Gleiche SOC-Bibliothek wie D (Orchestrator, Coulomb-Zählung, Glättung, Lade-/Entlade-Integration, Energiezähler,
  OCV-Initialisierung, Kalibrierlog mit 50 Slots), mit E3-Konstanten: Nennkapazität **100 Ah** (D: 50 Ah), Zyklenzahl =
  Durchsatz / 10 / 10,24 (D: 5,12), voll bei max. Zelle ≥ 3600 mV, leer bei min. Zelle ≤ 2800 mV, Anzeige-SOC bis zum
  Vollzustand auf 99 % begrenzt, Untergrenzen 1 % / 12 %.
- Ein **zweiter, Float-SOC** (`0x20002B78`, `SOC_FloatIntegrator_Callback` alle 50 ms) integriert den vom Wechselrichter
  empfangenen Stromwert (`0x04410110`) mit harten Synchronisationen bei 100 % / 0 % (Zellschwellen oben). Er ist der SOC, der
  in `0x1801AA01`, `0x1820AA0n` und `0x355` gesendet wird; der Struct-SOC geht ins Event-Log. Beide können voneinander
  abweichen.
- Batterieprofile (`SOC_LoadProfileTables` `0x0800E0E4`): Profil 0, 1 und ungültige Werte verwenden Satz 0; 2 → Satz 1, 3 → Satz 3,
  4 → Satz 2, 5 → Satz 4. Pro Satz: SOH-Kurve (0 → 100 %, 1000 → 95 %, 2000 → 90 %, 5000 → 80 %, 20000 Zyklen → 75 %; Satz 4:
  6000 → 70 %), Kapazität-über-Temperatur-Tabelle (in `.data`), OCV-Tabelle 11 × 4 Spalten, Lade-Matrix 14 × 11, Entlade-Matrix
  17 × 11.

## 8. Lade-/Entladegrenzen

| Punkt | Venus E3 115 | Venus D 118 |
|---|---|---|
| Skala der Stromgrenze | Matrix-% × 10 in 0,1 A → 100 % = **100,0 A** | % × 5 → 50,0 A |
| Lade-Matrix (Satz 0) | 14 Temperaturzeilen (−10…60 °C) × 11 SOC-Spalten: 0 % unter −10 °C, 5 % bei 0 °C, 30 % bei 10 °C, 80 % bei 20–25 °C, 100 % bei 30–50 °C (80 % über 90 % SOC, 50 % über 95 %), 30 % bei 55 °C, 0 % bei 60 °C | 18 × 12 Matrix, max. 50 A |
| Entlade-Matrix (Satz 0) | 17 Zeilen (−30…60 °C) × 11 SOC-Spalten: 0 bei ≤ −20 °C, 10–50 % bei −10…0 °C, 100 % bei 15–50 °C, 50 % bei 55 °C, 0 bei 60 °C | 18 × 5 Matrix über die Zellspannung |
| Ladespannungsgrenze | 57,6 V (PF3); `0x351` meldet 57,0 V + Pack-Anzahl × 0,1 V | 57,6 V, CAN-Clamp 60,9 V |
| Ladestrom | 0, wenn Laden nicht erlaubt ist (voll, Schutzfunktionen) | — |

## 9. Schutzfunktionen

Standard-Schwellen (`BMS_Config_Params_Factory_Reset`, Entprellung in 100-ms-Ticks; Werte wie bei D, sofern nicht anders vermerkt):

| Gruppe | Warnung | Auslösung | Freigabe |
|---|---|---|---|
| Zell-OVP | 3600 mV / 1 s | 3650 mV / 3 s, 3700 mV / 1 s | 3350 mV / 1 s |
| Zell-UVP (Laufzeit, mittlerer NTC > 0 °C) | 2700 mV | 2500 / 2450 mV | 3000 mV |
| Zell-UVP (mittlerer NTC ≤ 0 °C) | 2400 mV | 2200 / 2000 mV | 3000 mV |
| Entladestrom | 55 A / 3 s | 80 A / 5 s, 100 A / 0,5 s | Latch 60 s |
| Ladestrom | 55 A / 3 s | 65 A / 1 s, 100 A / 0,5 s | Latch 60 s |
| MOS-NTC | 90 °C | 115 °C | 85 °C |
| Umgebungs-NTC | 61 / −26 °C | 65 / −30 °C | 60 / −25 °C |
| Zell-NTC beim Entladen (5 s) | — | hoch 60 / 65 °C, tief −15 / −20 °C | 57 / −10 °C |
| Zell-NTC beim Laden (5 s) | — | hoch 55 / 60 °C, tief −15 / −20 °C | 50 / −10 °C |

Die D-Laufzeit-UVP-Werte sind 3100/3000/2500 mV (> 0 °C); der E3 hat keine Temperaturkompensation der OVP. Weitere Prüfungen:
Zellbruch (Pack ≥ 35,00 V und eine Zelle < 1500 mV), Zelldifferenz > 1000 mV, NTC-Ausreißer-Ausschluss (≥ 30 °C entfernt),
NTC-Durchbruch, MOSFET-Durchbruch (Strom > 5,0 A bei ausgeschaltetem MOSFET; D: 3,0 A).
**Kurzschluss** (`Protect_ShortCircuitCheckAndRecover`): Entlade-MOS ein und PB1 low für 5000 Timer-Ticks → Flag,
beide MOSFETs aus; danach bis zu 10 Wiederherstellungsversuche mit dem Vorlade-Ausgang PB2, und die MOSFETs werden wieder
eingeschaltet (D setzt nur das Flag).
**MOSFET-Steuerung** (`MOS_ControlFromProtectionFlags`, 100 ms): PC5 = Lade-MOS, PB0 = Entlade-MOS, PB2 = Vorladung;
Entladen ein über 7 Ticks Vorladung. Blockiert wie in den Code-Kommentaren aufgeführt (OVP/UVP, Überstrom, Temperaturen,
MOSFET-Durchbruch, Full-Latch, Shell-Force-Flags, AFE-Kommunikationsverlust, ungültige Daten).
**Tiefentladungs-Rettung** (Changelog-Punkt 2, `APP_ShutdownWake_Step`): Solange die Zell-UVP-Auslösung aktiv ist, löscht ein Ladestrom
> 2,0 A die UVP-Entladesperre und startet einen 200-s-Timer neu; nach 200 s ohne Ladestrom geht das BMS in den STOP-
Modus (`System_EnterStopMode_ThenReset`). D: nur eigene MOSFETs unter 2,0 A, Einzel-Pack → Fehlermodus 3 nach 200 s; A: wartet
120 s auf CAN `0x03410110`.
**Power-Taste** PB13: 3,0–6,99 s → Abschaltung (Event-Log, EEPROM-Stempel `0xFC08`, STOP-Modus mit Tasten-Wake), ≥ 7 s → Neustart.

## 10. Pflichtprüfungen (FW_Diff_Workflow)

| # | Prüfung | E3-BMS 115 |
|---|---|---|
| 6a | Grenze im Pro-Pack-Sender `CAN_TX_PerPack_10Msgs` | **`pack_id != 0 && pack_id < 10`** (Packs 1..9, Record-Index pack_id − 1) — wie D 118 (A 1105: `< 7`). Der einzige Aufrufer (`CAN_TX_Orchestrator`) übergibt immer **1**, sodass nur Pack 1 gesendet wird. |
| 6b | Pack-/Slot-Grenzen, Records | Record-Array 20 × 0x5A B; der Adresszuweisungs-Code würde bis zu 20 Slaves akzeptieren, ist aber unerreichbar; PF2 meldet konstant 1 Pack / Maske 1 |
| 6c | Control-Seite | E3 Control 151 behält 8 Records × 0x60 B und die Blöcke 34000–34699; mit diesem BMS lesen die Packs 2..7 immer 0 |
| 6d | Zellen pro Pack | **16** (Konfiguration, AFE-Kanalmaske, 16 Zell-Slots in 0x1830–0x1833 und 0x360–0x363) |
| — | Konsistenz mit einem festen Pack (Aussage des Eigners) | ja: konstantes PF2, Burst nur für Pack 1, keine Pack-Kodierung, keine Master/Slave-Rolle, Slave-Bus ohne Transport |
| 8 | Erneute Prüfung nicht funktionaler Register | es existiert noch keine E3-Register-CSV; die BMS-seitigen Fakten zu den offenen D/A-Registern stehen in §13 |

## 11. OTA/Update

- **CAN-OTA** (Klasse 3, Opcode 0xCE, `BMS_OTA_Flash_Writer`): 1-KB-Blöcke mit CRC16/Modbus, Staging bei `0x0801E000`,
  Prüfung des Tags **`'BMS1'`** bei `0x0802B800` und der Gesamt-CRC, dann `OTA_Set_Reboot_Params` (Boot-Request-Seite
  `0x08002000`, Magics `0x11223344`/`0x55667788`) und Reset — gleicher Ablauf und **gleicher Tag wie beim D-BMS**, sodass die
  Tag-Prüfung allein nicht verhindert, dass ein D-Image akzeptiert wird (A verwendet `BMS2`).
- **Ymodem** über die Konsole (`update`), gleicher Staging-Bereich.

## 12. Sicherheitshinweise

- Shell-User `BLUE` ohne Passwort; `restore`, `set_table`, `set_pro_val`, `chg_mos`/`dsg_mos`, `rtc_write`, `log_write`
  und `update` sind über die Konsole erreichbar.
- CAN-Befehle ohne Authentifizierung können die Batterie abschalten (Opcode 1, `0x300`), auf Werkseinstellungen zurücksetzen
  (Opcode 2), neu starten, das Batterieprofil dauerhaft ändern (`0x100601AA`), den SOC setzen, die MOSFETs schalten (`0xCF`)
  und eine neue Firmware schreiben (0xCE).

## 13. Register hinter BMS-Frames (Unterschiede zu D und A)

Der E3 Control 151 verwendet dieselbe Deskriptor-Tabelle und denselben `0xAA`-CAN-Dispatcher wie D 151 (17 Einträge, darunter
`0x1852` → Record +0x58 und `0x1853` → Record +0x50, die kein Modbus-Register bereitstellt). Mit BMS 115:

| Register | Quelle | E3-BMS 115 | D 118 / A 1105 |
|---|---|---|---|
| 32100, 32101, 32104, 32108/35010 | PF1 | real (eigenes Pack; SOC = Float-SOC) | real (Stack-Werte) |
| 32105 | PF2 b0–1 | konstant **5120** | Packs × 2560 (A: × 2080) |
| 32109 / 32110 / 32111 | PF2 b2 / b3–4 / b5 | konstant **1 / 1 / 1** | real |
| 32106, 32107, 35111, 35112 | PF3 | reale E3-Grenzen (0,1 A, bis 100 A) | real |
| 35110 | PF3 b0–1 | konstant 576 (57,6 V) | D konstant 576, A dynamisch |
| 32112 | PF4 b7 | **immer 0** (keine Fehlersperre) — beim E3 FUNCTIONLESS | reales Sperr-Flag |
| 32113 / 30109 | PF4 b6 | **immer 0** | D: Online-Flags |
| PF4 b4–5 (`g_pgn1804InfoBatWarn1`, Control-intern) | PF4 | 0x00AA nach einem IWDG-Reset | 0 |
| 34000–34003, 34005–34008, 34011–34016, 34018–34033, 30100–30108, 37005–37012 | Pro-Pack-Frames Pack 1 | real; Zellen 1–16 genutzt; 34002/37005 tragen den Float-SOC | real |
| 34004 / 30210 | 0x1821 b6 | MOS-Ausgangsbits (Bit0 PC5 Laden, Bit1 PB0 Entladen) | real |
| 34009 / 37011 | 0x1823 b4–5 | Alarmwort (Warnbits von C6/C7) | Warnwort |
| 34010 / 37012 / 30204 | 0x1823 b6–7 | **115** | 118 / 110 |
| **34017 / 30106** | 0x1841 b6–7 | **beim E3 FUNCTIONAL**: Low-Byte = Batterieprofil 0..5, High-Byte = MOS-Aus-Bits | FUNCTIONLESS (immer 0) |
| 34100–34699 (Packs 2–7) | — | immer 0 (nie gesendet) | D: real für vorhandene Packs; A: Packs 2–6 |
| **45011** | `0x100601AA` | **beim E3 FUNCTIONAL**: wählt das Batterieprofil (Control-Remap 1→1, 2→2, 3→4, 4→0, 5→5, sonst → 0; Profil 3 ist über Modbus nicht erreichbar) und speichert es im BMS-EEPROM | beim D UNKNOWN (kein BMS-Empfänger) |
| 45028 | `0x100101AA` 02 | FUNCTIONAL (Werksreset + Neustart) | gleich |
| 45027 | Klasse 3 Func 0x0A | FUNCTIONLESS (nicht behandelt) | D: Master-Neustart |
| 45029 | `0x100101AA` 03 + Encode | FUNCTIONLESS (keine Pack-Kodierung) | D: Pack-Kodierung |
| 45030 | `0x100101AA` 0x0B/0x0C | FUNCTIONLESS | D: keine, A: Master/Slave |
| 45031 | 0x04 dann 0xA0, Warten auf PF4 b4 = 0xAA | **beim E3 behandelt**: 0x04 startet neu, 0xA0 lässt das BMS schlafen, bis der IWDG es zurücksetzt, danach PF4 b4 = 0xAA — ob die 5-s-Fenster des Control ausreichen, ist offen (vermutlich nicht: IWDG ~10 s) | FUNCTIONLESS |

## 14. Zusammenspiel mit dem Venus-E3-Control 151 und dem Micro (VNS)

- Control: gleiche Anfrage-/Heartbeat-Frames wie bei D; das BMS beantwortet jedes `0x1801xxAA` mit einem Pack. Entlade-%,
  Master-Heartbeat und EMS-Flags des D (Nutzdaten von `0x1801FFAA`) werden nicht ausgewertet.
- Wechselrichter/Micro (angenommener Sender, noch nicht gegen den E3-Micro 119 verifiziert): `0x305`-Anfrage mit der
  Antwort im Pylontech-Stil, `0xCF`-MOSFET-Befehle, `0x300`-Abschaltung, `0x04410110`-Stromwert für den Float-SOC. Bei der
  Tiefentladungs-Rettung muss der Wechselrichter innerhalb von 200 s > 2,0 A Ladestrom liefern.

## 15. Offene Punkte

1. Sender und genaue Bedeutung von `0x04410110` (Bytes 4–5 vorzeichenbehaftet, 6–7 vorzeichenlos, skaliert × 10000 / …) sowie von `0xCF`/`0x300`/`0x305`
   — benötigt die Analyse des E3-Micro 119.
2. Changelog-Punkt 3 („no charge/discharge after a full charge“): keiner konkreten Code-Änderung zugeordnet.
3. Zweck der 4000-mV-Report-Clamp nach dem Start (§5) und der ungelesenen EEPROM-Blöcke `0x0000` und `0x012C`.
4. 45031-Timing (das Control wartet 5 s, das E3-BMS braucht ein IWDG-Timeout von etwa 10 s).
5. Einheit des Kurzschluss-Timer-Ticks (5000 Ticks; TIM2 mit 2 MHz würde 2,5 ms ergeben — nicht verifiziert).
6. Unterwegs gesehene Benennungsprobleme im Referenzprogramm (D 118, dort nicht geändert): `CAN_SetTxMailboxTDLR/TIR` und
   `CAN_Set_FilterBank_FR1..3`/`CAN_SetTxMailboxTDHR` sind GPIO-BRR/BSRR- und TIM-CCR-Setter, `GPIO_Init_Output_PP_50MHz` und
   `GPIO_Init_Input_Floating` sind bei D vertauscht, `APP_ShellTable_Init_Then_Main` ist `__scatterload`, `APP_SystemFaultHalt` ist
   `main()`, `NTC_ConvertAndUpdatePackTempExtremes` (D) ist eine OCV-Grenzen-Rekalibrierung (`SOC_RecalibrateBoundsFromOcv` im
   E3), `g_shellCrcSelfTestTable` ist die Tabelle der erwarteten KA495XX-Register, `HAL_UART4_SendByte_Blocking` verwendet USART1.
   Das E3-Programm behält die D-Namen, wo das Objekt dasselbe ist (die Plates sagen, was der Code wirklich tut).
