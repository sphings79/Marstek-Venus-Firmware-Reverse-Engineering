# VNSD BMS — Diff Beta 117.7 → 118

> 🇬🇧 English version: [VNSD_BMS_Diff_v1177_v118_de.md](VNSD_BMS_Diff_v1177_v118_de.md)

**Stand:** 2026-09-30.

**Englisch:** `VNSD_BMS_Diff_v1177_v118_de.md` (wird nach dieser Fassung erstellt) ·
**Vollanalysen:** `VNSD_BMS_FW_v1177_Vollanalyse_de.md` (117.7, diese Sitzung) ·
`VNSD_BMS_FW_v118_Vollanalyse_de.md` (118, wird parallel von anderer Stelle erstellt — hier nur verlinkt,
nicht editiert) ·
**Verwandt:** `VNSD_Ladegrenzen_Ladeleistung_de.md` (Ladegrenzen BMS/Control/Micro, verifiziert) ·
**Export/Quelle:** `Dekompilate/VNSD_BMS_1177/` (lokal, nicht im Repo), `Dekompilate/VNSD_BMS_118/` (je `<addr>_<name>.c`,
`_index.md`, `listing/` mit Flash-Rohdisassemblat + `globals_and_registers.txt`)

## Beta-Hinweis (Voraussetzung für dieses Dokument)

Im Firmware-Archiv (`Marstek FW Archiv/firmwares/VNSD-0/BMS/`) liegen für das BMS nur zwei Images:
`1177` (Beta) und `118`. **Ein Vorgänger-Image für die 117.7-Beta liegt nicht vor** — der naheliegende
Vorgänger wäre die stabile Version **BMS v117**, die im Archiv fehlt. Ein Diff 117 → 117.7 ist damit
**nicht möglich**. Dieses Dokument vergleicht ausschließlich **117.7 → 118** (den Nachfolger). Das
Changelog (`CHANGELOG.de.md` (lokal, nicht im Repo)) nennt für beide Stände nur "OTA-Test" (117.7 zusätzlich "7"), keine
inhaltliche Beschreibung — die tatsächlichen Änderungen mussten vollständig aus dem Code ermittelt
werden.

Die Beta-Natur von 117.7 zeigt sich auch im Code selbst: `PerPack_Struct_Builder` meldet in 117.7 den
Firmware-Versionswert `0x499` (1177 dezimal, d. h. "117.7" als durchlaufende Beta-Zahl `117.7 × 10`),
in 118 dagegen schlicht `0x76` (118) — zwei unterschiedliche Zählweisen für dieselbe Größe.

## 0. Kurzfassung

**Methode:** Ausgangspunkt ist der normalisierte Listing-Diff `l1177_118.json` (622 Funktionen
identisch, 88 geändert, 1 nur in 118 neu: `SOC_OcvBoundsRecalibrateOnFlag`). Für **jedes** der 88
geänderten Funktionspaare wurde zusätzlich der dekompilierte C-Code beider Versionen
Zeile für Zeile verglichen (Dateien gleichen Namens in `Dekompilate/VNSD_BMS_1177/` bzw.
`VNSD_BMS_118/`), um reine Adress-/Registernamen-Artefakte von echten Verhaltensänderungen zu trennen.

**Ergebnis: 11 von 88 gelisteten Änderungen sind real, 77 sind kosmetisch.** Zusammen mit der einen
komplett neuen Funktion `SOC_OcvBoundsRecalibrateOnFlag` ergeben sich **12 reale Änderungen**
insgesamt. Alle 77 kosmetischen Fälle sind entweder (a) reine Adresskonstanten, die sich durch die
Codegrößenänderung (+88 B netto durch die neue Funktion, +Erweiterungen an drei Stellen) automatisch
verschieben, (b) Decompiler-Registerbenennungen (`fVar2`/`fVar3`, `uStack_18`/`local_18` usw.) ohne
Bedeutung, oder (c) Folgeverschiebungen durch einen einzigen neuen 2-Byte-RAM-Global
(`g_dischargeStepRotateLimitCache`, s. §2.5). Jeder dieser 77 Fälle wurde einzeln am C-Code geprüft,
nicht nur am Listing-Diff geschätzt — bei einer Funktion (`PackMode_Shutdown_Handler`) hätte eine reine
Listing-Heuristik "Adressumbenennung" fälschlich als kosmetisch durchgehen lassen; erst der C-Vergleich
zeigte die reale Timing-Verschiebung (§2.2).

**Die wichtigsten realen Änderungen:**

1. **Entladelimit-Prozent (`g_dischargeLimitPct`) wird jetzt zusätzlich über den periodischen
   CAN-Heartbeat propagiert** und die davon abhängige Rotationslogik wurde repariert/erweitert
   (`CAN_RX_Handler`, `PackSelect_DischargeStepThreshold`, `PackSelect_Discharge_StepRotate`) — §2.1.
2. **Neue OCV-Rekalibrierungsfunktion `SOC_OcvBoundsRecalibrateOnFlag`**, eingehängt in
   `SOC_AlgorithmOrchestrator`, plus zwei neue RTC/EEPROM-Zeitstempel-Log-Einträge beim
   Moduswechsel in den Leerlauf — §2.3. **Einschränkung:** Das von der neuen Funktion konsumierte Flag
   `g_socOcvRecalibrateRequest` wird in keiner der 710/711 Funktionen jemals auf 1 oder 2 gesetzt —
   die Funktion ist nach aktuellem Kenntnisstand **totem Code**, der eigentliche OCV-Rekalibrierungspfad
   bleibt der bereits in 117.7 vorhandene 1-Stunden-Leerlauf-Timer in `SOC_AlgorithmOrchestrator`.
3. **Neues Fehler-Zeitstempel-Log beim Notreset** (`System_Reset_And_Halt_WithCurrentCheck`), wenn
   die zuletzt gemessene Pack-Leistung außerhalb ±5 kW lag — §2.4.
4. **Konfig-/Protokoll-Versionskennung `'u'`(117)→`'v'`(118)** an sechs Stellen synchron angehoben
   (Config-Guard, RS485-Version, Pack-Telemetrie) — §2.6.
5. **Shutdown-Timing** (`PackMode_Shutdown_Handler`): `g_pendingActivationFlag` wird erst nach den
   300 Ticks Wartezeit statt sofort gesetzt — §2.2.

**Keine Änderungen** an: KA495xx-SPI/CRC-Treiber, Ladestrom-/Spannungs-Grenztabellen (50 A, 57,6 V/
48,0 V), Kernprotokoll von CAN/RS485 (Kommandosätze, Message-IDs), OTA/Bootloader-Handshake
(`g_bootConfigPage`, Staging-Adresse, `'BMS1'`-Tag-Prüfung), Shell-Kommandotabelle (36 Einträge,
Nutzer `BLUE`), FreeRTOS-Taskliste (10 Tasks, gleiche Prioritäten/Stacks/Handles).

## 0a. Checkliste

| # | Frage | Antwort 117.7 → 118 | Beleg |
|---|---|---|---|
| 1 | CAN-Protokoll geändert? | Kommandosatz und Message-IDs unverändert. Eine reale Ergänzung: der periodische 0xfd00-Heartbeat-Frame (Master-Enable) überträgt jetzt zusätzlich `g_dischargeLimitPct` in Byte `frame+0xd` (nur RAM-Update, kein Flash-Write — die dedizierte CAN-Kommando-Route `0x100301aa` bleibt der Flash-persistente Pfad). | §2.1, Tabelle §1 (`CAN_RX_Handler`) |
| 2 | RS485 geändert? | Kommandosatz/Dispatcher-Struktur unverändert. Zwei Versions-Antwortbytes angehoben (`0x75`→`0x76`), sonst nur Adressverschiebungen. | §2.6, Tabelle §1 |
| 3 | Lade-/Entladegrenzen geändert? | Die harten Grenzen (50 A, 57,6 V/48,0 V, Temperatur×SOC-Matrix) sind **unverändert** — siehe `VNSD_Ladegrenzen_Ladeleistung_de.md`. Geändert wurde nur, **wie** ein bereits vorhandener konfigurierbarer Entladelimit-Prozentsatz (`g_dischargeLimitPct`) in der Pack-Rotationslogik wirksam wird (§2.1) — vorher hing `PackSelect_DischargeStepThreshold` gar nicht von diesem Global ab. | §2.1 |
| 4 | OTA-/Bootloader-Handshake geändert? | Unverändert: Image-Basis `0x08002800`, Versions-Tag `'BMS1'` @ `0x08010000`, Staging-Kopie @ `0x0801E000`, Tag-Prüfung @ `0x0802B800`, `BMS_OTA_Flash_Writer`/`_FinalizeTail` nur adressverschoben. | Tabelle §1 (`BMS_OTA_Flash_Writer_FinalizeTail`, `OTA_Set_Reboot_Params`, `HAL_Flash_Write_Config_Block`) |
| 5 | Shell-Kommandos geändert? | Unverändert (36 Einträge, Standardnutzer `BLUE`); die einzige geänderte Shell-nahe Funktion (`CLI_ShellInit`) unterscheidet sich nur in lokalen Variablennamen. | Tabelle §1 |
| 6 | RAM-Layout geändert? | `.bss` in 118 um **16 B größer** (`0x9fcc`→`0x9fdc`), FreeRTOS-Heap in 118 **8 B kleiner** (`0x4FF8`→`0x4FF0`); ein neuer 2-B-Global (`g_dischargeStepRotateLimitCache`) verschiebt sechs benachbarte Timer-/Zustands-Globals um +2 B. Die restlichen Bytes der Verschiebung sind nicht auf einzelne neue Funktionen zurückzuführen (Offener Punkt, §4). | §2.5 |

## 1. Tabelle aller 88 geänderten Funktionspaare

Adresse 117.7 / Adresse 118 / Funktion / real oder kosmetisch / was geändert wurde. Zusätzlich neu in
118 (nicht in dieser Tabelle, da kein Funktionspaar): `SOC_OcvBoundsRecalibrateOnFlag` @ `0x08016634`
(146 B) — siehe §2.3.

| Addr 117.7 | Addr 118 | Funktion | Real/Kosmetisch | Was geändert wurde |
|---|---|---|---|---|
| `0x08018eb8` | `0x08018ff0` | `APP_RegisterCallbackSlot` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08003188` | `0x08003188` | `APP_ShellTable_Init_Then_Main` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08012cac` | `0x08012d1c` | `APP_SystemFaultHalt` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080029a0` | `0x080029a0` | `APP_SystemFaultHalt_thunk` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800a920` | `0x0800a98c` | `BMS_Config_Params_Factory_Reset` | **real** | Config-Versions-Tag `g_cfgVersionTag` von `'u'` (0x75=117) auf `'v'` (0x76=118) angehoben. |
| `0x08009c60` | `0x08009ccc` | `BMS_Config_Params_Load_Or_Default` | **real** | Guard-Vergleich und Rückschreibwert des Config-Versions-Tags von `0x75` auf `0x76` angehoben (erzwingt Factory-Reset des Parameterblocks bei Tag-Mismatch). |
| `0x080096c8` | `0x08009734` | `BMS_Measurements_Valid_Check` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800d440` | `0x0800d4ac` | `BMS_OTA_Flash_Writer_FinalizeTail` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08009540` | `0x080095ac` | `BMS_ProtectionAndCommTick` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800e120` | `0x0800e18c` | `CAN_Pack_Activation_Config_Receive` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800f0bc` | `0x0800f128` | `CAN_RX_Command_Dispatcher` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080081dc` | `0x08008240` | `CAN_RX_Handler` | **real** | Im 0xfd00-Heartbeat-Zweig (Master-Enable) zusätzliche Zeile `g_dischargeLimitPct = frame[0xd]` — Entladelimit-Prozent wird jetzt auch über den periodischen Heartbeat aktualisiert (nur RAM, kein Flash-Write). |
| `0x0800db88` | `0x0800dbf4` | `CAN_RxSlavePackTelemetry_Unpack` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08007bd8` | `0x08007c3c` | `CAN_TX_PF2_Capacity` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08014470` | `0x080144e0` | `CLI_FindActiveShellInstance` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08014420` | `0x08014490` | `CLI_FormatVariableValueString` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08016044` | `0x080160e0` | `CLI_PrintSocStatus` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08010bcc` | `0x08010c2c` | `CLI_Printf_Wrapper` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800ec3c` | `0x0800eca8` | `CLI_Process_RX_Buffer` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08012c9c` | `0x08012d0c` | `CLI_SetDefaultCommandHandler` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801474c` | `0x080147bc` | `CLI_ShellInit` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800cd18` | `0x0800cd84` | `Cell_OVP_UVP_TempCompensated_Protection` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08002a86` | `0x08002a86` | `Default_Handler` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800ff00` | `0x0800ff6c` | `FATFS_Init_Log_File_If_Needed` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080129e4` | `0x08012a54` | `Full_OCV_Lookup` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800ff98` | `0x08010bd8` | `HAL_Flash_Write_Config_Block` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation); die Literal-Adressrechnung `uVar2*4+0x8002000` in 117.7 entspricht `&g_bootConfigPage+uVar2` in 118 — derselbe Wert, nur Symbolauflösung unterschiedlich. |
| `0x0800fffa` | `0x0800fff8` | `HAL_NoOpStub` | kosmetisch | Kein Codeunterschied; Grenzverschiebung zum benachbarten `'BMS1'`-Versions-Tag-Wort infolge der Relozierung von `__scatterload_copy` (s. Kurzfassung). |
| `0x08016974` | `0x08016ab0` | `KA495XX_CellBalanceChannelConfig` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08016a58` | `0x08016b94` | `KA495XX_Crc8` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080166c4` | `0x08016800` | `KA495XX_VerifyConfigRegisterTable` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08018a02` | `0x0800fffa` | `LIB_FreeRTOS_Clear_ListItem_Value` | kosmetisch | Kein Codeunterschied; Grenzverschiebung zum benachbarten `'BMS1'`-Versions-Tag-Wort infolge der Relozierung von `__scatterload_copy` (s. Kurzfassung). |
| `0x080125f0` | `0x08012660` | `Max_Charge_Current_Calc` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080126e8` | `0x08012758` | `Max_Discharge_Current_Calc` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800e1e8` | `0x0800e254` | `NTC_Lookup_Table_1` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800e3d8` | `0x0800e444` | `NTC_Lookup_Table_2` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080115a8` | `0x08011618` | `OTA_Set_Reboot_Params` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08006b60` | `0x08006bc4` | `OwnPack_MOSFET_Driver` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08007220` | `0x08007284` | `OwnPack_MOSFET_Driver_MasterVariant` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800f324` | `0x0800f390` | `PackMode_Shutdown_Handler` | **real** | `g_pendingActivationFlag=1` von State 0 (sofort) ans Ende von State 1 (erst nach den 300 Ticks Wartezeit) verschoben — reale Timing-Änderung der Shutdown-Zustandsmaschine. |
| `0x0801070c` | `0x0801070c` | `PackRecovery_LowCell_Charge_SM` | kosmetisch | Reine Folge der RAM-Verschiebung um 2 B durch den neuen Global `g_dischargeStepRotateLimitCache` (s. §2.5) — dieselben Felder, nur andere Adresse. |
| `0x08009354` | `0x080093bc` | `PackSelect_DischargeStepThreshold` | **real** | Formel komplett ersetzt: statt fester Stufentabelle (Schrittweite 5 je 10 % SOC, Minimum 9/10) wird der Schwellwert jetzt aus `100 - g_dischargeLimitPct` abgeleitet — Rotationsschwelle folgt jetzt dem konfigurierbaren Entladelimit. |
| `0x080063a0` | `0x080063dc` | `PackSelect_Discharge_StepRotate` | **real** | Neuer RAM-Cache `g_dischargeStepRotateLimitCache`: erkennt eine Laufzeit-Änderung von `g_dischargeLimitPct` und erzwingt dann eine Neuberechnung der Rotationsschwelle statt den Timer immer wieder auf 0 zurückzusetzen. |
| `0x08006174` | `0x080061b0` | `PackSelect_Mode1_Full` | kosmetisch | Reine Folge der RAM-Verschiebung um 2 B durch den neuen Global `g_dischargeStepRotateLimitCache` (s. §2.5) — dieselben Felder, nur andere Adresse. |
| `0x080062b8` | `0x080062f4` | `PackSelect_Mode2_LowSoc` | kosmetisch | Reine Folge der RAM-Verschiebung um 2 B durch den neuen Global `g_dischargeStepRotateLimitCache` (s. §2.5) — dieselben Felder, nur andere Adresse. |
| `0x08006008` | `0x08006044` | `PackSelect_Mode3_Empty` | kosmetisch | Reine Folge der RAM-Verschiebung um 2 B durch den neuen Global `g_dischargeStepRotateLimitCache` (s. §2.5) — dieselben Felder, nur andere Adresse. |
| `0x08007174` | `0x080071d8` | `PackSelect_Next_AboveSocThreshold` | kosmetisch | Reine Folge der RAM-Verschiebung um 2 B durch den neuen Global `g_dischargeStepRotateLimitCache` (s. §2.5) — dieselben Felder, nur andere Adresse. |
| `0x08003798` | `0x08003798` | `Pack_Online_Watchdog` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800c770` | `0x0800c7dc` | `PerPack_Struct_Builder` | **real** | Gemeldeter FW-Versionswert `g_pack0FwVersionTag` im Pack-Telemetrie-Record von `0x499` (1177, Beta-Zählweise) auf `0x76` (118) geändert. |
| `0x0800e608` | `0x0800e674` | `RCC_GetClockFrequencies` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08010004` | `0x08010004` | `RS485_Dispatcher` | **real** | Versions-Antwortbyte im Dispatcher-Puffer von `0x75` auf `0x76` angehoben (gleiche Config-Versions-Konstante wie oben). |
| `0x080123a4` | `0x08012414` | `RS485_Pack_Telemetry_Response` | **real** | Versionsbyte in der Pack-Telemetrie-Antwort (Offset 0x48) von `'u'` auf `'v'` angehoben. |
| `0x0800d888` | `0x0800d8f4` | `RS485_Protocol_Byte_Receiver` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08004ba8` | `0x08004be4` | `RS485_TxComplete_DMA_Callback` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800518c` | `0x080051c8` | `RS485_USART2_Init_DMA_RingBuffer` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0800e750` | `0x0800e7bc` | `RTOS_CreateAllTasks` | kosmetisch | Nur die 10 Task-Einstiegsadressen verschoben (Relozierung); Stacks, Prioritäten und Handles byte-genau identisch geprüft. |
| `0x08015d20` | `0x08015d90` | `SOC_AlgorithmOrchestrator` | **real** | Neuer Aufruf `SOC_OcvBoundsRecalibrateOnFlag(soc_state)` zu Beginn des Ticks; zusätzlich zwei neue RTC+EEPROM-Zeitstempel-Schreibvorgänge beim Eintritt in den Idle-Modus und beim Idle-Timeout (vorher nur SOH/Zykluszähler-Update ohne Zeitstempel). |
| `0x08012ce8` | `0x08012d58` | `SOC_InitCapacityParams` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08013d58` | `0x08013dc8` | `SOC_ResetEstimatorToPercent` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08015fdc` | `0x08016078` | `SOC_SaveCalibrationToFlash` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08016ac0` | `0x08016bfc` | `SOC_UpdateDisplayValueAndFullFlag` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08013cc4` | `0x08013d34` | `SOH_LookupByCycleCount` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08004998` | `0x08004998` | `System_Reset_And_Halt_WithCurrentCheck` | **real** | Neue Bedingung vor dem Reset: wenn `|g_bmsPackPowerWatts/1000| > 5` (kW), wird ein RTC-Zeitstempel in die EEPROM-Fault-Historie geschrieben, bevor der MCU-Reset ausgelöst wird. |
| `0x0800c0b8` | `0x0800c124` | `Voltage_Temp_Protection_Checker` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08012de8` | `0x08012e58` | `prvAddCurrentTaskToDelayedList` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08012e80` | `0x08012ef0` | `prvAddTaskToReadyList` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08012f38` | `0x08012fa8` | `prvCheckForValidListAndQueue` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08012f9c` | `0x0801300c` | `prvCheckTasksWaitingTermination` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801a9a8` | `0x0801aae0` | `prvCreateTimerTask` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080130dc` | `0x0801314c` | `prvHeapInit` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080131a8` | `0x08013218` | `prvInitialiseNewTask` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080132bc` | `0x0801332c` | `prvInitialiseTaskLists` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08013a10` | `0x08013a80` | `pvPortMalloc` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08013c28` | `0x08013c98` | `pxPortInitialiseStack` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08013e10` | `0x08013e80` | `shellAdd` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08018d98` | `0x08018ed0` | `vPortValidateInterruptPriority` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08019158` | `0x08019290` | `vTaskPriorityDisinheritAfterTimeout` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x080192dc` | `0x08019414` | `vTaskStartScheduler` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08019408` | `0x08019540` | `vTaskSwitchContext` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08019530` | `0x08019668` | `xPortStartScheduler` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801999c` | `0x08019ad4` | `xQueueGenericSend` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08019bd4` | `0x08019d0c` | `xQueueGenericSendFromISR` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08019d4c` | `0x08019e84` | `xQueueReceive` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x08019f38` | `0x0801a070` | `xQueueSemaphoreTake` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801a350` | `0x0801a488` | `xTaskIncrementTick` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801a518` | `0x0801a650` | `xTaskPriorityDisinherit` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801a674` | `0x0801a7ac` | `xTaskPriorityInherit` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801a73c` | `0x0801a874` | `xTaskRemoveFromEventList` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |
| `0x0801a838` | `0x0801a970` | `xTaskResumeAll` | kosmetisch | Nur Adress-/Registernamen-Unterschiede (Relozierung infolge Codewachstums bzw. Decompiler-Registerallokation), keine Verhaltensänderung. |

## 2. Details der realen Änderungen

### 2.1 Entladelimit-Prozent — neue Propagation und reparierte Rotationslogik

`g_dischargeLimitPct` (RAM `0x200028e0`, ushort) existiert bereits in 117.7 und wird dort schon von
`APP_LoadOrInitDischargeLimitConfig`, `PackSelect_MosMismatch_Fallback`, `PackSelect_Next_
AboveSocThreshold` und `CAN_TX_PF2_Capacity` gelesen bzw. per dediziertem CAN-Kommando `0x100301aa`
gesetzt und in Flash persistiert (`APP_WriteFlash_DischargeLimitPct`). Drei Stellen ändern sich in 118:

**a) `CAN_RX_Handler`** — im Zweig für den periodischen Master-Enable-/Heartbeat-Frame (Bedingung
`iVar2==0xfd00`) kommt eine zusätzliche Zeile hinzu:

```c
// 118, nach g_lastCanCmd04PayloadFlags = (uchar)frame[3];
g_dischargeLimitPct = (ushort)*(byte *)((int)frame + 0xd);
g_packOnlineWatchdogTimerMs = 0;
```

Das Entladelimit wird damit jetzt **zusätzlich** bei jedem Heartbeat aus Byte `frame+0xd` übernommen
(reines RAM-Update, kein `APP_WriteFlash_DischargeLimitPct`-Aufruf wie beim dedizierten Kommando
`0x100301aa`). 117.7 kennt diese Zeile nicht — dort ändert sich `g_dischargeLimitPct` nur über das
dedizierte CAN-Kommando oder die RS485-/CLI-Config-Pfade.

**b) `PackSelect_DischargeStepThreshold`** — die Berechnungsformel wurde komplett ersetzt:

```c
// 117.7: feste Stufentabelle, unabhängig von g_dischargeLimitPct
uVar2 = (&g_pack0Soc_x10)[pack_idx * 0x30] / 10 & 0xff;
uVar1 = (((int)(uVar2 - 10) / 10) * 5 & 0x7fU) * 2;
if (uVar1 < 10) { if (uVar2 < 0xb) uVar1 = 9; else if (10 < uVar2) uVar1 = 10; }

// 118: Schwelle aus dem konfigurierbaren Entladelimit abgeleitet
limit_pct = 100 - g_dischargeLimitPct & 0xff;
pack_soc_pct = (&g_pack0Soc_x10)[pack_idx * 0x30] / 10 & 0xff;
result = pack_soc_pct - 10 & 0xff;
if (result < limit_pct) {
  if (limit_pct < pack_soc_pct) result = limit_pct;
  else result = limit_pct - 1 & 0xff;
}
```

In 117.7 ignorierte diese Funktion `g_dischargeLimitPct` also komplett und benutzte eine fest
verdrahtete 10-%-Stufenlogik. In 118 folgt die Rotationsschwelle direkt dem konfigurierten Entladelimit
— eine reale Verhaltensänderung, keine reine Umbenennung.

**c) `PackSelect_Discharge_StepRotate`** — neuer RAM-Cache `g_dischargeStepRotateLimitCache`
(`0x20002a58`, 2 B, s. §2.5):

```c
if (g_dischargeStepRotateLimitCache == 0) g_dischargeStepRotateLimitCache = g_dischargeLimitPct;
if ((<SOC/Strom-Bedingung>) && (g_dischargeStepRotateLimitCache == g_dischargeLimitPct)) {
  g_dischargeStepRotateTimer = 0;           // "bleib aktiv" nur wenn Limit unverändert
} else {
  g_dischargeStepRotateTimer++;             // sonst zählt der Timer weiter Richtung Neuberechnung
  if (0x32 < g_dischargeStepRotateTimer) {
    g_dischargeStepRotateLimitCache = g_dischargeLimitPct;   // Cache aktualisieren
    ...
  }
}
```

In 117.7 wurde der Rotations-Timer bei erfüllter SOC-/Strom-Bedingung bedingungslos auf 0
zurückgesetzt — ein zur Laufzeit geändertes `g_dischargeLimitPct` hätte die Schwelle
(`PackSelect_DischargeStepThreshold`) dort ohnehin nicht beeinflusst, ist aber jetzt in 118 relevant
und wird durch diesen Cache-Vergleich innerhalb von höchstens 50 Ticks nachgezogen, statt beliebig
lange auf dem alten Wert zu verharren.

**Praktische Bedeutung:** In 117.7 hatte der Entladelimit-Prozentsatz keinen Einfluss auf die
Pack-Rotationsschwelle. In 118 wirkt er sich sowohl auf die Schwellenformel als auch — über den neuen
Heartbeat-Pfad — auf die Aktualisierungsfrequenz aus.

### 2.2 Shutdown-Timing: `PackMode_Shutdown_Handler`

```c
// 117.7 — State 0 (sofort beim Eintritt):
if (g_packModeShutdownState == '\0') {
  g_pendingActivationFlag = '\x01';
  g_pendingActivePackMask = '\0';
  ...
}
// 118 — dieselbe Zeile fehlt in State 0, taucht stattdessen am Ende von State 1 auf:
else if (g_packModeShutdownState == '\x01') {
  g_packModeShutdownTimer++;
  if (299 < g_packModeShutdownTimer) {
    g_packModeShutdownTimer = 0;
    g_pendingActivationFlag = '\x01';        // <- jetzt hier
    g_packModeShutdownState = '\x02';
  }
}
```

`g_pendingActivationFlag` wird in 118 erst nach den 300 Ticks Wartezeit (State 1) gesetzt statt sofort
beim Eintritt in die Shutdown-Sequenz (State 0). Dieser Fund zeigt, warum die reine Listing-Heuristik
nicht ausreicht: Der Listing-Diff zeigt hier ein `delete`+`insert` symmetrischer Instruktionsblöcke, was
oberflächlich wie eine reine Umsortierung aussieht — erst der C-Vergleich macht die tatsächliche
Verschiebung des Flag-Setzens um eine ganze Zustandsphase sichtbar.

### 2.3 SOC/OCV-Rekalibrierung: neue Funktion `SOC_OcvBoundsRecalibrateOnFlag`

Neu in 118, `0x08016634`, 146 B:

```c
void SOC_OcvBoundsRecalibrateOnFlag(int32_t *soc_state)
{
  if (g_socOcvRecalibrateRequest == '\x01') {
    if (<Temperatur außerhalb Knie-Band, s. Bedingung>) {
      NTC_ConvertAndUpdatePackTempExtremes(soc_state, 2);
    }
    g_socOcvRecalibrateRequest = '\0';
    RTC_Read_DateTime((int32_t *)g_socCalibTimestamp);
    I2C_EEPROM_Write_WithMutex(g_socCalibTimestamp, 0xfc02, 10);
  }
  else if (g_socOcvRecalibrateRequest == '\x02') {
    /* gleiche Struktur ohne die Knie-Band-Prüfung */
  }
}
```

Aufgerufen als erste Aktion in `SOC_AlgorithmOrchestrator`, sobald `g_socStateFlags` das
Initialisierungs-Bit anzeigt. Das konsumierte Flag `g_socOcvRecalibrateRequest` (RAM `0x200029ca`)
existiert bereits identisch in 117.7 — dort hat es aber **keinen Konsumenten**: keine Funktion in
117.7 liest oder schreibt es. In 118 wird es ausschließlich von dieser neuen Funktion gelesen; ein
**Schreiber** (Setter) dieses Flags ließ sich in keiner der beiden Firmware-Versionen finden (Offener
Punkt, §4). Die Namensermittlung für 118 kommt unabhängig zum selben Befund: Das Flag wird im
gesamten Image nirgends auf 1 oder 2 gesetzt (nur auf 0 gelöscht) — `SOC_OcvBoundsRecalibrateOnFlag`
ist damit **praktisch unerreichbarer Code**. Die tatsächlich wirksame OCV-Rekalibrierung läuft über
den bereits in 117.7 vorhandenen 1-Stunden-Leerlauf-Timer weiter unten in `SOC_AlgorithmOrchestrator`
(`g_socIdleTimeAccumMs`-Überlauf, unverändert in beiden Versionen).

Zusätzlich fügt 118 in `SOC_AlgorithmOrchestrator` selbst zwei neue RTC+EEPROM-Zeitstempel-Schreib-
vorgänge ein (`RTC_Read_DateTime` → `I2C_EEPROM_Write_WithMutex(…, 0xfc02, 10)`): einmal beim ersten
Eintritt in den Leerlaufmodus (`mode==3`, vorher `g_socChargeDischargeMode!=3`), einmal beim Ablauf des
Leerlauf-Timers (`g_socIdleTimeAccumMs` überläuft nach ~3 600 000 ms). In 117.7 wurden an diesen beiden
Stellen nur SOH/Zykluszähler aktualisiert, ohne Zeitstempel-Log.

### 2.4 Neues Fehler-Zeitstempel-Log beim Notreset

`System_Reset_And_Halt_WithCurrentCheck` (`0x08004998` in beiden Versionen, 117.7: 136 B, 118: 188 B):

```c
// 118, zusätzlich vor HAL_IWDG_Refresh():
if ((5 < (short)(g_bmsPackPowerWatts / 1000)) || ((short)(g_bmsPackPowerWatts / 1000) < -5)) {
  RTC_Read_DateTime((int32_t *)&g_rtcDateTimeBuf);
  I2C_EEPROM_Write_WithMutex((uint8_t *)&g_rtcDateTimeBuf, 0xfc02, 10);
}
```

Lag die zuletzt gemessene Pack-Leistung beim Reset außerhalb von ±5 kW, wird jetzt vor dem eigentlichen
MCU-Reset (SCB→AIRCR) ein RTC-Zeitstempel in dieselbe EEPROM-Fault-Historie geschrieben wie in §2.3.
117.7 kennt diese Bedingung nicht und springt direkt in die Watchdog-/Reset-Sequenz.

*Hinweis zur Dekompilat-Beschreibung:* Der automatisch erzeugte Funktionskopf-Kommentar in
`Dekompilate/VNSD_BMS_1177/08004998_System_Reset_And_Halt_WithCurrentCheck.c` beschreibt bereits das
118-Verhalten ("… loggt einen RTC-gestempelten Fehlerdatensatz, falls der zuletzt gesehene Stromwert
außerhalb des Bereichs liegt …"), obwohl der 117.7-Code diese Prüfung **nicht** enthält. Das ist ein
Namens-/Beschreibungsartefakt des Export-/Benennungslaufs, keine Aussage über den tatsächlichen
117.7-Code — maßgeblich ist ausschließlich der oben zitierte C-Code.

### 2.5 RAM-Layout-Folge: neuer Global `g_dischargeStepRotateLimitCache`

118 fügt den 2-B-Global `g_dischargeStepRotateLimitCache` bei `0x20002a58` ein (zwischen
`g_dischargeStepRotateTimer` und `g_socThresholdSelectFlag`). Er passt exakt in eine Ausrichtungslücke,
die in 117.7 an dieser Stelle bereits vorhanden war (`g_packRecoveryChargeTimerB` endet in beiden
Versionen erst bei `0x20002a6c`) — der Gesamtblock wächst dadurch **nicht**, aber sechs nachfolgende
Globals verschieben sich in 118 um +2 B:

| Global | 117.7 | 118 |
|---|---|---|
| `g_socThresholdSelectFlag` | `0x20002a58` | `0x20002a5a` |
| `g_mode1FullTimerA` | `0x20002a5a` | `0x20002a5c` |
| `g_mode1FullTimerB` | `0x20002a5c` | `0x20002a5e` |
| `g_mode2LowSocTimer` | `0x20002a5e` | `0x20002a60` |
| `g_mode3EmptyTimerA` | `0x20002a60` | `0x20002a62` |
| `g_mode3EmptyTimerB` | `0x20002a62` | `0x20002a64` |
| `g_packRecoveryChargeState` | `0x20002a64` | `0x20002a66` |
| `g_packRecoveryChargeTimerA` | `0x20002a66` | `0x20002a68` |
| `g_packRecoveryChargeTimerB` | `0x20002a68` | `0x20002a6a` |

Das erklärt die fünf als "Reine Folge …" markierten Zeilen in Tabelle §1
(`PackRecovery_LowCell_Charge_SM`, `PackSelect_Mode1_Full`, `PackSelect_Mode2_LowSoc`,
`PackSelect_Mode3_Empty`, `PackSelect_Next_AboveSocThreshold`): Diese Funktionen sind inhaltlich
identisch, referenzieren aber Felder aus diesem verschobenen Block.

### 2.6 Konfig-/Protokoll-Versionskennung `'u'` (117) → `'v'` (118)

Sechs Stellen heben denselben Versions-Byte-Wert synchron von `0x75` (ASCII `'u'`, 117 dezimal) auf
`0x76` (ASCII `'v'`, 118 dezimal) an:

| Funktion | Rolle |
|---|---|
| `BMS_Config_Params_Factory_Reset` | Schreibt `g_cfgVersionTag` beim Werksreset. |
| `BMS_Config_Params_Load_Or_Default` | Vergleicht `g_cfgVersionTag` beim Boot (Guard) und schreibt ihn beim Reset. |
| `RS485_Dispatcher` | Versions-Antwortbyte im RS485-Antwortpuffer. |
| `RS485_Pack_Telemetry_Response` | Versionsbyte in der Pack-Telemetrie-Antwort (Offset `0x48`). |

Ein Byte-Mismatch beim Boot löst in `BMS_Config_Params_Load_Or_Default` einen Factory-Reset des
Parameterblocks aus — ein Gerät, das von 117.7 auf 118 aktualisiert wird, bekommt damit automatisch
einen Parameter-Reset auf Werkswerte (unabhängig von individuell über RS485/CLI gesetzten
Protection-Grenzwerten). Separat davon meldet `PerPack_Struct_Builder` (§0, Beta-Hinweis) den
Firmware-Versionswert `g_pack0FwVersionTag` mit einer eigenen, inkonsistenten Zählweise
(`0x499`=1177 in 117.7 vs. `0x76`=118 in 118) — zwei unterschiedliche Versionskonventionen für
zwei unterschiedliche Zwecke (Config-Kompatibilität vs. gemeldete FW-Version).

## 3. Was sich NICHT geändert hat

- KA495xx-AFE-Treiber (SPI, CRC-8 `KA495XX_Crc8`, Register-Tabellen-Verifikation) — nur Adressen.
- Ladestrom-/Spannungsgrenzen (`Max_Charge_Current_Calc`, `Max_Discharge_Current_Calc`,
  Temperatur×SOC-Derating-Matrix, 50-A-Deckel, 57,6 V/48,0 V) — siehe auch
  `VNSD_Ladegrenzen_Ladeleistung_de.md`.
- CAN-Kommandosatz und Message-IDs (`CAN_RX_Command_Dispatcher`, `CAN_RxSlavePackTelemetry_Unpack`,
  `CAN_Pack_Activation_Config_Receive`) — nur Adressen/Variablennamen.
- OTA-/Bootloader-Handshake (`BMS_OTA_Flash_Writer_FinalizeTail`, `OTA_Set_Reboot_Params`,
  `HAL_Flash_Write_Config_Block`, Staging-Adresse `0x0801E000`, Tag-Prüfung `0x0802B800`).
- Shell-Kommandotabelle (36 Einträge, Nutzer `BLUE`) — die einzige geänderte CLI-Funktion
  (`CLI_ShellInit`) unterscheidet sich nur in lokalen Variablennamen.
- FreeRTOS-Taskliste: alle 10 Tasks mit identischen Stacks/Prioritäten/Handles
  (`RTOS_CreateAllTasks` byte-für-byte verglichen, nur die 10 Einstiegsadressen verschoben).
- Protection-Schwellen (`Cell_OVP_UVP_TempCompensated_Protection`, `Voltage_Temp_Protection_Checker`)
  — vollständiger C-Vergleich (nicht nur Listing-Diff) zeigt ausschließlich die eine global
  verschobene RAM-Adresse (`0x20004AD0`→`0x20004ADC`, s. §4), keine geänderten Schwellwerte.

## 4. Offene Punkte

- **Kein Schreiber für `g_socOcvRecalibrateRequest` gefunden.** Weder in 117.7 noch in 118 setzt eine
  der 710/711 dekompilierten Funktionen dieses Flag — es wird in 118 nur noch gelesen (§2.3). Möglich:
  externer Trigger (Debug/JTAG-Speicherschreibzugriff) oder eine Funktion außerhalb des BMS-Images
  (z. B. über RS485/CAN von Control/Micro, aber in keinem der 88 geänderten oder 622 identischen
  Funktionen sichtbar). Nicht abschließend geklärt.
- **RAM-Wachstum nur teilweise erklärt.** `.bss` wächst um 16 B (117.7: `0x9fcc` → 118: `0x9fdc`),
  der FreeRTOS-Heap wird um 8 B kleiner (`0x4FF8`→`0x4FF0`). Der neue 2-B-Global
  `g_dischargeStepRotateLimitCache` (§2.5) erklärt nur einen kleinen Teil dieser Verschiebung; die
  übrigen Bytes ließen sich nicht auf einen einzelnen neuen benannten Global zurückführen — vermutlich
  Heap_4-interne Ausrichtungseffekte (der Heap-Block direkt vor `ucHeap` verschiebt sich ebenfalls,
  s. Vollanalyse-Dokument §2). Für die Fragestellung dieses Diffs (reale vs. kosmetische
  Funktionsänderungen) ohne weitere Relevanz.
- **Globale Konstante `0x20004AD0`→`0x20004ADC` (14 Funktionen betroffen)**: Dieselbe RAM-Adresse
  verschiebt sich in 14 unabhängigen, sonst unveränderten Funktionen um exakt 0xC (12) Byte
  (`BMS_Measurements_Valid_Check`, `BMS_ProtectionAndCommTick`, `CAN_TX_PF2_Capacity`,
  `Cell_OVP_UVP_TempCompensated_Protection`, `OwnPack_MOSFET_Driver` + `_MasterVariant`,
  `PerPack_Struct_Builder`, `RS485_Pack_Telemetry_Response`, `SOC_AlgorithmOrchestrator`,
  `SOC_InitCapacityParams`, `SOC_ResetEstimatorToPercent`, `SOC_UpdateDisplayValueAndFullFlag`,
  `Voltage_Temp_Protection_Checker`). Das deutet auf ein einzelnes, um 12 B gewachsenes Datenobjekt vor
  dieser Adresse hin; welches genau, wurde in dieser Sitzung nicht identifiziert (kein Funktionsname
  in den 88 Änderungen deutet direkt darauf hin — vermutlich ein durch den Compiler platziertes,
  unbenanntes `.bss`-Objekt). Ohne Einfluss auf die Real/Kosmetisch-Einstufung dieser 14 Funktionen,
  da deren Codeverhalten identisch ist.
