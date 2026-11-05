# Diff-Tabelle Control v150.8 → v150.9 (Venus A)

> 🇬🇧 English version: [VNSA_Control_Diff_v1508_v1509.md](VNSA_Control_Diff_v1508_v1509.md)

**Stand:** 2026-11-05. Image `1508_control_VNSA-0_app_1508_0819_182932.bin` („A Control 1508“, Build-String `Aug 19 2026 18:29:09`) verglichen mit A Control 1509 (Build-String `Sep  8 2026 09:34:19`). Beide sind in Ghidra mit Basis 0x08004800 geladen (1508: 1935 Funktionen, 1509: 1941).
**Zugehörige Dokumente:** [VNSA_Control_FW_v1508_Vollanalyse_de.md](VNSA_Control_FW_v1508_Vollanalyse_de.md), [VNSA_Control_FW_v1509_Vollanalyse_de.md](VNSA_Control_FW_v1509_Vollanalyse_de.md), [VNSA_Control_Diff_v150_v1509_de.md](VNSA_Control_Diff_v150_v1509_de.md), [VNSA_Control_Diff_v1509_v151_de.md](VNSA_Control_Diff_v1509_v151_de.md)

## Kurzfazit

**Richtung:** 1508 ist der ältere Build (19. Aug 2026), 1509 der neuere (8. Sep 2026). „1508 hat X“ heißt: X ist in 1508 und nicht in 1509 (und umgekehrt).

- 1508 meldet sich anders als 1509: Gerätetyp-Strings `VEPRO-0` / `VenusE Pro` / BLE-Präfix `MARS_VEPRO_`, Versionskonstante 0x5E4 (1509: `VNSA-0` / `Venus A` / `MST_VNSA_`, 0x5E5). Das ist nur ein Fakt aus dem Code; die Bedeutung ist unklar (siehe „Geräte-Identität“).
- 1508 fehlen die Änderungen, die die frühere [Tabelle v150 → v150.9](VNSA_Control_Diff_v150_v1509_de.md) allein 1509 zuschreibt: die sechs Funktionen `OTA_ClassifyDeviceTypeTag`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18`, `ShellCmd_read_inv1`, `ShellCmd_read_inv2` existieren in 1508 nicht (Shell-Tabelle 1508: 79 Einträge, 1509: 81); ebenso fehlen die YMODEM-Gerätetyp-Prüfung, der MQTT-Befehlsblock mit `CAN_Write_Category0x18` und das Modbus-Schreibregister 47001.
- Weitere Unterschiede (EPS-Reserve-Hysterese, HTTP-Dispatcher, Grid-Export-Limiter, Telemetrie-Builder, ...) stehen in der Tabelle; wo nur ein Unterschied gesehen, aber die Bedeutung nicht geklärt wurde, steht dort „unklar“.
- Aussagen über Auswirkungen auf Hardware oder Betrieb werden nur gemacht, soweit sie aus dem Code folgen.

## Methodik

Anker: Nach Maskierung von Adressen und Aufrufoperanden sind **95,19 %** der 1508-Bytes (368.444 von 387.072) identisch zu 1509 (452 Blöcke). Über diese Anker werden Funktionen gepaart (1902 identische Paare, 20 Teilpaare, 19 der 1941 1509-Funktionen ohne direkten Partner; 16 Paare im umsortierten Bereich wurden per Instruktionsähnlichkeit geprüft und korrigiert). Danach wurden die geordneten Aufrufziele aller 1935 Paare verglichen: 21 Befunde, alle in ohnehin geänderten Funktionen, keiner unter den 1902 identischen Paaren (keine Fehlpaarung); die geänderten Funktionen wurden auf normalisierter Dekompilat-Ausgabe verglichen.
Kommentarköpfe: Die Kommentarköpfe der 1508-Funktionen wurden aus 1509 übernommen und widersprachen dem 1508-Code stellenweise (z. B. nannte `BLE_Set_Device_Name` `MST_VNSA_`). **Erledigt (2026-11-05):** 16 Plate-Kommentare wurden gegen die echten 1508-Strings korrigiert und lauten nun „In 1509: ...“ (`BLE_Set_Device_Name`, `BLE_Query_Device_Name`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `Cloud_Report_FillDeviceInfo`, `Config_LoadOrInit_DeviceSettings`, `Cloud_BuildDeviceSourceID`, `ShellCmd_get_ver`, `CLI_InitSession` und sechs RAM-Plates); die OTA-Modellprüfungen waren korrekt und blieben unverändert. Der Export wurde danach neu erzeugt. Maßgeblich bleibt der Code.

## Geräte-Identität (1508 vs 1509)

| Punkt | 1508 | 1509 |
|---|---|---|
| Build-String (`ShellCmd_get_ver`) | `Aug 19 2026 18:29:09` | `Sep  8 2026 09:34:19` |
| `ems_type` (`ShellCmd_get_ver`), MQTT-Topics (`MQTT_Topic_Builder`), UDP-Discovery-Antwort (`CH395_HandleBroadcastDiscovery`) | `VEPRO-0` | `VNSA-0` |
| Shell-Wurzeleintrag (`CLI_InitSession`) | `VEPRO-0 v1508` | `VNSA-0 v1509` |
| Versionskonstante (`g_deviceTypeCode`, `Config_LoadOrInit_DeviceSettings`; Cloud-Abfrage in `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher`) | 0x5E4 | 0x5E5 |
| Init `g_deviceModelTag` / `g_deviceTypeStr` | Worte 0x52504556 + 0x302D4F (ASCII `VEPRO-0`) | `VNSA` + `-0`; zusätzlich `g_deviceInfoReserved8aa = 0xA892`, `g_selLedEnable = 1` |
| Cloud-Gerätename (`Cloud_Report_FillDeviceInfo`) | `snprintf(out,0x14,fmt,"VenusE Pro")` | `memcpy(out,"Venus A",0x14)` |
| Cloud-Source-ID (`Cloud_BuildDeviceSourceID`) | `snprintf(buf,0x20,"%s-%s","VenusE Pro",mac)` | `sprintf(buf,"%s-%s","VNSA-0",mac)` |
| BLE-Namenspräfix (`BLE_Set_Device_Name`, `BLE_Query_Device_Name`) | `MARS_VEPRO_` | `MST_VNSA_` |
| OTA-Modellprüfung (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`) | Vergleich mit `"VNSA-0"` (Code identisch zu 1509) | `"VNSA-0"` |

Archivordner und Dateiname sagen `VNSA-0`, die 1508-Strings dagegen `VEPRO-0` / `VenusE Pro`. Dass „Venus E Pro“ gemeint ist, ist eine **Annahme** (nur Namensmuster); es gibt keinen Hardware-Beleg, und die Bedeutung der Identitätsänderung ist unklar. Hinweis: Die OTA-Modellprüfung in 1508 testet weiterhin auf `VNSA-0`, nicht auf den 1508-Gerätetyp-String.

## Änderungen 1508 vs 1509 (Funktionen)

Adressen: 1508-Einsprung (1509-Einsprung in Klammern, wenn abweichend). „unklar“ = Unterschied gesehen, Bedeutung nicht geklärt.

| Funktion | 1508-Adresse | Änderung 1508 vs 1509 | Beleg |
|---|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` | 0800AC1C | Logik umgebaut. 1508: Zustandsmaschine auf `g_epsReserveHysteresisState` (0/1/2). 1509: boolescher Latch `g_epsReserveLatched` mit `g_epsReserveActiveFlag`, CAN-Statuswerte 2/3, Schwellen `soc < g_dischargeCutoffSocRaw+5`, `soc < 9 && g_epsReserveEnable == 1`. Semantische Wirkung unklar. | Dekompilat-Diff (54 vs 45 Zeilen) |
| `BLE_App_Command_Dispatcher` | 0800C5BC (0800C5C8) | 1509 hat einen `EEPROM_Write`-Aufruf mehr (Aufruf 277 von 291). 1508-Switch mit Bereichsprüfung `cmd-2 < 0x5a` und umbasierten Case-Nummern kompiliert (Darstellung der Sprungtabelle; kein Unterschied im Befehlssatz gezeigt). 1508 schreibt `g_quectelHwResetBackoff_off4 = 0`, wo 1509 `g_netCmdActivityTick = 0` schreibt (gleicher Slot, Name aus RAM-Sync). | Aufrufzielliste, Dekompilat-Diff |
| `Quectel_AT_Recv_ByteAssembler` | 0801847C (08018484) | Puffer-Löschen `memset(&g_atUdpRxLen, 0x406)` (1508) vs `0x408` (1509); 1509 hat das Zusatzfeld `g_atRxByteCounter`, daher liegen die Assembler-Zustandsflags in 1508 bei +2/+3/+4 statt +4/+5/+6 (1509). Sonst gleiche Zustandsmaschine. | Dekompilat-Diff |
| `Quectel_SendCmd_WaitResponse`, `CH395_ExtractUdpData` | 0800B9D0, 0800BB48 | Nur dieselbe `memset`-Größe 0x406 vs 0x408 und umbenannte Globals. Code sonst identisch. | Dekompilat-Diff |
| `HTTP_ClientTask_Dispatcher` | 0801A284 (0801A294) | Stark umgebaut: 136 vs 364 Dekompilat-Zeilen, gleiche Callees in anderer Reihenfolge. 1508 hat keine `g_httpTimeSyncFastRetryFlag`-/`g_httpTimeSyncPollTick`-Schnellwiederholung und keinen `g_touReportAckPendingFlag`-Fall im verglichenen Teil; die Kein-Link-Reboot-Logik unterscheidet sich (1508: `telemetryHistoryRecordCount > 2 && oldStep == 0` und ein 0xE10-s-Timer; 1509: Prüfung `< 5` mit `g_httpNoLinkRebootTick`). Vollständigkeit des 1508-Dekompilats dieser Funktion nicht abschließend geklärt: unklar. | Dekompilat-Diff, Aufrufzielliste |
| `Cloud_UploadTelemetryData` (Rollenname) | 0801B220 (0801B23C) | Ältere, kürzere Form (581 vs 700 Dekompilat-Zeilen); gleiche Aufrufreihenfolge. Name nach Rolle (geringe Sicherheit). | Instruktionsähnlichkeit 0,334 |
| `Cloud_Report_FillDeviceInfo` | 08020F24 (08021018) | Gerätename `VenusE Pro` per `snprintf` statt `memcpy` von `Venus A`. | Dekompilat-Diff |
| `MQTT_JSON_RPC_Dispatcher` | 0801F130 (0801F21C) | Ein `EEPROM_Write`-Aufruf weniger in 1508; Guard `if (len < 0x12)` um die `ble_mac`-Behandlung in 1508; cJSON-Hook-Adressen unterscheiden sich (Relokation). | Aufrufzielliste, Dekompilat-Diff |
| `Grid_Export_Power_Limiter` | 080233A0 (08023488) | 1508 hat am Anfang keinen Aufruf `Power_ScaledValue_Calc_Negated`; begrenzt `power_hi` auf 0x9C4 (2500) und negiert `g_gridSamplePowerW` inline; 1509 nutzt den Helfer und hat an dieser Stelle keine 2500-Begrenzung. Obere Begrenzung 0x960 (2400) in beiden vorhanden. Wirkung auf das Einspeiselimit: unklar. | Aufrufzielliste, Dekompilat-Diff |
| `MQTT_CtrlCommand_Dispatcher` | 08023BE4 (08023D9C) | 1508 fehlen 13 Aufrufe: ein `strstr` + `MQTT_ConnectionStatus_Report_Send`, ein `EEPROM_Write` und ein Block mit `strstr`, `atoi`, 4x `MQTT_Publish_CommandResult`, `MQTT_Publish_WorkMode_Config`, 2x `CAN_Write_Category0x18` (ein Befehl, der nur in 1509 existiert). Die Callees `MQTT_ConnectionStatus_Report_Send`, `MQTT_Publish_WorkMode_Config`, `CAN_Write_Category0x18` haben in 1508 keine Entsprechung. | Aufrufzielliste |
| `MQTT_StatusReport_BuildAndPublish` | 08029F88 (0802A350) | Gleiche Größe (716 Instruktionen); lokales Variablenlayout und teils Reihenfolge unterscheiden sich (Ratio 0,57 dekompiliert, 0,84 nach Instruktionen). Inhaltliche Änderung: unklar. | Dekompilat-Diff |
| `DeviceInfo_BuildStatusString` | 0802AE1C (0802B1E8) | 1509 baut die Hardware-Varianten-Tag-Strings je Variante (HME-4, HME-3, SMR-0/1/2, TPM2-0); 1508 weicht in diesem Teil ab (Details unklar, Ratio 0,81 / 0,46 dekompiliert). | Dekompilat-Diff |
| `Meter_ParseUDPMeterInfo` | 0802B838 (0802BC00) | 1508 fügt `vTaskDelay((short)g_meterActiveCtType * 0x4b0)` hinzu und lässt `g_modemRecvOkCount++` weg. | Aufrufzielliste, Dekompilat-Diff |
| `Config_LoadOrInit_DeviceSettings` | 0802D66C (0802DA34) | Ein `EEPROM_Read` weniger (das Lesen von `g_selLedEnable` bei 0x395; 12 vs 13 Lesevorgänge in der verketteten Bedingung); Identität wie in „Geräte-Identität“. | Aufrufzielliste |
| `CT_GridPower_Controller` | 08030828 (08030C10) | 1508 fehlt ein Aufruf `Inverter_Power_Setpoint_Calc(1,0)` (der `g_epsReserveActiveFlag`-Zweig). | Aufrufzielliste, Dekompilat-Diff |
| `Modem_SendStatusTelemetry`, `Ethernet_SendStatusTelemetry` (Rollennamen) | 08031020, 080311BC | 1509 beginnt mit `FreeRTOS_xTaskGetTickCount`, 1508 nicht (je ein Aufruf weniger); ältere Form. | Aufrufzielliste |
| `Grid_PowerSync_StateMachine` | 08031508 (0803199C) | Variablennamen/Layout unterscheiden sich (`interval_ticks` vs `complete_timeout_s`, `last_deviation_channel`); algorithmischer Unterschied nicht belegt: unklar. | Dekompilat-Diff |
| `YModem_FirmwareUpgrade_Handler` | 080319A8 (08031E40) | 1508 fehlt ein `printf` (der Zweig „Bin type mismatch! Upgrade aborted“ für Ergebnis 0xFFFFFFFB). | Aufrufzielliste, Dekompilat-Diff |
| `Protocol_AA_SetSystemParams` (Rollenname) | 08033DCC (0803429C) | In 1508 deutlich kürzer (9 vs 24 Zeilen). | Instruktionsähnlichkeit 0,424 |
| `YModem_ReceiveFile` | 08034E3C (08035328) | 1508 fehlen `OTA_ClassifyDeviceTypeTag`, 3x `printf` und 2x `YModem_SendByte` (die Gerätetyp-Prüfung der empfangenen Datei gibt es nur in 1509). | Aufrufzielliste, Dekompilat-Diff |
| `CH395_Init_Modbus_TCP_Socket` | 0804D674 (0804DC1C) | 1508 ruft `CH395_ConfigTcpKeepalive_Defaults()` nicht auf. | Aufrufzielliste |
| `Write_Handler` | 08055EBC (08056960) | 1508 fehlt ein `CAN_Write_Category0x18`-Aufruf; im 1508-Dekompilat kein Literal 0xB799 (47001) und kein eigener Fall 0xAFE6 (45030) (in 1509 vorhanden); CAN-Zeitplan-Registercache als `g_canScheduleRegCache` / `...ValueHi` adressiert (+4-Offset-Form in 1509). | Aufrufzielliste, Dekompilat-Diff |
| `Cloud_PassiveMode_Connection_StateMachine` | 08015A3C (08015A28) | 1508 lässt die Vorbedingung `!g_ethLinkActive` weg und nutzt `g_cloudPassiveModeRetryCnt` als rohe Variable; Wirkung: unklar. | Dekompilat-Diff |
| `Quectel_AT_QVERSION_Query` | 0800FF58 (08010210) | Zeiger auf den erwarteten Antwort-String unterscheidet sich (Relokation); Kopie der Versionsworte gleich. | Dekompilat-Diff |
| `SMR_TIC_SelfUse_PowerController` | 0800851C | Nur die Log-Zeilennummern-Argumente unterscheiden sich (0x607/0x60C/0x616 vs 0x609/0x60E/0x618): Quellzeilen um 2 verschoben. Keine Logikänderung gesehen. | Dekompilat-Diff |
| `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData` | 080123A8, 0801985C | Versionskonstante 0x5E4 vs 0x5E5; Pufferadressen verschoben. | Dekompilat-Diff |
| `ShellCmd_get_ver` | 0803B0E4 | Build-String, `ems_type` `VEPRO-0`, Version 0x5E4 (siehe „Geräte-Identität“). | Dekompilat-Diff |
| `Cloud_BuildDeviceSourceID` | 08034580 (08034A74) | `snprintf(...,0x20,"%s-%s","VenusE Pro",mac)` vs `sprintf(...,"%s-%s","VNSA-0",mac)`. | Dekompilat-Diff, Aufrufzielliste |

Zusätzlich aus den Strings des 1508-Exports erkannt (nicht Teil der ersten Vergleichsliste, daher noch nicht Funktion für Funktion gegen 1509 verifiziert): `BLE_Set_Device_Name` / `BLE_Query_Device_Name` verwenden `MARS_VEPRO_` (1509: `MST_VNSA_`), `MQTT_Topic_Builder` und `CH395_HandleBroadcastDiscovery` verwenden `VEPRO-0` (1509: `VNSA-0`), `CLI_InitSession` den Wurzeleintrag `VEPRO-0 v1508`.

Reine Dekompilat-Darstellung oder RAM-Relokation, keine Codeänderung belegt: `Device_Network_Info_Init`, `BLE_Pending_Commands_Process`, `Util_RollingHistory_ShiftAndAverage_1s`, `Meter_GetMeterIP_FromResponse`, `MQTT_Publish_WiFi_Version`, `Board_GetUniqueID_HexString`, `debug_printf`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`.

### Nur in 1509

`OTA_ClassifyDeviceTypeTag`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18`, `ShellCmd_read_inv1`, `ShellCmd_read_inv2`. Nur in 1508: keine.

## Prüf-Checkliste (Standardprüfungen)

| Frage | Ergebnis 1508 vs 1509 |
|---|---|
| Modbus-Register | Die Deskriptortabelle (`RS485_Modbus_RegisterMap_Init`, `FC03_Read_Handler`) zeigt nur Darstellungsunterschiede (Typisierung der Globals); ein Register-für-Register-Vergleich des Tabelleninhalts wurde nicht durchgeführt: nicht geprüft. `Write_Handler`: 1508 hat kein Schreibregister 47001 (0xB799, CAN 0x18 über `CAN_Write_Category0x18`, 1509) und keinen eigenen Fall für 45030 (0xAFE6); der Text zum 44003-Limit (Ablehnung über 1500 W) ist identisch. |
| BLE | `BLE_Set_Device_Name` / `BLE_Query_Device_Name`: Präfix `MARS_VEPRO_` (1508) statt `MST_VNSA_` (1509), gleiches Format `AT+QBLENAME=%s%c%c%c%c`. `BLE_App_Command_Dispatcher`: ein `EEPROM_Write` weniger; Befehlssatz sonst nicht als abweichend gezeigt. Leistungsklassen/Limits: nicht geprüft. |
| AC-Einspeiselimit | Im Text von `Write_Handler` unverändert 1500 W (44003); `Grid_Export_Power_Limiter`: 1508 begrenzt den Startwert auf 2500 W, obere Begrenzung 2400 W in beiden. Effektives Einspeiselimit durch die 1508-Änderungen: unklar. |
| BLE-OTA | Die BLE-OTA-Funktionen (`BLE_Cmd_OTA_*`, `BLE_OTA_WriteDataChunk`) sind in 1508 vorhanden; der Paarvergleich fand dort keinen Unterschied, BLE-OTA ist also nach dem Code weiterhin möglich (nicht am Gerät getestet). YMODEM: 1508 fehlt die Bin-Typ-Prüfung (`OTA_ClassifyDeviceTypeTag`, „Bin type mismatch!“). Die OTA-Annahme prüft in 1508 den Modell-String weiterhin gegen `VNSA-0` (`OTA_Is_VNSA_Model`, Code identisch zu 1509). |
| Cloud-Domains | Gleiche Menge in beiden: `marstekcloud.com/data-upload/v1/venus/`, `hamedata.com`-Endpunkte (`getDateInfo`, `getDevicesApply`, `backup-configs`, `schedulings/policies`, FC41D-`.rbl`-OTA-URL). Geändert: Gerätename/Source-ID (`VenusE Pro` statt `Venus A` / `VNSA-0`), Version 0x5E4 vs 0x5E5 in der Cloud-Abfrage. |
| Pack-Kapazität | Unverändert: `BMS_CellVoltage_MinMax_PerPack` scannt in beiden 6 Packs (`< 6`) und 13 Zellen (`< 0xd`); `BLE_Pack_RegisterRecords` und `BLE_Build_BMS_Data_Response` zeigen nach Normalisierung keinen Unterschied. Die BMS-seitige Grenze (`CAN_TX_PerPack_10Msgs`) gehört nicht zum Control-Image: nicht geprüft. |
| Nicht funktionale Register | Statusvergleich auf Registerebene: nicht geprüft (47001 und 45030 existieren nur in 1509, siehe oben). |

## Offene Punkte

- Bedeutung der Identitäts-Strings `VEPRO-0` / `VenusE Pro` / `MARS_VEPRO_`: unklar; „Venus E Pro“ ist eine Annahme ohne Hardware-Beleg. Bedarf Bestätigung durch den Reporter oder Herstellerdokumentation.
- Wirkungen des EPS-Hysterese-Umbaus, der `Grid_Export_Power_Limiter`-Begrenzung, der HTTP-Dispatcher-Umstrukturierung und der Telemetrie-Builder sind nicht belegt (als „unklar“ markiert).
- Die 1508-Sprungtabellen-Labels von `HTTP_ClientTask_Dispatcher` wurden rekonstruiert (Basis gegenüber 1509 um 0x14 verschoben); über die 7 Einträge hinaus wurden sie nicht erneut gegen die Sprungtabelle des Images geprüft.
- Der Vergleich 1508 gegen das Release v150 ist nicht Teil dieses Dokuments: nicht geprüft.
- Die Paare `Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry` und `Protocol_AA_SetSystemParams` sind nach Rolle benannt (geringe Sicherheit).
- 111 RAM-Namen der Form `<object>_off<hex>` in 1508 sind nur Adressrelationen, keine Semantik.
