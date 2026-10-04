# Modbus descriptor table — diff v150 ↔ v151

> 🇩🇪 Deutsche Version: [VNSD_Descriptor_Table_Diff_v150_vs_v151_de.md](VNSD_Descriptor_Table_Diff_v150_vs_v151_de.md)

**Device:** Marstek Venus D (VNSD-0), Control firmware
**Date:** 2026-09-29
**German version:** `VNSD_Descriptor_Table_Diff_v150_vs_v151.md`
**Data:** `VNSD_Descriptor_Table_Decoded_v150.csv`, `VNSD_Descriptor_Table_Decoded_v151.csv`

## Summary

| Metric | v150 | v151 |
|---|---|---|
| Entries | 246 | 246 |
| Register range | 30000–38014 | 30000–38014 |
| Registers present in only one version | – | – |
| Type/size changes | – | **1 (32102)** |
| Scale/count changes | – | 0 |
| Table address (SRAM) | 0x20000354 | 0x2000034C |
| Source of 30010 | global variable 0x20000144 | field in the telemetry struct (0x200176D4 = base + 0x38) |

## 1. The only type change: 32102

| | v150 | v151 |
|---|---|---|
| Type | `float32` (0x24), element size 4 | `i16` (0x12), element size 2 |
| Occupies | 32102 + 32103 | 32102 only |
| Source | 0x20014EB8 (same as 30001) | 0x200176B8 (same as 30001) |

The FC03 handler and the read serializer are **instruction-identical** to v150; the consequences follow from the table alone:

| Request | v150 | v151 |
|---|---|---|
| 32103 alone | UsageFault (`VLDR` from unaligned 0x20014EBA), Modbus session dies | exception 02 (address is not covered by any entry) |
| Block spanning 32103 (e.g. 32100–32109, former workaround 32102–32104) | works | **whole block rejected with exception 02** |
| 32102 | float interpretation of an i16 quantity (meaningless) | same value as 30001 |

With the FC03 lookup logic, every register address that falls between two entries returns exception 02; in v151 this now includes 32103.
The HA integration (`d.yaml`) only reads 32104 from this range and is not affected.

## 2. Still broken

- **32101** (i16, ÷10): the read serializer still divides the zero-extended bit pattern (code identical to v150), negative currents stay wrong.

## 3. Shifted source pointers (layout only, no semantic change)

| Delta | Registers | Meaning |
|---|---|---|
| −8 | 30030, 30202–30203, 30205, 30211, 30213, 30300, 30303, 37001–37003, 38000–38014 | core globals moved down by 8 bytes |
| ±0 | 30200–30201, 30212, 37000 | unchanged (incl. flash constant 0x08010000) |
| +0x2800 | 30000–30007, 31000, 32102, 32200–32202, 32204, 32300–32302, 33000–33010 (even), 35000–35002, 35100, 36000, 36100, 36102, 37004, 37013, 37015 | telemetry struct moved (v150 0x20014EAx → v151 0x200176Ax) |
| +0x2820 | 177 registers (30020–30040, 30100–30110, 32100–32114, all pack blocks 34x00–34x18, 35010–35112, 37005–37024 …) | second part of the telemetry/BMS block |
| +0x2822/+0x2824/+0x2828 | 30301, 30304, 30350 / 30302 / 30400 | late SRAM buffers |
| **+0x17590** | **30010** | **new source**, see summary |

The 0x2800 shift equals the heap growth exactly (0x11800 → 0x14000), which suggests the heap sits before these variables in .bss (inference, not individually verified).

## 4. Method

The table is a Keil RW global stored LZ77-compressed in flash (scatter region 0 → 0x20000000, 0x2F00 bytes). v151 was decoded with an unpacker that reproduces the firmware's own `__decompress` (0x08005FBE) exactly. The older script `marstek_descriptor_unpack.py` (local only, not in the repo) does not handle the extension bytes (`(op & 7) == 0` and `(op >> 4) == 0`) but yields the same descriptor table (246/246 identical).
