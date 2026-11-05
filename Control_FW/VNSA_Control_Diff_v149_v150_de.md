# Diff-Tabelle Control v149 → v150 (Venus A)

> 🇬🇧 English version: [VNSA_Control_Diff_v149_v150.md](VNSA_Control_Diff_v149_v150.md)

**Stand:** 2026-11-05. Image `149_control_VNSA-0_app_0149_0528_102448.bin` („A Control 149“, Build-String `May 28 2026 10:23:00`) verglichen mit dem Release A Control 150 (`150_control_VNSA-0_app_0150_0811_114724.bin`, Build-String `Aug 11 2026 11:46:45`). Beide sind in Ghidra mit Basis 0x08004800 geladen (149: 1913 Funktionen, 150: 1935).
**Zugehörige Dokumente:** [VNSA_Control_FW_v149_Vollanalyse_de.md](VNSA_Control_FW_v149_Vollanalyse_de.md), [VNSA_Control_FW_v150_Vollanalyse_de.md](VNSA_Control_FW_v150_Vollanalyse_de.md), [VNSA_Control_Diff_v150_v1509_de.md](VNSA_Control_Diff_v150_v1509_de.md), [VNSA_Control_Diff_v1508_v1509_de.md](VNSA_Control_Diff_v1508_v1509_de.md)

## Kurzfazit

**Richtung:** 149 ist die ältere Build (28. Mai 2026), 150 die neuere (11. Aug 2026). „149 hat X“ heißt: X steht in 149 und nicht in 150; „150 hat X“ heißt: X steht in 150 und nicht in 149. Die Tabellen beschreiben 149, wie es ist, und nennen zum Vergleich das Verhalten von 150.

- **Gleiche Codebasis, deutlich älter.** Nach Maskierung von Adressen und Aufrufoperanden sind 92,24 % der 149-Bytes identisch mit 150 (150 selbst hat 387.072 B, 149 hat 382.976 B). 149 hat 1913 Funktionen, 150 hat 1935; **23 Funktionen gibt es nur in 150**, **1 nur in 149** (`Base64Url_Encode`).
- **149 meldet sich wie 150 als Venus A:** `VNSA-0`, `Venus A`, BLE-Präfix `MST_VNSA_`; nur die Versionskonstante unterscheidet sich (0x95 = 149 in 149, 0x96 = 150 in 150) zusammen mit dem Build-String. In 149 gibt es keine `VEPRO`-/`VenusE`-Strings (anders als in der späteren Beta 1508).
- **BMS-Packsatz:** 0x58 Bytes pro Pack in 149, 0x60 Bytes in 150. 150 ergänzt die Felder SoH und MinPackVoltage; der Telemetrie-Historienslot ist in 149 0x35b Bytes groß, in 150 0x35d.
- **Kein Power-Clamp-Modus in 149:** der Zweig `g_powerClampConfig == 1`, `Config_Read/Write_PowerClampMode_0x394`, die `Config_Write_PowerOffset`-Aufrufe in den Befehls-Dispatchern und der Netzleistungs-Offset gibt es nur in 150.
- **Kein HTTPS-Upload in 149:** Der Cloud-Upload ist die ältere HTTP-Form (AES-128-ECB + Base64Url-Payload über `HTTPS_POST_Request` an `hamedata.com`, `setVenusDReporting`). 150 sendet eine JSON-Payload an `https://api-<region>.marstekcloud.com` über eigene HTTPS-/TLS-Client-Funktionen (`HTTPS_FetchUrlData`, `HTTPS_TLSSessionInit`, `mbedTLS_SSL_Recv/Send_WithRetry`, ...), die in 149 alle fehlen.
- **Kein No-Link-Reboot-Watchdog** in `HTTP_ClientTask_Dispatcher` (149); **7 MQTT-SSL-Konfigurationsschritte ohne SNI** (150: zusätzlicher letzter Schritt `Quectel_SSL_ConfigSNI(1,0,1)`); die **SMR_TIC-Self-Use-Zustandsmaschine ist inline** in `CT_PowerSetpoint_Compute` (150: eigene Funktion `SMR_TIC_SelfUse_PowerController`).
- Wo nur ein Unterschied gesehen, seine Bedeutung aber nicht geklärt wurde, steht in der Tabelle „unklar“. Über Auswirkungen auf Hardware oder Betrieb wird nichts behauptet, was nicht aus dem Code folgt.

## Methodik

Anker: Nach Maskierung von Adressen und Aufrufoperanden sind **92,24 %** der 149-Bytes (353.276 von 382.976) identisch mit 150 (491 Blöcke; beim Vergleich 1508 ← 1509 waren es 95,19 %). Von den 1935 Funktionen von 150 ergaben die Anker **1844 identische Paare, 46 teilweise Paare und 45 ohne 149-Gegenstück** (1890 Paare); nach manuellen und instruktionsähnlichkeitsbasierten Korrekturen (umsortierter Bereich um 0x0800F900–0x08010600, verwaiste Coderuns, die zu Funktionen wurden) ergibt sich der Endstand von 1912 Paaren plus der ungepaarten `Base64Url_Encode`.
Aufrufzielprüfung: Die geordneten Aufrufziele (`bl`, `blx`, `b.w`) jedes Paares wurden verglichen, wobei die 150-Ziele über die Paarliste abgebildet wurden. Erster Lauf: 21 Befunde. **Drei davon deckten Paarungs-/Benennungsfehler des Ports selbst auf** (der Instruktionsähnlichkeitstest allein sieht sie nicht):

| 149-Adresse | War | Ist | Beleg |
|---|---|---|---|
| 08006a1c | `Timer_Callback_BLE_Modem_Supervisor` (522 B, zwei Funktionen verschmolzen) | getrennt: `Base64Url_Encode` 08006a1c–08006b53 (Rollenname, niedrige Konfidenz, kein 150-Partner) + `Timer_Callback_BLE_Modem_Supervisor` 08006b58–08006c25 (Paar 150 08006a1c, Ähnlichkeit 1,0, 80 vs. 80 Instruktionen) | Größe 206 vs. 522 B; der Base64-Alphabet-String in einem Timer-Callback; Callee von `Cloud_UploadTelemetryData` |
| 0801973c | `CH395_SPI_SendData_Verified` (152 B, zwei Funktionen verschmolzen) | getrennt: `HTTP_ParseUploadResponseCode` 0801973c–08019773 (Paar 150 0801bdec, Ähnlichkeit 0,96; Schlüssel `"de":` statt `"code":`) + zwei Strings (08019774, 0801977c; waren als Code disassembliert) + `CH395_SPI_SendData_Verified` 080197a4–080197d3 (Paar 150 08019f40, Ähnlichkeit 1,0) | Dekompilat zeigte `strstr`/`atoi` und den Logtext „Upload decrypt result“; Aufrufer `HTTP_ClientTask_Dispatcher` |
| 080145cc | `Quectel_SSL_ConfigSNI` (falscher Name) | `Quectel_SSL_ConfigVerifyLevel` (Paar 150 08014d90, Ähnlichkeit 1,0; String `AT+QSSLCFG="verify"`); 149 hat kein `Quectel_SSL_ConfigSNI` | Stringvergleich; Aufrufer Zustand 0 von `Quectel_MQTT_Config_StateMachine` |

Nach diesen Korrekturen bleiben 19 Befunde (13 Aufrufzahl-Unterschiede, 6 vertauschte Callee-Reihenfolgen), **alle in ohnehin geänderten Funktionen**. Drei weitere reine Lese-Prüfungen (String-Literale je Paar, Funktionsgröße je Paar, ASCII-Folgen im Code über das ganze Image) fanden keinen weiteren Fall. Gelernte Regel: Gleiche Einsprungadresse in 149 und 150 ist Zufall, nie ein Beleg (der Bereich ist um bis zu 0x7F4 Bytes umsortiert).
Die geänderten Funktionen wurden anschließend auf normalisierter Dekompilat-Ausgabe verglichen (Casts, `&` und untypisierte RAM-Namen ignoriert). Dieser Vergleich lief, bevor die RAM-Datentypen angewendet wurden (Paket P3b); viele rohe Dekompilat-Unterschiede waren daher Typisierungsrauschen und wurden herausgefiltert. Es wurden 43 Paare verglichen, 41 Plate-Kommentare „Differs from 150: …“ markieren die Funktionen im Programm.
Die übernommenen 150-Kommentare widersprachen dem 149-Code an 10 Stellen (Build-String, Version, `v149`, EMS-Version, ein Watchdog-Satz, MQTT-Konfigurationszustände, POST-Timeouts); sie wurden anhand der echten 149-Strings korrigiert (`ShellCmd_get_ver`, `Config_LoadOrInit_DeviceSettings`, `CLI_InitSession`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `prvHeapInit`, `HTTP_ClientTask_Dispatcher`, `Quectel_MQTT_Config_StateMachine`, `HTTP_Task_StateMachine_Handler`). Für 149 korrekte Kommentare blieben unverändert (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`, `BLE_Set_Device_Name`, `Cloud_BuildDeviceSourceID`, `Cloud_Report_FillDeviceInfo`).

## Geräte-Identität (149 vs. 150)

| Punkt | 149 | 150 |
|---|---|---|
| Build-String (`ShellCmd_get_ver`, String @ 0803a550) | `May 28 2026 10:23:00` | `Aug 11 2026 11:46:45` |
| Versionswort @ 08010000, gefolgt von `VNSA-0`; Footer `VNSA` @ 08061fe0 | 0x95 (149) | 0x96 (150) |
| `ems_type` / `ems_ver` (`ShellCmd_get_ver`) | `VNSA-0` / 0x95 (149) | `VNSA-0` / 150 (0x96) |
| `g_deviceTypeCode` (`Config_LoadOrInit_DeviceSettings`); Cloud-Abfrage in `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | 0x95 | 0x96 |
| Shell-Wurzeleintrag (`CLI_InitSession`) | `VNSA-0 v149` (08050924, 0805a8f5) | `VNSA-0 v150` |
| `VNSA-0` weitere Vorkommen | 08008cb0, 0800927c, 080094cc, 08009c8c, 08036cfc (UDP-Discovery-Antwort), 0802cb40 (`Config_LoadOrInit_DeviceSettings`) | gleiche Stellen |
| BLE-Namenspräfix (`BLE_Set_Device_Name`) | `MST_VNSA_` (0800ee58, 0800eef0) | `MST_VNSA_` |
| Cloud-Gerätename (`Cloud_Report_FillDeviceInfo`) | `Venus A` (080204c4) | `Venus A` |
| Cloud-Quell-ID (`Cloud_BuildDeviceSourceID`) | `VenusA-%s` (080339ac) | `VenusA-%s` |
| `VEPRO`-/`VenusE`-Strings | keine | keine |
| Cloud-Endpunkte | `http://<region>.hamedata.com/...`: `getDateInfo`, `backup-configs`, `ems/getDevicesApply`, `schedulings/policies` (https), `prod/api/v1/setVenusDReporting`; weitere Servertypen mit `http://%s/api/v1/data`, `/v1/data`, `/v1/json`; **kein `marstekcloud.com`** | `hamedata.com`-Endpunkte wie zuvor; Upload sendet an `https://api-<region>.marstekcloud.com` |
| MQTT-Topics | `marstek_energy/%s/device/%s/ctrl`, `marstek/%s/server|device/%s/ctrl` | gleich |

## Änderungen 149 vs. 150 (Funktionen)

Adressen: 149-Einsprung (150-Einsprung in Klammern, wo abweichend). „unklar“ = Unterschied gesehen, Bedeutung nicht geklärt.

| Funktion | 149-Adresse (150) | Änderung 149 vs. 150 | Beleg |
|---|---|---|---|
| `Timer_Callback_BLE_Modem_Supervisor` | 08006b58 (08006a1c) | Keine; nur die Paarung wurde korrigiert (siehe Methodik). | Ähnlichkeit 1,0 |
| `Telemetry_History_Record_Push`, `Telemetry_History_Buffer_Init` | 080102f0 (08010990), 0800ff3c | Historienslot-Schrittweite 0x35b (150: 0x35d), Payload 0x34f (150: 0x351); die Überlauf-Slot-Felder von 150 sind in 149 `_off<hex>`-Namen. | Dekompilat |
| `Quectel_MQTT_Config_StateMachine` | 08012928 (08013064) | 7 Schritte (0..6): Verify-Level, Cipher-Suite, SSL-Version, MQTT-SSL, MQTT-Version, Datentyp, Keepalive. 150 hat einen zusätzlichen letzten Schritt `Quectel_SSL_ConfigSNI(1,0,1)`. | Dekompilat, Aufrufe |
| `BLE_Build_BMS_Data_Response` | 0800f66c (0800fb90) | Der SoH-Slot wird mit 0 geschrieben (150: `g_bmsPack0Soh`); kein Update von `g_bmsSelfCheckFlag`. | Dekompilat-Diff |
| `BLE_Build_Cmd0x42_Response` | 080146ac | Tabellenwerte werden aus anderen BMS-Satz-Offsets gelesen (Satz-Layout, nächste Zeile); Wirkung unklar. | Dekompilat |
| BMS-Packsatz-Layout | viele | Packsatz 0x58 Bytes in 149 (0x2c Halbworte), 0x60 in 150 (0x30 Halbworte); 150 ergänzt die Felder SoH und MinPackVoltage. Sichtbar in `BMS_CellVoltage_MinMax_PerPack`, `ShellCmd_get_all_bms_data` (keine Zeilen `batsoh` / `batminvolt`; 2 Aufrufe weniger), `CAN_Detect_Mismatched_Nodes` (ebenfalls kein Mismatch-Zähler), `BMS_Version_Consistency_Check`, `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version`, `CAN_AA_StoreUnitFieldPair0/8/16/24_Notify/64/72/80`, `CAN_AA_StoreUnitIndexedFieldPair`. | Schrittweiten in den Dekompilaten |
| `Inverter_PowerSetpoint_DeadbandClamp`, `Remote_Power_Setpoint_Process` | 08017e50, 0802d1cc | Kein Zweig `g_powerClampConfig == 1`: Power-Clamp-Modus und Netzleistungs-Offset gibt es nur in 150 (ebenso `Config_Read/Write_PowerClampMode_0x394`, die `Config_Write_PowerOffset`-Aufrufe in den beiden Dispatchern und in `Config_LoadOrInit_DeviceSettings`). | Dekompilat, Aufrufe |
| `HTTP_FindHeaderBodySeparator` | 080185e0 (08018dcc) | Sucht `Connection:` und überspringt 0x15 Bytes; 150 sucht CRLFCRLF und überspringt 4. Logtext weicht ab („get bin_start error“). | Strings, Dekompilat |
| `Cloud_HTTP_Response_Parser` | 08027e30 (08028bf8) | Keine Behandlung des Schlüssels `ct_dev=`, kein `Config_Write_String_0x387`, kein Standard-Shelly-Port 0x3f2; 6 Aufrufe weniger. | Aufrufe, Dekompilat |
| `DeviceInfo_BuildStatusString` | 08029fdc (0802ae24) | Kein `Config_Read_String_0x387` für das Hardware-Varianten-Tag; Puffer-Clear 0x2c (150: 0x32); die HME-/SMR-/TPM2-Tags werden durch Wortschreibzugriffe gebaut; Rest unklar. | Dekompilat |
| `Meter_ParseUDPMeterInfo`, `Modem_SendStatusTelemetry`, `Ethernet_SendStatusTelemetry` | 0802a87c, 080303bc, 08030594 | Zusätzliche `g_meterSourceType`-Kette (3, 4, 8, 9, 10, 11), die `g_meterDataValidFlag` setzt; Callee-Reihenfolge vertauscht (`Meter_ParseActivePower` / `MPPT_ParseATResponse`; `sprintf` / `DeviceInfo_BuildStatusString`). `Modem_`/`Ethernet_SendStatusTelemetry` sind ältere Formen (Rollennamen). Wirkung unklar. | Aufrufe, Dekompilat |
| `Grid_PowerSync_StateMachine` | 0803091c | Die Bedingungsmengen zu `g_meterSourceType` weichen ab; Wirkung unklar. | Dekompilat |
| `Cloud_EdgeDetectAndWatchdog` | 08030c8c | Kein Qualifier `(g_mpptActiveChannel == 0 or 5)` an der Prüfung `g_ctLinkEdgeState == 3`. | Dekompilat |
| `Config_LoadOrInit_DeviceSettings` | 0802c6ec | Typcode 0x95 (150: 0x96); EEPROM-Konfigblock 0x300 Größe 0x85 (150: 0x98), zweiter Block 0x36bf (150: 0x3700); kein Default für `g_powerClampConfig`; ein EEPROM-Lesezugriff weniger. | Dekompilat |
| `Quectel_HTTP_AT_SendAndVerify` | 08010010 (08010594) | Ein `vTaskDelay` weniger; Log-Präfixe `[Rx..]` / `[Tx..]` ohne „Http“. | Aufrufe, Strings |
| `HTTP_Task_StateMachine_Handler` | 08010420 (08010ac0) | Ältere Form: kein `Quectel_HTTP_POST_SendAndVerify`, `Quectel_Modem_HardwareReset_Handler`, `Tick_Timer_Expired`; 2 `sprintf` und 2 `memset` weniger; stattdessen `Quectel_HTTP_GET_Request`, `HTTP_ParseUploadResponseCode`, `MQTT_Publish_Telemetry`, 2x `strstr`; keine `AT+QHTTPPOST`-/Header-Strings. | Aufrufe, Strings |
| `HTTP_ClientTask_Dispatcher` | 0801995c (0801a270) | Kein No-Link-Reboot-Watchdog (`System_Reboot`, `Tick_Timer_Check_Elapsed` fehlen); Slot-Schrittweite 0x35b; der Test des Upload-Ergebnisses ist invertiert; keine Eco-Tracker-Logstrings. | Aufrufe, Dekompilat |
| `Cloud_UploadTelemetryData` (Rollenname) | 0801a8d0 (0801b238) | Ältere Form: `AES128_ECB_Encrypt` + `Base64Url_Encode`-Payload über `HTTPS_POST_Request` an `hamedata.com` `setVenusDReporting`; 150: `HTTPS_FetchUrlData`, `marstekcloud.com`, JSON-Payload. | Aufrufe, Strings; Ähnlichkeit 0,28 |
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | 08019048, 08011c60 | Versionskonstante 0x95 (150: 0x96); zusätzliche Endpunkte `http://%s/api/v1/data` usw. | Strings |
| `mbedTLS_SSL_Connection_Init` | 0801c438 (0801cf50) | Ein `MQTT_TLS_Context_Reset`-Aufruf mehr; keine Logstrings für Fehler beim Laden von CA / Client-Zertifikat / privatem Schlüssel. | Aufrufe, Strings |
| `MQTT_CtrlCommand_Dispatcher` | 080230c8 (08023bec) | 20 Aufrufe weniger: `MQTT_Publish_CommandResult` x5, `Log_Printf` x6, `strstr` x2, `atoi` x2, `Config_Write_PowerClampMode_0x394` x2, `Config_Write_PowerOffset` x2, `MQTT_Publish_Grid_Status`. | Aufrufe |
| `BLE_App_Command_Dispatcher` | 0800c1b4 (0800c5b8) | 8 Aufrufe weniger (`Config_Write_PowerOffset` x2, PowerClampMode x2, `String_0x387`, `BLE_Send_Response`, `Log_Printf`, `memcpy`); Länge der Gerätestatus-Antwort 0xa4 (150: 0xa7), BMS-Info-Antwort 0x50 (150: 0x51); kein Befehl „Get new power data“; Befehl 0x16 ohne untere Längengrenze. | Aufrufe, Dekompilat |
| `CT_PowerSetpoint_Compute` (Rollenname, niedrige Konfidenz) | 0802fa5c (08030a74) | Die SMR_TIC-Zustandsmaschine ist inline (150: eigene Funktion `SMR_TIC_SelfUse_PowerController`); 16 Aufrufe mehr. Gleicher Aufrufer `CT_GridPower_Controller`, gleiche Parameter (`ct_power`, Modus); die sechs SMR_TIC-Variablen sind nach der 150-Zustandsmaschine benannt (`g_smrTicNewPowerW` usw.). | Aufrufe, Strings; Ähnlichkeit 0,57 |
| `ShellCmd_get_ver`, `CLI_InitSession` | 0803a4f4, 08050894 | Build `May 28 2026 10:23:00`, `ems_ver` 149 (0x95), `VNSA-0 v149`. | Strings |
| `HTTP_ParseData`, `HTTP_ParserOnBody`, `HTTP_ParserOnMessageComplete` | | Keine Logikänderung (nur Callback-Adressen / bool vs. int). | Dekompilat |
| `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `MQTT_Publish_BMS_Data` | | Gleiche Formatstrings; die übrigen Unterschiede sind Typisierungsrauschen, Wirkung unklar. | Dekompilat |

### Nur in 150 (23 Funktionen, kein 149-Partner)

`CH395CMDGetSocketStatus`, `SMR_TIC_SelfUse_PowerController`, `Config_Read_String_0x387`, `Config_Write_String_0x387`, `Config_Read_PowerClampMode_0x394`, `Config_Write_PowerClampMode_0x394`, `Quectel_HTTP_POST_SendAndVerify`, `Quectel_SSL_ConfigSNI`, `HTTPS_TLS_Context_Reset`, `mbedTLS_SSL_Recv_WithRetry`, `mbedTLS_SSL_Send_WithRetry`, `HTTPS_FetchUrlData`, `CH395_TCP_Socket_Open_HTTPS`, `MQTT_Publish_Grid_Status`, `HTTPS_PackPostRequest`, `HTTPS_ParseData`, `ShellCmd_FreeRtos_mem`, `CAN_AA_StoreUnitFieldPair88`, `HTTPS_TLSSessionInit`, `mbedtls_ssl_close_notify`, `HTTPS_ParserOnBody`, `FreeRTOS_xPortGetFreeHeapSize`, `FreeRTOS_xPortGetMinimumEverFreeHeapSize`.

### Nur in 149

`Base64Url_Encode` (Rollenname, niedrige Konfidenz; 150 lässt den Base64-Alphabet-String unreferenziert).

Nicht „nur in 150“, obwohl eine frühere Arbeitsnotiz das sagte: `Quectel_SSL_ConfigVerifyLevel`, `HTTP_ParseUploadResponseCode` und `CH395_SPI_SendData_Verified` gibt es in 149 (siehe Methodik); `CAN_Detect_Mismatched_Nodes`, `BMS_Version_Consistency_Check`, `Quectel_HTTP_ReadResponse_Trigger` und die acht `mbedtls_ssl_conf_*`-Setter (authmode, ca_chain, ciphersuites, curves, max_version, min_version, rng, sig_hashes; Namen nur nach Position und Größe, niedrige Konfidenz) gibt es ebenfalls in 149.

## Prüfliste (Standard-Prüfungen)

| Frage | Ergebnis 149 vs. 150 |
|---|---|
| Modbus-Register | `Write_Handler` ist ein identisches Paar (weder Schreibregister 47001 / 0xB799 noch eigener Fall 45030 / 0xAFE6 in einem von beiden; der Text zu 44003, Ablehnung über 1500 W bei Venus A, ist gleich). `FC03_Read_Handler` und `RS485_Modbus_RegisterMap_Init` sind identische Paare (nach Maskierung von Adressen und Aufrufoperanden byte-identisch), und die Registerdeskriptor-Tabelle `g_modbusRegDescTable` hat in 149 und 150 gleichen Typ und gleiche Größe (246 x 12 B). **Nicht geprüft:** der Tabelleninhalt Register für Register und die Statusklassifikation der Register (funktional / nicht funktional). |
| BLE | Namenspräfix `MST_VNSA_` in beiden. `BLE_App_Command_Dispatcher` weicht ab: 8 Aufrufe weniger (die Power-Offset- und Power-Clamp-Schreibzugriffe, der `String_0x387`-Schreibzugriff), Gerätestatus-Antwort 0xa4 (150: 0xa7), BMS-Info-Antwort 0x50 (150: 0x51), kein Befehl „Get new power data“, Befehl 0x16 ohne untere Längengrenze. `BLE_Build_BMS_Data_Response`: SoH-Slot wird mit 0 geschrieben. **Nicht geprüft:** Leistungsklassen und Limits. |
| AC-Einspeiselimit | `Grid_Export_Power_Limiter` ist ein identisches Paar: Startwert auf 2500 W (0x9c4) begrenzt, obere Grenze 2400 W (0x960) in beiden; `Write_Handler`-Text zu 44003 identisch (1500 W). Der Power-Clamp-Modus / Netzleistungs-Offset (nur 150) ändert den Sollwertpfad; Wirkung auf das effektive Einspeiselimit: **unklar**. |
| Update-Pfad (BLE-OTA) | Die BLE-OTA-Funktionen (`BLE_Cmd_OTA_Validate`, `BLE_Cmd_OTA_Init`, `BLE_Cmd_OTA_WriteSetup`, `BLE_OTA_WriteDataChunk`) gibt es in 149; der Paarvergleich fand keinen Unterschied, BLE-OTA ist also soweit der Code zeigt möglich (**nicht am Gerät getestet**). `OTA_Is_VNSA_Model` und `OTA_Slot_Config_Validate` prüfen auf `VNSA-0`. HTTP-Download-Pfad: `HTTP_FindHeaderBodySeparator` weicht ab (149 sucht `Connection:` und überspringt 0x15 Bytes; 150 sucht CRLFCRLF und überspringt 4), siehe die geräteübergreifende Referenz [OTA_HTTP_Body_Boundary_und_Diagnose_de.md](OTA_HTTP_Body_Boundary_und_Diagnose_de.md); Folgen für den OTA-Download von 149: **nicht geprüft**. `YModem_ReceiveFile` ist ein identisches Paar. |
| Cloud-Endpunkte | 149: nur `hamedata.com`-Endpunkte (siehe „Geräte-Identität“), Upload über `setVenusDReporting` mit AES-128-ECB + Base64Url-Payload; kein `marstekcloud.com`. 150: `marstekcloud.com`-Upload mit JSON-Payload über einen HTTPS-Client. Details der Cloud-TLS-Konfiguration: nicht geprüft. |
| Pack-Kapazität | Control-Seite unverändert: `BMS_CellVoltage_MinMax_PerPack` scannt in beiden 6 Packs (`< 6`) und 13 Zellen (`< 0xd`); nur die Packsatz-Schrittweite unterscheidet sich (0x58 vs. 0x60 Bytes, 150 ergänzt SoH und MinPackVoltage). Das BMS-seitige Limit (`CAN_TX_PerPack_10Msgs`) ist nicht Teil des Control-Images: **nicht geprüft**. |
| Nicht funktionale Register (Nachprüfung) | Statusvergleich auf Registerebene: **nicht geprüft**. |

## Offene Punkte

- Die Wirkungen der `g_meterSourceType`-Änderungen, `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `BLE_Build_Cmd0x42_Response` und des fehlenden Power-Clamp-Modus auf dem Sollwertpfad sind nicht geklärt (als „unklar“ markiert).
- `Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry`, `CT_PowerSetpoint_Compute` und `Base64Url_Encode` sind per Rolle benannt (niedrige Konfidenz); ob 150 einen Base64-Encoder enthält (sein Alphabet-String ist unreferenziert), ist nicht geklärt.
- Das BMS-seitige Pack-Limit, die Registertabelle Eintrag für Eintrag, die Cloud-TLS-Konfiguration und der Status nicht funktionaler Register sind nicht Teil dieses Vergleichs.
- 72 RAM-Plates von 150, die 150-Adressen enthalten, wurden bewusst nicht nach 149 übernommen; etwa 190 RAM-Namen der Form `<object>_off<hex>` oder Layout-abgeleitete Namen haben niedrige Konfidenz (nur Adressbezug). Siehe [VNSA_Control_FW_v149_Vollanalyse_de.md](VNSA_Control_FW_v149_Vollanalyse_de.md), §9.
- Der Vergleich von 149 mit den späteren Betas (1508, 1509) und mit v151 ist nicht Teil dieses Dokuments: nicht geprüft.
