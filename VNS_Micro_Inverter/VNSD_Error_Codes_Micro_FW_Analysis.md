# Venus D Error and Warning Codes — Analysis (status 14.08.2026, round 2)

> 🇩🇪 Deutsche Version: [VNSD_Fehlercodes_Micro_FW_Analyse_de.md](VNSD_Fehlercodes_Micro_FW_Analyse_de.md)

**Goal:** Plain-text meaning for **36000** (warn_code), **36100/37013** (error_code1), **36102/37015** (error_code2).
**Status:** Mechanism fully clarified, **10 bits assigned**, hardware protection path identified.
Point 7/step 2 carried out (section 8). **Live cross-check evaluated from existing scan logs
(section 9) — it confirms the static derivation and clarifies the u32 word order.**

---

## 1. Origin

| Micro FW v116 | Name | Control struct 0x20014E9C | Modbus |
|---|---|---|---|
| `0x200019F4` | `err1` (u32) | +0x08 | **36100 / 37013** |
| `0x200019F8` | `err2` (u32) | +0x0C | **36102 / 37015** |
| `0x200019FC` | `war1` (u32) | +0x04 | **36000** |

The Control generates **no** codes of its own — `build_telemetry_block` (Micro) reads the three words,
the Control passes them through to Modbus unchanged.

## 2. Prerequisite: Image Was Only Half Disassembled

The actual blocker was not the analysis, but the state of the Ghidra project:

| | before | after |
|---|---|---|
| disassembled | 56.0 % (64,794 B) | **87.0 % (100,640 B)** |
| Functions | 452 | **561** |
| in functions | 51.0 % | 61.4 % |
| Error load sites locatable | 6 | **22** |

101 gaps were disassembled in Thumb mode, after which Ghidra's auto-analysis ran (+109 functions).
**The state is saved.** Spot check verified: at 0x08018A60 the raw bytes are
`03 21 35 20 FC F7 45 FB 75 48 41 79 41 F0 08 01` = `movs r1,#3 / movs r0,#0x35 / bl / ldr r0,[pc] /
ldrb r1,[r0,#5] / orr.w r1,r1,#8` — real code, not a misinterpreted data segment.

## 3. Mechanism

```
  <condition>
  mov  r0,#<event code> ; optionally r1..r3 = measurements/channel
  bl   <event logger 0x080150F2>
  ldr  rX,[<literal = 0x200019F4>]
  ldrb rY,[rX,#<byte offset>]
  orr  rY,rY,#<bit>            ; bic when clearing the warning
  strb rY,[rX,#<byte offset>]
```
Byte offset 0–3 = err1, 4–7 = err2, 8–11 = war1. Clearing uses its **own** event code.

## 4. Assigned Bits

| Bit | Event (set/clear) | Trigger | Confidence |
|---|---|---|---|
| **err1 bit 20** | 0x33 / 0x18 | CAN status flag `0x20000448` == 0; logger receives the bxCAN registers `0x40006840/50/54` → **CAN communication fault** | high |
| **err1 bit 21** | 0x3B / 0x0B | `bms_error_bitmask >> 2 != 0` → **BMS reports fault** | high |
| **err2 bit 0** | 0x36 | HW latch `0x200002BA` bit 1 | high |
| **err2 bit 1** | 0x36 | HW latch `0x200002BA` bit 0 | high |
| **err2 bit 3** | 0x36 | HW latch `0x200002BA` bit 2 | high |
| **err2 bit 4** | 0x36 | set on **every** HW protection shutdown (collective bit) | high |
| **err2 bit 7** | 0x13 | State machine: `float @r4+0x54 < 0`, GPIOD pin 8 is switched off; afterwards retry timer 60000 | medium |
| **err2 bit 8** | 0x31 | **Overload integrator** `0x200002E0`: rises as long as the measured value is above the threshold from `0x0801B258`, is decremented by 240 resp. 1800 below 4.5; on overflow set bit and log event 0x31 with current, voltage and counter value (`0x0801B16E`). **Addendum 21.09.2026**, see section 10 | high |
| **err2 bit 9** | 0x29 | State machine, parameter = `r4+0x23` | low |
| **err2 bit 11** | 0x35 | Limit exceeded **for longer than 3000 cycles**; **5 set sites** with channel index as parameter (3,2)/(4,2)/(4,3)/(11,6)/(2,3) | medium |
| **err2 bit 16** | – | End of the init/self-test sequence: return value of `0x08002A68` != 0 → **self-test failed** | medium |
| err2 bit 31 | – | is **cleared** via `bic` (FUN_0801bb50) | — |

> **Addendum 21 September 2026 — bit 8 was missing.** The set site `0x0801B16E` loads the base of the
> error block into **`r8`** (`ldr.w r8,[0x0801b254]`) instead of `r0` as everywhere else, and
> then accesses it with `ldrb.w r0,[r8,#0x5]`. The backward walk used to search for the load sites in 2026-08
> got stuck at the nearest `ldr` — which there was a threshold from a
> different literal pool — and discarded the site. It was found via an exhaustive
> raw scan of all literal pool words that point to the block (14 in the D image, 22 users).
> The wording in section 1 ("10 bits assigned") is thereby obsolete: there are **11**.

## 5. The Hardware Protection Path (newly identified)

`0x0801AEAC` — renamed to **`HW_Protection_Latch_To_ErrorBits`**:

```c
HW_Protection_Latch_To_ErrorBits() {
    log_event(0x36, <float1>, DAT_200002BA, <float2>);
    if (DAT_200002BA & 0x01) err2 |= 0x0002;   // Bit 1
    if (DAT_200002BA & 0x02) err2 |= 0x0001;   // Bit 0
    if (DAT_200002BA & 0x04) err2 |= 0x0008;   // Bit 3
    err2 |= 0x0010;                            // Bit 4 (collective bit)
    DAT_200002BA = 0;                          // acknowledge latch
    shutdown(7); shutdown(8); shutdown(10); shutdown(0x17); shutdown(0x28);
}
```

`DAT_200002BA` is a **hardware latch** (comparator/break signals) that is set elsewhere
(`0x0801AF3E/AF42`, presumably an ISR). Called from the main state machine
(0x080171F0 / 0x0801722E). The three HW bits are thus the **fast hardware protection trips**
(overcurrent/overvoltage/desat) — which exactly is decided by the comparator configuration.

In the same state machine there are concrete limits in plain form:
`0x41F00000` = **+30.0** and `0xC1F00000` = **−30.0** (float) → ±30 A window, as well as a
time threshold of **3000** cycles for bit 11.

## 6. No Plain Text in the Firmware

A full-text search over Micro v116 and BMS v118 for `ovp|uvp|ocp|otp|scp|over|under|fault|protect|alarm`
finds **no bit-to-text table**. Every naming has to be inferred from the trigger condition —
that is the reason why effort is incurred per bit here.

## 7. What Is Still Missing

15 of the 22 load sites are in a contiguous code block (0x08017000–0x0801A000) that is
**reached only via jump** — no `bl`, no `push {lr}`. Ghidra does not create any functions there even after
auto-analysis, so there is no decompilation, only disassembly.

**Next steps, sorted by yield:**

1. **Live cross-check** — by far the fastest way. Provoke a known fault
   (backup overload as in `backup_overload_ac_disconnect.csv` (local, raw data), disconnect battery, pull CAN) and
   log 36000/36100/36102. Each attempt assigns a bit **unambiguously** and validates
   the static derivation at the same time.
2. Create functions in the block 0x08017000–0x0801A000 **manually** at the jump targets, then
   decompile and read out the remaining conditions.
3. Evaluate the comparator/break configuration (HRTIM/COMP peripheral) in order to name `DAT_200002BA`
   bit 0/1/2 physically.

---

## 8. Addendum: Functions Created in the Jump Block (point 7, step 2)

The block 0x08017000–0x0801A000 is reached only via jump — no `bl` targets, no
`push {lr}` prologues. Ghidra's function search fundamentally does not work there. Procedure instead:
from each set site search backwards for the nearest **jump target** whose predecessor
terminates the control flow (unconditional jump or return), and create a function there.

**Result: 11 of the 15 set sites are now in functions** and can be decompiled.

| Function (new) | covers set site | Finding |
|---|---|---|
| **`Protect_Err2Bit11_Timeout3000`** (0x080189FC) | 0x08018A68 | `if (3000 < param_2)` → **err2 bit 11 after timeout**; before that ratio check `float[r9+0xc] < float[r9+0x14] * 8.0`; event 0x35 with channel index |
| **`Protect_Err2Bit7_Ev13`** (0x08018CCE) | 0x08018CDA | err2 bit 7, event 0x13; trigger `float @r4+0x54 < 0`, GPIOD pin 8 is switched off, afterwards retry timer 60000 |
| **`Protect_Err2Bit9_Ev29`** (0x08019ED6) | 0x08019EE2 | err2 bit 9, event 0x29 with `r4[0x23]`; follow-up check `r10[0x19] > 99` |
| **`Inverter_StateMachine_CurrentWindow`** (0x0801710A) | 0x08017124 | checks current window against **+30.0 / −30.0** (float) and calls the HW protection path |
| FUN_080191d0 / FUN_08019364 / FUN_08019c5e | 6 more | additional set sites for **err2 bit 11** with other channel indices |

**This proves:** err2 bit 11 is not a single fault, but a **collective bit for
"limit exceeded for too long"**, which is set from at least 5 channels — the channel appears
only in the event log (parameter of event code 0x35), not in the Modbus register.

### Important Qualification

The created functions begin at a **jump target**, not at the true function start. The
triggering condition therefore partly lies *before* the function start and is missing in the decompilation —
it was supplemented from the disassembly for the cases above. The decompilations also show many
`unaff_*` variables (registers that were set before the artificial entry point). This is to be expected
and not an error, but makes the decompilations only of limited readability.

### Also Noticed

When closing the disassembly gaps, some **literal pools were also interpreted as code**
(e.g. 0x08016900–0x0801691C: `hlt`, `itttt vc`, `mov.vc r0,r0` — nonsensical instructions between
real code). This is the known downside of a blanket disassembly run. Affected are
padding areas between functions; the evaluated code sites were verified by spot check against the
raw bytes. A first attempt to create a function at 0x0801691E hit such a
pool and was removed again.

**State saved.** Functions: 561 → **571**.

---

## 9. Live Cross-Check — from the Existing Scan Logs

The measurement series were already in the project (`Scan_Logs/` (local, raw data)); a new measurement was not necessary.
The time courses of 36100/36101 (error_code1), 36102/36103 (error_code2)
and 36000/36001 (warn_code) were evaluated across all event logs.

### 9.1 The u32 Word Order Is Thereby Clarified

| Event | 36100 | 36101 |
|---|---|---|
| Battery disconnected (`battery_disconnect.csv` (local, raw data), 15:38:12) | **16** | 0 |
| Battery reconnected (15:38:42) | 0 | 0 |

`16` = `0x0010`. My static analysis had yielded **err1 bit 20 = CAN communication fault**
(condition: CAN status flag `0x20000448` == 0). When the battery is disconnected, exactly this
CAN connection to the BMS drops. Bit 20 lies in the **upper** half-word (16 + 4). Thus:

> **Register 36100 = high word (bits 16–31), register 36101 = low word (bits 0–15).**
> `error_code1 = (36100 << 16) | 36101` — analogously 36102/36103 and 36000/36001.

This is decisive for the HA integration and at the same time confirms the statically derived
meaning of bit 20 **independently**.

### 9.2 Confirmed and Newly Evidenced Bits

| Bit | Live evidence | Meaning |
|---|---|---|
| **err1 bit 20** | `battery_disconnect.csv`: 36100 goes to 16, back to 0 on reconnecting | **CAN communication fault** (statically + live confirmed) |
| **err1 bit 7** | `backup_overload_ac_disconnect.csv`, `unter_dod_backup_lässt_sich_nicht_einschalten…csv` (local, raw data), `discharge_under_dod.csv` (local, raw data), operation without battery after power cycle | **Backup/off-grid fault** — the off-grid branch cannot fulfill its function (overload, too little energy, no battery). The file name of the second log states it literally. |
| err1 bits 6 + 9 | `entladen_dc.csv` (local, raw data) and `unklar_watch discharge power 0→2500` (local, raw data): 36101 = 576 (`0x240`) during DC discharging | **observed, not confirmed** — occurs during DC discharge processes; only low word scanned |

Course in `discharge_under_dod.csv`, which shows bit 7 well:

```
10:12:55  36101=128   SOC 10.7 %   Offgrid 239.4 V   -> bit 7 set
10:14:04  36101=0     SOC  0.0 %   Offgrid   0.6 V   -> off-grid off, bit gone
10:16:22  36101=128   SOC 11.1 %   Offgrid 240.3 V   -> bit 7 set again
```

### 9.3 What the Logs Do *Not* Provide

- **error_code2 (36102/36103) and warn_code (36000/36001) stay 0 throughout in all clean logs.**
  The statically found err2 bits (0/1/3/4/7/9/11/16) were thus never triggered — the hardware
  protection paths never kicked in in the entire measurement material so far. That is a good sign
  for the device, but means: these bits can only be evidenced statically.
- The values in `entladen_lang.csv` (local, raw data) / `laden_lang.csv` (local, raw data) for the error registers (496, 500, 586, 601,
  2413, 2524 …) are **not bitmasks**, but the known individual outliers of these two logs
  (see reconciliation report section 7) — e.g. 500 is the value of `grid_pf`. Not usable.
- **err1 bit 21 (BMS fault)** was never observed: on battery disconnection the CAN drops first
  (bit 20), a substantive BMS fault was not present in any test.
  > **OBSOLETE 2026-08-21:** Bit 21 has now occurred live — set with bit 20 clear,
  > triggered by a communication fault in Pack 1 (newly installed pack on factory FW 117
  > versus 118 in the rest of the group).
  > The statically derived meaning is thereby confirmed.
  > See `BMS/VNSD_Pack1_BMS_Error_Analysis_2026-08-21.md`.

### 9.4 Why Bit 20 and Bit 21 Cannot Be Separated by Pulling Cables

The packs of the Venus D are **stacked and connected via contact surfaces**, the inverter
sits on top. There is no pluggable CAN cable — a targeted communication drop while the
battery is running is mechanically not possible. The obvious separation test is therefore ruled out.

### 9.5 The BMS Error Words Are Not Reachable via Modbus at All

Reconciliation of the BMS aggregate struct (`0x20014F8E`, CAN PGN 1801–1804) against the descriptor table:

| Struct offset | Field | Modbus |
|---|---|---|
| +0x18 | `bat_err1` | **no register** |
| +0x1A | `bat_err2` | **no register** |
| +0x1C | `bat_warn1` | **no register** |
| +0x0E / +0x0F | `dischrg_dod` / `key_sleepy` | no register |
| +0x16 / +0x17 | `chrg_flag` / `force_chrg` | no register |

The three BMS error/warning words exist in memory and are output by
`CAN_Battery_Telemetry_Debug_Print`, but have **no descriptor entry**. Via
Modbus a BMS fault is therefore visible only as the **collective bit err1 bit 21** — without
any detail information. This is a real gap in the Modbus interface, not an analysis error.

> **QUALIFIED 2026-08-21:** This applies to the **aggregate** error words. Via the
> **per-pack register `34x08` (`protect2`)** a BMS fault can very well be assigned to an individual
> pack and named via the documented Protect2 bitmask — in the fault case of
> 2026-08-21, `34008 = 0x4000` (bit 14, communication fault) with 0x0000 in all other
> packs. The gap is thus smaller than assumed here.
> See `BMS/VNSD_Pack1_BMS_Error_Analysis_2026-08-21.md`.

### 9.6 Practical Substitute: Scan the Pack Protection Registers Too

Instead of provoking bit 21, the BMS protection can be observed **directly**. Served per pack are:

| Register | Content |
|---|---|
| **34x08** | `protect1` — protection bitmask of the pack |
| **34x09** | `protect2` — second protection bitmask |
| 34x07 | Status/flag field (0..7) |
| 32112 / 30214 | `lock_flag` / `self_check` of the aggregate |

In all logs so far `protect1`/`protect2` are 0 throughout — so there has never been a
BMS protection case. If they are included in the standard scan, every future case is automatically
recorded, **without having to provoke anything**.

In addition, the two cases differ unambiguously in the accompanying picture:

- **Bit 20 (CAN drop):** the BMS registers (32100–32111, 34xxx) freeze or fall to 0
- **Bit 21 (BMS fault):** the BMS registers **continue to deliver plausible values**, but 34x08/34x09 ≠ 0

Thus the separation is possible even without pulling cables — purely from the data picture.

### 9.7 Remaining Useful Measurements

1. **Extend the standard scan** by 36100 **and** 36101 (36100 was missing in `entladen_dc.csv`, there only
   half of the error word is known) as well as 34x08/34x09 for all packs.
2. **Off-grid load just above rated power** → delimits bit 7 from bits 6/9.
3. **Charging at cell temperature < 0 °C** (occurs naturally in winter anyway): the LFP BMS blocks
   charging — the most likely harmless way to see bit 21 and `protect1` for real.

---

## 10. Addendum 21.09.2026: This Bit List Also Applies to the Venus E3

The occasion was the question whether the error texts of the integration are suitable for E3 and Venus A. For the **A**
not — its inverter is a foreign codebase, see
`VNSA_Error_Codes_Micro_FW_Analysis.md`. For the **E3**, however, yes.

### 10.1 The E3 Inverter Is the Same Codebase

Compared were `119_micro_VNSE3-0_inv_app_0119` (119,808 B),
`116_micro_VNSD-0_vd_inv_app_0116` (115,712 B) and `119_micro_VNSA-0_VA_inv_app_0119` (146,432 B):

| | D-116 | **E3-119** | A-119 |
|---|---|---|---|
| Reset handler | `0x080042AD` | **`0x080042AD`** | `0x08006435` |
| Initial SP | `0x20009A70` | `0x20009AC0` | `0x20002400` |
| Error block | `0x200019F4` | `0x20001A28` | `0x20101650` |
| Base loaded via | literal pool | **literal pool** | `movw`/`movt` |
| shared 24-B blocks with D-116 | — | **43.2 %** | **0.4 %** |

That both carry the version number 119 is a coincidence of the separate counting per product line (the A
is long since at 119.3 and 121.1 according to the changelog); the files have different checksums.

### 10.2 The Bits Match, Including Event Codes

| err2 bit | Event | D | E3 |
|---|---|---|---|
| 0, 1, 3, 4 | 0x36 (HW protection latch) | ✓ | ✓ |
| 7 | 0x13 | ✓ | ✓ |
| 8 | 0x31 | ✓ | ✓ |
| 9 | 0x29 | ✓ | ✓ |
| 11 | 0x35 | ✓ (5 sites) | ✓ (7 sites) |
| 16 | – | ✓ | ✓ |

### 10.3 One Difference: The E3 Writes the Warning Word

Section 9.3 states for the D that `warn` is never written. That is true for the D — an
exhaustive scan of all accesses to offset +8…+11 finds **zero** sites. In the E3 image it finds
**one**:

```
0x0801B6D8   ldrb r0,[r6,#0x8] ; orr r0,r0,#0x1 ; strb r0,[r6,#0x8]   ; set warn bit 0
0x0801B7A2   ldrb r1,[r6,#0x8] ; bic r1,r1,#0x1 ; strb r1,[r6,#0x8]   ; clear warn bit 0
```

Both hang on the same flag `0x200002B3`: bit 0 is set as long as the flag is 0, and cleared again
as soon as it is set. After that both branches compute against **different
limit tables** (`vldr.32` from `0x200042AC` resp. `0x20004278`, compared against `0x200002FC`).
This is an operating mode, not a fault — therefore `low confidence` in the integration and
switched off by default.

### 10.4 The Modbus Register Tables Are Model-Independent

When unpacking the descriptor tables (`Scripts/marstek_descriptor_unpack.py` (local, not in the repo)) across all three
Control v150 images:

| Comparison | Entries | only in one | Deviations in type / scale / element size / count |
|---|---|---|---|
| D ↔ E3 | 246 / 246 | none | **0** |
| D ↔ A | 246 / 246 | none | **0** |

The only differences are SRAM source pointers: on the A **identical** to the D, on the E3 shifted by −8
(212 registers), 29 unchanged, four individual cases. Register numbers, types and scales are
thus the same across the models — **what distinguishes a model is exclusively whether there is hardware
behind a register** (no PV → MPPT registers read 0; one storage unit instead of a stack →
pack 2…7 registers read 0) and what the inverter firmware writes into the error words.

The register map `e_v3.yaml` of the integration mapped only **82 of 410** served registers;
the error and warning block was missing entirely and was added on 21.09.2026.
