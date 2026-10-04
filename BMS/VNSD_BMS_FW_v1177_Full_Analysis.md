# BMS-FW v117.7 (Beta, Venus D) — Full analysis

> 🇩🇪 Deutsche Version: [VNSD_BMS_FW_v1177_Vollanalyse_de.md](VNSD_BMS_FW_v1177_Vollanalyse_de.md)

**Status:** 2026-09-30.

**German:** `VNSD_BMS_FW_v1177_Full_Analysis.md` ·
**Diff to successor v118:** `VNSD_BMS_Diff_v1177_v118.md` ·
**Full analysis v118:** `VNSD_BMS_FW_v118_Full_Analysis.md` (done in parallel from other source —
here only linked, not edited) ·
**Related:** `VNSD_Charge_Limits_Charge_Power.md` (charge limits BMS/Control/Micro, verified) ·
**Old analysis (background, base address wrong — see §13):** `VNSD_BMS_FW_Analysis_v117.7.md` ·
**Decompilation/Listing:** `Dekompilate/VNSD_BMS_1177/` (local only, not in the repo) (710 functions, `listing/` with Flash raw disassembly
+ `globals_and_registers.txt`)

## 0. Summary

BMS firmware **v117.7**, marked as **beta** (changelog `CHANGELOG.de.md` (local only, not in the repo): "v117.7 (Beta) —
2026-08-12", entries "OTA-Test" / "7"). Base address **`0x08002800`** (bootloader below),
image size `0x1A000` B (106,496 B), MCU **Nations N32G45x** (Cortex-M4F), Keil ARMCC, FreeRTOS
(`heap_4`, 10 tasks), letter-shell (36 commands, default user `BLUE`), FatFs, KA495xx cell monitor AFE
via SPI with CRC-8.

**Relation to v118:** 117.7 is in the firmware archive the **only existing predecessor** of v118 —
a beta of the (not archived) stable version v117. Code is in large parts identical to v118:
of 711 (118) resp. 710 (117.7) functions, **622 are byte-for-byte identical** (after address normalization)
and **77 more** differ only by relocation/decompiler artifacts
(no behavior change). Only **11 functions plus 1 completely new function** (`SOC_
OcvBoundsRecalibrateOnFlag`, only in 118) contain real changes — details and complete finding list in `VNSD_BMS_Diff_v1177_v118.md`. **This full analysis describes exclusively the
117.7 code** with its own addresses; where content is identical to 118, that is noted, but
always cited with the 117.7 address.

The main points in which 117.7 differs from 118 (details: diff document):

- `PackSelect_DischargeStepThreshold` (`0x08009354`) derives discharge rotation threshold from a
  **fixed** step table and ignores the already present configurable discharge limit
  percent `g_dischargeLimitPct` completely — in 118 this formula is replaced by one depending on the limit.
- The periodic CAN heartbeat frame (`CAN_RX_Handler`, `0x080081dc`) does **not** update
  `g_dischargeLimitPct` (only the dedicated CAN command `0x100301aa` does).
- No function `SOC_OcvBoundsRecalibrateOnFlag`; the flag it consumes in 118 (`g_socOcvRecalibrateRequest`
  `0x200029ca`) already exists in 117.7 RAM, but is likewise never written by any function — unused in both versions.
- Config version tag `g_cfgVersionTag` = `'u'` (0x75, represents "117"), not `'v'`.
- Reported pack FW version value `g_pack0FwVersionTag` = `0x499` (1177 decimal, beta count
  "117.7 × 10"), not `118`.
- No error timestamp log on `System_Reset_And_Halt_WithCurrentCheck` (`0x08004998`) — the
  power check before reset is missing.
- `PackMode_Shutdown_Handler` (`0x0800f324`) sets `g_pendingActivationFlag` immediately on entry
  to shutdown sequence, not after the 300 ticks wait.
- RAM: `.bss` `0x20002bc4`–`0x2000cb8f` (40,908 B, **16 B smaller** than 118), FreeRTOS heap **`0x4FF8`**
  (8 B larger than 118's `0x4FF0`), initial SP `0x2000CB90` (118: `0x2000CBA0`, exactly +16 B).

## 0a. Checklist vs. v118

| # | Question | Answer 117.7 | Evidence |
|---|---|---|---|
| 1 | CAN protocol as in 118? | Command set/message IDs identical. Only deviation: the periodic 0xfd00 heartbeat does **not** update `g_dischargeLimitPct` (this path exists only from 118 onward). | §4.1, diff §2.1 |
| 2 | RS485 as in 118? | Dispatcher structure/opcodes identical; version response byte `0x75` instead of `0x76`. | §4.2, diff §2.6 |
| 3 | Charge/discharge limits as in 118? | Hard limits (50 A, 57.6 V/48.0 V, temperature×SoC matrix) identical. The rotation threshold formula in `PackSelect_DischargeStepThreshold` differs (fixed table vs. limit-dependent formula). | §8, diff §2.1 |
| 4 | AFE/KA495xx driver as in 118? | Identical (only address shift). | §5 |
| 5 | OTA/bootloader handshake as in 118? | Identical: base `0x08002800`, tag `'BMS1'` @ `0x08010000`, staging @ `0x0801E000`, tag check @ `0x0802B800`. | §10 |
| 6 | Shell commands as in 118? | Identical (36 entries, user `BLUE`). | §11 |
| 7 | RAM layout as in 118? | `.bss` 16 B smaller, heap 8 B larger, six pack/rotation globals from `0x20002a58` lie 2 B "earlier" than in 118 (no `g_dischargeStepRotateLimitCache`). | §2, diff §2.5 |

## 1. Binary Fingerprint

| Feature | Value |
|---|---|
| File | `1177_bms_VNSD-0_20251010135647565eb2036.bin` |
| Size | 106,496 bytes (`0x1A000`) |
| SHA-256 | `69b18aa42831536f01546bbe707e36ee65b6faa84e6cf4e92f9fbfe237f67c77` (verified per `shasum -a 256`) |
| Link address | `0x08002800`–`0x0801C7FF` (bootloader below `0x08002800`) |
| MCU | Nations N32G45x (Cortex-M4F, Thumb-2); ADC1/ADC2 @ `0x40020800`/`0x40020C00`, UID @ `0x1FFFF7F0`, 102 vectors (86 IRQ + 16) |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW segment) |
| Libraries | FreeRTOS (`heap_4`), letter-shell, FatFs |
| Initial SP | `0x2000CB90` (from bytes 0–3 of image) |
| Reset vector | `0x08002A6D` (Thumb) → `Reset_Handler` (`0x08002a6c`) |
| Version constant | `'BMS1'` (`0x31534D42`) @ `0x08010000` |
| OTA staging | Copy @ `0x0801E000` (outside running image), tag check @ `0x0802B800` (`g_otaStagingVersionTag`) |
| Manufacturer changelog | v117.7 (Beta) — 2026-08-12: "OTA-Test", "7" (Issue #19) |

**Scatter-Load:** `.data` (`0x08002800`+offset, LZ77-compressed, 0x3C4 B) → `0x20002800`; `.bss`
(`__scatterload_zeroinit`) → `0x20002bc4`–`0x2000cb8f` (40,908 B); FreeRTOS heap `ucHeap` @ `0x200050d8`,
size `0x4FF8` (20,472 B). Initial SP `0x2000CB90` lies exactly at end of this memory range.

## 2. MCU / Memory Map / Ghidra Status

**Test status** (`Dekompilate/VNSD_BMS_1177/_index.md`):

| Check | Result |
|---|---|
| Functions | **710**, of which 710 named, 0 `FUN_*` |
| Ghidra symbols | 3,761 |
| Decompilation failed | 0 |
| Comparison to 118 | 711 functions (118) = 710 (117.7) + 1 new (`SOC_OcvBoundsRecalibrateOnFlag`) |

All verification gates (complete naming, no failed decompilations, no orphaned `FUN_*` remnants) passed — per the verification scheme established for v118. The naming was **derived from v118**: 622 of the 710 functions are byte-identical to their v118 counterpart (address-normalized) and adopt its already verified names unchanged; the remaining 88 were individually checked against actual 117.7 code (see diff document, §1).

**RAM/peripheral blocks** (`listing/globals_and_registers.txt`):

| Block | Range | Size |
|---|---|---|
| `.data` | `0x20002800`–`0x20002bc3` | `0x3C4` B (964 B) |
| `.bss` | `0x20002bc4`–`0x2000cb8f` | `0x9fcc` B (40,908 B) — **16 B smaller than 118** (`0x9fdc`) |
| FreeRTOS heap (`ucHeap`) | `0x200050d8`ff. | `0x4FF8` B (20,472 B) — **8 B larger than 118** (`0x4FF0`) |
| `flash_upper` (reserve/OTA staging) | `0x0801c800`–`0x0803ffff` | contains `g_otaStagingVersionTag` @ `0x0802b800` |
| `sysmem_info` | `0x1ffff000`–`0x1ffff7ff` | UID @ `0x1FFFF7F0`, alt-UID @ `0x1FFFF270`, factory word @ `0x1FFFF298` |

Initial SP `0x2000CB90` = end of `.bss` + 1, exactly consistent with scatter-load calculation.

## 3. FreeRTOS Tasks

From `RTOS_CreateAllTasks` (`0x0800e750`, 212 B) — ten direct `xTaskCreate()` calls:

| Task | Entry address | Stack (words) | Priority | Handle |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800e9c5` (Thumb) | `0x80` | 5 | `0x20002b08` |
| `RTOS_vTaskKeyScan` | `0x0800e9ab` | `0x80` | 9 | `0x20002b0c` |
| `RTOS_vTaskKA495XX` | `0x0800e99d` | `0x80` | 8 | `g_ka495xxTaskHandle` |
| `RTOS_vTaskCanRec` | `0x0800e945` | `0x180` | 4 | `0x20002b14` |
| `RTOS_vTaskCanSend` | `0x0800e955` | `0x80` | 3 | `0x20002b18` |
| `RTOS_vTaskRS485Comm` | `0x0800e9fb` | `0x180` | 1 | `0x20002b1c` |
| `RTOS_vTaskSocAlgorithm` | `0x0800ea09` | `0x100` | 7 | `0x20002b20` |
| `RTOS_vTaskProtectCheck` | `0x0800e9e5` | `0x100` | 6 | `0x20002b24` |
| `RTOS_vTaskControlLogic` | `0x0800e95f` | `0x80` | 2 | `0x20002b28` |
| `RTOS_vTaskDataSave` | `0x0800e979` | `0x80` | 0 | `0x20002b2c` |

Byte-for-byte identical to 118 except for the ten relocated entry addresses (verified, see
diff document §1). `RTOS_vTaskKA495XX` calls `KA495XX_TaskStateMachine` (`0x0800b1e0`) every 5 ticks.
`RTOS_vTaskSocAlgorithm` drives `SOC_AlgorithmOrchestrator` (§7). IDLE task and scheduler start follow
standard FreeRTOS pattern (`vTaskStartScheduler` `0x080192dc` → `xPortStartScheduler` `0x08019530`).

## 4. Interfaces

### 4.1 CAN

`CAN1_Init_And_CreateQueues` (`0x08003b88`, 284 B) initializes bxCAN (pure CAN1, no I2C part
despite earlier misname, see naming note in evidence). `CAN_RX_Handler` (`0x080081dc`,
104 B) decodes the extended arbitration ID:

| Command/range | Meaning |
|---|---|
| `0x100101aa` | → `CAN_CMD_01_Handler` |
| `0x100201aa` | → `CAN_CMD_02_Handler` |
| `0x100301aa` | Set discharge limit percent: `g_dischargeLimitPct = payload[0]`, persisted via `APP_WriteFlash_DischargeLimitPct` |
| `0x100401aa` | → `CAN_CMD_04_Handler` |
| `0x100501aa` | Payload `0x01` → `g_canRxRestartFlag_a70 = 1` |
| `0x180101aa`/`0x180102aa` | → `CAN_TX_Orchestrator` (pack query, broadcast/targeted) |
| Class `0xfd00` (heartbeat/master-enable) | `g_masterEnableReceivedFlag` (on payload byte `0x01`), `g_lastCanCmd04PayloadFlags`, `g_packOnlineWatchdogTimerMs=0`, `CAN_TX_Orchestrator(0xff)`. **Does not update `g_dischargeLimitPct` in 117.7** (first in 118, see diff §2.1). |

`CAN_RX_Command_Dispatcher` (`0x0800f0bc`, 324 B) handles a second ID range (opcode in
lowest byte, incl. `0xCE` = remote reset/OTA-related, `0x01`–`0x0A` status/mode commands).
`CAN_CMD_Pack_Activation` (`0x0800df04`, 396 B) handles pack activation/deactivation; opcode `0xCE`
with sub-ID 0 arms OTA takeover (`g_otaFinalizeRequestedFlag`-like flags), sub-ID 4 routes to
`CAN_RxSlavePackTelemetry_Unpack` (`0x0800db88`, 311 B, pack status bitfields). Responses run via
`CAN_TX_Orchestrator`, `CAN_TX_PF2_Capacity` (`0x08007bd8`, capacity) and `CAN_TX_PF3_
ChargeDischargeLimits` (`0x08007c74`, 318 B, charge/discharge limit broadcast, see §8).

### 4.2 RS485

`RS485_Dispatcher` (`0x08010004`, 717 B) — opcode-based protocol on USART2
(`0x40004400` → `0x20002BC4` ring buffer, `RS485_USART2_Init_DMA_RingBuffer`, `0x0800518c`). Noted opcode findings: `0x29`/`0x2e`/`0x30` copy large pack struct blocks (`0x60`/`0x54`/`0x2c` B) under raised base priority (interrupt-sensitive bulk transfers); `0x34` pulses relay pin (GPIOB
`0x1000`) for 80 ms; `0x2` triggers same DSB/AIRCR reset sequence as `HAL_NVIC_SystemReset`.
Version response byte in buffer = `0x75` (117, see §0). `RS485_Register_Read_Handler` (`0x08009f78`,
1,438 B) and `RS485_Register_Write_Handler` (`0x0800a524`, 758 B) are group-ID-based
config parameter accesses (groups 1:1 between read/write paths); unassigned group IDs
fall back to `RS485_RegisterWrite_DefaultGroup` (`0x0800a62a`). `RS485_Pack_Telemetry_Response`
(`0x080123a4`, 120 B, opcode `0x2e`) delivers the pack telemetry record incl. version byte `'u'` at
offset `0x48`. `RS485_OTA_Flash_Handler` (`0x0800c9d8`, 728 B) — see §10.

### 4.3 Debug Shell (letter-shell)

`shellAdd` (`0x08013e10`) registers the letter-shell instance; `CLI_ShellInit` (`0x0801474c`, 60 B)
logs in default user **`BLUE`** (`CLI_LoginUser`, `0x08014c60`) and confirms a command table with **36 entries** (naming evidence, `shellAdd` note: "Confirms the shell command
table has 36 entries"). `CLI_ValidateLoginPassword` (`0x08013e48`, 52 B) checks the password — per
diff artifact in literal pool the stored password is empty (not re-decoded in this session, see §12). `CLI_ProcessCommandLine`/`CLI_RunExtendedCommand`/`CLI_ResolveVariableReference`
implement letter-shell-typical command processing incl. `$variable` resolution and
tab completion. CAN/RS485-adjacent test commands exist (`CLI_Cmd_SendTestCanFrame`,
`RS485_SetDebugCheckoutFlag`, `RS485_SetDebugCmdPendingFlag`).

### 4.4 I2C EEPROM

`I2C_EEPROM_Write_WithMutex` (`0x08008c18`, 84 B, bit-banged I2C, no hardware I2C peripheral access)
is the central write path, e.g. for SOC calibration (§7) and (from 118 onward) error/event timestamps
at address `0xfc02` (10 B, RTC date/time from `RTC_Read_DateTime`, `0x080076ce`). `SOC_
EepromLoadCalibration` (`0x08015e9c`, 204 B) and `SOC_SaveCalibrationIfChanged`/`SOC_
ReadAndValidateFlashCalibSlot` manage a wear-leveled log area with 50 slots @ 20 B
(1000 B total) for SOC calibration data.

## 5. KA495xx Cell Monitor (AFE)

16 cells, 5 NTC channels, SPI with CRC-8 (polynomial `0xD5`, table `g_ka495xxCrc8Table`, check function
`KA495XX_Crc8` @ `0x08016a58`, 16 B). Driver cluster (all 117.7 addresses): `INIT_KA495XX_SPI_GPIO`
(`0x08007960`), `KA495XX_SPI_Read_All_Registers` (`0x0800798c`), `KA495XX_SPI_Write_7Registers`
(`0x080079b8`), `KA495XX_SPI_Read_Register`/`_Write_Register_2` (`0x08007abc`/`0x08007aec`),
`KA495XX_TaskStateMachine` (`0x0800b1e0`, 156 B, poll state machine of RTOS task), `KA495XX_
VerifyConfigRegisterTable` (`0x080166c4`, 60 B), `KA495XX_CellBalanceChannelConfig` (`0x08016974`,
126 B — returns 0 if configured cell count outside 4–17).

Cell balancing (`KA495XX_CellBalanceDecision`): start at cell voltage ≥ 3501 mV **and**
(cell − minimum) ≥ 30 mV; stop at < 3500 mV **and** difference < 20 mV; global abort at
(max − min) < 30 mV or current < 1001 (raw). `KA495XX_ReadRegister` checks a checksum via
`HAL_ShortBusyWaitLoop`, whose exact algorithm is not reconstructible in decompilation (open point, §12). All KA495xx functions **identical** between 117.7 and 118 except address shift
(diff document, table §1).

## 6. Pack Management

**Rotation/selection:** `PackSelect_ModeBySoc` (`0x080092cc`, 124 B) selects based on SoC state
(full/normal/low/empty) one of the branches `PackSelect_Mode1_Full` (`0x08006174`), `_Mode2_LowSoc`
(`0x080062b8`), `_Mode3_Empty` (`0x08006008`). `PackSelect_Discharge_StepRotate` (`0x080063a0`, 91 B)
and `PackSelect_DischargeStepThreshold` (`0x08009354`, 26 B, formula see §8/diff §2.1) control
discharge pack switch based on sliding SoC threshold; after ~50 ticks below threshold the pack with
highest voltage is re-found (`PackSelect_Next_
AboveSocThreshold`, `0x08007174`) and handed over if needed (`PackSelect_CommitPendingRotation`,
`0x0800bca8`). `PackHandoff_StateMachine` (`0x080066dc`, 354 B) sets the charge MOSFET mask to 0 or
exactly one bit — **physically only one pack charges** (see §8). `OwnPack_MOSFET_Driver`/
`_MasterVariant` (`0x08006b60`/`0x08007220`) switch the charge MOSFET accordingly.

**Recovery/state:** `PackRecovery_LowCell_Charge_SM` (`0x0801070c`, 513 B) handles packs with
critically low cell voltage in a separate charge state machine. `PackMode_Dispatcher`
(`0x08009778`, 138 B) routes between activation, transitional (`PackMode_Transitional_Handler`) and
shutdown states (`PackMode_Shutdown_Handler`, `0x0800f324`, 81 B — three-stage: reset GPIOB pin/counter → wait 300 ticks (then in 118, but already in 117.7 **immediately** on entry
`g_pendingActivationFlag`) → periodic off broadcast → `System_Reset_And_Halt_
WithCurrentCheck`).

**Communication/relay:** `PackSelect_RelayCommStateMachine` (`0x0800b298`, 514 B) and
`PackSelect_TriggerWakePulse` (`0x080155a4`) control an RS485/relay handshake line between
packs; exact protocol semantics not fully reconstructed (open point, §12).
`BMS_AggregateSystemPackStats` (`0x0800b834`, 526 B) clears protect/status fields of failed packs. `PerPack_Struct_Builder` (`0x0800c770`, 580 B) builds the per-pack telemetry record (cell voltages,
NTC extrema, SoC, current, MOS temperatures, relay status, cycle count, protect/warning words,
version field `g_pack0FwVersionTag = 0x499` = 1177, see §0).

## 7. SOC / SOH

`SOC_AlgorithmOrchestrator` (`0x08015d20`, 157 instructions/346 B) runs in `RTOS_vTaskSocAlgorithm`
and orchestrates per tick: `SOC_State_Detect` (`0x080121d8`) → mode finding (1=charge, 2=discharge,
3=idle) → `Coulomb_Counting` (`0x08011b38`, 334 B) → `SOC_Smoothing` (`0x08011878`) →
`Charge_SOC_Integration`/`Discharge_SOC_Integration` (`0x080119a8`/`0x08011df8`) per mode.
In idle mode (3): SOH/cycle count update from coulomb throughput (`SOH_From_Throughput`
`0x08012ab4`, `CycleCount_From_Throughput` `0x08012274`) plus a **1-hour idle timer**
(`g_socIdleTimeAccumMs`), which on expiration — depending on charge/discharge direction before idle — triggers an OCV window check and possibly `NTC_ConvertAndUpdatePackTempExtremes`. **This is the actually effective OCV recalibration path** in 117.7 — the newly added separate function `SOC_OcvBoundsRecalibrateOnFlag` in 118 does not exist in 117.7 and per naming evidence for 118 is anyway unreachable code (no writer for required flag, see diff document §2.3).

`SOC_CoulombCountTick` (`0x08016598`, 284 B) runs in idle branch with a 0.4-per-call
downsampling pattern. Calibration data persisted via `SOC_SaveCalibrationToFlash` (`0x08015fdc`),
`SOC_SaveCalibrationIfChanged` (`0x08016490`) and `SOC_EepromLoadCalibration`/`SOC_
ReadAndValidateFlashCalibSlot` in a 50-slot wear-leveled log (§4.4); only the first four
32-bit words of each slot checked on load against `0xf4241` (1,000,000, fixed-point ceiling) —
the fifth word taken unchecked (possible validation gap, open point §12).
`SOC_UpdateDisplayValueAndFullFlag` (`0x08016ac0`) forces display SoC to 100.0% (1000)
if full-charge latch is set and not discharging.

## 8. Charge/Discharge Limits

Fully covers content of verified finding in `VNSD_Charge_Limits_Charge_Power.md` (there with v118 addresses); here the corresponding **117.7 addresses**:

- `Max_Charge_Current_Calc` (`0x080125f0`, 84 instr.) computes active pack allowed charge current from temperature (axis `g_tempAxisTable` @ `0x0801B846`, 18 support points) and SoC (axis @
  `0x0801B86A`) via a percent-of-50-A matrix (`g_chargeCurrentDerateMatrix` @ `0x0801B884`,
  18×12×`uint16`); result in 0.1 A (100% = 500 = 50.0 A).
- `Max_Discharge_Current_Calc` (`0x080126e8`, 93 instr.) uses same temperature axis
  (`0x0801B846`) plus a discharge current matrix @ `0x0801BA34` and two voltage bracket index tables
  @ `0x0801BA48`/`0x0801BA3E`.
- `Master_Build_ChargeCurrentLimit` (`0x0800bb90`, 174 B) limits active pack value once more to
  **500 = 50.0 A** and sets charge termination voltage fixed at `g_sysChargeVoltLimit_x10`
  (`0x20004a16`) = `0x240` = 576 → **57.6 V**. `Master_Build_DischargeCurrentLimit` (`0x0800bc58`,
  64 B) sets accordingly `g_sysDischargeVoltLimit_x10` (`0x20004a1a`) = `0x1E0` = 480 → **48.0 V**.
  **Clarification** (per letter, corrected from earlier notes): both globals are **voltages**
  (57.6 V/48.0 V), not currents.
- `PackHandoff_StateMachine` (`0x080066dc`) enforces charge MOSFET mask never has more than one bit
  set → **physically only one pack charges**, independent of pack count.
- `CAN_TX_PF3_ChargeDischargeLimits` (`0x08007c74`, 318 B) sends current/voltage limit per CAN.

Result (unchanged to 118): maximum 50.0 A / 57.6 V per active pack, no summing across packs;
together with Micro/Control limits (2500 W) yields the overall limit described in `VNSD_Charge_Limits_Charge_Power.md`. The **only** 117.7-specific deviation affects not these limit values themselves, but the rotation threshold formula in `PackSelect_
DischargeStepThreshold` (§6, diff §2.1): in 117.7 this formula **does not** depend on
`g_dischargeLimitPct`, but uses a fixed 10%-step table with minimums 9/10.

## 9. Protection

`Cell_OVP_UVP_TempCompensated_Protection` (`0x0800cd18`, 441 instr.) and `Voltage_Temp_
Protection_Checker` (`0x0800c0b8`, 705 instr., despite name only temperature protection) form the
core of voltage/temperature monitoring. `Current_Protection_Checker` (`0x0800ac20`, 544 B)
implements two-stage OCP for charge and discharge direction plus a hardwired short-circuit-like trip. `HW_OvercurrentAndSensorProtection` (`0x0800aff4`, 366 B) additionally covers
NTC/MOS temperature plausibility. Bitmask construction: `Protect1_Bitmask_RawBuilder` (`0x0800da80`)
builds raw mask, `Protect1_Bitmask_BuilderWithDebounce` (`0x0800da30`) applies debounce; `Protect2_Bitmask_Builder` (`0x0800db30`) and `Warning_Bitmask_Builder` (`0x0800d980`) supplement
protect2/warning word. `BMS_ProtectionMasterTick` (`0x0800bac0`, 116 B) is the periodic
top-level dispatcher (from `RTOS_vTaskProtectCheck`). `AFE_CommLine_FaultChecker` (`0x0800bcbc`) and
`HW_RelayFaultChecker` (`0x0800bd48`) check AFE communication line resp. relay hardware fault.
`Cell_Balancing_PreconditionManager` (`0x0800f3e0`, 342 B) gates balancer based on
temperature/voltage thresholds (no OTA ref despite older misname). All stated functions
**identical** between 117.7 and 118 — full C comparison (not just listing diff) shows
no threshold value changes (diff document, §3).

## 10. OTA / Bootloader

Image base `0x08002800`, version tag `'BMS1'` (`0x31534D42`) @ `0x08010000`, staging copy of new
image @ `0x0801E000` (outside running app range `0x08002800`–`0x0801C7FF`), tag check of
staging copy @ `0x0802B800` (`g_otaStagingVersionTag`). `BMS_OTA_Flash_Writer` (`0x0800d1f8`, 584 B)
processes sub-opcodes (0=start/1=append/2=block-commit) from CAN transport
(`CAN_CMD_Pack_Activation` opcode `0xCE`); `BMS_OTA_Flash_Writer_FinalizeTail` (`0x0800d440`, 142 B)
closes write and calls `OTA_Set_Reboot_Params` (`0x080115a8`) — the "hand off to bootloader" — both from here and from `RS485_OTA_Flash_Handler` (`0x0800c9d8`,
728 B, RS485 transport path). `LIB_CRC16_Modbus_Calc` checks OTA flash blocks,
`Flash_AddressToSectorIndex` maps addresses to sector indexes. `HAL_Flash_Write_Config_Block`
(`0x0800ff98`, 74 B) writes to shared boot/app selection area `g_bootConfigPage`
(`0x08002000`, below app image). All stated OTA functions **identical** between 117.7 and 118 except address shift.

## 11. Security

- **Shell login:** default user `BLUE`, password check via `CLI_ValidateLoginPassword`
  (`0x08013e48`); per diff artifact in literal pool the stored password is empty — the
  password byte itself not re-decoded from raw binary in this session (open point).
- **Config integrity:** `g_cfgVersionTag` (`'u'` = 117) checked on boot against expected value
  (`BMS_Config_Params_Load_Or_Default`, `0x08009c60`); on mismatch factory reset of parameter block. `LIB_Checksum_Ones_Complement` (`0x0800b18c`) additionally secures config block
  via ones-complement checksum.
- **Error lock:** `APP_SetErrorLockFlag`/`APP_CheckErrorLockStatus` (`0x0800c028`/`0x0800c068`)
  implement persistent error lock (no AFE register access despite earlier misassignment), checked on power-on (`APP_CheckErrorLockStatus`, sole caller is init routine). `APP_EnsureFactoryMarkerByte` (`0x0800af40`) manages a factory marker byte, whose exact purpose is not fully clear.
- **OTA authentication:** no cryptographic signature check found in decompilation; sole integrity check is CRC16 block check (`LIB_CRC16_Modbus_Calc`) plus version tag comparison (`'BMS1'`). No encryption indication on OTA image.
- **I2C EEPROM:** bit-banged software implementation without hardware I2C peripheral; no
  access protection beyond mutex serialization (`I2C_EEPROM_Write_WithMutex`).

## 12. Open Points

- **KA495XX checksum algorithm** (`KA495XX_ReadRegister` → `HAL_ShortBusyWaitLoop`): the
  actual byte checksum calculation is not visible in decompilation (likely decompiler artifact on register tracking); algorithm (XOR/sum) not reconstructed.
- **KA495XX register loops** (`KA495XX_VerifyConfigRegisterTable`, `KA495XX_
  CheckFaultRegisters`): loop decompilation shows lost register tracking
  (`extraout_r2`/`r3`); abort condition inferred from surrounding pattern, not directly read.
- **Shell password:** not re-decoded from raw binary; earlier analysis noted empty
  password per literal pool comment.
- **`PackSelect_RelayCommStateMachine`/`PackSelect_TriggerWakePulse`:** exact
  handshake protocol semantics of relay line between packs not fully reconstructed.
- **SOC calibration validation:** `SOC_ReadAndValidateFlashCalibSlot` checks only first four of
  five 32-bit words per slot against ceiling `0xf4241`; fifth word (likely
  `TarSoc`) taken unchecked — possible but unconfirmed validation gap.
- **`g_socOcvRecalibrateRequest`:** present in 117.7 but without any reader or writer in
  entire image — purpose and intended external trigger unclear (see also diff document §4).
- **BMS_InitDefaultProtectionConfig:** several threshold values (`0x20004104`–`0x20004112`) could not
  be assigned concrete physical meaning from this session (likely further temperature protection thresholds).

## 13. Corrections vs. Old Analysis (`VNSD_BMS_FW_Analysis_v117.7.md`)

The old analysis started from a **wrong base address `0x08000000`** (instead of correct
`0x08002800`) — every flash address in it is 0x2800 too low. Specific corrections:

| Point | Old analysis | This analysis |
|---|---|---|
| Base address | `0x08000000` | `0x08002800` (bootloader offset) |
| Flash range | `0x08000000–0x08019FFF` | `0x08002800–0x0801C7FF` |
| Initial SP | `0x2000CB90` (accidentally correct, since RAM addresses unaffected by flash base shift) | `0x2000CB90` (confirmed) |
| Reset handler | `0x08002A6D` | `0x08002A6D` (vector entry, thumb bit set) → `Reset_Handler` @ `0x08002a6c` |
| Functions | 552 / 552 (100%) | **710** / 710 (100%) — significantly more functions through better auto-analysis/naming, not different firmware |
| MCU | Not explicitly identified as N32G45x; peripheral addresses interpreted without official SVD | **Nations N32G45x**, confirmed via ADC1/ADC2 base addresses, UID address and vector table size (see §1) |
| Version typo | File/title originally misnamed "v177.7" (corrected in document itself by 2026-07-09) | Version **117.7** confirmed via `g_pack0FwVersionTag = 0x499` (1177 decimal) |
| Charge limits | In old analysis not backed by Ghidra-verified addresses | Fully verified and aligned with `VNSD_Charge_Limits_Charge_Power.md` (118 addresses) and 117.7 addresses in §8 this document |
| "148-W plateau" | In related pack rotation/SOH documents earlier described as BMS behavior | **Withdrawn** — the effect came from Omnibattery integration, not the BMS (see project memo); not repeated here |

All other content findings of the old analysis (task list, KA495xx as cell monitor IC,
CAN+RS485 communication paths) were re-verified against fresh 117.7 export and presented in
§§3–11 with correct addresses.
