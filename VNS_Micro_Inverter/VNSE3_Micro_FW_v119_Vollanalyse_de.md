# Micro-FW v119 (Venus E3) — Vollanalyse

> 🇬🇧 English version: [VNSE3_Micro_FW_v119_Full_Analysis.md](VNSE3_Micro_FW_v119_Full_Analysis.md)

**Stand:** 2026-10-02.
**Image:** `119_micro_VNSE3-0_inv_app_0119_0326_ota_152726.bin` (Wechselrichterplatine Venus E 3.0, „inv_app“).
**Dekompilate/Listing:** `Dekompilate/VNSE3_Micro_119/` (nur lokal, nicht im Repo): 708 Funktionen, alle benannt und typisiert,
`listing/` mit Flash- und CCM-Rohdisassembly und `globals_and_registers.txt` (2509 benannte RAM-/Peripherie-Symbole).
**Verwandt:** [VNSD_Micro_FW_v116_Vollanalyse_de.md](VNSD_Micro_FW_v116_Vollanalyse_de.md) (Venus-D-Wechselrichter, nächste Referenz),
[VNSA_Micro_FW_v1211_Vollanalyse_de.md](VNSA_Micro_FW_v1211_Vollanalyse_de.md) (Venus-A-Wechselrichter),
`../Control_FW/VNSE3_Control_FW_v151_Vollanalyse_de.md`, `../BMS/VNSE3_BMS_FW_v115_Vollanalyse_de.md`.

## 0. Zusammenfassung

- **Gleiche MCU und gleiche Codebasis wie die Wechselrichter-Firmware des Venus D.** STM32G474 (Cortex-M4F, HRTIM, FDCAN, CORDIC),
  Keil ARMCC, FreeRTOS (heap_4), letter-shell; zeitkritischer Code läuft aus dem CCM-SRAM (0x6F74 Bytes nach 0x10000000 kopiert) und
  wird über Long-Branch-Veneers erreicht. **73,5 %** der Image-Bytes sind mit D Micro 116 identisch (Pointer und BL-Ziele maskiert),
  70,5 % mit D 115, nur 2,3 % mit A Micro 1211 (C-Bibliothek und Strings; andere MCU und Codebasis). 546 der 693 Funktionen von D 116
  sind byteidentisch, 64 geändert, 83 D-Funktionen haben an gleicher Stelle kein Gegenstück (verschoben, zusammengelegt oder entfernt);
  E3 hat 708 Funktionen (29 Namen gibt es in D 116 nicht).
- **Der Build ist älter als D 116:** „Mar 26 2026 15:20:51“ (D 116: „Jul 2 2026“). Versionswort 0x77 (119), gefolgt vom Gerätekennzeichen
  **„AC02“** an 0x08010000 (D: 0x74 + „VD01“); Image-Trailer **„VNSEE3“** (D: „VNSD“) — dasselbe Modellkennzeichen, das die E3 Control 151 prüft.
- Hersteller-Changelog v119: „1.修复soc低于6%不强充问题“ = Behoben: unterhalb von 6 % SOC wurde die Zwangsladung nicht gestartet. Die
  genaue Korrektur ist nicht eindeutig zuzuordnen (§5.4); beteiligt sind der Low-SOC-Code in `Inv_GridConnectionSupervisor` (Schwelle
  SOC < 6 %, neue Entprellung) und das neue CAN-Kommando 0x0A für den Abschalt-SOC (akzeptiert 6..79 %).
- **Kein PV/MPPT** (wie beim Venus D hatte der Micro nie die PV-Stufe; der E3 hat auch keinen Klasse-2-MPPT-Knoten). Ein ADC2-Kanal
  (dritter NTC) und die NTC-Tabelle 1 sind entfernt; zwei NTCs bleiben.
- **Änderungen im Leistungsteil:** Wechselrichterbrücke mit **30 kHz** PWM (D 28 kHz, HRTIM-Periode 45333 statt 48572), LLC-Start
  128 kHz (D 126 kHz), der Netzstromregler ist jetzt ein **PR-Regler (proportional-resonant)** statt einer PI-Schleife, dazu ein
  Burst/Skip-Verfahren am Nulldurchgang für kleine Sollwerte. Neue automatische **Lüfterregelung** (`Fan_UpdateDutyByTemperature`).
- **Leistungsgrenzen:** Die vom Nutzer setzbaren Grenzen sind unverändert — CAN 0x03/0x04 akzeptieren ≤ 2500 W, Modbus 44002/44003
  ≤ 2500 W, Sollwerte 0x56/0x58 ±2500 W. Aber die **Werkseinstellungen sind 3000 W** (D 2500), die eigenen Modbus-Register 43133/43134
  des Micro akzeptieren bis **3000 W**, drei Konstanten 3000,0f ersetzen 2500,0f in `Ctrl_MainControlIsr` (Reglergrenze bei der
  Netzzuschalt-Rampe, Basis der thermischen Leistungsreduzierung), und die Batteriekalibrierung / der Zeitplan in Betriebsart 3 nutzen
  ±3000 W. Register 45502 (Werkstest) nutzt −2400 W (D −2500 W).
- **CAN (Klasse 4, von der Control):** E3 ergänzt die Kommandos **0x08** (Parallel-/Betriebsmodus 0/1/2), **0x0A** (Abschalt-SOC
  6..79 %), **0x14** (Grenzen + SOC-Grenzen lesen) und **0x17** (Untergrenzen zweier neuer Energiezähler); die Telemetrie-Antwort 0x10
  wächst von 48 auf **56 Bytes** (zwei neue u32-Energiezähler). **0xCC (Datum/Uhrzeit) wird nicht ausgewertet — weder von E3 119 noch von D 116.**
- **0x04410110** (vom E3-BMS als „Strom vom Wechselrichter“ genutzt) ist **Frame Nr. 4 der Telemetrie-Antwort dieses Micro**
  (Funktion 0x10, Quelle 0x41, Ziel 0x01 = Control). Der Micro **sendet nie Standard-IDs (11 Bit)**, ist also nicht der Sender von
  0x305, 0x300 oder 0xCF (§4.1.5).
- **Warnwort:** E3 setzt **Bit 0 von `war1` (Modbus 36000)**, solange das Netzfreigabe-Flag 0 ist — D 116 schreibt das Warnwort nie.
  Alle Fehlerwort-Bits und Ereigniscodes sind ansonsten identisch mit D 116.
- Mehrere Namen aus D 116 erwiesen sich als irreführend und wurden in den Programmen D 115/116 und E3 korrigiert (§9.2).

### Antworten auf die Fragen zu diesem Image

| # | Frage | Antwort (Details) |
|---|---|---|
| a | Ausgewertete / ignorierte Klasse-4-Kommandos; wird 0xCC gebraucht? | Ausgewertet: 0x01–0x08, 0x0A, 0x10–0x14, 0x16, 0x17, 0x50–0x58, 0x60, 0xC1–0xC3, 0xCB, 0xCE. Ignoriert: 0x09, 0x0B–0x0F, 0x15, 0x18, 0x30–0x3E, 0x40–0x43, 0x59–0x5F, 0x61–0xC0, **0xC4, 0xCC**, 0xCF, 0xFB/0xFE/0xFF; Klasse-2/3-Frames werden im Hardwarefilter verworfen. Der Micro braucht 0xCC **nicht**: weder E3 119 noch D 116 haben einen 0xCC-Fall, die Zeit kommt aus der eigenen RTC (`RTC_ReadTime`). Der Wegfall von 0xCC/0xC4/0xCB-DLC2 in der E3 Control kostet auf Micro-Seite also nichts. (§4.1) |
| b | 0x04410110, 0x305, 0xCF, 0x300 | 0x04410110 = Telemetrie-Frame Nr. 4 (Bytes 24..31) der Micro-Antwort auf die 0x10-Abfrage der Control; das BMS liest mit: int16(Bytes 28..29) × 10000 / u16(Bytes 30..31 = Batteriespannung × 10) → mA. 0x305/0x300/0xCF sind 11-Bit-IDs; der Micro sendet nur 29-Bit-IDs → **nicht der Micro**. Lägen solche Frames auf dem Bus des Micro, würde sein Standard-ID-Filter (0x300/0x300) 0x300–0x3FF an den Klasse-4-Dispatcher geben (0x305 → Funktion 0x05 = Sichern + Neustart, 0x360 → Funktion 0x60 = Kalibriermodus). Daher ist es sehr unwahrscheinlich, dass der Pylontech-artige 0x305/0x35x/0x36x-Austausch des BMS auf dem internen Bus Control–Micro–BMS läuft. Am wahrscheinlichsten ein externer Batterieprotokoll-Anschluss; der Sender ist aus den vorhandenen Firmwares nicht bestimmbar. (§4.1.5) |
| c | Modbus-Handler, Register, Grenzen, Modi, Fehlerwörter | `Modbus_HandleRegisterAccess` (0x08016E34) bedient den **eigenen** RS485-Registerraum des Micro (40000, 41000/41001/41010/41100/41200, 42000/42010/42011, 43100–43134, 44000–44003, 44100, 45000, 45010–45019, 45200, 45300, 45400–45402, 45500–45502, 45600/45601, 45603/45604, 46000) — gleicher Umfang wie D 116; die Control leitet ihre Modbus-Register nicht dorthin weiter (Control-Register werden zu CAN-Kommandos). Grenzen 2500 W (CAN/Modbus), 0 wird angenommen und gespeichert (EEPROM 0x202/0x204), Laden ohne Bereichsprüfung, Werkseinstellung 3000 W. 45000: 0x55EE → Betriebsart 12, 0x55DD → Modi aus EEPROM neu laden, 0x55FF → Leistungsreduzierung aufheben + 2500 W. Fehlerwörter: gleiche Bits wie D 116 plus war1 Bit 0 (§6); die A-Bitnamen gelten nicht. (§4.2, §5.2, §6) |
| d | Unterschiede zu D 115/116 und A 1211 | §9 (Tabelle). Kernpunkte: 30 kHz PWM + PR-Stromregelung, intern/Werkseinstellung 3000 W gegenüber 2500 W Nutzergrenze, neue CAN-Kommandos 0x08/0x0A/0x14/0x17, 56-Byte-Telemetrie mit Energiezählern, war1 Bit 0, Lüfterregelung, USART1-Konsole über HAL-UART, kein dritter NTC; Standby-Latch und Schlafbehandlung identisch zu D. A 1211 ist ein anderes Design (NS800RT, kein RTOS, eigene PV-Stufe, 1450 W, Klasse-2-Handler, kein Modbus). |
| e | E3-spezifisch | Hardwarevarianten-Eingang (`g_hardwareVariant`, HRTIM-Basisoffset 340/410), CAN-Kommando für den Parallelmodus, Abschalt-SOC bis hinunter auf 6 %, Netznorm 4 mit eigenem Grenzwertsatz, Energiezähler, Lüfterkennlinie, Shell-Befehl `test` schreibt die Netznorm ohne Bereichsprüfung ins EEPROM. Aussage des Eigners: Der Venus E3 hat genau ein fest eingebautes Batteriepack, nicht erweiterbar — der Micro bedient genau ein BMS (Frames PF1–PF4, keine Pro-Pack-Daten). (§5, §9) |

**Offene Control-Fragen, von der Micro-Seite beantwortet**

- Vom E3-Micro ausgewertete Klasse-4-Codes: siehe Zeile (a). Von den Codes, die die E3 Control 151 noch sendet, erreichen
  **0x18, 0x30–0x3E, 0x40–0x43, 0xCF, 0xFB/0xFE/0xFF** im Micro keinen Handler (Klasse-2-Frames passieren nicht einmal den Filter).
- **Umsortierte „Inverter-Dirty-Bitmap“ (E3 Control 0x08009D74 / 0x08009D84):** Diese Bitmap hat **nichts** mit dem Micro zu tun. Die
  Control schreibt ihre vier Bytes `g_invRegPendingBits` per Bit-Bang-I2C als [Register, Wert] mit der Registertabelle
  {0x68, 0x6A, 0x6C, 0x6E} (E3 Control 0x08059AF5) — die Stellen-Register eines TM1650-artigen LED-/7-Segment-Treibers. Das E3-Layout
  (0x1C Bits, 7 pro Byte = 28 LED-Segmente; D 0x18 Bits, 6 pro Byte = 24) und die 2-Bit-Felder beschreiben also LED-Segmente der
  Frontblende; die zu D `Inverter_RegDirty_Mark_DischgVolt` / `_BatteryParams` byteidentischen Wrapper setzen nur LED-Segmente. Weder
  ein CAN-Frame noch ein Micro-Register ist beteiligt.

## 1. Binärer Fingerabdruck

| Merkmal | Wert |
|---|---|
| Datei | `119_micro_VNSE3-0_inv_app_0119_0326_ota_152726.bin` |
| Größe | 119.808 Bytes (0x1D400) |
| SHA-256 | `3c834d677e0d56868af3dab18bd7c1ae44ea6a2a776c70f3756991d6592eeb5d` |
| Linkadresse | `0x08004000`–`0x080213FF` (16-KB-Bootloader darunter) |
| MCU | STM32G474 (Cortex-M4F), FLASHSIZE-Register 0x1FFF75E0 für die OTA-Größenprüfung |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ-komprimiertes .data) |
| Bibliotheken | FreeRTOS (heap_4, 16-KB-Heap), letter-shell, ST-HAL (jetzt inklusive HAL-UART-Modul) |
| Vektortabelle | 0x08004000, initialer SP 0x20009AC0, Reset-Vektor 0x080042AD |
| Version | 0x77 (119) an 0x08010000, gefolgt von „AC02“; `ShellCmd_version` gibt SOFT_VERSION 119 und Build „Mar 26 2026 15:20:51“ aus |
| Trailer | Gerätekennzeichen „VNSEE3“ an 0x080213E0, Trailerwort 0x44443E54 an 0x080213FC (Bedeutung nicht bestimmt; D: „VNSD“, 0x44443331) |
| Changelog | v119: Zwangsladung unter 6 % SOC startete nicht — behoben |

Hinweis: Das Firmware-Archiv enthält zusätzlich einen VNSE3-0 Micro 116 (117.760 B), der hier nicht analysiert wurde.

**Scatter-Loading** (`Region$$Table$$Base` 0x08019250, `ScatterRegion[3]`):

| Quelle (Flash) | Ziel | Länge | Verfahren |
|---|---|---|---|
| 0x080195B0 (1120 B komprimiert) | 0x20000000 (.data) | 0x8FC | `__decompress` |
| 0x08019A10 | 0x10000000 (CCM-SRAM, Code + Daten) | 0x6F74 | `__scatterload_copy` |
| – | 0x200008FC (.bss) | 0x91C4 | Nullinitialisierung (endet am Stack-Top 0x20009AC0) |

## 2. Speicherkarte und Analysestand in Ghidra

Programm `/VNSE3/Micro/Micro_119_VNSE3-0_inv_app_0119_0326_ota_152726.bin`:

| Block | Bereich | Inhalt |
|---|---|---|
| `flash` | 0x08004000–0x080213FF | Code bis ~0x08018880, Strings/HAL-Tabellen, Region-Tabelle, `shellCommandTable` (51 Einträge @0x08019280), Ladeimages von .data und CCM, 0xFF-Füllung, Trailer |
| `.ccmram` | 0x10000000–0x10006F73 | CCM-Code (Veneers in den Flash 0x10000000–0x1000013F, Regel-ISRs, PLL, PR-Regler) und CCM-Daten |
| `.data` / `.bss` | 0x20000000–0x200008FB / 0x200008FC–0x20009ABF | Globale Variablen, FreeRTOS-Heap (`ucHeap` 0x200045BC, erster Block 0x200045C0) |
| Peripherie | APB1/APB2/AHB1/AHB2 (GPIO, ADC/DAC), FMC/QSPI, PPB | Registernamen aus der STM32G474-SVD |

Verifikations-Gates (`marstek_gates_micro.py`, Endstand): 0 undefinierte Bytes und 0 verwaiste Instruktionen (Flash + CCM), 0 Fluss in
Daten, 0 Funktionszeiger auf Nicht-Code, 0 reine Literal-Funktionen, 0 blockübergreifende Funktionskörper, 0 `FUN_`-/doppelte/temporäre
Namen, **0 unbenannte referenzierte RAM-Adressen**, 0 Default-Prototypen (Task-Einstiege `void f(void *pvParameters)`), No-Return-Flags
plausibel. Tags: `NAME_VERIFIED` an allen 708 Funktionen, `NAME_LOW_CONFIDENCE` an 74. Dekompilat-Export: 0 Fehlschläge, keine
unaufgelösten Sprungtabellen. Vorgehen: Namen, Prototypen, Kommentare und Globals wurden für byteidentische Segmente aus D 116
übernommen (Layout, Datentypen, Globals über ausgerichtete Instruktionsströme mit Nachbar-Delta-Konsistenzfilter; alle 546 identischen
Funktionen referenzieren dieselben Objekte wie in D), der Rest wurde aus dem E3-Code benannt.

## 3. FreeRTOS-Tasks

`Task_CreateAllAndStartScheduler` (0x0800F7C4) — dieselben sieben Tasks, Stacks und Prioritäten wie D 116:

| Task | Einstieg | Stack (Worte) | Prio |
|---|---|---|---|
| `vtask_led` | `Task_SystemHousekeepingLoop` 0x08016D34 | 128 | 16 |
| `vtask_time` | `Task_ModeRoundRobinLoop` 0x08016E14 | 256 | 15 |
| `vtask_modbus` | `Task_ModbusLoop` 0x08016D70 | 128 | 14 |
| `vtask_shell` | `Task_UpdateBms` 0x08016D88 | 512 | 9 |
| `vtask_prt` | `Task_AdcDebugPrintLoop` 0x08016D7C | 256 | 8 |
| `vtask_can` | `Task_CanTxLoop` 0x08016D18 | 256 | 6 |
| `vtask_can_receive` | `Task_CanRxLoop` 0x08016D28 | 256 | 7 |

Änderungen: Die CAN-RX-Queue fasst 20 Frames (D 40), und `Task_CanRxLoop` gibt pro Durchlauf einen Tick ab (D Busy-Loop).
Regel-Interrupts: `TIM7_ControlLoopCallback` (1 kHz) → `Ctrl_MainControlIsr` (CCM), `Inv_CurrentLoopIsr` (CCM, je PWM-Periode),
`HRTIM_TIMB_IRQn_IRQHandler` (Compare-Update der LLC-Stufe).

## 4. Schnittstellen

### 4.1 CAN (FDCAN1, 29-Bit-IDs)

**Filter** (`FDCAN1_ConfigFiltersAndMailboxes` 0x080058e8): Standard-ID 0x300/Maske 0x300 → FIFO0; extended 0x4000/0xFF00 und
0x4100/0xFF00 (Control Klasse 4, Knoten 0/1); extended **0x1800AA00/0x1FF0FF00** (BMS-Frames 0x18n0AAxx; D: 0x1801AA01/0x1FF0FFFF).
Globalfilter: nicht passende Standard- und Extended-Frames sowie Remote-Frames werden verworfen. Empfang: `HAL_FDCAN_RxFifo0Callback` →
`BMS_CAN_Parser` (Modus 3), `BMS_FwUpdate_CanHandler` (Modus 4) oder Queue → `CAN_RX_DispatchTask` → `EMS_Inverter_CAN_Dispatcher`.

**ID-Aufbau.** Funktionscode = ID-Bits 0–7; die Antwort geht an ID-Bits 16–23 der Anfrage. Vom Micro gesendete Frames:
`CAN_BuildArbID`: `Frameindex << 24 | 0x41 << 16 | Ziel << 8 | Funktion` (`g_canTxSrcAddr` = 0x41). Nutzdaten > 8 Bytes werden auf
Frames mit Index 1..n verteilt (plus Kopf-Frame 0 mit 4 Bytes, wenn die Länge kein Vielfaches von 8 ist). Alle Sendungen nutzen das
Extended-Format (`CAN_TX_SendFrame`).

#### 4.1.1 Klasse-4-Kommandos (`EMS_Inverter_CAN_Dispatcher` 0x0800597C)

| Funktion | Wirkung im E3 | gegenüber D 116 |
|---|---|---|
| 0x01 | Leistungssollwert (`Ctrl_SetPowerReferenceClamped`), nur Betriebsart 0; Echo | gleich |
| 0x02 | Freigabe-Flag des Backup-Ausgangs (EEPROM 0x300), nur Betriebsart 0 | gleich |
| 0x03 | max. Entladeleistung `g_powerLimitDischargeW`, Wert < 0x9C5 (≤ 2500 W) → EEPROM 0x204; Quittung auch beim Verwerfen | gleich |
| 0x04 | max. Ladeleistung `g_powerLimitChargeW`, < 0x9C5 → EEPROM 0x202 | gleich |
| 0x05 | Betriebsstatistik sichern, EEPROM 0x901 = 1, Watchdog-Neustart des Micro (von der Control beim Booten gesendet) | gleich |
| 0x06 | Standby-Latch `g_standbyLatch` = 1 (§5.3) | gleich |
| 0x07 | Sequenz-Handshake (`g_emsDispatchSeq` = Daten + 1) | gleich |
| **0x08** | Parallel-/Betriebsmodus `g_cfgParallelModeEnabled` = 0/1/2 (EEPROM 0x374; Wert 1 wird nicht gespeichert), nur Betriebsart 0; Quittung | **neu** (D ignoriert) |
| **0x0A** | Abschalt-SOC `g_cfgCutoffSocPct` = Daten (6..79 %), EEPROM 0x201, nur Betriebsart 0; Quittung | **neu** |
| 0x10 | Telemetrie, **56 Bytes** (§4.1.3) | D 48 Bytes |
| 0x11 | 8 B: Betriebsart, Standby-Latch, Batteriemodus, 0, Ladegrenze, Entladegrenze | gleich |
| 0x12 | 6 B Kalibrierstatus / Entlade- / Ladezeit | gleich |
| 0x13 | 4 B EMS-Lade-/Entladegrenzen (EEPROM 0x950/0x952) | gleich |
| **0x14** | 6 B: Ladegrenze, Entladegrenze, max. SOC (EEPROM 0x200), Abschalt-SOC (0x201) | **neu** |
| 0x16 | Untergrenzen der Lade-/Entladezähler (keine Antwort) | gleich |
| **0x17** | Untergrenzen der zwei neuen Energiezähler (0x20003EF3/EF7; keine Antwort) | **neu** |
| 0x50 | 0 → LLC/PFC Stopp; 1 → Betriebsart 1 + Dispatch 7 | gleich |
| 0x51–0x56 | Werkstests (Betriebsart 12): Piepton, Test-Unterkommandos, IO-Status, Leistungstest ±2500 W | gleich |
| 0x57 | Backup-Flag = (Betriebsart 1 && Daten == 1) | gleich |
| 0x58 | manueller Leistungssollwert ±2500 W, nur Betriebsart 1 | gleich |
| 0x60 | Betriebsart 4 (Batteriekalibrierung), Schritt = Daten | gleich |
| 0xC1 | EEPROM-Selbsttest, Antwort mit 0xC1 | gleich |
| 0xC2 | Debug-Level 1/2 | gleich |
| 0xC3 | Bytes in `g_shellRxQueue` (Eingang Shell-/BMS-Relais) | gleich |
| 0xCB | 8 B: Modellcode 0xD, Version 0x77, Hardwarevariante, 1 | gleiches Layout |
| 0xCE | CAN-Selbstupdate (§7) | gleich |
| andere (0x09, 0x0B–0x0F, 0x15, 0x18, 0x30–0x3E, 0x40–0x43, 0x59–0x5F, 0x61–0xC0, 0xC4, **0xCC**, 0xCF, 0xFB–0xFF) | ignoriert, keine Antwort | wie D 116 |

Der Switch über die Codes 0x01–0x17 wurde von 20 auf 23 Fälle vergrößert (TBB-Tabelle an 0x080059C4).

#### 4.1.2 BMS-Frames (`BMS_CAN_Parser` 0x08005928, identisch mit D)

`(id & 0xFFFFF) >> 16`: 1 = Packspannung/-strom + Temperatur/SOC (auch BMS-Lebenszeichen), 2 = Kapazität + Packinfo (Byte 7 =
Schlafflag des BMS), 3 = Ladespannungsgrenze + Entladestromgrenze (+ Lade-/Zwangslade-Anforderung), 4 = Fehler- und Warnwort.
Es wird nur ein BMS behandelt — keine Pro-Pack-Frames, passend zur Aussage des Eigners (genau ein festes Pack). Der
BMS-Kommunikations-Watchdog initialisiert FDCAN1 nach 10 s ohne Frames neu (`LED_TaskTick`).

#### 4.1.3 Telemetrie-Nutzdaten (Funktion 0x10, `g_emsTelemetryBlock` 0x2000391C, alle 400 ms aufgebaut)

Offsets 0x00–0x2F wie bei D 116: Zustand, Summer, Netzflag, Backup-Flag, Warnwort (+0x04), Fehlerwort 1 (+0x08), Fehlerwort 2
(+0x0C), Netzspannung ×10, Netzfrequenz ×10, Inselnetzspannung ×10, Netzfreigabe-Flag `g_gridPermitFlag` (+0x16), netzgekoppelte Leistung `g_gridTiedPower` (+0x18), Inselnetzleistung (+0x1A),
Batterieleistung `g_batteryPowerAvg` (+0x1C, int16, der Wert, aus dem das BMS seinen Strom berechnet), Batteriespannung ×10 (+0x1E), zwei Temperaturen ×10 (+0x20
LLC-Kühlkörper, +0x22 Wechselrichter — Reihenfolge gegenüber den D-116-Namen vertauscht), max./min. Leistungssollwert, Gesamt-Lade- /
Entladezähler (+0x28/+0x2C). **Neu: +0x30 / +0x34 = zwei u32-Energiezähler** (`g_energyPosWh` / `g_energyNegWh`, §5.5).

#### 4.1.4 Vom Micro gesendete Frames

Nur Antworten und Quittungen (0x01–0x08, 0x0A, 0x10–0x14, 0x51–0x58, 0x60, 0xC1, 0xC2, 0xCB, 0xCE), die Ausgabe der Debug-Queue
(`CAN_TX_FlushDebugQueue`) und das BMS-Firmware-Relais; kein periodisches Senden.

#### 4.1.5 Die BMS-IDs 0x04410110, 0x305, 0x300, 0xCF

- **0x04410110** = `4 << 24 | 0x41 << 16 | 0x01 << 8 | 0x10`: Frameindex 4 der Funktion-0x10-Antwort des Micro an die Control
  (Knoten 0x01) mit den Telemetrie-Bytes 24..31. Das E3-BMS 115 (`CAN_RX_AuxCommand_Handler`) rechnet int16(data[4..5]) × 10000 /
  u16(data[6..7]), also Telemetrie +0x1C × 10000 / (Batteriespannung × 10) → ein Strom in mA. Den Frame gibt es nur, solange die
  Control Funktion 0x10 abfragt; einen eigenen Sender hat der Micro nicht.
- **0x305, 0x300, 0xCF** sind Standard-IDs (11 Bit). Der Micro sendet nie Standard-IDs, ist also nicht ihr Sender. Sein
  Standard-ID-Filter (0x300/0x300) würde 0x300–0x3FF an den Klasse-4-Dispatcher geben (unteres Byte = Funktionscode): 0x305 → 0x05
  (Sichern + Neustart), 0x351 → 0x51, 0x356 → 0x56, 0x360 → 0x60 (Batteriekalibrierung). Ein Pylontech-artiger Austausch (Anfrage
  0x305, BMS antwortet mit 0x351/355/356/359/35C/360–363/36A/36B) auf demselben Bus würde den Wechselrichter also stören; er ist am
  wahrscheinlichsten für einen externen Anschluss gedacht (Fremd-Wechselrichter oder Servicewerkzeug). Ob CAN1 des E3-BMS nur am
  internen Bus hängt, ließ sich aus der Firmware nicht klären.

### 4.2 RS485 / Modbus (eigener Registerraum des Micro)

`Modbus_HandleRegisterAccess` (0x08016E34) mit `Modbus_ProcessRequest` (0x0800F03C) bedient den eigenen RS485-Modbus-Slave des Micro
(Adresse EEPROM 0x101). Registerumfang und Bedeutung sind identisch mit D 116 (siehe D-Dokument §4.2), mit diesen E3-Unterschieden:

| Register | E3 | D 116 |
|---|---|---|
| 43133 / 43134 (`g_ratedPowerConfigA/B`, EEPROM 0x800/0x802) | 0..**3000** | 0..2500 |
| 45502 (Werkstest, Betriebsart 0, Wert 1) | `Ctrl_SetPowerReferenceClamped(−2400.0)` | −2500.0 |
| 44000 / 44001 | Wert/10 → max. SOC / Abschalt-SOC (800..1000 / 120..300) | gleich (`g_cfgMaxSocPct` / `g_cfgCutoffSocPct`) |

Unverändert: 40000 (Kalibrierung 0x55AA/0x55A1–A6), 41000/41001 (0x55AA), 41010 (genau 800 → Reduzierungsflag + 800 W, genau 2500 →
aufheben), 41100, 41200, 42000 (0x55AA / 0x55BB Betriebsart neu laden), 44002/44003 (≤ 2500 W, 44003 ≤ 800 W bei aktiver Reduzierung),
44100 (Netznorm 0..4), 45000 (0x55EE / 0x55DD / 0x55FF), 45010–45019 (Zusatzblock, EEPROM 0x21), 45200/45300/45400–45402/45500–45502/
45600/45601 (Werkstests), 45603/45604, 46000 (0x5100 Debug, 1234/2345 BMS-Update).
Eigenheit (auch in D 116): Schreiben von 44000 legt Wert/10 im Max-SOC-Byte ab, speichert dieses Byte aber nach EEPROM 0x201 (Platz
des Abschalt-SOC).

Diese Register sind nur am eigenen RS485-Anschluss des Micro erreichbar; die Control leitet ihre Modbus-Register nicht an den Micro
weiter — Control-Register (z. B. 44002/44003, 45000–45010) werden in CAN-Kommandos übersetzt (0x04/0x03, 0x50, 0x06, 0x52–0x58).

### 4.3 Debug-Shell (letter-shell, USART1 115200 Bd)

`shellCommandTable` @0x08019280, 51 Einträge (D: 53): 40 Befehle plus Benutzereintrag und Tastenbelegungen. Befehle: `test`, `test_1`,
`version`, `log_data`, `log_list`, `set_n`, `sr_set` (direkt `Ctrl_SetDeadTimeOffset`), `log_err`, `log_clear`, `log_open`, `prt`,
`help_prt`, `reset`, `pf`, `pf_set`, `pf_power`, `power_cfg`, `rtos_status`, `io_show`, `io_set`, `fan_set`, `set_power`, `update`,
`update_bms`, `update_bms2`, `bat_mode`, `reset_memory`, `backup_mode`, `par_mode`, `set_mode`, `qpr`, `pi`, `show_power`, `alarm`,
`dac_high`, `dac_low`, `set_sr`, `old_ate`, `help`, `clear`.
Unterschiede zu D: `vol_pi` und `show_pi` fehlen; `qpr` setzt jetzt den PR-Regler (`Ctrl_SetPrControllerTarget`, „kp kr“), `power_cfg`
führt `Ctrl_HandleLlcPfcTestCommand(Modus, Wert)` aus (jedes LLC/PFC/Wechselrichter-Testkommando), `test` schreibt das Netznorm-Byte
**ohne Bereichsprüfung** nach EEPROM 0x206, ohne die Grenzwerte neu zu laden (D: nur Debug-Ausgabe); die Hilfe zu `set_mode` ergänzt
„4 ATE-Alterung“. Die Konsolen-UART wird jetzt von `Uart_InitUsart1Console` initialisiert (HAL-UART, FIFO-Modus), wo D einen leeren
Stub aufruft.

### 4.4 I2C-EEPROM-Belegung

Wie D 116, mit diesen E3-Werten: 0x000 Versionsflag (3 = gültig), 0x001 ID-Block (0x2F B, „AC02“, Modell 0xD, Marker 0x77), 0x021
Zusatzregisterblock (20 B), 0x080 Zeitstempel, 0x090 Reduzierungsflag, 0x100–0x104 (Standby-Latch, Modbus-Adresse, Batteriemodus, …),
0x150 Modusanzeige-Zähler, 0x200–0x206 (max. SOC, Abschalt-SOC, Ladegrenze, Entladegrenze, Netznorm), 0x300 Block 0x7C B
(Backup-Flag, 0x374 Parallelmodus), **0x500 Statistikblock 0x40 B** (D 0x30; enthält jetzt die Energiezähler), 0x800/0x802
Nennleistungskonfiguration, 0x900/0x901 Debug-Level / Boot-Flag, 0x950/0x952 EMS-Grenzen.
Werkseinstellungen (`Cfg_LoadFromEeprom` 0x08006B2C, angewendet bei gespeichertem Marker < 108): „AC02“, Modell 0xD, Marker 0x77,
max. SOC 100, Abschalt-SOC 12, Modbus-Adresse 1, Batteriemodus 1, **Lade-/Entladegrenze 3000 W**, Netznorm 0. Der Grenzwertblock wird
ohne Bereichsprüfung geladen; eine per CAN 0x03/0x04 geschriebene 0 überlebt also einen Neustart.

## 5. Regelung und Leistungsteil

### 5.1 PWM, Abtastung und Regler

- Wechselrichterbrücke 30 kHz PWM (HRTIM-Periode 45333 bei 1,36 GHz; D 48572 = 28 kHz), Regelabtastzeit 33,33 µs (D 35,71 µs),
  LLC-Timer-B-Startperiode 10625 = 128 kHz (D 10793 = 126 kHz), HRTIM-Basis-Compare-Offset jetzt ein RAM-Wert (`g_hrtimBaseOffset`,
  340 bzw. 410 mit dem Hardwarevarianten-Eingang; D Konstante 280).
- Netzstrom: PR-Regler (`Ctrl_PrControllerDesign` Ts, 6000,0, 50 Hz; kp 9,0, Grenzen ±370; `Ctrl_PrControllerStepClamped`, nur bei
  freigegebenen Brückenausgängen, `Inv_IsOutputEnabled`) statt der PI-Schleife von D (3,0/0,2/±500); Burst/Skip am Nulldurchgang für
  kleine Sollwerte; Überstrom (|i| > 30 A) erst nach 3 aufeinanderfolgenden Abtastungen (D sofort); HV-Bus > 530 V (570 V im LLC-Zustand
  0x0E) als getrennte Prüfung.
- ADC: ADC2 hat 4 Ränge (D 5): zwei NTCs bleiben (CH5 → `g_tempInverterC`, CH12 → `g_tempLlcHeatsinkC`, beide NTC-Tabelle 2); neue
  Strom-/Spannungsskalierung (Netzstrom 0,0183 A/LSB, Batteriespannung (Rohwert − 2048) × 0,0555 V).
- DAC-Komparatorfenster 2048 ± 1366 Counts (D ± 1466).

### 5.2 Betriebsarten (`g_workMode`, EEPROM 0x301)

Unverändert gegenüber D 116 (`ShellCmd_set_mode` ist byteidentisch): 0 Standard (CAN-Sollwert 0x01), 1 manuell (CAN 0x57/0x58),
2 Debug, 3 Zeitplan, 4 Batteriekalibrierung / ATE (CAN 0x60, Modbus 40000), 12 Werksmodus (Modbus 45000 = 0x55EE, CAN 0x51–0x56),
13 Spannungsquellenmodus. In den Betriebsarten 3/4 kommandiert `Bat_Mode3RoundRobinTimer` jetzt ±3000 W (D ±2500 W), der Timeout von
Schritt 0 beträgt 7200 s (D 9000 s).

### 5.3 Standby / Schlaf

Der Micro hat keinen Low-Power-Modus der MCU. `g_standbyLatch` wird durch CAN 0x06
(Control-Register 45002) gesetzt oder wenn das BMS sein Schlafflag meldet (PF2 Byte 7); solange es 1 ist, führt `Ctrl_MainControlIsr`
den Leistungsteil in den Stoppzustand 200, Telemetrie-Byte 0 liest 0. `LED_TaskTick` löscht es wieder, wenn das Schlafflag des BMS 0
ist und die BMS-Frames jünger als 20 s sind (EEPROM 0x100). CAN 0x05 sichert die Zähler und startet den Micro neu. Identisch mit D 116.

### 5.4 Niedriger SOC und Zwangsladung

`Inv_GridConnectionSupervisor` (0x100003A8) setzt das Low-SOC-Stoppflag (`g_lowSocStopFlag`) erst nach mehr als
15000 Supervisor-Aufrufen mit SOC < 6 % (D: sofort bei SOC < 6 %; der Zähler wird durch einen höheren SOC nicht zurückgesetzt) und
gibt es bei SOC ≥ Abschalt-SOC − 1 frei (D: − 2); ein zweites Flag löst zusätzlich aus, wenn der Backup-Modus aktiv und der SOC > 7 %
ist. Zusammen mit dem neuen CAN 0x0A (Abschalt-SOC ab 6 % akzeptiert) sind das die Änderungen rund um die 6-%-Schwelle; welche davon die
angekündigte Korrektur „Zwangsladung unter 6 % SOC“ ist, ließ sich statisch nicht beweisen.

### 5.5 Energiezähler (neu)

`Ctrl_MainControlIsr` summiert in jedem Regeltakt die positive und negative Leistung; alle 1000 Takte geht der Mittelwert an
`Energy_AddPositiveWs` / `Energy_AddNegativeWs` (0x200004A8/AC), und `Time_UpdateOperatingHoursAndFaultLog` überträgt sie je 3600 in
die u32-Zähler `g_energyPosWh` / `g_energyNegWh` (0x20003EEB/EEF), nach unten begrenzt durch CAN 0x17, gesendet in Telemetrie
+0x30/+0x34 und gespeichert im 0x40-Byte-Statistikblock. Welcher physikalische Fluss (Laden oder Entladen der Batterie) in welchem
Zähler steht, ist nicht bewiesen. Das passt zum Changelog-Eintrag der E3 Control „kumulierte Lade-/Entladeenergie der Batterie (nur VE3)“.

### 5.6 Lüfter

`Fan_UpdateDutyByTemperature` (0x08016120; das D-Gegenstück `Fan_TickManualOverrideTimer` zählt nur den manuellen Override herunter): TIM1-Tastgrad 400 bis 40 °C am heißesten NTC,
danach 300 − 30 × (T − 40), begrenzt auf 0..400; eine manuelle Vorgabe (Shell `fan_set` 3600 s, Modbus 100 s) setzt die Regelung aus.

### 5.7 Netznormen (`Grid_Protection_SetLimits` 0x08005290, Modbus 44100 / EEPROM 0x206)

Normen 2 und 3 unverändert; Standard (0/1): OV1 263 V (D 265), OV4 256 V (D 265), untere Spannungsgrenze 190 V (D 197,8); **Norm 4 hat
einen eigenen Satz** (51,0/48,0 Hz, 50,1/49,9 Hz, untere Grenze 207 V, Zeiten 200, Halte-/Wiederzuschaltzeit 65000).

## 6. Schutzfunktionen und Fehlercodes

Fehlerblock `g_protErrorWord1` 0x20001A28 (err1, Modbus 36100), `g_protErrorWord2` 0x20001A2C (err2, 36102), `g_protWarningWord1`
0x20001A30 (war1, 36000). Eine vollständige Suche aller Setz-/Löschstellen ergibt genau den Satz von D 116 plus ein E3-Bit:

| Wort / Bit | Ereignis | Bedingung | Funktion |
|---|---|---|---|
| err1 0 / 1 | 0x2C | Netz-Effektivspannung zu hoch / zu niedrig | `Ctrl_LlcVoltageRegulationAndProtection` |
| err1 2 / 3 | 0x2A | Netzfrequenz zu hoch / zu niedrig | dieselbe |
| err1 4 | 0x2D | Netz-Spitzenspannung | dieselbe |
| err1 7 | 0x2E | Kurvenformfehler | dieselbe |
| err1 20 | 0x33 (Löschen 0x18) | BMS-Kommunikation verloren | `Inv_GridConnectionSupervisor` |
| err1 21 | 0x3B (Löschen 0x0B) | BMS meldet Fehler | dieselbe |
| err2 0 / 1 / 3 / 4 | 0x36 | Hardware-Schutz-Latch (Bit 4 = Sammelbit) | `Prot_LatchHwFaultToErrorWord` |
| err2 5 | 0x3F | Spitzenstrom | `Inv_CurrentLoopIsr` |
| err2 6 | 0x34 | HV-Busspannung | dieselbe |
| err2 7 | 0x13 | Abschaltung durch die Zustandsmaschine: ≥ 1000 Regeltakte im aktuellen Zustand bei Flag [r4+0x0D] = 0 → Fehlerlog, Zustand 200, Neuversuch nach 60 s | `Ctrl_MainControlIsr` (gesetzt bei 0x10003ffe) |
| err2 8 | – | Überlast-Integrator Inselnetz | `Ctrl_RampDcBusReferenceWithFaultCheck` |
| err2 9 | 0x29 | Zustandsautomaten-Fehler | `Ctrl_MainControlIsr` |
| err2 11 | 0x35 | Zeitüberschreitung beim Zustandswechsel | dieselbe |
| err2 16 | – | Selbsttest fehlgeschlagen | `main` |
| **war1 0** | – | **gesetzt, solange das Netzfreigabe-Flag (Telemetrie +0x16) 0 ist, gelöscht, sobald es wieder gesetzt ist** | `Ctrl_LlcVoltageRegulationAndProtection` (**nur E3**) |

Die Texte des Fehlerprotokolls (`Log_PrintFaultEntry`) und die Ereigniscodes sind identisch mit D 116 (siehe D-Dokument §6). Die
Bitnamen des Venus A (`VNSA_Micro_FW_v1211_Vollanalyse_de.md` §6) gelten **nicht**. Schwellen haben sich nur dort geändert, wo in §5
vermerkt (Netznormen, Überstrom mit 3 Abtastungen, Stufen des DC-Bus-Rampentimers in `Ctrl_RampDcBusReferenceWithFaultCheck`).

## 7. OTA / Update

Unverändert gegenüber D 116 (alle OTA-Funktionen bis auf Adressen byteidentisch): YMODEM-Selbstupdate über die Debug-UART (`update`),
CAN-Selbstupdate über Funktion 0xCE (`OTA_SelfUpdate_StateMachine`, CRC16-Modbus, Zwischenablage ab 0x08044000, Deskriptor an
0x08040000, Größenprüfung gegen FLASHSIZE), BMS-Firmware-Relais über CAN (`BMS_FwUpdate_StateMachine`) oder RS485
(`BMS_FwUpdateStateMachineTask`, `update_bms`/`update_bms2`). Das Trailer-Kennzeichen „VNSEE3“ dient vermutlich dem Bootloader oder der
OTA-Prüfung der Control dazu, Images anderer Modelle abzulehnen (Bootloader nicht verfügbar). Keine Signatur, nur CRC.

## 8. Sicherheitshinweise

Wie D 116: nicht authentifizierte Werks-/Testmodi über das Modbus des Micro (45000, 46000, 41010, 0x55AA-Codes) und über
Klasse-4-CAN-Kommandos (0x50–0x58, 0x60); letter-shell-Benutzer mit leerem Passwort; OTA ohne Signatur. Neu: Shell `power_cfg` öffnet
jedes Testkommando des Leistungsteils, Shell `test` schreibt ein ungeprüftes Netznorm-Byte ins EEPROM. Der Micro hat keinen eigenen
Netzwerkzugang.

## 9. Unterschiede

### 9.1 Gegenüber D 115/116 und A 1211

| Thema | E3 119 | D 116 (D 115 gleich, sofern nicht vermerkt) | A 1211 |
|---|---|---|---|
| MCU / OS | STM32G474, FreeRTOS, CCM-Code | gleich | NS800RT Cortex-M7, ohne RTOS |
| Übereinstimmung | – | 73,5 % identisch (D 115: 70,5 %) | 2,3 % |
| Versionskennung / Trailer | 0x77 „AC02“ / „VNSEE3“ | 0x74 „VD01“ / „VNSD“ | 1211 / „VNSA“ |
| PWM / Regelung | 30 kHz, PR-Netzstromregler | 28 kHz, PI | 28 kHz, PI + QPR |
| PV | keine | keine (separater Klasse-2-MPPT-Knoten) | 4 PV-Eingänge auf der Platine |
| CAN Klasse 4 | + 0x08, 0x0A, 0x14, 0x17; Telemetrie 56 B | Telemetrie 48 B | ignoriert 0x08/0x0A/0x14/0x17; zusätzlich Klasse 2 |
| 0xCC Datum/Uhrzeit | nicht behandelt | nicht behandelt | Klasse 2 behandelt |
| Lade-/Entladegrenze (CAN/Modbus) | ≤ 2500 W | ≤ 2500 W (D 115: 2200 W) | ≤ 1450 W |
| Werkseinstellung Grenze | **3000 W** | 2500 W | 1450 W |
| Interne Rampen-/Reduzierungskonstanten | 3000,0 | 2500,0 (D 115: 2200,0) | – |
| Leistung Kalibrierung / Betriebsart 3 | ±3000 W | ±2500 W | ±2200 / ±1350 W |
| Grenze 43133/43134 | 3000 | 2500 (D 115: 2200) | kein Modbus |
| Testsollwert 45502 | −2400 W | −2500 W (D 115: −2200 W) | – |
| Warnwort | Bit 0 genutzt | nie geschrieben | nie geschrieben |
| Abschalt-SOC per CAN | 0x0A, 6..79 % | nicht möglich | – |
| Energiezähler | 2 neue u32 (Telemetrie, CAN 0x17) | – | Wh-Zähler in der Telemetrie |
| Netznorm 4 | eigener Grenzwertsatz | fällt in den Standardsatz | – |
| NTCs | 2 | 3 | 3 |
| Lüfter | automatische Kennlinie | nur manuell | – |
| Konsole | HAL-UART-Init USART1 | leerer Init-Stub | LPUART |
| Update-Wege | YMODEM, CAN 0xCE, BMS-Relais | gleich | nur CAN 0xCE |
| Packs | ein BMS (Eigner: ein festes Pack) | ein BMS-Framesatz | ein BMS-Framesatz |

### 9.2 Nach dem E3-Vergleich korrigierte Namen (in D 115/116 und E3 119 übernommen)

Diese D-116-Namen waren falsch; sie sind in den Ghidra-Programmen von D 115/116 korrigiert, und E3 119 verwendet dieselben Namen.

| Früherer D-Name | Name jetzt (D 115/116 und E3 119) | Grund |
|---|---|---|
| `I2C_SetBufferPointer` (+ `_veneer`) | `HAL_GPIO_WritePin` (+ `_veneer`) | schreibt GPIOx_BSRR/BRR |
| `GPIO_ConfigPin_Output` / `_AF` / `_Input` | `GPIO_ConfigPin_Input` / `_ExtiRising` / `_OutputPP` | Moduswerte 0 / 0x110000 / 1 |
| `HAL_UART_TxCpltCallback` | `HAL_UARTEx_RxEventCallback` | wird mit einer Länge aus dem Receive-to-Idle-Pfad aufgerufen |
| `CAN_ProcessRxFilterTable` | `shellHandler` | Tastenhandler der letter-shell |
| `Wdg_DecrementSafetyCounter` | D: `Fan_TickManualOverrideTimer`; E3: `Fan_UpdateDutyByTemperature` | zählt den manuellen Lüfter-Override-Timer herunter (Low-Byte von `g_wdgSafetyCounter`); der E3-Code ist zusätzlich die Lüfterregelung |
| `Prot_MonitorSyncLossTimeout` | `Prot_MonitorHvBusLevelFlags` | HV-Bus-Pegelflags (355 / 340 V und 480 / 520 V, Entprellung über 1000 Aufrufe) |

Globale Variablen (gleiches Objekt, gleiche Adressentsprechung wie in D 116):

| Früherer D-Name | Name jetzt | Bedeutung |
|---|---|---|
| `g_battPower` | `g_gridTiedPower` | netzgekoppelte (AC-Anschluss) Leistung, Telemetrie +0x18 |
| `g_ctrlLoopDebugFloat_e00` | `g_batteryPowerAvg` | Batterieleistung (DC), tiefpassgefiltert, Telemetrie +0x1C |
| `g_emsPairedFlag` | `g_standbyLatch` | Standby-Latch (EEPROM 0x100) |
| `g_gridProtCfgParam0` / `g_gridProtCfgParam1` | `g_cfgMaxSocPct` / `g_cfgCutoffSocPct` | max. SOC / Abschalt-SOC (EEPROM 0x200 / 0x201) |
| `g_llcStatusFlag` | `g_gridPermitFlag` | Netzfreigabe-Flag, Telemetrie +0x16 |
| `g_llcResonantGain` | `g_gridFrequencyHz` | Netzfrequenz |
| `g_llcHvBusVoltage` | `g_pllFrequencyEstimate` | PLL-Frequenzschätzung |
| `g_gridSyncReadyFlag` | `g_lowSocStopFlag` | Low-SOC-Stoppflag |
| `g_syncLossFlagA` / `g_syncLossFlagB` | `g_hvBusBelow480Flag` / `g_hvBusAbove355Flag` | HV-Bus-Pegelflags |
| `g_syncLossTimer_3b8` / `g_syncLossTimer_3bc` | `g_hvBusBelow480Timer` / `g_hvBusAbove355Timer` | deren Entprellzähler |
| `g_canFilterTableBase` / `g_canFilterTableCount` / `g_canAddrMatchAccum` | `g_shellCommandListBase` / `g_shellCommandListCount` / `g_shellKeyValue` | Befehlsliste und Tastenakkumulator der letter-shell |
| `g_bmsFilterRxByte` / `g_bmsCanQueue` | `g_shellRxByte` / `g_shellRxQueue` | Shell-Eingangsbyte und -Queue |

### 9.3 Erneute Prüfung nicht funktionaler Register (Pflichtprüfung 8)

Eine Register-CSV für den E3 gibt es noch nicht, `list_recheck_registers.py E3` ist daher nicht anwendbar. Die Micro-bezogenen Einträge
der D-Liste wurden gegen E3 119 geprüft: 30010 (Betriebsart, bedingt) — gleiches Micro-Verhalten; **36000 Warncode — beim E3 jetzt
funktional** (war1 Bit 0, §6); 38000–38014 (CAN 0x40–0x43) — weiterhin kein Sender/Empfänger im Micro; 45020–45024 (Klasse-2-Frames) —
weiterhin ausgefiltert; 46500–46544 (CAN 0x30–0x3E) und 47001 (CAN 0x18) — weiterhin kein Handler; 41001 / 45013–45019 — die eigenen
Register des Micro sind weiterhin nicht über die Control erreichbar.

## 10. Firmware-Eigenheiten (gemeldet, nicht behoben)

1. CAN 0x03/0x04 und Modbus 44002/44003 begrenzen auf 2500 W, die Werkseinstellung ist aber 3000 W — nach einem Werksreset ist die
   gespeicherte Grenze größer als jeder Wert, der zurückgeschrieben werden kann.
2. CAN 0x03/0x04 akzeptieren 0 W und speichern den Wert; beim Laden des Konfigurationsblocks gibt es keine Bereichsprüfung.
3. Modbus 44000 speichert das Max-SOC-Byte nach EEPROM 0x201 (Platz des Abschalt-SOC) — auch in D 116.
4. CAN 0x08 Wert 1 (Parallelmodus 1) wird nicht gespeichert (0 und 2 schon).
5. Shell `test` schreibt ein ungeprüftes Netznorm-Byte nach EEPROM 0x206 und lädt die Grenzwerte nicht neu.
6. Das ADC-Oversampling-Setup schreibt die Oversampling-Felder von ADC2 in das hADC5-Handle (Copy-Paste-Fehler, auch in D 116; mittlere
   Sicherheit, aus dem Strukturlayout abgeleitet).
7. Ein Standard-ID-Frame im Bereich 0x300–0x3FF auf dem Bus des Micro würde als Klasse-4-Kommando ausgeführt (§4.1.5).

## 11. Offene Punkte

1. Welche Codeänderung die v119-Notiz „Zwangsladung unter 6 % SOC“ umsetzt (§5.4).
2. Physikalische Bedeutung der zwei neuen Energiezähler (positive/negative Leistung → Laden/Entladen?).
3. Sender von 0x305/0x300/0xCF und ob der Standard-ID-Austausch des E3-BMS auf einem getrennten physischen Bus läuft.
4. Bedeutung des Hardwarevarianten-Eingangs (`g_hardwareVariant`, HRTIM-Offset 340/410, zweiter Digitalausgang) — zwei Platinenrevisionen?
5. Trailerwort 0x44443E54 und die Modellprüfung des Bootloaders (Bootloader nicht verfügbar).
