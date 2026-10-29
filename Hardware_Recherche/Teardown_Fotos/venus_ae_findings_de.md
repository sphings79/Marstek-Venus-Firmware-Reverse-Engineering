# Marstek Venus A/E Teardown-Bildsuche — Ergebnisse

> 🇬🇧 English version: [venus_ae_findings_de.md](venus_ae_findings_de.md)

**Datum:** 2026-09-30  
**Suchstrategie:** Web, GitHub, Foren (DE/EN/CN), YouTube, Bilibili

## Zusammenfassung

**Keine öffentlich verfügbaren Teardown-Fotos von Marstek Venus A oder Venus E gefunden.**

> Update 2026-10-28: Eigene Fotos der Venus-A-BMS-Platine liegen jetzt vor, siehe [VNSA_BMS_Board_Fotos_de.md](../Marstek%20Venus%20A/VNSA_BMS_Board_Fotos_de.md).

### Gesuchte Quellen

| Quelle | Status | Ergebnis |
|--------|--------|----------|
| photovoltaikforum.com | Durchsucht | BMS-/Fehler-Threads ohne Fotos; ein Nutzer erwähnt Firmware-Analyse |
| akkudoktor.net | Durchsucht | 9 Threads zu Venus E/A; Diskussionen nur Text, keine Hardware-Bilder |
| mikrocontroller.net | Durchsucht | Allg. Marstek-Diskussionen, keine Teardown-Fotos |
| GitHub | Durchsucht | Firmware-RE (rweijnen), Code-Analyse, aber kein Hardware-Teardown |
| YouTube | Durchsucht | Keine Disassembly-Videos indexiert |
| Bilibili | Durchsucht | Keine Marstek Venus-Teardown-Videos |
| Reddit | Durchsucht | Keine Treffer |

---

## Findings

### 1. Hersteller-Vorgabe: Disassembly verboten
- Alle Manuals (Venus A, D, E v1–v3): „Do not dismantle the MARSTEK VENUS without authorization"
- Reparaturen nur durch autorisierte Marstek-Service
- Folgerung: Kein manufacturer-approved Teardown publiziert

### 2. Community-Ressourcen
- **Akkudoktor, Photovoltaikforum:** Fehlerberichte (BMS-Fehler, FW-Versionen), aber keine Innenansichten
- **Ein Nutzer erwähnt Eigenöffnung** (Photovoltaikforum): „die Speicherbox zu öffnen, die Platine herauszunehmen…" — aber **keine Fotos geteilt**

### 3. Firmware-Reverse-Engineering
| Repo | Inhalt | Hardware-Bilder |
|------|--------|-----------------|
| rweijnen/marstek-venus-e-firmware-notes | Firmware-Analyse, Code, extracted/ | shell.png (RS485-Interface nur) |
| sphings79/Marstek-Venus-Firmware-Reverse-Engineering | Ghidra-Analyse, Modbus-Register | Keine Hardware-Fotos |
| sphings79/marstek-ble-control | BLE-Steuerpanel | Keine Venus-Teardown-Bilder |

### 4. Hardware-Chips (aus Firmware-RE bekannt)
- **Control:** STM32F3 (oder vergleichbar)
- **BMS:** Version 117–148 (BMS-Firmware bekannt, Hardware-Markings nicht öffentlich)
- **WiFi/BLE:** Quectel FC41D (erwähnt in Firmware)
- **Ethernet:** CH395 (Photovoltaikforum-Beitrag erwähnt Boot-Loop)

---

## Fazit

**Für Hardware-Bilder (Platinen, Chip-Markierungen):**
1. Defekte Unit lokal öffnen + fotografieren (z. B. RMA-Geräte)
2. Direkt Marstek/Distributor kontaktieren um Service-Manual

**Kein öffentlicher Teardown online verfügbar.**

---

## Suchquellen (Web-Archive)
- https://github.com/rweijnen/marstek-venus-e-firmware-notes
- https://github.com/sphings79/Marstek-Venus-Firmware-Reverse-Engineering
- https://www.photovoltaikforum.com/board/247-marstek-venus-series/
- https://akkudoktor.net/t/marstek-venus-a-erfahrungen/35044
- https://www.manualslib.com/manual/4100411/Marstek-Venus-E.html
