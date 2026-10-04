# BMS-FW v118 (Venus D) — Full analysis

> 🇩🇪 Deutsche Version: [VNSD_BMS_FW_v118_Vollanalyse_de.md](VNSD_BMS_FW_v118_Vollanalyse_de.md)

**Status:** 2026-09-30.

**German:** `VNSD_BMS_FW_v118_Full_Analysis.md` ·
**Predecessor full analysis:** `VNSD_BMS_FW_v1177_Full_Analysis.md` ·
**Diff 117.7 → 118:** `VNSD_BMS_Diff_v1177_v118.md` ·
**Charge limits (detailed):** `VNSD_Charge_Limits_Charge_Power.md` ·
**Decompilation/Listing:** `Dekompilate/VNSD_BMS_118/` (local only, not in the repo) (711 functions, 778 RAM globals, `listing/` with
Flash raw disassembly + `globals_and_registers.txt`) · Comparison export: `Dekompilate/VNSD_BMS_1177/`.

## 0. Summary

- Image base **0x08002800** (bootloader below, page `0x08002000` = boot/OTA handshake
  `g_bootConfigPage`), image size 0x1A000 bytes (106,496 B), MCU **Nations N32G45x** (Cortex-M4F),
  Keil ARMCC, FreeRTOS (`heap_4`, heap `0x4FF0` B), letter-shell (36 table entries, default user
  `BLUE`), communication CAN + RS485, cell monitoring KA495xx (SPI, CRC-8).
- **All 711 of 711 functions named**, all verification gates passed (see §2.3). Compared to the
  old analysis (`VNSD_BMS_FW_Analysis_v118.md`, 550 functions, wrong base `0x08000000`, MCU
  incorrectly identified as AT32F403A) this is a complete rebuild — details in §13.
- **Charge limit:** maximum 50.0 A per pack (temperature × SoC table), charge termination 57.6 V, only
  one pack physically loads at a time (`PackHandoff_StateMachine`/`OwnPack_MOSFET_Driver`). Summary in
  §8, full derivation incl. measurement verification in `VNSD_Charge_Limits_Charge_Power.md`.
  The notorious "148-W plateau" observation came **not** from the BMS, but from the
  Omnibattery integration, and is not repeated here.
- **Diff to 117.7 (§0a, complete details in `VNSD_BMS_Diff_v1177_v118.md`):** 88 of 711 functions
  marked as "changed". Each of the 88 pairs was checked line-by-line in C code (not just listing diff): **11 are real, 77 are cosmetic** (decompiler/register noise or pure
  address shift due to code growth). Together with the one completely new function
  `SOC_OcvBoundsRecalibrateOnFlag` (`0x08016634`), this gives **12 real changes** total. The most important ones (complete list incl. C code evidence in diff document, §2):
  1. **Discharge limit percent (`g_dischargeLimitPct`)** is now additionally propagated via the periodic
     CAN heartbeat (`CAN_RX_Handler`), and the pack rotation logic depending on it was repaired/extended (`PackSelect_DischargeStepThreshold`, `PackSelect_Discharge_StepRotate`) — in
     117.7 this percentage had no impact on the rotation threshold.
  2. **New OCV recalibration function `SOC_OcvBoundsRecalibrateOnFlag`**, hooked into
     `SOC_AlgorithmOrchestrator`, plus two new RTC/EEPROM timestamp log entries on mode change to idle. The flag it consumes `g_socOcvRecalibrateRequest` is never set to 1 or 2 in any of the 710/711 functions — the function is according to current knowledge **dead code**; the effective OCV recalibration path remains the existing 1-hour idle timer.
  3. **New error timestamp log on emergency reset** (`System_Reset_And_Halt_WithCurrentCheck`), if the
     last measured pack power was outside ±5 kW.
  4. **Config/protocol version tag** `g_cfgVersionTag` switches from `'u'` (117.7) to `'v'` (118)
     at six places in sync (config guard, RS485 version, pack telemetry). Practical consequence: a configuration block saved under 117.7 is invalid under 118 on first boot (tag mismatch) and is reset once to factory defaults and resaved.
  5. **Shutdown timing** (`PackMode_Shutdown_Handler`): `g_pendingActivationFlag` is set after the
     300 ticks wait time rather than immediately.
  All other CAN/RS485 opcodes, charge currents/tables and protection threshold **values** are identical between 117.7 and 118.

## 0a. Checklist

| Question | Answer | Evidence |
|---|---|---|
| CAN protocol changed compared to 117.7? | No (opcodes/IDs identical), **one** behavior change: `CAN_RX_Handler` additionally takes up `g_dischargeLimitPct` from byte 0xD in the master heartbeat frame. | `l1177_118.json` (`CAN_RX_Handler` marked as "changed") + manual diff `VNSD_BMS_1177/080081dc_...` vs. `VNSD_BMS_118/08008240_...` |
| RS485 protocol changed? | Opcodes/framing (XOR checksum, ~53 commands) identical. Only change: embedded version byte in pack telemetry response switches `'u'`→`'v'` (mirror of config tag). | Diff `RS485_Pack_Telemetry_Response` |
| Charge/discharge limits changed? | Hard limits unchanged: `Max_Charge_Current_Calc`/`Max_Discharge_Current_Calc` and underlying tables (`g_chargeCurrentDerateMatrix`, `g_dischargeCurrentDerateMatrix`, axes) are byte-identical. Only changed **how** the already existing configurable discharge limit percent (`g_dischargeLimitPct`) affects the pack rotation logic — `PackSelect_DischargeStepThreshold` did not depend on this global at all in 117.7 (see summary point 1). | Diff of both functions against 1177; diff document §2.1 |
| Protection thresholds changed? | No (numeric values identical: `Voltage_Temp_Protection_Checker`, `Cell_OVP_UVP_TempCompensated_Protection`, `BMS_Config_Params_Factory_Reset`/`Load_Or_Default` byte-identical except version tag). The long-standing inconsistency between the two reset paths (short-circuit stage-1 timeout 20000/20000 vs. 5000/10000 ms) exists in **both** firmware versions equally, so is not new to 118 — see §9. | Diff + `g/gout_02.json` note |
| OTA still possible (CAN/RS485, bootloader handshake)? | Yes, unchanged. `BMS_OTA_Flash_Writer`/`_FinalizeTail`, staging address `0x0801E000`, CRC16/Modbus, tag check `'BMS1'`, handshake magics `0x11223344`/`0x55667788` are byte-identical to 117.7. RS485 OTA path (`RS485_OTA_Flash_Handler`) likewise unchanged present. | Diff `BMS_OTA_Flash_Writer_FinalizeTail`, §10 |
| Cloud connection? | N/a — the BMS has no WAN/cloud connection, only CAN to Control/Micro and RS485 to slave packs. | — |

## 1. Binary Fingerprint

| File | Version | Size | SHA-256 |
|---|---|---|---|
| `118_bms_VNSD-0_20260119100535e43806957.bin` | 118 | 106,496 B (0x1A000) | `fdcefaec7a8c0cac8850b3abb53a87acff38dafb92fa0af6f4f2f191d9fc7589` |
| `1177_bms_VNSD-0_20251010135647565eb2036.bin` | 117.7 (Beta) | 106,496 B (0x1A000) | `69b18aa42831536f01546bbe707e36ee65b6faa84e6cf4e92f9fbfe237f67c77` |

(`shasum -a 256` against files under `Marstek FW Archiv/firmwares/VNSD-0/BMS/118/` resp. `/1177/`.)
Manufacturer changelog (`CHANGELOG.de.md` (local only, not in the repo)) lists for both versions only "OTA-Test" (117.7 additionally "7"),
no functional description.

## 2. MCU Identification, Memory Map, Processing Status

### 2.1 MCU: Nations N32G45x (Cortex-M4F)

- ADC1/ADC2 registers at `0x40020800`/`0x40020C00` — this is the N32G45x memory map, not
  the one from AT32F403A (which the old analysis was based on, see §13).
- 96-bit unique ID read at `0x1FFFF7F0` (`HAL_ReadUniqueDeviceID`), documented N32 UID address;
  alternative read path `0x1FFFF270` if main address is erased (`0xFFFFFFFF`).
- 102 entries in vector table (86 IRQs + 16 system vectors).
- Register names from official N32G457 SVD (NSING DFP 1.0.7). An undocumented register `0x40001800`
  is read by `PWR_ApplyCtrl3TrimFromReg40001800` (trim source, not in SVD).

### 2.2 Memory Map

| Range | Contents |
|---|---|
| `0x08002800` – `0x0801FFFF` | Firmware image (0x1A000 B), vector table from `0x08002800` |
| `0x08002000` (below) | Bootloader/boot config page (`g_bootConfigPage`, handshake see §10) |
| `0x08010000` | Version tag `'BMS1'` |
| `0x0801E000` | OTA staging copy (new image written here, tag check at `0x0802B800`) |
| `0x20000000` ff. | SRAM (144 KB); `.data` 0x3C4 B LZ-compressed in image (`Load$$RW_DATA$$Base`
  at image end) → unpacked to `0x20002800`; `.bss` to `0x2000CB9F` (118) |
| `0x2000CBA0` | Initial stack pointer (from vector table) |
| I2C-EEPROM `0x0000`–`0x0FFF` (example) | Configuration block, error lock, SOC calibration,
  event log — see §4.4 |

.data/.bss between 117.7 and 118: 117.7 has 16 B smaller `.bss` and heap `0x4FF8` B (instead of
`0x4FF0` B in 118); `__scatterload_copy` at `0x0800FFEC` in 117.7, at `0x08010D66` in 118;
two small functions accordingly moved (117.7 `0x0800FF8C`/`0x0800FF98` ↔ 118 `0x08010BCC`/`0x08010BD8`). This explains why practically every later function address between the two versions is shifted by a few bytes, although the code is identical.

### 2.3 Processing Status in Ghidra

- **All 711 of 711 functions named** (0 remaining `FUN_*`), 4,236 Ghidra symbols, 0 failed
  decompilations (`_index.md`, status 2026-09-30 19:21).
- **778 RAM globals** renamed and commented (`listing/globals_and_registers.txt`, incl.
  N32G45x peripheral registers).
- All verification gates (naming without placeholder remainders, cross-reference consistency
  between function and globals exports, alignment of 117.7↔118 function pairing) passed per project status; see `l1177_118.json` (88 changed / 1 only in 118 / 622 identical / 0 only in 117.7 — complete pairing, no orphaned pair).
- Restriction from naming notes (`out_*.json`/`gout_*.json`): several function names still carry a historical, provably false `FATFS_` prefix from an older naming run (e.g., `CLI_ProcessCommandLine`, `CLI_ResolveVariableReference`,
  `CLI_CheckCommandPermission` — all clearly letter-shell functions, not FatFs) and have been corrected in the current export; see §11 for remaining ambiguous cases.

## 3. FreeRTOS Tasks

`RTOS_CreateAllTasks` (`0x0800e7bc`) creates all 10 application tasks:

| Task Name | Entry | Stack (words) | Priority | TCB Handle |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x0800ea31` | `0x80` | 5 | `0x20002b08` |
| `RTOS_vTaskKeyScan` | `0x0800ea17` | `0x80` | 9 | `0x20002b0c` |
| `RTOS_vTaskKA495XX` | `0x0800ea09` | `0x80` | 8 | `g_ka495xxTaskHandle` |
| `RTOS_vTaskCanRec` | `0x0800e9b1` | `0x180` | 4 | `0x20002b14` |
| `RTOS_vTaskCanSend` | `0x0800e9c1` | `0x80` | 3 | `0x20002b18` |
| `RTOS_vTaskRS485Comm` | `0x0800ea67` | `0x180` | 1 | `0x20002b1c` |
| `RTOS_vTaskSocAlgorithm` | `0x0800ea75` | `0x100` | 7 | `0x20002b20` |
| `RTOS_vTaskProtectCheck` | `0x0800ea51` | `0x100` | 6 | `0x20002b24` |
| `RTOS_vTaskControlLogic` | `0x0800e9cb` | `0x80` | 2 | `0x20002b28` |
| `RTOS_vTaskDataSave` | `0x0800e9e5` | `0x80` | 0 | `0x20002b2c` |

Highest priority is `RTOS_vTaskKeyScan` (9), followed by `RTOS_vTaskKA495XX` (8, cell monitoring)
and `RTOS_vTaskSocAlgorithm` (7). RS485 communication and data save run at lowest priority (1 resp. 0). The shell debug command `task_size` (`RTOS_PrintTaskStackHighWaterMarks`,
`0x080156bc`) prints stack high water marks for all 10 tasks.

## 4. Interfaces

### 4.1 CAN

Addressing: `CAN_Build_Arbitration_ID` (`0x08003570`) packs a 29-bit ID from message group,
subtype/pack type and pack ID in Venus internal board-to-board layout; `g_pack0SlotId`
(own pack rotation slot, `0xFF` = unregistered/offline, `1` = active master) is included
if < 0xF, otherwise 0xF (broadcast) is used.

**Inbound** (`CAN_RX_Handler`, `0x08008240`, top-level dispatcher):

| CAN ID (or ID family) | Meaning |
|---|---|
| `0x180102aa` | Request broadcast → `CAN_TX_Orchestrator(0xff)` (all packs transmit) |
| `0x180101aa` | Pack-addressed status request → `CAN_TX_Orchestrator(pack_id)` |
| `0x100101aa` | `CAN_RX_System_Command_Handler`: byte 1=address request, 2=factory reset, 3=relay command, 8=clear error lock |
| `0x100201aa` | `CAN_CMD_02_Handler` |
| `0x100301aa` | Direct set of `g_dischargeLimitPct` + flash write |
| `0x100401aa` | `CAN_CMD_04_Handler` |
| `0x100501aa` | CAN RX restart flag (byte 1 == 1) |
| Family `…+0x100`/`0x200`/`0x300`/`0x400` (offset decoding) | Various status requests |
| Family `…+0xfd00` (master heartbeat/enable) | Sets `g_masterEnableReceivedFlag`, `g_lastCanCmd04PayloadFlags`, **as of v118 additionally `g_dischargeLimitPct`** (byte 0xD, see §0a), `g_packOnlineWatchdogTimerMs=0`, then `CAN_TX_Orchestrator(0xff)` |

Second command family with `(id>>0xc)==3` (`CAN_RX_Command_Dispatcher`, `0x0800f128`): opcode in
lowest byte of ID, addressed to own pack slot, broadcast (0xF), or 0:

| Opcode | Action |
|---|---|
| `0x01` | `CAN_TX_Send_Status_Response` |
| `0x02` | `CAN_Handle_Pack_Mode_Command` |
| `0x03` | Pack fault-mode address assignment/confirmation |
| `0x04` | `CAN_TX_BMS_Status_Reporter` (11 telemetry frames) |
| `0x05` | `APP_Factory_Reset_And_Reboot` |
| `0x0a` | `CAN_Set_Notify_Flag_And_Send` |
| `0xc4` | Set pending activation flag (only if sub-byte nibble 0) |
| `0xce` | `BMS_OTA_Flash_Writer` (CAN OTA, see §10); additionally sets `g_otaInProgressFlag` |

**Outbound** (`CAN_TX_Orchestrator`, `0x08007b84`): sends 4 fixed "protocol frame" messages once, then per pack a 10-message burst:

| Frame | Function | Contents |
|---|---|---|
| PF1 | `CAN_TX_PF1_PackMeasurements` (sub-ID `0x20`) | Voltage, current + 2 more 16-bit fields |
| PF2 | `CAN_TX_PF2_Capacity` | Capacity |
| PF3 | `CAN_TX_PF3_ChargeDischargeLimits` | Charge current limit, charge termination voltage (capped at 60.9 V max) — see §8 |
| PF4 | `CAN_TX_PF4_ProtectWarnings` | Two 16-bit warning words + global fault byte + relay eviction byte |
| PerPack ×10 | `CAN_TX_PerPack_10Msgs` (`0x08008358`) | Overview (voltage/current/SoC/cycles/limits/protect/version, groups `0x20`–`0x23`), 16 cell voltages in 4 frames (`0x30`–`0x33`), temperature/MOS data (`0x40`–`0x41`) |

`CAN_TX_BMS_Status_Reporter` (`0x0800ecd8`, 11 frames: SoC, cycle count, cell voltages/differences,
temperatures, currents, protect flags) is sent selectively on opcode `0x04` as response to an
EMS/Control request, not in broadcast burst.

**Pack activation:** `CAN_CMD_Pack_Activation` (`0x0800df70`) and
`CAN_Pack_Activation_Config_Receive` (`0x0800e18c`) handle the transfer of a 12-byte MCU UID per
slave pack from three consecutive CAN frames plus the activation configuration.
`CAN_RxSlavePackTelemetry_Unpack` (`0x0800dbf4`, 854 B) unpacks the complete pack telemetry
of a slave pack including MOS status bits.

### 4.2 RS485 to Slave Packs

Framing: XOR checksum over the frame (`LIB_XOR_Checksum`), byte 2 = opcode. `RS485_Dispatcher`
(`0x08010004`, 1,616 B) is the central dispatcher with **~53 opcodes** (`0x00`–`0x34`, not all
used); selected opcodes:

| Opcode | Meaning |
|---|---|
| `0x00` | Status word (relay GPIOs, pack status flags, KA495xx fault word) |
| `0x01` | Boot marker `0x41` |
| `0x02` | Software reset (SCB AIRCR) |
| `0x05` | Balancing precondition + magic byte check → `g_rs485DispatchPendingFlag` |
| `0x0e` | `RS485_OTA_Flash_Handler` (RS485 OTA passthrough, see §10) |
| `0x11`/`0x12` | Read/write parameter group (register group IDs, see below) |
| `0x18`–`0x1b` | Pack current, cell voltages + balance mask, NTC temperatures, SoC |
| `0x1e`–`0x21` | Read/write user flag / tail register incl. flash storage |
| `0x24` | SOC override (`SOC_ResetEstimatorToPercent`) |
| `0x29`/`0x2a`→internal | Inter-pack struct sync (pack slot records, 0x60 B each) |
| `0x2c`/`0x2d` | Read/write RTC |
| `0x2e`/`0x30` | Pack telemetry response to host (`RS485_Pack_Telemetry_Response`, see below) |
| `0x2a` | `System_Reset_And_Halt_WithCurrentCheck` |
| `0x31` | Checkout mode/channel switching |
| `0x34` | Relay pulse (GPIOB bit `0x1000`) |

**Parameter groups** (`RS485_Register_Read_Handler`/`_Write_Handler`, group ID = 16-bit word):

| Group ID | Contents |
|---|---|
| `0x0001` | Charge NTC OV/UV trip1/2 + release + timeouts |
| `0x0002` | Discharge NTC OV/UV trip1/2 + release + timeouts |
| `0x0004` | MOS temp B Hi/Lo trip1/2 + release + timeouts |
| `0x0008` | MOS temp A Hi trip1/2 + release + timeout |
| `0x0010` | Cell UVP trip1/2a/2b + release + timeouts |
| `0x0040` | Cell OVP trip1/2a/2b + release + timeouts |
| `0x0080` | 1 word, meaning not fully determined |
| `0x0100` | Discharge OC/SC trip1/2 + timeouts + reserve |
| `0x0200` | Charge OC/SC trip1/2 + timeouts + reserve |
| `0x0400` | 3 more, not fully named registers |

`RS485_Pack_Telemetry_Response` (`0x08012414`) writes at offset `0x48` the config version tag byte
(117.7: `'u'`, 118: `'v'` — see §0a) into the response buffer. `RS485_PrechargeRelaySequence`
(`0x0800d6dc`) runs a 4-stage relay sequence (pickup/dropout, GPIOB `0x1000`) with
acknowledgment check and retry.

### 4.3 Debug Shell (letter-shell)

`CLI_ShellInit` (`0x080147bc`) initializes the instance with 5 history slots and logs in by default
as user **`BLUE`** (`shellCommandTable` entry 0, password attribute `0x0800`, description
"default user"). The command table (`0x0801c394`, 36 entries × 16 B) contains then 10
key bindings (arrow keys, tab, backspace, delete, enter) and 25 actual commands:

| Command | Target function | Description |
|---|---|---|
| `help` | `CLI_Cmd_Var` (`0x08014654`) | "show command info\r\nhelp [cmd]" |
| `clear` | `CLI_NotifyPendingShell` (`0x08013f28`) | Clear console (ANSI `\x1b[2K\r`) |
| `bat_data` | `CLI_Cmd_PrintAllPackStatus` (`0x080151d0`) | Status of all slave packs (address, online, version, voltage, current, SoC, cycles, MOS, NTC) |
| `set_soc` | `SOC_Cmd_ForceSocOverride` (`0x0801561c`) | SOC override: 0=release, 1–99=set directly, 100=full latch |
| `set_dod` | `PackSelect_SetSocFloorMargin` (`0x08015234`) | SOC margin for pack selection lower limit, mirrors to AFE config register |
| `checkout_rs485` | `RS485_SetDebugCheckoutFlag` (`0x080151c4`) | Debug output via RS485 instead of UART |
| `force` | `Protect_PrintUnderVoltageStatus` (`0x08015274`) | UVP debug status (test index, UV flag, min voltage, MOS commands) |
| `set_work_pack` | `PackSelect_SetTargetPackIndex` (`0x08015608`) | Set target pack index for pack selection |
| `state_printf` | `PackSelect_PrintCurrentPackStatus` (`0x08015630`) | Active pack index + charge/discharge MOS commands |
| `sox_data` | `SOC_PrintDebugSummary` (`0x08015628`) | SOC/SOH debug summary |
| `wake` | `PackSelect_TriggerWakePulse` (`0x08015614`) | Wake pulse on GPIOB pin 3 |
| `inq_uid` | `Debug_TriggerSystemEventF` (`0x080153c8`) | Trigger system event `0xF` |
| `uid_printf` | `PackSelect_PrintPackUids` (`0x08015918`) | 12-byte UID of all 6 possible slave packs |
| `parall` | `BMS_PrintBatterySummary` (`0x08015448`) | Voltage, current, temperature extrema, SoC, protect words |
| `self_flag` | `BMS_SetEmsChgDsgFlag` (`0x08015248`) | Set EMS charge/discharge flag |
| `reset` | `NVIC_SystemReset` (`0x080155b8`) | Cortex-M reset |
| `can_send` | `CLI_Cmd_SendTestCanFrame` (`0x080151d8`) | Send test CAN frame ID `0x112233` |
| `update_printf` | `BMS_PrintSystemStatusFlags` (`0x08015998`) | Comm prohibit, upgrade/update finish, factory flag, EMS charge/discharge flag |
| `event` | `PackSelect_SetRelayCycleState` (`0x08015254`) | Set master relay switch state directly (0–4) |
| `prt` | `RS485_SetDebugCmdPendingFlag` (`0x080155ac`) | RS485 debug command pending flag |
| `task_size` | `RTOS_PrintTaskStackHighWaterMarks` (`0x080156bc`) | Stack high water mark of all 10 tasks |
| `factory` | `KA495XX_SetSleepMode` (`0x08015260`) | AFE sleep mode toggle (despite name "factory") |
| `lock_set` | `Protect_SetErrorLock` (`0x080153d4`) | Set/release error lock (AFE lock register + relay reset) |
| `volt_diff` | `BMS_PrintChgConnAndVoltDiff` (`0x08015b14`) | Cell voltage difference + charge connector GPIO |
| `set_flg` | `CAN_SetControlLinkFlag` (`0x080155fc`) | Flag "valid command/heartbeat from EMS over CAN received" |

The last column combines the shell help text byte embedded in the binary with the Ghidra comment
of the respective target function. Notable: `factory` command calls AFE sleep mode (name/function discrepancy in
original shell code, not in Ghidra naming) despite its name.

### 4.4 I2C EEPROM / Flash Layout

- **Config block**: `I2C_EEPROM_Read_WithMutex(600, 0xd4, &g_cfgMagicByte)` reads 0xD4 bytes from
  EEPROM address 600; validity via `g_cfgMagicByte=='b'` (implicitly, always reset),
  `g_cfgVersionTag` (`'v'` in 118, `'u'` in 117.7) and ones-complement checksum
  (`LIB_Checksum_Ones_Complement`, 0xD0 bytes) in `g_cfgChecksum`. On mismatch: complete
  reset to ~80 factory-default fields (`BMS_Config_Params_Load_Or_Default`, identical to
  `BMS_Config_Params_Factory_Reset` except timeout inconsistency noted in §9) and
  `APP_Config_Checksum_And_Save`.
- **Error lock**: `APP_SetErrorLockFlag` writes marker byte **`0x86`** (locked)
  resp. **`0xFF`** (cleared) to **flash/EEPROM offset `0xFC00`** and mirrors to
  `g_errorLockActiveFlag`. `OwnPack_MOSFET_Driver` blocks both charge/discharge paths while lock is active.
- **SOC calibration timestamp**: `SOC_OcvBoundsRecalibrateOnFlag` (new in 118, see §0a) writes
  an RTC timestamp (10 B) directly following at **`0xFC02`**.
- **SOC calibration data**: `SOC_SaveCalibrationToFlash` writes max/min/real/show SOC (each x10000) and total capacity (0x14 B) plus a complementary "erased" record
  (`g_flashParamBufFF`, 20×`0xFF`) to two rotating 20-byte slots from `0x400`
  (`APP_EventLogGetSlotAddrs`, 2 slots, ring index 0/1).
- **"FatFs" log**: `FATFS_Init_Log_File_If_Needed` (`0x0800ff6c`) despite its name performs no true
  FAT filesystem operation, but merely initializes if needed a second
  `CLI_ShellInit` instance with 512-byte buffer for RS485 checkout mode. The name is
  likely a remnant from an earlier, provably wrong naming run (several other `FATFS_*`
  functions in this image were already recognized as misnamings and corrected to their actual domain — letter-shell resp. FreeRTOS — see §2.3/§11). True FatFs binding (SD card etc.) is not detectable in the 711 functions.

## 5. Cell Monitoring KA495xx

- SPI connection (`INIT_KA495XX_SPI_GPIO`, `KA495XX_SPI_Read/Write_Register(s)`), frame protection by
  **CRC-8** (polynomial `0xD5`, start value `0xD5`, `g_ka495xxCrc8Table`, `KA495XX_Crc8` `0x08016b94`).
  16 cells, 5 NTC channels (`KA495XX_ReadTemperatureChannels`).
- **Written protection/config registers** (`KA495XX_WriteProtectionRegs`, `0x0801695a`, via
  `KA495XX_SpiWriteMaskedRegister`, each full 16-bit mask):
  - Register `0x0B`: Cell voltage protection configuration (`KA495XX_EncodeCellVoltageProtectionConfig`)
  - Register `0x0A`: Overcurrent/short circuit configuration (`KA495XX_EncodeOcscConfig`)
  - Register `0x13`: OVP/UVP/OCD thresholds (`KA495XX_EncodeOvpUvpOcdThresholds`)
  - Register `0x14`: Sleep timing configuration (`KA495XX_EncodeSleepTimingConfig`)
  - Write only if `KA495XX_CellBalanceChannelConfig` previously confirmed a valid
    channel configuration table.
- **Balancing decision** (`KA495XX_CellBalanceDecision`, `0x08017498`): balancing completely
  disabled while cell spread `<30 mV` (`0x1e`) or pack current `<1.001 A` (`0x3e9`); per cell turn-on threshold **3.501 V** (`0xdad`) unless `<30 mV` below max cell, turn-off
  threshold **3.500 V** (`0xdac`) or `<20 mV` below max, each with debounce
  (`Protect_Debounce_TripCounter`, 1000 ms timeout). Result mask feeds
  `KA495XX_BuildBalancingBitmask`.
- **Fault/status word**: `KA495XX_UpdateFaultStatusWord` consolidates AFE error bits into
  `g_ka495xxFaultStatusWordRaw` (also queryable via RS485 opcode `0x00`, see §4.2).
- Scaling: current/power values calculated with empirical correction factors **0.88** (current) resp.
  **0.92** (power) from raw AFE ADC value (`SOC_UpdatePowerAndCoulombCounters`,
  `0x08017c9c`) — presumably shunt/amplification corrections, not further documented.

## 6. Pack Management

- **Roles**: `g_pack0SlotId` = `0xFF` unregistered/offline, `1` = active master claim; values
  `2..n` are slave slots. Master runs pack selection and handoff state machines, slaves
  respond to CAN/RS485 requests and deliver telemetry.
- **PackSelect family** (`PackSelect_ModeDispatch`, `_Mode1_Full`, `_Mode2_LowSoc`, `_Mode3_Empty`,
  `_Mode3_TemperatureTimeout`, `_Discharge_StepRotate`, `_Charge_StepRotate`,
  `_Next_AboveSocThreshold`, `_HighestSoc_Online`, `_LowestSoc_Online`, `_Startup_Closest50` etc.)
  selects the next pack for rotation based on SOC mode (`PackSelect_ModeBySoc`), charge/discharge
  step thresholds (`PackSelect_ChargeStepTarget`/`_DischargeStepThreshold`) and online status;
  `PackSelect_Startup_Closest50` searches at startup for the pack whose SoC is closest to 50% (`CAN_Find_Closest_Pack_To_50pct`).
- **Handoff**: `PackHandoff_StateMachine` (`0x08006740`) transfers master/charge permission
  step by step from `active_pack` to `target_pack`: clear old discharge enable → wait until
  old pack MOS actually off → set new discharge enable → wait until **exactly one** pack
  has active MOS (`LIB_Popcount` check) → ramp charge enable while MOS safe (SOC threshold
  98.6% / `0x62`) → commit handoff and recalculate rotation thresholds
  (`PackSelect_ModeBySoc`/`_ChargeStepTarget`/`_DischargeStepThreshold`).
- **MOSFET control**: `OwnPack_MOSFET_Driver` (`0x08006bc4`) and the master variant
  `OwnPack_MOSFET_Driver_MasterVariant` (`0x08007284`) switch charge/discharge MOS/relays only when
  a long list of protection bits (short circuit, OV/UV, overcurrent, temperature, precharge status,
  communication watchdog, `g_errorLockActiveFlag`) is clean; otherwise both relays are hard
  switched off. Precharge sequence runs via `g_prechargeSmState`, GPIOB `0x1000`, with
  feedback check.
- **Precharge via RS485**: `RS485_PrechargeRelaySequence` (§4.2) runs an independent 4-stage
  sequence with retry for cases where relay control is triggered by RS485 command.
- **Single-pack charging enforced**: `PackHandoff_StateMachine` never sets the charge MOSFET mask to
  more than one bit at a time; `OwnPack_MOSFET_Driver` switches a pack in only if its bit
  is set in this mask — physically only one pack can charge at a time
  (detailed in `VNSD_Charge_Limits_Charge_Power.md` §1.2).

## 7. SOC/SOH

- **OCV initialization**: `SOC_InitFromOcvLookup` sets real/show/max/min/target SOC on first valid
  voltage/temperature values from `Full_OCV_Lookup` against `g_ocvSocCurveTable`
  (21 support points SoC % → OCV for 4 temperature zones, mV).
- **Coulomb counting**: `Coulomb_Counting` (`0x08011ba8`) integrates current × time with
  float residual accumulator (`g_coulombCountResidual`) against rounding drift; `Capacity_Factor_Calc`
  scales the calibrated capacity (`raw/10000/100`). The correction factors **0.88**/**0.92**
  mentioned in the task text are in the upstream current/power calculation
  (`SOC_UpdatePowerAndCoulombCounters`, §5), not directly in `Coulomb_Counting` itself.
  `SOC_CoulombCountTick` and `SOC_UpdatePowerAndCoulombCounters` additionally maintain 64-bit
  energy/coulomb counters (`g_bmsEnergyCounterLo/Hi`, `g_bmsCoulombCounterLo/Hi`).
- **Calibration/persistence**: `SOC_SaveCalibrationToFlash`/`SOC_LoadCalibration_Float`/
  `SOC_EepromLoadCalibration` store max/min/real/show SOC (x10000) and total capacity to the
  rotating EEPROM slots (§4.4). `SOC_OcvBoundsRecalibrateOnFlag` (new in 118) enforces
  OCV re-calibration on flag set outside a voltage "knee" band and stamps an RTC timestamp.
- **Cycle count**: `CycleCount_From_Throughput` converts accumulated charge throughput to
  full-cycle equivalents (`throughput/10/5.12`).
- **SOH**: `SOH_From_Throughput` → `SOH_LookupByCycleCount` interpolates linearly over a
  7-point table (`g_ocpDelayVsCurrentTable`, despite name reused here as cycles→SOH curve:
  (0, 100%), (500, 95%), (1000, 92.5%), (2000, 89.15%), (6000, 80%),
  (8000, 71%), (20000, 0%)); result is SOH ×1,000,000, capped at 100%. The table
  variable carries a name from another context (OCP delay), but is provably the cycles/SOH curve here — name debt, not functional bug.

## 8. Charge/Discharge Limits

Complete derivation incl. Control/Micro limits and measurement verification in
`VNSD_Charge_Limits_Charge_Power.md`. Key statements for the BMS (v118, unchanged to 117.7):

- **Charge current**: `Max_Charge_Current_Calc`/`Master_Build_ChargeCurrentLimit` limit to maximum
  **50.0 A** via an 18(temperature, −25…60 °C in 5 °C steps)×12(SoC support points
  0/5/10/20/30/40/50/60/70/80/90/95/100 %)-matrix (`g_chargeCurrentDerateMatrix`, percent value × 5 =
  0.1-A units). Charge termination voltage fixed **57.6 V** (`0x240`), capped at 60.9 V max on CAN.
- **Discharge current**: `Max_Discharge_Current_Calc` (`0x08012758`) uses its own
  18×5-matrix (`g_dischargeCurrentDerateMatrix`), indexed via temperature zone (same
  18-point axis `g_tempAxisTable`) and a **cell voltage bracket** (not SoC): for
  temperature zone index < 5, voltage axis `g_dischargeVoltAxisWarm`
  **{0, 2400, 2550, 2750, 3600} mV**, else `g_dischargeVoltAxisCold`
  **{0, 2750, 3000, 3050, 3600} mV** used (axis names appear swapped with temperature
  range — index < 5 corresponds to coldest zones per `g_tempAxisTable`;
  see §12). Read raw values (18×5-matrix, percent × 5 = 0.1-A units, 0 = blocked):

  | Temp zone (lower limit) | Bracket 0 (≤ axis value 1) | 1 | 2 | 3 | 4 (top cell) |
  |---|---|---|---|---|---|
  | ≤ −25 °C | 0 | 0 | 0 | 0 | 0 |
  | −20 °C | 0 | 0 | 10 | 20 | 50 |
  | −15 °C | 0 | 0 | 15 | 30 | 60 |
  | −10 °C | 0 | 0 | 20 | 30 | 60 |
  | −5 °C | 0 | 0 | 20 | 40 | 60 |
  | 0 °C | 0 | 0 | 20 | 50 | 70 |
  | 5 °C | 0 | 0 | 20 | 50 | 80 |
  | 10 °C | 0 | 0 | 20 | 50 | 90 |
  | 15–45 °C (zones 8–12) | 0 | 0 | 20 | 50 | 100 |
  | 50 °C | 0 | 0 | 20 | 40 | 80 |
  | 55 °C | 0 | 0 | 20 | 40 | 60 |
  | 60 °C | 0 | 0 | 20 | 40 | 40 |
  | 65 °C | 0 | 0 | 20 | 20 | 20 |
  | ≥ 70 °C | 0 | 0 | 0 | 0 | 0 |

  100% ⇒ 50.0 A. Unlike charging, discharge limit is thus primarily **voltage-**, not
  SoC-driven: only above the respective third/fourth voltage threshold is current
  released meaningfully, in worst bracket (low cell voltage) it stays at 0.
- **Always only one pack charges** — physically enforced (§6). This limit applies regardless of
  source (AC or PV) and is unchanged to 117.7.

## 9. Protection Functions

`BMS_Config_Params_Factory_Reset` (`0x0800a98c`) and `BMS_Config_Params_Load_Or_Default`
(`0x08009ccc`, on checksum/tag mismatch) set the same ~80 fields; default values
(trip1/trip2/release each in 0.1 °C resp. 0.1 V resp. ampere, timeouts in ms):

| Protection group | Trip1 | Trip1-TO | Trip2 | Trip2-TO | Release | Release-TO |
|---|---|---|---|---|---|---|
| MOS temp B Hi | 60.0 °C | 3000 | 65.0 °C | 3000 | 60.0 °C | 3000 |
| MOS temp B Lo | −25.0 °C | 3000 | −30.0 °C | 3000 | −25.0 °C | 3000 |
| MOS temp A Hi | 90.0 °C | 3000 | 114.6 °C | 3000 | 85.0 °C | 3000 |
| Charge NTC Hi/Lo | 60.0 / −15.0 °C | 5000 | 65.0 / −20.0 °C | 5000 | 56.6 / −17.0 °C | 5000 |
| Discharge NTC Hi/Lo | 55.0 / −15.0 °C | 5000 | 60.0 / −20.0 °C | 5000 | 56.6 / −17.0 °C | 5000 |
| Cell OVP | 3.600 V | 1000 | 3.650/3.700 V (a/b) | 3000/1000 | 3.350 V | 1000 |
| Cell UVP | 2.800 V | 1000 | 2.700/2.600 V (a/b) | 3000/1000 | 3.150 V | 1000 |
| Discharge OC/SC | 5.5 A (OC) | 3000 | 6.0 A (SC1) / 100 A (SC2) | **5000**/500 (Factory) or **20000**/500 (Load-Default) | Release-TO 60000 |
| Charge OC/SC | 5.5 A (OC) | 3000 | 6.0 A (SC1) / 100 A (SC2) | **10000**/500 (Factory) or **20000**/500 (Load-Default) | Release-TO 60000 |

**Confirmed inconsistency** (verified byte-by-byte in both functions, not just from task text): `g_cfgDsgScTrip1Timeout`/`g_cfgChgScTrip1Timeout` are in
`BMS_Config_Params_Factory_Reset` **5000 / 10000 ms**, but in `BMS_Config_Params_Load_Or_Default`
**20000 / 20000 ms**. Since Load_Or_Default applies on every boot with invalid/missing
configuration block (thus practically relevant path at initial startup or after version tag change, see §0a), field likely runs at 20000 ms mostly for both short-circuit stage-1
times, while an explicit factory reset (shell command `factory`? — no, that calls
`KA495XX_SetSleepMode`, see §4.3; the actual trigger for `BMS_Config_Params_Factory_Reset`
was not located in this session, see §12) would set different values. This inconsistency exists unchanged in 117.7 and 118 (diff shows no deviation except version tag).

**Status words**: `g_protState1` (bits 0/1 charge/discharge OC stage 2, bit 5 cell UVP stage 2 → SoC 0%,
bit 6 cell OVP stage 2 → SoC 100%), `g_protState2` (MOS/cell over-temperature latches, stage 1/2,
Hi/Lo, charge/discharge), `g_protState3` (HW overcurrent comparator cross-check, NTC/MOS sensor
range error bit 5–7, plus one bit read by MOSFET control), `g_protState4`
(AFE communication error latch bit 3, feeds `Warning_Bitmask_Builder`). These four bytes
feed `CAN_TX_PF4_ProtectWarnings` (two 16-bit warning words + global fault byte) and
RS485 opcode `0x09`.

## 10. OTA/Update

- **CAN OTA**: `BMS_OTA_Flash_Writer` (`0x0800d264`) buffers 1-KB blocks, checks each block by
  **CRC16/Modbus** (`LIB_CRC16_Modbus_Calc`), writes them to external/internal flash at
  **`0x0801E000`** (staging), verifies total CRC on completion against `g_otaImageExpectedCrc`
  and tag `g_otaStagingVersionTag == 0x31534D42` (ASCII `'BMS1'`, little-endian). On success,
  `OTA_ImageChecksum_OnesComplement` calculates an additional ones-complement checksum and
  `OTA_Set_Reboot_Params` (`0x08011618`) writes the **bootloader handshake**
  (magic `0x11223344`/`0x55667788`, target entry, size, mode 2) to the shared boot config page
  (`HAL_Flash_Write_Config_Block`) — then `vTaskDelay(1000)` and reboot request.
- **RS485 OTA**: `RS485_OTA_Flash_Handler` (`0x0800ca44`, opcode `0x0e`) is a separate,
  RS485-driven OTA path with its own target pack field (`g_rs485OtaTargetPackId`) — likely for
  passing firmware to slave packs over the RS485 bus, independent of the CAN OTA path.
- Byte-for-byte comparison with 117.7 shows **no** functional differences in
  `BMS_OTA_Flash_Writer`/`_FinalizeTail` or `OTA_Set_Reboot_Params` — only address-based
  literal shifts. OTA is in both versions possible via CAN and RS485, with identical
  handshake.

## 11. Security Findings

- **Error lock trivially clearable via CAN/RS485**: CAN opcode `0x08` (`CAN_RX_System_Command_Handler`)
  and RS485 (`lock_set` shell command or parameter group write path) can
  reset `g_errorLockActiveFlag` without further authentication except bus membership. Anyone with
  physical/electrical access to CAN or RS485 can lift a protection shutdown. Whether this is
  intended factory behavior (service reset) or weakness cannot be determined from firmware alone.
- **Default shell login `BLUE`** in clear text in image (`s_BLUE_0801bc48`) and auto-logged on every
  `CLI_ShellInit` (no password enforcement for factory session).
- **OTA without visible signature check**: Both CAN and RS485 OTA paths verify only CRC16 (transmission
  integrity) and a 4-byte ASCII tag (`'BMS1'`), no cryptographic signature. Anyone able to write to CAN or
  RS485 bus can load arbitrary firmware as long as CRC and tag match.
- **`g_ka495xxAdcLsbScale`** (AFE ADC scaling) flows into both hardware protection thresholds
  (OVP/UVP/OCD register encoding) and coulomb/SOC calculation — a wrong value
  (e.g., from faulty calibration) would simultaneously corrupt protection thresholds and displayed
  current/SoC (note from naming evidence, not further verified).
- **Historical misnamings with "FATFS_" prefix**: several initially classified as FatFs functions proved to be pure FreeRTOS or letter-shell internals
  (`CLI_ProcessCommandLine`, `CLI_ResolveVariableReference`, `CLI_CheckCommandPermission`,
  `usTaskCheckFreeStackSpace`, `pvTaskIncrementMutexHeldCount` etc., see §2.3) and were corrected in current export. `FATFS_Init_Log_File_If_Needed` (§4.4) carries the same suspicious
  prefix and was not renamed in this session — its actual behavior (shell init, no file operation) also speaks against true FatFs binding.

## 12. Open Points

- Which concrete trigger actually calls `BMS_Config_Params_Factory_Reset` (in contrast to the automatic
  `_Load_Or_Default` path) was not traced in this session
  (caller list shows 1 caller per `_index.md`, not further pursued).
  The shell command `factory` provably calls `KA495XX_SetSleepMode`, not this function.
- Meaning of register group ID `0x80` (RS485 parameter group, 1 word) and the three registers in
  group `0x400` not fully determined.
- Exact unit of `KA495XX_ScaleInternalTempCode` (raw code vs. 0.1 °C) not independently confirmed.
- Whether the swapping of axis names `g_dischargeVoltAxisWarm`/`_Cold` in
  `Max_Discharge_Current_Calc` (§8) is a naming error or whether the underlying
  `BMS_TempZoneIndexLookup` indexing here reads differently than in the discharge power table,
  not fully determined.
- Trigger condition/exact target string of `CLI_NotifyPendingShell` (also used for `clear`
  command) not fully verified (context globals vs. decompiled source discrepancy per naming evidence).
- Whether `RS485_Register_Read_Handler` group ID `0x80` and the HW overcurrent comparator bits in
  `g_protState3` are related, not checked.
- Exact address range and purpose of full I2C EEPROM layout (which ranges between `0x000` and
  `0xFC00` are occupied by what) determined only for fields concretely referenced in this document,
  not fully mapped.

## 13. Corrections vs. Old Analysis

Reference: `VNSD_BMS_FW_Analysis_v118.md` (old analysis, Ghidra import 2026-07-14/15).

- **Image base wrong**: old analysis started from `0x08000000`; correct is `0x08002800` (bootloader
  below). Every flash address in the old analysis is thus 0x2800 too low.
- **MCU incorrectly identified**: old analysis (and the 117.7 version it referenced) assumed
  **AT32F403A**; the actual peripheral address layout (ADC1/2 at `0x40020800`/`0x40020C00`,
  UID at `0x1FFFF7F0`) belongs to **Nations N32G45x** (see §2.1). The field observations/measurements of
  the old analysis remain unaffected, only the MCU/register assignment was wrong.
- **Function count**: old analysis 550/550 named; current export **711/711** — complete
  rebuild with corrected base address, not same function count plus delta.
- **One finding of old analysis independently confirmed**: the there (under wrong address,
  `FUN_08005a40` as presumed successor of `CAN_RX_Handler`) described new line — a byte
  from CAN frame offset `0xD` is additionally taken into `DAT_200028E0` — corresponds exactly to the finding in
  §0a against correct 118 export: `CAN_RX_Handler` since v118 additionally writes
  `g_dischargeLimitPct` from byte `0xD` of master heartbeat frame. `DAT_200028E0` and
  `g_dischargeLimitPct` thus designate the same RAM global, just under old resp. new name.
- **148-W plateau not repeated**: as per task instruction, the presumed BMS cause for the 148-W
  plateau in earlier documents is not adopted here — the cause was the
  Omnibattery integration, not the BMS (see `VNSD_Charge_Limits_Charge_Power.md` §4, there with
  measurement derivation).
- Further details (register/struct layout, according to old analysis header "still valid unless described differently here") were in this session not point-by-point compared, but completely rederived from 118 export with corrected base address; deviations to old analysis beyond those named were not systematically collected.
