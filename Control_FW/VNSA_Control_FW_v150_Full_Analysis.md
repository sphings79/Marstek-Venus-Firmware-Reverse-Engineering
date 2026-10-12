# Control FW v150 (Venus A) — Full Analysis and Differences to Venus D v150 and A v150.9

> 🇩🇪 Deutsche Version: [VNSA_Control_FW_v150_Vollanalyse_de.md](VNSA_Control_FW_v150_Vollanalyse_de.md)

**Status:** completed 2026-10-11. All statements were checked against the code of this image; open points are listed in §12.
**Binary:** `150_control_VNSA-0_app_0150_0811_114724.bin` (archive `Marstek FW Archiv/firmwares/VNSA-0/Control/150/`, release v150)
**References:** Venus D Control v150 ([VNSD_Control_FW_v150_Full_Analysis.md](VNSD_Control_FW_v150_Full_Analysis.md)) — same code base and same RAM layout;
Venus A Control v150.9 ([VNSA_Control_FW_v1509_Full_Analysis.md](VNSA_Control_FW_v1509_Full_Analysis.md)) and v151 ([VNSA_Control_FW_v151_Full_Analysis.md](VNSA_Control_FW_v151_Full_Analysis.md))
**Diffs:** [VNSA_Control_Diff_v150_v1509.md](VNSA_Control_Diff_v150_v1509.md) (v150 → beta v150.9), [VNSA_Control_Diff_v150_v151.md](VNSA_Control_Diff_v150_v151.md) (v150 → v151)
**Ghidra:** `/VNSA/Control/Control_150_VNSA-0_app_0150_0811_114724.bin` (project `Marstek_Venus`)
**Decompilation:** `Dekompilate/VNSA_Control_v150/` (local only, not in the repo): 1935 functions, 0 decompilation failures, + `listing/` (flash listing, `globals_and_registers.txt`)

---

## 0. Summary

- **Release order of the A Control:** v150 (build 11 Aug 2026) → beta v150.8 (19 Aug) → beta **v150.9** (8 Sep) → v151 (28 Sep).
  v150.9 is therefore a beta *after* v150 on the way to v151, not a beta of v150. All diffs below run in this chronological direction.
- **A v150 is the Venus A build of Venus D v150.** 98.2 % of the image matches D v150 once pointers and BL offsets are masked (191 segments);
  `.data`, `.bss` and the external SRAM have the same sizes and every RAM global sits at the same address as on D v150 (initial SP 0x2001F7E8).
  The Modbus descriptor table is byte-identical to D v150. Of the 1954 D v150 functions, 1899 are identical in name and code,
  31 differ (A constants, RS485 on USART1, renamed callees), 5 carry the current A names (e.g. `OTA_Is_VNSA_Model`, `USART1_IRQHandler`),
  5 are 2-byte code-folding stubs merged into their handlers and 14 are compiled out (key/protection monitor, RS485 direction pin).
- **The A-specific differences are the same compile-time constants as in A v150.9/v151:** power limits 1500 W, power classes
  800/1200/1500 W, 13 cells per pack, USART1 without driver-enable pin, no key/protection monitor, `VNSA` identity, export limiter clamp 2400 W.
  One A-only difference that is not a constant: after a long cloud upload failure `HTTP_ClientTask_Dispatcher` **reboots the device**
  (`System_Reboot`), where D v150 only resets the CH395 Ethernet chip (§4.5).
- **Modbus (register-visible):** same 246 read registers as v150.9/v151 except **32102** (`float32`, 2 registers; `i16` from v150.9).
  `Write_Handler` has the A limit for **44003 (rejected above 1500 W)**, 44002 accepts up to 2500 W (raw value to the inverter, EEPROM copy
  clamped to 1500 W) — exactly as in v150.9/v151. Missing compared with v150.9/v151: write register **47001** (CAN 0x18) and the separate
  case for 45030 (0xAFE6, here reached through the 45000 block bounds path with the same effect).
- **Cloud:** telemetry upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`, `getDateInfo` and all other endpoints on
  `hamedata.com` (as D v150 and A v150.9). Device source id `VenusA-<serial>` (v150.9: `VNSA-0-<serial>`).
- **Compared with v150.9** 42 functions changed and 6 are new in v150.9 (name-paired, complete; details in the diff document): EPS reserve
  logic, YMODEM bin-type check, `cd=0x42` inverter debug info and `read_inv1/2`, write register 47001, heap 70 → 80 KB, connection watchdog,
  meter query rate limit, Modbus TCP keepalive, status report field `df`.

## 0a. Checklist (standing checks of the FW diff workflow, A v150)

| # | Question | Answer A v150 | Evidence |
|---|---|---|---|
| 1 | Modbus registers different? | vs D v150: **no** (descriptor table byte-identical, `Write_Handler` same code except the 44003 limit 1500 W). vs A v150.9/v151: **32102** is `float32` (2 registers) instead of `i16`; **47001** (0xB799) is not implemented (no case in `Write_Handler`); 44002/44003 behave as in v150.9/v151. Read path (`FC03_Read_Handler`, `Read_Serializer`) identical. Register 30200 reads the version word 0x96 = **150**. | §5 |
| 2 | BLE different? | Device name `MST_VNSA_` + 4 ID chars via the fixed format `AT+QBLENAME=MST_VNSA_%c%c%c%c`. Power classes and limits as v150.9 (cmd 0x15: 800 → 1, 1200 → 4, 1500 → 5; 0x16 300–1500 W; 0x17 ≤ 1500 W). Cmd `T` (0x54) still resets the EPS hysteresis (removed in v150.9). | §4.1, §4.6 |
| 3 | AC feed-in / power limits? | **1500 W** everywhere (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock`, MQTT `md=`, JSON-RPC ±1500 W, remote command ±1500 W, Modbus 44003). `Grid_Export_Power_Limiter`: PV start value first clamped to 2500 W (as D v150), upper clamp **2400 W** (D: 4000 W). | §4.1 |
| 4 | Firmware update via BLE possible? | **Yes.** OTA/BLE-OTA code identical to D v150 apart from the model check `VNSA-0` (`OTA_Is_VNSA_Model`). YMODEM has **no** bin-type check yet (added in v150.9). | §4.6 |
| 5 | Cloud domains? | Upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`; `getDateInfo`, `backup-configs`, `schedulings/policies`, `getDevicesApply`, FC41D OTA on `hamedata.com`; only server types 0–2 valid. Same as D v150 and A v150.9; `getDateInfo` moves to marstekcloud only in v151. | §4.5 |
| 6 | Pack capacity? | Control: 7 per-pack records (Modbus 34000–34699), `BMS_CellVoltage_MinMax_PerPack` scans **13 cells for packs 1..6** — unchanged up to v151. The pack limit itself sits in the BMS (A BMS 1105: `pack_id != 0 && pack_id < 7` = packs 1..6); no BMS image is part of this release. | §6 |
| 7 | Diff table | [VNSA_Control_Diff_v150_v1509.md](VNSA_Control_Diff_v150_v1509.md), [VNSA_Control_Diff_v150_v151.md](VNSA_Control_Diff_v150_v151.md) (both name-paired, complete). | — |
| 8 | Re-check of non-functional registers | The A register classification (FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL) was made on v150.9/v151. For v150 the reasons are the same code (descriptor table, `Write_Handler`, CAN senders/parsers, per-pack records) with two register-visible exceptions: 32102 is a 2-register `float32` here, and 47001 is not implemented instead of sending CAN 0x18 (which the A micro ignores anyway). No status changes from FUNCTIONLESS/NO_OP to FUNCTIONAL. | §5 |

## 1. Binary fingerprint

| | A v150 | D v150 (reference) |
|---|---|---|
| Size | 387 072 B (0x5E800) | 389 120 B |
| SHA-256 | `7dbff622f394a8262f9f545fbdabff2e815aa73122ce37f5b9412ca631702560` | `4140e7a6…1447` |
| Link address | **0x08004800**–0x08062FFF (18 KB bootloader below) | 0x08004800–0x08063FFF |
| MCU | Nations **N32G45x** with XFMC (Cortex-M4F) | same |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW) | same |
| Build string | `Aug 11 2026 11:46:45` | `Aug  5 2026 11:51:21` |
| Version | version word 0x96 (150) @ 0x08010000 followed by `VNSA-0`; string `VNSA-0 v150` @ 0x0805B73D | 0x96, `VNSD-0` |
| Image footer | `VNSA` + 12 zero bytes @ 0x08062FE0, 0xFF fill from 0x08062C4C | `VNSD` |
| Release note | identical text for A and D (local API in Ethernet mode, HTTP update in Ethernet mode, peak shaving, long HTTP records, CT_TYPE meter connection) | same |

Base proof: vector table at 0x08004800 with initial SP 0x2001F7E8 and Reset 0x08004A71 (Keil pattern `06 48 80 47 06 48 00 47`);
178 Thumb function pointers resolve to prologues at this base (4 at 0x08000000). `SystemInit` @ 0x080327A0.

## 2. Memory map (Ghidra)

| Block | Range | Contents |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | not in the image |
| flash | 0x08004800–0x08062FFF | application |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-bit UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EFF | RW initial values (LZ77 @ 0x08061DB4, 2492 B) |
| .bss | 0x20002F00–0x2001F7E7 | zero-initialised; `ucHeap` @ 0x20003144 (**70 KB**, v150.9/v151: 80 KB) |
| sram_unused | 0x2001F7E8–0x20023FFF | up to the N32G45x 144 KB limit |
| .xmc_bss | 0x60000000–0x60026ADF | external SRAM via XFMC (LZ77 image @ 0x08062770, 1244 B) |
| peripherals | apb1/apb2/ahb/bit-band/ppb | register names from the Nations N32G457 SVD (copied from A v150.9) |

Scatter table `Region$$Table$$Base` @ 0x08061894: .data 0x08061DB4 → 0x20000000 (0x2F00, `__decompress` @ 0x08005FBE),
XMC 0x08062770 → 0x60000000 (0x26AE0), .bss zero-init 0x1C8E8 @ 0x20002F00. Identical sizes and addresses to D v150; A v150.9/v151 have a
larger .bss (0x1F110) and XMC (0x26AF8), so every RAM global from about 0x20000140 upwards sits at a different address there.

## 3. Relationship to Venus D v150 and A v150.9 (method and numbers)

| Comparison (name-paired, complete) | Same | Changed | Only left | Only right |
|---|---|---|---|---|
| D v150 → A v150 | 1899 | 31 | 14 compiled out (§4.4) + 5 code-folding stubs + 5 renamed | 5 renamed (`OTA_Is_VNSA_Model`, `USART1_IRQHandler`, `Inverter_RegDirty_Mark_GridPermit`/`_PvVoltage`, `Util_NoOp_StartupHook`) |
| A v150 → A v150.9 | 1893 | 42 | 0 | 6 |
| A v150 → A v151 | 1887 | 48 | 0 | 7 |

"Changed" = different instruction sequence with call targets compared by name (literal pool addresses and RAM addresses masked).
Of the 42 changes to v150.9, 24 are behavioural, 3 only carry the version/identity and 15 differ only in relocated RAM offsets or log line
numbers. D v150 has five 2-byte "identical code folding" stubs (shared fault-handler loops, `__scatterload_copy` body) that A v150.9/v151
model as part of the handler; A v150 follows the A v150.9 model, so these are not counted as differences.

Method: offline masked segment maps against D v150 (98.2 %), A v150.9 (92.5 %), A v151 (91.2 %) and D v151 (90.5 %); layout, names,
prototypes, globals and references taken from D v150 (same layout), then names/prototypes/plates of all byte-identical functions from
A v150.9 for cross-version name consistency, RAM names of A v150.9 mapped by a name-anchored piecewise address map; every changed function
reviewed by hand (§4) and commented for v150.

## 4. Functional differences

### 4.1 Power limits and power classes (A v150 vs D v150; identical in A v150.9)

| Place | A v150 | D v150 |
|---|---|---|
| `Config_Write_U16_0x202` (max charge power, EEPROM 0x202) | clamp 1500 W | 2500 W |
| `Config_Write_U16` (max discharge power, EEPROM 0x204) | clamp 1500 W | 2500 W |
| `Config_ClampDefaults_ParamBlock` | charge 0/>1500 → 1500 W; discharge 0/>1500 → 800 W | 2500 W |
| `Config_LoadOrInit_DeviceSettings` first-boot default max charge | 1500 W | 2500 W |
| `BLE_App_Command_Dispatcher` cmd 0x15 power class | 800 → 1, 1200 → 4, 1500 → 5 | 800 → 1, 1500/2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–1500 W / ≤ 1500 W | 300–2500 W / ≤ 2500 W |
| `MQTT_Config_Command_Handler` `md=` default | 1500 W | 2500 W |
| `MQTT_JSON_RPC_Dispatcher` passive setpoint, power class | ±1500 W, A classes | ±2500 W |
| `Remote_Power_Command_Execute` | ±1500 W | ±2500 W |
| `Write_Handler` 44003 | rejected above 1500 W | above 2500 W |
| `Grid_Export_Power_Limiter` | start value ≤ 2500 W, upper clamp 2400 W; maintenance mode: old-step 1 or 5, power command mode 3 | upper clamp 4000 W; step 5, mode 1 |

### 4.2 Battery

`BMS_CellVoltage_MinMax_PerPack` scans **13 cells** for packs 1..6 (D: 16). `Battery_EPS_Reserve_SOC_Hysteresis` (same code as D v150)
is a three-state machine: EPS reserve active when SOC ≤ discharge cutoff, released at cutoff + 5 %; below 9 % SOC the EPS function is
switched off and persisted (EEPROM 0x300 = 0) and **switched on again automatically** (EEPROM 0x300 = 1) once SOC exceeds 8 %; the state is
reported as CAN frame 0xCF. BLE command `T` and MQTT `cd=0x38` reset this machine. v150.9 removes the automatic re-enable and the resets.

### 4.3 RS485 / serial port

USART1 (PA9/PA10, IRQ 37, 115200 8N1) without driver-enable pin — `RS485_UART_Config_Init` (A variant), `RS485_UART_Send`,
`RS485_UART_SendByte`, `Protocol_PutChar`, `USART1_IRQHandler`. D v150 uses USART2 on PC8/PC9 plus `RS485_DE_Pin_Init`/`RS485_Set_TX_Mode`/
`RS485_Set_RX_Mode`. Identical to A v150.9/v151.

### 4.4 Functions of D v150 missing in A v150 (compiled out)

`Key_ProtectionMonitor`, `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`,
`Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot`,
`CAN_ParallelInvLink_DebounceHold`, `Board_GPIO_Init_ControlPins`, `RS485_DE_Pin_Init`, `RS485_Set_TX_Mode`, `RS485_Set_RX_Mode` (14).
`WDT_SystemHealthService` therefore only runs the 500 ms board tick; `Inverter_Default_Init_Once` only sends `Inverter_Set_Flag_Reg_0x05(0)`.
Same set as in A v150.9 (which additionally lacks the D v151-only `MQTT_CtrlResponse_Transmit`).

### 4.5 Cloud / HTTP / network

| Function | A v150 | Remark |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://<g_cloudDomainUrlTable[t]>.hamedata.com/app/neng/getDateInfo<g_cloudGetDateInfoSuffixTable[t]>.php`, server types 0–2, `sv=150` | as D v150 / A v150.9 |
| `Cloud_UploadTelemetryData` | JSON POST to `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`, payload `{"d":"di=…&sn=…"}` | same code as D v150; rewritten in v150.9 |
| `HTTP_ClientTask_Dispatcher` | connection watchdog: more than one unsent telemetry record, old-step 0 and 0x708 (1800) elapsed → log `[HTTP]ch395 reset!!!!` and **`System_Reboot`** | **D v150: `CH395_Reset_And_Reinit(0)`**; v150.9: ≥ 5 records and 3600, v151: 14 400 + suppression flag |
| `HTTP_Task_StateMachine_Handler` | `AT+QHTTPPOST=%d,60,60` | as D v150 / A v150.9 |
| `CH395_Init_Modbus_TCP_Socket`, `CH395_ConfigTcpKeepalive_Defaults` | no keepalive on socket 502; keepalive idle 10 s for the other sockets | keepalive on 502 from v150.9, 60 s in v151 |
| `Modem_/Ethernet_SendStatusTelemetry` | meter query on every call (no rate limit) | v150.9: every 2000 ticks, v151: 1100 |
| `Meter_ParseUDPMeterInfo` | after a parsed 3-phase answer `vTaskDelay(ct_type × 1200)` | counter instead of delay from v150.9 |
| `Cloud_BuildDeviceSourceID` | `VenusA-<serial>` | `VNSA-0-<serial>` from v150.9 |
| `Config_LoadOrInit_DeviceSettings` | type code 0x96, no boot load of SEL LED (0x395) or server type (0x441) | SEL LED loaded from v150.9 |

### 4.6 Identity and other details

- Strings: `VNSA-0`, `VNSA-0 v150`, `MST_VNSA_`, `Venus A`, `VenusA-%s`, type code 0x96 (`g_deviceTypeCode`).
- `OTA_Is_VNSA_Model` and `OTA_Slot_Config_Validate` accept `VNSA-0` images only. YMODEM (`YModem_ReceiveFile`, `YModem_FirmwareUpgrade_Handler`)
  checks size and CRC only, no bin-type check of the file name.
- `CAN_Broadcast_DateTime`: second 0xCC frame with DLC 8 (D: 7).
- `Protocol_AA_SetSystemParams` copies the received work-mode pair without range check (v150.9 adds one).
- `MQTT_JSON_RPC_Dispatcher` keeps the SEL LED setting in RAM only (v150.9 also writes EEPROM 0x395).
- `MQTT_CtrlCommand_Dispatcher`: highest `cd` is 0x41; `cd=0x42` (inverter debug info) and the `cd=19` connection report come with v150.9.

## 5. Modbus

- **Read range:** `g_modbusRegDescTable` @ 0x20000354, 246 × 12 B, byte-identical to D v150. Compared with A v150.9/v151 only entry 32102
  differs (`float32`, 2 registers, instead of `i16`); the decoded table of v150.9 applies otherwise. `FC03_Read_Handler` @ 0x08023214 and
  `Read_Serializer` @ 0x08054DB8 identical to D v150 and A v150.9 (known D bugs: 32101 sign, MBAP length of exception frames).
- **Write range:** `Write_Handler` @ 0x08055EB8, same code as D v150 except the 44003 limit:

| Register | A v150 |
|---|---|
| 44002 max charge power | accepted up to 2500 W (exception 3 above); raw value to the inverter (`Inverter_Write_Reg_0x04_U32Value`) and to the RAM cache read back by 44002; EEPROM copy (0x202) clamped to 1500 W by `Config_Write_U16_0x202` |
| 44003 max discharge power | **rejected (exception 3) above 1500 W**; ≤ 1500 W to the inverter (`Inverter_Write_Reg_0x03_U32Value`), RAM cache and EEPROM 0x204 |
| 45030 (0xAFE6) | handled through the bounds path of the 45000 block (`Config_Apply_SingleReg` 11/12), same effect as the separate case of v150.9 |
| 47001 (0xB799) | **not implemented** (no case); v150.9/v151 send CAN 0x18 (`CAN_Write_Category0x18`) |

  The 45000 command block, 46000 = 0x5100 (debug mode → shell on RS485 and TCP 8091) and 46500–46544 (schedule, CAN 0x30–0x3E) behave as on the D.
  Ghidra could not recover three switch tables of `Write_Handler` automatically (also on D v150); they are set as decompiler jump-table overrides.

## 6. Internal CAN bus and packs

`CAN_Build_Arbitration_ID` @ 0x080370D0, `CAN_FrameDispatcher` @ 0x08033420, `Telemetry_Register_Dispatcher` @ 0x08033F48,
`Modbus_ResponseDispatch` @ 0x080337DC, `Protocol_AA_CommandDispatch` @ 0x08033D34 — all send sites identical to D v150 (only the DLC of the
0xCC date frame differs). Class-2/4 commands as listed in the v150.9 analysis (§6 there), without 0x18 (comes with 47001 in v150.9).
Pack capacity: 7 per-pack records in the Control, 13-cell scan for packs 1..6; the A BMS 1105 sends packs 1..6 (`pack_id != 0 && pack_id < 7`).

## 7. Tasks, shell, protocols

Task table `g_taskDefTable` @ 0x0805A108 (14 tasks, same as D v150), `shellCommandTable` @ 0x080618C4 (79 entries, user `VNSA-0 v150`
with empty password; `read_inv1`/`read_inv2` come with v150.9), MQTT-RPC table `g_mqttRpcMethodTable` @ 0x0805D2E8 (16 methods),
mbedTLS cipher table @ 0x0805D440, CH395 sockets (502 Modbus TCP, 8091 shell, 8090 UDP).

## 8. EEPROM map

Same addresses as D v150. Default limits written on first boot: max charge power 1500 W. SEL LED (0x395) and cloud server type (0x441)
are not loaded at boot.

## 9. Strings of interest

`VNSA-0 v150`, `MST_VNSA_%c%c%c%c`, `Venus A`, `VenusA-%s`, `Build:\tAug 11 2026 11:46:45`, hamedata `getDateInfo` URL (two variants),
`AT+QHTTPPOST=%d,60,60`, `[HTTP]ch395 reset!!!!`, marstekcloud data-upload URL. Vendor default Wi-Fi credentials (`SM471`, …) and
`quit_debug_5100_` as on the D.

## 10. Security notes

All Control findings of D v150 apply (identical code): Modbus 46000 = 0x5100 → unauthenticated shell on RS485 and TCP 8091, shell user with
empty password, vendor default Wi-Fi credentials, plain-HTTP `getDateInfo` to hamedata.com.

## 11. Processing status in Ghidra

- Coverage: 0 undefined bytes, 0 orphan instructions; 1935 functions, all named, all with a prototype.
- `marstek_gates.py`: **all gates pass** (no code-as-data, no literal-only functions, no FUN_/tmp_/duplicates, 0 unnamed referenced RAM globals,
  no DEFAULT prototypes, task names match the task table, no-return flags sane); decompilation export 0 failures, no unrecovered jump tables;
  short strings checked (all genuine NUL-terminated texts).
- Tags: `NAME_VERIFIED` on all 1935 functions, of which 98 additionally `NAME_LOW_CONFIDENCE` (same set as A v150.9).
- Names, prototypes and RAM global names are consistent with A v150.9/v151 (same object = same name); D v150 names that were renamed later
  (e.g. `Inverter_RegDirty_Mark_GridPermit`/`_PvVoltage`, `OTA_Is_VNSA_Model`) use the current names.

## 12. Open questions

1. Which CAN frames the A micro consumes is answered in the A micro 1211 analysis; whether an A micro of the v150 period behaved differently is unknown (no older A micro image analysed).
2. The release note of A v150 is identical to D v150; the D v150 analysis maps its five items to code changes of D v149.2 → v150. An A v149 baseline has not been analysed, so the mapping is assumed to apply to A as well.
3. 33 RAM names of A v150.9 have no A v150 counterpart (objects introduced in v150.9, e.g. `g_vns_inv_debug`); not an open gap, listed for completeness.

## Appendix: method

Offline segment maps (masked pointers and BL offsets), `vna150_layout_sync.py` / `vna150_port_segments.py` / `vna150_port_signatures.py` /
`vna150_copy_by_address.py` / `vna150_port_refs.py` from D v150, then name/prototype/plate porting from A v150.9 for byte-identical functions,
name-anchored RAM mapping from A v150.9, manual review of all changed functions and the verification gates. Helper scripts are local only (not in the repo).
