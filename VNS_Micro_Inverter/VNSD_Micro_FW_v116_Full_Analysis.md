# Micro-FW v116 (Venus D) — Full analysis

> 🇩🇪 Deutsche Version: [VNSD_Micro_FW_v116_Vollanalyse_de.md](VNSD_Micro_FW_v116_Vollanalyse_de.md)

**Status:** 2026-09-30.

**German:** `VNSD_Micro_FW_v116_Full_Analysis.md` ·
**Diff to predecessor v115:** `VNSD_Micro_Diff_v115_v116.md` ·
**Decompilates/Listing:** `Dekompilate/VNSD_Micro_116/` (local only, not in the repo) (693 functions, 925 RAM globals, `listing/` with
Flash and CCM raw disassembly + `globals_and_registers.txt`)

## 0. Summary

- Image base **0x08004000** (16-KB bootloader below), image size 0x1C400 bytes (115,712 B),
  MCU **STM32G474** (HRTIM `0x40016800`, FDCAN, FLASHSIZE register `0x1FFF75E0`, official SVD
  applied), Keil ARMCC, FreeRTOS (RVDS/ARM_CM4F, `heap_4`, 16-KB heap), letter-shell.
  `0x6B50` B time-critical code (main control ISR, current control loop, PLL) run from **CCM-SRAM**
  `0x10000000`, reached via 10-byte veneers in Flash starting at `0x08004960`.
- Manufacturer changelog notes for v116 only "VD 2500w". This cannot be traced back to a single
  code location, but affects **six separate locations** — when constructing the diff document
  `VNSD_Micro_Diff_v115_v116.md`, the list was corrected again against the first pass of this session:
  `Bat_Mode3RoundRobinTimer` was added as a real confirmed location; the initially suspected change in
  `Ctrl_ResetControlState` proved to be **not real** on exact byte comparison (both versions already load
  2500.0f there, see §5) and is removed. All six locations raise the same load/discharge power limit
  from 2200 W to 2500 W (details in §4.1/§4.2/§5):
  1. `EMS_Inverter_CAN_Dispatcher` (`0x08005940`), CAN commands 0x03/0x04 (EMS sets charge/discharge
     limit): range check `< 0x899` (2201) → `< 0x9c5` (2501).
  2. `Modbus_HandleRegisterAccess` (`0x080164e8`), registers 43133/43134 (`g_ratedPowerConfigA`/`B`,
     rated power configuration): upper limit `0x898` (2200) → `0x9c4` (2500).
  3. Same function, register 45502 (factory test command "discharge reference"):
     `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)` → `(-2500.0)`.
  4–5. In CCM code of `Ctrl_MainControlIsr` (twice, PFC/LLC ramp-up and thermal derating) the float
     literal `0x45098000` (2200.0f) is replaced by `0x451C4000` (2500.0f).
  6. `Bat_Mode3RoundRobinTimer` (`0x08011834`, multi-pack round-robin timer for battery mode 3):
     two locations set `g_batPowerRefW` to `±2200.0` and `±2500.0` respectively — also raised.
  The **real-time hard limit** for the immediate power setpoint (CAN commands 0x56/0x58,
  check `wert+0x9c4<0x1389`, ≙ ±2500 W) was already **2500 W in both** versions.
  So v116 fixes an inconsistency: the sharply switched real-time setpoint was already allowed to go to 2500 W in v115,
  but several configuration/init/ramp paths capped the actually usable limit at 2200 W.
- **Correction of another error from the previous session:** The helper function called by
  `Inv_BuildTelemetryBlock` at `0x0800699c` (debounce every ~10 telemetry cycles, EMS dispatch sequence
  `g_emsDispatchSeq` 0–2, `Ctrl_HandleLlcPfcTestCommand_veneer`, EEPROM write of derating flag/test mode)
  was rated as "real new program path in v116". Byte-by-byte comparison of raw disassemblies shows
  identical code to v115 function `EMS_ResetMemoryByLevel` (`0x080069a4`, 86 B) — v116 simply did not
  assign a name in the automated naming run (Ghidra does not even register it as a separate function, only
  as code address `SUB_0800699c`, so it does not appear in the 693-function count). The surrounding debounce/dispatch
  logic in `Inv_BuildTelemetryBlock` is also line-identical to v115 (only variable names `uVar2`/`uVar3`
  differ). There is **no** v115→v116 behavior change at this location — see corrected version in §11.
- `Stub_CallHandlerOffset28` (`0x0800ffee`) really calls `HAL_UART_AbortCpltCallback` in v116; in
  v115 the identically named function (`0x0800fff0`) was a pure `return` stub.
- **Correction of my own error in this session:** `EEPROM_SaveOperatingStats`
  (`0x080129d4`) was initially thought to be a new v116 function (it replaces four of five call sites of a v115
  stub `Stub_EmptyNoOp2`). Byte-exact comparison shows: the 4-byte trampoline code (jump to
  `EEPROM_WriteVerify(0x500,&g_statsLastDay,0x30)`) existed in v115 **unchanged** at the same address —
  only the automated naming run had incorrectly filed it under the collective name `Stub_EmptyNoOp2`,
  while v116 correctly names it independently. Pure name correction, **no** new persistence function in v116
  (see §11).
- Also refuted: a raw diff finding that `ShellCmd_bat_mode`/`ShellCmd_pf_set` got additional debug output in v116 —
  both functions are byte-identical to v115, the finding was a misattribution of neighboring code by the diff tool (§11).
- **Important correction of the task assumption on Modbus "magic codes"**: Register 45000 knows the
  values `0x55DD`/`0x55EE`/`0x55FF`, but none of them enforces 800-W derating — `0x55FF` **clears**
  the derating flag and resets the discharge limit to 2500 W, `0x55EE` switches only to factory mode,
  `0x55DD` reads (not writes, see below) three mode bytes from EEPROM back. The actual 800-W
  forced derating is triggered **elsewhere**: by **literally** writing the value `800` to register 41010 or 44003 (§4.2, §8).
- **Name finding confirmed and open point resolved:** The correctly named function in the fresh export
  `EEPROM_ReadRegister` (`0x08006ce8`, previously incorrectly called `EEPROM_WriteRegister` in this analysis)
  internally executes `HAL_I2C_Master_Transmit` (register address) followed by `HAL_I2C_Master_Receive` (data)
  — an **I2C read access**, not a write. The previous `EEPROM_ReadRegister` (`0x08006d84`) is now called
  `EEPROM_ReadRegister_NoMutex` (identical flow without separate mutex management). Real writing remains
  exclusively handled by `EEPROM_WriteVerify`. This confirms: the `0x55DD` branch of Modbus register 45000
  **loads** `g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode` from EEPROM `0x102`/`0x300`/`0x301` back,
  rather than saving them; `0x55EE` sets `g_workMode=12` (factory mode) with `g_batteryMode=0` and
  `g_cfgBackupModeEnabled=0`; `0x55FF` sets the LLC/PFC command to 10, clears `g_powerLimitDeratedFlag`,
  sets the limit word to `0x09C4` (2500 W), zeros both modes and the power setpoint (§4.2).
- OTA/Update (own image via YMODEM over debug UART, plus a second self-update path via CAN, as well as
  relay to BMS via CAN/RS485) is structurally unchanged to v115 and thus still possible (§7). Cloud: not applicable,
  the Micro firmware has no separate network/cloud connection.

## 0a. Checklist

| # | Question | Answer v116 | Evidence |
|---|---|---|---|
| 1 | Modbus registers changed (vs. v115)? | Register table/addresses identical. Only two value changes: register 43133/43134 upper limit `0x898`→`0x9c4`, register 45502 test reference `-2200.0`→`-2500.0`. All magic codes (`0x55AA/0x55DD/0x55EE/0x55FF`) and remaining register table unchanged. | §4.2 |
| 2 | CAN protocol changed? | Only one constant in `EMS_Inverter_CAN_Dispatcher` (`0x08005940`), commands 0x03/0x04: range upper limit `0x899`→`0x9c5`. Command set, message IDs, and remaining logic unchanged (function size 1178→1174 B, only this one constant plus pure address shifts). | §4.1 |
| 3 | AC feed-in 2500 W changed? | Real-time hard limit (±2500 W, CAN commands 0x56/0x58) was **already 2500 W in v115**. New in v116: six previously 2200-W configuration/ramp/init values for the same power path are raised to 2500 W (list see §0, incl. `Bat_Mode3RoundRobinTimer`). Result: as of v116 consistently 2500 W is usable, where v115 sometimes blocked at 2200 W. | §0, §4.1, §4.2, §5 |
| 4 | Firmware update (own + BMS relay) still possible? | Yes. YMODEM self-update over debug UART, additional CAN self-update path (`OTA_SelfUpdate_StateMachine`) and BMS relay over CAN (`BMS_StartFwUpdateCan`/`BMS_FwUpdate_StateMachine`) or RS485 (`BMS_FwUpdateStateMachineTask`) are structurally and in the state machines unchanged. | §7 |
| 5 | Cloud domains changed? | Not applicable — Micro firmware has no separate cloud/network connection; internet communication runs exclusively through the Control board. | n/a |

## 1. Binary fingerprint

| Feature | Value |
|---|---|
| File | `116_micro_VNSD-0_vd_inv_app_0116_0702_ota_163439.bin` |
| Size | 115,712 bytes (0x1C400) |
| SHA-256 | `b81bb99f8213b98e11a082fba17603e5352d57b676ff6649e0323cac9f6fd860` |
| Link address | `0x08004000`–`0x080203FF` |
| MCU | STM32G474 (Cortex-M4F, Thumb-2), 16-KB bootloader below `0x08004000` |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW segment) |
| Libraries | FreeRTOS (heap_4), letter-shell |
| Initial SP | `0x20009A70` (vector table @ `0x08004000`, verified in raw image) |
| Reset vector | `0x080042AD` (Thumb) → `Reset_Handler` (`0x080042ac`) |
| Version constant | **0x74 (116 dec.)** @ `0x08010000`, followed by ASCII `"VD01"` (device/model tag) |
| Image trailer | Device tag `"VNSD"` @ `0x080203E0`; trailer word ASCII `"13DD"` shortly before image end — meaning not determined (v115 has `"I1DD"` at the same location, open point §10) |
| Manufacturer changelog | v116 — 2026-08-12: "VD 2500w" (Issue #18) |

**Scatter-Load** (calculated directly in raw image, see `listing/08000000_0800ffff.txt` and
`listing/08010000_0801ffff.txt`):

| Source (Flash) | Target | Length | Method |
|---|---|---|---|
| `0x08018c7c` (1,112 B compressed) | `0x20000000` (.data) | 0x8D8 (2,264 B) | `__decompress` (LZ77) |
| `0x080190d4` (27,472 B, uncompressed) | `0x10000000` (CCM-SRAM) | 0x6B50 (27,472 B) | `__scatterload_copy` (pure copy) |
| – | `0x200008D8` (.bss) | 0x9198 (37,272 B) | `__scatterload_zeroinit` (zero init) |

Verification: `0x200008D8 + 0x9198 = 0x20009A70` = stack top — matches vector table exactly.
Flash↔CCM calls run via 10-byte veneers (`movw`/`movt r12; bx r12`) from `0x08004960` in Flash
and beginning of CCM (`0x10000000`ff., named `<target>_veneer`).

## 2. Memory map + analysis status in Ghidra

**RAM/peripheral blocks** (`listing/globals_and_registers.txt`):

| Block | Range |
|---|---|
| `.data` | `0x20000000`–`0x200008D7` |
| `.bss` | `0x200008D8`–`0x20009A6F` |
| CCM-SRAM | `0x10000000`–`0x10006B4F` (0x6B50 B code+data) |
| APB1 | `0x40000000`–`0x4000AFFF` |
| APB2 (incl. HRTIM `0x40016800`) | `0x40010000`–`0x40017FFF` |
| AHB1 | `0x40020000`–`0x400243FF` |
| AHB2 GPIO | `0x48000000`–`0x48001BFF` |
| AHB2 Analog (ADC/DAC) | `0x50000000`–`0x50060BFF` |
| FMC/QSPI | `0xA0000000`–`0xA0001FFF` |
| PPB (Cortex system peripherals) | `0xE0000000`–`0xE00FFFFF` |

**Verification status** (program `/VNSD/Micro/Micro_116_VNSD-0_vd_inv_app_0116_0702_ota_163439.bin`):

| Check | Result |
|---|---|
| Functions | **693**, all 693 named, 0 `FUN_*` |
| Ghidra symbols | 4,626 |
| Decompilation failed | 0 |
| Named RAM globals (.data+.bss) | **925**, 0 unnamed remainder (remaining `DAT_` hits are cross-references in comments, not real gaps) |
| Veneer functions (Flash↔CCM) | 49 |
| `shellCommandTable` | `0x0801892c`, `ShellCommand[53]` (commands + 1 user entry + key bindings) |

File/folder convention (`<addr>_<name>.c`, `_index.md`, `listing/` with Flash and CCM raw disassembly + `globals_and_registers.txt`) is described in `Dekompilate/VNSD_Micro_116/`;
details see there.

## 3. FreeRTOS tasks

From `Task_CreateAllAndStartScheduler` (`0x0800f420`, 242 B) — eight direct `xTaskCreate()` calls,
then `xPortStartScheduler()`:

| Task | Entry function | Stack (words) | Priority | Handle |
|---|---|---|---|---|
| `vtask_led` | `0x080163e8` | 128 | 16 | `g_ledTaskHandle` |
| `vtask_time` | `0x080164c8` | 256 | 15 | `g_timeTaskHandle` |
| `vtask_modbus` | `0x08016424` | 128 | 14 | `g_modbusTaskHandle` |
| `vtask_shell` | `0x0801643c` | 512 | 9 | `g_shellTaskHandle` |
| `vtask_prt` | `0x08016430` | 256 | 8 | `g_canRxTaskHandle` * |
| `vtask_can` | `0x080163d0` | 256 | 6 | `g_canRxTaskHandle` * |
| `vtask_can_receive` | `0x080163e0` | 256 | 7 | `g_canRxTaskHandle` * |
| IDLE | `0x08012fd0` | 128 | 0 | `xIdleTaskHandle` |

\* All three tasks carry the same handle name `g_canRxTaskHandle` in the decompilate (three different write addresses,
same auto-name) — whether this is true multi-use of the same SRAM cell or a naming artifact was not conclusively
determined in this session (§10). If any `xTaskCreate` fails (return −1), the code prints `"..\..\SDK\FreeRTOS\tasks.c", 0x812`.
All priorities/stack sizes are — to the extent checked in this session — unchanged from v115.

## 4. Interfaces

### 4.1 CAN/FDCAN to EMS (Control) and BMS

FDCAN1 is configured in `FDCAN1_ConfigFiltersAndMailboxes` (`0x080058a8`) with a standard ID filter
(`0x300`/mask `0x300`, internal EMS↔inverter protocol) and three extended ID filters (`0x4000`,
`0x4100`, mask `0xff00` each; `0x1801aa01`/mask `0x1ff0ffff`, BMS telemetry). `HAL_FDCAN_RxFifo0Callback`
(`0x08009248`) routes each received message based on runtime mode `g_canRxMode`: `3` → directly to
`BMS_CAN_Parser`, `4` → directly to `BMS_FwUpdate_CanHandler`, else into `g_canRxQueue`, from which
`CAN_RX_DispatchTask` (`0x08005f04`) forwards frames to `EMS_Inverter_CAN_Dispatcher`. Where `g_canRxMode`
itself is set was not checked (§10).

**EMS↔inverter internal protocol** (`EMS_Inverter_CAN_Dispatcher`, `0x08005940`, 1,174 B, standard
11-bit ID, command = lower byte of arbitration ID):

| Cmd (hex) | Meaning | Direction/condition |
|---|---|---|
| 0x01 | Set power setpoint (`Ctrl_SetPowerReferenceClamped_veneer`), only `g_workMode==0` | EMS→inverter, echo |
| 0x02 | Enable/disable (`Cfg_SetStoredEnableFlag_veneer`), only `g_workMode==0` | EMS→inverter |
| 0x03 | Set max. discharge power limit, EEPROM `0x204` (`g_powerLimitDischargeW`); limit `<0x9c5`=2500 W (v115: `<0x899`=2200 W) | EMS→inverter |
| 0x04 | Set max. charge power limit, EEPROM `0x202` (`g_powerLimitChargeW`); same limit change | EMS→inverter |
| 0x05 | `g_emsDispatchFlag=1` (sleep handshake per function header) | EMS→inverter |
| 0x06 | `g_standbyLatch=1` (standby latch; formerly misnamed `g_emsPairedFlag`, not a pairing confirmation) | EMS→inverter |
| 0x07 | Sequence number handshake (`g_emsDispatchSeq=data+1`), echo | bidirectional |
| 0x10 | Read telemetry block, 0x30 B (`g_emsTelemetryBlock`) | inverter→EMS |
| 0x11 | Read debug TX payload, 8 B (`g_emsTxPayload`) | inverter→EMS |
| 0x12 | Read battery calibration status (status/charge/discharge time), 6 B | inverter→EMS |
| 0x13 | Read EMS charge/discharge limit, 4 B | inverter→EMS |
| 0x16 | Synchronize EMS charge/discharge hour counters (no echo) | EMS→inverter |
| 0x50 | LLC/PFC command: 0=stop, 1=start + `g_workMode=1` + `g_emsDispatchCmd=7` | EMS→inverter |
| 0x51 | Short buzzer beep, factory mode only (`g_workMode==0xC`) | EMS→inverter, echo |
| 0x52 | Factory test subdispatch → `g_emsDispatchCmd` 2/3/4, factory mode only | EMS→inverter |
| 0x53 | Factory test subdispatch → `g_emsDispatchCmd=5`, factory mode only | EMS→inverter |
| 0x54 | Read IO test status (`IO_TestStatusFlag`), 1 B | inverter→EMS |
| 0x55 | Factory test subdispatch → `g_emsDispatchCmd=6`, factory mode only | EMS→inverter |
| 0x56 | LLC/PFC test command with power value, range ±2500 W (`0x9c4`/`0x1389`, **unchanged in v115 and v116**), factory mode only | EMS→inverter |
| 0x57 | Set backup mode flag, only `g_workMode==1` | EMS→inverter |
| 0x58 | Set power setpoint in run mode (`g_workMode==1`), range ±2500 W (**same unchanged constant `0x9c4`**) | EMS→inverter |
| 0x60 | Start battery calibration mode (`g_workMode=4`) | EMS→inverter |
| 0xC1 | `g_emsDispatchCmd=1`, argument = source node byte of CAN ID | EMS→inverter |
| 0xC2 | Set debug level (1 or 2) | EMS→inverter |
| 0xC3 | Raw data passthrough to `g_shellRxQueue` (shell input queue drained into `shellHandler`, i.e. shell over CAN; formerly misnamed `g_bmsCanQueue`) | unclear |
| 0xCB | Device info/version query, 8 B | inverter→EMS |
| 0xCE | CAN OTA subprotocol (upper ID byte: 0=start, 1=chunk, 2=finish CRC) | bidirectional |

**BMS protocol** (extended 29-bit ID): `BMS_CAN_Parser` (`0x080058e8`, 80 B) evaluates
`(can_id & 0xfffff) >> 0x10` as discrimination field:

| PF value | Meaning | Target variables |
|---|---|---|
| 1 | Pack voltage/current + temperature/SOC | `g_bmsPackVoltageCurrent`, `g_bmsTempSoc`, updates `g_bmsLastRxTick` |
| 2 | Capacity (raw) + pack info | `g_bmsCapacityRaw`, `g_bmsPackInfo` |
| 3 | Charge voltage limit + discharge current limit | `g_bmsChargeVoltLimit`, `g_bmsDischargeCurrLimit` |
| 4 | Error/warning word | `g_bmsErrorWord`, `g_bmsWarningWord` |

`BMS_FwUpdate_CanHandler` (`0x08005868`, 60 B) handles BMS FW update subcommands (mask test
`(~can_id & 0xce)==0`, discrimination via upper ID byte): `0` = OTA ack/burst configuration
(0x80 or 0x10 chunks depending on data byte, chunk delay from `data[1]`, `g_bmsOtaAckFlag=2`); `2` =
OTA ack with CRC + chunk status, `g_bmsOtaAckFlag=4`. `BMS_StartFwUpdateCan` (`0x08010978`, 24 B,
sole caller: `OTA_CompleteFirmwareUpdate`) activates BMS OTA state machine for CAN transport
(`g_bmsFwUpdateMode=1`, `g_bmsOtaState=1`, `g_bmsOtaTransportState=4`).

**2200→2500 W finding in detail** (v115 `08005944_EMS_Inverter_CAN_Dispatcher.c`, 1,178 B ↔ v116
`08005940_EMS_Inverter_CAN_Dispatcher.c`, 1,174 B):

```c
// case 3 (discharge limit) and case 4 (charge limit), identical structure in both versions:
local_34 = *data;
if ((int)local_34 < 0x9c5) {          // v115: < 0x899
  g_powerLimitDischargeW = (ushort)local_34;   // or g_powerLimitChargeW
  EEPROM_WriteVerify(0x204,(uint8_t *)&g_powerLimitDischargeW,2);  // or 0x202
}
```

The filter values `0x4000`/`0x4100` from `FDCAN1_ConfigFiltersAndMailboxes` could not be uniquely mapped to either
`BMS_FwUpdate_CanHandler` case (open point §10).

### 4.2 RS485/Modbus

Dispatcher `Modbus_HandleRegisterAccess` (`0x080164e8`, 2,842 B, `uint (uchar *outBuf, uchar
*regTable, int isWrite, uint value)`); return values correspond to Modbus exceptions (`0`=OK,
`1`=rejected/wrong `g_workMode`, `3`=ILLEGAL DATA VALUE). The parameter `regTable` is despite its pointer
type the actual 16-bit register address (confirmed via `Modbus_WriteSingleRegister`, `0x0800f208`:
registers `<0x9c40` are not forwarded at all).

| Register (dec./hex) | Meaning | R/W | Details |
|---|---|---|---|
| 40000 / `0x9C40` | Battery calibration start (`g_battCalState`, state 0–7) | write only | `0x55AA`→reset (state 0); `0x55A1`–`0x55A6`→state 1–6; otherwise error 3. Sets `g_workMode=4`. |
| 41000 / `0xA028` | Factory ack flag A (`g_factoryCmdAckFlag`) | R/W | Only `0x55AA` writable → flag=1, otherwise error 3. |
| 41001 / `0xA029` | Factory ack flag B + clear stats | R/W | `0x55AA` → also `EEPROM_ClearStats(3)`. |
| 41010 / `0xA032` | Direct power limit/derating switch | R/W | Read: `{0, g_powerLimitDeratedFlag}`. Write **exactly `800`** → flag=1, discharge limit=800 W; **exactly `0x9C4`(2500)** → flag=0, discharge limit=2500 W; any other value error 3. Persists EEPROM `0x204`+`0x90`. |
| 41100 / `0xA08C` | Modbus slave address | R/W | `0` → error 3, else accept (EEPROM `0x101`). |
| 41200 / `0xA0F0` | Backup/off-grid output enable (inverted) | R/W | `0`→`Cfg_SetStoredEnableFlag_veneer(1)`, `1`→`…(0)`, else error 3. |
| 42000 / `0xA410` | Factory mode confirmation | R/W | `0x55AA`→no-op OK; `0x55BB`→`EEPROM_ReadRegister(0x301,&g_workMode,1,…)` reads `g_workMode` from EEPROM back (I2C read access, see below); else error 3. |
| 42010 / `0xA41A` | `g_modbusCfgByte1` | R/W | Valid 0–2. |
| 42011 / `0xA41B` | `g_modbusCfgByte2` | R/W | Valid 11–100 (`value-11≤89`), sets `g_modbusCfgByte1=3`. |
| ~43100–43129 / `0xA85C`–`0xA879` | Grid standard table `g_gridStandardTable` (6 of 10 rows visible, 10 B each) | R/W | Anchors verified: 43106 (row 1, field+4), 43124 (row 4, field+1). Remaining individual registers of this block not conclusively mapped (§10). |
| 43133 / `0xA87D` | `g_ratedPowerConfigA`, EEPROM `0x800` | R/W | Upper limit `0x898`(2200)→**`0x9c4`(2500)** — **finding #2 of 2200→2500 change.** |
| 43134 / `0xA87E` | `g_ratedPowerConfigB`, EEPROM `0x802` | R/W | Same change. |
| 44000 / `0xABE0` | `g_cfgMaxSocPct` (max. charge SOC %, raw value/10; formerly misnamed `g_gridProtCfgParam0`) | R/W | Range 800–1000. EEPROM `0x201`. |
| 44001 / `0xABE1` | `g_cfgCutoffSocPct` (discharge cut-off SOC %, raw value/10; formerly misnamed `g_gridProtCfgParam1`) | R/W | Range 120–300. EEPROM `0x201`+1. |
| 44002 / `0xABE2` | `g_powerLimitChargeW` | R/W | Upper limit 2500 W (unchanged vs. v115). EEPROM `0x202`. |
| 44003 / `0xABE3` | `g_powerLimitDischargeW` | R/W | Upper limit 2500 W; **additionally**: if `g_powerLimitDeratedFlag==1`, any value `>800` is rejected with error 3 — actual enforcement of 800-W derating. EEPROM `0x204`. |
| 44100 / `0xAC44` | `g_cfgGridStandard`, calls `Grid_Protection_SetLimits()` | R/W | Range 0–4. EEPROM `0x206`. |
| **45000 / `0xAFC8`** | **"Magic code" system command** | write only | See detail block below. |
| 45010–45019 / `0xAFD2`–`0xAFDB` | `g_cfgExtraRegBlock`, 10×16-bit | R/W | Write only when `g_workMode==0` or `0xC`. Last register saves 20-B block to EEPROM `0x21`. |
| 45200 / `0xB150` | Buzzer short beep (test) | write only | Factory mode `g_workMode==0xC`/`0` only. |
| 45300 / `0xB1B4` | TIM1 PWM test control | write only | Factory mode `g_workMode==0xC`/`0` only; `g_wdgSafetyCounter=100`. |
| 45400–45402 / `0xB198`–`0xB19A` | LLC/PFC test commands 1–3 | write only | Factory mode `g_workMode==0xC` only, calls `Ctrl_SetLlcPfcCommand_veneer(1/2/6)`. |
| 45500–45502 / `0xB1BC`–`0xB1BE` | Charge/discharge power reference (test) | write only | Register 45502: `g_workMode==0`, `value==1` → `Ctrl_SetPowerReferenceClamped_veneer(-2500.0)` (**v115: `-2200.0`** — **finding #3**). |
| 45600/45601 / `0xB220`/`0xB221` | Charge/discharge power, any float value | write only | Factory mode `g_workMode==0` only. |
| 45603/45604 / `0xB223`/`0xB224` | Battery calibration discharge/charge time, EEPROM `0x950`/`0x952` | read only | I2C read access. |
| 46000 / `0xB3B0` | Debug/BMS OTA entry | write only | `0x5100`→`g_debugLevel=1` (EEPROM `0x900`); `0x4D2`(1234)→also `g_bmsFwUpdateMode=1`; `0x929`(2345)→also `g_bmsOtaTransportState=2`; else error 1. |

> **Note (2026-10-01):** The registers 40000 / 41000 / 41001 / 42000 etc. listed here belong to the **Micro's OWN Modbus register space**. They are reachable only through the Micro's own RS485 Modbus slave. The Control's Modbus registers with the same numbers are **different registers** (e.g. Control 41001 is not implemented, exception 2; Control 41000 reboots the Control and the Micro). See `../Modbus_RS485_TCP/VNSD_Unclear_Registers_Resolution_2026-10-01.md`.

**Magic code register 0xAFC8 (45000) — `0x55DD`/`0x55EE`/`0x55FF` (literal code, identical in
v115 and v116):**

```c
if (value == 0x55ee) {
  Ctrl_SetLlcPfcCommand_veneer(0); Buzzer_Stop();
  g_batteryMode = 0; g_cfgBackupModeEnabled = 0;
  g_workMode = 0xC;                      // Factory/test mode
  return 0;
}
if (value != 0x55ff) {
  if (value == 0x55dd) {
    Ctrl_SetLlcPfcCommand_veneer(0); Buzzer_Stop();
    EEPROM_ReadRegister(0x102,&g_batteryMode,1,timeout);
    EEPROM_ReadRegister(0x300,&g_cfgBackupModeEnabled,1,timeout_00);
    EEPROM_ReadRegister(0x301,&g_workMode,1,timeout_01);
    return 0;
  }
  return 3;
}
// value == 0x55ff:
Ctrl_SetLlcPfcCommand_veneer(10); Buzzer_Stop();
g_powerLimitDeratedFlag = 0;             // Flag is CLEARED
puVar9[4] = 0xc4; puVar9[5] = 9;         // g_powerLimitDischargeW = 0x09C4 = 2500 W
g_modbusConfigParam1 = 0; g_batteryMode = 0; g_cfgBackupModeEnabled = 0; g_workMode = 0;
Ctrl_SetPowerReferenceClamped_veneer(0.0);
```

**Correction against task assumption:** None of these three codes sets 800-W derating.
`0x55EE` switches only to factory mode (`g_workMode=0xC`), `0x55DD` reads three status bytes via
`EEPROM_ReadRegister` from EEPROM, `0x55FF` **clears** the derating flag and **resets discharge
limit to 2500 W** — the opposite of throttling. The actual 800-W forced derating occurs at **two other**
registers (41010 and 44003, see above) by **literally** writing the value `800`.

**I2C direction finding (resolved):** The fresh export correctly names function `0x08006ce8` as
`EEPROM_ReadRegister` (in this analysis initially mistakenly called `EEPROM_WriteRegister`); the previous
`EEPROM_ReadRegister` (`0x08006d84`) is now called `EEPROM_ReadRegister_NoMutex`. Both execute
`HAL_I2C_Master_Transmit` (register address) followed by `HAL_I2C_Master_Receive` (data) — an
I2C random-access **read access**; `_NoMutex` runs without separate mutex management (caller holds it already).
Real writing (address+data in one transmit, with read-back verification) remains exclusively handled by
`EEPROM_WriteVerify` (`0x08006e40`). This confirms: the `0x55DD` branch of register 45000 **loads**
`g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode` from EEPROM `0x102`/`0x300`/`0x301` back
(restore last saved state), rather than saving current RAM state — likewise `0x55BB` on register 42000
(`0xA410`) reads `g_workMode` from the same EEPROM offset `0x301` back (see above). Previously open
point in §10, now clarified by naming in fresh export.

Generic activation code `0x55AA` appears independently of the three "magic codes" at several other
registers (40000, 41000, 41001, 42000) as factory reset/confirmation code.

### 4.3 Debug shell

`shellCommandTable` @ `0x0801892c`, `ShellCommand[53]` (53 entries = commands + 1 user entry +
key bindings), of which **42 real `ShellCmd_*` handlers** (earlier estimate "33" was too low).
Login runs via `shellSetUser`/letter-shell library; per diff artifact in `xTaskResumeAll`
(literal pool comment) one stored user has an **empty password**. The actual password byte itself
was not decoded from raw `ShellCommand[53]` array (open point §10).

| Command | Address | Brief description |
|---|---|---|
| `log_data` | `0800eb20` | Output up to 100 values from `g_sampleRingBuffer` via `printf("%f\r\n")`. |
| `pi` | `080127c0` | Set current PI controller (`Ctrl_SetRampTargetA`), print "Current PI parameter changed kp=%f, ki=%f". |
| `vol_pi` | `080127cc` | Set voltage PI controller (`Ctrl_SetRampTargetB`). |
| `qpr` | `080127d8` | Set resonance controller QPR (`Ctrl_SetPrControllerTarget`). |
| `power_cfg` | `080127e4` | LLC/PFC test command (`Ctrl_HandleLlcPfcTestCommand`). |
| `clear` | `08013658` | ANSI clear escape to terminal. |
| `help` | `08013c90` | Command list or help text for a command. |
| `alarm` | `080146f8` | Set buzzer period counter. |
| `backup_mode` | `080146fc` | Backup mode on/off (`Cfg_SetStoredEnableFlag`). |
| `bat_mode` | `0801474c` | Battery mode on/off (`Cfg_SetPersistedFlag102`) — **byte-identical to v115**, no new behavior (§11). |
| `dac_high`/`dac_low` | `080148fc`/`08014902` | Direct write DAC3/DAC1 (test output). |
| `help_prt` | `08014908` | Help text for `prt`. |
| `io_set` | `0801497c` | Switch digital I/Os (LED, grid/island relays, fault reset, aux). |
| `io_show` | `08014b14` | Output complete I/O status block. |
| `log_clear`/`log_err`/`log_list`/`log_open` | `08014dd0`–`08014ddc` | Clear/filter/full/live toggle fault log. |
| `old_ate` | `08014df4` | Burn-in mode (`g_workMode=4`), state 0–5. |
| `par_mode` | `08014e54` | Parallel operation mode (`Cfg_SetOperatingModeFlag`). |
| `pf`/`pf_power` | `08014ecc`/`08014f10` | Set reactive power setpoint. |
| `pf_set` | `08014f60` | Set protection comparator threshold (`Prot_ClampComparatorThreshold`) — **byte-identical to v115** (§11). |
| `prt` | `08015014` | Configure periodic log output (thunk to `0800ef10`). |
| `reset_memory` | `08015018` | Clear data (with/without sums), `EEPROM_SaveOperatingStats`. |
| `reset` | `0801507c` | `Board_SystemReset()`. |
| `rtos_status` | `08015080` | FreeRTOS task list. |
| `fan_set` | `080150b4` | Fan PWM (TIM1), 0–400. |
| `set_mode` | `080150c4` | Set `g_workMode` (0 default, 1 hand, 2 debug, 4 burn-in, 0xD voltage source), EEPROM retry write. |
| `set_n` | `0801518c` | Set droop parameter. |
| `set_power` | `080151bc` | Clamped power setpoint (`Ctrl_SetPowerReferenceClamped`). |
| `set_sr` | `080151ec` | Resonance controller clip flag (`Ctrl_SetPrClipFlag`). |
| `show_pi` | `080151f0` | Output current PI values. |
| `show_power` | `08015230` | Reference table of 24 test modes (0–23) for `set_mode`. |
| `test`/`test_1` | `08015578`/`08015610` | Debug measurements / current control loop enable flag. |
| `update`/`update_bms`/`update_bms2` | `0801561c`–`08015684` | Start firmware update (own/BMS transport variant 1/2). |
| `version` | `080156b4` | Version, build timestamp, hardware/set version. |
| `sr_set` | `08015768` | Deadtime/synchronous rectifier duty cycle. |

**Correction of raw diff finding:** A pre-computed function diff claimed that `bat_mode` and
`pf_set` got an additional code block (float debug printf + tail call) in v116. Both functions are in
v115 and v116 **exactly the same size** (38 and 48 B respectively) and logically identical; the supposed new
block actually lies in unsymbolized string/code gap directly after (content of `ShellCmd_pi`/`ShellCmd_pf_power`,
only at an unindexed address, duplicated) and was misattributed to the wrong function name by the diff tool (§11).

### 4.4 I2C EEPROM layout

Low-level: `EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex` (I2C random-access read, see §4.2),
`EEPROM_WriteVerify`/`_NoMutex` (real write, page write + read-back verification, up to 4 retries, on failure
`FaultLog_Enqueue()` + `I2C1_Init()` bus reset), `EEPROM_WriteVerify_veneer` (`0x10000104`, CCM callers like
`Prot_LatchHwFaultToErrorWord` can also write EEPROM). `EEPROM_SaveTimestamp` writes `HAL_GetTick()` to offset `0x80`
and immediately overwrites it with 0 (purpose unclear, §10). `EEPROM_ClearStats(level)` clears parts of the
48-B stats block based on level, but always writes back the complete block.

| Offset | Size | Field | Meaning |
|---|---|---|---|
| `0x000` | 1 B | Version/boot flag | `3`=fully valid, `2`=stats reload only, else factory reset |
| `0x001` | 0x2F B | `g_cfgDeviceIdBlock` | Main configuration block (incl. "VD01" signature, model code, HW variant); internal layout not re-verified |
| `0x080` | 1 B | Heartbeat/timestamp | See `EEPROM_SaveTimestamp` quirk above |
| `0x090` | 1 B | `g_powerLimitDeratedFlag` | 1=discharge limit capped at 800 W, 0=full 2500 W; `0xFF`→0 on load |
| `0x100`–`0x102` | 3 B | `g_standbyLatch`, `g_modbusSlaveAddress`, `g_batteryMode` | Bytes 3–4 unknown |
| `0x150` | 1 B | `g_ctrlModeDisplayCounter` | Countdown (CAN retry/mode change LED); `0xFF`→0 on load |
| `0x200`–`0x206` | 7 B | `g_cfgMaxSocPct`/`g_cfgCutoffSocPct` (each /10 scaled), `g_powerLimitChargeW`, `g_powerLimitDischargeW`, `g_cfgGridStandard` | Sum exactly 7 B |
| `0x300` | 0x7C B | `g_cfgBackupModeEnabled` + following block | Work mode/backup/scheduling block, internal layout not traced |
| `0x500` | 0x30 B | `g_statsLastDay` | Operating hours/energy statistics (`EEPROM_SaveOperatingStats`) |
| `0x800`/`0x802` | each 2 B | `g_ratedPowerConfigA`/`B` | Rated power configuration (physical meaning per naming with low confidence) |
| `0x900` | 2 B | `g_debugLevel` + boot safe flag | Debug/trace level; written only in factory reset path |
| `0x901` | 1 B | – | Reset to 0 on **every** boot unconditionally |
| `0x950`/`0x952` | each 2 B | `g_emsChargePowerLimitW`/`g_emsDischargePowerLimitW` | EMS charge/discharge limit |

Factory reset default values: `"VD01"`, model code `0xD`, version marker `0x74` (=116, matches FW version),
grid parameters `0x64`/`0xC`, Modbus address `1`, battery mode `1`, charge/discharge limit each `0x9C4` (2500 W).
Internal layout of blocks `0x001`–`0x02F`, `0x100` (bytes 3–4) and `0x300`–`0x37B` as well as sub-block
`0x021` (scheduling table per old docs) not traced in detail in this session (§10).

## 5. Control & power section

Two time-critical ISRs run from CCM-SRAM, reached via Flash veneers (`0x08004992`, `0x080049b0`):

- **`Inv_CurrentLoopIsr`** (`0x10001428`, 3,266 B): fast per-PWM cycle ISR — SOGI-PLL +
  PR current controller, ADC scaling, PWM setpoint update, fast protection checks. Does not itself write
  HRTIM compare/period registers.
- **`Ctrl_MainControlIsr`** (`0x1000219c`, 12,446 B): supervisory control ISR. Calls in sequence
  `ADC_ProcessSamples_veneer` → `Grid_GetFrequency_veneer` → `Ctrl_ComputeGridRmsAndPower` →
  `Ctrl_LlcVoltageRegulationAndProtection` → `Prot_MonitorHvBusLevelFlags` →
  `Ctrl_RampDcBusReferenceWithFaultCheck` → `Inv_GridConnectionSupervisor`, then a large
  state machine (PFC/LLC ramp-up, grid connection ramp, normal operation, fault/derating paths;
  due to size not fully decoded state-by-state, §10).

**PLL/SOGI chain** (all CCM): `PLL_InitCordicState` (init + CORDIC configuration for sin/cos),
`PLL_SogiUpdateStep` (one SOGI step, α/β signals, frequency error integration with clamping),
`PLL_IntegratePhaseAngle`/`PLL_NormalizeAngle` (phase integration/wrap), `PLL_UpdateFrequencyEstimate`
(RMS/cross-product accumulation, adaptive frequency estimation), `PLL_UpdatePhaseFilterFir`/
`PLL_UpdateGridFreqFilter` (FIR smoothing, notch/biquad coefficients).

**Current controller/PR chain**: `Ctrl_PrControllerStepDualMode` (combined SOGI-PLL+PR step),
`Ctrl_PrControllerDesign` (PR coefficients from cutoff/gain/resonance frequency),
`Ctrl_PiControllerStep`/`_Reset` (generic clamped PI), `Ctrl_BiquadFilterDesign`/
`Ctrl_NotchFilterDesign`, `Ctrl_InitResonantControllerParams`, `Ctrl_InitCurrentLoopState`.

**HRTIM**: The duty/phase calculation for LLC half-bridge sits in a **third, separate Flash ISR**
`HRTIM_TIMB_IRQn_IRQHandler` (`0x0800c8dc`, 338 B) — for each Timer-B repetition, CMP2–4/SET/RST
registers for two PWM legs are calculated mode-dependent (1–3), via `g_hrtimStateCh1/Ch2`,
`g_hrtimPeriodTicks`, `g_hrtimPhaseOffset`, `g_hrtimCompareOffset`, `g_softStartCounter`. Init/config:
`Ctrl_HRTIM_Init` (`0x0800d164`, 988 B, full bring-up: master + 6 timer units, compares, deadtime,
chopper, ADC trigger, fault config, start); `HRTIM_FaultConfig` (`0x0800ca38`, 648 B, source/polarity/filter
for 11 fault/external event channels); `HAL_HRTIM_FaultConfig` (`0x08009bd6`) as lock-protected wrapper.

**The 2200→2500 W constant in CCM** (float `0x451C4000`=2500.0f, v115 `0x45098000`=2200.0f, at two
relative locations — **correction against previous session:** one originally suspected third CCM location in
`Ctrl_ResetControlState` is not a real change, see below):

| Address | Context |
|---|---|
| `0x100029cc` | In `Ctrl_MainControlIsr`: once a counter reaches 120 ticks, the output/clamp limit of the power PI controller is set to 2500.0, `Ctrl_PiControllerReset()` called — start of grid connection ramp. |
| `0x10005174` | Base of thermal derating formula: from ~64 °C `(1.0-(T-64)/20)*2500.0`, additionally percentage capped — thermal derating of AC output limit. |

**Correction:** `Ctrl_ResetControlState` (v116 `0x10005fd8`, v115 `0x10005fe8`) was carried as third
CCM change location in previous session. Byte-by-byte comparison of literal pool entry shows however
that **both** versions already load `0x451C4000` (2500.0f) there — the complete function body is except for
address shift identical (130 B in both versions). There is **no** v115→v116 change at this location; it is
removed from findings list.

The remaining two CCM locations are consistent with "AC feed-in limit raised from 2200 W to 2500 W" —
at grid connection ramp start and as base value of thermal derating; together with the findings in §4.1/§4.2
and the finding in `Bat_Mode3RoundRobinTimer` (Flash code, not CCM), the six real 2200→2500 W findings listed in §0 are identified.

## 6. Protection & fault codes

`g_faultLogRing` (SRAM `0x20001a2c`, 200 entries of 20 B each) is filled via `FaultLog_Enqueue`
(`0x0800eb88`) and passed via `FaultLog_Dequeue`/`Log_CheckFaultEventsAndPrint` (only codes >0x27) to
`Log_PrintFaultEntry` (`0x0800dba0`, 1,712 B) for text output. A second, smaller ring `g_eventLogRing`
(`0x200029cc`, 12 B/entry, `Log_RecordFaultEvent` `0x0800daa4`) has no discernible print routine in the read files
(§10). `Prot_LatchHwFaultToErrorWord` (`0x10005dd8`, CCM) takes the HW comparator latch byte, logs event
`0x36` and maps latch bits into the `err2` word — matches **exactly** the field-verified bit mapping of old
documentation (bit0/1/3/4 ← HW latch, bit7=event 0x13, bit8=event 0x31, bit9=event 0x29,
bit11=event 0x35); no contradictions, only code address has expectedly shifted.

| Code | Meaning (English) | Additional data |
|---|---|---|
| default | Generic raw-value dump | 4× `%5d` |
| 1–4 | Generic dumps with 0–3 floats | data1–3 |
| 5 | Idle | – |
| 6 | Grid-side check OK | – |
| 7/8 | HV bus discharge/charge enabled | – |
| 9/0xa | Battery discharge/charge enabled | – |
| 0xb | Bad battery data check OK | – |
| 0xc | Precharge enabled | – |
| 0xe/0xf | H-bridge/charge H-bridge started | – |
| 0x10/0x11 | LLC stage started / LLC soft-start | – |
| 0x12 | Inverter shut down | set_power/max_power/low_power |
| 0x13 | System shut down (matches old docs "err2 bit7") | state ×2 |
| 0x14 | Backup function activated | – |
| 0x15 | BMS requests sleep mode | – |
| 0x16/0x17 | Forced charging requested / without communication | – |
| 0x18 | Battery communication OK | – |
| 0x19 | Grid frequency detected | feq |
| 0x1a | Reset cause (0=control board, 1=pin, 2=boot, 5=power-down) | subcode |
| 0x29 | State machine error (old docs: err2 bit9) | last/now |
| 0x2a/0x2b | Grid frequency/SPLL frequency error | feq |
| 0x2c | Grid VRMS error | V_rms, count, feq |
| 0x2d | Grid max voltage error | V_max |
| 0x2e | Voltage waveform error | V, V_rms, sin |
| 0x2f | Grid feed-in current error | U_rms, I_setpoint, I_rms |
| 0x30 | Software PLL restart | soft/HW PLL freq, U_rms |
| 0x31 | Island mode current error protection (old docs: err2 bit8) | current sum, I_island |
| 0x32 | Grid detection error | – |
| 0x33 | Battery communication error | IE/IR/ECR (CAN) |
| 0x34 | HV bus voltage fluctuation | voltage |
| 0x35 | State change timeout (old docs: err2 bit11) | status, step |
| 0x36 | Hardware protection trigger (collective event; subflags: bit0 inverter output overcurrent, bit1 bus overvoltage, bit2 battery-side overcurrent) | I_out, I_bat, flags |
| 0x37 | PWM limit protection | PWM_setpoint, U_hv, U_grid |
| 0x38 | Self-test error | error code |
| 0x3a/0x3b | Battery error (general) / battery-reported error code | – / code |
| 0x3c | Battery voltage error | BAT V, BMS V, mode |
| 0x3d | Battery output voltage enable error | BMS current, mode |
| 0x3e | Battery discharged/charged to SOC limit | subcode, SOC |
| 0x3f/0x40 | Peak current/voltage protection | max values |
| 0x41 | Watchdog reset | – |
| 0x42 | EEPROM write error | – |

Codes `0xd`, `0x1b`–`0x28`, `0x39` fall through to `default` branch (raw values only) — unused or visible only
via `g_eventLogRing` (§10). `Prot_SetFaultBitOnMatch` (`0x10005db0`) sets a fault bit on value equality with an
unnamed constant — purpose not resolvable (§10).

## 7. OTA/update

**Self-update via YMODEM** (over debug UART/console, same queue as shell commands `update`/`update_bms`/`update_bms2`):
classic YMODEM (SOH/STX/EOT/2×CAN abort), header packet with filename+size (`YModem_ParseSizeString`),
size check against free flash (default 0x80000), then `FLASH_EraseSectors` at `0x08040000` and
`FLASH_WriteData` at `0x08044000`. An explicit CRC16 check of YMODEM packet trailer bytes themselves could **not**
be demonstrated in the code (open point §10). Completion via `OTA_CompleteFirmwareUpdate`: if
`g_bmsOtaTransportState==0` it writes the image descriptor and triggers a reset via `SCB_AIRCR`
(handoff to bootloader); for `1`/`2` instead `OTA_ComputeChecksum` (simple inverted byte sum, **no** CRC16) is computed
over the received image and passed to `BMS_StartFwUpdateCan`/`BMS_StartFwUpdateRs485` — the image is then relayed to
BMS instead of being taken over itself.

**Second, independent self-update path via CAN** — `OTA_SelfUpdate_StateMachine` (`0x0800613c`, 386 B, driven by
`g_canOtaState`, CAN IDs around `0xCE`, see §4.1): erase → for each 8-byte chunk check with
`CRC16_Modbus` and write (`FLASH_WriteData` at `0x08044000`) → final `CRC16_Modbus` over complete image →
write descriptor (address/length/checksum) to `0x08040000` → `SCB_AIRCR` reset.

**BMS firmware relay via CAN**: `BMS_FwUpdate_StateMachine` (`0x080062f0`, 608 B, poll-loop, max 11 retries) —
state 1 sends header (total size + `CRC16_Modbus` of image), state 2 waits for start ack (timeout 1000 cycles),
state 3 sends up to 0x80 chunks of 8 B each (throttled via `g_bmsOtaChunkDelayMs`) with CRC16 over burst,
state 4 evaluates chunk ack (continue/retry/abort). Completion: `Ctrl_HandleLlcPfcTestCommand_veneer(1,0.0)`
(safe state), `EEPROM_SaveOperatingStats()`, `NVIC_SystemReset()`. A second, transport-independent variant
`BMS_FwUpdateStateMachineTask` (5 states: SendStart/SendDataChunk/SendSizeCmd/SendFinalize/done) likely used for
RS485 relay; both paths end with the same safety/persistence/reset sequence. `BMS_FwUpdate_CanHandler` (`0x08005868`)
is the RX side (§4.1).

**Conclusion**: Both own firmware update and relay to BMS (via CAN and — to the extent visible in code — RS485)
are structurally and in state machines unchanged to v115 and thus unrestricted and still possible.

## 8. Security findings

1. **Unauthenticated debug/factory mode access via Modbus:** Register 45000 (`0xAFC8`) switches
   with mere knowledge of the values `0x55EE`/`0x55DD`/`0x55FF` into factory mode or persists/reads
   operating states, without further authentication. Register 46000 (`0xB3B0`) switches permanently
   to elevated debug level with `0x5100` (EEPROM `0x900`) and can trigger BMS firmware update sequence
   with `0x4D2`/`0x929`.
2. **Generic activation code `0x55AA`** works unauthenticated at multiple registers (40000
   battery calibration, 41000/41001 factory ack, 42000 factory mode confirmation).
3. **Direct power limit bypass:** Register 41010 allows with literal value `800` or `0x9C4` to set/clear
   800-W forced derating without further check beyond exact numeric value.
4. **Default shell user with empty password** (letter-shell, one user entry, see §4.3).
5. **Name finding confirmed:** Function `0x08006ce8` is correctly called `EEPROM_ReadRegister` in fresh
   export and implements an I2C **read access**, not write (§4.2). The `0x55DD` path on register 45000
   thus restores a previously saved state rather than persisting current RAM state — security-wise less
   relevant than initially assumed, since no otherwise-lost state is persisted.
6. All mentioned mechanisms run over the internal RS485/CAN bus between control board, Micro and BMS;
   direct external reach (internet/WLAN) does not exist for Micro firmware itself — it has no separate network
   connection. The practical exploit rating depends on how the control board exposes this bus (see control FW analysis);
   that was not re-checked in this session.

## 9. Chinese strings

184 UTF-8 strings with Chinese text in export. The intermediate file used had corrupted encoding
(replacement characters) in 157 of 184 entries; all 184 were re-decoded in this session directly from raw image
(Flash region via file offset `address-0x08004000`) or CCM load image (file offset `0x150D4+(address-0x10000000`)
and fully translated.

**From `Log_PrintFaultEntry` (`0800dba0`) — fault log plain texts, see also §6:**

| Address | Chinese | English |
|---|---|---|
| `0800db08` | `\r\n日志清除成功\r\n` | "\r\nLog cleared successfully\r\n" |
| `0800df78` | `网侧检测OK` | Grid-side check OK |
| `0800df94` | `系统关闭 state = %d,%d` | System shut down, state = %d,%d |
| `0800dfbc` | `允许高压母线充电` | HV bus charge enabled |
| `0800dfe0` | `允许高压母线放电` | HV bus discharge enabled |
| `0800e004` | `允许电池放电` | Battery discharge enabled |
| `0800e024` | `允许电池充电` | Battery charge enabled |
| `0800e044` | `电池异常数据检测OK` | Bad battery data check OK |
| `0800e064` | `电池通讯OK` | Battery communication OK |
| `0800e078` | `开启备份功能` | Backup function activated |
| `0800e098` | `打开预充` | Precharge enabled |
| `0800e0b0` | `启动H桥` | H-bridge started |
| `0800e0c8` | `启动充电H桥` | Charge H-bridge started |
| `0800e0e4` | `启动LLC` | LLC started |
| `0800e0f8` | `LLC软起` | LLC soft-start |
| `0800e10c` | `BMS请求休眠` | BMS requests sleep mode |
| `0800e128` | `强制充电请求` | Forced charging requested |
| `0800e148` | `无通讯强制充电` | Forced charging without communication |
| `0800e168` | `复位原因 = %d` | Reset cause = %d |
| `0800e180` | `中控复位` | Control board reset |
| `0800e190` | `引脚复位` | Pin reset |
| `0800e1a0` | `boot复位` | Boot reset |
| `0800e1ac` | `掉电复位` | Power-down reset |
| `0800e1bc` | `检测电网频率 feq = %d` | Grid frequency detected, feq = %d |
| `0800e1dc` | `自检错误%d` | Self-test error %d |
| `0800e1fc` | `电网捕获频率异常` | Grid frequency capture anomaly |
| `0800e238` | `SPLL频率异常` | SPLL frequency anomaly |
| `0800e250` | `电网VRMS异常` | Grid VRMS anomaly |
| `0800e630` | `电网MAX电压异常` | Grid max voltage anomaly |
| `0800e660` | `电压波形异常` | Voltage waveform anomaly |
| `0800e6b0` | `电池电压异常` | Battery voltage anomaly |
| `0800e6fc` | `电池电压允许输出异常` | Battery output voltage enable anomaly |
| `0800e740` | `高压母线电压波动` | HV bus voltage fluctuation |
| `0800e774` | `并网电流异常` | Grid parallel current anomaly |
| `0800e798` | `锁相环重启` | PLL restart |
| `0800e7b0` | `电池异常` | Battery anomaly |
| `0800e7c0` | `电池上传故障` | Battery-reported error |
| `0800e7d8` | `故障代码 = %x` | Error code = %x |
| `0800e7ec` | `电网检测异常` | Grid detection anomaly |
| `0800e804` | `电池放电到上限soc %d` | Battery discharged to upper SOC limit %d |
| `0800e824` | `电池充电到上限soc %d` | Battery charged to upper SOC limit %d |
| `0800e844` | `电池通讯异常` | Battery communication anomaly |
| `0800e870` | `峰值电流保护` | Peak current protection |
| `0800e8a4` | `峰值电压保护` | Peak voltage protection |
| `0800e8d0` | `看门狗复位` | Watchdog reset |
| `0800e8e4` | `硬件保护` | Hardware protection |
| `0800e91c` | `逆变器输出过流` | Inverter output overcurrent |
| `0800e934` | `母线过压` | Bus overvoltage |
| `0800e944` | `电池测过流` | Battery-side overcurrent measured |
| `0800e958` | `PWM超过保护` | PWM limit protection exceeded |
| `0800e970` | `状态机异常` | State machine anomaly |
| `0800ea54` | `状态机切换超时` | State change timeout |
| `0800ea88` | `离网电流异常保护` | Island mode current anomaly protection |
| `0800eaa4` | `电流累加值 = %f , 当前离网电流 = %f` | Current sum = %f, current island mode = %f |
| `0800ead4` | `EEPROM写错误` | EEPROM write error |
| `0801862c` | `逆变器关闭 set_power = %6.3f \| max_power = %6.3f \| low_power = %6.3f` | Inverter shut down, set_power/max_power/low_power as shown |
| `0801867c` | `电压有效值 = %5f 电流设定值 = %5f 电流有效值 = %5f` | Voltage RMS / current setpoint / current RMS |
| `080186c4` | `软件锁相环频率 = %5f 硬件锁相环频率 = %5f 电压有效值 = %5f` | Software/hardware PLL frequency, voltage RMS |
| `08018718` | `pwm设定值 = %5f ,高压母线电压 = %5f,电网电压 = %5f` | PWM setpoint, HV bus voltage, grid voltage |

**From `Log_PrintDebugData` (`08010c40`) — status outputs:**

| Address | Chinese | English |
|---|---|---|
| `08010f3c` | `\r\n捕获频率 = %f\r\n` | Captured frequency = %f |
| `08010f58` | `\r\n锁相环频率 = %f\r\n` | PLL frequency = %f |
| `08010f80` | `\r\n电网电压有效值 = %f,avg = %f 10min= %f\r\n` | Grid voltage RMS, avg, 10-min average |
| `08010fbc` | `\r\n输出电压有效值 = %f , 平均功率 = %d \r\n` | Output voltage RMS, average power |
| `08010ff0` | `\r\n离网电压有效值 = %f\r\n` | Island mode voltage RMS |
| `08011014` | `\r\n并网电流有效值 = %f\r\n` | Grid parallel current RMS |
| `08011034` | `\r\n离网电流有效值 = %f\r\n` | Island mode current RMS |
| `080110a4` | `\r\n日充电%d W/H,日放电%d W/H,日收益%d\r\n` | Daily charge, daily discharge, daily gain |
| `08011124` | `\r\n设置功率 = %2f , 实际功率%2f ,发出功率%2f \r\n` | Setpoint power, actual power, output power |
| `08011164` | `\r\n有功 = %2f , 无功 = %2f , fo = %2f \r\n` | Real power, reactive power, fo |
| `08011198` | `\r\n高压 = %d ,低压 = %d \r\n` | High voltage, low voltage |
| `0801123c` | `\r\n ***原始值***\r\n` | ***Raw values*** |
| `080115b0` | `\r\n***实际值***\r\n` | ***Actual values*** |
| `08011724` | `电池状态\r\n` | Battery status |
| `08011810` | `bms最后通讯时间 = %d` | Last BMS communication time = %d |
| `0801854c` | `\r\n电池功率 = %f,离网功率 = %f,离网功率2 = %f,并网功率 = %f\r\n` | Battery power, island mode power ×2, grid parallel power |

**Shell help/status texts (various `ShellCmd_*`, no fixed "users" assignment in export):**

| Address | Chinese | English |
|---|---|---|
| `080065b4` | `其他错误` | Other error |
| `08014724`/`08014738` | `开启备份模式`/`关闭备份模式` | Enable/disable backup mode |
| `08014774`/`08014788` | `关闭电池模式`/`开启电池模式` | Disable/enable battery mode |
| `080147e8` | `修改电流pi参数 kp = %f , ki = %f` | Current PI parameter changed |
| `08014860` | `修改电压pi参数 kp = %f , ki = %f` | Voltage PI parameter changed |
| `080148d8` | `修改kpr参数 kp = %f, kr = %f` | kpr parameter changed |
| `08014928` | `prt 打印功能码` | prt print function code |
| `08014940`/`08014954`/`08014968` | `0 -- 电网参数`/`1 -- 运行状态`/`2 -- ADC采样` | 0 grid parameters / 1 operating state / 2 ADC samples |
| `08014b00` | `没有对应IO口` | No matching IO port |
| `08014bf8`/`08014cc4` | `********输出口状态*********`/`********输入口状态*********` | Output port status / input port status |
| `08014c34`–`08014cac` | `2 并网继电器 = %d` … `6 电池辅源 = %d` | 2 grid parallel relay … 6 battery auxiliary power |
| `08014e28`/`08014e3c` | `老化模式 %d`/`老化模式错误` | Burn-in mode %d / burn-in mode error |
| `08014e90`/`08014ea4`/`08014eb8` | `关闭并机模式`/`开启并机模式`/`并机准备模式` | Parallel off/on/prepare |
| `08014efc` | `设定值 = %f` | Setpoint = %f |
| `08014f40` | `设无功功率定值 = %f` | Reactive power setpoint = %f |
| `08014f90` | `固定pf = %f` | Fixed power factor (pf) = %f |
| `08014ff0` | `设定值1 = %f,设定值2 = %f` | Setpoint 1, setpoint 2 |
| `08015040`/`08015068` | `清空所有数据(不含累计数据)`/`清空所有数据` | Clear all data (without/with sums) |
| `08015134`–`0801517c` | `调试模式`/`手动模式`/`老化`/`ate老化`/`电压源模式` | debug/hand/burn-in/ATE burn-in/voltage source mode |
| `080151dc` | `设定功率 %f` | Power set %f |
| `080152a4`–`08015560` | Test modes 0–23 legend (`0.IDLE …` to `23. single PFC island`) | Complete English translation: IDLE, open-loop PFC test, island voltage/full output, charge (const voltage/current), backup test (single PFC), grid parallel test, LLC high/low side open-loop, LLC soft-start, LLC closed-loop charge, single PFC charge/grid parallel/island |
| `08017fa9`–`08018469` | `版本信息`, `打印数据`, `打印日志`, `设置下垂n`, `设置同步整流系数`, `打印错误日志`, `开启日志自动打印`, `设定pf值` etc. | Version info, print data, print log, set droop, set SR coeff, print fault log, enable auto log, set pf value, set reactive power/mode, RTOS/IO status, fan duty, PFC current limit, set burn-in mode (complete list: 35 entries, sensible English short descriptions of respective shell help texts) |
| `080185e8` | `任务名 任务状态 优先级 剩余栈 任务序号` | Task name / task status / priority / remaining stack / task number (header `rtos_status`) |
| `0801859c` | `1.RESET 复位控制状态，运行其他模式前需要先运行这个` | 1. RESET — reset control state, run before any other mode |

**From `Ctrl_HandleLlcPfcTestCommand` (`10000b58`, CCM) — factory test command outputs:**

| Address | Chinese | English |
|---|---|---|
| `10000e48` | `单PFC充电，母线电压 = %f\r\n` | Single PFC charge, bus voltage = %f |
| `10000e6c` | `过压测试\r\n` | Overvoltage test |
| `10000e7c` | `停\r\n` | Stop |
| `10000e90` | `整机充电，恒压 %f V，恒流%f A\r\n` | Full system charge, const voltage/current |
| `10000ebc` | `参数异常,要求40-60V，0-60A\r\n` | Parameter error, require 40–60 V, 0–60 A |
| `10000ee0` | `并网，功率 = %f\r\n` | Grid parallel, power = %f |
| `10000ef8` | `整机离网输出\r\n` | Full system island output |
| `10000f10` | `PFC开环占空比 %f \r\n` | PFC open-loop duty cycle |
| `10000f2c` | `PFC开环离网输出，电压峰值 %f V\r\n` | PFC open-loop island output, peak voltage |
| `10000f58` | `并机测试\r\n` | Parallel test |
| `10000f68` | `开环输出正弦波\r\n` | Open-loop output sine wave |
| `10000f80` | `单LLC开环充电,频率%fk\r\n` | Single LLC open-loop charge, frequency %f kHz |
| `10000fa0` | `参数异常\r\n` | Parameter error |
| `10000fb0` | `单LLC开环放电,频率%fk\r\n` | Single LLC open-loop discharge |
| `10000fd0` | `单LLC软起\r\n` | Single LLC soft-start |
| `10000fe0` | `参数异常,要求349V - 501V\r\n` | Parameter error, require 349–501 V |
| `10001004` | `单PFC并网，并网功率 = %f\r\n` | Single PFC grid parallel |
| `10001028` | `进入标准模式\r\n` | Entering standard mode |
| `10001040` | `进入恒压输出模式\r\n` | Entering constant voltage output mode |

All 184 strings were decoded; the above tables contain all entries with discernible flowing text/status content.
Purely numeric/format duplicates (e.g., multiple help text variants) are shown consolidated.

## 10. Open points

- Grid standard table `g_gridStandardTable` (Modbus ~43100–43129): only 2 of ~19 individual registers
  exactly verified.
- Fault/warning codes `0xd`, `0x1b`–`0x28`, `0x39` in `Log_PrintFaultEntry` without plain-text branch.
- Second fault ring `g_eventLogRing`/`Log_RecordFaultEvent` without discernible print routine.
- Handle name duplication `g_canRxTaskHandle` in three FreeRTOS tasks (`vtask_prt`, `vtask_can`,
  `vtask_can_receive`) — true multi-use or name artifact not resolved.
- CRC16 protection of YMODEM packet trailers not demonstrated in code (possibly not implemented
  or optimized away).
- `Ctrl_MainControlIsr` (12,446 B) not fully decoded state-by-state.
- `Prot_SetFaultBitOnMatch`: comparison constant/purpose not resolvable.
- Purpose of CAN command `0xC3` (raw data passthrough) and extended ID filter `0x4000`/`0x4100`
  mapping to `BMS_FwUpdate_CanHandler` cases not resolved.
- Internal layout of EEPROM blocks `0x001` (47 B), `0x100` (bytes 3–4), `0x300` (124 B), sub-block
  `0x021` not traced from code.
- Behavior of `EEPROM_SaveTimestamp` (write tick, immediately overwrite with 0) — purpose unclear.
- Meaning of trailer word at image end (`"13DD"` in v116, `"I1DD"` in v115) not determined.
- Real letter-shell password byte not decoded from raw `ShellCommand[53]` array (only indirectly
  via diff artifact as "empty" confirmed).
- Exact command semantics of `Ctrl_HandleLlcPfcTestCommand` (difference command 1 vs. 10 etc.)
  not fully decoded.
- Origin/call chain setting `g_emsDispatchSeq` not traced to triggering CAN frame.
- Additionally ~354 function/global names from name evidence files (`out_*.json`/`gout_*.json`)
  with `confidence: low` (mostly callback/veneer/thunk classes without callers in respective naming batch) —
  details see `.../scratchpad/micro116/out_00.json`…`out_03.json` and `g/gout_00.json`…`gout_03.json`.

## 11. Corrections against old analysis

**Against `VNSD_Micro_Inverter_FW_Analysis_vd_inv_app_0116.md`/`VNSD_Error_Codes_Micro_FW_Analysis.md`
(old base 0x08000000):** All Flash addresses there are 0x4000 too low; MCU was incorrectly called
STM32F3 instead of **STM32G474**, and the complete CCM code portion (`0x10000000`ff., incl. both central
control ISRs) was completely missing. The field-verified `err2` bit mapping and fault/warning code meanings
of old documentation match **exactly** with this session (§6) — only code addresses have shifted. Further
name corrections found this session (from comparison of name evidence files): I2C TX/RX swap in
`HAL_I2C_Master_Receive`/`_Transmit` in old version corrected; `I2C_ITTransferEnd` incorrectly labeled
as "UART_" function, actually I2C ISR status register; `RCC_GetPLLOutputClockFreq`/`RTC_EnterInitMode`/
`RTC_ExitInitMode` incorrectly labeled as I2C functions in old version (fallout from wrong base offset) —
now correctly mapped to RCC/RTC peripherals.

**Corrections to pre-computed function diffs (v115↔v116) — diff tool artifacts, refuted in this session
by direct decompilate comparison:**

- `ShellCmd_bat_mode`/`ShellCmd_pf_set`: no new behavior. Both functions are in v115 and v116 byte-identical
  (38 and 48 B respectively); the code block reported as "newly inserted" by diff (float debug printf + tail call)
  actually belongs to unsymbolized neighboring code (duplicate/variant of `ShellCmd_pi`/`ShellCmd_pf_power`),
  which the diff tool misattributed due to unclear function boundaries.
- `EXTI9_5_IRQHandler`: real interrupt handler (`0x08006994` in v116 / `0x0800699c` in v115, each 24 B)
  is identical in v115 and v116 (tail jump to common EXTI latch routine). The growth reported by diff concerns
  a **different, unnamed** function directly after in Flash (`0x0800699c` in v116), which is a regular `bl`
  target called from `Inv_BuildTelemetryBlock` — not an interrupt handler. **Further correction this session:** The
  function change at this location previously (also in §0 here) rated as "real" is itself an artifact — byte-by-byte
  comparison shows code identical to v115 function `EMS_ResetMemoryByLevel` (`0x080069a4`, 86 B); the function
  simply lost its name and Ghidra function registration in v116 (appears there only as code address `SUB_0800699c`,
  not as separate function). The code itself, including surrounding debounce/dispatch logic in `Inv_BuildTelemetryBlock`,
  is unchanged. There is **no** v115→v116 behavior change at this location.
- `EEPROM_SaveOperatingStats`: **no** new v116 function. The 4-byte trampoline code (`EEPROM_WriteVerify(0x500,&g_statsLastDay,0x30)`)
  already existed at identical address in v115 — there under generic collective name `Stub_EmptyNoOp2`
  (apparently assigned to two different, differently-sized addresses: one real 2-byte `return` stub and this
  4-byte trampoline). v116 naming run correctly named the trampoline independently; that is a naming improvement
  of this analysis session, not firmware change.
- **Really confirmed** (not refuted): `Stub_CallHandlerOffset28` — v115 (`0x0800fff0`, 2 B) is pure `return`
  stub, v116 (`0x0800ffee`, 10 B) really calls `HAL_UART_AbortCpltCallback`; both addresses and sizes were
  directly cross-checked in both decompilates.
- **EEPROM_ReadRegister/_NoMutex (formerly EEPROM_WriteRegister/EEPROM_ReadRegister):** The I2C direction
  finding formulated as suspicion in §0/§4.2/§8 of this session is confirmed. Fresh export correctly names
  `0x08006ce8` as `EEPROM_ReadRegister` (I2C read with mutex) and `0x08006d84` as `EEPROM_ReadRegister_NoMutex`
  (same flow without separate mutex management). The previously open point in §10 on I2C read direction thus
  becomes resolved; `0x55DD` branch of Modbus register 45000 definitely reads `g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode`
  from EEPROM back rather than saving them.

**Lesson for future diffs:** The function diff tool used (`l115_116.json`) apparently determines function
boundaries partly via next known symbol rather than actual function size, thus creating false "growth" findings
for functions with unsymbolized neighboring code. All diff results used in this analysis were therefore
cross-checked against actual function sizes in both `_index.md` files and/or real decompilate content before
being accepted.
