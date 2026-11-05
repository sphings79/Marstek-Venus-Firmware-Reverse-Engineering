# Control FW v149 (Venus A) — Full Analysis (port from 150)

> 🇩🇪 Deutsche Version: [VNSA_Control_FW_v149_Vollanalyse_de.md](VNSA_Control_FW_v149_Vollanalyse_de.md)

**Status:** completed 2026-11-05. The image was analysed by porting the finished v150 analysis and verifying the port against the 149 code; limits and unchecked points are listed in §9.
**Binary:** `149_control_VNSA-0_app_0149_0528_102448.bin` (archive `Marstek FW Archiv/firmwares/VNSA-0/Control/149/`, "A Control 149")
**Port source:** Venus A Control v150, see [VNSA_Control_FW_v150_Full_Analysis.md](VNSA_Control_FW_v150_Full_Analysis.md). Everything that 149 shares with 150 (comparison with Venus D, Modbus, CAN, EEPROM, tasks, security notes) is documented there and is **not repeated** here.
**Differences 149 → 150 (function table):** [VNSA_Control_Diff_v149_v150.md](VNSA_Control_Diff_v149_v150.md)
**Ghidra:** `/VNSA/Control/Control_149_VNSA-0_app_0149_0528_102448.bin` (project `Marstek_Venus`)
**Decompilation:** private working export (not part of this repository): 1913 functions, 0 decompilation failures, plus a flash listing and `globals_and_registers.txt` (2589 named RAM/peripheral symbols), same layout as for 150.

---

## 0. Summary

- **Same code base as 150, but the older build.** After masking addresses and call operands, 92.24 % of the 149 bytes are identical to 150. 149 has 1913 functions, 150 has 1935: **23 functions exist only in 150**, one (`Base64Url_Encode`) only in 149.
- **149 is Venus A:** `VNSA-0` / `Venus A` / BLE prefix `MST_VNSA_`, version word 0x95 (149), build string `May 28 2026 10:23:00` (§4). There are no `VEPRO` / `VenusE` strings.
- **Main differences to 150** (§5): BMS pack record 0x58 instead of 0x60 bytes (150 adds SoH and MinPackVoltage), telemetry history slot 0x35b instead of 0x35d, no power clamp mode / grid power offset, no HTTPS upload (older HTTP + AES-128-ECB + Base64Url upload to `hamedata.com`), no no-link reboot watchdog, 7 MQTT-SSL configuration steps without SNI, SMR_TIC state machine inlined in `CT_PowerSetpoint_Compute`.
- **The RAM layout is not identical to 150** (`.bss`, external SRAM and `.data` are smaller, many globals sit at other addresses). This was the main source of work in the port (§3.3).
- **Verification:** all gates pass (`ALL GATES PASS`); tags: 1786 functions `NAME_VERIFIED`, 127 `NAME_LOW_CONFIDENCE` (§7).

## 1. Binary fingerprint

| | A 149 | A 150 (reference) |
|---|---|---|
| Size | 382 976 B (0x5D800) | 387 072 B (0x5E800) |
| SHA-256 | `f14fede53a07462ecbba7c37c0c1b24ac910a5962206a54f2ab21c62034ea07e` | `7dbff622f394a8262f9f545fbdabff2e815aa73122ce37f5b9412ca631702560` |
| Link address | **0x08004800**–0x08061FFF | 0x08004800–0x08062FFF |
| MCU | Nations **N32G45x** (Cortex-M4F) | same |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, compressed RW) | same |
| Build string (`ShellCmd_get_ver`) | `May 28 2026 10:23:00` @ 0x0803A550 | `Aug 11 2026 11:46:45` |
| Version | version word 0x95 (149) @ 0x08010000, followed by the bytes `VNSA-0` | version word 0x96 (150), followed by `VNSA-0` |
| Image footer | `VNSA` + 12 zero bytes @ 0x08061FE0, before it 0xFF fill (1108 B) from 0x08061B8C | `VNSA` @ 0x08062FE0 |
| Functions in Ghidra | 1913 | 1935 |

Base proof: the initial stack pointer (0x2001F778) and the reset vector (0x08004A71) at the start of the image resolve at base 0x08004800 (`Reset_Handler` @ 0x08004A70, Thumb), as in 150; the scatter table, string pointers and function pointers resolve at this base.

## 2. Memory map (Ghidra)

Same block structure as 150 (see [§2 of the 150 analysis](VNSA_Control_FW_v150_Full_Analysis.md)): bootloader below 0x08004800 (not in the image), application flash, `sysinfo` (UID @ 0x1FFFF7F0), `.data`, `.bss`, external SRAM `.xmc_bss` @ 0x60000000, peripheral blocks (names from the Nations N32G457 SVD, copied from 150: 977 labels).

Scatter table (`Region$$Table$$Base` @ 0x08060840, three entries, `scatter_region_t[3]`; decompressor `__decompress` @ 0x08005FBE):

| Region | Load address (flash) | Execution address | Size | Compressed image | Size in 150 |
|---|---|---|---|---|---|
| `.data` | 0x08060D50 | 0x20000000 | 0x2EEC | 2480 B (`Load$$RW_IRAM1$$Base`) | 0x2F00 |
| external SRAM (`.xmc_bss`) | 0x08061700 | 0x60000000 | 0x242E4 | 1164 B (`Load$$RW_XMC$$Base`) | 0x26AE0 |
| `.bss` (zero-init) | — | 0x20002EEC | 0x1C88C | — | 0x1C8E8 at 0x20002F00 |

Behind the table: `ShellCommand[78]` @ 0x08060870 (`shellCommandTable`). The FreeRTOS task table `TaskDef_t[14]` @ 0x080592E0 (label `g_taskDefTable`) lists 14 tasks; all names match their functions. The llhttp text blob @ 0x0805D35A is typed as char arrays split at the referenced addresses (58 pieces).

## 3. Method

### 3.1 Port from 150

1. **Anchors:** byte blocks of the 150 image that reappear in 149 with pointer and BL operands masked: 491 blocks, 353 276 of 382 976 bytes = **92.24 %** identical (1508 ← 1509 was 95.19 %). Direction: 149 is the older image, so "unmapped" source functions are functions that are new in 150.
2. **Function pairs** via these anchors: of the 1935 source functions, **1844 identical, 46 partially identical, 45 without direct counterpart** (1890 pairs). Layout (1851 functions created, 1628 removed in two windows), segments, signatures (including local variables) and flash/peripheral references (+1564) were carried over from 150; RAM references were deliberately **not** copied (§3.3); 23 flow overrides were applied (2 reset, among them a bogus computed jump at 0x08005EFC; 6 not applicable). After the auto-analysis the import alone had 1638 functions; the port ends at 1913.
3. **Corrected assignments in the reordered area:** the functions in 0x0800F9xx–0x08010xxx are re-ordered by up to 0x7F4 bytes, so address-based pairing is wrong there. 14 pairs were corrected by normalised instruction similarity (examples: 0x080102A0 is `Quectel_HTTP_ReadResponse_Verify`, not `Telemetry_History_Buffer_Init`; 0x0802A7E0 is `Cloud_ExtractConfigValue`, not `HTTPS_ParseData`; 0x0800AE4C is `Config_Read_SavedDateTime_0x160`, not `Config_Read_String_0x387`; 0x08032E64 is `CAN_AA_StoreUnitFieldPair80`, not `...Pair88`). Ten `FUN_*` functions were identified (`Stub_Return_One`, `Quectel_AT_SendCommand_Locked`, `Util_NoOp_Stub`, `Quectel_HTTP_AT_SendAndVerify`, `CAN_Detect_Mismatched_Nodes`) or given role names with low confidence (`Cloud_UploadTelemetryData`, `CT_PowerSetpoint_Compute`, `mbedtls_ssl_conf_endpoint`, `_transport`, `_own_cert`).
4. **Orphan code, gaps, switch tables:** 9 orphan instruction runs (248 instructions) and 72 undefined runs (20 289 B) were closed: 11 further functions (`Quectel_HTTP_ReadResponse_Trigger`, `Telemetry_History_Buffer_Init`, `BMS_Version_Consistency_Check`, the eight `mbedtls_ssl_conf_*` setters), zero padding as `byte[n]`, compressed blobs as `byte[]`. Typical defects: code typed as `uint` at case entries, 5-byte "strings" that were code, strings that started one byte early and swallowed the top byte of the preceding literal pointer, an "orphan" that was a string (`[HTTP] Upload data_len ...`), an orphan that was the jump-table bytes of the BLE dispatcher. **64 TBB/TBH jump tables** (62 found first, 2 more inside previously undisassembled code) are all typed as arrays; decompiler overrides were needed for 24 of them (the previously truncated functions and every table the decompiler could not recover itself). The three big dispatchers were truncated before (BLE_App_Command_Dispatcher 72 of ~2962 instructions, MQTT_CtrlCommand_Dispatcher 287 of ~2957, HTTP_ClientTask_Dispatcher 98 of ~899); now 6860 B, 6900 B and 2000 B (`HTTP_Task_StateMachine_Handler`: 5364 B), no decompiler warnings. Case labels versus tables: 0 mismatches over all 64 tables; 39 tables have no `jmp_*` labels because the decompiler recovered them itself.
5. **Call-target check:** the ordered call targets of all pairs were compared (see [the diff table](VNSA_Control_Diff_v149_v150.md), Method). 21 findings; three of them exposed **merge/naming errors of the port itself** (`Base64Url_Encode` + `Timer_Callback_BLE_Modem_Supervisor` at 0x08006A1C; `HTTP_ParseUploadResponseCode` + `CH395_SPI_SendData_Verified` at 0x0801973C with two strings disassembled as code; `Quectel_SSL_ConfigSNI` was really `Quectel_SSL_ConfigVerifyLevel`). After correction 19 findings remain, all inside genuinely changed functions.

### 3.2 Changed functions

43 pairs were compared on normalised decompiler output (type, cast and `&` noise removed). The result is the table in [VNSA_Control_Diff_v149_v150.md](VNSA_Control_Diff_v149_v150.md); 41 plate comments "Differs from 150: …" mark the functions in the program. Where a difference was seen but its meaning not established, the diff table says "unclear". The comparison ran before the RAM data types existed, which is why the first raw diffs were dominated by untyped globals and missing peripheral labels (`DAT_40010c00` instead of `GPIOB_PL_CFG`).

### 3.3 RAM layout shift against 150

The RAM layout of 149 differs from 150 (`.bss` 0x1C88C vs 0x1C8E8, external SRAM 0x242E4 vs 0x26AE0; base literals differ; the layout shifts by cluster, for example -0x10 in the HTTP task cluster and -8 around the SMR/tic variables). Consequences and handling:

- **Names by votes, not by address.** The first name sync created **1236 RAM labels** from votes of real references (reference from byte-identical code); an audit against the votes confirmed 1235 of them. Spot checks compared the `ldr` literal in the same function with the target (e.g. `g_bleConnState` @ 0x20018AEE = base literal 0x200188E0 + 0x20E). 506 of 1240 addresses have a reference backed directly by a literal word, the other 734 are struct fields reached via base literal + offset.
- **Conflicts:** 70 conflicting addresses and 4 skipped ones (names voted for two addresses) went into the gate-4 package: 282 referenced globals were unnamed. Result: 96 names from vote + 150 layout agreeing, 83 names from the 150 layout shift alone (plate `NAME_LOW_CONFIDENCE: 150 layout shift only`), 10 by hand, 96 `<object>_off<hex>` names (plate `NAME_LOW_CONFIDENCE: address relation only`). Three wrong first-sync labels (1 vote each) were deleted: `g_invState` (real address 0x20014E74), `g_quectelAtCmdBuf` (0x2000180D), `g_modemHwResetFlag` (0x20000FFA).
- **SMR_TIC inline:** `g_ctPowerSetpoint` = 0x20000290 (50 votes); 0x200002F8 `g_smrTicNewPowerW`, 0x200002FC `g_smrTicRefCtPowerW`, 0x20000300 `g_smrTicSettleCount`, 0x20000304 `g_smrTicMaxCtPowerW`, 0x20000308 `g_smrTicStateTick`, 0x2000030A `g_smrTicStatePeriodMs` (decompiled against the 150 state machine; plate `NAME_FROM_CONTEXT`). In 149 `ucHeap` is 8-aligned (block header at +0, size at +4), so the 150 sub-labels (`ucHeap_firstFreeBlock_xBlockSize` etc.) were placed from the `prvHeapInit` votes.
- **Never copy by offset, only by name.** Data types: 1394 RAM/XMC labels of 149 have a same-named 150 label with data; **1336** of them with a defined type (minus `*_off<hex>`) were typed like in 150 (type resolved, length checked against 150, 1179 comments copied where 149 had none). Before: undefined1/2/4; after: `uchar`/`ushort`/`uint`/`int`/`bool`/pointer/`char[n]`/`uint[256]` (FT/RT tables)/`ModbusRegDesc_t[246]` etc. 150 has no pack or telemetry struct types, so the 0x58 pack record and the 0x35b slot stay as scalars plus `_off` labels. Two objects with labels inside were typed with that allowed (`g_modbusRegDescTable`, 246 x 12 B ends exactly at the next label; `ucHeap`, 71 680 B ends exactly at `g_protocolTxBuf`).
- **Narrowing side effect:** typing the smaller 150 type over a wider 149 object orphaned 94 referenced bytes (pack records, dns1 bytes, mppt/can fields); they received `<base>_off<hex>` names (`g_dns1Addr_1..3` follow the pattern of `g_dns2Addr_0..2`). Gate 4 was re-run: 94 → 0.
- **Peripherals:** 977 labels (SYSINFO/UID, 0x40000000…, PPB 0xE000…) copied from 150 as `uint` with comments; `Board_GetUniqueID_HexString` decompiles with `UID_31_0`, `UID_63_32`, `UID_95_64`.

### 3.4 Pitfalls found (for later ports)

- Same entry address in both images is a coincidence, never evidence (0x08006A1C, 0x08010010). Check body size and strings of every pair, not only instruction similarity.
- Clone scripts of an older port (1508/1509 names, addresses) must be checked before they run; one stale clone with 1508 addresses was found and rewritten as a read-only check.
- Typing a smaller type over a wider object orphans referenced bytes: always re-run gate 4 after a type sweep.
- Plates copied from 150 can contain 150 addresses (filter on `0x[0-9a-f]{8}`).
- Clearing a broad range around a gap also deletes valid instructions at case entries; after any clear + disassemble re-run the switch-table disassembly and then the orphan-attach step.
- A script that exceeds the tooling timeout can still have finished (output written, state intact); check the state before re-running.

## 4. Device identity (facts from the code)

| Item | Value in 149 |
|---|---|
| Version word @ 0x08010000, then `VNSA-0`; footer `VNSA` @ 0x08061FE0 | 0x95 (149) |
| `ShellCmd_get_ver` | `ems_type=VNSA-0`, `ems_ver=0x95` (149), `Build: May 28 2026 10:23:00` (string @ 0x0803A550) |
| `VNSA-0` further | 0x08008CB0, 0x0800927C, 0x080094CC, 0x08009C8C, 0x08036CFC (UDP discovery answer), 0x0802CB40 (`Config_LoadOrInit_DeviceSettings`, `g_deviceTypeCode = 0x95`) |
| Shell root entry (`CLI_InitSession`) | `VNSA-0 v149` (0x08050924, 0x0805A8F5) |
| BLE name prefix | `MST_VNSA_` (0x0800EE58, 0x0800EEF0) |
| Cloud device name / source id | `Venus A` (0x080204C4) / `VenusA-%s` (0x080339AC) |
| `VEPRO` / `VenusE` strings | none |
| Cloud | `http://<region>.hamedata.com/...` (`getDateInfo`, `backup-configs`, `ems/getDevicesApply`, `schedulings/policies` over https, `prod/api/v1/setVenusDReporting`); further server types with `http://%s/api/v1/data`, `/v1/data`, `/v1/json`; no `marstekcloud.com` (150 posts to `https://api-<region>.marstekcloud.com`) |
| MQTT topics | `marstek_energy/%s/device/%s/ctrl`, `marstek/%s/server|device/%s/ctrl` (as in 150) |

## 5. Differences to 150 (summary)

Detail table with addresses and evidence per function: [VNSA_Control_Diff_v149_v150.md](VNSA_Control_Diff_v149_v150.md). In short:

- **Data layout:** BMS pack record 0x58 bytes (150: 0x60, adds SoH and MinPackVoltage); telemetry history slot 0x35b (150: 0x35d). Visible in the pack-related functions of the diff table (`BMS_CellVoltage_MinMax_PerPack`, `ShellCmd_get_all_bms_data`, `MQTT_Publish_BMS_Data`, the `CAN_AA_StoreUnit*` family, ...).
- **Functions and code only in 150 (23):** the HTTPS/TLS upload client (`HTTPS_FetchUrlData`, `HTTPS_PackPostRequest`, `HTTPS_ParseData`, `HTTPS_TLSSessionInit`, `HTTPS_TLS_Context_Reset`, `HTTPS_ParserOnBody`, `CH395_TCP_Socket_Open_HTTPS`, `mbedTLS_SSL_Recv/Send_WithRetry`, `mbedtls_ssl_close_notify`, `Quectel_HTTP_POST_SendAndVerify`, `Quectel_SSL_ConfigSNI`), the power clamp mode and string accessors (`Config_Read/Write_PowerClampMode_0x394`, `Config_Read/Write_String_0x387`, `MQTT_Publish_Grid_Status`), `SMR_TIC_SelfUse_PowerController` (inlined in 149), `CAN_AA_StoreUnitFieldPair88`, `CH395CMDGetSocketStatus`, `ShellCmd_FreeRtos_mem`, `FreeRTOS_xPortGetFreeHeapSize`, `FreeRTOS_xPortGetMinimumEverFreeHeapSize`.
- **Older forms in 149:** cloud upload (`Cloud_UploadTelemetryData`: AES-128-ECB + Base64Url to `hamedata.com`), `HTTP_Task_StateMachine_Handler` (GET + `HTTP_ParseUploadResponseCode`), `HTTP_FindHeaderBodySeparator` (`Connection:` + 0x15 instead of CRLFCRLF + 4), `Modem_/Ethernet_SendStatusTelemetry`, `Quectel_MQTT_Config_StateMachine` (7 steps, no SNI), `mbedTLS_SSL_Connection_Init` (no certificate load-failure logs).
- **Missing behaviour in 149:** the no-link reboot watchdog in `HTTP_ClientTask_Dispatcher`, the `g_powerClampConfig == 1` branch in `Inverter_PowerSetpoint_DeadbandClamp` / `Remote_Power_Setpoint_Process`, the `ct_dev=` key in `Cloud_HTTP_Response_Parser`, 20 calls in `MQTT_CtrlCommand_Dispatcher`, 8 calls and the "Get new power data" command in `BLE_App_Command_Dispatcher`.
- **Effect unclear:** the `g_meterSourceType` chain in `Meter_ParseUDPMeterInfo` and `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `BLE_Build_Cmd0x42_Response`.

## 6. What is shared with 150

For the following topics the 150 analysis applies to 149 **as far as the pair comparison shows identical code** (1844 identical pairs, byte-identical after masking); it was not re-derived for 149 function by function:

| Topic | See |
|---|---|
| Relationship to Venus D v150 (instruction-identical share, compile-time device constants) | [150 analysis](VNSA_Control_FW_v150_Full_Analysis.md), §3 |
| Power limits, power classes, 13 cells / 6 packs per pack scan, RS485 on USART1, functions compiled out, cloud / HTTP / network (older variants differ, see §5) | [150 analysis](VNSA_Control_FW_v150_Full_Analysis.md), §4 |
| Modbus read and write range, 44002/44003 limits (the `Write_Handler` pair is identical) | [150 analysis](VNSA_Control_FW_v150_Full_Analysis.md), §5 |
| Internal CAN bus (target classes, function codes) | [150 analysis](VNSA_Control_FW_v150_Full_Analysis.md), §6 |
| Tasks, shell, protocols, EEPROM map, strings, security notes | [150 analysis](VNSA_Control_FW_v150_Full_Analysis.md), §7–§10 |

Where 149 differs (cloud, BMS record, power clamp, EEPROM config block sizes), §5 above and the diff table apply.

## 7. Processing status in Ghidra

- Coverage: 0 undefined bytes, 0 orphan instructions (coverage tool: 1-2 instruction runs are dead branches after noreturn calls, attached to the function); 1913 functions, all named (0 `FUN_*`), all with a prototype; task names match the task table.
- Gate script (run in parts because of the script timeout of the tooling): **`ALL GATES PASS`** (1913 functions): no `FUN_*`, no duplicate names, no temporary names or labels, no placeholders, no default prototypes, no noreturn mismatches, no shell mismatch, no flow into data, no function pointer to data, no literal-only code, 0 duplicate global labels, 0 unnamed referenced globals, task table matches.
- Jump-table labels: 33 imported `switch`/`case_N` labels were source leftovers; 32 duplicated a Ghidra `switchD_*` label and were deleted; `case_15` @ 0x080559FA (`Write_Handler`) stays global exactly as in 150 (the table has only 15 entries, the address is a `nop` beyond it).
- Tags: **1786 `NAME_VERIFIED`, 127 `NAME_LOW_CONFIDENCE`** (1913 functions; exclusive, none with both tags, none without). Rule: a source `LOW` wins; forced to `LOW`: 12 functions from the low-confidence list (the `mbedtls_ssl_conf_*` setters, `CT_PowerSetpoint_Compute`, `Cloud_UploadTelemetryData`, ...), 28 pairs with instruction similarity < 0.9, and `Base64Url_Encode` (no source).
- Comments: 41 plates "Differs from 150: …"; 10 inherited plates corrected against the real 149 strings (build/version, 0x95, `v149`, EMS version, a watchdog sentence, MQTT configuration states, POST timeouts, heap size). 285 RAM names carry plates (`NAME_LOW_CONFIDENCE` or `NAME_FROM_CONTEXT`).
- Smoke test: 83 functions (all with "Differs from 150" plates, `Write_Handler`, the three dispatchers, a strided sample): 0 decompile errors, 0 raw `0x2000xxxx` / `0x6000xxxx` addresses in the decompiles.
- Export: 1913 functions, 0 decompilation failures, no file with "decompilation failed" or `halt_baddata`; 2589 named RAM/peripheral symbols in the listing.

## 8. Standing checks for a new firmware version (149 against 150)

| Question | Result |
|---|---|
| Modbus registers | `Write_Handler`, `FC03_Read_Handler` and `RS485_Modbus_RegisterMap_Init` are identical pairs; no 47001 (0xB799) and no own case 45030 (0xAFE6) in either; limit text of 44003 identical (rejection above 1500 W). `g_modbusRegDescTable` has the same type and size (246 x 12 B) as in 150. **Not checked:** the table contents entry by entry. |
| BLE | Name prefix `MST_VNSA_` as in 150. `BLE_App_Command_Dispatcher` has 8 calls fewer (power-offset and power-clamp writes, `String_0x387`), shorter status responses (0xa4 / 0x50 instead of 0xa7 / 0x51), no "Get new power data" command. **Not checked:** power classes and limits. |
| AC feed-in limit | `Grid_Export_Power_Limiter` is an identical pair: 2500 W start clamp, 2400 W upper clamp; 44003 text unchanged (1500 W). Without the power clamp mode / grid power offset (150 only) the setpoint path differs; effective feed-in limit: **unclear**. |
| Update path (BLE OTA) | The BLE OTA functions are present and identical pairs; BLE OTA is therefore possible as far as the code shows (**not tested on a device**). Model check against `VNSA-0`. `HTTP_FindHeaderBodySeparator` differs (`Connection:` + 0x15 bytes), see [OTA_HTTP_Body_Boundary_and_Diagnostics.md](OTA_HTTP_Body_Boundary_and_Diagnostics.md); consequences for the 149 HTTP download: **not checked**. |
| Cloud endpoints | `hamedata.com` set (`getDateInfo`, `backup-configs`, `getDevicesApply`, `schedulings/policies`, `setVenusDReporting`) plus further server types (`http://%s/api/v1/data`, `/v1/data`, `/v1/json`); no `marstekcloud.com`; no HTTPS upload path. Cloud TLS configuration: **not checked**. |
| Pack capacity | Control side: 6 packs (`< 6`) and 13 cells (`< 0xd`) in `BMS_CellVoltage_MinMax_PerPack`, as in 150; pack record 0x58 bytes (150: 0x60). The BMS-side limit (`CAN_TX_PerPack_10Msgs`) is not part of the Control image: **not checked**. Note: packs count from 1 in the BMS/CAN. |
| Non-functional registers (re-check) | **Not checked** at register level. |

## 9. Open points and limits

1. **About 190 RAM names `<object>_off<hex>`** (96 from the gate-4 package, 94 from the type sweep) are address relations with low confidence, not semantics; **83 further names** come from the 150 layout shift alone (plate `NAME_LOW_CONFIDENCE: 150 layout shift only`). No field names were derived for them by hand.
2. **72 RAM plates not copied:** plates of 150 that contain 150 RAM/flash addresses (e.g. `ucHeap`, `g_rtcSetDateBuf_*`, `g_otaErrCode*`) were left out because they would be wrong for 149; those objects have no plate. They were not rewritten by hand.
3. **Eight `mbedtls_ssl_conf_*` setters** (authmode, ca_chain, ciphersuites, curves, max_version, min_version, rng, sig_hashes) and `mbedtls_ssl_conf_endpoint` / `_transport` / `_own_cert` are named by position and size only (no 149 string or caller evidence). Callers in the SSL setup code could confirm them.
4. **Role-named functions:** `Cloud_UploadTelemetryData` (similarity 0.28), `CT_PowerSetpoint_Compute` (0.57, SMR_TIC inlined; a name such as `..._WithSmrTic` was considered and not used), `Modem_/Ethernet_SendStatusTelemetry`, `Base64Url_Encode` (no 150 partner; whether 150 has a Base64 encoder is open, its alphabet string is unreferenced). 127 functions carry `NAME_LOW_CONFIDENCE` in total.
5. **Pairs by similarity only:** `BMS_Version_Consistency_Check` (0.929) and `Telemetry_History_Buffer_Init` (0.842) are plausible older forms of the 150 functions, not proven identical in behaviour.
6. **Stray references:** the refcheck found no bogus reference (4446 consistent, 713 without a base literal, 0 inconsistent; 60 derived references far from a literal were kept after review), but RAM references were created by name sync, not copied; a few derived references could still be imprecise.
7. **Not checked:** the descriptor table entry by entry against 150 and against D; the BMS-side pack limit; the status of non-functional registers; BLE power classes and limits; cloud TLS configuration; a comparison of 149 against the betas 1508/1509 and v151; everything that depends on the micro and BMS firmware of the same release (this analysis covers the Control image only).
8. **Unclear effects:** see the "unclear" rows of the diff table (`g_meterSourceType` conditions, `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `BLE_Build_Cmd0x42_Response`, effect of the missing power clamp mode on the feed-in limit).
9. **Shared with 150:** open questions about the A micro (CAN frames, power classes) from the 150 analysis apply to 149 unchanged.

## Appendix: method

Offline anchor map of the two images (masked pointers and BL offsets) → function pairs → carry-over of layout, signatures, flash/peripheral references, flow overrides and comments from 150 → verification of the pairs by normalised instruction similarity and correction by hand in the reordered area → orphan code, gaps and jump tables → RAM name sync from votes of real references, then naming of the remaining referenced globals (layout shift, vote, context) → data types and peripheral labels by name → call-target, string and size comparison of all pairs → tags → comments → gates → export. Helper scripts and the working export are local only (not in the repository).
