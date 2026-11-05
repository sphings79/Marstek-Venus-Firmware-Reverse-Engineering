# Diff Table Control v150.8 → v150.9 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_Control_Diff_v1508_v1509_de.md](VNSA_Control_Diff_v1508_v1509_de.md)

**As of:** 2026-11-05. Image `1508_control_VNSA-0_app_1508_0819_182932.bin` ("A Control 1508", build string `Aug 19 2026 18:29:09`) compared with A Control 1509 (build string `Sep  8 2026 09:34:19`). Both are loaded in Ghidra with base 0x08004800 (1508: 1935 functions, 1509: 1941).
**Related documents:** [VNSA_Control_FW_v1508_Full_Analysis.md](VNSA_Control_FW_v1508_Full_Analysis.md), [VNSA_Control_FW_v1509_Full_Analysis.md](VNSA_Control_FW_v1509_Full_Analysis.md), [VNSA_Control_Diff_v150_v1509.md](VNSA_Control_Diff_v150_v1509.md), [VNSA_Control_Diff_v1509_v151.md](VNSA_Control_Diff_v1509_v151.md)

## Summary

**Direction:** 1508 is the older build (19 Aug 2026), 1509 the newer one (8 Sep 2026). "1508 has X" means: X is in 1508 and not in 1509 (and vice versa).

- 1508 identifies itself differently from 1509: device-type strings `VEPRO-0` / `VenusE Pro` / BLE prefix `MARS_VEPRO_`, version constant 0x5E4 (1509: `VNSA-0` / `Venus A` / `MST_VNSA_`, 0x5E5). This is only a fact from the code; its meaning is unclear (see "Device identity").
- 1508 lacks the changes that the earlier [v150 → v150.9 table](VNSA_Control_Diff_v150_v1509.md) attributes to 1509 alone: the six functions `OTA_ClassifyDeviceTypeTag`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18`, `ShellCmd_read_inv1`, `ShellCmd_read_inv2` do not exist in 1508 (1508 shell table: 79 entries, 1509: 81), and the YMODEM device-type check, the MQTT command block with `CAN_Write_Category0x18`, and Modbus write register 47001 are missing.
- Other differences (EPS reserve hysteresis, HTTP dispatcher, grid export limiter, telemetry builders, ...) are listed in the table; where only a difference was seen but its meaning was not established, the table says "unclear".
- No statement is made about effects on hardware or operation beyond what follows from the code.

## Method

Anchors: after masking addresses and call operands, **95.19 %** of the 1508 bytes (368,444 of 387,072) are identical to 1509 (452 blocks). Functions are paired via these anchors (1902 identical pairs, 20 partial pairs, 19 of the 1941 1509 functions without direct partner; 16 pairs in the reordered area were checked and corrected by instruction similarity). Then the ordered call targets of all 1935 pairs were compared: 21 findings, all inside functions that are changed anyway, none among the 1902 identical pairs (no false pairing); the changed functions were compared on normalised decompiler output.
Comment headers: the comment headers of the 1508 functions were ported from 1509 and contradicted the 1508 code in places (for example `BLE_Set_Device_Name` said `MST_VNSA_`). **Resolved (2026-11-05):** 16 plate comments were corrected against the real 1508 strings and now say "In 1509: ..." (`BLE_Set_Device_Name`, `BLE_Query_Device_Name`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `Cloud_Report_FillDeviceInfo`, `Config_LoadOrInit_DeviceSettings`, `Cloud_BuildDeviceSourceID`, `ShellCmd_get_ver`, `CLI_InitSession` and six RAM plates); the OTA model checks were correct and are unchanged. The export was regenerated afterwards. The code remains authoritative.

## Device identity (1508 vs 1509)

| Item | 1508 | 1509 |
|---|---|---|
| Build string (`ShellCmd_get_ver`) | `Aug 19 2026 18:29:09` | `Sep  8 2026 09:34:19` |
| `ems_type` (`ShellCmd_get_ver`), MQTT topics (`MQTT_Topic_Builder`), UDP discovery answer (`CH395_HandleBroadcastDiscovery`) | `VEPRO-0` | `VNSA-0` |
| Shell root entry (`CLI_InitSession`) | `VEPRO-0 v1508` | `VNSA-0 v1509` |
| Version constant (`g_deviceTypeCode`, `Config_LoadOrInit_DeviceSettings`; cloud query in `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher`) | 0x5E4 | 0x5E5 |
| `g_deviceModelTag` / `g_deviceTypeStr` init | words 0x52504556 + 0x302D4F (ASCII `VEPRO-0`) | `VNSA` + `-0`; additionally `g_deviceInfoReserved8aa = 0xA892`, `g_selLedEnable = 1` |
| Cloud device name (`Cloud_Report_FillDeviceInfo`) | `snprintf(out,0x14,fmt,"VenusE Pro")` | `memcpy(out,"Venus A",0x14)` |
| Cloud source id (`Cloud_BuildDeviceSourceID`) | `snprintf(buf,0x20,"%s-%s","VenusE Pro",mac)` | `sprintf(buf,"%s-%s","VNSA-0",mac)` |
| BLE name prefix (`BLE_Set_Device_Name`, `BLE_Query_Device_Name`) | `MARS_VEPRO_` | `MST_VNSA_` |
| OTA model check (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`) | compares against `"VNSA-0"` (code identical to 1509) | `"VNSA-0"` |

The archive folder and the file name say `VNSA-0`, while the 1508 strings say `VEPRO-0` / `VenusE Pro`. That "Venus E Pro" is meant is an **assumption** (name pattern only); there is no hardware proof, and the meaning of the identity change is unclear. Note: the OTA model check in 1508 still tests for `VNSA-0`, not for the 1508 device-type string.

## Changes 1508 vs 1509 (functions)

Addresses: 1508 entry (1509 entry in brackets where it differs). "unclear" = difference seen, meaning not established.

| Function | 1508 address | Change 1508 vs 1509 | Evidence |
|---|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` | 0800AC1C | Logic restructured. 1508: state machine on `g_epsReserveHysteresisState` (0/1/2). 1509: boolean latch `g_epsReserveLatched` with `g_epsReserveActiveFlag`, CAN status values 2/3, thresholds `soc < g_dischargeCutoffSocRaw+5`, `soc < 9 && g_epsReserveEnable == 1`. Semantic effect unclear. | decompiler diff (54 vs 45 lines) |
| `BLE_App_Command_Dispatcher` | 0800C5BC (0800C5C8) | 1509 has one more `EEPROM_Write` call (call 277 of 291). 1508 switch compiled with a `cmd-2 < 0x5a` range check and rebased case numbers (jump-table presentation; no difference in the command set shown). 1508 writes `g_quectelHwResetBackoff_off4 = 0` where 1509 writes `g_netCmdActivityTick = 0` (same slot, name from RAM sync). | call-target list, decompiler diff |
| `Quectel_AT_Recv_ByteAssembler` | 0801847C (08018484) | Buffer clear `memset(&g_atUdpRxLen, 0x406)` (1508) vs `0x408` (1509); 1509 has the extra field `g_atRxByteCounter`, so the assembler state flags sit at +2/+3/+4 (1508) instead of +4/+5/+6 (1509). Same state machine otherwise. | decompiler diff |
| `Quectel_SendCmd_WaitResponse`, `CH395_ExtractUdpData` | 0800B9D0, 0800BB48 | Only the same `memset` size 0x406 vs 0x408 and renamed globals. Code otherwise identical. | decompiler diff |
| `HTTP_ClientTask_Dispatcher` | 0801A284 (0801A294) | Substantially restructured: 136 vs 364 decompiled lines, same callees in a different order. 1508 has no `g_httpTimeSyncFastRetryFlag` / `g_httpTimeSyncPollTick` fast-retry handling and no `g_touReportAckPendingFlag` case in the compared part; no-link reboot logic differs (1508: `telemetryHistoryRecordCount > 2 && oldStep == 0` and a 0xE10 s timer; 1509: `< 5` check with `g_httpNoLinkRebootTick`). Completeness of the 1508 decompile of this function was not fully established: unclear. | decompiler diff, call-target list |
| `Cloud_UploadTelemetryData` (role name) | 0801B220 (0801B23C) | Older, shorter form (581 vs 700 decompiled lines); same call sequence. Name is by role (low confidence). | instruction similarity 0.334 |
| `Cloud_Report_FillDeviceInfo` | 08020F24 (08021018) | Device name `VenusE Pro` via `snprintf` instead of `memcpy` of `Venus A`. | decompiler diff |
| `MQTT_JSON_RPC_Dispatcher` | 0801F130 (0801F21C) | One `EEPROM_Write` call fewer in 1508; guard `if (len < 0x12)` around the `ble_mac` handling in 1508; cJSON hook addresses differ (relocation). | call-target list, decompiler diff |
| `Grid_Export_Power_Limiter` | 080233A0 (08023488) | 1508 has no `Power_ScaledValue_Calc_Negated` call at the start; clamps `power_hi` to 0x9C4 (2500) and negates `g_gridSamplePowerW` inline; 1509 uses the helper and has no 2500 clamp at this place. Upper clamp 0x960 (2400) is present in both. Effect on the feed-in limit: unclear. | call-target list, decompiler diff |
| `MQTT_CtrlCommand_Dispatcher` | 08023BE4 (08023D9C) | 1508 lacks 13 calls: a `strstr` + `MQTT_ConnectionStatus_Report_Send`, one `EEPROM_Write`, and a block with `strstr`, `atoi`, 4x `MQTT_Publish_CommandResult`, `MQTT_Publish_WorkMode_Config`, 2x `CAN_Write_Category0x18` (a command that exists only in 1509). The callees `MQTT_ConnectionStatus_Report_Send`, `MQTT_Publish_WorkMode_Config`, `CAN_Write_Category0x18` have no counterpart in 1508. | call-target list |
| `MQTT_StatusReport_BuildAndPublish` | 08029F88 (0802A350) | Same size (716 instructions); local variable layout and some ordering differ (ratio 0.57 decompiled, 0.84 by instruction). Content change: unclear. | decompiler diff |
| `DeviceInfo_BuildStatusString` | 0802AE1C (0802B1E8) | 1509 builds the hardware-variant tag strings per variant (HME-4, HME-3, SMR-0/1/2, TPM2-0); 1508 differs in this part (details unclear, ratio 0.81 / 0.46 decompiled). | decompiler diff |
| `Meter_ParseUDPMeterInfo` | 0802B838 (0802BC00) | 1508 adds `vTaskDelay((short)g_meterActiveCtType * 0x4b0)` and drops `g_modemRecvOkCount++`. | call-target list, decompiler diff |
| `Config_LoadOrInit_DeviceSettings` | 0802D66C (0802DA34) | One `EEPROM_Read` fewer (the `g_selLedEnable` read at 0x395; 12 vs 13 reads in the chained condition); identity as in "Device identity". | call-target list |
| `CT_GridPower_Controller` | 08030828 (08030C10) | 1508 lacks one `Inverter_Power_Setpoint_Calc(1,0)` call (the `g_epsReserveActiveFlag` branch). | call-target list, decompiler diff |
| `Modem_SendStatusTelemetry`, `Ethernet_SendStatusTelemetry` (role names) | 08031020, 080311BC | 1509 starts with `FreeRTOS_xTaskGetTickCount`, 1508 does not (one call fewer each); older form. | call-target list |
| `Grid_PowerSync_StateMachine` | 08031508 (0803199C) | Variable names/layout differ (`interval_ticks` vs `complete_timeout_s`, `last_deviation_channel`); algorithmic difference not established: unclear. | decompiler diff |
| `YModem_FirmwareUpgrade_Handler` | 080319A8 (08031E40) | 1508 lacks one `printf` (the "Bin type mismatch! Upgrade aborted" branch for result 0xFFFFFFFB). | call-target list, decompiler diff |
| `Protocol_AA_SetSystemParams` (role name) | 08033DCC (0803429C) | Much shorter in 1508 (9 vs 24 lines). | instruction similarity 0.424 |
| `YModem_ReceiveFile` | 08034E3C (08035328) | 1508 lacks `OTA_ClassifyDeviceTypeTag`, 3x `printf` and 2x `YModem_SendByte` (the device-type check of the received file exists only in 1509). | call-target list, decompiler diff |
| `CH395_Init_Modbus_TCP_Socket` | 0804D674 (0804DC1C) | 1508 does not call `CH395_ConfigTcpKeepalive_Defaults()`. | call-target list |
| `Write_Handler` | 08055EBC (08056960) | 1508 lacks one `CAN_Write_Category0x18` call; no literal 0xB799 (47001) and no own case 0xAFE6 (45030) in the 1508 decompile (present in 1509); CAN schedule register cache addressed as `g_canScheduleRegCache` / `...ValueHi` (+4 offset form in 1509). | call-target list, decompiler diff |
| `Cloud_PassiveMode_Connection_StateMachine` | 08015A3C (08015A28) | 1508 omits the `!g_ethLinkActive` precondition and uses `g_cloudPassiveModeRetryCnt` as raw variable; effect: unclear. | decompiler diff |
| `Quectel_AT_QVERSION_Query` | 0800FF58 (08010210) | Expected-answer string pointer differs (relocation); copy of the version words the same. | decompiler diff |
| `SMR_TIC_SelfUse_PowerController` | 0800851C | Only log line-number arguments differ (0x607/0x60C/0x616 vs 0x609/0x60E/0x618): source lines shifted by 2. No logic change seen. | decompiler diff |
| `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData` | 080123A8, 0801985C | Version constant 0x5E4 vs 0x5E5; buffer addresses relocated. | decompiler diff |
| `ShellCmd_get_ver` | 0803B0E4 | Build string, `ems_type` `VEPRO-0`, version 0x5E4 (see "Device identity"). | decompiler diff |
| `Cloud_BuildDeviceSourceID` | 08034580 (08034A74) | `snprintf(...,0x20,"%s-%s","VenusE Pro",mac)` vs `sprintf(...,"%s-%s","VNSA-0",mac)`. | decompiler diff, call-target list |

Additionally identified from the strings of the 1508 export (not part of the first comparison list, so not yet verified function by function against 1509): `BLE_Set_Device_Name` / `BLE_Query_Device_Name` use `MARS_VEPRO_` (1509: `MST_VNSA_`), `MQTT_Topic_Builder` and `CH395_HandleBroadcastDiscovery` use `VEPRO-0` (1509: `VNSA-0`), `CLI_InitSession` uses the root entry `VEPRO-0 v1508`.

Pure decompiler presentation or RAM relocation, no code change established: `Device_Network_Info_Init`, `BLE_Pending_Commands_Process`, `Util_RollingHistory_ShiftAndAverage_1s`, `Meter_GetMeterIP_FromResponse`, `MQTT_Publish_WiFi_Version`, `Board_GetUniqueID_HexString`, `debug_printf`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`.

### Only in 1509

`OTA_ClassifyDeviceTypeTag`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18`, `ShellCmd_read_inv1`, `ShellCmd_read_inv2`. Only in 1508: none.

## Verification checklist (standing checks)

| Question | Result 1508 vs 1509 |
|---|---|
| Modbus registers | The descriptor table (`RS485_Modbus_RegisterMap_Init`, `FC03_Read_Handler`) shows only presentation differences (global typing); a register-by-register comparison of the table contents was not performed: not checked. `Write_Handler`: 1508 has no write register 47001 (0xB799, CAN 0x18 via `CAN_Write_Category0x18`, 1509) and no own case for 45030 (0xAFE6); the 44003 limit text (rejection above 1500 W) is identical. |
| BLE | `BLE_Set_Device_Name` / `BLE_Query_Device_Name`: prefix `MARS_VEPRO_` (1508) instead of `MST_VNSA_` (1509), same `AT+QBLENAME=%s%c%c%c%c` format. `BLE_App_Command_Dispatcher`: one `EEPROM_Write` fewer; command set otherwise not shown to differ. Power classes/limits: not checked. |
| AC feed-in limit | Unchanged 1500 W in `Write_Handler` text (44003); `Grid_Export_Power_Limiter`: 1508 clamps the start value to 2500 W, upper clamp 2400 W in both. Effective feed-in limit through the 1508 changes: unclear. |
| BLE OTA | The BLE OTA functions (`BLE_Cmd_OTA_*`, `BLE_OTA_WriteDataChunk`) are present in 1508; pair comparison found no difference there, so BLE-OTA is still possible as far as the code shows (not tested on a device). YMODEM: 1508 lacks the bin-type check (`OTA_ClassifyDeviceTypeTag`, "Bin type mismatch!"). OTA acceptance still tests the model string against `VNSA-0` in 1508 (`OTA_Is_VNSA_Model`, code identical to 1509). |
| Cloud domains | Identical set in both: `marstekcloud.com/data-upload/v1/venus/`, `hamedata.com` endpoints (`getDateInfo`, `getDevicesApply`, `backup-configs`, `schedulings/policies`, FC41D `.rbl` OTA URL). Changed: device name/source id (`VenusE Pro` instead of `Venus A` / `VNSA-0`), version 0x5E4 vs 0x5E5 in the cloud query. |
| Pack capacity | Unchanged: `BMS_CellVoltage_MinMax_PerPack` scans 6 packs (`< 6`) and 13 cells (`< 0xd`) in both; `BLE_Pack_RegisterRecords` and `BLE_Build_BMS_Data_Response` show no difference after normalisation. The BMS-side limit (`CAN_TX_PerPack_10Msgs`) is not part of the Control image: not checked. |
| Non-functional registers | Register-level status comparison: not checked (47001 and 45030 exist only in 1509, see above). |

## Open points

- Meaning of the identity strings `VEPRO-0` / `VenusE Pro` / `MARS_VEPRO_`: unclear; "Venus E Pro" is an assumption without hardware proof. Needs confirmation by the reporter or vendor documentation.
- Effects of the EPS hysteresis rework, the `Grid_Export_Power_Limiter` clamp, the HTTP dispatcher restructuring and the telemetry builders are not established (marked "unclear").
- The 1508 jump-table labels of `HTTP_ClientTask_Dispatcher` were reconstructed (base shifted by 0x14 relative to 1509); the labels were not re-verified against the jump table of the image beyond the 7 entries.
- Comparison 1508 against the release v150 is not part of this document: not checked.
- The `Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry` and `Protocol_AA_SetSystemParams` pairs are role-named (low confidence).
- 111 1508 RAM names of the form `<object>_off<hex>` are address relations only, not semantics.
