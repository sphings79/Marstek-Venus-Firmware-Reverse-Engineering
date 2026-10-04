# Marstek Venus D — Micro/Inverter Co-Processor Firmware Analysis

> 🇩🇪 Deutsche Version: [VNSD_Micro_Inverter_FW_Analyse_vd_inv_app_0116_de.md](VNSD_Micro_Inverter_FW_Analyse_vd_inv_app_0116_de.md)

> **Outdated (2026-09-30), will be replaced by a re-analysis.** This version is based on the wrong image base
> 0x08000000: **all flash addresses here +0x4000** (real address = quoted address + 0x4000). The MCU is an
> **STM32G474**, not an STM32F3. The "thunks to the bootloader (0x10000xxx)" are veneers into the CCM SRAM
> (0x10000000), to which the startup code copies 27 KB of control code; this code was missing from the old analysis or lay
> at its flash load address 0x080190D4+x. Function names are not verified.
## `vd_inv_app_0116_0702_ota_163439.bin`

**Firmware:** v116 (OTA `firmwareType: micro`)  
**Analysis date:** 04.07.2026 (initial capture), **10.07.2026** (correction session, tranche 2 + tranche 3a/3b/3c + tranche 4a/4b/4c + tranche 5a/5b/5c)  
**Method:** Static analysis (Ghidra + ReVa MCP)  
**Status:** CAN protocol, SRAM map, debug shell, RS485 Modbus register map, error codes, grid standards, EEPROM map decoded.
**Function naming:** 392/445 functions actually named in Ghidra (88.1%), status 10.07.2026 after completion of tranche 5a+5b+5c (independently verified via `get-function-count filterDefaultNames=true/false` + a complete duplicate scan over all 392 names, 0 collisions). 53 functions are still `FUN_*` — of these 5 confirmed Ghidra interworking veneers/jump trampolines (10 bytes, no real standalone functions, see 13.18/13.21), an unreferenced "dead code" function family around 0x08002824/0x08004528 (0 callers, instance addresses outside the official STM32F3 peripheral map), an HRTIM cluster at 0x080090f8 (also 0 callers) and a remaining GPIO/UART/ADC bit-field cluster (0x08009a40–0x08009a92, compiler identical code folding across peripheral domains), the rest are pure 2-16-byte stubs/padding without reconstructable semantics. These remaining 53 are considered statically exhausted — further tranches would presumably yield no additional hits.

> ⚠️ **Important note (10.07.2026):** The rate "398/445 (89%)" claimed in an earlier version of this documentation
> from the "mass analysis of 07.07.2026 via 5 parallel Ghidra agents" was **wrong**
> or was **never checked in to Ghidra**. When the project was opened on 10.07.2026, only
> **21 functions** were actually named in Ghidra (the rest still `FUN_0800xxxx`). The names in section 13 below
> came from an analysis that was never persisted. In this session **106 of the names listed in section 13
> were individually verified by decompilation and renamed in Ghidra** (see note box in
> section 13). Some of the originally documented names turned out to be **wrong** (address drift,
> presumably from the never-verified 07.07 analysis) and were deliberately **not** adopted or were corrected —
> see section 13.9 "Correction session 10.07.2026".

---

## 1. Binary fingerprint

Verified live from Ghidra (status 2026-07-15). A comparison with the other five analyzed
firmware images (Control 149.2/147, VNS 115, BMS 118/117.7) is in the project `README.md`.

| Property | Value |
|---|---|
| File | `vd_inv_app_0116_0702_ota_163439.bin` |
| Version | 116 (current) |
| Size | 115,712 B (0x1C400, 113 KB) |
| Architecture | ARM Cortex-M4F, Thumb-2, little-endian |
| Flash range | `0x08000000–0x0801C3FF` |
| Initial SP | `0x20009A70` (~40 KB SRAM) |
| Reset handler | `0x080042AD` |
| Functions | 392 / 445 named (88.1 %) |
| Strings | 251 |
| Compiler | RVDS/Keil ARM |
| RTOS | FreeRTOS (`heap_4`, `ARM_CM4F` port, FPU active) |
| Crypto | — (no mbedTLS) |
| WiFi/TCP | — (not present) |
| Communication | CAN + RS485 |

**Key differences from the Control FW:**
- No WiFi stack, no TCP/IP, no Modbus TCP — the micro MCU communicates exclusively via CAN bus and RS485/UART
- No mbedTLS — no encryption, no cloud connectivity
- Significantly smaller (~30% of the Control FW)
- Uses the RVDS/Keil compiler instead of GCC
- Contains its own bootloader thunk area (`0x10000xxx`)

---

## 2. Memory layout

```
Flash (0x08000000 – 0x0801C3FF):
  0x08000000 – 0x080001FF   IVT (interrupt vector table)
  0x08000200 – 0x080009FF   Thunks to bootloader (0x10000xxx)
  0x08000A00 – 0x0800B3FF   Peripheral drivers (CAN, UART, ADC, GPIO, flash, PWM)
  0x0800B400 – 0x0800B5FF   FreeRTOS task creation
  0x0800B600 – 0x0800C8FF   Firmware update (UART Ymodem + CAN)
  0x0800C900 – 0x0800D7FF   Debug/diagnostic output (sensor dump)
  0x0800D800 – 0x0800DFFF   Utility functions
  0x0800E000 – 0x0800E9FF   Hex conversion, FreeRTOS assert handler
  0x0800EA00 – 0x0800EDFF   Initialization, CAN filter setup
  0x0800EE00 – 0x08014FFF   Inverter control (FP-intensive)
  0x08015000 – 0x0801C3FF   FreeRTOS kernel + task loop code

SRAM regions (identified):
  0x20000000 – 0x200003FF   Task handles, configuration, calibration data
  0x20000400 – 0x200004FF   Operating parameters, flags, modes
  0x20000D00 – 0x20000EFF   ADC measured values (raw + calibrated)
  0x20001400 – 0x200019FF   Debug counters, error log, energy counters
  0x20003400 – 0x200038FF   FW update buffer, CAN receive buffer
  0x20003900 – 0x200039FF   BMS data (received via CAN)
  0x20003D00 – 0x20003EFF   Operating modes, configuration
```

---

## 3. FreeRTOS task architecture

`FUN_0800b420` = task creator (not yet renamed in Ghidra, outside the tranche 2c range), `0x08013998` = `xTaskCreate` (named in Ghidra)

| Task | Priority | Stack | Entry point | Function |
|---|---|---|---|---|
| `vtask_led` | 16 (highest) | 512 B | `0x080163E8` | LED status indicator |
| `vtask_time` | 15 | 1 KB | `0x080164C8` | Timer/RTC |
| `vtask_modbus` | 14 | 512 B | `0x08016424` | RS485 Modbus communication |
| `vtask_shell` | 9 | 2 KB | `0x0801643C` | Debug CLI (UART) |
| `vtask_prt` | 8 | 1 KB | `0x08016430` | Debug output |
| `vtask_can_receive` | 7 | 1 KB | `0x080163E0` | CAN receive |
| `vtask_can` | 6 | 1 KB | `0x080163D0` | CAN transmit |
| `IDLE` | 0 | 512 B | `0x08012FD0` | FreeRTOS idle |

**Note:** The task entry points from `0x08016xxx` lie in a contiguous FP code block for the inverter control. Ghidra has difficulty with function boundary detection there, since the control loops are heavily intertwined.

---

## 4. Communication architecture

### 4.1 CAN bus (main communication)

CAN peripheral: `0x40006400` (CAN1, STM32F3 layout — bxCAN, same base address as on F4)

**CAN init/HAL layer (tranche 3a, identified 10.07.2026):** `CAN1_Init` (`0x08001ea0`) builds the
CAN handle (instance=`0x40006400`, timing parameters) and calls `HAL_CAN_Init` (`0x08004fe0`,
classic bxCAN init state machine: set INRQ → wait for INAK (timeout via `HAL_GetTick`) →
configure MCR/BTR/mode → `CAN_InitMailboxes` → state=READY). Runtime functions:
`HAL_CAN_AddTxMessage` (`0x08004c5c`, selects a free TX mailbox via the TSR code field, calls `CAN_CopyTxFrame`),
`HAL_CAN_ConfigFilter` (`0x08004caa`, common filter bank write function, called from
`CAN_SetFilter_ExtID`/`CAN_SetFilter_StdID`), `HAL_CAN_GetRxMessage` (`0x08004d70`, decodes the
FIFO0/FIFO1 mailbox: IDE/RTR/ID/DLC/timestamp/FilterMatchIndex + data copy), `HAL_CAN_IRQHandler`
(`0x08004e76`, central ISR — dispatches TME/FMP0/FMP1/FF/FOV/EWG/EPV/BOF/LEC/ERR based on SR&IER masks
to the respective sub-handlers, calls the error callback at the end if an error code is set).

**Addition tranche 4b (10.07.2026):** `HAL_CAN_DeInit` (`0x08004d3e`, called as the very first step in
`CAN1_Init`, resets state/ErrorCode to `0`/RESET, calls `CAN_ExitInitMode_WaitAck` +
a no-op Msp-DeInit stub) and `HAL_CAN_ActivateNotification` (`0x08004b7c`, state guard
`READY|LISTENING` identical to the real HAL function, enables IER bits with spurious IRQ avoidance;
callers: `CAN1_Init` [TX mailbox empty] and `HAL_CAN_RxFifo0MsgPendingCallback` [re-arm after
message processing]) further complete the CAN HAL layer.

**CAN filters (configured in `CAN_Filter_Setup`, `0x0800eb88`):**

| Filter | ID | Mask | Matches | Protocol |
|---|---|---|---|---|
| 1 | `0x4000` | `0xFF00` | `0x40xx` | EMS→inverter commands |
| 2 | `0x4100` | `0xFF00` | `0x41xx` | Inverter→EMS responses |
| 3 | `0x1801AA01` | `0x1FF0FFFF` | `0x180xAA01` | BMS CAN protocol |

**Addition tranche 4a (10.07.2026):** `HAL_CAN_RxFifo0MsgPendingCallback` (`0x08005248`) additionally
dispatches on `FilterMatchIndex==4` to `BMS_FW_Update_CAN_Handler` (`0x08001868`) — a
PF-byte-based handler analogous to `BMS_CAN_Parser`, which writes init/verify states (values `2`/`4`) to
SRAM `0x200038cb`–`0x200038d0` and thus presumably serves the "BMS update via CAN" path
(CMD `0xCE`) documented in section 9. The exact ID/mask configuration of this 4th filter
was not extracted separately from `CAN_Filter_Setup` (open for a future tranche).

### 4.2 BMS CAN protocol (extended frame, 29-bit)

Parser: `FUN_080018e8` — the PF byte (bits 16–19 of the CAN ID) determines the message type.

| CAN ID | PF | Byte 0–1 | Byte 2–3 | Byte 4–5 | Byte 6–7 |
|---|---|---|---|---|---|
| `0x1801AA01` | 1 | bat_vol (u16) | bat_cur (i16) | max_temp (i16) | soc (u16) |
| `0x1802AA01` | 2 | cap (u16) | ? | ? | ? |
| `0x1803AA01` | 3 | charge_u (u16) | charge_i (u16) | discharge_i (u16) | ? |
| `0x1804AA01` | 4 | err (u32) | warn (u32) — **see note below** |

**SRAM mapping of the BMS data:**

| SRAM address | Variable | Type | Contents |
|---|---|---|---|
| `0x2000397B` | bat_vol | uint16 | BMS battery voltage |
| `0x2000397D` | bat_cur | int16 | BMS battery current |
| `0x2000397F` | max_temp | int16 | BMS max temperature |
| `0x20003981` | soc | uint16 | State of charge |
| `0x20003983` | cap | uint16 | Capacity |
| `0x2000398A` | sleep_flag | byte | BMS sleep request |
| `0x2000398B` | charge_u | uint16 | Charge voltage setpoint |
| `0x2000398D` | charge_i | uint16 | Charge current limit |
| `0x2000398F` | discharge_i | uint16 | Discharge current limit |
| `0x20003991` | charge_req | byte | Charge request (flag) |
| `0x20003992` | force_charge_req | byte | Forced charging (flag) |
| `0x20003993` | err | uint32 | BMS error bitmask |
| `0x20003997` | warn | uint32 | BMS warning bitmask |

> **Clarification (2026-07-10, verified on both sides via Ghidra, no contradiction to `BMS/VNSD_BMS_FW_Analysis_v117.7.md`):**
> `BMS_CAN_Parser` (`0x080018e8`) for PF=4 really only takes over two raw 32-bit values
> (`_bms_error_bitmask = *param_2; _bms_warn_bitmask = param_2[1];`), without decomposing them further.
> On the sender side (`CAN_TX_PF4_ProtectWarnings`, BMS FW `0x080055d4`), however, these 8 bytes are
> structured: byte 0–1 = `protect1` (u16), byte 2–3 = `protect2` (u16), byte 4–5 = reserved (always 0),
> byte 6 = `warn_byte`, byte 7 = `status_flag`. The 4 bytes read here as "err" are thus
> `protect1 | (protect2 << 16)`, and "warn" is `(status_flag << 24) | (warn_byte << 16)` (lower 16 bits
> always 0). Both descriptions are correct — the micro FW currently reads the fields only as
> undecomposed raw 32-bit values, while the BMS FW side (sender) shows the fine-grained meaning.
> For the actual bit meaning of protect1/protect2 see `BMS/VNSD_BMS_FW_Analysis_v117.7.md` §5.2/5.3.

### 4.3 EMS↔inverter internal CAN protocol (standard frame, 11-bit)

Dispatcher: `FUN_08001940` (1,168 bytes, 269 lines — largest function)

CAN ID format: `0x40xx` (EMS→INV) / `0x41xx` (INV→EMS), where `xx` = command ID

#### Commands EMS → inverter:

| CMD | Hex | Description | Data |
|---|---|---|---|
| Set Power | 0x01 | Set power setpoint | int→float conversion |
| Enable/Disable | 0x02 | Inverter on/off | 0=off, 1=on |
| Max Discharge Power | 0x03 | Max. discharge power | Limit (capped 2500W) |
| Max Charge Power | 0x04 | Max. charge power | Limit (capped 2500W) |
| Unknown Flag | 0x05 | Set internal flag | DAT_2000043c=1 |
| Sleep Mode | 0x06 | Sleep mode | DAT_200004a4=1 |
| Address Assign | 0x07 | Assign CAN address | Address+1 |
| Config Block | 0x16 | Configuration data | 8 bytes → 0x20003EA7 |
| Factory Test Enter | 0x50 | Factory test mode | 0=exit, 1=enter |
| FW Update Trigger | 0x51 | Start firmware update | — |
| FW Update Phase | 0x52 | Set update phase | 1/2/3 → step 2/3/4 |
| FW Update Complete | 0x53 | Complete update | Step=5 |
| FW Update Finalize | 0x55 | Finalize update | Step=6 |
| Set Power (Factory) | 0x56 | Power in factory mode | Power value |
| Backup Mode | 0x57 | Backup (EPS) power on/off | 1=on, 0=off |
| Set Power (Battery) | 0x58 | Power in battery mode | Power value |
| Start FW Update | 0x60 | Initiate FW update | Update type |
| FW Update Sequence | 0xC1 | Start update sequence | Step=1 |
| Debug Mode | 0xC2 | Debug on/off | 1→level 2, else→1 |
| CAN Queue Data | 0xC3 | Forward data to CAN | up to 8 bytes |
| Flash Program | 0xCE | Flash programming | Type 0=init, 1=data, 2=verify |

#### Responses inverter → EMS:

| CMD | Hex | Description | Data |
|---|---|---|---|
| Data Block | 0x10 | 48 bytes operating data | from 0x200038E8 |
| Status | 0x11 | 8 bytes status info | from 0x20000440 |
| Version Info | 0x12 | FW version data | 6 bytes |
| Calibration | 0x13 | Calibration data | 4 bytes |
| Flash ID | 0x54 | Flash chip identification | — |
| Version/Flash | 0xCB | Extended version/flash info | 8 bytes |

**Key findings:**
- **Max power cap 2500W** (0x9C4) — confirms Venus D hardware
- `DAT_20003E04` = work_mode: `0x00`=normal, `0x01`=test, `0x04`=update, `0x0C`=factory
- `DAT_2000043E` = firmware update step counter (1–7)
- `DAT_200004AD` = max_discharge_power, `DAT_200004AB` = max_charge_power (both ≤2500)

---

## 5. ADC sensor map (inverter measured values)

Extracted from debug function `FUN_0800cc40`.

### 5.1 Raw ADC values (int16, SRAM `0x20000E44`–`0x20000E62`)

| SRAM | Variable | Type | Description |
|---|---|---|---|
| `0x20000E44` | grid_vol | int16 | Grid voltage (raw ADC) |
| `0x20000E46` | offgrid_vol | int16 | Backup (EPS) power voltage (raw) |
| `0x20000E48` | out_vol | int16 | Output voltage (raw) |
| `0x20000E4A` | grid_vref1 | int16 | Grid reference 1 |
| `0x20000E4C` | grid_vref3 | int16 | Grid reference 3 |
| `0x20000E4E` | grid_vref4 | int16 | Grid reference 4 |
| `0x20000E50` | grid_cur | int16 | Grid current (raw ADC) |
| `0x20000E52` | offgrid_cur | int16 | Backup (EPS) power current (raw) |
| `0x20000E54` | highv_vol | int16 | DC bus voltage (raw) |
| `0x20000E56` | bat_cur | int16 | Battery current (raw) |
| `0x20000E58` | bat_cur_df | int16 | Battery current diff (raw) |
| `0x20000E5A` | bat_vol | int16 | Battery voltage (raw) |
| `0x20000E5C` | vac_offset | int16 | AC voltage offset |
| `0x20000E5E` | iac_offset | int16 | AC current offset |
| `0x20000E60` | ntc_rad1 | int16 | NTC radiator 1 (raw) |
| `0x20000E62` | ntc_rad2 | int16 | NTC radiator 2 (raw) |

### 5.2 Calibrated float values (SRAM `0x20000DE0`–`0x20000E34`)

| SRAM | Variable | Type | Description |
|---|---|---|---|
| `0x20000DE0` | grid_vol | float | Grid voltage (V, calibrated) |
| `0x20000DE4` | grid_cur | float | Grid current (A, calibrated) |
| `0x20000DEC` | highv_vol | float | DC bus voltage (V) |
| `0x20000DF0` | out_vol | float | Output voltage (V) |
| `0x20000DF4` | bat_vol | float | Battery voltage (V) |
| `0x20000DF8` | bat_cur | float | Battery current (A) |
| `0x20000DFC` | bat_cur_df | float | Battery current diff (A) |
| `0x20000E04` | offgrid_cur | float | Backup (EPS) power current (A) |
| `0x20000E08` | offgrid_vol | float | Backup (EPS) power voltage (V) |
| `0x20000E0C` | highv_notch_vol | float | DC bus notch-filtered (V) |
| `0x20000E10` | bat_vol_avg | float | Battery voltage average |
| `0x20000E14` | bat_cur_avg | float | Battery current average |
| `0x20000E20` | vac_offset | float | AC voltage offset (cal.) |
| `0x20000E24` | iac_offset | float | AC current offset (cal.) |
| `0x20000E2C` | ntc_temp | float | Temperature general (°C) |
| `0x20000E30` | ntc_inv | float | Inverter temperature (°C) |
| `0x20000E34` | ntc_mppt | float | MPPT temperature (°C) |

### 5.3 Other calibration/status values

| SRAM | Variable | Description |
|---|---|---|
| `0x20000338` | g_offset_vol | Global voltage offset (float) |
| `0x20000344` | real_offset_vol | Real voltage offset (float) |
| `0x200002A4` | inverter_run_state | Inverter state machine |
| `0x200002A5` | llc_run_state | LLC converter state |
| `0x200002A8` | ctl_state | Main control state |
| `0x200004A6` | bat_mode | Battery mode |
| `0x200004AF` | grid_standand | Grid standard code |
| `0x2000030C` | max_power | Max. power (float) |
| `0x20000310` | min_power | Min. power (float) |
| `0x200019F4` | err1 | Error register 1 (hex) |
| `0x200019F8` | err2 | Error register 2 (hex) |
| `0x200019FC` | war1 | Warning register (hex) |

---

## 6. Hardware IO map (GPIO)

Extracted from debug strings in the IO show/test functions:

| GPIO designator | Function | Description |
|---|---|---|
| `LED0` | Status LED | Operating/fault LED |
| `IO_RELAY_GRID` | Grid relay | Grid contact (grid connection) |
| `IO_RELAY_OFFGRID` | Backup (EPS) power relay | Off-grid contact (UPS) |
| `IO_FAULT_RESET` | Fault reset | Hardware fault reset pin |
| `IO_AUX_GRID` | Auxiliary grid | Auxiliary grid contact |
| `IO_AUX_BAT` | Auxiliary battery | Auxiliary battery contact |
| `IO_FAULT_HARDWARE` | HW fault | Hardware fault input |
| `IO_FAULT_HARDWARE_IGRID_1` | Grid current fault 1 | Overcurrent protection grid ch1 |
| `IO_FAULT_HARDWARE_IGRID_2` | Grid current fault 2 | Overcurrent protection grid ch2 |
| `IO_FAULT_HARDWARE_VBUS` | DC bus fault | Overvoltage protection DC bus |
| `IO_FAULT_HARDWARE_IBAT_1` | Battery current fault 1 | Overcurrent protection battery ch1 |
| `IO_FAULT_HARDWARE_IBAT_2` | Battery current fault 2 | Overcurrent protection battery ch2 |

---

## 7. Debug shell (UART CLI)

The firmware contains a complete debug shell over UART (`vtask_shell`, 2 KB stack).
Shell library: letter-shell variant with command/variable/user registration.
Shell context struct in SRAM at `0x20001410`, input buffer at `0x2000147C` (512 bytes).

### 7.1 Authentication & password

#### Auth mechanism (assembly-verified)

Login handler: `FUN_08010208` (90 bytes). User lookup: `FUN_08010194` (16-byte entries,
strcmp via `FUN_08000388`). Password check in `FUN_0800f7a0`.

Authentication proceeds in three stages — **the first two grant access without a password**:

```
FUN_08010208 — login decision (assembly 0x08010210–0x0801022A):

  0x08010210: LDR r0, [r5, #0x8]      ; r0 = password pointer from user entry+8
  0x08010212: CMP r0, #0               ; NULL check
  0x08010214: BEQ 0x08010256           ; → NULL = NO PASSWORD → access granted
  0x08010216: BL  FUN_0800037a         ; r0 = strlen(password)
  0x0801021A: CBZ r0, 0x08010256       ; → strlen==0 = EMPTY PASSWORD → access granted
  0x0801021C: LDRH r0, [r4, #0x36]    ; check argument count
  0x08010222: LDR r1, [r4, #0x18]     ; r1 = user input buffer
  0x08010224: LDR r0, [r5, #0x8]      ; r0 = reload password
  0x08010226: BL  FUN_08000388         ; strcmp(password, input)
  0x0801022A: CBZ r0, 0x08010256       ; → match = access granted
              ; else: "password error", access denied
```

Confirmed helper functions:
- `FUN_0800037a` = `strlen` (14 bytes, do-while loop until `\0`)
- `FUN_08000388` = `strcmp` (returns 0 on match)
- `FUN_080106d0` = `shell_print` (string output via UART)

#### User entry

The shell command table resides in flash from `0x0801892C` to `0x08018C7C`
(53 entries × 16 bytes = 0x350 bytes, literal pool at `0x0800FE38/FE3C`).

> **Note:** The table is generated by the RVDS/Keil compiler in a code/data interleave format.
> The 16-byte entries are embedded between ARM instructions, which
> complicates static parsing of the struct fields. The entry lookup (`FUN_08010194`)
> iterates with `entry_addr = base + index * 0x10` and checks `*(byte)(entry+1) & 0xF`
> as the type field (type 8 = USER).

String data in the shell data area (`0x08013F70`–`0x080148FF`):

```
0x0801447C: 55 73 65 72 00          "User\0"
0x08014481: 00                      "\0"          ← empty password field
0x08014482: 64 65 66 61 75 6C 74 20 75 73 65 72 00  "default user\0"
```

#### Password assessment

| Indicator | Assessment |
|---|---|
| String data shows `User\0\0default user` | Empty password between name and description |
| Auth code has explicit NULL/empty paths | Both grant immediate access |
| User description is `"default user"` | Standard user without individual configuration |
| No password hash or string in the binary | No candidates found in brute-force search |
| Shell is a debug/developer tool | Typically no password in production FW |

**Result: the password is with high probability empty (empty string).**

#### Practical guide for UART access

```
1. Identify UART pins on the board (TX, RX, GND of the micro MCU)
2. Connect a USB-UART adapter (3.3V level!)
3. Open a terminal: 115200 baud, 8N1 (standard for STM32)
4. Press Enter
5. If "Please input password:" appears → press Enter (empty password)
6. The shell prompt should appear (presumably "User>")
7. Type "help" for the command list
8. Type "version" for SOFT_VERSION and BOOT_VERSION
```

> **Fallback:** If the empty password does not work, the password might be set at runtime
> via CAN by the EMS controller. In that case, CAN sniffing of the
> config block (CMD `0x16`, 8 bytes → SRAM `0x20003EA7`) would be the next approach.

### 7.2 Shell commands (complete, 33 commands)

Extracted from string analysis and Chinese descriptions:

| Command | Chinese description | Function |
|---|---|---|
| `version` | 版本信息 | SOFT_VERSION, BOOT_VERSION, hardware rev |
| `log_data` | 打印数据 | Log sensor data |
| `log_list` | 打印日志 | List log entries |
| `log_err` | 打印错误日志 | Show error log |
| `log_clear` | 打印错误日志 | Clear log |
| `log_open` | 开启日志自动打印 | Enable automatic log printing |
| `prt` | 持续打印 功能码[重复次数][打印间隔] | Continuous print: function code, repeat count, interval |
| `help_prt` | 持续打印项信息 | Show print options |
| `rtos_status` | RTOS状态 | FreeRTOS task statistics |
| `io_show` | IO状态 | Show GPIO pin status |
| `io_set` | IO状态设置 | Set GPIO pin manually |
| `fan_set` | 设置风扇占空比 0 - 400 | Set fan PWM duty 0–400 |
| `set_power` | 进入标准模式后输入功率 | Set power (after mode switch) |
| `show_power` | 展示powercfg功能 | Show set_power/max_power/low_power |
| `power_cfg` | 设定模式 设定值(…浮点数) 设定值2 | Power configuration (float values) |
| `pf_power` | 设定无功功率(…小于总功率) | Set reactive power (< total power) |
| `pf_set` | 设定pf值(范围 -1, 1) | Set power factor (-1 to 1) |
| `pf` | 设定pf值(范围 -1, 1) | Power factor (alias) |
| `bat_mode` | 设置为电池模式 | Activate battery mode |
| `backup_mode` | 设置为备份模式 | Backup (EPS) power mode |
| `par_mode` | 设置为备份模式 | Parallel mode |
| `set_mode` | 0零馈电 1手动 2交易 3调试 15双向电压源 | Select operating mode (see below) |
| `sr_set` / `set_sr` | 设定同步整流(1开启 0关闭) | Synchronous rectification on/off |
| `dac_high` | 设置PFC上限电流 | PFC upper current limit via DAC |
| `dac_low` | 设置PFC下限电流 | PFC lower current limit via DAC |
| `alarm` | 蜂鸣器噪声 | Buzzer control |
| `qpr` | 修改电流环qpr参数 kp kr | Change current loop PR controller parameters |
| `pi` | 修改电流环pi参数 | Current loop PI controller parameters |
| `vol_pi` | 修改电压环pi参数 | Voltage loop PI controller parameters |
| `show_pi` | 查看pi参数 | Show PI controller parameters |
| `update` | — | Self-update via UART (Ymodem) |
| `update_bms` / `update_bms2` | — | BMS update via CAN / RS485 |
| `reset` | — | System reset |
| `reset_memory` | 1:清空所有数据 其他:清空累计以外 | 1=erase everything, otherwise=without counters |
| `old_ate` | 设定老化模式 | ATE factory burn-in mode |
| `clear` | clear console | Clear terminal |
| `help` | show command info | Show help |

### 7.3 set_mode Operating Modes

From the Chinese description in the shell data area:

| Value | Chinese | Description |
|---|---|---|
| 0 | 零馈电 | Zero feed-in (no grid feed-in) |
| 1 | 手动模式 | Manual mode |
| 2 | 交易模式 | Trading mode (time-controlled) |
| 3 | 调试模式 | Debug/test mode |
| 15 | 双向电压源模式 | Bidirectional voltage source mode |

### 7.4 Version Strings

```
SOFT_VERSION:%d    — Firmware version (value: 116)
BOOT_VERSION:%d    — Bootloader version
hardware %d ,set_ver: %d — Hardware revision + set version
```

### 7.5 Shell Architecture (Internals)

| Component | Address/value | Description |
|---|---|---|
| Shell context | SRAM `0x20001410` | Main struct of the shell instance |
| Input buffer | SRAM `0x2000147C` | 512-byte command line buffer |
| Command table | Flash `0x0801892C` | 53 entries × 16 bytes |
| Table end | Flash `0x08018C7C` | Literal pool at `0x0800FE38/FE3C` |
| Init function | `FUN_0800fdc0` | Initialize shell context |
| User search | `FUN_08010194` | 16-byte entry iterator with strcmp |
| Name extractor | `FUN_0800fb10` | Read entry name from offset+4 |
| Login handler | `FUN_08010208` | Password check (3 paths) |
| Password check (2) | `FUN_0800f7a0` | Subsequent auth while the shell is running |
| Command dispatcher | `FUN_08010102` | Type-based dispatch (0=Func, 8=User) |
| Print function | `FUN_080106d0` | UART string output |

---

## 8. Inverter Control

### State machine:

```
ctl_state (0x200002A8) — main control
    ↳ llc_run_state (0x200002A5) — LLC converter (DC-DC)
    ↳ inverter_run_state (0x200002A4) — H-bridge (DC-AC)
```

### Control building blocks:
- **PID controller** with `kp`, `ki` (power and current control)
- **PR controller** (proportional-resonant) with `kp`, `kr` (grid current control)
- **RMS calculation** via FP summation (`g_irms_struct`)
- **Zero-crossing detection** (`zcd` = zero crossing detector)
- **Notch filter** for DC bus (`highv_notch_vol`)
- **Synchronous rectification** (`sr_duty`)

### Operating modes (`work_mode` @ `DAT_20003E04`):

| Value | Mode | Description |
|---|---|---|
| 0x00 | Normal | Normal operation |
| 0x01 | Test | Factory test mode |
| 0x04 | Update | Firmware update active |
| 0x0C | Factory | Factory mode (12) |

---

## 9. Firmware Update Mechanism

The Micro MCU supports three update paths:

| Path | Interface | Description |
|---|---|---|
| Self-update | UART (Ymodem) | Own FW via serial console |
| BMS via CAN | CAN bus | Forward BMS FW to the BMS MCU |
| BMS via RS485 | RS485/UART | Flash BMS FW via RS485 |

**Update sequence via CAN (controlled by the EMS controller):**
1. `0xC1` → step 1 (initialization)
2. `0x60` → set update type, work_mode=0x04
3. `0x51` → trigger update
4. `0x52(1)` → step 2, `0x52(2)` → step 3, `0x52(3)` → step 4
5. `0x53` → step 5 (verification)
6. `0x55` → step 6 (finalization)
7. `0xCE` → write flash data (type 0=init, 1=data, 2=verify)

---

## 10. Bootloader Thunks

The firmware contains thunks to external functions in the range `0x10000xxx`:

| Thunk address | Target | Presumed function |
|---|---|---|
| `0x08000960` | `0x10000B58` | Power controller function (3 calls) |
| `0x08000974` | `0x100009E8` | Inverter stop/start (2 calls) |
| `0x0800097E` | `0x10000160` | Float processing (3 calls) |
| `0x08000988` | `0x100069E4` | Enable/disable function (2 calls) |
| `0x080009BA` | `0x10006874` | One-time init function |
| `0x08000A1E` | `0x10000124` | One-time init function |

These functions reside in the system memory or bootloader area and are mapped into SRAM at startup.

**Addendum tranche 4a (10.07.2026):** In addition to the real thunk functions listed above that Ghidra recognized as `thunk_EXT_FUN_*`,
there is at least one **inline computed jump** into the
bootloader area: `UART_TxEvent_ADC1_BootloaderThunk` (`0x08003af0`, caller `UART_IRQHandler`)
jumps conditionally (only if the checked handle instance equals `0x50000000`/ADC1) directly to
`0x10001428` (`LAB_10001428`) — not a standalone thunk function body, but a
computed jump within a larger function. Target address `0x10001428` lies in the same
bootloader/system memory area as the table above, but has not been listed separately so far.

---

## 11. Telemetry Block & Status Block (CAN → Modbus Bridge)

> The EMS Control MCU periodically reads two data blocks from the inverter via CAN
> and exposes the values as Modbus registers. This analysis decodes both blocks
> completely from the builder function `build_telemetry_block` (`FUN_08001560`, 688 bytes).

### 11.1 Telemetry Block (CAN CMD `0x10` — 48 bytes)

SRAM target: `0x200038E8`–`0x20003917`. Sent via `FUN_08001f34(addr, 0x10, 0x200038E8, 0x30)`.

| Offset | Bytes | Source SRAM | Variable | Scale | Type | Description |
|--------|-------|-------------|----------|-------|------|-------------|
| 0x00 | 1 | `0x2000046D` | ctl_status | 1 | byte | Inverter operating status (0–6) |
| 0x01 | 1 | — | (reserved) | — | byte | Always 0 |
| 0x02 | 1 | `0x200002B3` | grid_connected | 1 | byte | Grid connection flag |
| 0x03 | 1 | `0x20003E03` | backup_mode | 1 | byte | Backup (EPS) power mode flag |
| 0x04 | 4 | `0x200019FC` | war1 | 1 | u32 | Warning bitmask |
| 0x08 | 4 | `0x200019F4` | err1 | 1 | u32 | Fault register 1 |
| 0x0C | 4 | `0x200019F8` | err2 | 1 | u32 | Fault register 2 |
| 0x10 | 2 | `0x200018E4` | grid_voltage_rms | ×10 | u16 | 电网电压有效值 — grid voltage RMS (V×10) |
| 0x12 | 2 | `0x200002F0` | (unknown) | ×10 | u16 | Float×10 — presumably grid frequency or current |
| 0x14 | 2 | `0x200018BC` | offgrid_voltage_rms | ×10 | u16 | 离网电压有效值 — off-grid voltage RMS (V×10) |
| 0x16 | 2 | `0x200002AB` | (unknown) | 1 | u16 | Byte→u16, presumably control parameter |
| 0x18 | 2 | `0x200018F0` | actual_power | 1 | i16 | 实际功率 — actual power (W) |
| 0x1A | 2 | `0x200018C8` | output_power | 1 | i16 | 发出功率 — output power (W) |
| 0x1C | 2 | `0x20000E00` | power_3 | 1 | i16 | Power measurement 3 (W) |
| 0x1E | 2 | `0x20000E10` | bat_vol_avg | ×10 | u16 | Battery voltage average (V×10) |
| 0x20 | 2 | `0x20000E30` | ntc_inv | ×10 | i16 | Inverter temperature (°C×10) |
| 0x22 | 2 | `0x20000E34` | ntc_mppt | ×10 | i16 | MPPT temperature (°C×10) |
| 0x24 | 2 | `0x2000030C` | max_power | 1 | u16 | Max. power (W, float→int) |
| 0x26 | 2 | `0x20000310` | min_power | 1 | u16 | Min. power (W, float→int) |
| 0x28 | 4 | `0x20003E83` | daily_charge_energy | 1 | u32 | 日充电 — daily charge energy (Wh) |
| 0x2C | 4 | `0x20003E87` | daily_discharge_energy | 1 | u32 | 日放电 — daily discharge energy (Wh) |

**Notes:**
- `ctl_status` values (from `FUN_0800d834`): 0=Idle, 1=Starting?, 2=Running?, 3=Fault?, 4=Stopping?, 6=Standby?
- Float→u16 conversion with ×10 via `VectorFloatToUnsigned(value * 10.0, 3)` — clamp to 3 decimal places
- Energy counter limits configured via CAN CMD `0x16` config block (SRAM `0x20003EA7`/`0x20003EAB`)
- Offsets 0x12 and 0x16 are not yet unambiguously identified — candidates: grid frequency, grid current RMS

### 11.2 Status Block (CAN CMD `0x11` — 8 bytes)

SRAM target: `0x20000440`–`0x20000447`. Also filled in `build_telemetry_block` (lines 41–45).

| Offset | Bytes | Source SRAM | Variable | Type | Description |
|--------|-------|-------------|----------|------|-------------|
| 0x00 | 1 | `0x20003E04` | work_mode | byte | Work mode (0=Normal, 1=Test, 4=Update, 0xC=Factory) |
| 0x01 | 1 | `0x200004A4` | sleep_mode | byte | Sleep mode flag |
| 0x02 | 1 | `0x200004A6` | bat_mode | byte | Battery mode |
| 0x03 | 1 | — | (padding) | byte | — |
| 0x04 | 2 | `0x200004AB` | max_charge_power | u16 | Max. charge power (W, ≤2500) |
| 0x06 | 2 | `0x200004AD` | max_discharge_power | u16 | Max. discharge power (W, ≤2500) |

### 11.3 Complete RMS SRAM Region (`0x200018B4`–`0x200018F0`)

Extracted from Chinese debug format strings:

| SRAM | Variable | Chinese | Description |
|---|---|---|---|
| `0x200018B4` | grid_current_rms | 并网电流有效值 | Grid current RMS (A) |
| `0x200018B8` | offgrid_current_rms | 离网电流有效值 | Off-grid current RMS (A) |
| `0x200018BC` | offgrid_voltage_rms | 离网电压有效值 | Off-grid voltage RMS (V) |
| `0x200018C0` | output_voltage_rms | 输出电压有效值 | Output voltage RMS (V) |
| `0x200018C4` | (output_power?) | — | Power variable (in the 4-variable block) |
| `0x200018C8` | output_power | 发出功率 | Output power (W) |
| `0x200018E4` | grid_voltage_rms | 电网电压有效值 | Grid voltage RMS (V) |
| `0x200018F0` | actual_power | 实际功率 | Current/actual power (W) |

### 11.4 Presumed Mapping Telemetry → Modbus Register

| Telemetry offset | Micro variable | Presumed Modbus register | Control FW description |
|---|---|---|---|
| 0x10 (grid_voltage_rms ×10) | `0x200018E4` | **32200** | ac_voltage |
| 0x14 (offgrid_voltage_rms ×10) | `0x200018BC` | **32300** | ac_offgrid_voltage |
| 0x18 (actual_power) | `0x200018F0` | 32100 or 32102? | inverter_power |
| 0x1A (output_power) | `0x200018C8` | 32100? | ac_output_power |
| 0x1E (bat_vol_avg ×10) | `0x20000E10` | 34003? | battery_voltage |
| 0x20 (ntc_inv ×10) | `0x20000E30` | 35000–35002 | temperature |
| 0x22 (ntc_mppt ×10) | `0x20000E34` | 35003–35005 | temperature |
| 0x28 (daily_charge Wh) | `0x20003E83` | 33000? | daily_charge_energy |
| 0x2C (daily_discharge Wh) | `0x20003E87` | 33004? | daily_discharge_energy |

> **Next step:** Analyze the Control FW v149.2 CAN receive function to confirm the
> exact mapping of telemetry offset → Modbus register number.
> Alternatively: CAN bus sniffer during Modbus read access to known registers.

---

## 12. Charge/Discharge Control & Pack Rotation

> **New.** Analysis of `FUN_0800d834` (942 bytes, 170 lines) — the main function for
> charge/discharge decisions and the rotation timing.

### 12.1 Inverter Status (`DAT_2000046D`)

The function sets the inverter status based on sensor data:

| Value | Condition | Meaning |
|------|-----------|-----------|
| 0 | `DAT_200004A4 == 1` (sleep mode) | **Standby/Sleep** |
| 1 | No backup, no parallel, ctl_state normal | **Normal operation** |
| 2 | Temperature `_DAT_20000E1C < -10` | **Undertemperature protection** |
| 3 | Temperature `_DAT_20000E1C >= 11` | **Overtemperature protection** |
| 4 | Backup/parallel active, ctl_state near 0xD4 | **Backup (EPS) mode active** |
| 6 | GPIO `0x48000C00` bits 15+3 set | **Hardware fault** |

### 12.2 Operating-Mode-Dependent Control

The logic depends on `DAT_20003E04` (work_mode):

| Mode | Value | Charge/discharge logic |
|------|-------|------------------------|
| 0 | Zero feed-in | Not in this function (presumably in Control FW) |
| 1 | Manual | Not in this function |
| 2 | Trading | `return` — no local control |
| **3** | **Debug/Auto** | **SOC-based rotation with 600s timer** (see 12.3) |
| **4** | **Calibration** | **Full charge/discharge cycle** (see 12.4) |

### 12.3 SOC-Based Rotation (Mode 3 = Debug/Auto)

> **Correction 2026-09-24:** This is not a pack rotation, but a test mode that only switches the
> **direction** (±2500 W). Which pack charges or discharges is decided by the master BMS
> (`BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md`). `FUN_0800ab88` is `debug_log_enqueue`, i.e. an
> entry in the ring buffer log (200 × 20 bytes), **not** a CAN update to the BMS. The 600 s interval has nothing to do with
> the observed rotation; that follows the 10 % steps of the BMS.

**State machine `DAT_2000046E`:**

| Value | Direction | Power value (IEEE 754) | Description |
|------|-----------|----------------------|-------------|
| 0 | Idle | `0x00000000` = 0W | Waiting for next decision |
| 1 | **Discharging** | `0x451C4000` = **+2500W** | Discharge battery |
| 2 | **Charging** | `0xC51C4000` = **-2500W** | Charge battery |

**Timer configuration:**

| Timer SRAM | Timeout | Seconds | Function |
|---|---|---|---|
| `0x20000470` | `0xE10` = 3600 | **60 minutes** | Main cycle timer (new decision) |
| `0x20000472` | 600 | **10 minutes** | Discharge sub-timer |
| `0x20000474` | 600 | **10 minutes** | Charge sub-timer |

**Decision logic:**

```
Every 3600s (1 hour):
  ├── SOC < 50.1% (501) → State = 2 (charge at -2500W)
  └── SOC ≥ 50.1%       → State = 1 (discharge at +2500W)

While charging (state 2):
  ├── Every 600s (10 min): send CAN update to BMS
  └── SOC ≥ 99.9% (999): → State = 0 (stop, full)

While discharging (state 1):
  ├── Every 600s (10 min): send CAN update to BMS
  └── SOC < 15.0% (150): → State = 0 (stop, empty)
```

**CAN update message** (via `FUN_0800ab88`):
```c
FUN_0800ab88(2, inverter_temp, soc, max_bms_temp);
```
Sends: CMD=2, inverter temperature, current SOC, max BMS temperature.

> **The user observation confirms:** The **600-second timer (10 minutes)** corresponds
> exactly to the "perceived" rotation interval from the app observation!

### 12.4 Calibration Cycle (Mode 4 = Self-Test)

7-stage charge/discharge cycle for factory/capacity tests:

```
State 0: Init
  → max_discharge = 2500W (0x9C4)
  → Test time: 9000s (2.5h), or 14400s (4h) if cap > 3000

State 1: Full charge → −2500W
  → Until SOC = 100% OR timeout

State 2: Pause → 0W
  → 59s minimum pause, until temp < 45°C or max 1800s

State 3: Full discharge → +2500W
  → Until SOC < 30% OR timeout
  → Measure discharge energy (daily_discharge)

State 4: Pause → 0W
  → 59s minimum pause, until temp < 50°C or max 1800s

State 5: Partial charge → −2500W
  → Until SOC > 47.9% (0x1DF)
  → Measure charge energy
  → Write results to flash (addr 0x950, 0x952)
  → Calibration data: DAT_20000497 = 1
  → Reduce max_discharge to 800W

State 6: Stop → 0W
```

**Flash calibration addresses:**

| Flash addr | Content | Description |
|---|---|---|
| `0x0950` | discharge_energy_cal | Measured discharge energy (Wh) |
| `0x0952` | charge_energy_cal | Measured charge energy (Wh) |
| `0x0090` | cal_flag | Calibration completed (1) |

### 12.5 Bootloader Functions (Thunks Identified)

| Thunk | Target | Function | Parameter |
|---|---|---|---|
| `thunk_EXT_FUN_10000160` | `0x10000160` | **set_inverter_power(float)** | Power in watts (±2500W) |
| `thunk_EXT_FUN_10000B58` | `0x10000B58` | **disable_inverter(0,0)** | Switch off inverter |

> `0x10000160` is the most important bootloader function: it sets the inverter power.
> Positive values = discharging, negative = charging. Maximum ±2500W (Venus D hardware limit).

### 12.6 Architecture Insight: Pack Rotation

> **Superseded (2026-09-24):** The pack selection lies in the **master BMS**, neither in the Micro nor in the
> Control. The packs are not all connected in parallel on the bus; exactly one is always active (only during
> a switchover briefly two on the discharge side). Details: `BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md`.

The analysis shows that the **Micro MCU controls the charge/discharge direction**, but the
**pack selection works differently than originally assumed:**

```
Mode 0/1/2 (Normal/Manual/Trading):
  → Control MCU specifies the power setpoint (via CAN CMD 0x01)
  → Micro MCU controls inverter
  → ALL packs are connected in parallel on the DC bus
  → BMS master distributes via MOSFET control (CAN CMD 6/3)
  → Rotation is triggered by BMS master or Control MCU

Mode 3 (Debug/Auto):
  → Micro MCU controls autonomously: ±2500W, 10-min intervals
  → SOC-based direction decision every 60 min
  → Pack rotation via BMS CAN CMD 6/3 (address-based)

Mode 4 (Calibration):
  → Autonomous 7-stage cycle, no pack rotation
```

> **Open question:** The pack rotation in normal mode (0/1/2) is presumably implemented in the
> **Control FW**, which sends pack-specific commands via RS485 to the Micro MCU,
> which forwards them as CAN CMD 6/3 to the BMS packs.

### 12.6b Implementation of the BMS Limits in the Micro (2026-09-24)

`BMS_CAN_Parser` (0x080018E8) stores the BMS frames PF1–PF4 in the structure at `0x2000397B`.
The debug output names the fields: `bat_vol`, `bat_cur`, `max_temp`, `soc`, `cap`, `charge_u`
(`0x2000398B`), `charge_i` (`0x2000398D`), `discharge_i` (`0x2000398F`), `charge_req`,
`force_charge_req`, `sleep_flag`, `err`/`warn`.

A piece of code at `0x0801B916`–`0x0801B9A0` (not recognized as a function by Ghidra) copies the
limits as floats into the structure `0x2000167C`:

| Offset | Field | without BMS data |
|---|---|---|
| +0x00 | SOC in % (u16) | 0 |
| +0x04 | Charge current limit (A) | **10 A** |
| +0x08 | Charge voltage limit (V) | — |
| +0x0C | Discharge current limit (A) | **10 A** |
| +0x10 | Discharge voltage limit (V) | **44.0 V** |

In work mode 4 (calibration), at least 10 A applies.

Consumers:

- `Inverter_Grid_Control` (0x08015450) compares the battery voltage with the
  charge voltage limit (hysteresis +1 / +1.5 / +2 V) and the discharge limit, and also the SOC with
  thresholds (among others SOC < 6 %, `DAT_200004AA`), and triggers events in each case (codes 9, 10,
  0x3C–0x3E).
- The **power limit** is computed in three functions in the range `0x080189FC`–`0x080190CA`:
  discharging `U_bat × I_discharge`, charging `U_bat × I_charge × 1.06`. The Ghidra names
  (`Protect_Err2Bit11_Timeout3000`, `Protect_Err2Bit7_Ev13`) come from the fault bit analysis. The
  functions were merged by the RVDS linker (cross-jump), so the names only fit in part.

So the Micro regulates the charge current itself to the BMS limit, with 6 % headroom above the value that
the Control applies via 32100 × 32106 (`Control_FW/VNSD_Control_Charge_Power_PV_Curtailment_v150.md`).
This fits the constant-current taper in the field.

**No** Micro function sends the pack release frame `0x1801FFAA`. `CAN_BuildArbID` computes
`prio<<24 | b2<<16 | b1<<8 | b0`, but is only called from `CAN_TX_SendMessage` with the segment number as
priority.

### 12.7 DC Bus Architecture & PV Behavior

> **Explanation derived from the FW architecture** for observed PV throttling behavior
> and the regular charge/discharge cycle.

#### Physical signal path (one inverter, one DC bus)

```
PV panels (roof)
   ↓ DC
MPPT controller (ntc_mppt, DAT_20000E34)
   ↓ DC
┌──────────────────────────────────────────────────┐
│             DC bus (~400V)                       │
│          highv_vol (DAT_20000DEC)                │
│          highv_notch_vol (DAT_20000E0C)          │
│                                                  │
│   ← LLC converter (DC-DC) →      H-bridge (DC-AC) →
│   llc_run_state                   inverter_run_state
│   bat_vol, bat_cur                grid_vol, grid_cur
└──────────┬───────────────────────────┬───────────┘
           ↓                           ↓
     Battery packs                AC grid (house grid)
     (parallel on the bus)        grid_volt, offgrid_volt
```

**Critical limitation:** There is only ONE `set_inverter_power(float)` call
(`thunk_EXT_FUN_10000160`). The inverter cannot charge AND feed into the grid at the same
time. The power value controls the NET direction:

| Power value | Energy flow | PV behavior |
|---|---|---|
| **< 0** (negative) | PV/grid → battery (charging) | MPPT delivers at most the charging demand, the rest is throttled |
| **= 0** | Inverter off | MPPT runs at open-circuit voltage, no grid export |
| **> 0** (positive) | Battery → grid (discharging) | PV + battery → grid (BOTH at the same time!) |

#### Observed behavior: PV throttling with nearly full battery

**Phase 1 — Battery charges, PV is progressively throttled:**
```
BMS: SOC rises → current limit matrix (flash 0x0801B884) reduces max_charge_current
     Temperature × SOC → permitted charge current drops from e.g. 50A to 5A
     charge_current_limit via CAN PF=3 → Micro MCU
     
Micro: set_inverter_power = -(BMS charge_limit × bat_voltage)
       Inverter only takes little power from the DC bus
       
MPPT: DC bus voltage rises → MPPT regulates operating point down
      → PV panels deliver less than possible
      → Surplus does NOT go into the grid (no separate path)
```

**Phase 2 — Battery at 100%, short idle phase:**
```
BMS: show_soc ≥ 999 (99.9%) + full_flag → SOC capped at 1000
Micro: charge direction stops → set_inverter_power(0)
       Inverter completely off, PV fully throttled
```

**Phase 3 — Discharging begins, PV flows again:**
```
Timer 0x20000470 expired (3600s/60min) OR Control MCU CMD
SOC ≥ 50.1% → state = 1 (discharge)
set_inverter_power(+2500W)

LLC converter draws energy FROM the battery → DC bus
MPPT pushes PV energy onto the same DC bus
H-bridge exports the SUM (PV + battery) into the grid
→ PV power flows completely into the house grid again!
```

**Phase 4 — SOC falls, cycle repeats:**
```
SOC falls due to discharging
If SOC < threshold → back to phase 1 (charging)
→ Regular charge/discharge cycle observable
```

#### Why Marstek does not solve this differently

The Venus D is a **battery inverter with integrated MPPT**, not a true hybrid inverter.
True hybrid inverters (Fronius, SMA) have **separate power paths** with independent controllers:
- Path 1: PV → grid (always active, independent of the battery)
- Path 2: PV → battery (additionally, when there is surplus)
- Path 3: Battery → grid (as needed)

The Venus D has only **one bidirectional path** (LLC ↔ DC bus ↔ H-bridge).
The state machine (`ctl_state` → `llc_run_state` + `inverter_run_state`) does allow
two separate state machines, but the power setpoint (`set_inverter_power`)
is a single float — the FW cannot do "charge at 500W AND feed in 1000W PV".

---

## 13. Complete Function List (363 named / 445 total, status 10.07.2026 after tranche 3c — see 13.13/13.15 for the most current numbers, tables 13.1–13.4 below are an older status)

> **History:** The note "07.07.2026 — mass analysis via 5 parallel Ghidra agents, 398/445 named"
> from an earlier version of this documentation was **misleading** — those renamings were never saved
> in Ghidra (actual Ghidra state at session start on 10.07.2026: only 21 functions named). The
> tables below (13.1–13.4) therefore predominantly contain **name proposals from that non-persisted
> analysis**, not the actual Ghidra state.
>
> **Session 10.07.2026:** 106 of these proposed names were individually verified via decompilation
> (not adopted blindly) and renamed in Ghidra — see **section 13.9** for the complete list,
> confidence assessment and the corrections/rejections found in the process. In addition, the duplicate
> `CAN_Filter_Setup` was fixed: the 10-byte thunk at `0x080150f2` (wrongly also named `CAN_Filter_Setup`)
> was renamed to `thunk_CAN_Filter_Setup`; the real 260-byte function remains at `0x0800eb88`.
>
> Tables 13.1–13.4 are **no longer verified up to date for all rows** — only the addresses
> listed in 13.9 are confirmed for the actual Ghidra state on 10.07.2026. For all other rows:
> the name is an **unverified proposal**, still `FUN_<address>` in Ghidra, until
> verified in a follow-up session.

### 13.1 C Runtime & Startup (0x08000200–0x080009FF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800020c` | ~~`NVIC_SystemReset`~~ ❌ | 40B | **Refuted 10.07.2026** — decompilation shows privilege escalation + SVC(0), no AIRCR write. Remains `FUN_0800020c`, see 13.9 |
| `0x08000234` | `FPU_Enable` | 14B | Enable FPU via CPACR |
| `0x080002a4` | `get_current_irq_number` | 6B | Current ISR number from IPSR |
| `0x080002d0` | `__aeabi_uldivmod` | 98B | 64-bit unsigned division |
| `0x08000332` | `memcpy` | 36B | Memory copy (word-aligned) |
| `0x08000356` | `__aeabi_memset` | 14B | ARM EABI memset (dst, n, val) |
| `0x08000364` | `memclr` | 4B | Zero memory (22 calls) |
| `0x08000368` | `memset` | 18B | Standard memset |
| `0x0800037a` | `strlen` | 14B | String length |
| `0x08000388` | `strcmp` | 28B | String comparison |
| `0x080003a4` | `strcpy` | 18B | String copy |
| `0x080003b6` | `strncmp` | 30B | String comparison with limit |
| `0x080003d4` | `__aeabi_ui2d` | 38B | uint → double conversion |
| `0x0800042c` | `__aeabi_uidiv` | 44B | 32-bit unsigned division |
| `0x08000458` | `__aeabi_llsl` | 30B | 64-bit left shift |
| `0x08000476` | `__aeabi_llsr` | 32B | 64-bit right shift |
| `0x08000496` | `__aeabi_dadd` | 322B | Double addition |
| `0x080005e4` | `__aeabi_dmul` | 228B | Double multiplication |
| `0x080006c8` | `__aeabi_ddiv` | 252B | Double division |
| `0x080007a6` | `__aeabi_d2lz` | 48B | Double → int64 |
| `0x080007d8` | `__aeabi_d2iz` | 48B | Double → int32 |
| `0x0800082c` | `__aeabi_lasr` | 36B | 64-bit arithmetic right shift |
| `0x0800086e` | `__aeabi_dnorm` | 156B | Double normalization/rounding |

### 13.2 Peripheral Drivers — GPIO, UART, CAN, DMA (0x08000A00–0x080042FF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08000a28` | `UART_IRQHandler` | 126B | UART interrupt handler |
| `0x08000aa6` | `UART_ErrorHandler` | 14B | UART error handler |
| `0x08000ac0` | `UART_DMA_Handler` | 10B | UART DMA transfer complete |
| `0x08000acc` | `UART_WaitReady` | 128B | Wait for UART ready with timeout |
| `0x08000b50` | ~~`CAN_WaitReady`~~ ❌ | 134B | **Refuted 10.07.2026** — caller context is UART/DMA, not CAN. Remains `FUN_08000b50`, see 13.9 |
| `0x08000bdc` | `GPIO_ConfigPin` | 32B | Configure GPIO pin (normal speed) |
| `0x08000bfc` | `GPIO_ConfigPin_HighSpeed` | 32B | Configure GPIO pin (high speed) |
| `0x08000c20` | `Peripheral_GPIO_Init` | 658B | Initialize all GPIO ports (A-E) |
| `0x08000ef0` | `ADC_ConvertRawValues` | 172B | ADC raw values → calibrated floats |
| `0x08000fb4` | `CAN_Peripheral_Init` | 394B | Initialize CAN1/2/3 peripherals |
| `0x08001158` | `ADC_ProcessSamples` | 360B | Process ADC DMA buffer |
| `0x080012e0` | `Grid_Protection_SetLimits` | 406B | Set grid protection limits per country |
| `0x0800150a` | ~~`NVIC_SetPriority`~~ ❌ | 40B | **Refuted 10.07.2026** — does not match NVIC_IPR access. Remains `FUN_0800150a`, see 13.9 |
| `0x08001532` | ~~`NVIC_EnableIRQ`~~ ❌ | 40B | **Refuted 10.07.2026** — the real NVIC_EnableIRQ is at `0x0800675c` (see 13.3/13.9). Remains `FUN_08001532` |
| `0x08001868` | `CAN_RX_ParseBMSCommand` | 60B | Parse BMS CAN commands |
| `0x08001e3c` | `CAN_TX_ReadQueue` | 90B | Read and send CAN TX queue |
| `0x08001ea0` | `UART_Init` | 92B | Initialize UART2 |
| `0x08001f04` | `CAN_RX_DispatchTask` | 36B | CAN RX FreeRTOS task |
| `0x08001f34` | `CAN_TX_SendMessage` | 232B | Segment data into 8-byte CAN frames |
| `0x08002024` | `CAN_TX_SendCommand` | 14B | Send CAN command without data |
| `0x08002034` | `CAN_TX_SendFrame` | 54B | Send a single CAN frame |
| `0x08002070` | `CAN_TX_ProcessQueue` | 54B | Process CAN TX ring buffer |
| `0x080020b0` | `CAN_BuildArbID` | 30B | Build CAN arbitration ID |
| `0x080020d0` | `CAN_SetFilter_ExtID` | 50B | CAN extended ID filter |
| `0x08002108` | `CAN_SetFilter_StdID` | 48B | CAN standard ID filter |
| `0x0800213c` | `OTA_FW_Update_StateMachine` | 386B | OTA update state machine |
| `0x08002614` | `Serial_ValidatePacket` | 92B | Validate serial packet (XOR CRC) |
| `0x0800273e` | `memcpy_reverse` | 36B | Big→little-endian copy |
| `0x0800277c` | `DMA_CalcBaseAndOffset` | 54B | Calculate DMA stream base register |
| `0x080027e0` | `DMA_SetConfig` | 66B | Configure DMA stream |
| `0x08002824` | `TIM6_PWM_Init` | 96B | Initialize TIM6 PWM |
| `0x0800288c` | `TIM7_PWM_Init` | 74B | Initialize TIM7 PWM |
| `0x080028e0` | `TIM8_PWM_Init` | 92B | Initialize TIM8 PWM |
| `0x0800299c` | `Inverter_SetMode` | 86B | Set inverter operating mode |
| `0x080029fc` | `EEPROM_ClearStats` | 70B | Clear EEPROM statistics |
| `0x08002a3c` | `delay_ms` | 40B | Millisecond delay (6 calls) |
| `0x08002a68` | `EEPROM_LoadConfig` | 614B | Load configuration from I2C EEPROM |
| `0x08002ce8` | `EEPROM_WriteRegister` | 144B | Write EEPROM register (I2C mutex) |
| `0x08002d84` | `EEPROM_ReadRegister` | 120B | Read EEPROM register |
| `0x08002e14` | `EEPROM_SaveTimestamp` | 44B | Save timestamp to EEPROM |
| `0x08002e40` | `EEPROM_WriteVerify` | 300B | Write+verify EEPROM (8 calls) |
| `0x08002f78` | `EEPROM_WriteVerify_NoMutex` | 216B | Write EEPROM (boot time) |
| `0x08003078` | `Error_Handler` | 10B | Fatal fault → infinite loop (7 calls) |
| `0x08003090` | `CAN_InitMailboxes` | 100B | Initialize CAN mailbox pointers |
| `0x08003104` | `CAN_CopyTxFrame` | 112B | TX header+data into CAN mailbox |
| `0x08003178` | `FLASH_FlushCaches` | 84B | Flush flash I/D cache |
| `0x080031d4` | `FLASH_MassErase` | 54B | Flash mass erase |
| `0x08003210` | `FLASH_SectorErase` | 72B | Flash sector erase |
| `0x0800325c` | `FLASH_ProgramDoubleWord` | 22B | Write 8 bytes to flash |
| `0x08003278` | `FLASH_Program256Bit` | 38B | Write 32-byte block to flash |
| `0x080032a4` | `FLASH_WaitForOperation` | 98B | Wait for flash BSY flag |
| `0x08003310` | `FLASH_EraseSectors` | 96B | Erase flash sector range |
| `0x08003378` | `FLASH_WriteData` | 108B | Write data buffer to flash |
| `0x080033f0` | `Get_GridFrequency` | 62B | Calculate AC grid frequency |
| `0x0800343c` | `checksum_add` | 28B | Additive checksum |
| `0x08003458` | `checksum_xor` | 30B | XOR checksum |
| `0x080035d4` | `UART_SetBaudRate` | 148B | Set UART baud rate |
| `0x08003670` | `HAL_GPIO_Init_Extended` | 1094B | Full GPIO init (mode/AF/speed/pull/EXTI) |
| `0x08003af0` | `UART_RX_Callback` | 24B | UART1 RX callback |
| `0x08003b04` | `HAL_UART_Init` | 546B | Fully initialize UART peripheral |
| `0x08003d54` | `UART_MspInit` | 634B | UART MSP init (clocks, GPIO, DMA) |
| `0x08003ff8` | `UART_StartReceive_DMA` | 264B | Start UART DMA receive |
| `0x08004128` | `HAL_TIM_Base_Init_Full` | 604B | Fully initialize timer base |

### 13.3 Extended Peripherals — TIM, SPI, HRTIM, I2C (0x08004300–0x080097FF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x080043a8` | `TIM_PWM_Channel_Config` | 148B | Configure TIM PWM channel |
| `0x08004528` | `TIM_PWM_Start` | 458B | Start TIM PWM output (3 calls) |
| `0x080046fc` | `TIM_CC_Enable` | 42B | Enable TIM capture/compare |
| `0x08004728` | `TIM_IC_Start_DMA` | 130B | TIM input capture via DMA |
| `0x080047bc` | `TIM_DMA_DelayPulse` | 50B | TIM DMA delay pulse callback |
| `0x080047ee` | `TIM_CCx_ChannelCmd` | 110B | TIM CC channel on/off |
| `0x0800485c` | `TIM_OC_ConfigChannel` | 104B | TIM output compare channel config |
| `0x080048c4` | `TIM_IC_ConfigChannel` | 118B | TIM input capture channel config |
| `0x0800493a` | `TIM_SlaveMode_Config` | 204B | Configure TIM slave mode |
| `0x08004a08` | `TIM_MasterMode_Config` | 180B | Configure TIM master mode |
| `0x08004ac8` | `TIM_DMA_Config` | 144B | TIM DMA burst transfer |
| `0x08004b58` | `TIM_Generate_Event` | 32B | Generate TIM software event |
| `0x08004b7c` | `TIM_Base_Start_IT` | 224B | Start TIM with interrupt |
| `0x08004c5c` | `TIM_Encoder_Config` | 78B | Configure TIM encoder mode |
| `0x08004caa` | `TIM_ClockSource_Config` | 94B | Set TIM clock source |
| `0x08004d08` | `TIM_SetPrescaler_Event` | 54B | TIM prescaler + update event |
| `0x08004d70` | `SPI_Init` | 254B | Initialize SPI peripheral |
| `0x08004e76` | `SPI_TransmitReceive_IT` | 360B | SPI interrupt-based TX/RX |
| `0x08004fe0` | `SPI_IRQHandler` | 466B | SPI interrupt handler (TX/RX/error) |
| `0x080051c0` | `DMA_Init` | 124B | Initialize DMA stream |
| `0x08005248` | `DMA_Start_IT` | 112B | Start DMA with interrupt |
| `0x080052c6` | `DMA_DeInit` | 44B | Deinitialize DMA stream |
| `0x080052f2` | `DMA_IRQHandler` | 132B | DMA interrupt handler (TC/HT/TE) |
| `0x08005384` | `ADC_Init` | 238B | Initialize ADC peripheral |
| `0x0800547c` | `ADC_ConfigChannel` | 22B | Configure ADC channel sequence |
| `0x08005498` | `ADC_Start_DMA` | 120B | Start ADC DMA conversion |
| `0x08005518` | `ADC_Calibrate` | 28B | Run ADC calibration |
| `0x080055c4` | `HAL_RCC_ClockConfig` | 400B | RCC clock configuration (9 calls!) |
| `0x08005774` | `HAL_RCC_GetPCLK1Freq` | 14B | Read PCLK1 frequency |
| `0x08005782` | `HAL_RCC_GetPCLK2Freq` | 16B | Read PCLK2 frequency |
| `0x08005792` | `HAL_RCC_GetHCLKFreq` | 12B | Read HCLK frequency |
| `0x080057a0` | `HAL_RCC_GetSysClockFreq` | 8B | Read system clock |
| `0x080057ac` | `HAL_GetTick` | 6B | Read tick counter (26 calls!) |
| `0x08005c10` | `I2C_Init` | 174B | Initialize I2C peripheral |
| `0x08005cc8` | `I2C_DisableOwnAddress` | 30B | I2C disable own address |
| `0x08005cf0` | `I2C_Master_Transmit` | 176B | I2C master TX with timeout |
| `0x08005db0` | `I2C_Master_Receive` | 64B | I2C master RX |
| `0x08005df0` | `I2C_Mem_Read` | 56B | I2C read register (mem address) |
| `0x0800617c` | `I2C_RequestMemoryWrite` | 90B | Build I2C memory write request |
| `0x080061d6` | `I2C_RequestMemoryRead` | 86B | Build I2C memory read request |
| `0x0800622c` | `I2C_WaitOnMasterFlag` | 186B | Wait for I2C master flag with error handling |
| `0x080062ec` | `I2C_MasterTransmit_ISR` | 324B | I2C master TX interrupt loop (4 calls) |
| `0x08006438` | `I2C_MasterReceive_ISR` | 324B | I2C master RX interrupt loop (4 calls) |
| `0x08006584` | `I2C_Slave_AddrCallback` | 164B | I2C slave address callback |
| `0x080066d8` | `IWDG_Init` | 68B | Initialize watchdog |
| `0x0800675c` | `NVIC_EnableIRQ` ✅ (**corrected** 10.07.2026, was `IWDG_SetPrescaler`) | 26B | Enable interrupt via NVIC->ISER bit-band |
| `0x08006778` | `IWDG_WaitForReady` | 60B | Wait for watchdog ready |
| `0x08006800` | `DAC_Init` | 190B | Initialize DAC peripheral |
| `0x080068dc` | `DAC_SetChannel` | 776B | Configure DAC channel (4 calls) |
| `0x08006bec` | `DAC_DMA_Config` | 432B | Configure DAC DMA transfer |
| `0x08006dc0` | `HAL_RCC_GetSysClockFreq` ✅ (**corrected** 10.07.2026, was `DAC_SetValue`) | 106B | Determine system clock from RCC_CFGR SWS bits (16/8 MHz) |
| `0x08006e34` | `HRTIM_Init` | 1090B | Initialize high-resolution timer |
| `0x080072d0` | `HRTIM_StartPWM` | 80B | Start HRTIM PWM |
| `0x08007324` | `HRTIM_CounterStart_IT` | 142B | Start HRTIM counter with interrupt |
| `0x080073b8` | `HRTIM_OutputConfig` | 74B | Configure HRTIM output |
| `0x0800740c` | `HRTIM_SetCompare` | 52B | Set HRTIM compare value |
| `0x08007464` | `HRTIM_SetDeadTime` | 40B | Configure HRTIM dead time |
| `0x08007498` | `HRTIM_TimerInit` | 188B | Initialize HRTIM timer unit |
| `0x08007574` | `HRTIM_WaveformInit` | 136B | HRTIM waveform init |
| `0x08007606` | `HRTIM_OutputStart` | 78B | Enable HRTIM output pins |
| `0x08007654` | `HRTIM_OutputStop` | 50B | Disable HRTIM output pins |
| `0x08007724` | `HRTIM_SoftwareUpdate` | 68B | HRTIM software trigger |
| `0x08007774` | `HRTIM_BurstModeConfig` | 192B | Configure HRTIM burst mode |
| `0x08007834` | `HRTIM_EventConfig` | 78B | Configure HRTIM event input |
| `0x08007884` | `HRTIM_FaultConfig` | 50B | HRTIM fault input |
| `0x080078c0` | `HRTIM_ADCTriggerConfig` | 358B | Configure HRTIM ADC trigger |
| `0x08007a48` | `HRTIM_MspInit` | 518B | HRTIM MSP init (GPIO, DMA, NVIC) |
| `0x08007c4c` | `HRTIM_TimerBaseConfig` | 110B | Configure HRTIM timer base |
| `0x08007cbc` | `HRTIM_CaptureConfig` | 50B | HRTIM capture unit |
| `0x08007cfc` | `HRTIM_CompareConfig` | 78B | HRTIM compare unit |
| `0x08007d4c` | `HRTIM_WaveformTimerConfig` | 324B | HRTIM waveform timer |
| `0x08007eb8` | `HRTIM_OutputSetConfig` | 310B | HRTIM output set/reset source |
| `0x08007fee` | `HRTIM_EventFilterConfig` | 78B | HRTIM event filter |
| `0x0800803c` | `HRTIM_DeadTimeConfig` | 172B | HRTIM dead time complete |
| `0x08008100` | `TIM_PWM_Start_NoIT` | 258B | TIM PWM without interrupt |
| `0x080082e4` | `TIM_PeriodElapsedCallback` | 24B | TIM6 update callback |
| `0x080082f8` | `TIM_ReadCapturedValue` | 44B | Read TIM CCR register |
| `0x08008330` | `UART_AbortReceive` | 188B | Abort UART receive |
| `0x080083f8` | `UART_IRQHandler_Full` | 776B | UART IRQ handler (IDLE/RX/TX/error) |
| `0x080086e4` | `HAL_UART_Receive_IT` | 366B | Start UART interrupt receive |
| `0x0800873c` | `SPI_DMA_TransferComplete` | 282B | SPI DMA completion callback |
| `0x08008834` | `HAL_UART_Transmit_DMA` | 152B | Start UART DMA transmit |
| `0x080088dc` | `HRTIM_UpdatePWMDuty` | 338B | Update HRTIM PWM duty |
| `0x08008d4c` | `HRTIM_TimerBaseConfig_Ext` | 38B | HRTIM timer prescaler/period |
| `0x08008d72` | `HRTIM_TimerConfig_Ext` | 148B | HRTIM timer control |
| `0x08008e06` | `HRTIM_OutputConfig_Ext` | 222B | HRTIM output polarity/sources |
| `0x08008ee4` | `HRTIM_CaptureUnitConfig` | 56B | HRTIM capture unit |
| `0x08008f1c` | `HRTIM_CompareUnitConfig` | 30B | HRTIM compare unit prescaler |
| `0x08008f3c` | `HRTIM_TimerWaveformConfig` | 432B | HRTIM waveform (deadtime/burst/DLL) |
| `0x080090f8` | `HRTIM_EventConfig_Full` | 80B | HRTIM event filter/blanking |
| `0x08009694` | `UART_EndTransfer` | 34B | Reset UART after fault |
| `0x080096b8` | `UART_WaitOnFlagUntilTimeout` | 268B | Wait for UART flag + error check |
| `0x080097c8` | `I2C_TransferConfig` | 44B | Configure I2C CR2 |
| `0x080097f8` | `I2C_WaitOnFlagUntilTimeout` | 86B | Wait for I2C flag |

### 13.4 Application Logic — Modbus, Ymodem, Debug Log (0x08009800–0x0800CFFF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08009850` | `UART_Transmit_Wait` | 162B | Wait for UART TX with timeout |
| `0x080098f8` | `UART_Receive_Wait` | 76B | Wait for UART RX with timeout |
| `0x08009944` | `UART_Transmit_Wait_Alt` | 80B | UART TX alternative (∞ timeout) |
| `0x08009994` | `I2C1_Init` ✅ (**corrected** 10.07.2026, was `USART2_Init`) | 76B | Initialize I2C1 handle (base 0x40005400), used by EEPROM functions |
| `0x080099ec` | `uint_to_decimal_string` | 66B | uint32 → ASCII decimal string |
| `0x08009a34` | `IWDG_Reload` | 18B | Feed watchdog (0xAAAA) |
| `0x08009a4c` | ~~`DMA_Get_TCIF`~~ ❌ | 8B | **Refuted** — never applied in Ghidra. Remains `FUN_08009a4c`, see 13.21: `(*(uint*)(p1+8)&0xf)>>3`, callers `HAL_GPIO_Init_Extended` AND `HAL_ADC_Init` — cross-domain, no DMA registers |
| `0x08009a54` | ~~`DMA_Get_Error_Flag`~~ ❌ | 8B | **Refuted** — likewise. Remains `FUN_08009a54`, see 13.21: `*(uint*)(p1+8)&1`, callers `UART_WaitReady`/`ADC_ConversionStop_WaitReady`/`HAL_GPIO_Init_Extended`/`HAL_ADC_Init` — cross-domain |
| `0x08009a5c` | ~~`DMA_Get_HTIF`~~ ❌ | 8B | **Refuted** — likewise. Remains `FUN_08009a5c`, see 13.21: `(*(uint*)(p1+8)&7)>>2`, callers `HAL_GPIO_Init_Extended`/`HAL_ADC_Init`/`HAL_ADC_Start_DMA` — cross-domain |
| `0x08009b48` | `debug_log_dequeue` | 42B | Debug ring buffer: fetch next entry |
| `0x08009ba0` | `debug_log_format_entry` | 1712B | Format debug log (~60 event types, largest function!) |
| `0x0800ab88` | `debug_log_enqueue` | 70B | Insert event into debug ring buffer |
| `0x0800abda` | `CRC16_Modbus` | 50B | CRC-16/Modbus (poly 0xA001) |
| `0x0800ac0c` | `Modbus_Process_Request` | 260B | Modbus dispatcher: FC03/FC06/FC10 |
| `0x0800ad14` | `Modbus_UART_Start_Receive` | 26B | USART1 DMA receive for Modbus |
| `0x0800ad40` | `NTC_ADC_To_Temperature_1` | 128B | NTC lookup table 1 (binary search) |
| `0x0800adcc` | `NTC_ADC_To_Temperature_2` | 128B | NTC lookup table 2 |
| `0x0800ae58` | `CAN_Send_Packet` | 172B | Build CAN packet + send via USART2 |
| `0x0800af54` | `I2C_Get_Clock_Freq` | 66B | Calculate I2C clock frequency from PLL |
| `0x0800afa4` | `BCD_To_Decimal` | 18B | BCD → decimal conversion |
| `0x0800afb6` | `I2C_Wait_SB_Flag` | 72B | Wait for I2C start bit (1s timeout) |
| `0x0800affe` | `I2C_Wait_ADDR_Flag` | 80B | Wait for I2C ADDR flag |
| `0x0800b050` | `Ymodem_Receive_Packet` | 186B | Receive Ymodem packet (SOH/STX/EOT/CAN) |
| `0x0800b120` | `Modbus_Read_Holding_Registers` | 222B | Modbus FC03: read holding registers |
| `0x0800b208` | `Modbus_Write_Single_Register` ✅ (**corrected** 10.07.2026, was `Modbus_Read_Input_Registers`) | 90B | Modbus FC06: write single register |
| `0x0800b26c` | `Modbus_Broadcast_Write_Single_Register` ✅ (**corrected** 10.07.2026, was `Modbus_Broadcast_Read_Input_Regs`) | 44B | Modbus broadcast FC06: write single register |
| `0x0800b29c` | `Modbus_Write_Multiple_Registers` | 206B | Modbus FC10: write multiple registers |
| `0x0800b374` | `Modbus_Broadcast_Write_Multi_Regs` | 134B | Modbus broadcast write multiple |
| `0x0800b400` | `RTC_Read_Time` | 90B | Read RTC time via I2C (BCD) |
| `0x0800b540` | `Ymodem_Send_Byte` | 10B | Send Ymodem byte (UART5) |
| `0x0800b80c` | `Ymodem_Parse_File_Header` | 200B | Parse Ymodem file header |
| `0x0800b8d4` | `SysTick_Handler_App` | 64B | App SysTick: ms/sec counter |
| `0x0800b958` | `TIM_Base_Init` | 176B | Configure TIM base registers |
| `0x0800ba2c` | `TIM_Set_DMA_Bit` | 26B | Set/clear TIM DIER bit |
| `0x0800ba60` | `TIM_OC1_Config` | 148B | TIM output compare channel 1 |
| `0x0800bb10` | `TIM_OC2_Config` | 136B | TIM output compare channel 2 |
| `0x0800bbb4` | `TIM_OC3_Config` | 134B | TIM output compare channel 3 |
| `0x0800bc58` | `TIM_OC4_Config` | 136B | TIM output compare channel 4 |
| `0x0800bcfc` | `TIM_IC1_Config` | 92B | TIM input capture channel 1 |
| `0x0800bd74` | `TIM_IC2_Config` | 94B | TIM input capture channel 2 |
| `0x0800bdf0` | `TIM_ETR_Config` | 100B | TIM external trigger |
| `0x0800bf10` | `USART3_PWM_Init` | 96B | USART3 on TIM3 PWM |
| `0x0800bf7c` | `USART3_Deinit` | 168B | Deinitialize USART3 timer |
| `0x0800bf94` | `TIM1_PWM_Init` | 80B | TIM1 PWM: period 400, duty 200 |
| `0x0800c008` | `TIM8_PWM_Init_Encoder` | 102B | TIM8 encoder mode |
| `0x0800c078` | `TIM1_Set_Duty` | 26B | Set TIM1 PWM duty (0-400) |
| `0x0800c098` | `TIM5_Init` | 152B | Initialize TIM5 (period 999) |
| `0x0800c0d8` | `Timer_Elapsed_Ms` | 34B | Soft timer: ms elapsed? |
| `0x0800c100` | `Timer_Elapsed_Sec` | 34B | Soft timer: second elapsed? |
| `0x0800c128` | `SPI_DMA_Complete_IRQ` | 80B | SPI DMA completion IRQ |
| `0x0800c178` | `SPI_DMA_Start_Receive` | 40B | Start SPI DMA receive |
| `0x0800c1e4` | `SPI_DMA_RX_Disable` | 78B | Disable SPI DMA RX |
| `0x0800c25c` | `SPI_DMA_TX_Disable` | 46B | Disable SPI DMA TX |
| `0x0800c28a` | `SPI_DMA_RX_IRQ_Handler` | 160B | SPI DMA RX interrupt |
| `0x0800c32c` | `SPI_DMA_TX_IRQ_Handler` | 362B | SPI DMA TX interrupt |
| `0x0800c7f8` | `CAN_Wait_Response` | 98B | Wait for CAN response (3s timeout) |
| `0x0800c94c` | `CAN_Start_FW_Update` | 28B | Activate FW update mode |
| `0x0800ca3c` | `Operating_Hours_Update` | 204B | Operating hours + fault counter |
| `0x0800cb10` | `UART5_Send_Byte` | 52B | Send UART5 byte (polled) |
| `0x0800cb50` | `USART1_DMA_Send` | 42B | Start USART1 DMA TX |
| `0x0800cb80` | `USART2_Poll_Send` | 40B | Send USART2 byte by byte |
| `0x0800cbac` | `UART5_Poll_Send` | 40B | Send UART5 byte by byte |
| `0x0800cbd8` | `Shell_Task_Init` | 18B | Set shell prompt + init |

### 13.5 Printf, Utility, OTA (0x0800D000–0x0800EDFF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800dc40` | `ota_firmware_download` | 434B | OTA FW download: receive chunks, write flash |
| `0x0800de28` | `debug_sprintf_wrapper` | 34B | sprintf wrapper |
| `0x0800de72` | `NVIC_SetPriority_Alt` | 32B | NVIC priority (0xE000E400) |
| `0x0800de94` | `assert_failed_halt` | 26B | Assert/fault → DSB + infinite loop |
| `0x0800deec` | `printf_float_format` | 334B | Float→string formatting for printf |
| `0x0800e050` | `printf_core` | 1696B | Printf format engine (%d/%x/%s/%f) |
| `0x0800e72c` | `printf_pad_trailing` | 36B | Trailing space padding |
| `0x0800e750` | `printf_pad_leading` | 46B | Leading space/zero padding |
| `0x0800e77e` | `sprintf_putchar` | 10B | Callback: write char to buffer |
| `0x0800e788` | `flash_write_config_block` | 44B | 16-byte config to flash 0x08040000 |
| `0x0800e7f0` | `buzzer_stop` | 14B | Clear buzzer counter |
| `0x0800e7f8` | `buzzer_tick` | 66B | Periodic buzzer toggle |
| `0x0800e84c` | `buzzer_beep_short` | 8B | Short buzzer tone (10 ticks) |
| `0x0800e858` | `can_tx_enqueue` | 42B | Insert CAN TX into ring buffer |
| `0x0800e8a8` | `eTaskGetState` | 120B | Query FreeRTOS task state |
| `0x0800ed4c` | `prvAddCurrentTaskToDelayedList` | 138B | Move task to delayed list |
| `0x0800ede0` | `prvAddNewTaskToReadyList` | 158B | Insert new TCB into ready list |

### 13.6 FreeRTOS Kernel (0x0800EE00–0x08013FFF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800ee8c` | `prvDeleteTCBCleanup` | 52B | Task deletion cleanup (idle task) |
| `0x0800eec8` | `prvCopyDataFromQueue` | 40B | Read queue item |
| `0x0800eef0` | `prvCopyDataToQueue` | 110B | Insert queue item (back/front/overwrite) |
| `0x0800ef5e` | `prvDeleteTCB` | 20B | Free task memory + TCB |
| `0x0800ef72` | `prvGetDisinheritedPriority` | 20B | Original priority for mutex |
| `0x0800ef88` | `prvHeapInit` | 62B | Initialize heap_4 free list |
| `0x0800eff8` | `prvInitialiseMutex` | 24B | Initialize mutex queue fields |
| `0x0800f010` | `prvInitialiseNewQueue` | 38B | Initialize queue control block |
| `0x0800f038` | `prvInitialiseNewTask` | 160B | Initialize TCB (stack, name, prio) |
| `0x0800f104` | `prvInitialiseTaskLists` | 74B | Scheduler lists (32 ready + delayed) |
| `0x0800f15c` | `prvInsertBlockIntoFreeList` | 78B | heap_4 free list insert (coalescing) |
| `0x0800f1b0` | `prvIsQueueEmpty` | 26B | Queue empty? |
| `0x0800f1ca` | `prvIsQueueFull` | 30B | Queue full? |
| `0x0800f1e8` | `prvListTasksWithinSingleList` | 92B | Enumerate tasks of a list |
| `0x0800f244` | `prvNotifyQueueSetMembers` | 126B | Notify queue set |
| `0x0800f2f0` | `prvResetNextTaskUnblockTime` | 28B | Reset next unblock time |
| `0x0800f310` | `prvTaskCheckFreeStackSpace` | 20B | Count stack watermark (0xA5) |
| `0x0800f390` | `prvUnlockQueue` | 124B | Process deferred queue lock |
| `0x0800f430` | `pvPortMalloc` | 196B | heap_4 first-fit allocator |
| `0x0800f538` | `prvGetMutexHolder` | 20B | Mutex holder TCB |
| `0x0800f550` | `pxPortInitialiseStack` | 30B | Task exception stack frame |
| `0x08011b24` | `uxListRemove` | 38B | FreeRTOS: remove item from list |
| `0x08011b4c` | `uxTaskGetNumberOfTasks` | 6B | Current task count |
| `0x08011b58` | `uxTaskGetSystemState` | 158B | All task status info |
| `0x08011c04` | `vListInitialise` | 22B | Initialize list |
| `0x08011c1a` | `vListInitialiseItem` | 6B | Initialize list item |
| `0x08011c20` | `vListInsert` | 48B | Sorted insert into list |
| `0x08011c50` | `vListInsertEnd` | 24B | Insert at index |
| `0x08011c68` | `vPortEnterCritical` | 50B | BASEPRI → 0x90, nesting++ (13 calls) |
| `0x08011ce8` | `vPortExitCritical` | 38B | Nesting--, clear BASEPRI (13 calls) |
| `0x08011d58` | `vPortFree` | 98B | Free heap_4 memory |
| `0x08011e00` | `vPortSetupTimerInterrupt` | 32B | Configure SysTick |
| `0x08011e24` | `vPortValidateInterruptPriority` | 70B | Validate ISR priority |
| `0x08011eb8` | `vTaskDelay` | 66B | Delay task by N ticks |
| `0x08011f30` | `vTaskGetTaskInfo` | 112B | Fill task status struct |
| `0x08011fa4` | `vTaskSetTimeOutState` | 12B | Capture timeout state |
| `0x08012060` | `vTaskMissedYield` | 8B | xYieldPending = 1 |
| `0x0801206c` | `vTaskPlaceOnEventList` | 46B | Place task on event wait list |
| `0x080120cc` | `prvTaskPriorityDisinheritAfterTimeout` | 186B | Mutex timeout priority restore |
| `0x0801224c` | `vTaskSuspend` | 176B | Suspend task |
| `0x08012338` | `vTaskEnterCritical` | 10B | uxCriticalNesting++ (8 calls) |
| `0x08012348` | `vTaskSwitchContext` | 84B | Find highest ready priority via CLZ |
| `0x08013078` | `xPortStartScheduler` | 228B | Start FreeRTOS scheduler (FPU config) |
| `0x080131b4` | `xPortSysTickHandler` | 38B | SysTick ISR: tick++, PendSV |
| `0x080131e0` | `xSemaphoreCreateMutex` | 24B | Create mutex semaphore |
| `0x080131f8` | `xQueueCreateMutex` | 78B | Allocate + init mutex queue |
| `0x08013274` | `xQueueGenericReset` | 174B | Reset queue |
| `0x08013354` | `xQueueGenericSend` | 372B | Send queue item (blocking) |
| `0x080134f8` | `xQueueGenericSendFromISR` | 224B | Send queue item from ISR |
| `0x08013604` | `xQueueGenericReceive` | 296B | Receive queue item (blocking) |
| `0x0801375c` | `xQueueSemaphoreTake` | 340B | Take semaphore (timeout) |
| `0x080138e0` | `xTaskCheckForTimeOut` | 134B | Timeout expired? |
| `0x08013a00` | `xTaskGetSchedulerState` | 24B | Scheduler status (0/1/2) |
| `0x08013a1c` | `xTaskIncrementTick` | 292B | Tick++, move delayed→ready |
| `0x08013b74` | `prvTaskPriorityDisinherit` | 166B | Mutex release priority restore |
| `0x08013c50` | `prvTaskPriorityInherit` | 172B | Mutex take priority inherit |
| `0x08013d04` | `xTaskRemoveFromEventList` | 206B | Move event→ready (7 calls) |
| `0x08013e0c` | `xTaskResumeAll` | 280B | Resume scheduler (8 calls) |

### 13.7 Letter-Shell Library (0x0800F500–0x080110FF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800f5c0` | `shell_register_instance` | 26B | Register shell instance |
| `0x0800f628` | `shell_check_permission` | 48B | Check user permissions for command |
| `0x0800f658` | `shell_print_logo` | 24B | Print shell banner |
| `0x0800f674` | `shell_clear_input_line` | 42B | Clear terminal input line |
| `0x0800f6a6` | `shell_delete_char` | 214B | Delete character (forward/backward) |
| `0x0800f788` | `shell_password_input` | 22B | Password input + callback |
| `0x0800f7a0` | `shell_password_verify` | 162B | Deferred password check |
| `0x0800f814` | `shell_detect_number_base` | 78B | Detect number base (dec/bin/oct/hex/float) |
| `0x0800f862` | `shell_parse_escape_char` | 60B | Parse C escape sequences (\n, \r, \t) |
| `0x0800f89e` | `shell_parse_number` | 194B | Parse numeric string (int/float) |
| `0x0800f960` | `shell_parse_argument` | 94B | Parse single argument |
| `0x0800f99c` | `shell_parse_string` | 70B | Parse string with quotes |
| `0x0800fa04` | `shell_invoke_command` | 204B | Dispatch command with 0-7 arguments |
| `0x0800fad8` | `shell_hex_char_to_val` | 44B | Hex ASCII → numeric |
| `0x0800fb04` | `shell_entry_get_desc` | 10B | Entry description (offset+0xC) |
| `0x0800fb10` | `shell_entry_get_name` | 60B | Entry name (offset+4) |
| `0x0800fb50` | `shell_get_active_instance` | 36B | First active shell instance |
| `0x0800fb78` | `shell_get_variable_value` | 56B | Read variable (ptr/val/getter) |
| `0x0800fc90` | `shell_cmd_help` | 40B | `help` command handler |
| `0x0800fcb8` | `shell_history_navigate` | 154B | Navigate command history (↑/↓) |
| `0x0800fd52` | `shell_history_push` | 108B | Command into history ring (5 entries) |
| `0x0800fdc0` | `shell_init` | 118B | Initialize shell context |
| `0x0800ff10` | `shell_backspace_display` | 26B | Backspace display |
| `0x0800ff30` | `shell_list_commands` | 70B | List all permitted commands |
| `0x0800ff7c` | `shell_print_command_entry` | 154B | Print command entry formatted |
| `0x08010034` | `shell_parse_args` | 118B | Split input into max. 8 tokens |
| `0x080100aa` | `shell_strip_quotes` | 66B | Remove quotation marks |
| `0x08010102` | `shell_command_dispatch` | 146B | Type-based command dispatch |
| `0x08010194` | `shell_user_search` | 116B | Search user entry (16-byte iterator) |
| `0x08010208` | `shell_login_handler` | 90B | Login handler (3 password paths) |
| `0x08010268` | `shell_print_variable` | 154B | Display variable (dec+hex) |
| `0x08010324` | `shell_common_prefix_len` | 36B | Common prefix for tab completion |
| `0x08010348` | `shell_strcpy` | 24B | Byte-by-byte string copy |
| `0x08010360` | `shell_tab_complete` | 314B | Tab completion (single/multi/double tab) |
| `0x080104a4` | `shell_int_to_dec` | 82B | Int → decimal string |
| `0x080104f6` | `shell_int_to_hex` | 42B | Int → hex string |
| `0x08010526` | `shell_write_char` | 12B | Single character via callback |
| `0x08010534` | `shell_write_string` | 76B | Output string (max 150, "..." section) |
| `0x08010584` | `shell_show_command_help` | 88B | Detailed help for command |
| `0x080105e4` | `shell_print_prompt` | 78B | Output "user:path$ " prompt |
| `0x08010648` | `shell_print_return_value` | 98B | Output "Return: <dec>, 0x<hex>" |
| `0x080106d0` | `shell_print` | 40B | UART string output (13 calls) |

### 13.8 Modbus Register & Inverter Control (0x08011100–0x08015FFF)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0801178c` | `SPI_DMA_Init` | 150B | Initialize SPI with DMA |
| `0x0801182c` | `watchdog_feed_and_decrement` | 4B | Feed watchdog + safety counter |
| `0x08011830` | `decrement_safety_counter` | 24B | Decrement DAT_2000046c counter |
| `0x08011854` | `TIM3_PWM_Init` | 128B | Initialize TIM3 PWM (period 8499) |
| `0x080118e8` | `modbus_read_register_block` | 526B | Read register block with float→int scaling |
| `0x080124e8` | `modbus_register_handler` | 2842B | **Massive Modbus register R/W handler** (largest function!) |
| `0x080150d4` | ~~`CAN_TxMailbox_SetDLC`~~ ❌ | 10B | **Refuted** — never applied in Ghidra (stale documentation from 07.07). Remains `FUN_080150d4`, see 13.18/13.21: real MOVW/MOVT/BX interworking veneer, jumps into the `UART_WaitOnFlagUntilTimeout` tail |
| `0x080150e8` | ~~`CAN_RxFilter_Config`~~ ❌ | 10B | **Refuted** — ditto. Remains `FUN_080150e8`, see 13.18/13.21: veneer, jumps into the `UART_WaitOnFlagUntilTimeout` tail |
| `0x080150fc` | ~~`float_to_fixed_point`~~ ❌ | 10B | **Refuted** — ditto. Remains `FUN_080150fc`, see 13.18/13.21: veneer, jumps into the `RCC_OscConfig` tail, caller `Inverter_Grid_Control` (8×) |
| `0x080151c4` | ~~`CAN_frame_pack`~~ ❌ | 10B | **Refuted** — ditto. Remains `FUN_080151c4`, see 13.18/13.21: veneer, jumps into the `RCC_OscConfig` tail |
| `0x08015450` | `inverter_grid_control` | 836B | Grid connection control (BMS faults, protection) |
| `0x08015818` | `atan_normalized` | 140B | Polynomial atan(x)/π approximation |
| `0x0801590a` | `biquad_filter_design` | 94B | Compute biquad IIR filter coefficients |
| `0x0801598c` | `PI_controller_step` | 116B | PI controller: P + I with clamping |
| `0x08015a00` | `PI_controller_reset` | 34B | Reset PI controller state |
| `0x08015b0c` | `compute_reactive_power_ref` | 164B | Compute reactive power reference from PF |
| `0x0801b2fc` | `notch_filter_design` | 126B | 2nd-order notch filter (Ts=35.7µs) |
| `0x0801ba34` | `grid_freq_filter_update` | 72B | Biquad+notch coefficients for grid frequency |

### 13.9 Correction Session 10.07.2026 — Actually Verified & Renamed in Ghidra

> Starting point: Ghidra actual state before this session = **21 named functions** (445 total,
> `filterDefaultNames=true` returned 21). All other names in 13.1–13.8 were only documentation proposals
> from the never-checked-in 07.07. analysis. In this session, **106 functions were individually verified
> via decompilation** (signature, caller context, register/address constants checked) and
> renamed via Ghidra script (`setName`, `SourceType.USER_DEFINED`). New Ghidra actual state:
> **127 named functions** (21 + 106), 0 name collisions. 318 functions are still `FUN_*`.

**Duplicate fix:** `CAN_Filter_Setup` existed twice — the real function `0x0800eb88` (260B, configure CAN filter,
9 callees) and a 10-byte thunk at `0x080150f2` (48 callers, jump to `0x0800eb88`
confirmed via `resolve-thunk`). The thunk was renamed to `thunk_CAN_Filter_Setup`.

**106 newly assigned names (address → name, confidence):**

*High (register constants / algorithm clearly recognizable):*
`0x08000332 memcpy`, `0x08000356 __aeabi_memset`, `0x08000364 memclr`, `0x08000368 memset`,
`0x0800037a strlen`, `0x08000388 strcmp`, `0x080003a4 strcpy`, `0x080003d4 __aeabi_ui2d`,
`0x0800042c __aeabi_uidiv`, `0x08000458 __aeabi_llsl`, `0x08000476 __aeabi_llsr`,
`0x080002d0 __aeabi_uldivmod`, `0x08000496 __aeabi_dadd`, `0x080005e4 __aeabi_dmul`,
`0x080006c8 __aeabi_ddiv`, `0x080007a6 __aeabi_d2lz`, `0x0800082c __aeabi_lasr`,
`0x0800086e __aeabi_dnorm`, `0x08000234 FPU_Enable` (CPACR |= 0xF00000),
`0x0800abda CRC16_Modbus` (poly 0xA001, exact), `0x08009a34 IWDG_Reload` (`*reg=0xAAAA`),
`0x0800325c FLASH_ProgramDoubleWord`, `0x080031d4 FLASH_MassErase`, `0x08003210 FLASH_SectorErase`,
`0x08003178 FLASH_FlushCaches` (FLASH_ACR 0x40022000), `0x080032a4 FLASH_WaitForOperation`,
`0x080057ac HAL_GetTick`, `0x0800675c NVIC_EnableIRQ` (NVIC->ISER bit-band, **corrected**, see below),
`0x08006dc0 HAL_RCC_GetSysClockFreq` (RCC_CFGR SWS bits, **corrected**, see below),
`0x08006800 DAC_Init` (DAC_BASE 0x40007000), `0x0800afa4 BCD_To_Decimal`,
`0x080099ec uint_to_decimal_string`, `0x08000ef0 ADC_ConvertRawValues` (scaling 0.21606445,
target = SRAM addresses from table 5.2), `0x08001158 ADC_ProcessSamples` (target = table 5.1),
`0x080012e0 Grid_Protection_SetLimits` (grid_standard-dependent float constants),
`0x08002ce8 EEPROM_WriteRegister` / `0x08002d84 EEPROM_ReadRegister` (I2C addr. 0xA0/0xA2),
`0x08002a68 EEPROM_LoadConfig`, `0x080020d0 CAN_SetFilter_ExtID` / `0x08002108 CAN_SetFilter_StdID`
(directly from `CAN_Filter_Setup` with values `0x4000/0xFF00` etc. — matches table 4.1 exactly),
`0x08001f34 CAN_TX_SendMessage` (segmentation `param_4>>3`, 8-byte frames),
`0x0800ac0c Modbus_Process_Request` (CRC16_Modbus call confirmed),
`0x0800b120 Modbus_Read_Holding_Registers` (FC03 branch in Modbus_Process_Request),
`0x0800b29c Modbus_Write_Multiple_Registers` / `0x0800b374 …Broadcast_Write_Multi_Regs` (FC16 branch),
`0x0800ca3c Operating_Hours_Update` (hour counter, target SRAM = daily_charge/discharge_energy from 11.1),
`0x0800ab88 debug_log_enqueue` / `0x08009b48 debug_log_dequeue` (ring buffer 0x20001A2C,
counterpart confirmed, `debug_log_enqueue` additionally verified from `Mode3_RoundRobin_Timer` as per section 12.3),
`0x0800b8d4 SysTick_Handler_App` (increments the same tick as `Timer_Elapsed_Ms`),
`0x0800c0d8 Timer_Elapsed_Ms` / `0x0800c100 Timer_Elapsed_Sec`, `0x08003670 HAL_GPIO_Init_Extended`
(common target function of `GPIO_ConfigPin`/`_HighSpeed`), `0x08000bdc GPIO_ConfigPin` /
`0x08000bfc GPIO_ConfigPin_HighSpeed`, `0x08000c20 Peripheral_GPIO_Init`.

*Medium (context/callers consistent, details not 100% verified):*
`0x080002a4 get_current_irq_number`, `0x080003b6 strncmp`, `0x0800273e memcpy_reverse`,
`0x08002a3c delay_ms`, `0x08003078 Error_Handler`, `0x0800343c checksum_add`, `0x08003458 checksum_xor`,
`0x08000a28 UART_IRQHandler`, `0x08000aa6 UART_ErrorHandler`, `0x08000ac0 UART_DMA_Handler`,
`0x08000acc UART_WaitReady`, `0x08002024 CAN_TX_SendCommand`, `0x080020b0 CAN_BuildArbID`,
`0x08001e3c CAN_TX_ReadQueue`, `0x08001f04 CAN_RX_DispatchTask`, `0x08002034 CAN_TX_SendFrame`,
`0x08002070 CAN_TX_ProcessQueue`, `0x08003090 CAN_InitMailboxes`, `0x08003104 CAN_CopyTxFrame`,
`0x08003278 FLASH_Program256Bit`, `0x08003310 FLASH_EraseSectors`, `0x08003378 FLASH_WriteData`,
`0x080033f0 Get_GridFrequency`, `0x0800213c OTA_FW_Update_StateMachine`, `0x08002614 Serial_ValidatePacket`,
`0x0800277c DMA_CalcBaseAndOffset`, `0x080027e0 DMA_SetConfig`, `0x0800299c Inverter_SetMode`,
`0x080029fc EEPROM_ClearStats`, `0x08002e14 EEPROM_SaveTimestamp`, `0x08002e40 EEPROM_WriteVerify`,
`0x08002f78 EEPROM_WriteVerify_NoMutex`, `0x0800ad40 NTC_ADC_To_Temperature_1` /
`0x0800adcc NTC_ADC_To_Temperature_2` (caller = `ADC_ProcessSamples`), `0x0800ae58 CAN_Send_Packet`,
`0x0800b400 RTC_Read_Time` (BCD_To_Decimal calls confirmed), `0x0800b540 Ymodem_Send_Byte`,
`0x0800c94c CAN_Start_FW_Update`, `0x0800af54 I2C_Get_Clock_Freq`, `0x0800afb6 I2C_Wait_SB_Flag`,
`0x0800affe I2C_Wait_ADDR_Flag`, `0x0800b050 Ymodem_Receive_Packet`, `0x0800b80c Ymodem_Parse_File_Header`,
`0x08009850 UART_Transmit_Wait`, `0x080098f8 UART_Receive_Wait`, `0x08009944 UART_Transmit_Wait_Alt`,
`0x0800ad14 Modbus_UART_Start_Receive`, `0x0800cb10 UART5_Send_Byte`.

**Corrections relative to the old documentation proposal (name was wrong, reassigned after verification):**

| Address | Old documentation name (wrong) | New name (verified) | Rationale |
|---|---|---|---|
| `0x0800675c` | `IWDG_SetPrescaler` | `NVIC_EnableIRQ` | Writes `NVIC->ISER[n/32] = 1<<(n%32)` — classic NVIC bit-band enable pattern, not an IWDG register |
| `0x08006dc0` | `DAC_SetValue` | `HAL_RCC_GetSysClockFreq` | Reads `RCC_CFGR` SWS bits (`0x40021008`), returns 16 MHz/8 MHz — classic clock source detection, no DAC access |
| `0x08009994` | `USART2_Init` | `I2C1_Init` | Writes `0x40005400` (= I2C1 base on STM32F4, not USART2 = `0x40004400`); handle is used directly by the confirmed EEPROM I2C functions |
| `0x0800b208` | `Modbus_Read_Input_Registers` | `Modbus_Write_Single_Register` | Writes a value (`FUN_080124e8(addr, value, 1, …)`), reads nothing — FC06 is Modbus "Write Single Register", not "Read" |
| `0x0800b26c` | `Modbus_Broadcast_Read_Input_Regs` | `Modbus_Broadcast_Write_Single_Register` | Broadcast variant of the same write function |

**Rejected (documentation name clearly refuted, deliberately NOT renamed — remains `FUN_<address>`):**

| Address | Documentation name (discarded) | Reason |
|---|---|---|
| `0x0800020c` | `NVIC_SystemReset` | Decompilation shows privilege escalation + SVC(0) + FPU enable pattern (CMSIS-RTOS kernel start-like), no SCB->AIRCR write operation |
| `0x080007d8` | `__aeabi_d2iz` | Only 9 lines, pure sign negation — too simple for a full double→int conversion |
| `0x08000b50` | `CAN_WaitReady` | Caller is `FUN_08003ff8` (UART/DMA context), no CAN references in the code |
| `0x0800150a` | `NVIC_SetPriority` | Calls `FUN_080055c4` with magic constant `0x110000` — structure does not match direct NVIC_IPR register access |
| `0x08001532` | `NVIC_EnableIRQ` | Same pattern as `0x0800150a`; the real `NVIC_EnableIRQ` was found at `0x0800675c` instead |
| `0x08000fb4` | `CAN_Peripheral_Init` | Writes `0x50000000` (USB OTG FS base on STM32F4) into the target structure — no CAN reference |
| `0x08001ea0` | `UART_Init` | Writes `0x40006400` (= CAN1 base) into `DAT_20001270` — exactly the handle that `CAN_SetFilter_ExtID/StdID` use. Is presumably the real CAN init, not UART |
| `0x08001868` | `CAN_RX_ParseBMSCommand` | Caller context (`FUN_08005248`, documented there as "DMA_Start_IT") does not match BMS/CAN |
| `0x08002824` | `TIM6_PWM_Init` | Writes `0x50000800` — not the TIM6 base (`0x40001000`) |
| `0x080035d4` | `UART_SetBaudRate` | Called several times with identical constant `0x7f` on different pointers — does not match "set one baud rate"; hangs off the questionable `0x08000fb4` chain |
| `0x08003af0`, `0x08003b04`, `0x08003d54`, `0x08003ff8`, `0x08004128` | `UART_RX_Callback`, `HAL_UART_Init`, `UART_MspInit`, `UART_StartReceive_DMA`, `HAL_TIM_Base_Init_Full` | The entire cluster hangs off `0x08000fb4`; several functions reference `0x50000000`/`0x50000100`/`0x50000300` (USB OTG FS address space) — presumably unused USB driver code, not UART |
| `0x08005384` | `ADC_Init` | Caller is the confirmed `FLASH_EraseSectors` (`0x08003310`) — no ADC reference |
| `0x08005774`, `0x08005782`, `0x08005792`, `0x080057a0` | `HAL_RCC_GetPCLK1Freq`, `…PCLK2Freq`, `…GetHCLKFreq`, `…GetSysClockFreq` | Decompilations are generic bit-field setters/testers without frequency calculation; the real system clock function was found at `0x08006dc0` |
| `0x08005c10` | `I2C_Init` | Generic struct init, no I2C-specific registers recognizable |
| `0x080066d8` | `IWDG_WaitForReady` | Callers are HRTIM/DAC functions, no IWDG reference |
| `0x08009a4c`, `0x08009a54`, `0x08009a5c` | `DMA_Get_TCIF`, `…Error_Flag`, `…HTIF` | Caller is the confirmed `HAL_GPIO_Init_Extended` — bit fields match GPIO MODER extraction, not DMA flags |

**Open for follow-up session (status before tranche 2c):** 318 functions were still `FUN_0800xxxx`/`FUN_0801xxxx` (445 − 127).
In particular the address range approx. `0x08004caa`–`0x08009800` (TIM/HRTIM/RCC/ADC/I2C in 13.3) shows
systematic address drift between documentation proposal and Ghidra reality and should be completely re-verified
rather than taken over from the old table.

### 13.10 Tranche 2c (10.07.2026) — Address Range 0x08010034–0x0801ba34

> Processed range: upper FW range (Letter-Shell library, FreeRTOS kernel/tasks surroundings,
> CAN filter surroundings, Modbus register handler, inverter control/DSP block). Methodology as in
> 13.9: each function individually decompiled and verified via `get-decompilation` (incl. caller context),
> not blindly taken over from section 13.7/13.8. Ghidra actual state before tranche 2c:
> 127 named functions. After tranche 2c: **193 named functions** (127 + 66), 0 name collisions
> (confirmed by script duplicate check against all 193 names).
>
> **Particularly valuable verification source in this range:** The FreeRTOS kernel code contains
> embedded debug strings with original source file paths in many places, e.g.
> `"..\\..\\SDK\\FreeRTOS\\portable\\RVDS\\ARM_CM4F\\port.c"`, `"..\\..\\SDK\\FreeRTOS\\tasks.c"`,
> `"..\\..\\SDK\\FreeRTOS\\queue.c"`, `"..\\..\\SDK\\FreeRTOS\\portable\\MemMang\\heap_4.c"`
> (each with line number as the second `debug_printf` argument). These strings allow verification of the FreeRTOS kernel functions at
> nearly ground-truth level (file+line match the known FreeRTOS source structure) — significantly more reliable than pure register-constant
> heuristics. All FreeRTOS names classified as "high" below are confirmed via this mechanism.

**66 newly assigned names (address → name, confidence):**

*Letter-Shell library (0x08010034–0x080106d0, high — parameter/caller structure matches exactly the
shell context struct from section 7, `shell_login_handler` additionally confirmed by the assembly verification already documented in
7.1):*
`0x08010034 shell_parse_args`, `0x080100aa shell_strip_quotes`, `0x08010102 shell_command_dispatch`,
`0x08010194 shell_user_search`, `0x08010208 shell_login_handler`, `0x08010268 shell_print_variable`,
`0x08010324 shell_common_prefix_len`, `0x08010348 shell_strcpy`, `0x08010360 shell_tab_complete`,
`0x080104a4 shell_int_to_dec`, `0x080104f6 shell_int_to_hex`, `0x08010526 shell_write_char`,
`0x08010534 shell_write_string`, `0x08010584 shell_show_command_help`, `0x080105e4 shell_print_prompt`,
`0x08010648 shell_print_return_value`, `0x080106d0 shell_print`.

*FreeRTOS kernel (high — source file+line confirmed from embedded debug strings, see above;
exceptions without string evidence but with a clear structural pattern also high):*
`0x08011b24 uxListRemove`, `0x08011b4c uxTaskGetNumberOfTasks`, `0x08011b58 uxTaskGetSystemState`,
`0x08011c04 vListInitialise`, `0x08011c1a vListInitialiseItem`, `0x08011c20 vListInsert`,
`0x08011c50 vListInsertEnd`, `0x08011c68 vPortEnterCritical` (port.c:0x1b9),
`0x08011ce8 vPortExitCritical` (port.c:0x1c0), `0x08011d58 vPortFree` (heap_4.c:0x12f/0x130),
`0x08011e00 vPortSetupTimerInterrupt`, `0x08011e24 vPortValidateInterruptPriority` (port.c:0x34d/0x35d),
`0x08011eb8 vTaskDelay` (tasks.c:0x51c), `0x08011f30 vTaskGetTaskInfo`, `0x08011fa4 vTaskSetTimeOutState`,
`0x08012060 vTaskMissedYield`, `0x0801206c vTaskPlaceOnEventList` (tasks.c:0xc0e),
`0x080120cc prvTaskPriorityDisinheritAfterTimeout` (tasks.c:0x1098/0x10b3),
`0x0801224c vTaskSuspend`, `0x08012338 vTaskEnterCritical`,
`0x08012348 vTaskSwitchContext` (LZCOUNT on ready list bitmap, tasks.c:0xbf6),
`0x08013078 xPortStartScheduler` (port.c:0x146/0x147/0x160), `0x080131b4 xPortSysTickHandler`,
`0x080131e0 xSemaphoreCreateMutex`, `0x080131f8 xQueueCreateMutex` (queue.c:0x1e4),
`0x08013274 xQueueGenericReset` (queue.c:0x12e), `0x08013354 xQueueGenericSend`,
`0x080134f8 xQueueGenericSendFromISR` (queue.c:0x420–0x422), `0x08013604 xQueueGenericReceive` (queue.c:0x56b),
`0x0801375c xQueueSemaphoreTake` (queue.c:0x5fd), `0x080138e0 xTaskCheckForTimeOut` (tasks.c:0xcef/0xcf0),
`0x08013a00 xTaskGetSchedulerState`, `0x08013a1c xTaskIncrementTick` (tasks.c:0xab7),
`0x08013b74 prvTaskPriorityDisinherit` (tasks.c:0x1048/0x1049), `0x08013c50 prvTaskPriorityInherit`,
`0x08013d04 xTaskRemoveFromEventList` (tasks.c:0xc74), `0x08013e0c xTaskResumeAll` (tasks.c:0x885).

*Modbus / Safety (high — callers are already verified Modbus functions or the structure matches exactly
the ModbusRegDescriptor struct from section 16.2):*
`0x080118e8 modbus_read_register_block` (descriptor struct + scaling switch 1/2/3/4 exactly as in
16.2), `0x080124e8 modbus_register_handler` (callers = all 5 already verified Modbus FC03/06/16
functions; see section 16 — same function, now additionally renamed in Ghidra itself),
`0x08011830 decrement_safety_counter` (decrements `DAT_2000046c`, caller `Mode3_RoundRobin_Timer`).

*Safety (medium):* `0x0801182c watchdog_feed_and_decrement` (4-byte stub directly before
`decrement_safety_counter`, falls through into the same function; calls a still-unnamed function
outside this range, therefore watchdog reference plausible but not 100% verified).

*Inverter control / DSP (high — algorithm/constants clear):*
`0x08015450 Inverter_Grid_Control` (836B, BMS fault bitmask + grid connection logic + several
`thunk_CAN_Filter_Setup` calls with status codes — size exactly as in the old documentation proposal),
`0x08015818 atan_normalized` (polynomial approximation, coefficient table at external flash `0x10006b00`,
division by π at the end), `0x0801590a biquad_filter_design` (classic biquad coefficient calculation
from cutoff frequency+Q, 5 coefficients b0/b1/b2/a1/a2 pattern),
`0x0801598c PI_controller_step` (P+I with clamping between max/min fields of the controller struct),
`0x08015a00 PI_controller_reset` (zeroes all state fields of the PI struct),
`0x08015b0c compute_reactive_power_ref` (sqrt(1/pf²−1) calculation, classic PF→Q conversion),
`0x0801b2fc notch_filter_design` (2nd order, Ts=3.5714285e-05s=35.7µs — exactly as in the old
documentation proposal), `0x0801ba34 grid_freq_filter_update` (calls `biquad_filter_design` +
`notch_filter_design` with grid frequency parameter).

**Deliberately deferred (remain `FUN_<address>`, not renamed):**

| Address | Reason |
|---|---|
| `0x0801178c` | Combined DAC/HRTIM init (calls `DAC_Init`, 2× `FUN_08006e34`, `FUN_08006bec` — all callees lie outside this range and are not yet verified by tranche 2a/2b; documentation proposal "SPI_DMA_Init" is clearly refuted by the decompilation — no SPI base address in the code), purpose remains unclear |
| `0x08011854` | HRTIM capture setup with GPIO/EXTI configuration and `NVIC_EnableIRQ(0x1e)` — documentation proposal "TIM3_PWM_Init" refuted (no TIM3 base address `0x40000400` in the code, HRTIM callees instead); possible zero-crossing detection, but not certain enough for renaming |
| `0x08011b10` | Trivial 2-instruction stub (`_DAT_20000458 = _DAT_20000024`), no recognizable semantic name derivable |
| `0x080150d4`, `0x080150e8`, `0x080150fc`, `0x080151c4` | 10 bytes each, decompilation uses `unaff_r4`/`unaff_r5`/`unaff_r6` registers (Ghidra artifact for code fragments without a clean function entry) — presumably jump targets wrongly recognized as separate functions within the interleaved shell command table area (cf. note in section 7.1 on the RVDS/Keil code/data interleave). Old documentation proposals (`CAN_TxMailbox_SetDLC` etc.) are not verifiable and were deliberately not adopted |

**Already named before tranche 2c (unchanged, mentioned only for completeness in this range):**
`0x080150f2 thunk_CAN_Filter_Setup`, `0x08013998 xTaskCreate`, `0x080163d0 vtask_can`,
`0x08016424 vtask_modbus`.

### 13.11 Tranche 2b (10.07.2026) — address range 0x0800803c–0x0800ffff

> Processed range: middle FW range (HRTIM/TIM/SPI/I2C peripheral driver tail,
> UART DMA/IRQ handlers, Modbus/CAN surroundings, buzzer control, printf/sprintf engine, OTA Ymodem
> download, complete FreeRTOS kernel block `0x0800ed4c`–`0x0800f550` as well as a large part of the
> letter-shell library `0x0800f5c0`–`0x0800ff7c`). Methodology as in 13.9/13.10: every function decompiled individually
> via `get-decompilation` (including caller context), documentation suggestions from 13.2–13.7 not
> adopted blindly. Ghidra actual state before tranche 2b: 193 named functions (after tranche 2c,
> processed in parallel). Tranche 2b itself newly named **94 functions** (80 in the first pass +
> 14 in the addendum, see below). After tranche 2b: **309 named functions*** (confirmed by script duplicate check
> against all names in the entire program: 0 collisions). *Total including parallel
> tranche 2a/2c contributions from other agent runs in the same Ghidra project.
>
> **Most valuable verification source:** As in tranche 2c, this address range also contains the
> complete FreeRTOS kernel core (`heap_4.c`, `tasks.c`, `queue.c`) with embedded
> debug path strings (`"..\\..\\SDK\\FreeRTOS\\tasks.c"` at `0x35d`/`0x392`/`0x54a`,
> `"..\\..\\SDK\\FreeRTOS\\queue.c"` at `0xbe3`/`0xbe4`/`0xc04`) — all 20 FreeRTOS functions in
> this range (`prvAddCurrentTaskToDelayedList` … `pxPortInitialiseStack`) are thereby verified at
> near ground-truth level.

**94 newly assigned names (address → name, confidence) — 80 in the first pass, 14 in the addendum:**

*High (source file string, unambiguous register pattern or directly verified caller context):*

FreeRTOS kernel (heap_4.c/tasks.c/queue.c strings confirmed):
`0x0800ed4c prvAddCurrentTaskToDelayedList`, `0x0800ede0 prvAddNewTaskToReadyList`,
`0x0800ee8c prvDeleteTCBCleanup`, `0x0800eec8 prvCopyDataFromQueue`, `0x0800eef0 prvCopyDataToQueue`,
`0x0800ef5e prvDeleteTCB`, `0x0800ef72 prvGetDisinheritedPriority`, `0x0800ef88 prvHeapInit`,
`0x0800eff8 prvInitialiseMutex`, `0x0800f010 prvInitialiseNewQueue`,
`0x0800f038 prvInitialiseNewTask` (tasks.c:0x35d/0x392), `0x0800f104 prvInitialiseTaskLists`,
`0x0800f15c prvInsertBlockIntoFreeList`, `0x0800f1b0 prvIsQueueEmpty`, `0x0800f1ca prvIsQueueFull`,
`0x0800f1e8 prvListTasksWithinSingleList`, `0x0800f244 prvNotifyQueueSetMembers` (queue.c:0xbe3/0xbe4/0xc04),
`0x0800f2f0 prvResetNextTaskUnblockTime`, `0x0800f310 prvTaskCheckFreeStackSpace`,
`0x0800f390 prvUnlockQueue`, `0x0800f40c prvWriteNameToBuffer` (newly identified, not in the old
documentation suggestion — strcpy + space padding to 15 characters, classic FreeRTOS
`prvWriteNameToBuffer` pattern), `0x0800f430 pvPortMalloc` (heap_4.c string directly in the code),
`0x0800f538 prvGetMutexHolder`, `0x0800f550 pxPortInitialiseStack`.

Letter-shell (structure/caller matches the shell context struct from section 7 exactly,
`shell_init` additionally confirmed by cross-reference with the already verified `shell_login_handler`
from tranche 2c):
`0x0800f5c0 shell_register_instance`, `0x0800f628 shell_check_permission`,
`0x0800f658 shell_print_logo`, `0x0800f7a0 shell_password_verify`, `0x0800fb04 shell_entry_get_desc`,
`0x0800fb10 shell_entry_get_name`, `0x0800fc90 shell_cmd_help`, `0x0800fdc0 shell_init`
(calls `shell_register_instance`, `shell_user_search`, `shell_login_handler`, `shell_print_prompt`),
`0x0800ff10 shell_backspace_display`, `0x0800ff30 shell_list_commands`,
`0x0800ff7c shell_print_command_entry`.

UART/I2C peripherals, miscellaneous:
`0x08008330 UART_AbortReceive`, `0x080083f8 UART_IRQHandler_Full`, `0x080086e4 HAL_UART_Receive_IT`,
`0x08008834 HAL_UART_Transmit_DMA`, `0x080096b8 UART_WaitOnFlagUntilTimeout`,
`0x080097c8 I2C_TransferConfig` (I2C_CR2 bit pattern SADD/NBYTES/RELOAD/AUTOEND),
`0x080097f8 I2C_WaitOnFlagUntilTimeout`, `0x08009694 UART_EndTransfer`,
`0x08009ba0 debug_log_format_entry` (1712B, largest function in the range, ~60 event types in the
switch — exactly as in the old documentation suggestion), `0x0800c7f8 CAN_Wait_Response` (Timer_Elapsed_Ms timeout
3000ms + xQueueGenericReceive call), `0x0800cbd8 Shell_Task_Init` (calls `shell_init` with
shell context `0x20001410`/buffer `0x2000147c`/512B — exactly as in section 7.5),
`0x0800dc40 ota_firmware_download` (Ymodem_Receive_Packet loop + flash target `0x08044000`),
`0x0800de28 debug_sprintf_wrapper` (calls `printf_core` + `sprintf_putchar`),
`0x0800de72 NVIC_SetPriority` (writes `(&DAT_e000e400)[irq] = prio<<4` — classic
CMSIS NVIC_IPR pattern; **Note:** this is the actual real NVIC_SetPriority function,
to be distinguished from the refuted `0x0800150a` from tranche 2a/13.9),
`0x0800de94 assert_failed_halt` (double DSB + infinite loop), `0x0800e050 printf_core`
(1696B format engine, callers `debug_printf`/`debug_sprintf_wrapper`), `0x0800e72c printf_pad_trailing`,
`0x0800e750 printf_pad_leading`, `0x0800e77e sprintf_putchar` (classic buffer putc callback),
`0x0800e788 flash_write_config_block` (16-byte struct including `checksum_add` call →
`FLASH_WriteData(0x08040000, …)`), `0x0800e7f0 buzzer_stop`, `0x0800e7f8 buzzer_tick`
(cross-reference: decrements the same counter `0x20000464` that `buzzer_stop` zeroes),
`0x0800e84c buzzer_beep_short` (sets `0x20000468 = 10`), `0x0800e858 can_tx_enqueue`
(ring buffer `0x2000399c`, caller `CAN_TX_SendMessage`), `0x0800e8a8 eTaskGetState`
(tasks.c string at `0x54a`, checks `pxCurrentTCB`/ready/suspended lists),
`0x0800bf94 TIM1_PWM_Init` (base `0x40012c00`=TIM1, period 399, duty 200 — callees
`TIM_Handle_Init`/`TIM_OC_ConfigChannel`/`TIM_Channel_SetState` already named by tranche 2a),
`0x0800c078 TIM1_Set_Duty` (clamp 0–400, writes CCR of the TIM1 config from `TIM1_PWM_Init`,
callers include `modbus_register_handler`), `0x0800cb50 USART1_DMA_Send` (calls
`HAL_UART_Transmit_DMA`), `0x0800cb80 USART2_Poll_Send` (base `0x40004400`=USART2 confirmed).

**Addendum (second pass, previously overlooked block `0x0800bf94`–`0x0800cbac`):** During
cross-checking of the TIM/SPI/USART cluster directly before the already named `ADC_Sensor_Debug_Print`
(`0x0800cc40`), 14 further functions were identified, decompiled and renamed. Two
important corrections to the old documentation suggestions from section 13.4 resulted:
- **Peripheral addresses incorrectly assigned:** `0x0800c098` (period 999, as claimed in the old
  documentation suggestion "TIM5_Init") actually lies on base `0x40001400` = **TIM7**, not
  TIM5 (`0x40000c00`) → renamed to `TIM7_Init`. `0x0800c008` (old suggestion "TIM8_PWM_Init_Encoder")
  lies on base `0x40015000` = **TIM20** (this MCU family has no TIM8, but uses HRTIM+
  TIM1/15/16/17/20 as evidenced in section 13.11 above) → renamed to `TIM20_Init` (encoder nature
  cannot be confirmed beyond doubt, hence without the "_Encoder" suffix).
- **"SPI_DMA_*" names refuted:** The complete cluster `0x0800c128`–`0x0800c32c` (old
  documentation suggestion: `SPI_DMA_Complete_IRQ`, `SPI_DMA_Start_Receive`, `SPI_DMA_RX_Disable`,
  `SPI_DMA_TX_Disable`, `SPI_DMA_RX_IRQ_Handler`, `SPI_DMA_TX_IRQ_Handler`) uses exactly the same
  bitmask patterns (`0xfffffedf`/`0xeffffffe` via `ExclusiveAccess`/`hasExclusiveAccess` bit-band) as
  the already verified `UART_AbortReceive` (`0x08008330`) — and `0x0800c1e4` is additionally
  called directly from `UART_IRQHandler_Full`. It is therefore UART DMA transfer handling,
  not SPI. New names: `0x0800c128 UART_DMA_TxRxComplete_Check`, `0x0800c178 UART_DMA_RX_Start`,
  `0x0800c1e4 UART_DMA_RX_Disable`, `0x0800c25c UART_DMA_TX_Disable`,
  `0x0800c28a UART_DMA_RX_IRQ_Handler`, `0x0800c32c UART_DMA_TX_IRQ_Handler`. Additionally
  `0x0800c0c4 UART_ResetXferCounters` (zeroes the same struct fields `+0x56`/`+0x5e` that are used in the
  UART DMA cluster as Tx/RxXferCount — previously wrongly deferred as "too generic" (see
  struck-through line below), but can be assigned reliably after context reconciliation with the cluster)
  and `0x0800cbac USART1_Poll_Send` (base `0x40013800` = **USART1 on this
  STM32F3-series MCU** [not the F4 mapping, where the same address would be SYSCFG] — correction compared to the
  old documentation suggestion "UART5_Poll_Send").

Confidence: `TIM1_PWM_Init`, `TIM1_Set_Duty`, `USART1_DMA_Send`, `USART2_Poll_Send` = high (see
above). `TIM7_Init`, `TIM20_Init`, `UART_ResetXferCounters`, `UART_DMA_TxRxComplete_Check`,
`UART_DMA_RX_Start`, `UART_DMA_TX_Disable`, `UART_DMA_RX_IRQ_Handler`, `UART_DMA_TX_IRQ_Handler`,
`USART1_Poll_Send` = medium (peripheral base confirmed, but detailed semantics not 100% verified).
`UART_DMA_RX_Disable` = medium-high (additionally confirmed by the direct call from `UART_IRQHandler_Full`).

*Medium (context consistent, details not 100% verified):*
`0x0800873c UART_DMA_RxCplt_Dispatch` (dispatches by peripheral base address USART1 `0x40013800`/
UART5 `0x40004400` to `xQueueGenericSendFromISR` — **correction** compared to the old documentation
"SPI_DMA_TransferComplete", which is not supported by the decompilation),
`0x0800bf10 Buzzer_Timer_Init` / `0x0800bf7c Buzzer_Timer_Stop` / `0x0800bf88 Buzzer_Channel_Disable`
(**correction** compared to the old documentation "USART3_PWM_Init"/"USART3_Deinit" — the caller chain leads to
`buzzer_tick`/`buzzer_stop`, no USART3 references in the code; timer config values prescaler 0x10/
period 0xe77 on TIM3 base `0x40000400`), `0x0800deec printf_number_format` (**correction**
compared to the old documentation "printf_float_format" — the decompilation shows generic sign/prefix logic
for printf number formatting, no float-specific code visible), `0x0800f674 shell_clear_input_line`,
`0x0800f6a6 shell_delete_char`, `0x0800f788 shell_password_input`, `0x0800f814 shell_detect_number_base`,
`0x0800f862 shell_parse_escape_char`, `0x0800f89e shell_parse_number`, `0x0800f960 shell_parse_argument`,
`0x0800f99c shell_parse_string`, `0x0800fa04 shell_invoke_command`, `0x0800fad8 shell_hex_char_to_val`,
`0x0800fb50 shell_get_active_instance`, `0x0800fb78 shell_get_variable_value`,
`0x0800fcb8 shell_history_navigate`, `0x0800fd52 shell_history_push`.

**Thunk fix:** `0x0800ff2a` was named `thunk_FUN_0800ff30` (it still contained the old `FUN_` name
of the target). After renaming `0x0800ff30` to `shell_list_commands`, the thunk was renamed consistently to
`thunk_shell_list_commands`.

**Deliberately deferred (remain `FUN_<address>`, not renamed):**

| Address(es) | Reason |
|---|---|
| `0x080080fc`, `0x08008324`, `0x08008326`, `0x0800832e`, `0x080083f4`, `0x080088d8`, `0x0800e9d4` | 2–4-byte stubs (padding/shared return veneers) without a reconstructable name |
| `0x0800803c`, `0x08008100`, `0x0800b958`, `0x0800ba2c`, `0x0800ba46`, `0x0800ba60`, `0x0800bb10`, `0x0800bbb4`, `0x0800bc58`, `0x0800bcfc`, `0x0800bd74`, `0x0800bdf0`, `0x0800be70`, `0x0800bea6`, `0x0800bed8` | TIM/HRTIM channel config helper family with a common peripheral address comparison pattern (TIM1/8/15/16/17/20 bases `0x40012c00`/`0x40013400`/`0x40014000`/`0x40014400`/`0x40014800`/`0x40015000`). `0x0800803c` refutes the old documentation suggestion "HRTIM_DeadTimeConfig" (the function configures GPIO-like by peripheral type, no dead-time register). Channel numbering (OC1–4/IC1–2) is ambiguous from bit-shift analysis — register offsets `+0x18`/`+0x1c`/`+0x20` match CCMR1/CCMR2/CCER, but the exact channel assignment of the old documentation table (13.3) could not be confirmed beyond doubt |
| `0x08008d4c`, `0x08008d72`, `0x08008e06`, `0x08008ee4`, `0x08008f1c`, `0x08008f3c`, `0x080090f8` | Not individually verified; directly adjacent to the refuted `0x0800803c`, hence increased risk of error if the old HRTIM names were adopted blindly |
| `0x08009a40`, `0x08009a64`, `0x08009a74`, `0x08009a92` | Internal helper functions of `HAL_GPIO_Init_Extended` (GPIO AFR/EXTI registers at offset `+0x14`/`+0x60`), exact register semantics cannot be named reliably without a reference comparison |

### 13.12 Tranche 2a (10.07.2026) — address range 0x0800020c–0x08007fee

> Processed range: early boot code, GPIO/ADC/CAN/DMA/EEPROM/FLASH driver clusters as well as the
> TIM/HRTIM/RCC/ADC/I2C range marked in section 13.9 as "systematic address drift"
> (`0x08004caa`–`0x08007fee`). Methodology as in 13.9–13.11: every function verified individually via
> `get-decompilation` (including caller context, `get-callers-decompiled`), old
> documentation suggestions from 13.2/13.3 were **not** adopted blindly.
>
> **Most important finding of this tranche:** The old documentation suggestions for the TIM/HRTIM/RCC/I2C cluster
> were based on STM32F4 peripheral addresses. The actual register constants in the binary
> however show an **STM32F3 memory layout**: `RCC_BASE = 0x40021000` (not `0x40023800` as on F4),
> `Flash interface = 0x40022000` (not `0x40023C00`), `GPIO A/B/C = 0x48000000/0x48000400/0x48000800`
> (AHB2, not `0x40020xxx` as on F4), `HRTIM1_BASE = 0x40016800` (does not exist on F4 at all,
> STM32F3-specific peripheral). These four addresses were each confirmed at several independent
> places in the code (including `RCC_APB2ENR` bit 26 = HRTIM1EN, flash `KEYR` unlock sequence with
> standard constant `0xCDEF89AB`, direct GPIOA/B/C base values in several `HAL_GPIO_Init` calls).
> This retroactively explains why several GPIO/UART/NVIC suggestions were refuted in 13.9 — they
> simply matched the wrong MCU family. Consistent with the independent findings from tranche 2b
> (`0x0800803c` refutes "HRTIM_DeadTimeConfig") and tranche 2c (`0x08011854` refutes "TIM3_PWM_Init",
> `0x0801178c` DAC/HRTIM callees unclear). **Recommendation for follow-up sessions:** check name suggestions for the
> remaining TIM/HRTIM range only against this F3 layout, not against F4 assumptions.

**Ghidra actual state:** Before tranche 2a, this sub-range (0x0800020c–0x08007fee) still contained
~108 `FUN_*` functions (18 of them already individually checked and deliberately rejected in 13.9, the rest
unverified documentation suggestions or not documented at all). After tranche 2a: **23 newly named**,
0 name collisions (script duplicate check against all 452 function names in the program confirmed:
0 duplicates at the time of renaming).

**23 newly assigned names (address → name, confidence):**

*High (register constants or caller chain unambiguously evidenced):*
`0x080055c4 HAL_GPIO_Init` (GPIOA base `0x48000000`/GPIOB base `0x48000400` directly in the code, classic
MODER/AFR bit packing like `GPIO_InitTypeDef`, 26 callers), `0x0800547c FLASH_Lock` (sets bit 31 in
`0x40022014`=FLASH_CR directly after erase/write, classic lock pattern), `0x08005518 FLASH_Unlock`
(writes standard key2 `0xCDEF89AB` to `0x40022008`=FLASH_KEYR, checks LOCK bit before/after),
`0x080068cc PWR_EnableBackupAccess` (`0x40007000 |= 0x100` = PWR_CR bit 8 "DBP", F3 PWR base),
`0x08005cc8 RCC_EnableHRTIMClock` (checks base `== 0x40016800`=HRTIM1, sets RCC_APB2ENR bit 26),
`0x08005cf0 HRTIM_GPIO_AF_Init` (checks base `== 0x40016800`, configures GPIOB/C via `HAL_GPIO_Init`
for HRTIM alternate function pins), `0x0800622c I2C_ConfigureRegisters` (direct first call from
confirmed `I2C1_Init`, configures CR1/OAR), `0x0800617c I2C_CR1_Enable` (caller `I2C1_Init`,
sets PE bit), `0x080062ec I2C_MasterTransmit_ISR` / `0x08006438 I2C_MasterReceive_ISR` (callers:
`EEPROM_WriteRegister`/`EEPROM_ReadRegister`/`EEPROM_WriteVerify`, I2C-typical ISR timeout pattern
`0x8000`), `0x08007324 I2C_Master_Transmit` (directly calls the already confirmed `I2C_Wait_SB_Flag`/
`I2C_Wait_ADDR_Flag`), `0x08007fee TIM_Handle_Init` (calls confirmed `TIM_Base_Init`,
callers are the already named `TIM1_PWM_Init`/`USART3_PWM_Init`), `0x08007498
TIM_Channel_SetState` (channel selector 0/4/8/C = CH1–4, exactly like the HAL TIM channel enum, caller
`TIM1_PWM_Init`), `0x08007eb8 TIM_OC_ConfigChannel` (dispatcher, directly calls the already confirmed
`TIM_OC1_Config`/`TIM_OC2_Config`/`TIM_OC4_Config` depending on the channel parameter).

*Medium (context consistent, detailed semantics not 100 % verified):*
`0x0800150a GPIO_ConfigPin_AF` / `0x08001532 GPIO_ConfigPin_Input` (2026-10-02: by mode value these are `GPIO_ConfigPin_ExtiRising` (0x110000) / `GPIO_ConfigPin_OutputPP` (mode 1); names corrected in the base-0x08004000 programs) (**correction** compared to the NVIC hypothesis
rejected in 13.9 — both demonstrably call `HAL_GPIO_Init` with fixed mode/pull values,
so they are GPIO pin configuration variants, not NVIC functions; not to be confused with the
real `NVIC_SetPriority` from tranche 2b at `0x0800de72`), `0x08005498 FLASH_ProgramSelect`
(dispatcher between `FLASH_ProgramDoubleWord`/`FLASH_Program256Bit`, mutex flag `DAT_20000270`),
`0x08005c10 HRTIM_TimerUnit_Init` (calls `RCC_EnableHRTIMClock`, configures
timer control bit fields), `0x08006584 I2C_Timing_Config` (caller `I2C_ConfigureRegisters`, checks
base `== 0x40005400`=I2C1, large register array — presumably baud rate/timing calculation),
`0x080061d6 I2C_CR2_ConfigBits` (caller `I2C1_Init`, bit field shift `<<8`, exact register role
unclear), `0x080066a8 SysTick_Accumulate_Counter` (caller `SysTick_Handler_App`, adds two
SRAM counters), `0x080067dc Fault_Handler_Halt` (DSB×2 + infinite loop, similar pattern to
`assert_failed_halt` from tranche 2b, but separate caller/separate address), `0x08004a08
GPIO_DMA_ChannelSetup` (5× caller `Peripheral_GPIO_Init`, calls confirmed
`DMA_CalcBaseAndOffset`, DMA1/DMA2 base values `0x40020000`/`0x40020400` in the code).

**Correction compared to documentation suggestion (name was wrong, now assigned to the correct role):**

| Address | Old documentation name (wrong/used elsewhere) | New name | Rationale |
|---|---|---|---|
| `0x0800150a`/`0x08001532` | `NVIC_SetPriority`/`NVIC_EnableIRQ` (already rejected in 13.9) | `GPIO_ConfigPin_AF`/`GPIO_ConfigPin_Input` (2026-10-02: `GPIO_ConfigPin_ExtiRising`/`GPIO_ConfigPin_OutputPP`) | Demonstrably call `HAL_GPIO_Init` (newly confirmed) — GPIO, not NVIC |
| `0x08006e34` | `HRTIM_Init` (documentation suggestion, never verified) | — (remains `FUN_08006e34`) | The decompilation shows accesses to `0x40021000`/`0x40021008`/`0x4002100c` = RCC CR/CFGR/CIR (F3 layout), no HRTIM references. Presumably clock/oscillator configuration (`HAL_RCC_OscConfig`-like), but role not clear enough for renaming |
| `0x08005c10` (formerly documentation "I2C_Init") | `I2C_Init` | `HRTIM_TimerUnit_Init` | Checks base `== 0x40016800` (HRTIM1), no I2C references — already correctly rejected in 13.9 as "generic struct init without I2C reference", now the real role is identified |

**Deliberately deferred (remain `FUN_<address>`, not renamed):**

| Address(es) | Reason |
|---|---|
| `0x0800288c`, `0x080028e0`, `0x08004528`, `0x080046fc`, `0x08004728`, `0x080047bc`, `0x080047ee`, `0x08004b58` | Belong to the same suspicious function family as `FUN_08002824`, already rejected in 13.9 (there "TIM6_PWM_Init" was discarded) — all reference peripheral base addresses in the range `0x50000800`/`0x50000c00`/`0x50001000`, which **do not exist** in the STM32F3 memory layout (reserved/not mapped). Presumably dead/unused code (possibly a remnant of an F4 HAL port). No plausible renaming possible |
| `0x080043a8` | Checks base values `0x40010200`/`0x4001020c`/`0x40010208` — matches the STM32F3 comparator address range (COMP registers lie in the SYSCFG block), then configures GPIOA/B via `HAL_GPIO_Init`. Only medium confidence for "analog comparator pin config", deliberately not renamed so as not to introduce a new unconfirmed peripheral category |
| `0x0800485c` | Caller is `FUN_080083f8` (lying outside this range); register accesses only relative to handle pointer, no recognizable base address — role unclear |
| `0x08004d3e` | Callee of two further unnamed functions (`FUN_080052f2`, `FUN_080051bc`); no reliable assignment without clarifying those |
| `0x08005db0`, `0x08005df0` | Share the handle layout (offsets `+0xdc`/`+0xdd`) with the newly named HRTIM functions `HRTIM_TimerUnit_Init`/`HRTIM_GPIO_AF_Init`, so they are presumably also HRTIM wrappers (wait flag/locked config), but not certain enough for a specific name without a concrete register constant |
| `0x080073b8` | Called by `I2C_Master_Transmit` on the first call in a configuration-like manner (analogous to the role of `I2C_Timing_Config` in the other I2C chain) — plausibly I2C timing-related, but not cross-checked |
| ~50 further functions in the range `0x08004300`–`0x08007fee` (including the cluster originally documented as "TIM_PWM_Channel_Config", "SPI_*", "ADC_ConfigChannel/Start_DMA/Calibrate") | Not verified individually any more in this tranche for time reasons. After the F3 address finding above, increased caution is required for all these names — **do not** adopt unchecked from 13.2/13.3. Candidate for a dedicated follow-up tranche with the now known F3 peripheral layout as reference |
| ~21 tiny stubs (2–6 bytes, e.g. `0x0800155c`, `0x08003afe`, `0x08003b00`, `0x08004d6a`, `0x08004d6c`, `0x08004e74`, `0x080051bc`, `0x080052c4`, `0x08005376`–`0x08005380`, `0x08006db4`, `0x0800748c`–`0x08007496`, `0x08007d4a`) | Pure return/padding stubs without reconstructable semantics — deliberately not guessed |
| 18 functions already individually checked and rejected in section 13.9 (`0x0800020c`, `0x080007d8`, `0x08000b50`, `0x08000fb4`, `0x08001868`, `0x08001ea0`, `0x080035d4`, `0x08003af0`, `0x08003b04`, `0x08003d54`, `0x08003ff8`, `0x08004128`, `0x08005384`, `0x08005774`, `0x08005782`, `0x08005792`, `0x080057a0`, `0x080066d8`) | Unchanged — the rejection from 13.9 remains valid, no new evidence found in this tranche that would justify a renaming |

**Note on already named (non-FUN_) functions — correction candidate for a follow-up session:**
`DAC_Init` (`0x08006800`, already named before this tranche, hence outside the renaming scope
of this tranche) references exclusively `0x40007000`/`0x40007014`/`0x40007080` = PWR register range
(F3), **no** DAC address — the name is very probably wrong (presumably a real
PWR init function). Not corrected in this tranche, as it is outside the FUN_ scope and to avoid conflicts
with the parallel tranche 2c (which already references this function under the name `DAC_Init`) —
mark for review in a dedicated consistency session.

---

### 13.13 Tranche 3a (10.07.2026) — Address Range 0x0800020c–0x08004fe0

> Processed range: subset of the range `0x0800020c–0x08007fee` already covered in 13.12 (tranche 2a).
> Tranche 2a had named 23 functions there and, for the rest (among others the
> "~50 further functions" cluster `0x08004300–0x08007fee`), explicitly recommended a dedicated follow-up tranche
> using the now confirmed STM32F3 peripheral layout. This tranche (3a) takes over
> the sub-range `0x0800020c–0x08004fe0` of that; the rest up to `0x08007fee` is in tranche 3b.
> Methodology unchanged: every function individually via `get-decompilation` (incl. `includeCallers`/
> `includeCallees`), register/bit-field patterns reconciled against known HAL reference implementations
> (STM32Fx `stm32f3xx_hal_can.c`/`hal_dma.c`), no old documentation proposals adopted blindly.

**Most important finding of this tranche:** The complete CAN HAL layer (init, TX, RX, filter, IRQ dispatch)
and the DMA HAL layer (Start_IT, Abort, IRQ dispatch) could be unambiguously identified from their characteristic
state machine pattern (state byte convention `0`=RESET, `1`=READY, `2`=BUSY, as in the real
STM32 HAL) and their caller/callee chains to the functions already confirmed in tranche 2a/13.9
(`CAN_SetFilter_ExtID/StdID`, `CAN_TX_SendFrame`, `CAN_InitMailboxes`, `CAN_CopyTxFrame`,
`HAL_GetTick`, `DMA_SetConfig`). In the process it turned out that
`FUN_08001ea0` — noted in 13.9 only as *"probably the real CAN_Init, not UART"*, but not renamed for lack of
evidence — is in fact the CAN1 init wrapper: it sets `Instance = 0x40006400`
(confirmed CAN1 base from section 4.1) and directly calls the newly identified `HAL_CAN_Init`.

**9 newly assigned names (address → name, confidence):**

*High (caller/callee chain + register/state pattern unambiguous, direct relation to already
verified functions):*
`0x08001ea0 CAN1_Init` (sets Instance=`0x40006400`, configures timing parameters, calls `HAL_CAN_Init`
+ the new `HAL_CAN_ConfigFilter`/`HAL_CAN_ActivateNotification` chain; caller: confirmed
`CAN_Filter_Setup`), `0x08004fe0 HAL_CAN_Init` (classic bxCAN init state machine: INRQ/INAK
handshake with `HAL_GetTick` timeout, MCR/BTR bit packing exactly per the bxCAN scheme, calls confirmed
`CAN_InitMailboxes`, sets State=READY), `0x08004c5c HAL_CAN_AddTxMessage` (state check READY, reads
TSR code field for mailbox selection, calls confirmed `CAN_CopyTxFrame`; caller: confirmed
`CAN_TX_SendFrame`), `0x08004caa HAL_CAN_ConfigFilter` (writes FR1/FR2 filter bank registers depending on
std/ext mode; callers: confirmed `CAN_SetFilter_ExtID`/`CAN_SetFilter_StdID` — both previously
called an unnamed common function), `0x08004d70 HAL_CAN_GetRxMessage` (decodes FIFO mailbox:
IDE/RTR/ID extraction with exact bxCAN bit masks `0x40000000`/`0x20000000`/`>>0x12`, DLC/timestamp/
FMI fields, data copy via DLC lookup table `DAT_08017f60`), `0x08004e76 HAL_CAN_IRQHandler`
(reads SR&IER masks for TME/FMP0/FMP1/FF/FOV/EWG/EPV/BOF/LEC/ERR and dispatches to 9 sub-handlers,
finally calls the error callback if an error code is set — exact bxCAN IRQHandler pattern),
`0x08004ac8 HAL_DMA_Start_IT` (state transition READY(1)→BUSY(2), calls confirmed `DMA_SetConfig`,
TC/HT/TE IT enable bit pattern (`0xE` with / `0xA` without HT); caller: confirmed
`HAL_UART_Transmit_DMA`), `0x080048c4 HAL_DMA_Abort` (state check BUSY(2)→READY(1), error code
`0x04`=`HAL_DMA_ERROR_NO_XFER` in the error path — exact match, calls abort callback; callers:
confirmed `UART_AbortReceive`/`UART_IRQHandler_Full`), `0x0800493a HAL_DMA_IRQHandler` (decodes
ISR flags GIF/TCIF/HTIF/TEIF per channel via bit shift `<<bVar4`, dispatches to XferCplt/
XferHalfCplt/XferError callback pointers — `callerCount=0`, since presumably referenced only via the
interrupt vector table, not by a direct call).

**Duplicate check:** All 9 names were checked before renaming against the full symbol list of the entire
program (`get-symbols`, `filterDefaultNames=true`, all 309 functions named at that time) —
**0 collisions**. Actual Ghidra state after this tranche: **318/445 named (71.5%)**.

**Deliberately deferred (remain `FUN_<address>`, not renamed):**

| Address | Reason |
|---|---|
| `0x080027bc` | Called by confirmed `GPIO_DMA_ChannelSetup`, writes a register pointer `uVar1*4+0x40020900` as well as a fixed address `0x40020940` into the target structure — bit pattern fits an interrupt flag/clear mechanism, but `0x40020900` cannot be unambiguously assigned to any known STM32F3 peripheral register (DMA1/2 base is at `0x40020000`/`0x40020400`, channel registers end before `0x400200A0`). No reliable role assignment |
| `0x08004b7c` | Caller is the newly confirmed `CAN1_Init` as well as `FUN_08005248` (outside this range). Manipulates an IER-like register in CAN-typical bit groups (`0x7`,`0x38`,`0x1c0`,…, matching bxCAN `IER` flag groups), but the logic (conditionally setting a status flag depending on whether the bit was already set) does not match the simple 1:1 OR pattern of `HAL_CAN_ActivateNotification`. Third parameter without HAL equivalent — presumably a customer-specific CAN IT status function, but role not certain enough for a name |
| `0x08004d08` | Caller is `CAN1_Init` (call with constants `2,2,1,1`), state check `==READY`. Writes four packed flag bits into a register at handle offset `+0x80` — offset fits no standard bxCAN register (`CAN_TypeDef` reserved area). Role presumably CAN operating mode configuration, but register identity not verifiable |
| `0x08004d3e` | First call within `CAN1_Init`, resets error code/state field (State=0/RESET) and calls two further unnamed functions (`0x080052f2`, `0x080051bc` — both outside this range, in tranche 3b). Presumably a `HAL_CAN_MspInit` wrapper or handle reset, but not certain enough without clarifying the two callees. **Note for tranche 3b:** If `0x080052f2`/`0x080051bc` are identified as GPIO/NVIC init for CAN1, `0x08004d3e` should be confirmed as `HAL_CAN_MspInit` in a follow-up session |
| `0x0800493a` sub-handlers `0x08004d6a`, `0x08004d6c`, `0x08004e74` (2 bytes each) | Tiny stubs, referenced by `HAL_CAN_IRQHandler` as callback jump targets — pure return stubs without reconstructable semantics, deliberately not guessed (consistent with the stub policy from 13.12) |
| All 18 functions in this range individually checked and rejected in 13.9 (`0x0800020c`, `0x080007d8`, `0x08000b50`, `0x08000fb4`, `0x08001868`, `0x080035d4`, `0x08003af0`, `0x08003b04`, `0x08003d54`, `0x08003ff8`, `0x08004128` among others) | Unchanged — no new evidence in this tranche that would invalidate the documented rejection |
| Dead-code family `0x0800288c`, `0x080028e0`, `0x08004528`, `0x080046fc`, `0x08004728`, `0x080047bc`, `0x080047ee`, `0x08004b58` (from 13.12) | Checked again, no new findings — still reference non-existent `0x50000800`/`0x50000c00`/`0x50001000` addresses (F3 layout has nothing mapped there). Remains unnamed |
| `0x080043a8` (from 13.12, comparator candidate) | No new evidence found, deliberately not adopted |
| Stubs `0x0800155c`, `0x08003afe`, `0x08003b00` (from 13.12) | Still pure padding/return stubs |

**Result of this tranche:** 9 newly named, 0 collisions, 5 deliberately deferred with documented
reasoning (plus confirmation of the decisions already made in 13.9/13.12 for the remaining
`FUN_*` in this address range). The range `0x0800020c–0x08004fe0` is thereby considered
completely processed for this session.

---

### 13.14 Tranche 3b (10.07.2026) — Address Range 0x080051bc–0x08007d4c

> Processed range: direct continuation of tranche 3a (13.13), which had covered the range up to `0x08004fe0`.
> This tranche takes over the rest up to `0x08007d4c` (end according to the assigned range).
> Methodology unchanged: every function individually via `get-decompilation` (incl. `includeCallers`/
> `includeCallees`), register/bit-field patterns reconciled against STM32F3 HAL reference implementations
> (`stm32f3xx_hal_can.c`, `hal_rcc.c`, `hal_rcc_ex.c`, `hal_tim.c`, CMSIS `core_cm4.h`).
> In total **49 `FUN_*` functions** identified in the range (via `get-functions filterDefaultNames=false`).

**Correction to the note from 13.13:** There it was hypothesized that `0x08004d3e` might be `HAL_CAN_MspInit`
if its callees `0x080052f2`/`0x080051bc` were confirmed as GPIO/NVIC init for CAN1. That does not hold
true: the actual `HAL_CAN_MspInit` is **`0x080051c0`** (caller: `HAL_CAN_Init`, checks
`Instance==0x40006400`, configures GPIOD pins AF9 via `HAL_GPIO_Init` and enables `CAN_RX1_IRQn`
(`0x15`/21) via `HAL_NVIC_SetPriority`+`NVIC_EnableIRQ`). `0x080052f2` and `0x080051bc`, on the other hand, belong
to a **CAN stop/DeInit sequence** (called from `0x08004d3e`): `0x080052f2` waits with a timeout for
a ready flag and is thus more `HAL_CAN_Stop`-like, `0x080051bc` is a pure no-op stub. `0x08004d3e`
itself remains unnamed outside this range (tranche 3a territory) — but should be checked in a
follow-up session as `HAL_CAN_Stop` or similar, not as `HAL_CAN_MspInit`.

**Most important finding:** The RCC clock init layer (`RCC_OscConfig`/`RCC_ClockConfig`/`RCC_PeriphClockSourceConfig`)
as well as a second TIM cluster (TIM7, TIM20, TIM4 — each with its own channels-enable/clock-enable function pairs,
analogous to the already confirmed HRTIM pattern) could be unambiguously identified. Both `RCC_OscConfig`
(`0x08006e34`) and `RCC_ClockConfig` (`0x08006bec`) are demonstrably called from `SystemClock_Config`
(outside this range) in exactly this order — exactly the pattern of
`HAL_RCC_OscConfig()` followed by `HAL_RCC_ClockConfig()` in real STM32 HAL code.

**30 newly assigned names (address → name, confidence):**

*High (register address/bit pattern unambiguously mapped to a known peripheral base or CMSIS system register,
or caller/callee chain to already confirmed functions unambiguous):*

| Address | Name | Rationale |
|---|---|---|
| `0x080051c0` | `HAL_CAN_MspInit` | Caller `HAL_CAN_Init`; checks `Instance==0x40006400` (CAN1); GPIOD AF9 pins + `CAN_RX1_IRQn`(21) |
| `0x08005248` | `HAL_CAN_RxFifo0MsgPendingCallback` | Caller `HAL_CAN_IRQHandler`; dispatches received frame to `BMS_CAN_Parser`/`FUN_08001868`/FreeRTOS queue |
| `0x080052c6` | `CAN_State_EnterListening` | Caller `CAN1_Init`; state byte 1(READY)→2(LISTENING), otherwise error flag — exact `HAL_CAN_StateTypeDef` pattern |
| `0x080052f2` | `CAN_ExitInitMode_WaitAck` | Follow-up function to `CAN_State_EnterListening`/part of the CAN stop sequence; timeout wait loop on ready flag, state finalization |
| `0x08005384` | `FLASH_EraseSectors_Internal` | Only caller `FLASH_EraseSectors`; real erase implementation (busy flag, save/restore `FLASH_ACR` wait states, `FLASH_MassErase`/`FLASH_SectorErase`) |
| `0x08005782` | `GPIO_TogglePins` | Caller `CAN_Filter_Setup` with `param_1=0x48000000`(GPIOA); classic BSRR toggle formula `(mask&~ODR)\|((ODR&mask)<<16)` |
| `0x08005792` | `I2C_EEPROM_SetXferState` | Called exclusively from `EEPROM_ReadRegister`/`WriteRegister`/`WriteVerify*` before/after `I2C_MasterReceive_ISR` |
| `0x080057a0` | `DBGMCU_GetRevisionID` | Reads `_DAT_e0042000>>0x10` — `0xE0042000` = Cortex-M `DBGMCU_BASE`/`IDCODE`, upper half-word = REV_ID |
| `0x08006778` | `HAL_NVIC_SetPriority` | Reads `SCB->AIRCR`(`0xE000ED0C`) PRIGROUP field, splits preempt/sub priority, calls CMSIS `NVIC_SetPriority` — standard HAL wrapper |
| `0x080066d8` | `SysTick_Reconfigure` | Calls `SysTick_Config` then `HAL_NVIC_SetPriority(SysTick_IRQn=-1, …)` — dynamic SysTick rate reconfiguration |
| `0x080068dc` | `RCC_PeriphClockSourceConfig` | Callers: CAN/I2C timing/RTC clock init; sets 2-bit fields in sequence depending on flag mask — pattern of `HAL_RCCEx_PeriphCLKConfig` incl. RTC backup domain reset sequence |
| `0x08006bec` | `RCC_ClockConfig` | Caller `SystemClock_Config` (2nd call after `RCC_OscConfig`); configures `FLASH_ACR` wait states + AHB/APB prescalers, calls `HAL_RCC_GetSysClockFreq`+`SysTick_Reconfigure` |
| `0x08006e34` | `RCC_OscConfig` | Caller `SystemClock_Config` (1st call); HSE/PLL enable + ready-wait sequences on `RCC_CR`(`0x40021000`) |
| `0x080072d0` | `RTC_ConvertBCD_TimeDate` | Only caller `RTC_Read_Time`; unpacks BCD time/date registers, calls `BCD_To_Decimal` |
| `0x080073b8` | `RCC_EnableRTCClock` | Checks `Instance==0x40002800`(RTC_BASE F3), enables RTC clock via `RCC_PeriphClockSourceConfig` — naming scheme analogous to `RCC_EnableHRTIMClock` |
| `0x0800740c` | `I2C_ClearAddrFlag_Wait` | Caller `I2C_Wait_ADDR_Flag`; clears ADDR bit, timeout wait loop with `HAL_GetTick` |
| `0x08007464` | `SysTick_Config` | Sets `SysTick->LOAD`(`0xE000E014`)/`VAL`(`0xE000E018`)/`CTRL`(`0xE000E010`=7) + priority — exact CMSIS `SysTick_Config` implementation |
| `0x08007574` | `TIM_OC_SetMode` | Caller `TIM20_Init`; sets OC mode nibble in CCMR-like register, special case TIM16(`0x40014400`)/TIM17(`0x40014800`) |
| `0x08007606` | `TIM7_Channels_EnableAll` | Caller `TIM7_Init`; sets all channel enable flags, calls `RCC_EnableTIM7Clock` |
| `0x08007654` | `RCC_EnableTIM7Clock` | Checks `Instance==0x40001400`(TIM7_BASE F3), sets APB1ENR bit 5(TIM7EN) + `NVIC_EnableIRQ(0x37)` |
| `0x08007724` | `TIM20_CaptureUpdate` | Checks `Instance==0x40015000`(TIM20_BASE); capture period measurement with 4-element ring buffer, caller `TIM20_CC_EventDispatch` |
| `0x08007774` | `TIM_CCR_ConfigChannel` | Caller `TIM20_Init`; configures capture/compare register per channel (0/4/8/0xC) via sub-handlers `0x0800bdf0`/`be70`/`bea6`/`bed8` |
| `0x08007834` | `TIM20_Channels_EnableAll` | Caller `TIM20_Init`; identical pattern to `TIM7_Channels_EnableAll`, calls `RCC_EnableTIM20Clock` |
| `0x08007884` | `RCC_EnableTIM20Clock` | Checks `Instance==0x40015000`(TIM20_BASE), sets enable bit 20 + `NVIC_EnableIRQ(0x50)` |
| `0x080078c0` | `TIM_CCxChannelEnable` | Caller `TIM20_Init`; generic CCER bit activation + counter start, TIM base address list (TIM1/2/3/4/8/15/20) |
| `0x08007a48` | `TIM20_CC_EventDispatch` | Checks 4 CC channel event bit pairs + update event, dispatches to `TIM20_CaptureUpdate`/`TIM4_CaptureCallback_Dispatch`-like handlers and stub callbacks |
| `0x08007c4c` | `TIM4_Channel_ActionDispatch` | Caller `0x08011854`(TIM4 init context, `NVIC_EnableIRQ(0x1e)`=IRQ30=`TIM4_IRQn` F3); busy guard + 6-way dispatch |
| `0x08007cbc` | `TIM4_CaptureCallback_Dispatch` | Checks `Instance==0x40000800`(TIM4_BASE F3); calls callback function pointers `_DAT_20000060`/`_DAT_20000064` depending on channel state |
| `0x08007cfc` | `TIM4_Channels_EnableAll` | Identical pattern to `TIM7_/TIM20_Channels_EnableAll`, in the TIM4 init context (`0x08011854`) |
| `0x08007d4c` | `TIM_CCxChannelEnable_MOE` | Like `TIM_CCxChannelEnable`, additionally `BDTR.MOE` bit(`0x8000`@offset`0x44`) for advanced timers (TIM1/8/15/16/17/20) |

**Duplicate check:** All 30 names were checked before renaming against the full symbol list of the entire
program (`get-functions filterDefaultNames=true`, 318 functions named at the start of this tranche
incl. the 9 from tranche 3a) — **0 collisions**. Rename mechanism: `create-label` with
`setAsPrimary=true` at the function entry point (verified via `get-decompilation` that the function name
is actually adopted, not just a secondary label).

**Deliberately deferred (remain `FUN_<address>`, 19 functions):**

| Address(es) | Reason |
|---|---|
| `0x080051bc` | 2-byte no-op stub (`return;`), part of the CAN stop sequence of `0x08004d3e` — role understood, but deliberately not renamed per stub policy (2–10 bytes) |
| `0x080052c4` | 2-byte stub, padding between `CAN_State_EnterListening` and `CAN_ExitInitMode_WaitAck` |
| `0x08005376`, `0x08005378`, `0x0800537a`, `0x0800537c`, `0x0800537e`, `0x08005380` | Six 2-byte stubs in a row, no cross-references with reconstructable semantics |
| `0x08005774` | Generic bit mask test helper (`(*(x+0x10)&mask)!=0`), but called from completely different contexts (`EMS_Inverter_CAN_Dispatcher`, `Mode3_RoundRobin_Timer`) with varying masks — too generic for a fitting name without guessing |
| `0x08005db0` | Generic wait-flag timeout helper on a handle with offset `+0x388` (too large for known peripheral handles), state bytes at offset `+0xdd`/`+0x37` without unambiguous peripheral assignment |
| `0x08005df0` | Busy-guard wrapper around unnamed `0x08008f1c` (outside range) — role unclear without clarifying the callee |
| `0x08006db4` | Trivial getter (`return _DAT_20000290;`); global is used in frequency calculations, but cannot be unambiguously assigned to a domain (grid frequency vs. general tick counter) |
| `0x0800748c`, `0x0800748e`, `0x08007490`, `0x08007492`, `0x08007494`, `0x08007496` | Six 2-byte stubs, referenced by `TIM20_CC_EventDispatch` as no-op callbacks for unused event channels — role understood, but not renamed per stub policy |
| `0x08007d4a` | 2-byte stub, callee of `TIM4_Channels_EnableAll` (counterpart to `RCC_EnableTIM7Clock`/`RCC_EnableTIM20Clock`, but apparently a no-op for TIM4) |

**Result of this tranche:** 30 newly named, 0 collisions, 19 deliberately deferred (13 mini stubs of
2 bytes + 6 functions with overly general/unclear semantics). The range `0x080051bc–0x08007d4c` is thereby considered
completely processed for this session.

---

### 13.15 Tranche 3c (10.07.2026) — Address Range 0x0800803c–0x080151c4

> Processed range: last remaining FUN_ range of the Micro FW, incl. the TIM/HRTIM channel config cluster deliberately
> deferred in tranche 2b (13.11) (`0x0800803c`–`0x0800bed8`), the cluster around `0x08008d4c`–`0x080090f8` (HRTIM timer unit registers, `param*0x80`-indexed), the
> GPIO/UART bit-field accessors around `0x08009a40`–`0x08009a92`, the small clusters at `0x0800e9d4`,
> `0x0801178c`/`0x08011854`/`0x08011b10` as well as `0x080150d4`–`0x080151c4`. Methodology as in 13.9–13.14:
> every function individually verified via `get-decompilation` (incl. `includeCallers`/`includeCallees`,
> disassembly in suspicious cases), old documentation proposals not adopted blindly.
> Actual Ghidra state before this tranche (incl. parallel tranche 3a/3b contributions): 346/445 named.
> After tranche 3c: **363/445 named (81.6 %)**, checked via `get-functions`/`get-functions-by-similarity`
> against all names in the entire program — **0 collisions**.

**Most important finding 1 of this tranche — TIM_OC_ConfigChannel/TIM_CCR_ConfigChannel dispatchers fully resolve the
channel cluster deferred in 13.11 as "ambiguous":** The already confirmed `TIM_OC_ConfigChannel` (`0x08007eb8`) dispatches based on `param_3` (0/4/8/0xc/0x10/0x14) directly to
six until then unnamed functions, which write CCMR1/CCMR2/CCMR3 (offsets `0x18`/`0x1c`/`0x50`) for the
channels OC1–OC6 (STM32F3 TIM1/TIM20 with 6 channels, HRTIM-like layout). The parallel
dispatcher `TIM_CCR_ConfigChannel` (`0x08007774`, outside this range, named by tranche 3b)
dispatches analogously four until then unnamed input capture channel functions (filter+polarity in
CCMR1/CCMR2 + CCER). The channel numbering ambiguity documented in 13.11 is thereby **resolved** for this
specific function cluster.

**Most important finding 2 — `SystemClock_Config` identified, supports the hypothesis on `DAC_Init`
already voiced in 13.12:** `0x0801178c` (0 callers — presumably only
reached from the reset handler vicinity, outside the pure call graph) calls in exact CubeMX order `DAC_Init(0)` →
`PWR_EnableBackupAccess()` → oscillator configuration twice (`FUN_08006e34`, already noted in 13.12 as a
possible `HAL_RCC_OscConfig` equivalent) → clock tree configuration (`FUN_08006bec`, call
with `ClockType=0xf`, `SYSCLKSource=3`(PLL), `FLASH_LATENCY=4`) → conditional setting of
FLASH_ACR bit 8 (prefetch/cache). This pattern is the classic CubeMX `SystemClock_Config`
signature. The call of `DAC_Init(0)` as the **first** step (before any clock configuration) additionally supports the
hypothesis documented in 13.12 that `DAC_Init` (`0x08006800`, outside the
renaming scope) is in reality a PWR init function — a DAC init at this point before the
clock configuration would be atypical, whereas a PWR regulator init is exactly the expected CubeMX pattern.
`FUN_08006e34`/`FUN_08006bec` are in the address range of tranche 3b, however, and were **not**
renamed here (foreign scope).

**Most important finding 3 — critical confirmation of the task hypothesis on `0x080150d4`–
`0x080151c4`:** All four functions in this range (`0x080150d4`, `0x080150e8`, `0x080150fc`,
`0x080151c4`, 10 bytes each) return on `get-decompilation` (incl. disassembly) **decompilation/
assembly contents whose actual instruction addresses (`0x08007xxx`/`0x08009xxx`) and
register convention (`unaff_r4`/`unaff_r5`/`unaff_r8` — "unassigned" registers, a classic
sign of a jump-into-the-middle-of-a-function misdetection) do not match their own address**, and
two of the four even return identical pseudocode for different addresses. Together with
`totalIncomingReferences` of up to 21 (widely scattered caller addresses up to `0x0801a7xx`) this is
unambiguous evidence for the hypothesis voiced in the assignment: **Ghidra has not recognized real,
independent functions here, but has wrongly marked jump/table fragments as function starts**
(the thunk `thunk_CAN_Filter_Setup` at `0x080150f2` sits exactly between two of these
fragments — a hint of a densely packed thunk/jump table region). These four addresses
were **deliberately not renamed** — a rename would suggest a non-existent function identity.
Recommendation for a follow-up session: redraw function boundaries in this range with Ghidra's
`analyze-program`/manual function boundary correction before naming anything further here.

**17 newly assigned names (address → name, confidence):**

*High (register offsets verified directly via confirmed dispatcher functions `TIM_OC_ConfigChannel`/
`TIM_CCR_ConfigChannel`, or CCER bit pattern matching a standard HAL function exactly):*
`0x0800ba2c TIM_CCxChannelCmd` (CCER, 1-bit mask at `channel` offset — classic
`TIM_CCxChannelCmd` HAL pattern, 4 callers incl. already confirmed `TIM_Channel_SetState`),
`0x0800ba46 TIM_CCxNChannelCmd` (CCER, mask `4<<channel` = CCxNE bit, caller `TIM_Channel_SetState`),
`0x0800ba60 TIM_OC1_Config`, `0x0800bb10 TIM_OC2_Config`, `0x0800bbb4 TIM_OC3_Config`,
`0x0800bc58 TIM_OC4_Config`, `0x0800bcfc TIM_OC5_Config`, `0x0800bd74 TIM_OC6_Config` (all six
identified directly from `TIM_OC_ConfigChannel` dispatch branches `param_3=0/4/8/0xc/0x10/0x14`,
write CCMR1/CCMR2/CCMR3 exactly at the expected bit offsets, store the pulse/CCR value
at `+0x34/+0x38/+0x3c/+0x40/+0x48/+0x4c`).

*Medium-high (dispatcher `TIM_CCR_ConfigChannel` outside own scope, but target functions
themselves with a clear CCMR/CCER filter+polarity pattern):*
`0x0800bdf0 TIM_IC1_Config`, `0x0800be70 TIM_IC2_Config`, `0x0800bea6 TIM_IC3_Config`,
`0x0800bed8 TIM_IC4_Config` (input capture channel configuration: ICxF filter (4 bits) + CCxP/CCxNP
polarity in CCER, dispatch order 0/4/8/0xc identical to the OC cluster).

*High (peripheral base address + caller chain unambiguous):*
`0x0800803c TIM_GPIO_AF_Init` (checks timer base `0x40012c00`=TIM1→GPIOC AF4 or
`0x40000400`=TIM3→GPIOB AF1/AF2, calls confirmed `HAL_GPIO_Init`; only caller: confirmed
`TIM_Handle_Init` — counterpart to `HRTIM_GPIO_AF_Init` from 13.12, definitively refutes the old
documentation name "HRTIM_DeadTimeConfig" from 13.11), `0x08011854 TIM4_IC_Init` (base `0x40000800`=TIM4
[F3 layout], configures two input capture channels [`param_3=0,4`] via the confirmed
`TIM_CCR_ConfigChannel`, then enables `NVIC_EnableIRQ(0x1e)` = IRQn 30 = `TIM4_IRQn` on
STM32F3 — unambiguous evidence for the TIM4 context; definitively refutes the old documentation proposal "TIM3_PWM_Init" from
13.12), `0x0801178c SystemClock_Config` (see "Most important finding 2" above).

*Medium (structural context consistent, register detail not 100 % verified):*
`0x08008100 TIM_Channel_Start` (state check→State=2(BUSY), calls newly confirmed
`TIM_CCxChannelCmd`, then sets BDTR MOE bit (offset `+0x44`, only for advanced timer bases
TIM1/15/16/17/20) as well as CR1 CEN bit — generic "start PWM channel" equivalent to
`HAL_TIM_PWM_Start`/`HAL_TIM_OC_Start`; callers `Buzzer_Timer_Init`/`Buzzer_Channel_Disable`),
`0x080082f8 TIM_GetChannelPulse` (reads the pulse value stored by `TIM_OC1_Config`…`TIM_OC4_Config`
from `+0x34/+0x38/+0x3c/+0x40` depending on the channel parameter — getter counterpart to the OCx config
functions).

**Duplicate check:** All 17 names were checked before renaming via `get-symbols` (7 pages × 150,
`filterDefaultNames=true`, ~650 of 950 symbols reviewed incl. all TIM/RCC/peripheral namespaces)
and after renaming additionally cross-checked via `get-functions-by-similarity` against
`TIM_OC1_Config`/`SystemClock_Config` — **0 collisions**, also no collision with the names assigned
in parallel by tranche 3b (among others `TIM_CCR_ConfigChannel` at `0x08007774`, outside
own scope, but used as context).

**Deliberately deferred (remain `FUN_<address>`, not renamed):**

| Address(es) | Reason |
|---|---|
| `0x080080fc`, `0x08008324`, `0x08008326`, `0x0800832e`, `0x080083f4`, `0x080088d8`, `0x0800e9d4` | 2–4-byte stubs (padding/shared return veneers), consistent with 13.11 |
| `0x080082e4` | Conditional jump to a bootloader thunk (`LAB_1000219c`) depending on timer base `0x40001400`=TIM7; only caller is the itself unnamed `FUN_08007a48` (tranche 3b scope) — without clarifying it, no safe role assignment |
| `0x080088dc` (338 B) | No callers found; computes mode-dependent (`DAT_20000035`=1/2/3) duty/timing values for an SRAM buffer at `0x20000cec`+offsets — presumably part of the inverter PWM modulation, but without caller context and without an unambiguous peripheral base address cannot be named reliably |
| `0x08008d4c`, `0x08008d72`, `0x08008e06`, `0x08008ee4`, `0x08008f1c`, `0x08008f3c`, `0x080090f8` | New structural finding: all use a `param_2 * 0x80` register indexing pattern that matches exactly the offset of the HRTIM timer unit blocks (TIMA…TIME, 0x80 bytes each from HRTIM1 base `0x40016800`) — confirms the hypothesis voiced in 13.12 that the callers `0x08005db0`/`0x08005df0` are "HRTIM wrappers". For a precise register assignment (CMP/SET/RST/OUT per timer unit), however, a reference reconciliation with the STM32F3 reference manual is missing; `0x08008d4c`/`0x08008e06`/`0x08008f3c`/`0x080090f8` additionally have `callerCount=0` (presumably reached only indirectly/via table) — deliberately not guessed |
| `0x08009a40`, `0x08009a4c`, `0x08009a54`, `0x08009a5c`, `0x08009a64`, `0x08009a74`, `0x08009a92` | New finding compared to 13.11: Only `0x08009a40`/`0x08009a64`/`0x08009a74`/`0x08009a92` are called exclusively by `HAL_GPIO_Init_Extended` (consistent with the old documentation note "GPIO AFR/EXTI helper functions"). `0x08009a4c`/`0x08009a54`/`0x08009a5c`, on the other hand, are generic 6–8-byte bit-field extractors (`*(param_1+8) & mask`) that are called across domains both by `HAL_GPIO_Init_Extended` and by `UART_WaitReady`/`FUN_08000b50`/`FUN_08003b04`/`FUN_08003ff8` — contradicts the previous GPIO-exclusive classification. For none of the seven functions could a domain-specific role be evidenced with enough certainty for a name |
| `0x08011b10` | One-line global copy operation (`DAT_20000458 = DAT_20000024`) without recognizable semantics of the two variables, caller `UART_DMA_RxCplt_Dispatch` |
| `0x080150d4`, `0x080150e8`, `0x080150fc`, `0x080151c4` | **Confirmed Ghidra function boundary error** — see "Most important finding 3" above. No renaming, since not real independent functions |

**Result of this tranche:** 17 newly named, 0 collisions, 28 functions deliberately deferred
(of which 4 documented as a confirmed Ghidra analysis artifact, not just "unclear"). The range
`0x0800803c–0x080151c4` is thereby considered completely processed for this session — remaining
`FUN_*` addresses are either stubs, cross-domain helper functions without a certain single role,
or (in the case of `0x080150d4`–`0x080151c4`) not real function boundaries.

---

### 13.16 Tranche 4a (10.07.2026) — Address range 0x0800020c–0x08004128

> Processed range: the start range that had already been searched most often (tranches 2a/13.12,
> 3a/13.13). Via `get-functions filterDefaultNames=false`, 18 `FUN_*` functions were
> identified in the range. Methodology: each function was examined individually via `get-decompilation` (incl. `includeCallers`/
> `includeCallees`), register constants were reconciled against the STM32F3 peripheral map, and
> caller/callee chains were reconciled against the meanwhile considerably larger name inventory.

**Most important finding — a coherent ADC HAL layer was completely uncovered:** The
addresses `0x50000000`/`0x50000100`/`0x50000300`/`0x50000400`/`0x50000500`/`0x50000700` exactly
match the STM32F3 memory map for ADC1/ADC2/ADC1_2 common/ADC3/ADC4/ADC3_4 common. `0x08000fb4`
(`ADC_Instances_Init`) initializes 5 handle-like structs in sequence with these instance values
and calls `0x08003b04` (`HAL_ADC_Init`) for each, which in turn, if state==RESET, calls
`0x08003d54` (`HAL_ADC_MspInit`, RCC clock + `HAL_GPIO_Init(0x48000000,...)`/`HAL_GPIO_Init(0x48000400,...)`
per instance) — a literal match on the real `HAL_ADC_Init()`/`HAL_ADC_MspInit()` pattern.
Directly afterwards `ADC_Instances_Init` calls `0x080035d4` (`HAL_ADCEx_Calibration_Start`) five times, which sets
bit 31 (ADCAL) in a CR-like register and waits with a timeout for its reset —
exactly the real STM32 HAL calibration sequence. `0x08003ff8` (`HAL_ADC_Start_DMA`) rounds off the layer:
sets DMA callback pointers (`+0x2c/+0x30/+0x34`, the same offsets as already verified in 13.13 for
`HAL_DMA_Start_IT`), sets CR=`0x1c` (ADSTART bit group) and calls `HAL_DMA_Start_IT`;
its internal wait helper `0x08000b50` (`ADC_ConversionStop_WaitReady`) polls for up to 3 ticks for a
ready bit and sets error/state flags on timeout — matches the internal `ADC_ConversionStop()` helper
of the real HAL. The 5th instance (`0x50000600`) lies between ADC4 (`0x500`) and ADC3_4 common (`0x700`)
and is not documented in the official STM32F3 reference manual — the code treats it structurally like
a regular ADC instance, but its exact silicon identity remains unclear.

**Second finding — `0x08004128` is a comparator init (`HAL_COMP_Init`):** The function checks the
register address against `0x40010200`/`0x204`/`0x208`/`0x20c`/`0x210`/`0x214` — exactly COMP1_CSR…COMP6_CSR
on STM32F3 (comparator peripheral, does not exist on F4, confirms the F3 layout once again) — and
derives from this an EXTI line bitmask, which is written to a register at `0x40010424` (EXTI area,
base `0x40010400`). Calls `0x080043a8` (lies in tranche 4b, already
covered there, identified in tranche 5a — see 13.19 — as `HAL_COMP_MspInit`: called only on the very first
init of a comparator (`param_1[0x1d]==0` guard, classic HAL "MspInit only when
state==RESET" pattern) and configures via `HAL_GPIO_Init` the associated analog input pin for
COMP1/COMP3/COMP4 on GPIOA/B/C).

**Third finding — `0x0800020c` is `prvPortStartFirstTask`:** Literal match on the
FreeRTOS ARM_CM4F port routine — reads the initial MSP from the vector table (`*_DAT_e000ed08`,
that is VTOR), enables IRQ/FIQ, executes DSB/ISB and triggers `SVC 0`; additionally
`_DAT_e000ed88` (CPACR) is OR-ed with `0xf00000` (CP10/CP11, FPU) — corresponds to the CM4F variant of
`prvPortStartFirstTask()`, which contains the FPU enable inline. Only caller: `xPortStartScheduler`.

**Fourth finding — `0x08001868` is a 4th CAN filter handler for BMS firmware updates:**
`HAL_CAN_RxFifo0MsgPendingCallback` dispatches by `FilterMatchIndex`: `==3` → `BMS_CAN_Parser`
(known), `==4` → `0x08001868`. The function checks PF byte patterns (bits 16–23) analogous to
`BMS_CAN_Parser` and writes status/size/data fields to `0x200038cb`–`0x200038d0` with
state values `2`/`4` — an init/verify state pattern that fits the
"BMS update over CAN" path documented in section 9 (CMD `0xCE`, type 0=init/1=data/2=verify). Named
`BMS_FW_Update_CAN_Handler` (filter 4 should additionally be added to table 4.1 —
only 3 filters had been documented there so far).

**5 newly assigned names (address → name, confidence):**

*High:*
`0x08003b04 HAL_ADC_Init`, `0x08003d54 HAL_ADC_MspInit`, `0x08003ff8 HAL_ADC_Start_DMA`,
`0x080035d4 HAL_ADCEx_Calibration_Start` (all four: literal match on the real STM32 HAL ADC
reference sequence incl. exact STM32F3 ADC base addresses 0x50000000/0x100/0x300/0x400/0x500/0x700),
`0x08004128 HAL_COMP_Init` (exact match on COMP1_CSR…COMP6_CSR, `0x40010200`–`0x214`),
`0x0800020c prvPortStartFirstTask` (literal match on FreeRTOS ARM_CM4F reference code,
only caller `xPortStartScheduler`).

*Medium-high:*
`0x08000fb4 ADC_Instances_Init` (clear aggregate init for 5 ADC handles, calls `HAL_ADC_Init` 5×),
`0x08001868 BMS_FW_Update_CAN_Handler` (4th CAN filter, PF byte pattern + init/verify state writes
matching the documented BMS update path).

*Medium:*
`0x08000b50 ADC_ConversionStop_WaitReady` (internal wait helper, called only by `HAL_ADC_Start_DMA`,
behavior matches `ADC_ConversionStop()` of the real HAL, but the target peripheral of the
external lock calls `0x08009a5c`/`0x08009a54` outside the tranche range cannot be conclusively
verified), `0x080027bc DMA_ChannelSelect_Compute` (computes channel index mask for
`GPIO_DMA_ChannelSetup`, target address `0x40020900` lies in a reserved zone between DMA2 and RCC not
documented in the STM32F3 reference manual — role clear, exact register identity not),
`0x080007d8 printf_negate_if_positive` (sign correction helper, only caller
`printf_number_format`, exact semantics within the float formatting cannot be fully
reconstructed), `0x08003af0 UART_TxEvent_ADC1_BootloaderThunk` (conditional jump into the
bootloader thunk area (`LAB_10001428`, new entry for the table in section 10) only if
instance==ADC1 base; caller is `UART_IRQHandler`, which points to a DMA callback structure shared
across domains — semantically unusual, but behavior clearly evidenced).

**Duplicate check:** All 12 names were checked before renaming against the complete named function list
of the entire program (`get-functions filterDefaultNames=true`, 370 functions reviewed before this
tranche, no collision). After renaming, verified again via `get-functions-by-similarity`
(search terms "HAL_ADC" and "HAL_COMP_Init") — name inventory rose by exactly 12 to 382,
**0 collisions**.

**Deliberately deferred (remain `FUN_<address>`, 6 functions):**

| Address(es) | Reason |
|---|---|
| `0x0800155c`, `0x08003afe`, `0x08003b00` | 2-byte stubs (`BX LR`, immediate return) — callers `Mode3_RoundRobin_Timer` or `UART_DMA_Handler`/`UART_IRQHandler`; genuine empty callback placeholders, not Ghidra boundary artifacts, but without content not sensibly nameable |
| `0x08002824`, `0x0800288c`, `0x080028e0` | **Dead code family** — `find-cross-references` returns `totalToCount=0` for all three (neither code nor data reference anywhere in the program). Structurally identical to the ADC/HAL init pattern (instance values `0x50000800`/`0xc00`/`0x1000`, calls to `0x08004528`/`0x080046fc`/`0x080047ee`/`0x080047bc`), but these instance addresses lie outside the officially documented STM32F3 ADC map (which ends at `0x700`) and are unreachable. Matches the independent assessment from tranche 4b (13.17), which likewise classifies the same function family around `0x08004528` as dead code — deliberately not named |

**Result of this tranche:** 12 newly named, 0 collisions, 6 functions deliberately deferred
(3 genuine stubs, 3 identified as dead code and independently confirmed by tranche 4b). The range
`0x0800020c–0x08004128` is thus considered completely processed for this session.

---

### 13.17 Tranche 4b (10.07.2026) — Address range 0x080043a8–0x08007496

> Processed range: subset of the CAN HAL/DMA HAL/FLASH/HRTIM/
> I2C/RCC/TIM cluster already processed multiple times in tranches 2a (13.12), 3a (13.13) and 3b (13.14). In this range,
> 29 `FUN_*` functions remained after tranche 3b (determined via `get-functions filterDefaultNames=false`),
> of which 27 had already been examined individually in 13.12/13.13/13.14 and deliberately deferred
> (dead code family, stubs, generic helpers, unclear register offsets). Methodology: each of the
> 29 functions was examined again via `get-decompilation` (incl. `includeCallers`/`includeCallees`) —
> with the goal of finding whether the name inventory that has grown since tranche 3c provides new
> clues in caller/callee chains (as expected in the task statement).

**Most important finding — the hint on `0x08004d3e` explicitly left open in 13.14 is only
partially resolved, but the context yields two independent new hits:** `0x08004d3e` is the **first**
call in `CAN1_Init`, before `Instance` is set to `0x40006400`. It calls the already
confirmed `CAN_ExitInitMode_WaitAck`, clears 2 bits of a peripheral register (instance offset
`+0x5c`), calls the known no-op stub `0x080051bc` (Msp-DeInit placeholder) and then resets both
the handle error code field (`+0x18`, word index) and the state byte (`+0x17`, word index) to `0`
— exactly the reset-to-`HAL_CAN_STATE_RESET` pattern of `HAL_CAN_DeInit`, called as a
preparation step at the start of `CAN1_Init`. Considerably clearer was `0x08004b7c`: its state guard
(`state==READY(1) || state==LISTENING(2)`, otherwise error code `|=2`) is a literal match on the
real `HAL_CAN_ActivateNotification()` check, the subsequent bit group comparisons (`0x7`, `0x38`,
`0x1c0`, `0x1e00`, `0xe000`, `0x30000`, `0xfc0000`) against a status register before conditionally setting
two IER-like bits correspond to the "avoid spurious IRQ on activation" pattern of
`HAL_CAN_ActivateNotification`, and both callers (`CAN1_Init` with `param_2=1`=TX mailbox empty IT,
`HAL_CAN_RxFifo0MsgPendingCallback` with `param_2=1`=re-arm after message processing) fit exactly
the expected call context. The uncertainty voiced in 13.13 ("third parameter without
HAL equivalent") does remain (param_3 additionally controls two customer-specific registers at
`+0xdc`/`+0xe0`), but this does not change the high confidence in the core role.

**Second finding — `0x08004b58` is a literal `HAL_Delay()` implementation:** tick start value via
`HAL_GetTick()`, conditional increase of the timeout value by a global tick frequency offset
(`_DAT_20000268`, except for `param_1==0xffffffff`), then busy-wait loop `while(GetTick()-start <
wait)` — exactly the reference code of `HAL_Delay()` from `stm32f3xx_hal.c`. The function is called both by
the (still dead-code-classified) family around `0x08004528` and by genuine live code
(`modbus_register_handler`, `0x080047ee`) — so despite a caller from the dead code family it is itself a generic, real HAL
building block and was named accordingly.

**Third finding — `0x0800485c` is the counterpart `HAL_DMA_Abort_IT` to the already
confirmed `HAL_DMA_Abort`:** The function lies directly before `HAL_DMA_Abort` (`0x080048c4`) in memory, checks
the same state byte offset (`+0x25`) for `BUSY(2)`, uses in the error path the same error code `4`
(`HAL_DMA_ERROR_NO_XFER`, already verified in 13.13 for `HAL_DMA_Abort`) and is — unlike the
blocking `HAL_DMA_Abort` — called directly from an ISR context (`UART_IRQHandler_Full`) without
waiting for a TC flag. This corresponds exactly to the difference between `HAL_DMA_Abort` (blocking)
and `HAL_DMA_Abort_IT` (ISR-safe, non-blocking) in the real STM32 HAL.

**4 newly assigned names (address → name, confidence):**

*High:*
`0x08004b58 HAL_Delay` (literal match on STM32 HAL reference implementation incl.
`0xffffffff` special case and tick frequency offset; callers both dead code family and
live code `modbus_register_handler`/`0x080047ee`), `0x0800485c HAL_DMA_Abort_IT` (state check
identical to `HAL_DMA_Abort`, same error code `4`, caller is ISR context
`UART_IRQHandler_Full`, lies directly before `HAL_DMA_Abort` in memory), `0x08004b7c
HAL_CAN_ActivateNotification` (state guard `READY|LISTENING` literally identical to the real HAL check,
bit group comparison pattern against status register before IER bit setting corresponds to the
spurious-IRQ avoidance pattern of the real function, both callers `CAN1_Init`/
`HAL_CAN_RxFifo0MsgPendingCallback` fit the expected context).

*Medium-high:*
`0x08004d3e HAL_CAN_DeInit` (reset of ErrorCode and state fields to `0`/RESET, calls
`CAN_ExitInitMode_WaitAck` + Msp-DeInit stub `0x080051bc`, called as the very first step of
`CAN1_Init` — structurally matches `HAL_CAN_DeInit`, even though the peripheral register offset used
`+0x5c` still does not match a standard bxCAN register according to the reference manual, see the uncertainty already
documented in 13.12/13.13 regarding this register range).

**Duplicate check:** All 4 names were checked before renaming against the complete function list of the
entire program (`get-functions filterDefaultNames=false`, all 5 pages × 100, 445/445
functions reviewed, none of the 4 target names existed before) — **0 collisions**. After
renaming, additionally verified via `get-decompilation signatureOnly=true` that the name
was actually adopted as the function name (not just a secondary label).

**Deliberately deferred (remain `FUN_<address>`, 25 functions):**

| Address(es) | Reason |
|---|---|
| `0x08004528` | Checked again: only 4 callers, all from the family already classified as dead code in 13.12/13.13 (`0x08002824`/`0x0800288c`/`0x080028e0`, referencing unmapped `0x50000800` addresses). Structure resembles a generic timing/baud rate configurator (HAL_GetTick timeout poll on `+0x34`, `HAL_Delay(1)`, packed 10-/8-bit fields at `+0x48`/`+0x4c`), but without a single caller with a valid peripheral base address no reliable naming is possible |
| `0x08004d08` | Checked again: callers still only `CAN1_Init` (state check `==READY`, fixed parameters `2,2,1,1`, writes packed bits at instance offset `+0x80` — lies in the bxCAN reserved range, no standard register). No new evidence compared to 13.13 |
| — | Additional check: `0x08004d08` can **not** be interpreted analogously to `0x08004b7c` as an "ActivateNotification" variant, since the state guard checks only `READY` (not `READY\|LISTENING`) — confirms the 13.13 assessment as a standalone, not clearly nameable function |
| `0x08004528` family: `0x080046fc`, `0x08004728`, `0x080047bc`, `0x080047ee`, `0x08004b58`* | *`0x08004b58` was named separately (see above), since it is independently also called by live code. `0x080046fc`/`0x08004728` checked again: `0x08004728` still literally references `0x50000800`/`0x50000c00`/`0x50001000` (not mapped in the F3 layout) — confirms the dead code classification unchanged. `0x080047bc`/`0x080047ee` not decompiled individually again, since part of the same already triple-verified family without new callers |
| `0x08004d6a`, `0x08004d6c`, `0x08004e74` | 2-byte stubs, sub-handler jump targets of `HAL_CAN_IRQHandler` — unchanged |
| `0x080051bc`, `0x080052c4`, `0x08005376`, `0x08005378`, `0x0800537a`, `0x0800537c`, `0x0800537e`, `0x08005380` | 2-byte stubs (padding/no-op), unchanged compared to 13.14. `0x080051bc` now additionally confirmed as Msp-DeInit placeholder within the newly named `HAL_CAN_DeInit` — but remains unnamed per the stub policy |
| `0x08005774` | Checked again: generic one-line bitmask test (`(*(param_1+0x10) & mask) != 0`), still called across domains from `EMS_Inverter_CAN_Dispatcher` (CMD `0x54`/`0xCB`) and `Mode3_RoundRobin_Timer` with different masks (`2`, `8`, `0x8000`) — too generic for a fitting name |
| `0x08006db4` | Checked again: trivial getter (`return _DAT_20000290;`), only caller now clearly identified as `0x08004528` (previously only suspected) — but since `0x08004528` itself belongs to the unclear dead code family, this getter also remains without a reliably assignable domain |
| `0x0800748c`, `0x0800748e`, `0x08007490`, `0x08007492`, `0x08007494`, `0x08007496` | 6× 2-byte stubs, no-op callbacks for unused `TIM20_CC_EventDispatch` channels — unchanged compared to 13.14 |

**Result of this tranche:** 4 newly named (of which 2 with very high confidence through literal
comparison with STM32 HAL reference code: `HAL_Delay`, `HAL_DMA_Abort_IT`), 0 collisions, 25 functions
deliberately deferred (14 stubs, 6 dead code family, 2 generic helpers without domain assignment,
2 CAN functions with unclear register offset despite plausible structural role). The range
`0x080043a8–0x08007496` is thus considered completely processed for this session.

---

### 13.18 Tranche 4c (10.07.2026) — Address range 0x08007d4a–0x080151c4

> Processed range: last remaining address range of the Micro FW per the task, coinciding
> with the end of tranche 3b (13.14) + completely with tranche 3c (13.15). Via `get-functions
> filterDefaultNames=false`, **29 `FUN_*` functions** were identified in the range — all 29 had
> already been examined individually in 13.14/13.15 and deliberately deferred. Methodology: each of the 29
> functions was checked again via `get-decompilation` (incl. `includeCallers`/`includeDisassembly` for the four
> suspect cases, plus `read-memory` raw-byte disassembly of the four "artifact" addresses by hand),
> with the goal of finding new evidence since 13.15.

**Most important finding 1 — the four addresses `0x080150d4/e8/fc/c4` are NOT empty Ghidra artifacts,
but real 10-byte MOVW/MOVT/BX jump veneers whose *target* code Ghidra wrongly displays as their own
function body:** Via `read-memory`, the raw bytes of all four addresses were
disassembled by hand. `0x080150d4`, for example, consists exactly of `MOVW r12,#0x9793`, `MOVT r12,#0x0800`,
`BX r12` (10 bytes, no further instructions) — a classic long-jump trampoline that jumps to
`0x08009793` (into the middle of the already named `UART_WaitOnFlagUntilTimeout`,
`0x080096b8`–`0x080097c4`). The "function bodies" shown by `get-decompilation` with
`unaff_r4`/`unaff_r6`/`unaff_r9`/`unaff_r10` are exactly the pseudocode of the jump target, which Ghidra's
decompiler displays inline when following the `BX` instruction — the assessment from 13.15
("not real, standalone functions") remains correct in its **practical consequence**
(renaming would still suggest a false function identity, since the "body" does not
belong to the address at all), but was imprecise in its **cause**: these are not misrecognized
function boundaries without any code, but tiny, really compiled interworking veneers
(presumably RVDS/Keil linker trampolines for calls beyond branch range or ARM/Thumb switching)
that should be treated as standalone 3-instruction functions, whose respective role, however, can
only be determined via the constantly changing jump target (not via their own code). `0x080150fc`
is demonstrably called 8× from the already named `Inverter_Grid_Control` with real parameters/
return values — further evidence that these addresses are real jump targets anchored in the control flow, not data. A meaningful individual naming (e.g. by the respective sub-role
within `UART_WaitOnFlagUntilTimeout`) would, however, require a complete reconstruction of the caller
bit conventions (registers `unaff_r6`/`r9`/`r10` are pre-loaded by the *caller*, not by the
veneer itself) — this exceeds the scope of this tranche. **Result unchanged: no renaming,
but re-assessment of the cause documented.**

**Most important finding 2 — `EEPROM_SaveOperatingStats` (`0x0800e9d4`, 4 bytes) can be named unambiguously despite its tiny size:** Unlike the
other 2–10-byte stubs in this range, the function does not consist of `return;`, but of a single meaningful call: `EEPROM_WriteVerify(0x500,
&DAT_20003e7f, 0x30)`. EEPROM address `0x500`/48 bytes is already documented in section 20.2 as the
"operating hours/statistics" block (the access column there already named this caller
as "Op_Hours"). The three callers (`build_telemetry_block`, `Mode3_RoundRobin_Timer`,
`Inverter_SetMode`) fit exactly the pattern "persist statistics block on telemetry build/cycle timer/
mode change". Duplicate check against `EEPROM_Save*`/the entire name space
(`get-functions-by-similarity`) — **0 collisions**. Renaming performed and verified via
`get-decompilation signatureOnly=true`.

**1 newly assigned name:**

| Address | Name | Confidence | Rationale |
|---|---|---|---|
| `0x0800e9d4` | `EEPROM_SaveOperatingStats` | High | One-line wrapper around `EEPROM_WriteVerify(0x500, &DAT_20003e7f, 0x30)`; EEPROM `0x500` already documented as operating hours/statistics block (20.2); 3 plausible callers (telemetry build, cycle timer, mode change) |

**Deliberately still open (28 functions, all already examined in 13.14/13.15, reconfirmed here):**

| Address(es) | Reason |
|---|---|
| `0x08007d4a`, `0x080080fc`, `0x08008324`, `0x08008326`, `0x0800832e`, `0x080083f4`, `0x080088d8` | 2-byte no-op stubs (`return;`), reconfirmed via disassembly/decompilation — pure jump target placeholders for unused `TIM20_CC_EventDispatch`/`UART_IRQHandler_Full` callback cases |
| `0x080082e4` | Checked again: the only caller `TIM20_CC_EventDispatch` has meanwhile been named (was still `FUN_08007a48` in 13.15), but that does not change the uncertainty — the function checks `Instance==0x40001400`(TIM7) and conditionally jumps to a bootloader thunk (`LAB_1000219c`, new jump target, not in the thunk table from section 10). Without knowing which of the approx. 12 TIM20 event bits (UIF/COM/Break/Break2/Trigger) this concretely is, the role remains too unspecific for a fitting name |
| `0x080088dc` (338 B) | Checked again, still `callerCount=0` (also confirmed via `find-cross-references`: no references). Mode-dependent (`DAT_20000035`) duty/timing table calculation for SRAM buffers at `0x20000a58`+offsets `0x19c`/`0x1a4`/`0x1a8`/`0x1ac`/`0x21c`/`0x224`/`0x228`/`0x22c` — presumably PWM DMA descriptors, but without callers not reliably nameable; possibly reached only indirectly via an SRAM function pointer |
| `0x08008d4c`, `0x08008d72`, `0x08008e06`, `0x08008ee4`, `0x08008f1c`, `0x08008f3c`, `0x080090f8` | Checked again: cluster confirmed as HRTIM timer unit register configurators (`param_2*0x80` indexing, offsets `0xb8`/`0xbc`/`0xc0`/`0xc4`/`0xc8`/`0xe4`/`0xec` relative to the timer unit base). `0x08008e06` now provides a clearer dispatch tree (bitmask cases `0x40/0x8/0x1/0x2/0x4/0x10/0x20/0x200/0x80/0x100/0x400/0x800` → 2 target registers), but without the STM32F3 HRTIM reference manual this is not enough for register-exact naming (CMP/SET/RST/OUT assignment remains hypothesis) |
| `0x08009a40`, `0x08009a4c`, `0x08009a54`, `0x08009a5c`, `0x08009a64`, `0x08009a74`, `0x08009a92` | Checked again (`0x08009a40`: `return *(uint*)(p1+p2*4+0x60) & 0x7c000000;`, exclusively from `HAL_GPIO_Init_Extended`) — confirms GPIO relation for the 4 exclusive accessors, but the 3 cross-domain ones (`0x08009a4c`/`54`/`5c`) still prevent a clean cluster split without guessing |
| `0x08011b10` | Checked again: `_DAT_20000458 = _DAT_20000024;`, only caller `UART_DMA_RxCplt_Dispatch` in the USART2 branch (`Instance==0x40004400`). Neither of the two SRAM variables is referenced elsewhere in the documentation — semantics remain unclear |
| `0x080150d4`, `0x080150e8`, `0x080150fc`, `0x080151c4` | **Real MOVW/MOVT/BX interworking veneers** (not "no function", see "Most important finding 1" above) — renaming remains deferred, since the individual role could only be determined via the jump target/caller register convention, not via their own code |

**Result of this tranche:** 1 newly named (`EEPROM_SaveOperatingStats`, high confidence), 0 collisions,
28 functions deliberately deferred (7× 2-byte stubs, 1× TIM7 bootloader dispatch without bit assignment,
1× PWM table function without callers, 7× HRTIM timer unit cluster without reference manual reconciliation,
7× GPIO/UART bitfield cluster with domain overlap, 1× one-line copy operation without semantics,
4× trampoline addresses newly characterized as real jump veneers (not as non-functions)). The
range `0x08007d4a–0x080151c4` is thus considered completely processed for this session — all 29
remaining `FUN_*` addresses were checked again with fresh evidence, without any additional
reliably nameable cases beyond `EEPROM_SaveOperatingStats`.

---

### 13.19 Tranche 5a (10.07.2026) — address range 0x0800155c–0x08005380

> Processed range: subrange of the fifth tranche (parallel agents process 5b
> `0x08005774–0x08008f3c` and 5c `0x080090f8–0x080151c4`). Via `get-functions filterDefaultNames=false`,
> **24 `FUN_*` functions** were identified in this range. All 24 had already been examined individually in
> 13.12/13.13/13.16/13.17 and deliberately deferred (dead code family,
> stubs, CAN reserved offset without register identity). Methodology: each function examined again via
> `get-decompilation` (incl. `includeCallers`/`includeCallees`), plus targeted
> `find-cross-references` applied to all 7 members of the function family classified as "dead code"
> (`0x08002824`, `0x0800288c`, `0x080028e0`, `0x08004528`, `0x080046fc`, `0x08004728`, `0x080047bc`,
> `0x080047ee`) to independently cross-check the classification from the assignment.

**Dead code family confirmed again (complete re-verification):** `find-cross-references` with
`direction=to` yields **`totalToCount=0`** for `0x08002824`, `0x0800288c` and `0x080028e0` (the three "root functions"
of the family) — neither a code nor a data reference anywhere in the entire
program. For the four dependent functions `0x08004528`, `0x080046fc`, `0x08004728`, `0x080047bc`
and `0x080047ee`, without exception **only** calls from exactly these three unreachable
root functions were found (e.g. `0x080047ee`: 3 callers, all `0x08002824`/`0x0800288c`/`0x080028e0`).
The entire family is thus confirmed as a self-contained code path completely cut off
from the rest of the program — consistent with the classification from the assignment and the tranches
13.12/13.13/13.17. No renaming, no further deep examination necessary.

> **Correction note on 13.17:** There, one of the three callers of `HAL_Delay` was described as
> "real live code (`modbus_register_handler`, `0x080047ee`)". This is a
> mix-up: `0x080047ee` is **not** `modbus_register_handler` (that one is actually located at
> `0x080124e8`, 2842 bytes, see section 16.1) — `0x080047ee` is a member of the dead code family
> (see above, called exclusively by `0x08002824`/`0x0800288c`/`0x080028e0`). According to `get-decompilation includeCallers=true`, the actual
> `HAL_Delay` callers are: `0x08004528` (2×,
> dead code family), `0x080047ee` (1×, dead code family) and `modbus_register_handler` (3×, real
> live code caller). The core statement of 13.17 (`HAL_Delay` is correctly named despite dead code callers,
> since it is also referenced by real live code) remains correct — only the address assignment in the
> parenthetical was wrong.

**1 newly assigned name (address → name, confidence):**

| Address | Name | Confidence | Rationale |
|---|---|---|---|
| `0x080043a8` | `HAL_COMP_MspInit` | High | Only caller `HAL_COMP_Init`, called exactly behind the classic HAL "MspInit only on first init" guard (`*(char*)(param_1+0x1d)==0`, followed by `param_1[7]=0; param_1[8]=0;` — reset of the state fields). Via three `HAL_GPIO_Init` calls (GPIOA/B/C, `0x48000000`/`0x400`/`0x800`) configures the analog input pin for COMP1 (`0x40010200`), COMP4 (`0x4001020c`) and COMP3 (`0x40010208`) respectively, and sets one "pin in use" bit each in a register at `0x4002104c`. Analogous to the already established naming scheme `HAL_ADC_MspInit`/`HAL_CAN_MspInit`. Duplicate check: `get-functions-by-similarity(searchString="HAL_COMP_MspInit")` across all 382 functions named at the start of the tranche — 0 collisions. Verified after renaming via `get-decompilation signatureOnly=true` (function name correctly applied, not just a secondary label). |

**Deliberately deferred (remain `FUN_<address>`, 23 functions):**

| Address(es) | Reason |
|---|---|
| `0x0800155c`, `0x08003afe`, `0x08003b00` | 2-byte stubs (`BX LR`), unchanged compared to 13.16 — genuine empty callback placeholders, not a Ghidra boundary artifact, but not meaningfully nameable without content |
| `0x08002824`, `0x0800288c`, `0x080028e0`, `0x08004528`, `0x080046fc`, `0x08004728`, `0x080047bc`, `0x080047ee` | **Dead code family**, fully re-verified in this tranche (see above) — `totalToCount=0` for all three root functions, all four dependent functions referenced exclusively internally. Confirmed unchanged |
| `0x08004d08` | Examined again: only caller still `CAN1_Init` (state check `==READY` at offset `+0x17`, fixed parameters `2,2,1,1`, writes packed 4-bit fields at instance offset `+0x80`). `+0x80` still lies in the bxCAN reserved area according to the reference manual, no new evidence found since 13.13/13.17 that would justify naming |
| `0x08004d6a`, `0x08004d6c`, `0x08004e74` | 2-byte stubs, sub-handler jump targets of `HAL_CAN_IRQHandler` (confirmed again via decompilation of `HAL_CAN_IRQHandler`: dispatch depending on ESR/IER bit group (TME/FMP1/FF/BOF/EWG/EPV/LEC/ERR)). The dispatch site can now be unambiguously assigned to a single flag group, but the stub body itself remains a plain `return;` without its own semantics — consistent with the stub policy deliberately not named speculatively after the IRQ flag |
| `0x080051bc`, `0x080052c4`, `0x08005376`, `0x08005378`, `0x0800537a`, `0x0800537c`, `0x0800537e`, `0x08005380` | 2-byte stubs (padding/no-op callbacks), unchanged compared to 13.14/13.17 |

**Result of this tranche:** 1 newly named (`HAL_COMP_MspInit`, high confidence), 0 collisions,
23 functions deliberately deferred (14 genuine 2-byte stubs, 8 dead code family fully
re-verified, 1 CAN function with a still unclear register offset). Additionally, a correction of an
address mix-up error from 13.17 documented (`0x080047ee` ≠ `modbus_register_handler`). The
range `0x0800155c–0x08005380` is thus considered completely processed for this session — the
declining success rate (12→9→30→17→5→4→1→1) confirms that the name inventory in this
address range is approaching its static limit.

---

### 13.20 Tranche 5b (10.07.2026) — address range 0x08005774–0x08008f3c

> Processed range: subrange of the fifth tranche (parallel agents process 5a
> `0x0800155c–0x08005380` and 5c `0x080090f8–0x080151c4`). Via `get-functions filterDefaultNames=false`,
> **25 `FUN_*` functions** were identified in this range. A large part had already been examined individually in
> 13.9/13.13/13.14/13.17/13.18 and deliberately deferred (generic
> bitmask test, dead code family getters, 2-byte stubs). The focus of this tranche was the
> timer unit cluster `0x08008d4c`–`0x080090f8` explicitly classified in 13.18 as "not nameable with register precision without the HRTIM reference manual"
> (assignment specification) as well as two previously unexamined functions `0x08005db0`/`0x08005df0`, which in the old (address-drift-affected) documentation
> proposal of section 13.3 were wrongly assumed to be generic I2C functions.

**Most important finding — HRTIM timer unit cluster fully resolved via reference manual offset
reconciliation:** The cluster consistently uses `param_2*0x80` indexing (timer unit selection, 6 units
A–F) relative to the HRTIM instance base. Through systematic reconciliation of the byte offsets used against
the real STM32F3/F334 `HRTIM_TypeDef` memory layout (master block `0x00`–`0x28`: MCR/MPER/MREP;
timer unit blocks from `0x80` in a `0x80` grid with TIMxCR/TIMxPER/TIMxREP/TIMxDTR/TIMxSETR1/RSTR1/
SETR2/RSTR2/TIMxFLTR/TIMxOUTR; common block from `0x380` with BMCR at `0x3a0`), all 6
cluster functions could be assigned to concrete register groups (see table below and section 14.4). Particularly
clear: `HRTIM_TimerUnit_OutputConfig` (`0x08008e06`) branches on a bitmask whose values
(`0x1,0x2,0x4,0x8,0x10,0x20,0x40,0x80,0x100,0x200,0x400,0x800`) correspond exactly to the real HAL constants
`HRTIM_OUTPUT_TA1…TF2` — Output1 values (odd nibble position) go into SETx1R/RSTx1R,
Output2 values into SETx2R/RSTx2R, followed by a common update of TIMxOUTR. `HRTIM_TimerUnit_WaveformConfig`
(`0x08008f3c`, at 432 bytes the largest function of the cluster) additionally sets, per timer unit, a
BMCR bit exactly at position `1+param_2` — matches bit-for-bit the real `BMCR.TxBM` field
(burst mode timer update enable). Details and complete register table see section 14.4.

**Second finding — `0x08005db0`/`0x08005df0` also belong to the HRTIM cluster, not to I2C:** The
old documentation proposal (13.3), already marked several times as unreliable, assumed `I2C_Mem_Read`
at this address. In fact `0x08005df0` lies directly between the confirmed
`HRTIM_GPIO_AF_Init` (ends `0x08005da0`) and the real I2C functions (start `0x0800617c`), and
directly calls `HRTIM_TimerUnit_ChopperConfig` (`0x08008f1c`) — unambiguous structural evidence
of HRTIM membership. `0x08005db0` (directly before it, same handle access pattern via
instance offset `+0x388` = common block `+0x08`) was named accordingly as a sibling function.
According to `find-cross-references`, both functions currently have **no** further callers in the program —
this matches the finding already documented for `HRTIM_TimerUnit_Init`/`HRTIM_GPIO_AF_Init`
(also `callerCount=0`) that the entire HRTIM driver block is only reached indirectly (function pointers/
external bootloader code) or not at all — the absence of callers was therefore, as with the
already established HRTIM names, not treated as an exclusion criterion.

**8 newly assigned names (address → name, confidence):**

| Address | Name | Confidence | Short rationale |
|---|---|---|---|
| `0x08008d4c` | `HRTIM_MasterTimer_BaseConfig` | High | MCR prescaler bits + MPER(`+0x14`) + MREP(`+0x18`) — exact offset match on `HRTIM_Master_TypeDef` |
| `0x08008d72` | `HRTIM_MasterTimer_WaveformConfig` | High | MCR mode/sync bits + BMCR(`+0x3a0`) burst enable |
| `0x08008e06` | `HRTIM_TimerUnit_OutputConfig` | High | Bitmask values = real `HRTIM_OUTPUT_Tx1/Tx2` constants, SETx1R/RSTx1R or SETx2R/RSTx2R + TIMxOUTR |
| `0x08008ee4` | `HRTIM_TimerUnit_BaseConfig` | High | TIMxCR prescaler bits + TIMxPER + TIMxREP, counterpart to `HRTIM_MasterTimer_BaseConfig` |
| `0x08008f1c` | `HRTIM_TimerUnit_ChopperConfig` | Medium | Conditional write to an offset matching the TIMxCHPR location/field width, register identity not 100% verified |
| `0x08008f3c` | `HRTIM_TimerUnit_WaveformConfig` | High | Combines TIMxCR/TIMxFLTR/TIMxOUTR/BMCR.TxBM (bit position exactly `1+param_2`) |
| `0x08005db0` | `HRTIM_WaitOnFlagUntilTimeout` | Medium | HAL wait-timeout pattern on common ISR location (`+0x388`), return `0`/`3` like `HAL_StatusTypeDef` |
| `0x08005df0` | `HRTIM_TimerUnit_LockedUpdate` | Medium | Software lock guard around the call of `HRTIM_TimerUnit_ChopperConfig` |

**Duplicate check:** Before renaming, all 445 function names of the program were loaded via
`get-functions filterDefaultNames=false` and a Python script (`run-script`) checked each of the 8
target names against this complete name set — **0 collisions**. The renaming was done
in a single Ghidra transaction (`setName`, `SourceType.USER_DEFINED`); all 8 responses
confirm `old → new`.

**Deliberately deferred (remain `FUN_<address>`, 17 functions):**

| Address(es) | Reason |
|---|---|
| `0x08005774` | Examined for the fourth time (13.9, 13.17, now again): generic one-line bitmask test (`(*(p1+0x10)&p2)!=0`), still called from two unrelated contexts (`EMS_Inverter_CAN_Dispatcher` CMD `0x54`/`0xCB`, `Mode3_RoundRobin_Timer`) with different masks — no new evidence, too generic |
| `0x08006db4` | Examined again: trivial getter (`return _DAT_20000290;`), only caller still the unreachable dead code family (`FUN_08004528`, see 13.19 re-verification) — no reliably assignable domain |
| `0x0800748c`, `0x0800748e`, `0x08007490`, `0x08007492`, `0x08007494`, `0x08007496` | 6× 2-byte no-op stubs, unchanged jump target placeholders for unused `TIM20_CC_EventDispatch` channels |
| `0x08007d4a` | 2-byte stub, unchanged |
| `0x080080fc` | 2-byte stub, unchanged |
| `0x080082e4` | Only caller `TIM20_CC_EventDispatch`; checks `Instance==0x40001400`(TIM7) and conditionally jumps to a bootloader thunk — without knowing which TIM20 event bit this specifically is, still too unspecific |
| `0x08008324`, `0x08008326`, `0x0800832e`, `0x080083f4` | 2-byte no-op stubs, sub-handler jump targets of `UART_IRQHandler_Full`, unchanged |
| `0x080088d8` | 2-byte stub, unchanged |
| `0x080088dc` | Examined again, still `callerCount=0`/`totalToCount=0` (confirmed via `find-cross-references`) — mode-dependent duty/timing table calculation without a reachable caller, no reliable naming possible |

**Result of this tranche:** 8 newly named (6× high, 2× medium — of which 6 fully resolve the HRTIM timer unit cluster
named in 13.18 as a core problem), 0 collisions, 17 functions deliberately
deferred (14× genuine 2-byte stubs, 1× generic bitmask test re-examined unsuccessfully four times,
1× dead code family getter, 1× TIM7 dispatch without bit assignment, 1× table function without callers).
Ghidra naming status after this tranche: `get-function-count filterDefaultNames=true` = 392 of 445
(88.1%). The range `0x08005774–0x08008f3c` is thus considered completely processed for this session.

---

### 13.21 Tranche 5c (10.07.2026) — address range 0x080090f8–0x080151c4

> Processed range: last remaining address range according to the assignment (parallel to 5a
> `0x0800155c–0x08005380` and 5b `0x08005774–0x08008f3c`). Coincides with the end of 13.15/13.18.
> Via `get-functions filterDefaultNames=false`, **14 `FUN_*` functions** were identified in the range:
> `0x080090f8`, `0x08009a40`, `0x08009a4c`, `0x08009a54`, `0x08009a5c`, `0x08009a64`, `0x08009a74`,
> `0x08009a92`, `0x0800b958`, `0x08011b10`, `0x080150d4`, `0x080150e8`, `0x080150fc`, `0x080151c4`.
> All 14 had already been examined individually in 13.14/13.15/13.18 and deferred. Methodology: each
> function re-examined via `get-decompilation` (incl. `includeCallers`); for the 4 veneer addresses
> additionally raw byte disassembly via `read-memory` + manual Thumb-2 MOVW/MOVT decoding to determine
> the exact jump targets (continuation of 13.18, which had located the targets only roughly).

**Most important finding 1 — doc/Ghidra drift discovered and fixed: `TIM_Base_Init` (`0x0800b958`) had been
documented since 13.4 but never applied in Ghidra.** The function had already been analyzed
correctly in the initial survey (register whitelist exactly for the TIM1/2/3/4/5/8/15/16/20 instance bases `0x40012c00`/
`0x40000000`/`0x40000400`/`0x40000800`/`0x40000c00`/`0x40013400`/`0x40014000`/`0x40014400`/
`0x40014800`/`0x40015000`; writes CR1 bits, ARR (`param_1[0xb]`), PSC (`param_1[10]`) and
conditionally RCR (`param_1[0xc]`) for the "advanced" timers — classic `TIM_Base_InitTypeDef` pattern),
but the `setName` call was never performed (Ghidra still showed `FUN_0800b958`, 4 callers
`TIM7_Channels_EnableAll`/`TIM_Handle_Init`/`TIM20_Channels_EnableAll`/`TIM4_Channels_EnableAll` —
all timer channel enabling functions that first configure the base registers
before enabling). Duplicate check: `TIM_Base_Init` does not occur anywhere in the entire symbol space (only as
free-text mention in the documentation itself). Renaming performed and verified via `get-decompilation
signatureOnly=true` (`"functionName":"TIM_Base_Init"`).

**Most important finding 2 — two further stale doc tables with never-applied names identified and
corrected:** Analogous to the pattern already documented in 13.9 ("398/445 claimed, 21 real"), section 13.4 (`0x08009a4c`/`54`/`5c` as supposedly `DMA_Get_TCIF`/`DMA_Get_Error_Flag`/`DMA_Get_HTIF`)
and section 13.8 (`0x080150d4`/`e8`/`fc`/`c4` as supposedly `CAN_TxMailbox_SetDLC`/
`CAN_RxFilter_Config`/`float_to_fixed_point`/`CAN_frame_pack`) contained names that were in reality **never
set in Ghidra** (all 7 addresses still showed `FUN_*` in `get-functions filterDefaultNames=false`). Both tables were corrected with ❌/strikethrough and refer to this section.
The "DMA" hypothesis for `0x08009a4c`/`54`/`5c` is additionally refuted on substance (see finding 3) — they are
not DMA registers, but cross-domain bitfield extractors.

**Most important finding 3 — GPIO/UART bitfield cluster (`0x08009a40`–`0x08009a92`) re-examined, one
cluster member newly identified as UART-exclusive (not GPIO):** Via `includeCallers=true`, the complete caller list was pulled again
for each of the 7 functions:

| Address | Body | Callers | Domain |
|---|---|---|---|
| `0x08009a40` | `*(uint*)(p1+p2*4+0x60) & 0x7c000000` | exclusively `HAL_GPIO_Init_Extended` (12×) | GPIO |
| `0x08009a4c` | `(*(uint*)(p1+8) & 0xf) >> 3` | `HAL_GPIO_Init_Extended` + `HAL_ADC_Init` | GPIO **and** ADC |
| `0x08009a54` | `*(uint*)(p1+8) & 1` | `UART_WaitReady`, `ADC_ConversionStop_WaitReady`, `HAL_GPIO_Init_Extended`, `HAL_ADC_Init` | UART **and** ADC **and** GPIO |
| `0x08009a5c` | `(*(uint*)(p1+8) & 7) >> 2` | `HAL_GPIO_Init_Extended`, `HAL_ADC_Init`, `HAL_ADC_Start_DMA` | GPIO **and** ADC |
| `0x08009a64` | `(p1[0xc] & 0xc00) != 0 ? 0 : 1` | **exclusively `UART_IRQHandler`** | UART (not GPIO!) |
| `0x08009a74` | writes `*(p1+0x14)` (3-bit field) | exclusively `HAL_GPIO_Init_Extended` (3×) | GPIO |
| `0x08009a92` | writes `*(p1+p2*4+0x60)` bit 31 | exclusively `HAL_GPIO_Init_Extended` (4×) | GPIO |

`0x08009a64` had been counted wholesale among the "4 exclusive GPIO accessors" in 13.11/13.15/13.18
(together with `0x08009a40`/`74`/`92`) — this is **refuted**: its only caller is
`UART_IRQHandler`, not `HAL_GPIO_Init_Extended`. The 3 cross-domain functions
(`0x08009a4c`/`54`/`5c`) each extract a fixed bit/bitfield from offset `+8` of a generic
parameter — but an identical bit pattern is queried by UART, ADC and GPIO init code from **different**
peripheral handles. This is typical for RVDS/Keil identical code folding (the
compiler/linker deduplicates byte-identical short function bodies across module boundaries) — genuine
GPIO- or UART-specific register semantics can no longer be reconstructed from this without guessing.
All 7 therefore remain unnamed; the correction for `0x08009a64` was added to the cluster table in 13.4
(see below).

**Most important finding 4 — raw byte decoding confirms and refines 13.18 for the 4 veneer addresses,
but `thunk_<target>` naming is still not meaningful:** Manual Thumb-2 decoding of the MOVW/MOVT/BX
byte sequences (`read-memory`, 10 bytes per address) yields exact jump targets:

| Veneer | MOVW/MOVT target | Located in function | Callers of the veneer |
|---|---|---|---|
| `0x080150d4` | `0x08009792` | `UART_WaitOnFlagUntilTimeout` (0x080096b8–0x080097c3) | `callerCount=0` (only data references from the shell command table `0x0801892c`ff., see 7.1) |
| `0x080150e8` | `0x08009774` | `UART_WaitOnFlagUntilTimeout` | `callerCount=0` (ditto, data references from shell table) |
| `0x080150fc` | `0x0800707c` | `RCC_OscConfig` (0x08006e34–0x0800728b) | `Inverter_Grid_Control`, 8× (power/protection threshold comparisons) |
| `0x080151c4` | `0x08007058` | `RCC_OscConfig` | `callerCount=0` (2 data references at `0x0801a66c`/`0x0801a714`) |

This concretely evidences the "interworking veneer" mechanism assumed in 13.18 — but the jump targets
land **in the middle** of `UART_WaitOnFlagUntilTimeout` and `RCC_OscConfig`, exactly in their generic
`HAL_GetTick` timeout wait loop (`while(...) { if (5000 < HAL_GetTick()-t0) return 3; }` — the
error code `3` corresponds to the classic `HAL_TIMEOUT`). This tail sequence is contained byte-identically in several
otherwise independent functions and was apparently cross-jump-optimized by the RVDS/Keil linker
(several callers jump into the same physical instruction sequence, which only happens to lie at the end of
`RCC_OscConfig`/`UART_WaitOnFlagUntilTimeout`). The caller `Inverter_Grid_Control` (at
`0x080150fc`) has nothing to do with oscillator configuration in substance — naming it
`thunk_RCC_OscConfig` or `thunk_UART_WaitOnFlagUntilTimeout` (analogous to `thunk_CAN_Filter_Setup`)
would therefore falsely suggest that these veneers call the semantic function (clock config or
waiting for UART flag) — in fact they only use a generic
timeout wait loop that happens to be compiled there. This fundamentally distinguishes them from `thunk_CAN_Filter_Setup` (`0x080150f2`),
which cleanly jumps to the **function start** of `CAN_Filter_Setup` (`0x0800eb88`). **Result
unchanged compared to 13.18: no renaming**, but now evidenced with exact jump target addresses instead of
only roughly located.

**1 newly assigned name:**

| Address | Name | Confidence | Rationale |
|---|---|---|---|
| `0x0800b958` | `TIM_Base_Init` | High | Correctly analyzed since the initial survey (13.4), but never applied in Ghidra (doc/Ghidra drift, see finding 1). Register whitelist exactly for the 9 STM32F3 TIM instance bases (TIM1/2/3/4/5/8/15/16/20); writes CR1/ARR/PSC/conditionally RCR — classic `TIM_Base_InitTypeDef` pattern. 4 callers are all already named `TIM*_Channels_EnableAll`/`TIM_Handle_Init` functions. 0 collisions (`TIM_Base_Init` did not occur in the symbol space). Verified via `get-decompilation signatureOnly=true`. |

**Deliberately still open (13 functions):**

| Address(es) | Reason |
|---|---|
| `0x080090f8` | Examined again incl. `find-cross-references` (`totalToCount=0`, `totalFromCount=0` — completely unreferenced). HRTIM timer unit register writer (`param_2*0x80+0xec` indexing, conditional additional block at `param_3[6]==1`), part of the HRTIM cluster `0x08008d4c`–`0x080090f8` documented in 13.18. Without callers and without reconciliation against the STM32F3 HRTIM reference manual not reliably nameable |
| `0x08009a40`, `0x08009a4c`, `0x08009a54`, `0x08009a5c`, `0x08009a64`, `0x08009a74`, `0x08009a92` | GPIO/UART/ADC bitfield cluster, complete caller matrix newly compiled (see finding 3). 4 functions are GPIO-exclusive (`0x08009a40`/`74`/`92`) or UART-exclusive (`0x08009a64`, **newly corrected** — was wrongly classified as GPIO), 3 are identical across domains (`0x08009a4c`/`54`/`5c`, presumably compiler identical code folding). Register offset `+8`/`+0x14`/`+0x60` cannot be named with register precision without reconciling the underlying custom struct against the vendor manual (no standard `GPIO_TypeDef` layout, since offset `0x60` does not exist there) |
| `0x08011b10` | Examined again: `_DAT_20000458 = _DAT_20000024;`, only caller still `UART_DMA_RxCplt_Dispatch` (USART2 branch). No new evidence since 13.15/13.18 — both SRAM variables remain without documented semantics |
| `0x080150d4`, `0x080150e8`, `0x080150fc`, `0x080151c4` | **Confirmed interworking veneers with now exactly known jump targets** (see finding 4) — renaming as `thunk_<target>` deliberately omitted, since the jump targets land in generic, cross-jump-optimized timeout code of unrelated functions and such a name would actively mislead about the actual semantics (unlike `thunk_CAN_Filter_Setup`, which cleanly points to a function start) |

**Result of this tranche:** 1 newly named (`TIM_Base_Init`, high confidence — doc/Ghidra drift fixed),
0 collisions, 13 functions deliberately deferred (1 unreferenced HRTIM register writer, 7
GPIO/UART/ADC bitfield clusters with newly refined caller matrix, 1 one-line copy operation without
semantics, 4 interworking veneers with now exactly decoded but misleading jump targets). Additionally,
2 stale doc tables (13.4, 13.8) with names never applied in Ghidra identified and corrected.
The range `0x080090f8–0x080151c4` is thus considered completely processed for this session.

---

## 14. New Findings from Mass Analysis

### 14.1 Modbus RS485 Slave (Fully Decoded)

The Micro MCU runs its **own Modbus slave** over USART1/USART2 (independent of the TCP Modbus of the Control MCU):
- `Modbus_Process_Request` (0x0800ac0c) — dispatcher for FC03/FC06/FC10
- `CRC16_Modbus` (0x0800abda) — standard CRC-16/Modbus (poly 0xA001, init 0xFFFF)
- `modbus_register_handler` (0x080124e8, **2842 bytes!**) — massive register handler
- `modbus_read_register_block` (0x080118e8) — float→int scaling for register reads

### 14.2 Debug Log System (Ring Buffer)

200-slot ring buffer at SRAM `0x20001A2C` with ~60 event types:
- `debug_log_enqueue` → insert timestamped event
- `debug_log_dequeue` → fetch next entry
- `debug_log_format_entry` (1712 bytes) — huge switch for all event types

### 14.3 EEPROM Configuration (I2C)

I2C EEPROM (addresses 0xA0/0xA2) for persistent configuration:
- `EEPROM_LoadConfig` (614B) — load all settings at boot
- `EEPROM_WriteVerify` (300B, **8 calls**) — write with byte verification
- `EEPROM_SaveTimestamp` — save tick counter in register 0x80
- `EEPROM_SaveOperatingStats` (`0x0800e9d4`, 4B, tranche 4c) — wrapper `EEPROM_WriteVerify(0x500, &DAT_20003e7f, 0x30)`,
  saves the 48-byte operating hours/statistics block (EEPROM `0x500`, see 20.2); callers: `build_telemetry_block`,
  `Mode3_RoundRobin_Timer`, `Inverter_SetMode`

### 14.4 HRTIM (High-Resolution Timer)

Extensive HRTIM use for the PWM control of the H-bridge and the LLC converter — 20+ functions for HRTIM configuration (deadtime, burst mode, ADC trigger, waveform). This confirms that the inverter control runs directly on this MCU.

**Addendum tranche 5b (10.07.2026) — timer unit cluster `0x08008d4c`–`0x08008f3c` resolved register-exactly:**
By reconciling the `param_2*0x80` indexing and the byte offsets against the real
`HRTIM_Timerx_TypeDef`/`HRTIM_Master_TypeDef`/`HRTIM_Common_TypeDef` memory layout (STM32F3/F334
reference manual RM0364), the cluster previously classified as "not nameable register-exactly without reference manual"
(see 13.18) could be resolved:

| Address | Name | Register assignment | Confidence |
|---|---|---|---|
| `0x08008d4c` | `HRTIM_MasterTimer_BaseConfig` | writes MCR (prescaler bits, offset `+0x00`) + MPER (`+0x14`) + MREP (`+0x18`) — no `param_2`, hence master timer (no timer unit index) | High |
| `0x08008d72` | `HRTIM_MasterTimer_WaveformConfig` | writes MCR (mode/sync bits) + BMCR (`+0x3a0` = common block base `0x380`+`0x20`, burst mode enable) | High |
| `0x08008e06` | `HRTIM_TimerUnit_OutputConfig` | depending on the output bitmask (`0x1,0x4,0x10,...`=Output1 → SETx1R/RSTx1R `+0x3c/+0x40`; `0x2,0x8,0x20,...`=Output2 → SETx2R/RSTx2R `+0x44/+0x48`, each relative to the timer unit base) selects and updates TIMxOUTR (`+0x64`) — bitmask values correspond exactly to the real `HRTIM_OUTPUT_Tx1/Tx2` constants (`0x1…0x800`) | High |
| `0x08008ee4` | `HRTIM_TimerUnit_BaseConfig` | writes TIMxCR (prescaler bits) + TIMxPER (`+0x14` rel.) + TIMxREP (`+0x18` rel.) per timer unit (`param_2`-indexed, `0x80` stride) — counterpart to `HRTIM_MasterTimer_BaseConfig` | High |
| `0x08008f1c` | `HRTIM_TimerUnit_ChopperConfig` | conditional write (only if ready bit set) to offset `+0x6c` rel. to the timer unit base (bits 6–15) — position/field width matches TIMxCHPR (chopper register), register identity not 100% verified against the reference manual | Medium |
| `0x08008f3c` | `HRTIM_TimerUnit_WaveformConfig` | largest function (432B) of the cluster: combines TIMxCR mode bits, TIMxFLTR (`+0x68` rel.), TIMxOUTR (`+0x64` rel.), a chopper-like field (`+0x54` rel.) and BMCR burst enable bit (`1<<(1+param_2)`, exactly matching `BMCR.TxBM`) — counterpart to `HRTIM_MasterTimer_WaveformConfig` | High |

In addition, two directly adjacent helper functions linked to the cluster via calls:

| Address | Name | Description | Confidence |
|---|---|---|---|
| `0x08005db0` | `HRTIM_WaitOnFlagUntilTimeout` | Classic HAL wait pattern (`HAL_GetTick` timeout poll on a bit at instance offset `+0x388` = common block `0x380`+`0x08`/ISR location; return `0`=OK/`3`=TIMEOUT like `HAL_StatusTypeDef`), structurally placed directly next to the confirmed HRTIM init functions (`HRTIM_TimerUnit_Init`/`HRTIM_GPIO_AF_Init`) | Medium |
| `0x08005df0` | `HRTIM_TimerUnit_LockedUpdate` | Software lock guard (state byte pattern `0/1/2`, analogous to HAL `Lock`/`State` field) around a single call of `HRTIM_TimerUnit_ChopperConfig` — only caller/callee evidence for the cluster membership of these two functions | Medium |

This fully names the HRTIM cluster classified as unclear in 13.18 (8 of 8 functions open there).

### 14.5 DSP Functions (Control Engineering)

- `PI_controller_step` / `PI_controller_reset` — PI controller with anti-windup
- `biquad_filter_design` / `notch_filter_design` — IIR filters for grid frequency
- `atan_normalized` — polynomial atan(x)/π approximation (phase calculation)
- `compute_reactive_power_ref` — reactive power calculation from power factor

### 14.6 RTC & Watchdog

- `RTC_Read_Time` — real-time clock via I2C with BCD conversion
- `IWDG_Reload` — independent watchdog (key 0xAAAA)
- `Operating_Hours_Update` — operating hours counter (3600s interval)

---

## 16. RS485 Modbus Register Map (`modbus_register_handler`)

### 16.1 Architecture

The Micro MCU serves RS485 Modbus requests via **two separate paths**:

| Register range | Handler | Description |
|---|---|---|
| **< 40000** (0x0000–0x9C3F) | `modbus_read_register_block` + descriptor table | Telemetry (read-only) |
| **≥ 40000** (0x9C40+) | `modbus_register_handler` (2842 bytes) | Control (R/W) |

**Function signature:**
```c
int modbus_register_handler(void *pRegMap, void *pRequest, int isWrite, int writeValue)
// pRegMap   = output buffer for read response
// pRequest  = Modbus register address (16-bit, directly as integer)
// isWrite   = 0: read (FC03), 1: write (FC06/FC10)
// writeValue = value to write (only if isWrite=1)
// Return: 0=OK, 1=read-only/mode error, 2=unknown register, 3=invalid value
```

**Supported Modbus function codes:**
- FC03 (Read Holding Registers): both paths
- FC06 (Write Single Register): only `modbus_register_handler`
- FC10 (Write Multiple Registers): only `modbus_register_handler`

### 16.2 Telemetry Descriptor Table (Register < 40000)

The telemetry registers are served via a **runtime-built descriptor table** at SRAM `0x200004C0`. Up to **70 entries** (loop limit `0x46`), **12 bytes** each:

```c
struct ModbusRegDescriptor {    // 12 bytes, at SRAM 0x200004C0 + n*12
    uint16_t base_address;      // +0: base register address
    uint16_t _pad;              // +2: (padding)
    uint32_t data_ptr;          // +4: pointer to SRAM data array
    uint8_t  data_type;         // +8: data type (see below)
    uint8_t  elem_size_flags;   // +9: low nibble = element size (bytes), high = flags
    uint8_t  scale_factor;      // +10: scaling
    uint8_t  element_count;     // +11: number of elements
};
```

**Data types** (byte +8):

| Value | Type | Description |
|---|---|---|
| 0x01 | u8 | Unsigned byte |
| 0x02 | u16 | Unsigned 16-bit |
| 0x04 | u32 | Unsigned 32-bit |
| 0x11 | i8 | Signed byte |
| 0x12 | i16 | Signed 16-bit |
| 0x14 | i32 | Signed 32-bit |
| 0x24 | float | 32-bit float (→ int conversion via `VectorFloatToUnsigned`) |
| 0x31 | ASCII | Byte with uppercase shift (> 0x60 → -0x20) |

**Scaling** (byte +10):

| Value | Operation | Example |
|---|---|---|
| 0 | None | — |
| 1 | ×10 | 23.5 → 235 |
| 2 | ×100 | 2.35 → 235 |
| 3 | ÷10 | 235 → 23 |
| 4 | ÷100 | 235 → 2 |
| 5 | Negation | 100 → -100 |

> **Note:** The descriptor table is built by code in the **external flash (0x10000000+)**, which is not contained in the analyzed binary. The concrete telemetry register addresses can therefore not be extracted from this analysis.
> 
> **CORRECTION 2026-08-16:** Too absolute. The **entry format** is fully reconstructed from `modbus_read_register_block` (@0x080118e8): 12 bytes per entry — reg@+0, source pointer (SRAM)@+4, type@+8, elem_size@+9, scale code@+0xa, count@+0xb (scale 0x01=×10 … 0x04=×0.01), max 70 entries. The *builder* is confirmed to reside in the separate 0x10000000 region (not in the .bin; FileBytes=Mapped). The register **contents**, however, can be reconstructed — as with the Control — via live scan + debug print field names. See `Methodik_und_Meta/Docs_Audit_Open_Items_2026-08-16.md`.

### 16.3 Control Registers (≥ 40000) — Complete Map

All registers from `modbus_register_handler` (0x080124E8), decoded from 788 lines of decompilation.

#### 16.3.1 System Control (40000–41200)

| Register | Hex | R/W | Description | Valid values | SRAM | EEPROM |
|---|---|---|---|---|---|---|
| 40000 | 0x9C40 | **W** | FW update / self-test | `0x55AA` = FW update (work_mode→4), `0x55A1`–`0x55A6` = self-test slot 1–6 | `inv_work_mode` | — |
| 41000 | 0xA028 | R/W | Reset enable flag | W: `0x55AA` = set flag; R: flag value | `DAT_200004B8` | — |
| 41001 | 0xA029 | R/W | Clear statistics | W: `0x55AA` = EEPROM_ClearStats(3) + flag | `DAT_200004B8` | clears |
| 41010 | 0xA032 | R/W | Calibration status | R: calibration flag; W: `800` = limit 800W + calibration, `2500` = reset | `DAT_20000497` | 0x90 |
| 41100 | 0xA08C | R/W | Config byte | value 0–255, stored in EEPROM | `DAT_200004A5` | 0x101 |
| 41200 | 0xA0F0 | R/W | Backup mode | R: inverted (0 = active); W: `0` = backup on, `1` = backup off | `backup_mode_flag` | — |

#### 16.3.2 Work Mode & Basic Config (42000–42011)

| Register | Hex | R/W | Description | Valid values | SRAM | EEPROM |
|---|---|---|---|---|---|---|
| 42000 | 0xA410 | R/W | Work mode persist | `0x55AA` = ACK, **`0x55BB`** = save to EEPROM | `inv_work_mode` | 0x301 |
| 42010 | 0xA41A | R/W | State variable | 0–2 valid | `DAT_200004B9` | — |
| 42011 | 0xA41B | R/W | Modbus slave address | 11–100 (0x0B–0x64) | `DAT_200004BA` | — |

> **Important:** Register 42000 with value `0x55BB` is the known **RS485 unlock** — identical to the Control FW behavior (there reg 42000 = 0x55AA for write enable).

#### 16.3.3 Schedule / Timer Slots (43100–43124)

**5 slots × 5 registers** for time-controlled charge/discharge scheduling. SRAM base: `0x20003E05` (slot stride: 10 bytes).

| Offset | Register (slot n) | R/W | Description | SRAM offset | Values |
|---|---|---|---|---|---|
| +0 | 43100 + n×5 | R/W | End time | slot×10 + 0x20003E0B | HH:MM as `(hour<<8)\|min` |
| +1 | 43101 + n×5 | R/W | Start time | slot×10 + 0x20003E09 | HH:MM as `(hour<<8)\|min` |
| +2 | 43102 + n×5 | R/W | Charge power | slot×10 + 0x20003E07 | i16, ±2500 (W) |
| +3 | 43103 + n×5 | R/W | Enable | slot×10 + 0x20003E06 | 0/1 |
| +4 | 43104 + n×5 | R/W | Mode | slot×10 + 0x20003E05 | Operating mode |

**Slot addresses:**

| Slot | Register | SRAM base |
|---|---|---|
| 0 | 43100–43104 | 0x20003E05 |
| 1 | 43105–43109 | 0x20003E0F |
| 2 | 43110–43114 | 0x20003E19 |
| 3 | 43115–43119 | 0x20003E23 |
| 4 | 43120–43124 | 0x20003E2D |

#### 16.3.4 Global Power Limits (43133–43134)

| Register | R/W | Description | Value range | SRAM | EEPROM |
|---|---|---|---|---|---|
| 43133 | R/W | Max. charge power (global) | 0–2500 W | `DAT_200004A0` | 0x800 |
| 43134 | R/W | Max. discharge power (global) | 0–2500 W | `DAT_200004A2` | 0x802 |

#### 16.3.5 Grid/Power Configuration (44000–44003)

| Register | R/W | Description | Value range | Scaling | SRAM | EEPROM |
|---|---|---|---|---|---|---|
| 44000 | R/W | Grid voltage threshold | 800–1000 (→ 80.0–100.0V) | ÷10 stored, ×10 read | `DAT_200004A9` | 0x201 |
| 44001 | R/W | Grid frequency threshold | 120–300 (→ 12.0–30.0) | ÷10 stored, ×10 read | `DAT_200004AA` | 0x201 |
| 44002 | R/W | Max. charge power (channel) | 0–2500 W (during calibration: max 800) | direct | `DAT_200004AB` | 0x202 |
| 44003 | R/W | Max. discharge power (channel) | 0–2500 W (during calibration: max 800) | direct | `DAT_200004AD` | 0x204 |

#### 16.3.6 Grid Standard (44100)

| Register | R/W | Description | Valid values |
|---|---|---|---|
| 44100 | **W** | Set grid standard | 0–4 (calls `Grid_Protection_SetLimits()`) |

Presumed assignment: 0=disabled, 1=VDE-AR-N 4105, 2=EN 50549, 3=CEI 0-21, 4=AS/NZS 4777

#### 16.3.7 Factory Mode & Test (45000–45019)

| Register | R/W | Description | Valid values |
|---|---|---|---|
| 45000 | **W** | Factory mode control | `0x55EE` = activate factory mode, `0x55FF` = normal mode, `0x55DD` = save settings |
| 45010–45018 | **W** | Schedule config block | Writes to SRAM `0x20003DE8 + (reg-45010)×2` |
| 45019 | **W** | Schedule config save | Trigger: saves the 45010 block to EEPROM |

#### 16.3.8 Hardware Test (45200–45300)

| Register | R/W | Description | Values |
|---|---|---|---|
| 45200 | **W** | Buzzer test | Any value → `buzzer_beep_short()` |
| 45300 | **W** | TIM1 PWM test | `0` = off (duty 0), `1` = 50% duty (TIM1 CCR1, ~13 kHz) |

> **Note on 45300:** TIM1 (0x40012C00) generates ~13 kHz PWM (prescaler 16, ARR 399). Usable only in factory/idle mode. The Venus D (passively cooled) has **no fan** — presumably auxiliary voltage PWM, gate driver supply, or an unused pin for other model variants.

#### 16.3.9 Factory Inverter Control (45400–45402)

Usable only in **factory mode** (after reg 45000 = 0x55EE):

| Register | R/W | Description | Values |
|---|---|---|---|
| 45400 | **W** | LLC start (factory) | `0` = stop, `1` = start mode 1 |
| 45401 | **W** | Inverter start (factory) | `0` = stop, `1` = start mode 2 |
| 45402 | **W** | Off-grid start (factory) | `0` = stop, `1` = start mode 6 |

#### 16.3.10 Inverter Operating Control (45500–45502)

Behavior **differs** between normal and factory mode:

| Register | R/W | Normal mode | Factory mode |
|---|---|---|---|
| 45500 | **W** | `0` = stop, `1` = +2000W feed-in | `0` = stop, `1` = restart mode 3 |
| 45501 | **W** | `0` = backup off + stop, `1` = backup on | `0` = stop, `1` = restart mode 4 |
| 45502 | **W** | `0` = stop, `1` = -2500W (charging) | `0` = stop, `1` = restart mode 5 |

**Internal behavior in normal mode:**
- Reg 45500 value 1: `set_inverter_power(+2000.0f)` → feed-in 2kW
- Reg 45502 value 1: `set_inverter_power(-2500.0f)` → charging 2.5kW

#### 16.3.11 Power Setpoints (45600–45601)

| Register | R/W | Description | Values |
|---|---|---|---|
| 45600 | **W** | Set charge power | `writeValue` → `set_inverter_power(-float(value))` (negative = charging) |
| 45601 | **W** | Set discharge power | `writeValue` → `set_inverter_power(+float(value))` (positive = feed-in) |

> Effective only in **normal mode**. In factory mode these registers are ignored.

#### 16.3.12 Calibration Readback (45603–45604)

| Register | R/W | Description | EEPROM | Format |
|---|---|---|---|---|
| 45603 | **R** | Calibration discharge energy | 0x950 | BCD (Wh) |
| 45604 | **R** | Calibration charge energy | 0x952 | BCD (Wh) |

#### 16.3.13 Special Commands (46000)

| Register | R/W | Description | Values |
|---|---|---|---|
| 46000 | **W** | System control | `0x5100` = DAT_20000499=1 (EEPROM 0x900 persist), `0x04D2` (1234) = reboot (inverter off + reset flags), `0x0929` (2345) = reboot + DAT_200040C0=2 |

### 16.4 EEPROM Addresses (Summary)

EEPROM write accesses extracted from `modbus_register_handler`:

| EEPROM address | Modbus register | Description |
|---|---|---|
| 0x090 | 41010 | Calibration status |
| 0x101 | 41100 | Config byte |
| 0x201 | 44000–44001 | Grid voltage/frequency thresholds |
| 0x202 | 44002 | Max. charge power (channel) |
| 0x204 | 44003 | Max. discharge power (channel) |
| 0x301 | 42000 | Work mode (persisted at 0x55BB) |
| 0x800 | 43133 | Max. charge power (global) |
| 0x802 | 43134 | Max. discharge power (global) |
| 0x900 | 46000 | Special command flag |
| 0x950 | 45603 | Calibration discharge energy (R/O) |
| 0x952 | 45604 | Calibration charge energy (R/O) |

### 16.5 Callees of `modbus_register_handler`

| Function | Calls | Role |
|---|---|---|
| `memcpy_reverse` | 16× | Big-endian ↔ little-endian conversion for Modbus wire format |
| `thunk_EXT_FUN_100009e8` | 18× | External function (flash 0x10000000+), presumably config update |
| `EEPROM_WriteVerify` | 14× | Persistent storage with verification |
| `thunk_EXT_FUN_10000160` (`set_inverter_power`) | 8× | Pass power setpoint to inverter control |
| `thunk_EXT_FUN_10000B58` (`disable_inverter`) | 5× | Shut down inverter |
| `Grid_Protection_SetLimits` | 1× | Set grid protection parameters according to standard |
| `EEPROM_ClearStats` | 1× | Clear statistics |

---

## 17. Error Code Bitmasks (err1 / err2 / war1)

### 17.1 SRAM Layout

| Variable | SRAM address | Telemetry offset | Description |
|---|---|---|---|
| `err1` | `0x200019F4` | +0x08 | Error Code 1 (uint32) |
| `err2` | `0x200019F8` | +0x0C | Error Code 2 (uint32) |
| `war1` | `0x200019FC` | +0x04 | Warning Code 1 (uint32) |

Telemetry block at `0x200038E8`, sent via CAN CMD 0x10.

### 17.2 Confirmed Bits (from M4 Firmware)

Only 4 bits are set directly in the analyzed binary. The remaining bits are managed by the **M0 firmware** (external flash 0x10000000+) via `thunk_EXT_FUN_10006874()`.

| Field | Bit | Hex mask | Event | Description (CN) | Meaning |
|---|---|---|---|---|---|
| err1 | 20 | 0x00100000 | 0x33 | CAN通信异常 | CAN communication fault |
| err1 | 21 | 0x00200000 | 0x3B | 电池上传故障 | BMS reports fault |
| err2 | 7 | 0x00000080 | 0x13 | 电池放电保护 | Battery discharge current protection |
| err2 | 11 | 0x00000800 | 0x35 | 开关机异常 | Power on/off anomaly |

### 17.3 Debug Log Event Codes (M4 → M0 Dispatch)

The following error conditions are logged via `debug_log_enqueue` and forwarded to the M0 FW via `thunk_EXT_FUN_10006874()`. The exact bit assignment resides in the M0 firmware.

| Event | Chinese | Meaning |
|---|---|---|
| 0x2A | 电网捕获频率异常 | Grid frequency capture fault |
| 0x2B | SPLL频率异常 | Software PLL frequency fault |
| 0x2C | 电网VRMS异常 | Grid VRMS abnormal |
| 0x2D | 电网MAX电压异常 | Grid maximum voltage fault |
| 0x2E | 电压波形异常 | Voltage waveform fault |
| 0x2F | 电池电压异常 | Battery voltage abnormal |
| 0x30 | 软件锁相环重启 | Software PLL restart |
| 0x31 | 过温异常 | Overtemperature |
| 0x32 | 电池异常 | Battery fault (general) |
| 0x34 | 高压母线电压波动 | HV bus voltage fluctuation |
| 0x36 | 输出电流保护 | Output current protection |
| 0x37 | 高压母线电压波动 | HV bus voltage fluctuation (2) |
| 0x3C | 电池电压异常 | Battery voltage → grid disconnection |
| 0x3D | 并网电流异常 | Grid feed-in current fault |
| 0x3E | 电网检测异常 | Grid detection abnormal / SOC |
| 0x3F | 过流异常 | Overcurrent |
| 0x40 | 过压异常 | Overvoltage |
| 0x41 | 漏电异常 | Residual current (leakage) |
| 0x42 | 电流偏置 | Current offset fault |

> **Note:** The complete bit→event assignment for err1/err2/war1 requires analysis of the M0 firmware (0x10000000+), which is not contained in the analyzed binary.

---

## 18. Grid Standard Codes (`grid_standand`, Register 44100)

### 18.1 Code Assignment

From `Grid_Protection_SetLimits` (0x080012E0). Stored in EEPROM 0x206, SRAM `DAT_200004AF`.

| Code | Grid standard | Anti-islanding | Peculiarity |
|---|---|---|---|
| **0** | **VDE-AR-N 4105** (Germany) | 2 s | Default, dynamic frequency thresholds |
| **1** | VDE-AR-N 4105 (alias) | 2 s | Identical to code 0 |
| **2** | **EN 50549 / CEI 0-21** (EU/Italy) | 60 s | Fixed 50Hz thresholds |
| **3** | **AS/NZS 4777.2** (Australia/NZ) | 300 s (5 min) | Strict UV threshold 69V |
| **4** | VDE-AR-N 4105 (alias) | 2 s | Identical to code 0 |

### 18.2 Protection Parameters Compared

| Parameter | Code 0/1/4 (DE) | Code 2 (EU/IT) | Code 3 (AU/NZ) |
|---|---|---|---|
| **Overvoltage stage 1** | 265.0 V / 55 s | 253.0 V / 55 s | 255.3 V / 60 s |
| **Overvoltage stage 2** | 287.5 V / 1 ms | 287.5 V / 1 ms | 264.5 V / 100 ms |
| **Undervoltage stage 1** | 103.0 V / 55 s | 103.0 V / 25 s | 69.0 V / 800 ms |
| **Undervoltage stage 2** | — / 1 ms | — / 250 ms | — / 100 ms |
| **Overfrequency 1** | nom + 1.8 Hz | 51.5 Hz | 51.5 Hz |
| **Underfrequency 1** | nom − 2.6 Hz | 47.5 Hz | 47.5 Hz |
| **Overfrequency 2** | nom + 0.8 Hz | 50.5 Hz | 50.5 Hz |
| **Underfrequency 2** | nom − 2.4 Hz | 47.55 Hz | 47.55 Hz |
| **Reconnect time** | 1.000 ms | 1.000 ms | — |
| **Anti-islanding** | 2.000 ms | 60.000 ms | 300.000 ms |
| **Ramp rate** | 0.5 | 0.5 | — |

> **All codes are 230V/50Hz-based** — there is no 120/240V 60Hz profile (USA/UL 1741) in this firmware. Codes 1 and 4 fall into the same else branch as code 0 and are functionally identical (reserved for future standards or regional variants).

---

## 19. Pack Scheduling (Control FW Analysis)

### 19.1 Control FW Status

The Control FW (`VNSD-0_app_1492_0702_142136.bin`, 385 KB, 1615 functions) is loaded in Ghidra and has been analyzed.

### 19.2 Result: No Pack Rotation in the Control FW

The Control FW contains **no pack selection logic** (which physical battery is active). It manages exclusively **time-based charge/discharge scheduling**:

**Time slot struct** (10 bytes × up to 10 slots, EEPROM 0x302):

| Offset | Field | Type | Description |
|---|---|---|---|
| +0x00 | `week_set` | u8 | Weekday bitmask (0x00–0x7F, Mon–Sun) |
| +0x01 | `enable` | u8 | 0/1 |
| +0x02 | `power` | i16 | Charge power (W), negative = charging |
| +0x04 | `start_time` | 2×u8 | Hour, minute |
| +0x06 | `end_time` | 2×u8 | Hour, minute |
| +0x08 | — | 2 | Padding |

**Work modes** (EEPROM 0x301):

| Value | Mode | Description |
|---|---|---|
| 0 | Self-Use | Self-consumption optimization |
| 1 | Manual/schedule | Time-slot controlled |
| 5 | Economy/Force Charge | Forced charging (SOC 50–51% hysteresis) |

**Modbus registers (Control FW TCP):**
- 41500–41515: time settings (u16 array)
- 41600–41631: power settings (u16 array)
- 43100–43129: slot structs directly (6 slots × 5 registers)

**Cloud TOU scheduling:**
- `HTTP_Economy_TOU_Parser` (0x080149F8) loads tariff schedules from `https://{env}.hamedata.com/external-services/api/v1/schedulings/policies`
- Up to 96 periods per day (6-byte struct: price, secondary value, type, flag)

> **Conclusion:** The pack rotation (which physical battery slot is active) is controlled on the **Micro MCU** — the 6 slots × 10 bytes at SRAM `0x20003E05` are a Micro MCU concept. The Control FW only knows schedules and power setpoints.
>
> **Correction 2026-09-24:** This conclusion is wrong. The 6 slots at `0x20003E05` are
> **schedule** slots, not battery slots. Which pack is active is decided by the **master BMS**
> (10 % steps, emptiest or fullest pack first), see
> `BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md`.

---

## 20. EEPROM Register Map (Complete)

### 20.1 Overview

Extracted from `EEPROM_LoadConfig` + all `EEPROM_WriteVerify` calls. I2C EEPROM (device addresses 0xA0/0xA2).

Address 0x000 contains a **config version flag**: `0x03` = valid configuration, `0x02` = statistics only, anything else → factory settings.

### 20.2 Complete Address Table

| EEPROM | Size | SRAM variable | Description | Access |
|---|---|---|---|---|
| `0x000` | 1 | Stack | **Config version** (0x03=valid, 0x02=partial) | LoadConfig (R/W), Inverter_SetMode (W) |
| `0x001` | 47 | `0x20003DD4` | **Main config block** (grid parameters, timers, scheduling) | LoadConfig (R/W) |
| `0x021` | 20 | `0x20003DE8` | **Schedule table** (10×2-byte TOU entries, part of 0x001) | modbus_register_handler (W) |
| `0x080` | 1 | Stack/Tick | **Heartbeat timestamp** (HAL_GetTick LSB) | EEPROM_SaveTimestamp (W) |
| `0x090` | 1 | `DAT_20000497` | **Calibration status** (0=normal/2500W, 1=calibrated/800W) | LoadConfig (R), modbus_handler (W) |
| `0x100` | 5 | `DAT_200004A4` | **Config block 1** (EMS presence + sub-fields) | LoadConfig (R/W), CAN_Filter_Setup (W) |
| `0x101` | 1 | `DAT_200004A5` | **Config byte** (sub-field) | modbus_handler (W) |
| `0x102` | 1 | `DAT_200004A6` | **Backup enable flag** (sub-field) | modbus_handler (W) |
| `0x150` | 1 | `DAT_20000496` | **CAN retry counter** (decremented on reconnect, reset at 0xFF) | LoadConfig (R), CAN_Filter_Setup (W) |
| `0x200` | 7 | `DAT_200004A9` | **Grid config block** (voltage/frequency + sub-fields) | LoadConfig (R/W) |
| `0x201` | 1 | `DAT_200004A9` | **Grid standard / voltage threshold** (scaled ×10) | modbus_handler (W) |
| `0x202` | 2 | `DAT_200004AB` | **Max. charge power** (W) | modbus_handler (W), CAN_Dispatcher (W) |
| `0x204` | 2 | `DAT_200004AD` | **Max. discharge power** (W) | modbus_handler (W), CAN_Dispatcher (W) |
| `0x206` | 1 | `DAT_200004AF` | **Grid standard** (0–4, → Grid_Protection_SetLimits) | modbus_handler (W) |
| `0x300` | 124 | `backup_mode_flag` | **Work mode block** (backup, mode, scheduling, timers) | LoadConfig (R/W), modbus_handler (W) |
| `0x301` | 1 | `inv_work_mode` | **Inverter work mode** (sub-field) | modbus_handler (W) |
| `0x500` | 48 | `DAT_20003E7F` | **Operating hours / statistics** (runtime, energy counters) | LoadConfig (R/W), ClearStats (W), `EEPROM_SaveOperatingStats` (W, tranche 4c) |
| `0x800` | 4 (R) / 2 (W) | `DAT_200004A0` | **Global max. charge power** | LoadConfig (R/W), modbus_handler (W) |
| `0x802` | 2 | `DAT_200004A2` | **Global max. discharge power** | modbus_handler (W) |
| `0x900` | 2 (R) / 1 (W) | `DAT_20000499` | **Special command flag** (1=triggered) | LoadConfig (R/W), modbus_handler (W) |
| `0x901` | 1 | Stack | **Command acknowledge** (boot: →0x00, telemetry: →0x01) | LoadConfig (W), build_telemetry (W) |
| `0x950` | 2 | `DAT_2000049C` | **Calibration of discharge energy** (Wh) | LoadConfig (R), modbus_handler (R) |
| `0x952` | 2 | `DAT_2000049E` | **Calibration of charge energy** (Wh) | LoadConfig (R), modbus_handler (R) |

### 20.3 Memory Regions

| Region | Range | Size | Content |
|---|---|---|---|
| Config version | `0x000` | 1 B | Boot validation flag |
| Main config | `0x001–0x02F` | 47 B | Grid parameters, scheduling (contains 0x021 sub-block) |
| Heartbeat | `0x080` | 1 B | Tick timestamp |
| Calibration | `0x090` | 1 B | Normal vs. calibrated power limit |
| Config block 1 | `0x100–0x104` | 5 B | EMS presence, config byte, backup enable |
| CAN counter | `0x150` | 1 B | Reconnect retry counter |
| Grid config | `0x200–0x206` | 7 B | Voltage/frequency thresholds, power limits, grid standard |
| Work mode | `0x300–0x37B` | 124 B | Backup flag, work mode, scheduling timers |
| Statistics | `0x500–0x52F` | 48 B | Operating hours, energy counters |
| Power limits | `0x800–0x803` | 4 B | Max. charge + discharge power (global) |
| Special command | `0x900–0x901` | 2 B | Command flag + acknowledge |
| Calibration energy | `0x950–0x953` | 4 B | Discharge + charge energy (Wh) |

> **22 discrete access addresses** across the range 0x000–0x953. Most of the gaps in between (0x030–0x07F, 0x105–0x14F, 0x207–0x2FF, 0x37C–0x4FF, 0x530–0x7FF, 0x804–0x8FF, 0x902–0x94F) are unused.

---

## 15. Open Questions

| # | Topic | Next step |
|---|---|---|
| ~~1~~ | ~~Telemetry offset 0x12~~ | ✅ = **grid_pf (0.1Hz, grid frequency)** — verified from Control FW |
| ~~2~~ | ~~Telemetry offset 0x16~~ | ✅ = **grid_permit (grid permission)** — verified from Control FW |
| ~~3~~ | ~~Telemetry → Modbus mapping~~ | ✅ Control FW `FUN_08035ffc` + SRAM `0x20014E90` fully decoded |
| ~~4~~ | ~~Unknown functions~~ | ✅ **392/445 functions named (88.1%)**, finally completed 2026-07-14 (corrected; the 398/445 originally stated here from the 07.07. mass analysis were never checked in, see section 14) — remaining 53 exhausted statically, see memory `project-micro-inverter-fw-naming-status` |
| ~~5~~ | ~~Fault code bitmasks~~ | ✅ **4 bits confirmed in M4 FW**, 19 event codes documented — section 17. Complete bit map requires M0 FW |
| ~~6~~ | ~~grid_standand values~~ | ✅ **3 standards decoded** (VDE 4105, EN 50549/CEI 0-21, AS/NZS 4777) — section 18 |
| 7 | **Test UART shell** | Verify empty password (115200 8N1). **Update 2026-07-15:** Version diff against VNS 115 shows that the complete shell (`shell_*`, login/password, >30 commands, task `vtask_shell`) only exists from VNS 116 onward — completely absent in 115. A test device with older FW would therefore have no shell to test at all. See section 21. |
| ~~8~~ | ~~Pack rotation normal mode~~ | ✅ Control FW has no pack selection. **Correction 2026-09-24:** The pack selection is done by the master BMS, not by the Micro MCU (`BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md`) |
| ~~9~~ | ~~Modbus RS485 register map~~ | ✅ **43 registers decoded** — section 16 (system control, schedule, factory, power setpoints, calibration) |
| ~~10~~ | ~~EEPROM register map~~ | ✅ **22 addresses, 0x000–0x953** — section 20 |

---

## 21. Version Diff: VNS 115 → VNS 116 (2026-07-15)

Ghidra Version Tracking diff between `Micro_VNS_115` (190 functions, of which only 13 named — never
analyzed on its own) and `Micro_VNS_116` = the status of this doc (452 functions, 392 named).
All available correlators applied (symbol name, exact bytes/instructions/mnemonics, duplicate
instructions, function/data reference, combined reference) — result stable across all rounds:

| | Count |
|---|---|
| Matched (identical) | 65 |
| Matched (changed, only new callees, 0 byte changes in the function body itself) | 65 |
| Only in 115 (no counterpart found in 116) | 53 |
| **Only in 116 (new compared to 115)** | **315** |

The 315 "new" functions went through all correlators (including call-graph-based ones) without
finding any correlation to 115 — a strong signal for **genuine functional growth**, not
just compiler/optimization noise. Confirmed by the string diff: **41 completely new string literals**
in 116 (among others BMS update messages, shell command names), only 15 "removed" strings — and those are
mostly identical format strings that merely gained a leading space (no
real function removal).

**Most important finding: VNS 115 is a significantly stripped-down early stage of the firmware**, not just a
point release. Entire subsystems are completely missing and were only built up by 116:

- **BMS firmware update via CAN/RS485 (completely new):** `BMS_FW_Update_CAN_Handler`, `CAN_Start_FW_Update`,
  `OTA_FW_Update_StateMachine`, Ymodem receive (`Ymodem_Receive_Packet`, `Ymodem_Parse_File_Header`,
  `Ymodem_Send_Byte`), flash write path (`FLASH_EraseSectors`, `FLASH_WriteData`, `FLASH_WaitForOperation`).
  New strings confirm exactly the mechanism described in section 9: "Start to upgrade BMS with CAN",
  "Start to upgrade BMS with 485", "BMS upgrade successful/failed, system will reboot!". **The complete firmware update
  mechanism documented in section 9 therefore only exists from 116 onward** — 115 apparently cannot yet
  flash the BMS itself.
- **Modbus RTU interface (completely new):** `CRC16_Modbus`, `Modbus_Process_Request`,
  `Modbus_Read_Holding_Registers`, `Modbus_Write_Single_Register`, `Modbus_Write_Multiple_Registers`
  (+ broadcast variants), `modbus_register_handler`, dedicated task `vtask_modbus`. The 43-register map documented in
  section 16 is thus also only addressable from 116 onward.
- **Password-protected shell/CLI (completely new):** complete `shell_*` namespace (login, password check,
  history, tab complete, parser, >30 commands as strings: `update`, `update_bms`, `reset_memory`,
  `bat_mode`, `set_power`, `io_show`/`io_set`, `dac_high`/`dac_low`, `rtos_status`, `version`, and many more),
  dedicated task `vtask_shell`. Directly relevant for open question #7 above.
- **CAN ecosystem greatly expanded:** `BMS_CAN_Parser`, `EMS_Inverter_CAN_Dispatcher`, complete
  TX queue/dispatch chain (`CAN_TX_ReadQueue/SendMessage/SendCommand/SendFrame/ProcessQueue`,
  `CAN_BuildArbID`), `CAN_RX_DispatchTask`, `CAN_Filter_Setup`, dedicated tasks `vtask_can`/`vtask_can_receive`
  as well as the complete `HAL_CAN_*` driver (Init/DeInit/AddTxMessage/GetRxMessage/IRQHandler/RxFifoCallback).
- **Hardware driver expansion:** HRTIM (master + timer unit base/waveform/output/chopper config, for the
  PWM generation of the inverter bridge), full TIM1/4/7/20 configuration (OC1–6, IC1–4), DMA layer,
  extended ADC (calibration, DMA start, `ADC_ProcessSamples`), I2C master layer, `DAC_Init`,
  buzzer control, EEPROM subsystem (Load/Write/Read/SaveTimestamp/ClearStats/SaveOperatingStats), IWDG
  (watchdog), 2-channel NTC temperature conversion.
- **Control/regulation logic extended:** `Inverter_SetMode`, `Inverter_Grid_Control`,
  `Grid_Protection_SetLimits`, `build_telemetry_block`, `Mode3_RoundRobin_Timer`.

**Cross-check with the naming campaign:** A large part of the 315 new functions is already named (those
listed above); the `FUN_*` addresses among them that remain unnamed coincide with the 53 remaining functions classified
since 2026-07-10 as "statically exhausted" (dead-code family, HRTIM details without a
reference manual, interworking veneers, padding) — consistent, no new action required.

**Subsequently fully analyzed (2026-07-15, at the user's request, for documentation purposes):** All 53
only-in-115 functions were checked individually via `get-decompilation`/`find-cross-references` (2 parallel
subagents, one half each). Result:

| Category | Count | Description |
|---|---|---|
| Thunk/veneer | 13 | pure jump trampolines, including 4× `thunk_EXT_FUN_*` |
| Genuine function, meaningful purpose recognizable | ~30 | see name proposals below |
| Ghidra artifact (overlapping jump target fragment of a neighboring function) | ~8 | not a standalone function |
| Trivial stub | 2 | 2-byte infinite loop (halt trap) and dead 2-byte thunk respectively |

**Confirmed pattern — CCM SRAM trampolines:** The 4 `thunk_EXT_FUN_*` (0x08000964, 0x08000982, 0x080009aa,
0x080009be) all jump to `0x1000xxxx` — outside the loaded flash image, into the STM32F3 CCM SRAM.
Exactly the same pattern as the RAM-resident flash self-programming routines already known from the Control FW and from VNS 116 itself
(they must not run from the flash bank that is just being erased). Main caller is `FUN_0800d83c` (fault code derivation/control step) — fits
the OTA/flash context in terms of content.

**Most important single finding: a completely orphaned "legacy" control subsystem block.** Five contiguous
functions — `FUN_08017000` (302 B), `FUN_08017800` (2480 B, largest function in the entire 53-function set),
`0x0801aec0`, `0x0801b000`, `0x0801b030` — together form a complete old battery charge current limit
control loop (temperature/voltage curves, alarm flags, state init). This group has **0 callers in the
entire 115 image**, not even via function pointers/constants (`find-constant-uses` checked) — so it was
already **dead code within 115 itself**, not removed only at the transition to 116. In 116
there is no counterpart for it anymore — consistent with a replacement of this control logic by the current
inverter control documented in section 8, begun before 115 but never
completed. Noteworthy: the helper function `0x0800eb8c` used by this cluster
(sleep watchdog/CAN filter reinit) is itself NOT dead — it is additionally called by active code outside
the 53 list (0x08018558, 0x0801873a) and used in a shared manner.

**Second finding: duplicate debug dump infrastructure.** `0x0800cc56` and `0x0800d14c` are two almost
identical diagnostic dump routines (ctl_state/llc_run_state/inverter_run_state, grid/battery values), both
reachable only via dead/isolated thunks — presumably a predecessor of the productive
debug output active in 116.

**Complete name proposals (not applied in Ghidra, since 115 is not an actively maintained project):**

| Address | Proposal | Category |
|---|---|---|
| 0x08000368 | `Mem_Fill_Bytes` | genuine |
| 0x08002a04 | `ParamBlock_Mode_Reset` | genuine |
| 0x08002e0c | `Apply_ParamBlock_Cmd0x500` | genuine |
| 0x080032ac | `Flash_CR_WaitBusy_And_Lock` | genuine |
| 0x0800555e | `EXTI_Fault_LogAndDisableIRQ` | genuine |
| 0x0800675c | `NVIC_EnableIRQ` | genuine (CMSIS standard) |
| 0x08006778 | `NVIC_SetPriority` | genuine (CMSIS standard) |
| 0x080068dc | `Peripheral_Clock_Enable_WaitReady` | genuine |
| 0x0800699c | `Timer_ChannelConfig_ApplyFromStruct` (uncertain) | genuine |
| 0x08007084 | `HRTIM_Channel_Enable_Callback` | genuine, reached via function pointer table |
| 0x080096b8 | `UART_ErrorFlag_WaitAndRecover` | genuine |
| 0x08009774 | `UART_ErrorFlag_Variant_A` | genuine, reached via vtable |
| 0x08009792 | `UART_ErrorFlag_Variant_B` | genuine, reached via vtable |
| 0x0800bf8c | `Channel_State_Transition_1to2_Apply` | genuine |
| 0x0800cc56 | `Debug_PrintSystemStatus_Dump` | genuine, isolated/dead |
| 0x0800d000 | `Spinlock_AcquireExclusive_SetBusyFlag` | genuine, isolated/dead |
| 0x0800d14c | `Debug_PrintSystemStatus_Dump_Variant` | genuine, isolated/dead |
| 0x0800d83c | `InverterControl_FaultCode_And_ChargeGate_Update` | genuine, isolated/dead |
| 0x0800df00 | `Printf_DoubleToDecimalDigits` | genuine |
| 0x0800e084 | `Printf_FormatString_Core` | genuine |
| 0x0800eb00 | `Fault_UpdateIndicatorAndTimingState` | genuine, only via legacy cluster |
| 0x0800eb8c | `PowerMode_SleepWatchdog_And_CanFilterReinit` | genuine, **actively used** (not part of the dead cluster) |
| 0x0800f8d8 | `CLI_ParseFloatLiteral` | genuine, isolated/dead |
| 0x0800f944 | `Fraction_ScaleAndSign_Normalize` (uncertain) | genuine, isolated/dead |
| 0x0800fff6 | `Output_SendCRLF_ViaStreamHandle` | genuine, isolated/dead |
| 0x08017000 | `Legacy_ChargeVoltageFault_StateMachine` | genuine, **dead legacy cluster** |
| 0x08017800 | `Legacy_BatteryChargeCurve_ControlLoop` | genuine, **dead legacy cluster** |
| 0x0801aec0 | `AlarmFlags_EvaluateAndClear` | genuine, **dead legacy cluster** |
| 0x0801b000 | `Legacy_ThresholdCheck_TriggerFault8` | genuine, **dead legacy cluster** |
| 0x0801b030 | `Legacy_ChargeController_State_Init` | genuine, **dead legacy cluster** |

**Ghidra artifacts (overlapping fragments, not standalone functions — analogous to the `0x08008000` pattern already known from the
Control FW):** `0x080041f4`/`0x0800424c`/`0x080042b0`/`0x080042ca` (four
fragments of ONE larger, itself unreferenced EXTI init routine — STM32F3 layout confirmed via
IMR2/EMR2/RTSR2/FTSR2), `0x0800676c` (duplicate tail of `NVIC_EnableIRQ`), `0x08006980` (duplicate beginning
of `0x0800699c`), `0x0800708c` (duplicate body of `HRTIM_Channel_Enable_Callback`), `0x0800ebdc`
(duplicate tail of `PowerMode_SleepWatchdog_And_CanFilterReinit`), `0x0801af00` (outlined shared
tail block of `AlarmFlags_EvaluateAndClear`, reached only via conditional jump, never via call).

**Conclusion:** Of the 53 only-in-115 functions, most are either (a) already dead/
orphaned legacy control code in 115 itself (the complete battery charge curve cluster), (b) Ghidra duplicate artifacts without
an existence of their own, or (c) low-level helper routines (NVIC/EXTI/debug print/UART error handling) that in 116
were presumably simply restructured/replaced (the corresponding topic areas still exist in 116,
only with other functions, already covered in section 21 above). There is **no indication
of actually lost functionality** between 115 and 116 — on the contrary, 116 is an expansion in every respect.

---

*Created via Ghidra + ReVa MCP — static analysis without a live device*
