# Pack 1 BMS Fault: Backup Socket Cannot Be Switched On

> 🇩🇪 Deutsche Version: [VNSD_Pack1_BMS_Fehler_Analyse_2026-08-21_de.md](VNSD_Pack1_BMS_Fehler_Analyse_2026-08-21_de.md)

**Date:** 21 August 2026
**Occasion:** The Venus D refuses to switch on the backup (EPS) power socket. A
version scan simultaneously showed a BMS version mismatch between Pack 1 and the
other packs.
**Input:** Three live scans via Modbus TCP (proxy `192.168.1.60:1502` → device
`192.168.1.50:502`), final status `Scan_Logs/pack1_bms_fault_2026-08-21.csv` (local, raw data)
(540 of 582 registers, 12:07:16–12:08:48).
**Result:** Pack 1 is a **newly installed pack in as-delivered condition** — release FW
117, zero cycles, ~50 % storage SOC. The other packs run 118. Because of this
version skew, Pack 1 reports a **communication fault**, and the device then locks
the off-grid branch. The hardware is fine.

---

## 1. The Finding in One Line

`error_code1 = 0x00200000` → **bit 21 = BMS fault**. It is the only fault set
in the entire device.

| Register | Value | Meaning |
|---|---|---|
| `36100` / `37013` | `0x0020` | error_code1 high word → **bit 21 set** |
| `36101` / `37014` | `0x0000` | Low word — bit 7 (backup/off-grid fault) **not** set |
| `36102` / `36103` | `0x0000` | error_code2 clean |
| `36000` / `36001` | `0x0000` | warn_code clean |
| `37023` / `37024` | `0x0000` | MPPT fault/warning clean |
| `41200` | `0` | `backup_ups_enable` — backup disabled |
| `30005` / `30007` | `5` / `0` | Off-grid 0.5 V / 0 W — socket dead |

Word order according to `VNS_Micro_Inverter/VNSD_Error_Codes_Micro_FW_Analysis.md` §9.1:
`error_code1 = (36100 << 16) | 36101`. `0x0020` in the high word = bit 16+5 = **bit 21**.

**Bit 20 (CAN communication fault) is not set** — the CAN link
inverter ↔ BMS is up. The fault is internal to the BMS.

---

## 2. Pack Comparison (scan 12:07:16, raw values)

| Offset | Field | **Pack 1** | Pack 2 | Pack 3 | Pack 4 | Pack 5 | Pack 6 | Pack 7 |
|---|---|---|---|---|---|---|---|---|
| +00 | `bat_volt` | **5259** | 5329 | 5335 | 5281 | 5339 | 5338 | 0 |
| +01 | `bat_curr` | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| +02 | `soc` | **479** | 705 | 799 | 696 | 698 | 697 | 0 |
| +03 | `cycle_count` | **0** | 13 | 19 | 19 | 62 | 56 | 0 |
| +04 | `mos_status` | **3** | 0 | 0 | 0 | 0 | 0 | 0 |
| +05 | `max_cell_voltage` | **3289** | 3332 | 3336 | 3302 | 3339 | 3338 | 0 |
| +06 | `min_cell_voltage` | **3286** | 3330 | 3334 | 3300 | 3336 | 3336 | 0 |
| +07 | `protect1` | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| +08 | `protect2` | **16384** | 0 | 0 | 0 | 0 | 0 | 0 |
| +09 | `bms_reserved_5a` | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| +10 | `bms_version` | **117** | 118 | 118 | 118 | 118 | 118 | 0 |
| +11 | `temp_1` | 300 | 294 | 299 | 305 | 302 | 289 | 0 |
| +12 | `temp_2` | 232 | 238 | 238 | 244 | 236 | 227 | 0 |
| +13…+16 | `ntc_block` | 202–211 | 235–237 | 231–233 | 231–236 | 223–226 | 220–222 | 0 |

Pack 7 is 0 throughout — six packs are installed, that is the normal state.

Register base: `34x00`–`34x16`, names according to
`Modbus_RS485_TCP/VNSD_Register_Map_Final_generated.csv` (status of the 2026-08-16 correction,
frame 0x23: byte 0-1 = Protect1, byte 2-3 = Protect2, byte 4-5 = struct field @+0x5a).

---

## 3. Pack 1 Is Healthy Hardware-Wise

| Pack | Cells | min–max | Spread |
|---|---|---|---|
| **1** | 16 | 3286–3289 mV | **3 mV** |
| 2 | 16 | 3330–3332 mV | 2 mV |
| 3 | 16 | 3334–3336 mV | 2 mV |
| 4 | 16 | 3300–3302 mV | 2 mV |
| 5 | 16 | 3336–3339 mV | 3 mV |
| 6 | 16 | 3336–3338 mV | 2 mV |

All 16 cells of Pack 1 respond, the balance at 3 mV is as good as with the
healthy packs, the temperatures (30.0 °C / 23.2 °C, NTC 20.2–21.1 °C) are
unremarkable. **The BMS is communicating** — the fault is not a communication dropout
but a reported fault condition.

---

## 4. What Is Going On with Pack 1

**Pack 1 is newly installed and in as-delivered condition.** Three values that each look like a
fault on their own are together exactly the expected picture of a factory-fresh pack:

| Value | Observation | Explanation |
|---|---|---|
| `cycle_count = 0` | 13–62 on the other packs | **Never cycled** — the pack is new |
| `soc = 479` (47.9 %) | 69.6–79.9 % on the others | **Factory storage SOC** (~50 %, usual storage state of charge for lithium) |
| `bms_version = 117` | 118 on the others | **Factory release firmware**, not yet raised to 118 |

The lower pack voltage (52.59 V versus 53.29–53.39 V) and the lower
cell voltages (3286–3289 mV versus 3300–3339 mV) are simply the consequence of the lower
state of charge, not a separate anomaly.

The only real fault is **`protect2 = 0x4000`** = bit 14 = **communication fault**
(`VNSD_BMS_FW_Analysis_v117.7.md` §5.3, set condition `DAT_20002901 ≠ 0`) — triggered by the
**version skew 117 versus 118** in the pack group.

### 4.1 The Version Scheme

Addition to the previous documentation, which listed 116 / 1177 / 118 only as observed values:

| Raw value | Version | Type |
|---|---|---|
| `116` | v116 | Release |
| `117` | v117 | **Release** — delivery state of new packs |
| `118` | v118 | Release |
| `1177` | v117.7 | **Beta** |

> **Rule:** Integer values are **release** versions and are stored directly.
> Versions with a decimal place (`.x`) are **beta** builds and are stored as version × 10
> (`1177` = v117.7).

The same pattern applies to `ems_version` (`1492` = v149.2 beta, `150` = v150 release).

### 4.2 Why This Blocks Backup Operation

- **Version skew in the group:** Pack 1 speaks the protocol of FW 117, the other
  five that of 118. The aggregate BMS acknowledges this with the communication fault bit and
  does not release the group.
- **SOC skew:** At 52.59 V, Pack 1 is about **0.7–0.8 V below** the other packs.
  Connecting in parallel directly would drive a considerable equalizing current — a lock is
  technically correct here, regardless of the version problem.
- **`mos_status` inverted:** Pack 1 = 3 (charge and discharge MOSFET closed),
  packs 2–6 = 0 (both open).
- **`bat_curr = 0` on all packs** — no current flows anywhere, there is no
  enabled discharge path.

`error_code1` bit 7 (backup/off-grid fault) is consequently **not** set: the
off-grid branch does not run into an overload, it does not even start.

---

## 5. Two Corrections to the Existing Documentation

### 5.1 err1 Bit 21 Observed Live for the First Time

`VNS_Micro_Inverter/VNSD_Error_Codes_Micro_FW_Analysis.md` §9.3 states:

> "**err1 bit 21 (BMS fault)** was never observed: on battery disconnection the
> CAN drops first (bit 20), a substantive BMS fault was not present in any test."

Exactly this case is now present — **bit 21 set, bit 20 clear**. The statically
derived meaning is thereby confirmed live.

### 5.2 A BMS Fault Can Be Localized After All

The same file states in §9.5 that `bat_err1`/`bat_err2`/`bat_warn1` have no
descriptor entry, which is why a BMS fault is visible via Modbus "**only as the collective bit
err1 bit 21** — without any detail information".

That applies to the **aggregate** fault words, not to the overall picture: via the
**per-pack register `34x08` (`protect2`)** the fault can be assigned unambiguously to a
single pack and named via the documented Protect2 bitmask. The gap
in the Modbus interface is thus smaller than assumed.

---

## 6. Methodological Side Findings

- **Range scans over undocumented registers knock the Modbus server out.** A
  scan with `--regs 36000-36103` (101 of the 126 registers not in the map) led at 11:53
  to a connection drop; the proxy then reported `ConnectionRefused` on
  `192.168.1.50:502`. Consequence: **always use `--tiers ok,verm,unb` instead of raw
  range specifications** — this reads only mapped registers and the
  gaps are never touched.
- **42 registers without a response are not a fault.** They are exclusively write-only registers
  (the complete `45000` command block, `41000`, `41100`, `45603`, `45604`, `46000`,
  `46500`–`46503`). They have FC06/FC10 write handlers, but no read path.
- **The register map was unified on 21.08.** The former register map file
  (440 registers, outdated pack assignment at `34x07`–`34x09`) was removed; canonical
  is now `Modbus_RS485_TCP/VNSD_Register_Map_Final_generated.csv` (582 registers). The three scan scripts understand
  both column schemas, see `Methodik_und_Meta/VNSD_Analysis_Scripts.md`.

---

## 7. Open Points

| # | Point |
|---|---|
| 1 | Raise Pack 1 from factory FW 117 to 118 — trigger BMS OTA. Nothing can be forced via Modbus: `41200` does not lift the lock (it sits in the BMS), the version registers are read-only mirrors. |
| 2 | After the update, check whether `34008` returns to 0 and `36100` with it. |
| 3 | Open whether the communication fault depends solely on the version skew or additionally on the SOC skew. Will become clear as soon as Pack 1 is on 118: if the bit disappears despite the SOC still being low, it was purely the version. |
| 4 | Cross-check the `mos_status` inversion (Pack 1 = 3, rest = 0) in normal operation — unclear whether 0 on the other packs is the idle state or an active isolation. |
| 5 | Protect1 bits 12–15 and Protect2 bits 0–8, 10, 13 are still not decoded. |
| 6 | Carry the version scheme from §4.1 over into the register map notes for `30204` / `34x10` / `ems_version`. |
