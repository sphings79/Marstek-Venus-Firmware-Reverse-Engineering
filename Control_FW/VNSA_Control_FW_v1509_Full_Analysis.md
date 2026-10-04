# Control FW v150.9 (1509, Venus A) — Full Analysis and Differences to Venus D v151

> 🇩🇪 Deutsche Version: [VNSA_Control_FW_v1509_Vollanalyse_de.md](VNSA_Control_FW_v1509_Vollanalyse_de.md)

**Status:** completed 2026-10-02. All statements were checked against the code of this image; open points are listed in §12.
**Binary:** `1509_control_VNSA-0_ems_app_1509_0908_093804.bin` (archive `Marstek FW Archiv/firmwares/VNSA-0/Control/1509/`, beta "v150.9")
**Reference:** Venus D Control v151 (`151_control_VNSD-0_ems_app_0151_0929_094333.bin`), see [VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md](VNSD_Control_FW_Analysis_ems_app_0151_0929_094333.md)
**Ghidra:** `/VNSA/Control/Control_1509_VNSA-0_ems_app_1509_0908_093804.bin` (project `Marstek_Venus`)
**Decompilation:** `Dekompilate/VNSA_Control_v1509/` (local only, not in the repo): 1941 functions, 0 decompilation failures, + `listing/` (flash listing, `globals_and_registers.txt`)
**Modbus descriptor table:** [../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv](../Modbus_RS485_TCP/VNSA_Descriptor_Table_Decoded_v1509.csv)

---

## 0. Summary

- **Same code base as the Venus D.** 95.2 % of the image is byte-identical to D v151 once pointers and BL offsets are masked
  (282 segments). Of the 1956 D functions, 1899 are instruction-identical in the A, 34 differ in details, 8 exist at other
  addresses with identical logic and **15 are missing** (compiled out). The A has **no function of its own** that the D does not have.
- **RAM layout identical to D v151.** `.data` (12 032 B), `.bss` and the external SRAM (`.xmc_bss`) have the same sizes; all
  4526 RAM/XMC pointers in the matched code are identical; the initialised `.data` differs only in flash pointers. Every RAM
  global therefore sits at the same address as on the D (names were carried over 1:1).
- **Modbus register map identical to D v151.** The descriptor table (`g_modbusRegDescTable` @ 0x2000034C, 246 entries,
  30000–38014) is byte-identical, `Write_Handler` (holding registers) and the read path are instruction-identical.
  Only what feeds the RAM variables differs (the inverter of the A is a different device, see §5).
- **The device differences are compile-time constants**, not separate code paths: power limits (1500 W instead of 2500 W),
  power classes (800/1200/1500 W), 13 cells per battery pack, RS485 port on USART1 without driver-enable pin,
  no key/protection monitor, identity strings `VNSA`. The CAN code (frame building, dispatcher, all inverter commands
  incl. 0xFB/0xFE/0xFF, 0x18, 0x30–0x3E) is identical to the D; there is no A-specific CAN function in the Control.
- **Older cloud generation than D v151:** `getDateInfo` still goes to `http://<region>.hamedata.com/...` (like D v150),
  only 3 cloud server types are accepted, CH395 keepalive idle 10 s, older HTTP-POST and TOU-parser variants.

## 0a. Checklist (A 1509 compared with D 151)

| # | Question | Answer A 1509 | Evidence |
|---|---|---|---|
| 1 | Modbus registers different? | **No.** Descriptor table byte-identical (246 entries incl. 32102 `i16`); `Write_Handler` @ 0x08056960 instruction-identical (incl. 46000 = 0x5100 debug backdoor, 47001, 45000 block). Register 30200 reads the version word @ 0x08010000 = **1509**. | §5 |
| 2 | BLE different? | Device name `MST_VNSA_` + 4 ID bytes. Cmd 0x15 (power class): 800 W → class 1, **1200 W → class 4, 1500 W → class 5** (D: 800 → 1, 1500/2200 → 3, 2500 → 0). Cmd 0x16 max charge power 300–**1500** W, cmd 0x17 max discharge power ≤ **1500** W. | §4.1 |
| 3 | AC limits? | Max charge/discharge power clamped to **1500 W** everywhere (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock`, MQTT/JSON-RPC, remote command ±1500 W). Grid export limiter upper clamp **2400 W** (D: 4000 W). | §4.1 |
| 4 | Firmware update via BLE possible? | Yes, same OTA code as D 151; the model checks compare against `VNSA-0` (`OTA_Is_VNSA_Model`, `OTA_Slot_Config_Validate`). | §4.6 |
| 5 | Cloud domains? | Upload `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>` (like D). `getDateInfo` **`http://<region>.hamedata.com/app/neng/getDateInfo<suffix>.php`** (D 151: `api-<region>.marstekcloud.com`). Still hamedata: backup-configs, schedulings/policies, getDevicesApply, FC41D OTA. | §4.5 |

## 1. Binary fingerprint

| | A 1509 | D 151 (reference) |
|---|---|---|
| Size | 391 168 B (0x5F800) | 393 216 B |
| SHA-256 | `a8c58127189e19937cf179195e714492f890a6a829b03b8e218380042f8d9f23` | — |
| Link address | **0x08004800**–0x08063FFF (18 KB bootloader below) | 0x08004800–0x080647FF |
| MCU | Nations **N32G45x** with XFMC (Cortex-M4F), 144 KB SRAM | same |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW) | same |
| Build string | `Sep  8 2026 09:34:19` | `Sep 29 2026 09:42:27` |
| Version | string `VNSA-0 v1509` @ 0x0805C239; version word 0x5E5 (1509) @ 0x08010000 followed by `VNSA-0` | 0x97 (151), `VNSD-0` |
| Image footer | `VNSA` + 12 zero bytes @ 0x08063FE0, before it 0xFF fill from 0x080638C8 | `VNSD` |
| Release note | "PV dropout during surplus feed-in fixed, LED blinking fixed, CT connection/phase detection improved" | — |

Base proof: the vector table at 0x08004800 resolves exactly like on the D (Reset = Keil pattern, `SystemInit` @ 0x08032C68
writes `SCB->VTOR = 0x08004800`); the scatter table, string and function pointers all resolve at this base, and the
flash pointers in `.data` are those of D 151 shifted by the code size differences.

## 2. Memory map (Ghidra)

| Block | Range | Contents |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | not in the image; 0x08001000 bootloader version word, 0x08004000 update descriptor page |
| flash | 0x08004800–0x08063FFF | application |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-bit UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EFF | RW initial values (LZ77 @ 0x08062A34, 2486 B) |
| .bss | 0x20002F00–0x2002200F | zero-initialised; `ucHeap` @ 0x20003144 (80 KB) |
| sram_unused | 0x20022010–0x20023FFF | up to the N32G45x 144 KB limit |
| .xmc_bss | 0x60000000–0x60026AF7 | external SRAM via XFMC (LZ77 image @ 0x080633EC, 1244 B) |
| peripherals | apb1/apb2/ahb/bit-band/ppb | register names from the Nations N32G457 SVD (copied 1:1 from D 151) |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC and QSPI controller |

Scatter table `Region$$Table$$Base` @ 0x080624F4: .data 0x08062A34 → 0x20000000 (0x2F00, `__decompress` @ 0x08005FBE),
XMC 0x080633EC → 0x60000000 (0x26AF8), .bss zero-init 0x1F110 @ 0x20002F00.

## 3. Relationship to Venus D v151 (method and numbers)

| Category | Functions | Meaning |
|---|---|---|
| instruction-identical | 1899 | same bytes apart from call/pointer targets; names, prototypes, plates, locals taken from D 151 |
| changed | 34 | same function, different constants or small code changes (§4) |
| relocated | 8 | identical logic at another address: `Factory_Reset`, `RTC_Peripheral_Init`, `RS485_UART_Config_Init` (A variant), `RS485_UART_SendByte` (A variant), `AI_ScheduleQuery_Response_Send` (older variant), `Stub_Return_One`, `MQTT_Transport_NullCallback`, `Util_NoOp_StartupHook` |
| missing in A | 15 | see §4.4 |
| only in A | 0 | — |

Cross-check: for every function pair the sequence of call-target names is identical, except for the calls to the
missing functions. Comparison method: offline byte segments with masked pointers/BL offsets, then name-normalised
instruction diffs of the 34 changed functions.

## 4. Functional differences to Venus D v151

### 4.1 Power limits and power classes

| Place | A 1509 | D 151 |
|---|---|---|
| `Config_Write_U16_0x202` (max charge power, EEPROM 0x202) | clamp 1500 W | 2500 W |
| `Config_Write_U16` (max discharge power, EEPROM 0x204) | clamp 1500 W | 2500 W |
| `Config_ClampDefaults_ParamBlock` | charge 0/>1500 → 1500 W; discharge 0/>1500 → 800 W | 2500 W |
| `Config_LoadOrInit_DeviceSettings` first-boot default max charge | 1500 W | 2500 W |
| BLE cmd 0x15 power class | 800 → 1, 1200 → 4, 1500 → 5 | 800 → 1, 1500/2200 → 3, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–1500 W / ≤ 1500 W | 300–2500 W / ≤ 2500 W |
| MQTT `md=` default, `MQTT_Config_Command_Handler` | 1500 W | 2500 W |
| `MQTT_JSON_RPC_Dispatcher` passive setpoint clamp | ±1500 W | ±2500 W |
| `Remote_Power_Command_Execute` | ±1500 W | ±2500 W |
| `Grid_Export_Power_Limiter` upper clamp | 2400 W | 4000 W |

`Write_Handler` itself is unchanged, but it calls the two `Config_Write_U16*` functions, so Modbus writes of the power limits
are also capped at 1500 W on the A.

### 4.2 Battery

`BMS_CellVoltage_MinMax_PerPack` scans **13 cells** per pack (D: 16). The per-pack records keep the D layout (room for 16 cells).

### 4.3 RS485 / serial port

The A uses **USART1 (PA9 TX / PA10 RX, IRQ 37, 115200 8N1)** without a driver-enable pin: `RS485_UART_Config_Init` (A variant),
`RS485_UART_Send`, `RS485_UART_SendByte`, `Protocol_PutChar`, `USART1_IRQHandler`. The D uses USART2 on remapped PC8/PC9 plus
`RS485_DE_Pin_Init`/`RS485_Set_TX_Mode`/`RS485_Set_RX_Mode`. Modbus RTU, the debug shell on RS485 and YMODEM updates are otherwise identical.

### 4.4 Functions missing in the A (compiled out)

| D 151 function | Purpose on the D |
|---|---|
| `Key_ProtectionMonitor` | physical key on GPIOB.11, debounce and protection escalation |
| `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`, `Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot` | relay/latch pulse chain on GPIOB.10 / GPIOE.15 with CAN frame and reboot on persistent fault |
| `CAN_ParallelInvLink_DebounceHold` | debounce of the parallel-inverter link flag (only called by the key monitor) |
| `Board_GPIO_Init_ControlPins` | GPIOB.10/.11 and GPIOE.15 outputs |
| `RS485_DE_Pin_Init`, `RS485_Set_TX_Mode`, `RS485_Set_RX_Mode` | RS485 direction pin |
| `MQTT_CtrlResponse_Transmit` | helper for paginated 'ctrl' responses (the A sends the cd=58 answer inline, without pagination) |

`WDT_SystemHealthService` therefore only runs the 500 ms board tick. The RAM variables of the missing code still exist (identical
`.bss`), they are simply unused on the A.

### 4.5 Cloud / HTTP / network

| Function | A 1509 | D 151 |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://<g_cloudDomainUrlTable[t]>.hamedata.com/app/neng/getDateInfo<g_cloudGetDateInfoSuffixTable[t]>.php`, server types 0–2 | `api-<region>.marstekcloud.com`, types 0–4 |
| region table @ 0x20000FC8 | `eu, www, us, eu-dev, eu-staging` + suffix table @ 0x20000FDC | `eu, www, us, dev, staging` |
| `Config_LoadOrInit_DeviceSettings` | does not load EEPROM 0x441 at boot, no first-boot default for 0x395 | loads 0x441, writes 0x395 default |
| `Quectel_HTTP_POST_SendAndVerify` | older variant (no ERROR/+QIURC check, no post-data log) | newer variant |
| `HTTP_Task_StateMachine_Handler` | `AT+QHTTPPOST=%d,60,60`, retry period 600 ticks | 120/120 s, 6000 ticks |
| `HTTP_Economy_TOU_ParseResponse` | single CRC | additional `crc2` |
| `HTTP_ClientTask_Dispatcher` | refresh 3600 s | 14400 s |
| `CH395_ConfigTcpKeepalive_Defaults` | idle 10 000 ms | 60 000 ms |
| `Modem_/Ethernet_SendStatusTelemetry` | every 2000 ticks | 1100 ticks |
| `CT_GridPower_Controller` | EPS reserve active → setpoint update skipped | additionally forces a positive setpoint to 0 |

### 4.6 Identity and other details

- Strings: `VNSA-0`, `VNSA-0 v1509`, `MST_VNSA_`, `Venus A`, device source id `VNSA-0-<serial>`, type code 0x5E5 (`g_deviceTypeCode`).
- `OTA_Is_VNSA_Model` (D: `OTA_Is_VNSD_Model`) and `OTA_Slot_Config_Validate` accept `VNSA-0` images only.
- `Inverter_Default_Init_Once` only sends `Inverter_Set_Flag_Reg_0x05(0)` (D additionally `Inverter_Set_Reg_0x04(0)`).
- `CAN_Broadcast_DateTime`: second 0xCC frame with DLC 8 (D: 7).
- `Grid_Export_Power_Limiter`, maintenance mode: acts on old-step 1 or 5 and sends `Inverter_CAN_PowerCmd_Send` with mode **3** (D: step 5, mode 1).

## 5. Modbus

- **Read range:** `g_modbusRegDescTable` @ 0x2000034C, 246 × 12 B, byte-identical to D 151 (decoded CSV linked above; the `notes`
  column names the RAM global in the A program). `FC03_Read_Handler` @ 0x080232F4 and `Read_Serializer` @ 0x08055860 are
  instruction-identical, so the known D bugs (32101 sign, MBAP length of exception frames) apply to the A as well.
- **Write range:** `Write_Handler` @ 0x08056960 instruction-identical (same callees and constants). The 45000 command block,
  46000 = 0x5100 (debug mode → shell on RS485 and TCP 8091), 46500–46544 schedule (CAN 0x30–0x3E), 47001 (CAN 0x18) behave as on the D.
- **Consequence for unknown A registers:** a register reads the same RAM variable as on the D. The variables of the
  inverter/MPPT block (e.g. 30000–30040, 32100–32302, 35000–35112, 36000–36103) are filled by the identical CAN parsers from
  frames sent by the inverter (and MPPT). Their meaning on the A is defined by what the **A micro** sends in those frames;
  this has to be resolved from the A micro firmware (separate analysis), not from the Control.

## 6. Internal CAN bus (Control side)

Arbitration ID: `CAN_Build_Arbitration_ID` @ 0x08037678 (function code bits 0–7, node 8–11, target class 12–15).
All 57 send sites are identical to D 151 (only the DLC of the 0xCC date frame differs):

| Target class | Function codes sent by the Control (A 1509 = D 151) |
|---|---|
| **4** (inverter, D/E3-type micro) | 0x01 power setpoint, 0x02 `Config_Mode_Apply`, 0x03/0x04 u32 writes, 0x05, 0x06, 0x07, 0x08, 0x0A relay, 0x10, 0x11, 0x12, 0x13, 0x14, 0x16, 0x17, **0x18**, **0x30–0x3E** schedule, 0x40–0x43, 0x50–0x58, 0x60, 0xC1, 0xC2, 0xCB |
| **2** (MPPT / other inverter family; values cached in `g_mppt*`) | 0x01 on/off (`Inverter_Set_OnOff_Reg_0x01`), 0x02 power command (`Inverter_CAN_PowerCmd_Send`, 2 frames), 0x03 clear, 0x04, 0x06 work mode, **0xFB, 0xFE, 0xFF**, 0xCB clear, 0xCC date/time |
| **3** (BMS) | 0x0A, 0xC1, 0xC2, 0xC4, 0xCF |
| 1 / variable | 0xC2 `Debug_Mode_Set`, 0xC3, 0xCE |

Callers of the class-2 commands on the A: `Write_Handler` (45020–45024: on/off + power, 0xFB, 0xFE, 0xFF), `Grid_Export_Power_Limiter`
(power command, mode 3 on the A), `Inverter_SendOnCommand`/`Inverter_StopOutput`/`Idle_Watchdog_AutoReboot` (on/off),
`Timer_Callback_CAN_DateTimeBroadcast` (0xCC), shell `test`/`set_old`. Receive side: `CAN_FrameDispatcher` @ 0x080338E8 sends
frames with source class 4 to `Telemetry_Register_Dispatcher`, source class 2 to `Modbus_ResponseDispatch`, everything else to
`Protocol_AA_CommandDispatch`. **Which of these frames the A micro actually consumes cannot be decided from the Control**;
the D/E3 micro filters class 2 and 3, the A micro has a 0xFE handler (earlier finding), the complete list needs the A micro analysis.

## 7. Tasks, shell, protocols

Identical to D 151 apart from addresses: task table `g_taskDefTable` @ 0x0805ABC0 (14 tasks, same priorities/stacks),
`shellCommandTable` @ 0x08062524 (81 entries, user `VNSA-0 v1509` with empty password), MQTT-RPC table `g_mqttRpcMethodTable`
@ 0x0805DDE8 (16 methods), mbedTLS cipher table @ 0x0805DF40, CH395 sockets (502 Modbus TCP, 8091 shell, 8090 UDP).

## 8. EEPROM map

Same addresses as D 151 (see the D analysis). Differences: no first-boot default for 0x395 (SEL LED; reading/writing it via
BLE/MQTT is present), 0x441 (cloud server type) is read on demand only, not at boot. Default limits written on first boot:
max charge power 1500 W.

## 9. Strings of interest

`VNSA-0 v1509`, `MST_VNSA_`, `Venus A`, `Build:\tSep  8 2026 09:34:19`, the hamedata `getDateInfo` URL (two variants),
`AT+QHTTPPOST=%d,60,60`, `[HTTP] ERR: crc = %d, crc_data = %d, data_len = %d, d_count = %d.`, UTF-8 log strings
`[Http Tx←★★★] %s` / `[Http Rx→☆☆☆] %s` in `Quectel_HTTP_POST_SendAndVerify`. Default Wi-Fi credentials of the vendor
(`SM471`, …) and `quit_debug_5100_` are present as on the D.

## 10. Security notes

All Control findings of D 151 apply unchanged (identical code): Modbus 46000 = 0x5100 → unauthenticated shell on RS485 and
TCP 8091, shell user with empty password, vendor default Wi-Fi credentials, plain-HTTP `getDateInfo` (on the A to hamedata.com).

## 11. Processing status in Ghidra

- Coverage: 0 undefined bytes, 0 orphan instructions in the flash block; 1941 functions, all named, all with a prototype
  (task entries `void f(void *pvParameters)`).
- `marstek_gates.py`: **all gates pass** (no code-as-data, no literal-only functions, no FUN_/tmp_/duplicates, 0 unnamed referenced
  RAM globals, task names match the task table, no-return flags sane); decompilation export 0 failures, no unrecovered jump tables.
- Tags: `NAME_VERIFIED` on all 1941 functions (comment "Name, prototype and comment verified against the VNSA Control 1509 code"),
  of which 99 additionally `NAME_LOW_CONFIDENCE` (carried over from D where the purpose is not fully clear).
- RAM globals, peripheral registers and their comments were copied from D 151 by address (identical layout); comments with D-specific
  content (VNSD strings, 2500 W, USART2, EEPROM 0x441, D flash addresses) were corrected for the A.

## 12. Open questions

1. Which CAN frames does the A micro (and a possible MPPT node on class 2) send and consume? This decides the meaning of the
   inverter registers on the A (30000–30040, 32100–32302, 35000–35112, 36000–36103) and whether 45020–45024 / 0xFB/0xFE/0xFF act on the A.
2. Power class values 4 (1200 W) and 5 (1500 W): how the A micro interprets them (it receives the class via the work-mode/limit commands).
3. The A still ships the older cloud/HTTP code (hamedata `getDateInfo`, 3 server types) — whether a later A release adopts the D 151 path is unknown.

## Appendix: method

Offline segment map of the two images (masked pointers and BL offsets), `vna_layout_sync.py` / `vna_port_segments.py` /
`vna_port_signatures.py` / `vna_copy_by_address.py` / `vna_port_refs.py` to carry over layout, names, prototypes, globals, references
and flow overrides from D 151, then manual analysis of the 34 changed and 8 relocated functions, gap typing and the verification
gates. Helper scripts are local only (not in the repo).
