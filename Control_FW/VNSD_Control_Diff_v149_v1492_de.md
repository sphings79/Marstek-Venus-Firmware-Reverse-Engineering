# Diff-Tabelle Control v149 → v149.2 (Venus D)

> 🇬🇧 English version: [VNSD_Control_Diff_v149_v1492_de.md](VNSD_Control_Diff_v149_v1492_de.md)

**Stand:** 2026-09-30. Beide Images mit korrekter Basis 0x08004800 in Ghidra, 100 % benannt (0 `FUN_`, 100 % Byte-Coverage).
Adressen: v149 → v149.2. Funktionspaare über gleiche Namen (1931), instruktionsweise (LCS) verglichen: **58 geändert**
(davon **38 inhaltlich**, 20 rein kosmetisch), **0 nur in v149**, **4 nur in v149.2**; 1873 Funktionen byte-identisch.
Kein Herstell-Changelog-Punkt außer einem (siehe unten) — die meisten Aussagen aus dem Code.
**Englisch:** `VNSD_Control_Diff_v149_v1492_de.md` · **Nachfolger-Diff:** `VNSD_Control_Diff_v1492_v150_de.md` · **Haupthop:** `VNSD_Control_Diff_v149_v150_de.md`

## Einordnung

v149.2 ist ein **Beta** zwischen den Hauptständen v149 und v150 (Zeitstempel: v149 = 21.05.2026,
v149.2 = 12.08.2026 laut Changelog, Firmwaredatei `1492_..._0702_142136.bin`). Der Beta-Schritt enthält vier
eigenständige Themen: die Erweiterung des CAN-0xAA-BMS-Pack-Records um ein Feld (State-of-Health), eine
Aufspaltung des CT/SMR-TIC-Sollwertreglers in zwei Funktionen, einen neuen Geräte-Alias-Namen (EEPROM 0x387) und
umfangreiche Umbauten an der Cloud-/MQTT-/HTTP-Telemetriekette.

## Prüfliste

| Frage | Ergebnis |
|---|---|
| Modbus-Register | **unverändert.** Deskriptortabelle 246 Einträge in beiden, Registernummern/Typ/Skalierung/Anzahl identisch geprüft (alle 217 Einträge, die sich im Rohbyte-Vergleich unterscheiden, unterscheiden sich ausschließlich im 4‑Byte-Quelladressfeld — Folge der Neulinkung durch die zusätzlichen Felder/Funktionen, kein Registerinhalt geändert) |
| BLE | Befehlssatz **nummerisch gleich**, aber Payload von **0x18** ("Zählerquelle setzen") um ein Feld erweitert: ein neuer, am Ende angehängter String wird über `Config_Write_String_0x387` als Geräte-Alias in EEPROM 0x387/0x388 gespeichert. `BLE_Build_BMS_Data_Response` liefert im bisher fest auf 0 stehenden Reservefeld jetzt den echten SoH-Wert (`g_bmsPack0Soh`). `BLE_Build_Cmd0x42_Response` unterscheidet sich nur durch eine unbenannte Zieladresse (Layout-Verschiebung, kein Verhaltensunterschied) |
| AC-Einspeisung 2500 W | **unverändert.** `Grid_Export_Power_Limiter` und `Config_ClampDefaults_ParamBlock` byte-identisch. `CT_GridPower_Controller` ändert nur den Aufruf-Ziel (`CT_PowerSetpoint_Compute` statt `SMR_TIC_SelfUse_PowerController`, siehe unten) und zwei Variablennamen, keine Klemmwerte |
| BLE-OTA | **weiter möglich.** `BLE_Cmd_OTA_Validate/Init/WriteSetup`, `YModem_FirmwareUpgrade_Handler` und alle `OTA_*` byte-identisch |
| Cloud-Domains | **unverändert.** Identischer `hamedata.com`-Endpunktsatz (`/app/neng/`, `/ems/api/v1/get`, `/external-services/api/v1/...`, `/prod/api/v1/set`) in beiden, nur Stringadressen verschoben |

## Neue Funktionen (nur in v149.2)

| Funktion | Adresse | Zweck / Aufrufer |
|---|---|---|
| `CAN_AA_StoreUnitFieldPair88` | 0x080334b4 | Achtes Feldpaar-Handler für Protokoll 0xAA, Offset 0x58 im jetzt 96 B großen Pack-Record (siehe Zeile „BMS-Pack-Record" unten); Dispatch-Tabelle `g_canAaDispatchTable` in `Protocol_AA_CommandDispatch` von 16 auf 17 Einträge erweitert |
| `CT_PowerSetpoint_Compute` | 0x080304e8 | Übernimmt die allgemeine (Nicht-TIC) CT-Sollwertberechnung inkl. Klemmung, vorher inline in `SMR_TIC_SelfUse_PowerController`; einziger Aufrufer `CT_GridPower_Controller` (`CT_PowerSetpoint_Compute(delta_power, device_nb)`) |
| `Config_Read_String_0x387` | 0x0800b340 | Liest den 12-Zeichen-Geräte-Alias aus EEPROM-Offset 0x387/0x388; Aufrufer `DeviceInfo_BuildStatusString` |
| `Config_Write_String_0x387` | 0x0800b63c | Schreibt den Geräte-Alias (max. 12 Zeichen, nullterminiert in 13-Byte-Puffer `g_deviceAliasName`) nach EEPROM 0x387; Aufrufer `BLE_App_Command_Dispatcher` (Befehl 0x18) und `Cloud_HTTP_Response_Parser` (Cloud-Config-Feld `cd=18`) |

## Substantielle Änderungen (instruktionsverifiziert)

| Bereich | v149 | v149.2 | Fundstelle (v149 → v149.2) | Ratio |
|---|---|---|---|---|
| BMS-Pack-Record: neues SoH-Feld | Record 0x58 B (88 B) groß, 16-Einträge-Dispatch-Tabelle (`Protocol_AA_CommandDispatch`: `cmp r5,#0x10`), kein SoH-Feld | Record **0x60 B (96 B)**, Dispatch-Tabelle **17 Einträge** (`cmp r5,#0x11`); neues Feldpaar bei Offset 0x58 (`CAN_AA_StoreUnitFieldPair88`): obere 16 Bit = `g_bmsPack0Soh` (State-of-Health, %), ausgelesen von `BLE_Build_BMS_Data_Response`, `MQTT_Telemetry_Struct_Builder`, `ShellCmd_get_all_bms_data` ("bat soh ="); untere 16 Bit (`g_bmsPack0Reserved58`) und zweites 32-Bit-Feld bei 0x5c ungenutzt | `Protocol_AA_CommandDispatch` 08033754→080337ac; Stride-Anpassung (0x2c→0x30 Elemente) in `BMS_CellVoltage_MinMax_PerPack`, `BMS_Version_Consistency_Check`, `CAN_Detect_Mismatched_Nodes`, `CAN_AA_StoreUnitFieldPair{0,8,16,24_Notify,64,72,80}`, `CAN_AA_StoreUnitIndexedFieldPair`; neu `CAN_AA_StoreUnitFieldPair88` 080334b4 | 0.38–0.98 |
| CT/SMR-TIC-Sollwertregler aufgeteilt | Eine Funktion (974 B) macht TIC-Zustandsmaschine **und** allgemeine CT-Sollwertberechnung/-Klemmung inline; `CT_GridPower_Controller` ruft sie ohne Argumente auf | Aufgeteilt: `SMR_TIC_SelfUse_PowerController` (590 B) macht nur noch die TIC-Zustandsmaschine; neue Funktion `CT_PowerSetpoint_Compute` (siehe oben) übernimmt Sollwertberechnung/Klemmung/Anwendung; `CT_GridPower_Controller` ruft jetzt `CT_PowerSetpoint_Compute(delta_power, device_nb)` auf. Reine Umstrukturierung, kein Funktionsverlust — Aufrufe wurden verschoben, nicht entfernt | `SMR_TIC_SelfUse_PowerController` 08030074→08008654 (0.529), `CT_GridPower_Controller` 0802fe30→080302a4 (0.979) | 0.53 / 0.98 |
| Geräte-Alias-Name (neu) | – | 12-Zeichen-String in EEPROM 0x387/0x388 (`g_deviceAliasName`), schreibbar über BLE 0x18 (angehängtes Feld) und Cloud-Config `cd=18`, lesbar in `DeviceInfo_BuildStatusString` | `BLE_App_Command_Dispatcher` 0800c31c→0800c758, `Cloud_HTTP_Response_Parser` 08028450→08028928, `DeviceInfo_BuildStatusString` 0802a5fc→0802ab44 (283→214 Instr.) | 0.83 / 0.58 / 0.57 |
| MQTT BMS-Datenpaket | 517 Instr. | 466 Instr., umgebaut (u. a. SoH-Feld ergänzt) | `MQTT_Publish_BMS_Data` | 0.586 |
| Cloud-Telemetrie-Upload | 769 Instr. | 785 Instr., umgebaut | `Cloud_UploadTelemetryData` | 0.566 |
| Cloud-HTTP-Antwortparser | 252 → 292 Instr. | erweitert (u. a. `cd=18`-Alias-Schreibpfad) | `Cloud_HTTP_Response_Parser` | 0.577 |
| MQTT-Statusbericht | 752 → 755 Instr. | umgebaut | `MQTT_StatusReport_BuildAndPublish` | 0.685 |
| HTTP-Task-Statemachine | 2522 → 2589 Instr. | erweitert; Feininhalt jenseits der Größenänderung nicht vollständig instruktionsweise nachvollzogen | `HTTP_Task_StateMachine_Handler` | 0.691 |
| Ethernet-Statustelemetrie | 78 → 54 Instr. | reduziert | `Ethernet_SendStatusTelemetry` | 0.682 |
| UDP-Zähler-Parse | 98 → 74 Instr. | reduziert | `Meter_ParseUDPMeterInfo` | 0.721 |
| Shell `get_all_bms_data` | 195 → 184 Instr. | reduziert; zusätzliche `"bat soh ="`-Ausgabe | `ShellCmd_get_all_bms_data` | 0.749 |
| MQTT-Ctrl-Verteiler | 2947 → 2960 Instr. | umgebaut; Feininhalt jenseits der Größenänderung nicht vollständig instruktionsweise nachvollzogen | `MQTT_CtrlCommand_Dispatcher` | 0.764 |
| Modem-Statustelemetrie | 125 → 101 Instr. | reduziert | `Modem_SendStatusTelemetry` | 0.796 |
| BLE-Befehlsverteiler | 2969 → 2985 Instr. | Befehl 0x18 um Alias-Schreibpfad erweitert; übrige Größenänderung nicht vollständig instruktionsweise nachvollzogen | `BLE_App_Command_Dispatcher` | 0.827 |
| Config Laden/Init | 380 → 384 Instr. | angepasst | `Config_LoadOrInit_DeviceSettings` | 0.895 |
| MQTT-Telemetrie-Struct | 273 → 276 Instr. | Feld ergänzt (SoH) | `MQTT_Telemetry_Struct_Builder` | 0.965 |
| MQTT Telemetrie-Formatter | 358 Instr. | Feld-/Formatanpassung | `MQTT_Telemetry_String_Formatter` | 0.958 |
| MQTT BMS-Version | 72 Instr. | angepasst | `MQTT_Publish_BMS_Version` | 0.903 |
| Telemetrie-Historie Init | 20 Instr., gleiche Größe | umgebaut | `Telemetry_History_Buffer_Init` | 0.85 |
| Telemetrie-Historie Push | 85 Instr., gleiche Größe | umgebaut | `Telemetry_History_Record_Push` | 0.859 |
| HTTP Header/Body-Trenner | 20 Instr., gleiche Größe | 5 Ops geändert | `HTTP_FindHeaderBodySeparator` | 0.75 |
| CH395 UDP-Extraktion | 134 Instr., gleiche Größe | angepasst | `CH395_ExtractUdpData` | 0.97 |
| HTTP GET-URL-Daten | 241 Instr., gleiche Größe | angepasst | `HTTP_GetUrlData` | 0.983 |
| HTTP Client-Task-Verteiler | 910 Instr., gleiche Größe | angepasst | `HTTP_ClientTask_Dispatcher` | 0.992 |

## Kleinänderungen (kein Verhaltensunterschied)

**`__LINE__`-Verschiebungen (Logmakro).** Folgende Paare unterscheiden sich nur in **einer**
`movw`/`adds`-Konstante, die als Zeilennummer an den Logger (`Log_Printf`) übergeben wird — verschoben durch
darüber eingefügte Quelltextzeilen:
`AT_Copy_MPPT_Channel_Data`, `CH395_Socket_HandleStatusEvents`, `HTTP_Cloud_Reporting_Dispatcher`,
`HTTP_ParseData`, `HTTP_PackGetRequest`, `HTTP_ParserOnBody`, `HTTP_ParserOnMessageComplete`,
`Quectel_GetDeviceIP`, `Meter_GetMeterIP_FromResponse`, `Parse_IP_Address_String`,
`Device_Network_Info_Init`, `CAN_CommLink_Timeout_DataClear`, `ShellCmd_get_ver`, `__scatterload`
(Scatter-Tabelle).

**Literal-Pool-/Adressverschiebung durch Neulinkung.** Kein Instruktionsverhalten geändert, nur Konstanten-
oder Symboladressen um wenige Bytes verschoben, weil an anderer Stelle im Image Code/Daten eingefügt wurden:
`fp64_abs`, `pow` (beide nur verschobene `undefined4`-Literale), `llhttp__internal__run` (ein Padding-Byte
wird in v149.2 als Sprung disassembliert, Funktionsgröße identisch), `SPI_WriteByte` (ein Füllbyte für
Alignment). `MQTT_Transport_NullCallback` und `Stub_ImmediateReturn` sind zwei 2-Byte-Stubs (`bx lr`), die im
Speicher direkt hintereinander liegen; der scheinbare Unterschied ist nur die benachbarte Literal-Pool-Daten
(Firmware-Versionsstring „VNSD-0", Versionsnummer), die je nach Ausrichtung der einen oder anderen Funktion
zugerechnet werden — Funktionskörper selbst sind in beiden Ständen identisch (`return;`).

`BLE_Build_Cmd0x42_Response` unterscheidet sich nur dadurch, dass ein Zielfeld in v149.2 keinen Symbolnamen
erhalten hat (`UNK_20018f8d` statt `g_bleCmd0x42Resp_calcVal0`) und die Adressberechnung dafür mit mehr
Instruktionen erfolgt — reine Layout-/Benennungsverschiebung, keine Verhaltensänderung.

## Unverändert (explizit geprüft)

Modbus komplett (Registermap-Init, Dispatcher, Deskriptortabelle 246 Einträge — nur Quelladressen im
Rohbyte-Vergleich verschoben), Einspeisegrenzen (`Grid_Export_Power_Limiter`, `Config_ClampDefaults_ParamBlock`
byte-identisch), BLE-Befehls-IDs, kompletter OTA-/BLE-OTA-/YMODEM-Pfad, alle Cloud-Domains
(`hamedata.com`-Endpunkte), 1873 von 1931 gepaarten Funktionen byte-identisch.

## Changelog → Code

| Changelog-Punkt (v149.2) | Befund im Code |
|---|---|
| Fehlerhafte Datenanzeige durch ausbleibende HTTP-Uploads behoben | Kein einzelner isolierter Fix gefunden. Passt am ehesten zur umgebauten HTTP-/Cloud-Telemetriekette (`HTTP_Task_StateMachine_Handler`, `HTTP_ClientTask_Dispatcher`, `Cloud_UploadTelemetryData`, `Ethernet_SendStatusTelemetry`, `Modem_SendStatusTelemetry`, `MQTT_StatusReport_BuildAndPublish`) — genaue Ursache innerhalb dieser großen Funktionen nicht isoliert, siehe Offene Punkte |

## Offene Punkte

- Genaue Ursache des Changelog-Punkts „Fehlerhafte Datenanzeige durch ausbleibende HTTP-Uploads" nicht im
  Code isoliert (Kandidatenfunktionen siehe Changelog-Tabelle).
- Bedeutung der ungenutzten Hälften des neuen BMS-Pack-Feldpaares bei Offset 0x58: `g_bmsPack0Reserved58`
  (untere 16 Bit des ersten 32-Bit-Werts) und das komplette zweite 32-Bit-Feld bei Offset 0x5c
  (`g_bmsPack0Reserved5c` + oberes Halbwort) werden in v149.2 von keiner anderen Funktion gelesen.
- Feininhalt der großen Umbauten in `HTTP_Task_StateMachine_Handler`, `MQTT_CtrlCommand_Dispatcher` und
  `BLE_App_Command_Dispatcher` (2500–3000 Instruktionen) über die im Dokument belegten Punkte hinaus nicht
  vollständig instruktionsweise nachvollzogen.

## Datenbasis

- Dekompilat + Listing beider Stände: `Dekompilate/VNSD_Control_149/`, `Dekompilate/VNSD_Control_1492/` (lokal, nicht im Repo)
- Klassifikation: vollständiger namensgepaarter Funktionsdiff `scratchpad/v151/l149_1492.json` (lokal, nicht im Repo)
- Modbus-Deskriptoren: `scratchpad/v151/mbdesc_149.json`, `scratchpad/v151/mbdesc_1492.json`
- Firmware-Changelog: `Marstek FW Archiv/firmwares/VNSD-0/Control/CHANGELOG.de.md` (lokal, nicht im Repo)
