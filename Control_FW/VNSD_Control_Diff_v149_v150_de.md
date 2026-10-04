# Diff-Tabelle Control v149 → v150 (Venus D) — Haupthop

> 🇬🇧 English version: [VNSD_Control_Diff_v149_v150_de.md](VNSD_Control_Diff_v149_v150_de.md)

**Stand:** 2026-09-30. Beide Images mit korrekter Basis 0x08004800 in Ghidra, 100 % benannt.
Dies ist der Hauptversions-Sprung v149 → v150. Dazwischen liegt die Beta **v149.2**, über die der
Großteil der Entwicklung lief.
**Englisch:** `VNSD_Control_Diff_v149_v150_de.md`
**Detail-Hops:** `VNSD_Control_Diff_v149_v1492_de.md` (v149→v149.2) · `VNSD_Control_Diff_v1492_v150_de.md` (v149.2→v150)

**Korrektur gegenüber der Vorfassung dieses Dokuments:** Die alte Fassung beruhte auf einer damals
unvollständigen Paarung für den v149.2→v150-Schritt (nur 9 gefundene Unterschiede) und kam dadurch zu
falschen Schlüssen: BLE angeblich unverändert im Befehlssatz, Cloud-Domain angeblich komplett
unverändert, und zwei angebliche „echte" Änderungen (`CH395_TCP_Socket_Open_HTTPPort80`,
`HTTP_PackGetRequest`), die tatsächlich nur verschobene Log-Zeilennummern sind. Diese Fassung basiert auf
den heute neu erstellten, vollständigen Funktionsdiffs beider Hops plus einem direkten Vollvergleich
v149 ↔ v150.

## Methode

Direkter Vollvergleich v149 → v150 (`scratchpad/v151/l149_150.json` (lokal, nicht im Repo), normalisierte Instruktions-Opcodes):
Funktionspaare über gleiche Namen **1955**, davon **82 geändert**, **1 nur in v149** (`Base64URL_Encode`),
**24 nur in v150**, 1848 byte-identisch. Die beiden Hop-Dokus zählen zusätzlich getrennt: v149→v149.2
58 geändert (38 inhaltlich/20 kosmetisch), 0/4 nur A/B; v149.2→v150 56 geändert (26 inhaltlich), 1/20 nur
A/B. Die Summen der Hops sind wegen Rückbau-/Gegenläufig-Effekten nicht direkt auf die 82 der
Direktdiff addierbar (siehe „Offene Punkte" der Hop-Dokus); die Direktdiff-Zahlen sind für den
Gesamtsprung maßgeblich.

## Prüfliste (v149 → v150 gesamt)

| Frage | Ergebnis |
|---|---|
| Modbus-Register | **unverändert.** Deskriptortabelle 246 Einträge in beiden Ständen, Registernummer/Typ/Skalierung/Anzahl identisch. Nur RAM-Schattenadressen verschieben sich (Config-Struct wuchs um 6 Byte für Peak-Shaving; Neulinkung); Schedule-EEPROM ab Register 46500 zieht von 0x36BF nach 0x3700 um (`Config_LoadOrInit_DeviceSettings`, `Write_Handler`) |
| BLE | **erweitert, nicht unverändert.** Geräte-Alias-Feld an Kommando 0x18 angehängt (v149→v149.2); neues Kommando `0x29` „Set Peak shaving" (v149.2→v150); `BLE_Build_DeviceStatus_Response` (0xA4→0xA7 B) und `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` (0x50→0x51 B) um Klemm-Status bzw. Selbsttest-/MOS-Flag gewachsen; Fallthrough-Bug Kommando `Y`→`Z` behoben |
| AC-Einspeisung 2500 W | **Hartgrenze unverändert** — `Grid_Export_Power_Limiter` und `Config_ClampDefaults_ParamBlock` über beide Hops byte-identisch (taucht im Direktdiff nicht als geändert auf). Zusätzlich, unabhängig davon, neu: konfigurierbarer Peak-Shaving-Klemmwert (EEPROM 0x394), wirkt in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` |
| BLE-OTA | **weiterhin möglich.** Kompletter OTA-/BLE-OTA-/YMODEM-Pfad (`BLE_Cmd_OTA_*`, `YModem_FirmwareUpgrade_Handler`, `OTA_*`) über beide Hops byte-identisch, im Direktdiff nicht in der Änderungsliste |
| Cloud-Domains | `hamedata.com`-Endpunkte selbst **unverändert** (identische Strings in v149 und v150, per grep in beiden Listings bestätigt: `/app/neng/`, `/ems/api/v1/get...`, `/external-services/...`). Aber: der Telemetrie-Upload wechselt von `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (AES128-ECB + Base64URL im Query-String) auf `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (Klartext-JSON per HTTPS-POST über neuen mbedTLS-Client). Verifiziert: String `setVenusDReporting` kommt in keinem v150-Listing mehr vor, `marstekcloud.com` in keinem v149-Listing |

## Kernpunkte (wichtigste reale Änderungen, nach Thema)

| Thema | Änderung | Hop |
|---|---|---|
| Cloud-Upload-Protokoll | AES128-ECB+Base64URL-Query → Klartext-JSON per HTTPS-POST an neue Domain; dabei Pufferüberlauf in `Base64URL_Encode` behoben (behauptete Zielgröße 0x960 B vs. tatsächlicher Puffer 1200 B); `Cloud_UploadTelemetryData` 769→603 Instr. | v149.2→v150 |
| Verbindungs-Watchdogs | Neuer CH395-Reset bei Ethernet-Upload-Stau (>1800 Ticks ohne Fortschritt), neuer Modem-Hardware-Reset bei Mobilfunk-Upload-Stau (>900 Ticks); Mobilfunk-Upload zusätzlich von GET auf POST umgestellt | v149.2→v150 |
| Peak-Shaving / Leistungsklemmung | Neu: EEPROM 0x394 (Modus + Watt-Wert), BLE-Kommando `0x29`, MQTT-Kommandos `0x3f`/`0x41` (`MQTT_Publish_Grid_Status` neu), Klemmlogik in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` | v149.2→v150 |
| BMS-Pack-Record | Record 88→96 Byte, neues SoH-Feld (`g_bmsPack0Soh`), Dispatch-Tabelle 16→17 Einträge, `CAN_AA_StoreUnitFieldPair88` neu | v149→v149.2 |
| BMS-Zusatzflags | Selbsttest-Flag (CAN-PGN-1007) und MOS-Flag zusätzlich über BLE/MQTT gemeldet | v149.2→v150 |
| CT/SMR-TIC-Regler | Aufgeteilt in `SMR_TIC_SelfUse_PowerController` (nur TIC-FSM) + neue `CT_PowerSetpoint_Compute` (Klemmung); reine Umstrukturierung, keine Verhaltensänderung | v149→v149.2 |
| Meter-Klassifikation | `Grid_PowerSync_StateMachine`: Zählertypen mit „Direktübernahme ohne Deviation-Gate" von schmaler Liste (2,4–7,9,10,12) auf breiten Bereich (2,3–11,12,14+) erweitert | v149.2→v150 |
| Local-API-Hardwaretag | `DeviceInfo_BuildStatusString`: 6 feste Tags für Zählertypen 3/4/8/9/10/11 statt generischem EEPROM-String für alle | v149.2→v150 |
| Geräte-Alias | Neuer 12-Zeichen-Alias in EEPROM 0x387/0x388, schreibbar über BLE 0x18 und Cloud-Config; Off-by-one-Bug (Zugriff auf 0x388 statt 0x387) im Folgehop behoben | v149→v149.2, Fix in v149.2→v150 |
| Diverse Kleinfixes | `Quectel_MQTT_Session_StateMachine`: Factory-Reset im SSL-Watchdog nur noch bei `g_oldStep`==0; `Cloud_EdgeDetectAndWatchdog`: Retry an MPPT-Kanal gebunden; `Quectel_HTTP_AT_SendAndVerify`: zusätzliches Delay nach AT-Kommando; `mbedTLS_SSL_Connection_Init`: redundanten TLS-Reset entfernt; `Battery_EPS_Reserve_SOC_Hysteresis`: Low-Byte im CAN-0xCF-Report neu immer 1 (Bedeutung offen) | v149.2→v150 |

## Neue Funktionen (24 gesamt)

**v149→v149.2 (4):** `CAN_AA_StoreUnitFieldPair88`, `CT_PowerSetpoint_Compute`, `Config_Read_String_0x387`,
`Config_Write_String_0x387`

**v149.2→v150 (20):**
- HTTPS/TLS-Client (CH395+mbedTLS, neuer paralleler Pfad zum Cloud-Upload): `HTTPS_FetchUrlData`,
  `HTTPS_TLSSessionInit`, `HTTPS_TLS_Context_Reset`, `HTTPS_PackPostRequest`, `HTTPS_ParseData`,
  `HTTPS_ParserOnBody`, `CH395_TCP_Socket_Open_HTTPS`, `CH395CMDGetSocketStatus`,
  `mbedTLS_SSL_Send_WithRetry`, `mbedTLS_SSL_Recv_WithRetry`, `mbedtls_ssl_close_notify`
- Peak-Shaving: `Config_Read_PowerClampMode_0x394`, `Config_Write_PowerClampMode_0x394`,
  `MQTT_Publish_Grid_Status`
- Sonstige: `Quectel_SSL_ConfigVersion`, `Quectel_HTTP_POST_SendAndVerify`,
  `FreeRTOS_xPortGetFreeHeapSize`, `FreeRTOS_xPortGetMinimumEverFreeHeapSize`, `ShellCmd_FreeRtos_mem`,
  `__vfscanf`

## Entfernte Funktion

- **`Base64URL_Encode`** (v149→v149.2 vorhanden, v150 entfernt): war Teil des alten AES+Base64URL-Upload-
  Pfads (siehe Cloud-Upload-Protokoll oben); mit dem Umstieg auf Klartext-JSON/HTTPS überflüssig geworden
  und komplett entfernt, inklusive des darin steckenden Pufferüberlauf-Bugs.

## Im Code widerlegte Altannahmen (per Instruktions- und Quelltext-Vergleich geprüft)

- `CH395_TCP_Socket_Open_HTTPPort80`: einzige Instruktionsänderung `movs r0,#0x24`→`#0xaa`; Dekompilat
  v149 vs. v150 byte-für-byte identisch — der Wert ist ein Log_Printf-Zeilenargument, keine
  Funktionsänderung.
- `HTTP_PackGetRequest`: einzige Instruktionsänderung `movw r0,#0x153`→`mov.w r0,#0x280`; Dekompilat
  identisch, `out_size` bleibt unveränderter Funktionsparameter — ebenfalls nur ein Log-Zeilenargument,
  keine Puffervergrößerung.
- `HTTP_ParseUploadResponseCode`: keine reine Offsetverschiebung, sondern anderer JSON-Schlüssel
  (`"de":` 5 Zeichen → `"code":` 7 Zeichen, `adds r4,r4,#0x5`→`#0x7`).

## Unverändert (über beide Hops explizit geprüft)

Modbus-Registerinhalte komplett, Hartgrenze der Einspeisegrenzen (`Grid_Export_Power_Limiter`,
`Config_ClampDefaults_ParamBlock`), BLE-Befehls-IDs (nur Erweiterung, keine Umnummerierung), kompletter
OTA-/BLE-OTA-/YMODEM-Pfad, `hamedata.com`-Endpunkte selbst.

## Offene Punkte

- Genaue Codeursache des v149.2-Changelog-Punkts „Fehlerhafte Datenanzeige durch ausbleibende
  HTTP-Uploads" nicht isoliert (Kandidaten: HTTP-/Cloud-Telemetriekette, siehe Hop-Doku v149→v149.2).
- Bedeutung der ungenutzten Hälften des neuen BMS-Pack-Feldpaares bei Offset 0x58/0x5c.
- `Battery_EPS_Reserve_SOC_Hysteresis`: Bedeutung des neu auf 1 gesetzten Low-Bytes im CAN-0xCF-Report
  nicht ermittelt.
- `__vfscanf` (neu eingebunden) und `ShellCmd_FreeRtos_mem`: auslösende Aufrufstelle bzw. Aufrufer nicht
  lokalisiert.
- Ob der behobene Fallthrough-Bug BLE `Y`→`Z` in v149.2 tatsächlich beobachtbares Fehlverhalten auslöste,
  wurde nicht geprüft.
- Feininhalt der großen State-Machines (`HTTP_Task_StateMachine_Handler`, `MQTT_CtrlCommand_Dispatcher`,
  `BLE_App_Command_Dispatcher`) über die dokumentierten Punkte hinaus nicht vollständig
  instruktionsweise nachvollzogen.

## Datenbasis

- Direkter Vollvergleich v149 ↔ v150: `scratchpad/v151/l149_150.json`
- Detaillierte Einzeldiffs: `scratchpad/v151/l149_1492.json`, `scratchpad/v151/l1492_150.json`
- Dekompilat + Listing: `Dekompilate/VNSD_Control_149/`, `Dekompilate/VNSD_Control_1492/`,
  `Dekompilate/VNSD_Control_v150_base4800/` (lokal, nicht im Repo)
- Firmware-Changelog: `Marstek FW Archiv/firmwares/VNSD-0/Control/CHANGELOG.de.md` (lokal, nicht im Repo)
