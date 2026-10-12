# Diff-Tabelle Control v150 → v151 (Venus A)

> 🇬🇧 English version: [VNSA_Control_Diff_v150_v151.md](VNSA_Control_Diff_v150_v151.md)

**Stand:** 2026-10-11. **Hauptdokumentation:** [VNSA_Control_FW_v150_Vollanalyse_de.md](VNSA_Control_FW_v150_Vollanalyse_de.md), [VNSA_Control_FW_v151_Vollanalyse_de.md](VNSA_Control_FW_v151_Vollanalyse_de.md)
Versionsfolge des Venus-A-Controls: v150 (`app_0150_0811`, 11. Aug 2026) → Beta v150.8 → Beta v150.9 (8. Sep 2026) → **v151** (28. Sep 2026).
Zwischenschritte: [VNSA_Control_Diff_v150_v1509_de.md](VNSA_Control_Diff_v150_v1509_de.md) und [VNSA_Control_Diff_v1509_v151_de.md](VNSA_Control_Diff_v1509_v151_de.md).

**Diese Fassung ersetzt den früheren abgeleiteten Diff.** Die frühere Fassung musste ohne ein korrekt basiertes A-v150-Programm auskommen (maskierte Byte-
Segmente von v150 gegen v151, altes Verhalten aus Venus D v150 → v151). A v150 ist inzwischen vollständig in Ghidra analysiert (1935 Funktionen, alle Gates
bestanden); diese Tabelle ist daher **namensgepaart und vollständig**: Funktionspaare über identische Namen (1935), Vergleich Instruktion für Instruktion mit
Aufrufzielen nach Namen, referenzierten Strings und Immediate-Werten (Literal-Pool- und RAM-Adressen maskiert, das RAM-Layout unterscheidet sich). Altes und neues
Verhalten sind beide am A-Code verifiziert. Korrekturen gegenüber der abgeleiteten Fassung stehen am Ende.

Ergebnis: **48 geändert** (25 verhaltensrelevant, 4 nur Version/Identität — `CLI_InitSession` ist bis auf den String unverändert — und 19 nur verschobene RAM-Offsets/Logzeilennummern),
**7 nur in v151**, **0 nur in v150**.

## Prüfliste

| Frage | Ergebnis |
|---|---|
| Modbus-Register | Deskriptortabelle (246 Einträge) identisch außer **32102**: Typ `float32` (2 Register) → `i16` (1 Register); Tabelle verschoben 0x20000354 → 0x2000034C. `Write_Handler`: neues Schreibregister **47001** (0xB799, CAN 0x18), eigener Fall für 45030 (0xAFE6, gleiche Wirkung wie zuvor); 44002/44003 unverändert (44003 über 1500 W wird in v150 und v151 abgelehnt; 44002 bis 2500 W, EEPROM-Kopie auf 1500 W begrenzt). Lesepfad unverändert. Alles seit v150.9. |
| BLE | `BLE_App_Command_Dispatcher`: Befehl `T` setzt die EPS-Hysterese nicht mehr zurück; Namensformat `AT+QBLENAME=%s%c%c%c%c` statt des festen `MST_VNSA_%c%c%c%c` (gleicher Name). Seit v150.9. |
| AC-Einspeiselimit | 1500 W in v150 und v151 (alle Begrenzungen); `Grid_Export_Power_Limiter`: PV-Startwert wird nicht mehr zuerst auf 2500 W begrenzt (v150.9); obere Grenze 2400 W unverändert. |
| BLE-OTA | Weiterhin möglich; YMODEM-Bin-Typ-Prüfung seit v150.9 (`OTA_ClassifyDeviceTypeTag`, „Bin type mismatch! Upgrade aborted.“). |
| Cloud-Domains | `getDateInfo`: hamedata.com → `api-<region>.marstekcloud.com`, Servertypen 0–2 → 0–4 (v151); Upload in beiden auf marstekcloud.com; Geräte-Quell-ID `VenusA-<serial>` → `VNSA-0-<serial>` (v150.9). |
| Pack-Kapazität | Unverändert: 7 Pack-Datensätze, 13-Zellen-Scan für Packs 1..6 (`BMS_CellVoltage_MinMax_PerPack` identisch). |
| Nicht funktionale Register | 47001 wird implementiert (CAN 0x18, vom A-Micro ignoriert); 32102 ändert den Typ; keine weitere registersichtbare Änderung. |

## Änderungen v150 → v151 (Verhalten)

Spalte „Seit“: v150.9 = bereits in Beta v150.9 enthalten (Funktion in v150.9 und v151 in diesem Punkt identisch); v151 = neu in v151. RN = Punkt der v151-Release-Notes.

| Funktion (Adresse v150 → v151) | v150 | v151 | Seit | RN |
|---|---|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` 0800AC1C → 0800AC1C | Drei-Zustands-Automat; unter 9 % SOC EPS aus (EEPROM 0x300 = 0), über 8 % automatisch wieder an | Hysterese nur bei SOC ≠ 0, EPS-Flag wird nicht mehr automatisch wieder aktiviert | v150.9 | 2 |
| `BLE_App_Command_Dispatcher` 0800C5B8 → 0800C5C8, `MQTT_CtrlCommand_Dispatcher` 08023BEC → 08023E90 | BLE `T` / MQTT `cd=0x38` setzen die EPS-Hysterese und `g_epsReserveActiveFlag` zurück; höchstes `cd` 0x41 | Reset entfernt; `cd=0x42` „Get INV DEBUG info“ (neu `MQTT_Publish_WorkMode_Config` 080275B8, `g_vns_inv_debug`), `cd=19` Verbindungsbericht (neu `MQTT_ConnectionStatus_Report_Send` 08023BF4) | v150.9 | 2, 4 |
| `CT_GridPower_Controller` 08030830 → 08030D20 | gleicher Code wie D v150 | Sollwert-Update wird übersprungen, solange die EPS-Reserve aktiv ist (v150.9); positiver CT-Sollwert auf 0 zurückgesetzt (v151) | v150.9 + v151 | 2 |
| `MQTT_StatusReport_BuildAndPublish` 08029F90 → 0802A444 | Statusstring ohne `df` | neues Feld `,df=%d` | v150.9 | 2 |
| `YModem_ReceiveFile` 08034E38 → 08035448, `YModem_FirmwareUpgrade_Handler` 080319B0 → 08031F60, neu `OTA_ClassifyDeviceTypeTag` 0804E924 | nur Größen-/CRC-Prüfung | Bin-Typ-Prüfung des Dateinamens, Abbruch mit „Bin type mismatch! Upgrade aborted.“ | v150.9 | 3 |
| `AI_ScheduleQuery_Response_Send` 08023630 → 080237B8, neu `MQTT_CtrlResponse_Transmit` 08023990 | eine 1024-Byte-Antwort `AI%d=…` | seitenweise Antwort (`cd=%d,pg=%d/%d,`, `AI%u=…`) | v151 | 4 |
| `ShellCmd_read_inv1` 08052CC4, `ShellCmd_read_inv2` 08052CD0 (neu) | — | Shell-Befehle `read_inv1` (Wartungsmodus) und `read_inv2` (gibt `g_vns_inv_debug` aus) | v150.9 | – |
| `HTTP_Cloud_Reporting_Dispatcher` 08012394 → 08012424, `HTTP_GetUrlData` 0801984C → 08019910 | `getDateInfo` auf hamedata.com, Servertypen 0–2 | `api-<region>.marstekcloud.com`, Servertypen 0–4, Präfixe `dev`/`staging` | v151 | 5 |
| `Config_LoadOrInit_DeviceSettings` 0802D670 → 0802DB28 | SEL-LED (0x395) und Servertyp (0x441) werden beim Boot nicht geladen | SEL-LED geladen (v150.9); Servertyp geladen, Erststart-Default 0x395 = 1 (v151) | v150.9 + v151 | 5 |
| `MQTT_JSON_RPC_Dispatcher` 0801F144 → 0801F2D0 | SEL-LED-Einstellung nur im RAM | zusätzlich in EEPROM 0x395 geschrieben | v150.9 | – |
| `Quectel_HTTP_POST_SendAndVerify` 08010438, `HTTP_Task_StateMachine_Handler` 08010AC0 → 08010B4C, `HTTPS_FetchUrlData` 0801BB88 → 0801BD1C, `HTTP_Economy_TOU_ParseResponse` 0801936C → 08019418 | `AT+QHTTPPOST` 60/60 s, äußeres Timeout 600, eine TOU-Prüfsumme | HTTP-POST neu geschrieben, 120/120 s, äußeres Timeout 6000, TOU `crc2` | v151 | 6 |
| `Cloud_UploadTelemetryData` 0801B238 → 0801B300 | Payload-Builder von D v150 | neu geschriebener Payload-Builder (v150.9), Body-Längenbehandlung wie D v151 (v151) | v150.9 + v151 | 6 |
| `prvHeapInit` 0804F764 → 0804FEC4 | Heap 70 KB | 80 KB | v150.9 | 6 |
| `Quectel_AT_Recv_ByteAssembler` 0801846C → 08018510 | — | Zähler der empfangenen Bytes | v150.9 | – |
| `CH395_ConfigTcpKeepalive_Defaults` 080373BC → 08037A84 | Keepalive-Idle 10 s | 60 s | v151 | 1 |
| `CH395_Init_Modbus_TCP_Socket` 0804D674 → 0804DD3C | kein Keepalive auf Modbus-Socket 502 | Keepalive-Defaults auch für Socket 502 | v150.9 | 1 |
| `HTTP_ClientTask_Dispatcher` 0801A270 → 0801A344 | > 1 ungesendeter Datensatz, alter Schritt 0, 1800 abgelaufen → Log `[HTTP]ch395 reset!!!!` + `System_Reboot` | ≥ 5 Datensätze und 3600 (v150.9); 14 400 und Unterdrückungs-Flag `g_httpWatchdogSuppressFlag` (v151); immer `System_Reboot` | v150.9 + v151 | 1 |
| `Modem_SendStatusTelemetry` 08031028 → 08031538, `Ethernet_SendStatusTelemetry` 080311C4 → 08031728 | Zählerabfrage bei jedem Aufruf | höchstens alle 2000 Ticks (v150.9), 1100 Ticks (v151) | v150.9 + v151 | 7 |
| `Meter_ParseUDPMeterInfo` 0802B83C → 0802BCF4 | blockierendes `vTaskDelay(ct_type × 1200)` nach einer UDP-Zählerantwort | Zähler statt der Verzögerung | v150.9 | 7 |
| `Grid_PowerSync_StateMachine` 08031510 → 08031ABC | gleicher Code wie D v150 | geänderte Commit-Bedingung bei zwei oder mehr abweichenden Phasen | v150.9 | – |
| `Write_Handler` 08055EB8 → 08056A7C, neu `CAN_Write_Category0x18` 080319AC | kein 47001; 45030 über den 45000-Grenzpfad | Schreibregister 47001 (0xB799), eigener Fall für 45030 | v150.9 | – |
| `Grid_Export_Power_Limiter` 080233A8 → 0802353C | PV-Startwert auf 2500 W begrenzt; obere Grenze 2400 W | Startwert ohne die 2500-W-Begrenzung; obere Grenze 2400 W unverändert | v150.9 | – |
| `Protocol_AA_SetSystemParams` 08033DD4 → 080343BC | keine Bereichsprüfung | Bereichsprüfung des Betriebsmodus-Feldes | v150.9 | – |
| `BLE_Set_Device_Name` 0800F300 → 0800F2F8 | `AT+QBLENAME=MST_VNSA_%c%c%c%c` | `AT+QBLENAME=%s%c%c%c%c` mit `MST_VNSA_` | v150.9 | – |
| `Cloud_BuildDeviceSourceID` 08034588 → 08034B94 | `VenusA-<serial>` | `VNSA-0-<serial>` | v150.9 | – |
| Modbus-Deskriptor 32102 | `float32` | `i16` | v150.9 | – |

**Nur Version/Identität:** `ShellCmd_get_ver`, Versionswort 0x96 → 0x97 (auch in `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`,
`Config_LoadOrInit_DeviceSettings`, `Cloud_UploadTelemetryData`).
**Nur verschobene RAM-Offsets / Logzeilennummern:** `SMR_TIC_SelfUse_PowerController`, `Device_Network_Info_Init`, `Quectel_SendCmd_WaitResponse`,
`CH395_ExtractUdpData`, `Quectel_AT_QVERSION_Query`, `BLE_Pending_Commands_Process`, `Cloud_PassiveMode_Connection_StateMachine` (plus die
Ethernet-Link-Prüfung von v150.9), `Quectel_GetDeviceIP`, `Util_RollingHistory_ShiftAndAverage_1s`, `Meter_GetMeterIP_FromResponse`,
`MQTT_Publish_WiFi_Version`, `MQTT_Publish_PowerSchedule`, `Quectel_UDP_Socket_Open`, `debug_printf`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`.

## Korrekturen gegenüber der früheren abgeleiteten Fassung

| Punkt | Abgeleitete Fassung sagte | Am A v150 verifiziert |
|---|---|---|
| Verbindungs-Watchdog | in v151 eingeführt („14 400 + `System_Reboot`“) | A v150 bootet bereits neu (`System_Reboot`) nach 1800 bei > 1 ungesendetem Datensatz und loggt `[HTTP]ch395 reset!!!!` (Venus D v150 setzt nur den CH395 zurück); v150.9: ≥ 5 Datensätze und 3600; v151: 14 400 + Unterdrückungs-Flag |
| Zähler-Abfrageperiode | neu in v151 („höchstens alle 1100 Ticks“) | Ratenbegrenzung in v150.9 eingeführt (2000 Ticks); v151 verkürzt auf 1100 |
| `Cloud_UploadTelemetryData` | nur in v151 geändert | Payload-Builder bereits in v150.9 neu geschrieben, v151 ergänzt die Body-Längenbehandlung |
| `Config_LoadOrInit_DeviceSettings` | Typ und SEL-LED beim Boot in v151 geladen | SEL-LED bereits in v150.9 geladen; Servertyp und Erststart-Default 0x395 in v151 |
| `CT_GridPower_Controller` | positiver Sollwert in v151 zurückgesetzt | zusätzlich überspringt v150.9 bereits Sollwert-Updates, solange die EPS-Reserve aktiv ist |
| `Heap_Calloc`, `Quectel_AT_SendAndPollResponse` | als geändert gelistet | im Code unverändert (nur Aufrufoffsets/Adressen) |
| `MQTT_Transport_NullCallback`, `Util_NoOp_StartupHook` | „nur geändert, weil per Bytes nicht auffindbar“ | unverändert (namensgepaart) |
| Funktionszahlen | 58 geändert, 20 verschoben, 4 wahrscheinlich unverändert (Byte-Segmente, je v151-Funktion) | 48 geändert, 7 neu, 0 entfernt (namensgepaart) |
| `Protocol_AA_SetSystemParams`, `MQTT_JSON_RPC_Dispatcher` | nicht gelistet / nur „kleine Byte-Änderungen“ | verhaltensrelevant: Bereichsprüfung ergänzt; SEL-LED in EEPROM 0x395 persistiert (beide v150.9) |

Alle übrigen Aussagen der abgeleiteten Fassung (EPS-Hysterese, Statusfeld `df`, YMODEM-Bin-Typ-Prüfung, AI-Zeitplan-Paginierung, `cd=0x42`,
`read_inv1/2`, Umzug von `getDateInfo`, HTTP-POST-Neuschreibung, Heap-Größe, Keepalive, 47001/0xAFE6, Startwert des Einspeisebegrenzers, BLE-Namensformat,
Quell-ID, 32102) wurden am A-v150-Code bestätigt.

## Strings (v150 → v151)

Neu: `VNSA-0 v151`, marstekcloud-`getDateInfo`-URLs, `AT+QHTTPPOST=%d,120,120`, `crc2`-Fehlertext, `cd=%d,pg=%d/%d,`, `AI%u=…`,
`Bin type mismatch! Upgrade aborted.`, `Bin type mismatch! file=%s expect=%d got=%d`, `Bin type unrecognized! file=%s`,
`[MQTT] Type: %d, Get INV DEBUG info...`, `g_vns_inv_debug.*`-Feldnamen, `read_inv1`, `read_inv2`,
`cd=19,ct_st=%d,conn_st=%d,…`, `AT+QBLENAME=%s%c%c%c%c`, `,df=%d` (Statusbericht).
Entfernt: hamedata-`getDateInfo`-URLs, `AT+QHTTPPOST=%d,60,60`, alter Prüfsummen-Fehlertext, `AI%d=…`, `VenusA-%s`, `AT+QBLENAME=MST_VNSA_%c%c%c%c`,
`[HTTP]ch395 reset!!!!`.
