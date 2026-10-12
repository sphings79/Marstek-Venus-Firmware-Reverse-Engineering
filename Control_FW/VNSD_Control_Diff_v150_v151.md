# Diff Table Control v150 → v151 (Venus D)

> 🇩🇪 Deutsche Version: [VNSD_Control_Diff_v150_v151_de.md](VNSD_Control_Diff_v150_v151_de.md)

**As of:** 2026-09-30. Both images with correct base 0x08004800 in Ghidra, fully named.
Addresses: v150 → v151. Changelog numbers see main documentation §7.
Function pairs via identical names (1948), compared instruction-by-instruction: 57 changed (of which 29 substantive),
6 only in v150, 8 only in v151. The 6 v150 functions are pure linker artifacts (identical-code-folding /
2-byte stubs), see "Cosmetic Differences".
**German:** `VNSD_Control_Diff_v150_v151.md` · **Main documentation:** `VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md`

## Verification Checklist

| Question | Result |
|---|---|
| Modbus registers | Read table identical except 32102 (`float32` → `i16`); new write register 0xB799 (47001); one additional indexed write slot 0xAFE6 in high register block |
| BLE | Command set identical, 3 detail changes (0x0F, 0x54, 0x59), LED status persistent |
| AC feed-in 2500 W | Configurable limits unchanged; PV start value when feeding without 2500-W clamp |
| BLE-OTA | Further possible, additional YMODEM type check |
| Cloud domains | `getDateInfo`: hamedata → `api-<region>.marstekcloud.com`; all other cloud hosts unchanged |

## Changelog v151 → Code

| Changelog Item | Code Finding |
|---|---|
| Modbus-TCP stability | `CH395_ConfigTcpKeepalive_Defaults`: IDLE 10 000 → 60 000 ms; `CH395_Init_Modbus_TCP_Socket` now applies keepalive defaults also to Modbus socket; `HTTP_ClientTask_Dispatcher`: CH395 hang watchdog from soft-reset (backlog > 1, 1800 ticks) rebuilt to tiered response (backlog ≥ 5, 14 400 ticks, then `System_Reboot()`, new suppression flag) |
| DOD cannot discharge due to abnormal status | `Battery_EPS_Reserve_SOC_Hysteresis`: Hysteresis runs only at SoC ≠ 0, EEPROM EPS flag no longer auto-reset to 1 on recovery; `CT_GridPower_Controller`: new guard explicitly resets stuck positive CT setpoint to 0 when flag `DAT_200002B2` is set; `MQTT_StatusReport_BuildAndPublish` reports matching new field `,df=%d` (presumably DOD/discharge-fault status, see "Open Items") |
| YMODEM upgrade type check | `YModem_ReceiveFile` classifies filename via new function `OTA_ClassifyDeviceTypeTag` and aborts on type 0 (unrecognized) or mismatch with CAN-0x18 message and error code −5; `YModem_FirmwareUpgrade_Handler` gets new message branch "Bin type mismatch! Upgrade aborted." |
| AI mode check optimized / query current AI strategy | `AI_ScheduleQuery_Response_Send` (cd=0x3a) sends AI schedule response now page-by-page (`pg=%d/%d`, 24 entries/page, bounds check per `snprintf`) instead of like v150 everything in one 1024-byte buffer without overflow check |
| Pre-release server URL error (data upload error) | `HTTP_Cloud_Reporting_Dispatcher` and `HTTP_GetUrlData`: `getDateInfo` host switched from `hamedata.com` to `api-<region>.marstekcloud.com`; Region table EEPROM 0x441 expanded from 3 (eu/www/us) to 5 valid values (+ dev/staging) and sanitization adjusted accordingly; `Config_LoadOrInit_DeviceSettings` now loads server type at boot instead of only on demand |
| HTTP upload data stability optimized | `Quectel_HTTP_POST_SendAndVerify`: from two blind `vTaskDelay`s becomes real, repeatedly checked tick loop with ERROR/`+QIURC` recognition and exact byte count (`UART_TransmitBytes` instead of NUL-terminated send); `HTTP_Task_StateMachine_Handler`: `AT+QHTTPPOST` timeouts 60/60 s → 120/120 s, outer timeout 600 → 6000, and post length now explicitly calculated instead of (in v150) taken from unset register; `prvHeapInit`: FreeRTOS heap 70 KB → 80 KB (plausible for larger HTTP/JSON buffers) |
| CT communication frequency optimized to 1.1 s | `Modem_SendStatusTelemetry` and `Ethernet_SendStatusTelemetry` now run in a shared time window of at most every 1100 ticks (1.1 s, timestamp `DAT_20001F58`); `Meter_ParseUDPMeterInfo` removes old blocking `vTaskDelay` after successful parse and instead only increments a diagnostics counter |

## Changes

| Area | v150 | v151 | Location (v150 → v151) | CL |
|---|---|---|---|---|
| Version string / Build | `VNSD-0 v150`, 05.08.2026 11:51:21 | `VNSD-0 v151`, 29.09.2026 09:42:27 | Strings | – |
| Version constant (cloud URLs, 0x08010000) | 0x96 (150) | 0x97 (151) | `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData` (same literal also in `ShellCmd_get_ver`, `Cloud_UploadTelemetryData`, `MQTT_StatusReport_BuildAndPublish`, `Config_LoadOrInit_DeviceSettings`) | – |
| `getDateInfo` host | `http://%s.hamedata.com/app/neng/getDateInfo%s.php` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php` (HTTP, port 80) | 080124F4 → 08012584, 080199AC → 08019A70 | 5 |
| Region table (EEPROM 0x441) | eu, www, us, eu-dev, eu-staging; valid 0–2 | eu, www, us, **dev**, **staging**; valid 0–4 | `g_cloudServerRegionTable`, `HTTP_GetUrlData` | 5 |
| Load server type at startup | on demand only | at startup (`Config_LoadOrInit_DeviceSettings`) | 0802DCBC → 0802E17C | 5 |
| Device source ID (cloud) | `VenusD-<SN>` | `VNSD-0-<SN>` | `Cloud_BuildDeviceSourceID` 08034BDC → 080351F0 | – |
| CH395 keepalive IDLE | 10 000 ms | 60 000 ms (CNT 5, INTVL 3000 ms same) | `CH395_ConfigTcpKeepalive_Defaults` 08037A0C → 080380DC | 1 |
| Keepalive before Modbus socket 502 | no | yes | `CH395_Init_Modbus_TCP_Socket` 0804DCD0 → 0804E3A0 | 1 |
| CH395 hang watchdog | Backlog > 1, 1800 ticks, `CH395_Reset_And_Reinit` | Backlog ≥ 5, 14 400 ticks, **`System_Reboot()`**, suppression flag | `HTTP_ClientTask_Dispatcher` 0801A3D0 → 0801A4A4 | 1 |
| HTTP-POST over FC41D | 2 × blind `vTaskDelay(timeout)`, 400-tick wait, no ERROR check, NUL-terminated send | Tick loop, ERROR/`+QIURC`, exact byte count (`UART_TransmitBytes`) | `Quectel_HTTP_POST_SendAndVerify` 08010598 (same address) | 6 |
| `AT+QHTTPPOST` timeouts | 60, 60 s | 120, 120 s; outer timeout 600 → 6000 | `HTTP_Task_StateMachine_Handler` 08010C20 → 08010CAC | 6 |
| FreeRTOS heap | 0x11800 (70 KB) | 0x14000 (80 KB) | `prvHeapInit` 0804FDC0 → 08050528 | (6) |
| Meter query (Shelly `EM.GetStatus`/`EM1.GetStatus`, others) | every call | at most every 0x44C = 1100 ticks (1.1 s), timestamp 0x20001F58 | `Modem_SendStatusTelemetry` 08031678 → 08031B90, `Ethernet_SendStatusTelemetry` 08031814 → 08031D80 | 7 |
| UDP meter response | blocking `vTaskDelay` after success | Counter instead of delay | `Meter_ParseUDPMeterInfo` 0802BE88 → 0802C348 | 7 |
| Quectel modem byte assembler | no call/byte counter | new counter `DAT_2001D0A6` at each call, status buffer grew by 2 bytes | `Quectel_AT_Recv_ByteAssembler` 080185CC → 08018670 | – |
| YMODEM upgrade | no type check | Filename → type (`OTA_ClassifyDeviceTypeTag`), error −5 "Bin type mismatch! Upgrade aborted." | `YModem_ReceiveFile` 0803548C → 08035AA4, `YModem_FirmwareUpgrade_Handler` 08032000 → 080325B8 | 3 |
| AI strategy response (MQTT) | one `snprintf` without limit | page-by-page send | 08023C80 → `AI_ScheduleQuery_Response_Send` 08023E08 | 4 |
| Modbus write | – | new register **0xB799**, new call `CAN_Write_Category0x18()`; in high register block additionally separate write case for **0xAFE6** (`Config_Apply_SingleReg`), which in v150 was reachable only via bounds check | `Write_Handler` 08056514 → 080570E0 | 4 |
| MQTT `cd=0x42` "Get INV DEBUG info" | Case doesn't exist | new case with sub-value `debug=N`: 1 = maintenance mode on (`g_maintModeState=2`), 2 = `MQTT_Publish_WorkMode_Config` publishes `g_vns_inv_debug` status, 3/4 = `CAN_Write_Category0x18(1)`/`(0)` toggles CAN category 0x18 and exits maintenance mode again | `MQTT_CtrlCommand_Dispatcher` 0802423C → 080244E0; new: `CAN_Write_Category0x18` (08032004), `MQTT_Publish_WorkMode_Config` (08027C0C) | – |
| Shell commands `read_inv1`/`read_inv2` | not present | `read_inv1` sets same maintenance mode (`g_maintModeState=2`) as MQTT `cd=0x42,debug=1`; `read_inv2` outputs all fields of `g_vns_inv_debug` on console | 08053328, 08053334 (v151 only) | – |
| MQTT `cd=0x13` | – | Sub-command `debug` → `MQTT_ConnectionStatus_Report_Send` (connection/counter status report: `ct_st, conn_st, err, wifi, send_ok, recv_ok, at_fail, no_resp, no_reply, sock, ip, port, reason`) | `MQTT_CtrlCommand_Dispatcher` 0802423C → 080244E0; new: `MQTT_ConnectionStatus_Report_Send` (08024244) | – |
| CT setpoint stuck positive | – | Setpoint → 0 at new flag 0x200002B2 | `CT_GridPower_Controller` 08030E80 → 08031378 | 2 |
| EPS reserve hysteresis | runs even at SoC 0 %; sets EEPROM EPS flag to 1 on recovery | only at SoC ≠ 0; no automatic reset | `Battery_EPS_Reserve_SOC_Hysteresis` 0800AD3C | 2 |
| MQTT status report | – | new field `,df=%d` | `MQTT_StatusReport_BuildAndPublish` 0802A5DC → 0802AA98 | 2 |
| PV start value when feeding | `min(P_bms, 2500 W)` | `Grid measurement − P_neg`, without 2500-W clamp (4000-W total clamp same) | `Grid_Export_Power_Limiter` 08023A00 → 08023B94 | – |
| Grid sync takeover | Commit condition applies only at exactly 2 deviating phases (counter == 2); settled-average gate additionally requires "any phase was ever nonzero" | Commit condition applies at 2+ deviating phases (counter < 2 negated, so also 3 possible); additional "ever nonzero" condition in settled-average gate removed | `Grid_PowerSync_StateMachine` 08031B60 → 08032114 | – |
| Protocol-AA system parameters (CAN register 0x11) | one fixed value pair | Index from `reg_id`, 7-slot table with bounds check (register 0x10 has used indexed `Protocol_AA_SetChannelData` since v147) | `Protocol_AA_SetSystemParams` 08034428 → 08034A18 | – |
| Economy TOU response (checksum) | one XOR checksum, one comparison | same XOR checksum additionally computed second time into own variable, acceptance on match of **either one** (mathematically identical); error message expanded with field `crc2` | `HTTP_Economy_TOU_ParseResponse` 080194CC → 08019578 | – |
| LED status (SEL LED) | RAM only | EEPROM 0x395 (BLE `Y`, MQTT RPC 0xE, loaded at startup) | BLE/MQTT dispatcher, `Config_LoadOrInit_DeviceSettings` | – |
| BLE EPS on/off (0x0F), discharge limit (0x54) | reset EPS reserve hysteresis flags (`g_epsReserveSocHysteresisState`, at 0x54 also `g_epsReserveActiveFlag`) | no longer | `BLE_App_Command_Dispatcher` | – |
| Modbus 32102 | `float32` (2 registers) | `i16` (1 register) | `g_modbusRegDescTable` | – |
| Descriptor table | 0x20000354 | 0x2000034C | .data | – |
| Shell commands (total) | 68 (+ help, clear) | 70: new `read_inv1`, `read_inv2` (see above) | `shellCommandTable` 0x08062CC4 | – |

## Cosmetic Differences (no behavioral change)

Pure address/register relocation, without substantive change (Ghidra diff still shows "changed" because
constants/offsets could not be symbolized): `BLE_Pending_Commands_Process`, `CH395_ExtractUdpData`,
`Device_Network_Info_Init`, `HTTPS_FetchUrlData` (variable names only), `Meter_GetMeterIP_FromResponse`,
`MQTT_Publish_WiFi_Version`, `Quectel_AT_QVERSION_Query`, `Quectel_GetDeviceIP`, `Quectel_SendCmd_WaitResponse`,
`Quectel_UDP_Socket_Open`, `ShellCmd_get_ver` (only version/date literal, see above), `ShellCmd_mac_ble`,
`ShellCmd_trace_debug`, `Util_RollingHistory_ShiftAndAverage_1s`, `debug_printf`, `pow` (standard C library).

`BLE_Set_Device_Name` and `MQTT_Publish_PowerSchedule`: pure decompiler/register allocation differences,
generated code and behavior identical.

`SMR_TIC_SelfUse_PowerController`: in addition to relocation a new but ineffective bounds guard before the
`switch` (no state exceeds checked range anyway).

`SPI_WriteByte`, `Util_NoOp_Stub`: the byte changes marked as "insert"/"delete" are adjacent
literals (strings "staging"/"dev" or the version literal/"VNSD-0" string) in flash layout, not part of these
functions themselves.

`__scatterload_copy`: in v151 inlines the copy-loop body, which in v150 existed as separate symbol
`Scatterload_CopyBody` (linker ICF, see below) – behavior identical.

**Only in v150 (6, all linker artifacts, no function change):**
- `Fault_InfiniteLoopTrap`, `Stub_InfiniteLoop_1/2/3`, `Stub_ImmediateReturn` – 2-byte infinite-loop or
  immediate-return stubs, which the linker in v150 keeps as standalone symbols (identical-code folding), in
  v151 but combines differently or replaces via equally tiny stub of different name
  (`Stub_ImmediateReturn` → `Util_NoOp_StartupHook`, both no-ops, called from `Board_SystemStartup`).
- `Scatterload_CopyBody` – shared copy-loop body, in v151 inlined into `__scatterload_copy`.
- Likewise purely cosmetic: `BusFault_Handler`, `HardFault_Handler`, `MemManage_Handler`, `UsageFault_Handler`
  grow from 2 to 4 bytes (additional `b` jump target), but remain same infinite loop without
  recovery – again only different linker folding of shared trap code.

## Unchanged (explicitly verified)

Modbus read path (`FC03_Read_Handler`, `Read_Serializer`), 32101 sign error, MBAP length 4 at exceptions,
44002/44003 shadow value, CR-LF receive loop, 86-s throttle, WLAN twin 900 s, TLS authmode 0 at HTTPS upload,
task priorities/stacks, BLE-OTA, all configurable 2500-W limits, debug register 46000 (0x5100), FC03 buffer overflow.

`Task_udp`: Call order (`Cloud_PassiveMode_Connection_StateMachine` → `CH395_UDP_ServerTask` →
`vTaskDelay(100)` → `Util_NoOp_Stub`) is byte-identical in v150 and v151; `Cloud_PassiveMode_Connection_StateMachine`
itself was only unnamed in v150 (`FUN_08015b84`) and was assigned to v151 counterpart during name harmonization – **no** behavioral or order change (corrected vs. earlier version of this
document that wrongly claimed changed call order here).

`MQTT_Config_ExtractFields`: table already contains 6 fields including `to=` in v150 (string "to=" at
0x0805FC18, loop `uVar2 < 6` identical in both versions); function unchanged, only
data table at different address (v150 0x0805BE5C, v151 0x0805CA7C). Claim of new 6th field
in v151 in earlier version of this document was wrong.

BLE "Get err code info" (0x13): response identical in v150 and v151 (no difference, therefore no separate line
in change table anymore).

## Open Items

- Exact meaning of `df=%d` in `MQTT_StatusReport_BuildAndPublish`: plausibly a DOD/discharge-fault status
  related to `CT_GridPower_Controller`/`Battery_EPS_Reserve_SOC_Hysteresis` change, but bit/value meaning not verified at code.
- `Write_Handler` (corrected 2026-10-12): an earlier version of this document said that Ghidra could not
  reconstruct the internal jump tables in either version. That applied to the programs used at the time. In the
  current programs (D Control 150 at base 0x08004800 and D Control 151) all seven TBB/TBH dispatches of
  `Write_Handler` are resolved, three of them with manual jump-table overrides whose entries were read from the
  raw table bytes (all targets inside the function): v150 TBB 8 / TBB 6 / TBH 15 entries, v151 TBB 8 / TBB 6 /
  TBH 16 entries. Both decompilations contain no unrecovered jump table (77 vs 78 case labels). Case labels
  differ only in the high-register switch (v150 explicit `case 0xafdf` = `Inverter_Set_Reg_0xFE`, v151
  `case 0xafe6` plus the additional TBH entry); the callee multiset is identical except the new
  `CAN_Write_Category0x18` call. The core changes named above (registers 0xB799, 0xAFE6 slot,
  `CAN_Write_Category0x18`) are verified at code; a complete branch-by-branch equivalence proof of the
  remaining branches was still not done, so small deviations in uninvolved branches cannot be ruled out
  (medium rather than high confidence).
- Additional initializer `DAT_2001A8AA = 0xA892` in `Config_LoadOrInit_DeviceSettings` (v151, 2 bytes after
  "VNSD-0" string in device ID setup) – purpose not apparent from this function alone.
- Whether new MQTT command `cd=0x42` ("Get INV DEBUG info", maintenance mode + `MQTT_Publish_WorkMode_Config` +
  `CAN_Write_Category0x18`) maps to one of seven changelog items or is separate addition not listed there is open.
- Reason for duplicated (but mathematically equivalent) CRC calculation in `HTTP_Economy_TOU_ParseResponse`
  is not apparent from code.
