# Micro-FW v115 (Venus D) — Vollanalyse

> 🇬🇧 English version: [VNSD_Micro_FW_v115_Full_Analysis.md](VNSD_Micro_FW_v115_Full_Analysis.md)

**Stand:** 2026-09-30.

**Englisch:** `VNSD_Micro_FW_v115_Vollanalyse_de.md` (wird nach dieser Fassung erstellt) ·
**v116-Fassung:** `VNSD_Micro_FW_v116_Vollanalyse_de.md` ·
**Diff zum Nachfolger v116:** `VNSD_Micro_Diff_v115_v116_de.md` ·
**Dekompilat/Listing:** `Dekompilate/VNSD_Micro_115/` (lokal, nicht im Repo) (696 Funktionen, 932 benannte RAM-Globals,
`listing/` mit Flash- und CCM-Rohdisassemblat + `globals_and_registers.txt`)

Dieses Dokument ist die v115-Vollanalyse im selben Aufbau wie `VNSD_Micro_FW_v116_Vollanalyse_de.md`.
Inhalte, die zwischen v115 und v116 nachweislich identisch sind (per `VNSD_Micro_Diff_v115_v116_de.md`
byte-für-byte bzw. C-Logik-geprüft), werden hier nicht erneut ausgeschrieben, sondern mit
„unverändert gegenüber v116, siehe dort" referenziert — jeder Abschnitt nennt aber explizit die für
v115 geltenden Fakten (Adressen, Version, Grenzwerte), auch wenn der Inhalt sich sonst gleicht.

## 0. Kurzfassung

- Image-Basis **0x08004000** (16-KB-Bootloader darunter), Image-Größe 0x1C400 Byte (115 712 B),
  MCU **STM32G474**, Keil ARMCC, FreeRTOS (RVDS/ARM_CM4F, `heap_4`, 16-KB-Heap), letter-shell —
  identisch zu v116 (unverändert gegenüber v116, siehe dort §0/§1).
- `0x6B64` B zeitkritischer Code laufen aus CCM-SRAM `0x10000000`, erreicht über 10-Byte-Veneers in
  Flash ab `0x08004960`ff. — **20 Byte mehr** als v116 (`0x6B50`), da v116 an einer Stelle im CCM eine
  Funktionsregistrierung verliert (`EMS_ResetMemoryByLevel`/`SUB_0800699c`, siehe Diff-Dokument),
  Inhalt und Verhalten des CCM-Codes selbst sind ansonsten gleich.
- Herstell-Changelog nennt für v115 nur „Angepasst an Control-Firmware 147" (Issue #21); keine
  v114-Dekompilate verfügbar, daher in dieser Sitzung nicht gegen eine Vorversion überprüfbar.
- **v115 ist die 2200-W-Baseline** für den in v116 auf 2500 W angehobenen Lade-/Entlade-Leistungspfad
  (volle Herleitung mit Adressen in beiden Versionen: `VNSD_Micro_Diff_v115_v116_de.md`, Tabelle
  „2200 W → 2500 W"):
  1. `EMS_Inverter_CAN_Dispatcher` (`0x08005944`), CAN-Kommandos 0x03/0x04: Bereichsprüfung
     `< 0x899` (2201) — Obergrenze effektiv 2200 W.
  2. `Modbus_HandleRegisterAccess` (`0x080164d8`), Register 43133/43134 (`g_ratedPowerConfigA`/`B`):
     Obergrenze `0x898` (2200 W).
  3. Dieselbe Funktion, Register 45502 (Werks-Testbefehl „Entlade-Referenz"):
     `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)`.
  4. CCM `Ctrl_MainControlIsr`, Netz-Zuschaltrampe (`0x100029d8`): Ausgangs-/Clamp-Limit des
     Leistungs-PI-Reglers beim Rampenstart = `2200.0`.
  5. Dieselbe Funktion, thermische Derating-Basis (`0x10005180`) = `2200.0`.
  6. `Bat_Mode3RoundRobinTimer` (`0x0801183c`): `g_batPowerRefW` = `±2200.0`.
  Der **Echtzeit-Hartgrenzwert** für den unmittelbaren Leistungssollwert (CAN-Kommandos 0x56/0x58,
  Prüfung `wert+0x9c4<0x1389`, ≙ ±2500 W) ist dagegen bereits in v115 **2500 W** — dieselbe
  Inkonsistenz, die v116 an den o. g. sechs Stellen auflöst, ist in v115 bereits vorhanden.
  Bemerkenswert: `Ctrl_ResetControlState` (CCM, `0x10005fe8`-Literal) initialisiert das
  Leistungslimit-Feld **schon in v115** auf `2500.0`, nicht auf 2200.0 — diese Stelle gehört *nicht*
  zur 2200-W-Baseline (siehe Diff-Dokument, Abschnitt „Widerlegter Kandidat"). Ebenso sind die
  EEPROM-Werksreset-Defaultwerte für Lade-/Entladelimit bereits in v115 `0x9C4` (2500 W) — die
  2200-W-Deckelung wirkt nur über die genannten sechs Laufzeit-/Konfigurationspfade, nicht über den
  Werksreset-Pfad.
- Namensbefund zu `EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex`: In diesem frischen v115-Export
  sind beide Funktionen bereits korrekt benannt (`0x08006cf0`/`0x08006d8c`, I2C-Random-Access-Lesezugriff
  mit/ohne Mutex) — die frühere Namensverwechslung (`EEPROM_WriteRegister`) betraf nur eine ältere
  Zwischenfassung der v116-Analyse und ist für v115 kein offener Punkt (siehe v116-Dokument §4.2/§11).
- OTA/Update (YMODEM über Debug-UART, zusätzlicher CAN-Selbst-Update-Pfad, BMS-Weiterleitung über
  CAN/RS485) strukturell identisch zu v116 (unverändert, siehe v116-Dokument §7 und Diff-Dokument).
- Fehler-/Warncodes, `err2`-Bit-Zuordnung, Chinesische Strings, Shell-Befehlsliste, EEPROM-Layout:
  inhaltlich identisch zu v116, nur Adressen verschoben (siehe jeweilige Abschnitte unten und
  v116-Dokument).

## 0a. Prüfliste

| # | Frage | Antwort v115 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register geändert (ggü. Vorversion)? | Nicht prüfbar — keine v114-Dekompilate vorhanden. Ggü. **v116**: Register 43133/43134 Obergrenze hier `0x898`(2200) statt `0x9c4`(2500); Register 45502 Testreferenz `-2200.0` statt `-2500.0`. Alle Magic-Codes (`0x55AA/0x55DD/0x55EE/0x55FF`) identisch zu v116. | `VNSD_Micro_Diff_v115_v116_de.md` |
| 2 | CAN-Protokoll (ggü. Vorversion)? | Nicht prüfbar (keine v114-Daten). Ggü. v116: `EMS_Inverter_CAN_Dispatcher` (`0x08005944`) hat für Kommandos 0x03/0x04 die Bereichsobergrenze `0x899` statt `0x9c5`. Sonst identisch. | Diff-Dokument |
| 3 | AC-Einspeisung — Leistungsgrenze? | **2200 W** an sechs Konfigurations-/Rampen-/Init-Stellen (Liste s. §0); der CAN-Echtzeit-Hartgrenzwert (Kommandos 0x56/0x58) ist bereits 2500 W. v116 hebt die sechs 2200-W-Stellen auf 2500 W an. | §0, §4.1, §4.2, §5 |
| 4 | Firmware-Update (eigenes + BMS-Weiterleitung) möglich? | Ja, strukturell identisch zu v116 (§7). | §7 |
| 5 | Cloud-Domains? | Entfällt — keine eigene Cloud-/Netzwerkanbindung der Micro-Firmware (identisch zu v116). | n/a |

## 1. Binary-Fingerprint

| Merkmal | Wert |
|---|---|
| Datei | `115_micro_VNSD-0_202601230921310c0e30687.bin` |
| Größe | 115 712 Byte (0x1C400) |
| SHA-256 | `b2505973136f414928927df3211ce4ada90bdebcc100b5c770230cb5484c5c98` |
| Link-Adresse | `0x08004000`–`0x080203FF` |
| MCU | STM32G474 (Cortex-M4F, Thumb-2), 16-KB-Bootloader unterhalb `0x08004000` |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW-Segment) |
| Bibliotheken | FreeRTOS (heap_4), letter-shell |
| Initial-SP | `0x20009A70` (Vektortabelle @ `0x08004000`, identisch zu v116) |
| Reset-Vektor | `0x080042B1` (Thumb) → `Reset_Handler` (`0x080042b0`) |
| Versionskonstante | **0x73 (115 dez.)** @ `0x08010000`, gefolgt von ASCII `"VD01"` |
| Image-Trailer | Gerätetag `"VNSD"` @ `0x080203E0`; Trailer-Wort ASCII `"I1DD"` @ `0x080203FC` (v116 hat an derselben Stelle `"13DD"` — Bedeutung in beiden Versionen nicht ermittelt, s. §10) |
| Herstell-Changelog | v115 — 2026-08-12: „Angepasst an Control-Firmware 147" (Issue #21) |

**Scatter-Load** (nachgerechnet aus `listing/08000000_0800ffff.txt`/`listing/08010000_0801ffff.txt`):

| Quelle (Flash) | Ziel | Länge | Verfahren |
|---|---|---|---|
| `0x08018c80` (1112 B komprimiert) | `0x20000000` (.data) | 0x8D8 (2264 B) | `__decompress` (LZ77) |
| `0x080190d8` (27 492 B, unkomprimiert) | `0x10000000` (CCM-SRAM) | **0x6B64** (27 492 B) | `__scatterload_copy` (reines Kopieren) |
| – | `0x200008D8` (.bss) | 0x9198 (37 272 B) | `__scatterload_zeroinit` (Null-Init) |

Gegenprobe: `0x200008D8 + 0x9198 = 0x20009A70` = Stack-Top, identisch zu v116. `ScatterRegion`-Tabelle
bei `0x08018900` (v116: `0x0801892c`-Bereich, siehe unten), `shellCommandTable`/`ShellCommand[53]` bei
`0x08018930` (v116: `0x0801892c`).

## 2. Speicherkarte + Aufbereitungsstand in Ghidra

**RAM-/Peripherie-Blöcke** — identisch zu v116 (gleiche Adressgrenzen, RAM-Layout laut Vorgabe
unverändert):

| Block | Bereich |
|---|---|
| `.data` | `0x20000000`–`0x200008D7` |
| `.bss` | `0x200008D8`–`0x20009A6F` |
| CCM-SRAM | `0x10000000`–`0x10006B63` (**0x6B64** B Code+Daten, 20 B mehr als v116) |
| APB1 | `0x40000000`–`0x4000AFFF` |
| APB2 (u. a. HRTIM `0x40016800`) | `0x40010000`–`0x40017FFF` |
| AHB1 | `0x40020000`–`0x400243FF` |
| AHB2 GPIO | `0x48000000`–`0x48001BFF` |
| AHB2 Analog (ADC/DAC) | `0x50000000`–`0x50060BFF` |
| FMC/QSPI | `0xA0000000`–`0xA0001FFF` |
| PPB (Cortex-Systemperipherie) | `0xE0000000`–`0xE00FFFFF` |

**Prüfstand** (Programm `/VNSD/Micro/Micro_115_VNSD-0_202601230921310c0e30687.bin`):

| Prüfung | Ergebnis |
|---|---|
| Funktionen | **696**, davon 696 benannt, 0 `FUN_*` |
| Ghidra-Symbole | 4641 |
| Dekompilation fehlgeschlagen | 0 |
| Benannte RAM-Globals (.data+.bss, gleiche Zählmethode wie v116) | **932** (v116: 925 — Differenz durch Benennungslauf-Abdeckung, RAM-Adresslayout selbst identisch) |
| Veneer-Funktionen (Flash↔CCM) | 49 (identisch zu v116) |
| `shellCommandTable` | `0x08018930`, `ShellCommand[53]` (v116: `0x0801892c`) |

Datei-/Ordnerkonvention wie in `Dekompilate/VNSD_Micro_115/` beschrieben (identisch zu v116).

## 3. FreeRTOS-Tasks

Aus `Task_CreateAllAndStartScheduler` (`0x0800f424`, 242 B) — acht `xTaskCreate()`-Aufrufe, danach
`xPortStartScheduler()`. **Byte-für-Byte-Vergleich mit v116 bestätigt: Reihenfolge, Stackgrößen und
Prioritäten sind identisch** (`VNSD_Micro_Diff_v115_v116_de.md`), nur die Entry-Adressen verschieben sich
um die allgemeine Relokationsverschiebung:

| Task | Entry-Funktion (v115) | Stack (Wörter) | Priorität | Handle |
|---|---|---|---|---|
| `vtask_led` | `0x080163d8` | 128 | 16 | `g_ledTaskHandle` |
| `vtask_time` | `0x080164b8` | 256 | 15 | `g_timeTaskHandle` |
| `vtask_modbus` | `0x08016414` | 128 | 14 | `g_modbusTaskHandle` |
| `vtask_shell` | `0x0801642c` | 512 | 9 | `g_shellTaskHandle` |
| `vtask_prt` | `0x08016420` | 256 | 8 | `g_canRxTaskHandle` * |
| `vtask_can` | `0x080163c0` | 256 | 6 | `g_canRxTaskHandle` * |
| `vtask_can_receive` | `0x080163d0` | 256 | 7 | `g_canRxTaskHandle` * |
| IDLE | `0x08012fd8` | 128 | 0 | `xIdleTaskHandle` |

\* Dieselbe Mehrfachbelegung des Handle-Namens wie in v116 (offener Punkt, s. §10/v116-Dokument §10).

## 4. Schnittstellen

### 4.1 CAN/FDCAN zum EMS (Control) und BMS

Struktur, Filter, Kommando-Satz und Message-IDs sind **identisch zu v116** (`BMS_CAN_Parser`,
`BMS_FwUpdate_CanHandler`, `HAL_FDCAN_RxFifo0Callback`, `CAN_RX_DispatchTask` sind laut Diff
byte-identisch) — vollständige Kommandotabelle siehe v116-Dokument §4.1.

**Einzige inhaltliche Abweichung:** `EMS_Inverter_CAN_Dispatcher` (`0x08005944`, 494 B normalisierte
Opcode-Länge), Kommandos 0x03 (Entladelimit) und 0x04 (Ladelimit): Bereichsprüfung `< 0x899` (2201) —
in v116 auf `< 0x9c5` (2501) angehoben. Alle übrigen 20+ Kommandos (0x01–0x60, 0xC1–0xCE) sowie das
BMS-Protokoll (`BMS_CAN_Parser`, PF-Werte 1–4) und der OTA-Subprotokoll-Teil (`0xCE`) sind unverändert.

### 4.2 RS485/Modbus

Dispatcher `Modbus_HandleRegisterAccess` (`0x080164d8`, 1158 B normalisierte Opcode-Länge); Aufbau,
Rückgabewerte und die komplette Registerkarte sind **identisch zu v116** — vollständige Tabelle siehe
v116-Dokument §4.2.

**Einzige inhaltliche Abweichungen:**

| Register (dez./hex) | v115-Wert | v116-Wert |
|---|---|---|
| 43133/43134 / `0xA87D`/`0xA87E` (`g_ratedPowerConfigA`/`B`) | Obergrenze `0x898` (2200) | `0x9c4` (2500) |
| 45502 / `0xB1BE` (Werks-Testbefehl „Entlade-Referenz") | `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)` | `(-2500.0)` |

Register 41010 (Direkter Leistungslimit-/Derating-Schalter, `800`↔`0x9C4`) und 44003
(`g_powerLimitDischargeW`, 800-W-Enforcement bei gesetztem `g_powerLimitDeratedFlag`) sind bereits in
v115 identisch zu v116 (Obergrenze dort unabhängig vom 2200/2500-Wechsel).

**Magic-Code-Register 0xAFC8 (45000):** Logik von `0x55DD`/`0x55EE`/`0x55FF` ist **byte-identisch zu
v116** (siehe v116-Dokument §4.2 für den vollständigen Code-Auszug und die Auflösung des
I2C-Richtungsbefunds). Insbesondere ist die Funktion an `0x08006cf0` in diesem v115-Export bereits
korrekt als `EEPROM_ReadRegister` benannt (I2C-Lesezugriff mit Mutex), `0x08006d8c` als
`EEPROM_ReadRegister_NoMutex` — der `0x55DD`-Zweig liest auch in v115 nachweislich
`g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode` aus EEPROM `0x102`/`0x300`/`0x301` zurück,
statt sie zu sichern.

### 4.3 Debug-Shell

`shellCommandTable` @ `0x08018930`, `ShellCommand[53]`, 42 `ShellCmd_*`-Handler — identisch zu v116
(v116-Dokument §4.3 für die vollständige Befehlsliste; alle Adressen dort verschieben sich in v115 um
die allgemeine Relokationsverschiebung von ca. `+0x10`…`+0x18` Byte im relevanten Flash-Bereich).
`ShellCmd_bat_mode` (`0x0801473c`, 38 B) und `ShellCmd_pf_set` (`0x08014f50`, 48 B) sind **explizit
byte-für-byte gegen v116 geprüft und identisch** (siehe Diff-Dokument) — kein neues Debug-Verhalten in
keiner der beiden Versionen.

### 4.4 I2C-EEPROM-Layout

Layout, Offsets und Feldbedeutungen identisch zu v116 (siehe v116-Dokument §4.4 für die vollständige
Tabelle). Werksreset-Defaultwerte bereits in v115 `"VD01"`, Versionsmarker `0x73` (=115), Lade-/
Entladelimit je `0x9C4` (2500 W) — der Werksreset-Defaultwert war demnach in **beiden** Versionen
2500 W; die 2200-W-Deckelung in v115 wirkt ausschließlich über die in §0/§4.1/§4.2/§5 genannten
Laufzeit-/Konfigurationsprüfungen, nicht über den EEPROM-Werksreset-Pfad selbst.

## 5. Regelung & Leistungsteil

Struktur identisch zu v116 (`Inv_CurrentLoopIsr` @ CCM, `Ctrl_MainControlIsr` @ CCM, HRTIM-ISR in
Flash) — siehe v116-Dokument §5 für die vollständige Beschreibung der Regelketten (PLL/SOGI,
Stromregler/PR-Kette, HRTIM-Bring-up).

**Die 2200-W-Konstante in CCM** (Float `0x45098000` = 2200.0f, in v116 durch `0x451C4000` = 2500.0f
ersetzt) liegt an zwei Stellen:

| Adresse (v115) | Kontext |
|---|---|
| `0x100029d8` | In `Ctrl_MainControlIsr`: Ausgangs-/Clamp-Limit des Leistungs-PI-Reglers beim Start der Netz-Zuschaltrampe (Zähler=120 Ticks), `Ctrl_PiControllerReset()` aufgerufen. |
| `0x10005180` | Basis der thermischen Derating-Formel: ab ~64 °C `(1.0-(T-64)/20)*2200.0`. |

**`Ctrl_ResetControlState`** (`0x10005f58`, 130 B) initialisiert dasselbe Leistungslimit-Feld
**bereits in v115** auf `2500.0` (Literal-Pool-Eintrag `0x10005fe8` = `0x451C4000`) — diese dritte,
in einer früheren Analyse-Iteration vermutete CCM-Änderungsstelle ist **kein** Unterschied zu v116
(dort ebenfalls `2500.0`, byte-identischer Funktionskörper bis auf Adressverschiebung; Details und
Gegenprobe im Diff-Dokument).

## 6. Schutz & Fehlercodes

`g_faultLogRing`, `FaultLog_Enqueue`, `Log_PrintFaultEntry` (`0x0800dba4`, 574 B normalisiert),
`Log_PrintDebugData` (`0x08010c48`), `Prot_LatchHwFaultToErrorWord` (CCM) — Fehlercode-Tabelle,
`err2`-Bit-Zuordnung und Zusatzdaten sind **identisch zu v116** (vollständige Tabelle siehe
v116-Dokument §6); die Codeadressen verschieben sich lediglich um die allgemeine Relokation.

## 7. OTA/Update

Strukturell identisch zu v116: YMODEM-Selbst-Update über Debug-UART, zusätzlicher CAN-Selbst-Update-
Pfad (`OTA_SelfUpdate_StateMachine`), BMS-Weiterleitung über CAN (`BMS_StartFwUpdateCan`/
`BMS_FwUpdate_StateMachine`) bzw. RS485 (`BMS_FwUpdateStateMachineTask`). `OTA_CompleteFirmwareUpdate`
(`0x0800f550`) ist laut Diff nur adressverschoben, inhaltlich identisch. Vollständige Beschreibung der
Zustandsautomaten siehe v116-Dokument §7.

**Fazit**: Sowohl das eigene Firmware-Update als auch die Weiterleitung an das BMS sind in v115
uneingeschränkt möglich — identisch zu v116.

## 8. Sicherheitsbefunde

Dieselben strukturellen Befunde wie in v116 (v116-Dokument §8), auf die v115-Baseline bezogen:

1. **Unauthentifizierter Debug-/Werksmodus-Zugriff per Modbus:** Register 45000 (`0xAFC8`) schaltet
   mit `0x55EE`/`0x55DD`/`0x55FF` unauthentifiziert in den Werksmodus bzw. persistiert/liest
   Betriebszustände (identisch zu v116, s. §4.2).
2. **Generischer Freischaltcode `0x55AA`** wirkt unauthentifiziert an mehreren Registern (identisch).
3. **Direkter Leistungslimit-Bypass:** Register 41010 (identisch zu v116, unabhängig vom 2200/2500-W-
   Unterschied) — in v115 wechselt der Bypass zwischen 800 W und dem damaligen Regel-Höchstwert
   2500 W (Register-Defaultwert), obwohl die *sonstigen* Konfigurationspfade zu diesem Zeitpunkt nur
   2200 W als Obergrenze vorsehen — dieselbe Inkonsistenz wie in §0 beschrieben.
4. **Default-Shell-Benutzer mit leerem Passwort** (identisch zu v116).
5. `EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex` sind in diesem Export bereits korrekt benannt
   (kein offener Punkt für v115, s. §4.2).
6. Kein direkter Netzwerkzugriff auf die Micro-Firmware selbst (identisch zu v116).

## 9. Chinesische Strings

184 UTF-8-Strings mit chinesischem Text, inhaltlich und in der deutschen Übersetzung **identisch zu
v116** (vollständige Tabellen siehe v116-Dokument §9) — alle Adressen verschieben sich um die
allgemeine Relokationsverschiebung dieser Version (z. B. `Log_PrintFaultEntry`-Strings ab `0800db08`ff.
statt `0800db08`ff. in v116, jeweils um wenige bis einige Dutzend Byte versetzt).

## 10. Offene Punkte

Gelten unverändert auch für v115 (identisches RAM-Layout, identische Restlogik) — vollständige Liste
siehe v116-Dokument §10. Ergänzend für v115 spezifisch:

- Bedeutung des Trailer-Worts am Image-Ende (`"I1DD"` in v115, `"13DD"` in v116) nicht ermittelt.
- Herstell-Changelog-Zeile „Angepasst an Control-Firmware 147" nicht gegen eine Vorversion (v114,
  nicht verfügbar) verifizierbar.

## 11. Korrekturen gegenüber der alten Analyse

Dieselben Korrekturen wie im v116-Dokument §11 gelten unverändert auch gegenüber der Altdokumentation
(Basisadresse um `0x4000` zu niedrig, MCU fälschlich als STM32F3 statt STM32G474, CCM-Codeanteil
fehlte vollständig, I2C-TX/RX-Vertauschung, `RCC_GetPLLOutputClockFreq`/`RTC_EnterInitMode`/
`RTC_ExitInitMode` fälschlich als I2C-Funktionen) — siehe dort für Details, sie betreffen die
Basis-Disassemblierung und damit beide Versionen gleichermaßen.

**Korrekturen speziell aus dem v115↔v116-Diff dieser Sitzung** (Details:
`VNSD_Micro_Diff_v115_v116_de.md`):

- `Ctrl_ResetControlState`: keine reale 2200→2500-W-Änderung — v115 lädt an dieser Stelle bereits
  `2500.0`, byte-identisch zu v116.
- `EMS_ResetMemoryByLevel` (v115, `0x080069a4`): in v116 nicht umbenannt oder entfernt, sondern nur
  ohne Funktionsnamen/-registrierung übernommen (`SUB_0800699c`) — Code und Aufruflogik aus
  `Inv_BuildTelemetryBlock` sind identisch. Eine frühere v116-Analyse-Iteration hatte diese Stelle
  fälschlich als neuen Programmpfad gewertet.
- `Ctrl_ApplyLlcPfcTestParams`/`Ctrl_ApplyPrControllerParams`/`Ctrl_ApplyVoltagePiParams` (nur in
  v115 als eigene Funktion): in v116 direkt in `ShellCmd_power_cfg`/`ShellCmd_qpr`/`ShellCmd_vol_pi`
  einkopiert (Compiler-Inlining) — keine Verhaltensänderung.
