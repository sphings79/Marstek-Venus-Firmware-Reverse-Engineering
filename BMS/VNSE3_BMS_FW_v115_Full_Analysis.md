# BMS-FW V115 (Venus E 3.0) — Full analysis

> 🇩🇪 Deutsche Version: [VNSE3_BMS_FW_v115_Vollanalyse_de.md](VNSE3_BMS_FW_v115_Vollanalyse_de.md)

**Status:** 2026-10-02.

**Reference images:** Venus D BMS 118 (`VNSD_BMS_FW_v118_Full_Analysis.md`), Venus A BMS 110.5
(`VNSA_BMS_FW_v1105_Full_Analysis.md`) · **Decompilation/Listing:** `Dekompilate/VNSE3_BMS_115/` (local only, not in the
repo) (653 functions, `listing/` with flash disassembly + `globals_and_registers.txt`) · **Companion:**
`Control_FW/VNSE3_Control_FW_v151_Full_Analysis.md`.

## 0. Summary

- Image base **0x08002800**, image size 0x14800 bytes (83,968 B), MCU **Nations N32G45x** (Cortex-M4F) — same MCU,
  toolchain (Keil ARMCC), RTOS (FreeRTOS, here with `configASSERT` strings and only 5 priorities) and shell (letter-shell,
  user `BLUE`, empty password) as the Venus D/A BMS. Version string `Version: 115.0`, build date `Jun 29 2026 10:55:09`.
- **Code base:** the E3 BMS is **a separate code base**, not a sibling build of the D BMS. Only 17.7 % of the image
  bytes lie in segments identical to D 118 (pointer/BL-masked; A 1105: 16.9 %): C runtime, printf/dtoa, NTC tables,
  parts of FreeRTOS. A fuzzy function matcher paired 222 of the 653 functions with D 118 (6 more only with A 1105: RTC
  helpers, shell write callback, password check); all of them were reviewed against the E3 code. The remaining ~430
  functions were named from the E3 code.
- **Battery front end:** **KA495XX** (same driver family as D, SPI1 with CRC-8 0xD5), **16 cells**, 4 cell NTCs, plus a
  MOS NTC and an environment NTC on the MCU ADC.
- **One pack.** The owner states (hardware fact, not derived from the firmware) that the Venus E 3.0 has exactly one built-in
  pack and cannot be extended. The firmware is consistent with that: the CAN capacity frame `0x1802AA01` is a
  **constant** (1 pack, online mask 1, 5120 Wh), the per-pack burst is only ever sent for pack 1, and the remnants of a
  multi-pack/slave bus logic are unreachable (§6). The per-pack sender keeps the D bound `pack_id != 0 && pack_id < 10`.
- **Different CAN protocol set:** besides the D-style extended frames (`0x18xxAAyy`, `0x100x01AA`) the E3 BMS answers a
  **Pylontech-style 11-bit request `0x305`** with `0x351/0x355/0x356/0x359/0x35C`, `0x360–0x363` (16 cell voltages) and
  `0x36A/0x36B` (temperatures), takes MOSFET commands from `0xCF`, a shutdown from `0x300` and a current value from
  extended ID `0x04410110`. There is **no** pack coding, master/slave role, error lock or RS485 inter-pack protocol.
- **Six battery parameter sets ("profiles" 0..5):** SOH curve, OCV table, capacity-vs-temperature table and the charge
  and discharge current matrices are selected per profile (CAN `0x100601AA`, shell `set_table`) and stored in the EEPROM
  configuration block (§7).
- **Changelog items found in the code** (manufacturer text: "1. Optimised the start-up check when SOC reads 100 % or
  0 %; 2. Deep-discharge recovery together with the inverter; 3. Fixed a bug that could prevent charging and discharging
  after a full charge"):
  1. `SOC_AlgorithmEntry`: at start-up a stored SOC of 100 % with max cell < 3400 mV becomes 99 %, a stored 0 % with min
     cell > 2900 mV becomes 1 % (only with valid AFE data).
  2. `APP_ShutdownWake_Step` (1 s): while the cell under-voltage trip is active, a charge current > 2.0 A resets a 200 s
     timer and lifts the under-voltage discharge block (both MOSFETs may conduct); without charge current the BMS enters
     STOP mode after 200 s (§9).
  3. Not assigned with certainty (candidates: full-latch handling in the MOSFET control, the "report clamp" in §5).
- **All 653 functions named and typed**, 0 `FUN_*`, all verification gates passed (§2.3). Tags: `NAME_VERIFIED` 653,
  `NAME_LOW_CONFIDENCE` 10.

## 1. Binary Fingerprint

| File | Version | Size | SHA-256 |
|---|---|---|---|
| `115_bms_VNSE3-0_Ac3.0BmsAPP_V115_ota_110451.bin` | 115 | 83,968 B (0x14800) | `cb50cc792e247d490abfe8cbc45d611dae49e7b8a8e615000cc5962ece9af4be` |

Image version tag at `0x08010000`: **`BMS1`** — the same tag as the D BMS (A: `BMS2`). The CAN OTA writer also checks
`BMS1` at `0x0802B800` (§11).

## 2. MCU, Memory Map, Processing Status

### 2.1 MCU: Nations N32G45x (Cortex-M4F)

Same evidence as for the D BMS: 102-entry vector table, N32 RCC/PWR layout in `SystemInit` (`SCB_VTOR = 0x08002800`),
register and IRQ names from the N32G457 SVD. Peripherals used: SPI1 (KA495XX, chip select PA4, ready PA6), CAN1
(all traffic), CAN2 (RX0 interrupt only latches a frame, nothing evaluates it), USART1 (115200 8N1, letter-shell console
and Ymodem update), ADC1 with DMA (ch1, ch2 = MOS NTC, ch9 = environment NTC), TIM2/TIM3 (one-shot callbacks), I2C EEPROM
(bit-banged on PB10/PB11), GPIOD bit-banged serial bus to the RTC (`KA495XX_SPI_*`/`RTC_*` functions, code identical to D),
IWDG (prescaler 6, reload 0x30D, started in `System_Init_Main`).

### 2.2 Memory Map

| Range | Contents |
|---|---|
| `0x08000000`–`0x080027FF` | Bootloader (not part of the image); boot request page `0x08002000` (`OTA_Set_Reboot_Params`) |
| `0x08002800`–`0x08016FFF` | Application image; vector table at `0x08002800` |
| `0x08010000` | Image version tag `'BMS1'` |
| `0x08013BB0`–`0x080145D7` | RCC tables, NTC curves A/B (byte-identical to D) |
| `0x080145D8`–`0x080161B3` | E3 tables: RTC days per month, `g_shellCrcSelfTestTable` (KA495XX expected register values), channel masks, CRC-8 table, NTC curves C (−30…120 °C) and D (−40…100 °C), five parameter sets (SOH, OCV, charge and discharge matrices with axes) |
| `0x080169BC` | Keil region table (2 regions), `0x080169DC` `shellCommandTable` (39 × 16 B), `0x08016C4C` `.data` load image |
| `0x0801E000` | OTA staging area (CAN OTA and Ymodem); tag word at `0x0802B800` |
| `0x20002800`–`0x20002BF7` | `.data` (0x3F8 B, LZ-compressed in the image; D: 0x3C4 B) |
| `0x20002BF8`–`0x2000C69F` | `.bss` (0x9AA8 B); per-pack record array 20 × 0x5A B at `0x20003144`, configuration block `0x2000384C` (0xA8 B), SOC state block `0x2000397C` |
| `0x2000C6A0` | `__initial_sp` (stack top) |

The RAM layout differs from D and A; RAM names were taken over from D only where the aligned code uses the same object.

### 2.3 Processing Status in Ghidra

- Program `/VNSE3/BMS/BMS_115_VNSE3-0_Ac3.0BmsAPP_V115_ota_110451.bin`: **653 functions, all named**, every function has
  a user-defined prototype (task entries `void f(void *pvParameters)`), plate comments, 0 `FUN_*`, 0 duplicates; 613
  RAM globals named and typed (arrays/structures for cell voltages, per-pack records, configuration, SOC block, profile
  tables, FreeRTOS lists, shell instance).
- Gates (`ve3_bms_gates.py`, the split BMS gate set used for A 1105): 0 undefined bytes, 0 orphan instructions, 0 flows
  into data, 0 function pointers to non-code, 0 literal-only functions, 0 literal pools decoded as code, 0 calls into
  function bodies, 0 unnamed referenced RAM globals, 0 default prototypes, task names = `xTaskCreate` name strings,
  no-return flags sane. Decompilation export: 653 functions, **0 failures**.
- Structural fixes: 39 tail-called functions split out of their callers (the compiler ends many functions with `b.w` to
  the next one), literal pools behind non-returning reset calls typed as data (6 places), the empty shell password string
  `""` of user `BLUE` (pointer `0x080161CD`) is data, not a function, the E3 data tables typed (§2.2).

## 3. FreeRTOS Tasks

`RTOS_CreateAllTasks` (`0x08009990`) creates nine tasks (`configMAX_PRIORITIES` = 5; D: ten tasks, 10 priorities):

| Task | Entry | Stack (words) | Prio | E3 body |
|---|---|---|---|---|
| `RTOS_vTaskLED` | `0x08009BAC` | 0x200 | 4 | key-hold LED + dead slave poll (`APP_RS485_SlavePoll_Step`) |
| `RTOS_vTaskKeyScan` | `0x08009B9C` | 0x200 | 4 | power key PB13 (`KEY_ScanPowerButton`) + power state machine |
| `RTOS_vTaskKA495XX` | `0x08009B90` | 0x200 | 4 | KA495XX state machine every 5 ms |
| `RTOS_vTaskINVCanComm` | `0x08009B7C` | 0x200 | 4 | CAN RX queue → dispatchers, 5 ms (D: CanRec + CanSend) |
| `RTOS_vTaskRS485Comm` | `0x08009BCC` | 0x200 | 4 | letter-shell on USART1, or Ymodem receiver after `update` |
| `RTOS_vTaskSocAlgorithm` | `0x08009BF4` | 0x280 | 3 | SOC/SOH, 100 ticks |
| `RTOS_vTaskProtectCheck` | `0x08009BBC` | 0x100 | 4 | protection + MOSFET control, 100 ticks |
| `RTOS_vTaskControlLogic` | `0x08009B3C` | 0x100 | 4 | aggregation, shutdown/wake logic, 1 s |
| `RTOS_vTaskDataSave` | `0x08009B64` | 0x100 | 0 | 1 s tick, console status print, IWDG feed |

## 4. Interfaces

### 4.1 CAN (CAN1 only)

Accept-all filter, RX queue 50 × 20 B; there is no TX queue: `CAN_TX_SendExtFrame` / `CAN_TX_SendStdFrame` write
directly into a free mailbox (a frame is dropped silently when all three are busy). Inbound frames are N32
`CanRxMessage` structures, "byte n" = struct offset 0x0B + n. `CAN_CommLossWatchdog`: CAN1 re-init after 12000 idle
polls, system reset after 20 re-inits.

**Extended IDs, inbound** (`CAN_RX_Handler` `0x08005878`, `CAN_RX_AuxCommand_Handler` `0x08008988`):

| ID | E3 action | D 118 |
|---|---|---|
| `0x180101AA`…`0x180106AA`, `0x1801FFAA` | status request → `CAN_TX_Orchestrator`: PF1–PF4 + per-pack burst of pack 1 | `0x1801nnAA` per pack, `0x1801FFAA` = master heartbeat with enable/discharge % |
| `0x180107AA` | self check, reply same ID, 1 byte: 1 ok, 2 EEPROM, 3 flash, 4 AFE fault | — |
| `0x100101AA` | system command, opcode = byte 0 (table below) | system command |
| `0x100201AA` | set SOC to byte 0 % (`CAN_CMD_02_Handler`), echo | SOC reset |
| `0x100601AA` | **select battery profile** 0..5 (byte 0), store config to EEPROM 0x258, reload tables, echo | — |
| `0x100301AA`, `0x100401AA`, `0x100501AA` | not handled | discharge limit / cmd 04 / CAN restart flag |
| class 3 (`(id >> 12) & 0xF == 3`), low byte `0x08` | set RTC from a 32-bit epoch (+8 h) | — |
| class 3, low byte `0xCC` | set RTC date/time (two frames) | — |
| class 3, low byte `0xCE` | CAN OTA block writer (`BMS_OTA_Flash_Writer`) | same |
| class 3, other low bytes (1–5, 0x0A, 0xC4 …) | ignored | status, pack mode, handshake, factory reset, notify, activation |
| `0x04410110` | current/ratio value from the inverter (VNS) → float `0x2000299C`, used by the float SOC integrator | — (A: `0x03410110` PV rescue request) |

**System command opcodes `0x100101AA`** (`CAN_RX_System_Command_Handler`):

| Opcode | E3 | D 118 / A 1105 |
|---|---|---|
| 1 | **shutdown request** (`g_shutdownRequestFlag`): ControlLogic logs event 2, stores the RTC epoch at EEPROM `0xFC08`, STOP mode with key wake, reset | address request |
| 2 | **factory reset**: `BMS_Config_Params_Factory_Reset`, echo, 500 ms, power state 4 → reboot | factory reset + reboot |
| 4 | reboot (power state 4) | fault mode 4 |
| 0x0A | `CAN1_Reinit_OrSystemReset(0)` | — (A: restart CAN) |
| 0xA0 | wake/re-init request: re-initialise peripherals, IWDG reload 0x61A, sleep → the IWDG resets the MCU; ignored right after an IWDG reset | — |
| 3, 5–9, 0x0B, 0x0C | ignored (no relay, no lock/unlock, no master/slave) | relay, handoff, unlock/lock (8/9), A: master/slave (0x0B/0x0C) |

The opcode-1 behaviour of D/A (address request; on non-master packs part of the pack-coding sequence) does not exist:
on the E3 opcode 1 switches the battery off.

**Extended IDs, outbound** (all `0x18xxAAyy`, DLC 8, little endian):

| ID | Builder | Content (E3) | Difference to D |
|---|---|---|---|
| `0x1801AA01` | `CAN_TX_Build_PF1_PackMeasurements` | max pack voltage, summed current, max NTC, **float SOC** × 10 | D: average SOC of the packs |
| `0x1802AA01` | `CAN_TX_Build_PF2_Capacity` | **constant** `00 14 01 01 00 01 5A` + status byte: 5120 Wh, 1 pack, online mask 0x0001, active pack 1, discharge limit 90 %; byte 7 = key/shutdown state | computed from pack count/capacity |
| `0x1803AA01` | `CAN_TX_Build_PF3_ChargeDischargeLimits` | charge voltage limit 57.6 V, charge current, discharge current, byte 6 flags (bit0 charge, bit1 discharge allowed), byte 7 = 0 | D byte 7: balancer/aggregation flags; D clamps at 60.9 V |
| `0x1804AA01` | `CAN_TX_Build_PF4_ProtectWarnings` | protect word 1/2; **byte 4 = 0xAA after an IWDG reset**, bytes 5–7 = 0 | D byte 6 = online flags, byte 7 = error lock |
| `0x1820…0x1823AA00+n` | `CAN_TX_PerPack_10Msgs` | voltage, current, SOC, cycles / limits, MOS bits / max-min cell + temps / protect 1/2, alarm word, **version byte 115 (0x73)** | same frame set |
| `0x1830…0x1833AA00+n` | same | 16 cell voltages | same |
| `0x1840/0x1841AA00+n` | same | 4 cell NTCs / average, environment and MOS NTC, **byte 6 = battery profile, byte 7 = MOS-off bits** | D: bytes 6–7 = 0 |
| `0x1852/0x1853AA00+n` | same | **E3 only:** SOH × 10 and the empty threshold 2800 mV / charged and discharged energy counters | A: `0x1851/0x1852` with other content |

**Standard 11-bit IDs** (`CAN_RX_StdFrame_Handler` `0x08008AE4`):

| ID | Direction | Content |
|---|---|---|
| `0x305` | in | request → `CAN_TX_InverterProtocol_AllFrames` sends the frames below, 5 ms apart |
| `0x351` | out | charge voltage limit (0.1 V) = **57.0 V + pack count × 0.1 V** (57.1 V with one pack, clamp 58.9 V), charge current limit, discharge current limit, cycle count |
| `0x355` | out | SOC × 10 (float SOC), SOH, capacity in Wh (5120), byte 6 = 115 (version) |
| `0x356` | out | pack voltage (10 mV), current (0.1 A), max/min cell temperature |
| `0x359` | out | protect word 1, protect word 2, alarm word |
| `0x35C` | out | byte 0 charge/discharge allowed flags, byte 2 key-hold/shutdown flag |
| `0x360`–`0x363` | out | 16 cell voltages (mV) |
| `0x36A`, `0x36B` | out | 4 cell NTCs / environment and MOS NTC |
| `0xCF` | in | MOSFET command: byte 0 ≠ 0 enables control by byte 1 (1 = discharge MOS on, 2/3 = charge MOS on) |
| `0x300` | in | byte 0 == 1 → shutdown request (same path as opcode 1) |

There is no periodic transmission; everything is sent on request.

### 4.2 USART1: console and Ymodem (no RS485 inter-pack protocol)

`RTOS_vTaskRS485Comm` serves the letter-shell (115200 8N1). The shell command `update` switches the task to a Ymodem
receiver (`OTA_Ymodem_Receive`: first packet name + size, limit 1 MB, data to the staging area `0x0801E000`, ends in
`OTA_Set_Reboot_Params` and a reset). The D frame protocol (`'s'` start byte, XOR checksum, opcodes 0x00–0x34) is still
compiled in as a sender (`RS485_Send_Packet_Bus1`, `RS485_Build_And_Send_Frame`) but has no working transport: the ring
buffer driver only knows port 0, the sender uses port 1, USART2/UART4 are only de-initialised.

### 4.3 Debug Shell (letter-shell)

User `BLUE` (empty password), 39 table entries: 10 key bindings, `help`, `clear` and **27 commands** — none of the D/A
pack commands (`bat_data`, `inq_uid`, `set_work_pack`, `factory`, `lock_set`, …):

| Command | Handler | Function |
|---|---|---|
| `reset` | `CLI_Cmd_Reset` | NVIC system reset |
| `sox` | `CLI_Cmd_Sox` | SOC/SOH block print |
| `set_soc` | `SOC_SetSocPercent` | set SOC |
| `cyc_printf` | `CLI_Cmd_CycPrintf` | periodic status print on/off |
| `protect` | `CLI_Cmd_Protect` | active protections as text |
| `update` | `CLI_Cmd_Update` | Ymodem firmware update |
| `rtc_read` / `rtc_write` | `CLI_Cmd_RtcRead` / `CLI_Cmd_RtcWrite` | RTC |
| `chg_sop` / `dsg_sop` | `CLI_Cmd_ChgSop` / `CLI_Cmd_DsgSop` | current-matrix lookups |
| `self_check` | `CLI_Cmd_SelfCheck` | same as CAN `0x180107AA` |
| `restore` | `CLI_Cmd_Restore` | factory protection block + erase the SOC calibration log, reset |
| `set_table` | `CLI_Cmd_SetTable` | battery profile 0..5 |
| `chg_mos` / `dsg_mos` | `CLI_Cmd_ChgMos` / `CLI_Cmd_DsgMos` | force charge/discharge MOS off |
| `can_send` / `can_err` / `ban_can` | `CLI_Cmd_CanSend` / `CLI_Cmd_CanErr` / `CLI_Cmd_BanCan` | test frame `0x112233`, CAN error flag print, slave-poll inhibit |
| `set_pro_val` | `CLI_Cmd_SetProVal` | edit protection values (RAM only) |
| `reson` | `CLI_Cmd_ResetReason` | reset cause (IWDG, WWDG, soft, POR, hard, sleep) |
| `log_prt` / `log_write` / `log_read` / `log_clearAll` | `EventLog_*` | event log |
| `shell_100ms_tick`, `chgDsgIdleTime` | `CLI_Cmd_Shell100msTick`, `CLI_Cmd_ChgDsgIdleTime` | debug values (table description wrongly "log_ClearAll") |

### 4.4 EEPROM layout (I2C, bit-banged)

| Address | Contents |
|---|---|
| `0x0000` | state block 0x50 B (written on sleep/wake paths); byte 0 also used by the EEPROM self test |
| `0x012C` | auxiliary block 0x1C B (defaults 4000/100/120, no reader found) |
| `0x0258` (600), 0xA8 B | configuration block: magic `0x62`, `'s'`, cell count 16, profile index, protection values with delays, ones-complement checksum (`BMS_Config_Params_Load_Or_Default`) |
| `0x0400` + 0x14 × i, i = 0..49 | SOC calibration records (real/show/max/min SOC, throughput) — same slots as D |
| `0x1000` + 8 × i, i = 0..49 | event-log index (packed time, event id, record address) |
| `0x2000`–`0x2CE3` | event records 0x42 B (time, SOC, current, max/min cell, NTCs, 16 cells, protect words) |
| `0xFC00` / `0xFC04` | charged / discharged energy counters (on D `0xFC00` is the error-lock marker) |
| `0xFC08` | RTC epoch of the last shutdown |
| `0xFC0C` | float SOC × 10 for the next start |

Event IDs 0..10 are accepted (A: < 6); used: 1 protection change / key shutdown, 2 CAN shutdown, 3 SOC drop to 0 %,
4 SOC jump > 2 %.

## 5. Cell Monitoring — KA495XX

- SPI1, polled; read frame `E1, reg<<1, CRC` → `hi, lo, CRC`, write frame `E0, reg<<1, hi, lo, CRC`; CRC-8 polynomial
  0xD5 (same as D). 96-entry shadow register cache; configuration is re-checked every cycle (registers 7, 0x0A, 0x12,
  0x18) and against the 27-entry table `g_shellCrcSelfTestTable` (D name kept; it is the expected register table). The
  table equals D 118 except **register 0x0E: 0x0400 (D: 0x0B00)**; `KA495XX_RegisterInit` writes the same value.
- Cell count from the configuration (default 16, `g_afeCellChannelMaskTable` 4..17 cells, identical to D: for 16 cells
  AFE channel 14 is skipped). Cell mV = code × 5000 >> 14. 4 cell NTCs (NTC channel mask 0x1F).
- Measurement cycle every 5 ms task tick: 6 ADC rounds, value = (sum − max − min) >> 2; current with gain factor
  (clamped 0.5..2.0) and a fixed **1.08** correction (D: 0.88).
- **Balancing** (`KA495XX_CellBalanceDecision`): only when the cell spread ≥ 30 mV and the current > 1.0 A; a cell is
  switched on at ≥ 3501 mV and ≥ 30 mV above the minimum, off below 3500 mV or < 20 mV above; alternating odd/even
  channels. Thresholds identical to D.
- **Report clamp** (`BMS_CellVoltageReportClamp` `0x0800A1E0`, E3 only): a one-shot check after start (state 0xFF): if the
  pack voltage is ≥ 51.80 V or a cell ≥ 3700 mV for about 1000 cycles, the published cell voltages are frozen and after
  300 more cycles all published cells are set to **4000 mV**; it is released when the condition has been false for 20000
  cycles. If the condition is false first (600 cycles), the check ends for good. Protection uses the raw values. The intent
  (signalling "full" to the inverter?) is not documented.
- No heater control exists in the image.

## 6. Pack Management — single pack

- `g_pack0SlotId` (`0x20003144`) is set to 0xFF at start and only the unreachable address states would change it.
- `APP_PowerAndAddressStateMachine` (`0x080064D0`, KeyScan task): states 1 and 2 (slave id confirm / assignment with PB12
  pulses and frames 0x27/0x25 to up to 20 slaves) are **never entered** — only 0, 3 (key shutdown) and 4 (reboot) are
  written. `APP_RS485_SlavePoll_Step` (LED task) and `BMS_AggregatePackData` (loop over `g_slavePackCount` with online
  mask) run, but the slave count stays 0, so only record 0 (own pack) contributes.
- Per-pack record array: 20 records × 0x5A B (`0x20003144`–`0x2000384B`, D: 0x60-byte records). Record 0 is filled by
  `BMS_UpdatePublishedMeasurements`; records 1..19 are only read by the dead aggregation path.
- No error lock, no relay/precharge handshake between packs, no handoff, no pack selection.

## 7. SOC/SOH

- Same SOC library as D (orchestrator, coulomb counting, smoothing, charge/discharge integration, energy counters,
  OCV initialisation, 50-slot calibration log), with E3 constants: rated capacity **100 Ah** (D: 50 Ah), cycle count =
  throughput / 10 / 10.24 (D: 5.12), full at max cell ≥ 3600 mV, empty at min cell ≤ 2800 mV, show SOC capped at 99 %
  until full, floors 1 % / 12 %.
- A **second, float SOC** (`0x20002B78`, `SOC_FloatIntegrator_Callback` every 50 ms) integrates the current value received
  from the inverter (`0x04410110`) with hard syncs at 100 % / 0 % (cell thresholds above). It is the SOC sent in
  `0x1801AA01`, `0x1820AA0n` and `0x355`; the struct SOC goes to the event log. Both can differ.
- Battery profiles (`SOC_LoadProfileTables` `0x0800E0E4`): profile 0, 1 and invalid values use set 0; 2 → set 1, 3 → set 3,
  4 → set 2, 5 → set 4. Per set: SOH curve (0 → 100 %, 1000 → 95 %, 2000 → 90 %, 5000 → 80 %, 20000 cycles → 75 %; set 4:
  6000 → 70 %), capacity-vs-temperature table (in `.data`), OCV table 11 × 4 columns, charge matrix 14 × 11, discharge
  matrix 17 × 11.

## 8. Charge/Discharge Limits

| Item | Venus E3 115 | Venus D 118 |
|---|---|---|
| Current limit scale | matrix % × 10 in 0.1 A → 100 % = **100.0 A** | % × 5 → 50.0 A |
| Charge matrix (set 0) | 14 temperature rows (−10…60 °C) × 11 SOC columns: 0 % below −10 °C, 5 % at 0 °C, 30 % at 10 °C, 80 % at 20–25 °C, 100 % at 30–50 °C (80 % above 90 % SOC, 50 % above 95 %), 30 % at 55 °C, 0 % at 60 °C | 18 × 12 matrix, max 50 A |
| Discharge matrix (set 0) | 17 rows (−30…60 °C) × 11 SOC columns: 0 at ≤ −20 °C, 10–50 % at −10…0 °C, 100 % at 15–50 °C, 50 % at 55 °C, 0 at 60 °C | 18 × 5 matrix over cell voltage |
| Charge voltage limit | 57.6 V (PF3); `0x351` reports 57.0 V + pack count × 0.1 V | 57.6 V, CAN clamp 60.9 V |
| Charge current | 0 when charging is not allowed (full, protections) | — |

## 9. Protection Functions

Default thresholds (`BMS_Config_Params_Factory_Reset`, debounce in 100 ms ticks; values as in D unless noted):

| Group | Warn | Trip | Release |
|---|---|---|---|
| Cell OVP | 3600 mV / 1 s | 3650 mV / 3 s, 3700 mV / 1 s | 3350 mV / 1 s |
| Cell UVP (runtime, avg NTC > 0 °C) | 2700 mV | 2500 / 2450 mV | 3000 mV |
| Cell UVP (avg NTC ≤ 0 °C) | 2400 mV | 2200 / 2000 mV | 3000 mV |
| Discharge current | 55 A / 3 s | 80 A / 5 s, 100 A / 0.5 s | latch 60 s |
| Charge current | 55 A / 3 s | 65 A / 1 s, 100 A / 0.5 s | latch 60 s |
| MOS NTC | 90 °C | 115 °C | 85 °C |
| Environment NTC | 61 / −26 °C | 65 / −30 °C | 60 / −25 °C |
| Cell NTC discharging (5 s) | — | high 60 / 65 °C, low −15 / −20 °C | 57 / −10 °C |
| Cell NTC charging (5 s) | — | high 55 / 60 °C, low −15 / −20 °C | 50 / −10 °C |

D runtime UVP values are 3100/3000/2500 mV (> 0 °C); E3 has no temperature compensation of the OVP. Other checks: cell
break (pack ≥ 35.00 V and a cell < 1500 mV), cell difference > 1000 mV, NTC outlier exclusion (≥ 30 °C away), NTC
breakdown, MOSFET breakdown (current > 5.0 A with the MOSFET off; D: 3.0 A).
**Short circuit** (`Protect_ShortCircuitCheckAndRecover`): discharge MOS on and PB1 low for 5000 timer ticks → flag,
both MOSFETs off; then up to 10 recovery probes with the precharge output PB2 and the MOSFETs are switched on again (D only
sets the flag).
**MOSFET control** (`MOS_ControlFromProtectionFlags`, 100 ms): PC5 = charge MOS, PB0 = discharge MOS, PB2 = precharge;
discharge on via 7 ticks of precharge. Blocked as listed in the code comments (OVP/UVP, over-current, temperatures, MOSFET
breakdown, full latch, shell force flags, AFE communication loss, invalid data).
**Deep-discharge rescue** (changelog item 2, `APP_ShutdownWake_Step`): while the cell UVP trip is active, a charge current
> 2.0 A clears the UVP discharge block and restarts a 200 s timer; after 200 s without charge current the BMS enters STOP
mode (`System_EnterStopMode_ThenReset`). D: own MOSFETs only below 2.0 A, standalone pack → fault mode 3 after 200 s; A: waits
120 s for CAN `0x03410110`.
**Power key** PB13: 3.0–6.99 s → shutdown (event log, EEPROM `0xFC08` stamp, STOP mode with key wake), ≥ 7 s → reboot.

## 10. Standing Checks (FW_Diff_Workflow)

| # | Check | E3 BMS 115 |
|---|---|---|
| 6a | Bound in the per-pack sender `CAN_TX_PerPack_10Msgs` | **`pack_id != 0 && pack_id < 10`** (packs 1..9, record index pack_id − 1) — same as D 118 (A 1105: `< 7`). The only caller (`CAN_TX_Orchestrator`) always passes **1**, so only pack 1 is ever sent. |
| 6b | Pack/slot limits, records | record array 20 × 0x5A B; address assignment code would accept up to 20 slaves, but it is unreachable; PF2 reports a constant 1 pack / mask 1 |
| 6c | Control side | E3 Control 151 keeps 8 records × 0x60 B and the 34000–34699 blocks; with this BMS packs 2..7 always read 0 |
| 6d | Cells per pack | **16** (configuration, AFE channel mask, 16 cell slots in 0x1830–0x1833 and 0x360–0x363) |
| — | Consistency with one fixed pack (owner statement) | yes: constant PF2, burst only for pack 1, no pack coding, no master/slave role, slave bus without transport |
| 8 | Re-check of non-functional registers | no E3 register CSV exists yet; the BMS-side facts for the open D/A registers are listed in §13 |

## 11. OTA/Update

- **CAN OTA** (class 3, opcode 0xCE, `BMS_OTA_Flash_Writer`): 1 KB blocks with CRC16/Modbus, staging at `0x0801E000`,
  check of the tag **`'BMS1'`** at `0x0802B800` and of the total CRC, then `OTA_Set_Reboot_Params` (boot request page
  `0x08002000`, magics `0x11223344`/`0x55667788`) and reset — same flow and the **same tag as the D BMS**, so the tag check
  alone does not prevent a D image from being accepted (A uses `BMS2`).
- **Ymodem** over the console (`update`), same staging area.

## 12. Security Notes

- Shell user `BLUE` without password; `restore`, `set_table`, `set_pro_val`, `chg_mos`/`dsg_mos`, `rtc_write`, `log_write`
  and `update` are reachable on the console.
- CAN commands without authentication can switch the battery off (opcode 1, `0x300`), factory-reset it (opcode 2),
  reboot it, change the battery profile persistently (`0x100601AA`), set the SOC, switch the MOSFETs (`0xCF`) and write a
  new firmware (0xCE).

## 13. Registers behind BMS frames (differences to D and A)

The E3 Control 151 uses the same descriptor table and the same `0xAA` CAN dispatcher as D 151 (17 entries, including
`0x1852` → record +0x58 and `0x1853` → record +0x50, which no Modbus register exposes). With BMS 115:

| Register(s) | Source | E3 BMS 115 | D 118 / A 1105 |
|---|---|---|---|
| 32100, 32101, 32104, 32108/35010 | PF1 | real (own pack; SOC = float SOC) | real (stack values) |
| 32105 | PF2 b0–1 | constant **5120** | packs × 2560 (A: × 2080) |
| 32109 / 32110 / 32111 | PF2 b2 / b3–4 / b5 | constant **1 / 1 / 1** | real |
| 32106, 32107, 35111, 35112 | PF3 | real E3 limits (0.1 A, up to 100 A) | real |
| 35110 | PF3 b0–1 | constant 576 (57.6 V) | D constant 576, A dynamic |
| 32112 | PF4 b7 | **always 0** (no error lock) — FUNCTIONLESS on E3 | real lock flag |
| 32113 / 30109 | PF4 b6 | **always 0** | D: online flags |
| PF4 b4–5 (`g_pgn1804InfoBatWarn1`, Control-internal) | PF4 | 0x00AA after an IWDG reset | 0 |
| 34000–34003, 34005–34008, 34011–34016, 34018–34033, 30100–30108, 37005–37012 | per-pack frames pack 1 | real; cells 1–16 used; 34002/37005 carry the float SOC | real |
| 34004 / 30210 | 0x1821 b6 | MOS output bits (bit0 PC5 charge, bit1 PB0 discharge) | real |
| 34009 / 37011 | 0x1823 b4–5 | alarm word (warning bits of C6/C7) | warning word |
| 34010 / 37012 / 30204 | 0x1823 b6–7 | **115** | 118 / 110 |
| **34017 / 30106** | 0x1841 b6–7 | **FUNCTIONAL on E3**: low byte = battery profile 0..5, high byte = MOS-off bits | FUNCTIONLESS (always 0) |
| 34100–34699 (packs 2–7) | — | always 0 (never sent) | D: real for present packs; A: packs 2–6 |
| **45011** | `0x100601AA` | **FUNCTIONAL on E3**: selects the battery profile (Control remap 1→1, 2→2, 3→4, 4→0, 5→5, other → 0; profile 3 cannot be reached via Modbus) and stores it in the BMS EEPROM | UNKNOWN on D (no BMS receiver) |
| 45028 | `0x100101AA` 02 | FUNCTIONAL (factory reset + reboot) | same |
| 45027 | class 3 func 0x0A | FUNCTIONLESS (not handled) | D: master reboot |
| 45029 | `0x100101AA` 03 + encode | FUNCTIONLESS (no pack coding) | D: pack coding |
| 45030 | `0x100101AA` 0x0B/0x0C | FUNCTIONLESS | D: none, A: master/slave |
| 45031 | 0x04 then 0xA0, wait for PF4 b4 = 0xAA | **handled on E3**: 0x04 reboots, 0xA0 makes the BMS sleep until the IWDG resets it, afterwards PF4 b4 = 0xAA — whether the Control's 5 s windows are long enough is open (probably not: IWDG ~10 s) | FUNCTIONLESS |

## 14. Cooperation with the Venus E3 Control 151 and Micro (VNS)

- Control: same request/heartbeat frames as on D; the BMS answers every `0x1801xxAA` with one pack. Discharge limit %,
  master heartbeat and EMS flags of D (`0x1801FFAA` payload) are not evaluated.
- Inverter/Micro (assumed sender, not yet verified against the E3 Micro 119): `0x305` request with the Pylontech-style
  answer, `0xCF` MOSFET commands, `0x300` shutdown, `0x04410110` current value for the float SOC. In the
  deep-discharge rescue the inverter must supply > 2.0 A charge current within 200 s.

## 15. Open Points

1. Sender and exact meaning of `0x04410110` (bytes 4–5 signed, 6–7 unsigned, scaled × 10000 / …) and of `0xCF`/`0x300`/`0x305`
   — needs the E3 Micro 119 analysis.
2. Changelog item 3 ("no charge/discharge after a full charge"): not assigned to a specific code change.
3. Purpose of the 4000 mV report clamp after start (§5) and of the unread EEPROM blocks `0x0000` and `0x012C`.
4. 45031 timing (Control waits 5 s, the E3 BMS needs an IWDG timeout of about 10 s).
5. Short-circuit timer tick unit (5000 ticks; TIM2 at 2 MHz would give 2.5 ms — not verified).
6. Reference-program naming issues seen on the way (D 118, not changed there): `CAN_SetTxMailboxTDLR/TIR` and
   `CAN_Set_FilterBank_FR1..3`/`CAN_SetTxMailboxTDHR` are GPIO BRR/BSRR and TIM CCR setters, `GPIO_Init_Output_PP_50MHz` and
   `GPIO_Init_Input_Floating` are swapped in D, `APP_ShellTable_Init_Then_Main` is `__scatterload`, `APP_SystemFaultHalt` is
   `main()`, `NTC_ConvertAndUpdatePackTempExtremes` (D) is an OCV bound recalibration (`SOC_RecalibrateBoundsFromOcv` in
   E3), `g_shellCrcSelfTestTable` is the KA495XX expected register table, `HAL_UART4_SendByte_Blocking` uses USART1.
   The E3 program keeps the D names where the object is the same (plates say what the code really does).
