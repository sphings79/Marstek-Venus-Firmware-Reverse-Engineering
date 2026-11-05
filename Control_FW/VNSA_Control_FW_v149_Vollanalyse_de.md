# Control-FW v149 (Venus A) — Vollanalyse (Port aus 150)

> 🇬🇧 English version: [VNSA_Control_FW_v149_Full_Analysis.md](VNSA_Control_FW_v149_Full_Analysis.md)

**Status:** abgeschlossen 2026-11-05. Das Image wurde analysiert, indem die fertige v150-Analyse portiert und der Port am 149-Code verifiziert wurde; Grenzen und ungeprüfte Punkte stehen in §9.
**Binary:** `149_control_VNSA-0_app_0149_0528_102448.bin` (Archiv `Marstek FW Archiv/firmwares/VNSA-0/Control/149/`, „A Control 149“)
**Port-Quelle:** Venus A Control v150, siehe [VNSA_Control_FW_v150_Vollanalyse_de.md](VNSA_Control_FW_v150_Vollanalyse_de.md). Alles, was 149 mit 150 teilt (Vergleich mit Venus D, Modbus, CAN, EEPROM, Tasks, Sicherheitshinweise), ist dort dokumentiert und wird hier **nicht wiederholt**.
**Unterschiede 149 → 150 (Funktionstabelle):** [VNSA_Control_Diff_v149_v150_de.md](VNSA_Control_Diff_v149_v150_de.md)
**Ghidra:** `/VNSA/Control/Control_149_VNSA-0_app_0149_0528_102448.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** privater Arbeitsexport (nicht Teil dieses Repositorys): 1913 Funktionen, 0 Dekompilierfehler, dazu ein Flash-Listing und `globals_and_registers.txt` (2589 benannte RAM-/Peripherie-Symbole), gleiche Struktur wie bei 150.

---

## 0. Zusammenfassung

- **Gleiche Codebasis wie 150, aber die ältere Build.** Nach Maskierung von Adressen und Aufrufoperanden sind 92,24 % der 149-Bytes identisch mit 150. 149 hat 1913 Funktionen, 150 hat 1935: **23 Funktionen gibt es nur in 150**, eine (`Base64Url_Encode`) nur in 149.
- **149 ist Venus A:** `VNSA-0` / `Venus A` / BLE-Präfix `MST_VNSA_`, Versionswort 0x95 (149), Build-String `May 28 2026 10:23:00` (§4). Es gibt keine `VEPRO`-/`VenusE`-Strings.
- **Hauptunterschiede zu 150** (§5): BMS-Packsatz 0x58 statt 0x60 Bytes (150 ergänzt SoH und MinPackVoltage), Telemetrie-Historienslot 0x35b statt 0x35d, kein Power-Clamp-Modus / Netzleistungs-Offset, kein HTTPS-Upload (ältere HTTP- + AES-128-ECB- + Base64Url-Übertragung an `hamedata.com`), kein No-Link-Reboot-Watchdog, 7 MQTT-SSL-Konfigurationsschritte ohne SNI, SMR_TIC-Zustandsmaschine inline in `CT_PowerSetpoint_Compute`.
- **Das RAM-Layout ist nicht identisch mit 150** (`.bss`, externes SRAM und `.data` sind kleiner, viele Globals liegen an anderen Adressen). Das war die Hauptarbeit beim Port (§3.3).
- **Verifikation:** alle Gates bestanden (`ALL GATES PASS`); Tags: 1786 Funktionen `NAME_VERIFIED`, 127 `NAME_LOW_CONFIDENCE` (§7).

## 1. Binary-Fingerabdruck

| | A 149 | A 150 (Referenz) |
|---|---|---|
| Größe | 382 976 B (0x5D800) | 387 072 B (0x5E800) |
| SHA-256 | `f14fede53a07462ecbba7c37c0c1b24ac910a5962206a54f2ab21c62034ea07e` | `7dbff622f394a8262f9f545fbdabff2e815aa73122ce37f5b9412ca631702560` |
| Linkadresse | **0x08004800**–0x08061FFF | 0x08004800–0x08062FFF |
| MCU | Nations **N32G45x** (Cortex-M4F) | gleich |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, komprimiertes RW) | gleich |
| Build-String (`ShellCmd_get_ver`) | `May 28 2026 10:23:00` @ 0x0803A550 | `Aug 11 2026 11:46:45` |
| Version | Versionswort 0x95 (149) @ 0x08010000, danach die Bytes `VNSA-0` | Versionswort 0x96 (150), danach `VNSA-0` |
| Image-Footer | `VNSA` + 12 Nullbytes @ 0x08061FE0, davor 0xFF-Füllung (1108 B) ab 0x08061B8C | `VNSA` @ 0x08062FE0 |
| Funktionen in Ghidra | 1913 | 1935 |

Basis-Beleg: Der Initial-Stackpointer (0x2001F778) und der Reset-Vektor (0x08004A71) am Imageanfang lösen bei Basis 0x08004800 auf (`Reset_Handler` @ 0x08004A70, Thumb), wie bei 150; Scatter-Tabelle sowie String- und Funktionspointer lösen bei dieser Basis auf.

## 2. Speicherkarte (Ghidra)

Gleiche Blockstruktur wie 150 (siehe [§2 der 150-Analyse](VNSA_Control_FW_v150_Vollanalyse_de.md)): Bootloader unterhalb 0x08004800 (nicht im Image), Applikations-Flash, `sysinfo` (UID @ 0x1FFFF7F0), `.data`, `.bss`, externes SRAM `.xmc_bss` @ 0x60000000, Peripherieblöcke (Namen aus dem Nations-N32G457-SVD, aus 150 kopiert: 977 Labels).

Scatter-Tabelle (`Region$$Table$$Base` @ 0x08060840, drei Einträge, `scatter_region_t[3]`; Dekompressor `__decompress` @ 0x08005FBE):

| Region | Ladeadresse (Flash) | Ausführungsadresse | Größe | Komprimiertes Abbild | Größe in 150 |
|---|---|---|---|---|---|
| `.data` | 0x08060D50 | 0x20000000 | 0x2EEC | 2480 B (`Load$$RW_IRAM1$$Base`) | 0x2F00 |
| externes SRAM (`.xmc_bss`) | 0x08061700 | 0x60000000 | 0x242E4 | 1164 B (`Load$$RW_XMC$$Base`) | 0x26AE0 |
| `.bss` (zero-init) | — | 0x20002EEC | 0x1C88C | — | 0x1C8E8 bei 0x20002F00 |

Hinter der Tabelle: `ShellCommand[78]` @ 0x08060870 (`shellCommandTable`). Die FreeRTOS-Tasktabelle `TaskDef_t[14]` @ 0x080592E0 (Label `g_taskDefTable`) listet 14 Tasks; alle Namen passen zu ihren Funktionen. Der llhttp-Textblob @ 0x0805D35A ist als char-Arrays typisiert, an den referenzierten Adressen geteilt (58 Stücke).

## 3. Methode

### 3.1 Port aus 150

1. **Anker:** Byteblöcke des 150-Images, die in 149 mit maskierten Pointer- und BL-Operanden wieder auftauchen: 491 Blöcke, 353 276 von 382 976 Bytes = **92,24 %** identisch (1508 ← 1509 waren 95,19 %). Richtung: 149 ist das ältere Image, „unmapped“ Quellfunktionen sind also Funktionen, die in 150 neu sind.
2. **Funktionspaare** über diese Anker: Von den 1935 Quellfunktionen sind **1844 identisch, 46 teilweise identisch, 45 ohne direktes Gegenstück** (1890 Paare). Layout (1851 Funktionen angelegt, 1628 entfernt in zwei Fenstern), Segmente, Signaturen (inkl. lokaler Variablen) und Flash-/Peripherie-Referenzen (+1564) wurden aus 150 übernommen; RAM-Referenzen wurden bewusst **nicht** kopiert (§3.3); 23 Flow-Overrides wurden angewendet (2 zurückgesetzt, darunter ein falscher Computed Jump bei 0x08005EFC; 6 nicht anwendbar). Nach der Auto-Analyse hatte der Import allein 1638 Funktionen; der Port endet bei 1913.
3. **Korrigierte Zuordnungen im umsortierten Bereich:** Die Funktionen in 0x0800F9xx–0x08010xxx sind um bis zu 0x7F4 Bytes umsortiert, adressbasierte Paarung ist dort falsch. 14 Paare wurden per normalisierter Instruktionsähnlichkeit korrigiert (Beispiele: 0x080102A0 ist `Quectel_HTTP_ReadResponse_Verify`, nicht `Telemetry_History_Buffer_Init`; 0x0802A7E0 ist `Cloud_ExtractConfigValue`, nicht `HTTPS_ParseData`; 0x0800AE4C ist `Config_Read_SavedDateTime_0x160`, nicht `Config_Read_String_0x387`; 0x08032E64 ist `CAN_AA_StoreUnitFieldPair80`, nicht `...Pair88`). Zehn `FUN_*`-Funktionen wurden identifiziert (`Stub_Return_One`, `Quectel_AT_SendCommand_Locked`, `Util_NoOp_Stub`, `Quectel_HTTP_AT_SendAndVerify`, `CAN_Detect_Mismatched_Nodes`) oder erhielten Rollennamen mit niedriger Konfidenz (`Cloud_UploadTelemetryData`, `CT_PowerSetpoint_Compute`, `mbedtls_ssl_conf_endpoint`, `_transport`, `_own_cert`).
4. **Verwaister Code, Lücken, Switch-Tabellen:** 9 verwaiste Instruktionsläufe (248 Instruktionen) und 72 undefinierte Läufe (20 289 B) wurden geschlossen: 11 weitere Funktionen (`Quectel_HTTP_ReadResponse_Trigger`, `Telemetry_History_Buffer_Init`, `BMS_Version_Consistency_Check`, die acht `mbedtls_ssl_conf_*`-Setter), Null-Padding als `byte[n]`, komprimierte Blobs als `byte[]`. Typische Defekte: Code als `uint` an Case-Einsprüngen, 5-Byte-„Strings“, die Code waren, Strings, die ein Byte zu früh begannen und das oberste Byte des vorangehenden Literal-Pointers verschluckten, ein „Waisenlauf“, der ein String war (`[HTTP] Upload data_len ...`), ein Waisenlauf, der die Jump-Table-Bytes des BLE-Dispatchers war. **64 TBB/TBH-Sprungtabellen** (62 zuerst gefunden, 2 weitere in zuvor undisassembliertem Code) sind alle als Arrays typisiert; für 24 davon waren Decompiler-Overrides nötig (die zuvor abgeschnittenen Funktionen und jede Tabelle, die der Decompiler nicht selbst rekonstruierte). Die drei großen Dispatcher waren vorher abgeschnitten (BLE_App_Command_Dispatcher 72 von ~2962 Instruktionen, MQTT_CtrlCommand_Dispatcher 287 von ~2957, HTTP_ClientTask_Dispatcher 98 von ~899); jetzt 6860 B, 6900 B und 2000 B (`HTTP_Task_StateMachine_Handler`: 5364 B), keine Decompiler-Warnungen. Case-Labels gegen Tabellen: 0 Abweichungen über alle 64 Tabellen; 39 Tabellen haben keine `jmp_*`-Labels, weil der Decompiler sie selbst rekonstruierte.
5. **Aufrufzielprüfung:** Die geordneten Aufrufziele aller Paare wurden verglichen (siehe [Diff-Tabelle](VNSA_Control_Diff_v149_v150_de.md), Methodik). 21 Befunde; drei davon deckten **Verschmelzungs-/Benennungsfehler des Ports selbst** auf (`Base64Url_Encode` + `Timer_Callback_BLE_Modem_Supervisor` bei 0x08006A1C; `HTTP_ParseUploadResponseCode` + `CH395_SPI_SendData_Verified` bei 0x0801973C mit zwei als Code disassemblierten Strings; `Quectel_SSL_ConfigSNI` war in Wahrheit `Quectel_SSL_ConfigVerifyLevel`). Nach der Korrektur bleiben 19 Befunde, alle in tatsächlich geänderten Funktionen.

### 3.2 Geänderte Funktionen

43 Paare wurden auf normalisierter Dekompilat-Ausgabe verglichen (Typ-, Cast- und `&`-Rauschen entfernt). Das Ergebnis ist die Tabelle in [VNSA_Control_Diff_v149_v150_de.md](VNSA_Control_Diff_v149_v150_de.md); 41 Plate-Kommentare „Differs from 150: …“ markieren die Funktionen im Programm. Wo ein Unterschied gesehen, seine Bedeutung aber nicht geklärt wurde, steht in der Diff-Tabelle „unklar“. Der Vergleich lief, bevor die RAM-Datentypen existierten; deshalb waren die ersten rohen Diffs von untypisierten Globals und fehlenden Peripherie-Labels dominiert (`DAT_40010c00` statt `GPIOB_PL_CFG`).

### 3.3 RAM-Layout-Verschiebung gegenüber 150

Das RAM-Layout von 149 weicht von 150 ab (`.bss` 0x1C88C vs. 0x1C8E8, externes SRAM 0x242E4 vs. 0x26AE0; Basisliterale unterscheiden sich; die Verschiebung ist je Cluster verschieden, z. B. -0x10 im HTTP-Task-Cluster und -8 um die SMR/tic-Variablen). Folgen und Umgang:

- **Namen per Votes, nicht per Adresse.** Der erste Namensabgleich legte **1236 RAM-Labels** aus Votes echter Referenzen an (Referenz aus byte-identischem Code); ein Audit gegen die Votes bestätigte 1235 davon. Stichproben verglichen das `ldr`-Literal in derselben Funktion mit dem Ziel (z. B. `g_bleConnState` @ 0x20018AEE = Basisliteral 0x200188E0 + 0x20E). 506 von 1240 Adressen haben eine Referenz, die direkt von einem Literalwort gestützt wird, die übrigen 734 sind Struct-Felder über Basisliteral + Offset.
- **Konflikte:** 70 Konfliktadressen und 4 übersprungene (Namen mit Votes für zwei Adressen) gingen in das Gate-4-Paket: 282 referenzierte Globals waren unbenannt. Ergebnis: 96 Namen aus Vote + übereinstimmendem 150-Layout, 83 Namen allein aus der 150-Layout-Verschiebung (Plate `NAME_LOW_CONFIDENCE: 150 layout shift only`), 10 von Hand, 96 Namen `<object>_off<hex>` (Plate `NAME_LOW_CONFIDENCE: address relation only`). Drei falsche Labels des ersten Abgleichs (je 1 Vote) wurden gelöscht: `g_invState` (echte Adresse 0x20014E74), `g_quectelAtCmdBuf` (0x2000180D), `g_modemHwResetFlag` (0x20000FFA).
- **SMR_TIC inline:** `g_ctPowerSetpoint` = 0x20000290 (50 Votes); 0x200002F8 `g_smrTicNewPowerW`, 0x200002FC `g_smrTicRefCtPowerW`, 0x20000300 `g_smrTicSettleCount`, 0x20000304 `g_smrTicMaxCtPowerW`, 0x20000308 `g_smrTicStateTick`, 0x2000030A `g_smrTicStatePeriodMs` (gegen die 150-Zustandsmaschine dekompiliert; Plate `NAME_FROM_CONTEXT`). In 149 ist `ucHeap` 8-aligned (Blockkopf bei +0, Größe bei +4), daher wurden die 150-Unterlabels (`ucHeap_firstFreeBlock_xBlockSize` usw.) nach den Votes aus `prvHeapInit` gesetzt.
- **Nie nach Offset kopieren, nur nach Name.** Datentypen: 1394 RAM-/XMC-Labels von 149 haben ein gleichnamiges 150-Label mit Daten; **1336** davon mit definiertem Typ (abzüglich `*_off<hex>`) wurden wie in 150 typisiert (Typ aufgelöst, Länge gegen 150 geprüft, 1179 Kommentare kopiert, wo 149 keinen hatte). Vorher: undefined1/2/4; nachher: `uchar`/`ushort`/`uint`/`int`/`bool`/Pointer/`char[n]`/`uint[256]` (FT/RT-Tabellen)/`ModbusRegDesc_t[246]` usw. 150 hat keine Pack- oder Telemetrie-Struct-Typen, daher bleiben der 0x58-Packsatz und der 0x35b-Slot Skalare plus `_off`-Labels. Zwei Objekte mit Labels im Inneren wurden mit dieser Erlaubnis typisiert (`g_modbusRegDescTable`, 246 x 12 B endet genau am nächsten Label; `ucHeap`, 71 680 B endet genau an `g_protocolTxBuf`).
- **Nebenwirkung des Verengens:** Der kleinere 150-Typ über einem breiteren 149-Objekt ließ 94 referenzierte Bytes verwaisen (Packsätze, dns1-Bytes, mppt/can-Felder); sie erhielten `<base>_off<hex>`-Namen (`g_dns1Addr_1..3` folgen dem Muster von `g_dns2Addr_0..2`). Gate 4 wurde erneut ausgeführt: 94 → 0.
- **Peripherie:** 977 Labels (SYSINFO/UID, 0x40000000…, PPB 0xE000…) aus 150 als `uint` mit Kommentaren kopiert; `Board_GetUniqueID_HexString` dekompiliert mit `UID_31_0`, `UID_63_32`, `UID_95_64`.

### 3.4 Gefundene Fallen (für spätere Ports)

- Gleiche Einsprungadresse in beiden Images ist Zufall, nie ein Beleg (0x08006A1C, 0x08010010). Funktionsgröße und Strings jedes Paares prüfen, nicht nur die Instruktionsähnlichkeit.
- Klon-Skripte eines älteren Ports (1508/1509-Namen, -Adressen) vor dem Lauf prüfen; ein veralteter Klon mit 1508-Adressen wurde gefunden und als reine Leseprüfung neu geschrieben.
- Ein kleinerer Typ über einem breiteren Objekt lässt referenzierte Bytes verwaisen: nach einem Typ-Sweep immer Gate 4 erneut ausführen.
- Aus 150 kopierte Plates können 150-Adressen enthalten (Filter auf `0x[0-9a-f]{8}`).
- Das Löschen eines breiten Bereichs um eine Lücke löscht auch gültige Instruktionen an Case-Einsprüngen; nach jedem Clear + Disassemble die Switch-Tabellen-Disassemblierung und danach den Orphan-Attach-Schritt erneut ausführen.
- Ein Skript, das das Tooling-Timeout überschreitet, kann trotzdem fertig geworden sein (Ausgabe geschrieben, Zustand intakt); vor dem erneuten Lauf den Zustand prüfen.

## 4. Geräte-Identität (Fakten aus dem Code)

| Punkt | Wert in 149 |
|---|---|
| Versionswort @ 0x08010000, danach `VNSA-0`; Footer `VNSA` @ 0x08061FE0 | 0x95 (149) |
| `ShellCmd_get_ver` | `ems_type=VNSA-0`, `ems_ver=0x95` (149), `Build: May 28 2026 10:23:00` (String @ 0x0803A550) |
| `VNSA-0` weitere | 0x08008CB0, 0x0800927C, 0x080094CC, 0x08009C8C, 0x08036CFC (UDP-Discovery-Antwort), 0x0802CB40 (`Config_LoadOrInit_DeviceSettings`, `g_deviceTypeCode = 0x95`) |
| Shell-Wurzeleintrag (`CLI_InitSession`) | `VNSA-0 v149` (0x08050924, 0x0805A8F5) |
| BLE-Namenspräfix | `MST_VNSA_` (0x0800EE58, 0x0800EEF0) |
| Cloud-Gerätename / Quell-ID | `Venus A` (0x080204C4) / `VenusA-%s` (0x080339AC) |
| `VEPRO`-/`VenusE`-Strings | keine |
| Cloud | `http://<region>.hamedata.com/...` (`getDateInfo`, `backup-configs`, `ems/getDevicesApply`, `schedulings/policies` über https, `prod/api/v1/setVenusDReporting`); weitere Servertypen mit `http://%s/api/v1/data`, `/v1/data`, `/v1/json`; kein `marstekcloud.com` (150 sendet an `https://api-<region>.marstekcloud.com`) |
| MQTT-Topics | `marstek_energy/%s/device/%s/ctrl`, `marstek/%s/server|device/%s/ctrl` (wie in 150) |

## 5. Unterschiede zu 150 (Zusammenfassung)

Detailtabelle mit Adressen und Beleg je Funktion: [VNSA_Control_Diff_v149_v150_de.md](VNSA_Control_Diff_v149_v150_de.md). Kurz:

- **Datenlayout:** BMS-Packsatz 0x58 Bytes (150: 0x60, ergänzt SoH und MinPackVoltage); Telemetrie-Historienslot 0x35b (150: 0x35d). Sichtbar in den packbezogenen Funktionen der Diff-Tabelle (`BMS_CellVoltage_MinMax_PerPack`, `ShellCmd_get_all_bms_data`, `MQTT_Publish_BMS_Data`, die `CAN_AA_StoreUnit*`-Familie, ...).
- **Funktionen und Code nur in 150 (23):** der HTTPS-/TLS-Upload-Client (`HTTPS_FetchUrlData`, `HTTPS_PackPostRequest`, `HTTPS_ParseData`, `HTTPS_TLSSessionInit`, `HTTPS_TLS_Context_Reset`, `HTTPS_ParserOnBody`, `CH395_TCP_Socket_Open_HTTPS`, `mbedTLS_SSL_Recv/Send_WithRetry`, `mbedtls_ssl_close_notify`, `Quectel_HTTP_POST_SendAndVerify`, `Quectel_SSL_ConfigSNI`), der Power-Clamp-Modus und die String-Zugriffsfunktionen (`Config_Read/Write_PowerClampMode_0x394`, `Config_Read/Write_String_0x387`, `MQTT_Publish_Grid_Status`), `SMR_TIC_SelfUse_PowerController` (in 149 inline), `CAN_AA_StoreUnitFieldPair88`, `CH395CMDGetSocketStatus`, `ShellCmd_FreeRtos_mem`, `FreeRTOS_xPortGetFreeHeapSize`, `FreeRTOS_xPortGetMinimumEverFreeHeapSize`.
- **Ältere Formen in 149:** Cloud-Upload (`Cloud_UploadTelemetryData`: AES-128-ECB + Base64Url an `hamedata.com`), `HTTP_Task_StateMachine_Handler` (GET + `HTTP_ParseUploadResponseCode`), `HTTP_FindHeaderBodySeparator` (`Connection:` + 0x15 statt CRLFCRLF + 4), `Modem_/Ethernet_SendStatusTelemetry`, `Quectel_MQTT_Config_StateMachine` (7 Schritte, kein SNI), `mbedTLS_SSL_Connection_Init` (keine Logs zu Zertifikat-Ladefehlern).
- **In 149 fehlendes Verhalten:** der No-Link-Reboot-Watchdog in `HTTP_ClientTask_Dispatcher`, der Zweig `g_powerClampConfig == 1` in `Inverter_PowerSetpoint_DeadbandClamp` / `Remote_Power_Setpoint_Process`, der Schlüssel `ct_dev=` in `Cloud_HTTP_Response_Parser`, 20 Aufrufe in `MQTT_CtrlCommand_Dispatcher`, 8 Aufrufe und der Befehl „Get new power data“ in `BLE_App_Command_Dispatcher`.
- **Wirkung unklar:** die `g_meterSourceType`-Kette in `Meter_ParseUDPMeterInfo` und `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `BLE_Build_Cmd0x42_Response`.

## 6. Was mit 150 gemeinsam ist

Für die folgenden Themen gilt die 150-Analyse auch für 149, **soweit der Paarvergleich identischen Code zeigt** (1844 identische Paare, nach Maskierung byte-identisch); für 149 wurde sie nicht Funktion für Funktion neu hergeleitet:

| Thema | Siehe |
|---|---|
| Verhältnis zu Venus D v150 (instruktionsidentischer Anteil, Compile-Time-Gerätekonstanten) | [150-Analyse](VNSA_Control_FW_v150_Vollanalyse_de.md), §3 |
| Leistungslimits, Leistungsklassen, 13 Zellen / 6 Packs je Pack-Scan, RS485 an USART1, herausgecompilierte Funktionen, Cloud / HTTP / Netzwerk (ältere Varianten weichen ab, siehe §5) | [150-Analyse](VNSA_Control_FW_v150_Vollanalyse_de.md), §4 |
| Modbus-Lese- und Schreibbereich, 44002/44003-Limits (das Paar `Write_Handler` ist identisch) | [150-Analyse](VNSA_Control_FW_v150_Vollanalyse_de.md), §5 |
| Interner CAN-Bus (Zielklassen, Funktionscodes) | [150-Analyse](VNSA_Control_FW_v150_Vollanalyse_de.md), §6 |
| Tasks, Shell, Protokolle, EEPROM-Map, Strings, Sicherheitshinweise | [150-Analyse](VNSA_Control_FW_v150_Vollanalyse_de.md), §7–§10 |

Wo 149 abweicht (Cloud, BMS-Satz, Power-Clamp, EEPROM-Konfigblockgrößen), gelten §5 oben und die Diff-Tabelle.

## 7. Bearbeitungsstand in Ghidra

- Abdeckung: 0 undefinierte Bytes, 0 verwaiste Instruktionen (Abdeckungswerkzeug: Läufe von 1–2 Instruktionen sind tote Zweige nach noreturn-Aufrufen, an die Funktion angehängt); 1913 Funktionen, alle benannt (0 `FUN_*`), alle mit Prototyp; Tasknamen passen zur Tasktabelle.
- Gate-Skript (wegen des Skript-Timeouts des Tooling in Teilen ausgeführt): **`ALL GATES PASS`** (1913 Funktionen): keine `FUN_*`, keine doppelten Namen, keine temporären Namen oder Labels, keine Platzhalter, keine Default-Prototypen, keine noreturn-Abweichungen, keine Shell-Abweichung, kein Flow in Daten, kein Funktionspointer auf Daten, kein Literal-only-Code, 0 doppelte globale Labels, 0 unbenannte referenzierte Globals, Tasktabelle passt.
- Jump-Table-Labels: 33 importierte `switch`-/`case_N`-Labels waren Quell-Überbleibsel; 32 duplizierten ein Ghidra-`switchD_*`-Label und wurden gelöscht; `case_15` @ 0x080559FA (`Write_Handler`) bleibt global genau wie in 150 (die Tabelle hat nur 15 Einträge, die Adresse ist ein `nop` dahinter).
- Tags: **1786 `NAME_VERIFIED`, 127 `NAME_LOW_CONFIDENCE`** (1913 Funktionen; exklusiv, keine mit beiden Tags, keine ohne). Regel: ein Quell-`LOW` gewinnt; auf `LOW` erzwungen: 12 Funktionen aus der Low-Confidence-Liste (die `mbedtls_ssl_conf_*`-Setter, `CT_PowerSetpoint_Compute`, `Cloud_UploadTelemetryData`, ...), 28 Paare mit Instruktionsähnlichkeit < 0,9 und `Base64Url_Encode` (keine Quelle).
- Kommentare: 41 Plates „Differs from 150: …“; 10 übernommene Plates anhand der echten 149-Strings korrigiert (Build/Version, 0x95, `v149`, EMS-Version, ein Watchdog-Satz, MQTT-Konfigurationszustände, POST-Timeouts, Heap-Größe). 285 RAM-Namen tragen Plates (`NAME_LOW_CONFIDENCE` oder `NAME_FROM_CONTEXT`).
- Smoke-Test: 83 Funktionen (alle mit „Differs from 150“-Plates, `Write_Handler`, die drei Dispatcher, eine Stichprobe in festem Abstand): 0 Dekompilierfehler, 0 rohe `0x2000xxxx`-/`0x6000xxxx`-Adressen in den Dekompilaten.
- Export: 1913 Funktionen, 0 Dekompilierfehler, keine Datei mit „decompilation failed“ oder `halt_baddata`; 2589 benannte RAM-/Peripherie-Symbole im Listing.

## 8. Standardprüfungen bei neuer Firmware-Version (149 gegen 150)

| Frage | Ergebnis |
|---|---|
| Modbus-Register | `Write_Handler`, `FC03_Read_Handler` und `RS485_Modbus_RegisterMap_Init` sind identische Paare; weder 47001 (0xB799) noch ein eigener Fall 45030 (0xAFE6) in einem von beiden; Limit-Text zu 44003 identisch (Ablehnung über 1500 W). `g_modbusRegDescTable` hat gleichen Typ und gleiche Größe (246 x 12 B) wie in 150. **Nicht geprüft:** der Tabelleninhalt Eintrag für Eintrag. |
| BLE | Namenspräfix `MST_VNSA_` wie in 150. `BLE_App_Command_Dispatcher` hat 8 Aufrufe weniger (Power-Offset- und Power-Clamp-Schreibzugriffe, `String_0x387`), kürzere Statusantworten (0xa4 / 0x50 statt 0xa7 / 0x51), keinen Befehl „Get new power data“. **Nicht geprüft:** Leistungsklassen und Limits. |
| AC-Einspeiselimit | `Grid_Export_Power_Limiter` ist ein identisches Paar: 2500 W Startklemme, 2400 W obere Grenze; 44003-Text unverändert (1500 W). Ohne Power-Clamp-Modus / Netzleistungs-Offset (nur 150) weicht der Sollwertpfad ab; effektives Einspeiselimit: **unklar**. |
| Update-Pfad (BLE-OTA) | Die BLE-OTA-Funktionen sind vorhanden und identische Paare; BLE-OTA ist daher soweit der Code zeigt möglich (**nicht am Gerät getestet**). Modellprüfung gegen `VNSA-0`. `HTTP_FindHeaderBodySeparator` weicht ab (`Connection:` + 0x15 Bytes), siehe [OTA_HTTP_Body_Boundary_und_Diagnose_de.md](OTA_HTTP_Body_Boundary_und_Diagnose_de.md); Folgen für den HTTP-Download von 149: **nicht geprüft**. |
| Cloud-Endpunkte | `hamedata.com`-Satz (`getDateInfo`, `backup-configs`, `getDevicesApply`, `schedulings/policies`, `setVenusDReporting`) plus weitere Servertypen (`http://%s/api/v1/data`, `/v1/data`, `/v1/json`); kein `marstekcloud.com`; kein HTTPS-Upload-Pfad. Cloud-TLS-Konfiguration: **nicht geprüft**. |
| Pack-Kapazität | Control-Seite: 6 Packs (`< 6`) und 13 Zellen (`< 0xd`) in `BMS_CellVoltage_MinMax_PerPack`, wie in 150; Packsatz 0x58 Bytes (150: 0x60). Das BMS-seitige Limit (`CAN_TX_PerPack_10Msgs`) ist nicht Teil des Control-Images: **nicht geprüft**. Hinweis: Packs zählen im BMS/CAN ab 1. |
| Nicht funktionale Register (Nachprüfung) | Auf Registerebene **nicht geprüft**. |

## 9. Offene Punkte und Grenzen

1. **Etwa 190 RAM-Namen `<object>_off<hex>`** (96 aus dem Gate-4-Paket, 94 aus dem Typ-Sweep) sind Adressbezüge mit niedriger Konfidenz, keine Semantik; **83 weitere Namen** stammen allein aus der 150-Layout-Verschiebung (Plate `NAME_LOW_CONFIDENCE: 150 layout shift only`). Für sie wurden keine Feldnamen von Hand abgeleitet.
2. **72 RAM-Plates nicht übernommen:** Plates von 150, die 150-RAM-/Flash-Adressen enthalten (z. B. `ucHeap`, `g_rtcSetDateBuf_*`, `g_otaErrCode*`), wurden weggelassen, weil sie für 149 falsch wären; diese Objekte haben keine Plate. Sie wurden nicht von Hand neu geschrieben.
3. **Acht `mbedtls_ssl_conf_*`-Setter** (authmode, ca_chain, ciphersuites, curves, max_version, min_version, rng, sig_hashes) und `mbedtls_ssl_conf_endpoint` / `_transport` / `_own_cert` sind nur nach Position und Größe benannt (kein 149-String- oder Aufrufer-Beleg). Aufrufer im SSL-Setup-Code könnten sie bestätigen.
4. **Per Rolle benannte Funktionen:** `Cloud_UploadTelemetryData` (Ähnlichkeit 0,28), `CT_PowerSetpoint_Compute` (0,57, SMR_TIC inline; ein Name wie `..._WithSmrTic` wurde erwogen und nicht verwendet), `Modem_/Ethernet_SendStatusTelemetry`, `Base64Url_Encode` (kein 150-Partner; ob 150 einen Base64-Encoder hat, ist offen, sein Alphabet-String ist unreferenziert). Insgesamt tragen 127 Funktionen `NAME_LOW_CONFIDENCE`.
5. **Nur per Ähnlichkeit gepaart:** `BMS_Version_Consistency_Check` (0,929) und `Telemetry_History_Buffer_Init` (0,842) sind plausible ältere Formen der 150-Funktionen, nicht als verhaltensgleich bewiesen.
6. **Streureferenzen:** Der Refcheck fand keine falsche Referenz (4446 konsistent, 713 ohne Basisliteral, 0 inkonsistent; 60 abgeleitete Referenzen weit weg von einem Literal blieben nach Prüfung erhalten), aber RAM-Referenzen entstanden durch Namensabgleich, nicht durch Kopie; einige abgeleitete Referenzen könnten noch ungenau sein.
7. **Nicht geprüft:** die Deskriptor-Tabelle Eintrag für Eintrag gegen 150 und gegen D; das BMS-seitige Pack-Limit; der Status nicht funktionaler Register; BLE-Leistungsklassen und Limits; die Cloud-TLS-Konfiguration; ein Vergleich von 149 mit den Betas 1508/1509 und v151; alles, was von der Micro- und BMS-Firmware desselben Releases abhängt (diese Analyse betrifft nur das Control-Image).
8. **Unklare Wirkungen:** siehe die „unklar“-Zeilen der Diff-Tabelle (`g_meterSourceType`-Bedingungen, `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `BLE_Build_Cmd0x42_Response`, Wirkung des fehlenden Power-Clamp-Modus auf das Einspeiselimit).
9. **Gemeinsam mit 150:** Offene Fragen zum A-Micro (CAN-Frames, Leistungsklassen) aus der 150-Analyse gelten für 149 unverändert.

## Anhang: Methode

Offline-Ankerkarte der beiden Images (maskierte Pointer und BL-Offsets) → Funktionspaare → Übernahme von Layout, Signaturen, Flash-/Peripherie-Referenzen, Flow-Overrides und Kommentaren aus 150 → Verifikation der Paare per normalisierter Instruktionsähnlichkeit und Korrektur von Hand im umsortierten Bereich → verwaister Code, Lücken und Sprungtabellen → RAM-Namensabgleich aus Votes echter Referenzen, dann Benennung der übrigen referenzierten Globals (Layout-Verschiebung, Vote, Kontext) → Datentypen und Peripherie-Labels nach Namen → Aufrufziel-, String- und Größenvergleich aller Paare → Tags → Kommentare → Gates → Export. Hilfsskripte und der Arbeitsexport sind nur lokal (nicht im Repository).
