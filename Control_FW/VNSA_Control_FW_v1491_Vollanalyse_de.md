# Control-FW v1491 (Venus A) — Vollanalyse (Port aus 149)

> 🇬🇧 English version: [VNSA_Control_FW_v1491_Full_Analysis.md](VNSA_Control_FW_v1491_Full_Analysis.md)

**Status:** abgeschlossen 2026-11-05. Das Image wurde analysiert, indem die fertige v149-Analyse portiert und der Port am 1491-Code verifiziert wurde. 1491 ist **funktional identisch zu 149**; nur die Firmware-Versionskonstante unterscheidet sich.
**Binary:** `1491_control_VNSA-0_app_1491_0528_145327.bin` (Archiv `Marstek FW Archiv/firmwares/VNSA-0/Control/1491/`, „A Control 1491“)
**Port-Quelle:** Venus A Control v149, siehe [VNSA_Control_FW_v149_Vollanalyse_de.md](VNSA_Control_FW_v149_Vollanalyse_de.md). Alles, was 1491 mit 149 teilt (Methode des Ports aus 150, RAM-Layout, Geräteidentität, Unterschiede zu 150, gemeinsame Themen), ist dort dokumentiert und wird hier **nicht wiederholt**.
**Unterschiede 149 → 1491:** [VNSA_Control_Diff_v149_v1491_de.md](VNSA_Control_Diff_v149_v1491_de.md)
**Ghidra:** `/VNSA/Control/Control_1491_VNSA-0_app_1491_0528_145327.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** privater Arbeitsexport (nicht Teil dieses Repositorys): 1913 Funktionen, 0 Dekompilierfehler, dazu ein Flash-Listing und `globals_and_registers.txt` (2589 benannte RAM-/Peripherie-Symbole), gleiche Struktur wie bei 149.

---

## 0. Zusammenfassung

- **Gleicher Code wie 149.** Nach Maskierung von Adressen und Call-Operanden sind 99,39 % der Bytes (380.646 von 382.976) identisch; der Instruktionsvergleich der Flash-Listings findet nur die Versionskonstante und ihre Nebenwirkungen (Diff-Tabelle). Jede Aussage der 149-Analyse zu Code, Modbus, BLE, OTA, Cloud, Packs und zum Vergleich mit 150 gilt daher unverändert für 1491.
- **1491 ist Venus A:** `VNSA-0` / `Venus A` / BLE-Präfix `MST_VNSA_`, Build-String `May 28 2026 14:52:08` (149: `May 28 2026 10:23:00`, am selben Tag).
- **Was sich unterscheidet:** die Firmware-Versionskonstante 0x95 (149) → 0x5d3 (1491) an fünf `movw`-Stellen (`HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `Config_LoadOrInit_DeviceSettings`, `ShellCmd_get_ver`), das Versionswort bei 0x08010000, der Shell-Root-Eintrag `VNSA-0 v149` → `VNSA-0 v1491`, der Build-String; das Layout verschiebt sich hinter der ersten Stelle um 4 bis 16 B.
- **Verifikation:** alle Gates bestanden (siehe §3); Tags identisch zu 149 (1786 `NAME_VERIFIED`, 127 `NAME_LOW_CONFIDENCE`).

## 1. Binär-Fingerabdruck

| | A 1491 | A 149 (Referenz) |
|---|---|---|
| Größe | 382.976 B (0x5D800) | 382.976 B |
| SHA-256 | `271c60265478cf92eba21824814632c185d9344244112a842b547fd1e65b6a16` | `f14fede53a07462ecbba7c37c0c1b24ac910a5962206a54f2ab21c62034ea07e` |
| Linkadresse | **0x08004800**–0x08061FFF | gleich |
| MCU / Toolchain | Nations **N32G45x** (Cortex-M4F), Keil ARMCC | gleich |
| Build-String (`ShellCmd_get_ver`) | `May 28 2026 14:52:08` @ 0x0803A55F | `May 28 2026 10:23:00` @ 0x0803A557 |
| Versionswort | 0x5d3 (1491) @ 0x08010000, danach `VNSA-0` | 0x95 (149) |
| Shell-Root-Eintrag (`CLI_InitSession`) | `VNSA-0 v1491` @ 0x0805092C, 0x0805A908 | `VNSA-0 v149` |
| Image-Footer | `VNSA` @ 0x08061FE0, 0xFF-Füllung (1092 B) ab 0x08061B9C | 1108 B ab 0x08061B8C |
| Funktionen in Ghidra | 1913 | 1913 |

Basisbeleg wie bei 149: initialer Stackpointer und Reset-Vektor lösen bei Basis 0x08004800 auf. Speicherkarte, Scatter-Tabelle (hier `Region$$Table$$Base` @ 0x08060850, `.data`-Ladeimage @ 0x08060D60, XMC-Ladeimage @ 0x08061710, `ShellCommand[78]` @ 0x08060880; Größen unverändert) und RAM-Layout entsprechen [149, §2 und §3.3](VNSA_Control_FW_v149_Vollanalyse_de.md); das RAM-Layout ist identisch zu 149 (nur Flash-Adressen verschieben sich).

## 2. Was sich gegenüber 149 unterscheidet

Details mit Adressen und Belegen: [VNSA_Control_Diff_v149_v1491_de.md](VNSA_Control_Diff_v149_v1491_de.md).

- Fünf Funktionen laden die Konstante: aus `movs rN,#0x95` wurde `movw rN,#0x5d3` (je +2 B); die Literal-Pools dahinter wandern um +4 / +8 B, Einstiegspunkte verschieben sich (z. B. `Config_LoadOrInit_DeviceSettings` 0802c6ec → 0802c6f4).
- Die Konstante geht hinaus als `ems_ver` (Shell `get_ver`), als Gerätetyp-Code `g_deviceTypeCode` (Cloud-Report, BLE-Gerätestatus, MQTT-Telemetrie) sowie als Version in den HTTP-URL-Daten (`sv=`) und im Telemetrie-Upload.
- Kein anderer Opcode, String, Formatstring oder aufgerufene Funktion unterscheidet sich; die übrigen 1908 Funktionspaare unterscheiden sich nur in Adressoperanden.

## 3. Bearbeitungsstand in Ghidra

- Import 1638 Funktionen (Auto-Analyse), Port aus 149 über Anker: 1908 identische Paare, 5 teilweise, 0 ohne Gegenstück; Layout (1913 Funktionen), Segmente, Signaturen, Flash-Referenzen, Flow-Overrides (28 angewendet, 2 zurückgesetzt), 64 TBB/TBH-Sprungtabellen typisiert (24 Decompiler-Overrides, Case-Labels gegen Tabellen: 0 Abweichungen), RAM-Labels (1429 aus dem Namensabgleich, 89 weitere nach Prüfung auf gleiche Referenzmengen übernommen, 94 `<object>_off<hex>`), 1336 RAM-/XMC-Datentypen, 977 Peripherie-Labels, 26 Plates.
- Gates (alle 0): undefinierte Läufe, verwaiste Läufe, Fluss in Daten, Funktionszeiger auf Nicht-Code, Code nur über Literal, `FUN_*`-Namen, doppelte Namen/Labels, Platzhalter, Shell-Command-Plates, Default-Prototypen, Task-Tabellen-Abweichungen, noreturn; unbenannte referenzierte RAM-Globals 0. Call-Target-Prüfung über 9115 Aufrufe: 0 Befunde.
- Plates mit Versions- oder Build-Zeit-Angaben wurden für 1491 korrigiert (`ShellCmd_get_ver`, `Config_LoadOrInit_DeviceSettings`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `CLI_InitSession`).
- Export: 1913 dekompilierte Funktionen mit 0 Fehlern, Flash-Listing, `globals_and_registers.txt` (2589 Symbole). Namensqualität wie bei 149: `<object>_off<hex>` und layoutabgeleitete Namen sind Adressrelations-Namen (geringe Sicherheit).

## 4. Standard-Prüfungen

Alle gegenüber 149 unverändert (Code identisch): Modbus-Register, BLE, AC-Einspeiselimit (2500 W Startklemme / 2400 W obere Klemme, 1500-W-Text bei 44003), BLE-OTA-Pfad, Cloud-Endpunkte (nur `hamedata.com`), Pack-Kapazität (Control-Seite 6 Packs / 13 Zellen). Beleg je Punkt und Grenzen der Prüfung: [VNSA_Control_Diff_v149_v1491_de.md](VNSA_Control_Diff_v149_v1491_de.md), „Prüfliste“. Der gemeldete Versionswert ist die einzige sichtbare Änderung.

## 5. Offene Punkte und Grenzen

- **Wort bei 0x08010000** (0x95 in 149, 0x5d3 in 1491, vor `VNSA-0`): in keinem Image eine Code-Referenz und kein Literal-Pool-Wort, das darauf zeigt. Verbraucher unbekannt; wahrscheinlich der Bootloader oder das Update-Werkzeug, das den Anwendungs-Header liest (**vermutet, nicht gezeigt**). Als unbenanntes `uint` belassen.
- Dass 0x5d3 die Firmware-Versionsnummer ist, wird aus der Zahlengleichheit mit dem Dateinamen und aus den Verwendungen (`ems_ver=%d`, `g_deviceTypeCode`, `sv=`) geschlossen; kein Symbol und keine Dokumentation benennt sie.
- Wie Cloud oder App auf Version 1491 statt 149 reagieren: nicht geprüft.
- Alles, was die 149-Analyse als ungeprüft aufführt (Registertabelle Eintrag für Eintrag, BMS-seitige Pack-Grenze, Cloud-TLS, Status nicht funktionaler Register), ist auch für 1491 ungeprüft.
- Nicht Teil dieses Dokuments: Vergleich von 1491 mit 150, 151 und den Betas.
