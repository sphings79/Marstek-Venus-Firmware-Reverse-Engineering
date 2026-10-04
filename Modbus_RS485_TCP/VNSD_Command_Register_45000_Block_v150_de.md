# Command-Register-Block 45000–46000 (Control FW v150)

> 🇬🇧 English version: [VNSD_Command_Register_45000_Block_v150.md](VNSD_Command_Register_45000_Block_v150.md)

> **Firmware:** `VNSD-0_app_0150_0805_115146.bin` (v150) · ARM Cortex-M, FreeRTOS
> **Quelle:** Statische Analyse `Write_Handler` (`FUN_08051d14`, 0x08051d14) in Ghidra
> **Stand:** 2026-08-14
> **Adressierung:** direkt (PDU = Register-Nummer, kein Offset) · Write via **FC06/FC10**

---

## 1. Was dieser Block ist

Die Register **45000–46000** sind **kein** normaler Steuer-Registersatz wie
42010 (`force_mode`) oder 43000 (`work_mode`), sondern ein **Service-/Engineering-
Kommando-Interface**. Fast jedes Kommando übersetzt einen Modbus-Write in eine
**interne CAN-Bus-Nachricht an den Wechselrichter-MCU** — gebaut über
`CAN_Build_Arbitration_ID(funccode, …)` + `Register_WriteValue()`. Einige wenige
Kommandos schalten stattdessen **lokale Hardware** (GPIO-Pins, Relais, CH395-
Netzwerkchip, BLE) oder lösen **OTA/EEPROM** aus.

Eigenschaften des Blocks:

- **Write-only** (Lesen → Modbus-Exception `2`), außer den drei Read-Registern
  45603/45604/45605.
- Ungültige Werte → Exception `3`.
- Viele Funktionen **überschneiden sich** mit den offiziellen Steuerregistern
  (42010/42020/43000) — nur auf niedrigerer, direkterer Ebene.
- **Mehrere sind gefährlich** (OTA, Relais, GPIO, SMR-ADC) und ohne
  Hardware-Verständnis nicht zu benutzen.

### CAN-Funktionscodes (beobachtet)

| CAN-Func | Bedeutung | genutzt von |
|---|---|---|
| 0x01 | Inverter On/Off (Klasse 2: kein Empfänger an der Venus D) | 45021, (45020, 45000) |
| 0x06 | Flag-Register 0x06 | 45002 |
| 0x50 | Feature-Enable | 45000 |
| 0x52 | WorkMode (1 Byte) | 45004 |
| 0x55 | Config-Param 55 | 45007 |
| 0x56 | Leistungs-**Sollwert** (u32) | 45008 |
| 0x57 | Flag 0x57 (Ein/Aus) | 45009 |
| 0x58 | Manueller Leistungs-**Sollwert** (u32, nur Modus 1; kein Limit) | 45010 |
| 0xC1 | Zähler-Reset (Frame ohne Empfänger; 45001 ist kein BMS-Reset, siehe Abschnitt 8) | 45001 |
| 0xFB | Reg 0xFB | 45022 |
| 0xFE | Reg 0xFE | 45023 |

---

## 2. ⚠️ Adress-Drift 149.2 → v150

Die frühere `Modbus_RS485_TCP/Scan_Logs/Write_Handler_Register_Map.csv` wurde aus **FW 149.2** abgeleitet.
Die dort genannten `FUN_`-Ziel­adressen (z. B. `FUN_08005f94`, `FUN_08005ccc`,
`FUN_08002e10`) sind in v150 **verschoben** und zeigen dort auf andere Funktionen.
Der `Write_Handler` selbst hat sich geändert (Ghidra-Similarity 149.2→150 = 0.629).

**Register-Nummern und Grundbedeutung bleiben gültig** — die konkreten Funktions-
namen/Adressen unten stammen aus v150. Die CSV wurde entsprechend korrigiert.

Zusätzlich: Der **Kommentar-Header im Binär-`Write_Handler`** benennt die Read-
Register falsch (nennt „45539–45541" und „45597/0xB28D"). Der tatsächliche Code
adressiert **45603/45604 (0xB223/0xB224)** und **45605 (0xB225)** — passend zur
Register-Map-CSV, nicht zum Kommentar.

---

## 3. Kommando-Tabelle (v150-verifiziert)

| Reg | Name | Werte | Wirkung | Backend |
|---|---|---|---|---|
| 45000 | cmd_micro_factory_mode | `0x55EE` / `0x55BB` (sonst Err 3) | `0x55EE` → Micro-`g_workMode` = 12 (Werksmodus), `0x55BB` → Modus 1 (manuell); beide senden zusätzlich Klasse-2-„Aus" (kein Empfänger an der D); Statusbyte=2 (korrigiert, siehe Abschnitt 8) | CAN 0x50 |
| 45001 | cmd_inverter_eeprom_test | beliebig | Micro-EEPROM-Schreib-/Rücklese-Selbsttest (Ergebnis in 30213); **kein** BMS-Reset (korrigiert, siehe Abschnitt 8) | CAN 0xC1 |
| 45002 | cmd_function_1 | beliebig | `Inverter_Set_Flag_Reg_0x06(1)` | CAN 0x06 |
| 45003 | cmd_function_2 | beliebig | `Config_Param51_Reset()` | CAN |
| 45004 | **cmd_mode_select** | `1`/`2`/`3` | `Config_WorkMode_Set(1/2/3)` — Wert 1:1 als Modus | CAN 0x52 (1 B) |
| 45005 | cmd_function_3 | beliebig | `Config_Param53_Activate()` | CAN |
| 45006 | cmd_function_4 | beliebig | `Config_PostWriteCommit()` | CAN |
| 45007 | cmd_function_5 | beliebig | `Config_Param55_Set(1)` | CAN 0x55 |
| 45008 | **cmd_set_power_a** | 1–2500 W; >32767 = neg. (Wert−65535); 0/2501–32767 ignoriert | `Config_PowerSetpoint_Write()` — **Sollwert** ±2500 W | CAN 0x56 (u32) |
| 45009 | cmd_enable_disable_a | `0`/`1` | `Inverter_Set_Flag_Reg_0x57(0/1)` | CAN 0x57 |
| 45010 | **cmd_manual_power_setpoint_w** | wie 45008 | `Inverter_Set_PowerLimit_Reg_0x58()` — der echte manuelle **Leistungs-Sollwert** ±2500 W (+ Entladen / − Laden), wirksam **nur im manuellen Modus 1** (nach 45000=`0x55BB`), sonst quittiert und ignoriert; kein Limit (korrigiert, siehe Abschnitt 8) | CAN 0x58 (u32) |
| 45011 | **cmd_mode_select_ext** | 1–9 (>9 Err 3) | `WorkMode_Register_Write()` mit Remap **1→1, 2→2, 3→4, 4→0, 5→5, 6–9→0** | 8-B Reg-Write |
| 45012 | cmd_led_lamp_test_ms | Zeit in ms | Lampentest der Frontplatten-LEDs (alle an, active-low) für die angegebene Zeit in ms; keine Leistungswirkung (korrigiert, siehe Abschnitt 8) | lokale HW (GPIO) |
| 45020 | cmd_class2_unit1_on_and_power_w | 1–3999 (sonst keine Aktion) | Klasse-2-Frames (Einheit ein + Leistungsbefehl bis 3999 W); die Micro der Venus D filtert Klasse 2, kein Empfänger an der D (korrigiert, siehe Abschnitt 8) | CAN Klasse 2 |
| 45021 | cmd_class2_unit1_on_off | `0`/`1` | Klasse-2-Ein/Aus-Frame, kein Empfänger an der D; „SMR" in der Firmware ist ein fremder Begriff (korrigiert, siehe Abschnitt 8) | CAN 0x01 |
| 45022 | cmd_class2_unit1_reg_0xFB | beliebig | `Inverter_Set_Reg_0xFB(1)` — Klasse-2-Frame, kein Empfänger an der D | CAN 0xFB |
| 45023 | cmd_class2_unit1_reg_0xFE | beliebig | `Inverter_Set_Reg_0xFE(1)` — Klasse-2-Frame, kein Empfänger an der D | CAN 0xFE |
| 45024 | cmd_class2_unit1_reg_0xFF | beliebig | `Inverter_Set_Reg_0xFF(1)` — Klasse-2-Frame, kein Empfänger an der D (frühere Lesart `Inverter_RS485_Cmd_Reset` / RS485 ist falsch) | CAN 0xFF |
| 45025 | dbg_gpio_pb10_pack_encode_line | `1`=setzen / `0`=löschen | GPIO-Bit **0x400** (PB10); im Normalbetrieb nach ≤10 ms wieder überschrieben, bleibt nur im Testmodus / bei Tastensequenzen | lokale HW |
| 45026 | dbg_gpio_pe15_button_led | `1`/`0` (**invertiert!**) | GPIO-Bit **0x8000** (PE15), Logik gegenüber 45025 vertauscht; im Leerlauf alle 10 ms neu geschrieben, kosmetisch | lokale HW |
| 45027 | cmd_bms_master_reboot | beliebig | `CAN_SendTriggerCategory0x0A(1)` — Klasse-3-Frame → **sofortiger Neustart des BMS-Master-Packs** (kein WLAN-/Netzwerk-Reset, nicht harmlos; korrigiert, siehe Abschnitt 8) | CAN Klasse 3 → BMS |
| 45028 | cmd_bms_factory_reset | beliebig | BMS-**Werksreset + Neustart aller Packs** (Broadcast-ID; kein Relais; korrigiert, siehe Abschnitt 8) | CAN → BMS |
| 45029 | cmd_battery_pack_encode_start | `1`/`0` | startet die BMS-Pack-Adresscodierung / Master-Claim (`g_encodeFlag`); Status 2 (Erfolg) nur mit genau 2 Packs online (korrigiert, siehe Abschnitt 8) | CAN → BMS |
| 45030 | cmd_bms_syscmd_0x0B_0x0C | `1`=ein / `0`=aus | `Config_Apply_SingleReg(0xB)` bzw. `(0xC)` — **funktionslos an der D** (kein Empfänger behandelt 0x0B/0x0C; kein BLE-Enable; korrigiert, siehe Abschnitt 8) | keiner |
| 45031 | cmd_bms_restart_handshake | nur `1` | Ack-Bedingung mit BMS 117.7/118 unmöglich: **funktionslos an der D**; einziger Effekt ist ein ~5 s Modbus-Task-Stall + Exception 3 (korrigiert, siehe Abschnitt 8) | keiner |
| 45603 | adc_version_bcd_ch1 | **read-only** | Queues leeren, `Inverter_Clear_Reg_0x13()`, 200 ms warten, **BCD**-Version Kanal 1 (`DAT_08052de8`) | Queue/ADC |
| 45604 | adc_version_bcd_ch2 | **read-only** | wie 45603, Kanal 2 (`DAT_08052dec`) | Queue/ADC |
| 45605 | status_query | **read-only** | `CH395_SPI_ReadByte()` — Statusbyte vom **CH395-Netzwerkchip** über SPI | SPI |
| 46000 | **ota_command** | `0x5100`=EEPROM-Flag (Addr 0x900); `0x4D2`(1234)=Mode 1; `0x929`(2345)=Mode 4; `0xD80`(3456)=Mode 3; `0x11D7`(4567)=Mode 2 | OTA-Mode-Flag `DAT_08052df4` bzw. EEPROM-Flag `DAT_08052df0` | OTA/EEPROM |

> Register **45013–45019** sind reserviert und geben Exception `2` (illegal data address) zurück.

---

## 4. Die drei „Set-Power"-Varianten

Optisch identisch (alle 1–2500 W, signed), aber **drei verschiedene Backends** (⚠️ die Semantik-Spalte unten ist durch Abschnitt 8 überholt: nur 45010 ist ein echter Leistungs-Sollwert im manuellen Modus; 45008 ist ein Werkstest in Modus 12; 45012 ist ein LED-Lampentest):

| Reg | Funktion | Pfad | Semantik |
|---|---|---|---|
| 45008 | `Config_PowerSetpoint_Write` | CAN-Func **0x56** | Leistungs-**Sollwert** an Inverter |
| 45010 | `Inverter_Set_PowerLimit_Reg_0x58` | CAN-Func **0x58** | Leistungs-**Limit** |
| 45012 | `Inverter_Power_Setpoint_Apply` | **I2C**-Registersync | separater I2C-Pfad, Default-Fallback 2000 W |

Das erklärt die scheinbare Redundanz: Sollwert vs. Limit vs. I2C-gekoppelter Pfad.

---

## 5. Modus-Auswahl: 45004 vs. 45011

- **45004** (`cmd_mode_select`): sendet Wert 1/2/3 direkt per CAN-0x52. Nur 1–3 gültig.
- **45011** (`cmd_mode_select_ext`): akzeptiert 1–9, **mappt aber um** und schreibt
  einen 8-Byte-WorkMode-Record. Das interne Modus-Codeset ist **nicht** deckungs-
  gleich mit der Eingabe:

  | Eingabe | 1 | 2 | 3 | 4 | 5 | 6–9 | >9 |
  |---|---|---|---|---|---|---|---|
  | intern | 1 | 2 | **4** | **0** | 5 | 0 | Error 3 |

---

## 6. Gefahren-Hinweis

Diese Register sind Werks-/Debug-Funktionen ohne Missbrauchsschutz:

- **46000 (OTA)** — kann Update-Prozess auslösen.
- **45028 (BMS-Werksreset + Neustart aller Packs, siehe Abschnitt 7) und 45027 (sofortiger Neustart des BMS-Master-Packs)**; 45031 blockiert den Modbus-Task ~5 s (sonst funktionslos). 45001 ist **kein** BMS-Reset (Abschnitt 8).
- **45020 (Klasse-2-Einheit ein + Leistungsbefehl)** — kein Empfänger an der Venus D (Abschnitt 8).
- **45025/45026 (GPIO)** — direkter Pin-Eingriff (alle 10 ms neu geschrieben); 45023/45024 sind Klasse-2-Frames ohne Empfänger an der D; 45027 ist *kein* WLAN-Reset, sondern ein BMS-Master-Neustart.

Auf produktiven Geräten nur die klar verstandenen Register (Mode/Power/BLE/Version)
verwenden — deren Funktion ist ohnehin weitgehend über die offiziellen Register
42010/42020/43000 abgedeckt.

---

## 7. Korrekturen aus der v151-Codeanalyse (2026-10-01)

> ⚠️ Teilweise überholt durch Abschnitt 8 (Registeraufloesung vom 2026-10-01); wo sie abweichen, gelten Abschnitt 8 und die CSV.

Neu geprüft am `Write_Handler` von **v151** (`0x080570e0`), ausgehend von der Registernummer. Wo die v150-Lesart oben abweicht, zeigt der
v151-Code Folgendes (der v150-Handler wurde für diese Register nicht erneut gelesen; `VNSD_Register_Map_Final_generated.csv` trägt die neuen
Namen und die alten als `frueher: …`):

| Reg | Früherer Name / Lesart | v151-Code |
|---|---|---|
| 45000 | `cmd_system_control` | Testmodus: `0x55EE`/`0x55BB` schreiben CAN-Reg 0x50 (Micro-`g_workMode` 12 bzw. 1) und senden Klasse-2-„Aus"; `g_maintModeState = 2`. Name jetzt `cmd_micro_factory_mode`. Lesen → Exception 3 |
| 45020 | `cmd_smr_relay_adc` (SMR/ADC) | Klasse-2-**Einheit EIN + Leistungsbefehl**: Wert 1…3999 W → `Inverter_Set_OnOff_Reg_0x01(1,1)`, dann `Inverter_CAN_PowerCmd_Send(1,1,Wert, …)` mit Ladespannung und Temperaturen als Zusatzfelder; **kein Empfänger an der Venus D** (Klasse 2 wird von der Micro gefiltert). Name jetzt `cmd_class2_unit1_on_and_power_w` |
| 45021 | `cmd_smr_enable` | einfaches Klasse-2-Ein/Aus (`Inverter_Set_OnOff_Reg_0x01`); „SMR" ist falsch (fremder Firmware-Begriff); kein Empfänger an der D. Name jetzt `cmd_class2_unit1_on_off` |
| 45023 / 45024 | `cmd_gpio_debug_1/2` (GPIO) | CAN-Befehle an Wechselrichter-Register **0xFE** und **0xFF** (`Inverter_Set_Reg_0xFE/0xFF(1)`), kein GPIO-Zugriff |
| 45025 / 45026 | GPIO 0x400 / 0x8000 | weiterhin GPIO, aber die Pins/Ports sind **PB10 (0x400)** und **PE15 (0x8000)**; die Polarität ist zwischen beiden vertauscht (45025: 1 = Bit löschen, 45026: 1 = Bit setzen) |
| 45027 | `cmd_wifi_reset` | `CAN_SendTriggerCategory0x0A(1)` – ein Klasse-3-CAN-Trigger, kein WLAN-Reset; startet das BMS-Master-Pack neu. Name jetzt `cmd_bms_master_reboot` |
| 45029 | `cmd_debug_flag` | `g_encodeFlag`: Statusbyte der Schlüssel-/Pairing-„Encode"-Sequenz (0 idle, 1 angefordert, 2 fertig bei BMS-Online-Maske == 3, 3 Timeout); auch über 30110/32114 lesbar. Name jetzt `cmd_battery_pack_encode_start` |
| 45603 / 45604 | `adc_version_bcd_ch1/2` (ADC) | Antwort einer CAN-Anfrage (Reg 0x13), als BCD über zwei Queues geliefert (`g_regGetOldQueueLo/Hi`) |
| 45605 | `status_query` | **CH395-Chipversion** (`CH395CMDGetVer`) |

Unverändert/bestätigt: 45002–45012 (CAN-Register 0x06, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58 und der 8-Byte-Modus-Record).

⚠️ **45028 ist kein Relais.** Es sendet Byte 2 auf CAN-ID `0x100101AA`; BMS 118 wertet das als **Werksreset + Neustart** (`APP_Factory_Reset_And_Reboot`). Jeder geschriebene Wert löst das aus, ohne Authentifizierung. **45001** löst diesen Reset **nicht** aus (die frühere Fall-through-Behauptung war falsch, siehe Abschnitt 8). Der Name von 45028 ist jetzt `cmd_bms_factory_reset`.
**45031** blockiert den Modbus-Task etwa 5 s und liefert Exception 3 (funktionslos an der D). **45030** (BLE-Enable, `Config_Apply_SingleReg(0x0B/0x0C)`) gibt es in allen Versionen v147–v151; im v151-Handler ist es nicht neu, und BMS 118 behandelt die Bytes 0x0B/0x0C nicht.
Neu in v151: Register **47001** (`0xB799`, `can_cat0x18_command`): beliebiges u16, das Low-Byte wird als 8-Byte-CAN-Frame gesendet (ID `0x00014118`, Funktion 0x18, Node 1), kein EEPROM; dieselbe Senke wie die OTA-Zielwahl in 46000 und MQTT-Debug 3/4. In Micro 116 und BMS 118 wurde kein Empfänger für Funktion 0x18 gefunden. Lesen liefert Exception 2.
In v147–v150 verhalten sich die Zweige 45020/45023/45024/45027/45029 wie in v151 (durch Emulation des jeweiligen Handlers geprüft).

**Frame-Klassen (ergänzt 2026-10-01).** Die Control-Firmware sendet ihre CAN-Frames in zwei Hauptklassen, unterschieden durch das Ziel-Nibble (CAN-ID-Bits 12–15) aus `CAN_Build_Arbitration_ID`: **Klasse 4** (Leistungsvorgabe/-limit, Arbeitsmodus, 0x50/0x52–0x58, 0x60, 0x18, Schedule 0x30+idx, EEPROM-nahe Befehle, `Config_Mode_Apply` = Backup-Modus) und **Klasse 2** (`Inverter_Set_OnOff_Reg_0x01`, `Inverter_CAN_PowerCmd_Send`, 0x04, 0xFB, 0xFE, 0xFF, Arbeitsmodus-Reg 0x06, Datum/Uhrzeit-Broadcast 0xCC); der Relais-Trigger 0x0A (45027) nutzt Klasse 3, die die Micro filtert, das BMS aber annimmt. Die **Micro-Firmware von Venus D und Venus E3** (115/116, 119) stellt ihre CAN-Filter auf `(ID & 0xFF00) == 0x4000/0x4100` plus BMS-Bereich, **Klasse-2- und Klasse-3-Frames erreichen ihren Dispatcher also nie**. Folge für die D: 45020/45021 (Wechselrichter Ein/Aus + Leistungsbefehl), 45022–45024 und der Ein/Aus-Teil von 45000 bleiben für den Wechselrichter wirkungslos; die Venus-A-Micro (1211) hat einen Handler für 0xFE. Klasse-4-Befehle, die die D-Micro erreichen: 0x50 (45000: setzt Micro-`g_workMode` auf 12 bzw. 1 plus einen Leistungsstufen-Testcode, siehe Abschnitt 8), 0x52–0x58 (Test-/Engineering-Zustände; 0x58 = manueller Leistungs-Sollwert von 45010, nur Modus 1), 0x60, `Config_Mode_Apply` (41200, gespeichertes Backup-Modus-Flag bei EEPROM 0x300).

---

## 8. Korrekturen aus der Registeraufloesung vom 2026-10-01

Vollständige Analyse jedes Registers: [VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md](VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md). Die Formulierungen folgen `befund_D` in `VNSD_Register_Map_Final_generated.csv`. Wo die Abschnitte 1–7 abweichen, gilt dieser Abschnitt.

| Reg | Korrektur |
|---|---|
| 45000 | `0x55EE` schreibt **Micro-`g_workMode` = 12** (Werksmodus), `0x55BB` wählt **Modus 1** (manuell); in beiden Modi werden EMS-Leistungssollwerte ignoriert. Beide Zweige senden zusätzlich Klasse-2-„Aus". Stoppt die Ausgabe bis zu einem Control-Neustart (ein Control-Boot sendet CAN-Kommando 5, das die Micro neu startet; deshalb stellt 41000 den Normalzustand wieder her). Name `cmd_micro_factory_mode`. Nie pollen (Lesen → Exception 3, TCP-Verbindung hängt). |
| 45001 | **Micro-EEPROM-Schreib-/Rücklese-Selbsttest** (Ergebnis in 30213), pro Aufruf ein Micro-EEPROM-Byte geschrieben. **KEIN BMS-Reset**: der Reset-Flag-Frame (`0x100701AA` in v147–149.2, `0x180701AA` in v150/151) hat keinen Empfänger, und im Code gibt es keinen Fall-through zum BMS-Reset. Name `cmd_inverter_eeprom_test`. |
| 45004 / 45008 | Werkstests (Open-Loop-Test, Netzkopplungs-Leistungstest bei P W); nur nach 45000=`0x55EE` (Modus 12), sonst quittiert und ignoriert. |
| 45010 | Der **echte manuelle Leistungs-Sollwert** (+ Entladen / − Laden), **nur im manuellen Modus 1** (nach 45000=`0x55BB`); im Normalmodus 0 quittiert und ignoriert. Kein „Limit". Negative Werte kommen 1 W zu knapp an; 0 W lässt sich nicht vorgeben. Umgeht SoC-Abschaltung, EPS-Reserve, Fehlermaske und Lade-/Entladegrenzen; bleibt bis zum Control-Neustart bestehen. Name `cmd_manual_power_setpoint_w`. |
| 45011 | Funktionslos an der D (kein Empfänger). |
| 45012 | **Lampentest der Frontplatten-LEDs** (alle an, active-low, für v ms), identisch mit dem Shell-Befehl `led_continue_ctrl`; **kein Leistungs-Sollwert**, kein I2C-Leistungspfad. Name `cmd_led_lamp_test_ms`. |
| 45013–45019 | Explizites Exception 2 in allen fünf Control-Versionen; kein anderer Code referenziert sie. Der eigene 45010–45019-Block der Micro ist ein getrennter Registerraum am eigenen RS485-Port der Micro und wird von der Control nicht weitergereicht. (Der TCP-Exception-Frame trägt eine falsche MBAP-Länge, siehe `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md`.) |
| 45020–45024 | **Klasse-2-Frames** (`cmd_class2_unit1_*`; Register 0xFB / 0xFE / 0xFF, Ein/Aus, Ein + Leistung bis 3999 W). Die Micro der Venus D (115/116) filtert Klasse 2, und das BMS hat keinen Klasse-2-Pfad, also **gibt es an der D keinen Empfänger**. Der Empfänger ist eine separate Einheit, höchstwahrscheinlich ein MPPT-/PV-Controller (nach der `g_mppt*`-Benennung); dessen Firmware liegt nicht vor, „kein Empfänger" ist also nur für die vorhandene Firmware belegt. „SMR" hat nichts damit zu tun (`SMR_TIC_SelfUse_PowerController` / HW-Tag `SMR-`). 45024 ist kein RS485-Befehl. Status FUNCTIONLESS. |
| 45025 / 45026 | GPIO PB10 / PE15 (PB10 vermutlich Pack-Encode-/Handshake-Leitung, PE15 vermutlich Power-Taster-LED). Im Normalbetrieb **alle 10 ms neu geschrieben**, praktisch ohne Wirkung; ein Schreibzugriff bleibt nur im Testmodus (nach 45000) oder in Tasten-/Latch-Zuständen. Status CONDITIONAL. |
| 45027 | **NICHT harmlos / kein WLAN-Reset**: Klasse-3-Frame (die BMS-Befehlsklasse; nur die Micro filtert sie) → **sofortiger Neustart des BMS-Master-Packs** (Node 1 = Slot 1; Packs mit anderem Slot ignorieren ihn). Ein Pack, das zur Laufzeit Master wurde, bekommt zusätzlich EEPROM 0x202 = 0xFF (Self-Claim als Master). Die Batterie trennt sich während des Resets. Name `cmd_bms_master_reboot`. Die Aussage „keine Wirkung an der D" in älteren Dokumenten ist falsch. |
| 45028 | **BMS-Werksreset + Neustart aller Packs** (Broadcast-ID, von BMS 117.7 und 118 behandelt): setzt den BMS-Parameterblock auf Defaults; der Master kann unregistriert booten (EEPROM 0x202 = 1). Kein Relais. |
| 45029 | Pack-Adresscodierung / Master-Claim (`g_encodeFlag`; schaltet den BMS-Pack-Select-Pin PB12). Läuft bei `g_keyEventType == 0` und `g_maintModeState != 2`; **Erfolg (Status 2) nur mit genau 2 Packs online** (Maske 3); 1 Pack oder 3+ Packs enden mit Status 3 (Timeout). Name `cmd_battery_pack_encode_start`. |
| 45030 / 45031 | **Funktionslos an der D.** 45030: kein Empfänger behandelt die Bytes 0x0B/0x0C in BMS 117.7/118, keine lokale BLE-/FC41D-Aktion („BLE-Enable" ist unbegründet). 45031: Ack-Bedingung unmöglich, 0xA0 unbehandelt; ein Schreibzugriff blockiert den Modbus-Task ~5 s und liefert Exception 3, ein Lesezugriff sofort Exception 2. |

## 9. Referenzen

- `Write_Handler` @ `0x08051d14` (v150) — Dispatch für gesamten 40000–47400-Bereich
- Dispatch-Selektor 45000-Block: `iVar11 = param_2 − 45000` (switch), 45000 selbst über `uVar8 == 0x76a`
- `Modbus_RS485_TCP/Scan_Logs/Write_Handler_Register_Map.csv` — vollständige Write-Register-Map (v150-korrigiert)
- `Control_FW/VNSD_Control_FW_Analyse_app_0150_0805_115146_de.md` — v150-Gesamtanalyse
