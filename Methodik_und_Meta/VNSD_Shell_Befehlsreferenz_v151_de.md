# Marstek Venus D — CLI-Shell Befehlsreferenz (Control v151)

> 🇬🇧 English version: [VNSD_Shell_Command_Reference_v151.md](VNSD_Shell_Command_Reference_v151.md)

**Stand:** 2026-10-17.
**Firmware:** Control v151 (`VNSD-0`), Build-String `Sep 29 2026 09:42:27`.
**Methode:** statisches Lesen der dekompilierten `ShellCmd_*`-Handler und des CLI-Kerns (`CLI_*`). **Nichts wurde auf Hardware getestet.**
**Begleitdokument:** [VNSD_Shell_de.md](VNSD_Shell_de.md) (Architektur und Zugangswege, Stand v149.2; wird hier korrigiert, wo es abweicht).

---

## 1. So funktioniert die Shell in v151

- Die Shell ist eine **letter-shell**: ein normaler Text-Prompt. Ein Befehl ist `name arg1 arg2 …` mit Enter am Ende. Die Zeile wird zerlegt (`CLI_TokenizeInput`), der Befehl über den **Namensstring** gefunden (`CLI_FindEntryByName`, `strcmp`). Zahlen liest `CLI_ParseNumber` (dezimal oder hex). Die Handler nehmen Integer-Argumente; ein Handler ohne Argumente ignoriert zusätzliche Tokens.
- `CLI_DispatchInputByte` erkennt **keine** Befehle. Es vergleicht nur mit den **10 Tastenbelegungen** der Tabelle (Eintragstyp 9: Pfeile, Backspace, Delete, Enter, Clear). Alle anderen Bytes landen im Zeilenpuffer. Die „4-Byte-Befehlsmuster“ in [VNSD_Shell_de.md](VNSD_Shell_de.md) §3 sind diese Tastenbelegungen, keine Befehlsnamen.
- **Tabelle:** `shellCommandTable` @ 0x08062cc4, **81 Einträge** = 68 Befehle + `help` + `clear` + 1 Benutzereintrag `VNSD-0 v151` + 10 Tastenbelegungen. Es gibt 70 `ShellCmd_*`-Handler (68 + `help` + `clear`), und alle 70 Befehlsnamen stehen als Strings im Image. Die Zuordnung Namensstring → Handler wurde **nicht** Eintrag für Eintrag dekodiert; die Zuordnung unten folgt den Handlernamen.
- **Login:** `CLI_InitSession` setzt beim Anlegen der Sitzung das Bit „authentifiziert“, ein Passwort wird nicht abgefragt. Der Benutzereintrag `VNSD-0 v151` hat ein leeres Passwort.
- **Ausgabe:** `printf` läuft über `Protocol_PutChar`. Ist ein TCP-Client an Port 8091 verbunden (`g_debugTcpSocketConnected`), geht der Text über CH395-Socket 1 raus (Flush bei LF, `$`, `:` oder 256 Byte). Ohne TCP-Client geht er auf die RS485-UART (nur in Modus 1). `Log_Printf`-Ausgaben erscheinen nur, wenn `g_debugMode == 1` ist, der Log-Level es erlaubt und das Logging an ist. Mehrere Strings sind chinesisch.

### 1.1 Zugang (Moduswechsel)

`Task_rs485` bedient die Shell nur bei `g_debugMode == 1` (EEPROM 0x900, gültig 0–2):

| `g_debugMode` | Was `Task_rs485` tut |
|---|---|
| 0 (Standard) | RS485-Modbus-RTU (`RS485_RTU_Frame_Dispatcher`); ein TCP-Client auf 8091 wird nicht gelesen |
| 1 | Shell (`Network_ReceiveAndDispatchData`) auf TCP 8091 oder RS485 |
| 2 | Wartet auf die Zeichenfolge `quit_debug_5100_` (`RS485_DebugConsole_ScanQuitPhrase`) und schaltet dann auf 1 |

- **Einschalten:** Modbus-Schreibzugriff mit `0x5100` (20736) auf Register 46000 setzt Modus 1 und speichert ihn in EEPROM 0x900 (`Write_Handler`; siehe [VNSD_Command_Register_45000_Block_v150_de.md](../Modbus_RS485_TCP/VNSD_Command_Register_45000_Block_v150_de.md)). Bleibt über Neustarts bestehen.
  - Frame (Modbus TCP, Port 502, Unit 1, FC06 Write Single Register, Register 46000 = 0xB3B0, Wert 0x5100): `00 01 00 00 00 06 01 06 B3 B0 51 00`. Ein normales Modbus-Gerät schickt die Anfrage als Antwort zurück. Nicht getestet.
  - **Genau `0x5100` senden.** Andere Werte in 46000 wählen OTA-Modi (`0x04D2`, `0x0929`, `0x0D80`, `0x11D7`), die die Tasks für Modbus TCP, WLAN und Cloud löschen und den Inverter stoppen.
- **Ausschalten:** Shell-Befehl `modbus` (setzt Modus 0 und speichert ihn).
- **Nebenwirkung:** In Modus 1 wird der RS485-Modbus-RTU-Port nicht bedient. Ob Modbus TCP (Port 502) weiterläuft, wurde nicht geprüft; er nutzt einen anderen Socket.
- **Warum die früheren Live-Tests stumm blieben:** Das Gerät stand in Modus 0, auf 8091 hat also niemand den Socket gelesen.

---

## 2. Befehle

Adresse = Handler-Funktion. „Args“ sind die Handler-Parameter laut Dekompilat.

### 2.1 Nur lesend (geben Werte aus)

| Befehl | Handler | Args | Ausgabe |
|---|---|---|---|
| `get_ver` | 0x0803be10 | – | Build-String, `ems_type`, `boot_ver`, `ems_ver` (eine Konstante, 0x97 = 151), `mppt_ver` (neu und alt), `bms_ver`, `vns_ver` |
| `get_venus_data` | 0x0803baa4 | – | Inverter-Block: Zustand, Fehler-/Warnwörter, Netz- und Off-Grid-Spannung/-Frequenz/-Leistung, Batterieleistung/-spannung, Temperaturen, Leistungsgrenzen, Energiezähler, Hard-/Soft-/Boot-Version, Work-Mode, Sleep-Flag, Batteriemodus |
| `read_inv2` | 0x08053334 | – | Struktur `g_vns_inv_debug` (Work-Mode, Grenzen, Allow-Flags, `err_state`, `bat_err_req`, …) |
| `get_all_bms_data` | 0x0803ae6c | – | Für jeden Datensatz `0 … bat_total_nb-1`: Spannung, Strom, SoC, Zyklen, Ladegrenzen, MOS-Flag, min/max Zellspannung und Temperatur, err1/err2/warn1, BMS-Version, Lade-/Entladeleistung, SoH, 16 Zellspannungen, NTC-Temperaturen, Umgebungs- und MOS-Temperatur, Batterietyp |
| `get_main_bms_data` | 0x0803b394 | – | Summen aus PGN 1801/1802/1803/1804/1007: Spannung, Strom, SoC, Energie, `bat_total_nb`, `bat_online_mask`, `work_bat_idx`, DOD, Ladegrenzen, Flags, Fehler, Lock-Flag, Selbsttest |
| `get_mppt_data` | 0x0803b754 | – | MPPT-Zustand/-Fehler/-Temperatur/-Warnung, 4× PV-Spannung/-Strom/-Leistung, Kapazitätszähler, Batterie- und Busspannungen (dieselben Daten wie MQTT `cd=45`) |
| `ems_rtrl_info` | 0x0803a2d4 | – | Letzte Werte, die das EMS per CAN an den Inverter (Leistung, Sendezähler) und an den MPPT (Spannung, 2 Temperaturen, Leistung, Modus, Sendezähler) geschickt hat |
| `energy_show` | 0x0803a41c | – | Lade- und Entladeenergie gesamt/Monat/Tag (Wh) |
| `err_code` | 0x0803a63c | – | Bitmasken `err_code` und `warn_code` des Controllers |
| `get_log` | 0x08053030 | `0` / `1` | `0`: 20 Fehlerlog-Slots; `1`: 20 Eventlog-Slots (Datum und Codes) |
| `get_ota_log` | 0x0803ba04 | – | OTA-CRC, `ota_flag`, `fail_flag`, 4 Fehlercode/Daten-Paare |
| `ext_info` | 0x0803a6cc | – | Lade-/Entlade-Cutoff-SoC, max. Lade-/Entladeleistung, Netznorm |
| `power_check` | 0x0804ff6c | – | CT-Leistung, Venus-/BMS-Leistung, Last, PV, Backup-Leistung |
| `rtos_status` | 0x0805378c | – | FreeRTOS-Taskliste als JSON (Name, Zustand, Priorität, freier Stack, Nummer) |
| `FreeRtos_mem` | 0x0802fa38 | – | Freier Heap und historisches Minimum (über `Log_Printf`, also nur bei eingeschaltetem Logging sichtbar) |
| `reset_reason` | 0x080512de | – | Letzte Reset-Ursache |
| `get_time` / `rtc_show` | 0x080531a4 / 0x0802fa30 | – | RTC-Datum und -Uhrzeit |
| `wifi_info` | 0x080531f4 | – | RSSI und Verbindungsstatus |
| `ch395_get_ip` | 0x08037e70 | – | CH395: IP/Gateway/Maske/DNS/DHCP-Status |
| `ch395_ver` | 0x0803866c | – | CH395-Chipversion |
| `mac_ble` | 0x08052f78 | – | BLE-/Cloud-Identitätsfelder (type, id, mac, mac_devid, vid, gid, xid) |
| `get_meter_mac` | 0x0803b734 | – | Gibt die Variable `g_deviceSerialNumber` unter der Bezeichnung `meter_mac` aus (Bezeichnung und Variable passen nicht zusammen) |
| `show_modbus` | 0x080539a4 | – | Modbus-Slave-Adresse |
| `check_ssl_cert` | 0x08052fb8 | – | TLS-Zertifikatsversion |
| `ver_fc` | 0x08052ffc | – | Sendet `AT+QVERSION` an das FC41D und gibt dessen Firmware-Version aus |
| `key_state` | 0x0803c208 | – | Taster: Entprellzähler, Ereignistyp, Tastenzustand, Encode-Flag |
| `get_old` | 0x08053180 | – | Burn-in-Schrittzähler |
| `help` | 0x08052300 | keins / Name | Befehlsliste oder Hilfe zu einem Befehl |
| `clear` | 0x08051b64 | – | Bildschirm löschen |

### 2.2 Ändern Einstellungen oder Log-Zustand (geringes Risiko)

| Befehl | Handler | Args | Wirkung |
|---|---|---|---|
| `set_level` | 0x080329d8 | level | Log-Level (`Log_SetLevel`) |
| `debug_fc` | 0x08052f6c | mode | `Log_SetModeAndLevel(mode)`; laut Dekompiler-Zusammenfassung 0 keine, 1 FC41D, 2 CT, 3 SYS, 4 CH395, 5 alle (nicht verifiziert) |
| `trace_debug` | 0x08053b64 | `2025` | Setzt `g_debugPrintfEnableFlag` und `g_httpVerboseLogFlag`, wenn das Argument 2025 (0x7e9) ist, sonst löscht es sie |
| `api_set` | 0x080537c4 | `1` / `0` / sonst | Lokale UDP-API ein / aus (EEPROM 0x371) / Status ausgeben |
| `api_port` | 0x080538ac | port | Port der lokalen API (EEPROM 0x372) |
| `modbus` | 0x0805330c | – | Setzt `g_debugMode = 0` (EEPROM 0x900): zurück auf Modbus-RTU |
| `get_backup_datas` | 0x0803b324 | value | **Setzt** `g_backupDataRequestFlag` (dasselbe Flag wie MQTT `cd=61`), trotz des Namens |
| `read_inv1` | 0x08053328 | – | Setzt `g_maintModeState = 2` |
| `code_test` | 0x08037c08 | value | Setzt `g_encodeFlag` |
| `Resume_task` | 0x0802f680 | – | Setzt angehaltene FreeRTOS-Tasks fort |
| `ch395q_test` | 0x08038820 | cmd | Schickt ein rohes Kommandobyte an den CH395 und gibt die Antwort aus |

### 2.3 Stellend oder zerstörend (nicht auf einem produktiven Gerät ausführen)

| Befehl | Handler | Args | Wirkung |
|---|---|---|---|
| `restore_factory` | 0x0802f590 | `1` / `2` / `3` | `Factory_Reset`: 1 alles löschen, 2 teilweise löschen (angesammelte Daten bleiben), 3 Zertifikate löschen |
| `venus_rfd` | 0x08056ff2 | mode | Inverter-Werksreset (`Inverter_Write_Reg_0x07_Value(1, mode)`); laut Dekompiler-Zusammenfassung: 0 alles, 1 das meiste, 2 das meiste außer angesammelter Energie |
| `venus_poweroff` | 0x08056fe4 | sleep_flag | Inverter-Register für Sleep/Ausschalten (`Inverter_Set_Flag_Reg_0x06`) |
| `reset` | 0x080512d6 | – | Startet den MCU neu |
| `set_wifi` | 0x0805394c | – | Überschreibt gespeicherte WLAN-SSID/-Passwort mit den eingebauten Defaults `SM471` / `Hame147258` (EEPROM 0x400/0x420) und löst eine Neuverbindung aus |
| `delate_ssl_cert` | 0x08052fe8 | – | Löscht das gespeicherte TLS-Benutzerzertifikat samt Schlüssel |
| `fc41d_reset` | 0x08052ff0 | – | Hardware-Reset des FC41D-Moduls |
| `hard_reset_395` | 0x0803bf4c | 0/1 | Schaltet den Reset-GPIO des CH395 auf Low/High |
| `recon_ch395` | 0x080512cc | – | Reset und Neuinitialisierung des CH395 |
| `free_mqtts` | 0x0803a9d8 | – | Setzt `g_ch395ResetPendingFlag`; die Dekompiler-Zusammenfassung sagt „MQTTS abbauen“, der Variablenname sagt CH395-Reset — ungeklärt |
| `change_soc` | 0x08052f60 | 0…100 | Sendet den Wert als 8-Byte-CAN-Frame `0x100201AA` (`Config_Write_TargetSOC_Percent`); Werte über 100 werden ignoriert. Das ist ein **Ziel-SoC-Schreibzugriff Richtung BMS**, keine Änderung des gemeldeten SoC. Die Wirkung im BMS wurde nicht verfolgt |
| `full_dischrg` | 0x0803a9e8 | 0/1 | `Config_Write_BoolFlag_0x375` (Überschuss-Einspeisung, dasselbe Flag wie MQTT `cd=43`) |
| `lck_bms` | 0x0803c298 | lock | `Config_Apply_WorkModeReg(lock)` (BMS sperren/entsperren, derselbe Pfad wie MQTT `cd=44`) |
| `ctl_mppt_power` | 0x080388c4 | power | Setzt `g_gridExportPowerCap3 = power` und `g_maintModeState = 3` |
| `ctrl_led_light` | 0x08038978 | 0…7 | LED-Helligkeit (0 am hellsten) über den prüfenden Setter |
| `led_continue_ctrl` | 0x0803c2a4 | mode, value | Trotz des Namens: `Inverter_Power_Setpoint_Apply(mode, value)` |
| `test` | 0x080539d0 | – | `g_maintModeState = 2`, Inverter an, danach ein CAN-Leistungskommando mit Rohwert 1000 |
| `set_old` | 0x0805392c | step < 6 | Burn-in: `g_maintModeState = 1`, Inverter-Work-Mode-Register 0x06 und 0x60 |
| `vns_ate_test` | 0x08057000 | – | `Config_Apply_SingleReg(1)` (ATE-Testkommando des Inverters) |
| `debug_mode` | 0x08056f98 | 0/1 | `Debug_Mode_Set`: löscht vier CAN-0xC2-Unterregister; mit 1 setzt es zusätzlich `g_debugMode = 2` (die Shell-Eingabe wird dann durch den Quit-Phrase-Scan ersetzt, bis `quit_debug_5100_` getippt wird) |
| `update_ems` / `update_vns` / `update_bms` / `update_mppt` | 0x08055f60 / 0x08056268 / 0x08055f20 / 0x08055fa0 | – | Setzen `g_otaTargetDevice` = 1 / 4 / 3 / 2. `Task_rs485` ruft danach `CLI_FirmwareUpdate_Prepare` auf (YModem über TCP/RS485, siehe §4) |
| `button_ctrl` | 0x08037dcc | level | Schaltet den Taster-GPIO (Port E, Maske 0x8000) auf Low (0) oder High |
| `eeprom_flash_test` | 0x0800b78c | addr | **Überschreibt 32 Byte EEPROM ab `addr`** (0xFF, 0…31, danach wieder 0xFF) |
| `flash_test` | 0x0803a7b0 | addr | **Löscht und beschreibt 30 Byte Flash ab `addr`** |
| `xfmc_test` | 0x08035a9c | – | Selbsttest des externen SRAM |
| `check_periph` | 0x08053a64 | – | Peripherie-Selbsttest; gibt EEPROM/Flash/CH395/SRAM als OK oder Fehler aus (chinesisch) |

---

## 3. Nutzen für das Projekt

Lesende Befehle, mit denen sich statische Befunde am laufenden Gerät prüfen lassen (nur nach Entscheidung des Eigners und nur in Modus 1):

| Frage | Befehl |
|---|---|
| Versionsprüfung bei jeder neuen Firmware (Pflichtprüfliste) | `get_ver`, `ver_fc` |
| Pack-Kapazität und angemeldete Packs (Pflichtprüfung „Pack-Kapazität“) | `get_main_bms_data` (`bat_total_nb`, `bat_online_mask`), `get_all_bms_data` |
| Pack-Nummerierung: Datensatz-Index gegen CAN-Unit (Datensatz-Index = CAN-Unit − 1) | `get_all_bms_data` gegen MQTT `cd=42` |
| Fehlerwörter von Inverter und BMS | `err_code`, `get_venus_data` (`error_code1/2`), `read_inv2`, `get_log` |
| Im RAM aktive Registerwerte (Grenzen, Cutoff-SoC, Netznorm) | `ext_info`, `get_venus_data`, `read_inv2` |
| Was das EMS an MPPT/Inverter sendet | `ems_rtrl_info` |
| Taskliste und Stack-Reserven | `rtos_status`, `FreeRtos_mem` |
| OTA-Ergebnis | `get_ota_log` |

Auffälligkeiten für die Registerklassifikation: `get_backup_datas` und `read_inv1` schreiben Zustand, obwohl sie wie Leser aussehen; `led_continue_ctrl` ist kein LED-Befehl; `get_meter_mac` gibt die Variable der Geräte-Seriennummer aus. `lck_bms`, `full_dischrg` und `get_backup_datas` erreichen dieselben Ziele wie MQTT `cd=44`, `cd=43` und `cd=61`. `change_soc` hat in der Befehlszuordnung keine MQTT-Entsprechung.

---

## 4. Der `update_*`-Pfad: Firmware-Upload über TCP oder RS485 (YModem)

Am 2026-10-17 aus dem v151-Dekompilat gelesen; **nicht auf Hardware getestet**. Das ist der dokumentierte Kontrollfluss eines vom Hersteller ausgelieferten Update-Pfads, hier für die projekteigene Pflichtprüfliste des Firmware-Diffs festgehalten — keine Anleitung für Dritte.

1. **Auslöser.** `g_otaTargetDevice` muss ungleich 0 sein: Modbus-Schreibzugriff mit 1234 / 2345 / 3456 / 4567 auf Register 46000 (Ziele 1 / 4 / 3 / 2) oder die Shell-Befehle `update_ems` / `update_vns` / `update_bms` / `update_mppt` (1 / 4 / 3 / 2). `Task_rs485` prüft dieses Flag vor dem Debug-Modus-Zweig, es gilt also unabhängig von `g_debugMode`.
2. **Vorbereitung (`CLI_FirmwareUpdate_Prepare`).** Löscht die Tasks FC41D, Parser, HTTP, UDP, CT, Kanal-HTTP, Kanal-MQTTS und Modbus TCP, ruft `Inverter_BeginShutdown(0)` auf, leert den CH395-Empfangspuffer und ruft dann `YModem_FirmwareUpgrade_Handler` auf. Der Inverter stoppt; Cloud und Modbus TCP sind bis zum Ende der Übertragung nicht verfügbar.
3. **Übertragung (`YModem_ReceiveFile`).** Ist ein Client an Port 8091 verbunden, kommen die Blöcke von CH395-Socket 1 (`CH395_Packet_Receive_Parse`); sonst vom RS485-Parser. Die Blöcke haben die übliche YModem-Form (SOH/STX-Header, Sequenzbyte, invertiertes Sequenzbyte, Nutzdaten, 2 Schlussbytes). Der Parser prüft nur das Header-Byte und `seq == ~seq`; die beiden Schlussbytes jedes Blocks werden nicht ausgewertet.
4. **Prüfungen bei Block 0 (Dateikopf).** Das Namenskürzel der Datei (in Großbuchstaben, abgeglichen gegen EMS/BMS/INV/MPPT in dieser Reihenfolge) muss auf dasselbe Ziel wie `g_otaTargetDevice` abbilden; die angekündigte Größe darf höchstens 0x7CFFF (511999) Byte betragen. Danach wird der Staging-Bereich des Ziels (aus einer 4-Einträge-Adresstabelle) gelöscht und die Datenblöcke werden dort geschrieben.
5. **Abschlussprüfung (`OTA_CRC_Verify`).** Eine laufende Modbus-CRC16 über die empfangenen Bytes wird mit der CRC16 derselben, aus dem Flash zurückgelesenen Staging-Bytes verglichen. Beide Werte stammen aus derselben Übertragung, es ist also eine Übertragungs-/Flash-Integritätsprüfung, keine Echtheitsprüfung: Der Handler bekommt keinen unabhängig mitgelieferten Hash, keine Signatur und keinen Schlüssel zum Vergleich. Die getrennte Prüfkette 0x401/0x402/0x403 (`OTA_FW_Verify_SetStatus`, die tatsächlich gegen eine *angekündigte* CRC sowie eine Geräte-/Modellkennung vergleicht) gehört zum HTTP-/BLE-OTA-Pfad (ihr Aufrufer ist `OTA_Flash_Page_Writer`) und wird hier nicht aufgerufen.
6. **Aktivierung (`ProcessFirmwareUpdateCommand`).** Für Ziel 1 (dieses Control) schreibt sie einen Pending-Boot-Descriptor (Staging-Adresse, Größe, CRC16, Kennung `VNSD-0`, eigene CRC16) nach `BOOTLOADER_ParamPage` (0x08004000, 40 Byte) und startet neu. Für Ziele 2–4 legt sie ein ausstehendes, per CAN weitergereichtes Update für MPPT/BMS/Inverter an, ohne Neustart.
7. **Bei Fehler** (Abbruch, falsche Kennung, zu groß, CRC-Abweichung): `System_ResetToDefaults` schaltet den Inverter wieder ein und setzt die angehaltenen Tasks fort; die gelöschten Tasks werden dabei nicht neu angelegt, ein vollständiger Neustart kann also nötig sein (nicht verifiziert).

**Der Control-Bootloader wurde nicht geprüft und kann es mit dem, was dieses Projekt vorliegen hat, auch nicht sein.** Der Bootloader ist ein eigener 18-KB-Bereich (0x08000000–0x080047FF) unterhalb der Link-Adresse der App (0x08004800, bestätigt durch die Vektortabelle in diesem v151-Export und durchgehend in allen analysierten Control-Builds D/A/E3). Es ist Firmware, die auf demselben MCU läuft, aber **in keinem der von diesem Projekt dekompilierten oder gedumpten App-Images enthalten ist** — jede bisher untersuchte `.bin` beginnt erst bei 0x08004800. Die einzige dokumentierte Interaktion der App mit ihm ist das Schreiben des 40-Byte-Descriptors nach 0x08004000 und das Lesen eines Versionsworts bei 0x08001000; was der Bootloader selbst prüft, bevor er das Staging-Image über die laufende App flasht, ist unbekannt. Zum Lesen bräuchte es einen Flash-Dump der unteren 18 KB, z. B. per SWD/JTAG (siehe [VNSD_SWD_JTAG_Dump_Plan_de.md](VNSD_SWD_JTAG_Dump_Plan_de.md)) — für diese Frage nicht unternommen.

**Nicht der werkseigene Bootloader des MCU.** Der N32G45x (wie seine pinkompatiblen STM32-Verwandten) hat einen eigenen ROM-residenten System-Bootloader für UART-ISP, der über den BOOT-Pin bzw. Option Bytes angesprungen wird und in einem eigenen System-Speicherbereich liegt, nicht im Hauptflash. Dieser 18-KB-Bereich sitzt dagegen genau am normalen Flash-Boot-Vektor (0x08000000) — der Adresse, die die CPU nach einem gewöhnlichen Reset mit Flash-Boot erreicht —, und das Deskriptorformat, das er auswerten soll (Kennung `VNSD-0`, 40-Byte-Struktur, Modbus-CRC16), ist produktspezifisch. Beides spricht für einen vom Hersteller selbst geschriebenen Zweitstufen-Bootloader, der Teil dieser Firmware ist, nicht für generischen Silizium-Hersteller-Code — das ist aber aus der Lage und aus dem, was die App hineinschreibt, gefolgert, nicht aus dem Lesen des Bootloader-Codes selbst.

Betrieblich: Die Übertragung stoppt den Inverter und beide Netzwerk-Steuerpfade für ihre Dauer, und das Verhalten bei einer abgebrochenen oder abgeschnittenen Übertragung wurde nicht verifiziert — nichts, was man an einem produktiven Gerät ausprobiert.

---

## 5. Offene Punkte

1. Nicht dekodiert: der Tabelleneintrag je Befehl (Namensstring → Handler-Zeiger, Flags). Die Zuordnung oben folgt den Handlernamen. Das Lesen der 81 Einträge aus dem Image würde es klären.
2. Die Argumentsyntax wurde nicht getestet (Trennzeichen, Hex-Eingabe, fehlende Argumente). Die Handler lesen ihre Parameter ohne Prüfung der Argumentanzahl.
3. Der Control-Bootloader (0x08000000–0x080047FF) und die Bootloader von MPPT/BMS/Inverter sind ungelesen; siehe den Schlussabsatz von §4 zum Warum und was dafür nötig wäre.
4. `free_mqtts`: Name gegen Flag. `debug_fc`: Die Modusbedeutungen stammen aus der Dekompiler-Zusammenfassung.
5. Ob Modbus TCP in Modus 1 verfügbar bleibt.
6. Andere Versionen (147, 149, 149.2, 150; A- und E3-Linie) wurden nicht verglichen. Laut v150-Analyse hat v150 66 Befehle (ohne `read_inv1`/`read_inv2`); v151 hat 68.
