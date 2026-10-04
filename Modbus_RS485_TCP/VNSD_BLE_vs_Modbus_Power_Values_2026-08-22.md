# BLE Power Values vs. Modbus — Load Test and Firmware Evidence

> 🇩🇪 Deutsche Version: [VNSD_BLE_vs_Modbus_Leistungswerte_2026-08-22_de.md](VNSD_BLE_vs_Modbus_Leistungswerte_2026-08-22_de.md)

**Date:** 22 August 2026
**Trigger:** The VenusControl app shows power values that could not be found in the Modbus
register set. In addition, the question was open whether `bat_sample_power` (30001) is scaled
correctly — at idle it differed from the app by a factor of 2.
**Input:** Two full register scans (`--tiers ok,verm,unb`, 540 of 582 registers each) at idle
and under 600 W discharge load, plus two app screenshots of the same operating states.
Run in parallel with the HA integration, without disturbance.
**Result:** All app values are explained. Two of them are computed by the **firmware** in the
BLE payload, not by the app. `bat_sample_power` is correct.

---

## 1. Measurement Points

| | Idle (bypass) | 600 W discharge |
|---|---|---|
| App time | 00:37 | 01:01 |
| Scan | 00:53–00:55 | 01:01–01:03 |
| `30001` bat_sample_power | −34 | **−648** |
| `30006` grid_sample_power | 0 | **601** |
| `30007` / `32302` off_grid_power | 48 | 50 |
| `35100` inv_state | 6 (bypass) | 3 (discharge) |

App display of the same states:

| App field | Idle | 600 W |
|---|---|---|
| AC Power | 0 W | 601 W |
| Backup Load | 48 W | 49 W |
| Battery Power | −16 W | −651 W |
| Grid Power | −48 W | 552 W |

## 2. Mapping App → Modbus

The BLE payload is assembled in `FUN_0800b024` (`0x0800b024`, 1104 bytes).
Buffer: `0x20018857`. Sources: `0x20014E9C` (inverter telemetry), `0x20014F4C`
(MPPT array), `0x20014F8E` (BMS CAN aggregate), `0x20014D98` (energy counter).

| BLE offset | App field | Firmware | Modbus |
|---|---|---|---|
| `0x00` | Backup Load | telemetry `+0x1A` `off_grid_power` | **30007 / 32302** |
| `0x02` | AC Power | telemetry `+0x18` `grid_sample_power` | **30006 / 32202 / 37004** |
| `0x8C` | Battery Power | computed, see §4 | **no register** |
| `0x90` | Grid Power | computed, see §3 | **no register** |

The first two are direct copies and match the measured values exactly
(601 = 601, 49/50 = 50). **HA's `AC-Leistung` corresponds to the app row "AC Power", not to
"Grid Power".** This mix-up was the trigger for the whole investigation.

## 3. `Grid Power` (BLE `0x90`) — Formula Decoded

From `FUN_0800b024`, lines 20–31 and 180:

```c
if (telemetrie[+0x18] /* grid_sample_power */ == 0) {
    if (telemetrie[+0x00] /* inv_state */ == 4)   // 4 = Backup Mode
        s10 = telemetrie[+0x1A];                  // off_grid_power
    else
        s10 = 0;
} else {
    s10 = telemetrie[+0x18];                      // grid_sample_power
}
s1 = telemetrie[+0x1A];                           // off_grid_power

payload[0x90] = s10 - s1;      // Grid Power
payload[0x92] = s10;
```

Cross-check at both measurement points:

| | s10 | s1 | s10 − s1 | App |
|---|---|---|---|---|
| 600 W (`grid_sample_power` = 601) | 601 | 49 | **552** | 552 |
| Idle (`= 0`, inv_state 6 ≠ 4) | 0 | 48 | **−48** | −48 |

Both exact. `Grid Power` is therefore **not a separate measured value**, but a
firmware calculation from two registers that Modbus already provides. The special case
`inv_state == 4` handles backup mode.

**Practical consequence:** A zero-feed-in controller does not need a missing register for this.
`30006 − 30007` reproduces the value, with the case distinction from §3 for
backup mode.

## 4. `Battery Power` (BLE `0x8C`) — Is a Product, Not a Measured Value

From `FUN_0800b024`, lines 168–178:

```c
u11 = uint(bms[+0x00]) * 0.01;        // BMS voltage    -> Modbus 32100
u12 = int16(bms[+0x02]) * 0.1;        // BMS current    -> Modbus 32101
payload[0x8C] = int(u12 * u11);       // Battery Power
```

`bms` = `0x20014F8E`, the CAN aggregate from PGN 1801.

This explains the idle deviation: the app shows **voltage × current from the BMS
aggregate**, whereas Modbus `30001` shows the **inverter's own power measurement**
(`bat_sample_power`, telemetry `+0x1C`). Two independent derivations. Under load they agree to 0.5 % (−651 vs. −648); at idle
the difference in which current/voltage pairing is used dominates (−16 vs. −34).

**`bat_sample_power` (30001) is scaled correctly.** A factor of 2 could not have hidden at −648 W.

### Open Question on 32101

The back-calculation for the 600 W point gives: −651 W / 53.37 V = −12.2 A, so
`bms[+0x02]` must have contained **−122** at that time (i16 × 0.1 A). Exactly this
value was read simultaneously by `34301` (pack 4 current).

The Modbus scan of `32101`, however, returned 39309 (i16 −26227). Either the
`source_ptr` of 32101 is misassigned, or there is a conversion between the SRAM field and the Modbus output
that is not yet understood. The descriptor's `fw_scale` `/10` does not explain it — dividing 39309 by 10 leaves no plausible current.

This is the most promising open point: the firmware path in `FUN_0800b024` clearly specifies
the unit (0.1 A) and the sign handling (i16).

## 5. The Venus D Only Ever Discharges One Pack

`bms_active_pack_index` (32111) was **4** in both scans. Comparing both
runs, exactly one pack moved:

```
Pack   Current (34x01)    Voltage (34x00)      Cell voltages
 1        0 ->    0        5338 -> 5338         ±1
 2        0 ->    0        5328 -> 5328         ±1
 3        0 ->    0        5332 -> 5332         ±1
 4       -3 -> -122        5250 -> 5195         -34 mV
 5        0 ->    0        5334 -> 5333         ±1
 6        0 ->    0        5334 -> 5334         ±1
```

−122 at display factor 0.1 A = −12.2 A, matching −648 W at 51.95 V.

**Consequence for interpreting registers:** A `0` in a pack current register means
"pack inactive", **not** "register unused". An earlier note on `30101` / `34001`
claimed the opposite and has been corrected.

## 6. Further Observations from the Double Scan

113 registers changed between idle and load. Notable ones:

| Register | Idle → load | Interpretation |
|---|---|---|
| `30028` inv_bat_voltage | 526 → 519 | 52.6 → 51.9 V — voltage sag under 12 A |
| `30000`/`34000` bat_volt | 523 → 516 and 5338 → 5338 respectively | the inverter measurement sags, the BMS terminal measurement does not — the difference is contact resistance |
| `32203` grid_sample_power_w2 | 0 → 601 | confirms 32202/32203 as the sign-extended 32-bit version |
| `32101` bms_battery_current | 39321 → 39309 | Δ −12 at Δ −11.6 A ⇒ roughly 1 A per count step |
| `42021` | 0 → 600 | discharge setpoint, readback value of the write register |
| `32102`/`32103` bat_sample_power | 0 → 0 | **stays dead**, even though it uses the same pointer as 30001 — descriptor contradiction as with 32202 |

## 7. Methodology Note

Both scans ran in parallel with the running HA integration via the proxy
`192.168.1.60:1502`. The integration reported 100 % successful reads and
zero timeouts throughout. The proxy therefore multiplexes cleanly — a scan during normal operation is
not a risk.

The double scan under two defined load states clarified more than any
single measurement: slopes, active packs, and dead registers only become visible in the delta.
