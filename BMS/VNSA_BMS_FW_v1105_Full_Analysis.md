# BMS-FW V110.5 (Venus A) — Full analysis

> 🇩🇪 Deutsche Version: [VNSA_BMS_FW_v1105_Vollanalyse_de.md](VNSA_BMS_FW_v1105_Vollanalyse_de.md)

**Status:** 2026-10-02.

**Reference image:** Venus D BMS 118 (`VNSD_BMS_FW_v118_Full_Analysis.md`) ·
**Decompilation/Listing:** `Dekompilate/VNSA_BMS_1105/` (local only, not in the repo) (698 functions, `listing/` with
flash disassembly + `globals_and_registers.txt`) · **Companion:** `Control_FW/VNSA_Control_FW_v1509_Full_Analysis.md`.

## 0. Summary

- Image base **0x08002800**, image size 0x19800 bytes (104,448 B), MCU **Nations N32G45x** (Cortex-M4F) — same MCU,
  toolchain (Keil ARMCC), RTOS (FreeRTOS, `heap_4`, heap `0x4FF0` B) and shell (letter-shell, default user `BLUE`) as
  the Venus D BMS. Version string `Version: 110.5`, build date `May 11 2026 09:47:24`.
- **Code base:** the A BMS is a sibling build of the D BMS code base. 77 % of the image bytes lie in segments that are
  identical to D 118 (pointer/BL-masked); 506 of the 711 D functions occur byte-identically, 57 with changes,
  148 D functions are absent (mainly the KA495XX AFE driver and the second RS485 bus). The A image adds ~130 functions
  of its own (SH36735 AFE driver, event log, RTC/counter/balancing shell commands, master marker, LED patterns).
- **Battery front end:** the A uses an **SH36735** AFE (part name from the strings `SH36735`, `SH36735_PRT`; vendor not verified) instead of the
  KA495XX of the D, with **13 cells per pack** (default; 4..12 configurable via AFE register 0x43, anything else falls
  back to 13). The D has 16 cells. The FreeRTOS task keeps the D name `RTOS_vTaskKA495XX`.
- **Voltage-related limits scaled to 13S LFP:** discharge voltage limit 48.0 V → **39.0 V**, CAN charge-voltage clamp
  60.9 V → **46.8 V**, charge-voltage limit fixed 57.6 V → **pack voltage + 2.0 V, max 48.0 V**, capacity factor in the
  CAN capacity frame 512 → **416** (13 × 32 instead of 16 × 32). Cell-level tables (OCV/SOC curve, NTC curves, charge
  and discharge derating matrices, axes, SOH curve) are **byte-identical** to D 118.
- **Changelog items found in the code:**
  1. "Deep-discharge recovery via PV start together with the inverter": while cell undervoltage level 2 is active the BMS
     now waits up to **120 s** for a CAN frame **`0x03410110`** from the inverter (payload byte 1 > 8 =
     `g_pvRescueRequestFlag`); without it the BMS enters stop mode (`System_Reset_And_Halt_WithCurrentCheck`). In
     addition the CAN charge-current limit is forced to **10.0 A while the pack voltage is below 35.00 V**, the UVP level-2
     stage-A threshold is 2800 mV (D: 3000 mV) and the min-cell sensor-fault trip is only active above 39.00 V.
  2. "Forced-charge logic optimisation": not identified with certainty; candidates are the 10 A low-voltage charge clamp
     and the new "never fully charged" derating in `Max_Charge_Current_Calc` (see §8).
- **All 698 functions named and typed**, 0 `FUN_*`, all verification gates passed (§2.3). Tags: `NAME_VERIFIED` 698,
  `NAME_LOW_CONFIDENCE` 49.

## 1. Binary Fingerprint

| File | Version | Size | SHA-256 |
|---|---|---|---|
| `1105_bms_VNSA-0_VA50A_APP_V1105_ota_144658.bin` | 110.5 (Beta) | 104,448 B (0x19800) | `c642c7ea1e84dc8d5848f86fb97f69baaa7e9c4f4ee41473bf972a836a0dfc3d` |

Manufacturer changelog (translated): "Use only together with inverter firmware 119.3 or later. 1. Ultra-low-voltage PV
start rescue of the battery together with the inverter. 2. Forced-charge logic optimisation." Image version tag at
`0x08010000`: **`BMS2`** (D: `BMS1`).

## 2. MCU, Memory Map, Processing Status

### 2.1 MCU: Nations N32G45x (Cortex-M4F)

Same evidence as for the D BMS: 102-entry vector table (16 + 86 IRQs), UID read at `0x1FFFF7F0`
(`HAL_ReadUniqueDeviceID`), N32 RCC/PWR layout in `SystemInit` (sets `SCB_VTOR = 0x08002800`). Register and IRQ names
from the N32G457 SVD. Peripherals used: SPI1 (AFE, PA4 chip select, polled), CAN1, USART1 (RS485, 9600 baud),
ADC, TIM2/TIM3, a GPIOD bit-banged serial bus to the RTC (code identical to D, where it is named `KA495XX_SPI_*`; since the A
has no KA495XX, this is very likely a separate RTC chip), I2C EEPROM.

### 2.2 Memory Map

| Range | Contents |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (not part of the image) |
| `0x08002000` | Boot/OTA handshake page `g_bootConfigPage` (inside the bootloader area) |
| `0x08002800`–`0x0801BFFF` | Application image; vector table at `0x08002800` |
| `0x08010000` | Image version tag `g_imageVersionTag` = `'BMS2'` |
| `0x0801AB10` | `g_crc8Poly07Table` (CRC-8, polynomial 0x07, SH36735 frames) |
| `0x0801B9D4` | Keil region table (2 regions), `0x0801B9F4` `shellCommandTable` (52 × 16 B), `0x0801BD34` `.data` load image |
| `0x0801E000` | OTA staging copy `g_otaStagingImage` (tag word at `0x0802B800` must be `'BMS2'`) |
| `0x20002800`–`0x20002B9F` | `.data` (0x3A0 B, LZ-compressed in the image; D: 0x3C4 B) |
| `0x20002BA0`–`0x2000C95F` | `.bss` (0x9DC0 B; D: 0x9FDC B); FreeRTOS `ucHeap` at `0x200046D8`, Keil `malloc` heap `0x20009964`–`0x2000A95F` |
| `0x2000C960` | `__initial_sp` (stack top) |
| I2C EEPROM | configuration block at 600, master marker `0x01F6`, counter `0x0254`, event log `0x2000`/`0x3000`, full-charge marker `0xFC02` (§4.4) |

The RAM layout differs from D 118 (piecewise shifts of 4–2,584 bytes); all RAM names were therefore ported by matching
the literal-pool references of paired code, not by address.

### 2.3 Processing Status in Ghidra

- Program `/VNSA/BMS/BMS_1105_VNSA-0_VA50A_APP_V1105_ota_144658.bin`: **698 functions, all named**, every function has a
  user-defined prototype (task entries `void f(void *pvParameters)`), plate comments, 0 `FUN_*`, 0 duplicates.
- Gates (`vna_bms_gates.py`, a split version of `marstek_gates_micro.py` plus two extra checks): 0 undefined bytes,
  0 orphan instructions, 0 flows into data, 0 function pointers to non-code, 0 literal-only functions, **0 literal pools
  decoded as code**, **0 calls into function bodies**, 0 unnamed referenced RAM globals, 0 default prototypes,
  task names = `xTaskCreate` name strings, no-return flags sane. Decompilation export: 698 functions, **0 failures**.
- Port: 563 D↔A function pairs (506 identical, 57 changed) carried names, prototypes, plates and locals over; each of
  the 57 changed functions was reviewed against the D code (52 keep the D name with an A-specific plate, 5 renamed).
  The ~130 A-only functions and 145 A-only RAM globals were named from the code.
- Corrections relative to a plain port: literal pools behind non-returning calls that the reference program also
  decodes as code (6 places, D 118 has the same defect), a 2-byte shift of the system SOC block
  (`g_sysMaxCellVoltage`… at `0x20004068`), D names in the AFE RAM area that belong to the old KA495XX driver.

## 3. FreeRTOS Tasks

`RTOS_CreateAllTasks` (`0x0800EA34`) creates the same 10 tasks as on D (names, stacks and priorities identical):

| Task | Entry | Stack (words) | Prio | A body |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800ECC0` | 0x80 | 5 | LED pattern (`LED_StatusPatternUpdate`, PB5) + idle relay toggle |
| `RTOS_vTaskKeyScan` | `0x0800ECA0` | 0x80 | 9 | pack-select relay state machines (no key scan) |
| `RTOS_vTaskKA495XX` | `0x0800EC88` | 0x80 | 8 | **SH36735** worker every 5 ms + `PerPack_Struct_Builder` |
| `RTOS_vTaskCanRec` | `0x0800EC28` | 0x180 | 4 | CAN RX processing |
| `RTOS_vTaskCanSend` | `0x0800EC38` | 0x80 | 3 | CAN TX queue |
| `RTOS_vTaskRS485Comm` | `0x0800ECFA` | 0x180 | 1 | RS485 (one bus) |
| `RTOS_vTaskSocAlgorithm` | `0x0800ED08` | 0x100 | 7 | SOC/SOH |
| `RTOS_vTaskProtectCheck` | `0x0800ECE4` | 0x100 | 6 | protection |
| `RTOS_vTaskControlLogic` | `0x0800EC42` | 0x80 | 2 | aggregation, protection/comm tick |
| `RTOS_vTaskDataSave` | `0x0800EC5C` | 0x80 | 0 | IWDG feed, RTC read, screen refresh |

The independent watchdog is only fed (`HAL_IWDG_Refresh`); unlike D (`System_Init_Main`: prescaler 6, reload 0x30D,
start) the A application never configures or starts it.

## 4. Interfaces

### 4.1 CAN (internal bus to Control, inverter and other packs)

Identifiers are 29-bit, same internal layout as D. Inbound frames are N32 `CanRxMessage` structures, so "byte n" below
is payload byte n (struct offset 0x0B + n).

**Outbound** (`CAN_TX_Orchestrator` `0x08007ACC`, answer to `0x1801nnAA`):

| ID | Function | Content (A) | Difference to D |
|---|---|---|---|
| `0x1801AA01` | `CAN_TX_PF1_PackMeasurements` | pack voltage (10 mV), total current (0.1 A), max NTC, average SOC | — |
| `0x1802AA01` | `CAN_TX_PF2_Capacity` | total capacity = packs × capacity × **416** / 10, pack count, online mask, active pack, discharge %, relay state | factor 512 on D |
| `0x1803AA01` | `CAN_TX_PF3_ChargeDischargeLimits` | charge voltage limit (clamped **46.8 V**), charge current (**10.0 A if pack voltage < 35.00 V**, 25.0 A on pack fault), discharge current, status bits | clamp 60.9 V, no low-voltage clamp on D |
| `0x1804AA01` | `CAN_TX_PF4_ProtectWarnings` | protect word 1/2, status code `g_chargeStatusCode`, online, error lock | D sends status code 0 |
| `0x1820..0x1823AA00+n`, `0x1830..0x1833AA00+n`, `0x1840/0x1841AA00+n` | `CAN_TX_PerPack_10Msgs` | per pack n = 1..6 (one-based: the bound is `pack_id != 0 && pack_id < 7`, records are indexed with `pack_id - 1`, so six packs): overview, limits, min/max, protect + FW tag; cell voltages (16 slots, 13 used); NTC/MOS temperatures | same frame set |
| `0x1851AA00+n`, `0x1852AA00+n` | `CAN_TX_PerPack_SocDiag` | **A only**: 7 words from `0x20004020`–`0x2000402E` (writer not found) | — |
| `0xA5A5` | `CAN_TX_KeepaliveA5A5` | **A only**: 1-byte frame (data 1) from the 500 ms idle tick when enabled; also used by the RS485 echo test | — |

On `0x100501AA` a pack answers with `CAN_TX_OwnPackReport` (PF1–PF4 when online/master + own per-pack frames + SOC diag
frames); on D this ID only set a CAN-restart flag. The 11-frame slave status report
(`CAN_TX_BMS_Status_Reporter`) is now sent after a slot-dependent delay ((slot − 1) × 15 ms,
`CAN_StatusReply_StaggerTick`) instead of immediately.

**Inbound** (`CAN_RX_Handler` `0x08008264`, `APP_CAN_RX_Task_Process` → `CAN_RX_System_Command_Handler`):

| ID | Meaning (A) | D |
|---|---|---|
| `0x1801nnAA` (nn = 1..6) | status request for pack nn → `CAN_TX_Orchestrator(nn)` | same |
| `0x1801FFAA` | master heartbeat to all packs: master-enable (byte 0 == 1), EMS flags (byte 1), discharge % (byte 2), watchdog reset → `CAN_TX_Orchestrator(0xFF)` | same |
| `0x100101AA` | system command, opcode = byte 0 (table below) | subset |
| `0x100201AA` | SOC reset to byte 0 % (`CAN_CMD_02_Handler`, event 3) | same id |
| `0x100301AA` | discharge limit % (`g_dischargeLimitPct`) + EEPROM write | same |
| `0x100401AA` | `CAN_CMD_04_Handler` | same |
| `0x100501AA` | second heartbeat: same fields as `0x1801FFAA`, but the reply is `CAN_TX_OwnPackReport` (own pack only) | only a CAN-restart flag (byte 0 == 1) |
| **`0x03410110`** | **A only:** `g_pvRescueRequestFlag = (byte 1 > 8)` — inverter request that keeps the BMS awake during deep undervoltage | — |
| `0xA5A5` | **A only:** echo-test reply | — |
| prefix 3 (`CAN_RX_Command_Dispatcher`) | slot commands: 1 status, 2 pack mode, 3 handshake, 4 status/alarm, 5 factory reset, 0x0A notify, 0xC4 activation, 0xCE OTA write | same set |

**System command opcodes `0x100101AA`** (`CAN_CMD_01_Handler` / `CAN_RX_System_Command_Handler`):

| Opcode | Action | D 118 |
|---|---|---|
| 1 | address request (pending system command 1) | yes |
| 2 | factory reset + reboot (`APP_Factory_Reset_And_Reboot`) | yes |
| 3 | relay command | yes |
| 4 | fault mode 4 | yes |
| 7 | pack handoff request | yes |
| 8 | **unlock**: clear error lock (EEPROM `0xFC00` marker) | yes |
| 9 | **lock**: set error lock | yes |
| 0x0A | **restart CAN** (`CAN1_Reinit_OrSystemReset`) | **new** |
| 0x0B | **become master**: master marker → EEPROM, event 2, factory reset/reboot, slot 1 | **new** |
| 0x0C | **become slave**: clear master marker, slot 0xFF, system reset | **new** |

An "encode" opcode does not exist in the BMS. `CAN_CMD_Pack_Activation` additionally knows opcode 5 (factory reset)
and opcode 8 (**set RTC** from a packed 32-bit date/time: year 6 bit, month 4, day 5, hour 5, minute 6, second 6).

### 4.2 RS485

Only **one** RS485 bus on USART1 (9600 baud, PA9/PA10, no driver-enable pin); D drives two (USART2 + UART4, DMA). The
IRQ handler is `USART1_IRQHandler` (vector IRQ37). Frames are delimited by a 3,645 µs gap timer and start with `'s'`;
the dispatcher (`RS485_Dispatcher` `0x08011004`) keeps the D opcode set plus:

| Opcode | A behaviour |
|---|---|
| `0x03` | **new:** balancing on/off (manual mode, all cells) |
| `0x05` | always answers with version byte `0x6E`; the D handshake (`A1 B1` → pending flag) is gone |
| `0x10` | **new:** cell-voltage stream on/off (every 100 polls an `0x19` frame) |
| `0x19` | cell voltages: 13 cells (0x1A bytes) instead of 16 |
| `0x2F` | **new:** CAN echo test (frame `0xA5A5`) |
| `0x00` | status reply without the raw KA495XX fault word |

`RS485_Pack_Telemetry_Response` reports config version byte `'n'` (0x6E; D 118: `'v'`). The dispatcher still contains
a D-ported test for channel 3 (UART4), which has no context in the A image (dead path).

### 4.3 Debug Shell (letter-shell)

User `BLUE` ("default user"), 52 table entries: 10 key bindings, `help`, `clear` and **39 commands** (D: 25). All 23
remaining D commands exist with the same names (`bat_data`, `set_soc`, `set_dod`, `checkout_rs485`, `force`,
`set_work_pack`, `state_printf`, `sox_data`, `wake`, `inq_uid`, `uid_printf`, `parall`, `self_flag`, `reset`, `can_send`,
`update_printf`, `event`, `prt`, `task_size`, `factory`, `lock_set`, `volt_diff`, `set_flg`). New on A:

| Command | Handler | Function |
|---|---|---|
| `TOL_V` | `CLI_Cmd_PrintTolVoltage` | ADC channel 0x0B → V (/4095 × 3.3 / 0.0208) |
| `BAL_ON` / `BAL_OFF` | `CLI_Cmd_BalanceOn` / `CLI_Cmd_BalanceOff` | manual balancing all cells / off |
| `EVEN_BALL` / `ODD_BALL` | `CLI_Cmd_BalanceEvenCellsOn` / `CLI_Cmd_BalanceOddCellsOn` | balance even / odd cells |
| `SH36735` | `CLI_Cmd_PrintSH36735Raw` | AFE raw values (VADC, CADC, CCUR, VBAT1/2, balance index, SELF flag) |
| `ctl_prt` | `CLI_Cmd_PrintLccAndLoadVoltage` | LCC pin (PC2) and load voltage |
| `cnt_read` / `cnt_write` | `CLI_Cmd_CounterRead` / `CLI_Cmd_CounterWrite` | EEPROM counter at `0x0254` |
| `addr_save` / `addr_reset` | `CLI_Cmd_AddrSave` / `CLI_Cmd_AddrReset` | master marker save / clear |
| `log_prt`, `log_read`, `log_write` | `CLI_Cmd_LogPrintIndex`, `CLI_Cmd_LogRead`, `CLI_Cmd_LogWrite` | event log |
| `rtc_read` / `rtc_write` | `CLI_Cmd_RtcRead` / `CLI_Cmd_RtcWrite` | external RTC |

After login the shell prints an ASCII banner with build date and version (`CLI_LoginUser`).

### 4.4 EEPROM layout (I2C)

| Address | Contents |
|---|---|
| 600 (0x258), 0xD4 B | configuration block (`BMS_Config_Params_Load_Or_Default`), version tag **`'n'`** (D: `'v'`), ones-complement checksum |
| `0x01F6` | **A only:** master marker `{00 AA FF 55}` (complement check); read at boot → `g_pack0SlotId = 1`, else `0xFF` |
| `0x0254` | **A only:** counter `{v, ~v}` (shell `cnt_read`/`cnt_write`) |
| `0x2000`… | **A only:** event-log index, 50 × 8 B `{packed time u32, event u16, record address u16}` |
| `0x3000`–`0x3CE3` | **A only:** event records, 0x42 B: time, SOC, current, max/min cell, max/min NTC, MOS temps, 16 cell slots, 4 NTC, protect words 1/2 |
| `0xFC00` | error-lock marker (`0x86` locked / `0xFF` clear), same as D |
| `0xFC02` | **A:** full-charge marker `0xA5` (written at the first full detection). D writes a 10-byte RTC stamp here |

Event IDs: 0 SOC calibration point (full/empty), 1 protection state changed, 2 became master, 3 SOC reset via CAN,
4 display SOC jump > 5 %.

## 5. Cell Monitoring — SH36735

- SPI1, polled, chip select PA4 (`SH36735_SpiTransfer` → `HAL_SPI_TransmitReceive_Polling`). Read frame
  `{02, reg, len}` → reply `{FF, 02, reg, len, data…, crc}`; write frame `{01, reg, value, crc}` → ack `… A5`.
  CRC-8 polynomial **0x07** (`SH36735_Crc8`, `g_crc8Poly07Table`; D KA495XX: polynomial 0xD5). 5 retries per transfer;
  persistent failure sets `g_protState4` bit 4 (new Protect2 bit `0x400`) and, debounced, `g_sh36735StatusFlags` bit 7.
- Registers used: configuration 0x40–0x54 (written and verified every second, 0x51–0x54 fixed `96 76 76 9E`),
  balance mask 0x55–0x57, flags 0x58/0x59/0x5B/0x5C, NTC 0x5D (4 channels), internal temperature 0x65, "VADC" 0x67,
  cell voltages 0x69 (13 × 16 bit), current "CADC" 0x91, pack voltage 0x93, second voltage 0x95, counter 0x97.
- Cell count: configuration image register 0x43, low 5 bits; default `0x6D` → **13 cells** (`INIT_SH36735_Subsystem`).
- Scaling: cell mV = raw × 5000 / 32000; pack mV = raw × 50000 / 12800; current = 4-sample average with 100 mA dead
  band; NTC via resistance and the D NTC tables. Zero-offset auto calibration of the current channels
  (`SH36735_ZeroOffsetCalibration`, "E2ssCurrOffset1/2").
- Timing: worker every 5 ms, measurement cycle ~70 ms, slow path 1 s (`SH36735_TaskStateMachine`).
- **Balancing** (`SH36735_UpdateBalancingMask`): automatic only if the cell spread ≥ 20 mV and the pack current
  ≥ 1000 mA; a cell is balanced when ≥ 3000 mV and ≥ 50 mV (below 3400 mV) resp. ≥ 20 mV above the minimum cell for
  29 consecutive passes. D (KA495XX): spread ≥ 30 mV, current ≥ 1.001 A, on threshold 3501 mV. Manual modes via shell
  (`BAL_ON`, `EVEN_BALL`, `ODD_BALL`) and RS485 opcode 3.

## 6. Pack Management

Same master/slave model and pack-selection family as D. Differences: all elections (`PackSelect_Mode3_Empty`,
`_Discharge_StepRotate`, `_Charge_StepRotate`, `_HighestSoc_Online`) skip packs in `g_packExcludeMask`; the master role
can be fixed persistently by CAN opcodes 0x0B/0x0C (EEPROM master marker); `Pack_LowVoltage_Balancer` additionally pulls a
pack into rotation when its minimum cell is below 2800 mV. MOSFET gates (`OwnPack_MOSFET_Driver`, `_MasterVariant`)
use the SH36735 comm-failure bit instead of the KA495XX watchdog and are blocked until the zero-offset calibration is
done ("SELF" flag, master variant only above 39.00 V).

## 7. SOC/SOH

Identical algorithm and tables (OCV/SOC curve, coulomb counting, SOH curve `SOH_LookupByCycleCount` over
`g_ocpDelayVsCurrentTable` — a misnomer inherited from D). Differences: empty threshold 2800 mV at ≥ 0 °C (D: 2750 mV;
2420 mV below 0 °C unchanged), full threshold 3700/3600 mV unchanged; full/empty detections write event 0; the first full
detection writes the EEPROM marker `0xFC02 = 0xA5`; `SOC_OcvBoundsRecalibrateOnFlag` and the RTC stamp writes of D 118
are absent; the display SOC update logs event 4 on jumps > 5 %.

## 8. Charge/Discharge Limits

| Item | Venus A 110.5 | Venus D 118 |
|---|---|---|
| Max charge current | 50.0 A (table, identical matrix) | 50.0 A |
| Charge current while never fully charged (`g_neverFullChargedFlag`) | 50.0 A below 45 °C, then 25.0 A − 1.67 A/°C | — |
| System charge current when not in master run state | 50.0 A, 0 while the zero-offset calibration flag is set | 0 |
| Charge voltage limit | pack voltage + 2.0 V, max 48.0 V; CAN clamp 46.8 V | 57.6 V fixed; CAN clamp 60.9 V |
| CAN charge current at pack voltage < 35.00 V | 10.0 A | no clamp |
| Discharge voltage limit | 39.0 V | 48.0 V |
| Discharge current | 50.0 A cap, identical voltage/temperature matrix | same |
| Config: short-circuit stage-1 timeout | 5000 ms (both reset paths) | 20000 ms |

## 9. Protection Functions

Configuration defaults are identical to D (cell OVP 3600/3650/3700 mV, UVP 2800/2700/2600 mV, release 3350/3150 mV,
OC 55 A/SC 60/100 A, temperature thresholds), except the short-circuit stage-1 timeout (5 s). Code differences:

- `Cell_OVP_UVP_TempCompensated_Protection`: UVP level-2 stage A (≥ 0 °C) 3000 → **2800 mV**; a trip sets
  `g_protState1` bit 5, clears the trip counters and resets the SOC estimator to 0 %. Min-cell sensor-fault trip
  (< 1500 mV) only while the pack voltage is above 39.00 V.
- **Deep-undervoltage handling** (`BMS_ProtectionAndCommTick`, 1 s tick): D — with UVP level 2 active the own MOSFETs are
  enabled only below 2.0 A, and a standalone pack goes to fault mode 3 after 200 s. A — counts `g_uvpLvl2ShutdownTimerS`;
  after **120 s without `g_pvRescueRequestFlag`** (CAN `0x03410110`, byte 1 > 8) the BMS calls
  `System_Reset_And_Halt_WithCurrentCheck` (STOP mode, then system reset); the D current/standalone logic is gone.
- `HW_OvercurrentAndSensorProtection`: comparator cross-check window ±3.0 A → ±70.0 A.
- Protect2 word: new bits `0x400` (SH36735 communication failure) and `0x2000` (latched by an address-reset command);
  fault scan / all-off masks extended to `0x2E00`.
- `System_Reset_And_Halt_WithCurrentCheck`: no EEPROM fault stamp before the STOP mode (D writes an RTC stamp to `0xFC02`
  when |P| > 5 kW).

## 10. OTA/Update

Same CAN (`BMS_OTA_Flash_Writer`, opcode 0xCE) and RS485 (`RS485_OTA_Flash_Handler`) paths, staging copy at `0x0801E000`,
CRC16 check, bootloader handshake magics `0x11223344`/`0x55667788`. **The staging image must carry `'BMS2'`** (D: `'BMS1'`),
so D BMS images are rejected by the A BMS and vice versa. New: `OTA_EnsureBootConfigPage` repairs the bootloader page
`0x08002000` (state word 3) at boot if it is inconsistent.

## 11. Security Notes

- Shell login user `BLUE` and the full command set (incl. `factory`, `lock_set`, `addr_save`/`addr_reset`, `rtc_write`,
  `log_write`) are reachable on the RS485 debug path as on D.
- CAN commands without authentication can switch the master role persistently (0x0B/0x0C), set/clear the error lock
  (8/9), factory-reset (2, slot command 5) and set the RTC.
- The IWDG is never started by the application (only fed); a hang is not caught unless the bootloader starts it.

## 12. Cooperation with the Venus A Inverter (Micro 1211) and Control 1509

- The inverter must send CAN **`0x03410110`** with payload byte 1 > 8 within 120 s of a cell UVP level-2 event, otherwise
  the BMS sleeps; this is the BMS side of the "PV rescue start" (changelog: inverter ≥ 119.3).
- During the rescue the BMS announces only 10.0 A charge current (`0x1803AA01`) while the pack voltage is below 35.00 V,
  and a 46.8 V charge-voltage ceiling.
- Control/EMS: same request/heartbeat protocol; the heartbeat `0x100501AA` now triggers a full report of the own
  pack; system opcodes 0x0A–0x0C are new.

## 13. Open Points

1. Exact meaning of the payload of `0x03410110` (byte 1 > 8 — PV voltage, state code?) — needs the A Micro analysis.
2. Writer of the words `0x20004020`–`0x2000402E` sent in `0x1851/0x1852AA00+n` (no code writes them in this image).
3. Meaning of SH36735 configuration registers 0x41–0x54, 0x95, 0x97 and the VADC channel.
4. Polarity of the "SELF" flag (`g_socStateFlags` bit 5): the calibration sets it when done and the MOSFET gates need it,
   but `Master_Build_ChargeCurrentLimit` sets the charge limit to 0 while it is set (outside master run state).
5. "Forced-charge logic optimisation" of the changelog: not assigned to a specific code change.
6. Reference-program issues found on the way (D 118, not changed there): `APP_SystemFaultHalt` is the C `main()`;
   `g_ocpDelayVsCurrentTable` is the SOH curve; the `KA495XX_SPI_*`/`RTC_*` bit-bang functions very likely talk to a separate RTC
   chip, not to the AFE; six literal pools behind non-returning calls are decoded as code.
