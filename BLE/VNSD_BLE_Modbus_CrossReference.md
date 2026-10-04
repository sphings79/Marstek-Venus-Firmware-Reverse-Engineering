# BLE ↔ Modbus Cross-Reference

> 🇩🇪 Deutsche Version: [VNSD_BLE_Modbus_CrossReference_de.md](VNSD_BLE_Modbus_CrossReference_de.md)

Systematic reconciliation between the BLE protocol (marstek-venus-monitor) and the Modbus TCP registers.

## Methodology

1. Decompiled the BLE RuntimeInfo builder (FUN_0800af78) in the Control FW
2. Resolved SRAM source addresses via the literal pool (DAT_0800b380 ff.)
3. Identified shared SRAM structs between the BLE and Modbus handlers
4. Semantic correlation via field names, data types and scan values

### Key structs (SRAM v148)

| Literal Pool | SRAM address | Usage |
|---|---|---|
| DAT_0800b380 | 0x20014E90 | Inverter status struct (workMode, gridPower, batteryPower) |
| DAT_0800b384 | 0x20014F40 | Pack status array (SOC, cell voltages per pack) |
| DAT_0800b388 | 0x2001884A | BLE RuntimeInfo output buffer (0xA4 bytes) |
| DAT_0800b394 | 0x20014F82 | BMS data struct |
| DAT_0800b3a0 | 0x20014CDC | Version info struct |
| DAT_0800b3a4 | 0x20014D8E | Energy counter struct (daily/monthly/total charge/discharge) |
| DAT_0800b3a8 | 0x20014CFC | Configuration struct (73+ fields, also written via BLE cmd 0x28) |
| DAT_0800b3b4 | 0x20000136 | EU power limit flag |
| DAT_0800b3b8 | 0x2000012B | Power rating struct |

## Confirmed mappings

### RuntimeInfo (BLE cmd 0x03) → Modbus read register

| BLE Offset | BLE Field | BLE Type | Modbus Reg | Modbus Name | FW source |
|---|---|---|---|---|---|
| 0x00 | gridPower | i16 | 30006 | ac_power | DAT_0800b380+0x1A |
| 0x02 | batteryPower | i16 | 30001 | battery_power | DAT_0800b380+0x18 |
| 0x04 | workMode | u8 | 35100 | inverter_state | *DAT_0800b380 |
| 0x0C | deviceFwVersion | u16 | 30200 | ems_version | DAT_0800b3a0+0x16 |
| 0x0E | dailyCharge | u32/100 | 33004 | daily_charging_energy | DAT_0800b3a4+0x14 |
| 0x12 | monthlyCharge | u32/1000 | 33008 | monthly_charging_energy | DAT_0800b3a4+0x0C |
| 0x16 | dailyDischarge | u32/100 | 33006 | daily_discharging_energy | DAT_0800b3a4+0x18 |
| 0x1A | monthlyDischarge | u32/100 | 33010 | monthly_discharging_energy | DAT_0800b3a4+0x10 |
| 0x29 | totalCharge | u32/100 | 33000 | total_charging_energy | DAT_0800b3a4+0x04 |
| 0x2D | totalDischarge | u32/100 | 33002 | total_discharging_energy | DAT_0800b3a4+0x08 |
| 0x3D | wifiRssi | u8 | 30303 | wifi_signal_strength | DAT_0800b38c[1] |
| 0x4F | bmsVersion | u16 | 30204 | bms_version | DAT_0800b394[0x11] |

### DeveloperModeInfo (BLE cmd 0x0D) → Modbus read register

| BLE Offset | BLE Field | Modbus Reg | Modbus Name |
|---|---|---|---|
| 0x01 | lineFrequency | 32204 | ac_frequency |
| 0x03 | acVoltage | 32200 | ac_voltage |
| 0x07 | temperature1 | 35000 | internal_temperature |
| 0x09 | temperature2 | 35001 | mos1_temperature |
| 0x0B | temperature3 | 35002 | mos2_temperature |
| 0x0D | temperature4 | 35010 | max_cell_temperature |
| 0x0F | temperature5 | 35011 | min_cell_temperature (NEW) |

### BMSData (BLE cmd 0x14) → pack registers (34000+)

| BLE Offset | BLE Field | Modbus Pack1 Reg | Scale |
|---|---|---|---|
| 0x00 | bmsVersion | 34010 | ×1 |
| 0x08 | remainingCapacity (SOC) | 34002 | ÷10 % |
| 0x0A | stateOfHealth | — | not in the Modbus scan |
| 0x0C | designCapacity | 32105 | ÷1000 kWh |
| 0x0E | voltage | 34000 | ÷100 V |
| 0x10 | current | 34001 | ÷10 A |
| 0x18 | cycleCount (b_cpc) | 34003 | ×1 |
| 0x26 | mosfetTemperature | 34015 | ÷10 °C |
| 0x28-0x2E | temperature1-4 | 34011-34014 | ÷10 °C |
| 0x30-0x51 | cellVoltages[0-16] | 34018-34033 | ÷1000 V |

## BLE fields without a known Modbus register

These fields exist in the BLE protocol but have no known Modbus equivalent. They may be located in unscanned register ranges or be available only via BLE.

| BLE Field | BLE Cmd | Type | SRAM source | EEPROM | Description |
|---|---|---|---|---|---|
| euPowerLimit | 0x03 | u8 | 0x20000136 | — | EU 800W limit (0=off, 1=active) |
| powerRating | 0x03 | u16 | 0x2000012F | — | Rated power in W |
| detectedCtType | 0x03 | u8 | Config+0x68 | — | CT meter type (0=none, 3=HME-4, 4=Shelly Pro EM, ...) |
| batteryPhasePos | 0x03 | u8 | Config+0x69 | — | Battery phase position (1=A, 2=B, 3=C) |
| parallelMode | 0x03 | u8 | Config+0x67 | — | Parallel operation (0/1) |
| localApiEnabled | 0x03 | u8 | Config+0x71 | 0x371 | Local API active (0/1) — controls Modbus TCP! |
| apiPort | 0x03 | u16 | Config+0x72 | 0x372 | API port (default: 502) |
| batteryPackCount | 0x03 | u8 | Pack+0x0C (filtered) | — | Number of detected battery packs |
| installedPackMask | 0x03 | u8 | Pack+0x08 (bool) | — | Bitmask of installed packs |
| workingPackIndex | 0x03 | u8 | v156+ | — | Currently active pack index |
| userWorkMode | 0x03 | u8 | Config+0x01 | — | Mode selected by the user |
| autoWorkModeChange | 0x03 | u8 | Config+0x66 | — | Automatic mode change (0/1) |
| httpServerType | 0x03 | u8 | DAT_0800b3b0+0x60 | 0x441 | Cloud server subdomain (0-4) |
| stateOfHealth | 0x14 | u16 | — | — | SOH in % (BLE only, not Modbus?) |

## WorkMode enum (from the BLE tool, FW-verified)

| Value | Mode | Description |
|---|---|---|
| 0 | Auto | Self-consumption optimization |
| 1 | Standby | No charging/discharging |
| 2 | Charging | Charging only |
| 3 | Sell Electricity | Grid feed-in |
| 4 | UPS/EPS | Backup (EPS) power mode |
| 5 | Force Charge | Forced charging |
| 6 | Grid Export | Grid export |
| 7 | Schedule/TOU | Schedule/time-of-use |

## Key finding: BLE cmd 0x28 controls Modbus TCP

The BLE command 0x28 enables/disables the local Modbus TCP API and sets the port:
- Byte 4: Enable (0/1) → SRAM Config+0x71, EEPROM 0x371
- Bytes 5-6: Port (u16 LE) → SRAM Config+0x72, EEPROM 0x372

This confirms that the Modbus TCP interface can be controlled via BLE.

## Register candidates for unknown scan values

| Modbus Reg | Scan value | Possible BLE field | Rationale |
|---|---|---|---|
| 30102 | 3198 | max_cell_voltage_aggregate | 3.198V typical for LiFePO4 |
| 30103 | 3194 | min_cell_voltage_aggregate | 3.194V |
| 30210 | 3 | detectedCtType? | Value 3 = HME-4 (CT002) |
| 30212 | 2 | batteryPhasePos? | Value 2 = phase B |
| 37000 | 1 | batteryPackCount? | 1 pack installed |
| 35110 | 576 | euPowerLimit related? | Unclear |
| 35111 | 500 | powerRating/10? | 500×5=2500W? Speculation |
