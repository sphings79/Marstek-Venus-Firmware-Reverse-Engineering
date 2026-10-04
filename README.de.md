# Marstek Venus D — Firmware Reverse-Engineering

> 🇬🇧 English version: [README.md](README.md)

Statische Reverse-Engineering-Analyse der Marstek-Venus-D-Firmware (Control/EMS, Micro-Inverter/VNS,
BMS) mittels Ghidra, als Grundlage für eine Home-Assistant-Integration über die Modbus-TCP-Schnittstelle
des Geräts. Alle Erkenntnisse basieren auf statischer Analyse der Firmware-Images, nicht auf Live-Debug-
Zugriff (SWD/JTAG), sofern nicht anders vermerkt.

**Gerät:** Marstek Venus D/E (VNSD-0), STM32 ARM Cortex-M, FreeRTOS
**Analysierte Firmware-Versionen:** Control 150 (aktuell) / 149.2 / 147, Micro/VNS 116 (aktuell) / 115,
BMS 118 (aktuell) / 117.7

## Binary-Fingerprint (alle analysierten Firmware-Images)

Live aus Ghidra verifiziert (Stand 2026-07-15). Der "Funktionen benannt"-Anteil ist reiner
Ghidra-Symbolstatus. BMS 118 wurde inhaltlich vollständig analysiert (Diff, Struct-/Register-Layouts)
und die Erkenntnisse wurden am 2026-07-15 als Ghidra-Symbolnamen zurückgeschrieben (526 per
Diff-Markup-Transfer aus 117.7, 24 neue einzeln benannt) — daher 100 %. Control 147 wurde am selben Tag
komplett neu erschlossen: 1530 Funktionen per Diff-Markup-Transfer aus 149.2, 340 nur-in-147-Funktionen
per Subagenten-Analyse einzeln benannt (davon 23 als Ghidra-Funktionsgrenzen-Artefakte markiert statt
mit erfundener Semantik versehen) — Details in `Control_FW/VNSD_Control_FW_Analyse_147_202601281721320b2053125_de.md`.

| Firmware | Version | Datei | Größe | Flash-Bereich | Initial SP | Reset Handler | Funktionen (benannt/gesamt) | Strings |
|---|---|---|---|---|---|---|---|---|
| Control | **150 (aktuell)** | `VNSD-0_app_0150_0805_115146.bin` | 389.120 B (380 KB) | `0x08000000–0x0805EFFF` | `0x2001F7E8` | `0x08004A71` | 1627 / 1884 (86,4 %) ¹ | 1804 ¹ |
| Control | 149.2 | `Control_149.2_VNSD-0_app_1492_0702_142136.bin` | 385.024 B (376 KB) | `0x08000000–0x0805DFFF` | `0x2001F7D8` | `0x08004A71` | 1618 / 1622 (99,8 %) | 1743 |
| Control | 147 | `Control_147_...0b2053125.bin` | 372.736 B (364 KB) | `0x08000000–0x0805AFFF` | `0x2001F3E0` | `0x08004A71` | 1870 / 1870 (100 %) | 1302 |
| Micro/VNS | **116 (aktuell)** | `Micro_VNS_116_vd_inv_app_0116_0702_ota_163439.bin` | 115.712 B (113 KB) | `0x08000000–0x0801C3FF` | `0x20009A70` | `0x080042AD` | 392 / 445 (88,1 %) | 251 |
| Micro/VNS | 115 | `Micro_VNS_115_...0c0e30687.bin` | 115.712 B (113 KB) | `0x08000000–0x0801C3FF` | `0x20009A70` | `0x080042B1` | 13 / 190 (6,8 %, nur Versions-Diff gg. 116) | 102 |
| BMS | **118 (aktuell)** | `BMS_118_20260119100535e43806957.bin` | 106.496 B (104 KB) | `0x08000000–0x08019FFF` | `0x2000CBA0` | `0x08002A6D` | 578 / 649 (89,1 %) ¹ | 263 ¹ |
| BMS | 117.7 | `BMS_117.7_20251010135647565eb2036.bin` | 106.496 B (104 KB) | `0x08000000–0x08019FFF` | `0x2000CB90` | `0x08002A6D` | 552 / 552 (100 %) | 260 |

¹ Stand 2026-09-24, live aus Ghidra gezählt. Die Zahlen sind höher als früher, weil seitdem
weitere Funktionen und Strings definiert wurden. Die übrigen Zeilen sind Stand 2026-07-15.

**Gemeinsame Eigenschaften aller sieben Images:** ARM Cortex-M4F, Thumb-2, Little-Endian, FreeRTOS
(`heap_4`, `ARM_CM4F`-Port), Flash-Basis `0x08000000`.

| Firmware | Compiler | Crypto | Zellmonitoring | Kommunikation |
|---|---|---|---|---|
| Control | GCC | mbedTLS 2.28.10 | — | WiFi + Ethernet (CH395) + RS485 + CAN |
| Micro/VNS | RVDS/Keil ARM | — (keine) | — | CAN + RS485 |
| BMS | RVDS/Keil ARM | — (keine) | KA495XX (BMIC) | CAN + RS485 |

Auffällig: Micro/VNS 115 und 116 sowie BMS 117.7 und 118 belegen jeweils exakt denselben
Flash-Adressbereich trotz unterschiedlichen Funktionsumfangs — spricht für eine feste OTA-Slot-Größe
pro Firmware-Typ statt variabler Image-Größe.

## Ordnerstruktur

**Dateibenennung:** Dokumente, die nur für ein Gerät gelten, tragen das Gerätepräfix `VNSD_`
(Venus D), `VNSA_` (Venus A) oder `VNSE3_` (Venus E3). Ohne Präfix sind nur geräteübergreifende
Dokumente (Methodik, Gerätevergleich, Diff-Workflow). Nicht umbenannt ist der Unterordner
`Modbus_RS485_TCP/Scan_Logs/`.

Dieses Verzeichnis ist die Wurzel des Repositories; alle Pfade in den Dokumenten sind
repo-relativ (z. B. `Modbus_RS485_TCP/…`, `Control_FW/…`, `BMS/…`). Was nur lokal existiert,
steht im Abschnitt „Nicht im Repository (nur lokal)“ weiter unten.

| Ordner | Inhalt |
|---|---|
| `Control_FW/` | Control-/EMS-Firmware (Hauptprozessor): vollständige Funktionsanalyse für v149.2 (Namens-Tracking-Tabelle, ~1600 Funktionen, Batch-Historie) und v147 (Diff-basierte Analyse gg. 149.2, 340 nur-in-147-Funktionen einzeln dokumentiert). Für v150 zusätzlich der Cloud-Watchdog samt Ursachenanalyse der 30-Minuten-Netzausfälle (§4a der v150-Analyse) und die Fallstudie `VNSD_Cloud_Watchdog_Fallstudie_2026-08-26_de.md` — inklusive der vier widerlegten Zwischenhypothesen und der Methodik, die sie erledigt hat. Umrechnung des BMS-Ladelimits in Leistung, PV-Abregelung und die 42011/force_mode-Kopplung in `VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md`; Venus-A-Control-1509-Analyse `VNSA_Control_FW_v1509_Vollanalyse_de.md`; Firmware-Fehler WLAN-Disconnect-URC `VNSD_FW_Bug_WLAN_Disconnected_URC_2026-10-02_de.md` |
| `VNS_Micro_Inverter/` | Micro-Inverter-/VNS-Firmware: Funktionsanalyse, Register-Maps, Versions-Diff 115→116, Fehler-/Warncodes. Dateien mit Gerätepräfix (`VNSA_…`) gelten **nur** für dieses Modell — der Venus-A-Wechselrichter ist eine eigene Codebasis (0,4 % gemeinsamer Code mit dem D), seine Fehlerbits sind in `VNSA_Fehlercodes_Micro_FW_Analyse_de.md` getrennt dokumentiert; Venus-A-Micro 1211 (Novosense NS800RT) Analyse: `VNSA_Micro_FW_v1211_Vollanalyse_de.md` |
| `BMS/` | BMS-Firmware-Analyse (v117.7, v118); Pack-Rotation, Entlade-Untergrenze, Sondermodi und gemessener Lade-Taper in `VNSD_BMS_Pack_Rotation_Taper_v118_de.md`; SOH, Zyklenzähler, Voll-/Leererkennung, Zellbalancing und Entladestromgrenze in `VNSD_BMS_SOH_Zyklen_Entladelimit_v118_de.md`; Akkupack-Wiederherstellung, Firmware-Seite (automatische Wiederherstellungsladung, Remote-BMS-Befehle) in `VNSD_Akkupack_Wiederherstellung_Firmware_de.md`; Venus-A-BMS-1105-Analyse: `VNSA_BMS_FW_v1105_Vollanalyse_de.md` |
| `Modbus_RS485_TCP/` | Externe Modbus-Schnittstelle des Geräts (TCP **und** RS485 RTU, paralleler Stack mit gemeinsamer Descriptor-Tabelle): Verbindungsreferenz, Protokoll-Eigenheiten, Scale-Faktoren, RS485-Freischaltung, vollständige Register-Map (CSV), Register-Vermutungen aus Scan-Auswertung, dekodierte Descriptor-Tabellen (CSV) samt Versions-Diffs, Read-/Write-Handler-Register-Maps in `Scan_Logs/` (die rohen Scan-Logs selbst liegen nur lokal), Abgleich der BLE-Leistungswerte gegen Modbus (Lasttest 600 W), Firmware-Fehler im Read-Serializer (dt. Analyse + engl. Herstellermeldung), Write-Register-Block 46500-46544, Registeraufloesung (Funktionsstatus je Register); vollständige, codeverifizierte Registerreferenz (alle Register mit Status, Ursprung/Ziel und Integrationshinweisen) in `VNSD_Register_Referenz_de.md`; Venus-A-Registerkarte und -Referenz (alle Register, Vergleich mit D): `VNSA_Register_Referenz_de.md` |
| `CAN_Bus/` | Interner CAN-Bus zwischen den MCUs: Aufbau der Arbitration-ID, Zielklassen (Wechselrichter/BMS/RS485-Pfad), vollständige Funktionscode-Tabellen aller 46 Sendestellen des Control gegen die Empfänger-Dispatcher von Micro v116 und BMS v118, Abdeckungslücken |
| `BLE/` | Bluetooth-LE-Schnittstelle des Control: Befehlsabbildung (v150) und Querverweis BLE ↔ Modbus-Register |
| `Methodik_und_Meta/` | Reverse-Engineering-Methodik, Ghidra-Workflow/Lessons-Learned, Firmware-Diff-Workflow, Seriennummern-Format, offene Fragen. **Über ReVa gemachte Ghidra-Änderungen speichern:** `ReVa_Aenderungen_speichern_de.md` (Speichern im Projektfenster reicht nicht) |
| `HM_HIE_FC41D/` | Kommunikationsmodul Quectel FC41D (Beken BK7231N): OTA-/Update-Analyse, Software-Stack, Versions-Diff 202409 → 202512, SoftAP/Update-Pfad, Hamedata-Anpassungen, Anleitungen für Traffic-Mitschnitt (Raspberry Pi, Laptop-AP, BLE-Update) |
| `App/` | Analyse der Marstek-Smartphone-App (`com.hamedata.marstek`): API-Recherche, Befehlsvokabular der App 1.6.76, Zuordnung der App-Befehle zur Firmware, Dekompilierungsmethode |
| `Hardware_Recherche/` | Internet-Recherchen zu Bauteilen: MCU-Identifikation (Control/BMS: Nations N32G45x, Micro: STM32G474), Teardown-/Foto-Recherche, Zulassungs- und Zertifikatsunterlagen (nur Markdown-Auswertungen, Quellen als Links) |
| `SVD/` | Hersteller-SVD-Dateien (Nations N32G451/452/455/457, Artery AT32F403A, STM32G474) zur Benennung der Peripherie-Register in Ghidra |

**Achtung Begriffsverwechslung:** "RS485" taucht in der Firmware in zwei unabhängigen Rollen auf —
(1) der **externe** RS485-RTU-Modbus-Stack (47.400 Baud, FC03/FC06/FC10, paralleler Zugang zu
denselben Registern wie Modbus TCP, in `Modbus_RS485_TCP/` dokumentiert) und (2) **interne**
RS485/UART-Verbindungen zwischen den MCUs (Control↔Micro-Inverter für Leistungssollwerte/Modi, BMS-
Master↔Slave-Packs für 96-Byte-Struct-Transfers) — ein proprietäres Struct-Protokoll, **kein Modbus**,
dokumentiert in `Control_FW/` bzw. `BMS/`.

## Nicht im Repository (nur lokal)

Folgendes gehört zum Projekt, ist aber bewusst nicht Teil dieses Repositories. Die Dokumente verweisen
an den betreffenden Stellen darauf und kennzeichnen es mit „(lokal, nicht im Repo)“:

- **Dekompilat-Exporte** (`Dekompilate/`): vollständige Dekompilate und Listings je Firmware. Sie lassen
  sich mit dem dokumentierten Ghidra-Workflow reproduzieren, siehe `Methodik_und_Meta/Dekompilat_Export_de.md`.
- **Eigene Hilfsskripte** (Python): Scan-/Monitoring-Werkzeuge für Modbus, Export-/Speicher-Skripte für
  Ghidra, Verifikations- und Diff-Hilfen.
- **Rohe Scan-Logs** (CSV/Logdateien der Modbus-Scans verschiedener Betriebszustände).
- **Fremde PDFs, Screenshots und Hersteller-Header**: Datenblätter, Handbücher, Prüfberichte,
  Teardown-Screenshots und der Hersteller-Header `n32g45x.h` (urheberrechtlich geschützt; Quellen als Links
  in `Hardware_Recherche/`).
- **Entschlüsselte Hersteller-Firmware** und Archive der Update-Images des Kommunikationsmoduls.
- **Vertrauliches Sicherheitsmaterial** (`security/`), siehe nächster Abschnitt.

## Wichtiger Hinweis zu Sicherheitsbefunden

Im Zuge der Analyse wurden sicherheitsrelevante Befunde gemacht (u. a. ein produktlinienübergreifend
geteiltes, hartcodiertes AWS-IoT-Client-Zertifikat samt Private Key sowie ein Firmware-Update-Pfad ohne
Bounds-Check). Diese wurden dem Hersteller per Responsible Disclosure vertraulich gemeldet und werden
**bewusst nicht in diesem Repository veröffentlicht**, bis eine Behebung vorliegt. Betroffene
Textstellen in den obigen Dokumenten sind entsprechend gekennzeichnet und verweisen auf interne,
nicht-öffentliche Dokumentation.

## Nutzung für die Home-Assistant-Integration

Relevant sind vor allem `Modbus_RS485_TCP/` (Verbindungsaufbau, Register-Map, Scale-Faktoren,
Fallstricke wie das 32-Register-Batch-Limit) sowie die Register-/Struct-Beschreibungen in `Control_FW/`
und `VNS_Micro_Inverter/`.
