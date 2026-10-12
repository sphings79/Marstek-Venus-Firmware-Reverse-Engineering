# Control FW v151 (Venus A) — Full Analysis and Differences to A v150.9 and Venus D v151

> 🇩🇪 Deutsche Version: [VNSA_Control_FW_v151_Vollanalyse_de.md](VNSA_Control_FW_v151_Vollanalyse_de.md)

**Status:** completed 2026-10-11. All statements were checked against the code of this image; open points are listed in §13.
**Binary:** `151_control_VNSA-0_ems_app_0151_0928_095826.bin` (archive `Marstek FW Archiv/firmwares/VNSA-0/Control/151/`)
**Predecessor:** Venus A Control v150.9 (beta), see [VNSA_Control_FW_v1509_Full_Analysis.md](VNSA_Control_FW_v1509_Full_Analysis.md)
**Reference:** Venus D Control v151, see [VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md](VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md)
**Diff tables:** [VNSA_Control_Diff_v1509_v151.md](VNSA_Control_Diff_v1509_v151.md), [VNSA_Control_Diff_v150_v151.md](VNSA_Control_Diff_v150_v151.md)
**Ghidra:** `/VNSA/Control/Control_151_VNSA-0_ems_app_0151_0928_095826.bin` (project `Marstek_Venus`)
**Decompilation:** `Dekompilate/VNSA_Control_v151/` (local only, not in the repo): 1942 functions, 0 decompilation failures, + `listing/` (flash listing, `globals_and_registers.txt`)
**Modbus descriptor table:** byte-identical to v150.9, see [../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv](../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv)

---

## 0. Summary

- **A v151 = A v150.9 + the cloud/HTTP part of Venus D v151.** 97.6 % of the image is identical to A v150.9 once pointers and
  BL offsets are masked. Of the 1942 functions, 1924 are instruction-identical to v150.9, 17 changed, 1 is new
  (`MQTT_CtrlResponse_Transmit`), none was removed. 15 of the 17 changed functions are now instruction-identical to the
  corresponding Venus D v151 function; the other two (`Config_LoadOrInit_DeviceSettings`, `Quectel_AT_SendAndPollResponse`)
  differ from D only in the A constants respectively a call offset.
- **What changed versus v150.9** (all items match the release note): `getDateInfo` now goes to `http://api-<region>.marstekcloud.com`
  (no longer `hamedata.com`), cloud server types 0–4 with the corrected pre-release prefixes `dev`/`staging`, server type loaded at boot,
  HTTP POST over the cellular/Wi-Fi module rewritten (real response loop, 120/120 s timeouts), TOU response with a second checksum,
  AI schedule query answered page by page, CH395 keepalive idle 10 s → 60 s, CT/meter query at most every 1.1 s,
  the connection watchdog in `HTTP_ClientTask_Dispatcher` rebuilt (3600 → 14 400 ticks, then `System_Reboot`, suppression flag).
- **Unchanged versus v150.9:** Modbus register map (descriptor table and `Write_Handler`), BLE command dispatcher, all CAN senders
  and receivers, power limits (1500 W), power classes (800/1200/1500 W), 13 cells per pack, 7 per-pack records, OTA/YMODEM checks,
  RS485 on USART1, RAM layout.
- **Venus A specifics compared with D v151 are the same as in v150.9** (§5): 1500 W limits, A power classes, 13 cells, USART1 without
  driver-enable pin, no key/protection monitor, identity `VNSA`. Newly documented: Modbus 44003 (max discharge power) is rejected above
  1500 W by `Write_Handler` (D: 2500 W) — this constant already existed in v150.9 although the v150.9 analysis called `Write_Handler`
  identical to D.

## 0a. Checklist (standing checks, A v151 compared with A v150.9)

| # | Question | Answer A v151 | Evidence |
|---|---|---|---|
| 1 | Modbus registers different? | **No.** Descriptor table `g_modbusRegDescTable` @ 0x2000034C (246 entries, 30000–38014) byte-identical to v150.9; `Write_Handler` @ 0x08056A7C and `FC03_Read_Handler`/`Read_Serializer` instruction-identical. Register 30200 now reads the version word 0x97 = **151**. | §6 |
| 2 | BLE different? | **No.** `BLE_App_Command_Dispatcher` @ 0x0800C5C8 instruction-identical (name `MST_VNSA_…`, cmd 0x15 classes 800/1200/1500 W, cmd 0x16/0x17 ≤ 1500 W). | §5 |
| 3 | AC limits / feed-in limit? | **Unchanged: 1500 W** (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock`, MQTT, JSON-RPC ±1500 W, `Remote_Power_Command_Execute` ±1500 W, Modbus 44003 > 1500 W rejected); grid export limiter upper clamp 2400 W. | §5 |
| 4 | Firmware update via BLE possible? | **Yes**, OTA/YMODEM code identical to v150.9 (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate` accept only `VNSA-0`; YMODEM bin-type check `OTA_ClassifyDeviceTypeTag` already present in v150.9). | §4.4 |
| 5 | Cloud domains? | `getDateInfo` **`http://api-<region>.marstekcloud.com/app/neng/getDateInfo<suffix>.php`** (v150.9: `http://<region>.hamedata.com/…`), plain HTTP. Upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>` unchanged. Still hamedata.com: backup-configs, schedulings/policies, getDevicesApply, FC41D OTA. | §4.1 |
| 6 | Pack capacity? | Control unchanged: 7 per-pack records (Modbus 34000–34699), `BMS_CellVoltage_MinMax_PerPack` scans 6 packs × 13 cells. The A BMS 1105 sends packs 1..6 (`pack_id < 7`), so pack 7 reads 0. | §7 |
| 7 | Diff table | [VNSA_Control_Diff_v1509_v151.md](VNSA_Control_Diff_v1509_v151.md) (name-paired, complete), [VNSA_Control_Diff_v150_v151.md](VNSA_Control_Diff_v150_v151.md) (name-paired, complete; replaces the earlier derived diff) | — |
| 8 | Re-check of non-functional registers | All 216 A registers with status FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL stay as classified: none of the Control code they depend on changed (descriptor table, `Write_Handler`, CAN senders/dispatchers, per-pack records), and no new A micro/BMS firmware is part of this release. | §6 |

## 1. Binary fingerprint

| | A v151 | A v150.9 (predecessor) |
|---|---|---|
| Size | 391 168 B (0x5F800) | 391 168 B |
| SHA-256 | `5cceb230f28c9c3a67be5e897afd5722f795b3e8f6c39d58d5a2c2873d994346` | `a8c58127…9f23` |
| Link address | **0x08004800**–0x08063FFF (18 KB bootloader below) | same |
| MCU | Nations **N32G45x** with XFMC (Cortex-M4F), 144 KB SRAM | same |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW) | same |
| Build string | `Sep 28 2026 09:54:57` | `Sep  8 2026 09:34:19` |
| Version | `VNSA-0 v151`; version word 0x97 (151) @ 0x08010000 followed by `VNSA-0` | `VNSA-0 v1509`, 0x5E5 |
| Image footer | `VNSA` + 12 zero bytes @ 0x08063FE0, 0xFF fill from 0x08063A0C | same footer |
| Release note (vendor, translated) | Modbus TCP stability; DOD abnormal state could not discharge; YMODEM upgrade type check; AI mode check and query of the current AI strategy; pre-release server URL error (data upload failure); HTTPS upload stability; CT communication every 1.1 s | — |

Base proof: the vector table at 0x08004800 resolves (initial SP 0x20022010, Reset 0x08004A71 = Keil pattern
`ldr r0,=SystemInit; blx r0; ldr r0,=__main; bx r0`), `SystemInit` @ 0x08032D88 writes `SCB_VTOR = &__Vectors` (0x08004800),
and 177 Thumb function pointers hit a function prologue at this base versus 5 at 0x08000000.

## 2. Memory map (Ghidra)

| Block | Range | Contents |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | not in the image |
| flash | 0x08004800–0x08063FFF | application |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-bit UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EFF | RW initial values (LZ77 @ 0x08062B74, 2492 B) |
| .bss | 0x20002F00–0x2002200F | zero-initialised; `ucHeap` @ 0x20003144 (80 KB) |
| sram_unused | 0x20022010–0x20023FFF | up to the N32G45x 144 KB limit |
| .xmc_bss | 0x60000000–0x60026AF7 | external SRAM via XFMC (LZ77 image @ 0x08063530, 1244 B) |
| peripherals | apb1/apb2/ahb/bit-band/ppb | register names from the Nations N32G457 SVD |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC and QSPI controller |

Scatter table `Region$$Table$$Base` @ 0x08062634: .data 0x08062B74 → 0x20000000 (0x2F00, `__decompress` @ 0x08005FBE),
XMC 0x08063530 → 0x60000000 (0x26AF8), .bss zero-init 0x1F110 @ 0x20002F00. Region sizes are identical to v150.9 and D v151, and the
unpacked `.data`/XMC images are identical to both apart from flash pointers, so every RAM global sits at the same address and has the
same name as in v150.9 and D v151.

## 3. Relationship to A v150.9 and D v151 (method and numbers)

| Category | Functions | Meaning |
|---|---|---|
| instruction-identical to v150.9 | 1924 | names, prototypes, plates, locals, tags taken from the v150.9 program |
| changed versus v150.9 | 17 (+ 10 that only reference relocated tables/literals or the new version string) | 15 of them instruction-identical to D v151 (markup taken from D v151), 2 changed versus both (§4) |
| new versus v150.9 | 1 | `MQTT_CtrlResponse_Transmit` @ 0x08023990 (exists in D v151; paginated `ctrl` responses) |
| removed versus v150.9 | 0 | — |
| differing from D v151 | — | exactly the A-specific functions already listed in the v150.9 analysis (limits, power classes, identity strings, USART1, 13 cells; §5) |
| missing compared with D v151 | 15 | unchanged (§5.4) |

Method: offline segment map of the images (flash pointers and BL/B.W offsets masked) against v150.9 and D v151; function layout,
names, signatures, data types, labels, comments, references and flow overrides were carried over from v150.9 for the v150.9 segments
and from D v151 for the code that only D v151 shares; then a name-paired instruction diff (call targets by name, referenced strings,
immediate values) over all 1942 functions against both references, manual review of every changed function, gap typing and the
verification gates (§12).

## 4. Changes versus A v150.9

All changed code is the Venus D v151 cloud/HTTP code; the descriptions below were verified on this image.

### 4.1 Cloud endpoints and server types (release note item 5)

| Function | v150.9 | v151 |
|---|---|---|
| `HTTP_Cloud_Reporting_Dispatcher` @ 0x08012424 (Quectel `AT+QHTTPCFG`) | `http://%s.hamedata.com/app/neng/getDateInfo%s.php?…` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php?…` |
| `HTTP_GetUrlData` @ 0x08019910 (CH395 path) | same hamedata URL, server types 0–2 accepted | marstekcloud URL, server types 0–4 accepted (others → 0) |
| `g_cloudDomainUrlTable` @ 0x20000FC8 | eu, www, us, eu-dev, eu-staging | eu, www, us, **dev**, **staging** |
| `Config_LoadOrInit_DeviceSettings` @ 0x0802DB28 | EEPROM 0x441 (server type) read on demand only; no first-boot default for 0x395 | loads 0x441 and 0x395 at boot; first boot writes 0x395 (SEL LED) = 1 |
| version in URLs/telemetry/device info | 0x5E5 (1509) | 0x97 (151) |

The suffix table `g_cloudGetDateInfoSuffixTable` @ 0x20000FDC (eu, '', us, eu-dev, eu-staging) is unchanged. The upload endpoint
`https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>` and the remaining hamedata.com URLs are unchanged.

### 4.2 HTTP upload (release note item 6)

| Function | v150.9 | v151 |
|---|---|---|
| `Quectel_HTTP_POST_SendAndVerify` @ 0x08010438 | waits blindly, no ERROR/`+QIURC` check, NUL-terminated send | polling loop with ERROR/`+QIURC` detection, exact byte count, `[Http Tx…] post data=%s` log |
| `HTTP_Task_StateMachine_Handler` @ 0x08010B4C | `AT+QHTTPPOST=%d,60,60`, outer timeout 600, fixed length | `AT+QHTTPPOST=%d,120,120`, outer timeout 6000, length from `g_cloudUploadBodyLen` |
| `Cloud_UploadTelemetryData` @ 0x0801B300 | — | same payload, body length handling as D v151, version 151 |
| `HTTP_Economy_TOU_ParseResponse` @ 0x08019418 | one XOR checksum | additionally `crc2` (same XOR over the data, accepted if either matches), new error text with `crc2` |
| `HTTPS_FetchUrlData` @ 0x0801BD1C | — | one small change in the parameter setup (D v151 code) |

### 4.3 Connection watchdog and Modbus TCP (release note item 1)

| Function | v150.9 | v151 |
|---|---|---|
| `CH395_ConfigTcpKeepalive_Defaults` @ 0x08037A84 | keepalive idle 10 000 ms | 60 000 ms |
| `HTTP_ClientTask_Dispatcher` @ 0x0801A344 | no-link counter `g_httpNoLinkRebootTick` limit 3600 | limit 14 400, then `System_Reboot()`; counter is held at 0 while `g_httpWatchdogSuppressFlag` (0x60025B1E) is 1 |

`CH395_Init_Modbus_TCP_Socket` was already the D v151 variant in v150.9.

### 4.4 AI schedule, CT, meter (release note items 4 and 7)

| Function | v150.9 | v151 |
|---|---|---|
| `AI_ScheduleQuery_Response_Send` @ 0x080237B8 | answer `cd=58` in one 1024-byte buffer (`AI%d=…`), sent inline | paginated: 24 entries per page, `cd=%d,pg=%d/%d,` header, `AI%u=…` lines with bounds check, sent via the new `MQTT_CtrlResponse_Transmit` |
| `Modem_SendStatusTelemetry` @ 0x08031538, `Ethernet_SendStatusTelemetry` @ 0x08031728 | period 2000 ticks | 1100 ticks (1.1 s; meter `EM.GetStatus`/`EM1.GetStatus` query) |
| `CT_GridPower_Controller` @ 0x08030D20 | — | additionally resets a positive CT setpoint to 0 (D v151 code) |
| `SMR_TIC_SelfUse_PowerController` @ 0x0800851C | — | only the log line numbers changed |

Release note items 2 (DOD) and 3 (YMODEM type check) were already contained in v150.9 (`Battery_EPS_Reserve_SOC_Hysteresis`,
`YModem_ReceiveFile`/`OTA_ClassifyDeviceTypeTag` are instruction-identical), see the v150 → v151 diff.

### 4.5 Identity

`VNSA-0 v151` (`CLI_InitSession`, shell user), build `Sep 28 2026 09:54:57` (`ShellCmd_get_ver`), `g_deviceTypeCode` = 0x97.
`Quectel_AT_SendAndPollResponse` differs from v150.9 only in a call offset (same logic).

## 5. Remaining differences to Venus D v151

Unchanged from v150.9 (the functions are instruction-identical to v150.9):

### 5.1 Power limits and power classes

| Place | A v151 | D v151 |
|---|---|---|
| `Config_Write_U16_0x202` (max charge power, EEPROM 0x202) | clamp 1500 W | 2500 W |
| `Config_Write_U16` (max discharge power, EEPROM 0x204) | clamp 1500 W | 2500 W |
| `Config_ClampDefaults_ParamBlock` | charge 0/>1500 → 1500 W; discharge 0/>1500 → 800 W | 2500 W |
| `Config_LoadOrInit_DeviceSettings` first boot | max charge 1500 W, max discharge 800 W | 2500 W |
| `Write_Handler` 44003 (max discharge power) | > 1500 → exception 3 | > 2500 |
| `Write_Handler` 44002 (max charge power) | > 2500 → exception 3; raw value to inverter reg 0x04, EEPROM value clamped to 1500 W | same check, clamp 2500 W |
| BLE cmd 0x15 power class | 800 → 1, 1200 → 4, 1500 → 5 | 800 → 1, 1500/2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–1500 W / ≤ 1500 W | 300–2500 W / ≤ 2500 W |
| MQTT `md=` default, `MQTT_Config_Command_Handler` | 1500 W | 2500 W |
| `MQTT_JSON_RPC_Dispatcher` passive setpoint clamp | ±1500 W | ±2500 W |
| `Remote_Power_Command_Execute` | ±1500 W | ±2500 W |
| `Grid_Export_Power_Limiter` upper clamp | 2400 W | 4000 W |

### 5.2 Battery

`BMS_CellVoltage_MinMax_PerPack` @ 0x08017A58 scans **13 cells** per pack (D: 16).

### 5.3 RS485 / serial port

USART1 (PA9/PA10, IRQ 37, 115200 8N1) without driver-enable pin: `RS485_UART_Config_Init`, `RS485_UART_Send`, `RS485_UART_SendByte`,
`Protocol_PutChar`, `USART1_IRQHandler` @ 0x0803313C. D: USART2 + `RS485_DE_Pin_Init`/`RS485_Set_TX_Mode`/`RS485_Set_RX_Mode`.

### 5.4 Functions missing in the A (compiled out)

`Key_ProtectionMonitor`, `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`,
`Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot`,
`CAN_ParallelInvLink_DebounceHold`, `Board_GPIO_Init_ControlPins`, `RS485_DE_Pin_Init`, `RS485_Set_TX_Mode`, `RS485_Set_RX_Mode`,
`USART2_IRQHandler`; `OTA_Is_VNSD_Model` exists as `OTA_Is_VNSA_Model`. `MQTT_CtrlResponse_Transmit` (missing in v150.9) is now present.

### 5.5 Other details

`Inverter_Default_Init_Once` only sends `Inverter_Set_Flag_Reg_0x05(0)` (D additionally `Inverter_Set_Reg_0x04(0)`);
`CAN_Broadcast_DateTime` second 0xCC frame DLC 8 (D: 7); `Grid_Export_Power_Limiter` maintenance mode acts on old-step 1 or 5 with
mode 3 (D: step 5, mode 1); `WDT_SystemHealthService` without key monitor; identity strings `VNSA-0`, `MST_VNSA_`, `Venus A`,
`VNSA-0-<serial>`.

## 6. Modbus

- **Read range:** `g_modbusRegDescTable` @ 0x2000034C, 246 × 12 B, byte-identical to v150.9 and D v151; `FC03_Read_Handler` @ 0x080233A8 and
  `Read_Serializer` @ 0x0805597C instruction-identical (the known D bugs apply).
- **Write range:** `Write_Handler` @ 0x08056A7C instruction-identical to v150.9; compared with D v151 only the 44003 limit differs (§5.1).
  46000 = 0x5100 (debug mode → shell on RS485 and TCP 8091), 45000 block, 46500–46544, 47001 behave as on the D.
- **Re-check of non-functional registers (standing check 8):** the A register map (`VNSA_Register_Map_Final_generated.csv`) has 216
  registers with status FUNCTIONLESS (135), CONDITIONAL (67), NO_OP (12) or UNKNOWN (2). Their reasons lie in the descriptor table,
  `Write_Handler`, the CAN senders/receivers, the per-pack records, the A micro 1211 and the A BMS 1105. None of these changed in
  Control v151 and no new micro/BMS image is involved, so every one of them stays as classified ("still so").

## 7. Internal CAN bus and packs

All CAN send sites, `CAN_Build_Arbitration_ID` @ 0x08037798, `CAN_FrameDispatcher` @ 0x08033A08 and `Protocol_AA_CommandDispatch`
@ 0x0803431C are instruction-identical to v150.9; the tables of the v150.9 analysis (§6 there) apply unchanged. Which class-2/4
frames the A micro consumes is documented in the A micro 1211 analysis ([../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Full_Analysis.md](../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Full_Analysis.md)):
it ignores class-4 0x18, 0x30–0x3E, 0x40–0x43 and class-2 0xFB.
Pack capacity: the Control keeps 7 per-pack records (Modbus 34000–34699); the A BMS 1105 sends packs 1..6 (`CAN_TX_PerPack_10Msgs`,
`pack_id != 0 && pack_id < 7`), so the 7th record stays 0.

## 8. Tasks, shell, protocols

Task table `g_taskDefTable` @ 0x0805ACDC (14 tasks, unchanged), `shellCommandTable` @ 0x08062664 (81 entries, user `VNSA-0 v151` with empty
password), MQTT-RPC table `g_mqttRpcMethodTable` @ 0x0805DF00 (16 methods), mbedTLS cipher table `mbedtls_cipher_definitions` @ 0x0805E058,
CH395 sockets (502 Modbus TCP, 8091 shell, 8090 UDP).

## 9. EEPROM map

Same addresses as v150.9 and D v151. New in v151: 0x441 (cloud server type) and 0x395 (SEL LED) are loaded at boot; first boot writes 0x395 = 1.

## 10. Strings of interest

New: `VNSA-0 v151`, `Build:\tSep 28 2026 09:54:57`, the marstekcloud `getDateInfo` URL (two variants), `AT+QHTTPPOST=%d,120,120`,
`[HTTP] ERR: crc = %d, crc2 = %d, crc_data = %d, data_len = %d, d_count = %d.`, `cd=%d,pg=%d/%d,`, `AI%u=%d|%d|%d|%d|%d.`,
`] post data=%s`, `staging`, `dev`. Removed: the hamedata `getDateInfo` URLs, `AT+QHTTPPOST=%d,60,60`, the old checksum error text, `AI%d=…`.

## 11. Security notes

Unchanged: Modbus 46000 = 0x5100 → unauthenticated shell on RS485 and TCP 8091, shell user with empty password, vendor default Wi-Fi
credentials (`SM471`). `getDateInfo` still uses plain HTTP (now to marstekcloud.com). New behaviour to be aware of: after 14 400 ticks without
cloud link progress the Control reboots (`HTTP_ClientTask_Dispatcher`), as on D v151.

## 12. Processing status in Ghidra

- Coverage: 0 undefined bytes, 0 orphan instructions in the flash block; 1942 functions, all named, all with a prototype.
- `marstek_gates.py`: **all gates pass** (no code-as-data, no function pointer to non-code, no literal-only functions, no FUN_/tmp_/duplicates,
  no duplicate global labels, 0 unnamed referenced RAM globals, no DEFAULT prototypes, task names match the task table, no-return flags sane);
  short-string check: 2 hits, both genuine strings (`"%d\t"`, `"\b \b"`, same as v150.9); decompilation export 0 failures.
- Tags: `NAME_VERIFIED` on all 1942 functions, of which 99 additionally `NAME_LOW_CONFIDENCE`.
- Name consistency: function and RAM names match D v151 (the two LED-bitmap wrappers carry the corrected D names
  `Inverter_RegDirty_Mark_GridPermit` / `Inverter_RegDirty_Mark_PvVoltage`).

## 13. Open questions

1. Resolved 2026-10-11: the v150.9 analysis said `Write_Handler` was identical to D v151; one constant differs (44003 limit 1500 W instead of 2500 W)
   in v150.9 and v151. The v150.9 documents, the v150.9 Ghidra program (LED wrapper renames `Inverter_RegDirty_Mark_GridPermit`/`_PvVoltage`,
   `Write_Handler` plate) and its export have been corrected.
2. 44002 accepts up to 2500 W and forwards the raw value to the inverter while the stored value is clamped to 1500 W; the A micro limits to
   1450 W anyway (A micro analysis). Whether the reported register value then differs from the effective limit has not been measured.
3. The A v150 → v151 diff is now name-paired ([VNSA_Control_Diff_v150_v151.md](VNSA_Control_Diff_v150_v151.md); A v150 is analysed in Ghidra, see [VNSA_Control_FW_v150_Full_Analysis.md](VNSA_Control_FW_v150_Full_Analysis.md)). This point is resolved.

## Appendix: method

`segmatch.py` (offline segment map with masked pointers/branches), `vna151_layout_sync.py` (v150.9, then D v151 with protected layout),
`vna151_port_segments.py`, `vna151_copy_by_address.py`, `vna151_port_signatures.py`, `vna151_port_refs.py`, `vna151_port_flow_overrides.py`,
gap repair, `vna151_token_dump.py` + name-paired diff, `marstek_gates.py`, windowed decompilation export. Helper scripts are local only (not in the repo).
