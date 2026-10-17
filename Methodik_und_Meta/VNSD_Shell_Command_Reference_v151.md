# Marstek Venus D — CLI Shell Command Reference (Control v151)

> 🇩🇪 Deutsche Version: [VNSD_Shell_Befehlsreferenz_v151_de.md](VNSD_Shell_Befehlsreferenz_v151_de.md)

**Status:** 2026-10-17.
**Firmware:** Control v151 (`VNSD-0`), build string `Sep 29 2026 09:42:27`.
**Method:** static reading of the decompiled `ShellCmd_*` handlers and of the CLI core (`CLI_*`). **Nothing was tested on hardware.**
**Companion document:** [VNSD_Shell.md](VNSD_Shell.md) (architecture and access paths, based on v149.2; corrected by this document where they differ).

---

## 1. How the shell works in v151

- The shell is a **letter-shell**: a normal text prompt. A command is `name arg1 arg2 …` followed by Enter. The line is tokenized (`CLI_TokenizeInput`) and the command is found by **name string** (`CLI_FindEntryByName`, `strcmp`). Numbers are parsed by `CLI_ParseNumber` (decimal or hex). Handlers take integer arguments; a handler without arguments ignores extra tokens.
- `CLI_DispatchInputByte` does **not** match commands. It only matches the **10 key-binding entries** of the table (entry type 9: arrows, backspace, delete, enter, clear). All other bytes are inserted into the line buffer. The "4-byte command patterns" described in [VNSD_Shell.md](VNSD_Shell.md) §3 are these key bindings, not command names.
- **Table:** `shellCommandTable` @ 0x08062cc4, **81 entries** = 68 commands + `help` + `clear` + 1 user entry `VNSD-0 v151` + 10 key bindings. 70 `ShellCmd_*` handler functions exist (68 + `help` + `clear`), and all 70 command names exist as strings in the image. The binding of each name string to its handler was **not** decoded entry by entry; the assignment below follows the handler names.
- **Login:** `CLI_InitSession` sets the "authenticated" bit when the session is created, so no password is asked. The user entry `VNSD-0 v151` has an empty password.
- **Output:** `printf` goes through `Protocol_PutChar`. With a TCP client connected to port 8091 (`g_debugTcpSocketConnected`) the text is sent over CH395 socket 1 (flush at LF, `$`, `:` or 256 bytes). Without a TCP client it goes out on the RS485 UART (only in mode 1). `Log_Printf` output is printed only when `g_debugMode == 1`, the log level allows it and logging is enabled. Several strings are Chinese.

### 1.1 Access (mode switch)

`Task_rs485` serves the shell only when `g_debugMode == 1` (EEPROM 0x900, valid 0–2):

| `g_debugMode` | What `Task_rs485` does |
|---|---|
| 0 (default) | RS485 Modbus RTU (`RS485_RTU_Frame_Dispatcher`); a TCP client on 8091 is not read |
| 1 | Shell (`Network_ReceiveAndDispatchData`) on TCP 8091 or RS485 |
| 2 | Waits for the phrase `quit_debug_5100_` (`RS485_DebugConsole_ScanQuitPhrase`), then switches to 1 |

- **Enable:** Modbus write of `0x5100` (20736) to register 46000 sets mode 1 and stores it in EEPROM 0x900 (`Write_Handler`; see [VNSD_Command_Register_45000_Block_v150.md](../Modbus_RS485_TCP/VNSD_Command_Register_45000_Block_v150.md)). Persistent across reboots.
  - Frame (Modbus TCP, port 502, unit 1, FC06 Write Single Register, register 46000 = 0xB3B0, value 0x5100): `00 01 00 00 00 06 01 06 B3 B0 51 00`. A normal Modbus device echoes the request as the answer. Not tested.
  - **Send exactly `0x5100`.** Other values in 46000 select OTA modes (`0x04D2`, `0x0929`, `0x0D80`, `0x11D7`) that delete the Modbus TCP / Wi-Fi / cloud tasks and stop the inverter.
- **Disable:** shell command `modbus` (sets mode 0 and stores it).
- **Side effect:** in mode 1 the RS485 Modbus RTU port is not served. Whether Modbus TCP (port 502) keeps running was not checked; it uses another socket.
- **Why the earlier live tests were silent:** the device was in mode 0, so nobody read the socket on 8091.

---

## 2. Commands

Address = handler function. "Args" are the handler parameters as seen in the decompilation.

### 2.1 Read-only (print values)

| Command | Handler | Args | Prints |
|---|---|---|---|
| `get_ver` | 0x0803be10 | – | Build string, `ems_type`, `boot_ver`, `ems_ver` (a constant, 0x97 = 151), `mppt_ver` (new and old), `bms_ver`, `vns_ver` |
| `get_venus_data` | 0x0803baa4 | – | Inverter block: state, error/warning words, grid and off-grid voltage/frequency/power, battery power/voltage, temperatures, power limits, energy counters, hard/soft/boot version, work mode, sleep flag, battery mode |
| `read_inv2` | 0x08053334 | – | `g_vns_inv_debug` struct (work mode, limits, allow flags, `err_state`, `bat_err_req`, …) |
| `get_all_bms_data` | 0x0803ae6c | – | For every record `0 … bat_total_nb-1`: voltage, current, SoC, cycles, charge limits, MOS flag, min/max cell voltage and temperature, err1/err2/warn1, BMS version, charge/discharge power, SoH, 16 cell voltages, NTC temperatures, environment and MOS temperature, battery type |
| `get_main_bms_data` | 0x0803b394 | – | PGN 1801/1802/1803/1804/1007 totals: voltage, current, SoC, energy, `bat_total_nb`, `bat_online_mask`, `work_bat_idx`, DOD, charge limits, flags, errors, lock flag, self-check |
| `get_mppt_data` | 0x0803b754 | – | MPPT state/error/temperature/warning, 4× PV voltage/current/power, capacity counters, battery and bus voltages (same data as MQTT `cd=45`) |
| `ems_rtrl_info` | 0x0803a2d4 | – | Last values the EMS sent over CAN to the inverter (power, send counter) and to the MPPT (voltage, 2 temperatures, power, mode, send counter) |
| `energy_show` | 0x0803a41c | – | Total/month/day charge and discharge energy (Wh) |
| `err_code` | 0x0803a63c | – | `err_code` and `warn_code` bit masks of the controller |
| `get_log` | 0x08053030 | `0` / `1` | `0`: 20 error-log slots; `1`: 20 event-log slots (date and codes) |
| `get_ota_log` | 0x0803ba04 | – | OTA CRC, `ota_flag`, `fail_flag`, 4 error code/data pairs |
| `ext_info` | 0x0803a6cc | – | Charge/discharge cut-off SoC, max charge/discharge power, grid standard |
| `power_check` | 0x0804ff6c | – | CT power, Venus/BMS power, load, PV, backup power |
| `rtos_status` | 0x0805378c | – | FreeRTOS task list as JSON (name, state, priority, free stack, number) |
| `FreeRtos_mem` | 0x0802fa38 | – | Free heap and historic minimum (via `Log_Printf`, so only visible with logging on) |
| `reset_reason` | 0x080512de | – | Last reset cause |
| `get_time` / `rtc_show` | 0x080531a4 / 0x0802fa30 | – | RTC date and time |
| `wifi_info` | 0x080531f4 | – | RSSI and link status |
| `ch395_get_ip` | 0x08037e70 | – | CH395 IP/gateway/mask/DNS/DHCP status |
| `ch395_ver` | 0x0803866c | – | CH395 chip version |
| `mac_ble` | 0x08052f78 | – | BLE/cloud identity fields (type, id, mac, mac_devid, vid, gid, xid) |
| `get_meter_mac` | 0x0803b734 | – | Prints the variable `g_deviceSerialNumber` under the label `meter_mac` (label and variable do not match) |
| `show_modbus` | 0x080539a4 | – | Modbus slave address |
| `check_ssl_cert` | 0x08052fb8 | – | TLS certificate version |
| `ver_fc` | 0x08052ffc | – | Sends `AT+QVERSION` to the FC41D and prints its firmware version |
| `key_state` | 0x0803c208 | – | Button debounce counters, event type, key state, encode flag |
| `get_old` | 0x08053180 | – | Burn-in step counter |
| `help` | 0x08052300 | none / name | Command list or help for one command |
| `clear` | 0x08051b64 | – | Clear screen |

### 2.2 Change settings or log state (low risk)

| Command | Handler | Args | Effect |
|---|---|---|---|
| `set_level` | 0x080329d8 | level | Log level (`Log_SetLevel`) |
| `debug_fc` | 0x08052f6c | mode | `Log_SetModeAndLevel(mode)`; per the decompiler summary 0 none, 1 FC41D, 2 CT, 3 SYS, 4 CH395, 5 all (not verified) |
| `trace_debug` | 0x08053b64 | `2025` | Sets `g_debugPrintfEnableFlag` and `g_httpVerboseLogFlag` when the argument is 2025 (0x7e9), otherwise clears them |
| `api_set` | 0x080537c4 | `1` / `0` / other | Enable / disable (EEPROM 0x371) / print the local UDP API state |
| `api_port` | 0x080538ac | port | Local API port (EEPROM 0x372) |
| `modbus` | 0x0805330c | – | Sets `g_debugMode = 0` (EEPROM 0x900): back to Modbus RTU |
| `get_backup_datas` | 0x0803b324 | value | **Sets** `g_backupDataRequestFlag` (same flag as MQTT `cd=61`), despite the name |
| `read_inv1` | 0x08053328 | – | Sets `g_maintModeState = 2` |
| `code_test` | 0x08037c08 | value | Sets `g_encodeFlag` |
| `Resume_task` | 0x0802f680 | – | Resumes suspended FreeRTOS tasks |
| `ch395q_test` | 0x08038820 | cmd | Sends a raw command byte to the CH395 and prints the answer |

### 2.3 Actuating or destructive (do not run on a productive device)

| Command | Handler | Args | Effect |
|---|---|---|---|
| `restore_factory` | 0x0802f590 | `1` / `2` / `3` | `Factory_Reset`: 1 clear all, 2 clear part (keep accumulated data), 3 clear certificates |
| `venus_rfd` | 0x08056ff2 | mode | Inverter factory reset (`Inverter_Write_Reg_0x07_Value(1, mode)`); decompiler summary: 0 all, 1 most, 2 most except accumulated energy |
| `venus_poweroff` | 0x08056fe4 | sleep_flag | Inverter sleep/power-off flag register (`Inverter_Set_Flag_Reg_0x06`) |
| `reset` | 0x080512d6 | – | Reboots the MCU |
| `set_wifi` | 0x0805394c | – | Overwrites the stored WLAN SSID/password with the built-in defaults `SM471` / `Hame147258` (EEPROM 0x400/0x420) and triggers a reconnect |
| `delate_ssl_cert` | 0x08052fe8 | – | Deletes the stored user TLS certificate and key |
| `fc41d_reset` | 0x08052ff0 | – | Hardware reset of the FC41D module |
| `hard_reset_395` | 0x0803bf4c | 0/1 | Drives the CH395 reset GPIO low/high |
| `recon_ch395` | 0x080512cc | – | Reset and re-init of the CH395 |
| `free_mqtts` | 0x0803a9d8 | – | Sets `g_ch395ResetPendingFlag`; the decompiler summary says "MQTTS teardown", the variable name says CH395 reset — unresolved |
| `change_soc` | 0x08052f60 | 0…100 | Sends the value as 8-byte CAN frame `0x100201AA` (`Config_Write_TargetSOC_Percent`); values above 100 are ignored. It is a **target SoC write toward the BMS**, not a change of the reported SoC. The effect inside the BMS was not traced |
| `full_dischrg` | 0x0803a9e8 | 0/1 | `Config_Write_BoolFlag_0x375` (surplus feed-in, same flag as MQTT `cd=43`) |
| `lck_bms` | 0x0803c298 | lock | `Config_Apply_WorkModeReg(lock)` (BMS lock/unlock, same path as MQTT `cd=44`) |
| `ctl_mppt_power` | 0x080388c4 | power | Sets `g_gridExportPowerCap3 = power` and `g_maintModeState = 3` |
| `ctrl_led_light` | 0x08038978 | 0…7 | LED brightness (0 brightest) via the validated setter |
| `led_continue_ctrl` | 0x0803c2a4 | mode, value | Despite the name: `Inverter_Power_Setpoint_Apply(mode, value)` |
| `test` | 0x080539d0 | – | `g_maintModeState = 2`, inverter on, then a CAN power command with raw value 1000 |
| `set_old` | 0x0805392c | step < 6 | Burn-in: `g_maintModeState = 1`, inverter work-mode registers 0x06 and 0x60 |
| `vns_ate_test` | 0x08057000 | – | `Config_Apply_SingleReg(1)` (inverter ATE test command) |
| `debug_mode` | 0x08056f98 | 0/1 | `Debug_Mode_Set`: clears four CAN 0xC2 sub-registers; with 1 it also sets `g_debugMode = 2` (shell input is then replaced by the quit-phrase scan until `quit_debug_5100_` is typed) |
| `update_ems` / `update_vns` / `update_bms` / `update_mppt` | 0x08055f60 / 0x08056268 / 0x08055f20 / 0x08055fa0 | – | Set `g_otaTargetDevice` = 1 / 4 / 3 / 2. `Task_rs485` then calls `CLI_FirmwareUpdate_Prepare` (YModem over TCP/RS485, see §4) |
| `button_ctrl` | 0x08037dcc | level | Drives the button GPIO (port E, mask 0x8000) low (0) or high |
| `eeprom_flash_test` | 0x0800b78c | addr | **Overwrites 32 bytes of EEPROM at `addr`** (0xFF, 0…31, then 0xFF again) |
| `flash_test` | 0x0803a7b0 | addr | **Erases and rewrites 30 bytes of flash at `addr`** |
| `xfmc_test` | 0x08035a9c | – | External SRAM self-test |
| `check_periph` | 0x08053a64 | – | Peripheral self-test; prints EEPROM/flash/CH395/SRAM OK or fault (Chinese) |

---

## 3. Use for this project

Read-only commands that can verify static findings on a live device (to be run only after a decision by the owner, and only in mode 1):

| Question | Command |
|---|---|
| Version check for every new firmware (standing check list) | `get_ver`, `ver_fc` |
| Pack capacity and online packs (standing check "pack capacity") | `get_main_bms_data` (`bat_total_nb`, `bat_online_mask`), `get_all_bms_data` |
| Pack numbering: record index versus CAN unit (record index = CAN unit − 1) | `get_all_bms_data` against MQTT `cd=42` |
| Inverter and BMS error words | `err_code`, `get_venus_data` (`error_code1/2`), `read_inv2`, `get_log` |
| Register values that are live in RAM (limits, cut-off SoC, grid standard) | `ext_info`, `get_venus_data`, `read_inv2` |
| What the EMS sends to MPPT/inverter | `ems_rtrl_info` |
| Task list and stack margins | `rtos_status`, `FreeRtos_mem` |
| OTA result | `get_ota_log` |

Observations for the register classification: `get_backup_datas` and `read_inv1` write state although they look like readers; `led_continue_ctrl` is not an LED command; `get_meter_mac` prints the device serial variable. `lck_bms`, `full_dischrg` and `get_backup_datas` reach the same targets as MQTT `cd=44`, `cd=43` and `cd=61`. `change_soc` has no MQTT counterpart in the command map.

---

## 4. The `update_*` path: firmware upload over TCP or RS485 (YModem)

Read from the v151 decompilation on 2026-10-17; **not tested on hardware**. This is the documented control-flow of a vendor-shipped update path, kept here for the project's own standing firmware-diff checklist — not instructions for third-party use.

1. **Trigger.** `g_otaTargetDevice` must be non-zero: Modbus write of 1234 / 2345 / 3456 / 4567 to register 46000 (targets 1 / 4 / 3 / 2) or the shell commands `update_ems` / `update_vns` / `update_bms` / `update_mppt` (1 / 4 / 3 / 2). `Task_rs485` tests this flag before the debug-mode branch, so it applies regardless of `g_debugMode`.
2. **Preparation (`CLI_FirmwareUpdate_Prepare`).** Deletes the tasks FC41D, parser, HTTP, UDP, CT, channel-HTTP, channel-MQTTS and Modbus TCP, calls `Inverter_BeginShutdown(0)`, clears the CH395 receive buffer, then calls `YModem_FirmwareUpgrade_Handler`. The inverter stops; cloud and Modbus TCP are unavailable until the transfer ends.
3. **Transport (`YModem_ReceiveFile`).** With a client connected to port 8091 the blocks come from CH395 socket 1 (`CH395_Packet_Receive_Parse`); otherwise from the RS485 parser. Blocks are the standard YModem shape (SOH/STX header, sequence byte, inverted sequence byte, payload, 2 trailing bytes). The parser checks only the header byte and `seq == ~seq`; it does not evaluate the two trailing bytes of each block.
4. **Checks on block 0 (file header).** The file-name tag (case-folded, matched against EMS/BMS/INV/MPPT in that order) must resolve to the same target as `g_otaTargetDevice`; the announced size must be at most 0x7CFFF (511999) bytes. The target's staging region (from a 4-entry address table) is then erased and the data blocks are written there.
5. **End-of-transfer check (`OTA_CRC_Verify`).** A running Modbus CRC16 over the bytes as received is compared against the CRC16 of the same staged bytes read back from flash. Both operands derive from the one transfer, so this is a transport/flash-integrity check, not an authenticity check: the handler receives no independently supplied hash, signature or key to compare against. The separate 0x401/0x402/0x403 check chain (`OTA_FW_Verify_SetStatus`, which does compare against an *announced* CRC and a device/model tag) belongs to the HTTP/BLE OTA path (its caller is `OTA_Flash_Page_Writer`) and is not invoked here.
6. **Activation (`ProcessFirmwareUpdateCommand`).** For target 1 (this Control) it writes a pending-boot descriptor (staging address, size, CRC16, `VNSD-0` tag, its own CRC16) to `BOOTLOADER_ParamPage` (0x08004000, 40 bytes) and reboots. For targets 2–4 it records a pending CAN-forwarded update for MPPT/BMS/inverter without rebooting.
7. **On failure** (abort, wrong tag, oversize, CRC mismatch): `System_ResetToDefaults` re-enables the inverter and resumes the suspended tasks; it does not recreate the deleted tasks, so a full reboot may be required (not verified).

**The Control bootloader was not checked, and cannot be from what this project holds.** The bootloader is a separate 18 KB region (0x08000000–0x080047FF) below the app's link address (0x08004800, confirmed by the vector table in this v151 export and consistent across all analysed Control builds D/A/E3). It is firmware that runs on the same MCU but is **not part of any app image this project has decompiled or dumped** — every `.bin` examined so far starts at 0x08004800. The app's only documented interaction with it is writing the 40-byte descriptor to 0x08004000 and reading a version word at 0x08001000; what the bootloader itself checks before flashing that staged image over the running app is unknown. Reading it needs a flash dump of the low 18 KB, e.g. via SWD/JTAG (see [VNSD_SWD_JTAG_Dump_Plan.md](VNSD_SWD_JTAG_Dump_Plan.md)) — not attempted for this question.

**Not the MCU's factory bootloader.** The N32G45x (like its STM32-pin-compatible relatives) has its own ROM-resident system bootloader for UART ISP, entered via the BOOT pin/option bytes and mapped in a dedicated system-memory region, not in main flash. This 18 KB region instead sits exactly at the normal flash-boot vector (0x08000000) — the address the CPU reaches after an ordinary reset with flash boot selected — and the descriptor format it is meant to parse (`VNSD-0` tag, 40-byte structure, Modbus CRC16) is product-specific. Both points indicate a vendor-written second-stage bootloader that ships as part of this firmware, not generic silicon-vendor code — but this is inferred from its location and from what the app writes to it, not from reading the bootloader's own code.

Operationally: the transfer stops the inverter and both network control paths for its duration, and an interrupted or truncated transfer's recovery behavior was not verified — not something to exercise on a production unit.

---

## 5. Open points

1. Not decoded: the table entry for each command (name string to handler pointer, flags). The mapping above follows the handler names. Reading the 81 entries from the image would settle it.
2. Argument syntax was not tested (separator, hex input, missing arguments). Handlers read their parameters without checking the argument count.
3. The Control bootloader (0x08000000–0x080047FF) and the MPPT/BMS/inverter bootloaders are unread; see §4's closing paragraph for why and what it would take.
4. `free_mqtts`: name versus flag. `debug_fc`: mode meanings are taken from the decompiler summary.
5. Whether Modbus TCP stays available in mode 1.
6. Other versions (147, 149, 149.2, 150; A and E3 lines) were not compared. Per the v150 analysis, v150 has 66 commands (without `read_inv1`/`read_inv2`); v151 has 68.
