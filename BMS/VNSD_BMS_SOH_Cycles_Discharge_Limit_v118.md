# Venus D — BMS 118: SOH, Cycle Counter and Discharge Current Limit

> 🇩🇪 Deutsche Version: [VNSD_BMS_SOH_Zyklen_Entladelimit_v118_de.md](VNSD_BMS_SOH_Zyklen_Entladelimit_v118_de.md)

> **Note (2026-09-30):** Flash addresses in this document come from the old analysis with the wrong base address (0x2800 too low). Overall analysis with correct addresses: [VNSD_BMS_FW_v118_Full_Analysis.md](VNSD_BMS_FW_v118_Full_Analysis.md).

**Status:** 2026-09-24
**Applies to:** Venus D, BMS 118. The SOH and cycle logic is present byte-identical in 117.7.
**Sources:** Ghidra `/VNSD/BMS/BMS_118_…`, raw image, decompilation export `Dekompilate/VNSD_BMS_118/` (local, not in the repo).
Control side: `/VNSD/Control/VNSD-0_app_0150_…`.

The 117.7 documentation (§4) describes the SOC algorithm. Two parts of it were misinterpreted: the
supposed OCV correction at rest and the "Quick OCV table". Both belong to the
SOH calculation (§6).

---

## 1. Key findings

1. **SOH is pure ageing by cycle count.** It is not measured but interpolated over the cycle count from a
   table with 7 support points. The table lives in the calibration flash behind the image end and is not readable.
2. **Cycles = energy throughput ÷ (2 × 2.56 kWh).** Charging and discharging both count. The
   cycle count is rounded down and appears per pack in Modbus **34x03**.
3. **SOH does not leave the BMS.** There is no CAN, RS485 or Modbus field for it, only the
   debug shell (`Soh:`). The Control's cloud upload contains `soh=0`. An SOH in HA has to be calculated by the
   integration itself.
4. **The SOH controls the capacity of the Coulomb counting:** nominal capacity 50 Ah × SOH. As the
   SOH drops, the pack SOC moves faster per Ah.
5. **The discharge current limit depends on the lowest cell voltage, not on the SOC.**
   The charge current limit, in contrast, depends on the displayed SOC.
6. **Register 32105** (`bat_energy`) is the fixed nominal energy: pack count × 50 Ah × 51.2 V,
   17.92 kWh with 7 packs (`CAN_TX_PF2_Capacity`). It is neither SOH-adjusted nor the remaining energy.
7. **Discharging stops in the Control at the total SOC** as soon as it reaches the backup reserve
   (default 12 %). The pack SOCs play no role in this.

---

## 2. SOC structure (base `0x20004A38`)

The layout is derived from the debug output `CLI_Print_SOC_Status` (0x080138E0) and the write
locations. It corrects the table in `VNSD_BMS_FW_Analysis_v117.7.md` §4.5.

| Offset | Address | Field | Unit |
|---|---|---|---|
| +0x00 | `0x20004A38` | real_soc | 1e-4 % |
| +0x04 | `0x20004A3C` | show_soc (displayed, goes into Modbus 34x02) | 1e-4 % |
| +0x08 / +0x0C | | max_soc / min_soc | 1e-4 % |
| +0x10 | | target_soc | 1e-4 % |
| +0x24 | `0x20004A5C` | **Cycle count** (u16) | 1 |
| +0x28 | `0x20004A60` | **SOH** (u32) | **1e-4 %**, 1,000,000 = 100 % |
| +0x2C | `0x20004A64` | **Energy throughput** charge + discharge (u32) | **0.1 kWh** |
| +0x30 / +0x32 | | max. charge / discharge current | 0.1 A |
| +0x34 / +0x36 | | max. / min. cell voltage | mV |
| +0x38 | | Temperature | 0.1 °C |
| +0x3E | | Current | 0.1 A |
| +0x40 | | Pack voltage ("Tal Vol") | 0.1 V |

**Corrections to 117.7 §4.5:** The throughput is listed there as `total_capacity` (capacity in kWh).
The SOH is given there with ÷10, but the unit is 1e-4 %. The debug shell does in fact divide
by 10 and would print "100000.0" at 100 %. That is a display bug of the
firmware, not the value.

---

## 3. Energy throughput

`Coulomb_Counting` delivers every 100 ms (SOC task: `SOC_Algorithm_Entry(100)`, then
`vTaskDelay(100)`):

```
q  = I[0,1 A] × dt[ms] / 36                → µAh
dE = |q| × U[0,1 V] / 100                  → 10 µWh
```

`Charge_Energy_Calc` and `Discharge_Energy_Calc` sum `dE` in one accumulator each
(`0x200029F4` charge, `0x200029FC` discharge). Whole 10⁷ units (= 0.1 kWh) are moved each time into
`+0x2C`, the remainder stays in the accumulator. **Both directions increment the same counter.**

`+0x2C` is persisted together with the real/show/max/min SOC in an **EEPROM ring buffer starting at
`0x400`**, 50 slots of 20 bytes each (wear levelling; `SOC_EEPROM_LoadCalibration` searches for the most recent
slot at startup). `SOC_EEPROM_SaveIfChanged` writes as soon as real_soc has changed by more than 1 %
or the throughput by at least 0.1 kWh.

---

## 4. Cycle count — `CycleCount_From_Throughput` (0x0800FAE4)

```c
zyklen = (durchsatz / 10.0) / (2.56 × 2.0);   // kWh / (2 × nominal energy), rounded down
```

The 2.56 kWh is hard-coded in the image as a float constant (`0x0800FB34`), not a
configuration. One cycle is therefore 5.12 kWh of throughput, 2.56 kWh in and 2.56 kWh out.

The cycle count is recalculated **only on the transition to the rest state** (§6), and additionally
once at startup. It is an integer and therefore jumps in steps. It is transmitted
via the pack structure (+0x24 "Cyc Cnt") up to Modbus **34x03**.

Field values on 2026-09-24: P1 9, P2 30, P3 28, P4 33, P5 32, P6 75, P7 69. That corresponds to roughly
46 to 384 kWh of throughput per pack.

---

## 5. SOH — `SOH_From_Throughput` (0x08010324) / `SOH_Table_Lookup_ByCycles` (0x08011534)

```c
zyklen = CycleCount_From_Throughput(durchsatz);
if (zyklen valid) soh = SOH_Table_Lookup_ByCycles(zyklen);
soh = min(soh, 1,000,000);                    // at most 100 %
```

The table has 7 entries of 8 bytes each, `{zyklen_schwelle u32, soh u32}`, at **`0x0801B874`**.
That is behind the image end `0x08019FFF`, in the calibration flash. Between the support points it is
interpolated linearly. Below the first support point its value applies, above the last one the value of the
last entry. The table presumably comes from the manufacturer's cell specification.
Without an SWD dump it remains unknown.

Usage:

- `Coulomb_Counting`: `capacity factor = soh / 1e6`, i.e. effective capacity
  = 50 Ah × SOH. With factor 0 the function does not count.
- `SOC_Init_Capacity_Params`: at startup sets up SOH in % (u16, `0x20004ADE`) and capacity values
  derived from it. `0x20004ADE` is never updated afterwards and is not read by any
  transmit function.

**Not passed to the outside:**

- `PerPack_Struct_Builder`, `CAN_TX_PerPack_10Msgs` and `RS485_Pack_Telemetry_Response`
  contain no SOH.
- The Control's pack structure has a field `bat_soh` at +0x5A (debug output
  `bat soh =%d`). No CAN frame fills it, and it is not mapped to any Modbus register
  (pack 1: `0x2001500E`).
- The cloud upload has the key `soh`, with the value 0 in the capture
  (`Control_FW/VNSD_Cloud_Upload_Payload_v150.md`).
- BLE: `BLE_Build_BMS_Data_Response` (Control 150, 0x0800B4F4) puts exactly this
  never-populated pack 1 field +0x5A (`0x2001500E`) into the response as word 5. The BLE SOH is thus also 0.
  Word 11 of the same response is the **remaining energy** = SOC (32104, 0.1 %) × nominal energy
  (32105, Wh) ÷ 10000 in 10 Wh. It is not SOH-adjusted either; HA "stored energy"
  13.08 kWh at 73 % × 17.92 kWh fits this.

**Consequence for the HA integration:** An SOH can only be emulated from Modbus: cycles per pack
from 34x03, plus a custom ageing curve. We do not know the firmware's 7 support points.
A capacity measurement (Ah drawn between two full detections) is independent of this and
would be the better measure, because the firmware itself measures nothing.

---

## 6. When SOH and cycles are recalculated

`SOC_Algorithm_Orchestrator`, state 3 (rest), only on the **first** pass after charging or
discharging:

```c
if (vorheriger_zustand != 3) {
    RTC timestamp → EEPROM address 0xFC02 (10 bytes);
    soh    = SOH_From_Throughput(durchsatz);        // until 2026-09-24 "OCV_Correction"
    zyklen = CycleCount_From_Throughput(durchsatz); // until 2026-09-24 "OCV_Voltage_To_Index"
}
```

`SOC_OCV_Bounds_Check_And_Recalibrate` (0x08013E34), which the orchestrator also calls on every
pass, is dead code. Its trigger flag `0x200029CA` is never set
(`VNSD_BMS_FW_Analysis_v118.md` §4.7).

The real rest recalibration is separate from this. After 3,600,000 ms (1 h) of rest the
orchestrator checks min/max cell voltage against thresholds from the table at `0x0801B8D4` and, if applicable, calls
`Full_Recalibration`. That remains as described in 117.7 §4.4.

At startup: `SOC_Init_SOH_Cycles_From_Throughput` (0x0801351E, until 2026-09-24
`SOC_Calculate_OCV_From_Voltage`) loads the throughput from the EEPROM ring and calculates both
once.

---

## 7. Current limiting

### 7.1 Charging — `Max_Charge_Current_Calc` (0x0800FE60)

Row = temperature index (18 levels, thresholds `0x0801B97E`), column = SOC zone (12 zones,
thresholds `0x0801B9A2`), matrix `0x0801B9BC`. What matters is **show_soc** in whole percent.
Entry × 0.5 A, at most 50 A. At show_soc = 100 % → 0 A. Field values:
`VNSD_BMS_Pack_Rotation_Taper_v118.md` §8.

### 7.2 Discharging — `Max_Discharge_Current_Calc` (0x0800FF58)

- **Row:** temperature index as for charging.
- **Column:** one of 5 levels of the **lowest cell voltage** (`+0x36`), not of the SOC.
- There are two threshold tables: temperature index < 5 → `0x0801BB76`, otherwise `0x0801BB6C`.
- Matrix `0x0801BB80`, 5 columns per row. Entry × 0.5 A, at most 50 A.
- All tables are in the calibration flash, none is readable.

`SOC_Current_Bracket_Lookup` has a **hysteresis**: downwards the level changes immediately, upwards
only when the cell voltage is at least **100 mV** above the lower limit of the current
level. Below the second threshold, level 0 applies, presumably 0 A. This is the
voltage-based discharge lower limit. The temperature axis (`Temperature_Index_Lookup`,
18 levels) also has a hysteresis.

For the remaining time until empty this means: as long as the cell voltage does not fall into the lower levels,
the BMS does not limit the discharge (up to 50 A). At the end of a pack the power may drop,
depending on voltage. This has not been measured in the field.

`Master_Build_DischargeCurrentLimit` takes the value of the active pack **without**
forming a minimum (`VNSD_BMS_FW_Analysis_v118.md` §11.2). Discharge voltage limit fixed at 48.0 V.

---

## 7b. Full and empty detection (SOC jumps)

`Charge_SOC_Integration` / `Discharge_SOC_Integration` (charging and discharging state respectively). What matters
is the **highest or lowest cell voltage** of the pack, not the Coulomb counter:

| Condition | Effect |
|---|---|
| Max Cell ≥ **3600 mV** (temperature < 0 °C: 3700 mV), `SOC_Is_Full_ByMaxCell` | SOC immediately **100 %** (`SOC_Set_Full_Charge_1000000`) |
| Max Cell 3501–3549 mV | SOC is pulled voltage-proportionally towards **96 %** |
| Max Cell 3551–3599 mV | SOC is pulled voltage-proportionally towards **100 %** |
| Max Cell < 3500 mV | Coulomb counting; show_soc follows a deviating target_soc at half or double speed respectively |
| before full detection | show_soc at most **99.0 %** |
| Min Cell ≤ **2750 mV** (temperature < 0 °C: 2420 mV), `SOC_Is_Empty_ByMinCell` | SOC immediately **0 %** |
| Min Cell < 2900 mV | `SOC_Voltage_Drop_Compensator` pulls the SOC down; held at 1.0 % before empty detection |

This explains the field finding from 22.09.: 97.2 → 98.6 → 100 % in 30 s at 56.8 V pack voltage
(≈ 3.55 V per cell on average, highest cell at the 3.6 V limit). **For a
remaining-time display this means:** The last percent before 100 % (and before 0 %) do not run linearly
with the energy. The end is determined by the cell voltage, not the Ah balance.

The protection limits independent of this (overvoltage 3700–3900 mV → SOC 100, undervoltage
2500/2300 mV → SOC 0) are in `VNSD_BMS_FW_Analysis_v118.md` §4.6.

`BMS_Check_FullCharge_Condition` (0x08006F34) was misnamed and is now called
`BMS_Measurements_Valid_Check`. The function sets bit 5 in `0x20004B28` as soon as 100
consecutive measurements were valid (all cells ≥ 1000 mV, NTC between −40 and 120 °C).
Only after that does the SOC algorithm run at all.

**Display SOC after full detection** (`SOC_Display_And_FullFlag_Update`): The full flag holds the
display at 100.0 % until discharging occurs or the SOC falls below 99.5 %.

## 7c. Cell balancing — only at the top, only while charging

`KA495XX_Cell_Balance_Decision` (0x08014C98) with the cell voltage structure `0x20004068`:
cells from +0x0C, min. cell +0x54, max. cell +0x58, all in mV.

| Condition | Effect |
|---|---|
| max − min < **30 mV** or `s32@0x20004070` ≤ 1000 | no balancing, mask cleared |
| Cell ≥ **3501 mV** and ≥ 30 mV above the lowest (debounced) | cell is balanced |
| Cell < 3500 mV and < **20 mV** above the lowest (debounced) | balancing of the cell ends |
| all cells would be balanced | none (mask 0) |

`0x20004070` is calculated by `KA495XX_Compute_Power_And_Energy` from a BMIC measurement value. It is presumably
the charge current in mA, i.e. condition > 1 A. This is not firmly evidenced.

`KA495XX_Cell_Balance_StateMachine` (0x08014D90) switches **alternately the odd and the
even cells** (mask `0x20003C70 + 0x3B4`, 4 passes each). Two adjacent
cells are never active at the same time. The balancing bits go into the BMIC registers `0x15`/`0x16`,
controlled via registers `0x0C` and `0x17`. The function `APP_CAN_Send_With_Validation` is
**not a CAN transmission** but a BMIC register access (register, value, mask). The name is
wrong, but left untouched.

**Consequence:** Balancing only happens when a cell is above 3.5 V. With LFP that is only at the very top,
in the last minutes before 100 % (field 22.09.: ≈ 3.50–3.55 V from ~97 %). A pack that is rarely
full practically never balances. Regular full charges of each pack are the only
lever for this.

Field status 24.09. at ~70 % SOC: 2–3 mV cell difference **within** each pack (HA
`zellspannungs_differenz`). The 127 mV from Omnibattery (`cell_delta`) is the span **across all
packs** and reflects the different pack SOCs, not a cell imbalance. At 70 % a
small cell difference says little; it only becomes meaningful at the top, from ~3.4 V.

## 8. Control: backup reserve and power cap

`Inverter_Power_Setpoint_Calc` (Control 150, 0x08006168) applies to every setpoint:

| Condition | Effect |
|---|---|
| Setpoint > 0 (discharging) and reserve `u8@0x2000012C` ≥ total SOC (`0x20014F94`, 32104) | Setpoint = 0 |
| Setpoint > `u16@0x2000012F` (44003) | capped to 44003 |
| Setpoint < −`u16@0x2000012D` (44002) | capped to −44002 |
| Setpoint < P_bms | to P_bms (`Control_FW/VNSD_Control_Charge_Power_PV_Curtailment_v150.md` §2) |
| Flag `0x200002BA` set and setpoint > 0 | Setpoint = 0 |

The reserve is `100 − app DoD`, default 12 %, app range DoD 30–88 %
(`Modbus_RS485_TCP/VNSD_Register_Persistence_RAM_vs_EEPROM_v150.md`). Whether the limit takes effect exactly at SOC =
reserve cannot be determined with certainty from the fp64 comparison in the decompilation.

**Reconciliation with the field (20.09.):** Omnibattery switched off at **pack** SOC 12.0 %. The
total SOC was around 21 % at that time (P3/P4/P6 at 28.9 %). The Control reserve cannot
have been the cause. This fits Omnibattery as the cause.

---

## 9. Ghidra renames (2026-09-24)

| Program | Address | before | after |
|---|---|---|---|
| 118 | `0x08010324` | `OCV_Correction` | `SOH_From_Throughput` |
| 118 | `0x0800fae4` | `OCV_Voltage_To_Index` | `CycleCount_From_Throughput` |
| 118 | `0x08011534` | `Quick_OCV_Lookup` | `SOH_Table_Lookup_ByCycles` |
| 118 | `0x0801351e` | `SOC_Calculate_OCV_From_Voltage` | `SOC_Init_SOH_Cycles_From_Throughput` |
| 118 | `0x080138e0` | `FUN_080138e0` | `CLI_Print_SOC_Status` |
| 117.7 | `0x080102b4` | `OCV_Correction` | `SOH_From_Throughput` |
| 117.7 | `0x0800fa74` | `OCV_Voltage_To_Index` | `CycleCount_From_Throughput` |
| 117.7 | `0x080114c4` | `Quick_OCV_Lookup` | `SOH_Table_Lookup_ByCycles` |
| 117.7 | `0x080134ae` | `SOC_Calculate_OCV_From_Voltage` | `SOC_Init_SOH_Cycles_From_Throughput` |
| 118 | `0x0800fd4c` | `SOC_Check_Max_Capacity_Valid` | `SOC_Is_Full_ByMaxCell` |
| 118 | `0x0800fd7c` | `SOC_Check_Min_Capacity_Valid` | `SOC_Is_Empty_ByMinCell` |
| 118 | `0x08006f34` | `BMS_Check_FullCharge_Condition` | `BMS_Measurements_Valid_Check` |
| 117.7 | `0x0800fcdc` | `SOC_Check_Max_Capacity_Valid` | `SOC_Is_Full_ByMaxCell` |
| 117.7 | `0x0800fd0c` | `SOC_Check_Min_Capacity_Valid` | `SOC_Is_Empty_ByMinCell` |

---

## 10. Open items

- SOH table (`0x0801B874`), discharge matrix (`0x0801BB6C`–`0x0801BB80` ff.) and
  temperature thresholds: only via SWD dump.
- Which function fills the cloud key `soh` (the format string is referenced via a table). Given the BLE finding, presumably the same empty field.
- Control pack +0x50/+0x54 (`bat_charge_power` / `bat_discharge_power`, u32) is filled by
  `CAN_Store_PerPack_GrpExt_0x50` from an additional frame that BMS 118 does **not** send. The fields
  therefore stay 0, there are no per-pack energy counters. Which BMS version sends the frame
  is open.
