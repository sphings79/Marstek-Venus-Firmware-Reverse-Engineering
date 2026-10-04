# Venus D — Battery Charge Limits (Charge Current / Charge Power)

> 🇩🇪 Deutsche Version: [VNSD_Ladegrenzen_Ladeleistung_de.md](VNSD_Ladegrenzen_Ladeleistung_de.md)

**Status:** 2026-09-30.
**Firmware versions:** BMS v118, Control (EMS) v151, Micro/VNS v116.
**Sources:** Ghidra programs with correct base address (BMS `0x08002800`, Control `0x08004800`, Micro `0x08004000`)
and the exports in `Dekompilate/VNSD_BMS_118/` (local only, not in the repo), `Dekompilate/VNSD_Control_v151/`, `Dekompilate/VNSD_Micro_116/`.
All addresses in this document are true flash addresses of these programs.

## Question

The Venus D has 4 PV inputs @ 1000 W each and can charge via AC at maximum 2500 W. To what charge current resp. charge power is battery charging limited in the firmware? Can the batteries be charged with 4000 W?

## Short Answer

**No, 4000 W charge power is not possible with this firmware.** There are two independent limits:

1. **BMS: maximum 50.0 A charge current, and always only one pack charges.** This yields at most roughly 2.7 kW
   (50 A × ~54 V), mathematically 2.88 kW at charge termination 57.6 V. This limit applies regardless of
   power source (AC or PV).
2. **Control and Micro/VNS: maximum 2500 W.** Both firmwares hard-cap charge power to 2500 W (`0x9C4`).

Below 90% SoC (at 20–40 °C cell temperature) the 2500 W limit from Control/Micro applies; above 90% SoC the BMS current table applies.

## 1. BMS (v118)

### 1.1 Charge Current per Pack: Temperature × SoC Table, 50 A Ceiling

Function `FUN_08012660` (@`0x08012660`, named in Ghidra program) computes pack allowed charge current:

- SoC = 100% → 0 A.
- Otherwise: temperature stage from `g_tempAxisTable` (@`0x0801B97E`, 18 support points −25 … 60 °C) via
  `BMS_TempZoneIndexLookup` (@`0x08012B90`, with hysteresis), SoC stage from SoC axis @`0x0801B9A2`
  (0, 5, 10, 20, 30, 40, 50, 60, 70, 80, 90, 95, 100 %).
- Value from `g_chargeCurrentDerateMatrix` (@`0x0801B9BC`, 18 × 12 × uint16) = percent of 50 A;
  result in 0.1 A: `percent × 5` → 100% = 500 = **50.0 A**. Values over 100 also yield 500.

`Master_Build_ChargeCurrentLimit` (@`0x0800BBFC`) takes the active pack value (during pack switch
the minimum of both involved packs, **never a sum**) and limits it again to **500 = 50.0 A**.
The same function sets charge termination voltage fixed at `0x240` = 576 → **57.6 V**. Both are sent per CAN
(`CAN_TX_PF3_ChargeDischargeLimits` @`0x08007CD8`; the voltage is additionally capped at 60.9 V max there).

The table from image (value = % of 50 A; row = temperature range from … to):

| Cell temperature | 0–5% | 5–10% | 10–20% | 20–90% | 90–95% | 95–100% |
|---|---|---|---|---|---|---|
| to −20 °C | 0 | 0 | 0 | 0 | 0 | 0 |
| −20 … −15 °C | 5 | 5 | 5 | 5 | 5 | 5 |
| −15 … −5 °C | 10 | 10 | 10 | 10 | 10 | 10 |
| −5 … 0 °C | 15 | 15 | 15 | 15 | 10 | 10 |
| 0 … 5 °C | 25 | 25 | 30 | 30 | 10 | 10 |
| 5 … 10 °C | 30 | 30 | 40 | 40 | 15 | 15 |
| 10 … 15 °C | 50 | 50 | 50 | 50 | 15 | 15 |
| 15 … 20 °C | 50 | 50 | 80 | 80 | 40 | 15 |
| 20 … 40 °C | 100 | 100 | 100 | 100 | 50 | 20 |
| 40 … 45 °C | 80 | 80 | 80 | 80 | 50 | 20 |
| 45 … 50 °C | 40 | 60 | 60 | 60 | 40 | 20 |
| 50 … 55 °C | 20 | 40 | 40 | 40 | 40 | 20 |
| 55 … 60 °C | 20 | 20 | 20 | 20 | 20 | 20 |
| over 60 °C | 0 | 0 | 0 | 0 | 0 | 0 |

(Columns 20–30% through 80–90% are identical in all rows and combined here. Row labels follow the indexing in `BMS_TempZoneIndexLookup`: stage *i* = temperature greater than support point *i* through including support point *i+1*.)

In typical range 20–40 °C this means: **50 A up to 90% SoC, 25 A from 90 to 95%, 10 A from 95 to 100%.**

### 1.2 Always Only One Pack Charges — Enforced in Code

- `PackHandoff_StateMachine` (@`0x08006740`) sets the charge MOSFET mask `DAT_20002a17` to 0 or exactly
  one bit (`1 << pack_idx`), never multiple.
- `OwnPack_MOSFET_Driver` (@`0x08006BC4`) switches a pack's charge MOSFET in only if its bit is set in this
  mask.

Thus physically only one pack can have its charge MOSFET on at a time. The active pack's charge limit is
simultaneously the system's charge limit — regardless of how many packs are installed.

### 1.3 Pack Voltage

Charge termination 57.6 V (`Master_Build_ChargeCurrentLimit`) and discharge termination 48.0 V (`0x1E0`,
`Master_Build_DischargeCurrentLimit` @`0x0800BCC4`) fit 16 LFP cells in series (3.60 V resp. 3.00 V per cell,
51.2 V nominal). No cell count variable for this; derived from the constants.

## 2. Control / EMS (v151)

- `g_maxChargePowerW` (EEPROM `0x202`): preset and simultaneously ceiling at **2500 W**
  (`Config_ClampDefaults_ParamBlock` @`0x0800935C`: value 0 or > `0x9C4` → `0x9C4`).
- App/BLE command `0x16` accepts only 300–2500 W (`BLE_App_Command_Dispatcher` @`0x0800C730`).
- Forced charging and remote control/Modbus charge commands likewise capped at ±2500 W
  (`Battery_Charge_Power_Limiter` @`0x08008BFC`, `Remote_Power_Command_Execute` @`0x08023538`).
- Additionally a dynamic limit from pack voltage × BMS charge current limit (`g_pgn1801InfoBatVolt` ×
  `g_pgn1803InfoChrgCurr`, `Power_ScaledValue_Calc_Negated` @`0x08017AD8`), used in `Battery_Charge_Power_Limiter`
  and `Inverter_Power_Setpoint_Calc` @`0x0800A968`. Per-pack values (`g_bmsPack0..7MaxChargeCurrent`) read only
  for telemetry, **never summed**.
- Result: charge setpoint = min(requested power, BMS limit, `g_maxChargePowerW` ≤ 2500 W).

**PV/MPPT:** The 4 PV channels come from separate MPPT unit via dedicated ASCII protocol over
Ethernet/UDP (`MPPT_ParseATResponse` @`0x08033D28`, `AT_Copy_MPPT_Channel_Data` @`0x080089A0`). Only
this receive path is found in Control code; no command back to MPPT unit on this path. MPPT firmware
itself not provided.

## 3. Micro / VNS (v116)

- Charge and discharge power limited only in watts: `g_powerLimitChargeW` / `g_powerLimitDischargeW`
  (EEPROM `0x202`/`0x204`, Modbus 44002/44003), CAN commands `0x03`/`0x04` in `EMS_Inverter_CAN_Dispatcher`
  reject values ≥ 2501 → **hard limit 2500 W**. Likewise real-time limiting (±2500 W) and other
  config/ramp values (since v116 consistently 2500 W, in v115 partly 2200 W).
- Micro receives BMS charge voltage/current limit via CAN (`BMS_CAN_Parser` → `g_bmsChargeVoltLimit`, carries
  voltage and charge current limit).
- Micro **has no MPPT/PV code**.

## 4. Alignment with Measured Values

Measurements without Omnibattery influence (7 packs, AC limit 2500 W): < 90% SoC ≈ 2310 W; 90 → 95% ≈ 1260–1290 W;
95 → 100% ≈ 480–490 W; between packs 15–30 s at 0 W.

- 90–95%: 25 A × ~53–54 V ≈ 1.3–1.35 kW on battery side → matches ~1270 W.
- 95–100%: 10 A × ~54 V ≈ 0.54 kW → matches ~485 W.
- < 90%: 50 A × ~53 V would be ~2.65 kW; 2500-W limit from Control/Micro applies first.

The step pattern thus comes exactly from the BMS table (columns 90–95% and 95–100%, row 20–40 °C).
Measured values slightly below calculated; how much is inverter loss and how much actual
pack voltage cannot be determined from firmware alone.

## 5. Open Points

- Whether PV physically goes through Micro's DC/DC converter (LLC) into battery or hangs directly at battery,
  cannot be fully determined without MPPT firmware. For the answer: 50-A limit of BMS applies either way.
- Not checked: whether BMS actively cuts off at > 50 A (overcurrent threshold in KA495xx AFE) or only
  reports 50 A as limit to Control and Micro.
- `Grid_Export_Power_Limiter` (@`0x08023B94`) in Control has hard cap of **4000 W**. Affects export/feed-in
  direction and a CAN command with "mppt" fields to node 1, not charge power. What this command exactly
  controls cannot be determined from Control alone.
