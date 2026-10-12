# BMS-FW V115 (Venus A) — Full analysis

> 🇩🇪 Deutsche Version: [VNSA_BMS_FW_v115_Vollanalyse_de.md](VNSA_BMS_FW_v115_Vollanalyse_de.md)

**Status:** 2026-10-11.

**Predecessors/reference:** Venus A BMS 110 (`VNSA_BMS_FW_v110_Full_Analysis.md`), Venus A BMS 110.5
(`VNSA_BMS_FW_v1105_Full_Analysis.md`), Venus D BMS 118 (`VNSD_BMS_FW_v118_Full_Analysis.md`) · **Diff table:**
[VNSA_BMS_Diff_v110_v115.md](VNSA_BMS_Diff_v110_v115.md) (with a 110.5 column) · **Decompilation/Listing:**
`Dekompilate/VNSA_BMS_115/` (local only, not in the repo) (727 functions, `listing/` with flash disassembly +
`globals_and_registers.txt`) · **Companion:** `Control_FW/VNSA_Control_FW_v151_Full_Analysis.md`.

## 0. Summary

- Image base **0x08002800**, image size 0x1A800 bytes (108,544 B), MCU **Nations N32G45x** (Cortex-M4F), Keil ARMCC,
  FreeRTOS, letter-shell (user `BLUE`) — same platform as A 110/110.5. Version string `Version: 115.0`, build date
  `Sep  1 2026 11:14:53`. The file-name stem `VEPRO_BMS` is only a name: the code is the Venus A BMS code base.
- **Code family (checked, not assumed):** pointer/BL-masked segment coverage of the 115 image is 82.4 % against A 110.5,
  81.4 % against A 110, 65.7 % against D 118 and only 12.6 % against E3 BMS 115; the string set equals A 110/110.5
  except the build/version strings. 115 is therefore **not** an E3- or D-like branch.
- **115 merges both A branches:** the A 110.5 beta logic (PV-rescue CAN `0x03410110`, 120 s STOP in cell UVP level 2,
  10.0 A charge clamp below 35.00 V, ±70 A comparator window, 2800 mV empty and UVP-2A thresholds, low-voltage balancer
  with min-cell < 2800 mV, SELF-flag gate only above 39.00 V) is back, while parts of A 110/109 remain (no
  `g_chargeStatusCode` 2/3/4, fixed charge-voltage limit). The **system current caps are 50.0 A again** (110: 40.0 A) and
  the charge-voltage limit is a fixed **48.0 V** (CAN clamp 46.8 V unchanged).
- **Changelog items found in the code** (remark of the OTA record, translated): (1) pack switch when the inter-pack SOC
  difference exceeds 20 → `PackSelect_SocImbalanceSwitch`; (2) dynamic correction of the charge/discharge targets →
  `PackSelect_ChargeStepTarget` (95/100 % rules) and threshold re-computation in the step rotations; (3) end-of-charge/
  discharge correction needs three cells at the threshold → `APP_Get_Pack_Summary_Data` cell counters used by
  `Charge_SOC_Integration`/`Discharge_SOC_Integration`; (4) improved charge/discharge MOS recovery →
  `OwnPack_MosRecovery_SM` (new output PE7, CAN `0x1805AA01`).
- **Not in the changelog, but new:** MCU RTC wake-up timer before STOP mode (LSI, nominal 6 h) with automatic return to
  STOP after 50 protection ticks; pack under-voltage STOP below 36.00 V; protection counters cleared during an AFE
  communication fault; two new inbound CAN ids (`0x01210103`, `0x04410110`); SOC handoff state machines bound to an
  EMS mode byte; configuration block grown to 0xD8 bytes with version tag `'s'`; SOC calibration log reduced to 5 slots.
- **Pack capacity:** `CAN_TX_PerPack_10Msgs` sends packs 1..6 (`pack_id != 0 && pack_id < 7`), **13 cells** per pack —
  unchanged.
- **All 727 functions named and typed**, 0 `FUN_*`, all verification gates passed (§2.3). Tags: `NAME_VERIFIED` 727,
  `NAME_LOW_CONFIDENCE` 51.

## 1. Binary Fingerprint

| File | Version | Size | SHA-256 |
|---|---|---|---|
| `115_bms_VNSA-0_VEPRO_BMS_V115_APP_20260901_ota_174540.bin` | 115.0 | 108,544 B (0x1A800) | `d44833daebebf49350deeffbb725b15b80babd33f35d69e94a004005c8cc8758` |

Manufacturer changelog: "VNSA115 Official Edition"; OTA remark (translated): "Official version; 1. added pack switching
when the SOC difference between packs is greater than 20, to avoid a too large SOC difference; 2. added dynamic
correction of the charge/discharge target values, to avoid old values influencing the pack switch in special cases;
3. reduced the sensitivity of the end-of-charge/discharge correction, three cell voltages must now reach the threshold;
4. optimised the charge/discharge MOS recovery mechanism to avoid extreme situations". OTA record: edition 4563,
CRC 30760, not forced. Image version tag at `0x08010000`: **`BMS2`** (unchanged).

The file-name stem `VEPRO_BMS` (A 110: `VABMS`, A 110.5: `VA50A`) suggests a planned "Venus E Pro" (app type codes
`VEPRO`/`VEPRO2`) that reuses the Venus A BMS code; this is an assumption from the name, not confirmed by the firmware
(details: [VNSA_BMS_FW_v115_Open_Points.md](VNSA_BMS_FW_v115_Open_Points.md) §0).

## 2. MCU, Memory Map, Processing Status

### 2.1 MCU

Nations N32G45x as in 110/110.5/D (102-entry vector table, UID at `0x1FFFF7F0`, `SystemInit` `0x08011768`). New
peripherals in use: **RTC** (`0x40002800`: WRP, CTRL, INITSTS, WKUPT), **backup registers** (`0x40006C00`, BKP_DAT1/2),
PWR DBKP and RCC LSI/RTC bits via bit-band aliases; the RTC interrupt (vector IRQ3 = `RTC_IRQn`, A 110: default handler)
is now used.

**Image base proof:** vector table word 0 = SP `0x2000C9A8`, word 1 = Reset `0x08002A6D`. File offset 0x26C holds the
Keil `Reset_Handler` (`ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`) whose literals `0x08011769` (SystemInit) and
`0x08002999` (`__main`) point into the image only at base 0x08002800; the Keil region table at `0x0801CA30` holds
load/execution addresses consistent with that base.

### 2.2 Memory Map

| Range | Contents |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (not part of the image); boot/OTA page `g_bootConfigPage` at `0x08002000` |
| `0x08002800`–`0x0801CFFF` | Application image; vector table at `0x08002800` |
| `0x08010000` | Image version tag `g_imageVersionTag` = `'BMS2'` |
| `0x0801BB6C` | `g_crc8Poly07Table` (CRC-8, polynomial 0x07, SH36735 frames) |
| `0x0801CA30` | Keil region table (2 regions), `0x0801CA50` `shellCommandTable` (52 × 16 B), `0x0801CD90` `.data` load image |
| `0x0801E000` | OTA staging copy `g_otaStagingImage` (tag word `0x0802B800` must be `'BMS2'`) |
| `0x20002800`–`0x20002BDF` | `.data` (0x3E0 B; 110: 0x398 B, 110.5: 0x3A0 B) |
| `0x20002BE0`–`0x2000C9A7` | `.bss` (0x9DC8 B); FreeRTOS `ucHeap` at `0x20004718` (already 8-byte aligned, no pad) |
| `0x2000C9A8` | `__initial_sp` (110: `0x2000C958`) |
| I2C EEPROM | configuration block at 600 (**0xD8 B, tag `'s'`**), SOC calibration log **0x440, 5 × 20 B**, master marker `0x01F6`, counter `0x0254`, event log `0x2000`/`0x3000`, error lock `0xFC00`, full-charge marker `0xFC02` |

RAM names were ported by aligning the literal-pool references of paired code against 110 (and checked against 110.5);
the layout differs from 110 by piecewise shifts of 1–0x50 bytes (new variables, removed relay-comm substate/timer).
A plain address port would have been wrong in several places (e.g. `g_pack0SlotId` is `0x200037DE`, the twelve
high-temperature counters are shifted by 0xC).

### 2.3 Processing Status in Ghidra

- Program `/VNSA/BMS/BMS_115_VNSA-0_VEPRO_BMS_V115_APP_20260901_ota_174540.bin`: **727 functions, all named**, every
  function has a user-defined prototype and plate comment, 0 `FUN_*`, 0 duplicates.
- Port from 110: segment matching (374 segments, 98 after merging equal-size gaps), 661 function pairs by address
  (636 masked-identical incl. locals) + 30 by instruction similarity (monotone order) + 6 moved/re-created by hand
  (`CAN_TX_PF2_Capacity`, `PackHandoff_Tick`, `PackSelect_ChargeStepTarget`, `APP_Get_Pack_Summary_Data`,
  `SoftFloat_Round_To_Even`, `SoftFloat_Pack_Float_Result`). 30 new functions analysed and named from the code (RTC/BKP
  drivers named after the N32 standard library; `SoftFloat_Int_To_Double`/`_Double_To_Float` and
  `HAL_RCC_Get_Flag_Status` byte-identical to E3 115 resp. D 118 and named like there).
- Structural repairs: literal pools behind non-returning/tail calls, three functions that had swallowed a following new
  function split, fake function inside the shell user string removed, strings and tail tables re-typed, two spurious
  analyser references deleted. `SOC_Select_Threshold_2420_2750` renamed back to **`SOC_Select_Threshold_2420_2800`**
  (code = 110.5).
- Gates (`vna115_bms_gates.py`, same checks as for 110/110.5): 0 undefined bytes, 0 orphan instructions, 0 flows into
  data, 0 function pointers to non-code, 0 literal-only functions, 0 literal pools decoded as code, 0 calls into function
  bodies, 0 unnamed referenced RAM globals, 0 default prototypes, task names = `xTaskCreate` name strings, no-return
  flags sane, 0 duplicate labels; one 1-character string flagged = empty shell password (as in 110/110.5).
  Decompilation export: 727 functions, **0 failures**.

## 3. FreeRTOS Tasks

`RTOS_CreateAllTasks` (`0x0800F9FC`) creates the same 10 tasks as 110 (names, stacks, priorities unchanged):

| Task | Entry | Stack (words) | Prio |
|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800FC88` | 0x80 | 5 |
| `RTOS_vTaskKeyScan` | `0x0800FC68` | 0x80 | 9 |
| `RTOS_vTaskKA495XX` (SH36735 worker) | `0x0800FC50` | 0x80 | 8 |
| `RTOS_vTaskCanRec` | `0x0800FBF0` | 0x180 | 4 |
| `RTOS_vTaskCanSend` | `0x0800FC00` | 0x80 | 3 |
| `RTOS_vTaskRS485Comm` | `0x0800FCC2` | 0x180 | 1 |
| `RTOS_vTaskSocAlgorithm` | `0x0800FCD0` | 0x100 | 7 |
| `RTOS_vTaskProtectCheck` | `0x0800FCAC` | 0x100 | 6 |
| `RTOS_vTaskControlLogic` | `0x0800FC0A` | 0x80 | 2 |
| `RTOS_vTaskDataSave` | `0x0800FC24` | 0x80 | 0 |

The IWDG is still only fed, never configured or started by the application.

## 4. Interfaces

### 4.1 CAN

Frame set and layouts as in 110.5 (`VNSA_BMS_FW_v1105_Full_Analysis.md` §4.1), with these differences:

**Outbound**

| ID | Function | A 115 |
|---|---|---|
| `0x1802AA01` | `CAN_TX_PF2_Capacity` `0x080084EC` | byte 7 = **pack-select acknowledge** (`g_packSelectAckReport`: 2 once the PB13 acknowledge was seen, cleared after sending); 110/110.5: relay-comm substate 1/2/3 |
| `0x1803AA01` | `CAN_TX_PF3_ChargeDischargeLimits` `0x080085D4` | charge voltage clamp 46.8 V; charge current = system limit **≤ 50.0 A**, **10.0 A while the own pack voltage is below 35.00 V** (as 110.5), 25.0 A on pack fault; discharge current ≤ 50.0 A |
| `0x1804AA01` | `CAN_TX_PF4_ProtectWarnings` `0x0800876C` | byte 4 = **0** (110: `g_chargeStatusCode`, 110.5: 2/3/4) |
| `0x1805AA01` | `APP_IdleKeepaliveBroadcast_Tick` `0x0800F144` | **new**, DLC 1: `g_mosRecoveryReport` (0 idle, 1 MOS recovery running, 2 finished); sent from the idle tick once 50 ms have elapsed (on non-master packs on every call after that, because the timer is only reset on the master path) |
| `0x1820..0x1841AA00+n` | `CAN_TX_PerPack_10Msgs` `0x08008DE8` | packs 1..6 (`pack_id != 0 && pack_id < 7`); 0x1821 charge current **10.0 A for every pack while the own pack voltage is below 35.00 V** (as 110.5); 0x1823 FW tag of the own pack **0x73 = 115** |
| `0x1851/0x1852AA00+n` | `CAN_TX_PerPack_SocDiag` | unchanged |

**Inbound** (`CAN_RX_Handler` `0x08008BD4`)

| ID | A 115 |
|---|---|
| `0x03410110` (inverter PV rescue) | `g_pvRescueRequestFlag` = (byte 1 > 8) **or (byte 6 == 1)**; 110.5: byte 1 > 8 only; 110: ignored |
| `0x1801FFAA` (master broadcast) | as before, additionally **byte 3 → `g_emsHeartbeatModeByte`**; value 2 enables the SOC handoff state machines and the relay-comm inhibit (§6) |
| `0x01210103` | **new:** `g_can01210103Flag` = high nibble of byte 0 != 0 (keeps the relay-comm inhibit set); sender not identified |
| `0x04410110` | **new:** `g_can04410110Ratio` = int16(bytes 4–5) × 10000.0 / uint16(bytes 6–7) as float; no reader in this image; sender not identified |

System command opcodes (`0x100101AA`, incl. 0x0A–0x0C), `0x100501AA` heartbeat reply, the staggered slave status
report and the `0xA5A5` echo test are unchanged (code as 110.5).

The A Control 151 fills only bytes 0–2 of `0x1801FFAA` (`Grid_BuildStatusFlagsReg`); byte 3 stays 0, so with Control 151
the mode-2 paths of §6 are not activated (from the code; not observed on the bus).

### 4.2 RS485

Unchanged apart from the version byte **`'s'` (0x73)** in opcode 5 and in `RS485_Pack_Telemetry_Response` (110: `'n'`).
Code otherwise identical to 110.5.

### 4.3 Debug Shell (letter-shell)

Unchanged: user `BLUE`, 52 table entries, 39 commands. The login banner prints `Version: 115.0` and the build date.

### 4.4 EEPROM layout (I2C)

| Address | Change in 115 |
|---|---|
| 600 (0x258) configuration block | **0xD8 bytes** (110: 0xD4), version tag **`'s'`** (110: `'n'`), new field `g_cfgPackUvShutdownTimeout` at offset 0xD4 (default 120). The checksum (offset 0xD0) is computed over offsets 0..0xD3, i.e. **over its own previous value**, and does not cover the new field (§13). |
| `0x0440`… SOC calibration log | **5 slots × 20 B at 0x440** (110: 50 slots at 0x400). The format marker at `0x024E` (`'f'`) is not reset, so an EEPROM formatted by an older version is not reformatted; the new slots start at old slot 3 + 4 bytes. |
| `0x01F6`, `0x0254`, `0x2000`/`0x3000`, `0xFC00`, `0xFC02` | unchanged |

## 5. Cell Monitoring — SH36735

Unchanged: SPI1, CRC-8 polynomial 0x07, **13 cells** (`INIT_SH36735_Subsystem` `0x0800E530`, only its version
constant changed to 0x73), balancing rules, zero-offset calibration ("SELF" flag). New: during an AFE communication
fault (`g_protState4` bit 4) the cell, current and temperature protection counters are cleared and not evaluated.

## 6. Pack Management

- **SOC-imbalance switch** (`PackSelect_SocImbalanceSwitch` `0x08007B98`, called in pack-select mode 0): if highest
  minus lowest SOC of the online packs is **≥ 20 %**, a switch is requested after a power-dependent time
  (|I| × V of the active pack > 1500 W: 1200 ticks, > 1000 W: 1800, > 500 W: 2400, else 3000): while charging
  (> 1.0 A) to the lowest-SOC pack, while discharging (< −1.0 A) to the highest-SOC pack.
- **Dynamic targets:** `PackSelect_ChargeStepTarget` returns 100 % if all online packs are ≥ 95 %, 95 % if all are ≥ 90 %
  and the pack is below 95 %, otherwise the old 10 % step. `PackSelect_Charge_StepRotate` / `_Discharge_StepRotate`
  recompute a stored threshold that is more than 10 % away from the active pack's SOC.
- **SOC handoff state machines** (`PackHandoff_Charge_SocSwitch_SM` `0x08006B5C`, `PackHandoff_Discharge_SocSwitch_SM`
  `0x08006D0C`): used by `PackHandoff_Tick` instead of `PackHandoff_StateMachine` when `g_emsHeartbeatModeByte == 2`
  and the PV-rescue flag is set; they switch the charge and discharge enables of old and new pack in an order that
  depends on the current direction (charging: old discharge off → new charge on → old charge off → new discharge on;
  discharging: old charge off → new discharge on → old discharge off → new charge on), each step waiting for the MOS
  status, and skip the charge-MOS wait when the new pack is > 98 %.
- Enable-mask hygiene: `PackSelect_ChargeEnable_Maintain`, `_MosMismatch_Fallback` and `_Next_AboveSocThreshold` reset
  the charge/discharge enable masks to the active pack only before re-electing; the active pack is re-added to the
  discharge-enable mask when no protection blocks it and its SOC is above 100 % − discharge limit.
- Relay-comm state machine: the substate (1/2/3) and its 300-tick command timer are gone; the new
  `g_relayCommEscalationInhibit` (mode-2 PV rescue, CAN `0x01210103` flag or |system current| ≥ 1.1 A) suppresses the
  300/600-tick escalations to fault modes 3/2 and the related LED pattern.
- **MOS recovery** (`OwnPack_MosRecovery_SM` `0x08005BA4`, ticked every 100 ms by `BMS_Role_Dispatcher`): when the master
  variant of the MOSFET driver finds the charge-enable output PC5 low although it should be on, it drives the new output
  **PE7** (switch 7) until the own pack voltage is above 8.49 V, waits 20 ticks, then switches PE7 off and PC5 on;
  progress is broadcast in `0x1805AA01`. Retry/fail states (with `g_protState1` bit 2) exist, but no path sets them in
  this image. The board function of PE7 is not identifiable from the firmware.
- `Pack_LowVoltage_Balancer` and the SELF-flag gate as 110.5 (min cell < 2800 mV criterion; relays forced off without
  SELF flag only above 39.00 V).

## 7. SOC/SOH

Tables unchanged (byte-identical). Changes: empty threshold **2800 mV** at ≥ 0 °C (`SOC_Select_Threshold_2420_2800`
`0x08014624`, as 110.5; 110: 2750 mV); the end-of-charge interpolation in `Charge_SOC_Integration` starts once
**3 cells ≥ 3500 mV** (110: max cell ≥ 3500 mV), the end-of-discharge compensation in `Discharge_SOC_Integration` once
**3 cells ≤ 2900 mV** (110: min cell ≤ 2900 mV); the counters come from `APP_Get_Pack_Summary_Data` (bytes 14/15).
The SOC calibration log has 5 slots (§4.4).

## 8. Charge/Discharge Limits

| Item | Venus A 115 | Venus A 110 | Venus A 110.5 |
|---|---|---|---|
| System charge current cap (`g_sysChargeCurrentLimit_x10`, Modbus 32106) | **50.0 A** | 40.0 A | 50.0 A |
| System charge current outside master run state 2 | 50.0 A, 0 while `g_socStateFlags` bit5 is set | 0 | 50.0 A (0 with status code 4 while bit5 is set) |
| System discharge current cap (Modbus 32107) | **50.0 A** | 40.0 A | 50.0 A |
| Charge voltage limit (`g_sysChargeVoltLimit_x10`, Modbus 35110) | **48.0 V fixed**, CAN clamp 46.8 V | 46.8 V fixed | pack voltage + 2.0 V, max 48.0 V, CAN clamp 46.8 V |
| CAN charge current below 35.00 V own-pack voltage (0x1803 and 0x1821) | **10.0 A** | no clamp | 10.0 A |
| Per-pack max charge current (`Max_Charge_Current_Calc` `0x08014A08`) | table, max 50.0 A; never-fully-charged derating | same | same |
| Discharge voltage limit | 39.0 V | 39.0 V | 39.0 V |

With the Control the system limits become power: 50.0 A at the nominal 13S voltage (41.6 V) is ~2080 W, above the A
micro limit, so the battery limit normally does not cap the AC power (inference, not measured).

## 9. Protection Functions

Configuration defaults unchanged (new field: `g_cfgPackUvShutdownTimeout` = 120). Code:

- `Cell_OVP_UVP_TempCompensated_Protection` (`0x0800DBB0`): as 110.5 (UVP level 2 stage A 2800 mV and stage B 2500 mV
  both latch Protect1 bit5 and reset the SOC to 0 %; sensor-fault trip only above 39.00 V); **new:** all counters cleared
  while the AFE comm fault is set. Same reset block added to `Current_Protection_Checker` and
  `Voltage_Temp_Protection_Checker`.
- `BMS_ProtectionAndCommTick` (`0x0800A18C`): as 110.5 — in cell UVP level 2, **120 s without `g_pvRescueRequestFlag` →
  STOP** (`System_Reset_And_Halt_WithCurrentCheck`); own MOSFETs disabled during UVP2; `g_standaloneModeFlag` written.
- **New `Pack_UnderVoltageShutdown_Check`** (`0x0800D590`): own pack voltage **< 36.00 V** for
  `g_cfgPackUvShutdownTimeout × 10` ticks (default 1200) → `g_protState4` bit5 and, unless the PV-rescue flag is set,
  STOP; release above 37.00 V. Not mapped into the Protect/Warning words (internal; logged as event 1 via the snapshot).
- `HW_OvercurrentAndSensorProtection` (`0x0800BC70`): comparator window **±70.0 A** (as 110.5); **new:** NTC-count and
  MOS-temperature range checks skipped below 20.00 V pack voltage.
- **New output PD8** (`Pack_LowVoltageOutput_PD8_Update`): high while the own pack voltage is below 20.00 V.

## 10. STOP Mode and RTC Wake-up (new)

`System_Reset_And_Halt_WithCurrentCheck` (`0x08004B1C`) now calls `RTC_ArmWakeupTimer_Before_Stop` (`0x0800FD1C`)
before entering STOP:

1. PWR clock and backup-domain write access on, LSI on (wait for LSIRD), RTC clock source = LSI, RTC clock on, wait for
   synchronisation.
2. Only if `BKP_DAT1 != 0x5460`: wake-up timer off, `WKUPSEL = 4` (ck_spre), `WKUPT = 0x5460` (21600), marker
   `BKP_DAT1 = 0x5460`. 21601 periods = **6.0 h at a nominal 1 Hz**; the RTC prescaler is not written (reset value), so
   with the uncalibrated LSI (~40 kHz typical) the real period depends on the LSI frequency (about 4.9 h at exactly
   40 kHz with the default /128/256 prescaler) — derived from the code and the reference manual, not measured.
3. EXTI line 20 rising edge + NVIC IRQ3 (`INIT_RtcWakeupExti_And_NVIC`), wake-up interrupt enabled, backup domain locked,
   wake-up timer enabled; then STOP, then system reset.

On wake-up `RTC_IRQHandler` (`0x0800F974`) clears the flags, writes **`BKP_DAT2 = 1`** and stops the RTC clock. At boot
`BKP_ReadAndClearRtcWakeFlag` copies BKP_DAT2 to `g_rtcWakeupFlag` and clears it; `GPIO_Init_All_Pins` then re-initialises
PC5/PB0/PB2/PB5, and `RTC_WakeCycle_ReturnToStopTick` sends the BMS **back to STOP after 50 protection ticks** (the flag is
only cleared by the next boot). A BMS that was put to sleep (UVP2 without PV rescue, pack < 36.00 V, 600 s offline,
system command 2) therefore wakes up periodically for a short time instead of sleeping until an external wake-up. Which
external events end the cycle (e.g. charger present, CAN activity) is not visible in this path; the wake flag itself is
never cleared during the run (open point).

## 11. OTA/Update

Unchanged: CAN `BMS_OTA_Flash_Writer` (`0x0800E0EC`, opcode 0xCE) and `RS485_OTA_Flash_Handler` (`0x0800D870`), staging
copy `0x0801E000`, CRC16, tag `'BMS2'`. A 115 image is accepted by an A BMS running 109/110/110.5 and vice versa (same tag);
D images (`'BMS1'`) are rejected. **After the update the configuration block is reloaded with defaults** because the
version tag changes (`'n'` → `'s'`) and the block length changes (0xD4 → 0xD8).

## 12. Security Notes

Unchanged against 110.5 (shell on the RS485 debug path, unauthenticated CAN commands, IWDG not started). New
unauthenticated CAN inputs: `0x01210103` (relay-comm inhibit), `0x04410110` (write-only value) and the extended
`0x03410110` (keeps the BMS awake in UVP2 and in the new pack-undervoltage path).

## 13. Open Points

In-depth treatment of points 1–3 and 6 (arithmetic, senders/receivers, pins, proof levels):
[VNSA_BMS_FW_v115_Open_Points.md](VNSA_BMS_FW_v115_Open_Points.md).

1. **Configuration checksum:** `APP_Config_Checksum_And_Save` (`0x0800B7E4`) and `BMS_Config_Params_Load_Or_Default`
   (`0x0800A840`) compute the one's-complement byte sum over offsets 0..0xD3, which includes the checksum word at 0xD0
   itself. A stored block only verifies when the byte sum of the new and the previous checksum word happens to be equal;
   otherwise the defaults are loaded (and saved) at the next boot, so changed parameters would not persist. Derived
   from the code, not tested on a device.
2. Senders and meaning of `0x01210103` and `0x04410110`; meaning of byte 6 of `0x03410110`; who sends
   `g_emsHeartbeatModeByte == 2` (Control 151 sends 0).
3. Board function of the new outputs PE7 (MOS recovery) and PD8 (< 20.00 V) and their pin-mode setup (only struct-based
   GPIO inits remain unresolved); meaning of the 8.49 V condition in the recovery.
4. Real RTC wake-up period (LSI) and the field behaviour of the periodic wake/STOP cycle; what ends it.
5. SOC calibration after an update: the old log (50 slots at 0x400) is read with the new 5-slot layout at 0x440; whether
   a valid record is found at first boot is not tested.
6. Receiver of `0x1805AA01` in the A Control/Micro: no handler found in Control 151 (search for the id in the
   decompilation export).
