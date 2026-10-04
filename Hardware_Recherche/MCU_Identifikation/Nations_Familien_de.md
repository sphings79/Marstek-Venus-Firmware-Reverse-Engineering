# Nations/NSING MCU-Familien vs. Firmware-Fingerprint (Marstek Venus D Control-Board)

> 🇬🇧 English version: [Nations_Families.md](Nations_Families.md)

Quellen: alle PDFs lokal unter `scratchpad/mcu_search/` gespeichert (curl von nsing.com.sg / nationstech.com).

## Zusammenfassung der Kernfrage

Frage: Hat irgendeine Nations-Familie einen externen Speicher-Controller (FSMC/XFMC/FEMC/EXMC)
mit Steuerregistern bei 0xA000_0000 und einer Bank bei 0x6000_0000?

**Antwort: Ja – aber nur die N32H47x/N32H48x-Serie ("FEMC"), und diese Serie hat ein
komplett anderes, NICHT STM32F1-kompatibles Peripherie-Layout.** Keine Familie erfüllt
gleichzeitig das STM32F1-artige Peripherie-Layout (GPIOA..G bei 0x4001x, AFIO 0x40010000,
CAN1/2 bei 0x40006400/800, RCC-PLL-Maske 0x083C0000, SAC bei 0x40024000, UID bei
0x1FFFF7F0) UND einen externen Speicher-Controller mit Basis 0xA000_0000 / Bank 0x6000_0000.

## Kandidat 1: N32G45x-Serie (N32G457/G455/G452/G451) — EN_UM_N32G45x_Series_User_Manual.pdf (lokal, nicht im Repo) (816 S.)

Quelle: https://www.nsing.com.sg/uploads/MCUProducts/N32G457/Chip_Documentation/User_Manual/EN_UM_N32G45x_Series_User_Manual.pdf

- Speicherkarte (S. um Zeile 2536 ff. im extrahierten Text): `0xA000_0000 – 0xA000_0FFF = Reserved`,
  `0xA000_1000 – 0xA000_1FFF = QSPI Register`, `0x6000_0000 – 0x8FFF_FFFF = Reserved`.
- Keine Treffer für „XFMC", „FSMC", „FEMC", „EXMC" im gesamten 816-Seiten-Handbuch.
- GPIOA..GPIOG bei 0x4001_0800..0x4001_2000, AFIO 0x4001_0000 → passt zum Fingerprint.
- **CONFLICT**: Kein externer Speicher-Controller vorhanden. 0xA000_0000 ist "Reserved", nicht
  belegt mit BK1CTRL/BK1TMG-artigen Registern. 0x6000_0000 ist ebenfalls "Reserved", keine
  Möglichkeit, dort 155 KB externes SRAM zu mappen.
- Alle anderen Fingerprint-Punkte (Peripherie-Map, RCC-PLL-Maske, SAC, UID, Flash/SRAM-Größe,
  102-Einträge-Vektortabelle, Image-Base 0x08004800) passen zu dieser Familie laut Auftrag –
  nur der externe Speicher-Controller fehlt komplett.

## Kandidat 2: N32G4FR-Serie — EN_UM_N32G4FR (709 S., V3.5.0)

Quelle: nsing.com.sg/api/dowfilebefore?did=2698 (N32G4FR Series User Manual)

- Identische Speicherkarte wie N32G45x: `0xA000_0000–0xA000_0FFF Reserved`,
  `0xA000_1000–0xA000_13FF QSPI Register`, `0x6000_0000–0x8FFF_FFFF Reserved`.
- GPIOA base 0x4001_0800, AFIO base 0x4001_0000 (Zeilen 7056/7359 im Text) → identisch zu N32G45x.
- Keine Treffer für FEMC/XFMC/FSMC/EXMC.
- **CONFLICT**: Gleiches Problem wie N32G45x – kein externer Speicher-Controller.

## Kandidat 3: N32WB452 (BLE5.0 Dual-Core M4F+M0) — EN_DS_N32WB452_Series_Datasheet.pdf (lokal, nicht im Repo) (89 S.)

Quelle: https://www.nsing.com.sg/uploads/WirelessRF/N32WB452/Chip_Documentation/Datasheet/EN_DS_N32WB452_Series_Datasheet.pdf

- Keine Treffer für FEMC/XFMC/FSMC/EXMC, keine Treffer für 0x6000/0xA000-Adressen im Datasheet.
- Teilt laut Boot-/PLL-Guides die gleiche Peripherie-Basis wie N32G45x/N32G4FR.
- **CONFLICT**: Kein Hinweis auf externen Speicher-Controller.

## Kandidat 4: N32H47x/N32H48x-Serie (N32H474/N32H473) — Datasheet (131 S.) + User Manual (1880 S., kombiniert für H47x/H48x)

Quellen: nsing.com.sg/api/dowfilebefore?did=2917 (Datasheet), did=5194 (User Manual)

- **Hat tatsächlich einen "Flexible External Memory Controller (FEMC)".**
  Datasheet-Speicherkarte: `0xA000_2000–0xA000_2FFF FEMC Register`, `0xA000_1000–0xA000_1FFF
  XSPI Register`, `0x6000_0000–0x63FF_FFFF FEMC Bank1 NOR/PSRAM1` (und weitere Bänke bis
  0x8FFF_FFFF für NAND).
- User-Manual-Registerkapitel (Zeile 86048 ff.): **„FEMC Base Address: 0xA000 0000"**, erstes
  Register ist `FEMC_SNCTRL1` bei Offset 000h (= 0xA000_0000), gefolgt von `FEMC_SNTCFG1`
  (Timing, 004h) und `FEMC_SNWTCFG1` (Write-Timing, 008h) – exakt das gesuchte Muster
  „Bank1-Chipselect-Control-/Timing-Register", strukturell wie STM32F1-FSMC BCR1/BTR1 bzw.
  AT32 XMC BK1CTRL1/BK1TMG1.
  → Das ist der einzige Treffer für den externen-Speicher-Fingerprint unter allen geprüften
  Nations-Familien.
- **ABER: Restliches Peripherie-Layout ist inkompatibel zum Fingerprint:**
  - GPIOA base = 0x4002_C000, GPIOG base = 0x4002_D800 (nicht 0x4001_0800…0x4001_2000).
  - CAN-Controller sind FDCAN1 (0x4000_5000) / FDCAN2 (0x4000_5400), nicht CAN1/CAN2 bei
    0x4000_6400/0x4000_6800.
  - 0x4002_4000 ist im N32H474-Memory-Map „Reserved", keine SAC-Krypto-Engine an dieser Adresse.
  - RCC-PLL-Feld ist ein zusammenhängendes `PLLMULFCT[5:0]` (Bits 3–8) in einem einzigen Register,
    nicht die gesplittete N32G45x-Maske 0x083C0000 (Bits 18–21 + Bit 27).
  - AFIO base 0x4001_0800 (nicht 0x4001_0000 wie im Fingerprint).
  → Mehrere CONFLICT-Punkte gegen das STM32F1-artige Peripherie-Layout, das im Rest der
  Firmware nachgewiesen ist.

## Kandidat 5/6: N32G43x, N32L43x/L40x — ausgeschlossen (Kapazität)

- N32G43x (N32G430/432/435): max. 128 KB Flash, 32 KB SRAM.
- N32L40x: max. 128 KB Flash, 24 KB SRAM. N32L43x: max. 128 KB Flash, 32 KB SRAM.
- Fingerprint verlangt ≥400 KB Flash und ≥144 KB SRAM → diese Familien scheiden allein wegen
  Flash-/RAM-Größe aus, unabhängig vom externen Speicher-Controller. (Quellen: WebSearch-
  Zusammenfassungen von nationstech.com/sekorm-Produktbriefs, keine vollständigen PDFs
  heruntergeladen, da Kapazität bereits disqualifiziert.)

## Fazit

1. **N32G457/G455/G452/G451 (N32G45x-Serie) und N32G4FR** passen exakt auf ALLE Fingerprint-Punkte
   außer dem externen Speicher-Controller: Bei ihnen ist 0xA000_0000–0xA000_0FFF und
   0x6000_0000–0x8FFF_FFFF laut Handbuch „Reserved", es existiert keine FSMC/FEMC/XFMC-Einheit.
2. **N32H47x/N32H48x** ist die einzige geprüfte Nations-Familie mit einem externen Speicher-
   Controller (FEMC) exakt an 0xA000_0000 (Bank1-Registerlayout SNCTRL1/SNTCFG1/SNWTCFG1) und
   einer Speicherbank exakt bei 0x6000_0000 – das löst den CONFLICT-Punkt der Aufgabe vollständig.
   Aber: GPIO-, CAN- und SAC-Basisadressen sowie das RCC-PLL-Bitfeld unterscheiden sich
   grundlegend vom Rest des Fingerprints (STM32F1-Stil), sodass diese Serie die anderen
   6 Fingerprint-Punkte NICHT erfüllt.
3. Es gibt unter den elf geprüften Nations/NSING-Cortex-M4-Familien **keinen einzigen Kandidaten,
   der gleichzeitig ALLE Fingerprint-Punkte (STM32F1-Peripherie-Map + externer Speicher-Controller
   bei 0xA000_0000/0x6000_0000) erfüllt.** Die widersprüchliche Kombination aus dem Auftrag bleibt
   ungelöst innerhalb der aktuellen Nations-Produktpalette. Denkbare Erklärungen (nicht verifiziert):
   ein noch nicht öffentlich/vollständig dokumentierter N32G45x-Ableger mit FSMC-artigem
   Zusatzblock, ein kundenspezifisches ASIC/eine Sonderversion für Marstek, oder die externe-
   Speicher-Nutzung im Firmware-Code stammt aus wiederverwendetem Referenzcode (z. B. eines
   AT32/STM32-Referenzdesigns), der auf dem N32G45x/N32G4FR-Board evtl. nie tatsächlich
   funktionsfähig war bzw. nur für ein Nicht-Serienboard vorbereitet wurde.
