# Diff Table Control v150 → v151 (Venus E3)

> 🇩🇪 Deutsche Version: [VNSE3_Control_Diff_v150_v151_de.md](VNSE3_Control_Diff_v150_v151_de.md)

**As of:** 2026-10-14. Both images with the correct base 0x08004800 in Ghidra, fully named. Addresses: v150 → v151.
Function pairs via identical names (1920 pairs): 1871 byte-identical after masking pointers and BL offsets, 49 changed
(19 cosmetic, 30 with a real change), **0 only in v150**, **7 only in v151**. The comparison is name-paired and complete
(no Version-Tracking pairing); the changed functions were additionally compared on decompilation level, string literals and
descriptor/data tables. v150 has 1920 functions, v151 has 1927.
**Main documentation:** [VNSE3_Control_FW_v150_Full_Analysis.md](VNSE3_Control_FW_v150_Full_Analysis.md) · [VNSE3_Control_FW_v151_Full_Analysis.md](VNSE3_Control_FW_v151_Full_Analysis.md)
**Same step on the Venus D:** [VNSD_Control_Diff_v150_v151.md](VNSD_Control_Diff_v150_v151.md) — the E3 changes are the same set as on the D
(same code base); the E3 differences to that table are listed at the end of this file.

## Verification Checklist (standing checks)

| # | Question | Result |
|---|---|---|
| 1 | Modbus registers | Read table (246 descriptors, 30000–38014) identical except **32102** (`float32`, 2 registers in v150 → `i16`, 1 register in v151) and the RAM source pointers. Write side: new write register **0xB799 (47001)** with the new call `CAN_Write_Category0x18(value)`; the OTA-target select in the 46000 block (values 1234 / 2345 / 3456 / 4567 → `g_otaTargetDevice` 1 / 4 / 3 / 2) answers with status 2 in v150 but calls `CAN_Write_Category0x18(value)` and returns success in v151. The high-block registers 0xAFDF (`Inverter_Set_Reg_0xFE`) and 0xAFE6 (`Config_Apply_SingleReg` 11 / 12) exist in both versions, only compiled into different dispatch paths. Descriptor table moves 0x2000034C → 0x20000340. |
| 2 | BLE | Command set identical (same 600/800/2500 W selector, same 0x03/0x04/0x13 layouts); detail changes: cmd 0x0F and 0x54 no longer reset the EPS state variable (v151 has none), LED status (cmd 0x59) is persisted in EEPROM 0x395 in v151 (RAM only in v150), one schedule index bound check added. BLE error history 0x13 unchanged (20 × 14 bytes). |
| 3 | AC feed-in limit 2500 W | Unchanged: `Config_Write_U16_0x202`, `Config_Write_U16`, defaults 0x9C4; selector 600 / 800 / 2500 W in both. No grid export limiter on the E3 in either version. |
| 4 | Firmware update / BLE OTA | BLE OTA stays possible. v151 adds a YMODEM file-type check (`OTA_ClassifyDeviceTypeTag`, error −5 "Bin type mismatch! Upgrade aborted."); the `VNSEE3-0` type checks of the OTA paths and the HTTP OTA download path (`HTTP_FindHeaderBodySeparator`, `HTTPS_POST_ReceiveResponseData`, `OTA_WriteDataToRingBuffer`) are byte-identical. |
| 5 | Cloud endpoints | `getDateInfo` host changes from `http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php` to `http://api-<region>.marstekcloud.com/app/neng/getDateInfo<suffix>.php`; region table 3 → 5 valid values; all other cloud hosts unchanged (hamedata schedulings/policies/backup-configs, marstekcloud data upload, MQTT). |
| 6 | Pack capacity | Unchanged: `Protocol_AA_CommandDispatch` unit bound `< 9`, 8 per-pack records of 0x60 bytes (`g_bmsPack0Voltage` + k·0x60), 16 cells (MQTT telemetry prints the 16 cell voltages of record 0), Modbus per-pack blocks 34000–34699. The BMS-side sender bound is not part of this Control image (see the E3 BMS 115 analysis). |
| 7 | Diff table | This file. |
| 8 | Re-check of non-functional registers | No register changed state on the Control side except 32102 (v150: float32 reading 4 bytes of the i16 pair `g_batSamplePowerW` + `g_batSampleVoltage_0p1V`, i.e. a meaningless value; v151: exact alias of 30001). Reasons that live in Micro/BMS (receiver filters, missing senders) are not touched by a Control-only image. The new write path (47001, 46000 target select) sends CAN category 0x18; its receiver is the Micro (not re-checked here). |

## Changelog v151 → Code (Venus E3)

The vendor changelog of E3 v151 lists the same seven items as the Venus D; they map onto the following code changes (same functions as on the D, E3 addresses):

| Changelog item | Code finding (v150 → v151) |
|---|---|
| Modbus-TCP stability | `CH395_ConfigTcpKeepalive_Defaults`: IDLE 10 000 → 60 000 ms (CNT 5 and INTVL 3000 ms unchanged); `CH395_Init_Modbus_TCP_Socket` now applies the keepalive defaults first; `HTTP_ClientTask_Dispatcher`: CH395 hang watchdog (queued telemetry records > 1, 1800 ticks, `CH395_Reset_And_Reinit`) rebuilt to a tiered response (records ≥ 5, 14 400 ticks, then `System_Reboot()`, new suppression flag `g_httpWatchdogSuppressFlag`) |
| DOD cannot discharge due to abnormal status | `Battery_EPS_Reserve_SOC_Hysteresis`: 3-state machine with automatic re-enable (`g_epsReserveState`) replaced by a stateless variant that runs only at SOC ≠ 0 and never re-enables the EEPROM flag automatically; `CT_GridPower_Controller`: new guard resets a stuck positive setpoint to 0 while `g_epsReserveActiveFlag` is set; `MQTT_StatusReport_BuildAndPublish` appends `,df=%d` |
| YMODEM upgrade type check | `YModem_ReceiveFile` classifies the file name with the new `OTA_ClassifyDeviceTypeTag` (EMS / BMS / INV / MPPT) and aborts on type 0 or on a mismatch (two YMODEM CAN bytes 0x18, error code −5); `YModem_FirmwareUpgrade_Handler` gets the message "Bin type mismatch! Upgrade aborted." |
| AI mode check optimized / query current AI strategy | `AI_ScheduleQuery_Response_Send` (cd 0x3A): v150 formats all entries into one 1024-byte buffer without a per-entry overflow check; v151 sends page by page (`cd=%d,pg=%d/%d,`, 24 entries per page) through the new `MQTT_CtrlResponse_Transmit` |
| Pre-release server URL error (data upload error) | `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`: `getDateInfo` host `hamedata.com` → `api-<region>.marstekcloud.com`; the file suffix after `getDateInfo` is fixed `eu` in v151 (v150: from `g_cloudGetDateInfoSuffixTable`, indexed by server type); region table 3 → 5 valid values (`dev`, `staging` instead of `eu-dev`, `eu-staging`); `Config_LoadOrInit_DeviceSettings` loads the server type at boot |
| HTTP upload data stability optimized | `Quectel_HTTP_POST_SendAndVerify`: from two blind `vTaskDelay` calls to a real tick loop bounded by `timeout_s` with ERROR / `+QIURC` detection and an exact byte count (`UART_TransmitBytes`); `HTTP_Task_StateMachine_Handler`: `AT+QHTTPPOST=<len>,60,60` → `,120,120`, outer timeout 600 → 6000, body length now `g_cloudUploadBodyLen`; `prvHeapInit`: FreeRTOS heap 0x117F0 (70 KB) → 0x13FF0 (80 KB) |
| CT communication frequency optimized to 1.1 s | `Modem_SendStatusTelemetry` / `Ethernet_SendStatusTelemetry` now run at most every 0x44C = 1100 ticks (`g_statusTelemetryNextTick`); `Meter_ParseUDPMeterInfo` drops the blocking `vTaskDelay(g_meterActiveCtType * 1200)` after a valid sample and only counts `g_modemRecvOkCount` |

## Changes

| Area | v150 | v151 | Location (v150 → v151) |
|---|---|---|---|
| Version string / build | `VNSE3-0 v150`, `Aug  4 2026 15:11:03` | `VNSE3-0 v151`, `Sep 28 2026 09:39:27` | strings; `ShellCmd_get_ver` 08038960 → 08038F80 |
| Version constant (0x08010000, cloud URLs, device type code) | 0x96 (150) | 0x97 (151) | `HTTP_Cloud_Reporting_Dispatcher` 08011B60 → 08011BF0, `HTTP_GetUrlData` 08018CB8 → 08018D78, `Cloud_UploadTelemetryData` 0801A690 → 0801A754, `Config_LoadOrInit_DeviceSettings` 0802B28C → 0802B69C, `ShellCmd_get_ver` |
| `getDateInfo` host | `http://%s.hamedata.com/app/neng/getDateInfo%s.php` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php` (HTTP, port 80) | `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` |
| `getDateInfo` file suffix | `g_cloudGetDateInfoSuffixTable[type]` (`eu`, empty, `us`, …) | constant `eu` | same |
| Region table (EEPROM 0x441) | `eu`, `www`, `us`, `eu-dev`, `eu-staging`; valid 0–2 (BLE cmd 0x02 accepts 0–4 but `HTTP_GetUrlData` resets 3/4 to 0) | `eu`, `www`, `us`, `dev`, `staging`; valid 0–4 | `g_cloudDomainUrlTable`, `HTTP_GetUrlData` |
| Load server type at startup | on demand only | at startup (also LED flag, EEPROM 0x395) | `Config_LoadOrInit_DeviceSettings` |
| Device source ID (cloud) | `VenusE 3.0-<id>` | `VNSE3-0-<id>` | `Cloud_BuildDeviceSourceID` 08031DF8 → 0803235C |
| CH395 keepalive IDLE | 10 000 ms | 60 000 ms (CNT 5, INTVL 3000 ms same) | `CH395_ConfigTcpKeepalive_Defaults` 08034C2C → 0803524C |
| Keepalive before Modbus socket 502 | no | yes | `CH395_Init_Modbus_TCP_Socket` 0804AEF0 → 0804B510 |
| CH395 hang watchdog | records > 1, 1800 ticks, `CH395_Reset_And_Reinit`, log `[HTTP]ch395 reset!!!!` | records ≥ 5, 14 400 ticks, **`System_Reboot()`**, suppression flag | `HTTP_ClientTask_Dispatcher` 080196CC → 0801979C |
| HTTP POST over FC41D | two blind `vTaskDelay`, fixed 400-tick wait, no ERROR check, NUL-terminated send | tick loop, ERROR / `+QIURC`, exact byte count (`UART_TransmitBytes`) | `Quectel_HTTP_POST_SendAndVerify` 0800FBDC (same address) |
| `AT+QHTTPPOST` timeouts | `60,60`, outer 600 | `120,120`, outer 6000 | `HTTP_Task_StateMachine_Handler` 0801028C → 08010318 |
| FreeRTOS heap | 0x117F0 (70 KB) | 0x13FF0 (80 KB) | `prvHeapInit` 0804CFE0 → 0804D698 |
| Meter query (Shelly `EM.GetStatus`/`EM1.GetStatus`, others) | every call | at most every 1100 ticks, modem counters (AT fail / no response / no reply / send OK) | `Modem_SendStatusTelemetry` 0802EBA8 → 0802F010, `Ethernet_SendStatusTelemetry` 0802ED44 → 0802F200 |
| UDP meter response | blocking `vTaskDelay(g_meterActiveCtType * 1200)` after a valid sample | counter `g_modemRecvOkCount` | `Meter_ParseUDPMeterInfo` 080294FC → 0802990C |
| Quectel modem byte assembler | no call counter, status buffer 0x406 bytes | counter `g_atRxByteCounter` per call, buffer 0x408 bytes (also in `CH395_ExtractUdpData`, `Quectel_SendCmd_WaitResponse`) | `Quectel_AT_Recv_ByteAssembler` 08017900 → 080179A0 |
| Passive-mode UDP connection | requires `g_networkLinkUp` only | additionally requires `!g_ethLinkActive` | `Cloud_PassiveMode_Connection_StateMachine` 0801504C → 080150DC |
| YMODEM upgrade | no type check | file name → type (`OTA_ClassifyDeviceTypeTag`, new 0804C0F8), error −5 | `YModem_ReceiveFile` 080326AC → 08032C10, `YModem_FirmwareUpgrade_Handler` 0802F530 → 0802FA38 |
| AI strategy response (MQTT cd 0x3A) | one buffer, all entries, no paging | paged, `MQTT_CtrlResponse_Transmit` (new 08022520) | `AI_ScheduleQuery_Response_Send` 08022274 → 08022348 |
| Modbus write | register 46000 target select (1234 / 2345 / 3456 / 4567): sets `g_otaTargetDevice`, returns status 2; no 47001 | same select now also calls `CAN_Write_Category0x18(value)` (new 0802F484) and returns 0; new register **0xB799** (47001) jumps into that code. 0xAFDF and 0xAFE6 exist in both (different dispatch only) | `Write_Handler` 08053738 → 08054254 |
| MQTT `cd=0x42` "Get INV DEBUG info" | does not exist | sub-value `debug=N`: 1 = maintenance mode on (`g_maintModeState = 2`), 2 = `MQTT_Publish_WorkMode_Config` (new 08025E18), 3 / 4 = `CAN_Write_Category0x18(1)` / `(0)` | `MQTT_CtrlCommand_Dispatcher` 08022830 → 08022A20 |
| MQTT `cd=0x13` debug | – | sub-command `debug` → `MQTT_ConnectionStatus_Report_Send` (new 08022784: `ct_st, conn_st, err, wifi, send_ok, recv_ok, at_fail, no_resp, no_reply, sock, ip, port, reason`) | `MQTT_CtrlCommand_Dispatcher` |
| Shell commands | 68 commands (incl. `help`, `clear`), table 79 entries @ 0805ECD0 | 70 commands: new `read_inv1` (0805049C; maintenance mode like `cd=0x42,debug=1`), `read_inv2` (080504A8; prints `g_vns_inv_debug`), table 81 entries @ 0805F9C4 | `shellCommandTable` |
| CT setpoint stuck positive | – | setpoint → 0 while `g_epsReserveActiveFlag` | `CT_GridPower_Controller` 0802E488 → 0802E8D0 |
| EPS reserve hysteresis | 3 states (idle / latched / disabled), runs at SOC 0 %, re-enables EEPROM 0x300 automatically above 8 % | stateless, only at SOC ≠ 0, no automatic re-enable | `Battery_EPS_Reserve_SOC_Hysteresis` 0800A714 (same address) |
| BLE EPS on/off (0x0F), discharge limit (0x54); MQTT cd 0x0B, 0x38 | reset `g_epsReserveState` (0x54 / 0x38 also `g_epsReserveLatched`) | no longer | `BLE_App_Command_Dispatcher` 0800C0D4 → 0800C0E4, `MQTT_CtrlCommand_Dispatcher` |
| MQTT status report | – | new field `,df=%d` | `MQTT_StatusReport_BuildAndPublish` 08027D68 → 08028170 |
| Grid sync takeover | commit condition at exactly 2 deviating phases (`g_gridSyncDeviationCount == 2`) | at 2 or more (`< 2` negated), `g_gridSyncStatusCode = 2` set in one more branch | `Grid_PowerSync_StateMachine` 0802F090 → 0802F594 |
| Protocol-AA system parameters (CAN register 0x11) | one fixed value pair (two globals, 18 bytes of code) | index from `reg_id`, 7-slot table with bounds check (56 bytes) | `Protocol_AA_SetSystemParams` 08031864 → 08031DA4 |
| Economy TOU response (checksum) | one XOR checksum, one comparison | XOR checksum computed a second time into its own variable, either one accepted (mathematically identical); error message gets `crc2` | `HTTP_Economy_TOU_ParseResponse` 080187D8 → 08018880 |
| LED status (SEL LED) | RAM only | EEPROM 0x395 (BLE `Y`, MQTT, JSON-RPC, loaded at boot) | BLE / MQTT / JSON-RPC dispatchers, `Config_LoadOrInit_DeviceSettings` |
| Device-info reserved word | – | `g_deviceInfoReserved8aa = 0xA882` initialised in `Config_LoadOrInit_DeviceSettings` | purpose unknown |
| Modbus 32102 | `float32` (2 registers) | `i16` (1 register) | `g_modbusRegDescTable` |
| JSON-RPC / MQTT bounds | no explicit bound check before the dispatch switch | explicit `method < 0x12`, `cd < 0x43` guards (no functional change observed) | `MQTT_JSON_RPC_Dispatcher` 0801E58C → 0801E658, `MQTT_CtrlCommand_Dispatcher` |
| BLE switch | direct switch on the command byte | rebased index `cmd − 2 < 0x5A` (same cases) | `BLE_App_Command_Dispatcher` |
| BLE name format | `AT+QBLENAME=MST_VNSE3_%c%c%c%c` | `AT+QBLENAME=%s%c%c%c%c` with `MST_VNSE3_` (same result) | `BLE_Set_Device_Name` 0800EBDC → 0800EBD0 |
| Telemetry upload JSON | – | same keys (`pv/sk/mv/me/ma/fu` absent in both); version constant 0x96 → 0x97 | `Cloud_UploadTelemetryData` |
| Memory map | `.data` 12 024 B, `.bss` ends 0x2001F5DF, `.xmc_bss` 113 732 B | `.data` 12 016 B, `.bss` ends 0x20021DF7, `.xmc_bss` 113 752 B | scatter table 0805ECA0 → 0805F994 |
| Image size | 374 784 B (ends 0x0805FFFF) | 378 880 B (ends 0x08060FFF) | – |

## Cosmetic Differences (no behavioral change)

Byte differences caused only by relocation or register allocation (decompilation identical or differing only in local variable names):
`CH395_ExtractUdpData` (apart from the 0x408 buffer size), `Device_Network_Info_Init`, `Heap_Calloc`, `HTTPS_FetchUrlData`, `Meter_GetMeterIP_FromResponse`,
`MQTT_Publish_PowerSchedule`, `MQTT_Publish_WiFi_Version`, `Quectel_AT_QVERSION_Query`, `Quectel_AT_SendAndPollResponse`, `Quectel_GetDeviceIP`,
`Quectel_UDP_Socket_Open`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`, `Util_RollingHistory_ShiftAndAverage_1s`, `debug_printf`,
`BLE_Pending_Commands_Process` (pending-command table label only).
`BLE_Set_Device_Name`: only the format string was split (see table). `SMR_TIC_SelfUse_PowerController`: log line numbers (0x605 → 0x60C …) and a
variable renaming; the new-but-ineffective bounds guard of the D is not visible as a behavioral change here.

## Unchanged (explicitly verified)

Modbus read path (`FC03_Read_Handler` 080220E0, `Read_Serializer` 08052638), BLE error history 0x13 (`EventLog_*`), OTA slot diagnostics (`OTA_Set_SlotStatus`
codes 0x400–0x404), the `VNSEE3-0` type checks (`OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`,
`ProcessFirmwareUpdateCommand`), the HTTP OTA download chain (`HTTP_FindHeaderBodySeparator` → `strstr("\r\n\r\n") + 4`, `HTTPS_POST_ReceiveResponseData`,
`OTA_WriteDataToRingBuffer`, `HTTPS_POST_Request`), the 2500 W limits, the task table (14 tasks), `g_mqttRpcMethodTable` (16 methods), the CAN dispatch tables,
the Modbus descriptor table apart from 32102, task priorities and stacks, vendor default Wi-Fi credentials, debug register 46000 (0x5100).

## Only in v151 (7)

`CAN_Write_Category0x18` (0802F484), `MQTT_ConnectionStatus_Report_Send` (08022784), `MQTT_CtrlResponse_Transmit` (08022520),
`MQTT_Publish_WorkMode_Config` (08025E18), `OTA_ClassifyDeviceTypeTag` (0804C0F8), `ShellCmd_read_inv1` (0805049C), `ShellCmd_read_inv2` (080504A8).
v150 has no function that is missing in v151.

## Differences of the E3 to the Venus D table

The Venus D table lists the same changes with these deviations for the E3: no grid export limiter (the D row "PV start value when feeding" does not apply);
the E3 has no PV/MPPT code at all; the E3 passive-mode gating row and the `g_deviceInfoReserved8aa` value (0xA882, D: 0xA892) are E3-specific; the E3 heap is
70 → 80 KB like the D; `Meter_ParseUDPMeterInfo` and the status telemetry change exactly as on the D (D row "CT communication frequency").

## Open Items

- `Write_Handler`: the scalar constants of both functions were compared exhaustively (all immediates ≥ 0x100); apart from the items above only switch bases (the same registers are reached through differently compiled dispatch paths) and
  log line numbers differ. A complete branch-by-branch equivalence proof of the remaining branches was not done (medium confidence), as for the D.
- Exact meaning of `df=%d` and of `g_deviceInfoReserved8aa` (0xA882): not visible from the Control code alone.
- Whether `cd=0x42` / `read_inv1/2` map to a changelog item or are an unlisted addition is open (same as on the D).
- The E3 micro consumers of CAN category 0x18 (Modbus 46000 / 47001, MQTT `cd=0x42`) were not re-checked in this Control-only comparison.
