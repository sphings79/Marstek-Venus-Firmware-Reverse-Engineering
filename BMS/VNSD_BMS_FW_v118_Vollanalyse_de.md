# BMS-FW v118 (Venus D) — Vollanalyse

> 🇬🇧 English version: [VNSD_BMS_FW_v118_Full_Analysis.md](VNSD_BMS_FW_v118_Full_Analysis.md)

**Stand:** 2026-09-30.

**Englisch:** `VNSD_BMS_FW_v118_Vollanalyse_de.md` (wird nach dieser Fassung erstellt) ·
**Vorgänger-Vollanalyse:** `VNSD_BMS_FW_v1177_Vollanalyse_de.md` (wird nach dieser Fassung erstellt) ·
**Diff 117.7 → 118:** `VNSD_BMS_Diff_v1177_v118_de.md` (wird nach dieser Fassung erstellt) ·
**Ladegrenzen (vertiefend):** `VNSD_Ladegrenzen_Ladeleistung_de.md` ·
**Dekompilat/Listing:** `Dekompilate/VNSD_BMS_118/` (lokal, nicht im Repo) (711 Funktionen, 778 RAM-Globals, `listing/` mit
Flash-Rohdisassemblat + `globals_and_registers.txt`) · Vergleichsexport: `Dekompilate/VNSD_BMS_1177/`.

## 0. Kurzfassung

- Image-Basis **0x08002800** (Bootloader darunter, Seite `0x08002000` = Boot/OTA-Handshake
  `g_bootConfigPage`), Image-Größe 0x1A000 Byte (106 496 B), MCU **Nations N32G45x** (Cortex-M4F),
  Keil ARMCC, FreeRTOS (`heap_4`, Heap `0x4FF0` B), letter-shell (36 Tabelleneinträge, Default-User
  `BLUE`), Kommunikation CAN + RS485, Zellüberwachung KA495xx (SPI, CRC-8).
- **711 von 711 Funktionen benannt**, alle Verifikationsgates bestanden (siehe §2.3). Gegenüber der
  alten Analyse (`VNSD_BMS_FW_Analyse_v118_de.md`, 550 Funktionen, falsche Basis `0x08000000`, MCU
  fälschlich AT32F403A) ist das ein vollständiger Neuaufbau — Details in §13.
- **Ladegrenze:** maximal 50,0 A pro Pack (Tabelle Temperatur × SoC), Ladeschluss 57,6 V, es lädt
  physisch immer nur ein Pack (`PackHandoff_StateMachine`/`OwnPack_MOSFET_Driver`). Kurzfassung in
  §8, vollständige Herleitung inkl. Abgleich mit Messwerten in `VNSD_Ladegrenzen_Ladeleistung_de.md`.
  Die berüchtigte „148-W-Plateau"-Beobachtung stammte **nicht** vom BMS, sondern von der
  Omnibattery-Integration, und wird hier nicht wiederholt.
- **Diff zu 117.7 (§0a, vollständige Details in `VNSD_BMS_Diff_v1177_v118_de.md`):** 88 von 711 Funktionen
  als „changed" markiert. Jedes der 88 Paare wurde Zeile für Zeile am C-Code geprüft (nicht nur am
  Listing-Diff): **11 sind real, 77 sind kosmetisch** (Decompiler-/Register-Rauschen bzw. reine
  Adressverschiebung durch Codewachstum). Zusammen mit der einen komplett neuen Funktion
  `SOC_OcvBoundsRecalibrateOnFlag` (`0x08016634`) ergeben sich **12 reale Änderungen** insgesamt. Die
  wichtigsten (vollständige Liste inkl. C-Code-Belegen im Diff-Dokument, §2):
  1. **Entladelimit-Prozent (`g_dischargeLimitPct`)** wird jetzt zusätzlich über den periodischen
     CAN-Heartbeat propagiert (`CAN_RX_Handler`), und die davon abhängige Pack-Rotationslogik wurde
     repariert/erweitert (`PackSelect_DischargeStepThreshold`, `PackSelect_Discharge_StepRotate`) — in
     117.7 hatte dieser Prozentsatz keinen Einfluss auf die Rotationsschwelle.
  2. **Neue OCV-Rekalibrierungsfunktion `SOC_OcvBoundsRecalibrateOnFlag`**, eingehängt in
     `SOC_AlgorithmOrchestrator`, plus zwei neue RTC/EEPROM-Zeitstempel-Log-Einträge beim
     Moduswechsel in den Leerlauf. Das von ihr konsumierte Flag `g_socOcvRecalibrateRequest` wird in
     keiner der 710/711 Funktionen je auf 1 oder 2 gesetzt — die Funktion ist nach aktuellem
     Kenntnisstand **totem Code**; der wirksame OCV-Rekalibrierungspfad bleibt der bereits in 117.7
     vorhandene 1-Stunden-Leerlauf-Timer.
  3. **Neues Fehler-Zeitstempel-Log beim Notreset** (`System_Reset_And_Halt_WithCurrentCheck`), wenn die
     zuletzt gemessene Pack-Leistung außerhalb ±5 kW lag.
  4. **Config-/Protokoll-Versionskennung** `g_cfgVersionTag` wechselt von `'u'` (117.7) auf `'v'` (118)
     an sechs Stellen synchron (Config-Guard, RS485-Version, Pack-Telemetrie). Praktische Folge: ein
     unter 117.7 gespeicherter Konfigurationsblock gilt unter 118 beim ersten Start als ungültig
     (Tag-Mismatch) und wird einmalig auf Werksdefaults zurückgesetzt und neu gespeichert.
  5. **Shutdown-Timing** (`PackMode_Shutdown_Handler`): `g_pendingActivationFlag` wird erst nach den
     300 Ticks Wartezeit statt sofort gesetzt.
  Alle übrigen CAN-/RS485-Opcodes, Ladeströme/-tabellen und Schutzschwellen-**Werte** sind zwischen
  117.7 und 118 identisch.

## 0a. Prüfliste

| Frage | Antwort | Beleg |
|---|---|---|
| CAN-Protokoll zu Micro/EMS geändert ggü. 117.7? | Nein (Opcodes/IDs identisch), **eine** Verhaltensänderung: `CAN_RX_Handler` übernimmt im Master-Heartbeat-Frame zusätzlich `g_dischargeLimitPct` aus Byte 0xD. | `l1177_118.json` (`CAN_RX_Handler` als „changed" markiert) + manueller Diff `VNSD_BMS_1177/080081dc_...` vs. `VNSD_BMS_118/08008240_...` |
| RS485-Protokoll geändert? | Opcodes/Framing (XOR-Checksumme, ~53 Befehle) identisch. Einzige Änderung: das eingebettete Versions-Byte in der Pack-Telemetrie-Antwort wechselt `'u'`→`'v'` (Spiegel des Config-Tags). | Diff `RS485_Pack_Telemetry_Response` |
| Lade-/Entladegrenzen geändert? | Die harten Grenzen sind unverändert: `Max_Charge_Current_Calc`/`Max_Discharge_Current_Calc` und die zugrundeliegenden Tabellen (`g_chargeCurrentDerateMatrix`, `g_dischargeCurrentDerateMatrix`, Achsen) sind byte-identisch. Geändert wurde nur, **wie** der bereits vorhandene konfigurierbare Entladelimit-Prozentsatz (`g_dischargeLimitPct`) in der Pack-Rotationslogik wirksam wird — `PackSelect_DischargeStepThreshold` hing in 117.7 gar nicht von diesem Global ab (s. Kurzfassung Punkt 1). | Diff beider Funktionen gegen 1177; Diff-Dokument §2.1 |
| Schutzschwellen geändert? | Nein (Zahlenwerte identisch: `Voltage_Temp_Protection_Checker`, `Cell_OVP_UVP_TempCompensated_Protection`, `BMS_Config_Params_Factory_Reset`/`Load_Or_Default` byte-identisch bis auf den Versions-Tag). Die intern schon länger bestehende Inkonsistenz zwischen den beiden Reset-Pfaden (Kurzschluss-Stufe-1-Timeout 20000/20000 vs. 5000/10000 ms) existiert in **beiden** Firmwareständen gleichermaßen, ist also keine 118-Neuerung — siehe §9. | Diff + `g/gout_02.json` Notiz |
| OTA weiterhin möglich (CAN/RS485, Bootloader-Handshake)? | Ja, unverändert. `BMS_OTA_Flash_Writer`/`_FinalizeTail`, Staging-Adresse `0x0801E000`, CRC16/Modbus, Tag-Prüfung `'BMS1'`, Handshake-Magics `0x11223344`/`0x55667788` sind byte-identisch zu 117.7. RS485-OTA-Pfad (`RS485_OTA_Flash_Handler`) ebenfalls unverändert vorhanden. | Diff `BMS_OTA_Flash_Writer_FinalizeTail`, §10 |
| Cloud-Bezug? | Entfällt (n/a) — das BMS hat keine WAN-/Cloud-Anbindung, nur CAN zu Control/Micro und RS485 zu den Slave-Packs. | — |

## 1. Binary-Fingerprint

| Datei | Version | Größe | SHA-256 |
|---|---|---|---|
| `118_bms_VNSD-0_20260119100535e43806957.bin` | 118 | 106 496 B (0x1A000) | `fdcefaec7a8c0cac8850b3abb53a87acff38dafb92fa0af6f4f2f191d9fc7589` |
| `1177_bms_VNSD-0_20251010135647565eb2036.bin` | 117.7 (Beta) | 106 496 B (0x1A000) | `69b18aa42831536f01546bbe707e36ee65b6faa84e6cf4e92f9fbfe237f67c77` |

(`shasum -a 256` gegen die Dateien unter `Marstek FW Archiv/firmwares/VNSD-0/BMS/118/` bzw. `/1177/`.)
Herstell-Changelog (`CHANGELOG.de.md` (lokal, nicht im Repo)) nennt für beide Stände nur „OTA-Test" (117.7 zusätzlich „7"),
keine funktionale Beschreibung.

## 2. MCU-Identifikation, Speicherkarte, Aufbereitungsstand

### 2.1 MCU: Nations N32G45x (Cortex-M4F)

- ADC1/ADC2-Register liegen auf `0x40020800`/`0x40020C00` — das ist die N32G45x-Speicherkarte, nicht
  die von AT32F403A (auf dem die alte Analyse basierte, s. §13).
- 96-Bit-Unique-ID wird bei `0x1FFFF7F0` gelesen (`HAL_ReadUniqueDeviceID`), dokumentierte
  N32-UID-Adresse; alternativer Lesepfad `0x1FFFF270`, falls die Hauptadresse gelöscht (`0xFFFFFFFF`)
  ist.
- 102 Einträge in der Vektortabelle (86 IRQs + 16 Systemvektoren).
- Registernamen stammen aus dem offiziellen N32G457-SVD (NSING DFP 1.0.7). Ein undokumentiertes
  Register `0x40001800` wird von `PWR_ApplyCtrl3TrimFromReg40001800` gelesen (Trim-Quelle, nicht in
  der SVD enthalten).

### 2.2 Speicherkarte

| Bereich | Inhalt |
|---|---|
| `0x08002800` – `0x0801FFFF` | Firmware-Image (0x1A000 B), Vektortabelle ab `0x08002800` |
| `0x08002000` (darunter) | Bootloader/Boot-Config-Seite (`g_bootConfigPage`, Handshake s. §10) |
| `0x08010000` | Versions-Tag `'BMS1'` |
| `0x0801E000` | OTA-Staging-Kopie (neues Image wird hierhin geschrieben, Tag-Prüfung bei `0x0802B800`) |
| `0x20000000` ff. | SRAM (144 KB); `.data` 0x3C4 B LZ-komprimiert im Image (`Load$$RW_DATA$$Base`
  am Image-Ende) → entpackt nach `0x20002800`; `.bss` bis `0x2000CB9F` (118) |
| `0x2000CBA0` | Initial-Stackpointer (aus Vektortabelle) |
| I2C-EEPROM `0x0000`–`0x0FFF` (Beispiel) | Konfigurationsblock, Fehlersperre, SOC-Kalibrierung,
  Ereignislog — siehe §4.4 |

.data/.bss zwischen 117.7 und 118: 117.7 hat 16 B kleineres `.bss` und Heap `0x4FF8` B (statt
`0x4FF0` B in 118); `__scatterload_copy` liegt in 117.7 bei `0x0800FFEC`, in 118 bei `0x08010D66`;
zwei kleine Funktionen sind entsprechend verschoben (117.7 `0x0800FF8C`/`0x0800FF98` ↔ 118
`0x08010BCC`/`0x08010BD8`). Das erklärt, warum praktisch jede spätere Funktionsadresse zwischen
beiden Ständen um ein paar Bytes verschoben ist, obwohl der Code identisch ist.

### 2.3 Aufbereitungsstand in Ghidra

- **711 von 711 Funktionen benannt** (0 verbleibende `FUN_*`), 4236 Ghidra-Symbole, 0 gescheiterte
  Dekompilationen (`_index.md`, Stand 2026-09-30 19:21).
- **778 RAM-Globals** durchbenannt und kommentiert (`listing/globals_and_registers.txt`, inkl.
  N32G45x-Peripherieregistern).
- Alle Verifikationsgates (Namensvergabe ohne verbleibende Platzhalter, Cross-Referenz-Konsistenz
  zwischen Funktions- und Globals-Export, Abgleich der 117.7↔118-Funktionspaarung) sind laut
  Projektstand bestanden; siehe `l1177_118.json` (88 changed / 1 nur in 118 / 622 identisch / 0 nur
  in 117.7 — vollständige Paarung, kein verwaistes Funktionspaar).
- Einschränkung aus den Namensbegründungen (`out_*.json`/`gout_*.json`): einzelne Funktionsnamen
  tragen weiterhin ein historisches, nachweislich falsches `FATFS_`-Präfix aus einem älteren
  Benennungslauf (z. B. `CLI_ProcessCommandLine`, `CLI_ResolveVariableReference`,
  `CLI_CheckCommandPermission` — alle eindeutig letter-shell-Funktionen, nicht FatFs) und wurden im
  aktuellen Export bereits korrigiert; siehe §11 für die verbleibenden Zweifelsfälle.

## 3. FreeRTOS-Tasks

`RTOS_CreateAllTasks` (`0x0800e7bc`) legt alle 10 Anwendungs-Tasks an:

| Task-Name | Entry | Stack (Worte) | Priorität | TCB-Handle |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800ea31` | `0x80` | 5 | `0x20002b08` |
| `RTOS_vTaskKeyScan` | `0x0800ea17` | `0x80` | 9 | `0x20002b0c` |
| `RTOS_vTaskKA495XX` | `0x0800ea09` | `0x80` | 8 | `g_ka495xxTaskHandle` |
| `RTOS_vTaskCanRec` | `0x0800e9b1` | `0x180` | 4 | `0x20002b14` |
| `RTOS_vTaskCanSend` | `0x0800e9c1` | `0x80` | 3 | `0x20002b18` |
| `RTOS_vTaskRS485Comm` | `0x0800ea67` | `0x180` | 1 | `0x20002b1c` |
| `RTOS_vTaskSocAlgorithm` | `0x0800ea75` | `0x100` | 7 | `0x20002b20` |
| `RTOS_vTaskProtectCheck` | `0x0800ea51` | `0x100` | 6 | `0x20002b24` |
| `RTOS_vTaskControlLogic` | `0x0800e9cb` | `0x80` | 2 | `0x20002b28` |
| `RTOS_vTaskDataSave` | `0x0800e9e5` | `0x80` | 0 | `0x20002b2c` |

Höchste Priorität hat `RTOS_vTaskKeyScan` (9), gefolgt von `RTOS_vTaskKA495XX` (8, Zellüberwachung)
und `RTOS_vTaskSocAlgorithm` (7). RS485-Kommunikation und Datensicherung laufen mit der niedrigsten
Priorität (1 bzw. 0). Der Shell-Debug-Befehl `task_size` (`RTOS_PrintTaskStackHighWaterMarks`,
`0x080156bc`) druckt für alle 10 Tasks den Stack-High-Water-Mark.

## 4. Schnittstellen

### 4.1 CAN

Adressierung: `CAN_Build_Arbitration_ID` (`0x08003570`) packt eine 29-Bit-ID aus Nachrichtengruppe,
Subtyp/Pack-Typ und Pack-Id im Venus-internen Board-zu-Board-Layout zusammen; `g_pack0SlotId`
(eigener Pack-Rotationsslot, `0xFF` = unangemeldet/offline, `1` = aktiver Master) fließt mit ein,
sofern < 0xF, sonst wird 0xF (Broadcast) verwendet.

**Eingehend** (`CAN_RX_Handler`, `0x08008240`, Top-Level-Dispatcher):

| CAN-ID (bzw. ID-Familie) | Bedeutung |
|---|---|
| `0x180102aa` | Anfrage-Broadcast → `CAN_TX_Orchestrator(0xff)` (alle Packs senden) |
| `0x180101aa` | pack-adressierte Statusanfrage → `CAN_TX_Orchestrator(pack_id)` |
| `0x100101aa` | `CAN_RX_System_Command_Handler`: Byte 1=Adress-Anfrage, 2=Factory-Reset, 3=Relais-Kommando, 8=Fehlersperre löschen |
| `0x100201aa` | `CAN_CMD_02_Handler` |
| `0x100301aa` | direktes Setzen von `g_dischargeLimitPct` + Flash-Schreiben |
| `0x100401aa` | `CAN_CMD_04_Handler` |
| `0x100501aa` | CAN-RX-Neustart-Flag (Byte 1 == 1) |
| Familie `…+0x100`/`0x200`/`0x300`/`0x400` (Offset-Dekodierung) | diverse Statusanfragen |
| Familie `…+0xfd00` (Master-Heartbeat/Freigabe) | setzt `g_masterEnableReceivedFlag`, `g_lastCanCmd04PayloadFlags`, **seit v118 zusätzlich `g_dischargeLimitPct`** (Byte 0xD, s. §0a), `g_packOnlineWatchdogTimerMs=0`, dann `CAN_TX_Orchestrator(0xff)` |

Zweite Befehlsfamilie mit `(id>>0xc)==3` (`CAN_RX_Command_Dispatcher`, `0x0800f128`): Opcode im
niedrigsten Byte der ID, adressiert an eigenen Pack-Slot, Broadcast (0xF) oder 0:

| Opcode | Aktion |
|---|---|
| `0x01` | `CAN_TX_Send_Status_Response` |
| `0x02` | `CAN_Handle_Pack_Mode_Command` |
| `0x03` | Pack-Fault-Mode-Adressvergabe/-Bestätigung |
| `0x04` | `CAN_TX_BMS_Status_Reporter` (11 Telemetrie-Frames) |
| `0x05` | `APP_Factory_Reset_And_Reboot` |
| `0x0a` | `CAN_Set_Notify_Flag_And_Send` |
| `0xc4` | Pending-Activation-Flag setzen (nur wenn Sub-Byte-Nibble 0) |
| `0xce` | `BMS_OTA_Flash_Writer` (CAN-OTA, s. §10); setzt zusätzlich `g_otaInProgressFlag` |

**Ausgehend** (`CAN_TX_Orchestrator`, `0x08007b84`): sendet 4 feste „Protocol-Frame"-Nachrichten
einmalig, dann pro Pack einen 10-Nachrichten-Burst:

| Frame | Funktion | Inhalt |
|---|---|---|
| PF1 | `CAN_TX_PF1_PackMeasurements` (Sub-ID `0x20`) | Spannung, Strom + 2 weitere 16-Bit-Felder |
| PF2 | `CAN_TX_PF2_Capacity` | Kapazität |
| PF3 | `CAN_TX_PF3_ChargeDischargeLimits` | Ladestromlimit, Ladeschlussspannung (auf max. 60,9 V begrenzt) — s. §8 |
| PF4 | `CAN_TX_PF4_ProtectWarnings` | zwei 16-Bit-Warnwörter + globales Fault-Byte + Relais-Eviction-Byte |
| PerPack ×10 | `CAN_TX_PerPack_10Msgs` (`0x08008358`) | Übersicht (Spannung/Strom/SoC/Zyklen/Limits/Protect/Version, Gruppen `0x20`–`0x23`), 16 Zellspannungen in 4 Frames (`0x30`–`0x33`), Temperatur/MOS-Daten (`0x40`–`0x41`) |

`CAN_TX_BMS_Status_Reporter` (`0x0800ecd8`, 11 Frames: SOC, Zyklenzahl, Zellspannungen/-differenzen,
Temperaturen, Ströme, Protect-Flags) wird gezielt bei Opcode `0x04` als Antwort auf eine
EMS/Control-Anfrage gesendet, nicht im Broadcast-Burst.

**Pack-Aktivierung:** `CAN_CMD_Pack_Activation` (`0x0800df70`) und
`CAN_Pack_Activation_Config_Receive` (`0x0800e18c`) übernehmen die Übergabe einer 12-Byte-MCU-UID je
Slave-Pack aus drei aufeinanderfolgenden CAN-Frames sowie die Aktivierungs-Konfiguration.
`CAN_RxSlavePackTelemetry_Unpack` (`0x0800dbf4`, 854 B) entpackt die vollständige Pack-Telemetrie
eines Slave-Packs inklusive MOS-Statusbits.

### 4.2 RS485 zu Slave-Packs

Framing: XOR-Checksumme über den Frame (`LIB_XOR_Checksum`), Byte 2 = Opcode. `RS485_Dispatcher`
(`0x08010004`, 1616 B) ist der zentrale Dispatcher mit **~53 Opcodes** (`0x00`–`0x34`, nicht alle
belegt); ausgewählte Opcodes:

| Opcode | Bedeutung |
|---|---|
| `0x00` | Statuswort (Relais-GPIOs, Pack-Statusflags, KA495xx-Fault-Wort) |
| `0x01` | Boot-Marker `0x41` |
| `0x02` | Software-Reset (SCB-AIRCR) |
| `0x05` | Balancing-Vorbedingung + Magic-Byte-Prüfung → `g_rs485DispatchPendingFlag` |
| `0x0e` | `RS485_OTA_Flash_Handler` (RS485-OTA-Passthrough, s. §10) |
| `0x11`/`0x12` | Parametergruppe lesen/schreiben (Register-Gruppen-IDs, s. u.) |
| `0x18`–`0x1b` | Pack-Strom, Zellspannungen + Balance-Maske, NTC-Temperaturen, SOC |
| `0x1e`–`0x21` | Nutzer-Flag / Tail-Register lesen/schreiben inkl. Flash-Speicherung |
| `0x24` | SOC-Override (`SOC_ResetEstimatorToPercent`) |
| `0x29`/`0x2a`→intern | Inter-Pack-Struct-Sync (Pack-Slot-Datensätze, je 0x60 B) |
| `0x2c`/`0x2d` | RTC lesen/schreiben |
| `0x2e`/`0x30` | Pack-Telemetrie-Antwort an Host (`RS485_Pack_Telemetry_Response`, s. u.) |
| `0x2a` | `System_Reset_And_Halt_WithCurrentCheck` |
| `0x31` | Checkout-Modus/Kanalumschaltung |
| `0x34` | Relais-Puls (GPIOB Bit `0x1000`) |

**Parametergruppen** (`RS485_Register_Read_Handler`/`_Write_Handler`, Gruppen-ID = 16-Bit-Wort):

| Gruppen-ID | Inhalt |
|---|---|
| `0x0001` | Lade-NTC OV/UV Trip1/2 + Release + Timeouts |
| `0x0002` | Entlade-NTC OV/UV Trip1/2 + Release + Timeouts |
| `0x0004` | MOS-Temperatur B Hi/Lo Trip1/2 + Release + Timeouts |
| `0x0008` | MOS-Temperatur A Hi Trip1/2 + Release + Timeout |
| `0x0010` | Zell-UVP Trip1/2a/2b + Release + Timeouts |
| `0x0040` | Zell-OVP Trip1/2a/2b + Release + Timeouts |
| `0x0080` | 1 Wort, Bedeutung nicht abschließend geklärt |
| `0x0100` | Entlade-OC/SC Trip1/2 + Timeouts + Reserve |
| `0x0200` | Lade-OC/SC Trip1/2 + Timeouts + Reserve |
| `0x0400` | 3 weitere, nicht vollständig benannte Register |

`RS485_Pack_Telemetry_Response` (`0x08012414`) schreibt an Offset `0x48` das Konfig-Versions-Tag-Byte
(117.7: `'u'`, 118: `'v'` — s. §0a) mit in den Antwortpuffer. `RS485_PrechargeRelaySequence`
(`0x0800d6dc`) fährt eine 4-Stufen-Relaissequenz (Anzug/Abfall, GPIOB `0x1000`) mit
Quittungsprüfung und Retry.

### 4.3 Debug-Shell (letter-shell)

`CLI_ShellInit` (`0x080147bc`) initialisiert die Instanz mit 5-History-Slots und loggt per Default
als Nutzer **`BLUE`** ein (`shellCommandTable`-Eintrag 0, Passwort-Attribut `0x0800`, Beschreibung
„default user"). Die Kommandotabelle (`0x0801c394`, 36 Einträge × 16 B) enthält danach 10
Tasten-Bindungen (Pfeiltasten, Tab, Backspace, Delete, Enter) und 25 echte Kommandos:

| Kommando | Zielfunktion | Beschreibung |
|---|---|---|
| `help` | `CLI_Cmd_Var` (`0x08014654`) | „show command info\r\nhelp [cmd]" |
| `clear` | `CLI_NotifyPendingShell` (`0x08013f28`) | löscht die Konsole (ANSI `\x1b[2K\r`) |
| `bat_data` | `CLI_Cmd_PrintAllPackStatus` (`0x080151d0`) | Status aller Slave-Packs (Adresse, online, Version, Spannung, Strom, SOC, Zyklen, MOS, NTC) |
| `set_soc` | `SOC_Cmd_ForceSocOverride` (`0x0801561c`) | SOC-Override: 0=lösen, 1–99=direkt setzen, 100=Voll-Latch |
| `set_dod` | `PackSelect_SetSocFloorMargin` (`0x08015234`) | SOC-Margin für Pack-Auswahl-Untergrenze, spiegelt in ein AFE-Konfigregister |
| `checkout_rs485` | `RS485_SetDebugCheckoutFlag` (`0x080151c4`) | Debug-Ausgabe über RS485 statt UART |
| `force` | `Protect_PrintUnderVoltageStatus` (`0x08015274`) | UVP-Debugstatus (Testindex, UV-Flag, Min-Spannung, MOS-Kommandos) |
| `set_work_pack` | `PackSelect_SetTargetPackIndex` (`0x08015608`) | Ziel-Pack-Index für Pack-Auswahl setzen |
| `state_printf` | `PackSelect_PrintCurrentPackStatus` (`0x08015630`) | aktiver Pack-Index + Lade-/Entlade-MOS-Kommandos |
| `sox_data` | `SOC_PrintDebugSummary` (`0x08015628`) | SOC/SOH-Debug-Summary |
| `wake` | `PackSelect_TriggerWakePulse` (`0x08015614`) | Wake-Puls auf GPIOB Pin 3 |
| `inq_uid` | `Debug_TriggerSystemEventF` (`0x080153c8`) | löst System-Event `0xF` aus |
| `uid_printf` | `PackSelect_PrintPackUids` (`0x08015918`) | 12-Byte-UID aller 6 möglichen Slave-Packs |
| `parall` | `BMS_PrintBatterySummary` (`0x08015448`) | Spannung, Strom, Temp-Extrema, SOC, Protect-Wörter |
| `self_flag` | `BMS_SetEmsChgDsgFlag` (`0x08015248`) | EMS-Lade-/Entlade-Flag setzen |
| `reset` | `NVIC_SystemReset` (`0x080155b8`) | Cortex-M-Reset |
| `can_send` | `CLI_Cmd_SendTestCanFrame` (`0x080151d8`) | Test-CAN-Frame ID `0x112233` senden |
| `update_printf` | `BMS_PrintSystemStatusFlags` (`0x08015998`) | Comm-Prohibit, Upgrade/Update-Finish, Factory-Flag, EMS-Lade/Entladeflag |
| `event` | `PackSelect_SetRelayCycleState` (`0x08015254`) | Master-Relaisschaltzustand direkt setzen (0–4) |
| `prt` | `RS485_SetDebugCmdPendingFlag` (`0x080155ac`) | RS485-Debug-Kommando-Pending-Flag |
| `task_size` | `RTOS_PrintTaskStackHighWaterMarks` (`0x080156bc`) | Stack-High-Water-Mark aller 10 Tasks |
| `factory` | `KA495XX_SetSleepMode` (`0x08015260`) | AFE-Sleep-Modus umschalten (trotz Namens „factory") |
| `lock_set` | `Protect_SetErrorLock` (`0x080153d4`) | Fehlersperre setzen/lösen (AFE-Lock-Register + Relais-Reset) |
| `volt_diff` | `BMS_PrintChgConnAndVoltDiff` (`0x08015b14`) | Zellspannungsdifferenz + Ladeanschluss-GPIO |
| `set_flg` | `CAN_SetControlLinkFlag` (`0x080155fc`) | Flag „gültiges Kommando/Heartbeat von EMS über CAN erhalten" |

Die letzte Spalte kombiniert das im Binary hinterlegte Shell-Hilfetext-Byte mit dem Ghidra-Kommentar
der jeweiligen Zielfunktion. Auffällig: `factory`-Kommando ruft entgegen seinem Namen keine
Werksrückstellung auf, sondern schaltet den AFE-Sleep-Modus (Namens-/Funktions-Diskrepanz im
Original-Shell-Code, nicht in der Ghidra-Benennung).

### 4.4 I2C-EEPROM / Flash-Layout

- **Konfigurationsblock**: `I2C_EEPROM_Read_WithMutex(600, 0xd4, &g_cfgMagicByte)` liest 0xD4 Byte ab
  EEPROM-Adresse 600; Gültigkeit über `g_cfgMagicByte=='b'` (implizit, wird immer neu gesetzt),
  `g_cfgVersionTag` (`'v'` in 118, `'u'` in 117.7) und eine Einerkomplement-Prüfsumme
  (`LIB_Checksum_Ones_Complement`, 0xD0 Byte) in `g_cfgChecksum`. Bei Mismatch: kompletter
  Reset auf ~80 Werksdefault-Felder (`BMS_Config_Params_Load_Or_Default`, identisch zu
  `BMS_Config_Params_Factory_Reset` bis auf die in §9 genannte Timeout-Inkonsistenz) und
  `APP_Config_Checksum_And_Save`.
- **Fehlersperre (Error-Lock)**: `APP_SetErrorLockFlag` schreibt Marker-Byte **`0x86`** (gesperrt)
  bzw. **`0xFF`** (gelöscht) an **Flash/EEPROM-Offset `0xFC00`** und spiegelt in
  `g_errorLockActiveFlag`. `OwnPack_MOSFET_Driver` blockiert beide Lade-/Entlade-Pfade, solange die
  Sperre aktiv ist.
- **SOC-Kalibrierungs-Zeitstempel**: `SOC_OcvBoundsRecalibrateOnFlag` (neu in 118, s. §0a) schreibt
  einen RTC-Zeitstempel (10 B) direkt anschließend an **`0xFC02`**.
- **SOC-Kalibrierungsdaten**: `SOC_SaveCalibrationToFlash` schreibt Max-/Min-/Real-/Show-SOC (je
  x10000) und Gesamtkapazität (0x14 B) sowie einen komplementären „gelöscht"-Datensatz
  (`g_flashParamBufFF`, 20×`0xFF`) an zwei rotierende 20-Byte-Slots ab `0x400`
  (`APP_EventLogGetSlotAddrs`, 2 Slots, Ringindex 0/1).
- **„FatFs"-Log**: `FATFS_Init_Log_File_If_Needed` (`0x0800ff6c`) trägt trotz Namens keine echte
  FAT-Dateisystem-Operation aus, sondern initialisiert bei Bedarf lediglich eine zweite
  `CLI_ShellInit`-Instanz mit 512-Byte-Puffer für den RS485-Checkout-Modus. Der Name ist
  wahrscheinlich ein Überbleibsel aus einem früheren, nachweislich falschen Benennungslauf (mehrere
  andere `FATFS_*`-Funktionen in diesem Image wurden bereits als Fehlbenennungen erkannt und auf
  ihre tatsächliche Domäne — letter-shell bzw. FreeRTOS — korrigiert, s. §2.3/§11). Eine echte
  FatFs-Anbindung (SD-Karte o. ä.) ist in den 711 Funktionen nicht nachweisbar.

## 5. Zellüberwachung KA495xx

- SPI-Anbindung (`INIT_KA495XX_SPI_GPIO`, `KA495XX_SPI_Read/Write_Register(s)`), Rahmenschutz per
  **CRC-8** (Polynom `0xD5`, Startwert `0xD5`, `g_ka495xxCrc8Table`, `KA495XX_Crc8` `0x08016b94`).
  16 Zellen, 5 NTC-Kanäle (`KA495XX_ReadTemperatureChannels`).
- **Geschriebene Schutz-/Konfigurationsregister** (`KA495XX_WriteProtectionRegs`, `0x0801695a`, über
  `KA495XX_SpiWriteMaskedRegister`, jeweils volle 16-Bit-Maske):
  - Register `0x0B`: Zellspannungs-Schutzkonfiguration (`KA495XX_EncodeCellVoltageProtectionConfig`)
  - Register `0x0A`: Überstrom-/Kurzschluss-Konfiguration (`KA495XX_EncodeOcscConfig`)
  - Register `0x13`: OVP/UVP/OCD-Schwellen (`KA495XX_EncodeOvpUvpOcdThresholds`)
  - Register `0x14`: Sleep-Timing-Konfiguration (`KA495XX_EncodeSleepTimingConfig`)
  - Schreiben erfolgt nur, wenn `KA495XX_CellBalanceChannelConfig` zuvor eine gültige
    Kanal-Konfigurationstabelle bestätigt.
- **Balancing-Entscheidung** (`KA495XX_CellBalanceDecision`, `0x08017498`): Balancing wird komplett
  deaktiviert, solange Zell-Spread `<30 mV` (`0x1e`) oder Packstrom `<1,001 A` (`0x3e9`); pro Zelle
  Einschaltschwelle **3,501 V** (`0xdad`) sofern nicht `<30 mV` unter der Maximalzelle, Ausschalt-
  schwelle **3,500 V** (`0xdac`) bzw. `<20 mV` unter Max, jeweils mit Debounce
  (`Protect_Debounce_TripCounter`, 1000 ms Timeout). Ergebnis-Maske geht in
  `KA495XX_BuildBalancingBitmask`.
- **Fault-/Statuswort**: `KA495XX_UpdateFaultStatusWord` konsolidiert AFE-Fehlerbits in
  `g_ka495xxFaultStatusWordRaw` (auch über RS485-Opcode `0x00` abfragbar, s. §4.2).
- Skalierung: Strom-/Leistungswerte werden mit empirischen Korrekturfaktoren **0,88** (Strom) bzw.
  **0,92** (Leistung) aus dem rohen AFE-ADC-Wert berechnet (`SOC_UpdatePowerAndCoulombCounters`,
  `0x08017c9c`) — vermutlich Shunt-/Verstärkungs-Korrekturen, nicht weiter dokumentiert.

## 6. Pack-Management

- **Rollen**: `g_pack0SlotId` = `0xFF` unangemeldet/offline, `1` = aktiver Master-Anspruch; Werte
  `2..n` sind Slave-Slots. Der Master fährt die Pack-Auswahl- und Handoff-State-Machines, Slaves
  antworten auf CAN-/RS485-Anfragen und liefern Telemetrie.
- **PackSelect-Familie** (`PackSelect_ModeDispatch`, `_Mode1_Full`, `_Mode2_LowSoc`, `_Mode3_Empty`,
  `_Mode3_TemperatureTimeout`, `_Discharge_StepRotate`, `_Charge_StepRotate`,
  `_Next_AboveSocThreshold`, `_HighestSoc_Online`, `_LowestSoc_Online`, `_Startup_Closest50` u. a.)
  wählt anhand SOC-Modus (`PackSelect_ModeBySoc`), Lade-/Entlade-Stufenschwellen
  (`PackSelect_ChargeStepTarget`/`_DischargeStepThreshold`) und Online-Status das jeweils nächste
  Pack für Rotation aus; `PackSelect_Startup_Closest50` sucht beim Hochlauf das Pack, dessen SOC am
  nächsten an 50 % liegt (`CAN_Find_Closest_Pack_To_50pct`).
- **Handoff**: `PackHandoff_StateMachine` (`0x08006740`) übergibt die Master-/Ladeberechtigung
  schrittweise von `active_pack` zu `target_pack`: altes Entlade-Enable löschen → warten, bis das
  alte Pack-MOS tatsächlich aus ist → neues Entlade-Enable setzen → warten, bis **genau ein** Pack
  ein aktives MOS hat (`LIB_Popcount`-Prüfung) → Lade-Enable rampen, solange MOS-sicher (SOC-Schwelle
  98,6 % / `0x62`) → Übergabe committen und Rotationsschwellen neu berechnen
  (`PackSelect_ModeBySoc`/`_ChargeStepTarget`/`_DischargeStepThreshold`).
- **MOSFET-Steuerung**: `OwnPack_MOSFET_Driver` (`0x08006bc4`) und die Master-Variante
  `OwnPack_MOSFET_Driver_MasterVariant` (`0x08007284`) schalten Lade-/Entlade-MOS/Relais nur, wenn
  eine lange Liste von Schutzbits (Kurzschluss, OV/UV, Überstrom, Temperatur, Precharge-Status,
  Kommunikations-Watchdog, `g_errorLockActiveFlag`) sauber ist; sonst werden beide Relais hart
  abgeschaltet. Die Prechargesequenz läuft über `g_prechargeSmState`, GPIOB `0x1000`, mit
  Rückmeldeprüfung.
- **Precharge über RS485**: `RS485_PrechargeRelaySequence` (§4.2) fährt eine unabhängige 4-Stufen-
  Sequenz mit Retry für den Fall, dass die Relaisansteuerung per RS485-Kommando ausgelöst wird.
- **Einzelpack-Laden erzwungen**: `PackHandoff_StateMachine` setzt die Lade-MOSFET-Maske nie auf
  mehr als ein Bit gleichzeitig; `OwnPack_MOSFET_Driver` schaltet ein Pack nur ein, wenn dessen Bit
  in dieser Maske gesetzt ist — physisch kann also nie mehr als ein Pack gleichzeitig laden
  (ausführlich in `VNSD_Ladegrenzen_Ladeleistung_de.md` §1.2).

## 7. SOC/SOH

- **OCV-Initialisierung**: `SOC_InitFromOcvLookup` setzt Real-/Show-/Max-/Min-/Tar-SOC beim ersten
  Vorliegen gültiger Spannungs-/Temperaturwerte aus `Full_OCV_Lookup` gegen `g_ocvSocCurveTable`
  (21 Stützstellen SOC % → OCV für 4 Temperaturzonen, mV).
- **Coulomb-Zählung**: `Coulomb_Counting` (`0x08011ba8`) integriert Strom × Zeit mit
  Float-Restakkumulator (`g_coulombCountResidual`) gegen Rundungsdrift; `Capacity_Factor_Calc`
  skaliert die kalibrierte Kapazität (`raw/10000/100`). Die vom Aufgabentext genannten Faktoren
  **0,88**/**0,92** stecken in der vorgelagerten Strom-/Leistungsberechnung
  (`SOC_UpdatePowerAndCoulombCounters`, §5), nicht direkt in `Coulomb_Counting` selbst.
  `SOC_CoulombCountTick` und `SOC_UpdatePowerAndCoulombCounters` pflegen zusätzlich 64-Bit-
  Energie-/Coulomb-Zähler (`g_bmsEnergyCounterLo/Hi`, `g_bmsCoulombCounterLo/Hi`).
- **Kalibrierung/Persistenz**: `SOC_SaveCalibrationToFlash`/`SOC_LoadCalibration_Float`/
  `SOC_EepromLoadCalibration` sichern Max-/Min-/Real-/Show-SOC (x10000) und Gesamtkapazität in den
  rotierenden EEPROM-Slots (§4.4). `SOC_OcvBoundsRecalibrateOnFlag` (neu in 118) erzwingt bei
  gesetztem Flag eine OCV-Nachkalibrierung außerhalb eines Spannungs-„Knee"-Bands und stempelt einen
  RTC-Zeitstempel.
- **Zyklenzahl**: `CycleCount_From_Throughput` rechnet akkumulierten Ladedurchsatz in
  Vollzyklen-Äquivalente um (`throughput/10/5.12`).
- **SOH**: `SOH_From_Throughput` → `SOH_LookupByCycleCount` interpoliert linear über eine
  7-Punkte-Tabelle (`g_ocpDelayVsCurrentTable`, trotz Namens hier als Zyklen→SOH-Kurve
  wiederverwendet: (0, 100 %), (500, 95 %), (1000, 92,5 %), (2000, 89,15 %), (6000, 80 %),
  (8000, 71 %), (20000, 0 %)); Ergebnis ist SOH ×1 000 000, auf 100 % gedeckelt. Die Tabellen-
  variable trägt einen aus einem anderen Kontext übernommenen Namen (OCP-Delay), ist an dieser
  Stelle aber nachweislich die Zyklen/SOH-Kurve — Namens-Altlast, kein Funktionsfehler.

## 8. Lade-/Entladegrenzen

Vollständige Herleitung inkl. Control/Micro-Grenzen und Messwert-Abgleich in
`VNSD_Ladegrenzen_Ladeleistung_de.md`. Kernaussagen für das BMS (v118, unverändert zu 117.7):

- **Ladestrom**: `Max_Charge_Current_Calc`/`Master_Build_ChargeCurrentLimit` begrenzen auf maximal
  **50,0 A** über eine 18(Temperatur, −25…60 °C in 5-°C-Schritten)×12(SOC-Stützstellen
  0/5/10/20/30/40/50/60/70/80/90/95/100 %)-Matrix (`g_chargeCurrentDerateMatrix`, Prozentwert × 5 =
  0,1-A-Einheiten). Ladeschlussspannung fest **57,6 V** (`0x240`), am CAN auf max. 60,9 V begrenzt.
- **Entladestrom**: `Max_Discharge_Current_Calc` (`0x08012758`) verwendet eine eigene
  18×5-Matrix (`g_dischargeCurrentDerateMatrix`), indiziert über Temperaturzone (dieselbe
  18-Punkte-Achse `g_tempAxisTable`) und eine **Zellspannungs-Bracket** (nicht SOC): für
  Temperaturzonen-Index < 5 wird die Spannungsachse `g_dischargeVoltAxisWarm`
  **{0, 2400, 2550, 2750, 3600} mV**, sonst `g_dischargeVoltAxisCold`
  **{0, 2750, 3000, 3050, 3600} mV** verwendet (Achsen-Namen scheinen mit dem
  Temperaturbereich vertauscht — Index < 5 entspricht laut `g_tempAxisTable` den kältesten Zonen;
  siehe §12). Ausgelesene Rohwerte (18×5-Matrix, Prozent × 5 = 0,1-A-Einheiten, 0 = Sperre):

  | Temp.-Zone (untere Grenze) | Bracket 0 (≤ Achswert 1) | 1 | 2 | 3 | 4 (oberste Zelle) |
  |---|---|---|---|---|---|
  | ≤ −25 °C | 0 | 0 | 0 | 0 | 0 |
  | −20 °C | 0 | 0 | 10 | 20 | 50 |
  | −15 °C | 0 | 0 | 15 | 30 | 60 |
  | −10 °C | 0 | 0 | 20 | 30 | 60 |
  | −5 °C | 0 | 0 | 20 | 40 | 60 |
  | 0 °C | 0 | 0 | 20 | 50 | 70 |
  | 5 °C | 0 | 0 | 20 | 50 | 80 |
  | 10 °C | 0 | 0 | 20 | 50 | 90 |
  | 15–45 °C (Zonen 8–12) | 0 | 0 | 20 | 50 | 100 |
  | 50 °C | 0 | 0 | 20 | 40 | 80 |
  | 55 °C | 0 | 0 | 20 | 40 | 60 |
  | 60 °C | 0 | 0 | 20 | 40 | 40 |
  | 65 °C | 0 | 0 | 20 | 20 | 20 |
  | ≥ 70 °C | 0 | 0 | 0 | 0 | 0 |

  100 % ⇒ 50,0 A. Anders als beim Laden ist die Entladegrenze also primär **spannungs-**, nicht
  SOC-geführt: erst oberhalb der jeweiligen dritten/vierten Spannungsschwelle wird nennenswert Strom
  freigegeben, im ungünstigsten Bracket (niedrige Zellspannung) bleibt sie bei 0.
- **Immer nur ein Pack lädt** — physisch erzwungen (§6). Diese Grenze gilt unabhängig von der
  Quelle (AC oder PV) und ist unverändert zu 117.7.

## 9. Schutzfunktionen

`BMS_Config_Params_Factory_Reset` (`0x0800a98c`) und `BMS_Config_Params_Load_Or_Default`
(`0x08009ccc`, bei Checksum-/Tag-Mismatch) setzen dieselben ~80 Felder; Ausgangswerte
(Trip1/Trip2/Release jeweils in 0,1 °C bzw. 0,1 V bzw. Ampere, Timeouts in ms):

| Schutzgruppe | Trip1 | Trip1-TO | Trip2 | Trip2-TO | Release | Release-TO |
|---|---|---|---|---|---|---|
| MOS-Temp B Hi | 60,0 °C | 3000 | 65,0 °C | 3000 | 60,0 °C | 3000 |
| MOS-Temp B Lo | −25,0 °C | 3000 | −30,0 °C | 3000 | −25,0 °C | 3000 |
| MOS-Temp A Hi | 90,0 °C | 3000 | 114,6 °C | 3000 | 85,0 °C | 3000 |
| Lade-NTC Hi/Lo | 60,0 / −15,0 °C | 5000 | 65,0 / −20,0 °C | 5000 | 56,6 / −17,0 °C | 5000 |
| Entlade-NTC Hi/Lo | 55,0 / −15,0 °C | 5000 | 60,0 / −20,0 °C | 5000 | 56,6 / −17,0 °C | 5000 |
| Zell-OVP | 3,600 V | 1000 | 3,650/3,700 V (a/b) | 3000/1000 | 3,350 V | 1000 |
| Zell-UVP | 2,800 V | 1000 | 2,700/2,600 V (a/b) | 3000/1000 | 3,150 V | 1000 |
| Entlade-OC/SC | 5,5 A (OC) | 3000 | 6,0 A (SC1) / 100 A (SC2) | **5000**/500 (Factory) bzw. **20000**/500 (Load-Default) | Release-TO 60000 |
| Lade-OC/SC | 5,5 A (OC) | 3000 | 6,0 A (SC1) / 100 A (SC2) | **10000**/500 (Factory) bzw. **20000**/500 (Load-Default) | Release-TO 60000 |

**Bestätigte Inkonsistenz** (Byte-für-Byte in beiden Funktionen nachvollzogen, nicht nur aus dem
Aufgabentext übernommen): `g_cfgDsgScTrip1Timeout`/`g_cfgChgScTrip1Timeout` sind in
`BMS_Config_Params_Factory_Reset` **5000 / 10000 ms**, in `BMS_Config_Params_Load_Or_Default`
dagegen **20000 / 20000 ms**. Da Load_Or_Default bei jedem Boot mit ungültigem/fehlendem
Konfigurationsblock greift (also der praktisch relevante Pfad bei Erstinbetriebnahme oder nach
Versions-Tag-Wechsel, s. §0a), gilt im Feld vermutlich meist 20000 ms für beide Kurzschluss-Stufe-1-
Zeiten, während ein expliziter Werksreset (Shell-Kommando `factory`? — nein, das ruft
`KA495XX_SetSleepMode` auf, s. §4.3; der tatsächliche Trigger für `BMS_Config_Params_Factory_Reset`
wurde in dieser Sitzung nicht lokalisiert, s. §12) andere Werte setzen würde. Diese Inkonsistenz
existiert unverändert in 117.7 und 118 (Diff zeigt keine Abweichung außer dem Versions-Tag).

**Statuswörter**: `g_protState1` (Bit 0/1 Lade-/Entlade-OC Stufe 2, Bit 5 Zell-UVP Stufe 2 → SOC 0 %,
Bit 6 Zell-OVP Stufe 2 → SOC 100 %), `g_protState2` (MOS-/Zell-Übertemperatur-Latches, Stufe 1/2,
Hi/Lo, Lade/Entlade), `g_protState3` (HW-Überstromkomparator-Gegenprüfung, NTC-/MOS-Sensor-
Bereichsfehler Bit 5–7, plus ein von der MOSFET-Ansteuerung gelesenes Bit), `g_protState4`
(AFE-Kommunikationsfehler-Latch Bit 3, geht in `Warning_Bitmask_Builder` ein). Diese vier Bytes
speisen `CAN_TX_PF4_ProtectWarnings` (zwei 16-Bit-Warnwörter + globales Fault-Byte) und den
RS485-Opcode `0x09`.

## 10. OTA/Update

- **CAN-OTA**: `BMS_OTA_Flash_Writer` (`0x0800d264`) puffert 1-KB-Blöcke, prüft jeden Block per
  **CRC16/Modbus** (`LIB_CRC16_Modbus_Calc`), schreibt sie ins externe/interne Flash bei
  **`0x0801E000`** (Staging), verifiziert nach Abschluss die Gesamt-CRC gegen `g_otaImageExpectedCrc`
  und das Tag `g_otaStagingVersionTag == 0x31534D42` (ASCII `'BMS1'`, little-endian). Bei Erfolg
  berechnet `OTA_ImageChecksum_OnesComplement` eine zusätzliche Einerkomplement-Prüfsumme und
  `OTA_Set_Reboot_Params` (`0x08011618`) schreibt den **Bootloader-Handshake**
  (Magic `0x11223344`/`0x55667788`, Ziel-Entry, Größe, Modus 2) in die gemeinsame Boot-Config-Seite
  (`HAL_Flash_Write_Config_Block`) — danach `vTaskDelay(1000)` und Reboot-Anforderung.
- **RS485-OTA**: `RS485_OTA_Flash_Handler` (`0x0800ca44`, Opcode `0x0e`) ist ein separater,
  RS485-getriebener OTA-Pfad mit eigenem Ziel-Pack-Feld (`g_rs485OtaTargetPackId`) — vermutlich zum
  Durchreichen von Firmware an Slave-Packs über den RS485-Bus, unabhängig vom CAN-OTA-Pfad.
- Byte-für-Byte-Vergleich mit 117.7 zeigt **keine** funktionalen Unterschiede in
  `BMS_OTA_Flash_Writer`/`_FinalizeTail` oder `OTA_Set_Reboot_Params` — nur adressbedingte
  Literal-Verschiebungen. OTA ist in beiden Ständen über CAN und RS485 möglich, mit identischem
  Handshake.

## 11. Sicherheitsbefunde

- **Fehlersperre trivial per CAN/RS485 löschbar**: CAN-Opcode `0x08` (`CAN_RX_System_Command_Handler`)
  und RS485 (`lock_set`-Shell-Kommando bzw. Parametergruppen-Schreibpfad) können
  `g_errorLockActiveFlag` ohne weitere Authentisierung außer der Bus-Zugehörigkeit zurücksetzen. Wer
  physischen/elektrischen Zugriff auf CAN oder RS485 hat, kann eine Schutzabschaltung aufheben.
  Ob das beabsichtigtes Werksverhalten (Service-Reset) oder eine Schwachstelle ist, lässt sich aus
  der Firmware allein nicht beurteilen.
- **Default-Shell-Login `BLUE`** ist im Klartext im Image hinterlegt (`s_BLUE_0801bc48`) und wird bei
  jeder `CLI_ShellInit` automatisch eingeloggt (kein Passwort-Zwang für die Werks-Session).
- **OTA ohne sichtbare Signaturprüfung**: Sowohl der CAN- als auch der RS485-OTA-Pfad verifizieren
  nur CRC16 (Übertragungsintegrität) und ein 4-Byte-ASCII-Tag (`'BMS1'`), keine kryptographische
  Signatur. Wer auf den CAN- oder RS485-Bus schreiben kann, kann beliebige Firmware einspielen,
  solange CRC und Tag passen.
- **`g_ka495xxAdcLsbScale`** (AFE-ADC-Skalierung) fließt sowohl in die Hardware-Schutzschwellen
  (OVP/UVP/OCD-Register-Encoding) als auch in die Coulomb-/SOC-Berechnung ein — ein falscher Wert
  (z. B. durch fehlerhafte Kalibrierung) würde gleichzeitig Schutzschwellen und angezeigten
  Strom/SOC verfälschen (Notiz aus dem Namensbefund, nicht weiter verifiziert).
- **Historische Fehlbenennungen mit „FATFS_"-Präfix**: mehrere zunächst als FatFs-Funktionen
  eingestufte Symbole erwiesen sich als reine FreeRTOS- oder letter-shell-Interna
  (`CLI_ProcessCommandLine`, `CLI_ResolveVariableReference`, `CLI_CheckCommandPermission`,
  `usTaskCheckFreeStackSpace`, `pvTaskIncrementMutexHeldCount` u. a., s. §2.3) und wurden im
  aktuellen Export korrigiert. `FATFS_Init_Log_File_If_Needed` (§4.4) trägt denselben verdächtigen
  Präfix und wurde in dieser Sitzung nicht umbenannt — sein tatsächliches Verhalten (Shell-Init,
  keine Dateioperation) spricht ebenfalls gegen eine echte FatFs-Anbindung.

## 12. Offene Punkte

- Welcher konkrete Auslöser `BMS_Config_Params_Factory_Reset` (im Gegensatz zum automatischen
  `_Load_Or_Default`-Pfad) tatsächlich aufruft, wurde in dieser Sitzung nicht zurückverfolgt
  (Caller-Liste zeigt 1 Aufrufer laut `_index.md`, nicht weiter verfolgt).
  Das Shell-Kommando `factory` ruft nachweislich `KA495XX_SetSleepMode` auf, nicht diese Funktion.
- Bedeutung der Register-Gruppen-ID `0x80` (RS485-Parametergruppe, 1 Wort) und der drei Register in
  Gruppe `0x400` nicht abschließend geklärt.
- Genaue Einheit von `KA495XX_ScaleInternalTempCode` (Rohcode vs. 0,1 °C) nicht unabhängig bestätigt.
- Ob die Vertauschung der Achsennamen `g_dischargeVoltAxisWarm`/`_Cold` in
  `Max_Discharge_Current_Calc` (§8) ein Benennungsfehler ist oder ob die zugrunde liegende
  `BMS_TempZoneIndexLookup`-Indexierung hier anders zu lesen ist als in der Ladeleistungs-Tabelle,
  wurde nicht abschließend geklärt.
- Auslösebedingung/genaue Zielzeichenkette von `CLI_NotifyPendingShell` (auch für das `clear`-
  Kommando genutzt) nicht vollständig verifiziert (Diskrepanz zwischen Kontext-Globals und
  dekompiliertem Quelltext laut Namensbefund).
- Ob `RS485_Register_Read_Handler`-Gruppen-ID `0x80` und die HW-Overcurrent-Komparator-Bits in
  `g_protState3` in Zusammenhang stehen, nicht geprüft.
- Genauer Adressbereich und Zweck des I2C-EEPROM-Gesamtlayouts (welche Bereiche zwischen `0x000` und
  `0xFC00` durch was belegt sind) wurde nur für die in diesem Dokument konkret referenzierten Felder
  geklärt, nicht vollständig kartiert.

## 13. Korrekturen gegenüber der alten Analyse

Bezug: `VNSD_BMS_FW_Analyse_v118_de.md` (Altanalyse, Ghidra-Import 2026-07-14/15).

- **Image-Basis falsch**: Altanalyse ging von `0x08000000` aus; korrekt ist `0x08002800` (Bootloader
  darunter). Jede in der Altanalyse genannte Flash-Adresse ist dadurch um `0x2800` zu niedrig.
- **MCU falsch identifiziert**: Altanalyse (und der dort referenzierte 117.7-Stand) nahmen
  **AT32F403A** an; die tatsächliche Peripherie-Adresslage (ADC1/2 auf `0x40020800`/`0x40020C00`,
  UID bei `0x1FFFF7F0`) gehört zu **Nations N32G45x** (s. §2.1). Die Feldbeobachtungen/Messwerte der
  Altanalyse bleiben davon unberührt, nur die MCU-/Register-Zuordnung war falsch.
- **Funktionsanzahl**: Altanalyse 550/550 benannt; aktueller Export **711/711** — vollständiger
  Neuaufbau mit korrigierter Basisadresse, nicht dieselbe Funktionsmenge zuzüglich Delta.
- **Ein Fund der Altanalyse bestätigt sich unabhängig**: Die dort (unter falscher Adresse,
  `FUN_08005a40` als vermuteter Nachfolger von `CAN_RX_Handler`) beschriebene neue Zeile — ein Byte
  aus CAN-Frame-Offset `0xD` wird zusätzlich nach `DAT_200028E0` übernommen — entspricht exakt dem in
  §0a gegen den korrekten 118-Export bestätigten Fund: `CAN_RX_Handler` schreibt seit v118 zusätzlich
  `g_dischargeLimitPct` aus Byte `0xD` des Master-Heartbeat-Frames. `DAT_200028E0` und
  `g_dischargeLimitPct` bezeichnen also denselben RAM-Global, nur unter altem bzw. neuem Namen.
- **148-W-Plateau nicht wiederholt**: Wie im Auftrag vorgegeben, wird die in früheren Dokumenten
  vermutete BMS-Ursache für das 148-W-Plateau hier nicht übernommen — die Ursache war die
  Omnibattery-Integration, nicht das BMS (s. `VNSD_Ladegrenzen_Ladeleistung_de.md` §4, dort mit
  Messwert-Herleitung).
- Weitere Einzelheiten (Register-/Struct-Layout, die laut Altanalyse-Kopfzeile „weiterhin gültig,
  sofern hier nicht abweichend beschrieben" sein sollten) wurden in dieser Sitzung nicht Punkt für
  Punkt nachverglichen, sondern komplett neu aus dem 118-Export mit korrigierter Basisadresse
  hergeleitet; Abweichungen zur Altanalyse jenseits der oben genannten wurden nicht systematisch
  gesammelt.
