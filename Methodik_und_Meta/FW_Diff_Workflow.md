# Marstek Venus — Firmware diff workflow

> 🇩🇪 Deutsche Version: [FW_Diff_Workflow_de.md](FW_Diff_Workflow_de.md)

Guide: how to proceed with a new firmware version in order to identify changes, transfer labels and update the documentation.

---

## Prerequisites

- Ghidra with ReVa MCP is connected
- The old FW version has already been analyzed in Ghidra and is fully labeled
- The new FW binary is available as a file (provided by the user or via OTA download)

### Known Ghidra projects (status July 2026)

| FW | Ghidra path | Doc file | Functions |
|---|---|---|---|
| BMS v117.7 | `/20251010135647565eb2036.bin` | `BMS/VNSD_BMS_FW_Analysis_v117.7.md` | 540 (100% named) |
| Control v149.2 | *(check path in Ghidra)* | `Control_FW/VNSD_Control_FW_Analysis_app_1492_0702_142136.md` | 1,623 (1,619 named, see Control_FW/VNSD_Control_FW_Function_Tracking_new.md) |
| Micro/Inverter v116 | *(check path in Ghidra)* | `VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analysis_vd_inv_app_0116.md` | 452 |

---

## Step 1: Import and analyze the new binary

```
Tool: ghidra:import-file
  filePath: <local path to the new binary>
  programName: <e.g. "BMS_v178" or "Control_v150">

Tool: ghidra:analyze-program
  programPath: <Ghidra path of the imported program>
  forceFullAnalysis: true
  waitSeconds: 30
```

**Important for STM32 binaries (BMS, Micro):**
- Set flash base to `0x08000000` (rename memory block to "flash", write=false)
- Compiler: RVDS/Keil ARM (Thumb-2, little endian)
- After import: `analyze-program` with `forceFullAnalysis: true`

**For Control FW:**
- Compiler: GCC (different from BMS/Micro!)
- Same flash base `0x08000000`

---

## Step 2: Create diff session

```
Tool: ghidra:diff-create-session
  sourceProgramPath: <path of the OLD, fully labeled FW>
  destinationProgramPath: <path of the NEW FW>
  waitSeconds: 60
```

The source is always the old, known version — labels/markup flow FROM source TO destination.

Standard correlators (executed automatically):
1. `symbol-name` — Matches functions with the same name
2. `exact-bytes` — Identical byte sequences
3. `exact-instructions` — Identical instructions
4. `exact-mnemonics` — Identical mnemonics
5. `duplicate-instructions` — Duplicated instructions
6. `function-reference` — Reference-based matching

If many functions remain unmatched, additionally:
```
Tool: ghidra:diff-add-correlator
  correlator: "combined-reference"
```

---

## Step 3: Retrieve diff summary

```
Tool: ghidra:diff-summary
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
  topN: 20
  includeBodyByteChanges: true
```

This returns:
- Number of matched/unmatched functions
- Top N of the most heavily changed functions
- New and removed functions

---

## Step 4: Transfer labels and markup

### Automatic (high confidence)

```
Tool: ghidra:diff-transfer-markup
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
  confidence: 0.95
  waitSeconds: 30
```

Automatically transfers: function names, prototypes, data types, comments — for all matches with similarity >= 95%.

### Manual (low confidence)

For matches below the confidence threshold, check individually:

```
Tool: ghidra:diff-function
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
  function: <function name or address>
```

Then, on a positive match, apply individually:

```
Tool: ghidra:diff-apply-match
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
  sourceAddress: <address in old FW>
  destinationAddress: <address in new FW>
```

---

## Step 5: Analyze changes in detail

### List changed functions

```
Tool: ghidra:diff-list-functions
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
  category: "changed"
  sortBy: "similarity"
  includeBodyByteChanges: true
```

### New functions (only in new FW)

```
Tool: ghidra:diff-list-functions
  category: "added"
```

### Removed functions (only in old FW)

```
Tool: ghidra:diff-list-functions
  category: "removed"
```

### String changes

```
Tool: ghidra:diff-strings
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
```

New strings often point to new features, registers or protocol changes.

### Data changes

```
Tool: ghidra:diff-data
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
```

---

## Step 6: Analyze and label new functions

For every new (unmatched) function in the new FW:

1. Decompile: `ghidra:get-decompilation`
2. Check cross-references: `ghidra:find-cross-references`
3. Caller context: `ghidra:get-callers-decompiled`
4. Assign label according to the naming convention (see below)

---

## Step 7: Update documentation

### Determine doc file

Each firmware has its own Markdown file — findings always belong in the matching FW file:

| FW type | File |
|---|---|
| BMS | `BMS/VNSD_BMS_FW_Analyse_v<VERSION>.md` |
| Control | `Control_FW/VNSD_Control_FW_Analyse_app_<VERSION>.md` |
| Micro/Inverter | `VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analyse_vd_inv_app_<VERSION>.md` |

### What must be updated

1. **Binary fingerprint (§1):** New version, size, function count
2. **Version history:** Add a new changelog entry (see format below)
3. **Affected sections:** If, for example, the SOC algorithm has changed → update §3/4
4. **Function list (§12 for BMS):** Enter new/changed/removed functions

### Changelog format

```markdown
### vOLD → vNEW (date)

**Summary:** Short description of the main changes

**Changed functions (X):**
- `Function_Name` (similarity X%) — What changed
- ...

**New functions (X):**
- `New_Name` @ `0xADDRESS` (XB) — Description

**Removed functions (X):**
- `Old_Name` — No longer present

**String changes:**
- New: "new string" — Meaning
- Removed: "old string"
```

---

## Naming conventions

Function names follow the scheme `PREFIX_Description`:

| Prefix | Meaning | Example |
|---|---|---|
| `APP_` | BMS application logic | `APP_Get_Protection_Status_Byte` |
| `HAL_` | Hardware abstraction | `HAL_GPIO_Write_Pin` |
| `LIB_` | Library/utility | `LIB_CRC16_Modbus_Calc` |
| `INIT_` | Initialization | `INIT_All_Peripherals` |
| `CLI_` | Letter-Shell debug | `CLI_Parse_Arguments` |
| `KA495XX_` | BMIC SPI driver | `KA495XX_Read_CellVoltages_And_Temps` |
| `SOC_` | State of charge | `SOC_Algorithm_Orchestrator` |
| `CAN_` | CAN bus protocol | `CAN_TX_BMS_Status_Reporter` |
| `RS485_` | RS485 communication | `RS485_Register_Read_Handler` |
| `I2C_EEPROM_` | EEPROM access | `I2C_EEPROM_Read_WithMutex` |
| `Protect_` | Battery protection | `Protect_Update_NTC_MinMax` |
| `NTC_` | Temperature sensors | `NTC_Calculate_Temperature` |
| `OTA_` | Firmware update | `BMS_OTA_Upgrade_StateMachine` |
| `RTOS_` | FreeRTOS kernel | `RTOS_vPortExitCritical` |
| `SoftFloat_` | Software float | `SoftFloat_Double_To_Int_Low` |
| `FATFS_` | File system | `FATFS_Find_Dir_Entry` |
| `BMS_` | BMS-specific logic | `BMS_Config_Params_Load_Or_Default` |

Additionally for Control FW:

| Prefix | Meaning |
|---|---|
| `WiFi_` | WiFi/WLAN |
| `ETH_` | Ethernet |
| `Modbus_` | Modbus TCP |
| `Cloud_` | Cloud telemetry |
| `BLE_` | Bluetooth LE |
| `SMR_` | Energy management |
| `MPPT_` | PV tracker |

**Rules:**
- CamelCase after the prefix: `SOC_Calc_Percentage_From_Range`
- Do not leave generic names such as `FUN_xxxxxxxx` in place
- For library functions (FreeRTOS, printf etc.) use the original name
- Keep compiler intrinsics: `__adddf3`, `__muldf3`, `__aeabi_*`

---

## Cleanup after the diff

```
Tool: ghidra:diff-delete-session
  sourceProgramPath: <old FW>
  destinationProgramPath: <new FW>
```

Diff sessions are not deleted automatically — clean them up manually after completing the analysis.

---

## Standing checks for every new firmware version

Report these explicitly at the top of every version-diff report (state "unchanged" or the old → new value; never skip one silently, say if something could not be verified):

1. **Modbus registers**: added/removed registers, type/scale/count changes, served range (decode the `.data` descriptor table and diff it).
2. **BLE**: command table, response layouts, new/removed commands.
3. **AC-side feed-in limit**: maximum AC export/feed-in power and its clamping (2500 W on the Venus D; 1500 W / classes 800/1200/1500 W on the Venus A).
4. **Firmware-update path**: is a BLE OTA still possible after installing the version (new signature/type checks, removed BLE OTA, server-only updates)?
5. **Cloud endpoints**: domains, ports, paths, certificate pinning (hamedata.com, marstekcloud.com, MQTT hosts).
6. **Pack capacity (BMS and Control)**: the maximum number of battery packs the firmware really handles. Check (a) the bound in the BMS per-pack telemetry sender `CAN_TX_PerPack_10Msgs` (`pack_id < N`: D BMS 118 = `< 10` i.e. packs 1..9, A BMS 1105 = `< 7` i.e. packs 1..6), (b) the pack/slot limits in the master logic (`g_highestPackIndex`, slot assignment, per-pack record arrays and their stride), (c) the number of per-pack records in the Control (`Protocol_AA_CommandDispatch`: unit < 9; 7 records on D/A Control) and the Modbus per-pack blocks 34x00–34x33 (7 packs), (d) the cell count per pack (16 on D, 13 on A) and which cell/NTC registers stay 0, (e) whether the Suite per-pack entities (`battery_1…7_*`) still match. A pack whose frames are never sent reads 0 over Modbus although its registers exist.
Known values (extend this table for every new version):

| Image | `CAN_TX_PerPack_10Msgs` bound | Packs sent | Cells per pack |
|---|---|---|---|
| D BMS 117.7 | `pack_id < 10` | 1..9 | 16 |
| D BMS 118 | `pack_id < 10` | 1..9 | 16 |
| A BMS 1105 | `pack_id < 7` | 1..6 | 13 |
| A BMS 110 | `pack_id < 7` | 1..6 | 13 |
| A BMS 115 | `pack_id < 7` | 1..6 | 13 |
| E3 BMS 115 | to be checked | – | – |

Counting note: `pack_id` is one-based (the condition is `pack_id != 0 && pack_id < N`, records are indexed with `pack_id - 1`), so `< 7` means packs 1..6 (six packs) and `< 10` means packs 1..9 (nine packs); `pack_id` 1 is the master's own pack.

The Control keeps 7 per-pack records on D and A (Modbus 34000–34699), so on the A a 7th pack reads 0 over Modbus even though its registers exist.

7. **Diff table**: per-version diff table (own file, EN + DE), one row per change, old → new value and where in the code (function + address).

8. **Re-check of non-functional registers**: every register classified FUNCTIONLESS, NO_OP, UNKNOWN (open) or CONDITIONAL, and every DEV register that stays DEV, must be re-verified for each new version, because the reason may disappear (a receiver gains a handler, a frame gets a sender, a constant becomes a real value). The list per device is generated from the register CSV (`Scripts/list_recheck_registers.py D|A|E3`); the diff report states per register "still dead" / "now functional" / "changed" (recipes below). Registers that were working must of course be re-checked as well through the normal descriptor/handler diff (point 1).

### Re-check recipes for non-functional registers

| What made the register dead | Re-check in the new version |
|---|---|
| Control `Write_Handler` has no case or returns an exception (41001, 44000/44001, 45013–45019 reserved) | diff the Write_Handler switch (new cases or return values), the descriptor table and `Modbus_Dispatcher` range checks |
| Receiver filters or ignores the CAN frame (D/E3 micro class 2/3 frames, A micro 0x18, 0x30–0x3E, 0x40–0x43, 0xFB; BMS class-3 codes; 46500–46544, 47001) | diff the FDCAN filter setup and the dispatcher case lists of Micro and BMS (new `case` values), and the Control's sender functions for those func codes |
| Nobody sends the frame that fills the value (38000–38014 = frames 0x40–0x43, 30214 = PGN 0x1007AA/0x1807AA, 0x1851, CAN id 0x00210100, 45011 = 0x100601AA) | grep all firmware images (Micro, BMS, Control) for the sender (`CAN_Build_Arbitration_ID` with that func/ID) and the Control's receiver table (`g_canAaDispatchTable`: decode the `.data` image) |
| The sender hard-codes or never writes the field (34x17 / 30106 = 0x1841 bytes 6–7, warning word 36000, cells 14–16, error bits that no code sets, SOH words 0x1852) | diff the sender function (`CAN_TX_PerPack_10Msgs`, telemetry builder) and every writer of the source global (grep the global name and its address) |
| Pack or cell capacity bound (pack 7 on the A, `pack_id < N`) | re-run the pack-capacity check (standing check 6) |
| Control-local effect only / value discarded (47400, 46500 cache without consumer, `g_*` variables that nothing reads) | grep the global for readers (literal-pool references included); check new tasks/timers |
| Value only valid after a trigger (30211, 30213, 37001–37003, 30030–30035) | re-check that the trigger still exists and that the value is not now refreshed continuously |
| Counterpart without available firmware (class-2/MPPT node on the D, A/E2 devices): status UNKNOWN (open) | check whether a firmware for that unit has become available; otherwise stay UNKNOWN and DEV |

## Checklist

- [ ] New binary imported and analyzed
- [ ] Diff session created, summary checked
- [ ] Labels/markup transferred automatically (confidence >= 0.95)
- [ ] Remaining matches checked manually and transferred
- [ ] New functions analyzed and labeled
- [ ] String and data changes checked
- [ ] Changelog entered in doc file
- [ ] Affected doc sections updated
- [ ] Function list updated
- [ ] Diff session deleted
- [ ] Standing checks 1–8 (Modbus, BLE, feed-in limit, update path, cloud, pack capacity, diff table, re-check of non-functional registers) answered
- [ ] Ghidra project saved (`ghidra:checkin-program`)
