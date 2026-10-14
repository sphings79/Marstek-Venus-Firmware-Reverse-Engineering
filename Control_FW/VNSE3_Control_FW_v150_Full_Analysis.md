# Control FW v150 (Venus E3) — Full Analysis and Differences to Venus D v150 and E3 v151

> 🇩🇪 Deutsche Version: [VNSE3_Control_FW_v150_Vollanalyse_de.md](VNSE3_Control_FW_v150_Vollanalyse_de.md)

**Status:** completed 2026-10-14. All statements were checked against the code of this image; open points are listed in §12.
**Binary:** `150_control_VNSE3-0_app_0150_0804_151249.bin` (archive `Marstek FW Archiv/firmwares/VNSE3-0/Control/150/`, release v150)
**References:** Venus D Control v150 ([VNSD_Control_FW_v150_Full_Analysis.md](VNSD_Control_FW_v150_Full_Analysis.md)) — same code base; E3 Control v151
([VNSE3_Control_FW_v151_Full_Analysis.md](VNSE3_Control_FW_v151_Full_Analysis.md)), the finished analysis of the successor image
**Diff:** [VNSE3_Control_Diff_v150_v151.md](VNSE3_Control_Diff_v150_v151.md) (v150 → v151, name-paired and complete)
**Ghidra:** `/VNSE3/Control/Control_150_VNSE3-0_app_0150_0804_151249.bin` (project `Marstek_Venus`)
**Decompilation:** `Dekompilate/VNSE3_Control_v150/` (local only, not in the repo): 1920 functions, 0 decompilation failures, + `listing/` (flash listing, `globals_and_registers.txt`)
**Modbus descriptor table:** [../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v150.csv](../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v150.csv)

---

## 0. Summary

- **E3 v150 (built 4 Aug 2026, released 12 Aug 2026) is the predecessor of E3 v151 (built 28 Sep 2026) and the same code base as the Venus D.** 91.4 % of the image is
  identical to E3 v151 once pointers and BL offsets are masked (976 segments), 90.8 % to D v150 (914 segments). Name-paired against E3 v151: 1871 of 1920 functions
  are byte-identical (masked), 49 differ (19 cosmetic, 30 real changes, see the diff document), none exists only in v150, 7 exist only in v151.
  Name-paired against D v150: 1915 pairs, 1845 identical, 70 changed, 5 only in the E3 (E3 dirty-bit wrappers, `OTA_Is_VNSE3_Model`, `Util_NoOp_StartupHook`),
  39 only in the D (the compiled-out functions listed in the E3 v151 analysis §4.6: key/protection chain, grid export limiter, PV/MPPT and multi-pack reporting, CAN date/time broadcast, 0xC4 writes; the D's OTA model check, which the E3 has under another name; and 6 linker stubs).
- **RAM layout differs from D v150 and from E3 v151:** `.data` 12 024 B (0x2EF8), `.bss` ends 0x2001F5DF (initial SP 0x2001F5E0), external SRAM `.xmc_bss` 113 732 B.
  The FreeRTOS heap is **70 KB** (v151: 80 KB), which accounts for 10 240 of the 10 400 bytes by which `.bss` is smaller than in v151. Names were carried over from E3 v151 by
  code alignment, descriptor-table pairing and anchor interpolation (§11).
- **Modbus register map identical to E3 v151 except 32102.** Same 246 descriptor entries (30000–38014); register **32102 is a `float32`** that reads 4 bytes of the
  i16 pair `g_batSamplePowerW` + `g_batSampleVoltage_0p1V` (meaningless; v151: `i16` alias of 30001). `Write_Handler` lacks the write register
  47001 (0xB799) of v151, and its 46000 OTA-target select answers with status 2 instead of sending CAN category 0x18 (§5).
- **Power limits 2500 W**, selector 600 W (mode 2) / 800 W (mode 1) / 2500 W (mode 0) in BLE, MQTT and JSON-RPC — the D's 2200 W (mode 3) does not exist.
- **No PV/MPPT, no battery stack, no parallel inverters** (as in v151): PV fields missing in BLE/MQTT/cloud payloads, no grid export limiter, MQTT cd 0x28–0x2D and BLE
  cmds 0x40–0x43 removed or empty, CAN date/time broadcast (0xCC) and 0xC4 writes not sent.
- **Cloud:** `getDateInfo` on `http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php` (v151: `api-<region>.marstekcloud.com`), server types 0–2 only;
  device source id `VenusE 3.0-<BLE MAC id>` (v151: `VNSE3-0-<id>`).
- **HTTP OTA download (GitHub issue #3, verified):** the body-start selector of v150 is `strstr(response, "\r\n\r\n") + 4`; the old `strstr(response, "Connection:") + 21`
  logic is present in 147/148/149 and gone from 150 on. The first-read handling of the streaming reader is unchanged and has no NULL check and no accumulation (§4.7).
- **Two spellings of the model id** (as in v151): runtime `VNSE3-0` (MQTT topics, cloud, shell user `VNSE3-0 v150`, BLE name `MST_VNSE3_`), OTA/boot descriptor check
  `VNSEE3-0`, image footer `VNSEE3`.

## 0a. Checklist (standing checks, E3 v150)

| # | Question | Answer E3 v150 | Evidence |
|---|---|---|---|
| 1 | Modbus registers different? | **Read table identical to E3 v151 except 32102** (`float32` @ 4 bytes in v150, `i16` in v151) and the RAM source pointers (178 of the 246 are 0x2818 lower than in v151, 33 are 0x27F8 lower, 24 are 8 higher, 3 unchanged). Compared with D v150 the read table is identical in register number, type, scale and count for all 246 entries (212 source pointers are 8 bytes lower than on the D). `Write_Handler` @ 0x08053738: same register set as v151 except 47001 (0xB799) and the reaction of the 46000 OTA-target select (v150: status 2; v151: `CAN_Write_Category0x18(value)` and success); 0xAFDF and 0xAFE6 exist in both. Register 30200 reads the version word @ 0x08010000 = **150** (0x96). | §5, diff §1 |
| 2 | BLE different? | Device name `MST_VNSE3_` + 4 ID bytes. Cmd 0x15 power selector 600/800/2500 W (D: 800/2200/2500 W); cmds 0x40–0x43 do nothing; cmd 0x03 status reply 0x8B bytes (D 0xA7); cmd 0x04 info string without `mppt_v`; cmd 0x13 error history = 20 × 14 bytes (§4.7). v150-specific: cmd 0x0F / 0x54 reset `g_epsReserveState`, LED (0x59) RAM only. | §4.1, §4.3 |
| 3 | AC limits? | Max charge/discharge power **2500 W** (`Config_Write_U16_0x202` clamp 0x9C4, `Config_Write_U16`, defaults 0x9C4); BLE 0x16: 300–2500 W, 0x17: ≤ 2500 W, 800 W schedule clamp in mode 1; no grid export limiter. | §4.1 |
| 4 | Firmware update via BLE possible? | Yes. Same OTA code as v151; model checks compare against **`VNSEE3-0`** (`OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`, boot descriptor in `ProcessFirmwareUpdateCommand`). YMODEM path has **no** file-type check yet (v151 adds `OTA_ClassifyDeviceTypeTag`). | §4.5 |
| 5 | Cloud domains? | `getDateInfo` via `hamedata.com` (plain HTTP, port 80, region prefix `eu`/`www`/`us`); policies, backup-configs, data upload and MQTT hosts as in v151. | §4.4 |
| 6 | Pack capacity? | Control keeps the D structures: `Protocol_AA_CommandDispatch` accepts units `< 9` (one-based), 8 per-pack records of 0x60 B (`g_bmsPack0Voltage` @ 0x20014FAC + k·0x60, last record @ 0x2001524C), 16 cells (MQTT telemetry prints the 16 cell voltages of record 0), Modbus per-pack blocks 34000–34699 unchanged. Multi-pack evaluation removed. The BMS-side bound is decided by the E3 BMS 115 analysis. | §6a |
| 7 | Diff table | [VNSE3_Control_Diff_v150_v151.md](VNSE3_Control_Diff_v150_v151.md) (EN + DE) | – |
| 8 | Re-check of non-functional registers | Control side unchanged compared with v151 apart from 32102; reasons that live in the micro or BMS cannot change with a Control-only image. | diff §8 |

## 1. Binary fingerprint

| | E3 v150 | E3 v151 | D v150 |
|---|---|---|---|
| Size | 374 784 B (0x5B800) | 378 880 B | 389 120 B |
| SHA-256 | `f043becb81ba2c7a5fb5e13edcdc625c6778724bc27073056967c728c989841b` | `a2f566eb…0148` | – |
| Vendor CRC16 (Modbus) | 10332 (0x285C) | – | – |
| Link address | **0x08004800**–0x0805FFFF (18 KB bootloader below) | 0x08004800–0x08060FFF | 0x08004800– |
| MCU | Nations **N32G45x** with XFMC (Cortex-M4F), 144 KB SRAM | same | same |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW) | same | same |
| Build string | `Aug  4 2026 15:11:03` | `Sep 28 2026 09:39:27` | `Aug  5 2026` |
| Version | `VNSE3-0 v150` (shell root); version word 0x96 (150) @ 0x08010000 followed by `VNSEE3-0` | 0x97 | 0x96 |
| Image footer | `VNSEE3` + zero bytes @ 0x0805FFE0, 0xFF fill from 0x0805FEF0 | @ 0x08060FE0 | `VNSD` |
| Device type code | `g_deviceTypeCode` = 0x96 | 0x97 | 0x96 |

Base proof: the vector table at 0x08004800 gives SP 0x2001F5E0 and Reset 0x08004A71 (Keil `06 48 80 47 06 48 00 47`); the scatter table at 0x0805ECA0 lies inside the image and
all three of its regions (LZ77 sources inside the image, `__decompress` @ 0x08005FBE) resolve at this base; `SystemInit` @ 0x08030260 sets `SCB->VTOR = 0x08004800`.

## 2. Memory map (Ghidra)

| Block | Range | Contents |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | not in the image |
| flash | 0x08004800–0x0805FFFF | application |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-bit UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EF7 | RW initial values (LZ77 @ 0x0805F1C0, 2480 B) |
| .bss | 0x20002EF8–0x2001F5DF | zero-initialised; `ucHeap` @ 0x2000313C (70 KB); initial SP 0x2001F5E0 |
| sram_unused | 0x2001F5E0–0x20023FFF | up to the N32G45x 144 KB limit |
| .xmc_bss | 0x60000000–0x6001BC43 | external SRAM via XFMC (LZ77 image @ 0x0805FB70, 896 B) |
| peripherals | apb1/apb2/ahb/bit-band/ppb | register names from the Nations N32G457 SVD (copied 1:1 from E3 v151, same hardware map) |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC and QSPI controller |

Scatter table `Region$$Table$$Base` @ 0x0805ECA0 (limit 0x0805ECD0): .data 0x0805F1C0 → 0x20000000 (0x2EF8), XMC 0x0805FB70 → 0x60000000 (0x1BC44), .bss zero-init 0x1C6E8 @ 0x20002EF8.
The shell command table follows directly (0x0805ECD0, 79 entries).

## 3. Relationship to other versions (method and numbers)

| Comparison | Segments / coverage | Functions |
|---|---|---|
| E3 v150 vs E3 v151 | 976 segments, 91.4 % of the image (96.3 % byte-identical inside) | 1871 identical, 49 changed, 7 only in v151 |
| E3 v150 vs D v150 | 914 segments, 90.8 % | 1845 identical, 70 changed, 5 only E3, 39 only D |
| E3 v150 vs A v150 | 930 segments, 90.9 % | – |
| E3 v150 vs E3 v149 | 902 segments, 87.9 % | – |

Method: offline byte segments with masked pointers and BL offsets (anchor chain), then in Ghidra layout synchronisation from E3 v151 (function bodies, names, thunks),
data and label transfer for matched segments, prototype / plate / local-variable transfer for the 1846 identical and 40 changed pairs, RAM names by aligned-instruction
voting, Modbus descriptor pairing, interpolation between anchors, then manual work on the 34 functions that the segment map could not pair (31 were paired to a v151 function by masked-byte similarity in address order: 25 exact, 6 changed with similarity 0.69–0.96;
3 — `AI_ScheduleQuery_Response_Send`, `Modem_SendStatusTelemetry`, `Protocol_AA_SetSystemParams` — by position, callers and behaviour) and
review of all changed functions by decompilation diff.

## 4. Functional differences to Venus D v150 (E3 reductions) and to E3 v151

### 4.1 Power limits and power selector

| Place | E3 v150 | D v150 |
|---|---|---|
| `Config_Write_U16_0x202` (max charge power, EEPROM 0x202) | clamp 2500 W | 2500 W |
| `Config_Write_U16` (max discharge power, EEPROM 0x204) | clamp 2500 W | 2500 W |
| BLE cmd 0x15 / MQTT cd 0x0F / JSON-RPC "set version" | **600 W → mode 2**, 800 W → mode 1 (+ schedule clamp to 800 W), 2500 W → mode 0 | 800 → 1, 2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–2500 W / ≤ 2500 W | same |
| Grid export limiter | **not present** | present |

### 4.2 Battery / BMS

Single-pack view as in v151: `BMS_CellVoltage_MinMax_PerPack`, `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version`, `BLE_Build_Cmd0x42_Response` and the MQTT stack commands are
not part of the image (§4.6); `BMS_Version_Consistency_Check` only copies the pack-0 version; `CAN_CommLink_Timeout_DataClear` monitors BMS (1) and inverter (2) only.
EPS reserve: `Battery_EPS_Reserve_SOC_Hysteresis` @ 0x0800A714 is a three-state machine in v150 (`g_epsReserveState`: 0 idle, 1 latched, 2 disabled by low SOC): latch when
SOC ≤ discharge cut-off, disable the reserve and write EEPROM 0x300 = 0 below 9 %, unlatch at cut-off + 5 %, automatic re-enable above 8 %; it also runs at SOC 0 %.
v151 replaces it by a stateless variant (diff table).

### 4.3 BLE and MQTT command differences

- **BLE** (`BLE_App_Command_Dispatcher` @ 0x0800C0D4): direct switch on the command byte; cmds 0x40–0x43 empty; cmd 0x03 reply 0x8B bytes; cmd 0x04 info string
  `type=%s,id=%s,mac=%s,dev_ver=%d,bms_ver=%d,fc_ver=%s,inv_ver=%d` (no `mppt_v`); cmd 0x02 (server type) accepts 0–4; LED (0x59) is RAM only; cmd 0x0F and 0x54 reset the EPS state.
- **MQTT ctrl** (`MQTT_CtrlCommand_Dispatcher` @ 0x08022830): cd 0x28–0x2D removed; no cd 0x42; no `debug` sub-command for cd 0x13 (v151 adds both).
- **JSON-RPC** (`MQTT_JSON_RPC_Dispatcher` @ 0x0801E58C): method 5 (PV data) answers "Method not found" (error −0x7F59).
- **Status/telemetry:** `MQTT_StatusReport_BuildAndPublish` @ 0x08027D68 without the D's PV block and without v151's `,df=%d`; `MQTT_Publish_PowerSchedule` publishes p1/p2 with up to 30 entries
  (D: p1–p7, 15 entries); the 1 s power histories keep grid and off-grid power only (29 samples each); peak-shaving fields `peak_status` / `peak_power` are part of the status string.

### 4.4 Cloud / HTTP

| Function | E3 v150 | D v150 |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://%s.hamedata.com/app/neng/getDateInfo%s.php?uid=…&fcv=…&aid=…&sv=0x96&sbv=…&mv=101&cert=…&boot=…&inv=…` (+ OTA status block `&rst=…&mod0=vns&…&mod3=bms…` when `g_otaFlag` is set); region prefix from `g_cloudDomainUrlTable[type]`, file suffix from `g_cloudGetDateInfoSuffixTable[type]`; valid types 0–2 (`eu`, `www`, `us`) | same plus `&mpptv=%d` |
| `Cloud_UploadTelemetryData` (`{"d":"di=…"}`) | keys `pv, sk, mv, me, ma, fu` removed; record stride 0x227; version constant 0x96 | stride 0x35D |
| `Cloud_BuildDeviceSourceID` | `VenusE 3.0-<BLE MAC id>` | `VenusD-<SN>` |
| `Cloud_Report_FillDeviceInfo` | device name `VenusE 3.0` | `Venus D` |
| `Energy_TOU_Counters_Update` | 2 counters (`"%d,%d"`) | 3 counters |
| Upload via FC41D | `AT+QHTTPPOST=<len>,60,60`, outer timeout 600; `Quectel_HTTP_POST_SendAndVerify` with two blind delays | – |

Domains, paths and the TLS code otherwise as in v151 (policies and backup-configs on `hamedata.com`, data upload on `marstekcloud.com`).

### 4.5 Identity and OTA

- Runtime strings: `VNSE3-0` (`g_deviceModelTag`/`g_deviceTypeStr` = `VNSE` + `3-0`), `VNSE3-0 v150`, `MST_VNSE3_`, `VenusE 3.0`, device source id `VenusE 3.0-<BLE MAC id>`.
- OTA / boot: `OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate` and `OTA_FW_Verify_SetStatus` accept an image whose type string is a **substring of `VNSEE3-0`**
  (`strstr("VNSEE3-0", received)`; slot status 0x400 comm timeout, 0x401 CRC, 0x402 device mask, 0x403 type, 0x404 CAN update failure, see §4.7).
  `ProcessFirmwareUpdateCommand` writes the boot descriptor with tag `VNSEE3-0`. The YMODEM path (`YModem_ReceiveFile`) does not classify the file name in v150.
- "AI mode": `AI_ScheduleQuery_Response_Send` (cd 58, only for `g_userWorkMode == 5`) answers with all entries in one message (v151 pages them).
- "CT frequency": `Meter_ParseUDPMeterInfo` blocks in `vTaskDelay(g_meterActiveCtType * 1200)` after every valid meter sample and `Modem_/Ethernet_SendStatusTelemetry` run on every call;
  v151 changes both to a 1.1 s rate limit.
- Connection watchdog: `HTTP_ClientTask_Dispatcher` @ 0x080196CC resets the CH395 after 1800 ticks without progress when more than one telemetry record is queued; it never reboots the device (v151 does after 14 400 ticks).

### 4.6 Functions missing in the E3 (compiled out) and E3 wrappers

Same set as in E3 v151 §4.6 (key/protection chain, export limiter and surplus feed-in, PV/MPPT, multi-pack reporting, CAN date/time and category-0xC4 commands) plus the D-only
linker stubs (6 in D v150). The inverter dirty-bit bitmap has the E3 layout (0x1C bits packed 7 per byte; wrappers `Inverter_RegDirty_Mark_BleState`, `_GridPermit`, `_MeterInvalid`, `_NetworkLink`, `_ChgVolt`);
`Grid_BuildStatusFlagsReg` sends an all-zero 8-byte frame to CAN id 0x1801FFAA.

### 4.7 HTTP OTA download path and diagnostics (GitHub issue #3, verified on this image)

| Item | E3 v150 | Notes |
|---|---|---|
| Body-start selector `HTTP_FindHeaderBodySeparator` @ 0x08018238 | `strstr(resp, "\r\n\r\n") + 4`; NULL → log "Not found split mark \r\n\r\n, header incomplete" and return NULL | v147/148/149: `strstr(resp, "Connection:") + 21` (verified on the raw archive files, selectors 0x08016EF0 / 0x08017570 / 0x08017A58); first appears fixed in 150 |
| Socket response reader `HTTPS_POST_ReceiveResponseData` @ 0x08019240 | in OTA mode every `CH395GetRecvData` overwrites the buffer start; the first read is parsed on its own (`HTTP_ParseContentLengthHeader`, selector, `body_len = read_len − (sep − buf)`, `OTA_WriteDataToRingBuffer(sep, body_len, 0)`) **without NULL check and without accumulating** an incomplete header; later reads are body data. Non-OTA mode (`HTTP_GetUrlData` calls) accumulates up to the buffer size and does not use the selector | identical in 151 (0x08019310) and (by raw bytes) in 148 (0x08018560) |
| OTA cache writer `OTA_WriteDataToRingBuffer` @ 0x08029038 | 5 slots × 0x800 B ring buffer; returns −1 for NULL / length 0, −2 if the OTA stream is not active | identical in 151 (0x08029448) |
| OTA URL state machine | `HTTP_ClientTask_Dispatcher` @ 0x080196CC, called from `Task_ch_http` @ 0x08030362; OTA branch: `HTTPS_POST_Request(6, 0x14, g_otaUrlSlot0_url + idx·0xFB, len, ota_mode = 1)` | 151: 0x0801979C |
| Generic HTTP request `HTTPS_POST_Request` @ 0x08018BA0 | builds a GET (`HTTP_PackGetRequest`), resolves DNS, opens the CH395 socket, calls the reader | 6 call sites in `HTTP_GetUrlData` use `ota_mode = 0` |
| BLE 0x13 error history | `BLE_Send_Response(0x13, g_displayErrorLog, 0x118)` = 20 × 14 bytes (year u16, month/day/hour/minute, two u32 codes); EEPROM 0x1100; dispatcher 0x0800C0D4, loader `EventLog_DisplayError_LoadAndValidate` 0x080172A4, writer `EventLog_Record_DisplayError` 0x0802B8CC | – |
| Component-OTA diagnostics | `OTA_Set_SlotStatus(slot, code, detail)` @ 0x0802BBD4: 4 slots × 8 bytes (u16 code, u32 detail) @ `g_otaErrCode0` 0x20017E28; writers: 0x400 comm timeout (`Comm_Watchdog_CheckTimeout`), 0x401 / 0x402 / 0x403 (`OTA_FW_Verify_SetStatus` @ 0x080093B8), 0x404 (`CAN_UpdateResultHandler` @ 0x0802F898); console `ShellCmd_get_ota_log` @ 0x08038554 | the same functions exist at other addresses in 151 |

The vendor changelog of v150 lists "Fixed failing HTTP updates in Ethernet mode" and "Fixed data loss caused by overlong HTTP payloads"; the selector change is consistent with these items but the code does not state its intent.

## 5. Modbus

- **Read range:** `g_modbusRegDescTable` @ 0x2000034C, 246 × 12 B (30000–38014). Types, scales and counts are identical to E3 v151 and to D v150 except register 32102 (type 0x24 `float32`, element size 4; v151 `i16`).
  Source pointers (v150 minus v151): 178 entries −0x2818, 33 entries −0x27F8 (RAM shift), 24 entries +8 (`.data`), 3 unchanged (external SRAM), 6 other. `FC03_Read_Handler` @ 0x080220E0 and `Read_Serializer` @ 0x08052638 are instruction-identical to v151, so the known D bugs (32101 sign, MBAP length of exception frames) apply.
- **Write range:** `Write_Handler` @ 0x08053738 (45000 command block, 46000 = 0x5100 debug mode, 46500–46544 schedule, high block 0xAFxx); jump tables TBB 8 @ 0x080538F4, TBB 6 @ 0x0805392E, TBH 15 @ 0x080539E6 (manual overrides, all targets inside the function). Differences to v151: no 47001 (0xB799); the 46000 block (values 1234 / 2345 / 3456 / 4567 select `g_otaTargetDevice` 1 / 4 / 3 / 2) returns status 2 after the selection, v151 calls `CAN_Write_Category0x18(value)` and returns 0. 0xAFDF (`Inverter_Set_Reg_0xFE`) and 0xAFE6 (`Config_Apply_SingleReg` 11 / 12) are handled in both versions, only through differently compiled dispatch paths.
- **E3-specific registers:** none; as in v151, registers fed by PV/MPPT or by packs 2–8 have no data source. Descriptor decode: [VNSE3_Descriptor_Table_Decoded_v150.csv](../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v150.csv).

## 6. Internal CAN bus (Control side)

Arbitration id: `CAN_Build_Arbitration_ID` @ 0x08034940. Receive side: `CAN_FrameDispatcher` @ 0x08030EE0 → class 4 `Telemetry_Register_Dispatcher`, class 2 `Modbus_ResponseDispatch`, else
`Protocol_AA_CommandDispatch` @ 0x080317C4. Function codes sent as in v151 §6 except the category-0x18 writes from the Modbus 46000 / 47001 path and the MQTT `cd=0x42` path, which only v151 has.
Not sent: 0xCC date/time, 0xC4 category writes, 0xCB DLC2, 0x04 DLC2, 0x03 DLC2 clear.

### 6a. Pack capacity (standing check 6)

| Item | E3 v150 | D v150 |
|---|---|---|
| `Protocol_AA_CommandDispatch` unit bound | `(id & 0xFF) < 9` (identical to v151) | `< 9` |
| Per-pack records | 8 × 0x60 B (`g_bmsPack0Voltage` @ 0x20014FAC + k·0x60, `g_bmsPack7Voltage` @ 0x2001524C) | 8 × 0x60 B |
| Modbus per-pack blocks | 34000–34699, identical descriptors | same |
| Multi-pack evaluation | removed | present |
| Cells per pack | 16 | 16 |

The Control has room for 8 units but evaluates one pack; the number of packs the E3 BMS really reports (`CAN_TX_PerPack_10Msgs`, `pack_id < N`, count from 1) is decided by the E3 BMS 115 analysis and is not part of this image.

## 7. Tasks, shell, protocols

Task table `g_taskDefTable` @ 0x08057988 (14 tasks, same names/stacks/priorities as v151), `shellCommandTable` @ 0x0805ECD0 (79 entries: 66 commands, user `VNSE3-0 v150` with empty password,
10 key bindings, `help`, `clear`; v151 adds `read_inv1`, `read_inv2`), `g_mqttRpcMethodTable` @ 0x0805AB6C (16 methods), `mbedtls_cipher_definitions` @ 0x0805ACC4 (7), CH395 sockets (502 Modbus TCP, 8091 shell, 8090 UDP).

## 8. EEPROM map

Same addresses as v151 (`Config_LoadOrInit_DeviceSettings` @ 0x0802B28C: identity, 0x200, 0x300, 0x900, 0x36BD, 0x500, 0x400 Wi-Fi, 0x484/0x488/0x492 TOU counters). v150 does not touch
0x395 (LED flag) and reads the server type 0x441 on demand instead of at boot.

## 9. Strings of interest

`VNSE3-0 v150`, `VNSEE3-0`, `MST_VNSE3_`, `VenusE 3.0`, `VenusE 3.0-%s`, `Build:\tAug  4 2026 15:11:03`, `[HTTP] Not found split mark \r\n\r\n, header incomplete`, `[HTTP]ch395 reset!!!!`,
the `getDateInfo` URL on `hamedata.com`, `AT+QHTTPPOST=%d,60,60`, the shortened `{"d":"di=…"}` upload format. Vendor default Wi-Fi credentials and `quit_debug_5100_` are present as on the D.

## 10. Security notes

All Control findings of D v150 / E3 v151 apply (identical code): Modbus 46000 = 0x5100 → unauthenticated shell on RS485 and TCP 8091, shell user with empty password, vendor default Wi-Fi credentials,
plain-HTTP `getDateInfo`. The OTA type check uses `strstr("VNSEE3-0", received_type)`, i.e. any substring of the constant passes. The HTTP OTA reader trusts the first TCP read to contain the complete header and does not
check the selector result (§4.7); the downloaded image is CRC16-, device-mask- and type-checked afterwards (`OTA_FW_Verify_SetStatus`); a failed check sets slot status 0x401 / 0x402 / 0x403.

## 11. Processing status in Ghidra

- Coverage: 0 undefined bytes, 0 orphan instructions in the flash block; 1920 functions, all named, all with a prototype (task entries `void f(void *pvParameters)`); 3 manual jump-table overrides in `Write_Handler`, no unrecovered jump table in the export.
- `marstek_gates.py`: **all gates pass** (no code-as-data except the two known padding cases, no literal-only functions, no FUN_/tmp_/duplicates, 0 unnamed referenced RAM globals, task names match the task table,
  no-return flags sane); decompilation export 0 failures; short-string check reviewed (`,` `.` `|` `]` `x` `eu` `ip` … are genuine format fragments).
- Tags: `NAME_VERIFIED` on all 1920 functions, of which 93 additionally `NAME_LOW_CONFIDENCE` (same marking as v151).
- RAM globals: 1869 labels (v151: 1900; the difference consists of variables that only v151 has and a few unreferenced slots): about 1100 from aligned-instruction voting, 124 from the Modbus descriptor pairing, about 570 from anchor interpolation, 50 telemetry-record fields,
  a handful by hand where the vote was wrong (grid-power baseline block, BLE MAC id characters, EPS state byte, cloud URL tables, meter IP string). 15 interior addresses in 7 functions are still shown as `uRam…`/`bRam…` in the export — exactly the same functions as in the v151 export.
- Plates of the 27 functions whose behaviour differs from v151 were rewritten for v150.

## 12. Open questions

1. Exact meaning of the 32102 `float32` descriptor: it reads `g_batSamplePowerW` and the following 16-bit word as one float; whether any client ever relied on it is unknown (v151 removes it).
2. Purpose of `g_deviceInfoReserved8aa` (set to 0xA882 in v151, absent in v150).
3. Category 0x18 (sent by v151 only, ID 0x00014118, Modbus 46000 / 47001 and MQTT `cd=0x42`; its purpose is not stated in the code): checked against E3 Micro 119 and E3 BMS 115, both ignore it (Micro 119: no case in `EMS_Inverter_CAN_Dispatcher`, default exit; BMS 115: no match in `CAN_RX_Handler` / `CAN_RX_AuxCommand_Handler`). Control has no handler for an incoming 0x18 either. Open: other nodes (FC41D etc.) and whether BMS and Control share one physical bus.
4. `Write_Handler`: constants were compared exhaustively against v151, but a branch-by-branch equivalence proof was not done.
5. The behaviour of the E3 micro and BMS images that shipped together with v150 is outside this Control-only analysis.
6. HTTP OTA body-start selector, OTA reader first-read behaviour, BLE 0x13 vs OTA diagnostic slots, the 0x401/0x402/0x403 chain and category 0x18 across versions and families: see [OTA_HTTP_Body_Boundary_and_Diagnostics.md](OTA_HTTP_Body_Boundary_and_Diagnostics.md).

## Appendix: method

Offline segment map of the images (masked pointers and BL offsets), `ve3_150_layout_sync.py` / `ve3_150_port_segments.py` / `ve3_150_port_signatures.py` / `ve3_150_copy_by_address.py` (peripherals only) / `ve3_150_port_refs.py` (flash targets only) /
`ve3_150_port_globals_aligned.py` / `ve3_150_ram_interp_port.py` to carry layout, names, prototypes, references and RAM globals over from E3 v151; byte-match pairing of the 34 unpaired functions;
manual typing of the image tail and literal pools; review of all changed functions by name-paired decompilation diff; verification gates. The scripts are local only (not in the repo).
