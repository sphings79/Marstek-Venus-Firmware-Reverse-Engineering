# Diff Table Control v149 → v149.2 (Venus D)

> 🇩🇪 Deutsche Version: [VNSD_Control_Diff_v149_v1492_de.md](VNSD_Control_Diff_v149_v1492_de.md)

**As of:** 2026-09-30. Both images with correct base 0x08004800 in Ghidra, 100 % named (0 `FUN_`, 100 % byte coverage).
Addresses: v149 → v149.2. Function pairs via identical names (1931), instruction-wise (LCS) compared: **58 changed**
(of which **38 substantive**, 20 purely cosmetic), **0 only in v149**, **4 only in v149.2**; 1873 functions byte-identical.
No manufacturer changelog point except one (see below) — most statements from code.
**German:** `VNSD_Control_Diff_v149_v1492.md` · **Successor Diff:** `VNSD_Control_Diff_v1492_v150.md` · **Main Hop:** `VNSD_Control_Diff_v149_v150.md`

## Classification

v149.2 is a **Beta** between main releases v149 and v150 (timestamp: v149 = 2026-05-21,
v149.2 = 2026-08-12 per changelog, firmware file `1492_..._0702_142136.bin`). Beta step contains four
independent topics: extension of CAN 0xAA BMS pack record by a field (state-of-health), split of CT/SMR-TIC setpoint controller into two functions, new device alias name (EEPROM 0x387), and extensive restructuring of cloud/MQTT/HTTP telemetry chain.

## Checklist

| Question | Result |
|---|---|
| Modbus registers | **unchanged.** descriptor table 246 entries in both, register numbers/type/scaling/count identical verified (all 217 entries that differ in raw byte comparison differ exclusively in 4-byte source address field — consequence of relinking due to additional fields/functions, no register content changed) |
| BLE | Command set **numerically identical**, but payload of **0x18** ("set counter source") extended by one field: new string appended, saved via `Config_Write_String_0x387` as device alias in EEPROM 0x387/0x388. `BLE_Build_BMS_Data_Response` delivers actual SoH value (`g_bmsPack0Soh`) in previously fixed-to-0 reserve field. `BLE_Build_Cmd0x42_Response` differs only by unnamed target address (layout shift, no behavioral difference) |
| AC feed-in 2500 W | **unchanged.** `Grid_Export_Power_Limiter` and `Config_ClampDefaults_ParamBlock` byte-identical. `CT_GridPower_Controller` changes only call target (`CT_PowerSetpoint_Compute` instead of `SMR_TIC_SelfUse_PowerController`, see below) and two variable names, no clamp values |
| BLE-OTA | **continues to work.** `BLE_Cmd_OTA_Validate/Init/WriteSetup`, `YModem_FirmwareUpgrade_Handler` and all `OTA_*` byte-identical |
| Cloud domains | **unchanged.** identical `hamedata.com` endpoint set (`/app/neng/`, `/ems/api/v1/get`, `/external-services/api/v1/...`, `/prod/api/v1/set`) in both, only string addresses shifted |

## New Functions (only in v149.2)

| Function | Address | Purpose / Caller |
|---|---|---|
| `CAN_AA_StoreUnitFieldPair88` | 0x080334b4 | eighth field-pair handler for protocol 0xAA, offset 0x58 in now-96-byte pack record (see "BMS pack record" line below); dispatch table `g_canAaDispatchTable` in `Protocol_AA_CommandDispatch` extended from 16 to 17 entries |
| `CT_PowerSetpoint_Compute` | 0x080304e8 | takes over general (non-TIC) CT setpoint calculation including clamping, previously inline in `SMR_TIC_SelfUse_PowerController`; sole caller `CT_GridPower_Controller` (`CT_PowerSetpoint_Compute(delta_power, device_nb)`) |
| `Config_Read_String_0x387` | 0x0800b340 | reads 12-character device alias from EEPROM offset 0x387/0x388; caller `DeviceInfo_BuildStatusString` |
| `Config_Write_String_0x387` | 0x0800b63c | writes device alias (max 12 characters, null-terminated in 13-byte buffer `g_deviceAliasName`) to EEPROM 0x387; callers `BLE_App_Command_Dispatcher` (command 0x18) and `Cloud_HTTP_Response_Parser` (cloud config field `cd=18`) |

## Substantive Changes (instruction-verified)

| Area | v149 | v149.2 | Code Location (v149 → v149.2) | Ratio |
|---|---|---|---|---|
| BMS pack record: new SoH field | Record 0x58 B (88 B), 16-entry dispatch table (`Protocol_AA_CommandDispatch`: `cmp r5,#0x10`), no SoH field | Record **0x60 B (96 B)**, dispatch table **17 entries** (`cmp r5,#0x11`); new field pair at offset 0x58 (`CAN_AA_StoreUnitFieldPair88`): upper 16 bits = `g_bmsPack0Soh` (state-of-health, %), read by `BLE_Build_BMS_Data_Response`, `MQTT_Telemetry_Struct_Builder`, `ShellCmd_get_all_bms_data` ("bat soh ="); lower 16 bits (`g_bmsPack0Reserved58`) and second 32-bit field at 0x5c unused | `Protocol_AA_CommandDispatch` 08033754→080337ac; stride adjustment (0x2c→0x30 elements) in `BMS_CellVoltage_MinMax_PerPack`, `BMS_Version_Consistency_Check`, `CAN_Detect_Mismatched_Nodes`, `CAN_AA_StoreUnitFieldPair{0,8,16,24_Notify,64,72,80}`, `CAN_AA_StoreUnitIndexedFieldPair`; new `CAN_AA_StoreUnitFieldPair88` 080334b4 | 0.38–0.98 |
| CT/SMR-TIC setpoint controller split | Single function (974 B) does TIC state machine **and** general CT setpoint calculation/clamping inline; `CT_GridPower_Controller` calls it without arguments | Split: `SMR_TIC_SelfUse_PowerController` (590 B) now only does TIC state machine; new function `CT_PowerSetpoint_Compute` (see above) takes over setpoint calculation/clamping/application; `CT_GridPower_Controller` now calls `CT_PowerSetpoint_Compute(delta_power, device_nb)`. Pure restructuring, no function loss — calls moved, not removed | `SMR_TIC_SelfUse_PowerController` 08030074→08008654 (0.529), `CT_GridPower_Controller` 0802fe30→080302a4 (0.979) | 0.53 / 0.98 |
| Device alias name (new) | – | 12-character string in EEPROM 0x387/0x388 (`g_deviceAliasName`), writable via BLE 0x18 (appended field) and cloud config `cd=18`, readable in `DeviceInfo_BuildStatusString` | `BLE_App_Command_Dispatcher` 0800c31c→0800c758, `Cloud_HTTP_Response_Parser` 08028450→08028928, `DeviceInfo_BuildStatusString` 0802a5fc→0802ab44 (283→214 instr.) | 0.83 / 0.58 / 0.57 |
| MQTT BMS data packet | 517 instr. | 466 instr., restructured (including SoH field) | `MQTT_Publish_BMS_Data` | 0.586 |
| Cloud telemetry upload | 769 instr. | 785 instr., restructured | `Cloud_UploadTelemetryData` | 0.566 |
| Cloud HTTP response parser | 252 → 292 instr. | extended (including `cd=18` alias write path) | `Cloud_HTTP_Response_Parser` | 0.577 |
| MQTT status report | 752 → 755 instr. | restructured | `MQTT_StatusReport_BuildAndPublish` | 0.685 |
| HTTP task state machine | 2522 → 2589 instr. | extended; fine content beyond size change not fully traced instruction-wise | `HTTP_Task_StateMachine_Handler` | 0.691 |
| Ethernet status telemetry | 78 → 54 instr. | reduced | `Ethernet_SendStatusTelemetry` | 0.682 |
| UDP counter parse | 98 → 74 instr. | reduced | `Meter_ParseUDPMeterInfo` | 0.721 |
| Shell `get_all_bms_data` | 195 → 184 instr. | reduced; additional "bat soh =" output | `ShellCmd_get_all_bms_data` | 0.749 |
| MQTT control dispatcher | 2947 → 2960 instr. | restructured; fine content beyond size change not fully traced instruction-wise | `MQTT_CtrlCommand_Dispatcher` | 0.764 |
| Modem status telemetry | 125 → 101 instr. | reduced | `Modem_SendStatusTelemetry` | 0.796 |
| BLE command dispatcher | 2969 → 2985 instr. | command 0x18 extended with alias write path; remaining size change not fully traced instruction-wise | `BLE_App_Command_Dispatcher` | 0.827 |
| Config load/init | 380 → 384 instr. | adjusted | `Config_LoadOrInit_DeviceSettings` | 0.895 |
| MQTT telemetry struct | 273 → 276 instr. | field added (SoH) | `MQTT_Telemetry_Struct_Builder` | 0.965 |
| MQTT telemetry formatter | 358 instr. | field/format adjustment | `MQTT_Telemetry_String_Formatter` | 0.958 |
| MQTT BMS version | 72 instr. | adjusted | `MQTT_Publish_BMS_Version` | 0.903 |
| Telemetry history init | 20 instr., same size | restructured | `Telemetry_History_Buffer_Init` | 0.85 |
| Telemetry history push | 85 instr., same size | restructured | `Telemetry_History_Record_Push` | 0.859 |
| HTTP header/body separator | 20 instr., same size | 5 ops changed | `HTTP_FindHeaderBodySeparator` | 0.75 |
| CH395 UDP extraction | 134 instr., same size | adjusted | `CH395_ExtractUdpData` | 0.97 |
| HTTP GET URL data | 241 instr., same size | adjusted | `HTTP_GetUrlData` | 0.983 |
| HTTP client task dispatcher | 910 instr., same size | adjusted | `HTTP_ClientTask_Dispatcher` | 0.992 |

## Minor Changes (no behavioral difference)

**`__LINE__` shifts (log macro).** Following pairs differ only in **one**
`movw`/`adds` constant, passed as line number to logger (`Log_Printf`) — shifted through inserted source lines:
`AT_Copy_MPPT_Channel_Data`, `CH395_Socket_HandleStatusEvents`, `HTTP_Cloud_Reporting_Dispatcher`,
`HTTP_ParseData`, `HTTP_PackGetRequest`, `HTTP_ParserOnBody`, `HTTP_ParserOnMessageComplete`,
`Quectel_GetDeviceIP`, `Meter_GetMeterIP_FromResponse`, `Parse_IP_Address_String`,
`Device_Network_Info_Init`, `CAN_CommLink_Timeout_DataClear`, `ShellCmd_get_ver`, `__scatterload`
(scatter table).

**Literal pool/address shift through relinking.** No instruction behavior changed, only constants or
symbol addresses shifted by a few bytes because code/data inserted elsewhere in image:
`fp64_abs`, `pow` (both only shifted `undefined4` literals), `llhttp__internal__run` (one padding byte
becomes jump disassembly in v149.2, function size identical), `SPI_WriteByte` (one alignment fill byte). `MQTT_Transport_NullCallback` and `Stub_ImmediateReturn` are two 2-byte stubs (`bx lr`), located
directly adjacent in memory; apparent difference is only adjacent literal pool data
(firmware version string "VNSD-0", version number), allocated to one or the other function depending on
alignment — function body itself identical in both states (`return;`).

`BLE_Build_Cmd0x42_Response` differs only in that target field gets no symbol name in v149.2 (`UNK_20018f8d` instead of `g_bleCmd0x42Resp_calcVal0`) and address calculation takes more instructions — pure layout/naming shift, no behavioral change.

## Unchanged (explicitly verified)

Modbus completely (register map init, dispatcher, descriptor table 246 entries — only source addresses shifted in raw byte comparison), feed-in limits (`Grid_Export_Power_Limiter`, `Config_ClampDefaults_ParamBlock`
byte-identical), BLE command IDs, entire OTA/BLE-OTA/YMODEM path, all cloud domains
(`hamedata.com` endpoints), 1873 of 1931 paired functions byte-identical.

## Changelog → Code

| Changelog Point (v149.2) | Code Finding |
|---|---|
| Fixed erroneous data display from missed HTTP uploads | No single isolated fix found. Best fit to restructured HTTP/cloud telemetry chain (`HTTP_Task_StateMachine_Handler`, `HTTP_ClientTask_Dispatcher`, `Cloud_UploadTelemetryData`, `Ethernet_SendStatusTelemetry`, `Modem_SendStatusTelemetry`, `MQTT_StatusReport_BuildAndPublish`) — exact cause within these large functions not isolated, see Open Points |

## Open Points

- Exact cause of changelog point "Fixed erroneous data display from missed HTTP uploads" not isolated in code (candidate functions see changelog table).
- Meaning of unused halves of new BMS pack field pair at offset 0x58: `g_bmsPack0Reserved58`
  (lower 16 bits of first 32-bit value) and complete second 32-bit field at offset 0x5c
  (`g_bmsPack0Reserved5c` + upper halfword) are never read by any other function in v149.2.
- Fine content of large restructurings in `HTTP_Task_StateMachine_Handler`, `MQTT_CtrlCommand_Dispatcher` and
  `BLE_App_Command_Dispatcher` (2500–3000 instructions) beyond points documented in this file not fully traced instruction-wise.

## Data Basis

- Decompilat + listing of both states: `Dekompilate/VNSD_Control_149/`, `Dekompilate/VNSD_Control_1492/` (local only, not in the repo)
- Classification: complete name-paired function diff `scratchpad/v151/l149_1492.json` (local only, not in the repo)
- Modbus descriptors: `scratchpad/v151/mbdesc_149.json`, `scratchpad/v151/mbdesc_1492.json`
- Firmware changelog: `Marstek FW Archiv/firmwares/VNSD-0/Control/CHANGELOG.de.md` (local only, not in the repo)
