# Diff-Tabelle Control v150 → v151 (Venus E3)

> 🇬🇧 English version: [VNSE3_Control_Diff_v150_v151.md](VNSE3_Control_Diff_v150_v151.md)

**Stand:** 2026-10-14. Beide Images mit korrekter Basis 0x08004800 in Ghidra, vollständig benannt. Adressen: v150 → v151.
Funktionspaare über identische Namen (1920 Paare): 1871 nach Maskierung von Pointern und BL-Offsets byteidentisch, 49 geändert
(19 kosmetisch, 30 mit echter Änderung), **0 nur in v150**, **7 nur in v151**. Der Vergleich ist namensgepaart und vollständig
(keine Version-Tracking-Paarung); die geänderten Funktionen wurden zusätzlich auf Dekompilat-Ebene, an String-Literalen und
Deskriptor-/Datentabellen verglichen. v150 hat 1920 Funktionen, v151 hat 1927.
**Hauptdokumentation:** [VNSE3_Control_FW_v150_Vollanalyse_de.md](VNSE3_Control_FW_v150_Vollanalyse_de.md) · [VNSE3_Control_FW_v151_Vollanalyse_de.md](VNSE3_Control_FW_v151_Vollanalyse_de.md)
**Gleicher Schritt auf der Venus D:** [VNSD_Control_Diff_v150_v151_de.md](VNSD_Control_Diff_v150_v151_de.md) — die E3-Änderungen sind dieselbe Menge wie auf der D
(gleiche Codebasis); die Abweichungen der E3 von dieser Tabelle stehen am Ende dieser Datei.

## Prüfliste (Dauerprüfungen)

| # | Frage | Ergebnis |
|---|---|---|
| 1 | Modbus-Register | Lesetabelle (246 Deskriptoren, 30000–38014) identisch außer **32102** (`float32`, 2 Register in v150 → `i16`, 1 Register in v151) und den RAM-Quellpointern. Schreibseite: neues Schreibregister **0xB799 (47001)** mit dem neuen Aufruf `CAN_Write_Category0x18(value)`; die OTA-Ziel-Auswahl im 46000-Block (Werte 1234 / 2345 / 3456 / 4567 → `g_otaTargetDevice` 1 / 4 / 3 / 2) antwortet in v150 mit Status 2, ruft in v151 aber `CAN_Write_Category0x18(value)` auf und liefert Erfolg. Die Register des hohen Blocks 0xAFDF (`Inverter_Set_Reg_0xFE`) und 0xAFE6 (`Config_Apply_SingleReg` 11 / 12) gibt es in beiden Versionen, nur über unterschiedlich kompilierte Dispatch-Pfade. Deskriptortabelle wandert 0x2000034C → 0x20000340. |
| 2 | BLE | Befehlssatz identisch (gleicher 600/800/2500-W-Wähler, gleiche Layouts von 0x03/0x04/0x13); Detailänderungen: Befehle 0x0F und 0x54 setzen die EPS-Zustandsvariable nicht mehr zurück (v151 hat keine), der LED-Status (Befehl 0x59) wird in v151 im EEPROM 0x395 gespeichert (v150: nur RAM), eine Fahrplan-Indexprüfung kommt hinzu. BLE-Fehlerhistorie 0x13 unverändert (20 × 14 Byte). |
| 3 | AC-Einspeisegrenze 2500 W | Unverändert: `Config_Write_U16_0x202`, `Config_Write_U16`, Defaults 0x9C4; Wähler 600 / 800 / 2500 W in beiden. Auf der E3 gibt es in keiner Version einen Einspeisebegrenzer. |
| 4 | Firmware-Update / BLE-OTA | BLE-OTA bleibt möglich. v151 ergänzt eine YMODEM-Dateityp-Prüfung (`OTA_ClassifyDeviceTypeTag`, Fehler −5 „Bin type mismatch! Upgrade aborted.“); die `VNSEE3-0`-Typprüfungen der OTA-Pfade und der HTTP-OTA-Downloadpfad (`HTTP_FindHeaderBodySeparator`, `HTTPS_POST_ReceiveResponseData`, `OTA_WriteDataToRingBuffer`) sind byteidentisch. |
| 5 | Cloud-Endpunkte | Der `getDateInfo`-Host wechselt von `http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php` zu `http://api-<region>.marstekcloud.com/app/neng/getDateInfo<suffix>.php`; Regionstabelle 3 → 5 gültige Werte; alle anderen Cloud-Hosts unverändert (hamedata schedulings/policies/backup-configs, marstekcloud Daten-Upload, MQTT). |
| 6 | Pack-Kapazität | Unverändert: `Protocol_AA_CommandDispatch` Unit-Grenze `< 9`, 8 Pack-Datensätze à 0x60 Byte (`g_bmsPack0Voltage` + k·0x60), 16 Zellen (die MQTT-Telemetrie gibt die 16 Zellspannungen von Datensatz 0 aus), Modbus-Pack-Blöcke 34000–34699. Die BMS-seitige Sendergrenze ist nicht Teil dieses Control-Images (siehe E3-BMS-115-Analyse). |
| 7 | Diff-Tabelle | Diese Datei. |
| 8 | Neuprüfung nicht funktionaler Register | Auf der Control-Seite hat kein Register seinen Zustand geändert außer 32102 (v150: `float32`, liest 4 Byte des i16-Paars `g_batSamplePowerW` + `g_batSampleVoltage_0p1V`, also ein sinnloser Wert; v151: exakter Alias von 30001). Gründe, die im Micro/BMS liegen (Empfängerfilter, fehlende Sender), werden von einem reinen Control-Image nicht berührt. Der neue Schreibpfad (47001, 46000-Zielauswahl) sendet CAN-Kategorie 0x18; sein Empfänger ist das Micro (hier nicht neu geprüft). |

## Changelog v151 → Code (Venus E3)

Das Hersteller-Changelog von E3 v151 nennt dieselben sieben Punkte wie das der Venus D; sie bilden sich auf folgende Codeänderungen ab (gleiche Funktionen wie auf der D, E3-Adressen):

| Changelog-Punkt | Codebefund (v150 → v151) |
|---|---|
| Modbus-TCP-Stabilität | `CH395_ConfigTcpKeepalive_Defaults`: IDLE 10 000 → 60 000 ms (CNT 5 und INTVL 3000 ms unverändert); `CH395_Init_Modbus_TCP_Socket` wendet die Keepalive-Defaults jetzt zuerst an; `HTTP_ClientTask_Dispatcher`: CH395-Hang-Watchdog (angestaute Telemetrie-Datensätze > 1, 1800 Ticks, `CH395_Reset_And_Reinit`) zu einer gestuften Reaktion umgebaut (Datensätze ≥ 5, 14 400 Ticks, dann `System_Reboot()`, neues Unterdrückungs-Flag `g_httpWatchdogSuppressFlag`) |
| DOD kann wegen abnormalem Status nicht entladen | `Battery_EPS_Reserve_SOC_Hysteresis`: Drei-Zustands-Maschine mit automatischem Wiedereinschalten (`g_epsReserveState`) durch eine zustandslose Variante ersetzt, die nur bei SOC ≠ 0 läuft und das EEPROM-Flag nie automatisch wieder setzt; `CT_GridPower_Controller`: neue Schutzabfrage setzt einen hängenden positiven Sollwert auf 0, solange `g_epsReserveActiveFlag` gesetzt ist; `MQTT_StatusReport_BuildAndPublish` hängt `,df=%d` an |
| YMODEM-Upgrade-Typprüfung | `YModem_ReceiveFile` klassifiziert den Dateinamen mit der neuen `OTA_ClassifyDeviceTypeTag` (EMS / BMS / INV / MPPT) und bricht bei Typ 0 oder bei Nichtübereinstimmung ab (zwei YMODEM-CAN-Bytes 0x18, Fehlercode −5); `YModem_FirmwareUpgrade_Handler` erhält die Meldung „Bin type mismatch! Upgrade aborted.“ |
| AI-Modus-Prüfung optimiert / aktuelle AI-Strategie abfragen | `AI_ScheduleQuery_Response_Send` (cd 0x3A): v150 formatiert alle Einträge in einen 1024-Byte-Puffer ohne Überlaufprüfung je Eintrag; v151 sendet seitenweise (`cd=%d,pg=%d/%d,`, 24 Einträge je Seite) über die neue `MQTT_CtrlResponse_Transmit` |
| Pre-Release-Server-URL-Fehler (Daten-Upload-Fehler) | `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`: `getDateInfo`-Host `hamedata.com` → `api-<region>.marstekcloud.com`; das Dateisuffix hinter `getDateInfo` ist in v151 fest `eu` (v150: aus `g_cloudGetDateInfoSuffixTable`, nach Servertyp indiziert); Regionstabelle 3 → 5 gültige Werte (`dev`, `staging` statt `eu-dev`, `eu-staging`); `Config_LoadOrInit_DeviceSettings` lädt den Servertyp beim Start |
| HTTP-Upload-Datenstabilität optimiert | `Quectel_HTTP_POST_SendAndVerify`: von zwei blinden `vTaskDelay`-Aufrufen zu einer echten Tick-Schleife, begrenzt durch `timeout_s`, mit ERROR- / `+QIURC`-Erkennung und exakter Byte-Anzahl (`UART_TransmitBytes`); `HTTP_Task_StateMachine_Handler`: `AT+QHTTPPOST=<len>,60,60` → `,120,120`, äußeres Timeout 600 → 6000, Body-Länge jetzt `g_cloudUploadBodyLen`; `prvHeapInit`: FreeRTOS-Heap 0x117F0 (70 KB) → 0x13FF0 (80 KB) |
| CT-Kommunikationsfrequenz auf 1,1 s optimiert | `Modem_SendStatusTelemetry` / `Ethernet_SendStatusTelemetry` laufen jetzt höchstens alle 0x44C = 1100 Ticks (`g_statusTelemetryNextTick`); `Meter_ParseUDPMeterInfo` entfernt das blockierende `vTaskDelay(g_meterActiveCtType * 1200)` nach einem gültigen Sample und zählt nur noch `g_modemRecvOkCount` |

## Änderungen

| Bereich | v150 | v151 | Ort (v150 → v151) |
|---|---|---|---|
| Versionsstring / Build | `VNSE3-0 v150`, `Aug  4 2026 15:11:03` | `VNSE3-0 v151`, `Sep 28 2026 09:39:27` | Strings; `ShellCmd_get_ver` 08038960 → 08038F80 |
| Versionskonstante (0x08010000, Cloud-URLs, Gerätetypcode) | 0x96 (150) | 0x97 (151) | `HTTP_Cloud_Reporting_Dispatcher` 08011B60 → 08011BF0, `HTTP_GetUrlData` 08018CB8 → 08018D78, `Cloud_UploadTelemetryData` 0801A690 → 0801A754, `Config_LoadOrInit_DeviceSettings` 0802B28C → 0802B69C, `ShellCmd_get_ver` |
| `getDateInfo`-Host | `http://%s.hamedata.com/app/neng/getDateInfo%s.php` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php` (HTTP, Port 80) | `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` |
| `getDateInfo`-Dateisuffix | `g_cloudGetDateInfoSuffixTable[type]` (`eu`, leer, `us`, …) | Konstante `eu` | dito |
| Regionstabelle (EEPROM 0x441) | `eu`, `www`, `us`, `eu-dev`, `eu-staging`; gültig 0–2 (BLE-Befehl 0x02 akzeptiert 0–4, aber `HTTP_GetUrlData` setzt 3/4 auf 0) | `eu`, `www`, `us`, `dev`, `staging`; gültig 0–4 | `g_cloudDomainUrlTable`, `HTTP_GetUrlData` |
| Servertyp beim Start laden | nur bei Bedarf | beim Start (auch LED-Flag, EEPROM 0x395) | `Config_LoadOrInit_DeviceSettings` |
| Geräte-Source-ID (Cloud) | `VenusE 3.0-<id>` | `VNSE3-0-<id>` | `Cloud_BuildDeviceSourceID` 08031DF8 → 0803235C |
| CH395-Keepalive IDLE | 10 000 ms | 60 000 ms (CNT 5, INTVL 3000 ms gleich) | `CH395_ConfigTcpKeepalive_Defaults` 08034C2C → 0803524C |
| Keepalive vor Modbus-Socket 502 | nein | ja | `CH395_Init_Modbus_TCP_Socket` 0804AEF0 → 0804B510 |
| CH395-Hang-Watchdog | Datensätze > 1, 1800 Ticks, `CH395_Reset_And_Reinit`, Log `[HTTP]ch395 reset!!!!` | Datensätze ≥ 5, 14 400 Ticks, **`System_Reboot()`**, Unterdrückungs-Flag | `HTTP_ClientTask_Dispatcher` 080196CC → 0801979C |
| HTTP-POST über FC41D | zwei blinde `vTaskDelay`, feste 400-Tick-Wartezeit, keine ERROR-Prüfung, NUL-terminiertes Senden | Tick-Schleife, ERROR / `+QIURC`, exakte Byte-Anzahl (`UART_TransmitBytes`) | `Quectel_HTTP_POST_SendAndVerify` 0800FBDC (gleiche Adresse) |
| `AT+QHTTPPOST`-Timeouts | `60,60`, außen 600 | `120,120`, außen 6000 | `HTTP_Task_StateMachine_Handler` 0801028C → 08010318 |
| FreeRTOS-Heap | 0x117F0 (70 KB) | 0x13FF0 (80 KB) | `prvHeapInit` 0804CFE0 → 0804D698 |
| Zählerabfrage (Shelly `EM.GetStatus`/`EM1.GetStatus`, andere) | bei jedem Aufruf | höchstens alle 1100 Ticks, Modem-Zähler (AT fail / no response / no reply / send OK) | `Modem_SendStatusTelemetry` 0802EBA8 → 0802F010, `Ethernet_SendStatusTelemetry` 0802ED44 → 0802F200 |
| UDP-Zählerantwort | blockierendes `vTaskDelay(g_meterActiveCtType * 1200)` nach einem gültigen Sample | Zähler `g_modemRecvOkCount` | `Meter_ParseUDPMeterInfo` 080294FC → 0802990C |
| Quectel-Modem-Byte-Assembler | kein Aufrufzähler, Statuspuffer 0x406 Byte | Zähler `g_atRxByteCounter` je Aufruf, Puffer 0x408 Byte (auch in `CH395_ExtractUdpData`, `Quectel_SendCmd_WaitResponse`) | `Quectel_AT_Recv_ByteAssembler` 08017900 → 080179A0 |
| Passive-Mode-UDP-Verbindung | verlangt nur `g_networkLinkUp` | verlangt zusätzlich `!g_ethLinkActive` | `Cloud_PassiveMode_Connection_StateMachine` 0801504C → 080150DC |
| YMODEM-Upgrade | keine Typprüfung | Dateiname → Typ (`OTA_ClassifyDeviceTypeTag`, neu 0804C0F8), Fehler −5 | `YModem_ReceiveFile` 080326AC → 08032C10, `YModem_FirmwareUpgrade_Handler` 0802F530 → 0802FA38 |
| AI-Strategie-Antwort (MQTT cd 0x3A) | ein Puffer, alle Einträge, keine Seiten | seitenweise, `MQTT_CtrlResponse_Transmit` (neu 08022520) | `AI_ScheduleQuery_Response_Send` 08022274 → 08022348 |
| Modbus-Schreiben | Register-46000-Zielauswahl (1234 / 2345 / 3456 / 4567): setzt `g_otaTargetDevice`, liefert Status 2; kein 47001 | dieselbe Auswahl ruft jetzt zusätzlich `CAN_Write_Category0x18(value)` (neu 0802F484) auf und liefert 0; neues Register **0xB799** (47001) springt in diesen Code. 0xAFDF und 0xAFE6 gibt es in beiden (nur anderer Dispatch) | `Write_Handler` 08053738 → 08054254 |
| MQTT `cd=0x42` „Get INV DEBUG info“ | existiert nicht | Unterwert `debug=N`: 1 = Wartungsmodus an (`g_maintModeState = 2`), 2 = `MQTT_Publish_WorkMode_Config` (neu 08025E18), 3 / 4 = `CAN_Write_Category0x18(1)` / `(0)` | `MQTT_CtrlCommand_Dispatcher` 08022830 → 08022A20 |
| MQTT `cd=0x13` Debug | – | Unterbefehl `debug` → `MQTT_ConnectionStatus_Report_Send` (neu 08022784: `ct_st, conn_st, err, wifi, send_ok, recv_ok, at_fail, no_resp, no_reply, sock, ip, port, reason`) | `MQTT_CtrlCommand_Dispatcher` |
| Shell-Befehle | 68 Befehle (inkl. `help`, `clear`), Tabelle 79 Einträge @ 0805ECD0 | 70 Befehle: neu `read_inv1` (0805049C; Wartungsmodus wie `cd=0x42,debug=1`), `read_inv2` (080504A8; gibt `g_vns_inv_debug` aus), Tabelle 81 Einträge @ 0805F9C4 | `shellCommandTable` |
| CT-Sollwert hängt positiv | – | Sollwert → 0, solange `g_epsReserveActiveFlag` | `CT_GridPower_Controller` 0802E488 → 0802E8D0 |
| EPS-Reserve-Hysterese | 3 Zustände (Leerlauf / eingerastet / deaktiviert), läuft bei SOC 0 %, schaltet EEPROM 0x300 über 8 % automatisch wieder ein | zustandslos, nur bei SOC ≠ 0, kein automatisches Wiedereinschalten | `Battery_EPS_Reserve_SOC_Hysteresis` 0800A714 (gleiche Adresse) |
| BLE EPS an/aus (0x0F), Entladegrenze (0x54); MQTT cd 0x0B, 0x38 | setzen `g_epsReserveState` zurück (0x54 / 0x38 auch `g_epsReserveLatched`) | nicht mehr | `BLE_App_Command_Dispatcher` 0800C0D4 → 0800C0E4, `MQTT_CtrlCommand_Dispatcher` |
| MQTT-Statusbericht | – | neues Feld `,df=%d` | `MQTT_StatusReport_BuildAndPublish` 08027D68 → 08028170 |
| Netz-Sync-Übernahme | Commit-Bedingung bei genau 2 abweichenden Phasen (`g_gridSyncDeviationCount == 2`) | ab 2 (`< 2` negiert), `g_gridSyncStatusCode = 2` in einem weiteren Zweig gesetzt | `Grid_PowerSync_StateMachine` 0802F090 → 0802F594 |
| Protocol-AA-Systemparameter (CAN-Register 0x11) | ein fester Wertepaar-Speicher (zwei Globals, 18 Byte Code) | Index aus `reg_id`, 7-Slot-Tabelle mit Grenzprüfung (56 Byte) | `Protocol_AA_SetSystemParams` 08031864 → 08031DA4 |
| Economy-TOU-Antwort (Prüfsumme) | eine XOR-Prüfsumme, ein Vergleich | XOR-Prüfsumme ein zweites Mal in eine eigene Variable berechnet, eine von beiden wird akzeptiert (mathematisch identisch); Fehlermeldung erhält `crc2` | `HTTP_Economy_TOU_ParseResponse` 080187D8 → 08018880 |
| LED-Status (SEL LED) | nur RAM | EEPROM 0x395 (BLE `Y`, MQTT, JSON-RPC, beim Start geladen) | BLE-/MQTT-/JSON-RPC-Dispatcher, `Config_LoadOrInit_DeviceSettings` |
| Device-Info-Reserviertwort | – | `g_deviceInfoReserved8aa = 0xA882` in `Config_LoadOrInit_DeviceSettings` initialisiert | Zweck unbekannt |
| Modbus 32102 | `float32` (2 Register) | `i16` (1 Register) | `g_modbusRegDescTable` |
| JSON-RPC- / MQTT-Grenzen | keine explizite Grenzprüfung vor dem Dispatch-Switch | explizite Prüfungen `method < 0x12`, `cd < 0x43` (keine funktionale Änderung beobachtet) | `MQTT_JSON_RPC_Dispatcher` 0801E58C → 0801E658, `MQTT_CtrlCommand_Dispatcher` |
| BLE-Switch | direkter Switch über das Befehlsbyte | umgerechneter Index `cmd − 2 < 0x5A` (gleiche Fälle) | `BLE_App_Command_Dispatcher` |
| BLE-Namensformat | `AT+QBLENAME=MST_VNSE3_%c%c%c%c` | `AT+QBLENAME=%s%c%c%c%c` mit `MST_VNSE3_` (gleiches Ergebnis) | `BLE_Set_Device_Name` 0800EBDC → 0800EBD0 |
| Telemetrie-Upload-JSON | – | gleiche Schlüssel (`pv/sk/mv/me/ma/fu` in beiden nicht vorhanden); Versionskonstante 0x96 → 0x97 | `Cloud_UploadTelemetryData` |
| Speicherkarte | `.data` 12 024 B, `.bss` endet 0x2001F5DF, `.xmc_bss` 113 732 B | `.data` 12 016 B, `.bss` endet 0x20021DF7, `.xmc_bss` 113 752 B | Scatter-Tabelle 0805ECA0 → 0805F994 |
| Image-Größe | 374 784 B (endet 0x0805FFFF) | 378 880 B (endet 0x08060FFF) | – |

## Kosmetische Unterschiede (keine Verhaltensänderung)

Byte-Unterschiede, die nur durch Relokation oder Registerzuteilung entstehen (Dekompilat identisch oder nur in lokalen Variablennamen verschieden):
`CH395_ExtractUdpData` (abgesehen von der Puffergröße 0x408), `Device_Network_Info_Init`, `Heap_Calloc`, `HTTPS_FetchUrlData`, `Meter_GetMeterIP_FromResponse`,
`MQTT_Publish_PowerSchedule`, `MQTT_Publish_WiFi_Version`, `Quectel_AT_QVERSION_Query`, `Quectel_AT_SendAndPollResponse`, `Quectel_GetDeviceIP`,
`Quectel_UDP_Socket_Open`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`, `Util_RollingHistory_ShiftAndAverage_1s`, `debug_printf`,
`BLE_Pending_Commands_Process` (nur das Label der Pending-Command-Tabelle).
`BLE_Set_Device_Name`: nur der Formatstring wurde aufgeteilt (siehe Tabelle). `SMR_TIC_SelfUse_PowerController`: Log-Zeilennummern (0x605 → 0x60C …) und eine
Variablenumbenennung; die neue, aber wirkungslose Grenzprüfung der D ist hier nicht als Verhaltensänderung sichtbar.

## Unverändert (ausdrücklich geprüft)

Modbus-Lesepfad (`FC03_Read_Handler` 080220E0, `Read_Serializer` 08052638), BLE-Fehlerhistorie 0x13 (`EventLog_*`), OTA-Slot-Diagnose (`OTA_Set_SlotStatus`
Codes 0x400–0x404), die `VNSEE3-0`-Typprüfungen (`OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`,
`ProcessFirmwareUpdateCommand`), die HTTP-OTA-Downloadkette (`HTTP_FindHeaderBodySeparator` → `strstr("\r\n\r\n") + 4`, `HTTPS_POST_ReceiveResponseData`,
`OTA_WriteDataToRingBuffer`, `HTTPS_POST_Request`), die 2500-W-Grenzen, die Task-Tabelle (14 Tasks), `g_mqttRpcMethodTable` (16 Methoden), die CAN-Dispatch-Tabellen,
die Modbus-Deskriptortabelle außer 32102, Task-Prioritäten und -Stacks, Hersteller-Default-WLAN-Zugangsdaten, Debug-Register 46000 (0x5100).

## Nur in v151 (7)

`CAN_Write_Category0x18` (0802F484), `MQTT_ConnectionStatus_Report_Send` (08022784), `MQTT_CtrlResponse_Transmit` (08022520),
`MQTT_Publish_WorkMode_Config` (08025E18), `OTA_ClassifyDeviceTypeTag` (0804C0F8), `ShellCmd_read_inv1` (0805049C), `ShellCmd_read_inv2` (080504A8).
v150 hat keine Funktion, die in v151 fehlt.

## Abweichungen der E3 von der Venus-D-Tabelle

Die Venus-D-Tabelle listet dieselben Änderungen mit diesen Abweichungen für die E3: kein Einspeisebegrenzer (die D-Zeile „PV-Startwert bei Einspeisung“ entfällt);
die E3 hat gar keinen PV/MPPT-Code; die E3-Zeile zur Passive-Mode-Bedingung und der Wert von `g_deviceInfoReserved8aa` (0xA882, D: 0xA892) sind E3-spezifisch; der E3-Heap wächst
wie auf der D von 70 auf 80 KB; `Meter_ParseUDPMeterInfo` und die Status-Telemetrie ändern sich genau wie auf der D (D-Zeile „CT-Kommunikationsfrequenz“).

## Offene Punkte

- `Write_Handler`: die skalaren Konstanten beider Funktionen wurden vollständig verglichen (alle Immediates ≥ 0x100); außer den oben genannten Punkten unterscheiden sich nur Switch-Basen (dieselben Register werden über unterschiedlich kompilierte Dispatch-Pfade erreicht) und
  Log-Zeilennummern. Ein vollständiger Zweig-für-Zweig-Äquivalenzbeweis der übrigen Zweige wurde nicht geführt (mittlere Sicherheit), wie bei der D.
- Genaue Bedeutung von `df=%d` und von `g_deviceInfoReserved8aa` (0xA882): aus dem Control-Code allein nicht ersichtlich.
- Ob `cd=0x42` / `read_inv1/2` auf einen Changelog-Punkt abgebildet werden oder eine nicht gelistete Ergänzung sind, ist offen (wie auf der D).
- Die E3-Micro-Empfänger der CAN-Kategorie 0x18 (Modbus 46000 / 47001, MQTT `cd=0x42`) wurden in diesem reinen Control-Vergleich nicht neu geprüft.
