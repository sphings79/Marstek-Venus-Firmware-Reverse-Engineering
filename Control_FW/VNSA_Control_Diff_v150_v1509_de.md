# Diff-Tabelle Control v150 → v150.9 (Venus A)

> 🇬🇧 English version: [VNSA_Control_Diff_v150_v1509.md](VNSA_Control_Diff_v150_v1509.md)

**Stand:** 2026-10-11. Beide Images mit korrekter Basis 0x08004800 in Ghidra, vollständig benannt (v150: 1935 Funktionen, v150.9: 1941).
**Hauptdokumentation:** [VNSA_Control_FW_v150_Vollanalyse_de.md](VNSA_Control_FW_v150_Vollanalyse_de.md), [VNSA_Control_FW_v1509_Vollanalyse_de.md](VNSA_Control_FW_v1509_Vollanalyse_de.md)
**Richtung:** v150 (Build 11. Aug 2026, Release) → Beta v150.8 (19. Aug, nicht analysiert) → Beta v150.9 (8. Sep). v150.9 ist eine Beta *nach*
v150 auf dem Weg zu v151 (das Archiv-Changelog führt sie als „v150.9 (Beta)“); die Tabelle zeigt also, was die Beta gegenüber dem Release v150 ergänzt.

Funktionspaare über gleiche Namen (1935), Instruktion für Instruktion verglichen mit Aufrufzielen über Namen, referenzierten Strings und
Immediate-Werten (Literal-Pool- und RAM-Adressen maskiert, weil sich das RAM-Layout unterscheidet): **42 geändert** — 24 mit geändertem Verhalten,
3 nur Version/Identität, 15 nur verschobene RAM-Offsets oder Log-Zeilennummern — **6 nur in v150.9**, **0 nur in v150**.

## Prüfliste (Pflichtprüfungen)

| Frage | Ergebnis |
|---|---|
| Modbus-Register | Deskriptortabelle: dieselben 246 Register, nur **32102** wechselt von `float32` (2 Register) zu `i16` (1 Register); Tabelle von 0x20000354 nach 0x2000034C verschoben. `Write_Handler`: neues Schreibregister **47001** (0xB799, CAN 0x18 über das neue `CAN_Write_Category0x18`), 45030 (0xAFE6) erhält einen eigenen Fall (gleiche Wirkung); Grenzen von 44002/44003 unverändert (44003 > 1500 W in beiden abgewiesen). Lesepfad unverändert. 30200 liest 150 → 1509. |
| BLE | `BLE_App_Command_Dispatcher`: Kmd `T` (0x54) setzt die EPS-Hysterese nicht mehr zurück; BLE-Name jetzt `AT+QBLENAME=%s%c%c%c%c` mit `MST_VNSA_` als Argument (`BLE_Set_Device_Name`, gleiches Ergebnis). Leistungsklassen/-grenzen unverändert. |
| AC-Einspeisegrenze | Unverändert 1500 W (alle Begrenzungen, 44003). `Grid_Export_Power_Limiter`: der PV-Startwert wird nicht mehr zuerst auf 2500 W begrenzt; obere Klemme 2400 W unverändert. |
| BLE-OTA | Weiterhin möglich, BLE-OTA-Pfad unverändert. Neu: YMODEM-Bin-Typ-Prüfung (`OTA_ClassifyDeviceTypeTag`, „Bin type mismatch! Upgrade aborted.“). |
| Cloud-Domains | Unverändert: Upload auf marstekcloud.com, `getDateInfo` und übrige Endpunkte auf hamedata.com, Servertypen 0–2. Geräte-Quell-ID `VenusA-<serial>` → `VNSA-0-<serial>`. |
| Pack-Kapazität | Unverändert: 7 Datensätze je Pack, 13 Zellen für Packs 1..6 geprüft. |
| Nicht funktionale Register | 47001 wird implementiert (sendet CAN 0x18, vom A-Micro ignoriert → bleibt NO_OP/FUNCTIONLESS); 32102 ändert den Typ. Keine weitere registersichtbare Änderung. |

## Änderungen v150 → v150.9 (Verhalten)

| Funktion (Adresse v150 → v150.9) | v150 | v150.9 |
|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` 0800AC1C → 0800AC1C | Automat mit drei Zuständen; unter 9 % SOC EPS aus (EEPROM 0x300 = 0) und über 8 % **automatisch wieder an** | Hysterese nur bei SOC ≠ 0, EPS-Flag wird nicht mehr automatisch wieder gesetzt |
| `BLE_App_Command_Dispatcher` 0800C5B8 → 0800C5C8, `MQTT_CtrlCommand_Dispatcher` 08023BEC → 08023D9C | BLE `T` / MQTT `cd=0x38` setzen den EPS-Hysteresezustand und `g_epsReserveActiveFlag` zurück; höchstes `cd` 0x41 | Rücksetzen entfernt; neu `cd=0x42` „Get INV DEBUG info“ (`MQTT_Publish_WorkMode_Config`, `g_vns_inv_debug`) und `cd=19`-Verbindungsbericht (`MQTT_ConnectionStatus_Report_Send`) |
| `CT_GridPower_Controller` 08030830 → 08030C10 | gleicher Code wie D v150 | Sollwert-Aktualisierung entfällt, solange die EPS-Reserve aktiv ist |
| `Config_LoadOrInit_DeviceSettings` 0802D670 → 0802DA34 | SEL-LED beim Start nicht geladen; Typcode 0x96 | SEL-LED aus EEPROM 0x395 geladen; Typcode 0x5E5 |
| `MQTT_JSON_RPC_Dispatcher` 0801F144 → 0801F21C | SEL-LED-Einstellung nur im RAM | zusätzlich in EEPROM 0x395 geschrieben |
| `MQTT_StatusReport_BuildAndPublish` 08029F90 → 0802A350 | Statusstring ohne `df` | neues Feld `,df=%d`, Energiefelder umsortiert |
| `Cloud_UploadTelemetryData` 0801B238 → 0801B23C | Nutzlast-Aufbau wie D v150 | neu geschriebener Nutzlast-Aufbau (581 → 700 Instruktionen) |
| `Cloud_PassiveMode_Connection_StateMachine` 08015A24 → 08015A28 | — | zusätzliche Prüfung des Ethernet-Links |
| `HTTP_ClientTask_Dispatcher` 0801A270 → 0801A294 | > 1 ungesendeter Datensatz, Old-Step 0, 1800 abgelaufen → Log `[HTTP]ch395 reset!!!!` + `System_Reboot` | ≥ 5 ungesendete Datensätze, 3600 abgelaufen → `System_Reboot` (ohne Log) |
| `Modem_SendStatusTelemetry` 08031028 → 08031418, `Ethernet_SendStatusTelemetry` 080311C4 → 08031608 | Zählerabfrage bei jedem Aufruf | höchstens alle 2000 Ticks; Modem-AT-Fehler-/OK-Zähler |
| `Meter_ParseUDPMeterInfo` 0802B83C → 0802BC00 | `vTaskDelay(ct_type × 1200)` nach geparster 3-Phasen-Antwort | Empfangszähler statt Verzögerung |
| `Quectel_AT_Recv_ByteAssembler` 0801846C → 08018484 | — | Zähler empfangener Bytes |
| `CH395_Init_Modbus_TCP_Socket` 0804D674 → 0804DC1C | kein Keepalive auf Socket 502 | `CH395_ConfigTcpKeepalive_Defaults` auch für Socket 502 (Leerlauf weiter 10 s) |
| `Grid_Export_Power_Limiter` 080233A8 → 08023488 | PV-Startwert auf 2500 W begrenzt, obere Klemme 2400 W | Startwert ohne 2500-W-Begrenzung, obere Klemme 2400 W |
| `Grid_PowerSync_StateMachine` 08031510 → 0803199C | gleicher Code wie D v150 | geänderte Übernahmebedingung, wenn zwei oder mehr Phasen abweichen |
| `Protocol_AA_SetSystemParams` 08033DD4 → 0803429C | übernimmt das empfangene Paar ohne Prüfung | Bereichsprüfung des Arbeitsmodus-Felds |
| `Write_Handler` 08055EB8 → 08056960 | kein 47001; 45030 über den Bereichspfad des 45000-Blocks | neu 47001 (CAN 0x18), eigener Fall für 45030 |
| `YModem_ReceiveFile` 08034E38 → 08035328, `YModem_FirmwareUpgrade_Handler` 080319B0 → 08031E40 | nur Größen-/CRC-Prüfung | Bin-Typ-Prüfung des Dateinamens (`OTA_ClassifyDeviceTypeTag`) |
| `prvHeapInit` 0804F764 → 0804FDA4 | Heap 0x11800 (70 KB) | 0x14000 (80 KB) |
| `BLE_Set_Device_Name` 0800F300 → 0800F2F8 | `AT+QBLENAME=MST_VNSA_%c%c%c%c` | `AT+QBLENAME=%s%c%c%c%c` + `MST_VNSA_` |
| `Cloud_BuildDeviceSourceID` 08034588 → 08034A74 | `VenusA-<serial>` | `VNSA-0-<serial>` |

## Änderungen v150 → v150.9 (nur Version/Identität)

`HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData` (`sv=` 150 → 1509), `ShellCmd_get_ver` (Build-Stempel, `ems_ver`).

## Änderungen v150 → v150.9 (nur verschobene RAM-Offsets / Log-Zeilennummern)

`BLE_Pending_Commands_Process`, `CH395_ExtractUdpData`, `Device_Network_Info_Init`, `MQTT_Publish_PowerSchedule`, `MQTT_Publish_WiFi_Version`,
`Meter_GetMeterIP_FromResponse`, `Quectel_AT_QVERSION_Query`, `Quectel_GetDeviceIP`, `Quectel_SendCmd_WaitResponse`, `Quectel_UDP_Socket_Open`,
`SMR_TIC_SelfUse_PowerController`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`, `Util_RollingHistory_ShiftAndAverage_1s`, `debug_printf`.

## Nur in v150.9

`OTA_ClassifyDeviceTypeTag`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18`,
`ShellCmd_read_inv1`, `ShellCmd_read_inv2` (Shell-Tabelle 79 → 81 Einträge).

## Release-Notizen

v150.9: „PV-Abbruch bei Überschusseinspeisung behoben, LED-Dauerblinken behoben, CT-Verbindung/Phasenerkennung verbessert“. Codeänderungen,
die zu diesen Punkten passen (Zuordnung nicht vom Hersteller bestätigt): Netzsynchronisation (`Grid_PowerSync_StateMachine`, Startwert von
`Grid_Export_Power_Limiter`), CT-/Zählerabfrage (Ratenbegrenzung in `Modem_/Ethernet_SendStatusTelemetry`, `Meter_ParseUDPMeterInfo`),
Speicherung der SEL-LED (`Config_LoadOrInit_DeviceSettings`, `MQTT_JSON_RPC_Dispatcher`). In der Notiz nicht erwähnt: Umbau der EPS-Reserve,
YMODEM-Bin-Typ-Prüfung, Wechselrichter-Debuginfo, 47001, Heap-Größe, Watchdog- und Keepalive-Änderungen.

## Strings (v150 → v150.9)

Neu: `VNSA-0 v1509`, `Build:\tSep  8 2026 09:34:19`, `AT+QBLENAME=%s%c%c%c%c`, `Bin type mismatch! …`, `Bin type unrecognized! …`, `nametpype =%d`,
`[MQTT] Type: %d, Get INV DEBUG info...`, `g_vns_inv_debug.*`, `read_inv1`, `read_inv2`, `cd=19,ct_st=%d,conn_st=%d,…`, `cd=%d,wm=%d,sl_f=%d,…`,
neue Statusbericht- und Upload-Formatstrings, `%s-%s`.
Entfernt: `VNSA-0 v150`, `AT+QBLENAME=MST_VNSA_%c%c%c%c`, `VenusA-%s`, `[HTTP]ch395 reset!!!!`, alte Statusbericht- und Upload-Formatstrings.
