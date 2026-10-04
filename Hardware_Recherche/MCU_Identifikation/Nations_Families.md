# Nations/NSING MCU families vs. firmware fingerprint (Marstek Venus D Control board)

> 🇩🇪 Deutsche Version: [Nations_Familien_de.md](Nations_Familien_de.md)

Sources: all PDFs stored locally under `scratchpad/mcu_search/` (curl from nsing.com.sg / nationstech.com).

## Summary of the core question

Question: Does any Nations family have an external memory controller (FSMC/XFMC/FEMC/EXMC)
with control registers at 0xA000_0000 and a bank at 0x6000_0000?

**Answer: Yes – but only the N32H47x/N32H48x series ("FEMC"), and this series has a
completely different peripheral layout that is NOT STM32F1-compatible.** No family satisfies
both the STM32F1-style peripheral layout (GPIOA..G at 0x4001x, AFIO 0x40010000,
CAN1/2 at 0x40006400/800, RCC PLL mask 0x083C0000, SAC at 0x40024000, UID at
0x1FFFF7F0) AND an external memory controller with base 0xA000_0000 / bank 0x6000_0000.

## Candidate 1: N32G45x series (N32G457/G455/G452/G451) — EN_UM_N32G45x_Series_User_Manual.pdf (local, not in the repo) (816 pp.)

Source: https://www.nsing.com.sg/uploads/MCUProducts/N32G457/Chip_Documentation/User_Manual/EN_UM_N32G45x_Series_User_Manual.pdf

- Memory map (p. around line 2536 ff. in the extracted text): `0xA000_0000 – 0xA000_0FFF = Reserved`,
  `0xA000_1000 – 0xA000_1FFF = QSPI Register`, `0x6000_0000 – 0x8FFF_FFFF = Reserved`.
- No hits for "XFMC", "FSMC", "FEMC", "EXMC" in the entire 816-page manual.
- GPIOA..GPIOG at 0x4001_0800..0x4001_2000, AFIO 0x4001_0000 → matches the fingerprint.
- **CONFLICT**: No external memory controller present. 0xA000_0000 is "Reserved", not
  occupied by BK1CTRL/BK1TMG-style registers. 0x6000_0000 is likewise "Reserved", no
  way to map 155 KB of external SRAM there.
- All other fingerprint points (peripheral map, RCC PLL mask, SAC, UID, flash/SRAM size,
  102-entry vector table, image base 0x08004800) match this family as per the task —
  only the external memory controller is missing entirely.

## Candidate 2: N32G4FR series — EN_UM_N32G4FR (709 pp., V3.5.0)

Source: nsing.com.sg/api/dowfilebefore?did=2698 (N32G4FR Series User Manual)

- Identical memory map to N32G45x: `0xA000_0000–0xA000_0FFF Reserved`,
  `0xA000_1000–0xA000_13FF QSPI Register`, `0x6000_0000–0x8FFF_FFFF Reserved`.
- GPIOA base 0x4001_0800, AFIO base 0x4001_0000 (lines 7056/7359 in the text) → identical to N32G45x.
- No hits for FEMC/XFMC/FSMC/EXMC.
- **CONFLICT**: Same problem as N32G45x – no external memory controller.

## Candidate 3: N32WB452 (BLE5.0 dual-core M4F+M0) — EN_DS_N32WB452_Series_Datasheet.pdf (local, not in the repo) (89 pp.)

Source: https://www.nsing.com.sg/uploads/WirelessRF/N32WB452/Chip_Documentation/Datasheet/EN_DS_N32WB452_Series_Datasheet.pdf

- No hits for FEMC/XFMC/FSMC/EXMC, no hits for 0x6000/0xA000 addresses in the datasheet.
- According to boot/PLL guides, shares the same peripheral base as N32G45x/N32G4FR.
- **CONFLICT**: No indication of an external memory controller.

## Candidate 4: N32H47x/N32H48x series (N32H474/N32H473) — datasheet (131 pp.) + user manual (1880 pp., combined for H47x/H48x)

Sources: nsing.com.sg/api/dowfilebefore?did=2917 (datasheet), did=5194 (user manual)

- **Does actually have a "Flexible External Memory Controller (FEMC)".**
  Datasheet memory map: `0xA000_2000–0xA000_2FFF FEMC Register`, `0xA000_1000–0xA000_1FFF
  XSPI Register`, `0x6000_0000–0x63FF_FFFF FEMC Bank1 NOR/PSRAM1` (and further banks up to
  0x8FFF_FFFF for NAND).
- User manual register chapter (line 86048 ff.): **"FEMC Base Address: 0xA000 0000"**, first
  register is `FEMC_SNCTRL1` at offset 000h (= 0xA000_0000), followed by `FEMC_SNTCFG1`
  (timing, 004h) and `FEMC_SNWTCFG1` (write timing, 008h) – exactly the sought pattern
  "bank1 chip-select control/timing registers", structurally like STM32F1 FSMC BCR1/BTR1 or
  AT32 XMC BK1CTRL1/BK1TMG1.
  → This is the only hit for the external memory fingerprint among all Nations families
  examined.
- **BUT: the remaining peripheral layout is incompatible with the fingerprint:**
  - GPIOA base = 0x4002_C000, GPIOG base = 0x4002_D800 (not 0x4001_0800…0x4001_2000).
  - CAN controllers are FDCAN1 (0x4000_5000) / FDCAN2 (0x4000_5400), not CAN1/CAN2 at
    0x4000_6400/0x4000_6800.
  - 0x4002_4000 is "Reserved" in the N32H474 memory map, no SAC crypto engine at this address.
  - The RCC PLL field is a contiguous `PLLMULFCT[5:0]` (bits 3–8) in a single register,
    not the split N32G45x mask 0x083C0000 (bits 18–21 + bit 27).
  - AFIO base 0x4001_0800 (not 0x4001_0000 as in the fingerprint).
  → Several CONFLICT points against the STM32F1-style peripheral layout that is evidenced in the rest of the
  firmware.

## Candidate 5/6: N32G43x, N32L43x/L40x — excluded (capacity)

- N32G43x (N32G430/432/435): max. 128 KB flash, 32 KB SRAM.
- N32L40x: max. 128 KB flash, 24 KB SRAM. N32L43x: max. 128 KB flash, 32 KB SRAM.
- The fingerprint requires ≥400 KB flash and ≥144 KB SRAM → these families are ruled out solely because of
  flash/RAM size, independent of the external memory controller. (Sources: web search
  summaries of nationstech.com/sekorm product briefs, no full PDFs
  downloaded, since capacity already disqualifies them.)

## Conclusion

1. **N32G457/G455/G452/G451 (N32G45x series) and N32G4FR** match exactly ALL fingerprint points
   except the external memory controller: for them, 0xA000_0000–0xA000_0FFF and
   0x6000_0000–0x8FFF_FFFF are "Reserved" according to the manual, no FSMC/FEMC/XFMC unit exists.
2. **N32H47x/N32H48x** is the only Nations family examined with an external memory
   controller (FEMC) exactly at 0xA000_0000 (bank1 register layout SNCTRL1/SNTCFG1/SNWTCFG1) and
   a memory bank exactly at 0x6000_0000 – this fully resolves the CONFLICT point of the task.
   But: GPIO, CAN and SAC base addresses as well as the RCC PLL bit field differ
   fundamentally from the rest of the fingerprint (STM32F1 style), so this series does NOT meet the other
   6 fingerprint points.
3. Among the eleven Nations/NSING Cortex-M4 families examined there is **not a single candidate
   that meets ALL fingerprint points at once (STM32F1 peripheral map + external memory controller
   at 0xA000_0000/0x6000_0000).** The contradictory combination from the task remains
   unresolved within the current Nations product range. Conceivable explanations (not verified):
   a not yet publicly/fully documented N32G45x derivative with an FSMC-like
   additional block, a customer-specific ASIC/a special version for Marstek, or the external-
   memory usage in the firmware code stems from reused reference code (e.g. from an
   AT32/STM32 reference design) that may never have actually worked on the N32G45x/N32G4FR board
   or was only prepared for a non-production board.
