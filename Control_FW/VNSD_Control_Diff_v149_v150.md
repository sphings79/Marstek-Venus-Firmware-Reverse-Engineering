# Diff Table Control v149 → v150 (Venus D) — Major Hop

> 🇩🇪 Deutsche Version: [VNSD_Control_Diff_v149_v150_de.md](VNSD_Control_Diff_v149_v150_de.md)

**As of:** 2026-09-30. Both images with correct base 0x08004800 in Ghidra, 100% named.
This is the major version jump v149 → v150. In between lies the beta **v149.2**, through which
most of the development ran.
**German:** `VNSD_Control_Diff_v149_v150.md`
**Detail hops:** `VNSD_Control_Diff_v149_v1492.md` (v149→v149.2) · `VNSD_Control_Diff_v1492_v150.md` (v149.2→v150)

**Correction vs. earlier version of this document:** The old version was based on an at-that-time
incomplete pairing for the v149.2→v150 step (only 9 differences found) and came to wrong conclusions: BLE allegedly unchanged in command set, cloud domain allegedly completely unchanged, and two alleged "real" changes (`CH395_TCP_Socket_Open_HTTPPort80`,
`HTTP_PackGetRequest`) that are actually only shifted log line numbers. This version is based on
newly created, complete function diffs for both hops plus a direct full comparison
v149 ↔ v150.

## Method

Direct full comparison v149 → v150 (`scratchpad/v151/l149_150.json` (local only, not in the repo), normalized instruction opcodes):
Function pairs via identical names **1955**, of which **82 changed**, **1 only in v149** (`Base64URL_Encode`),
**24 only in v150**, 1848 byte-identical. The two hop docs additionally count separately: v149→v149.2
58 changed (38 substantive/20 cosmetic), 0/4 only A/B; v149.2→v150 56 changed (26 substantive), 1/20 only
A/B. Sums of hops not directly additive to 82 of direct diff due to rebuild/counter-acting effects (see "Open Items" of hop docs); direct diff numbers are authoritative
for overall jump.

## Verification Checklist (v149 → v150 overall)

| Question | Result |
|---|---|
| Modbus registers | **Unchanged.** Descriptor table 246 entries in both states, register number/type/scaling/count identical. Only RAM shadow addresses shift (config struct grew 6 bytes for peak-shaving; relink); schedule EEPROM from register 46500 moves from 0x36BF to 0x3700 (`Config_LoadOrInit_DeviceSettings`, `Write_Handler`) |
| BLE | **Extended, not unchanged.** Device alias field appended to command 0x18 (v149→v149.2); new command `0x29` "Set Peak shaving" (v149.2→v150); `BLE_Build_DeviceStatus_Response` (0xA4→0xA7 B) and `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` (0x50→0x51 B) grew by clamp status and self-test/MOS flag respectively; fallthrough bug command `Y`→`Z` fixed |
| AC feed-in 2500 W | **Hard limit unchanged** — `Grid_Export_Power_Limiter` and `Config_ClampDefaults_ParamBlock` byte-identical over both hops (doesn't appear as changed in direct diff). Additionally, independent of that, new: configurable peak-shaving clamp value (EEPROM 0x394), acts in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` |
| BLE-OTA | **Remains possible.** Complete OTA/BLE-OTA/YMODEM path (`BLE_Cmd_OTA_*`, `YModem_FirmwareUpgrade_Handler`, `OTA_*`) byte-identical over both hops, not in change list of direct diff |
| Cloud domains | `hamedata.com` endpoints themselves **unchanged** (identical strings in v149 and v150, per grep confirmed in both listings: `/app/neng/`, `/ems/api/v1/get...`, `/external-services/...`). But: telemetry upload switches from `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (AES128-ECB + base64URL in query string) to `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (cleartext JSON per HTTPS-POST over new mbedTLS client). Verified: string `setVenusDReporting` no longer appears in any v150 listing, `marstekcloud.com` in no v149 listing |

## Core Items (most important real changes, by topic)

| Topic | Change | Hop |
|---|---|---|
| Cloud upload protocol | AES128-ECB+base64URL query → cleartext JSON per HTTPS-POST to new domain; fixing buffer overflow in `Base64URL_Encode` in process (claimed target size 0x960 B vs. actual buffer 1200 B); `Cloud_UploadTelemetryData` 769→603 instr. | v149.2→v150 |
| Connection watchdogs | New CH395 reset on Ethernet upload stall (>1800 ticks no progress), new modem hardware reset on mobile upload stall (>900 ticks); mobile upload additionally switched from GET to POST | v149.2→v150 |
| Peak-shaving / power clamping | New: EEPROM 0x394 (mode + watt value), BLE command `0x29`, MQTT commands `0x3f`/`0x41` (`MQTT_Publish_Grid_Status` new), clamp logic in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` | v149.2→v150 |
| BMS pack record | Record 88→96 bytes, new SoH field (`g_bmsPack0Soh`), dispatch table 16→17 entries, `CAN_AA_StoreUnitFieldPair88` new | v149→v149.2 |
| BMS additional flags | Self-test flag (CAN PGN 1007) and MOS flag additionally reported via BLE/MQTT | v149.2→v150 |
| CT/SMR-TIC controller | Split into `SMR_TIC_SelfUse_PowerController` (TIC FSM only) + new `CT_PowerSetpoint_Compute` (clamping); pure restructuring, no behavior change | v149→v149.2 |
| Meter classification | `Grid_PowerSync_StateMachine`: meter types with "direct takeover without deviation gate" expanded from narrow list (2,4–7,9,10,12) to wide range (2,3–11,12,14+) | v149.2→v150 |
| Local API hardware tag | `DeviceInfo_BuildStatusString`: 6 fixed tags for meter types 3/4/8/9/10/11 instead of generic EEPROM string for all | v149.2→v150 |
| Device alias | New 12-character alias in EEPROM 0x387/0x388, writable via BLE 0x18 and cloud config; off-by-one bug (access 0x388 instead of 0x387) in follow-up hop fixed | v149→v149.2, fix in v149.2→v150 |
| Various small fixes | `Quectel_MQTT_Session_StateMachine`: factory reset in SSL watchdog only at `g_oldStep`==0; `Cloud_EdgeDetectAndWatchdog`: retry bound to MPPT channel; `Quectel_HTTP_AT_SendAndVerify`: additional delay after AT command; `mbedTLS_SSL_Connection_Init`: redundant TLS reset removed; `Battery_EPS_Reserve_SOC_Hysteresis`: low byte in CAN-0xCF report now always 1 (meaning open) | v149.2→v150 |

## New Functions (24 total)

**v149→v149.2 (4):** `CAN_AA_StoreUnitFieldPair88`, `CT_PowerSetpoint_Compute`, `Config_Read_String_0x387`,
`Config_Write_String_0x387`

**v149.2→v150 (20):**
- HTTPS/TLS client (CH395+mbedTLS, new parallel path for cloud upload): `HTTPS_FetchUrlData`,
  `HTTPS_TLSSessionInit`, `HTTPS_TLS_Context_Reset`, `HTTPS_PackPostRequest`, `HTTPS_ParseData`,
  `HTTPS_ParserOnBody`, `CH395_TCP_Socket_Open_HTTPS`, `CH395CMDGetSocketStatus`,
  `mbedTLS_SSL_Send_WithRetry`, `mbedTLS_SSL_Recv_WithRetry`, `mbedtls_ssl_close_notify`
- Peak-shaving: `Config_Read_PowerClampMode_0x394`, `Config_Write_PowerClampMode_0x394`,
  `MQTT_Publish_Grid_Status`
- Other: `Quectel_SSL_ConfigVersion`, `Quectel_HTTP_POST_SendAndVerify`,
  `FreeRTOS_xPortGetFreeHeapSize`, `FreeRTOS_xPortGetMinimumEverFreeHeapSize`, `ShellCmd_FreeRtos_mem`,
  `__vfscanf`

## Removed Function

- **`Base64URL_Encode`** (present v149→v149.2, removed in v150): was part of old AES+base64URL upload
  path (see cloud upload protocol above); with switch to cleartext JSON/HTTPS became obsolete
  and completely removed, including contained buffer overflow bug.

## Code Assumptions Contradicted (via instruction and source text comparison)

- `CH395_TCP_Socket_Open_HTTPPort80`: only instruction change `movs r0,#0x24`→`#0xaa`; decompiled
  v149 vs. v150 byte-for-byte identical — value is log_printf line argument, no
  function change.
- `HTTP_PackGetRequest`: only instruction change `movw r0,#0x153`→`mov.w r0,#0x280`; decompiled
  identical, `out_size` remains unchanged function parameter — likewise only log line argument,
  no buffer enlargement.
- `HTTP_ParseUploadResponseCode`: not pure offset shift, but different JSON key
  (`"de":` 5 chars → `"code":` 7 chars, `adds r4,r4,#0x5`→`#0x7`).

## Unchanged (explicitly verified over both hops)

Modbus register contents complete, hard limits of feed-in limits (`Grid_Export_Power_Limiter`,
`Config_ClampDefaults_ParamBlock`), BLE command IDs (extension only, no renumbering), complete
OTA/BLE-OTA/YMODEM path, `hamedata.com` endpoints themselves.

## Open Items

- Exact code cause of v149.2 changelog item "Incorrect data display from missing HTTP uploads" not isolated (candidates: HTTP/cloud telemetry chain, see hop doc v149→v149.2).
- Meaning of unused halves of new BMS pack field pair at offset 0x58/0x5c.
- `Battery_EPS_Reserve_SOC_Hysteresis`: meaning of newly set low byte in CAN-0xCF report
  not determined.
- `__vfscanf` (newly linked in v149.2→v150 step C library function) and `ShellCmd_FreeRtos_mem`: triggering call site or caller not
  located.
- Whether fixed fallthrough bug BLE `Y`→`Z` in v149.2 actually triggered observable misbehavior
  was not verified.
- Fine content of large state machines (`HTTP_Task_StateMachine_Handler`, `MQTT_CtrlCommand_Dispatcher`,
  `BLE_App_Command_Dispatcher`) beyond documented items not fully traced
  instruction-by-instruction.

## Data Basis

- Direct full comparison v149 ↔ v150: `scratchpad/v151/l149_150.json`
- Detailed individual diffs: `scratchpad/v151/l149_1492.json`, `scratchpad/v151/l1492_150.json`
- Decompilation + listing: `Dekompilate/VNSD_Control_149/`, `Dekompilate/VNSD_Control_1492/`,
  `Dekompilate/VNSD_Control_v150_base4800/` (local only, not in the repo)
- Firmware changelog: `Marstek FW Archiv/firmwares/VNSD-0/Control/CHANGELOG.de.md` (local only, not in the repo)
