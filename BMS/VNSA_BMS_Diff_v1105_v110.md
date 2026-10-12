# Diff Table BMS v110.5 (1105) → v110 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_BMS_Diff_v1105_v110_de.md](VNSA_BMS_Diff_v1105_v110_de.md)

**As of:** 2026-10-11. Both images with base 0x08002800 in Ghidra, fully named (1105: 698 functions, 110: 697).
Addresses: 1105 → 110. **Main documentation:** [VNSA_BMS_FW_v110_Full_Analysis.md](VNSA_BMS_FW_v110_Full_Analysis.md),
predecessor analysis [VNSA_BMS_FW_v1105_Full_Analysis.md](VNSA_BMS_FW_v1105_Full_Analysis.md).

Function pairs via identical names (697), compared instruction by instruction with call targets and globals by name,
referenced strings and immediate values; literal-pool values compared separately on the raw images (pointer/BL-masked):
**15 functions changed in behaviour or constants**, **5 changed only in relocated table/literal addresses**,
**0 only in 110**, **1 only in 1105** (`SoftFloat_Double_OrderedCompare_Rev`).

## Which version is older?

| Indicator | 1105 | 110 |
|---|---|---|
| Version string (shell banner) | `Version: 110.5` | `Version: 110.0` |
| Build date | May 11 2026 09:47:24 | May 20 2026 13:59:48 |
| Cloud OTA record | edition 3316, upload folder 20260512 | edition 3430, upload folder 20260520 |
| FW tag in the pack record (`g_pack0FwVersionTag`, Modbus 30204/34010/37012) | 0x451 = 1105 | 0x44C = 1100 |
| Code base | A 109 + PV-rescue/low-voltage changes | **A 109 (Dec 26 2025) + 40 A cap** |

**Result:** 110 is the *newer* release (built nine days later, higher OTA edition), but it is **not** a successor of
1105. 110 is byte-identical to A BMS 109 (`109_bms_VNSA-0_20251226103854b52c66481.bin`, build Dec 26 2025) except for
the 40 A cap, the version constants and the build/version string (334 differing bytes in total, see below). 1105
("110.5") is a parallel beta branch on top of 109; none of its own changes were merged into 110. The number 1105 is
"110.5" × 10, not a version above 110.

110 against 109 (raw byte comparison, same length, same layout):

| Location | 109 | 110 |
|---|---|---|
| `Master_Build_ChargeCurrentLimit` / `Master_Build_DischargeCurrentLimit` (4 immediates) | 0x1F4 = 500 (50.0 A) | 0x190 = 400 (40.0 A) |
| `PerPack_Struct_Builder` FW tag (`movs` → `movw`, function 2 bytes longer, following code shifted) | 0x6D = 109 | 0x44C = 1100 |
| Config version tag `g_cfgVersionTag` (`BMS_Config_Params_Load_Or_Default`, `_Factory_Reset`) | `'m'` (0x6D) | `'n'` (0x6E) |
| RS485 version byte (`RS485_Dispatcher` op 0x05, `RS485_Pack_Telemetry_Response`), `g_sh36735InitConst110` | 0x6D | 0x6E |
| Build date / version string | Dec 26 2025 10:34:57, `109.0` | May 20 2026 13:59:48, `110.0` |
| Image trailer word 0x0801BFFC | `B5 82` | `0F 90` |

## Verification checklist (standing checks)

| Question | Result |
|---|---|
| Modbus registers | The BMS has no Modbus table; the Control serves its CAN values. **Changed values:** 32106/35111 (system charge current limit) max 40.0 A instead of 50.0 A and 0 outside the master run state (1105: 50.0 A); 32107/35112 (discharge current limit) max 40.0 A; 35110 (charge voltage limit) constant 46.8 V (1105: pack voltage + 2.0 V, max 46.8 V after the CAN clamp); 30204/34010/37012 (BMS version) read **1100** (1105: 1105). Per-pack charge current in 0x1821 is no longer forced to 10.0 A below 35.00 V. Register list, types and scales unchanged (Control-side). |
| BLE | Not applicable: the BMS has no BLE. Unchanged. |
| AC feed-in limit | Not in the BMS. Indirect effect: the battery-side limit is 40.0 A; at the nominal 13S voltage (41.6 V) that is ~1660 W, above the A micro limit of 1450 W, so the AC power is normally not limited by it; at low pack voltage (~36–37 V) 40 A ≈ 1450–1500 W (inference, not measured). |
| Update path | Unchanged: CAN (`BMS_OTA_Flash_Writer`) and RS485 (`RS485_OTA_Flash_Handler`) paths, staging copy 0x0801E000, image tag `'BMS2'`, CRC16. 1105 → 110 keeps the config version tag `'n'`, so the stored configuration survives; 109 → 110 changes `'m'` → `'n'` and loads the defaults (code fact; field behaviour not observed). |
| Cloud endpoints | Not applicable: the BMS has no network access. |
| Pack capacity | Unchanged: `CAN_TX_PerPack_10Msgs` bound `pack_id != 0 && pack_id < 7` (packs 1..6, records `pack_id - 1`), same in `CAN_TX_PerPack_SocDiag`; per-pack record stride 0x60, 13 cells (`INIT_SH36735_Subsystem` unchanged), 16 cell slots in the frames (14–16 = 0). |
| Cell count | 13 (unchanged). |
| Non-functional registers | All A registers classified FUNCTIONLESS/NO_OP/UNKNOWN/CONDITIONAL keep their status with BMS 110: pack 7 (34600–34633) still never sent (`< 7`), cells 14–16 still 0, 0x1841 bytes 6–7 still 0 (34x17/30106), 0x1851/0x1852 writer still absent, 45031 still impossible (`g_chargeStatusCode` only holds the EEPROM 0xFC02 byte read at boot, the BMS writes 0xA5 there; the 0x00AA pattern that 45031 waits for is never produced), 45011/47001 still without receiver. |

## Release note → code

| Release note item | Code finding |
|---|---|
| "Maximum charge-discharge current changed to 40 A" | `Master_Build_ChargeCurrentLimit` and `Master_Build_DischargeCurrentLimit`: cap 500 → **400** (0.1 A units). Per-pack limits (`Max_Charge_Current_Calc`, `Max_Discharge_Current_Calc`, 50.0 A table maximum) are unchanged; the 40 A cap applies only to the system limits sent in 0x1803AA01. |

## Changes 1105 → 110

| Area | 1105 | 110 | Location (1105 → 110) |
|---|---|---|---|
| System charge current cap | 500 (50.0 A) | **400 (40.0 A)** | `Master_Build_ChargeCurrentLimit` 0800BCF0 → 0800BC20 |
| System charge current outside master run state 2 | 50.0 A, or 0 with status code 4 while `g_socStateFlags` bit5 is set | **0** | same |
| System charge voltage limit | own pack voltage + 2.0 V, max 48.0 V (PF3 clamps to 46.8 V) | **fixed 46.8 V** (0x1D4) | same |
| `g_chargeStatusCode` writes | 2 offline pack, 3 SOC 100 %, 4 SELF flag | none (only the boot value from EEPROM 0xFC02) | `Master_Build_ChargeCurrentLimit`, `BMS_AggregateSystemPackStats` 0800B984 → 0800B8C0 |
| System discharge current cap | 500 (50.0 A) | **400 (40.0 A)** | `Master_Build_DischargeCurrentLimit` 0800BDA4 → 0800BC90 |
| CAN 0x1803AA01 charge current while the own pack voltage (`g_pack0Voltage_x10`) is below 35.00 V | forced 100 (10.0 A) | system limit, no override | `CAN_TX_PF3_ChargeDischargeLimits` 08007C5C → 08007C1C |
| CAN 0x1821AA00+n charge current while the own pack voltage is below 35.00 V | forced 100 (10.0 A) for every pack | the pack's own limit | `CAN_TX_PerPack_10Msgs` 080083C0 → 08008338 |
| PV-rescue CAN id 0x03410110 | sets `g_pvRescueRequestFlag` (byte 1 > 8) | **not handled** (variable removed) | `CAN_RX_Handler` 08008264 → 080081FC |
| 0x100201AA in the system command handler | additionally calls `CAN_CMD_02_Handler` | only in `CAN_RX_Handler` (as before) | `CAN_RX_System_Command_Handler` 0800FEB8 → 0800FD80 |
| Cell UVP level 2 handling (1 s tick) | 120 s without PV-rescue request → `System_Reset_And_Halt_WithCurrentCheck` | **D logic:** own MOSFETs only below 2.0 A, standalone pack → fault mode 3 after 200 s (`g_protCommTickCounter200`) | `BMS_ProtectionAndCommTick` 080096E4 → 0800963C |
| UVP level 2 stage A (≥ 0 °C) | 2800 mV, trip = protection (Protect1 bit5, SOC reset to 0 %) | **3000 mV, trip only sets `g_protState5` bit2** (as D) | `Cell_OVP_UVP_TempCompensated_Protection` 0800CE7C → 0800CD68 |
| Min-cell sensor-fault trip (< 1500 mV) | only above 39.00 V pack voltage | at any pack voltage | same |
| Master MOSFET gate without SELF flag | relays forced off only above 39.00 V | relays forced off at any voltage | `OwnPack_MOSFET_Driver_MasterVariant` 08007180 → 08007150 |
| Low-voltage balancer criterion | SOC < 8 %, Protect1 bit0 **or min cell < 2800 mV** | SOC < 8 % or Protect1 bit0 | `Pack_LowVoltage_Balancer` 08007F00 → 08007EAC |
| HW overcurrent comparator cross-check | ±70.0 A | **±3.0 A** (as D) | `HW_OvercurrentAndSensorProtection` 0800B160 → 0800B09C |
| SOC "empty" threshold (≥ 0 °C) | 2800 mV | **2750 mV** (as D); function renamed `SOC_Select_Threshold_2420_2750` | `SOC_Select_Threshold_2420_2800` 080135C4 → 08013474 |
| FW tag in the pack record | 0x451 (1105) | 0x44C (1100) | `PerPack_Struct_Builder` 0800C934 → 0800C820 |
| Max charge current, table branch | extra double compare against 0.0 (no effect found) | without it; `SoftFloat_Double_OrderedCompare_Rev` removed | `Max_Charge_Current_Calc` 080139A8 → 08013858 |
| Version string / build | `110.5`, May 11 2026 | `110.0`, May 20 2026 | login banner (`CLI_LoginUser`) |
| RAM | `.data` 0x3A0 B; `g_uvpLvl2ShutdownTimerS`, `g_pvRescueRequestFlag` | `.data` 0x398 B (both variables absent); 0x20002858 is `g_protCommTickCounter200` again (1105: `g_standaloneModeFlag`); stack top 0x2000C958 | region table 0x0801B9D4 → 0x0801B860 |

## Unchanged (explicitly checked)

- Everything else: the other 677 of 697 functions are instruction-identical (calls, globals, strings and immediates
  compared by name/value); the "never fully charged" charge
  derating (`Max_Charge_Current_Calc`, EEPROM 0xFC02) is present in 110 (and 109), so it is **not** a 1105 feature.
- CAN frame set and IDs, opcodes 0x0A–0x0C, `0x100501AA` heartbeat reply, letter-shell (52 entries, user `BLUE`),
  RS485 opcode set, SH36735 driver, SOC/SOH tables, OTA paths and the `'BMS2'` tag.
- Discharge voltage limit 39.0 V, PF3 charge-voltage clamp 46.8 V, capacity factor 416.

## Cosmetic differences (no behavioural change)

`RTOS_CreateAllTasks`, `__main`, `__dtoa_engine`, `_printf_core_IntEntry`, `prvProcessReceivedCommands`: only
relocated literal/table addresses (stack top, switch table, FreeRTOS assert string).
