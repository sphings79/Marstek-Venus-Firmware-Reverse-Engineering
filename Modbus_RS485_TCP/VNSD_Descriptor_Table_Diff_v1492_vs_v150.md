# Descriptor Table: Diff v149.2 → v150 (Marstek Venus D, Control FW)

> 🇩🇪 Deutsche Version: [VNSD_Descriptor_Table_Diff_v1492_vs_v150_de.md](VNSD_Descriptor_Table_Diff_v1492_vs_v150_de.md)

Sources:
- `VNSD-0_app_1492_0702_142136.bin` (v149.2) → `VNSD_Descriptor_Table_Decoded_app_1492.csv`
- `VNSD-0_app_0150_0805_115146.bin` (v150)  → `VNSD_Descriptor_Table_Decoded_v150.csv`

Both statically unpacked with `marstek_descriptor_unpack.py` (local, not in the repo) from the LZ77-packed
`.data` image.

## Overall Result

| Metric | v149.2 | v150 |
|---|---|---|
| Entries | 246 | 246 |
| Register range | 30000 – 38014 | 30000 – 38014 |
| Only in this version | – | – |
| Type changes | \- | **0** |
| Scale changes | \- | **0** |
| elem_size changes | \- | **0** |
| count changes | \- | **0** |

**The Modbus semantics are unchanged between v149.2 and v150.** No register
was added, none was removed, and no type or scale was changed. The only
difference is shifted SRAM source pointers — i.e. pure memory-layout drift
caused by newly inserted variables, not a protocol change.

Consequence for `marstek_venus_modbus`: **no version-dependent case distinction
needed.** One register definition covers both firmware versions.

## Pointer Shifts by Block

| Delta | Registers | SRAM range | Interpretation |
|---|---|---|---|
| **+12** | 204 registers (30000–37024, excluding those listed below) | `0x20014EA0…0x20015236` | 12 bytes were inserted before the inverter/BMS telemetry block. The telemetry struct now starts at `0x20014EAC` instead of `0x20014EA0`. |
| **+10** | 33000, 33002, 33004, 33006, 33008, 33010 | `0x20014D92…0x20014DA6` → `0x20014D9C…0x20014DB0` | Energy counter block (u32, ÷10) shifted by 10 bytes |
| **+16** | 30301, 30302, 30304, 30400 | `0x20018B4C`, `0x2001915D`, `0x2001893E`, `0x2001ADB5` | separate late SRAM buffers, +16 each |
| **+4** | 31000 | `0x20014CDC` → `0x20014CE0` | device_name buffer |
| **±0** | 31 registers | `0x200000xx…0x200001xx`, `0x20001F4Cx` | unchanged core globals |

## Key Individual Findings

**Block 38000–38014 is NOT new in v150.**
All 15 entries already exist in v149.2 with identical descriptors —
same registers, same types, same scales and **identical source pointers**
(`0x20000168`–`0x20000184`, delta ±0). The block belongs to the unshifted
core globals. The fact that it returns only zeros in the live scan is therefore not
a version issue, but means: the source variables are simply not written in the tested
operating state.

**32101 (dc_current)** is defined identically in both versions:
`i16`, scale code 3 (÷10), count 1.
Source: v149.2 `0x20014F84` → v150 `0x20014F90`.
The source is already 16 bits wide, nothing is truncated in the serializer.
The discharge value `0x9965` (as i16 = −26267, ÷10 = −2626.7) therefore originates in the
**writer** of the SRAM slot, not in the Modbus path.

**Scale inconsistencies (identical in both versions, relevant for the HA integration):**
- 32104 (global SOC): scale code 3 → **the firmware itself divides by 10**
- 34002 (pack 0 SOC): scale code 0 → **raw value in 0.1 % units**, the client must apply ×0.1
- 37005: same source as 34002 (`0x20014FAC`/`0x20014FB8`), but scale code 3 → already delivers the same measurement divided
- 32200/32204 (ac_voltage, ac_frequency): scale code 0 → raw value in 0.1 V and 0.1 Hz respectively, no FW scaling

**Double mappings (in both versions):**
- 32102 (`float32`) and 30001 (`i16`) point to the same address — one of the two type declarations must be misinterpreted
- 32200/32201 and 32300/32301 each share one source
- 35000/35001 are aliases of 30002/30003; 37013/37015 are aliases of 36100/36102
- 32202: `i16` with `elem_size` 4 — inconsistent, needs verification
