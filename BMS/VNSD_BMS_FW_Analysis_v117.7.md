# Marstek Venus D — BMS Firmware Analysis

> 🇩🇪 Deutsche Version: [VNSD_BMS_FW_Analyse_v117.7_de.md](VNSD_BMS_FW_Analyse_v117.7_de.md)

> **Obsolete (2026-09-30):** This analysis is based on a wrong base address (0x08000000; all flash addresses here are 0x2800 too low) and wrongly assumed an AT32F403A (it is actually a Nations N32G45x). The authoritative documents are [VNSD_BMS_FW_v1177_Full_Analysis.md](VNSD_BMS_FW_v1177_Full_Analysis.md) and [VNSD_BMS_Diff_v1177_v118.md](VNSD_BMS_Diff_v1177_v118.md).

## `20251010135647565eb2036.bin`

**Firmware:** BMS v117.7 (OTA `firmwareType: bms`) — **Correction 2026-07-09:** The file name/title wrongly said "v177.7" (typo in the original OTA metadata hypothesis, see section 10.3). The hardcoded version value in the binary (`0x499` = 1177) and the live Modbus scan after the BMS upgrade (registers 30204/34010 etc. = 1177) consistently confirm **v117.7**.  
**Analysis date:** 06.07.2026  
**Method:** Static analysis (Ghidra + ReVa MCP)  
**Status:** Extended analysis — task architecture, SOC, CAN, RS485, KA495XX SPI driver, protect bitmasks, OTA update decoded

---

## 1. Binary Fingerprint

Verified live from Ghidra (status 2026-07-15). A comparison with the other five analyzed
firmware images (Control 149.2/147, VNS 116/115, BMS 118) is in the project `README.md`.

| Property | Value |
|---|---|
| File | `20251010135647565eb2036.bin` |
| Version | 117.7 |
| Size | 106,496 B (0x1A000, 104 KB) |
| Architecture | ARM Cortex-M4F, Thumb-2, little-endian |
| Flash range | `0x08000000–0x08019FFF` |
| Initial SP | `0x2000CB90` (~52 KB SRAM) |
| Reset handler | `0x08002A6D` |
| Functions | 552 / 552 named (100 %) |
| Strings | 260 |
| Compiler | RVDS/Keil ARM |
| RTOS | FreeRTOS (`heap_4`, `ARM_CM4F` port) |
| Crypto | — (none) |
| Cell monitoring | KA495XX (BMIC) |
| Communication | CAN + RS485 |

---

## 2. FreeRTOS Tasks (10 tasks)

| Task | Name | Function |
|---|---|---|
| `RTOS_vTaskLED` | LED | Status LED control |
| `RTOS_vTaskKeyScan` | KeyScan | Button/key polling |
| `RTOS_vTaskCanRec` | CAN Receive | CAN bus receive (→ inverter/Control) |
| `RTOS_vTaskCanSend` | CAN Send | CAN bus send |
| `RTOS_vTaskRS485Comm` | RS485 | RS485 communication |
| **`RTOS_vTaskSocAlgorithm`** | **SOC** | **SOC calculation** (own task!) |
| **`RTOS_vTaskProtectCheck`** | **Protection** | **Battery protection check** |
| `RTOS_vTaskControlLogic` | Control | Charge/discharge control |
| `RTOS_vTaskDataSave` | DataSave | Persistent data storage (flash/EEPROM) |
| **`RTOS_vTaskKA495XX`** | **BMIC** | **KA495XX cell monitoring IC driver** |

**Notable points:**
- SOC has its **own dedicated task** (`vTaskSocAlgorithm`) — not calculated inline
- The protection logic is also its own task (`vTaskProtectCheck`)
- KA495XX is the cell monitoring IC (similar to BQ769x0 / ISL94202)

---

## 3. Battery Measurements & BMIC Structure

Extracted from debug output strings:

### 3.1 Pack-Level Measurements

| Format string | Variable | Unit | Note |
|---|---|---|---|
| `BBat Volt:%0.01fV` | bat_voltage | V | Total pack voltage (float) |
| `Bat Curr:%0.1fA` | bat_current | A | Pack current (float, 0.1A resolution) |
| `Bat Soc:%.1f` | bat_soc | % | SOC (float, 0.1% resolution!) |
| `Cyc Cnt:%d` / `Cycle Count:%d` | cycle_count | — | Cycle counter |
| `Soh:%.1f` | soh | % | State of health |
| `Total Cap:%.1f kwh` | total_capacity | kWh | Total capacity |

### 3.2 Cell Measurements

| Format string | Variable | Description |
|---|---|---|
| `Cell Volt:` | cell_voltages[] | 16 individual cell voltages |
| `Cell NTC:` | cell_ntc[] | Cell temperature sensors |
| `Max Cell:%d  Min Cell:%d` | max_cell / min_cell | Cell voltage extremes (mV) |

### 3.3 Temperature Sensors (NTC)

| Format string | Variable | Description |
|---|---|---|
| `Max NTC:%d  Min NTC:%d Ave NTC:%d` | max/min/avg_ntc | Pack temperature statistics |
| `MOS NTC:%d  ENV NTC:%d` | mos_ntc / env_ntc | MOSFET temperature / ambient |

### 3.4 MOSFET Status

| Format string | Variable | Description |
|---|---|---|
| `Chg Mos:%d  Dsg Mos:%d` | chg_mos / dsg_mos | Charge/discharge MOSFET status |
| `Chg_MOS_CMD:%d` | chg_mos_cmd | Charge MOSFET command |
| `Dsg_MOS_CMD:%d` | dsg_mos_cmd | Discharge MOSFET command |

### 3.5 Protection Status

| Format string | Variable | Description |
|---|---|---|
| `Protect1:%d  Protect2:%d` | protect1 / protect2 | Protection bitmasks |

### 3.6 BMIC (Battery Monitor IC) Internals

| Format string | Variable | Unit | Description |
|---|---|---|---|
| `BMIC_Info.lBatPackFastCur_mA:%d` | fast_current | mA | Fast current measurement |
| `BMIC_Info.lBatPackCur_100uA:%d` | precise_current | 100µA | Precise current measurement (coulomb counting) |

---

## 4. SOC Algorithm (fully decoded)

Orchestrator: `SOC_Algorithm_Orchestrator` (346 bytes), called from `RTOS_vTaskSocAlgorithm` via `SOC_Algorithm_Entry`.

### 4.1 Architecture Overview

```
                   ┌─────────────────────────┐
                   │  RTOS_vTaskSocAlgorithm  │
                   └────────────┬─────────────┘
                                │
                   ┌────────────▼─────────────┐
                   │   SOC_State_Detect         │
                   │   State detection         │
                   │   → 1=Charge 2=Discharge  │
                   │   → 3=Idle                │
                   └────────────┬─────────────┘
                     ┌──────────┼──────────┐
                     ▼          ▼          ▼
              ┌──────────┐ ┌──────────┐ ┌──────────────────┐
              │ Charge   │ │ Discharge│ │ Idle (State 3)   │
              │ State 1  │ │ State 2  │ │                  │
              └────┬─────┘ └────┬─────┘ │ OCV correction   │
                   │            │       │ OCV_Correction   │
              ┌────▼────────────▼────┐  │                  │
              │ Coulomb_Counting     │  │ After ~1h idle:  │
              │ Coulomb counting     │  │ full calibration │
              │ (current × time)     │  │ Full_Recalibration│
              └────┬─────────────────┘  └────────┬─────────┘
                   │                             │
              ┌────▼─────────────────────────────▼──┐
              │ SOC_Smoothing — SOC smoothing        │
              │ real_soc → show_soc (smoothing)      │
              └────┬────────────────────────────────┘
                   │
              ┌────▼─────────────────────────────────┐
              │ Max_Charge_Current_Calc — Max Charge Current     │
              │ Max_Discharge_Current_Calc — Max Discharge Current  │
              └──────────────────────────────────────┘
```

### 4.2 Coulomb Counting (`Coulomb_Counting`, 334 bytes)

**Core formula:**
```
delta_soc = (current × time_ms) / (1800 × capacity_factor)
```

**Step-by-step calculation:**
1. `capacity_factor` = `Capacity_Factor_Calc(param_1 + 0x28)` → capacity factor (float)
2. `current` = `*(short *)(param_1 + 0x3E)` → current from BMIC (0.1A units)
3. Guard: if `capacity == 0` or `current == 0x7FFF` → return 0 (invalid)
4. 64-bit multiplication: `raw = current × time_delta_ms`
5. Division: `raw / 36` (time base conversion)
6. SOC delta: `delta = raw_coulombs / (50.0 × capacity_factor)`
7. **Remainder accumulator** (`_DAT_200029E0`, float):
   - Accumulates sub-integer remainders
   - If `remainder ≥ 1.0` → `delta += 1`, `remainder -= 1.0`
   - If `remainder ≤ -1.0` → `delta -= 1`, `remainder += 1.0`
   - **Prevents rounding drift** over long periods

**Outputs:**
- `*param_4` = raw coulomb counter (for energy calculation)
- `*param_5` = SOC delta (integer, added to `real_soc`)
- `*param_6` = energy delta (Wh)

### 4.3 State-Dependent Paths

| State | Functions | Description |
|---|---|---|
| **1 = Charge** | `Coulomb_Counting` → `SOC_Smoothing` → `Charge_SOC_Integration` → `Charge_Energy_Calc` | Coulomb count + charge SOC integration + energy calculation |
| **2 = Discharge** | `Coulomb_Counting` → `SOC_Smoothing` → `Discharge_SOC_Integration` → `Discharge_Energy_Calc` | Coulomb count + discharge SOC integration + energy calculation |
| **3 = Idle** | `OCV_Correction` → `OCV_Voltage_To_Index` → (1h) → `Full_Recalibration` | OCV correction + (after ~1h) full calibration |

### 4.4 OCV Calibration (idle state)

> **Correction 2026-09-24:** `OCV_Correction`, `OCV_Voltage_To_Index` and `Quick_OCV_Lookup`
> calculate **SOH and cycle count from the energy throughput**, not the SOC from the resting voltage.
> They are now named `SOH_From_Throughput`, `CycleCount_From_Throughput` and
> `SOH_Table_Lookup_ByCycles`. `total_capacity` is the energy throughput (0.1 kWh), `soh` has the
> unit 1e-4 %. Details: `VNSD_BMS_SOH_Cycles_Discharge_Limit_v118.md` (applies byte-identically to 117.7).

- After **~1 hour of idle** (timer: 3,599,999 ms) the SOC calibration is triggered
- `OCV_Correction` → Calculates SOC from **open circuit voltage** (resting voltage)
- `OCV_Voltage_To_Index` → **Lookup table** voltage → SOC (stored in flash)
- `Full_Recalibration` → **Full recalibration** using cell voltage thresholds
  from the flash table at `0x0801B79C` (charge threshold) / `0x0801B832` (discharge threshold)

### 4.5 SOC SRAM Map (0x20004A38–0x20004A78)

> **Correction 2026-09-24:** `OCV_Correction`, `OCV_Voltage_To_Index` and `Quick_OCV_Lookup`
> calculate **SOH and cycle count from the energy throughput**, not the SOC from the resting voltage.
> They are now named `SOH_From_Throughput`, `CycleCount_From_Throughput` and
> `SOH_Table_Lookup_ByCycles`. `total_capacity` is the energy throughput (0.1 kWh), `soh` has the
> unit 1e-4 %. Details: `VNSD_BMS_SOH_Cycles_Discharge_Limit_v118.md` (applies byte-identically to 117.7).

Extracted from `soc_algorithm_dump` (`soc_algorithm_dump`, 106 bytes):

| SRAM | Variable | Type | Scale | Description |
|---|---|---|---|---|
| `0x20004A38` | **real_soc** | u32 | **÷10000** | Calculated SOC (coulomb-based) |
| `0x20004A3C` | **show_soc** | u32 | **÷10000** | Displayed SOC (smoothed) |
| `0x20004A40` | max_soc | u32 | ÷10000 | Maximum SOC |
| `0x20004A44` | min_soc | u32 | ÷10000 | Minimum SOC |
| `0x20004A48` | target_soc | u32 | ÷10000 | Target SOC (calibration) |
| `0x20004A5C` | cycle_count | u32 | ×1 | Cycle counter |
| `0x20004A60` | soh | u32 | ÷10 | State of health (%) |
| `0x20004A64` | total_capacity | u32 | ÷10 | Capacity (kWh) |
| `0x20004A68` | max_chg_current | u16 | ÷10 | Max. charge current (A) |
| `0x20004A6A` | max_dsg_current | u16 | ÷10 | Max. discharge current (A) |
| `0x20004A6C` | max_cell_volt | u16 | mV | Highest cell voltage |
| `0x20004A6E` | min_cell_volt | u16 | mV | Lowest cell voltage |
| `0x20004A70` | avg_temp | i16 | °C | Average temperature |
| `0x20004A72` | max_temp | i16 | °C | Max. temperature |
| `0x20004A74` | min_temp | i16 | °C | Min. temperature |
| `0x20004A76` | current | i16 | ÷10 → A | Current |
| `0x20004A78` | tail_voltage | u16 | ÷10 → V | Tail voltage (end of charge) |
| `0x20004B1C` | full_flag | u32 | Bit0 | Full-charge detection |

### 4.6 SOC Conversion Chain (real_soc → Modbus)

```
64-bit double (internal calculation, sub-0.01% precision)
  ↓ Quantization
u32 real_soc @ 0x20004A38 (÷10000, e.g. 146000 = 14.6%)
  ↓ Smoothing/filtering (SOC_Smoothing)
u32 show_soc @ 0x20004A3C (÷10000, smoothed)
  ↓ Division by 1000
u16 @ 0x20004A14 (÷10, for CAN TX — value 146 = 14.6%)
  ↓ CAN bus (PF=1, bytes 6-7)
Micro MCU SRAM 0x20003981
  ↓ Telemetry block offset 0x0E (via per-pack struct)
Control MCU SRAM 0x20014E90+descriptor offset
  ↓ Modbus descriptor table
**Register 34002** (scale ×0.1 → 146 = 14.6%)
```

**Finding for the HA integration:** The SOC value in the Modbus register is `show_soc`, not `real_soc`.
After charge/discharge pauses, `show_soc` can "hang" for a few seconds because the
smoothing function only tracks the value slowly. This is **not a bug**, but intended behavior
to avoid SOC jumps in the user display.

### 4.7 OCV Lookup Table & Calibration Flash

> All calibration data is located in the flash range `0x0801B73C`–`0x0801BA48+`, **beyond the
> FW binary** (which ends at `0x08019FFF`). The data is factory-calibrated, cell-chemistry-specific
> and is not overwritten during FW updates.

#### Quick OCV Table (`Quick_OCV_Lookup`)

> **Correction 2026-09-24:** `OCV_Correction`, `OCV_Voltage_To_Index` and `Quick_OCV_Lookup`
> calculate **SOH and cycle count from the energy throughput**, not the SOC from the resting voltage.
> They are now named `SOH_From_Throughput`, `CycleCount_From_Throughput` and
> `SOH_Table_Lookup_ByCycles`. `total_capacity` is the energy throughput (0.1 kWh), `soh` has the
> unit 1e-4 %. Details: `VNSD_BMS_SOH_Cycles_Discharge_Limit_v118.md` (applies byte-identically to 117.7).

**Address:** `0x0801B73C`–`0x0801B773` (56 bytes)
**Format:** 7 entries × 8 bytes = `[voltage_index (u32)] [soc_value (u32)]`

```
Algorithm:
  1. Cell voltage → index via OCV_Voltage_To_Index: index = voltage / 51.2
  2. Find first entry where index <= voltage_threshold
  3. Linear interpolation between adjacent entries:
     soc = soc[n-1] + (soc[n] - soc[n-1]) × (index - volt[n-1]) / (volt[n] - volt[n-1])
  4. Result × 10000 → u32 SOC (÷10000 = %)
```

**Call chain:** Idle timer (1h) → `OCV_Correction` → `OCV_Voltage_To_Index` (index) → `Quick_OCV_Lookup` (lookup)

#### Main OCV Calibration Table (`Full_OCV_Lookup`)

**Address:** `0x0801B774`–`0x0801B845` (210 bytes)
**Format:** 21 entries × 10 bytes = 5 × u16 per row

```
Row (10 bytes):
  ┌──────────┬────────────┬────────────┬────────────┬────────────┐
  │ SOC (u16)│ Volt_Chg   │ Volt_Dsg   │ Volt_Idle  │ Volt_Col4  │
  │ Offset+0 │ Offset+2   │ Offset+4   │ Offset+6   │ Offset+8   │
  └──────────┴────────────┴────────────┴────────────┴────────────┘
  × 21 rows (presumably 0%, 5%, 10%, ..., 100%)
```

**4 voltage columns** for different operating modes:
- **Column 1** (param_2=1): charge OCV curve (higher voltage at the same SOC)
- **Column 2** (param_2=2): discharge OCV curve (lower voltage at the same SOC)
- **Column 3** (param_2=3): rest/idle OCV curve
- **Column 4** (param_2=4): reserve/calibration

**Interpolation:** Identical to the quick table, but with mode selection via `param_2`.

#### Temperature/Current Limit Matrices

**Temperature index table:** `0x0801B846`
- Access: `Temperature_Index_Lookup(0, &index, temperature/10, 0x801B846)`
- Converts temperature (°C÷10) → matrix row index

**Charge temperature ranges:** `0x0801B86A`
- Format: u16 array, thresholds for temperature zones
- Access: `*(u16)(zone * 2 + 0x801B86A)`

**Charge current matrix:** `0x0801B884`
- Format: 2D matrix, stride 0x18 (24 bytes/row = 12 × u16)
- Access: `temp_index * 0x18 + 0x801B884 + soc_zone * 2`
- Result: max. allowed charge current for temperature × SOC range

**Discharge current temperature table:** `0x0801BA34`
- Separate temperature indexing for discharge limits

**Discharge current matrix:** `0x0801BA48`
- Format: 2D matrix, stride 10 (5 × u16 per row)
- Access: `temp_index * 10 + 0x801BA48 + soc_zone * 2`
- Result: max. allowed discharge current

#### Complete Calibration Flash Layout

```
0x0801B73C ┌───────────────────────────────────────────┐
           │ Quick OCV Table (7 × 8 = 56 bytes)       │
0x0801B774 ├───────────────────────────────────────────┤
           │ Full OCV Table (21 × 10 = 210 bytes)     │
           │  └── 4 columns: Chg/Dsg/Idle/Reserve     │
0x0801B846 ├───────────────────────────────────────────┤
           │ Temperature Index Table                    │
0x0801B86A ├───────────────────────────────────────────┤
           │ Charge Temperature Ranges                  │
0x0801B884 ├───────────────────────────────────────────┤
           │ Charge Current Limit Matrix (24B/row)     │
0x0801BA34 ├───────────────────────────────────────────┤
           │ Discharge Temperature Index                │
0x0801BA48 ├───────────────────────────────────────────┤
           │ Discharge Current Limit Matrix (10B/row)  │
           └───────────────────────────────────────────┘
```

> **To read out the values:** flash dump via SWD/JTAG probe or via the debug shell
> (UART CMD `bat_data` shows some SOC values). Alternatively: RS485 CMD 0x2E sends 84 bytes
> of per-pack data, which may contain calibration parameters.

### 4.8 SOC Calibration in Multi-Pack Round-Robin Operation

The Venus D charges/discharges the packs in a **round-robin scheme** — about 10% per pack,
then rotation to the next. The question is: how does the SOC stay calibrated anyway?

#### Per-Pack Independence

Each BMS pack runs its **own `RTOS_vTaskSocAlgorithm` instance**. There is no
central SOC coordinator. While one pack is active, its BMIC counts coulombs;
the other packs are in the idle state (state 3) and wait.

#### 4 Calibration Mechanisms

**1. Coulomb counting with remainder accumulator (primary)**
- The float remainder (`_DAT_200029E0`) is accumulated over any number of short cycles
- Even with 30-second rotations no fraction is lost
- BMIC resolution: 100µA → at 10A load = 0.001% resolution per measurement
- **Round-robin is not a problem** — each pack counts its own As independently

**2. Full-flag anchor (100% reset)**
- If `show_soc > 99.4%` AND `full_flag` is set → SOC capped to 1000 (= 100.0%)
- Code in `PerPack_Struct_Builder` lines 40–43: `if (show_soc > 0x3E2 && full_flag) → soc = 1000`
- Every full charge cycle is a reset point
- Round-robin: every pack eventually reaches 100% and calibrates itself there

**3. Nightly OCV calibration (secondary)**
- Trigger: 1 hour of continuous idle (timer: 3,599,999 ms)
- At night (no solar, no consumption) all packs go idle simultaneously
- The 4 OCV columns take the previous operating state into account
- 21 support points with linear interpolation

**4. Tail-voltage detection**
- `tail_voltage` (`0x20004A78`) detects the end of the charge curve
- If the current drops exponentially at high SOC → second 100% anchor
- Independent of the full flag

#### Weak Point: LiFePO4 Mid-Range

The OCV curve of LiFePO4 cells is extremely flat in the 20–80% range (~3.2V ± 20mV).
Even with a 21-point table and 4 mode columns, the voltage→SOC resolution there is low.

**Consequence:** If a pack never reaches 100% or 0% for weeks (because the rotation keeps it
between 30–70%), the coulomb counter can slowly drift. The nightly OCV correction
catches this partially, but the correction accuracy in the mid-range is limited.

**Observable effect:** Packs sometimes show slightly different SOC values despite
the same amount of energy — this is **not a bug**, but an inherent limitation of
coulomb counting with a flat voltage curve without regular full-charge calibration.

### 4.9 Round-Robin Mechanism (verified from FW)

> **Qualification 2026-09-24:** The statement that the rotation lies solely with the Micro is
> obsolete. The pack selection by SOC (mode selection `PackSelect_Mode_BySoc`,
> recovery charging `PackRecovery_LowCell_Charge_SM`) runs in the master BMS and is already
> present byte-identically with 118 in 117.7. Description for 118:
> `VNSD_BMS_Pack_Rotation_Taper_v118.md`. The CAN commands below still exist; whether
> the Micro sends them in normal operation has not been checked.

> The rotation is **not** controlled by the BMS itself. Each BMS pack is a passive
> receiver of activation commands. The orchestration lies with the Micro/inverter MCU,
> which addresses individual packs via CAN commands.

#### Pack Mode Register (`DAT_200028CA`)

| Value | Mode | Description |
|------|-------|-------------|
| 0 | **Idle** | MOSFETs off, pack waits for activation |
| 1 | **Transitional** | Transition state (handoff to the next pack) |
| 2 | **Active** | MOSFETs on, pack actively charges/discharges |
| 3 | **Shutdown** | Sleep mode (watchdog timeout) |
| 4 | **Error** | Fault state (forced stop) |

#### CAN Command-Based Pack Activation (`CAN_CMD_Pack_Activation`)

**Address matching (lines 19–20):**
```c
if (((uVar2 & 0xFFF) >> 8 == DAT_200041B6) ||   // Match own address
    ((uVar2 & 0xFFF) >> 8 == 0xF) ||              // Broadcast (0xF)
    ((uVar2 & 0xFFF) >> 8 == 0))                   // Broadcast (0x0)
```

The CAN message contains the **target pack address** in bits 8–11 of the data.
The Micro MCU can thereby address individual packs selectively.

**Activation (CAN CMD 6):**
```c
if (bVar1 == 6) {
    if (DAT_200028CA == 0 && DAT_200028E3 < 11) {
        DAT_200028CA = 2;    // Idle → Active!
    }
}
```
Guard: only if the pack is in idle mode AND there are fewer than 11 error events.

**Handoff to the next pack (CAN CMD 3):**
```c
if (DAT_200028CA == 2 &&                              // Pack is active
    *(byte*)(param_1 + 0xB) == DAT_200041B6 + 1) {   // Data byte = NEXT pack address
    DAT_200028D9 = 1;                                  // "Ready for handoff" flag
}
```
The data byte of the CAN message contains the address of the **next** pack.
The current pack sets a ready flag and prepares the MOSFET shutdown.

#### Complete Round-Robin Flow

```
Micro MCU (inverter) orchestrates the cycle:

1. CAN CMD 6 → Pack 1 (addr match)      Pack 1: Idle → Active (MOSFETs ON)
   Pack 2-6: stay Idle                         ↓ Charge/discharge for ~10% SOC
                                               ↓
2. CAN CMD 3 → Pack 1 (data=Pack2-Addr) Pack 1: DAT_200028D9 = 1 (handoff ready)
                                               ↓ Prepare MOSFET shutdown
                                               ↓
3. CAN CMD 6 → Pack 2 (addr match)      Pack 2: Idle → Active (MOSFETs ON)
   Pack 1: Active → Idle                      ↓ Charge/discharge for ~10% SOC
                                               ↓
4. CAN CMD 3 → Pack 2 (data=Pack3-Addr) Pack 2: handoff ready
                                               ↓
5. ... (repeated for all packs)                ↓
                                               ↓
N. CAN CMD 3 → Pack 6 (data=Pack1-Addr) Pack 6: handoff → back to Pack 1
```

#### Architecture Finding

```
Control MCU (EMS)
  │ RS485: "Charge at 2000W"
  ↓
Micro MCU (inverter)
  │ Decides: which pack, when to switch
  │ CAN CMD 6: activate Pack N
  │ CAN CMD 3: Pack N → Pack N+1 handoff
  ↓
BMS master (Pack 1)
  │ RS485 CMD 0x29: forwarding to slaves
  ↓
BMS slaves (Pack 2-7)
  │ MOSFET on/off according to mode
  ↓
Battery packs (physically on the DC bus)
```

> **The Micro MCU is the rotation orchestrator.** It decides when a pack has
> charged/discharged enough and activates the next one. The Control MCU only specifies
> the total power; the pack selection is the job of the Micro MCU.

#### Verified Timers (from Micro FW `FUN_0800d834`)

| Timer | SRAM | Timeout | Function |
|---|---|---|---|
| Main cycle | `0x20000470` | 3600s (60 min) | Direction decision: SOC<50%→charge, SOC≥50%→discharge |
| Discharge sub | `0x20000472` | 600s (10 min) | CAN update to BMS during discharge |
| Charge sub | `0x20000474` | 600s (10 min) | CAN update to BMS during charge |

#### Effect on PV Feed-in

The current limit matrix in the calibration flash (`0x0801B884`, 2D: temperature × SOC zone)
progressively reduces the allowed charge current as SOC rises. Since the Venus D has only
**one bidirectional power path** (no separate PV→grid path), the
BMS throttling causes PV power to be increasingly curtailed from ~90% SOC:

```
SOC  50% → charge_current_limit = 50A → Inverter: -2500W → PV fully used
SOC  80% → charge_current_limit = 30A → Inverter: -1500W → PV partially curtailed
SOC  95% → charge_current_limit = 5A  → Inverter: -250W  → PV heavily curtailed
SOC 100% → charge_current_limit = 0A  → Inverter: 0W     → PV completely curtailed
→ Direction change after timer → discharge → PV flows into the grid again
```

> Detailed analysis of the DC bus architecture and PV behavior: see
> `VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analysis_vd_inv_app_0116.md`, section 12.7

---

## 5. Protection Logic & Protect Bitmasks (fully decoded)

### 5.1 Protect Register Architecture

```
Protection-Checker Tasks  →  3 Status-Bytes  →  2 Bitmask-Builder  →  Per-Pack Struct  →  CAN TX
                              DAT_200028C1        Protect1_Bitmask_Builder  0x2000420C          PF=4
                              DAT_200028C2        Protect2_Bitmask_Builder  0x2000420E          Bytes 0-3
                              DAT_200028C3
```

**Status bytes (protection sources):**
- `DAT_200028C1` — current protection (set by `Current_Protection_Checker`, 544 bytes)
- `DAT_200028C2` — voltage/current protection (set by `Voltage_Temp_Protection_Checker`)
- `DAT_200028C3` — temperature protection (set by `HW_Overcurrent_Protection`)

### 5.2 Protect1 Bitmask (`Protect1_Bitmask_Caller` → `Protect1_Bitmask_Builder`)

> **Correction 2026-09-24 — Bit 0 and bit 1 were swapped.** The assignment "bit 0 ← C1 bit 5,
> bit 1 ← C1 bit 6" is correct, the meaning is not. In BMS 118,
> `Cell_OVP_UVP_TempCompensated_Protection` (0x0800a584) sets **C1 bit 5 at `Min Cell < 2500 mV`**
> (cold 2300 mV) and forces the SOC to 0, and **C1 bit 6 at `Max Cell` above 3700/3800 mV** (cold
> 3850/3900 mV) and forces the SOC to 100. Bit 0 = **cell undervoltage**, bit 1 =
> **cell overvoltage**. This fits the recovery charging, which charges packs with bit 0 set until
> `Min Cell > 3100 mV`. In 117.7 no writer for C1 bit 5/6 is disassembled,
> but the function is present there unchanged. The table below is left in place as the original status.

| Bit | Value | Source | Verified from | Protection type |
|-----|------|--------|----------------|-----------|
| 0 | 0x0001 | C1 bit 5 | BMIC/KA495XX HW register | **Cell overvoltage (Cell OVP)** |
| 1 | 0x0002 | C1 bit 6 | BMIC/KA495XX HW register | **Cell undervoltage (Cell UVP)** |
| 2 | 0x0004 | C1 bit 1 | `Current_Protection_Checker`: `current > DAT_200049C8` | **Charge overcurrent L2 (Charge OCP)** |
| 3 | 0x0008 | C1 bit 0 | `Current_Protection_Checker`: `|current| > DAT_200049B8` | **Discharge overcurrent L2 (Discharge OCP)** |
| 4 | 0x0010 | C2 bit 5 | `Voltage_Temp_Protection_Checker`: `max_cell_ntc > DAT_20004978` (charging active) | **Charge overtemperature (Charge OTP)** |
| 5 | 0x0020 | C2 bit 6 | `Voltage_Temp_Protection_Checker`: `max_cell_ntc < threshold` (charging active) | **Charge undertemperature (Charge UTP)** |
| 6 | 0x0040 | C2 bit 3 | `Voltage_Temp_Protection_Checker`: `max_cell_ntc > threshold` (discharging active) | **Discharge overtemperature (Discharge OTP)** |
| 7 | 0x0080 | C2 bit 4 | `Voltage_Temp_Protection_Checker`: `max_cell_ntc < threshold` (discharging active) | **Discharge undertemperature (Discharge UTP)** |
| 8 | 0x0100 | C2 bit 0 | `Voltage_Temp_Protection_Checker`: `mos_ntc > DAT_20004958` | **MOS overtemperature (MOS OTP)** |
| 9 | 0x0200 | C1 bit 2 | `voltage_protection_check`: GPIOC.2=LOW + GPIOC.1=HIGH | **HW short circuit (SCP, 25× debounce!)** |
| 10 | 0x0400 | C3 bit 7 | `HW_Overcurrent_Protection`: GPIOD.5=LOW + `current > 3A` | **HW charge overcurrent + relay** |
| 11 | 0x0800 | C3 bit 6 | `HW_Overcurrent_Protection`: GPIOC.1=LOW + `current < -3A` | **HW discharge overcurrent + relay** |

### 5.3 Protect2 Bitmask (`Protect2_Bitmask_Builder`)

| Bit | Value | Source | Verified from | Protection type |
|-----|------|--------|----------------|-----------|
| 9 | 0x0200 | C3 bit 5 | `HW_Overcurrent_Protection`: MOS/ENV NTC >150.1°C or <-30.1°C | **NTC out of range (sensor fault)** |
| 11 | 0x0800 | C3 bit 3 | — (setter not identified) | **Temperature protection level 3** |
| 12 | 0x1000 | C2 bit 2 | `Voltage_Temp_Protection_Checker`: `env_ntc > DAT_20004940` | **Ambient overtemperature (ENV OTP)** |
| 14 | 0x4000 | `DAT_20002901` ≠ 0 | Checked directly | **Communication fault** |
| 15 | 0x8000 | `DAT_20002A0F/10` ≠ 0 | Checked directly | **Critical fault / error lock** |

### 5.4 2-Level Protection System

```
Level 1: WARNING (C5/C6 → warning register via Warning_Bitmask_Builder)
  → Lower thresholds, shorter timeouts
  → Sets warning flags only, no MOSFET shutdown

Level 2: PROTECTION (C1/C2/C3 → Protect1/2 via Protect1_Bitmask_Caller/Protect2_Bitmask_Builder)
  → Higher thresholds, longer timeouts
  → Switches off the charge/discharge MOSFETs
  → On short circuit: hardware shutdown via KA495XX GPIO
```

**Verified Temperature Thresholds (from `Voltage_Temp_Protection_Checker`):**

| SRAM (Threshold) | Measurement source | Protection type | For C-byte |
|---|---|---|---|
| `0x20004954` | MOS NTC (`0x200041D6`) | MOS overtemp SET (L1) | C5 |
| `0x20004958` | MOS NTC | MOS overtemp SET (L2) | C2 bit 0 |
| `0x2000495C` | MOS NTC | MOS overtemp CLEAR | Recovery |
| `0x2000493C` | ENV NTC (`0x200041D8`) | ENV overtemp SET (L1) | C6 |
| `0x20004940` | ENV NTC | ENV overtemp SET (L2) | C2 bit 2 |
| `0x20004944` | ENV NTC | ENV overtemp CLEAR | Recovery |
| `0x20004948` | ENV NTC | ENV undertemp SET (L1) | C6 |
| `0x2000494C` | ENV NTC | ENV undertemp SET (L2) | C2 bit 1 |
| `0x20004950` | ENV NTC | ENV undertemp CLEAR | Recovery |
| `0x20004978` | Max Cell NTC (`0x200041C8`) | Cell charge overtemp | C2 bit 5 |

**Verified Current Thresholds (from `Current_Protection_Checker`):**

| SRAM (Threshold) | Measurement source | Protection type | For C-byte |
|---|---|---|---|
| `0x200049B6` | Pack current (`0x20004202`) | Discharge overcurrent L1 | C6 bit 6 |
| `0x200049B8` | Pack current | Discharge overcurrent L2 | C1 bit 0 |
| `0x200049BA` | Pack current | Discharge overcurrent L3 | C1 bit 0 |
| `0x200049C2` | — | Discharge overcurrent recovery | Recovery |
| `0x200049C6` | Pack current | Charge overcurrent L1 | C6 bit 7 |
| `0x200049C8` | Pack current | Charge overcurrent L2 | C1 bit 1 |
| `0x200049CA` | Pack current | Charge overcurrent L3 | C1 bit 1 |
| `0x200049D2` | — | Charge overcurrent recovery | Recovery |

### 5.5 Undervoltage Protection Flow

From debug strings and `Current_Protection_Checker`/`Voltage_Temp_Protection_Checker`:

```
g_u16UvMinVolt falls below threshold
  → g_u8UvChgIndex++ (level counter)
  → When limit is reached:
      g_u8UvpAllowCutDmosFlag = 1
      → Discharge MOSFET is switched off (Dsg_MOS_CMD = 0)
      → "error lock set" → permanent fault state
      → Reset only via "error lock reset" or power cycle
```

### 5.6 Per-Pack Struct Population (`PerPack_Struct_Builder`, 580 bytes)

Additional findings from the struct builder function:

| Struct offset | Source | Conversion | Description |
|---|---|---|---|
| 0x0E (SOC) | `_DAT_20004AD0` (show_soc u16) | direct | Capped at 1000 (=100%) if >994 AND full_flag |
| 0x10 (MOSFET) | `DAT_20004B1C` bits 2-3 | bit mapping | Charge(bit1)/discharge(bit0) status |
| 0x10 (MOSFET) | `DAT_200028C1` bits 5-6 | bits 2-3 | OV/UV protection active |
| 0x10 (MOSFET) | GPIO `0x40010C00` bit 1 | bit 4 | Hardware MOSFET status |
| 0x10 (MOSFET) | GPIO `0x40011000` bit 5 | bit 5 | Hardware MOSFET status |
| 0x4A (Bat Volt) | `_DAT_200040B4` | **÷10** | Source in mV, target in 0.01V |
| 0x4C (Bat Curr) | `_DAT_2000406C` | **÷1000** | Source in µA (BMIC), target in 0.1A |
| 0x54 (Version) | Hardcoded `0x499` | — | **= 1177 → version 117.7** |

---

## 6. CAN Protocol (fully decoded)

> The CAN TX data at `0x20004Axx` is an **aggregation** across all packs.
> The per-pack raw data is in the cell data struct (section 6.0).

All TX functions (`CAN_TX_PF1_PackMeasurements`–`CAN_TX_PF4_ProtectWarnings`) and the RX handler (`CAN_RX_Handler`)
have been decompiled. Sender orchestration: `CAN_TX_Orchestrator` calls all 4 TX functions sequentially.

### 6.0 Per-Pack Data Structure (96 bytes / 0x60)

Extracted from the debug dump function `bms_data_printf` (`bms_data_printf`, 770 bytes).
Iterates over `pack_count` (`DAT_200028DB`) packs.

**Struct base:** `0x200041B6 + pack_index × 0x60`

| Offset | SRAM (Pack 0) | Type | Scale | Debug name | Description |
|--------|--------------|------|-------|------------|-------------|
| 0x00 | `0x200041B6` | u8 | — | Addr | Pack address/ID |
| 0x01 | `0x200041B7` | u8 | — | Online | Connection status (0/1) |
| 0x02 | `0x200041B8` | 12B | — | (UID) | Board UID from OTP `0x1FFFF270` |
| 0x0E | `0x200041C4` | **u16** | **÷10** | **Bat Soc** | **SOC (0.1% units!)** |
| 0x10 | `0x200041C6` | u8 | bits | Chg/Dsg Mos | Bit5=charge, bit4=discharge MOSFET |
| 0x12 | `0x200041C8` | i16 | °C | Max NTC | Max. temperature |
| 0x14 | `0x200041CA` | i16 | °C | Min NTC | Min. temperature |
| 0x16 | `0x200041CC` | i16[5] | °C | Cell NTC[0–4] | 5 temperature sensors (10 bytes) |
| 0x20 | `0x200041D6` | i16 | °C | MOS NTC | MOSFET temperature |
| 0x22 | `0x200041D8` | i16 | °C | ENV NTC | Ambient temperature |
| 0x24 | `0x200041DA` | u16 | — | Cyc Cnt | Cycle counter |
| 0x26 | `0x200041DC` | **u16[16]** | **mV** | **Cell Volt[0–15]** | **16 individual cell voltages (32B)** |
| 0x46 | `0x200041FC` | u16 | mV | Max Cell | Highest cell voltage |
| 0x48 | `0x200041FE` | u16 | mV | Min Cell | Lowest cell voltage |
| 0x4A | `0x20004200` | u16 | **÷100→V** | Bat Volt | Pack voltage (0.01V) |
| 0x4C | `0x20004202` | i16 | **÷10→A** | Bat Curr | Pack current (0.1A) |
| 0x4E | `0x20004204` | i16 | °C | Ave NTC | Average temperature |
| 0x50 | `0x20004206` | u16 | **÷10→A** | Max Chg Curr | **Max. charge current of this pack (0.1A)** — see addendum below |
| 0x52 | `0x20004208` | u16 | **÷10→A** | Max Dsg Curr | **Max. discharge current of this pack (0.1A)**, 0 below 8 % SOC — see addendum below |
| 0x54 | `0x2000420A` | u16 | — | Version | BMS pack FW version |
| 0x56 | `0x2000420C` | u16 | bits | Protect1 | Protection bitmask 1 |
| 0x58 | `0x2000420E` | u16 | bits | Protect2 | Protection bitmask 2 |
| 0x5A | `0x20004210` | u16 | bits | Warn1 | **Warning bitmask** (`Warning_Bitmask_Builder`) — see addendum below |
| 0x5C–0x5F | — | 4B | — | — | (Padding, set to 0 by the builder) |

> **Addendum (2026-09-22) — offsets 0x50/0x52/0x5A resolved, verified against BMS 118:**
>
> `PerPack_Struct_Builder` (118: `0x08009fdc`) writes starting from base `0x200041DC` — that is struct offset
> 0x26, the start of the cell array. This puts `+0x2A` at 0x50, `+0x2C` at 0x52, `+0x2E` at 0x54,
> `+0x30` at 0x56, `+0x32` at 0x58 and `+0x34` at 0x5A. The anchoring is unambiguous because three
> of these targets are already documented and match exactly: `+0x2E` writes the constant `0x76` = 118
> to the version offset 0x54, `+0x30`/`+0x32` deliver `Protect1_Bitmask_Builder()` and
> `Protect2_Bitmask_Builder()` respectively to 0x56/0x58.
>
> | Offset | Source in the builder | Meaning |
> |---|---|---|
> | 0x50 | `Protect_Get_MaxChgCurrent_Div10()` → SOC SRAM map `0x20004A38 + 0x30` | **max_charge_current**, 0.1 A |
> | 0x52 | `SOC_Get_Charge_Current_If_SOC_Above_8()` → `0x20004A38 + 0x32` | **max_discharge_current**, 0.1 A, **0 as long as `real_soc < 80000` (= 8 %)** |
> | 0x5A | `Warning_Bitmask_Builder()` | **warn1** bitmask, not padding |
>
> **Where 0x50 comes from:** `SOC_Algorithm_Orchestrator` stores the result of `Max_Charge_Current_Calc`
> (118: `0x0800fe60`) at `0x20004A38 + 0x30`. This function indexes the charge current matrix from
> the calibration flash (in 118 at `0x0801B9BC`, temperature index `0x0801B97E`, zone thresholds
> `0x0801B9A2`) by **its own SOC × its own temperature**, i.e. strictly per pack. The matrix value is
> a **percentage relative to 50 A full scale**: `result = matrix_value × 50.0 / 10.0`, capped
> at 500 (= 50.0 A) as soon as the matrix value is ≥ 101. At SOC = exactly 100 % the function
> returns 0 directly, and `0xFFFF` for an invalid SOC/temperature.
>
> **Who consumes the fields (master side):**
> - `FUN_080093fc` forms the **minimum over exactly two pack indices** from offset **0x50** —
>   `DAT_20002A11` (active pack) and `DAT_20002A12` (handoff target) — and only while
>   `DAT_20002A0E == 0` (handoff in progress); otherwise `DAT_20002A11` alone counts. The result is capped at
>   500 and written to `0x20004A18` = `charge_current_limit` in CAN PF=3. In addition: 0 as soon as the
>   aggregate SOC is ≥ 1000 (100.0 %) or the pack mode `DAT_20002A0C != 2`. The
>   `charge_voltage_limit` next to it (`0x20004A16`) is the constant `0x240` = **57.6 V**.
> - `FUN_080094c4` does the same for offset **0x52** → `0x20004A1C` = `discharge_current_limit`,
>   but **without** forming a minimum, from the active pack only; `discharge_voltage_limit` (`0x20004A1A`) is
>   the constant `0x1E0` = **48.0 V**.
>
> **There is no summation across packs in this path** — the value the inverter receives is
> always the minimum or the single value of one pack, never the sum of several. Practical consequence:
> a single pack in the top-of-charge taper caps the total charge power of the device.
>
> **0.1 A scaling confirmed three times:** cap 500 = 50 A, error fallback value 250 = 25 A in
> `CAN_TX_PF3_ChargeDischargeLimits`, and the 50 A limit at 50 % SOC mentioned in §4.9.
>
> **Validity for 117.7:** `PerPack_Struct_Builder` is identical in content between 117.7 and 118
> except for the version constant (see `VNSD_BMS_FW_Analysis_v118.md` §4.7) — the mapping applies to both
> versions. The flash addresses of the matrices differ (117.7: `0x0801B884`, see §4.7).

**Scale factors confirmed from debug print conversions:**
- SOC: `VectorUnsignedToFloat(raw) / 10.0` → **scale ×0.1** (confirms Modbus reg 34002!)
- Bat Volt: `VectorUnsignedToFloat(raw) / 100.0` → **scale ×0.01**
- Bat Curr: `VectorSignedToFloat(raw) / 10.0` → **scale ×0.1**
- Cell Volt: directly in mV (no divisor) → **scale ×0.001** for volts

**Multi-pack memory (up to 8 packs):**
```
Pack 0: 0x200041B6 – 0x20004215
Pack 1: 0x20004216 – 0x20004275
Pack 2: 0x20004276 – 0x200042D5
  ...
Pack 7: 0x200044B6 – 0x20004515
→ Aggregated CAN TX data: 0x20004A0C – 0x20004A32
```

### 6.0.1 Hypothesized Struct → Modbus Register Mapping

| Struct offset | Field | Pack 1 reg | Pack 2 reg | Scale |
|---|---|---|---|---|
| 0x4A (Bat Volt) | pack_voltage | **34000** | **34100** | ×0.01 |
| 0x4C (Bat Curr) | pack_current | **34001** | **34101** | ×0.1 |
| 0x0E (Bat Soc) | pack_soc | **34002** | **34102** | **×0.1** |
| 0x24 (Cyc Cnt) | cycle_count | **34003** | **34103** | ×1 |
| 0x54 (Version) | bms_version | **34010** | **34110** | ×1 |
| — (Cell NTC 0-3, MOS/ENV/Avg NTC) | cell_ntc_0..3, mos_ntc, env_ntc, avg_ntc | **34011–34017** | **34111–34117** | ÷10 °C |
| 0x26 (Cell[0]) | cell_volt_1 | **34018** | **34118** | ×0.001 |
| 0x28 (Cell[1]) | cell_volt_2 | **34019** | **34119** | ×0.001 |
| ... | ... | ... | ... | ... |
| 0x44 (Cell[15]) | cell_volt_16 | **34033** | **34133** | ×0.001 |
| — (Msg 1, undocumented, see §6.1) | charge_status | **34004** | **34104** | ×1 |
| 0x46 (Max Cell) | max_cell_volt | **34005** | **34105** | ×0.001 |
| 0x48 (Min Cell) | min_cell_volt | **34006** | **34106** | ×0.001 |
| 0x12 (Max NTC) | max_ntc | **34007** | **34107** | ×1 |
| 0x14 (Min NTC) | min_ntc | *(no dedicated Modbus register — see correction below)* | | ×1 |

> **Correction (2026-07-10, verified via Ghidra + live scan CSV):** Registers 34004–34007 were
> wrongly shifted by 1 (34004?=max_cell, 34005?=min_cell, 34006?=max_ntc, 34007?=min_ntc — all
> marked uncertain). Cause: the CAN sender `CAN_TX_PerPack_10Msgs` (`0x08005af0`) sends a so far **undocumented Msg 1**
> (`0x1821AAxx`, struct offsets 0x50/0x52/0x10 + a charge/discharge flag computed from `bat_curr`)
> between Msg 0 (`0x1820AAxx`) and Msg 2 (`0x1822AAxx`). It occupies
> register 34004 — the live scan finding for 34004 (`pack1_charge_status`,
> "=3 exactly when the pack is actively charging, =0 when idle. Not a voltage delta!") matches exactly. As a result,
> Msg 2 (max_cell_volt/min_cell_volt/max_ntc) shifts back by one register each to 34005–34007;
> for "min_ntc" (Msg 2 byte 6-7, offset 0x14) there is **no** dedicated Modbus register according to the CSV
> (34008/34009 are already unambiguously protect1/protect2 from Msg 3, no shift needed there).

> **Correction (2026-07-10, verified via Ghidra + live scan CSV):** The 16 cell voltages were originally
> wrongly mapped here to 34011–34026 (off-by-7 error). The live scan (`Modbus_RS485_TCP/VNSD_Register_Map_Final_generated.csv`,
> ✅ FW-verified) clearly shows: 34011–34017 contain the 7 NTC temperature values (values in the range 190–400,
> matching ÷10°C), while 34018–34033 contain the 16 cell voltages (values in the range 3190–3400,
> matching ×0.001V for Li-ion cells). `BLE/VNSD_BLE_Modbus_CrossReference.md` had already documented the correct offset (34018–34033);
> this file was the deviating/incorrect source.

### 6.1 BMS → Inverter (TX, Extended Frame)

CAN ID format: `0x18 0x PF AA 01` — BMS (address `0xAA`) sends to inverter (address `0x01`).

#### PF=1: Pack Measurements (`0x1801AA01`) — `CAN_TX_PF1_PackMeasurements`

| Byte | BMS SRAM | Type | Content |
|------|----------|-----|--------|
| 0–1 | `0x20004A0C` | u16 | **bat_voltage** (pack voltage, 0.01V) |
| 2–3 | `0x20004A0E` | i16 | **bat_current** (pack current, 0.01A) |
| 4–5 | `0x20004A10` | i16 | **max_temperature** (°C) |
| 6–7 | `0x20004A14` | u16 | **soc** (scale 0.1%, show SOC!) |

#### PF=2: Capacity & Pack Info (`0x1802AA01`) — `CAN_TX_PF2_Capacity`

| Byte | BMS SRAM | Type | Content |
|------|----------|-----|--------|
| 0–1 | computed | u16 | **design_capacity** = (pack_count+1) × cell_cap × 512 / 10 |
| 2 | `0x200028DB` +1 | u8 | **pack_count** + 1 |
| 3–4 | `0x200028DC` | u16 | (unknown, u16) |
| 5 | `0x20002A11` +1 | u8 | (counter + 1) |
| 6 | `0x200028E0` | u8 | (config value, settable via CAN CMD 0x1003) |
| 7 | `0x200028D0` | u8 | (status byte) |

> **Capacity calculation:** `(DAT_200028DB + 1) * _DAT_20004AE4 * 0x200 / 10`
> where `_DAT_20004AE4` is the cell/module capacity and `0x200 = 512` is a scaling factor.

#### PF=3: Charge/Discharge Limits (`0x1803AA01`) — `CAN_TX_PF3_ChargeDischargeLimits`

| Byte | BMS SRAM | Type | Content |
|------|----------|-----|--------|
| 0–1 | `0x20004A16` | u16 | **charge_voltage_limit** (max 0x261 = 609 → capped!) |
| 2–3 | `0x20004A18` / 0xFA | u16 | **charge_current_limit** (250 on fault) |
| 4–5 | `0x20004A1C` / 0 | u16 | **discharge_current_limit** (0 on fault) |
| 6 | `0x20002A6C` | u8 | **cell_status_flags** (bit0: all cells full, bit1: cell at 0) |
| 7 | computed | u8 | **combined_flags** (bit0: FUN_058A4 result, bit1: charge_mode==2 && !flag) |

**Cell voltage scan logic (lines 23–30):**
```
For each cell (0..pack_count):
  if cell_voltage == 1000 → full_count++
  if cell_voltage == 0    → empty_count++
→ bit0 of cell_status_flags = (full_count >= pack_count)
```
Cell data struct: `DAT_200041C4 + cell_index * 0x60` (96 bytes per cell/module!)

#### PF=4: Protection & Warnings (`0x1804AA01`) — `CAN_TX_PF4_ProtectWarnings`

| Byte | BMS SRAM | Type | Content |
|------|----------|-----|--------|
| 0–1 | `0x20004A30` | u16 | **protect1** (protection bitmask 1) |
| 2–3 | `0x20004A32` | u16 | **protect2** (protection bitmask 2) |
| 4–5 | — | u16 | (reserved, always 0) |
| 6 | `0x20002894` | u8 | **warn_byte** |
| 7 | `0x20002A71` | u8 | **status_flag** |

### 6.2 Inverter/Control → BMS (RX) — `CAN_RX_Handler`

CAN ID formats:
- `0x10 PF 01 AA` — direct commands to BMS (address `0xAA`)
- `0x18 01 xx AA` — relayed/periodic messages

#### Direct Commands (`0x10xxyyAA`)

| CAN ID | PF | Handler | Description |
|--------|------|---------|-------------|
| `0x100101AA` | 01 | `CAN_CMD_01_Handler(byte)` | Single-byte command |
| `0x100201AA` | 02 | `CAN_CMD_02_Handler(byte)` | Single-byte command (different handler) |
| `0x100301AA` | 03 | `DAT_200028E0 = byte` + `KA495XX_Write_Balancing_Register()` | Set config value |
| `0x100401AA` | 04 | `CAN_CMD_04_Handler(data_ptr)` | Multi-byte data handler |
| `0x100501AA` | 05 | `DAT_20002A70 = 1` (only if byte==1) | Set simple flag |

#### Relay/Periodic (`0x1801xxAA`)

| CAN ID | PS | Description |
|--------|------|-------------|
| `0x180101AA` | 01 | Invoke periodic send with address byte |
| `0x180102AA` | 02 | Invoke periodic send (from inverter) |
| `0x180103AA` | 03 | Invoke periodic send |
| `0x180104AA` | 04 | Invoke periodic send |
| `0x180106AA` | 06 | Invoke periodic send |
| `0x1801FFAA` | FF | **FW update trigger**: `DAT_20002A72=1`, pack ID=data[1] |

### 6.3 CAN Addressing

| Address | Meaning |
|---|---|
| `0x01` | Inverter (Micro MCU) |
| `0xAA` | BMS (default address) |
| `0xFF` | Broadcast / FW update |

### 6.4 BMS SRAM Summary (CAN-relevant addresses)

| SRAM | Variable | Source | Description |
|---|---|---|---|
| `0x20004A0C` | bat_voltage | ADC/BMIC | Pack voltage (u16) |
| `0x20004A0E` | bat_current | ADC/BMIC | Pack current (i16) |
| `0x20004A10` | max_temperature | NTC scan | Max. temperature (i16) |
| `0x20004A14` | show_soc | SOC task | Displayed SOC (u16, ×0.1%) |
| `0x20004A16` | charge_volt_limit | Protection | Charge voltage limit (u16, max 609) |
| `0x20004A18` | charge_curr_limit | Protection | Charge current limit (u16) |
| `0x20004A1C` | discharge_curr_limit | Protection | Discharge current limit (u16) |
| `0x20004A24` | cell_count | Config | Number of cells/modules |
| `0x20004A30` | protect1 | Protection | Protection bitmask 1 (u16) |
| `0x20004A32` | protect2 | Protection | Protection bitmask 2 (u16) |
| `0x20004AE4` | cell_capacity | Config | Cell/module capacity |
| `0x200041C4` | cell_data[0] | BMIC | Cell data array (96B per cell!) |
| `0x200028DB` | pack_count | Config | Number of additional packs (total = +1) |
| `0x200028DC` | pack_online_mask | Watchdog | Bitmask of online packs (one bit per pack) |
| `0x200028E0` | config_value | CAN CMD 03 | Config value settable via CAN |
| `0x20002894` | warn_byte | Protection | Warning byte |
| `0x20002A6C` | cell_status_flags | computed | Bit0: all full, bit1: empty cell |
| `0x20002A71` | status_flag | Control | General status flag |

### 6.5 Multi-Pack Per-Pack CAN Protocol (fully decoded)

> **New.** The sender function `CAN_TX_PerPack_10Msgs` (1,868 bytes!) sends **10 CAN messages per pack**.
> Called by `CAN_TX_Orchestrator`: for `param_1 == 0xFF` (broadcast) for all packs,
> otherwise for a single pack.

#### CAN ID encoding

```
CAN-ID = 0x18 [MsgGroup] AA [PackID]

PackID:  1-based (1 = Pack 1, 2 = Pack 2, ... max 9)
MsgGroup: 0x20-0x23 = pack overview (4 messages)
          0x30-0x33 = cell voltages (4 messages × 4 cells = 16 cells)
          0x40-0x41 = temperatures (2 messages)
AA:      BMS address (fixed 0xAA)
```

**Examples:**
- Pack 1 overview Msg 0: `0x1820AA01`
- Pack 1 cells 0–3:     `0x1830AA01`
- Pack 2 overview Msg 0: `0x1820AA02`
- Pack 3 cells 12–15:   `0x1833AA03`

#### 10 CAN messages per pack

**Msg 0 — Pack overview (`0x1820AA0x`):**

| Byte | Struct offset | Content | Scale |
|------|---------------|---------|-------|
| 0–1 | 0x4A | **bat_volt** (pack voltage) | ×0.01 V |
| 2–3 | 0x4C | **bat_curr** (pack current) | ×0.1 A |
| 4–5 | 0x0E | **bat_soc** (SOC) | ×0.1 % |
| 6–7 | 0x24 | **cycle_count** | ×1 |

**Msg 1 — Limits & status (`0x1821AA0x`):**

| Byte | Struct offset / source | Content |
|------|------------------------|---------|
| 0–1 | `_DAT_20004A16` (global) | **charge_volt_limit** |
| 2–3 | 0x50 | **max_charge_current** (`DAT_20004206`, ×0.1 A) |
| 4–5 | 0x52 | **max_discharge_current** (`DAT_20004208`, ×0.1 A) |
| 6 | 0x10 (bits) | MOSFET status (Bit0: OV_active, Bit1: UV_active) |
| 7 | computed | Charge/discharge direction: 0=idle, 1=charging (>5), 2=discharging (<-5) |

**Msg 2 — Cell extremes & temperature (`0x1822AA0x`):**

| Byte | Struct offset | Content | Scale |
|------|---------------|---------|-------|
| 0–1 | 0x46 | **max_cell_volt** | mV |
| 2–3 | 0x48 | **min_cell_volt** | mV |
| 4–5 | 0x12 | **max_ntc** | °C |
| 6–7 | 0x14 | **min_ntc** | °C |

**Msg 3 — Protection & version (`0x1823AA0x`):**

| Byte | Struct offset | Content |
|------|---------------|---------|
| 0–1 | 0x56 | **protect1** (bitmask, see section 5.2) |
| 2–3 | 0x58 | **protect2** (bitmask, see section 5.3) |
| 4–5 | 0x5A | **warn1** (warning bitmask, `DAT_20004210`) |
| 6–7 | 0x54 | **version** (e.g. 0x499 = v117.7) |

**Msg 4–7 — 16 cell voltages (`0x1830AA0x` – `0x1833AA0x`):**

| CAN ID | Byte 0–1 | Byte 2–3 | Byte 4–5 | Byte 6–7 |
|--------|----------|----------|----------|----------|
| `0x1830` | **Cell[0]** (0x26) | **Cell[1]** (0x28) | **Cell[2]** (0x2A) | **Cell[3]** (0x2C) |
| `0x1831` | **Cell[4]** (0x2E) | **Cell[5]** (0x30) | **Cell[6]** (0x32) | **Cell[7]** (0x34) |
| `0x1832` | **Cell[8]** (0x36) | **Cell[9]** (0x38) | **Cell[10]** (0x3A) | **Cell[11]** (0x3C) |
| `0x1833` | **Cell[12]** (0x3E) | **Cell[13]** (0x40) | **Cell[14]** (0x42) | **Cell[15]** (0x44) |

All cell voltages in **mV** (scale ×0.001 for volts).

**Msg 8 — NTC temperatures (`0x1840AA0x`):**

| Byte | Struct offset | Content |
|------|---------------|---------|
| 0–1 | 0x16 | **ntc[0]** (Cell NTC 0) |
| 2–3 | 0x18 | **ntc[1]** (Cell NTC 1) |
| 4–5 | 0x1A | **ntc[2]** (Cell NTC 2) |
| 6–7 | 0x1C | **ntc[3]** (Cell NTC 3) |

**Msg 9 — Extended temperatures (`0x1841AA0x`):**

| Byte | Struct offset | Content |
|------|---------------|---------|
| 0–1 | 0x4E | **avg_temp** (average) |
| 2–3 | 0x22 | **env_ntc** (ambient) |
| 4–5 | 0x20 | **mos_ntc** (MOSFET) |
| 6–7 | — | (padding, 0) |

#### Multi-pack addressing

| Parameter | SRAM | Description |
|---|---|---|
| `DAT_200028DB` | pack_count | Number of **additional** packs (0 = master only) |
| `_DAT_200028DC` | pack_online_mask | Bitmask: bit n = pack n online |
| Struct base | `0x200041B6 + (pack_id - 1) × 0x60` | Per-pack struct (1-based) |
| Max packs | 9 | `if (param_1 != 0 && param_1 < 10)` |
| Broadcast | `0xFF` | Sends for all packs |

**Pack online watchdog** (from `Pack_Online_Watchdog`):
- For each pack (1..pack_count): timer at `0x20004B20 + pack × 2`
- Timeout: 19,999 ms (~20 s) → pack is marked offline
- `_DAT_200028DC &= ~(1 << pack)` → bit cleared
- `*(0x200041B7 + pack × 0x60) = 0` → online flag in struct cleared

#### Mapping per-pack CAN → Modbus registers

The Control MCU receives the per-pack messages and maps them to Modbus registers
following the scheme `34000 + pack_index × 100 + field_offset`:

| CAN Msg | CAN ID Pack 1 | Modbus Pack 1 | Modbus Pack 2 | Content |
|---|---|---|---|---|
| Msg 0 B0-1 | `0x1820AA01` | **34000** | **34100** | bat_volt |
| Msg 0 B2-3 | | **34001** | **34101** | bat_curr |
| Msg 0 B4-5 | | **34002** | **34102** | bat_soc |
| Msg 0 B6-7 | | **34003** | **34103** | cycle_count |
| Msg 1 (undoc.) | `0x1821AA01` | **34004** | **34104** | charge_status (charge/discharge flag, see correction §6.0.1) |
| Msg 2 B0-1 | `0x1822AA01` | **34005** | **34105** | max_cell |
| Msg 2 B2-3 | | **34006** | **34106** | min_cell |
| Msg 2 B4-5 | | **34007** | **34107** | max_ntc |
| Msg 2 B6-7 | | *(no register)* | | min_ntc |
| Msg 3 B0-1 | `0x1823AA01` | **34008** | **34108** | protect1 |
| Msg 3 B2-3 | | **34009** | **34109** | protect2 |
| Msg 3 B6-7 | | **34010** | **34110** | version |
| Msg 4 B0-1 | `0x1830AA01` | **34018** | **34118** | cell[0] |
| Msg 4 B2-3 | | **34019** | **34119** | cell[1] |
| ... | | ... | ... | ... |
| Msg 7 B6-7 | `0x1833AA01` | **34033** | **34133** | cell[15] |

> **Correction (2026-07-10):** The offset was wrongly 34011–34026 (see note in §6.0.1). Registers
> 34011–34017 are instead the NTC temperature values, see corrected table above.

---

## 7. RS485 Protocol (fully decoded)

Dispatcher: `RS485_Dispatcher` (1,614 bytes, 346 lines). Called by the RX handlers
of both RS485 UARTs (USART2 `0x40004400` = main bus, UART4 `0x40004C00` = inter-pack,
see §7.3) — **Correction 2026-07-10:** it is NOT bound to USART1/`0x40013000`; according to
Ghidra this address is unambiguously SPI1 (KA495XX interface, see §8.1), not USART1.

### 7.1 Frame format

```
┌──────┬────────┬─────────┬──────────┬──────────────┬──────────┐
│ Sync │ Length │ Command │ Address  │ Data[0..N-1] │ Checksum │
│ 's'  │ u8     │ u8      │ u8       │ var          │ u8       │
│ 0x73 │ 4-1024 │ 0x00-   │ PackID / │              │ XOR/Sum  │
│      │        │  0x34   │ 0xFF=BC  │              │          │
└──────┴────────┴─────────┴──────────┴──────────────┴──────────┘
```

- **Sync:** `'s'` (0x73) — fixed
- **Length:** total length (4–1024 bytes)
- **Command:** command ID (0x00–0x34)
- **Address:** pack address (`DAT_200041B6`), `0xFF` = broadcast, `0xFE` = broadcast type 2
- **Checksum:** verified via `LIB_XOR_Checksum` (last byte = computed value)

Response frame via `RS485_Send_Response(port, source_addr, cmd, data_ptr, data_len)`.

### 7.2 Command table

#### Status queries (read)

| CMD | Hex | Response len | Content | Description |
|-----|-----|-------------|--------|-------------|
| 0 | 0x00 | 6 B | GPIO status + `_DAT_200040D8` (u32) | MOSFET/hardware status |
| 1 | 0x01 | 1 B | `0x41` ('A') | Alive ping (ACK) |
| 7 | 0x07 | 1 B | `0x00` | Simple status (always 0) |
| 8 | 0x08 | 1 B | GPIO GPIOC Bit 13 | Read pin state |
| 9 | 0x09 | 6 B | `DAT_200028C1`–`C6` | **Protection status bytes** (6 bytes!) |

#### Battery data (read)

| CMD | Hex | Response len | Content | Description |
|-----|-----|-------------|--------|-------------|
| 24 | 0x18 | 4 B | `DAT_20004070` | BMIC raw data |
| 25 | 0x19 | 34 B | 32B cell voltages + 2B status | **16 cell voltages** (u16 × 16, mV) |
| 26 | 0x1A | 14 B | 5×NTC÷10 + MOS÷10 + ENV÷10 | **All temperatures** (7 values, ÷10→°C) |
| 27 | 0x1B | 1 B | `SOC ÷ 10` (0–100) | **Quick SOC** (1 byte!) |
| 44 | 0x2C | 10 B | `DAT_20003C64` (3×u32 + u16) | Energy counter data |

#### Pack data transfer (bidirectional)

| CMD | Hex | Len | Content | Description |
|-----|-----|-----|--------|-------------|
| 41 | 0x29 | **97 B** | Addr + 96B per-pack struct | **Full struct transfer** between packs |
| 46 | 0x2E | **85 B** | Addr + 84B detail data | **Per-pack detail dump** (via `RS485_Pack_Telemetry_Response`) |
| 48 | 0x30 | **45 B** | Addr + 44B aggregated CAN data | **CAN TX data dump** (from `0x20004A0C`) |

> **CMD 0x29** is the centerpiece of inter-pack communication: the master BMS
> receives the complete 96-byte struct of each slave pack and stores it at
> `DAT_200041B6 + (pack_id - 1) × 0x60`. This way the master always has a current
> image of all connected packs.

> **CMD 0x30** sends the aggregated CAN TX data (44 bytes from `0x20004A0C`)
> — presumably the response that the Control MCU polls periodically.

#### Configuration (read/write)

| CMD | Hex | R/W | Content | Description |
|-----|-----|-----|--------|-------------|
| 15 | 0x0F | R/W | `DAT_200028BC` (1 B) | Read/write config byte |
| 30 | 0x1E | Write | 2B → `DAT_200049F2` | Write parameter |
| 31 | 0x1F | Read | 2B ← `DAT_200049F2` | Read parameter |
| 32 | 0x20 | Write | 2B → `DAT_20004A06` | Write parameter |
| 33 | 0x21 | Read | 2B ← `DAT_20004A06` | Read parameter |
| 36 | 0x24 | Write | 1B SOC (0–100) | **Set SOC manually** |
| 37 | 0x25 | Write | 1B DOD value | Set DOD (only if `DAT_200028CA == 1`) |
| 45 | 0x2D | Write | 10B → `DAT_20003C64` | Write energy counter |
| 49 | 0x31 | Write | 1B → mode set | Set mode (relay to another port) |

#### Control & system

| CMD | Hex | Description |
|-----|-----|-------------|
| 2 | 0x02 | **HARD RESET** (DSB + infinite loop → watchdog reset) |
| 5 | 0x05 | FW update init (magic `0xA1` + `0xB1` at bytes 4-5) |
| 6 | 0x06 | Multi-pack state reset (master only, Addr=1) |
| 10 | 0x0A | Flash operation: 1=FW update, otherwise=data save |
| 14 | 0x0E | Assign pack address |
| 52 | 0x34 | Hardware test: GPIO toggle + 80ms delay |

### 7.3 UART assignment

| Peripheral | SRAM handle | Function | Baud rate |
|---|---|---|---|
| USART1 (`0x40013800`) | — | **Debug shell** (letter-shell) | — |
| USART2 (`0x40004400`) | `0x20002BC4` | **RS485 main bus** (→ Micro/inverter MCU) | 9600 baud |
| UART4 (`0x40004C00`) | `0x200033EC` | **RS485 inter-pack** (→ BMS slaves) | 9600 baud |

> **Correction (2026-07-10, Ghidra-verified):** USART1 was wrongly documented as `0x40013000` —
> that is demonstrably the SPI1 base address (see §8.1, KA495XX driver: `SPI1_Init`, `HAL_SPI1_TransmitReceive_Byte`
> etc. reference `0x40013000` explicitly and exclusively). The generic USART init code
> (`USART_Init_Config`/`USART_SPI_Clock_Reset`) accepts `0x40013800` as a valid USART base address —
> that is the correct USART1 address (standard STM32 peripheral layout, also matches USART1=`0x40013800`
> in the Micro inverter FW, see `VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analysis_vd_inv_app_0116.md`).

> **Correction:** `0x40004C00` is **UART4** on STM32F4xx, not USART3 as previously documented.
> The UART4 RX handler (`UART4_RX_Process`) checks for the 's' marker and dispatches to `RS485_Dispatcher(3)`.

### 7.4 Inter-pack communication (master↔slave)

```
        USART2 (RS485 main bus, 9600 baud)
Master BMS ◄══════════════════►  Micro MCU (inverter)
(Pack 1)                            │
  │                                 └──► Control MCU (→ Modbus TCP)
  │
  │  CAN bus (500 kbit/s)
  ├──────────────► Micro MCU (parallel to RS485)
  │
  │  UART4 (RS485 inter-pack, 9600 baud)
  ├══════════════════► Slave BMS Pack 2
  ├══════════════════► Slave BMS Pack 3
  ├══════════════════► ...
  └══════════════════► Slave BMS Pack 7
```

**Flow:**
1. Master BMS polls slave packs via RS485 CMD `0x29` (→ 96-byte struct)
2. Master aggregates all pack data in `0x20004Axx` (CAN TX buffer)
3. Master sends via CAN (PF 1-4 aggregated + per-pack Msg 0-9)
4. Micro MCU forwards to the Control MCU (telemetry block)
5. Control MCU exposes it as Modbus registers

| Variable | Description |
|---|---|
| `g_u8UpgradeFlag` | FW upgrade active |
| `g_u8UpdateFinishFlag` | Update finished |
| `g_u8ProhibitCommFlag` | Communication blocked during update |
| `u8FactoryFlag` | Factory mode flag |
| `Force Index:%d` | Forced update index |

---

## 8. KA495XX SPI Driver (fully decoded)

### 8.1 Hardware connection

| Parameter | Value |
|---|---|
| SPI peripheral | SPI1 (`0x40013000`) |
| Chip select | GPIOB pin 4 (software CS) |
| Chip type | "KA495XX" (Korea Analog Semiconductor) — exact type can only be determined from the PCB |
| RTOS task | `RTOS_vTaskKA495XX` → `RTOS_vTaskKA495XX` → `KA495XX_Main_StateMachine` (state machine) |

### 8.2 SPI protocol

**Single-register read** (`KA495XX_SPI_Read_Single_Reg`, 230B):

```
TX: [0xE1] [(reg & 0x7F) << 1] [CRC]    (3 bytes)
RX: [Status] [Data_Hi] [Data_Lo]          (3 bytes, 16-bit value)
```

- Command byte `0xE1` = read command (constant)
- Addressing: 7-bit register address, left-shifted by 1 bit
- CRC is computed over command + address byte
- CS (GPIOB.4): low before TX, high after RX

**Bulk register read** (`KA495XX_SPI_Bulk_Read`, 314B):

```
TX: [0xE1] [(start_reg & 0x7F) << 1 | 0x01] [CRC] [0x00...]    (4+ bytes)
RX: [N × 2 bytes data] [CRC]
```

- Bit 0 of the address byte = burst flag
- Typical call: `(1, 0x59)` = starting at register 1, read 89 registers (178 data bytes)

### 8.3 Register map (partially decoded)

From `KA495XX_Read_CellVoltages_And_Temps` (674B) — cell/temperature/current readout:

| Register | Content | Count | Remark |
|---|---|---|---|
| 0x1C | Status/WDT | 1 | Bit 7 = conversion-complete flag |
| 0x28–0x37 | Cell voltages | 16 | cell_volt[0..15], 16-bit each |
| 0x3E | Temperature sensor 1 | 1 | NTC |
| 0x44 | Temperature sensor 2 | 1 | NTC |
| 0x47 | Pack current | 1 | Coulomb counting input |
| 0x49 | Temperature sensor 3 | 1 | NTC |
| 0x4A | Temperature sensor 4 | 1 | NTC |
| 0x4B | Temperature sensor 5 | 1 | NTC |
| 0x58–0x59 | Current calibration | 2 | Offset/gain? |

### 8.4 Task state machine (15 states)

`KA495XX_Main_StateMachine` (186B) — main loop, called from `RTOS_vTaskKA495XX`:

| State | Action | Function |
|---|---|---|
| 0 | Init + status clear | `APP_Clear_SOC_Accumulator_Arrays` + `KA495XX_ClearMinMaxArrays` + `KA495XX_Check_And_Clear_Watchdog` (Reg 0x1C WDT) |
| 1, 3, 5, 7, 9, 11 | Start ADC conversion | `KA495XX_ADC_Conversion_Trigger` (trigger + wait for conversion complete) |
| 2, 4, 6, 8, 10, 12 | Cell readout | `KA495XX_Read_CellVoltages_And_Temps` (16 cell voltages + 5 temperatures + current) |
| 13 | Data processing | `KA495XX_Compute_ADC_Offsets` + `KA495XX_Scale_To_Millivolts` + `KA495XX_Compute_Power_And_Energy` + `NTC_Convert_Active_Sensors` → averages, then `PerPack_Struct_Builder` |
| 14 | Maintenance | `KA495XX_Read_Status_And_Check_Balancing` (bulk dump 89 reg) + `KA495XX_Read_Fault_Status` (balancing) + diagnostics |
| Default | Reset | 100ms delay (`LIB_vTaskDelayWithAssert`), back to state 0 |

**6 ADC cycles per pass** (states 1–12): each cycle triggers an ADC conversion and reads the results. The 6 samples are averaged in state 13 — effective noise suppression.

**Cycle time:** ~600ms ADC + processing + 100ms delay ≈ **~700ms per complete measurement cycle**.

---

## 8b. BMS OTA Firmware Update (decoded)

### Trigger sequence

```
1. CAN RX (ID 0x1801FFAA)
   └── CAN_RX_Handler (CAN_RX_Handler, line 52-59)
       └── Sets DAT_20002a72 (dead flag) → OTA readiness

2. RS485 CMD 0x05 (magic bytes)
   └── RS485_Dispatcher (lines 111-122)
       ├── Byte[4] == 0xA1 → sets DAT_20002831 (OTA unlock)
       └── Byte[4] == 0xB1 → alternative OTA unlock

3. OTA precondition manager (OTA_Precondition_Manager, 342B)
   └── State machine via DAT_200028ad
       └── Checks unlock flag, prepares flash
```

### Flash writer (`BMS_OTA_Flash_Writer`, 910B)

| Sub-CMD | Function | Details |
|---|---|---|
| 0 | **Init** | Flash unlock, erase sector starting at `0x0801E000` |
| 1 | **Data** | Write 8 bytes per packet (double-word aligned) |
| 2 | **Verify** | CRC/checksum over the written area |

**Target address:** `0x0801E000` — lies **outside** the current app area (`0x08000000–0x08019FFF`), presumably a staging area for a new bootloader or second stage.

> **Security-relevant:** OTA requires both the CAN trigger (dead flag) and the RS485 magic bytes — dual-channel authentication.

---

## 9. Debug Shell (UART)

Same letter-shell library as the Micro FW (with "default user", password presumably empty).

### Shell commands

| Command | Description |
|---|---|
| `bat_data` | Print BMS data (`BMS_Data_Printf`) |
| `set_soc` | Set SOC manually (`Set_BMS_Soc`) |
| `set_dod` | Set depth of discharge (`Set_BMS_DOD`) |
| `checkout_rs485` | Configure RS485 mode (`Set_Rs485_Mode`) |
| `force` | Forced action |
| `clear` | Clear console |
| `help` | Show command list |

### Debug output blocks

**Block 1 — Online status:**
```
Online:%d
Version:%d
```

**Block 2 — Pack overview:**
```
BBat Volt:%0.01fV
Bat Curr:%0.1fA
Bat Soc:%.1f
Cyc Cnt:%d
Chg Mos:%d  Dsg Mos:%d
Max NTC:%d  Min NTC:%d Ave NTC:%d
MOS NTC:%d  ENV NTC:%d
Max Cell:%d  Min Cell:%d
Cell NTC: [16 values]
Cell Volt: [16 values]
Protect1:%d  Protect2:%d
```

**Block 3 — SOC details:**
```
Max Vol:%d  Min Vol:%d
Ave Temp:%d  Max Temp:%d  Min Temp:%d
Current:%.1f
Tal Vol:%.1f
FMax Soc:%lf  Min Soc:%lf
Real Soc:%lf  Tar Soc:%lf  Show Soc:%lf
Full Flag:%d
Soh:%.1f
Total Cap:%.1f kwh
Max Chg Cur:%.1f  Max Dsg Cur:%.1f
Cycle Count:%d
```

---

## 10. Relevance for Modbus registers (34000–34600)

The BMS data is sent via the per-pack CAN protocol (section 6.5, 10 messages per pack)
to the Micro MCU and from there to the Control MCU, which exposes it as Modbus registers.

### 10.1 Confirmed register mapping (from live scan + FW analysis)

| CAN Msg | Struct offset | BMS variable | Pack 1 | Pack 2 | Scale | Confirmed? |
|---|---|---|---|---|---|---|
| Msg 0 B0-1 | 0x4A | bat_voltage | **34000** | **34100** | ×0.01→V | ✅ Scan: 5114=51.14V |
| Msg 0 B2-3 | 0x4C | bat_current | **34001** | **34101** | ×0.1→A | ✅ Scan: 0 |
| Msg 0 B4-5 | 0x0E | **show_soc** | **34002** | **34102** | **×0.1→%** | ✅ Scan: 146=14.6% |
| Msg 0 B6-7 | 0x24 | cycle_count | **34003** | **34103** | ×1 | ✅ Scan: 19 |
| Msg 1 B6-7 (undoc.) | — | charge_status | **34004** | **34104** | ×1 | ✅ Confirmed (live scan CSV: "=3 during active charging, =0 when idle") |
| Msg 2 B0-1 | 0x46 | max_cell_volt | **34005** | **34105** | ×0.001→V | ✅ Confirmed (live scan CSV) |
| Msg 2 B2-3 | 0x48 | min_cell_volt | **34006** | **34106** | ×0.001→V | ✅ Confirmed (live scan CSV) |
| Msg 2 B4-5 | 0x12 | max_ntc | **34007** | **34107** | ×1→°C | ✅ Confirmed (live scan CSV, value constant 0) |
| Msg 2 B6-7 | 0x14 | min_ntc | *(no Modbus register)* | | ×1→°C | — |
| Msg 3 B0-1 | 0x56 | protect1 | **34008** | **34108** | Bitmask | ✅ Confirmed (live scan CSV) |
| Msg 3 B2-3 | 0x58 | protect2 | **34009** | **34109** | Bitmask | ✅ Confirmed (live scan CSV + discharge test) |
| Msg 3 B6-7 | 0x54 | bms_version | **34010** | **34110** | ×1 | ✅ Confirmed (v116/v117.7) |
| Msg 4-7 | 0x26-0x44 | cell_volt[0–15] | **34018–34033** | **34118–34133** | ×0.001→V | ✅ Confirmed (live scan CSV, corrected 2026-07-10) |
| Msg 8 | 0x16-0x1C | ntc[0–3] | **34011–34014** | **34111–34114** | ÷10→°C | ✅ Confirmed (live scan CSV) |
| Msg 9 B0-1 | 0x4E | avg_temp | **34017** | **34117** | ÷10→°C | ✅ Confirmed (live scan CSV) |
| Msg 9 B2-3 | 0x22 | env_ntc | **34016** | **34116** | ÷10→°C | ✅ Confirmed (live scan CSV) |
| Msg 9 B4-5 | 0x20 | mos_ntc | **34015** | **34115** | ÷10→°C | ✅ Confirmed (live scan CSV) |

> **Correction (2026-07-10):** The cell voltage offset was wrongly 34011–34026 (off-by-7 error, confused
> with the NTC block). The register mapping is now corrected based on `Modbus_RS485_TCP/VNSD_Register_Map_Final_generated.csv`
> (✅ FW-verified) and the previously open NTC registers (Msg 8/9) have been added.
>
> **Correction 2 (2026-07-10, via Ghidra decompilation of `CAN_TX_PerPack_10Msgs`, `0x08005af0`,
> verified against the live scan CSV):** 34004–34007 were also shifted by 1 (max_cell_volt
> wrongly at 34004 instead of 34005, etc.). Cause: between Msg 0 and Msg 2 the BMS CAN stack sends
> a previously undocumented **Msg 1** (`0x1821AAxx`) containing, among other things, a charge/discharge flag computed from `bat_current` —
> this occupies register 34004, exactly matching the live scan finding for
> `pack1_charge_status` (CSV note: "=3 exactly when the pack is actively charging, =0 when idle. Not voltage delta!").
> Msg 1 has 3 more fields that are not mapped to Modbus (struct offsets 0x50, 0x52, 0x10).
> Their content has since been resolved — 0x50 = max_charge_current, 0x52 = max_discharge_current
> (see addendum in §6.0) — but they still have no Modbus register of their own. For "min_ntc" (Msg 2 byte 6-7, offset 0x14) no separate register exists according to the
> CSV — 34008/34009 are clearly protect1/protect2 (Msg 3), no further shift there.

### 10.2 SOC conversion chain (confirmed)

```
BMS: double real_soc (64-bit FP, sub-0.01% precision)
  → u32 ÷10000 (4 decimal places internally)
  → smoothing → u32 show_soc ÷10000
  → cap at 100.0% (if >99.4% AND full_flag)
  → u16 ÷10 (for CAN TX, value e.g. 146 = 14.6%)
  → CAN Msg 0 bytes 4-5 (PF=1 aggregated + per-pack)
  → Micro MCU → Control MCU
  → Modbus register 34002 (scale ×0.1)
```

### 10.3 Version note

The version hardcoded in the binary is `0x499` = **1177** (decimal), which corresponds to **v117.7**.
The file name version v177.7 deviates from this — it was a typo/guessing error in
the original naming, not a different firmware.

> **Note (original):** Version v117.7 is newer than the v116 that appears in the live scan.
> Register 34010 shows the hardcoded value of the actually installed BMS FW.

> **✅ Confirmed 2026-07-09:** After the BMS OTA update on the live device, the Modbus scan
> (`nach_bms_upgrade_auf_117_7_fw.csv` (local, raw data)) shows exactly the value predicted here: register 30204
> (bms_version), 34010/34110/34210/34310/34410/34510 (pack1–6_bms_version) and 37012 (mirror)
> now all read **1177 = v117.7** — identical to the hardcoded `0x499` in this binary.
> This confirms beyond doubt: this analyzed binary (`20251010135647565eb2036.bin`)
> is exactly the firmware that was rolled out via OTA, and the correct version status is
> **v117.7** (not v177.7). Side finding: the version register apparently encodes from the first
> decimal place onward as raw value×10 (previously, with the integer v116, the raw value was simply 116) —
> the same pattern as with ems_version (30200) at the jump to Control FW v149.2 (raw value 1492).

---

## 11. Open questions

### Resolved in this session:

| # | Topic | Status |
|---|---|---|
| ~~1~~ | CAN protocol | ✅ TX (PF 1-4) + RX + per-pack (10 msgs) fully decoded |
| ~~2~~ | SOC algorithm | ✅ Coulomb counting + OCV calibration + show_soc smoothing |
| ~~3~~ | Protect1/Protect2 bitmasks | ✅ 12+5 bits decoded, setter functions verified, thresholds identified |
| ~~4~~ | Multi-pack addressing | ✅ CAN ID `0x18[MsgGrp]AA[PackID]`, max 9 packs, 20 s watchdog |
| ~~5~~ | KA495XX / cell data struct | ✅ 96-byte struct with 22 fields, scale factors confirmed |
| ~~6~~ | **RS485 protocol** | ✅ 34 commands decoded, frame format, inter-pack architecture |
| ~~7~~ | KA495XX SPI driver | ✅ SPI1 protocol (0xE1 cmd, 7-bit addr, CRC), register map partial, 15-state task machine |
| ~~8~~ | C3 bit 3 setter | ✅ Dead code — no setter in entire FW, Protect2[11] never activated |
| ~~9~~ | C1 bits 5/6 setter | ✅ Dead code — never set by software. HW OVP/UVP presumably directly via GPIO |
| ~~10~~ | FW update sequence | ✅ CAN 0x1801FFAA (dead flag) + RS485 CMD 0x05 (0xA1/0xB1 magic) + flash writer 0x0801E000 |
| ~~11~~ | UART4 (formerly "USART3") | ✅ Inter-pack RS485 bus for BMS slave communication, 9600 baud, UART4 (not USART3) |

### Still open:

| # | Topic | Priority | Next step |
|---|---|---|---|
| 1 | **Shell password** | Low | Presumably empty (same letter-shell as Micro FW), test UART — **blocked, requires live device** |
| 2 | **SOC OCV lookup table** | Medium | Flash `0x0801B79C`/`0x0801B832` → extract voltage curve — requires flash dump (binary ends at `0x08019FFF`) — **blocked, requires physical flash dump** |
| 3 | **KA495XX exact chip type** | Low | Determinable only by PCB inspection or SPI sniffing — fully analyzed on the FW side — **blocked, rechecked 2026-07-10 (see 11a), no further static lever** |

---

## 11a. SRAM 0x2000493C–0x200049B4 — Register function codes (addendum 2026-07-10)

Static Ghidra analysis to resolve the item from `AES_Crypto_Stack_Analysis.md` (local, not in the repo). **Correction of the original assumption:** the region is not a BMIC/SPI register code block, but part of a larger **RS485 protection parameter configuration block** (`0x20004938`–`0x20004A0C`, 212 bytes in total), addressed via RS485 commands **0x11 (read)** / **0x12 (write)** with a group ID bitmask in the frame header (`[GroupCode(u16)][GroupCode repeated][data...]`). Persisted in **EEPROM offset 600** (212 bytes).

**Core functions:**
- `RS485_Register_Read_Handler` (0x08007778) / `RS485_Register_Write_Handler` (0x08007d24) — dispatch by group ID, after setting, the write path calls `LIB_Checksum_Ones_Complement` + `I2C_EEPROM_Write_WithMutex(&DAT_20004938, 600, 0xd4)`
- `BMS_Config_Params_Factory_Reset` (0x08008120) — default values of all groups
- `BMS_Config_Params_Load_Or_Default` (0x08007460) — EEPROM load at boot
- `Voltage_Temp_Protection_Checker` (0x080098b8) — main consumer of the temperature groups

| GroupID | SRAM range | Meaning (confidence) |
|---|---|---|
| `0x04` | `0x2000493C–0x20004952` | **High.** ENV NTC temperature protection: OT_L1 (60.0°C/3000ms), OT_L2 (65.0°C), OT_Recovery (60.0°C), UT_L1 (-25.0°C), UT_L2 (-30.0°C), UT_Recovery (-25.0°C) |
| `0x08` | `0x20004954–0x2000495E` | **High.** MOS NTC temperature protection: Trigger1 (90.0°C/3000), Trigger2 (115.0°C/3000), Recovery (85.0°C/3000) |
| `0x01` | `0x20004960–0x20004976` | **High.** Cell temperature protection, discharge direction (6 trigger/delay pairs, OT/UT L1/L2 + recovery) |
| `0x02` | `0x20004978–0x2000498E` | **High.** Cell temperature protection, charge direction (symmetric to group 0x01) |
| `0x40` | `0x20004996–0x200049A4` | **High (resolved 2026-07-14).** Cell overvoltage protection. No checker in v117.7 itself (confirmed: the constant 3800mV does not occur anywhere in 117.7) — **a consumer exists only from BMS FW v118** (`FUN_0800a584`, see `VNSD_BMS_FW_Analysis_v118.md` section 4.6), temperature-compensated there (thresholds depend on the sign of the Ave NTC) |
| `0x10` | `0x200049A6–0x200049B4` | **High (resolved 2026-07-14).** Cell undervoltage protection — same finding as group `0x40`: consumer only from v118, demonstrably unused in 117.7 |
| `0x80` | 1×u16 | **Low.** No write handler branch — presumably status/reserve |
| `0x100` | `0x200049B6–0x200049C4` | **High (resolved 2026-07-14).** Not balance/SOC, but **charge overcurrent stages** — the consumer is the already known `Current_Protection_Checker` (0x08008420), present unchanged in v118 |
| `0x200` | `0x200049C6–0x200049D4` | **High (resolved 2026-07-14).** **Discharge overcurrent stages**, counterpart to `0x100`, likewise `Current_Protection_Checker` |
| `0x400` | `0x200049D6–0x200049DA` | **Low.** 3 values, too small for a reliable assignment — still without an identified consumer in v118 as well |

**BMIC_Info struct:** exists only as a debug print string (`" BMIC_Info.lBatPackFastCur_mA:%d\r\n"` @ 0x08013227, `"BMIC_Info.lBatPackCur_100uA:%d\r\n"` @ 0x0801324c), no formal Ghidra structure (0 structures in the image — RVDS release build without DWARF). The referenced fields (`0x2000406c`, `0x20004070`) are written by `KA495XX_Compute_Power_And_Energy` (0x08015360). The printing function `UndefinedFunction_08013124` (caller: `Debug_Print_Device_UIDs` 0x080130f4) is still unnamed.

**Conclusion (updated 2026-07-14 during the BMS v118 diff):** All groups except `0x80`/`0x400` are now resolved with high confidence. Groups 0x01/0x02/0x04/0x08 (temperature protection) were already actively consumed in 117.7. Groups 0x100/0x200 are not balance/SOC values, but charge/discharge overcurrent stages (`Current_Protection_Checker`, unchanged since 117.7). Groups 0x40/0x10 (cell OVP/UVP) actually had **no** consumer in 117.7 — the logic for them (`FUN_0800a584`, temperature-compensated) was only retrofitted with **BMS FW v118**, see `VNSD_BMS_FW_Analysis_v118.md` section 4.6. `0x80`/`0x400` remain unresolved (too little data / no consumer in either version).

**Side finding (2026-07-14):** The previously unnamed `UndefinedFunction_08013124` (caller: `Debug_Print_Device_UIDs`, 0x080130f4) outputs a debug status dump (`g_u8ProhibitCommFlag`, `g_u8UpgradeFlag`, `g_u8UpdateFinishFlag`, BMIC currents, factory flag, `g_u8EmsChgDsgFlg`). In Ghidra it existed only as a temporary undefined-function stub (1 byte) — formally created with `create-function` (correct size: 0x08013124–0x080131af, 140 bytes) and named **`Debug_Print_System_Status_Flags`** (duplicate check via name similarity search negative). Belongs to the debug cluster that was removed completely in v118 (see `VNSD_BMS_FW_Analysis_v118.md` section 3.2) — noted and named here anyway for 117.7 completeness.

---

## 12. Complete function list (540/540 — 100%)

Status: 06.07.2026

### SoftFloat — software floating point (15 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08000290` | `SoftFloat_Uint64_Divide` | 98B | 64-bit unsigned division |
| `0x080002f2` | `SoftFloat_Int64_Divide` | 98B | 64-bit signed division (wrapper) |
| `0x08000420` | `__adddf3` | 322B | IEEE 754 double addition |
| `0x08000562` | `SoftFloat_Add_Double` | 6B | Wrapper for __adddf3 |
| `0x0800056e` | `__muldf3` | 228B | IEEE 754 double multiplication |
| `0x08000652` | `__divdf3` | 252B | IEEE 754 double division |
| `0x08000730` | `SoftFloat_UInt_To_Double` | 26B | uint → double conversion |
| `0x0800074a` | `SoftFloat_Double_To_Int_Low` | 50B | double → int (lower part) |
| `0x0800077c` | `SoftFloat_Float_To_Double` | 38B | float → double promotion |
| `0x080007a4` | `SoftFloat_Negate_If_Positive` | 48B | Sign negation if positive |
| `0x08000800` | `SoftFloat_Shift_Left_64` | 30B | 64-bit left shift |
| `0x0800081e` | `SoftFloat_Shift_Right_64` | 32B | 64-bit right shift |
| `0x0800083e` | `SoftFloat_Arith_Shift_Right_64` | 36B | 64-bit arithmetic right shift |
| `0x08000862` | `SoftFloat_RoundToInt` | 60B | Round to integer |
| `0x080008bc` | `__aeabi_dmul_normalize` | 156B | double multiplication normalization |
| `0x08000958` | `SoftFloat_Double_To_Int_Store` | 48B | double → int with store |
| `0x080009ac` | `SoftFloat_Round_To_Even` | 18B | Banker's rounding |
| `0x080009be` | `SoftFloat_Pack_Float_Result` | 92B | Pack float result |
| `0x0800e46c` | `float_round_to_int` | 154B | Round float to int |

### LIB — library / utility (34 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08000354` | `LIB_Memcpy_Aligned` | 36B | Aligned memcpy |
| `0x08000378` | `LIB_Memset` | 14B | memset |
| `0x08000386` | `LIB_Memset_Zero` | 4B | memset(0) shortcut |
| `0x0800038a` | `LIB_Memset_ReturnDst` | 18B | memset returning dst |
| `0x0800039c` | `LIB_Strstr` | 36B | strstr |
| `0x080003c0` | `LIB_strncpy` | 24B | strncpy |
| `0x080003d8` | `LIB_Strlen` | 14B | strlen |
| `0x080003e6` | `LIB_strcmp` | 28B | strcmp |
| `0x080007d4` | `LIB_Unsigned_Divide` | 44B | Software division loop |
| `0x08000a1a` | `LIB_Decompress_RLE` | 86B | RLE decompression |
| `0x08001b58` | `LIB_XOR_Checksum` | 28B | XOR checksum |
| `0x0800898c` | `LIB_Checksum_Ones_Complement` | 32B | One's complement checksum |
| `0x0800adc0` | `LIB_CRC16_Modbus_Calc` | 56B | CRC16 Modbus calculation |
| `0x080101b4` | `LIB_Saturated_Add_Unsigned` | 44B | Saturated addition (0..1000000) |
| `0x080107e4` | `LIB_RingBuffer_Advance_WritePtr` | 42B | Advance ring buffer write pointer |
| `0x0801097e` | `LIB_Queue_Init_Static` | 42B | Initialize static queue |
| `0x08010b24` | `LIB_vPortFree_InsertBlock` | 96B | Insert heap block on free |
| `0x08010b8c` | `LIB_FreeRTOS_Timer_Check_Overflow` | 80B | Check timer overflow |
| `0x08010bfe` | `LIB_RingBuffer_Is_Empty` | 30B | Ring buffer empty? |
| `0x08011bba` | `LIB_HexChar_To_Nibble` | 54B | Hex character → 4-bit value |
| `0x08012568` | `LIB_String_Match_Length` | 44B | Prefix match length |
| `0x08012594` | `LIB_Strcpy_U16` | 24B | String copy with u16 limit |
| `0x08012712` | `LIB_Uint32_To_HexString` | 56B | uint32 → hex string |
| `0x08014258` | `LIB_Delay_Loop_Byte` | 30B | Delay loop (byte-based) |
| `0x080148c0` | `LIB_LinkedList_Remove_Node` | 40B | Remove doubly-linked list node |
| `0x080161cc` | `LIB_Delay_Loop_Short` | 28B | Short delay loop |
| `0x080161e8` | `LIB_LinkedList_Init` | 26B | Initialize linked list |
| `0x08016202` | `LIB_FreeRTOS_Clear_ListItem_Value` | 6B | Clear ListItem value |
| `0x08016208` | `LIB_LinkedList_Insert_Sorted` | 52B | Sorted insert |
| `0x0801623c` | `LIB_List_InsertBefore` | 24B | Insert before element |
| `0x08016254` | `LIB_vPortEnterCritical` | 94B | Enter critical section |
| `0x08016730` | `LIB_vTaskDelayWithAssert` | 100B | vTaskDelay with assert |
| `0x08016bf8` | `LIB_FreeRTOS_Inc_SchedulerSuspended` | 12B | Scheduler suspend counter++ |
| `0x08016fd8` | `LIB_xQueueCreateStatic` | 106B | Create static queue |
| `0x08017ac4` | `LIB_xTaskCreate` | 96B | Create FreeRTOS task |
| `0x0800d43c` | `LIB_TestBitMask_U16` | 26B | Bitmask test (u16) |
| `0x0800ed54` | `LIB_Printf_Pad_Spaces` | 36B | Printf: space padding |
| `0x0800ed78` | `LIB_Printf_Pad_Char` | 46B | Printf: character padding |

### HAL — hardware abstraction layer (75 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08000264` | `HAL_Get_CurrentIRQ_Number` | 6B | Read current IRQ number |
| `0x08000a70` | `HAL_UART_Clear_IDLEIE` | 10B | UART IDLE interrupt off |
| `0x08000a7a` | `HAL_UART_Set_Address_Mask` | 8B | Set UART address mask |
| `0x08000af0` | `GPIO_Set_AF_And_Pin_Mapping` | 200B | Configure AF pins |
| `0x08000bb8` | `HAL_Set_Pin_Bit0_With_Delay` | 36B | Set pin with delay |
| `0x08000bdc` | `HAL_GPIO_Set_AF_Mode` | 22B | GPIO alternate function mode |
| `0x08000bf2` | `HAL_SPI_Is_TX_Complete` | 26B | SPI TX complete? |
| `0x08000c0c` | `HAL_UART_Get_BRR_Low16` | 8B | UART baud rate lower 16 bits |
| `0x08000c14` | `HAL_Check_Bits_U32` | 18B | Bit check (u32) |
| `0x08000c26` | `HAL_DMA_Check_ISR_Flag` | 18B | Check DMA ISR flag |
| `0x08000c38` | `HAL_CAN_Init_Registers` | 80B | Initialize CAN registers |
| `0x08000c90` | `HAL_UART_Set_IDLEIE` | 10B | UART IDLE interrupt on |
| `0x08000c9a` | `HAL_SPI_Set_Error_Flag_And_Delay` | 28B | SPI error flag + delay |
| `0x08000d70` | `CAN_Build_Arbitration_ID` | 106B | Build CAN arbitration ID |
| `0x0800120c` | `HAL_DMA_Transfer_Single` | 66B | Single DMA transfer |
| `0x08001640` | `HAL_Get_DMA_Channel_Struct` | 42B | Get DMA channel struct |
| `0x0800167c` | `GPIO_Init_Generic` | 136B | Generic GPIO init |
| `0x08001720` | `GPIO_Init_All_Pins` | 312B | Initialize all GPIO pins |
| `0x08001870` | `GPIO_Init_Input_Floating` | 148B | Configure floating input |
| `0x08001920` | `HAL_GPIO_ReadInputBit_Wrapper` | 16B | Read GPIO input (wrapper) |
| `0x08001930` | `GPIO_Init_Output_PP_50MHz` | 148B | Push-pull output 50MHz |
| `0x080019e0` | `HAL_GPIO_ReadOutputBit_Wrapper` | 16B | Read GPIO output (wrapper) |
| `0x080019f0` | `HAL_GPIO_Write_Pin` | 30B | Set/clear GPIO pin |
| `0x08001a18` | `GPIO_DeInit_And_CAN_NVIC_Init` | 300B | GPIO DeInit + CAN NVIC |
| `0x08001f1c` | `HAL_Delay_Short_30Cycles` | 14B | ~30 cycle delay |
| `0x08002d04` | `HAL_TIM_OC_Reconfigure` | 52B | Reconfigure timer output compare |
| `0x08002e6c` | `HAL_I2C_Clock_Reset` | 50B | I2C clock reset |
| `0x08002ea4` | `HAL_Set_Bit16_U32` | 22B | Set bit 16 in u32 |
| `0x08002eba` | `HAL_Set_Clear_Bits_Offset14` | 18B | Set/clear bits (offset 14) |
| `0x08002fe0` | `HAL_I2C_Struct_Init_Default` | 32B | I2C struct default values |
| `0x08004d84` | `HAL_DBGMCU_APB1_FreezeBitSet` | 26B | Debug MCU APB1 freeze |
| `0x080051d4` | `HAL_OneWire_Read_Byte` | 106B | Read OneWire byte |
| `0x080062e0` | `HAL_GPIO_PortB_Reset_All` | 28B | Reset GPIO port B completely |
| `0x0800649c` | `HAL_FLASH_Set_CR_Bits` | 12B | Set flash CR register bits |
| `0x080064ac` | `HAL_Flash_Program_Word_2` | 78B | Program flash word (variant 2) |
| `0x08006500` | `HAL_Flash_Get_Status` | 76B | Query flash status |
| `0x08006550` | `HAL_FLASH_Enable_PrefetchBuffer` | 14B | Enable flash prefetch |
| `0x08006564` | `HAL_Flash_Program_Word` | 82B | Program flash word |
| `0x080065bc` | `HAL_FLASH_Write_KEY2` | 12B | Flash unlock KEY2 |
| `0x0800663c` | `HAL_NVIC_Set_Priority_Field` | 60B | Set NVIC priority |
| `0x0800667c` | `AFIO_Remap_Configure` | 554B | Configure AFIO remap |
| `0x080068b0` | `GPIO_Init_Pin` | 278B | Init single GPIO pin |
| `0x080069c6` | `HAL_GPIO_ReadInputBit` | 18B | Read GPIO input bit |
| `0x080069d8` | `HAL_GPIO_ReadOutputBit` | 18B | Read GPIO output bit |
| `0x08006a3c` | `HAL_Read_Unique_Device_ID` | 78B | Read STM32 unique ID |
| `0x0800b038` | `HAL_IWDG_Start` | 10B | Start watchdog |
| `0x0800b048` | `HAL_I2C_Check_StatusFlag` | 20B | Check I2C status flag |
| `0x0800b060` | `HAL_IWDG_Refresh` | 10B | Feed watchdog |
| `0x0800b02c` | `HAL_IWDG_Set_Prescaler` | 6B | Watchdog prescaler |
| `0x0800b070` | `HAL_IWDG_Set_Reload` | 6B | Watchdog reload value |
| `0x0800b07c` | `HAL_IWDG_Write_KR` | 6B | Watchdog key register |
| `0x0800bb0c` | `HAL_NVIC_EnableIRQ_WithPriority` | 100B | Enable IRQ with priority |
| `0x0800bb7c` | `HAL_NVIC_SystemReset` | 10B | Trigger system reset |
| `0x0800bb90` | `HAL_Enter_Sleep_Mode` | 62B | Enter sleep mode |
| `0x0800bcf8` | `HAL_RCC_DisablePLLSAI` | 24B | Disable PLL-SAI |
| `0x0800bd14` | `HAL_RCC_Set_PREDIV1` | 18B | Set RCC PREDIV1 |
| `0x0800bd2c` | `HAL_RCC_Set_ADC_Prescaler` | 30B | Set ADC prescaler |
| `0x0800bd50` | `HAL_RCC_Set_ADC_Prescaler_2` | 18B | ADC prescaler (variant 2) |
| `0x0800bd68` | `HAL_RCC_CSR_BitSet` | 26B | RCC CSR register bit |
| `0x0800bd88` | `HAL_RCC_CFGR2_BitSet` | 26B | RCC CFGR2 register bit |
| `0x0800bda8` | `HAL_RCC_BDCR_BitSet` | 26B | RCC BDCR register bit |
| `0x0800bdc8` | `HAL_RCC_AHBRSTR_BitSet` | 26B | RCC AHB reset bit |
| `0x0800bde8` | `HAL_RCC_APB1ENR_BitSet` | 26B | RCC APB1 clock enable |
| `0x0800be08` | `RCC_GetClockFrequencies` | 240B | Calculate clock frequencies |
| `0x0800bf14` | `HAL_RCC_Get_Flag_Status` | 56B | Read RCC flag status |
| `0x0800c22c` | `HAL_Set_Bit6_U16` | 24B | Set bit 6 in u16 |
| `0x0800c244` | `HAL_SPI_Reset_Peripheral` | 76B | Reset SPI peripheral |
| `0x0800c29c` | `HAL_SPI_Check_StatusFlag` | 18B | Check SPI status flag |
| `0x0800c2ae` | `HAL_SPI_Get_DR` | 6B | Read SPI data register |
| `0x0800c2b4` | `HAL_SPI_Set_DR` | 4B | Write SPI data register |
| `0x0800c2b8` | `HAL_DMA_Apply_Channel_Config` | 60B | Configure DMA channel |
| `0x0800c2f4` | `RCC_Clock_Init` | 218B | Initialize RCC clock tree |
| `0x0800cf92` | `HAL_TIM_Set_Clear_DIER_Bits` | 18B | Set/clear timer DIER bits |
| `0x0800d084` | `HAL_Set_Bit0_U32` | 22B | Set bit 0 in u32 |
| `0x0800d0a0` | `HAL_Check_IRQ_Flag_And_Enable` | 34B | Check IRQ flag and enable |
| `0x0800d0c4` | `TIM_Base_Init` | 314B | Timer base initialization |
| `0x0800d278` | `RS485_Send_Response` | 102B | Send RS485 response |
| `0x0800d2f0` | `HAL_TIM_Write_ARR_Inverted` | 18B | Write timer ARR inverted |
| `0x0800d302` | `HAL_NVIC_Set_Bit_By_IRQn` | 74B | NVIC bit by IRQ number |
| `0x0800d424` | `HAL_TIM_Set_Bit13_CR2` | 24B | Set timer CR2 bit 13 |
| `0x0800d456` | `HAL_Check_IRQ_Pending` | 84B | Check pending IRQ |
| `0x0800d798` | `HAL_Flash_Write_Config_Block` | 74B | Write config block to flash |
| `0x0800e3cc` | `CLI_Printf_Wrapper` | 22B | Printf wrapper for CLI |
| `0x0800e3ec` | `HAL_System_Reset_DSB_Loop` | 56B | System reset (DSB + loop) |
| `0x0800e42c` | `HAL_System_Reset_DSB_Loop2` | 56B | System reset variant 2 |
| `0x0800f31a` | `HAL_Delay_Us` | 30B | Microsecond delay |
| `0x08014240` | `HAL_GPIO_Read_Pin_PB6` | 20B | Read GPIO PB6 |
| `0x08015ffc` | `HAL_GPIO_Set_ChargeEnable_Pin` | 28B | Control charge enable pin |
| `0x080020a4` | `HAL_I2C_Bitbang_Start_Sequence` | 42B | I2C start condition (bitbang) |
| `0x0800254c` | `HAL_I2C_Bitbang_SCL_Low_Short` | 40B | I2C SCL low short |
| `0x0800257c` | `HAL_I2C_Bitbang_SCL_Low_Long` | 40B | I2C SCL low long |
| `0x08002000` | `HAL_SPI_BitBang_Write_Byte` | 98B | SPI bitbang send byte |
| `0x080026b4` | `HAL_SPI1_TransmitReceive_Byte_2` | 12B | SPI1 TX/RX (variant 2) |
| `0x080026c0` | `HAL_SPI1_TransmitReceive_Byte` | 54B | SPI1 TX/RX byte |
| `0x08005244` | `SPI_Bitbang_Send_Byte` | 114B | SPI bitbang send |
| `0x08002cbe` | `HAL_RingBuffer_Write` | 70B | Ring buffer write |

### INIT — initialization (18 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08001d8c` | `System_Init_Main` | 278B | Main system initialization |
| `0x08001eb8` | `INIT_All_Peripherals` | 40B | Initialize all peripherals |
| `0x080011e4` | `INIT_GPIO_SPI_CS_Pins` | 28B | SPI chip-select pins |
| `0x080014b4` | `INIT_I2C2_GPIO` | 30B | I2C2 GPIO pins |
| `0x08001f2c` | `INIT_I2C_EEPROM_GPIO` | 78B | I2C EEPROM GPIO pins |
| `0x080020d4` | `INIT_GPIO_PortC_Pins10_11` | 22B | Port C pins 10/11 |
| `0x08002500` | `INIT_GPIOD_SPI_Pins` | 70B | GPIO-D SPI pins |
| `0x080025c4` | `INIT_SPI1_GPIO_Pins` | 76B | SPI1 GPIO pins |
| `0x08002878` | `INIT_CAN_Filter_Config` | 102B | Configure CAN filter |
| `0x0800298c` | `INIT_RS485_UART_Structs` | 98B | RS485 UART structs |
| `0x08002ba0` | `INIT_USART_And_SPI_GPIO_Deinit` | 64B | USART/SPI GPIO DeInit |
| `0x08002cae` | `INIT_USART_All` | 16B | Initialize all USARTs |
| `0x0800afe4` | `INIT_EXTI_And_Timer_IRQ` | 70B | EXTI + timer IRQs |
| `0x08010abc` | `INIT_FreeRTOS_Timers` | 70B | FreeRTOS timer system |
| `0x080108dc` | `INIT_FreeRTOS_Heap_Metadata` | 98B | FreeRTOS heap metadata |
| `0x08001d28` | `INIT_TIM2_TIM3_NVIC_And_Reset` | 96B | TIM2/TIM3 NVIC + reset |
| `0x08005160` | `INIT_KA495XX_SPI_GPIO` | 40B | KA495XX SPI GPIO pins |
| `0x08015e14` | `INIT_KA495XX_Subsystem` | 74B | KA495XX subsystem init |

### FATFS — file system (16 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08000402` | `FATFS_Compare_Strings` | 30B | String comparison |
| `0x080125ac` | `FATFS_Dir_Entry_Match` | 266B | Match directory entry |
| `0x08012338` | `FATFS_DiskIO_Dispatch` | 174B | Disk I/O dispatcher |
| `0x080124a8` | `FATFS_PutNumeric` | 158B | Write numeric value |
| `0x08011f4c` | `FATFS_File_Init` | 142B | Initialize file |
| `0x080123e6` | `FATFS_Find_Dir_Entry` | 122B | Find directory entry |
| `0x08011ed2` | `FATFS_Dir_Entry_Advance` | 122B | To next entry |
| `0x0801182c` | `FATFS_Process_Filename_Entry` | 108B | Process file name entry |
| `0x08011680` | `FATFS_Check_File_Access_Permission` | 54B | Check file access |
| `0x08011648` | `FATFS_Validate_And_Init_File` | 52B | Validate + init file |
| `0x08011bf0` | `FATFS_Get_Data_Pointer` | 48B | Get data pointer |
| `0x08011c70` | `FATFS_Find_Active_FileObj` | 48B | Find active file object |
| `0x0800d700` | `FATFS_Init_Log_File_If_Needed` | 46B | Init log file if needed |
| `0x08011acc` | `FATFS_Find_And_Open_Dir_Entry` | 36B | Find + open entry |
| `0x08011610` | `FATFS_Register_Mount_Slot` | 34B | Register mount slot |
| `0x08010be4` | `FATFS_Is_Queue_Empty` | 26B | Queue empty? |
| `0x08011408` | `FATFS_IncrementRefCount` | 26B | Reference count++ |
| `0x080120c4` | `FATFS_DecrementAndNotify` | 26B | Reference count-- + notify |
| `0x08011098` | `FATFS_Count_Erased_Bytes_Div4` | 22B | Count erased bytes |
| `0x08011818` | `FATFS_Process_And_Mark_Entry` | 20B | Process entry + mark |

### CLI — Letter-Shell Debug Console (28 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08011cf8` | `LetterShell_Key_Processor` | 232B | Key input processing |
| `0x08011fec` | `CLI_Line_Insert_Char` | 212B | Insert character into line |
| `0x0801170c` | `CLI_Delete_Char` | 218B | Delete character |
| `0x08011e10` | `CLI_History_Navigate` | 194B | Navigate command history |
| `0x08012138` | `CLI_Print_Command_Entry` | 194B | Print command entry |
| `0x08011af0` | `CLI_Native_Function_Call` | 194B | Call native function |
| `0x08012224` | `CLI_Parse_Arguments` | 142B | Parse arguments |
| `0x080122b2` | `CLI_Strip_Quotes_From_Args` | 104B | Strip quotation marks |
| `0x08012898` | `CLI_Print_Return_Value` | 104B | Print return value |
| `0x080127c4` | `CLI_Cmd_FindFile` | 98B | FindFile command |
| `0x080118a0` | `CLI_Detect_Number_Base` | 88B | Detect number base (hex/dec) |
| `0x08012768` | `CLI_Output_Line_WithLimit` | 88B | Line output with limit |
| `0x08011a2c` | `CLI_Parse_Token_Value` | 86B | Parse token value |
| `0x080126bc` | `CLI_Int_To_DecString` | 86B | Int → decimal string |
| `0x08012830` | `CLI_Print_Command_Help` | 84B | Command help |
| `0x08011ca4` | `CLI_Get_Variable_Value_By_Type` | 78B | Variable value by type |
| `0x08011c20` | `CLI_Get_Value_String_By_Type` | 76B | Value string by type |
| `0x080118f8` | `CLI_Parse_Escape_Char` | 70B | Parse escape character |
| `0x080120ec` | `CLI_List_Commands` | 70B | Print command list |
| `0x08012460` | `CLI_Register_Command_Table` | 66B | Register command table |
| `0x08011a82` | `CLI_Parse_Quoted_String` | 74B | Parse quoted string |
| `0x08012924` | `CLI_Write_String_Via_Callback` | 48B | Output string via callback |
| `0x0800c43c` | `CLI_Process_RX_Buffer` | 38B | Process RX buffer |
| `0x080116d4` | `CLI_Pad_Spaces_And_Set_Cursor` | 40B | Space padding + cursor |
| `0x080117e8` | `CLI_Send_Repeated_Char` | 28B | Send character repeatedly |
| `0x08012208` | `CLI_Clear_Flag_And_Insert_Char` | 28B | Clear flag + insert character |
| `0x0801089e` | `CLI_Get_Free_Args` | 20B | Get free arguments |
| `0x0801088c` | `CLI_Free_Command` | 18B | Free command |
| `0x080116fc` | `CLI_Backspace` | 16B | Backspace |
| `0x08011638` | `CLI_Delete_Forward` | 14B | Delete forward |
| `0x0801274a` | `CLI_History_Up` | 14B | History up |
| `0x08012758` | `CLI_Write_Callback_1Byte` | 14B | 1 byte via callback |
| `0x0801231a` | `CLI_TxBuffer_Send_Next_Byte` | 30B | TX buffer next byte |
| `0x08011de4` | `CLI_Cmd_Dispatch_By_Argc` | 44B | Dispatch by argc |
| `0x080120de` | `CLI_Show_Help` | 12B | Show help |

### RTOS — FreeRTOS Kernel (20 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08017738` | `xQueueSemaphoreTake` | 494B | Take semaphore |
| `0x0801719c` | `xQueueGenericSend` | 476B | Generic queue send |
| `0x0801754c` | `xQueueReceive` | 398B | Queue receive |
| `0x08011210` | `pvPortMalloc` | 340B | Heap allocation |
| `0x08017b50` | `xTaskIncrementTick` | 300B | Increment tick counter |
| `0x080173d4` | `xQueueSendFromISR` | 290B | Queue send from ISR |
| `0x08016958` | `vTaskPriorityDisinheritAfterTimeout` | 260B | Priority disinherit after timeout |
| `0x08018038` | `xTaskResumeAll` | 240B | Resume all tasks |
| `0x08017d18` | `xTaskPriorityDisinherit` | 218B | Priority disinherit |
| `0x080109a8` | `prvInitialiseNewTask` | 204B | Initialize new task |
| `0x08017e74` | `vTaskPriorityInherit` | 186B | Priority inherit |
| `0x08016410` | `vPortFree` | 172B | Heap free |
| `0x080170ac` | `xQueueGenericReset` | 160B | Reset queue |
| `0x080179bc` | `xTaskCheckForTimeOut` | 160B | Timeout check |
| `0x08010680` | `prvAddTaskToReadyList` | 156B | Task into ready list |
| `0x0801825c` | `xTimerGenericCommand` | 140B | Send timer command |
| `0x08017f3c` | `xTaskRemoveFromEventList` | 142B | Task from event list |
| `0x08016598` | `vPortValidateInterruptPriority` | 136B | Validate interrupt priority |
| `0x08016c08` | `vTaskSwitchContext` | 134B | Context switch |
| `0x08010c1c` | `prvTimerCallback_Handler` | 128B | Timer callback handler |
| `0x0801080e` | `prvCopyDataToQueue` | 126B | Copy data into queue |
| `0x08011192` | `prvUnlockQueue` | 126B | Unlock queue |
| `0x08010f94` | `prvTimerProcessExpired` | 170B | Process expired timers |
| `0x08016350` | `RTOS_vPortExitCritical` | 74B | Exit critical section |
| `0x08016820` | `RTOS_vTaskPlaceOnEventList` | 66B | Task onto event list |
| `0x080168b8` | `RTOS_vTaskPlaceOnEventListRestricted` | 74B | Event list (restricted) |
| `0x08010f34` | `RTOS_Update_Current_TCB_Runtime` | 42B | Update TCB runtime |
| `0x08010f68` | `RTOS_Check_Tick_Overflow` | 40B | Check tick overflow |
| `0x08011428` | `RTOS_Init_Task_Stack_Frame` | 38B | Init task stack frame |
| `0x080148e8` | `RTOS_Get_Task_Stack_Watermark` | 34B | Query stack watermark |
| `0x08016fb4` | `RTOS_Create_Binary_Semaphore` | 34B | Create binary semaphore |
| `0x08010960` | `RTOS_Semaphore_Init_And_Give` | 30B | Semaphore init + give |

### APP — BMS Application Logic (35 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08007460` | `BMS_Config_Params_Load_Or_Default` | 788B | Load config or defaults |
| `0x0800702c` | `bms_data_printf` | 770B | BMS data debug output |
| `0x08008120` | `BMS_Config_Params_Factory_Reset` | 728B | Config to factory settings |
| `0x08009f70` | `PerPack_Struct_Builder` | 580B | Build per-pack struct |
| `0x08000f98` | `Pack_Online_Watchdog` | 374B | Pack online watchdog (20s) |
| `0x080142c0` | `SOC_Display_And_FullFlag_Update` | 318B | SOC display + full flag |
| `0x08012e4c` | `Debug_Print_Task_Stack_Usage` | 304B | Print task stack usage |
| `0x080056c0` | `Pack_LowVoltage_Balancer` | 252B | Pack balancing at low voltage |
| `0x08004c6c` | `Pack_MasterSelect_Monitor` | 224B | Master pack selection monitor |
| `0x08002198` | `System_Reset_And_Halt` | 136B | System reset and halt |
| `0x08000a84` | `APP_DMA_Channel_Mode_Switch` | 90B | Switch DMA channel mode |
| `0x08000cd4` | `APP_CAN_RX_Task_Process` | 98B | Process CAN RX task |
| `0x08008784` | `APP_Pack_Discovery_Init` | 94B | Initialize pack discovery |
| `0x0800c7f4` | `APP_Get_Protection_Status_Byte` | 88B | Get protection status byte |
| `0x080157cc` | `APP_Clear_SOC_Accumulator_Arrays` | 80B | Clear SOC accumulator arrays |
| `0x080086a4` | `APP_Read_Flash_Boot_Count_Validated` | 78B | Read boot counter from flash |
| `0x08006cc8` | `Pack_Get_Protect1Bit0_Mask` | 70B | Mask of packs with Protect1 bit 0 (cell undervoltage); until 2026-09-24 `APP_Get_Packs_Charging_Bitmask` |
| `0x08005854` | `APP_Get_Effective_SOC_Display` | 62B | Get effective display SOC |
| `0x08006b54` | `APP_Calc_Pack_SOC_From_Voltage` | 62B | Calculate SOC from voltage |
| `0x08014ec0` | `APP_CAN_Reset_Counters_Init` | 62B | Reset CAN counters |
| `0x08006c00` | `APP_Get_Packs_Charging_Bitmask_2` | 60B | Charging bitmask (variant 2) |
| `0x08006c48` | `APP_Get_Packs_Discharging_Bitmask` | 60B | Discharging bitmask of all packs |
| `0x08015a3c` | `APP_Shutdown_Countdown_Tick` | 64B | Shutdown countdown |
| `0x080065fc` | `APP_Factory_Reset_And_Reboot` | 64B | Factory reset + reboot |
| `0x08014200` | `APP_CAN_Send_With_Validation` | 64B | CAN send with validation |
| `0x08013ec4` | `APP_CAN_Verify_Config_Table` | 60B | Verify CAN config table |
| `0x08006a94` | `APP_Set_Pack_MOS_Status_Flags` | 50B | Set pack MOS status |
| `0x08006c90` | `APP_Calc_Pack_SOC_Percent_Rounded` | 50B | Calculate SOC % rounded |
| `0x0801427c` | `APP_CAN_Clamp_And_Scale_Temperature` | 48B | Clamp + scale temperature |
| `0x080097bc` | `APP_Toggle_Precharge_Pin` | 42B | Toggle precharge pin |
| `0x080097f0` | `APP_Check_And_Clear_Wakeup_Flag` | 46B | Check + clear wakeup flag |
| `0x0800a4e8` | `APP_Periodic_600_Tick_Callback` | 38B | Periodic 600-tick callback |
| `0x080083fc` | `APP_Config_Checksum_And_Save` | 30B | Config checksum + save |
| `0x080089ac` | `APP_Clear_ErrorCounters` | 24B | Clear error counters |
| `0x0800872a` | `APP_Swap_Params_And_Call` | 20B | Swap parameters + call |
| `0x08000cb8` | `APP_Clear_SPI_BusyFlag` | 10B | Clear SPI busy flag |
| `0x08000cc8` | `APP_Clear_CAN_ErrorCounter` | 8B | Clear CAN error counter |
| `0x0800f9a4` | `APP_Get_RS485_Address` | 8B | Read RS485 address |
| `0x0801049c` | `APP_Set_CLI_DefaultHandler` | 8B | Set CLI default handler |
| `0x080104dc` | `APP_Clear_CLI_OutputLength` | 8B | Clear CLI output length |
| `0x080161c0` | `APP_Set_ChargeDischargeState` | 8B | Set charge/discharge state |
| `0x08016814` | `APP_Set_NeedYield_Flag` | 8B | Set NeedYield flag |
| `0x080142b4` | `APP_Get_SOC_AccumulatedEnergy` | 6B | Read accumulated energy |
| `0x08006d1c` | `APP_Get_PackFaultStatus` | 6B | Read pack fault status |
| `0x0800edec` | `APP_Get_ProtectFlags_Byte` | 6B | Protection flags byte |
| `0x08017b24` | `APP_Get_Calibration_Status` | 24B | Calibration status |
| `0x08017b44` | `APP_Get_SystemTickCounter` | 6B | System tick counter |
| `0x080167fc` | `APP_Get_PackVoltage_Current` | 14B | Read pack voltage/current |
| `0x080058a4` | `APP_Is_Flag_2A6D_Set` | 14B | Flag 0x2A6D set? |
| `0x0800edd8` | `APP_Is_Not_Initialized_4043` | 14B | Check init flag 0x4043 |
| `0x0800edf8` | `APP_Is_Flag_4015_Set` | 14B | Flag 0x4015 set? |
| `0x08014004` | `APP_Is_Both_Conditions_Met` | 26B | Both conditions met? |
| `0x0800ee0c` | `APP_Try_Acquire_Lock` | 38B | Try to acquire lock |
| `0x0800d78c` | `APP_vTaskDelayUntil_Wrapper` | 12B | vTaskDelayUntil wrapper |
| `0x08010168` | `APP_Get_Pack_Summary_Data` | 68B | Get pack summary |
| `0x080058b8` | `APP_Find_Pack_With_Highest_SOC` | 66B | Pack with highest SOC |
| `0x0800598c` | `APP_Find_Pack_With_Lowest_SOC` | 66B | Pack with lowest SOC |
| `0x0800b8c0` | `APP_Accumulate_Idle_Charge_Counter` | 76B | Accumulate idle charge counter |
| `0x08006ec8` | `BMS_Check_FullCharge_Condition` | 158B | Check full-charge condition |

### CAN — CAN Bus Protocol (30 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08005af0` | `CAN_TX_PerPack_10Msgs` | 1868B | 10 CAN messages per pack |
| `0x0800c46c` | `CAN_TX_BMS_Status_Reporter` | 888B | BMS status CAN report |
| `0x0800b388` | `CAN_Unpack_Pack_Activation_Data` | 850B | Unpack pack activation |
| `0x0800c8bc` | `CAN_RX_Command_Dispatcher` | 324B | CAN RX command dispatcher |
| `0x0800b704` | `CAN_CMD_Pack_Activation` | 392B | Pack activation command |
| `0x080053d8` | `CAN_TX_PF2_Capacity` | 124B | TX frame: capacity |
| `0x08005474` | `CAN_TX_PF3_ChargeDischargeLimits` | 318B | TX frame: charge/discharge limits |
| `0x08002d3c` | `CAN_FilterBank_Configure` | 258B | Configure filter bank |
| `0x08003000` | `CAN_RxMailbox_Read` | 240B | Read RX mailbox |
| `0x080030f0` | `CAN_TxMailbox_Write` | 294B | Write TX mailbox |
| `0x080026fc` | `TIM_PWM_Configure_Channel` | 344B | Configure PWM channel |
| `0x080059dc` | `CAN_RX_Handler` | 222B | CAN RX handler |
| `0x0800b920` | `CAN_Pack_Activation_Config_Receive` | 178B | Receive pack activation config |
| `0x0800ca88` | `CAN_TX_Send_Status_Response` | 146B | Send status response |
| `0x08005908` | `CAN_Find_Closest_Pack_To_50pct` | 120B | Pack closest to 50% SOC |
| `0x08005640` | `CAN_CMD_04_Handler` | 116B | CAN command 0x04 |
| `0x0800536c` | `CAN_TX_PF1_PackMeasurements` | 94B | TX frame: measurements |
| `0x0800c850` | `CAN_RX_System_Command_Handler` | 88B | System command handler |
| `0x080055d4` | `CAN_TX_PF4_ProtectWarnings` | 88B | TX frame: protection warnings |
| `0x080057dc` | `CAN_CMD_01_Handler` | 88B | CAN command 0x01 |
| `0x0800ca2c` | `CAN_Handle_Pack_Mode_Command` | 74B | Pack mode command |
| `0x08005320` | `CAN_TX_Orchestrator` | 72B | TX orchestrator |
| `0x08001344` | `CAN_Send_Data_Frame` | 64B | Send data frame |
| `0x08000e50` | `CAN_Send_Status_Message` | 74B | Send status message |
| `0x08000f58` | `CAN_Send_Broadcast_Message` | 58B | Send broadcast |
| `0x08006470` | `CAN_Set_Notify_Flag_And_Send` | 44B | Notify flag + send |
| `0x080065d4` | `CAN_Send_With_Retry` | 38B | Send with retry |
| `0x08008674` | `CAN_Load_Or_Init_Config_Value` | 42B | Load config or init |
| `0x08002e44` | `CAN_RxFIFO0_Read_And_Set_Flag` | 28B | Read RX FIFO0 + flag |
| `0x080129c4` | `CAN_Set_Config_Value` | 16B | Set config value |
| `0x0800f8b0` | `CAN_Extract_StdId_From_ExtId` | 10B | Extract StdId from ExtId |
| `0x0800d574` | `CAN_Get_DLC` | 10B | Read DLC |
| `0x0800d57e` | `CAN_Set_DLC` | 8B | Set DLC |
| `0x080069ea` | `CAN_Set_TxMailbox_TDLR` | 4B | TX mailbox data low |
| `0x080069ee` | `CAN_Set_TxMailbox_TIR` | 4B | TX mailbox identifier |
| `0x0800d228` | `CAN_Set_TxMailbox_TDHR` | 6B | TX mailbox data high |
| `0x0800cf8c` | `CAN_Set_FilterMask_Inverted` | 6B | Filter mask inverted |
| `0x0800d09a` | `CAN_Get_RxFIFO_Data_Low` | 6B | RX FIFO data low |
| `0x0800d21c` | `CAN_Set_FilterBank_FR1` | 4B | Filter bank register 1 |
| `0x0800d220` | `CAN_Set_FilterBank_FR2` | 4B | Filter bank register 2 |
| `0x0800d224` | `CAN_Set_FilterBank_FR3` | 4B | Filter bank register 3 |
| `0x08005ae4` | `CAN_CMD_02_Handler` | 12B | CAN command 0x02 |

### RS485 — RS485 Communication (16 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800d804` | `RS485_Dispatcher` | 1614B | RS485 main dispatcher |
| `0x08007778` | `RS485_Register_Read_Handler` | 1438B | Register read handler |
| `0x08007d24` | `RS485_Register_Write_Handler` | 1004B | Register write handler |
| `0x0800fba4` | `RS485_Pack_Telemetry_Response` | 296B | Pack telemetry response |
| `0x08002a24` | `USART2_UART4_Init` | 360B | USART2/UART4 init |
| `0x08002bec` | `UART_IRQ_Handler` | 194B | UART IRQ handler |
| `0x08002234` | `USART2_RX_Process` | 170B | USART2 RX processing |
| `0x080022fc` | `UART4_RX_Process` | 134B | UART4 RX processing (inter-pack) |
| `0x0800b088` | `RS485_Protocol_Byte_Receiver` | 120B | Protocol byte receiver |
| `0x0800b10c` | `RS485_Build_And_Send_Frame` | 112B | Build + send frame |
| `0x0800adf8` | `RS485_Process_RX_Command` | 110B | Process RX command |
| `0x08002420` | `Parse_Checkout_RS485_Config` | 106B | Parse RS485 config |
| `0x0800d278` | `RS485_Send_Response` | 102B | Send response |
| `0x08002138` | `RS485_Send_Packet_Bus1` | 96B | Send packet on bus 1 |
| `0x080024a0` | `RS485_Send_Packet_Bus3` | 96B | Send packet on bus 3 |
| `0x0800af74` | `RS485_Send_Error_Response_If_Pending` | 72B | Error response if pending |
| `0x08015ba4` | `RS485_Reset_Transceiver` | 40B | Reset transceiver |
| `0x0800ee38` | `RS485_Send_Bytes` | 38B | Send bytes |
| `0x0800d4ac` | `USART_Init_Config` | 188B | USART configuration |
| `0x0800d34c` | `USART_SPI_Clock_Reset` | 186B | USART/SPI clock reset |
| `0x0800cfa4` | `TIM_Clock_Reset` | 196B | Timer clock reset |

### I2C_EEPROM — EEPROM Access (13 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08002ecc` | `I2C_Config_Apply` | 276B | Apply I2C configuration |
| `0x0800150c` | `I2C1_Init` | 292B | Initialize I2C1 |
| `0x08001388` | `SPI2_And_I2C1_Init` | 284B | SPI2 + I2C1 init |
| `0x08004da4` | `I2C_EEPROM_Read` | 148B | Read EEPROM |
| `0x08004e38` | `I2C_EEPROM_Write` | 150B | Write EEPROM |
| `0x08008fd4` | `I2C_EEPROM_Format_EventLog` | 94B | Format event log |
| `0x080063c0` | `I2C_EEPROM_Read_WithMutex` | 84B | Read with mutex |
| `0x08006418` | `I2C_EEPROM_Write_WithMutex` | 84B | Write with mutex |
| `0x080103d0` | `EEPROM_EventLog_GetSlotAddrs` | 82B | Event log slot addresses |
| `0x08001fb8` | `I2C_EEPROM_Read_Byte_BitBang` | 68B | Read byte (bit-bang) |
| `0x08002068` | `I2C_EEPROM_Read_Byte_Sequence` | 56B | Read byte sequence |
| `0x08001ee0` | `I2C_EEPROM_Write_Byte_Sequence` | 56B | Write byte sequence |
| `0x08001f84` | `I2C_EEPROM_Stop_Condition` | 46B | Stop condition |
| `0x080020f0` | `I2C_EEPROM_Read_ACK_BitBang` | 66B | Read ACK (bit-bang) |
| `0x0800f8e4` | `I2C_EEPROM_Send_Byte_Wait` | 44B | Send byte + wait |
| `0x0800f8ba` | `I2C_EEPROM_Calc_Checksum` | 42B | EEPROM checksum |
| `0x080051b8` | `I2C_EEPROM_Write_7Bytes` | 28B | Write 7 bytes |

### KA495XX — BMIC Cell Monitoring SPI Driver (35 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x080150ac` | `KA495XX_Read_CellVoltages_And_Temps` | 674B | Read cell voltages + temperatures |
| `0x08014c54` | `KA495XX_Cell_Balance_StateMachine` | 612B | Cell balancing state machine |
| `0x08014f0c` | `KA495XX_Compute_ADC_Offsets` | 398B | Compute ADC offsets |
| `0x0801550c` | `KA495XX_Scale_To_Millivolts` | 390B | Scaling → millivolts |
| `0x08015bd4` | `KA495XX_Read_Fault_Status` | 340B | Read fault status |
| `0x0801491c` | `KA495XX_SPI_Bulk_Read` | 314B | SPI bulk read |
| `0x08015360` | `KA495XX_Compute_Power_And_Energy` | 306B | Compute power + energy |
| `0x08014510` | `KA495XX_Encode_OVP_UVP_OCD_Thresholds` | 284B | Encode thresholds |
| `0x08015ed4` | `KA495XX_Register_Init` | 260B | Initialize registers |
| `0x08014b5c` | `KA495XX_Cell_Balance_Decision` | 232B | Balancing decision |
| `0x08014410` | `KA495XX_SPI_Read_Single_Reg` | 230B | SPI read single register |
| `0x0801475c` | `KA495XX_Encode_Protection_Config` | 226B | Encode protection config |
| `0x080160c8` | `KA495XX_ADC_Conversion_Trigger` | 230B | Trigger ADC conversion |
| `0x08015824` | `KA495XX_Init_Default_Params` | 214B | Set default parameters |
| `0x08015d34` | `KA495XX_Compute_Calibration_Factor` | 202B | Compute calibration factor |
| `0x08015914` | `KA495XX_Main_StateMachine` | 186B | Main state machine (15 states) |
| `0x08013f44` | `KA495XX_Read_Temperature_Sensors` | 188B | Read temperature sensors |
| `0x080146a8` | `KA495XX_Encode_OCSC_Config` | 174B | Encode OCSC config |
| `0x08015af8` | `KA495XX_Init_Sequence` | 168B | Init sequence |
| `0x0801572c` | `KA495XX_ClearMinMaxArrays` | 150B | Clear min/max arrays |
| `0x0801401e` | `KA495XX_WriteProtectionRegs` | 150B | Write protection registers |
| `0x08014174` | `KA495XX_Cell_Balance_Config` | 126B | Configure balancing |
| `0x08015a80` | `KA495XX_Wakeup_Sequence` | 114B | Wakeup sequence |
| `0x080156b0` | `KA495XX_Cell_Balance_Check` | 110B | Balancing check |
| `0x08014634` | `KA495XX_Pack_Config_Bitfield` | 112B | Pack config bitfield |
| `0x08015e70` | `KA495XX_Verify_Config_Registers` | 96B | Verify config registers |
| `0x08014a6c` | `KA495XX_SPI_Write_Register` | 92B | SPI write register |
| `0x080159e4` | `KA495XX_Poll_Watchdog_Flag` | 84B | Poll watchdog flag |
| `0x0801601c` | `KA495XX_Build_Balancing_Bitmask` | 76B | Build balancing bitmask |
| `0x08016078` | `KA495XX_Handle_Fault_Flags` | 72B | Handle fault flags |
| `0x08014ad0` | `KA495XX_SPI_Write_Masked_Register` | 72B | SPI masked write |
| `0x08014b18` | `KA495XX_Check_And_Clear_Watchdog` | 66B | Check + clear watchdog |
| `0x08013f04` | `KA495XX_Check_Fault_Registers` | 64B | Check fault registers |
| `0x08009828` | `KA495XX_Write_Error_Lock_Register` | 58B | Write error lock |
| `0x08009868` | `KA495XX_Check_Error_Lock_Status` | 60B | Check error lock status |
| `0x080086f2` | `KA495XX_Write_Balancing_Register` | 56B | Write balancing register |
| `0x0800505c` | `KA495XX_SPI_Send_Sleep_Command` | 44B | Send sleep command |
| `0x080052bc` | `KA495XX_SPI_Read_Register` | 42B | SPI read register |
| `0x080052ec` | `KA495XX_SPI_Write_Register_2` | 42B | SPI write register (V2) |
| `0x0800518c` | `KA495XX_SPI_Read_All_Registers` | 38B | Read all registers |
| `0x080032d4` | `KA495XX_Check_Tick_Overflow` | 30B | Check tick overflow |
| `0x08015fd8` | `KA495XX_Read_Status_And_Check_Balancing` | 28B | Check status + balancing |
| `0x080089d8` | `KA495XX_Reset_And_Init` | 8B | Reset + init |
| `0x08014a60` | `KA495XX_SPI_BulkRead_Reg0x59` | 12B | Bulk read register 0x59 |
| `0x08014500` | `KA495XX_Get_Register` | 10B | Get register value |
| `0x08014910` | `KA495XX_Set_Register` | 8B | Set register value |
| `0x08014110` | `KA495XX_Comm_Error_Check_Bit1` | 94B | Check comm error bit 1 |
| `0x080140b4` | `KA495XX_Comm_Error_Check_Bit0` | 88B | Check comm error bit 0 |

### SOC — State of Charge Algorithm (30 functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x080098b8` | `Voltage_Temp_Protection_Checker` | 1600B | Voltage/temp protection |
| `0x08013520` | `SOC_Algorithm_Orchestrator` | 346B | SOC algorithm orchestrator |
| `0x0800f338` | `Coulomb_Counting` | 334B | Coulomb counting |
| `0x08013318` | `SOC_Clamp_And_Store` | 312B | Clamp SOC + store |
| `0x0800f5f8` | `Discharge_SOC_Integration` | 370B | Discharge SOC integration |
| `0x0800ef40` | `Charge_Energy_Calc` | 296B | Charge energy calculation |
| `0x0800f490` | `Discharge_Energy_Calc` | 296B | Discharge energy calculation |
| `0x0800f1a8` | `Charge_SOC_Integration` | 256B | Charge SOC integration |
| `0x0800f7b8` | `SOC_Voltage_Drop_Compensator` | 244B | Voltage drop compensation |
| `0x080104e8` | `SOC_Init_Capacity_Params` | 242B | Init capacity parameters |
| `0x0800ee60` | `SOC_Energy_Calibration_Update` | 214B | Energy calibration |
| `0x0800f0b4` | `SOC_Charge_Interpolation` | 232B | Charge interpolation |
| `0x0801369c` | `SOC_EEPROM_LoadCalibration` | 204B | Load calibration from EEPROM |
| `0x080101e4` | `Full_OCV_Lookup` | 204B | Full OCV table |
| `0x0800fade` | `SOC_Current_Bracket_Lookup` | 198B | Current bracket lookup |
| `0x08011558` | `SOC_Set_From_External` | 174B | Set SOC externally |
| `0x08010320` | `Temperature_Index_Lookup` | 174B | Temperature index lookup |
| `0x0800fdf0` | `Max_Charge_Current_Calc` | 220B | Calculate max charge current |
| `0x0800fee8` | `Max_Discharge_Current_Calc` | 236B | Calculate max discharge current |
| `0x0800fd3c` | `SOC_LoadCalibration_Float` | 166B | Load calibration as float |
| `0x08013c90` | `SOC_EEPROM_SaveIfChanged` | 154B | Save to EEPROM on change |
| `0x080114c4` | `Quick_OCV_Lookup` | 142B | Quick OCV table |
| `0x08013d98` | `SOC_Coulomb_Count_Tick` | 284B | Coulomb count tick |
| `0x08009f70` | `PerPack_Struct_Builder` | 580B | Build per-pack struct |
| `0x0800f9d8` | `SOC_State_Detect` | 126B | Detect SOC state (charging/discharging/idle) |
| `0x080105e8` | `prvAddCurrentTaskToDelayedList` | 124B | FreeRTOS internal (caller of `vTaskDelay`, `vTaskPlaceOnEventList*`); until 2026-09-24 wrongly `SOC_Update_Accumulator` |
| `0x080100f4` | `SOC_Calc_Percentage_From_Range` | 112B | Calculate percentage from range |
| `0x0801042c` | `SOC_Calc_Percentage_Inverse` | 108B | Calculate percentage inverse |
| `0x08013844` | `soc_algorithm_dump` | 106B | SOC algorithm debug dump |
| `0x080137dc` | `SOC_Save_To_EEPROM` | 94B | Save SOC to EEPROM |
| `0x08013d44` | `SOC_Init_From_OCV_Lookup` | 84B | Init SOC from OCV table |
| `0x08013460` | `SOC_Check_Temperature_Current_Reset` | 78B | Temp/current reset check |
| `0x080134ae` | `SOC_Calculate_OCV_From_Voltage` | 64B | Calculate OCV from voltage |
| `0x0800fa74` | `OCV_Voltage_To_Index` | 78B | Voltage → OCV index |
| `0x080102b4` | `OCV_Correction` | 44B | OCV correction |
| `0x080102e4` | `Capacity_Factor_Calc` | 44B | Calculate capacity factor |
| `0x0800f078` | `SOC_Smoothing` | 60B | SOC smoothing |
| `0x080134f0` | `SOC_Algorithm_Entry` | 38B | SOC algorithm entry point |
| `0x0800f2f4` | `SOC_Init_And_Load_Calibration` | 38B | Init + load calibration |
| `0x0800f2b8` | `SOC_Set_NearFull_Threshold_990000` | 54B | Near-full threshold (99%) |
| `0x0800f914` | `SOC_Set_Full_Charge_1000000` | 60B | Full-charge threshold (100%) |
| `0x0800f95c` | `SOC_Set_Empty_Discharge_Zero` | 60B | Empty threshold (0%) |
| `0x0800f778` | `SOC_Set_LowSOC_Threshold_10000` | 58B | Low-SOC threshold (1%) |
| `0x0800f5c8` | `SOC_Init_Capacity_And_Percentages` | 42B | Init capacity + percentages |
| `0x0800fcdc` | `SOC_Check_Max_Capacity_Valid` | 44B | Max capacity valid? |
| `0x0800fd0c` | `SOC_Check_Min_Capacity_Valid` | 44B | Min capacity valid? |
| `0x0800f9bc` | `SOC_Get_Charge_Current_If_SOC_Above_8` | 20B | Charge current if SOC > 8% |
| `0x0800fa60` | `SOC_Select_Threshold_3700_3600` | 18B | Threshold 3700/3600 mV |
| `0x0800facc` | `SOC_Select_Threshold_2420_2750` | 18B | Threshold 2420/2750 mV |
| `0x08013798` | `SOC_Read_Flash_Cycle_Counters` | 64B | Cycle counters from flash |
| `0x0801376c` | `SOC_Init_Params_To_Invalid` | 42B | Init parameters to invalid |
| `0x08016cec` | `Full_Recalibration` | 66B | Full recalibration |
| `0x08011940` | `str_to_float` | 232B | String → float conversion |

### Protect — Battery Protection (8 Functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x08008420` | `Current_Protection_Checker` | 544B | Current protection checker |
| `0x080087f4` | `HW_Overcurrent_Protection` | 366B | HW overcurrent protection |
| `0x0800b180` | `Warning_Bitmask_Builder` | 170B | Build warning bitmask |
| `0x0800b280` | `Protect1_Bitmask_Caller` | 170B | Protect1 bitmask (caller) |
| `0x0800b230` | `Protect1_Bitmask_Builder` | 76B | Build Protect1 bitmask |
| `0x0800b330` | `Protect2_Bitmask_Builder` | 72B | Build Protect2 bitmask |
| `0x08014844` | `Protect_Update_NTC_MinMax` | 102B | Update NTC min/max |
| `0x08012b64` | `Protect_Set_Error_Lock` | 68B | Set error lock |
| `0x0800933c` | `Protect_Debounce_Counter_A` | 42B | Debounce counter A |
| `0x08009366` | `Protect_Debounce_Counter_B` | 42B | Debounce counter B |
| `0x08009548` | `voltage_protection_check` | 38B | Voltage protection check |
| `0x0800f9b0` | `Protect_Get_MaxChgCurrent_Div10` | 6B | Max charge current ÷ 10 |

### NTC — Temperature Sensors (6 Functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800b9e8` | `NTC_Lookup_Table_1` | 282B | NTC lookup table 1 |
| `0x0800bbd8` | `NTC_Lookup_Table_2` | 282B | NTC lookup table 2 |
| `0x08001254` | `ADC_Read_NTC_Temperature_Ch9` | 106B | Read ADC NTC channel 9 |
| `0x080012cc` | `ADC_Read_NTC_Temperature_Ch2` | 106B | Read ADC NTC channel 2 |
| `0x08011454` | `NTC_Calculate_Temperature` | 104B | Calculate temperature |
| `0x080154b4` | `NTC_Convert_Active_Sensors` | 76B | Convert active sensors |
| `0x080069f4` | `NTC_ADC_To_Temperature` | 68B | ADC value → temperature |
| `0x08008fa0` | `NTC_Count_Out_Of_Range_Sensors` | 46B | Count out-of-range sensors |
| `0x08001144` | `ADC_DMA_Init` | 148B | Initialize ADC DMA |

### OTA — Firmware Update (7 Functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800df0c` | `PackRecovery_LowCell_Charge_SM` | 1128B | Recovery charging of deeply discharged packs, **not OTA**; until 2026-09-24 `BMS_OTA_Upgrade_StateMachine` (see `VNSD_BMS_Pack_Rotation_Taper_v118.md` §7.4) |
| `0x0800a9f8` | `BMS_OTA_Flash_Writer` | 910B | OTA flash writer |
| `0x0800a1d8` | `RS485_OTA_Flash_Handler` | 728B | RS485 OTA flash handler |
| `0x0800cbe0` | `OTA_Precondition_Manager` | 342B | Check OTA preconditions |
| `0x0800ae70` | `OTA_PreCharge_Relay_Sequence` | 242B | OTA precharge relay sequence |
| `0x08001b74` | `OTA_Heartbeat_And_Relay_Sequence` | 202B | OTA heartbeat + relay |
| `0x08006acc` | `PackSelect_Mode_BySoc` | 124B | Selection mode 0–3 by SOC of the new pack; until 2026-09-24 wrongly named `OTA_Validate_FW_Version` |
| `0x0800d5f4` | `Flash_Erase_Range` | 124B | Erase flash range |
| `0x0800d674` | `Flash_Write_Words` | 130B | Write words to flash |
| `0x0800eda8` | `OTA_Set_Reboot_Params` | 34B | Set reboot parameters |

### Miscellaneous (9 Functions)

| Address | Name | Size | Description |
|---|---|---|---|
| `0x0800e678` | `vprintf` | 1696B | Printf implementation |
| `0x0800e514` | `__dtoa_engine` | 334B | Double-to-ASCII engine |
| `0x0800cb24` | `Precharge_StateMachine` | 162B | Precharge state machine |
| `0x0800632c` | `EXTI_Config` | 142B | Configure EXTI interrupt |
| `0x08004ece` | `RTC_Read_DateTime` | 246B | Read RTC date/time |
| `0x0800508c` | `RTC_Set_DateTime_BCD` | 212B | Set RTC (BCD format) |
| `0x08004fc4` | `INIT_RTC_And_DateTime` | 50B | Initialize RTC |
| `0x08001c4c` | `TIM2_TIM3_Init` | 208B | Initialize timers 2/3 |
| `0x080089e0` | `RTOS_vTaskKA495XX` | 156B | FreeRTOS KA495XX task |
| `0x08003dc4` | `HAL_GPIO_Control_Switch` | 90B | GPIO control switch |

---

*Created via Ghidra + ReVa MCP — static analysis without a live device*
