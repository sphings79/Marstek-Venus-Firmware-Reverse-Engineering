# Diff-Tabelle Control v150.9 → v151 (Venus A)

> 🇬🇧 English version: [VNSA_Control_Diff_v1509_v151.md](VNSA_Control_Diff_v1509_v151.md)

**Stand:** 2026-10-11. Beide Images mit korrekter Basis 0x08004800 in Ghidra, vollständig benannt (v150.9: 1941 Funktionen, v151: 1942).
Adressen: v150.9 → v151. **Hauptdokumentation:** [VNSA_Control_FW_v151_Vollanalyse_de.md](VNSA_Control_FW_v151_Vollanalyse_de.md)
Funktionspaare über gleiche Namen (1941), Instruktion für Instruktion verglichen mit Aufrufzielen per Name, referenzierten Strings und
Immediate-Werten: **17 geändert** (16 inhaltlich, 1 nur Aufruf-Offset), **10 nur in verschobenen Tabellen-/Literaladressen oder im Versionsstring geändert**,
**1 nur in v151**, **0 nur in v150.9**.

## Prüf-Checkliste

| Frage | Ergebnis |
|---|---|
| Modbus-Register | Unverändert: Deskriptortabelle bytegleich (246 Einträge), `Write_Handler`, `FC03_Read_Handler`, `Read_Serializer` instruktionsgleich; 30200 liest 151 |
| BLE | Unverändert: `BLE_App_Command_Dispatcher` instruktionsgleich |
| AC-Einspeiselimit | Unverändert: überall 1500 W (Modbus 44003 > 1500 W abgewiesen, BLE 0x16/0x17, MQTT, JSON-RPC, Fernbefehl), Grenze des Exportbegrenzers 2400 W |
| BLE-OTA | Weiterhin möglich, OTA-/YMODEM-Code unverändert (Modellprüfung `VNSA-0`, Bin-Typprüfung schon in v150.9) |
| Cloud-Domains | `getDateInfo`: `http://<region>.hamedata.com` → `http://api-<region>.marstekcloud.com`; Upload und alle anderen Hosts unverändert |
| Pack-Kapazität | Unverändert: 7 Pack-Datensätze im Control, 13 Zellen für 6 Packs durchsucht; A-BMS 1105 sendet Packs 1..6 |
| Nicht funktionale Register | Alle 216 A-Register (FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL) weiterhin wie eingestuft: keine ihrer Control-seitigen Ursachen hat sich geändert |

## Release-Note → Code

| Release-Note-Punkt | Befund im Code |
|---|---|
| 1 Modbus-TCP-Stabilität | `CH395_ConfigTcpKeepalive_Defaults` Keepalive-Idle 10 000 → 60 000 ms; `HTTP_ClientTask_Dispatcher` Verbindungs-Watchdog Grenze 3600 → 14 400, danach `System_Reboot()`, neues Unterdrückungs-Flag `g_httpWatchdogSuppressFlag` |
| 2 DOD-Fehlerzustand | keine Änderung gegenüber v150.9 (schon enthalten, siehe [VNSA_Control_Diff_v150_v151_de.md](VNSA_Control_Diff_v150_v151_de.md)); `CT_GridPower_Controller` erhält die D-v151-Prüfung, die einen positiven CT-Sollwert auf 0 zurücksetzt |
| 3 YMODEM-Upgrade-Typprüfung | keine Änderung gegenüber v150.9 (schon enthalten) |
| 4 KI-Modus-Prüfung / Abfrage der aktuellen KI-Strategie | `AI_ScheduleQuery_Response_Send` seitenweise, neue Hilfsfunktion `MQTT_CtrlResponse_Transmit` |
| 5 Fehler in der Vorab-Server-URL | `getDateInfo`-Host auf `api-<region>.marstekcloud.com`, Regionspräfixe `dev`/`staging`, Servertypen 0–4, Typ wird beim Booten geladen |
| 6 Stabilität des HTTPS-Uploads | `Quectel_HTTP_POST_SendAndVerify` neu geschrieben, `AT+QHTTPPOST` 120/120 s, äußerer Timeout 6000, TOU-`crc2` |
| 7 CT-Kommunikation 1,1 s | `Modem_SendStatusTelemetry` / `Ethernet_SendStatusTelemetry` Periode 2000 → 1100 Ticks |

## Änderungen

| Bereich | v150.9 | v151 | Ort (v150.9 → v151) | RN |
|---|---|---|---|---|
| Versionsstring / Build | `VNSA-0 v1509`, 8. Sep 2026 09:34:19 | `VNSA-0 v151`, 28. Sep 2026 09:54:57 | `CLI_InitSession` 08051CE4 → 08051E04, `ShellCmd_get_ver` 0803B68C → 0803B7AC | – |
| Versionswort / Typcode | 0x5E5 (1509) | 0x97 (151) | 0x08010000; `Config_LoadOrInit_DeviceSettings`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `ShellCmd_get_ver` | – |
| `getDateInfo`-Host (Quectel-Pfad) | `http://%s.hamedata.com/app/neng/getDateInfo%s.php` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php` | `HTTP_Cloud_Reporting_Dispatcher` 08012394 → 08012424 | 5 |
| `getDateInfo`-Host (CH395-Pfad) | gleiche hamedata-URL, Servertypen 0–2 | marstekcloud-URL, Servertypen 0–4 (andere → 0) | `HTTP_GetUrlData` 0801986C → 08019910 | 5 |
| Regionspräfix-Tabelle `g_cloudDomainUrlTable` (0x20000FC8) | eu, www, us, eu-dev, eu-staging | eu, www, us, dev, staging | .data, String-Tabelle 0x08060220… | 5 |
| EEPROM-Laden beim Booten | 0x441 und 0x395 nicht beim Booten geladen; kein Erststart-Default für 0x395 | 0x441 (Servertyp) und 0x395 (SEL-LED) geladen; Erststart schreibt 0x395 = 1 | `Config_LoadOrInit_DeviceSettings` 0802DA34 → 0802DB28 | 5 |
| HTTP-POST über das Modul | blindes Warten, keine ERROR-/`+QIURC`-Prüfung | Antwortschleife, ERROR/`+QIURC`, exakte Byteanzahl, Log der Post-Daten | `Quectel_HTTP_POST_SendAndVerify` 08010438 | 6 |
| `AT+QHTTPPOST`-Timeouts / äußerer Timeout | 60, 60 s / 600 | 120, 120 s / 6000, Body-Länge aus `g_cloudUploadBodyLen` | `HTTP_Task_StateMachine_Handler` 08010AC0 → 08010B4C | 6 |
| Telemetrie-Upload | – | Behandlung der Body-Länge wie D v151 | `Cloud_UploadTelemetryData` 0801B23C → 0801B300 | 6 |
| HTTPS-GET | – | Parameteraufbau wie D v151 | `HTTPS_FetchUrlData` 0801BC64 → 0801BD1C | 6 |
| Prüfsumme der TOU-Antwort | eine XOR-Prüfsumme | zweite Prüfsumme `crc2` (gleiches XOR), akzeptiert, wenn eine passt; Fehlertext mit `crc2` | `HTTP_Economy_TOU_ParseResponse` 0801938C → 08019418 | 6 |
| CH395-Keepalive-Idle | 10 000 ms | 60 000 ms | `CH395_ConfigTcpKeepalive_Defaults` 08037964 → 08037A84 | 1 |
| Cloud-Verbindungs-Watchdog | Grenze 3600 | Grenze 14 400, danach `System_Reboot()`; Zähler bleibt auf 0, solange `g_httpWatchdogSuppressFlag` = 1 | `HTTP_ClientTask_Dispatcher` 0801A294 → 0801A344 | 1 |
| KI-Fahrplanantwort (`cd=58`) | ein 1024-Byte-Puffer, `AI%d=…`, direkt gesendet | Seiten zu 24 Einträgen, Kopf `cd=%d,pg=%d/%d,`, `AI%u=…`, Grenzprüfung, gesendet über `MQTT_CtrlResponse_Transmit` | `AI_ScheduleQuery_Response_Send` 08023704 → 080237B8 | 4 |
| Neue Funktion | – | `MQTT_CtrlResponse_Transmit` (sendet eine `ctrl`-Antwort direkt über den Mobilfunk-Datenkanal oder als MQTT-PUBLISH) | 08023990 (nur v151) | 4 |
| Periode der Zähler-/CT-Abfrage | 2000 Ticks | 1100 Ticks | `Modem_SendStatusTelemetry` 08031418 → 08031538, `Ethernet_SendStatusTelemetry` 08031608 → 08031728 | 7 |
| CT-Sollwert | – | positiver CT-Sollwert wird zusätzlich auf 0 gesetzt (D-v151-Prüfung) | `CT_GridPower_Controller` 08030C10 → 08030D20 | 2 |
| Log-Zeilennummern | – | um 3 verschoben | `SMR_TIC_SelfUse_PowerController` 0800851C | – |

## Kosmetische Unterschiede (kein geändertes Verhalten)

- `Quectel_AT_SendAndPollResponse` 0800C1CC: ein Aufruf-Offset (gleiches Ziel, gleiche Logik).
- Nur verschobene Tabellen/Literale oder der neue Versionsstring: `Task_CreateTasksAndStartScheduler`, `MQTT_JSON_RPC_Dispatcher`,
  `MQTT_CtrlCommand_Dispatcher`, `pow`, `printf_Float_To_Digits`, `printf_Format_Engine`, `llhttp__internal__run`,
  `ssl_derive_master_secret`, `__main_after_scatterload`, `CLI_InitSession` (String `VNSA-0 v151`).

## Unverändert (ausdrücklich geprüft)

Modbus-Deskriptortabelle und -Handler, BLE-Dispatcher, alle CAN-Sender/-Dispatcher, OTA/YMODEM, Leistungsgrenzen und -klassen,
13-Zellen-Suche, RS485/USART1-Code, Task-Tabelle (14 Tasks), Shell-Kommandotabelle (81 Einträge), MQTT-RPC-Tabelle (16 Methoden), RAM-Layout
(Größen von `.data`/`.bss`/XMC und Initialwerte bis auf Flash-Pointer).
