# Control FW v150.8 (1508, Venus A) — Full Analysis (port from 1509)

> 🇩🇪 Deutsche Version: [VNSA_Control_FW_v1508_Vollanalyse_de.md](VNSA_Control_FW_v1508_Vollanalyse_de.md)

**Status:** completed 2026-11-05. The image was analysed by porting the finished 1509 analysis and verifying the port against the 1508 code; limits and unchecked points are listed in §9.
**Binary:** `1508_control_VNSA-0_app_1508_0819_182932.bin` (archive `Marstek FW Archiv/firmwares/VNSA-0/Control/1508/`, beta "v150.8", "A Control 1508")
**Port source:** Venus A Control v150.9, see [VNSA_Control_FW_v1509_Full_Analysis.md](VNSA_Control_FW_v1509_Full_Analysis.md). Everything that 1508 shares with 1509 (D/A comparison, Modbus, CAN, EEPROM, tasks, security notes) is documented there and is **not repeated** here.
**Differences 1508 → 1509 (function table):** [VNSA_Control_Diff_v1508_v1509.md](VNSA_Control_Diff_v1508_v1509.md)
**Ghidra:** `/VNSA/Control/Control_1508_VNSA-0_app_1508_0819_182932.bin` (project `Marstek_Venus`)
**Decompilation:** private working export (not part of this repository): 1935 functions, 0 decompilation failures, plus a flash listing and `globals_and_registers.txt`, same layout as for 1509.

---

## 0. Summary

- **Same code base as 1509.** After masking pointers and call operands, 95.19 % of the 1508 bytes are identical to 1509. 1508 has 1935 functions, 1509 has 1941; 1508 has **no function of its own** (only in 1508: none), 6 functions exist only in 1509.
- **1508 is the older build** (build string `Aug 19 2026 18:29:09`, 1509: `Sep  8 2026 09:34:19`). It lacks the changes that the [v150 → v150.9 table](VNSA_Control_Diff_v150_v1509.md) attributes to 1509: the OTA device-type classification (`OTA_ClassifyDeviceTypeTag`, YMODEM device-type check), the MQTT command block with `CAN_Write_Category0x18`, `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `ShellCmd_read_inv1/2` and Modbus write register 47001.
- **1508 identifies itself differently:** `VEPRO-0` / `VenusE Pro` / BLE prefix `MARS_VEPRO_` / version constant 0x5E4, where 1509 has `VNSA-0` / `Venus A` / `MST_VNSA_` / 0x5E5 (§4). This is a fact from the code; its meaning is not established.
- **The RAM layout is not identical to 1509** (`.bss` is smaller, many globals sit at other addresses). This was the main source of work in the port (§3.3).
- **Verification:** all gates pass (`ALL GATES PASS`); tags: 1825 functions `NAME_VERIFIED`, 110 `NAME_LOW_CONFIDENCE` (§7).

## 1. Binary fingerprint

| | A 1508 | A 1509 (reference) |
|---|---|---|
| Size | 387 072 B (0x5E800) | 391 168 B (0x5F800) |
| SHA-256 | `664623dd130c9ded1225ed7e9223db65b73a4c51add6e53fb66c1014ab3b1e4f` | `a8c58127189e19937cf179195e714492f890a6a829b03b8e218380042f8d9f23` |
| Link address | **0x08004800**–0x08062FFF | 0x08004800–0x08063FFF |
| MCU | Nations **N32G45x** (Cortex-M4F) | same |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW) | same |
| Build string (`ShellCmd_get_ver`) | `Aug 19 2026 18:29:09` @ 0x0803B147 (string start) | `Sep  8 2026 09:34:19` |
| Version | version word 0x5E4 (1508) @ 0x08010000, followed by the bytes `VNSA-0` | version word 0x5E5, followed by `VNSA-0` |
| Image footer | `VNSA` + 12 zero bytes @ 0x08062FE0, before it 0xFF fill (912 B) from 0x08062C50 | `VNSA` @ 0x08063FE0 |
| Functions in Ghidra | 1935 | 1941 |

Base proof: the initial stack pointer (0x2001F7E8) and the reset vector (0x08004A71) at the start of the image resolve at base 0x08004800, as in 1509; the scatter table and the string and function pointers resolve at this base.

## 2. Memory map (Ghidra)

Same block structure as 1509 (see [§2 of the 1509 analysis](VNSA_Control_FW_v1509_Full_Analysis.md)): bootloader below 0x08004800 (not in the image), application flash, `sysinfo` (UID @ 0x1FFFF7F0), `.data`, `.bss`, external SRAM `.xmc_bss` @ 0x60000000, peripheral blocks (names from the Nations N32G457 SVD, copied from 1509: 977 labels).

Scatter table (`Region$$Table$$Base` @ 0x0806189C, three entries, decompressor `__decompress` @ 0x08005FBE):

| Region | Load address (flash) | Execution address | Size | Compressed image |
|---|---|---|---|---|
| `.data` | 0x08061DBC | 0x20000000 | 0x2F00 | 2488 B |
| external SRAM (`.xmc_bss`) | 0x08062774 | 0x60000000 | 0x26AE0 (1509: 0x26AF8) | 1244 B |
| `.bss` (zero-init) | — | 0x20002F00 | 0x1C8E8 (1509: 0x1F110) | — |

Behind the table: `ShellCommand[79]` @ 0x080618CC (1509: 81 entries). The initialised `.data` has the same size as in 1509 (0x2F00); `.bss` is 0x2828 bytes and the external SRAM 24 bytes smaller.

## 3. Method

### 3.1 Port from 1509

1. **Anchors:** byte blocks of the 1509 image that reappear in 1508 with pointer and BL operands masked: 452 blocks, 368 444 of 387 072 bytes = **95.19 %** identical.
2. **Function pairs** via these anchors: of the 1941 source functions, **1902 identical, 20 partially identical, 19 without direct counterpart**. Layout (functions, data items, labels, comments), signatures (1922 pairs, including local variables), references and 25 flow overrides were carried over from 1509.
3. **Corrected assignments in the reordered area:** in the area around 0x0800FF08 and in a few places up to 0x080176F4–0x08017700 the functions are in a different order than in 1509, so the anchor assignment was wrong or missing for 16 pairs. They were checked by normalised instruction similarity and corrected by hand (`Board_GetUniqueID_HexString`, `Quectel_Modem_HardwareReset_Handler`, `Quectel_AT_QVERSION_Query`, `BLE_GATT_HexBuffer_Send`, `Stub_Return_One`, `Util_NoOp_Stub`, `GPIO_SetBits`, `GPIO_ResetBits`, `QSPI_Flash_QuadRead_ByteLoop`, `Util_NoOp_StartupHook`, `CH395_Init_Modbus_TCP_Socket`, `MQTT_Transport_NullCallback` (stub, low confidence) and four older-form variants named by their role: `Cloud_UploadTelemetryData`, `Modem_SendStatusTelemetry`, `Ethernet_SendStatusTelemetry`, `Protocol_AA_SetSystemParams`). Afterwards no `FUN_*`, no duplicate and no temporary names remained, and all pairs have an instruction similarity ≥ 0.9 except the genuinely changed functions (§5).
4. **Switch tables:** 11 jump tables in the changed code were typed as arrays and given decompiler overrides; the 63 table destinations were disassembled (computed-jump references), 13 in-function code gaps were disassembled. In `HTTP_ClientTask_Dispatcher` the table (`tbh` @ 0x0801A38A, 7 entries) was labelled `case_0`…`case_6`; its targets are those of 1509 shifted by 0x14. The labels follow the table entries, they were not verified beyond these 7 entries.
5. **Call-target check:** the ordered call targets (`bl`, `blx`, `b.w` to function entries) of **all 1935 pairs** were compared. 21 findings, all inside functions that are changed anyway (13 call-count differences, 6 reordered callees in `HTTP_ClientTask_Dispatcher`, 2 `sprintf`/`snprintf`/`memcpy` substitutions); none among the 1902 identical pairs, so no false pairing and no name correction was necessary.

### 3.2 Changed functions

36 pairs with a similarity < 0.98 plus `Cloud_PassiveMode_Connection_StateMachine`, `MQTT_JSON_RPC_Dispatcher` and `Board_GetUniqueID_HexString` were compared on normalised decompiler output (type, cast and `&` noise removed). The result is the table in [VNSA_Control_Diff_v1508_v1509.md](VNSA_Control_Diff_v1508_v1509.md); 29 plate comments "Differs from 1509: …" mark the functions in the program. Where a difference was seen but its meaning not established, the diff table says "unclear".

### 3.3 RAM layout shift against 1509

The RAM layout of 1508 differs from 1509 (example: `g_pendingBootDescriptor` @ 0x200152D4 in 1508 vs 0x20017AF4 in 1509; the 1508 `.bss` is 0x2828 bytes smaller, the heap `ucHeap` is shorter, `g_protocolTxBuf` @ 0x20014944 lies inside the 81 920 B that the 1509 heap occupies). Consequences and handling:

- Copying references by address from 1509 produced **bogus references** to 1509 RAM addresses. They were cleaned by comparing every reference with the pc-relative literal of the 1508 bytes: 627 instruction references corrected (352 targets), 29 parameter references corrected, 294 references without support in a RAM literal deleted, 3 data-word references deleted, 11 derived references deleted.
- 133 RAM labels that the first name sync had put on 1509 addresses were deleted; 3 were moved. Names were then assigned only from clean votes (reference from byte-identical code): 1287 labels from the first sync (wrong ones removed as described above), 450 further "exact" names, 22 by majority vote ≥ 50 % (medium confidence), 2 by code context.
- **Never copy by offset, only by name:** 1610 1508 RAM/XMC labels have a same-named 1509 label; of these 1541 data items were typed like in 1509 (type, length checked against 1509), the rest were not typed (§9).
- 111 RAM addresses have no counterpart name; they carry the convention name `<preceding object>_off<hex>` (e.g. `g_ch395RxBuf_off1029`). Such a name states **only the address relation, not a meaning**; the symbols carry the plate comment "NAME_LOW_CONFIDENCE: address relation only".
- Three names were renamed because of identical code roles but different struct offsets: `g_atRxAssemblerState`, `g_atRxSuppressFirstTerm`, `g_atUdpRxDoneFlag` (1508: +2/+3/+4 from `g_atUdpRxLen`; 1509: +4/+5/+6, because 1509 has the extra field `g_atRxByteCounter`).
- Five single-byte variables received names from neighbouring 1509 symbols and from paired functions that reference the same 1509 symbol (medium confidence): `g_commCh0StaleFlag` (0x2000013D), `g_canLinkMissCount` (0x2000013E), `g_canLinkStaleFlag` (0x2000013F), `g_commCh2StaleFlag` (0x20000141), `g_canRegCbDevState` (0x20000162).

## 4. Device identity (facts from the code)

| Item | 1508 | 1509 |
|---|---|---|
| Device-type string (`ShellCmd_get_ver` `ems_type`, MQTT topics, UDP discovery answer) | `VEPRO-0` | `VNSA-0` |
| Shell root entry (`CLI_InitSession`) | `VEPRO-0 v1508` @ 0x08051734 | `VNSA-0 v1509` |
| Cloud device name (`Cloud_Report_FillDeviceInfo`) | `VenusE Pro` @ 0x08020FC4 | `Venus A` |
| Cloud source id (`Cloud_BuildDeviceSourceID`) | `"%s-%s"` with `VenusE Pro` | `"%s-%s"` with `VNSA-0` |
| BLE name prefix (`BLE_Set_Device_Name`, `BLE_Query_Device_Name`) | `MARS_VEPRO_` @ 0x0800F370 | `MST_VNSA_` |
| Version constant (`g_deviceTypeCode`, cloud query) | 0x5E4 | 0x5E5 |
| OTA model check (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`, update magic) | compares against `VNSA-0` | `VNSA-0` |

The archive folder, the file name, the version block at 0x08010000 and the image footer say `VNSA` / `VNSA-0`; the strings the running code uses to identify itself say `VEPRO-0` / `VenusE Pro`. The OTA model check in 1508 still tests for `VNSA-0`.

**Assumption, not a finding:** the name pattern suggests that "Venus E Pro" is meant. There is **no hardware proof** for this (no device, no vendor document); the meaning of the identity change is unclear. The documentation therefore uses the archive name `VNSA` and states the strings explicitly.

## 5. Differences to 1509 (summary)

Detail table with addresses and evidence per function: [VNSA_Control_Diff_v1508_v1509.md](VNSA_Control_Diff_v1508_v1509.md). In short:

- **Identity** (§4) and the version constant in the cloud query.
- **Functions and code only in 1509:** `OTA_ClassifyDeviceTypeTag` (device-type check of the received file in `YModem_ReceiveFile`, "Bin type mismatch" branch in `YModem_FirmwareUpgrade_Handler`), `MQTT_Publish_WorkMode_Config`, `MQTT_ConnectionStatus_Report_Send`, `CAN_Write_Category0x18` (1508: no Modbus write register 47001, no own case for 45030 in `Write_Handler`, no matching command block in `MQTT_CtrlCommand_Dispatcher`), `ShellCmd_read_inv1`, `ShellCmd_read_inv2` (shell table 79 vs 81 entries).
- **Restructured logic (effect unclear):** `Battery_EPS_Reserve_SOC_Hysteresis` (state machine instead of latch), `HTTP_ClientTask_Dispatcher` (136 vs 364 decompiled lines, same callees in another order, no time-sync fast-retry handling in the compared part), `Grid_Export_Power_Limiter` (inline negation and a 2500 W clamp of `power_hi`; the 2400 W upper clamp is in both), `CT_GridPower_Controller` (one `Inverter_Power_Setpoint_Calc(1,0)` call fewer), `Cloud_PassiveMode_Connection_StateMachine`, `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `DeviceInfo_BuildStatusString`.
- **Older forms:** `Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry`, `Protocol_AA_SetSystemParams` (role-named, low confidence); `CH395_Init_Modbus_TCP_Socket` does not call `CH395_ConfigTcpKeepalive_Defaults`; `Config_LoadOrInit_DeviceSettings` has one `EEPROM_Read` fewer (0x395, `g_selLedEnable`); `Meter_ParseUDPMeterInfo` adds a `vTaskDelay`.
- **Only presentation or relocation, no code change established:** `SMR_TIC_SelfUse_PowerController` (log line numbers shifted by 2), `Quectel_SendCmd_WaitResponse`/`CH395_ExtractUdpData`/`Quectel_AT_Recv_ByteAssembler` (`memset` 0x406 vs 0x408), and the functions listed under "pure presentation" in the diff table.

## 6. What is shared with 1509

For the following topics the 1509 analysis applies to 1508 **as far as the pair comparison shows identical code** (1902 identical pairs); it was not re-derived for 1508 function by function:

| Topic | See |
|---|---|
| Relationship to Venus D v151 (instruction-identical share, compile-time device constants) | §0, §3, §4 of the 1509 analysis |
| Power limits, power classes, 13 cells / 6 packs per pack scan, RS485 on USART1, functions compiled out | §4 of the 1509 analysis |
| Cloud / HTTP / network (hamedata `getDateInfo`, 3 server types, older HTTP-POST/TOU variants) | §4.5 of the 1509 analysis |
| Modbus read and write range, 44002/44003 limits | §5 of the 1509 analysis (1508: the 44003 limit text is identical; 47001 and 45030 differ, see §5 above) |
| Internal CAN bus (target classes, function codes) | §6 of the 1509 analysis |
| Tasks, shell, protocols, EEPROM map, strings, security notes | §7–§10 of the 1509 analysis |

## 7. Processing status in Ghidra

- Coverage: 0 undefined bytes, 0 orphan instructions; 1935 functions, all named (0 `FUN_*`), all with a prototype; task names match the task table.
- `marstek_gates.py` (run in parts because of the script timeout of the tooling): **`ALL GATES PASS`**.
- Tags: **1825 `NAME_VERIFIED`, 110 `NAME_LOW_CONFIDENCE`** (1935 functions; exclusive, none with both tags — in 1509, 99 functions carry both tags, so the counts are not directly comparable). Forced to `LOW`: the four role-named functions, `MQTT_Transport_NullCallback` and 8 partial pairs with similarity < 0.9 (`Battery_EPS_Reserve_SOC_Hysteresis`, `Quectel_AT_Recv_ByteAssembler`, `HTTP_ClientTask_Dispatcher`, `Grid_Export_Power_Limiter`, `MQTT_StatusReport_BuildAndPublish`, `DeviceInfo_BuildStatusString`, `Cloud_BuildDeviceSourceID`, `YModem_ReceiveFile`).
- Comments: ported header comments contradicted the code in 16 plates (device-identity text taken over from 1509). These were corrected against the real 1508 strings and now say "In 1509: …": `BLE_Set_Device_Name`, `BLE_Query_Device_Name`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `Cloud_Report_FillDeviceInfo`, `Config_LoadOrInit_DeviceSettings`, `Cloud_BuildDeviceSourceID`, `ShellCmd_get_ver`, `CLI_InitSession` and the RAM plates `g_maxChargePowerW`, `g_deviceModelTag`, `g_deviceTypeCode`, `g_httpCloudServerType`, `g_deviceTypeStr`, `g_cloudDeviceSourceId`. Left unchanged because correct in 1508: the OTA model checks (`VNSA-0`).
- Export: 1935 functions, 0 decompilation failures, no file with "decompilation failed"; 2721 named RAM/peripheral symbols in the listing.

## 8. Standing checks for a new firmware version (1508 against 1509)

| Question | Result |
|---|---|
| Modbus registers | `Write_Handler`: no 47001 (0xB799) and no own case 45030 (0xAFE6) in 1508; limit text of 44003 identical. The descriptor table shows only presentation differences; **not checked:** register-by-register comparison of the table contents. |
| BLE | Name prefix `MARS_VEPRO_` instead of `MST_VNSA_`; one `EEPROM_Write` fewer in `BLE_App_Command_Dispatcher`; the command set is not shown to differ. **Not checked:** power classes and limits. |
| AC feed-in limit | 44003 text unchanged (1500 W); `Grid_Export_Power_Limiter` differs (§5), upper clamp 2400 W in both; effective feed-in limit of the 1508 changes: **unclear**. |
| BLE OTA | The BLE OTA functions are present, no difference found; BLE OTA is therefore possible as far as the code shows (**not tested on a device**). YMODEM: no device-type check in 1508. |
| Cloud domains | Same set as in 1509; changed: device name/source id and version constant in the cloud query. |
| Pack capacity | Control side unchanged: 6 packs (`< 6`), 13 cells (`< 0xd`) in `BMS_CellVoltage_MinMax_PerPack`. The BMS-side limit (`CAN_TX_PerPack_10Msgs`) is not part of the Control image: **not checked**. |
| Non-functional registers (re-check) | **Not checked** at register level. |

## 9. Open points and limits

1. **Meaning of the identity** `VEPRO-0` / `VenusE Pro` / `MARS_VEPRO_`: unclear; "Venus E Pro" is an assumption without hardware proof.
2. **111 RAM names `<object>_off<hex>`** are address relations with low confidence, not semantics (§3.3). No field names were derived for them by hand.
3. **10 type conflicts** (list in the working data, file `p3b_conflicts_100.json`; `p3b_type_conflicts.json` itself is empty): the 1509 object would swallow a label that exists only in 1508, i.e. the layout differs there. These objects were **not** typed: `s_quit_debug_5100`, `g_modbusRegDescTable` (label `_off9` inside), `g_bleOtaDeviceTypeTable`, `g_cloudDomainUrlTable`, `ecp_supported_grp_id`, `ucHeap` (the 1508 heap is shorter), `g_wifiSsid`, `g_wifiPassword`, `g_tlsCertDecodeBuf`, `g_telHistDatePacked`. Further RAM labels without a same-named, typed 1509 counterpart are untyped.
4. **5 single-byte variables** (§3.3) have names of medium confidence (neighbour layout plus paired-function evidence).
5. **Stray references:** references on named objects can still contain a few bogus ones that lie within 0x1000 of a literal; they are not detectable without data flow. The unsupported-reference filter catches only far-off ones.
6. **Not checked:** the descriptor table entry by entry against 1509 and against D; the BMS-side pack limit; the status of non-functional registers; BLE power classes and limits; a comparison 1508 against the release v150 (the v150 → v150.9 table was not re-run for 1508); everything that depends on the micro and BMS firmware of the same release (this analysis covers the Control image only).
7. **Unclear effects:** EPS reserve hysteresis, `Grid_Export_Power_Limiter` clamp, restructuring of `HTTP_ClientTask_Dispatcher` (completeness of its 1508 decompile not fully established, its jump-table labels were reconstructed), `Grid_PowerSync_StateMachine`, `MQTT_StatusReport_BuildAndPublish`, `DeviceInfo_BuildStatusString`, `Cloud_PassiveMode_Connection_StateMachine`.
8. **Role-named pairs** (`Cloud_UploadTelemetryData`, `Modem_/Ethernet_SendStatusTelemetry`, `Protocol_AA_SetSystemParams`) and the stub `MQTT_Transport_NullCallback` have low confidence. One jump-table label in `Write_Handler` (`case_15`) was placed from neighbouring labels, not from the table.
9. **Shared with 1509:** open questions about the A micro (CAN frames, power classes 4/5) from §12 of the 1509 analysis apply to 1508 unchanged.

## Appendix: method

Offline anchor map of the two images (masked pointers and BL offsets) → function pairs → carry-over of layout, signatures, references, flow overrides and comments from 1509 → verification of the pairs by normalised instruction similarity and correction by hand in the reordered area → cleaning of bogus RAM references and name sync from clean votes → data types and peripheral labels by name → switch tables → call-target comparison of all pairs → tags → identity comments → gates → export. Helper scripts and the working export are local only (not in the repository).
