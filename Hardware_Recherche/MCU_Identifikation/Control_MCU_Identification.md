# Control (EMS) Board: AT32F403A vs. N32G45x — Offline Analysis (Control v151)

> 🇩🇪 Deutsche Version: [Control_MCU_Identifikation_de.md](Control_MCU_Identifikation_de.md)

Source: `Dekompilate/VNSD_Control_v151/` (local, not in the repo) (export, no Ghidra/ReVa live tools used).

## 1. Registers at 0xA0000000 / 0xA0001000 and RCC/CRM AHBEN

**Board_XMC_SRAM_Init @ 0x0801e094** (called from **SystemInit @ 0x080333e0**, line `Board_XMC_SRAM_Init();` — i.e. a real, executed boot path, not dead code):
- calls `RCC_AHBPeriphClockCmd(0x100, 1)` (0x0801e09a/0x0801e09c), then `Board_XMC_ConfigGPIO_AddressBus @ 0801e100`, `Board_XMC_ConfigGPIO_DataBus @ 0801e240`, `Board_XMC_ConfigGPIO_ControlBus @ 0801e288`.
- then builds a register block on the stack, `bank_regs[0]=0xa0000000` (BCR1), calls `Board_XMC_BuildControlAndTimingRegs @ 080358ce` (ORs timing fields → BCR1/BTR1/BWTR1 pattern, classic ST/AT32 FSMC/XMC model) and `Board_XMC_SetBankEnable @ 080358b8` (sets/clears only bit 0 = bank enable).

**Finding AT32F403Axx_v2.svd**: peripheral `XMC` exactly at `0xA0000000`, size `0x1000`, with registers `BK1CTRL1` (offset 0x0), `BK1TMGWR1`/`BK1TMGWR4` (0x104/0x11C) etc. — this is exactly the register layout the code describes (BCR/BTR/BWTR, bit0=enable).

**Finding N32G457.svd / N32G455.svd / N32G452.svd**: **no** peripheral entry at `0xA0000000`. The only entry above `0x90000000` is `QSPI` at `0xA0001000` (size 0x400). According to the N32G457 datasheet (p. 15, "Figure 2-1 Memory map"), `0xA000_0000–0xA000_0FFF` is explicitly **"Reserved"**; likewise `0x6000_0000–0x8FFF_FFFF` ("Reserved", no external memory controller) — the official "QSPI Bank" XIP window is instead at `0x9000_0000–0x9FFF_FFFF`.

→ The BCR/BTR/BWTR write accesses at 0xA0000000 match **AT32** (XMC) bit-exactly, and are **reserved address space on N32G45x according to the datasheet**.

**0xA0001000** is used separately (`QSPI_Flash_IsBusy @ 08017914`, `QSPI_TransferReady_Poll @ 0801792c`, `QSPI_WriteComplete_Check @ 08017944`, all via `PTR_EXTFLASH_REG_000` = 0xa0001000, status word at offset +0x28):
- Bit0 is checked as "busy", bit3 as "transfer ready", bit1 (inverted) as "write complete".

**Finding N32G457/455/452.svd**: `QSPI_STS` @ offset `0x28`, bits: `BUSY`=0, `TXFNF`=1, `RXFNE`=3, `RXFF`=4, `DC_ERR`=6 — **bit-exact match** for all three bits used (0,1,3).
**Finding AT32F403Axx_v2.svd**: **no QSPI peripheral at all** present (grep yields 0 hits) — 0xA0001000 on AT32 would be unused/undocumented address space directly behind the 0x1000-sized XMC block.

→ For 0xA0001000 **only N32G45x** is a valid interpretation at all; AT32 simply has no hardware there.

**RCC/CRM AHBEN (0x40021014)**: the call `RCC_AHBPeriphClockCmd(0x100,1)` in `Board_XMC_SRAM_Init` sets bit 8 of the AHB clock enable register — on AT32 CRM_AHBEN this is exactly `XMCEN`; on N32 RCC_AHBPCLKEN a different/no feature would have to sit there (not sensibly nameable in the present context without a documented SMC) — this point is of little significance on its own, since bit names come from the (AT32) SVD assignment.

## 2. Use of 0x60000000..0x60026AF7

The scatter/linker region `.xmc_bss` (`60000000-60026af7`, 235 referenced addresses) is used like **real, writable RAM**, not an XIP window:
- `Board_ExternalSRAM_SelfTest @ 0x08035980` (shell command **"xfmc_test"**, string literal at `0x0805b58e` present in the image, i.e. real in the firmware image, not an analysis artifact): reads/writes bytes at `0x60000000+idx`, fills with an increment pattern or 0xFF, everything verified via `printf` output — a classic external SRAM self-test.
- Further use as persistent counters/buffers (`DAT_600238c8`, JSON-RPC receive buffer from `DAT_60025518`, etc. according to `globals_and_registers.txt`).

It is thus the same external SRAM address space that `Board_XMC_SRAM_Init` configures via the BCR/BTR registers at 0xA0000000 (Bank1 NOR/SRAM window, classic ST/AT32 layout: control register @0xA000_0000, data window Bank1 @0x6000_0000). According to the N32G457 datasheet, exactly this memory range (0x6000_0000–0x8FFF_FFFF) is **"Reserved"** — there is no documented external memory controller there.

## 3. Further distinguishing peripherals

- **GPIOF/GPIOG** (0x40011C00 / 0x40012000): `Board_XMC_ConfigGPIO_AddressBus @ 0801e100` configures, according to the comment "GPIOF/GPIOG (A0-A9,A10-A15) and GPIOD (A16-A18)", AF push-pull 50 MHz pins for the address bus — **actually used, functional GPIO ports**.
  - AT32F403Axx_v2.svd: only `GPIOA..GPIOE` (0x40010800–0x40011800) present, **no GPIOF/GPIOG**.
  - N32G457.svd: `GPIOA..GPIOG` incl. `GPIOF@0x40011C00`, `GPIOG@0x40012000` present.
  - → GPIOF/G usage is **not possible** on AT32F403A (per this SVD/RM), **an exact fit** on N32G45x.

- **RCC/CRM_CFG PLL multiplier field** (`RCC_GetClocksFreq @ 0802d864`): mask `0x83c0000` = bits {18,19,20,21,27}; if bit 27 is set: `mult = field - 0x1ef` (range 17–32), otherwise `mult = field + 2` (range 2–17).
  - **N32G457.svd RCC_CFG**: `PLLMULFCT` bits 21:18 (4 bits) **+ PLLMULFCT_H bit 27** (1 bit) — **bit-exact match**, incl. exactly the same bit position 27 for the high bit.
  - **AT32F403Axx_v2.svd CRM_CFG**: `PLLMULT3_0` bits 21:18, but the extension is at **PLLMULT5_4 (bits 30:29)**, **PLLRANGE at bit 31**, bit 27 there is `USBDIV2` — does not match the code's mask.
  - The resulting multiplier range (max. ~32, with 8 MHz crystal/HSE prescaler ⇒ max. system clock ~144 MHz) matches the documented N32G457 upper limit, not AT32F403A (up to 240 MHz, where a 6-bit multiplier field with bits 29/30 would be needed).
  - → Clear, bit-exact match for **N32G45x**, counter-evidence against AT32.

- Function names of the clock drivers (`RCC_HSEConfig`, `RCC_AHBPeriphClockCmd`, `RCC_APB1PeriphClockCmd`, `RCC_APB2PeriphClockCmd`, `RCC_GetClocksFreq`, `RCC_WaitForHSEStartUp`) follow the classic ST/N32 SPL naming convention (`RCC_...`), not AT32's `crm_...` firmware library convention. Counted only as a supporting indication (names could come from a shared/ported SDK base), not as proof.

- UID readout `Board_GetUniqueID_HexString @ 0801xxxx` (see brief) reads at `0x1FFFF7F0` — matches N32G45x documentation (AT32/STM32F1: `0x1FFFF7E8`). Only reconfirmed here, not re-checked.

- No indications in the export of AT32-exclusive peripherals such as SDIO2 (0x40023400), ACC (0x40015800), UART7/8 (0x40016400/0x40016800) (no hit for these addresses in `globals_and_registers.txt`).

## 4. Conclusion

**N32G45x**, with high confidence — probably **N32G457** (or a closely related sibling model G455/G452 with an identical QSPI/RCC core; all three SVDs show the same QSPI base address/registers).

**In favor (bit-exact, primary sources):**
1. QSPI status register at `0xA0001000+0x28`: the bits used 0/1/3 match N32 `QSPI_STS` (BUSY/TXFNF/RXFNE) exactly; AT32F403A has no QSPI peripheral at all.
2. RCC_CFG PLL multiplier decoding (bits 18-21 + bit 27) matches N32G457 `PLLMULFCT`/`PLLMULFCT_H` exactly; AT32 CRM_CFG has the extension at a different bit position (29/30).
3. The firmware configures and uses GPIOF and GPIOG — not present on AT32F403A per SVD/RM, present on N32G45x.
4. UID address 0x1FFFF7F0 (from the brief) matches N32, not AT32/STM32F1.
5. Consistent with the N32G45x already proven for the BMS board of the same platform (ADC @ 0x40020800).

**Against / unresolved:**
- The complete "XMC" code path (`Board_XMC_SRAM_Init`, BCR/BTR/BWTR registers at 0xA0000000, external SRAM window 0x60000000–0x60026AF7, self-test shell command "xfmc_test") corresponds bit-exactly to AT32's XMC peripheral and is actually executed from `SystemInit` — but according to the N32G457 datasheet (p. 15) this address space is explicitly "Reserved" (no memory controller documented). Most plausible explanation: a shared/ported reference BSP (possibly originally written for an AT32 reference design and carried over unchanged into the N32 firmware); on real N32G45x silicon the code would go nowhere (reserved bus range, presumably no hard fault, but also no working external SRAM) or points to an internal test extension not documented in the public datasheet. This remains an unresolved remainder, but given the three independent bit-exact matches (QSPI, PLL, GPIOF/G) it does not change the overall assessment.

**Not conclusively clarified:** exact N32 variant (452 vs. 455 vs. 457) — this would require a reconciliation of pin count/package-specific peripherals (e.g. CAN2, USB, DAC, additional UARTs), which was not checked further within the scope of this task.
