# Venus D — BMS 118: Pack Rotation, Lower Limit and Charge Taper

> 🇩🇪 Deutsche Version: [VNSD_BMS_Pack_Rotation_Taper_v118_de.md](VNSD_BMS_Pack_Rotation_Taper_v118_de.md)

> **Note (2026-09-30):** Flash addresses in this document come from the old analysis with a wrong base address (0x2800 too low). Charge limits and current table with correct addresses: [VNSD_Charge_Limits_Charge_Power.md](VNSD_Charge_Limits_Charge_Power.md), full analysis: [VNSD_BMS_FW_v118_Full_Analysis.md](VNSD_BMS_FW_v118_Full_Analysis.md).

**Status:** 2026-09-24
**Applies to:** Venus D (VNSD-0), BMS firmware 118 (`BMS_118_20260119100535e43806957.bin`).
According to a byte search, the Venus A BMS has the same 10 % target function, but the discharge and
lower-limit logic only exists here — details in `Methodik_und_Meta/Device_Comparison_VNSD_VNSA_VNSE3.md` §3f.
**Sources:** Ghidra project `/VNSD/BMS/BMS_118_…`, decompilation export `Dekompilate/VNSD_BMS_118/` (local, not in the repo),
field data from Home Assistant (carport device, 7 packs of 2.56 kWh each) from 19–22 Sep 2026.

Predecessor of this analysis: `VNSD_BMS_FW_Analysis_v118.md` §11 (limit aggregation, handoff). What was
open there is clarified or corrected here (§10).

---

## 1. Key findings

1. **Exactly one pack charges and discharges at any time.** The selection is made by the master BMS, not
   by the Control and not by the inverter.
2. **Charging:** The target is the next ten-percent step above the current SOC (57 % → 60 %, 60 % → 70 %).
   Once it is reached, the pack with the **lowest** SOC takes over. On a tie, the
   lower index wins.
3. **Discharging:** The threshold is 10 percentage points below the SOC **at the time of the last
   recalculation**. It is not a ten-percent step and never smaller than `100 − DoD` (default 10 %).
   When the pack falls below it, the **fullest** pack takes over.
4. **Target and threshold go stale.** They are recalculated only on a pack change or when the
   active pack reaches its target and still remains the most extreme one. If a charge ends before
   a switch has happened, the old values remain valid. This is exactly what explains the discharge
   from 28.7 to 12.0 % without an intermediate stop on 20 Sep (§5).
5. **Taper is constant current per SOC zone,** not constant power. Measured on 22 Sep:
   90 → 95 % ≈ 23.2 A, 95 → 100 % ≈ 9.1 A. The table itself is not in the image (§8).
6. Below 11 % SOC the logic switches to dedicated modes. If no pack is fuller than the
   active one at that point, the BMS goes into pack mode **Shutdown** (§7).

---

## 2. Time base

The master logic runs in its own FreeRTOS task (code from `0x0800C24C`, not recognized by Ghidra
as a function, determined by raw decoding of the BL instructions):

```
loop:
    FUN_0800932c();
    FUN_080087ac();
    BMS_Role_Dispatcher();        // → Master_MultiPack_Tick → pack selection
    vTaskDelay(100);
```

SysTick is loaded in `System_Init_Main` with `SystemCoreClock / 1000 − 1`, i.e. 1 kHz.
`LIB_vTaskDelayWithAssert` is an unmodified `vTaskDelay` (ticks, no ms conversion).

**One tick of the pack logic ≈ 100 ms** (plus the runtime of the three calls, so rather a bit more).
All counter values below are converted into this unit.

---

## 3. Data basis

### 3.1 Pack array

Base `0x200041B6`, stride `0x60`, index 0 = own pack. The field names come from
`BMS_Debug_PerPack_Detail_Print` (plain-text output in the image):

| Offset | Field | Unit |
|---|---|---|
| +0x00 | Addr | — |
| +0x01 | Online | 0/1 |
| +0x0E | Bat Soc | 0.1 % |
| +0x10 | Bit 5 = Chg MOS, Bit 4 = Dsg MOS | — |
| +0x48 | Min Cell | mV |
| +0x4C | Bat Curr | 0.1 A, i16, **charging positive** |
| +0x50 / +0x52 | max. charge / discharge current | 0.1 A |
| +0x56 / +0x58 | Protect1 / Protect2 | bitmask |

### 3.2 SRAM variables

| Address | Meaning |
|---|---|
| `0x20002A0C` | Master run state (0 → 1 → 2) |
| `0x20002A11` | active pack |
| `0x20002A12` | target pack of the handoff |
| `0x20002A17` | **target mask Chg-MOS** (which packs may charge) |
| `0x20002A18` | **target mask Dsg-MOS** (which packs may discharge) |
| `0x20002A14` | exclusion mask: offline or Protect1 `0x1F0` / Protect2 `0x9000` |
| `0x20002A1E` | recovery charge active (§7.4), `+1` mask, `+2` min cell voltage |
| `0x20002A22` | **charge target** in % (ten-percent step) |
| `0x20002A23` | **discharge threshold** in % |
| `0x20002A24` | selection mode 0–3 (§7) |
| `0x20002A44` | handoff requested |
| `0x20002A54` / `0x20002A56` | counter charge / discharge changes |
| `0x20002A58` | copy of the DoD, to detect changes |
| `0x200028CA` | pack mode (0 Idle, 1 Transitional, 2 Active, 3 Shutdown, 4 Error) |
| `0x200028DB` | number of further packs (packs − 1) |
| `0x200028DC` | online mask |
| `0x200028E0` | **DoD** in %, default `0x5A` = 90 |

`0x200028E0` is the depth of discharge. It is not an "SOC balance parameter", as stated in
`VNSD_BMS_FW_Analysis_v118.md` §4.8/§11.5. Evidence:

- `CAN_Load_Or_Init_Config_Value` reads the value from flash, default 90.
- `CAN_TX_PF2_Capacity` sends it to the Control.
- The Control outputs it as `pgn_1802_info.dischrg_dod`.

It is set via CAN (`0x100201AA`, and also byte 13 in the frame `0x1801FFAA`) and RS485.

The Control's backup reserve (EEPROM `0x201`, app setting 30–88 %, default
reserve 12 %) is a **different** parameter. See
`Modbus_RS485_TCP/VNSD_Register_Persistence_RAM_vs_EEPROM_v150.md`, addendum 2026-08-22.

---

## 4. Call chain

```
Master_MultiPack_Tick
 └─ Master_RunState_Dispatcher                      (0x20002A0C)
     0: FUN_080040e8                                  → 1
     1: Master_RunState_InitSelect                    → 2
          ├─ Protect1 bit 0 pack present? → request recovery (§7.4)
          └─ otherwise PackSelect_Startup_Closest50   (§7.5)
     2: 0x20002A1E == 0 ?
          ├─ PackHandoff_Tick
          │    ├─ PackSelect_ModeDispatch             (mode 0–3, §5–§7)
          │    ├─ PackHandoff_StateMachine            (§6)
          │    ├─ PackSelect_MosMismatch_Fallback     (§7.6)
          │    └─ PackSelect_ChargeEnable_Maintain    (§7.6)
          ├─ PackSelect_Tick                          (fault path, §7.6)
          └─ PackRecovery_Request_Check
        otherwise PackRecovery_LowCell_Charge_SM      (§7.4)
```

---

## 5. Mode 0 — normal rotation

In mode 0, `PackSelect_ModeDispatch` calls both functions in every tick, the one for charging
and the one for discharging. The sign of the pack current decides which one takes effect.

### 5.1 Charging — `PackSelect_Charge_StepRotate` (0x08004CD0)

```c
if (soc% < ziel || strom < 1.1 A)          // target not reached or not charging
    zaehler = 0;
else if (++zaehler > 50 || soc > 99.9 %) {  // > 5 s at the target with charge current
    kandidat = pack with the lowest SOC (online, strictly smaller → lower index wins);
    if (kandidat == aktiv) {                 // stays the emptiest
        ziel     = PackSelect_ChargeStepTarget(aktiv);
        schwelle = PackSelect_DischargeStepThreshold(aktiv);
    } else
        handoff to kandidat;
}
```

`PackSelect_ChargeStepTarget` (0x08006CFC): `((soc% + 10) / 10) × 10`, at most 100.
`soc%` is the integer percentage value here (57.6 % → 57).

**Correction to the short version of 24 Sep:** The counter only runs while the pack charges with
**at least** 1.1 A, not with less than 1.1 A. If the charge breaks off at the target within
the 5 s, no switch happens.

Nothing special happens in the selection at 90 %. The steps at 90 and 95 % seen in the field
come from the current table (§8).

### 5.2 Discharging — `PackSelect_Discharge_StepRotate` (0x08003BDC)

```c
if (dod changed) counter runs immediately;
if ((soc% > schwelle || strom > −1.1 A) && dod unchanged)
    zaehler = 0;
else if (++zaehler > 50) {                  // > 5 s at/below the threshold with discharge current
    kandidat = pack with the highest SOC;
    if (kandidat == aktiv) {
        ziel     = PackSelect_ChargeStepTarget(aktiv);
        schwelle = PackSelect_DischargeStepThreshold(aktiv);
    } else
        handoff to kandidat;
}
```

`PackSelect_DischargeStepThreshold` (0x08006BBC), with `L = 100 − DoD`:

| Situation | Result |
|---|---|
| `soc% − 10 ≥ L` | `soc% − 10` |
| `soc% − 10 < L < soc%` | `L` |
| `soc% ≤ L` | `L − 1` |

A switch happens as soon as `soc% ≤ threshold`, i.e. with a threshold of 20 from 20.9 %.

### 5.3 Stale target and stale threshold — field evidence

Both values are reset **only** in three places: at the end of a handoff (§6, step 5),
in the startup path (§7.5) and in the two `kandidat == aktiv` branches above. The HA history
(Omnibattery SOC per pack) shows exactly this behavior:

| Time | Observation | Explanation from the code |
|---|---|---|
| 19 Sep 02:00–02:21 | P1 discharges 30.6 → **20.9 %**, then another pack takes over | P1 activated at ~30 % → threshold 20 → switch at `soc% ≤ 20` |
| 19 Sep 13:53–14:02 | P1 charges 19.6 → 30.1 %, then the charge ends | Target 20; at 20.x P1 is still the emptiest → target 30, **threshold = Clamp(20) = 10**. At 30.1 % the current breaks off before 5 s are up → **no switch**, P1 stays active |
| 20 Sep 23:05–23:42 | P1 discharges **28.7 → 12.0 %** without a stop, P3/P4/P6 stand at 28.9 % | Threshold is still 10 → switch only at `soc% ≤ 10`. At 12.0 % Omnibattery switched off |
| 21 Sep 07:37–08:32 | P1 charges 12.0 → **30.1 %**, although P2 at 12.0 % is equally empty and P5/P7 at ~18 % are emptier | Target is still **30** from 19 Sep. Only at 30.1 % does it switch, to P2 (12.1 %) |
| 21 Sep 08:32–08:50 | P2 12 → 20, P7 17.7 → 20, P5 18.6 → 20, then P2 20.7 → 30 | regular rotation, tie at 20.1 % → lower index (P2) |

The forced discharge via RS485 on 20 Sep plays no role here. The BMS does not know the Control's
mode, it only sees current and SOC.

**What would have happened without Omnibattery:** P1 would have run down to 10.9 %, then the fullest
pack (P3, 28.9 %) would have taken over. Below 11 %, P1 would never again have taken over via mode 0 (§7).

---

## 6. Handoff — `PackHandoff_StateMachine` (0x08003F40)

The jump table has meanwhile been resolved, all steps are evidenced:

| Step | Action | Proceeds when | Duration |
|---|---|---|---|
| 0 | Enable minimum formation of the limits (`0x20002A0E = 0`) | 100 ticks | ≈ 10 s |
| 1 | Clear Chg-MOS target (`0x20002A17 = 0`) | no pack charges any more | MOS switching time |
| 2 | Dsg-MOS target `|= new` — **old and new pack in parallel on the discharge side** | Dsg-MOS of the new pack on | |
| 3 | Dsg-MOS target `= new` | exactly one Dsg-MOS on, and it is the new one | |
| 4 | Chg-MOS target `= new` | new pack charges or SOC > 98 % | |
| 5 | active = new; recalculate mode, charge target and discharge threshold | immediately | |

**Correction to `VNSD_BMS_FW_Analysis_v118.md` §11.4:** `0x20002A18` is the target mask of the
**discharge** MOSFETs, not a general enable mask. The make-before-break only affects the
discharge side; during the handoff, charging never happens across two packs.

This matches the measured 15–30 s at 0 W between two packs: 10 s wait time plus
MOSFET switching operations plus re-regulation of the inverter.

---

## 7. Special cases

### 7.1 Mode selection — `PackSelect_Mode_BySoc` (0x08006B34)

Earlier Ghidra name `OTA_Validate_FW_Version`, wrong. The function is called only when
a pack is activated (handoff step 5, startup path, recovery):

| SOC of the new pack | Mode |
|---|---|
| `soc% < 9` and `0x20002A75 == 0` | 2 (SOC is remembered as the start value) |
| `soc% < 9` and `0x20002A75 != 0` | 3 |
| `9 ≤ soc% < 11` | 2 |
| `soc% ≥ 100` | 1 |
| otherwise | 0 |

`0x20002A75` is set only by `CAN_CMD_04_Handler` (subcommand 4, ID `0x100401AA`). Who this
command comes from is open (§10).

### 7.2 Mode 2 — low SOC (`PackSelect_Mode2_LowSoc`, 0x08003AF4)

- If the pack is charging (flag `0x20002A1D` bit 0 set and ≥ 0.5 A): charge target 9 or 10 %,
  discharge threshold 20 %, then normal rotation (§5). The pack is thus swapped immediately against the
  emptiest one.
- Otherwise: after 600 ticks (≈ 60 s) **or** 1.0 % SOC drop since activation it switches to the
  fullest pack, but only if that one stands above 10.0 %.

### 7.3 Mode 3 — empty (`PackSelect_Mode3_Empty`, 0x08003844)

- Discharging (< −1.0 A) for 100 ticks (≈ 10 s), **or** idle (|I| < 1.0 A) for 1200 ticks
  (≈ 120 s) → switch to the fullest pack.
- **If the active pack itself is the fullest, the function sets pack mode
  `0x200028CA = 3` (Shutdown).** This is the behavior of the device at the very bottom.
- Charging (≥ 1.0 A) resets the counters and, if `0x20002A6D == 0`, sets the mode to 0.

### 7.4 Recovery charge — `PackRecovery_LowCell_Charge_SM` (0x0800DF0C)

Earlier Ghidra name `BMS_OTA_Upgrade_StateMachine`, wrong. The function has nothing to do with OTA.
It runs instead of the normal rotation as soon as a pack reports Protect1 bit 0
(`Pack_Get_Protect1Bit0_Mask`, set via `PackRecovery_Request_Check`):

1. All Dsg-MOS off.
2. From the affected packs, choose the one with the **lowest min cell voltage** and
   charge it exclusively (Chg-MOS on, Dsg-MOS off).
3. From 30 ticks with ≥ 1.1 A, additionally enable the Dsg-MOS.
4. Done when `Min Cell > 3100 mV` **and** SOC > 14 % (in step 4 > 13 %) **and** bit 0
   is cleared. Then the next affected pack follows.
5. When all are through: activate the pack **closest to 50 %**, recalculate mode/target/threshold,
   back to normal rotation.
6. 300 s without charge current (< 1.1 A) or 60 s without switching → pack mode 3 (Shutdown).

**Protect1 bit 0 is cell undervoltage.** `Cell_OVP_UVP_TempCompensated_Protection` sets it
at `Min Cell < 2500 mV` (cold 2300 mV) and thereby forces the pack's SOC to 0; bit 1 is
overvoltage and forces the SOC to 100. Details in `VNSD_BMS_FW_Analysis_v118.md` §4.6. In
`VNSD_BMS_FW_Analysis_v117.7.md` §5.2, bit 0 and bit 1 were swapped; this is corrected there.

The same function also exists byte-identical in 117.7 (`0x0800df0c`), as does `PackSelect_Mode_BySoc`
(`0x08006acc`).

### 7.5 Startup path — `PackSelect_Startup_Closest50` (0x08004148)

This resolves the open point `FUN_08004148` from §11.7:

1. wait until no pack is charging any more;
2. among the online packs with SOC > 20 %, choose the one with the smallest distance to 50 %, otherwise
   pack 0; enable only its Dsg-MOS;
3. as soon as it discharges: at SOC > 98 % done, **without** charge enable (an almost full pack should
   not be charged). Otherwise enable the Chg-MOS and wait until it charges;
4. timeout 600 ticks (≈ 60 s) → pack mode 4 (Error).

The > 98 % check is therefore not a calibration path, but prevents the charge enable for a
full pack at startup.

### 7.6 Fault and fallback paths

| Function | Trigger | Effect |
|---|---|---|
| `PackSelect_ChargeEnable_Maintain` (0x0800352C) | active pack full (`0x20002A6F` or 100.0 %) | clear Chg-MOS target; on target/actual deviation > 600 ticks → pack mode 4 |
| `PackSelect_MosMismatch_Fallback` (0x08003708) | target/actual mask of the MOS deviates > 300 ticks | another pack with `soc% > L` as target; after 3 attempts pack mode 4 |
| `PackFault_Build_ExcludeMask` (0x08004924) | offline or Protect1 `0x1F0` / Protect2 `0x9000` | exclusion mask `0x20002A14` |
| `PackSelect_Next_AboveSocThreshold` (0x080049D8) | exclusion mask ≠ 0 | last fault-free pack with `soc% > L` becomes the target |
| `PackSelect_FaultPath_AllOff` (0x080034AC) | Protect1 `0xE0C` / Protect2 `0x2200` | all MOS target masks 0; if the fault is gone → pack mode 4 |
| `Master_Check_AllPacksOnline` (0x0800364C) | online count ≠ pack count | `0x20002A0F = 1` (feeds into Protect2 bit 15) |

---

### 7.7 Own MOSFETs, precharge and startup

**Startup — `Master_Startup_WaitForEnable` (0x080040E8, master run state 0):**

- If a pack is still charging, only the Chg-MOS target mask is cleared.
- Otherwise the master enables the **discharge MOSFETs of all packs** (`0x20002A18` = all bits) and
  waits until `0x20002A72` = 1 and no error lock (`0x20002A71`) exists.
- `0x20002A72` is set only by the CAN frame `0x1801FFAA` with byte 11 = 1 (`CAN_RX_Handler`). The
  master therefore starts the pack selection only after an enable from outside. Until then all
  packs hang in parallel on the discharge side.

**Driver — `OwnPack_MOSFET_Driver` (0x080043C4), runs in every master tick:**

| `HAL_GPIO_Control_Switch` | Pin | Function |
|---|---|---|
| 1 / 2 | GPIOC pin 5 on / off | **charge MOSFET** (pack struct +0x10 bit 5) |
| 3 / 4 | GPIOB pin 0 on / off | **discharge MOSFET** (+0x10 bit 4) |
| 5 / 6 | GPIOB pin 2 on / off | **precharge switch** |

- The own bit is `1 << (address − 1)`.
- Charge MOSFET on: own bit in `0x20002A17` or override `0x20002A1A` = 1, and no
  charge-blocking protection bit.
- Discharge MOSFET on: own bit in `0x20002A18` or override `0x20002A19` = 1, and no
  discharge-blocking protection bit.
- If the discharge MOSFET is still off, precharge runs first.
- With blocking protection bits: all three off.
- The overrides are set by `CAN_Periodic_Alarm_Broadcast` in its startup phase: first ten times at a
  500 ms interval `CAN_PackCmd_Build_Type6`, then 300 times at a 1 s interval.

**Precharge — `OwnPack_Precharge_SM` (0x080046B8):**

1. Precharge switch on; measure `ADC_Read_Precharge_Voltage` (ADC channel 1, divider 2.1 × 3.3 V, range
   0–69 V).
2. Done when the measured value is ≤ 10 V **or** measured value + pack voltage > 44 V. Then
   precharge switch off, discharge MOSFET on. If the measured value does not decrease (< 1 V per pass,
   three times) or it takes longer than 1 s: precharge switch off, 1 s pause, new attempt.
3. After 2 retries: **set C1 bit 2 → Protect1 bit 9** (listed in 117.7 §5.2 as
   "short circuit HW"). A pack that cannot precharge the bus thus reports the same
   fault as a short circuit.
4. If GPIOC pin 1 is high as an input, the sequence aborts immediately and switches the
   discharge MOSFET on directly.

What the ADC channel physically measures (presumably the voltage across the open
discharge path) is not evidenced.

## 8. Charge taper

### 8.1 Code

`Max_Charge_Current_Calc` (0x0800FE60) takes, per pack, SOC zone × temperature row from three
tables: temperature index `0x0801B97E`, SOC zone thresholds `0x0801B9A2`, current matrix
`0x0801B9BC`. Result = entry × 0.5 A (entry in % of 50 A). At SOC = 100 % → 0 A.

**All three tables lie beyond the image end `0x08019FFF`** and cannot be read from the OTA image.
`Master_Build_ChargeCurrentLimit` (0x080093FC) takes the active pack from these
(during the handoff the minimum of both), at most 50 A. The end-of-charge voltage is fixed at 57.6 V.
No cell voltage enters anywhere.

### 8.2 Field — 22 Sep 2026, P1, max. cell temperature 33–34 °C

HA sensors `batterieleistung_bms` (= 32100 × 32101) and `bms_batteriespannung` (32100):

| SOC range | Power on the BMS side | Voltage | Current | Duration |
|---|---|---|---|---|
| < 90 % | ~2310 W at the battery (AC limit 2500 W) | — | < 50 A, **not** BMS-limited | — |
| 90.2 → 95.1 % | 1273–1301 W | 54.8–55.3 V | **≈ 23.2 A constant** | 6 min 26 s |
| 95.1 → 97.2 % (→ 100) | 497–511 W | 54.6–56.8 V | **≈ 9.1 A constant** | 7 min 43 s |

Power rises with voltage, current stays the same. The steps are therefore
constant-current zones of the table. At 97.2 % the SOC jumps to 100 % within 30 s
(full detection at 56.8 V), then a 15–30 s pause and the next pack starts with ~1280 W.

Back-calculation to matrix entries (0.5 A grid): **46 or 47** (23.0 / 23.5 A) and
**18 or 19** (9.0 / 9.5 A). Which one cannot be decided from the actual current, because the
inverter regulates just below the limit. This is a measurement, not a table value.

**Reliable way without a flash dump:** log register **32106** (`bms_charge_current_limit`,
0.1 A, SRAM `0x20014FA0`). It is exactly the result of
`Master_Build_ChargeCurrentLimit` and delivers the table value per SOC zone and temperature.
In the HA integration it is currently **not** a sensor.

### 8.3 Correction

In `VNSD_BMS_FW_Analysis_v118.md` §11, until 2026-09-24 there was a plateau of 142–150 W above 90 %
with a back-calculation to a matrix entry of 5. **This plateau was an effect of the
Omnibattery integration, not of the BMS firmware.** The passage has been removed.

---

## 9. Ghidra renamings (2026-09-24)

| Address | before | after |
|---|---|---|
| `0x08006b34` | `OTA_Validate_FW_Version` | `PackSelect_Mode_BySoc` |
| `0x08006cfc` | `APP_Calc_Pack_SOC_Percent_Rounded` | `PackSelect_ChargeStepTarget` |
| `0x08006bbc` | `PackSelect_SOCBalance_Clamp` | `PackSelect_DischargeStepThreshold` |
| `0x0800df0c` | `BMS_OTA_Upgrade_StateMachine` | `PackRecovery_LowCell_Charge_SM` |
| `0x08006d34` | `APP_Get_Packs_Charging_Bitmask` | `Pack_Get_Protect1Bit0_Mask` |
| `0x08006c6c` | `APP_Get_Packs_Charging_Bitmask_2` | `Pack_Get_ChgMosOn_Mask` |
| `0x08006cb4` | `APP_Get_Packs_Discharging_Bitmask` | `Pack_Get_DsgMosOn_Mask` |
| `0x08004cd0` | `FUN_08004cd0` | `PackSelect_Charge_StepRotate` |
| `0x08003bdc` | `FUN_08003bdc` | `PackSelect_Discharge_StepRotate` |
| `0x0800368c` | `FUN_0800368c` | `PackSelect_ModeDispatch` |
| `0x080039b0` | `FUN_080039b0` | `PackSelect_Mode1_Full` |
| `0x08003af4` | `FUN_08003af4` | `PackSelect_Mode2_LowSoc` |
| `0x08003844` | `FUN_08003844` | `PackSelect_Mode3_Empty` |
| `0x08010070` | `FUN_08010070` | `PackSelect_HighestSoc_Online` |
| `0x080100c0` | `FUN_080100c0` | `PackSelect_LowestSoc_Online` |
| `0x08004148` | `FUN_08004148` | `PackSelect_Startup_Closest50` |
| `0x08004394` | `FUN_08004394` | `Master_RunState_InitSelect` |
| `0x08004dc4` | `FUN_08004dc4` | `PackRecovery_Request_Check` |
| `0x08003708` | `FUN_08003708` | `PackSelect_MosMismatch_Fallback` |
| `0x0800352c` | `FUN_0800352c` | `PackSelect_ChargeEnable_Maintain` |
| `0x080034ac` | `FUN_080034ac` | `PackSelect_FaultPath_AllOff` |
| `0x08004924` | `FUN_08004924` | `PackFault_Build_ExcludeMask` |
| `0x0800364c` | `FUN_0800364c` | `Master_Check_AllPacksOnline` |
| `0x08004368` | `FUN_08004368` | `PackHandoff_Tick` |
| `0x08003634` | `FUN_08003634` | `LIB_Popcount` |
| `0x080040e8` | `FUN_080040e8` | `Master_Startup_WaitForEnable` |
| `0x080043c4` | `FUN_080043c4` | `OwnPack_MOSFET_Driver` |
| `0x080046b8` | `FUN_080046b8` | `OwnPack_Precharge_SM` |
| `0x08010110` | `FUN_08010110` | `ADC_Read_Precharge_Voltage` |

Decompilation export `Dekompilate/VNSD_BMS_118/` regenerated afterwards. The misnamed functions in the 117.7 program are
also corrected in the addendum below.

Addendum on the same day:

| Program | Address | before | after |
|---|---|---|---|
| 118 | `0x08010658` | `SOC_Update_Accumulator` | `prvAddCurrentTaskToDelayedList` (FreeRTOS, caller `vTaskDelay`/`vTaskPlaceOnEventList*`) |
| 117.7 | `0x08006acc` | `OTA_Validate_FW_Version` | `PackSelect_Mode_BySoc` |
| 117.7 | `0x0800df0c` | `BMS_OTA_Upgrade_StateMachine` | `PackRecovery_LowCell_Charge_SM` |
| 117.7 | `0x080105e8` | `SOC_Update_Accumulator` | `prvAddCurrentTaskToDelayedList` |
| 117.7 | `0x08006cc8` | `APP_Get_Packs_Charging_Bitmask` | `Pack_Get_Protect1Bit0_Mask` |

The function table in `VNSD_BMS_FW_Analysis_v117.7.md` has been adjusted.

---

## 10. Open points

- **Mode 1** (`PackSelect_Mode1_Full`, full pack enabled): evaluates `0x20002A1D` bit 0/1.
  Bit 0 without bit 1 → immediately to the emptiest pack. Bit 0 and bit 1 → after 300 s of discharging or
  60 s of idle. Bit 0 off → stays. `0x20002A1D` arrives as byte 12 of the CAN frame
  `0x1801FFAA`, whose ID is also present in the Control image (table `0x08056A94`). Which
  Control function sends it and what the bits mean is open.
- **`0x20002A75`** (mode 3 instead of 2 below 9 %): sender of CAN `0x100401AA` subcommand 4 open.
- **Sender of `0x1801FFAA`** (enable byte 11, flags byte 12, DoD byte 13): not found
  statically. Status of the search as of 2026-09-24:
  - The ID appears as a constant **only in the Control**, in all four D versions (147, 149, 149.2,
    150). It sits in a 7-entry table directly behind the constants (v150: `0x08056A94`:
    `0x1801FFAA, 0x100101AA, 0x100201AA, 0x100301AA, 0x100401AA, 0x100601AA, 0x180701AA`).
  - Nothing points to the table: no literal, no PC-relative `ADR`/`LDR`, no pointer in the
    unpacked `.data` (`Scripts/marstek_descriptor_unpack.py` (local, not in the repo)). It is either dead code or
    is addressed via a path that static analysis does not see.
  - Micro 116 contains no ID of this scheme and does not build one at runtime either:
    `CAN_BuildArbID` (`prio<<24 | b2<<16 | b1<<8 | b0`) is only called from `CAN_TX_SendMessage` with the
    segment number as priority; `0x18` does not arise there.
  - BMS 118 only receives it (`CAN_RX_Handler`: `0x180102AA + 0xFD00`). Its transmit IDs all
    come from `CAN_Build_Arbitration_ID`, whose low byte is the function code; `0xAA` cannot
    arise there.
  - The only other use in the Control: `FUN_0801D4E4` sends `0x100101AA` (CMD 01) and then pulses
    GPIOB pin 10. The function hangs off a key handler (`FUN_080025A4`, input
    GPIOB pin 11), so it is a manual wake-up or activation sequence. It does not
    send the enable frame.
  - This could only be clarified on the bus: take a CAN capture at device start-up and watch for frames with
    ID `0x1801FFAA`.
- **Taper table:** values only via register 32106 or an SWD dump
  (`Methodik_und_Meta/VNSD_SWD_JTAG_Dump_Plan.md`).
- **Behavior below 12 %:** The prediction from §7 has not been verified on the device. Test proposal:
  run a forced discharge via RS485 without Omnibattery until below 11 % on the active pack,
  logging SOC per pack, 34x04 (MOS status) and 32111 (`work_bat_idx`) at a 10 s interval.
  Expectation: switch at `soc% ≤ 10` to the fullest pack. Below 9 % mode 2/3, shutdown only
  when no pack is fuller.
