# Hardware-Recherche Marstek Venus (Stand 2026-09-30)

> 🇬🇧 English version: [README.md](README.md)

Die Original-PDFs und Screenshots sind urheberrechtlich geschützt und liegen nur lokal; die Quellen-Links stehen in den Markdown-Dateien.

Sammlung der Internet-Recherchen zu Bauteilen, Zulassungsunterlagen und Innenfotos. Die Firmware-Analysen selbst
liegen in `Control_FW/`, `VNS_Micro_Inverter/`, `BMS/`.

## MCU-Identifikation (`MCU_Identifikation/`)

| Board | MCU | Belege |
|---|---|---|
| Control/EMS | **Nations N32G45x mit XFMC** | QSPI-Statusbits bei 0xA0001028, RCC-PLL-Bits 18–21 + 27, GPIOF/G, SAC bei 0x40024000, UID bei 0x1FFFF7F0; externer Speicher-Controller bei 0xA0000000 mit SRAM ab 0x60000000 (Shell-Befehl `xfmc_test`). Der offizielle Nations-Header `n32g45x.h` (Kopie lokal, nicht im Repo) definiert `XFMC_REG_BASE 0xA0000000` und `QSPI_BASE = XFMC_REG_BASE + 0x1000`; die Nations-Treiberbibliothek enthält `n32g45x_xfmc.c`. Das aktuelle englische User Manual markiert 0x6000_0000/0xA000_0000 dagegen als „Reserved“. |
| BMS | **Nations N32G45x** | ADC1/ADC2 bei 0x40020800/0x40020C00, UID bei 0x1FFFF7F0, 102 Vektoren |
| Micro/VNS | STMicroelectronics STM32G474 | HRTIM, FDCAN, CCM-SRAM, Flash-Size-Register (siehe Micro-Doku) |
| Kommunikationsmodul | Quectel FC41D (Beken BK7231N) | entschlüsseltes Update-Image, Strings `beken378/…` |

- `Hardware_Recherche/MCU_Identifikation/Control_MCU_Identifikation_de.md` – bitgenauer Abgleich Control-Firmware ↔ N32G457/AT32F403A.
- `Hardware_Recherche/MCU_Identifikation/Nations_Familien_de.md` – Prüfung aller Nations-Familien (N32G45x, N32G4FR, N32WB452, N32H47x …).
- `Hardware_Recherche/MCU_Identifikation/Andere_Hersteller_de.md` – Merkmalsvergleich mit Artery AT32F403A/407 und GigaDevice GD32F30x.
- Hinweis: Die beiden Recherche-Berichte gehen noch davon aus, dass der N32G45x keinen externen Speicher-Controller
  hat. Das ist durch den Header `n32g45x.h` widerlegt (siehe Tabelle).

## Innenfotos / Teardowns (`Teardown_Fotos/`)

- Öffentliche Teardown-Fotos von Venus D gibt es nicht (Suche in Foren, YouTube, Bilibili, Reddit, GitHub).
- Einzige Quelle mit Platinenfotos: YouTube-Video „Ремонт Marstek Venus A, не вмикається!“ des Kanals **AMP Service**
  (Winnyzja), https://www.youtube.com/watch?v=ujSDFebJVoE, und Instagram-Reel von `amp_service_vin`.
  Board-Aufdruck **HAME-POWER-H1200_V4.0, 2025-09-25** (Venus A). Drei QFP-Prozessoren (Aufdrucke durch Vergussmasse
  nicht lesbar), Stiftleisten J15 `3.3V/DIO/CLK/GND` (SWD) und J16 `3.3V/TX/RX/GND` (UART), eine weitere mit `GND/TMS/3.3V`.
  Im Video eingeblendete Bauteile: Infineon BSC026N08NS5 (80 V MOSFET), Silan SVS60R190 (600 V MOSFET).
- Die beiden PNG-Dateien sind Screenshots aus diesem Video.
- `Hardware_Recherche/Teardown_Fotos/*_findings.md` – Suchprotokolle der drei Such-Durchläufe (enthalten die veraltete Annahme „STM32F3“ für das Control).

## Zulassungsunterlagen (`Zertifikate/`)

- FCC: Hamedata (Grantee 2ADOW) hat nur Powerstations/Powerbanks registriert, keinen Venus.
- Gefunden (alle Venus E, Modelle MST-BIE5/2.5-2500/0800): Einheitenzertifikate VDE-AR-N 4105 (Bureau Veritas, Kiwa),
  VDE-0124-100-Prüfbericht (231 S., nur Außenfotos), IEC-62040-1-Bericht, UN38.3-Bericht (Zelle GSP50160119F,
  3,2 V/100 Ah, 16S), IP65-Bericht, WEEE-Recyclingtest (Innenfotos ohne lesbare Bauteile).
- `germany_grid_shopify.pdf` und `venus-e3_vde0124_reichelt.pdf` (beide lokal, nicht im Repo) sind dieselbe Datei (gleiche Größe, zwei Quellen).
- Keines der Dokumente nennt Bauteil-Typen (MCU, AFE, Funkmodul).
- Details und URLs: `Hardware_Recherche/Zertifikate/Zertifikate_Register_Recherche_de.md`.
