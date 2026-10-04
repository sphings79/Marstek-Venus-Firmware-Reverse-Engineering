# Command Register Block 45000–46000 (Control FW v150)

> 🇩🇪 Deutsche Version: [VNSD_Command_Register_45000_Block_v150_de.md](VNSD_Command_Register_45000_Block_v150_de.md)

> **Firmware:** `VNSD-0_app_0150_0805_115146.bin` (v150) · ARM Cortex-M, FreeRTOS
> **Source:** Static analysis of `Write_Handler` (`FUN_08051d14`, 0x08051d14) in Ghidra
> **Status:** 2026-08-14
> **Addressing:** direct (PDU = register number, no offset) · write via **FC06/FC10**

---

## 1. What This Block Is

The registers **45000–46000** are **not** a normal control register set like
42010 (`force_mode`) or 43000 (`work_mode`), but a **service/engineering
command interface**. Almost every command translates a Modbus write into an
**internal CAN bus message to the inverter MCU** — built via
`CAN_Build_Arbitration_ID(funccode, …)` + `Register_WriteValue()`. A few
commands instead switch **local hardware** (GPIO pins, relays, CH395
network chip, BLE) or trigger **OTA/EEPROM**.

Properties of the block:

- **Write-only** (reading → Modbus exception `2`), except for the three read registers
  45603/45604/45605.
- Invalid values → exception `3`.
- Many functions **overlap** with the official control registers
  (42010/42020/43000) — only at a lower, more direct level.
- **Several are dangerous** (OTA, relay, GPIO, SMR ADC) and not to be used
  without understanding the hardware.

### CAN Function Codes (observed)

| CAN func | Meaning | used by |
|---|---|---|
| 0x01 | Inverter on/off (class 2: no receiver on the Venus D) | 45021, (45020, 45000) |
| 0x06 | Flag register 0x06 | 45002 |
| 0x50 | Feature enable | 45000 |
| 0x52 | WorkMode (1 byte) | 45004 |
| 0x55 | Config param 55 | 45007 |
| 0x56 | Power **setpoint** (u32) | 45008 |
| 0x57 | Flag 0x57 (on/off) | 45009 |
| 0x58 | Manual power **setpoint** (u32, mode 1 only; not a limit) | 45010 |
| 0xC1 | Counter reset (frame has no receiver; 45001 is not a BMS reset, see section 8) | 45001 |
| 0xFB | Reg 0xFB | 45022 |
| 0xFE | Reg 0xFE | 45023 |

---

## 2. ⚠️ Address Drift 149.2 → v150

The earlier `Modbus_RS485_TCP/Scan_Logs/Write_Handler_Register_Map.csv` was derived from **FW 149.2**.
The `FUN_` target addresses named there (e.g. `FUN_08005f94`, `FUN_08005ccc`,
`FUN_08002e10`) are **shifted** in v150 and point to different functions there.
The `Write_Handler` itself has changed (Ghidra similarity 149.2→150 = 0.629).

**Register numbers and basic meaning remain valid** — the concrete function
names/addresses below come from v150. The CSV has been corrected accordingly.

In addition: the **comment header in the binary `Write_Handler`** names the read
registers incorrectly (says "45539–45541" and "45597/0xB28D"). The actual code
addresses **45603/45604 (0xB223/0xB224)** and **45605 (0xB225)** — matching the
register map CSV, not the comment.

---

## 3. Command Table (v150-verified)

| Reg | Name | Values | Effect | Backend |
|---|---|---|---|---|
| 45000 | cmd_micro_factory_mode | `0x55EE` / `0x55BB` (otherwise err 3) | `0x55EE` → Micro `g_workMode` = 12 (factory mode), `0x55BB` → mode 1 (manual); both also send class-2 "off" (no receiver on D); status byte=2 (corrected, see section 8) | CAN 0x50 |
| 45001 | cmd_inverter_eeprom_test | any | Micro EEPROM write/read-back self-test (result in 30213); **not** a BMS reset (corrected, see section 8) | CAN 0xC1 |
| 45002 | cmd_function_1 | any | `Inverter_Set_Flag_Reg_0x06(1)` | CAN 0x06 |
| 45003 | cmd_function_2 | any | `Config_Param51_Reset()` | CAN |
| 45004 | **cmd_mode_select** | `1`/`2`/`3` | `Config_WorkMode_Set(1/2/3)` — value 1:1 as mode | CAN 0x52 (1 B) |
| 45005 | cmd_function_3 | any | `Config_Param53_Activate()` | CAN |
| 45006 | cmd_function_4 | any | `Config_PostWriteCommit()` | CAN |
| 45007 | cmd_function_5 | any | `Config_Param55_Set(1)` | CAN 0x55 |
| 45008 | **cmd_set_power_a** | 1–2500 W; >32767 = neg. (value−65535); 0/2501–32767 ignored | `Config_PowerSetpoint_Write()` — **setpoint** ±2500 W | CAN 0x56 (u32) |
| 45009 | cmd_enable_disable_a | `0`/`1` | `Inverter_Set_Flag_Reg_0x57(0/1)` | CAN 0x57 |
| 45010 | **cmd_manual_power_setpoint_w** | like 45008 | `Inverter_Set_PowerLimit_Reg_0x58()` — the real manual power **setpoint** ±2500 W (+ discharge / − charge), effective **only in manual mode 1** (after 45000=`0x55BB`), otherwise acknowledged and ignored; not a limit (corrected, see section 8) | CAN 0x58 (u32) |
| 45011 | **cmd_mode_select_ext** | 1–9 (>9 err 3) | `WorkMode_Register_Write()` with remap **1→1, 2→2, 3→4, 4→0, 5→5, 6–9→0** | 8-B reg write |
| 45012 | cmd_led_lamp_test_ms | time in ms | front-panel LED lamp test (all LEDs on, active-low) for the given time in ms; no power effect (corrected, see section 8) | local HW (GPIO) |
| 45020 | cmd_class2_unit1_on_and_power_w | 1–3999 (otherwise no action) | class-2 frames (unit on + power command up to 3999 W); the Venus D Micro filters class 2, no receiver on D (corrected, see section 8) | CAN class 2 |
| 45021 | cmd_class2_unit1_on_off | `0`/`1` | class-2 on/off frame, no receiver on D; "SMR" in the firmware is unrelated (corrected, see section 8) | CAN 0x01 |
| 45022 | cmd_class2_unit1_reg_0xFB | any | `Inverter_Set_Reg_0xFB(1)` — class-2 frame, no receiver on D | CAN 0xFB |
| 45023 | cmd_class2_unit1_reg_0xFE | any | `Inverter_Set_Reg_0xFE(1)` — class-2 frame, no receiver on D | CAN 0xFE |
| 45024 | cmd_class2_unit1_reg_0xFF | any | `Inverter_Set_Reg_0xFF(1)` — class-2 frame, no receiver on D (earlier reading `Inverter_RS485_Cmd_Reset` / RS485 is wrong) | CAN 0xFF |
| 45025 | dbg_gpio_pb10_pack_encode_line | `1`=set / `0`=clear | GPIO bit **0x400** (PB10); overwritten again within ≤10 ms in normal operation, persists only in test mode / key sequences | local HW |
| 45026 | dbg_gpio_pe15_button_led | `1`/`0` (**inverted!**) | GPIO bit **0x8000** (PE15), logic swapped compared to 45025; rewritten every 10 ms in idle, cosmetic | local HW |
| 45027 | cmd_bms_master_reboot | any | `CAN_SendTriggerCategory0x0A(1)` — class-3 frame → **immediate reboot of the BMS master pack** (not a Wi-Fi/network reset, not harmless; corrected, see section 8) | CAN class 3 → BMS |
| 45028 | cmd_bms_factory_reset | any | BMS **factory reset + reboot of all packs** (broadcast ID; not a relay; corrected, see section 8) | CAN → BMS |
| 45029 | cmd_battery_pack_encode_start | `1`/`0` | starts the BMS pack address encoding / master claim (`g_encodeFlag`); status 2 (success) only with exactly 2 packs online (corrected, see section 8) | CAN → BMS |
| 45030 | cmd_bms_syscmd_0x0B_0x0C | `1`=on / `0`=off | `Config_Apply_SingleReg(0xB)` resp. `(0xC)` — **functionless on D** (no receiver handles 0x0B/0x0C; not a BLE enable; corrected, see section 8) | none |
| 45031 | cmd_bms_restart_handshake | only `1` | ack condition impossible with BMS 117.7/118: **functionless on D**; only effect is a ~5 s Modbus task stall + exception 3 (corrected, see section 8) | none |
| 45603 | adc_version_bcd_ch1 | **read-only** | flush queues, `Inverter_Clear_Reg_0x13()`, wait 200 ms, **BCD** version channel 1 (`DAT_08052de8`) | Queue/ADC |
| 45604 | adc_version_bcd_ch2 | **read-only** | like 45603, channel 2 (`DAT_08052dec`) | Queue/ADC |
| 45605 | status_query | **read-only** | `CH395_SPI_ReadByte()` — status byte from the **CH395 network chip** via SPI | SPI |
| 46000 | **ota_command** | `0x5100`=EEPROM flag (addr 0x900); `0x4D2`(1234)=mode 1; `0x929`(2345)=mode 4; `0xD80`(3456)=mode 3; `0x11D7`(4567)=mode 2 | OTA mode flag `DAT_08052df4` resp. EEPROM flag `DAT_08052df0` | OTA/EEPROM |

> Registers **45013–45019** are reserved and return exception `2` (illegal data address).

---

## 4. The Three "Set-Power" Variants

Visually identical (all 1–2500 W, signed), but **three different backends** (⚠️ the semantics column below is superseded by section 8: only 45010 is a real, manual-mode power setpoint; 45008 is a factory test in mode 12; 45012 is an LED lamp test):

| Reg | Function | Path | Semantics |
|---|---|---|---|
| 45008 | `Config_PowerSetpoint_Write` | CAN func **0x56** | Power **setpoint** to inverter |
| 45010 | `Inverter_Set_PowerLimit_Reg_0x58` | CAN func **0x58** | Power **limit** |
| 45012 | `Inverter_Power_Setpoint_Apply` | **I2C** register sync | separate I2C path, default fallback 2000 W |

This explains the apparent redundancy: setpoint vs. limit vs. I2C-coupled path.

---

## 5. Mode Selection: 45004 vs. 45011

- **45004** (`cmd_mode_select`): sends value 1/2/3 directly via CAN 0x52. Only 1–3 valid.
- **45011** (`cmd_mode_select_ext`): accepts 1–9, **but remaps** and writes
  an 8-byte WorkMode record. The internal mode code set is **not** congruent
  with the input:

  | Input | 1 | 2 | 3 | 4 | 5 | 6–9 | >9 |
  |---|---|---|---|---|---|---|---|
  | internal | 1 | 2 | **4** | **0** | 5 | 0 | Error 3 |

---

## 6. Danger Notice

These registers are factory/debug functions without misuse protection:

- **46000 (OTA)** — can trigger the update process.
- **45028 (BMS factory reset + reboot of all packs, see section 7) and 45027 (immediate reboot of the BMS master pack)**; 45031 blocks the Modbus task ~5 s (functionless otherwise). 45001 is **not** a BMS reset (section 8).
- **45020 (class-2 unit on + power command)** — no receiver on the Venus D (section 8).
- **45025/45026 (GPIO)** — direct pin intervention (rewritten every 10 ms); 45023/45024 are class-2 frames without receiver on D; 45027 is *not* a Wi-Fi reset but a BMS master reboot.

On production devices use only the clearly understood registers (mode/power/BLE/version)
— their function is largely covered anyway by the official registers
42010/42020/43000.

---

## 7. Corrections from the v151 code analysis (2026-10-01)

> ⚠️ Superseded in parts by section 8 (register resolution of 2026-10-01); where they differ, section 8 and the CSV apply.

Re-checked against the `Write_Handler` of **v151** (`0x080570e0`), starting from the register number. Where the v150 reading
above differs, the v151 code shows the following (the v150 handler was not re-read for these registers; `VNSD_Register_Map_Final_generated.csv`
carries the new names and the old ones as `frueher: …`):

| Reg | Earlier name / reading | v151 code |
|---|---|---|
| 45000 | `cmd_system_control` | test mode: `0x55EE`/`0x55BB` write CAN reg 0x50 (Micro `g_workMode` 12 resp. 1) and send class-2 "off"; `g_maintModeState = 2`. Name now `cmd_micro_factory_mode`. Read → exception 3 |
| 45020 | `cmd_smr_relay_adc` (SMR/ADC) | class-2 **unit on + power command**: value 1…3999 W → `Inverter_Set_OnOff_Reg_0x01(1,1)`, then `Inverter_CAN_PowerCmd_Send(1,1,value, …)` with charge voltage and temperatures as extra fields; **no receiver on the Venus D** (class 2 is filtered by the Micro). Name now `cmd_class2_unit1_on_and_power_w` |
| 45021 | `cmd_smr_enable` | plain class-2 on/off (`Inverter_Set_OnOff_Reg_0x01`); "SMR" is wrong (unrelated firmware term); no receiver on D. Name now `cmd_class2_unit1_on_off` |
| 45023 / 45024 | `cmd_gpio_debug_1/2` (GPIO) | CAN commands to inverter registers **0xFE** and **0xFF** (`Inverter_Set_Reg_0xFE/0xFF(1)`), no GPIO access |
| 45025 / 45026 | GPIO 0x400 / 0x8000 | still GPIO, but the pins/ports are **PB10 (0x400)** and **PE15 (0x8000)**; polarity is swapped between the two (45025: 1 = reset bit, 45026: 1 = set bit) |
| 45027 | `cmd_wifi_reset` | `CAN_SendTriggerCategory0x0A(1)` – a class-3 CAN trigger, not a Wi-Fi reset; reboots the BMS master pack. Name now `cmd_bms_master_reboot` |
| 45029 | `cmd_debug_flag` | `g_encodeFlag`: status byte of the key/pairing "encode" sequence (0 idle, 1 requested, 2 done when the BMS online mask == 3, 3 timeout); also readable via 30110/32114. Name now `cmd_battery_pack_encode_start` |
| 45603 / 45604 | `adc_version_bcd_ch1/2` (ADC) | answer of a CAN request (reg 0x13) delivered as BCD via two queues (`g_regGetOldQueueLo/Hi`) |
| 45605 | `status_query` | **CH395 chip version** (`CH395CMDGetVer`) |

Unchanged/confirmed: 45002–45012 (CAN registers 0x06, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58 and the 8-byte mode record).

⚠️ **45028 is not a relay.** It sends byte 2 on CAN id `0x100101AA`; BMS 118 treats this as **factory reset + reboot** (`APP_Factory_Reset_And_Reboot`). Any written value does it, without authentication. **45001** does **not** trigger this reset (the earlier fall-through claim was wrong, see section 8). Name of 45028 is now `cmd_bms_factory_reset`.
**45031** blocks the Modbus task for about 5 s and returns exception 3 (functionless on D). **45030** (BLE enable, `Config_Apply_SingleReg(0x0B/0x0C)`) is present in all versions v147–v151; the v151 handler is not new here, and BMS 118 does not handle the bytes 0x0B/0x0C.
New in v151: register **47001** (`0xB799`, `can_cat0x18_command`): any u16, low byte sent as an 8-byte CAN frame (id `0x00014118`, function 0x18, node 1), no EEPROM; same sink as the OTA target selection in 46000 and MQTT debug 3/4. No receiver for function 0x18 was found in Micro 116 or BMS 118. Reading gives exception 2.
In v147–v150 the 45020/45023/45024/45027/45029 branches behave like in v151 (checked by emulating each version's handler).

**Frame classes (added 2026-10-01).** The Control firmware sends its CAN frames in two main classes, distinguished by the destination nibble (CAN-ID bits 12–15) built in `CAN_Build_Arbitration_ID`: **class 4** (set power reference/limit, work mode, 0x50/0x52–0x58, 0x60, 0x18, schedule 0x30+idx, EEPROM-related commands, `Config_Mode_Apply` = backup mode) and **class 2** (`Inverter_Set_OnOff_Reg_0x01`, `Inverter_CAN_PowerCmd_Send`, 0x04, 0xFB, 0xFE, 0xFF, work-mode reg 0x06, date/time broadcast 0xCC); the relay trigger 0x0A (45027) uses class 3, which the Micro filters but the BMS accepts. The **Venus D and Venus E3 micro firmware** (115/116, 119) set their CAN filters to `(ID & 0xFF00) == 0x4000/0x4100` plus the BMS range, so **class-2 and class-3 frames never reach their dispatcher**. Consequence for the D: 45020/45021 (inverter on/off + power command), 45022–45024 and the on/off part of 45000 have no effect on the inverter; the Venus A micro (1211) has a handler for 0xFE. Class-4 commands that do reach the D micro: 0x50 (45000: sets Micro `g_workMode` to 12 resp. 1 plus a power-stage test code, see section 8), 0x52–0x58 (test/engineering states; 0x58 = manual power setpoint of 45010, mode 1 only), 0x60, `Config_Mode_Apply` (41200, stored backup-mode flag at EEPROM 0x300).

---

## 8. Corrections from the register resolution of 2026-10-01

Full analysis of every register: [VNSD_Unclear_Registers_Resolution_2026-10-01.md](VNSD_Unclear_Registers_Resolution_2026-10-01.md) (English). Wording follows `befund_D` in `VNSD_Register_Map_Final_generated.csv`. Where sections 1–7 differ, this section applies.

| Reg | Correction |
|---|---|
| 45000 | `0x55EE` writes **Micro `g_workMode` = 12** (factory mode), `0x55BB` selects **mode 1** (manual); in both modes EMS power setpoints are ignored. Both branches also send class-2 "off". Stops output until a Control reboot (a Control boot sends CAN cmd 5, which reboots the Micro; this is why 41000 restores it). Name `cmd_micro_factory_mode`. Never poll (read → exception 3, TCP connection stalls). |
| 45001 | **Micro EEPROM write/read-back self-test** (result in 30213), one Micro EEPROM byte written per call. **NOT a BMS reset**: the reset-flag frame (`0x100701AA` in v147–149.2, `0x180701AA` in v150/151) has no receiver, and the code has no fall-through to the BMS reset. Name `cmd_inverter_eeprom_test`. |
| 45004 / 45008 | Factory tests (open-loop test, grid-tie power test at P W); only after 45000=`0x55EE` (mode 12), otherwise acknowledged and ignored. |
| 45010 | The **real manual power setpoint** (+ discharge / − charge), **only in manual mode 1** (after 45000=`0x55BB`); in normal mode 0 it is acknowledged and ignored. Not a "limit". Negative values arrive 1 W short; 0 W cannot be commanded. Bypasses SoC cut-off, EPS reserve, error mask and charge/discharge limits; persists until a Control reboot. Name `cmd_manual_power_setpoint_w`. |
| 45011 | Functionless on the D (no receiver). |
| 45012 | **Front-panel LED lamp test** (all LEDs on, active-low, for v ms), same as the shell command `led_continue_ctrl`; **not a power setpoint**, no I2C power path. Name `cmd_led_lamp_test_ms`. |
| 45013–45019 | Explicit exception 2 in all five Control versions; no other code references them. The Micro's own 45010–45019 block is a separate register space on the Micro's own RS485 port and is not forwarded by the Control. (The TCP exception frame carries a wrong MBAP length, see `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20.md`.) |
| 45020–45024 | **Class-2 frames** (`cmd_class2_unit1_*`; 0xFB / 0xFE / 0xFF registers, on/off, on + power up to 3999 W). The Venus D Micro (115/116) filters class 2 and the BMS has no class-2 path, so **there is no receiver on the D**. The receiver is a separate unit, most likely an MPPT/PV controller (per `g_mppt*` naming); its firmware is not available, so "no receiver" is proven only for the firmware we have. "SMR" is unrelated (`SMR_TIC_SelfUse_PowerController` / HW tag `SMR-`). 45024 is not an RS485 command. Status FUNCTIONLESS. |
| 45025 / 45026 | GPIO PB10 / PE15 (PB10 inferred as pack encode/handshake line, PE15 inferred as power-button LED). **Rewritten every 10 ms** in normal operation, so practically no-op; a write only sticks in test mode (after 45000) or in key/latch states. Status CONDITIONAL. |
| 45027 | **NOT harmless / not a Wi-Fi reset**: class-3 frame (the BMS command class; only the Micro filters it) → **immediate reboot of the BMS master pack** (node 1 = slot 1; packs with another slot ignore it). A pack that became master at runtime additionally gets EEPROM 0x202 = 0xFF (self-claim master). Battery disconnects during the reset. Name `cmd_bms_master_reboot`. The statement "no effect on the D" in older documents is wrong. |
| 45028 | **BMS factory reset + reboot of all packs** (broadcast ID, handled by BMS 117.7 and 118): wipes the BMS parameter block to defaults; the master may boot unregistered (EEPROM 0x202 = 1). Not a relay. |
| 45029 | Pack address encoding / master claim (`g_encodeFlag`; toggles BMS pack-select pin PB12). Runs when `g_keyEventType == 0` and `g_maintModeState != 2`; **success (status 2) only with exactly 2 packs online** (mask 3); 1 pack or 3+ packs end in status 3 (timeout). Name `cmd_battery_pack_encode_start`. |
| 45030 / 45031 | **Functionless on D.** 45030: no receiver handles bytes 0x0B/0x0C in BMS 117.7/118, no local BLE/FC41D action ("BLE enable" is unfounded). 45031: ack condition impossible, 0xA0 unhandled; a write blocks the Modbus task ~5 s and returns exception 3, a read returns exception 2 immediately. |

## 9. References

- `Write_Handler` @ `0x08051d14` (v150) — dispatch for the entire 40000–47400 range
- Dispatch selector 45000 block: `iVar11 = param_2 − 45000` (switch), 45000 itself via `uVar8 == 0x76a`
- `Modbus_RS485_TCP/Scan_Logs/Write_Handler_Register_Map.csv` — complete write register map (v150-corrected)
- `Control_FW/VNSD_Control_FW_Analysis_app_0150_0805_115146.md` — v150 overall analysis
