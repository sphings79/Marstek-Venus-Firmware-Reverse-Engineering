# Diff-Tabelle Control v147 → v149 (Venus D) — Haupthop

> 🇬🇧 English version: [VNSD_Control_Diff_v147_v149_de.md](VNSD_Control_Diff_v147_v149_de.md)

**Stand:** 2026-09-30. Beide Images mit Basis 0x08004800, alle Prüfschritte bestanden (`marstek_gates.py`, lokal, nicht im Repo).
Adressen: v147 → v149. Funktionspaare über gleiche Namen (1906), instruktionsweise verglichen:
**124 geänderte Funktionen** (davon 75 inhaltlich, 49 rein kosmetisch), 2 nur in v147, 25 nur in v149;
Inhalte der Änderungen per Code-Review (Offline-Agenten, Kernaussagen im Code nachgeprüft).
**Englisch:** `VNSD_Control_Diff_v147_v149_de.md` · **Nächster Hop:** `VNSD_Control_Diff_v149_v1492_de.md`, `VNSD_Control_Diff_v149_v150_de.md`

## Prüfliste

| Frage | Ergebnis |
|---|---|
| Modbus-Register | **15 neu**: 38000–38014 (kumulierte Energiezähler `g_energyCounterAt*`, `g_energyRecA/B/C_*`); keine entfernt, Typ/Skalierung/Anzahl der übrigen 231 unverändert (per Byte-Vergleich der Deskriptor-Einträge bestätigt — nur die RAM-Quelladresse verschiebt sich). Deskriptortabelle wächst von 0xe7 (231) auf 0xf6 (246) Einträge. Lesen von 38000–39014 setzt in v149 zusätzlich `g_canCat40to43PollEnableFlag` (CAN-Kategorie 0x40–0x43; Bereich `37999 < reg < 0x9867` im Code verifiziert). Schreibpfad: neuer Block 46500–46544 (0xB5A4–0xB5D0, 15 × 3 Register → `Inverter_Set_Schedule_Reg`/`Config_Param0A_Relay`, EEPROM 0x36BF+8·n) |
| BLE | Befehlssatz **erweitert**: neu 0x5A (Cloud-Backup-Abruf anstoßen, `bk_en`), 0x5B (IP/MAC/Seriennummer); 0x54 (Entladegrenze) setzt zusätzlich EPS-Reserve-Flags zurück; Antworten/Statusfelder ergänzt (`BLE_Build_DeviceStatus_Response`: +Power-Limit-Flag, +PV-Kanal-Delta, Antwortlänge 0xa1→0xa4 Byte) |
| AC-Einspeisung / Leistung | **2200 W → 2500 W**, durchgängig in mindestens 12 Funktionen umgesetzt (Clamp-Defaults, EEPROM-Defaults, `Config_Write_U16`/`_0x202`, `Grid_Export_Power_Limiter`, `Battery_Charge_Power_Limiter`, `Remote_Power_Command_Execute`, MQTT-/BLE-/JSON-RPC-Setter, `Write_Handler`-Ladelimit) — siehe Abschnitt „Leistungsgrenze 2200→2500 W"; `Config_Apply_WorkModePowerPreset` (800/600/2200/1200 W je Modus) entfällt ersatzlos; neue RPC-Methode `Set.Ver` (JSON-RPC 0x10) wählt 800 / 2200 / 2500 W |
| BLE-OTA | **weiter möglich** — `BLE_Cmd_OTA_*`, YMODEM und alle `OTA_*`-Funktionen vollzählig und namensgleich in beiden Versionen vorhanden; YMODEM-Zielcodes gleich (0x04D2 → EMS, 0x0929 → VENUS) |
| Cloud-Domains | **unverändert** (`%s.hamedata.com`, in beiden Images vorhanden); neu nur der Pfad `/external-services/api/v1/devices/backup-configs?devid=%s&latest=true` |

## Changelog v149 → Code

| Changelog-Punkt | Befund im Code |
|---|---|
| weitere OpenAPI-Funktionen | `MQTT_JSON_RPC_Dispatcher`: **11 → 16 Methoden** (Methodentabelle `memcpy` 0x58→0x80 Byte = 11→16 Einträge à 8 B; Fall 0 ist „Method not found“, Methoden 1–0xB → zusätzlich 0xD–0x11). Neu: `Ble.Adv` 0x0D (BLE-Advertising-Enable, EEPROM 0x36bd), `Led.Ctrl` 0x0E (`g_selLedEnable`), `Ota.Set` 0x0F (OTA-URL-Slot: Device-Type/Mod/Version/URL/Size/CRC, Flash-Schreiben je Slot, `g_otaUpdatePendingFlag`), `Set.Ver` 0x10 (Work-Mode/Preset 800/2200(0x898)/2500(0x9c4) W), `Reset.Factory` 0x11 (Typ 1/2/3, `Factory_Reset`) — Methodennamen aus dem Stringpool verifiziert. Zusätzlich (nicht im Changelog erwähnt): dedizierte, nicht mehr überlappende Antwortpuffer je Methode (v147 nutzte einen gemeinsamen Puffer mit Offsets) |
| ES liefert PV = 0 | `Cloud_Report_FillGridData`: v147 schreibt PV-Summe (+0x27) und PV-Jahreskapazität (+0x31) fest 0; v149 rechnet sie aus den 4 MPPT-Kanälen bzw. `g_mpptPvYearCap10Wh` |
| I²C-Deadlock-Recovery | `EEPROM_I2C_ReadBytes/WriteBytes`: Timeout 3000 → 0x64000 (409600), neuer Aufruf `EEPROM_I2C_BusRecovery()` in den Fehlerpfaden. Die Recovery-Kette besteht aus drei neuen Funktionen: `EEPROM_I2C_BusRecovery` (loggt, ruft die beiden folgenden auf plus `EEPROM_I2C_Peripheral_Init`), `EEPROM_I2C_ClockDisable_Reset` (I2C1-Periph-Reset, APB1-Takt aus, AFIO-Remap zurück) und `EEPROM_I2C_GPIO_ClockPulseRecovery` (SCL 9× als GPIO gepulst — klassisches Bus-Recovery für einen hängenden Slave) |
| französische Zähler | `SMR_TIC_SelfUse_PowerController`: komplett neue tick-getaktete Zustandsmaschine (States 0–5) für `g_meterSourceType == 0x0A` (TIC/Linky), Setpoint aus `g_smrPowerDeltaAvg`; `CT_GridPower_Controller` bekommt eine neue Ramp-Up-Phase für Überschusseinspeisung und überspringt für Meter-Typ 10 die Delta-Erkennung; `Grid_PowerSync_StateMachine` akzeptiert Zählertypen 9/0x0A/0x0C zusätzlich, eigener 60-Zyklen-Timeout für Typ 12, invertierte Timer-Bedingung bei der Baseline-Erfassung (wirkt wie ein Bugfix) und entfernte Obergrenze bei der Abweichungserkennung |
| HTTP-Upload + IP + kumulierte Energie | `Cloud_UploadTelemetryData`: lokale IP (Ethernet oder aus `g_devNetInfo`) im neuen Feld `ip=%s`, dazu neues Feld `bkd=%s` (passend zum neuen Backup-Kommando 0x5A); Energiezähler auch als Modbus 38000–38014; Upload-Puffer 0x44c→0x4b0 Byte, Telemetrie-Verlaufs-Stride 0x30b→0x35b Byte |
| Überschusseinspeisung optimiert | `Battery_Charge_Power_Limiter`: Ergebnis von `Power_ScaledValue_Calc_Negated()` wird jetzt auf minimal −2500 W (`-0x9c4`) geklemmt (in v147 unbegrenzt); `CT_GridPower_Controller`: neue Ramp-Up-Phase über `Inverter_Power_Setpoint_Calc(1,10)` beim Aktivieren |
| DOD falsch gesendet | `Cloud_DOD_Command_Handler`: Bereichsprüfung 30–88 % (0x1e–0x58; v147 echote nur den gelesenen Wert), Antwortformat auf `cd=%d,ret=0/1` umgestellt; `Config_Set_DischargeCutoff_WithRelay` nutzt jetzt den neuen Helfer `Config_Param0A_Relay` statt `Inverter_Set_Schedule_Reg` direkt (funktional gleichwertig, ausgelagert weil `Inverter_Set_Schedule_Reg` in v149 eine reg-Bereichsprüfung 0x30–0x3e bekam, die den alten CAN-Reg-0x0A-Pfad abgelehnt hätte) und normalisiert den Rückgabewert auf 0/1; BLE 0x54, MQTT 0x38 (`Cloud_DOD_Command_Handler`) und MQTT 0xb (EPS enable/disable) setzen vorher `g_epsReserveSocHysteresisState`/`g_epsReserveActiveFlag` zurück |
| HTTP-Upload-Intervall 5 min | im Code **nicht als Unterschied gefunden** (HTTP-Task-Timer in beiden Versionen gleich) |
| max. Leistung konfigurierbar / 2500 W | siehe Prüfliste und Abschnitt „Leistungsgrenze 2200→2500 W" |
| VA-Zellspannung | betrifft Venus A, nicht geprüft |

## Leistungsgrenze 2200 W → 2500 W (funktionsübergreifend)

Eine einzelne Grenzwertanhebung, konsistent an folgenden Stellen umgesetzt (0x898→0x9c4 bzw. Obergrenzen 0x899→0x9c5):

| Funktion | Fundstelle |
|---|---|
| `Config_ClampDefaults_ParamBlock` | Default-/Clamp-Wert für Lade-/Entladegrenze |
| `Config_LoadOrInit_DeviceSettings` | `g_maxChargePowerW`-Default beim ersten Boot |
| `Config_Write_U16` / `Config_Write_U16_0x202` | Clamp beim Schreiben von max. Entlade-/Ladeleistung |
| `Grid_Export_Power_Limiter` | neuer Clamp `power_target` auf max. 0x9c4 vor Verrechnung mit `g_gridSamplePowerW` |
| `Battery_Charge_Power_Limiter` | neue Untergrenze −2500 W (siehe Changelog-Tabelle) |
| `Remote_Power_Command_Execute` | Auto-Limit-Modus nutzt jetzt 0x9c4 statt 0x898 |
| `MQTT_CtrlCommand_Dispatcher` case 0x10/0x11 | Obergrenze beim Setzen von max. Lade-/Entladeleistung 0x899→0x9c5 |
| `BLE_App_Command_Dispatcher` cd 0x16/0x17 | dieselbe Grenze über BLE |
| `MQTT_JSON_RPC_Dispatcher` „Set.Ver" (0x10) | Preset-Werte 800/2200/2500 W |
| `Write_Handler` | Ladeleistungsgrenze im neuen Register-Block 46500–46544 |

Zusätzlich entfällt `Config_Apply_WorkModePowerPreset` (feste Zuordnung Modus 1/2/3/4 → 800/600/2200/1200 W) komplett — siehe „Entfernte Funktionen". Die Preset-Tabellen in `MQTT_CtrlCommand_Dispatcher` (case 0xf) und `BLE_App_Command_Dispatcher` (cd 0x15) wurden dabei überarbeitet: das 1200-W-Preset (0x4b0) entfällt, neu ist ein 2500-W-Preset (0x9c4); in `MQTT_CtrlCommand_Dispatcher` wird zudem ein Bug behoben, bei dem `Config_Write_U16` bisher eine hartkodierte statt der tatsächlich geparsten Leistung schrieb.

## Weitere Änderungen (nicht im Changelog)

### EPS-Reserve-Hysterese

| Funktion | v147 | v149 | Bedeutung |
|---|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` | einfacher SoC-±5-%-Vergleich, ein Flag | 3-Zustands-State-Machine (`g_epsReserveSocHysteresisState`/`…Count`), Deaktivierung nur bei `g_epsReserveEnable==1` | Deutlich robustere Entprellung beim Umschalten der EPS-Reserve |
| `MQTT_CtrlCommand_Dispatcher` (case 0xb, 0x38), `BLE_App_Command_Dispatcher` (cd 0x0f, 'T') | kein Reset | setzen `g_epsReserveSocHysteresisState`/`g_epsReserveActiveFlag` beim manuellen Enable/Disable bzw. beim Setzen des Discharge-Cutoff zurück | Hysterese-Zustand wird bei jedem externen Eingriff synchron gehalten |

### MQTT/Cloud

| Funktion | v147 | v149 | Bedeutung |
|---|---|---|---|
| `MQTT_CtrlCommand_Dispatcher` | `/App/<id>` nur gegen Topic-/Gruppen-ID geprüft | zusätzlich `strcmp("update", device_id)` setzt `g_mqttCloudUpdateRequestFlag` | Neuer Trigger für Cloud-Update-Anfrage über Topic-ID „update" |
| `MQTT_CtrlCommand_Dispatcher` case 3 | bei `time_num >= 10` keine Reaktion | sendet Fehlerantwort `cd=3,ret=0` | Neue Fehlerquittierung |
| `MQTT_CtrlCommand_Dispatcher` case 0x1e | Enable → `g_apiConnState='\x03'` | Enable → `g_apiConnState='\x01'`, neues Flag `g_localApiUdpEnable` | Neuer UDP-Schalter für lokale API |
| `MQTT_CtrlCommand_Dispatcher` case 0x3d (neu) | – | parst `bk_en=`, setzt `g_backupDataRequestFlag`, quittiert | Neues Kommando „Backup-Daten anfordern" |
| `MQTT_CtrlCommand_Dispatcher` case 0x3e (neu) | – | ruft `MQTT_Publish_DeviceIDs` | Neues Kommando „Geräte-IDs publizieren" |
| `MQTT_Connect_And_Subscribe` | kein Statusflag | setzt `g_quectelWifiMqttReadyFlag` nach erfolgreichem Connect, löscht es im Fehlerfall | Neuer „WiFi/MQTT bereit"-Merker |
| `MQTT_Server_Config_Select` | Fallback nur bei `g_mqttVidSessionState==1`; `void` | Bedingung entfällt, Fallback greift immer bei ungültiger XID; liefert `ctx` zurück | Bugfix: `ctx` konnte in v147 unkonfiguriert bleiben |
| `MQTT_Telemetry_Struct_Builder` | `g_mqttTeleCloudLinkActive = g_quectelWifiMqttReadyFlag` | zusätzlich `\|\| g_mqttSessionActive` | Cloud-Link-Status berücksichtigt aktive MQTT-Session |
| `MQTT_Telemetry_String_Formatter` | ein zusammengesetzter String | getrennte Energiezähler-Gruppen A (Kat. 4) / B (Kat. 3) plus neues Feld Batterie-Lade-/Entladeleistung | Erweiterte Telemetrie |
| `MQTT_StatusReport_BuildAndPublish` | Format endet `…,led=%d,as=%d` | zwei weitere Felder `,lf=%d,pl=%d` (Argumente weiterhin fest 0); zusätzlicher Aufruf `Inverter_PVString_Channel_ValueDelta_Calc()` | Neue Platzhalterfelder (Bedeutung offen) plus aktualisierte Kanal-Delta-Berechnung vor dem Report |
| `MQTT_Publish_Network_Info` | `ct_connect_ip` = `g_meterQueryIpStr` | `ct_connect_ip` = `g_meterResolvedIpStr` (Kurzform bleibt bei `g_meterQueryIpStr`) | Ausführlicher Report meldet jetzt die aufgelöste statt der Ziel-IP |
| `MQTT_JSON_RPC_Dispatcher` case 7 „Manual" | kein Reset | setzt `g_workModeUpsSubFlag=false`, nach Speichern eines Zeitfensters zusätzlich `g_aiSchedule9Enable='\0'` | State-Reset beim Wechsel auf manuellen Modus (Bedeutung von „Schedule 9" offen) |
| `Quectel_MQTT_ConnectionWatchdog_Tick` | Passive-Mode-Timeout-Check nur bei `!g_ethLinkActive` | läuft unabhängig vom Ethernet-Link (nur noch an `g_bleOtaBusyState` gebunden) | Wurde vorher bei aktivem Ethernet komplett übersprungen |
| `Quectel_MQTT_Session_StateMachine` | SSL/Stale-Watchdog: nur Modem-Hardware-Reset | zusätzlich `Factory_Reset(3, …)` (löscht TLS-Zertifikat + EEPROM 0x3530, `System_Reboot()`) vor dem Reset | Deutlich härtere Reaktion auf eine hängende SSL-Verbindung |
| `Quectel_MQTT_Session_StateMachine` | periodisches `MQTT_Config_VID_Clear()` alle 600 Ticks im XID-Zweig (3 Stellen) | Codepfad komplett entfernt, Rückgabewert von `Quectel_MQTT_Subscribe` nicht mehr ausgewertet | Entspricht der Entfernung von `MQTT_Config_VID_Clear` (siehe „Entfernte Funktionen") |
| `MQTT_Config_VID_Init` | wird unbedingt aufgerufen | nur wenn `g_mqttXidConfigXid` leer ist; Flash-Adresse 0x315000→0x317000, Lesegröße 0xe4→0xe5 | Verhindert unnötiges Überschreiben bereits geladener XID-Daten |
| `EEPROM_Config_Factory_Write` | Flash-Region 0x315000 | Flash-Region 0x317000 | Factory-MQTT-VID/XID-Region um 0x2000 verschoben |

**Neues Cloud-Config-Backup-Feature** (Ethernet- und Mobilfunk-Pfad): `HTTP_ClientTask_Dispatcher` und `HTTP_Task_StateMachine_Handler` erkennen jetzt `g_backupDataRequestFlag`, fragen über einen neuen State bzw. `HTTP_GetUrlData(8,…)`/`HTTP_Cloud_Reporting_Dispatcher` case 8 die URL `…/backup-configs?devid=%s&latest=true` ab und übergeben das Ergebnis an den neuen `MQTT_Config_Command_Handler` (Parser für `key=value;…`-Konfigurationskommandos: Work-Mode, BLE-Advertise, Feed-in-Flag, lokale API, Firmware-Version, max. Lade-/Entladeleistung mit 2500-W-Fallback, WLAN-Zugangsdaten, DOD, HTTP-Server-Typ, Backup-Enable, sowie bis zu 10 TOU-Zeitplan-Einträge über `MQTT_Config_ParseScheduleEntries`/`Parse_TimeString_To_HourMinute`). Kommt über HTTP-Cloud-Anfragen herein, nicht direkt über den MQTT-Client — siehe „Neue Funktionen".

### Energiezähler-Kategorie 4 (Netzeinspeisung/Grid-Sell)

| Funktion | v147 | v149 | Bedeutung |
|---|---|---|---|
| `Config_Load_RuntimeCounters` / `Config_Save_RuntimeCounters` | EEPROM-Block 0x28 Byte, nur Kategorien 1–3 geprüft/repariert | Block 0x34 Byte, zusätzlicher Reparaturblock für `g_energyCat4TotalWh`/`DayWh`/`MonthWh` | Neue vierte Energiezähler-Kategorie |
| `Energy_Stats_Accumulate_And_Save` | nur Kategorien 1–3 akkumuliert | neuer Block für Kategorie 4 (`g_gridSellEnergyWh` → `g_energyCat4*`), auch im Speicher-Trigger | Grid-Sell-Akkumulation |
| `Stats_Clear_Counters` | Scopes 1–3, memset-Block 0x28; Scope 1 nullt fälschlich `g_energyCat3MonthWh` | Scope 2 nullt zusätzlich `g_energyCat4Day/MonthWh`; Scope 1 nullt jetzt `g_energyCat3DayWh`/`g_energyCat4DayWh`; Block 0x34 | Kategorie-4-Zähler ins Reset-Schema aufgenommen |
| `Telemetry_Register_Dispatcher` | behandelt IDs 0x10–0x14, 0xc1, 0xc3, 0xcb, 0xce, 0x54 | zusätzlich IDs 0x40–0x43 → neue Funktion `Telemetry_Store_EnergyCounters` | Neue CAN-Telemetrie-Register für Kategorie 4 |
| `CAN_Broadcast_Category0x17_PackedValues` (neu) | – | packt `g_energyCat3TotalWh`/`g_energyCat4TotalWh` in CAN-Broadcast Kategorie 0x17 | siehe „Neue Funktionen" |
| `CAN_Broadcast_ClearCategories0x40to0x43` (neu) | – | vier Null-Payload-CAN-Frames für Kategorien 0x40–0x43 | siehe „Neue Funktionen" |
| `Config_LoadOrInit_DeviceSettings` | EEPROM-Energiestatistik-Block 0x28 Byte | 0x34 Byte | Direkte Folge der neuen Kategorie |

### BMS-Pack-Record (je Pack/CAN-Knoten)

Grundlage bereits bekannt und jetzt am Code verifiziert: Record wächst von 0x28×uint16 (80 B) auf
0x2C×uint16 (88 B) je Pack — `CAN_AA_StoreUnitFieldPair8` nutzt in v147 `*0x28`, in v149 `*0x2c`
(direkt im Code geprüft). Alle sechs `CAN_AA_StoreUnitFieldPair*`-Varianten sowie
`CAN_AA_StoreUnitIndexedFieldPair`, `CAN_Detect_Mismatched_Nodes`, `BMS_CellVoltage_MinMax_PerPack`,
`BMS_Version_Consistency_Check` und `MQTT_Publish_BMS_Data` ändern sich **nur** durch mitwandernde
Feldoffsets (kosmetischer Folgeeffekt, keine eigene Logikänderung). Neu in v149:

| Funktion | Bedeutung |
|---|---|
| `CAN_AA_StoreUnitFieldPair80` (neu) | schreibt ein 8-Byte-Wertepaar an Offset 0x50 des Pack-Records in `g_bmsPack0ChargePower`/`DischargePower` — neues Datenfeld je Pack |
| `ShellCmd_get_all_bms_data` | zwei neue printf-Zeilen „bat charge power"/„bat discharge power" (liest dieselben neuen Felder); zusätzlich unabhängiger Stride-Wechsel 0x50→0x58 bei den `mos_flag`-Werten (anderes Struct, Bedeutung offen) |

### Modbus/CAN-Übertragungspfad und Mobilfunk-Fallback

| Funktion | v147 | v149 | Bedeutung |
|---|---|---|---|
| `FC03_Read_Handler` / `RS485_FC03_ReadWrite_Handler` | Tabelle 0xe7 (231) Einträge, Antwort nur über CH395/Ethernet | Tabelle 0xf6 (246); neue Bedingung setzt `g_canCat40to43PollEnableFlag` bei Registerzugriff 38000–39014; Antwort zusätzlich in `g_modbusLteTxFrame` kopiert und bei inaktivem Ethernet-Link per `Quectel_AT_QISEND_HexData` über Mobilfunk gesendet | Modbus-Lesen jetzt auch per LTE möglich, neuer CAN-40-43-Poll-Trigger |
| `TCP_FC06_WriteSingle_Handler` | Antwort immer über CH395 | bei `g_ethLinkActive==false` stattdessen `Quectel_AT_QISEND_HexData` | Gleicher LTE-Fallback für FC06-Write-Antworten |
| `Task_Modbus_tcp` | ruft `Modbus_Dispatcher()` bedingungslos | nur wenn `g_ethLinkActive==true` | Ergänzung zum LTE-Fallback |
| `Quectel_AT_QISEND_HexData`, `Quectel_AT_SendRaw_WithMutex`, `Quectel_UART_RxByte_Parser`, `StrCat` (alle neu) | – | Hex-Encoding + mutex-geschütztes AT-Senden + Byte-Assembler für die Modem-UART | Infrastruktur für den LTE-Fallback, siehe „Neue Funktionen" |
| `Quectel_UART_RxByte_ISR_Handler` | Rohmodus (case 5) rief nur den HTTPS-Response-Matcher | neuer eigener case 5 puffert Bytes in `g_quectelAtRawRxBuf` mit EOL-Erkennung, benachrichtigt `hTask_Modbus_tcp`; HTTPS-Matcher wandert zu case 6; zusätzlicher Aufruf von `Quectel_UART_RxByte_Parser` | Erweiterter UART-Demultiplexer für das rohe Modbus-über-Quectel |
| `Write_Handler` | kein Registerbereich 0xb5a4–0xb5d0 | neuer Block 0xb5a4–0xb5d0 (46500–46544): 3-Felder-Tabelle `g_canScheduleRegCache`, EEPROM-persistiert (`idx*8+0x36bf`), ruft `Inverter_Set_Schedule_Reg` | Neue persistente Leistungslimit-Tabelle über Modbus (siehe Prüfliste) |
| `Inverter_Set_Schedule_Reg` | keine Bereichsprüfung von `reg_id`; sendet nur 4 CAN-Byte, `value_hi` nie benutzt | Guard `0x2f<reg_id<0x3f`; `CAN_Build_Arbitration_ID`-Rollen von `reg_id`/Konstante vertauscht; sendet 8 Byte mit vollem `value_lo`/`value_hi` | Realer Protokoll-Fix: zuvor komplett ungenutzter `value_hi`-Parameter wird jetzt übertragen |
| `Config_Param0A_Relay` (neu) | – | schreibt 1-Byte-Relais-Wert in CAN-Parameter 0x0A für eine Node-ID | siehe „Entfernte/Neue Funktionen" – Auslagerung aus `Config_Set_DischargeCutoff_WithRelay` |
| `Protocol_AA_CommandDispatch` | Dispatch-Tabelle 15 Einträge | 16 Einträge | Ein weiterer 0xAA-Befehl, Bedeutung offen |

### Grid-/Meter-Regelung (jenseits der französischen Zähler)

| Funktion | v147 | v149 | Bedeutung |
|---|---|---|---|
| `Inverter_Power_Setpoint_Calc` | PV-Summe nur aus den vier MPPT-Kanälen | zusätzlicher fixer Offset +10 (FP64) vor der Summierung; neues Flag `g_invPowerSetpointClampedFlag`, wenn der Sollwert an der Grenze anschlägt | Neuer Watt-Headroom plus Diagnoseflag für geklemmten Sollwert |
| `RTC_TimeOfDay_To5MinSlotIndex` | Minute / 10 | Minute / 5 | Divisor für das 5-Minuten-Raster geändert — wirkt sich direkt auf den berechneten Slot-Index aus |
| `HexChar_To_TimeOffsetIndex` | 3 Hex-Zeichen aus `g_touHexConfigBuf`, XOR, mod 10 | 1 Hex-Zeichen aus `g_bleMacIdChar11`, mod 5 | Komplett andere Eingabequelle/Algorithmik, Wertebereich halbiert — genauer fachlicher Effekt auf TOU-Zeitfenster offen |
| `Timer_Periodic_DispatchAll` | 9 feste Callbacks; Cat-0x17-Intervall 10000 ms; MaintMode 1→Reg 0x11, 2→Reg 0x12 (mit CAN-Clear @0x2d0 ms) | neuer Callback „EPS-SoC-Hysterese" @6000 ms; Cat-0x17-Intervall 50000 ms; MaintMode-Register vertauscht (1→0x12, 2→0x11); CAN-40-43-Clear nur noch wenn `g_canCat40to43PollEnableFlag` gesetzt | Mehrere reale Timing-/Reihenfolgeänderungen |
| `WorkMode_ChangeHandler` | `Inverter_Power_Setpoint_Calc(1,0)` bei jedem Wechsel | nicht mehr wenn `g_userWorkMode==3` | Setpoint-Reset unterdrückt für den neuen TIC-Modus (vermutet) |
| `WorkMode_Flag_Reset_And_TriggerSetpointEval` | kein Reset | setzt zusätzlich `g_smrTicState='\0'` | Reset der neuen TIC-State-Machine beim Moduswechsel |
| `Surplus_FeedIn_Enable_Check` | zwei Zweige mit unterschiedlicher fester Belegung des gepackten `reserved`-Felds | einer der beiden Zweige entfällt, der andere packt das Feld anders (`CONCAT11(1, g_surplusFeedInReserved)`) | Eindeutig anderes Verhalten, genaue Bitfeld-Semantik offen |
| `Board_SystemStartup` | `Config_LoadOrInit_DeviceSettings()` vor `Board_PeripheralSelfTest()` | Reihenfolge vertauscht | Boot-Reihenfolge geändert |
| `Config_LoadOrInit_DeviceSettings` | `g_deviceTypeCode=0x93`; `g_workMode`-Bereich bis 4; kein Fehlerbit-Reset | `g_deviceTypeCode=0x95`; Bereich bis 5; `g_errCodeBitmask &= ~0x100` bei normalem Boot | Mehrere reale Detailänderungen beim Config-Laden |

### HTTP-Puffer vergrößert (~1024/1100 → ~1200 Byte)

Durchgängige Vergrößerung mehrerer HTTP-/AT-Puffer, konsistent über: `HTTP_ParserOnBody`
(Body-Copy 0x400→0x4b0), `HTTPS_POST_Request` (genullter Bereich +0xB0), `HTTP_GetUrlData`
(snprintf-Bound 0x44c→0x4b0), `Quectel_HTTP_GET_StateMachine` (AT-Kommandopuffer 0x44c→0x4b0),
`Cloud_UploadTelemetryData` (Upload-Puffer 0x44c→0x4b0). Passend dazu wächst der
Telemetrie-Verlaufs-Record in `Telemetry_History_Buffer_Init`/`_Record_Push` und
`HTTP_Task_StateMachine_Handler` von 0x30b auf 0x35b Byte (+80 B, Inhalt der neuen 80 Byte nicht
im Detail bestimmt — vermutlich die neuen Kategorie-4-Felder).

### Sonstiges

| Funktion | v147 | v149 | Bedeutung |
|---|---|---|---|
| `QSPI_Flash_QuadRead`, `QSPI_WaitBusyDone` | `Delay_BusyWait_Us(…)` | `Delay_BusyWait_Ms(…)` (gleicher Zahlenwert) | Deutlich längere Wartezeiten (µs→ms) beim QSPI-Zugriff |
| `Time_RtosAwareDelayMs` | ruft `Delay_BusyWait_Us(0xfa)` 4× | ruft `Delay_BusyWait_Ms(0xfa)` 4× | Gleiche Umstellung wie bei QSPI — Bedeutung/Auswirkung auf Timing offen |
| `Quectel_AT_SendCommand_Locked` | `g_quectelAtRxMode='\x05'` | `='\x06'` | RX-Modus-Konstante geändert, Bedeutung offen |
| `sscanf_Format_Parser` | bricht bei jedem Fehlerfall ohne Rückgabewert-Tracking ab | führt Konvertierungszähler; EOF vor erstem Match liefert −1, sonst Anzahl Konvertierungen | Verhalten näher am Standard-`sscanf`; könnte auch von einem Compiler-/Bibliotheks-Update statt einer bewussten Firmware-Änderung stammen |
| `ShellCmd_api_set` | setzt nur `g_localApiEnable`/`g_apiConnState` | zusätzlich `g_localApiUdpEnable` | Neues UDP-Flag (siehe MQTT-Tabelle) |
| `ShellCmd_set_wifi` | kein Flag nach EEPROM-Write | setzt `g_wifiCredApplyPendingFlag` | Stößt vermutlich asynchrones Anwenden der WLAN-Zugangsdaten an |
| `CAN_CommLink_Timeout_DataClear` | `memset(&g_invState, 0x34, …)` | `memset(&g_invState, 0x38, …)` | Inverter-State-Block +4 Byte, unabhängig vom BMS-Pack-Wachstum, Bedeutung offen |
| `Energy_TOU_Counters_Update` | Feldoffset `energy_ctx+0x2cb` | `energy_ctx+0x2e3` | Struct um 0x18 (24 B) gewachsen — nicht dieselbe Verschiebung wie beim BMS-Pack-Record, Bedeutung offen |

## Kosmetische Änderungen

49 der 124 geänderten Funktionen unterscheiden sich nachweislich nicht im Verhalten:

- **Reine Registerzuteilung/Decompiler-Variablennamen** (kein Text-/Logikunterschied im C-Code oder
  nur andere Variablennamen für dieselbe Semantik): `BLE_Build_Cmd0x42_Response`,
  `Board_ClockSource_TrySwitchToHSE`, `CH395_ExtractUdpData`, `CH395_Socket_Dest_Config`,
  `Config_Mode_SyncIfChanged`, `Device_Network_Info_Init`, `Factory_Reset`, `Heap_Calloc`,
  `MQTT_Publish_BMS_Version`, `MQTT_Publish_PowerSchedule`, `MQTT_Session_Init`, `MQTT_SetWorkMode`,
  `MQTT_TLS_Context_Reset`, `Quectel_AT_QVERSION_Query`, `Quectel_GetDeviceIP`,
  `Quectel_HTTP_AT_SendAndVerify`, `Quectel_HTTP_GET_Request`, `Quectel_MQTT_Subscribe`,
  `Quectel_OTA_StartUpdate_AndReboot`, `Quectel_OTA_StartUpdate_FromURL_AndReboot`,
  `Quectel_UDP_Socket_Open`, `SPI_WriteByte`, `ShellCmd_set_old`, `Stub_ImmediateReturn`,
  `TLS_Cert_Decrypt_All`, `__scatterload`, `fp64_abs`, `mbedTLS_SSL_Connection_Init`, `pow`,
  `Meter_ParseUDPMeterInfo` (nur Log-Text)
- **Nur Umbenennung derselben globalen Variable** (keine Logikänderung):
  `HTTP_EcoTracker_ParsePower`, `HTTP_Economy_TOU_ParseResponse`, `HTTP_P1Meter_ParseActivePower`,
  `HTTP_StormOneMeter_ParseData`, `Cloud_HTTP_Response_Parser` (u. a. `g_meterProtocolClass`→
  `g_meterConnectReadyState`, `g_meterSourceChangeFlag`→`g_meterDataFreshFlag`),
  `Meter_DisconnectWatchdog_Check`, `MQTT_Publish_Meter_Data`
- **Feldoffsets als Folge der BMS-Pack-Record-Vergrößerung** (0x28→0x2c je Pack), keine eigene
  Logik: `CAN_AA_StoreUnitFieldPair0`, `CAN_AA_StoreUnitFieldPair8`, `CAN_AA_StoreUnitFieldPair16`,
  `CAN_AA_StoreUnitFieldPair24_Notify`, `CAN_AA_StoreUnitFieldPair64`, `CAN_AA_StoreUnitFieldPair72`,
  `CAN_AA_StoreUnitIndexedFieldPair`, `CAN_Detect_Mismatched_Nodes`, `BMS_CellVoltage_MinMax_PerPack`,
  `BMS_Version_Consistency_Check`, `MQTT_Publish_BMS_Data`
- **Nur informative Anzeigewerte / Kommentare / Build-Infos ohne Verzweigungsänderung**:
  `ShellCmd_get_ver` (Build-Datum, `ems_ver` 0x93→0x95)
- **Reine Literal-Pool-/Adressverschiebung** (Text im Dekompilat identisch, nur Assembly-Adressen
  unterschiedlich): keine der obigen doppelt gezählt — betrifft vor allem die o. g. Registerzuteilungs-Fälle.

## Neue Funktionen (nur v149, 25)

**EEPROM-I2C-Bus-Recovery** (4 Funktionen, ein zusammenhängender Mechanismus): `EEPROM_I2C_BusRecovery`
(Timeout-Callback: loggt, ruft die folgenden zwei plus `EEPROM_I2C_Peripheral_Init` auf, aufgerufen aus
`EEPROM_I2C_ReadBytes`/`WriteBytes`), `EEPROM_I2C_ClockDisable_Reset` (I2C1-Periph-Reset, APB1-Takt aus,
AFIO-Remap rückgängig), `EEPROM_I2C_GPIO_ClockPulseRecovery` (SCL als GPIO, 9× 5 µs High/Low —
klassisches manuelles I2C-Bus-Recovery-Verfahren), `Delay_BusyWait_Ms` (allgemeiner ms-Busy-Wait für
QSPI-Code und `Time_RtosAwareDelayMs`, wird von der I2C-Kette selbst nicht genutzt — die nutzt
`Delay_BusyWait_Us`).

**MQTT-/Cloud-Konfigurationsparser** (7 Funktionen, ein neues Feature): `MQTT_Config_Command_Handler`
ist der neue Top-Level-Parser für ein per Cloud/HTTP empfangenes `key=value;…`-Konfigurationskommando
(Work-Mode, BLE-Advertise-Enable, Feed-in-Flag `fu=`, lokales API-Enable+Port `ap=`,
Firmware-/Versionsnummer `sv=`, max. Lade-/Entladeleistung `mc=`/`md=` mit 2500-W-Fallback bei
Parse-Fehler, WLAN-SSID/-Passwort `ws=`/`wp=`, DOD-Wert `dv=` über `Config_Write_DischargeCutoffSOC`,
HTTP-Server-Typ `ht=`, Backup-Enable). Nutzt die typisierten Extraktoren `MQTT_Config_ExtractU8/U16/
U32Value` (strstr+atoi), `MQTT_Config_ExtractStringValue` (Kopie bis `;`), `MQTT_Config_ExtractFields`
(generische Schleife über eine 6-Einträge-Tabelle, u. a. für Energiezähler-Felder wie
`g_totalChargeEnergyWh`) und `MQTT_Config_ParseScheduleEntries` (bis zu 10 TOU-Einträge `nm0=…` mit
Unterfeldern `bt=`/`et=`/`wk=`/`vv=`/`as=`, Zeiten via `Parse_TimeString_To_HourMinute`
("HH:MM"-Parser mit Bereichsprüfung), Ziel ist die aktive TOU-Tabelle `g_touSlot0WeekdayMask`, nur
wenn `g_mqttCfgWorkMode==1`). Aufrufer: `HTTP_ClientTask_Dispatcher` und
`HTTP_Task_StateMachine_Handler` — das Feature kommt über HTTP-Cloud-Anfragen herein, nicht direkt
über den MQTT-Client.

**CAN-Bus** (3 Funktionen + 1 thematisch verwandt): `CAN_AA_StoreUnitFieldPair80` (Protokoll-0xAA-
Handler, schreibt 8-Byte-Wertepaar an Offset 0x50 des 88-Byte-Pack-Records in
`g_bmsPack0ChargePower`/`DischargePower`; kein Text-Aufrufer gefunden, vermutlich über eine
Dispatch-Tabelle), `CAN_Broadcast_Category0x17_PackedValues` (Energiezähler-Kategorien 3/4 als
CAN-Broadcast Kategorie 0x17), `CAN_Broadcast_ClearCategories0x40to0x43` (vier Null-Payload-Frames
für Kategorien 0x40–0x43), `Config_Param0A_Relay` (schreibt Relais-Steuerwert in CAN-Parameter 0x0A,
aufgerufen aus `Config_Set_DischargeCutoff_WithRelay`).

**Quectel-Modem/UART** (3 Funktionen + `StrCat`): `Quectel_AT_SendRaw_WithMutex` (mutex-geschütztes
AT-Senden mit Warten auf Antwort), `Quectel_AT_QISEND_HexData` (baut `AT+QISEND=<socket>,<len>,
"<hex>"` aus einem Byte-Puffer, aufgerufen aus `TCP_FC06_WriteSingle_Handler` und `FC03_Read_Handler`
— Modbus-Antworten können jetzt über das Mobilfunkmodem als Hex-Payload gesendet werden),
`Quectel_UART_RxByte_Parser` (Byte-Assembler für eingehende Modem-UART-Daten, CRLF-Erkennung,
Sonderzustand für `"+QIURC: \"recv\",11"`, aufgerufen aus `Quectel_UART_RxByte_ISR_Handler`),
`StrCat` (triviale strcat-Implementierung, hängt das abschließende Anführungszeichen an den
Hex-String).

**Shell-Kommandos** (2): `ShellCmd_get_backup_datas` (setzt `g_backupDataRequestFlag`),
`ShellCmd_power_check` (gibt CT-/Venus-/BMS-/Last-/PV-/Backup-Leistung aus, u. a. über
`Inverter_PVString_Channel_ValueDelta_Calc`). Für beide kein Text-Aufrufer gefunden (Registrierung
über die Shell-Kommandotabelle).

**Sonstiges/Telemetrie** (2): `Inverter_PVString_Channel_ValueDelta_Calc` (Basiswert + Delta für den
aktiven PV-String-Kanal 1–4 aus CT-/Netz-/Insel-Leistung, aufgerufen aus
`BLE_Build_DeviceStatus_Response`, `MQTT_StatusReport_BuildAndPublish`, `ShellCmd_power_check`),
`Telemetry_Store_EnergyCounters` (schreibt 8-Byte-Energiezähler-Record in einen von vier Slots,
Selektor `'@'/'A'/'B'/'C'`, aufgerufen aus `Telemetry_Register_Dispatcher`),
`MQTT_Publish_DeviceIDs` (publiziert `"cd=%d, VID=%s, XID=%s GID=%s"`, aufgerufen aus
`MQTT_CtrlCommand_Dispatcher` case 0x3e).

## Entfernte Funktionen (nur v147, 2)

- **`Config_Apply_WorkModePowerPreset`**: bildete `g_workMode` 1/2/3/4 auf `g_maxDischargePowerW`
  800/600/2200/1200 W ab (Fallback 800 W). Aufrufer in v147: `Config_ClampDefaults_ParamBlock`,
  `BLE_Build_DeviceStatus_Response`, `MQTT_Telemetry_Struct_Builder`,
  `MQTT_StatusReport_BuildAndPublish`. Alle vier Aufrufer existieren in v149 weiter, der Aufruf ist
  aber überall ersatzlos entfernt — `Config_ClampDefaults_ParamBlock` behält nur noch einen
  Plausibilitäts-Fallback (0 oder > 2500 W → 800 W/Modus 1), die übrigen drei lesen
  `g_maxDischargePowerW`/`g_workMode` jetzt direkt, ohne Neuberechnung. Die feste Preset-Tabelle
  existiert in v149 nirgends mehr; die Werte werden offenbar über den neuen
  `MQTT_Config_Command_Handler` (Felder `md=`/`mc=`) gepflegt.
- **`MQTT_Config_VID_Clear`**: löschte den 0xE4-Byte MQTT-VID/Credential-Block im externen SPI-Flash
  bei 0x315000 (derselbe Block, den `MQTT_Config_VID_Init` beschreibt). Aufrufer in v147: nur
  `Quectel_MQTT_Session_StateMachine`, an drei Stellen in Fehler-/Reset-Zweigen (periodisch alle
  600 Ticks im XID-Zweig). In v149 existiert die Funktion nicht mehr, und an den entsprechenden
  State-Übergängen fehlt jeder Ersatzaufruf — das periodische Löschen des VID/XID-Blocks bei
  Fehlerfällen wurde ersatzlos gestrichen (unabhängig verifiziert).

## Unverändert (explizit geprüft)

OTA/BLE-OTA/YMODEM (alle Funktionsnamen vollzählig und adressunabhängig in beiden Versionen vorhanden),
Cloud-Hosts (`hamedata.com`), Debug-Shell-Freischaltung über Modbus 46000 = 0x5100 (in beiden Versionen
vorhanden!), TCP-Shell 8091, feste WLAN-Zugangsdaten SM471/Hame147258, MQTT-TLS authmode 2,
Base64URL_Encode (vorhanden, entfällt erst in v150). Shell-Kommandos: **65 in v147, 67 in v149**
(+`get_backup_datas`, +`power_check`; die alte Angabe „63" war ungenau).

## Offene Punkte

Änderungen, deren Fundstelle eindeutig ist, deren fachliche Bedeutung sich aus dem Code allein aber
nicht sicher bestimmen ließ — hier absichtlich **nicht geraten**:

- `Energy_TOU_Counters_Update`: Feldoffset-Verschiebung um 0x18 (24 B) im `energy_ctx`-Struct — welche
  neuen Felder dafür verantwortlich sind, ist unklar.
- `HexChar_To_TimeOffsetIndex`: komplett andere Eingabequelle und Algorithmik (3 Hex-Zeichen/mod 10 →
  1 Hex-Zeichen/mod 5) — fachliche Auswirkung auf die TOU-Zeitfenster-Berechnung offen.
- `CAN_CommLink_Timeout_DataClear`: genullter Inverter-State-Block wächst um 4 Byte, unabhängig vom
  BMS-Pack-Wachstum — Inhalt der neuen 4 Byte offen.
- `ShellCmd_get_all_bms_data`: `mos_flag`-Stride wechselt unabhängig von der BMS-Pack-Vergrößerung von
  0x50 auf 0x58 — betroffenes Struct nicht identifiziert.
- `Surplus_FeedIn_Enable_Check`: gepacktes `reserved`-Feld wird anders befüllt, aber die genaue
  Bitfeld-Semantik ist unklar.
- `Time_RtosAwareDelayMs`: Callee wechselt von `Delay_BusyWait_Us` auf `Delay_BusyWait_Ms` (gleicher
  Zahlenwert) — ob dies eine bewusste, deutlich längere Verzögerung ist oder ein Versehen, ist offen.
- `Quectel_AT_SendCommand_Locked`: RX-Modus-Konstante `g_quectelAtRxMode` ändert sich von 5 auf 6 —
  Bedeutung der Modi nicht dokumentiert.
- `sscanf_Format_Parser`: Rückgabewert-/EOF-Verhalten ändert sich grundlegend — möglich, dass dies aus
  einem Compiler-/Bibliotheks-Update stammt statt einer bewussten Firmware-Änderung.
- `MQTT_StatusReport_BuildAndPublish`: zwei neue Report-Felder `lf=`/`pl=`, deren Argumente aktuell
  immer 0 sind (vermutlich Platzhalter für ein zukünftiges Feature).
- `MQTT_JSON_RPC_Dispatcher` case 7 „Manual": neues Flag `g_aiSchedule9Enable` wird zurückgesetzt —
  Bedeutung von „Schedule 9" unklar.
- `Protocol_AA_CommandDispatch`: Dispatch-Tabelle wächst von 15 auf 16 Einträge — welcher neue
  0xAA-Befehl das ist, wurde nicht ermittelt.
- `CAN_AA_StoreUnitFieldPair80`, `CAN_Broadcast_Category0x17_PackedValues`,
  `CAN_Broadcast_ClearCategories0x40to0x43`, `ShellCmd_get_backup_datas`, `ShellCmd_power_check`:
  kein per Textsuche auffindbarer Aufrufer im Quellcode-Satz (vermutlich über eine
  Dispatch-/Kommandotabelle außerhalb der durchsuchten Dateien referenziert).
- Modbus-Register 36000: Ghidra benennt das zugrundeliegende Global in v147 `g_warnCode`, in v149
  `g_vnsInvChargeLimitPower` (gleicher relativer Offset). `g_warnCode` wird an anderer Stelle
  (`Protocol_AA_SetChannelData`) als Array-Basisadresse indiziert — spricht für ein Namensartefakt der
  automatischen Benennung statt einer echten Funktionsänderung des Registers, aber nicht im Code
  bestätigt. Vier weitere Register (30000, 30028, 30029, 30035) zeigen dieselbe Art abweichender
  Fallback-Benennung (`g_paramStore_*`) — ebenfalls kein funktionaler Befund.
- `BLE_App_Command_Dispatcher`: der neue case 'Z' (0x5A) fällt ohne `break` in case '[' (0x5B) durch,
  d. h. auch der alte case 'Y' (0x59) löst jetzt zusätzlich die Backup-/Netzwerkinfo-Logik aus — ob
  das beabsichtigt ist, ist unklar.
- `MQTT_JSON_RPC_Dispatcher`: die Antwortpuffer der einzelnen Methoden sind in v149 dediziert statt
  wie in v147 überlappend mit Offsets — funktional vermutlich gleichwertig, aber relevant falls andere
  Funktionen noch die alten Offset-Adressen referenzieren (nicht geprüft).

## Datenbasis

- Vollständige namensbasierte Paarung: `scratchpad/v151/l147_149.json` (lokal, nicht im Repo) (1782 identisch, 124 geändert,
  2 nur v147, 25 nur v149); Modbus-Deskriptor-Vergleich `mbdesc_147.json`/`mbdesc_149.json`
- Dekompilat + Listing: `Dekompilate/VNSD_Control_147/`, `Dekompilate/VNSD_Control_149/` (lokal, nicht im Repo)
  (`listing/globals_and_registers.txt` je Version für benannte Globals)
