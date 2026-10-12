# Control-FW v149.2 (Beta) (Venus D) — Vollanalyse

> 🇬🇧 English version: [VNSD_Control_FW_v1492_Full_Analysis.md](VNSD_Control_FW_v1492_Full_Analysis.md)

**Stand:** 2026-09-30. Ersetzt für alle Adress- und Namensangaben die frühere Fassung
`VNSD_Control_FW_Analyse_app_1492_0702_142136_de.md` (Modbus-Register-Doku aus Live-Scans, vor dem Basis-Fix
0x08004800; dort zitierte Flash-Adressen sind durchweg um 0x4800 zu niedrig, einige Namen falsch). Diese
Live-Messungen selbst (Register-Verhalten) bleiben gültig und sind, soweit hier bestätigt, übernommen.
**Englisch:** `VNSD_Control_FW_v1492_Vollanalyse_de.md` · **Diff zum Vorgänger v149:**
`VNSD_Control_Diff_v149_v1492_de.md` · **Diff zum Nachfolger v150:** `VNSD_Control_Diff_v1492_v150_de.md` ·
**Dekompilat/Listing:** `Dekompilate/VNSD_Control_1492/` (lokal, nicht im Repo)

## 0. Kurzfassung

v149.2 ist ein **Beta** zwischen den Hauptständen v149 (21.05.2026) und v150 (05.08.2026), veröffentlicht
02.07.2026 (Issue #17: „Fehlerhafte Datenanzeige durch ausbleibende HTTP-Uploads behoben"). **Es ist NICHT
praktisch identisch mit v150:** gegenüber v149 ändert der Beta-Schritt bereits 58 Funktionen (38 inhaltlich,
20 kosmetisch) und bringt 4 neue hinzu; danach folgen bis v150 nochmals **56 geänderte Funktionen (26
inhaltlich), 20 neue und 1 entfallene** — u. a. wechselt der Telemetrie-Upload von einer AES-ECB+Base64URL-
kodierten GET-Anfrage an `hamedata.com` auf einen neuen HTTPS/mbedTLS-POST an `marstekcloud.com` (dabei wird
ein realer Pufferüberlauf behoben), kommt die Peak-Shaving-Klemmfunktion (EEPROM 0x394) hinzu, und zwei neue
Watchdogs reagieren auf hängende Uploads. Modbus-Registerinhalt, die 2500-W-Hartgrenze, die
BLE-Kommando-IDs und OTA/BLE-OTA sind gegenüber v149 **unverändert**; die inhaltlichen Änderungen im
v149→v149.2-Schritt liegen in Cloud-/MQTT-Telemetrie, Status-Reporting, dem BMS-Pack-Record (neu 96 statt
88 B) und dem französischen Eigenverbrauchsregler (SMR/TIC). Sicherheitslage wie in allen Versionen 147–151:
Modbus-Register 46000 = 0x5100 schaltet ohne Authentifizierung die Debug-Shell auf RS485 **und TCP 8091**
dauerhaft frei (§8).

## 0a. Prüfliste für die Update-Entscheidung (v149 → v149.2 → v150)

| # | Frage | v149 → v149.2 | v149.2 → v150 |
|---|---|---|---|
| 1 | Modbus-Register geändert? | **Unverändert.** `RS485_Modbus_RegisterMap_Init`, `Modbus_Dispatcher`, Lese-/Schreibpfad byte-identisch; Deskriptortabelle 246 Einträge (0x20000354) in beiden Ständen | **Registerinhalt unverändert** (alle 246 Deskriptoren: Nummer/Typ/Skalierung/Anzahl identisch; nur bei 215 Einträgen verschiebt sich die RAM-Schattenadresse durch ein um 6 B gewachsenes Config-Struct). Das Schedule-EEPROM ab Register 46500 zieht aber von 0x36BF nach 0x3700 um |
| 2 | BLE geändert? | **Befehlssatz gleich**, nur Antwort-/Telemetrie-Aufbau: `BLE_App_Command_Dispatcher` (~145 Ops geändert), `BLE_Build_Cmd0x42_Response` (15 Ops), `BLE_Build_BMS_Data_Response` (3 Ops). Kein neuer/entfernter Befehl | **Erweitert.** Neues Kommando `)` (0x29) „Set Peak Shaving" (Klemm-Modus + Watt-Wert, EEPROM 0x394); `BLE_Build_DeviceStatus_Response` meldet zusätzlich Klemm-Modus/-Wert (0xA4→0xA7 Byte); `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` melden zusätzlich Selbsttest-/MOS-Flag (0x50→0x51 Byte); Fallthrough-Bug Kommando `Y`→`Z` behoben |
| 3 | AC-Einspeisung 2500 W geändert? | **Unverändert.** `Grid_Export_Power_Limiter` byte-identisch; alle 4 Fundstellen der 2500-Konstante (0x9C4) gleich | **Hartgrenze unverändert** (`Config_ClampDefaults_ParamBlock`, `Grid_Export_Power_Limiter` weiterhin byte-identisch); zusätzlich (separat, ersetzt die 2500-W-Grenze nicht) neue konfigurierbare Peak-Shaving-Klemme (EEPROM 0x394), wirkt in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` |
| 4 | Firmware-Update per BLE weiter möglich? | **Ja**, komplett byte-identisch: `BLE_Cmd_OTA_Init/WriteSetup/Validate`, `YModem_FirmwareUpgrade_Handler`, alle `OTA_*` | **Ja** (kompletter OTA-Pfad byte-identisch) |
| 5 | Cloud-Domains geändert? | **Unverändert.** Identischer `hamedata.com`-Endpunktsatz in beiden Ständen | **`hamedata.com`-Endpunkte selbst unverändert**, aber der Telemetrie-Upload (`Cloud_UploadTelemetryData`) wechselt auf ein neues HTTPS/JSON-POST an `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (ersetzt die AES-ECB+Base64URL-GET-Anfrage an `hamedata.com`) — `marstekcloud.com` existiert also **bereits ab v150**, nicht erst ab v151 |

Vollständige Tabellen mit Fundstellen: `VNSD_Control_Diff_v149_v1492_de.md` (58 geänderte Funktionen [38 inhaltlich,
20 kosmetisch], 4 neue; Schwerpunkt Cloud/MQTT-Telemetrie, Status-Reporting, franz. Eigenverbrauchsregler) und
`VNSD_Control_Diff_v1492_v150_de.md` (**56 geänderte Funktionen [26 inhaltlich], 20 neue, 1 entfallene** —
HTTPS/mbedTLS-Cloud-Upload zu `marstekcloud.com`, Peak-Shaving, zwei Upload-Watchdogs u. a.; eine frühere
Analysefassung hatte hier fälschlich nur 3 reine `__LINE__`-Log-Konstanten als „echte Änderungen" gewertet).
Details auch in §7.

## 1. Binary-Fingerprint

| Merkmal | Wert |
|---|---|
| Datei | `1492_control_VNSD-0_app_1492_0702_142136.bin` |
| Größe | 385 024 Byte (0x5E000) |
| SHA-256 | `9b24db59c4dbde6c17e1666a778c8f44618558ef2760268a4be95570fc450796` (deckt sich mit Changelog-Präfix `9b24db59c4db…`) |
| Version | Konstante **0x5D4 (1492)** @ 0x08010000, String `VNSD-0` @ 0x08010004, Versionsstring `VNSD-0 v1492` @ 0x0805af6d |
| Laden | Basis 0x08004800 (18-KB-Bootloader darunter, nicht Teil des Images); Reset-Vektor → `SystemInit` (setzt `SCB->VTOR = 0x08004800`) → `__main` → `__scatterload`; Image-Ende 0x08062800 (0xFF-Füllung danach) |
| Scatter-Tabelle | `Region$$Table$$Base` @ 0x08060ec8 (3 × 16 B): **.data** Quelle 0x080613d8 (2488 B komprimiert) → Ziel 0x20000000, 12 028 B (0x2EFC) über `__decompress` @ 0x08005fbe; **XMC (.xmc_bss)** Quelle 0x08061d90 (2656 B komprimiert) → Ziel 0x60000000, 148 996 B (0x24404), gleicher `__decompress`; **.bss** Zero-Init ab 0x20002EFC, 117 468 B (0x1C8DC) über Routine @ 0x08035c08 |
| Release-Notiz | „Fehlerhafte Datenanzeige durch ausbleibende HTTP-Uploads behoben" (Changelog, Issue #17) |

## 1a. MCU-Identifikation: Nations N32G45x, nicht Artery AT32F403A

- Der Control/EMS-MCU ist ein **Nations N32G45x mit XFMC** (External Memory Controller), nicht der zuvor (in älteren Dokumentversionen) angenommene Artery AT32F403A.
- Belege: QSPI-Controller bei 0xA0001000 mit N32-QSPI_STS-Bits; RCC-PLL-Multiplikator in Bit 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG genutzt; SAC-Krypto-Coprozessor bei 0x40024000 (früher fälschlich „RNGX" genannt); 96-Bit-UID @ 0x1FFFF7F0; 102 Vektoren.
- Externes SRAM 0x60000000–0x60026AF7 läuft über XFMC (s. Scatter-Tabelle oben, `.xmc_bss`), Steuerregister bei 0xA0000000: der offizielle Nations-Header `n32g45x.h` (lokal, nicht im Repo) definiert `XFMC_REG_BASE 0xA0000000` und `QSPI_BASE = XFMC_REG_BASE + 0x1000`; die Nations-Treiberbibliothek enthält `n32g45x_xfmc.c`; die Firmware selbst hat den Shell-Befehl `xfmc_test` (das aktuelle englische N32G45x-Benutzerhandbuch markiert diesen Adressraum fälschlich als „Reserved").
- Details/Header-Kopie: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Aufbereitungsstand in Ghidra (`/VNSD/Control/Control_1492_VNSD-0_app_1492_0702_142136.bin`)

Alle Prüfschritte bestanden (`marstek_gates.py` (lokal, nicht im Repo), 2026-09-30):

| Prüfung | Ergebnis |
|---|---|
| Undefinierte Flash-Bytes / Instruktionen ohne Funktion | 0 / 0 |
| Code als Daten (Sprungziele, Funktionszeiger inkl. Tabelleneinträge, Literal-„Funktionen") | 0 |
| Funktionen | **1935**, alle benannt, keine `FUN_`/`tmp_`/Duplikate |
| Prototypen | alle gesetzt (keine Default-Signatur) |
| Referenzierte RAM-/XMC-Globals ohne Namen | 0 |
| Task-Namen gegen `g_taskDefTable`, Shell-Handler gegen `shellCommandTable` | stimmen |
| Doppelte Global-Labels | keine |

## 3. FreeRTOS-Tasks (Tabelle `g_taskDefTable` @ 0x08059958, `TaskDef_t[14]`, 24 B/Eintrag)

| Task | Prio | Stack (Wörter) | Funktion |
|---|---|---|---|
| Task_ch395 | 15 | 256 | 0x08032305 |
| Task_ch_mqtts | 14 | 1024 | 0x08032325 |
| Task_parser | 13 | 512 | 0x080323b9 |
| Task_ch_http | 13 | 512 | 0x08032317 |
| Task_fc41d | 12 | 512 | 0x08032375 |
| Task_http | 11 | 512 | 0x0803238d |
| Task_udp | 10 | 256 | 0x0803240d |
| Task_ems | 9 | 256 | 0x08032361 |
| Task_ct | 8 | 512 | 0x08032347 |
| Task_can_comm | 7 | 256 | 0x080322e9 |
| Task_Modbus_tcp | 7 | 512 | 0x080322c9 |
| Task_can_receive | 6 | 256 | 0x080322f7 |
| Task_rs485 | 3 | 512 | 0x080323cd |
| Task_iwdg | 1 | 128 | 0x080323a5 |

14 Einträge, alle mit Handle in `TaskDef_t.pHandle` (0x20000000–0x20000034); Erzeugung in
`Task_CreateTasksAndStartScheduler` @ 0x0801dc7c (Schleife über `g_taskDefTable[0..13]`,
`FreeRTOS_xTaskCreate`, danach `vTaskStartScheduler`). Werte direkt aus dem Binary dekodiert (kein
`Task_fc41d`-Sonderfall wie in der v151-Doku beschrieben — hier vollständig regulärer Tabelleneintrag 0).

## 4. Schnittstellen

### 4.1 Modbus (TCP 502 über CH395, RTU über RS485)

- Deskriptortabelle `g_modbusRegDescTable` @ **0x20000354** (identisch mit v150, abweichend von v151s
  0x2000034C), `ModbusRegDesc_t[246]`, Register 30000–38014 — unverändert seit v149 (die 15 Register
  38000–38014 aus v149 sind enthalten).
- `Write_Handler` @ 0x08055720 (Register ≥ 40000 und Sonderadressen 0xA0xx–0xB7xx):
  - **Register 46000** (intern `iVar8 == 1000`, 45000+1000): Wert **0x5100** → `g_debugMode = 1`,
    `EEPROM_Write(0x900, …)` — Debug-Shell dauerhaft frei (§8).
  - YMODEM-Zielauswahl am selben Register: `0x04D2` → Ziel 1 (EMS), `0x0929` → Ziel 4 (VENUS),
    `0x0D80` → Ziel 3 (MPPT), `0x11D7` → Ziel 2 (BMS) — vier statt der in v151 dokumentierten drei Werte.
  - Block **46500–46544** (0xB5A4–0xB5D0, 15 × 3 Register, `Inverter_Set_Schedule_Reg`,
    EEPROM 0x36BF+8·n) — seit v149 vorhanden.
  - Fern-Lade-/Entladeleistung (Register 45900 / 45901 / 0xA424 / 0x19B-Block): alle mit Klemme
    `< 0x9C5` (≤ 2500 W) geprüft — konsistent mit §5.
  - Register **47400** (`iVar8 == 0x960`): unveränderter Testzweig mit zwei chinesischen
    Debug-`printf`s — schon in v149.2 vorhanden, nicht erst ab v151 (§11).
- Kein Register 0xB799 (47001) und kein `CAN_Write_Category0x18` — beides erst v151.

### 4.2 Local API / MQTT-RPC (`g_mqttRpcMethodTable` @ 0x0805cb1c, 16 Methoden, dispatcht in
`MQTT_JSON_RPC_Dispatcher` @ 0x0801eda4)

Marstek.GetDevice (1), Wifi.GetStatus (2), BLE.GetStatus (3), Bat.GetStatus (4), PV.GetStatus (5),
ES.GetStatus (6), ES.SetMode (7), ES.GetMode (8), EM.GetStatus (9), Marstek.GetData (10), DOD.SET (11),
Ble.Adv (13), Led.Ctrl (14), Ota.Set (15), Set.Ver (16), Reset.Factory (17). Identisch mit v149 (die
5 in v149 neuen Methoden sind vorhanden, keine weitere Änderung). `Set.Ver` (Fall 0x10) akzeptiert genau
800 W (→ `g_workMode=1`), 0x898=2200 W (→ `g_workMode=3`) oder 0x9C4=2500 W (→ `g_workMode=0`).

### 4.3 Debug-Shell (letter-shell, `shellCommandTable` @ 0x08060ef8, 78 Einträge × 16 B)

**65 Kommandos** + 1 Benutzerdatensatz (`VNSD-0 v1492`, Passwortfeld leer) + 10 Tastenbelegungen
(↑/↓/→/←, Tab, 2× Backspace, Entf, 2× Enter) + `help`/`clear`. Identisch mit v149 (63 + `get_backup_datas`,
`power_check`); keine Änderung gegenüber v149, keine der v151-Neuerungen (`read_inv1`, `read_inv2`).

Kommandoliste (dekodiert): set_level, debug_fc, test, check_periph, xfmc_test, get_ver, set_old, get_old,
change_soc, reset, flash_test, eeprom_flash_test, ch395_get_ip, ch395q_test, rtc_show, rtos_status,
update_ems, ctl_mppt_power, update_vns, update_bms, update_mppt, Resume_task, restore_factory, debug_mode,
venus_poweroff, venus_rfd, get_mppt_data, get_venus_data, get_main_bms_data, get_all_bms_data,
delate_ssl_cert, check_ssl_cert, set_wifi, mac_ble, ver_fc, fc41d_reset, trace_debug, modbus, show_modbus,
get_log, wifi_info, api_set, api_port, get_time, vns_ate_test, ctrl_led_light, energy_show, err_code,
ext_info, reset_reason, button_ctrl, led_continue_ctrl, recon_ch395, hard_reset_395, ch395_ver, key_state,
code_test, lck_bms, ems_rtrl_info, full_dischrg, free_mqtts, get_meter_mac, get_backup_datas, power_check,
get_ota_log.

**Erreichbarkeit:** über RS485-Debugkonsole und CH395-TCP-Server Port **8091** (0x1F9B,
`CH395_Init_TCPServer_Socket` @ 0x0803740c, Socket-Deskriptor `g_ch395SockTcpServer8091` in der
Socket-Tabelle @ 0x2001adcc), aktiviert über Modbus 46000 = 0x5100 (§8).

### 4.4 Cloud

`hamedata.com`-Endpunkte (unverändert seit v147, kein `marstekcloud.com` — das kommt erst mit v151):

| Pfad | Schema | Funktion |
|---|---|---|
| `/app/neng/getDateInfo%s.php?uid=…` | HTTP | `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` |
| `/prod/api/v1/setVenusDReporting?v=%s` | HTTP | `Cloud_UploadTelemetryData` (Haupt-Telemetrie-Upload) |
| `/ems/api/v1/getDevicesApply?devid=%s` | HTTP | WLAN-Registrierungspfad |
| `/external-services/api/v1/devices/backup-configs?devid=%s&latest=true` | HTTP | Cloud-Backup-Konfiguration (seit v149) |
| `/external-services/api/v1/schedulings/policies?devid=%s` | HTTPS | Zeitplan-/Policy-Abruf |
| `www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl` | HTTP | FC41D-Modul-OTA (`AT+QWLANOTA=`) |

Regionstabelle (Index aus EEPROM-Feld, ausgewählt in `%s.hamedata.com`): nur 3 Einträge
`eu, www, us` — kein `dev`/`staging` (das kommt erst mit v151; Regionsindex dort auf 0–4 erweitert).
Lokale lastbezogene Endpunkte `http://%s/api/v1/data`, `/v1/data`, `/v1/json` (IP aus `g_bleIpAddrBuf`)
sind **keine** Cloud-Pfade, sondern die Local-API-Abfrage externer Zähler/Wechselrichter im lokalen Netz.

MQTT-Themen: `marstek_energy/%s/device/%s/ctrl`, `marstek_energy/%s/App/%s/ctrl`. TLS via mbedTLS
(authmode 2 für MQTT, siehe §8).

### 4.5 BLE (Quectel FC41D, AT-Kommandos)

Gerätename `MST_VNSD_` + 4 Zeichen (`AT+QBLENAME=MST_VNSD_%c%c%c%c`). Befehlsverteiler
`BLE_App_Command_Dispatcher` @ 0x0800c758 — Befehlssatz identisch mit v149 (0x5A/0x5B aus v149 vorhanden),
laut Diff-Dokument nur Antwort-/Telemetrieaufbau geändert (~145 Op-Unterschiede zu v149, dispatcher-weit).
**BLE-OTA:** `BLE_Cmd_OTA_Init` (0x0803306c), `BLE_Cmd_OTA_WriteSetup` (0x08033100),
`BLE_Cmd_OTA_Validate` (0x08032f8c) vorhanden und laut Diff-Dokument byte-identisch zu v149/v150 —
BLE-OTA funktioniert. `Base64URL_Encode` (URL-sicheres Base64 für den AES-ECB-verschlüsselten
Telemetrie-Body) ist in v149.2 noch vorhanden (entfällt erst mit v150).

## 5. Leistungsgrenzen

- `Config_ClampDefaults_ParamBlock` @ 0x080093f0: Lade- **und** Entladeleistung werden auf **0x9C4 = 2500 W**
  begrenzt (ungültig/0 → Default; Entladen zusätzlich auf 800 W mit `g_workMode = 1`, falls ungültig).
  Ladegrenze `g_chargeCutoffSoc` 1–100 % (Default 100), Entladegrenze `g_dischargeCutoffSocRaw` 12–70 %
  (Default 12). Identisch mit v149 (keine Änderung).
- `Grid_Export_Power_Limiter` @ 0x08023438: Startwert `min(P_Netz, 2500 W)`, danach PV-Überschuss-Pfad,
  Gesamtklemme **4000 W**. Byte-identisch zu v149 und v150 laut Diff-Dokumenten — die 151-Änderung
  (Startwert ohne 2500-W-Klemme) betrifft v149.2 **nicht**.
- `Set.Ver` (MQTT-RPC 0x10, §4.2): 800 / 2200 (0x898) / 2500 W (0x9C4), identisch mit v149.
- Fernsteuer-Register (Modbus 45900/45901/0xA424/0x19B, §4.1): alle mit `< 0x9C5` (≤ 2500 W) geklemmt.
- **In v149.2 noch nicht vorhanden:** die separate, konfigurierbare Peak-Shaving-Klemme (EEPROM 0x394,
  BLE-Kommando 0x29, MQTT-Kommandos 0x3f/0x41) kommt erst mit v150 hinzu und ersetzt die 2500-W-Hartgrenze
  nicht, sondern wirkt zusätzlich (Diff v1492→v150).

## 6. Struktur-/Datenbefunde

- mbedTLS-Cipher-Tabellen (`aes_info`, `aes_128/192/256_ecb_info`, `gcm_aes_info`,
  `aes_128/192/256_gcm_info`, `mbedtls_cipher_definitions[7]`, `ecp_supported_curves[2]` — nur secp256r1)
  liegen unmittelbar hinter `g_mqttRpcMethodTable` @ 0x0805cb9c ff., wie in den übrigen Versionen.
- `MQTT_Transport_NullCallback` @ **0x0800fffe** (`bx lr`-Stub) ist der Wait-Ready-Hook des
  FC41D-Transports, keine tote Funktion (Adresse versionsspezifisch, in v147 0x0800FFFC).
- `CH395_ConfigTcpKeepalive_Defaults` @ 0x08036e34 (Funktion setzt TCP-Keepalive-Parameter).
- **BMS-Pack-Record (v149.2):** Basis `g_bmsPack0Voltage` @ **0x20014fa8**, Schrittweite **0x60 = 96 B**
  je Pack (8 B mehr als v149s 88 B — Feldpaar-Funktionen `CAN_AA_StoreUnitFieldPair0…88` belegen Offsets
  0, 8, 16, 24, 64, 72, 80, 88 innerhalb des 96-B-Records). CAN-Batteriezahl weiterhin auf 8 begrenzt,
  Modbus-Blöcke 34000…34600 (7 Packs). `BMS_CellVoltage_MinMax_PerPack` @ 0x08017860 läuft weiterhin nur
  über **`pack < 6`** (16 Zellen je Pack, `g_bmsPack0CellVoltageArray` @ 0x20014fc8) — der 7. Pack
  (Index 6) fehlt wie in allen Versionen 147–151 in `g_bmsCellStat*`/der MQTT-Telemetrie, ist aber per
  Modbus vorhanden. (Adresse dieser Funktion und der Records ist versionsspezifisch — hier für v149.2
  direkt aus dem Export bestätigt, **nicht** die in älteren Dokus für andere Versionen genannten Adressen.)
- **Pack-Nummerierung/Versatz im MQTT-Pfad:** wie in 147–151 schreibt `CAN_AA_StoreUnit*` nach
  Record `Einheit − 1`; Modbus 34000+100·n und `ShellCmd_get_all_bms_data` lesen konsistent Record n;
  `MQTT_Publish_BMS_Data` liest für Index p ≥ 1 dagegen Record p (Off-by-one) — Code in v149.2 vom
  gleichen Aufbau wie in den anderen Versionen (nicht separat neu diszassembliert, aus Konsistenz mit
  den Feldpaar-Funktionen und der bekannten Struktur übernommen; siehe „Bekannte Fakten" der Aufgabe).
- **FC03-Pufferüberlauf** (in der v151-Doku nur für v150/v151 verifiziert) ist **auch in v149.2 bereits
  vorhanden**: `FC03_Read_Handler` @ 0x080232a4 kopiert die Antwort nach dem Aufbau unbedingt
  (`for (uVar4 = 9; uVar4 <= uVar5; …) g_modbusLteTxFrame[uVar4] = (&g_modbusTcpTxFrame)[uVar4];`) in
  den Spiegelpuffer `g_modbusLteTxFrame`, unabhängig von der angeforderten Registeranzahl N ≤ 125 — der
  Fehler ist strukturell nicht neu in v150, sondern schon hier vorhanden (§11).
- Region-String-Tabelle enthält nur `eu`, `www`, `us` (3 Einträge) — kein `dev`/`staging` (erst v151).

## 7. Nur in v149.2 / geändert ggü. Vorgänger (v149) und Nachfolger (v150)

Kurzfassung (Belege und Instruktions-Ratios: `VNSD_Control_Diff_v149_v1492_de.md` /
`VNSD_Control_Diff_v1492_v150_de.md`):

- **BMS-Pack-Record 88 → 96 B** (8 B mehr je Pack) — einzige strukturelle Änderung mit Signalwirkung
  für Downstream-Code (`BMS_CellVoltage_MinMax_PerPack`, `MQTT_Publish_BMS_Version`, `get_all_bms_data`).
- **Französischer Eigenverbrauchsregler (SMR/TIC, Linky)** stark vereinfacht: 408 → 247 Instruktionen in
  `SMR_TIC_SelfUse_PowerController`, mehrfache `VectorSignedToFloat`/Skalierungsaufrufe entfallen,
  Setpoint-Berechnung nur noch einmal statt dreimal.
- **Cloud-/MQTT-Telemetrie umgebaut** (v149 → v149.2, dann bis v150 unverändert): `MQTT_Publish_BMS_Data`,
  `Cloud_UploadTelemetryData`, `DeviceInfo_BuildStatusString`, `Cloud_HTTP_Response_Parser`,
  `MQTT_StatusReport_BuildAndPublish`, `MQTT_Telemetry_String_Formatter`, `HTTP_Task_StateMachine_Handler`,
  `BLE_App_Command_Dispatcher` (nur Antwort-/Feldanpassungen), `MQTT_CtrlCommand_Dispatcher` — passend zum
  einzigen Changelog-Punkt „Fehlerhafte Datenanzeige durch ausbleibende HTTP-Uploads behoben".
- **Gegenüber dem Nachfolger v150 ist v149.2 nicht „praktisch fertig":** es folgen nochmals **56 geänderte
  Funktionen (26 inhaltlich), 20 neue und 1 entfallene** (Diff `VNSD_Control_Diff_v1492_v150_de.md`). Wichtigste
  Punkte: Cloud-Telemetrie-Upload komplett umgebaut auf einen neuen HTTPS/mbedTLS-Client, JSON-POST an
  `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` ersetzt die AES-ECB+Base64URL-GET-Anfrage an
  `hamedata.com` (behebt dabei einen realen Pufferüberlauf in `Base64URL_Encode`, das mit einer behaupteten
  Zielgröße von 2400 B auf den tatsächlich nur 1200 B großen `g_cloudUploadBodyBuf` schrieb); neue
  Peak-Shaving-/Leistungsklemme (EEPROM 0x394, BLE-Kommando 0x29, MQTT-Kommandos 0x3f/0x41, neue Funktion
  `MQTT_Publish_Grid_Status`); zwei neue Watchdogs bei hängenden Uploads (CH395-Chip-Reset,
  Modem-Hardware-Reset); `DeviceInfo_BuildStatusString` bekommt feste Hardware-Tags für 6 weitere
  Zählertypen; `Grid_PowerSync_StateMachine` erweitert den Zählertyp-Bereich für die Direktübernahme;
  BMS-Selbsttest-/MOS-Flag zusätzlich gemeldet; neuer Shell-Befehl `ShellCmd_FreeRtos_mem`.
  **Korrektur gegenüber einer früheren Analysefassung:** die zuvor als einzige drei Unterschiede genannten
  „HTTP-Upload-Leseoffset +5→+7", „CH395-Socket-Parameter 0x24→0xAA" und „HTTP-GET-Puffer 0x168→0x280 B"
  waren reine `__LINE__`-Log-Argumente ohne Verhaltensänderung; die tatsächliche Änderung an
  `HTTP_ParseUploadResponseCode` ist ein anderer JSON-Schlüssel (`"code":` statt `"de":`).
- **Unverändert bleiben ggü. v150:** Modbus-Registerinhalt, die 2500-W-Hartgrenze, OTA/BLE-OTA und die
  `hamedata.com`-Endpunkte selbst. **Nicht unverändert:** der BLE-Befehlssatz wächst um Kommando 0x29
  (Peak Shaving, s. o.) — die in §0a genannte „nummerisch gleich" gilt nur für den Schritt v149→v149.2,
  nicht für v149.2→v150.
- Sämtliche Unterschiede zum übernächsten Vorgänger v147 (2200→2500 W, 11→16 RPC-Methoden, 15 neue
  Modbus-Register, Shell +2 Befehle, TIC-Unterstützung, DOD-Bereichsprüfung usw.) sind bereits in v149
  enthalten und bleiben in v149.2 bestehen — Details: `VNSD_Control_Diff_v147_v149_de.md`.

**Beobachtung zur v151-Doku (jetzt aufgelöst):** Die v151-Doku hatte „„Get err code info" antwortet jetzt
(Cmd 0x13, 280 B)" als BLE-Änderung von v150 nach v151 aufgeführt. In v149.2 antwortet
`BLE_App_Command_Dispatcher`, Fall `0x13`, bereits unverändert mit `BLE_Send_Response('\x13', &g_displayErrorLog, 0x118)`
(280 B) — Quelltext identisch zu dem in `Dekompilate/VNSD_Control_v151/0800c730_BLE_App_Command_Dispatcher.c`.
Der überarbeitete Diff `VNSD_Control_Diff_v150_v151_de.md` bestätigt das inzwischen ausdrücklich: „BLE „Get err
code info" (0x13): Antwort in v150 und v151 identisch (kein Unterschied)". Die frühere Einordnung als
neu-in-v151 war demnach falsch (nicht mehr offen).

## 8. Sicherheitsbefunde (identisch mit 147–151, soweit nicht anders angegeben)

1. **Unauthentifizierte Fernkonsole:** Modbus-TCP-Schreibzugriff (Port 502, ohne Auth) auf Register
   46000 = 0x5100 → `g_debugMode = 1` dauerhaft im EEPROM (0x900) → letter-shell auf RS485 **und**
   TCP 8091 (`CH395_Init_TCPServer_Socket`, Socket 1 @ 0x2001adcc). Erlaubt u. a. Werksreset, WLAN-Zugangsdaten
   setzen, Firmware-Update starten, Neustart.
2. **Fest eingebaute WLAN-Zugangsdaten** `SM471` / `Hame147258` (Strings @ 0x0802d578 / 0x0802d580,
   `Config_LoadOrInit_DeviceSettings` @ 0x0802d174, Werksdefault bei leerem EEPROM 0x400).
3. **Cloud-Upload über unverschlüsseltes HTTP** (`http://%s.hamedata.com/prod/api/v1/setVenusDReporting…`
   u. a.); Nutzdaten AES-ECB + Base64URL (`Base64URL_Encode`, entfällt erst ab v150). Policy-Abruf
   (`schedulings/policies`) läuft über HTTPS, alle anderen `hamedata.com`-Pfade über Klartext-HTTP.
4. MQTT-TLS mit `authmode 2` (Zertifikatsprüfung aktiv) — nicht separat neu geprüft für v149.2, aber
   keine Änderung an der TLS-Konfiguration in den Diff-Dokumenten gefunden.
5. Zwei chinesische Klartext-Debug-`printf`s im `Write_Handler`-Testzweig für Register 47400 (§4.1, §11) —
   kein Sicherheitsrisiko im engeren Sinn, aber Debug-Restcode im Produktivbuild.

## 9. Offene Punkte

- MQTT-TLS-Zertifikatsprüfung (`authmode`), Ablaufdatumsprüfung und die verschlüsselten
  TLS-Zertifikate/Schlüssel wurden für v149.2 nicht eigenständig disassembliert, sondern nur als in den
  Diff-Dokumenten nicht aufgeführte („unverändert") Bereiche übernommen — keine Instruktions-Verifikation.
- `MQTT_Publish_BMS_Data`-Off-by-one und `g_pendingBootDescriptor`-Überlauf für Pack-Index 8: für v149.2
  nicht durch eigenen Blick in die Dekompilation der Funktion bestätigt, sondern aus der versionsübergreifend
  bekannten Struktur (identische Feldpaar-Funktionen, identischer BMS-Codepfad) übernommen.
- Keine belastbare Aussage zum genauen Umfang „Fehlerhafte Datenanzeige durch ausbleibende HTTP-Uploads"
  (Changelog) — die Diff-Tabelle nennt viele umgebaute Telemetrie-/HTTP-Funktionen, aber keinen einzelnen
  eindeutigen Bugfix-Commit-Charakter wie bei anderen Changelog-Punkten.

## 10. Korrekturen an den bereits „fertigen" Ständen (bei der v147-Arbeit gefunden, betreffen auch v149.2)

- v149.2: **3 Codestellen** waren fälschlich als Daten typisiert (v149: 33), **2 Shell-Handler** als
  Bytefelder (v149: 3); ein Task-Trampolin-Problem wie bei v149 (Task-Namen verschoben) wurde ebenfalls
  behoben.
- Alle Versionen (147–151), gilt daher auch für v149.2:
  - letter-shell `CLI_Backspace`/`CLI_DeleteCharForward` waren vertauscht benannt, `CLI_DeleteChar`-Richtung
    falsch beschrieben.
  - mbedTLS-Cipher-Bereich war fälschlich als `mbedtls_eckey_info` markiert.
  - Task-Tabelle war in 149/149.2 als falsches `ciphersuite_preference` benannt (jetzt `g_taskDefTable`).
  - `CH395_ConfigUDP_Defaults` heißt jetzt `CH395_ConfigTcpKeepalive_Defaults` (setzt TCP-Keepalive,
    hier @ 0x08036e34).
  - Der `bx lr`-Stub heißt jetzt `MQTT_Transport_NullCallback` (hier @ 0x0800fffe, versionsspezifische
    Adresse).
- Nacharbeit 2026-10-12 (wie bei D 151 / A 1509): `Inverter_RegDirty_Mark_DischgVolt` → `Inverter_RegDirty_Mark_GridPermit` (übergibt 1, solange das Micro-Telemetriewort `g_invState+0x16` 0 ist) und `Inverter_RegDirty_Mark_BatteryParams` → `Inverter_RegDirty_Mark_PvVoltage` (übergibt 1, solange alle vier PV-String-Spannungen unter 0x8D = 14,1 V liegen); das Code-Muster wurde in diesem Image geprüft. Das „Inverter-Dirty-Bitmap" ist das Frontpanel-LED-Bitmap eines TM1650-artigen Treibers (Bit-Bang-I2C auf die Digit-Register 0x68/0x6A/0x6C/0x6E); Plate-Kommentare der LED-Wrapper und Globals aktualisiert. Gates erneut gelaufen: alle bestanden; Dekompilat- und Listing-Export aktualisiert.

## 11. Chinesische Strings

115 eindeutige UTF-8-Strings mit chinesischem Text im Export, alle mit `ZH->EN`-Kommentar übersetzt
(überwiegend Shell-Kommandobeschreibungen und Debug-Meldungen). Auswahl mit deutscher Übersetzung:

| Chinesisch (gekürzt) | Kontext | Deutsch |
|---|---|---|
| 测试命令 | Shell `test` | Testbefehl |
| 外设启动查询 | Shell `check_periph` | Peripherie-Startprüfung |
| 外部sram测试 | Shell `xfmc_test` | Externer-SRAM-Test |
| 获取所有版本信息 | Shell `get_ver` | Alle Versionsinformationen abrufen |
| 设置老化版本 | Shell `set_old` | Burn-in-(Einbrenn-)Version setzen |
| 修改soc | Shell `change_soc` | SOC ändern |
| 重启系统 | Shell `reset` | System neu starten |
| flash分页写入测试 | Shell `flash_test` | Flash-Seitenschreibtest |
| RTOS状态 | Shell `rtos_status` | RTOS-Status |
| 唤醒被挂起的所有任务 | Shell `Resume_task` | Alle pausierten Tasks aufwecken |
| 开关逆变器调试功能 | Shell `debug_mode` | Wechselrichter-Debug-Funktion ein/aus |
| 休眠设备 0:正常工作 1:休眠 | Shell `venus_poweroff` | Gerät schlafen legen (0: Normalbetrieb, 1: Schlaf) |
| 删除SSL证书 | Shell `delate_ssl_cert` | SSL-Zertifikat löschen |
| 设置WIFI信息 | Shell `set_wifi` | WLAN-Zugangsdaten setzen |
| 查询FC41D模组固件版本 | Shell `ver_fc` | FC41D-Modul-Firmwareversion abfragen |
| 进入modbus | Shell `modbus` | In Modbus-Modus wechseln |
| 显示modbus地址 | Shell `show_modbus` | Modbus-Adresse anzeigen |
| Local API 设置：0 关闭 1 开启 其他 查询 | Shell `api_set` | Local API setzen: 0 aus, 1 an, sonst abfragen |
| 控制led灯亮度 0 最亮 7-1依次减弱 | Shell `ctrl_led_light` | LED-Helligkeit steuern (0 am hellsten, 7→1 schwächer) |
| 余电上网功能: 0关闭 1开启 | Shell `full_dischrg` | Überschusseinspeisung: 0 aus, 1 an |

**Sonderfall:** Zwei chinesische Zeichenketten liegen nicht nur als Kommentar, sondern **direkt im
kompilierten `printf`-Aufruf** des Modbus-`Write_Handler`-Testzweigs für Register 47400 (§4.1):
„47400要写进去里面的数据 = 0x%x" (dt.: „In 47400 zu schreibende Daten = 0x%x") und „没到里面吗123456"
(dt.: „Kommt er nicht rein? 123456" — offensichtlicher Debug-Restcode, keine Fehlermeldung für Nutzer).
