# Registers 46500–46544: 15 structures of 8 bytes each, EEPROM-persistent, forwarded over CAN

> 🇩🇪 Deutsche Version: [VNSD_Write_Register_46500_Block_v150_de.md](VNSD_Write_Register_46500_Block_v150_de.md)

**Date:** 22 August 2026
**Firmware:** Control v150, `Write_Handler` @ `0x08051D14`
**Status:** Mechanism fully decoded, field meaning open

---

## 1. Addressing

```c
if (0xb5a3 < reg && reg < 0xb5d1) {          // 46500 .. 46544
    idx   = (reg - 0xb5a4) / 3;              // 0..14  -> 15 structures
    field = (reg - 0xb5a4) % 3;              // 0..2
```

Structure base: SRAM **`0x20014ED4`**, 8 bytes per entry.

| `field` | Register (example entry 0) | Access |
|---|---|---|
| 0 | 46500 | `u16` at `+0` |
| 1 | 46501 | upper 16 bits of the `u32` at `+4` |
| 2 | 46502 | lower 16 bits of the same `u32` |

There is a **read path** (`param_3 == 0`) — confirmed on the device: 46500–46512
respond, all with 0 because they are unused.

## 2. What a write triggers

```c
EEPROM_Write(idx * 8 + 0x3700, base + idx * 8, 8);
Inverter_Set_Schedule_Reg(idx + 0x30, u16@+0, u32@+4);
```

Two side effects on **every** write:

1. **EEPROM** at address `0x3700 + idx*8`, 8 bytes. **No change guard** — unlike
   for 44002/44003, even an unchanged value ends up in non-volatile memory.
   Do not write cyclically.
2. **CAN to the inverter**, register index `0x30 + idx` (valid `0x30`–`0x3E`,
   exactly 15). Payload 8 bytes: `u16` in bytes 0–1, bytes 2–3 zero, `u32` in bytes 4–7.
   Built in `Inverter_Set_Schedule_Reg` @ `0x08005CE0` via
   `CAN_Build_Arbitration_ID(reg, 1, 4, 0)` and `Register_WriteValue(id, buf, 8)`.

## 3. The inverter does not know these messages

**Broken down 2026-08-22.** The CAN ID is built in `CAN_Build_Arbitration_ID`
(`0x08032F20`):

```
Bits 0–7    function code      here 0x30 + idx
Bits 8–11   source             here 1
Bits 12–15  destination        here 4
Bits 24–28  counter/flags      here 0
```

The receiver in the Micro/VNS firmware v116 is
`EMS_Inverter_CAN_Dispatcher` (`0x08001940`). It decodes the function code as
`param_1 & 0xff` — the same position the builder fills — and branches to:

| Codes | Meaning |
|---|---|
| `0x01`–`0x07` | SetPower, Enable/Disable, max discharge/charge power (cap 2500 W), sleep |
| `0x10`–`0x13` | Telemetry block (48 B), further queries |
| `0x16` | two u32 into the inverter struct |
| `0x50`–`0x58` | Factory test, FW update, backup mode |
| `0x60`, `0xC1`–`0xC3`, `0xCB`, `0xCE` | Debug and miscellaneous |

**`0x30`–`0x3E` is missing entirely.** The values land in the `uVar14 < 0x52` branch,
match none of the sub-conditions there, and are discarded without any reaction.

A write to 46500–46544 therefore consumes one EEPROM cycle and produces
a CAN message that **nobody evaluates** — at least not the inverter
in v116. Another bus participant remains conceivable (the destination field is set to 4) or
a function that is not fully implemented in this firmware pairing.

## 3b. The BMS is not the addressee either

**Checked 2026-08-22 against `BMS_118_20260119100535e43806957.bin`.**

The concrete CAN ID of the schedule message can be computed. With
`param_4 = 0` the builder yields `(0 + 0x10000) & 0xe00f0000 = 0x10000`, plus
`destination 4 << 12`, `source 1 << 8` and the function code:

```
ID = 0x00014130 + idx        (idx = 0..14)   → extended frame
```

The BMS receiver `CAN_RX_Command_Dispatcher` (`0x0800C928`) checks the destination field **first**:

```c
if ((ushort)local_10 >> 0xc == 3) {      // bits 12-15 must be 3
    bVar1 = (byte)local_10;              // only then the function code
```

Our messages carry a **4** there. They are discarded before the
function code is even read. And even if they got through: the BMS
only knows `0x01`–`0x05`, `0x0A`, `0xC4`, `0xCE`. The second receive path
`CAN_RX_Handler` (`0x08005A40`) compares fixed full IDs (`0x180102AA`,
`0x100401AA` and derivatives) — none of them lies in our range. A scan
over all 649 BMS functions for a comparison against `0x30`–`0x3E` yields
five hits, all uninvolved (I²C EEPROM, printf digit parser,
EEPROM log slots, CLI escape, SoC calibration).

### Target classes

46 callers of `CAN_Build_Arbitration_ID` in the Control yield: destination 4 =
inverter (34 send sites), destination 3 = BMS (5), destination 2 = RS485 path (13),
destination 1 = Control itself (1). Our schedule message goes to destination 4 — so the
addressee is **correctly chosen**; the inverter *is* the appropriate
receiver for schedule data. It just does not understand these commands in v116.

Full table of all send sites, receiver code lists and
coverage gaps: `../CAN_Bus/VNSD_CAN_Target_Classes_and_Function_Codes.md`.

**Important context:** `0x30`–`0x3E` is *not* the only gap. To destination 4 the Control sends
a total of **23 codes for which there is no receiver in Micro v116** — additionally `0x08`, `0x0A`, `0x14`, `0x17` and `0x40`–`0x43`.
The schedule block is thus not an isolated case but part of a pattern:
the Control firmware serves Venus A, D and E v3 from one code base and
contains commands that this specific inverter firmware does not know.

This does not weaken the statement — for the write access to 46500–46544
it remains the case that in this firmware pairing nobody
evaluates the message. It only means: a missing receiver does not prove that the
function is dead in general. Another firmware version or hardware variant
might know it.

## 4. Why the meaning nevertheless remains open

A cross-reference run on `0x20014ED4` yields **zero** hits: the Control reads
the block nowhere, does not evaluate it and makes no decision with it. It
is a pure pass-through — Modbus in, EEPROM and CAN out.

The semantics therefore lie in the **Micro/VNS firmware**, in its registers
`0x30`–`0x3E`. A first search there remained without result: `modbus_register_handler`
(`0x080124E8`) handles its own schedule with **10-byte** entries and
HH/MM validation (`< 0x18` hours, `< 0x3c` minutes) — that is a different
table than our 8-byte structure. The CAN receive path has not yet been
broken down.

The name `Inverter_Set_Schedule_Reg` assigned by Ghidra is a **hypothesis**
and should not count as evidence for "schedule".

## 5. Delimitation

Do not confuse with:

- **43100–43129** — the six schedule slots that the integration maps
  (10-byte struct: enable/start/end/power).
- **`0x20014D02`** — 10 entries of 10 bytes each, power at `+2`. This is the block that
  `Config_ScheduleEntries_ClampPower800W` clamps for the 800 W power class.
- **41500–41631** — raw arrays for time and power, also SRAM.

## 6. Recommendation

**Do not include in the integration.** A write consumes one
EEPROM cycle and produces a CAN message for which there is no receiver
in v116. Reading is harmless — as long as all 15 entries are zero, there is however
nothing to see there.

The BMS firmware has since been checked (section 3b, section 7): it drops the message.

## 7. Note from the register resolution of 2026-10-01: the block is functionless on the Venus D

Full analysis: [VNSD_Unclear_Registers_Resolution_2026-10-01.md](VNSD_Unclear_Registers_Resolution_2026-10-01.md). The block (CSV status `FUNCTIONLESS`, `befund_D`: no consumer on the Control besides read-back) does nothing on the Venus D:

- The CAN frame with function `0x30 + idx` **passes the Micro's FDCAN filter**, but the dispatcher of Micro 115/116 has **no branch** for `0x30`–`0x3E`; it is discarded.
- The **BMS drops** the frame (destination class 4 instead of 3; see section 3b); BMS 117.7 and 118 have no receiver for `0x30`–`0x3E`.
- **Nothing else reads the cache** (`g_canScheduleRegCache`, SRAM, 120 bytes). Reads return only what was written.
- The only real effect of a write is EEPROM wear (no change guard). EEPROM base is **`0x3700`** in v150 and later (`0x36BF` in v149 / v149.2); the block is **absent in v147**.

Recommendation unchanged: do not expose. Sections 4 and 6 (meaning open, BMS unchecked) are superseded by this note.
