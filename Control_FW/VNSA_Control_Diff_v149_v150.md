# Diff Table Control v149 → v150 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_Control_Diff_v149_v150_de.md](VNSA_Control_Diff_v149_v150_de.md)

**As of:** 2026-11-05. Image `149_control_VNSA-0_app_0149_0528_102448.bin` ("A Control 149", build string `May 28 2026 10:23:00`) compared with the release A Control 150 (`150_control_VNSA-0_app_0150_0811_114724.bin`, build string `Aug 11 2026 11:46:45`). Both are loaded in Ghidra with base 0x08004800 (149: 1913 functions, 150: 1935).
**Related documents:** [VNSA_Control_FW_v149_Full_Analysis.md](VNSA_Control_FW_v149_Full_Analysis.md), [VNSA_Control_FW_v150_Full_Analysis.md](VNSA_Control_FW_v150_Full_Analysis.md), [VNSA_Control_Diff_v150_v1509.md](VNSA_Control_Diff_v150_v1509.md), [VNSA_Control_Diff_v1508_v1509.md](VNSA_Control_Diff_v1508_v1509.md)

## Summary

**Direction:** 149 is the older build (28 May 2026), 150 the newer one (11 Aug 2026). "149 has X" means: X is in 149 and not in 150; "150 has X" means: X is in 150 and not in 149. The tables below describe 149 as it is, and name the 150 behaviour for comparison.

- **Same code base, clearly older.** After masking addresses and call operands, 92.24 % of the 149 bytes are identical to 150 (150 itself is 387,072 B, 149 is 382,976 B). 149 has 1913 functions, 150 has 1935; **23 functions exist only in 150**, **1 only in 149** (`Base64Url_Encode`).
- **149 identifies itself as Venus A** like 150: `VNSA-0`, `Venus A`, BLE prefix `MST_VNSA_`; only the version constant differs (0x95 = 149 in 149, 0x96 = 150 in 150) together with the build string. There are no `VEPRO` / `VenusE` strings in 149 (unlike the later beta 1508).
- **BMS pack record:** 0x58 bytes per pack in 149, 0x60 bytes in 150. 150 adds the fields SoH and MinPackVoltage; the telemetry history slot is 0x35b bytes in 149 and 0x35d in 150.
- **No power clamp mode in 149:** the `g_powerClampConfig == 1` branch, `Config_Read/Write_PowerClampMode_0x394`, the `Config_Write_PowerOffset` calls in the command dispatchers and the grid power offset exist only in 150.
- **No HTTPS upload path in 149:** the cloud upload is the older HTTP form (AES-128-ECB + Base64Url payload via `HTTPS_POST_Request` to `hamedata.com`, `setVenusDReporting`). 150 posts a JSON payload to `https://api-<region>.marstekcloud.com` through its own HTTPS/TLS client functions (`HTTPS_FetchUrlData`, `HTTPS_TLSSessionInit`, `mbedTLS_SSL_Recv/Send_WithRetry`, ...), all of which are absent in 149.
- **No no-link reboot watchdog** in `HTTP_ClientTask_Dispatcher` (149); **7 MQTT-SSL configuration steps without SNI** (150: an additional last step `Quectel_SSL_ConfigSNI(1,0,1)`); the **SMR_TIC self-use state machine is inlined** in `CT_PowerSetpoint_Compute` (150: own function `SMR_TIC_SelfUse_PowerController`).
- Where only a difference was seen but its meaning was not established, the table says "unclear". No statement is made about effects on hardware or operation beyond what follows from the code.

## Method

Anchors: after masking addresses and call operands, **92.24 %** of the 149 bytes (353,276 of 382,976) are identical to 150 (491 blocks; the 1508 ← 1509 comparison had 95.19 %). Of the 1935 150 functions, the anchors gave **1844 identical pairs, 46 partial pairs and 45 without a 149 counterpart** (1890 pairs); after manual and instruction-similarity corrections (reordered area around 0x0800F900–0x08010600, orphan code runs that became functions) the final set is 1912 pairs plus the unpaired `Base64Url_Encode`.
Call-target check: the ordered call targets (`bl`, `blx`, `b.w`) of every pair were compared, with the 150 targets mapped through the pair list. First run: 21 findings. **Three of them exposed pairing/naming errors of the port itself** (the instruction-similarity test alone cannot see them):

| 149 address | Was | Is | Evidence |
|---|---|---|---|
| 08006a1c | `Timer_Callback_BLE_Modem_Supervisor` (522 B, two functions merged) | split: `Base64Url_Encode` 08006a1c–08006b53 (role name, low confidence, no 150 partner) + `Timer_Callback_BLE_Modem_Supervisor` 08006b58–08006c25 (pair 150 08006a1c, similarity 1.0, 80 vs 80 instructions) | size 206 vs 522 B; the Base64 alphabet string in a timer callback; callee of `Cloud_UploadTelemetryData` |
| 0801973c | `CH395_SPI_SendData_Verified` (152 B, two functions merged) | split: `HTTP_ParseUploadResponseCode` 0801973c–08019773 (pair 150 0801bdec, similarity 0.96; key `"de":` instead of `"code":`) + two strings (08019774, 0801977c; had been disassembled as code) + `CH395_SPI_SendData_Verified` 080197a4–080197d3 (pair 150 08019f40, similarity 1.0) | decompile showed `strstr`/`atoi` and the log string "Upload decrypt result"; caller `HTTP_ClientTask_Dispatcher` |
| 080145cc | `Quectel_SSL_ConfigSNI` (wrong name) | `Quectel_SSL_ConfigVerifyLevel` (pair 150 08014d90, similarity 1.0; string `AT+QSSLCFG="verify"`); 149 has no `Quectel_SSL_ConfigSNI` | string comparison; caller state 0 of `Quectel_MQTT_Config_StateMachine` |

After these corrections 19 findings remain (13 call-count differences, 6 swapped callee orders), **all inside functions that are changed anyway**. Three further read-only checks (string literals per pair, body size per pair, ASCII runs held as code over the whole image) found no other case. Rule learned: the same entry address in 149 and 150 is a coincidence, never evidence (the area is reordered by up to 0x7F4 bytes).
The changed functions were then compared on normalised decompiler output (casts, `&` and untyped RAM names ignored). That comparison ran before the RAM data types were applied (package P3b), so many raw decompiler differences were typing noise; they were filtered out. 43 pairs were compared, 41 plate comments "Differs from 150: …" mark the functions in the program.
The ported 150 comments contradicted the 149 code in 10 places (build string, version, `v149`, EMS version, a watchdog sentence, MQTT configuration states, POST timeouts); they were corrected against the real 149 strings (`ShellCmd_get_ver`, `Config_LoadOrInit_DeviceSettings`, `CLI_InitSession`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `prvHeapInit`, `HTTP_ClientTask_Dispatcher`, `Quectel_MQTT_Config_StateMachine`, `HTTP_Task_StateMachine_Handler`). Comments that are correct for 149 were left unchanged (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`, `BLE_Set_Device_Name`, `Cloud_BuildDeviceSourceID`, `Cloud_Report_FillDeviceInfo`).

## Device identity (149 vs 150)

| Item | 149 | 150 |
|---|---|---|
| Build string (`ShellCmd_get_ver`, string @ 0803a550) | `May 28 2026 10:23:00` | `Aug 11 2026 11:46:45` |
| Version word @ 08010000, followed by `VNSA-0`; footer `VNSA` @ 08061fe0 | 0x95 (149) | 0x96 (150) |
| `ems_type` / `ems_ver` (`ShellCmd_get_ver`) | `VNSA-0` / 0x95 (149) | `VNSA-0` / 150 (0x96) |
| `g_deviceTypeCode` (`Config_LoadOrInit_DeviceSettings`); cloud query in `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | 0x95 | 0x96 |
| Shell root entry (`CLI_InitSession`) | `VNSA-0 v149` (08050924, 0805a8f5) | `VNSA-0 v150` |
| `VNSA-0` further occurrences | 08008cb0, 0800927c, 080094cc, 08009c8c, 08036cfc (UDP discovery answer), 0802cb40 (`Config_LoadOrInit_DeviceSettings`) | same places |
| BLE name prefix (`BLE_Set_Device_Name`) | `MST_VNSA_` (0800ee58, 0800eef0) | `MST_VNSA_` |
| Cloud device name (`Cloud_Report_FillDeviceInfo`) | `Venus A` (080204c4) | `Venus A` |
| Cloud source id (`Cloud_BuildDeviceSourceID`) | `VenusA-%s` (080339ac) | `VenusA-%s` |
| `VEPRO` / `VenusE` strings | none | none |
| Cloud endpoints | `http://<region>.hamedata.com/...`: `getDateInfo`, `backup-configs`, `ems/getDevicesApply`, `schedulings/policies` (https), `prod/api/v1/setVenusDReporting`; further server types with `http://%s/api/v1/data`, `/v1/data`, `/v1/json`; **no `marstekcloud.com`** | `hamedata.com` endpoints as before; upload posts to `https://api-<region>.marstekcloud.com` |
| MQTT topics | `marstek_energy/%s/device/%s/ctrl`, `marstek/%s/server|device/%s/ctrl` | same |

## Changes 149 vs 150 (functions)

Addresses: 149 entry (150 entry in brackets where it differs). "unclear" = difference seen, meaning not established.

| Function | 149 address (150) | Change 149 vs 150 | Evidence |
|---|---|---|---|
| `Timer_Callback_BLE_Modem_Supervisor` | 08006b58 (08006a1c) | None; only the pairing was corrected (see Method). | similarity 1.0 |
| `Telemetry_History_Record_Push`, `Telemetry_History_Buffer_Init` | 080102f0 (08010990), 0800ff3c | History slot stride 0x35b (150: 0x35d), payload 0x34f (150: 0x351); the 150 overflow-slot fields are `_off<hex>` names in 149. | decompile |
| `Quectel_MQTT_Config_StateMachine` | 08012928 (08013064) | 7 steps (0..6): verify level, cipher suite, SSL version, MQTT SSL, MQTT version, data type, keepalive. 150 has an additional last step `Quectel_SSL_ConfigSNI(1,0,1)`. | decompile, calls |
| `BLE_Build_BMS_Data_Response` | 0800f66c (0800fb90) | The SoH slot is written 0 (150: `g_bmsPack0Soh`); no `g_bmsSelfCheckFlag` update. | decompile diff |
| `BLE_Build_Cmd0x42_Response` | 080146ac | Table values are read from other BMS record offsets (record layout, next row); effect unclear. | decompile |
| BMS pack record layout | many | Per-pack record 0x58 bytes in 149 (0x2c halfwords), 0x60 in 150 (0x30 halfwords); 150 adds the fields SoH and MinPackVoltage. Visible in `BMS_CellVoltage_MinMax_PerPack`, `ShellCmd_get_all_bms_data` (no `batsoh` / `batminvolt` lines; 2 calls fewer), `CAN_Detect_Mismatched_Nodes` (also no mismatch counter), `BMS_Version_Consistency_Check`, `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version`, `CAN_AA_StoreUnitFieldPair0/8/16/24_Notify/64/72/80`, `CAN_AA_StoreUnitIndexedFieldPair`. | strides in the decompiles |
| `Inverter_PowerSetpoint_DeadbandClamp`, `Remote_Power_Setpoint_Process` | 08017e50, 0802d1cc | No `g_powerClampConfig == 1` branch: the power clamp mode and grid power offset exist only in 150 (likewise `Config_Read/Write_PowerClampMode_0x394`, the `Config_Write_PowerOffset` calls in the two dispatchers and in `Config_LoadOrInit_DeviceSettings`). | decompile, calls |
| `HTTP_FindHeaderBodySeparator` | 080185e0 (08018dcc) | Searches `Connection:` and skips 0x15 bytes; 150 searches CRLFCRLF and skips 4. Log text differs ("get bin_start error"). | strings, decompile |
| `Cloud_HTTP_Response_Parser` | 08027e30 (08028bf8) | No `ct_dev=` key handling, no `Config_Write_String_0x387`, no default shelly port 0x3f2; 6 calls fewer. | calls, decompile |
| `DeviceInfo_BuildStatusString` | 08029fdc (0802ae24) | No `Config_Read_String_0x387` for the hardware-variant tag; buffer clear 0x2c (150: 0x32); the HME-/SMR-/TPM2 tags are built by word stores; rest unclear. | decompile |
| `Meter_ParseUDPMeterInfo`, `Modem_SendStatusTelemetry`, `Ethernet_SendStatusTelemetry` | 0802a87c, 080303bc, 08030594 | Extra `g_meterSourceType` chain (3, 4, 8, 9, 10, 11) setting `g_meterDataValidFlag`; callee order swapped (`Meter_ParseActivePower` / `MPPT_ParseATResponse`; `sprintf` / `DeviceInfo_BuildStatusString`). `Modem_`/`Ethernet_SendStatusTelemetry` are older forms (role names). Effect unclear. | calls, decompile |
| `Grid_PowerSync_StateMachine` | 0803091c | The `g_meterSourceType` condition sets differ; effect unclear. | decompile |
| `Cloud_EdgeDetectAndWatchdog` | 08030c8c | No `(g_mpptActiveChannel == 0 or 5)` qualifier on the `g_ctLinkEdgeState == 3` check. | decompile |
| `Config_LoadOrInit_DeviceSettings` | 0802c6ec | Type code 0x95 (150: 0x96); EEPROM config block 0x300 size 0x85 (150: 0x98), second block 0x36bf (150: 0x3700); no `g_powerClampConfig` default; one EEPROM read fewer. | decompile |
| `Quectel_HTTP_AT_SendAndVerify` | 08010010 (08010594) | One `vTaskDelay` fewer; log prefixes `[Rx..]` / `[Tx..]` without "Http". | calls, strings |
| `HTTP_Task_StateMachine_Handler` | 08010420 (08010ac0) | Older form: no `Quectel_HTTP_POST_SendAndVerify`, `Quectel_Modem_HardwareReset_Handler`, `Tick_Timer_Expired`; 2 `sprintf` and 2 `memset` fewer; instead `Quectel_HTTP_GET_Request`, `HTTP_ParseUploadResponseCode`, `MQTT_Publish_Telemetry`, 2x `strstr`; no `AT+QHTTPPOST` / header strings. | calls, strings |
| `HTTP_ClientTask_Dispatcher` | 0801995c (0801a270) | No no-link reboot watchdog (`System_Reboot`, `Tick_Timer_Check_Elapsed` absent); slot stride 0x35b; the upload result test is inverted; no Eco-Tracker log strings. | calls, decompile |
| `Cloud_UploadTelemetryData` (role name) | 0801a8d0 (0801b238) | Older form: `AES128_ECB_Encrypt` + `Base64Url_Encode` payload via `HTTPS_POST_Request` to `hamedata.com` `setVenusDReporting`; 150: `HTTPS_FetchUrlData`, `marstekcloud.com`, JSON payload. | calls, strings; similarity 0.28 |
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | 08019048, 08011c60 | Version constant 0x95 (150: 0x96); extra `http://%s/api/v1/data` etc. endpoints. | strings |
| `mbedTLS_SSL_Connection_Init` | 0801c438 (0801cf50) | One more `MQTT_TLS_Context_Reset` call; no CA / client certificate / private key load-failure log strings. | calls, strings |
| `MQTT_CtrlCommand_Dispatcher` | 080230c8 (08023bec) | 20 calls fewer: `MQTT_Publish_CommandResult` x5, `Log_Printf` x6, `strstr` x2, `atoi` x2, `Config_Write_PowerClampMode_0x394` x2, `Config_Write_PowerOffset` x2, `MQTT_Publish_Grid_Status`. | calls |
| `BLE_App_Command_Dispatcher` | 0800c1b4 (0800c5b8) | 8 calls fewer (`Config_Write_PowerOffset` x2, PowerClampMode x2, `String_0x387`, `BLE_Send_Response`, `Log_Printf`, `memcpy`); device status response length 0xa4 (150: 0xa7), BMS info response 0x50 (150: 0x51); no "Get new power data" command; command 0x16 without lower length bound. | calls, decompile |
| `CT_PowerSetpoint_Compute` (role name, low confidence) | 0802fa5c (08030a74) | The SMR_TIC state machine is inlined (150: own function `SMR_TIC_SelfUse_PowerController`); 16 calls more. Same caller `CT_GridPower_Controller`, same parameters (`ct_power`, mode); the six SMR_TIC variables are named from the 150 state machine (`g_smrTicNewPowerW` etc.). | calls, strings; similarity 0.57 |
| `ShellCmd_get_ver`, `CLI_InitSession` | 0803a4f4, 08050894 | Build `May 28 2026 10:23:00`, `ems_ver` 149 (0x95), `VNSA-0 v149`. | strings |
| `HTTP_ParseData`, `HTTP_ParserOnBody`, `HTTP_ParserOnMessageComplete` | | No logic change (callback addresses / bool vs int only). | decompile |
| `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `MQTT_Publish_BMS_Data` | | Same format strings; the remaining differences are typing noise, effect unclear. | decompile |

### Only in 150 (23 functions, no 149 partner)

`CH395CMDGetSocketStatus`, `SMR_TIC_SelfUse_PowerController`, `Config_Read_String_0x387`, `Config_Write_String_0x387`, `Config_Read_PowerClampMode_0x394`, `Config_Write_PowerClampMode_0x394`, `Quectel_HTTP_POST_SendAndVerify`, `Quectel_SSL_ConfigSNI`, `HTTPS_TLS_Context_Reset`, `mbedTLS_SSL_Recv_WithRetry`, `mbedTLS_SSL_Send_WithRetry`, `HTTPS_FetchUrlData`, `CH395_TCP_Socket_Open_HTTPS`, `MQTT_Publish_Grid_Status`, `HTTPS_PackPostRequest`, `HTTPS_ParseData`, `ShellCmd_FreeRtos_mem`, `CAN_AA_StoreUnitFieldPair88`, `HTTPS_TLSSessionInit`, `mbedtls_ssl_close_notify`, `HTTPS_ParserOnBody`, `FreeRTOS_xPortGetFreeHeapSize`, `FreeRTOS_xPortGetMinimumEverFreeHeapSize`.

### Only in 149

`Base64Url_Encode` (role name, low confidence; 150 keeps the Base64 alphabet string unreferenced).

Not "only in 150" although an earlier working note said so: `Quectel_SSL_ConfigVerifyLevel`, `HTTP_ParseUploadResponseCode` and `CH395_SPI_SendData_Verified` exist in 149 (see Method); `CAN_Detect_Mismatched_Nodes`, `BMS_Version_Consistency_Check`, `Quectel_HTTP_ReadResponse_Trigger` and the eight `mbedtls_ssl_conf_*` setters (authmode, ca_chain, ciphersuites, curves, max_version, min_version, rng, sig_hashes; names by position and size only, low confidence) also exist in 149.

## Verification checklist (standing checks)

| Question | Result 149 vs 150 |
|---|---|
| Modbus registers | `Write_Handler` is an identical pair (no 47001 / 0xB799 write register and no own case 45030 / 0xAFE6 in either; the 44003 text, rejection above 1500 W on Venus A, is the same). `FC03_Read_Handler` and `RS485_Modbus_RegisterMap_Init` are identical pairs (byte-identical after masking addresses and call operands), and the register descriptor table `g_modbusRegDescTable` has the same type and size (246 x 12 B) in 149 and 150. **Not checked:** the table contents register by register, and the register status classification (functional / non-functional). |
| BLE | Name prefix `MST_VNSA_` in both. `BLE_App_Command_Dispatcher` differs: 8 calls fewer (the power-offset and power-clamp writes, the `String_0x387` write), device status response 0xa4 (150: 0xa7), BMS info response 0x50 (150: 0x51), no "Get new power data" command, command 0x16 without lower length bound. `BLE_Build_BMS_Data_Response`: SoH slot written 0. **Not checked:** power classes and limits. |
| AC feed-in limit | `Grid_Export_Power_Limiter` is an identical pair: start value clamped to 2500 W (0x9c4), upper clamp 2400 W (0x960) in both; `Write_Handler` text for 44003 identical (1500 W). The power clamp mode / grid power offset (150 only) changes the setpoint path; effect on the effective feed-in limit: **unclear**. |
| Update path (BLE OTA) | The BLE-OTA functions (`BLE_Cmd_OTA_Validate`, `BLE_Cmd_OTA_Init`, `BLE_Cmd_OTA_WriteSetup`, `BLE_OTA_WriteDataChunk`) exist in 149; the pair comparison found no difference, so BLE OTA is possible as far as the code shows (**not tested on a device**). `OTA_Is_VNSA_Model` and `OTA_Slot_Config_Validate` test `VNSA-0`. HTTP download path: `HTTP_FindHeaderBodySeparator` differs (149 searches `Connection:` and skips 0x15 bytes; 150 searches CRLFCRLF and skips 4), see the cross-device reference [OTA_HTTP_Body_Boundary_and_Diagnostics.md](OTA_HTTP_Body_Boundary_and_Diagnostics.md); consequences for the 149 OTA download: **not checked**. `YModem_ReceiveFile` is an identical pair. |
| Cloud endpoints | 149: `hamedata.com` endpoints only (see "Device identity"), upload via `setVenusDReporting` with AES-128-ECB + Base64Url payload; no `marstekcloud.com`. 150: `marstekcloud.com` upload with JSON payload via an HTTPS client. Cloud TLS details: not checked. |
| Pack capacity | Control side unchanged: `BMS_CellVoltage_MinMax_PerPack` scans 6 packs (`< 6`) and 13 cells (`< 0xd`) in both; only the pack record stride differs (0x58 vs 0x60 bytes, 150 adds SoH and MinPackVoltage). The BMS-side limit (`CAN_TX_PerPack_10Msgs`) is not part of the Control image: **not checked**. |
| Non-functional registers (re-check) | Register-level status comparison: **not checked**. |

## Open points

- Effects of the `g_meterSourceType` changes, `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `BMS_StatusQuery_Response_Send`, `BLE_Build_Cmd0x42_Response` and the missing power clamp mode on the setpoint path are not established (marked "unclear").
- `Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry`, `CT_PowerSetpoint_Compute` and `Base64Url_Encode` are role-named (low confidence); whether 150 contains a Base64 encoder (its alphabet string is unreferenced) is not established.
- The BMS-side pack limit, the register table entry by entry, the cloud TLS configuration and the status of non-functional registers are not part of this comparison.
- 72 RAM plates of 150 that contain 150 addresses were deliberately not copied to 149; about 190 RAM names of the form `<object>_off<hex>` or layout-derived names have low confidence (address relation only). See [VNSA_Control_FW_v149_Full_Analysis.md](VNSA_Control_FW_v149_Full_Analysis.md), §9.
- The comparison of 149 against the later betas (1508, 1509) and against v151 is not part of this document: not checked.
