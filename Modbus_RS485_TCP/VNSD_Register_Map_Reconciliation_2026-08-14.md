# Register map reconciliation: firmware analysis ↔ live scans

> 🇩🇪 Deutsche Version: [VNSD_Register_Map_Abgleich_2026-08-14_de.md](VNSD_Register_Map_Abgleich_2026-08-14_de.md)

**Date:** 14 August 2026
**Input:** 31 scan logs (536.570 data points), descriptor table v150, BMS v118, Micro v116
**Output:** `VNSD_Register_Map_Final_generated.csv` (505 rows)

---

## 1. What the new overall map contains

| Source | Rows | Content |
|---|---|---|
| **FC03 descriptor** | 410 | 246 descriptor entries, **expanded** to all actually occupied registers |
| **Write handler** | 95 | 41xxx/42xxx config/write registers |

The expansion is the key: the descriptor lists only **start registers**. Occupied registers =
`ceil(elem_size × count / 2)`. This resolves all supposedly "missing" registers —
e.g. 30304→30304-30309 (MAC), 31000→31000-31009 (device_name), 34018→34018-34033
(16 cell voltages), 33000→33000-33001 (u32). **495 of the 505 registers have live values.**

## 2. Two scales — the most common mistake

The old map and the descriptor table seemingly contradicted each other in 162 cases. Almost all
of them are **not a contradiction**, but two different things:

- **`fw_scale`** — what the *firmware* computes (scale code from the descriptor)
- **`anzeige_faktor`** — what the *client* has to compute to obtain the physical unit

Example 30000: `fw_scale = x1` (the FW does not scale) **and** `anzeige_faktor = 0.1` (the
raw value is in 0.1 V). Both are correct. The new CSV therefore carries **both columns**.

Real deviations remain only where type or meaning differ (section 4).

## 3. Confirmed by live data

| Register | Interpretation | Discharging | Charging | Verdict |
|---|---|---|---|---|
| 30001 | bat_sample_power, i16, W | **−2676 W** | **+2310 W** | ✓ Sign correct |
| 30006 | grid_sample_power, i16, W | +2511 W (feed-in) | −2467 W (import) | ✓ |
| 32100 | bms_battery_voltage, **10 mV** | **53.10 V** | **54.49 V** | ✓ Charging > discharging |
| 34002 | pack1_soc, 0.1 % | 57.7 % | 70.1 % | ✓ |
| 32204 | grid_pf, 0.1 Hz | 50.0 Hz | 50.0 Hz | ✓ |
| 34x18–34x33 | Cell voltages, mV | 3177–3441 mV | | ✓ typical for LFP |
| 34x13–34x16 | NTC block, 0.1 °C | 11.6–36.1 °C | | ✓ 4 elements confirmed |
| 34x10 | bms_version | 116 / 118 / 1177 | | ✓ real FW versions |
| 34x03 | cycle_count | 0/1/6/7/10/12 | | ✓ small, monotonic |
| 34x04 | mos_status | 0..3 | | ✓ exactly 2 flags (chg/dsg MOS) |

## 4. Where the live data **refuted** my firmware hypothesis

From the BMS CAN frame layout I had derived: 34x07 = `Max NTC`, 34x08 = `Min NTC`.
The scans show something different:

| Register | Live value range | Conclusion |
|---|---|---|
| 34x07 | 0..7, almost always 0 | **not a temperature** → status/flag field |
| 34x08 | consistently 0 | **not a temperature** → protection bitmask (`protect1`) |
| 34x09 | almost always 0 | protection bitmask (`protect2`) |
| 34x11 / 34x12 | 23.5–45.2 °C / 21.4–56.9 °C | **real temperatures** ✓ |

A temperature would constantly be at 200–400 (0.1 °C). The older live analysis
(`VNSD_Register_Map_Final_generated.csv`) had already listed 34x08/34x09 as `protect1/protect2` —
**the live data are right, my frame derivation was wrong.** Corrected.

The temperature statement for 34x11/34x12 stands and is now evidenced from both sides
(BMS CAN group 0x41 **and** plausible measured values).

## 5. Confirmed defect: register 32101 (bms_battery_current)

| State | Raw value | Interpreted | Plausible? |
|---|---|---|---|
| **Charging** | 42 | 42 A | ✓ 2310 W / 54.49 V = **42.4 A** |
| **Discharging** | 39268…39270 | −26267 A | ✗ unusable |

The discharge value **varies by ±1**, so it stems from a real measured value — it is not a
constant bogus value, but a wrong scaling/sign handling in the writer of the
SRAM slot. When charging, the chain is correct.

**For the HA integration:** use 32101 only when charging; when discharging, fall back to 34x01
(`pack_bat_curr`) or the power 30001.

## 6. Pack numbering unified

The descriptor analysis had used pack0..pack6, the rest of the project (old map,
scan scripts, live CSV columns such as `p1_bat_soc`) counts **pack1..pack7**. Switched to the
project convention: **34000s = pack1**, 34100s = pack2, …, 34600s = pack7.

## 7. Data quality of the logs

24 of the 26 logs are internally consistent. Only `entladen_lang.csv` (local, raw data) and `laden_lang.csv` (local, raw data) contain
isolated outliers (e.g. `work_mode` 62859 instead of 0, `grid_pf` 49320 instead of 500) — in each case
single measured values during state changes, no systematic offset. The columns `min_all`/
`max_all` of the new CSV deliberately contain these outliers unfiltered; for the assessment, the
columns `v_entladen`/`v_laden`/`v_ref_v150` (snapshots from defined states) are
more meaningful.

## 8. Confidence status after the reconciliation

| Level | Count (descriptor entries) |
|---|---|
| high | 129 |
| medium | 86 |
| low | 24 |
| hypothesis | **7** (of originally 121 open) |
