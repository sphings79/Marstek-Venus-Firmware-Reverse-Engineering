# Diff-Tabelle Control v150 → v151 (Venus D)

> 🇬🇧 English version: [VNSD_Control_Diff_v150_v151_de.md](VNSD_Control_Diff_v150_v151_de.md)

**Stand:** 2026-09-30. Beide Images mit korrekter Basis 0x08004800 in Ghidra, vollständig benannt.
Adressen: v150 → v151. Changelog-Nummern siehe Hauptdoku §7.
Funktionspaare über gleiche Namen (1948), instruktionsweise verglichen: 57 geändert (davon 29 inhaltlich),
6 nur in v150, 8 nur in v151. Die 6 v150-Funktionen sind reine Linker-Artefakte (identical-code-folding /
2-Byte-Stubs), siehe „Kosmetische Unterschiede“.
**Englisch:** `VNSD_Control_Diff_v150_v151_de.md` · **Hauptdoku:** `VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md`

## Prüfliste

| Frage | Ergebnis |
|---|---|
| Modbus-Register | Lesetabelle identisch außer 32102 (`float32` → `i16`); neues Schreibregister 0xB799 (47001); ein zusätzlicher indizierter Schreib-Slot 0xAFE6 im hohen Registerblock |
| BLE | Befehlssatz gleich, 3 Detailänderungen (0x0F, 0x54, 0x59), LED-Status persistent |
| AC-Einspeisung 2500 W | konfigurierbare Grenzen unverändert; PV-Startwert beim Einspeisen ohne 2500-W-Klemme |
| BLE-OTA | weiter möglich, zusätzlich YMODEM-Typprüfung |
| Cloud-Domains | `getDateInfo`: hamedata → `api-<region>.marstekcloud.com`; alle anderen Cloud-Hosts unverändert |

## Changelog v151 → Code

| Changelog-Punkt | Befund im Code |
|---|---|
| Modbus-TCP-Stabilität | `CH395_ConfigTcpKeepalive_Defaults`: IDLE 10 000 → 60 000 ms; `CH395_Init_Modbus_TCP_Socket` wendet die Keepalive-Defaults jetzt auch auf den Modbus-Socket an; `HTTP_ClientTask_Dispatcher`: CH395-Hänger-Watchdog von Soft-Reset (Rückstau > 1, 1800 Ticks) auf gestaffelte Reaktion umgebaut (Rückstau ≥ 5, 14 400 Ticks, dann `System_Reboot()`, neues Unterdrückungs-Flag) |
| DOD kann wegen abnormalem Status nicht entladen werden | `Battery_EPS_Reserve_SOC_Hysteresis`: Hysterese läuft nur noch bei SoC ≠ 0, EEPROM-EPS-Flag wird bei Erholung nicht mehr automatisch auf 1 zurückgeschrieben; `CT_GridPower_Controller`: neuer Guard setzt einen hängengebliebenen positiven CT-Sollwert bei gesetztem Flag `DAT_200002B2` explizit auf 0 zurück; `MQTT_StatusReport_BuildAndPublish` meldet dazu passend ein neues Feld `,df=%d` (vermutlich DOD-/Discharge-Fault-Status, siehe „Offene Punkte“) |
| YMODEM-Upgrade-Typprüfung | `YModem_ReceiveFile` klassifiziert den Dateinamen über die neue Funktion `OTA_ClassifyDeviceTypeTag` und bricht bei Typ 0 (unerkannt) oder Mismatch mit CAN-0x18-Meldung und Fehlercode −5 ab; `YModem_FirmwareUpgrade_Handler` bekommt dafür einen neuen Meldungszweig „Bin type mismatch! Upgrade aborted.“ |
| KI-Modus-Überprüfung optimiert / aktuelle KI-Strategie abfragen | `AI_ScheduleQuery_Response_Send` (cd=0x3a) sendet die AI-Zeitplan-Antwort jetzt seitenweise (`pg=%d/%d`, 24 Einträge/Seite, Bounds-Check je `snprintf`) statt wie v150 alles in einem 1024-Byte-Puffer ohne Überlaufprüfung |
| URL-Fehler des Vorabversionsservers (Datenupload-Fehler) | `HTTP_Cloud_Reporting_Dispatcher` und `HTTP_GetUrlData`: `getDateInfo`-Host von `hamedata.com` auf `api-<region>.marstekcloud.com` umgestellt; Regionstabelle EEPROM 0x441 von 3 (eu/www/us) auf 5 gültige Werte (+ dev/staging) erweitert und Sanitierung entsprechend angepasst; `Config_LoadOrInit_DeviceSettings` lädt den Servertyp jetzt schon beim Boot statt nur bei Bedarf |
| HTTP-Upload-Daten-Stabilität optimiert | `Quectel_HTTP_POST_SendAndVerify`: aus zwei blinden `vTaskDelay`s wird eine echte, wiederholt geprüfte Tick-Schleife mit ERROR-/`+QIURC`-Erkennung und exakter Byteanzahl (`UART_TransmitBytes` statt NUL-terminiertem Senden); `HTTP_Task_StateMachine_Handler`: `AT+QHTTPPOST`-Timeouts 60/60 s → 120/120 s, äußerer Timeout 600 → 6000, und die Postlänge wird jetzt explizit berechnet statt (in v150) aus einem nicht gesetzten Register übernommen; `prvHeapInit`: FreeRTOS-Heap 70 KB → 80 KB (plausibel für die größeren HTTP/JSON-Puffer) |
| CT-Kommunikationsfrequenz auf 1,1 s optimiert | `Modem_SendStatusTelemetry` und `Ethernet_SendStatusTelemetry` laufen jetzt in einem gemeinsamen Zeitfenster von höchstens alle 1100 Ticks (1,1 s, Zeitstempel `DAT_20001F58`); `Meter_ParseUDPMeterInfo` entfernt dafür das alte, blockierende `vTaskDelay` nach erfolgreichem Parse und zählt stattdessen nur noch einen Diagnosezähler hoch |

## Änderungen

| Bereich | v150 | v151 | Fundstelle (v150 → v151) | CL |
|---|---|---|---|---|
| Versions-String / Build | `VNSD-0 v150`, 05.08.2026 11:51:21 | `VNSD-0 v151`, 29.09.2026 09:42:27 | Strings | – |
| Versionskonstante (Cloud-URLs, 0x08010000) | 0x96 (150) | 0x97 (151) | `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData` (derselbe Literal auch in `ShellCmd_get_ver`, `Cloud_UploadTelemetryData`, `MQTT_StatusReport_BuildAndPublish`, `Config_LoadOrInit_DeviceSettings`) | – |
| `getDateInfo`-Host | `http://%s.hamedata.com/app/neng/getDateInfo%s.php` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php` (HTTP, Port 80) | 080124F4 → 08012584, 080199AC → 08019A70 | 5 |
| Regionstabelle (EEPROM 0x441) | eu, www, us, eu-dev, eu-staging; gültig 0–2 | eu, www, us, **dev**, **staging**; gültig 0–4 | `g_cloudServerRegionTable`, `HTTP_GetUrlData` | 5 |
| Servertyp beim Start laden | nur bei Bedarf | beim Start (`Config_LoadOrInit_DeviceSettings`) | 0802DCBC → 0802E17C | 5 |
| Geräte-Quell-ID (Cloud) | `VenusD-<SN>` | `VNSD-0-<SN>` | `Cloud_BuildDeviceSourceID` 08034BDC → 080351F0 | – |
| CH395-Keepalive IDLE | 10 000 ms | 60 000 ms (CNT 5, INTVL 3000 ms gleich) | `CH395_ConfigTcpKeepalive_Defaults` 08037A0C → 080380DC | 1 |
| Keepalive vor Modbus-Socket 502 | nein | ja | `CH395_Init_Modbus_TCP_Socket` 0804DCD0 → 0804E3A0 | 1 |
| CH395-Hänger-Watchdog | Rückstau > 1, 1800 Ticks, `CH395_Reset_And_Reinit` | Rückstau ≥ 5, 14 400 Ticks, **`System_Reboot()`**, Unterdrückungs-Flag | `HTTP_ClientTask_Dispatcher` 0801A3D0 → 0801A4A4 | 1 |
| HTTP-POST über FC41D | 2 × blindes `vTaskDelay(timeout)`, 400-Tick-Wartezeit, kein ERROR-Check, NUL-terminiertes Senden | Tick-Schleife, ERROR/`+QIURC`, exakte Byteanzahl (`UART_TransmitBytes`) | `Quectel_HTTP_POST_SendAndVerify` 08010598 (gleiche Adresse) | 6 |
| `AT+QHTTPPOST`-Timeouts | 60, 60 s | 120, 120 s; äußerer Timeout 600 → 6000 | `HTTP_Task_StateMachine_Handler` 08010C20 → 08010CAC | 6 |
| FreeRTOS-Heap | 0x11800 (70 KB) | 0x14000 (80 KB) | `prvHeapInit` 0804FDC0 → 08050528 | (6) |
| Zählerabfrage (Shelly `EM.GetStatus`/`EM1.GetStatus`, sonstige) | jeder Aufruf | höchstens alle 0x44C = 1100 Ticks (1,1 s), Zeitstempel 0x20001F58 | `Modem_SendStatusTelemetry` 08031678 → 08031B90, `Ethernet_SendStatusTelemetry` 08031814 → 08031D80 | 7 |
| UDP-Zählerantwort | blockierendes `vTaskDelay` nach Erfolg | Zähler statt Delay | `Meter_ParseUDPMeterInfo` 0802BE88 → 0802C348 | 7 |
| Quectel-Modem-Byteassembler | kein Aufruf-/Bytezähler | neuer Zähler `DAT_2001D0A6` bei jedem Aufruf, Statuspuffer um 2 Byte gewachsen | `Quectel_AT_Recv_ByteAssembler` 080185CC → 08018670 | – |
| YMODEM-Upgrade | keine Typprüfung | Dateiname → Typ (`OTA_ClassifyDeviceTypeTag`), Fehler −5 „Bin type mismatch! Upgrade aborted.“ | `YModem_ReceiveFile` 0803548C → 08035AA4, `YModem_FirmwareUpgrade_Handler` 08032000 → 080325B8 | 3 |
| KI-Strategie-Antwort (MQTT) | ein `snprintf` ohne Begrenzung | seitenweise Sendung | 08023C80 → `AI_ScheduleQuery_Response_Send` 08023E08 | 4 |
| Modbus-Schreiben | – | neues Register **0xB799**, neuer Aufruf `CAN_Write_Category0x18()`; im hohen Registerblock zusätzlich ein eigener Schreib-Case für **0xAFE6** (`Config_Apply_SingleReg`), der in v150 nur über eine Bereichsprüfung erreichbar war | `Write_Handler` 08056514 → 080570E0 | 4 |
| MQTT `cd=0x42` „Get INV DEBUG info“ | Case existiert nicht | neuer Case mit Unterwert `debug=N`: 1 = Wartungsmodus an (`g_maintModeState=2`), 2 = `MQTT_Publish_WorkMode_Config` publiziert den `g_vns_inv_debug`-Status, 3/4 = `CAN_Write_Category0x18(1)`/`(0)` schaltet CAN-Kategorie 0x18 und beendet den Wartungsmodus wieder | `MQTT_CtrlCommand_Dispatcher` 0802423C → 080244E0; neu: `CAN_Write_Category0x18` (08032004), `MQTT_Publish_WorkMode_Config` (08027C0C) | – |
| Shell-Befehle `read_inv1`/`read_inv2` | nicht vorhanden | `read_inv1` setzt denselben Wartungsmodus (`g_maintModeState=2`) wie MQTT `cd=0x42,debug=1`; `read_inv2` gibt alle Felder von `g_vns_inv_debug` auf der Konsole aus | 08053328, 08053334 (nur v151) | – |
| MQTT `cd=0x13` | – | Unterbefehl `debug` → `MQTT_ConnectionStatus_Report_Send` (Verbindungs-/Zähler-Statusbericht: `ct_st, conn_st, err, wifi, send_ok, recv_ok, at_fail, no_resp, no_reply, sock, ip, port, reason`) | `MQTT_CtrlCommand_Dispatcher` 0802423C → 080244E0; neu: `MQTT_ConnectionStatus_Report_Send` (08024244) | – |
| CT-Sollwert hängt positiv | – | Sollwert → 0 bei neuem Flag 0x200002B2 | `CT_GridPower_Controller` 08030E80 → 08031378 | 2 |
| EPS-Reserve-Hysterese | läuft auch bei SoC 0 %; setzt EEPROM-EPS-Flag bei Erholung auf 1 | nur bei SoC ≠ 0; kein automatisches Zurücksetzen | `Battery_EPS_Reserve_SOC_Hysteresis` 0800AD3C | 2 |
| MQTT-Statusbericht | – | neues Feld `,df=%d` | `MQTT_StatusReport_BuildAndPublish` 0802A5DC → 0802AA98 | 2 |
| PV-Startwert beim Einspeisen | `min(P_bms, 2500 W)` | `Netz-Messwert − P_neg`, ohne 2500-W-Klemme (4000-W-Gesamtklemme gleich) | `Grid_Export_Power_Limiter` 08023A00 → 08023B94 | – |
| Netz-Sync-Übernahme | Commit-Bedingung greift nur bei genau 2 abweichenden Phasen (Zähler == 2); Settled-Average-Gate verlangt zusätzlich „irgendeine Phase war je ungleich 0“ | Commit-Bedingung greift ab 2 abweichenden Phasen (Zähler < 2 negiert, also auch 3 möglich); die zusätzliche „je ungleich 0“-Bedingung im Settled-Average-Gate entfällt | `Grid_PowerSync_StateMachine` 08031B60 → 08032114 | – |
| Protocol-AA-Systemparameter (CAN-Register 0x11) | ein festes Wertepaar | Index aus `reg_id`, 7-Slot-Tabelle mit Grenzprüfung (Register 0x10 nutzt schon seit v147 das indexierte `Protocol_AA_SetChannelData`) | `Protocol_AA_SetSystemParams` 08034428 → 08034A18 | – |
| Economy-TOU-Antwort (Prüfsumme) | eine XOR-Prüfsumme, ein Vergleich | dieselbe XOR-Prüfsumme wird zusätzlich ein zweites Mal in eine eigene Variable berechnet, Annahme bei Treffer auf **einem der beiden** (rechnerisch identisch); Fehlermeldung um Feld `crc2` erweitert | `HTTP_Economy_TOU_ParseResponse` 080194CC → 08019578 | – |
| LED-Status (SEL LED) | nur RAM | EEPROM 0x395 (BLE `Y`, MQTT RPC 0xE, beim Start geladen) | BLE-/MQTT-Verteiler, `Config_LoadOrInit_DeviceSettings` | – |
| BLE EPS ein/aus (0x0F), Entladegrenze (0x54) | setzen EPS-Reserve-Hystereseflags zurück (`g_epsReserveSocHysteresisState`, bei 0x54 auch `g_epsReserveActiveFlag`) | nicht mehr | `BLE_App_Command_Dispatcher` | – |
| Modbus 32102 | `float32` (2 Register) | `i16` (1 Register) | `g_modbusRegDescTable` | – |
| Deskriptortabelle | 0x20000354 | 0x2000034C | .data | – |
| Shell-Befehle (Gesamtzahl) | 68 (+ help, clear) | 70: neu `read_inv1`, `read_inv2` (siehe oben) | `shellCommandTable` 0x08062CC4 | – |

## Kosmetische Unterschiede (keine Verhaltensänderung)

Reine Adress-/Register-Relokation, ohne inhaltliche Änderung (Ghidra-Diff zeigt trotzdem "changed", weil
Konstanten/Offsets nicht symbolisiert werden konnten): `BLE_Pending_Commands_Process`, `CH395_ExtractUdpData`,
`Device_Network_Info_Init`, `HTTPS_FetchUrlData` (nur Variablennamen), `Meter_GetMeterIP_FromResponse`,
`MQTT_Publish_WiFi_Version`, `Quectel_AT_QVERSION_Query`, `Quectel_GetDeviceIP`, `Quectel_SendCmd_WaitResponse`,
`Quectel_UDP_Socket_Open`, `ShellCmd_get_ver` (nur der Versions-/Datumsliteral, s.o.), `ShellCmd_mac_ble`,
`ShellCmd_trace_debug`, `Util_RollingHistory_ShiftAndAverage_1s`, `debug_printf`, `pow` (Standard-C-Bibliothek).

`BLE_Set_Device_Name` und `MQTT_Publish_PowerSchedule`: reine Decompiler-/Registerallokations-Unterschiede,
erzeugter Code bzw. Verhalten identisch.

`SMR_TIC_SelfUse_PowerController`: zusätzlich zur Relokation ein neuer, aber wirkungsloser Bounds-Guard vor dem
`switch` (kein State überschreitet ohnehin den geprüften Bereich).

`SPI_WriteByte`, `Util_NoOp_Stub`: die als "insert"/"delete" markierten Byte-Änderungen sind benachbarte
Literale (Strings „staging“/„dev“ bzw. der Versions-Literal/„VNSD-0“-String) im Flash-Layout, nicht Teil dieser
Funktionen selbst.

`__scatterload_copy`: übernimmt in v151 den Kopier-Loop-Body inline, der in v150 als separates Symbol
`Scatterload_CopyBody` existierte (Linker-ICF, siehe unten) – Verhalten identisch.

**Nur in v150 (6, alle Linker-Artefakte, keine Funktionsänderung):**
- `Fault_InfiniteLoopTrap`, `Stub_InfiniteLoop_1/2/3`, `Stub_ImmediateReturn` – 2-Byte-Endlosschleifen- bzw.
  Sofort-Return-Stubs, die der Linker in v150 als eigenständige Symbole belässt (identical-code-folding), in
  v151 aber anders zusammenfasst bzw. durch einen gleich winzigen Stub anderen Namens ersetzt
  (`Stub_ImmediateReturn` → `Util_NoOp_StartupHook`, beide No-Ops, aufgerufen aus `Board_SystemStartup`).
- `Scatterload_CopyBody` – geteilter Kopier-Loop-Body, in v151 inline in `__scatterload_copy` übernommen.
- Ebenso rein kosmetisch: `BusFault_Handler`, `HardFault_Handler`, `MemManage_Handler`, `UsageFault_Handler`
  wachsen von 2 auf 4 Byte (ein zusätzliches `b`-Sprungziel), bleiben aber dieselbe Endlosschleife ohne
  Recovery – auch hier nur andere Linker-Faltung des gemeinsamen Trap-Codes.

## Unverändert (explizit geprüft)

Modbus-Lesepfad (`FC03_Read_Handler`, `Read_Serializer`), 32101-Vorzeichenfehler, MBAP-Länge 4 bei Exceptions,
44002/44003-Schattenwert, CR-LF-Empfangsschleife, 86-s-Drossel, WLAN-Zwilling 900 s, TLS authmode 0 beim HTTPS-Upload,
Task-Prioritäten/Stacks, BLE-OTA, alle konfigurierbaren 2500-W-Grenzen, Debug-Register 46000 (0x5100), FC03-Pufferüberlauf.

`Task_udp`: Aufrufreihenfolge (`Cloud_PassiveMode_Connection_StateMachine` → `CH395_UDP_ServerTask` →
`vTaskDelay(100)` → `Util_NoOp_Stub`) ist in v150 und v151 byte-identisch; `Cloud_PassiveMode_Connection_StateMachine`
selbst war in v150 nur unbenannt (`FUN_08015b84`) und wurde bei der Namens-Harmonisierung dem v151-Gegenstück
zugeordnet – **keine** Verhaltens- oder Reihenfolgeänderung (korrigiert gegenüber einer früheren Fassung dieses
Dokuments, die hier fälschlich eine geänderte Aufrufreihenfolge behauptet hatte).

`MQTT_Config_ExtractFields`: die Tabelle enthält bereits in v150 6 Felder inklusive `to=` (String „to=“ bei
0x0805FC18, Schleife `uVar2 < 6` identisch in beiden Versionen); die Funktion ist unverändert, nur die
Datentabelle liegt an anderer Adresse (v150 0x0805BE5C, v151 0x0805CA7C). Die Behauptung eines neuen 6. Feldes
in v151 in einer früheren Fassung dieses Dokuments war falsch.

BLE „Get err code info“ (0x13): Antwort in v150 und v151 identisch (kein Unterschied, daher keine eigene Zeile
mehr in der Änderungstabelle).

## Offene Punkte

- Genaue Bedeutung von `df=%d` in `MQTT_StatusReport_BuildAndPublish`: plausibel ein DOD-/Discharge-Fault-Status
  im Zusammenhang mit der `CT_GridPower_Controller`/`Battery_EPS_Reserve_SOC_Hysteresis`-Änderung, aber Bit-/
  Wertebedeutung nicht am Code verifiziert.
- `Write_Handler`: Die internen Sprungtabellen konnten in beiden Versionen von Ghidra nicht rekonstruiert
  werden („too many branches“) und die Funktion ist um 4 Byte gewachsen; ein lückenloser Soll-Ist-Vergleich war
  im verfügbaren Rahmen nicht möglich. Die genannten Kernänderungen (Register 0xB799, 0xAFE6-Slot,
  `CAN_Write_Category0x18`) sind am Code verifiziert, weitere kleine Abweichungen in unbeteiligten Zweigen
  können nicht ausgeschlossen werden (mittlere statt hohe Konfidenz).
- Zusätzlicher Initialisierer `DAT_2001A8AA = 0xA892` in `Config_LoadOrInit_DeviceSettings` (v151, 2 Byte nach
  dem „VNSD-0“-String im Geräte-ID-Setup) – Zweck aus dieser Funktion allein nicht erkennbar.
- Ob das neue MQTT-Kommando `cd=0x42` („Get INV DEBUG info“, Wartungsmodus + `MQTT_Publish_WorkMode_Config` +
  `CAN_Write_Category0x18`) einem der sieben Changelog-Punkte zuzuordnen ist oder eine separate, dort nicht
  gelistete Ergänzung ist, ist offen.
- Grund für die verdoppelte (aber rechnerisch gleichwertige) CRC-Berechnung in `HTTP_Economy_TOU_ParseResponse`
  ist aus dem Code nicht ersichtlich.
