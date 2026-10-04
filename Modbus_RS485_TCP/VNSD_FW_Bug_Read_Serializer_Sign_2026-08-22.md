# Firmware bug: signed registers with a division scale

> 🇩🇪 Deutsche Version: [VNSD_FW_Bug_Read_Serializer_Vorzeichen_2026-08-22_de.md](VNSD_FW_Bug_Read_Serializer_Vorzeichen_2026-08-22_de.md)

**Date:** 22 August 2026
**Firmware:** Control v150 (`VNSD-0_app_0150_0805_115146.bin`), Venus D
**Affected function:** `Read_Serializer` @ `0x08055414` (v150, correct base 0x08004800; originally noted as `0x08050c14` — old Ghidra analysis with wrong base 0x08000000, difference 0x4800)
**Status:** Cause evidenced in the decompilation, reproduced exactly with two hardware measurements
**Impact today:** exactly **one** register (32101), but the defect lies in the shared path

---

## 1. The bug

`Read_Serializer` puts the value read into a local variable of type `uint` and
applies the scale code to it:

```c
uint local_38;
...
else if (bVar2 == 0x12) {                       // i16
    local_38 = (uint)*(short *)(quelle);        // sign extension to uint
}
...
switch ((char)param_4[5]) {
    case 1: local_38 = local_38 * 10;  break;   // ok
    case 2: local_38 = local_38 * 100; break;   // ok
    case 3: local_38 = local_38 / 10;  break;   // BUG for negative values
    case 4: local_38 = local_38 / 100; break;   // BUG for negative values
    case 5: local_38 = -local_38;      break;   // ok
}
```

The sign extension is correct — `-122` becomes `0xFFFFFF86`. Multiplication and
negation continue to work correctly in two's complement. The **division does not**: `local_38`
is unsigned, so the compiler divides 4,294,967,174 instead of −122.

`Byte_Swap_Copy` then takes the lower two bytes of the result. What remains is
a value unrelated to the measured value.

## 2. Reproduction

`32101` (`bms_battery_current`, i16, scale code 3) at two operating points:

| Operating state | SRAM `0x20014F90` | Modbus returns | Expected |
|---|---|---|---|
| Idle, approx. −0.3 A | `-3` | **39321** (`0x9999`) | `0` |
| 600 W discharge, −12.2 A | `-122` | **39309** (`0x998D`) | `-12` |
| **600 W charge, +10.3 A** | `+103` | **10** | `10` — correct |

Recalculation of the faulty path:

```python
def fw_read(sram_i16):
    u = sram_i16 & 0xFFFFFFFF     # (uint)(short)x
    u = u // 10                   # unsigned
    return u & 0xFFFF             # Byte_Swap_Copy, 2 bytes

fw_read(-3)    ->  39321
fw_read(-122)  ->  39309
```

Both measured values are hit exactly. The SRAM value comes from the double scan of 22 Aug
(`Scan_Logs/idle_bypass_backup48w_2026-08-22.csv` (local, raw data) and
`discharge600w_backup49w_2026-08-22.csv` (local, raw data)); as a cross-check, `34301`
(pack 4 current, no scale code) was at exactly `-3` and `-122` respectively at the same time.

Positive values pass through the path correctly — and this has by now been **measured**, not just
calculated: when charging at 600 W, the same register returned `10` with a pack current of
`103`. The bug therefore occurs **only with negative values**; for a battery storage system
that means: during every discharge.

Side finding from the same scan: when **charging** too, only one pack is active (pack 4, `+103`,
all others `0`). The statement "only one pack at a time" thus holds in both directions.

## 3. Scope of impact in v150

Every combination of a **signed type** (`0x11` i8, `0x12` i16,
`0x14` i32) and a **division scale code** (3 = ÷10, 4 = ÷100) is affected.

Evaluation of the 246 descriptor entries:

| Type | Scale | Entries | |
|---|---|---|---|
| i16 | ÷10 | **1** | affected — 32101 |
| i16 | ×1 | 38 | unremarkable |
| i16 | ×10 | 2 | unremarkable |
| u16 | ÷10 | 2 | unsigned, correct |
| u32 | ÷10 | 6 | unsigned, correct |
| others | ×1 | 197 | no scaling |

**So today a single register.** The defect sits in the serializer, however, which all
FC03 read accesses pass through — any future register with this type/scale combination
inherits it. For Venus A and E v3 (same firmware base) the count has not been checked;
their descriptor tables are not available decoded.

## 4. Consequence for the evaluation

`32101` is **not** usable as a current measurement. The value can be roughly
back-calculated — `current ≈ (raw value − 39321) A` — but only with 1 A quantization, because the
division takes place before truncation to 16 bits, and asymmetrically between charging and
discharging. Unusable for measurement purposes.

**Replacement:** the pack current registers `34x01` carry no scale code and are intact.
Since the Venus D only ever discharges one pack, their sum yields the same current that
the firmware uses internally. This is exactly how it is implemented in the HA integration
(`battery_power_bms`).

## 5. Classification

The back-calculated "resolution of about 1 A per count", which was derived on 22 Aug from the delta
of two measurements, was not a property of the register, but the
quantization of this bug. The corresponding note in the register map is therefore
obsolete.

An English text for the report to the manufacturer is in
`VNSD_FW_Bug_Report_EN_Read_Serializer.md`.

## 6. Status in v151 (firmware 151, checked 2026-10-01)

**Unchanged.** In v151, `Read_Serializer` is at `0x08055fe0` (626 bytes, identical size) and is instruction-identical to v150 except for
decompiler variable names (`bVar1`/`bVar2` swapped to `bVar2`/`cVar1`). The
`switch` cases 3/4 still operate on the unsigned variable `int_val`:

```c
switch(*(undefined1 *)((int)field_desc + 10)) {
case 3:
  int_val = int_val / 10;   // still a BUG for negative values
  break;
case 4:
  int_val = int_val / 100;  // still a BUG for negative values
  break;
```

Register 32101 remains in v151, by the same evaluation (type i16, scale code ÷10), the only affected register
(see `VNSD_Descriptor_Table_Diff_v150_vs_v151.md`, section "Unverändert fehlerhaft"). Source: decompilation
`Dekompilate/VNSD_Control_v151/08055fe0_Read_Serializer.c` (local, not in the repo).
