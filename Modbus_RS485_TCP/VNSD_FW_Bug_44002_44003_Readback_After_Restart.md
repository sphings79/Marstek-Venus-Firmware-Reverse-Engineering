# 44002 / 44003 Permanently Read 0 After a Restart

> 🇩🇪 Deutsche Version: [VNSD_FW_Bug_44002_44003_Ruecklesewert_nach_Neustart_de.md](VNSD_FW_Bug_44002_44003_Ruecklesewert_nach_Neustart_de.md)

**Device:** Marstek Venus D (VNSD-0)
**Firmware:** Control/EMS **v150** (`VNSD-0_app_0150_0805_115146.bin`)
**Registers:** 44002 `max_charge_power`, 44003 `max_discharge_power`
**Date:** 20 September 2026
**Status:** Cause found · evidenced on the device · **impact on operation ruled out** (section 2b)

**Note on addresses:** All flash addresses mentioned below have been converted for this version to the correct
Ghidra base 0x08004800 (originally noted with the wrong base 0x08000000, difference
consistently 0x4800). The old address is noted in parentheses in each case.

---

## 1. Summary

After a restart of the Control, 44002 and 44003 return the value **0** and keep it
until something writes them.

**Cause:** There are two memory locations, and the read path accesses the wrong one.

| | Address | Who fills it | Role |
|---|---|---|---|
| **Config block** | `0x2000012B` (+2 charge, +4 discharge) | loaded from EEPROM `0x200` at startup and validated | **the effective limit** |
| **Shadow value** | `0x20000156` (+0 charge, +2 discharge) | only on a write access | **what 44002/44003 return** |

The shadow value is not initialized anywhere. After a restart it holds its
BSS zero value while the config block is loaded correctly — that is why 0 is read while the
device continues to limit correctly.

**It is a pure readback bug, not a loss of function.** For a client this means:
**a 0 in these two registers means "unknown", not "zero watts".**

---

## 2. Measurement

### 2a. Course Over a Restart

Recorded directly on the device (`192.168.1.50:502`, proxy stopped), one-second interval:

| Time | 44002 | 44003 | bat_power | SoC | ems_ver |
|---|---|---|---|---|---|
| 11:43:18 | 2500 | 2500 | −12 W | 23 | 150 |
| 11:43:29 | — device not reachable — | | | | |
| 11:43:59 | **0** | **0** | −12 W | 0 | 150 |
| 11:44:52 | **0** | **0** | −13 W | 24 | 150 |

The SoC recovers from 0 to 24 and `ems_ver` is back at 150 — so the device has
fully booted. Only the two maximum values stay at 0. Even minutes later
nothing changes; only a write access restores them.

### 2b. Does the Device Still Charge? — A/B Test

Same charge command, once with `max_charge = 0` and once with 2500. Setup:
42000 = `0x55AA` (RS485 enable, needed after a restart), 42020 = 400 W, 42010 = 1 (charging).
Register 30001 was measured over 30 seconds each:

| | measured charge power |
|---|---|
| **A: max_charge = 0** | 342–349 W |
| **B: max_charge = 2500** | 347–348 W |

Identical. The 0 does not prevent charging. Sign for context: 30001 is at
−12/−13 W at idle (self-consumption) and at +348 W while charging.

---

## 3. Refuted Hypothesis — Please Do Not Repeat

On the first pass it was concluded from `Inverter_Power_Setpoint_Calc` (0x0800a968, old 0x08006168) that
a 0 must clamp every setpoint and thus cripple the battery. The clamp there
reads `DAT_080063d8 + 2` and `+ 4` (the old, not yet fully resolved notation; in the current
decompilation the two fields are already resolved directly as named globals), and `DAT_080063d8` resolves
to `0x2000012B` — the config block from which 44002/44003 are also served — without any recognizable
null check:

```c
if (*(ushort *)(DAT_080063d8 + 4) < uVar6)  uVar6 = *(ushort *)(DAT_080063d8 + 4);
if (... uVar6 <= -*(ushort *)(DAT_080063d8 + 2))  uVar6 = -*(ushort *)(DAT_080063d8 + 2);
```

Current decompilation (`Inverter_Power_Setpoint_Calc`, same clamp, fully symbolized):

```c
if ((int)(uint)g_maxDischargePowerW < (int)power_setpoint) {
    power_setpoint = (uint)g_maxDischargePowerW;
}
if (-power_setpoint != (uint)g_maxChargePowerW &&
    (int)power_setpoint <= (int)-(uint)g_maxChargePowerW) {
    power_setpoint = -(uint)g_maxChargePowerW;
}
```

`g_maxChargePowerW` (0x2000012D) and `g_maxDischargePowerW` (0x2000012F) are exactly the config block fields
from section 4a — the clamp is confirmed to be the same one, via direct access instead of via the old `DAT_` pointer.

**The A/B test in section 2b refutes this conclusion.** With `max_charge = 0` the
device charges unchanged. Either this branch is not taken in the `force_mode` path, or `uVar6`
does not carry the setpoint at this point. Where exactly the misinterpretation lies is open —
the measurement is authoritative in any case.

The lesson is a general one: seeing a clamp in the decompilation does not mean
that it is also executed in the operating case under consideration.

**Addendum — now resolved.** The branch is executed after all, it just never reads a 0:
`DAT_080063d8` is the config block `0x2000012B`, and that is loaded correctly after startup.
The 0 is solely in the shadow value `0x20000156`, which the clamp does not touch at all
(section 4). So the A/B test did not refute the clamp, but showed that
the clamp and the Modbus return come from different sources — which is the actual finding
of this document.

---

## 4. The Two Memory Locations

### 4a. The Config Block `0x2000012B` — the Effective Limit

Loaded at startup. `Board_SystemStartup` (0x0801e194, old `FUN_08019994` @ 0x08019994) is the top-level
init sequence — IRQs off, init chain, IRQs on, CH395 setup — and calls `Config_LoadOrInit_DeviceSettings`
(0x0802dcbc, old `FUN_080294bc` @ 0x080294bc) within it, the config loader:

```c
iVar2 = EEPROM_Read(0x200, &g_chargeCutoffSoc, 7);    // g_chargeCutoffSoc @ 0x2000012B
```

(old notation: `DAT_0802987c`, not yet resolved to the named global)

Seven bytes from EEPROM `0x200`, i.e. exactly the block:

| Offset | SRAM | Field | EEPROM |
|---|---|---|---|
| +0 | 0x2000012B | charge_cutoff_soc | 0x200 |
| +1 | 0x2000012C | discharge_cutoff_soc | 0x201 |
| +2 | 0x2000012D | **max_charge_power** | **0x202** |
| +4 | 0x2000012F | **max_discharge_power** | **0x204** |
| +6 | 0x20000131 | grid_standard | 0x206 |

At the end of `Config_LoadOrInit_DeviceSettings` the validator
`Config_ClampDefaults_ParamBlock` (0x0800935c, old `FUN_08004b5c` @ 0x08004b5c) runs **unconditionally** and
raises implausible values to defaults:

| Field | at 0 or out of range |
|---|---|
| charge_cutoff_soc | → 100 |
| discharge_cutoff_soc | → 12 |
| **max_charge_power** | → **2500** |
| **max_discharge_power** | → **800** |

After startup this block therefore holds a usable value in every case. The
power limiting reads it — `Inverter_Power_Setpoint_Calc`, `CT_PowerSetpoint_Compute`,
`Grid_Export_Power_Limiter`, `WorkMode_State_Machine` — and keeps working accordingly.

As an aside: the default for max_discharge_power is **800**, not 2500. Anyone who
finds 800 there has not "set 800", but has a discarded EEPROM value.

### 4b. The Shadow Value `0x20000156` — What 44002/44003 Return

A write access to 44002 (`Write_Handler` @ 0x08056514, old 0x08051D14) does **three** things — current
decompilation, fully named:

```c
Inverter_Write_Reg_0x04_U32Value(1, value);        // via CAN to the micro inverter
Config_Write_U16_0x202(max_charge_power_w);        // config block 0x2000012D + EEPROM 0x202
_g_maxChargePowerRegCache = max_charge_power_w;    // shadow value 0x20000156
```

The read path (`is_write == 0`), however, returns **only the shadow value**:

```c
if (is_write == 0) {
    local_34 = (uint)_g_maxChargePowerRegCache;    // 0x20000156, not 0x2000012D
    Byte_Swap_Copy(rsp, &local_34, 2);
    return 0;
}
```

(old notation: `*DAT_080529bc = uVar2` and `DAT_080529bc[1]` respectively — the same accesses, not yet resolved to the
named globals `g_maxChargePowerRegCache`/`g_maxDischargePowerRegCache`)

`_g_maxChargePowerRegCache` resolves to **0x20000156**, not to 0x2000012D; 44003 analogously uses
`g_maxDischargePowerRegCache` @ **0x20000158**, not 0x2000012F. A cross-reference on 0x20000156 yields
**exactly four hits**, and none of them is a load:

| From | Kind |
|---|---|
| `Write_Handler` 0x0805700c (old 0x0805280c) | WRITE — Modbus write access |
| `Write_Handler` 0x0805701a (old 0x0805281a) | READ — Modbus read path |
| `Protocol_AA_SetExtendedParams` 0x0803449c (old 0x0802fc9c) | WRITE — copied from a CAN frame |

No `EEPROM_Read`, no init, no restore. After a restart the BSS zero value sits there.

### 4c. Why the Value Sometimes Comes Back on Its Own

`Protocol_AA_SetExtendedParams` (0x08034494, old 0x0802fc94) copies six bytes from a CAN frame into the
shadow:

```c
*DAT_0802fca8 = *param_2;                        // 4 bytes: 44002 and 44003
*(undefined2 *)(puVar1 + 1) = *(undefined2 *)(param_2 + 1);
```

If a CAN node sends this frame, the shadow fills without any action by a Modbus client.
That explains observations where the readback value was correct again after a while.

### 4d. Correction to the Existing Documentation

`VNSD_Register_Persistence_RAM_vs_EEPROM_v150.md` and the function header of `Write_Handler` list
an "SRAM mirror 0x2000012D" for 44002 and "0x2000012F" for 44003. That describes the
**write path** correctly, but the **read path** incorrectly — it reads 0x20000156/0x20000158.

---

## 5. Workaround

A one-time write access after the restart restores the readback value. The value in
EEPROM is only touched if it changes (change guard in `Config_Write_U16*`),
so writing back the same value costs no EEPROM cycle:

```python
client.write_register(44002, 2500)   # insert your own value
client.write_register(44003, 2500)
```

For an integration it makes sense to remember the last known value and, after a
detected restart — e.g. on an SoC jump to 0 or on connection loss — write it back once.
Alternatively, display the 0 as "unknown" instead of as a number.

---

## 6. For a Vendor Report

Robust and concisely formulable:

1. The effective configuration is in `0x2000012B` and is loaded correctly from EEPROM
   `0x200` at startup and validated. Nothing is missing there.
2. The FC03 read path for 44002/44003, however, does not read this block, but a
   separate shadow value at `0x20000156`, which only a write access or a CAN frame
   fills.
3. Consequence: FC03 on 44002/44003 returns 0 after every restart, even though the limiting is active and
   correct.
4. Not a safety issue — the device keeps limiting. It is a display bug, but one that
   misleads every client that reads the value or regulates on it.
5. The fix is small and there are two clean variants: either serve the read path directly
   from the config block (`0x2000012D` / `0x2000012F` instead of `0x20000156`), or initialize the
   shadow once at startup from the config block.

---

## 6b. Venus A — Same Defect, Plus Differing Limits

Static, from `150_control_VNSA-0_app_0150_0811_114724.bin`, not verified on the device.

**The readback bug is identical.** The read path of the A `Write_Handler` (@ `0x080516B8`)
serves 44002 and 44003 from the same shadow value, and the pointer `DAT_08052360` resolves to
**`0x20000156`** — the same SRAM address as on the D. There, too, nothing is loaded at startup. A Venus A therefore also reads 0
after a restart.

**The limits differ, and asymmetrically:**

| | Venus D | Venus A |
|---|---|---|
| `Write_Handler`, 44002 (`uVar8 == 900`) | `param_4 < 0x9C5` → ≤2500 | `param_4 < 0x9C5` → **≤2500** |
| `Config_Write_U16_0x202` (stores 44002) | clamp `0x9C4` = 2500 | clamp **`0x5DC` = 1500** |
| `Write_Handler`, 44003 (`uVar8 == 0x385`) | `param_4 < 0x9C5` → ≤2500 | **`param_4 < 0x5DD` → ≤1500** |
| `Config_Write_U16` (stores 44003) | clamp 2500 | clamp **1500** |
| `FUN_08004b5c` validator default | 2500 | **1500** |

For a client on the Venus A this means:

- **44002:** a write access with 2000 is **accepted** (return 0, no exception) and
  silently clamped to 1500 when stored. Anyone who judges success by the return value notices nothing.
- **44003:** the same write access is **rejected**, `return 3`, i.e. exception 3
  *illegal data value*.

Two twin registers, two different checks. For a vendor report this is a
separate, small point: the input validation of 44002 on the A does not match the limit that is
actually applied when storing.

The workaround from section 5 applies unchanged, only with 1500 instead of 2500 as the sensible value.

---

## 6c. Status in v151 (Firmware 151, checked 2026-10-01)

**Unchanged.** All involved functions exist in v151 with identical size/logic, only at different
addresses (heap enlargement 0x11800 → 0x14000 shifts the following code):

| Function | v150 | v151 |
|---|---|---|
| `Write_Handler` | 0x08056514 | 0x080570e0 |
| `Config_LoadOrInit_DeviceSettings` | 0x0802dcbc | 0x0802e17c |
| `Config_ClampDefaults_ParamBlock` | 0x0800935c | 0x0800935c (unchanged, same address) |
| `Board_SystemStartup` | 0x0801e194 | 0x0801e324 |
| `Protocol_AA_SetExtendedParams` | 0x08034494 | 0x08034aa8 |

The read path for 44002 in `Write_Handler` (v151, register offset `uVar3 == 900`) still returns only the
shadow value:

```c
if (uVar3 == 900) {
  if (is_write == 0) {
    bcd_reply_value = (uint)_g_maxChargePowerRegCache;   // still 0x20000156, not the config block
    Byte_Swap_Copy(rsp,&bcd_reply_value,2);
    return 0;
  }
  ...
  _g_maxChargePowerRegCache = max_charge_power_w;
  return 0;
}
```

For 44003 (register offset `uVar6 == 0xabe3`) it is identical with `g_maxDischargePowerRegCache`
(still 0x20000158). In v151 `Config_LoadOrInit_DeviceSettings` additionally initializes the server type
at boot (see `Control_FW/VNSD_Control_Diff_v150_v151.md`), but **not** the two shadow values — they remain
at their BSS zero value at startup. The readback bug after a restart persists unchanged in v151.

## 7. Cross-References

- `VNSD_Register_Persistence_RAM_vs_EEPROM_v150.md` — write paths, EEPROM offsets, change guard
- `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20.md` — the other open firmware finding
- `VNSD_FW_Bug_Report_EN_Read_Serializer.md` — sign bug in the read serializer (32101)
- Recording of the restart and the A/B test: see `Scan_Logs/` (local, raw data)
