# Diff-Tabelle BMS v110.5 (1105) → v110 (Venus A)

> 🇬🇧 English version: [VNSA_BMS_Diff_v1105_v110.md](VNSA_BMS_Diff_v1105_v110.md)

**Stand:** 2026-10-11. Beide Images mit Basis 0x08002800 in Ghidra, vollständig benannt (1105: 698 Funktionen, 110: 697).
Adressen: 1105 → 110. **Hauptdokumentation:** [VNSA_BMS_FW_v110_Vollanalyse_de.md](VNSA_BMS_FW_v110_Vollanalyse_de.md),
Vorgänger-Analyse [VNSA_BMS_FW_v1105_Vollanalyse_de.md](VNSA_BMS_FW_v1105_Vollanalyse_de.md).

Funktionspaare über identische Namen (697), Befehl für Befehl verglichen, Aufrufziele und Globals über den Namen,
referenzierte Strings und Immediate-Werte; Literal-Pool-Werte separat auf den Roh-Images (Pointer/BL maskiert):
**15 Funktionen im Verhalten oder in Konstanten geändert**, **5 nur in verschobenen Tabellen-/Literaladressen**,
**0 nur in 110**, **1 nur in 1105** (`SoftFloat_Double_OrderedCompare_Rev`).

## Welche Version ist älter?

| Merkmal | 1105 | 110 |
|---|---|---|
| Versionsstring (Shell-Banner) | `Version: 110.5` | `Version: 110.0` |
| Build-Datum | May 11 2026 09:47:24 | May 20 2026 13:59:48 |
| Cloud-OTA-Eintrag | Edition 3316, Upload-Ordner 20260512 | Edition 3430, Upload-Ordner 20260520 |
| FW-Kennung im Pack-Datensatz (`g_pack0FwVersionTag`, Modbus 30204/34010/37012) | 0x451 = 1105 | 0x44C = 1100 |
| Codebasis | A 109 + PV-Rettung/Niederspannungs-Änderungen | **A 109 (26.12.2025) + 40-A-Grenze** |

**Ergebnis:** 110 ist das *neuere* Release (neun Tage später gebaut, höhere OTA-Edition), aber **kein** Nachfolger von
1105. 110 ist byte-identisch mit A BMS 109 (`109_bms_VNSA-0_20251226103854b52c66481.bin`, Build 26.12.2025) bis auf die
40-A-Grenze, die Versionskonstanten und den Build-/Versionsstring (insgesamt 334 abweichende Bytes, siehe unten). 1105
(„110.5") ist ein paralleler Beta-Zweig auf Basis von 109; keine seiner eigenen Änderungen wurde in 110 übernommen. Die
Zahl 1105 ist „110.5" × 10, keine Version oberhalb von 110.

110 gegen 109 (Rohbyte-Vergleich, gleiche Länge, gleiches Layout):

| Stelle | 109 | 110 |
|---|---|---|
| `Master_Build_ChargeCurrentLimit` / `Master_Build_DischargeCurrentLimit` (4 Immediates) | 0x1F4 = 500 (50,0 A) | 0x190 = 400 (40,0 A) |
| FW-Kennung in `PerPack_Struct_Builder` (`movs` → `movw`, Funktion 2 Bytes länger, nachfolgender Code verschoben) | 0x6D = 109 | 0x44C = 1100 |
| Konfigurations-Versionskennung `g_cfgVersionTag` (`BMS_Config_Params_Load_Or_Default`, `_Factory_Reset`) | `'m'` (0x6D) | `'n'` (0x6E) |
| RS485-Versionsbyte (`RS485_Dispatcher` Op 0x05, `RS485_Pack_Telemetry_Response`), `g_sh36735InitConst110` | 0x6D | 0x6E |
| Build-Datum / Versionsstring | Dec 26 2025 10:34:57, `109.0` | May 20 2026 13:59:48, `110.0` |
| Image-Endwort 0x0801BFFC | `B5 82` | `0F 90` |

## Prüfliste (ständige Prüfpunkte)

| Frage | Ergebnis |
|---|---|
| Modbus-Register | Das BMS hat keine Modbus-Tabelle; das Control stellt seine CAN-Werte bereit. **Geänderte Werte:** 32106/35111 (System-Ladestromgrenze) max. 40,0 A statt 50,0 A und 0 außerhalb des Master-Laufzustands (1105: 50,0 A); 32107/35112 (Entladestromgrenze) max. 40,0 A; 35110 (Ladespannungsgrenze) konstant 46,8 V (1105: Packspannung + 2,0 V, nach der CAN-Begrenzung max. 46,8 V); 30204/34010/37012 (BMS-Version) lesen **1100** (1105: 1105). Der Pack-Ladestrom in 0x1821 wird unter 35,00 V nicht mehr auf 10,0 A gesetzt. Registerliste, Typen und Skalen unverändert (Control-Seite). |
| BLE | Nicht zutreffend: das BMS hat kein BLE. Unverändert. |
| AC-Einspeisegrenze | Nicht im BMS. Indirekte Wirkung: die batterieseitige Grenze ist 40,0 A; bei der Nennspannung 13S (41,6 V) sind das ~1660 W, mehr als die 1450-W-Grenze des A-Micro, die AC-Leistung wird dadurch normalerweise nicht begrenzt; bei niedriger Packspannung (~36–37 V) sind 40 A ≈ 1450–1500 W (Schlussfolgerung, nicht gemessen). |
| Update-Pfad | Unverändert: CAN (`BMS_OTA_Flash_Writer`) und RS485 (`RS485_OTA_Flash_Handler`), Staging-Kopie 0x0801E000, Image-Kennung `'BMS2'`, CRC16. 1105 → 110 behält die Konfigurations-Versionskennung `'n'`, die gespeicherte Konfiguration bleibt also erhalten; 109 → 110 wechselt `'m'` → `'n'` und lädt die Standardwerte (Code-Befund; Verhalten im Feld nicht beobachtet). |
| Cloud-Endpunkte | Nicht zutreffend: das BMS hat keinen Netzzugang. |
| Pack-Kapazität | Unverändert: Grenze in `CAN_TX_PerPack_10Msgs` `pack_id != 0 && pack_id < 7` (Packs 1..6, Datensatz `pack_id - 1`), gleich in `CAN_TX_PerPack_SocDiag`; Abstand der Pack-Datensätze 0x60, 13 Zellen (`INIT_SH36735_Subsystem` unverändert), 16 Zellplätze in den Frames (14–16 = 0). |
| Zellzahl | 13 (unverändert). |
| Nicht funktionale Register | Alle A-Register mit Status FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL behalten ihren Status mit BMS 110: Pack 7 (34600–34633) wird weiterhin nie gesendet (`< 7`), Zellen 14–16 weiterhin 0, 0x1841 Bytes 6–7 weiterhin 0 (34x17/30106), Schreiber von 0x1851/0x1852 weiterhin nicht vorhanden, 45031 weiterhin unmöglich (`g_chargeStatusCode` enthält nur das beim Start gelesene EEPROM-Byte 0xFC02, das BMS schreibt dort 0xA5; das von 45031 erwartete Muster 0x00AA entsteht nie), 45011/47001 weiterhin ohne Empfänger. |

## Release Note → Code

| Release-Note-Punkt | Code-Befund |
|---|---|
| „Maximaler Lade-/Entladestrom auf 40 A geändert" | `Master_Build_ChargeCurrentLimit` und `Master_Build_DischargeCurrentLimit`: Grenze 500 → **400** (Einheit 0,1 A). Die Pack-Grenzen (`Max_Charge_Current_Calc`, `Max_Discharge_Current_Calc`, Tabellenmaximum 50,0 A) sind unverändert; die 40-A-Grenze gilt nur für die in 0x1803AA01 gesendeten Systemgrenzen. |

## Änderungen 1105 → 110

| Bereich | 1105 | 110 | Stelle (1105 → 110) |
|---|---|---|---|
| Grenze System-Ladestrom | 500 (50,0 A) | **400 (40,0 A)** | `Master_Build_ChargeCurrentLimit` 0800BCF0 → 0800BC20 |
| System-Ladestrom außerhalb Master-Laufzustand 2 | 50,0 A, bzw. 0 mit Statuscode 4 solange `g_socStateFlags` Bit5 gesetzt ist | **0** | ebenda |
| System-Ladespannungsgrenze | eigene Packspannung + 2,0 V, max. 48,0 V (PF3 begrenzt auf 46,8 V) | **fest 46,8 V** (0x1D4) | ebenda |
| Schreibzugriffe auf `g_chargeStatusCode` | 2 Pack offline, 3 SOC 100 %, 4 SELF-Flag | keine (nur der Startwert aus EEPROM 0xFC02) | `Master_Build_ChargeCurrentLimit`, `BMS_AggregateSystemPackStats` 0800B984 → 0800B8C0 |
| Grenze System-Entladestrom | 500 (50,0 A) | **400 (40,0 A)** | `Master_Build_DischargeCurrentLimit` 0800BDA4 → 0800BC90 |
| CAN 0x1803AA01 Ladestrom, solange die eigene Packspannung (`g_pack0Voltage_x10`) unter 35,00 V liegt | fest 100 (10,0 A) | Systemgrenze, kein Überschreiben | `CAN_TX_PF3_ChargeDischargeLimits` 08007C5C → 08007C1C |
| CAN 0x1821AA00+n Ladestrom, solange die eigene Packspannung unter 35,00 V liegt | fest 100 (10,0 A) für jedes Pack | die eigene Grenze des Packs | `CAN_TX_PerPack_10Msgs` 080083C0 → 08008338 |
| PV-Rettungs-CAN-ID 0x03410110 | setzt `g_pvRescueRequestFlag` (Byte 1 > 8) | **nicht behandelt** (Variable entfällt) | `CAN_RX_Handler` 08008264 → 080081FC |
| 0x100201AA im Systembefehl-Handler | ruft zusätzlich `CAN_CMD_02_Handler` | nur in `CAN_RX_Handler` (wie bisher) | `CAN_RX_System_Command_Handler` 0800FEB8 → 0800FD80 |
| Behandlung Zell-UVP Stufe 2 (1-s-Takt) | 120 s ohne PV-Rettungsanforderung → `System_Reset_And_Halt_WithCurrentCheck` | **D-Logik:** eigene MOSFETs nur unter 2,0 A, alleinstehendes Pack → Fehlermodus 3 nach 200 s (`g_protCommTickCounter200`) | `BMS_ProtectionAndCommTick` 080096E4 → 0800963C |
| UVP Stufe 2, Stufe A (≥ 0 °C) | 2800 mV, Auslösung = Schutz (Protect1 Bit5, SOC-Reset auf 0 %) | **3000 mV, Auslösung setzt nur `g_protState5` Bit2** (wie D) | `Cell_OVP_UVP_TempCompensated_Protection` 0800CE7C → 0800CD68 |
| Sensorfehler minimale Zelle (< 1500 mV) | nur oberhalb 39,00 V Packspannung | bei jeder Packspannung | ebenda |
| Master-MOSFET-Freigabe ohne SELF-Flag | Relais nur oberhalb 39,00 V zwangsweise aus | Relais bei jeder Spannung zwangsweise aus | `OwnPack_MOSFET_Driver_MasterVariant` 08007180 → 08007150 |
| Kriterium Niederspannungs-Balancer | SOC < 8 %, Protect1 Bit0 **oder minimale Zelle < 2800 mV** | SOC < 8 % oder Protect1 Bit0 | `Pack_LowVoltage_Balancer` 08007F00 → 08007EAC |
| Plausibilisierung HW-Überstromkomparator | ±70,0 A | **±3,0 A** (wie D) | `HW_OvercurrentAndSensorProtection` 0800B160 → 0800B09C |
| SOC-Schwelle „leer" (≥ 0 °C) | 2800 mV | **2750 mV** (wie D); Funktion umbenannt in `SOC_Select_Threshold_2420_2750` | `SOC_Select_Threshold_2420_2800` 080135C4 → 08013474 |
| FW-Kennung im Pack-Datensatz | 0x451 (1105) | 0x44C (1100) | `PerPack_Struct_Builder` 0800C934 → 0800C820 |
| Max. Ladestrom, Tabellenzweig | zusätzlicher Double-Vergleich gegen 0,0 (keine Wirkung gefunden) | ohne; `SoftFloat_Double_OrderedCompare_Rev` entfällt | `Max_Charge_Current_Calc` 080139A8 → 08013858 |
| Versionsstring / Build | `110.5`, 11.05.2026 | `110.0`, 20.05.2026 | Login-Banner (`CLI_LoginUser`) |
| RAM | `.data` 0x3A0 B; `g_uvpLvl2ShutdownTimerS`, `g_pvRescueRequestFlag` | `.data` 0x398 B (beide Variablen fehlen); 0x20002858 ist wieder `g_protCommTickCounter200` (1105: `g_standaloneModeFlag`); Stack-Ende 0x2000C958 | Regionstabelle 0x0801B9D4 → 0x0801B860 |

## Unverändert (ausdrücklich geprüft)

- Alles andere: die übrigen 677 von 697 Funktionen sind befehlsidentisch (Aufrufe, Globals, Strings und Immediates
  über Name/Wert verglichen); die Lade-Drosselung „nie voll geladen" (`Max_Charge_Current_Calc`, EEPROM 0xFC02) ist in
  110 (und 109) enthalten, ist also **keine** 1105-Neuerung.
- CAN-Frame-Satz und IDs, Opcodes 0x0A–0x0C, Antwort auf den Heartbeat `0x100501AA`, letter-shell (52 Einträge, Benutzer
  `BLUE`), RS485-Opcodes, SH36735-Treiber, SOC/SOH-Tabellen, OTA-Pfade und die Kennung `'BMS2'`.
- Entladespannungsgrenze 39,0 V, PF3-Ladespannungsbegrenzung 46,8 V, Kapazitätsfaktor 416.

## Kosmetische Unterschiede (kein geändertes Verhalten)

`RTOS_CreateAllTasks`, `__main`, `__dtoa_engine`, `_printf_core_IntEntry`, `prvProcessReceivedCommands`: nur
verschobene Literal-/Tabellenadressen (Stack-Ende, Switch-Tabelle, FreeRTOS-Assert-String).
