# Marstek Venus D Teardown-Foto-Recherche – Ergebnisse

> 🇬🇧 English version: [venus_d_findings.md](venus_d_findings.md)

**Suchumfang:** englische, deutsche, chinesische Quellen (photovoltaikforum, akkudoktor, reddit, GitHub, YouTube, Bilibili, Zhihu, Reparaturforen, 1688)
**Suchdatum:** 2026-09-30
**Update:** 2026-10-26 – eigene Fotos eines geöffneten Venus D ergänzt; MCU-Tabelle korrigiert
**Status:** Keine *öffentlichen* Teardown-Fotos eines geöffneten Venus D gefunden. Eigene Fotos liegen vor (siehe unten).

## Eigene Fotos

Fotos eines geöffneten Venus D liegen in [`../Marstek Venus D/`](../Marstek%20Venus%20D/VNSD_Teardown_Fotos_de.md).
Die Leistungsplatine ist komplett vergossen; IC-Aufdrucke sind nicht lesbar. Sichtbar sind Stecker (J26 mit `GND/485B/485A/5V`),
Relais (Hongfa HF140FF-G, HF115F, HF118F), Elkos, der Funkmodul-Aufdruck `YDCFH0CA2AA_WIFI_X1`
und Kabelbeschriftungen (`通讯线` = „Kommunikationsleitung“). Details: [VNSD_Teardown_Fotos_de.md](../Marstek%20Venus%20D/VNSD_Teardown_Fotos_de.md).

## Kernbefund (öffentliche Quellen)

Marstek verbietet in allen Benutzerhandbüchern ausdrücklich das eigenmächtige Zerlegen. Das erklärt vermutlich, warum es keine öffentliche
Teardown-Dokumentation gibt. Siehe: [Marstek Venus D User Manual](https://www.marstek.nl/wp-content/uploads/2025/12/Marstek-Venus-D-user-manual.pdf)

## Interne Bauteile (aus der Firmware-Analyse)

| Bauteil | Details | Quelle |
|-----------|---------|--------|
| **MCU – Control/EMS** | Nations N32G45x mit XFMC (externer Speicher-Controller) | Firmware-Analyse, siehe [MCU-Identifikation](../MCU_Identifikation/Control_MCU_Identifikation_de.md) |
| **MCU – BMS** | Nations N32G45x | Firmware-Analyse (ADC bei 0x40020800, UID bei 0x1FFFF7F0) |
| **MCU – Micro/VNS** | STMicroelectronics STM32G474 | Firmware-Analyse (HRTIM, FDCAN, CCM-SRAM) |
| **WLAN-/BLE-Modul** | Quectel FC41D (Beken BK7231N) | Entschlüsseltes Update-Image; die Fotos zeigen den Modul-Aufdruck `YDCFH0CA2AA_WIFI_X1` (Zusammenhang mit dem FC41D nicht geprüft) |
| **Ethernet-IC** | CH395 | Control-Firmware (siehe Haupt-README) |
| **Batterie-Management-IC** | KA495XX BMIC (Zellüberwachung) | BMS-Firmware (Task `RTOS_vTaskKA495XX`) |
| **Interner Bus** | CAN + RS485 zwischen den MCUs | Firmware-Analyse; J26 in den Fotos führt RS485 + 5 V |

Die frühere Fassung dieser Datei nannte für alle drei MCUs „STM32F3 / Cortex-M4F“. Das war eine ungeprüfte Annahme und ist falsch.

## Durchsuchte Quellen (keine Fotos gefunden)

| URL | Board-Typ | Hinweise |
|-----|-----------|-------|
| photovoltaikforum.com/thread/247315 | Venus-D-Diskussion | Mehrere Threads; nur Installations-/Einrichtungsfotos, keine geöffneten Geräte |
| akkudoktor.net | Diverse Venus-D-Probleme | Fehlersuche; keine Innenfotos |
| GitHub: sphings79/Marstek-Venus-Firmware-Reverse-Engineering | n. a. – nur Firmware | Register-/Protokoll-Zuordnung; keine Hardware-Fotos |
| YouTube (4 Unboxing-/Testvideos) | Venus-D-Einrichtung | Verpackung und Außenansichten; keine Zerlegung |
| Heise.de Marstek-Venus-D-Test | Produkttest | Nur Außenfotos und App-Screenshots |
| Marstek-A1000-Teardown (chongdiantou.com) | Powerstation | Anderes Produkt; zeigt Marsteks Bauteilauswahl, aber nicht Venus D |

## Weitere technische Quellen

- **Firmware-Reverse-Engineering-Repo:** [sphings79/Marstek-Venus-Firmware-Reverse-Engineering](https://github.com/sphings79/Marstek-Venus-Firmware-Reverse-Engineering)
- **Community-Monitor-Platine:** [stdioGangsta/MarstekMonitor](https://github.com/stdioGangsta/MarstekMonitor) (KiCAD-Dateien für ein externes Überwachungsgerät, keine Venus-D-Interna)
- **Modbus-Integration:** [ViperRNMC/marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus)

## Fazit

Öffentliche Teardown-Fotos des Venus D gibt es nicht. Bauteilangaben stammen aus der Firmware-Analyse und aus den eigenen Fotos,
die eine komplett vergossene Platine zeigen und deshalb nur Stecker, Relais und Elkos belegen.
