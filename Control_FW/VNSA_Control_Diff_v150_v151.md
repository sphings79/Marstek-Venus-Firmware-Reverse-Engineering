# Diff Table Control v150 → v151 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_Control_Diff_v150_v151_de.md](VNSA_Control_Diff_v150_v151_de.md)

**As of:** 2026-10-11. **Main documentation:** [VNSA_Control_FW_v150_Full_Analysis.md](VNSA_Control_FW_v150_Full_Analysis.md), [VNSA_Control_FW_v151_Full_Analysis.md](VNSA_Control_FW_v151_Full_Analysis.md)
Release history of the Venus A Control: v150 (`app_0150_0811`, 11 Aug 2026) → beta v150.8 → beta v150.9 (8 Sep 2026) → **v151** (28 Sep 2026).
Intermediate steps: [VNSA_Control_Diff_v150_v1509.md](VNSA_Control_Diff_v150_v1509.md) and [VNSA_Control_Diff_v1509_v151.md](VNSA_Control_Diff_v1509_v151.md).

**This version replaces the earlier derived diff.** The earlier version had to work without a correctly based A v150 program (masked byte
segments of v150 against v151, old behaviour taken from Venus D v150 → v151). A v150 is now fully analysed in Ghidra (1935 functions, all gates
pass), so this table is **name-paired and complete**: function pairs via identical names (1935), compared instruction by instruction with call
targets by name, referenced strings and immediate values (literal-pool and RAM addresses masked, the RAM layout differs). Old and new behaviour
are both verified on the A code. Corrections against the derived version are listed at the end.

Result: **48 changed** (25 behavioural, 4 version/identity only — `CLI_InitSession` is unchanged apart from its string — and 19 relocated RAM offsets/log
line numbers only), **7 only in v151**, **0 only in v150**.

## Verification checklist

| Question | Result |
|---|---|
| Modbus registers | Descriptor table (246 entries) identical except **32102**: type `float32` (2 registers) → `i16` (1 register); table moved 0x20000354 → 0x2000034C. `Write_Handler`: new write register **47001** (0xB799, CAN 0x18), own case for 45030 (0xAFE6, same effect as before); 44002/44003 unchanged (44003 above 1500 W rejected in v150 and v151; 44002 up to 2500 W, EEPROM copy clamped to 1500 W). Read path unchanged. All since v150.9. |
| BLE | `BLE_App_Command_Dispatcher`: cmd `T` no longer resets the EPS hysteresis; name format `AT+QBLENAME=%s%c%c%c%c` instead of the fixed `MST_VNSA_%c%c%c%c` (same name). Since v150.9. |
| AC feed-in limit | 1500 W in v150 and v151 (all clamps); `Grid_Export_Power_Limiter`: PV start value no longer clamped to 2500 W first (v150.9); upper clamp 2400 W unchanged. |
| BLE OTA | Still possible; YMODEM bin-type check added in v150.9 (`OTA_ClassifyDeviceTypeTag`, "Bin type mismatch! Upgrade aborted."). |
| Cloud domains | `getDateInfo`: hamedata.com → `api-<region>.marstekcloud.com`, server types 0–2 → 0–4 (v151); upload on marstekcloud.com in both; device source id `VenusA-<serial>` → `VNSA-0-<serial>` (v150.9). |
| Pack capacity | Unchanged: 7 per-pack records, 13-cell scan for packs 1..6 (`BMS_CellVoltage_MinMax_PerPack` identical). |
| Non-functional registers | 47001 becomes implemented (CAN 0x18, ignored by the A micro); 32102 changes type; no other register-visible change. |

## Changes v150 → v151 (behaviour)

Column "Since": v150.9 = already contained in beta v150.9 (function identical in v150.9 and v151 in this respect); v151 = new in v151. RN = item of the v151 release note.

| Function (v150 → v151 address) | v150 | v151 | Since | RN |
|---|---|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` 0800AC1C → 0800AC1C | three-state machine; below 9 % SOC EPS off (EEPROM 0x300 = 0), automatically back on above 8 % | hysteresis only at SOC ≠ 0, EPS flag no longer re-enabled automatically | v150.9 | 2 |
| `BLE_App_Command_Dispatcher` 0800C5B8 → 0800C5C8, `MQTT_CtrlCommand_Dispatcher` 08023BEC → 08023E90 | BLE `T` / MQTT `cd=0x38` reset the EPS hysteresis and `g_epsReserveActiveFlag`; highest `cd` 0x41 | reset removed; `cd=0x42` "Get INV DEBUG info" (new `MQTT_Publish_WorkMode_Config` 080275B8, `g_vns_inv_debug`), `cd=19` connection report (new `MQTT_ConnectionStatus_Report_Send` 08023BF4) | v150.9 | 2, 4 |
| `CT_GridPower_Controller` 08030830 → 08030D20 | same code as D v150 | setpoint update skipped while the EPS reserve is active (v150.9); positive CT setpoint reset to 0 (v151) | v150.9 + v151 | 2 |
| `MQTT_StatusReport_BuildAndPublish` 08029F90 → 0802A444 | status string without `df` | new field `,df=%d` | v150.9 | 2 |
| `YModem_ReceiveFile` 08034E38 → 08035448, `YModem_FirmwareUpgrade_Handler` 080319B0 → 08031F60, new `OTA_ClassifyDeviceTypeTag` 0804E924 | size/CRC check only | bin-type check of the file name, abort with "Bin type mismatch! Upgrade aborted." | v150.9 | 3 |
| `AI_ScheduleQuery_Response_Send` 08023630 → 080237B8, new `MQTT_CtrlResponse_Transmit` 08023990 | one 1024-byte answer `AI%d=…` | paginated answer (`cd=%d,pg=%d/%d,`, `AI%u=…`) | v151 | 4 |
| `ShellCmd_read_inv1` 08052CC4, `ShellCmd_read_inv2` 08052CD0 (new) | — | shell commands `read_inv1` (maintenance mode) and `read_inv2` (print `g_vns_inv_debug`) | v150.9 | – |
| `HTTP_Cloud_Reporting_Dispatcher` 08012394 → 08012424, `HTTP_GetUrlData` 0801984C → 08019910 | `getDateInfo` on hamedata.com, server types 0–2 | `api-<region>.marstekcloud.com`, server types 0–4, prefixes `dev`/`staging` | v151 | 5 |
| `Config_LoadOrInit_DeviceSettings` 0802D670 → 0802DB28 | SEL LED (0x395) and server type (0x441) not loaded at boot | SEL LED loaded (v150.9); server type loaded, first-boot default 0x395 = 1 (v151) | v150.9 + v151 | 5 |
| `MQTT_JSON_RPC_Dispatcher` 0801F144 → 0801F2D0 | SEL LED setting in RAM only | also written to EEPROM 0x395 | v150.9 | – |
| `Quectel_HTTP_POST_SendAndVerify` 08010438, `HTTP_Task_StateMachine_Handler` 08010AC0 → 08010B4C, `HTTPS_FetchUrlData` 0801BB88 → 0801BD1C, `HTTP_Economy_TOU_ParseResponse` 0801936C → 08019418 | `AT+QHTTPPOST` 60/60 s, outer timeout 600, one TOU checksum | HTTP POST rewrite, 120/120 s, outer timeout 6000, TOU `crc2` | v151 | 6 |
| `Cloud_UploadTelemetryData` 0801B238 → 0801B300 | payload builder of D v150 | rewritten payload builder (v150.9), body length handling as D v151 (v151) | v150.9 + v151 | 6 |
| `prvHeapInit` 0804F764 → 0804FEC4 | heap 70 KB | 80 KB | v150.9 | 6 |
| `Quectel_AT_Recv_ByteAssembler` 0801846C → 08018510 | — | received-byte counter | v150.9 | – |
| `CH395_ConfigTcpKeepalive_Defaults` 080373BC → 08037A84 | keepalive idle 10 s | 60 s | v151 | 1 |
| `CH395_Init_Modbus_TCP_Socket` 0804D674 → 0804DD3C | no keepalive on Modbus socket 502 | keepalive defaults also for socket 502 | v150.9 | 1 |
| `HTTP_ClientTask_Dispatcher` 0801A270 → 0801A344 | > 1 unsent record, old-step 0, 1800 elapsed → log `[HTTP]ch395 reset!!!!` + `System_Reboot` | ≥ 5 records and 3600 (v150.9); 14 400 and suppression flag `g_httpWatchdogSuppressFlag` (v151); always `System_Reboot` | v150.9 + v151 | 1 |
| `Modem_SendStatusTelemetry` 08031028 → 08031538, `Ethernet_SendStatusTelemetry` 080311C4 → 08031728 | meter query on every call | at most every 2000 ticks (v150.9), 1100 ticks (v151) | v150.9 + v151 | 7 |
| `Meter_ParseUDPMeterInfo` 0802B83C → 0802BCF4 | blocking `vTaskDelay(ct_type × 1200)` after a UDP meter answer | counter instead of the delay | v150.9 | 7 |
| `Grid_PowerSync_StateMachine` 08031510 → 08031ABC | same code as D v150 | changed commit condition for two or more deviating phases | v150.9 | – |
| `Write_Handler` 08055EB8 → 08056A7C, new `CAN_Write_Category0x18` 080319AC | no 47001; 45030 via the 45000 bounds path | write register 47001 (0xB799), own case for 45030 | v150.9 | – |
| `Grid_Export_Power_Limiter` 080233A8 → 0802353C | PV start value clamped to 2500 W; upper clamp 2400 W | start value without the 2500 W clamp; upper clamp 2400 W unchanged | v150.9 | – |
| `Protocol_AA_SetSystemParams` 08033DD4 → 080343BC | no range check | range check of the work-mode field | v150.9 | – |
| `BLE_Set_Device_Name` 0800F300 → 0800F2F8 | `AT+QBLENAME=MST_VNSA_%c%c%c%c` | `AT+QBLENAME=%s%c%c%c%c` with `MST_VNSA_` | v150.9 | – |
| `Cloud_BuildDeviceSourceID` 08034588 → 08034B94 | `VenusA-<serial>` | `VNSA-0-<serial>` | v150.9 | – |
| Modbus descriptor 32102 | `float32` | `i16` | v150.9 | – |

**Version/identity only:** `ShellCmd_get_ver`, version word 0x96 → 0x97 (also in `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`,
`Config_LoadOrInit_DeviceSettings`, `Cloud_UploadTelemetryData`).
**Relocated RAM offsets / log line numbers only:** `SMR_TIC_SelfUse_PowerController`, `Device_Network_Info_Init`, `Quectel_SendCmd_WaitResponse`,
`CH395_ExtractUdpData`, `Quectel_AT_QVERSION_Query`, `BLE_Pending_Commands_Process`, `Cloud_PassiveMode_Connection_StateMachine` (plus the
Ethernet-link check of v150.9), `Quectel_GetDeviceIP`, `Util_RollingHistory_ShiftAndAverage_1s`, `Meter_GetMeterIP_FromResponse`,
`MQTT_Publish_WiFi_Version`, `MQTT_Publish_PowerSchedule`, `Quectel_UDP_Socket_Open`, `debug_printf`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`.

## Corrections against the earlier derived version

| Point | Derived version said | Verified on A v150 |
|---|---|---|
| Connection watchdog | introduced in v151 ("14 400 + `System_Reboot`") | A v150 already reboots (`System_Reboot`) after 1800 with > 1 unsent record and logs `[HTTP]ch395 reset!!!!` (Venus D v150 resets only the CH395 instead); v150.9: ≥ 5 records and 3600; v151: 14 400 + suppression flag |
| Meter query period | new in v151 ("at most every 1100 ticks") | rate limit introduced in v150.9 (2000 ticks); v151 shortens it to 1100 |
| `Cloud_UploadTelemetryData` | changed in v151 only | payload builder already rewritten in v150.9, v151 adds the body length handling |
| `Config_LoadOrInit_DeviceSettings` | type and SEL LED loaded at boot in v151 | SEL LED loaded already in v150.9; server type and first-boot default 0x395 in v151 |
| `CT_GridPower_Controller` | positive setpoint reset in v151 | in addition v150.9 already skips setpoint updates while EPS reserve is active |
| `Heap_Calloc`, `Quectel_AT_SendAndPollResponse` | listed as changed | unchanged in code (only call offsets/addresses) |
| `MQTT_Transport_NullCallback`, `Util_NoOp_StartupHook` | "changed only because not locatable by bytes" | unchanged (name-paired) |
| Function counts | 58 changed, 20 relocated, 4 probably unchanged (byte segments, per v151 function) | 48 changed, 7 new, 0 removed (name-paired) |
| `Protocol_AA_SetSystemParams`, `MQTT_JSON_RPC_Dispatcher` | not listed / only "small byte changes" | behavioural: range check added; SEL LED persisted in EEPROM 0x395 (both v150.9) |

All other statements of the derived version (EPS hysteresis, status field `df`, YMODEM bin-type check, AI schedule pagination, `cd=0x42`,
`read_inv1/2`, `getDateInfo` move, HTTP POST rewrite, heap size, keepalive, 47001/0xAFE6, export limiter start value, BLE name format,
source id, 32102) were confirmed on the A v150 code.

## Strings (v150 → v151)

New: `VNSA-0 v151`, marstekcloud `getDateInfo` URLs, `AT+QHTTPPOST=%d,120,120`, `crc2` error text, `cd=%d,pg=%d/%d,`, `AI%u=…`,
`Bin type mismatch! Upgrade aborted.`, `Bin type mismatch! file=%s expect=%d got=%d`, `Bin type unrecognized! file=%s`,
`[MQTT] Type: %d, Get INV DEBUG info...`, `g_vns_inv_debug.*` field names, `read_inv1`, `read_inv2`,
`cd=19,ct_st=%d,conn_st=%d,…`, `AT+QBLENAME=%s%c%c%c%c`, `,df=%d` (status report).
Removed: hamedata `getDateInfo` URLs, `AT+QHTTPPOST=%d,60,60`, old checksum error text, `AI%d=…`, `VenusA-%s`, `AT+QBLENAME=MST_VNSA_%c%c%c%c`,
`[HTTP]ch395 reset!!!!`.
