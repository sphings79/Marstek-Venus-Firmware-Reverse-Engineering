# Control-FW v149 (Venus D) — Vollanalyse

> 🇬🇧 English version: [VNSD_Control_FW_v149_Full_Analysis.md](VNSD_Control_FW_v149_Full_Analysis.md)

**Stand:** 2026-09-30.
**Englisch:** `VNSD_Control_FW_v149_Vollanalyse_de.md` ·
**Diff zum Vorgänger:** `VNSD_Control_Diff_v147_v149_de.md` · **Diff zum Beta-Nachfolger:** `VNSD_Control_Diff_v149_v1492_de.md` ·
**Diff zum Hauptnachfolger:** `VNSD_Control_Diff_v149_v150_de.md` · **Dekompilat/Listing:** `Dekompilate/VNSD_Control_149/` (lokal, nicht im Repo)

## 0. Kurzfassung

- Image-Basis **0x08004800** (18-KB-Bootloader darunter), Nations N32G45x (früher fälschlich als Artery AT32F403A
  angenommen, s. §1a), Keil ARMCC, FreeRTOS, mbedTLS 2.28, cJSON, llhttp, letter-shell, CH395, Quectel FC41D.
  Gleiche Codebasis wie v147/v150/v1492/v151.
- v149 ist der erste Stand, der **2500 W** Grenzleistung, die **16-Methoden-Local-API/MQTT-RPC**, das
  **246-Register-Modbus-Abbild** (30000–38014) und die **französische Zähleranbindung (TIC/Linky)** einführt —
  alles noch in v151 unverändert vorhanden (§0a, §7).
- Sicherheitslage wie in Vorgänger/Nachfolger unverändert: Modbus-Register 46000 = 0x5100 schaltet ohne
  Authentifizierung die Debug-Shell auf RS485 **und TCP 8091** dauerhaft frei (§8).

## 0a. Prüfliste (v149 gegenüber Vorgänger v147 und Nachfolger v150)

| # | Frage | Antwort v149 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register geändert? | Ggü. v147: **15 neu** (38000–38014, kumulierte Energiezähler), keine entfernt, 231 → **246** Einträge; neuer Schreibblock 46500–46544 (`Write_Handler`, `(reg-46500)/3`). Ggü. v150: Registerinhalt **unverändert** (Registermap/Deskriptortabelle über v149→v149.2→v150 identisch, nur RAM-Schattenadressen verschieben sich); das Schedule-EEPROM ab Register 46500 zieht aber im Schritt v149.2→v150 von 0x36BF nach 0x3700 um. | §4.1, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 2 | BLE geändert? | Ggü. v147: Befehlssatz erweitert um 0x5A (Cloud-Backup anstoßen) und 0x5B (IP/MAC/SN); 0x54 setzt zusätzlich EPS-Reserve-Flags zurück. Ggü. v150: Befehlssatz **wächst um 1** – neues Kommando 0x29 „Set Peak Shaving" (kommt erst im Schritt v149.2→v150 hinzu); außerdem ändern sich Antwort-/Telemetrieaufbau (Schritt v149→v149.2) sowie BMS-Selbsttest-/MOS-Flag und ein Fallthrough-Bugfix Y→Z (Schritt v149.2→v150). | §4.5, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 3 | AC-Einspeisegrenze 2500 W geändert? | Ggü. v147: **ja, eingeführt** (2200 W → 2500 W, `Config_ClampDefaults_ParamBlock`); `Config_Apply_WorkModePowerPreset` entfällt, dafür RPC `Set.Ver` (800/2200/2500 W). Ggü. v150: Hartgrenze **unverändert**, `Grid_Export_Power_Limiter` über beide Hops byte-identisch; **zusätzlich** (ersetzt die 2500-W-Grenze nicht) kommt im Schritt v149.2→v150 eine separate, konfigurierbare Peak-Shaving-Klemme (EEPROM 0x394) hinzu, siehe §5. | §5, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 4 | BLE-OTA weiter möglich? | **Ja**, in beide Richtungen unverändert: `BLE_Cmd_OTA_Init/WriteSetup/Validate`, YMODEM und alle `OTA_*`-Funktionen sind v147↔v149↔v149.2↔v150 byte-identisch bzw. funktionsgleich; YMODEM-Zielcodes 0x04D2 (EMS), 0x0929 (VENUS) unverändert (v149 kennt zusätzlich 0x0D80 und 0x11D7 als weitere Zielgeräte, s. §4.5). | §4.5, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 5 | Cloud-Domains geändert? | Ggü. v147: **unverändert** (`%s.hamedata.com`), neu nur der Pfad `/external-services/api/v1/devices/backup-configs`. Ggü. v150: `hamedata.com`-Endpunkte selbst **unverändert**, aber der Telemetrie-Upload (`Cloud_UploadTelemetryData`) wechselt im Schritt v149.2→v150 auf einen neuen HTTPS/JSON-POST an `marstekcloud.com` und ersetzt damit die AES-ECB+Base64URL-kodierte GET-Anfrage an `hamedata.com` (behebt dabei einen realen Pufferüberlauf, siehe §7). | §4.4, Diff v147→v149, Diff v1492→v150 |

## 1. Binary-Fingerprint

| Merkmal | Wert |
|---|---|
| Datei | `149_control_VNSD-0_app_0149_0521_094621.bin` |
| Größe | 385 024 Byte (0x5E000) |
| SHA-256 | `a8e5e9285e6ca5a327444d1bd942ece22a7955159e729b0dab0106f2bc8f8ace` (per `shasum -a 256` neu berechnet) |
| Version | Konstante **0x95 (149)** @ 0x08010000, String `VNSD-0` @ 0x08010004, Versionsstring `VNSD-0 v149` (u. a. @ 0x08050f50, @ 0x0805af21) |
| Laden | Basis 0x08004800, Vektortabelle dort (SP = 0x2001F778, Reset = 0x08004A71); `Reset_Handler` → `SystemInit` (setzt `SCB->VTOR = 0x08004800`) → `__main` → `__scatterload` → `main` |
| Scatter-Tabelle | `Region$$Table$$Base` @ 0x08060E58, 3 Einträge à 16 B (Quelle, Ziel, Größe, Routine): **.data** 0x08061368 → 0x20000000, 0x2EEC B, `__decompress` @ 0x08005FBE; **XMC (externes SRAM)** 0x08061D18 → 0x60000000, 0x242E4 B, `__decompress`; **.bss (ZI)** → 0x20002EEC, 0x1C88C B, `__scatterload_zeroinit` @ 0x08035BB0. Ende .bss = Initial-SP 0x2001F778 |
| Release-Notiz (Changelog) | „Weitere OpenAPI-Funktionen ergänzt; ES lieferte PV=0 und UPS-Fehlanzeige behoben; I²C-Deadlock-Recovery; französische Zähler; HTTP-Upload um IP/kumulierte Energie erweitert (nur VE3); Überschusseinspeisung optimiert; DOD-Übertragung behoben; Upload-Intervall 5 min; max. Leistung konfigurierbar; Venus D unterstützt 2500 W“ (Issue #4) |

## 1a. MCU-Identifikation: Nations N32G45x, nicht Artery AT32F403A

- Der Control/EMS-MCU ist ein **Nations N32G45x mit XFMC** (External Memory Controller), nicht der zuvor angenommene Artery AT32F403A.
- Belege: QSPI-Controller bei 0xA0001000 mit N32-QSPI_STS-Bits; RCC-PLL-Multiplikator in Bit 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG genutzt; SAC-Krypto-Coprozessor bei 0x40024000 (früher fälschlich „RNGX" genannt); 96-Bit-UID @ 0x1FFFF7F0; 102 Vektoren.
- Externes SRAM 0x60000000–0x60026AF7 läuft über XFMC, Steuerregister bei 0xA0000000: der offizielle Nations-Header `n32g45x.h` (lokal, nicht im Repo) definiert `XFMC_REG_BASE 0xA0000000` und `QSPI_BASE = XFMC_REG_BASE + 0x1000`; die Nations-Treiberbibliothek enthält `n32g45x_xfmc.c`; die Firmware selbst hat den Shell-Befehl `xfmc_test` (das aktuelle englische N32G45x-Benutzerhandbuch markiert diesen Adressraum fälschlich als „Reserved").
- Details/Header-Kopie: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Aufbereitungsstand in Ghidra (`/VNSD/Control/Control_149_VNSD-0_app_0149_0521_094621.bin`)

Alle Prüfschritte bestanden (`marstek_gates.py` (lokal, nicht im Repo), 2026-09-30):

| Prüfung | Ergebnis |
|---|---|
| Undefinierte Flash-Bytes / Instruktionen ohne Funktion | 0 / 0 |
| Code als Daten (Sprungziele, Funktionszeiger inkl. Tabelleneinträge, Literal-„Funktionen“) | 0 |
| Funktionen | **1931**, alle benannt, keine `FUN_`/`tmp_`/Duplikate |
| Prototypen | alle gesetzt (keine Default-Signatur) |
| Referenzierte RAM-/XMC-Globals ohne Namen | 0 |
| Task-Namen gegen `g_taskDefTable`, Shell-Handler gegen `shellCommandTable` | stimmen |
| Doppelte Global-Labels | keine |

## 3. FreeRTOS-Tasks (`g_taskDefTable` @ 0x0805990C, `TaskDef_t[14]`, 24 B/Eintrag)

Aus dem Flash-Image dekodiert (Funktion, Name, Stack in Wörtern, Priorität):

| Task | Priorität | Stack (Wörter) |
|---|---|---|
| Task_ch395 | 15 | 256 |
| Task_ch_mqtts | 14 | 1024 |
| Task_parser | 13 | 512 |
| Task_ch_http | 13 | 512 |
| Task_fc41d | 12 | 512 |
| Task_http | 11 | 512 |
| Task_udp | 10 | 256 |
| Task_ems | 9 | 256 |
| Task_ct | 8 | 512 |
| Task_Modbus_tcp | 7 | 512 |
| Task_can_comm | 7 | 256 |
| Task_can_receive | 6 | 256 |
| Task_rs485 | 3 | 512 |
| Task_iwdg | 1 | 128 |

`Task_CreateTasksAndStartScheduler` erzeugt alle 14 Tasks aus dieser Tabelle und startet den Scheduler; Priorität/Stack
sind identisch zu v151 (dort tabellarisch dokumentiert) — **über v149–v151 unverändert**.

## 4. Schnittstellen

### 4.1 Modbus (TCP 502 über CH395, RTU über RS485)

- Deskriptortabelle `g_modbusRegDescTable` @ RAM **0x20000348**, Typ `ModbusRegDesc_t[246]` (2952 B) — verifiziert
  (in v147 waren es 231 Einträge; v149 ergänzt die 15 Register **38000–38014**, kumulierte Energiezähler).
- Schreibpfad `Write_Handler` @ 0x080556D4: Sonderregister ab 45000 per `(reg-45000)`-Index ausgewertet.
  - **Register 46000** (Index 1000): Wert `0x5100` → `g_debugMode = 1`, dauerhaft in EEPROM 0x900 gespeichert.
    Wert `0x4D2` (1234) → `g_otaTargetDevice = 1` (EMS); `0x929` (2345) → `= 4` (VENUS); zusätzlich (nicht in den
    bekannten Fakten genannt, hier verifiziert) `0xD80` (3456) → `= 3` und `0x11D7` (4567) → `= 2` als weitere
    YMODEM-Zielgeräte.
  - Neuer Block **46500–46544** (0xB5A4–0xB5D0, `(reg-46500)/3`/`%3`, 15×3 Register) liest/schreibt
    `g_canScheduleRegCache`/`g_canScheduleRegCacheValueHi` (Zeitplan-Leistungsvorgaben je Slot) — in v147 nicht vorhanden.
- Variablen ohne fachlichen Namen heißen `g_mbReg<Register>`.

### 4.2 Local API / MQTT-RPC (`g_mqttRpcMethodTable` @ 0x0805CACC, `MqttRpcMethod_t[16]`)

Alle 16 Methoden aus dem Flash-Image gelesen (Name, interne ID):

| ID | Methode | ID | Methode |
|---|---|---|---|
| 1 | Marstek.GetDevice | 0xB | DOD.SET |
| 2 | Wifi.GetStatus | 0xD | Ble.Adv |
| 3 | BLE.GetStatus | 0xE | Led.Ctrl |
| 4 | Bat.GetStatus | 0xF | Ota.Set |
| 5 | PV.GetStatus | 0x10 | Set.Ver |
| 6 | ES.GetStatus | 0x11 | Reset.Factory |
| 7 | ES.SetMode | | |
| 8 | ES.GetMode | | |
| 9 | EM.GetStatus | | |
| 0xA | Marstek.GetData | | |

Gegenüber v147 (11 Methoden) sind **Ble.Adv, Led.Ctrl, Ota.Set, Set.Ver, Reset.Factory** neu
(`MQTT_JSON_RPC_Dispatcher` @ 0x0801E844, Case 0xD/0xE/0xF/0x10/0x11). `Set.Ver` (Case 0x10) akzeptiert exakt
800/0x898 (2200)/0x9C4 (2500) W und schreibt `g_workMode`; `DOD.SET` (Case 0xB) prüft den Bereich 0x1E–0x58 (30–88 %).

### 4.3 Debug-Shell (letter-shell, `shellCommandTable` @ 0x08060E88, `ShellCommand[78]`)

78 Tabelleneinträge (v147: 76) — 67 `ShellCmd_*`-Handlerfunktionen (65 Kommandos + `help` + `clear`) plus
Tastenbelegungen ohne eigenen Handler. Gegenüber v147 neu: **`get_backup_datas`**, **`power_check`** (bestätigt v147→v149-Diff).
Zugang über RS485-Debugkonsole und CH395-TCP-Server Port 8091 (`CH395_Init_TCPServer_Socket`,
`g_ch395SockTcpServer8091_localPort = 0x1F9B` = 8091) nach Freischaltung (§8).

**Korrigiert (2026-09-30):** Die Plate-Kommentare der vier Handler `update_bms/ems/mppt/vns` waren um eine Position
verschoben (z. B. trug `ShellCmd_update_bms` den Text von `FreeRtos_mem` — diesen Befehl gibt es erst ab v150).
Funktionsnamen waren korrekt; die Kommentare sind in Ghidra berichtigt (gleicher Fehler in v149.2, ebenfalls behoben).

### 4.4 Cloud

Endpunkte unter `http://%s.hamedata.com/…`: `/app/neng/getDateInfo%s.php`, `/ems/api/v1/getDevicesApply`,
`/prod/api/v1/setVenusDReporting` (Telemetrie-Upload), `https://%s.hamedata.com/external-services/api/v1/schedulings/policies`
und — **neu gegenüber v147** — `/external-services/api/v1/devices/backup-configs?devid=%s&latest=true`
(`HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher`). FC41D-Modul-OTA: `http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl`.
MQTT über TLS (mbedTLS, authmode 2), Topic `marstek_energy/%s/device/%s/ctrl` (App-seitig `.../App/%s/ctrl`).

### 4.5 BLE (Quectel FC41D, AT-Kommandos)

Befehlsverteiler `BLE_App_Command_Dispatcher` @ 0x0800C31C. Gegenüber v147 neu: **0x5A** (Cloud-Backup-Abruf
anstoßen, `bk_en`), **0x5B** (IP/MAC/Seriennummer); **0x54** (Entladegrenze) setzt zusätzlich die EPS-Reserve-Flags
zurück. OTA über BLE: `BLE_Cmd_OTA_Init`/`WriteSetup`/`Validate` (@ 0x0803301C/0x080330B0/0x08032F3C) + YMODEM
(`YModem_ReceiveFile`, `YModem_FirmwareUpgrade_Handler`) — BLE-OTA ist möglich und zu v147/v150 unverändert.

## 5. Leistungsgrenzen (für die 2500-W-Frage)

- `Config_ClampDefaults_ParamBlock` @ 0x0800901C (verifiziert): Lade- **und** Entladeleistung werden auf
  **0x9C4 = 2500 W** begrenzt; Default bei ungültigem Wert: Entladen 800 W, `g_workMode = 1`. (v147: 0x898 = 2200 W.)
- `Config_Apply_WorkModePowerPreset` aus v147 (800/600/2200/1200 W je Modus) **entfällt in v149** — ersetzt durch
  die RPC-Methode **`Set.Ver`** (`MQTT_JSON_RPC_Dispatcher` Case 0x10): akzeptiert nur 800, 0x898 (2200) oder
  0x9C4 (2500) W, schreibt `g_workMode` (1/3/0) und persistiert per `Config_Write_U16`/EEPROM 0x90.
- `Grid_Export_Power_Limiter` @ 0x08022ED8 (verifiziert): Einspeise-/Ladesollwert wird auf `min(P_bms, 0x9C4)`
  begrenzt, danach auf eine Gesamtklemme **4000 W** — unverändert zu v147 und v150 (siehe Diff-Dokumente).
- Die 2500-W-Hartgrenze bleibt bis v151 bestehen; **erst in v150** kommt zusätzlich eine separate, konfigurierbare
  Peak-Shaving-Klemme hinzu (EEPROM 0x394, BLE-Kommando 0x29, MQTT-Kommandos 0x3f/0x41) — sie ersetzt die
  2500-W-Grenze nicht, sondern wirkt zusätzlich in `Inverter_PowerSetpoint_DeadbandClamp`/
  `Remote_Power_Setpoint_Process` (Diff v1492→v150).

## 6. Struktur-/Datenbefunde

- mbedTLS-Cipher-Tabelle direkt hinter `g_mqttRpcMethodTable` (`aes_info` @ 0x0805CB4C, `aes_128/192/256_ecb_info`
  ff.) — gleicher Aufbau wie in v147/v151.
- `Task_CreateTasksAndStartScheduler` @ 0x0801D720 erstellt alle 14 Tasks aus `g_taskDefTable` und protokolliert
  Erfolg/Fehler je Task per `Log_Printf`.
- **BMS-Pack-Anzahl (gilt für 147–151):** 8 Pack-Records im RAM, Batteriezahl vom CAN auf 8 begrenzt, Modbus-Blöcke
  34000…34600 (7 Packs). **Pack-Record-Größe in v149: 0x2C × 2 B = 88 Byte** je Pack (verifiziert über
  `CAN_AA_StoreUnitFieldPair0` @ 0x08033338, Indexschritt `iVar1 * 0x2c` auf `uint16_t`-Basis; v147 hatte 80 B, ab
  v149.2 sind es 96 B). Die Plate-Kommentare der `CAN_AA_StoreUnitFieldPair*`-Handler nannten
  fälschlich 96 B und der Handler für Offset 0x48 hieß `…Pair88` — beides in Ghidra korrigiert (jetzt
  `CAN_AA_StoreUnitFieldPair72`, Kommentare mit 88 B). Die Felder aller 8 Records heißen jetzt einheitlich
  `g_bmsPack<k><Feld>` (vorher teils `g_paramStore_*`/`g_mbReg341xx`).
- `BMS_CellVoltage_MinMax_PerPack` @ 0x08017348 (früher fälschlich `Schedule_MinMax_PerGroup_Calc`) läuft nur über
  **6 Packs** (`uVar3 < 6`, je 16 Zellen aus `g_bmsPack0CellVoltageArray`, Schritt `uVar3*0x2c`) in die Tabelle
  `g_bmsCellStat0…` (6 × 6 B) — der 7. Pack (Index 6) fehlt in dieser Statistik und der daraus gebauten
  MQTT-Telemetrie, ist aber per Modbus (34600-Block) vorhanden. Gilt für alle Versionen 147–151.
- **Pack-Nummerierung / Versatz im MQTT-Pfad (gilt für 147–151):** CAN-Einheit u → Record u−1
  (`CAN_AA_StoreUnitFieldPair*`); Modbus 34000+100·n und `ShellCmd_get_all_bms_data` lesen Record n (konsistent).
  `MQTT_Publish_BMS_Data` liest für Index p ≥ 1 dagegen Record p → Off-by-one (MQTT „BMS(1)“ = CAN-Einheit 2,
  Einheit 1 per MQTT nicht einzeln abrufbar); Index 8 liest hinter das Record-Array (reiner Lesezugriff auf
  `g_pendingBootDescriptor`-Bereich, kein Speicherfehler).

## 6a. Chinesische Strings

Das Export enthält 127 UTF-8-Strings mit Nicht-ASCII-Zeichen; **106 davon enthalten chinesischen Text und sind
bereits in Ghidra per EOL-Kommentar `ZH->EN` ins Englische übersetzt** (z. B. Shell-Hilfetexte für praktisch jedes
`ShellCmd_*`-Kommando, FC41D/BLE-Reset-Meldungen, TLS-Fehlermeldungen). Die restlichen 21 Strings sind rein
symbolisch (Pfeile `←`/`→`, Rahmenzeichen `◆◇■□`, Gradzeichen `°`) und benötigen keine Übersetzung. Auswahl mit
deutscher Übersetzung:

| String (Original) | Deutsch |
|---|---|
| `*** 模组复位, ble_state: %d.` | *** Modul(FC41D)-Reset, ble_state: %d. |
| `CRC校验不通过` | CRC-Prüfung fehlgeschlagen |
| `设备各项校验通过，本次升级成功！！！` | Alle Geräteprüfungen bestanden, Update erfolgreich!!! |
| `恢复出厂设置（中控） 1:清除所有; 2:保留累计数据; 3:恢复证书` | Werksreset (Zentralsteuerung): 1 alles löschen; 2 kumulierte Daten behalten; 3 Zertifikate wiederherstellen |
| `复位原因：独立看门狗复位` | Reset-Grund: unabhängiger Watchdog-Reset |
| `[BLE] 余电上网中!!!!` | [BLE] Überschusseinspeisung aktiv!!! |
| `47400要写进去里面的数据 = 0x%x` | In 47400 zu schreibender Wert = 0x%x |
| `没到里面吗123456` | „Nicht reingekommen? 123456“ (Entwickler-Debugausgabe im Testzweig von Register 47400) |

Die beiden letztgenannten Strings stehen im `Write_Handler`-Testzweig um Register 47400 — dieselbe Stelle, die im
v151-Dokument als „zwei chinesische Debug-`printf`“ beschrieben ist; in v149 tragen beide bereits eine
`ZH->EN`-Übersetzung.

## 7. Nur in v149 / geändert ggü. Vorgänger v147 und Nachfolgern (Kurzfassung, Details in den Diff-Dokumenten)

**Neu ggü. v147** (Diff `VNSD_Control_Diff_v147_v149_de.md`, 96 geänderte, 42 neue, 23 entfallene Funktionen):
- Leistungsobergrenze 2200 W → 2500 W, `Set.Ver`-RPC statt `Config_Apply_WorkModePowerPreset`.
- Local-API/MQTT-RPC 11 → 16 Methoden (+ Ble.Adv, Led.Ctrl, Ota.Set, Set.Ver, Reset.Factory).
- Modbus 231 → 246 Register (38000–38014 kumulierte Energiezähler); Schreibblock 46500–46544 neu.
- Französische Zähleranbindung (TIC/Linky) in `SMR_TIC_SelfUse_PowerController`/`CT_GridPower_Controller` neu.
- `Cloud_Report_FillGridData` liefert PV-Summe/-Jahreskapazität jetzt real statt fest 0.
- DOD-Bereichsprüfung (30–88 %) in `Cloud_DOD_Command_Handler` neu (v147: immer Erfolg, ohne Prüfung).
- I²C-EEPROM-Bus-Recovery (`EEPROM_I2C_BusRecovery`), PV-Überschuss-Ladeboost wird jetzt angewendet (v147: toter Code).
- BMS-Pack-Record 80 B → 88 B; Cloud-Backup-Konfiguration (BLE 0x5A, MQTT `bk_en`) neu; Shell +2 Kommandos.

**Geändert im Beta-Nachfolger v149.2** (Diff `VNSD_Control_Diff_v149_v1492_de.md`: **58 geänderte Funktionen**
[38 inhaltlich, 20 kosmetisch], **4 neue** [`CAN_AA_StoreUnitFieldPair88`, `CT_PowerSetpoint_Compute`,
`Config_Read_String_0x387`, `Config_Write_String_0x387`]): BMS-Pack-Record 88 B → 96 B (neues SoH-Feld);
CT/SMR-TIC-Sollwertregler in zwei Funktionen aufgeteilt; neuer Geräte-Alias-Name (EEPROM 0x387); Umbau von
Cloud-/MQTT-Telemetrie, Status-Reporting und Telemetrie-Historie. **Modbus und Einspeisegrenzen bleiben dabei
unverändert; der BLE-Befehlssatz bekommt keine neue Kommando-ID, aber Antwort-/Payloadfelder ändern sich**
(§4.5-Diff).

**Geändert im Hauptnachfolger v150 (Schritt v149.2 → v150)** (Diff `VNSD_Control_Diff_v1492_v150_de.md`: **56
geänderte Funktionen** [26 inhaltlich], **20 neue**, **1 entfallene** — deutlich mehr als der kleine
Versionssprung vermuten lässt, v149.2 ist **nicht** „praktisch v150"): Cloud-Telemetrie-Upload komplett
umgebaut — ein neuer HTTPS/mbedTLS-Client sendet ein JSON-POST an
`https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` und ersetzt die bisherige AES-ECB+Base64URL-kodierte
GET-Anfrage an `hamedata.com`; dabei wird ein realer Pufferüberlauf behoben (`Base64URL_Encode` wurde in v149.2
mit einer behaupteten Zielgröße von 0x960 = 2400 B aufgerufen, der tatsächliche Puffer `g_cloudUploadBodyBuf`
ist aber nur 1200 B groß). Weitere neue Funktionen/Merkmale: Peak-Shaving-/Leistungsklemm-Funktion (EEPROM
0x394, BLE-Kommando 0x29, MQTT-Kommandos 0x3f/0x41, neue Funktion `MQTT_Publish_Grid_Status`); zwei neue
Watchdogs bei hängenden Uploads (CH395-Chip-Reset bzw. Modem-Hardware-Reset); `DeviceInfo_BuildStatusString`
bekommt feste Hardware-Tags für 6 weitere Zählertypen; `Grid_PowerSync_StateMachine` erweitert den
Zählertypbereich für die Direktübernahme; BMS-Selbsttest-/MOS-Flag zusätzlich gemeldet; neuer Shell-Befehl
`ShellCmd_FreeRtos_mem` (freier/minimal-freier FreeRTOS-Heap). **Korrektur gegenüber einer früheren
Analysefassung:** Die zuvor als einzige drei Unterschiede genannten „HTTP-Upload-Leseoffset +5→+7",
„CH395-Socket-Parameter 0x24→0xAA" und „HTTP-GET-Puffer 360→640 B" waren reine `__LINE__`-Log-Argumente ohne
Verhaltensänderung; die tatsächliche Änderung an `HTTP_ParseUploadResponseCode` ist ein anderer JSON-Schlüssel
(`"code":` statt `"de":`). Modbus-Registerinhalt, die 2500-W-Hartgrenze, OTA/BLE-OTA und die
`hamedata.com`-Endpunkte selbst bleiben unverändert (§0a).

## 8. Sicherheitsbefunde (verifiziert in v149)

1. **Unauthentifizierte Fernkonsole:** Modbus-TCP-Schreibzugriff (Port 502, ohne Auth) auf Register 46000 = 0x5100
   → `g_debugMode = 1` dauerhaft im EEPROM (0x900) → letter-shell auf RS485 und TCP 8091
   (`CH395_Init_TCPServer_Socket`, Port 0x1F9B = 8091) — identisch zu v147/v150/v151.
2. **Fest eingebaute WLAN-Zugangsdaten** `SM471` / `Hame147258` (`Quectel_WiFi_AutoConnect_StateMachine` @
   0x08015FB8, `Config_LoadOrInit_DeviceSettings` @ 0x0802CD0C).
3. Cloud-Upload über unverschlüsseltes HTTP (`http://%s.hamedata.com/prod/api/v1/setVenusDReporting`); Nutzdaten
   AES-ECB + Base64URL (`Base64URL_Encode`, vorhanden bis v149.2, entfällt ab v150).
4. Neuer Cloud-Backup-Abruf-Pfad (`backup-configs`) ebenfalls über Klartext-HTTP.

## 9. Offene Punkte

- Changelog-Punkt „HTTP-Upload-Intervall 5 min“: im Code kein Unterschied zu v147 gefunden (bereits im v147-Doc
  vermerkt) — nicht belegbar.
- Die exakte Bedeutung der 15 Bytefelder des Telemetrie-Historie-Records ist aus v149 allein nicht abschließend
  bestimmbar (neutrale Offsetnamen, siehe v147-Doc §9).
- Ob die vom Server verwendete BMS-Indexzählung 0- oder 1-basiert ist (relevant für den in §6 beschriebenen
  MQTT-Off-by-one), lässt sich aus der Firmware nicht feststellen.
- Zwei zusätzliche YMODEM-Zielcodes (0xD80 → Ziel 3, 0x11D7 → Ziel 2, siehe §4.1) sind im Code vorhanden, aber ohne
  Shell-/BLE-/MQTT-Aufrufpfad in diesem Export gefunden — welches Zielgerät „3“ bzw. „2“ tatsächlich ist, bleibt offen.

## 10. Korrekturen

Im Rahmen der v149-Arbeit wurden keine neuen Korrekturen an bereits „fertigen“ Versionen gefunden; die in §10 des
v147-Dokuments gelisteten Korrekturen (Task-Trampoline, Prototypen, letter-shell-Vertauschungen, mbedTLS-Cipher-
Bereich, Task-Tabellen-Fehlbenennung, `CH395_ConfigTcpKeepalive_Defaults`, `MQTT_Transport_NullCallback`) betreffen
149/149.2/150/151 gemeinsam und sind dort bereits behoben. Zusätzlich am 2026-09-30 in Ghidra
korrigiert (siehe §4.3, §6): verschobene Plate-Kommentare der `update_*`-Shell-Handler, falsche Record-Größe in den
`CAN_AA_StoreUnitFieldPair*`-Kommentaren, Name `…Pair88` → `…Pair72`, einheitliche Namen der BMS-Pack-Records und
versionsübergreifend vereinheitlichte Namen globaler Variablen (gleiches Objekt = gleicher Name wie in v151).
- Nacharbeit 2026-10-12 (wie bei D 151 / A 1509): `Inverter_RegDirty_Mark_DischgVolt` → `Inverter_RegDirty_Mark_GridPermit` (übergibt 1, solange das Micro-Telemetriewort `g_invState+0x16` 0 ist) und `Inverter_RegDirty_Mark_BatteryParams` → `Inverter_RegDirty_Mark_PvVoltage` (übergibt 1, solange alle vier PV-String-Spannungen unter 0x8D = 14,1 V liegen); das Code-Muster wurde in diesem Image geprüft. Das „Inverter-Dirty-Bitmap" ist das Frontpanel-LED-Bitmap eines TM1650-artigen Treibers (Bit-Bang-I2C auf die Digit-Register 0x68/0x6A/0x6C/0x6E); Plate-Kommentare der LED-Wrapper und Globals aktualisiert. Gates erneut gelaufen: alle bestanden; Dekompilat- und Listing-Export aktualisiert.
