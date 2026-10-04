# Control FW v151 — Full Analysis and Diff Against v150

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md](VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md)

**Status:** completed (as of 2026-09-30). All statements have been verified against the code; exceptions are marked.
**Binary:** `151_control_VNSD-0_ems_app_0151_0929_094333.bin` (archive `Marstek FW Archiv/firmwares/VNSD-0/Control/151/`)
**Reference:** `150_control_VNSD-0_app_0150_0805_115146.bin` (v150, newly imported with correct base)
**Ghidra:** `/VNSD/Control/Control_151_VNSD-0_ems_app_0151_0929_094333.bin` and `/VNSD/Control/Control_150_VNSD-0_app_0150_0805_115146.bin` (project `Marstek_Venus`)
**Decompilation:** `Dekompilate/VNSD_Control_v151/` (local only, not in the repo) (1956 functions) + `listing/` (flash listing, RAM globals, peripheral registers)
**German version:** `VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md`
**Diff table:** `VNSD_Control_Diff_v150_v151.md` (EN) / `VNSD_Control_Diff_v150_v151.md` (DE)

---

## 0. Most important finding up front: the image base is 0x08004800

All Control firmwares of the Venus D (147, 149, 149.2, 150, 151) are **linked for 0x08004800**, not for
0x08000000. Ahead of that sits an 18 KB bootloader (`0x08000000`–`0x080047FF`).

| Evidence | Finding |
|---|---|
| `SystemInit` (0x080333E0) | writes `SCB->VTOR = 0x08004800` |
| Vector table at 0x08004800 | Reset = Keil pattern `ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`; fault handler `nop; b .`; SVC/PendSV = FreeRTOS port; default = `b .` |
| Pointer literals (4-byte aligned) | at 0x08004800: 405 hits on string starts, 178 on function prologues; at 0x08000000: 70 and 9 respectively. Same ratio for 147/149/149.2/150 |

**Consequence for all older analyses (v147–v150):** every flash address in the older documents is too low by 0x4800
(true address = cited address + 0x4800), and function names derived from incorrectly resolved strings were
sometimes wrong. v150 was therefore re-imported and completely reprocessed (separate document).

## 0a. Checklist for the update decision (v150 → v151)

| # | Question | Answer v151 | Evidence |
|---|---|---|---|
| 1 | Modbus registers changed? | **Register set identical** (246 entries, 30000–38014). Only change in the read range: **32102** `float32` → `i16`. Consequence: reading 32103 alone → exception 02 instead of UsageFault; block reads spanning 32103 are rejected. Write range: **new holding register 0xB799 (47001)** and a new CAN call (`CAN_Write_Category0x18`) in the AI-mode path. The 32101 sign error, the MBAP length error, and the 44002/44003 shadow value persist (§9). | §3, §9 |
| 2 | BLE changed? | Device name (`MST_VNSD_` + 4 characters) and command set unchanged. Detail changes: EPS on/off (0x0F) and discharge limit (0x54) no longer reset the EPS reserve hysteresis flags; **LED status is stored in EEPROM (0x395)**. Still broken: Cmd 0x27 (power history) does not respond at all when ≥ 15 values are requested (§11). | §8 |
| 3 | AC feed-in limit 2500 W changed? | **The configurable 2500 W limits are not** (Modbus write, BLE, MQTT, configuration check, remote control command: all unchanged). **What changed is the PV upper bound when feeding in**, in `Grid_Export_Power_Limiter`: v150 started with `min(P_bms, 2500 W)`, v151 with `grid measurement − P_neg` without the 2500 W clamp; the overall 4000 W clamp and the surplus path are unchanged. | §8 |
| 4 | Firmware update via BLE still possible? | **Yes.** BLE-OTA cases, flash/OTA/CAN update/QSPI functions, and the update check (CRC, device mask, project type) are functionally identical to v150. New is a **type check in the YMODEM path** (shell `update_*` over RS485/TCP 8091): the file name must match the target ("Bin type mismatch! Upgrade aborted."). Of the bootloader area, the app only writes the **update descriptor page 0x08004000** (40 bytes, "VNSD-0" magic, CRC16) and reads the bootloader version word at 0x08001000; bootloader code is not touched. | §8 |
| 5 | Cloud domains changed? | **Yes, one endpoint:** `getDateInfo` moves from `http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php` to **`http://api-<region>.marstekcloud.com/app/neng/getDateInfo<suffix>.php`** (plain HTTP, port 80). Upload unchanged: `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`. Still on hamedata: `backup-configs` (HTTP), `schedulings/policies` (HTTPS), `getDevicesApply` (HTTP, WLAN path), FC41D OTA `http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl`. The MQTT host comes from the configuration (flash block 0x317000). Region table: indices 3/4 are now named `dev`/`staging` instead of `eu-dev`/`eu-staging`. Device source ID in the cloud payload: `VenusD-<SN>` → `VNSD-0-<SN>`. | §7, §8 |

Region = index from EEPROM 0x441 into the table `eu, www, us, dev, staging` (invalid → `eu`); on upload, `www` is redirected to `us`.

## 1. Binary fingerprint

| | v151 |
|---|---|
| Size | 393 216 B (0x60000) |
| SHA-256 | `db3638cb68683175debe283375f36b02427b9e5e7a647ce2045e8135177fea47` |
| Link address | `0x08004800`–`0x080647FF` |
| MCU | Nations **N32G45x** with XFMC (Cortex-M4F), previously misidentified as Artery AT32F403A/407 (see §1a); SRAM **144 KB**; `SystemCoreClock` = 144 MHz (init value) |
| Toolchain | **Keil ARMCC** (`__main`/`__scatterload`, LZ77-compressed RW) |
| Libraries | FreeRTOS (heap_4, 80 KB), mbedTLS 2.28.10 (TLS-ECDHE-RSA-AES128-GCM-SHA256 / secp256r1 only), cJSON (vendor-modified), llhttp, letter-shell, N32 standard peripheral library (n32_stdperiph, previously listed as "AT32 firmware library"), WCH CH395 library |
| Initial SP | `0x20022010` (= end of .bss) |
| Version string | `VNSD-0 v151` (0x0805C9B9); version constant 0x97 at 0x08010000 |
| Image footer | `VNSD` + 12 zero bytes at 0x080647E0, preceded by 0xFF padding |

## 1a. MCU identification: Nations N32G45x, not Artery AT32F403A

- The Control/EMS MCU is a **Nations N32G45x with XFMC** (external memory controller), not the previously assumed Artery AT32F403A.
- Evidence: QSPI controller at 0xA0001000 with N32 QSPI_STS bits; RCC PLL multiplier in bits 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG in use; SAC crypto coprocessor at 0x40024000 (formerly misnamed "RNGX"); 96-bit UID @ 0x1FFFF7F0; 102 vectors.
- External SRAM 0x60000000–0x60026AF7 runs through XFMC, control registers at 0xA0000000: the official Nations device header `n32g45x.h` (local only, not in the repo) defines `XFMC_REG_BASE 0xA0000000` and `QSPI_BASE = XFMC_REG_BASE + 0x1000`; the Nations driver library contains `n32g45x_xfmc.c`; the firmware itself has the shell command `xfmc_test` (the current English N32G45x user manual incorrectly marks this address space "Reserved").
- Details/header copy: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

## 2. Memory map (created in Ghidra)

| Block | Range | Contents |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | not part of the image; 0x08001000 = bootloader version word, 0x08004000 = update descriptor page |
| flash | 0x08004800–0x080647FF | application |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | device signature (UID) |
| .data | 0x20000000–0x20002EFF | RW initial values, LZ77 decompressed |
| .bss | 0x20002F00–0x2002200F | zero-initialized; `ucHeap` 0x20003144 (0x14000 B) |
| .xmc_bss | 0x60000000–0x60026AF7 | external SRAM via XFMC (telemetry ring buffer, HTTP/MQTT/TLS buffers, event logs); scatter-load section name `.xmc_bss` kept unchanged |
| apb1/apb2/ahb/ppb | peripherals | registers named per the official Nations SVD (N32G457, 942 entries), plus 16 XFMC registers from `n32g45x.h` and 5 SAC registers (not in the SVD) + Cortex-M4 core registers; replaces the earlier, incorrect mapping to the Artery SVD `AT32F403Axx_v2.svd` |
| extflash_ctrl | 0xA0001000–0xA00011FF | **QSPI controller** (`QSPI_BASE` = `XFMC_REG_BASE` + 0x1000), previously listed as an undocumented register block (see §1a, §13) |

Scatter-load table `Region$$Table$$Base` @ 0x08062C94: .data (0x080631D4 → 0x20000000, 0x2F00, `__decompress`),
XMC (0x08063B8C → 0x60000000, 0x26AF8), .bss (zero-init 0x1F110). Decompressor `__decompress` @ 0x08005FBE:
`op=*src++; n=op&7 (0→*src++); L=op>>4 (0→*src++); n−1 literals; op&8 ? back-reference d=*src++, L+2 bytes : L zero bytes`.

## 3. Modbus

### 3.1 Read range (descriptor table)

`g_modbusRegDescTable` @ **0x2000034C** (v150: 0x20000354), type `ModbusRegDesc_t[246]`
(`reg_addr`, `pad`, `source`, `type_code`, `type_code2`, `scale`, `count`), registers 30000–38014.
The variables behind the registers are named `g_mbReg<no>` in Ghidra, wherever no domain-specific name has been assigned.

- No registers added or removed; the only semantic change is **32102** `float32` → `i16`.
- `FC03_Read_Handler` and `Read_Serializer` are instruction-identical to v150.

| | v150 | v151 |
|---|---|---|
| Reading 32103 alone | UsageFault (unaligned VLDR), session dies | exception 02 |
| Block spanning 32103 (workaround 32102–32104) | works | **rejected (exception 02)** |
| 32101 negative (Read-Serializer sign error) | wrong | **still wrong** |

### 3.2 Write range (`Write_Handler` @ 0x080570E0)

- New register **0xB799 (47001)** and new call `CAN_Write_Category0x18()` in the AI/grid-filter path (changelog 4).
- The internal base value of the block starting at 45000 shifts by 1 (v150 `reg−45000`, v151 `reg−45001`), because
  0xAFE6 was inserted as its own case; the externally visible register numbers stay the same.
- **Register 46000** (unchanged since v150): value **0x5100** permanently turns on debug mode
  (EEPROM 0x900) → debug shell on RS485 **and TCP 8091** (§10.1). Values 0x04D2/0x0929/0x0D80 select the YMODEM update target.

### 3.3 Modbus TCP (CH395)

Socket table `g_ch395Sock…` @ 0x2001D604 (8 × 36 B): socket 0 = **Modbus TCP 502**, socket 1 = **TCP server 8091
(debug shell)**, socket 2 = UDP broadcast 8090, socket 4 = UDP for meter/heartbeat. Keepalive (WCH functions
`CH395KeepLiveCNT/IDLE/INTVL`): CNT 5, **IDLE 10,000 → 60,000 ms**, INTVL 3000 ms; in v151 it is additionally
set **before opening the Modbus socket** (changelog 1).

## 4. Debug shell (letter-shell)

Command table `shellCommandTable` @ 0x08062CC4 (81 × 16 B): **70 commands** (+ `help`, `clear`), 1 user
(`VNSD-0 v151`, **empty password**), 10 key bindings. New compared to v150: `read_inv1`, `read_inv2`.
All command functions are named `ShellCmd_<command>`.

**Reachability:** `Task_rs485` operates depending on `g_debugMode` (EEPROM 0x900):
0 = Modbus RTU on RS485; 1 = **shell on RS485 and via TCP port 8091**; 2 = waiting for the string
`quit_debug_5100_` (RS485 or TCP 8091), then mode 1. Mode 1, among other things, sets Modbus register 46000 = 0x5100 (§10.1).
The shell command `modbus` switches back to 0.

Command list (translated): set_level, debug_fc, test, check_periph, xfmc_test, get_ver, set_old/get_old, change_soc,
reset/reset_reason, flash_test/eeprom_flash_test, ch395_get_ip/ch395q_test/ch395_ver/recon_ch395/hard_reset_395,
rtc_show/get_time, rtos_status/FreeRtos_mem/Resume_task, update_ems/vns/bms/mppt, ctl_mppt_power, restore_factory
(1 = everything, 2 = keep energy data, 3 = certificates), venus_rfd, debug_mode, venus_poweroff, get_mppt_data/get_venus_data/
get_main_bms_data/get_all_bms_data, delate_ssl_cert/check_ssl_cert, set_wifi/wifi_info/mac_ble/ver_fc/fc41d_reset,
trace_debug, modbus/show_modbus, get_log, api_set/api_port, vns_ate_test, ctrl_led_light/led_continue_ctrl
(actually calls `Inverter_Power_Setpoint_Apply`), energy_show/err_code/ext_info, button_ctrl/key_state,
read_inv1/read_inv2, code_test, lck_bms, ems_rtrl_info, full_dischrg, free_mqtts, get_meter_mac, get_backup_datas,
power_check, get_ota_log.

## 5. FreeRTOS tasks

Table `g_taskTable` @ 0x0805B358: **14 application tasks** (earlier versions counted 13;
`Task_fc41d` @ 0x08033540 had been overlooked).

| Task | Priority | Stack (words) |
|---|---|---|
| Task_ch395 | 15 | 256 |
| Task_ch_mqtts | 14 | 1024 |
| Task_parser | 13 | 512 |
| Task_ch_http | 13 | 1024 |
| Task_http | 11 | 512 |
| Task_udp | 10 | 256 |
| Task_ems | 9 | 256 |
| Task_ct | 8 | 512 |
| Task_Modbus_tcp | 7 | 512 |
| Task_can_comm | 7 | 256 |
| Task_can_receive | 6 | 256 |
| Task_rs485 | 3 | 512 |
| Task_iwdg | 1 | 128 |
| Task_fc41d | (entry 0) | – |

Priorities and stacks are unchanged from v150 to v151. `Task_Modbus_tcp` (priority 7) continues to run below the TLS tasks (13/14).

## 6. Chinese strings

121 UTF-8 strings with Chinese text, all translated (Ghidra: EOL comment `ZH->EN`). Two Chinese
debug `printf` calls remain in the Modbus `Write_Handler` (register 47400, test branch).

## 7. Changelog v151 → code

| # | Changelog (vendor) | Implementation in code |
|---|---|---|
| 1 | Modbus TCP stability | CH395 keepalive IDLE 10 → 60 s, set before opening the Modbus socket (`CH395_ConfigTcpKeepalive_Defaults` from `CH395_Init_Modbus_TCP_Socket`); CH395 hang watchdog in `HTTP_ClientTask_Dispatcher`: backlog `>1`→`≥5`, 1800 → 14,400 ticks, soft chip reset → **full MCU restart**, new suppression flag |
| 2 | DOD fault state, no discharge possible | `CT_GridPower_Controller`: a stuck positive CT setpoint is set to 0 when a new flag (0x200002B2) is set; `Battery_EPS_Reserve_SOC_Hysteresis` now only runs when SoC ≠ 0 and no longer resets the EEPROM EPS flag itself; new status field `df=` in the MQTT status report. *Which of these items fixes the field issue cannot be determined from the code alone.* |
| 3 | YMODEM upgrade type check | `YModem_ReceiveFile` classifies the file name (`OTA_ClassifyDeviceTypeTag`); `YModem_FirmwareUpgrade_Handler` aborts on −5 with "Bin type mismatch! Upgrade aborted." |
| 4 | AI mode check + query current AI strategy | `AI_ScheduleQuery_Response_Send` now sends page by page instead of one unbounded `snprintf`; `Write_Handler`: new register 0xB799 + `CAN_Write_Category0x18` |
| 5 | Pre-release server URL fix | `getDateInfo` → `api-<region>.marstekcloud.com`; region index 0–4 valid (previously 0–2); server type EEPROM 0x441 is loaded at startup; table `dev`/`staging` |
| 6 | HTTPS upload stability | `Quectel_HTTP_POST_SendAndVerify`: real tick loop instead of a blind `vTaskDelay`, ERROR/`+QIURC` handling, exact byte count, `AT+QHTTPPOST` timeout 60 → 120 s, outer timeout 600 → 6000; heap +10 KB |
| 7 | CT communication 1.1 s | `Ethernet_/Modem_SendStatusTelemetry`: meter query (Shelly `EM.GetStatus`/`EM1.GetStatus` etc.) at most every 0x44C = 1100 ticks (shared timestamp 0x20001F58); `Meter_ParseUDPMeterInfo` without a blocking `vTaskDelay` |

## 8. Diff v150 → v151 (summary)

Full table with old/new values and locations: `VNSD_Control_Diff_v150_v151.md`. Not in the changelog:

- PV upper bound when feeding in, without the 2500 W clamp on the start value (`Grid_Export_Power_Limiter`)
- Grid sync (`Grid_PowerSync_StateMachine`) now also adopts a mode change with 3 deviating phases (previously only with exactly 2)
- FreeRTOS heap 0x11800 → 0x14000
- MQTT `cd=0x42` "Get INV DEBUG info" (new case): `debug=1` turns on maintenance mode (`g_maintModeState=2`),
  `debug=2` publishes `g_vns_inv_debug` via `MQTT_Publish_WorkMode_Config`, `debug=3/4` toggle CAN category 0x18
  via `CAN_Write_Category0x18` and exit maintenance mode again
- MQTT `cd=0x13`: new subcommand `debug` → `MQTT_ConnectionStatus_Report_Send`
- Protocol AA system parameters: 7-slot table with range check instead of fixed values
- Device source ID `VenusD-<SN>` → `VNSD-0-<SN>`
- new shell commands `read_inv1`, `read_inv2`

Explicitly **not** changed (an earlier version of this document wrongly claimed otherwise): `MQTT_Config_ExtractFields`
already has 6 entries including `to=` in v150; `Task_udp`'s call order is byte-identical between v150 and v151
(details: `VNSD_Control_Diff_v150_v151.md`, "Unchanged" section).

## 9. Known v150 bugs in v151

| # | Bug | Status v151 |
|---|---|---|
| 1 | 30-minute reset of the Ethernet bridge (CH395 watchdog) | **changed**: ≥ 5 records backlog, 14 400 s, then MCU restart instead of chip reset, can be disabled via flag |
| 2 | WLAN twin 900 s (FC41D hardware reset) | unchanged |
| 3 | Receive loop requires CR LF at the end | unchanged |
| 4 | 86 s throttle from backlog > 3 | unchanged |
| 5 | ~4 s Modbus stall per upload (TLS handshake) | unchanged (no session cache, same priorities) |
| 6 | HTTPS upload without certificate validation (authmode 0) | unchanged |
| 7 | 32103 read alone → UsageFault | fixed (via the 32102 table change) |
| 8 | 32101 sign error | unchanged |
| 9 | MBAP length 4 instead of 3 on exception responses | unchanged |
| 10 | 44002/44003 read 0 after restart (shadow value) | unchanged |

## 10. Security findings (v150 and v151 alike, unless stated otherwise)

### 10.1 Unauthenticated remote console via Modbus register 46000 + TCP 8091

A Modbus TCP write (port 502, no authentication) to **register 46000 with value 0x5100**
sets `g_debugMode = 1` and stores it in EEPROM (0x900). From then on, `Task_rs485` serves the letter-shell on
RS485 **and on TCP port 8091** (CH395 socket 1) — user `VNSD-0 v151` with an empty password. Among other things, the shell allows
factory reset, setting WLAN credentials, starting an update, restarting, and `test` (turns on the inverter and sets
a CAN power setpoint). RS485 Modbus RTU is off in this mode. The state survives restarts.

### 10.2 Additional findings

| Finding | Location |
|---|---|
| HTTPS upload path without server certificate validation (`mbedtls_ssl_conf_authmode(…,0)`); MQTT validates (`2`) | `HTTPS_TLSSessionInit` 0x0803BF70, `mbedTLS_SSL_Connection_Init` 0x0801D2C4 |
| Certificate validity period is never checked: `mbedtls_x509_time_is_past/_is_future` are `return 0` stubs (`MBEDTLS_HAVE_TIME_DATE` off) — also applies to MQTT | 0x0804E394 / 0x0804E39A |
| Factory WLAN credentials SSID `SM471` / password `Hame147258` (factory default, fallback when EEPROM is empty, shell `set_wifi`) | `Config_LoadOrInit_DeviceSettings` 0x0802E17C, `Quectel_WiFi_AutoConnect_StateMachine` 0x0801681C |
| Embedded TLS certificates/keys encrypted with a hardcoded AES-128 key derived by byte rotation | `g_certAesKeyDeriveSeed` 0x20000FF0, `TLS_Cert_Decrypt_All` |
| MQTT client/device ID via AES-128-ECB with hardcoded, obfuscated keys; the plaintext ID goes into the debug log | `g_mqttDeviceIdKeySeed` 0x20001E9C among others, 0x080134D4 |
| Magic word `quit_debug_5100_` unlocks the shell in mode 2 | `RS485_DebugConsole_ScanQuitPhrase` 0x08055CA8 |
| `getDateInfo` and `backup-configs` in plaintext HTTP | §0a no. 5 |

## 11. Additional code bugs (newly found, verified)

| Bug | Location |
|---|---|
| **FC03 buffer overflow (v150 + v151):** the response (9 + 2·N bytes, N ≤ 125) is written into the 128-byte TX frame and then **always** copied into a mirror buffer that, after 32 bytes, spills into the MQTT RPC work area (0x6002683C…). From N > 11, RPC fields get overwritten; from N > 59, the TX frame runs into the RX frame. The response bytes themselves remain correct. | `FC03_Read_Handler` 0x08023A00, `g_modbusTcpTxFrame` 0x6002671C |
| BLE Cmd 0x27 (power history): a count > 15 is clamped to 15, but only counts < 15 are answered → no response for ≥ 15 | `BLE_App_Command_Dispatcher` |
| BLE Cmd 0x23 (generator on/off) responds with command ID 0x0F instead of 0x23 | `BLE_App_Command_Dispatcher` (`BLE_Send_Response('\x0f', &g_generatorEnable, 1)`) |
| `HTTP_Economy_TOU_ParseResponse`: `crc` and `crc2` are computed with the same XOR loop and checked ORed together — the second checksum has no effect (the XOR check itself works) | 0x08019578 |
| `Util_ReadBigEndianUInt16` returns `short` (sign issue for values > 0x7FFF) *(agent finding, not separately verified)* | – |
| **BMS pack index off by one (all versions 147–151):** CAN unit u ends up in record u−1 (`CAN_AA_StoreUnit*`), Modbus 34000+100·n and `get_all_bms_data` read record n; the MQTT BMS query (command 0x2A), however, reads record p for index p ≥ 1 → "BMS(1)" = CAN unit 2, unit 1 cannot be queried individually via MQTT, index 8 reads past the array (from offset 0x20 `g_pendingBootDescriptor`, read only) | `MQTT_Publish_BMS_Data` 0x08028DA4, `MQTT_CtrlCommand_Dispatcher` case 0x2A |
| **Cell statistic covers only 6 packs (all versions):** `BMS_CellVoltage_MinMax_PerPack` loops `pack < 6` although 8 records / 7 Modbus blocks exist → 7th pack missing from `g_bmsCellStat*` and the MQTT telemetry built from it | `BMS_CellVoltage_MinMax_PerPack`, `MQTT_Telemetry_String_Formatter` |

## 12. Processing status in Ghidra

| | v151 |
|---|---|
| Functions | 1956, all named (0 `FUN_`), all with a prototype (`__stdcall`), plate comment, local variables where determinable; 112 names tagged "low confidence" after the second-pass review, 31 corrected in the second-pass review |
| Flash coverage | 100%: 0 undefined bytes, 0 instructions outside functions |
| RAM globals | all ~1,830 named, typed, commented (exception: 2 offcut labels inside typed arrays) |
| Peripherals | registers of the N32G457 SVD (942 entries) + 16 XFMC + 5 SAC + core registers, plus `sysinfo` and `extflash_ctrl` |
| Switch tables | all TBB/TBH typed as arrays, 17 decompiler overrides |

## 13. Open items

- ~~Register block 0xA0001000 (external SPI flash "QSPI" driver): reserved according to the AT32F403A reference manual; the layout does not match the QSPI1 of the AT32F435. Which MCU block this actually is remains open.~~ **Resolved:** the MCU is a Nations N32G45x with XFMC; 0xA0001000 = `QSPI_BASE` (`XFMC_REG_BASE` 0xA0000000 + 0x1000) per the official `n32g45x.h`, confirmed by the shell command `xfmc_test` (see §1a).
- `Board_GetUniqueID_HexString` reads 0x1FFFF7F0/F4/F8; the documented UID ends at 0x1FFFF7F3.
- Who sets debug mode 2 in the field (`Debug_Mode_Set` callers from MQTT/BLE), and whether the shell command `modbus` also resets the mode in EEPROM, has not been checked.

## Appendix M. Methodology

- Function names: offline agents (41 batches) based on the decompilation, v150 names used only as a hint; second-pass review of 137 low-confidence functions after full naming was complete.
- Globals: 34 batches; sizes established via address gaps, `memset` sizes (Keil `__aeabi_memset(dst, n, c)`), and format strings. Names harmonized with all other versions (same object = same name, ~300 renames overall), including all 8 BMS pack records onto `g_bmsPack<k><Field>`, `g_spiFlashStatusCmdTable` and `g_meterCtStabilizeDelayTable` newly typed, and previously undocumented peripherals named: register block 0x40024000 (back then `RNGX_*`, meanwhile identified as the N32G45x SAC crypto coprocessor, see §1a), GPIOF/GPIOG at 0x40011C00/0x40012000 (marked reserved in the N32G45x user manual), `RCC_UNDOC_2C`, `PWR_UNDOC_0C`, `RTC_UNDOC_4C`, bit-band aliases `BB_*`.
- Corrected Ghidra artifacts: pointer + string merged together (~300), strings as code (> 250), unresolved TBB/TBH tables, literal pools in code (`__scatterload`), a spurious function in `small_primes_u8`, constant-propagation references into code, incorrect `BRANCH` overrides onto `pop {…,pc}`, `llhttp_errno_name` (jump into string data, CALL_RETURN override).
- Literal pool words holding a RAM/peripheral address typed as pointers, so the decompiler shows globals directly.
- Never `endTransaction(tx, False)` in ReVa scripts; errors in the script would otherwise keep partial changes.
