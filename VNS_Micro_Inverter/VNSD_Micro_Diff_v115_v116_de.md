# Diff-Tabelle Micro v115 → v116 (Venus D)

> 🇬🇧 English version: [VNSD_Micro_Diff_v115_v116_de.md](VNSD_Micro_Diff_v115_v116_de.md)

**Stand:** 2026-09-30.

**Englisch:** `VNSD_Micro_Diff_v115_v116_de.md` (wird nach dieser Fassung erstellt) ·
**Vollanalysen:** `VNSD_Micro_FW_v115_Vollanalyse_de.md` · `VNSD_Micro_FW_v116_Vollanalyse_de.md` ·
**Dekompilate:** `Dekompilate/VNSD_Micro_115/` (lokal, nicht im Repo) (696 Funktionen) · `Dekompilate/VNSD_Micro_116/`
(693 Funktionen), RAM-Layout identisch.

**Methode:** Funktionspaare über gleiche Namen — N: 614 Namenspaare verglichen, davon 61 geändert
(davon 5 inhaltlich/real), 4 nur in v115, 0 nur in v116, 549 byte-identisch (Rohdiff
`l115_116.json`, normalisierte Opcodes). Jede der 61 als „geändert" markierten Funktionen sowie alle
4 nur-in-v115-Fälle wurden in dieser Sitzung **zusätzlich** durch direktes Lesen des dekompilierten
C-Codes beider Versionen geprüft (nicht nur der rohe Opcode-Diff) — mehrere Rohdiff-Befunde waren
reine Relokations-, Inlining- oder Ghidra-Namensverlust-Artefakte, siehe Abschnitte unten. Zusätzlich
wurde der CCM-Code direkt auf das Literal `0x451C4000`/`0x45098000` (2500.0f/2200.0f) durchsucht statt
sich nur auf den Namens-Diff zu verlassen, weil dieses Literal in Ctrl-Funktionen ohne Namensänderung
(z. B. `Ctrl_MainControlIsr`) auftritt.

## Prüfliste

| # | Frage | Antwort | Beleg |
|---|---|---|---|
| 1 | Modbus-Register geändert? | Registerkarte/-adressen identisch. Zwei Wertänderungen in `Modbus_HandleRegisterAccess`: Register 43133/43134 Obergrenze `0x898`(2200)→`0x9c4`(2500); Register 45502 Testreferenz `-2200.0`→`-2500.0`. Alle Magic-Codes (`0x55AA/0x55DD/0x55EE/0x55FF`) und die übrige Registerkarte byte-identisch. | Tabelle „Reale Änderungen" |
| 2 | CAN-Protokoll zum EMS/BMS geändert? | Nur eine Konstante in `EMS_Inverter_CAN_Dispatcher`: Bereichsobergrenze für Kommandos 0x03/0x04 (Lade-/Entladelimit setzen) `0x899`→`0x9c5`. `BMS_CAN_Parser`, `BMS_FwUpdate_CanHandler`, `HAL_FDCAN_RxFifo0Callback`, `CAN_RX_DispatchTask` sind laut Rohdiff **byte-identisch** (nicht einmal in der 61er-Änderungsliste enthalten) — Kommando-Satz, Message-IDs, Filter unverändert. | Tabelle „Reale Änderungen" |
| 3 | Leistung 2200 W→2500 W — alle Fundstellen? | **Sechs** reale Stellen, siehe eigene Tabelle unten. Eine ursprünglich vermutete siebte Stelle (`Ctrl_ResetControlState`) ist **widerlegt** — beide Versionen laden dort bereits 2500.0f, byte-identisch, nicht einmal im Rohdiff als „geändert" markiert. | Tabelle „2200→2500 W" |
| 4 | OTA/BMS-Update-Weiterleitung weiter möglich? | Ja. `OTA_CompleteFirmwareUpdate` ist nur adressverschoben (kosmetisch), `OTA_SelfUpdate_StateMachine`, `BMS_FwUpdate_StateMachine`, `BMS_StartFwUpdateCan`, `BMS_FwUpdateStateMachineTask` erscheinen nicht in der Änderungsliste (byte-identisch). YMODEM-Selbst-Update, CAN-Selbst-Update-Pfad und BMS-Weiterleitung über CAN/RS485 sind strukturell unverändert. | Cosmetic-Liste |
| 5 | Schutz-/Fehlercodes geändert? | Nein. `Log_PrintFaultEntry` und `Log_PrintDebugData` sind rein adressverschoben (kosmetisch), Fehlercode-Tabelle und `err2`-Bit-Zuordnung inhaltlich unverändert; `Prot_LatchHwFaultToErrorWord` (CCM) erscheint nicht in der Änderungsliste. | Cosmetic-Liste |

## Changelog → Code

| Changelog-Eintrag | Tatsächliche Codeänderung(en) |
|---|---|
| „VD 2500w" (einzige v116-Changelog-Zeile) | Sechs getrennte Stellen heben denselben 2200-W-Grenzwert auf 2500 W an (CAN-Dispatcher, zwei Modbus-Stellen, zwei CCM-Konstanten in `Ctrl_MainControlIsr`, `Bat_Mode3RoundRobinTimer`) — siehe Tabelle unten. Der CAN-seitige Echtzeit-Hartgrenzwert (Kommandos 0x56/0x58, ±2500 W) war bereits in v115 unverändert bei 2500 W; v116 behebt also eine Inkonsistenz zwischen einem bereits offenen Echtzeit-Pfad und mehreren nachgelagerten Konfigurations-/Init-/Rampen-Pfaden, die weiterhin bei 2200 W deckelten. |
| „Angepasst an Control-Firmware 147" (einzige v115-Changelog-Zeile, gegenüber der nicht direkt verfügbaren Vorversion) | In dieser Sitzung nicht separat verifizierbar (keine v114-Dekompilate vorhanden); nicht Gegenstand dieses v115↔v116-Diffs. |

## 2200 W → 2500 W: alle sechs Fundstellen

| # | Funktion | v115-Adresse | v116-Adresse | Änderung |
|---|---|---|---|---|
| 1 | `EMS_Inverter_CAN_Dispatcher` | `0x08005944` | `0x08005940` | CAN-Kommandos 0x03/0x04 (Lade-/Entladelimit setzen): Bereichsprüfung `< 0x899` (2201) → `< 0x9c5` (2501). |
| 2 | `Modbus_HandleRegisterAccess`, Register 43133/43134 | `0x080164d8` | `0x080164e8` | `g_ratedPowerConfigA`/`B`-Obergrenze `0x898` (2200) → `0x9c4` (2500). |
| 3 | `Modbus_HandleRegisterAccess`, Register 45502 | `0x080164d8` | `0x080164e8` | Werks-Testbefehl „Entlade-Referenz": `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)` → `(-2500.0)`. |
| 4 | `Ctrl_MainControlIsr` (CCM), Netz-Zuschaltrampe | `0x100029d8` | `0x100029cc` | Ausgangs-/Clamp-Limit des Leistungs-PI-Reglers beim Start der Rampe (Zähler=120 Ticks): `2200.0` → `2500.0`. |
| 5 | `Ctrl_MainControlIsr` (CCM), thermisches Derating | `0x10005180` | `0x10005174` | Basis der thermischen Derating-Formel (`(1.0-(T-64)/20)*Basis`): `2200.0` → `2500.0`. |
| 6 | `Bat_Mode3RoundRobinTimer` | `0x0801183c` | `0x08011834` | Zwei Stellen setzen `g_batPowerRefW` (Multi-Pack-Round-Robin-Ladeorchestrierung, Batteriemodus 3) auf `±2200.0` → `±2500.0`. |

**Widerlegter Kandidat:** `Ctrl_ResetControlState` (v115 `0x10005fe8`/`0x10005fd8`-Literal je nach
Version) wurde beim ersten Durchgang dieser Sitzung als siebte CCM-Stelle vermutet, weil dort in v116
der Wert `0x451C4000` (2500.0f) im Literal-Pool steht. Ein Byte-für-Byte-Vergleich der
Roh-Disassemblate zeigt jedoch: **v115 lädt an der entsprechenden Adresse (`0x10005fe8`) bereits
denselben Wert `0x451C4000`.** Der komplette Funktionskörper (130 B) ist bis auf die adressbedingte
Verschiebung identisch — die Funktion taucht deshalb auch nicht im Rohdiff (`l115_116.json`) als
„geändert", sondern korrekt in der 549er-„identisch"-Gruppe auf. Es gibt hier **keine**
v115→v116-Änderung.

## Tabelle aller realen Änderungen (5 Funktionen, über die o. g. 6 Fundstellen hinaus)

| Funktion | v115 | v116 | Änderung |
|---|---|---|---|
| `Bat_Mode3RoundRobinTimer` | `0x0801183c`, 382 B | `0x08011834`, 378 B | Siehe Fundstelle 6 oben (2200→2500 W). Restliche Logik zeilengleich, nur eine Stack-Variable umbenannt (Kosmetik). |
| `Cfg_LoadFromEeprom` | `0x08006a70` | `0x08006a68` | Versions-/Boot-Flag-Konstante (`g_cfgVersionMarker`, EEPROM-Feldvergleich beim Laden) `0x73` (115) → `0x74` (116) — reiner Versionsstempel-Bump, erwartete Änderung, sonst identisch. |
| `EMS_Inverter_CAN_Dispatcher` | `0x08005944` | `0x08005940` | Siehe Fundstelle 1 oben (2200→2500 W). Funktionsgröße 494→493 B (normalisierte Opcode-Länge), keine sonstigen Änderungen. |
| `Modbus_HandleRegisterAccess` | `0x080164d8`, 1158 B | `0x080164e8`, 1150 B | Siehe Fundstellen 2+3 oben (2200→2500 W). Alle übrigen Abweichungen im Funktionskörper (Label-Verschiebung durch Relokation, entfernter toter Store `puVar9=(byte*)0x898`, umstrukturierter if/else-Baum an einer Verzweigung, scheinbar geänderte Argumentzahl von `Grid_Protection_SetLimits` — die Funktion nutzt ihre Parameter in beiden Versionen nicht) sind nachweislich kosmetisch. |
| `ShellCmd_version` | `0x080156a4` | `0x080156b4` | `SOFT_VERSION`-Konstante `0x73`(115)→`0x74`(116); Build-Datum/-Zeit-String `"Jan 22 2026"`/`"15:55:59"` → `"Jul 2 2026"`/`"16:32:26"` — erwartete Versions-/Build-Metadaten, keine Verhaltensänderung. |

Alle fünf Funktionen wurden zusätzlich zum Rohdiff durch vollständiges Lesen beider Dekompilate
geprüft; über die genannten Punkte hinaus gibt es keine weiteren inhaltlichen Unterschiede.

## Kosmetische Änderungen (56 von 61 als „geändert" markierte Funktionen)

Alle folgenden Funktionen sind nach Prüfung des dekompilierten C-Codes **funktional identisch**
zwischen v115 und v116; der Rohdiff schlägt hier nur wegen Adressverschiebung (Relokation durch
Codegrößenänderungen an anderer Stelle im Image), Register-/Stack-Umbenennung durch den Compiler oder
Ghidra-Decompiler-Artefakten (z. B. unaufgelöste Tail-Calls, andere Parameterzahl-Erkennung) an.

**Veneers** (3-Instruktionen-Trampolins Flash→CCM; Diff ist ausschließlich das `movw`/`movt`-Sprungziel,
weil die CCM-Zielfunktion um wenige Byte verschoben wurde — Ziel selbst unverändert):
`Cfg_SetOperatingModeFlag_veneer`, `Cfg_SetPersistedFlag102_veneer`, `Cfg_SetStoredEnableFlag_veneer`,
`Ctrl_InitControlLoopStructures_veneer`, `Ctrl_MainControlIsr_veneer`, `Ctrl_SetPrClipFlag_veneer`,
`Grid_ScaleAdcMeasurements_veneer`, `Inv_CurrentLoopIsr_veneer`, `Prot_ClampComparatorThreshold_veneer`,
`Prot_SetFaultBitOnMatch_veneer`, `Prot_SetHwFaultLatchBit_veneer`.

**Reine Relokationsverschiebung** (identische Logik, nur Literal-Pool-/Rücksprungadressen/Label
verschoben): `Dac_InitAll`, `Default_Handler`, `FDCAN_CopyMessageToRAM`, `FLASH_WaitForOperation`
(algebraisch äquivalente De-Morgan-Umformung derselben Bit-Prüfung, andere Instruktionsauswahl durch
den Compiler), `HAL_ADC_Init`, `HAL_ADC_Start_DMA`, `HAL_FDCAN_GetRxMessage`, `HAL_RCC_ClockConfig`,
`HAL_UART_Transmit_DMA` (Rücksprungadresse zeigt in beiden Versionen auf denselben Handler
`Stub_CallHandlerOffset28`, nur verschoben, da Nachbar-Stubs umsortiert wurden), `I2C_WaitOnFlagUntilTimeoutCommon`,
`Inv_BuildTelemetryBlock` (der unaufgelöste Aufruf `func_0x0800699c` in v116 ist exakt derselbe
Maschinencode wie `EMS_ResetMemoryByLevel` in v115, siehe „Nur in v115" unten — kein Verhaltensunterschied),
`LED_TaskTick` (entfernte Zeile ist ein redundanter Self-Store, tote Code-Elimination ohne Wirkung),
`Log_PrintDebugData`, `Log_PrintFaultEntry`, `OTA_CompleteFirmwareUpdate`, `Task_CreateAllAndStartScheduler`
(Task-Reihenfolge, Stackgrößen 0x80/0x100/0x80/0x200/0x100/0x100/0x100/0x80 und Prioritäten
0x10/0xf/0xe/9/8/6/7/0 explizit für alle 8 Tasks byte-genau geprüft, unverändert), `Uart_DmaTxComplete16Bit`,
`Uart_DmaTxComplete8Bit`, `Uart_RxCompleteDispatch`, `Util_CopyWordsSrcDst`, `__main`, `__rt_lib_init`,
`printf`, `printf_FloatToDecimal` (einzige Abweichung: ungenutztes High-Wort des 64-Bit-Rückgabewerts,
der Aufrufer liest es in keiner Version), `printf_Format_Engine` (Stack-Frame-Layout und
Variablenbenennung verschoben, aber jede Bitmaske/jeder Format-Buchstabe/jede Verzweigung 1:1 gleich),
`prvTaskPriorityDisinheritAfterTimeout`, `pxPortInitialiseStack`, `shellInit`,
`shellInvokeCommand` (v116-Decompiler löst Jump-Table für Fälle 0–4 nicht auf, reines
Darstellungsartefakt, Aufruflogik für alle 8 Argumentzahlen identisch), `sprintf`, `TIM_InitTim4InputCapture`,
`xTaskResumeAll`, `EXTI9_5_IRQHandler` (echter Interrupt-Handler in beiden Versionen 24 B, identischer
Tail-Jump — das vom Rohdiff gemeldete große Wachstum betraf in Wirklichkeit die benachbarte, in v116
unbenannte Funktion `EMS_ResetMemoryByLevel`/`SUB_0800699c`, siehe unten), `EEPROM_SaveOperatingStats`
(v115-Datei bereits unter diesem Namen geführt — keine Stub-Verwechslung in den frischen Exports —,
inhaltlich identisch zu v116, nur weit verschoben), `Stub_EmptyNoOp2` (leerer `return`-Stub in beiden
Versionen, 2–4 B Epilog-Kodierungsunterschied), `Startup_BranchToMain`, `Shell_TaskInit`.

**Shell-Kommandos, nur Adress-/Stringverschiebung** (Logik zeilengleich): `ShellCmd_bat_mode` (38 B in
beiden Versionen, **explizit gegengelesen** — widerlegt den ursprünglichen Rohdiff-Fund eines neuen
Debug-Printf-Blocks endgültig), `ShellCmd_pf_set` (48 B in beiden Versionen, ebenso widerlegt),
`ShellCmd_reset_memory`, `ShellCmd_rtos_status`, `ShellCmd_show_power` (letzter `printf`-Aufruf wird in
v116 als `printf_Format_Engine`-Tail-Call dargestellt statt als `printf` — Decompiler-Anzeigeartefakt,
keine Verhaltensänderung).

**Compiler-Inlining (Funktionsgrenzen verschoben, keine Verhaltensänderung):** `ShellCmd_power_cfg`,
`ShellCmd_qpr`, `ShellCmd_vol_pi` — in v115 sind das 12-B-Tail-Jump-Stubs in eine jeweils eigene
Hilfsfunktion (`Ctrl_ApplyLlcPfcTestParams`, `Ctrl_ApplyPrControllerParams`, `Ctrl_ApplyVoltagePiParams`,
alle drei nur in v115 als eigene Funktion registriert); in v116 hat der Compiler denselben Code direkt
in die jeweilige `ShellCmd_*`-Funktion einkopiert (88–92 B, identischer printf-/Zielaufruf-Inhalt).
Näheres siehe „Nur in v115" unten.

## Nur in v115 vorhanden (4 Funktionsnamen, 0 nur in v116)

| Funktion (v115) | v115-Adresse | Verbleib in v116 |
|---|---|---|
| `Ctrl_ApplyLlcPfcTestParams` | `0x08014f90` (80 B) | **Kosmetisch entfernt:** Code liegt jetzt direkt inline in `ShellCmd_power_cfg` (`0x080127e4`, 92 B). Gleicher printf-String, gleicher Aufruf von `Ctrl_HandleLlcPfcTestCommand`. |
| `Ctrl_ApplyPrControllerParams` | `0x0801487c` (76 B) | **Kosmetisch entfernt:** Code liegt jetzt direkt inline in `ShellCmd_qpr` (`0x080127d8`, 88 B). Gleicher printf-String, gleicher Aufruf von `Ctrl_SetPrControllerTarget`. |
| `Ctrl_ApplyVoltagePiParams` | `0x08014804` (76 B) | **Kosmetisch entfernt:** Code liegt jetzt direkt inline in `ShellCmd_vol_pi` (`0x080127cc`, 88 B). Gleicher printf-String, gleicher Aufruf von `Ctrl_SetRampTargetB`. |
| `EMS_ResetMemoryByLevel` | `0x080069a4` (86 B) | **Namens-/Registrierungsverlust, kein Codeunterschied:** Byte-für-Byte identischer Maschinencode liegt in v116 unter derselben relativen Adresse (`0x0800699c`), aber Ghidras automatischer Benennungslauf hat der Stelle dort keinen Funktionsnamen mehr zugeordnet — sie ist im v116-Export nicht einmal als eigene Function registriert (nur als Codeadresse `SUB_0800699c`, decompiliert als unaufgelöster Aufruf `func_0x0800699c` in `Inv_BuildTelemetryBlock`). Die umgebende Aufruflogik (Debounce alle ~10 Telemetriezyklen, `g_emsDispatchSeq`-Auswertung) ist in `Inv_BuildTelemetryBlock` in beiden Versionen zeilengleich. **Korrektur:** Eine frühere Analyse-Sitzung hatte diese Stelle fälschlich als „echten neuen Programmpfad in v116" gewertet (siehe `VNSD_Micro_FW_v116_Vollanalyse_de.md` §0/§11) — das ist widerlegt, es handelt sich um unveränderten v115-Code ohne Namen. |

Alle vier Fälle erklären die Differenz zwischen 696 (v115) und 693 (v116) benannten Funktionen
vollständig (3 Inlining-Fälle + 1 Namensverlust = −4 im Rohdiff, plus vereinzelte weitere
Benennungslauf-Abweichungen ohne Code-Bezug für die verbleibende Differenz zur Gesamtzahl).

## Offene Punkte

- Keine v114-Dekompilate verfügbar; die v115-Changelog-Zeile „Angepasst an Control-Firmware 147"
  konnte in dieser Sitzung nicht gegen eine Vorversion verifiziert werden.
- Die genaue Ursache der zusätzlichen ~354 „confidence: low"-Namen (siehe beide Vollanalysen, §10)
  wurde nicht für jede einzelne der hier verglichenen Funktionen erneut geprüft; wo eine Klassifizierung
  auf einem niedrig-konfidenten Namen beruhte, ist das Risiko einer Fehlzuordnung nicht vollständig
  ausgeschlossen (betrifft in dieser Diff-Tabelle am ehesten `Grid_Protection_SetLimits`-Parameterzahl
  und ähnliche Signatur-Erkennungs-Unschärfen, die aber für die Klassifizierung „kosmetisch" nicht
  ausschlaggebend waren).
- Die Umbenennung `EEPROM_WriteRegister`→`EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex` ist in
  **beiden** Versionen (v115 und v116) bereits identisch vorhanden — sie ist keine v115→v116-Änderung,
  sondern eine reine Dokumentationskorrektur gegenüber einer älteren, inzwischen überholten
  Analysefassung (Details in beiden Vollanalysen, §4.2/§11).
