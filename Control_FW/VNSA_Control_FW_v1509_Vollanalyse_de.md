# Control-FW v150.9 (1509, Venus A) — Vollanalyse und Unterschiede zur Venus D v151

> 🇬🇧 English version: [VNSA_Control_FW_v1509_Full_Analysis.md](VNSA_Control_FW_v1509_Full_Analysis.md)

**Stand:** abgeschlossen 2026-10-02. Alle Aussagen wurden am Code dieses Images geprüft; offene Punkte stehen in §12.
**Binary:** `1509_control_VNSA-0_ems_app_1509_0908_093804.bin` (Archiv `Marstek FW Archiv/firmwares/VNSA-0/Control/1509/`, Beta „v150.9")
**Referenz:** Venus D Control v151 (`151_control_VNSD-0_ems_app_0151_0929_094333.bin`), siehe [VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md](VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md)
**Ghidra:** `/VNSA/Control/Control_1509_VNSA-0_ems_app_1509_0908_093804.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** `Dekompilate/VNSA_Control_v1509/` (nur lokal, nicht im Repo): 1941 Funktionen, 0 Dekompilierfehler, + `listing/` (Flash-Listing, `globals_and_registers.txt`)
**Modbus-Deskriptortabelle:** [../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv](../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv)

---

## 0. Zusammenfassung

- **Gleiche Codebasis wie die Venus D.** 95,2 % des Images sind bytegleich mit D v151, sobald Pointer und BL-Offsets maskiert
  werden (282 Segmente). Von den 1956 D-Funktionen sind 1898 im A instruktionsgleich, 35 weichen im Detail ab, 8 liegen mit
  gleicher Logik an anderer Adresse und **15 fehlen** (wegkompiliert). Das A hat **keine eigene Funktion**, die das D nicht hat.
- **RAM-Layout identisch zu D v151.** `.data` (12 032 B), `.bss` und das externe SRAM (`.xmc_bss`) haben dieselben Größen; alle
  4526 RAM/XMC-Pointer im gematchten Code sind identisch; das initialisierte `.data` unterscheidet sich nur in Flash-Pointern.
  Jede RAM-Variable liegt also an derselben Adresse wie beim D (Namen 1:1 übernommen).
- **Modbus-Registerkarte identisch zu D v151.** Die Deskriptortabelle (`g_modbusRegDescTable` @ 0x2000034C, 246 Einträge,
  30000–38014) ist bytegleich, der Lesepfad ist instruktionsgleich und `Write_Handler` (Holding-Register) hat denselben Registersatz
  und Code mit einer A-spezifischen Konstante (44003 nimmt höchstens 1500 W an, siehe §5). Ansonsten ist nur anders,
  was die RAM-Variablen füllt (der Wechselrichter des A ist ein anderes Gerät, siehe §5).
- **Die Geräteunterschiede sind Compile-Zeit-Konstanten**, keine eigenen Codepfade: Leistungsgrenzen (1500 W statt 2500 W),
  Leistungsklassen (800/1200/1500 W), 13 Zellen pro Akkupack, RS485-Port auf USART1 ohne Driver-Enable-Pin, kein
  Tasten-/Schutzmonitor, Kennungen `VNSA`. Der CAN-Code (Frame-Aufbau, Dispatcher, alle Wechselrichter-Kommandos inkl.
  0xFB/0xFE/0xFF, 0x18, 0x30–0x3E) ist identisch zum D; es gibt im Control keine A-spezifische CAN-Funktion.
- **Ältere Cloud-Generation als D v151:** `getDateInfo` geht noch an `http://<region>.hamedata.com/...` (wie D v150), nur 3
  Cloud-Servertypen werden akzeptiert, CH395-Keepalive-Idle 10 s, ältere Varianten von HTTP-POST und TOU-Parser.

## 0a. Checkliste (A 1509 gegenüber D 151)

| # | Frage | Antwort A 1509 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register anders? | **Nein.** Deskriptortabelle bytegleich (246 Einträge inkl. 32102 `i16`); Gleicher Registersatz. `Write_Handler` @ 0x08056960 gleicher Code wie D 151 (inkl. 46000 = 0x5100 Debug-Hintertür, 47001, 45000-Block) bis auf eine Konstante: ein Schreibzugriff auf 44003 über **1500 W** wird abgewiesen (D: 2500 W); 44002 nimmt weiterhin bis 2500 W an (siehe §5). Register 30200 liest das Versionswort @ 0x08010000 = **1509**. | §5 |
| 2 | BLE anders? | Gerätename `MST_VNSA_` + 4 ID-Bytes. Cmd 0x15 (Leistungsklasse): 800 W → Klasse 1, **1200 W → Klasse 4, 1500 W → Klasse 5** (D: 800 → 1, 1500/2200 → 3, 2500 → 0). Cmd 0x16 max. Ladeleistung 300–**1500** W, Cmd 0x17 max. Entladeleistung ≤ **1500** W. | §4.1 |
| 3 | AC-Grenzen? | Max. Lade-/Entladeleistung überall auf **1500 W** begrenzt (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock`, MQTT/JSON-RPC, Fernkommando ±1500 W). Obere Grenze des Einspeisebegrenzers **2400 W** (D: 4000 W). | §4.1 |
| 4 | Firmware-Update per BLE möglich? | Ja, gleicher OTA-Code wie D 151; die Modellprüfungen vergleichen gegen `VNSA-0` (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`). | §4.6 |
| 5 | Cloud-Domains? | Upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>` (wie D). `getDateInfo` **`http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php`** (D 151: `api-<region>.marstekcloud.com`). Weiterhin hamedata: backup-configs, schedulings/policies, getDevicesApply, FC41D-OTA. | §4.5 |

## 1. Binary-Fingerabdruck

| | A 1509 | D 151 (Referenz) |
|---|---|---|
| Größe | 391 168 B (0x5F800) | 393 216 B |
| SHA-256 | `a8c58127189e19937cf179195e714492f890a6a829b03b8e218380042f8d9f23` | — |
| Linkadresse | **0x08004800**–0x08063FFF (18 KB Bootloader darunter) | 0x08004800–0x080647FF |
| MCU | Nations **N32G45x** mit XFMC (Cortex-M4F), 144 KB SRAM | gleich |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW) | gleich |
| Build-String | `Sep  8 2026 09:34:19` | `Sep 29 2026 09:42:27` |
| Version | String `VNSA-0 v1509` @ 0x0805C239; Versionswort 0x5E5 (1509) @ 0x08010000, danach `VNSA-0` | 0x97 (151), `VNSD-0` |
| Image-Footer | `VNSA` + 12 Nullbytes @ 0x08063FE0, davor 0xFF-Füllung ab 0x080638C8 | `VNSD` |
| Release Note | „PV-Ausfall bei Überschusseinspeisung behoben, LED-Blinken behoben, CT-Verbindung/Phasenerkennung verbessert" | — |

Basisnachweis: Die Vektortabelle bei 0x08004800 löst genau wie beim D auf (Reset = Keil-Muster, `SystemInit` @ 0x08032C68
schreibt `SCB->VTOR = 0x08004800`); Scatter-Tabelle, String- und Funktionspointer lösen alle bei dieser Basis auf, und die
Flash-Pointer im `.data` sind die von D 151, verschoben um die Unterschiede der Codegröße.

## 2. Speicherkarte (Ghidra)

| Block | Bereich | Inhalt |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | nicht im Image; 0x08001000 Bootloader-Versionswort, 0x08004000 Update-Deskriptorseite |
| flash | 0x08004800–0x08063FFF | Anwendung |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-Bit-UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EFF | RW-Startwerte (LZ77 @ 0x08062A34, 2486 B) |
| .bss | 0x20002F00–0x2002200F | nullinitialisiert; `ucHeap` @ 0x20003144 (80 KB) |
| sram_unused | 0x20022010–0x20023FFF | bis zur 144-KB-Grenze des N32G45x |
| .xmc_bss | 0x60000000–0x60026AF7 | externes SRAM über XFMC (LZ77-Image @ 0x080633EC, 1244 B) |
| Peripherie | apb1/apb2/ahb/Bit-Band/ppb | Registernamen aus der Nations-SVD N32G457 (1:1 von D 151 übernommen) |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC- und QSPI-Controller |

Scatter-Tabelle `Region$$Table$$Base` @ 0x080624F4: .data 0x08062A34 → 0x20000000 (0x2F00, `__decompress` @ 0x08005FBE),
XMC 0x080633EC → 0x60000000 (0x26AF8), .bss Zero-Init 0x1F110 @ 0x20002F00.

## 3. Verhältnis zur Venus D v151 (Methode und Zahlen)

| Kategorie | Funktionen | Bedeutung |
|---|---|---|
| instruktionsgleich | 1898 | gleiche Bytes bis auf Aufruf-/Pointerziele; Namen, Prototypen, Plates, Lokale aus D 151 übernommen |
| geändert | 35 | gleiche Funktion, andere Konstanten oder kleine Codeänderungen (§4; inkl. `Write_Handler`, siehe §5) |
| verschoben | 8 | gleiche Logik an anderer Adresse: `Factory_Reset`, `RTC_Peripheral_Init`, `RS485_UART_Config_Init` (A-Variante), `RS485_UART_SendByte` (A-Variante), `AI_ScheduleQuery_Response_Send` (ältere Variante), `Stub_Return_One`, `MQTT_Transport_NullCallback`, `Util_NoOp_StartupHook` |
| fehlt im A | 15 | siehe §4.4 |
| nur im A | 0 | — |

Gegenprobe: Für jedes Funktionspaar ist die Folge der Aufrufziel-Namen identisch, abgesehen von den Aufrufen der fehlenden
Funktionen. Vergleichsmethode: Offline-Bytesegmente mit maskierten Pointern/BL-Offsets, danach namensnormierte
Instruktionsdiffs der geänderten Funktionen.

**Korrektur (2026-10-11):** `Write_Handler` war ursprünglich als instruktionsgleich gezählt; er unterscheidet sich von D 151 in einer
Vergleichskonstante (44003-Grenze 0x5DC statt 0x9C4, §5). Die Zahlen oben wurden von 1899/34 auf 1898/35 korrigiert.

## 4. Funktionale Unterschiede zur Venus D v151

### 4.1 Leistungsgrenzen und Leistungsklassen

| Stelle | A 1509 | D 151 |
|---|---|---|
| `Config_Write_U16_0x202` (max. Ladeleistung, EEPROM 0x202) | Grenze 1500 W | 2500 W |
| `Config_Write_U16` (max. Entladeleistung, EEPROM 0x204) | Grenze 1500 W | 2500 W |
| `Config_ClampDefaults_ParamBlock` | Laden 0/>1500 → 1500 W; Entladen 0/>1500 → 800 W | 2500 W |
| `Config_LoadOrInit_DeviceSettings` Erststart-Default max. Laden | 1500 W | 2500 W |
| BLE Cmd 0x15 Leistungsklasse | 800 → 1, 1200 → 4, 1500 → 5 | 800 → 1, 1500/2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–1500 W / ≤ 1500 W | 300–2500 W / ≤ 2500 W |
| MQTT-`md=`-Default, `MQTT_Config_Command_Handler` | 1500 W | 2500 W |
| `MQTT_JSON_RPC_Dispatcher` Passiv-Sollwert-Grenze | ±1500 W | ±2500 W |
| `Remote_Power_Command_Execute` | ±1500 W | ±2500 W |
| `Grid_Export_Power_Limiter` obere Grenze | 2400 W | 4000 W |

Modbus-Schreibzugriffe auf die Leistungsgrenzen über `Write_Handler` (korrigiert 2026-10-11, die frühere Aussage „Write_Handler
unverändert, alle Modbus-Schreibzugriffe auf 1500 W begrenzt" war falsch):

| Register | A 1509 | D 151 |
|---|---|---|
| 44002 max. Ladeleistung (`uVar == 0xABE2`) | angenommen bis **2500 W** (darüber Exception 3); der **Rohwert** geht an den Wechselrichter (`Inverter_Write_Reg_0x04_U32Value`) und in den RAM-Cache, den 44002 zurückliest; nur die EEPROM-Kopie (0x202) wird von `Config_Write_U16_0x202` auf 1500 W begrenzt | angenommen bis 2500 W, EEPROM-Grenze 2500 W |
| 44003 max. Entladeleistung (`0xABE3`) | **über 1500 W abgewiesen (Exception 3)**; ≤ 1500 W geht an den Wechselrichter (`Inverter_Write_Reg_0x03_U32Value`), RAM-Cache und EEPROM 0x204 | über 2500 W abgewiesen |

Beim A wird also z. B. ein Modbus-Schreibzugriff von 2000 W auf 44002 angenommen und unbegrenzt an den Wechselrichter weitergegeben,
bis zum nächsten Neustart (danach gilt der EEPROM-Wert 1500 W); das A-Micro begrenzt ohnehin auf 1450 W (A-Micro-Analyse).

### 4.2 Akku

`BMS_CellVoltage_MinMax_PerPack` wertet **13 Zellen** pro Pack aus (D: 16). Die Pack-Datensätze behalten das D-Layout (Platz für 16 Zellen).

### 4.3 RS485 / serielle Schnittstelle

Das A nutzt **USART1 (PA9 TX / PA10 RX, IRQ 37, 115200 8N1)** ohne Driver-Enable-Pin: `RS485_UART_Config_Init` (A-Variante),
`RS485_UART_Send`, `RS485_UART_SendByte`, `Protocol_PutChar`, `USART1_IRQHandler`. Das D nutzt USART2 auf umgemappten PC8/PC9 plus
`RS485_DE_Pin_Init`/`RS485_Set_TX_Mode`/`RS485_Set_RX_Mode`. Modbus RTU, Debug-Shell auf RS485 und YMODEM-Updates sind sonst identisch.

### 4.4 Im A fehlende Funktionen (wegkompiliert)

| Funktion in D 151 | Zweck beim D |
|---|---|
| `Key_ProtectionMonitor` | physische Taste an GPIOB.11, Entprellung und Schutz-Eskalation |
| `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`, `Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot` | Relais-/Latch-Pulskette an GPIOB.10 / GPIOE.15 mit CAN-Frame und Neustart bei Dauerfehler |
| `CAN_ParallelInvLink_DebounceHold` | Entprellung des Parallel-Wechselrichter-Link-Flags (nur vom Tastenmonitor aufgerufen) |
| `Board_GPIO_Init_ControlPins` | Ausgänge GPIOB.10/.11 und GPIOE.15 |
| `RS485_DE_Pin_Init`, `RS485_Set_TX_Mode`, `RS485_Set_RX_Mode` | RS485-Richtungspin |
| `MQTT_CtrlResponse_Transmit` | Helfer für seitenweise 'ctrl'-Antworten (das A sendet die cd=58-Antwort inline, ohne Seitenaufteilung) |

`WDT_SystemHealthService` führt daher nur den 500-ms-Board-Tick aus. Die RAM-Variablen des fehlenden Codes existieren weiterhin
(identisches `.bss`), sie werden beim A nur nicht benutzt.

### 4.5 Cloud / HTTP / Netzwerk

| Funktion | A 1509 | D 151 |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://<g_cloudDomainUrlTable[t]>.hamedata.com/app/neng/getDateInfo<g_cloudGetDateInfoSuffixTable[t]>.php`, Servertypen 0–2 | `api-<region>.marstekcloud.com`, Typen 0–4 |
| Regionstabelle @ 0x20000FC8 | `eu, www, us, eu-dev, eu-staging` + Suffixtabelle @ 0x20000FDC | `eu, www, us, dev, staging` |
| `Config_LoadOrInit_DeviceSettings` | lädt EEPROM 0x441 nicht beim Start, kein Erststart-Default für 0x395 | lädt 0x441, schreibt 0x395-Default |
| `Quectel_HTTP_POST_SendAndVerify` | ältere Variante (ohne ERROR/+QIURC-Prüfung, ohne Post-Data-Log) | neuere Variante |
| `HTTP_Task_StateMachine_Handler` | `AT+QHTTPPOST=%d,60,60`, Wiederholperiode 600 Ticks | 120/120 s, 6000 Ticks |
| `HTTP_Economy_TOU_ParseResponse` | eine CRC | zusätzlich `crc2` |
| `HTTP_ClientTask_Dispatcher` | Aktualisierung 3600 s | 14400 s |
| `CH395_ConfigTcpKeepalive_Defaults` | Idle 10 000 ms | 60 000 ms |
| `Modem_/Ethernet_SendStatusTelemetry` | alle 2000 Ticks | 1100 Ticks |
| `CT_GridPower_Controller` | EPS-Reserve aktiv → Sollwert-Update übersprungen | setzt zusätzlich einen positiven Sollwert auf 0 |

### 4.6 Kennungen und weitere Details

- Strings: `VNSA-0`, `VNSA-0 v1509`, `MST_VNSA_`, `Venus A`, Geräte-Source-ID `VNSA-0-<Seriennummer>`, Typcode 0x5E5 (`g_deviceTypeCode`).
- `OTA_Is_VNSA_Model` (D: `OTA_Is_VNSD_Model`) und `OTA_Slot_Config_Validate` akzeptieren nur `VNSA-0`-Images.
- `Inverter_Default_Init_Once` sendet nur `Inverter_Set_Flag_Reg_0x05(0)` (D zusätzlich `Inverter_Set_Reg_0x04(0)`).
- `CAN_Broadcast_DateTime`: zweiter 0xCC-Frame mit DLC 8 (D: 7).
- `Grid_Export_Power_Limiter`, Wartungsmodus: reagiert auf Old-Step 1 oder 5 und sendet `Inverter_CAN_PowerCmd_Send` mit Modus **3** (D: Step 5, Modus 1).

## 5. Modbus

- **Lesebereich:** `g_modbusRegDescTable` @ 0x2000034C, 246 × 12 B, bytegleich mit D 151 (dekodierte CSV oben verlinkt; die Spalte
  `notes` nennt die RAM-Variable im A-Programm). `FC03_Read_Handler` @ 0x080232F4 und `Read_Serializer` @ 0x08055860 sind
  instruktionsgleich, die bekannten D-Fehler (Vorzeichen 32101, MBAP-Länge der Exception-Frames) gelten also auch für das A.
- **Schreibbereich:** `Write_Handler` @ 0x08056960 hat denselben Code und dieselben Aufrufziele wie D 151 bis auf eine Konstante: 44003
  (max. Entladeleistung) wird über 1500 W abgewiesen (D: 2500 W); 44002 (max. Ladeleistung) nimmt bis 2500 W an, siehe §4.1. Der 45000-Kommandoblock,
  46000 = 0x5100 (Debugmodus → Shell auf RS485 und TCP 8091), 46500–46544 Zeitplan (CAN 0x30–0x3E), 47001 (CAN 0x18) verhalten sich wie beim D.
- **Folge für unbekannte A-Register:** Ein Register liest dieselbe RAM-Variable wie beim D. Die Variablen des Wechselrichter-/MPPT-Blocks
  (z. B. 30000–30040, 32100–32302, 35000–35112, 36000–36103) werden von den identischen CAN-Parsern aus Frames gefüllt, die der
  Wechselrichter (und der MPPT) sendet. Ihre Bedeutung beim A ergibt sich daraus, was der **A-Micro** in diesen Frames sendet; das ist aus
  der A-Micro-Firmware zu klären (eigene Analyse), nicht aus dem Control.

## 6. Interner CAN-Bus (Control-Seite)

Arbitrierungs-ID: `CAN_Build_Arbitration_ID` @ 0x08037678 (Funktionscode Bits 0–7, Knoten 8–11, Zielklasse 12–15).
Alle 57 Sendestellen sind identisch zu D 151 (nur die DLC des 0xCC-Datum-Frames unterscheidet sich):

| Zielklasse | Vom Control gesendete Funktionscodes (A 1509 = D 151) |
|---|---|
| **4** (Wechselrichter, Micro vom D/E3-Typ) | 0x01 Leistungssollwert, 0x02 `Config_Mode_Apply`, 0x03/0x04 u32-Schreibzugriffe, 0x05, 0x06, 0x07, 0x08, 0x0A Relais, 0x10, 0x11, 0x12, 0x13, 0x14, 0x16, 0x17, **0x18**, **0x30–0x3E** Zeitplan, 0x40–0x43, 0x50–0x58, 0x60, 0xC1, 0xC2, 0xCB |
| **2** (MPPT / andere Wechselrichter-Familie; Werte in `g_mppt*` gespiegelt) | 0x01 Ein/Aus (`Inverter_Set_OnOff_Reg_0x01`), 0x02 Leistungskommando (`Inverter_CAN_PowerCmd_Send`, 2 Frames), 0x03 Löschen, 0x04, 0x06 Arbeitsmodus, **0xFB, 0xFE, 0xFF**, 0xCB Löschen, 0xCC Datum/Uhrzeit |
| **3** (BMS) | 0x0A, 0xC1, 0xC2, 0xC4, 0xCF |
| 1 / variabel | 0xC2 `Debug_Mode_Set`, 0xC3, 0xCE |

Aufrufer der Klasse-2-Kommandos beim A: `Write_Handler` (45020–45024: Ein/Aus + Leistung, 0xFB, 0xFE, 0xFF), `Grid_Export_Power_Limiter`
(Leistungskommando, beim A Modus 3), `Inverter_SendOnCommand`/`Inverter_StopOutput`/`Idle_Watchdog_AutoReboot` (Ein/Aus),
`Timer_Callback_CAN_DateTimeBroadcast` (0xCC), Shell `test`/`set_old`. Empfangsseite: `CAN_FrameDispatcher` @ 0x080338E8 gibt Frames
mit Quellklasse 4 an `Telemetry_Register_Dispatcher`, Quellklasse 2 an `Modbus_ResponseDispatch`, alles andere an
`Protocol_AA_CommandDispatch`. **Welche dieser Frames der A-Micro tatsächlich auswertet, ist aus dem Control nicht entscheidbar**;
der D/E3-Micro filtert Klasse 2 und 3, der A-Micro hat einen 0xFE-Handler (früherer Befund), die vollständige Liste braucht die A-Micro-Analyse.

## 7. Tasks, Shell, Protokolle

Identisch zu D 151 bis auf Adressen: Task-Tabelle `g_taskDefTable` @ 0x0805ABC0 (14 Tasks, gleiche Prioritäten/Stacks),
`shellCommandTable` @ 0x08062524 (81 Einträge, Benutzer `VNSA-0 v1509` mit leerem Passwort), MQTT-RPC-Tabelle `g_mqttRpcMethodTable`
@ 0x0805DDE8 (16 Methoden), mbedTLS-Cipher-Tabelle @ 0x0805DF40, CH395-Sockets (502 Modbus TCP, 8091 Shell, 8090 UDP).

## 8. EEPROM-Karte

Gleiche Adressen wie D 151 (siehe D-Analyse). Unterschiede: kein Erststart-Default für 0x395 (SEL-LED; Lesen/Schreiben über
BLE/MQTT ist vorhanden), 0x441 (Cloud-Servertyp) wird nur bei Bedarf gelesen, nicht beim Start. Beim Erststart geschriebene
Default-Grenze: max. Ladeleistung 1500 W.

## 9. Interessante Strings

`VNSA-0 v1509`, `MST_VNSA_`, `Venus A`, `Build:\tSep  8 2026 09:34:19`, die hamedata-`getDateInfo`-URL (zwei Varianten),
`AT+QHTTPPOST=%d,60,60`, `[HTTP] ERR: crc = %d, crc_data = %d, data_len = %d, d_count = %d.`, UTF-8-Logstrings
`[Http Tx←★★★] %s` / `[Http Rx→☆☆☆] %s` in `Quectel_HTTP_POST_SendAndVerify`. Die WLAN-Werkszugangsdaten des Herstellers
(`SM471`, …) und `quit_debug_5100_` sind wie beim D vorhanden.

## 10. Sicherheitshinweise

Alle Control-Befunde von D 151 gelten unverändert (identischer Code): Modbus 46000 = 0x5100 → Shell ohne Authentifizierung auf RS485
und TCP 8091, Shell-Benutzer mit leerem Passwort, WLAN-Werkszugangsdaten des Herstellers, `getDateInfo` über unverschlüsseltes HTTP
(beim A an hamedata.com).

## 11. Bearbeitungsstand in Ghidra

- Abdeckung: 0 undefinierte Bytes, 0 verwaiste Instruktionen im Flash-Block; 1941 Funktionen, alle benannt, alle mit Prototyp
  (Task-Einstiege `void f(void *pvParameters)`).
- `marstek_gates.py`: **alle Gates bestanden** (kein Code-als-Daten, keine Literal-Funktionen, keine FUN_/tmp_/Duplikate, 0 unbenannte
  referenzierte RAM-Variablen, Tasknamen passen zur Task-Tabelle, No-Return-Flags plausibel); Dekompilat-Export 0 Fehler, keine
  unaufgelösten Sprungtabellen.
- Tags: `NAME_VERIFIED` an allen 1941 Funktionen (Kommentar „Name, prototype and comment verified against the VNSA Control 1509 code"),
  davon 99 zusätzlich `NAME_LOW_CONFIDENCE` (vom D übernommen, wo der Zweck nicht ganz klar ist).
- RAM-Variablen, Peripherieregister und ihre Kommentare wurden adressgleich von D 151 kopiert (identisches Layout); Kommentare mit
  D-spezifischem Inhalt (VNSD-Strings, 2500 W, USART2, EEPROM 0x441, D-Flashadressen) wurden für das A korrigiert.
- Nacharbeit 2026-10-11 (wie bei D 151 / A 151): `Inverter_RegDirty_Mark_DischgVolt` → `Inverter_RegDirty_Mark_GridPermit` und
  `Inverter_RegDirty_Mark_BatteryParams` → `Inverter_RegDirty_Mark_PvVoltage` (benannt nach der Aufrufbedingung in
  `Inverter_Apply_BatteryParams`; Muster in diesem Image geprüft). Die „Inverter-Dirty-Bitmap" ist die Frontpanel-LED-Bitmap eines
  TM1650-artigen Treibers (I2C-Digitregister 0x68/0x6A/0x6C/0x6E); Plate-Kommentare der LED-Wrapper und Globals aktualisiert, Plate von
  `Write_Handler` korrigiert (44003-Grenze). Gates erneut gelaufen: alle bestanden; Export erneuert.

## 12. Offene Fragen

1. Welche CAN-Frames sendet und verarbeitet der A-Micro (und ein möglicher MPPT-Knoten in Klasse 2)? Davon hängen die Bedeutung der
   Wechselrichter-Register beim A (30000–30040, 32100–32302, 35000–35112, 36000–36103) ab und ob 45020–45024 / 0xFB/0xFE/0xFF beim A wirken.
2. Leistungsklassen 4 (1200 W) und 5 (1500 W): wie der A-Micro sie auswertet (er erhält die Klasse über die Arbeitsmodus-/Grenzwertkommandos).
3. Das A enthält noch den älteren Cloud-/HTTP-Code (hamedata-`getDateInfo`, 3 Servertypen) — ob ein späteres A-Release den Pfad von D 151 übernimmt, ist unbekannt.

## Anhang: Methode

Offline-Segmentkarte der beiden Images (maskierte Pointer und BL-Offsets), `vna_layout_sync.py` / `vna_port_segments.py` /
`vna_port_signatures.py` / `vna_copy_by_address.py` / `vna_port_refs.py`, um Layout, Namen, Prototypen, Globals, Referenzen und
Flow-Overrides von D 151 zu übernehmen, danach manuelle Analyse der geänderten und 8 verschobenen Funktionen, Typisierung der
Lücken und die Prüf-Gates. Die Hilfsskripte liegen nur lokal (nicht im Repo).
