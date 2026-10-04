# VNSD – Auflösung der bislang unklaren Modbus-Register (Venus D, 2026-10-01)

> 🇬🇧 English version: [VNSD_Unclear_Registers_Resolution_2026-10-01.md](VNSD_Unclear_Registers_Resolution_2026-10-01.md)

Ergebnis einer vollständigen Codeanalyse für jedes Register, das in der Integration Marstek Modbus Suite (`d.yaml`) fehlte und kein reiner Alias eines enthaltenen Registers war. Umfang: Venus D, Control 147 / 149 / 149.2 / 150 / 151, Micro 115 / 116, BMS 117.7 / 118, FC41D 202409 / 202512. Venus A und E3 folgen in separaten Dokumenten. Alles ist aus dem dekompilierten Code (`Dekompilate/`, lokal) abgeleitet – keine Vermutungen; wo der Code eine Aussage nicht belegen kann, wird das ausdrücklich gesagt.

## 1. Methode

1. Start beim Modbus-Register: Handler-Case in `Write_Handler` / Lese-Deskriptor (alle fünf Control-Versionen, dezimale und hexadezimale Switch-Form).
2. Den Wert bis zum letzten Verbraucher bzw. Ursprung über Geräte hinweg verfolgen: Control → CAN (Frame-Klasse = ID-Bits 12–15, FDCAN-Filter, Dispatcher-Case, Global, Verbraucher) → Micro / BMS; Control → UART → FC41D; GPIO; EEPROM.
3. **Wirkungslosigkeitsprüfung (Functionless-Check)** für jedes gefundene Ziel: Wert wird nie gelesen, Zweig nicht erreichbar, Empfänger filtert den Frame, Befehl wird sofort überschrieben, Modus-Gate wird nie erfüllt.
4. Unabhängige adversariale Verifikation der 21 folgenreichsten Aussagen durch zwei weitere Analysten, die nur mit dem Code arbeiteten: keine widerlegt, eine teilweise falsch, mehrere Formulierungen präzisiert (unten eingearbeitet).

Statuswerte: **FUNCTIONAL** = hat die dokumentierte Wirkung · **CONDITIONAL** = nur in einem Sondermodus / nach einem anderen Schreibzugriff · **NO_OP** = wird akzeptiert, hat aber keine Wirkung · **FUNCTIONLESS** = Ziel existiert, wirkt aber nie (gefiltert, nie gelesen, kein Empfänger) · **UNKNOWN** = Ziel gefunden, Bedeutung nicht beweisbar.

Anzahl (179 Register): FUNCTIONAL 60, FUNCTIONLESS 49, CONDITIONAL 39, UNKNOWN 23, NO_OP 8.

## 2. Wichtigste Erkenntnisse

- **41001 ist auf dem Venus-D-Control nicht implementiert** (kein Case in irgendeiner Version; Lesen und Schreiben liefern Modbus-Exception 2, kein Seiteneffekt). Der Suite-Button `factory_reset` (41001) tut daher nichts. Der Micro hat ein eigenes 41001 (Statistik löschen, Micro-Neustart), das aber nur über den eigenen RS485-Modbus-Slave des Micro erreichbar ist, nicht über das Control.
- **41000**: `0x55AA` = sauberer Control-Neustart; nach dem Boot sendet das Control den CAN-Befehl 0x05, der auch den Micro neu startet (deshalb stellt 41000 den Normalbetrieb nach 45000 = `0x55EE` wieder her). `0xAA11` = vollständiger Werksreset von Control, Micro und BMS (löscht auch die Cloud-TLS-Zertifikate des FC41D und den Cloud-Identitätsblock). Es schreibt zuerst die RS485-Adresse 17, aber der Reset lädt beim nächsten Boot die Defaults neu (Adresse wird 1).
- **40000** startet den Batteriekalibrierungs-Test (Zyklus) des Micro (auf 100 % laden, unter 30 % entladen, wieder laden, bis 2,5 kW); der Micro ignoriert währenddessen die Leistungs-Sollwerte des Control. Gefährlich – nie bereitstellen.
- **45000** (`0x55EE` / `0x55BB`) schreibt `g_workMode` des Micro (12 = Werksmodus / 1 = manuell) über den CAN-0x50-Handler und sendet Klasse-2-„aus“. Es senkt den Leerlaufverbrauch nicht (live getestet). Ein Control-Boot stellt den Zustand wieder her.
- **Der Klasse-2-CAN-Knoten eines Venus D ist eine separate Einheit, höchstwahrscheinlich der MPPT-/PV-Controller** (weder Micro 115/116 noch das BMS können Klasse-2-Frames senden, dennoch zeigen 30028/30036 und `mppt_ver` Live-Werte). Die Befehle 45020–45024 erreichen ihn; was sie dort bewirken, lässt sich ohne seine Firmware nicht bestimmen.
- **„Battery Pack Recovery“ in der Marstek-App ist Pack-Adresskodierung, kein Wiederherstellungsladen** (durch Dekompilieren der App 1.6.76 belegt): Die Schaltfläche sendet BLE `73 06 23 40 00 16` / MQTT `cd=40,number_as`, d. h. CAN `0x100101AA` Byte 3, was den Kodierungs-Handshake des BMS scharf schaltet. Modbus **45029** ist das Modbus-seitige Äquivalent (es sendet auch die PB10-Impulsfolge). Details: `../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md`.
- **45027** ist NICHT harmlos (ältere Dokumente): Klasse-3-Frame an das BMS-Master-Pack – sofortiger BMS-Neustart (und EEPROM 0x202 = 0xFF bei einem Pack, das zur Laufzeit Master geworden ist). **45028** = BMS-Werksreset + Neustart aller Packs. Nie bereitstellen.
- **45001 ist kein BMS-Reset** (ältere Dokumente): es löst einen EEPROM-Schreib-Selbsttest des Micro aus (Ergebnis in 30213). **45012** ist ein Lampentest der Frontpanel-LEDs, kein Leistungs-Sollwert. **45010** ist der echte manuelle Leistungs-Sollwert (+ Entladen / − Laden, nur Modus 1, keine Control-seitigen Schutzfunktionen).
- **WLAN 41500–41631** funktioniert als Pfad „WLAN-Zugangsdaten setzen“ zum FC41D (`AT+QSTAAPINFODEF`), in allen Versionen identisch. Zwei Register pro zwei Zeichen, die SSID-Register lesen die SSID zurück, die Passwort-Register lesen 0. Herstellerfehler: Der Passwort-Bereich EEPROM 0x420–0x45F überlappt 0x441 (`g_httpCloudServerType`).
- **46000**: `0x5100` macht aus dem RS485-Modbus-RTU-Port die CLI-Shell (persistent, EEPROM 0x900); die OTA-Codes löschen die Modbus-TCP-/WLAN-/Cloud-Tasks und stoppen den Wechselrichter. **46500–46544** und **47001** sind auf dem D FUNCTIONLESS (kein Empfänger). **47400** ist ein No-Op.
- **38000–38014**: kein Sender unter der bekannten Firmware (Micro 115/116 und BMS 117.7/118 senden die Frames 0x40–0x43 nie; der Block fehlt in v147), aber die Einheit vom Typ Klasse 2/MPPT und Hardware vom Typ A/E2 lassen sich nicht ausschließen: Status **UNKNOWN (offen)**, als DEV beibehalten (deaktiviert, nie abgefragt). Ein Lesezugriff in diesem Bereich lässt das Control bis zum Neustart vier CAN-Frames pro Sekunde broadcasten.
- **34009 / 34109 / …** sind nicht „reserviert“: Sie sind die Warn-Bitmaske des BMS pro Pack (Wert 2 = Zellunterspannung; Schwelle 3100 mV bei ≥ 0 °C, 2750 mV unter 0 °C; zweite Stufe 3000 / 2550 mV). **32112** = BMS-Sperre aktiv; **32107** (= 35112) = Entladestromgrenze des BMS (0,1 A).
- **32101** ist die vorzeichenbehaftete Summe der Pack-Ströme (0,1 A), aber der Modbus-Lese-Serializer teilt vorzeichenlos (Laden liest ganze Ampere, Entladen Müll) – nicht bereitstellen.
- Nebenbefund (alle Versionen): Jede Modbus-TCP-Exception-Antwort hat MBAP-Länge 4, aber es folgen nur 3 Bytes, sodass Clients auf ein fehlendes Byte warten (siehe `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md`).
- Nebenbefund: `Idle_Watchdog_AutoReboot` sendet nach 3600 s Leerlauf BMS-Opcode 1 auf CAN-ID 0x100101AA; bei Nicht-Master-Packs läuft das in einen BMS-Werksreset durch.

## 3. Registertabelle

Spalten: `status`, `target` (Gerät: Verbraucher) und die Empfehlung für die Integration. Vollständige Details (Lese-/Schreibverhalten, Datenfluss, Versionen) stehen in den CSV-Spalten `status_D`, `ziel_D`, `leseeffekt_D`, `befund_D`, `gefahr_D`, `suite_empfehlung_D`, `befund_konfidenz_D` der `VNSD_Register_Map_Final_generated.csv`. Aufeinanderfolgende Register mit identischem Ergebnis sind in einer Zeile zusammengefasst.

### Wartungs- und Reset-Register (40000–41001)

| Register | Name | Status | Ziel | Empfehlung |
|---|---|---|---|---|
| 40000 | `burnin_cycle_test_start` | FUNCTIONAL | Micro: autonomer Batterie-Lade-/Entlade-Zyklustest des Micro (g_workMode=4, g_battCalState) | NICHT bereitstellen (höchstens ein reiner DEV-Button 'Start burn-in / battery cycle test' mit Warnhinweis). Der Fortschritt wäre in 37001 und 30010 (=4) sichtbar. |
| 41000 | `system_command` | FUNCTIONAL | Control: 0x55AA: Neustart von Control + Micro. 0xAA11: Werksreset von Control, Micro und BMS (+ FC41D-Zertifikate) | 'reset_device' als Button 'Restart system' (0x55AA) beibehalten. Er startet Control und Micro sauber neu. Optional ein separater, standardmäßig deaktivierter Button 'Factory reset (Control+Inv... |
| 41001 | `unhandled_register_41001` | FUNCTIONLESS | keins: keins (Exception 2) | Den Button 'factory_reset' für Venus D entfernen: Er tut nichts und ist falsch beschriftet. Ein echter Werksreset ist 41000=0xAA11. |
| 41500–41515 | `wifi_ssid_00 … wifi_ssid_15` | CONDITIONAL | FC41D: WLAN-STA-Konfiguration des FC41D (AT+QSTAAPINFODEF), Control-EEPROM 0x400 bei Erfolg - nur über den Passwort-Commit | Nicht als Entitäten bereitstellen. Nur innerhalb eines optionalen, schreibgeschützten (write-only) 'set Wi-Fi'-Dienstes verwenden (erweitert/DEV). Ein Diagnosesensor zum Zurücklesen der SSID ist möglich, aber datenschutzsensi... |
| 41600–41631 | `wifi_password_00 … wifi_password_31` | FUNCTIONAL | FC41D: WLAN-STA-Zugangsdaten des FC41D (AT+QSTAAPINFODEF -> FC41D-Flash-IDs 0x11111104/05 + Assoziation); Control-EEPROM 0x400/0x420 nach bestätigter Co... | Nicht als Entitäten. Optionaler Write-only-Dienst 'set_wifi(ssid,password)' hinter einer erweiterten/DEV-Option, mit strikter Validierung; nie zurücklesen. |

### MPPT-/Wechselrichter-/Konfigurations-Leseregister (30028–30214, 37000–37003)

| Register | Name | Status | Ziel | Empfehlung |
|---|---|---|---|---|
| 30028 | `mppt_battery_side_voltage` | FUNCTIONAL | keins: Ursprung: batterieseitige Spannungsmessung des MPPT-Controllers; Verbraucher: nur Shell/MQTT | Als DEV (standardmäßig deaktiviert) hinzufügen: batterieseitige Spannung der Einheit vom Typ MPPT (nicht die PV-Strings, die in der Suite bereits als mppt1..4_voltage vorhanden sind); die Ursprungseinheit hat ... |
| 30029 | `mppt_battery_side_current` | FUNCTIONAL | keins: Ursprung: batterieseitiger Strom des MPPT-Controllers (eigener Wandlerstrom, nicht der Pack-Strom) | Als DEV (standardmäßig deaktiviert) hinzufügen: batterieseitiger Wandlerstrom der Einheit vom Typ MPPT (vorzeichenbehaftet, 0,1 A; nicht die PV-Strings und nicht der Pack-Strom); die Ursprungseinheit hat kein... |
| 30030 | `mppt_reply_0xfe_word` | CONDITIONAL | keins: keins (Wert wird nur für den Modbus-Lesezugriff gespeichert) | Als DEV (standardmäßig deaktiviert) beibehalten: Antwortwörter der Einheit vom Typ Klasse 2/MPPT, nur nach einem Schreibzugriff auf 45023/45024 gültig; eine Zuordnung ist ohne ihre Firmware unmöglich. |
| 30031–30035 | `mppt_reply_0xff_slot1_w0 … mppt_reply_0xff_slot2_w0` | CONDITIONAL | keins: keins | Als DEV (standardmäßig deaktiviert) beibehalten: Antwortwörter der Einheit vom Typ Klasse 2/MPPT, nur nach einem Schreibzugriff auf 45023/45024 gültig; eine Zuordnung ist ohne ihre Firmware unmöglich. |
| 30036 | `mppt_base_voltage` | FUNCTIONAL | BMS: Ursprung MPPT; Verbraucher: Bit 'MPPT vorhanden & fehlerfrei' in 0x1801FFAA -> BMS-Master-Enable-Handshake | Als DEV (standardmäßig deaktiviert) beibehalten: konstante Referenz von ca. 2,19 V (0,01 V) der Einheit vom Typ MPPT; ein Wert ungleich null (bei g_mpptError == 0) setzt das Master-Enable-Byte in C... |
| 30211 | `inverter_hw_fault_input` | CONDITIONAL | Micro: Ursprung: Pegel des Micro-GPIO PB1 (Eingang des Hardware-Fehler-Komparators) | Nur DEV, gepaart mit einem 45006-'query'-Button; Name 'Inverter hardware-fault input (PB1)' |
| 30213 | `inverter_eeprom_test_ack` | CONDITIONAL | Micro: Ursprung: ACK des EEPROM-Schreib-/Rücklesetests des Micro (immer 1) | Nicht bereitstellen (der Wert ist das konstante ACK des Micro-EEPROM-Schreib-Selbsttests 45001; 45001 ist KEIN BMS-Reset). |
| 30214 | `bms_self_check` | UNKNOWN | keins: keins auf dem Venus D (kein Sender) | Als DEV (standardmäßig deaktiviert) beibehalten: benötigt einen Statusframe, den weder BMS 117.7/118 senden; Sender unbekannt (BMS vom Typ A/E2 oder andere Einheit) - UNKNOWN, für A/E3 erneut prüfen. |
| 37000 | `rs485_slave_address` | FUNCTIONAL | Control: RS485-RTU-Adressfilter | Diagnosesensor 'RS485 slave address' (redundant, wenn 41100 bereitgestellt wird) |
| 37001–37003 | `inverter_calibration_step … inverter_calibration_counter_step3` | CONDITIONAL | Micro: Ursprung: Zustandsautomat der Batteriekapazitätskalibrierung des Micro | Nur DEV (40000 nicht bereitstellen) |

### BMS-Leseregister (32000–34999)

| Register | Name | Status | Ziel | Empfehlung |
|---|---|---|---|---|
| 32101 | `bms_battery_current` | FUNCTIONAL | BMS: Ursprung: BMS-Summe der Ströme der online befindlichen Packs (0,1 A vorzeichenbehaftet) | Nicht bereitstellen. Die vorhandene Summe der Pack-Ströme 34x01 beibehalten (identische Größe, unskaliert, kein Fehler). |
| 32107 | `bms_discharge_current_limit` | FUNCTIONAL | BMS: Entladeleistungsgrenze des Control = Packspannung x dieses Limit | Eines von 32107/35112 als Diagnosesensor 'BMS discharge current limit' (A, x0.1) bereitstellen, Spiegel des vorhandenen 32106. |
| 32108 | `bms_max_cell_temperature` | FUNCTIONAL | BMS: Ursprung: maximale NTC-Temperatur des BMS | Nicht hinzufügen (Duplikat von 35010). |
| 32112 | `bms_lock_active` | FUNCTIONAL | BMS: persistente BMS-Sperre, die Laden und Entladen blockiert | Als diagnostischen binary_sensor 'BMS lock active' (Klasse problem) bereitstellen. |
| 32113 | `bms_factory_mode` | FUNCTIONAL | BMS: Werksmodus-Flag des BMS (auf dem Control nur informativ) | Optional DEV/Diagnose 'BMS factory mode'; 30109 oder 32113 verwenden, nicht beide. |
| 32114 | `encode_test_status` | CONDITIONAL | Control: Control-lokaler Status eines 2-Pack-'Encode'-Tests (Pack-Aktivierung) | Nicht bereitstellen (live immer 0; Status der Testsequenz). |
| 34009 | `pack1_bms_warning_flags` | FUNCTIONAL | BMS: Ursprung: BMS-Warnflags pro Pack | Pro Pack als Diagnosesensor 'Pack N BMS warnings' bereitstellen (Bitmaske, dekodierte Attribute); DEV 'reserved_5a' umbenennen. |
| 34109 | `pack2_bms_warning_flags` | FUNCTIONAL | BMS: Ursprung: BMS-Warnflags pro Pack | Pro Pack als Diagnosesensor 'Pack N BMS warnings' bereitstellen (Bitmaske, dekodierte Attribute); DEV 'reserved_5a' umbenennen. |
| 34209 | `pack3_bms_warning_flags` | FUNCTIONAL | BMS: Ursprung: BMS-Warnflags pro Pack | Pro Pack als Diagnosesensor 'Pack N BMS warnings' bereitstellen (Bitmaske, dekodierte Attribute); DEV 'reserved_5a' umbenennen. |
| 34309 | `pack4_bms_warning_flags` | FUNCTIONAL | BMS: Ursprung: BMS-Warnflags pro Pack | Pro Pack als Diagnosesensor 'Pack N BMS warnings' bereitstellen (Bitmaske, dekodierte Attribute); DEV 'reserved_5a' umbenennen. |
| 34409 | `pack5_bms_warning_flags` | FUNCTIONAL | BMS: Ursprung: BMS-Warnflags pro Pack | Pro Pack als Diagnosesensor 'Pack N BMS warnings' bereitstellen (Bitmaske, dekodierte Attribute); DEV 'reserved_5a' umbenennen. |
| 34509 | `pack6_bms_warning_flags` | FUNCTIONAL | BMS: Ursprung: BMS-Warnflags pro Pack | Pro Pack als Diagnosesensor 'Pack N BMS warnings' bereitstellen (Bitmaske, dekodierte Attribute); DEV 'reserved_5a' umbenennen. |
| 34609 | `pack7_bms_warning_flags` | FUNCTIONAL | BMS: Ursprung: BMS-Warnflags pro Pack | Pro Pack als Diagnosesensor 'Pack N BMS warnings' bereitstellen (Bitmaske, dekodierte Attribute); DEV 'reserved_5a' umbenennen. |

### Weitere Leseregister (37021, 38000–38014)

| Register | Name | Status | Ziel | Empfehlung |
|---|---|---|---|---|
| 37021 | `mppt_field_2c_raw` | UNKNOWN | keins: Ursprung MPPT (Feldbedeutung vom MPPT definiert); Verbraucher: nur Shell/MQTT/Cloud | NICHT als energy/total_increasing bereitstellen. Nur DEV: zwei rohe u16 (37021, 37022) |
| 38000–38014 | `can40_word1 … can43_word4` | UNKNOWN | keins: keins (kein Erzeuger, kein Verbraucher) | Als DEV beibehalten (standardmäßig deaktiviert, NICHT in Poll-/Block-Reads): ein Lesezugriff in 38000-39014 lässt das Control bis zum Neustart vier CAN-Frames pro Sekunde broadcasten. Kein Sender unter ... |

### Befehlsregister 45000–45031

| Register | Name | Status | Ziel | Empfehlung |
|---|---|---|---|---|
| 45000 | `cmd_micro_factory_mode` | FUNCTIONAL | Micro: g_workMode des Micro (0x20003e04) = 12 oder 1 + Leistungsstufen-Testcode 1/9; Control g_maintModeState=2; wahrscheinlich MPPT ein/aus | NICHT bereitstellen. |
| 45001 | `cmd_inverter_eeprom_test` | FUNCTIONAL | Micro: EEPROM-0x80-Schreib-/Rücklesetest des Micro; Statusbyte 30213 | Nicht bereitstellen (höchstens DEV-Diagnose, mit 30213). |
| 45002 | `cmd_inverter_sleep_request` | CONDITIONAL | Micro: Sleep-Flag des Micro -> vorübergehender Stopp des Run-Zustandsautomaten | Nicht bereitstellen. |
| 45003 | `cmd_factory_beep` | CONDITIONAL | Micro: Buzzer des Micro (nur Werksmodus) | Nicht bereitstellen. |
| 45004 | `cmd_factory_openloop_test` | CONDITIONAL | Micro: Open-Loop-Tests der Leistungsstufe des Micro | Nie bereitstellen. |
| 45005 | `cmd_factory_overvoltage_test` | CONDITIONAL | Micro: Überspannungstest des Micro | Nie bereitstellen. |
| 45006 | `cmd_read_inverter_input_pb1` | FUNCTIONAL | Micro: Zustand des Micro-GPIO PB1, lesbar bei 30211 | Nicht bereitstellen (höchstens DEV-Diagnose mit 30211). |
| 45007 | `cmd_factory_offgrid_output_test` | CONDITIONAL | Micro: Off-Grid-Ausgangstest des Micro | Nie bereitstellen. |
| 45008 | `cmd_factory_gridtie_power_test_w` | CONDITIONAL | Micro: netzgekoppelter Leistungstest des Micro mit P W | Nie bereitstellen. |
| 45009 | `cmd_manual_backup_output_flag` | CONDITIONAL | Micro: Freigabe-Flag des Backup-/Off-Grid-Ausgangs des Micro | Nicht bereitstellen; 41200 verwenden. |
| 45010 | `cmd_manual_power_setpoint_w` | CONDITIONAL | Micro: Leistungsreferenz des Micro (+ Entladen / - Laden) | Nie bereitstellen; 42000=0x55AA + 42010/42020/42021 verwenden. |
| 45011 | `cmd_mode_record_0x1006` | UNKNOWN | keins: keins | Nicht bereitstellen. |
| 45012 | `cmd_led_lamp_test_ms` | FUNCTIONAL | GPIO: Frontpanel-LEDs (alle an, active-low) für v ms | Nicht bereitstellen (höchstens DEV-Button 'LED test'). |
| 45013–45019 | `reserved_13 … reserved_19` | NO_OP | keins: keins | Nicht bereitstellen, nie abfragen. |
| 45020 | `cmd_class2_unit1_on_and_power_w` | UNKNOWN | keins: Klasse-2-Knoten-1-Einheit (offenbar MPPT/DC-Lader, nach der g_mppt*-Benennung) - auf dem Venus D nicht vorhanden | Nicht bereitstellen. |
| 45021–45024 | `cmd_class2_unit1_on_off … cmd_class2_unit1_reg_0xFF` | UNKNOWN | keins: keins auf dem D | Nicht bereitstellen. |
| 45025 | `dbg_gpio_pb10_pack_encode_line` | CONDITIONAL | GPIO: Control PB10 = Encode-/Handshake-Leitung der Batteriepacks (abgeleitet PB10 -> erstes BMS PB13) | Nicht bereitstellen (höchstens DEV). |
| 45026 | `dbg_gpio_pe15_button_led` | CONDITIONAL | GPIO: Anzeige-LED des Power-Tasters (abgeleitet) | Nicht bereitstellen. |
| 45027 | `cmd_bms_master_reboot` | FUNCTIONAL | BMS: BMS-Master-Pack: System-Reset (+ optional EEPROM 0x202 = 0xFF = beim nächsten Boot selbst Master beanspruchen) | Nie bereitstellen. |
| 45028 | `cmd_bms_factory_reset` | FUNCTIONAL | BMS: alle BMS-Packs: Werksreset der Schutz-/Konfigurationsparameter + Neustart | Nie bereitstellen. |
| 45029 | `cmd_battery_pack_encode_start` | CONDITIONAL | BMS: BMS-Pack-Adresskodierung / Master-Beanspruchung | Nicht als Steuerung bereitstellen (höchstens DEV-Button). Es ist das Äquivalent zur 'Battery Pack Recovery' der App (Pack-Adresskodierung), kein Wiederherstellungsladen; 32114 zeigt den ... |
| 45030–45031 | `cmd_bms_syscmd_0x0B_0x0C … cmd_bms_restart_handshake` | FUNCTIONLESS | keins: keins auf dem D | Nicht bereitstellen. |

### Verschiedene Schreib-/Diagnoseregister (45603–47400)

| Register | Name | Status | Ziel | Empfehlung |
|---|---|---|---|---|
| 45603 | `inverter_calibration_value_a` | FUNCTIONAL | Micro: EEPROM 0x950 des Micro (Delta des Ergebniszählers der Batteriekalibrierung) | DEV-/Diagnosesensor, bei Bedarf oder selten lesen (nicht im schnellen Poll-Block), BCD dekodieren |
| 45604 | `inverter_calibration_value_b` | FUNCTIONAL | Micro: EEPROM 0x952 des Micro (Delta des Ergebniszählers der Batteriekalibrierung) | DEV-/Diagnosesensor, bei Bedarf oder selten lesen (nicht im schnellen Poll-Block), BCD dekodieren |
| 45605 | `ethernet_chip_version` | FUNCTIONAL | CH395: Versionsbyte des CH395-Ethernet-Controllers | DEV-/Diagnosesensor, einmal beim Start lesen |
| 46000 | `ota_debug_command` | FUNCTIONAL | EEPROM: RS485-Task-Modus des Control (EEPROM 0x900) / YModem-Firmware-Update-Modus des Control | NICHT bereitstellen; nie lesen oder schreiben |
| 46500–46544 | `can_sched_reg_00_u16 … can_sched_reg_14_u32_lo` | FUNCTIONLESS | keins: keins (nur RAM-/EEPROM-Speicherung) | NICHT bereitstellen |
| 47001 | `can_cat0x18_command` | FUNCTIONLESS | keins: keins auf dem Venus D | NICHT bereitstellen |
| 47400 | `debug_test_register` | NO_OP | keins: keins | nicht als Entität bereitstellen; als interne Verbindungs-/Handshake-Sonde verwendbar |

## 4. Empfehlungen für die Marstek Modbus Suite (Venus D)

- **Entfernen:** Button `factory_reset` (41001, tut auf dem D nichts) (die 15 DEV-Einträge 38000–38014 bleiben als DEV: standardmäßig deaktiviert und vom Polling ausgeschlossen, weil ihr Lesen einen CAN-Broadcast auslöst und ihre Gegenstelle unbekannt ist).
- **Beibehalten:** `reset_device` (41000 = `0x55AA`): ein echter Neustart von Control und Micro.
- **Als reguläre Register hinzufügen (nur lesend):** BMS-Entladestromgrenze 32107 (0,1 A, neben 32106); BMS-Sperre aktiv 32112 (Binärsensor); BMS-Warnflags pro Pack 34009 / 34109 / … (Bitmaske); RS485-Slave-Adresse 37000 (Diagnose).
- **Nur DEV / Diagnose:** 30028 / 30029 / 30036 (MPPT), 37021 (roh), 30211, 37001–37003, 32113, 45603–45605 (das Lesen von 45603/45604 blockiert den Modbus-Task bis zu 200 Ticks).
- **Nie bereitstellen:** 40000, 41000 = `0xAA11`, 45000–45031, 46000, 46500–46544, 47001, 47400 (gefährlich oder wirkungslos).
- **Optional:** ein Write-only-Dienst `set_wifi` für 41500–41631 (SSID 1–31 Zeichen, Passwort leer oder 8–32, druckbares ASCII ohne `,` und `"`; zuerst die SSID mit einem 0x00-Terminator schreiben, dann das Passwort bis zum Terminator-Register).

## 5. Offene Fragen (benötigen Ghidra, einen Schaltplan oder MPPT-Firmware)

- Physische Verdrahtung des USART2-Modbus-Slaves des Micro (interner BMS-Bus?)
- Provisioniert das Gerät die TLS-Zertifikate / MQTT-Identität des FC41D nach 0xAA11 automatisch neu?
- Alte WLAN-Zugangsdaten bei EEPROM 0x400/0x420 werden vom Defaults-Zweig nicht neu geschrieben; tauchen sie bei einem späteren Boot wieder auf?
- Begrenzt Ctrl_SetPowerReferenceClamped auf die per CAN gesetzten Micro-Limits (z. B. 800 W Entlade-Default)?
- Tick-Einheit von g_appTickMs (Sekunden?)
- FC41D-SSID-Vorverarbeitung FUN_0003c2b8/FUN_0003c300 (Nicht-ASCII-/UTF-8-SSIDs) nicht analysiert
- Ob QueryConnectionState(2) Erfolg melden kann, während das Modul noch mit dem ALTEN AP assoziiert ist (Race zwischen QSTAAPINFODEF und dem Disconnect-URC) - würde ungetestete Zugangsdaten persistieren
- FC41D-32-Byte-Flash-Item für ID 0x11111105: bestätigen, dass Passwörter > 32 Zeichen beim Modulstart abgeschnitten werden
- Ist der Klasse-2-Empfänger die MPPT-Platine, d. h. stoppt 45000 das PV-Laden?
- Leistungsreferenz nach dem 0x55BB-Neustart (setzt stop ctx+0x54 zurück?)
- Genaue Dauer und ob ein laufender Wechselrichter gestoppt oder nur ein Start blockiert wird
- Womit PB1 verdrahtet ist
- Vorzeichenkonvention von Testcode 3
- Empfänger auf anderen Produkten (E2/A-BMS)?

Verifikationshinweis: Zwei unabhängige Analysten haben diese Ergebnisse aus dem Code neu hergeleitet; Korrekturen (z. B. 41000-`0xAA11`-Adresse, EEPROM-Schreibbedingung von 45027, Verhalten von 45029/45031, BMS-Warnschwellen) sind bereits in der CSV-Spalte `befund_D` enthalten.
