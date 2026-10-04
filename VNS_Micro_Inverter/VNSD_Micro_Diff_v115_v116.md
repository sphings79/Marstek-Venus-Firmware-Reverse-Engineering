# Diff table Micro v115 → v116 (Venus D)

> 🇩🇪 Deutsche Version: [VNSD_Micro_Diff_v115_v116_de.md](VNSD_Micro_Diff_v115_v116_de.md)

**Status:** 2026-09-30.

**German:** `VNSD_Micro_Diff_v115_v116.md` ·
**Full analyses:** `VNSD_Micro_FW_v115_Full_Analysis.md` · `VNSD_Micro_FW_v116_Full_Analysis.md` ·
**Decompilates:** `Dekompilate/VNSD_Micro_115/` (local only, not in the repo) (696 functions) · `Dekompilate/VNSD_Micro_116/`
(693 functions), RAM layout identical.

**Method:** Function pairs via matching names — N: 614 name pairs compared, of which 61 changed
(of which 5 content/real), 4 only in v115, 0 only in v116, 549 byte-identical (raw diff
`l115_116.json`, normalized opcodes). Each of the 61 marked "changed" functions and all
4 only-in-v115 cases were in this session **also** verified by directly reading decoded C code of both versions
(not just raw opcode diff) — several raw diff findings were pure relocation, inlining, or Ghidra name loss artifacts,
see sections below. Additionally, CCM code was searched directly for literal `0x451C4000`/`0x45098000`
(2500.0f/2200.0f) rather than relying only on name diff, because this literal appears in Ctrl functions
without name change (e.g., `Ctrl_MainControlIsr`).

## Checklist

| # | Question | Answer | Evidence |
|---|---|---|---|
| 1 | Modbus registers changed? | Register table/addresses identical. Two value changes in `Modbus_HandleRegisterAccess`: register 43133/43134 upper limit `0x898`(2200)→`0x9c4`(2500); register 45502 test reference `-2200.0`→`-2500.0`. All magic codes (`0x55AA/0x55DD/0x55EE/0x55FF`) and remaining register table byte-identical. | Table "Real changes" |
| 2 | CAN protocol to EMS/BMS changed? | Only one constant in `EMS_Inverter_CAN_Dispatcher`: range upper limit for commands 0x03/0x04 (charge/discharge limit set) `0x899`→`0x9c5`. `BMS_CAN_Parser`, `BMS_FwUpdate_CanHandler`, `HAL_FDCAN_RxFifo0Callback`, `CAN_RX_DispatchTask` are per raw diff **byte-identical** (not even in 61-change list) — command set, message IDs, filters unchanged. | Table "Real changes" |
| 3 | Power 2200 W→2500 W — all findings? | **Six** real locations, see separate table below. One originally suspected seventh location (`Ctrl_ResetControlState`) is **refuted** — both versions already load 2500.0f there, byte-identical, not even in raw diff as "changed" but correctly in 549-"identical" group. | Table "2200→2500 W" |
| 4 | OTA/BMS update relay still possible? | Yes. `OTA_CompleteFirmwareUpdate` is only address-shifted (cosmetic), `OTA_SelfUpdate_StateMachine`, `BMS_FwUpdate_StateMachine`, `BMS_StartFwUpdateCan`, `BMS_FwUpdateStateMachineTask` not in change list (byte-identical). YMODEM self-update, CAN self-update path and BMS relay over CAN/RS485 are structurally unchanged. | Cosmetic list |
| 5 | Protection/fault codes changed? | No. `Log_PrintFaultEntry` and `Log_PrintDebugData` are pure address-shifted (cosmetic), fault code table and `err2` bit mapping logically unchanged; `Prot_LatchHwFaultToErrorWord` (CCM) not in change list. | Cosmetic list |

## Changelog → Code

| Changelog entry | Actual code change(s) |
|---|---|
| "VD 2500w" (sole v116 changelog line) | Six separate locations raise the same 2200-W limit to 2500 W (CAN dispatcher, two Modbus locations, two CCM constants in `Ctrl_MainControlIsr`, `Bat_Mode3RoundRobinTimer`) — see table below. CAN-side real-time hard limit (commands 0x56/0x58, ±2500 W) was already in v115 unchanged at 2500 W; v116 thus fixes inconsistency between already-open real-time path and multiple downstream config/init/ramp paths still capped at 2200 W. |
| "Adapted to Control firmware 147" (sole v115 changelog line, vs. not directly available prior version) | Not separately verifiable this session (no v114 decompilates available); not subject of this v115↔v116 diff. |

## 2200 W → 2500 W: all six findings

| # | Function | v115 address | v116 address | Change |
|---|---|---|---|---|
| 1 | `EMS_Inverter_CAN_Dispatcher` | `0x08005944` | `0x08005940` | CAN commands 0x03/0x04 (charge/discharge limit set): range check `< 0x899` (2201) → `< 0x9c5` (2501). |
| 2 | `Modbus_HandleRegisterAccess`, register 43133/43134 | `0x080164d8` | `0x080164e8` | `g_ratedPowerConfigA`/`B` upper limit `0x898` (2200) → `0x9c4` (2500). |
| 3 | `Modbus_HandleRegisterAccess`, register 45502 | `0x080164d8` | `0x080164e8` | Factory test command "discharge reference": `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)` → `(-2500.0)`. |
| 4 | `Ctrl_MainControlIsr` (CCM), grid connection ramp | `0x100029d8` | `0x100029cc` | Output/clamp limit of power PI controller at ramp start (counter=120 ticks): `2200.0` → `2500.0`. |
| 5 | `Ctrl_MainControlIsr` (CCM), thermal derating | `0x10005180` | `0x10005174` | Base of thermal derating formula (`(1.0-(T-64)/20)*base`): `2200.0` → `2500.0`. |
| 6 | `Bat_Mode3RoundRobinTimer` | `0x0801183c` | `0x08011834` | Two locations set `g_batPowerRefW` (multi-pack round-robin charge orchestration, battery mode 3) to `±2200.0` → `±2500.0`. |

**Refuted candidate:** `Ctrl_ResetControlState` (v115 `0x10005fe8`/`0x10005fd8` literal per version) was suspected
as seventh CCM location in first session pass because v116 has value `0x451C4000` (2500.0f) in literal pool. Byte-by-byte
comparison of raw disassemblies shows however: **v115 already loads the same value `0x451C4000`** at corresponding address
(`0x10005fe8`). Complete function body (130 B) is except for address shift identical — function therefore not in raw diff
(`l115_116.json`) as "changed", correctly in 549-"identical" group. There is **no** v115→v116 change here.

## Table of all real changes (5 functions, across 6 findings above)

| Function | v115 | v116 | Change |
|---|---|---|---|
| `Bat_Mode3RoundRobinTimer` | `0x0801183c`, 382 B | `0x08011834`, 378 B | See finding 6 above (2200→2500 W). Rest of logic line-identical, only stack variable renamed (cosmetic). |
| `Cfg_LoadFromEeprom` | `0x08006a70` | `0x08006a68` | Version/boot flag constant (`g_cfgVersionMarker`, EEPROM field compare on load) `0x73` (115) → `0x74` (116) — pure version stamp bump, expected change, else identical. |
| `EMS_Inverter_CAN_Dispatcher` | `0x08005944` | `0x08005940` | See finding 1 above (2200→2500 W). Function size 494→493 B (normalized opcode length), no other changes. |
| `Modbus_HandleRegisterAccess` | `0x080164d8`, 1,158 B | `0x080164e8`, 1,150 B | See findings 2+3 above (2200→2500 W). All remaining deviations in function body (label shift by relocation, removed dead store `puVar9=(byte*)0x898`, restructured if/else branch at one fork, apparently changed argument count of `Grid_Protection_SetLimits` — function does not use its parameters in either version) are provably cosmetic. |
| `ShellCmd_version` | `0x080156a4` | `0x080156b4` | `SOFT_VERSION` constant `0x73`(115)→`0x74`(116); build date/time string `"Jan 22 2026"`/`"15:55:59"` → `"Jul 2 2026"`/`"16:32:26"` — expected version/build metadata, no behavior change. |

All five functions were additionally verified beyond raw diff by fully reading both decompilates;
beyond stated points no further content differences.

## Cosmetic changes (56 of 61 marked "changed" functions)

All following functions are after C code inspection **functionally identical**
between v115 and v116; raw diff only flags address shift (relocation from code size changes elsewhere),
register/stack rename by compiler, or Ghidra decompiler artifacts (e.g., unresolved tail calls,
different parameter count recognition).

**Veneers** (3-instruction trampolines Flash→CCM; diff is only the `movw`/`movt` jump target
because CCM target function shifted by few bytes — target itself unchanged):
`Cfg_SetOperatingModeFlag_veneer`, `Cfg_SetPersistedFlag102_veneer`, `Cfg_SetStoredEnableFlag_veneer`,
`Ctrl_InitControlLoopStructures_veneer`, `Ctrl_MainControlIsr_veneer`, `Ctrl_SetPrClipFlag_veneer`,
`Grid_ScaleAdcMeasurements_veneer`, `Inv_CurrentLoopIsr_veneer`, `Prot_ClampComparatorThreshold_veneer`,
`Prot_SetFaultBitOnMatch_veneer`, `Prot_SetHwFaultLatchBit_veneer`.

**Pure relocation shift** (identical logic, only literal pool/return addresses/labels shifted):
`Dac_InitAll`, `Default_Handler`, `FDCAN_CopyMessageToRAM`, `FLASH_WaitForOperation`
(algebraically equivalent De Morgan transformation of same bit check, different instruction choice by compiler),
`HAL_ADC_Init`, `HAL_ADC_Start_DMA`, `HAL_FDCAN_GetRxMessage`, `HAL_RCC_ClockConfig`,
`HAL_UART_Transmit_DMA` (return address in both versions points to same handler `Stub_CallHandlerOffset28`,
only shifted, neighboring stubs reordered), `I2C_WaitOnFlagUntilTimeoutCommon`,
`Inv_BuildTelemetryBlock` (unresolved call `func_0x0800699c` in v116 is exactly same machine code as
`EMS_ResetMemoryByLevel` in v115, see "only in v115" below — no behavior difference),
`LED_TaskTick` (removed line is redundant self-store, dead code elimination with no effect),
`Log_PrintDebugData`, `Log_PrintFaultEntry`, `OTA_CompleteFirmwareUpdate`, `Task_CreateAllAndStartScheduler`
(task order, stack sizes 0x80/0x100/0x80/0x200/0x100/0x100/0x100/0x80 and priorities
0x10/0xf/0xe/9/8/6/7/0 explicitly byte-verified for all 8 tasks, unchanged), `Uart_DmaTxComplete16Bit`,
`Uart_DmaTxComplete8Bit`, `Uart_RxCompleteDispatch`, `Util_CopyWordsSrcDst`, `__main`, `__rt_lib_init`,
`printf`, `printf_FloatToDecimal` (only difference: unused high word of 64-bit return, caller reads it in neither version),
`printf_Format_Engine` (stack frame layout and variable naming shifted, but every bitmask/format character/branch 1:1 identical),
`prvTaskPriorityDisinheritAfterTimeout`, `pxPortInitialiseStack`, `shellInit`,
`shellInvokeCommand` (v116 decompiler does not resolve jump table for cases 0–4, pure display artifact, call logic for all 8 argument counts identical),
`sprintf`, `TIM_InitTim4InputCapture`, `xTaskResumeAll`, `EXTI9_5_IRQHandler` (real interrupt handler in both versions 24 B,
identical tail jump — size growth reported by diff actually concerned neighboring, v116-unnamed function `EMS_ResetMemoryByLevel`/`SUB_0800699c`,
see below), `EEPROM_SaveOperatingStats` (v115 file already filed under this name — no stub confusion in fresh exports —,
logically identical to v116, only widely shifted), `Stub_EmptyNoOp2` (empty `return` stub in both versions, 2–4 B epilog encoding difference),
`Startup_BranchToMain`, `Shell_TaskInit`.

**Shell commands, address/string shift only** (logic line-identical): `ShellCmd_bat_mode` (38 B in both versions,
**explicitly re-read** — definitively refutes original raw diff finding of new debug printf block),
`ShellCmd_pf_set` (48 B in both versions, likewise refuted), `ShellCmd_reset_memory`, `ShellCmd_rtos_status`,
`ShellCmd_show_power` (last `printf` call displayed as `printf_Format_Engine` tail call in v116 vs. as `printf` — decompiler display artifact,
no behavior change).

**Compiler inlining (function boundaries shifted, no behavior change):** `ShellCmd_power_cfg`,
`ShellCmd_qpr`, `ShellCmd_vol_pi` — in v115 these are 12-B tail-jump stubs to separate helper functions
(`Ctrl_ApplyLlcPfcTestParams`, `Ctrl_ApplyPrControllerParams`, `Ctrl_ApplyVoltagePiParams`,
all three only in v115 as separate function registration); in v116 compiler has inline-copied same code directly
into respective `ShellCmd_*` function (88–92 B, identical printf/target call content).
See "only in v115" below for details.

## Only in v115 (4 function names, 0 only in v116)

| Function (v115) | v115 address | Fate in v116 |
|---|---|---|
| `Ctrl_ApplyLlcPfcTestParams` | `0x08014f90` (80 B) | **Cosmetically removed:** code now directly inline in `ShellCmd_power_cfg` (`0x080127e4`, 92 B). Same printf string, same `Ctrl_HandleLlcPfcTestCommand` call. |
| `Ctrl_ApplyPrControllerParams` | `0x0801487c` (76 B) | **Cosmetically removed:** code now directly inline in `ShellCmd_qpr` (`0x080127d8`, 88 B). Same printf string, same `Ctrl_SetPrControllerTarget` call. |
| `Ctrl_ApplyVoltagePiParams` | `0x08014804` (76 B) | **Cosmetically removed:** code now directly inline in `ShellCmd_vol_pi` (`0x080127cc`, 88 B). Same printf string, same `Ctrl_SetRampTargetB` call. |
| `EMS_ResetMemoryByLevel` | `0x080069a4` (86 B) | **Name/registration loss, no code difference:** byte-by-byte identical machine code lies in v116 at same relative address (`0x0800699c`), but Ghidra's automated naming run did not assign function name there — appears in v116 export only as code address `SUB_0800699c`, not as separate function registration (not even in 693-function count). Surrounding call logic (debounce every ~10 telemetry cycles, `g_emsDispatchSeq` evaluation) in `Inv_BuildTelemetryBlock` is line-identical in both versions. **Correction:** earlier session rated this location as "real new program path in v116" (see `VNSD_Micro_FW_v116_Full_Analysis.md` §0/§11) — that is refuted, code is unchanged v115 without name. |

All four cases fully explain difference between 696 (v115) and 693 (v116) named functions
(3 inlining cases + 1 name loss = −4 in raw diff, plus scattered further naming run deviations not code-related for remaining total count difference).

## Open points

- No v114 decompilates available; v115 changelog line "Adapted to Control firmware 147"
  could not be verified against prior version this session.
- Exact cause of additional ~354 "confidence: low" names (see both full analyses, §10)
  was not re-verified for each individual compared function; where classification rested on low-confidence
  name, misattribution risk not fully excluded (affects in this diff table foremost `Grid_Protection_SetLimits`
  parameter count and similar signature recognition fuzziness, but not determinative for "cosmetic" classification).
- Renaming `EEPROM_WriteRegister`→`EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex` exists in
  **both** versions (v115 and v116) already identically — it is **not** a v115→v116 change,
  but pure documentation correction against older, now-superseded analysis version (details in both full analyses, §4.2/§11).
