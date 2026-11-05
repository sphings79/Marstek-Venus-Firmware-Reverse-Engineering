# VNSD-0 MPPT firmware — architecture and load address

> 🇩🇪 Deutsche Version: [VNSD_MPPT_C28x_Image_Setup_de.md](VNSD_MPPT_C28x_Image_Setup_de.md)

Result of the first analysis of the MPPT image (v104, 98304 bytes, SHA-256 `4237eb5e8216a5ac3caf6c5adaafdf6bee31ee78843a488206239991c1a4091d`). Before this, the project had no MPPT image (see [MPPT_Hardware_External_Research.md](MPPT_Hardware_External_Research.md)).

## Summary

| Property | Value | Evidence |
|---|---|---|
| Core | **TI TMS320C28x** (C2000, Piccolo class) | opcode statistics, startup code (below) |
| Exact chip | **TMS320F2803x, most likely F28035** (package/Q1 variant open; board is potted, marking not readable) | PIE table, LIN-A UART, CLA, flash size; see [VNSD_MPPT_Firmware_Findings.md](VNSD_MPPT_Firmware_Findings.md) |
| Word-addressed | yes, 1 address = 16 bits | architecture |
| Image size | 0xC000 words (48 K words) | file size / 2 |
| Load address | **0x3E8000** (word address) | entry branch + startup code (below) |
| Entry point | `0x3F0BAE` (`c_int00`) | first word of the image |

File word `i` maps to word address `0x3E8000 + i`; the byte offset in the file is `2·i`.

## How the architecture was identified

The image is not Cortex-M (no vector table, no stack pointer in the first words). Opcode statistics over the code range showed the C28x signature: 1548 × `0x761F` (`MOVW DP,#imm`), 369 × `0x0006` (`LRETR`), 699 × `0x8F00`. The `MOVW DP` operands (data pages of 64 words) point at peripheral frames typical for a Piccolo device: `0x0B00` (ADC results), `0x6800` (ePWM), `0x7000`/`0x7100` (system control / ADC), `0x7900` (I2C), `0x8000+` (RAM). This fits an MPPT converter; these frames alone do **not** identify the derivative (F2803x and F2806x have a very similar layout). The later analysis of the interrupt table, the LIN-A UART and the CLA narrowed it to the F2803x family, most likely the F28035.

## How the load address was derived

The first two words (`0x007F 0x0BAE`) decode as `LB 0x3F0BAE`. Assuming the image starts at `0x3E8000`:

- the target is file word `0x8BAE`, and the word before it is `LRETR`, i.e. the target is a function start;
- the code there is a textbook `c_int00`:

```
3F0BAE  SETC  Objmode
3F0BAF  EALLOW
3F0BB0  MOVZ  DP,#0x1c0          ; page 0x7000 (system control)
3F0BB1  MOV   @0x29,#0x68        ; WDCR = 0x68 -> watchdog disabled
3F0BB3  EDIS
3F0BB4  LB    0x3F0347           ; continue with the C runtime setup
```

- `0x3F0347` is again preceded by `LRETR` and sets up the stack and status bits (`MOV @SP,#0x50`, `SETC Objmode`, `CLRC AMODE`, `SETC M0M1MAP`, `ASP`) and then calls the startup/`main` chain.

Both branch targets only land on function starts for this base address. The base is therefore well supported, but not proven by a datasheet. The exact flash size of the chip is not known, so the image is mapped only over its own length.

## Ghidra setup

Ghidra 12.1.1 has no C28x processor. The community module [ghidra-tms320c28x](https://github.com/outlandnish/ghidra-tms320c28x) (Apache-2.0, unofficial, designed against the F28377D, still work in progress) was used.

1. Clone the repository and compile the specs with Ghidra's own compiler:
   `support/sleigh data/languages/tms320c28x.slaspec` and `support/sleigh data/languages/tms320c28x_cla.slaspec`
2. Copy `data/`, `ghidra_scripts/`, `Module.manifest`, `extension.properties`, `LICENSE`, `NOTICE` into `~/Library/ghidra/<ghidra version>/Extensions/ghidra-tms320c28x/` and set `version=` in `extension.properties` to the installed Ghidra version (the repository says 12.1.2).
3. Restart Ghidra (languages are scanned at startup).
4. Import the `.bin` with language `TMS320C28x:LE:32:default`, loader *Raw Binary*, base address `0x3E8000`.

Notes:
- **Local patch to the module:** the module (as of its upstream commit of 2026-09-28) does not decode `IACK #imm16` (opcode `0x763F` + imm16). This image uses it three times and without it Ghidra splits the functions. One constructor was added to `data/languages/tms320c28x.sinc` (marked "LOCAL PATCH"), then `tms320c28x.slaspec` was recompiled; the original is kept next to it as `tms320c28x.sinc.orig_upstream`. The opcode assignment comes from context and is not confirmed against SPRU430.
- Addresses in a word-addressed space: in scripts, `Address.getOffset()` is a **byte** offset (word × 2); `toAddr("3E8000")` takes the word address as a string, `toAddr(0x3E8000)` does not.
- The module targets the F28377D. A Piccolo part has a smaller instruction subset, so the default language should decode it; peripheral register names from the module's setup scripts do not apply.
- The module is not an official TI/NSA product; verify questionable decodes against the TI reference manual (SPRU430).

## Analysis state (Ghidra project `/VNSD/MPPT`)

| Item | State |
|---|---|
| Code range | word `0x3E8000`–`0x3F0BE1` (35810 words), decoded to 99.6 % (remaining words are the header padding) |
| Functions | 454, all named (408 high, 46 medium, 0 low confidence; the confidence is in each function comment). Code outside any function: none. Findings: [VNSD_MPPT_Firmware_Findings.md](VNSD_MPPT_Firmware_Findings.md) |
| Data tail | word `0x3F0BE2`–`0x3F3FFF`: strings (16-bit characters, one character per word), constant tables and the C runtime copy table (`.cinit`, read by `c_int00` at `0x3F2B30`); typed as words / strings. The only functions there are four helpers in a small code island `0x3F2CA0`–`0x3F2D19` |
| RAM / peripherals | three blocks are mapped for labelling (`0x0000`–`0x1FFF`, `0x6000`–`0x7FFF`, `0x8000`–`0x9FFF`); 188 globals are named from console strings and code evidence; six structure types (`ChanCtx`, `DataToCan`, `FaultSnapshot`, `CanFrameEntry`, `TimerDrv`, `AdcPipeline`) are defined and applied where the base address is known |
| Decompilation | all 454 functions decompile without failure; local export under `Dekompilate/VNSD_MPPT_104/` (private repo only) |

How the functions were found: the entry point plus every undecoded position in the code range was treated as a possible function start and followed until no new code appeared. Functions without callers (ISRs, indirect calls) are therefore plausible but unverified; a first attempt also covered the data tail and produced about 340 false functions there, which were removed (the last called function ends at `0x3F0BE2`).

The first attempt at removing the false functions in the data tail also removed the code island above; it was restored after the analysis showed that real code calls into it.

How the names were produced: functions were named bottom-up in three waves along the call graph (leaf functions first), each by a separate reviewer reading the decompilation, strings and the callees' names; every name carries a confidence level. The names are static-analysis labels, not verified on hardware.

Interrupt-handler entries and a few uncalled routines (16 functions) are reached only through the PIE table or not at all, so the call-graph-based search missed them in the first pass. They were found by listing all decoded instructions outside any function and are now functions, named and decompiled. The initial vector table (`0x3F296A`) is decoded; see the findings document.

Not done yet: peripheral register labels beyond the 188 named globals.

## Open

- Package/Q1 variant of the F28035: the 80-pin PN package is likely (the firmware uses GPIO44, which the 64-pin package lacks); not proven. `PARTID` at `0x3D7E80` is never read by the firmware.
- Meaning of the word `0x7CE8` in the image footer (`0x2222` is the device mask of the MPPT; `0x7CE8` matches no tested checksum and is read by no code).
- The MPPT loader that receives the image over CAN is not part of the image (see the findings document).
- Hardware divider/shunt values, CAN bit rate, watchdog timeout.
- Register names and layouts, the watchdog, the PIE table and the IACK/CLA task mapping were checked against the F2803x manual, data manual and errata; remaining unchecked items are listed in the findings document.
