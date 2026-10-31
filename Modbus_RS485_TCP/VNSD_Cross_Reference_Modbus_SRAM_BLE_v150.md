# Cross-Reference Modbus Register ↔ SRAM ↔ BLE Payload (v150)

> 🇩🇪 Deutsche Version: [VNSD_Querverweis_Modbus_SRAM_BLE_v150_de.md](VNSD_Querverweis_Modbus_SRAM_BLE_v150_de.md)

**Date:** 22 August 2026
**Data:** `VNSD_Querverweis_Modbus_SRAM_BLE_v150.csv`
**Purpose:** Place the three endpoints of the same device side by side — and above all
make visible which values exist **on one side only**.

---

## 1. How the Table Is Built

The SRAM address is the common key.

| Side | Source | What it provides |
|---|---|---|
| Modbus | `VNSD_Descriptor_Table_Decoded_v150.csv` | Register → SRAM pointer, type, scale |
| BLE | `FUN_0800b024` (`0x0800b024`), buffer `0x20018857` | Payload offset → SRAM source |
| App | `StatePayload.ts` from VenusControl | Payload offset → field name |

From the decompilation of `FUN_0800b024`, 81 assignments into the BLE buffer were
extracted; for 42 of them the source could be resolved directly to an SRAM address.
The rest are constants, intermediate variables or calculations — these are listed in the column
`herkunft`.

## 2. Result

| | Rows |
|---|---|
| occupied on both sides (Modbus **and** BLE) | 19 |
| BLE only | 59 |
| BLE and API, not Modbus | 1 (`CTConnected`, BLE `0x07`: the meter-data-valid flag, also `ct_state` in the API) |
| Modbus only | 362 |

**The BLE payload is therefore not the poorer interface.** It carries 59 fields that cannot be reached via
Modbus — including the two calculated power values.

### Secured on Both Sides

| BLE | App field | SRAM | Modbus |
|---|---|---|---|
| `0x00` | BackupLoadPower | `0x20014EB6` | 30007 / 32302 |
| `0x02` | AcOutputPower | `0x20014EB4` | 30006 / 32202 / 37004 |
| `0x04` | InverterState | `0x20014E9C` | 35100 |

These three are verified against measured values (load test 600 W, see
`VNSD_BLE_vs_Modbus_Power_Values_2026-08-22.md`).

### BLE Only, Because Calculated

| BLE | App field | Formula |
|---|---|---|
| `0x8C` | BatteryPower | `32100 (×0.01 V) × 32101 (×0.1 A)` |
| `0x90` | GridPower | `s10 − off_grid_power`, with `s10` depending on `grid_sample_power` and `inv_state` |
| `0x8E` | — | `MPPT sum / 10 − s10` |
| `0x92` | — | `s10` |

These values do not need a missing register: they can be reconstructed from existing
registers.

## 3. Limitations — Please Keep in Mind When Continuing

The table is a **work in progress**, not a verified reference. Three known weaknesses:

1. **Overlapping descriptor entries.** Registers with `elem_size 4` occupy two words,
   which is why e.g. `32202` appears both at BLE `0x00` and at `0x02`. Only the
   exact base address is reliable, not every subsequent address.
2. **The app field names** come from a regex over `StatePayload.ts` and are assigned by
   offset. Where the app does not decode an offset, the column stays empty —
   that does not mean nothing is there.
3. **Coincidental hits in the join.** Some assignments in the 41000 range (`WorkMode` →
   43101, `CTType` → 41507 …) are based on SRAM addresses that are occupied in both worlds,
   without any content-wise relationship being evidenced. They are usable as a hint,
   not as a statement.

Reliable are the rows whose SRAM address corresponds exactly to a descriptor base entry —
and the three above, which were cross-checked against hardware.

## 4. What the Table Is Good For

Three questions of this evening could have been answered with it in minutes instead of hours:

- *"Is there a Modbus register for Grid Power?"* → row `0x90`, column `nur` = BLE,
  column `herkunft` names the formula.
- *"Why does Battery Power deviate?"* → row `0x8C`, different origin than Modbus 30001.
- *"Which register holds the same value as X?"* → group by the SRAM column.

The next sensible extension would be the reverse direction: go through the 362 Modbus registers without a
BLE counterpart to see which of them have ever delivered a value other than zero — for that,
the two load scans already provide two data points.
