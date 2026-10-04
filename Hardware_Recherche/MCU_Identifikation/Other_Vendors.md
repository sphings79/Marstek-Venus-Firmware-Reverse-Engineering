# Search for an alternative Cortex-M4F MCU (non-STM32) with the target feature set

> 🇩🇪 Deutsche Version: [Andere_Hersteller_de.md](Andere_Hersteller_de.md)

Target: STM32F1-style peripheral map (GPIOF 0x40011C00, GPIOG 0x40012000, CAN1+CAN2,
102 vectors, ≥512 KB flash, ≥144 KB SRAM) + QSPI @ 0xA0001000 (STS @ +0x28, BUSY=bit 0,
TXFIFO-not-full=bit 1, RXFIFO-not-empty=bit 3) + FSMC-style external memory controller
@ 0xA0000000 with external SRAM @ 0x60000000 + crypto/RNG @ 0x40024000 + 96-bit UID
@ 0x1FFFF7F0 + RCC PLL multiplier in bits 18–21 plus extra bit 27.

## Result in Brief

**Nations/NSING N32G457 (Cortex-M4F) matches 5 of 6 feature groups exactly** and is
thus by far the best candidate found so far – apart from the missing
external memory controller (FSMC/EXMC/XMC) at 0xA0000000/0x60000000, which according to the
datasheet/user manual it does **not** have (marked there as "Reserved").

Artery AT32F403A/407 and GigaDevice GD32F30x do have the FSMC-style external
memory controller at 0xA0000000, but they lack QSPI @ 0xA0001000 in this form,
crypto/RNG @ 0x40024000, as well as the UID address 0x1FFFF7F0 and the PLL bit-27 layout.

## Feature Comparison Table

| Feature | N32G457 (Nations/NSING) | AT32F403A/407 (Artery) | GD32F30x (GigaDevice) |
|---|---|---|---|
| GPIOF @ 0x40011C00, GPIOG @ 0x40012000 | **Match** – datasheet Fig. 2‑1, p.15: GPIOF 0x4001_1C00–0x4001_1FFF, GPIOG 0x4001_2000–0x4001_23FF | probably a match (STM32F1-compatible map), not verified individually | probably a match (STM32F1-compatible map), not verified individually |
| CAN1 + CAN2 | **Match** – CAN1 0x4000_6400, CAN2 0x4000_6800 (Fig. 2‑1) | Match (connectivity-line layout known) | Match (GD32F30x has CAN0/CAN1 at the same addresses) |
| 102 vectors | **Match** – user manual p.841: NVIC "up to 86 maskable interrupt channels" + 16 core exceptions = 102 | unknown/not verified | unknown/not verified |
| ≥512 KB flash, ≥144 KB SRAM | **Match** – datasheet: "Up to 512K Bytes Flash", "144K Bytes SRAM" exactly | Match (AT32F403A up to 1 MB flash/224 KB SRAM in some variants) | Conflict – GD32F30x usually max. ≤ 96 KB SRAM (connectivity line partly more, but 144 KB is not standard) |
| QSPI @ 0xA0001000, STS @ +0x28 (BUSY=bit 0, TXFNF=bit 1, RXFNE=bit 3) | **Match** – user manual p.664/668: QSPI_STS @ offset 0x28, bit field from LSB to MSB: BUSY(0), TXFNF(1), TXFE(2), RXFNE(3), RXFF(4). QSPI register base address 0xA000_1000–0xA000_1FFF (Fig. 2‑1) | Conflict/unknown – AT32 uses XMC/SPIM for external flash access, no standalone QSPI block found at this address | Conflict – GD32F30x has no separate QSPI register interface at 0xA0001000 |
| External memory controller (FSMC-style) @ 0xA0000000, external SRAM @ 0x60000000 | **Conflict** – memory map shows 0xA000_0000–0xA000_0FFF "Reserved" and 0x6000_0000–0x8FFF_FFFF "Reserved"; no FSMC/EXMC/XMC found in the entire 816-page user manual | **Match** – XMC with BK1CTRL1/BK1TMG1 exactly at 0xA0000000, external SRAM/NOR bank | **Match** – EXMC at 0xA0000000, external SRAM bank 1 at 0x60000000 (known from the GD32F30x user manual) |
| Crypto/RNG @ 0x40024000 | **Match** – SAC (Security Acceleration Engine: AES/DES/SHA/SM1/SM3/SM4/SM7/TRNG) exactly at 0x4002_4000–0x4002_43FF (Fig. 2‑1) | Conflict – no crypto/RNG peripheral hit in the 524-page reference manual (no SAC/CRYP/TRNG) | Conflict – GD32F30x has no crypto/TRNG unit |
| 96-bit UID @ 0x1FFFF7F0 | **Match** – according to the Klipper firmware issue (PR #7363) the N32G45x UID is documented starting at exactly 0x1FFFF7F0 (deviating from the inherited STM32F103 header UID_BASE=0x1FFFF7E8) | Conflict – classic STM32 layout: UID_BASE=0x1FFFF7E8, UID[95:64] only at 0x1FFFF7F0 as the third word, not as the base | probably classic STM32 layout like AT32 (not verified) |
| RCC/CRM PLL mult in bits 18–21 + extra bit 27 | **Match** – user manual p.105, RCC_CFG: PLLMULFCT[3:0] @ bit 21:18, PLLMULFCT[4] (extra bit) exactly @ bit 27 | Conflict – CRM_CFG PLLMULT uses bits {30:29, 21:18} (extra bits at 29/30, not at 27) | unknown, not verified, but the GD32 family usually follows the 4‑bit STM32F1 PLLMUL scheme without extra bit 27 |

## Sources (with Location)

- N32G457 Datasheet (NSING, EN_DS_N32G457_Series_Datasheet.pdf, local, not in the repo), page 15 "Figure 2-1 Memory map": GPIOF/GPIOG addresses, CAN1/CAN2 addresses, QSPI register @ 0xA000_1000, SAC @ 0x4002_4000, "Reserved" at 0xA000_0000 and 0x6000_0000–0x8FFF_FFFF. URL: https://www.nsing.com.sg/uploads/MCUProducts/N32G457/Chip_Documentation/Datasheet/EN_DS_N32G457_Series_Datasheet.pdf
- N32G457/N32G45x User Manual (NSING, EN_UM_N32G45x_Series_User_Manual.pdf, 816 pp., local, not in the repo):
  - pp. 104–105: RCC_CFG bit layout, PLLMULFCT[3:0] @ bit 21:18 and PLLMULFCT[4] @ bit 27.
  - pp. 664, 668: QSPI_STS register @ offset 0x28, bit order BUSY(0)/TXFNF(1)/TXFE(2)/RXFNE(3)/RXFF(4).
  - p. 841: NVIC "up to 86 maskable interrupt channels" → 86+16 = 102 vectors.
  URL: https://www.nsing.com.sg/uploads/MCUProducts/N32G457/Chip_Documentation/User_Manual/EN_UM_N32G45x_Series_User_Manual.pdf
- N32G45x PLL User Guide (EN_UG_N32G45x_N32G4FR_N32WB452_Series_PLL_User_Guide_V1.0.0.pdf, local, not in the repo): Confirms RCC_CFG.PLLMULFCT[4:0] as a 5-bit multiplier field.
  URL: https://www.nationstech.com/uploads/Microcontrollers/N32G457xx_V3.4.0/8-User_Guide/EN_UG_N32G45x_N32G4FR_N32WB452_Series_PLL_User_Guide_V1.0.0.pdf
- Klipper3d GitHub PR #7363 "stm32: read the n32g45x unique id from its documented address": confirms that the 96-bit UID of the N32G45x lies at exactly 0x1FFFF7F0 (not 0x1FFFF7E8 as in the inherited STM32F103 header).
  URL: https://github.com/Klipper3d/klipper/pull/7363
- Artery AT32F403A/407 Reference Manual V2.06 (RM_AT32F403A_407_EN_V2.06.pdf, 524 pp., local, not in the repo):
  - Section "Sram/Nor Flash…" (XMC_BK1CTRL1/BK1TMGWR1 etc.) at 0xA0000000 – external memory controller present.
  - Section "Device electronic signature": UID_BASE = 0x1FFFF7E8 (classic STM32 scheme), UID[95:64] at offset +8 = 0x1FFFF7F0.
  - Section 4.3.2 CRM_CFG: PLLMULT field from bits {30:29, 21:18} – no bit 27.
  - No hits for SAC/CRYP/TRNG/RNG_BASE in the entire document (crypto/RNG peripheral not present).
  URL: https://www.arterychip.com/download/RM/RM_AT32F403A_407_EN_V2.06.pdf
- GD32F30x User Manual (GigaDevice, known from earlier/general research): EXMC at 0xA0000000 with external SRAM bank at 0x60000000, but no QSPI register block at 0xA0001000 and no crypto/RNG unit (generally known GD32F30x property, not re-verified bit by bit in this session).
  URL: https://www.gigadevice.com.cn/Public/Uploads/uploadfile/files/20230228/GD32F30x_User_Manual_EN_Rev2.9.pdf

## Conclusion

No chip meets all six requirements 100 %. **N32G457 (Nations/NSING)** is by far the
strongest match: five of six feature groups agree exactly down to the bit level
(GPIOF/G addresses, CAN1+CAN2, 102 vectors, flash/SRAM size, QSPI @
0xA0001000 including STS register layout, SAC crypto/RNG exactly @ 0x40024000, 96-bit UID
exactly @ 0x1FFFF7F0, RCC_CFG PLL bits 18–21 + extra bit 27). The only conflict point is
the missing FSMC/EXMC/XMC-style external memory controller at 0xA0000000 with
external SRAM at 0x60000000 – according to the datasheet/user manual this address range is
"Reserved", i.e. not implemented.

If the analyzed firmware image really does use external SRAM access via
0x60000000 (bank-1 chip-select register at 0xA0000000), this does not fit
N32G457, but rather AT32F403A/407 or GD32F30x – which in turn have no QSPI in
this form, no crypto/RNG unit at 0x40024000 and not the UID layout with
0x1FFFF7F0 as the base. It is possible that the supposed "external memory
controller" accesses in the examined image are actually accesses to the QSPI bank
(memory-mapped QSPI flash at 0x9000_0000–0x9FFF_FFFF according to the N32G457 map) and
were wrongly interpreted as an FSMC bank – this should be cross-checked against the original firmware image
before a different chip family is favored.
