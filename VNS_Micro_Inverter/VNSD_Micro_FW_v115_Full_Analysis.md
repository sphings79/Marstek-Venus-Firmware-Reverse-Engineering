# Micro-FW v115 (Venus D) — Full analysis

> 🇩🇪 Deutsche Version: [VNSD_Micro_FW_v115_Vollanalyse_de.md](VNSD_Micro_FW_v115_Vollanalyse_de.md)

**Status:** 2026-09-30.

**German:** `VNSD_Micro_FW_v115_Full_Analysis.md` ·
**v116 version:** `VNSD_Micro_FW_v116_Full_Analysis.md` ·
**Diff to successor v116:** `VNSD_Micro_Diff_v115_v116.md` ·
**Decompilates/Listing:** `Dekompilate/VNSD_Micro_115/` (local only, not in the repo) (696 functions, 932 named RAM globals,
`listing/` with Flash and CCM raw disassembly + `globals_and_registers.txt`)

This document is the v115 full analysis with the same structure as `VNSD_Micro_FW_v116_Full_Analysis.md`.
Contents that are provably identical between v115 and v116 (per `VNSD_Micro_Diff_v115_v116.md`
byte-by-byte or C logic verified) are not re-written here, but referenced with "unchanged vs. v116, see there" —
however, each section names the facts specific to v115 (addresses, version, limits), even if content otherwise matches.

## 0. Summary

- Image base **0x08004000** (16-KB bootloader below), image size 0x1C400 bytes (115,712 B),
  MCU **STM32G474**, Keil ARMCC, FreeRTOS (RVDS/ARM_CM4F, `heap_4`, 16-KB heap), letter-shell —
  identical to v116 (unchanged vs. v116, see there §0/§1).
- `0x6B64` B time-critical code run from CCM-SRAM `0x10000000`, reached via 10-byte veneers in
  Flash starting `0x08004960`ff. — **20 bytes more** than v116 (`0x6B50`), since v116 loses a function
  registration at one CCM location (`EMS_ResetMemoryByLevel`/`SUB_0800699c`, see diff document),
  content and behavior of CCM code otherwise identical.
- Manufacturer changelog notes for v115 only "Adapted to Control firmware 147" (Issue #21); no
  v114 decompilates available, thus in this session not verifiable against a prior version.
- **v115 is the 2200-W baseline** for the v116-raised 2500 W charge/discharge power path
  (full derivation with addresses in both versions: `VNSD_Micro_Diff_v115_v116.md`, table
  "2200 W → 2500 W"):
  1. `EMS_Inverter_CAN_Dispatcher` (`0x08005944`), CAN commands 0x03/0x04: range check
     `< 0x899` (2201) — upper limit effectively 2200 W.
  2. `Modbus_HandleRegisterAccess` (`0x080164d8`), registers 43133/43134 (`g_ratedPowerConfigA`/`B`):
     upper limit `0x898` (2200 W).
  3. Same function, register 45502 (factory test command "discharge reference"):
     `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)`.
  4. CCM `Ctrl_MainControlIsr`, grid connection ramp (`0x100029d8`): output/clamp limit of power
     PI controller at ramp start = `2200.0`.
  5. Same function, thermal derating base (`0x10005180`) = `2200.0`.
  6. `Bat_Mode3RoundRobinTimer` (`0x0801183c`): `g_batPowerRefW` = `±2200.0`.
  The **real-time hard limit** for immediate power setpoint (CAN commands 0x56/0x58,
  check `wert+0x9c4<0x1389`, ≙ ±2500 W) is already in v115 **2500 W** — the same inconsistency
  that v116 resolves at the six locations above is already present in v115.
  Noteworthy: `Ctrl_ResetControlState` (CCM, `0x10005fe8` literal) initializes power limit field
  **already in v115** to `2500.0`, not 2200.0 — this location does **not** belong to 2200-W baseline
  (see diff document, section "refuted candidate"). Similarly, EEPROM factory reset defaults for
  charge/discharge limit are already in v115 `0x9C4` (2500 W) — 2200-W capping acts only via the
  six runtime/configuration checks named, not via EEPROM factory reset path itself.
- Name finding on `EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex`: in this fresh v115 export
  both functions already correctly named (`0x08006cf0`/`0x08006d8c`, I2C random-access read with/without mutex) —
  the earlier name swap (`EEPROM_WriteRegister`) affected only an older intermediate v116 analysis version
  and is not an open point for v115 (see v116 document §4.2/§11).
- OTA/update (YMODEM over debug UART, additional CAN self-update path, BMS relay over CAN/RS485)
  structurally identical to v116 (unchanged, see v116 document §7 and diff document).
- Fault/warning codes, `err2` bit mapping, Chinese strings, shell command list, EEPROM layout:
  logically identical to v116, addresses only shifted (see respective sections below and v116 document).

## 0a. Checklist

| # | Question | Answer v115 | Evidence |
|---|---|---|---|
| 1 | Modbus registers changed (vs. prior version)? | Not verifiable — no v114 decompilates. Vs. **v116**: register 43133/43134 upper limit here `0x898`(2200) vs `0x9c4`(2500); register 45502 test reference `-2200.0` vs `-2500.0`. All magic codes (`0x55AA/0x55DD/0x55EE/0x55FF`) identical to v116. | `VNSD_Micro_Diff_v115_v116.md` |
| 2 | CAN protocol (vs. prior version)? | Not verifiable (no v114 data). Vs. v116: `EMS_Inverter_CAN_Dispatcher` (`0x08005944`) has for commands 0x03/0x04 range upper limit `0x899` vs `0x9c5`. Otherwise identical. | Diff document |
| 3 | AC feed-in — power limit? | **2200 W** at six configuration/ramp/init locations (list see §0); CAN real-time hard limit (commands 0x56/0x58) already 2500 W. v116 raises six 2200-W locations to 2500 W. | §0, §4.1, §4.2, §5 |
| 4 | Firmware update (own + BMS relay) possible? | Yes, structurally identical to v116 (§7). | §7 |
| 5 | Cloud domains? | Not applicable — Micro firmware has no separate cloud/network connection (identical to v116). | n/a |

## 1. Binary fingerprint

| Feature | Value |
|---|---|
| File | `115_micro_VNSD-0_202601230921310c0e30687.bin` |
| Size | 115,712 bytes (0x1C400) |
| SHA-256 | `b2505973136f414928927df3211ce4ada90bdebcc100b5c770230cb5484c5c98` |
| Link address | `0x08004000`–`0x080203FF` |
| MCU | STM32G474 (Cortex-M4F, Thumb-2), 16-KB bootloader below `0x08004000` |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-compressed RW segment) |
| Libraries | FreeRTOS (heap_4), letter-shell |
| Initial SP | `0x20009A70` (vector table @ `0x08004000`, identical to v116) |
| Reset vector | `0x080042B1` (Thumb) → `Reset_Handler` (`0x080042b0`) |
| Version constant | **0x73 (115 dec.)** @ `0x08010000`, followed by ASCII `"VD01"` |
| Image trailer | Device tag `"VNSD"` @ `0x080203E0`; trailer word ASCII `"I1DD"` @ `0x080203FC` (v116 has `"13DD"` at same location — meaning in both versions not determined, see §10) |
| Manufacturer changelog | v115 — 2026-08-12: "Adapted to Control firmware 147" (Issue #21) |

**Scatter-load** (calculated from `listing/08000000_0800ffff.txt`/`listing/08010000_0801ffff.txt`):

| Source (Flash) | Target | Length | Method |
|---|---|---|---|
| `0x08018c80` (1,112 B compressed) | `0x20000000` (.data) | 0x8D8 (2,264 B) | `__decompress` (LZ77) |
| `0x080190d8` (27,492 B, uncompressed) | `0x10000000` (CCM-SRAM) | **0x6B64** (27,492 B) | `__scatterload_copy` (pure copy) |
| – | `0x200008D8` (.bss) | 0x9198 (37,272 B) | `__scatterload_zeroinit` (zero init) |

Verification: `0x200008D8 + 0x9198 = 0x20009A70` = stack top, identical to v116. `ScatterRegion` table
at `0x08018900` (v116: `0x0801892c` range, see below), `shellCommandTable`/`ShellCommand[53]` at
`0x08018930` (v116: `0x0801892c`).

## 2. Memory map + analysis status in Ghidra

**RAM/peripheral blocks** — identical to v116 (same address boundaries, RAM layout per specification
unchanged):

| Block | Range |
|---|---|
| `.data` | `0x20000000`–`0x200008D7` |
| `.bss` | `0x200008D8`–`0x20009A6F` |
| CCM-SRAM | `0x10000000`–`0x10006B63` (**0x6B64** B code+data, 20 B more than v116) |
| APB1 | `0x40000000`–`0x4000AFFF` |
| APB2 (incl. HRTIM `0x40016800`) | `0x40010000`–`0x40017FFF` |
| AHB1 | `0x40020000`–`0x400243FF` |
| AHB2 GPIO | `0x48000000`–`0x48001BFF` |
| AHB2 Analog (ADC/DAC) | `0x50000000`–`0x50060BFF` |
| FMC/QSPI | `0xA0000000`–`0xA0001FFF` |
| PPB (Cortex system peripherals) | `0xE0000000`–`0xE00FFFFF` |

**Verification status** (program `/VNSD/Micro/Micro_115_VNSD-0_202601230921310c0e30687.bin`):

| Check | Result |
|---|---|
| Functions | **696**, all 696 named, 0 `FUN_*` |
| Ghidra symbols | 4,641 |
| Decompilation failed | 0 |
| Named RAM globals (.data+.bss, same counting method as v116) | **932** (v116: 925 — difference due to naming run coverage, RAM address layout itself identical) |
| Veneer functions (Flash↔CCM) | 49 (identical to v116) |
| `shellCommandTable` | `0x08018930`, `ShellCommand[53]` (v116: `0x0801892c`) |

File/folder convention as described in `Dekompilate/VNSD_Micro_115/` (identical to v116).

## 3. FreeRTOS tasks

From `Task_CreateAllAndStartScheduler` (`0x0800f424`, 242 B) — eight `xTaskCreate()` calls, then
`xPortStartScheduler()`. **Byte-by-byte comparison with v116 confirms: order, stack sizes and priorities
are identical** (`VNSD_Micro_Diff_v115_v116.md`), only entry addresses shift by general relocation:

| Task | Entry function (v115) | Stack (words) | Priority | Handle |
|---|---|---|---|---|
| `vtask_led` | `0x080163d8` | 128 | 16 | `g_ledTaskHandle` |
| `vtask_time` | `0x080164b8` | 256 | 15 | `g_timeTaskHandle` |
| `vtask_modbus` | `0x08016414` | 128 | 14 | `g_modbusTaskHandle` |
| `vtask_shell` | `0x0801642c` | 512 | 9 | `g_shellTaskHandle` |
| `vtask_prt` | `0x08016420` | 256 | 8 | `g_canRxTaskHandle` * |
| `vtask_can` | `0x080163c0` | 256 | 6 | `g_canRxTaskHandle` * |
| `vtask_can_receive` | `0x080163d0` | 256 | 7 | `g_canRxTaskHandle` * |
| IDLE | `0x08012fd8` | 128 | 0 | `xIdleTaskHandle` |

\* Same handle name duplication as in v116 (open point, see §10/v116 document §10).

## 4. Interfaces

### 4.1 CAN/FDCAN to EMS (Control) and BMS

Structure, filters, command set and message IDs are **identical to v116** (`BMS_CAN_Parser`,
`BMS_FwUpdate_CanHandler`, `HAL_FDCAN_RxFifo0Callback`, `CAN_RX_DispatchTask` are per diff
byte-identical) — complete command table see v116 document §4.1.

**Only content difference:** `EMS_Inverter_CAN_Dispatcher` (`0x08005944`, 494 B normalized opcode length),
commands 0x03 (discharge limit) and 0x04 (charge limit): range check `< 0x899` (2201) —
in v116 raised to `< 0x9c5` (2501). All other 20+ commands (0x01–0x60, 0xC1–0xCE) as well as BMS protocol
(`BMS_CAN_Parser`, PF values 1–4) and OTA subprotocol part (`0xCE`) are unchanged.

### 4.2 RS485/Modbus

Dispatcher `Modbus_HandleRegisterAccess` (`0x080164d8`, 1,158 B normalized opcode length); structure,
return values and complete register table are **identical to v116** — complete table see v116 document §4.2.

**Only content differences:**

| Register (dec./hex) | v115 value | v116 value |
|---|---|---|
| 43133/43134 / `0xA87D`/`0xA87E` (`g_ratedPowerConfigA`/`B`) | Upper limit `0x898` (2200) | `0x9c4` (2500) |
| 45502 / `0xB1BE` (factory test command "discharge reference") | `Ctrl_SetPowerReferenceClamped_veneer(-2200.0)` | `(-2500.0)` |

Register 41010 (direct power limit/derating switch, `800`↔`0x9C4`) and 44003 (`g_powerLimitDischargeW`,
800-W enforcement when `g_powerLimitDeratedFlag` set) are already in v115 identical to v116 (upper limit
independent of 2200/2500 change).

**Magic code register 0xAFC8 (45000):** Logic of `0x55DD`/`0x55EE`/`0x55FF` is **byte-identical to v116**
(see v116 document §4.2 for complete code snippet and I2C direction resolution). Specifically, function at
`0x08006cf0` in this v115 export is already correctly named `EEPROM_ReadRegister` (I2C read with mutex),
`0x08006d8c` as `EEPROM_ReadRegister_NoMutex` — the `0x55DD` branch provably reads
`g_batteryMode`/`g_cfgBackupModeEnabled`/`g_workMode` from EEPROM `0x102`/`0x300`/`0x301` back in v115,
rather than saving them.

### 4.3 Debug shell

`shellCommandTable` @ `0x08018930`, `ShellCommand[53]`, 42 `ShellCmd_*` handlers — identical to v116
(v116 document §4.3 for complete command list; all addresses there shift in v115 by general relocation of
~`+0x10`…`+0x18` B in relevant Flash area). `ShellCmd_bat_mode` (`0x0801473c`, 38 B) and `ShellCmd_pf_set`
(`0x08014f50`, 48 B) are **explicitly byte-by-byte verified against v116 and identical** (see diff document) —
no new debug behavior in either version.

### 4.4 I2C EEPROM layout

Layout, offsets and field meanings identical to v116 (see v116 document §4.4 for complete table). Factory reset
default values already in v115 `"VD01"`, version marker `0x73` (=115), charge/discharge limit each `0x9C4`
(2500 W) — factory reset default was thus in **both** versions 2500 W; 2200-W capping in v115 acts exclusively
via runtime/configuration checks named in §0/§4.1/§4.2/§5, not via EEPROM factory reset path itself.

## 5. Control & power section

Structure identical to v116 (`Inv_CurrentLoopIsr` @ CCM, `Ctrl_MainControlIsr` @ CCM, HRTIM ISR in
Flash) — see v116 document §5 for complete description of control chains (PLL/SOGI,
current controller/PR chain, HRTIM bring-up).

**The 2200-W constant in CCM** (float `0x45098000` = 2200.0f, in v116 replaced by `0x451C4000` = 2500.0f)
is at two locations:

| Address (v115) | Context |
|---|---|
| `0x100029d8` | In `Ctrl_MainControlIsr`: output/clamp limit of power PI controller at start of grid connection ramp (counter=120 ticks), `Ctrl_PiControllerReset()` called. |
| `0x10005180` | Base of thermal derating formula: from ~64 °C `(1.0-(T-64)/20)*2200.0`. |

**`Ctrl_ResetControlState`** (`0x10005f58`, 130 B) initializes the same power limit field
**already in v115** to `2500.0` (literal pool entry `0x10005fe8` = `0x451C4000`) — this third,
suspected CCM change location from earlier analysis is **not** a difference from v116
(there also `2500.0`, byte-identical function body except for address shift; details and cross-check
in diff document).

## 6. Protection & fault codes

`g_faultLogRing`, `FaultLog_Enqueue`, `Log_PrintFaultEntry` (`0x0800dba4`, 574 B normalized),
`Log_PrintDebugData` (`0x08010c48`), `Prot_LatchHwFaultToErrorWord` (CCM) — fault code table,
`err2` bit mapping and additional data are **identical to v116** (complete table see v116 document §6);
code addresses only shift by general relocation.

## 7. OTA/update

Structurally identical to v116: YMODEM self-update over debug UART, additional CAN self-update
path (`OTA_SelfUpdate_StateMachine`), BMS relay via CAN (`BMS_StartFwUpdateCan`/
`BMS_FwUpdate_StateMachine`) or RS485 (`BMS_FwUpdateStateMachineTask`). `OTA_CompleteFirmwareUpdate`
(`0x0800f550`) is per diff only address-shifted, logically identical. Complete description of
state machines see v116 document §7.

**Conclusion**: Both own firmware update and relay to BMS are in v115 unrestricted possible — identical to v116.

## 8. Security findings

Same structural findings as in v116 (v116 document §8), related to v115 baseline:

1. **Unauthenticated debug/factory mode access via Modbus:** register 45000 (`0xAFC8`) switches
   with `0x55EE`/`0x55DD`/`0x55FF` unauthenticated into factory mode or persists/reads operating states
   (identical to v116, see §4.2).
2. **Generic activation code `0x55AA`** works unauthenticated at multiple registers (identical).
3. **Direct power limit bypass:** register 41010 (identical to v116, independent of 2200/2500 W
   difference) — in v115 the bypass alternates between 800 W and the then-current rule maximum
   2500 W (register default value), although *other* configuration paths at that time only allowed
   2200 W as upper limit — same inconsistency as described in §0.
4. **Default shell user with empty password** (identical to v116).
5. `EEPROM_ReadRegister`/`EEPROM_ReadRegister_NoMutex` are in this export already correctly named
   (no open point for v115, see §4.2).
6. No direct network access to Micro firmware itself (identical to v116).

## 9. Chinese strings

184 UTF-8 strings with Chinese text, logically and in German translation **identical to v116**
(complete tables see v116 document §9) — all addresses shift by general relocation of this version
(e.g., `Log_PrintFaultEntry` strings from `0800db08`ff. rather than `0800db08`ff. in v116,
each offset by few to several dozen bytes).

## 10. Open points

Apply unchanged also to v115 (identical RAM layout, identical remaining logic) — complete list see
v116 document §10. Additionally v115-specific:

- Meaning of trailer word at image end (`"I1DD"` in v115, `"13DD"` in v116) not determined.
- Manufacturer changelog line "Adapted to Control firmware 147" not verifiable against prior version
  (v114, not available).

## 11. Corrections against old analysis

Same corrections as in v116 document §11 apply unchanged also against old documentation
(base address 0x4000 too low, MCU incorrectly STM32F3 vs STM32G474, CCM code portion completely missing,
I2C TX/RX swap, incorrect I2C function names from wrong base offset) — see there for details, they affect
base disassembly and thus both versions equally.

**Corrections specific to v115↔v116 diff of this session** (details:
`VNSD_Micro_Diff_v115_v116.md`):

- `Ctrl_ResetControlState`: no real 2200→2500 W change — v115 already loads `2500.0` at this location,
  byte-identical to v116.
- `EMS_ResetMemoryByLevel` (v115, `0x080069a4`): in v116 not renamed or removed, but taken over
  without function name/registration (`SUB_0800699c`) — code and call logic from `Inv_BuildTelemetryBlock`
  are identical. An earlier v116 analysis iteration incorrectly rated this location as "real new program path".
- `Ctrl_ApplyLlcPfcTestParams`/`Ctrl_ApplyPrControllerParams`/`Ctrl_ApplyVoltagePiParams` (only in
  v115 as separate function): in v116 directly inline-copied into `ShellCmd_power_cfg`/`ShellCmd_qpr`/`ShellCmd_vol_pi`
  (compiler inlining) — no behavior change.
