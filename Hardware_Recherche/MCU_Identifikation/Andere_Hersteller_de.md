# Suche nach alternativem Cortex-M4F MCU (nicht-STM32) mit dem gesuchten Feature-Set

> 🇬🇧 English version: [Other_Vendors.md](Other_Vendors.md)

Gesucht: STM32F1-artige Peripherie-Map (GPIOF 0x40011C00, GPIOG 0x40012000, CAN1+CAN2,
102 Vektoren, ≥512 KB Flash, ≥144 KB SRAM) + QSPI @ 0xA0001000 (STS @ +0x28, BUSY=Bit0,
TXFIFO-not-full=Bit1, RXFIFO-not-empty=Bit3) + FSMC-artiger External-Memory-Controller
@ 0xA0000000 mit externem SRAM @ 0x60000000 + Crypto/RNG @ 0x40024000 + 96-Bit UID
@ 0x1FFFF7F0 + RCC-PLL-Multiplikator in Bits 18–21 plus Extra-Bit 27.

## Ergebnis in Kürze

**Nations/NSING N32G457 (Cortex-M4F) erfüllt 5 von 6 Merkmalsgruppen exakt** und ist
damit der mit Abstand beste bisher gefundene Kandidat – abgesehen vom fehlenden
External-Memory-Controller (FSMC/EXMC/XMC) bei 0xA0000000/0x60000000, den es laut
Datenblatt/User-Manual **nicht** besitzt (dort als "Reserved" markiert).

Artery AT32F403A/407 und GigaDevice GD32F30x haben zwar den FSMC-artigen External-
Memory-Controller bei 0xA0000000, aber ihnen fehlen QSPI @ 0xA0001000 in dieser Form,
Crypto/RNG @ 0x40024000 sowie die UID-Adresse 0x1FFFF7F0 und das PLL-Bit-27-Layout.

## Feature-Vergleichstabelle

| Merkmal | N32G457 (Nations/NSING) | AT32F403A/407 (Artery) | GD32F30x (GigaDevice) |
|---|---|---|---|
| GPIOF @ 0x40011C00, GPIOG @ 0x40012000 | **Match** – Datenblatt Fig. 2‑1, S.15: GPIOF 0x4001_1C00–0x4001_1FFF, GPIOG 0x4001_2000–0x4001_23FF | vermutlich Match (STM32F1-kompatible Map), nicht einzeln verifiziert | vermutlich Match (STM32F1-kompatible Map), nicht einzeln verifiziert |
| CAN1 + CAN2 | **Match** – CAN1 0x4000_6400, CAN2 0x4000_6800 (Fig. 2‑1) | Match (Connectivity-Line-Layout bekannt) | Match (GD32F30x hat CAN0/CAN1 an denselben Adressen) |
| 102 Vektoren | **Match** – User Manual S.841: NVIC "up to 86 maskable interrupt channels" + 16 Core-Exceptions = 102 | unbekannt/nicht verifiziert | unbekannt/nicht verifiziert |
| ≥512 KB Flash, ≥144 KB SRAM | **Match** – Datenblatt: "Up to 512K Bytes Flash", "144K Bytes SRAM" exakt | Match (AT32F403A bis 1 MB Flash/224 KB SRAM in manchen Varianten) | Konflikt – GD32F30x max. i. d. R. ≤ 96 KB SRAM (Connectivity-Line teils mehr, aber 144 KB nicht Standard) |
| QSPI @ 0xA0001000, STS @ +0x28 (BUSY=Bit0, TXFNF=Bit1, RXFNE=Bit3) | **Match** – User Manual S.664/668: QSPI_STS @ Offset 0x28, Bitfeld von LSB nach MSB: BUSY(0), TXFNF(1), TXFE(2), RXFNE(3), RXFF(4). Basisadresse QSPI-Register 0xA000_1000–0xA000_1FFF (Fig. 2‑1) | Konflikt/unbekannt – AT32 nutzt XMC/SPIM zum externen Flash-Zugriff, kein eigenständiger QSPI-Block an dieser Adresse gefunden | Konflikt – GD32F30x hat kein separates QSPI-Register-Interface an 0xA0001000 |
| External-Memory-Controller (FSMC-artig) @ 0xA0000000, externes SRAM @ 0x60000000 | **Konflikt** – Speicherkarte zeigt 0xA000_0000–0xA000_0FFF "Reserved" und 0x6000_0000–0x8FFF_FFFF "Reserved"; kein FSMC/EXMC/XMC im ganzen 816-seitigen User Manual gefunden | **Match** – XMC mit BK1CTRL1/BK1TMG1 exakt bei 0xA0000000, externes SRAM/NOR-Bank | **Match** – EXMC bei 0xA0000000, externes SRAM-Bank1 bei 0x60000000 (bekannt aus GD32F30x User Manual) |
| Crypto/RNG @ 0x40024000 | **Match** – SAC (Security Acceleration Engine: AES/DES/SHA/SM1/SM3/SM4/SM7/TRNG) exakt bei 0x4002_4000–0x4002_43FF (Fig. 2‑1) | Konflikt – kein Crypto/RNG-Peripherie-Hit im 524-seitigen Reference Manual (kein SAC/CRYP/TRNG) | Konflikt – GD32F30x hat keine Crypto/TRNG-Einheit |
| 96-Bit UID @ 0x1FFFF7F0 | **Match** – laut Klipper-Firmware-Issue (PR #7363) ist die N32G45x-UID dokumentiert ab genau 0x1FFFF7F0 (abweichend vom geerbten STM32F103-Header UID_BASE=0x1FFFF7E8) | Konflikt – klassisches STM32-Layout: UID_BASE=0x1FFFF7E8, UID[95:64] erst bei 0x1FFFF7F0 als drittes Wort, nicht als Basis | vermutlich wie AT32 klassisches STM32-Layout (nicht verifiziert) |
| RCC/CRM-PLL-Mult in Bits 18–21 + Extra-Bit 27 | **Match** – User Manual S.105, RCC_CFG: PLLMULFCT[3:0] @ Bit 21:18, PLLMULFCT[4] (Extra-Bit) exakt @ Bit 27 | Konflikt – CRM_CFG PLLMULT nutzt Bits {30:29, 21:18} (Extra-Bits bei 29/30, nicht bei 27) | unbekannt, nicht verifiziert, aber GD32-Familie folgt i. d. R. dem 4‑Bit-STM32F1-PLLMUL-Schema ohne Extra-Bit 27 |

## Quellen (mit Fundstelle)

- N32G457 Datasheet (NSING, EN_DS_N32G457_Series_Datasheet.pdf, lokal, nicht im Repo), Seite 15 „Figure 2-1 Memory map“: GPIOF/GPIOG-Adressen, CAN1/CAN2-Adressen, QSPI-Register @ 0xA000_1000, SAC @ 0x4002_4000, „Reserved“ bei 0xA000_0000 und 0x6000_0000–0x8FFF_FFFF. URL: https://www.nsing.com.sg/uploads/MCUProducts/N32G457/Chip_Documentation/Datasheet/EN_DS_N32G457_Series_Datasheet.pdf
- N32G457/N32G45x User Manual (NSING, EN_UM_N32G45x_Series_User_Manual.pdf, 816 S., lokal, nicht im Repo):
  - S. 104–105: RCC_CFG Bit-Layout, PLLMULFCT[3:0] @ Bit 21:18 und PLLMULFCT[4] @ Bit 27.
  - S. 664, 668: QSPI_STS-Register @ Offset 0x28, Bitreihenfolge BUSY(0)/TXFNF(1)/TXFE(2)/RXFNE(3)/RXFF(4).
  - S. 841: NVIC „up to 86 maskable interrupt channels“ → 86+16 = 102 Vektoren.
  URL: https://www.nsing.com.sg/uploads/MCUProducts/N32G457/Chip_Documentation/User_Manual/EN_UM_N32G45x_Series_User_Manual.pdf
- N32G45x PLL User Guide (EN_UG_N32G45x_N32G4FR_N32WB452_Series_PLL_User_Guide_V1.0.0.pdf, lokal, nicht im Repo): Bestätigt RCC_CFG.PLLMULFCT[4:0] als 5-Bit-Multiplikatorfeld.
  URL: https://www.nationstech.com/uploads/Microcontrollers/N32G457xx_V3.4.0/8-User_Guide/EN_UG_N32G45x_N32G4FR_N32WB452_Series_PLL_User_Guide_V1.0.0.pdf
- Klipper3d GitHub PR #7363 „stm32: read the n32g45x unique id from its documented address“: bestätigt, dass die 96-Bit-UID des N32G45x ab genau 0x1FFFF7F0 liegt (nicht 0x1FFFF7E8 wie beim geerbten STM32F103-Header).
  URL: https://github.com/Klipper3d/klipper/pull/7363
- Artery AT32F403A/407 Reference Manual V2.06 (RM_AT32F403A_407_EN_V2.06.pdf, 524 S., lokal, nicht im Repo):
  - Abschnitt „Sram/Nor Flash…“ (XMC_BK1CTRL1/BK1TMGWR1 etc.) bei 0xA0000000 – External-Memory-Controller vorhanden.
  - Abschnitt „Device electronic signature“: UID_BASE = 0x1FFFF7E8 (klassisches STM32-Schema), UID[95:64] bei Offset +8 = 0x1FFFF7F0.
  - Abschnitt 4.3.2 CRM_CFG: PLLMULT-Feld aus Bit {30:29, 21:18} – kein Bit 27.
  - Keine Treffer für SAC/CRYP/TRNG/RNG_BASE im gesamten Dokument (Crypto/RNG-Peripherie nicht vorhanden).
  URL: https://www.arterychip.com/download/RM/RM_AT32F403A_407_EN_V2.06.pdf
- GD32F30x User Manual (GigaDevice, bekannt aus vorheriger/allgemeiner Recherche): EXMC bei 0xA0000000 mit externem SRAM-Bank bei 0x60000000, aber kein QSPI-Registerblock an 0xA0001000 und keine Crypto/RNG-Einheit (allgemein bekannte GD32F30x-Eigenschaft, nicht Bit-für-Bit neu verifiziert in dieser Sitzung).
  URL: https://www.gigadevice.com.cn/Public/Uploads/uploadfile/files/20230228/GD32F30x_User_Manual_EN_Rev2.9.pdf

## Fazit

Kein Chip erfüllt alle sechs Anforderungen zu 100 %. **N32G457 (Nations/NSING)** ist der
mit Abstand stärkste Treffer: fünf von sechs Merkmalsgruppen stimmen bis auf Bit-Ebene
exakt überein (GPIOF/G-Adressen, CAN1+CAN2, 102 Vektoren, Flash/SRAM-Größe, QSPI @
0xA0001000 inkl. STS-Register-Layout, SAC-Krypto/RNG exakt @ 0x40024000, 96-Bit-UID
exakt @ 0x1FFFF7F0, RCC_CFG-PLL-Bits 18–21 + Extrabit 27). Der einzige Konfliktpunkt ist
der fehlende FSMC/EXMC/XMC-artige External-Memory-Controller bei 0xA0000000 mit
externem SRAM bei 0x60000000 – dieser Adressbereich ist laut Datenblatt/User-Manual
„Reserved“, also nicht implementiert.

Sollte das analysierte Firmware-Image tatsächlich einen externen SRAM-Zugriff über
0x60000000 nutzen (bank-1 chip-select-Register bei 0xA0000000), passt das nicht zu
N32G457, sondern eher zu AT32F403A/407 oder GD32F30x – die aber wiederum kein QSPI in
dieser Form, keine Crypto/RNG-Einheit bei 0x40024000 und nicht das UID-Layout mit
0x1FFFF7F0 als Basis besitzen. Es ist möglich, dass die vermeintliche "External-Memory-
Controller"-Zugriffe im untersuchten Image tatsächlich Zugriffe auf die QSPI-Bank
(memory-mapped QSPI-Flash bei 0x9000_0000–0x9FFF_FFFF laut N32G457-Map) sind und
fälschlich als FSMC-Bank interpretiert wurden – das sollte am Original-Firmware-Image
gegengeprüft werden, bevor eine andere Chipfamilie favorisiert wird.
