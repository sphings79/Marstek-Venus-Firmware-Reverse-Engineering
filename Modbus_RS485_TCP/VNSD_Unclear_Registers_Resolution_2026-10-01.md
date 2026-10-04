# VNSD – Resolution of the formerly unclear Modbus registers (Venus D, 2026-10-01)

> 🇩🇪 Deutsche Version: [VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md](VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md)

Result of a complete code analysis for every register that was missing in the Marstek Modbus Suite integration (`d.yaml`) and was not a pure alias of an included register. Scope: Venus D, Control 147 / 149 / 149.2 / 150 / 151, Micro 115 / 116, BMS 117.7 / 118, FC41D 202409 / 202512. Venus A and E3 follow in separate documents. Everything is derived from the decompiled code (`Dekompilate/`, local) – no guesses; where the code cannot prove a statement this is said explicitly.

## 1. Method

1. Start at the Modbus register: handler case in `Write_Handler` / read descriptor (all five Control versions, decimal and hex switch forms).
2. Follow the value to its final consumer or origin across devices: Control → CAN (frame class = ID bits 12–15, FDCAN filters, dispatcher case, global, consumers) → Micro / BMS; Control → UART → FC41D; GPIO; EEPROM.
3. **Functionless check** for every target found: value never read, branch unreachable, receiver filters the frame, command overwritten at once, mode gate never satisfied.
4. Independent adversarial verification of the 21 most consequential claims by two further analysts working only from the code: none refuted, one partly wrong, several wordings tightened (included below).

Status values: **FUNCTIONAL** = has the documented effect · **CONDITIONAL** = only in a special mode / after another write · **NO_OP** = accepted but no effect · **FUNCTIONLESS** = target exists but never acts (filtered, never read, no receiver) · **UNKNOWN** = target found, meaning not provable.

Counts (179 registers): FUNCTIONAL 60, FUNCTIONLESS 49, CONDITIONAL 39, UNKNOWN 23, NO_OP 8.

## 2. Key findings

- **41001 is not implemented on the Venus D Control** (no case in any version; read and write return Modbus exception 2, no side effect). The Suite button `factory_reset` (41001) therefore does nothing. The Micro has its own 41001 (clear statistics, Micro reboot) but it is reachable only through the Micro's own RS485 Modbus slave, not through the Control.
- **41000**: `0x55AA` = clean Control reboot; after boot the Control sends CAN command 0x05, which also reboots the Micro (this is why 41000 restores normal operation after 45000 = `0x55EE`). `0xAA11` = full factory reset of Control, Micro and BMS (also deletes FC41D cloud TLS certificates and the cloud identity block). It first writes RS485 address 17, but the reset reloads the defaults at the next boot (address becomes 1).
- **40000** starts the Micro's battery calibration (cycle) test (charge to 100 %, discharge below 30 %, recharge, up to 2.5 kW); the Micro ignores Control power setpoints meanwhile. Dangerous – never expose.
- **45000** (`0x55EE` / `0x55BB`) writes the Micro's `g_workMode` (12 = factory mode / 1 = manual) through the CAN 0x50 handler and sends class-2 "off". It does not reduce idle consumption (live-tested). A Control boot restores it.
- **The class-2 CAN node on a Venus D is a separate unit, most likely the MPPT/PV controller** (neither Micro 115/116 nor the BMS can send class-2 frames, yet 30028/30036 and `mppt_ver` show live values). Commands 45020–45024 reach it; what they do there cannot be determined without its firmware.
- **"Battery Pack Recovery" in the Marstek app is pack address coding, not a recovery charge** (proven by decompiling app 1.6.76): the button sends BLE `73 06 23 40 00 16` / MQTT `cd=40,number_as`, i.e. CAN `0x100101AA` byte 3, which arms the BMS coding handshake. Modbus **45029** is the Modbus-side equivalent (it also sends the PB10 pulse train). Details: `../BMS/VNSD_Battery_Pack_Recovery_Firmware.md`.
- **45027** is NOT harmless (older docs): class-3 frame to the BMS master pack – immediate BMS reboot (and EEPROM 0x202 = 0xFF on a pack that became master at runtime). **45028** = BMS factory reset + reboot of all packs. Never expose.
- **45001 is not a BMS reset** (older docs): it triggers a Micro EEPROM write self-test (result in 30213). **45012** is a front-panel LED lamp test, not a power setpoint. **45010** is the real manual power setpoint (+ discharge / − charge, mode 1 only, no Control-side protections).
- **Wi-Fi 41500–41631** work as a "set Wi-Fi credentials" path to the FC41D (`AT+QSTAAPINFODEF`), identical in all versions. Two registers per two characters, the SSID registers read back the SSID, the password registers read 0. Vendor bug: the password area EEPROM 0x420–0x45F overlaps 0x441 (`g_httpCloudServerType`).
- **46000**: `0x5100` turns the RS485 Modbus RTU port into the CLI shell (persistent, EEPROM 0x900); the OTA codes delete the Modbus TCP / Wi-Fi / cloud tasks and stop the inverter. **46500–46544** and **47001** are FUNCTIONLESS on the D (no receiver). **47400** is a no-op.
- **38000–38014**: no sender among the known firmware (Micro 115/116 and BMS 117.7/118 never send frames 0x40–0x43; the block is absent in v147), but the class-2/MPPT-type unit and A/E2-type hardware cannot be excluded: status **UNKNOWN (open)**, keep as DEV (disabled, never polled). A read in that range makes the Control broadcast four CAN frames per second until reboot.
- **34009 / 34109 / …** are not "reserved": they are the per-pack BMS warning bitmask (value 2 = cell under-voltage; threshold 3100 mV at ≥ 0 °C, 2750 mV below 0 °C; second stage 3000 / 2550 mV). **32112** = BMS lock active; **32107** (= 35112) = BMS discharge current limit (0.1 A).
- **32101** is the signed sum of the pack currents (0.1 A) but the Modbus read serializer divides unsigned (charging reads whole amps, discharging garbage) – do not expose.
- Side finding (all versions): every Modbus TCP exception reply has MBAP length 4 but only 3 bytes follow, so clients wait for a missing byte (see `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20.md`).
- Side finding: `Idle_Watchdog_AutoReboot` sends BMS opcode 1 on CAN id 0x100101AA after 3600 s idle; on non-master packs this falls through to a BMS factory reset.

## 3. Register table

Columns: `status`, `target` (device: consumer) and the recommendation for the integration. Full details (read/write behaviour, data flow, versions) are in the CSV columns `status_D`, `ziel_D`, `leseeffekt_D`, `befund_D`, `gefahr_D`, `suite_empfehlung_D`, `befund_konfidenz_D` of `VNSD_Register_Map_Final_generated.csv`. Consecutive registers with identical result are merged into one row.

### Maintenance and reset registers (40000–41001)

| Register | Name | Status | Target | Recommendation |
|---|---|---|---|---|
| 40000 | `burnin_cycle_test_start` | FUNCTIONAL | Micro: Micro autonomous battery charge/discharge cycle test (g_workMode=4, g_battCalState) | Do NOT expose (at most a DEV-only button 'Start burn-in / battery cycle test' with a warning). Progress would be visible in 37001 and 30010 (=4). |
| 41000 | `system_command` | FUNCTIONAL | Control: 0x55AA: Control + Micro reboot. 0xAA11: factory reset of Control, Micro and BMS (+ FC41D certificates) | Keep 'reset_device' as button 'Restart system' (0x55AA). It reboots Control and Micro cleanly. Optional separate disabled-by-default button 'Factory reset (Control+Inv... |
| 41001 | `unhandled_register_41001` | FUNCTIONLESS | none: none (exception 2) | Remove the 'factory_reset' button for Venus D: it does nothing and is mislabelled. A real factory reset is 41000=0xAA11. |
| 41500–41515 | `wifi_ssid_00 … wifi_ssid_15` | CONDITIONAL | FC41D: FC41D Wi-Fi STA config (AT+QSTAAPINFODEF), Control EEPROM 0x400 on success - only via the password commit | Do not expose as entities. Use only inside an optional write-only 'set Wi-Fi' service (advanced/DEV). An SSID read-back diagnostic sensor is possible but privacy-sensi... |
| 41600–41631 | `wifi_password_00 … wifi_password_31` | FUNCTIONAL | FC41D: FC41D Wi-Fi STA credentials (AT+QSTAAPINFODEF -> FC41D flash ids 0x11111104/05 + association); Control EEPROM 0x400/0x420 after confirmed co... | Not as entities. Optional write-only service 'set_wifi(ssid,password)' behind an advanced/DEV option, with strict validation; never read back. |

### MPPT / inverter / configuration read registers (30028–30214, 37000–37003)

| Register | Name | Status | Target | Recommendation |
|---|---|---|---|---|
| 30028 | `mppt_battery_side_voltage` | FUNCTIONAL | none: origin: MPPT controller battery-side voltage measurement; consumers: shell/MQTT only | Add as DEV (disabled by default): battery-side voltage of the MPPT-type unit (not the PV strings, which are already in the Suite as mppt1..4_voltage); origin unit has ... |
| 30029 | `mppt_battery_side_current` | FUNCTIONAL | none: origin: MPPT controller battery-side current (own converter current, not pack current) | Add as DEV (disabled by default): battery-side converter current of the MPPT-type unit (signed, 0.1 A; not the PV strings and not the pack current); origin unit has no... |
| 30030 | `mppt_reply_0xfe_word` | CONDITIONAL | none: none (value only stored for Modbus read) | Keep as DEV (disabled by default): reply words of the class-2/MPPT-type unit, only valid after a 45023/45024 write; assignment impossible without its firmware. |
| 30031–30035 | `mppt_reply_0xff_slot1_w0 … mppt_reply_0xff_slot2_w0` | CONDITIONAL | none: none | Keep as DEV (disabled by default): reply words of the class-2/MPPT-type unit, only valid after a 45023/45024 write; assignment impossible without its firmware. |
| 30036 | `mppt_base_voltage` | FUNCTIONAL | BMS: origin MPPT; consumer: 'MPPT present & error-free' bit in 0x1801FFAA -> BMS master-enable handshake | Keep as DEV (disabled by default): constant ~2.19 V (0.01 V) reference of the MPPT-type unit; a non-zero value (with g_mpptError == 0) sets the master-enable byte in C... |
| 30211 | `inverter_hw_fault_input` | CONDITIONAL | Micro: origin: Micro GPIO PB1 (hardware-fault comparator input) level | DEV only, paired with a 45006 'query' button; name 'Inverter hardware-fault input (PB1)' |
| 30213 | `inverter_eeprom_test_ack` | CONDITIONAL | Micro: origin: Micro EEPROM write/readback test ACK (always 1) | Do not expose (value is the constant ACK of the 45001 Micro EEPROM write self-test; 45001 is NOT a BMS reset). |
| 30214 | `bms_self_check` | UNKNOWN | none: none on Venus D (no sender) | Keep as DEV (disabled by default): needs a status frame that neither BMS 117.7/118 sends; sender unknown (A/E2-type BMS or other unit) - UNKNOWN, re-check for A/E3. |
| 37000 | `rs485_slave_address` | FUNCTIONAL | Control: RS485 RTU address filter | Diagnostic sensor 'RS485 slave address' (redundant if 41100 is exposed) |
| 37001–37003 | `inverter_calibration_step … inverter_calibration_counter_step3` | CONDITIONAL | Micro: origin: Micro battery-capacity-calibration state machine | DEV only (do not expose 40000) |

### BMS read registers (32000–34999)

| Register | Name | Status | Target | Recommendation |
|---|---|---|---|---|
| 32101 | `bms_battery_current` | FUNCTIONAL | BMS: Origin: BMS sum of online pack currents (0.1 A signed) | Do not expose. Keep the existing sum of per-pack currents 34x01 (identical quantity, unscaled, no bug). |
| 32107 | `bms_discharge_current_limit` | FUNCTIONAL | BMS: Control discharge power cap = pack voltage x this limit | Expose one of 32107/35112 as diagnostic sensor 'BMS discharge current limit' (A, x0.1), mirror of existing 32106. |
| 32108 | `bms_max_cell_temperature` | FUNCTIONAL | BMS: Origin: BMS max NTC temperature | Do not add (duplicate of 35010). |
| 32112 | `bms_lock_active` | FUNCTIONAL | BMS: BMS persistent lock that blocks charge and discharge | Expose as diagnostic binary_sensor 'BMS lock active' (problem class). |
| 32113 | `bms_factory_mode` | FUNCTIONAL | BMS: BMS factory-mode flag (informational on Control) | Optional DEV/diagnostic 'BMS factory mode'; use 30109 or 32113, not both. |
| 32114 | `encode_test_status` | CONDITIONAL | Control: Control-local status of a 2-pack 'encode' (pack activation) test | Do not expose (live always 0; test-sequence status). |
| 34009 | `pack1_bms_warning_flags` | FUNCTIONAL | BMS: Origin: per-pack BMS warning flags | Expose per pack as diagnostic sensor 'Pack N BMS warnings' (bitmask, decoded attributes); rename DEV 'reserved_5a'. |
| 34109 | `pack2_bms_warning_flags` | FUNCTIONAL | BMS: Origin: per-pack BMS warning flags | Expose per pack as diagnostic sensor 'Pack N BMS warnings' (bitmask, decoded attributes); rename DEV 'reserved_5a'. |
| 34209 | `pack3_bms_warning_flags` | FUNCTIONAL | BMS: Origin: per-pack BMS warning flags | Expose per pack as diagnostic sensor 'Pack N BMS warnings' (bitmask, decoded attributes); rename DEV 'reserved_5a'. |
| 34309 | `pack4_bms_warning_flags` | FUNCTIONAL | BMS: Origin: per-pack BMS warning flags | Expose per pack as diagnostic sensor 'Pack N BMS warnings' (bitmask, decoded attributes); rename DEV 'reserved_5a'. |
| 34409 | `pack5_bms_warning_flags` | FUNCTIONAL | BMS: Origin: per-pack BMS warning flags | Expose per pack as diagnostic sensor 'Pack N BMS warnings' (bitmask, decoded attributes); rename DEV 'reserved_5a'. |
| 34509 | `pack6_bms_warning_flags` | FUNCTIONAL | BMS: Origin: per-pack BMS warning flags | Expose per pack as diagnostic sensor 'Pack N BMS warnings' (bitmask, decoded attributes); rename DEV 'reserved_5a'. |
| 34609 | `pack7_bms_warning_flags` | FUNCTIONAL | BMS: Origin: per-pack BMS warning flags | Expose per pack as diagnostic sensor 'Pack N BMS warnings' (bitmask, decoded attributes); rename DEV 'reserved_5a'. |

### Further read registers (37021, 38000–38014)

| Register | Name | Status | Target | Recommendation |
|---|---|---|---|---|
| 37021 | `mppt_field_2c_raw` | UNKNOWN | none: origin MPPT (field meaning MPPT-defined); consumers: shell/MQTT/cloud only | Do NOT expose as energy/total_increasing. DEV only: two raw u16 (37021, 37022) |
| 38000–38014 | `can40_word1 … can43_word4` | UNKNOWN | none: none (no producer, no consumer) | Keep as DEV (disabled by default, NOT in poll/block reads): a read in 38000-39014 makes the Control broadcast four CAN frames per second until reboot. No sender among ... |

### Command registers 45000–45031

| Register | Name | Status | Target | Recommendation |
|---|---|---|---|---|
| 45000 | `cmd_micro_factory_mode` | FUNCTIONAL | Micro: Micro g_workMode (0x20003e04) = 12 or 1 + power-stage test code 1/9; Control g_maintModeState=2; probably MPPT on/off | Do NOT expose. |
| 45001 | `cmd_inverter_eeprom_test` | FUNCTIONAL | Micro: Micro EEPROM 0x80 write/read-back test; status byte 30213 | Do not expose (DEV diagnostic at most, with 30213). |
| 45002 | `cmd_inverter_sleep_request` | CONDITIONAL | Micro: Micro sleep flag -> temporary stop of the run state machine | Do not expose. |
| 45003 | `cmd_factory_beep` | CONDITIONAL | Micro: Micro buzzer (factory mode only) | Do not expose. |
| 45004 | `cmd_factory_openloop_test` | CONDITIONAL | Micro: Micro power-stage open-loop tests | Never expose. |
| 45005 | `cmd_factory_overvoltage_test` | CONDITIONAL | Micro: Micro overvoltage test | Never expose. |
| 45006 | `cmd_read_inverter_input_pb1` | FUNCTIONAL | Micro: State of micro GPIO PB1, readable at 30211 | Do not expose (DEV diagnostic at most with 30211). |
| 45007 | `cmd_factory_offgrid_output_test` | CONDITIONAL | Micro: Micro off-grid output test | Never expose. |
| 45008 | `cmd_factory_gridtie_power_test_w` | CONDITIONAL | Micro: Micro grid-tied power test at P W | Never expose. |
| 45009 | `cmd_manual_backup_output_flag` | CONDITIONAL | Micro: Micro backup/off-grid output enable flag | Do not expose; use 41200. |
| 45010 | `cmd_manual_power_setpoint_w` | CONDITIONAL | Micro: Micro power reference (+ discharge / - charge) | Never expose; use 42000=0x55AA + 42010/42020/42021. |
| 45011 | `cmd_mode_record_0x1006` | UNKNOWN | none: none | Do not expose. |
| 45012 | `cmd_led_lamp_test_ms` | FUNCTIONAL | GPIO: Front-panel LEDs (all on, active-low) for v ms | Do not expose (DEV button 'LED test' at most). |
| 45013–45019 | `reserved_13 … reserved_19` | NO_OP | none: none | Do not expose, never poll. |
| 45020 | `cmd_class2_unit1_on_and_power_w` | UNKNOWN | none: class-2 node-1 unit (apparently MPPT/DC charger, per g_mppt* naming) - absent on Venus D | Do not expose. |
| 45021–45024 | `cmd_class2_unit1_on_off … cmd_class2_unit1_reg_0xFF` | UNKNOWN | none: none on D | Do not expose. |
| 45025 | `dbg_gpio_pb10_pack_encode_line` | CONDITIONAL | GPIO: Control PB10 = battery pack encode/handshake line (inferred PB10 -> first BMS PB13) | Do not expose (DEV at most). |
| 45026 | `dbg_gpio_pe15_button_led` | CONDITIONAL | GPIO: power-button indicator LED (inferred) | Do not expose. |
| 45027 | `cmd_bms_master_reboot` | FUNCTIONAL | BMS: BMS master pack: system reset (+ optional EEPROM 0x202 = 0xFF = self-claim master on next boot) | Never expose. |
| 45028 | `cmd_bms_factory_reset` | FUNCTIONAL | BMS: all BMS packs: factory reset of protection/config parameters + reboot | Never expose. |
| 45029 | `cmd_battery_pack_encode_start` | CONDITIONAL | BMS: BMS pack address encoding / master claim | Do not expose as control (DEV button at most). It is the equivalent of the app's 'Battery Pack Recovery' (pack address coding), not a recovery charge; 32114 shows the ... |
| 45030–45031 | `cmd_bms_syscmd_0x0B_0x0C … cmd_bms_restart_handshake` | FUNCTIONLESS | none: none on D | Do not expose. |

### Miscellaneous write/diagnostic registers (45603–47400)

| Register | Name | Status | Target | Recommendation |
|---|---|---|---|---|
| 45603 | `inverter_calibration_value_a` | FUNCTIONAL | Micro: Micro EEPROM 0x950 (battery calibration result counter delta) | DEV/diagnostic sensor, read on demand or rarely (not in fast poll block), decode BCD |
| 45604 | `inverter_calibration_value_b` | FUNCTIONAL | Micro: Micro EEPROM 0x952 (battery calibration result counter delta) | DEV/diagnostic sensor, read on demand or rarely (not in fast poll block), decode BCD |
| 45605 | `ethernet_chip_version` | FUNCTIONAL | CH395: CH395 Ethernet controller version byte | DEV/diagnostic sensor, read once at startup |
| 46000 | `ota_debug_command` | FUNCTIONAL | EEPROM: Control RS485 task mode (EEPROM 0x900) / Control YModem firmware-update mode | do NOT expose; never read or write |
| 46500–46544 | `can_sched_reg_00_u16 … can_sched_reg_14_u32_lo` | FUNCTIONLESS | none: none (RAM/EEPROM storage only) | do NOT expose |
| 47001 | `can_cat0x18_command` | FUNCTIONLESS | none: none on Venus D | do NOT expose |
| 47400 | `debug_test_register` | NO_OP | none: none | do not expose as entity; usable as internal connection/handshake probe |

## 4. Recommendations for the Marstek Modbus Suite (Venus D)

- **Remove:** button `factory_reset` (41001, does nothing on the D) (the 15 DEV entries 38000–38014 stay as DEV: disabled by default and excluded from polling, because reading them triggers a CAN broadcast and their peer is unknown).
- **Keep:** `reset_device` (41000 = `0x55AA`): a real reboot of Control and Micro.
- **Add as regular registers (read-only):** BMS discharge current limit 32107 (0.1 A, next to 32106); BMS lock active 32112 (binary sensor); per-pack BMS warning flags 34009 / 34109 / … (bitmask); RS485 slave address 37000 (diagnostic).
- **DEV / diagnostic only:** 30028 / 30029 / 30036 (MPPT), 37021 (raw), 30211, 37001–37003, 32113, 45603–45605 (reading 45603/45604 blocks the Modbus task up to 200 ticks).
- **Never expose:** 40000, 41000 = `0xAA11`, 45000–45031, 46000, 46500–46544, 47001, 47400 (dangerous or functionless).
- **Optional:** a write-only `set_wifi` service for 41500–41631 (SSID 1–31 characters, password empty or 8–32, printable ASCII without `,` and `"`; write the SSID first with a 0x00 terminator, then the password up to the terminator register).

## 5. Open questions (need Ghidra, a schematic, or MPPT firmware)

- Physical wiring of the Micro's USART2 Modbus slave (internal BMS bus?)
- Does the device re-provision the FC41D TLS certs / MQTT identity automatically after 0xAA11?
- Old Wi-Fi credentials at EEPROM 0x400/0x420 are not rewritten by the defaults branch; do they reappear on a later boot?
- Does Ctrl_SetPowerReferenceClamped clamp to the CAN-pushed Micro limits (e.g. 800 W discharge default)?
- Tick unit of g_appTickMs (seconds?)
- FC41D SSID pre-processing FUN_0003c2b8/FUN_0003c300 (non-ASCII/UTF-8 SSIDs) not analysed
- Whether QueryConnectionState(2) can report success while the module is still associated to the OLD AP (race between QSTAAPINFODEF and the disconnect URC) - would persist untested creds
- FC41D 32-byte flash item for id 0x11111105: confirm passwords >32 chars are truncated at module boot
- Is the class-2 receiver the MPPT board, i.e. does 45000 stop PV charging?
- Power reference after 0x55BB restart (does stop reset ctx+0x54?)
- Exact duration and whether a running inverter is stopped or only a start blocked
- What PB1 is wired to
- Sign convention of test code 3
- Receiver on other products (E2/A BMS)?

Verification note: two independent analysts re-derived these results from the code; corrections (e.g. 41000 `0xAA11` address, 45027 EEPROM write condition, 45029/45031 behaviour, BMS warning thresholds) are already included in the CSV column `befund_D`.
