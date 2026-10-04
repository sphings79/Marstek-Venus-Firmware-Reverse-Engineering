# Modbus Descriptor Table — Unpack Format Fully Reverse-Engineered

> 🇩🇪 Deutsche Version: [VNSD_Descriptor_Table_Unpack_Format_de.md](VNSD_Descriptor_Table_Unpack_Format_de.md)

**Device:** Marstek Venus D (VNSD-0), STM32 ARM Cortex-M, FreeRTOS
**Firmwares:** `VNSD-0_app_1492_0702_142136.bin` (v149.2) and `VNSD-0_app_0150_0805_115146.bin` (v150)
**Date:** 14 August 2026
**Status:** Format cracked, 246/246 entries of both versions decoded and validated
**Tools:** Python + Capstone (custom solver). **Without Ghidra, without a device, purely static.**

---

## 1. Starting Point and What Was Wrong With It

The previous project status read:

> The descriptor table at SRAM `0x20000354` is built at runtime. The 246
> register numbers cannot be statically enumerated from flash (four independent
> PyGhidra checks). Register numbers are computed in init code. Enumeration only
> via live scan, emulation or SWD dump.

**This conclusion is refuted.**

The table is not computed. It is an ordinary `.data` global. The `.data` image is stored
**LZ77-compressed at the end of flash** and is unpacked at boot by the
C runtime.

This explains all four negative earlier findings at once:

| Earlier finding | Explanation |
|---|---|
| No ascending register array in flash | it is stored compressed |
| No dense register `MOVW` immediates | register numbers are data, not code |
| Only 2 code references to `0x20000354` (both readers, no builder) | correct — there *is* no builder |
| No `.data` template with a pointer signature | the template is compressed, hence unrecognizable |

The observed "markers" `0x5A0C` (23052) and `0x1A0C` (6668) are explained as well:
they are not markers, but two frequent opcode/distance pairs (see section 3).

---

## 2. How to Find the Packed Region

The pack format encodes **every zero byte as a zero-run opcode**. The compressed stream
therefore is guaranteed to contain **not a single `0x00`**. That is the signature:

> Search the flash image for the longest contiguous region without a `0x00` byte.

| Firmware | Packed region | Length |
|---|---|---|
| v149.2 | `0x0805CBCC` … `0x0805D7D3` | approx. 2.5 KB |
| v150 | analogous, at the end of flash | analogous |

It is followed by `0xFF` fill up to the image end, with the signature `VNSD` at
`0x0805DFE0` (v149.2).

Additionally confirmed: there is **no** `MOVT #0x0805` in the entire image and **no**
literal pool word that points to the start of the stream. The unpacker therefore gets its
source pointer some other way (scatter/region mechanism). The reset vector in the OTA image
(`IVT[1]` = `0x08004A71`) points to an ordinary application function, not to a
C startup — the startup path is not reachable via the vector table of this image.

---

## 3. The Pack Format (complete)

One-byte opcode, followed by optional literals and a distance byte.

```
op = *src++

nlit = (op & 0x07) - 1        // 0..6 literals; (op & 7) == 0 = escape
len4 = (op >> 4) & 0x0F       // length field, 4 bits (bit 7 belongs to it)

<copy nlit literal bytes from the stream>

if (op & 0x08) {              // bit 3 set = LZ77 match
    dist = *src++;            // 1 byte backward distance
    mlen = len4 + 2;
    <copy mlen bytes from (out_ptr - dist), overlap allowed>
} else {                      // bit 3 cleared = zero run
    <output len4 zero bytes>
}
```

### Bit Fields

| Bits | Meaning |
|---|---|
| 0–2 | Literal count + 1 (value 0 = escape, see below) |
| 3 | 0 = zero run, 1 = LZ77 match |
| 4–7 | Match length − 2 or number of zero bytes |

### The Most Frequent Opcodes

| Opcode | Meaning | Typical use |
|---|---|---|
| `0x1A` | 1 literal + match length 3 | Register number + `75 00 00` from the previous entry |
| `0x5A` | 1 literal + match length 7 | Pointer low byte + rest of the previous entry |
| `0x1B` | 2 literals + match length 3 | Spanning an entry boundary |
| `0x23` | 2 literals + 2 zero bytes | Register number + padding field |
| `0x17` | 6 literals + 1 zero byte | Full pointer + type/size |
| `0x13` | 2 literals + 1 zero byte | Type/size + scale |

The distances are almost always multiples of 12 (`0x0C` = 12, `0x18` = 24, `0x24` = 36,
`0x30` = 48) — i.e. back-references to previous descriptor entries. That is exactly where the
apparent "markers" `0x5A0C` and `0x1A0C` come from: they are the byte pairs
`<opcode 0x5A|0x1A><distance 0x0C>`.

### The Escape Case

`(op & 7) == 0` is rare — seven occurrences in the entire v149.2 image, of which **exactly one
within the descriptor table** (`0x0805CEA3`, op `0x18`). The semantics are not yet
fully formalized. The decoder resolves these spots via lookahead search: it tries
literal counts and chooses the one that leaves the following 12-byte structure intact.
For the table this is backed up by the 246-entry validation; for other
`.data` regions one should not rely on it blindly.

Observed escape locations v149.2:

```
0x0805CEA3  op=18  nlit=9   consume_extra=1   <- inside the table
0x0805D2BA  op=E0  nlit=0   consume_extra=0
0x0805D320  op=68  nlit=21  consume_extra=0
0x0805D3AF  op=10  nlit=32  consume_extra=1
0x0805D430  op=08  nlit=24  consume_extra=0
0x0805D54F  op=E0  nlit=0   consume_extra=0
0x0805D582  op=00  nlit=16  consume_extra=0
```

---

## 4. Entry Layout (12 bytes) — confirmed

```
+0   u16   Register number   (direct PDU address, NO offset, not 0-based)
+2   u16   0                (padding, zero in all 246 entries)
+4   u32   Source pointer   (SRAM, in one case flash)
+8   u8    Type code
+9   u8    Element size     (low nibble)
+10  u8    Scale code
+11  u8    Element count
```

| Type code | Type | | Scale code | Effect |
|---|---|---|---|---|
| `0x01` | u8 | | 0 | ×1 (no conversion) |
| `0x02` | u16 | | 1 | ×10 |
| `0x04` | u32 | | 2 | ×100 |
| `0x11` | i8 | | 3 | ÷10 |
| `0x12` | i16 | | 4 | ÷100 |
| `0x14` | i32 | | 5 | negate |
| `0x24` | float32 | | | |
| `0x31` | ASCII (memcpy) | | | |

**Important:** The scale code describes what the **firmware** computes — not the unit.
Scale code 0 means "raw value unchanged", even if the raw value is in 0.1 V or
0.1 % units. The conversion to the display unit then has to be done by the client.

---

## 5. Validation

1. **Exactly 246 valid entries** — identical to the loop bound in the FC03 handler
   (`FUN_0801eaa4`, v149.2).
2. All 246: padding `+2` = 0, valid type code, pointer in a plausible SRAM/flash range,
   register numbers monotonically increasing 30000 → 38014.
3. **218 different start offsets** yield a **byte-identical** table
   (SHA-256 `becbf8146ce51ffb…`). The decoder resynchronizes robustly; the result does not depend
   on the guessed stream start.
4. Cross-check against known registers passed — see section 7.
5. The v149.2↔v150 diff sorts itself into **five clean pointer delta groups** instead of
   noise. That would not be the case with a decoding error.

---

## 6. Version Comparison v149.2 ↔ v150

| Metric | v149.2 | v150 |
|---|---|---|
| Entries | 246 | 246 |
| Register range | 30000 – 38014 | 30000 – 38014 |
| Registers in only one version | – | – |
| Type changes | – | **0** |
| Scale changes | – | **0** |
| elem_size changes | – | **0** |
| count changes | – | **0** |

**The Modbus semantics are unchanged between v149.2 and v150.** The only difference is
shifted SRAM source pointers (memory layout drift due to inserted variables).

For `marstek_venus_modbus` this means: **no version-dependent case distinction needed.**

### Pointer Shifts

| Delta | Register | Range | Interpretation |
|---|---|---|---|
| **+12** | 204 | `0x20014EA0…0x20015236` | 12 bytes were inserted before the inverter/BMS telemetry struct; base is now `0x20014EAC` instead of `0x20014EA0` |
| **+10** | 6 | 33000–33010 | Energy counter block |
| **+16** | 4 | 30301, 30302, 30304, 30400 | separate late SRAM buffers |
| **+4** | 1 | 31000 | device_name buffer |
| **±0** | 31 | `0x200000xx…0x200001xx`, `0x20001F4x` | unchanged core globals |

### Block 38000–38014 Is NOT New in v150

All 15 entries already exist in v149.2 — same registers, same types, same
scales and **identical source pointers** (`0x20000168`–`0x20000184`, delta ±0). They belong to
the unshifted core globals.

The originally planned counter-test ("38000-38014 must newly appear in v150") therefore fails
— not because of the decoding, but because the premise is wrong. The fact that the block
only returns zeros in the live scan is not a version issue: the source variables are not written in the
tested operating state. The trigger is presumably the state flag at
SRAM `0x20000EE5`, which according to the v149.2 analysis is set on read accesses in the range 38000–39014.

---

## 7. Register 32101 (dc_current) — Status

Defined identically in **both** versions:

| Field | Value |
|---|---|
| Type | `i16` (type code `0x12`) |
| Scale | code 3 = ÷10 |
| Count | 1 |
| Source v149.2 | `0x20014F84` |
| Source v150 | `0x20014F90` |

**The truncation hypothesis is thus off the table.** The source is already 16 bits wide; the
read serializer copies the full register width, nothing is truncated.

`0x9965` read as `i16` is **−26267**, divided by 10 that is **−2626.7 A** — physically
nonsensical. The error therefore clearly arises in the **writer** of the SRAM slot, not in the
Modbus path. Probably a u16 value is written during discharge that gets
misinterpreted as a negative quantity.

**Open:** Identification of the writer. This is the only remaining point that requires Ghidra
(create SRAM block → cross-references to `0x20014F90` → read write site →
charge/discharge branch).

---

## 8. Corrections to the Existing Register Map

| Register | Previously documented | Firmware truth |
|---|---|---|
| **32301** | `ac_offgrid_current`, uint16, 0.01 A | **Duplicate of 32300** — identical source pointer, i.e. the offgrid voltage again. No separate measured value. |
| **32204** | int16 | **u16** (type code `0x02`) |
| **32302** | int32, 2 registers | **i16**, 1 register |
| **36100** | uint64 over 4 registers | **two separate u32**: 36100/36101 and 36102/36103 |
| **30303** | Scale −1 (dBm) | Scale code 0 — the firmware sets **no** sign |
| **34010** | partly listed as temperature | Version — identical source pointer as 30204 |
| **30005** | Pack count assumed | `ac_offgrid_voltage` — identical source as 32300/32301 |
| **33000s** | Scale 0.01 kWh (assumption) | confirmed: scale code 3, FW itself divides by 10 |

---

## 9. Scale Inconsistencies (same in both versions)

Directly relevant for the HA integration:

| Register | Scale code | Consequence |
|---|---|---|
| **32104** (global SOC) | 3 = ÷10 | Firmware divides **itself** → client must **not** apply ×0.1 again |
| **34002** (pack 0 SOC) | 0 = ×1 | Raw value in 0.1 % units → client **must** apply ×0.1 |
| **37005** | 3 = ÷10 | same source as 34002, but already divided |
| **32200 / 32204** | 0 = ×1 | Raw values in 0.1 V and 0.1 Hz respectively, no FW scaling |
| **30104 / 30105** | 1 = ×10 | Firmware **multiplies** by 10 |

---

## 10. Alias Groups — 32 Sources Served Multiple Times

Many "unknown" registers are not new measured values but duplicates. List for v150
(v149.2 analogous, pointer −12):

| Source (v150) | Registers |
|---|---|
| `0x20014EAC` | 30004, 32200, 32201 |
| `0x20014EB0` | 30005, 32300, 32301 |
| `0x20014EB4` | 30006, 32202, 37004 |
| `0x20014EB6` | 30007, 32302 |
| `0x20014EB8` | 30001, 32102 |
| `0x20014EBC` | 30002, 35000 |
| `0x20014EBE` | 30003, 35001, 35002 |
| `0x20014EA4` | 36100, 37013 |
| `0x20014EA8` | 36102, 37015 |
| `0x20014F58 / F5E / F64 / F6A` | 30037/37017, 30038/37018, 30039/37019, 30040/37020 |
| `0x20014F92` | 32108, 35010 |
| `0x20014FA0 / FA2` | 32106/35111, 32107/35112 |
| `0x20014FAC / FAF` | 30109/32113, 30110/32114 |
| `0x20014FB4 / FB6` | 30100/34000, 30101/34001 |
| `0x20014FB8` | 34002, 37005 |
| `0x20014FC2` | 30210, 34004 |
| `0x20014FC4 / FC6` | 30102/34005/37007, 30103/34006/37008 |
| `0x20014FC8 / FCA` | 30104/37006, 30105/35011 |
| `0x20014FCC / FD0` | 34007/37009, 34009/37011 |
| `0x20014FD2` | 30204, 34010 (= 37012) |
| `0x20014FFE / 0x20015000 / 0x20015002` | 30107/34011, 30108/34012, 30106/34017 |

---

## 11. Newly Derived Structures

### MPPT Struct — 4 Strings, Stride 6 Bytes

Base v150 `0x20014F54` (v149.2 `0x20014F48`):

| Field offset | Registers | Meaning |
|---|---|---|
| +0 | 30020, 30021, 30022, 30023 | mppt1–4 voltage |
| +2 | 30024, 30025, 30026, 30027 | mppt1–4 current |
| +4 | 30037, 30038, 30039, 30040 | mppt1–4 power |

Previously only the voltage block was documented; current and power fall directly out of the
pointer stride.

### Pack Struct — 7 Blocks, Stride 0x60

34000 / 34100 / … / 34600, 18 descriptor entries each (master + 6 packs).

### Registers with `count > 1` (occupy multiple Modbus registers)

| Register | count | elem | occupies |
|---|---|---|---|
| 30304 | 12 | 1 | 30304–30309 (MAC, ASCII) |
| 30350 | 12 | 1 | 30350–30355 (comm module FW) |
| 30400 | 8 | 1 | 30400–30403 (IP addresses) |
| 31000 | 20 | 1 | 31000–31009 (device_name) |
| 34x13 | 4 | 2 | 34x13–34x16 (NTC block, per pack) |
| 34x18 | 16 | 2 | 34x18–34x33 (16 cell voltages, per pack) |
| 37009 | 2 | 2 | 37009–37010 |
| 37011 | 2 | 2 | 37011–37012 (37012 = bms_version) |

---

## 12. Confidence Status of the Register Map

The descriptor table provides the **serving contract** conclusively: address, type, scale,
element size, count, source pointer. It does **not** provide the meaning — that requires
the writer of the respective SRAM slot.

| Confidence | Registers | Basis |
|---|---|---|
| high | 62 | Struct offset + live scan + telemetry field names agree |
| medium | 62 | derived from pack struct stride or alias, plausible but not verified |
| low | 1 | 32102 (type conflict with 30001: same source, i16 there, float32 here) |
| **open** | **121** | Type/scale certain, meaning unknown |

---

## 13. Open Points

1. **Lay the telemetry struct against the descriptor pointers.** The 48-byte block of the Micro FW is
   already decoded to 20 fields. A pure offset reconciliation against the source pointers from
   `0x20014EAC` (v150) should name 30–40 of the 121 open registers in one go.
   Pure computation, no Ghidra needed.
2. **Writer analysis in Ghidra** for the remaining open slots — mechanical with the now known
   target addresses: create SRAM block, cross-references, read write site.
3. **32101 writer** (see section 7).
4. **Formalize the escape opcode** (`op & 7 == 0`, 7 occurrences).
5. **Live cross-check** with PV running and offgrid active to confirm the derived
   MPPT current/power registers.
6. **32202** has `elem_size` 4 with type `i16` — inconsistent, re-check.
7. **32110** reads from an odd address (`0x20014F99`) — unaligned u16, re-check.

---

## 14. Tools and Files

### Created in This Session

| File | Content |
|---|---|
| `marstek_descriptor_unpack.py` (local, not in the repo) | Standalone unpacker, works on any Venus D Control FW |
| `VNSD_Descriptor_Table_Decoded_app_1492.csv` | 246 entries v149.2 |
| `VNSD_Descriptor_Table_Decoded_v150.csv` | 246 entries v150 |
| `VNSD_Register_Map_v150_annotiert.csv` | 246 entries with `name`, `unit`, `confidence`, both pointer columns, notes |
| `VNSD_Descriptor_Table_Diff_v1492_vs_v150.md` | Detailed diff of the two versions |
| `VNSD_Descriptor_Table_Unpack_Format.md` | this document |

### Invocation

```bash
python3 marstek_descriptor_unpack.py <firmware.bin> [ausgabe.csv]
```

No dependencies other than Python 3. Runtime a few seconds. With 246 entries the
decoding is confirmed.

Since the register set is identical between the versions, the notes can be carried over to any new version by joining
on the `register` column.

### Analysis Environment of This Session

No Ghidra/ReVa MCP connected. Used: filesystem MCP (access restricted to
the project directory), a custom Linux container with Python 3 and Capstone 5.0.7
for Thumb-2 disassembly, and a self-written solver for the pack format.

### Methodology That Led to the Goal

1. Searched flash for register numbers as u16 literals → found cluster at the end of flash.
2. Built an entropy/zero-byte profile of the tail → isolated the zero-free region.
3. `MOVW`/`MOVT` scan across the entire image → no construction of a `0x0805xxxx` address.
4. Set anchor: `30 75` (register 30000) followed by a plausible SRAM pointer
   `ae 4e 01 20` (`0x20014EAE`) — the known address of `bat_sample_volt`.
5. From this anchor, derived the opcode semantics by constraint solving
   (known expected output ↔ observed byte stream).
6. Implemented the format, validated across start offsets and escape variants until exactly 246
   valid entries came out.

A detour via an SWD/OpenOCD dump of the live table is therefore no longer necessary.
