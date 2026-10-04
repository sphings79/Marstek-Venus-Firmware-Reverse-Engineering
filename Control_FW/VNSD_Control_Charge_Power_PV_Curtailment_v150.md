# Venus D — Control 150: Charge Power from the BMS Limit and PV Curtailment

> 🇩🇪 Deutsche Version: [VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md](VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md)

**Status:** 2026-09-24
**Applies to:** Venus D (VNSD-0), Control/EMS 150 (`VNSD-0_app_0150_0805_115146.bin`).
Differences from D 147/149.2 are in §5; Venus A and E3 in
`Methodik_und_Meta/Device_Comparison_VNSD_VNSA_VNSE3.md` §3f.
**BMS-side counterpart:** `BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md`.

Source basis: §1–§4 were checked in this session against the Ghidra project and the raw image (literals
resolved). The version comparisons in §5 and the statement on `charge_cutoff_soc` come from the
analysis of the same day and were **not** re-checked here.

---

## 1. Key statements

1. The Control converts the BMS charge current limit into a power:
   **P_bms = 32100 × 32106** (voltage × current limit). When charging, this value limits every
   setpoint, including forced charging via 42020.
2. The Control has no pack selection by SOC of its own. The BMS does that.
3. **With positive AC power, the Control curtails the PV to the BMS charge limit**
   (`min(P_bms, 2500 W)`). In the top-of-charge taper (≈ 1270 W or ≈ 500 W, see BMS doc §8)
   the PV upper limit is correspondingly low as long as "feed in surplus" is not
   active. How much PV this costs in the field has not been measured.
4. Anyone who writes 42011 **also sets 42010 (force_mode) to 3**. Mode 3 then drives
   with fixed ±2500 W toward the total SOC 32104.

---

## 2. Conversion limit → power

### 2.1 `Power_Limit_Clamp` (0x08013234)

```c
return −( u16@0x20014F8E × 0.01   // 32100 bat_volt, 10 mV
        × u16@0x20014FA0 × 0.1 ); // 32106 resp. 35111 chrg_curr_limit, 0.1 A
```

Negative means charging (sign convention of the setpoint). The current limit is directly the value
that the BMS sends from `Master_Build_ChargeCurrentLimit` via CAN PF3 (pack table, 50 A at most,
see BMS doc §8).

### 2.2 `Battery_Charge_Power_Limiter` (0x080043FC)

Called from `Inverter_Power_Setpoint_Calc` (0x08006168, call at 0x08006176). `param` is
the setpoint (negative = charging).

```c
if (bms.flags@+0x17 & 1)                  // force_chrg from the BMS
    return max(param, P_bms);              // charging at most P_bms
if (param < 0 && any MPPT ≥ 10.1 W) {      // 0x20014F58/5E/64/6A = 30037–30040, 0.1 W
    pv    = sum MPPT1–4;
    bound = pv + max(P_bms, −2500);
    if (bound < 0) param = max(param, bound);   // grid share = |P_bms| − PV
    else           param = 0;                   // PV covers the limit: no grid import
}
return param;
```

Independently of this, `Inverter_Power_Setpoint_Calc` afterwards clamps the result once more
from below: `if (setpoint < P_bms) setpoint = P_bms`. This limit always applies, with and without PV.

Effectively the battery charges at `min(PV + grid share, |P_bms|)`. This also holds for forced charging
via 42020, because both limits sit behind every setpoint source.

---

## 3. PV upper limit — `Grid_Export_Power_Limiter` (0x0801F200)

Runs every 4 ticks, independent of the operating mode. Computes the PV upper limit and sends it via
`Inverter_CAN_PowerCmd_Send` to the inverter (§3.2).

### 3.1 Calculation

```c
lim  = min(−P_bms, 2500);
cap  = −P_bms;                             // again, if BMS +0x17 bit 1
if (!(bms.flags@+0x16 & 1)) cap = 0;       // BMS does not release charging
ac   = s16@0x20014EB4;                     // 30006, AC power, > 0 = feed-in

if (surplus_flag@0x20000285)               // "feed in surplus" active
    out = u16@0x2000012F;                  // 44003
else if (ac ≥ 1)                           // feed-in
    out = lim;                             // ← PV curtailed to the charge limit
else
    out = min(cap, ac + lim + 300);
out = min(out, 4000);
```

### 3.2 Transmission

`Inverter_CAN_PowerCmd_Send` (0x08005B54, until 2026-09-24 `Inverter_RS485_Command_Send`) builds
the ID with `CAN_Build_Arbitration_ID(2, 1, 2, 0/1)` and writes 4 + 8 bytes. This is a
**CAN** transmission to target class 2, not RS485.

| State byte `0x200002B4` (Modbus 40000/45000) | sent |
|---|---|
| 0 | `(1, 0, out, …)` |
| 1 | `(1, 1, max(0, −30001 − P_bms), …)`, only if `0x2000014C` (37001) == 5, otherwise `(1, 1, 0, …)` |
| 3 | `(1, 1, min(out, u16@0x200002BE), …)` |
| other | nothing |

The short version from 24.09. said "every mode". Strictly speaking, states 0 and 3 send
the PV upper limit, 1 sends a different value, and 2 sends nothing. What the states
mean functionally has not been investigated here.

The receiver is, as far as currently known, a separate MPPT controller with no image available
(MPPT version 104 according to the firmware string `V150.116.104.118`). Micro 116 contains no
PV curtailment; it only translates inverter power and BMS current/voltage limits.

### 3.3 "Feed in surplus" — `Surplus_FeedIn_Enable_Check` (0x080052B0)

Until 2026-09-24 the function was called `Battery_Forced_Charge_Check`; it has nothing to do with forced charging.
Every 300 ticks the following are checked:

- `full_en` (`0x20014D00 + 0x75`, EEPROM `0x375`) set,
- grid enable (`0x20014E9C + 0x16`) ≠ 0,
- at least one PV string ≥ 14.1 V,
- SOC ≥ 95 %.

If one condition is missing, both flags are cleared. At **SOC ≥ 100 %** and cleared
`chrg_flag` (BMS `+0x16` bit 0), `0x20000285` is set and stays set until the SOC falls below 95 %.
`0x20000286` is added as soon as the AC power reaches 44003 − 200 W.

---

## 4. `charge_to_soc` (42011) and force_mode

### 4.1 Write path

`Write_Handler` (0x08051D14), branch `0xA41B`:

```c
if (13 ≤ value ≤ 100) {
    *(u8*)0x2000034A = value;  // 42011, RAM only
    *(u8*)0x20000349 = 3;      // 42010 force_mode ← side effect
}
```

The line `puVar3[1] = 3` points via `puVar3 = 0x20000348` to `0x20000349`. Every valid
write access to 42011 therefore starts mode 3, even if 42010 was 0 beforehand. This matches
the field finding in `Modbus_RS485_TCP/VNSD_Register_Persistence_RAM_vs_EEPROM_v150.md`
("Setting it to 70 immediately triggers a discharge").

### 4.2 Mode 3 — `Remote_Power_Command_Execute` (0x0801EBA4)

```c
case 3:
    soc = u16@0x20014F94 / 10;             // 32104, TOTAL SOC
    if      (charge_to_soc < soc) Setpoint(1, +2500);  // discharge
    else if (charge_to_soc > soc) Setpoint(1, −2500);  // charge
    else    force_mode = 0;                           // target reached → off
```

- The comparison basis is the total SOC, not the SOC of the active pack.
- The power is fixed at 2500 W. 42020/42021 have no effect here; the only limit is §2.2.
- On equality the Control resets force_mode to 0 by itself.

`charge_cutoff_soc` (EEPROM `0x200`) has no effect in D 150 (taken over from the analysis of 24.09.).

---

## 5. Version differences (from the analysis of 24.09., not re-checked)

PV upper limit during feed-in (`ac ≥ 1`):

| Version | Upper limit | Note |
|---|---|---|
| D 147 | `ac + P_bms`, without the 2500 W limit | two-stage handling with a full battery |
| D 149.2 | `min(P_bms, 2500)` | |
| D 150 | `min(P_bms, 2500)` | |

The curtailment to the charge limit therefore came with 149.2. Under 147 the PV was allowed, during feed-in, to exceed
the charge limit by the amount of the feed-in.

---

## 6. Ghidra renamings (2026-09-24, program D 150 only)

| Address | before | after |
|---|---|---|
| `0x080052b0` | `Battery_Forced_Charge_Check` | `Surplus_FeedIn_Enable_Check` |
| `0x08005b54` | `Inverter_RS485_Command_Send` | `Inverter_CAN_PowerCmd_Send` |

In 149.2, 147 and the A/E3 programs the same functions still carry the old names. Older
documents (`VNSD_Control_FW_Function_Tracking_new.md`, `Modbus_RS485_TCP/VNSD_Command_Register_45000_Block_v150.md`,
device comparison §3e) also use the old names. Decompilation export
`Dekompilate/VNSD_Control_v150/` (local, not in the repo) has been regenerated.

---

## 7. Open items

- Meaning of the states of `0x200002B4` in the limiter and of the value `0x200002BE` (state 3).
- Modbus address of the current limit on the E3 (see device comparison §3f).
- Field measurement for §1.3 is missing: log MPPT power (30037–30040) against AC power and 32106 above
  90 % SOC with PV surplus. Only then is it clear whether and how much PV is lost.
  "Feed in surplus" (`full_en`) only kicks in from 100 % SOC according to the code.
