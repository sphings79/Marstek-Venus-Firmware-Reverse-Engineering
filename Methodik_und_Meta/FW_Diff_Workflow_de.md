# Marstek Venus — Firmware-Diff-Workflow

> 🇬🇧 English version: [FW_Diff_Workflow_de.md](FW_Diff_Workflow_de.md)

Anleitung: Wie bei einer neuen Firmware-Version vorgegangen wird, um Änderungen zu identifizieren, Labels zu übertragen und die Dokumentation zu aktualisieren.

---

## Voraussetzungen

- Ghidra mit ReVa MCP ist verbunden
- Die alte FW-Version ist bereits in Ghidra analysiert und vollständig gelabelt
- Das neue FW-Binary liegt als Datei vor (vom User bereitgestellt oder per OTA-Download)

### Bekannte Ghidra-Projekte (Stand Juli 2026)

| FW | Ghidra-Pfad | Doku-Datei | Funktionen |
|---|---|---|---|
| BMS v117.7 | `/20251010135647565eb2036.bin` | `BMS/VNSD_BMS_FW_Analyse_v117.7_de.md` | 540 (100% benannt) |
| Control v149.2 | *(Pfad in Ghidra prüfen)* | `Control_FW/VNSD_Control_FW_Analyse_app_1492_0702_142136_de.md` | 1.623 (1.619 benannt, s. Control_FW/VNSD_Control_FW_Function_Tracking_new_de.md) |
| Micro/Inverter v116 | *(Pfad in Ghidra prüfen)* | `VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analyse_vd_inv_app_0116_de.md` | 452 |

---

## Schritt 1: Neues Binary importieren und analysieren

```
Tool: ghidra:import-file
  filePath: <lokaler Pfad zum neuen Binary>
  programName: <z.B. "BMS_v178" oder "Control_v150">

Tool: ghidra:analyze-program
  programPath: <Ghidra-Pfad des importierten Programms>
  forceFullAnalysis: true
  waitSeconds: 30
```

**Wichtig bei STM32-Binaries (BMS, Micro):**
- Flash-Basis auf `0x08000000` setzen (Memory Block umbenennen zu "flash", write=false)
- Compiler: RVDS/Keil ARM (Thumb-2, Little Endian)
- Nach Import: `analyze-program` mit `forceFullAnalysis: true`

**Bei Control FW:**
- Compiler: GCC (anders als BMS/Micro!)
- Gleiche Flash-Basis `0x08000000`

---

## Schritt 2: Diff-Session erstellen

```
Tool: ghidra:diff-create-session
  sourceProgramPath: <Pfad der ALTEN, vollständig gelabelten FW>
  destinationProgramPath: <Pfad der NEUEN FW>
  waitSeconds: 60
```

Die Source ist immer die alte, bekannte Version — Labels/Markup fließen VON source NACH destination.

Standard-Correlators (werden automatisch ausgeführt):
1. `symbol-name` — Matcht Funktionen mit gleichem Namen
2. `exact-bytes` — Identische Byte-Sequenzen
3. `exact-instructions` — Identische Instruktionen
4. `exact-mnemonics` — Identische Mnemonics
5. `duplicate-instructions` — Duplizierte Instruktionen
6. `function-reference` — Referenz-basiertes Matching

Falls viele Funktionen unmatched bleiben, zusätzlich:
```
Tool: ghidra:diff-add-correlator
  correlator: "combined-reference"
```

---

## Schritt 3: Diff-Summary abrufen

```
Tool: ghidra:diff-summary
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
  topN: 20
  includeBodyByteChanges: true
```

Das liefert:
- Anzahl gematchter/ungematchter Funktionen
- Top-N der am stärksten geänderten Funktionen
- Neue und entfernte Funktionen

---

## Schritt 4: Labels und Markup übertragen

### Automatisch (hohe Konfidenz)

```
Tool: ghidra:diff-transfer-markup
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
  confidence: 0.95
  waitSeconds: 30
```

Überträgt automatisch: Funktionsnamen, Prototypen, Datentypen, Kommentare — für alle Matches mit Similarity >= 95%.

### Manuell (niedrige Konfidenz)

Für Matches unter dem Confidence-Schwellwert einzeln prüfen:

```
Tool: ghidra:diff-function
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
  function: <Funktionsname oder Adresse>
```

Dann bei positivem Match einzeln anwenden:

```
Tool: ghidra:diff-apply-match
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
  sourceAddress: <Adresse in alter FW>
  destinationAddress: <Adresse in neuer FW>
```

---

## Schritt 5: Änderungen im Detail analysieren

### Geänderte Funktionen auflisten

```
Tool: ghidra:diff-list-functions
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
  category: "changed"
  sortBy: "similarity"
  includeBodyByteChanges: true
```

### Neue Funktionen (nur in neuer FW)

```
Tool: ghidra:diff-list-functions
  category: "added"
```

### Entfernte Funktionen (nur in alter FW)

```
Tool: ghidra:diff-list-functions
  category: "removed"
```

### String-Änderungen

```
Tool: ghidra:diff-strings
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
```

Neue Strings deuten oft auf neue Features, Register oder Protokolländerungen hin.

### Daten-Änderungen

```
Tool: ghidra:diff-data
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
```

---

## Schritt 6: Neue Funktionen analysieren und labeln

Für jede neue (ungematchte) Funktion in der neuen FW:

1. Dekompilieren: `ghidra:get-decompilation`
2. Cross-References prüfen: `ghidra:find-cross-references`
3. Caller-Kontext: `ghidra:get-callers-decompiled`
4. Label vergeben nach Naming-Convention (siehe unten)

---

## Schritt 7: Dokumentation aktualisieren

### Doku-Datei bestimmen

Jede Firmware hat ihre eigene Markdown-Datei — Erkenntnisse gehören immer in die passende FW-Datei:

| FW-Typ | Datei |
|---|---|
| BMS | `BMS/VNSD_BMS_FW_Analyse_v<VERSION>.md` |
| Control | `Control_FW/VNSD_Control_FW_Analyse_app_<VERSION>.md` |
| Micro/Inverter | `VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analyse_vd_inv_app_<VERSION>.md` |

### Was aktualisiert werden muss

1. **Binary-Fingerprint (§1):** Neue Version, Größe, Funktionsanzahl
2. **Versionshistorie:** Neuen Changelog-Eintrag hinzufügen (siehe Format unten)
3. **Betroffene Sektionen:** Wenn sich z.B. der SOC-Algorithmus geändert hat → §3/4 aktualisieren
4. **Funktionsliste (§12 bei BMS):** Neue/geänderte/entfernte Funktionen einpflegen

### Changelog-Format

```markdown
### vALT → vNEU (Datum)

**Summary:** Kurzbeschreibung der Hauptänderungen

**Geänderte Funktionen (X):**
- `Funktionsname` (Similarity X%) — Was hat sich geändert
- ...

**Neue Funktionen (X):**
- `Neuer_Name` @ `0xADRESSE` (XB) — Beschreibung

**Entfernte Funktionen (X):**
- `Alter_Name` — Nicht mehr vorhanden

**String-Änderungen:**
- Neu: "neuer string" — Bedeutung
- Entfernt: "alter string"
```

---

## Naming-Conventions

Funktionsnamen folgen dem Schema `PREFIX_Beschreibung`:

| Prefix | Bedeutung | Beispiel |
|---|---|---|
| `APP_` | BMS Applikationslogik | `APP_Get_Protection_Status_Byte` |
| `HAL_` | Hardware Abstraction | `HAL_GPIO_Write_Pin` |
| `LIB_` | Bibliothek/Utility | `LIB_CRC16_Modbus_Calc` |
| `INIT_` | Initialisierung | `INIT_All_Peripherals` |
| `CLI_` | Letter-Shell Debug | `CLI_Parse_Arguments` |
| `KA495XX_` | BMIC SPI-Treiber | `KA495XX_Read_CellVoltages_And_Temps` |
| `SOC_` | State of Charge | `SOC_Algorithm_Orchestrator` |
| `CAN_` | CAN-Bus Protokoll | `CAN_TX_BMS_Status_Reporter` |
| `RS485_` | RS485 Kommunikation | `RS485_Register_Read_Handler` |
| `I2C_EEPROM_` | EEPROM Zugriff | `I2C_EEPROM_Read_WithMutex` |
| `Protect_` | Batterieschutz | `Protect_Update_NTC_MinMax` |
| `NTC_` | Temperatursensoren | `NTC_Calculate_Temperature` |
| `OTA_` | Firmware Update | `BMS_OTA_Upgrade_StateMachine` |
| `RTOS_` | FreeRTOS Kernel | `RTOS_vPortExitCritical` |
| `SoftFloat_` | Software Float | `SoftFloat_Double_To_Int_Low` |
| `FATFS_` | Dateisystem | `FATFS_Find_Dir_Entry` |
| `BMS_` | BMS-spezifische Logik | `BMS_Config_Params_Load_Or_Default` |

Für Control FW zusätzlich:

| Prefix | Bedeutung |
|---|---|
| `WiFi_` | WiFi/WLAN |
| `ETH_` | Ethernet |
| `Modbus_` | Modbus TCP |
| `Cloud_` | Cloud-Telemetrie |
| `BLE_` | Bluetooth LE |
| `SMR_` | Energy Management |
| `MPPT_` | PV-Tracker |

**Regeln:**
- CamelCase nach Prefix: `SOC_Calc_Percentage_From_Range`
- Keine generischen Namen wie `FUN_xxxxxxxx` stehen lassen
- Bei Library-Funktionen (FreeRTOS, printf etc.) den Originalnamen verwenden
- Compiler-Intrinsics behalten: `__adddf3`, `__muldf3`, `__aeabi_*`

---

## Aufräumen nach dem Diff

```
Tool: ghidra:diff-delete-session
  sourceProgramPath: <alte FW>
  destinationProgramPath: <neue FW>
```

Diff-Sessions werden nicht automatisch gelöscht — nach Abschluss der Analyse manuell aufräumen.

---

## Dauerhafte Prüfungen für jede neue Firmware-Version

Diese Punkte stehen in jedem Versions-Diff-Bericht ganz oben („unverändert“ oder alter → neuer Wert; keinen still überspringen, nicht Verifizierbares benennen):

1. **Modbus-Register**: hinzugekommene/entfallene Register, Typ-/Skalen-/Anzahl-Änderungen, bediente Bereiche (die `.data`-Deskriptor-Tabelle dekodieren und vergleichen).
2. **BLE**: Befehlstabelle, Antwortlayouts, neue/entfallene Befehle.
3. **AC-seitiges Einspeiselimit**: maximale AC-Export-/Einspeiseleistung und ihre Begrenzung (2500 W bei der Venus D; 1500 W / Klassen 800/1200/1500 W bei der Venus A).
4. **Firmware-Update-Pfad**: ist ein BLE-OTA nach der Installation weiterhin möglich (neue Signatur-/Typprüfungen, entfernter BLE-OTA, reine Server-Updates)?
5. **Cloud-Endpunkte**: Domains, Ports, Pfade, Zertifikats-Pinning (hamedata.com, marstekcloud.com, MQTT-Hosts).
6. **Pack-Kapazität (BMS und Control)**: die maximale Anzahl Akkupacks, die die Firmware wirklich behandelt. Prüfen: (a) die Grenze im BMS-Sender der Pack-Telemetrie `CAN_TX_PerPack_10Msgs` (`pack_id < N`: D-BMS 118 = `< 10`, also Pack 1..9; A-BMS 1105 = `< 7`, also Pack 1..6), (b) die Pack-/Slot-Grenzen in der Master-Logik (`g_highestPackIndex`, Slot-Vergabe, Pack-Datensatz-Arrays und deren Schrittweite), (c) die Anzahl der Pack-Datensätze im Control (`Protocol_AA_CommandDispatch`: Einheit < 9; 7 Datensätze bei D-/A-Control) und die Modbus-Pack-Blöcke 34x00–34x33 (7 Packs), (d) die Zellzahl je Pack (16 bei D, 13 bei A) und welche Zell-/NTC-Register 0 bleiben, (e) ob die Pack-Entitäten der Suite (`battery_1…7_*`) noch passen. Ein Pack, dessen Frames nie gesendet werden, liest per Modbus 0, obwohl seine Register existieren.
Bekannte Werte (diese Tabelle bei jeder neuen Version erweitern):

| Image | Grenze in `CAN_TX_PerPack_10Msgs` | gesendete Packs | Zellen je Pack |
|---|---|---|---|
| D-BMS 117.7 | `pack_id < 10` | 1..9 | 16 |
| D-BMS 118 | `pack_id < 10` | 1..9 | 16 |
| A-BMS 1105 | `pack_id < 7` | 1..6 | 13 |
| A-BMS 110 | `pack_id < 7` | 1..6 | 13 |
| A-BMS 115 | `pack_id < 7` | 1..6 | 13 |
| E3-BMS 115 | noch zu prüfen | – | – |

Zählhinweis: `pack_id` zählt ab 1 (die Bedingung lautet `pack_id != 0 && pack_id < N`, die Datensätze werden mit `pack_id - 1` indiziert), `< 7` bedeutet also Pack 1..6 (sechs Packs) und `< 10` Pack 1..9 (neun Packs); `pack_id` 1 ist das eigene Pack des Master-BMS.

Das Control hält bei D und A je 7 Pack-Datensätze (Modbus 34000–34699); bei der A liest ein 7. Pack deshalb 0 per Modbus, obwohl seine Register existieren.

7. **Diff-Tabelle**: Diff-Tabelle je Version (eigene Datei, EN + DE), eine Zeile je Änderung, alter → neuer Wert und Fundstelle (Funktion + Adresse).

8. **Erneute Prüfung nicht funktionaler Register**: Jedes Register mit Status FUNCTIONLESS, NO_OP, UNKNOWN (offen) oder CONDITIONAL sowie jedes DEV-Register, das DEV bleibt, wird bei jeder neuen Version erneut geprüft, weil der Grund wegfallen kann (ein Empfänger bekommt einen Handler, ein Frame bekommt einen Sender, eine Konstante wird zum echten Wert). Die Liste je Gerät entsteht aus der Register-CSV (`Scripts/list_recheck_registers.py D|A|E3`); der Diff-Bericht nennt je Register „weiterhin funktionslos“ / „jetzt funktional“ / „geändert“ (Rezepte unten). Funktionierende Register werden natürlich weiter über den normalen Deskriptor-/Handler-Vergleich geprüft (Punkt 1).

### Erneute Prüfung nicht funktionaler Register (Rezepte)

| Was das Register wirkungslos machte | Prüfung in der neuen Version |
|---|---|
| Der Control-`Write_Handler` hat keinen Fall oder gibt eine Exception zurück (41001, 44000/44001, 45013–45019 reserviert) | Switch des Write_Handler (neue Fälle oder Rückgabewerte), Deskriptor-Tabelle und Bereichsprüfungen in `Modbus_Dispatcher` vergleichen |
| Der Empfänger filtert oder ignoriert den CAN-Frame (Micro D/E3: Class-2/3-Frames; Micro A: 0x18, 0x30–0x3E, 0x40–0x43, 0xFB; BMS-Class-3-Codes; 46500–46544, 47001) | FDCAN-Filter und Dispatcher-Fälle von Micro und BMS vergleichen (neue `case`-Werte) sowie die Sender des Control für diese Funktionscodes |
| Niemand sendet den Frame, der den Wert füllt (38000–38014 = Frames 0x40–0x43, 30214 = PGN 0x1007AA/0x1807AA, 0x1851, CAN-ID 0x00210100, 45011 = 0x100601AA) | alle Firmware-Images (Micro, BMS, Control) nach dem Sender durchsuchen (`CAN_Build_Arbitration_ID` mit dieser Funktion/ID) und die Empfangstabelle des Control (`g_canAaDispatchTable`, `.data`-Image dekodieren) prüfen |
| Der Sender setzt das Feld fest oder schreibt es nie (34x17 / 30106 = 0x1841 Byte 6–7, Warnwort 36000, Zellen 14–16, Fehlerbits, die kein Code setzt, SOH-Wörter 0x1852) | Sender-Funktion (`CAN_TX_PerPack_10Msgs`, Telemetrie-Builder) und jeden Schreiber des Quell-Globals vergleichen (Name und Adresse durchsuchen) |
| Pack- oder Zellgrenze (Pack 7 auf der A, `pack_id < N`) | Pack-Kapazitäts-Prüfung wiederholen (Standing Check 6) |
| Nur lokale Control-Wirkung / Wert wird verworfen (47400, 46500-Cache ohne Verbraucher, `g_*`-Variablen, die niemand liest) | Global auf Leser durchsuchen (Literal-Pool-Verweise eingeschlossen); neue Tasks/Timer prüfen |
| Wert nur nach einem Auslöser gültig (30211, 30213, 37001–37003, 30030–30035) | prüfen, ob der Auslöser noch existiert und der Wert nicht jetzt laufend aufgefrischt wird |
| Gegenstelle ohne verfügbare Firmware (Class-2-/MPPT-Knoten der D, A/E2-Geräte): Status UNKNOWN (offen) | prüfen, ob für diese Einheit inzwischen Firmware vorliegt; sonst bleibt es UNKNOWN und DEV |

## Checkliste

- [ ] Neues Binary importiert und analysiert
- [ ] Diff-Session erstellt, Summary geprüft
- [ ] Labels/Markup automatisch übertragen (confidence >= 0.95)
- [ ] Restliche Matches manuell geprüft und übertragen
- [ ] Neue Funktionen analysiert und gelabelt
- [ ] String- und Daten-Änderungen geprüft
- [ ] Changelog in Doku-Datei eingetragen
- [ ] Betroffene Doku-Sektionen aktualisiert
- [ ] Funktionsliste aktualisiert
- [ ] Diff-Session gelöscht
- [ ] Dauerhafte Prüfungen 1–8 (Modbus, BLE, Einspeiselimit, Update-Pfad, Cloud, Pack-Kapazität, Diff-Tabelle, erneute Prüfung nicht funktionaler Register) beantwortet
- [ ] Ghidra-Projekt gespeichert (`ghidra:checkin-program`)
