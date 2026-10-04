# Control FW v150 (Venus D) — Full Analysis

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_v150_Vollanalyse_de.md](VNSD_Control_FW_v150_Vollanalyse_de.md)

**As of:** 2026-09-30. Replaces `VNSD_Control_FW_Analysis_app_0150_0805_115146.md` for all address and
name specifications (that version was based on wrong image base 0x08000000; all flash addresses there
are 0x4800 too low). Field measurements from old version remain valid and are taken over here as
field observation with reference.

**German:** `VNSD_Control_Diff_v150_v151.md` (created after this version) ·
**Diff to predecessor:** `VNSD_Control_Diff_v149_v150.md` (major hop, with intermediate steps
`VNSD_Control_Diff_v149_v1492.md` and `VNSD_Control_Diff_v1492_v150.md`) ·
**Diff to successor:** `VNSD_Control_Diff_v150_v151.md` ·
**Decompilation/listing:** `Dekompilate/VNSD_Control_v150_base4800/` (local only, not in the repo) (1954 functions) ·
**Additional v150 docs:** `VNSD_Cloud_Upload_Payload_v150.md` (field capture of cloud upload),
`VNSD_Control_Charge_Power_PV_Curtailment_v150.md` (BMS limit → power, PV throttling in detail)

## 0. Summary

- Image base **0x08004800** (18-KB bootloader below), Nations N32G45x (previously misidentified as Artery
  AT32F403A, see §1a), Keil ARMCC, FreeRTOS, mbedTLS 2.28, cJSON, llhttp, letter-shell, CH395, Quectel FC41D.
  Same codebase as v147–v151.
- v149 → v149.2 (beta) carried major substantive development (BMS pack record 88→96 B, simplified
  French self-use controller SMR/TIC, rebuild internal telemetry structures; 43 changed
  functions, register/behavior interfaces unchanged). Step v149.2 → v150 is
  **no minor step**: of 1934 named-paired functions 56 differ, **26 of them
  substantively**, plus **20 new and 1 removed function** (detail: `VNSD_Control_Diff_v1492_v150.md`).
- Largest changes v149.2 → v150: completely new HTTPS/TLS client (mbedTLS over CH395) for
  cloud telemetry upload, replacing AES-128-ECB-encrypted, base64URL-encoded GET to
  `hamedata.com` with cleartext JSON-POST to `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s`
  and fixing real buffer overflow in v149.2 (§4.4); new peak-shaving/
  power-clamping feature (EEPROM 0x394, new BLE command, two new MQTT commands); two new
  connection watchdogs (CH395 chip reset, modem reset) plus GET→POST rebuild of mobile upload
  (§7).
- Manufacturer changelog names five items (local API error Ethernet, HTTP update error Ethernet,
  peak-shaving, data loss on long HTTP records, CT_TYPE meter connection); all five now map to
  concrete code changes (§7, mapping table in `VNSD_Control_Diff_v1492_v150.md`).
- New shell command `FreeRtos_mem` (heap display, §4.3) is among 20 new functions of
  v149.2→v150 step (`ShellCmd_FreeRtos_mem`) and recorded in diff doc (§7/§9).
- Security posture as in neighbor versions: Modbus register 46000 = 0x5100 enables without
  authentication debug shell on RS485 **and TCP 8091** persistently (§8).

## 0a. Verification Checklist (v150 versus predecessor 149/149.2 and successor 151)

| # | Question | Answer v150 | Evidence |
|---|---|---|---|
| 1 | Modbus registers changed (vs. 149/149.2)? | **Register table itself no.** `g_modbusRegDescTable` (0x20000354, `ModbusRegDesc_t[246]`) byte-identical over both hops (register number, type, scaling, count); only 215 entries shift RAM shadow address because config struct grew 6 B for peak-shaving (§7). Schedule EEPROM (registers ≥46500) moves from 0x36BF to 0x3700. Vs. **151**: read table identical except 32102 (`float32`→`i16` in 151), 151 adds write register 0xB799. | §4.1, diff docs |
| 2 | BLE changed (vs. 149/149.2)? | **Extended.** v149.2 → v150 adds new command `)` (0x29, "Set Peak Shaving"), two response lengths grow (`BLE_Build_DeviceStatus_Response` 0xA4→0xA7 B, `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` 0x50→0x51 B) and fallthrough bug (command `Y`→`Z`, missing `break`) fixed. Vs. **151**: 3 additional detail changes (EPS reset flag removed, LED status EEPROM fix in 151). | §4.5, §7 |
| 3 | AC feed-in 2500 W changed? | **Hard limit unchanged** across all three versions (`Grid_Export_Power_Limiter` at v150 starts with `min(P_bms, 2500 W)`). Additionally, independent of that: v149.2 → v150 introduces configurable peak-shaving clamp value (EEPROM 0x394, acts in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process`, §7) — supplements 2500-W clamp, doesn't replace it. Only **151** changes feed-in start value itself (grid measurement − P_neg, without 2500-W clamp). | §5, §7 |
| 4 | Firmware update via BLE still possible? | **Yes**, OTA/BLE-OTA/YMODEM path byte-identical since v149 through v150 inclusive; **151** adds only type check in YMODEM path (filename→target device), doesn't change BLE-OTA path itself. | §4.5 |
| 5 | Cloud domains changed? | **Telemetry upload switches precisely in this step** from `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s` (AES+base64URL, GET) to `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (JSON, POST, §4.4); all other `hamedata.com` endpoints (`getDateInfo`, `getDevicesApply`, `backup-configs`, `schedulings/policies`) remain unchanged. Only **151** additionally moves `getDateInfo` from hamedata to `marstekcloud.com`. | §4.4, §7 |

## 1. Binary Fingerprint

| Property | Value |
|---|---|
| File | `150_control_VNSD-0_app_0150_0805_115146.bin` |
| Size | 389 120 bytes (0x5F000) |
| SHA-256 | `4140e7a6d38ac90c8ceb6a5682f7704fd3ad4a45c2cb3fc1f05c95bfc1b71447` |
| API-CRC (manufacturer) | 7326 |
| Link address | `0x08004800`–`0x08063FFF` |
| MCU | Nations N32G45x with XFMC (Cortex-M4F, Thumb-2), previously misidentified as Artery AT32F403A/407 (see §1a) |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW) |
| Libraries | FreeRTOS, mbedTLS 2.28, cJSON, llhttp, letter-shell, N32 standard peripheral library (n32_stdperiph, previously listed as "AT32 firmware library"), WCH CH395 library |
| Initial SP | `0x2001F7E8` (vector table @ 0x08004800, verified in raw image) |
| Reset vector | `0x08004A71` (Thumb) → `SystemInit` → `__main` → `__scatterload` |
| Version constant | **0x96 (150)** @ 0x08010000, string `VNSD-0` @ 0x08010004 |
| Version string | `VNSD-0 v150` (among others @ 0x08051D90) |
| Build stamp | `Aug  5 2026 11:51:21` |
| Image footer | `VNSD` + null bytes at end of 380-KB payload, then 0xFF padding to 0x08063FFF |

**Scatter-load table** `Region$$Table$$Base` @ 0x08061EEC (`scatter_region_t[3]`, directly
verified in raw image):

| # | Source (flash) | Target | Length | Method |
|---|---|---|---|---|
| 1 | 0x0806240C | 0x20000000 (.data) | 0x2F00 (12 032 B) | `__decompress` @ 0x08005FBE (LZ77) |
| 2 | 0x08062DC4 | 0x60000000 (.xmc_bss) | 0x26AE0 (157 952 B) | `__decompress` @ 0x08005FBE (LZ77) |
| 3 | – | 0x20002F00 (.bss) | 0x1C8E8 (117 000 B) | `__scatterload_zeroinit` @ 0x080367E0 (null-init) |

`__scatterload_copy` (0x080367D0, 2 B) jumps into separate 12-byte copy loop
`Scatterload_CopyBody` (0x080367D8) — in v151 merged into single 14-byte function,
functionally identical.

## 1a. MCU identification: Nations N32G45x, not Artery AT32F403A

- The Control/EMS MCU is a **Nations N32G45x with XFMC** (external memory controller), not the previously assumed Artery AT32F403A.
- Evidence: QSPI controller at 0xA0001000 with N32 QSPI_STS bits; RCC PLL multiplier in bits 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG in use; SAC crypto coprocessor at 0x40024000 (formerly misnamed "RNGX"); 96-bit UID @ 0x1FFFF7F0; 102 vectors.
- External SRAM 0x60000000–0x60026AF7 runs through XFMC, control registers at 0xA0000000: the official Nations device header `n32g45x.h` (local only, not in the repo) defines `XFMC_REG_BASE 0xA0000000` and `QSPI_BASE = XFMC_REG_BASE + 0x1000`; the Nations driver library contains `n32g45x_xfmc.c`; the firmware itself has the shell command `xfmc_test` (the current English N32G45x user manual incorrectly marks this address space "Reserved").
- Details/header copy: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Processing Status in Ghidra (`/VNSD/Control/Control_150_VNSD-0_app_0150_0805_115146.bin`)

All verification steps passed (`marstek_gates.py` (local only, not in the repo), 2026-09-30):

| Check | Result |
|---|---|
| Undefined flash bytes / instructions without function | 0 / 0 |
| Code as data (jump targets, function pointers, table entries, literal "functions") | 0 |
| Functions | **1954**, all named, no `FUN_`/`tmp_`/duplicates |
| Prototypes | all set (no default signature) |
| Referenced RAM/XMC globals without names | 0 |
| Task names vs. `g_taskDefTable`, shell handlers vs. `shellCommandTable` | match |
| Duplicate global labels | 0 |

Some functions already carry plate comments with explicit v150↔v151 behavior comparisons
(e.g. `Grid_Export_Power_Limiter`, `Write_Handler`, `BLE_App_Command_Dispatcher`, `HTTP_GetUrlData`,
`CH395_Init_Modbus_TCP_Socket`) — these were used as primary source for present analysis
and spot-checked against raw image (including scatter table, task table,
RPC method table, see below).

## 3. FreeRTOS Tasks

Table `g_taskDefTable` @ **0x0805A764** (`TaskDef_t[14]`, format `{func, name, stackWords, param,
priority, handle*}`, 24 B/entry), created by `Task_CreateTasksAndStartScheduler` (0x0801E244).
Decoded directly from raw image:

| Task | Priority | Stack (words) |
|---|---|---|
| Task_ch395 | 15 | 256 |
| Task_ch_mqtts | 14 | 1024 |
| Task_parser | 13 | 512 |
| Task_ch_http | 13 | 1024 |
| Task_fc41d | 12 | 512 |
| Task_http | 11 | 512 |
| Task_udp | 10 | 256 |
| Task_ems | 9 | 256 |
| Task_ct | 8 | 512 |
| Task_Modbus_tcp | 7 | 512 |
| Task_can_comm | 7 | 256 |
| Task_can_receive | 6 | 256 |
| Task_rs485 | 3 | 512 |
| Task_iwdg | 1 | 128 |

All priorities and stack sizes identical with v151 (`VNSD_Control_Diff_v150_v151.md` confirms
this explicitly as unchanged). `Task_Modbus_tcp` (priority 7) falls below both TLS tasks
(`Task_ch_mqtts` 14, `Task_ch_http`/`Task_parser` 13) — Modbus TCP can wait during TLS activity.

## Chinese Strings

119 UTF-8 strings with Chinese text in export, all translated with `ZH->EN` comment (2 fewer than
v151, which additionally has 2 new strings for shell commands `read_inv1`/`read_inv2` there).
Two Chinese debug `printf` remain in use code path untranslated (not pure comment, but runtime
output): in `Write_Handler` test branch for register 47400 —
`"47400要写进去里面的数据 = 0x%x\r\n"` ("Value to write into 47400 = 0x%x") and
`"没到里面吗123456\r\n"` ("Didn't go in, or? 123456", obvious debug remainder).

## 4. Interfaces

### 4.1 Modbus (TCP 502 over CH395, RTU over RS485)

- Read descriptor table `g_modbusRegDescTable` @ **0x20000354**, `ModbusRegDesc_t[246]`
  (`reg_addr`, `pad`, `source`, `type_code`, `type_code2`, `scale`, `count`), registers 30000–38014 —
  byte-identical across entire v149→v150 jump (`FC03_Read_Handler` @ 0x0802386C,
  `Read_Serializer` @ 0x08055414, unchanged since v149).
- Write path `Write_Handler` @ **0x08056514** (3998 B): register **46000** (internally `reg−45000 = 1000`)
  with value **0x5100** sets `g_debugMode = 1`, persistent in EEPROM 0x900 — permanent debug mode.
  Values 0x04D2/0x0929/0x0D80/0x11D7 each select YMODEM update target (`g_otaTargetDevice` 1/4/3/2).
  Register 47400 (test branch) still has Chinese debug `printf` from above.
- Register 0xB799 (47001, AI mode/`CAN_Write_Category0x18`) and sub-indexing shifted by one register
  in 45000 block don't exist in v150 **yet** — both come in v151
  (`VNSD_Control_Diff_v150_v151.md`).

### 4.2 Local API / MQTT-RPC (`g_mqttRpcMethodTable` @ 0x0805D944, `MqttRpcMethod_t[16]`)

Decoded directly from raw image, dispatcher `MQTT_JSON_RPC_Dispatcher` @ 0x0801F36C:

| ID | Method | ID | Method |
|---|---|---|---|
| 1 | Marstek.GetDevice | 9 | EM.GetStatus |
| 2 | Wifi.GetStatus | 10 | Marstek.GetData |
| 3 | BLE.GetStatus | 11 | DOD.SET |
| 4 | Bat.GetStatus | 13 | Ble.Adv |
| 5 | PV.GetStatus | 14 | Led.Ctrl |
| 6 | ES.GetStatus | 15 | Ota.Set |
| 7 | ES.SetMode | 16 | Set.Ver |
| 8 | ES.GetMode | 17 | Reset.Factory |

16 entries, ID 12 (0xC) not assigned (no gap in table, just skipped).
Per embedded comparison comment, this function substantively identical with v151 (only
v151 change: EEPROM persistence of SEL LED flag 0x395 at `Led.Ctrl`, not present in v150 —
set value doesn't survive reboot in v150).

### 4.3 Debug Shell (letter-shell, `shellCommandTable` @ 0x08061F1C, `ShellCommand[79]`)

79 entries = **66 commands** (68 `ShellCmd_*` handlers including `help`/`clear`) + 1 user entry
(`VNSD-0 v150`, empty password) + 10 key bindings. Versus v149/v149.2 (`ShellCommand[78]`,
65 commands) new: **`FreeRtos_mem`** (0x0802F540, outputs current and historical minimum heap,
`FreeRTOS_xPortGetFreeHeapSize`/`_xPortGetMinimumEverFreeHeapSize`) — one of 20 new functions of
v149.2→v150 step, see §7/§9. Versus v151 (70 commands) missing `read_inv1`/`read_inv2`.

Accessibility: `Task_rs485` serves per `g_debugMode` (EEPROM 0x900) Modbus RTU (0) or shell
on RS485 and CH395 TCP server **port 8091** (`g_ch395SockModbusTcp` table @ 0x2001ADDC, socket 1);
mode 2 waits for string `quit_debug_5100_` (`RS485_DebugConsole_ScanQuitPhrase` @
0x080550DC), then mode 1. Shell command `modbus` switches back to 0.

### 4.4 Cloud

All endpoints verified in raw image:

| Endpoint | Schema | Purpose |
|---|---|---|
| `http://%s.hamedata.com/app/neng/getDateInfo%s.php?...` | HTTP, port 80 | Version/time sync (`sv=0x96`=150 hardcoded) |
| `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` | HTTPS | Telemetry upload — **new in v150** (see below) |
| `http://%s.hamedata.com/ems/api/v1/getDevicesApply?devid=%s` | HTTP | WiFi config path |
| `http://%s.hamedata.com/external-services/api/v1/devices/backup-configs?devid=%s&latest=true` | HTTP | Cloud backup configuration |
| `https://%s.hamedata.com/external-services/api/v1/schedulings/policies?devid=%s` | HTTPS | Schedule/policy synchronization |
| `http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl` | HTTP | FC41D module OTA |

**Telemetry upload new in v150 (detail: `VNSD_Control_Diff_v1492_v150.md`):** v149.2 sent telemetry packet still AES-128-ECB-encrypted and base64URL-encoded as query parameter `v=` via cleartext HTTP-GET to `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s`. v150 forgoes encryption and encoding, wraps payload only in `{"d":"..."}` and sends via new HTTPS/TLS client (mbedTLS over CH395: `HTTPS_FetchUrlData`, `HTTPS_TLSSessionInit`,
`HTTPS_PackPostRequest`, port 443) as JSON-POST to above marstekcloud.com endpoint (Ethernet path;
for mobile `Quectel_HTTP_POST_SendAndVerify` sends same JSON body per `AT+QHTTPPOST`). Thereby fixes real buffer overflow from v149.2: `Base64URL_Encode` was called there with claimed target buffer size 0x960 (2400 B), but actual buffer `g_cloudUploadBodyBuf` is only `uchar[1200]` (1200 B) — on AES outputs with base64 length between 1200 and 2400 B the function wrote past buffer end into adjacent globals. Explains changelog item "data loss on long HTTP records fixed" directly as memory-overflow fix, not just efficiency gain.
All other `hamedata.com` endpoints remain unchanged in this step.

Region table (EEPROM 0x441, `g_cloudDomainUrlTable` @ 0x20000FD0): `eu, www, us, eu-dev, eu-staging`,
**valid only 0–2** (`HTTP_GetUrlData` @ 0x080199AC silently resets any other EEPROM value to 0 — only v151 expands check to 0–4). MQTT topics: `marstek_energy/%s/device/%s/ctrl`
and `marstek_energy/%s/App/%s/ctrl`, TLS over mbedTLS with `authmode 2` (§8).

### 4.5 BLE (Quectel FC41D, AT commands)

Device name `MST_VNSD_`+4 characters (`BLE_Set_Device_Name` @ 0x0800F464, `AT+QBLENAME=`).
Command dispatcher `BLE_App_Command_Dispatcher` @ 0x0800C720 (7062 B, ~40 sub-commands) — v149.2 → v150 adds
command `)` (0x29, peak shaving) and fallthrough bug (`Y`→`Z`) fixed, otherwise command set
since v149 unchanged, only response/telemetry construction changed (§7). BLE-OTA: `BLE_Cmd_OTA_Init` (0x08033C48),
`_WriteSetup` (0x08033CDC), `_Validate` (0x08033B68), `BLE_OTA_WriteDataChunk` (0x080340E0) — path byte-identical since
v149, BLE firmware update therefore unrestricted.

## 5. Power Limits (for the 2500-W question)

- `Config_ClampDefaults_ParamBlock` @ **0x0800935C**: `g_maxChargePowerW`/`g_maxDischargePowerW` clamped
  at 0 or > 0x9C4 (2500 W) to **0x9C4 = 2500 W** (discharge fallback otherwise 800 W +
  `g_workMode = 1`). `g_chargeCutoffSoc` default 100 %, `g_dischargeCutoffSocRaw` range 12–70 %,
  default 12 %.
- `Set.Ver` (MQTT-RPC method 16, `MQTT_JSON_RPC_Dispatcher` case 0x10): only 800, 0x898 (2200 W) or
  0x9C4 (2500 W) accepted, else `Invalid params` — identical with v149.
- `Grid_Export_Power_Limiter` @ **0x08023A00** (every 4 ticks): PV ceiling at feed-in
  (`g_gridSamplePowerW ≥ 1`) = **`min(-P_bms, 2500 W)`** (0x9C4 clamp direct in code), else
  `min(cap, ac + lim + 300)`; total clamp in any case **4000 W**. Only v151 replaces
  feed-in start value with `grid measurement − P_neg` without 2500-W clamp (total clamp 4000 W stays).
- BMS current limit → power conversion (`P_bms = 32100 × 32106`) and interplay with PV surplus,
  force-charge (42020) and `charge_to_soc` (42011, sets `force_mode` to 3) in detail:
  `VNSD_Control_Charge_Power_PV_Curtailment_v150.md` — also field assessment there that 2500-W PV throttle at top-of-charge taper of battery (≈1270→≈485 W, see BMS doc) could be noticeable, without field measurement of actual PV loss.

## 6. Structure/Data Findings

- mbedTLS cipher table `aes_info`/`aes_128/192/256_ecb_info`/`gcm_aes_info`/`aes_128/192/256_gcm_info`
  from **0x0805D9C4**, `mbedtls_cipher_definitions[7]` @ 0x0805DA9C — structure identical with other
  versions, only relocated.
- **BMS pack count (applies for 147–151, field observation see below):** 8 pack records in RAM starting
  area **0x200177d4** (stride in v150/v149.2/151 = **96 B**, in v149 still 88 B —
  `BMS_CellVoltage_MinMax_PerPack` @ **0x08017B14** enlarged record in v149→v149.2 step),
  CAN battery count limited to 8, Modbus blocks 34000…34600 (7 packs). Cell statistics run only over
  **6 packs** (`pack < 6`, table `g_bmsCellStat0…` from 0x20018FC0, 6 × 6 B) → 7th pack (Modbus
  present) missing in `g_bmsCellStat*` and built-from MQTT telemetry.
- **Pack numbering / off-by-one in MQTT path (applies 147–151):** CAN unit u → record u−1
  (`CAN_AA_StoreUnit*`); Modbus 34000+100·n and `ShellCmd_get_all_bms_data` read consistent record n.
  `MQTT_Publish_BMS_Data` @ **0x080288E8** (MQTT command 0x2A) reads for index p ≥ 1 record p
  (index 0 = PGN-0x1801/0x1802 totals, 0xFF = fp64-scaled overview) → "BMS(1)" per MQTT shows
  CAN unit 2, unit 1 individually unreachable per MQTT. Index 8 reads past record array;
  from there lies in v150 `g_pendingBootDescriptor` @ **0x200152D4** (boot descriptor: parameters, CRC16,
  version tag) — read-only access, no memory violation.
- Factory WiFi credentials `SM471`/`Hame147258` at three places in code (string literals
  @ 0x080169B8/0x0802E0D0/0x08052DBC): `Config_LoadOrInit_DeviceSettings` (0x0802DCBC),
  `Quectel_WiFi_AutoConnect_StateMachine` (0x0801677C), `ShellCmd_set_wifi` (0x08052D80).
- **Field observation (cloud upload payload, not re-verified this session):** field capture of
  HTTP upload payload including verified field names (`wm`, `sc`, `pb`, `bv`, `bi`, `go`, `mc`/`md` etc.)
  in `VNSD_Cloud_Upload_Payload_v150.md`; also region `eu` and `getDateInfo` parameters there
  measured confirmed.

## 7. Only in v150 / Changed vs. Predecessor (149/149.2) and Successor (151)

**v149 → v149.2 (detail: `VNSD_Control_Diff_v149_v1492.md`, 43 functions), taken over unchanged to v150:**
- BMS pack record 88 → 96 B (§6).
- French self-use controller SMR/TIC (Linky) greatly simplified (408 → 247 instructions).
- Rebuild cloud/MQTT telemetry, status reporting, telemetry history, UDP meter parse (register
  and behavior interfaces unchanged, internal structure only).

**v149.2 → v150 (detail: `VNSD_Control_Diff_v1492_v150.md`; 56 of 1934 named-paired functions
differ, 26 substantively, plus 20 new and 1 removed function):**
- **Cloud telemetry upload completely rebuilt** (`Cloud_UploadTelemetryData`) — new HTTPS/TLS client
  (mbedTLS over CH395: `HTTPS_FetchUrlData`, `HTTPS_TLSSessionInit`, `HTTPS_TLS_Context_Reset`,
  `HTTPS_PackPostRequest`, `HTTPS_ParseData`/`_ParserOnBody`, `CH395_TCP_Socket_Open_HTTPS`,
  `CH395CMDGetSocketStatus`, `mbedTLS_SSL_Send/Recv_WithRetry`) replaces AES-128-ECB+base64URL-GET to
  hamedata.com with JSON-POST to marstekcloud.com and fixes real buffer overflow in
  `Base64URL_Encode` (§4.4).
- **Peak-shaving/power clamp mode new** (EEPROM 0x394, 6 B): `Config_Read_/Write_PowerClampMode_0x394`,
  new BLE command `)` (0x29), new MQTT commands 0x3f (set)/0x41 (`MQTT_Publish_Grid_Status`,
  read), acts in `Inverter_PowerSetpoint_DeadbandClamp` and `Remote_Power_Setpoint_Process` additionally
  to unchanged 2500-W hard limit (§5).
- **Two new connection watchdogs** plus GET→POST rebuild of mobile upload: `HTTP_ClientTask_Dispatcher`
  triggers CH395 chip reset after 1800 ticks no progress, `HTTP_Task_StateMachine_Handler` modem reset after
  900 ticks; states 6–9 of modem state machine now send `AT+QHTTPPOST`
  (`Quectel_HTTP_POST_SendAndVerify`) instead of GET.
- `DeviceInfo_BuildStatusString`: 6 new hardware variant tags for meter types 3/4/8/9/10/11 instead of
  previously generic EEPROM string.
- `Grid_PowerSync_StateMachine`: meter type range for CT direct takeover without deviation gate greatly
  expanded.
- BMS self-test/MOS flag additionally reported (`BLE_Build_BMS_Data_Response`, `BMS_StatusQuery_Response_Send`).
- Off-by-one bug fixed: `Config_Read_String_0x387`/`_Write_String_0x387` despite names accessed
  EEPROM 0x388 instead of 0x387.
- `HTTPS_POST_Request`: receive buffer 6488 → 7616 B, now shared with HTTPS path.
- `HTTP_ParseUploadResponseCode`: now searches JSON key `"code":` instead of `"de":` — different
  key, not just moved read offset.
- Six minor behavior changes: `MQTT_Publish_Network_Info` reports actual instead of queried meter IP; `Quectel_MQTT_Config_StateMachine`/`_Session_StateMachine` change TLS config order or condition factory reset in SSL watchdog; `mbedTLS_SSL_Connection_Init`
  loses redundant TLS reset; `Battery_EPS_Reserve_SOC_Hysteresis` newly sets low byte in
  CAN-0xCF report to 1 (meaning open); `Cloud_EdgeDetectAndWatchdog` binds retry additionally
  to MPPT channel; `Quectel_HTTP_AT_SendAndVerify` gets additional wait after send.
- **Correction vs. earlier version of this diff step:** `CH395_TCP_Socket_Open_HTTPPort80`
  (formerly rated as parameter change 0x24→0xAA) and `HTTP_PackGetRequest` (formerly as
  buffer enlargement 360→640 B) are actually **cosmetic** — both function bodies byte-identical,
  changed values only `__LINE__` log arguments.
- New shell command `FreeRtos_mem` (`ShellCmd_FreeRtos_mem`, §4.3) among 20 new functions
  of this step and recorded in diff doc — no open contradiction (§9).

**v150 → v151 (detail: `VNSD_Control_Diff_v150_v151.md`):** new Modbus write register 0xB799 +
`CAN_Write_Category0x18`, Modbus 32102 `float32`→`i16`, `getDateInfo` host switches to
`marstekcloud.com`, region table 0–4 instead of 0–2, CH395 keepalive IDLE 10 000→60 000 ms,
CH395 hang watchdog now triggers full reboot, YMODEM type check, LED status becomes
EEPROM-persistent, PV feed-in start value without 2500-W clamp (§5), new shell commands `read_inv1`/
`read_inv2`. Complete table with old/new values in diff doc.

## 8. Security Findings

1. **Unauthenticated remote console:** Modbus TCP write access (port 502, no auth) to register
   **46000 = 0x5100** → `g_debugMode = 1` persistent in EEPROM (0x900) → letter-shell on RS485 **and**
   TCP port 8091 (`g_ch395SockModbusTcp` socket 1, @ 0x2001ADDC), user `VNSD-0 v150` with empty
   password. Magic word `quit_debug_5100_` (`RS485_DebugConsole_ScanQuitPhrase` @ 0x080550DC) also enables
   mode 2 alike.
2. **Hardcoded WiFi credentials** `SM471`/`Hame147258` (§6).
3. **HTTPS upload path without server certificate check:** `mbedtls_ssl_conf_authmode(…, 0)` in
   `HTTPS_TLSSessionInit` @ 0x0803B8A0; MQTT checks with `authmode 2` (`mbedTLS_SSL_Connection_Init` @
   0x0801D134).
4. **Certificate validity period never checked:** `mbedtls_x509_time_is_past` (0x0804DCCA) and
   `mbedtls_x509_time_is_future` (0x0804DCC4) are pure `return 0` stubs
   (`MBEDTLS_HAVE_TIME_DATE` off) — affects MQTT too.
5. Cloud telemetry report (`getDateInfo`) continues over cleartext HTTP (port 80) to `hamedata.com`
   (§4.4); actual telemetry upload already on HTTPS/`marstekcloud.com`.
6. **Buffer overflow FC03 (applies also to v150, see v151 doc §11):** `FC03_Read_Handler` @ 0x0802386C
   writes response (up to 128 B, N ≤ 125 registers) in TX frame and copies it afterwards into mirror buffer, which from offset 32 goes into MQTT-RPC workspace — structure spot-checked in v150 export
   (max register count 0x7D like v151), full byte-by-byte recalculation not repeated.

## 9. Open Items

- As in neighbor versions unchanged and not re-proven in detail here: 32101 sign error in read serializer, MBAP length 4 instead of 3 at exception responses, 44002/44003 read 0 after reboot (shadow value) — from diff docs/v151 full analysis, not re-traced step-by-step for v150.
- How much PV power 2500-W throttle actually costs at battery top-of-charge taper not measured (`VNSD_Control_Charge_Power_PV_Curtailment_v150.md` §7).
- 31 unmatched fields in cloud upload payload (`VNSD_Cloud_Upload_Payload_v150.md`).
- `__vfscanf` (newly linked in v149.2→v150 step C library function): triggering call site not located in code (`VNSD_Control_Diff_v1492_v150.md`).
- `ShellCmd_FreeRtos_mem`: linked via shell command linker section instead of normal call table, no code cross-reference found (function itself clear: outputs free/minimum-ever-free FreeRTOS heap, §4.3).
- Fallthrough fix at BLE command `Y`→`Z` (§7): whether misalignment observable in v149.2 (missing `break`) actually triggered or was consequence-free was not verified.

## 10. Corrections

**Corrections to this document (this revision, 2026-09-30):** Earlier version wrongly downrated
v149.2 → v150 as "three minor items plus new shell function" and took over incomplete function pairing from then-earlier version of `VNSD_Control_Diff_v1492_v150.md`
(only 9 different functions found). Complete name pairing shows: 56 of 1934 function pairs
differ, 26 substantively, plus 20 new and 1 removed function — including completely new HTTPS/TLS
client replacing cloud upload plus fixed buffer overflow in v149.2,
peak-shaving and two new watchdogs (§0, §4.4, §7). Of three previously "the 3 behavior changes"
listed two were misevaluated: `CH395_TCP_Socket_Open_HTTPPort80` and
`HTTP_PackGetRequest` cosmetic (only `__LINE__` log arguments), `HTTP_ParseUploadResponseCode`
changes JSON key not read offset. New shell command `FreeRtos_mem` was noted as contradiction to diff doc;
resolved — now recorded among 20 new functions.

**Ghidra corrections this session (version-spanning, also affect v150 export):** all 8
BMS pack records uniformly named as `g_bmsPack<k><Field>` (in v149 one function was still
`CAN_AA_StoreUnitFieldPair88`, now `…Pair72`); ~300 global names version-spanning harmonized — in
this document affects `g_cloudServerRegionTable` → `g_cloudDomainUrlTable` (§4.4); short previously number-typed strings now string-typed; SPI-flash command table
`g_spiFlashStatusCmdTable` and `g_meterCtStabilizeDelayTable` typed; previously undocumented
peripherals named (block 0x40024000, back then `RNGX_*`, meanwhile identified as the N32G45x SAC crypto
coprocessor and renamed to `SAC_R00`/`R04`/`R0C`/`R10`/`R14`, used by `Flash_ReadWords`/`Flash_ReadWithECC`;
GPIOF/GPIOG @ 0x40011C00/0x40012000; `RCC_UNDOC_2C`, `PWR_UNDOC_0C`, `RTC_UNDOC_4C`; bit-band
aliases `BB_*`). New verification gates (tmp-label, placeholder globals, shell plate match) pass for all
versions including v150; decompilation export has 0 errors.

Corrections documented for v147 (`VNSD_Control_FW_Analysis_147_202601281721320b2053125.md` §10)
concern artifacts already cleaned up pre-export for v150 (task trampolines,
cipher range misnaming, `CH395_ConfigTcpKeepalive_Defaults`, `MQTT_Transport_NullCallback`).
