# Diff Table BMS v110 → v115 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_BMS_Diff_v110_v115_de.md](VNSA_BMS_Diff_v110_v115_de.md)

**As of:** 2026-10-11. All three images with base 0x08002800 in Ghidra, fully named (110: 697 functions, 110.5: 698,
115: 727). Addresses: 110 → 115. **Main documentation:** [VNSA_BMS_FW_v115_Full_Analysis.md](VNSA_BMS_FW_v115_Full_Analysis.md),
predecessor analyses [VNSA_BMS_FW_v110_Full_Analysis.md](VNSA_BMS_FW_v110_Full_Analysis.md) and
[VNSA_BMS_FW_v1105_Full_Analysis.md](VNSA_BMS_FW_v1105_Full_Analysis.md).

Function pairs via identical names, compared instruction by instruction with call targets and globals by name,
referenced strings and immediate values (RAM names of 115 reviewed first, see the full analysis §2.2):

| Comparison | New in 115 | Only in predecessor | Changed (incl. address-only) |
|---|---|---|---|
| 110 → 115 | 30 | 0 | 61 |
| 110.5 → 115 | 31 | 2 (`SoftFloat_Double_OrderedCompare_Rev`; `SOC_Select_Threshold_2420_2800` only renamed in 110) | 56 |

## Code base

| Indicator | 110 | 110.5 | 115 |
|---|---|---|---|
| Version string / build | `110.0`, May 20 2026 | `110.5`, May 11 2026 | `115.0`, Sep 1 2026 |
| OTA edition | 3430 | 3316 | 4563 |
| Masked segment coverage of the 115 image | 81.4 % | 82.4 % | — |
| FW tag in the pack record (Modbus 30204/34010/37012) | 0x44C = 1100 | 0x451 = 1105 | **0x73 = 115** |
| Config / RS485 version tag | `'n'` | `'n'` | **`'s'`** |

115 is the first image that merges both A branches: the 110.5 beta code (PV rescue, low-voltage clamps and thresholds)
is contained, the 110/109 parts without the 110.5 charge-status codes and pack-voltage-dependent charge voltage remain,
and new features are added. Coverage against D 118 is 65.7 %, against E3 BMS 115 12.6 %: 115 is not an E3 or D branch.

## Verification checklist (standing checks)

| Question | Result |
|---|---|
| Modbus registers | The BMS has no Modbus table; the Control serves its CAN values. Changed values with BMS 115 (register list/types/scales are Control-side and unchanged): **32106/35111** system charge current limit max **50.0 A** (110: 40.0 A), 50.0 A also outside the master run state unless the SELF flag is set (110: 0); **32107/35112** discharge current limit max **50.0 A**; **35110** charge voltage limit **48.0 V** internally, the CAN frame still carries ≤ 46.8 V (110: 46.8 V); **30204/34010/37012** (and 34x10 of slaves running 115) BMS version **115** (110: 1100); 32106 and per-pack charge current (0x1821) are forced to **10.0 A while the master's own pack is below 35.00 V**; **32105** capacity unchanged; PF4 byte 4 (charge status code) is always 0. SOC registers (32104, 34x02, 37005): same meaning, but the end-of-charge/discharge correction now needs three cells (changed SOC curve near 0/100 %) and the empty threshold is 2800 mV. |
| BLE | Not applicable: the BMS has no BLE. Unchanged. |
| AC feed-in limit | Not in the BMS. Indirect: battery-side limit 50.0 A ≈ 2080 W at 41.6 V, above the A micro limit, so normally not limiting; below 35.00 V own-pack voltage the CAN charge current is 10.0 A (≈ 350 W charge) (inference, not measured). |
| Update path | Unchanged: CAN and RS485 OTA, staging 0x0801E000, tag `'BMS2'`, CRC16. **Config version tag `'n'` → `'s'` and block 0xD4 → 0xD8 bytes: the stored configuration is replaced by defaults after the update** (code fact). Downgrade 115 → 110 likewise resets the configuration. |
| Cloud endpoints | Not applicable: the BMS has no network access. |
| Pack capacity | Unchanged: `CAN_TX_PerPack_10Msgs` (`0x08008DE8`) bound `pack_id != 0 && pack_id < 7` → packs 1..6 (records `pack_id - 1`), per-pack record stride 0x60, `g_highestPackIndex` logic unchanged. |
| Cell count | 13 (`INIT_SH36735_Subsystem` unchanged apart from its version constant; `APP_Get_Pack_Summary_Data` loops over 13 cells). |
| Non-functional registers | Re-checked against the BMS side: pack 7 (34600–34633) still never sent (`< 7`); cells 14–16 still 0; 0x1841 bytes 6–7 still 0 (34x17/30106); writer of the 0x1851/0x1852 words still absent (`g_cfgTailReg*` only from the config block); **45031 still impossible** (PF4 byte 4 is now constantly 0); 45011/47001 still without receiver. New outbound frame `0x1805AA01`: no receiver found in A Control 151. |

## Release note → code

| Release note item (OTA remark, translated) | Code finding |
|---|---|
| 1. Pack switch when the inter-pack SOC difference is > 20 | `PackSelect_SocImbalanceSwitch` `0x08007B98` (new), called by `PackSelect_ModeDispatch` in mode 0: spread ≥ 20 %, power-dependent delay 1200–3000 ticks, charge → lowest SOC, discharge → highest SOC |
| 2. Dynamic correction of the charge/discharge target values | `PackSelect_ChargeStepTarget` `0x08009FF4` (95/100 % rules via new `PackSelect_AllOnlineSocAtLeast`), threshold re-computation in `PackSelect_Charge_StepRotate` / `_Discharge_StepRotate` |
| 3. End-of-charge/discharge correction needs three cells | `APP_Get_Pack_Summary_Data` `0x08014DD8` counts cells ≥ 3500 mV / ≤ 2900 mV; `Charge_SOC_Integration` / `Discharge_SOC_Integration` use "≥ 3 cells" instead of max/min cell |
| 4. Improved charge/discharge MOS recovery | `OwnPack_MosRecovery_SM` `0x08005BA4` (new, PE7 output, CAN `0x1805AA01`), trigger in `OwnPack_MOSFET_Driver_MasterVariant` |

## Changes 110 → 115 (with 110.5 for reference)

| Area | 110 | 110.5 | 115 | Location (110 → 115) |
|---|---|---|---|---|
| System charge current cap | 400 (40.0 A) | 500 | **500 (50.0 A)** | `Master_Build_ChargeCurrentLimit` 0800BC20 → 0800C8C0 |
| System charge current outside master run state 2 | 0 | 50.0 A / 0 with SELF flag (status code 4) | **50.0 A / 0 with SELF flag** (no status code) | same |
| System charge voltage limit | 46.8 V fixed | pack voltage + 2.0 V, max 48.0 V | **48.0 V fixed** (PF3 clamps CAN to 46.8 V) | same |
| System discharge current cap | 400 | 500 | **500** | `Master_Build_DischargeCurrentLimit` 0800BC90 → 0800C93C |
| `g_chargeStatusCode` writes / PF4 byte 4 | boot value only / sent | 2/3/4 / sent | none / **byte 4 = 0** | `CAN_TX_PF4_ProtectWarnings` 08007DA0 → 0800876C |
| CAN 0x1803/0x1821 charge current below 35.00 V own-pack voltage | no clamp | 10.0 A | **10.0 A** | `CAN_TX_PF3_ChargeDischargeLimits` 08007C1C → 080085D4, `CAN_TX_PerPack_10Msgs` 08008338 → 08008DE8 |
| PF2 byte 7 | relay-comm substate 1/2/3 | same | **pack-select ack (2), cleared after sending** | `CAN_TX_PF2_Capacity` 08007B50 → 080084EC |
| CAN 0x03410110 | ignored | `g_pvRescueRequestFlag` = byte 1 > 8 | byte 1 > 8 **or byte 6 == 1** | `CAN_RX_Handler` 080081FC → 08008BD4 |
| CAN 0x1801FFAA byte 3 | ignored | ignored | **`g_emsHeartbeatModeByte`** | same |
| CAN 0x01210103, 0x04410110 | — | — | **new** (inhibit flag; float ratio without reader) | same |
| CAN 0x1805AA01 (out) | — | — | **new**, MOS-recovery progress, from the idle tick | `APP_IdleKeepaliveBroadcast_Tick` 0800E290 → 0800F144 |
| Idle tick period | 500 ms | 500 ms | **50 ms** | same |
| Cell UVP level 2 (1 s tick) | D logic: MOSFETs < 2.0 A, standalone → fault mode 3 after 200 s | 120 s without PV rescue → STOP | **as 110.5** | `BMS_ProtectionAndCommTick` 0800963C → 0800A18C |
| UVP level 2 stage A (≥ 0 °C) | 3000 mV, warning only | 2800 mV, protection | **2800 mV, protection** | `Cell_OVP_UVP_TempCompensated_Protection` 0800CD68 → 0800DBB0 |
| Min-cell sensor-fault trip | any voltage | > 39.00 V | **> 39.00 V** | same |
| Protection during AFE comm fault | evaluated | evaluated | **all counters cleared, not evaluated** (cell, current, temperature) | same, `Current_Protection_Checker` 0800AC6C → 0800B808, `Voltage_Temp_Protection_Checker` 0800C168 → 0800CE54 |
| Pack under-voltage STOP | — | — | **new: < 36.00 V for 1200 ticks → STOP (unless PV rescue)** | `Pack_UnderVoltageShutdown_Check` 0800D590 |
| HW comparator cross-check | ±3.0 A | ±70.0 A | **±70.0 A**; NTC/MOS-temp range checks skipped below 20.00 V | `HW_OvercurrentAndSensorProtection` 0800B09C → 0800BC70 |
| PD8 output | — | — | **new: high below 20.00 V** | `Pack_LowVoltageOutput_PD8_Update` 0800A330 |
| Master MOSFET gate without SELF flag | off at any voltage | off > 39.00 V | **off > 39.00 V**; PE7 off with the other outputs; MOS recovery start | `OwnPack_MOSFET_Driver_MasterVariant` 08007150 → 080078C8 |
| Low-voltage balancer | SOC < 8 % or Protect1 bit0 | + min cell < 2800 mV | **as 110.5** | `Pack_LowVoltage_Balancer` 08007EAC → 08008870 |
| SOC empty threshold (≥ 0 °C) | 2750 mV | 2800 mV | **2800 mV**; function renamed `SOC_Select_Threshold_2420_2800` | 08013474 → 08014624 |
| End-of-charge / end-of-discharge correction | max cell ≥ 3500 / min cell ≤ 2900 mV | same | **≥ 3 cells** | `Charge_SOC_Integration` 08012A9C → 08013C50, `Discharge_SOC_Integration` 08012F10 → 080140C0 |
| Charge step target | 10 % steps | same | **100 % if all ≥ 95 %, 95 % if all ≥ 90 %** | `PackSelect_ChargeStepTarget` 080094C4 → 08009FF4 |
| SOC-imbalance switch | — | — | **new (≥ 20 %)** | `PackSelect_SocImbalanceSwitch` 08007B98 |
| Pack handoff | `PackHandoff_StateMachine` | same | + **SOC switch state machines** when mode byte 2 and PV rescue | `PackHandoff_Tick` 08006A48 → 08007150, new 08006B5C / 08006D0C |
| Enable masks before re-election | kept | kept | **reset to the active pack** | `PackSelect_ChargeEnable_Maintain`, `_MosMismatch_Fallback`, `_Next_AboveSocThreshold` |
| Relay-comm state machine | substate 1/2/3, command flag cleared after 300 ticks | same | **no substate, flag cleared immediately, escalations 300/600 inhibited** by `g_relayCommEscalationInhibit` | `PackSelect_RelayCommStateMachine` 0800B2CC → 0800BEE8 |
| LED relay-comm phases | always | always | suppressed while inhibited | `LED_StatusPatternUpdate` 0800BCF4 → 0800C9C4 |
| Precharge retry | retry counted unless the voltage still drops by > 1.0 V | same | also not counted while the difference metric is > 7 | `OwnPack_Precharge_SM` 08006D84 → 080074D0 |
| Switch outputs | actions 1–6 | same | **+ 7/8 = PE7 on/off** | `HAL_GPIO_Control_Switch` 08006508 → 08006890 |
| Pack status flags | bits 0–5 | same | **+ bit6 = PB2, bit7 = PE7** | `PerPack_Struct_Builder` 0800C820 → 0800D63C |
| STOP mode | STOP + reset | same | **RTC wake-up timer armed before STOP (nominal 6 h, LSI)**, wake flag in BKP_DAT2, back to STOP after 50 ticks | `System_Reset_And_Halt_WithCurrentCheck` 08004A40 → 08004B1C, 16 new RTC/BKP/PWR/RCC functions |
| Boot | — | — | **`BKP_ReadAndClearRtcWakeFlag`**, PC8 setup, PC5/PB0/PB2/PB5 re-init after an RTC wake | `System_Init_Main` 08004670 → 08004748, `GPIO_Init_All_Pins` 080040B8 → 08004148, `GPIO_DeInit_And_CAN_NVIC_Init` 080043D0 → 0800449C |
| Configuration block | 0xD4 B, tag `'n'`, checksum 0..0xCF | same | **0xD8 B, tag `'s'`, checksum 0..0xD3 (includes itself)**, new field 0xD4 = 120 | `APP_Config_Checksum_And_Save` 0800AC48 → 0800B7E4, `BMS_Config_Params_Load_Or_Default` 08009CAC → 0800A840, `RS485_RegisterWrite_DefaultGroup` |
| SOC calibration log | 50 × 20 B at 0x400 | same | **5 × 20 B at 0x440** | `APP_EventLogGetSlotAddrs`, `APP_FormatEventLog_IfNeeded`, `SOC_EepromLoadCalibration` |
| Version constants | 0x6E, 1100 | 0x6E, 1105 | **0x73, 0x73** | `RS485_Dispatcher`, `RS485_Pack_Telemetry_Response`, `INIT_SH36735_Subsystem`, `BMS_Config_Params_Factory_Reset`, `PerPack_Struct_Builder` |
| Library | — | — | `SoftFloat_Int_To_Double`, `SoftFloat_Double_To_Float` added (as in E3 115); `__dtoa_engine` differently compiled | — |
| RAM | `.data` 0x398 B | 0x3A0 B | **0x3E0 B**, `.bss` 0x9DC8 B, stack top 0x2000C9A8, 30 new globals | region table 0801B860 → 0801CA30 |

## Behaviour of an A system after the update 110 → 115 (from the code)

- Charge/discharge limits up from 40.0 A to 50.0 A, charge voltage limit 48.0 V internally.
- Configuration reset to defaults at the first boot (tag change); because of the self-inclusive checksum possibly at
  every boot (open point 1 of the full analysis).
- A pack that falls into cell UVP level 2 sleeps after 120 s unless the inverter sends the PV-rescue frame; a pack below
  36.00 V sleeps after 1200 ticks; sleeping packs wake up via the RTC about every 5–6 h and go back to sleep.
- Pack switching additionally at ≥ 20 % SOC spread; switching thresholds adapt near full.
- SOC end corrections later (three cells instead of one), empty threshold 2800 mV.

## Unchanged (explicitly checked)

CAN frame set apart from the rows above, opcodes 0x0A–0x0C, letter-shell (52 entries, user `BLUE`), RS485 opcode set,
SH36735 driver and balancing, SOC/SOH and derating tables (byte-identical), OTA paths and `'BMS2'` tag, FreeRTOS tasks,
discharge voltage limit 39.0 V, capacity factor 416, never-fully-charged charge derating.

## Cosmetic differences (no behavioural change)

`RTOS_CreateAllTasks`, `__main`, `_printf_core_IntEntry`, `prvProcessReceivedCommands`, `prvHeapInit`,
`BMS_OTA_Flash_Writer_FinalizeTail`, `CAN_CMD_Pack_Activation`, `Max_Charge_Current_Calc` (float clamp moved to the
integer result, same values), `PackMode_Dispatcher` / `PackMode_Active_MasterClaim` (only the removed substate writes).
