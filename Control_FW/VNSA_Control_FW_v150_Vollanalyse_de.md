# Control-FW v150 (Venus A) — Vollanalyse und Unterschiede zu Venus D v150 und A v150.9

> 🇬🇧 English version: [VNSA_Control_FW_v150_Full_Analysis.md](VNSA_Control_FW_v150_Full_Analysis.md)

**Status:** abgeschlossen 2026-10-11. Alle Aussagen wurden am Code dieses Images geprüft; offene Punkte stehen in §12.
**Binary:** `150_control_VNSA-0_app_0150_0811_114724.bin` (Archiv `Marstek FW Archiv/firmwares/VNSA-0/Control/150/`, Release v150)
**Referenzen:** Venus D Control v150 ([VNSD_Control_FW_v150_Vollanalyse_de.md](VNSD_Control_FW_v150_Vollanalyse_de.md)) — gleiche Codebasis und gleiches RAM-Layout;
Venus A Control v150.9 ([VNSA_Control_FW_v1509_Vollanalyse_de.md](VNSA_Control_FW_v1509_Vollanalyse_de.md)) und v151 ([VNSA_Control_FW_v151_Vollanalyse_de.md](VNSA_Control_FW_v151_Vollanalyse_de.md))
**Diffs:** [VNSA_Control_Diff_v150_v1509_de.md](VNSA_Control_Diff_v150_v1509_de.md) (v150 → Beta v150.9), [VNSA_Control_Diff_v150_v151_de.md](VNSA_Control_Diff_v150_v151_de.md) (v150 → v151)
**Ghidra:** `/VNSA/Control/Control_150_VNSA-0_app_0150_0811_114724.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** `Dekompilate/VNSA_Control_v150/` (nur lokal, nicht im Repo): 1935 Funktionen, 0 Dekompilierfehler, + `listing/` (Flash-Listing, `globals_and_registers.txt`)

---

## 0. Zusammenfassung

- **Release-Reihenfolge des A-Control:** v150 (Build 11. Aug 2026) → Beta v150.8 (19. Aug) → Beta **v150.9** (8. Sep) → v151 (28. Sep).
  v150.9 ist also eine Beta *nach* v150 auf dem Weg zu v151, keine Beta von v150. Alle Diffs unten laufen in dieser zeitlichen Richtung.
- **A v150 ist der Venus-A-Build von Venus D v150.** 98,2 % des Images stimmen mit D v150 überein, sobald Pointer und BL-Offsets maskiert
  sind (191 Segmente); `.data`, `.bss` und das externe SRAM haben dieselben Größen, jede RAM-Variable liegt an derselben Adresse wie bei
  D v150 (initialer SP 0x2001F7E8). Die Modbus-Deskriptortabelle ist bytegleich mit D v150. Von den 1954 D-v150-Funktionen sind 1899 in Name
  und Code gleich, 31 weichen ab (A-Konstanten, RS485 an USART1, umbenannte Aufrufziele), 5 tragen die aktuellen A-Namen (z. B.
  `OTA_Is_VNSA_Model`, `USART1_IRQHandler`), 5 sind 2-Byte-Code-Folding-Stubs, die in ihre Handler eingegliedert sind, und 14 sind
  wegkompiliert (Tasten-/Schutzüberwachung, RS485-Richtungspin).
- **Die A-spezifischen Unterschiede sind dieselben Compile-Zeit-Konstanten wie in A v150.9/v151:** Leistungsgrenzen 1500 W, Leistungsklassen
  800/1200/1500 W, 13 Zellen pro Pack, USART1 ohne Driver-Enable-Pin, keine Tasten-/Schutzüberwachung, Identität `VNSA`, Klemme des
  Einspeisebegrenzers 2400 W. Ein A-eigener Unterschied, der keine Konstante ist: nach längerem Ausfall des Cloud-Uploads **startet**
  `HTTP_ClientTask_Dispatcher` **das Gerät neu** (`System_Reboot`), während D v150 nur den CH395-Ethernet-Chip zurücksetzt (§4.5).
- **Modbus (registersichtbar):** dieselben 246 Leseregister wie v150.9/v151 bis auf **32102** (`float32`, 2 Register; `i16` ab v150.9).
  `Write_Handler` hat die A-Grenze für **44003 (über 1500 W abgewiesen)**, 44002 nimmt bis 2500 W an (Rohwert an den Wechselrichter,
  EEPROM-Kopie auf 1500 W begrenzt) — genau wie in v150.9/v151. Gegenüber v150.9/v151 fehlen das Schreibregister **47001** (CAN 0x18) und der
  eigene Fall für 45030 (0xAFE6, hier über den Bereichspfad des 45000-Blocks mit derselben Wirkung erreicht).
- **Cloud:** Telemetrie-Upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`, `getDateInfo` und alle anderen
  Endpunkte auf `hamedata.com` (wie D v150 und A v150.9). Geräte-Quell-ID `VenusA-<serial>` (v150.9: `VNSA-0-<serial>`).
- **Gegenüber v150.9** sind 42 Funktionen geändert und 6 in v150.9 neu (namensgepaart, vollständig; Details im Diff-Dokument):
  EPS-Reserve-Logik, YMODEM-Bin-Typ-Prüfung, `cd=0x42` Wechselrichter-Debuginfo und `read_inv1/2`, Schreibregister 47001, Heap 70 → 80 KB,
  Verbindungs-Watchdog, Ratenbegrenzung der Zählerabfrage, Modbus-TCP-Keepalive, Statusbericht-Feld `df`.

## 0a. Checkliste (Pflichtprüfungen des FW-Diff-Workflows, A v150)

| # | Frage | Antwort A v150 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register anders? | Ggü. D v150: **nein** (Deskriptortabelle bytegleich, `Write_Handler` gleicher Code bis auf die 44003-Grenze 1500 W). Ggü. A v150.9/v151: **32102** ist `float32` (2 Register) statt `i16`; **47001** (0xB799) ist nicht implementiert (kein Fall im `Write_Handler`); 44002/44003 verhalten sich wie in v150.9/v151. Lesepfad (`FC03_Read_Handler`, `Read_Serializer`) gleich. Register 30200 liest das Versionswort 0x96 = **150**. | §5 |
| 2 | BLE anders? | Gerätename `MST_VNSA_` + 4 ID-Zeichen über das feste Format `AT+QBLENAME=MST_VNSA_%c%c%c%c`. Leistungsklassen und Grenzen wie v150.9 (Kmd 0x15: 800 → 1, 1200 → 4, 1500 → 5; 0x16 300–1500 W; 0x17 ≤ 1500 W). Kmd `T` (0x54) setzt die EPS-Hysterese noch zurück (in v150.9 entfernt). | §4.1, §4.6 |
| 3 | AC-Einspeisung / Leistungsgrenzen? | **1500 W** überall (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock`, MQTT `md=`, JSON-RPC ±1500 W, Fernbefehl ±1500 W, Modbus 44003). `Grid_Export_Power_Limiter`: PV-Startwert zuerst auf 2500 W begrenzt (wie D v150), obere Klemme **2400 W** (D: 4000 W). | §4.1 |
| 4 | Firmware-Update per BLE möglich? | **Ja.** OTA-/BLE-OTA-Code gleich D v150 bis auf die Modellprüfung `VNSA-0` (`OTA_Is_VNSA_Model`). YMODEM hat noch **keine** Bin-Typ-Prüfung (kommt mit v150.9). | §4.6 |
| 5 | Cloud-Domains? | Upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`; `getDateInfo`, `backup-configs`, `schedulings/policies`, `getDevicesApply`, FC41D-OTA auf `hamedata.com`; nur Servertypen 0–2 gültig. Wie D v150 und A v150.9; `getDateInfo` wandert erst in v151 zu marstekcloud. | §4.5 |
| 6 | Pack-Kapazität? | Control: 7 Datensätze je Pack (Modbus 34000–34699), `BMS_CellVoltage_MinMax_PerPack` prüft **13 Zellen für Packs 1..6** — bis v151 unverändert. Die Pack-Grenze selbst liegt im BMS (A-BMS 1105: `pack_id != 0 && pack_id < 7` = Packs 1..6); zu diesem Release gehört kein BMS-Image. | §6 |
| 7 | Diff-Tabelle | [VNSA_Control_Diff_v150_v1509_de.md](VNSA_Control_Diff_v150_v1509_de.md), [VNSA_Control_Diff_v150_v151_de.md](VNSA_Control_Diff_v150_v151_de.md) (beide namensgepaart, vollständig). | — |
| 8 | Erneute Prüfung nicht funktionaler Register | Die A-Registerklassifikation (FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL) wurde an v150.9/v151 erstellt. Für v150 liegen die Gründe im selben Code (Deskriptortabelle, `Write_Handler`, CAN-Sender/-Parser, Pack-Datensätze) mit zwei registersichtbaren Ausnahmen: 32102 ist hier ein 2-Register-`float32`, und 47001 ist nicht implementiert, statt CAN 0x18 zu senden (das der A-Micro ohnehin ignoriert). Kein Status wechselt von FUNCTIONLESS/NO_OP zu FUNCTIONAL. | §5 |

## 1. Binär-Fingerabdruck

| | A v150 | D v150 (Referenz) |
|---|---|---|
| Größe | 387 072 B (0x5E800) | 389 120 B |
| SHA-256 | `7dbff622f394a8262f9f545fbdabff2e815aa73122ce37f5b9412ca631702560` | `4140e7a6…1447` |
| Linkadresse | **0x08004800**–0x08062FFF (18 KB Bootloader darunter) | 0x08004800–0x08063FFF |
| MCU | Nations **N32G45x** mit XFMC (Cortex-M4F) | gleich |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW) | gleich |
| Build-String | `Aug 11 2026 11:46:45` | `Aug  5 2026 11:51:21` |
| Version | Versionswort 0x96 (150) @ 0x08010000, danach `VNSA-0`; String `VNSA-0 v150` @ 0x0805B73D | 0x96, `VNSD-0` |
| Image-Ende | `VNSA` + 12 Nullbytes @ 0x08062FE0, 0xFF-Füllung ab 0x08062C4C | `VNSD` |
| Release-Notiz | gleicher Text für A und D (Local API im Ethernet-Modus, HTTP-Update im Ethernet-Modus, Peak-Shaving, lange HTTP-Datensätze, CT_TYPE-Zähleranbindung) | gleich |

Basisnachweis: Vektortabelle bei 0x08004800 mit initialem SP 0x2001F7E8 und Reset 0x08004A71 (Keil-Muster `06 48 80 47 06 48 00 47`);
178 Thumb-Funktionszeiger zeigen bei dieser Basis auf Prologe (4 bei 0x08000000). `SystemInit` @ 0x080327A0.

## 2. Speicherkarte (Ghidra)

| Block | Bereich | Inhalt |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | nicht im Image |
| flash | 0x08004800–0x08062FFF | Anwendung |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-Bit-UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EFF | RW-Startwerte (LZ77 @ 0x08061DB4, 2492 B) |
| .bss | 0x20002F00–0x2001F7E7 | nullinitialisiert; `ucHeap` @ 0x20003144 (**70 KB**, v150.9/v151: 80 KB) |
| sram_unused | 0x2001F7E8–0x20023FFF | bis zur 144-KB-Grenze des N32G45x |
| .xmc_bss | 0x60000000–0x60026ADF | externes SRAM über XFMC (LZ77-Image @ 0x08062770, 1244 B) |
| Peripherie | apb1/apb2/ahb/Bit-Band/ppb | Registernamen aus der Nations-N32G457-SVD (von A v150.9 übernommen) |

Scatter-Tabelle `Region$$Table$$Base` @ 0x08061894: .data 0x08061DB4 → 0x20000000 (0x2F00, `__decompress` @ 0x08005FBE),
XMC 0x08062770 → 0x60000000 (0x26AE0), .bss-Nullinit 0x1C8E8 @ 0x20002F00. Größen und Adressen wie D v150; A v150.9/v151 haben ein größeres
.bss (0x1F110) und XMC (0x26AF8), dort liegt daher jede RAM-Variable ab etwa 0x20000140 an einer anderen Adresse.

## 3. Verhältnis zu Venus D v150 und A v150.9 (Methode und Zahlen)

| Vergleich (namensgepaart, vollständig) | Gleich | Geändert | Nur links | Nur rechts |
|---|---|---|---|---|
| D v150 → A v150 | 1899 | 31 | 14 wegkompiliert (§4.4) + 5 Code-Folding-Stubs + 5 umbenannt | 5 umbenannt (`OTA_Is_VNSA_Model`, `USART1_IRQHandler`, `Inverter_RegDirty_Mark_GridPermit`/`_PvVoltage`, `Util_NoOp_StartupHook`) |
| A v150 → A v150.9 | 1893 | 42 | 0 | 6 |
| A v150 → A v151 | 1887 | 48 | 0 | 7 |

„Geändert“ = andere Instruktionsfolge, Aufrufziele über Namen verglichen (Literal-Pool- und RAM-Adressen maskiert). Von den 42 Änderungen
zu v150.9 sind 24 Verhaltensänderungen, 3 tragen nur Version/Identität und 15 unterscheiden sich nur in verschobenen RAM-Offsets oder
Log-Zeilennummern. D v150 hat fünf 2-Byte-Stubs aus „Identical Code Folding“ (gemeinsame Fault-Handler-Schleifen, Rumpf von
`__scatterload_copy`), die A v150.9/v151 als Teil des Handlers modellieren; A v150 folgt dem A-v150.9-Modell, sie zählen daher nicht als Unterschied.

Methode: Offline-Segmentkarten mit Maskierung gegen D v150 (98,2 %), A v150.9 (92,5 %), A v151 (91,2 %) und D v151 (90,5 %); Layout, Namen,
Prototypen, Globals und Referenzen von D v150 (gleiches Layout) übernommen, danach Namen/Prototypen/Plates aller bytegleichen Funktionen von
A v150.9 für die versionsübergreifende Namenskonsistenz, RAM-Namen von A v150.9 über eine namensverankerte stückweise Adresskarte übertragen;
jede geänderte Funktion von Hand geprüft (§4) und für v150 kommentiert.

## 4. Funktionale Unterschiede

### 4.1 Leistungsgrenzen und Leistungsklassen (A v150 ggü. D v150; in A v150.9 gleich)

| Stelle | A v150 | D v150 |
|---|---|---|
| `Config_Write_U16_0x202` (max. Ladeleistung, EEPROM 0x202) | Grenze 1500 W | 2500 W |
| `Config_Write_U16` (max. Entladeleistung, EEPROM 0x204) | Grenze 1500 W | 2500 W |
| `Config_ClampDefaults_ParamBlock` | Laden 0/>1500 → 1500 W; Entladen 0/>1500 → 800 W | 2500 W |
| `Config_LoadOrInit_DeviceSettings` Erststart-Vorgabe max. Ladeleistung | 1500 W | 2500 W |
| `BLE_App_Command_Dispatcher` Kmd 0x15 Leistungsklasse | 800 → 1, 1200 → 4, 1500 → 5 | 800 → 1, 1500/2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–1500 W / ≤ 1500 W | 300–2500 W / ≤ 2500 W |
| `MQTT_Config_Command_Handler` Vorgabe `md=` | 1500 W | 2500 W |
| `MQTT_JSON_RPC_Dispatcher` Passiv-Sollwert, Leistungsklasse | ±1500 W, A-Klassen | ±2500 W |
| `Remote_Power_Command_Execute` | ±1500 W | ±2500 W |
| `Write_Handler` 44003 | über 1500 W abgewiesen | über 2500 W |
| `Grid_Export_Power_Limiter` | Startwert ≤ 2500 W, obere Klemme 2400 W; Wartungsmodus: Old-Step 1 oder 5, Leistungsbefehl Modus 3 | obere Klemme 4000 W; Step 5, Modus 1 |

### 4.2 Akku

`BMS_CellVoltage_MinMax_PerPack` prüft **13 Zellen** für die Packs 1..6 (D: 16). `Battery_EPS_Reserve_SOC_Hysteresis` (gleicher Code wie
D v150) ist ein Automat mit drei Zuständen: EPS-Reserve aktiv bei SOC ≤ Entladegrenze, frei ab Grenze + 5 %; unter 9 % SOC wird die
EPS-Funktion abgeschaltet und gespeichert (EEPROM 0x300 = 0) und **automatisch wieder eingeschaltet** (EEPROM 0x300 = 1), sobald der SOC
über 8 % steigt; der Zustand geht als CAN-Frame 0xCF hinaus. BLE-Kommando `T` und MQTT `cd=0x38` setzen den Automaten zurück. v150.9 entfernt
das automatische Wiedereinschalten und die Rücksetzungen.

### 4.3 RS485 / serielle Schnittstelle

USART1 (PA9/PA10, IRQ 37, 115200 8N1) ohne Driver-Enable-Pin — `RS485_UART_Config_Init` (A-Variante), `RS485_UART_Send`,
`RS485_UART_SendByte`, `Protocol_PutChar`, `USART1_IRQHandler`. D v150 nutzt USART2 an PC8/PC9 plus `RS485_DE_Pin_Init`/`RS485_Set_TX_Mode`/
`RS485_Set_RX_Mode`. Gleich wie A v150.9/v151.

### 4.4 Funktionen von D v150, die im A v150 fehlen (wegkompiliert)

`Key_ProtectionMonitor`, `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`,
`Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot`,
`CAN_ParallelInvLink_DebounceHold`, `Board_GPIO_Init_ControlPins`, `RS485_DE_Pin_Init`, `RS485_Set_TX_Mode`, `RS485_Set_RX_Mode` (14).
`WDT_SystemHealthService` führt daher nur den 500-ms-Board-Tick aus; `Inverter_Default_Init_Once` sendet nur `Inverter_Set_Flag_Reg_0x05(0)`.
Gleiche Menge wie in A v150.9 (dem zusätzlich das nur in D v151 vorhandene `MQTT_CtrlResponse_Transmit` fehlt).

### 4.5 Cloud / HTTP / Netzwerk

| Funktion | A v150 | Bemerkung |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://<g_cloudDomainUrlTable[t]>.hamedata.com/app/neng/getDateInfo<g_cloudGetDateInfoSuffixTable[t]>.php`, Servertypen 0–2, `sv=150` | wie D v150 / A v150.9 |
| `Cloud_UploadTelemetryData` | JSON-POST an `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`, Nutzlast `{"d":"di=…&sn=…"}` | gleicher Code wie D v150; in v150.9 neu geschrieben |
| `HTTP_ClientTask_Dispatcher` | Verbindungs-Watchdog: mehr als ein nicht gesendeter Telemetrie-Datensatz, Old-Step 0 und 0x708 (1800) abgelaufen → Log `[HTTP]ch395 reset!!!!` und **`System_Reboot`** | **D v150: `CH395_Reset_And_Reinit(0)`**; v150.9: ≥ 5 Datensätze und 3600, v151: 14 400 + Unterdrückungs-Flag |
| `HTTP_Task_StateMachine_Handler` | `AT+QHTTPPOST=%d,60,60` | wie D v150 / A v150.9 |
| `CH395_Init_Modbus_TCP_Socket`, `CH395_ConfigTcpKeepalive_Defaults` | kein Keepalive auf Socket 502; Keepalive-Leerlauf 10 s für die übrigen Sockets | Keepalive auf 502 ab v150.9, 60 s in v151 |
| `Modem_/Ethernet_SendStatusTelemetry` | Zählerabfrage bei jedem Aufruf (keine Ratenbegrenzung) | v150.9: alle 2000 Ticks, v151: 1100 |
| `Meter_ParseUDPMeterInfo` | nach geparster 3-Phasen-Antwort `vTaskDelay(ct_type × 1200)` | ab v150.9 Zähler statt Verzögerung |
| `Cloud_BuildDeviceSourceID` | `VenusA-<serial>` | ab v150.9 `VNSA-0-<serial>` |
| `Config_LoadOrInit_DeviceSettings` | Typcode 0x96, beim Start kein Laden von SEL-LED (0x395) oder Servertyp (0x441) | SEL-LED wird ab v150.9 geladen |

### 4.6 Identität und weitere Details

- Strings: `VNSA-0`, `VNSA-0 v150`, `MST_VNSA_`, `Venus A`, `VenusA-%s`, Typcode 0x96 (`g_deviceTypeCode`).
- `OTA_Is_VNSA_Model` und `OTA_Slot_Config_Validate` nehmen nur `VNSA-0`-Images an. YMODEM (`YModem_ReceiveFile`, `YModem_FirmwareUpgrade_Handler`)
  prüft nur Größe und CRC, keine Bin-Typ-Prüfung des Dateinamens.
- `CAN_Broadcast_DateTime`: zweiter 0xCC-Frame mit DLC 8 (D: 7).
- `Protocol_AA_SetSystemParams` übernimmt das empfangene Arbeitsmodus-Paar ohne Bereichsprüfung (v150.9 ergänzt eine).
- `MQTT_JSON_RPC_Dispatcher` hält die SEL-LED-Einstellung nur im RAM (v150.9 schreibt zusätzlich EEPROM 0x395).
- `MQTT_CtrlCommand_Dispatcher`: höchstes `cd` ist 0x41; `cd=0x42` (Wechselrichter-Debuginfo) und der `cd=19`-Verbindungsbericht kommen mit v150.9.

## 5. Modbus

- **Lesebereich:** `g_modbusRegDescTable` @ 0x20000354, 246 × 12 B, bytegleich mit D v150. Gegenüber A v150.9/v151 weicht nur Eintrag 32102
  ab (`float32`, 2 Register, statt `i16`); ansonsten gilt die dekodierte Tabelle von v150.9. `FC03_Read_Handler` @ 0x08023214 und
  `Read_Serializer` @ 0x08054DB8 gleich D v150 und A v150.9 (bekannte D-Fehler: Vorzeichen 32101, MBAP-Länge der Exception-Frames).
- **Schreibbereich:** `Write_Handler` @ 0x08055EB8, gleicher Code wie D v150 bis auf die 44003-Grenze:

| Register | A v150 |
|---|---|
| 44002 max. Ladeleistung | angenommen bis 2500 W (darüber Exception 3); Rohwert an den Wechselrichter (`Inverter_Write_Reg_0x04_U32Value`) und in den RAM-Cache, den 44002 zurückliest; EEPROM-Kopie (0x202) von `Config_Write_U16_0x202` auf 1500 W begrenzt |
| 44003 max. Entladeleistung | **über 1500 W abgewiesen (Exception 3)**; ≤ 1500 W an den Wechselrichter (`Inverter_Write_Reg_0x03_U32Value`), RAM-Cache und EEPROM 0x204 |
| 45030 (0xAFE6) | über den Bereichspfad des 45000-Blocks behandelt (`Config_Apply_SingleReg` 11/12), gleiche Wirkung wie der eigene Fall in v150.9 |
| 47001 (0xB799) | **nicht implementiert** (kein Fall); v150.9/v151 senden CAN 0x18 (`CAN_Write_Category0x18`) |

  Der 45000-Kommandoblock, 46000 = 0x5100 (Debugmodus → Shell auf RS485 und TCP 8091) und 46500–46544 (Zeitplan, CAN 0x30–0x3E) verhalten sich wie beim D.
  Drei Switch-Tabellen von `Write_Handler` konnte Ghidra nicht automatisch auflösen (auch bei D v150); sie sind als Sprungtabellen-Overrides im Decompiler gesetzt.

## 6. Interner CAN-Bus und Packs

`CAN_Build_Arbitration_ID` @ 0x080370D0, `CAN_FrameDispatcher` @ 0x08033420, `Telemetry_Register_Dispatcher` @ 0x08033F48,
`Modbus_ResponseDispatch` @ 0x080337DC, `Protocol_AA_CommandDispatch` @ 0x08033D34 — alle Sendestellen gleich D v150 (nur die DLC des
0xCC-Datumsframes weicht ab). Klasse-2/4-Kommandos wie in der v150.9-Analyse (§6 dort) aufgeführt, ohne 0x18 (kommt mit 47001 in v150.9).
Pack-Kapazität: 7 Datensätze je Pack im Control, 13-Zellen-Prüfung für Packs 1..6; das A-BMS 1105 sendet Packs 1..6 (`pack_id != 0 && pack_id < 7`).

## 7. Tasks, Shell, Protokolle

Task-Tabelle `g_taskDefTable` @ 0x0805A108 (14 Tasks, wie D v150), `shellCommandTable` @ 0x080618C4 (79 Einträge, Benutzer `VNSA-0 v150`
mit leerem Passwort; `read_inv1`/`read_inv2` kommen mit v150.9), MQTT-RPC-Tabelle `g_mqttRpcMethodTable` @ 0x0805D2E8 (16 Methoden),
mbedTLS-Cipher-Tabelle @ 0x0805D440, CH395-Sockets (502 Modbus TCP, 8091 Shell, 8090 UDP).

## 8. EEPROM-Belegung

Gleiche Adressen wie D v150. Beim Erststart geschriebene Grenzwerte: max. Ladeleistung 1500 W. SEL-LED (0x395) und Cloud-Servertyp (0x441)
werden beim Start nicht geladen.

## 9. Auffällige Strings

`VNSA-0 v150`, `MST_VNSA_%c%c%c%c`, `Venus A`, `VenusA-%s`, `Build:\tAug 11 2026 11:46:45`, hamedata-`getDateInfo`-URL (zwei Varianten),
`AT+QHTTPPOST=%d,60,60`, `[HTTP]ch395 reset!!!!`, marstekcloud-Data-Upload-URL. Hersteller-Standard-WLAN-Zugangsdaten (`SM471`, …) und
`quit_debug_5100_` wie beim D.

## 10. Sicherheitshinweise

Alle Control-Befunde von D v150 gelten (gleicher Code): Modbus 46000 = 0x5100 → Shell ohne Authentifizierung auf RS485 und TCP 8091,
Shell-Benutzer mit leerem Passwort, Hersteller-Standard-WLAN-Zugangsdaten, `getDateInfo` im Klartext-HTTP an hamedata.com.

## 11. Bearbeitungsstand in Ghidra

- Abdeckung: 0 undefinierte Bytes, 0 verwaiste Instruktionen; 1935 Funktionen, alle benannt, alle mit Prototyp.
- `marstek_gates.py`: **alle Gates bestanden** (kein Code-als-Daten, keine Literal-Funktionen, keine FUN_/tmp_/Duplikate, 0 unbenannte
  referenzierte RAM-Variablen, keine DEFAULT-Prototypen, Tasknamen passen zur Task-Tabelle, No-Return-Flags plausibel); Dekompilat-Export
  0 Fehler, keine unaufgelösten Sprungtabellen; kurze Strings geprüft (alle echte NUL-terminierte Texte).
- Tags: `NAME_VERIFIED` an allen 1935 Funktionen, davon 98 zusätzlich `NAME_LOW_CONFIDENCE` (gleiche Menge wie A v150.9).
- Namen, Prototypen und RAM-Variablennamen sind mit A v150.9/v151 konsistent (gleiches Objekt = gleicher Name); später umbenannte D-v150-Namen
  (z. B. `Inverter_RegDirty_Mark_GridPermit`/`_PvVoltage`, `OTA_Is_VNSA_Model`) tragen die aktuellen Namen.

## 12. Offene Fragen

1. Welche CAN-Frames der A-Micro verarbeitet, beantwortet die A-Micro-1211-Analyse; ob ein A-Micro aus der v150-Zeit sich anders verhielt, ist unbekannt (kein älteres A-Micro-Image analysiert).
2. Die Release-Notiz von A v150 ist identisch mit D v150; die D-v150-Analyse ordnet ihre fünf Punkte Codeänderungen von D v149.2 → v150 zu. Eine A-v149-Basis wurde nicht analysiert, die Zuordnung gilt daher als für A übertragbar angenommen.
3. 33 RAM-Namen von A v150.9 haben kein Gegenstück in A v150 (in v150.9 eingeführte Objekte, z. B. `g_vns_inv_debug`); keine offene Lücke, nur der Vollständigkeit halber aufgeführt.

## Anhang: Methode

Offline-Segmentkarten (maskierte Pointer und BL-Offsets), `vna150_layout_sync.py` / `vna150_port_segments.py` / `vna150_port_signatures.py` /
`vna150_copy_by_address.py` / `vna150_port_refs.py` von D v150, danach Übernahme von Namen/Prototypen/Plates aus A v150.9 für bytegleiche
Funktionen, namensverankerte RAM-Karte von A v150.9, Prüfung aller geänderten Funktionen von Hand und die Prüf-Gates. Die Hilfsskripte liegen
nur lokal (nicht im Repo).
