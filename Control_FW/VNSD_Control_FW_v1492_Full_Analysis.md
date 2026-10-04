# Control-FW v149.2 (Beta) (Venus D) — Full Analysis

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_v1492_Vollanalyse_de.md](VNSD_Control_FW_v1492_Vollanalyse_de.md)

**As of:** 2026-09-30. Replaces for all address and name specifications the earlier version
`VNSD_Control_FW_Analysis_app_1492_0702_142136.md` (Modbus register documentation from live scans, before base fix
0x08004800; flash addresses cited there are consistently 0x4800 too low, some names incorrect). These live measurements
themselves (register behavior) remain valid and are incorporated here insofar as confirmed.
**German:** `VNSD_Control_FW_v1492_Full_Analysis.md` · **Diff to predecessor v149:**
`VNSD_Control_Diff_v149_v1492.md` · **Diff to successor v150:** `VNSD_Control_Diff_v1492_v150.md` ·
**Decompilation/Listing:** `Dekompilate/VNSD_Control_1492/` (local only, not in the repo)

## 0. Summary

v149.2 is a **beta** between main versions v149 (2026-05-21) and v150 (2026-08-05), released
2026-07-02 (Issue #17: "Fixed incorrect data display due to missing HTTP uploads"). **It is NOT
practically identical to v150:** compared to v149, the beta step already changes 58 functions (38 content,
20 cosmetic) and adds 4 new ones; then until v150 follows **56 changed functions (26
content), 20 new and 1 removed** — including the telemetry upload switching from AES-ECB+Base64URL-
encoded GET request to `hamedata.com` to new HTTPS/mbedTLS-POST to `marstekcloud.com` (thereby fixing
a real buffer overflow), peak shaving clamp function (EEPROM 0x394) arrives, and two new
watchdogs respond to hanging uploads. Modbus register contents, the 2500 W hard limit, the
BLE command IDs and OTA/BLE-OTA are **unchanged** vs v149; the content changes in
v149→v149.2 step lie in cloud/MQTT telemetry, status reporting, BMS pack record (now 96 instead
of 88 B) and French self-consumption controller (SMR/TIC). Security posture as in all versions 147–151:
Modbus register 46000 = 0x5100 activates debug shell on RS485 **and TCP 8091**
without authentication permanently (§8).

## 0a. Verification Checklist for Update Decision (v149 → v149.2 → v150)

| # | Question | v149 → v149.2 | v149.2 → v150 |
|---|---|---|---|
| 1 | Modbus registers changed? | **Unchanged.** `RS485_Modbus_RegisterMap_Init`, `Modbus_Dispatcher`, read/write path byte-identical; descriptor table 246 entries (0x20000354) in both versions | **Register contents unchanged** (all 246 descriptors: number/type/scaling/count identical; only in 215 entries does RAM shadow address shift due to config struct growing by 6 B). However, schedule EEPROM from register 46500 moves from 0x36BF to 0x3700 |
| 2 | BLE changed? | **Command set same**, only response/telemetry structure: `BLE_App_Command_Dispatcher` (~145 ops changed), `BLE_Build_Cmd0x42_Response` (15 ops), `BLE_Build_BMS_Data_Response` (3 ops). No new/removed command | **Extended.** New command `)` (0x29) "Set Peak Shaving" (clamp mode + watt value, EEPROM 0x394); `BLE_Build_DeviceStatus_Response` additionally reports clamp mode/value (0xA4→0xA7 bytes); `BLE_Build_BMS_Data_Response`/`BMS_StatusQuery_Response_Send` additionally report self-test/MOS flag (0x50→0x51 bytes); fallthrough bug command `Y`→`Z` fixed |
| 3 | AC feed-in 2500 W changed? | **Unchanged.** `Grid_Export_Power_Limiter` byte-identical; all 4 locations of constant 2500 (0x9C4) same | **Hard limit unchanged** (`Config_ClampDefaults_ParamBlock`, `Grid_Export_Power_Limiter` continue byte-identical); additionally (separate, does not replace 2500 W limit) new configurable peak shaving clamp (EEPROM 0x394), acts in `Inverter_PowerSetpoint_DeadbandClamp`/`Remote_Power_Setpoint_Process` |
| 4 | Firmware update via BLE still possible? | **Yes**, completely byte-identical: `BLE_Cmd_OTA_Init/WriteSetup/Validate`, `YModem_FirmwareUpgrade_Handler`, all `OTA_*` | **Yes** (entire OTA path byte-identical) |
| 5 | Cloud domains changed? | **Unchanged.** Identical `hamedata.com` endpoint set in both versions | **`hamedata.com` endpoints themselves unchanged**, but telemetry upload (`Cloud_UploadTelemetryData`) switches to new HTTPS/JSON-POST at `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` (replaces AES-ECB+Base64URL-GET request to `hamedata.com`) — `marstekcloud.com` therefore **exists as of v150**, not from v151 |

Complete tables with evidence locations: `VNSD_Control_Diff_v149_v1492.md` (58 changed functions [38 content,
20 cosmetic], 4 new; focus cloud/MQTT telemetry, status reporting, French self-consumption controller) and
`VNSD_Control_Diff_v1492_v150.md` (**56 changed functions [26 content], 20 new, 1 removed** —
HTTPS/mbedTLS cloud upload to `marstekcloud.com`, peak shaving, two upload watchdogs etc.; an earlier
analysis version incorrectly rated only 3 pure `__LINE__` log constants as "real changes").
Details also in §7.

## 1. Binary Fingerprint

| Feature | Value |
|---|---|
| File | `1492_control_VNSD-0_app_1492_0702_142136.bin` |
| Size | 385 024 bytes (0x5E000) |
| SHA-256 | `9b24db59c4dbde6c17e1666a778c8f44618558ef2760268a4be95570fc450796` (matches changelog prefix `9b24db59c4db…`) |
| Version | Constant **0x5D4 (1492)** @ 0x08010000, string `VNSD-0` @ 0x08010004, version string `VNSD-0 v1492` @ 0x0805af6d |
| Load | Base 0x08004800 (18-KB bootloader below, not part of image); reset vector → `SystemInit` (sets `SCB->VTOR = 0x08004800`) → `__main` → `__scatterload`; image end 0x08062800 (0xFF padding after) |
| Scatter table | `Region$$Table$$Base` @ 0x08060ec8 (3 × 16 B): **.data** source 0x080613d8 (2488 B compressed) → destination 0x20000000, 12 028 B (0x2EFC) via `__decompress` @ 0x08005fbe; **XMC (.xmc_bss)** source 0x08061d90 (2656 B compressed) → destination 0x60000000, 148 996 B (0x24404), same `__decompress`; **.bss** zero-init from 0x20002EFC, 117 468 B (0x1C8DC) via routine @ 0x08035c08 |
| Release note | "Fixed incorrect data display due to missing HTTP uploads" (changelog, issue #17) |

## 1a. MCU identification: Nations N32G45x, not Artery AT32F403A

- The Control/EMS MCU is a **Nations N32G45x with XFMC** (external memory controller), not the Artery AT32F403A assumed in earlier document versions.
- Evidence: QSPI controller at 0xA0001000 with N32 QSPI_STS bits; RCC PLL multiplier in bits 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG in use; SAC crypto coprocessor at 0x40024000 (formerly misnamed "RNGX"); 96-bit UID @ 0x1FFFF7F0; 102 vectors.
- External SRAM 0x60000000–0x60026AF7 runs through XFMC (see scatter table above, `.xmc_bss`), control registers at 0xA0000000: the official Nations device header `n32g45x.h` (local only, not in the repo) defines `XFMC_REG_BASE 0xA0000000` and `QSPI_BASE = XFMC_REG_BASE + 0x1000`; the Nations driver library contains `n32g45x_xfmc.c`; the firmware itself has the shell command `xfmc_test` (the current English N32G45x user manual incorrectly marks this address space "Reserved").
- Details/header copy: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Preparation Status in Ghidra (`/VNSD/Control/Control_1492_VNSD-0_app_1492_0702_142136.bin`)

All verification steps passed (`marstek_gates.py` (local only, not in the repo), 2026-09-30):

| Check | Result |
|---|---|
| Undefined flash bytes / instructions without function | 0 / 0 |
| Code as data (jump targets, function pointers including table entries, literal "functions") | 0 |
| Functions | **1935**, all named, no `FUN_`/`tmp_`/duplicates |
| Prototypes | all set (no default signature) |
| Referenced RAM/XMC globals without names | 0 |
| Task names vs `g_taskDefTable`, shell handlers vs `shellCommandTable` | match |
| Duplicate global labels | none |

## 3. FreeRTOS Tasks (Table `g_taskDefTable` @ 0x08059958, `TaskDef_t[14]`, 24 B/entry)

| Task | Priority | Stack (words) | Function |
|---|---|---|---|
| Task_ch395 | 15 | 256 | 0x08032305 |
| Task_ch_mqtts | 14 | 1024 | 0x08032325 |
| Task_parser | 13 | 512 | 0x080323b9 |
| Task_ch_http | 13 | 512 | 0x08032317 |
| Task_fc41d | 12 | 512 | 0x08032375 |
| Task_http | 11 | 512 | 0x0803238d |
| Task_udp | 10 | 256 | 0x0803240d |
| Task_ems | 9 | 256 | 0x08032361 |
| Task_ct | 8 | 512 | 0x08032347 |
| Task_can_comm | 7 | 256 | 0x080322e9 |
| Task_Modbus_tcp | 7 | 512 | 0x080322c9 |
| Task_can_receive | 6 | 256 | 0x080322f7 |
| Task_rs485 | 3 | 512 | 0x080323cd |
| Task_iwdg | 1 | 128 | 0x080323a5 |

14 entries, all with handle in `TaskDef_t.pHandle` (0x20000000–0x20000034); creation in
`Task_CreateTasksAndStartScheduler` @ 0x0801dc7c (loop over `g_taskDefTable[0..13]`,
`FreeRTOS_xTaskCreate`, then `vTaskStartScheduler`). Values decoded directly from binary (no
`Task_fc41d` special case as in v151 docs — here complete regular table entry 0).

## 4. Interfaces

### 4.1 Modbus (TCP 502 over CH395, RTU over RS485)

- Descriptor table `g_modbusRegDescTable` @ **0x20000354** (identical to v150, differs from v151's
  0x2000034C), `ModbusRegDesc_t[246]`, registers 30000–38014 — unchanged since v149 (the 15 registers
  38000–38014 from v149 are included).
- `Write_Handler` @ 0x08055720 (registers ≥ 40000 and special addresses 0xA0xx–0xB7xx):
  - **Register 46000** (internally `iVar8 == 1000`, 45000+1000): value **0x5100** → `g_debugMode = 1`,
    `EEPROM_Write(0x900, …)` — debug shell permanently enabled (§8).
  - YMODEM target selection at same register: `0x04D2` → target 1 (EMS), `0x0929` → target 4 (VENUS),
    `0x0D80` → target 3 (MPPT), `0x11D7` → target 2 (BMS) — four instead of three values documented in v151.
  - Block **46500–46544** (0xB5A4–0xB5D0, 15 × 3 registers, `Inverter_Set_Schedule_Reg`,
    EEPROM 0x36BF+8·n) — present since v149.
  - Remote charge/discharge power (registers 45900 / 45901 / 0xA424 / 0x19B block): all clamped
    `< 0x9C5` (≤ 2500 W) — consistent with §5.
  - Register **47400** (`iVar8 == 0x960`): unchanged test branch with two Chinese
    debug `printf`s — already present in v149.2, not from v151 (§11).
- No register 0xB799 (47001) and no `CAN_Write_Category0x18` — both first appear in v151.

### 4.2 Local API / MQTT-RPC (`g_mqttRpcMethodTable` @ 0x0805cb1c, 16 methods, dispatched in
`MQTT_JSON_RPC_Dispatcher` @ 0x0801eda4)

Marstek.GetDevice (1), Wifi.GetStatus (2), BLE.GetStatus (3), Bat.GetStatus (4), PV.GetStatus (5),
ES.GetStatus (6), ES.SetMode (7), ES.GetMode (8), EM.GetStatus (9), Marstek.GetData (10), DOD.SET (11),
Ble.Adv (13), Led.Ctrl (14), Ota.Set (15), Set.Ver (16), Reset.Factory (17). Identical to v149 (the
5 new methods in v149 are present, no further change). `Set.Ver` (case 0x10) accepts exactly
800 W (→ `g_workMode=1`), 0x898=2200 W (→ `g_workMode=3`), or 0x9C4=2500 W (→ `g_workMode=0`).

### 4.3 Debug Shell (letter-shell, `shellCommandTable` @ 0x08060ef8, 78 entries × 16 B)

**65 commands** + 1 user record (`VNSD-0 v1492`, password field empty) + 10 key bindings
(↑/↓/→/←, Tab, 2× backspace, delete, 2× enter) + `help`/`clear`. Identical to v149 (63 + `get_backup_datas`,
`power_check`); no change vs v149, none of v151's new features (`read_inv1`, `read_inv2`).

Command list (decoded): set_level, debug_fc, test, check_periph, xfmc_test, get_ver, set_old, get_old,
change_soc, reset, flash_test, eeprom_flash_test, ch395_get_ip, ch395q_test, rtc_show, rtos_status,
update_ems, ctl_mppt_power, update_vns, update_bms, update_mppt, Resume_task, restore_factory, debug_mode,
venus_poweroff, venus_rfd, get_mppt_data, get_venus_data, get_main_bms_data, get_all_bms_data,
delate_ssl_cert, check_ssl_cert, set_wifi, mac_ble, ver_fc, fc41d_reset, trace_debug, modbus, show_modbus,
get_log, wifi_info, api_set, api_port, get_time, vns_ate_test, ctrl_led_light, energy_show, err_code,
ext_info, reset_reason, button_ctrl, led_continue_ctrl, recon_ch395, hard_reset_395, ch395_ver, key_state,
code_test, lck_bms, ems_rtrl_info, full_dischrg, free_mqtts, get_meter_mac, get_backup_datas, power_check,
get_ota_log.

**Accessibility:** via RS485 debug console and CH395 TCP server port **8091** (0x1F9B,
`CH395_Init_TCPServer_Socket` @ 0x0803740c, socket descriptor `g_ch395SockTcpServer8091` in socket
table @ 0x2001adcc), activated via Modbus 46000 = 0x5100 (§8).

### 4.4 Cloud

`hamedata.com` endpoints (unchanged since v147, no `marstekcloud.com` — that comes with v151):

| Path | Schema | Function |
|---|---|---|
| `/app/neng/getDateInfo%s.php?uid=…` | HTTP | `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` |
| `/prod/api/v1/setVenusDReporting?v=%s` | HTTP | `Cloud_UploadTelemetryData` (main telemetry upload) |
| `/ems/api/v1/getDevicesApply?devid=%s` | HTTP | WLAN registration path |
| `/external-services/api/v1/devices/backup-configs?devid=%s&latest=true` | HTTP | Cloud backup config (since v149) |
| `/external-services/api/v1/schedulings/policies?devid=%s` | HTTPS | Schedule/policy fetch |
| `www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl` | HTTP | FC41D module OTA (`AT+QWLANOTA=`) |

Region table (index from EEPROM field, selected in `%s.hamedata.com`): only 3 entries
`eu, www, us` — no `dev`/`staging` (that comes with v151; region index there expanded to 0–4).
Local data-dependent endpoints `http://%s/api/v1/data`, `/v1/data`, `/v1/json` (IP from `g_bleIpAddrBuf`)
are **not** cloud paths but local-API queries of external meters/inverters in local network.

MQTT topics: `marstek_energy/%s/device/%s/ctrl`, `marstek_energy/%s/App/%s/ctrl`. TLS via mbedTLS
(authmode 2 for MQTT, see §8).

### 4.5 BLE (Quectel FC41D, AT Commands)

Device name `MST_VNSD_` + 4 characters (`AT+QBLENAME=MST_VNSD_%c%c%c%c`). Command dispatcher
`BLE_App_Command_Dispatcher` @ 0x0800c758 — command set identical to v149 (0x5A/0x5B from v149 present),
per diff document only response/telemetry structure changed (~145 op differences to v149, dispatcher-wide).
**BLE-OTA:** `BLE_Cmd_OTA_Init` (0x0803306c), `BLE_Cmd_OTA_WriteSetup` (0x08033100),
`BLE_Cmd_OTA_Validate` (0x08032f8c) present and per diff document byte-identical to v149/v150 —
BLE-OTA works. `Base64URL_Encode` (URL-safe base64 for AES-ECB-encrypted telemetry body) is still present
in v149.2 (removed starting v150).

## 5. Power Limits

- `Config_ClampDefaults_ParamBlock` @ 0x080093f0: charge **and** discharge power clamped to **0x9C4 = 2500 W**
  (invalid/0 → default; discharge additionally to 800 W with `g_workMode = 1` if invalid).
  Charge limit `g_chargeCutoffSoc` 1–100 % (default 100), discharge limit `g_dischargeCutoffSocRaw` 12–70 %
  (default 12). Identical to v149 (no change).
- `Grid_Export_Power_Limiter` @ 0x08023438: start value `min(P_net, 2500 W)`, then PV excess path,
  overall clamp **4000 W**. Byte-identical to v149 and v150 per diff documents — the v151 change
  (start value without 2500 W clamp) does **not** affect v149.2.
- `Set.Ver` (MQTT-RPC 0x10, §4.2): 800 / 2200 (0x898) / 2500 W (0x9C4), identical to v149.
- Remote control registers (Modbus 45900/45901/0xA424/0x19B, §4.1): all clamped `< 0x9C5` (≤ 2500 W).
- **Not yet in v149.2:** the separate, configurable peak shaving clamp (EEPROM 0x394,
  BLE command 0x29, MQTT commands 0x3f/0x41) arrives first with v150 and does not replace the 2500 W
  hard limit but acts additionally (diff v1492→v150).

## 6. Structure/Data Findings

- mbedTLS cipher tables (`aes_info`, `aes_128/192/256_ecb_info`, `gcm_aes_info`,
  `aes_128/192/256_gcm_info`, `mbedtls_cipher_definitions[7]`, `ecp_supported_curves[2]` — only secp256r1)
  lie immediately after `g_mqttRpcMethodTable` @ 0x0805cb9c ff., as in other versions.
- `MQTT_Transport_NullCallback` @ **0x0800fffe** (`bx lr` stub) is the wait-ready hook of
  FC41D transport, not dead code (address version-specific, in v147 0x0800FFFC).
- `CH395_ConfigTcpKeepalive_Defaults` @ 0x08036e34 (function sets TCP keepalive parameters).
- **BMS pack record (v149.2):** base `g_bmsPack0Voltage` @ **0x20014fa8**, step **0x60 = 96 B**
  per pack (8 B more than v149's 88 B — pair functions `CAN_AA_StoreUnitFieldPair0…88` cover offsets
  0, 8, 16, 24, 64, 72, 80, 88 within 96-B record). CAN battery count still limited to 8,
  Modbus blocks 34000…34600 (7 packs). `BMS_CellVoltage_MinMax_PerPack` @ 0x08017860 still runs only
  over **`pack < 6`** (16 cells per pack, `g_bmsPack0CellVoltageArray` @ 0x20014fc8) — the 7th pack
  (index 6) missing from this statistic and MQTT telemetry built from it, but present via Modbus. (Address
  of this function and records is version-specific — here for v149.2 confirmed directly from export,
  **not** the addresses named in earlier docs for other versions.)
- **Pack numbering/MQTT path offset:** as in 147–151, `CAN_AA_StoreUnit*` writes to
  record `unit − 1`; Modbus 34000+100·n and `ShellCmd_get_all_bms_data` read record n consistently;
  `MQTT_Publish_BMS_Data` for index p ≥ 1 reads record p instead (off-by-one) — code in v149.2 same
  structure as other versions (not separately disassembled, incorporated for consistency with
  pair functions and known structure; see "Known facts" of task).
- **FC03 buffer overflow** (in v151 docs verified only for v150/v151) is **also present in v149.2**:
  `FC03_Read_Handler` @ 0x080232a4 unconditionally copies response after construction
  (`for (uVar4 = 9; uVar4 <= uVar5; …) g_modbusLteTxFrame[uVar4] = (&g_modbusTcpTxFrame)[uVar4];`) to
  mirror buffer `g_modbusLteTxFrame`, regardless of requested register count N ≤ 125 — the
  error is not structurally new in v150 but already present here (§11).
- Region string table contains only `eu`, `www`, `us` (3 entries) — no `dev`/`staging` (first in v151).

## 7. Only in v149.2 / Changed vs Predecessor (v149) and Successor (v150)

Summary (evidence and instruction ratios: `VNSD_Control_Diff_v149_v1492.md` /
`VNSD_Control_Diff_v1492_v150.md`):

- **BMS pack record 88 → 96 B** (8 B more per pack) — only structural change with downstream
  signal for code (`BMS_CellVoltage_MinMax_PerPack`, `MQTT_Publish_BMS_Version`, `get_all_bms_data`).
- **French self-consumption controller (SMR/TIC, Linky)** greatly simplified: 408 → 247 instructions in
  `SMR_TIC_SelfUse_PowerController`, multiple `VectorSignedToFloat`/scaling calls disappear,
  setpoint calculation only once instead of three times.
- **Cloud/MQTT telemetry rebuilt** (v149 → v149.2, then through v150 unchanged): `MQTT_Publish_BMS_Data`,
  `Cloud_UploadTelemetryData`, `DeviceInfo_BuildStatusString`, `Cloud_HTTP_Response_Parser`,
  `MQTT_StatusReport_BuildAndPublish`, `MQTT_Telemetry_String_Formatter`, `HTTP_Task_StateMachine_Handler`,
  `BLE_App_Command_Dispatcher` (response/field adjustments only), `MQTT_CtrlCommand_Dispatcher` — fitting
  the only changelog point "Fixed incorrect data display due to missing HTTP uploads".
- **Compared to successor v150, v149.2 is not "practically complete":** **56 changed
  functions (26 content), 20 new and 1 removed** follow next (diff `VNSD_Control_Diff_v1492_v150.md`). Key
  points: cloud telemetry upload completely rebuilt to new HTTPS/mbedTLS client, JSON-POST to
  `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` replaces AES-ECB+Base64URL-GET request to
  `hamedata.com` (thereby fixes real buffer overflow in `Base64URL_Encode`, which was called in v149.2 with
  alleged target size 2400 B but actual buffer `g_cloudUploadBodyBuf` only 1200 B); new
  peak shaving/power clamp (EEPROM 0x394, BLE command 0x29, MQTT commands 0x3f/0x41, new function
  `MQTT_Publish_Grid_Status`); two new watchdogs for hanging uploads (CH395 chip reset,
  modem hardware reset); `DeviceInfo_BuildStatusString` gets fixed hardware tags for 6 additional
  meter types; `Grid_PowerSync_StateMachine` expands meter type range for direct takeover;
  BMS self-test/MOS flag additionally reported; new shell command `ShellCmd_FreeRtos_mem`.
  **Correction vs earlier analysis version:** the previously named three only differences
  "HTTP upload read offset +5→+7", "CH395 socket parameter 0x24→0xAA" and "HTTP GET
  buffer 0x168→0x280 B" were pure `__LINE__` log arguments without behavior change; the actual change to
  `HTTP_ParseUploadResponseCode` is a different JSON key (`"code":` instead of `"de":`).
- **Unchanged vs v150:** Modbus register contents, the 2500 W hard limit, OTA/BLE-OTA and the
  `hamedata.com` endpoints themselves. **Not unchanged:** BLE command set grows by command 0x29
  (peak shaving, see above) — the "numerically same" mentioned in §0a applies only to step v149→v149.2,
  not v149.2→v150.
- All differences from two-versions-prior predecessor v147 (2200→2500 W, 11→16 RPC methods, 15 new
  Modbus registers, shell +2 commands, TIC support, DOD range check etc.) are already in v149
  and persist in v149.2 — details: `VNSD_Control_Diff_v147_v149.md`.

**Observation on v151 docs (now resolved):** The v151 doc listed ""Get err code info" now answers
(Cmd 0x13, 280 B)" as BLE change from v150 to v151. In v149.2, `BLE_App_Command_Dispatcher`,
case `0x13`, already responds unchanged with `BLE_Send_Response('\x13', &g_displayErrorLog, 0x118)`
(280 B) — source identical to that in `Dekompilate/VNSD_Control_v151/0800c730_BLE_App_Command_Dispatcher.c`.
The revised diff `VNSD_Control_Diff_v150_v151.md` now explicitly confirms: "BLE "Get err
code info" (0x13): response in v150 and v151 identical (no difference)". The earlier classification as
new-in-v151 was thus incorrect (no longer open).

## 8. Security Findings (identical with 147–151, unless otherwise stated)

1. **Unauthenticated remote console:** Modbus-TCP write access (port 502, no auth) to register
   46000 = 0x5100 → `g_debugMode = 1` permanently in EEPROM (0x900) → letter-shell on RS485 **and**
   TCP 8091 (`CH395_Init_TCPServer_Socket`, socket 1 @ 0x2001adcc). Allows e.g. factory reset, set WLAN
   credentials, start firmware update, restart.
2. **Built-in WLAN credentials** `SM471` / `Hame147258` (strings @ 0x0802d578 / 0x0802d580,
   `Config_LoadOrInit_DeviceSettings` @ 0x0802d174, factory default on empty EEPROM 0x400).
3. **Cloud upload over unencrypted HTTP** (`http://%s.hamedata.com/prod/api/v1/setVenusDReporting…`
   etc.); payload AES-ECB + Base64URL (`Base64URL_Encode`, removed starting v150). Policy fetch
   (`schedulings/policies`) runs over HTTPS, all other `hamedata.com` paths over cleartext HTTP.
4. MQTT-TLS with `authmode 2` (certificate check active) — not separately reverified for v149.2, but
   no change to TLS config found in diff documents.
5. Two Chinese cleartext debug `printf`s in `Write_Handler` test branch for register 47400 (§4.1, §11) —
   not a security risk in the strict sense, but debug residue in production build.

## 9. Open Items

- MQTT-TLS certificate check (`authmode`), expiration check and the encrypted
  TLS certificates/keys were not separately disassembled for v149.2 but only incorporated as
  ("unchanged") areas not listed in diff documents — no instruction verification.
- `MQTT_Publish_BMS_Data` off-by-one and `g_pendingBootDescriptor` overflow for pack index 8: not
  confirmed for v149.2 by own look at function decompilation but incorporated from version-spanning
  known structure (identical pair functions, identical BMS code path).
- No firm statement on exact scope "Fixed incorrect data display due to missing HTTP uploads"
  (changelog) — the diff table lists many rebuilt telemetry/HTTP functions but no single
  unambiguous bugfix-commit character like other changelog points.

## 10. Corrections on Already "Complete" Versions (found during v147 work, affect also v149.2)

- v149.2: **3 code locations** incorrectly typed as data (v149: 33), **2 shell handlers** as
  byte fields (v149: 3); a task trampoline problem like in v149 (task names shifted) was also
  fixed.
- All versions (147–151), thus also apply to v149.2:
  - letter-shell `CLI_Backspace`/`CLI_DeleteCharForward` were incorrectly named swapped, `CLI_DeleteChar`
    direction misdescribed.
  - mbedTLS cipher range incorrectly marked as `mbedtls_eckey_info`.
  - Task table in 149/149.2 incorrectly named as `ciphersuite_preference` (now `g_taskDefTable`).
  - `CH395_ConfigUDP_Defaults` now named `CH395_ConfigTcpKeepalive_Defaults` (sets TCP keepalive,
    here @ 0x08036e34).
  - The `bx lr` stub now named `MQTT_Transport_NullCallback` (here @ 0x0800fffe, version-specific
    address).

## 11. Chinese Strings

115 unique UTF-8 strings with Chinese text in export, all translated to English via `ZH->EN` comment
(predominantly shell command descriptions and debug messages). Selection with German translation:

| Chinese (abbreviated) | Context | German |
|---|---|---|
| 测试命令 | Shell `test` | Test command |
| 外设启动查询 | Shell `check_periph` | Peripheral startup check |
| 外部sram测试 | Shell `xfmc_test` | External SRAM test |
| 获取所有版本信息 | Shell `get_ver` | Get all version information |
| 设置老化版本 | Shell `set_old` | Set burn-in version |
| 修改soc | Shell `change_soc` | Change SOC |
| 重启系统 | Shell `reset` | Restart system |
| flash分页写入测试 | Shell `flash_test` | Flash page write test |
| RTOS状态 | Shell `rtos_status` | RTOS status |
| 唤醒被挂起的所有任务 | Shell `Resume_task` | Wake all suspended tasks |
| 开关逆变器调试功能 | Shell `debug_mode` | Toggle inverter debug function |
| 休眠设备 0:正常工作 1:休眠 | Shell `venus_poweroff` | Sleep device (0: normal, 1: sleep) |
| 删除SSL证书 | Shell `delate_ssl_cert` | Delete SSL certificate |
| 设置WIFI信息 | Shell `set_wifi` | Set WLAN credentials |
| 查询FC41D模组固件版本 | Shell `ver_fc` | Query FC41D module firmware version |
| 进入modbus | Shell `modbus` | Enter modbus mode |
| 显示modbus地址 | Shell `show_modbus` | Show modbus address |
| Local API 设置：0 关闭 1 开启 其他 查询 | Shell `api_set` | Local API set: 0 off, 1 on, else query |
| 控制led灯亮度 0 最亮 7-1依次减弱 | Shell `ctrl_led_light` | Control LED brightness (0 brightest, 7→1 dimmer) |
| 余电上网功能: 0关闭 1开启 | Shell `full_dischrg` | Excess feed-in: 0 off, 1 on |

**Special case:** Two Chinese character strings exist not only as comment but **directly in compiled
`printf` call** of Modbus `Write_Handler` test branch for register 47400 (§4.1):
"47400要写进去里面的数据 = 0x%x" (en.: "Data to write into 47400 = 0x%x") and "没到里面吗123456"
(en.: "Didn't it get through? 123456" — obvious debug residue, not user error message).
