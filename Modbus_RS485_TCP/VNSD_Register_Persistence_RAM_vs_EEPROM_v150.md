# Register Persistence: RAM vs. EEPROM (Control FW v150)

> 🇩🇪 Deutsche Version: [VNSD_Register_Persistenz_RAM_vs_EEPROM_v150_de.md](VNSD_Register_Persistenz_RAM_vs_EEPROM_v150_de.md)

> **Firmware:** `VNSD-0_app_0150_0805_115146.bin` (v150) · ARM Cortex-M, FreeRTOS
> **Source:** Static analysis `Write_Handler` @ `0x08051D14` in Ghidra (ReVa)
> **Status:** 2026-08-14
> **Addressing:** direct (PDU = register number, no offset) · Write via **FC06/FC10**

---

## 1. Question

With zero feed-in control, the power setpoints are rewritten every few seconds
to minutes. If each of these writes ends up in non-volatile memory, that would be a wear problem. This document clarifies for
each relevant write register **whether it only touches SRAM or writes through to the EEPROM**.

**Short answer:**

| Register | Target | Persistent? | Suitable for cyclic writing |
|---|---|---|---|
| 42010 `force_mode` | SRAM `0x20000349` | ❌ no | ✅ yes |
| 42011 `charge_to_soc` | SRAM `0x2000034A` | ❌ no | ✅ yes |
| **42020** charge power | SRAM `0x2000034C` | ❌ no | ✅ yes |
| **42021** discharge power | SRAM `0x2000034E` | ❌ no | ✅ yes |
| 45008 power setpoint | CAN only → Micro MCU | ❌ no | ✅ yes |
| **44002** `max_charge_power` | SRAM + **EEPROM `0x202`** | ✅ **yes** | ⚠️ **no** |
| **44003** `max_discharge_power` | SRAM + **EEPROM `0x204`** | ✅ **yes** | ⚠️ **no** |

---

## 2. 42020 / 42021 — pure SRAM

### Code evidence (`Write_Handler`, decompiled)

```c
if (param_2 == 0xa424) {                       // Register 42020
    if (param_3 == 0) {                        // read mode (FC03)
        Byte_Swap_Copy(param_1, DAT_0805259c + 4, 2);
        return 0;
    }
    if ((int)param_4 < 0x9c5) {                // value range 0..2500
        *(ushort *)(DAT_0805259c + 4) = uVar2; // → SRAM 0x2000034C
        return 0;                              // NO EEPROM_Write
    }
    return 3;                                  // Exception 3
}
// 42021 (0xa425) analogous → *(ushort *)(DAT_0805259c + 6) = SRAM 0x2000034E
```

`DAT_0805259c` = `0x20000348` (literal pool). The block starting at `0x20000348`:

| SRAM | Content | Register |
|---|---|---|
| `0x20000348` | Reset/command status | 41000 |
| `0x20000349` | `force_mode` (0/1/2/3) | 42010 |
| `0x2000034A` | `charge_to_soc` (13–100) | 42011 |
| `0x2000034C` | Charge power setpoint (u16, W) | **42020** |
| `0x2000034E` | Discharge power setpoint (u16, W) | **42021** |

### Cross-reference proof

`find-cross-references` on `0x2000034C` and `0x2000034E` yields **exactly three
references each**, all from two functions:

| From | Type |
|---|---|
| `Write_Handler` @ `0x08051D14` | WRITE (Modbus write) |
| `Write_Handler` @ `0x08051D14` | PARAM (readback path) |
| `Remote_Power_Command_Execute` @ `0x0801EBA4` | READ |

**No** `EEPROM_Write`, **no** `EEPROM_Read`, **no** init from the
config block, **no** save-on-shutdown and no periodic config flush
(`Config_Save_RuntimeCounters` writes EEPROM `0x500`, `Config_Save_UserDataBlock`
writes `0x4000` — neither touches the block `0x20000348`).

> **Consequence:** After a reboot/power cycle, 42010/42011/42020/42021 are at **0**
> (BSS initialization). The value does not survive a restart.
> An impression to the contrary typically arises because the
> automation (HA/Node-RED) immediately writes again after the reconnect.

### Application's view

```c
void Remote_Power_Command_Execute(void)      // 0x0801EBA4
{
  cVar1 = *(char *)(0x20000348 + 1);          // force_mode
  if      (cVar1 == 0) Inverter_Power_Setpoint_Calc(1,  0);
  else if (cVar1 == 1) Inverter_Power_Setpoint_Calc(1, -(int)*(short *)0x2000034C);
  else if (cVar1 == 2) Inverter_Power_Setpoint_Calc(1, +(int)*(short *)0x2000034E);
  else if (cVar1 == 3) { /* charge_to_soc control against SOC */ }
  else                 Inverter_Power_Setpoint_Calc(1,  0);
}
```

Called from `WorkMode_ChangeHandler` @ `0x0803080C`.

> ### ⚠️ Correction of the register meaning
>
> Negative setpoint = **charging**, positive = **discharging** (confirmed by the
> clamping logic in `Inverter_Power_Setpoint_Calc`, see §4).
> Since `force_mode == 1` (= charging, verified live) uses the value from **42020**
> negated and `force_mode == 2` (= discharging) uses the value from **42021**
> positive, the following applies:
>
> | Register | correct | wrong (legacy) |
> |---|---|---|
> | **42020** | `charge_power_limit` — **charging** | ~~discharge_power_limit~~ |
> | **42021** | `discharge_power_limit` — **discharging** | ~~charge_power_limit~~ |
>
> The plate comment in the Ghidra `Write_Handler` wrongly names 42020 as
> "discharge_power_limit" — corrected on 2026-08-14.

---

## 3. 44002 / 44003 — EEPROM-persistent

### Code evidence

```c
// Register 44002 (0xABE2)
if ((int)param_4 < 0x9c5) {
    Inverter_Write_Reg_0x04_U32Value(1, param_4);   // CAN reg 0x04 → Micro MCU
    Config_Write_U16_0x202(param_4);                // → EEPROM 0x202
    *(ushort *)0x20000156 = uVar2;                  // shadow copy
    return 0;
}

// Register 44003 (0xABE3)
if ((int)param_4 < 0x9c5) {
    Inverter_Write_Reg_0x03_U32Value(1, param_4);   // CAN reg 0x03 → Micro MCU
    Config_Write_U16(param_4);                      // → EEPROM 0x204
    *(ushort *)0x20000158 = uVar2;                  // shadow copy
    return 0;
}
```

### The persistence helpers

```c
void Config_Write_U16_0x202(uint v)          // 0x08006C64  — register 44002
{
  if (0x9c4 < (int)v) v = 0x9c4;             // hard clamp to 2500 W
  if (*(ushort *)0x2000012D != v) {          // ← change guard!
      *(short *)0x2000012D = (short)v;
      EEPROM_Write(0x202, 0x2000012D, 2);
  }
}

void Config_Write_U16(uint v)                // 0x08006D30  — register 44003
{
  if (0x9c4 < (int)v) v = 0x9c4;
  if (*(ushort *)0x2000012F != v) {          // ← change guard!
      *(short *)0x2000012F = (short)v;
      EEPROM_Write(0x204, 0x2000012F, 2);
  }
}
```

Two important properties:

1. **Change guard:** A write with the *already stored* value triggers **no**
   EEPROM cycle. Only real value changes cost write cycles.
2. **Hard clamp** to `0x9C4` = 2500 W — in addition to the range check in
   `Write_Handler`.

### It is an external I2C EEPROM, not MCU flash

```c
int EEPROM_Write(offset, src, len)           // 0x08006BBC
{
  if (src == 0) return 1;
  if (xQueueReceive(mutex, 1000) == 0) {
      log_printf(2,3,"Failed to acquire lock during write");
      return 2;
  }
  rc = EEPROM_I2C_WriteBytes(offset, src, len);
  EEPROM_Mutex_Wait(5);
  if (rc != 0) log_printf(1,3,"eeprom write error:%d", rc);
  xQueueSend(mutex, 0, 0);
  return rc;
}
```

`EEPROM_Write` has **54 callers** in the FW and goes exclusively via
`EEPROM_I2C_WriteBytes` — i.e. a separate I2C EEPROM device (bit-bang I2C
routines `I2C_BitBang_*` @ `0x08002014`ff), **not** the internal STM32 flash.
Typical endurance of such a device: ~1 million cycles per cell instead of ~10 k
for MCU flash.

> **Still: do not write cyclically.** There is no wear leveling — 44002 and
> 44003 reside at two fixed addresses. Changing values every 5 seconds
> would result in ~6 million cycles per year on the same cell.

---

## 4. The config block `0x2000012B` (EEPROM `0x200`ff)

The field meanings are **evidenced by the FW itself** — `Battery_Config_Debug_Print`
@ `0x080357FC` outputs the block field by field with plaintext format strings:

```c
void Battery_Config_Debug_Print(void)
{
  printf("charge_cutoff_soc  : %d",    *(byte   *)(base + 0));
  printf("discharge_cutoff_soc: %d",   *(byte   *)(base + 1));
  printf("max_charge_power   : %d",    *(ushort *)(base + 2));
  printf("max_discharge_power: %d",    *(ushort *)(base + 4));
  printf("grid_standand      : %d",    *(byte   *)(base + 6));   // [sic]
}
```

| SRAM offset | Field (FW string) | EEPROM | Modbus register | Write helper |
|---|---|---|---|---|
| `0x2000012B` +0 | `charge_cutoff_soc` | `0x200` | — | — |
| `0x2000012C` +1 | `discharge_cutoff_soc` | `0x201` | — | `Config_Write_DischargeCutoffSOC` |
| `0x2000012D` +2 | **`max_charge_power`** | `0x202` | **44002** | `Config_Write_U16_0x202` |
| `0x2000012F` +4 | **`max_discharge_power`** | `0x204` | **44003** | `Config_Write_U16` |
| `0x20000131` +6 | `grid_standand` (grid standard) | `0x206` | — | — |

### Use as a clamp limit

`Inverter_Power_Setpoint_Calc` @ `0x08006168` clamps the final setpoint exactly
against these two values:

```c
  if ((int)*(ushort *)(0x2000012B + 4) < (int)setpoint)      // upper limit
      setpoint =  (uint)*(ushort *)(0x2000012B + 4);         // = max_discharge_power (positive)

  if (setpoint <= -(uint)*(ushort *)(0x2000012B + 2))        // lower limit
      setpoint = -(uint)*(ushort *)(0x2000012B + 2);         // = max_charge_power (negative)
```

Positive branch → offset +4 → 44003 → **discharging**.
Negative branch → offset +2 → 44002 → **charging**.

> ### ⚠️ Second correction in the Ghidra plate comment
>
> The `Write_Handler` header claimed "REG 44002: discharge_power_max /
> REG 44003: charge_power_max". That is **swapped**. Evidenced by the
> FW's own debug strings *and* the clamping direction:
> **44002 = max. charge power, 44003 = max. discharge power.**
> The existing register map (`44002 max_charge_power`, `44003 max_discharge_power`)
> was thus correct. Ghidra comment corrected on 2026-08-14.

### Further writers of the same block

| Function | What |
|---|---|
| `MQTT_Config_Command_Handler` @ `0x08014278` | Cloud/app sets `0x2000012D` and `0x2000012F` |
| `Grid_Export_Power_Limiter` @ `0x0801F200` | reads `+4` |
| `WorkMode_State_Machine` @ `0x0802D360` | reads `+4` (4×) |
| `Battery_Forced_Charge_Check` @ `0x080052B0` | reads `+4` |
| `CT_PowerSetpoint_Compute` @ `0x0802C8C4` | reads `+2` and `+4` |
| `MQTT_Telemetry_Struct_Builder` @ `0x0801275C` | reads both (telemetry) |
| `Config_Write_PowerOffset` @ `0x08006EE0` | reads both |

> Practical consequence: A change of 44002/44003 via Modbus is
> **equivalent to the setting in the app/cloud** — both end up in the same config
> slot. The app can therefore overwrite the value again at any time.

---

## 5. 45008 — CAN setpoint, also volatile

For completeness, because 45008 is occasionally used as an alternative to 42020/42021:

```c
void Config_PowerSetpoint_Write(int v)       // 0x08005E3A
{
  local = v;
  id = CAN_Build_Arbitration_ID(0x56, 1, 4, 0);
  Register_WriteValue(id, &local, 4);        // pure CAN write, no EEPROM
}
```

45008 thus only sets the live setpoint in the inverter MCU (CAN reg `0x56`).
From the Control MCU's point of view **no persistence**. (Whether the Micro MCU
stores the value itself was not examined in this analysis.)

---

## 6. Recommendation for zero feed-in / HA integration

```python
# Safe to write every second (pure SRAM):
client.write_register(42000, 0x55AA)   # unlock RS485 control (once)
client.write_register(42010, 1)        # 1 = charging, 2 = discharging, 0 = off
client.write_register(42020, 1500)     # charge power in W  (0..2500)
client.write_register(42021, 800)      # discharge power in W (0..2500)

# Do NOT write cyclically — every changed value = one EEPROM cycle:
# client.write_register(44002, 2500)   # max. charge power
# client.write_register(44003, 2500)   # max. discharge power
```

Recommendation for an integration: model 44002/44003 as a **configuration entity**
(e.g. `number` with `mode: box`) and compare against the last read value before
the write — the FW does have a change guard,
but an own guard additionally saves the Modbus round trip.
42010/42020/42021, on the other hand, may be used without concern as control variables.

> **Note:** After every reboot of the device, 42010/42020/42021 are at 0. A
> zero feed-in automation must set the state again after reconnect and
> must not rely on stored values. The
> RS485 unlock (42000 = `0x55AA`) is also required again after a reboot.

---

## 6a. What 42000 really is — and what happens when it falls back

*(2026-08-25, Write_Handler v150 lines 341–380 plus measurement on the device.)*

`Modbus_RS485_TCP/Scan_Logs/Write_Handler_Register_Map.csv` already notes "shares SRAM byte with reg 43000".
The decompiled code shows what that means — both registers write
the same variable `DAT_080525c4[1]`:

```c
// 42000
if (param_4 == 0x55aa) { DAT_080525c4[1] = 0x0A; return 0; }        // "Unlock"
if (param_4 == 0x55bb) { EEPROM_Read(0x301, DAT_080525c4+1, 1); }   // back from EEPROM

// 43000 — same variable
if (read) { byte==0x01 ? 0 : byte==0x02 ? 2 : 1 }
if (param_4 == 0) byte = 0x01;   // self-consumption
if (param_4 == 1) byte = 0x00;   // anti-feed-in
if (param_4 == 2) byte = 0x05;   // trading
EEPROM_Write(0x301, ...);
```

So there is **no separate RS485 control mode**. There is a mode byte that is
either at `0x0A` — then 42000 reads `0x55AA` — or at a normal
operating mode. None of the three options of 43000 writes `0x0A`.

**Measured effect (Venus D, 2026-08-24):** with forced discharge running
at 605 W, switching the byte away from `0x0A` → discharge drops to **12 W**
(inverter's own consumption). `force_mode` (42010) is thus only obeyed
as long as the byte is `0x0A`.

**The write handler does not check the byte in the process.** The branches for 42011 (`0xA41B`)
and 42020 (`0xA424`) validate only the value range and then write.
Write commands are thus still acknowledged while the control ignores them — the failure is invisible
from the outside.

**Practical consequence for integrations:** whoever switches 43000 thereby ends
RS485 control; whoever sets 42000 to `0x55AA` takes the device out of the
configured operating mode (43000 then reads `1`, because `0x0A` falls into the
else branch of the read handler). Two controls, one state.

**Open:** why the byte falls back to the EEPROM value on its own in the field.
Observed on Venus E v3 (three times, each together with a
communication disturbance) and on Venus D (once, about one minute after a
restart of the Modbus client). The CH395 reset from §4a of the v150 Control analysis
is not the cause — it only affects the network side.

---

## 7. Reproduction

```
get-decompilation  programPath=/VNSD-0_app_0150_0805_115146.bin
                   functionNameOrAddress=Write_Handler          # 0x08051D14, 968 lines

find-cross-references  location=0x2000034c  direction=to        # 42020 → only Write_Handler + reader
find-cross-references  location=0x2000034e  direction=to        # 42021 → ditto
find-cross-references  location=0x2000012d  direction=to        # 44002 → 13 refs, EEPROM path
find-cross-references  location=0x2000012f  direction=to        # 44003 → 20 refs, EEPROM path

get-decompilation  functionNameOrAddress=Config_Write_U16_0x202 # EEPROM 0x202 + change guard
get-decompilation  functionNameOrAddress=Config_Write_U16       # EEPROM 0x204 + change guard
get-decompilation  functionNameOrAddress=EEPROM_Write           # → EEPROM_I2C_WriteBytes
get-decompilation  functionNameOrAddress=Battery_Config_Debug_Print   # field names in plaintext
get-decompilation  functionNameOrAddress=Inverter_Power_Setpoint_Calc # clamping direction
get-decompilation  functionNameOrAddress=Remote_Power_Command_Execute # sign convention
```

Literal pool resolution (little-endian, `read-memory`):

```
0x0805259C → 48 03 00 20  →  0x20000348   (block 41000/42010/42011/42020/42021)
0x080529BC → 56 01 00 20  →  0x20000156   (shadow copy 44002/44003)
0x08006C94 → 2B 01 00 20  →  0x2000012B   (config block, EEPROM 0x200ff)
0x08006D60 → 2B 01 00 20  →  0x2000012B   (ditto)
0x080063D8 → 2B 01 00 20  →  0x2000012B   (clamp limits in Setpoint_Calc)
```

---

## 8. Open items

1. **Micro MCU side of 45008 / CAN reg `0x56`** not examined — does the
   inverter firmware store the setpoint non-volatilely itself?
2. **Live verification:** set device to 42020 = 1234, power cycle, read back 42020
   → expected **0**. If 1234 is there, there is a persistence path not found so far.
3. **Overall EEPROM map:** The offsets `0x200`–`0x206` are hereby evidenced.
   A complete EEPROM map (incl. `0x300`/`0x301`, `0x374`, `0x387`,
   `0x394`, `0x3700`ff, `0x500`, `0x2000`ff, `0x4000`) is still missing.
4. **Register 43513** (schedule power): check for persistence. (Correction 2026-10-01: 41600ff is **not** a schedule block — it is the write-only Wi-Fi password buffer `g_wifiPassword`, see `VNSD_Register_Map_RAM_Addresses.md`; the schedule block persisted via `EEPROM_Write` at `0x3700`ff is the 46500 block.)

---

## Addendum 2026-08-22: Backup reserve / depth of discharge — no Modbus access

The app setting **"Battery depth of discharge (DOD)"** (screenshot: "30 % discharged
energy / 70 % backup capacity", setting range 30–88 %) can be neither read nor written via Modbus. Complete chain:

| Location | Content |
|---|---|
| SRAM `0x2000012C` | Backup reserve in percent (in the screenshot: 70) |
| EEPROM `0x201` | persistent storage, 1 byte |
| App/BLE display | `100 − reserve` = depth of discharge |

**Setter** `Config_Write_DischargeCutoffSOC` @ `0x08006d64`:

```c
if ((param_1 == 0) || (0x1d < param_1 && param_1 < 0x59)) {   // 0 or 30..88
    cVar2 = (param_1 == 0) ? 12 : ('d' - param_1);            // 100 - input
    *(char *)0x2000012C = cVar2;
    EEPROM_Write(0x201, ..., 1);
    return 0;
}
return 1;                                                      // value rejected
```

The accepted range 30–88 matches the app's hint text exactly.
Input 0 sets the default 12 (= 88 % depth of discharge).

**Reachable via exactly three paths** (`Config_Set_DischargeCutoff_WithRelay`
@ `0x0802dce4`, cross-reference run over `0x2000012C`):

| Caller | Route |
|---|---|
| `MQTT_JSON_RPC_Dispatcher` @ `0x0801ab6c` | Cloud |
| `FUN_08002538` | BLE command (`DEPTH_OF_DISCHARGE_CONTROL` `0x54`) |
| `FUN_08007674` | Text console, `atoi` via a queue |

**No `Write_Handler`** — and thus no Modbus register, not even any of the still
unnamed ones. The write handler map (172 registers) contains not a single
SRAM target value in the config block `0x2000012B..0x20000131`.

Additionally, the boot routine `FUN_08004b5c` validates the same block:

```
+0  Percent value     0 or >100          -> 100
+1  Backup reserve    0, >70, <12        -> 12
+2  Charge power      0 or >2500         -> 2500
+4  Discharge power   0 or >2500         -> 800
```

The clamping 12..70 on `+1` explains the app range 30–88 as `100 − reserve`.

### Delimitation from register 42011

42011 (`charge_to_soc`, "Maximum SoC" in the HA integration) is **not** this
parameter. Verified on the device: setting it to 70 immediately triggers a discharge that
stops at SoC 70 — a target limit, not a discharge floor. Value range 13–100,
volatile SRAM, 0 after restart. Do not confuse the two.

---

## Correction 2026-09-20 — Read path of 44002/44003

The table above lists "SRAM + EEPROM" for 44002/44003 and the function header of
`Write_Handler` names "SRAM mirror 0x2000012D" and "0x2000012F" respectively. That describes the
**write path** correctly, but not the **read path**.

In read mode, `Write_Handler` returns `DAT_080529bc`, and this resolves to **0x20000156**
— a separate shadow value, not the config block. The shadow is not initialized anywhere,
which is why FC03 on 44002/44003 returns 0 after every restart, while the
limiting itself continues to work correctly (it reads the config block `0x2000012B`, which is
loaded and validated from EEPROM `0x200` at startup).

Evidenced on the device and fully written up in
`VNSD_FW_Bug_44002_44003_Readback_After_Restart.md`.
