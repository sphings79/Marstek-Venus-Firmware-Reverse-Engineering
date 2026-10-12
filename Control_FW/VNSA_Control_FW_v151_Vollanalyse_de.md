# Control-FW v151 (Venus A) — Vollanalyse und Unterschiede zu A v150.9 und Venus D v151

> 🇬🇧 English version: [VNSA_Control_FW_v151_Full_Analysis.md](VNSA_Control_FW_v151_Full_Analysis.md)

**Stand:** abgeschlossen 2026-10-11. Alle Aussagen wurden am Code dieses Images geprüft; offene Punkte stehen in §13.
**Binary:** `151_control_VNSA-0_ems_app_0151_0928_095826.bin` (Archiv `Marstek FW Archiv/firmwares/VNSA-0/Control/151/`)
**Vorgänger:** Venus A Control v150.9 (Beta), siehe [VNSA_Control_FW_v1509_Vollanalyse_de.md](VNSA_Control_FW_v1509_Vollanalyse_de.md)
**Referenz:** Venus D Control v151, siehe [VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md](VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md)
**Diff-Tabellen:** [VNSA_Control_Diff_v1509_v151_de.md](VNSA_Control_Diff_v1509_v151_de.md), [VNSA_Control_Diff_v150_v151_de.md](VNSA_Control_Diff_v150_v151_de.md)
**Ghidra:** `/VNSA/Control/Control_151_VNSA-0_ems_app_0151_0928_095826.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** `Dekompilate/VNSA_Control_v151/` (nur lokal, nicht im Repo): 1942 Funktionen, 0 Dekompilierfehler, + `listing/` (Flash-Listing, `globals_and_registers.txt`)
**Modbus-Deskriptortabelle:** bytegleich mit v150.9, siehe [../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv](../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv)

---

## 0. Zusammenfassung

- **A v151 = A v150.9 + der Cloud-/HTTP-Teil der Venus D v151.** 97,6 % des Images sind mit A v150.9 identisch, sobald Pointer und
  BL-Offsets maskiert werden. Von den 1942 Funktionen sind 1924 instruktionsgleich mit v150.9, 17 geändert, 1 neu
  (`MQTT_CtrlResponse_Transmit`), keine entfallen. 15 der 17 geänderten Funktionen sind jetzt instruktionsgleich mit der
  entsprechenden Funktion der Venus D v151; die beiden anderen (`Config_LoadOrInit_DeviceSettings`, `Quectel_AT_SendAndPollResponse`)
  unterscheiden sich vom D nur durch die A-Konstanten bzw. einen Aufruf-Offset.
- **Was sich gegenüber v150.9 geändert hat** (alle Punkte passen zur Release-Note): `getDateInfo` geht jetzt an `http://api-<region>.marstekcloud.com`
  (nicht mehr `hamedata.com`), Cloud-Servertypen 0–4 mit den korrigierten Vorab-Präfixen `dev`/`staging`, Servertyp wird beim Booten geladen,
  HTTP-POST über das Mobilfunk-/WLAN-Modul neu geschrieben (echte Antwortschleife, Timeouts 120/120 s), TOU-Antwort mit zweiter Prüfsumme,
  KI-Fahrplanabfrage seitenweise beantwortet, CH395-Keepalive-Idle 10 s → 60 s, CT-/Zählerabfrage höchstens alle 1,1 s,
  Verbindungs-Watchdog in `HTTP_ClientTask_Dispatcher` umgebaut (3600 → 14 400 Ticks, dann `System_Reboot`, Unterdrückungs-Flag).
- **Unverändert gegenüber v150.9:** Modbus-Registerkarte (Deskriptortabelle und `Write_Handler`), BLE-Kommando-Dispatcher, alle CAN-Sender
  und -Empfänger, Leistungsgrenzen (1500 W), Leistungsklassen (800/1200/1500 W), 13 Zellen pro Pack, 7 Pack-Datensätze, OTA-/YMODEM-Prüfungen,
  RS485 auf USART1, RAM-Layout.
- **Die Venus-A-Besonderheiten gegenüber D v151 sind dieselben wie in v150.9** (§5): 1500-W-Grenzen, A-Leistungsklassen, 13 Zellen, USART1 ohne
  Driver-Enable-Pin, kein Tasten-/Schutzmonitor, Kennung `VNSA`. Neu dokumentiert: Modbus 44003 (max. Entladeleistung) wird vom `Write_Handler`
  oberhalb von 1500 W abgewiesen (D: 2500 W) — diese Konstante gab es schon in v150.9, obwohl die v150.9-Analyse den `Write_Handler` als
  identisch zum D bezeichnet.

## 0a. Checkliste (Pflichtprüfungen, A v151 gegenüber A v150.9)

| # | Frage | Antwort A v151 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register anders? | **Nein.** Deskriptortabelle `g_modbusRegDescTable` @ 0x2000034C (246 Einträge, 30000–38014) bytegleich mit v150.9; `Write_Handler` @ 0x08056A7C und `FC03_Read_Handler`/`Read_Serializer` instruktionsgleich. Register 30200 liest jetzt das Versionswort 0x97 = **151**. | §6 |
| 2 | BLE anders? | **Nein.** `BLE_App_Command_Dispatcher` @ 0x0800C5C8 instruktionsgleich (Name `MST_VNSA_…`, Kommando 0x15 Klassen 800/1200/1500 W, Kommando 0x16/0x17 ≤ 1500 W). | §5 |
| 3 | AC-Grenzen / Einspeiselimit? | **Unverändert: 1500 W** (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock`, MQTT, JSON-RPC ±1500 W, `Remote_Power_Command_Execute` ±1500 W, Modbus 44003 > 1500 W abgewiesen); obere Grenze des Netz-Exportbegrenzers 2400 W. | §5 |
| 4 | Firmware-Update per BLE möglich? | **Ja**, OTA-/YMODEM-Code identisch zu v150.9 (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate` akzeptieren nur `VNSA-0`; YMODEM-Bin-Typprüfung `OTA_ClassifyDeviceTypeTag` schon in v150.9 vorhanden). | §4.4 |
| 5 | Cloud-Domains? | `getDateInfo` **`http://api-<region>.marstekcloud.com/app/neng/getDateInfo<suffix>.php`** (v150.9: `http://<region>.hamedata.com/…`), unverschlüsseltes HTTP. Upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>` unverändert. Weiterhin hamedata.com: backup-configs, schedulings/policies, getDevicesApply, FC41D-OTA. | §4.1 |
| 6 | Pack-Kapazität? | Control unverändert: 7 Pack-Datensätze (Modbus 34000–34699), `BMS_CellVoltage_MinMax_PerPack` durchsucht 6 Packs × 13 Zellen. Das A-BMS 1105 sendet Packs 1..6 (`pack_id < 7`), Pack 7 liest also 0. | §7 |
| 7 | Diff-Tabelle | [VNSA_Control_Diff_v1509_v151_de.md](VNSA_Control_Diff_v1509_v151_de.md) (namensgepaart, vollständig), [VNSA_Control_Diff_v150_v151_de.md](VNSA_Control_Diff_v150_v151_de.md) (namensgepaart, vollständig; ersetzt den früheren abgeleiteten Diff) | — |
| 8 | Erneute Prüfung nicht funktionaler Register | Alle 216 A-Register mit Status FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL bleiben wie eingestuft: Keiner der Control-Codeteile, von denen sie abhängen, hat sich geändert (Deskriptortabelle, `Write_Handler`, CAN-Sender/-Dispatcher, Pack-Datensätze), und zu diesem Release gehört kein neues A-Micro-/BMS-Image. | §6 |

## 1. Binary-Steckbrief

| | A v151 | A v150.9 (Vorgänger) |
|---|---|---|
| Größe | 391 168 B (0x5F800) | 391 168 B |
| SHA-256 | `5cceb230f28c9c3a67be5e897afd5722f795b3e8f6c39d58d5a2c2873d994346` | `a8c58127…9f23` |
| Linkadresse | **0x08004800**–0x08063FFF (18 KB Bootloader darunter) | gleich |
| MCU | Nations **N32G45x** mit XFMC (Cortex-M4F), 144 KB SRAM | gleich |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW) | gleich |
| Build-String | `Sep 28 2026 09:54:57` | `Sep  8 2026 09:34:19` |
| Version | `VNSA-0 v151`; Versionswort 0x97 (151) @ 0x08010000, danach `VNSA-0` | `VNSA-0 v1509`, 0x5E5 |
| Image-Fußzeile | `VNSA` + 12 Nullbytes @ 0x08063FE0, 0xFF-Füllung ab 0x08063A0C | gleiche Fußzeile |
| Release-Note (Hersteller, übersetzt) | Modbus-TCP-Stabilität; DOD-Fehlerzustand verhinderte Entladen; YMODEM-Upgrade-Typprüfung; KI-Modus-Prüfung und Abfrage der aktuellen KI-Strategie; Fehler in der Vorab-Server-URL (Upload schlug fehl); Stabilität des HTTPS-Uploads; CT-Kommunikation alle 1,1 s | — |

Basisnachweis: Die Vektortabelle bei 0x08004800 löst auf (initialer SP 0x20022010, Reset 0x08004A71 = Keil-Muster
`ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`), `SystemInit` @ 0x08032D88 schreibt `SCB_VTOR = &__Vectors` (0x08004800),
und 177 Thumb-Funktionspointer treffen bei dieser Basis einen Funktionsprolog, bei 0x08000000 nur 5.

## 2. Speicherkarte (Ghidra)

| Block | Bereich | Inhalt |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | nicht im Image |
| flash | 0x08004800–0x08063FFF | Anwendung |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-Bit-UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EFF | RW-Initialwerte (LZ77 @ 0x08062B74, 2492 B) |
| .bss | 0x20002F00–0x2002200F | nullinitialisiert; `ucHeap` @ 0x20003144 (80 KB) |
| sram_unused | 0x20022010–0x20023FFF | bis zur 144-KB-Grenze des N32G45x |
| .xmc_bss | 0x60000000–0x60026AF7 | externes SRAM über XFMC (LZ77-Image @ 0x08063530, 1244 B) |
| Peripherie | apb1/apb2/ahb/Bit-Band/ppb | Registernamen aus der Nations-N32G457-SVD |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC- und QSPI-Controller |

Scatter-Tabelle `Region$$Table$$Base` @ 0x08062634: .data 0x08062B74 → 0x20000000 (0x2F00, `__decompress` @ 0x08005FBE),
XMC 0x08063530 → 0x60000000 (0x26AF8), .bss Nullinit 0x1F110 @ 0x20002F00. Die Regionsgrößen sind identisch mit v150.9 und D v151, und die
entpackten `.data`-/XMC-Images sind mit beiden bis auf Flash-Pointer identisch; jede RAM-Variable liegt also an derselben Adresse und trägt
denselben Namen wie in v150.9 und D v151.

## 3. Verhältnis zu A v150.9 und D v151 (Methode und Zahlen)

| Kategorie | Funktionen | Bedeutung |
|---|---|---|
| instruktionsgleich mit v150.9 | 1924 | Namen, Prototypen, Plate-Kommentare, lokale Variablen, Tags aus dem v150.9-Programm übernommen |
| gegenüber v150.9 geändert | 17 (+ 10, die nur verschobene Tabellen/Literale oder den neuen Versionsstring referenzieren) | 15 davon instruktionsgleich mit D v151 (Markup aus D v151 übernommen), 2 gegenüber beiden geändert (§4) |
| neu gegenüber v150.9 | 1 | `MQTT_CtrlResponse_Transmit` @ 0x08023990 (existiert in D v151; seitenweise `ctrl`-Antworten) |
| entfallen gegenüber v150.9 | 0 | — |
| abweichend von D v151 | — | genau die A-spezifischen Funktionen, die schon die v150.9-Analyse aufführt (Grenzen, Leistungsklassen, Kennungen, USART1, 13 Zellen; §5) |
| fehlend gegenüber D v151 | 15 | unverändert (§5.4) |

Methode: Offline-Segmentkarte der Images (Flash-Pointer und BL/B.W-Offsets maskiert) gegen v150.9 und D v151; Funktionslayout, Namen,
Signaturen, Datentypen, Labels, Kommentare, Referenzen und Flow-Overrides wurden für die v150.9-Segmente aus v150.9 und für den Code,
den nur D v151 teilt, aus D v151 übernommen; danach ein namensgepaarter Instruktions-Diff (Aufrufziele per Name, referenzierte Strings,
Immediate-Werte) über alle 1942 Funktionen gegen beide Referenzen, Durchsicht jeder geänderten Funktion von Hand, Typisierung der Lücken und
die Prüf-Gates (§12).

## 4. Änderungen gegenüber A v150.9

Der gesamte geänderte Code ist der Cloud-/HTTP-Code der Venus D v151; die folgenden Beschreibungen wurden an diesem Image geprüft.

### 4.1 Cloud-Endpunkte und Servertypen (Release-Note Punkt 5)

| Funktion | v150.9 | v151 |
|---|---|---|
| `HTTP_Cloud_Reporting_Dispatcher` @ 0x08012424 (Quectel `AT+QHTTPCFG`) | `http://%s.hamedata.com/app/neng/getDateInfo%s.php?…` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php?…` |
| `HTTP_GetUrlData` @ 0x08019910 (CH395-Pfad) | gleiche hamedata-URL, Servertypen 0–2 akzeptiert | marstekcloud-URL, Servertypen 0–4 akzeptiert (andere → 0) |
| `g_cloudDomainUrlTable` @ 0x20000FC8 | eu, www, us, eu-dev, eu-staging | eu, www, us, **dev**, **staging** |
| `Config_LoadOrInit_DeviceSettings` @ 0x0802DB28 | EEPROM 0x441 (Servertyp) nur bei Bedarf gelesen; kein Erststart-Default für 0x395 | lädt 0x441 und 0x395 beim Booten; Erststart schreibt 0x395 (SEL-LED) = 1 |
| Version in URLs/Telemetrie/Geräteinfo | 0x5E5 (1509) | 0x97 (151) |

Die Suffix-Tabelle `g_cloudGetDateInfoSuffixTable` @ 0x20000FDC (eu, '', us, eu-dev, eu-staging) ist unverändert. Der Upload-Endpunkt
`https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>` und die übrigen hamedata.com-URLs sind unverändert.

### 4.2 HTTP-Upload (Release-Note Punkt 6)

| Funktion | v150.9 | v151 |
|---|---|---|
| `Quectel_HTTP_POST_SendAndVerify` @ 0x08010438 | wartet blind, keine ERROR-/`+QIURC`-Prüfung, NUL-terminiertes Senden | Abfrageschleife mit ERROR-/`+QIURC`-Erkennung, exakte Byteanzahl, Log `[Http Tx…] post data=%s` |
| `HTTP_Task_StateMachine_Handler` @ 0x08010B4C | `AT+QHTTPPOST=%d,60,60`, äußerer Timeout 600, feste Länge | `AT+QHTTPPOST=%d,120,120`, äußerer Timeout 6000, Länge aus `g_cloudUploadBodyLen` |
| `Cloud_UploadTelemetryData` @ 0x0801B300 | — | gleiche Nutzlast, Behandlung der Body-Länge wie D v151, Version 151 |
| `HTTP_Economy_TOU_ParseResponse` @ 0x08019418 | eine XOR-Prüfsumme | zusätzlich `crc2` (gleiches XOR über die Daten, akzeptiert, wenn eine von beiden passt), neuer Fehlertext mit `crc2` |
| `HTTPS_FetchUrlData` @ 0x0801BD1C | — | eine kleine Änderung beim Parameteraufbau (Code von D v151) |

### 4.3 Verbindungs-Watchdog und Modbus TCP (Release-Note Punkt 1)

| Funktion | v150.9 | v151 |
|---|---|---|
| `CH395_ConfigTcpKeepalive_Defaults` @ 0x08037A84 | Keepalive-Idle 10 000 ms | 60 000 ms |
| `HTTP_ClientTask_Dispatcher` @ 0x0801A344 | Zähler ohne Verbindung `g_httpNoLinkRebootTick`, Grenze 3600 | Grenze 14 400, danach `System_Reboot()`; solange `g_httpWatchdogSuppressFlag` (0x60025B1E) 1 ist, bleibt der Zähler auf 0 |

`CH395_Init_Modbus_TCP_Socket` war schon in v150.9 die Variante von D v151.

### 4.4 KI-Fahrplan, CT, Zähler (Release-Note Punkte 4 und 7)

| Funktion | v150.9 | v151 |
|---|---|---|
| `AI_ScheduleQuery_Response_Send` @ 0x080237B8 | Antwort `cd=58` in einem 1024-Byte-Puffer (`AI%d=…`), direkt gesendet | seitenweise: 24 Einträge je Seite, Kopf `cd=%d,pg=%d/%d,`, Zeilen `AI%u=…` mit Grenzprüfung, gesendet über das neue `MQTT_CtrlResponse_Transmit` |
| `Modem_SendStatusTelemetry` @ 0x08031538, `Ethernet_SendStatusTelemetry` @ 0x08031728 | Periode 2000 Ticks | 1100 Ticks (1,1 s; Zählerabfrage `EM.GetStatus`/`EM1.GetStatus`) |
| `CT_GridPower_Controller` @ 0x08030D20 | — | setzt zusätzlich einen positiven CT-Sollwert auf 0 zurück (Code von D v151) |
| `SMR_TIC_SelfUse_PowerController` @ 0x0800851C | — | nur die Zeilennummern im Log geändert |

Release-Note-Punkte 2 (DOD) und 3 (YMODEM-Typprüfung) waren schon in v150.9 enthalten (`Battery_EPS_Reserve_SOC_Hysteresis`,
`YModem_ReceiveFile`/`OTA_ClassifyDeviceTypeTag` sind instruktionsgleich), siehe Diff v150 → v151.

### 4.5 Kennung

`VNSA-0 v151` (`CLI_InitSession`, Shell-Benutzer), Build `Sep 28 2026 09:54:57` (`ShellCmd_get_ver`), `g_deviceTypeCode` = 0x97.
`Quectel_AT_SendAndPollResponse` unterscheidet sich von v150.9 nur durch einen Aufruf-Offset (gleiche Logik).

## 5. Verbleibende Unterschiede zur Venus D v151

Unverändert gegenüber v150.9 (die Funktionen sind instruktionsgleich mit v150.9):

### 5.1 Leistungsgrenzen und Leistungsklassen

| Stelle | A v151 | D v151 |
|---|---|---|
| `Config_Write_U16_0x202` (max. Ladeleistung, EEPROM 0x202) | Begrenzung 1500 W | 2500 W |
| `Config_Write_U16` (max. Entladeleistung, EEPROM 0x204) | Begrenzung 1500 W | 2500 W |
| `Config_ClampDefaults_ParamBlock` | Laden 0/>1500 → 1500 W; Entladen 0/>1500 → 800 W | 2500 W |
| `Config_LoadOrInit_DeviceSettings` Erststart | max. Laden 1500 W, max. Entladen 800 W | 2500 W |
| `Write_Handler` 44003 (max. Entladeleistung) | > 1500 → Exception 3 | > 2500 |
| `Write_Handler` 44002 (max. Ladeleistung) | > 2500 → Exception 3; Rohwert an Wechselrichter-Register 0x04, EEPROM-Wert auf 1500 W begrenzt | gleiche Prüfung, Begrenzung 2500 W |
| BLE-Kommando 0x15 Leistungsklasse | 800 → 1, 1200 → 4, 1500 → 5 | 800 → 1, 1500/2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–1500 W / ≤ 1500 W | 300–2500 W / ≤ 2500 W |
| MQTT `md=`-Default, `MQTT_Config_Command_Handler` | 1500 W | 2500 W |
| `MQTT_JSON_RPC_Dispatcher` Passiv-Sollwertgrenze | ±1500 W | ±2500 W |
| `Remote_Power_Command_Execute` | ±1500 W | ±2500 W |
| `Grid_Export_Power_Limiter` obere Grenze | 2400 W | 4000 W |

### 5.2 Batterie

`BMS_CellVoltage_MinMax_PerPack` @ 0x08017A58 durchsucht **13 Zellen** pro Pack (D: 16).

### 5.3 RS485 / serielle Schnittstelle

USART1 (PA9/PA10, IRQ 37, 115200 8N1) ohne Driver-Enable-Pin: `RS485_UART_Config_Init`, `RS485_UART_Send`, `RS485_UART_SendByte`,
`Protocol_PutChar`, `USART1_IRQHandler` @ 0x0803313C. D: USART2 + `RS485_DE_Pin_Init`/`RS485_Set_TX_Mode`/`RS485_Set_RX_Mode`.

### 5.4 Im A fehlende Funktionen (wegkompiliert)

`Key_ProtectionMonitor`, `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`,
`Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot`,
`CAN_ParallelInvLink_DebounceHold`, `Board_GPIO_Init_ControlPins`, `RS485_DE_Pin_Init`, `RS485_Set_TX_Mode`, `RS485_Set_RX_Mode`,
`USART2_IRQHandler`; `OTA_Is_VNSD_Model` existiert als `OTA_Is_VNSA_Model`. `MQTT_CtrlResponse_Transmit` (in v150.9 fehlend) ist jetzt vorhanden.

### 5.5 Weitere Details

`Inverter_Default_Init_Once` sendet nur `Inverter_Set_Flag_Reg_0x05(0)` (D zusätzlich `Inverter_Set_Reg_0x04(0)`);
`CAN_Broadcast_DateTime` zweiter 0xCC-Frame mit DLC 8 (D: 7); `Grid_Export_Power_Limiter` reagiert im Wartungsmodus auf Old-Step 1 oder 5 mit
Modus 3 (D: Step 5, Modus 1); `WDT_SystemHealthService` ohne Tastenmonitor; Kennungen `VNSA-0`, `MST_VNSA_`, `Venus A`,
`VNSA-0-<Seriennummer>`.

## 6. Modbus

- **Lesebereich:** `g_modbusRegDescTable` @ 0x2000034C, 246 × 12 B, bytegleich mit v150.9 und D v151; `FC03_Read_Handler` @ 0x080233A8 und
  `Read_Serializer` @ 0x0805597C instruktionsgleich (die bekannten D-Fehler gelten).
- **Schreibbereich:** `Write_Handler` @ 0x08056A7C instruktionsgleich mit v150.9; gegenüber D v151 weicht nur die 44003-Grenze ab (§5.1).
  46000 = 0x5100 (Debug-Modus → Shell auf RS485 und TCP 8091), 45000-Block, 46500–46544, 47001 verhalten sich wie beim D.
- **Erneute Prüfung nicht funktionaler Register (Pflichtprüfung 8):** Die A-Registerkarte (`VNSA_Register_Map_Final_generated.csv`) hat 216
  Register mit Status FUNCTIONLESS (135), CONDITIONAL (67), NO_OP (12) oder UNKNOWN (2). Ihre Gründe liegen in der Deskriptortabelle, im
  `Write_Handler`, in den CAN-Sendern/-Empfängern, den Pack-Datensätzen, im A-Micro 1211 und im A-BMS 1105. Nichts davon hat sich in
  Control v151 geändert, und es ist kein neues Micro-/BMS-Image beteiligt; jedes dieser Register bleibt daher wie eingestuft („weiterhin so“).

## 7. Interner CAN-Bus und Packs

Alle CAN-Sendestellen, `CAN_Build_Arbitration_ID` @ 0x08037798, `CAN_FrameDispatcher` @ 0x08033A08 und `Protocol_AA_CommandDispatch`
@ 0x0803431C sind instruktionsgleich mit v150.9; die Tabellen der v150.9-Analyse (dort §6) gelten unverändert. Welche Frames der Klassen 2/4
das A-Micro verarbeitet, steht in der A-Micro-1211-Analyse ([../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Vollanalyse_de.md](../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Vollanalyse_de.md)):
Es ignoriert Klasse-4 0x18, 0x30–0x3E, 0x40–0x43 und Klasse-2 0xFB.
Pack-Kapazität: Das Control hält 7 Pack-Datensätze (Modbus 34000–34699); das A-BMS 1105 sendet Packs 1..6 (`CAN_TX_PerPack_10Msgs`,
`pack_id != 0 && pack_id < 7`), der 7. Datensatz bleibt also 0.

## 8. Tasks, Shell, Protokolle

Task-Tabelle `g_taskDefTable` @ 0x0805ACDC (14 Tasks, unverändert), `shellCommandTable` @ 0x08062664 (81 Einträge, Benutzer `VNSA-0 v151` mit leerem
Passwort), MQTT-RPC-Tabelle `g_mqttRpcMethodTable` @ 0x0805DF00 (16 Methoden), mbedTLS-Cipher-Tabelle `mbedtls_cipher_definitions` @ 0x0805E058,
CH395-Sockets (502 Modbus TCP, 8091 Shell, 8090 UDP).

## 9. EEPROM-Karte

Gleiche Adressen wie v150.9 und D v151. Neu in v151: 0x441 (Cloud-Servertyp) und 0x395 (SEL-LED) werden beim Booten geladen; der Erststart schreibt 0x395 = 1.

## 10. Bemerkenswerte Strings

Neu: `VNSA-0 v151`, `Build:\tSep 28 2026 09:54:57`, die marstekcloud-`getDateInfo`-URL (zwei Varianten), `AT+QHTTPPOST=%d,120,120`,
`[HTTP] ERR: crc = %d, crc2 = %d, crc_data = %d, data_len = %d, d_count = %d.`, `cd=%d,pg=%d/%d,`, `AI%u=%d|%d|%d|%d|%d.`,
`] post data=%s`, `staging`, `dev`. Entfallen: die hamedata-`getDateInfo`-URLs, `AT+QHTTPPOST=%d,60,60`, der alte Prüfsummen-Fehlertext, `AI%d=…`.

## 11. Sicherheitshinweise

Unverändert: Modbus 46000 = 0x5100 → Shell ohne Authentifizierung auf RS485 und TCP 8091, Shell-Benutzer mit leerem Passwort, WLAN-Standardzugang
des Herstellers (`SM471`). `getDateInfo` nutzt weiterhin unverschlüsseltes HTTP (jetzt zu marstekcloud.com). Neues Verhalten, das man kennen sollte:
Nach 14 400 Ticks ohne Fortschritt der Cloud-Verbindung startet das Control neu (`HTTP_ClientTask_Dispatcher`), wie bei D v151.

## 12. Bearbeitungsstand in Ghidra

- Abdeckung: 0 undefinierte Bytes, 0 verwaiste Instruktionen im Flash-Block; 1942 Funktionen, alle benannt, alle mit Prototyp.
- `marstek_gates.py`: **alle Gates bestanden** (kein Code als Daten, kein Funktionspointer auf Nicht-Code, keine Funktionen nur aus Literalen,
  keine FUN_/tmp_/Duplikate, keine doppelten globalen Labels, 0 unbenannte referenzierte RAM-Variablen, keine DEFAULT-Prototypen, Tasknamen passen zur
  Task-Tabelle, No-Return-Flags plausibel); Prüfung kurzer Strings: 2 Treffer, beide echte Strings (`"%d\t"`, `"\b \b"`, wie in v150.9);
  Dekompilat-Export 0 Fehler.
- Tags: `NAME_VERIFIED` an allen 1942 Funktionen, davon 99 zusätzlich `NAME_LOW_CONFIDENCE`.
- Namenskonsistenz: Funktions- und RAM-Namen stimmen mit D v151 überein (die beiden LED-Bitmap-Wrapper tragen die korrigierten D-Namen
  `Inverter_RegDirty_Mark_GridPermit` / `Inverter_RegDirty_Mark_PvVoltage`).

## 13. Offene Fragen

1. Erledigt 2026-10-11: Die v150.9-Analyse sagte, der `Write_Handler` sei identisch mit D v151; eine Konstante weicht ab (Grenze 44003 1500 W statt 2500 W)
   in v150.9 und v151. Die v150.9-Dokumente, das v150.9-Ghidra-Programm (LED-Wrapper-Umbenennungen `Inverter_RegDirty_Mark_GridPermit`/`_PvVoltage`,
   Plate von `Write_Handler`) und sein Export wurden korrigiert.
2. 44002 nimmt bis 2500 W an und gibt den Rohwert an den Wechselrichter weiter, während der gespeicherte Wert auf 1500 W begrenzt wird; das A-Micro
   begrenzt ohnehin auf 1450 W (A-Micro-Analyse). Ob der zurückgelesene Registerwert dann vom wirksamen Limit abweicht, wurde nicht gemessen.
3. Der Diff A v150 → v151 ist inzwischen namensgepaart ([VNSA_Control_Diff_v150_v151_de.md](VNSA_Control_Diff_v150_v151_de.md); A v150 ist in Ghidra analysiert, siehe [VNSA_Control_FW_v150_Vollanalyse_de.md](VNSA_Control_FW_v150_Vollanalyse_de.md)). Dieser Punkt ist erledigt.

## Anhang: Methode

`segmatch.py` (Offline-Segmentkarte mit maskierten Pointern/Sprüngen), `vna151_layout_sync.py` (v150.9, danach D v151 mit geschütztem Layout),
`vna151_port_segments.py`, `vna151_copy_by_address.py`, `vna151_port_signatures.py`, `vna151_port_refs.py`, `vna151_port_flow_overrides.py`,
Lückenreparatur, `vna151_token_dump.py` + namensgepaarter Diff, `marstek_gates.py`, Dekompilat-Export in Fenstern. Hilfsskripte liegen nur lokal (nicht im Repo).
