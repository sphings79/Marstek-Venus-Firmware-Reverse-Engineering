# BLE Command Map — Marstek Venus (Control FW v150)

> 🇩🇪 Deutsche Version: [VNSD_BLE_Command_Map_v150_de.md](VNSD_BLE_Command_Map_v150_de.md)

> **Source:** `BLE_Cmd_Dispatch` @0x08007F20 (v150), cleanly decompiled.
> The region was not recognized as a function in Ghidra → defined via `create-function`,
> after which the decompiler resolves the switch. **Command ID = Byte[3]** of the packet.
> **Status:** 2026-08-14

## Packet format

```
73 | LEN | 23 | CMD | payload… | XOR
```
- `73` magic, `LEN` total length, `23` fixed, `CMD` = command ID (Byte[3]), `XOR` = XOR over all preceding bytes.
- The payload starts at Byte[4]. In the tool = the bytes that the `*Payload` class serializes.
- Multi-byte numbers are **little-endian**, unless noted otherwise.

## Legend

**Tool:** ✓ = present in venuscontrol, ❌ = missing.

**Effective?** — does the written value actually take effect in operation? (Method: cross-reference on the *live SRAM config variable*, not just on the handler. If only the handler itself + a telemetry builder read it → "set-and-report". If a task/control function reads it → functional.)
- ✓ = **effective** — value is used in the control path (code-verified)
- ⚠ = **not effective** — set-and-report only (only its own handler + telemetry read the value)
- 📖 = read command (returns data)
- ? = not verified

## Full command table (v150)

| CMD | Feature | Payload (from Byte[4]) | Effective? | Tool |
|---|---|---|---|---|
| 0x02 | Set server type (cloud server 0–4) | u8 | ? | ❌ |
| 0x03 | Get work status info (STATE) | – (read) | 📖 | ✓ |
| 0x04 | Device info | – (read) | 📖 | ✓ |
| 0x05 | Info block (resp 0x20) | – | 📖 | ❌ |
| 0x06 | Factory reset | u8 (1/2/3) | ✓ | ✓ |
| 0x08 | Read wifi name | – (read) | 📖 | ❌ |
| 0x09 | Set work mode / economy | u8 mode (+ slot data for mode=1, economy for =5) | ✓ | ✓ |
| 0x0A | Get set info | – (read) | 📖 | ✓ |
| 0x0B | **Sys time set (RTC)** | year(u16 BE, -2000), mon,day,hour,min,sec | ✓ | ✓ |
| 0x0C | Set develop mode | magic 0A 0B 0C + u8 (0..6) | ? | ❌ |
| 0x0D | Develop mode info | – (read) | 📖 | ❌ |
| 0x0E | Set work mode auto change | u8 (EEPROM 0x366) | ⚠ | ❌ |
| 0x0F | Set EPS/backup enable | u8 (0/1, EEPROM 0x300) | ✓ | ✓ |
| 0x10 | OTA start | magic + u8 | ✓ | (OTA) |
| 0x11 | Query modem version (AT+QVERSION) | magic | 📖 | ❌ |
| 0x12 | EEPROM clear | magic | ✓ | ❌ |
| 0x13 | Get err code info | – (read) | 📖 | ❌ |
| 0x14 | Get bms data info | – (read) | 📖 | (0x42 in the tool) |
| 0x15 | **Device power class 800/2200/2500** (Config_Write_U16, EEPROM 0x90/0x204, clamps schedules at 800) | u16 ∈ {800, 2200, 2500} | ✓ | ✓ |
| 0x16 | **Max charge power** (EEPROM 0x202, 300–2500 W) | u16 W | ✓ | ✓ |
| 0x17 | **Max discharge power** (EEPROM 0x204, 0–2500 W) | u16 W | ✓ | ✓ |
| 0x18 | CT type (meter) | u8 | ✓ | ✓ |
| 0x19 | CT mode | u8 (EEPROM 0x367) | ✓ | ✓ |
| 0x1A | CT readings | – (read) | 📖 | ✓ |
| 0x1B | Set url port / OTA URL | subcmd + data | ? | ❌ |
| 0x1C | Get event log info | – (read) | 📖 | ❌ |
| 0x1D | Phase autodetection | – | ✓ | ✓ |
| 0x1E | Config U8 | u8 | ? | ❌ |
| 0x1F | Info active upgrade | magic | 📖 | ❌ |
| 0x20 | **Set parallel machine** | magic + u8 (0/1/2) | ✓ | ❌ |
| 0x21 | **Set/Read meter IP** | 0x0A=set+string / 0x0B=read (EEPROM 0x3500) | ⚠ | ❌ |
| 0x22 | Config Byte 0x36b | u8 | ? | ❌ |
| 0x23 | **Set generator enable/disable** | u8 (0/1) | ⚠ | ❌ |
| 0x24 | VID read | subcmd | 📖 | ❌ |
| 0x25 | Config U16 0x36e | u16 (BE) | ? | ❌ |
| 0x26 | Config capacity factor | u8 | ? | ❌ |
| 0x27 | Power data batch | u8 count | 📖 | ❌ |
| 0x28 | **Set local API enable + port** (local API = UDP JSON-RPC, **NOT** Modbus TCP; EEPROM 0x371/0x372) | u8 enable + u16 port | ✓ | ✓ |
| 0x29 | **PEAK SHAVING** | u8 peak_state + int16 power(W) | ✓ | ✓ |
| 0x40 | Config reg offset3 | u8 | ? | ❌ |
| 0x41 | Surplus feed-in (surplus) | u8 (0/1, EEPROM 0x375) | ✓ | ✓ |
| 0x42 | Battery modules state | u8=1 (read) | 📖 | ✓ |
| 0x43 | Apply work-mode reg | 0x01 + u8 | ? | ❌ |
| 0x50 | Write vid/xid info | subcmd + string | ✓ | ❌ |
| 0x51 | Read vid/xid info | subcmd | 📖 | ❌ |
| 0x52 | OTA commands | sub-typed (0x0A–0x0D) | ✓ | (OTA) |
| 0x53 | Set BLE adv enable | 0x0A/0x0B + u8 (EEPROM 0x36bd) | ✓ | ✓ |
| 0x54 | Depth of Discharge | (FUN_08002538) | ✓ | ✓ |
| 0x55 | **Set self-control power offset** | int16 (EEPROM 0x383) | ✓ | ✓ |
| 0x59 | LED control | u8 (1=OPEN/0=Close) | ✓ | ✓ |
| 0x5A | Enable flag | u8 (=1) | ? | ❌ |
| 0x5B | Config read/apply | 0x01 → resp 0x3a | 📖 | ❌ |

## "Not effective" — set-and-report-only commands (details)

These commands **store** the value (and report it back in telemetry/app), but **no control or task function reads it** — without effect in v150 operation. That is why they are deliberately **not** exposed in the tool:

| CMD | Feature | Why without effect |
|---|---|---|
| 0x0E | Work-mode auto change | Live var `0x20014D66` is read only by its own handler + telemetry. |
| 0x21 | Set meter IP | Var (EEPROM 0x3500) is touched only by its own handler; the connection IP is **auto-discovered** at runtime — consistent with the fact that the meter IP cannot be set in the app either. |
| 0x23 | Generator enable/disable | Live var `0x20014D6C` is read only by telemetry (no charge/discharge path evaluates the generator flag). |

## Verification evidence for the "effective" rating (selection)

- **0x55 Self-Control Offset →** `CT_GridPower_Controller @0x0802c680` reads `0x20014D83`:
  `local_10 = grid_power - *(short*)(config+0x83)` → the controller regulates `grid_power → offset` instead of to 0 W. Callers: `WorkMode_ChangeHandler`, `TimePlan_Evaluate_Setpoint`. **Functional.**
- **0x28 Local API →** Live var `0x20014D71/72` is read by the UDP server task (`CH395_UDP_ServerTask` / `Quectel_UDP_CommStateMachine`) → actually switches the local JSON-RPC server. **Functional.**
- **0x29 Peak Shaving →** `Config_Write_PowerClampMode_0x394` (EEPROM 0x394) → `Inverter_PowerSetpoint_DeadbandClamp` clamps the power to the threshold. **Functional.**
- **0x0B Time →** direct `RTC_SetDateTime` call. **Functional.**
- **0x15/0x16/0x17 Power limits →** feed into the setpoint/schedule clamping. **Functional.**

## Peak Shaving (0x29) — detail

```c
case 0x29:
  peak_state = byte[4];               // 1 = on, 0 = off
  peak_power = int16_LE(byte[5..6]);  // power in W (peak threshold)
  Config_Write_PowerClampMode_0x394(peak_state, peak_power);  // EEPROM 0x394
  // Effect: Inverter_PowerSetpoint_DeadbandClamp clamps feed-in/discharge power to the threshold
```

Example frame "Peak Shaving on, 600 W":
`73 08 23 29 01 58 02 <XOR>`  (600 = 0x0258 → LE: 58 02)

## Self-Control Power Offset (0x55) — detail

```c
case 0x55:
  Config_Write_PowerOffset((int)int16_LE(byte[4..5]));   // clamp against charge/discharge max, EEPROM 0x383
  // Effect: CT_GridPower_Controller computes  regel_fehler = netzleistung - offset
  //          → self-consumption regulates to 'offset' W grid import instead of 0 W
```

## Implementation status in the tool (venuscontrol)

**Newly integrated & code-verified (all "effective"):**
- 0x29 Peak Shaving · 0x15 Device power class · 0x17 Max discharge power (bug fix: was wrongly mapped as 0x15) · 0x28 Local API · 0x0B Set time · 0x55 Self-Control Offset

**Deliberately not integrated (not effective, see above):** 0x0E Auto work-mode change · 0x21 Meter IP · 0x23 Generator

**Still open / low priority:** 0x20 Parallel operation (effective, but niche setup), various dev/config commands (0x0C, 0x1B, 0x1E, 0x22, 0x25, 0x26, 0x40, 0x43, 0x5A) — unverified.
