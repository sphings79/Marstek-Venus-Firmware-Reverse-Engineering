# VNSD BMS — Diff Beta 117.7 → 118

> 🇩🇪 Deutsche Version: [VNSD_BMS_Diff_v1177_v118_de.md](VNSD_BMS_Diff_v1177_v118_de.md)

**Status:** 2026-09-30.

**German:** `VNSD_BMS_Diff_v1177_v118.md` ·
**Full analyses:** `VNSD_BMS_FW_v1177_Full_Analysis.md` (117.7, this session) ·
`VNSD_BMS_FW_v118_Full_Analysis.md` (118, done in parallel from other source — here only linked,
not edited) ·
**Related:** `VNSD_Charge_Limits_Charge_Power.md` (charge limits BMS/Control/Micro, verified) ·
**Export/source:** `Dekompilate/VNSD_BMS_1177/` (local only, not in the repo), `Dekompilate/VNSD_BMS_118/` (each `<addr>_<name>.c`,
`_index.md`, `listing/` with Flash raw disassembly + `globals_and_registers.txt`)

## Beta Note (prerequisite for this document)

In the firmware archive (`Marstek FW Archiv/firmwares/VNSD-0/BMS/`) there are for the BMS only two images:
`1177` (beta) and `118`. **No predecessor image for the 117.7 beta exists** — the likely predecessor would be the stable version **BMS v117**, which is missing from the archive. A diff 117 → 117.7 is thus **not possible**. This document compares exclusively **117.7 → 118** (the successor). The changelog (`CHANGELOG.de.md` (local only, not in the repo)) lists for both versions only "OTA-Test" (117.7 additionally "7"), no functional description — actual changes had to be fully derived from code.

The beta nature of 117.7 shows also in the code itself: `PerPack_Struct_Builder` reports in 117.7 the
firmware version value `0x499` (1177 decimal, i.e., "117.7" as running beta number `117.7 × 10`),
in 118 instead simply `0x76` (118) — two different count conventions for the same quantity.

## 0. Summary

**Method:** starting point is the normalized listing diff `l1177_118.json` (622 functions
identical, 88 changed, 1 only in 118 new: `SOC_OcvBoundsRecalibrateOnFlag`). For **each** of the 88
changed function pairs the decompiled C code of both versions was additionally
compared line by line (same-name files in `Dekompilate/VNSD_BMS_1177/` resp.
`VNSD_BMS_118/`), to separate pure address/register name artifacts from real behavior changes.

**Result: 11 of 88 listed changes are real, 77 are cosmetic.** Together with the one
completely new function `SOC_OcvBoundsRecalibrateOnFlag` there are **12 real changes**
total. All 77 cosmetic cases are either (a) pure address constants that shift automatically due to code size change (+88 B net from new function, +extensions at three places), (b) decompiler register names (`fVar2`/`fVar3`, `uStack_18`/`local_18` etc.) without meaning, or (c) follow-on shifts from a single new 2-byte
RAM global (`g_dischargeStepRotateLimitCache`, see §2.5). Each of these 77 cases was individually checked on C code, not just estimated from listing diff — with one function (`PackMode_Shutdown_Handler`) a pure listing heuristic would have falsely rated it "cosmetic"; only the C comparison revealed the real timing shift (§2.2).

**The most important real changes:**

1. **Discharge limit percent (`g_dischargeLimitPct`) now additionally propagated via periodic
   CAN heartbeat** and the pack rotation logic depending on it was repaired/extended
   (`CAN_RX_Handler`, `PackSelect_DischargeStepThreshold`, `PackSelect_Discharge_StepRotate`) — §2.1.
2. **New OCV recalibration function `SOC_OcvBoundsRecalibrateOnFlag`**, hooked into
   `SOC_AlgorithmOrchestrator`, plus two new RTC/EEPROM timestamp log entries on mode change to idle — §2.3. **Restriction:** the flag it consumes
   `g_socOcvRecalibrateRequest` is never set to 1 or 2 in any of the 710/711 functions — the function is per current knowledge **dead code**, the actual OCV recalibration path remains the existing 1-hour idle timer in `SOC_AlgorithmOrchestrator`.
3. **New error timestamp log on emergency reset** (`System_Reset_And_Halt_WithCurrentCheck`), if
   the last measured pack power was outside ±5 kW — §2.4.
4. **Config/protocol version tag `'u'` (117)→`'v'` (118)** at six places in sync
   (config guard, RS485 version, pack telemetry) — §2.6.
5. **Shutdown timing** (`PackMode_Shutdown_Handler`): `g_pendingActivationFlag` set after
   300 ticks wait time rather than immediately — §2.2.

**No changes** to: KA495xx SPI/CRC driver, charge current/voltage limit tables (50 A, 57.6 V/
48.0 V), core protocol of CAN/RS485 (command sets, message IDs), OTA/bootloader handshake
(`g_bootConfigPage`, staging address, `'BMS1'` tag check), shell command table (36 entries,
user `BLUE`), FreeRTOS task list (10 tasks, same priorities/stacks/handles).

## 0a. Checklist

| # | Question | Answer 117.7 → 118 | Evidence |
|---|---|---|---|
| 1 | CAN protocol changed? | Command set and message IDs unchanged. One real extension: the periodic 0xfd00 heartbeat frame (master-enable) now additionally carries `g_dischargeLimitPct` in byte `frame+0xd` (RAM update only, no flash write like dedicated CAN command `0x100301aa`). | §2.1, table §1 (`CAN_RX_Handler`) |
| 2 | RS485 changed? | Command set/dispatcher structure unchanged. Two version response bytes raised (`0x75`→`0x76`), else only address shifts. | §2.6, table §1 |
| 3 | Charge/discharge limits changed? | Hard limits (50 A, 57.6 V/48.0 V, temperature×SoC matrix) **unchanged** — see `VNSD_Charge_Limits_Charge_Power.md`. Only changed **how** an already present configurable discharge limit percent (`g_dischargeLimitPct`) affects the pack rotation logic (§2.1) — previously `PackSelect_DischargeStepThreshold` did not depend on this global at all. | §2.1 |
| 4 | OTA/bootloader handshake changed? | Unchanged: image base `0x08002800`, version tag `'BMS1'` @ `0x08010000`, staging copy @ `0x0801E000`, tag check @ `0x0802B800`, `BMS_OTA_Flash_Writer`/`_FinalizeTail` only address-shifted. | Table §1 (`BMS_OTA_Flash_Writer_FinalizeTail`, `OTA_Set_Reboot_Params`, `HAL_Flash_Write_Config_Block`) |
| 5 | Shell commands changed? | Unchanged (36 entries, default user `BLUE`); sole changed shell-adjacent function (`CLI_ShellInit`) differs only in local variable names. | Table §1 |
| 6 | RAM layout changed? | `.bss` in 118 **16 B larger** (`0x9fcc`→`0x9fdc`), FreeRTOS heap in 118 **8 B smaller** (`0x4FF8`→`0x4FF0`); one new 2-B global (`g_dischargeStepRotateLimitCache`) shifts six neighboring timer/state globals by +2 B. Remaining bytes of shift not attributable to single new functions (open point, §4). | §2.5 |

## 1. Table of All 88 Changed Function Pairs

Address 117.7 / address 118 / function / real or cosmetic / what changed. Additionally new in
118 (not in this table since no function pair): `SOC_OcvBoundsRecalibrateOnFlag` @ `0x08016634`
(146 B) — see §2.3.

| Addr 117.7 | Addr 118 | Function | Real/Cosmetic | What changed |
|---|---|---|---|---|
| `0x08018eb8` | `0x08018ff0` | `APP_RegisterCallbackSlot` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08003188` | `0x08003188` | `APP_ShellTable_Init_Then_Main` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08012cac` | `0x08012d1c` | `APP_SystemFaultHalt` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080029a0` | `0x080029a0` | `APP_SystemFaultHalt_thunk` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800a920` | `0x0800a98c` | `BMS_Config_Params_Factory_Reset` | **real** | Config version tag `g_cfgVersionTag` raised from `'u'` (0x75=117) to `'v'` (0x76=118). |
| `0x08009c60` | `0x08009ccc` | `BMS_Config_Params_Load_Or_Default` | **real** | Guard comparison and re-write value of config version tag raised from `0x75` to `0x76` (enforces factory reset of parameter block on tag mismatch). |
| `0x080096c8` | `0x08009734` | `BMS_Measurements_Valid_Check` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800d440` | `0x0800d4ac` | `BMS_OTA_Flash_Writer_FinalizeTail` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08009540` | `0x080095ac` | `BMS_ProtectionAndCommTick` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800e120` | `0x0800e18c` | `CAN_Pack_Activation_Config_Receive` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800f0bc` | `0x0800f128` | `CAN_RX_Command_Dispatcher` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080081dc` | `0x08008240` | `CAN_RX_Handler` | **real** | In 0xfd00 heartbeat branch (master-enable) additional line `g_dischargeLimitPct = frame[0xd]` — discharge limit percent now also updated via periodic heartbeat (RAM only, no flash write). |
| `0x0800db88` | `0x0800dbf4` | `CAN_RxSlavePackTelemetry_Unpack` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08007bd8` | `0x08007c3c` | `CAN_TX_PF2_Capacity` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08014470` | `0x080144e0` | `CLI_FindActiveShellInstance` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08014420` | `0x08014490` | `CLI_FormatVariableValueString` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08016044` | `0x080160e0` | `CLI_PrintSocStatus` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08010bcc` | `0x08010c2c` | `CLI_Printf_Wrapper` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800ec3c` | `0x0800eca8` | `CLI_Process_RX_Buffer` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08012c9c` | `0x08012d0c` | `CLI_SetDefaultCommandHandler` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801474c` | `0x080147bc` | `CLI_ShellInit` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800cd18` | `0x0800cd84` | `Cell_OVP_UVP_TempCompensated_Protection` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08002a86` | `0x08002a86` | `Default_Handler` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800ff00` | `0x0800ff6c` | `FATFS_Init_Log_File_If_Needed` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080129e4` | `0x08012a54` | `Full_OCV_Lookup` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800ff98` | `0x08010bd8` | `HAL_Flash_Write_Config_Block` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation); the literal address calculation `uVar2*4+0x8002000` in 117.7 equals `&g_bootConfigPage+uVar2` in 118 — same value, just symbol resolution different. |
| `0x0800fffa` | `0x0800fff8` | `HAL_NoOpStub` | cosmetic | No code difference; boundary shift next to `'BMS1'` version tag word from relocation of `__scatterload_copy` (see summary). |
| `0x08016974` | `0x08016ab0` | `KA495XX_CellBalanceChannelConfig` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08016a58` | `0x08016b94` | `KA495XX_Crc8` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080166c4` | `0x08016800` | `KA495XX_VerifyConfigRegisterTable` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08018a02` | `0x0800fffa` | `LIB_FreeRTOS_Clear_ListItem_Value` | cosmetic | No code difference; boundary shift next to `'BMS1'` version tag word from relocation of `__scatterload_copy` (see summary). |
| `0x080125f0` | `0x08012660` | `Max_Charge_Current_Calc` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080126e8` | `0x08012758` | `Max_Discharge_Current_Calc` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800e1e8` | `0x0800e254` | `NTC_Lookup_Table_1` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800e3d8` | `0x0800e444` | `NTC_Lookup_Table_2` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080115a8` | `0x08011618` | `OTA_Set_Reboot_Params` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08006b60` | `0x08006bc4` | `OwnPack_MOSFET_Driver` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08007220` | `0x08007284` | `OwnPack_MOSFET_Driver_MasterVariant` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800f324` | `0x0800f390` | `PackMode_Shutdown_Handler` | **real** | `g_pendingActivationFlag=1` moved from state 0 (immediately) to end of state 1 (only after 300 ticks wait) — real timing change to shutdown state machine. |
| `0x0801070c` | `0x0801070c` | `PackRecovery_LowCell_Charge_SM` | cosmetic | Pure follow-on from 2-B RAM shift due to new global `g_dischargeStepRotateLimitCache` (see §2.5) — same fields, just different address. |
| `0x08009354` | `0x080093bc` | `PackSelect_DischargeStepThreshold` | **real** | Formula completely replaced: instead of fixed step table (step width 5 per 10% SoC, minimum 9/10) threshold now derived from `100 - g_dischargeLimitPct` — rotation threshold now follows configurable discharge limit. |
| `0x080063a0` | `0x080063dc` | `PackSelect_Discharge_StepRotate` | **real** | New RAM cache `g_dischargeStepRotateLimitCache`: detects runtime change of `g_dischargeLimitPct` and then forces threshold recalculation instead of repeatedly resetting timer to 0. |
| `0x08006174` | `0x080061b0` | `PackSelect_Mode1_Full` | cosmetic | Pure follow-on from 2-B RAM shift due to new global `g_dischargeStepRotateLimitCache` (see §2.5) — same fields, just different address. |
| `0x080062b8` | `0x080062f4` | `PackSelect_Mode2_LowSoc` | cosmetic | Pure follow-on from 2-B RAM shift due to new global `g_dischargeStepRotateLimitCache` (see §2.5) — same fields, just different address. |
| `0x08006008` | `0x08006044` | `PackSelect_Mode3_Empty` | cosmetic | Pure follow-on from 2-B RAM shift due to new global `g_dischargeStepRotateLimitCache` (see §2.5) — same fields, just different address. |
| `0x08007174` | `0x080071d8` | `PackSelect_Next_AboveSocThreshold` | cosmetic | Pure follow-on from 2-B RAM shift due to new global `g_dischargeStepRotateLimitCache` (see §2.5) — same fields, just different address. |
| `0x08003798` | `0x08003798` | `Pack_Online_Watchdog` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800c770` | `0x0800c7dc` | `PerPack_Struct_Builder` | **real** | Reported FW version value `g_pack0FwVersionTag` in pack telemetry record changed from `0x499` (1177, beta count) to `0x76` (118). |
| `0x0800e608` | `0x0800e674` | `RCC_GetClockFrequencies` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08010004` | `0x08010004` | `RS485_Dispatcher` | **real** | Version response byte in dispatcher buffer raised from `0x75` to `0x76` (same config version constant as above). |
| `0x080123a4` | `0x08012414` | `RS485_Pack_Telemetry_Response` | **real** | Version byte in pack telemetry response (offset 0x48) raised from `'u'` to `'v'`. |
| `0x0800d888` | `0x0800d8f4` | `RS485_Protocol_Byte_Receiver` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08004ba8` | `0x08004be4` | `RS485_TxComplete_DMA_Callback` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800518c` | `0x080051c8` | `RS485_USART2_Init_DMA_RingBuffer` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0800e750` | `0x0800e7bc` | `RTOS_CreateAllTasks` | cosmetic | Only the 10 task entry addresses shifted (relocation); stacks, priorities and handles verified byte-identical. |
| `0x08015d20` | `0x08015d90` | `SOC_AlgorithmOrchestrator` | **real** | New call `SOC_OcvBoundsRecalibrateOnFlag(soc_state)` at start of tick; additionally two new RTC+EEPROM timestamp writes on entry to idle mode and on idle timeout (previously only SOH/cycle count update without timestamp). |
| `0x08012ce8` | `0x08012d58` | `SOC_InitCapacityParams` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08013d58` | `0x08013dc8` | `SOC_ResetEstimatorToPercent` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08015fdc` | `0x08016078` | `SOC_SaveCalibrationToFlash` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08016ac0` | `0x08016bfc` | `SOC_UpdateDisplayValueAndFullFlag` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08013cc4` | `0x08013d34` | `SOH_LookupByCycleCount` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08004998` | `0x08004998` | `System_Reset_And_Halt_WithCurrentCheck` | **real** | New condition before reset: if `|g_bmsPackPowerWatts/1000| > 5` (kW), RTC timestamp written to EEPROM fault history before MCU reset triggered. |
| `0x0800c0b8` | `0x0800c124` | `Voltage_Temp_Protection_Checker` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08012de8` | `0x08012e58` | `prvAddCurrentTaskToDelayedList` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08012e80` | `0x08012ef0` | `prvAddTaskToReadyList` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08012f38` | `0x08012fa8` | `prvCheckForValidListAndQueue` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08012f9c` | `0x0801300c` | `prvCheckTasksWaitingTermination` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801a9a8` | `0x0801aae0` | `prvCreateTimerTask` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080130dc` | `0x0801314c` | `prvHeapInit` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080131a8` | `0x08013218` | `prvInitialiseNewTask` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080132bc` | `0x0801332c` | `prvInitialiseTaskLists` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08013a10` | `0x08013a80` | `pvPortMalloc` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08013c28` | `0x08013c98` | `pxPortInitialiseStack` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08013e10` | `0x08013e80` | `shellAdd` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08018d98` | `0x08018ed0` | `vPortValidateInterruptPriority` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08019158` | `0x08019290` | `vTaskPriorityDisinheritAfterTimeout` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x080192dc` | `0x08019414` | `vTaskStartScheduler` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08019408` | `0x08019540` | `vTaskSwitchContext` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08019530` | `0x08019668` | `xPortStartScheduler` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801999c` | `0x08019ad4` | `xQueueGenericSend` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08019bd4` | `0x08019d0c` | `xQueueGenericSendFromISR` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08019d4c` | `0x08019e84` | `xQueueReceive` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x08019f38` | `0x0801a070` | `xQueueSemaphoreTake` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801a350` | `0x0801a488` | `xTaskIncrementTick` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801a518` | `0x0801a650` | `xTaskPriorityDisinherit` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801a674` | `0x0801a7ac` | `xTaskPriorityInherit` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801a73c` | `0x0801a874` | `xTaskRemoveFromEventList` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |
| `0x0801a838` | `0x0801a970` | `xTaskResumeAll` | cosmetic | Only address/register name differences (relocation from code growth or decompiler register allocation), no behavior change. |

## 2. Details of Real Changes

### 2.1 Discharge Limit Percent — New Propagation and Repaired Rotation Logic

`g_dischargeLimitPct` (RAM `0x200028e0`, ushort) already exists in 117.7 and is read there by
`APP_LoadOrInitDischargeLimitConfig`, `PackSelect_MosMismatch_Fallback`, `PackSelect_Next_
AboveSocThreshold` and `CAN_TX_PF2_Capacity`, or set via dedicated CAN command `0x100301aa`
and persisted to flash (`APP_WriteFlash_DischargeLimitPct`). Three places change in 118:

**a) `CAN_RX_Handler`** — in the branch for periodic master-enable/heartbeat frame (condition
`iVar2==0xfd00`) an additional line appears:

```c
// 118, after g_lastCanCmd04PayloadFlags = (uchar)frame[3];
g_dischargeLimitPct = (ushort)*(byte *)((int)frame + 0xd);
g_packOnlineWatchdogTimerMs = 0;
```

Discharge limit is thus now **additionally** taken from byte `frame+0xd` on every heartbeat
(pure RAM update, no `APP_WriteFlash_DischargeLimitPct` call as with dedicated command
`0x100301aa`). 117.7 lacks this line — `g_dischargeLimitPct` there changes only via dedicated CAN command or RS485/CLI config paths.

**b) `PackSelect_DischargeStepThreshold`** — the calculation formula was completely replaced:

```c
// 117.7: fixed step table, independent of g_dischargeLimitPct
uVar2 = (&g_pack0Soc_x10)[pack_idx * 0x30] / 10 & 0xff;
uVar1 = (((int)(uVar2 - 10) / 10) * 5 & 0x7fU) * 2;
if (uVar1 < 10) { if (uVar2 < 0xb) uVar1 = 9; else if (10 < uVar2) uVar1 = 10; }

// 118: threshold derived from configurable discharge limit
limit_pct = 100 - g_dischargeLimitPct & 0xff;
pack_soc_pct = (&g_pack0Soc_x10)[pack_idx * 0x30] / 10 & 0xff;
result = pack_soc_pct - 10 & 0xff;
if (result < limit_pct) {
  if (limit_pct < pack_soc_pct) result = limit_pct;
  else result = limit_pct - 1 & 0xff;
}
```

In 117.7 this function thus ignored `g_dischargeLimitPct` entirely and used a fixed 10%-step logic. In 118 the rotation threshold directly follows the configured discharge limit
— a real behavior change, not just renaming.

**c) `PackSelect_Discharge_StepRotate`** — new RAM cache `g_dischargeStepRotateLimitCache`
(`0x20002a58`, 2 B, see §2.5):

```c
if (g_dischargeStepRotateLimitCache == 0) g_dischargeStepRotateLimitCache = g_dischargeLimitPct;
if ((<SOC/current condition>) && (g_dischargeStepRotateLimitCache == g_dischargeLimitPct)) {
  g_dischargeStepRotateTimer = 0;           // "stay active" only if limit unchanged
} else {
  g_dischargeStepRotateTimer++;             // else timer counts toward recalculation
  if (0x32 < g_dischargeStepRotateTimer) {
    g_dischargeStepRotateLimitCache = g_dischargeLimitPct;   // cache update
    ...
  }
}
```

In 117.7 the rotation timer was unconditionally reset to 0 on met SOC/current condition — a runtime-changed `g_dischargeLimitPct` would not have affected the threshold
(`PackSelect_DischargeStepThreshold`) anyway there, but is now relevant in 118 and is picked up via this cache comparison within at most 50 ticks, instead of persisting indefinitely on old value.

**Practical impact:** in 117.7 the discharge limit percent had no effect on pack rotation threshold. In 118 it affects both the threshold formula and — via new heartbeat path — the update frequency.

### 2.2 Shutdown Timing: `PackMode_Shutdown_Handler`

```c
// 117.7 — state 0 (immediately on entry):
if (g_packModeShutdownState == '\0') {
  g_pendingActivationFlag = '\x01';
  g_pendingActivePackMask = '\0';
  ...
}
// 118 — same line missing in state 0, appears instead at end of state 1:
else if (g_packModeShutdownState == '\x01') {
  g_packModeShutdownTimer++;
  if (299 < g_packModeShutdownTimer) {
    g_packModeShutdownTimer = 0;
    g_pendingActivationFlag = '\x01';        // <- now here
    g_packModeShutdownState = '\x02';
  }
}
```

`g_pendingActivationFlag` is in 118 set only after the 300 ticks wait (state 1) rather than immediately on entry to shutdown sequence (state 0). This finding shows why pure listing heuristic is insufficient: the listing diff here shows a symmetric delete+insert of instruction blocks, looking superficially like pure reordering — only the C comparison reveals the actual shift of flag-setting across an entire state phase.

### 2.3 SOC/OCV Recalibration: New Function `SOC_OcvBoundsRecalibrateOnFlag`

New in 118, `0x08016634`, 146 B:

```c
void SOC_OcvBoundsRecalibrateOnFlag(int32_t *soc_state)
{
  if (g_socOcvRecalibrateRequest == '\x01') {
    if (<temperature outside knee band, see condition>) {
      NTC_ConvertAndUpdatePackTempExtremes(soc_state, 2);
    }
    g_socOcvRecalibrateRequest = '\0';
    RTC_Read_DateTime((int32_t *)g_socCalibTimestamp);
    I2C_EEPROM_Write_WithMutex(g_socCalibTimestamp, 0xfc02, 10);
  }
  else if (g_socOcvRecalibrateRequest == '\x02') {
    /* same structure without knee band check */
  }
}
```

Called as first action in `SOC_AlgorithmOrchestrator` once `g_socStateFlags` shows init bit. The consumed flag `g_socOcvRecalibrateRequest` (RAM `0x200029ca`)
exists already identical in 117.7 — there it has **no consumer**: no function in
117.7 reads or writes it. In 118 it is read exclusively by this new function; a
**writer** (setter) of this flag could not be found in either firmware version (open
point, §4). The naming determination for 118 independently reaches the same finding: the flag is never set to 1 or 2 anywhere in the image (only cleared to 0) — `SOC_OcvBoundsRecalibrateOnFlag`
is thus **practically unreachable code**. The actually effective OCV recalibration continues running via
the existing 1-hour idle timer further down in `SOC_AlgorithmOrchestrator`
(`g_socIdleTimeAccumMs` overflow, unchanged in both versions).

Additionally 118 adds in `SOC_AlgorithmOrchestrator` itself two new RTC+EEPROM timestamp write operations (`RTC_Read_DateTime` → `I2C_EEPROM_Write_WithMutex(…, 0xfc02, 10)`): once on first entry to idle mode (`mode==3`, previously `g_socChargeDischargeMode!=3`), once on idle timer expiration (`g_socIdleTimeAccumMs` overflows after ~3,600,000 ms). In 117.7 at these two points only SOH/cycle count were updated, without timestamp logging.

### 2.4 New Error Timestamp Log on Emergency Reset

`System_Reset_And_Halt_WithCurrentCheck` (`0x08004998` in both versions, 117.7: 136 B, 118: 188 B):

```c
// 118, additionally before HAL_IWDG_Refresh():
if ((5 < (short)(g_bmsPackPowerWatts / 1000)) || ((short)(g_bmsPackPowerWatts / 1000) < -5)) {
  RTC_Read_DateTime((int32_t *)&g_rtcDateTimeBuf);
  I2C_EEPROM_Write_WithMutex((uint8_t *)&g_rtcDateTimeBuf, 0xfc02, 10);
}
```

If last measured pack power on reset was outside ±5 kW, an RTC timestamp is now written to same EEPROM fault history as in §2.3 before the actual
MCU reset (SCB→AIRCR). 117.7 lacks this condition and jumps directly into watchdog/reset sequence.

*Note on decompilation description:* The automatically generated function header comment in
`Dekompilate/VNSD_BMS_1177/08004998_System_Reset_And_Halt_WithCurrentCheck.c` already describes the
118 behavior ("… logs an RTC-timestamped error record if last seen current value outside range …"), although the 117.7 code **does not** contain this check. This is a naming/description artifact of the export/naming run, not a statement about actual
117.7 code — only the C code quoted above is authoritative.

### 2.5 RAM Layout Follow-on: New Global `g_dischargeStepRotateLimitCache`

118 adds the 2-B global `g_dischargeStepRotateLimitCache` at `0x20002a58` (between
`g_dischargeStepRotateTimer` and `g_socThresholdSelectFlag`). It fits exactly into an alignment gap
that already existed at this location in 117.7 (`g_packRecoveryChargeTimerB` ends in both
versions only at `0x20002a6c`) — the overall block doesn't grow **as a result**, but six neighboring
globals shift in 118 by +2 B:

| Global | 117.7 | 118 |
|---|---|---|
| `g_socThresholdSelectFlag` | `0x20002a58` | `0x20002a5a` |
| `g_mode1FullTimerA` | `0x20002a5a` | `0x20002a5c` |
| `g_mode1FullTimerB` | `0x20002a5c` | `0x20002a5e` |
| `g_mode2LowSocTimer` | `0x20002a5e` | `0x20002a60` |
| `g_mode3EmptyTimerA` | `0x20002a60` | `0x20002a62` |
| `g_mode3EmptyTimerB` | `0x20002a62` | `0x20002a64` |
| `g_packRecoveryChargeState` | `0x20002a64` | `0x20002a66` |
| `g_packRecoveryChargeTimerA` | `0x20002a66` | `0x20002a68` |
| `g_packRecoveryChargeTimerB` | `0x20002a68` | `0x20002a6a` |

This explains the five rows marked "pure follow-on …" in table §1
(`PackRecovery_LowCell_Charge_SM`, `PackSelect_Mode1_Full`, `PackSelect_Mode2_LowSoc`,
`PackSelect_Mode3_Empty`, `PackSelect_Next_AboveSocThreshold`): these functions are content-identical, but reference fields from this shifted block.

### 2.6 Config/Protocol Version Tag `'u'` (117) → `'v'` (118)

Six places raise the same version byte value in sync from `0x75` (ASCII `'u'`, 117 decimal) to
`0x76` (ASCII `'v'`, 118 decimal):

| Function | Role |
|---|---|
| `BMS_Config_Params_Factory_Reset` | Writes `g_cfgVersionTag` on factory reset. |
| `BMS_Config_Params_Load_Or_Default` | Compares `g_cfgVersionTag` on boot (guard) and writes it on reset. |
| `RS485_Dispatcher` | Version response byte in RS485 response buffer. |
| `RS485_Pack_Telemetry_Response` | Version byte in pack telemetry response (offset `0x48`). |

A byte mismatch on boot triggers factory reset of parameter block in `BMS_Config_Params_Load_Or_Default` — a device upgrading from 117.7 to 118 gets an automatic parameter reset to factory values (independent of individually-set RS485/CLI protection thresholds). Separately from this, `PerPack_Struct_Builder` (§0, beta note) reports firmware version value `g_pack0FwVersionTag` with its own, inconsistent count convention
(`0x499`=1177 in 117.7 vs. `0x76`=118 in 118) — two different version conventions for
two different purposes (config compatibility vs. reported FW version).

## 3. What Did NOT Change

- KA495xx AFE driver (SPI, CRC-8 `KA495XX_Crc8`, register table verification) — only addresses.
- Charge current/voltage limit tables (`Max_Charge_Current_Calc`, `Max_Discharge_Current_Calc`,
  temperature×SoC derating matrix, 50-A cap, 57.6 V/48.0 V) — see also
  `VNSD_Charge_Limits_Charge_Power.md`.
- CAN command set and message IDs (`CAN_RX_Command_Dispatcher`, `CAN_RxSlavePackTelemetry_Unpack`,
  `CAN_Pack_Activation_Config_Receive`) — only addresses/variable names.
- OTA/bootloader handshake (`BMS_OTA_Flash_Writer_FinalizeTail`, `OTA_Set_Reboot_Params`,
  `HAL_Flash_Write_Config_Block`, staging address `0x0801E000`, tag check `0x0802B800`).
- Shell command table (36 entries, user `BLUE`) — sole changed CLI function
  (`CLI_ShellInit`) differs only in local variable names.
- FreeRTOS task list: all 10 tasks with identical stacks/priorities/handles
  (`RTOS_CreateAllTasks` verified byte-for-byte, only 10 entry addresses shifted).
- Protection thresholds (`Cell_OVP_UVP_TempCompensated_Protection`, `Voltage_Temp_Protection_Checker`)
  — full C comparison (not just listing diff) shows only the one globally shifted RAM address
  (`0x20004AD0`→`0x20004ADC`, see §4), no changed threshold values.

## 4. Open Points

- **No writer for `g_socOcvRecalibrateRequest` found.** Neither in 117.7 nor 118 does any
  of the 710/711 decompiled functions set this flag — it is in 118 only read (§2.3). Possible:
  external trigger (debug/JTAG memory write) or function outside BMS image
  (e.g., via RS485/CAN from Control/Micro, but not visible in any of the 88 changed or 622 identical
  functions). Not fully determined.
- **RAM growth only partially explained.** `.bss` grows 16 B (117.7: `0x9fcc` → 118: `0x9fdc`),
  FreeRTOS heap shrinks 8 B (`0x4FF8`→`0x4FF0`). The new 2-B global
  `g_dischargeStepRotateLimitCache` (§2.5) explains only part of this shift; remaining bytes not
  attributable to single new named global — likely heap_4 internal alignment effects (the heap block directly before `ucHeap` also shifts, see full analysis document §2). For this diff's question (real vs. cosmetic function changes) without further relevance.
- **Global constant `0x20004AD0`→`0x20004ADC` (14 functions affected)**: same RAM address
  shifts in 14 independent, otherwise unchanged functions by exactly 0xC (12) bytes
  (`BMS_Measurements_Valid_Check`, `BMS_ProtectionAndCommTick`, `CAN_TX_PF2_Capacity`,
  `Cell_OVP_UVP_TempCompensated_Protection`, `OwnPack_MOSFET_Driver` + `_MasterVariant`,
  `PerPack_Struct_Builder`, `RS485_Pack_Telemetry_Response`, `SOC_AlgorithmOrchestrator`,
  `SOC_InitCapacityParams`, `SOC_ResetEstimatorToPercent`, `SOC_UpdateDisplayValueAndFullFlag`,
  `Voltage_Temp_Protection_Checker`). Suggests a single 12-B-grown data object before this address; which exactly was not identified in this session (no function name in the 88 changes points directly to it — likely compiler-placed, unnamed `.bss` object). Without impact on real/cosmetic classification of these 14 functions, since their code behavior is identical.
