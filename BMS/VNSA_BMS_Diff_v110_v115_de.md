# Diff-Tabelle BMS v110 → v115 (Venus A)

> 🇬🇧 English version: [VNSA_BMS_Diff_v110_v115.md](VNSA_BMS_Diff_v110_v115.md)

**Stand:** 2026-10-11. Alle drei Images mit Basis 0x08002800 in Ghidra, vollständig benannt (110: 697 Funktionen,
110.5: 698, 115: 727). Adressen: 110 → 115. **Hauptdokumentation:**
[VNSA_BMS_FW_v115_Vollanalyse_de.md](VNSA_BMS_FW_v115_Vollanalyse_de.md), Vorgängeranalysen
[VNSA_BMS_FW_v110_Vollanalyse_de.md](VNSA_BMS_FW_v110_Vollanalyse_de.md) und
[VNSA_BMS_FW_v1105_Vollanalyse_de.md](VNSA_BMS_FW_v1105_Vollanalyse_de.md).

Funktionspaare über gleiche Namen, Befehl für Befehl verglichen mit Aufrufzielen und Globals nach Namen, referenzierten
Strings und Direktwerten (RAM-Namen von 115 zuvor geprüft, siehe Vollanalyse §2.2):

| Vergleich | Neu in 115 | Nur im Vorgänger | Geändert (inkl. nur Adressen) |
|---|---|---|---|
| 110 → 115 | 30 | 0 | 61 |
| 110.5 → 115 | 31 | 2 (`SoftFloat_Double_OrderedCompare_Rev`; `SOC_Select_Threshold_2420_2800` nur in 110 umbenannt) | 56 |

## Codebasis

| Merkmal | 110 | 110.5 | 115 |
|---|---|---|---|
| Versionsstring / Build | `110.0`, 20. Mai 2026 | `110.5`, 11. Mai 2026 | `115.0`, 1. Sep 2026 |
| OTA-Edition | 3430 | 3316 | 4563 |
| Maskierte Segmentabdeckung des 115-Images | 81,4 % | 82,4 % | — |
| FW-Kennung im Pack-Datensatz (Modbus 30204/34010/37012) | 0x44C = 1100 | 0x451 = 1105 | **0x73 = 115** |
| Konfigurations-/RS485-Versionskennung | `'n'` | `'n'` | **`'s'`** |

115 ist das erste Image, das beide A-Zweige zusammenführt: Der Beta-Code von 110.5 (PV-Rettung,
Niedrigspannungsbegrenzungen und -schwellen) ist enthalten, die Teile von 110/109 ohne die Ladestatuscodes von 110.5 und
ohne packspannungsabhängige Ladespannung bleiben, und neue Funktionen kommen hinzu. Die Abdeckung gegen D 118 beträgt
65,7 %, gegen E3 BMS 115 12,6 %: 115 ist kein E3- oder D-Zweig.

## Prüfliste (Pflichtprüfungen)

| Frage | Ergebnis |
|---|---|
| Modbus-Register | Das BMS hat keine Modbus-Tabelle; das Control stellt seine CAN-Werte bereit. Mit BMS 115 geänderte Werte (Registerliste/Typen/Skalen sind Control-seitig und unverändert): **32106/35111** System-Ladestromgrenze max. **50,0 A** (110: 40,0 A), 50,0 A auch außerhalb des Master-Laufzustands, sofern das SELF-Flag nicht gesetzt ist (110: 0); **32107/35112** Entladestromgrenze max. **50,0 A**; **35110** Ladespannungsgrenze intern **48,0 V**, der CAN-Frame trägt weiterhin ≤ 46,8 V (110: 46,8 V); **30204/34010/37012** (und 34x10 von Slaves mit 115) BMS-Version **115** (110: 1100); 32106 und der Ladestrom pro Pack (0x1821) werden auf **10,0 A gezwungen, solange das eigene Pack des Masters unter 35,00 V liegt**; **32105** Kapazität unverändert; PF4 Byte 4 (Ladestatuscode) ist immer 0. SOC-Register (32104, 34x02, 37005): gleiche Bedeutung, aber die Lade-/Entlade-Endkorrektur braucht jetzt drei Zellen (geänderter SOC-Verlauf nahe 0/100 %) und die Leer-Schwelle ist 2800 mV. |
| BLE | Nicht anwendbar: Das BMS hat kein BLE. Unverändert. |
| AC-Einspeisegrenze | Nicht im BMS. Indirekt: batterieseitige Grenze 50,0 A ≈ 2080 W bei 41,6 V, über der Grenze des A-Micro, also normalerweise nicht begrenzend; unter 35,00 V eigener Packspannung beträgt der CAN-Ladestrom 10,0 A (≈ 350 W Ladung) (Schlussfolgerung, nicht gemessen). |
| Update-Pfad | Unverändert: CAN- und RS485-OTA, Staging 0x0801E000, Kennung `'BMS2'`, CRC16. **Konfigurations-Versionskennung `'n'` → `'s'` und Block 0xD4 → 0xD8 Byte: die gespeicherte Konfiguration wird nach dem Update durch Defaults ersetzt** (Code-Fakt). Ein Downgrade 115 → 110 setzt die Konfiguration ebenso zurück. |
| Cloud-Endpunkte | Nicht anwendbar: Das BMS hat keinen Netzzugang. |
| Pack-Kapazität | Unverändert: `CAN_TX_PerPack_10Msgs` (`0x08008DE8`) Grenze `pack_id != 0 && pack_id < 7` → Packs 1..6 (Datensätze `pack_id - 1`), Datensatz-Schrittweite 0x60, Logik um `g_highestPackIndex` unverändert. |
| Zellzahl | 13 (`INIT_SH36735_Subsystem` bis auf seine Versionskonstante unverändert; `APP_Get_Pack_Summary_Data` läuft über 13 Zellen). |
| Nicht funktionale Register | BMS-seitig erneut geprüft: Pack 7 (34600–34633) wird weiterhin nie gesendet (`< 7`); Zellen 14–16 weiterhin 0; 0x1841 Bytes 6–7 weiterhin 0 (34x17/30106); Schreiber der 0x1851/0x1852-Worte weiterhin nicht vorhanden (`g_cfgTailReg*` nur aus dem Konfigurationsblock); **45031 weiterhin unmöglich** (PF4 Byte 4 ist jetzt konstant 0); 45011/47001 weiterhin ohne Empfänger. Neuer ausgehender Frame `0x1805AA01`: im A-Control 151 kein Empfänger gefunden. |

## Release-Note → Code

| Release-Note-Punkt (OTA-Remark, übersetzt) | Code-Befund |
|---|---|
| 1. Packwechsel bei SOC-Differenz zwischen Packs > 20 | `PackSelect_SocImbalanceSwitch` `0x08007B98` (neu), von `PackSelect_ModeDispatch` in Modus 0 aufgerufen: Spreizung ≥ 20 %, leistungsabhängige Verzögerung 1200–3000 Ticks, Laden → niedrigster SOC, Entladen → höchster SOC |
| 2. Dynamische Korrektur der Lade-/Entlade-Zielwerte | `PackSelect_ChargeStepTarget` `0x08009FF4` (95-/100-%-Regeln über das neue `PackSelect_AllOnlineSocAtLeast`), Neuberechnung der Schwellen in `PackSelect_Charge_StepRotate` / `_Discharge_StepRotate` |
| 3. Lade-/Entlade-Endkorrektur braucht drei Zellen | `APP_Get_Pack_Summary_Data` `0x08014DD8` zählt Zellen ≥ 3500 mV / ≤ 2900 mV; `Charge_SOC_Integration` / `Discharge_SOC_Integration` nutzen „≥ 3 Zellen“ statt Max-/Min-Zelle |
| 4. Verbesserte Lade-/Entlade-MOS-Befreiung | `OwnPack_MosRecovery_SM` `0x08005BA4` (neu, Ausgang PE7, CAN `0x1805AA01`), Auslöser in `OwnPack_MOSFET_Driver_MasterVariant` |

## Änderungen 110 → 115 (mit 110.5 zum Vergleich)

| Bereich | 110 | 110.5 | 115 | Ort (110 → 115) |
|---|---|---|---|---|
| System-Ladestromgrenze | 400 (40,0 A) | 500 | **500 (50,0 A)** | `Master_Build_ChargeCurrentLimit` 0800BC20 → 0800C8C0 |
| System-Ladestrom außerhalb Master-Laufzustand 2 | 0 | 50,0 A / 0 mit SELF-Flag (Statuscode 4) | **50,0 A / 0 mit SELF-Flag** (kein Statuscode) | dito |
| System-Ladespannungsgrenze | 46,8 V fest | Packspannung + 2,0 V, max. 48,0 V | **48,0 V fest** (PF3 klemmt CAN auf 46,8 V) | dito |
| System-Entladestromgrenze | 400 | 500 | **500** | `Master_Build_DischargeCurrentLimit` 0800BC90 → 0800C93C |
| `g_chargeStatusCode`-Schreibzugriffe / PF4 Byte 4 | nur Startwert / gesendet | 2/3/4 / gesendet | keine / **Byte 4 = 0** | `CAN_TX_PF4_ProtectWarnings` 08007DA0 → 0800876C |
| CAN-Ladestrom 0x1803/0x1821 unter 35,00 V eigener Packspannung | keine Begrenzung | 10,0 A | **10,0 A** | `CAN_TX_PF3_ChargeDischargeLimits` 08007C1C → 080085D4, `CAN_TX_PerPack_10Msgs` 08008338 → 08008DE8 |
| PF2 Byte 7 | Relay-Comm-Unterzustand 1/2/3 | gleich | **Packauswahl-Quittung (2), nach dem Senden gelöscht** | `CAN_TX_PF2_Capacity` 08007B50 → 080084EC |
| CAN 0x03410110 | ignoriert | `g_pvRescueRequestFlag` = Byte 1 > 8 | Byte 1 > 8 **oder Byte 6 == 1** | `CAN_RX_Handler` 080081FC → 08008BD4 |
| CAN 0x1801FFAA Byte 3 | ignoriert | ignoriert | **`g_emsHeartbeatModeByte`** | dito |
| CAN 0x01210103, 0x04410110 | — | — | **neu** (Sperrflag; float-Verhältnis ohne Leser) | dito |
| CAN 0x1805AA01 (ausgehend) | — | — | **neu**, Fortschritt der MOS-Befreiung, aus dem Leerlauf-Tick | `APP_IdleKeepaliveBroadcast_Tick` 0800E290 → 0800F144 |
| Periode Leerlauf-Tick | 500 ms | 500 ms | **50 ms** | dito |
| Zell-UVP Stufe 2 (1-s-Tick) | D-Logik: MOSFETs < 2,0 A, Einzelpack → Fehlermodus 3 nach 200 s | 120 s ohne PV-Rettung → STOP | **wie 110.5** | `BMS_ProtectionAndCommTick` 0800963C → 0800A18C |
| UVP Stufe 2, Stufe A (≥ 0 °C) | 3000 mV, nur Warnung | 2800 mV, Schutz | **2800 mV, Schutz** | `Cell_OVP_UVP_TempCompensated_Protection` 0800CD68 → 0800DBB0 |
| Min-Zellen-Sensorfehler-Auslösung | jede Spannung | > 39,00 V | **> 39,00 V** | dito |
| Schutz während AFE-Kommunikationsfehler | ausgewertet | ausgewertet | **alle Zähler gelöscht, nicht ausgewertet** (Zelle, Strom, Temperatur) | dito, `Current_Protection_Checker` 0800AC6C → 0800B808, `Voltage_Temp_Protection_Checker` 0800C168 → 0800CE54 |
| Pack-Unterspannungs-STOP | — | — | **neu: < 36,00 V für 1200 Ticks → STOP (außer bei PV-Rettung)** | `Pack_UnderVoltageShutdown_Check` 0800D590 |
| HW-Komparator-Gegenprüfung | ±3,0 A | ±70,0 A | **±70,0 A**; NTC-/MOS-Temperatur-Bereichsprüfungen unter 20,00 V übersprungen | `HW_OvercurrentAndSensorProtection` 0800B09C → 0800BC70 |
| Ausgang PD8 | — | — | **neu: High unter 20,00 V** | `Pack_LowVoltageOutput_PD8_Update` 0800A330 |
| Master-MOSFET-Sperre ohne SELF-Flag | aus bei jeder Spannung | aus > 39,00 V | **aus > 39,00 V**; PE7 zusammen mit den übrigen Ausgängen aus; Start der MOS-Befreiung | `OwnPack_MOSFET_Driver_MasterVariant` 08007150 → 080078C8 |
| Niedrigspannungs-Balancer | SOC < 8 % oder Protect1 Bit 0 | + Min-Zelle < 2800 mV | **wie 110.5** | `Pack_LowVoltage_Balancer` 08007EAC → 08008870 |
| SOC-Leer-Schwelle (≥ 0 °C) | 2750 mV | 2800 mV | **2800 mV**; Funktion umbenannt in `SOC_Select_Threshold_2420_2800` | 08013474 → 08014624 |
| Lade-End- / Entlade-End-Korrektur | Max-Zelle ≥ 3500 / Min-Zelle ≤ 2900 mV | gleich | **≥ 3 Zellen** | `Charge_SOC_Integration` 08012A9C → 08013C50, `Discharge_SOC_Integration` 08012F10 → 080140C0 |
| Lade-Stufenziel | 10-%-Stufen | gleich | **100 %, wenn alle ≥ 95 %, 95 %, wenn alle ≥ 90 %** | `PackSelect_ChargeStepTarget` 080094C4 → 08009FF4 |
| SOC-Ungleichgewichts-Wechsel | — | — | **neu (≥ 20 %)** | `PackSelect_SocImbalanceSwitch` 08007B98 |
| Pack-Übergabe | `PackHandoff_StateMachine` | gleich | + **SOC-Wechsel-Zustandsautomaten** bei Modus-Byte 2 und PV-Rettung | `PackHandoff_Tick` 08006A48 → 08007150, neu 08006B5C / 08006D0C |
| Freigabemasken vor Neuwahl | beibehalten | beibehalten | **auf das aktive Pack zurückgesetzt** | `PackSelect_ChargeEnable_Maintain`, `_MosMismatch_Fallback`, `_Next_AboveSocThreshold` |
| Relay-Comm-Zustandsautomat | Unterzustand 1/2/3, Befehlsflag nach 300 Ticks gelöscht | gleich | **kein Unterzustand, Flag sofort gelöscht, Eskalationen 300/600 durch `g_relayCommEscalationInhibit` gesperrt** | `PackSelect_RelayCommStateMachine` 0800B2CC → 0800BEE8 |
| LED-Phasen Relay-Comm | immer | immer | unterdrückt, solange gesperrt | `LED_StatusPatternUpdate` 0800BCF4 → 0800C9C4 |
| Vorlade-Wiederholung | Wiederholung gezählt, außer die Spannung fällt noch um > 1,0 V | gleich | auch nicht gezählt, solange die Differenzkennzahl > 7 ist | `OwnPack_Precharge_SM` 08006D84 → 080074D0 |
| Schaltausgänge | Aktionen 1–6 | gleich | **+ 7/8 = PE7 ein/aus** | `HAL_GPIO_Control_Switch` 08006508 → 08006890 |
| Pack-Statusflags | Bits 0–5 | gleich | **+ Bit 6 = PB2, Bit 7 = PE7** | `PerPack_Struct_Builder` 0800C820 → 0800D63C |
| STOP-Modus | STOP + Reset | gleich | **RTC-Weckzeitgeber vor STOP scharf geschaltet (nominal 6 h, LSI)**, Weckflag in BKP_DAT2, nach 50 Ticks zurück in STOP | `System_Reset_And_Halt_WithCurrentCheck` 08004A40 → 08004B1C, 16 neue RTC-/BKP-/PWR-/RCC-Funktionen |
| Start | — | — | **`BKP_ReadAndClearRtcWakeFlag`**, Einrichtung PC8, Neuinitialisierung PC5/PB0/PB2/PB5 nach RTC-Wecken | `System_Init_Main` 08004670 → 08004748, `GPIO_Init_All_Pins` 080040B8 → 08004148, `GPIO_DeInit_And_CAN_NVIC_Init` 080043D0 → 0800449C |
| Konfigurationsblock | 0xD4 B, Kennung `'n'`, Prüfsumme 0..0xCF | gleich | **0xD8 B, Kennung `'s'`, Prüfsumme 0..0xD3 (schließt sich selbst ein)**, neues Feld 0xD4 = 120 | `APP_Config_Checksum_And_Save` 0800AC48 → 0800B7E4, `BMS_Config_Params_Load_Or_Default` 08009CAC → 0800A840, `RS485_RegisterWrite_DefaultGroup` |
| SOC-Kalibrierlog | 50 × 20 B ab 0x400 | gleich | **5 × 20 B ab 0x440** | `APP_EventLogGetSlotAddrs`, `APP_FormatEventLog_IfNeeded`, `SOC_EepromLoadCalibration` |
| Versionskonstanten | 0x6E, 1100 | 0x6E, 1105 | **0x73, 0x73** | `RS485_Dispatcher`, `RS485_Pack_Telemetry_Response`, `INIT_SH36735_Subsystem`, `BMS_Config_Params_Factory_Reset`, `PerPack_Struct_Builder` |
| Bibliothek | — | — | `SoftFloat_Int_To_Double`, `SoftFloat_Double_To_Float` ergänzt (wie in E3 115); `__dtoa_engine` anders übersetzt | — |
| RAM | `.data` 0x398 B | 0x3A0 B | **0x3E0 B**, `.bss` 0x9DC8 B, Stack-Obergrenze 0x2000C9A8, 30 neue Globals | Regionstabelle 0801B860 → 0801CA30 |

## Verhalten eines A-Systems nach dem Update 110 → 115 (aus dem Code)

- Lade-/Entladegrenzen steigen von 40,0 A auf 50,0 A, Ladespannungsgrenze intern 48,0 V.
- Konfiguration beim ersten Start auf Defaults zurückgesetzt (Kennungswechsel); wegen der sich selbst einschließenden
  Prüfsumme möglicherweise bei jedem Start (offener Punkt 1 der Vollanalyse).
- Ein Pack in Zell-UVP Stufe 2 schläft nach 120 s, sofern der Wechselrichter den PV-Rettungs-Frame nicht sendet; ein
  Pack unter 36,00 V schläft nach 1200 Ticks; schlafende Packs wachen über die RTC etwa alle 5–6 h auf und legen sich
  wieder schlafen.
- Packwechsel zusätzlich bei ≥ 20 % SOC-Spreizung; Wechselschwellen passen sich nahe voll an.
- SOC-Endkorrekturen später (drei Zellen statt einer), Leer-Schwelle 2800 mV.

## Unverändert (ausdrücklich geprüft)

CAN-Frame-Satz bis auf die Zeilen oben, Opcodes 0x0A–0x0C, letter-shell (52 Einträge, Benutzer `BLUE`), RS485-Opcode-Satz,
SH36735-Treiber und Balancing, SOC/SOH- und Derating-Tabellen (byte-identisch), OTA-Pfade und Kennung `'BMS2'`,
FreeRTOS-Tasks, Entladespannungsgrenze 39,0 V, Kapazitätsfaktor 416, Lade-Derating „nie voll geladen“.

## Kosmetische Unterschiede (keine Verhaltensänderung)

`RTOS_CreateAllTasks`, `__main`, `_printf_core_IntEntry`, `prvProcessReceivedCommands`, `prvHeapInit`,
`BMS_OTA_Flash_Writer_FinalizeTail`, `CAN_CMD_Pack_Activation`, `Max_Charge_Current_Calc` (float-Klemmung auf das
Ganzzahlergebnis verlagert, gleiche Werte), `PackMode_Dispatcher` / `PackMode_Active_MasterClaim` (nur die entfallenen
Schreibzugriffe auf den Unterzustand).
