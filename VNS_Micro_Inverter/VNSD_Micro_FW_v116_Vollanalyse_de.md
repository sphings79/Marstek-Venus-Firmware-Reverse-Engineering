# Micro-FW v116 (Venus D) — Vollanalyse

> 🇬🇧 English version: [VNSD_Micro_FW_v116_Full_Analysis.md](VNSD_Micro_FW_v116_Full_Analysis.md)

**Stand:** 2026-09-30.

**Englisch:** `VNSD_Micro_FW_v116_Vollanalyse_de.md` (wird nach dieser Fassung erstellt) ·
**Diff zum Vorgänger v115:** `VNSD_Micro_Diff_v115_v116_de.md` (wird nach dieser Fassung erstellt) ·
**Dekompilat/Listing:** `Dekompilate/VNSD_Micro_116/` (lokal, nicht im Repo) (693 Funktionen, 925 RAM-Globals, `listing/` mit
Flash- und CCM-Rohdisassemblat + `globals_and_registers.txt`)

## 0. Kurzfassung

- Image-Basis **0x08004000** (16-KB-Bootloader darunter), Image-Größe 0x1C400 Byte (115 712 B),
  MCU **STM32G474** (HRTIM `0x40016800`, FDCAN, FLASHSIZE-Register `0x1FFF75E0`, offizielles SVD
  angewandt), Keil ARMCC, FreeRTOS (RVDS/ARM_CM4F, `heap_4`, 16-KB-Heap), letter-shell.
  `0x6B50` B zeitkritischer Code (Haupt-Regel-ISR, Stromregelkreis, PLL) laufen aus **CCM-SRAM**
  `0x10000000`, erreicht über 10-Byte-Veneers in Flash ab `0x08004960`.
- Herstell-Changelog nennt für v116 nur „VD 2500w". Das lässt sich **nicht** auf eine einzelne
  Codestelle zurückführen, sondern betrifft **sechs getrennte Stellen** — beim Aufbau des
  Diff-Dokuments `VNSD_Micro_Diff_v115_v116_de.md` wurde die Liste gegenüber dem ersten Durchgang dieser
  Sitzung noch einmal korrigiert: `Bat_Mode3RoundRobinTimer` kam als real bestätigte Stelle hinzu,
  die zunächst vermutete Änderung in `Ctrl_ResetControlState` erwies sich dagegen bei genauem
  Bytevergleich als **nicht real** (beide Versionen laden dort bereits 2500.0f, s. §5) und entfällt.
  Alle sechs Stellen heben denselben Lade-/Entlade-Leistungs-Grenzwert von 2200 W auf 2500 W an
  (Details in §4.1/§4.2/§5):
  1. `EMS_Inverter_CAN_Dispatcher` (`0x08005940`), CAN-Kommandos 0x03/0x04 (EMS setzt Lade-/
     Entladelimit): Bereichsprüfung `< 0x899` (2201) → `< 0x9c5` (2501).
  2. `Modbus_HandleRegisterAccess` (`0x080164e8`), Register 43133/43134 (`g_ratedPowerConfigA`/`B`,
     Nennleistungs-Konfiguration): Obergrenze `0x898` (2200) → `0x9c4` (2500).
  3. Dieselbe Funktion, Register 45502 (Werks-Testbefehl „Entlade-Referenz"):
     `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)` → `(-2500.0)`.
  4–5. Im CCM-Code von `Ctrl_MainControlIsr` (zweimal, PFC/LLC-Hochlauframpe und thermisches
     Derating) wird der Float-Literal `0x45098000` (2200.0f) durch `0x451C4000` (2500.0f) ersetzt.
  6. `Bat_Mode3RoundRobinTimer` (`0x08011834`, Mehrfach-Pack-Round-Robin-Timer für Batteriemodus 3):
     zwei Stellen setzen `g_batPowerRefW` auf `±2200.0` bzw. `±2500.0` — ebenfalls angehoben.
  Der **Echtzeit-Hartgrenzwert** für den unmittelbaren Leistungssollwert (CAN-Kommandos 0x56/0x58,
  Prüfung `wert+0x9c4<0x1389`, ≙ ±2500 W) war dagegen in **beiden** Versionen bereits 2500 W.
  v116 behebt also eine Inkonsistenz: der scharf geschaltete Echtzeit-Sollwert durfte schon in v115
  bis 2500 W gehen, mehrere Konfigurations-/Init-/Rampen-Pfade deckelten das tatsächlich nutzbare
  Limit aber weiterhin bei 2200 W.
- **Korrektur eines weiteren Fehlschlusses der vorigen Sitzung:** Die dort als „echter neuer
  Programmpfad in v116" gewertete, von `Inv_BuildTelemetryBlock` aufgerufene Hilfsfunktion bei
  `0x0800699c` (Debounce alle ~10 Telemetriezyklen, EMS-Dispatch-Sequenz `g_emsDispatchSeq` 0–2,
  `Ctrl_HandleLlcPfcTestCommand_veneer`, EEPROM-Schreiben von Derating-Flag/Testmodus) ist **keine
  neue Funktion**. Byte-für-Byte-Vergleich der Rohdisassemblate zeigt identischen Code zur
  v115-Funktion `EMS_ResetMemoryByLevel` (`0x080069a4`, 86 B) — v116 hat dieser Stelle im
  automatischen Benennungslauf lediglich keinen Namen mehr zugeordnet (Ghidra registriert sie dort
  nicht einmal als eigene Function, sondern nur als Codeadresse `SUB_0800699c`, daher taucht sie auch
  nicht in der 693er-Funktionszählung auf). Die umgebende Debounce-/Dispatch-Logik in
  `Inv_BuildTelemetryBlock` ist ebenfalls zeilengleich zu v115 (nur Variablennamen `uVar2`/`uVar3`
  unterscheiden sich). Es gibt an dieser Stelle **keine** v115→v116-Verhaltensänderung — siehe
  korrigierte Fassung in §11.
- `Stub_CallHandlerOffset28` (`0x0800ffee`) ruft in v116 real `HAL_UART_AbortCpltCallback` auf; in
  v115 war die gleichnamige Funktion (`0x0800fff0`) ein reiner `return`-Stub.
- **Korrektur eines eigenen Fehlschlusses in dieser Sitzung:** `EEPROM_SaveOperatingStats`
  (`0x080129d4`) wurde zunächst für eine neue v116-Funktion gehalten (sie ersetzt in v116 vier von
  fünf Aufrufstellen eines v115-Stubs `Stub_EmptyNoOp2`). Byte-genauer Vergleich zeigt: der
  4-Byte-Trampolin-Code (Sprung zu `EEPROM_WriteVerify(0x500,&g_statsLastDay,0x30)`) existierte in
  v115 **unverändert** unter derselben Adresse — nur der automatische Benennungslauf hatte ihn dort
  fälschlich in den Sammelnamen `Stub_EmptyNoOp2` einsortiert, während v116 ihn korrekt eigen
  benannt hat. Reine Namenskorrektur, **keine** neue Persistenzfunktion in v116 (siehe §11).
- Ebenfalls widerlegt: ein Rohdiff-Fund, wonach `ShellCmd_bat_mode`/`ShellCmd_pf_set` in v116 eine
  zusätzliche Debug-Ausgabe bekommen hätten — beide Funktionen sind byte-identisch zu v115, der Fund
  war eine Fehlzuordnung von Nachbarcode durch das Diff-Werkzeug (§11).
- **Wichtige Korrektur der Auftrags-Annahme zu den Modbus-„Magic Codes"**: Register 45000 kennt die
  Werte `0x55DD`/`0x55EE`/`0x55FF`, aber keiner davon erzwingt ein 800-W-Derating — `0x55FF` **löscht**
  das Derating-Flag und setzt das Entladelimit auf 2500 W zurück, `0x55EE` schaltet nur in den
  Werksmodus, `0x55DD` liest (nicht schreibt, s. u.) drei Modus-Bytes aus dem EEPROM zurück. Das
  tatsächliche 800-W-Zwangsderating wird an **anderer Stelle** ausgelöst: durch das **wörtliche**
  Schreiben des Werts `800` auf Register 41010 oder 44003 (§4.2, §8).
- **Namensbefund bestätigt und Offener Punkt aufgelöst:** Die im frischen Export korrekt benannte
  Funktion `EEPROM_ReadRegister` (`0x08006ce8`, vormals in dieser Analyse irrtümlich
  `EEPROM_WriteRegister` genannt) führt intern `HAL_I2C_Master_Transmit` (Registeradresse) gefolgt von
  `HAL_I2C_Master_Receive` (Daten) aus — ein **I2C-Lesezugriff**, kein Schreibzugriff. Die vormalige
  `EEPROM_ReadRegister` (`0x08006d84`) heißt jetzt `EEPROM_ReadRegister_NoMutex` (identischer Ablauf
  ohne eigene Mutex-Verwaltung). Echtes Schreiben übernimmt weiterhin ausschließlich
  `EEPROM_WriteVerify`. Damit ist bestätigt: der `0x55DD`-Zweig von Modbus-Register 45000 **lädt**
  `g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode` aus EEPROM `0x102`/`0x300`/`0x301` zurück,
  statt sie zu sichern; `0x55EE` schaltet `g_workMode=12` (Werksmodus) mit `g_batteryMode=0` und
  `g_cfgBackupModeEnabled=0`; `0x55FF` setzt das LLC/PFC-Kommando auf 10, löscht
  `g_powerLimitDeratedFlag`, setzt das Limitwort auf `0x09C4` (2500 W), nullt beide Modi sowie den
  Leistungssollwert (§4.2).
- OTA/Update (eigenes Image per YMODEM über die Debug-UART, zusätzlich ein zweiter Selbst-Update-Pfad
  über CAN, sowie Weiterleitung an das BMS über CAN/RS485) ist strukturell unverändert zu v115
  weiter möglich (§7). Cloud: entfällt, die Micro-Firmware hat keine eigene Netzwerk-/Cloud-Anbindung.

## 0a. Prüfliste

| # | Frage | Antwort v116 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register geändert (ggü. v115)? | Registertabelle/-adressen identisch. Nur zwei Wertänderungen: Register 43133/43134 Obergrenze `0x898`→`0x9c4`, Register 45502 Testreferenz `-2200.0`→`-2500.0`. Alle Magic-Codes (`0x55AA/0x55DD/0x55EE/0x55FF`) und die übrige Registerkarte unverändert. | §4.2 |
| 2 | CAN-Protokoll geändert? | Nur eine Konstante in `EMS_Inverter_CAN_Dispatcher` (`0x08005940`), Kommandos 0x03/0x04: Bereichsobergrenze `0x899`→`0x9c5`. Kommando-Satz, Message-IDs und übrige Logik unverändert (Funktionsgröße 1178→1174 B, nur diese eine Konstante plus reine Adressverschiebungen). | §4.1 |
| 3 | AC-Einspeisung 2500 W geändert? | Der Echtzeit-Hartgrenzwert (±2500 W, CAN-Kommandos 0x56/0x58) war **schon in v115** 2500 W. Neu in v116: sechs zuvor bei 2200 W liegende Konfigurations-/Rampen-/Init-Werte für denselben Leistungspfad werden auf 2500 W angehoben (Liste s. §0, inkl. `Bat_Mode3RoundRobinTimer`). Im Ergebnis ist ab v116 durchgängig 2500 W nutzbar, wo v115 teils bei 2200 W blockierte. | §0, §4.1, §4.2, §5 |
| 4 | Firmware-Update (eigenes + BMS-Weiterleitung) weiter möglich? | Ja. YMODEM-Selbst-Update über die Debug-UART, zusätzlicher CAN-Selbst-Update-Pfad (`OTA_SelfUpdate_StateMachine`) und BMS-Weiterleitung über CAN (`BMS_StartFwUpdateCan`/`BMS_FwUpdate_StateMachine`) bzw. RS485 (`BMS_FwUpdateStateMachineTask`) sind strukturell und in den Zustandsautomaten unverändert. | §7 |
| 5 | Cloud-Domains geändert? | Entfällt — die Micro-Firmware hat keine eigene Cloud-/Netzwerkanbindung; Internetkommunikation läuft ausschließlich über das Control-Board. | n/a |

## 1. Binary-Fingerprint

| Merkmal | Wert |
|---|---|
| Datei | `116_micro_VNSD-0_vd_inv_app_0116_0702_ota_163439.bin` |
| Größe | 115 712 Byte (0x1C400) |
| SHA-256 | `b81bb99f8213b98e11a082fba17603e5352d57b676ff6649e0323cac9f6fd860` |
| Link-Adresse | `0x08004000`–`0x080203FF` |
| MCU | STM32G474 (Cortex-M4F, Thumb-2), 16-KB-Bootloader unterhalb `0x08004000` |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW-Segment) |
| Bibliotheken | FreeRTOS (heap_4), letter-shell |
| Initial-SP | `0x20009A70` (Vektortabelle @ `0x08004000`, im Rohimage geprüft) |
| Reset-Vektor | `0x080042AD` (Thumb) → `Reset_Handler` (`0x080042ac`) |
| Versionskonstante | **0x74 (116 dez.)** @ `0x08010000`, gefolgt von ASCII `"VD01"` (Geräte-/Modelltag) |
| Image-Trailer | Gerätetag `"VNSD"` @ `0x080203E0`; Trailer-Wort ASCII `"13DD"` kurz vor Image-Ende — Bedeutung nicht ermittelt (v115 hat an derselben Stelle `"I1DD"`, Offener Punkt §10) |
| Herstell-Changelog | v116 — 2026-08-12: „VD 2500w" (Issue #18) |

**Scatter-Load** (direkt im Rohimage nachgerechnet, siehe `listing/08000000_0800ffff.txt` bzw.
`listing/08010000_0801ffff.txt`):

| Quelle (Flash) | Ziel | Länge | Verfahren |
|---|---|---|---|
| `0x08018c7c` (1112 B komprimiert) | `0x20000000` (.data) | 0x8D8 (2264 B) | `__decompress` (LZ77) |
| `0x080190d4` (27 472 B, unkomprimiert) | `0x10000000` (CCM-SRAM) | 0x6B50 (27 472 B) | `__scatterload_copy` (reines Kopieren) |
| – | `0x200008D8` (.bss) | 0x9198 (37 272 B) | `__scatterload_zeroinit` (Null-Init) |

Rechnerische Gegenprobe: `0x200008D8 + 0x9198 = 0x20009A70` = Stack-Top — stimmt exakt mit der
Vektortabelle überein. Flash↔CCM-Aufrufe laufen über 10-Byte-Veneers (`movw`/`movt r12; bx r12`)
ab `0x08004960` in Flash bzw. am Anfang von CCM (`0x10000000`ff., benannt `<Ziel>_veneer`).

## 2. Speicherkarte + Aufbereitungsstand in Ghidra

**RAM-/Peripherie-Blöcke** (`listing/globals_and_registers.txt`):

| Block | Bereich |
|---|---|
| `.data` | `0x20000000`–`0x200008D7` |
| `.bss` | `0x200008D8`–`0x20009A6F` |
| CCM-SRAM | `0x10000000`–`0x10006B4F` (0x6B50 B Code+Daten) |
| APB1 | `0x40000000`–`0x4000AFFF` |
| APB2 (u. a. HRTIM `0x40016800`) | `0x40010000`–`0x40017FFF` |
| AHB1 | `0x40020000`–`0x400243FF` |
| AHB2 GPIO | `0x48000000`–`0x48001BFF` |
| AHB2 Analog (ADC/DAC) | `0x50000000`–`0x50060BFF` |
| FMC/QSPI | `0xA0000000`–`0xA0001FFF` |
| PPB (Cortex-Systemperipherie) | `0xE0000000`–`0xE00FFFFF` |

**Prüfstand** (Programm `/VNSD/Micro/Micro_116_VNSD-0_vd_inv_app_0116_0702_ota_163439.bin`):

| Prüfung | Ergebnis |
|---|---|
| Funktionen | **693**, davon 693 benannt, 0 `FUN_*` |
| Ghidra-Symbole | 4626 |
| Dekompilation fehlgeschlagen | 0 |
| Benannte RAM-Globals (.data+.bss) | **925**, 0 unbenannte Reste (verbleibende `DAT_`-Treffer sind Querverweise in Kommentaren, keine echten Lücken) |
| Veneer-Funktionen (Flash↔CCM) | 49 |
| `shellCommandTable` | `0x0801892c`, `ShellCommand[53]` (Kommandos + 1 Nutzereintrag + Tastenbelegungen) |

Die Datei-/Ordnerkonvention (`<addr>_<name>.c`, `_index.md`, `listing/` mit Flash- und
CCM-Rohdisassemblat + `globals_and_registers.txt`) ist in `Dekompilate/VNSD_Micro_116/` beschrieben;
Details siehe dort.

## 3. FreeRTOS-Tasks

Aus `Task_CreateAllAndStartScheduler` (`0x0800f420`, 242 B) — acht direkte `xTaskCreate()`-Aufrufe
(keine Tabellen-Iteration), danach `xPortStartScheduler()`:

| Task | Entry-Funktion | Stack (Wörter) | Priorität | Handle |
|---|---|---|---|---|
| `vtask_led` | `0x080163e8` | 128 | 16 | `g_ledTaskHandle` |
| `vtask_time` | `0x080164c8` | 256 | 15 | `g_timeTaskHandle` |
| `vtask_modbus` | `0x08016424` | 128 | 14 | `g_modbusTaskHandle` |
| `vtask_shell` | `0x0801643c` | 512 | 9 | `g_shellTaskHandle` |
| `vtask_prt` | `0x08016430` | 256 | 8 | `g_canRxTaskHandle` * |
| `vtask_can` | `0x080163d0` | 256 | 6 | `g_canRxTaskHandle` * |
| `vtask_can_receive` | `0x080163e0` | 256 | 7 | `g_canRxTaskHandle` * |
| IDLE | `0x08012fd0` | 128 | 0 | `xIdleTaskHandle` |

\* Alle drei Tasks tragen im Dekompilat denselben Handle-Namen `g_canRxTaskHandle` (drei
unterschiedliche Schreibadressen, gleicher Auto-Name) — ob das eine echte Mehrfachbelegung derselben
SRAM-Zelle ist oder ein Namensartefakt, wurde in dieser Sitzung nicht abschließend geklärt (§10).
Bei Fehlschlag eines `xTaskCreate` (Rückgabe −1) druckt der Code `"..\..\SDK\FreeRTOS\tasks.c", 0x812`.
Alle Prioritäten/Stackgrößen sind — soweit in dieser Sitzung geprüft — unverändert zu v115.

## 4. Schnittstellen

### 4.1 CAN/FDCAN zum EMS (Control) und BMS

FDCAN1 wird in `FDCAN1_ConfigFiltersAndMailboxes` (`0x080058a8`) mit einem Standard-ID-Filter
(`0x300`/Maske `0x300`, internes EMS↔Inverter-Protokoll) und drei Extended-ID-Filtern (`0x4000`,
`0x4100`, je Maske `0xff00`; `0x1801aa01`/Maske `0x1ff0ffff`, BMS-Telemetrie) konfiguriert.
`HAL_FDCAN_RxFifo0Callback` (`0x08009248`) routet jede empfangene Nachricht je nach Laufzeitmodus
`g_canRxMode`: `3` → direkt an `BMS_CAN_Parser`, `4` → direkt an `BMS_FwUpdate_CanHandler`, sonst in
`g_canRxQueue`, von wo `CAN_RX_DispatchTask` (`0x08005f04`) die Frames an
`EMS_Inverter_CAN_Dispatcher` weiterreicht. Wo `g_canRxMode` selbst gesetzt wird, wurde nicht
geprüft (§10).

**EMS↔Inverter-internes Protokoll** (`EMS_Inverter_CAN_Dispatcher`, `0x08005940`, 1174 B, Standard-
11-Bit-ID, Kommando = unteres Byte der Arbitration-ID):

| Cmd (hex) | Bedeutung | Richtung/Bedingung |
|---|---|---|
| 0x01 | Leistungs-Sollwert setzen (`Ctrl_SetPowerReferenceClamped_veneer`), nur `g_workMode==0` | EMS→Inverter, Echo |
| 0x02 | Enable/Disable (`Cfg_SetStoredEnableFlag_veneer`), nur `g_workMode==0` | EMS→Inverter |
| 0x03 | Max. Entlade-Leistungslimit setzen, EEPROM `0x204` (`g_powerLimitDischargeW`); Grenze `<0x9c5`=2500 W (v115: `<0x899`=2200 W) | EMS→Inverter |
| 0x04 | Max. Lade-Leistungslimit setzen, EEPROM `0x202` (`g_powerLimitChargeW`); dieselbe Grenzänderung | EMS→Inverter |
| 0x05 | `g_emsDispatchFlag=1` (Sleep-Handshake laut Funktionskopf) | EMS→Inverter |
| 0x06 | `g_standbyLatch=1` (Standby-Latch; früher fälschlich `g_emsPairedFlag`, keine Pairing-Bestätigung) | EMS→Inverter |
| 0x07 | Sequenznummer-Handshake (`g_emsDispatchSeq=data+1`), Echo | bidirektional |
| 0x10 | Telemetrie-Block lesen, 0x30 B (`g_emsTelemetryBlock`) | Inverter→EMS |
| 0x11 | Debug-TX-Payload lesen, 8 B (`g_emsTxPayload`) | Inverter→EMS |
| 0x12 | Batterie-Kalibrierstatus lesen (Status/Lade-/Entladezeit), 6 B | Inverter→EMS |
| 0x13 | EMS-Lade-/Entladelimit lesen, 4 B | Inverter→EMS |
| 0x16 | Lade-/Entladestunden-Zähler von EMS synchronisieren (kein Echo) | EMS→Inverter |
| 0x50 | LLC/PFC-Kommando: 0=Stopp, 1=Start + `g_workMode=1` + `g_emsDispatchCmd=7` | EMS→Inverter |
| 0x51 | Kurzer Buzzer-Beep, nur Werksmodus (`g_workMode==0xC`) | EMS→Inverter, Echo |
| 0x52 | Werkstest-Subdispatch → `g_emsDispatchCmd` 2/3/4, nur Werksmodus | EMS→Inverter |
| 0x53 | Werkstest-Subdispatch → `g_emsDispatchCmd=5`, nur Werksmodus | EMS→Inverter |
| 0x54 | IO-Teststatus lesen (`IO_TestStatusFlag`), 1 B | Inverter→EMS |
| 0x55 | Werkstest-Subdispatch → `g_emsDispatchCmd=6`, nur Werksmodus | EMS→Inverter |
| 0x56 | LLC/PFC-Testkommando mit Leistungswert, Bereich ±2500 W (`0x9c4`/`0x1389`, **in v115 und v116 unverändert**), nur Werksmodus | EMS→Inverter |
| 0x57 | Backup-Modus-Flag setzen, nur `g_workMode==1` | EMS→Inverter |
| 0x58 | Leistungs-Sollwert setzen im Run-Modus (`g_workMode==1`), Bereich ±2500 W (**gleiche unveränderte Konstante `0x9c4`**) | EMS→Inverter |
| 0x60 | Batterie-Kalibriermodus starten (`g_workMode=4`) | EMS→Inverter |
| 0xC1 | `g_emsDispatchCmd=1`, Argument = Quell-Node-Byte der CAN-ID | EMS→Inverter |
| 0xC2 | Debug-Level setzen (1 oder 2) | EMS→Inverter |
| 0xC3 | Rohdaten-Passthrough in `g_shellRxQueue` (Shell-Eingabe-Queue, die in `shellHandler` geleert wird, also Shell über CAN; früher fälschlich `g_bmsCanQueue`) | unklar |
| 0xCB | Geräteinfo-/Versionsabfrage, 8 B | Inverter→EMS |
| 0xCE | CAN-OTA-Subprotokoll (oberes ID-Byte: 0=Start, 1=Chunk, 2=Abschluss-CRC) | bidirektional |

**BMS-Protokoll** (Extended 29-Bit-ID): `BMS_CAN_Parser` (`0x080058e8`, 80 B) wertet
`(can_id & 0xfffff) >> 0x10` als Unterscheidungsfeld aus:

| PF-Wert | Bedeutung | Zielvariablen |
|---|---|---|
| 1 | Pack-Spannung/-Strom + Temperatur/SOC | `g_bmsPackVoltageCurrent`, `g_bmsTempSoc`, aktualisiert `g_bmsLastRxTick` |
| 2 | Kapazität (roh) + Pack-Info | `g_bmsCapacityRaw`, `g_bmsPackInfo` |
| 3 | Ladespannungslimit + Entladestromlimit | `g_bmsChargeVoltLimit`, `g_bmsDischargeCurrLimit` |
| 4 | Fehler-/Warnwort | `g_bmsErrorWord`, `g_bmsWarningWord` |

`BMS_FwUpdate_CanHandler` (`0x08005868`, 60 B) behandelt BMS-FW-Update-Subkommandos (Maskentest
`(~can_id & 0xce)==0`, Unterscheidung über oberstes ID-Byte): `0` = OTA-Ack/Burst-Konfiguration
(0x80 oder 0x10 Chunks je nach Datenbyte, Chunk-Delay aus `data[1]`, `g_bmsOtaAckFlag=2`); `2` =
OTA-Ack mit CRC + Chunk-Status, `g_bmsOtaAckFlag=4`. `BMS_StartFwUpdateCan` (`0x08010978`, 24 B,
einziger Aufrufer: `OTA_CompleteFirmwareUpdate`) schaltet die BMS-OTA-Statemachine für CAN-Transport
scharf (`g_bmsFwUpdateMode=1`, `g_bmsOtaState=1`, `g_bmsOtaTransportState=4`).

**2200→2500-W-Fund im Detail** (v115 `08005944_EMS_Inverter_CAN_Dispatcher.c`, 1178 B ↔ v116
`08005940_EMS_Inverter_CAN_Dispatcher.c`, 1174 B):

```c
// case 3 (Entladelimit) und case 4 (Ladelimit), identischer Aufbau in beiden Versionen:
local_34 = *data;
if ((int)local_34 < 0x9c5) {          // v115: < 0x899
  g_powerLimitDischargeW = (ushort)local_34;   // bzw. g_powerLimitChargeW
  EEPROM_WriteVerify(0x204,(uint8_t *)&g_powerLimitDischargeW,2);  // bzw. 0x202
}
```

Die Filterwerte `0x4000`/`0x4100` aus `FDCAN1_ConfigFiltersAndMailboxes` ließen sich keiner der
beiden `BMS_FwUpdate_CanHandler`-Fälle eindeutig zuordnen (Offener Punkt §10).

### 4.2 RS485/Modbus

Dispatcher `Modbus_HandleRegisterAccess` (`0x080164e8`, 2842 B, `uint (uchar *outBuf, uchar
*regTable, int isWrite, uint value)`); Rückgabewerte entsprechen Modbus-Exceptions (`0`=OK,
`1`=abgelehnt/falscher `g_workMode`, `3`=ILLEGAL DATA VALUE). Der Parameter `regTable` ist trotz des
Zeiger-Typs die tatsächliche 16-Bit-Registeradresse (bestätigt über `Modbus_WriteSingleRegister`,
`0x0800f208`: Register `<0x9c40` werden gar nicht erst weitergereicht).

| Register (dez./hex) | Bedeutung | R/W | Details |
|---|---|---|---|
| 40000 / `0x9C40` | Batterie-Kalibrierstart (`g_battCalState`, State 0–7) | nur W | `0x55AA`→Reset (State 0); `0x55A1`–`0x55A6`→State 1–6; sonst Fehler 3. Setzt `g_workMode=4`. |
| 41000 / `0xA028` | Werks-Ack-Flag A (`g_factoryCmdAckFlag`) | R/W | Nur `0x55AA` schreibbar → Flag=1, sonst Fehler 3. |
| 41001 / `0xA029` | Werks-Ack-Flag B + Statistik löschen | R/W | `0x55AA` → zusätzlich `EEPROM_ClearStats(3)`. |
| 41010 / `0xA032` | Direkter Leistungslimit-/Derating-Schalter | R/W | Lesen: `{0, g_powerLimitDeratedFlag}`. Schreiben **exakt `800`** → Flag=1, Entladelimit=800 W; **exakt `0x9C4`(2500)** → Flag=0, Entladelimit=2500 W; jeder andere Wert Fehler 3. Persistiert EEPROM `0x204`+`0x90`. |
| 41100 / `0xA08C` | Modbus-Slave-Adresse | R/W | `0` → Fehler 3, sonst übernehmen (EEPROM `0x101`). |
| 41200 / `0xA0F0` | Backup-/Off-Grid-Ausgang Enable (invertiert) | R/W | `0`→`Cfg_SetStoredEnableFlag_veneer(1)`, `1`→`…(0)`, sonst Fehler 3. |
| 42000 / `0xA410` | Werksmodus-Bestätigung | R/W | `0x55AA`→No-op OK; `0x55BB`→`EEPROM_ReadRegister(0x301,&g_workMode,1,…)` liest `g_workMode` aus EEPROM zurück (I2C-Lesezugriff, s. u.); sonst Fehler 3. |
| 42010 / `0xA41A` | `g_modbusCfgByte1` | R/W | Gültig 0–2. |
| 42011 / `0xA41B` | `g_modbusCfgByte2` | R/W | Gültig 11–100 (`value-11≤89`), setzt `g_modbusCfgByte1=3`. |
| ~43100–43129 / `0xA85C`–`0xA879` | Netz-Standard-Tabelle `g_gridStandardTable` (6 von 10 Zeilen sichtbar, je 10 B) | R/W | Anker verifiziert: 43106 (Zeile 1, Feld+4), 43124 (Zeile 4, Feld+1). Übrige Einzelregister dieses Blocks nicht abschließend zugeordnet (§10). |
| 43133 / `0xA87D` | `g_ratedPowerConfigA`, EEPROM `0x800` | R/W | Obergrenze `0x898`(2200)→**`0x9c4`(2500)** — **Fund #2 der 2200→2500-Änderung.** |
| 43134 / `0xA87E` | `g_ratedPowerConfigB`, EEPROM `0x802` | R/W | Dieselbe Änderung. |
| 44000 / `0xABE0` | `g_cfgMaxSocPct` (max. Lade-SOC in %, Rohwert/10; früher fälschlich `g_gridProtCfgParam0`) | R/W | Bereich 800–1000. EEPROM `0x201`. |
| 44001 / `0xABE1` | `g_cfgCutoffSocPct` (Entlade-Abschalt-SOC in %, Rohwert/10; früher fälschlich `g_gridProtCfgParam1`) | R/W | Bereich 120–300. EEPROM `0x201`+1. |
| 44002 / `0xABE2` | `g_powerLimitChargeW` | R/W | Obergrenze 2500 W (unverändert ggü. v115). EEPROM `0x202`. |
| 44003 / `0xABE3` | `g_powerLimitDischargeW` | R/W | Obergrenze 2500 W; **zusätzlich**: ist `g_powerLimitDeratedFlag==1`, wird jeder Wert `>800` mit Fehler 3 abgelehnt — das eigentliche Enforcement des 800-W-Derating. EEPROM `0x204`. |
| 44100 / `0xAC44` | `g_cfgGridStandard`, ruft `Grid_Protection_SetLimits()` | R/W | Bereich 0–4. EEPROM `0x206`. |
| **45000 / `0xAFC8`** | **„Magic-Code"-Systemkommando** | nur W | Siehe Detailblock unten. |
| 45010–45019 / `0xAFD2`–`0xAFDB` | `g_cfgExtraRegBlock`, 10×16-Bit | R/W | Nur bei `g_workMode==0` oder `0xC` schreibbar. Letztes Register sichert den 20-B-Block nach EEPROM `0x21`. |
| 45200 / `0xB150` | Buzzer-Kurzton (Test) | nur W | Nur `g_workMode==0xC`/`0`. |
| 45300 / `0xB1B4` | TIM1-PWM-Testansteuerung | nur W | Nur `g_workMode==0xC`/`0`; `g_wdgSafetyCounter=100`. |
| 45400–45402 / `0xB198`–`0xB19A` | LLC/PFC-Testkommandos 1–3 | nur W | Nur `g_workMode==0xC`, ruft `Ctrl_SetLlcPfcCommand_veneer(1/2/6)`. |
| 45500–45502 / `0xB1BC`–`0xB1BE` | Lade-/Entlade-Leistungsreferenz (Test) | nur W | Register 45502: `g_workMode==0`, `value==1` → `Ctrl_SetPowerReferenceClamped_veneer(-2500.0)` (**v115: `-2200.0`** — **Fund #3**). |
| 45600/45601 / `0xB220`/`0xB221` | Lade-/Entlade-Leistung, beliebiger Float-Wert | nur W | Nur `g_workMode==0`. |
| 45603/45604 / `0xB223`/`0xB224` | Batteriekalibrierung Entlade-/Ladezeit, EEPROM `0x950`/`0x952` | nur R | I2C-Lesezugriff. |
| 46000 / `0xB3B0` | Debug-/BMS-OTA-Einstieg | nur W | `0x5100`→`g_debugLevel=1` (EEPROM `0x900`); `0x4D2`(1234)→zusätzlich `g_bmsFwUpdateMode=1`; `0x929`(2345)→zusätzlich `g_bmsOtaTransportState=2`; sonst Fehler 1. |

> **Hinweis (2026-10-01):** Die hier aufgeführten Register 40000 / 41000 / 41001 / 42000 usw. gehören zum **EIGENEN Modbus-Registerraum der Micro**. Sie sind nur über den eigenen RS485-Modbus-Slave der Micro erreichbar. Die Modbus-Register der Control mit denselben Nummern sind **andere Register** (z. B. ist Control 41001 nicht implementiert, Exception 2; Control 41000 startet Control und Micro neu). Siehe `../Modbus_RS485_TCP/VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md`.

**Magic-Code-Register 0xAFC8 (45000) — `0x55DD`/`0x55EE`/`0x55FF` (wörtlicher Code, identisch in
v115 und v116):**

```c
if (value == 0x55ee) {
  Ctrl_SetLlcPfcCommand_veneer(0); Buzzer_Stop();
  g_batteryMode = 0; g_cfgBackupModeEnabled = 0;
  g_workMode = 0xC;                      // Werks-/Testmodus
  return 0;
}
if (value != 0x55ff) {
  if (value == 0x55dd) {
    Ctrl_SetLlcPfcCommand_veneer(0); Buzzer_Stop();
    EEPROM_ReadRegister(0x102,&g_batteryMode,1,timeout);
    EEPROM_ReadRegister(0x300,&g_cfgBackupModeEnabled,1,timeout_00);
    EEPROM_ReadRegister(0x301,&g_workMode,1,timeout_01);
    return 0;
  }
  return 3;
}
// value == 0x55ff:
Ctrl_SetLlcPfcCommand_veneer(10); Buzzer_Stop();
g_powerLimitDeratedFlag = 0;             // Flag wird GELÖSCHT
puVar9[4] = 0xc4; puVar9[5] = 9;         // g_powerLimitDischargeW = 0x09C4 = 2500 W
g_modbusConfigParam1 = 0; g_batteryMode = 0; g_cfgBackupModeEnabled = 0; g_workMode = 0;
Ctrl_SetPowerReferenceClamped_veneer(0.0);
```

**Korrektur gegenüber der Annahme im Auftrag:** Keiner dieser drei Codes setzt ein 800-W-Derating.
`0x55EE` schaltet nur in den Werksmodus (`g_workMode=0xC`), `0x55DD` liest drei Statusbytes über
`EEPROM_ReadRegister` aus dem EEPROM zurück, `0x55FF` **löscht** das Derating-Flag und **setzt das
Entladelimit auf 2500 W zurück** — das Gegenteil einer Drosselung. Das tatsächliche 800-W-
Zwangsderating erfolgt an **zwei anderen** Registern (41010 und 44003, s. o.) durch **wörtliches**
Schreiben des Werts `800`.

**I2C-Richtungsbefund (aufgelöst):** Der frische Export benennt die Funktion `0x08006ce8` korrekt als
`EEPROM_ReadRegister` (in dieser Analyse zunächst irrtümlich `EEPROM_WriteRegister` genannt); die
vormalige `EEPROM_ReadRegister` (`0x08006d84`) heißt jetzt `EEPROM_ReadRegister_NoMutex`. Beide führen
`HAL_I2C_Master_Transmit` (Registeradresse) gefolgt von `HAL_I2C_Master_Receive` (Daten) aus — ein
I2C-Random-Access-**Lesezugriff**; `_NoMutex` läuft ohne eigene Mutex-Verwaltung (Aufrufer hält sie
bereits). Ein echtes Schreiben (Adresse+Daten in einem Transmit, mit Rücklese-Verifikation) übernimmt
weiterhin ausschließlich `EEPROM_WriteVerify` (`0x08006e40`). Damit ist bestätigt: der `0x55DD`-Zweig
von Register 45000 **lädt** `g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode` aus EEPROM
`0x102`/`0x300`/`0x301` zurück (letzten gespeicherten Zustand wiederherstellen), statt den aktuellen
RAM-Zustand zu sichern — ebenso liest `0x55BB` auf Register 42000 (`0xA410`) `g_workMode` aus
demselben EEPROM-Offset `0x301` zurück (s. o.). Vormals Offener Punkt in §10, jetzt durch die
Benennung im frischen Export geklärt.

Generischer Freischaltcode `0x55AA` erscheint unabhängig von den drei „Magic Codes" bei mehreren
weiteren Registern (40000, 41000, 41001, 42000) als Werksreset-/Bestätigungscode.

### 4.3 Debug-Shell

`shellCommandTable` @ `0x0801892c`, `ShellCommand[53]` (53 Einträge = Kommandos + 1 Nutzereintrag +
Tastenbelegungen), davon **42 echte `ShellCmd_*`-Handler** (ältere Schätzung „33" war zu niedrig).
Login läuft über `shellSetUser`/letter-shell-Bibliothek; laut Diff-Artefakt in `xTaskResumeAll`
(Literal-Pool-Kommentar) hat der eine hinterlegte Nutzer ein **leeres Passwort**. Das tatsächliche
Passwort-Byte selbst wurde nicht aus dem rohen `ShellCommand[53]`-Array dekodiert (Offener Punkt §10).

| Befehl | Adresse | Kurzbeschreibung |
|---|---|---|
| `log_data` | `0800eb20` | Gibt bis zu 100 Werte aus `g_sampleRingBuffer` per `printf("%f\r\n")` aus. |
| `pi` | `080127c0` | Setzt Strom-PI-Regler (`Ctrl_SetRampTargetA`), druckt „Strom-PI-Parameter geändert kp=%f, ki=%f". |
| `vol_pi` | `080127cc` | Setzt Spannungs-PI-Regler (`Ctrl_SetRampTargetB`). |
| `qpr` | `080127d8` | Setzt Resonanzregler QPR (`Ctrl_SetPrControllerTarget`). |
| `power_cfg` | `080127e4` | LLC/PFC-Testkommando (`Ctrl_HandleLlcPfcTestCommand`). |
| `clear` | `08013658` | ANSI-Clear-Escape an das Terminal. |
| `help` | `08013c90` | Kommandoliste bzw. Hilfetext zu einem Kommando. |
| `alarm` | `080146f8` | Setzt Buzzer-Periodenzähler. |
| `backup_mode` | `080146fc` | Backup-Modus ein/aus (`Cfg_SetStoredEnableFlag`). |
| `bat_mode` | `0801474c` | Batteriemodus ein/aus (`Cfg_SetPersistedFlag102`) — **byte-identisch zu v115**, kein neues Verhalten (§11). |
| `dac_high`/`dac_low` | `080148fc`/`08014902` | Direktschreiben DAC3/DAC1 (Testausgang). |
| `help_prt` | `08014908` | Hilfetext zu `prt`. |
| `io_set` | `0801497c` | Schaltet Digital-I/Os (LED, Netz-/Insel-Relais, Fault-Reset, Aux). |
| `io_show` | `08014b14` | Gibt kompletten I/O-Statusblock aus. |
| `log_clear`/`log_err`/`log_list`/`log_open` | `08014dd0`–`08014ddc` | Fehler-Log löschen/gefiltert/vollständig ausgeben/Live-Toggle. |
| `old_ate` | `08014df4` | Burn-in-Modus (`g_workMode=4`), Zustand 0–5. |
| `par_mode` | `08014e54` | Parallelbetrieb-Modus (`Cfg_SetOperatingModeFlag`). |
| `pf`/`pf_power` | `08014ecc`/`08014f10` | Blindleistungs-Sollwert setzen. |
| `pf_set` | `08014f60` | Schutz-Komparatorschwelle setzen (`Prot_ClampComparatorThreshold`) — **byte-identisch zu v115** (§11). |
| `prt` | `08015014` | Periodische Log-Ausgabe konfigurieren (Thunk auf `0800ef10`). |
| `reset_memory` | `08015018` | Daten löschen (mit/ohne Summenwerte), `EEPROM_SaveOperatingStats`. |
| `reset` | `0801507c` | `Board_SystemReset()`. |
| `rtos_status` | `08015080` | FreeRTOS-Tasklist. |
| `fan_set` | `080150b4` | Lüfter-PWM (TIM1), 0–400. |
| `set_mode` | `080150c4` | `g_workMode` setzen (0 Default, 1 Hand, 2 Debug, 4 Burn-in, 0xD Spannungsquelle), EEPROM-Retry-Schreiben. |
| `set_n` | `0801518c` | Droop-Parameter setzen. |
| `set_power` | `080151bc` | Geclampter Leistungssollwert (`Ctrl_SetPowerReferenceClamped`). |
| `set_sr` | `080151ec` | Clip-Flag Resonanzregler (`Ctrl_SetPrClipFlag`). |
| `show_pi` | `080151f0` | Aktuelle PI-Werte ausgeben. |
| `show_power` | `08015230` | Referenztabelle der 24 Testmodi (0–23) für `set_mode`. |
| `test`/`test_1` | `08015578`/`08015610` | Debug-Messgrößen bzw. Stromregelkreis-Enable-Flag. |
| `update`/`update_bms`/`update_bms2` | `0801561c`–`08015684` | Firmware-Update starten (eigen/BMS-Transportvariante 1/2). |
| `version` | `080156b4` | Version, Build-Zeitstempel, Hardware-/Set-Version. |
| `sr_set` | `08015768` | Deadtime/Synchrongleichrichtungs-Tastverhältnis. |

**Korrektur eines Rohdiff-Fundes:** Ein vorab berechneter Funktionsdiff hatte für `bat_mode` und
`pf_set` einen neuen Codeblock (Float-Debug-Printf + Tail-Call) behauptet. Beide Funktionen sind in
v115 und v116 **exakt gleich groß** (38 bzw. 48 B) und inhaltlich identisch; der vermeintliche neue
Block liegt tatsächlich im unsymbolisierten String-/Codezwischenraum direkt dahinter (Inhalt von
`ShellCmd_pi`/`ShellCmd_pf_power`, nur an nicht indizierter Adresse dupliziert) und wurde vom
Diff-Werkzeug dem falschen Funktionsnamen zugeschlagen (§11).

### 4.4 I2C-EEPROM-Layout

Low-Level: `EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex` (I2C-Random-Access-Lesezugriff, s. §4.2),
`EEPROM_WriteVerify`/`_NoMutex` (echtes Schreiben, Page-Write + Rücklese-Verifikation, bis zu
4 Retries, bei Fehlschlag `FaultLog_Enqueue()` + `I2C1_Init()`-Bus-Reset), `EEPROM_WriteVerify_veneer`
(`0x10000104`, CCM-Aufrufer wie `Prot_LatchHwFaultToErrorWord` können damit ebenfalls EEPROM
schreiben). `EEPROM_SaveTimestamp` schreibt `HAL_GetTick()` nach Offset `0x80` und überschreibt ihn
sofort wieder mit 0 (Zweck unklar, §10). `EEPROM_ClearStats(level)` löscht je nach Level Teile des
48-B-Statistikblocks, schreibt aber immer den kompletten Block zurück.

| Offset | Größe | Feld | Bedeutung |
|---|---|---|---|
| `0x000` | 1 B | Versions-/Boot-Flag | `3`=voll gültig, `2`=nur Statistik-Reload, sonst Werksreset |
| `0x001` | 0x2F B | `g_cfgDeviceIdBlock` | Hauptkonfigblock (u. a. „VD01"-Signatur, Modellcode, HW-Variante); Innenlayout nicht neu verifiziert |
| `0x080` | 1 B | Heartbeat/Timestamp | Siehe `EEPROM_SaveTimestamp`-Eigenheit oben |
| `0x090` | 1 B | `g_powerLimitDeratedFlag` | 1=Entladelimit auf 800 W gedeckelt, 0=volle 2500 W; `0xFF`→0 beim Laden |
| `0x100`–`0x102` | 3 B | `g_standbyLatch`, `g_modbusSlaveAddress`, `g_batteryMode` | Byte 3–4 unbekannt |
| `0x150` | 1 B | `g_ctrlModeDisplayCounter` | Countdown (CAN-Retry/Moduswechsel-LED); `0xFF`→0 beim Laden |
| `0x200`–`0x206` | 7 B | `g_cfgMaxSocPct`/`g_cfgCutoffSocPct` (je /10 skaliert), `g_powerLimitChargeW`, `g_powerLimitDischargeW`, `g_cfgGridStandard` | Summe exakt 7 B |
| `0x300` | 0x7C B | `g_cfgBackupModeEnabled` + Folgeblock | Work-Mode-/Backup-/Scheduling-Block, Innenlayout nicht nachgelesen |
| `0x500` | 0x30 B | `g_statsLastDay` | Betriebsstunden-/Energiestatistik (`EEPROM_SaveOperatingStats`) |
| `0x800`/`0x802` | je 2 B | `g_ratedPowerConfigA`/`B` | Nennleistungs-Konfiguration (physische Bedeutung laut Namensbeleg mit niedriger Konfidenz) |
| `0x900` | 2 B | `g_debugLevel` + Bootsafe-Flag | Debug-/Trace-Level; nur im Werksreset-Pfad geschrieben |
| `0x901` | 1 B | – | Bei **jedem** Boot bedingungslos auf 0 zurückgeschrieben |
| `0x950`/`0x952` | je 2 B | `g_emsChargePowerLimitW`/`g_emsDischargePowerLimitW` | EMS-Lade-/Entladelimit |

Werksreset-Defaultwerte: `"VD01"`, Modellcode `0xD`, Versionsmarker `0x74` (=116, passt zur
FW-Version), Netzparameter `0x64`/`0xC`, Modbus-Adresse `1`, Batteriemodus `1`, Lade-/Entladelimit
je `0x9C4` (2500 W). Innenlayout der Blöcke `0x001`–`0x02F`, `0x100` (Byte 3–4) und `0x300`–`0x37B`
sowie der Sub-Block `0x021` (Schedule-Tabelle laut Altdoku) nicht im Detail nachvollzogen (§10).

## 5. Regelung & Leistungsteil

Zwei zeitkritische ISRs laufen aus CCM-SRAM, erreicht über Flash-Veneers (`0x08004992`,
`0x080049b0`):

- **`Inv_CurrentLoopIsr`** (`0x10001428`, 3266 B): schnelle Pro-PWM-Zyklus-ISR — SOGI-PLL +
  PR-Stromregler, ADC-Skalierung, PWM-Sollwertaktualisierung, schnelle Schutzabfragen. Schreibt
  selbst keine HRTIM-Compare-/Periodenregister.
- **`Ctrl_MainControlIsr`** (`0x1000219c`, 12 446 B): übergeordnete Regel-ISR. Ruft der Reihe nach
  `ADC_ProcessSamples_veneer` → `Grid_GetFrequency_veneer` → `Ctrl_ComputeGridRmsAndPower` →
  `Ctrl_LlcVoltageRegulationAndProtection` → `Prot_MonitorHvBusLevelFlags` →
  `Ctrl_RampDcBusReferenceWithFaultCheck` → `Inv_GridConnectionSupervisor`, danach eine große
  Zustandsmaschine (PFC/LLC-Hochlauf, Netzzuschaltrampe, Normalbetrieb, Fehler-/Derating-Pfade;
  wegen der Größe nicht vollständig zustandsweise dekodiert, §10).

**PLL/SOGI-Kette** (alle CCM): `PLL_InitCordicState` (Init + CORDIC-Konfiguration für sin/cos),
`PLL_SogiUpdateStep` (ein SOGI-Schritt, α/β-Signale, Frequenzfehler-Integration mit Clamping),
`PLL_IntegratePhaseAngle`/`PLL_NormalizeAngle` (Phasenintegration/-wrap), `PLL_UpdateFrequencyEstimate`
(RMS-/Kreuzprodukt-Akkumulation, adaptive Frequenzschätzung), `PLL_UpdatePhaseFilterFir`/
`PLL_UpdateGridFreqFilter` (FIR-Glättung, Notch-/Biquad-Koeffizienten).

**Stromregler/PR-Kette**: `Ctrl_PrControllerStepDualMode` (kombinierter SOGI-PLL+PR-Schritt),
`Ctrl_PrControllerDesign` (PR-Koeffizienten aus Grenzfrequenz/Gain/Resonanzfrequenz),
`Ctrl_PiControllerStep`/`_Reset` (generischer geclampter PI), `Ctrl_BiquadFilterDesign`/
`Ctrl_NotchFilterDesign`, `Ctrl_InitResonantControllerParams`, `Ctrl_InitCurrentLoopState`.

**HRTIM**: Die Duty-/Phasenberechnung für die LLC-Halbbrücke sitzt in einer **dritten, separaten
Flash-ISR** `HRTIM_TIMB_IRQn_IRQHandler` (`0x0800c8dc`, 338 B) — je Timer-B-Repetition werden
CMP2–4/SET/RST-Register für zwei PWM-Beine modusabhängig (1–3) berechnet, über
`g_hrtimStateCh1/Ch2`, `g_hrtimPeriodTicks`, `g_hrtimPhaseOffset`, `g_hrtimCompareOffset`,
`g_softStartCounter`. Init/Konfiguration: `Ctrl_HRTIM_Init` (`0x0800d164`, 988 B, kompletter
Bring-up: Master + 6 Timer-Einheiten, Compares, Deadtime, Chopper, ADC-Trigger, Fault-Config, Start);
`HRTIM_FaultConfig` (`0x0800ca38`, 648 B, Quelle/Polarität/Filter für 11 Fault-/External-Event-Kanäle);
`HAL_HRTIM_FaultConfig` (`0x08009bd6`) als Lock-geschützter Wrapper.

**Die 2200→2500-W-Konstante in CCM** (Float `0x451C4000`=2500.0f, v115 `0x45098000`=2200.0f, an zwei
relativen Stellen — **Korrektur gegenüber der vorigen Sitzung:** eine ursprünglich vermutete dritte
CCM-Stelle in `Ctrl_ResetControlState` ist keine reale Änderung, siehe unten):

| Adresse | Kontext |
|---|---|
| `0x100029cc` | In `Ctrl_MainControlIsr`: sobald ein Zähler 120 Ticks erreicht, wird das Ausgangs-/Clamp-Limit des Leistungs-PI-Reglers auf 2500.0 gesetzt, `Ctrl_PiControllerReset()` aufgerufen — Startpunkt der Netz-Zuschaltrampe. |
| `0x10005174` | Basis der thermischen Derating-Formel: ab ~64 °C `(1.0-(T-64)/20)*2500.0`, zusätzlich prozentual gedeckelt — thermisches Derating des AC-Ausgangslimits. |

**Korrektur:** `Ctrl_ResetControlState` (v116 `0x10005fd8`, v115 `0x10005fe8`) wurde in der vorigen
Sitzung als dritte CCM-Änderungsstelle geführt. Ein Byte-für-Byte-Vergleich des Literal-Pool-Eintrags
zeigt jedoch, dass **beide** Versionen dort bereits `0x451C4000` (2500.0f) laden — der komplette
Funktionskörper ist bis auf die adressbedingte Verschiebung identisch (130 B in beiden Versionen).
Es gibt an dieser Stelle **keine** v115→v116-Änderung; sie entfällt aus der Fundstellen-Liste.

Die verbleibenden zwei CCM-Stellen sind konsistent mit „AC-Einspeiselimit von 2200 W auf 2500 W
angehoben" — beim Start der Netz-Zuschaltrampe und als Basiswert des thermischen Derating; zusammen
mit den in §4.1/§4.2 gefundenen Modbus-/CAN-seitigen Änderungen und dem Fund in
`Bat_Mode3RoundRobinTimer` (Flash-Code, kein CCM) ergeben sich die in §0 gelisteten sechs realen
2200→2500-W-Fundstellen.

## 6. Schutz & Fehlercodes

`g_faultLogRing` (SRAM `0x20001a2c`, 200 Einträge à 20 B) wird über `FaultLog_Enqueue`
(`0x0800eb88`) befüllt und über `FaultLog_Dequeue`/`Log_CheckFaultEventsAndPrint` (nur Codes >0x27)
an `Log_PrintFaultEntry` (`0x0800dba0`, 1712 B) zur Textausgabe übergeben. Ein zweiter, kleinerer
Ring `g_eventLogRing` (`0x200029cc`, 12 B/Eintrag, `Log_RecordFaultEvent` `0x0800daa4`) hat in den
gelesenen Dateien keine erkennbare Print-Routine (§10). `Prot_LatchHwFaultToErrorWord`
(`0x10005dd8`, CCM) übernimmt das HW-Komparator-Latch-Byte, loggt Event `0x36` und mappt die
Latch-Bits in das `err2`-Wort — deckt sich **exakt** mit der feldverifizierten Bit-Zuordnung der
Altdokumentation (Bit0/1/3/4 ← HW-Latch, Bit7=Event 0x13, Bit8=Event 0x31, Bit9=Event 0x29,
Bit11=Event 0x35); keine Widersprüche, nur die Codeadresse hat sich erwartungsgemäß verschoben.

| Code | Bedeutung (Deutsch) | Zusatzdaten |
|---|---|---|
| default | Generischer Rohwert-Dump | 4× `%5d` |
| 1–4 | Generische Dumps mit 0–3 Floats | data1–3 |
| 5 | Leerlauf (Idle) | – |
| 6 | Netzseitige Prüfung OK | – |
| 7/8 | HV-Zwischenkreis-Entladung/-Ladung freigegeben | – |
| 9/0xa | Batterieladung/-entladung freigegeben | – |
| 0xb | Prüfung auf fehlerhafte Batteriedaten OK | – |
| 0xc | Vorladung (Precharge) eingeschaltet | – |
| 0xe/0xf | H-Brücke/Lade-H-Brücke gestartet | – |
| 0x10/0x11 | LLC-Stufe gestartet / LLC-Softstart | – |
| 0x12 | Wechselrichter abgeschaltet | set_power/max_power/low_power |
| 0x13 | System heruntergefahren (deckt sich mit Altdoku „err2 Bit7") | state ×2 |
| 0x14 | Backup-Funktion aktiviert | – |
| 0x15 | BMS fordert Schlafmodus an | – |
| 0x16/0x17 | Zwangsladung angefordert / ohne Kommunikation | – |
| 0x18 | Batteriekommunikation OK | – |
| 0x19 | Netzfrequenz erfasst | feq |
| 0x1a | Reset-Ursache (0=Zentralsteuerung, 1=Pin, 2=Boot, 5=Power-Down) | Subcode |
| 0x29 | Zustandsautomat-Fehler (Altdoku: err2 Bit9) | last/now |
| 0x2a/0x2b | Netzfrequenz-/SPLL-Frequenzfehler | feq |
| 0x2c | Netz-VRMS-Fehler | V_rms, count, feq |
| 0x2d | Netz-Maximalspannungsfehler | V_max |
| 0x2e | Spannungs-Kurvenformfehler | V, V_rms, sin |
| 0x2f | Netzeinspeisestrom-Fehler | U_rms, I_soll, I_rms |
| 0x30 | Software-PLL-Neustart | Soft-/HW-PLL-Freq, U_rms |
| 0x31 | Inselbetrieb-Stromfehlerschutz (Altdoku: err2 Bit8) | Stromsumme, I_insel |
| 0x32 | Netzerkennungsfehler | – |
| 0x33 | Batteriekommunikationsfehler | IE/IR/ECR (CAN) |
| 0x34 | HV-Zwischenkreisspannung schwankt | Spannung |
| 0x35 | Zeitüberschreitung Zustandswechsel (Altdoku: err2 Bit11) | status, step |
| 0x36 | Hardware-Schutzauslösung (Sammelereignis; Subflags: Bit0 Wechselrichter-Ausgangsüberstrom, Bit1 Zwischenkreis-Überspannung, Bit2 Überstrom batterieseitig) | I_out, I_bat, Flags |
| 0x37 | PWM-Grenzwertschutz | PWM_soll, U_hv, U_netz |
| 0x38 | Selbsttestfehler | Fehlercode |
| 0x3a/0x3b | Batteriefehler (allgemein) / von Batterie gemeldeter Fehlercode | – / Code |
| 0x3c | Batteriespannungsfehler | BAT V, BMS V, Modus |
| 0x3d | Freigabefehler Batterie-Ausgangsspannung | BMS-Strom, Modus |
| 0x3e | Batterie bis SOC-Limit entladen/geladen | Subcode, SOC |
| 0x3f/0x40 | Spitzenstrom-/Spitzenspannungsschutz | Maximalwerte |
| 0x41 | Watchdog-Reset | – |
| 0x42 | EEPROM-Schreibfehler | – |

Codes `0xd`, `0x1b`–`0x28`, `0x39` fallen auf den `default`-Zweig (nur Rohwerte) — ungenutzt oder nur
über `g_eventLogRing` sichtbar (§10). `Prot_SetFaultBitOnMatch` (`0x10005db0`) setzt ein Fault-Bit
bei Wertegleichheit mit einer unbenannten Konstante — Zweck nicht auflösbar (§10).

## 7. OTA/Update

**Selbst-Update per YMODEM** (über die Debug-UART/Konsole, dieselbe Queue wie die Shell-Befehle
`update`/`update_bms`/`update_bms2`): klassisches YMODEM (SOH/STX/EOT/2×CAN-Abbruch), Header-Paket
mit Dateiname+Größe (`YModem_ParseSizeString`), Größenprüfung gegen freien Flash (Default 0x80000),
danach `FLASH_EraseSectors` ab `0x08040000` und `FLASH_WriteData` ab `0x08044000`. Eine explizite
CRC16-Prüfung der YMODEM-Paket-Trailer-Bytes selbst konnte im Code **nicht** nachgewiesen werden
(Offener Punkt §10). Abschluss über `OTA_CompleteFirmwareUpdate`: bei
`g_bmsOtaTransportState==0` schreibt es den Image-Deskriptor und löst einen Reset über `SCB_AIRCR`
aus (Übergabe an den Bootloader); bei `1`/`2` wird stattdessen `OTA_ComputeChecksum` (einfache
invertierte Byte-Summe, **kein** CRC16) über das empfangene Image gebildet und an
`BMS_StartFwUpdateCan`/`BMS_StartFwUpdateRs485` weitergereicht — das Image wird dann an das BMS
durchgereicht statt selbst übernommen zu werden.

**Zweiter, unabhängiger Selbst-Update-Pfad über CAN** — `OTA_SelfUpdate_StateMachine`
(`0x0800613c`, 386 B, getrieben von `g_canOtaState`, CAN-IDs um `0xCE`, s. §4.1): Erase → je
8-Byte-Chunk mit `CRC16_Modbus`-Prüfung schreiben (`FLASH_WriteData` ab `0x08044000`) → finale
`CRC16_Modbus` über das Gesamtimage → Deskriptor (Adresse/Länge/Prüfsumme) nach `0x08040000`
schreiben → `SCB_AIRCR`-Reset.

**BMS-Firmware-Weiterleitung über CAN**: `BMS_FwUpdate_StateMachine` (`0x080062f0`, 608 B,
Poll-Loop, max. 11 Retries) — State 1 sendet Header (Gesamtgröße + `CRC16_Modbus` des Images),
State 2 wartet auf Start-Ack (Timeout 1000 Zyklen), State 3 sendet bis zu 0x80 Chunks à 8 B
(gedrosselt über `g_bmsOtaChunkDelayMs`) mit CRC16 über den Burst, State 4 wertet Chunk-Ack aus
(Fortsetzen/Wiederholen/Abbruch). Abschluss: `Ctrl_HandleLlcPfcTestCommand_veneer(1,0.0)`
(Sicherheitszustand), `EEPROM_SaveOperatingStats()`, `NVIC_SystemReset()`. Eine zweite, transport-
unabhängige Variante `BMS_FwUpdateStateMachineTask` (5 Zustände: SendStart/SendDataChunk/
SendSizeCmd/SendFinalize/fertig) wird vermutlich für die RS485-Weiterleitung genutzt; beide Wege
enden mit demselben Sicherheits-/Persistenz-/Reset-Ablauf. `BMS_FwUpdate_CanHandler` (`0x08005868`)
ist die RX-Seite (§4.1).

**Fazit**: Sowohl das eigene Firmware-Update als auch die Weiterleitung an das BMS (über CAN und —
soweit im Code erkennbar — RS485) sind strukturell und in den Zustandsautomaten unverändert zu v115
und damit uneingeschränkt weiter möglich.

## 8. Sicherheitsbefunde

1. **Unauthentifizierter Debug-/Werksmodus-Zugriff per Modbus:** Register 45000 (`0xAFC8`) schaltet
   mit dem bloßen Kenntnis der Werte `0x55EE`/`0x55DD`/`0x55FF` in den Werksmodus bzw. persistiert/
   liest Betriebszustände, ohne weitere Authentifizierung. Register 46000 (`0xB3B0`) schaltet mit
   `0x5100` dauerhaft einen erhöhten Debug-Level frei (EEPROM `0x900`) und kann mit `0x4D2`/`0x929`
   eine BMS-Firmware-Update-Sequenz auslösen.
2. **Generischer Freischaltcode `0x55AA`** wirkt unauthentifiziert an mehreren Registern (40000
   Batterie-Kalibrierung, 41000/41001 Werks-Ack, 42000 Werksmodus-Bestätigung).
3. **Direkter Leistungslimit-Bypass:** Register 41010 erlaubt mit dem literalen Wert `800` bzw.
   `0x9C4` das Setzen/Aufheben des 800-W-Zwangsderating ohne weitere Prüfung außer dem exakten
   Zahlenwert.
4. **Default-Shell-Benutzer mit leerem Passwort** (letter-shell, ein Nutzereintrag, s. §4.3).
5. **Namensbefund bestätigt:** Die Funktion `0x08006ce8` heißt im frischen Export korrekt
   `EEPROM_ReadRegister` und implementiert einen I2C-**Lesezugriff**, keinen Schreibzugriff (§4.2). Der
   `0x55DD`-Pfad auf Register 45000 stellt damit einen zuvor gespeicherten Zustand wieder her, statt
   den aktuellen RAM-Zustand zu sichern — sicherheitstechnisch weniger relevant als ursprünglich
   vermutet, da kein sonst verlorener Zustand persistiert wird.
6. Alle genannten Mechanismen laufen über den internen RS485-/CAN-Bus zwischen Control-Board, Micro
   und BMS; eine direkte Erreichbarkeit von außerhalb des Geräts (Internet/WLAN) besteht für die
   Micro-Firmware selbst nicht — sie hat keine eigene Netzwerkanbindung. Die Einstufung der
   praktischen Ausnutzbarkeit hängt daher davon ab, wie das Control-Board diesen Bus exponiert
   (siehe Control-FW-Analyse); das wurde in dieser Sitzung nicht erneut geprüft.

## 9. Chinesische Strings

184 UTF-8-Strings mit chinesischem Text im Export. Die im Zwischenstand verwendete Auszugsdatei
enthielt bei 157 von 184 Einträgen beschädigte Kodierung (Ersatzzeichen); alle 184 wurden in dieser
Sitzung erneut direkt aus dem Rohimage (Flash-Bereich per Datei-Offset `Adresse-0x08004000`) bzw. aus
dem CCM-Lade-Image (Datei-Offset `0x150D4+(Adresse-0x10000000)`) dekodiert und vollständig übersetzt.

**Aus `Log_PrintFaultEntry` (`0800dba0`) — Klartexte des Fehler-Logs, s. auch §6:**

| Adresse | Chinesisch | Deutsch |
|---|---|---|
| `0800db08` | `\r\n日志清除成功\r\n` | „\r\nLog erfolgreich gelöscht\r\n" |
| `0800df78` | `网侧检测OK` | Netzseitige Prüfung OK |
| `0800df94` | `系统关闭 state = %d,%d` | System wird abgeschaltet, state = %d,%d |
| `0800dfbc` | `允许高压母线充电` | Laden der HV-Zwischenkreisspannung freigegeben |
| `0800dfe0` | `允许高压母线放电` | Entladen der HV-Zwischenkreisspannung freigegeben |
| `0800e004` | `允许电池放电` | Batterieentladung freigegeben |
| `0800e024` | `允许电池充电` | Batterieladung freigegeben |
| `0800e044` | `电池异常数据检测OK` | Prüfung auf anomale Batteriedaten OK |
| `0800e064` | `电池通讯OK` | Batteriekommunikation OK |
| `0800e078` | `开启备份功能` | Backup-Funktion aktiviert |
| `0800e098` | `打开预充` | Vorladung gestartet |
| `0800e0b0` | `启动H桥` | H-Brücke gestartet |
| `0800e0c8` | `启动充电H桥` | Lade-H-Brücke gestartet |
| `0800e0e4` | `启动LLC` | LLC gestartet |
| `0800e0f8` | `LLC软起` | LLC-Softstart |
| `0800e10c` | `BMS请求休眠` | BMS fordert Schlafmodus an |
| `0800e128` | `强制充电请求` | Zwangsladeanforderung |
| `0800e148` | `无通讯强制充电` | Zwangsladung ohne Kommunikation |
| `0800e168` | `复位原因 = %d` | Reset-Ursache = %d |
| `0800e180` | `中控复位` | Reset durch Zentralsteuerung |
| `0800e190` | `引脚复位` | Pin-Reset |
| `0800e1a0` | `boot复位` | Boot-Reset |
| `0800e1ac` | `掉电复位` | Reset durch Spannungsausfall |
| `0800e1bc` | `检测电网频率 feq = %d` | Netzfrequenz erkannt, feq = %d |
| `0800e1dc` | `自检错误%d` | Selbsttestfehler %d |
| `0800e1fc` | `电网捕获频率异常` | Anomalie bei Netzfrequenzerfassung |
| `0800e238` | `SPLL频率异常` | SPLL-Frequenzanomalie |
| `0800e250` | `电网VRMS异常` | Netz-VRMS-Anomalie |
| `0800e630` | `电网MAX电压异常` | Netz-Maximalspannungs-Anomalie |
| `0800e660` | `电压波形异常` | Spannungswellenform-Anomalie |
| `0800e6b0` | `电池电压异常` | Batteriespannungs-Anomalie |
| `0800e6fc` | `电池电压允许输出异常` | Anomalie bei zulässiger Batterie-Ausgangsspannung |
| `0800e740` | `高压母线电压波动` | Schwankung der HV-Zwischenkreisspannung |
| `0800e774` | `并网电流异常` | Netzparallelbetrieb-Stromanomalie |
| `0800e798` | `锁相环重启` | PLL-Neustart |
| `0800e7b0` | `电池异常` | Batterieanomalie |
| `0800e7c0` | `电池上传故障` | Von Batterie gemeldeter Fehler |
| `0800e7d8` | `故障代码 = %x` | Fehlercode = %x |
| `0800e7ec` | `电网检测异常` | Netzerkennungs-Anomalie |
| `0800e804` | `电池放电到上限soc %d` | Batterie entladen bis oberes SOC-Limit %d |
| `0800e824` | `电池充电到上限soc %d` | Batterie geladen bis oberes SOC-Limit %d |
| `0800e844` | `电池通讯异常` | Batteriekommunikations-Anomalie |
| `0800e870` | `峰值电流保护` | Spitzenstromschutz |
| `0800e8a4` | `峰值电压保护` | Spitzenspannungsschutz |
| `0800e8d0` | `看门狗复位` | Watchdog-Reset |
| `0800e8e4` | `硬件保护` | Hardwareschutz |
| `0800e91c` | `逆变器输出过流` | Wechselrichter-Ausgangsüberstrom |
| `0800e934` | `母线过压` | Zwischenkreis-Überspannung |
| `0800e944` | `电池测过流` | Überstrom batterieseitig gemessen |
| `0800e958` | `PWM超过保护` | PWM-Grenzwertschutz überschritten |
| `0800e970` | `状态机异常` | Zustandsmaschinen-Anomalie |
| `0800ea54` | `状态机切换超时` | Zeitüberschreitung beim Zustandswechsel |
| `0800ea88` | `离网电流异常保护` | Schutz bei Inselbetrieb-Stromanomalie |
| `0800eaa4` | `电流累加值 = %f , 当前离网电流 = %f` | Stromsummenwert = %f, aktueller Inselbetrieb-Strom = %f |
| `0800ead4` | `EEPROM写错误` | EEPROM-Schreibfehler |
| `0801862c` | `逆变器关闭 set_power = %6.3f \| max_power = %6.3f \| low_power = %6.3f` | Wechselrichter ausgeschaltet, set_power/max_power/low_power wie angegeben |
| `0801867c` | `电压有效值 = %5f 电流设定值 = %5f 电流有效值 = %5f` | Spannungseffektivwert / Stromsollwert / Stromeffektivwert |
| `080186c4` | `软件锁相环频率 = %5f 硬件锁相环频率 = %5f 电压有效值 = %5f` | Software-/Hardware-PLL-Frequenz, Spannungseffektivwert |
| `08018718` | `pwm设定值 = %5f ,高压母线电压 = %5f,电网电压 = %5f` | PWM-Sollwert, HV-Zwischenkreisspannung, Netzspannung |

**Aus `Log_PrintDebugData` (`08010c40`) — Statusausgaben:**

| Adresse | Chinesisch | Deutsch |
|---|---|---|
| `08010f3c` | `\r\n捕获频率 = %f\r\n` | Erfasste Frequenz = %f |
| `08010f58` | `\r\n锁相环频率 = %f\r\n` | PLL-Frequenz = %f |
| `08010f80` | `\r\n电网电压有效值 = %f,avg = %f 10min= %f\r\n` | Netzspannungseffektivwert, avg, 10-min-Mittel |
| `08010fbc` | `\r\n输出电压有效值 = %f , 平均功率 = %d \r\n` | Ausgangsspannungseffektivwert, mittlere Leistung |
| `08010ff0` | `\r\n离网电压有效值 = %f\r\n` | Inselbetrieb-Spannungseffektivwert |
| `08011014` | `\r\n并网电流有效值 = %f\r\n` | Netzparallel-Stromeffektivwert |
| `08011034` | `\r\n离网电流有效值 = %f\r\n` | Inselbetrieb-Stromeffektivwert |
| `080110a4` | `\r\n日充电%d W/H,日放电%d W/H,日收益%d\r\n` | Tagesladung, Tagesentladung, Tagesertrag |
| `08011124` | `\r\n设置功率 = %2f , 实际功率%2f ,发出功率%2f \r\n` | Sollleistung, Istleistung, abgegebene Leistung |
| `08011164` | `\r\n有功 = %2f , 无功 = %2f , fo = %2f \r\n` | Wirkleistung, Blindleistung, fo |
| `08011198` | `\r\n高压 = %d ,低压 = %d \r\n` | Hochspannung, Niederspannung |
| `0801123c` | `\r\n ***原始值***\r\n` | ***Rohwerte*** |
| `080115b0` | `\r\n***实际值***\r\n` | ***Istwerte*** |
| `08011724` | `电池状态\r\n` | Batteriestatus |
| `08011810` | `bms最后通讯时间 = %d` | Letzte BMS-Kommunikationszeit = %d |
| `0801854c` | `\r\n电池功率 = %f,离网功率 = %f,离网功率2 = %f,并网功率 = %f\r\n` | Batterieleistung, Inselbetrieb-Leistung ×2, Netzparallel-Leistung |

**Shell-Hilfe-/Statustexte (verschiedene `ShellCmd_*`, keine feste „users"-Zuordnung im Export):**

| Adresse | Chinesisch | Deutsch |
|---|---|---|
| `080065b4` | `其他错误` | Sonstiger Fehler |
| `08014724`/`08014738` | `开启备份模式`/`关闭备份模式` | Backup-Modus aktivieren/deaktivieren |
| `08014774`/`08014788` | `关闭电池模式`/`开启电池模式` | Batteriemodus deaktivieren/aktivieren |
| `080147e8` | `修改电流pi参数 kp = %f , ki = %f` | Strom-PI-Parameter geändert |
| `08014860` | `修改电压pi参数 kp = %f , ki = %f` | Spannungs-PI-Parameter geändert |
| `080148d8` | `修改kpr参数 kp = %f, kr = %f` | kpr-Parameter geändert |
| `08014928` | `prt 打印功能码` | prt-Druckfunktionscode |
| `08014940`/`08014954`/`08014968` | `0 -- 电网参数`/`1 -- 运行状态`/`2 -- ADC采样` | 0 Netzparameter / 1 Betriebszustand / 2 ADC-Abtastwerte |
| `08014b00` | `没有对应IO口` | Kein passender IO-Port |
| `08014bf8`/`08014cc4` | `********输出口状态*********`/`********输入口状态*********` | Status Ausgangsports / Eingangsports |
| `08014c34`–`08014cac` | `2 并网继电器 = %d` … `6 电池辅源 = %d` | 2 Netzparallel-Relais … 6 Batterie-Hilfsversorgung |
| `08014e28`/`08014e3c` | `老化模式 %d`/`老化模式错误` | Burn-in-Modus %d / Fehler im Burn-in-Modus |
| `08014e90`/`08014ea4`/`08014eb8` | `关闭并机模式`/`开启并机模式`/`并机准备模式` | Parallelbetrieb aus/ein/Vorbereitung |
| `08014efc` | `设定值 = %f` | Sollwert = %f |
| `08014f40` | `设无功功率定值 = %f` | Blindleistungs-Sollwert gesetzt |
| `08014f90` | `固定pf = %f` | Fester Leistungsfaktor (pf) = %f |
| `08014ff0` | `设定值1 = %f,设定值2 = %f` | Sollwert1, Sollwert2 |
| `08015040`/`08015068` | `清空所有数据(不含累计数据)`/`清空所有数据` | Alle Daten löschen (ohne/mit Summenwerten) |
| `08015134`–`0801517c` | `调试模式`/`手动模式`/`老化`/`ate老化`/`电压源模式` | Debug-/Hand-/Burn-in-/ATE-Burn-in-/Spannungsquellen-Modus |
| `080151dc` | `设定功率 %f` | Leistung eingestellt %f |
| `080152a4`–`08015560` | Testmodus-Legende 0–23 (`0.IDLE …` bis `23. 单PFC离网`) | Vollständige deutsche Übersetzung: IDLE, Open-Loop-PFC-Test, Inselbetrieb-Spannung/-Vollausgabe, Laden (Konstantspannung/-strom), Backup-Test (Single-PFC), Netzparallelbetrieb-Test, LLC Hoch-/Niederspannungsseite Open-Loop, LLC-Softstart, LLC-Closed-Loop-Laden, Single-PFC Laden/Netzparallel/Inselbetrieb |
| `08017fa9`–`08018469` | `版本信息`, `打印数据`, `打印日志`, `设置下垂n`, `设置同步整流系数`, `打印错误日志`, `开启日志自动打印`, `设定pf值` u. a. | Versionsinfo, Daten/Log ausgeben, Droop-/SR-Koeffizient setzen, Fehlerlog ausgeben, automatische Log-Ausgabe, pf-Wert einstellen, Blindleistung/Modus einstellen, RTOS-/IO-Status, Lüfter-Tastverhältnis, PFC-Strombegrenzung, Burn-in-Modus einstellen (vollständige Liste: 35 Einträge, sinngemäß deutsche Kurzbeschreibungen der jeweiligen Shell-Hilfetexte) |
| `080185e8` | `任务名 任务状态 优先级 剩余栈 任务序号` | Taskname / Taskstatus / Priorität / Reststack / Tasknummer (Kopfzeile `rtos_status`) |
| `0801859c` | `1.RESET 复位控制状态，运行其他模式前需要先运行这个` | 1. RESET — Regelzustand zurücksetzen, vor anderen Modi zuerst ausführen |

**Aus `Ctrl_HandleLlcPfcTestCommand` (`10000b58`, CCM) — Werks-Testkommando-Ausgaben:**

| Adresse | Chinesisch | Deutsch |
|---|---|---|
| `10000e48` | `单PFC充电，母线电压 = %f\r\n` | Einzel-PFC-Laden, Zwischenkreisspannung = %f |
| `10000e6c` | `过压测试\r\n` | Überspannungstest |
| `10000e7c` | `停\r\n` | Stopp |
| `10000e90` | `整机充电，恒压 %f V，恒流%f A\r\n` | Gesamtgerät lädt, Konstantspannung/-strom |
| `10000ebc` | `参数异常,要求40-60V，0-60A\r\n` | Parameterfehler, erforderlich 40–60 V, 0–60 A |
| `10000ee0` | `并网，功率 = %f\r\n` | Netzparallelbetrieb, Leistung = %f |
| `10000ef8` | `整机离网输出\r\n` | Gesamtgerät im Inselbetrieb-Ausgang |
| `10000f10` | `PFC开环占空比 %f \r\n` | PFC Open-Loop-Tastverhältnis |
| `10000f2c` | `PFC开环离网输出，电压峰值 %f V\r\n` | PFC Open-Loop Inselbetrieb, Spitzenspannung |
| `10000f58` | `并机测试\r\n` | Parallelbetriebstest |
| `10000f68` | `开环输出正弦波\r\n` | Open-Loop-Ausgabe Sinuswelle |
| `10000f80` | `单LLC开环充电,频率%fk\r\n` | Einzel-LLC Open-Loop-Laden, Frequenz %f kHz |
| `10000fa0` | `参数异常\r\n` | Parameterfehler |
| `10000fb0` | `单LLC开环放电,频率%fk\r\n` | Einzel-LLC Open-Loop-Entladen |
| `10000fd0` | `单LLC软起\r\n` | Einzel-LLC-Softstart |
| `10000fe0` | `参数异常,要求349V - 501V\r\n` | Parameterfehler, erforderlich 349–501 V |
| `10001004` | `单PFC并网，并网功率 = %f\r\n` | Einzel-PFC Netzparallelbetrieb |
| `10001028` | `进入标准模式\r\n` | Standardmodus wird aktiviert |
| `10001040` | `进入恒压输出模式\r\n` | Konstantspannungs-Ausgabemodus wird aktiviert |

Alle 184 Strings wurden dekodiert; die o. g. Tabellen enthalten alle Einträge mit erkennbarem
Fließtext-/Statusgehalt. Rein numerische/Format-Duplikate (z. B. mehrfach vorkommende
Hilfetext-Varianten) wurden zusammengefasst dargestellt.

## 10. Offene Punkte

- Grid-Standard-Tabelle `g_gridStandardTable` (Modbus ~43100–43129): nur 2 von ca. 19 Einzelregistern
  exakt verifiziert.
- Fehler-/Warncodes `0xd`, `0x1b`–`0x28`, `0x39` in `Log_PrintFaultEntry` ohne Klartext-Zweig.
- Zweiter Fault-Ring `g_eventLogRing`/`Log_RecordFaultEvent` ohne erkennbare Print-Routine.
- Handle-Namensdopplung `g_canRxTaskHandle` bei drei FreeRTOS-Tasks (`vtask_prt`, `vtask_can`,
  `vtask_can_receive`) — echte Mehrfachbelegung oder Namensartefakt nicht geklärt.
- CRC16-Absicherung der YMODEM-Paket-Trailer im Code nicht nachgewiesen (evtl. nicht implementiert
  oder wegoptimiert).
- `Ctrl_MainControlIsr` (12 446 B) nicht vollständig zustandsweise dekodiert.
- `Prot_SetFaultBitOnMatch`: Vergleichskonstante/Zweck nicht auflösbar.
- Zweck von CAN-Kommando `0xC3` (Rohdaten-Passthrough) und der Extended-ID-Filter `0x4000`/`0x4100`-
  Zuordnung zu `BMS_FwUpdate_CanHandler`-Fällen nicht geklärt.
- Innenlayout der EEPROM-Blöcke `0x001` (47 B), `0x100` (Byte 3–4), `0x300` (124 B), Sub-Block
  `0x021` nicht aus dem Code nachvollzogen.
- Verhalten von `EEPROM_SaveTimestamp` (Tick schreiben, sofort mit 0 überschreiben) — Zweck unklar.
- Bedeutung des Trailer-Worts am Image-Ende (`"13DD"` in v116, `"I1DD"` in v115) nicht ermittelt.
- Reales letter-shell-Passwort-Byte nicht aus dem `ShellCommand[53]`-Array dekodiert (nur indirekt
  über einen Diff-Artefakt als „leer" belegt).
- Genaue Kommando-Semantik von `Ctrl_HandleLlcPfcTestCommand` (Unterschied Kommando 1 vs. 10 u. a.)
  nicht vollständig dekodiert.
- Ursprung/Aufrufkette, die `g_emsDispatchSeq` setzt, nicht bis zum auslösenden CAN-Frame verfolgt.
- Zusätzlich ~354 Funktions-/Global-Namen aus dem Namensbeleg-Bestand (`out_*.json`/`gout_*.json`)
  mit `confidence: low` (überwiegend Callback-/Veneer-/Thunk-Klassen ohne Aufrufer im jeweiligen
  Benennungs-Batch) — Details siehe `.../scratchpad/micro116/out_00.json`…`out_03.json` und
  `g/gout_00.json`…`gout_03.json`.

## 11. Korrekturen gegenüber der alten Analyse

**Gegenüber `VNSD_Micro_Inverter_FW_Analyse_vd_inv_app_0116_de.md`/`VNSD_Fehlercodes_Micro_FW_Analyse_de.md`
(alte Basis 0x08000000):** Alle dortigen Flash-Adressen sind um `0x4000` zu niedrig; die MCU wurde
dort fälschlich als STM32F3 statt **STM32G474** geführt, und der komplette CCM-Codeanteil
(`0x10000000`ff., u. a. beide zentralen Regel-ISRs) fehlte vollständig. Die feldverifizierte
`err2`-Bit-Zuordnung sowie die Fehler-/Warncode-Bedeutungen der Altdokumentation stimmen dagegen
**exakt** mit dieser Sitzung überein (§6) — nur die Codeadressen haben sich verschoben. Weitere in
dieser Sitzung gefundene Namenskorrekturen (aus dem Abgleich der Namensbeleg-Dateien): I2C-TX/RX-
Vertauschung bei `HAL_I2C_Master_Receive`/`_Transmit` in der Altfassung korrigiert;
`I2C_ITTransferEnd` fälschlich als „UART_"-Funktion geführt, tatsächlich I2C-ISR-Statusregister;
`RCC_GetPLLOutputClockFreq`/`RTC_EnterInitMode`/`RTC_ExitInitMode` in der Altfassung fälschlich als
I2C-Funktionen benannt (Fallout des falschen Basis-Offsets) — jetzt korrekt der RCC-/RTC-Peripherie
zugeordnet.

**Korrekturen an vorab berechneten Funktions-Diffs (v115↔v116) — Diff-Werkzeug-Artefakte, in dieser
Sitzung durch direkten Vergleich beider Dekompilate widerlegt:**

- `ShellCmd_bat_mode`/`ShellCmd_pf_set`: kein neues Verhalten. Beide Funktionen sind in v115 und
  v116 byte-identisch (38 bzw. 48 B); der vom Diff als „neu eingefügt" gemeldete Codeblock
  (Float-Debug-Printf + Tail-Call) gehört tatsächlich zu unsymbolisiertem Nachbarcode
  (Duplikat/Variante von `ShellCmd_pi`/`ShellCmd_pf_power`), den das Diff-Werkzeug mangels sauberer
  Funktionsgrenzen fälschlich zugeordnet hat.
- `EXTI9_5_IRQHandler`: der echte Interrupt-Handler (`0x08006994` in v116 / `0x0800699c` in v115, je
  24 B) ist in v115 und v116 identisch (Tail-Jump in eine gemeinsame EXTI-Latch-Routine). Das vom
  Diff gemeldete Wachstum betrifft eine **andere, unbenannte** Funktion direkt dahinter im Flash
  (`0x0800699c` in v116), die als gewöhnliche `bl`-Zielfunktion aus `Inv_BuildTelemetryBlock` aufgerufen
  wird — kein Interrupt-Handler. **Weitere Korrektur in dieser Sitzung:** Die zuvor (auch im hiesigen
  §0) als „real" gewertete Funktionsänderung an dieser Stelle ist selbst ein Artefakt — ein
  Byte-für-Byte-Vergleich zeigt, dass der Code dort identisch zur v115-Funktion
  `EMS_ResetMemoryByLevel` (`0x080069a4`, 86 B) ist; die Funktion hat in v116 lediglich ihren Namen
  und ihre Ghidra-Funktionsregistrierung verloren (erscheint dort nur noch als Codeadresse
  `SUB_0800699c`, nicht als eigene Function). Der Code selbst, inklusive der umgebenden
  Debounce-/Dispatch-Logik in `Inv_BuildTelemetryBlock`, ist unverändert. Es gibt an dieser Stelle
  **keine** v115→v116-Verhaltensänderung.
- `EEPROM_SaveOperatingStats`: **keine** neue v116-Funktion. Der 4-Byte-Trampolincode
  (`EEPROM_WriteVerify(0x500,&g_statsLastDay,0x30)`) existierte unter identischer Adresse bereits in
  v115 — dort lediglich unter dem generischen Sammelnamen `Stub_EmptyNoOp2` geführt (der in v115
  offenbar für zwei verschiedene, unterschiedlich große Adressen vergeben wurde: einen echten
  2-Byte-`return`-Stub und dieses 4-Byte-Trampolin). v116s Benennungslauf hat das Trampolin korrekt
  eigen benannt; das ist eine Namensverbesserung dieser Analyse-Sitzung, keine Firmware-Änderung.
- **Real bestätigt** (nicht widerlegt): `Stub_CallHandlerOffset28` — v115 (`0x0800fff0`, 2 B) ist ein
  reiner `return`-Stub, v116 (`0x0800ffee`, 10 B) ruft real `HAL_UART_AbortCpltCallback` auf; beide
  Adressen und Größen wurden direkt in beiden Dekompilaten gegengelesen.
- **EEPROM_ReadRegister/_NoMutex (vormals EEPROM_WriteRegister/EEPROM_ReadRegister):** Der in §0/§4.2/
  §8 dieser Sitzung zunächst als Vermutung formulierte I2C-Richtungsbefund ist bestätigt. Der frische
  Export benennt `0x08006ce8` korrekt als `EEPROM_ReadRegister` (I2C-Lesezugriff mit Mutex) und
  `0x08006d84` als `EEPROM_ReadRegister_NoMutex` (derselbe Ablauf ohne eigene Mutex-Verwaltung). Der
  vormals in §10 geführte Offene Punkt zur I2C-Leserichtung entfällt damit; der `0x55DD`-Zweig von
  Modbus-Register 45000 liest nachweislich `g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode` aus
  dem EEPROM zurück, statt sie zu sichern.

**Lehre für künftige Diffs:** Das verwendete Funktions-Diff-Werkzeug (`l115_116.json`) bestimmt
Funktionsgrenzen offenbar teils über das nächste bekannte Symbol statt über die tatsächliche
Funktionsgröße und erzeugt dadurch bei Funktionen mit unsymbolisiertem Nachbarcode falsche
„Wachstums"-Befunde. Alle in dieser Analyse verwendeten Diff-Ergebnisse wurden deshalb gegen die
tatsächlichen Funktionsgrößen in beiden `_index.md`-Dateien und/oder den realen Dekompilat-Inhalt
gegengeprüft, bevor sie übernommen wurden.
