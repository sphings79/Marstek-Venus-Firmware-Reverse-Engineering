# Diff Table Control v147 → v149 (Venus D) — Main Hop

> 🇩🇪 Deutsche Version: [VNSD_Control_Diff_v147_v149_de.md](VNSD_Control_Diff_v147_v149_de.md)

**As of:** 2026-09-30. Both images with base 0x08004800, all verification steps passed (`marstek_gates.py`, local only, not in the repo).
Addresses: v147 → v149. Function pairs via identical names (1906), instruction-wise compared:
**124 changed functions** (of which 75 substantive, 49 purely cosmetic), 2 only in v147, 25 only in v149;
contents of changes via code review (offline agents, core statements verified against code).
**German:** `VNSD_Control_Diff_v147_v149.md` · **Next Hop:** `VNSD_Control_Diff_v149_v1492.md`, `VNSD_Control_Diff_v149_v150.md`

## Checklist

| Question | Result |
|---|---|
| Modbus registers | **15 new**: 38000–38014 (cumulated energy counters `g_energyCounterAt*`, `g_energyRecA/B/C_*`); none removed, type/scaling/count of remaining 231 unchanged (verified by byte comparison of descriptor entries — only the RAM source address shifts). Descriptor table grows from 0xe7 (231) to 0xf6 (246) entries. Reading from 38000–39014 additionally sets `g_canCat40to43PollEnableFlag` in v149 (CAN category 0x40–0x43; range `37999 < reg < 0x9867` verified in code). Write path: new block 46500–46544 (0xB5A4–0xB5D0, 15 × 3 registers → `Inverter_Set_Schedule_Reg`/`Config_Param0A_Relay`, EEPROM 0x36BF+8·n) |
| BLE | Command set **extended**: new 0x5A (trigger cloud backup fetch, `bk_en`), 0x5B (IP/MAC/serial number); 0x54 (discharge limit) additionally resets EPS reserve flags; responses/status fields extended (`BLE_Build_DeviceStatus_Response`: +power limit flag, +PV channel delta, response length 0xa1→0xa4 bytes) |
| AC feed-in / power | **2200 W → 2500 W**, consistently implemented across at least 12 functions (clamp defaults, EEPROM defaults, `Config_Write_U16`/`_0x202`, `Grid_Export_Power_Limiter`, `Battery_Charge_Power_Limiter`, `Remote_Power_Command_Execute`, MQTT/BLE/JSON-RPC setters, `Write_Handler` charge limit) — see "Power limit 2200→2500 W" section; `Config_Apply_WorkModePowerPreset` (800/600/2200/1200 W per mode) removed entirely; new RPC method `Set.Ver` (JSON-RPC 0x10) selects 800 / 2200 / 2500 W |
| BLE-OTA | **continues to work** — `BLE_Cmd_OTA_*`, YMODEM and all `OTA_*` functions fully present and identically named in both versions; YMODEM target codes identical (0x04D2 → EMS, 0x0929 → VENUS) |
| Cloud domains | **unchanged** (`%s.hamedata.com`, present in both images); only new is path `/external-services/api/v1/devices/backup-configs?devid=%s&latest=true` |

## Changelog v149 → Code

| Changelog point | Code finding |
|---|---|
| further OpenAPI functions | `MQTT_JSON_RPC_Dispatcher`: **11 → 16 methods** (method table `memcpy` 0x58→0x80 bytes = 11→16 entries @ 8 B each; case 0 is "Method not found", methods 1–0xB → additionally 0xD–0x11). New: `Ble.Adv` 0x0D (BLE advertising enable, EEPROM 0x36bd), `Led.Ctrl` 0x0E (`g_selLedEnable`), `Ota.Set` 0x0F (OTA URL slot: device type/mod/version/URL/size/CRC, flash write per slot, `g_otaUpdatePendingFlag`), `Set.Ver` 0x10 (work mode/preset 800/2200(0x898)/2500(0x9c4) W), `Reset.Factory` 0x11 (type 1/2/3, `Factory_Reset`) — method names verified from string pool. Additionally (not mentioned in changelog): dedicated, non-overlapping response buffers per method (v147 used a single shared buffer with offsets) |
| ES delivers PV = 0 | `Cloud_Report_FillGridData`: v147 writes PV sum (+0x27) and PV yearly capacity (+0x31) fixed to 0; v149 calculates them from the 4 MPPT channels or `g_mpptPvYearCap10Wh` |
| I²C deadlock recovery | `EEPROM_I2C_ReadBytes/WriteBytes`: timeout 3000 → 0x64000 (409600), new call `EEPROM_I2C_BusRecovery()` in error paths. Recovery chain consists of three new functions: `EEPROM_I2C_BusRecovery` (logs, calls the following two plus `EEPROM_I2C_Peripheral_Init`), `EEPROM_I2C_ClockDisable_Reset` (I2C1 periph reset, APB1 clock off, AFIO remap back), and `EEPROM_I2C_GPIO_ClockPulseRecovery` (SCL 9× as GPIO pulsed — classic manual I2C bus recovery for a stuck slave) |
| French meters | `SMR_TIC_SelfUse_PowerController`: completely new tick-clocked state machine (states 0–5) for `g_meterSourceType == 0x0A` (TIC/Linky), setpoint from `g_smrPowerDeltaAvg`; `CT_GridPower_Controller` gets new ramp-up phase for surplus feed-in and skips delta detection for meter type 10; `Grid_PowerSync_StateMachine` additionally accepts meter types 9/0x0A/0x0C, separate 60-cycle timeout for type 12, inverted timer condition for baseline capture (acts like a bugfix), and removed upper limit in deviation detection |
| HTTP upload + IP + cumulated energy | `Cloud_UploadTelemetryData`: local IP (ethernet or from `g_devNetInfo`) in new field `ip=%s`, plus new field `bkd=%s` (matching new backup command 0x5A); energy counters also as Modbus 38000–38014; upload buffer 0x44c→0x4b0 bytes, telemetry history stride 0x30b→0x35b bytes |
| Surplus feed-in optimized | `Battery_Charge_Power_Limiter`: result from `Power_ScaledValue_Calc_Negated()` now clamped to minimum −2500 W (`-0x9c4`) (in v147 unbounded); `CT_GridPower_Controller`: new ramp-up phase via `Inverter_Power_Setpoint_Calc(1,10)` on activation |
| DOD sent incorrectly | `Cloud_DOD_Command_Handler`: range check 30–88 % (0x1e–0x58; v147 just echoed the read value), response format switched to `cd=%d,ret=0/1`; `Config_Set_DischargeCutoff_WithRelay` now uses new helper `Config_Param0A_Relay` instead of `Inverter_Set_Schedule_Reg` directly (functionally equivalent, extracted because `Inverter_Set_Schedule_Reg` in v149 got a register range check 0x30–0x3e that would have rejected the old CAN reg 0x0A path) and normalizes return value to 0/1; BLE 0x54, MQTT 0x38 (`Cloud_DOD_Command_Handler`), and MQTT 0xb (EPS enable/disable) previously reset `g_epsReserveSocHysteresisState`/`g_epsReserveActiveFlag` |
| HTTP upload interval 5 min | in code **not found as difference** (HTTP task timer identical in both versions) |
| max power configurable / 2500 W | see checklist and "Power limit 2200→2500 W" section |
| VA cell voltage | affects Venus A, not checked |

## Power Limit 2200 W → 2500 W (cross-functional)

A single limit increase, consistently implemented at the following locations (0x898→0x9c4 or upper bounds 0x899→0x9c5):

| Function | Location |
|---|---|
| `Config_ClampDefaults_ParamBlock` | Default/clamp value for charge/discharge limit |
| `Config_LoadOrInit_DeviceSettings` | `g_maxChargePowerW` default on first boot |
| `Config_Write_U16` / `Config_Write_U16_0x202` | Clamp when writing max discharge/charge power |
| `Grid_Export_Power_Limiter` | new clamp `power_target` to max 0x9c4 before calculation with `g_gridSamplePowerW` |
| `Battery_Charge_Power_Limiter` | new lower bound −2500 W (see changelog table) |
| `Remote_Power_Command_Execute` | auto-limit mode now uses 0x9c4 instead of 0x898 |
| `MQTT_CtrlCommand_Dispatcher` case 0x10/0x11 | upper bound when setting max charge/discharge power 0x899→0x9c5 |
| `BLE_App_Command_Dispatcher` cd 0x16/0x17 | same limit via BLE |
| `MQTT_JSON_RPC_Dispatcher` "Set.Ver" (0x10) | preset values 800/2200/2500 W |
| `Write_Handler` | charge power limit in new register block 46500–46544 |

Additionally, `Config_Apply_WorkModePowerPreset` (fixed mapping mode 1/2/3/4 → 800/600/2200/1200 W) is removed entirely — see "Removed Functions". The preset tables in `MQTT_CtrlCommand_Dispatcher` (case 0xf) and `BLE_App_Command_Dispatcher` (cd 0x15) were reworked accordingly: the 1200 W preset (0x4b0) is removed, new is a 2500 W preset (0x9c4); in `MQTT_CtrlCommand_Dispatcher` a bug is also fixed where `Config_Write_U16` was writing a hardcoded instead of the actually parsed power.

## Further Changes (not in changelog)

### EPS Reserve Hysteresis

| Function | v147 | v149 | Meaning |
|---|---|---|---|
| `Battery_EPS_Reserve_SOC_Hysteresis` | simple SoC ±5% comparison, one flag | 3-state state machine (`g_epsReserveSocHysteresisState`/`…Count`), deactivation only when `g_epsReserveEnable==1` | Significantly more robust debouncing on EPS reserve switch |
| `MQTT_CtrlCommand_Dispatcher` (case 0xb, 0x38), `BLE_App_Command_Dispatcher` (cd 0x0f, 'T') | no reset | set `g_epsReserveSocHysteresisState`/`g_epsReserveActiveFlag` on manual enable/disable or on setting discharge cutoff | Hysteresis state kept synchronized on every external intervention |

### MQTT/Cloud

| Function | v147 | v149 | Meaning |
|---|---|---|---|
| `MQTT_CtrlCommand_Dispatcher` | `/App/<id>` only checked against topic/group ID | additionally `strcmp("update", device_id)` sets `g_mqttCloudUpdateRequestFlag` | New trigger for cloud update request via topic ID "update" |
| `MQTT_CtrlCommand_Dispatcher` case 3 | at `time_num >= 10` no response | sends error response `cd=3,ret=0` | New error acknowledgment |
| `MQTT_CtrlCommand_Dispatcher` case 0x1e | enable → `g_apiConnState='\x03'` | enable → `g_apiConnState='\x01'`, new flag `g_localApiUdpEnable` | New UDP switch for local API |
| `MQTT_CtrlCommand_Dispatcher` case 0x3d (new) | – | parses `bk_en=`, sets `g_backupDataRequestFlag`, acknowledges | New "request backup data" command |
| `MQTT_CtrlCommand_Dispatcher` case 0x3e (new) | – | calls `MQTT_Publish_DeviceIDs` | New "publish device IDs" command |
| `MQTT_Connect_And_Subscribe` | no status flag | sets `g_quectelWifiMqttReadyFlag` after successful connect, clears on error | New "WiFi/MQTT ready" marker |
| `MQTT_Server_Config_Select` | fallback only at `g_mqttVidSessionState==1`; `void` | condition removed, fallback always applies on invalid XID; returns `ctx` | Bugfix: `ctx` could remain unconfigured in v147 |
| `MQTT_Telemetry_Struct_Builder` | `g_mqttTeleCloudLinkActive = g_quectelWifiMqttReadyFlag` | additionally `\|\| g_mqttSessionActive` | cloud link status accounts for active MQTT session |
| `MQTT_Telemetry_String_Formatter` | single composed string | separate energy counter groups A (cat. 4) / B (cat. 3) plus new field battery charge/discharge power | Extended telemetry |
| `MQTT_StatusReport_BuildAndPublish` | format ends `…,led=%d,as=%d` | two more fields `,lf=%d,pl=%d` (arguments still fixed 0); additional call `Inverter_PVString_Channel_ValueDelta_Calc()` | New placeholder fields (meaning open) plus updated channel delta calculation before report |
| `MQTT_Publish_Network_Info` | `ct_connect_ip` = `g_meterQueryIpStr` | `ct_connect_ip` = `g_meterResolvedIpStr` (short form stays `g_meterQueryIpStr`) | detailed report now sends resolved instead of queried IP |
| `MQTT_JSON_RPC_Dispatcher` case 7 "Manual" | no reset | sets `g_workModeUpsSubFlag=false`, after saving a time window additionally `g_aiSchedule9Enable='\0'` | state reset on switch to manual mode (meaning of "Schedule 9" open) |
| `Quectel_MQTT_ConnectionWatchdog_Tick` | passive-mode timeout check only at `!g_ethLinkActive` | runs independent of ethernet link (only bound to `g_bleOtaBusyState`) | Previously skipped entirely when ethernet active |
| `Quectel_MQTT_Session_StateMachine` | SSL/stale watchdog: modem hardware reset only | additionally `Factory_Reset(3, …)` (erases TLS certificate + EEPROM 0x3530, `System_Reboot()`) before reset | Significantly harsher response to hung SSL connection |
| `Quectel_MQTT_Session_StateMachine` | periodic `MQTT_Config_VID_Clear()` every 600 ticks in XID branch (3 places) | code path entirely removed, return value from `Quectel_MQTT_Subscribe` no longer evaluated | corresponds to removal of `MQTT_Config_VID_Clear` (see "Removed Functions") |
| `MQTT_Config_VID_Init` | always called | only if `g_mqttXidConfigXid` empty; flash address 0x315000→0x317000, read size 0xe4→0xe5 | prevents unnecessary overwriting of already loaded XID data |
| `EEPROM_Config_Factory_Write` | flash region 0x315000 | flash region 0x317000 | factory MQTT VID/XID region shifted by 0x2000 |

**New cloud config backup feature** (ethernet and cellular path): `HTTP_ClientTask_Dispatcher` and `HTTP_Task_StateMachine_Handler` now recognize `g_backupDataRequestFlag`, query via new state or `HTTP_GetUrlData(8,…)`/`HTTP_Cloud_Reporting_Dispatcher` case 8 the URL `…/backup-configs?devid=%s&latest=true`, and pass the result to new `MQTT_Config_Command_Handler` (parser for `key=value;…` config commands: work mode, BLE advertise, feed-in flag, local API, firmware version, max charge/discharge power with 2500 W fallback, WLAN credentials, DOD, HTTP server type, backup enable, and up to 10 TOU schedule entries via `MQTT_Config_ParseScheduleEntries`/`Parse_TimeString_To_HourMinute`). Comes via HTTP cloud requests, not directly via MQTT client — see "New Functions".

### Energy Counter Category 4 (grid feed-in/grid-sell)

| Function | v147 | v149 | Meaning |
|---|---|---|---|
| `Config_Load_RuntimeCounters` / `Config_Save_RuntimeCounters` | EEPROM block 0x28 bytes, only categories 1–3 checked/repaired | block 0x34 bytes, additional repair block for `g_energyCat4TotalWh`/`DayWh`/`MonthWh` | new fourth energy counter category |
| `Energy_Stats_Accumulate_And_Save` | only categories 1–3 accumulated | new block for category 4 (`g_gridSellEnergyWh` → `g_energyCat4*`), also in memory trigger | grid-sell accumulation |
| `Stats_Clear_Counters` | scopes 1–3, memset block 0x28; scope 1 falsely nulls `g_energyCat3MonthWh` | scope 2 additionally nulls `g_energyCat4Day/MonthWh`; scope 1 now nulls `g_energyCat3DayWh`/`g_energyCat4DayWh`; block 0x34 | category 4 counters incorporated in reset schema |
| `Telemetry_Register_Dispatcher` | handles IDs 0x10–0x14, 0xc1, 0xc3, 0xcb, 0xce, 0x54 | additionally IDs 0x40–0x43 → new function `Telemetry_Store_EnergyCounters` | new CAN telemetry registers for category 4 |
| `CAN_Broadcast_Category0x17_PackedValues` (new) | – | packs `g_energyCat3TotalWh`/`g_energyCat4TotalWh` in CAN broadcast category 0x17 | see "New Functions" |
| `CAN_Broadcast_ClearCategories0x40to0x43` (new) | – | four null-payload CAN frames for categories 0x40–0x43 | see "New Functions" |
| `Config_LoadOrInit_DeviceSettings` | EEPROM energy statistics block 0x28 bytes | 0x34 bytes | direct consequence of new category |

### BMS Pack Record (per pack/CAN node)

Foundation already known and now verified against code: record grows from 0x28×uint16 (80 B) to
0x2C×uint16 (88 B) per pack — `CAN_AA_StoreUnitFieldPair8` uses `*0x28` in v147, `*0x2c`
in v149 (verified directly in code). All six `CAN_AA_StoreUnitFieldPair*` variants as well as
`CAN_AA_StoreUnitIndexedFieldPair`, `CAN_Detect_Mismatched_Nodes`, `BMS_CellVoltage_MinMax_PerPack`,
`BMS_Version_Consistency_Check`, and `MQTT_Publish_BMS_Data` change **only** through migrating
field offsets (cosmetic consequence, no own logic change). New in v149:

| Function | Meaning |
|---|---|
| `CAN_AA_StoreUnitFieldPair80` (new) | writes an 8-byte value pair at offset 0x50 of pack record in `g_bmsPack0ChargePower`/`DischargePower` — new data field per pack |
| `ShellCmd_get_all_bms_data` | two new printf lines "bat charge power"/"bat discharge power" (reads same new fields); additionally independent stride switch 0x50→0x58 for `mos_flag` values (different struct, meaning open) |

### Modbus/CAN transmission path and cellular fallback

| Function | v147 | v149 | Meaning |
|---|---|---|---|
| `FC03_Read_Handler` / `RS485_FC03_ReadWrite_Handler` | table 0xe7 (231) entries, response only via CH395/ethernet | table 0xf6 (246); new condition sets `g_canCat40to43PollEnableFlag` on register access 38000–39014; response additionally copied to `g_modbusLteTxFrame` and sent via `Quectel_AT_QISEND_HexData` over cellular when ethernet link inactive | Modbus read now also possible via LTE, new CAN 40-43 poll trigger |
| `TCP_FC06_WriteSingle_Handler` | response always via CH395 | on `g_ethLinkActive==false` instead `Quectel_AT_QISEND_HexData` | same LTE fallback for FC06 write responses |
| `Task_Modbus_tcp` | calls `Modbus_Dispatcher()` unconditionally | only if `g_ethLinkActive==true` | complement to LTE fallback |
| `Quectel_AT_QISEND_HexData`, `Quectel_AT_SendRaw_WithMutex`, `Quectel_UART_RxByte_Parser`, `StrCat` (all new) | – | hex encoding + mutex-protected AT sending + byte assembler for modem UART | infrastructure for LTE fallback, see "New Functions" |
| `Quectel_UART_RxByte_ISR_Handler` | raw mode (case 5) called only HTTPS response matcher | new own case 5 buffers bytes in `g_quectelAtRawRxBuf` with EOL detection, notifies `hTask_Modbus_tcp`; HTTPS matcher moves to case 6; additional call to `Quectel_UART_RxByte_Parser` | extended UART demultiplexer for raw Modbus-over-Quectel |
| `Write_Handler` | no register range 0xb5a4–0xb5d0 | new block 0xb5a4–0xb5d0 (46500–46544): 3-field table `g_canScheduleRegCache`, EEPROM-persisted (`idx*8+0x36bf`), calls `Inverter_Set_Schedule_Reg` | new persistent power limit table via Modbus (see checklist) |
| `Inverter_Set_Schedule_Reg` | no range check of `reg_id`; sends only 4 CAN bytes, `value_hi` never used | guard `0x2f<reg_id<0x3f`; `CAN_Build_Arbitration_ID` roles of `reg_id`/constant swapped; sends 8 bytes with full `value_lo`/`value_hi` | real protocol fix: previously completely unused `value_hi` parameter now transmitted |
| `Config_Param0A_Relay` (new) | – | writes 1-byte relay value in CAN parameter 0x0A for a node ID | see "Removed/New Functions" – extracted from `Config_Set_DischargeCutoff_WithRelay` |
| `Protocol_AA_CommandDispatch` | dispatch table 15 entries | 16 entries | one more 0xAA command, meaning open |

### Grid/meter regulation (beyond French meters)

| Function | v147 | v149 | Meaning |
|---|---|---|---|
| `Inverter_Power_Setpoint_Calc` | PV sum only from four MPPT channels | additional fixed offset +10 (FP64) before summation; new flag `g_invPowerSetpointClampedFlag` when setpoint hits limit | new watt headroom plus diagnostic flag for clamped setpoint |
| `RTC_TimeOfDay_To5MinSlotIndex` | minute / 10 | minute / 5 | divisor for 5-minute grid changed — directly affects calculated slot index |
| `HexChar_To_TimeOffsetIndex` | 3 hex chars from `g_touHexConfigBuf`, XOR, mod 10 | 1 hex char from `g_bleMacIdChar11`, mod 5 | completely different input source/algorithm, value range halved — precise factual effect on TOU time windows open |
| `Timer_Periodic_DispatchAll` | 9 fixed callbacks; cat-0x17 interval 10000 ms; maint mode 1→reg 0x11, 2→reg 0x12 (with CAN clear @0x2d0 ms) | new callback "EPS SoC hysteresis" @6000 ms; cat-0x17 interval 50000 ms; maint mode registers swapped (1→0x12, 2→0x11); CAN 40-43 clear only if `g_canCat40to43PollEnableFlag` set | several real timing/sequence changes |
| `WorkMode_ChangeHandler` | `Inverter_Power_Setpoint_Calc(1,0)` on every switch | not when `g_userWorkMode==3` | setpoint reset suppressed for new TIC mode (suspected) |
| `WorkMode_Flag_Reset_And_TriggerSetpointEval` | no reset | additionally sets `g_smrTicState='\0'` | reset of new TIC state machine on mode switch |
| `Surplus_FeedIn_Enable_Check` | two branches with different fixed assignment of packed `reserved` field | one branch removed, other packs field differently (`CONCAT11(1, g_surplusFeedInReserved)`) | clearly different behavior, exact bitfield semantics open |
| `Board_SystemStartup` | `Config_LoadOrInit_DeviceSettings()` before `Board_PeripheralSelfTest()` | sequence reversed | boot sequence changed |
| `Config_LoadOrInit_DeviceSettings` | `g_deviceTypeCode=0x93`; `g_workMode` range to 4; no error bit reset | `g_deviceTypeCode=0x95`; range to 5; `g_errCodeBitmask &= ~0x100` on normal boot | several real detail changes in config loading |

### HTTP buffer enlarged (~1024/1100 → ~1200 bytes)

Consistent enlargement of several HTTP/AT buffers throughout: `HTTP_ParserOnBody`
(body copy 0x400→0x4b0), `HTTPS_POST_Request` (nulled range +0xB0), `HTTP_GetUrlData`
(snprintf bound 0x44c→0x4b0), `Quectel_HTTP_GET_StateMachine` (AT command buffer 0x44c→0x4b0),
`Cloud_UploadTelemetryData` (upload buffer 0x44c→0x4b0). Accordingly, telemetry history record in
`Telemetry_History_Buffer_Init`/`_Record_Push` and `HTTP_Task_StateMachine_Handler` grows from
0x30b to 0x35b bytes (+80 B, content of new 80 bytes not determined in detail — likely new category 4 fields).

### Miscellaneous

| Function | v147 | v149 | Meaning |
|---|---|---|---|
| `QSPI_Flash_QuadRead`, `QSPI_WaitBusyDone` | `Delay_BusyWait_Us(…)` | `Delay_BusyWait_Ms(…)` (same numerical value) | significantly longer wait times (µs→ms) on QSPI access |
| `Time_RtosAwareDelayMs` | calls `Delay_BusyWait_Us(0xfa)` 4× | calls `Delay_BusyWait_Ms(0xfa)` 4× | same transition as QSPI — meaning/effect on timing open |
| `Quectel_AT_SendCommand_Locked` | `g_quectelAtRxMode='\x05'` | `='\x06'` | RX mode constant changed, meaning open |
| `sscanf_Format_Parser` | breaks on every error without return value tracking | performs conversion counter; EOF before first match returns −1, else count of conversions | behavior closer to standard `sscanf`; could also stem from compiler/library update rather than conscious firmware change |
| `ShellCmd_api_set` | sets only `g_localApiEnable`/`g_apiConnState` | additionally `g_localApiUdpEnable` | new UDP flag (see MQTT table) |
| `ShellCmd_set_wifi` | no flag after EEPROM write | sets `g_wifiCredApplyPendingFlag` | likely triggers asynchronous application of WLAN credentials |
| `CAN_CommLink_Timeout_DataClear` | `memset(&g_invState, 0x34, …)` | `memset(&g_invState, 0x38, …)` | inverter state block +4 bytes, independent of BMS pack growth, meaning open |
| `Energy_TOU_Counters_Update` | field offset `energy_ctx+0x2cb` | `energy_ctx+0x2e3` | struct grown by 0x18 (24 B) — not same shift as BMS pack record, meaning open |

## Cosmetic Changes

49 of 124 changed functions demonstrably show no behavioral difference:

- **Pure register allocation/decompiler variable names** (no text/logic difference in C code or only different variable names for same semantics): `BLE_Build_Cmd0x42_Response`,
  `Board_ClockSource_TrySwitchToHSE`, `CH395_ExtractUdpData`, `CH395_Socket_Dest_Config`,
  `Config_Mode_SyncIfChanged`, `Device_Network_Info_Init`, `Factory_Reset`, `Heap_Calloc`,
  `MQTT_Publish_BMS_Version`, `MQTT_Publish_PowerSchedule`, `MQTT_Session_Init`, `MQTT_SetWorkMode`,
  `MQTT_TLS_Context_Reset`, `Quectel_AT_QVERSION_Query`, `Quectel_GetDeviceIP`,
  `Quectel_HTTP_AT_SendAndVerify`, `Quectel_HTTP_GET_Request`, `Quectel_MQTT_Subscribe`,
  `Quectel_OTA_StartUpdate_AndReboot`, `Quectel_OTA_StartUpdate_FromURL_AndReboot`,
  `Quectel_UDP_Socket_Open`, `SPI_WriteByte`, `ShellCmd_set_old`, `Stub_ImmediateReturn`,
  `TLS_Cert_Decrypt_All`, `__scatterload`, `fp64_abs`, `mbedTLS_SSL_Connection_Init`, `pow`,
  `Meter_ParseUDPMeterInfo` (log text only)
- **Only rename of same global variable** (no logic change):
  `HTTP_EcoTracker_ParsePower`, `HTTP_Economy_TOU_ParseResponse`, `HTTP_P1Meter_ParseActivePower`,
  `HTTP_StormOneMeter_ParseData`, `Cloud_HTTP_Response_Parser` (e.g. `g_meterProtocolClass`→
  `g_meterConnectReadyState`, `g_meterSourceChangeFlag`→`g_meterDataFreshFlag`),
  `Meter_DisconnectWatchdog_Check`, `MQTT_Publish_Meter_Data`
- **Field offsets as consequence of BMS pack record enlargement** (0x28→0x2c per pack), no own logic: `CAN_AA_StoreUnitFieldPair0`, `CAN_AA_StoreUnitFieldPair8`, `CAN_AA_StoreUnitFieldPair16`,
  `CAN_AA_StoreUnitFieldPair24_Notify`, `CAN_AA_StoreUnitFieldPair64`, `CAN_AA_StoreUnitFieldPair72`,
  `CAN_AA_StoreUnitIndexedFieldPair`, `CAN_Detect_Mismatched_Nodes`, `BMS_CellVoltage_MinMax_PerPack`,
  `BMS_Version_Consistency_Check`, `MQTT_Publish_BMS_Data`
- **Only informative display values / comments / build info without branch change**:
  `ShellCmd_get_ver` (build date, `ems_ver` 0x93→0x95)
- **Pure literal pool/address shift** (text in decompilat identical, only assembly addresses different): none counted doubly above — mainly affects register allocation cases noted above.

## New Functions (only v149, 25)

**EEPROM I2C bus recovery** (4 functions, one cohesive mechanism): `EEPROM_I2C_BusRecovery`
(timeout callback: logs, calls following two plus `EEPROM_I2C_Peripheral_Init`, called from
`EEPROM_I2C_ReadBytes`/`WriteBytes`), `EEPROM_I2C_ClockDisable_Reset` (I2C1 periph reset, APB1 clock off,
AFIO remap reversed), `EEPROM_I2C_GPIO_ClockPulseRecovery` (SCL as GPIO, 9× 5 µs high/low —
classic manual I2C bus recovery procedure), `Delay_BusyWait_Ms` (general ms busy-wait for
QSPI code and `Time_RtosAwareDelayMs`, not used by I2C chain itself — uses `Delay_BusyWait_Us`).

**MQTT/cloud config parser** (7 functions, new feature): `MQTT_Config_Command_Handler`
is new top-level parser for `key=value;…` config command received via cloud/HTTP
(work mode, BLE advertise enable, feed-in flag `fu=`, local API enable+port `ap=`,
firmware/version number `sv=`, max charge/discharge power `mc=`/`md=` with 2500 W fallback on parse error, WLAN SSID/password `ws=`/`wp=`, DOD value `dv=` via `Config_Write_DischargeCutoffSOC`,
HTTP server type `ht=`, backup enable). Uses typed extractors `MQTT_Config_ExtractU8/U16/
U32Value` (strstr+atoi), `MQTT_Config_ExtractStringValue` (copy to `;`), `MQTT_Config_ExtractFields`
(generic loop over 6-entry table, e.g. for energy counter fields like
`g_totalChargeEnergyWh`) and `MQTT_Config_ParseScheduleEntries` (up to 10 TOU entries `nm0=…` with
subfields `bt=`/`et=`/`wk=`/`vv=`/`as=`, times via `Parse_TimeString_To_HourMinute`
("HH:MM" parser with range check), target is active TOU table `g_touSlot0WeekdayMask`, only
when `g_mqttCfgWorkMode==1`). Callers: `HTTP_ClientTask_Dispatcher` and
`HTTP_Task_StateMachine_Handler` — feature comes via HTTP cloud requests, not directly via MQTT client.

**CAN bus** (3 functions + 1 thematically related): `CAN_AA_StoreUnitFieldPair80` (protocol 0xAA
handler, writes 8-byte value pair at offset 0x50 of 88-byte pack record in
`g_bmsPack0ChargePower`/`DischargePower`; no text caller found, likely via dispatch table), `CAN_Broadcast_Category0x17_PackedValues` (energy counter categories 3/4 as
CAN broadcast category 0x17), `CAN_Broadcast_ClearCategories0x40to0x43` (four null-payload frames
for categories 0x40–0x43), `Config_Param0A_Relay` (writes relay control value in CAN parameter 0x0A,
called from `Config_Set_DischargeCutoff_WithRelay`).

**Quectel modem/UART** (3 functions + `StrCat`): `Quectel_AT_SendRaw_WithMutex` (mutex-protected
AT sending with wait for response), `Quectel_AT_QISEND_HexData` (builds `AT+QISEND=<socket>,<len>,
"<hex>"` from byte buffer, called from `TCP_FC06_WriteSingle_Handler` and `FC03_Read_Handler`
— Modbus responses can now be sent via cellular modem as hex payload),
`Quectel_UART_RxByte_Parser` (byte assembler for incoming modem UART data, CRLF detection,
special case for `"+QIURC: \"recv\",11"`, called from `Quectel_UART_RxByte_ISR_Handler`),
`StrCat` (trivial strcat implementation, appends closing quote to hex string).

**Shell commands** (2): `ShellCmd_get_backup_datas` (sets `g_backupDataRequestFlag`),
`ShellCmd_power_check` (outputs CT/Venus/BMS/load/PV/backup power, including via
`Inverter_PVString_Channel_ValueDelta_Calc`). For both no text caller found (registration
via shell command table).

**Miscellaneous/telemetry** (2): `Inverter_PVString_Channel_ValueDelta_Calc` (base value + delta for active PV string channel 1–4 from CT/grid/island power, called from
`BLE_Build_DeviceStatus_Response`, `MQTT_StatusReport_BuildAndPublish`, `ShellCmd_power_check`),
`Telemetry_Store_EnergyCounters` (writes 8-byte energy counter record to one of four slots,
selector `'@'/'A'/'B'/'C'`, called from `Telemetry_Register_Dispatcher`),
`MQTT_Publish_DeviceIDs` (publishes `"cd=%d, VID=%s, XID=%s GID=%s"`, called from
`MQTT_CtrlCommand_Dispatcher` case 0x3e).

## Removed Functions (only v147, 2)

- **`Config_Apply_WorkModePowerPreset`**: mapped `g_workMode` 1/2/3/4 to `g_maxDischargePowerW`
  800/600/2200/1200 W (fallback 800 W). Callers in v147: `Config_ClampDefaults_ParamBlock`,
  `BLE_Build_DeviceStatus_Response`, `MQTT_Telemetry_Struct_Builder`,
  `MQTT_StatusReport_BuildAndPublish`. All four callers exist in v149, but call is
  removed everywhere without replacement — `Config_ClampDefaults_ParamBlock` keeps only
  plausibility fallback (0 or > 2500 W → 800 W/mode 1), other three now read
  `g_maxDischargePowerW`/`g_workMode` directly without recalculation. Fixed preset table
  no longer exists in v149 anywhere; values apparently maintained via new
  `MQTT_Config_Command_Handler` (fields `md=`/`mc=`).
- **`MQTT_Config_VID_Clear`**: cleared 0xE4-byte MQTT VID/credential block in external SPI flash
  at 0x315000 (same block `MQTT_Config_VID_Init` writes). Callers in v147: only
  `Quectel_MQTT_Session_StateMachine`, at three places in error/reset branches (periodically every
  600 ticks in XID branch). In v149 function no longer exists, and at corresponding
  state transitions no replacement call — periodic clearing of VID/XID block on errors was removed entirely (independently verified).

## Unchanged (explicitly checked)

OTA/BLE-OTA/YMODEM (all function names fully present and address-independent in both versions),
cloud hosts (`hamedata.com`), debug shell unlock via Modbus 46000 = 0x5100 (in both versions present!), TCP shell 8091, fixed WLAN credentials SM471/Hame147258, MQTT TLS authmode 2,
Base64URL_Encode (present, only removed in v150). Shell commands: **65 in v147, 67 in v149**
(+`get_backup_datas`, +`power_check`; old figure "63" was imprecise).

## Open Points

Changes whose location is definitive, but whose technical meaning cannot be safely determined from code alone —
deliberately **not guessed here**:

- `Energy_TOU_Counters_Update`: field offset shift by 0x18 (24 B) in `energy_ctx` struct — which
  new fields are responsible unclear.
- `HexChar_To_TimeOffsetIndex`: completely different input source and algorithm (3 hex chars/mod 10 →
  1 hex char/mod 5) — technical effect on TOU time window calculation open.
- `CAN_CommLink_Timeout_DataClear`: nulled inverter state block grows by 4 bytes, independent of
  BMS pack growth — content of new 4 bytes open.
- `ShellCmd_get_all_bms_data`: `mos_flag` stride switches independent of BMS pack enlargement from
  0x50 to 0x58 — affected struct not identified.
- `Surplus_FeedIn_Enable_Check`: packed `reserved` field filled differently, but exact
  bitfield semantics unclear.
- `Time_RtosAwareDelayMs`: callee switches from `Delay_BusyWait_Us` to `Delay_BusyWait_Ms` (same
  numerical value) — whether this is conscious, significantly longer delay or oversight is open.
- `Quectel_AT_SendCommand_Locked`: RX mode constant `g_quectelAtRxMode` changes from 5 to 6 —
  meaning of modes not documented.
- `sscanf_Format_Parser`: return value/EOF behavior changes fundamentally — possible this stems
  from compiler/library update rather than conscious firmware change.
- `MQTT_StatusReport_BuildAndPublish`: two new report fields `lf=`/`pl=`, whose arguments currently
  always 0 (likely placeholders for future feature).
- `MQTT_JSON_RPC_Dispatcher` case 7 "Manual": new flag `g_aiSchedule9Enable` reset — meaning of "Schedule 9" unclear.
- `Protocol_AA_CommandDispatch`: dispatch table grows from 15 to 16 entries — which new
  0xAA command this is, was not determined.
- `CAN_AA_StoreUnitFieldPair80`, `CAN_Broadcast_Category0x17_PackedValues`,
  `CAN_Broadcast_ClearCategories0x40to0x43`, `ShellCmd_get_backup_datas`, `ShellCmd_power_check`:
  no text-searchable caller in source set (likely referenced via dispatch/command table outside
  searched files).
- Modbus register 36000: Ghidra names underlying global `g_warnCode` in v147, `g_vnsInvChargeLimitPower` in v149
  (same relative offset). `g_warnCode` indexed elsewhere as array base address (`Protocol_AA_SetChannelData`) — suggests naming artifact of auto-naming rather than real function change of register, but not confirmed in code. Four more registers (30000, 30028, 30029, 30035) show same kind of differing fallback naming (`g_paramStore_*`) — likewise no functional finding.
- `BLE_App_Command_Dispatcher`: new case 'Z' (0x5A) falls through without `break` into case '[' (0x5B),
  i.e. old case 'Y' (0x59) also triggers backup/network-info logic — whether intentional is unclear.
- `MQTT_JSON_RPC_Dispatcher`: response buffers of individual methods dedicated in v149 instead of
  like v147 overlapping with offsets — functionally likely equivalent, but relevant if other
  functions still reference old offset addresses (not checked).

## Data Basis

- complete name-based pairing: `scratchpad/v151/l147_149.json` (local only, not in the repo) (1782 identical, 124 changed,
  2 only v147, 25 only v149); Modbus descriptor comparison `mbdesc_147.json`/`mbdesc_149.json`
- decompilat + listing: `Dekompilate/VNSD_Control_147/`, `Dekompilate/VNSD_Control_149/` (local only, not in the repo)
  (`listing/globals_and_registers.txt` per version for named globals)
