# Control-FW v150 (Venus E3) — Vollanalyse und Unterschiede zur Venus D v150 und E3 v151

> 🇬🇧 English version: [VNSE3_Control_FW_v150_Full_Analysis.md](VNSE3_Control_FW_v150_Full_Analysis.md)

**Stand:** abgeschlossen 2026-10-14. Alle Aussagen wurden am Code dieses Images geprüft; offene Punkte stehen in §12.
**Binary:** `150_control_VNSE3-0_app_0150_0804_151249.bin` (Archiv `Marstek FW Archiv/firmwares/VNSE3-0/Control/150/`, Release v150)
**Referenzen:** Venus D Control v150 ([VNSD_Control_FW_v150_Vollanalyse_de.md](VNSD_Control_FW_v150_Vollanalyse_de.md)) — gleiche Codebasis; E3 Control v151
([VNSE3_Control_FW_v151_Vollanalyse_de.md](VNSE3_Control_FW_v151_Vollanalyse_de.md)), die fertige Analyse des Nachfolge-Images
**Diff:** [VNSE3_Control_Diff_v150_v151_de.md](VNSE3_Control_Diff_v150_v151_de.md) (v150 → v151, namensgepaart und vollständig)
**Ghidra:** `/VNSE3/Control/Control_150_VNSE3-0_app_0150_0804_151249.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** `Dekompilate/VNSE3_Control_v150/` (nur lokal, nicht im Repo): 1920 Funktionen, 0 Dekompilierfehler, + `listing/` (Flash-Listing, `globals_and_registers.txt`)
**Modbus-Deskriptortabelle:** [../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v150.csv](../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v150.csv)

---

## 0. Zusammenfassung

- **E3 v150 (gebaut 4. Aug 2026, veröffentlicht 12. Aug 2026) ist der Vorgänger von E3 v151 (gebaut 28. Sep 2026) und dieselbe Codebasis wie die Venus D.** 91,4 % des Images sind nach Maskierung
  von Pointern und BL-Offsets identisch zu E3 v151 (976 Segmente), 90,8 % zu D v150 (914 Segmente). Namensgepaart gegen E3 v151: 1871 von 1920 Funktionen
  sind byteidentisch (maskiert), 49 weichen ab (19 kosmetisch, 30 echte Änderungen, siehe Diff-Dokument), keine existiert nur in v150, 7 existieren nur in v151.
  Namensgepaart gegen D v150: 1915 Paare, 1845 identisch, 70 geändert, 5 nur in der E3 (E3-Dirty-Bit-Wrapper, `OTA_Is_VNSE3_Model`, `Util_NoOp_StartupHook`),
  39 nur in der D (die in der E3-v151-Analyse §4.6 aufgelisteten herauskompilierten Funktionen: Schutzketten-/Tastenüberwachung, Einspeisebegrenzer, PV/MPPT- und Multi-Pack-Meldungen, CAN-Datum/Uhrzeit-Broadcast, 0xC4-Schreibzugriffe; die OTA-Modellprüfung der D, die die E3 unter anderem Namen hat; und 6 Linker-Stubs).
- **RAM-Layout weicht von D v150 und von E3 v151 ab:** `.data` 12 024 B (0x2EF8), `.bss` endet bei 0x2001F5DF (initialer SP 0x2001F5E0), externes SRAM `.xmc_bss` 113 732 B.
  Der FreeRTOS-Heap ist **70 KB** (v151: 80 KB); das erklärt 10 240 der 10 400 Byte, um die `.bss` kleiner ist als in v151. Namen wurden aus E3 v151 über
  Code-Alignment, Deskriptortabellen-Paarung und Anker-Interpolation übernommen (§11).
- **Modbus-Registerkarte identisch zu E3 v151 außer 32102.** Gleiche 246 Deskriptoreinträge (30000–38014); Register **32102 ist ein `float32`**, das 4 Byte des
  i16-Paars `g_batSamplePowerW` + `g_batSampleVoltage_0p1V` liest (sinnlos; v151: `i16`-Alias von 30001). `Write_Handler` fehlt das Schreibregister
  47001 (0xB799) aus v151, und die OTA-Zielauswahl im 46000-Block antwortet mit Status 2, statt CAN-Kategorie 0x18 zu senden (§5).
- **Leistungsgrenzen 2500 W**, Wähler 600 W (Modus 2) / 800 W (Modus 1) / 2500 W (Modus 0) in BLE, MQTT und JSON-RPC — die 2200 W (Modus 3) der D gibt es nicht.
- **Kein PV/MPPT, kein Batterie-Stack, keine Parallel-Wechselrichter** (wie in v151): PV-Felder fehlen in BLE-/MQTT-/Cloud-Nutzdaten, kein Einspeisebegrenzer, MQTT cd 0x28–0x2D und BLE-
  Befehle 0x40–0x43 entfernt bzw. leer, CAN-Datum/Uhrzeit-Broadcast (0xCC) und 0xC4-Schreibzugriffe werden nicht gesendet.
- **Cloud:** `getDateInfo` auf `http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php` (v151: `api-<region>.marstekcloud.com`), nur Servertypen 0–2;
  Geräte-Source-ID `VenusE 3.0-<BLE-MAC-ID>` (v151: `VNSE3-0-<id>`).
- **HTTP-OTA-Download (GitHub-Issue #3, verifiziert):** Der Body-Start-Selektor von v150 ist `strstr(response, "\r\n\r\n") + 4`; die alte Logik `strstr(response, "Connection:") + 21`
  steckt in 147/148/149 und ist ab 150 verschwunden. Die Erst-Lese-Behandlung des Streaming-Readers ist unverändert, ohne NULL-Prüfung und ohne Akkumulation (§4.7).
- **Zwei Schreibweisen der Modellkennung** (wie in v151): Laufzeit `VNSE3-0` (MQTT-Topics, Cloud, Shell-Benutzer `VNSE3-0 v150`, BLE-Name `MST_VNSE3_`), OTA-/Boot-Deskriptor-Prüfung
  `VNSEE3-0`, Image-Footer `VNSEE3`.

## 0a. Checkliste (Dauerprüfungen, E3 v150)

| # | Frage | Antwort E3 v150 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register anders? | **Lesetabelle identisch zu E3 v151 außer 32102** (`float32` @ 4 Byte in v150, `i16` in v151) und den RAM-Quellpointern (178 der 246 liegen 0x2818 tiefer als in v151, 33 liegen 0x27F8 tiefer, 24 liegen 8 höher, 3 unverändert). Gegenüber D v150 ist die Lesetabelle in Registernummer, Typ, Skalierung und Anzahl für alle 246 Einträge identisch (212 Quellpointer liegen 8 Byte tiefer als auf der D). `Write_Handler` @ 0x08053738: gleicher Registersatz wie v151 außer 47001 (0xB799) und der Reaktion der OTA-Zielauswahl im 46000-Block (v150: Status 2; v151: `CAN_Write_Category0x18(value)` und Erfolg); 0xAFDF und 0xAFE6 gibt es in beiden. Register 30200 liest das Versionswort @ 0x08010000 = **150** (0x96). | §5, Diff §1 |
| 2 | BLE anders? | Gerätename `MST_VNSE3_` + 4 ID-Bytes. Befehl 0x15 Leistungswähler 600/800/2500 W (D: 800/2200/2500 W); Befehle 0x40–0x43 tun nichts; Befehl 0x03 Statusantwort 0x8B Byte (D 0xA7); Befehl 0x04 Infostring ohne `mppt_v`; Befehl 0x13 Fehlerhistorie = 20 × 14 Byte (§4.7). v150-spezifisch: Befehl 0x0F / 0x54 setzen `g_epsReserveState` zurück, LED (0x59) nur im RAM. | §4.1, §4.3 |
| 3 | AC-Grenzen? | Max. Lade-/Entladeleistung **2500 W** (`Config_Write_U16_0x202` Klemme 0x9C4, `Config_Write_U16`, Defaults 0x9C4); BLE 0x16: 300–2500 W, 0x17: ≤ 2500 W, 800-W-Fahrplanklemme in Modus 1; kein Einspeisebegrenzer. | §4.1 |
| 4 | Firmware-Update per BLE möglich? | Ja. Gleicher OTA-Code wie v151; die Modellprüfungen vergleichen mit **`VNSEE3-0`** (`OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`, Boot-Deskriptor in `ProcessFirmwareUpdateCommand`). Der YMODEM-Pfad hat **noch keine** Dateityp-Prüfung (v151 ergänzt `OTA_ClassifyDeviceTypeTag`). | §4.5 |
| 5 | Cloud-Domains? | `getDateInfo` über `hamedata.com` (Klartext-HTTP, Port 80, Regionspräfix `eu`/`www`/`us`); Policies, Backup-Configs, Daten-Upload und MQTT-Hosts wie in v151. | §4.4 |
| 6 | Pack-Kapazität? | Die Control behält die D-Strukturen: `Protocol_AA_CommandDispatch` akzeptiert Units `< 9` (einsbasiert), 8 Pack-Datensätze à 0x60 B (`g_bmsPack0Voltage` @ 0x20014FAC + k·0x60, letzter Datensatz @ 0x2001524C), 16 Zellen (die MQTT-Telemetrie gibt die 16 Zellspannungen von Datensatz 0 aus), Modbus-Pack-Blöcke 34000–34699 unverändert. Multi-Pack-Auswertung entfernt. Die BMS-seitige Grenze klärt die E3-BMS-115-Analyse. | §6a |
| 7 | Diff-Tabelle | [VNSE3_Control_Diff_v150_v151_de.md](VNSE3_Control_Diff_v150_v151_de.md) (EN + DE) | – |
| 8 | Neuprüfung nicht funktionaler Register | Control-Seite gegenüber v151 unverändert außer 32102; Gründe, die im Micro oder BMS liegen, können sich mit einem reinen Control-Image nicht ändern. | Diff §8 |

## 1. Binary-Fingerabdruck

| | E3 v150 | E3 v151 | D v150 |
|---|---|---|---|
| Größe | 374 784 B (0x5B800) | 378 880 B | 389 120 B |
| SHA-256 | `f043becb81ba2c7a5fb5e13edcdc625c6778724bc27073056967c728c989841b` | `a2f566eb…0148` | – |
| Hersteller-CRC16 (Modbus) | 10332 (0x285C) | – | – |
| Linkadresse | **0x08004800**–0x0805FFFF (18-KB-Bootloader darunter) | 0x08004800–0x08060FFF | 0x08004800– |
| MCU | Nations **N32G45x** mit XFMC (Cortex-M4F), 144 KB SRAM | gleich | gleich |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW) | gleich | gleich |
| Build-String | `Aug  4 2026 15:11:03` | `Sep 28 2026 09:39:27` | `Aug  5 2026` |
| Version | `VNSE3-0 v150` (Shell-Root); Versionswort 0x96 (150) @ 0x08010000, gefolgt von `VNSEE3-0` | 0x97 | 0x96 |
| Image-Footer | `VNSEE3` + Nullbytes @ 0x0805FFE0, 0xFF-Füllung ab 0x0805FEF0 | @ 0x08060FE0 | `VNSD` |
| Gerätetypcode | `g_deviceTypeCode` = 0x96 | 0x97 | 0x96 |

Basisnachweis: Die Vektortabelle bei 0x08004800 liefert SP 0x2001F5E0 und Reset 0x08004A71 (Keil `06 48 80 47 06 48 00 47`); die Scatter-Tabelle bei 0x0805ECA0 liegt im Image und
alle drei Regionen (LZ77-Quellen im Image, `__decompress` @ 0x08005FBE) lösen auf dieser Basis auf; `SystemInit` @ 0x08030260 setzt `SCB->VTOR = 0x08004800`.

## 2. Speicherkarte (Ghidra)

| Block | Bereich | Inhalt |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | nicht im Image |
| flash | 0x08004800–0x0805FFFF | Anwendung |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-Bit-UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EF7 | RW-Initialwerte (LZ77 @ 0x0805F1C0, 2480 B) |
| .bss | 0x20002EF8–0x2001F5DF | nullinitialisiert; `ucHeap` @ 0x2000313C (70 KB); initialer SP 0x2001F5E0 |
| sram_unused | 0x2001F5E0–0x20023FFF | bis zur 144-KB-Grenze des N32G45x |
| .xmc_bss | 0x60000000–0x6001BC43 | externes SRAM über XFMC (LZ77-Image @ 0x0805FB70, 896 B) |
| Peripherie | apb1/apb2/ahb/Bit-Band/ppb | Registernamen aus dem Nations-N32G457-SVD (1:1 aus E3 v151 kopiert, gleiche Hardwarekarte) |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC- und QSPI-Controller |

Scatter-Tabelle `Region$$Table$$Base` @ 0x0805ECA0 (Ende 0x0805ECD0): .data 0x0805F1C0 → 0x20000000 (0x2EF8), XMC 0x0805FB70 → 0x60000000 (0x1BC44), .bss Null-Init 0x1C6E8 @ 0x20002EF8.
Die Shell-Befehlstabelle folgt direkt (0x0805ECD0, 79 Einträge).

## 3. Beziehung zu anderen Versionen (Methode und Zahlen)

| Vergleich | Segmente / Abdeckung | Funktionen |
|---|---|---|
| E3 v150 gegen E3 v151 | 976 Segmente, 91,4 % des Images (96,3 % byteidentisch darin) | 1871 identisch, 49 geändert, 7 nur in v151 |
| E3 v150 gegen D v150 | 914 Segmente, 90,8 % | 1845 identisch, 70 geändert, 5 nur E3, 39 nur D |
| E3 v150 gegen A v150 | 930 Segmente, 90,9 % | – |
| E3 v150 gegen E3 v149 | 902 Segmente, 87,9 % | – |

Methode: Offline-Bytesegmente mit maskierten Pointern und BL-Offsets (Ankerkette), dann in Ghidra Layout-Synchronisation aus E3 v151 (Funktionskörper, Namen, Thunks),
Daten- und Label-Übertragung für gematchte Segmente, Übertragung von Prototyp / Plate / lokalen Variablen für die 1846 identischen und 40 geänderten Paare, RAM-Namen per Abstimmung
über ausgerichtete Befehle, Modbus-Deskriptor-Paarung, Interpolation zwischen Ankern, danach Handarbeit an den 34 Funktionen, die die Segmentkarte nicht paaren konnte (31 wurden per maskierter Byte-Ähnlichkeit in Adressreihenfolge einer v151-Funktion zugeordnet: 25 exakt, 6 geändert mit Ähnlichkeit 0,69–0,96;
3 — `AI_ScheduleQuery_Response_Send`, `Modem_SendStatusTelemetry`, `Protocol_AA_SetSystemParams` — über Position, Aufrufer und Verhalten) und
Durchsicht aller geänderten Funktionen per Dekompilat-Diff.

## 4. Funktionale Unterschiede zur Venus D v150 (E3-Reduktionen) und zu E3 v151

### 4.1 Leistungsgrenzen und Leistungswähler

| Stelle | E3 v150 | D v150 |
|---|---|---|
| `Config_Write_U16_0x202` (max. Ladeleistung, EEPROM 0x202) | Klemme 2500 W | 2500 W |
| `Config_Write_U16` (max. Entladeleistung, EEPROM 0x204) | Klemme 2500 W | 2500 W |
| BLE-Befehl 0x15 / MQTT cd 0x0F / JSON-RPC „set version“ | **600 W → Modus 2**, 800 W → Modus 1 (+ Fahrplanklemme auf 800 W), 2500 W → Modus 0 | 800 → 1, 2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–2500 W / ≤ 2500 W | gleich |
| Einspeisebegrenzer | **nicht vorhanden** | vorhanden |

### 4.2 Batterie / BMS

Ein-Pack-Sicht wie in v151: `BMS_CellVoltage_MinMax_PerPack`, `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version`, `BLE_Build_Cmd0x42_Response` und die MQTT-Stack-Befehle sind
nicht Teil des Images (§4.6); `BMS_Version_Consistency_Check` kopiert nur die Pack-0-Version; `CAN_CommLink_Timeout_DataClear` überwacht nur BMS (1) und Wechselrichter (2).
EPS-Reserve: `Battery_EPS_Reserve_SOC_Hysteresis` @ 0x0800A714 ist in v150 eine Drei-Zustands-Maschine (`g_epsReserveState`: 0 Leerlauf, 1 eingerastet, 2 durch niedrigen SOC deaktiviert): Einrasten bei
SOC ≤ Entlade-Abschaltschwelle, unter 9 % Reserve deaktivieren und EEPROM 0x300 = 0 schreiben, Ausrasten bei Abschaltschwelle + 5 %, automatisches Wiedereinschalten über 8 %; sie läuft auch bei SOC 0 %.
v151 ersetzt sie durch eine zustandslose Variante (Diff-Tabelle).

### 4.3 BLE- und MQTT-Befehlsunterschiede

- **BLE** (`BLE_App_Command_Dispatcher` @ 0x0800C0D4): direkter Switch über das Befehlsbyte; Befehle 0x40–0x43 leer; Befehl 0x03 Antwort 0x8B Byte; Befehl 0x04 Infostring
  `type=%s,id=%s,mac=%s,dev_ver=%d,bms_ver=%d,fc_ver=%s,inv_ver=%d` (ohne `mppt_v`); Befehl 0x02 (Servertyp) akzeptiert 0–4; LED (0x59) nur RAM; Befehl 0x0F und 0x54 setzen den EPS-Zustand zurück.
- **MQTT-Ctrl** (`MQTT_CtrlCommand_Dispatcher` @ 0x08022830): cd 0x28–0x2D entfernt; kein cd 0x42; kein `debug`-Unterbefehl für cd 0x13 (v151 ergänzt beides).
- **JSON-RPC** (`MQTT_JSON_RPC_Dispatcher` @ 0x0801E58C): Methode 5 (PV-Daten) antwortet „Method not found“ (Fehler −0x7F59).
- **Status/Telemetrie:** `MQTT_StatusReport_BuildAndPublish` @ 0x08027D68 ohne den PV-Block der D und ohne v151s `,df=%d`; `MQTT_Publish_PowerSchedule` veröffentlicht p1/p2 mit bis zu 30 Einträgen
  (D: p1–p7, 15 Einträge); die 1-s-Leistungshistorien behalten nur Netz- und Off-Grid-Leistung (je 29 Samples); die Peak-Shaving-Felder `peak_status` / `peak_power` sind Teil des Statusstrings.

### 4.4 Cloud / HTTP

| Funktion | E3 v150 | D v150 |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://%s.hamedata.com/app/neng/getDateInfo%s.php?uid=…&fcv=…&aid=…&sv=0x96&sbv=…&mv=101&cert=…&boot=…&inv=…` (+ OTA-Statusblock `&rst=…&mod0=vns&…&mod3=bms…`, wenn `g_otaFlag` gesetzt ist); Regionspräfix aus `g_cloudDomainUrlTable[type]`, Dateisuffix aus `g_cloudGetDateInfoSuffixTable[type]`; gültige Typen 0–2 (`eu`, `www`, `us`) | gleich plus `&mpptv=%d` |
| `Cloud_UploadTelemetryData` (`{"d":"di=…"}`) | Schlüssel `pv, sk, mv, me, ma, fu` entfernt; Datensatz-Stride 0x227; Versionskonstante 0x96 | Stride 0x35D |
| `Cloud_BuildDeviceSourceID` | `VenusE 3.0-<BLE-MAC-ID>` | `VenusD-<SN>` |
| `Cloud_Report_FillDeviceInfo` | Gerätename `VenusE 3.0` | `Venus D` |
| `Energy_TOU_Counters_Update` | 2 Zähler (`"%d,%d"`) | 3 Zähler |
| Upload über FC41D | `AT+QHTTPPOST=<len>,60,60`, äußeres Timeout 600; `Quectel_HTTP_POST_SendAndVerify` mit zwei blinden Verzögerungen | – |

Domains, Pfade und der TLS-Code sonst wie in v151 (Policies und Backup-Configs auf `hamedata.com`, Daten-Upload auf `marstekcloud.com`).

### 4.5 Identität und OTA

- Laufzeit-Strings: `VNSE3-0` (`g_deviceModelTag`/`g_deviceTypeStr` = `VNSE` + `3-0`), `VNSE3-0 v150`, `MST_VNSE3_`, `VenusE 3.0`, Geräte-Source-ID `VenusE 3.0-<BLE-MAC-ID>`.
- OTA / Boot: `OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate` und `OTA_FW_Verify_SetStatus` akzeptieren ein Image, dessen Typstring ein **Teilstring von `VNSEE3-0`** ist
  (`strstr("VNSEE3-0", received)`; Slot-Status 0x400 Kommunikations-Timeout, 0x401 CRC, 0x402 Gerätemaske, 0x403 Typ, 0x404 CAN-Update-Fehler, siehe §4.7).
  `ProcessFirmwareUpdateCommand` schreibt den Boot-Deskriptor mit dem Tag `VNSEE3-0`. Der YMODEM-Pfad (`YModem_ReceiveFile`) klassifiziert den Dateinamen in v150 nicht.
- „AI-Modus“: `AI_ScheduleQuery_Response_Send` (cd 58, nur bei `g_userWorkMode == 5`) antwortet mit allen Einträgen in einer Nachricht (v151 paginiert).
- „CT-Frequenz“: `Meter_ParseUDPMeterInfo` blockiert nach jedem gültigen Zählersample in `vTaskDelay(g_meterActiveCtType * 1200)`, und `Modem_/Ethernet_SendStatusTelemetry` laufen bei jedem Aufruf;
  v151 ändert beides auf eine 1,1-s-Ratenbegrenzung.
- Verbindungs-Watchdog: `HTTP_ClientTask_Dispatcher` @ 0x080196CC setzt den CH395 nach 1800 Ticks ohne Fortschritt zurück, wenn mehr als ein Telemetrie-Datensatz anliegt; das Gerät wird nie neu gestartet (v151 tut das nach 14 400 Ticks).

### 4.6 In der E3 fehlende Funktionen (herauskompiliert) und E3-Wrapper

Gleiche Menge wie in E3 v151 §4.6 (Schutzketten-/Tastenüberwachung, Einspeisebegrenzer und Überschuss-Einspeisung, PV/MPPT, Multi-Pack-Meldungen, CAN-Datum/Uhrzeit- und Kategorie-0xC4-Befehle) plus die nur in der D vorhandenen
Linker-Stubs (6 in D v150). Die Inverter-Dirty-Bit-Bitmap hat das E3-Layout (0x1C Bits, 7 pro Byte gepackt; Wrapper `Inverter_RegDirty_Mark_BleState`, `_GridPermit`, `_MeterInvalid`, `_NetworkLink`, `_ChgVolt`);
`Grid_BuildStatusFlagsReg` sendet einen 8-Byte-Frame aus lauter Nullen an die CAN-ID 0x1801FFAA.

### 4.7 HTTP-OTA-Downloadpfad und Diagnose (GitHub-Issue #3, an diesem Image verifiziert)

| Punkt | E3 v150 | Anmerkungen |
|---|---|---|
| Body-Start-Selektor `HTTP_FindHeaderBodySeparator` @ 0x08018238 | `strstr(resp, "\r\n\r\n") + 4`; NULL → Log „Not found split mark \r\n\r\n, header incomplete“ und Rückgabe NULL | v147/148/149: `strstr(resp, "Connection:") + 21` (an den rohen Archivdateien verifiziert, Selektoren 0x08016EF0 / 0x08017570 / 0x08017A58); erstmals in 150 behoben |
| Socket-Antwort-Reader `HTTPS_POST_ReceiveResponseData` @ 0x08019240 | im OTA-Modus überschreibt jedes `CH395GetRecvData` den Pufferanfang; der erste Lesevorgang wird für sich ausgewertet (`HTTP_ParseContentLengthHeader`, Selektor, `body_len = read_len − (sep − buf)`, `OTA_WriteDataToRingBuffer(sep, body_len, 0)`) **ohne NULL-Prüfung und ohne Akkumulation** eines unvollständigen Headers; spätere Lesevorgänge sind Body-Daten. Der Nicht-OTA-Modus (Aufrufe aus `HTTP_GetUrlData`) akkumuliert bis zur Puffergröße und nutzt den Selektor nicht | identisch in 151 (0x08019310) und (laut Rohbytes) in 148 (0x08018560) |
| OTA-Cache-Writer `OTA_WriteDataToRingBuffer` @ 0x08029038 | Ringpuffer 5 Slots × 0x800 B; liefert −1 bei NULL / Länge 0, −2 wenn der OTA-Stream nicht aktiv ist | identisch in 151 (0x08029448) |
| OTA-URL-Zustandsmaschine | `HTTP_ClientTask_Dispatcher` @ 0x080196CC, aufgerufen aus `Task_ch_http` @ 0x08030362; OTA-Zweig: `HTTPS_POST_Request(6, 0x14, g_otaUrlSlot0_url + idx·0xFB, len, ota_mode = 1)` | 151: 0x0801979C |
| Generische HTTP-Anfrage `HTTPS_POST_Request` @ 0x08018BA0 | baut ein GET (`HTTP_PackGetRequest`), löst DNS auf, öffnet den CH395-Socket, ruft den Reader auf | 6 Aufrufstellen in `HTTP_GetUrlData` verwenden `ota_mode = 0` |
| BLE-0x13-Fehlerhistorie | `BLE_Send_Response(0x13, g_displayErrorLog, 0x118)` = 20 × 14 Byte (Jahr u16, Monat/Tag/Stunde/Minute, zwei u32-Codes); EEPROM 0x1100; Dispatcher 0x0800C0D4, Loader `EventLog_DisplayError_LoadAndValidate` 0x080172A4, Writer `EventLog_Record_DisplayError` 0x0802B8CC | – |
| Komponenten-OTA-Diagnose | `OTA_Set_SlotStatus(slot, code, detail)` @ 0x0802BBD4: 4 Slots × 8 Byte (u16 Code, u32 Detail) @ `g_otaErrCode0` 0x20017E28; Schreiber: 0x400 Kommunikations-Timeout (`Comm_Watchdog_CheckTimeout`), 0x401 / 0x402 / 0x403 (`OTA_FW_Verify_SetStatus` @ 0x080093B8), 0x404 (`CAN_UpdateResultHandler` @ 0x0802F898); Konsole `ShellCmd_get_ota_log` @ 0x08038554 | dieselben Funktionen liegen in 151 an anderen Adressen |

Das Hersteller-Changelog von v150 nennt „Fixed failing HTTP updates in Ethernet mode“ und „Fixed data loss caused by overlong HTTP payloads“; die Selektor-Änderung passt zu diesen Punkten, der Code nennt seine Absicht aber nicht.

## 5. Modbus

- **Lesebereich:** `g_modbusRegDescTable` @ 0x2000034C, 246 × 12 B (30000–38014). Typen, Skalierungen und Anzahlen sind identisch zu E3 v151 und zu D v150 außer Register 32102 (Typ 0x24 `float32`, Elementgröße 4; v151 `i16`).
  Quellpointer (v150 minus v151): 178 Einträge −0x2818, 33 Einträge −0x27F8 (RAM-Verschiebung), 24 Einträge +8 (`.data`), 3 unverändert (externes SRAM), 6 sonstige. `FC03_Read_Handler` @ 0x080220E0 und `Read_Serializer` @ 0x08052638 sind befehlsidentisch zu v151, die bekannten D-Fehler (Vorzeichen 32101, MBAP-Länge von Exception-Frames) gelten also.
- **Schreibbereich:** `Write_Handler` @ 0x08053738 (45000-Befehlsblock, 46000 = 0x5100 Debug-Modus, 46500–46544 Fahrplan, hoher Block 0xAFxx); Sprungtabellen TBB 8 @ 0x080538F4, TBB 6 @ 0x0805392E, TBH 15 @ 0x080539E6 (manuelle Overrides, alle Ziele innerhalb der Funktion). Unterschiede zu v151: kein 47001 (0xB799); der 46000-Block (Werte 1234 / 2345 / 3456 / 4567 wählen `g_otaTargetDevice` 1 / 4 / 3 / 2) liefert nach der Auswahl Status 2, v151 ruft `CAN_Write_Category0x18(value)` auf und liefert 0. 0xAFDF (`Inverter_Set_Reg_0xFE`) und 0xAFE6 (`Config_Apply_SingleReg` 11 / 12) werden in beiden Versionen behandelt, nur über unterschiedlich kompilierte Dispatch-Pfade.
- **E3-spezifische Register:** keine; wie in v151 haben Register, die aus PV/MPPT oder aus Pack 2–8 gespeist werden, keine Datenquelle. Deskriptor-Dekodierung: [VNSE3_Descriptor_Table_Decoded_v150.csv](../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v150.csv).

## 6. Interner CAN-Bus (Control-Seite)

Arbitration-ID: `CAN_Build_Arbitration_ID` @ 0x08034940. Empfangsseite: `CAN_FrameDispatcher` @ 0x08030EE0 → Klasse 4 `Telemetry_Register_Dispatcher`, Klasse 2 `Modbus_ResponseDispatch`, sonst
`Protocol_AA_CommandDispatch` @ 0x080317C4. Gesendete Funktionscodes wie in v151 §6, außer den Kategorie-0x18-Schreibzugriffen aus dem Modbus-46000-/47001-Pfad und dem MQTT-`cd=0x42`-Pfad, die nur v151 hat.
Nicht gesendet: 0xCC Datum/Uhrzeit, 0xC4-Kategorieschreibzugriffe, 0xCB DLC2, 0x04 DLC2, 0x03-DLC2-Clear.

### 6a. Pack-Kapazität (Dauerprüfung 6)

| Punkt | E3 v150 | D v150 |
|---|---|---|
| Unit-Grenze in `Protocol_AA_CommandDispatch` | `(id & 0xFF) < 9` (identisch zu v151) | `< 9` |
| Pack-Datensätze | 8 × 0x60 B (`g_bmsPack0Voltage` @ 0x20014FAC + k·0x60, `g_bmsPack7Voltage` @ 0x2001524C) | 8 × 0x60 B |
| Modbus-Pack-Blöcke | 34000–34699, identische Deskriptoren | gleich |
| Multi-Pack-Auswertung | entfernt | vorhanden |
| Zellen pro Pack | 16 | 16 |

Die Control hat Platz für 8 Units, wertet aber ein Pack aus; wie viele Packs das E3-BMS tatsächlich meldet (`CAN_TX_PerPack_10Msgs`, `pack_id < N`, ab 1 zählen), klärt die E3-BMS-115-Analyse und ist nicht Teil dieses Images.

## 7. Tasks, Shell, Protokolle

Task-Tabelle `g_taskDefTable` @ 0x08057988 (14 Tasks, gleiche Namen/Stacks/Prioritäten wie v151), `shellCommandTable` @ 0x0805ECD0 (79 Einträge: 66 Befehle, Benutzer `VNSE3-0 v150` mit leerem Passwort,
10 Tastenbelegungen, `help`, `clear`; v151 ergänzt `read_inv1`, `read_inv2`), `g_mqttRpcMethodTable` @ 0x0805AB6C (16 Methoden), `mbedtls_cipher_definitions` @ 0x0805ACC4 (7), CH395-Sockets (502 Modbus TCP, 8091 Shell, 8090 UDP).

## 8. EEPROM-Karte

Gleiche Adressen wie v151 (`Config_LoadOrInit_DeviceSettings` @ 0x0802B28C: Identität, 0x200, 0x300, 0x900, 0x36BD, 0x500, 0x400 WLAN, 0x484/0x488/0x492 TOU-Zähler). v150 fasst
0x395 (LED-Flag) nicht an und liest den Servertyp 0x441 bei Bedarf statt beim Start.

## 9. Auffällige Strings

`VNSE3-0 v150`, `VNSEE3-0`, `MST_VNSE3_`, `VenusE 3.0`, `VenusE 3.0-%s`, `Build:\tAug  4 2026 15:11:03`, `[HTTP] Not found split mark \r\n\r\n, header incomplete`, `[HTTP]ch395 reset!!!!`,
die `getDateInfo`-URL auf `hamedata.com`, `AT+QHTTPPOST=%d,60,60`, das gekürzte `{"d":"di=…"}`-Upload-Format. Hersteller-Default-WLAN-Zugangsdaten und `quit_debug_5100_` sind wie auf der D vorhanden.

## 10. Sicherheitshinweise

Alle Control-Befunde der D v150 / E3 v151 gelten (identischer Code): Modbus 46000 = 0x5100 → unauthentifizierte Shell auf RS485 und TCP 8091, Shell-Benutzer mit leerem Passwort, Hersteller-Default-WLAN-Zugangsdaten,
Klartext-HTTP-`getDateInfo`. Die OTA-Typprüfung nutzt `strstr("VNSEE3-0", received_type)`, d. h. jeder Teilstring der Konstante besteht. Der HTTP-OTA-Reader verlässt sich darauf, dass der erste TCP-Lesevorgang den vollständigen Header enthält, und
prüft das Selektor-Ergebnis nicht (§4.7); das heruntergeladene Image wird danach per CRC16, Gerätemaske und Typ geprüft (`OTA_FW_Verify_SetStatus`); eine fehlgeschlagene Prüfung setzt den Slot-Status 0x401 / 0x402 / 0x403.

## 11. Bearbeitungsstand in Ghidra

- Abdeckung: 0 undefinierte Bytes, 0 verwaiste Befehle im Flash-Block; 1920 Funktionen, alle benannt, alle mit Prototyp (Task-Einstiege `void f(void *pvParameters)`); 3 manuelle Sprungtabellen-Overrides in `Write_Handler`, keine nicht erkannte Sprungtabelle im Export.
- `marstek_gates.py`: **alle Gates bestanden** (kein Code-als-Daten außer den zwei bekannten Padding-Fällen, keine Literal-only-Funktionen, keine FUN_/tmp_/Duplikate, 0 unbenannte referenzierte RAM-Globals, Task-Namen passen zur Task-Tabelle,
  No-Return-Flags plausibel); Dekompilat-Export 0 Fehler; Kurzstring-Prüfung gesichtet (`,` `.` `|` `]` `x` `eu` `ip` … sind echte Formatfragmente).
- Tags: `NAME_VERIFIED` auf allen 1920 Funktionen, davon 93 zusätzlich `NAME_LOW_CONFIDENCE` (gleiche Kennzeichnung wie v151).
- RAM-Globals: 1869 Labels (v151: 1900; der Unterschied besteht aus Variablen, die nur v151 hat, und einigen unreferenzierten Slots): rund 1100 aus der Abstimmung über ausgerichtete Befehle, 124 aus der Modbus-Deskriptor-Paarung, rund 570 aus der Anker-Interpolation, 50 Telemetrie-Datensatzfelder,
  einige von Hand, wo die Abstimmung falsch lag (Netzleistungs-Baseline-Block, BLE-MAC-ID-Zeichen, EPS-Zustandsbyte, Cloud-URL-Tabellen, Zähler-IP-String). 15 Innenadressen in 7 Funktionen erscheinen im Export noch als `uRam…`/`bRam…` — exakt dieselben Funktionen wie im v151-Export.
- Die Plates der 27 Funktionen, deren Verhalten von v151 abweicht, wurden für v150 neu geschrieben.

## 12. Offene Fragen

1. Genaue Bedeutung des `float32`-Deskriptors von 32102: er liest `g_batSamplePowerW` und das folgende 16-Bit-Wort als einen Float; ob sich je ein Client darauf verlassen hat, ist unbekannt (v151 entfernt ihn).
2. Zweck von `g_deviceInfoReserved8aa` (in v151 auf 0xA882 gesetzt, in v150 nicht vorhanden).
3. Kategorie 0x18 (nur v151 sendet sie, ID 0x00014118, Modbus 46000 / 47001 und MQTT `cd=0x42`; der Zweck steht nicht im Code): gegen E3-Micro 119 und E3-BMS 115 geprüft, beide ignorieren sie (Micro 119: kein Fall in `EMS_Inverter_CAN_Dispatcher`, Default-Ausgang; BMS 115: kein Treffer in `CAN_RX_Handler` / `CAN_RX_AuxCommand_Handler`). Auch Control hat keinen Handler für ein eingehendes 0x18. Offen: andere Knoten (FC41D usw.) und ob BMS und Control denselben physischen Bus teilen.
4. `Write_Handler`: die Konstanten wurden vollständig gegen v151 verglichen, ein Zweig-für-Zweig-Äquivalenzbeweis wurde aber nicht geführt.
5. Das Verhalten der zusammen mit v150 ausgelieferten E3-Micro- und BMS-Images liegt außerhalb dieser reinen Control-Analyse.
6. HTTP-OTA-Body-Start-Selector, Verhalten des OTA-Readers beim ersten Read, BLE 0x13 gegen OTA-Diagnose-Slots, die Kette 0x401/0x402/0x403 und Kategorie 0x18 über Versionen und Familien: siehe [OTA_HTTP_Body_Boundary_und_Diagnose_de.md](OTA_HTTP_Body_Boundary_und_Diagnose_de.md).

## Anhang: Methode

Offline-Segmentkarte der Images (maskierte Pointer und BL-Offsets), `ve3_150_layout_sync.py` / `ve3_150_port_segments.py` / `ve3_150_port_signatures.py` / `ve3_150_copy_by_address.py` (nur Peripherie) / `ve3_150_port_refs.py` (nur Flash-Ziele) /
`ve3_150_port_globals_aligned.py` / `ve3_150_ram_interp_port.py`, um Layout, Namen, Prototypen, Referenzen und RAM-Globals aus E3 v151 zu übernehmen; Byte-Match-Paarung der 34 ungepaarten Funktionen;
manuelle Typisierung des Image-Endes und der Literalpools; Durchsicht aller geänderten Funktionen per namensgepaartem Dekompilat-Diff; Verifikations-Gates. Die Skripte sind nur lokal (nicht im Repo).
