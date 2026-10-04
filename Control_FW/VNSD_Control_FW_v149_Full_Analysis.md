# Control-FW v149 (Venus D) — Full Analysis

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_v149_Vollanalyse_de.md](VNSD_Control_FW_v149_Vollanalyse_de.md)

**As of:** 2026-09-30.
**German:** `VNSD_Control_FW_v149_Full_Analysis.md` ·
**Diff to predecessor:** `VNSD_Control_Diff_v147_v149.md` · **Diff to beta successor:** `VNSD_Control_Diff_v149_v1492.md` ·
**Diff to main successor:** `VNSD_Control_Diff_v149_v150.md` · **Decompilation/Listing:** `Dekompilate/VNSD_Control_149/` (local only, not in the repo)

## 0. Summary

- Image base **0x08004800** (18-KB bootloader below), Nations N32G45x (previously misidentified as Artery
  AT32F403A, see §1a), Keil ARMCC, FreeRTOS, mbedTLS 2.28, cJSON, llhttp, letter-shell, CH395, Quectel FC41D.
  Same codebase as v147/v150/v1492/v151.
- v149 is the first version to introduce **2500 W** power limit, **16-method Local API/MQTT-RPC**, the
  **246-register Modbus image** (30000–38014), and **French meter binding (TIC/Linky)** —
  all still present unchanged in v151 (§0a, §7).
- Security posture same as in predecessor/successor: Modbus register 46000 = 0x5100 activates debug shell on RS485
  **and TCP 8091** without authentication permanently (§8).

## 0a. Verification checklist (v149 vs predecessor v147 and successor v150)

| # | Question | v149 Answer | Evidence |
|---|---|---|---|
| 1 | Modbus registers changed? | vs v147: **15 new** (38000–38014, cumulative energy counters), none removed, 231 → **246** entries; new write block 46500–46544 (`Write_Handler`, `(reg-46500)/3`). vs v150: register contents **unchanged** (register map/descriptor table identical across v149→v149.2→v150, only RAM shadow addresses shift); however, schedule EEPROM from register 46500 moves from 0x36BF to 0x3700 in step v149.2→v150. | §4.1, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 2 | BLE changed? | vs v147: command set expanded by 0x5A (trigger cloud backup) and 0x5B (IP/MAC/SN); 0x54 additionally resets EPS reserve flags. vs v150: command set **grows by 1** – new command 0x29 "Set Peak Shaving" (added in step v149.2→v150); response/telemetry structure also changes (step v149→v149.2) plus BMS self-test/MOS flag and fallthrough bugfix Y→Z (step v149.2→v150). | §4.5, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 3 | AC feed-in limit 2500 W changed? | vs v147: **yes, introduced** (2200 W → 2500 W, `Config_ClampDefaults_ParamBlock`); `Config_Apply_WorkModePowerPreset` removed, replaced by RPC `Set.Ver` (800/2200/2500 W). vs v150: hard limit **unchanged**, `Grid_Export_Power_Limiter` byte-identical across both hops; **additionally** (does not replace 2500-W limit) a separate, configurable peak shaving clamp arrives in step v149.2→v150 (EEPROM 0x394), see §5. | §5, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 4 | BLE-OTA still possible? | **Yes**, in both directions unchanged: `BLE_Cmd_OTA_Init/WriteSetup/Validate`, YMODEM and all `OTA_*` functions are v147↔v149↔v149.2↔v150 byte-identical or functionally equivalent; YMODEM target codes 0x04D2 (EMS), 0x0929 (VENUS) unchanged (v149 additionally knows 0x0D80 and 0x11D7 as additional target devices, see §4.5). | §4.5, Diff v147→v149, Diff v149→v1492, Diff v1492→v150 |
| 5 | Cloud domains changed? | vs v147: **unchanged** (`%s.hamedata.com`), new only the path `/external-services/api/v1/devices/backup-configs`. vs v150: `hamedata.com` endpoints themselves **unchanged**, but telemetry upload (`Cloud_UploadTelemetryData`) switches in step v149.2→v150 to new HTTPS/JSON-POST at `marstekcloud.com`, replacing AES-ECB+Base64URL-encoded GET request to `hamedata.com` (thereby fixing a real buffer overflow, see §7). | §4.4, Diff v147→v149, Diff v1492→v150 |

## 1. Binary Fingerprint

| Feature | Value |
|---|---|
| File | `149_control_VNSD-0_app_0149_0521_094621.bin` |
| Size | 385 024 bytes (0x5E000) |
| SHA-256 | `a8e5e9285e6ca5a327444d1bd942ece22a7955159e729b0dab0106f2bc8f8ace` (recomputed via `shasum -a 256`) |
| Version | Constant **0x95 (149)** @ 0x08010000, string `VNSD-0` @ 0x08010004, version string `VNSD-0 v149` (among others @ 0x08050f50, @ 0x0805af21) |
| Load | Base 0x08004800, vector table there (SP = 0x2001F778, Reset = 0x08004A71); `Reset_Handler` → `SystemInit` (sets `SCB->VTOR = 0x08004800`) → `__main` → `__scatterload` → `main` |
| Scatter table | `Region$$Table$$Base` @ 0x08060E58, 3 entries × 16 B (source, destination, size, routine): **.data** 0x08061368 → 0x20000000, 0x2EEC B, `__decompress` @ 0x08005FBE; **XMC (external SRAM)** 0x08061D18 → 0x60000000, 0x242E4 B, `__decompress`; **.bss (ZI)** → 0x20002EEC, 0x1C88C B, `__scatterload_zeroinit` @ 0x08035BB0. .bss end = initial SP 0x2001F778 |
| Release note (Changelog) | "Added further OpenAPI functions; ES delivered PV=0 and UPS false alarm fixed; I²C deadlock recovery; French meters; HTTP upload extended by IP/cumulative energy (VE3 only); excess feed-in optimized; DOD transfer fixed; upload interval 5 min; max power configurable; Venus D supports 2500 W" (Issue #4) |

## 1a. MCU identification: Nations N32G45x, not Artery AT32F403A

- The Control/EMS MCU is a **Nations N32G45x with XFMC** (external memory controller), not the previously assumed Artery AT32F403A.
- Evidence: QSPI controller at 0xA0001000 with N32 QSPI_STS bits; RCC PLL multiplier in bits 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG in use; SAC crypto coprocessor at 0x40024000 (formerly misnamed "RNGX"); 96-bit UID @ 0x1FFFF7F0; 102 vectors.
- External SRAM 0x60000000–0x60026AF7 runs through XFMC, control registers at 0xA0000000: the official Nations device header `n32g45x.h` (local only, not in the repo) defines `XFMC_REG_BASE 0xA0000000` and `QSPI_BASE = XFMC_REG_BASE + 0x1000`; the Nations driver library contains `n32g45x_xfmc.c`; the firmware itself has the shell command `xfmc_test` (the current English N32G45x user manual incorrectly marks this address space "Reserved").
- Details/header copy: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Preparation status in Ghidra (`/VNSD/Control/Control_149_VNSD-0_app_0149_0521_094621.bin`)

All verification steps passed (`marstek_gates.py` (local only, not in the repo), 2026-09-30):

| Check | Result |
|---|---|
| Undefined flash bytes / instructions without function | 0 / 0 |
| Code as data (jump targets, function pointers including table entries, literal "functions") | 0 |
| Functions | **1931**, all named, no `FUN_`/`tmp_`/duplicates |
| Prototypes | all set (no default signature) |
| Referenced RAM/XMC globals without names | 0 |
| Task names vs `g_taskDefTable`, shell handlers vs `shellCommandTable` | match |
| Duplicate global labels | none |

## 3. FreeRTOS Tasks (`g_taskDefTable` @ 0x0805990C, `TaskDef_t[14]`, 24 B/entry)

Decoded from flash image (function, name, stack in words, priority):

| Task | Priority | Stack (words) |
|---|---|---|
| Task_ch395 | 15 | 256 |
| Task_ch_mqtts | 14 | 1024 |
| Task_parser | 13 | 512 |
| Task_ch_http | 13 | 512 |
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

`Task_CreateTasksAndStartScheduler` creates all 14 tasks from this table and starts the scheduler; priority/stack
are identical to v151 (documented there in tabular form) — **unchanged across v149–v151**.

## 4. Interfaces

### 4.1 Modbus (TCP 502 over CH395, RTU over RS485)

- Descriptor table `g_modbusRegDescTable` @ RAM **0x20000348**, type `ModbusRegDesc_t[246]` (2952 B) — verified
  (v147 had 231 entries; v149 adds 15 registers **38000–38014**, cumulative energy counters).
- Write path `Write_Handler` @ 0x080556D4: special registers from 45000 evaluated per `(reg-45000)` index.
  - **Register 46000** (index 1000): value `0x5100` → `g_debugMode = 1`, permanently stored in EEPROM 0x900.
    Value `0x4D2` (1234) → `g_otaTargetDevice = 1` (EMS); `0x929` (2345) → `= 4` (VENUS); additionally (not mentioned
    in known facts, verified here) `0xD80` (3456) → `= 3` and `0x11D7` (4567) → `= 2` as additional
    YMODEM target devices.
  - New block **46500–46544** (0xB5A4–0xB5D0, `(reg-46500)/3`/`%3`, 15×3 registers) reads/writes
    `g_canScheduleRegCache`/`g_canScheduleRegCacheValueHi` (schedule power setpoints per slot) — not present in v147.
- Variables without domain names are called `g_mbReg<Register>`.

### 4.2 Local API / MQTT-RPC (`g_mqttRpcMethodTable` @ 0x0805CACC, `MqttRpcMethod_t[16]`)

All 16 methods read from flash image (name, internal ID):

| ID | Method | ID | Method |
|---|---|---|---|
| 1 | Marstek.GetDevice | 0xB | DOD.SET |
| 2 | Wifi.GetStatus | 0xD | Ble.Adv |
| 3 | BLE.GetStatus | 0xE | Led.Ctrl |
| 4 | Bat.GetStatus | 0xF | Ota.Set |
| 5 | PV.GetStatus | 0x10 | Set.Ver |
| 6 | ES.GetStatus | 0x11 | Reset.Factory |
| 7 | ES.SetMode | | |
| 8 | ES.GetMode | | |
| 9 | EM.GetStatus | | |
| 0xA | Marstek.GetData | | |

Compared to v147 (11 methods), **Ble.Adv, Led.Ctrl, Ota.Set, Set.Ver, Reset.Factory** are new
(`MQTT_JSON_RPC_Dispatcher` @ 0x0801E844, case 0xD/0xE/0xF/0x10/0x11). `Set.Ver` (case 0x10) accepts exactly
800/0x898 (2200)/0x9C4 (2500) W and writes `g_workMode`; `DOD.SET` (case 0xB) checks range 0x1E–0x58 (30–88 %).

### 4.3 Debug Shell (letter-shell, `shellCommandTable` @ 0x08060E88, `ShellCommand[78]`)

78 table entries (v147: 76) — 67 `ShellCmd_*` handler functions (65 commands + `help` + `clear`) plus
key bindings without their own handler. Compared to v147, new: **`get_backup_datas`**, **`power_check`** (confirmed v147→v149 diff).
Access via RS485 debug console and CH395 TCP server port 8091 (`CH395_Init_TCPServer_Socket`,
`g_ch395SockTcpServer8091_localPort = 0x1F9B` = 8091) after activation (§8).

**Corrected (2026-09-30):** The plate comments of four handlers `update_bms/ems/mppt/vns` were shifted by one position
(e.g., `ShellCmd_update_bms` carried the text of `FreeRtos_mem` — this command does not exist until v150).
Function names were correct; the comments are corrected in Ghidra (same error in v149.2, also fixed).

### 4.4 Cloud

Endpoints under `http://%s.hamedata.com/…`: `/app/neng/getDateInfo%s.php`, `/ems/api/v1/getDevicesApply`,
`/prod/api/v1/setVenusDReporting` (telemetry upload), `https://%s.hamedata.com/external-services/api/v1/schedulings/policies`
and — **new compared to v147** — `/external-services/api/v1/devices/backup-configs?devid=%s&latest=true`
(`HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher`). FC41D module OTA: `http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl`.
MQTT over TLS (mbedTLS, authmode 2), topic `marstek_energy/%s/device/%s/ctrl` (app-side `.../App/%s/ctrl`).

### 4.5 BLE (Quectel FC41D, AT Commands)

Command dispatcher `BLE_App_Command_Dispatcher` @ 0x0800C31C. Compared to v147, new: **0x5A** (trigger cloud backup fetch,
`bk_en`), **0x5B** (IP/MAC/serial number); **0x54** (discharge limit) additionally resets EPS reserve flags.
OTA over BLE: `BLE_Cmd_OTA_Init`/`WriteSetup`/`Validate` (@ 0x0803301C/0x080330B0/0x08032F3C) + YMODEM
(`YModem_ReceiveFile`, `YModem_FirmwareUpgrade_Handler`) — BLE-OTA is possible and unchanged vs v147/v150.

## 5. Power Limits (for the 2500 W question)

- `Config_ClampDefaults_ParamBlock` @ 0x0800901C (verified): charge **and** discharge power are clamped to
  **0x9C4 = 2500 W**; default on invalid value: discharge 800 W, `g_workMode = 1`. (v147: 0x898 = 2200 W.)
- `Config_Apply_WorkModePowerPreset` from v147 (800/600/2200/1200 W per mode) **is removed in v149** — replaced by
  RPC method **`Set.Ver`** (`MQTT_JSON_RPC_Dispatcher` case 0x10): accepts only 800, 0x898 (2200), or
  0x9C4 (2500) W, writes `g_workMode` (1/3/0) and persists via `Config_Write_U16`/EEPROM 0x90.
- `Grid_Export_Power_Limiter` @ 0x08022ED8 (verified): feed-in/charge setpoint is clamped to `min(P_bms, 0x9C4)`,
  then to overall clamp **4000 W** — unchanged from v147 and v150 (see diff documents).
- The 2500 W hard limit persists through v151; **only in v150** does a separate, configurable
  peak shaving clamp arrive (EEPROM 0x394, BLE command 0x29, MQTT commands 0x3f/0x41) — it does not replace
  the 2500 W limit but acts additionally in `Inverter_PowerSetpoint_DeadbandClamp`/
  `Remote_Power_Setpoint_Process` (diff v1492→v150).

## 6. Structure/Data Findings

- mbedTLS cipher table directly after `g_mqttRpcMethodTable` (`aes_info` @ 0x0805CB4C, `aes_128/192/256_ecb_info`
  etc.) — same structure as in v147/v151.
- `Task_CreateTasksAndStartScheduler` @ 0x0801D720 creates all 14 tasks from `g_taskDefTable` and logs
  success/failure per task via `Log_Printf`.
- **BMS pack count (applies to 147–151):** 8 pack records in RAM, battery count limited to 8 on CAN, Modbus blocks
  34000…34600 (7 packs). **Pack record size in v149: 0x2C × 2 B = 88 bytes** per pack (verified via
  `CAN_AA_StoreUnitFieldPair0` @ 0x08033338, index step `iVar1 * 0x2c` on `uint16_t` basis; v147 had 80 B, from
  v149.2 it is 96 B). The plate comments of `CAN_AA_StoreUnitFieldPair*` handlers incorrectly stated 96 B and the
  handler for offset 0x48 was named `…Pair88` — both corrected in Ghidra (now
  `CAN_AA_StoreUnitFieldPair72`, comments with 88 B). All 8 record fields are now uniformly named
  `g_bmsPack<k><field>` (previously partly `g_paramStore_*`/`g_mbReg341xx`).
- `BMS_CellVoltage_MinMax_PerPack` @ 0x08017348 (previously incorrectly named `Schedule_MinMax_PerGroup_Calc`) runs only over
  **6 packs** (`uVar3 < 6`, 16 cells each from `g_bmsPack0CellVoltageArray`, step `uVar3*0x2c`) into table
  `g_bmsCellStat0…` (6 × 6 B) — the 7th pack (index 6) is missing from this statistic and the MQTT telemetry built from it,
  but is present via Modbus (34600 block). Applies to all versions 147–151.
- **Pack numbering / MQTT path offset (applies to 147–151):** CAN unit u → record u−1
  (`CAN_AA_StoreUnitFieldPair*`); Modbus 34000+100·n and `ShellCmd_get_all_bms_data` read record n (consistent).
  `MQTT_Publish_BMS_Data` for index p ≥ 1 reads record p instead — off-by-one (MQTT "BMS(1)" = CAN unit 2,
  unit 1 not individually retrievable via MQTT); index 8 reads past the record array (pure read access to
  `g_pendingBootDescriptor` area, no memory corruption).

## 6a. Chinese Strings

The export contains 127 UTF-8 strings with non-ASCII characters; **106 of them contain Chinese text and are already
translated to English in Ghidra via end-of-line comment `ZH->EN`** (e.g., shell help texts for practically every
`ShellCmd_*` command, FC41D/BLE reset messages, TLS error messages). The remaining 21 strings are purely
symbolic (arrows `←`/`→`, frame characters `◆◇■□`, degree sign `°`) and require no translation. Selection with
German translation:

| String (Original) | German |
|---|---|
| `*** 模组复位, ble_state: %d.` | *** Module (FC41D) reset, ble_state: %d. |
| `CRC校验不通过` | CRC check failed |
| `设备各项校验通过，本次升级成功！！！` | All device checks passed, update successful!!! |
| `恢复出厂设置（中控） 1:清除所有; 2:保留累计数据; 3:恢复证书` | Factory reset (central control): 1 clear all; 2 keep cumulative data; 3 restore certificates |
| `复位原因：独立看门狗复位` | Reset reason: independent watchdog reset |
| `[BLE] 余电上网中!!!!` | [BLE] Excess feed-in active!!!! |
| `47400要写进去里面的数据 = 0x%x` | Data to write into 47400 = 0x%x |
| `没到里面吗123456` | "Didn't it get through? 123456" (developer debug output in test branch of register 47400) |

The last two strings are in the `Write_Handler` test branch around register 47400 — the same location described in
the v151 document as "two Chinese debug `printf`"; in v149 both already carry a `ZH->EN` translation.

## 7. Only in v149 / Changed vs Predecessor v147 and Successors (Summary, Details in Diff Documents)

**New vs v147** (Diff `VNSD_Control_Diff_v147_v149.md`, 96 changed, 42 new, 23 removed functions):
- Power limit 2200 W → 2500 W, `Set.Ver` RPC instead of `Config_Apply_WorkModePowerPreset`.
- Local API/MQTT-RPC 11 → 16 methods (+ Ble.Adv, Led.Ctrl, Ota.Set, Set.Ver, Reset.Factory).
- Modbus 231 → 246 registers (38000–38014 cumulative energy counters); write block 46500–46544 new.
- French meter binding (TIC/Linky) in `SMR_TIC_SelfUse_PowerController`/`CT_GridPower_Controller` new.
- `Cloud_Report_FillGridData` now delivers real PV sum/annual capacity instead of fixed 0.
- DOD range check (30–88 %) in `Cloud_DOD_Command_Handler` new (v147: always success, no check).
- I²C EEPROM bus recovery (`EEPROM_I2C_BusRecovery`), PV excess charge boost now applied (v147: dead code).
- BMS pack record 80 B → 88 B; cloud backup config (BLE 0x5A, MQTT `bk_en`) new; shell +2 commands.

**Changed in beta successor v149.2** (Diff `VNSD_Control_Diff_v149_v1492.md`: **58 changed functions**
[38 content, 20 cosmetic], **4 new** [`CAN_AA_StoreUnitFieldPair88`, `CT_PowerSetpoint_Compute`,
`Config_Read_String_0x387`, `Config_Write_String_0x387`]): BMS pack record 88 B → 96 B (new SoH field);
CT/SMR-TIC setpoint controller split into two functions; new device alias name (EEPROM 0x387); cloud/MQTT
telemetry, status reporting and telemetry history rebuilt. **Modbus and feed-in limits remain unchanged; BLE
command set gets no new command ID but response/payload fields change** (§4.5 diff).

**Changed in main successor v150 (step v149.2 → v150)** (Diff `VNSD_Control_Diff_v1492_v150.md`: **56
changed functions** [26 content], **20 new**, **1 removed** — far more than the small version bump suggests,
v149.2 is **not** "practically v150"): cloud telemetry upload completely rebuilt — a new HTTPS/mbedTLS client
sends JSON-POST to
`https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` and replaces the previous AES-ECB+Base64URL-encoded
GET request to `hamedata.com`; thereby fixes a real buffer overflow (`Base64URL_Encode` was called in v149.2 with
alleged target size 0x960 = 2400 B, but actual buffer `g_cloudUploadBodyBuf` is only 1200 B). Other new functions/features:
peak shaving/power clamp function (EEPROM 0x394, BLE command 0x29, MQTT commands 0x3f/0x41, new function
`MQTT_Publish_Grid_Status`); two new watchdogs for hanging uploads (CH395 chip reset or modem hardware reset);
`DeviceInfo_BuildStatusString` gets fixed hardware tags for 6 additional meter types; `Grid_PowerSync_StateMachine`
expands meter type range for direct takeover; BMS self-test/MOS flag additionally reported; new shell command
`ShellCmd_FreeRtos_mem` (free/min-free FreeRTOS heap). **Correction vs earlier analysis version:** The previously
named three only differences "HTTP upload read offset +5→+7", "CH395 socket parameter 0x24→0xAA" and "HTTP GET
buffer 360→640 B" were pure `__LINE__` log arguments without behavior change; the actual change to `HTTP_ParseUploadResponseCode`
is a different JSON key (`"code":` instead of `"de":`). Modbus register contents, the 2500 W hard limit, OTA/BLE-OTA
and the `hamedata.com` endpoints themselves remain unchanged (§0a).

## 8. Security Findings (verified in v149)

1. **Unauthenticated remote console:** Modbus-TCP write access (port 502, no auth) to register 46000 = 0x5100
   → `g_debugMode = 1` permanently in EEPROM (0x900) → letter-shell on RS485 and TCP 8091
   (`CH395_Init_TCPServer_Socket`, port 0x1F9B = 8091) — identical to v147/v150/v151.
2. **Built-in WLAN credentials** `SM471` / `Hame147258` (`Quectel_WiFi_AutoConnect_StateMachine` @
   0x08015FB8, `Config_LoadOrInit_DeviceSettings` @ 0x0802CD0C).
3. Cloud upload over unencrypted HTTP (`http://%s.hamedata.com/prod/api/v1/setVenusDReporting`); payload
   AES-ECB + Base64URL (`Base64URL_Encode`, present through v149.2, removed from v150).
4. New cloud backup fetch path (`backup-configs`) also over cleartext HTTP.

## 9. Open Items

- Changelog point "HTTP upload interval 5 min": no difference from v147 found in code (already noted in v147 doc)
  — not verifiable.
- Exact meaning of 15 byte fields of telemetry history record not fully determinable from v149 alone (neutral
  offset names, see v147 doc §9).
- Whether server-side BMS index counting is 0- or 1-based (relevant to the MQTT off-by-one described in §6)
  cannot be determined from firmware alone.
- Two additional YMODEM target codes (0xD80 → target 3, 0x11D7 → target 2, see §4.1) are present in code but
  no shell/BLE/MQTT call path found in this export — which target device "3" or "2" actually is remains open.

## 10. Corrections

During v149 work, no new corrections to already "complete" versions were found; the corrections listed in §10 of the
v147 document (task trampolines, prototypes, letter-shell swaps, mbedTLS cipher range, task table misname,
`CH395_ConfigTcpKeepalive_Defaults`, `MQTT_Transport_NullCallback`) affect 149/149.2/150/151 together and are already
corrected there. Additionally corrected in Ghidra on 2026-09-30 (see §4.3, §6): shifted plate comments of `update_*`
shell handlers, wrong record size in `CAN_AA_StoreUnitFieldPair*` comments, name `…Pair88` → `…Pair72`, uniform
names of BMS pack records and version-spanning unified names of global variables (same object = same name as in v151).
