# Venus A — Error and Warning Codes of the Inverter Firmware

> 🇩🇪 Deutsche Version: [VNSA_Fehlercodes_Micro_FW_Analyse_de.md](VNSA_Fehlercodes_Micro_FW_Analyse_de.md)

> **Correction (2026-10-02):** this analysis was made on an image imported at the wrong base address. The full
> re-analysis on the correct base (0x08006000) is in [VNSA_Micro_FW_v1211_Full_Analysis.md](VNSA_Micro_FW_v1211_Full_Analysis.md) §6.
> The set of bits that exist is confirmed (err1 0-4, 7, 20, 21; err2 5, 6, 7, 8, 9, 11, 13; warn never set), but several
> meanings change: err1 bit 0/1 = grid over-/under-voltage, bit 2/3 = grid over-/under-frequency, err1 bit 20 = BMS
> communication lost, bit 21 = BMS error code >= 4, err2 bit 6 = DC bus > 530 V (570 V in one LLC mode), err2 bit 7 =
> state-machine shutdown (reason 5), bit 9 = invalid state index, bit 11 = state-transition timeout, bit 13 = relay check
> failed. Function names and addresses in this document refer to the old wrong-base program.

**Device:** Marstek Venus A (VNSA-0), micro/inverter MCU (`VA_inv_app_*`)
**Status:** 21 September 2026
**Trigger:** User report from the forum — `fault_status_2_low` (36103) briefly jumped several times
to values around 2330–2380, along with `mppt_error`/`mppt_warning` not equal to 0. No debug logs available.
**Status:** Error word block located, Modbus mapping for Control v150 proven, **all**
set sites conclusively enumerated. **The triggering case is resolved:** not a device fault, but
shifted Modbus responses after a read timeout — the exclusion was achieved via the bit list
(section 9).
**Tools:** Ghidra (re-import + gap disassembly), Python raw scanner over the images.

---

## 1. Starting point

`a.yaml` of the HA integration carries the same `bit_descriptions` for `fault_status*` as `d.yaml`
("Hardware protection latch A/B/C", "Limit exceeded for more than 3000 cycles"). They come from
`VNSD_Error_Codes_Micro_FW_Analysis.md`, i.e. from the **Venus D** micro firmware v116. Whether they
apply to the A was open — `Methodik_und_Meta/Device_Comparison_VNSD_VNSA_VNSE3.md` only covers the
Control, not the inverter.

**They do not apply.** This is now proven, no longer just hypothesized.

## 2. The A inverter firmware is a different code base

Block comparison over all 24-byte windows (step 4) of the images:

| Comparison | shared blocks | Share |
|---|---|---|
| D-Micro 116 ↔ **A-Micro 1211** | 103 | **0.4 %** |
| D-Micro 116 ↔ D-Micro 115 | 17,582 | 62.9 % |
| A-Micro 1211 ↔ A-Micro 119 | 19,192 | 56.4 % |

Within one family two versions share about 60 % of the blocks, across families
practically nothing. Additionally checked: the instruction sequence of `HW_Protection_Latch_To_ErrorBits`
(D-Micro, `0a 79 42 f0 02 02 0a 71 …`) occurs in **none** of the three A images.

This is the opposite of the Control, which is functionally byte-identical between D and A
(`Methodik_und_Meta/Device_Comparison_VNSD_VNSA_VNSE3.md`, section 1). The Control, however, does not generate error codes,
it passes them through — the semantics come from the inverter, and for the A that is a different device.

**The Venus E3 is the opposite case:** its inverter (`inv_app_0119`) shares 43.2 % of the blocks with
the D image, has the same reset handler `0x080042AD`, also reaches the error block via a
literal pool and carries a bit-exact identical assignment including event codes. The D texts
therefore apply to the E3 — only not to the A. Details in `VNSD_Error_Codes_Micro_FW_Analysis.md`, section 10.

### Binary fingerprint of the A-Micro images

| Version | File | Size | Flash | Initial SP | Reset handler | MD5 |
|---|---|---|---|---|---|---|
| **1211** | `1211_micro_VNSA-0_VA_inv_app_1211_0520_ota_193621.bin` | 146,432 B | `0x08000000–0x08023BFF` | `0x20002400` | `0x08006435` | `c1d5ff1c…` |
| 1193 | `1193_micro_VNSA-0_VA_inv_app_1193_0512_ota_092355.bin` | 146,432 B | same | same | same | `78683f90…` |
| 119 | `119_micro_VNSA-0_VA_inv_app_0119_0326_ota_144725.bin` | 146,432 B | same | same | same | `7e4cca4a…` |

Fixed OTA slot size as with D-Micro and BMS. **Open:** the initial SP is at `0x20002400`,
but the globals are at `0x2010xxxx`/`0x2012xxxx`. Either two RAM banks or a different
controller vendor than the D — not pursued further, irrelevant for the error codes.

## 3. Ghidra setup

A raw binary cannot be imported via ReVa's `import-file` (batch import does not recognize a
format). Instead via script: create a `ProgramDB` with `ARM:LE:32:Cortex`/`default`,
`createInitializedBlock("flash", 0x08000000, …)` from the file, plus an uninitialized
`sram` block from `0x20000000` (0x8000) — the latter only so that Ghidra creates references
and `DAT_` symbols for the globals, which the D-Micro import does not have.

Afterwards as with the D: auto-analysis, vector table as function entry points, then
gap disassembly in Thumb mode in several passes.

| | after auto-analysis | after gap run |
|---|---|---|
| disassembled | 27.6 % (40,410 B) | **83.4 %** (122,070 B) |
| Functions | 117 | **294** |
| in functions | — | 58.1 % |

Program in the project: `/VNSA/Micro/Micro_VNSA_1211_VA_inv_app_1211.bin`.

The known downside of the blanket run (literal pools as code) applies here too; the
evaluated sites were verified against the raw bytes.

## 4. Where the error words are located

Three `u32` starting at SRAM **`0x20101650`**:

| Offset | Address | Contents |
|---|---|---|
| +0x00 | `0x20101650` | err1 |
| +0x04 | `0x20101654` | err2 |
| +0x08 | `0x20101658` | warn |

The order in memory is the same as for the D (`0x200019F4`/`F8`/`FC` = err1/err2/war1).

## 5. Proof of the Modbus mapping

`FUN_08002a80` (telemetry builder, called from `FUN_08013290`) fills a buffer starting at
`0x2012046c` and sends **48 bytes** via `FUN_080145b0(handle, 0x30, &DAT_2012046c)`:

```
_DAT_20120470 = _DAT_20101658;   // Payload +0x04  <- warn
_DAT_20120474 = _DAT_20101650;   // Payload +0x08  <- err1
_DAT_20120478 = _DAT_20101654;   // Payload +0x0C  <- err2
```

That this payload is 1:1 the Control struct `0x20014E9C` is shown by the field comparison — not only
the three error words, but the entire rest:

| Payload | Type | Control struct (from `Modbus_RS485_TCP/VNSD_Descriptor_Open_Registers_Ghidra_Finding.md`, §2a) |
|---|---|---|
| +0x04 / +0x08 / +0x0C | u32 | +0x04 `warn_code`, +0x08 `error_code1`, +0x0C `error_code2` |
| +0x14 … +0x27 | consistently u16 | +0x14 `off_grid_volt` … +0x26 `min_power` |
| +0x28, +0x2C | u32 | +0x28 `chrg_energy`, +0x2C `dischrg_energy` |

The u16/u32 boundaries coincide exactly. Therefore:

| SRAM | Control struct | Modbus |
|---|---|---|
| `0x20101658` warn | +0x04 | **36000 / 36001** |
| `0x20101650` err1 | +0x08 | **36100 / 36101** (alias 37013) |
| `0x20101654` err2 | +0x0C | **36102 / 36103** (alias 37015) |

The descriptor table of the A Control is identical to the D at these locations — verified by
unpacking (`Scripts/marstek_descriptor_unpack.py` (local, not in the repo)) A-v150 (`0x20014EA0/A4/A8`, like D-v150)
and A-1509 (`0x200176A0/A4/A8`, only layout drift). 246 entries in both cases.

**Independent cross-check on the device:** the reporting user states 36102 (high word) = 0 and
36000/36001 = 0. Both are structurally mandatory — for err2 there is not a single set site
above bit 13, and for `warn` there is **no** set site at all. The mapping is thus
also confirmed from the device side.

## 6. The mechanism

Identical to the D, only with a different event logger and different codes:

```
  <condition>            ; usually: measured value against limit, then cycle counter against trip time
  movs r0,#<event code>  ; optionally r1..r3 = measured values or channel index
  bl   0x080103e0        ; event logger
  movw r0,#0x1650
  movt r0,#0x2010
  ldrb r1,[r0,#<byte offset>]
  orr  r1,r1,#<bit>      ; and/bic when clearing
  strb r1,[r0,#<byte offset>]
```

### Why the bit list is complete

The base address is loaded **exclusively** via a `movw`/`movt` pair — in the whole
image there is no literal pool word with the value `0x2010164c`/`50`/`54`/`58`/`5c`. A raw scan over the
bytes after `movw rD,#0x1650` + `movt rD,#0x2010` finds for all 13 possible target registers
**26 sites in 1211** — exactly the 26 that Ghidra also knows. Additionally the idiom
`ldrb/orr|bic/strb` was searched raw over the whole image (55 hits, all base registers): among them
no further site with offset 0–11 on this block.

The enumeration is thus not "everything Ghidra found", but conclusive.

Two subtleties that cost hits when reproducing this:

- The final `strb` is missing at some sites because the compiler jumps to a
  shared store instruction instead (`orr r1,r1,#0x1 ; b …`). Anyone who forces `strb` loses,
  among others, err1 bits 0–2.
- A single site loads the base **after** the `orr` (setting err1 bit 21, `0x08002314`:
  `orr r0,r0,#0x20 ; movw r1,#0x1650 ; movt r1,#0x2010 ; strb r0,[r1,#0x2]`). The raw scanner
  does not find it; it came from Ghidra. Telltale sign: a bit appears in the scanner output
  only as `clear`.

## 7. The bits

All three archive versions (0119, 1193, 1211) have **the same bit assignment** — only the addresses of the
set sites and two event codes shift (0119: 0x42/0x47 instead of 0x43/0x48).

### err2 — 36102 / 36103

| Bit | Event | Trigger | Confidence |
|---|---|---|---|
| **5** | 0x43 | Protection shutdown; additionally sets the flags `0x201004dd` and `0x20120511`, logs three measured values | medium |
| **6** | 0x35 | Measured value **> 568.0** (`0x440E0000`, float) — DC link/DC overvoltage; two set sites (`0x08004db6`, `0x08004e1a`) | medium |
| **7** | 0x15 | State machine, channel parameter 5; in the alternative branch peripheral `0x40009444` bit 6 is set instead | low |
| **8** | 0x31 / 0x32 *(logged after* setting) | **Two integrators, see below** | **high** |
| **9** | 0x29 | Comparisons against 100.0 and ±500.0 (float) | low |
| **11** | 0x36 | **Collective bit "limit exceeded for too long"**, 4 set sites, see below | high |
| **13** | 0x48 | Cycle counter ≥ 20001 (`0x4E21`), logs `[0x2010056c+0xC]` | medium |

Bits 0–4, 10, 12, 14–31: **no set site**. err2 can arithmetically only assemble values from
{32, 64, 128, 256, 512, 2048, 8192}.

#### Bit 8 — the overload integrator (`FUN_080149a8`, 0x08014a7e / 0x08014b12)

Event 0x31: an accumulator `_DAT_20101578` is modified on each pass, depending on the measured value
`_DAT_20121b8c` (current):

| Measured value | Step |
|---|---|
| > 7.85 | +60 |
| > 7.2 | +10 |
| > 6.6 | +1 |
| > 6.54 | −1 |
| > 3.27 | −240 |
| ≤ 3.27 | −1800 |

If < 0 it is clamped to 0, at **> 600,000** err2 bit 8 is set and event 0x31 is logged with
current, voltage and counter value. This is an i²t-like overload integral: short peaks
are tolerated, continuous load just above the rated power builds up after a while.
(Side finding: at > 60,000 `DAT_201233a5` is set — apparently a pre-warning threshold.)

Event 0x32: only in operating modes `0x18`/`0x04`. If `_DAT_20121b94` is **below 207.0**,
a counter runs up to **201**, then the same bit 8 is set. Grid undervoltage.

Bit 8 thus does not mean "a specific fault", but **"operating limit violated over an extended period"**
— overload or undervoltage.

#### Bit 11 — collective bit with four channels

All four set sites have the same form: one condition, one cycle counter, one threshold.

| Address | Threshold | Event parameters (r1, r2) |
|---|---|---|
| `0x08007912` | 101 (`0x65`) | 0x0B, 0x0D |
| `0x0800926e` | 5001 (`0x1389`) | 0x03, 0x05 |
| `0x0800947a` | 10001 (`0x2711`) | 0x0B, 0x07 |
| `0x08009684` | 300,001 / 6,000,001 | Channel byte from `[r5]` |

Structurally the same as err2 bit 11 on the D — only the D counts 3000 cycles, the A 101 / 5001 /
10001 / 300000. **The text "Limit exceeded for more than 3000 cycles" is therefore numerically
wrong for the A**, the principle is correct. Which channel triggered is, as on the D, only found in the
event log (parameters of event 0x36), not in the Modbus register.

### err1 — 36100 / 36101

Bits 0–4 and 7 are all set in `FUN_0800e4a0`. The function is the **grid protection**: it
compares `_DAT_20121b44` against 47.0 / 53.0 / 57.0 / 63.0 (frequency windows 50 and 60 Hz) and
`_DAT_20100640` against 170.0 (voltage), operates a ±2.0 Hz window around `_DAT_20100690` with a
2000-cycle counter (event 0x30) and otherwise works throughout with limits from a
configuration struct plus trip counters.

| Bit | Event | Trigger | Confidence |
|---|---|---|---|
| **0** | 0x2c | Frequency **above** limit `_DAT_20120410` for longer than the counter in `[r7+0x0c]` | medium |
| **1** | 0x2c | Frequency **below** limit `_DAT_2012040c` for longer than `[r7+0x1c]` | medium |
| **2** | 0x2a | Limit violation, counter against `[r7+0x24]`, logs two measured values | medium |
| **3** | 0x2a | same, counter against `[r7+0x2c]` | medium |
| **4** | 0x2d | `|_DAT_20100570|` **> 430.0** (`0x43D70000`) | medium |
| **7** | 0x2e | Counter ≥ 4, logs three measured values | low |
| **20** | 0x1a | set `0x08002358`, cleared `0x08002344` (`and #0xef`) — both in `FUN_080022a0`, the communication monitoring | medium |
| **21** | 0x0c when clearing | set `0x08002314` (`orr #0x20`), cleared `0x080022e4` (`bic #0x20`), same function | medium |

Bits 5, 6, 8–19, 22–31: no set site.

The parallel to the D is striking: there, err1 bit 20 = CAN communication fault and bit 21 =
BMS reports fault. On the A the same two bits are in a communication monitoring function.
**Not verified** — the hypothesis is obvious, but a device test is missing.

### warn — 36000 / 36001

**Not a single set site in the entire image.** The word is filled, transmitted and output,
but never written. On the A, `warn_code` is thus permanently 0 — unlike on the D, where it would
at least theoretically be writable.

## 8. Acknowledgement

Two sites clear the **lower halfword** of err2 and leave the upper one in place:

```
0x0800961c   ldrh r1,[r0,#0x6] ; lsls r1,r1,#0x10 ; str r1,[r0,#0x4]   ; err2 = (err2 & 0xFFFF0000)
0x08005630   movw r6,#0xffff   ; bics r3,r6       ; str r3,[r2,#0x4]   ; err2 &= ~0xFFFF
```

Since nothing above bit 13 is set anyway, "clear lower halfword" on the A is effectively
"clear err2 completely".

> These sites at first looked like the explanation for the reported values falling back to 0
> after 7–8 seconds. That was a false lead — no fault was ever present there, and the return
> to 0 was simply the next read without response offset (section 9.3). The
> acknowledgement path itself is unaffected by this and remains valid.

## 9. The reported values were not error codes at all

**Resolved.** 36103 never indicated a fault — the integration read shifted
Modbus responses. The path to this conclusion is in 9.1 to 9.4, because the firmware analysis
delivered the decisive exclusion.

From the user report (36103, each present for 7–8 s):

| Value | Hex | Bits |
|---|---|---|
| 2328 | `0x0918` | 3, 4, 8, 11 |
| 2337 | `0x0921` | 0, 5, 8, 11 |
| 2349 | `0x092D` | 0, 2, 3, 5, 8, 11 |
| 2350 | `0x092E` | 1, 2, 3, 5, 8, 11 |
| 2380 | `0x094C` | 2, 3, 6, 8, 11 |

**Bits 0–4 have no set site in err2** — in none of the three archived versions. They
exist only in err1. err2 cannot take these values.

This exclusion was exactly the lever: as long as you consider the bits real, you search for a
device fault. As soon as the firmware says these bits do not exist, it is clear that the value
does not come from err2 — and the search shifts to the path leading there.

### 9.1 The device's firmware is known — and is in the archive

The device's combined version sensor shows `V150.9.1193.0.1105`. Resolved (for the formatting
see section 10, item 5):

| Component | Value | Archive |
|---|---|---|
| Control/EMS | 150.9 | `1509_control_VNSA-0_ems_app_1509_0908_093804.bin` ✔ |
| Inverter | **119.3** | `1193_micro_VNSA-0_VA_inv_app_1193_0512_092355.bin` ✔ |
| MPPT | 0 | not present |
| BMS | 110.5 | `1105_bms_VNSA-0_VA50A_APP_V1105_ota_144658.bin` ✔ |

**The obvious explanation "firmware newer than the archive" is thus ruled out.** The device runs
1193, and its bit assignment has been fully evaluated here — identical to 0119 and 1211.

That also invalidated the second suspicion, that **Control 1509** might fill the struct differently.
It was not unfounded — 1509 rebuilt the area, `work_mode` (30010) is in v150 still
on the standalone global `0x20000144`, in 1509 however at `0x200176D4`, i.e. **inside**
the struct at +0x38. The error fields themselves, however, moved as a block by +0x2800
(36000/36100/36102 → `0x200176A0/A4/A8`, 30000–30007 and 32300/32302 with the same delta), and
the debug log solves the case without this hypothesis.

### 9.2 The debug log

The reporting user supplied an HA debug log from 17 to 21 September 2026
(**not stored in the project** — personal grid and consumption data). It was evaluated
on three questions: time correlation, value range, connection state.

**Time correlation.** Each of the five events is preceded 3–4 seconds earlier by a failed
read access to **register 36100** — the first register of exactly this block. Four of five
additionally have a pymodbus transaction ID offset:

```
2026-09-20 22:21:27.296  Failed to read register 36100 (0x8D04) after 1 attempt(s)
2026-09-20 22:21:28.401  ERROR: request ask for transaction_id=1487 but got id=1486, Skipping.
2026-09-20 22:21:31      fault_status_2_low = 2350
```

| Event | TID offset | Read failure 36100 |
|---|---|---|
| 2328 @ 21.09. 11:35:35 | −3 s | −3 s |
| 2380 @ 21.09. 13:50:53 | – | −3 s |
| 2350 @ 20.09. 22:21:31 | −3 s | −3 s |
| 2349 @ 20.09. 22:06:11 | −3 s | −3 s |
| 2337 @ 20.09. 20:21:34 | −3 s | −4 s |

TID offsets occur 1507 times in the log in 104.8 h, i.e. one every 250 s. The probability
of a chance hit in the ±15 s window is 11.6 %; four of five hits, and every
time at exactly −3 s rather than scattered, rule out chance.

**Value range.** In the debug window (21.09., 21:23–21:28) `ac_voltage` (32200) reads values from
**2322 to 2333**. The five "error values" 2328/2337/2349/2350/2380 lie in exactly this band,
**2328 is literally one of the measured `ac_voltage` values**. No other polled register
lies in this range.

**Connection state.** ~13,000 integration warnings and 130–1200 errors per day; 1209 times
"Modbus client not connected, attempting reconnect before write to register 42021". A
zero-feed-in automation writes `set_discharge_power` (42021) every second while polling
in parallel. The device's Modbus stack cannot keep up — known behavior, see the
`VNSD_FW_Bug_*` documents under `Modbus_RS485_TCP/`.

### 9.3 The sequence

1. The block reader for 36100–36103 runs into a timeout; the device's response stays in transit.
2. The integration falls back to **single reads** of the four registers.
3. The delayed response arrives, the request→response assignment is offset by one position.
   Because `ac_voltage` and `fault_status_2_low` are both single-register reads, the length matches;
   only the transaction ID would catch it — and that is exactly what fails.
4. `fault_status_2_low` takes over the value of a different single read.
5. The next clean cycle 7–8 s later restores 0. That looked like self-acknowledgement,
   but was only the first read without offset.

`coordinator.py` already knows the danger — before the fallback `_async_recover_half_open` runs with
the comment "every fallback read below starts on the same suspect socket". That only takes effect with
half-open sockets, not with one in which a valid, delayed response is still stuck.

### 9.4 The MPPT values

Presumably the same mechanism, **not proven** — at the times in question (20/21.09. afternoon)
no debug log was running yet. Supporting this: `mppt_version` is 0, the device has no
MPPT stage at all that could report anything; and `set_discharge_power` (42021) is continuously written by the same
automation with values that are at 700–770 W in the evening — at midday
1098/1100/1367 W is exactly the expected range.

### 9.5 What this means for the bit lists

The interpretation from section 7 remains valid, it was just not applicable here: there was no fault.
The value of this analysis in this case lies in the **exclusion** — without the complete bit list
one would have continued to interpret the values as hardware protection trips and searched at the wrong end.

## 10. Consequences for the integration

As of 21.09.2026, items 1, 2, 5 and 7 are implemented in the project *Marstek Modbus Suite*;
3 and 4 are mere findings, 6 is open.

1. **`a.yaml` must not carry the D bit texts.** *(implemented)* "Hardware protection latch A/B/C" does not exist in the
   A code, and the 3000 cycles are 101/5001/10001/300000 on the A. Either remove the
   `bit_descriptions` for `fault_status*` in `a.yaml` or replace them with the tables from
   section 7.
2. **`warn_code` (36000/36001) is dead on the A.** *(implemented: no bit listed)* Can stay disabled as an entity.
3. **The high word of err1 and err2 is almost empty on the A** — err2 completely, err1 only bits 20/21.
4. The word order (36102 = high, 36103 = low) applies to the A as to the D; the descriptor table
   and the FC03 serializer are identical.
5. **The combined version sensor only formatted the EMS part.** *(implemented)* `MarstekVersionSensor.
   calculate_value` (`sensor.py`) applies the tenths rule "four digits → `x // 10` . `x % 10`"
   exclusively to `ems`; `vms`, `mppt` and `bms` are appended raw. Marstek, however, numbers
   **all** components the same way, as the firmware archive shows:

   | Component | Archived values | meant |
   |---|---|---|
   | Control | 148, **1487**, 149, 150, **1508**, **1509** | 148, 148.7, 149, 150, 150.8, 150.9 |
   | Inverter (A) | 119, **1193**, **1211** | 119, 119.3, 121.1 |
   | BMS (A) | 109, **1105** | 109, 110.5 |
   | Inverter (D) | 115, 116 | unchanged |
   | BMS (D) | **1177**, 118 | 117.7, 118 |

   That is why the reported device shows `V150.9.1193.0.1105` instead of "150.9 / 119.3 / – / 110.5".
   Two consequential bugs: the dots no longer separate unambiguously (`150.9.119.3.110.5` would not be better
   readable, a different separator is needed here), and the MPPT part is always 0 on the A,
   because the device has no MPPT stage — the mode `ems_vms_mppt_bms` produces a
   meaningless zero there. Belongs in the Modbus Suite project, merely recorded here.
6. **Reconnect hard after a read timeout** *(open)*, not just check for half-open. Otherwise the
   entire single-read fallback runs on a socket in which a delayed response is still stuck, and
   returns values of foreign registers (section 9.3). This is the cause of the triggering case.
7. **Plausibility check on the error registers.** *(stored as a comment in `a.yaml`, the check itself open)* From this analysis it is known which bits
   exist at all — on the A `err1 = 0–4, 7, 20, 21` and `err2 = 5, 6, 7, 8, 9, 11, 13`, on the D
   the list from `VNSD_Error_Codes_Micro_FW_Analysis.md`. A value with other bits is by definition
   a misread and should be discarded rather than displayed. In the triggering case this one
   check would have caught all five false reports.

## 11. Open

1. **Analyze Control 1509** — no longer urgent now that the triggering case has been cleared up, but
   open: which function fills the struct `0x2017769C` from the inverter's 48-byte payload
   and whether it reads the same offsets as the v150 Control. The descriptor pointers
   have shifted uniformly by +0x2800, but `work_mode` has slipped into the struct (section 9.1) —
   the mapping "36102/36103 = err2" is therefore strictly speaking only proven for **v150**
   (section 5). The image is in the archive but not yet in Ghidra.
2. **`mppt_error` / `mppt_warning` (37023/37024)** — the triggering case is with high probability
   the same misreading (section 9.4); the semantics of the registers remain open independently of that.
   The A's descriptor table maps them, as on the D, to inverter struct +0x02 and +0x06 (`0x2001776E`/`0x20017772` at 1509), filled
   via `Modbus_StoreRegisterSlot` from the CAN slots. A bit-to-text table exists in no
   firmware, for no model. Additional finding from the D logs:
   `Scan_Logs/battery_disconnect_and_then_power_cycle_without_battery.csv` (local, raw data) shows 37024 in a
   single sample at **1345 = 0x0541** with otherwise constant 0 and all fault registers at 0
   — the same value family as the 1367 of the A user. A brief spike of
   `mppt_warning` thus also occurs on the D, without an accompanying fault.
   **New:** on the reported device `mppt_version` (30205) is **0** — there is no
   MPPT stage there. Nevertheless 37023/37024 return non-zero values. What the CAN slots
   write into the inverter struct on the A without an MPPT controller is open; a stale
   or foreign-assigned slot is the most obvious hypothesis.
3. **Event codes** — the logger `0x080103e0` writes a code plus up to three parameters into a
   ring buffer. Where it ends up and whether it can be read out was not followed up. That would be the way to
   the channel indices behind err2 bit 11.
4. **Missing flash tail.** The debug print at `0x080169c8` reads the three words via
   `ldm.w r0,{r1,r2,r3}` and loads a format string from `0x08024419` — which lies beyond the
   image end `0x08023BFF`. As with the D, the tail is missing, so there are no plain-text names from the
   firmware.
5. **Internal fault word `0x201005b0`.** `FUN_08020360` systematically sets bits 0–15 and
   20–25 there (22 set sites), `FUN_08014c68` clears it. It is **not** copied into the telemetry
   and is not visible via Modbus. Presumably the protection shutdown capture of the
   power stage. Unverified, but the most obvious next chunk, should a
   fault word with densely populated lower bits still be needed.

---

## Appendix: Tools

The raw scanner that extracts the bit lists from an image without Ghidra is available as
`Scripts/vnsa_micro_error_bits.py` (local, not in the repo). It searches for `movw`/`movt` pairs on a given base address,
decodes the following `ldrb` + `orr|bic|and #imm`, resolves the Thumb `ModImm` immediate, and
assigns byte offset plus mask to a word number; `bic`/`and` are counted as all-clear.
The event code is searched via the nearest preceding `bl`, so that nothing wrong is reported for sites without a
logger call.

```
python3 Scripts/vnsa_micro_error_bits.py <firmware.bin>              # bit list
python3 Scripts/vnsa_micro_error_bits.py <firmware.bin> --find-bases # find base address
```

`--find-bases` reports `0x20101650` (16 sites) and `0x201005b0`
(12 sites, the internal word from section 11.5) in all three A images. Applied to the D Micro images it returns
**0 hits** — they do not use the idiom, which confirms the finding from section 2 once more.
