# Diff Table Control v149.2 → v150 (Venus D)

> 🇩🇪 Deutsche Version: [VNSD_Control_Diff_v1492_v150_de.md](VNSD_Control_Diff_v1492_v150_de.md)

**As of:** 2026-09-30. Both images with correct base 0x08004800 in Ghidra, 100 % named.
Addresses: v149.2 → v150. Function pairs via identical names (1934), instruction-wise compared:
**56 changed** (of which **26 substantive**), 1 only in v149.2, 20 only in v150.
**German:** `VNSD_Control_Diff_v1492_v150.md` · **Predecessor Diff:** `VNSD_Control_Diff_v149_v1492.md`

## Classification

v149.2 (beta, 2026-07-02) and v150 (2026-08-05) are only 5 weeks apart, but differ in code significantly more than the small version jump suggests: v150 brings completely new HTTPS/TLS client (mbedTLS over CH395), fundamentally restructures cloud telemetry upload (JSON instead of AES+Base64URL, new domain), introduces "peak shaving"/power clamping function (EEPROM 0x394, new BLE/MQTT commands) and fixes several ethernet/modem stability issues via two new watchdogs.
For v150 a manufacturer changelog exists (`CHANGELOG.de.md`, local only, not in the repo); mapped to code point by point below.

**Correction from prior version of this document:** Previous version was based on incomplete function pairing, found only 9 different functions and falsely concluded
"no changelog present, no Modbus/BLE changes." Actually a changelog with 5 points exists, 20 new and 1 removed function plus 26 substantive changes present. Two of three previously listed "real" changes were also misinterpreted (see "Cosmetic" below).

## Checklist

| Question | Result |
|---|---|
| Modbus registers | **register table itself unchanged** (all 246 descriptors: register number, type, scaling, count identical between v149.2 and v150 — only at 215 entries RAM shadow address shifts because config struct grew by 6 bytes, see peak shaving below). Schedule EEPROM of registers from 46500 moves from 0x36BF to 0x3700 (`Config_LoadOrInit_DeviceSettings`, `Write_Handler`) |
| BLE | **extended**: new command `)` (0x29) "set peak shaving" (clamp mode + watt value, EEPROM 0x394); `BLE_Build_DeviceStatus_Response` additionally reports clamp mode + value (response length 0xA4→0xA7 bytes); `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` additionally report BMS self-test and MOS flag (0x50→0x51 bytes); fallthrough bug command `Y`→`Z` fixed (missing `break`) |
| AC feed-in 2500 W | **hard limit unchanged** (`Config_ClampDefaults_ParamBlock`, `Grid_Export_Power_Limiter` remain byte-identical). Independently new: configurable peak shaving clamp value (EEPROM 0x394), acts additionally in `Inverter_PowerSetpoint_DeadbandClamp` and `Remote_Power_Setpoint_Process` |
| BLE-OTA | **continues to work** (OTA/YMODEM/`BLE_Cmd_OTA_*` completely byte-identical, none of these functions in changes list) |
| Cloud domains | `hamedata.com` endpoints themselves **unchanged** (identical strings in both versions). Telemetry upload (`Cloud_UploadTelemetryData`) however switches from `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (AES+Base64URL-encoded query parameter) to `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (plaintext JSON POST body over new HTTPS client) |

## New Functions (20)

**HTTPS/TLS client for cloud upload** (CH395 + mbedTLS, new, parallel to existing plaintext path):
- `HTTPS_FetchUrlData` – orchestrates complete HTTPS POST: path/host from URL, `HTTPS_PackPostRequest`,
  optionally DNS, `CH395_TCP_Socket_Open_HTTPS` (port 443), `CH395CMDGetSocketStatus` poll,
  `HTTPS_TLSSessionInit`, send/receive over retry wrappers, optional `HTTPS_ParseData`, then
  `HTTPS_TLS_Context_Reset` + socket close. Sole caller: `Cloud_UploadTelemetryData`
- `HTTPS_TLSSessionInit` – builds mbedTLS handshake (SSL/config/CTR-DRBG/entropy, TLS 1.2 fixed,
  SNI, up to 31 handshake retries)
- `HTTPS_TLS_Context_Reset` – sends `close_notify` if needed (`mbedtls_ssl_close_notify`) and frees all
  mbedTLS contexts
- `HTTPS_PackPostRequest` – builds HTTP/1.1 POST headers (host, connection,
  `Content-Type: application/json`, `Content-Length`) around JSON body from `g_cloudUploadBodyBuf`
- `HTTPS_ParseData` / `HTTPS_ParserOnBody` – initializes `llhttp` over HTTPS response, copies up to
  0x4B0 bytes body to `g_httpsBodyBuf`
- `CH395_TCP_Socket_Open_HTTPS` – opens CH395 TCP socket on port 443
- `CH395CMDGetSocketStatus` – reads socket status via SPI command 0x2F
- `mbedTLS_SSL_Send_WithRetry` / `mbedTLS_SSL_Recv_WithRetry` – wrapper around `mbedtls_ssl_write`/`_read`
  with retry on `WANT_READ`/`WANT_WRITE`
- `mbedtls_ssl_close_notify` – standard mbedTLS library function, newly linked

**Peak Shaving / Power Clamp Mode** (EEPROM 0x394, 6 bytes):
- `Config_Read_PowerClampMode_0x394` / `Config_Write_PowerClampMode_0x394` – byte 0 = mode (0/1,
  invalid values reset to 0 on read), bytes 2–3 = clamp power in watts, rest reserved. Read once at boot (`Config_LoadOrInit_DeviceSettings`), written via
  BLE command `)` and MQTT command `0x3f`
- `MQTT_Publish_Grid_Status` – new MQTT response (command `0x41`) with grid/battery status including
  `g_minPowerLimit`/`g_maxPowerLimit`, grid voltage/frequency/power, error/warning codes; same
  control topic as other `MQTT_Publish_*` functions

**Miscellaneous infrastructure:**
- `Quectel_SSL_ConfigVersion` – sends `AT+QSSLCFG="version",<ctx>,<tls_version>` to cellular modem;
  new step in `Quectel_MQTT_Config_StateMachine`
- `Quectel_HTTP_POST_SendAndVerify` – POST pendant to `Quectel_HTTP_AT_SendAndVerify` for modem-side
  `AT+QHTTPPOST` sequence in `HTTP_Task_StateMachine_Handler`
- `FreeRTOS_xPortGetFreeHeapSize` / `FreeRTOS_xPortGetMinimumEverFreeHeapSize` + new shell command
  `ShellCmd_FreeRtos_mem` – outputs free/minimal-free FreeRTOS heap via console
- `__vfscanf` – C library routine, newly linked (one scanf code path reached for first time in v150;
  triggering call site not localized)

## Removed Function

- **`Base64URL_Encode`** (only in v149.2, sole caller was `Cloud_UploadTelemetryData`): v149.2 AES-128-ECB-encrypted
  telemetry packet (`AES128_ECB_Encrypt`, key via `KeyDerive_CopyAndROT`) and Base64URL-encoded to fixed
  0x960-byte buffer, sent as query parameter `v=` to `setVenusDReporting`. v150 entirely omits
  encryption and encoding: packet only wrapped in `{"d":"..."}` and sent plaintext via
  HTTPS POST to new endpoint (transport security via TLS). See "Real Changes" Nr. 1.

## Real Changes (26)

**1. Cloud telemetry upload fundamentally overworked — and buffer overflow in v149.2 fixed** —
`Cloud_UploadTelemetryData` (0801af64→0801b398)
Payload now JSON `{"d":"...&soh=%d"}\r\n` instead of raw query string; encryption/encoding
(AES128_ECB_Encrypt + Base64URL_Encode) entirely removed; target switches from
`http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (GET, query param) to
`https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (POST body); when ethernet active now
calls `HTTPS_FetchUrlData` (new TLS client) instead of previous `HTTPS_POST_Request`, on cellular
still `Quectel_HTTP_AT_SendAndVerify`.
**Specific bug found in v149.2:** `Base64URL_Encode` called with claimed target buffer size
of `0x960` (2400 bytes), but `g_cloudUploadBodyBuf` per globals reference only
`uchar[1200]` (1200 bytes). `Base64URL_Encode` only checks if *passed* size suits output length (`param_4 < (param_2+2)/3*4`), not if it fits actual buffer — on AES outputs whose Base64 encoding falls between 1200 and 2400 bytes, function wrote past end of `g_cloudUploadBodyBuf` into adjacent globals. v150 removes entire encoding step and copies JSON body exact to sprintf length (no fixed encoding buffer anymore). Explains "Fixed data loss from overly long HTTP records" directly as memory overflow fix, not just efficiency gain.

**2. Two new connection watchdogs plus GET→POST restructure of cellular upload**
- `HTTP_ClientTask_Dispatcher` (08019ff0→0801a3d0): new block – if upload backlog > 1 and no progress for 0x708 (1800) ticks and `g_oldStep`==0, triggers `CH375_Reset_And_Reinit(0)`
  (ethernet chip reset) and resets dispatcher state (log "[HTTP]ch395 reset!!!!")
- `HTTP_Task_StateMachine_Handler` (080109e0→08010c20): new block – if upload backlog exists
  and no progress for 900 ticks, triggers `Quectel_Modem_HardwareReset_Handler(0)`
  (cellular modem reset) (log "[HTTP]fc41d reset!!!!"). Additionally states 6–9 of this
  state machine restructured: v149.2 used `Quectel_HTTP_GET_Request`/`Quectel_HTTP_ReadResponse_Verify`
  (GET-based, including special case for modem firmware "2025"); v150 instead sends first
  `AT+QHTTPCFG="header","Content-Type","application/json"`, then `AT+QHTTPPOST=%d,60,60` followed by
  `Quectel_HTTP_POST_SendAndVerify(...,"CONNECT",...,g_cloudUploadBodyBuf,...)` — real POST with
  JSON body from Nr. 1 instead of GET

Fits "Fixed incorrect sending of local API in ethernet mode" /
"Fixed failed HTTP updates in ethernet mode" and complements Nr. 1 for cellular part of protocol restructure.

**3. `DeviceInfo_BuildStatusString`** (0802ab44→0802b470): hardware variant tag for 6 more meter types. Previously delivered generic EEPROM string for **all** meter types
(`Config_Read_String_0x387`) as hardware variant of local API. v150 adds fixed tags for `g_meterSourceType` 3,
4, 8, 9, 10, 11 (`HME-4`, `HME-3`, `SMR-0`, `SMR-1`, `SMR-2`, `TPM2-0`) – remaining types fall back to EEPROM string. Directly explains "Fixed incorrect sending of
local API in ethernet mode".

**4. `Grid_PowerSync_StateMachine`** (08030f84→08031b60): meter type classification for
CT direct adoption extended. Group of `g_meterSourceType` values taking "without
deviation gate, adopt directly" branch expands from narrow list (`2, 9, 10, 12, 4–7`) to
broad range (`2, 12, 3–11, 14+`). Fits "Switched CT_TYPE meter connection".

**5. Peak Shaving/Power Clamp Mode** (new, EEPROM 0x394, 6 bytes) — several places:
- `Config_LoadOrInit_DeviceSettings` (0802d174→0802dcbc): reads clamp mode at boot
  (`Config_Read_PowerClampMode_0x394`); surrounding EEPROM config block (base 0x300) grows from 0x92
  to 0x98 bytes for new field; schedule EEPROM (`g_canScheduleRegCache`) independently moves from
  0x36BF to 0x3700 (identical in `Write_Handler`, 08055720→08056514)
- `Inverter_PowerSetpoint_DeadbandClamp` (080184c8→0801877c): with active clamp mode (byte 0 = 1),
  setpoint only raised downward to limit (floor) instead normally pressed upward to limit (ceiling)
- `Remote_Power_Setpoint_Process` (0802dc58→0802e7b0): discharge power calculated from schedule (TOU)
  additionally limited to `current CT grid power (g_ctPowerNow) − configured clamp value` with active mode
- `BLE_App_Command_Dispatcher` (0800c758→0800c720): new command `)` (0x29) sets/reads mode+value
  via `Config_Write_PowerClampMode_0x394` + `Config_Write_PowerOffset`; also fixes
  fallthrough bug (missing `break` after command `Y`, unwanted fallthrough to `Z`)
- `BLE_Build_DeviceStatus_Response` (0800f778→0800f824): reports mode+value additionally
  (`BLE_Send_Response` length 0xA4→0xA7)
- `MQTT_CtrlCommand_Dispatcher` (08023c60→0802423c): new command `0x3f` "set peak shaving"
  (parameters `as=`/`vv=`), old command `0x3e` (`MQTT_Publish_DeviceIDs`) moved to `0x40`, new
  command `0x41` "get INV data" calls `MQTT_Publish_Grid_Status`
- `MQTT_StatusReport_BuildAndPublish` (08029cc0→0802a5dc): status line gets new fields
  `peak_status=%d,peak_power=%d`

Fits changelog point "Peak shaving function added".

**6. BMS self-test and MOS flag additionally reported** (independent of peak shaving)
- `BLE_Build_BMS_Data_Response` (0800fc30→0800fcf4): new field `g_bmsSelfCheckFlag`
  (= `g_pgn1007InfoSelfCheck`, CAN PGN-1007 self-check), response length 0x50→0x51
- `BMS_StatusQuery_Response_Send` (080238d0→08023e98): status line gets `self_check=%d,mos=%d`
  (latter = `g_bmsPack0MosFlag`); stack frame thereby 0x98→0xa0

**7. `Config_Read_String_0x387`/`Config_Write_String_0x387` — off-by-one bug in EEPROM address
fixed.** Both functions accessed EEPROM 0x388 instead of 0x387 in v149.2 despite their name
(13-byte alias name record occupied 0x388–0x394). v150 corrects to 0x387 (0x387–0x393) —
likely needed so new 6-byte clamp mode record at 0x394 (Nr. 5) doesn't collide with alias name end.

**8. `HTTPS_POST_Request`** (080195c0→08019894): receive buffer enlarged and consolidated with HTTPS path.
Nulled area for response grows from 0x1958 (6488) to 0x1DC0 (7616) bytes;
`HTTPS_POST_ReceiveResponseData` now receives into shared buffer `g_httpsRespBuf` instead of previous `g_httpRxBuf`. Fits equally "Fixed data loss from overly long HTTP records".

**9. `HTTP_ParseUploadResponseCode`** (08019dd0→0801bf4c): different JSON key. Now searches
`"code":` (7 chars) instead of `"de":` (5 chars) in upload response — not just a two-byte-shifted offset like claimed in prior version of this document, but completely different
JSON key.

**10. `MQTT_Publish_Network_Info`** (08026f18→080276fc): in short status branch now reports
`g_meterResolvedIpStr` (actually meter-returned IP) instead of `g_meterQueryIpStr`
(queried IP) — as detailed branch already did anyway.

**11. `Quectel_MQTT_Config_StateMachine`** (08012f94→080131c4): TLS config sequence changed,
new step. Sequence now: `Quectel_SSL_ConfigVerifyLevel` first (instead `ConfigSNI`), new
intermediate step `Quectel_SSL_ConfigVersion` (TLS version), `Quectel_SSL_ConfigSNI` now last
(parameter 2→1).

**12. `Quectel_MQTT_Session_StateMachine`** (08012808→08012a30): factory reset in SSL watchdog now conditional. Works reset after 1800 s SSL inactivity only when `g_oldStep`
(work mode intermediate step) == 0; plain modem hardware reset still runs always.

**13. `mbedTLS_SSL_Connection_Init`** (0801cb64→0801d134): removed duplicate TLS reset. Called
`MQTT_TLS_Context_Reset()` on failure removed because caller (`MQTT_Session_Init`
→ `CH395_MQTT_Socket_Close_And_TLS_Reset`) already calls it — pure redundancy cleanup.

**14. `Battery_EPS_Reserve_SOC_Hysteresis`** (0800add0→0800ad3c): one byte in CAN 0xCF report changed.
Lower byte of sent 16-bit field always 0 in v149.2, always 1 in v150
(`CONCAT11(count,1)` instead `count<<8`). Bit value meaning open.

**15. `Cloud_EdgeDetectAndWatchdog`** (080312f4→08031ebc): retry additionally bound to MPPT channel. Retry
(`Cloud_Response_Action(1)`) on edge case 3 now only when `g_mpptActiveChannel` 0 or 5 (previously unconditional).

**16. `Quectel_HTTP_AT_SendAndVerify`** (080104c4→080106f4): additional delay. `vTaskDelay(delay_ms)`
right after sending AT command added (before notify wait) — gives modem more ramp time.

## Cosmetic (no behavioral difference)

- **Embedded firmware version literal** (0x5D4 = 1492 → 0x96 = 150) routinely updated, purely
  informative: `Config_LoadOrInit_DeviceSettings`, `ShellCmd_get_ver`, `HTTP_Cloud_Reporting_Dispatcher`,
  `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `MQTT_Transport_NullCallback`/`Util_NoOp_Stub`
  (data literal in constant block)
- **`__LINE__` log macro shifts** (Log_Printf line argument, no effect), verified through
  identical surrounding code: `CH395_ExtractUdpData`, `CH395_TCP_Socket_Open_HTTPPort80` (**correction:**
  previously wrongly rated as real parameter change 0x24→0xAA — function body byte-for-byte identical, value is only log line argument), `CT_GridPower_Controller`,
  `Device_Network_Info_Init`, `HTTP_PackGetRequest` (**correction:** previously wrongly rated as
  buffer enlargement 0x168→0x280 — `out_size` remains unchanged function parameter, value is also only log line argument), `Meter_GetMeterIP_FromResponse`, `Meter_ParseUDPMeterInfo`,
  `MQTT_Connect_And_Subscribe`, `MQTT_TLS_Context_Reset`, `Parse_IP_Address_String`,
  `Quectel_GetDeviceIP`, `Quectel_UDP_Socket_Open`, `SMR_TIC_SelfUse_PowerController` (9 constants, all
  shifted by exactly +0xE)
- **Compiler/decompiler noise**: `MQTT_Publish_Meter_Data` (only type annotation `pointer` →
  `undefined *`), `llhttp_errno_name` (in the earlier v149.2 program the HPE_* string table behind the
  function was disassembled as code, which the decompiler showed as an unrecovered jump table in the `default` case;
  corrected 2026-10-12 to the v150 state — error name table itself unchanged), `pow`/`fp64_abs`/`__scatterload`/`sscanf_Format_Parser`/`vPortStartFirstTask`/
  `SPI_WriteByte`/`ShellCmd_reset` (literal pool shifts, register allocation)
- `HTTP_ParserOnBody`/`HTTP_ParserOnMessageComplete`/`HTTP_ParseData` (0804d8b4/0804d924/0802b130): rename
  `g_httpRespBodyBuf` → `g_httpBodyBuf` affects same address (0x6001fa68, naming loss only this round), copy limit stays exact 0x4B0 (1200 bytes); remaining changed constants (0xd3/0xdc/0xe8) also only `__LINE__` arguments of associated
  Log_Printf calls. Actual new separate `HTTPS_ParserOnBody`/`g_httpsBodyBuf` instance for HTTPS path
  listed under "New Functions"
- Name swap `g_bleDevStatusPvMinusGridEst`/`g_bleDevStatusLoadPowerEst` in
  `BLE_Build_DeviceStatus_Response`: same struct offsets, same formulas — only analysis-internal
  rename, no real field swap

## Changelog v150 → Code

| Changelog Point | Code Finding |
|---|---|
| Fixed incorrect sending of local API in ethernet mode | `DeviceInfo_BuildStatusString`: 6 new hardware variant tags for meter types 3/4/8/9/10/11 (see "Real Changes" Nr. 3) |
| Fixed failed HTTP updates in ethernet mode | new CH395 and modem watchdogs plus GET→POST restructure of cellular upload in `HTTP_ClientTask_Dispatcher`/`HTTP_Task_StateMachine_Handler` (Nr. 2) |
| Peak shaving function added | New EEPROM 0x394 + BLE command `)` + MQTT commands 0x3f/0x41 + clamp logic in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` (Nr. 5) |
| Fixed data loss from overly long HTTP records | `Cloud_UploadTelemetryData`: fixed buffer overflow in `Base64URL_Encode` (claimed capacity 2400 B vs actual buffer 1200 B), complete switch to JSON/HTTPS without AES+Base64URL (Nr. 1); `HTTPS_POST_Request` receive buffer 6488→7616 bytes (Nr. 8) |
| Switched CT_TYPE meter connection | `Grid_PowerSync_StateMachine`: meter type range for direct adoption extended (Nr. 4) |

## Open Points

- `Battery_EPS_Reserve_SOC_Hysteresis`: meaning of newly set low byte in CAN 0xCF report not determined
- `__vfscanf` (newly linked C library function): triggering call site not localized
- `ShellCmd_FreeRtos_mem`: embedded via shell command linker section instead normal call table, no code cross-reference found (function itself unambiguous: outputs free/minimal free FreeRTOS heap)
- Fallthrough fix BLE command `Y`→`Z` (Nr. 5): whether observable v149.2 misbehavior triggered by this or consequence unverified

## Data Basis

- Instruction diff: `scratchpad/v151/l1492_150.json` (local only, not in the repo) (1878 identical, 56 changed, 1 only v149.2, 20 only v150)
- Modbus descriptors: `scratchpad/v151/mbdesc_1492.json`, `mbdesc_150.json`
- Decompilat + listing: `Dekompilate/VNSD_Control_1492/`, `Dekompilate/VNSD_Control_v150_base4800/` (local only, not in the repo)
