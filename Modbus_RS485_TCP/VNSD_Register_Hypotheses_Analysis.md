# Marstek Venus D — Register Hypotheses

> 🇩🇪 Deutsche Version: [VNSD_Vermutungen_Register_Analyse_de.md](VNSD_Vermutungen_Register_Analyse_de.md)

Status: 2026-07-07
Basis: 4 single scans + 3 long-term scans + boot scan + battery disconnect tests + discharge steps test
FW: EMS:147, VNS:115, MPPT:104, BMS:116 (no update between the scans)
Versioning scheme: suffix-based (e.g. 149=release, 1492=beta of 149)

---

## Overview of scan conditions

| Scan | State | SOC | Duration |
|------|-------|-----|----------|
| scan1 | Idle, new FW | 14% | Single scan |
| scan2 | Standby, old FW | 95% | Single scan |
| scan3 | Manual discharge 500W | ~88-94% | Single scan |
| scan4 | Manual charge 2200W | ~50% | Single scan |
| langzeit1 | Charging | ~65→70% | 16 min |
| langzeit2 | Charging | ~70% | 10 min |
| langzeit3 | Charging | 65%→100% | 2h18min |
| boot_scan | 3x restart (power cycle) | ~95% | 18 scans |
| battery_disconnect | Battery disconnected + reconnected | ~95% | 18 scans, ~2 min |
| power_cycle_no_bat | Battery removed → 2x AC cycle → battery reconnected | ~99% | 41 scans, ~7 min |
| discharge_steps 1-3 | Discharge with stepwise power increase (schedule) | 90-99% | 69 scans, ~29 min |
| backup_load | Backup mode with various loads (52W to 3kW+) | 40-44% | 28 scans, ~10 min |
| backup_overload | Backup overload ~3.2kW (>2.5kW limit) | 30-38% | 35 scans, ~4 min |
| backup_overload_ac_disconnect | Backup overload + AC disconnect | ~30% | 14 scans, ~2 min |
| backup_no_discharge | Backup active, but 42010=0 (no forced discharge after reboot) | ~30% | 3 scans |
| discharge_under_dod | Long-term discharge to below DOD, AC disconnect, island operation, reserve cutoff | 30%→7% | 548 scans, ~3h17min |
| unter_dod_backup_nicht_aktivierbar | Below DOD: backup cannot be switched on, forced charging on AC reconnect | 7-8% | 21 scans (scan_powercycle), ~4 min |
| zwangsladung_unter_dod | Forced charging below DOD — detail scan with the full 419 registers | 8% | 7 scans, ~2 min |
| fw_upgrade_149_2 | FW update from 148 to 149.2 — device ~25 min offline | 60% | 22 scans (scan_powercycle), ~27 min |
| laden_6pack_149_2 | 6 packs, FW 149.2, charging with massive SOC difference, without fan | 50-60% | 5 scans (full), ~7 min |
| nach_bms_upgrade_117_7 | 6 packs, BMS update from v116 to v117.7, otherwise standby/idle | ~50% | 4 scans, ~5 min (15:43-15:48) |

---

## Hypotheses (🔍) with justification

### 30005 — `backup_voltage` (backup output voltage) ✅ backup test

- **Values:** ~1V without backup (inactive), 236-242V with backup (scale 0.1)
- **Justification:** In the backup test the value jumps from ~10 (1.0V = off) to ~2418 (241.8V) as soon as a backup load is present. Shows the typical voltage drop under load: 242V at low load (<200W), 237V at high load (>3kW). This is exactly the AC voltage behavior of a loaded inverter output.
- **Previously:** Mistakenly assumed to be `comm_queue_depth`. The small values (2-19) without backup were noise/standby residual voltage.
- **Source:** backup_load_activated.csv (local, raw data)
- **Confidence:** High — the voltage profile under load is unambiguous.

### 30006 — `ac_power` (AC inverter output power) ✅ confirmed + corrected

- **Values:** int16; negative = grid import (charging), positive = AC output (discharging)
- **Justification discharge steps:** In the discharge steps test, 30006 tracks the setpoint from 43103 exactly. Without backup = feed-in power.
- **CORRECTION backup test:** 30006 is NOT the grid feed-in, but the total AC output power of the inverter. In backup mode, 30006 stays constant at ~2208W, even though the actual grid feed-in decreases (observed: at 2kW backup only ~200W into the grid, at 3kW backup even 1kW grid import). The actual grid power results from: **Grid = 30006 - 30007**.
- **Without backup:** 30007=0, therefore Grid = 30006 (special case — that is why it was previously not distinguishable)
- **Source:** discharge_w_steps 1-3, backup_load_activated.csv
- **For the HA integration:** In backup mode do NOT use 30006 directly as the grid sensor! Instead: `grid_power = 30006 - 30007`
- **Confidence:** High — the backup test calculation matches the observed behavior exactly.

### 30007 — `backup_power` (backup output power) ✅ backup test

- **Values:** 0 without backup, 0-3271W with backup
- **Justification:** Was always 0 in ALL previous scans (idle, standby, charge, discharge, boot, disconnect). Only in the backup test did it show active values for the first time. Shows the power output at the backup output. Values correlate exactly with the connected load (52W to 3kW+).
- **If 30007 > 30006:** The inverter cannot supply the backup load alone, the difference is drawn from the house grid. At 30007=3271W and 30006=2208W: 3271-2208 = 1063W grid import (observed: ~1kW).
- **Source:** backup_load_activated.csv
- **Confidence:** High — the energy balance matches exactly.

### 30301 — `active_inverter_state` (inverter state?) 🔍 hypothesis

- **Values:** 1, 2, 3
- **Justification:** Was constant 1 in all previous scans (previously listed as bluetooth_status). In the backup test the value changes 1→2→3→2→3. The changes correlate loosely in time with pack rotations (P1→P2→P4), but it is not a 1:1 mapping to the pack index. Could be an internal inverter routing state.
- **Previously:** Assumed to be bluetooth_status (always 1 = "on"), but the changes in the backup test rule out Bluetooth.
- **Source:** backup_load_activated.csv
- **Confidence:** Low — function unclear, only the correlation with state changes is certain.

### 30101 — `pack1_current_mirror` (pack 1 current mirror)

- **Values:** 0 (pack 1 idle), -25 to -160 (pack 1 active, scale 0.1A)
- **Justification:** In the discharge steps test exactly identical to 34001 (pack1_current) when pack 1 is active. On a pack change (pack 1→pack 2), 30101 immediately goes to 0, even though the discharge continues. Not a system current, but a pack 1 mirror.
- **Previously:** Listed as battery_current, but was always 0 in all scans (because by chance pack 1 was never active or no discharge was running).
- **Source:** discharge_w_steps1.csv (local, raw data) (scan 5-32: pack 1 active, scan 33+: pack 2 active)
- **Confidence:** High — exact value match with 34001.

### 30029 — `battery_power_factor` (power index, NOT linear)

- **Values:** int16, -18 to +12
- **Charging (positive):**
  - 2035W → 11-12
  - 1260W → 6
  - 480W → 1
  - Idle → -1 to -2
- **Discharging (negative, discharge steps test):**
  - 500W → -5
  - 1000W → -9
  - 1500W → -12
  - 2000W → -16
  - 2200W → -18
- **Linearity checked: NO.** The ratio 30029/power is not constant. At low power the index per watt is larger, at high power smaller. Not a linear scaling factor, rather an internal power index or PWM duty cycle.
- **Confidence:** Medium — correlation clear, but not linear and exact meaning unclear.

### 30036 — `hardware_version` (hardware version/configuration)

- **Values:** Constant 219 across all scans and 2+ hours of long-term
- **Justification:** Absolutely stable = not a sensor, but configuration. 219 could be a HW revision, board variant or calibration parameter.
- **Confidence:** Medium — that it is not a sensor is certain. Whether HW version or another parameter, unclear.

### 30205 — `mppt_version` (MPPT firmware version) ✅ OTA-confirmed

- **Value:** 104 (constant)
- **Correction:** Was mistakenly listed as `bms_version_sub`.
- **Justification:** OTA screenshot shows 4 FW components: EMS:142→147, VNS:114→115, **MPPT:104**, BMS:116. Register 30205=104 is the MPPT version, not the BMS sub-version.
- **OTA also confirms:** 30200=EMS, 30202=VNS (not VMS!), 30204=BMS.
- **Confidence:** Confirmed by the OTA display.

### 30201 / 30203 — `ems_build` / `vns_build` (build/beta suffix?)

- **Values:** 18 / 106 (constant)
- **Justification:** Located directly after the version registers. The OTA display shows no sub-versions, but Marstek uses a suffix scheme: e.g. "149" = release, "1492" = beta of 149 with suffix 2. The values 18/106 could be such suffixes or internal build numbers.
- **Alternative:** Could be versions of further FW modules (bootloader etc.) — in the Ghidra analysis there were flash regions without assignment. The communication module (WiFi) is probably NOT it, however: its version is a date timestamp (e.g. "202409090159" = 2024-09-09 01:59), which does not fit into a uint16 and is probably queried directly from the WiFi chip, not via Modbus.
- **Confidence:** Low — connection with versioning plausible, exact format unclear.

> **Update 2026-08-14 — build theory refuted:** 30201 (=18) and 30203 (=106) remained constant across
> **all** scans, even though the associated version registers changed (30200 ems:
> 147→1492→150; 30202 vns: 115→116). A build/beta suffix of the main FW would change with every
> update — it does not. Thus 30201/30203 are **not build numbers**; in the
> register map they are now listed as `unknown_30201/_30203` (constant, meaning open).
> Also corrected: "1492" is **not** "beta of 149 with suffix 2", but the ×10 encoding
> of a decimal version (1492 = v149.2, analogous to 1177 = BMS v117.7); integer versions are stored raw
> (116, 118, 150). And the date build of the communication module (202409090159) IS readable via Modbus after all
> — as ASCII across 6 registers in **30350–30355** (not in 30201/30203).

### 30210 — `active_pack_charge_status` (mirror of the active pack)

- **Values:** 0 or 3
- **Justification:** In the long-term scan exactly identical to the 34x04 value of the currently active pack. When pack 1 charges: 30210=3 and 34004=3. When pack 2 charges: 30210=0 (because it mirrors the pack 1 value, and pack 1 is then idle). Alternatively: 30210 always mirrors pack 1's status.
- **Confidence:** High for the correlation, interpretation as "mirror" certain.

### 34x04 — `packX_charge_status` (pack charge status)

- **Registers:** 34004, 34104, 34204, 34304, **34404, 34504** (packs 1–6; packs 5/6 confirmed since the v150 scan)
- **Values:** 0=idle, 3=active (charging or discharging)
- **Justification:** In the long-term scan: 34x04 =3 exactly when the respective pack draws/delivers current (34x01 ≠ 0). =0 when the pack is idle. Previously labeled "max_cell_voltage_delta", which makes no sense for values 0-3.
- **Important (offset):** charge_status is at pack offset **+4** (34x04); the **max_cell_v** is at **+5** (34x05), min_cell_v at +6 — do not confuse (see the corrected pack layout in the connection reference §3.7).
- **Open question:** Whether there are further values (1, 2?) — so far only 0 and 3 observed. Maybe 1=standby, 2=balancing?
- **Confidence:** High — that it is a status is clear. Exact meaning of the states still open.

### 35011 — `pack1_env_ntc_mirror` (pack 1 ambient temperature)

- **Values:** 190-326 (scale 0.1°C → 19.0-32.6°C)
- **Justification:** In the long-term scan over 276 data points exactly identical to 34016 (pack1_env_ntc). No offset, no lag.
- **Confidence:** Very high.

### 35110 / 35111 / 35112 — power configuration block

**35110 — `max_charge_power` (max charge power)**
- **Values:** Constant 576
- **Justification:** Forms a block together with 35111/35112. 576 could be an internal reference value for the maximum charge power (possibly in internal units).

**35111 — `power_level` (power level, charging AND discharging)**
- **Values:** 500, 250, 100, 0
- **Justification:** When charging, the levels correlate with the CC-CV charge curve:
  - SOC < 90% → 500 → full load ~2035W
  - SOC 90-95% → 250 → ~1260W (half power)
  - SOC 95-100% → 100 → ~480W (CV phase)
  - SOC = 100% → 0 → pack finished
- **New (discharge steps test):** Also active when discharging! Switches between 100/250/500, presumably depending on pack rotation and current power utilization. Not only charge-specific.
- **Confidence:** Very high for the correlation. General power level indicator.

**35112 — `max_discharge_power` (max discharge power)**
- **Values:** Constant 500
- **Justification:** Analogous to 35110, but for discharging. 500 = same value as 35111 at full load.

### 37000 — `system_online` (system flag)

- **Values:** Constant 1
- **Justification:** First register in the 37xxx block, always 1. Could mean "system active".
- **Confidence:** Low — pure speculation.

### 37012 — `bms_version_mirror` (BMS version mirror)

- **Values:** Constant 116 (until 2026-07-08), **1177 from 2026-07-09** after the BMS update
- **Justification:** Identical to 30204 (bms_version). Redundant mirror in the 37xxx block.
- **Update 2026-07-09:** After the BMS FW upgrade to v117.7 the value jumps from 116 to 1177 — exactly in sync with 30204 and all 34x10 registers (see section "BMS FW update 116→117.7" below). Confirms the mirror character again, this time across a real version jump instead of just a constant value.
- **Confidence:** High → now ✅ confirmed instead of just a hypothesis.

### 37013 — `fault_status_mirror` (fault status mirror)

- **Values:** 0 (normal), 16 (0x0010) on battery disconnect
- **Justification:** In the battery disconnect test exactly identical to 36100 (fault_status). Both jump to 16 at the same time (bit 4 = BMS offline), both return to 0 at the same time. Redundant mirror in the 37xxx block, analogous to 37012 (bms_version_mirror).
- **Source:** battery_disconnect.csv (local, raw data)
- **Confidence:** High — exact match in timing and value.

### 36101 — `fault_status_2` (fault status word 2, bit 7 = AC/grid disconnect)

- **Values:** 0 (normal), 128 (0x80) on AC disconnect
- **Justification:** In the power cycle test without battery, 36101=128 appears exactly in the scans immediately before the AC disconnect:
  - Scan 18 (16:01:25): 128 → next scan after a gap (AC was disconnected)
  - Scan 26-27 (16:03:23-16:03:30): 128 → then a gap again (second AC disconnect)
- The inverter apparently detects the grid loss even before the TCP connection drops and sets bit 7.
- After AC reconnect (reboot): cleared automatically.
- **Source:** battery_disconnect_and_then_power_cycle_without_battery.csv (local, raw data)
- **Confidence:** High — exact timing pattern, reproducible with both AC disconnects.

### 37014 — `fault_status_2_mirror` (fault status 2 mirror)

- **Values:** 0 (normal), 128 (0x80) on AC disconnect
- **Justification:** Exactly identical to 36101 in timing and value. Completes the mirror pattern in the 37xxx block:
  - 37012 = 30204 (bms_version_mirror) ✅
  - 37013 = 36100 (fault_status_mirror) 🔍
  - 37014 = 36101 (fault_status_2_mirror) 🔍
- **Source:** battery_disconnect_and_then_power_cycle_without_battery.csv
- **Confidence:** High.

### 34x09 — `packX_protect2` bit 1 (0x0002) = low-SOC protection ✅ discharge test

- **Values:** 0 (normal), 0x0002 (bit 1) at low SOC
- **Justification:** In the long-term discharge test, bit 1 occurs on every pack as soon as the pack SOC falls below ~11%. On discharge below 10.5% it occurs again. Affects all 4 packs (34009, 34109, 34209, 34309).
- **Special value 0x0C93:** Pack 1 protect2 on complete shutdown (scan 225, SOC=0) — several protection bits at the same time.
- **Source:** discharge_under_dod.csv (local, raw data)
- **Confidence:** High — reproducible on all 4 packs.

### 37016 — `offgrid_ac_voltage` (off-grid AC voltage)

- **Values:** 2416/2393/2401 at standby/discharge, 0 when charging
- **Justification:** Values ≈ ac_voltage (30004/32200), but =0 when the inverter draws current from the grid (charging). Active only on feed-in or standby. Could be the AC voltage at the inverter output (not at the grid connection).
- **Confidence:** Medium.

### 37022 — `offgrid_dc_voltage` (off-grid DC voltage)

- **Values:** 513/534/527 at standby/discharge, 0 when charging
- **Justification:** Analogous to 37016, but DC side. Values ≈ dc_bus_voltage (30000/30028). Same pattern: =0 on grid import.
- **Confidence:** Medium.

---

## CC-CV charge curve analysis

The Marstek Venus D uses a **stepped CC-CV profile** with 3 power levels:

| Phase | SOC range | Power | Pack current | Reg 35111 |
|-------|-------------|----------|------------|-----------|
| CC (full load) | < 90% | ~2035W | 37.6A | 500 |
| CV stage 1 | 90-95% | ~1260W | 23A | 250 |
| CV stage 2 | 95-100% | ~480W | 9A | 100 |
| Finished | 100% | 0W | 0A | 0 |

The reduction to ~550W towards the end is therefore **normal behavior** (LiFePO4 CC-CV), not a defect.

### Round-robin charging
- Only 1 pack active at a time, rotation P1→P2→P3→P4
- ~3 minutes per pack (~10% SOC gain)
- Pack change recognizable from 34x04 (charge_status) and 34x01 (current)
- Inverter state (35100) flickers briefly 2→1→2 on the change

---

## Discharge steps test (discharging with power steps)

Discharge via app schedule with stepwise increase of the feed-in power from 100W to 2200W. 3 CSV files, 69 scans over ~29 minutes at SOC ~90-99%.

### Findings

**30006 = AC grid power (int16, ✅ confirmed)**
- Was 0 in previous single scans or appeared to be only the feed-in power
- Actually int16 with sign: **negative = grid import (charging), positive = feed-in (discharging)**
  - Charging full load: -2172W
  - CV phase: -541W
  - Idle: 0
  - Discharging: +852W
- **37004 ≠ exact mirror of 30006!** 37004 reacts ~1 scan faster than 30006 on a setpoint change (shows the new value earlier). 37004 is presumably the target/control value, 30006 the actual value

**43xxx = schedule configuration block (✅ confirmed)**

| Register | Value | Meaning |
|----------|------|-----------|
| 43000 | 0 | work_mode changes 1→0 when schedule is active |
| 43100 | 127 | weekday bitmask (0b1111111 = all 7 days) |
| 43102 | 2359 | end time (23:59 as HHMM integer) |
| 43103 | 100-2200 | discharge power setpoint in watts |
| 43104 | 1 | discharge active flag |

**30101 = pack 1 current mirror (🔍 newly identified)**
- Was 0 in ALL previous scans
- Now clear: mirrors 34001 (pack1_current) exactly when pack 1 is active
- On a pack change (pack 1→pack 2 at scan 33) immediately 0
- Not the total system current, but a pack-1-specific mirror

**35111 also active when discharging**
- Switches between 100/250/500 also during discharge
- Not only charge_power_setpoint, but a general power level indicator
- Renamed to `power_level`

**30029 also scales when discharging**
- Negative values proportional to the discharge power: -2 (idle) to -18 (2331W)
- Confirms its role as a power index/factor for both directions

---

## Battery disconnect test

Battery physically disconnected from the inverter (DC connector pulled) and reconnected. 18 scans over ~2 minutes.

### Findings

**36100 — fault_status bit 4 (0x0010) = BMS offline**
- Normal: 36100 = 0
- Battery disconnected: 36100 = 16 (0x0010)
- Battery reconnected: 36100 = 0 (after a few seconds)
- Confirmed: bit 4 is the BMS communication fault (battery unreachable)

**37013 = fault_status mirror**
- Previously listed as ❓ unknown, always 0 in all previous scans
- On disconnect: 37013 = 16, exactly like 36100
- New label: fault_status_mirror (🔍 hypothesis)

**35100 — inverter_state state change**
- With battery: 35100 = 1 (standby)
- Without battery: 35100 = 2 (active/searching)
- Interpretation: Inverter switches to an active search mode when the BMS does not respond

**35111 — charge_power_setpoint reset**
- On disconnect: 35111 = 0 (logical, no charging without a battery)

**34x08/34x09 — protection registers stay 0**
- Even on battery disconnect the pack protection bitmasks set no bits
- The protection registers apparently react only to BMS-internal events (overvoltage, overtemperature), not to communication loss

---

## Power cycle without battery

Battery physically removed → inverter could not be switched off via the power button → 2x AC disconnected/connected → battery reconnected. 41 scans over ~7 minutes.

### Timeline

| Phase | Scans | Time | State |
|-------|-------|------|---------|
| Normal | 1-5 | 15:59:20-15:59:49 | Battery in, SOC=99%, state=1 |
| Battery disconnected | 6-7 | 15:59:57-16:00:04 | SOC→0, 35111→0, fault not yet set |
| BMS offline | 8-17 | 16:00:11-16:01:18 | 36100=16, state=2, ~15s delay |
| AC disconnect 1 | 18-19 | 16:01:25-16:01:33 | 36101=128, then connection gone |
| AC reconnect 1 | 20-25 | 16:02:39-16:03:16 | Fresh boot, all faults cleared, state=2 |
| AC disconnect 2 | 26-27 | 16:03:23-16:03:30 | 36101=128, then connection gone |
| AC reconnect 2 | 28-38 | 16:04:06-16:05:48 | state=2, 37024=1345 spike at scan 36 |
| Battery reconnect | 39-40 | 16:05:56-16:06:03 | state→1 |

### Findings

**36101 bit 7 (0x80) = AC/grid disconnect**
- New fault bit, never observed before (was always 0)
- Appears exactly on AC disconnect, reproducible with both disconnects
- The inverter detects grid loss even before TCP drops
- After AC reconnect (reboot) cleared automatically

**37014 = 36101 mirror (newly identified)**
- Completes the mirror pattern: 37013=36100, 37014=36101
- The 37xxx block thus contains at least 37012-37014 as mirror registers

**37024 = 1345 (single spike)**
- Once at scan 36 (16:05:34), ~20s before battery reconnect
- Could be a BMS handshake, initialization value or timing artifact
- Requires further observation

**BMS fault timing**
- ~15 seconds delay between physical disconnect and fault bit (scan 6→8)
- SOC and charge_power_setpoint react immediately (scan 6), the fault takes longer

**AC reboot clears faults**
- After AC cycle without battery: 36100=0 (BMS fault cleared), even though the battery is still missing
- The inverter only sets the BMS fault again after a renewed timeout — in the test AC was disconnected again before that

**Inverter cannot be switched off without battery**
- Power button does not work without battery → only AC disconnect possible

---

## Boot scan (3x restart)

Marstek restarted 3x via power cycle, event scanner was running alongside.

### Findings

- **Always-0 registers are indeed placeholders** — not a single previously unknown register briefly showed a value at boot
- **42010 (force_mode) is reset to 0 on restart** — a set force mode is lost on power failure
- No boot-specific transitional states in the observed registers

---

## Power flow and inverter efficiency

### What do 30001, 30006 and 30007 measure?

The Marstek Venus has three central power registers. To understand what they show, it helps to picture the energy flow inside the device:

```
                                              ┌─── Backup output (30007)
                                              │    (30005 = voltage)
Power grid (AC 230V)  ←→  [Inverter]  ←→  [AC bus]
                          (losses)            │
                                              └─── Battery (DC ~530V)
                                                   (30001 = DC power)
```

**Register 30001 — Battery power (DC side)**

Shows how much power the battery is currently absorbing or delivering. Positive means the battery is being charged, negative means it is being discharged. This is the power directly at the battery, i.e. on the direct current (DC) side of the inverter. Important: this value shows only the DC power of the grid inverter, not the backup load (see below).

**Register 30006 — Inverter AC output power**

Shows the total AC output power of the inverter. Negative = grid import (charging), positive = AC output (discharging). Without backup operation this corresponds to the grid feed-in. In backup mode, however, this AC output is split between the grid and the backup output — 30006 then stays constant at the inverter limit (~2200W), even if less goes into the grid.

**Register 30007 — Backup output power**

Shows the power at the backup output (backup (EPS) power). It is 0 as long as no backup load is connected. Rises with the load on the backup output (0-3271W measured).

**Calculating the actual grid power:**

The actual feed-in or grid import results from:

`Grid power = 30006 - 30007`

Example: 30006=2208W, 30007=3271W → Grid = 2208-3271 = -1063W (1kW grid import, because the backup load exceeds the inverter).

### Why do the two values differ?

Between battery and grid sits the inverter. It converts direct current to alternating current (or vice versa). During this conversion, part of the energy is lost as heat. That is why the two values are never exactly equal:

| Process | 30001 (battery) | 30006 (grid) | Loss | Efficiency |
|---------|-----------------|-------------|---------|-------------|
| Charging full load | +2035W | -2172W | ~137W | ~93.7% |
| Charging CV phase | +480W | -541W | ~61W | ~88.7% |
| Discharging 850W | -905W | +852W | ~53W | ~94.1% |
| Discharging 2200W | -2331W | +2209W | ~122W | ~94.8% |

**How to read the table?**

When charging at full load, the inverter draws 2172W from the grid (30006 = -2172). Of that, 2035W arrive at the battery (30001 = +2035). The remaining 137W are lost as heat in the inverter. The efficiency is therefore 2035/2172 = 93.7%.

When discharging, the battery delivers 905W (30001 = -905). Of that, 852W are fed into the grid (30006 = +852). Again ~53W are lost as heat. Efficiency: 852/905 = 94.1%.

### Findings

- Efficiency is better at higher powers (~94-95%) than at low ones (~88-89%)
- Discharging is slightly more efficient than charging
- For the Home Assistant integration:
  - **Without backup:** use 30006 directly as grid sensor (30007=0, so grid=30006)
  - **With backup:** calculate `grid_power = 30006 - 30007`! 30006 alone does NOT show the grid power
  - Create **30007** as its own sensor "Backup power"
  - **30005** (scale 0.1) as "Backup voltage" — shows whether backup is active and how stable the supply is
- **30001 is relevant** if you want to monitor the state of charge and the load on the battery
- From the difference, a custom sensor "Inverter losses" can be calculated in HA: `abs(30001) - abs(30006)`
- **42021** shows the current inverter power limit (2200W in the test)

### 37004 — Grid power setpoint/control value

Register 37004 shows almost the same value as 30006, but reacts to power changes about one scan cycle (~20-30 seconds) faster. 37004 appears to be the internal setpoint the inverter regulates towards, while 30006 shows the actually measured power. The small deviations (+1 to +8W) between 30006 and the setpoint from 43103 confirm that 30006 is a real measured value and does not simply echo the configured value.

---

## Backup load test (backup (EPS) power with various loads)

**File:** `backup_load_activated.csv` — 28 scans, 08:12-08:22, SOC 40-44%, backup mode active, load increased step by step (52W to 3kW+).

### Timeline

| Phase | Scans | 30007 (backup load) | 30005 (backup V) | 30006 (inverter) | Grid effective | 30301 |
|-------|-------|--------------------|--------------------|------------------|---------------|-------|
| No backup | 1-2 | 0W | ~1V | 2209W | 2209W (feed-in) | 1 |
| ~90W load | 3-6 | 92-94W | 242V | 2209W | ~2117W | 1 |
| ~2kW load | 7-9 | 1871-1991W | 238V | 2208W | ~217W | 1→2 |
| ~90W load | 10-11 | 93-95W | 242V | 2209W | ~2116W | 2 |
| Ramp up | 12-13 | 134→647W | 242V | 2208W | ~1561W | 2 |
| ~2.2kW load | 14-17 | 2209-2226W | 238V | 2208W | ~-1W (neutral) | 2 |
| ~350W load | 18-21 | 352-354W | 242V | 2209W | ~1856W | 3 |
| ~2.1kW load | 22 | 2143W | 239V | 2208W | ~65W | 3 |
| ~3.2kW load | 23-25 | 3247-3271W | 237V | 2208W | ~-1063W (grid import!) | 2 |
| ~350W load | 26-28 | 267-366W | 242V | 2209W | ~1845W | 2→3 |

### Findings

**New registers identified:**

- **30005 = backup_voltage** (0.1V) — backup output voltage. 0V when backup is off, ~242V at low load, drops to ~237V at 3kW (normal voltage drop under load).
- **30007 = backup_power** (W) — backup output power. Was always 0 in ALL previous tests. Active for the first time with a backup load.
- **42021 = inverter_power_limit** — constant 2200W. Maximum inverter output power. Explains why 30006 never goes above 2209W.

**30006 correction — not grid power but inverter output:**

30006 shows the total AC output power of the inverter, NOT the grid feed-in. Without backup this was not distinguishable (30007=0, so grid=30006). In the backup test, 30006 stays constant at ~2208W although the actual grid power fluctuates drastically:

- Backup 92W → Grid: 2208-92 = 2116W feed-in ✓
- Backup 1991W → Grid: 2208-1991 = 217W feed-in ✓ (on-site observation: ~200W)
- Backup 2209W → Grid: 2208-2209 = -1W neutral ✓
- Backup 3271W → Grid: 2208-3271 = -1063W grid import ✓ (on-site observation: ~1kW)

**Energy balance:**

30001 (~2335W DC) also stays constant — the grid inverter keeps working at its limit unchanged. The backup load is served from the AC output power of the inverter. When the backup load exceeds the inverter power (~2200W), the difference is drawn from the house grid.

**32300/32301/32302 — off-grid block confirmed:**

- 32300 and 32301 are identical in ALL 28 scans — both show the backup voltage (like 30005). The FW description "32301=current" is wrong, it is also a voltage.
- 32302 mirrors 30007 (backup power) with a slight delay.

**30301 — Not Bluetooth:**

Changes 1→2→3→2→3 in the backup test. Previously suspected to be bluetooth_status (always 1). Correlates loosely in time with pack rotation (P1→P2→P4) but no direct pack index mapping. Function still unclear.

**Voltage drop under backup load:**

| Load range | Avg voltage (30005) | Avg power (30007) |
|-------------|-------------------|--------------------|
| < 200W | 241.9V | 100W |
| 200-1000W | 241.7V | 382W |
| 1-2.5kW | 238.4V | 2107W |
| > 2.5kW | 236.9V | 3257W |

---

## Backup overload test (overload at the backup output)

**File:** `backup_overload.csv` — 35 scans, 08:32-08:36, SOC 30-38%, backup mode with ~3.2kW continuous load.

### Timeline

| Phase | Scans | 30007 (backup) | 30001 (battery) | SOC | Fault? |
|-------|-------|---------------|------------------|-----|---------|
| ~360W load | 1-4 | 359-361W | -2340W | 37-38% | No |
| ~3.2kW overload | 5-29 | 3154-3227W | -2338...-2342W | 31-37% | **No!** |
| ~360W load | 30-34 | 275-382W | -2341W | 30-31% | No |
| Connection lost | 35 | 361W | -2341W | - | (scan incomplete) |

### Findings

**No overload protection at 3.2kW!** According to the specification, the Marstek is supposed to shut down at >2.5kW backup load. That does not happen:

- **All fault registers (36000-36103, 37013, 37014): 0 throughout** — not a single fault in 4 minutes of continuous operation at 3.2kW
- **All pack protection registers (34x08, 34x09): 0 throughout** — no BMS alarm
- **35100 stays at 3** (active operation), no state change
- **30001 stays at ~-2340W** — the grid inverter keeps working at its limit unchanged
- **SOC drops quickly** from 38% to 30% in 4 minutes (at 3.2kW backup + 2.2kW grid = ~5.4kW total draw)

The firmware apparently has no active overload protection for the backup output, or the limit is significantly higher than 2.5kW.

---

## Backup overload + AC disconnect test

**File:** `backup_overload_ac_disconnect.csv` (local, raw data) — 14 scans, 08:39-08:41, SOC ~30%, backup overload followed by AC disconnection.

### Timeline

| Phase | Scans | 30007 | 30001 | 30005 | 35100 | 36101 | Description |
|-------|-------|-------|-------|-------|-------|-------|-------------|
| ~360W backup | 1-4 | 353-362W | -2341W | 242V | 3 | 0 | Normal |
| ~3.2kW overload | 5-7 | 3159-3198W | -2340W | 237V | 3 | 0 | Overload, no fault |
| **AC disconnect** | **8** | **3208W** | **-2339W** | **237V** | **1** | **0x0080** | **Grid disconnected!** |
| Everything off | 9-13 | 0W | -12W | 0.5V | 1 | 0→0 | Complete shutdown |
| Conn. lost | 14 | 0W | -12W | 0.7V | - | - | Scan incomplete |

### Findings

**AC disconnect under backup load — complete shutdown:**

- **36101 = 0x0080 (bit 7)** — AC/grid disconnect fault, as already identified in the power-cycle test. Confirmed again.
- **37014 = 0x0080** — mirror of 36101, confirms the mirror hypothesis again.
- **Fault clears itself after ~20s** (scans 8-10 active, 0 again from scan 11).
- **35100 changes 3→1** — inverter goes from "active" to "idle/standby".

**Backup shutdown on AC disconnect + overload:**

On AC disconnection under 3.2kW overload, the Venus also switches off the backup output:
- 30005 drops from 237V to 0.5V (backup de-energized)
- 30007 drops from 3208W to 0W
- 30001 drops from -2339W to -12W (standby consumption only)

**CAUTION:** This does NOT mean that the Venus cannot do island operation. The Venus can do island operation in principle. The shutdown here was presumably caused by the combination of AC disconnect + overload (3.2kW >> 2.5kW limit). After the restart, the inverter had to be restarted to re-enable the backup socket.

**No overload faults here either** — 3.2kW backup in scans 5-8 without warning, the only fault comes from the AC disconnection.

---

## Backup without battery discharge (after reboot)

**File:** `backup_load_activated_no_battery_discharge.csv` (local, raw data) — 3 scans, 08:47-08:48, SOC ~30%.

After the restart (due to backup shutdown caused by overload + AC disconnect), the Venus supplies the backup socket only from the grid, not from the battery.

### Measured values

| Register | Value | Meaning |
|----------|------|-----------|
| **42010** | **0** | Force mode off! (before: 2 = force discharge) |
| 30001 | -34W | Standby consumption only, no discharge |
| 30006 | 0W | No grid feed-in |
| 30007 | 122-124W | Backup load is being served |
| 30005 | 239.2-239.5V | Backup voltage active |
| **35100** | **6** | **New inverter state!** (so far only 1 and 3 seen) |

### Cause

**42010 = 0 after reboot** — this is the known effect: force mode is reset on restart (boot scan finding). Without 42010=2 (force discharge), the Venus does not discharge the battery. The backup load is instead supplied entirely from the grid.

Energy balance confirms: Grid = 30006 - 30007 = 0 - 124 = -124W → the backup load is drawn from the grid.

### New inverter state: 35100 = 6

All known states:
- **0** = Shutdown (complete shutdown, SOC=0)
- **1** = Idle/standby (after AC disconnect, boot, reserve cutoff)
- **2** = Charging
- **3** = Discharging (active discharge with grid feed-in)
- **4** = Island operation (backup from battery without grid connection, 36101=0x0080)
- **6** = Backup passthrough (DOD reached, grid→backup pass-through, no battery discharge for the grid)

---

## Long-term discharge below DOD + island operation + reserve cutoff

**File:** `discharge_under_dod.csv` — 548 scans, 08:53-12:10 (~3h17min), SOC 30%→7%, backup active, 4 packs.

The most extensive test so far: discharge below the DOD limit, AC disconnect, island operation and final shutdown at the reserve limit.

### Phase 1 — Normal discharge (scans 1-161, 08:53-09:51, SOC 30%→10%)

35100=3, 42010=2. Grid feed-in ~2209W + backup ~360W. Standard round-robin discharge:

| Order | Pack | SOC before | SOC after | Duration |
|-------------|------|-----------|------------|-------|
| 1 | P2 | 30.0% | 20.6% | ~5 min |
| 2 | P4 | 30.8% | 20.7% | ~6 min |
| 3 | P1 | 30.7% | 20.7% | ~5 min |
| 4 | P3 | 30.7% | 10.7% | ~14 min |
| 5 | P4 | 20.7% | 10.7% | ~9 min |
| 6 | P1 | 20.5% | 10.7% | ~18 min |
| 7 | P2 (last) | 20.5% | → Phase 2 | continues |

Each pack is discharged individually down to ~10.7%, then it switches to the next one with the highest SOC. P2 remains as the last one (19.6%).

### Phase 2 — Backup-only below DOD (scans 162-221, 09:51-10:12, SOC=10%)

**35100 jumps to 6** — grid feed-in stops (30006=0), 30001 drops to -35W. Backup is supplied from the grid (passthrough). Only Pack2 still has charge left (19.6%), discharging at minimal current (-3 to -4, presumably self-consumption).

DOD limit: system SOC ~10% = all packs at ~10.7% except the last active one.

### Phase 3 — AC disconnect + crash (scans 222-225, 10:12-10:14)

35100 changes to **4** (island operation attempt), 36101=0x0080. At ~2kW backup load (30007=1344W scan 223) the system collapses: scan 225 SOC=0, 35100=0.

34009=0x0C93 at shutdown — several protection bits active simultaneously.

### Phase 4 — Reboot + island operation (scans 226-540, 10:15-12:07)

After reboot: 35100=**4** (island operation), 42010=0, 36101=0x0080 **stays permanently set**.

30006=0 (no grid), backup is supplied ONLY from the battery. This confirms: **the Venus CAN do island operation!** The crash in phase 3 was overload, not a missing capability.

Pack2 is the only one with reserves (19.3%) and is discharged slowly:
- Scans 229-499: Pack2 from 19.3%→12.7%, low current (-21 to -69), backup ~330W
- Scan 500: load increased to ~1575W, Pack2 current rises to ~-345
- Scan 514: load increased to ~2040W, Pack2 at 11.1% → rotation begins

### Phase 5 — Rapid rotation + reserve cutoff (scans 514-541, 11:58-12:08)

When all packs are near 11%, **frantic pack switching** begins — every 2-3 scans (~40-60s) instead of the usual ~10 minutes:

```
P2(11.1%)→P4(11.5%)→P1(11.0%)→P3(11.0%)→P1(10.6%)→P4(10.6%)→P2(10.5%)→P1(9.9%)
```

Each pack is discharged only 2-3% before switching. The system tries to distribute the load evenly.

**34x09 bit 1 (0x0002) = low-SOC protection:**

| Pack | First trigger | SOC | Second trigger | SOC |
|------|----------------|-----|------------------|-----|
| P3 | Scan 82 | 11.2% | Scan 521 | 10.4% |
| P4 | Scan 107 | 11.8% | Scan 528 | 10.3% |
| P1 | Scan 158 | 11.4% | Scan 526 | 10.1% |
| P2 | Scan 511 | 11.9% | Scan 530 | 10.5% |

Bit 1 occurs twice for each pack: once on first reaching the DOD limit (~11%), and again on the final deep discharge below 10.5%.

**Shutdown at Pack1 = 7.9%:**

Scan 540 (12:07:41): Pack1 reaches 7.9% → 35100 changes to 1 (idle), backup stops. The other packs were still at 9.8-10.0%. The reserve limit therefore applies **per pack**, not as a system average — as soon as the **first** pack reaches ~8%, it is over.

### DOD and reserve findings

```
100% ─────────────── Full
 │
 │   Normal operating range (grid + backup)
 │
12% ── DOD limit ── Grid feed-in stops (35100: 3→6)
 │
 │   Backup reserve (backup (EPS) power only, no grid)
 │
~8% ── Reserve limit ── Complete shutdown (per pack, not system!)
```

- **DOD = 88%** (100% → 12% usable for grid feed-in)
- **Reserve = ~4%** (12% → ~8% additionally for backup/backup (EPS) power)
- **Reserve cutoff is per pack** — when ONE pack reaches 8%, everything stops (even if others still have 10%)
- **Below 11% SOC: rapid rotation** — packs switch every 40-60s instead of every 10 minutes

### Island operation confirmed

The Venus can do island operation (35100=4):
- 36101=0x0080 indicates missing grid connection
- Backup is supplied from the battery (30001 negative, 30006=0)
- Works until the reserve cutoff (~8% per pack)
- Limitation: with overload (>2.5kW) during the switchover to island operation, the system can crash

---

## Backup below DOD + EMS forced charging on AC reconnect

**Scans:** `unter_dod_backup_lässt_sich_nicht_einschalten_zwangsladung_bei_ac_connect.csv` (local, raw data) (21 scans, scan_powercycle, 12:11-12:15) + `zwangsladung_unter_dod.csv` (local, raw data) (7 scans, 419 registers, 12:16-12:18)

**Starting point:** System SOC ~7-8%, below DOD (12%), AC was still disconnected from the previous test (36101=0x0080).

### Backup cannot be activated below DOD

Scans 1-11 (file 1): Despite backup mode being enabled, 35100=1 (idle) remains. 30007=0 (no backup output), 30005=0.4-0.8V (backup output inactive). The EMS refuses backup activation when SOC is below DOD — the emergency reserve is protected.

### AC reconnect and fault clearing

Scan 6 (12:13:18): 36101 jumps from 0x0080 to 0x0000 — AC was reconnected. The AC disconnect fault clears itself on reconnect (no reboot needed). Nevertheless 35100=1 remains — no immediate state change.

### Automatic forced charging by the EMS

Scan 12 (12:14:19, ~60s after AC reconnect): 35100 changes to 2 (charging). From scan 13: 30001=~920W charge power, 30006 as int16 = -990W (grid import). Critical: **42010=0** — the forced charging is NOT controlled via force_mode, but is internal EMS behavior. 42011 (charge_to_soc), 42020 (set_charge_power) and 42021 (set_discharge_power) also all remain 0.

### Detail data from the full register scan (file 2)

7 scans during active forced charging show:

- **Only pack 2 is being charged:** 34104=3 (charging), all other packs 34004/34204/34304=0 (idle)
- **Pack SOCs:** P1=8.0%, P2=13→14%, P3=10%, P4=11%
- **Pack 2 current:** 34101=178-179 (17.8-17.9A charge current)
- **Charge power:** 30001=919-920W constant
- **Grid import:** 30006 int16 = -990W (losses: ~70W for inverter self-consumption)

Noteworthy: P1 has the lowest SOC (8%) but P2 is being charged (13→14%). Possible explanation: P1 was already charged earlier (forced charging has been running for ~2 min), or the EMS follows the normal pack rotation instead of charging the lowest one first.

### Summary of forced charging

```
State machine below DOD:
  SOC < DOD + AC disconnected → 35100=1 (idle), backup blocked
  SOC < DOD + AC connected → ~60s delay → 35100=2 (charging)
  Forced charging: ~920W, grid import ~990W, 42010=0 (no force_mode)
  Only 1 pack charged at a time, normal charge_status=3
```

For the HA integration: if 35100=2 and 42010=0 and SOC < DOD → the system is in EMS forced charging (not user-initiated).

---

## FW update 149.2 + 6 packs

**Scans:** `fw_upgradr_auf_149_2.csv` (local, raw data) (22 scans, scan_powercycle) + `laden_von_unter_dod_mit_6_packs_und_massivem_soc_unterschied_nach_upgrade_auf_149_2_fw.csv` (local, raw data) (5 scans, 419 registers)

### FW version changes

Only 30200 (EMS) has changed: 148→1492 (v149.2, new format x10). MPPT (30205=104), VNS (30202=115), BMS (30204=116) all unchanged. During the update the device was ~25 minutes offline (scans 11-22 completely empty).

### 6 packs confirmed

Pack 5 (34400-34433) and Pack 6 (34500-34533) now show full data — identical register structure to packs 1-4. Pack 7 (34600+) remains empty.

- Pack 5: 54.03-54.27V, 37.5A charge current, SOC 51.3→59.6%, 49 cycles, BMS v116
- Pack 6: 52.83-52.85V, 0A (idle), SOC 50.0%, 45 cycles, BMS v116

### 30212 and 31003/31004 are pack-dependent

Correction to the earlier analysis: these registers depend on the number of connected packs, not on the FW version.

| Register | 4 packs (FW148) | 6 packs (FW149.2) | 6 packs (FW150) |
|----------|-----------------|-------------------|-----------------|
| 30212    | 2               | 5                 | 5               |
| 31003    | 0               | 5                 | **0**           |
| 31004    | 0               | 1543 (0x0607)     | **0**           |

The exact meaning is still unclear — 30212=5 and 31003=5 with 6 packs is not a direct mapping to the pack count.

**Update 2026-08-14 (v150 scan, `control_150_vns_116.csv` (local, raw data), 6 packs):** 30212 stayed 5, but
**31003 and 31004 are 0 again under FW150** — although 6 packs are still connected.
So they are *not* purely dependent on the pack count, but state/version dependent
(possibly no longer used under v150). Important: 31003/31004 carry real values in the 149.2 scans
(5 and 1543 respectively) and are therefore **not device_name padding**, even though the FW read handler nominally lists the block
31000–31009 as `device_name`. In the register map they are therefore listed as
`pack_dependent_31003/_31004` (hypothesis), 31005–31009 (0 in all scans) as unknown.

**Measured 2026-10-03:** 30212 is the boot self-test result (`Board_PeripheralSelfTest` @ 0x0801dcf4), not a pack-dependent value. Two real Venus D units (BMS 118, Micro 116, Control 151) both read 5 although LAN works; CH395 version register 45605 returned 0x4A on one unit while the self-test expects 0x4B. Stage 5 with a working LAN is harmless; real faults are 2 (EEPROM) and 3 (flash).

### EMS charging strategy with SOC difference

123 scans over 3.5 hours show the complete equalization strategy:

1. P5 charged first (51→60%), then P6 (50→60%) — lowest SOC first, sequentially
2. Then round-robin through all 6 packs, each up to the next 10% step: 60→70→80→90%
3. Only 1 pack active at a time (charge_status=3)
4. Charge power: ~2030W up to 80% SOC, then manually reduced to ~900W, from ~90% manually to ~440W

### Temperature comparison: with vs. without fan (long-term)

Pack 1 cell temperature (34011) over 3.5h of charging:
- Start (without fan): 39.7°C, rising
- Peak (without fan): **41.9°C** at scan 46 (~16:27, after ~75 min)
- After placing the fan: continuous decline
- End (with fan): **36.4°C** at scan 123 (~18:35)

The fan lowers the cell temperature by ~5°C.

### Inverter temperatures (35000/35001/35002)

Registers 35000 (inverter internal temp), 35001/35002 (MOS temps) show the actual enclosure heat — and correlate directly with the charge power:

| Charge power | 35000 (internal) | 35001/35002 (MOS) |
|-------------|----------------|-------------------|
| 2030W       | 50→54°C (rising) | 53→54°C        |
| 1380W       | ~37°C          | ~37°C             |
| 900W        | ~33°C          | ~33°C             |
| 440W        | ~31°C          | ~31°C             |

Peak: **54.2°C** (35001) after ~17 minutes at 2kW continuous load. 35001 and 35002 are identical in all scans. 30002 and 30003 are mirrors of 35000 and 35001, respectively.

**Anomaly pack 4+5:** For pack 4 and pack 5, 34x12 (min_cell_temp) is partly higher than 34x11 (max_cell_temp). Possibly the registers are assigned differently for certain packs, or max/min refer to different sensor groups.

---

## BMS FW update 116 → 117.7

**Scan:** `nach_bms_upgrade_auf_117_7_fw.csv` (local, raw data) — 4 scans, 15:43-15:48 (2026-07-09), 6 packs, ~50% SOC, otherwise standby/idle.

### Only BMS version affected

Of all 419 scanned registers, only the BMS version registers changed. All other firmware version registers remained unchanged:

| Register | Before | After | Meaning |
|----------|--------|---------|-----------|
| 30200 (ems_version) | 1492 | 1492 | unchanged (Control FW still 149.2) |
| 30202 (vns_version) | 115 | 115 | unchanged (VNS still v115) |
| 30204 (bms_version) | 116 | **1177** | **v116 → v117.7** |
| 30205 (mppt_version) | 104 | 104 | unchanged |
| 34010/34110/34210/34310/34410/34510 (packX_bms_version) | 116 | **1177** | all 6 packs updated in sync |
| 37012 (bms_version_mirror) | 116 | **1177** | mirror confirmed |

### x10 encoding pattern confirmed (2nd observation)

The raw value jump 116→1177 follows exactly the same pattern as ems_version during the Control FW update to v149.2 (147→1492, see section "FW update 149.2 + 6 packs"): As long as the version number has no decimal place, the plain integer is used as the raw value (116 = v116). As soon as a decimal place is added, the register switches to ×10 encoding (1177 = v117.7). This has now been observed in two independent version registers (ems_version, bms_version) — presumably a general behavior of the descriptor table for all `_version` registers, not just a single case.

### Cross-validation with static analysis

The BMS binary previously analyzed with Ghidra (`20251010135647565eb2036.bin`, see `BMS/VNSD_BMS_FW_Analysis_v117.7.md`) had already contained a hardcoded version value `0x499` = 1177 decimal — i.e. v117.7, before the update was rolled out live. The current live scan confirms exactly this predicted value. The file was originally mistakenly titled "v177.7" (typo); corrected to v117.7.

---

## Scan scenarios still open

### Recommended further scans
1. **PV charging (solar)** — If PV is connected: MPPT registers 30020-30040 should then show values. Completely different charge path.
2. ~~**Fault provocation**~~ — partly done: battery disconnect test revealed 36100 bit4 and 36101 bit7. Protection registers 34x08/34x09 stay 0 even on disconnect — apparently they only react to BMS-internal events. Real overload/overtemperature still open.
3. **Force-mode switching** — toggle 42010 during the scan (charge→discharge→auto) and scan while doing so. Shows transition registers.
4. **Scanning with PV power** — Since no scan with PV power has taken place yet, the charge, discharge and backup scenarios still have to be scanned and added. Especially with regard to the registers concerning AC-side feed-in and grid import as well as battery power.
