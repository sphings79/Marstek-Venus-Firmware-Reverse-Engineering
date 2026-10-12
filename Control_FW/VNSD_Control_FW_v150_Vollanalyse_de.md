# Control-FW v150 (Venus D) — Vollanalyse

> 🇬🇧 English version: [VNSD_Control_FW_v150_Full_Analysis.md](VNSD_Control_FW_v150_Full_Analysis.md)

**Stand:** 2026-09-30. Ersetzt `VNSD_Control_FW_Analyse_app_0150_0805_115146_de.md` für alle Adress- und
Namensangaben (jene Fassung beruhte auf der falschen Image-Basis 0x08000000; alle dortigen Flash-Adressen
sind um 0x4800 zu niedrig). Feldmessungen aus der alten Fassung bleiben gültig und werden hier als
Feldbeobachtung mit Verweis übernommen.

**Englisch:** `VNSD_Control_FW_v150_Vollanalyse_de.md` (wird nach dieser Fassung erstellt) ·
**Diff zum Vorgänger:** `VNSD_Control_Diff_v149_v150_de.md` (Haupthop, mit Zwischenschritten
`VNSD_Control_Diff_v149_v1492_de.md` und `VNSD_Control_Diff_v1492_v150_de.md`) ·
**Diff zum Nachfolger:** `VNSD_Control_Diff_v150_v151_de.md` ·
**Dekompilat/Listing:** `Dekompilate/VNSD_Control_v150_base4800/` (lokal, nicht im Repo) (1954 Funktionen) ·
**Weitere v150-Dokus:** `VNSD_Cloud_Upload_Payload_v150_de.md` (Feldmitschnitt des Cloud-Uploads),
`VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md` (BMS-Limit → Leistung, PV-Abregelung im Detail)

## 0. Kurzfassung

- Image-Basis **0x08004800** (18-KB-Bootloader darunter), Nations N32G45x (früher fälschlich als Artery AT32F403A
  angenommen, s. §1a), Keil ARMCC, FreeRTOS, mbedTLS 2.28, cJSON, llhttp, letter-shell, CH395, Quectel FC41D.
  Gleiche Codebasis wie v147–v151.
- v149 → v149.2 (Beta) trug die große inhaltliche Entwicklung (BMS-Pack-Record 88→96 B, vereinfachter
  französischer Eigenverbrauchsregler SMR/TIC, Umbau interner Telemetrie-Strukturen; 43 geänderte
  Funktionen, Register-/Verhaltensschnittstellen dabei unverändert). Der Schritt v149.2 → v150 ist
  **kein kleiner Schritt**: von 1934 namensgepaarten Funktionen unterscheiden sich 56, **26 davon
  inhaltlich**, dazu **20 neue und 1 entfernte Funktion** (Detail: `VNSD_Control_Diff_v1492_v150_de.md`).
- Größte Änderungen v149.2 → v150: ein komplett neuer HTTPS/TLS-Client (mbedTLS über CH395) für den
  Cloud-Telemetrie-Upload, der eine AES-128-ECB-verschlüsselte, Base64URL-kodierte GET-Anfrage an
  `hamedata.com` durch einen Klartext-JSON-POST an `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s`
  ersetzt und dabei einen echten Pufferüberlauf in v149.2 behebt (§4.4); eine neue Peak-Shaving-/
  Leistungsklemmfunktion (EEPROM 0x394, neues BLE-Kommando, zwei neue MQTT-Kommandos); zwei neue
  Verbindungs-Watchdogs (CH395-Chip-Reset, Modem-Reset) plus GET→POST-Umbau des Mobilfunk-Uploads
  (§7).
- Herstell-Changelog nennt fünf Punkte (Local-API-Fehler Ethernet, HTTP-Update-Fehler Ethernet,
  Peak-Shaving, Datenverlust bei langen HTTP-Datensätzen, CT_TYPE-Zähleranbindung); alle fünf lassen
  sich jetzt konkreten Codeänderungen zuordnen (§7, Zuordnungstabelle in `VNSD_Control_Diff_v1492_v150_de.md`).
- Neuer Shell-Befehl `FreeRtos_mem` (Heap-Anzeige, §4.3) gehört zu den 20 neuen Funktionen des
  v149.2→v150-Schritts (`ShellCmd_FreeRtos_mem`) und ist im Diff-Dokument erfasst (§7/§9).
- Sicherheitslage wie in den Nachbarversionen: Modbus-Register 46000 = 0x5100 schaltet ohne
  Authentifizierung die Debug-Shell auf RS485 **und TCP 8091** dauerhaft frei (§8).

## 0a. Prüfliste (v150 gegenüber Vorgänger 149/149.2 und Nachfolger 151)

| # | Frage | Antwort v150 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register geändert (ggü. 149/149.2)? | **Registertabelle selbst nein.** `g_modbusRegDescTable` (0x20000354, `ModbusRegDesc_t[246]`) byte-identisch über beide Hops (Registernummer, Typ, Skalierung, Anzahl); nur bei 215 Einträgen verschiebt sich die RAM-Schattenadresse, weil das Config-Struct für Peak-Shaving um 6 B wuchs (§7). Schedule-EEPROM (Register ≥46500) zieht dabei von 0x36BF nach 0x3700 um. Ggü. **151**: Lesetabelle bis auf 32102 (`float32`→`i16` in 151) identisch, 151 ergänzt Schreibregister 0xB799. | §4.1, Diff-Dokus |
| 2 | BLE geändert (ggü. 149/149.2)? | **Erweitert.** v149.2 → v150 kommt ein neues Kommando `)` (0x29, „Set Peak Shaving") hinzu, zwei Antwortlängen wachsen (`BLE_Build_DeviceStatus_Response` 0xA4→0xA7 B, `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` 0x50→0x51 B) und ein Fallthrough-Bug (Kommando `Y`→`Z`, fehlendes `break`) wird behoben. Ggü. **151**: 3 weitere Detailänderungen (EPS-Reset-Flag entfällt, LED-Status-EEPROM-Fix in 151). | §4.5, §7 |
| 3 | AC-Einspeisung 2500 W geändert? | **Hartgrenze unverändert** über alle drei Versionen (`Grid_Export_Power_Limiter` bei v150 startet mit `min(P_bms, 2500 W)`). Zusätzlich, unabhängig davon: v149.2 → v150 führt einen konfigurierbaren Peak-Shaving-Klemmwert ein (EEPROM 0x394, wirkt in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process`, §7) — er ergänzt die 2500-W-Klemme, ersetzt sie nicht. Erst **151** ändert den Einspeise-Startwert selbst (Netz-Messwert − P_neg, ohne 2500-W-Klemme). | §5, §7 |
| 4 | Firmware-Update per BLE weiter möglich? | **Ja**, OTA-/BLE-OTA-/YMODEM-Pfad seit v149 byte-identisch bis einschließlich v150; **151** ergänzt nur eine Typprüfung im YMODEM-Weg (Dateiname→Zielgerät), ändert den BLE-OTA-Pfad selbst nicht. | §4.5 |
| 5 | Cloud-Domains geändert? | **Der Telemetrie-Upload wechselt genau in diesem Schritt** von `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (AES+Base64URL, GET) auf `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (JSON, POST, §4.4); alle übrigen `hamedata.com`-Endpunkte (`getDateInfo`, `getDevicesApply`, `backup-configs`, `schedulings/policies`) bleiben unverändert. Erst **151** verschiebt zusätzlich `getDateInfo` von hamedata auf `marstekcloud.com`. | §4.4, §7 |

## 1. Binary-Fingerprint

| Merkmal | Wert |
|---|---|
| Datei | `150_control_VNSD-0_app_0150_0805_115146.bin` |
| Größe | 389 120 Byte (0x5F000) |
| SHA-256 | `4140e7a6d38ac90c8ceb6a5682f7704fd3ad4a45c2cb3fc1f05c95bfc1b71447` |
| API-CRC (Hersteller) | 7326 |
| Link-Adresse | `0x08004800`–`0x08063FFF` |
| MCU | Nations N32G45x mit XFMC (Cortex-M4F, Thumb-2), früher fälschlich als Artery AT32F403A/407 eingeordnet (s. §1a) |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW) |
| Bibliotheken | FreeRTOS, mbedTLS 2.28, cJSON, llhttp, letter-shell, N32-Standard-Peripheriebibliothek (n32_stdperiph, früher als „AT32-Firmware-Library" geführt), WCH CH395-Library |
| Initial-SP | `0x2001F7E8` (Vektortabelle @ 0x08004800, geprüft im Rohimage) |
| Reset-Vektor | `0x08004A71` (Thumb) → `SystemInit` → `__main` → `__scatterload` |
| Versionskonstante | **0x96 (150)** @ 0x08010000, String `VNSD-0` @ 0x08010004 |
| Versionsstring | `VNSD-0 v150` (u. a. @ 0x08051D90) |
| Build-Stempel | `Aug  5 2026 11:51:21` |
| Image-Footer | `VNSD` + Nullbytes am Ende der 380-KB-Nutzdaten, danach 0xFF-Füllung bis 0x08063FFF |

**Scatter-Load-Tabelle** `Region$$Table$$Base` @ 0x08061EEC (`scatter_region_t[3]`, direkt im
Rohimage nachgerechnet):

| # | Quelle (Flash) | Ziel | Länge | Verfahren |
|---|---|---|---|---|
| 1 | 0x0806240C | 0x20000000 (.data) | 0x2F00 (12 032 B) | `__decompress` @ 0x08005FBE (LZ77) |
| 2 | 0x08062DC4 | 0x60000000 (.xmc_bss) | 0x26AE0 (157 952 B) | `__decompress` @ 0x08005FBE (LZ77) |
| 3 | – | 0x20002F00 (.bss) | 0x1C8E8 (117 000 B) | `__scatterload_zeroinit` @ 0x080367E0 (Null-Init) |

`__scatterload_copy` (0x080367D0, 2 B) springt dabei in eine separate 12-Byte-Kopierschleife
`Scatterload_CopyBody` (0x080367D8) — in v151 zu einer einzigen 14-Byte-Funktion verschmolzen,
funktional identisch.

## 1a. MCU-Identifikation: Nations N32G45x, nicht Artery AT32F403A

- Der Control/EMS-MCU ist ein **Nations N32G45x mit XFMC** (External Memory Controller), nicht der zuvor angenommene Artery AT32F403A.
- Belege: QSPI-Controller bei 0xA0001000 mit N32-QSPI_STS-Bits; RCC-PLL-Multiplikator in Bit 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG genutzt; SAC-Krypto-Coprozessor bei 0x40024000 (früher fälschlich „RNGX" genannt); 96-Bit-UID @ 0x1FFFF7F0; 102 Vektoren.
- Externes SRAM 0x60000000–0x60026AF7 läuft über XFMC, Steuerregister bei 0xA0000000: der offizielle Nations-Header `n32g45x.h` (lokal, nicht im Repo) definiert `XFMC_REG_BASE 0xA0000000` und `QSPI_BASE = XFMC_REG_BASE + 0x1000`; die Nations-Treiberbibliothek enthält `n32g45x_xfmc.c`; die Firmware selbst hat den Shell-Befehl `xfmc_test` (das aktuelle englische N32G45x-Benutzerhandbuch markiert diesen Adressraum fälschlich als „Reserved").
- Details/Header-Kopie: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Aufbereitungsstand in Ghidra (`/VNSD/Control/Control_150_VNSD-0_app_0150_0805_115146.bin`)

Alle Prüfschritte bestanden (`marstek_gates.py` (lokal, nicht im Repo), 2026-09-30):

| Prüfung | Ergebnis |
|---|---|
| Undefinierte Flash-Bytes / Instruktionen ohne Funktion | 0 / 0 |
| Code als Daten (Sprungziele, Funktionszeiger, Tabelleneinträge, Literal-„Funktionen") | 0 |
| Funktionen | **1954**, alle benannt, keine `FUN_`/`tmp_`/Duplikate |
| Prototypen | alle gesetzt (keine Default-Signatur) |
| Referenzierte RAM-/XMC-Globals ohne Namen | 0 |
| Task-Namen gegen `g_taskDefTable`, Shell-Handler gegen `shellCommandTable` | stimmen |
| Doppelte Global-Labels | 0 |

Ein Teil der Funktionen trägt bereits Plate-Kommentare mit expliziten v150↔v151-Verhaltensvergleichen
(z. B. `Grid_Export_Power_Limiter`, `Write_Handler`, `BLE_App_Command_Dispatcher`, `HTTP_GetUrlData`,
`CH395_Init_Modbus_TCP_Socket`) — diese wurden für die vorliegende Analyse als Primärquelle verwendet
und stichprobenartig gegen das Rohimage nachgerechnet (u. a. Scatter-Tabelle, Task-Tabelle,
RPC-Methodentabelle, s. unten).

## 3. FreeRTOS-Tasks

Tabelle `g_taskDefTable` @ **0x0805A764** (`TaskDef_t[14]`, Format `{func, name, stackWords, param,
priority, handle*}`, 24 B/Eintrag), erzeugt von `Task_CreateTasksAndStartScheduler` (0x0801E244).
Direkt aus dem Rohimage dekodiert:

| Task | Prio | Stack (Wörter) |
|---|---|---|
| Task_ch395 | 15 | 256 |
| Task_ch_mqtts | 14 | 1024 |
| Task_parser | 13 | 512 |
| Task_ch_http | 13 | 1024 |
| Task_fc41d | 12 | 512 |
| Task_http | 11 | 512 |
| Task_udp | 10 | 256 |
| Task_ems | 9 | 256 |
| Task_ct | 8 | 512 |
| Task_Modbus_tcp | 7 | 512 |
| Task_can_comm | 7 | 256 |
| Task_can_receive | 6 | 256 |
| Task_rs485 | 3 | 512 |
| Task_iwdg | 1 | 128 |

Alle Prioritäten und Stackgrößen sind identisch mit v151 (`VNSD_Control_Diff_v150_v151_de.md` bestätigt
dies ausdrücklich als unverändert). `Task_Modbus_tcp` (Prio 7) liegt unter den beiden TLS-Tasks
(`Task_ch_mqtts` 14, `Task_ch_http`/`Task_parser` 13) — Modbus-TCP kann bei TLS-Aktivität warten.

## Chinesische Strings

119 UTF-8-Strings mit chinesischem Text im Export, alle mit `ZH->EN`-Kommentar übersetzt (2 weniger als
v151, das zusätzlich 2 neue Strings für die dortigen Shell-Befehle `read_inv1`/`read_inv2` enthält).
Zwei chinesische Debug-`printf` bleiben unübersetzt im Nutzpfad stehen (kein reiner Kommentar, sondern
Laufzeitausgabe): im `Write_Handler`-Testzweig für Register 47400 —
`"47400要写进去里面的数据 = 0x%x\r\n"` („Zu schreibender Wert für 47400 = 0x%x") und
`"没到里面吗123456\r\n"` („Nicht reingekommen, oder? 123456", offensichtlicher Debug-Rest).

## 4. Schnittstellen

### 4.1 Modbus (TCP 502 über CH395, RTU über RS485)

- Lese-Deskriptortabelle `g_modbusRegDescTable` @ **0x20000354**, `ModbusRegDesc_t[246]`
  (`reg_addr`, `pad`, `source`, `type_code`, `type_code2`, `scale`, `count`), Register 30000–38014 —
  über den gesamten v149→v150-Sprung byte-identisch (`FC03_Read_Handler` @ 0x0802386C,
  `Read_Serializer` @ 0x08055414, unverändert seit v149).
- Schreibpfad `Write_Handler` @ **0x08056514** (3998 B): Register **46000** (intern `reg−45000 = 1000`)
  mit Wert **0x5100** setzt `g_debugMode = 1`, persistiert EEPROM 0x900 — dauerhafter Debug-Modus.
  Werte 0x04D2/0x0929/0x0D80/0x11D7 wählen je das YMODEM-Update-Ziel (`g_otaTargetDevice` 1/4/3/2).
  Register 47400 (Testzweig) hat noch die chinesischen Debug-`printf` von oben.
- Register 0xB799 (47001, KI-Modus/`CAN_Write_Category0x18`) und die um ein Register verschobene
  Sub-Indizierung im 45000er-Block gibt es in v150 **noch nicht** — beides kommt erst mit v151
  (`VNSD_Control_Diff_v150_v151_de.md`).

### 4.2 Local API / MQTT-RPC (`g_mqttRpcMethodTable` @ 0x0805D944, `MqttRpcMethod_t[16]`)

Direkt aus dem Rohimage dekodiert, Dispatcher `MQTT_JSON_RPC_Dispatcher` @ 0x0801F36C:

| ID | Methode | ID | Methode |
|---|---|---|---|
| 1 | Marstek.GetDevice | 9 | EM.GetStatus |
| 2 | Wifi.GetStatus | 10 | Marstek.GetData |
| 3 | BLE.GetStatus | 11 | DOD.SET |
| 4 | Bat.GetStatus | 13 | Ble.Adv |
| 5 | PV.GetStatus | 14 | Led.Ctrl |
| 6 | ES.GetStatus | 15 | Ota.Set |
| 7 | ES.SetMode | 16 | Set.Ver |
| 8 | ES.GetMode | 17 | Reset.Factory |

16 Einträge, ID 12 (0xC) ist nicht vergeben (keine Lücke in der Tabelle, sondern schlicht übersprungen).
Laut eingebettetem Vergleichskommentar ist diese Funktion inhaltlich identisch mit v151 (einzige
v151-Änderung: EEPROM-Persistenz des SEL-LED-Flags 0x395 bei `Led.Ctrl`, in v150 noch nicht vorhanden —
der gesetzte Wert übersteht in v150 daher keinen Neustart).

### 4.3 Debug-Shell (letter-shell, `shellCommandTable` @ 0x08061F1C, `ShellCommand[79]`)

79 Einträge = **66 Befehle** (68 `ShellCmd_*`-Handler inkl. `help`/`clear`) + 1 Benutzereintrag
(`VNSD-0 v150`, leeres Passwort) + 10 Tastenbelegungen. Gegenüber v149/v149.2 (`ShellCommand[78]`,
65 Befehle) neu: **`FreeRtos_mem`** (0x0802F540, gibt aktuellen und historischen Minimal-Heap aus,
`FreeRTOS_xPortGetFreeHeapSize`/`_xPortGetMinimumEverFreeHeapSize`) — eine der 20 neuen Funktionen des
v149.2→v150-Schritts, siehe §7/§9. Gegenüber v151 (70 Befehle) fehlen `read_inv1`/`read_inv2`.

Erreichbarkeit: `Task_rs485` bedient je nach `g_debugMode` (EEPROM 0x900) Modbus RTU (0) oder die Shell
auf RS485 und CH395-TCP-Server **Port 8091** (`g_ch395SockModbusTcp`-Tabelle @ 0x2001ADDC, Socket 1);
Modus 2 wartet auf die Zeichenfolge `quit_debug_5100_` (`RS485_DebugConsole_ScanQuitPhrase` @
0x080550DC), danach Modus 1. Shell-Befehl `modbus` schaltet zurück auf 0.

### 4.4 Cloud

Alle Endpunkte im Rohimage verifiziert:

| Endpunkt | Schema | Zweck |
|---|---|---|
| `http://%s.hamedata.com/app/neng/getDateInfo%s.php?...` | HTTP, Port 80 | Versions-/Zeitabgleich (`sv=0x96`=150 fest kodiert) |
| `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` | HTTPS | Telemetrie-Upload — **neu seit v150** (s. u.) |
| `http://%s.hamedata.com/ems/api/v1/getDevicesApply?devid=%s` | HTTP | WLAN-Konfigurationspfad |
| `http://%s.hamedata.com/external-services/api/v1/devices/backup-configs?devid=%s&latest=true` | HTTP | Cloud-Backup-Konfiguration |
| `https://%s.hamedata.com/external-services/api/v1/schedulings/policies?devid=%s` | HTTPS | Zeitplan-/Richtliniensynchronisation |
| `http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl` | HTTP | FC41D-Modul-OTA |

**Telemetrie-Upload neu seit v150 (Detail: `VNSD_Control_Diff_v1492_v150_de.md`):** v149.2 sendete das
Telemetrie-Paket noch AES-128-ECB-verschlüsselt und Base64URL-kodiert als Query-Parameter `v=` per
Klartext-HTTP-GET an `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s`. v150 verzichtet auf
Verschlüsselung und Kodierung, verpackt die Nutzdaten nur noch in `{"d":"..."}` und sendet sie über einen
neuen HTTPS/TLS-Client (mbedTLS über CH395: `HTTPS_FetchUrlData`, `HTTPS_TLSSessionInit`,
`HTTPS_PackPostRequest`, Port 443) als JSON-POST an den obigen marstekcloud.com-Endpunkt (Ethernet-Pfad;
bei Mobilfunk sendet `Quectel_HTTP_POST_SendAndVerify` denselben JSON-Body per `AT+QHTTPPOST`). Dabei
wird ein echter Pufferüberlauf aus v149.2 behoben: `Base64URL_Encode` wurde dort mit einer behaupteten
Zielpuffergröße von 0x960 (2400 B) aufgerufen, der tatsächliche Puffer `g_cloudUploadBodyBuf` ist aber
nur `uchar[1200]` (1200 B) groß — bei AES-Ausgaben mit Base64-Länge zwischen 1200 und 2400 B schrieb die
Funktion über das Pufferende hinaus in benachbarte Globals. Erklärt den Changelog-Punkt „Datenverlust
durch zu lange HTTP-Datensätze behoben" direkt als Speicherüberlauf-Fix, nicht nur als Effizienzgewinn.
Alle übrigen `hamedata.com`-Endpunkte bleiben in diesem Schritt unverändert.

Regionstabelle (EEPROM 0x441, `g_cloudDomainUrlTable` @ 0x20000FD0): `eu, www, us, eu-dev, eu-staging`,
**gültig nur 0–2** (`HTTP_GetUrlData` @ 0x080199AC setzt jeden anderen Wert im EEPROM stillschweigend
auf 0 zurück — erst v151 erweitert die Prüfung auf 0–4). MQTT-Themen: `marstek_energy/%s/device/%s/ctrl`
und `marstek_energy/%s/App/%s/ctrl`, TLS über mbedTLS mit `authmode 2` (§8).

### 4.5 BLE (Quectel FC41D, AT-Kommandos)

Gerätename `MST_VNSD_`+4 Zeichen (`BLE_Set_Device_Name` @ 0x0800F464, `AT+QBLENAME=`).
Befehlsverteiler `BLE_App_Command_Dispatcher` @ 0x0800C720 (7062 B, ~40 Unterbefehle) — v149.2 → v150 kommt
Kommando `)` (0x29, Peak Shaving) hinzu und ein Fallthrough-Bug (`Y`→`Z`) wird behoben, sonst Befehlssatz
seit v149 unverändert, nur Antwort-/Telemetrieaufbau geändert (§7). BLE-OTA: `BLE_Cmd_OTA_Init` (0x08033C48),
`_WriteSetup` (0x08033CDC), `_Validate` (0x08033B68), `BLE_OTA_WriteDataChunk` (0x080340E0) — Pfad seit
v149 byte-identisch, BLE-Firmware-Update also uneingeschränkt möglich.

## 5. Leistungsgrenzen (für die 2500-W-Frage)

- `Config_ClampDefaults_ParamBlock` @ **0x0800935C**: `g_maxChargePowerW`/`g_maxDischargePowerW` werden
  bei 0 oder > 0x9C4 (2500 W) auf **0x9C4 = 2500 W** geklemmt (Entladen-Fallback sonst 800 W +
  `g_workMode = 1`). `g_chargeCutoffSoc` Default 100 %, `g_dischargeCutoffSocRaw` Bereich 12–70 %,
  Default 12 %.
- `Set.Ver` (MQTT-RPC-Methode 16, `MQTT_JSON_RPC_Dispatcher` case 0x10): nur 800, 0x898 (2200 W) oder
  0x9C4 (2500 W) werden akzeptiert, sonst `Invalid params` — identisch mit v149.
- `Grid_Export_Power_Limiter` @ **0x08023A00** (alle 4 Ticks): PV-Obergrenze bei Einspeisung
  (`g_gridSamplePowerW ≥ 1`) = **`min(-P_bms, 2500 W)`** (0x9C4-Klemme direkt im Code), sonst
  `min(cap, ac + lim + 300)`; Gesamtklemme in jedem Fall **4000 W**. Erst v151 ersetzt den
  Einspeise-Startwert durch `Netz-Messwert − P_neg` ohne die 2500-W-Klemme (Gesamtklemme 4000 W bleibt).
- Umrechnung BMS-Stromlimit → Leistung (`P_bms = 32100 × 32106`) und das Zusammenspiel mit PV-Überschuss,
  Zwangsladung (42020) und `charge_to_soc` (42011, setzt `force_mode` auf 3) im Detail:
  `VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md` — dort auch die Feldeinschätzung, dass die
  2500-W-PV-Abregelung im Top-of-Charge-Taper der Batterie (≈1270→≈485 W, s. BMS-Doku) spürbar sein
  könnte, ohne Feldmessung des tatsächlichen PV-Verlusts.

## 6. Struktur-/Datenbefunde

- mbedTLS-Cipher-Tabelle `aes_info`/`aes_128/192/256_ecb_info`/`gcm_aes_info`/`aes_128/192/256_gcm_info`
  ab **0x0805D9C4**, `mbedtls_cipher_definitions[7]` @ 0x0805DA9C — Struktur identisch mit den anderen
  Versionen, nur relokiert.
- **BMS-Pack-Anzahl (gilt für 147–151, Feldbeobachtung s. unten):** 8 Pack-Records im RAM ab dem
  Bereich **0x200177d4** (Schrittweite in v150/v149.2/151 = **96 B**, in v149 noch 88 B —
  `BMS_CellVoltage_MinMax_PerPack` @ **0x08017B14** vergrößerte den Record im v149→v149.2-Schritt),
  CAN-Batteriezahl auf 8 begrenzt, Modbus-Blöcke 34000…34600 (7 Packs). Die Zellstatistik läuft aber nur
  über **6 Packs** (`pack < 6`, Tabelle `g_bmsCellStat0…` ab 0x20018FC0, 6 × 6 B) → der 7. Pack (Modbus
  vorhanden) fehlt in `g_bmsCellStat*` und der daraus gebauten MQTT-Telemetrie.
- **Pack-Nummerierung / Off-by-one im MQTT-Pfad (gilt für 147–151):** CAN-Einheit u → Record u−1
  (`CAN_AA_StoreUnit*`); Modbus 34000+100·n und `ShellCmd_get_all_bms_data` lesen konsistent Record n.
  `MQTT_Publish_BMS_Data` @ **0x080288E8** (MQTT-Befehl 0x2A) liest dagegen für Index p ≥ 1 Record p
  (Index 0 = PGN-0x1801/0x1802-Gesamtwerte, 0xFF = fp64-skalierte Übersicht) → „BMS(1)" per MQTT zeigt
  CAN-Einheit 2, Einheit 1 ist per MQTT einzeln nicht abrufbar. Index 8 liest hinter das Record-Array;
  ab dort liegt in v150 `g_pendingBootDescriptor` @ **0x200152D4** (Update-Deskriptor: Parameter, CRC16,
  Versions-Tag) — reiner Lesezugriff, keine Speicherverletzung.
- Werkseitige WLAN-Zugangsdaten `SM471`/`Hame147258` an drei Stellen im Code (String-Literale
  @ 0x080169B8/0x0802E0D0/0x08052DBC): `Config_LoadOrInit_DeviceSettings` (0x0802DCBC),
  `Quectel_WiFi_AutoConnect_StateMachine` (0x0801677C), `ShellCmd_set_wifi` (0x08052D80).
- **Feldbeobachtung (Cloud-Upload-Payload, nicht in dieser Sitzung erneut geprüft):** Feldmitschnitt des
  HTTP-Upload-Payloads inkl. bestätigter Feldnamen (`wm`, `sc`, `pb`, `bv`, `bi`, `go`, `mc`/`md` u. a.)
  in `VNSD_Cloud_Upload_Payload_v150_de.md`; ebenso die Region `eu` und die `getDateInfo`-Parameter dort
  gemessen bestätigt.

## 7. Nur in v150 / geändert ggü. Vorgänger (149/149.2) und Nachfolger (151)

**v149 → v149.2 (Detail: `VNSD_Control_Diff_v149_v1492_de.md`, 43 Funktionen), unverändert nach v150
übernommen:**
- BMS-Pack-Record 88 → 96 B (§6).
- Französischer Eigenverbrauchsregler SMR/TIC (Linky) stark vereinfacht (408 → 247 Instruktionen).
- Umbau von Cloud-/MQTT-Telemetrie, Status-Reporting, Telemetrie-Historie, UDP-Zähler-Parse (Register-
  und Verhaltensschnittstellen dabei unverändert, nur interner Aufbau).

**v149.2 → v150 (Detail: `VNSD_Control_Diff_v1492_v150_de.md`; 56 der 1934 namensgepaarten Funktionen
unterscheiden sich, davon 26 inhaltlich, dazu 20 neue und 1 entfernte Funktion):**
- **Cloud-Telemetrie-Upload komplett überarbeitet** (`Cloud_UploadTelemetryData`) — neuer HTTPS/TLS-Client
  (mbedTLS über CH395: `HTTPS_FetchUrlData`, `HTTPS_TLSSessionInit`, `HTTPS_TLS_Context_Reset`,
  `HTTPS_PackPostRequest`, `HTTPS_ParseData`/`_ParserOnBody`, `CH395_TCP_Socket_Open_HTTPS`,
  `CH395CMDGetSocketStatus`, `mbedTLS_SSL_Send/Recv_WithRetry`) ersetzt AES-128-ECB+Base64URL-GET an
  hamedata.com durch JSON-POST an marstekcloud.com und behebt dabei einen echten Pufferüberlauf in
  `Base64URL_Encode` (§4.4).
- **Peak-Shaving/Leistungsklemmmodus neu** (EEPROM 0x394, 6 B): `Config_Read_/Write_PowerClampMode_0x394`,
  neues BLE-Kommando `)` (0x29), neue MQTT-Kommandos 0x3f (Setzen)/0x41 (`MQTT_Publish_Grid_Status`,
  Lesen), wirkt in `Inverter_PowerSetpoint_DeadbandClamp` und `Remote_Power_Setpoint_Process` zusätzlich
  zur unveränderten 2500-W-Hartgrenze (§5).
- **Zwei neue Verbindungs-Watchdogs** plus GET→POST-Umbau des Mobilfunk-Uploads: `HTTP_ClientTask_Dispatcher`
  löst nach 1800 Ticks ohne Fortschritt einen CH395-Chip-Reset aus, `HTTP_Task_StateMachine_Handler` nach
  900 Ticks einen Modem-Reset; die Zustände 6–9 der Modem-Zustandsmaschine senden jetzt `AT+QHTTPPOST`
  (`Quectel_HTTP_POST_SendAndVerify`) statt eines GET.
- `DeviceInfo_BuildStatusString`: 6 neue Hardware-Varianten-Tags für Zählertypen 3/4/8/9/10/11 statt des
  bisher generischen EEPROM-Strings.
- `Grid_PowerSync_StateMachine`: Zählertyp-Bereich für die CT-Direktübernahme ohne Deviation-Gate deutlich
  erweitert.
- BMS-Selbsttest-/MOS-Flag zusätzlich gemeldet (`BLE_Build_BMS_Data_Response`, `BMS_StatusQuery_Response_Send`).
- Off-by-one-Bug behoben: `Config_Read_String_0x387`/`_Write_String_0x387` griffen trotz Namens auf
  EEPROM 0x388 statt 0x387 zu.
- `HTTPS_POST_Request`: Empfangspuffer 6488 → 7616 B, jetzt gemeinsam mit dem HTTPS-Pfad genutzt.
- `HTTP_ParseUploadResponseCode`: sucht jetzt den JSON-Schlüssel `"code":` statt `"de":` — ein anderer
  Schlüssel, kein bloß verschobenes Leseoffset.
- Sechs kleinere Verhaltensänderungen: `MQTT_Publish_Network_Info` meldet die tatsächliche statt der
  angefragten Zähler-IP; `Quectel_MQTT_Config_StateMachine`/`_Session_StateMachine` ändern die
  TLS-Konfigurationsreihenfolge bzw. machen den Factory-Reset im SSL-Watchdog bedingt; `mbedTLS_SSL_Connection_Init`
  verliert einen redundanten TLS-Reset; `Battery_EPS_Reserve_SOC_Hysteresis` setzt ein Low-Byte im
  CAN-0xCF-Report neu auf 1 (Bedeutung offen); `Cloud_EdgeDetectAndWatchdog` bindet einen Retry zusätzlich
  an den MPPT-Kanal; `Quectel_HTTP_AT_SendAndVerify` bekommt eine zusätzliche Wartezeit nach dem Senden.
- **Korrektur gegenüber der Vorfassung dieses Diff-Schritts:** `CH395_TCP_Socket_Open_HTTPPort80`
  (vormals als Parameteränderung 0x24→0xAA gewertet) und `HTTP_PackGetRequest` (vormals als
  Puffervergrößerung 360→640 B gewertet) sind tatsächlich **kosmetisch** — beide Funktionskörper sind
  byte-identisch, die geänderten Werte sind nur `__LINE__`-Logargumente.
- Neuer Shell-Befehl `FreeRtos_mem` (`ShellCmd_FreeRtos_mem`, §4.3) gehört zu den 20 neuen Funktionen
  dieses Schritts und ist im Diff-Dokument erfasst — kein offener Widerspruch mehr (§9).

**v150 → v151 (Detail: `VNSD_Control_Diff_v150_v151_de.md`):** neues Modbus-Schreibregister 0xB799 +
`CAN_Write_Category0x18`, Modbus 32102 `float32`→`i16`, `getDateInfo`-Host wechselt zu
`marstekcloud.com`, Regionstabelle 0–4 statt 0–2, CH395-Keepalive IDLE 10 000→60 000 ms,
CH395-Hänger-Watchdog löst jetzt einen kompletten Neustart aus, YMODEM-Typprüfung, LED-Status wird
EEPROM-persistent, PV-Einspeise-Startwert ohne 2500-W-Klemme (§5), neue Shell-Befehle `read_inv1`/
`read_inv2`. Vollständige Tabelle mit alten/neuen Werten im Diff-Dokument.

## 8. Sicherheitsbefunde

1. **Unauthentifizierte Fernkonsole:** Modbus-TCP-Schreibzugriff (Port 502, ohne Auth) auf Register
   **46000 = 0x5100** → `g_debugMode = 1` dauerhaft im EEPROM (0x900) → letter-shell auf RS485 **und**
   TCP-Port 8091 (`g_ch395SockModbusTcp` Socket 1, @ 0x2001ADDC), Benutzer `VNSD-0 v150` mit leerem
   Passwort. Zauberwort `quit_debug_5100_` (`RS485_DebugConsole_ScanQuitPhrase` @ 0x080550DC) schaltet
   in Modus 2 ebenfalls frei.
2. **Fest eingebaute WLAN-Zugangsdaten** `SM471`/`Hame147258` (§6).
3. **HTTPS-Upload-Pfad ohne Server-Zertifikatsprüfung:** `mbedtls_ssl_conf_authmode(…, 0)` in
   `HTTPS_TLSSessionInit` @ 0x0803B8A0; MQTT prüft mit `authmode 2` (`mbedTLS_SSL_Connection_Init` @
   0x0801D134).
4. **Zertifikats-Gültigkeitszeitraum wird nie geprüft:** `mbedtls_x509_time_is_past` (0x0804DCCA) und
   `mbedtls_x509_time_is_future` (0x0804DCC4) sind reine `return 0`-Stubs
   (`MBEDTLS_HAVE_TIME_DATE` aus) — betrifft auch MQTT.
5. Cloud-Telemetrie-Report (`getDateInfo`) weiterhin über Klartext-HTTP (Port 80) an `hamedata.com`
   (§4.4); der eigentliche Telemetrie-Upload liegt dagegen bereits auf HTTPS/`marstekcloud.com`.
6. **Pufferüberlauf FC03 (gilt auch für v150, s. v151-Doku §11):** `FC03_Read_Handler` @ 0x0802386C
   schreibt die Antwort (bis 128 B, N ≤ 125 Register) in den TX-Rahmen und kopiert sie danach in einen
   Spiegelpuffer, der ab Offset 32 in den MQTT-RPC-Arbeitsbereich übergeht — Struktur im v150-Export
   stichprobenartig bestätigt (max. Registeranzahl 0x7D wie in v151), volle Byte-für-Byte-Nachrechnung
   nicht wiederholt.

## 9. Offene Punkte

- Wie in den Nachbarversionen unverändert und hier nicht erneut im Detail nachgewiesen: 32101-
  Vorzeichenfehler im Read-Serializer, MBAP-Länge 4 statt 3 bei Exception-Antworten, 44002/44003 lesen
  nach Neustart 0 (Schattenwert) — Übernahme aus den Diff-Dokumenten/der v151-Vollanalyse, für v150
  nicht erneut Schritt für Schritt nachgerechnet.
- Wie viel PV-Leistung die 2500-W-Abregelung im Top-of-Charge-Taper der Batterie tatsächlich kostet, ist
  nicht gemessen (`VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md` §7).
- 31 unzugeordnete Felder im Cloud-Upload-Payload (`VNSD_Cloud_Upload_Payload_v150_de.md`).
- `__vfscanf` (neu im v149.2→v150-Schritt eingebundene C-Bibliotheksfunktion): auslösende Aufrufstelle
  im Code nicht lokalisiert (`VNSD_Control_Diff_v1492_v150_de.md`).
- `ShellCmd_FreeRtos_mem`: über die Shell-Kommando-Linker-Sektion eingebunden statt über eine normale
  Aufruftabelle, kein Code-Querverweis auffindbar (Funktion selbst eindeutig: gibt freien/minimal freien
  FreeRTOS-Heap aus, §4.3).
- Fallthrough-Fix beim BLE-Kommando `Y`→`Z` (§7): ob das in v149.2 beobachtbare Fehlverhalten (fehlendes
  `break`) tatsächlich auslöste oder folgenlos blieb, wurde nicht geprüft.

## 10. Korrekturen

**Korrekturen an diesem Dokument (diese Überarbeitung, 2026-09-30):** Die Vorfassung stufte
v149.2 → v150 fälschlich als „drei Kleinigkeiten plus eine neue Shell-Funktion" ein und übernahm damit
eine unvollständige Funktionspaarung der damaligen Vorfassung von `VNSD_Control_Diff_v1492_v150_de.md`
(nur 9 unterschiedliche Funktionen gefunden). Die vollständige Namenspaarung zeigt: 56 der 1934
Funktionspaare unterscheiden sich, 26 davon inhaltlich, dazu 20 neue und 1 entfernte Funktion — u. a. ein
komplett neuer HTTPS/TLS-Client, der Cloud-Upload-Umbau samt behobenem Pufferüberlauf in v149.2,
Peak-Shaving und zwei neue Watchdogs (§0, §4.4, §7). Von den zuvor als „die 3 Verhaltensänderungen"
gelisteten Funktionen waren zwei falsch bewertet: `CH395_TCP_Socket_Open_HTTPPort80` und
`HTTP_PackGetRequest` sind kosmetisch (nur `__LINE__`-Logargumente), `HTTP_ParseUploadResponseCode`
ändert einen JSON-Schlüssel statt eines Leseoffsets. Der neue Shell-Befehl `FreeRtos_mem` war in der
Vorfassung als Widerspruch zum Diff-Dokument vermerkt; das ist erledigt — er ist dort jetzt unter den 20
neuen Funktionen erfasst.

**Ghidra-Korrekturen dieser Sitzung (versionsübergreifend, betreffen auch den v150-Export):** alle 8
BMS-Pack-Records einheitlich als `g_bmsPack<k><Field>` benannt (in v149 hieß eine Funktion noch
`CAN_AA_StoreUnitFieldPair88`, jetzt `…Pair72`); ~300 Globalnamen versionsübergreifend harmonisiert — in
diesem Dokument betrifft das `g_cloudServerRegionTable` → `g_cloudDomainUrlTable` (§4.4); kurze, vormals
als Zahlen typisierte Strings sind jetzt als Strings typisiert; SPI-Flash-Kommandotabelle
`g_spiFlashStatusCmdTable` und `g_meterCtStabilizeDelayTable` typisiert; bisher undokumentierte
Peripherie benannt (Block 0x40024000, damals `RNGX_*`, mittlerweile als N32G45x-SAC-Krypto-Coprozessor
identifiziert und in `SAC_R00`/`R04`/`R0C`/`R10`/`R14` umbenannt, genutzt von `Flash_ReadWords`/`Flash_ReadWithECC`;
GPIOF/GPIOG @ 0x40011C00/0x40012000; `RCC_UNDOC_2C`, `PWR_UNDOC_0C`, `RTC_UNDOC_4C`; Bit-Band-Aliase
`BB_*`). Neue Verifikationsgates (tmp-Label, Platzhalter-Globals, Shell-Plate-Abgleich) bestehen für alle
Versionen inkl. v150; der Dekompilat-Export hat 0 Fehler.

Die für v147 dokumentierten Korrekturen (`VNSD_Control_FW_Analyse_147_202601281721320b2053125_de.md` §10)
betreffen Artefakte, die für den v150-Export bereits vorab bereinigt waren (Task-Trampoline,
Cipher-Bereichs-Fehlbenennung, `CH395_ConfigTcpKeepalive_Defaults`, `MQTT_Transport_NullCallback`).
- Nacharbeit 2026-10-12 (wie bei D 151 / A 1509): `Inverter_RegDirty_Mark_DischgVolt` → `Inverter_RegDirty_Mark_GridPermit` (übergibt 1, solange das Micro-Telemetriewort `g_invState+0x16` 0 ist) und `Inverter_RegDirty_Mark_BatteryParams` → `Inverter_RegDirty_Mark_PvVoltage` (übergibt 1, solange alle vier PV-String-Spannungen unter 0x8D = 14,1 V liegen); das Code-Muster wurde in diesem Image geprüft. Das „Inverter-Dirty-Bitmap" ist das Frontpanel-LED-Bitmap eines TM1650-artigen Treibers (Bit-Bang-I2C auf die Digit-Register 0x68/0x6A/0x6C/0x6E); Plate-Kommentare der LED-Wrapper und Globals aktualisiert. Gates erneut gelaufen: alle bestanden; Dekompilat- und Listing-Export aktualisiert.
