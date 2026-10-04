# Micro-FW v1211 (Venus A) — Vollanalyse

> 🇬🇧 English version: [VNSA_Micro_FW_v1211_Full_Analysis.md](VNSA_Micro_FW_v1211_Full_Analysis.md)

**Stand:** 2026-10-02.
**Image:** `1211_micro_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin` (Venus-A-Wechselrichterplatine, "VA_inv_app").
**Dekompilate/Listing:** `Dekompilate/VNSA_Micro_1211/` (nur lokal, nicht im Repo): 409 Funktionen, alle benannt
und typisiert, `listing/` mit ITCM- und Flash-Rohdisassembly sowie `globals_and_registers.txt` (1253 benannte RAM-/Peripherie-Symbole).
**Verwandt:** [VNSA_Fehlercodes_Micro_FW_Analyse_de.md](VNSA_Fehlercodes_Micro_FW_Analyse_de.md) (frühere Analyse auf falscher
Image-Basis, korrigiert in §6), [VNSD_Micro_FW_v116_Vollanalyse_de.md](VNSD_Micro_FW_v116_Vollanalyse_de.md) (Venus-D-Wechselrichter),
`../Control_FW/VNSA_Control_FW_v1509_Vollanalyse_de.md`, `../BMS/VNSA_BMS_FW_v1105_Vollanalyse_de.md`.

## 0. Zusammenfassung

- **Andere MCU und andere Codebasis als der Venus-D-Wechselrichter.** Cortex-M7 (I/D-Cache, ITCM/DTCM) aus der
  **Novosense-NS800RT-Familie** (C2000-artige Peripherie ePWM/eCAP/ADC A–D/CMPSS plus NXP-artiges FlexCAN, eDMA, DMAMUX,
  LPUART und ein Mathe-Beschleuniger). Die Speicheraufteilung ist identisch mit der NS800RT7-Linkerdatei (ITCM 0x00000000,
  DTCM 0x20000000, SRAM1 0x20100000, SRAM2 0x20120000); die Peripherie-Basisadressen von RCC/eDMA/GPIO weichen vom
  öffentlichen NS800RT7-Header ab, der genaue Typ (Single-Core-Baustein der NS800RT5xxx-Klasse) ist daher nicht bestimmt. 260 MHz Kerntakt.
- **Bare Metal, kein RTOS.** `main` (0x0801e540) läuft als Superloop (letter-shell 3.2.1, CAN-Dienst, Anwendungs-Task,
  Watchdog). Die Leistungselektronik-Regelung läuft in Interrupts: `Ctrl_MainControlIsr` (ITCM) mit **28 kHz** nach jedem
  ADC-DMA-Block, `Ctrl_SlowControlTask` (ITCM, 1-kHz-BTIM1-Interrupt), die zuerst die 231-Zustände-
  `Inv_StateMachine` und dann die MPPT-Zustandsmaschine aufruft.
- **Die A-Wechselrichterplatine enthält den gesamten Leistungspfad:** bidirektionaler Einphasen-Wechselrichter (netzgekoppelt
  + Off-Grid-/Backup-Ausgang, SOGI-PLL, QPR-Stromregler, P-f/Q-V-Droop für Off-Grid), einen PFC/DC-Zwischenkreis (400-V-Klasse,
  Abschaltung 530 V), eine LLC-DC/DC-Stufe (67 kHz, Synchrongleichrichtung) zur ~48-V-Batterie und **vier PV-Eingänge (MPPT,
  Buck/Boost je Kanal, EPWM4/6/7/8, 200 kHz)**. Beim Venus D ist der PV-Wandler ein eigener CAN-Knoten („Klasse 2"); beim
  Venus A **ist** dieser Micro der Klasse-2-Knoten und beantwortet die MPPT-Befehle des Control selbst.
- **Kein Modbus-/RS485-Register-Handler** (der D-Micro hat `Modbus_HandleRegisterAccess`). Die gesamte Konfiguration kommt
  als CAN-Befehle; die einzige UART ist die Debug-Shell (115200 Bd), die auch über CAN getunnelt werden kann (Funktion 0xC2/0xC3).
- **Leistungsgrenzen:** zwei EEPROM-Grenzen (0x203 Laden, 0x205 Entladen, Vorgabe **1450 W**), CAN-Klasse-4-Funktionen 0x04/0x03
  akzeptieren nur Werte **< 1451 W** (größere Werte vom Control — der bis 2500/1500 W zulässt — werden stillschweigend verworfen,
  der Frame wird trotzdem quittiert). Einspeisemaximum 1450 W, oberhalb von 73 °C Kühlkörpertemperatur linear abgeregelt.
  Off-Grid-Ausgang ausgelegt auf 1500 W (6,54-A-i²t-Integrator bei 230 V). MPPT-Gesamtleistungsgrenze 2400 W.
- **0x03410110 (BMS-„PV-Rettung")** ist kein eigener Frame: es ist **Frame-Index 3 der Telemetrie-Antwort des Micros
  (Klasse 4, Funktion 0x10)** mit den Nutzdatenbytes 16..23. Das BMS prüft Datenbyte 1 = High-Byte der Netzspannung
  in 0,1 V → `> 8` bedeutet Netzspannung ≥ 230,4 V. Der A-Micro sendet nie die ID 0x00210100 (BMS-„Lock"-ID) und ignoriert
  0x100101AA.
- Gefundene Firmware-Auffälligkeiten (§10): Unterfrequenz-Abschaltung ohne Verzögerung (vertauschter Vergleich), die Antwort
  auf Funktion 0x54 sendet ein nicht initialisiertes Stack-Byte, der Shell-Hilfetext für `set_mode` passt nicht zum Code, der
  EEPROM-Reset „Zähler behalten" löscht den Zählerblock trotzdem, der EEPROM-Selbsttest (CAN-Testbefehl) überschreibt EEPROM 0x100/0x101.
- Hersteller-Changelog zu 1211: „增强VA离网口带RCD载能力" = verbesserte Fähigkeit des Off-Grid-Ausgangs, Lasten hinter einem
  FI-Schutzschalter (RCD) zu versorgen.

### Antworten auf die Fragen der Venus-A-Registerauflösung

| # | Frage | Antwort (Details) |
|---|---|---|
| 1 | CAN-Empfangspfad, akzeptierte Befehle, Telemetrie | FlexCAN-RX-MB16..31 → Queue (50) → `CAN_RxDispatch`. Akzeptiert: Klasse 2 Funktion 0x01/0x02/0x03/0x04/0x06/0xCB/0xCC/0xFE/0xFF; Klasse 4 Funktion 0x01–0x07, 0x10–0x13, 0x16, 0x50–0x58, 0x60, 0xC1–0xC3, 0xCB; 0xCE OTA (jede Klasse); BMS-Frames 0x18n_AA01. Ignoriert: Klasse 4 0x08, 0x0A, 0x14, 0x15, 0x17, **0x18, 0x30–0x3E, 0x40–0x43**; Klasse 2 0x05, **0xFB**. Telemetrie: Funktion 0x10 (48 B, Aufbau §4.1.4), 0x11 (8 B Status), 0x12 (6 B ATE-Alterungsstand/-ergebnisse = „get old"), 0x13 (4 B gespeicherte Alterungsergebnisse), 0x54 (1 B, Müll), 0xCB (8 B Versionen). **Es werden keine 0x40–0x43- und keine 0x14-Frames gesendet** (§4.1). |
| 2 | BMS-Rettungs-Handshake | 0x03410110 = Frame 3 der 0x10-Telemetrie-Antwort, data[1] = High-Byte der Netzspannung; wird nur gesendet, wenn der Control Funktion 0x10 abfragt. Keine Reaktion auf 0x100101AA, sendet nie 0x00210100 (§4.2). |
| 3 | Leistungsklasse / Grenzen / Modi / Register-Handler | Keine Leistungsklassen-Logik; feste 1450-W-Grenzen (EEPROM 0x203/0x205), CAN 0x03/0x04 akzeptieren < 1451 W. Betriebsarten 0 Standard, 1 manuell, 2 Debug, 3 Alterung, 4 ATE-Alterung, 12 CAN-Testmodus, 13 Spannungsquelle (EEPROM 0x14). Backup = Flag (CAN Klasse 4 Funktion 0x02 in Modus 0 / Funktion 0x57 in Modus 1, EEPROM 0x11); Off-Grid-Ausgang über Zustand 30 / Standardzustand 200. Kein Modbus-Register-Handler (§5). |
| 4 | Fehler-/Warnwörter | err1 Bits 0–4, 7, 20, 21; err2 Bits 5, 6, 7, 8, 9, 11, 13; warn wird nie gesetzt (§6). |
| 5 | MPPT / PV | Ja: 4 PV-Eingänge auf dieser Platine (je ≈16–63 V, 20 A); die Klasse-2-Befehle = die `mppt*`-Register des Control adressieren diese MPPT-Stufe (§5.4). |

## 1. Binär-Fingerabdruck

| Merkmal | Wert |
|---|---|
| Datei | `1211_micro_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin` |
| Größe | 146.432 Byte (0x23C00) |
| SHA-256 | `277611ea8982adad4d9ac8e1a1154a9cdea6c9f717b1f60743be640f0023723d` |
| Linkadresse | `0x08006000`–`0x08029BFF` (24-KB-Bootloader darunter; Reset-Handler-Bytes `09 48 80 47 09 48 00 47` bei Dateioffset 0x434) |
| MCU | Cortex-M7, Novosense-NS800RT-Familie (siehe §0), 260 MHz (ePWM-Zeitbasis) |
| Toolchain | Keil MDK, armclang (LLVM): movw/movt-Konstanten, `mov pc,rX`-Sprungtabellen, ARM-C-Bibliothek (armlib) |
| Bibliotheken | letter-shell 3.2.1 (Build „May 20 2026 18:12:41"), FlexCAN-/eDMA-Treiber im MCUXpresso-Stil |
| Vektortabelle | 184 Einträge bei 0x08006000; initialer SP 0x20002400; Reset_Handler 0x08006434 → SystemInit 0x0801d278 → __main 0x080062e0 |
| Version | **1211 (0x4BB)** bei `0x08010000`; wird auch in EEPROM 0x03 geschrieben und über CAN 0xCB gemeldet |
| Trailer | Geräte-Tag `'VNSA'` bei 0x08029BE0, Prüfsummenwort bei 0x08029BFC |
| Shell-Benutzer | „letter V1211" mit **leerem Passwort** |

**Scatter-Load** (Region$$Table bei 0x080248FC, 7 Einträge):

| Quelle (Flash) | Ziel | Länge | Handler |
|---|---|---|---|
| 0x08024DC8 | 0x00000000 (ITCM-Code) | 0x4160 | `__scatterload_copy` |
| 0x08028F28 | 0x20000000 (DTCM, RAM-Vektortabelle) | 0x2E0 | `__decompress` (komplett null) |
| 0x08028F30 | 0x20100000 (SRAM1 .data) | 0x118 | `__scatterload_copy` |
| 0x08029048 | 0x20120000 (SRAM2 .data) | 0x2A0 | `__decompress` |
| – | 0x20000400 / 0x20100118 / 0x201202A0 (.bss) | 0x2000 / 0x3588 / 0x33F8 | `__scatterload_zeroinit` |

Stack 0x20000400–0x200023FF (DTCM), Heap 0x201016A0–0x2010369F (`__user_initial_stackheap`).

## 2. Speicheraufteilung und Analysestand in Ghidra

Programm `/VNSA/Micro/Micro_1211_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin`:

| Block | Bereich | Inhalt |
|---|---|---|
| `.itcm_code` | 0x00000000–0x0000415F | zeitkritischer Code (ISR, langsamer Regel-Task, Verzögerungsschleifen, Veneers ab 0x40DC) |
| `flash` | 0x08006000–0x08029BFF | Code 0x08006000–0x080211B9, danach Rodata/Strings, Region$$Table, Shell-Tabelle, ITCM-Ladeabbild, .data-Abbilder, 0xFF-Füllung, Trailer |
| `.data_20000000` | 0x20000000–0x200002DF | `g_ramVectorTable` (184 Zeiger) |
| `.bss_20000400` | 0x20000400–0x200023FF | Stack |
| `.data/.bss SRAM1` | 0x20100000–0x2010369F | Messwerte, Regelung, Schutz, Energiezähler, Heap |
| `.data/.bss SRAM2` | 0x20120000–0x20123697 | CAN-Queues, MPPT, Zustandsmaschine, Log-Ring, Shell |
| `periph` | 0x40000000–0x400FFFFF | 509 Register-Labels (NS800RT7-Registerlayouts, wo sie zum Code passen: EPWM1–8, ECAP1, ADCA–D, ADCx RESULT, IWDG1, ANASS, PWR, LPUART „UART1", BTIM1/2, CAN1, CRC, RCC, FLASHC, MMATH; eigene Layouts für GPIOA/B, CMPSS1–7, EDMA1/2, DMAMUX1/2) |
| `ppb` | 0xE0000000–0xE00FFFFF | Cortex-M7-Kernregister (SCB, NVIC, SysTick, Cache-Wartung) |

Prüf-Gates (`vna_mic_gates.py`, alle bestanden auf dem Endstand): 0 undefinierte Bytes und 0 verwaiste Instruktionen in
Flash und ITCM, 0 Kontrollfluss in Daten, 0 als Code dekodierte Literalwörter, 0 Aufrufe mitten in Funktionen, 0 Funktionszeiger
auf Nicht-Code, 0 `FUN_`-/doppelte/temporäre Namen, **0 unbenannte referenzierte RAM-/Peripherieadressen**, 0 Standard-Prototypen,
zur Laufzeit installierte IRQ-Handler benannt, No-Return-Flags plausibel. Tags: `NAME_VERIFIED` an allen 409 Funktionen,
`NAME_LOW_CONFIDENCE` an 27. Dekompilat-Export: 0 Fehler, keine nicht aufgelösten Sprungtabellen. Hinweis: die ITCM-Funktion
`Time_DelayUs` liegt an Adresse 0, daher erscheinen NULL-Zeiger-Vergleiche im dekompilierten Code als Vergleiche mit `Time_DelayUs`.

## 3. Ausführungsmodell

| Kontext | Rate | Funktion |
|---|---|---|
| Superloop | freilaufend | `main` → `shellTask`, `Shell_FlushOutputToCan`, `App_MainTask` (Warnwort, `Mode_UpdateStatusAndAging`, `CAN_CommTask`, OTA-Überwachung, LED GPIOA23 500 ms, Zähler alle 10 min ins EEPROM), IWDG1-Refresh |
| DMA1-IRQ (Slot 38) | 28 kHz | `DMA1_IRQHandler` → `Ctrl_MainControlIsr` (ITCM): ADC-Skalierung, `Ctrl_InverterControlStep` (Wechselrichter-/PFC-/LLC-Regelkreise), PV-/Batteriegrenzen und CMPSS-Latches → MPPT-Fehlerwort, Aktualisierung der vier MPPT-Wandler |
| BTIM1-IRQ (Slot 113) | 1 kHz | `BTIM1_IRQHandler` → `Ctrl_SlowControlTask` (ITCM): `Inv_StateMachine`, MPPT-Zustandsmaschine/Tracking, Abregelung, PV-Energie |
| BTIM2-IRQ (Slot 114) | 1 kHz | Millisekunden-/Sekundenzähler |
| CAN1-IRQ (Slot 111) | Ereignis | `CAN1_IRQHandler` → RX-Queue |
| EPWM1-IRQ (Slot 80) | 67-kHz-Ereignis | LLC-Registeraktualisierung (`Llc_UpdatePwmRegisters`) |
| ECAP1-IRQ (Slot 88) | Netzperiode | 4 Capture-Werte → Netzfrequenz |
| ADCB/ADCC-IRQ (Slots 62/67) | – | nur Flag löschen |
| IWDG1-IRQ (Slot 1) | Frühwarnung | alle PWM aus, 30 ms, Reset |

Die RAM-Vektortabelle (`IRQ_InitRamVectorTable`) wird mit `IRQ_DefaultHandler` gefüllt; jeder Treiber trägt seine ISR mit
`*(SCB_VTOR + 4*(16+irq)) = handler` ein.

## 4. Schnittstellen

### 4.1 CAN (500 kbit/s, 29-Bit-IDs)

Treiber: FlexCAN bei 0x40074000 (8-MHz-Takt), Handle 0x201204A0, MB0..15 TX reihum, MB16..31 RX. RX-ISR → Queue
`g_canRxQueue` (0x20120518, 50 × 16 B: ID, data[8], DLC); TX-Queue 0x2012084C (300 × 16 B), ein Frame alle 10 ms
(`CAN_CommTask` → `CAN_TransmitFrame`).

ID-Aufbau (identisch mit `CAN_Build_Arbitration_ID` des Control): Byte0 = Funktionscode, Byte1 = Klasse<<4 | Knoten,
Byte2 = Quelle (Control = 0x01), Byte3 Bits 0–4 = Frame-Index. Der Micro antwortet auf Klasse 4 mit der festen ID
`idx<<24 | 0x410100 | func` und auf Klasse 2 mit vertauschten Quell-/Zielbytes (`0x..2101xx`).

#### 4.1.1 Klasse 2 („MPPT"-/Register-Klasse des Control) — `CAN_RxDispatch`

| Funktion | Nutzdaten | Aktion |
|---|---|---|
| 0x01 | Frame 0, data[4] | 0 → `ShellCmd_stop` (MPPT-Regelung aus), 1 → `ShellCmd_start`; leere Quittung |
| 0x02 | Frame 1: u16 Modus, u16 Leistung W, u16 Spannung 0,1 V | Modus 1 = keine Leistungsaufteilung zwischen den Kanälen, 3 = Alterung (Stopp + Neustart); Leistung → `ShellCmd_mppt_pow` (Grenze 2400 W, Minimum 50 W), Spannung → `ShellCmd_mppt_vol` (44,2–47,5 V, EEPROM 0x541); Leistung 0 mit Spannung ≠ 0 = Leistungsstopp; Quittung |
| 0x03 | – | `Mppt_BuildStatusReport`, 8 Antwort-Frames (PV-U/I/P je Kanal, Batterie-U/I, Zustand, Leistungsvorgabe) |
| 0x04 | – | Neustart-Anforderung (`g_canRebootRequest`) |
| 0x06 | – | MPPT-Stopp + Energiezähler-Ebenen 0..4 löschen |
| 0xCB | – | Antwort 0x04BB0065 (Version 1211) + Versionswort |
| 0xCC | 7 Byte | Datum/Uhrzeit → `Meas_RollEnergyCountersOnDateChange` |
| 0xFE | – | PE-Offset-Kalibrierung (`Mppt_CalibratePeOffset`), Antwort Wert ×0,000791 (fünf Frames) |
| 0xFF | – | Antwort: vier PV-Strom-Offsets und Batteriestrom-Offset |
| sonstige (0x05, 0xFB …) | – | ignoriert |

Das sind genau die Befehle, die der Control in `Inverter_Set_OnOff_Reg_0x01`, `Inverter_CAN_PowerCmd_Send`
(g_mpptMode/g_mpptPowerW/g_mpptVoltage_0p1V), `Inverter_Set_Reg_0xFE/0xFF`, `Inverter_Set_WorkMode_Reg_0x06` usw. sendet —
beim Venus A erreichen sie die MPPT-Stufe dieses Micros. Die Antworten gehen an `Modbus_ResponseDispatch` des Control
(0x03, 0xCB, 0xCE, 0xFE, 0xFF).

#### 4.1.2 Klasse 4 (Wechselrichter-Klasse)

| Funktion | Aktion im A-Micro | Antwort |
|---|---|---|
| 0x01 | i32-Leistungsvorgabe W → `Inv_SetPowerSetpoint` (nur in Betriebsart 0) | 4 B (Wert) |
| 0x02 | data[0] < 2 → `Mode_SetBackupMode` (Backup-Flag, EEPROM 0x11; nur in Betriebsart 0) — Control: `Config_Mode_Apply` | Quittung |
| 0x03 | vorzeichenbehaftet i32 < 1451 → `g_powerLimitDischargeW`, EEPROM 0x205 (Control: max. Entladeleistung, erlaubt ≤ 1500); größere Werte werden verworfen, negative nicht abgewiesen | Quittung (auch wenn verworfen) |
| 0x04 | vorzeichenbehaftet i32 < 1451 → `g_powerLimitChargeW`, EEPROM 0x203 (Control: max. Ladeleistung, erlaubt ≤ 2500); größere Werte werden verworfen | Quittung (auch wenn verworfen) |
| 0x05 | Neustart (Zähler sichern, PWM aus, Reset) | Quittung |
| 0x06 | `g_cfgInverterDisable` = 1 (Startsperre, Telemetrie inv_state 0) | Quittung |
| 0x07 | Parameter-Reset Typ data[0] → `Param_ResetToDefaultsAndReboot` | Quittung |
| 0x10 | – | 48-Byte-Telemetrie (6 Frames, §4.1.4) |
| 0x11 | – | 8 B: Betriebsart, Wechselrichter-Sperrflag, Batteriemodus, 0, Ladegrenze, Entladegrenze (Control `g_vnsInv*`: work_mode, sleep_flag, bat_mode, hardware, charge/discharge_limit_power) |
| 0x12 | – | 6 B: `g_ateAgingState`, ATE-Alterungsergebnisse A/B („get old" = 老化 Alterung; Control `g_oldStep` → Register 37001/37002) |
| 0x13 | – | 4 B: gespeicherte Alterungsergebnisse (EEPROM 0x302/0x306) → Get-old-Queues des Control |
| 0x16 | 8 B Gesamt-Lade-/Entladeenergie (Wh) vom Control → `g_energyChargeWhRestore`/`...Discharge...` (Untergrenze für die Zähler) | – |
| 0x50 | data 1 → Betriebsart 12 (Testmodus), GPIO-/CMPSS-Latch-Reset, Testbefehl 7; data 0 → nur Latch-Reset | – |
| 0x51/0x53 | – | Quittung |
| 0x52 | Modus 12: Testbefehl data+1 (1..3) | Quittung |
| 0x54 | – | 1 B (nicht initialisiertes Stack-Byte, siehe §10) |
| 0x55 | Modus 12: Testbefehl 6 | Quittung |
| 0x56 | Modus 12: Leistungsvorgabe ±1449 W (Leistungsregler) | Quittung |
| 0x57 | Backup-Flag = (Betriebsart 1 && data == 1) | Quittung |
| 0x58 | Modus 1: Leistungsvorgabe ±2499 W (trotzdem auf die 1450-W-Grenze begrenzt) | Quittung |
| 0x60 | Betriebsart 4 (ATE-Alterung), `g_ateAgingState` = data | Quittung |
| 0xC1 | Testbefehl 1 (EEPROM-Selbsttest) | – |
| 0xC2 | data 1 → Shell über CAN an (Quittung), sonst aus (keine Quittung) | Quittung / – |
| 0xC3 | Shell-Eingabebytes (wenn Shell über CAN aktiv); Ausgabe geht als ID 0x004101C3 | – |
| 0xCB | – | 8 B: Versionswort (1211), Bootloader-Version, 1 |
| 0x08, 0x0A, 0x14, 0x15, 0x17, 0x18, 0x30–0x3E, 0x40–0x43, 0x59–0x5F, 0x61–0xC0 … | **ignoriert, keine Antwort** | – |

#### 4.1.3 Vom Micro gesendete Frames

Nur Antworten (kein periodischer Broadcast): 0x10, 0x11, 0x12, 0x13, 0x54, 0xCB und Quittungen (Klasse 4, ID 0x410100|func),
Klasse-2-Antworten (0x2101xx), 0xC3-Shell-Ausgabe, 0xCE-OTA-Quittungen. **Die Energie-Register des Control, die über die IDs
0x40–0x43 und 0x14 gefüllt werden (`Telemetry_Store_EnergyCounters`, `Protocol_AA_SetExtendedParams`), werden beim Venus A nie beschrieben.**

#### 4.1.4 Telemetrie-Nutzdaten (Funktion 0x10, `g_emsTelemetryBlock` 0x2012046C, gebaut von `CAN_UpdateTelemetryPayload` alle 400 ms)

Die Frames mit Index 1..6 tragen je 8 Byte; der Control kopiert sie 1:1 in seine Struktur `g_invState` (A Control 1509
0x2001769C).

| Offset | Größe | Quelle im Micro | Control-Feld / Modbus |
|---|---|---|---|
| 0x00 | u8 | `g_invTelemetryState` (0 gesperrt, 1 Leerlauf, 2 Laden < −10 W, 3 Entladen > 10 W, 4 Backup/Parallel, 6 beide Relais geschlossen) | inv_state 35100 |
| 0x01 | u8 | 0 | buz_state |
| 0x02 | u8 | `g_batDischargePermit` | chrg_flag |
| 0x03 | u8 | `g_cfgBackupModeEnabled` | back_func |
| 0x04 | u32 | `g_protWarningWord1` | warn_code 36000 |
| 0x08 | u32 | `g_protErrorWord1` | error_code1 36100 |
| 0x0C | u32 | `g_protErrorWord2` | error_code2 36102 |
| 0x10 | u16 | Netzspannung RMS ×10 | grid_volt (0,1 V) 30004 |
| 0x12 | u16 | Netzfrequenz ×10 | Netzfrequenz (0,1 Hz) |
| 0x14 | u16 | Off-Grid-Spannung ×10 | off_grid_volt 30005 |
| 0x16 | u16 | `g_gridPermit` | Netzfreigabe |
| 0x18 | i16 | Netzleistung W | grid_sample_power |
| 0x1A | i16 | Off-Grid-Leistung W | off_grid_power |
| 0x1C | i16 | Batterieleistung W | bat_sample_power |
| 0x1E | u16 | Batteriespannung ×10 | bat_sample_volt 30000 |
| 0x20 | i16 | Umgebungstemperatur ×10 | env_temp |
| 0x22 | i16 | Kühlkörpertemperatur ×10 | radiator_temp |
| 0x24 | i16 | `g_invPowerLimitMaxW` | max_power |
| 0x26 | i16 | `g_invPowerSetpointMinW` (negativ) | min_power |
| 0x28 | u32 | Ladeenergie Wh | chrg_energy |
| 0x2C | u32 | Entladeenergie Wh | dischrg_energy |

#### 4.1.5 OTA (Funktion 0xCE)

Frame-Index 0: Start (Image-Länge, CRC16) → 232 Seiten ab 0x08046000 löschen; Index 1: 8 Datenbytes in einen 1-KB-Puffer;
Index 2: Blockende mit CRC16 (CRC-Einheit) → 1 KB bei 0x08046000 + Block × 0x400 programmieren, Antwortstatus 1/2/3. Nach dem
letzten Block wird die CRC über das ganze Image geprüft, `OTA_WriteBootDescriptor` schreibt {0x08046000, Länge, 1, CRC} nach
0x08040000 und `App_MainTask` löst den Reset aus (`SCB_SystemResetOtaApply`). Ein 60-s-Timeout bricht die Sitzung ab.

### 4.2 BMS-Frames und der „PV-Rettungs"-Handshake

Akzeptierte BMS-Frames: ID `(id & 0x1FF0FFFF) == 0x1800AA01`, Byte2-Low-Nibble n = 1..4, gespeichert bei 0x20120440..0x2012045F:
Frame 1 Batteriespannung ×0,1, Strom, Maximaltemperatur, SoC ×0,1; Frame 2 Kapazität, Schlaf-Flag; Frame 3 Ladespannung ×0,1,
max. Lade-/Entladestrom ×0,1, Ladeanforderung, **Zwangsladeanforderung**; Frame 4 BMS-Fehler- und -Warnwort.
Frame 1 erneuert den BMS-Lebend-Zeitstempel (lebend = < 21 s). `Bat_UpdatePermits` leitet daraus die Lade-/Entladefreigaben,
die Low-SoC-Sperre (≤ 5 %), err1 Bit 20 (Kommunikation verloren) und Bit 21 (BMS-Fehler ≥ 4) ab. Ohne BMS (Batteriemodus 0)
gelten feste Grenzen 35,75 / 47,125 V, 60 / 70 A. Die Ereignisse 24/25 („Zwangsladeanforderung", „Zwangsladung ohne
Kommunikation") gehören zur Zwangslade-Sequenz in `Mode_UpdateStatusAndAging`/`Inv_StateMachine`.

**0x03410110:** Das A-BMS 1105 wartet in Zellunterspannung Stufe 2 bis zu 120 s auf die ID 0x03410110 mit data[1] > 8. Diese
ID ist `0x03<<24 | 0x410100 | 0x10` = Frame-Index 3 der 0x10-Telemetrie-Antwort des Micros (gebaut von
`CAN_TxQueue_PushMultiFrame`), also Telemetriebytes 16..23. Data[1] = Byte 17 = High-Byte von `Netzspannung ×10`, die
Bedingung bedeutet also **Netzspannung ≥ 230,4 V (0x900)**. Der Frame liegt nur auf dem Bus, solange der Control Funktion 0x10
abfragt; der Micro hat keinen eigenen Rettungs-Sender. Ob die BMS-Entwickler die Netzspannung oder ein anderes Feld gemeint
haben, lässt sich von der Micro-Seite aus nicht entscheiden (offener Punkt).
`0x100101AA` (Byte1 = 0x01, Klasse 0) wird ignoriert; die ID `0x00210100` (BMS-Fehlersperre) wäre eine Klasse-2-Antwort auf
Funktion 0x00, die der Micro nie erzeugt.

### 4.3 Debug-Shell (letter-shell 3.2.1, UART1 115200 Bd oder CAN)

68 Tabelleneinträge bei 0x08024988 (51 Befehle plus eingebaute Befehle/Tasten). Befehle (Handler `ShellCmd_<name>`):
`mppt_buck/boost/exbuck/exboost/buck_boost` (Tastgrad je Kanal im offenen Regelkreis), `mppt_epwm_start/stop`, `adcshow`, `prtshow`,
`dshow/cshow/mshow/ashow/pshow` (MPPT-Status), `start/stop/ss`, `mppt_pow`, `bms_power`, `mppt_vol`, `clear_fault`,
`sr/sw` (Software-I2C-Test), `inv_hr_start/stop/set`, `llc_start/stop`, `log_list`, `prt`, `llc_feq`, `bat_mode`,
`reset_memory`, `set_power`, `backup_mode`, `par_mode`, `set_mode`, `off_pi/on_pi/qpr` (Reglerverstärkungen), `reset`,
`io_show/io_set`, `rec`, `set_sr/open_sr`, `version`, `set_soft`, `power_cfg`, `test`.
Codes von `power_cfg`/`Inv_RequestCommand`: 1 Stopp, 2 PFC-Tastgrad offen, 3 PFC offen Off-Grid, 4 Off-Grid-Ausgang,
5 Laden CV/CC (35–45 V, 0–30 A), 6 Paralleltest, 8 netzgekoppelte Leistung, 9 Sinus offen, 10/11 LLC offen
Laden/Entladen (66–69 kHz), 12 LLC-Softstart, 21/22 Einzel-PFC Laden/netzgekoppelt, 30 Standardmodus, 40 Konstantspannungs-
ausgang, 50 Überspannungstest.

### 4.4 Serielles EEPROM (24Cxx, Software-I2C SCL GPIOA26 / SDA GPIOA27, WP GPIOB1, Geräteadresse 0xA0)

| Adresse | Inhalt |
|---|---|
| 0x001 | Reset-Typ: 3 = initialisiert, 2 = Vorgaben neu laden (Summen behalten — siehe aber §10), 0 = komplett löschen |
| 0x003 | gespeicherte Firmwareversion (u16, Vorgaben werden neu geschrieben, wenn < 108) |
| 0x011–0x015 | Backup-Flag, Parallelmodus, Batteriemodus, **Betriebsart (0x014)**, Wechselrichter-Sperrflag |
| 0x100 | 20 B Energie-/Betriebszähler (Lade-/Entlade-Wh …) |
| 0x10C / 0x110 | Energie-Wiederherstellungswerte aus CAN-Funktion 0x16 |
| 0x200–0x206 | Netzstandard-Code, max. SoC 100, min. SoC 12, **0x203 Ladegrenze, 0x205 Entladegrenze (Vorgabe 1450 W)** |
| 0x300–0x308 | Test-/Alterungsparameter; 0x302/0x306 ATE-Alterungsergebnisse |
| 0x502–0x50C | Datumswörter der Energiezähler-Hierarchie |
| 0x510–0x524 | sechs u32-PV-Energiezähler |
| 0x540/0x541 | Flag + Wert der manuellen MPPT-Spannungsvorgabe (0,1 V) |

## 5. Regelung und Leistungsteil

### 5.1 Hardware-Zuordnung

EPWM2/EPWM3 = Wechselrichter-H-Brücke (28 kHz up-down, HRPWM), EPWM1/EPWM5 = LLC-Halbbrücken (67 kHz, 66–69 kHz zulässig),
EPWM4/6/7/8 = PV-Wandler 1..4 (200 kHz). Die Ergebnisse von ADC A/B/C werden per DMA kopiert; Kanäle: Batteriestrom/-spannung,
Netzspannung, Wechselrichter-Drosselstrom, Lastspannung/-strom, PFC-Spannung, 400-V-Zwischenkreis, PE-Spannung, PV1..4 U/I, drei NTCs,
Chiptemperatur. CMPSS1–7 = Hardware-Überstrom-/Überspannungskomparatoren, Latch-Reset über GPIOA29, Abschalteingang GPIOA28.
Netzrelais GPIOB26, Off-Grid-Relais GPIOB6. Netzfrequenz aus ECAP1 (5,2e8 / Summe von 4 Perioden).

### 5.2 Betriebsarten (`g_cfgWorkMode` 0x2010077F, EEPROM 0x014)

| Wert | Bedeutung |
|---|---|
| 0 | Standard („Ctl_mode_default"): CAN-Leistungsvorgabe (Klasse 4 0x01) und Backup-Befehl (0x02) werden ausgeführt |
| 1 | manuell: CAN 0x57 (Backup) und 0x58 (±2499-W-Vorgabe) werden ausgeführt |
| 2 | Debug (Ersatzwert für jeden nicht aufgeführten Wert, auch 15) |
| 3 | Alterung: ±2200-W-Zyklen zwischen SoC 15 % und 99,9 % (`Mode_UpdateStatusAndAging`) |
| 4 | ATE-Alterung (CAN 0x60): −1350 W bis 100 %, Pause, +1350 W bis 30 %, Pause; Ergebnisse nach EEPROM 0x302/0x306 |
| 12 | CAN-Testmodus (Funktion 0x50), Voraussetzung für 0x51–0x56 |
| 13 | Spannungsquellen-Modus |

Der Shell-Hilfetext („0 Nulleinspeisung, 1 manuell, 2 Handelsmodus, 3 Debug, 15 bidirektionale Spannungsquelle") ist vom Control
übernommen und passt nicht zum Code. Weitere Flags: Backup (EEPROM 0x11), Parallel 0/1/2 (0x12), Batteriemodus
(0x13: 1 = BMS, 0 = feste Grenzen), Wechselrichter-Sperre (0x15).

### 5.3 Zustandsmaschine und Betriebsbefehle

`Inv_StateMachine` (Zustand `g_invState` 0x20120510, 231 Zustände, 1 kHz). Zustand 0 = Leerlauf: der anstehende Befehl
(0x20121B34, aus `Inv_RequestCommand`/CAN/Boot) wählt die Startsequenz: 10 Laden CV/CC, 20 netzgekoppelte Leistung,
30 Off-Grid-Ausgang, 40 Paralleltest, 70/80 Einzel-PFC, 100 LLC-Softstart, **200 Standardmodus** (automatischer
netzgekoppelter / Backup-Betrieb, wird beim Booten gestartet, außer bei Betriebsart 2 oder 13), 0xDC–0xE4 Tests im offenen
Regelkreis. Die Sequenzen prüfen die Netzfreigabe (Netzqualifizierung durch `Grid_CheckVoltageFrequency`), die Batteriefreigaben,
die Zwischenkreis-Vorladung (`Bus_CheckChargeDischargePermit`: Entladen ab ≥ 355 V, Laden solange ≤ 480 V), schalten die Relais
und starten PFC/LLC/H-Brücke; Fehler protokollieren den Grund (Ereignis-Log, §6.3) und setzen err2-Bits 7/9/11/13.

Regler (`Ctrl_InitControllers`): netzgekoppelter Strom-PI Kp 12 / Ki 0,6, Off-Grid-Spannungs-PI Kp 0,11 / Ki 0,13,
Zwischenkreis-PI Kp 28 / Ki 0,35, QPR-Stromregler (50 Hz), SOGI-PLL, Notch-Filter auf doppelter Netzfrequenz für die
Zwischenkreisspannung, Off-Grid-Droop 0,0004 Hz/W und 0,001 V/var um 326 V Scheitel / 50 Hz. Leistungsvorgabe-Begrenzung
`Inv_SetPowerSetpoint`: max. `g_invPowerLimitMaxW` (1450 W, über 73 °C abgeregelt), min. `g_invPowerSetpointMinW` = −Grenze oder
−(BMS-Max.-Ladestrom × Batteriespannung × 1,06); die Low-SoC-Sperre blockiert das Entladen.

Netzgrenzen (`Prot_LoadGridLimits`, Netzstandard EEPROM 0x200): Vorgabe 275 V/55 s, 287,5 V/1 ms, 174 V/55 s,
103 V/1 ms, 51,8/47,4 Hz, Wiederzuschaltung 265/197,8 V, 50,8/47,6 Hz, Wartezeiten 1 s/2 s; Standard 2 (EU-ähnlich) und 3 enger.
50/60-Hz-Automatik für die Standards 0/1.

### 5.4 MPPT (vier PV-Eingänge)

Je Kanal: Buck, Boost, Ex-Buck, Ex-Boost (und Buck-Boost in der Shell), Modus nach dem Übersetzungsverhältnis
Batterieseite/PV-Spannung gewählt (`Mppt_EpwmApplyMode`); Compare-Aktualisierung im geschlossenen Regelkreis in
`Ctrl_MainControlIsr`, Tracking in `Ctrl_SlowControlTask` mit ΔU/ΔI-Totbändern (Inkrementalleitwert-artig; genaue Variante nicht
vollständig verifiziert), Start ab ≈16 V PV. Grenzen: PV-Überstrom 20 A, Rückstrom −3 A, PV-Überspannung ≈63 V, Batterieseite
≈55 V, PE-Spannung 60 V, Temperaturen 90 °C (Abregelung ab 73 °C). Gesamtleistungsgrenze 2400 W, Leistungsaufteilung
zwischen den Kanälen, außer bei „keine Leistungsaufteilung" (Klasse-2-Modus 1). Status-/Fehlerwörter: MPPT-Fehlerwort
0x201005B0 (Codes 0x440–0x46A), Warnwort 0x201220EA (Codes 0x540/0x541, 0x553–0x55A). Beim Venus A beschreiben die
`mppt*`-Register des Control (g_mpptMode, g_mpptPowerW, g_mpptVoltage_0p1V, Klasse-2-Antworten 0x03/0xFE/0xFF) also diese
MPPT-Stufe auf der Platine.

## 6. Schutzfunktionen und Fehlercodes

Setzmuster: `Log_AddEvent(id, …)`, dann ODER ins Wort. Alle Setzstellen aufgezählt (korrigiert gegenüber der früheren
Analyse auf falscher Basis).

### 6.1 err1 `g_protErrorWord1` (0x20101650) → Modbus 36100/36101

| Bit | Ereignis | Bedingung (Vorgabegrenzen) | Funktion |
|---|---|---|---|
| 0 | 0x2C | Netz-Urms > 275 V für 55 s oder > 287,5 V | `Grid_CheckVoltageFrequency` |
| 1 | 0x2C | Netz-Urms < 174 V für 55 s oder < 103 V | dieselbe |
| 2 | 0x2A | Netzfrequenz > 51,8 Hz für 30 ms | dieselbe |
| 3 | 0x2A | Netzfrequenz < 47,4 Hz — **löst sofort aus** (vertauschter Zählervergleich, §10) | dieselbe |
| 4 | 0x2D | Momentan-Netzspannung > 430 V | dieselbe |
| 7 | 0x2E | Kurvenformabweichung |u − Urms·√2·sin θ| > 80 V für 5 Abtastwerte | dieselbe |
| 20 | (0x1A beim Löschen) | BMS-Kommunikation verloren (21 s kein Frame 1) | `Bat_UpdatePermits` |
| 21 | 0x3D | BMS-Fehlercode ≥ 4 (Frame 4) | dieselbe |

Die Bits 0–15 werden gelöscht, sobald sich das Netz wieder qualifiziert (Ereignis 6). Die frühere Analyse führte die Bits 0/1 als
Frequenz und 2/3 als unbekannt — das beruhte auf der falschen Image-Basis; die Tabelle oben stammt aus dem korrekt basierten Image.

### 6.2 err2 `g_protErrorWord2` (0x20101654) → Modbus 36102/36103

| Bit | Ereignis | Bedingung | Funktion |
|---|---|---|---|
| 5 | 0x43 | Wechselrichterstrom |I| > 15 A Spitze (alle PWM aus) | `Ctrl_InverterControlStep` |
| 6 | 0x35 | Zwischenkreis > 530 V (> 570 V im LLC-Modus 0x0E) | dieselbe |
| 7 | 0x15 | Abschaltung durch die Zustandsmaschine mit Grundcode 5 | `Inv_StateMachine` |
| 8 | 0x31 / 0x32 | Off-Grid-Überlast-i²t-Integrator > 600000 (Nennwert 6,54 A) oder Off-Grid-Spannung < 207 V für 200 ms | `Prot_UpdateOffGridOverload` |
| 9 | 0x29 | ungültiger Zustandsindex (> 0xE6) | `Inv_StateMachine` |
| 11 | 0x36 | Zeitüberschreitung beim Zustandswechsel (Zwischenkreis-Vorladung / LLC-Start / Relaissequenz), 3 Stellen | `Inv_StateMachine` |
| 13 | 0x48 | Relaisprüfung nach 20 s fehlgeschlagen (Ereignistext „Netzrelais verschweißt") | `Inv_StateMachine` |

### 6.3 warn `g_protWarningWord1` (0x20101658) → Modbus 36000/36001

Keine Setzstelle im Image — immer 0.

Ereignis-Log (`Log_AddEvent`, Ring 0x20122118, 230 × 20 B, Shell `log_list`): 5 Leerlauf, 6 Netzerkennung OK, 7/8 HV-Bus
Entladen/Laden erlaubt, 9/10 Batterie Laden/Entladen erlaubt, 13 Vorladung ein, 15/16 H-Brücke starten, 17/18 LLC-Start /
Softstart, 20 Wechselrichter aus, 21 System aus, 22 Backup ein, 23 BMS-Schlafanforderung, 24 Zwangsladeanforderung, 25 Zwangsladung
ohne Kommunikation, 26 Batteriekommunikation OK, 27 Netzfrequenz erkannt, 28 Reset-Grund, 41 Zustandsmaschinenfehler,
42 Netzfrequenzfehler, 43 SPLL-Fehler, 44 Netz-URMS-Fehler, 45 Netz-Spitzenspannung, 46 Kurvenform, 47 Netzstrom,
48 Software-PLL-Neustart, 49/50 Off-Grid-Strom-/-Spannungsschutz, 51 Netzerkennungsfehler, 52 Batteriekommunikations-
fehler, 53 Zwischenkreisspannungsschwankung, 54 Zeitüberschreitung Zustandswechsel, 55/56 Hardwareschutz, 57 PWM über Grenze,
58 Selbsttest, 60 Batterie auffällig, 61 Batterie-Fehlercode, 62 Batteriespannung, 63 zulässiger Ausgangsstrom, 64 SoC-Grenze,
66 Zwischenkreisspannung, 67 Spitzenstrom, 68 Spitzenspannung, 69 Watchdog-Reset, 70 EEPROM-Schreibfehler, 72/73 Netz-/Off-Grid-
Relais verschweißt, 74 Off-Grid-Nulldurchgangs-Einschaltung.

## 7. OTA / Update

Nur der CAN-Weg 0xCE (§4.1.5) in den Zwischenspeicher 0x08046000 plus Boot-Deskriptor bei 0x08040000; der Bootloader
(nicht Teil dieses Images) installiert es. Kein YMODEM-/UART-Update und kein BMS-Weiterleitungs-Update in diesem Image (anders als
beim D-Micro). Integrität: nur CRC16.

## 8. Sicherheitshinweise

- Shell-Benutzer „letter V1211" ohne Passwort; Shell über CAN (Funktion 0xC2/0xC3) gibt jedem am internen CAN-Bus (und über die
  Debug-Weiterleitung des Control) vollen Konsolenzugriff: `reset_memory`, `set_mode`, `power_cfg`-PWM-Befehle im offenen
  Regelkreis, `io_set` (Relais), `llc_feq` …
- OTA über CAN akzeptiert jedes Image mit passender CRC16 — keine Signatur.
- Die CAN-Testbefehle 0x50–0x58 (Modus 12) und die Alterungsmodi betreiben den Leistungsteil außerhalb der normalen Überwachung.

## 9. Unterschiede zur Venus-D-Wechselrichter-Firmware (v116)

| Thema | Venus-D-Micro 116 | Venus-A-Micro 1211 |
|---|---|---|
| MCU | STM32G474 (Cortex-M4F), CCM-Code | Cortex-M7 der NS800RT-Familie, ITCM/DTCM |
| Betriebssystem | FreeRTOS | Bare Metal |
| CAN | FDCAN, nur Klasse 4 (Klasse 2 ignoriert) | FlexCAN, Klasse 4 **und Klasse 2 (MPPT)** |
| PV | eigener Klasse-2-MPPT-Knoten | 4 PV-Eingänge auf dieser Platine |
| RS485 / Modbus-Register | `Modbus_HandleRegisterAccess` (40000/41000/45000 …) | keine |
| Leistungsgrenze | 2500 W (v116), CAN 0x03/0x04 < 2501 | 1450 W, CAN 0x03/0x04 < 1451 |
| Echtzeit-Vorgabe | 0x56/0x58 ±2500 W | 0x56 ±1449 W (Modus 12), 0x58 ±2499 W (Modus 1), Begrenzung 1450 W |
| Fehlerwörter | 0x200019F4.. (D-Bitbedeutungen) | 0x20101650.. (Bedeutungen in §6, abweichend) |
| Update-Wege | YMODEM über UART, CAN-Selbstupdate, BMS-Weiterleitung | nur CAN 0xCE |
| EEPROM | I2C-Hardware-Peripherie, Belegung 0x102/0x300/0x301 … | Software-I2C, Belegung §4.4 |

## 10. Firmware-Auffälligkeiten (gemeldet, nicht korrigiert)

1. `Grid_CheckVoltageFrequency` 0x0801478C: `cmp r1,r2; bcs skip` — der Unterfrequenz-Zähler löst aus, solange
   `Zähler < Grenze` gilt, also beim ersten Abtastwert unter 47,4 Hz (Überfrequenz verwendet `Zähler > Grenze`).
2. Klasse-4-Funktion 0x54 antwortet mit dem nicht initialisierten Stack-Byte `local_35`.
3. Der Hilfetext von `set_mode` passt nicht zu den akzeptierten Werten (15 → 2).
4. `Param_LoadFromEeprom` liest beim Reset-Typ 2 den Zählerblock und füllt ihn danach mit Nullen (der Reset „Summen behalten" löscht sie).
5. `EEPROM_SelfTest` (CAN-Testbefehl 1) schreibt EEPROM 0x100/0x101 (Zählerblock).
6. `Flash_ProgramDoubleWords` kehrt bei einem Verifikationsfehler zurück, ohne PG zu löschen bzw. den Flash zu sperren.
7. Der Fehlertext von `power_cfg` nennt 40–60 V / 0–60 A, geprüft werden 35–45 V / 0–30 A.
8. `_sys_exit` ist leer implementiert (würde in den Reset-Pfad zurückkehren).

## 11. Offene Punkte

1. Genauer NS800RT-Typ (Peripherie-Basisadressen weichen vom öffentlichen RT7-Header ab); das CMPSS-Layout ist vorläufig.
2. Bedeutung von 0x03410110 data[1] > 8 auf BMS-Seite (Netzspannung ≥ 230,4 V, wie der Code nahelegt, oder ein
   falsch interpretiertes Feld?) — erfordert die Absicht der BMS-Entwickler oder einen Gerätetest.
3. Variante des MPPT-Tracking-Algorithmus und Bedeutung der fünf Energiezähler-Ebenen (Hypothese Datumshierarchie).
4. Grundcode 5 von err2 Bit 7 und die genaue Relais-/Spannungsbedingung hinter err2 Bit 13.
5. PE-Spannungsfehler (MPPT-Bit 10/11) — Hypothese Isolations-/PE-Überwachung.
