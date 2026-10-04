# Control FW v151 (Venus E3) — Full Analysis and Differences to Venus D v151

> 🇩🇪 Deutsche Version: [VNSE3_Control_FW_v151_Vollanalyse_de.md](VNSE3_Control_FW_v151_Vollanalyse_de.md)

**Status:** completed 2026-10-02. All statements were checked against the code of this image; open points are listed in §12.
**Binary:** `151_control_VNSE3-0_ems_app_0151_0928_094421.bin` (archive `Marstek FW Archiv/firmwares/VNSE3-0/Control/151/`)
**Reference:** Venus D Control v151 (`151_control_VNSD-0_ems_app_0151_0929_094333.bin`), see [VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md](VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md); second relative: [VNSA_Control_FW_v1509_Full_Analysis.md](VNSA_Control_FW_v1509_Full_Analysis.md)
**Ghidra:** `/VNSE3/Control/Control_151_VNSE3-0_ems_app_0151_0928_094421.bin` (project `Marstek_Venus`)
**Decompilation:** `Dekompilate/VNSE3_Control_v151/` (local only, not in the repo): 1927 functions, 0 decompilation failures, + `listing/` (flash listing, `globals_and_registers.txt`)
**Modbus descriptor table:** [../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v151.csv](../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v151.csv)

---

## 0. Summary

- **Same code base as the Venus D, built without PV/MPPT, stack and parallel features.** 93.4 % of the image is byte-identical
  to D v151 once pointers and BL offsets are masked (410 segments; 80.2 % without BL masking). Of the 1956 D functions, 1845 are
  instruction-identical in the E3, 49 are the same function with changes, 24 sit at other addresses (moved or reduced, see §3) and
  **31 are compiled out**. The E3 has one small function of its own (a dirty-bit wrapper, §4.6).
- **RAM layout differs from the D** (unlike the Venus A): `.data` 12 016 B (D 12 032 B), `.bss` ends 0x20021DF7 (D 0x2002200F),
  external SRAM `.xmc_bss` 113 752 B (D 158 456 B). Most globals sit 16 bytes lower than on the D; the big step in external SRAM
  (−44 640 B) is the telemetry history ring (144 records of 0x227 B instead of 0x35D B, PV/MPPT fields removed). Names were carried
  over by code alignment, not by address (§11).
- **Modbus register map identical to D v151.** Same 246 descriptor entries (30000–38014), same types/scales/counts; only the RAM
  source pointers differ. `Write_Handler`, `FC03_Read_Handler` and `Read_Serializer` are instruction-identical (§5).
- **Power limits unchanged (2500 W)**, but the "set version" power selector now offers **600 W** (new, mode 2), 800 W (mode 1) and
  2500 W (mode 0); the D's 2200 W (mode 3) is gone (BLE, MQTT and JSON-RPC paths).
- **No PV/MPPT, no battery stack, no parallel inverters:** PV fields in BLE/MQTT/cloud payloads, the grid export limiter, the surplus
  feed-in check, the MQTT stack-BMS / BMS-data / MPPT commands (cd 0x28–0x2D), BLE commands 0x40–0x43, the CAN date/time broadcast
  (0xCC) and the 0xC4 writes are removed; parallel-node detection/sync functions are empty.
- **Two spellings of the model id:** runtime model/type string `VNSE3-0` (MQTT topics, cloud, shell user `VNSE3-0 v151`,
  BLE name `MST_VNSE3_`), but the OTA/boot descriptor check uses **`VNSEE3-0`** and the image footer reads `VNSEE3`.

## 0a. Checklist (E3 151 compared with D 151)

| # | Question | Answer E3 151 | Evidence |
|---|---|---|---|
| 1 | Modbus registers different? | **No.** Descriptor table entry-by-entry identical (246 entries); `Write_Handler` @ 0x08054254 instruction-identical (46000 = 0x5100 debug backdoor, 45000 block, 47001 as on the D). Register 30200 reads the version word @ 0x08010000 = **151**. Registers fed by PV/MPPT or by packs 2–8 have no data source on the E3. | §5 |
| 2 | BLE different? | Device name `MST_VNSE3_` + 4 ID bytes. Cmd 0x15 power selector 600/800/2500 W (D: 800/2200/2500 W). Cmds 0x40–0x43 do nothing. Cmd 0x03 status reply 0x8B bytes (D 0xA7), cmd 0x04 info string without `mppt_v`. | §4.1, §4.3 |
| 3 | AC limits? | Max charge/discharge power **2500 W** (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock` identical to D); new 600 W selector value; no grid export limiter (D: 4000 W clamp). | §4.1 |
| 4 | Firmware update via BLE possible? | Yes, same OTA code; model checks compare against **`VNSEE3-0`** (`OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`, boot descriptor in `ProcessFirmwareUpdateCommand`). | §4.5 |
| 5 | Cloud domains? | Unchanged vs D 151 (`api-<region>.marstekcloud.com` getDateInfo, marstekcloud data-upload, hamedata schedulings/policies/backup-configs). `getDateInfo` without `&mpptv=%d`; upload JSON without `pv/sk/mv/me/ma/fu`. | §4.4 |
| 6 | Pack capacity? | Control keeps the D structures: `Protocol_AA_CommandDispatch` accepts units < 9, 8 per-pack records of 0x60 B (`g_bmsPack0..7`, `g_bmsPack<k>Voltage` @ 0x200177C4 + k·0x60), Modbus per-pack blocks 34000–34699 unchanged. Everything that summarises several packs was removed; MQTT telemetry carries the 16 cell voltages of pack record 0 only. | §6a |

## 1. Binary fingerprint

| | E3 151 | D 151 (reference) |
|---|---|---|
| Size | 378 880 B (0x5C800) | 393 216 B |
| SHA-256 | `a2f566ebdf3f94a5b2b171e071bd65237d0cd0ae68035dab5fc34734cf250148` | — |
| Link address | **0x08004800**–0x08060FFF (18 KB bootloader below) | 0x08004800–0x080647FF |
| MCU | Nations **N32G45x** with XFMC (Cortex-M4F), 144 KB SRAM | same |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW) | same |
| Build string | `Sep 28 2026 09:39:27` | `Sep 29 2026 09:42:27` |
| Version | `VNSE3-0 v151` (shell root); version word 0x97 (151) @ 0x08010000 followed by `VNSEE3-0` | 0x97, `VNSD-0` |
| Image footer | `VNSEE3` + zero bytes @ 0x08060FE0, before it 0xFF fill from 0x08060C10 | `VNSD` |
| Device type code | `g_deviceTypeCode` = 0x97 (same value as D) | 0x97 |

Base proof: the vector table at 0x08004800 resolves exactly like on the D (Keil reset pattern; `SystemInit` @ 0x080307A0 writes
`SCB->VTOR = 0x08004800`); the scatter table, string and function pointers resolve at this base.

## 2. Memory map (Ghidra)

| Block | Range | Contents |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | not in the image; 0x08001000 bootloader version word, 0x08004000 boot parameter page |
| flash | 0x08004800–0x08060FFF | application |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-bit UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EEF | RW initial values (LZ77 @ 0x0805FED4, 2492 B) |
| .bss | 0x20002EF0–0x20021DF7 | zero-initialised; `ucHeap` @ 0x20003134 (80 KB); initial SP 0x20021DF8 |
| sram_unused | 0x20021DF8–0x20023FFF | up to the N32G45x 144 KB limit |
| .xmc_bss | 0x60000000–0x6001BC57 | external SRAM via XFMC (LZ77 image @ 0x08060890, 896 B) |
| peripherals | apb1/apb2/ahb/bit-band/ppb | register names from the Nations N32G457 SVD (copied 1:1 from D 151, same hardware map) |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC and QSPI controller |

Scatter table `Region$$Table$$Base` @ 0x0805F994: .data 0x0805FED4 → 0x20000000 (0x2EF0, `__decompress` @ 0x08005FBE),
XMC 0x08060890 → 0x60000000 (0x1BC58), .bss zero-init 0x1EF08 @ 0x20002EF0.

## 3. Relationship to Venus D v151 (method and numbers)

| Category | Functions | Meaning |
|---|---|---|
| instruction-identical | 1845 | same bytes apart from call/pointer targets; names, prototypes, plates, locals taken from D 151 |
| changed | 49 | same function, different constants or code (§4) |
| other address / reduced | 24 | e.g. `ProcessFirmwareUpdateCommand`, `Inverter_Power_Setpoint_Calc`, `BLE_Build_DeviceStatus_Response`, `Cloud_UploadTelemetryData`, `Config_LoadOrInit_DeviceSettings`, `Factory_Reset`, `Energy_TOU_Counters_Update`, `MQTT_Telemetry_String_Formatter`, `Quectel_UART_Send_Data`, `TLS_SSL_Read_WithTimeout`, the stubs; empty in E3: `CAN_Detect_Mismatched_Nodes`, `CAN_Parallel_Inverter_Sync`, `Inverter_ClearReg_0x03_Default` |
| missing in E3 | 31 | see §4.6 |
| only in E3 | 1 | `Inverter_RegDirty_Mark_BleState` (+ 4 renamed wrappers, §4.6) |

Six D names that the offline pairing had attached to the wrong E3 function were corrected after review (e.g. 0x0802B69C is
`Config_LoadOrInit_DeviceSettings`, not the removed `CAN_Send_ClearCmd_0xCB_DLC2`; 0x08009F48 sends 0xFB, i.e.
`Inverter_Set_Reg_0xFB`). Method: offline byte segments with masked pointers/BL offsets, entry-based pairing, name-normalised
instruction diffs and decompilation review of all changed and moved functions.

## 4. Functional differences to Venus D v151

### 4.1 Power limits and power selector

| Place | E3 151 | D 151 |
|---|---|---|
| `Config_Write_U16_0x202` (max charge power, EEPROM 0x202) | clamp 2500 W | 2500 W |
| `Config_Write_U16` (max discharge power, EEPROM 0x204) | clamp 2500 W | 2500 W |
| `Config_LoadOrInit_DeviceSettings` defaults | 2500 W | 2500 W |
| BLE cmd 0x15 / MQTT cd 0x0F / JSON-RPC "set version" | **600 W → mode 2 (new)**, 800 W → mode 1 (+ schedule clamp to 800 W), 2500 W → mode 0 | 800 → 1, **2200 → 3**, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–2500 W / ≤ 2500 W | same |
| Grid export limiter (`Grid_Export_Power_Limiter`, 4000 W clamp) | **not present** | present |
| `Inverter_Power_Setpoint_Calc` | no `Battery_Charge_Power_Limiter` step, no PV surplus add, no forced-charge clamp; remaining clamps: error bit 0, max discharge / −max charge, `Power_ScaledValue_Calc` limits, discharge-cutoff SOC, EPS reserve, 3-repeat debounce | full chain |

### 4.2 Battery / BMS

- Single-pack view: `BMS_CellVoltage_MinMax_PerPack`, `BMS_Version_Consistency_Check` (now only copies the pack-0 version),
  `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version` and the MQTT stack commands are gone (§6a).
- `CAN_CommLink_Timeout_DataClear` monitors only the BMS link (1) and the inverter link (2); the MPPT link 0 is skipped.

### 4.3 BLE and MQTT command differences

- **BLE** (`BLE_App_Command_Dispatcher` @ 0x0800C0E4): cmds 0x40 (`BLE_Apply_ConfigReg_Offset3`), 0x41 (surplus electricity),
  0x42 (BMS data response) and 0x43 are empty; cmd 0x03 status reply 0x8B bytes (`BLE_Build_DeviceStatus_Response` without PV1–4,
  PV capacity, MPPT version and pack-count fields); cmd 0x04 info string without `,mppt_v=%d`; develop-mode sub-code 5 does nothing.
  No command exists in E3 that the D lacks.
- **MQTT ctrl** (`MQTT_CtrlCommand_Dispatcher` @ 0x08022A20): cd 0x28 (stack BMS set), 0x29 (BMS version), 0x2A (BMS data),
  0x2B (surplus electricity), 0x2C (lock BMS), 0x2D (MPPT data) removed; `MQTT_Config_Command_Handler` drops the `fu=` key.
- **JSON-RPC** (`MQTT_JSON_RPC_Dispatcher`): method 5 (PV data) answers "Method not found".
- **Status/telemetry:** `MQTT_StatusReport_BuildAndPublish` drops the `,mppt=…,pv1..pv4,…,pack=…,fu=…,em=%x` block;
  `MQTT_Telemetry_String_Formatter` drops PV string, pack error summary and feed-in flags and sends 16 cell voltages of pack 0
  instead of the D's per-pack min/max statistics of 6 packs; `MQTT_Publish_PowerSchedule` publishes p1/p2 with up to 30 entries
  (D: p1–p7, 15 entries); the 1 s power histories keep grid and off-grid power only (29 samples each).

### 4.4 Cloud / HTTP

| Function | E3 151 | D 151 |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php?...&boot=%d&inv=%d` | same plus `&mpptv=%d` |
| `Cloud_UploadTelemetryData` (`{"d":"di=…"}`) | keys `pv, sk, mv, me, ma, fu` removed; record stride 0x227 | stride 0x35D |
| `Cloud_Report_FillDeviceInfo` | device name `VenusE 3.0` | `Venus D` |
| `Cloud_Report_FillGridData` / `FillPowerFlow` | PV powers removed (field written 0), battery power stored instead of PV sum | PV1–4 |
| `Energy_TOU_Counters_Update` | 2 counters (`"%d,%d"`) | 3 counters |

Domains, paths and the TLS code are unchanged; there is no E3-specific cloud endpoint.

### 4.5 Identity and OTA

- Runtime strings: `VNSE3-0` (`g_deviceModelTag`/`g_deviceTypeStr` = `VNSE` + `3-0`), `VNSE3-0 v151`, `MST_VNSE3_`, `VenusE 3.0`,
  device source id `VNSE3-0-<BLE MAC id>`.
- OTA / boot: `OTA_Is_VNSE3_Model` (D `OTA_Is_VNSD_Model`), `OTA_Slot_Config_Validate` and `OTA_FW_Verify_SetStatus` accept an image
  whose type string is a **substring of `VNSEE3-0`** (`strstr("VNSEE3-0", received)`, failure codes 0x401 CRC / 0x402 device mask /
  0x403 type unchanged). `ProcessFirmwareUpdateCommand` writes the boot descriptor with tag `VNSEE3-0` (version tag copied up to
  `-`, max 9 chars; D max 7). This is the "type check" mentioned in the E3 changelog; the same mechanism exists on D 151 with `VNSD-0`.
  The YMODEM update path (`YModem_*`, `CLI_FirmwareUpdate_Prepare`) is instruction-identical to D 151.
- "AI mode": `AI_ScheduleQuery_Response_Send` (cd 58, only when `g_userWorkMode == 5`) is present and identical to D 151.
- "CT 1.1 s": no E3-specific CT timing was found. The CT/meter code is identical to D 151; the only 1100-tick constant is the status
  telemetry interval in `Modem_/Ethernet_SendStatusTelemetry`, which D 151 has as well.

### 4.6 Functions missing in the E3 (compiled out) and E3 wrappers

| D 151 function(s) | Purpose on the D |
|---|---|
| `Key_ProtectionMonitor`, `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`, `Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot`, `CAN_ParallelInvLink_DebounceHold`, `Board_GPIO_Init_ControlPins` | key/protection relay chain on GPIOB.10/.11, GPIOE.15 (same set as missing on the A) |
| `Grid_Export_Power_Limiter`, `Grid_Export_Limit_Periodic_StateMachine`, `Surplus_FeedIn_Enable_Check`, `Battery_Charge_Power_Limiter` | export limiter and surplus feed-in |
| `Inverter_PVString_ChannelValue_PeriodicCheck_Save`, `Cloud_Report_FillPvStringDetails`, `MQTT_BuildPvDataResponse`, `Grid_UpdateMpptFaultLatch`, `MQTT_Publish_Inverter_Telemetry` | PV/MPPT |
| `BMS_CellVoltage_MinMax_PerPack`, `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version`, `BLE_Build_Cmd0x42_Response` | multi-pack reporting |
| `CAN_Broadcast_DateTime`, `Config_Write_Category0xC4`, `Config_Write_ParamPair`, `CAN_Send_ClearCmd_0xCB_DLC2`, `Inverter_Set_Reg_0x04`, `Inverter_ClearReg_0x03`, `BLE_Apply_ConfigReg_Offset3` | CAN commands not sent on the E3 (§6) |

The inverter "dirty register" bitmap has a different layout on the E3 (0x1C bits packed 7 per byte instead of 0x18 bits / 6 per byte,
`Inverter_RegDirty_Bitmap_SetClear`); the field wrappers were named after their E3 callers: `Inverter_RegDirty_Mark_BleState`
(0x1A, E3 only), `_GridPermit` (bit 2), `_MeterInvalid` (0x16), `_NetworkLink` (0x18), `_ChgVolt` (bit 4, D 0x14).
`Grid_BuildStatusFlagsReg` sends an all-zero 8-byte status frame to CAN id 0x1801FFAA (D: MPPT fault latch, PV over-voltage,
cut-off SOC).

## 5. Modbus

- **Read range:** `g_modbusRegDescTable` @ 0x20000340 (D 0x2000034C), 246 × 12 B; register numbers, types, scales, element sizes and
  counts are identical to D 151; 213 source pointers are 16 bytes lower, the rest follow the other RAM shifts (−444/−532/−533, +44,
  flash/none unchanged). The decoded CSV names the RAM global of the E3 program in the `notes` column. `FC03_Read_Handler` @ 0x080221B4
  and `Read_Serializer` @ 0x08053154 are instruction-identical, so the known D bugs (32101 sign, MBAP length of exception frames) apply.
- **Write range:** `Write_Handler` @ 0x08054254 instruction-identical (45000 command block, 46000 = 0x5100 debug mode → shell on RS485
  and TCP 8091, 46500–46544 schedule (CAN 0x30–0x3E), 47001 (CAN 0x18)).
- **E3-specific registers:** none. What differs is only the data source: registers that the D fills from the class-2 MPPT node
  (e.g. 30028, 30036, MPPT version) have no source on the E3 (no MPPT/PV hardware, the MPPT link is no longer monitored);
  per-pack blocks of packs that never report stay 0; register 30200 reads 151.

## 6. Internal CAN bus (Control side)

Arbitration id: `CAN_Build_Arbitration_ID` @ 0x08034F60 (function code bits 0–7, node 8–11, target class 12–15). 51 send sites
(D: 57). Receive side identical: `CAN_FrameDispatcher` @ 0x08031420 → source class 4 `Telemetry_Register_Dispatcher`,
class 2 `Modbus_ResponseDispatch`, else `Protocol_AA_CommandDispatch` @ 0x08031D04.

| Function codes built by the E3 Control | Notes |
|---|---|
| 0x01 power setpoint (`Inverter_Power_Setpoint_Calc`, 4 B) and on/off, 0x02 mode/power command, 0x03/0x04 u32 writes, 0x05, 0x06, 0x07, 0x08, 0x0A, 0x10–0x14, 0x16, 0x17, 0x18, 0x30–0x3E schedule (via `Write_Handler`), 0x40–0x43, 0x50–0x58, 0x60, 0xC1–0xC3, 0xCB (DLC4 only), 0xCE (node 1 fixed), 0xCF, 0xFB, 0xFE, 0xFF | same send functions as D 151 |
| **not sent any more** | 0xCC date/time broadcast, 0xC4 category writes, 0xCB DLC2, 0x04 DLC2 (`Inverter_Set_Reg_0x04`), 0x03 DLC2 clear |

The E3 micro (Micro 119, STM32G474, same family as the D micro) is the consumer of the class-4 commands; which of the codes it
actually handles will be confirmed by the E3 Micro analysis. The CAN date/time broadcast is gone, so the E3 micro gets no time from
the Control over CAN. `Register_WriteCategory0xCE` always addresses node 1 and `CAN_ParallelInverterDataParser` keeps a single value
per bank (no per-unit index).

### 6a. Pack capacity (standing check 6)

| Item | E3 151 | D 151 |
|---|---|---|
| `Protocol_AA_CommandDispatch` unit bound | `(id & 0xFF) < 9` (identical code) | `< 9` |
| Per-pack records | 8 × 0x60 B (`g_bmsPack<k>Voltage` @ 0x200177C4 + k·0x60; the 0x300-byte block is cleared on BMS link timeout) | 8 × 0x60 B (`g_bmsPack0Voltage` @ 0x200177D4) |
| Modbus per-pack blocks | 34000–34699, identical descriptors | same |
| Multi-pack evaluation | removed (cell min/max per pack, version consistency, MQTT BMS data/version/stack commands, BLE cmd 0x42) | present |
| Cells per pack | 16 (MQTT telemetry prints 16 cell voltages of record 0) | 16 |

So the Control still has room for 8 units but no longer evaluates more than one pack; how many packs the E3 BMS really reports
(`CAN_TX_PerPack_10Msgs` bound) is decided by the E3 BMS 115 analysis.

## 7. Tasks, shell, protocols

Identical to D 151 apart from addresses: task table `g_taskDefTable` @ 0x080584B4 (14 tasks, same priorities/stacks),
`shellCommandTable` @ 0x0805F9C4 (81 entries, user `VNSE3-0 v151` with empty password), `g_mqttRpcMethodTable` @ 0x0805B6DC
(16 methods), `mbedtls_cipher_definitions` @ 0x0805B834, CH395 sockets (502 Modbus TCP, 8091 shell, 8090 UDP).

## 8. EEPROM map

Same addresses as D 151 (`Config_LoadOrInit_DeviceSettings` reads/writes the same blocks: 1 identity, 0x200, 0x300, 0x900,
0x36BD, 0x500, 0x395, 0x400 Wi-Fi, 0x441 server type, 0x484/0x488/0x492 TOU counters). The identity block now holds `VNSE3-0`.

## 9. Strings of interest

`VNSE3-0 v151`, `VNSEE3-0` (OTA type tag), `MST_VNSE3_`, `VenusE 3.0`, `Build:\tSep 28 2026 09:39:27`,
`dev_type=%s, bin_type=%s`, the `getDateInfo` URL without `mpptv`, the shortened `{"d":"di=…"}` upload format. Vendor default Wi-Fi
credentials (`SM471`, …) and `quit_debug_5100_` are present as on the D.

## 10. Security notes

All Control findings of D 151 apply unchanged (identical code): Modbus 46000 = 0x5100 → unauthenticated shell on RS485 and TCP 8091,
shell user with empty password, vendor default Wi-Fi credentials, plain-HTTP `getDateInfo`. The OTA type check uses
`strstr("VNSEE3-0", received_type)`, i.e. any substring of the constant (also an empty string) passes; the same argument order exists on D 151.

## 11. Processing status in Ghidra

- Coverage: 0 undefined bytes, 0 orphan instructions in the flash block; 1927 functions, all named, all with a prototype
  (task entries `void f(void *pvParameters)`); 13 jump-table overrides, no unrecovered jump tables.
- `marstek_gates.py`: **all gates pass** (no code-as-data, no literal-only functions, no FUN_/tmp_/duplicates, 0 unnamed referenced
  RAM globals, task names match the task table, no-return flags sane); decompilation export 0 failures.
- Tags: `NAME_VERIFIED` on all 1927 functions (comment "Name, prototype and comment verified against the VNSE3 Control 151 code"),
  of which 94 additionally `NAME_LOW_CONFIDENCE`.
- RAM globals were carried over by instruction alignment of paired functions (majority vote) plus interpolation between anchors and
  checked against an offline pointer map; mismatches were corrected by hand (e.g. OTA error code slots, model/type string layout,
  telemetry history fields). Peripheral and XFMC labels were copied 1:1 (same hardware map). Comments with D-specific content
  (VNSD strings, D flash/RAM addresses) were rewritten for the E3.

## 12. Open questions

1. Which class-4 commands does the E3 micro (Micro 119) consume, and does it still expect the CAN date/time (0xCC) that the E3 Control no longer sends?
2. How many packs does the E3 BMS 115 report (per-pack sender bound)? The Control would accept up to 8 units.
3. Meaning of the reshuffled inverter dirty-bitmap fields (bits 2, 0x16, 0x18, 0x1A): names follow the E3 callers; the register behind each bit needs the E3 micro.
4. Why two model spellings (`VNSE3-0` at runtime, `VNSEE3-0` for OTA/boot)? This image's own footer (`VNSEE3`) passes the check, but an
   image tagged `VNSE3-0` would be rejected (`VNSE3-0` is not a substring of `VNSEE3-0`); whether other official E3 images all carry the
   `VNSEE3` footer is not verified.

## Appendix: method

Offline segment map of the two images (masked pointers and BL offsets), `ve3_layout_sync.py` / `ve3_port_segments.py` /
`ve3_port_signatures.py` / `ve3_copy_by_address.py` (peripherals only) / `ve3_port_refs.py` (flash targets only) /
`ve3_port_globals_aligned.py` / `ve3_ram_interp_port.py` to carry over layout, names, prototypes, references and RAM globals from D 151,
then review of all changed and moved functions, gap typing and the verification gates. Helper scripts are local only (not in the repo).
