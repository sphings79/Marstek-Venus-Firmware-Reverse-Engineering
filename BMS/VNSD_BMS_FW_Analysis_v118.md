# Marstek Venus D — BMS Firmware Analysis v118

> 🇩🇪 Deutsche Version: [VNSD_BMS_FW_Analyse_v118_de.md](VNSD_BMS_FW_Analyse_v118_de.md)

> **Obsolete (2026-09-30):** This analysis is based on a wrong base address (0x08000000; all flash addresses here are 0x2800 too low) and wrongly assumed an AT32F403A (it is actually a Nations N32G45x). The authoritative document is [VNSD_BMS_FW_v118_Full_Analysis.md](VNSD_BMS_FW_v118_Full_Analysis.md). Field measurements in this document may still be valid.

## `20260119100535e43806957.bin`

**Firmware:** BMS v118 (Ghidra import 2026-07-14)
**Analysis date:** 14.07.2026
**Method:** Static analysis (Ghidra + ReVa MCP), version tracking diff against v117.7
**Base document:** `VNSD_BMS_FW_Analysis_v117.7.md` (register/struct layout remains valid there unless described differently here)
**Status:** Memory map corrected, full analysis performed, diff to 117.7 evaluated, all 24 new functions decompiled manually, all 550 functions named in Ghidra (2026-07-15)

---

## Binary Fingerprint

Verified live from Ghidra (status 2026-07-15). A comparison with the other five analyzed
firmware images (Control 149.2/147, VNS 116/115, BMS 117.7) is in the project `README.md`.

| Property | Value |
|---|---|
| File | `20260119100535e43806957.bin` |
| Version | 118 (current) |
| Size | 106,496 B (0x1A000, 104 KB) |
| Architecture | ARM Cortex-M4F, Thumb-2, little-endian |
| Flash range | `0x08000000–0x08019FFF` |
| Initial SP | `0x2000CBA0` (~52 KB SRAM) |
| Reset handler | `0x08002A6D` |
| Functions | 550 / 550 named (100 % — 526 taken over from 117.7 via Ghidra diff markup transfer, 24 new ones named individually, see section 9) |
| Strings | 256 |
| Compiler | RVDS/Keil ARM |
| RTOS | FreeRTOS (`heap_4`, `ARM_CM4F` port) |
| Crypto | — (none) |
| Cell monitoring | KA495XX (BMIC) |
| Communication | CAN + RS485 |

---

## 1. Memory Map Correction

The Ghidra import had created a wrong default memory block:

| | Before (import) | After (corrected) |
|---|---|---|
| Name | `ram` | `flash` |
| Range | `0x00000000–0x00019FFF` | `0x08000000–0x08019FFF` |
| Permissions | R/W/X | R/X (not writable) |

Because of the wrong base address (0x0 instead of 0x08000000, as with 117.7), the preliminary analysis was incomplete: only 262 of the actual 550 functions and 497 of 1123 symbols were recognized (the vector table pointed to non-existent addresses). After moving the block to the correct flash base and a forced full analysis:

- **550 functions / 1123 symbols** (117.7 for comparison: 551 functions / 1128 symbols)
- Size unchanged: 106,496 B (0x1A000), identical to 117.7
- Flash base `0x08000000`, architecture ARM Cortex-M4F Thumb-2 LE — both identical to 117.7

---

## 2. Version Fingerprint

Confirmed via the per-pack struct builder function (SRAM `0x2000420A`, per-pack struct offset 0x54, `successor of PerPack_Struct_Builder` → `FUN_08009fdc`):

| Version | Hardcoded value | Encoding |
|---|---|---|
| 117.7 | `0x499` = 1177 | Digits of the version without the dot (117.7 → "1177") |
| **118** | `0x76` = **118** | Direct integer value (no ×10/decimal-place scheme) |

The encoding scheme has changed — 117.7 apparently followed "digits without dot" (117.7 → 1177), whereas v118 encodes the version number directly as an integer. Relevant for the HA integration if register 34010 (pack FW version) is used for display/evaluation: the raw value for v118 is **118**, not 1180.

---

## 3. Diff Result (Ghidra Version Tracking)

Correlators: symbol name, exact bytes, exact instructions, exact mnemonics, duplicate instructions, function reference.

| | 117.7 | 118 |
|---|---|---|
| Functions total | 551 | 550 |
| Identical (bytes) | 218 | 218 |
| Matched, but "changed" | 308 | 308 |
| Without counterpart | 25 | 24 |

The 308 "changed" hits are mostly **correlation noise**: same function, identical bytes (similarity 1.0), but marked as "changed" because one or more of their called callee functions belong to the unassigned group (the 24/25 below). No real logic changes were found in otherwise identical functions — all substantive changes are in the 24 new + 25 removed functions.

### 3.1 Removed Functions (25, present only in 117.7)

| Function (117.7) | Address | Presumed replacement in v118 |
|---|---|---|
| `System_Reset_And_Halt` | 0x08002198 | `FUN_08002198` (same address, extended — see 4.2) |
| `INIT_RS485_UART_Structs` | 0x0800298c | `FUN_080029c8` (DMA ring buffer init, see 4.4) |
| `USART2_UART4_Init` | 0x08002a24 | `FUN_08002a60` (see 4.4) |
| `Pack_MasterSelect_Monitor` | 0x08004c6c | New subsystem `FUN_08006fe4` + helpers (see 4.1) |
| `INIT_RTC_And_DateTime` | 0x08004fc4 | cannot be assigned unambiguously |
| `CAN_RX_Handler` | 0x080059dc | `FUN_08005a40` (confirmed, see 4.8 — identical magic ID table, 1 new row) |
| `APP_Calc_Pack_SOC_From_Voltage` | 0x08006b54 | presumably absorbed into the `FUN_0800a584` neighborhood |
| `bms_data_printf` | 0x0800702c | removed (debug function, see 3.2) |
| `RS485_Register_Write_Handler` | 0x08007d24 | `FUN_08007d90` (significantly extended, see 4.5) |
| `voltage_protection_check` | 0x08009548 | presumably core absorbed into `FUN_0800a584` (see 4.6) |
| `PerPack_Struct_Builder` | 0x08009f70 | `FUN_08009fdc` (see 2) |
| `Precharge_StateMachine` | 0x0800cb24 | `FUN_0800cb90` (part of the new pack mode subsystem, see 4.1) |
| `CAN_Send_Frame_Log` | 0x08012984 | removed (debug, see 3.2) |
| `Debug_Print_Battery_Status` | 0x08012c50 | removed (debug) |
| `CAN_RX_State_Flag_Set` | 0x08012d8c | removed (debug) |
| `Nop_Return_Stub` | 0x08012d90 | removed (debug) |
| `Clear_Debug_Print_Flag` | 0x08012d94 | removed (debug) |
| `Debug_Print_MOS_Command_Status` | 0x08012dbc | removed (debug) |
| `Debug_Print_Discharge_MOS_Cmd` | 0x08012dec | removed (debug) |
| `Debug_Print_Nop_Stub` | 0x08012e04 | removed (debug) |
| `Debug_TaskStack_Fragment` | 0x08012ff4 | removed (debug) |
| `Debug_Print_Device_UIDs` | 0x080130f4 | removed (debug) |
| `Debug_Print_EMS_ChgDsg_Flag` | 0x080131b0 | removed (debug) |
| `SOC_Algorithm_Orchestrator` | 0x08013520 | `FUN_08013590` (extended, see 4.7) |
| `soc_algorithm_dump` | 0x08013844 | removed (debug dump) |

### 3.2 Debug Code Cleanup

11 of the 25 removed functions (address range `0x08012984–0x080131B0`) are a contiguous cluster of CLI debug print helpers (`Debug_Print_*`, `CAN_Send_Frame_Log`, `CAN_RX_State_Flag_Set`, `Nop_Return_Stub`, `Clear_Debug_Print_Flag`). Together with `bms_data_printf` and `soc_algorithm_dump` these are **13 pure debug/logging functions** that were removed completely in v118 — at this point v118 looks like a cleaned-up release build. The flash space that was freed explains part of the address shift that made the 24 new functions possible further forward in the image.

---

## 4. New Functions (24) — Analysis

All 24 functions newly added in v118 were decompiled manually. They group into seven subsystems:

### 4.1 New Pack Mode / Master Election Subsystem (11 functions)

**Replaces:** `Pack_MasterSelect_Monitor` (117.7). In 117.7 the pack mode register (`DAT_200028CA`, values 0=Idle/1=Transitional/2=Active/3=Shutdown/4=Error, see 117.7 doc section 4.9) was a simple CAN-command-driven state. In v118 it has become a multi-stage state machine cascade that additionally actively controls the **precharge relay** during the mode transitions:

| Function | Role |
|---|---|
| `FUN_08006fe4` | Top-level dispatcher: `switch(DAT_200028CA)` — calls the matching sub-state machine depending on the pack mode |
| `FUN_080095e4` | Sub-state machine for mode **1 (Transitional)**: adopts new pack address (`DAT_200028DA`), switches the precharge relay (3-phase toggle, see below), sends CAN handoff messages |
| `FUN_08008d28` | Sub-state machine for mode **2 (Active)**: sets **own pack address tentatively to 1** (`DAT_200041B6 = 1`), also switches the precharge relay — looks like an address self-adoption/master claim procedure when becoming active |
| `FUN_0800cb90` | Sub-state machine for mode **3 (Shutdown)**, successor of `Precharge_StateMachine`: GPIO mode change on GPIOB (see below), periodic CAN broadcasts, ends with a complete forced reset via `FUN_08002198` (see 4.2) |
| `FUN_0800a9f4` | Runs in the idle state (mode 0): periodic relay toggle (every ~2000 ticks, toggle 20×) |
| `FUN_08006bbc` | Clamp/selection helper based on pack SOC (`per-pack struct offset 0x0E`) and a configurable parameter `DAT_200028E0` (settable via CAN **and** RS485, see 4.8) — SOC balance criterion for pack selection |
| `FUN_08006c04` | Determines the **minimum cell voltage across all active packs** (`per-pack struct offset 0x48 "Min Cell"`, active bitmask `DAT_200028DC`) |
| `FUN_08000e20` / `FUN_08000ea8` / `FUN_08000ef0` / `FUN_08000f20` / `FUN_08000f58` | CAN command builder+sender family (types 2/3/6/7 via the common frame builder `FUN_08000d70` + send function `FUN_080172d4`), used by the state machines above for handoff/broadcast messages |

**Trigger chain verified (cross-reference check on `DAT_200028CA`/`DAT_200028DA`):** The **external trigger remains unchanged** — `FUN_0800b770` (successor of `CAN_CMD_Pack_Activation`, still detected as "changed" via normal VT correlation, not part of the 24 new functions) as well as the `0x100101AA` branch in the new `CAN_RX_Handler` successor (`FUN_08005840`, see 4.8) still set the pack mode and target pack address from outside (Micro/inverter MCU). What is new is exclusively the **internal processing** of these states (relay coupling, address self-adoption) — the round-robin protocol to the Micro MCU itself (CAN CMD 3/6 from 117.7 doc section 4.9) has not changed.

**Correction to the initial analysis — no GPIO daisy chain, but precharge relay:** `0x40010C00` is **GPIOB** (confirmed via the clock enable table in `FUN_0800167c`: 0x40010800=GPIOA…0x40011400=GPIOD, ascending in 0x400 steps, classic STM32F1 layout). Bit `0x1000` (bit 12) on GPIOB is **the same pin that `FUN_0800aedc` (successor of `OTA_PreCharge_Relay_Sequence`) also switches** — so the "GPIO handshakes" in the new pack mode state machines are **precharge relay control**, not hardware address chain signaling. v118 apparently moves the relay control directly into the pack mode transitions (Transitional/Active/Idle) instead of handling it only in the separate precharge sequence. The `FUN_0800167c` call with value `0x20` in `FUN_0800cb90` (shutdown start) is not a pin drive, but a **GPIO mode reconfiguration** (call pattern identical to pin config code in `FUN_08002a60`/`FUN_08005028`) — which exact pin/mode that is remains open (see section 7).

### 4.2 Extended Forced Reset with Safety Check (1 function)

`FUN_08002198` (same address as `System_Reset_And_Halt` in 117.7, but 188 instead of 136 bytes) checks for the first time before the watchdog reset whether significant pack current is still flowing (SRAM `0x2000406C`). If so: an event with an RTC timestamp is logged (code **`0xfc02`**, helpers `FUN_08004f32`/`FUN_08006480`) before the reset is executed. Callers: `RS485_Dispatcher` (reset command 0x2A) and `FUN_0800cb90` (shutdown state machine, see 4.1).

### 4.3 New Event Code `0xfc02`

The same logging helper (`FUN_08004f32` = RTC timestamp encoder, `FUN_08006480` = event log writer) with code `0xfc02` is called at three places:
- `FUN_08002198` (forced reset while current is still flowing, see 4.2)
- `FUN_08013590` (SOC algorithm successor, 2×, see 4.7)
- `FUN_08013e34` (OCV limit check, see 4.7 — dead code)

All three places concern an "unexpected/critical state" — `0xfc02` is presumably a new generic fault event code for safety-relevant events that did not yet exist in 117.7.

> **Correction 2026-09-24:** `0xFC02` is not an event code, but the **EEPROM address**. The
> calls are `I2C_EEPROM_Write_WithMutex(0x20004A90, 0xFC02, 10)`; the signature
> (source, address, length) is evidenced in `SOC_Save_To_EEPROM`. What gets written is the current
> RTC timestamp (10 bytes, read with `RTC_Read_DateTime` into `0x20004A90`). In the orchestrator
> this happens on the transition to the idle state, i.e. on every SOH/cycle recalculation
> (`VNSD_BMS_SOH_Cycles_Discharge_Limit_v118.md` §6), not on a fault. The third caller is
> dead (§4.7).

### 4.4 RS485/UART Init Switched to DMA Ring Buffer (4 functions)

**Replaces:** `INIT_RS485_UART_Structs` + `USART2_UART4_Init`.

- `FUN_080029c8` / `FUN_08002a60` — initialize two UART channels (register bases `0x40004400`, `0x40004c00`), each with **1024-byte ring buffer descriptors** (buffer pointer, size `0x400`, callback addresses) instead of the previous simpler structure — indication of a switch from interrupt-based to DMA-buffered UART handling.
- `FUN_08002938` / `FUN_0800d7b0` — new blocking byte-by-byte UART send routine with timeout polling (TXE flag, 5000 cycles) plus wrapper.

### 4.5 Extended RS485 Register Write Handler (1 function, large)

`FUN_08007d90` (1010 bytes) replaces `RS485_Register_Write_Handler` (0x08007d24). Significantly more parameter block IDs than in 117.7 are served (`switch` over block ID with many `case` branches, among others block `0x10` → known current threshold registers `0x200049A6+`, block `1` → max/min cell NTC threshold block `0x20004960–0x20004976` — **this block already existed in 117.7**, see correction in 4.6). v118 thus makes **more configuration parameters writable via RS485** than 117.7.

### 4.6 New Cell Overvoltage/Undervoltage Protection with Temperature Compensation (1 function, large)

**Correction to the initial analysis** (verified by cross-check with 117.7 via subagent): `FUN_0800a584` (1048 bytes) is **not a temperature protection function, but a cell voltage protection function** — cell overvoltage/undervoltage protection (cell OVP/UVP). It compares `_DAT_200041FC`/`_DAT_200041FE` (documented "Max Cell"/"Min Cell" voltage in mV) against thresholds in the RS485 config block `0x20004996–0x200049B4` (117.7 doc section 11a, groups `0x40`/`0x10` — marked there as "no consumer found"). The thresholds themselves are **temperature-compensated**: depending on the sign of `_DAT_20004204` ("Ave NTC", average temperature) the function selects different OV limits (e.g. 3700mV for a warm pack, 3800/3850/3900mV for a cold pack) and uses them to update the warning/protect bitmasks (`DAT_200028C1`, `DAT_200028C5`) via debounce counters.

**Verified via subagent (cross-check against 117.7):** This function is **demonstrably not present** in 117.7 — the threshold constant 3800mV (`0xed8`) does not occur anywhere in the entire 117.7 firmware, and groups `0x40`/`0x10` indeed had no checker there. **This resolves a question that has been open since the 117.7 analysis** (117.7 doc section 11a: "no direct checker consumer found" for these groups): v118 adds the missing cell voltage protection logic for these two groups completely new.

**Bit assignment (addendum 2026-09-24):** The function writes to C1 (`0x200028C1`) and C5
(`0x200028C5`):

| Condition (warm / cold, Ave NTC < 0 °C = cold) | Bit | Consequence |
|---|---|---|
| Max Cell > 3700 / 3800 mV | C5 bit 3 | Warning |
| Max Cell > 3700 resp. 3800 / 3850 resp. 3900 mV | C1 bit 6 → **Protect1 bit 1** | SOC := 100 |
| Min Cell < 3100 resp. 3000 / 2750 resp. 2550 mV | C5 bit 2 | Warning |
| Min Cell < 2500 / 2300 mV | C1 bit 5 → **Protect1 bit 0** | SOC := 0 |
| Min Cell > 3100 / 2850 mV or charge current ≥ 1.0 A | — | Undervoltage reset |

Protect1 bit 0 is thus **undervoltage**, bit 1 **overvoltage**. In
`VNSD_BMS_FW_Analysis_v117.7.md` §5.2 the two were swapped (corrected there).

**Clarification on the max/min cell NTC block (`0x20004960–0x20004976`):** Contrary to the original assumption in this doc, this block is **not new** — `Voltage_Temp_Protection_Checker` already reads it identically in 117.7 (verified via subagent, the same comparisons line by line as in the v118 successor `FUN_08009924`). It is a pre-existing protection function that was already active in 117.7, not a v118 novelty.

**Side finding from the cross-check:** The RS485 groups `0x100`/`0x200` (117.7 doc: "presumably balance thresholds or SOC%", low confidence) are in reality **charge/discharge overcurrent stages**, consumed by the already named `Current_Protection_Checker` — present unchanged in both versions. Group `0x400` remains without an identifiable consumer in both versions.

### 4.7 SOC Algorithm Extensions (2 functions)

`FUN_08013590` (successor of `SOC_Algorithm_Orchestrator`, 388 instead of 346 bytes) calls the same SOC helpers as 117.7, but additionally two new event logging calls (code `0xfc02`, see 4.3) as well as a new function `FUN_08013e34`: It checks whether the most recently measured cell voltage limits lie outside the known OCV calibration table (`_DAT_0801B8D8`/`_DAT_0801B96E`, in the documented calibration flash range `0x0801B73C+`), triggers a recalibration when exceeded (`FUN_08016e24(..,2)`) and writes the event to a **new SOC event log** at SRAM `0x20004A90` (directly following the documented SOC SRAM map `0x20004A38–0x20004A78`).

> **Correction 2026-09-24 — `FUN_08013e34` (`SOC_OCV_Bounds_Check_And_Recalibrate`) is dead
> code.** The function only works if the byte `0x200029CA` has the value 1 (limit check,
> then `Full_Recalibration(…, 2)`) or 2 (unconditional recalibration). Afterwards it resets
> it to 0. **No code sets the byte to 1 or 2:**
>
> - The address appears exactly once as a literal in the whole image, in the function's own pool
>   (`0x08013EC8`); no further occurrence even unaligned.
> - Ghidra knows only the function's own accesses: one read access and two write accesses with 0.
> - Via the neighboring base addresses `0x200029C0`–`0x200029C8` no code accesses it
>   with a matching offset.
>
> After startup the byte is 0 (`.bss`), so the function always returns immediately. In 117.7
> it does not exist yet. It is presumably a prepared, never wired-up trigger, e.g.
> for a later command. The real recalibration still runs via the 1-h idle path in the
> orchestrator (`Full_Recalibration`, thresholds from `0x0801B8D4`). A SOC correction in the field must
> therefore **not** be attributed to this function.

`FUN_08009fdc` (successor of `PerPack_Struct_Builder`) is identical in content except for the changed version constant (see 2).

### 4.8 CAN_RX_Handler Successor + Other New Functions (3 functions)

- **`FUN_08005a40` = confirmed successor of `CAN_RX_Handler`** (117.7, 0x080059dc). No match could be found via `diff-function` (the VT correlator failed for this function, presumably because too many of its callees are themselves unnamed/shifted), but the magic ID dispatch table is **identical** to 117.7 (`0x100101AA…0x100501AA`, `0x180101AA/0x180102AA`, caller `FUN_08000cd4` = `APP_CAN_RX_Task_Process`, unchanged at the same address). Exactly **one row is new**: in the existing `0xfd00` branch (pack activation configuration message, only if the pack is in idle mode) a byte from the CAN frame (offset 0xD) is additionally adopted into `DAT_200028E0` — **the same SOC balance parameter that `FUN_08006bbc` uses in 4.1 for pack selection.** The `0x100201AA` branch (formerly `CAN_CMD_02_Handler`) now also writes `DAT_200028E0` directly and calls `FUN_0800875e` (logs the new value as event `0x1FE` if < 91). **Thus `DAT_200028E0` in v118 is settable both via CAN (from the Micro/inverter MCU as part of the pack activation) and via RS485** — a new configuration channel for the SOC balance criterion of the pack selection.
- `FUN_08005028` — peripheral init/self-test (register `0x40011400`) with error code logging: if the status register check fails (bit `0x80`), an error record with code `0x7E7` and several sub-values (`0xE/0xB/0x17/0x1D`) is written instead of a normal event log entry.
- `FUN_08003334` — tick-driven periodic alarm broadcast function: after ~5s (499 ticks) sends up to 10× a CAN alarm message (`FUN_08000ef0`, type 6), controlled among others via protect byte `DAT_200028C1`.

---

## 5. Follow-up Check of the "changed" Bucket (308 functions)

Sample over the entire similarity range (head + tail of the list sorted by similarity, 110 of 308 functions, incl. `includeBodyByteChanges`): **all** checked entries have `similarity 1.0`. Where `bodyBytesChanged=true` occurs, the cause is solely that call/literal addresses point to shifted callees (correlators "Exact/Duplicate Instructions Match") — no real logic change. Result: **no hidden logic changes** in the 308 matched "changed" functions; all substantive differences are entirely in the already analyzed 24 new + 25 removed functions.

## 6. All Three Open Points from the Initial Analysis Resolved

- **GPIO `0x40010C00`:** is **GPIOB** (confirmed via the clock enable assignment in `FUN_0800167c`). Bit `0x1000` = **precharge relay** (identical pin as in `FUN_0800aedc`/`OTA_PreCharge_Relay_Sequence`) — not an address chain but relay control, now moved directly into the pack mode transitions (see 4.1, corrected). The bit `0x20` call in `FUN_0800cb90` (shutdown entry) reconfigures GPIOB pin 5 to **analog input** (fully derived, no live test needed, see section 7).
- **Threshold block `0x20004960–0x20004976`:** fully assigned field by field (max/min cell NTC protection stages, warning + protect bit, each SET+CLEAR+debounce) — **correction after cross-check with 117.7 (see 4.6): this block is not new**, it was already consumed identically in 117.7. What is actually new in v118 is instead `FUN_0800a584` (cell OVP/UVP, see 4.6) — that was a mix-up in the initial analysis.
- **`CAN_RX_Handler`:** successor identified = `FUN_08005a40` (identical magic ID table), with exactly one substantive addition: new CAN configuration channel for the SOC balance parameter `DAT_200028E0`, see 4.8.
- `FUN_08000f58` additionally decompiled: 5th member of the CAN builder family from 4.1, no new insight.

## 7. Addendum: GPIO mechanism + trigger chain

- **`FUN_08006918`** decompiled: generic STM32 GPIO helper that writes the CRL/CRH mode bits (4-bit CNF+MODE per pin) and optionally the initial state via BSRR/BRR — used throughout the firmware image for practically every pin configuration (UART, SPI, relays, …), nothing BMS-118-specific.
- `FUN_0800167c(GPIOB, 0x20)` in `FUN_0800cb90` (shutdown entry) uses it to call **pin 5 of GPIOB** (bit mask `0x20`) with a new mode configuration — **now determined exactly, no register trace needed:** `FUN_0800167c` builds the configuration word passed to `FUN_08006918` from `CONCAT22(<upper halfword of param_4>, param_2)` and then unconditionally masks the result with `& 0xFFFFFF`. As a result, **byte 3 of the configuration word is always 0**, regardless of what is in the parameters `param_3`/`param_4` that are not passed explicitly — and byte 3 is exactly what `FUN_08006918` uses to derive the 4-bit CNF+MODE value for the pin. CNF+MODE = `0000` corresponds to **analog input** in the STM32 CRL/CRH scheme (no push-pull/open-drain output, no BSRR/BRR write access, since byte 3 is used for that as well). **Result: GPIOB pin 5 is put into the high-impedance analog-input state on shutdown entry** — the relay/driver peripheral on this pin is thereby actively disabled/released, rather than merely having its level toggled.
- **Trigger chain confirmed:** the external triggers for pack mode changes remain `FUN_0800b770` (the successor of `CAN_CMD_Pack_Activation`) and the `0x100101AA` CAN command (`FUN_08005840`) — both unchanged compared with 117.7 (see 4.1). The 24 new functions are purely internal execution, with no outward protocol change.

## 8. Remaining open items

- Addendum 2026-09-22: the analysis of the limit aggregation added new open items (`FUN_08004148`, `PackSelect_SOCBalance_Clamp`, error path of the pack selection) — see §11.7. Resolved on 2026-09-24 in `VNSD_BMS_Pack_Rotation_Taper_v118.md`.
- Only a live-test recommendation remains: the address self-adoption in `FUN_08008d28` (`DAT_200041B6 = 1`) and the new precharge relay control in the pack mode transitions should be verified on a real multi-pack setup before any HA integration assumptions are derived from them. Statically/with the decompiler, everything from the 117.7→118 diff is thereby exhausted.

## 9. Ghidra renaming applied (2026-07-15)

Until 2026-07-15 the analysis above was only documented, but never written back as Ghidra symbol names (the function counter stood at 0/550). Done retroactively in two steps:

1. **526 matched functions** (218 identical + 308 "changed"/correlation noise, see section 3) were taken over automatically via `diff-transfer-markup` from the existing 117.7→118 diff session (confidence threshold 0.9999 — all matches had similarity 1.0).
2. **24 new functions** (section 4) were named individually via Ghidra script, according to the analysis documented above:

| Address | Name | Role |
|---|---|---|
| `0x08006fe4` | `PackMode_Dispatcher` | Top-level dispatcher of the pack mode cascade |
| `0x080095e4` | `PackMode_Transitional_Handler` | Sub-state machine, mode 1 |
| `0x08008d28` | `PackMode_Active_MasterClaim` | Sub-state machine, mode 2 |
| `0x0800cb90` | `PackMode_Shutdown_Handler` | Sub-state machine, mode 3, successor of `Precharge_StateMachine` |
| `0x0800a9f4` | `PackMode_Idle_RelayToggle` | Idle state, periodic relay toggle |
| `0x08006bbc` | `PackSelect_SOCBalance_Clamp` | SOC balance clamp/selection helper |
| `0x08006c04` | `Pack_MinCellVoltage_AcrossActive` | Minimum cell voltage across active packs |
| `0x08000e20` | `CAN_PackCmd_Build_Type2` | CAN command builder, type 2 |
| `0x08000ea8` | `CAN_PackCmd_Build_Type3` | CAN command builder, type 3 |
| `0x08000ef0` | `CAN_PackCmd_Build_Type6` | CAN command builder, type 6 |
| `0x08000f20` | `CAN_PackCmd_Build_Type7` | CAN command builder, type 7 |
| `0x08002198` | `System_Reset_And_Halt_WithCurrentCheck` | extended forced reset with pack current check (see 4.2) |
| `0x080029c8` | `RS485_USART2_Init_DMA_RingBuffer` | UART init with 1024B ring buffer (0x40004400) |
| `0x08002a60` | `RS485_UART4_Init_DMA_RingBuffer` | UART init with 1024B ring buffer (0x40004c00) |
| `0x08002938` | `RS485_UART_Send_Byte_Blocking` | blocking byte send with timeout polling |
| `0x0800d7b0` | `RS485_UART_Send_Byte_Blocking_Wrapper` | wrapper for it |
| `0x08007d90` | `RS485_Register_Write_Handler` | extended successor (more block IDs, see 4.5) |
| `0x0800a584` | `Cell_OVP_UVP_TempCompensated_Protection` | new cell voltage protection logic (see 4.6) |
| `0x08013590` | `SOC_Algorithm_Orchestrator` | successor, same role + event logging (see 4.7) |
| `0x08013e34` | `SOC_OCV_Bounds_Check_And_Recalibrate` | OCV limit check + recalibration trigger — **dead code**, the trigger flag `0x200029CA` is never set (see correction in 4.7) |
| `0x08009fdc` | `PerPack_Struct_Builder` | functionally identical successor (only a version constant changed) |
| `0x08005a40` | `CAN_RX_Handler` | manually confirmed successor (VT correlator found no match) |
| `0x08005028` | `GPIOD_Peripheral_SelfTest_Init` | peripheral init/self-test with error code logging |
| `0x08003334` | `CAN_Periodic_Alarm_Broadcast` | tick-driven periodic CAN alarm message |

Verified: 550/550 functions uniquely named, no duplicate names (Ghidra script check after application).

### 9.1 Second pass (2026-09-22) — multi-pack limit subsystem

Nine functions that had been named generically until then were renamed after the analysis in §11:

| Address | before | after |
|---|---|---|
| `0x080093fc` | `FUN_080093fc` | `Master_Build_ChargeCurrentLimit` |
| `0x080094c4` | `FUN_080094c4` | `Master_Build_DischargeCurrentLimit` |
| `0x08003f40` | `FUN_08003f40` | `PackHandoff_StateMachine` |
| `0x080049d8` | `FUN_080049d8` | `PackSelect_Next_AboveSocThreshold` |
| `0x08003ee0` | `FUN_08003ee0` | `Master_RunState_Dispatcher` |
| `0x080040d8` | `FUN_080040d8` | `Master_MultiPack_Tick` |
| `0x080048f8` | `FUN_080048f8` | `PackSelect_Tick` |
| `0x0800345c` | `FUN_0800345c` | `PackFault_Scan_SetFallbackFlag` |
| `0x08003ce8` | `FUN_08003ce8` | `BMS_Role_Dispatcher` |

`FUN_08004148` was deliberately **not** renamed (see §11.7). This was made up for on 2026-09-24,
together with 25 further names, table in `VNSD_BMS_Pack_Rotation_Taper_v118.md` §9. The decompilation export under
`Dekompilate/VNSD_BMS_118/` (local, not in the repo) was regenerated afterwards (2026-09-23, 649 functions, of which 560
named) and carries the new names.

---

## 10. CAN transmit path to the Control — deriving the Modbus pack registers (2026-08-14)

In the course of resolving the open Modbus descriptor registers (Control v150), the
**sender side** in this BMS FW was evaluated. It is the source of the Modbus registers 34xxx and
32xxx and proves their meaning independently of the Control image.

### 10.1 Aggregate telemetry: PF1–PF4 = PGN 1801–1804

Four send functions map exactly to the four PGNs that the Control stores in its
aggregate struct (`0x20014F8E`) and serves via Modbus:

| BMS function | Address | PGN | Control Modbus register |
|---|---|---|---|
| `CAN_TX_PF1_PackMeasurements` | `0x080053D0` | 1801 | 32100 bat_volt (10 mV), 32101 bat_curr (100 mA), 32108 bat_temp, 32104 soc |
| `CAN_TX_PF2_Capacity` | `0x0800543C` | 1802 | 32105 bat_energy, 32109 pack_count, 32110 online_mask, 32111 work_bat_idx |
| `CAN_TX_PF3_ChargeDischargeLimits` | `0x080054D8` | 1803 | 35110 chrg_volt, 32106 chrg_curr, 32107 dischrg_curr |
| `CAN_TX_PF4_ProtectWarnings` | `0x08005638` | 1804 | fault/warning bits, 32113 factory_test, 32112 lock_flag |

Cross-check: the Control FW outputs the same fields with the debug strings
`pgn_1801_info.bat_volt(10mv)`, `pgn_1802_info.bat_total_nb`, `pgn_1803_info.chrg_volt(0.1V)`,
`pgn_1804_info.bat_err1`. All 14 registers of the aggregate struct are thereby
**evidenced on both sides**.

**Unit correction:** `bat_volt` is sent in **10 mV** (not 0.1 V) and `bat_curr` in
**100 mA**. Live cross-check: register 32100 = 5372 → 53.72 V.

### 10.2 Per-pack telemetry: `CAN_TX_PerPack_10Msgs`

`CAN_TX_PerPack_10Msgs` (`0x08005B58`, 1868 bytes) sends **10 frames per pack**, called from
`CAN_TX_Orchestrator` in a loop over all active packs (max. 9).

- CAN ID scheme: `0x18[Grp]AA[PackID]`
- Source: BMS-internal pack array starting at approx. `0x200041C4`, **stride 0x60**
- The Control stores the data in its own pack struct starting at `0x20014FB4` —
  **likewise stride 0x60**

| CAN group | Content | Control pack offset | Modbus register |
|---|---|---|---|
| 0x20–0x23 | Overview (volt, curr, soc, cycle, limits, protect, version) | +0x00 … +0x1E | 34x00–34x10 |
| 0x30–0x33 | **16 cell voltages** (4 frames × 4) | **+0x20 … +0x3F** | 34x18–34x33 ✓ |
| 0x40 | **4 NTC temperatures** | **+0x40 … +0x47** | 34x13–34x16 ✓ |
| 0x41 | **Ave / MOS / ENV NTC** | **+0x4A, +0x4C, +0x4E** | 34x11, 34x12, 34x17 |

The two rows marked with ✓ match the Control's descriptor table **byte for byte**
(there 16 and 4 elements at exactly these offsets, respectively). From this it follows with certainty that
**34x11 / 34x12 / 34x17 are temperature values** — which is also supported by `BLE/VNSD_BLE_Modbus_CrossReference.md`
(listed there as `temperature1-4` / `mosfetTemperature`).

### 10.3 Field names from the debug strings

The BMS FW contains a plaintext block of the pack overview at `0x0800739F`–`0x080074AC`
(no longer referenced in the release build, but present in the image):

```
Addr / Online / Version / Bat Volt / Bat Curr / Bat Soc / Cyc Cnt
Chg Mos + Dsg Mos / Max NTC + Min NTC + Ave NTC / MOS NTC + ENV NTC
Max Cell + Min Cell / Protect1 + Protect2
```

From this follows for the pack registers that are open in the Control:

| Register | Pack offset | Derivation | Confidence |
|---|---|---|---|
| 34x03 | +0x06 | `Cyc Cnt` (cycle count) | **medium** — independently confirmed by BLE offset 0x18 → 34003 |
| 34x04 | +0x0E | `Chg Mos` / `Dsg Mos` (MOSFET status) | hypothesis |
| 34x07 | +0x18 | `Max NTC` | hypothesis |
| 34x08 | +0x1A | `Min NTC` | hypothesis |
| 34x09 | +0x1C | `Protect1` (protection bit mask) | hypothesis |

The four hypotheses are derived from the field order, but are **not verified byte for byte**
— the frame layout of groups 0x20–0x23 was not traced back byte by byte.

### 10.4 Protection codes without plaintext

`Protect1` / `Protect2` are output by the BMS only numerically
(`Protect1:%d  Protect2:%d`). A full-text search over the image for protection terms
(`ovp|uvp|ocp|otp|scp|over|under|fault|protect`) finds **no bit-to-text table** — only
`error lock on/set/reset`. To obtain plaintext, the set locations of the individual bits would have to be derived from their
trigger conditions (a separate work package, cf.
`Modbus_RS485_TCP/VNSD_Descriptor_Open_Registers_Ghidra_Finding.md` section 9).

### 10.5 Side note on the Control FW

The Control FW had named `0x08032F20` `Register_PackDescriptor`. The function
does not register anything, but builds the 29-bit CAN arbitration ID — the direct counterpart of
`CAN_Build_Arbitration_ID` (`0x08000D70`) in this BMS FW. It was renamed accordingly.

---

## 11. Charge/discharge limit aggregation and pack handoff (2026-09-22)

Subject: the aggregation chain from the charge current limit of the individual pack to the CAN value that the
micro inverter obeys, and the switchover between two packs. All functions below were decompiled from
v118, the pointers resolved from the image (not guessed).

> **Continuation 2026-09-24:** pack selection, discharge lower limit, special modes and the taper steps
> measured in the field are in `VNSD_BMS_Pack_Rotation_Taper_v118.md`. The open items from §11.7
> are also resolved there, and §11.4 is corrected.

### 11.1 Chain: pack limit → CAN PF=3

```
Max_Charge_Current_Calc (0x0800fe60)          per pack: own SOC × own temperature
   └─ calibration matrix (pointer 0x0801B9BC) → result in SOC SRAM map 0x20004A38 + 0x30
        │
Protect_Get_MaxChgCurrent_Div10 (0x0800fa20)  reads 0x20004A38 + 0x30
        │
PerPack_Struct_Builder (0x08009fdc)           writes per-pack struct offset 0x50
        │                                     (slave → master via CAN msg 1 / RS485 0x29)
Master_Build_ChargeCurrentLimit (0x080093fc)  MINIMUM over two pack indices, cap 500
        │
0x20004A18 → CAN_TX_PF3_ChargeDischargeLimits → charge_current_limit to the micro MCU
```

### 11.2 `Master_Build_ChargeCurrentLimit` (0x080093fc, formerly `FUN_080093fc`)

```c
if (soc_aggregat /*0x20004A14*/ < 1000) {
  if (master_run_state /*0x20002A0C*/ == 2) {
    if (handoff_single /*0x20002A0E*/ == 0) {          // handoff active: two packs relevant
      a = pack[DAT_20002A11].max_chg;                  // struct offset 0x50
      b = pack[DAT_20002A12].max_chg;
      agg = (a < b) ? a : b;                           // MINIMUM, never a sum
    } else {
      agg = pack[DAT_20002A11].max_chg;                // active pack only
    }
    if (agg > 500) agg = 500;                          // cap 50.0 A
  } else agg = 0;
} else agg = 0;                                        // from 100.0 % SOC: 0
*(u16*)0x20004A18 = agg;
*(u16*)0x20004A16 = 0x240;                             // charge_voltage_limit = 57.6 V (constant)
```

**Key finding: there is no summation across packs anywhere in the path.** The value the
inverter receives is always the minimum or the single value of *one* pack. Even if several pack MOSFETs
were closed at the same time, the reported total charge power would stay at the
level of the weakest pack involved. A single pack in the top-of-charge taper thus caps
the whole device.

Counterpart: `Master_Build_DischargeCurrentLimit` (0x080094c4, formerly `FUN_080094c4`) reads
struct offset 0x52 **without** forming a minimum, only from the active pack `DAT_20002A11`, cap likewise
500, target `0x20004A1C`; `discharge_voltage_limit` (`0x20004A1A`) is constant `0x1E0` = 48.0 V.

### 11.3 Scaling of the matrix

`Max_Charge_Current_Calc` returns `matrix value × 50.0 / 10.0` as soon as the matrix value is < 101,
otherwise the fixed value 500. The matrix entry is thus a **percentage relative to the 50 A
full scale**, and the result has the unit 0.1 A. Special cases: SOC exactly 100 % → 0,
invalid SOC/temperature value (`-1` or `0x7FFF`) → `0xFFFF`.

> **The matrices themselves cannot be read from the OTA image.** The pointers point to
> `0x0801B97E` (temperature index), `0x0801B9A2` (SOC zone thresholds) and `0x0801B9BC`
> (charge current matrix, stride 0x18 = 12 zones × u16) — all beyond the image end
> `0x08019FFF`. They reside in a separate calibration flash partition and can only be obtained via an
> SWD/JTAG dump (see `Methodik_und_Meta/VNSD_SWD_JTAG_Dump_Plan.md`). The addresses named in the
> 117.7 documentation (`0x0801B884` ff.) are likewise outside and
> differ by 0x138 — whether the layout has changed or only the pointer, cannot be decided
> without a dump.

Steps measured in the field (22.09.2026: 90 → 95 % ≈ 23.2 A, 95 → 100 % ≈ 9.1 A) and the way
to obtain the table values without a dump via register 32106:
`VNSD_BMS_Pack_Rotation_Taper_v118.md` §8.

### 11.4 Pack handoff: briefly two packs on the bus

`PackHandoff_StateMachine` (0x08003f40, formerly `FUN_08003f40`), called with `param_1` =
`0x20002A11` (active pack) and `param_2` = `0x20002A12` (target pack):

| `DAT_20002A45` | Action |
|---|---|
| — | `*param_1 == *param_2` → `DAT_20002A0E = 1` (no handoff, limit only from the active pack) |
| 0 | `DAT_20002A0E = 0` (from here on the minimum formation applies), wait counter up to 99 |
| 1 | `DAT_20002A17 = 0`, waits until no pack is charging any more |
| 2 | **`DAT_20002A18 \|= 1 << new`** — the Dsg MOS target mask now contains **old and new pack** |
| 3 | `DAT_20002A18 = 1 << new` — from here on exclusively the new pack |
| 4 | `DAT_20002A17 = 1 << new`, completion when the pack is charging or SOC > 98 % |
| 5 | `*param_1 = new`, update characteristic values, back to step 0 |

The `|=` in step 2 is **make-before-break**: the hardware tolerates two parallel packs
on the DC bus, protected by the precharge relay (see §4.1). **Correction 2026-09-24:**
`0x20002A18` is the target mask of the **discharge** MOSFETs, `0x20002A17` that of the charge MOSFETs
(reconciliation against `Pack_Get_ChgMosOn_Mask`/`Pack_Get_DsgMosOn_Mask`). The two
packs are therefore parallel only on the discharge side; during the handoff, charging never goes over two packs. Run times of the
steps: `VNSD_BMS_Pack_Rotation_Taper_v118.md` §6. Parallel operation is thus
not electrically taboo — it is just limited to the switchover window, and exactly for this
window the minimum branch in §11.2 exists.

In continuous operation it stays with one pack. Confirmed by live logs
(`Modbus_RS485_TCP/Scan_Logs/laden_lang.csv` (local, raw data), register 34x04): during 2310 W of charging
exactly one `packN_mos_status` is always at 3, all others at 0; pack 1 charges 57.6 → 70.1 %,
then pack 2 takes over.

### 11.5 Surrounding state machine

| Function (newly named) | Address | Role |
|---|---|---|
| `BMS_Role_Dispatcher` | 0x08003ce8 | Branches on the own pack address (`0x200041B6`): `0xFF` = unconfigured, `1` + `pack_count == 0` = single-pack master (sets `a11 = a12 = 0`, `0x20002A0C = 2`, `0x20002A0E = 1`), `1` + `pack_count != 0` = multi-pack master (`0x20002A0C = 0`, `0x20002A0E = 0`), otherwise slave |
| `Master_MultiPack_Tick` | 0x080040d8 | Calls `Master_RunState_Dispatcher`, `FUN_080043c4`, `Master_Check_AllPacksOnline` |
| `Master_RunState_Dispatcher` | 0x08003ee0 | Three-stage via `0x20002A0C`: 0 → 1 → 2; in state 2, `PackHandoff_Tick` (contains `PackHandoff_StateMachine`), `PackSelect_Tick` and `PackRecovery_Request_Check` run. If `0x20002A1E` is set, `PackRecovery_LowCell_Charge_SM` runs instead — **not OTA**, but the recovery charging of deeply discharged packs (until 2026-09-24 wrongly named `BMS_OTA_Upgrade_StateMachine`, see `VNSD_BMS_Pack_Rotation_Taper_v118.md` §7.4) |
| `PackSelect_Tick` | 0x080048f8 | Calls `PackFault_Build_ExcludeMask` and `PackFault_Scan_SetFallbackFlag`; in a clean state `PackSelect_Next_AboveSocThreshold`, otherwise the error path `PackSelect_FaultPath_AllOff` |
| `PackFault_Scan_SetFallbackFlag` | 0x0800345c | Sets `DAT_20002A15 = 1` if any pack reports Protect1 bits `0xE0C` or Protect2 bit `0x200` |
| `PackSelect_Next_AboveSocThreshold` | 0x080049d8 | Selects `DAT_20002A12` = online, fault-free pack with `soc% > (100 − DAT_200028E0)`; if none is found, `DAT_20002A15 = 1` is set |

`DAT_200028E0` is the parameter described in §4.8, which in v118 can be set both via CAN and via
RS485. **Correction 2026-09-24:** it is the **depth of discharge (DoD)**, default 90, not an
SOC balance parameter. The lower limit of the pack selection is `100 − DoD`
(`VNSD_BMS_Pack_Rotation_Taper_v118.md` §3.2).

### 11.6 SRAM variables of this subsystem

| Address | Meaning (verified via the consumers above) |
|---|---|
| `0x20002A0C` | Master run state; limits valid only when `== 2` |
| `0x20002A0E` | 0 = handoff active (minimum over two packs), 1 = active pack only |
| `0x20002A11` | Index of the active pack |
| `0x20002A12` | Index of the handoff target/candidate pack |
| `0x20002A15` | Error/fallback flag of the pack selection |
| `0x20002A17` | Target mask, charge MOSFETs (corrected 2026-09-24) |
| `0x20002A18` | Target mask, discharge MOSFETs — in handoff step 2 with two bits set (corrected 2026-09-24) |
| `0x20002A45` | Step counter of the `PackHandoff_StateMachine` |
| `0x20004A16` / `0x20004A18` | Aggregate: charge_voltage_limit (const. 57.6 V) / charge_current_limit |
| `0x20004A1A` / `0x20004A1C` | Aggregate: discharge_voltage_limit (const. 48.0 V) / discharge_current_limit |

### 11.7 Open items

> **Status 2026-09-24:** The first three items are resolved in `VNSD_BMS_Pack_Rotation_Taper_v118.md`:
> `FUN_08004148` = `PackSelect_Startup_Closest50` (§7.5), `PackSelect_SOCBalance_Clamp`
> = `PackSelect_DischargeStepThreshold` (§5.2), `FUN_0800364c`/`FUN_08004dc4`/`FUN_080034ac`
> (§7.4, §7.6). Only `FUN_080043c4` remains open. The original status is below.

- **`FUN_08004148`** (caller `FUN_08004394`) also writes `DAT_20002A11`/`DAT_20002A12`
  and selects the pack with the smallest distance to 50 % SOC among all packs > 20 % SOC —
  but afterwards checks for SOC > 98 %. This combination is not conclusively explained; the function
  was **deliberately not renamed**. Presumably a startup/calibration path, not the
  normal rotation cycle. To be clarified.
- **`PackSelect_SOCBalance_Clamp`** (0x08006bbc) and the `DAT_20002A22`/
  `DAT_20002A23` fed by it have not yet been traced back to their consumers; the exact
  semantics of `DAT_200028E0` (threshold vs. window) is therefore open.
- **`FUN_080043c4`**, **`FUN_0800364c`**, **`FUN_08004dc4`**, **`FUN_080034ac`** (error path of the
  pack selection) have not been analyzed.
- The struct offsets 0x50/0x52/0x5A have been added to `VNSD_BMS_FW_Analysis_v117.7.md` §6.0
  (addendum of 2026-09-22) — the derivation of the assignment is also there.
