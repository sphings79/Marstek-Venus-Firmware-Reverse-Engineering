# Control FW v151 — Vollanalyse und Diff gegen v150

> 🇬🇧 English version: [VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md](VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md)

**Status:** abgeschlossen (Stand 2026-09-30). Alle Aussagen sind gegen den Code geprüft; Ausnahmen sind markiert.
**Binary:** `151_control_VNSD-0_ems_app_0151_0929_094333.bin` (Archiv `Marstek FW Archiv/firmwares/VNSD-0/Control/151/`)
**Referenz:** `150_control_VNSD-0_app_0150_0805_115146.bin` (v150, neu importiert mit korrekter Basis)
**Ghidra:** `/VNSD/Control/Control_151_VNSD-0_ems_app_0151_0929_094333.bin` und `/VNSD/Control/Control_150_VNSD-0_app_0150_0805_115146.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** `Dekompilate/VNSD_Control_v151/` (lokal, nicht im Repo) (1956 Funktionen) + `listing/` (Flash-Listing, RAM-Globals, Peripherie-Register)
**Englische Fassung:** `VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md`
**Diff-Tabelle:** `VNSD_Control_Diff_v150_v151_de.md` (DE) / `VNSD_Control_Diff_v150_v151_de.md` (EN)

---

## 0. Wichtigster Befund vorab: Die Image-Basis ist 0x08004800

Alle Control-Firmwares des Venus D (147, 149, 149.2, 150, 151) sind **für 0x08004800 gelinkt**, nicht für
0x08000000. Davor liegt ein 18-KB-Bootloader (`0x08000000`–`0x080047FF`).

| Beleg | Befund |
|---|---|
| `SystemInit` (0x080333E0) | schreibt `SCB->VTOR = 0x08004800` |
| Vektortabelle bei 0x08004800 | Reset = Keil-Muster `ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`; Fault-Handler `nop; b .`; SVC/PendSV = FreeRTOS-Port; Default = `b .` |
| Zeiger-Literale (4-Byte-ausgerichtet) | bei 0x08004800: 405 Treffer auf String-Anfänge, 178 auf Funktionsprologe; bei 0x08000000: 70 bzw. 9. Gleiches Verhältnis bei 147/149/149.2/150 |

**Folge für alle älteren Analysen (v147–v150):** Jede Flash-Adresse in den älteren Dokus ist um 0x4800 zu niedrig
(echte Adresse = zitierte Adresse + 0x4800), und Funktionsnamen, die aus falsch aufgelösten Strings abgeleitet
wurden, waren teils falsch. v150 wurde deshalb neu importiert und vollständig neu aufbereitet (eigene Doku).

## 0a. Prüfliste für die Update-Entscheidung (v150 → v151)

| # | Frage | Antwort v151 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register geändert? | **Registersatz identisch** (246 Einträge, 30000–38014). Einzige Änderung im Lesebereich: **32102** `float32` → `i16`. Folge: 32103 einzeln → Exception 02 statt UsageFault; Blockabfragen über 32103 werden abgelehnt. Schreibbereich: **neues Holding-Register 0xB799 (47001)** und ein neuer CAN-Aufruf (`CAN_Write_Category0x18`) im KI-Modus-Pfad. 32101-Vorzeichenfehler, MBAP-Längenfehler und 44002/44003-Schattenwert bestehen weiter (§9). | §3, §9 |
| 2 | BLE geändert? | Gerätename (`MST_VNSD_` + 4 Zeichen) und Befehlssatz unverändert. Detailänderungen: EPS ein/aus (0x0F) und Entladegrenze (0x54) setzen die EPS-Reserve-Hystereseflags nicht mehr zurück; **LED-Status wird im EEPROM (0x395) gespeichert**. Unverändert fehlerhaft: Cmd 0x27 (Leistungsverlauf) antwortet bei Anforderung von ≥ 15 Werten gar nicht (§11). | §8 |
| 3 | AC-Einspeisegrenze 2500 W geändert? | **Die konfigurierbaren 2500-W-Grenzen nicht** (Modbus-Schreiben, BLE, MQTT, Konfigurationsprüfung, Fernsteuerbefehl: alle unverändert). **Geändert hat sich die PV-Obergrenze beim Einspeisen** in `Grid_Export_Power_Limiter`: v150 startete mit `min(P_bms, 2500 W)`, v151 mit `Netz-Messwert − P_neg` ohne 2500-W-Klemme; die Gesamtklemme 4000 W und der Überschuss-Pfad sind unverändert. | §8 |
| 4 | Firmware-Update per BLE weiter möglich? | **Ja.** BLE-OTA-Fälle, Flash/OTA/CAN-Update/QSPI-Funktionen und die Update-Prüfung (CRC, Gerätemaske, Projekttyp) sind funktionsgleich zu v150. Neu ist eine **Typprüfung im YMODEM-Weg** (Shell `update_*` über RS485/TCP 8091): Dateiname muss zum Ziel passen („Bin type mismatch! Upgrade aborted.“). Die App beschreibt vom Bootloader-Bereich nur die **Update-Deskriptor-Seite 0x08004000** (40 Byte, „VNSD-0“-Magic, CRC16) und liest das Bootloader-Versionswort bei 0x08001000; Bootloader-Code wird nicht angefasst. | §8 |
| 5 | Cloud-Domains geändert? | **Ja, ein Endpunkt:** `getDateInfo` zieht von `http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php` nach **`http://api-<region>.marstekcloud.com/app/neng/getDateInfo<suffix>.php`** (Klartext-HTTP, Port 80). Upload unverändert `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`. Weiterhin hamedata: `backup-configs` (HTTP), `schedulings/policies` (HTTPS), `getDevicesApply` (HTTP, WLAN-Pfad), FC41D-OTA `http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl`. MQTT-Host kommt aus der Konfiguration (Flash-Block 0x317000). Regionstabelle: Index 3/4 heißen jetzt `dev`/`staging` statt `eu-dev`/`eu-staging`. Geräte-Quell-ID im Cloud-Payload: `VenusD-<SN>` → `VNSD-0-<SN>`. | §7, §8 |

Region = Index aus EEPROM 0x441 in Tabelle `eu, www, us, dev, staging` (ungültig → `eu`); beim Upload wird `www` auf `us` umgelenkt.

## 1. Binary-Fingerprint

| | v151 |
|---|---|
| Größe | 393 216 B (0x60000) |
| SHA-256 | `db3638cb68683175debe283375f36b02427b9e5e7a647ce2045e8135177fea47` |
| Link-Adresse | `0x08004800`–`0x080647FF` |
| MCU | Nations **N32G45x** mit XFMC (Cortex-M4F), früher fälschlich als Artery AT32F403A/407 eingeordnet (s. §1a); SRAM **144 KB**; `SystemCoreClock` = 144 MHz (Init-Wert) |
| Toolchain | **Keil ARMCC** (`__main`/`__scatterload`, LZ77-komprimiertes RW) |
| Bibliotheken | FreeRTOS (heap_4, 80 KB), mbedTLS 2.28.10 (nur TLS-ECDHE-RSA-AES128-GCM-SHA256 / secp256r1), cJSON (hersteller-modifiziert), llhttp, letter-shell, N32-Standard-Peripheriebibliothek (n32_stdperiph, früher als „AT32-Firmware-Library" geführt), WCH CH395-Library |
| Initial-SP | `0x20022010` (= Ende von .bss) |
| Versions-String | `VNSD-0 v151` (0x0805C9B9); Versionskonstante 0x97 bei 0x08010000 |
| Image-Footer | `VNSD` + 12 Nullbytes bei 0x080647E0, davor 0xFF-Füllung |

## 1a. MCU-Identifikation: Nations N32G45x, nicht Artery AT32F403A

- Der Control/EMS-MCU ist ein **Nations N32G45x mit XFMC** (External Memory Controller), nicht der zuvor angenommene Artery AT32F403A.
- Belege: QSPI-Controller bei 0xA0001000 mit N32-QSPI_STS-Bits; RCC-PLL-Multiplikator in Bit 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG genutzt; SAC-Krypto-Coprozessor bei 0x40024000 (früher fälschlich „RNGX" genannt); 96-Bit-UID @ 0x1FFFF7F0; 102 Vektoren.
- Externes SRAM 0x60000000–0x60026AF7 läuft über XFMC, Steuerregister bei 0xA0000000: der offizielle Nations-Header `n32g45x.h` (lokal, nicht im Repo) definiert `XFMC_REG_BASE 0xA0000000` und `QSPI_BASE = XFMC_REG_BASE + 0x1000`; die Nations-Treiberbibliothek enthält `n32g45x_xfmc.c`; die Firmware selbst hat den Shell-Befehl `xfmc_test` (das aktuelle englische N32G45x-Benutzerhandbuch markiert diesen Adressraum fälschlich als „Reserved").
- Details/Header-Kopie: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Speicherkarte (in Ghidra angelegt)

| Block | Bereich | Inhalt |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | nicht Teil des Images; 0x08001000 = Bootloader-Versionswort, 0x08004000 = Update-Deskriptor-Seite |
| flash | 0x08004800–0x080647FF | Applikation |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | Geräte-Signatur (UID) |
| .data | 0x20000000–0x20002EFF | RW-Startwerte, LZ77 entpackt |
| .bss | 0x20002F00–0x2002200F | nullinitialisiert; `ucHeap` 0x20003144 (0x14000 B) |
| .xmc_bss | 0x60000000–0x60026AF7 | externes SRAM über XFMC (Telemetrie-Ringpuffer, HTTP/MQTT/TLS-Puffer, Ereignislogs); Scatter-Load-Sektionsname `.xmc_bss` unverändert übernommen |
| apb1/apb2/ahb/ppb | Peripherie | Register nach offiziellem Nations-SVD (N32G457, 942 Einträge) benannt, dazu 16 XFMC-Register aus `n32g45x.h` und 5 SAC-Register (nicht in der SVD) + Cortex-M4-Kernregister; ersetzt die frühere, falsche Zuordnung zur Artery-SVD `AT32F403Axx_v2.svd` |
| extflash_ctrl | 0xA0001000–0xA00011FF | **QSPI-Controller** (`QSPI_BASE` = `XFMC_REG_BASE` + 0x1000), früher als nicht dokumentierter Registerblock geführt (s. §1a, §13) |

Scatter-Load-Tabelle `Region$$Table$$Base` @ 0x08062C94: .data (0x080631D4 → 0x20000000, 0x2F00, `__decompress`),
XMC (0x08063B8C → 0x60000000, 0x26AF8), .bss (Zero-Init 0x1F110). Entpacker `__decompress` @ 0x08005FBE:
`op=*src++; n=op&7 (0→*src++); L=op>>4 (0→*src++); n−1 Literale; op&8 ? Rückverweis d=*src++, L+2 Bytes : L Nullbytes`.

## 3. Modbus

### 3.1 Lesebereich (Deskriptortabelle)

`g_modbusRegDescTable` @ **0x2000034C** (v150: 0x20000354), Typ `ModbusRegDesc_t[246]`
(`reg_addr`, `pad`, `source`, `type_code`, `type_code2`, `scale`, `count`), Register 30000–38014.
Die Variablen hinter den Registern heißen in Ghidra `g_mbReg<Nr>`, soweit kein fachlicher Name vergeben ist.

- Keine neuen oder entfallenen Register; einzige Semantikänderung **32102** `float32` → `i16`.
- `FC03_Read_Handler` und `Read_Serializer` sind instruktionsgleich zu v150.

| | v150 | v151 |
|---|---|---|
| 32103 einzeln lesen | UsageFault (VLDR unausgerichtet), Session stirbt | Exception 02 |
| Block über 32103 (Workaround 32102–32104) | funktioniert | **abgelehnt (Exception 02)** |
| 32101 negativ (Vorzeichenfehler Read-Serializer) | falsch | **weiter falsch** |

### 3.2 Schreibbereich (`Write_Handler` @ 0x080570E0)

- Neues Register **0xB799 (47001)** und neuer Aufruf `CAN_Write_Category0x18()` im KI-/Netzfilter-Pfad (Changelog 4).
- Der interne Basiswert des Blocks ab 45000 verschiebt sich um 1 (v150 `reg−45000`, v151 `reg−45001`), weil
  0xAFE6 als eigener Fall eingefügt wurde; die Registernummern nach außen bleiben gleich.
- **Register 46000** (unverändert seit v150): Wert **0x5100** schaltet den Debug-Modus dauerhaft ein
  (EEPROM 0x900) → Debug-Shell auf RS485 **und TCP 8091** (§10.1). Werte 0x04D2/0x0929/0x0D80 wählen das YMODEM-Update-Ziel.

### 3.3 Modbus TCP (CH395)

Socket-Tabelle `g_ch395Sock…` @ 0x2001D604 (8 × 36 B): Socket 0 = **Modbus TCP 502**, Socket 1 = **TCP-Server 8091
(Debug-Shell)**, Socket 2 = UDP-Broadcast 8090, Socket 4 = UDP für Zähler/Heartbeat. Keepalive (WCH-Funktionen
`CH395KeepLiveCNT/IDLE/INTVL`): CNT 5, **IDLE 10 000 → 60 000 ms**, INTVL 3000 ms; in v151 wird er zusätzlich
**vor dem Öffnen des Modbus-Sockets** gesetzt (Changelog 1).

## 4. Debug-Shell (letter-shell)

Befehlstabelle `shellCommandTable` @ 0x08062CC4 (81 × 16 B): **70 Befehle** (+ `help`, `clear`), 1 Benutzer
(`VNSD-0 v151`, **leeres Passwort**), 10 Tastenbelegungen. Neu gegenüber v150: `read_inv1`, `read_inv2`.
Alle Befehlsfunktionen heißen `ShellCmd_<befehl>`.

**Erreichbarkeit:** `Task_rs485` bedient abhängig von `g_debugMode` (EEPROM 0x900):
0 = Modbus RTU auf RS485; 1 = **Shell auf RS485 und über TCP-Port 8091**; 2 = Warten auf die Zeichenfolge
`quit_debug_5100_` (RS485 oder TCP 8091), danach Modus 1. Modus 1 setzt u. a. Modbus-Register 46000 = 0x5100 (§10.1).
Der Shell-Befehl `modbus` schaltet zurück auf 0.

Befehlsliste (übersetzt): set_level, debug_fc, test, check_periph, xfmc_test, get_ver, set_old/get_old, change_soc,
reset/reset_reason, flash_test/eeprom_flash_test, ch395_get_ip/ch395q_test/ch395_ver/recon_ch395/hard_reset_395,
rtc_show/get_time, rtos_status/FreeRtos_mem/Resume_task, update_ems/vns/bms/mppt, ctl_mppt_power, restore_factory
(1 alles, 2 Energie behalten, 3 Zertifikate), venus_rfd, debug_mode, venus_poweroff, get_mppt_data/get_venus_data/
get_main_bms_data/get_all_bms_data, delate_ssl_cert/check_ssl_cert, set_wifi/wifi_info/mac_ble/ver_fc/fc41d_reset,
trace_debug, modbus/show_modbus, get_log, api_set/api_port, vns_ate_test, ctrl_led_light/led_continue_ctrl
(ruft tatsächlich `Inverter_Power_Setpoint_Apply`), energy_show/err_code/ext_info, button_ctrl/key_state,
read_inv1/read_inv2, code_test, lck_bms, ems_rtrl_info, full_dischrg, free_mqtts, get_meter_mac, get_backup_datas,
power_check, get_ota_log.

## 5. FreeRTOS-Tasks

Tabelle `g_taskTable` @ 0x0805B358: **14 Anwendungs-Tasks** (in früheren Fassungen wurden 13 gezählt;
`Task_fc41d` @ 0x08033540 wurde übersehen).

| Task | Prio | Stack (Wörter) |
|---|---|---|
| Task_ch395 | 15 | 256 |
| Task_ch_mqtts | 14 | 1024 |
| Task_parser | 13 | 512 |
| Task_ch_http | 13 | 1024 |
| Task_http | 11 | 512 |
| Task_udp | 10 | 256 |
| Task_ems | 9 | 256 |
| Task_ct | 8 | 512 |
| Task_Modbus_tcp | 7 | 512 |
| Task_can_comm | 7 | 256 |
| Task_can_receive | 6 | 256 |
| Task_rs485 | 3 | 512 |
| Task_iwdg | 1 | 128 |
| Task_fc41d | (Eintrag 0) | – |

Prioritäten und Stacks sind v150 → v151 unverändert. `Task_Modbus_tcp` (Prio 7) läuft weiter unter den TLS-Tasks (13/14).

## 6. Chinesische Strings

121 UTF-8-Strings mit chinesischem Text, alle übersetzt (Ghidra: EOL-Kommentar `ZH->EN`). Zwei chinesische
Debug-`printf` stehen noch im Modbus-`Write_Handler` (Register 47400, Testzweig).

## 7. Changelog v151 → Code

| # | Changelog (Hersteller) | Umsetzung im Code |
|---|---|---|
| 1 | Modbus-TCP-Stabilität | CH395-Keepalive IDLE 10 → 60 s, vor dem Öffnen des Modbus-Sockets gesetzt (`CH395_ConfigTcpKeepalive_Defaults` aus `CH395_Init_Modbus_TCP_Socket`); CH395-Hänger-Watchdog in `HTTP_ClientTask_Dispatcher`: Rückstau `>1`→`≥5`, 1800 → 14 400 Ticks, weicher Chip-Reset → **kompletter MCU-Neustart**, neues Unterdrückungs-Flag |
| 2 | DOD-Fehlzustand, keine Entladung möglich | `CT_GridPower_Controller`: hängender positiver CT-Sollwert wird bei neuem Flag (0x200002B2) auf 0 gesetzt; `Battery_EPS_Reserve_SOC_Hysteresis` läuft nur noch bei SoC ≠ 0 und setzt das EEPROM-EPS-Flag nicht mehr selbst zurück; neues Statusfeld `df=` im MQTT-Statusbericht. *Welcher dieser Punkte den Feldfehler behebt, ist aus dem Code allein nicht entscheidbar.* |
| 3 | YMODEM-Upgrade-Typprüfung | `YModem_ReceiveFile` klassifiziert den Dateinamen (`OTA_ClassifyDeviceTypeTag`), `YModem_FirmwareUpgrade_Handler` bricht bei −5 mit „Bin type mismatch! Upgrade aborted.“ ab |
| 4 | KI-Modus-Prüfung + aktuelle KI-Strategie abfragen | `AI_ScheduleQuery_Response_Send` sendet seitenweise statt eines unbegrenzten `snprintf`; `Write_Handler`: neues Register 0xB799 + `CAN_Write_Category0x18` |
| 5 | Pre-Release-Server-URL-Fix | `getDateInfo` → `api-<region>.marstekcloud.com`; Regionsindex 0–4 (vorher 0–2) gültig; Servertyp EEPROM 0x441 wird beim Start geladen; Tabelle `dev`/`staging` |
| 6 | HTTPS-Upload-Stabilität | `Quectel_HTTP_POST_SendAndVerify`: echte Tick-Schleife statt blinder `vTaskDelay`, ERROR/`+QIURC`-Behandlung, exakte Byteanzahl, `AT+QHTTPPOST`-Timeout 60 → 120 s, äußerer Timeout 600 → 6000; Heap +10 KB |
| 7 | CT-Kommunikation 1,1 s | `Ethernet_/Modem_SendStatusTelemetry`: Zählerabfrage (Shelly `EM.GetStatus`/`EM1.GetStatus` usw.) höchstens alle 0x44C = 1100 Ticks (gemeinsamer Zeitstempel 0x20001F58); `Meter_ParseUDPMeterInfo` ohne blockierendes `vTaskDelay` |

## 8. Diff v150 → v151 (Kurzfassung)

Vollständige Tabelle mit alten/neuen Werten und Fundstellen: `VNSD_Control_Diff_v150_v151_de.md`. Nicht im Changelog:

- PV-Obergrenze beim Einspeisen ohne 2500-W-Klemme am Startwert (`Grid_Export_Power_Limiter`)
- Netz-Sync (`Grid_PowerSync_StateMachine`) übernimmt einen Moduswechsel jetzt auch bei 3 abweichenden Phasen (vorher nur bei genau 2)
- FreeRTOS-Heap 0x11800 → 0x14000
- MQTT `cd=0x42` „Get INV DEBUG info" (neuer Case): `debug=1` schaltet Wartungsmodus an (`g_maintModeState=2`),
  `debug=2` publiziert `g_vns_inv_debug` per `MQTT_Publish_WorkMode_Config`, `debug=3/4` schalten CAN-Kategorie 0x18
  über `CAN_Write_Category0x18` und beenden den Wartungsmodus wieder
- MQTT `cd=0x13`: neuer Unterbefehl `debug` → `MQTT_ConnectionStatus_Report_Send`
- Protocol-AA-Systemparameter: 7-Slot-Tabelle mit Grenzprüfung statt fester Werte
- Geräte-Quell-ID `VenusD-<SN>` → `VNSD-0-<SN>`
- neue Shell-Befehle `read_inv1`, `read_inv2`

Ausdrücklich **keine** Änderung (frühere Fassung dieses Dokuments hatte das fälschlich behauptet): `MQTT_Config_ExtractFields`
hat bereits in v150 6 Einträge inklusive `to=`; `Task_udp`s Aufrufreihenfolge ist in v150 und v151 byte-identisch
(Details: `VNSD_Control_Diff_v150_v151_de.md`, Abschnitt „Unverändert").

## 9. Bekannte v150-Fehler in v151

| # | Fehler | Status v151 |
|---|---|---|
| 1 | 30-min-Reset der Ethernet-Bridge (CH395-Watchdog) | **geändert**: ≥ 5 Datensätze Rückstau, 14 400 s, danach MCU-Neustart statt Chip-Reset, abschaltbar per Flag |
| 2 | WLAN-Zwilling 900 s (FC41D-Hardware-Reset) | unverändert |
| 3 | Empfangsschleife verlangt CR LF am Ende | unverändert |
| 4 | 86-s-Drossel ab Rückstau > 3 | unverändert |
| 5 | ~4 s Modbus-Stillstand pro Upload (TLS-Handshake) | unverändert (kein Session-Cache, Prioritäten gleich) |
| 6 | HTTPS-Upload ohne Zertifikatsprüfung (authmode 0) | unverändert |
| 7 | 32103 einzeln → UsageFault | behoben (über Tabellenänderung 32102) |
| 8 | 32101 Vorzeichenfehler | unverändert |
| 9 | MBAP-Länge 4 statt 3 bei Exception-Antworten | unverändert |
| 10 | 44002/44003 lesen nach Neustart 0 (Schattenwert) | unverändert |

## 10. Sicherheitsbefunde (v150 und v151 gleichermaßen, sofern nicht anders angegeben)

### 10.1 Unauthentifizierte Fernkonsole über Modbus-Register 46000 + TCP 8091

Ein Modbus-TCP-Schreibzugriff (Port 502, ohne Authentifizierung) auf **Register 46000 mit Wert 0x5100**
setzt `g_debugMode = 1` und speichert ihn im EEPROM (0x900). Ab dann bedient `Task_rs485` die letter-shell auf
RS485 **und auf TCP-Port 8091** (CH395-Socket 1) — Benutzer `VNSD-0 v151` mit leerem Passwort. Die Shell erlaubt u. a.
Werksreset, WLAN-Zugangsdaten setzen, Update-Start, Neustart und `test` (schaltet den Wechselrichter ein und setzt
einen CAN-Leistungssollwert). RS485-Modbus-RTU ist in diesem Modus aus. Der Zustand überlebt Neustarts.

### 10.2 Weitere

| Befund | Fundstelle |
|---|---|
| HTTPS-Upload-Pfad ohne Server-Zertifikatsprüfung (`mbedtls_ssl_conf_authmode(…,0)`); MQTT prüft (`2`) | `HTTPS_TLSSessionInit` 0x0803BF70, `mbedTLS_SSL_Connection_Init` 0x0801D2C4 |
| Zertifikats-Gültigkeitszeitraum wird nie geprüft: `mbedtls_x509_time_is_past/_is_future` sind `return 0`-Stubs (`MBEDTLS_HAVE_TIME_DATE` aus) — gilt auch für MQTT | 0x0804E394 / 0x0804E39A |
| Werkseitige WLAN-Zugangsdaten SSID `SM471` / Passwort `Hame147258` (Werkseinstellung, Fallback bei leerem EEPROM, Shell `set_wifi`) | `Config_LoadOrInit_DeviceSettings` 0x0802E17C, `Quectel_WiFi_AutoConnect_StateMachine` 0x0801681C |
| Eingebettete TLS-Zertifikate/Schlüssel mit fest einkodiertem, per Byte-Rotation abgeleitetem AES-128-Schlüssel verschlüsselt | `g_certAesKeyDeriveSeed` 0x20000FF0, `TLS_Cert_Decrypt_All` |
| MQTT-Client-/Geräte-ID per AES-128-ECB mit fest einkodierten, verschleierten Schlüsseln; Klartext-ID geht ins Debug-Log | `g_mqttDeviceIdKeySeed` 0x20001E9C u. a., 0x080134D4 |
| Zauberwort `quit_debug_5100_` schaltet im Modus 2 die Shell frei | `RS485_DebugConsole_ScanQuitPhrase` 0x08055CA8 |
| `getDateInfo` und `backup-configs` im Klartext-HTTP | §0a Nr. 5 |

## 11. Weitere Code-Fehler (neu gefunden, verifiziert)

| Fehler | Fundstelle |
|---|---|
| **Pufferüberlauf FC03 (v150 + v151):** Antwort (9 + 2·N Byte, N ≤ 125) wird in den 128-Byte-TX-Rahmen geschrieben und danach **immer** in einen Spiegelpuffer kopiert, der nach 32 Byte in den MQTT-RPC-Arbeitsbereich (0x6002683C…) übergeht. Ab N > 11 werden RPC-Felder überschrieben, ab N > 59 läuft der TX-Rahmen in den RX-Rahmen. Die Antwortbytes selbst bleiben korrekt. | `FC03_Read_Handler` 0x08023A00, `g_modbusTcpTxFrame` 0x6002671C |
| BLE Cmd 0x27 (Leistungsverlauf): Anzahl > 15 wird auf 15 begrenzt, beantwortet werden aber nur Anzahlen < 15 → bei ≥ 15 keine Antwort | `BLE_App_Command_Dispatcher` |
| BLE Cmd 0x23 (Generator ein/aus) antwortet mit Befehls-ID 0x0F statt 0x23 | `BLE_App_Command_Dispatcher` (`BLE_Send_Response('\x0f', &g_generatorEnable, 1)`) |
| `HTTP_Economy_TOU_ParseResponse`: `crc` und `crc2` werden mit derselben XOR-Schleife berechnet und ODER-verknüpft geprüft — die zweite Prüfsumme ist wirkungslos (die XOR-Prüfung selbst funktioniert) | 0x08019578 |
| `Util_ReadBigEndianUInt16` liefert `short` (Vorzeichen bei Werten > 0x7FFF) *(Agentenbefund, nicht separat geprüft)* | – |
| **BMS-Pack-Index um eins versetzt (alle Versionen 147–151):** CAN-Einheit u landet in Record u−1 (`CAN_AA_StoreUnit*`), Modbus 34000+100·n und `get_all_bms_data` lesen Record n; die MQTT-BMS-Abfrage (Befehl 0x2A) liest für Index p ≥ 1 aber Record p → „BMS(1)“ = CAN-Einheit 2, Einheit 1 per MQTT nicht einzeln abrufbar, Index 8 liest hinter das Array (ab Offset 0x20 `g_pendingBootDescriptor`, nur lesend) | `MQTT_Publish_BMS_Data` 0x08028DA4, `MQTT_CtrlCommand_Dispatcher` Fall 0x2A |
| **Zellstatistik nur 6 Packs (alle Versionen):** `BMS_CellVoltage_MinMax_PerPack` läuft `pack < 6`, obwohl 8 Records / 7 Modbus-Blöcke existieren → 7. Pack fehlt in `g_bmsCellStat*` und der daraus gebauten MQTT-Telemetrie | `BMS_CellVoltage_MinMax_PerPack`, `MQTT_Telemetry_String_Formatter` |

## 12. Aufbereitungsstand in Ghidra

| | v151 |
|---|---|
| Funktionen | 1956, alle benannt (0 `FUN_`), alle mit Prototyp (`__stdcall`), Plate-Kommentar, lokale Variablen soweit bestimmbar; 112 Namen nach Zweitprüfung „niedrige Sicherheit“ getaggt, 31 in der Zweitprüfung korrigiert |
| Flash-Abdeckung | 100 %: 0 undefinierte Bytes, 0 Instruktionen außerhalb von Funktionen |
| RAM-Globals | alle ~1 830 benannt, getypt, kommentiert (Ausnahme: 2 Offcut-Labels innerhalb getypter Arrays) |
| Peripherie | Register der N32G457-SVD (942 Einträge) + 16 XFMC + 5 SAC + Kernregister, dazu `sysinfo` und `extflash_ctrl` |
| Switch-Tabellen | alle TBB/TBH als Arrays getypt, 17 Decompiler-Overrides |

## 13. Offene Punkte

- ~~Registerblock 0xA0001000 (externer SPI-Flash-„QSPI“-Treiber): laut AT32F403A-Referenzhandbuch reserviert, Layout passt nicht zum QSPI1 des AT32F435. Welcher MCU-Block das ist, ist offen.~~ **Geklärt:** Die MCU ist ein Nations N32G45x mit XFMC; 0xA0001000 = `QSPI_BASE` (`XFMC_REG_BASE` 0xA0000000 + 0x1000) laut offiziellem `n32g45x.h`, bestätigt durch den Shell-Befehl `xfmc_test` (s. §1a).
- `Board_GetUniqueID_HexString` liest 0x1FFFF7F0/F4/F8; die dokumentierte UID endet bei 0x1FFFF7F3.
- Wer den Debug-Modus 2 im Feld setzt (`Debug_Mode_Set`-Aufrufer aus MQTT/BLE) und ob der Shell-Befehl `modbus` den Modus auch im EEPROM zurücksetzt, ist nicht geprüft.

## Anhang M. Methodik

- Funktionsnamen: Offline-Agenten (41 Batches) auf Basis des Dekompilats, v150-Namen nur als Hinweis; Zweitprüfung von 137 unsicheren Funktionen nach vollständiger Benennung.
- Globals: 34 Batches; Größen über Adressabstände, `memset`-Größen (Keil `__aeabi_memset(dst, n, c)`) und Format-Strings belegt. Namen mit allen anderen Versionen harmonisiert (gleiches Objekt = gleicher Name, ~300 Umbenennungen insgesamt), u. a. alle 8 BMS-Pack-Records auf `g_bmsPack<k><Field>`, `g_spiFlashStatusCmdTable` und `g_meterCtStabilizeDelayTable` neu getypt, sowie bislang undokumentierte Peripherie benannt: Registerblock 0x40024000 (damals `RNGX_*`, mittlerweile als N32G45x-SAC-Krypto-Coprozessor identifiziert, s. §1a), GPIOF/GPIOG bei 0x40011C00/0x40012000 (im N32G45x-Benutzerhandbuch als reserviert markiert), `RCC_UNDOC_2C`, `PWR_UNDOC_0C`, `RTC_UNDOC_4C`, Bit-Band-Aliase `BB_*`.
- Korrigierte Ghidra-Artefakte: Zeiger + String verklebt (~300), Strings als Code (> 250), unaufgelöste TBB/TBH-Tabellen, Literal-Pools im Code (`__scatterload`), Scheinfunktion in `small_primes_u8`, Konstantenpropagations-Referenzen auf Code, falsche `BRANCH`-Overrides auf `pop {…,pc}`, `llhttp_errno_name` (Sprung in Stringdaten, CALL_RETURN-Override).
- Literal-Pool-Wörter mit RAM-/Peripherie-Adresse als Zeiger getypt, damit der Decompiler Globals direkt zeigt.
- Nie `endTransaction(tx, False)` in ReVa-Skripten; Fehler im Skript übernehmen Teiländerungen.
