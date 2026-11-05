# Control-FW v150.8 (1508, Venus A) — Vollanalyse (Port von 1509)

> 🇬🇧 English version: [VNSA_Control_FW_v1508_Full_Analysis.md](VNSA_Control_FW_v1508_Full_Analysis.md)

**Stand:** abgeschlossen 2026-11-05. Das Image wurde analysiert, indem die fertige 1509-Analyse portiert und der Port am 1508-Code geprüft wurde; Grenzen und ungeprüfte Punkte stehen in §9.
**Binary:** `1508_control_VNSA-0_app_1508_0819_182932.bin` (Archiv `Marstek FW Archiv/firmwares/VNSA-0/Control/1508/`, Beta „v150.8“, „A Control 1508“)
**Port-Quelle:** Venus-A-Control v150.9, siehe [VNSA_Control_FW_v1509_Vollanalyse_de.md](VNSA_Control_FW_v1509_Vollanalyse_de.md). Alles, was 1508 mit 1509 gemeinsam hat (D/A-Vergleich, Modbus, CAN, EEPROM, Tasks, Sicherheitshinweise), steht dort und wird hier **nicht wiederholt**.
**Unterschiede 1508 → 1509 (Funktionstabelle):** [VNSA_Control_Diff_v1508_v1509_de.md](VNSA_Control_Diff_v1508_v1509_de.md)
**Ghidra:** `/VNSA/Control/Control_1508_VNSA-0_app_1508_0819_182932.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** privater Arbeits-Export (nicht Teil dieses Repositories): 1935 Funktionen, 0 Dekompilierfehler, dazu Flash-Listing und `globals_and_registers.txt`, gleiches Layout wie bei 1509.

---

## 0. Zusammenfassung

- **Gleiche Codebasis wie 1509.** Nach Maskierung von Pointern und Aufrufoperanden sind 95,19 % der 1508-Bytes identisch mit 1509. 1508 hat 1935 Funktionen, 1509 hat 1941; 1508 hat **keine eigene Funktion** (nur in 1508: keine), 6 Funktionen gibt es nur in 1509.
- **1508 ist der ältere Build** (Build-String `Aug 19 2026 18:29:09`, 1509: `Sep  8 2026 09:34:19`). Es fehlen die Änderungen, die die [Tabelle v150 → v150.9](VNSA_Control_Diff_v150_v1509_de.md) allein 1509 zuschreibt: die OTA-Gerätetyp-Klassifizierung (`OTA_ClassifyDeviceTypeTag`, YMODEM-Gerätetypprüfung), der MQTT-Kommandoblock mit `CAN_Write_Category0x18`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `ShellCmd_read_inv1/2` und das Modbus-Schreibregister 47001.
- **1508 identifiziert sich anders:** `VEPRO-0` / `VenusE Pro` / BLE-Präfix `MARS_VEPRO_` / Versionskonstante 0x5E4, wo 1509 `VNSA-0` / `Venus A` / `MST_VNSA_` / 0x5E5 hat (§4). Das ist ein Fakt aus dem Code; die Bedeutung ist nicht geklärt.
- **Das RAM-Layout ist nicht identisch mit 1509** (`.bss` ist kleiner, viele Globals liegen an anderen Adressen). Das war der Hauptaufwand beim Port (§3.3).
- **Verifikation:** alle Gates bestehen (`ALL GATES PASS`); Tags: 1825 Funktionen `NAME_VERIFIED`, 110 `NAME_LOW_CONFIDENCE` (§7).

## 1. Binary-Fingerabdruck

| | A 1508 | A 1509 (Referenz) |
|---|---|---|
| Größe | 387 072 B (0x5E800) | 391 168 B (0x5F800) |
| SHA-256 | `664623dd130c9ded1225ed7e9223db65b73a4c51add6e53fb66c1014ab3b1e4f` | `a8c58127189e19937cf179195e714492f890a6a829b03b8e218380042f8d9f23` |
| Linkadresse | **0x08004800**–0x08062FFF | 0x08004800–0x08063FFF |
| MCU | Nations **N32G45x** (Cortex-M4F) | gleich |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW) | gleich |
| Build-String (`ShellCmd_get_ver`) | `Aug 19 2026 18:29:09` @ 0x0803B147 (Stringanfang) | `Sep  8 2026 09:34:19` |
| Version | Versionswort 0x5E4 (1508) @ 0x08010000, danach die Bytes `VNSA-0` | Versionswort 0x5E5, danach `VNSA-0` |
| Image-Footer | `VNSA` + 12 Nullbytes @ 0x08062FE0, davor 0xFF-Füllung (912 B) ab 0x08062C50 | `VNSA` @ 0x08063FE0 |
| Funktionen in Ghidra | 1935 | 1941 |

Basis-Beleg: Der Initial-Stackpointer (0x2001F7E8) und der Reset-Vektor (0x08004A71) am Imageanfang lösen bei Basis 0x08004800 wie bei 1509 auf; Scatter-Tabelle sowie String- und Funktionspointer lösen bei dieser Basis auf.

## 2. Speicherkarte (Ghidra)

Gleiche Blockstruktur wie 1509 (siehe [§2 der 1509-Analyse](VNSA_Control_FW_v1509_Vollanalyse_de.md)): Bootloader unter 0x08004800 (nicht im Image), Applikations-Flash, `sysinfo` (UID @ 0x1FFFF7F0), `.data`, `.bss`, externer SRAM `.xmc_bss` @ 0x60000000, Peripherieblöcke (Namen aus dem Nations-N32G457-SVD, von 1509 kopiert: 977 Labels).

Scatter-Tabelle (`Region$$Table$$Base` @ 0x0806189C, drei Einträge, Dekompressor `__decompress` @ 0x08005FBE):

| Region | Ladeadresse (Flash) | Ausführungsadresse | Größe | Komprimiertes Abbild |
|---|---|---|---|---|
| `.data` | 0x08061DBC | 0x20000000 | 0x2F00 | 2488 B |
| externer SRAM (`.xmc_bss`) | 0x08062774 | 0x60000000 | 0x26AE0 (1509: 0x26AF8) | 1244 B |
| `.bss` (Zero-Init) | — | 0x20002F00 | 0x1C8E8 (1509: 0x1F110) | — |

Hinter der Tabelle: `ShellCommand[79]` @ 0x080618CC (1509: 81 Einträge). Das initialisierte `.data` hat dieselbe Größe wie bei 1509 (0x2F00); `.bss` ist 0x2828 Bytes und der externe SRAM 24 Bytes kleiner.

## 3. Methode

### 3.1 Port von 1509

1. **Anker:** Bytebereiche des 1509-Images, die in 1508 mit maskierten Pointer- und BL-Operanden wieder auftauchen: 452 Blöcke, 368 444 von 387 072 Bytes = **95,19 %** identisch.
2. **Funktionspaare** über diese Anker: von den 1941 Quellfunktionen sind **1902 identisch, 20 teilweise identisch, 19 ohne direktes Gegenstück**. Layout (Funktionen, Datenobjekte, Labels, Kommentare), Signaturen (1922 Paare, inkl. lokaler Variablen), Referenzen und 25 Flow-Overrides wurden von 1509 übernommen.
3. **Korrigierte Zuordnungen im umsortierten Bereich:** Im Bereich um 0x0800FF08 und an einigen Stellen bis 0x080176F4–0x08017700 stehen die Funktionen in anderer Reihenfolge als in 1509; die Anker-Zuordnung war bei 16 Paaren falsch oder fehlte. Sie wurden über normalisierte Instruktionsähnlichkeit geprüft und von Hand korrigiert (`Board_GetUniqueID_HexString`, `Quectel_Modem_HardwareReset_Handler`, `Quectel_AT_QVERSION_Query`, `BLE_GATT_HexBuffer_Send`, `Stub_Return_One`, `Util_NoOp_Stub`, `GPIO_SetBits`, `GPIO_ResetBits`, `QSPI_Flash_QuadRead_ByteLoop`, `Util_NoOp_StartupHook`, `CH395_Init_Modbus_TCP_Socket`, `MQTT_Transport_NullCallback` (Stub, niedrige Konfidenz) und vier ältere Varianten, die nach ihrer Rolle benannt sind: `Cloud_UploadTelemetryData`, `Modem_SendStatusTelemetry`, `Ethernet_SendStatusTelemetry`, `Protocol_AA_SetSystemParams`). Danach blieben kein `FUN_*`, kein Duplikat und kein temporärer Name übrig, und alle Paare haben eine Instruktionsähnlichkeit ≥ 0,9 außer den tatsächlich geänderten Funktionen (§5).
4. **Schalttabellen:** 11 Sprungtabellen im geänderten Code wurden als Arrays typisiert und mit Decompiler-Overrides versehen; die 63 Tabellenziele wurden disassembliert (Computed-Jump-Referenzen), 13 Code-Lücken innerhalb von Funktionen wurden disassembliert. In `HTTP_ClientTask_Dispatcher` wurde die Tabelle (`tbh` @ 0x0801A38A, 7 Einträge) mit `case_0`…`case_6` beschriftet; ihre Ziele sind die von 1509 um 0x14 verschoben. Die Labels folgen den Tabelleneinträgen und wurden über diese 7 Einträge hinaus nicht verifiziert.
5. **Aufrufziel-Prüfung:** Die geordneten Aufrufziele (`bl`, `blx`, `b.w` auf Funktionseinsprünge) **aller 1935 Paare** wurden verglichen. 21 Befunde, alle in ohnehin geänderten Funktionen (13 Aufrufanzahl-Unterschiede, 6 umgeordnete Callees in `HTTP_ClientTask_Dispatcher`, 2 Ersetzungen `sprintf`/`snprintf`/`memcpy`); keiner unter den 1902 identischen Paaren, also keine Fehlzuordnung und keine Namenskorrektur nötig.

### 3.2 Geänderte Funktionen

36 Paare mit Ähnlichkeit < 0,98 sowie `Cloud_PassiveMode_Connection_StateMachine`, `MQTT_JSON_RPC_Dispatcher` und `Board_GetUniqueID_HexString` wurden an normalisierter Decompiler-Ausgabe verglichen (Typ-, Cast- und `&`-Rauschen entfernt). Das Ergebnis ist die Tabelle in [VNSA_Control_Diff_v1508_v1509_de.md](VNSA_Control_Diff_v1508_v1509_de.md); 29 Plate-Kommentare „Differs from 1509: …“ markieren die Funktionen im Programm. Wo ein Unterschied gesehen, seine Bedeutung aber nicht geklärt wurde, steht in der Diff-Tabelle „unklar“.

### 3.3 RAM-Layout-Verschiebung gegenüber 1509

Das RAM-Layout von 1508 weicht von 1509 ab (Beispiel: `g_pendingBootDescriptor` @ 0x200152D4 in 1508 gegenüber 0x20017AF4 in 1509; die 1508-`.bss` ist 0x2828 Bytes kleiner, der Heap `ucHeap` ist kürzer, `g_protocolTxBuf` @ 0x20014944 liegt innerhalb der 81 920 B, die der 1509-Heap belegt). Folgen und Behandlung:

- Das Kopieren von Referenzen nach Adresse aus 1509 erzeugte **Schein-Referenzen** auf 1509-RAM-Adressen. Sie wurden bereinigt, indem jede Referenz mit dem pc-relativen Literal der 1508-Bytes verglichen wurde: 627 Instruktionsreferenzen korrigiert (352 Ziele), 29 Parameterreferenzen korrigiert, 294 Referenzen ohne Stütze in einem RAM-Literal gelöscht, 3 Datenwort-Referenzen gelöscht, 11 abgeleitete Referenzen gelöscht.
- 133 RAM-Labels, die der erste Namensabgleich auf 1509-Adressen gesetzt hatte, wurden gelöscht; 3 wurden verschoben. Namen wurden danach nur aus sauberen Stimmen vergeben (Referenz aus byte-identischem Code): 1287 Labels aus dem ersten Abgleich (falsche wie oben beschrieben entfernt), 450 weitere „exakte“ Namen, 22 per Mehrheitsstimme ≥ 50 % (mittlere Konfidenz), 2 aus dem Code-Kontext.
- **Nie nach Offset kopieren, nur nach Name:** 1610 RAM/XMC-Labels von 1508 haben ein gleichnamiges 1509-Label; davon wurden 1541 Datenobjekte wie in 1509 typisiert (Typ, Länge gegen 1509 geprüft), der Rest nicht (§9).
- 111 RAM-Adressen haben keinen Gegenstück-Namen; sie tragen den Konventionsnamen `<vorangehendes Objekt>_off<hex>` (z. B. `g_ch395RxBuf_off1029`). Ein solcher Name sagt **nur die Adressbeziehung, keine Bedeutung**; die Symbole tragen den Plate-Kommentar „NAME_LOW_CONFIDENCE: address relation only“.
- Drei Namen wurden wegen gleicher Coderollen bei abweichenden Struktur-Offsets umbenannt: `g_atRxAssemblerState`, `g_atRxSuppressFirstTerm`, `g_atUdpRxDoneFlag` (1508: +2/+3/+4 ab `g_atUdpRxLen`; 1509: +4/+5/+6, weil 1509 das zusätzliche Feld `g_atRxByteCounter` hat).
- Fünf Ein-Byte-Variablen erhielten Namen aus benachbarten 1509-Symbolen und aus Paarfunktionen, die dasselbe 1509-Symbol referenzieren (mittlere Konfidenz): `g_commCh0StaleFlag` (0x2000013D), `g_canLinkMissCount` (0x2000013E), `g_canLinkStaleFlag` (0x2000013F), `g_commCh2StaleFlag` (0x20000141), `g_canRegCbDevState` (0x20000162).

## 4. Geräte-Identität (Fakten aus dem Code)

| Punkt | 1508 | 1509 |
|---|---|---|
| Gerätetyp-String (`ShellCmd_get_ver` `ems_type`, MQTT-Topics, UDP-Discovery-Antwort) | `VEPRO-0` | `VNSA-0` |
| Shell-Wurzeleintrag (`CLI_InitSession`) | `VEPRO-0 v1508` @ 0x08051734 | `VNSA-0 v1509` |
| Cloud-Gerätename (`Cloud_Report_FillDeviceInfo`) | `VenusE Pro` @ 0x08020FC4 | `Venus A` |
| Cloud-Source-ID (`Cloud_BuildDeviceSourceID`) | `"%s-%s"` mit `VenusE Pro` | `"%s-%s"` mit `VNSA-0` |
| BLE-Namenspräfix (`BLE_Set_Device_Name`, `BLE_Query_Device_Name`) | `MARS_VEPRO_` @ 0x0800F370 | `MST_VNSA_` |
| Versionskonstante (`g_deviceTypeCode`, Cloud-Abfrage) | 0x5E4 | 0x5E5 |
| OTA-Modellprüfung (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`, Update-Magic) | vergleicht gegen `VNSA-0` | `VNSA-0` |

Archivordner, Dateiname, der Versionsblock bei 0x08010000 und der Image-Footer sagen `VNSA` / `VNSA-0`; die Strings, mit denen sich der laufende Code identifiziert, sagen `VEPRO-0` / `VenusE Pro`. Die OTA-Modellprüfung in 1508 testet weiterhin auf `VNSA-0`.

**Annahme, kein Befund:** Das Namensmuster legt nahe, dass „Venus E Pro“ gemeint ist. Dafür gibt es **keinen Hardware-Beleg** (kein Gerät, kein Hersteller-Dokument); die Bedeutung der Identitätsänderung ist unklar. Die Dokumentation verwendet deshalb den Archivnamen `VNSA` und nennt die Strings ausdrücklich.

## 5. Unterschiede zu 1509 (Zusammenfassung)

Detailtabelle mit Adressen und Beleg je Funktion: [VNSA_Control_Diff_v1508_v1509_de.md](VNSA_Control_Diff_v1508_v1509_de.md). Kurz:

- **Identität** (§4) und die Versionskonstante in der Cloud-Abfrage.
- **Funktionen und Code nur in 1509:** `OTA_ClassifyDeviceTypeTag` (Gerätetypprüfung der empfangenen Datei in `YModem_ReceiveFile`, Zweig „Bin type mismatch“ in `YModem_FirmwareUpgrade_Handler`), `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18` (1508: kein Modbus-Schreibregister 47001, kein eigener Fall für 45030 in `Write_Handler`, kein entsprechender Kommandoblock in `MQTT_CtrlCommand_Dispatcher`), `ShellCmd_read_inv1`, `ShellCmd_read_inv2` (Shell-Tabelle 79 gegen 81 Einträge).
- **Umstrukturierte Logik (Wirkung unklar):** `Battery_EPS_Reserve_SOC_Hysteresis` (Zustandsautomat statt Latch), `HTTP_ClientTask_Dispatcher` (136 gegen 364 dekompilierte Zeilen, gleiche Callees in anderer Reihenfolge, im verglichenen Teil keine Time-Sync-Fast-Retry-Behandlung), `Grid_Export_Power_Limiter` (Negation inline und 2500-W-Begrenzung von `power_hi`; die obere Begrenzung 2400 W gibt es in beiden), `CT_GridPower_Controller` (ein Aufruf `Inverter_Power_Setpoint_Calc(1,0)` weniger), `Cloud_PassiveMode_Connection_StateMachine`, `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `DeviceInfo_BuildStatusString`.
- **Ältere Formen:** `Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry`, `Protocol_AA_SetSystemParams` (nach Rolle benannt, niedrige Konfidenz); `CH395_Init_Modbus_TCP_Socket` ruft `CH395_ConfigTcpKeepalive_Defaults` nicht auf; `Config_LoadOrInit_DeviceSettings` hat ein `EEPROM_Read` weniger (0x395, `g_selLedEnable`); `Meter_ParseUDPMeterInfo` ergänzt ein `vTaskDelay`.
- **Nur Darstellung oder Verschiebung, keine Code-Änderung festgestellt:** `SMR_TIC_SelfUse_PowerController` (Log-Zeilennummern um 2 verschoben), `Quectel_SendCmd_WaitResponse`/`CH395_ExtractUdpData`/`Quectel_AT_Recv_ByteAssembler` (`memset` 0x406 gegen 0x408) und die in der Diff-Tabelle unter „reine Darstellung“ aufgeführten Funktionen.

## 6. Gemeinsam mit 1509

Für die folgenden Themen gilt die 1509-Analyse auch für 1508, **soweit der Paarvergleich identischen Code zeigt** (1902 identische Paare); sie wurde für 1508 nicht Funktion für Funktion neu hergeleitet:

| Thema | Siehe |
|---|---|
| Verhältnis zur Venus D v151 (instruktionsidentischer Anteil, Compile-Time-Gerätekonstanten) | §0, §3, §4 der 1509-Analyse |
| Leistungsgrenzen, Leistungsklassen, 13 Zellen / 6 Packs im Pack-Scan, RS485 an USART1, wegkompilierte Funktionen | §4 der 1509-Analyse |
| Cloud / HTTP / Netzwerk (hamedata `getDateInfo`, 3 Servertypen, ältere HTTP-POST-/TOU-Varianten) | §4.5 der 1509-Analyse |
| Modbus-Lese- und Schreibbereich, Limits 44002/44003 | §5 der 1509-Analyse (1508: der Limit-Text von 44003 ist identisch; 47001 und 45030 unterscheiden sich, siehe §5 oben) |
| Interner CAN-Bus (Zielklassen, Funktionscodes) | §6 der 1509-Analyse |
| Tasks, Shell, Protokolle, EEPROM-Karte, Strings, Sicherheitshinweise | §7–§10 der 1509-Analyse |

## 7. Bearbeitungsstand in Ghidra

- Abdeckung: 0 undefinierte Bytes, 0 verwaiste Instruktionen; 1935 Funktionen, alle benannt (0 `FUN_*`), alle mit Prototyp; Task-Namen passen zur Task-Tabelle.
- `marstek_gates.py` (wegen des Skript-Timeouts des Werkzeugs in Teilen ausgeführt): **`ALL GATES PASS`**.
- Tags: **1825 `NAME_VERIFIED`, 110 `NAME_LOW_CONFIDENCE`** (1935 Funktionen; exklusiv, keine mit beiden Tags — in 1509 tragen 99 Funktionen beide Tags, die Zahlen sind also nicht direkt vergleichbar). Auf `LOW` erzwungen: die vier nach Rolle benannten Funktionen, `MQTT_Transport_NullCallback` und 8 Teilpaare mit Ähnlichkeit < 0,9 (`Battery_EPS_Reserve_SOC_Hysteresis`, `Quectel_AT_Recv_ByteAssembler`, `HTTP_ClientTask_Dispatcher`, `Grid_Export_Power_Limiter`, `MQTT_StatusReport_BuildAndPublish`, `DeviceInfo_BuildStatusString`, `Cloud_BuildDeviceSourceID`, `YModem_ReceiveFile`).
- Kommentare: Übernommene Kopfkommentare widersprachen dem Code in 16 Plates (aus 1509 übernommener Identitätstext). Sie wurden gegen die echten 1508-Strings korrigiert und lauten nun „In 1509: …“: `BLE_Set_Device_Name`, `BLE_Query_Device_Name`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `Cloud_Report_FillDeviceInfo`, `Config_LoadOrInit_DeviceSettings`, `Cloud_BuildDeviceSourceID`, `ShellCmd_get_ver`, `CLI_InitSession` sowie die RAM-Plates `g_maxChargePowerW`, `g_deviceModelTag`, `g_deviceTypeCode`, `g_httpCloudServerType`, `g_deviceTypeStr`, `g_cloudDeviceSourceId`. Unverändert gelassen, weil in 1508 korrekt: die OTA-Modellprüfungen (`VNSA-0`).
- Export: 1935 Funktionen, 0 Dekompilierfehler, keine Datei mit „decompilation failed“; 2721 benannte RAM-/Peripherie-Symbole im Listing.

## 8. Standardprüfungen bei neuer Firmware-Version (1508 gegen 1509)

| Frage | Ergebnis |
|---|---|
| Modbus-Register | `Write_Handler`: in 1508 kein 47001 (0xB799) und kein eigener Fall 45030 (0xAFE6); Limit-Text von 44003 identisch. Die Deskriptortabelle zeigt nur Darstellungsunterschiede; **nicht geprüft:** registerweiser Vergleich des Tabelleninhalts. |
| BLE | Namenspräfix `MARS_VEPRO_` statt `MST_VNSA_`; ein `EEPROM_Write` weniger in `BLE_App_Command_Dispatcher`; der Befehlssatz zeigt keinen Unterschied. **Nicht geprüft:** Leistungsklassen und Limits. |
| AC-Einspeiselimit | 44003-Text unverändert (1500 W); `Grid_Export_Power_Limiter` weicht ab (§5), obere Begrenzung 2400 W in beiden; tatsächliches Einspeiselimit der 1508-Änderungen: **unklar**. |
| BLE-OTA | Die BLE-OTA-Funktionen sind vorhanden, kein Unterschied gefunden; BLE-OTA ist laut Code also möglich (**nicht am Gerät getestet**). YMODEM: in 1508 keine Gerätetypprüfung. |
| Cloud-Domains | Gleiche Menge wie in 1509; geändert: Gerätename/Source-ID und Versionskonstante in der Cloud-Abfrage. |
| Pack-Kapazität | Control-Seite unverändert: 6 Packs (`< 6`), 13 Zellen (`< 0xd`) in `BMS_CellVoltage_MinMax_PerPack`. Die BMS-seitige Grenze (`CAN_TX_PerPack_10Msgs`) gehört nicht zum Control-Image: **nicht geprüft**. |
| Nicht funktionale Register (Neuprüfung) | Auf Registerebene **nicht geprüft**. |

## 9. Offene Punkte und Grenzen

1. **Bedeutung der Identität** `VEPRO-0` / `VenusE Pro` / `MARS_VEPRO_`: unklar; „Venus E Pro“ ist eine Annahme ohne Hardware-Beleg.
2. **111 RAM-Namen `<Objekt>_off<hex>`** sind Adressbeziehungen mit niedriger Konfidenz, keine Semantik (§3.3). Für sie wurden keine Feldnamen von Hand abgeleitet.
3. **10 Typkonflikte** (Liste in den Arbeitsdaten, Datei `p3b_conflicts_100.json`; `p3b_type_conflicts.json` selbst ist leer): Das 1509-Objekt würde ein Label verschlucken, das nur in 1508 existiert, das Layout weicht dort also ab. Diese Objekte wurden **nicht** typisiert: `s_quit_debug_5100`, `g_modbusRegDescTable` (Label `_off9` darin), `g_bleOtaDeviceTypeTable`, `g_cloudDomainUrlTable`, `ecp_supported_grp_id`, `ucHeap` (der 1508-Heap ist kürzer), `g_wifiSsid`, `g_wifiPassword`, `g_tlsCertDecodeBuf`, `g_telHistDatePacked`. Weitere RAM-Labels ohne gleichnamiges, typisiertes 1509-Gegenstück sind untypisiert.
4. **5 Ein-Byte-Variablen** (§3.3) haben Namen mittlerer Sicherheit (Nachbarlayout plus Beleg über Paarfunktionen).
5. **Streu-Referenzen:** Referenzen auf benannten Objekten können noch einzelne Schein-Referenzen enthalten, die innerhalb von 0x1000 eines Literals liegen; sie sind ohne Datenfluss nicht erkennbar. Der Filter für unbelegte Referenzen fängt nur weit entfernte.
6. **Nicht geprüft:** die Deskriptortabelle Eintrag für Eintrag gegen 1509 und gegen D; die BMS-seitige Pack-Grenze; der Status nicht funktionaler Register; BLE-Leistungsklassen und -Limits; ein Vergleich 1508 gegen das Release v150 (die Tabelle v150 → v150.9 wurde für 1508 nicht neu erstellt); alles, was von der Micro- und BMS-Firmware desselben Releases abhängt (diese Analyse betrifft nur das Control-Image).
7. **Unklare Wirkungen:** EPS-Reserve-Hysterese, Begrenzung in `Grid_Export_Power_Limiter`, Umstrukturierung von `HTTP_ClientTask_Dispatcher` (Vollständigkeit seiner 1508-Dekompilierung nicht abschließend geklärt, seine Sprungtabellen-Labels wurden rekonstruiert), `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `DeviceInfo_BuildStatusString`, `Cloud_PassiveMode_Connection_StateMachine`.
8. **Nach Rolle benannte Paare** (`Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry`, `Protocol_AA_SetSystemParams`) und der Stub `MQTT_Transport_NullCallback` haben niedrige Konfidenz. Ein Sprungtabellen-Label in `Write_Handler` (`case_15`) wurde aus Nachbar-Labels platziert, nicht aus der Tabelle.
9. **Gemeinsam mit 1509:** Die offenen Fragen zum A-Micro (CAN-Frames, Leistungsklassen 4/5) aus §12 der 1509-Analyse gelten für 1508 unverändert.

## Anhang: Methode

Offline-Ankerkarte der beiden Images (maskierte Pointer und BL-Offsets) → Funktionspaare → Übernahme von Layout, Signaturen, Referenzen, Flow-Overrides und Kommentaren aus 1509 → Verifikation der Paare über normalisierte Instruktionsähnlichkeit und Korrektur von Hand im umsortierten Bereich → Bereinigung von Schein-RAM-Referenzen und Namensabgleich aus sauberen Stimmen → Datentypen und Peripherie-Labels nach Name → Schalttabellen → Aufrufziel-Vergleich aller Paare → Tags → Identitätskommentare → Gates → Export. Hilfsskripte und der Arbeits-Export sind nur lokal (nicht im Repository).
