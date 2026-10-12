# Diff Table Control v150 → v150.9 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_Control_Diff_v150_v1509_de.md](VNSA_Control_Diff_v150_v1509_de.md)

**As of:** 2026-10-11. Both images with correct base 0x08004800 in Ghidra, fully named (v150: 1935 functions, v150.9: 1941).
**Main documentation:** [VNSA_Control_FW_v150_Full_Analysis.md](VNSA_Control_FW_v150_Full_Analysis.md), [VNSA_Control_FW_v1509_Full_Analysis.md](VNSA_Control_FW_v1509_Full_Analysis.md)
**Direction:** v150 (build 11 Aug 2026, release) → beta v150.8 (19 Aug, not analysed) → beta v150.9 (8 Sep). v150.9 is a beta *after* v150
on the way to v151 (the archive changelog lists it as "v150.9 (Beta)"), so this table shows what the beta added to the release v150.

Function pairs via identical names (1935), compared instruction by instruction with call targets by name, referenced strings and immediate
values (literal-pool and RAM addresses masked, because the RAM layout differs): **42 changed** — 24 with changed behaviour, 3 only version/identity,
15 only relocated RAM offsets or log line numbers — **6 only in v150.9**, **0 only in v150**.

## Verification checklist (standing checks)

| Question | Result |
|---|---|
| Modbus registers | Descriptor table: same 246 registers, only **32102** changes `float32` (2 registers) → `i16` (1 register); table moved 0x20000354 → 0x2000034C. `Write_Handler`: new write register **47001** (0xB799, CAN 0x18 via new `CAN_Write_Category0x18`), 45030 (0xAFE6) gets its own case (same effect); 44002/44003 limits unchanged (44003 > 1500 W rejected in both). Read path unchanged. 30200 reads 150 → 1509. |
| BLE | `BLE_App_Command_Dispatcher`: cmd `T` (0x54) no longer resets the EPS hysteresis; BLE name now `AT+QBLENAME=%s%c%c%c%c` with `MST_VNSA_` as argument (`BLE_Set_Device_Name`, same result). Power classes/limits unchanged. |
| AC feed-in limit | Unchanged 1500 W (all clamps, 44003). `Grid_Export_Power_Limiter`: the PV start value is no longer clamped to 2500 W first; upper clamp 2400 W unchanged. |
| BLE OTA | Still possible, BLE-OTA path unchanged. New: YMODEM bin-type check (`OTA_ClassifyDeviceTypeTag`, "Bin type mismatch! Upgrade aborted."). |
| Cloud domains | Unchanged: upload on marstekcloud.com, `getDateInfo` and other endpoints on hamedata.com, server types 0–2. Device source id `VenusA-<serial>` → `VNSA-0-<serial>`. |
| Pack capacity | Unchanged: 7 per-pack records, 13 cells scanned for packs 1..6. |
| Non-functional registers | 47001 becomes implemented (sends CAN 0x18, ignored by the A micro → stays NO_OP/FUNCTIONLESS); 32102 changes type. No other register-visible change. |

## Changes v150 → v150.9 (behaviour)

| Function (v150 → v150.9 address) | v150 | v150.9 |
|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` 0800AC1C → 0800AC1C | three-state machine; below 9 % SOC EPS off (EEPROM 0x300 = 0) and **automatically back on** above 8 % | hysteresis only when SOC ≠ 0, EPS flag no longer re-enabled automatically |
| `BLE_App_Command_Dispatcher` 0800C5B8 → 0800C5C8, `MQTT_CtrlCommand_Dispatcher` 08023BEC → 08023D9C | BLE `T` / MQTT `cd=0x38` reset the EPS hysteresis state and `g_epsReserveActiveFlag`; highest `cd` 0x41 | reset removed; new `cd=0x42` "Get INV DEBUG info" (`MQTT_Publish_WorkMode_Config`, `g_vns_inv_debug`) and `cd=19` connection report (`MQTT_ConnectionStatus_Report_Send`) |
| `CT_GridPower_Controller` 08030830 → 08030C10 | same code as D v150 | setpoint update skipped while the EPS reserve is active |
| `Config_LoadOrInit_DeviceSettings` 0802D670 → 0802DA34 | SEL LED not loaded at boot; type code 0x96 | SEL LED loaded from EEPROM 0x395; type code 0x5E5 |
| `MQTT_JSON_RPC_Dispatcher` 0801F144 → 0801F21C | SEL LED setting kept in RAM only | additionally written to EEPROM 0x395 |
| `MQTT_StatusReport_BuildAndPublish` 08029F90 → 0802A350 | status string without `df` | new field `,df=%d`, energy fields reordered |
| `Cloud_UploadTelemetryData` 0801B238 → 0801B23C | payload builder of D v150 | rewritten payload builder (581 → 700 instructions) |
| `Cloud_PassiveMode_Connection_StateMachine` 08015A24 → 08015A28 | — | additional Ethernet-link check |
| `HTTP_ClientTask_Dispatcher` 0801A270 → 0801A294 | > 1 unsent record, old-step 0, 1800 elapsed → log `[HTTP]ch395 reset!!!!` + `System_Reboot` | ≥ 5 unsent records, 3600 elapsed → `System_Reboot` (no log) |
| `Modem_SendStatusTelemetry` 08031028 → 08031418, `Ethernet_SendStatusTelemetry` 080311C4 → 08031608 | meter query on every call | at most every 2000 ticks; modem AT fail/OK counters |
| `Meter_ParseUDPMeterInfo` 0802B83C → 0802BC00 | `vTaskDelay(ct_type × 1200)` after a parsed 3-phase answer | receive counter instead of the delay |
| `Quectel_AT_Recv_ByteAssembler` 0801846C → 08018484 | — | received-byte counter |
| `CH395_Init_Modbus_TCP_Socket` 0804D674 → 0804DC1C | no keepalive on socket 502 | `CH395_ConfigTcpKeepalive_Defaults` also for socket 502 (idle still 10 s) |
| `Grid_Export_Power_Limiter` 080233A8 → 08023488 | PV start value clamped to 2500 W, upper clamp 2400 W | start value without the 2500 W clamp, upper clamp 2400 W |
| `Grid_PowerSync_StateMachine` 08031510 → 0803199C | same code as D v150 | changed commit condition when two or more phases deviate |
| `Protocol_AA_SetSystemParams` 08033DD4 → 0803429C | stores the received pair without check | range check of the work-mode field |
| `Write_Handler` 08055EB8 → 08056960 | no 47001; 45030 via the 45000 bounds path | new 47001 (CAN 0x18), own case for 45030 |
| `YModem_ReceiveFile` 08034E38 → 08035328, `YModem_FirmwareUpgrade_Handler` 080319B0 → 08031E40 | size/CRC check only | bin-type check of the file name (`OTA_ClassifyDeviceTypeTag`) |
| `prvHeapInit` 0804F764 → 0804FDA4 | heap 0x11800 (70 KB) | 0x14000 (80 KB) |
| `BLE_Set_Device_Name` 0800F300 → 0800F2F8 | `AT+QBLENAME=MST_VNSA_%c%c%c%c` | `AT+QBLENAME=%s%c%c%c%c` + `MST_VNSA_` |
| `Cloud_BuildDeviceSourceID` 08034588 → 08034A74 | `VenusA-<serial>` | `VNSA-0-<serial>` |

## Changes v150 → v150.9 (version/identity only)

`HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData` (`sv=` 150 → 1509), `ShellCmd_get_ver` (build stamp, `ems_ver`).

## Changes v150 → v150.9 (relocated RAM offsets / log line numbers only)

`BLE_Pending_Commands_Process`, `CH395_ExtractUdpData`, `Device_Network_Info_Init`, `MQTT_Publish_PowerSchedule`, `MQTT_Publish_WiFi_Version`,
`Meter_GetMeterIP_FromResponse`, `Quectel_AT_QVERSION_Query`, `Quectel_GetDeviceIP`, `Quectel_SendCmd_WaitResponse`, `Quectel_UDP_Socket_Open`,
`SMR_TIC_SelfUse_PowerController`, `ShellCmd_mac_ble`, `ShellCmd_trace_debug`, `Util_RollingHistory_ShiftAndAverage_1s`, `debug_printf`.

## Only in v150.9

`OTA_ClassifyDeviceTypeTag`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18`,
`ShellCmd_read_inv1`, `ShellCmd_read_inv2` (shell table 79 → 81 entries).

## Release notes

v150.9: "PV dropout during surplus feed-in fixed, LED blinking fixed, CT connection/phase detection improved". Code changes that fit these items (assignment not confirmed by the vendor): grid sync
(`Grid_PowerSync_StateMachine`, `Grid_Export_Power_Limiter` start value), CT/meter query (`Modem_/Ethernet_SendStatusTelemetry` rate limit,
`Meter_ParseUDPMeterInfo`), SEL LED persistence (`Config_LoadOrInit_DeviceSettings`, `MQTT_JSON_RPC_Dispatcher`). Not mentioned in the note:
EPS reserve rework, YMODEM bin-type check, inverter debug info, 47001, heap size, watchdog and keepalive changes.

## Strings (v150 → v150.9)

New: `VNSA-0 v1509`, `Build:\tSep  8 2026 09:34:19`, `AT+QBLENAME=%s%c%c%c%c`, `Bin type mismatch! …`, `Bin type unrecognized! …`, `nametpype =%d`,
`[MQTT] Type: %d, Get INV DEBUG info...`, `g_vns_inv_debug.*`, `read_inv1`, `read_inv2`, `cd=19,ct_st=%d,conn_st=%d,…`, `cd=%d,wm=%d,sl_f=%d,…`,
new status-report and upload format strings, `%s-%s`.
Removed: `VNSA-0 v150`, `AT+QBLENAME=MST_VNSA_%c%c%c%c`, `VenusA-%s`, `[HTTP]ch395 reset!!!!`, old status-report and upload format strings.
