# Diff Table Control v150.9 → v151 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_Control_Diff_v1509_v151_de.md](VNSA_Control_Diff_v1509_v151_de.md)

**As of:** 2026-10-11. Both images with correct base 0x08004800 in Ghidra, fully named (v150.9: 1941 functions, v151: 1942).
Addresses: v150.9 → v151. **Main documentation:** [VNSA_Control_FW_v151_Full_Analysis.md](VNSA_Control_FW_v151_Full_Analysis.md)
Function pairs via identical names (1941), compared instruction by instruction with call targets by name, referenced strings and
immediate values: **17 changed** (16 substantive, 1 call offset only), **10 changed only in relocated table/literal addresses or the version string**,
**1 only in v151**, **0 only in v150.9**.

## Verification checklist

| Question | Result |
|---|---|
| Modbus registers | Unchanged: descriptor table byte-identical (246 entries), `Write_Handler`, `FC03_Read_Handler`, `Read_Serializer` instruction-identical; 30200 reads 151 |
| BLE | Unchanged: `BLE_App_Command_Dispatcher` instruction-identical |
| AC feed-in limit | Unchanged: 1500 W everywhere (Modbus 44003 > 1500 W rejected, BLE 0x16/0x17, MQTT, JSON-RPC, remote command), export limiter clamp 2400 W |
| BLE OTA | Still possible, OTA/YMODEM code unchanged (model check `VNSA-0`, bin-type check already in v150.9) |
| Cloud domains | `getDateInfo`: `http://<region>.hamedata.com` → `http://api-<region>.marstekcloud.com`; upload and all other hosts unchanged |
| Pack capacity | Unchanged: 7 per-pack records in the Control, 13 cells scanned for 6 packs; A BMS 1105 sends packs 1..6 |
| Non-functional registers | All 216 A registers (FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL) still as classified: none of their Control-side causes changed |

## Release note → code

| Release note item | Code finding |
|---|---|
| 1 Modbus TCP stability | `CH395_ConfigTcpKeepalive_Defaults` keepalive idle 10 000 → 60 000 ms; `HTTP_ClientTask_Dispatcher` connection watchdog limit 3600 → 14 400 then `System_Reboot()`, new suppression flag `g_httpWatchdogSuppressFlag` |
| 2 DOD abnormal state | no change against v150.9 (already contained, see [VNSA_Control_Diff_v150_v151.md](VNSA_Control_Diff_v150_v151.md)); `CT_GridPower_Controller` gains the D v151 guard resetting a positive CT setpoint to 0 |
| 3 YMODEM upgrade type check | no change against v150.9 (already contained) |
| 4 AI mode check / query current AI strategy | `AI_ScheduleQuery_Response_Send` paginated, new helper `MQTT_CtrlResponse_Transmit` |
| 5 Pre-release server URL error | `getDateInfo` host to `api-<region>.marstekcloud.com`, region prefixes `dev`/`staging`, server types 0–4, type loaded at boot |
| 6 HTTPS upload stability | `Quectel_HTTP_POST_SendAndVerify` rewritten, `AT+QHTTPPOST` 120/120 s, outer timeout 6000, TOU `crc2` |
| 7 CT communication 1.1 s | `Modem_SendStatusTelemetry` / `Ethernet_SendStatusTelemetry` period 2000 → 1100 ticks |

## Changes

| Area | v150.9 | v151 | Location (v150.9 → v151) | RN |
|---|---|---|---|---|
| Version string / build | `VNSA-0 v1509`, Sep 8 2026 09:34:19 | `VNSA-0 v151`, Sep 28 2026 09:54:57 | `CLI_InitSession` 08051CE4 → 08051E04, `ShellCmd_get_ver` 0803B68C → 0803B7AC | – |
| Version word / type code | 0x5E5 (1509) | 0x97 (151) | 0x08010000; `Config_LoadOrInit_DeviceSettings`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `ShellCmd_get_ver` | – |
| `getDateInfo` host (Quectel path) | `http://%s.hamedata.com/app/neng/getDateInfo%s.php` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php` | `HTTP_Cloud_Reporting_Dispatcher` 08012394 → 08012424 | 5 |
| `getDateInfo` host (CH395 path) | same hamedata URL, server types 0–2 | marstekcloud URL, server types 0–4 (others → 0) | `HTTP_GetUrlData` 0801986C → 08019910 | 5 |
| Region prefix table `g_cloudDomainUrlTable` (0x20000FC8) | eu, www, us, eu-dev, eu-staging | eu, www, us, dev, staging | .data, string table 0x08060220… | 5 |
| Boot-time EEPROM loads | 0x441 and 0x395 not loaded at boot; no first-boot default for 0x395 | 0x441 (server type) and 0x395 (SEL LED) loaded; first boot writes 0x395 = 1 | `Config_LoadOrInit_DeviceSettings` 0802DA34 → 0802DB28 | 5 |
| HTTP POST via module | blind wait, no ERROR/`+QIURC` check | response loop, ERROR/`+QIURC`, exact byte count, post-data log | `Quectel_HTTP_POST_SendAndVerify` 08010438 | 6 |
| `AT+QHTTPPOST` timeouts / outer timeout | 60, 60 s / 600 | 120, 120 s / 6000, body length from `g_cloudUploadBodyLen` | `HTTP_Task_StateMachine_Handler` 08010AC0 → 08010B4C | 6 |
| Telemetry upload | – | body length handling as D v151 | `Cloud_UploadTelemetryData` 0801B23C → 0801B300 | 6 |
| HTTPS GET | – | parameter setup as D v151 | `HTTPS_FetchUrlData` 0801BC64 → 0801BD1C | 6 |
| TOU response checksum | one XOR checksum | second checksum `crc2` (same XOR), accepted if either matches; error text with `crc2` | `HTTP_Economy_TOU_ParseResponse` 0801938C → 08019418 | 6 |
| CH395 keepalive idle | 10 000 ms | 60 000 ms | `CH395_ConfigTcpKeepalive_Defaults` 08037964 → 08037A84 | 1 |
| Cloud connection watchdog | limit 3600 | limit 14 400, then `System_Reboot()`; counter held at 0 while `g_httpWatchdogSuppressFlag` = 1 | `HTTP_ClientTask_Dispatcher` 0801A294 → 0801A344 | 1 |
| AI schedule answer (`cd=58`) | one 1024-byte buffer, `AI%d=…`, sent inline | pages of 24 entries, header `cd=%d,pg=%d/%d,`, `AI%u=…`, bounds check, sent via `MQTT_CtrlResponse_Transmit` | `AI_ScheduleQuery_Response_Send` 08023704 → 080237B8 | 4 |
| New function | – | `MQTT_CtrlResponse_Transmit` (sends a `ctrl` response directly over the cellular data channel or as MQTT PUBLISH) | 08023990 (v151 only) | 4 |
| Meter / CT query period | 2000 ticks | 1100 ticks | `Modem_SendStatusTelemetry` 08031418 → 08031538, `Ethernet_SendStatusTelemetry` 08031608 → 08031728 | 7 |
| CT setpoint | – | positive CT setpoint additionally reset to 0 (D v151 guard) | `CT_GridPower_Controller` 08030C10 → 08030D20 | 2 |
| Log line numbers | – | shifted by 3 | `SMR_TIC_SelfUse_PowerController` 0800851C | – |

## Cosmetic differences (no behavioural change)

- `Quectel_AT_SendAndPollResponse` 0800C1CC: one call offset (same callee, same logic).
- Only relocated tables/literals or the new version string: `Task_CreateTasksAndStartScheduler`, `MQTT_JSON_RPC_Dispatcher`,
  `MQTT_CtrlCommand_Dispatcher`, `pow`, `printf_Float_To_Digits`, `printf_Format_Engine`, `llhttp__internal__run`,
  `ssl_derive_master_secret`, `__main_after_scatterload`, `CLI_InitSession` (string `VNSA-0 v151`).

## Unchanged (explicitly checked)

Modbus descriptor table and handlers, BLE dispatcher, all CAN senders/dispatchers, OTA/YMODEM, power limits and classes,
13-cell scan, RS485/USART1 code, task table (14 tasks), shell command table (81 entries), MQTT-RPC table (16 methods), RAM layout
(`.data`/`.bss`/XMC sizes and initial values apart from flash pointers).
