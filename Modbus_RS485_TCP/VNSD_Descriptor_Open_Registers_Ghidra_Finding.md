# Open Descriptor Registers — Ghidra Analysis (Control FW v150)

> 🇩🇪 Deutsche Version: [VNSD_Descriptor_Offene_Register_Ghidra_Befund_de.md](VNSD_Descriptor_Offene_Register_Ghidra_Befund_de.md)

**Device:** Marstek Venus D (VNSD-0), STM32 Cortex-M
**Firmware:** `VNSD-0_app_0150_0805_115146.bin` (v150), Ghidra/ReVa, 1618 functions
**Date:** August 14, 2026
**Starting point:** 121 registers with a confirmed type/scale, but unknown meaning.
**Result:** **0 registers without an assignment.** 100 high · 101 medium · 16 low · 29 hypotheses.
After cross-checking against BMS FW v118 and Micro FW v116 (section 7) and evaluating the
version debug print (section 8).

---

## 1. Methodology

SRAM (0x2000xxxx) is **not** a memory block in Ghidra — the descriptor source pointers are not
present as data. The breakthrough came via two levers:

1. **movw/movt reference resolution.** Cortex-M loads SRAM addresses as `movw/movt` immediates; Ghidra
   reconstructs references from these with target address **and** type (READ/WRITE). A script over all
   instructions yields the writer/reader map per SRAM address.
2. **Struct base back-calculation from debug print functions.** The FW contains `printf` debug output
   with **plaintext field names** in `.rodata`. From `[rBase,#offset]` plus the target address resolved
   by Ghidra follows the struct base — this makes every field offset assignable to a register address.
   Pairing is done positionally: the n-th resolved SRAM access ↔ the n-th format string.

### Two pitfalls (documented so they do not recur)

- **`mem.getBytes(addr, bytearray)` does not work in PyGhidra/Jython** — the Python buffer
  stays filled with zeros, and the script silently reports "nothing found". `mem.getInt()` /
  `mem.getByte()` must be used instead. An initial literal scan produced a
  false negative result because of this.
- **Decompiler symbols such as `_DAT_080365a4` are artifacts** that overlap an adjacent
  string ("Globals starting with '_' overlap smaller symbols"). The real struct base comes
  from reference resolution, not from the symbol name.

### Finding: the image is incomplete

314 flash pointers in the image point **past the end of the image** (0x0805EFFF), of which **287 are
compactly located in 0x0805F000–0x08062FFF**. That is exactly where the cloud `snprintf` format strings
are located (`DAT_0801f82c` → 0x08060524, `DAT_08024ef0` → 0x08060830). The available `.bin` is thus
missing a **~16 KB rodata/string blob**. A wrong load base was ruled out (at −0x10000 there is
Thumb code, not a string). **This is the only reason the pack field names remain open.**

---

## 2. Four identified source structs

### 2a. Inverter/grid telemetry — base `0x20014E9C` (v149.2: 0x20014E90, +12 drift)
Source: `Inverter_Telemetry_Debug_Print` (0x08036BD4), 28 printfs with plaintext names.

| Off | SRAM | Field (FW string) | Register |
|---|---|---|---|
| +0x00 | 20014E9C | inv_state | **35100** |
| +0x01…03 | 9D/9E/9F | buz_state, chrg_flag, back_func | – |
| +0x04 | 20014EA0 | warn_code (u32) | **36000** |
| +0x08 | 20014EA4 | error_code1 (u32) | **36100 / 37013** |
| +0x0c | 20014EA8 | error_code2 (u32) | **36102 / 37015** |
| +0x10 | 20014EAC | grid_volt (0.1 V) | 30004 / 32200 / 32201 |
| +0x12 | 20014EAE | grid_pf (0.1 Hz) | 32204 |
| +0x14 | 20014EB0 | off_grid_volt (0.1 V) | 30005 / 32300 / 32301 |
| +0x16 | 20014EB2 | grid_permit | – |
| +0x18 | 20014EB4 | grid_sample_power (1 W) | 30006 / 32202 / 37004 |
| +0x1a | 20014EB6 | off_grid_power (1 W) | 30007 / 32302 |
| +0x1c | 20014EB8 | bat_sample_power (1 W, i16) | 30001 / **32102** |
| +0x1e | 20014EBA | bat_sample_volt (0.1 V) | 30000 |
| +0x20 | 20014EBC | env_temp (0.1 °C) | 30002 / 35000 |
| +0x22 | 20014EBE | radiator_temp (0.1 °C) | 30003 / 35001 / 35002 |
| +0x24/26 | EC0/EC2 | max_power, min_power | – |
| +0x28/2c | EC4/EC8 | chrg_energy, dischrg_energy (1 Wh) | – |

Additional blocks from the same function: `0x2000015C` hard_ver (not served), **`0x2000015E` soft_ver =
30202**, **`0x20000160` boot_ver = 30203**, `0x20000162` dev_state; `0x20000144` work_mode = 30010,
+1 sleep_flag, +2 bat_mode.

### 2b. BMS aggregate — base `0x20014F8E` (CAN PGN 1801–1804)
Source: `CAN_Battery_Telemetry_Debug_Print` (0x080364C4).

| Off | Register | Field | Unit |
|---|---|---|---|
| +0x00 | **32100** | bat_volt | **10 mV** |
| +0x02 | **32101** | bat_curr | 100 mA |
| +0x04 | **32108** / 35010 | bat_temp | 0.1 °C |
| +0x06 | 32104 | bat_soc | 0.1 % |
| +0x08 | **32105** | bat_energy | Wh |
| +0x0a | **32109** | bat_total_nb (pack count) | – |
| +0x0b | **32110** | bat_online_mask (u16) | bitmask |
| +0x0d | **32111** | work_bat_idx | – |
| +0x10 | **35110** | chrg_volt | 0.1 V |
| +0x12 | **32106** / 35111 | chrg_curr | 100 mA |
| +0x14 | **32107** / 35112 | dischrg_curr | 100 mA |
| +0x1e | **32113** / 30109 | factory_test | flag |
| +0x1f | **32112** | lock_flag | flag |
| +0x20 | **30214** | self_check | flag |

### 2c. Inverter/MPPT — base `0x20014F4C`
Source: `MPPT_Debug_Print` (0x08036884). Populated via `Modbus_StoreRegisterSlot`
(CAN fc 0x03, 8 slots × 8 bytes).

| Off | Register | Field |
|---|---|---|
| +0x02 | **37023** | Mppt_Error |
| +0x06 | **37024** | Mppt_Warning |
| +0x08…1e | 30020–30040 | PV1–4 Vol/Cur/Pow |
| +0x2c | **37021** | PV_Year_Cap (u32, 10 Wh) |
| +0x30 | **30028** | bat_vol (0.1 V) |
| +0x32 | **30029** | bat_cur (0.1 A) |
| +0x34 | **30036** | base_vol (0.01 V) |
| +0x38…3e | 30031–30034 | Slot 8, not output by MPPT_Debug_Print |

### 2d. Pack struct — base `0x20014FB4`, stride 0x60
Confirmed threefold (`Schedule_MinMax_PerGroup_Calc`, `BLE_Build_BMS_Data_Response`,
`BLE_Build_DevelopModeInfo_Response`). `Schedule_MinMax_PerGroup_Calc` iterates 6 packs, reads the
16 cell voltages starting at Pack+0x20 (= reg 34x18 ✓) and stores max-index/min-index/max/min per pack.
The open fields 34x03/04/07/08/09/11/12/17 are precisely located; the names are in the missing
flash tail.

---

## 3. CAN telemetry dispatch (Protocol AA)

`CAN_FrameDispatcher` → `Telemetry_Register_Dispatcher` dispatches by CAN ID low byte:
0x10–0x14 Protocol_AA_Set*, 0x40–0x43 energy counters, 0x54/0xC1/0xCB/0xCE Telemetry_Store_*.

**The 38000 block is thereby fully resolved** — four CAN frames, payload 1:1 as 16-bit registers:

| CAN ID | Target SRAM | Length | Register |
|---|---|---|---|
| 0x40 | 0x20000168 | 8 B | 38000–38003 |
| 0x41 | 0x20000170 | 6 B | 38004–38006 |
| 0x42 | 0x20000176 | 8 B | 38007–38010 |
| 0x43 | 0x2000017E | 8 B | 38011–38014 |

---

## 4. Corrections to the previous register map

| Register | Previously | Firmware truth |
|---|---|---|
| **32102** | float32 (type conflict with 30001) | **Conflict resolved:** source 0x20014EB8 is i16 `bat_sample_power(1W)`. The float32 statement of the descriptor is the wrong side. |
| **36000** | alarm_status | `warn_code` |
| **36100 / 37013** | fault_status_lo | `error_code1` |
| **36102 / 37015** | fault_status_hi | `error_code2` |
| **30201** | ems_sub_version | **`ems_boot_ver`** — Control bootloader version |
| **30202** | vms_version | **`vns_ver`** — VNS (micro-inverter), not "vms". Writer `Telemetry_Store_RegCB` (CAN ID 0xCB) |
| **30203** | vms_sub_version | **`vns_boot_ver`** — ditto. hard_ver @0x2000015C is not served |
| **30205** | bms_sub_version | **`mppt_ver`** — MPPT firmware version (u16), see section 8 |
| **32101** | dc_current, truncation assumed | `bat_curr(100mA)` from the BMS CAN aggregate — no truncation |
| **32100** | dc_voltage | `bat_volt` in **10 mV** (not 0.1 V) |
| **32114 / 30110** | sensor assumed | is **written by the write handler** → config setpoint |

---

## 5. Verification against live scan (`control_150_vns_116.csv` (local, raw data))

| Reg | Raw | Interpreted | Expectation | ✓ |
|---|---|---|---|---|
| 32100 | 5372 | 53.72 V (10 mV) | battery voltage | ✓ |
| 30028 | 533 | 53.3 V (0.1 V) | battery voltage, inverter side | ✓ |
| 35110 | 576 | 57.6 V (0.1 V) | charge voltage limit LFP | ✓ |
| 34002 | 887 | 88.7 % | pack SOC | ✓ |
| 30036 | 219 | 2.19 V (0.01 V) | internal reference | plausible |

---

## 6. Remaining points

1. **Missing flash tail (0x0805F000–0x08063000).** Contains the cloud format strings and thus the
   exact names of the 60 pack fields. Acquisition: a more complete image from the FW archive or
   an SWD/OpenOCD dump. The "Marstek FW Archive" folder was not connected during this session.
2. **30031–30035:** inverter struct tail (slot 8 or fc 0xff slot 2), no plaintext name in the image.
3. **30212 (0x20000138):** no resolved writers/readers.
4. **32202:** `elem_size` 4 with type i16 remains a descriptor inconsistency.

---

## 7. Cross-FW verification against BMS v118 and Micro v116

Both neighboring firmwares are loaded in Ghidra and provide independent confirmation —
they are the **senders** of the data that Control merely forwards.

### 7a. Version block: VNS, not "VMS"

`Telemetry_Store_RegCB` writes block `0x2000015C` from a **CAN frame (ID 0xCB)** —
the values thus come from an external module, not from Control itself. Together with the
reading `Inverter_Telemetry_Debug_Print`, the assignment to the **micro-inverter (VNS)** follows.
The register staggering is thereby consistent:

| Register | Module | Field |
|---|---|---|
| 30200 / 30201 | **EMS** (Control) | ems_version (flash 0x08010000) / ems_sub_version |
| 30202 / 30203 | **VNS** (micro-inverter) | soft_ver / boot_ver |
| 30204 / 30205 | **BMS** | bms_version / bms_sub_version |

### 7b. BMS v118 fully confirms the aggregate struct

The BMS FW contains exactly named transmit functions that correspond 1:1 to the Control debug strings:

| BMS function | CAN PGN | Control struct 0x20014F8E |
|---|---|---|
| `CAN_TX_PF1_PackMeasurements` | 1801 | bat_volt, bat_curr, bat_temp, bat_soc |
| `CAN_TX_PF2_Capacity` | 1802 | bat_energy, bat_total_nb, online_mask, work_bat_idx |
| `CAN_TX_PF3_ChargeDischargeLimits` | 1803 | chrg_volt, chrg_curr, dischrg_curr, chrg_flag |
| `CAN_TX_PF4_ProtectWarnings` | 1804 | bat_err1/2, bat_warn1, factory_test, lock_flag |

This means all 14 registers of the aggregate struct are evidenced **on both sides**.

### 7c. Pack struct: frame layout closes the gap

`CAN_TX_PerPack_10Msgs` (BMS, 1868 B) sends **10 frames per pack**, ID `0x18[Grp]AA[PackID]`,
from a BMS-internal pack array with **likewise stride 0x60**:

| CAN group | Content | Control pack offset | Register |
|---|---|---|---|
| 0x20–0x23 | Overview (volt, curr, soc, cycle, limits, protect, version) | +0x00…+0x1e | 34x00–34x10 |
| 0x30–0x33 | **16 cell voltages** | **+0x20…+0x3F** | 34x18–34x33 ✓ |
| 0x40 | **4 NTC temperatures** | **+0x40…+0x47** | 34x13–34x16 ✓ |
| 0x41 | **Ave / MOS / ENV NTC** | **+0x4A, +0x4C, +0x4E** | **34x11, 34x12, 34x17** |

The two lines marked ✓ match exactly with the descriptor table (16 and 4
elements respectively at exactly these offsets) — a strong, independent confirmation. It follows that
**34x11/34x12/34x17 are temperature values**; the older `BLE/VNSD_BLE_Modbus_CrossReference.md` likewise
lists 34011–34014 as `temperature1-4` and 34015 as `mosfetTemperature`.

From the BMS debug strings (`Cyc Cnt`, `Chg Mos/Dsg Mos`, `Max NTC/Min NTC/Ave NTC`,
`MOS NTC/ENV NTC`, `Max Cell/Min Cell`, `Protect1/Protect2`), the overview fields result in:
**34x03 = cycle_count** (independently confirmed by the BLE cross reference,
BLE offset 0x18 → 34003), 34x04 = MOSFET status, 34x07/34x08 = Max/Min NTC, 34x09 = Protect1.
These four remain marked as a **hypothesis**, since the field order within the frame has not been
verified byte-exactly.

### 7d. Micro v116 confirms the inverter side

The Micro FW contains the counterparts to the Control fields:
`err1 = %x err2 = %x, war1 = %x` (→ error_code1/2, warn_code), `max_power/min_power`,
`work_mode = %d, bat_mode = %d, backup = %d`, `grid_standand`, as well as the BMS view
`charge_u / charge_i / discharge_i / soc / cap / bat_vol / bat_cur / max_temp /
charge_req / force_charge_req / sleep_flag`. The latter matches the aggregate struct field for field
— with the note that `bat_temp` is called **`max_temp`** there, so it is
presumably the highest cell temperature.

### 7e. The missing string blob is in no archive image

All 14 Control images in the FW archive (VNSD-0, VNSA-0, VNSE3-0, HMG-50) contain the same
three `pgn_180x` debug strings and **no** cloud string blob. The region
0x0805F000–0x08063000 is thus not distributed via OTA. For the exact cloud field names of
the pack registers, only an **SWD/OpenOCD dump** of this flash region from the running device remains.

---

## 8. Version debug print — all version registers finally clarified

At `0x08036F40` there is a version output (not recognized by Ghidra as a function). It resolves
the entire 302xx block and corrects two registers:

```
Build:          Aug  5 2026 11:51:21
ems_type=       VNSD-0
boot_ver=       [0x20000038]                    -> Reg 30201
ems_ver=        150 (Konstante 0x96)            -> Reg 30200
=============================
mppt_ver(new)=  [0x20000188] | [0x20000189]<<8  -> Reg 30205   (u16)
mppt_ver(old)=  [0x20000188]                                    (u8)
bms_ver=        [0x20014FB0+0x22] = 0x20014FD2  -> Reg 30204
vns_ver=        [0x2000015E]                    -> Reg 30202
```

| Register | Source | Meaning | Status |
|---|---|---|---|
| 30200 | Flash 0x08010000 | **ems_ver** — Control app | confirmed |
| 30201 | 0x20000038 | **ems_boot_ver** — Control bootloader | **corrected** (was ems_sub_version) |
| 30202 | 0x2000015E | **vns_ver** — micro-inverter software | confirmed |
| 30203 | 0x20000160 | **vns_boot_ver** — micro-inverter bootloader | confirmed |
| 30204 | 0x20014FD2 | **bms_ver** — BMS (Pack0 struct +0x1e) | confirmed |
| 30205 | 0x20000188 | **mppt_ver** — MPPT firmware | **corrected** (was bms_sub_version) |

The firmware thus knows **four** independent modules: EMS (Control), VNS (micro-inverter),
MPPT and BMS. `hard_ver` (0x2000015C) and `dev_state` (0x20000162) are not served via Modbus.
Interesting is the double output `mppt_ver(new)`/`mppt_ver(old)`: the MPPT version
was extended from u8 to u16, and the register delivers the new 16-bit form.

---

## 9. Error and warning codes (36000 / 36100 / 36102)

### Origin clearly clarified

The three codes originate in the **micro-inverter FW v116** and are only
forwarded by Control:

| Micro FW SRAM | Micro name | Control field | Modbus |
|---|---|---|---|
| 0x200019F4 | `err1` | error_code1 | **36100 / 37013** |
| 0x200019F8 | `err2` | error_code2 | **36102 / 37015** |
| 0x200019FC | `war1` | warn_code | **36000** |

Evidence: `build_telemetry_block` (Micro) reads exactly these three words into the telemetry block
that Control takes over into its struct 0x20014E9C. Confirmed by the debug strings
`err1 = %x err2 = %x, war1 = %x` (Micro, `ADC_Sensor_Debug_Print`) and `err_code=%x` /
`warn_code=%x` (Control, `Debug_PrintErrorCodes`).

In addition, Control maintains two ring buffers with 20 entries each
(`Debug_PrintErrorAndEventLog`): an **error log**, 14 bytes/entry (code u16 + 4 bytes + timestamp)
and an **event log**, 9 bytes/entry. These are not served via Modbus.

### Plaintext: not present in the firmware

A full-text search across Micro v116 and BMS v118 for error terms (`ovp|uvp|ocp|otp|scp|
over|under|fault|protect|alarm|warn`) finds **no bit-to-text table** — only generic
messages (`Error: %s, %d`, `error lock on/set/reset`). The codes are pure bitmasks; the
mapping only exists in Marstek-internal documentation or the app.

**Reconstruction via the set sites is feasible:** every site that sets a bit in
`err1/err2/war1` is under a trigger condition (threshold comparison) from which
the meaning can be derived. `Inverter_Grid_Control`, for instance, demonstrably writes into the
upper half of `err1` (0x200019F6). This is a **separate work package** (the Micro FW contains
~1000 `orr`/`bic` instructions across 168 functions; of these, only the protection logic is relevant)
and cannot be done in one pass together with the register analysis.

The same applies on the BMS side to `Protect1`/`Protect2` (pack register 34x09) — the BMS FW outputs
them only numerically (`Protect1:%d  Protect2:%d`).

---

## 10. Ghidra correction

`0x08032F20` was named **`Register_PackDescriptor`** — however, the function does not register
anything, but builds a 29-bit CAN arbitration ID from four components (function code,
source, destination, counter). It has 41 callers. Renamed to **`CAN_Build_Arbitration_ID`**,
analogous to the identically named function in the BMS FW, and given an explanatory comment.

---

## 11. Addendum 2026-08-16 — re-analysis of open points 1–4

Another Ghidra pass (ReVa/PyGhidra on v150, 1630 functions) with the goal of further
resolving the remaining points listed in section 6 by generating code from the decompilation.
**Methodological finding upfront:** this image does **not** load SRAM addresses via
`movw/movt` (0 `movt` instructions in the entire image), but exclusively via
**literal pools** (`ldr rX,[pc,#…]`). The writer/reader scan was converted accordingly to
pool-word resolution (pool word = SRAM target address, Ghidra reference present).

### Point 1 — pack field names: blocker reproduced, offsets further hardened

- **Missing flash tail confirmed.** A full scan of the flash words yields **290 pointers
  pointing past the end of the image (0x0805EFFF)**, concentrated at **0x08061000 (165),
  0x08060000 (71), 0x0805F000 (47)**. Exactly there lie the cloud `snprintf` format strings
  with the pack field names. The finding from section 1 is thus independently reproduced:
  the exact names are **physically not in the `.bin`**. More decompilation cannot reconstruct
  strings that are not in the image → **hard data-availability blocker**,
  resolvable only via an SWD/OpenOCD dump of the region.
- **Offsets additionally hardened.** `Schedule_MinMax_PerGroup_Calc` (0x08013314) iterates,
  confirmed, 6 packs with stride 0x60 and reads the 16 cells starting at Pack+0x20 (= 34x18) ✓.
  `BLE_Build_BMS_Data_Response` (0x0800b4f4) independently reads **Pack+0x4A and +0x4C** →
  confirms **34x11 = Ave-NTC** and **34x12 = MOS-NTC** (second source alongside BMS CAN 0x41).
  This upgrades 34x11/34x12 from "hypothesis" to **confirmed**; 34x03/04/07/08/09
  remain evidenced in content by the BMS FW, just without cloud plaintext names.

### Point 2 — 30031–30035 (inverter/MPPT struct tail): **writer identified** ✓

No longer an "unresolved" tail. The write chain is complete:

```
CAN_FrameDispatcher (protocol type field bits 20-23 == 2)
  -> Modbus_ResponseDispatch (0x0802f630), branches by CAN function code (low byte):
       fc 0x03 -> Modbus_StoreRegisterSlot  -> base 0x20014F4C, slots 1..7 (+0x00..+0x30)
       fc 0xFF -> Modbus_StorePairSlot      -> base 0x20014F84 (= +0x38), 8-byte pair
```

| Register | SRAM | Writer | Meaning |
|---|---|---|---|
| 30031–30034 | 0x20014F84–0x20014F8A | `Modbus_StorePairSlot`, fc 0xFF slot 1 | Extended telemetry channel of the inverter/PV CAN node (same source as 30020–30040) |
| 30035 | 0x20014F8C (i16) | fc 0xFF slot 2 | ditto |

`Modbus_StoreRegisterSlot` (fc 0x03) limits the slot index to `< 8` (slots 1–7), so it
**never** writes the +0x38 tail — hence fc 0xFF (`Modbus_StorePairSlot`) fills these four
registers. `MPPT_Debug_Print` does not output slot 8 → **no plaintext name in the image**, but
origin and structure are now unambiguous. Live = 0 (no PV/inverter active), consistent.
A durable Ghidra comment was set on `Modbus_StorePairSlot`.

### Point 3 — 30212 / 0x20000138: **no runtime writer — static read-only byte** ✓

- A **full store scan across the entire image (8341 stores)** finds **not a single
  write access** to 0x20000138 — neither via an immediate nor via a covering word/
  doubleword store.
- An initially reported `strd` hit in `MQTT_JSON_RPC_Dispatcher` (0x0801c070) was a
  **false positive**: the `strd r0,r1,[sp,#0]` is a **stack store**; due to the
  strd operand order, the resolver incorrectly read r1 (=0x20000136) as the base.
  The actual write accesses there are `strb r0,[0x20000136]` with values **0/1/3** depending
  on the **power class** (600/800/1200/1500/2200/2500 W) — i.e. the class config byte
  @0x20000136, **not** 0x20000138.
- **Conclusion:** 0x20000138 (reg 30212, u8, live constant **5**) is not written by any
  business logic. It is a **statically initialized, read-only-via-descriptor-exposed
  config/variant byte** (initialized from the `.data` image on reset). The earlier
  "no resolved writers/readers" is thereby explained rather than open. A durable
  Ghidra comment was set on 0x0801c054/0x0801c070.
- **Correction (measured 2026-10-03):** the byte is the boot self-test result written by `Board_PeripheralSelfTest` @ 0x0801dcf4 (see `VNSD_Register_Reference.md`, 30212). Two real Venus D units (BMS 118, Micro 116, Control 151) read 5 although LAN works; CH395 version register 45605 returned 0x4A on one unit while the self-test expects 0x4B. Stage 5 with a working LAN is harmless; real faults are 2 (EEPROM) and 3 (flash).

### Point 4 — 32202 (elem_size 4 with i16): **no type conflict, 4-byte window** ✓

Descriptor entry: `32202, i16, elem_size 4, count 1, src 0x20014EB4`. A comparison of all
`elem_size==4` entries shows: **all others** are real 32-bit types (u32/float32,
e.g. 33000, 36000, 32102). **32202 is the only i16 with elem_size 4.** Consequence in
the serializer: **4 bytes starting at 0x20014EB4** are output = two adjacent i16 fields:

| Register | SRAM | Field |
|---|---|---|
| 32202 | 0x20014EB4 | `grid_sample_power` (i16, 1 W) — identical to 30006/37004 |
| 32203 | 0x20014EB6 | `off_grid_power` (i16, 1 W) — the "phantom" register from elem_size 4 |

This also means **32203 is named** (Live=0, off-grid inactive → plausible). It is not a
data-type contradiction, but a descriptor peculiarity (elem_size set as for a 32-bit field,
type byte i16) that exposes two i16 values as a register pair. For the HA integration:
32202 = grid_sample_power (i16), 32203 = off_grid_power (i16).

### Status after the addendum

| Point | Before | Now |
|---|---|---|
| 1 Pack field names | open (flash tail missing) | **blocker reproduced & confirmed**; 34x11/34x12 upgraded to confirmed |
| 2 30031–30035 | unresolved | **resolved** — writer/source identified (CAN fc 0xFF pair slot) |
| 3 30212 / 0x20000138 | no writers/readers | **explained** — static read-only config byte, no runtime writer (false positive disproved) |
| 4 32202 | descriptor inconsistency | **explained** — 4-byte window; 32202=grid_sample_power, 32203=off_grid_power |

*Generated via ReVa/PyGhidra scripts (literal-pool resolution, full store scan, effective-address resolver). Ghidra comments set durably on Modbus_StorePairSlot and MQTT_JSON_RPC_Dispatcher.*

---

## 12. Addendum 2026-08-16 (Part 2) — Pack field names found in the FW after all

**Correction of the core finding from section 1/6.** The assumption "pack field names only in the missing
flash tail" was too pessimistic (analogous to the Modbus table, which also at first seemed "not in the bin"
and then turned out to be there after all). After the FW archive was unlocked and a fresh full check across all three
firmwares, it is now established: **the field names exist as plain text in the firmware.**

### 12.1 Names are present as strings in the image (verified directly)

- **Control v150** contains, among other things, the complete per-pack debug string
  `cd=%d,b_ver,b_chv,b_rci,b_rdi,b_soc,b_soh,b_cap,b_vol,b_cur,b_tem,b_chf,b_slf,b_cpc,b_err,b_war,
  b_ret,b_ent,b_mot,b_tp1..b_tp4,b_vo1..b_vo16,self_check,mos` as well as the cloud JSON key string
  (`{"d":"di=%s&sn=%s&...&t1..t3&vc&tc&cy&..."}`) and PV/BMS strings. Only the *printing* cloud
  function is behind the image end — the **names themselves are present**.
- **BMS v118** contains the relevant field names as debug strings (`Bat Volt`, `Bat Curr`,
  `Bat Soc`, `Cyc Cnt`, `Chg Mos/Dsg Mos`, `Max/Min/Ave NTC`, `MOS NTC/ENV NTC`, `Max/Min Cell`,
  `Cell NTC`, `Cell Volt`, `Protect1/Protect2`, `Version`, `Soh`, `Max/Min Temp`).

### 12.2 Cause of the missing refs: non-disassembled code

These strings had `referenceCount 0` in Ghidra because the associated functions were **not
disassembled**. Coverage analysis BMS: ~29 KB instructions, but **~21 KB undefined
bytes**, of which large blocks with clear Thumb prologues (`push {..,lr}`) = code. So there
were indeed still areas not yet decompiled.

**Caught up (disassembly run over prologue candidates in the gaps):**

| Firmware | Functions before | after | new |
|---|---|---|---|
| BMS v118 | 550 | 649 | **+99** |
| Control v150 | 1630 | 1884 | **+254** |
| Micro v116 | 571 | 584 | **+13** |

### 12.3 Authoritative pack field list (from the newly decompiled BMS function)

`BMS_Debug_PerPack_Detail_Print` @0x08007098 (formerly FUN_/undisassembled) iterates over all packs
(BMS-internal struct, base 0x200041b6, **stride 0x60**) and prints each field with name, scale and
offset. This means the field list is **firmware-evidenced** (no longer guessed):

| Field | Scale/type | Field | Scale/type |
|---|---|---|---|
| Bat Volt | u16, /100 → 10 mV | Cell NTC[5] | i16 (5 cell temps) |
| Bat Curr | i16, /10 → 0.1 A | Cell Volt[16] | u16, mV |
| Bat Soc | u16, /10 → 0.1 % | Max/Min Cell | u16, mV |
| Cyc Cnt | u16 | Max/Min NTC | i16 |
| Chg/Dsg Mos | u8 bitfield | Ave NTC / MOS NTC / ENV NTC | i16 |
| Protect1/Protect2 | u16 (bitmask) | Version | u16 |

The BMS debug string `cd=%d, BMS(%d): num,vol,cur,soc,c_vol,c_cur,d_cur,mos,ver,max_v,min_v,
max_t,min_t,b_err1,b_err2,b_war1,b_vol[16],temp[5],env,mos` additionally provides the
overview field order of the per-pack CAN frames.

### 12.4 Effect on the register map

The Control pack registers (34xNN, pack struct 0x20014FB4, stride 0x60) are thus
**named with FW evidence** instead of hypothetically. In `VNSD_Register_Map_Final_generated.csv`,
**91 pack registers** (all 7 packs) were given the BMS field evidence and the confidence
upgraded (among others 34x00 Bat Volt, 34x01 Bat Curr, 34x05/06 Max/Min Cell, 34x08/09 Protect1/2,
34x17 ENV NTC → **high**). Already confirmed previously: 34x02 SOC, 34x03 Cyc Cnt, 34x04 Chg/Dsg Mos,
34x10 Version, 34x11/12 Ave/MOS NTC, 34x13–16 cell NTC, 34x18–33 16 cell voltages.

### 12.5 What genuinely remains open

- **Exact byte order of some overview fields** (e.g. 34x07 @+0x18, live 0..7 = flag, not a
  temperature) — resolvable via the Control per-pack CAN store function not yet extracted.
- **Cloud JSON short keys** (di/sn/to/…): the format string is in the image, the assembling
  cloud function is in the truncated region 0x0805F000+. Irrelevant for the Modbus registers,
  however, since the BMS names are authoritative.

**Conclusion:** The thesis "names cannot be reconstructed" is refuted. The pack field names are present
in the firmware, were uncovered by disassembling hidden code, and have been carried over into the
register map.

---

## 13. Addendum 2026-08-16 (Part 3) — Pack overview byte-exact + corrections

Your objection ("built at runtime does not mean unfindable") was spot on. The complete
per-pack store path has now been reconstructed from the flash — the handlers were simply not
disassembled.

### 13.1 Store path

```
CAN_FrameDispatcher (0x0802f274) -> else -> Protocol_AA_CommandDispatch (0x0802fb88)
   -> dispatch via SRAM table 0x2000018c (matches CAN group id>>0x10)
   -> 8 store handlers at 0x0802f764..0x0802f8b8 (formerly undisassembled, now named):
```

| Handler | CAN group | Pack offset | Content |
|---|---|---|---|
| CAN_Store_PerPack_Grp20_Overview0 | 0x20 | +0x00 | 34x00 BatVolt, 34x01 BatCurr, 34x02 Soc, 34x03 CycCnt |
| CAN_Store_PerPack_Grp21_Overview1 | 0x21 | +0x08 | +0x08/0c misc, **34x04(+0x0e)=Chg/Dsg mos flag** |
| CAN_Store_PerPack_Grp22_Overview2 | 0x22 | +0x10 | 34x05 MaxCell, 34x06 MinCell, +0x14 MaxNTC, +0x16 MinNTC |
| CAN_Store_PerPack_Grp23_Overview3 | 0x23 | +0x18 | **34x07 Protect1, 34x08 Protect2, 34x09 rsv(+0x5a), 34x10 Version** |
| CAN_Store_PerPack_Grp30_33_Cells | 0x30-33 | +0x20 | 34x18-34x33 = 16 cell voltages |
| CAN_Store_PerPack_Grp40_CellNTC | 0x40 | +0x40 | 34x13-34x16 = 4 cell NTC |
| CAN_Store_PerPack_Grp41_NTC | 0x41 | +0x48 | +0x48 AveNTC (no reg), **34x11(+0x4a)=ENV NTC, 34x12(+0x4c)=MOS NTC, 34x17(+0x4e)=0** |
| CAN_Store_PerPack_GrpExt_0x50 | – | +0x50 | Additional frame (not in CAN_TX_PerPack_10Msgs) |

The byte semantics of each frame are verified from the BMS sender `CAN_TX_PerPack_10Msgs` (0x08005b58,
BMS v118) — it reads the BMS struct fields and packs them 1:1 into the 8-byte frames.

### 13.2 Corrections to the register map (42 lines, all 7 packs)

| Register | before | **now (FW-evidenced)** |
|---|---|---|
| **34x07** (+0x18) | status_flags (hypothesis) | **Protect1** (protection bitmask 1; live 0..7 fits) |
| **34x08** (+0x1a) | protect1 | **Protect2** (protection bitmask 2) |
| **34x09** (+0x1c) | protect2 | **BMS field @struct+0x5a** (unnamed in the debug print, live ~0) |
| **34x11** (+0x4a) | temp_1 / "Ave NTC" | **ENV NTC** (ambient temperature) |
| **34x12** (+0x4c) | temp_2 | **MOS NTC** (MOSFET temperature) |
| **34x17** (+0x4e) | temp_3 (low, "live mostly 0") | **always 0 by construction** (frame 0x41 byte6-7 = 0) |

Confirmed (unchanged): 34x00 BatVolt, 34x01 BatCurr, 34x02 Soc(0.1%), 34x03 CycCnt,
34x04 Chg/Dsg mos, 34x05 MaxCell, 34x06 MinCell, 34x10 Version, 34x13-16 cell NTC,
34x18-33 16 cell voltages.

### 13.3 Result

**The pack overview is thus resolved gaplessly and byte-exactly — the last open question
(34x07) is answered.** There are no more "unreconstructable" pack registers; the
only fields not exposed as Modbus registers (+0x08/0c misc, +0x14/16 max/min NTC,
+0x48 ave NTC) are identified but simply not output by the descriptor.
