# Diff-Tabelle Control v149 → v1491 (Venus A)

> 🇬🇧 English version: [VNSA_Control_Diff_v149_v1491.md](VNSA_Control_Diff_v149_v1491.md)

**Stand:** 2026-11-05. Image `149_control_VNSA-0_app_0149_0528_102448.bin` („A Control 149“, Build-String `May 28 2026 10:23:00`) verglichen mit `1491_control_VNSA-0_app_1491_0528_145327.bin` („A Control 1491“, Build-String `May 28 2026 14:52:08`). Beides sind Venus-A-Images (`VNSA-0`), beide 382.976 B, beide in Ghidra mit Basis 0x08004800 geladen (je 1913 Funktionen).
**Zugehörige Dokumente:** [VNSA_Control_FW_v1491_Vollanalyse_de.md](VNSA_Control_FW_v1491_Vollanalyse_de.md), [VNSA_Control_FW_v149_Vollanalyse_de.md](VNSA_Control_FW_v149_Vollanalyse_de.md), [VNSA_Control_Diff_v149_v150_de.md](VNSA_Control_Diff_v149_v150_de.md)

## Zusammenfassung

**Richtung:** 149 ist die ältere Build, 1491 die neuere, am selben Tag gebaut (28. Mai 2026, rund 4,5 Stunden später). „1491 hat X“ heißt: X ist in 1491 und nicht in 149.

- **Die einzige funktionale Änderung ist die Firmware-Versionskonstante: 149 (0x95) → 1491 (0x5d3).** Sie wird an fünf Stellen (fünf Funktionen) geladen und steht im Versionswort bei 0x08010000. Kein anderer Opcode, Stringzeiger, Formatstring, aufgerufene Funktion oder Datenwert unterscheidet sich. Der Shell-Root-Eintrag wechselt von `VNSA-0 v149` zu `VNSA-0 v1491`, der Build-String von `May 28 2026 10:23:00` zu `May 28 2026 14:52:08`.
- Das 16-Bit-`movs rN,#0x95` wurde zum 32-Bit-`movw rN,#0x5d3` (+2 B je Stelle), dadurch verschiebt sich der Code hinter jeder Stelle und die Literal-Pools wandern; zusätzlich wurde aus dem String `v149` ein `v1491` (+1 B), und das Alignment-Padding ändert sich. Die Image-Größe bleibt gleich (382.976 B), weil das Padding vor den komprimierten Datenblöcken das Wachstum auffängt (siehe „Layout“).
- Die Versionskonstante wird nach außen gemeldet (`ems_ver` in der Shell, Gerätetyp-Code im Cloud-Report, BLE-Gerätestatus und MQTT-Telemetrie). Dass 0x5d3 die Firmware-Versionsnummer ist, folgt aus der Zahlengleichheit mit dem Dateinamen und aus diesen Verwendungen (siehe unten); der Code selbst trägt dafür keine weitere Bedeutung.

## Methode (wie „nur die Versionskonstante ändert sich“ geprüft wurde)

1. **Bytes.** Beide Dateien sind gleich groß; SHA-256 unterscheidet sich (`f14fede5...` vs. `271c6026...`). 99,39 % der 1491-Bytes (380.646 von 382.976) sind nach Maskierung von Adressen und Call-Operanden in 45 Ankerblöcken identisch zu 149; der Rest ist der Verschiebungsbereich um die fünf Stellen.
2. **Instruktionen.** Die aus Ghidra exportierten Flash-Listings (123.591 vs. 123.592 Einträge) wurden Eintrag für Eintrag verglichen, nachdem alle Adressoperanden (`adr`, `bl`, `ldr`-Literale, Zeigerwörter) maskiert waren. Ergebnis: 15 Differenzblöcke, jeder ist erklärt:

| # | Block | Bedeutung |
|---|---|---|
| 1 | 08010000 `uint 0x95` → `uint 0x5d3` | Versionswort |
| 2-6 | `movs rN,#0x95` → `movw rN,#0x5d3` bei 149 08011d00 / 080190e0 / 0801ad76 / 0802c8f4 / 0803a50e (1491: 08011d00 / 080190e4 / 0801ad7e / 0802c8fc / 0803a516) | die fünf Versions-Ladestellen (Register r3, r3, r3, r0, r1) |
| 7-10 | fünf `Alignment align(2)`-Einträge erscheinen / verschwinden neben den Literal-Pools dieser Funktionen | Padding wandert mit dem 2-B-Wachstum (Typisierung, kein Code) |
| 11 | `Alignment align(3)` bei 08050939 | Padding hinter dem längeren String `VNSA-0 v1491` |
| 12-14 | Strings „left“, „help“ als `string` bzw. `TerminatedCString` typisiert; `byte[3]`-Pad bei 0805a991 | Ghidra-Typisierung derselben Bytes, keine Inhaltsänderung |

   Kein anderes Mnemonic und kein anderer Operand unterscheidet sich, sobald Adressen maskiert sind.
3. **Dekompilat.** Die 1913 exportierten Funktionen lassen sich 1:1 per Name paaren. Die fünf Funktionen mit dem Versions-Load sind die einzigen mit anderem Opcode (gleiche Instruktionszahlen 234/234, 225/225, 751/751, 369/369, 35/35 und gleicher Kontrollfluss). Die übrigen 1908 unterscheiden sich nur in Adressoperanden, weil der Code hinter jeder Stelle um +2, +4 oder +8 Byte gewandert ist (Einstiegspunkte verschieben sich, z. B. `HTTP_GetUrlData` 08019048 → 0801904c, `Cloud_UploadTelemetryData` 0801a8d0 → 0801a8d8, `Config_LoadOrInit_DeviceSettings` 0802c6ec → 0802c6f4, `ShellCmd_get_ver` 0803a4f4 → 0803a4fc; spätere Funktionen um +8).
4. **Paarprüfung in Ghidra.** Anker: 1908 identisch, 5 teilweise, 0 ohne Gegenstück (von 1913); Call-Target-Prüfung über 9115 Aufrufe: 0 Befunde; RAM-Layout identisch zu 149 (alle 1429 synchronisierten Labels haben dieselbe Adresse und denselben Namen; die Mengen der referenzierenden Funktionen von 89 weiteren Globals sind identisch).

## Diff-Tabelle

| Bereich | 149 | 1491 | Funktion / Adresse (1491) | Beleg |
|---|---|---|---|---|
| Versionswort | `uint 0x95` | `uint 0x5d3` | 08010000 (danach `VNSA-0` bei 08010004) | Bytes `95 00 00 00` → `d3 05 00 00`; in keiner Version eine Code-Referenz |
| Shell `get_ver` | `ems_ver=\t%d` gibt 0x95 aus | gibt 0x5d3 aus | `ShellCmd_get_ver` 0803a4fc, `movw r1,#0x5d3` bei 0803a516 | Listing-Diff; Dekompilat `printf("ems_ver=\t%d\r\n",0x5d3)` |
| Build-String | `May 28 2026 10:23:00` @ 0803a557 | `May 28 2026 14:52:08` @ 0803a55f | `ShellCmd_get_ver` | Bytesuche |
| Gerätetyp-Code | `g_deviceTypeCode = 0x95` | `= 0x5d3` | `Config_LoadOrInit_DeviceSettings` 0802c6f4, `movw r0,#0x5d3` bei 0802c8fc (neben `g_deviceTypeStr = "VNSA-0"`) | Listing-Diff; Dekompilat |
| Version im Cloud-Report | 0x95 | 0x5d3 | `HTTP_Cloud_Reporting_Dispatcher` 08011c60 (`movw r3` bei 08011d00) | Listing-Diff |
| Version in HTTP-URL-Daten (`sv=`) | 0x95 | 0x5d3 | `HTTP_GetUrlData` 0801904c (`movw r3` bei 080190e4) | Listing-Diff |
| Version im Telemetrie-Upload | 0x95 | 0x5d3 | `Cloud_UploadTelemetryData` 0801a8d8 (`movw r3` bei 0801ad7e) | Listing-Diff |
| Shell-Root-Eintrag | `VNSA-0 v149` @ 08050924 / 0805a8f5 | `VNSA-0 v1491` @ 0805092c / 0805a908 | `CLI_InitSession` | Bytesuche; Dekompilat `CLI_FindEntryByName(session,"VNSA-0 v1491",...)` |
| Verbraucher der Konstante (Code unverändert) | | | `g_deviceTypeCode` wird gelesen von `Cloud_Report_FillDeviceInfo`, `BLE_Build_DeviceStatus_Response`, `BLE_App_Command_Dispatcher` (formatierte Geräteinfo), `MQTT_Telemetry_Struct_Builder` (`g_telemetryDeviceVersion`), `MQTT_StatusReport_BuildAndPublish` | Dekompilat-Exporte: das Gerät meldet 1491, wo es 149 meldete |

### Layout (nicht funktional)

| Punkt | 149 | 1491 |
|---|---|---|
| Image-Größe | 382.976 B | 382.976 B |
| Scatter-Tabelle `Region$$Table$$Base` | 08060840 | 08060850 |
| `ShellCommand[78]` | 08060870 | 08060880 |
| `.data`-Ladeimage (2480 B, `Load$$RW_IRAM1$$Base`) | 08060D50 | 08060D60 |
| XMC-Ladeimage (1164 B, `Load$$RW_XMC$$Base`) | 08061700 | 08061710 |
| 0xFF-Füllung vor dem Footer | 1108 B ab 08061B8C | 1092 B ab 08061B9C |
| Footer `VNSA` | 08061FE0 | 08061FE0 |

Das Code-Wachstum (5 x 2 B für das `movw`, 1 B für `v1491`, Alignment) verschiebt alles hinter der ersten Stelle um 4 bis 16 B nach oben; der Footer bleibt bei 08061FE0, weil die 0xFF-Füllung um 16 B schrumpft.

## Prüfliste (Standard-Prüfungen)

| Frage | Ergebnis 149 vs. 1491 |
|---|---|
| Modbus-Register | **Unverändert.** `Write_Handler`, `FC03_Read_Handler`, `RS485_Modbus_RegisterMap_Init` sind identische Paare (nach Adressmaskierung identisch); `g_modbusRegDescTable` liegt an derselben RAM-Adresse mit gleichem Typ und gleicher Größe (246 x 12 B). Kein Register hinzugekommen oder entfallen. Registerinhalte und Statusklassifikation wurden nicht neu abgeleitet (wie bei 149). |
| BLE | **Unverändert** bis auf die gemeldete Version. Namenspräfix `MST_VNSA_`; `BLE_App_Command_Dispatcher` ist ein identisches Paar, Befehlstabelle und Antwortlayouts sind also unverändert. Die Gerätestatus-Antwort (`BLE_Build_DeviceStatus_Response`) trägt über `g_deviceTypeCode` jetzt 1491 statt 149. |
| AC-Einspeiselimit | **Unverändert.** `Grid_Export_Power_Limiter` identisches Paar: Startwert auf 2500 W (0x9c4) begrenzt, obere Grenze 2400 W (0x960); `Write_Handler`-Text für 44003 (1500 W) identisch. |
| Update-Pfad (BLE-OTA) | **Unverändert.** `BLE_Cmd_OTA_Validate`, `BLE_Cmd_OTA_Init`, `BLE_Cmd_OTA_WriteSetup`, `BLE_OTA_WriteDataChunk`, `OTA_Is_VNSA_Model` (prüft `VNSA-0`), `OTA_Slot_Config_Validate`, `YModem_ReceiveFile` und `HTTP_FindHeaderBodySeparator` sind identische Paare. BLE-OTA ist soweit der Code zeigt möglich (**nicht am Gerät getestet**). Ein Versionsvergleich für Update-Angebote (falls vorhanden) passiert außerhalb des Images, **nicht geprüft**. |
| Cloud-Endpunkte | **Unverändert.** Nur `hamedata.com`, Upload über `setVenusDReporting` (AES-128-ECB + Base64Url), kein `marstekcloud.com`; alle Endpunkt-Strings gleich. Geändert: der Versionswert (`sv=`, Geräteinfo, Telemetrie-Upload) lautet jetzt 1491. Wie die Cloud diesen Wert behandelt: **nicht geprüft**. |
| Pack-Kapazität | **Unverändert.** `BMS_CellVoltage_MinMax_PerPack` identisches Paar (6 Packs `< 6`, 13 Zellen `< 0xd`), gleicher 0x58-B-Packdatensatz und 0x35b-B-Historienslot. Die BMS-seitige Grenze (`CAN_TX_PerPack_10Msgs`) gehört nicht zum Control-Image: **nicht geprüft**. |
| Nicht funktionale Register (Neuprüfung) | **Aus diesem Image keine Änderung möglich**, da Registercode und -tabelle unverändert sind; der Stand von 149 gilt weiter (die A-Registerklassifikation selbst: nicht geprüft). |
| Diff-Tabelle | Oben. |

## Offene Punkte

- Das Wort bei 08010000 (0x95 in 149, 0x5d3 in 1491, danach `VNSA-0`) hat in keinem Image eine Code-Referenz und kein Literal-Pool-Wort, das darauf zeigt. Sein Verbraucher ist unbekannt; ein Bootloader oder das Update-Werkzeug, das den Anwendungs-Header liest, ist **vermutet, nicht gezeigt**.
- Ob der geänderte Versionswert auf Cloud- oder App-Seite anderes Verhalten auslöst, ist nicht geklärt (er wird nur gemeldet).
- Nicht geprüft: der Vergleich von 1491 mit 150, 151 und den Betas (1508, 1509); in diesem Dokument ist 1491 gegen 149 der einzige Vergleich.
