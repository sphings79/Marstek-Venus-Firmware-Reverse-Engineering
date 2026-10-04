# Diff-Tabelle Control v149.2 → v150 (Venus D)

> 🇬🇧 English version: [VNSD_Control_Diff_v1492_v150_de.md](VNSD_Control_Diff_v1492_v150_de.md)

**Stand:** 2026-09-30. Beide Images mit korrekter Basis 0x08004800 in Ghidra, 100 % benannt.
Adressen: v149.2 → v150. Funktionspaare über gleiche Namen (1934), instruktionsweise verglichen:
**56 geändert** (davon **26 inhaltlich**), 1 nur in v149.2, 20 nur in v150.
**Englisch:** `VNSD_Control_Diff_v1492_v150_de.md` · **Vorgänger-Diff:** `VNSD_Control_Diff_v149_v1492_de.md`

## Einordnung

v149.2 (Beta, 02.07.2026) und v150 (05.08.2026) liegen nur 5 Wochen auseinander, unterscheiden sich im
Code aber deutlich mehr als der kleine Versionssprung vermuten lässt: v150 bringt einen komplett neuen
HTTPS/TLS-Client (mbedTLS über CH395), baut den Cloud-Telemetrie-Upload grundlegend um (JSON statt
AES+Base64URL, neue Domain), führt die „Peak-Shaving"/Leistungs-Klemm-Funktion ein (EEPROM 0x394, neue
BLE-/MQTT-Kommandos) und behebt mehrere Ethernet-/Modem-Stabilitätsprobleme über zwei neue Watchdogs.
Für v150 liegt ein Hersteller-Changelog vor (`CHANGELOG.de.md`, lokal, nicht im Repo); er wird unten Punkt für Punkt dem Code
zugeordnet.

**Korrektur gegenüber der Vorfassung dieses Dokuments:** Die alte Fassung beruhte auf einer
unvollständigen Funktionspaarung, fand nur 9 unterschiedliche Funktionen und schloss daraus fälschlich
„kein Changelog vorhanden, keine Modbus-/BLE-Änderung". Tatsächlich existiert ein Changelog mit 5
Punkten, es gibt 20 neue und 1 entfernte Funktion sowie 26 inhaltliche Änderungen. Zwei der drei zuvor
als „echt" gelisteten Änderungen waren zudem falsch interpretiert (siehe „Kosmetisch" unten).

## Prüfliste

| Frage | Ergebnis |
|---|---|
| Modbus-Register | **Registertabelle selbst unverändert** (alle 246 Deskriptoren: Registernummer, Typ, Skalierung, Anzahl identisch zwischen v149.2 und v150 — nur bei 215 Einträgen verschiebt sich die RAM-Schattenadresse, weil ein Config-Struct um 6 Byte wuchs, siehe Peak-Shaving unten). Das Schedule-EEPROM der Register ab 46500 zieht aber von 0x36BF nach 0x3700 um (`Config_LoadOrInit_DeviceSettings`, `Write_Handler`) |
| BLE | **erweitert**: neues Kommando `)` (0x29) „Set Peak shaving" (Klemm-Modus + Watt-Wert, EEPROM 0x394); `BLE_Build_DeviceStatus_Response` meldet zusätzlich Klemm-Modus + -Wert (Antwortlänge 0xA4→0xA7 Byte); `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` melden zusätzlich BMS-Selbsttest- und MOS-Flag (0x50→0x51 Byte); Fallthrough-Bug Kommando `Y`→`Z` behoben (fehlendes `break`) |
| AC-Einspeisung 2500 W | **Hartgrenze unverändert** (`Config_ClampDefaults_ParamBlock`, `Grid_Export_Power_Limiter` weiterhin byte-identisch). Unabhängig davon neu: konfigurierbarer Peak-Shaving-Klemmwert (EEPROM 0x394), wirkt zusätzlich in `Inverter_PowerSetpoint_DeadbandClamp` und `Remote_Power_Setpoint_Process` |
| BLE-OTA | **weiterhin möglich** (OTA/YMODEM/`BLE_Cmd_OTA_*` komplett byte-identisch, keine dieser Funktionen in der Änderungsliste) |
| Cloud-Domains | `hamedata.com`-Endpunkte selbst **unverändert** (identische Strings in beiden Versionen). Der Telemetrie-Upload (`Cloud_UploadTelemetryData`) wechselt aber von `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (AES+Base64URL-kodierter Query-Parameter) auf `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (Klartext-JSON-POST-Body über den neuen HTTPS-Client) |

## Neue Funktionen (20)

**HTTPS/TLS-Client für den Cloud-Upload** (CH395 + mbedTLS, neu, parallel zum bestehenden Klartext-Pfad):
- `HTTPS_FetchUrlData` – orchestriert einen kompletten HTTPS-POST: Pfad/Host aus URL, `HTTPS_PackPostRequest`,
  ggf. DNS, `CH395_TCP_Socket_Open_HTTPS` (Port 443), `CH395CMDGetSocketStatus`-Poll,
  `HTTPS_TLSSessionInit`, Senden/Empfangen über die Retry-Wrapper, optional `HTTPS_ParseData`, danach
  `HTTPS_TLS_Context_Reset` + Socket schließen. Einziger Aufrufer: `Cloud_UploadTelemetryData`
- `HTTPS_TLSSessionInit` – baut den mbedTLS-Handshake auf (SSL/Config/CTR-DRBG/Entropy, TLS 1.2 fest,
  SNI, bis zu 31 Handshake-Retries)
- `HTTPS_TLS_Context_Reset` – sendet ggf. `close_notify` (`mbedtls_ssl_close_notify`) und gibt alle
  mbedTLS-Kontexte frei
- `HTTPS_PackPostRequest` – baut die HTTP/1.1-POST-Kopfzeilen (Host, Connection,
  `Content-Type: application/json`, `Content-Length`) um den JSON-Body aus `g_cloudUploadBodyBuf`
- `HTTPS_ParseData` / `HTTPS_ParserOnBody` – initialisiert `llhttp` über die HTTPS-Antwort, kopiert bis
  zu 0x4B0 Byte Body in `g_httpsBodyBuf`
- `CH395_TCP_Socket_Open_HTTPS` – öffnet einen CH395-TCP-Socket auf Port 443
- `CH395CMDGetSocketStatus` – liest per SPI-Kommando 0x2F den Socket-Status
- `mbedTLS_SSL_Send_WithRetry` / `mbedTLS_SSL_Recv_WithRetry` – Wrapper um `mbedtls_ssl_write`/`_read`
  mit Retry bei `WANT_READ`/`WANT_WRITE`
- `mbedtls_ssl_close_notify` – Standard-mbedTLS-Bibliotheksfunktion, neu mit eingebunden

**Peak-Shaving / Leistungs-Klemm-Modus** (EEPROM 0x394, 6 Byte):
- `Config_Read_PowerClampMode_0x394` / `Config_Write_PowerClampMode_0x394` – Byte 0 = Modus (0/1,
  ungültige Werte werden beim Lesen auf 0 zurückgesetzt), Byte 2–3 = Klemm-Leistung in Watt, Rest
  reserviert. Gelesen einmalig beim Boot (`Config_LoadOrInit_DeviceSettings`), geschrieben über
  BLE-Kommando `)` und MQTT-Kommando `0x3f`
- `MQTT_Publish_Grid_Status` – neue MQTT-Antwort (Kommando `0x41`) mit Netz-/Batteriestatus inkl.
  `g_minPowerLimit`/`g_maxPowerLimit`, Netzspannung/-frequenz/-leistung, Fehler-/Warncodes; gleiches
  Ctrl-Topic wie die übrigen `MQTT_Publish_*`-Funktionen

**Sonstige Infrastruktur:**
- `Quectel_SSL_ConfigVersion` – sendet `AT+QSSLCFG="version",<ctx>,<tls_version>` ans Mobilfunkmodem;
  neuer Zwischenschritt in `Quectel_MQTT_Config_StateMachine`
- `Quectel_HTTP_POST_SendAndVerify` – POST-Pendant zu `Quectel_HTTP_AT_SendAndVerify` für den
  modemseitigen `AT+QHTTPPOST`-Ablauf in `HTTP_Task_StateMachine_Handler`
- `FreeRTOS_xPortGetFreeHeapSize` / `FreeRTOS_xPortGetMinimumEverFreeHeapSize` + neuer Shell-Befehl
  `ShellCmd_FreeRtos_mem` – gibt freien/minimal-freien FreeRTOS-Heap über die Konsole aus
- `__vfscanf` – C-Bibliotheksroutine, neu mit eingebunden (ein scanf-Codepfad wird in v150 erstmals
  erreicht; auslösende Aufrufstelle nicht lokalisiert)

## Entfernte Funktion

- **`Base64URL_Encode`** (nur in v149.2, einziger Aufrufer war `Cloud_UploadTelemetryData`): v149.2 hat
  das Telemetrie-Paket vor dem Versand AES-128-ECB-verschlüsselt (`AES128_ECB_Encrypt`, Schlüssel über
  `KeyDerive_CopyAndROT`) und anschließend Base64URL-kodiert in einen festen 0x960-Byte-Puffer
  geschrieben, der als Query-Parameter `v=` an `setVenusDReporting` ging. v150 verzichtet komplett auf
  Verschlüsselung und Kodierung: das Paket wird nur noch in `{"d":"..."}` gewrappt und im Klartext per
  HTTPS-POST an den neuen Endpunkt geschickt (Transportsicherheit übernimmt TLS). Siehe „Echte
  Änderungen" Nr. 1.

## Echte Änderungen (26)

**1. Cloud-Telemetrie-Upload grundlegend überarbeitet — und ein Pufferüberlauf in v149.2 behoben** —
`Cloud_UploadTelemetryData` (0801af64→0801b398)
Payload jetzt JSON `{"d":"...&soh=%d"}\r\n` statt rohem Query-String; Verschlüsselung/Kodierung
(AES128_ECB_Encrypt + Base64URL_Encode) entfällt komplett; Ziel wechselt von
`http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (GET, Query-Param) auf
`https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (POST-Body); bei aktivem Ethernet wird jetzt
`HTTPS_FetchUrlData` (neuer TLS-Client) statt der bisherigen `HTTPS_POST_Request` gerufen, bei Mobilfunk
weiterhin `Quectel_HTTP_AT_SendAndVerify`.
**Konkret gefundener Fehler in v149.2:** `Base64URL_Encode` wurde mit einer behaupteten Zielpuffer-Größe
von `0x960` (2400 Byte) aufgerufen, `g_cloudUploadBodyBuf` ist laut Globals-Referenz aber nur
`uchar[1200]` (1200 Byte) groß. `Base64URL_Encode` prüft nur, ob die *übergebene* Größe für die
Ausgabelänge reicht (`param_4 < (param_2+2)/3*4`), nicht ob sie zum tatsächlichen Puffer passt — bei
AES-Ausgaben, deren Base64-Kodierung zwischen 1200 und 2400 Byte liegt, schrieb die Funktion also über
das Ende von `g_cloudUploadBodyBuf` hinaus in benachbarte Globals. v150 entfernt diesen ganzen
Kodierungsschritt vollständig und kopiert den JSON-Body exakt in Sprintf-Länge (kein fester
Kodierungspuffer mehr). Erklärt „Datenverlust durch zu lange HTTP-Datensätze behoben" direkt als
Speicherüberlauf-Fix, nicht nur als Effizienzgewinn.

**2. Zwei neue Verbindungs-Watchdogs plus GET→POST-Umbau des Mobilfunk-Uploads**
- `HTTP_ClientTask_Dispatcher` (08019ff0→0801a3d0): neuer Block – wenn der Upload-Rückstand > 1 ist und
  seit 0x708 (1800) Ticks kein Fortschritt vorliegt und `g_oldStep`==0, löst `CH395_Reset_And_Reinit(0)`
  (CH395-Ethernet-Chip-Reset) aus und setzt den Dispatcher-Zustand zurück (Log „[HTTP]ch395 reset!!!!")
- `HTTP_Task_StateMachine_Handler` (080109e0→08010c20): neuer Block – wenn ein Upload-Rückstand besteht
  und seit 900 Ticks kein Fortschritt vorliegt, löst `Quectel_Modem_HardwareReset_Handler(0)`
  (Mobilfunkmodem-Reset) aus (Log „[HTTP]fc41d reset!!!!"). Zusätzlich werden die Zustände 6–9 dieser
  Zustandsmaschine umgebaut: v149.2 nutzte dort `Quectel_HTTP_GET_Request`/`Quectel_HTTP_ReadResponse_Verify`
  (GET-basiert, inkl. Sonderfall für Modem-Firmware „2025"); v150 sendet stattdessen zuerst
  `AT+QHTTPCFG="header","Content-Type","application/json"`, dann `AT+QHTTPPOST=%d,60,60` gefolgt von
  `Quectel_HTTP_POST_SendAndVerify(...,"CONNECT",...,g_cloudUploadBodyBuf,...)` — ein echter POST mit dem
  JSON-Body aus Nr. 1 statt eines GET

Passt zu „Fehlerhaftes Senden der Local API im Ethernet-Modus behoben" /
„Fehlgeschlagene HTTP-Updates im Ethernet-Modus behoben" und ergänzt Nr. 1 um den mobilfunkseitigen Teil
des Protokollumbaus.

**3. `DeviceInfo_BuildStatusString`** (0802ab44→0802b470): Hardware-Varianten-Tag für 6 weitere
Zählertypen. Bisher lieferte die Funktion für **alle** Zählertypen den generischen EEPROM-String
(`Config_Read_String_0x387`) als Hardware-Variante der Local API. v150 fügt für `g_meterSourceType` 3,
4, 8, 9, 10, 11 feste Tags ein (`HME-4`, `HME-3`, `SMR-0`, `SMR-1`, `SMR-2`, `TPM2-0`) – nur die
restlichen Typen fallen weiter auf den EEPROM-String zurück. Erklärt direkt „Fehlerhaftes Senden der
Local API im Ethernet-Modus behoben".

**4. `Grid_PowerSync_StateMachine`** (08030f84→08031b60): Zählertyp-Klassifikation für die
CT-Direktübernahme erweitert. Die Gruppe von `g_meterSourceType`-Werten, die den Zweig „ohne
Deviation-Gate direkt übernehmen" nimmt, wächst von einer schmalen Liste (`2, 9, 10, 12, 4–7`) auf einen
breiten Bereich (`2, 12, 3–11, 14+`). Passt zu „Anbindung des Stromzählers auf CT_TYPE-Verbindung
umgestellt".

**5. Peak-Shaving/Leistungs-Klemm-Modus** (neu, EEPROM 0x394, 6 Byte) — mehrere Stellen:
- `Config_LoadOrInit_DeviceSettings` (0802d174→0802dcbc): liest den Klemm-Modus beim Boot
  (`Config_Read_PowerClampMode_0x394`); das umgebende EEPROM-Konfigblock (Basis 0x300) wächst von 0x92
  auf 0x98 Byte für das neue Feld; das Schedule-EEPROM (`g_canScheduleRegCache`) zieht unabhängig davon von
  0x36BF nach 0x3700 um (identisch in `Write_Handler`, 08055720→08056514)
- `Inverter_PowerSetpoint_DeadbandClamp` (080184c8→0801877c): bei aktivem Klemm-Modus (Byte 0 = 1) wird
  der Setpoint nur noch nach unten auf die Grenze angehoben (Boden) statt wie sonst nach oben auf die
  Grenze gedrückt (Deckel)
- `Remote_Power_Setpoint_Process` (0802dc58→0802e7b0): die aus dem Zeitplan (TOU) berechnete
  Entladeleistung wird bei aktivem Klemm-Modus zusätzlich auf `aktuelle CT-Netzleistung (g_ctPowerNow) −
  konfigurierter Klemmwert` begrenzt
- `BLE_App_Command_Dispatcher` (0800c758→0800c720): neues Kommando `)` (0x29) setzt/liest Modus+Wert
  über `Config_Write_PowerClampMode_0x394` + `Config_Write_PowerOffset`; außerdem behebt v150 einen
  Fallthrough-Bug (fehlendes `break` nach Kommando `Y`, das ungewollt in `Z` durchlief)
- `BLE_Build_DeviceStatus_Response` (0800f778→0800f824): meldet Modus+Wert zusätzlich
  (`BLE_Send_Response`-Länge 0xA4→0xA7)
- `MQTT_CtrlCommand_Dispatcher` (08023c60→0802423c): neues Kommando `0x3f` „Set peak shaving"
  (Parameter `as=`/`vv=`), altes Kommando `0x3e` (`MQTT_Publish_DeviceIDs`) auf `0x40` verschoben, neues
  Kommando `0x41` „Get INV data" ruft `MQTT_Publish_Grid_Status`
- `MQTT_StatusReport_BuildAndPublish` (08029cc0→0802a5dc): Statuszeile bekommt neue Felder
  `peak_status=%d,peak_power=%d`

Passt zum Changelog-Punkt „Peak-Shaving-Funktion ergänzt".

**6. BMS-Selbsttest- und MOS-Flag zusätzlich gemeldet** (unabhängig von Peak-Shaving)
- `BLE_Build_BMS_Data_Response` (0800fc30→0800fcf4): neues Feld `g_bmsSelfCheckFlag`
  (= `g_pgn1007InfoSelfCheck`, CAN-PGN-1007-Selbsttest), Antwortlänge 0x50→0x51
- `BMS_StatusQuery_Response_Send` (080238d0→08023e98): Statuszeile bekommt `self_check=%d,mos=%d`
  (letzteres = `g_bmsPack0MosFlag`); Stack-Rahmen dadurch 0x98→0xa0

**7. `Config_Read_String_0x387`/`Config_Write_String_0x387` — Off-by-one-Bug in der EEPROM-Adresse
behoben.** Beide Funktionen griffen in v149.2 trotz ihres Namens auf EEPROM 0x388 statt 0x387 zu
(13-Byte-Alias-Namen-Datensatz belegte 0x388–0x394). v150 korrigiert auf 0x387 (0x387–0x393) —
vermutlich nötig, damit der neue 6-Byte-Klemm-Modus-Datensatz auf 0x394 (Nr. 5) nicht mit dem Ende des
Alias-Namens kollidiert.

**8. `HTTPS_POST_Request`** (080195c0→08019894): Empfangspuffer vergrößert und mit HTTPS-Pfad
zusammengelegt. Der für die Antwort genullte Bereich wächst von 0x1958 (6488) auf 0x1DC0 (7616) Byte;
`HTTPS_POST_ReceiveResponseData` empfängt jetzt in den gemeinsamen Puffer `g_httpsRespBuf` statt in das
bisherige `g_httpRxBuf`. Passt ebenfalls zu „Datenverlust durch zu lange HTTP-Datensätze behoben".

**9. `HTTP_ParseUploadResponseCode`** (08019dd0→0801bf4c): anderer JSON-Schlüssel. Sucht jetzt
`"code":` (7 Zeichen) statt `"de":` (5 Zeichen) in der Upload-Antwort — nicht nur ein um zwei Byte
verschobenes Offset wie in der Vorfassung dieses Dokuments behauptet, sondern ein komplett anderer
JSON-Schlüssel.

**10. `MQTT_Publish_Network_Info`** (08026f18→080276fc): meldet im kurzen Statuszweig jetzt
`g_meterResolvedIpStr` (tatsächlich vom Zähler zurückgemeldete IP) statt `g_meterQueryIpStr`
(angefragte IP) — wie im ausführlichen Zweig ohnehin schon.

**11. `Quectel_MQTT_Config_StateMachine`** (08012f94→080131c4): TLS-Konfigurationsreihenfolge geändert,
neuer Schritt. Reihenfolge jetzt: `Quectel_SSL_ConfigVerifyLevel` zuerst (statt `ConfigSNI`), neuer
Zwischenschritt `Quectel_SSL_ConfigVersion` (TLS-Version), `Quectel_SSL_ConfigSNI` jetzt letzter Schritt
(Parameter 2→1).

**12. `Quectel_MQTT_Session_StateMachine`** (08012808→08012a30): Factory-Reset im SSL-Watchdog jetzt
bedingt. Der Werksreset nach 1800 s SSL-Inaktivität erfolgt nur noch, wenn `g_oldStep`
(Arbeitsmodus-Zwischenschritt) == 0 ist; der reine Modem-Hardware-Reset läuft weiterhin immer.

**13. `mbedTLS_SSL_Connection_Init`** (0801cb64→0801d134): doppelten TLS-Reset entfernt. Der bei
Fehlschlag aufgerufene `MQTT_TLS_Context_Reset()` entfällt hier, weil der Aufrufer (`MQTT_Session_Init`
→ `CH395_MQTT_Socket_Close_And_TLS_Reset`) ihn ohnehin schon aufruft — reine Redundanz-Bereinigung.

**14. `Battery_EPS_Reserve_SOC_Hysteresis`** (0800add0→0800ad3c): ein Byte im CAN-0xCF-Report geändert.
Das niederwertige Byte des gesendeten 16-Bit-Feldes ist in v149.2 immer 0, in v150 immer 1
(`CONCAT11(count,1)` statt `count<<8`). Bedeutung des Bit-Werts offen.

**15. `Cloud_EdgeDetectAndWatchdog`** (080312f4→08031ebc): Retry zusätzlich an MPPT-Kanal gebunden. Der
Retry (`Cloud_Response_Action(1)`) bei Edge-Case 3 läuft jetzt nur noch, wenn `g_mpptActiveChannel` 0
oder 5 ist (vorher unbedingt).

**16. `Quectel_HTTP_AT_SendAndVerify`** (080104c4→080106f4): zusätzliche Wartezeit. Ein
`vTaskDelay(delay_ms)` direkt nach dem Senden des AT-Kommandos kommt hinzu (vor dem Warten auf die
Notify) — gibt dem Modem mehr Anlaufzeit.

## Kosmetisch (kein Verhaltensunterschied)

- **Embedded-Firmware-Versionsliteral** (0x5D4 = 1492 → 0x96 = 150) turnusmäßig aktualisiert, rein
  informativ: `Config_LoadOrInit_DeviceSettings`, `ShellCmd_get_ver`, `HTTP_Cloud_Reporting_Dispatcher`,
  `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `MQTT_Transport_NullCallback`/`Util_NoOp_Stub`
  (Datenliteral im Konstantenblock)
- **`__LINE__`-Logmakro-Verschiebungen** (Log_Printf-Zeilenargument, ohne Effekt), verifiziert durch
  identischen umgebenden Code: `CH395_ExtractUdpData`, `CH395_TCP_Socket_Open_HTTPPort80` (**Korrektur:**
  in der Vorfassung fälschlich als echte Parameteränderung 0x24→0xAA gewertet — der Funktionskörper ist
  byte-für-byte identisch, der Wert ist nur ein Log-Zeilenargument), `CT_GridPower_Controller`,
  `Device_Network_Info_Init`, `HTTP_PackGetRequest` (**Korrektur:** in der Vorfassung fälschlich als
  Puffervergrößerung 0x168→0x280 gewertet — `out_size` bleibt unveränderter Funktionsparameter, der Wert
  ist ebenfalls nur ein Log-Zeilenargument), `Meter_GetMeterIP_FromResponse`, `Meter_ParseUDPMeterInfo`,
  `MQTT_Connect_And_Subscribe`, `MQTT_TLS_Context_Reset`, `Parse_IP_Address_String`,
  `Quectel_GetDeviceIP`, `Quectel_UDP_Socket_Open`, `SMR_TIC_SelfUse_PowerController` (9 Konstanten, alle
  um exakt +0xE verschoben)
- **Compiler-/Decompiler-Rauschen**: `MQTT_Publish_Meter_Data` (nur Typ-Anmerkung `pointer` →
  `undefined *`), `llhttp_errno_name` (Ghidra erkennt die Sprungtabelle des `default`-Falls in v149.2
  nicht und dekompiliert Tabellen-Padding als Fehlcode — die Fehlernamen-Tabelle selbst ist
  unverändert), `pow`/`fp64_abs`/`__scatterload`/`sscanf_Format_Parser`/`vPortStartFirstTask`/
  `SPI_WriteByte`/`ShellCmd_reset` (Literal-Pool-Verschiebungen, Register-Zuteilung)
- `HTTP_ParserOnBody`/`HTTP_ParserOnMessageComplete`/`HTTP_ParseData` (0804d8b4/0804d924/0802b130): die
  Umbenennung `g_httpRespBodyBuf` → `g_httpBodyBuf` betrifft dieselbe Adresse (0x6001fa68, nur
  Namensverlust in dieser Analyse-Runde), die Kopiergrenze bleibt exakt 0x4B0 (1200 Byte); die restlichen
  geänderten Konstanten (0xd3/0xdc/0xe8) sind ebenfalls nur `__LINE__`-Argumente der zugehörigen
  Log_Printf-Aufrufe. Die tatsächlich neue, separate `HTTPS_ParserOnBody`/`g_httpsBodyBuf`-Instanz für den
  HTTPS-Pfad ist unter „Neue Funktionen" gelistet
- Namensvertauschung `g_bleDevStatusPvMinusGridEst`/`g_bleDevStatusLoadPowerEst` in
  `BLE_Build_DeviceStatus_Response`: gleiche Struct-Offsets, gleiche Formeln — nur analyse-interne
  Umbenennung, keine reale Feldvertauschung

## Changelog v150 → Code

| Changelog-Punkt | Befund im Code |
|---|---|
| Fehlerhaftes Senden der Local API im Ethernet-Modus behoben | `DeviceInfo_BuildStatusString`: 6 neue Hardware-Varianten-Tags für Zählertypen 3/4/8/9/10/11 (siehe „Echte Änderungen" Nr. 3) |
| Fehlgeschlagene HTTP-Updates im Ethernet-Modus behoben | Neue CH395- und Modem-Watchdogs plus GET→POST-Umbau des Mobilfunk-Uploads in `HTTP_ClientTask_Dispatcher`/`HTTP_Task_StateMachine_Handler` (Nr. 2) |
| Peak-Shaving-Funktion ergänzt | Neues EEPROM 0x394 + BLE-Kommando `)` + MQTT-Kommandos 0x3f/0x41 + Klemmlogik in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` (Nr. 5) |
| Datenverlust durch zu lange HTTP-Datensätze behoben | `Cloud_UploadTelemetryData`: behobener Pufferüberlauf in `Base64URL_Encode` (behauptete Kapazität 2400 B vs. tatsächliche Puffergröße 1200 B), komplette Umstellung auf JSON/HTTPS ohne AES+Base64URL (Nr. 1); `HTTPS_POST_Request`-Empfangspuffer 6488→7616 Byte (Nr. 8) |
| Anbindung des Stromzählers auf CT_TYPE-Verbindung umgestellt | `Grid_PowerSync_StateMachine`: Zählertyp-Bereich für Direktübernahme erweitert (Nr. 4) |

## Offene Punkte

- `Battery_EPS_Reserve_SOC_Hysteresis`: Bedeutung des neu auf 1 gesetzten Low-Bytes im CAN-0xCF-Report
  nicht ermittelt
- `__vfscanf` (neu eingebundene C-Bibliotheksfunktion): auslösende Aufrufstelle nicht lokalisiert
- `ShellCmd_FreeRtos_mem`: über die Shell-Kommando-Linker-Sektion eingebunden statt über eine normale
  Aufruftabelle, kein Code-Querverweis auffindbar (Funktion selbst aber eindeutig: gibt freien/minimal
  freien FreeRTOS-Heap aus)
- Fallthrough-Fix BLE-Kommando `Y`→`Z` (Nr. 5): ob das in v149.2 beobachtbare Fehlverhalten auslöste
  oder folgenlos blieb, wurde nicht geprüft

## Datenbasis

- Instruktions-Diff: `scratchpad/v151/l1492_150.json` (lokal, nicht im Repo) (1878 identisch, 56 geändert, 1 nur v149.2, 20 nur v150)
- Modbus-Deskriptoren: `scratchpad/v151/mbdesc_1492.json`, `mbdesc_150.json`
- Dekompilat + Listing: `Dekompilate/VNSD_Control_1492/`, `Dekompilate/VNSD_Control_v150_base4800/` (lokal, nicht im Repo)
