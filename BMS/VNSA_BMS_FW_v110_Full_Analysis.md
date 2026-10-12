# BMS-FW V110 (Venus A) — Full analysis

> 🇩🇪 Deutsche Version: [VNSA_BMS_FW_v110_Vollanalyse_de.md](VNSA_BMS_FW_v110_Vollanalyse_de.md)

**Status:** 2026-10-11.

**Predecessor/reference:** Venus A BMS 110.5 (`VNSA_BMS_FW_v1105_Full_Analysis.md`), Venus D BMS 118
(`VNSD_BMS_FW_v118_Full_Analysis.md`) · **Diff table:** [VNSA_BMS_Diff_v1105_v110.md](VNSA_BMS_Diff_v1105_v110.md) ·
**Decompilation/Listing:** `Dekompilate/VNSA_BMS_110/` (local only, not in the repo) (697 functions, `listing/` with flash
disassembly + `globals_and_registers.txt`) · **Companion:** `Control_FW/VNSA_Control_FW_v151_Full_Analysis.md`.

## 0. Summary

- Image base **0x08002800**, image size 0x19800 bytes (104,448 B), MCU **Nations N32G45x** (Cortex-M4F), Keil ARMCC,
  FreeRTOS, letter-shell (user `BLUE`) — same platform as A 110.5. Version string `Version: 110.0`, build date
  `May 20 2026 13:59:48`.
- **Code base: A 109, not A 110.5.** The image is byte-identical to A BMS 109 (build Dec 26 2025) apart from the
  system current cap, the version constants and the build/version string (334 differing bytes). The changes of the
  1105 beta ("110.5", build May 11 2026) are **not** contained. By build date and OTA edition (3430 vs 3316) 110 is the
  newer release; by content it is a parallel branch (109 + 40 A). Details: [diff table](VNSA_BMS_Diff_v1105_v110.md).
- **Changelog "maximum charge-discharge current changed to 40 A":** `Master_Build_ChargeCurrentLimit` and
  `Master_Build_DischargeCurrentLimit` cap the **system** limits at **400 (40.0 A)** instead of 500. These are the values
  sent in CAN `0x1803AA01` and served by the Control as Modbus 32106/32107 (aliases 35111/35112). The per-pack limits
  (charge/discharge derating tables, 50.0 A table maximum) are unchanged.
- **Behaviour against 1105 (what an update 1105 → 110 changes):** no PV-rescue handshake (CAN `0x03410110` ignored, no
  120 s sleep in cell UVP level 2; instead the Venus D logic: own MOSFETs only below 2.0 A, standalone pack fault
  mode 3 after 200 s), no 10.0 A charge clamp below 35.00 V, charge voltage limit fixed 46.8 V, UVP level-2 stage A
  3000 mV as warning only, overcurrent comparator cross-check ±3.0 A, empty threshold 2750 mV, BMS version reported as
  **1100**.
- **Pack capacity:** `CAN_TX_PerPack_10Msgs` sends packs 1..6 (`pack_id != 0 && pack_id < 7`), **13 cells** per pack.
- **All 697 functions named and typed**, 0 `FUN_*`, all verification gates passed (§2.3). Tags: `NAME_VERIFIED` 697,
  `NAME_LOW_CONFIDENCE` 49.

## 1. Binary Fingerprint

| File | Version | Size | SHA-256 |
|---|---|---|---|
| `110_bms_VNSA-0_VABMS_APP_110_ota_163748.bin` | 110.0 | 104,448 B (0x19800) | `f4ee89c8837f1178efe085aae107f733483055bc2d91e57d510e354167283559` |

Manufacturer changelog (translated): "Maximum charge-discharge current changed to 40 A." OTA record: edition 3430,
CRC 13436, not forced. Image version tag at `0x08010000`: **`BMS2`** (unchanged). For comparison: A 109
(`109_bms_VNSA-0_20251226103854b52c66481.bin`) changelog "max. requested charge/discharge current changed to 50 A".

## 2. MCU, Memory Map, Processing Status

### 2.1 MCU

Nations N32G45x as in 1105/D (102-entry vector table, UID at `0x1FFFF7F0`, `SystemInit` `0x080105D4` sets
`SCB_VTOR = 0x08002800`). Register and IRQ names from the N32G457 SVD (ported from 1105).

**Image base proof:** vector table word 0 = SP `0x2000C958`, word 1 = Reset `0x08002A6D`. File offset 0x26C holds the
Keil `Reset_Handler` (`ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`) whose literals `0x080107D5` (SystemInit) and
`0x08002999` (`__main`) point into the image only at base 0x08002800; the Keil region table at `0x0801B860` holds
load/execution addresses consistent with that base.

### 2.2 Memory Map

| Range | Contents |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (not part of the image) |
| `0x08002800`–`0x0801BFFF` | Application image; vector table at `0x08002800` |
| `0x08010000` | Image version tag `g_imageVersionTag` = `'BMS2'` (fixed address; `CAN_TX_BMS_Status_Reporter` follows) |
| `0x0801A99C` | `g_crc8Poly07Table` (CRC-8, polynomial 0x07, SH36735 frames) |
| `0x0801B860` | Keil region table (2 regions), `0x0801B880` `shellCommandTable` (52 × 16 B), `0x0801BBC0` `.data` load image |
| `0x0801E000` | OTA staging copy `g_otaStagingImage` (tag word `0x0802B800` must be `'BMS2'`) |
| `0x20002800`–`0x20002B97` | `.data` (0x398 B; 1105: 0x3A0 B) |
| `0x20002B98`–`0x2000C957` | `.bss` (0x9DC0 B); FreeRTOS `ucHeap` at `0x200046D0`, Keil heap `0x2000995C`–`0x2000A957` |
| `0x2000C958` | `__initial_sp` (stack top; 1105: `0x2000C960`) |

The RAM layout equals 1105 minus the two 1105-only variables `g_uvpLvl2ShutdownTimerS` and `g_pvRescueRequestFlag`
(the addresses behind them are shifted by up to 8 bytes, e.g. `ucHeap` 0x200046D8 → 0x200046D0). RAM names were ported by aligning the literal-pool
references of paired functions (693 names) plus interpolation (2).

### 2.3 Processing Status in Ghidra

- Program `/VNSA/BMS/BMS_110_VNSA-0_VABMS_APP_110_ota_163748.bin`: **697 functions, all named**, every function has a
  user-defined prototype, plate comments, 0 `FUN_*`, 0 duplicates.
- Port from 1105: offline segment matching (pointer/BL masked, 350 segments, 92.6 % of the bytes; extended to 29
  equal-size regions), function layout and markup copied, 697 function pairs (680 masked-identical incl. locals),
  7 small functions moved by the linker (e.g. `SysTick_Handler`, `HAL_NoOpStub`, `CAN_EchoTest_Poll`), 4 pairs set by
  hand (`OwnPack_MOSFET_Driver_MasterVariant`, `Pack_LowVoltage_Balancer`, `CAN_RX_Handler`,
  `Master_Build_ChargeCurrentLimit`). Peripheral/system blocks copied by address. Flow and fall-through overrides
  (17) and float/double pool types ported; ARM constant references re-created per function.
- Changed functions reviewed against 1105 and D 118; plates rewritten; `SOC_Select_Threshold_2420_2800` renamed
  `SOC_Select_Threshold_2420_2750` (code = D), RAM `0x20002858` named `g_protCommTickCounter200` (D name and use; 1105
  reuses the address as `g_standaloneModeFlag`).
- Gates (`vna110_bms_gates.py`, same checks as for 1105): 0 undefined bytes, 0 orphan instructions, 0 flows into data,
  0 function pointers to non-code, 0 literal-only functions, 0 literal pools decoded as code, 0 calls into function
  bodies, 0 unnamed referenced RAM globals, 0 default prototypes, task names = `xTaskCreate` name strings, no-return
  flags sane, 0 duplicate labels. Decompilation export: 697 functions, **0 failures**.

## 3. FreeRTOS Tasks

`RTOS_CreateAllTasks` (`0x0800E8FC`) creates the same 10 tasks as 1105/D (names, stacks, priorities unchanged):

| Task | Entry | Stack (words) | Prio |
|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800EB88` | 0x80 | 5 |
| `RTOS_vTaskKeyScan` | `0x0800EB68` | 0x80 | 9 |
| `RTOS_vTaskKA495XX` (SH36735 worker) | `0x0800EB50` | 0x80 | 8 |
| `RTOS_vTaskCanRec` | `0x0800EAF0` | 0x180 | 4 |
| `RTOS_vTaskCanSend` | `0x0800EB00` | 0x80 | 3 |
| `RTOS_vTaskRS485Comm` | `0x0800EBC2` | 0x180 | 1 |
| `RTOS_vTaskSocAlgorithm` | `0x0800EBD0` | 0x100 | 7 |
| `RTOS_vTaskProtectCheck` | `0x0800EBAC` | 0x100 | 6 |
| `RTOS_vTaskControlLogic` | `0x0800EB0A` | 0x80 | 2 |
| `RTOS_vTaskDataSave` | `0x0800EB24` | 0x80 | 0 |

As in 1105 the IWDG is only fed, never configured or started by the application.

## 4. Interfaces

### 4.1 CAN

Frame set, IDs and layouts as in 1105 (see `VNSA_BMS_FW_v1105_Full_Analysis.md` §4.1). Differences in 110:

| ID | 110 |
|---|---|
| `0x1803AA01` (`CAN_TX_PF3_ChargeDischargeLimits` `0x08007C1C`) | charge voltage limit **46.8 V** (fixed, also PF3 clamp), charge current = system limit **≤ 40.0 A** (25.0 A on pack fault, **no 10.0 A clamp** below 35.00 V), discharge current **≤ 40.0 A**, status bits unchanged |
| `0x1804AA01` (`CAN_TX_PF4_ProtectWarnings`) | byte 4 `g_chargeStatusCode` = only the byte read from EEPROM 0xFC02 at boot (0xA5 after the first full charge); 1105 also sends 2/3/4 |
| `0x1820..0x1841AA00+n` (`CAN_TX_PerPack_10Msgs` `0x08008338`) | packs 1..6 (`pack_id != 0 && pack_id < 7`); 0x1821 carries the pack's own max charge current (no 10.0 A override); 0x1823 FW tag **1100** |
| `0x03410110` (inverter PV-rescue request) | **not handled** (1105: `g_pvRescueRequestFlag`) |
| `0x100201AA` | SOC reset via `CAN_RX_Handler` → `CAN_CMD_02_Handler` (1105 additionally decodes it in `CAN_RX_System_Command_Handler`) |

System command opcodes on `0x100101AA` (incl. 0x0A CAN restart, 0x0B become master, 0x0C become slave), the
`0x100501AA` heartbeat reply (`CAN_TX_OwnPackReport`), the staggered slave status report and the `0xA5A5` keepalive/echo
frames are unchanged.

### 4.2 RS485

Unchanged against 1105 (one bus on USART1, 9600 baud, `RS485_Dispatcher` `0x08010EB4`, version byte `'n'` = 0x6E,
13 cell voltages in opcode 0x19).

### 4.3 Debug Shell (letter-shell)

Unchanged: user `BLUE`, 52 table entries (10 key bindings, `help`, `clear`, 39 commands). The login banner prints
`Version: 110.0` and the build date.

### 4.4 EEPROM layout (I2C)

Unchanged against 1105: configuration block at 600 with version tag **`'n'`** (A 109: `'m'`; a tag mismatch makes
`BMS_Config_Params_Load_Or_Default` load the defaults), master marker `0x01F6`, counter `0x0254`, event log
`0x2000`/`0x3000`, error lock `0xFC00`, full-charge marker `0xFC02`.

## 5. Cell Monitoring — SH36735

Unchanged against 1105: SH36735 AFE on SPI1, CRC-8 polynomial 0x07, **13 cells** (configuration register 0x43,
`INIT_SH36735_Subsystem` `0x0800D680`), balancing rules, zero-offset calibration ("SELF" flag).

## 6. Pack Management

Master/slave model, elections, exclude mask, persistent master marker as in 1105. Differences:
`Pack_LowVoltage_Balancer` (`0x08007EAC`) only pulls packs with SOC < 8 % or Protect1 bit0 into rotation (no
"min cell < 2800 mV" criterion); `OwnPack_MOSFET_Driver_MasterVariant` (`0x08007150`) keeps the relays off while the
SELF flag is not set at any pack voltage (1105: only above 39.00 V).

## 7. SOC/SOH

As in 1105, except the empty threshold: `SOC_Select_Threshold_2420_2750` (`0x08013474`) returns **2750 mV** at ≥ 0 °C
(2420 mV below 0 °C), the Venus D value (1105: 2800 mV). The first full detection still writes the EEPROM marker
`0xFC02 = 0xA5`; the "never fully charged" derating is present (also in A 109).

## 8. Charge/Discharge Limits

| Item | Venus A 110 | Venus A 110.5 | Venus D 118 |
|---|---|---|---|
| System charge current cap (`g_sysChargeCurrentLimit_x10`, Modbus 32106) | **40.0 A** | 50.0 A | 50.0 A |
| System charge current outside master run state 2 | 0 | 50.0 A (0 while SELF flag set) | 0 |
| System discharge current cap (Modbus 32107) | **40.0 A** | 50.0 A | 50.0 A |
| Per-pack max charge current (`Max_Charge_Current_Calc` `0x08013858`) | table, max 50.0 A; while never fully charged 50.0 A below 45 °C, then 25.0 A − 1.67 A/°C | same | table only |
| Per-pack discharge current (`Max_Discharge_Current_Calc` `0x080139AC`) | table, max 50.0 A | same | same |
| Charge voltage limit (Modbus 35110) | **46.8 V fixed** | pack voltage + 2.0 V, max 48.0 V, CAN clamp 46.8 V | 57.6 V, CAN clamp 60.9 V |
| CAN charge current below 35.00 V own-pack voltage | no clamp | 10.0 A | no clamp |
| Discharge voltage limit | 39.0 V | 39.0 V | 48.0 V |

The system limit is the active pack's limit capped at 40.0 A (`Master_Build_ChargeCurrentLimit` `0x0800BC20`,
`Master_Build_DischargeCurrentLimit` `0x0800BC90`); the Control converts it into power (charge clamp ≈ V × I).

## 9. Protection Functions

Configuration defaults unchanged. Code against 1105:

- `Cell_OVP_UVP_TempCompensated_Protection` (`0x0800CD68`): UVP level-2 stage A (≥ 0 °C) **3000 mV**, a trip only sets
  `g_protState5` bit2 (warning); stage B (2500 mV) sets Protect1 bit5, clears the counters and resets the SOC estimator
  to 0 %. Min-cell sensor-fault trip (< 1500 mV) at any pack voltage. (= Venus D behaviour.)
- `BMS_ProtectionAndCommTick` (`0x0800963C`): while cell UVP level 2 is active the own MOSFETs are enabled only below
  2.0 A pack current and while `g_protCommFlagA` (PB1 output low and PE9 input high) is clear; a standalone pack (slot 0xFF, or slot 1 as
  the only pack) enters fault mode 3 after 200 s. No 120 s wait for an inverter request and no STOP mode from this
  path (1105: `System_Reset_And_Halt_WithCurrentCheck` after 120 s without `0x03410110`).
- `HW_OvercurrentAndSensorProtection` (`0x0800B09C`): comparator cross-check window **±3.0 A** (1105: ±70.0 A).
- Unchanged: Protect2 bits `0x400`/`0x2000`, fault masks, `System_Reset_And_Halt_WithCurrentCheck`.

## 10. OTA/Update

Unchanged: CAN `BMS_OTA_Flash_Writer` (`0x0800D23C`, opcode 0xCE) and `RS485_OTA_Flash_Handler` (`0x0800CA28`),
staging copy `0x0801E000`, CRC16, tag `'BMS2'`, boot-page repair. A 110 image is accepted by an A BMS running 109 or 1105
and vice versa (same tag); D images (`'BMS1'`) are rejected.

## 11. Security Notes

Unchanged against 1105 (shell commands on the RS485 debug path, unauthenticated CAN commands for master role, error
lock, factory reset and RTC, IWDG not started).

## 12. Cooperation with the Venus A Inverter (Micro 1211) and Control 151

- The 1105 changelog required inverter firmware ≥ 119.3 for the PV-rescue start. BMS 110 has **no** BMS side for it:
  frames `0x03410110` from the inverter are ignored, and the BMS does not enter the 120 s/STOP sequence in UVP level 2.
  Whether the A Micro 1211 rescue still works with BMS 110 depends on the Micro behaviour (not tested).
- Control: system limits ≤ 40.0 A (Modbus 32106/32107), constant 46.8 V charge voltage limit (35110), BMS version
  1100 (30204/34010/37012).

## 13. Open Points

1. Why the manufacturer released 110 from the 109 code base instead of 1105: not derivable from the code. Whether a
   later version merges both branches remains to be checked with the next image.
2. Actual field behaviour of the 40 A cap (CAN/Modbus values with BMS 110) not measured.
3. Open points 2–4 of the 1105 analysis (writer of the words sent in `0x1851/0x1852`, SH36735 registers, SELF-flag
   polarity) apply unchanged (same code).
4. Naming inherited from 1105 that looks wrong but was kept for cross-version consistency: `g_cliInsertCharEscSeqPtr`
   (`0x20002B34`) holds the login-banner pointer used by `CLI_LoginUser`.
