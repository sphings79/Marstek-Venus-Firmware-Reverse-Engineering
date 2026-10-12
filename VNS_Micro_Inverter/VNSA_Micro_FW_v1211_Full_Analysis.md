# Micro-FW v1211 (Venus A) — Full analysis

> 🇩🇪 Deutsche Version: [VNSA_Micro_FW_v1211_Vollanalyse_de.md](VNSA_Micro_FW_v1211_Vollanalyse_de.md)

**Status:** 2026-10-02; §4.1.6 (CAN receive filters and update start conditions) added 2026-10-12.
**Image:** `1211_micro_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin` (Venus A inverter board, "VA_inv_app").
**Decompilates/Listing:** `Dekompilate/VNSA_Micro_1211/` (local only, not in the repo): 409 functions, all named
and typed, `listing/` with ITCM + flash raw disassembly and `globals_and_registers.txt` (1253 named RAM/peripheral symbols).
**Related:** [VNSA_Error_Codes_Micro_FW_Analysis.md](VNSA_Error_Codes_Micro_FW_Analysis.md) (earlier analysis on a wrong
image base, corrected in §6 below), [VNSD_Micro_FW_v116_Full_Analysis.md](VNSD_Micro_FW_v116_Full_Analysis.md) (Venus D
inverter), `../Control_FW/VNSA_Control_FW_v1509_Full_Analysis.md`, `../BMS/VNSA_BMS_FW_v1105_Full_Analysis.md`.

## 0. Summary

- **Different MCU and different code base from the Venus D inverter.** Cortex-M7 (I/D cache, ITCM/DTCM) of the
  **Novosense NS800RT family** (C2000-style peripherals ePWM/eCAP/ADC A-D/CMPSS plus NXP-style FlexCAN, eDMA, DMAMUX,
  LPUART and a math accelerator). The memory map is identical to the NS800RT7 linker file (ITCM 0x00000000, DTCM
  0x20000000, SRAM1 0x20100000, SRAM2 0x20120000); the peripheral base addresses of RCC/eDMA/GPIO differ from the
  public NS800RT7 header, so the exact member (single-core NS800RT5xxx-class part) is not identified. 260 MHz core clock.
- **Bare metal, no RTOS.** `main` (0x0801e540) runs a super loop (letter-shell 3.2.1, CAN service, application task,
  watchdog). The power-electronics control runs in interrupts: `Ctrl_MainControlIsr` (ITCM) at **28 kHz** after every
  ADC DMA block, `Ctrl_SlowControlTask` (ITCM, 1 kHz BTIM1 interrupt) which first calls the 231-state
  `Inv_StateMachine` and then the MPPT state machine.
- **The A inverter board contains the whole power path:** bidirectional single-phase inverter (grid-tied + off-grid /
  backup output, SOGI-PLL, QPR current loop, P-f/Q-V droop for off-grid), a PFC/DC bus (400 V class, trip 530 V),
  an LLC DC/DC stage (67 kHz, synchronous rectification) to the ~48 V battery, and **four PV inputs (MPPT, buck/boost
  per channel, EPWM4/6/7/8, 200 kHz)**. On the Venus D the PV converter is a separate CAN node ("class 2"); on the
  Venus A this micro **is** the class-2 node and answers the Control's MPPT commands itself.
- **No Modbus/RS485 register handler** (the D micro has `Modbus_HandleRegisterAccess`). All configuration arrives as
  CAN commands; the only UART is the debug shell (115200 Bd), which can also be tunnelled over CAN (func 0xC2/0xC3).
- **Power limits:** two EEPROM limits (0x203 charge, 0x205 discharge, default **1450 W**), CAN class-4 func 0x04/0x03
  accept only values **< 1451 W** (larger values from the Control — which allows up to 2500/1500 W — are silently dropped,
  although the frame is still acknowledged). Feed-in maximum 1450 W, linearly derated above 73 °C radiator temperature. Off-grid output rated
  1500 W (6.54 A at 230 V i²t integrator). MPPT total power clamp 2400 W.
- **0x03410110 (BMS "PV rescue")** is not a dedicated frame: it is **frame index 3 of the micro's telemetry reply
  (class-4 func 0x10)**, which carries payload bytes 16..23. The BMS tests data byte 1 = high byte of the grid voltage
  in 0.1 V → `> 8` means grid voltage ≥ 230.4 V. The A micro never sends id 0x00210100 (the BMS "lock" id) and ignores
  0x100101AA.
- Firmware quirks found (§10): under-frequency trip without delay (inverted compare), reply to func 0x54 sends an
  uninitialised stack byte, the shell help text for `set_mode` does not match the code, the "keep counters" EEPROM reset
  also clears the counter block, EEPROM self-test (CAN test command) overwrites EEPROM 0x100/0x101.
- Manufacturer changelog for 1211: "增强VA离网口带RCD载能力" = enhanced the off-grid port's ability to drive loads behind
  an RCD (residual-current device).

### Answers to the Venus A register-resolution questions

| # | Question | Answer (details) |
|---|---|---|
| 1 | CAN receive path, accepted commands, telemetry | FlexCAN RX MB16..31 → queue (50) → `CAN_RxDispatch`. Accepted: class 2 func 0x01/0x02/0x03/0x04/0x06/0xCB/0xCC/0xFE/0xFF; class 4 func 0x01-0x07, 0x10-0x13, 0x16, 0x50-0x58, 0x60, 0xC1-0xC3, 0xCB; 0xCE OTA (no class/node check in software, but the FlexCAN RX masks only pass id byte1 0x40/0x41/0x21, §4.1.6); BMS frames 0x18n_AA01. Ignored: class-4 0x08, 0x0A, 0x14, 0x15, 0x17, **0x18, 0x30-0x3E, 0x40-0x43**; class-2 0x05, **0xFB**. Telemetry: func 0x10 (48 B, layout §4.1.4), 0x11 (8 B status), 0x12 (6 B ATE-aging state/results = "get old"), 0x13 (4 B stored aging results), 0x54 (1 B, garbage), 0xCB (8 B versions). **No 0x40-0x43 and no 0x14 frames are sent** (§4.1). |
| 2 | BMS rescue handshake | 0x03410110 = frame 3 of the func-0x10 telemetry reply, data[1] = grid voltage high byte; sent only when the Control polls func 0x10. No reaction to 0x100101AA, never sends 0x00210100 (§4.2). |
| 3 | Power class / limits / modes / register handler | No power-class logic; fixed 1450 W limits (EEPROM 0x203/0x205), CAN 0x03/0x04 accept < 1451 W. Work modes 0 default, 1 manual, 2 debug, 3 aging, 4 ATE aging, 12 CAN test mode, 13 voltage source (EEPROM 0x14). Backup = flag (CAN class-4 func 0x02 in mode 0 / func 0x57 in mode 1, EEPROM 0x11); off-grid output via state 30 / standard state 200. No Modbus register handler (§5). |
| 4 | Error/warning words | err1 bits 0-4, 7, 20, 21; err2 bits 5, 6, 7, 8, 9, 11, 13; warn never set (§6). |
| 5 | MPPT / PV | Yes: 4 PV inputs on this board (≈16-63 V each, 20 A), class-2 commands = the Control's `mppt*` registers address this MPPT stage (§5.4). |

## 1. Binary fingerprint

| Feature | Value |
|---|---|
| File | `1211_micro_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin` |
| Size | 146,432 bytes (0x23C00) |
| SHA-256 | `277611ea8982adad4d9ac8e1a1154a9cdea6c9f717b1f60743be640f0023723d` |
| Link address | `0x08006000`–`0x08029BFF` (24 KB bootloader below; reset handler bytes `09 48 80 47 09 48 00 47` at file offset 0x434) |
| MCU | Cortex-M7, Novosense NS800RT family (see §0), 260 MHz (ePWM time base) |
| Toolchain | Keil MDK, armclang (LLVM): movw/movt constants, `mov pc,rX` branch tables, ARM C library (armlib) |
| Libraries | letter-shell 3.2.1 (build "May 20 2026 18:12:41"), MCUXpresso-style FlexCAN/eDMA drivers |
| Vector table | 184 entries at 0x08006000; initial SP 0x20002400; Reset_Handler 0x08006434 → SystemInit 0x0801d278 → __main 0x080062e0 |
| Version | **1211 (0x4BB)** at `0x08010000`; also written to EEPROM 0x03 and reported via CAN 0xCB |
| Trailer | device tag `'VNSA'` at 0x08029BE0, checksum word at 0x08029BFC |
| Shell user | "letter V1211" with **empty password** |

**Scatter loading** (Region$$Table at 0x080248FC, 7 entries):

| Source (flash) | Target | Length | Handler |
|---|---|---|---|
| 0x08024DC8 | 0x00000000 (ITCM code) | 0x4160 | `__scatterload_copy` |
| 0x08028F28 | 0x20000000 (DTCM, RAM vector table) | 0x2E0 | `__decompress` (all zero) |
| 0x08028F30 | 0x20100000 (SRAM1 .data) | 0x118 | `__scatterload_copy` |
| 0x08029048 | 0x20120000 (SRAM2 .data) | 0x2A0 | `__decompress` |
| – | 0x20000400 / 0x20100118 / 0x201202A0 (.bss) | 0x2000 / 0x3588 / 0x33F8 | `__scatterload_zeroinit` |

Stack 0x20000400–0x200023FF (DTCM), heap 0x201016A0–0x2010369F (`__user_initial_stackheap`).

## 2. Memory map and analysis status in Ghidra

Program `/VNSA/Micro/Micro_1211_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin`:

| Block | Range | Content |
|---|---|---|
| `.itcm_code` | 0x00000000–0x0000415F | time-critical code (ISR, slow control task, delay loops, veneers 0x40DC..) |
| `flash` | 0x08006000–0x08029BFF | code 0x08006000–0x080211B9, then rodata/strings, Region$$Table, shell table, ITCM load image, .data images, 0xFF fill, trailer |
| `.data_20000000` | 0x20000000–0x200002DF | `g_ramVectorTable` (184 pointers) |
| `.bss_20000400` | 0x20000400–0x200023FF | stack |
| `.data/.bss SRAM1` | 0x20100000–0x2010369F | measurement, control, protection, energy counters, heap |
| `.data/.bss SRAM2` | 0x20120000–0x20123697 | CAN queues, MPPT, state machine, log ring, shell |
| `periph` | 0x40000000–0x400FFFFF | 509 register labels (NS800RT7 register layouts where they match the code: EPWM1-8, ECAP1, ADCA-D, ADCx RESULT, IWDG1, ANASS, PWR, LPUART "UART1", BTIM1/2, CAN1, CRC, RCC, FLASHC, MMATH; own layouts for GPIOA/B, CMPSS1-7, EDMA1/2, DMAMUX1/2) |
| `ppb` | 0xE0000000–0xE00FFFFF | Cortex-M7 core registers (SCB, NVIC, SysTick, cache maintenance) |

Verification gates (`vna_mic_gates.py`, all passed on the final state): 0 undefined bytes and 0 orphan instructions in
flash and ITCM, 0 flow into data, 0 literal words decoded as code, 0 calls into function bodies, 0 function pointers to
non-code, 0 `FUN_`/duplicate/temporary names, **0 unnamed referenced RAM/peripheral addresses**, 0 default prototypes,
runtime IRQ handlers named, no-return flags sane. Tags: `NAME_VERIFIED` on all 409 functions, `NAME_LOW_CONFIDENCE` on 27.
Decompilation export: 0 failures, no unrecovered jump tables. Note: the ITCM function `Time_DelayUs` lives at address 0,
so NULL-pointer comparisons in the decompiled code appear as comparisons with `Time_DelayUs`.

## 3. Execution model

| Context | Rate | Function |
|---|---|---|
| super loop | free running | `main` → `shellTask`, `Shell_FlushOutputToCan`, `App_MainTask` (warning word, `Mode_UpdateStatusAndAging`, `CAN_CommTask`, OTA supervision, LED GPIOA23 500 ms, counters to EEPROM every 10 min), IWDG1 refresh |
| DMA1 IRQ (slot 38) | 28 kHz | `DMA1_IRQHandler` → `Ctrl_MainControlIsr` (ITCM): ADC scaling, `Ctrl_InverterControlStep` (inverter/PFC/LLC loops), PV/battery limits and CMPSS latches → MPPT fault word, four MPPT converter updates |
| BTIM1 IRQ (slot 113) | 1 kHz | `BTIM1_IRQHandler` → `Ctrl_SlowControlTask` (ITCM): `Inv_StateMachine`, MPPT state machine/tracking, derating, PV energy |
| BTIM2 IRQ (slot 114) | 1 kHz | ms / seconds counters |
| CAN1 IRQ (slot 111) | event | `CAN1_IRQHandler` → RX queue |
| EPWM1 IRQ (slot 80) | 67 kHz event | LLC register update (`Llc_UpdatePwmRegisters`) |
| ECAP1 IRQ (slot 88) | grid period | 4 capture values → grid frequency |
| ADCB/ADCC IRQ (slots 62/67) | – | flag clear only |
| IWDG1 IRQ (slot 1) | early warning | all PWM off, 30 ms, reset |

The RAM vector table (`IRQ_InitRamVectorTable`) is filled with `IRQ_DefaultHandler`; each driver installs its ISR with
`*(SCB_VTOR + 4*(16+irq)) = handler`.

## 4. Interfaces

### 4.1 CAN (500 kbit/s, 29-bit ids)

Driver: FlexCAN at 0x40074000 (8 MHz clock), handle 0x201204A0, MB0..15 TX round robin, MB16..31 RX with individual
acceptance masks (§4.1.6). RX ISR → queue
`g_canRxQueue` (0x20120518, 50 × 16 B: id, data[8], dlc); TX queue 0x2012084C (300 × 16 B), one frame every 10 ms
(`CAN_CommTask` → `CAN_TransmitFrame`).

Id layout (identical to the Control's `CAN_Build_Arbitration_ID`): byte0 = function code, byte1 = class<<4 | node,
byte2 = source (Control = 0x01), byte3 bits 0-4 = frame index. The micro replies to class 4 with fixed id
`idx<<24 | 0x410100 | func` and to class 2 with source/destination bytes swapped (`0x..2101xx`).

#### 4.1.1 Class 2 (Control "MPPT"/register class) — `CAN_RxDispatch`

| Func | Payload | Action |
|---|---|---|
| 0x01 | frame 0, data[4] | 0 → `ShellCmd_stop` (MPPT closed loop off), 1 → `ShellCmd_start`; empty ack |
| 0x02 | frame 1: u16 mode, u16 power W, u16 voltage 0.1 V | mode 1 = no power distribution between channels, 3 = aging (stop + restart); power → `ShellCmd_mppt_pow` (clamp 2400 W, floor 50 W), voltage → `ShellCmd_mppt_vol` (44.2–47.5 V, EEPROM 0x541); power 0 with voltage ≠ 0 = power stop; ack |
| 0x03 | – | `Mppt_BuildStatusReport`, 8 reply frames (PV V/I/P per channel, battery V/I, state, power reference) |
| 0x04 | – | reboot request (`g_canRebootRequest`) |
| 0x06 | – | MPPT stop + clear energy counter levels 0..4 |
| 0xCB | – | reply 0x04BB0065 (version 1211) + version word |
| 0xCC | 7 bytes | date/time → `Meas_RollEnergyCountersOnDateChange` |
| 0xFE | – | PE-offset calibration (`Mppt_CalibratePeOffset`), reply value ×0.000791 (five frames) |
| 0xFF | – | reply four PV current offsets and battery current offset |
| other (0x05, 0xFB …) | – | ignored |

These are exactly the commands the Control sends in `Inverter_Set_OnOff_Reg_0x01`, `Inverter_CAN_PowerCmd_Send`
(g_mpptMode/g_mpptPowerW/g_mpptVoltage_0p1V), `Inverter_Set_Reg_0xFE/0xFF`, `Inverter_Set_WorkMode_Reg_0x06` etc.
— on the Venus A they reach the MPPT stage of this micro. Replies go to the Control's `Modbus_ResponseDispatch`
(0x03, 0xCB, 0xCE, 0xFE, 0xFF).

#### 4.1.2 Class 4 (inverter class)

| Func | Action on the A micro | Reply |
|---|---|---|
| 0x01 | i32 power setpoint W → `Inv_SetPowerSetpoint` (only in work mode 0) | 4 B (value) |
| 0x02 | data[0] < 2 → `Mode_SetBackupMode` (backup flag, EEPROM 0x11; only in work mode 0) — Control: `Config_Mode_Apply` | ack |
| 0x03 | signed i32 < 1451 → `g_powerLimitDischargeW`, EEPROM 0x205 (Control: max discharge power, allows ≤ 1500); larger values are dropped, negative values are not rejected | ack (also when dropped) |
| 0x04 | signed i32 < 1451 → `g_powerLimitChargeW`, EEPROM 0x203 (Control: max charge power, allows ≤ 2500); larger values are dropped | ack (also when dropped) |
| 0x05 | reboot (counters saved, PWM off, reset) | ack |
| 0x06 | `g_cfgInverterDisable` = 1 (start inhibit, telemetry inv_state 0) | ack |
| 0x07 | parameter reset type data[0] → `Param_ResetToDefaultsAndReboot` | ack |
| 0x10 | – | 48-byte telemetry (6 frames, §4.1.4) |
| 0x11 | – | 8 B: work mode, inverter-disable flag, battery mode, 0, charge limit, discharge limit (Control `g_vnsInv*`: work_mode, sleep_flag, bat_mode, hardware, charge/discharge_limit_power) |
| 0x12 | – | 6 B: `g_ateAgingState`, ATE aging results A/B ("get old" = 老化 aging; Control `g_oldStep` → regs 37001/37002) |
| 0x13 | – | 4 B: stored aging results (EEPROM 0x302/0x306) → Control get-old queues |
| 0x16 | 8 B total charge/discharge energy (Wh) from the Control → `g_energyChargeWhRestore`/`...Discharge...` (lower bound for the counters) | – |
| 0x50 | data 1 → work mode 12 (test mode), GPIO/CMPSS latch reset, test command 7; data 0 → latch reset only | – |
| 0x51/0x53 | – | ack |
| 0x52 | mode 12: test command data+1 (1..3) | ack |
| 0x54 | – | 1 B (uninitialised stack byte, see §10) |
| 0x55 | mode 12: test command 6 | ack |
| 0x56 | mode 12: power reference ±1449 W (`Ctrl` power loop) | ack |
| 0x57 | backup flag = (work mode 1 && data == 1) | ack |
| 0x58 | mode 1: power setpoint ±2499 W (still clamped to the 1450 W limit) | ack |
| 0x60 | work mode 4 (ATE aging), `g_ateAgingState` = data | ack |
| 0xC1 | test command 1 (EEPROM self-test) | – |
| 0xC2 | data 1 → shell over CAN on (ack), else off (no ack) | ack / – |
| 0xC3 | shell input bytes (when shell over CAN is on); output is sent as id 0x004101C3 | – |
| 0xCB | – | 8 B: version word (1211), boot version, 1 |
| 0x08, 0x0A, 0x14, 0x15, 0x17, 0x18, 0x30–0x3E, 0x40–0x43, 0x59–0x5F, 0x61–0xC0 … | **ignored, no reply** | – |

#### 4.1.3 Frames sent by the micro

Only replies (no periodic broadcast): 0x10, 0x11, 0x12, 0x13, 0x54, 0xCB and acks (class 4, id 0x410100|func),
class-2 replies (0x2101xx), 0xC3 shell output, 0xCE OTA acks. **The Control's energy registers fed by ids 0x40–0x43
and 0x14 (`Telemetry_Store_EnergyCounters`, `Protocol_AA_SetExtendedParams`) are never written on a Venus A.**

#### 4.1.4 Telemetry payload (func 0x10, `g_emsTelemetryBlock` 0x2012046C, built by `CAN_UpdateTelemetryPayload` every 400 ms)

Frames with index 1..6 carry 8 bytes each; the Control copies them 1:1 into its struct `g_invState` (A Control 1509
0x2001769C).

| Offset | Size | Source in the micro | Control field / Modbus |
|---|---|---|---|
| 0x00 | u8 | `g_invTelemetryState` (0 disabled, 1 idle, 2 charging < −10 W, 3 discharging > 10 W, 4 backup/parallel, 6 both relays closed) | inv_state 35100 |
| 0x01 | u8 | 0 | buz_state |
| 0x02 | u8 | `g_batDischargePermit` | chrg_flag |
| 0x03 | u8 | `g_cfgBackupModeEnabled` | back_func |
| 0x04 | u32 | `g_protWarningWord1` | warn_code 36000 |
| 0x08 | u32 | `g_protErrorWord1` | error_code1 36100 |
| 0x0C | u32 | `g_protErrorWord2` | error_code2 36102 |
| 0x10 | u16 | grid voltage RMS ×10 | grid_volt (0.1 V) 30004 |
| 0x12 | u16 | grid frequency ×10 | grid freq (0.1 Hz) |
| 0x14 | u16 | off-grid voltage ×10 | off_grid_volt 30005 |
| 0x16 | u16 | `g_gridPermit` | grid permit |
| 0x18 | i16 | grid power W | grid_sample_power |
| 0x1A | i16 | off-grid power W | off_grid_power |
| 0x1C | i16 | battery power W | bat_sample_power |
| 0x1E | u16 | battery voltage ×10 | bat_sample_volt 30000 |
| 0x20 | i16 | ambient temperature ×10 | env_temp |
| 0x22 | i16 | radiator temperature ×10 | radiator_temp |
| 0x24 | i16 | `g_invPowerLimitMaxW` | max_power |
| 0x26 | i16 | `g_invPowerSetpointMinW` (negative) | min_power |
| 0x28 | u32 | charge energy Wh | chrg_energy |
| 0x2C | u32 | discharge energy Wh | dischrg_energy |

#### 4.1.5 OTA (func 0xCE)

Frame index 0: start (image length, CRC16) → erase 232 pages from 0x08046000; index 1: 8 data bytes into a 1 KB buffer;
index 2: block end with CRC16 (CRC unit) → program 1 KB at 0x08046000 + block × 0x400, reply status 1/2/3. After the last
block the CRC over the whole image is checked, `OTA_WriteBootDescriptor` writes {0x08046000, length, 1, CRC} to
0x08040000 and `App_MainTask` resets (`SCB_SystemResetOtaApply`) about 3 s after the last 0xCE frame. Without a verified
image the session is aborted after about 3 s without any 0xCE frame (not 60 s as stated in an earlier version of this
document); details and receive filters in §4.1.6.

#### 4.1.6 Which 0xCE frames reach the micro (receive filters, update start conditions)

Question: can a CAN update meant for another node (BMS packs, other micros, MPPT) start the update routine of this
micro? **Answer: no for BMS packs and for any node address other than class 4 node 0/1 and class 2 node 1 — those
frames are rejected by the CAN controller before software sees them. PROVEN from code.** The software path itself has
no address check, so everything that does pass the filters is treated as an update for this micro.

**CAN controller.** The register layout used by the driver is NXP FlexCAN: MCR at +0x00 (FRZ/HALT/SOFTRST/MAXMB,
IRMQ = bit 16, SRX_DIS = bit 17), CTRL1 +0x04, RXMGMASK/RX14MASK/RX15MASK +0x10/+0x14/+0x18, IMASK1 +0x28, IFLAG1 +0x30,
CTRL2 +0x34, message buffers from +0x80, individual masks RXIMR from +0x880. The NS800RT therefore carries a
FlexCAN-compatible controller (consistent with §1).

**Filter set-up (`CAN_Init` 0x080088c0, identical re-init in `CAN1_IRQHandler`).** `FLEXCAN_Init` runs with
individual RX masking enabled (config byte +0x248 = 1 → MCR.IRMQ = 1; self reception disabled) and 16 RX
message-buffer descriptors from `g_canFlexcanRxMbConfigArray` (0x20120010, 16 × 24 B, initial values from the
scatter-loaded `.data`). `FLEXCAN_SetRxMbConfig` writes id and IDE into each MB, sets it to RX-empty and writes the mask
into RXIMR[mb]. RX mailboxes 0..15 are not configured for reception (they are the TX round robin; `CAN_TransmitFrame`
only touches MB 0..15). No other code writes the RX mailboxes or RXIMR.

| RX MB | Extended id | Mask | Frames that pass |
|---|---|---|---|
| 16–23 | 0x00004100 | 0x0000FE00 | id byte1 = 0x40 or 0x41 (class 4, node 0 or 1), any function/source/frame index |
| 24–25 | 0x1801AA01 | 0x1FF0FFFF | BMS info frames 0x180nAA01 (§4.2) |
| 26–31 | 0x00002100 | 0x0000FF00 | id byte1 = 0x21 (class 2, node 1) |

CTRL2.EACEN stays 0, so the IDE bit is always compared: standard-id frames never match. Everything else is not stored
and raises no interrupt.

**What the Control sends (A Control 150 / 150.9 / 151, functionally identical; same code in D Control 151).**
`ProcessFirmwareUpdateCommand` stores up to three pending CAN update jobs; the job type byte is copied into the target
class: slot 0 "mppt" = class 2, slot 1 "bms" = class 3, slot 2 "venus" = class 4 (`CAN_Select_Master_Node` accepts only
classes 2..4). `CAN_Update_Init` stops the inverter output on the Control side for every target. All frames are built by
`Register_WriteCategory0xCE` → `CAN_Build_Arbitration_ID(0xCE, node, class, frame index)`:

`id = frame_index << 24 | 0x01 << 16 | class << 12 | node << 8 | 0xCE` (byte2 = source 0x01 = Control).

| Job | node | Ids on the bus (start / data / block end) | Reaches the A micro? |
|---|---|---|---|
| "venus" (class 4) | 1 | 0x000141CE / 0x010141CE / 0x020141CE | yes (MB16–23) — intended target |
| "mppt" (class 2) | 1 | 0x000121CE / 0x010121CE / 0x020121CE | yes (MB26–31) — on the Venus A the class-2 node *is* this micro (§4.1.1) |
| "bms" (class 3) | 0xF (all packs, no version mismatch) or n+1 (single pack whose version differs) | 0x00013FCE …, 0x00013nCE … | **no** — byte1 0x3F/0x3n matches no RX mask |

Payloads: start = u32 image length + u16 CRC16 (6 B), data = 8 B, block end = u16 block CRC + u16 block number (the
micro programs at block number × 0x400). The BMS replies with byte1 = 0x01 (class 0 node 1, the Control) and byte2 =
0x30 | slot (`CAN_Build_Arbitration_ID` of BMS 1105), pack-to-pack 0xCE traffic has byte1 = 0x3n; neither passes the
micro's masks.

**BMS side (A BMS 1105).** One filter bank with id 0 / mask 0 (accept all); the software filter is
`CAN_RX_Command_Dispatcher`: class 3 only, node = own slot, 0xF or 0; 0xCE additionally requires the high nibble of byte2
to be 0 before `BMS_OTA_Flash_Writer` runs (any class-3 0xCE frame sets `g_otaInProgressFlag`). Update frames for class
2/4 are therefore ignored by the packs as well.

**What the micro checks before erasing (`CAN_RxDispatch`, function code 0xCE).**

| Frame index (byte3 bits 0–4) | Condition | Action |
|---|---|---|
| any | – | `g_otaStallSeconds` = 0 (keeps a running session alive) |
| 0 (start) | only: session not active (`g_otaFlags` bit 2 = 0) | `Inv_EpwmForceTrip`, `Mppt_EpwmDisable`, `ShellCmd_stop`, `g_cfgParallelMode` = 1 (inverter start blocked), clear OTA state, **erase 232 × 1 KB from 0x08046000** (synchronously in the dispatcher), store length + CRC, ack `byte1<<16 \| byte2<<8 \| 0xCE` with data 01 00 01 01, `g_otaFlags` \|= 5 |
| 1 (data) | `g_otaFlags` bit 0 | append 8 bytes to the 1 KB stage buffer |
| 2 (block end) | none (no session check) | CRC16 of the stage buffer = received CRC → program 1 KB at 0x08046000 + block × 0x400 (writes outside 0x08000000–0x0807FFFF are refused by `Flash_ProgramDoubleWords`, so only the staging area is reachable); after the last block CRC16 over `length` bytes → `OTA_WriteBootDescriptor` (0x08040000), `g_otaFlags` bit 3 |

No check of class, node, source, device type, firmware version, image length or header before the erase. The only
integrity checks are the CRC16 values computed with the on-chip CRC unit (0x40082000, seed 0xFFFF) per 1 KB block and
over the whole image; no signature, no encryption, no version/rollback check. The bootloader that installs the staged
image is not part of this image, so whether it checks anything beyond the descriptor is **OPEN**.

`App_MainTask` supervises a session: once per second without a 0xCE frame `g_otaStallSeconds` is incremented; at 3 it
either resets into the bootloader (image verified) or aborts: `ShellCmd_start` (MPPT on again), `g_cfgParallelMode` = 0
(set to 0 regardless of its value before the update) and OTA state cleared. The inverter PWM itself is restarted by the
normal state machine.

**Consequences.**

- A BMS update (class 3, broadcast 0xF or single pack) cannot start the micro's update routine — **PROVEN** (hardware
  masks above; the BMS dispatcher has the mirror-image class-3 check).
- Updates for other micros (class 4 node ≥ 2) or other class-2 nodes (node ≠ 1) are also rejected in hardware —
  **PROVEN**. The Control's update code always uses node 1 for classes 2 and 4, so such frames are not generated anyway.
- An update job of type "mppt" (class 2) on a Venus A is received and executed by this micro exactly like a "venus" job:
  PWM off, staging area erased, image programmed and installed if both CRC16 values match — **PROVEN** for the micro
  side. This is consistent with the A design (the class-2 MPPT node is this micro); whether the cloud ever issues an
  "mppt" job to a Venus A, and what the bootloader does with an image built for another device, is **OPEN**.
- Any device on the internal CAN bus can stop power conversion for at least ~3 s and erase the staging area with one
  start frame to byte1 0x40/0x41/0x21 (no authentication) — **PROVEN** from code; same class of finding as the
  unauthenticated shell over CAN (§8).

**Comparison with Venus D micro 116 and E3 micro 119 (read-only).** Both use the STM32G474 FDCAN with extended filters
0x4000/0xFF00 and 0x4100/0xFF00 (byte1 0x40 or 0x41), the BMS filter (D 0x1801AA01/0x1FF0FFFF, E3 0x1800AA00/0x1FF0FF00),
a standard-id filter 0x300/0x300 and "reject non-matching" global filter (`FDCAN1_ConfigFiltersAndMailboxes`,
`FDCAN1_Init`). Class-2 and class-3 update frames therefore do not reach them either. Their 0xCE start additionally
checks the announced image size (100 bytes … half the flash − 16 KB) before `OTA_SelfUpdate_StateMachine` erases, and
the image is checked with CRC16/Modbus — no signature either. **A-specific** is only the additional class-2 node-1
filter (because the MPPT stage sits in this micro) and the unconditional erase/PWM trip on the first start frame. Side
note for D/E3 (INFERRED from the filter and dispatcher code, not tested): their standard-id filter passes 11-bit ids
0x300–0x3FF and the dispatcher takes the low byte as the function code, so a standard frame 0x3CE would enter the 0xCE
start path; no firmware in the set sends standard ids on this bus.

### 4.2 BMS frames and the "PV rescue" handshake

Accepted BMS frames: id `(id & 0x1FF0FFFF) == 0x1800AA01`, byte2 low nibble n = 1..4, stored at 0x20120440..0x2012045F:
frame 1 battery V ×0.1, current, max temperature, SoC ×0.1; frame 2 capacity, sleep flag; frame 3 charge voltage ×0.1,
max charge / discharge current ×0.1, charge request, **force-charge request**; frame 4 BMS error and warning word.
Frame 1 refreshes the BMS-alive timestamp (alive = < 21 s). `Bat_UpdatePermits` derives the charge/discharge permits,
low-SoC latch (≤ 5 %), err1 bit 20 (communication lost) and bit 21 (BMS error ≥ 4). Without BMS (battery mode 0) fixed
limits 35.75 / 47.125 V, 60 / 70 A are used. Events 24/25 ("forced-charge request", "forced charge without
communication") belong to the forced-charge sequence in `Mode_UpdateStatusAndAging`/`Inv_StateMachine`.

**0x03410110:** the A BMS 1105 waits up to 120 s in cell-UVP level 2 for id 0x03410110 with data[1] > 8. That id is
`0x03<<24 | 0x410100 | 0x10` = frame index 3 of the micro's func-0x10 telemetry reply (built by
`CAN_TxQueue_PushMultiFrame`), i.e. telemetry bytes 16..23. Data[1] = byte 17 = high byte of `grid voltage ×10`, so the
condition means **grid voltage ≥ 230.4 V (0x900)**. The frame is only on the bus while the Control polls func 0x10;
the micro has no special rescue sender. Whether the BMS developers intended the grid voltage or another field cannot be
decided from the micro side (open point).
`0x100101AA` (byte1 = 0x01, class 0) is ignored; id `0x00210100` (BMS error lock) would be a class-2 reply to func 0x00,
which the micro never generates.

### 4.3 Debug shell (letter-shell 3.2.1, UART1 115200 Bd or CAN)

68 table entries at 0x08024988 (51 commands plus built-ins/keys). Commands (handler `ShellCmd_<name>`):
`mppt_buck/boost/exbuck/exboost/buck_boost` (open-loop duty per channel), `mppt_epwm_start/stop`, `adcshow`, `prtshow`,
`dshow/cshow/mshow/ashow/pshow` (MPPT status), `start/stop/ss`, `mppt_pow`, `bms_power`, `mppt_vol`, `clear_fault`,
`sr/sw` (software-I2C test), `inv_hr_start/stop/set`, `llc_start/stop`, `log_list`, `prt`, `llc_feq`, `bat_mode`,
`reset_memory`, `set_power`, `backup_mode`, `par_mode`, `set_mode`, `off_pi/on_pi/qpr` (controller gains), `reset`,
`io_show/io_set`, `rec`, `set_sr/open_sr`, `version`, `set_soft`, `power_cfg`, `test`.
`power_cfg`/`Inv_RequestCommand` codes: 1 stop, 2 PFC open-loop duty, 3 PFC open-loop off-grid, 4 off-grid output,
5 charge CV/CC (35–45 V, 0–30 A), 6 parallel test, 8 grid-tied power, 9 open-loop sine, 10/11 LLC open-loop
charge/discharge (66–69 kHz), 12 LLC soft start, 21/22 single-PFC charge/grid-tied, 30 standard mode, 40 constant-voltage
output, 50 over-voltage test.

### 4.4 Serial EEPROM (24Cxx, software I2C SCL GPIOA26 / SDA GPIOA27, WP GPIOB1, device 0xA0)

| Address | Content |
|---|---|
| 0x001 | reset type: 3 = initialised, 2 = reload defaults (keep totals — but see §10), 0 = full clear |
| 0x003 | stored firmware version (u16, defaults re-written if < 108) |
| 0x011–0x015 | backup flag, parallel mode, battery mode, **work mode (0x014)**, inverter-disable flag |
| 0x100 | 20 B energy/operating counters (charge/discharge Wh …) |
| 0x10C / 0x110 | energy restore values from CAN func 0x16 |
| 0x200–0x206 | grid standard code, max SoC 100, min SoC 12, **0x203 charge limit, 0x205 discharge limit (default 1450 W)** |
| 0x300–0x308 | test/aging parameters; 0x302/0x306 ATE-aging results |
| 0x502–0x50C | date words of the energy-counter hierarchy |
| 0x510–0x524 | six u32 PV energy counters |
| 0x540/0x541 | manual MPPT voltage override flag + value (0.1 V) |

## 5. Control and power section

### 5.1 Hardware mapping

EPWM2/EPWM3 = inverter H-bridge (28 kHz up-down, HRPWM), EPWM1/EPWM5 = LLC legs (67 kHz, 66–69 kHz allowed),
EPWM4/6/7/8 = PV converters 1..4 (200 kHz). ADC A/B/C results are DMA-copied; channels: battery current/voltage, grid
voltage, inverter inductor current, load voltage/current, PFC voltage, 400 V bus, PE voltage, PV1..4 V/I, three NTCs,
chip temperature. CMPSS1-7 = hardware over-current/over-voltage comparators, trip latch reset via GPIOA29, trip input
GPIOA28. Grid relay GPIOB26, off-grid relay GPIOB6. Grid frequency from ECAP1 (5.2e8 / sum of 4 periods).

### 5.2 Work modes (`g_cfgWorkMode` 0x2010077F, EEPROM 0x014)

| Value | Meaning |
|---|---|
| 0 | default ("Ctl_mode_default"): CAN power setpoint (class-4 0x01) and backup command (0x02) honoured |
| 1 | manual: CAN 0x57 (backup) and 0x58 (±2499 W setpoint) honoured |
| 2 | debug (fallback for every value not listed, including 15) |
| 3 | aging: ±2200 W cycles between SoC 15 % and 99.9 % (`Mode_UpdateStatusAndAging`) |
| 4 | ATE aging (CAN 0x60): −1350 W to 100 %, rest, +1350 W to 30 %, rest; results to EEPROM 0x302/0x306 |
| 12 | CAN test mode (func 0x50), required for 0x51–0x56 |
| 13 | voltage-source mode |

The shell help text ("0 zero feed-in, 1 manual, 2 trading, 3 debug, 15 bidirectional voltage source") is copied from
the Control and does not match the code. Further flags: backup (EEPROM 0x11), parallel 0/1/2 (0x12), battery mode
(0x13: 1 = BMS, 0 = fixed limits), inverter disable (0x15).

### 5.3 State machine and operating commands

`Inv_StateMachine` (state `g_invState` 0x20120510, 231 states, 1 kHz). State 0 = idle: the pending command
(0x20121B34, from `Inv_RequestCommand`/CAN/boot) selects the start sequence: 10 charge CV/CC, 20 grid-tied power,
30 off-grid output, 40 parallel test, 70/80 single-PFC, 100 LLC soft start, **200 standard mode** (automatic
grid-tied / backup operation, entered at boot unless work mode is 2 or 13), 0xDC–0xE4 open-loop tests. Sequences check
grid permit (grid qualification by `Grid_CheckVoltageFrequency`), battery permits, DC-bus pre-charge
(`Bus_CheckChargeDischargePermit`: discharge after ≥ 355 V, charge while ≤ 480 V), switch relays and start
PFC/LLC/H-bridge; failures log the reason (event log, §6.3) and set err2 bits 7/9/11/13.

Controllers (`Ctrl_InitControllers`): on-grid current PI Kp 12 / Ki 0.6, off-grid voltage PI Kp 0.11 / Ki 0.13,
bus PI Kp 28 / Ki 0.35, QPR current loop (50 Hz), SOGI-PLL, bus notch at 2× grid frequency, off-grid droop
0.0004 Hz/W and 0.001 V/var around 326 V peak / 50 Hz. Power setpoint clamp `Inv_SetPowerSetpoint`: max
`g_invPowerLimitMaxW` (1450 W, derated above 73 °C), min `g_invPowerSetpointMinW` = −limit or −(BMS max charge current ×
battery V × 1.06); low-SoC latch blocks discharge.

Grid limits (`Prot_LoadGridLimits`, grid standard EEPROM 0x200): default 275 V/55 s, 287.5 V/1 ms, 174 V/55 s,
103 V/1 ms, 51.8/47.4 Hz, reconnect 265/197.8 V, 50.8/47.6 Hz, waits 1 s/2 s; standard 2 (EU-like) and 3 tighter.
50/60 Hz auto-detection for standards 0/1.

### 5.4 MPPT (four PV inputs)

Per channel: buck, boost, ex-buck, ex-boost (and buck-boost in the shell), mode chosen from the conversion ratio
battery-side/PV voltage (`Mppt_EpwmApplyMode`); closed-loop compare updates in `Ctrl_MainControlIsr`, tracking in
`Ctrl_SlowControlTask` with ΔV/ΔI dead bands (incremental-conductance style; exact variant not fully verified),
start above ≈16 V PV. Limits: PV over-current 20 A, reverse current −3 A, PV over-voltage ≈63 V, battery-side
≈55 V, PE voltage 60 V, temperatures 90 °C (derating from 73 °C). Total power reference clamp 2400 W, power
balancing between channels unless "no power distribution" (class-2 mode 1). Status/fault words: MPPT fault word
0x201005B0 (codes 0x440–0x46A), warning word 0x201220EA (codes 0x540/0x541, 0x553–0x55A). On the Venus A the
Control's `mppt*` registers (g_mpptMode, g_mpptPowerW, g_mpptVoltage_0p1V, class-2 replies 0x03/0xFE/0xFF) therefore
describe this on-board MPPT stage.

## 6. Protection and fault codes

Setter idiom: `Log_AddEvent(id, …)` then OR into the word. All set sites enumerated (corrected against the earlier
wrong-base analysis).

### 6.1 err1 `g_protErrorWord1` (0x20101650) → Modbus 36100/36101

| Bit | Event | Condition (default limits) | Function |
|---|---|---|---|
| 0 | 0x2C | grid Vrms > 275 V for 55 s or > 287.5 V | `Grid_CheckVoltageFrequency` |
| 1 | 0x2C | grid Vrms < 174 V for 55 s or < 103 V | same |
| 2 | 0x2A | grid frequency > 51.8 Hz for 30 ms | same |
| 3 | 0x2A | grid frequency < 47.4 Hz — **trips immediately** (inverted counter compare, §10) | same |
| 4 | 0x2D | instantaneous grid voltage > 430 V | same |
| 7 | 0x2E | waveform deviation |v − Vrms·√2·sin θ| > 80 V for 5 samples | same |
| 20 | (0x1A on clear) | BMS communication lost (no frame 1 for 21 s) | `Bat_UpdatePermits` |
| 21 | 0x3D | BMS error code ≥ 4 (frame 4) | same |

Bits 0–15 are cleared when the grid qualifies again (event 6). The earlier analysis had bits 0/1 as frequency and
2/3 as unknown — that was based on the wrong image base; the table above is from the correctly based image.

### 6.2 err2 `g_protErrorWord2` (0x20101654) → Modbus 36102/36103

| Bit | Event | Condition | Function |
|---|---|---|---|
| 5 | 0x43 | inverter current |I| > 15 A peak (all PWM stop) | `Ctrl_InverterControlStep` |
| 6 | 0x35 | DC bus > 530 V (> 570 V in LLC mode 0x0E) | same |
| 7 | 0x15 | state-machine shutdown with reason code 5 | `Inv_StateMachine` |
| 8 | 0x31 / 0x32 | off-grid overload i²t integrator > 600000 (rated 6.54 A) or off-grid voltage < 207 V for 200 ms | `Prot_UpdateOffGridOverload` |
| 9 | 0x29 | invalid state index (> 0xE6) | `Inv_StateMachine` |
| 11 | 0x36 | state-transition timeout (bus pre-charge / LLC start / relay sequence), 3 sites | `Inv_StateMachine` |
| 13 | 0x48 | relay check failed after 20 s (event text "grid relay welded") | `Inv_StateMachine` |

### 6.3 warn `g_protWarningWord1` (0x20101658) → Modbus 36000/36001

No set site in the image — always 0.

Event log (`Log_AddEvent`, ring 0x20122118, 230 × 20 B, shell `log_list`): 5 idle, 6 grid detection OK, 7/8 HV-bus
discharge/charge allowed, 9/10 battery charge/discharge allowed, 13 pre-charge on, 15/16 start H-bridge, 17/18 LLC start /
soft start, 20 inverter off, 21 system off, 22 backup on, 23 BMS sleep request, 24 forced-charge request, 25 forced charge
without communication, 26 battery communication OK, 27 grid frequency detected, 28 reset reason, 41 state-machine fault,
42 grid frequency fault, 43 SPLL fault, 44 grid VRMS fault, 45 grid peak voltage, 46 waveform, 47 grid current,
48 software-PLL restart, 49/50 off-grid current/voltage protection, 51 grid detection fault, 52 battery communication
fault, 53 bus voltage fluctuation, 54 state switch timeout, 55/56 hardware protection, 57 PWM over limit, 58 self-test,
60 battery abnormal, 61 battery fault code, 62 battery voltage, 63 allowed output current, 64 SoC limit, 66 bus voltage,
67 peak current, 68 peak voltage, 69 watchdog reset, 70 EEPROM write error, 72/73 grid/off-grid relay welded,
74 off-grid zero-crossing close.

## 7. OTA / update

Only the CAN path 0xCE (§4.1.5) into the staging area 0x08046000 plus boot descriptor at 0x08040000; the bootloader
(not part of this image) installs it. No YMODEM/UART update and no BMS relay update in this image (unlike the D micro).
Integrity: CRC16 only. Accepted senders and start conditions: §4.1.6.

## 8. Security notes

- Shell user "letter V1211" without password; shell over CAN (func 0xC2/0xC3) gives everybody on the internal CAN bus
  (and the Control's debug forwarding) full console access: `reset_memory`, `set_mode`, `power_cfg` open-loop PWM
  commands, `io_set` (relays), `llc_feq` …
- OTA over CAN accepts any image with a matching CRC16 — no signature. The first start frame (no address, length or
  device check) trips the PWM and erases the staging area; class-3 (BMS) update frames are filtered in hardware (§4.1.6).
- CAN test commands 0x50–0x58 (mode 12) and the aging modes run the power stage outside normal supervision.

## 9. Differences to the Venus D inverter firmware (v116)

| Topic | Venus D micro 116 | Venus A micro 1211 |
|---|---|---|
| MCU | STM32G474 (Cortex-M4F), CCM code | NS800RT-family Cortex-M7, ITCM/DTCM |
| OS | FreeRTOS | bare metal |
| CAN | FDCAN, class 4 only (class 2 ignored) | FlexCAN, class 4 **and class 2 (MPPT)** |
| PV | separate class-2 MPPT node | 4 PV inputs on this board |
| RS485 / Modbus registers | `Modbus_HandleRegisterAccess` (40000/41000/45000 …) | none |
| Power limit | 2500 W (v116), CAN 0x03/0x04 < 2501 | 1450 W, CAN 0x03/0x04 < 1451 |
| Real-time setpoint | 0x56/0x58 ±2500 W | 0x56 ±1449 W (mode 12), 0x58 ±2499 W (mode 1), clamp 1450 W |
| Error words | 0x200019F4.. (D bit meanings) | 0x20101650.. (meanings in §6, different) |
| Update paths | YMODEM UART, CAN self-update, BMS relay | CAN 0xCE only |
| EEPROM | I2C hardware peripheral, map 0x102/0x300/0x301 … | software I2C, map §4.4 |

## 10. Firmware quirks (reported, not fixed)

1. `Grid_CheckVoltageFrequency` 0x0801478C: `cmp r1,r2; bcs skip` — the under-frequency counter trips while
   `count < limit`, i.e. on the first sample below 47.4 Hz (over-frequency uses `count > limit`).
2. Class-4 func 0x54 replies with the uninitialised stack byte `local_35`.
3. `set_mode` help text does not match the accepted values (15 → 2).
4. `Param_LoadFromEeprom` with reset type 2 reads and then zero-fills the counter block (the "keep totals" reset clears them).
5. `EEPROM_SelfTest` (CAN test command 1) writes EEPROM 0x100/0x101 (counter block).
6. `Flash_ProgramDoubleWords` returns on a verify error without clearing PG / locking.
7. `power_cfg` error text says 40–60 V / 0–60 A, the check is 35–45 V / 0–30 A.
8. `_sys_exit` is implemented empty (would return into the reset path).

## 11. Open points

1. Exact NS800RT member (peripheral base map differs from the public RT7 header); CMPSS layout is tentative.
2. Meaning of 0x03410110 data[1] > 8 on the BMS side (grid voltage ≥ 230.4 V as the code implies, or a
   misinterpreted field?) — needs the BMS developers' intent or a device test.
3. MPPT tracking algorithm variant and the meaning of the five energy-counter levels (date hierarchy hypothesis).
4. Reason code 5 of err2 bit 7 and the exact relay/voltage condition behind err2 bit 13.
5. PE-voltage fault (MPPT bit 10/11) — insulation / PE monitoring hypothesis.
6. Checks done by the A micro bootloader before it installs a staged image (bootloader not in the image, §4.1.6).
