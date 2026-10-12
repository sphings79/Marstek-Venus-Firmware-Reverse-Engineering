# Micro-FW v119 (Venus E3) — Full analysis

> 🇩🇪 Deutsche Version: [VNSE3_Micro_FW_v119_Vollanalyse_de.md](VNSE3_Micro_FW_v119_Vollanalyse_de.md)

**Status:** 2026-10-02.
**Image:** `119_micro_VNSE3-0_inv_app_0119_0326_ota_152726.bin` (Venus E 3.0 inverter board, "inv_app").
**Decompilates/Listing:** `Dekompilate/VNSE3_Micro_119/` (local only, not in the repo): 708 functions, all named and typed,
`listing/` with flash and CCM raw disassembly and `globals_and_registers.txt` (2509 named RAM/peripheral symbols).
**Related:** [VNSD_Micro_FW_v116_Full_Analysis.md](VNSD_Micro_FW_v116_Full_Analysis.md) (Venus D inverter, nearest reference),
[VNSA_Micro_FW_v1211_Full_Analysis.md](VNSA_Micro_FW_v1211_Full_Analysis.md) (Venus A inverter),
`../Control_FW/VNSE3_Control_FW_v151_Full_Analysis.md`, `../BMS/VNSE3_BMS_FW_v115_Full_Analysis.md`.

## 0. Summary

- **Same MCU and same code base as the Venus D inverter firmware.** STM32G474 (Cortex-M4F, HRTIM, FDCAN, CORDIC), Keil ARMCC,
  FreeRTOS (heap_4), letter-shell; time-critical code runs from CCM-SRAM (0x6F74 bytes copied to 0x10000000) and is reached through
  long-branch veneers. **73.5 %** of the image bytes are identical to D Micro 116 (pointer and BL targets masked), 70.5 % to D 115,
  only 2.3 % to A Micro 1211 (C library and strings; different MCU and code base). 546 of the 693 D 116 functions are byte-identical,
  64 are changed, 83 D functions have no counterpart at the same place (moved, merged or removed); E3 has 708 functions (29 names do not
  exist in D 116).
- **Build is older than D 116:** "Mar 26 2026 15:20:51" (D 116: "Jul 2 2026"). Version word 0x77 (119) followed by the device tag
  **"AC02"** at 0x08010000 (D: 0x74 + "VD01"); image trailer tag **"VNSEE3"** (D: "VNSD") — the same model tag the E3 Control 151 checks.
- Manufacturer changelog v119: "1.修复soc低于6%不强充问题" = fixed forced charging not starting below 6 % SOC. The exact fix is not
  pinpointed (§5.4); the low-SOC code in `Inv_GridConnectionSupervisor` (SOC < 6 % threshold, new debounce) and the new CAN cut-off-SOC
  command 0x0A (accepts 6..79 %) are the code locations involved.
- **No PV/MPPT** (as on the Venus D, the micro never had the PV stage; the E3 also has no class-2 MPPT node). One ADC2 channel (third
  NTC) and the NTC table 1 are removed; two NTCs remain.
- **Power path changes:** inverter bridge PWM **30 kHz** (D 28 kHz, HRTIM period 45333 vs 48572), LLC start 128 kHz (D 126 kHz),
  grid-current controller is now a **PR (proportional-resonant) controller** instead of a PI loop, plus a zero-crossing burst/skip scheme
  for small references. New automatic **fan control** (`Fan_UpdateDutyByTemperature`).
- **Power limits:** the user-facing limits are unchanged — CAN 0x03/0x04 accept ≤ 2500 W, Modbus 44002/44003 ≤ 2500 W, setpoints
  0x56/0x58 ±2500 W. But the **factory defaults are 3000 W** (D 2500), the micro's own Modbus 43133/43134 accept up to **3000 W**,
  three 3000.0f constants replace 2500.0f in `Ctrl_MainControlIsr` (controller limit at grid-connect ramp, thermal derating base), and
  the battery calibration / work-mode-3 schedule use ±3000 W. Register 45502 (factory test) uses −2400 W (D −2500 W).
- **CAN (class 4, from the Control):** E3 adds commands **0x08** (parallel/operating mode 0/1/2), **0x0A** (cut-off SOC 6..79 %),
  **0x14** (read limits + SOC limits) and **0x17** (floors of two new energy counters); the telemetry reply 0x10 grows from 48 to
  **56 bytes** (two new u32 energy counters). **0xCC (date/time) is not consumed — neither by E3 119 nor by D 116.**
- **0x04410110** (used by the E3 BMS as "current from the inverter") is **frame #4 of this micro's telemetry reply** (func 0x10,
  source 0x41, destination 0x01 = Control). The micro **never sends standard 11-bit ids**, so it is not the sender of 0x305, 0x300 or
  0xCF (§4.1.5).
- **Warning word:** E3 sets **bit 0 of `war1` (Modbus 36000)** while the grid-permit flag is 0 — D 116 never writes the warning word.
  All error-word bits and event codes are otherwise identical to D 116.
- Several D 116 names turned out to be misnomers and were corrected in the D 115/116 and E3 programs (§9.2).

### Answers to the questions for this image

| # | Question | Answer (details) |
|---|---|---|
| a | Class-4 commands consumed / ignored; 0xCC needed? | Consumed: 0x01–0x08, 0x0A, 0x10–0x14, 0x16, 0x17, 0x50–0x58, 0x60, 0xC1–0xC3, 0xCB, 0xCE. Ignored: 0x09, 0x0B–0x0F, 0x15, 0x18, 0x30–0x3E, 0x40–0x43, 0x59–0x5F, 0x61–0xC0, **0xC4, 0xCC**, 0xCF, 0xFB/0xFE/0xFF; class-2/3 frames are filtered in hardware. The micro does **not** need 0xCC: neither E3 119 nor D 116 has a 0xCC case, its time comes from its own RTC (`RTC_ReadTime`). The E3 Control's removal of 0xCC/0xC4/0xCB-DLC2 therefore costs nothing on the micro side. (§4.1) |
| b | 0x04410110, 0x305, 0xCF, 0x300 | 0x04410110 = telemetry frame #4 (bytes 24..31) of the micro's reply to the Control's 0x10 poll; the BMS sniffs it: int16(bytes 28..29) × 10000 / u16(bytes 30..31 = battery voltage × 10) → mA. 0x305/0x300/0xCF are 11-bit ids; the micro only transmits 29-bit ids → **not the micro**. If such frames were on the micro's bus, its std-id filter (0x300/0x300) would feed 0x300–0x3FF into the class-4 dispatcher (0x305 → func 0x05 = save + reboot, 0x360 → func 0x60 = calibration mode), which makes it very unlikely that the BMS's Pylontech-style 0x305/0x35x/0x36x exchange runs on the internal Control–micro–BMS bus. Most likely an external battery-protocol port; sender not identifiable from the firmware set. (§4.1.5) |
| c | Modbus handler, registers, limits, modes, error words | `Modbus_HandleRegisterAccess` (0x08016E34) serves the micro's **own** RS485 register space (40000, 41000/41001/41010/41100/41200, 42000/42010/42011, 43100–43134, 44000–44003, 44100, 45000, 45010–45019, 45200, 45300, 45400–45402, 45500–45502, 45600/45601, 45603/45604, 46000) — identical set to D 116; the Control does not forward its Modbus registers to it (Control registers become CAN commands). Limits 2500 W (CAN/Modbus), 0 is accepted and persisted (EEPROM 0x202/0x204), reload without range check, factory default 3000 W. 45000: 0x55EE → work mode 12, 0x55DD → reload modes from EEPROM, 0x55FF → clear derating + 2500 W. Error words: same bit set as D 116 plus war1 bit 0 (§6); A bit names do not apply. (§4.2, §5.2, §6) |
| d | Differences to D 115/116 and A 1211 | §9 (table). Main points: 30 kHz PWM + PR current control, 3000 W internal/default limits vs 2500 W user limits, new CAN 0x08/0x0A/0x14/0x17, 56-byte telemetry with energy counters, war1 bit 0, fan control, USART1 console via HAL UART, no third NTC; standby latch and sleep handling identical to D. A 1211 is a different design (NS800RT, no RTOS, own PV stage, 1450 W, class-2 handler, no Modbus). |
| e | E3-specific | Hardware-variant input (`g_hardwareVariant`, HRTIM base offset 340/410), parallel-mode CAN command, cut-off SOC down to 6 %, grid standard 4 with its own limit set, energy counters, fan curve, shell `test` writes the grid standard to EEPROM without range check. Owner statement: the Venus E3 has exactly one fixed built-in battery pack, not expandable — the micro handles exactly one BMS (PF1–PF4 frames, no per-pack data). (§5, §9) |

**Open Control questions answered from the micro side**

- Class-4 codes consumed by the E3 micro: see row (a). Of the codes the E3 Control 151 still sends, **0x18, 0x30–0x3E, 0x40–0x43,
  0xCF, 0xFB/0xFE/0xFF** reach no handler in the micro (class-2 frames do not even pass the filter).
- **Reshuffled "inverter dirty bitmap" (E3 Control 0x08009D74 / 0x08009D84):** this bitmap is **not** related to the micro. The
  Control writes its four bytes `g_invRegPendingBits` over bit-banged I2C as [register, value] with the register table
  {0x68, 0x6A, 0x6C, 0x6E} (E3 Control 0x08059AF5) — the digit registers of a TM1650-type LED/7-segment driver. The E3 layout
  (0x1C bits, 7 per byte = 28 LED segments; D 0x18 bits, 6 per byte = 24) and the 2-bit fields therefore describe front-panel LED
  segments; the wrappers byte-identical to D's `Inverter_RegDirty_Mark_GridPermit` / `_PvVoltage` (formerly `_DischgVolt` / `_BatteryParams`) just set LED segments. No CAN
  frame and no micro register is involved.

## 1. Binary fingerprint

| Feature | Value |
|---|---|
| File | `119_micro_VNSE3-0_inv_app_0119_0326_ota_152726.bin` |
| Size | 119,808 bytes (0x1D400) |
| SHA-256 | `3c834d677e0d56868af3dab18bd7c1ae44ea6a2a776c70f3756991d6592eeb5d` |
| Link address | `0x08004000`–`0x080213FF` (16 KB bootloader below) |
| MCU | STM32G474 (Cortex-M4F), FLASHSIZE register 0x1FFF75E0 used by the OTA size check |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ-compressed .data) |
| Libraries | FreeRTOS (heap_4, 16 KB heap), letter-shell, ST HAL (now including the HAL UART module) |
| Vector table | 0x08004000, initial SP 0x20009AC0, reset vector 0x080042AD |
| Version | 0x77 (119) at 0x08010000, followed by "AC02"; `ShellCmd_version` prints SOFT_VERSION 119 and build "Mar 26 2026 15:20:51" |
| Trailer | device tag "VNSEE3" at 0x080213E0, trailer word 0x44443E54 at 0x080213FC (meaning not determined; D: "VNSD", 0x44443331) |
| Changelog | v119: fixed forced charging not starting below 6 % SOC |

Note: the firmware archive also contains a VNSE3-0 Micro 116 (117,760 B) that was not analysed here.

**Scatter loading** (`Region$$Table$$Base` 0x08019250, `ScatterRegion[3]`):

| Source (flash) | Target | Length | Method |
|---|---|---|---|
| 0x080195B0 (1120 B compressed) | 0x20000000 (.data) | 0x8FC | `__decompress` |
| 0x08019A10 | 0x10000000 (CCM-SRAM, code + data) | 0x6F74 | `__scatterload_copy` |
| – | 0x200008FC (.bss) | 0x91C4 | zero init (ends at the stack top 0x20009AC0) |

## 2. Memory map and analysis status in Ghidra

Program `/VNSE3/Micro/Micro_119_VNSE3-0_inv_app_0119_0326_ota_152726.bin`:

| Block | Range | Content |
|---|---|---|
| `flash` | 0x08004000–0x080213FF | code to ~0x08018880, strings/HAL tables, region table, `shellCommandTable` (51 entries @0x08019280), .data and CCM load images, 0xFF fill, trailer |
| `.ccmram` | 0x10000000–0x10006F73 | CCM code (veneers to flash 0x10000000–0x1000013F, control ISRs, PLL, PR controller) and CCM data |
| `.data` / `.bss` | 0x20000000–0x200008FB / 0x200008FC–0x20009ABF | globals, FreeRTOS heap (`ucHeap` 0x200045BC, first block 0x200045C0) |
| peripherals | APB1/APB2/AHB1/AHB2 (GPIO, ADC/DAC), FMC/QSPI, PPB | STM32G474 SVD register names |

Verification gates (`marstek_gates_micro.py`, final state): 0 undefined bytes and 0 orphan instructions (flash + CCM), 0 flow into
data, 0 function pointers to non-code, 0 literal-only functions, 0 cross-block bodies, 0 `FUN_`/duplicate/temporary names,
**0 unnamed referenced RAM addresses**, 0 default prototypes (task entries `void f(void *pvParameters)`), no-return flags sane.
Tags: `NAME_VERIFIED` on all 708 functions, `NAME_LOW_CONFIDENCE` on 74. Decompilation export: 0 failures, no unrecovered jump tables.
Method: names, prototypes, comments and globals were ported from D 116 for byte-identical segments (layout, data types, globals by
aligned instruction streams with a neighbour-delta consistency filter; all 546 identical functions reference the same objects as in
D), the rest was named from the E3 code.

## 3. FreeRTOS tasks

`Task_CreateAllAndStartScheduler` (0x0800F7C4) — same seven tasks, stacks and priorities as D 116:

| Task | Entry | Stack (words) | Prio |
|---|---|---|---|
| `vtask_led` | `Task_SystemHousekeepingLoop` 0x08016D34 | 128 | 16 |
| `vtask_time` | `Task_ModeRoundRobinLoop` 0x08016E14 | 256 | 15 |
| `vtask_modbus` | `Task_ModbusLoop` 0x08016D70 | 128 | 14 |
| `vtask_shell` | `Task_UpdateBms` 0x08016D88 | 512 | 9 |
| `vtask_prt` | `Task_AdcDebugPrintLoop` 0x08016D7C | 256 | 8 |
| `vtask_can` | `Task_CanTxLoop` 0x08016D18 | 256 | 6 |
| `vtask_can_receive` | `Task_CanRxLoop` 0x08016D28 | 256 | 7 |

Changes: the CAN RX queue holds 20 frames (D 40) and `Task_CanRxLoop` yields one tick per pass (D busy loop). Control interrupts:
`TIM7_ControlLoopCallback` (1 kHz) → `Ctrl_MainControlIsr` (CCM), `Inv_CurrentLoopIsr` (CCM, per PWM period),
`HRTIM_TIMB_IRQn_IRQHandler` (LLC compare update).

## 4. Interfaces

### 4.1 CAN (FDCAN1, 29-bit ids)

**Filters** (`FDCAN1_ConfigFiltersAndMailboxes` 0x080058e8): standard id 0x300/mask 0x300 → FIFO0; extended 0x4000/0xFF00 and
0x4100/0xFF00 (Control class 4, node 0/1); extended **0x1800AA00/0x1FF0FF00** (BMS frames 0x18n0AAxx; D: 0x1801AA01/0x1FF0FFFF).
Global filter: non-matching standard and extended frames and remote frames are rejected. RX: `HAL_FDCAN_RxFifo0Callback` →
`BMS_CAN_Parser` (mode 3), `BMS_FwUpdate_CanHandler` (mode 4) or queue → `CAN_RX_DispatchTask` → `EMS_Inverter_CAN_Dispatcher`.

**Id layout.** Function code = id bits 0–7; the reply goes to id bits 16–23 of the request. Frames sent by the micro:
`CAN_BuildArbID`: `frame_index << 24 | 0x41 << 16 | destination << 8 | func` (`g_canTxSrcAddr` = 0x41). Payloads > 8 bytes are
split into frames with index 1..n (plus a 4-byte header frame 0 if the length is not a multiple of 8). All transmissions use the
extended id format (`CAN_TX_SendFrame`).

#### 4.1.1 Class-4 commands (`EMS_Inverter_CAN_Dispatcher` 0x0800597C)

| Func | E3 action | vs D 116 |
|---|---|---|
| 0x01 | power setpoint (`Ctrl_SetPowerReferenceClamped`), work mode 0 only; echo | same |
| 0x02 | backup output enable flag (EEPROM 0x300), work mode 0 only | same |
| 0x03 | max discharge power `g_powerLimitDischargeW`, value < 0x9C5 (≤ 2500 W) → EEPROM 0x204; ack also when dropped | same |
| 0x04 | max charge power `g_powerLimitChargeW`, < 0x9C5 → EEPROM 0x202 | same |
| 0x05 | save operating stats, EEPROM 0x901 = 1, watchdog reboot of the micro (sent by the Control at boot) | same |
| 0x06 | standby latch `g_standbyLatch` = 1 (§5.3) | same |
| 0x07 | sequence handshake (`g_emsDispatchSeq` = data + 1) | same |
| **0x08** | parallel/operating mode `g_cfgParallelModeEnabled` = 0/1/2 (EEPROM 0x374; value 1 is not persisted), work mode 0 only; ack | **new** (D ignores) |
| **0x0A** | cut-off SOC `g_cfgCutoffSocPct` = data (6..79 %), EEPROM 0x201, work mode 0 only; ack | **new** |
| 0x10 | telemetry, **56 bytes** (§4.1.3) | D 48 bytes |
| 0x11 | 8 B: work mode, standby latch, battery mode, 0, charge limit, discharge limit | same |
| 0x12 | 6 B battery calibration state / discharge / charge time | same |
| 0x13 | 4 B EMS charge/discharge limits (EEPROM 0x950/0x952) | same |
| **0x14** | 6 B: charge limit, discharge limit, max SOC (EEPROM 0x200), cut-off SOC (0x201) | **new** |
| 0x16 | floors for the charge/discharge counters (no reply) | same |
| **0x17** | floors for the two new energy counters (0x20003EF3/EF7; no reply) | **new** |
| 0x50 | 0 → LLC/PFC stop; 1 → work mode 1 + dispatch 7 | same |
| 0x51–0x56 | factory tests (work mode 12): beep, test sub-commands, IO status, power test ±2500 W | same |
| 0x57 | backup flag = (work mode 1 && data == 1) | same |
| 0x58 | manual power setpoint ±2500 W, work mode 1 only | same |
| 0x60 | work mode 4 (battery calibration), step = data | same |
| 0xC1 | EEPROM self-test, answered with 0xC1 | same |
| 0xC2 | debug level 1/2 | same |
| 0xC3 | bytes into `g_shellRxQueue` (shell/BMS relay input) | same |
| 0xCB | 8 B: model code 0xD, version 0x77, hardware variant, 1 | same layout |
| 0xCE | CAN self-update (§7) | same |
| others (0x09, 0x0B–0x0F, 0x15, 0x18, 0x30–0x3E, 0x40–0x43, 0x59–0x5F, 0x61–0xC0, 0xC4, **0xCC**, 0xCF, 0xFB–0xFF) | ignored, no reply | same as D 116 |

The switch on codes 0x01–0x17 was enlarged from 20 to 23 cases (TBB table at 0x080059C4).

#### 4.1.2 BMS frames (`BMS_CAN_Parser` 0x08005928, identical to D)

`(id & 0xFFFFF) >> 16`: 1 = pack voltage/current + temperature/SOC (also BMS-alive tick), 2 = capacity + pack info (byte 7 = BMS
sleep flag), 3 = charge voltage limit + discharge current limit (+ charge/force-charge request bytes), 4 = error and warning word.
Only one BMS is handled — no per-pack frames, matching the owner's statement of exactly one fixed pack. The BMS communication
watchdog re-initialises FDCAN1 after 10 s without frames (`LED_TaskTick`).

#### 4.1.3 Telemetry payload (func 0x10, `g_emsTelemetryBlock` 0x2000391C, built every 400 ms)

Offsets 0x00–0x2F as on D 116: state, buzzer, grid flag, backup flag, warning word (+0x04), error word 1 (+0x08), error word 2
(+0x0C), grid voltage ×10, grid frequency ×10, off-grid voltage ×10, grid-permit flag `g_gridPermitFlag` (+0x16), grid-tied power `g_gridTiedPower` (+0x18), off-grid power (+0x1A),
battery power `g_batteryPowerAvg` (+0x1C, int16, the value the BMS uses for its current calculation), battery voltage ×10 (+0x1E), two temperatures ×10 (+0x20 LLC
heatsink, +0x22 inverter — order swapped versus the D 116 names), max/min power reference, total charge / discharge counters
(+0x28/+0x2C). **New: +0x30 / +0x34 = two u32 energy counters** (`g_energyPosWh` / `g_energyNegWh`, §5.5).

#### 4.1.4 Frames sent by the micro

Only replies and acknowledgements (0x01–0x08, 0x0A, 0x10–0x14, 0x51–0x58, 0x60, 0xC1, 0xC2, 0xCB, 0xCE), the debug queue output
(`CAN_TX_FlushDebugQueue`) and the BMS firmware relay; no periodic broadcast.

#### 4.1.5 The BMS ids 0x04410110, 0x305, 0x300, 0xCF

- **0x04410110** = `4 << 24 | 0x41 << 16 | 0x01 << 8 | 0x10`: frame index 4 of the micro's func-0x10 reply to the Control (node 0x01),
  carrying telemetry bytes 24..31. The E3 BMS 115 (`CAN_RX_AuxCommand_Handler`) computes int16(data[4..5]) × 10000 / u16(data[6..7]),
  i.e. telemetry +0x1C × 10000 / (battery voltage × 10) → a current in mA. The frame exists only while the Control polls func 0x10;
  the micro has no dedicated sender.
- **0x305, 0x300, 0xCF** are standard 11-bit ids. The micro never transmits standard ids, so it is not their sender. Its standard-id
  filter (0x300/0x300) would pass 0x300–0x3FF to the class-4 dispatcher (low byte = function code): 0x305 → 0x05 (save + reboot),
  0x351 → 0x51, 0x356 → 0x56, 0x360 → 0x60 (battery calibration mode). A Pylontech-style exchange (0x305 request, BMS answers
  0x351/355/356/359/35C/360–363/36A/36B) on the same bus would therefore disturb the inverter; this exchange is most likely meant for
  an external port (third-party inverter or service tool). Whether the E3 BMS's CAN1 is wired to the internal bus only could not be
  decided from firmware.

### 4.2 RS485 / Modbus (micro's own register space)

`Modbus_HandleRegisterAccess` (0x08016E34) with `Modbus_ProcessRequest` (0x0800F03C) serves the micro's own RS485 Modbus slave
(address EEPROM 0x101). Register set and semantics are identical to D 116 (see the D document §4.2), with these E3 differences:

| Register | E3 | D 116 |
|---|---|---|
| 43133 / 43134 (`g_ratedPowerConfigA/B`, EEPROM 0x800/0x802) | 0..**3000** | 0..2500 |
| 45502 (factory test, work mode 0, value 1) | `Ctrl_SetPowerReferenceClamped(−2400.0)` | −2500.0 |
| 44000 / 44001 | value/10 → max SOC / cut-off SOC (800..1000 / 120..300) | same (`g_cfgMaxSocPct` / `g_cfgCutoffSocPct`) |

Unchanged: 40000 (calibration 0x55AA/0x55A1–A6), 41000/41001 (0x55AA), 41010 (exactly 800 → derating flag + 800 W, exactly 2500 →
clear), 41100, 41200, 42000 (0x55AA / 0x55BB reload work mode), 44002/44003 (≤ 2500 W, 44003 ≤ 800 W while derated), 44100 (grid
standard 0..4), 45000 (0x55EE / 0x55DD / 0x55FF), 45010–45019 (extra block, EEPROM 0x21), 45200/45300/45400–45402/45500–45502/45600/45601
(factory tests), 45603/45604, 46000 (0x5100 debug, 1234/2345 BMS update).
Quirk (also in D 116): writing 44000 stores value/10 into the max-SOC byte but persists that byte to EEPROM 0x201 (the cut-off SOC
slot).

These registers are reachable only on the micro's own RS485 port; the Control does not forward its Modbus registers to the micro —
Control registers (e.g. 44002/44003, 45000–45010) are translated into CAN commands (0x04/0x03, 0x50, 0x06, 0x52–0x58).

### 4.3 Debug shell (letter-shell, USART1 115200 Bd)

`shellCommandTable` @0x08019280, 51 entries (D: 53): 40 commands plus user entry and key bindings. Commands: `test`, `test_1`, `version`,
`log_data`, `log_list`, `set_n`, `sr_set` (directly `Ctrl_SetDeadTimeOffset`), `log_err`, `log_clear`, `log_open`, `prt`, `help_prt`,
`reset`, `pf`, `pf_set`, `pf_power`, `power_cfg`, `rtos_status`, `io_show`, `io_set`, `fan_set`, `set_power`, `update`, `update_bms`,
`update_bms2`, `bat_mode`, `reset_memory`, `backup_mode`, `par_mode`, `set_mode`, `qpr`, `pi`, `show_power`, `alarm`, `dac_high`,
`dac_low`, `set_sr`, `old_ate`, `help`, `clear`.
Differences to D: `vol_pi` and `show_pi` are gone; `qpr` now sets the PR controller (`Ctrl_SetPrControllerTarget`, "kp kr"),
`power_cfg` runs `Ctrl_HandleLlcPfcTestCommand(mode, value)` (any LLC/PFC/inverter test command), `test` writes the grid standard byte
to EEPROM 0x206 **without range check** and without reloading the limits (D: debug print only); the `set_mode` help adds "4 ATE aging".
The console UART is now initialised by `Uart_InitUsart1Console` (HAL UART, FIFO mode) where D calls an empty stub.

### 4.4 I2C EEPROM layout

As D 116 with these E3 values: 0x000 version flag (3 = valid), 0x001 id block (0x2F B, "AC02", model 0xD, marker 0x77), 0x021 extra
register block (20 B), 0x080 timestamp, 0x090 derating flag, 0x100–0x104 (standby latch, Modbus address, battery mode, …), 0x150 mode
display counter, 0x200–0x206 (max SOC, cut-off SOC, charge limit, discharge limit, grid standard), 0x300 block 0x7C B (backup flag,
0x374 parallel mode), **0x500 stats block 0x40 B** (D 0x30; now includes the energy counters), 0x800/0x802 rated power config,
0x900/0x901 debug level / boot flag, 0x950/0x952 EMS limits.
Factory defaults (`Cfg_LoadFromEeprom` 0x08006B2C, applied when the stored marker is < 108): "AC02", model 0xD, marker 0x77, max SOC
100, cut-off SOC 12, Modbus address 1, battery mode 1, **charge/discharge limit 3000 W**, grid standard 0. The limit block is loaded
without range check, so a 0 written via CAN 0x03/0x04 survives a restart.

## 5. Control and power section

### 5.1 PWM, sampling and controllers

- Inverter bridge PWM 30 kHz (HRTIM period 45333 at 1.36 GHz; D 48572 = 28 kHz), control sample time 33.33 µs (D 35.71 µs),
  LLC timer B start period 10625 = 128 kHz (D 10793 = 126 kHz), HRTIM base compare offset now a RAM value (`g_hrtimBaseOffset`,
  340, or 410 with the hardware-variant input; D constant 280).
- Grid current: PR controller (`Ctrl_PrControllerDesign` Ts, 6000.0, 50 Hz; kp 9.0, limits ±370; `Ctrl_PrControllerStepClamped`,
  only while the bridge outputs are enabled, `Inv_IsOutputEnabled`) instead of the D PI loop (3.0/0.2/±500); zero-crossing burst/skip
  for small references; overcurrent (|i| > 30 A) needs 3 consecutive samples (D immediate); HV bus > 530 V (570 V in LLC state 0x0E)
  checked separately.
- ADC: ADC2 has 4 ranks (D 5): two NTCs remain (CH5 → `g_tempInverterC`, CH12 → `g_tempLlcHeatsinkC`, both NTC table 2); new
  current/voltage scaling (grid current 0.0183 A/LSB, battery voltage (raw − 2048) × 0.0555 V).
- DAC comparator windows 2048 ± 1366 counts (D ± 1466).

### 5.2 Work modes (`g_workMode`, EEPROM 0x301)

Unchanged from D 116 (`ShellCmd_set_mode` is byte-identical): 0 default (CAN setpoint 0x01), 1 manual (CAN 0x57/0x58), 2 debug, 3
timed schedule, 4 battery calibration / ATE (CAN 0x60, Modbus 40000), 12 factory mode (Modbus 45000 = 0x55EE, CAN 0x51–0x56), 13
voltage-source mode. In work modes 3/4 `Bat_Mode3RoundRobinTimer` now commands ±3000 W (D ±2500 W) and the step-0 timeout is 7200 s
(D 9000 s).

### 5.3 Standby / sleep

The micro has no MCU low-power mode. `g_standbyLatch` is set by CAN 0x06 (Control register 45002)
or when the BMS reports its sleep flag (PF2 byte 7); while it is 1 `Ctrl_MainControlIsr` moves the power stage to the stop state 200
and telemetry byte 0 reads 0. `LED_TaskTick` clears it again when the BMS sleep flag is 0 and BMS frames are younger than 20 s
(EEPROM 0x100). CAN 0x05 saves the counters and reboots the micro. Identical to D 116.

### 5.4 Low SOC and forced charging

`Inv_GridConnectionSupervisor` (0x100003A8) sets the low-SOC stop flag (`g_lowSocStopFlag`) only after more than
15000 supervisor calls with SOC < 6 % (D: immediately at SOC < 6 %; the counter is not reset by a higher SOC), and releases it at
SOC ≥ cut-off − 1 (D: − 2); a second flag additionally trips when backup mode is on and SOC > 7 %. Together with the new CAN 0x0A
(cut-off SOC accepted from 6 %), these are the changes around the 6 % threshold; which one is the advertised "forced charging below
6 % SOC" fix could not be proven statically.

### 5.5 Energy counters (new)

`Ctrl_MainControlIsr` sums the positive and negative power every control tick; every 1000 ticks the mean goes to
`Energy_AddPositiveWs` / `Energy_AddNegativeWs` (0x200004A8/AC), and `Time_UpdateOperatingHoursAndFaultLog` rolls them per 3600 into
the u32 counters `g_energyPosWh` / `g_energyNegWh` (0x20003EEB/EEF), floored by CAN 0x17, sent in telemetry +0x30/+0x34 and saved in
the 0x40-byte stats block. Which physical flow (battery charge vs discharge) each counter holds is not proven. This matches the E3
Control changelog item "battery cumulative charge/discharge energy (VE3 only)".

### 5.6 Fan

`Fan_UpdateDutyByTemperature` (0x08016120; the D counterpart `Fan_TickManualOverrideTimer` only counts the manual override down): TIM1 duty 400 up to 40 °C hottest NTC, then
300 − 30 × (T − 40), clamped 0..400; a manual override (shell `fan_set` 3600 s, Modbus 100 s) suspends it.

### 5.7 Grid standards (`Grid_Protection_SetLimits` 0x08005290, Modbus 44100 / EEPROM 0x206)

Standards 2 and 3 unchanged; default (0/1): OV1 263 V (D 265), OV4 256 V (D 265), lower voltage bound 190 V (D 197.8); **standard 4 has
its own set** (51.0/48.0 Hz, 50.1/49.9 Hz, 207 V lower bound, timings 200, hold/reconnect 65000).

## 6. Protection and fault codes

Error block `g_protErrorWord1` 0x20001A28 (err1, Modbus 36100), `g_protErrorWord2` 0x20001A2C (err2, 36102), `g_protWarningWord1`
0x20001A30 (war1, 36000). An exhaustive scan of all set/clear sites gives exactly the D 116 set plus one E3 bit:

| Word / bit | Event | Condition | Function |
|---|---|---|---|
| err1 0 / 1 | 0x2C | grid Vrms high / low | `Ctrl_LlcVoltageRegulationAndProtection` |
| err1 2 / 3 | 0x2A | grid frequency high / low | same |
| err1 4 | 0x2D | grid peak voltage | same |
| err1 7 | 0x2E | waveform error | same |
| err1 20 | 0x33 (clear 0x18) | BMS communication lost | `Inv_GridConnectionSupervisor` |
| err1 21 | 0x3B (clear 0x0B) | BMS reports an error | same |
| err2 0 / 1 / 3 / 4 | 0x36 | hardware protection latch (bit 4 = collective) | `Prot_LatchHwFaultToErrorWord` |
| err2 5 | 0x3F | peak current | `Inv_CurrentLoopIsr` |
| err2 6 | 0x34 | HV bus voltage | same |
| err2 7 | 0x13 | state-machine shutdown: ≥ 1000 control ticks in the current state while flag [r4+0x0D] = 0 → fault log, state 200, 60 s retry | `Ctrl_MainControlIsr` (set at 0x10003ffe) |
| err2 8 | – | off-grid overload integrator | `Ctrl_RampDcBusReferenceWithFaultCheck` |
| err2 9 | 0x29 | state machine error | `Ctrl_MainControlIsr` |
| err2 11 | 0x35 | state change timeout | same |
| err2 16 | – | self-test failed | `main` |
| **war1 0** | – | **set while the grid-permit flag (telemetry +0x16) is 0, cleared when it is set again** | `Ctrl_LlcVoltageRegulationAndProtection` (**E3 only**) |

The fault log texts (`Log_PrintFaultEntry`) and event codes are identical to D 116 (see the D document §6). The Venus A bit names
(`VNSA_Micro_FW_v1211_Full_Analysis.md` §6) do **not** apply. Thresholds changed only where noted in §5 (grid standards, 3-sample
overcurrent, DC-bus ramp timer levels in `Ctrl_RampDcBusReferenceWithFaultCheck`).

## 7. OTA / update

Unchanged from D 116 (all OTA functions are byte-identical apart from addresses): YMODEM self-update over the debug UART
(`update`), CAN self-update via func 0xCE (`OTA_SelfUpdate_StateMachine`, CRC16-Modbus, staging at 0x08044000, descriptor at
0x08040000, size check against FLASHSIZE), BMS firmware relay over CAN (`BMS_FwUpdate_StateMachine`) or RS485
(`BMS_FwUpdateStateMachineTask`, `update_bms`/`update_bms2`). The image trailer tag "VNSEE3" is presumably what the bootloader or the
Control's OTA check uses to reject images for other models (bootloader not available). No signature, CRC only.

## 8. Security notes

Same as D 116: unauthenticated factory/test modes via the micro's Modbus (45000, 46000, 41010, 0x55AA codes) and via CAN class-4
commands (0x50–0x58, 0x60); letter-shell user with empty password; OTA without signature. New: shell `power_cfg` exposes every
power-stage test command, shell `test` writes an unchecked grid-standard byte to EEPROM. The micro has no network access of its own.

## 9. Differences

### 9.1 Versus D 115/116 and A 1211

| Topic | E3 119 | D 116 (D 115 same unless noted) | A 1211 |
|---|---|---|---|
| MCU / OS | STM32G474, FreeRTOS, CCM code | same | NS800RT Cortex-M7, bare metal |
| Identity | – | 73.5 % identical (D 115: 70.5 %) | 2.3 % |
| Version tag / trailer | 0x77 "AC02" / "VNSEE3" | 0x74 "VD01" / "VNSD" | 1211 / "VNSA" |
| PWM / control | 30 kHz, PR grid-current controller | 28 kHz, PI | 28 kHz, PI + QPR |
| PV | none | none (separate class-2 MPPT node) | 4 PV inputs on board |
| CAN class 4 | + 0x08, 0x0A, 0x14, 0x17; telemetry 56 B | telemetry 48 B | ignores 0x08/0x0A/0x14/0x17; also class 2 |
| 0xCC date/time | not handled | not handled | class 2 handled |
| Charge/discharge limit (CAN/Modbus) | ≤ 2500 W | ≤ 2500 W (D 115: 2200 W) | ≤ 1450 W |
| Factory default limit | **3000 W** | 2500 W | 1450 W |
| Internal ramp/derating constants | 3000.0 | 2500.0 (D 115: 2200.0) | – |
| Calibration / mode-3 power | ±3000 W | ±2500 W | ±2200 / ±1350 W |
| 43133/43134 limit | 3000 | 2500 (D 115: 2200) | no Modbus |
| 45502 test reference | −2400 W | −2500 W (D 115: −2200 W) | – |
| Warning word | bit 0 used | never written | never written |
| Cut-off SOC by CAN | 0x0A, 6..79 % | not possible | – |
| Energy counters | 2 new u32 (telemetry, CAN 0x17) | – | Wh counters in telemetry |
| Grid standard 4 | own limit set | falls into the default set | – |
| NTCs | 2 | 3 | 3 |
| Fan | automatic curve | manual only | – |
| Console | HAL UART USART1 init | empty init stub | LPUART |
| Update paths | YMODEM, CAN 0xCE, BMS relay | same | CAN 0xCE only |
| Packs | one BMS (owner: one fixed pack) | one BMS frame set | one BMS frame set |

### 9.2 Names corrected after the E3 comparison (applied to D 115/116 and E3 119)

These D 116 names were wrong; they are corrected in the Ghidra programs of D 115/116 and E3 119 uses the same names.

| Former D name | Name now (D 115/116 and E3 119) | Reason |
|---|---|---|
| `I2C_SetBufferPointer` (+ `_veneer`) | `HAL_GPIO_WritePin` (+ `_veneer`) | writes GPIOx_BSRR/BRR |
| `GPIO_ConfigPin_Output` / `_AF` / `_Input` | `GPIO_ConfigPin_Input` / `_ExtiRising` / `_OutputPP` | mode values 0 / 0x110000 / 1 |
| `HAL_UART_TxCpltCallback` | `HAL_UARTEx_RxEventCallback` | called with a size from the receive-to-idle path |
| `CAN_ProcessRxFilterTable` | `shellHandler` | letter-shell key handler |
| `Wdg_DecrementSafetyCounter` | D: `Fan_TickManualOverrideTimer`; E3: `Fan_UpdateDutyByTemperature` | decrements the manual fan override timer (low byte of `g_wdgSafetyCounter`); the E3 code additionally is the fan control |
| `Prot_MonitorSyncLossTimeout` | `Prot_MonitorHvBusLevelFlags` | HV-bus level flags (355 / 340 V and 480 / 520 V, 1000-call debounce) |

Globals (same object, same address correspondence as in D 116):

| Former D name | Name now | Meaning |
|---|---|---|
| `g_battPower` | `g_gridTiedPower` | grid-tied (AC port) power, telemetry +0x18 |
| `g_ctrlLoopDebugFloat_e00` | `g_batteryPowerAvg` | battery (DC) power, low-pass filtered, telemetry +0x1C |
| `g_emsPairedFlag` | `g_standbyLatch` | standby latch (EEPROM 0x100) |
| `g_gridProtCfgParam0` / `g_gridProtCfgParam1` | `g_cfgMaxSocPct` / `g_cfgCutoffSocPct` | max SOC / cut-off SOC (EEPROM 0x200 / 0x201) |
| `g_llcStatusFlag` | `g_gridPermitFlag` | grid-permit flag, telemetry +0x16 |
| `g_llcResonantGain` | `g_gridFrequencyHz` | grid frequency |
| `g_llcHvBusVoltage` | `g_pllFrequencyEstimate` | PLL frequency estimate |
| `g_gridSyncReadyFlag` | `g_lowSocStopFlag` | low-SOC stop flag |
| `g_syncLossFlagA` / `g_syncLossFlagB` | `g_hvBusBelow480Flag` / `g_hvBusAbove355Flag` | HV-bus level flags |
| `g_syncLossTimer_3b8` / `g_syncLossTimer_3bc` | `g_hvBusBelow480Timer` / `g_hvBusAbove355Timer` | their debounce counters |
| `g_canFilterTableBase` / `g_canFilterTableCount` / `g_canAddrMatchAccum` | `g_shellCommandListBase` / `g_shellCommandListCount` / `g_shellKeyValue` | letter-shell command list and key accumulator |
| `g_bmsFilterRxByte` / `g_bmsCanQueue` | `g_shellRxByte` / `g_shellRxQueue` | shell input byte and queue |

### 9.3 Re-check of non-functional registers (standing check 8)

There is no E3 register CSV yet, so `list_recheck_registers.py E3` is not applicable. The micro-related entries of the D list were
re-checked against E3 119: 30010 (work mode, conditional) — same micro behaviour; **36000 warning code — now functional on E3**
(war1 bit 0, §6); 38000–38014 (CAN 0x40–0x43) — still no sender/receiver in the micro; 45020–45024 (class-2 frames) — still filtered;
46500–46544 (CAN 0x30–0x3E) and 47001 (CAN 0x18) — still no handler; 41001 / 45013–45019 — the micro's own registers are still not
reachable through the Control.

## 10. Firmware quirks (reported, not fixed)

1. CAN 0x03/0x04 and Modbus 44002/44003 cap at 2500 W while the factory default is 3000 W — after a factory reset the stored limit is
   larger than any value that can be written back.
2. CAN 0x03/0x04 accept 0 W and persist it; the boot loader of the config block applies no range check.
3. Modbus 44000 persists the max-SOC byte to EEPROM 0x201 (cut-off SOC slot) — also in D 116.
4. CAN 0x08 value 1 (parallel mode 1) is not persisted (0 and 2 are).
5. Shell `test` writes an unchecked grid-standard byte to EEPROM 0x206 and does not reload the limits.
6. ADC oversampling setup writes the ADC2 oversampling fields into the hADC5 handle (copy-paste slip, also in D 116; medium confidence,
   from structure layout).
7. A standard-id frame in 0x300–0x3FF on the micro's bus would be executed as a class-4 command (§4.1.5).

## 11. Open points

1. Which code change implements the v119 note "forced charging below 6 % SOC" (§5.4).
2. Physical meaning of the two new energy counters (positive/negative power → charge/discharge?).
3. Sender of 0x305/0x300/0xCF and whether the E3 BMS's standard-id exchange is on a separate physical bus.
4. Meaning of the hardware-variant input (`g_hardwareVariant`, HRTIM offset 340/410, second digital output) — two board revisions?
5. Trailer word 0x44443E54 and the bootloader's model check (bootloader not available).
