# Cloud Upload Key Table (Control v151, Venus D)

> 🇩🇪 Deutsche Version: [VNSD_Cloud_Upload_Schluesseltabelle_v151_de.md](VNSD_Cloud_Upload_Schluesseltabelle_v151_de.md)

**Status:** 2026-10-15. Derived from the VNSD Control v151 listing and decompilation; checked against live uploads of three Venus D (control 151) and against the Modbus values Home Assistant read for the same devices. Not tested on hardware beyond that comparison.

This completes `VNSD_Cloud_Upload_Payload_v150.md`, whose "Not yet assigned" list is resolved here except for one key (`hd`). The upload body is `d=<query string>`; the firmware 151 sends 76 keys: `di` and `sn` (device id and serial, not listed below) and the 74 keys in the tables below.

Comparison of the key sources across all ten decompiled control versions: [Cloud_Upload_Key_Sources_Version_Comparison.md](Cloud_Upload_Key_Sources_Version_Comparison.md)

## 1. Summary

- The upload is **one `sprintf`** in `Cloud_UploadTelemetryData` (0x0801B460). Every argument is a copy of a named live variable or a field of a telemetry record, so the meaning of a key follows from its source variable. No key had to be guessed from its value.
- Two builders feed the record: `MQTT_Telemetry_Struct_Builder` (0x08016FFC) writes the scalar fields from the live variables, `MQTT_Telemetry_String_Formatter` (0x08034CE0) writes the comma-separated list fields. The record is pushed into a ring of 144 slots by `Telemetry_History_Record_Push` (0x08010B7C), and the upload reads one slot (`upload_slot_index`).
- **Hex fields:** `eb, e1, ea, et, me, ma` and `sk[4..9]` are printed with `%lx` / `%llx`. `10` there means 0x10, not ten.
- **`dn`** is a literal in the call (0x97 = 151), not a variable: it always shows the firmware that built the message.
- **`sv` is not the version** here. In the `getDateInfo` URL `sv` is the Control version; in the upload it is `g_workMode`.
- The decompiler shows only the first arguments of the call. The remaining ones were read from the assembler listing (stack slots `sp+0x000 … sp+0x140`, 79 slots; the 64-bit `e1` takes two) and joined to the struct fields through the global addresses. The Ghidra names of some struct fields are wrong (for example the field behind `t1` is called "GridExportRaw1" but is the internal temperature); the source variable in the builder is the reliable part.

Evidence: **Modbus (HA)** = the value matched the Modbus register or a Home Assistant sensor of the same device exactly (counters, steps) or within the polling lag. **code** = meaning from the source variable only; the unit is given where the code shows it (for example a `_0p1V` suffix or a `/ 10`), otherwise the raw value stands.

## 2. Scalar keys

| Key | Meaning | Unit / scale | Firmware source | Evidence |
|---|---|---|---|---|
| `to` | Charge energy, total | ×0.01 kWh | `g_totalChargeEnergyWh` / 10 | Modbus (HA) |
| `td` | Discharge energy, total | ×0.01 kWh | `g_totalDischargeEnergyWh` / 10 | Modbus (HA) |
| `em` | Charge energy, this month | ×0.01 kWh | `g_monthChargeEnergyWh` / 10 | Modbus (HA) |
| `gm` | Discharge energy, this month | ×0.01 kWh | `g_monthDischargeEnergyWh` / 10 | Modbus (HA) |
| `ed` | Charge energy, today | ×0.01 kWh | `g_dayChargeEnergyWh` / 10 | Modbus (HA) |
| `gd` | Discharge energy, today | ×0.01 kWh | `g_dayDischargeEnergyWh` / 10 | Modbus (HA) |
| `bp` | Stored energy (SoC × pack energy) | ×0.01 kWh | `g_pgn1801InfoBatSoc * g_pgn1802InfoBatEnergy / 10000` | Modbus (HA) |
| `sc` | State of charge | % | `g_pgn1801InfoBatSoc` × 0.1 | Modbus (HA) |
| `pb` | Battery power (negative = discharging) | W | `bv × 0.01 × bi × 0.1`, computed | Modbus (HA) |
| `bv` | Battery voltage | ×0.01 V | `g_pgn1801InfoBatVolt` | Modbus (HA) |
| `bi` | Battery current | ×0.1 A | `g_pgn1801InfoBatCurr` | Modbus (HA) |
| `ci` | BMS charge current limit | ×0.1 A | `g_pgn1803InfoChrgCurr` | Modbus (HA) |
| `ri` | BMS discharge current limit | ×0.1 A | `g_pgn1803InfoDischrgCurr` | Modbus (HA) |
| `mc` | Max charge power | W | `g_maxChargePowerW` | Modbus (HA) |
| `md` | Max discharge power | W | `g_maxDischargePowerW` | Modbus (HA) |
| `go` | AC / grid power | W | `g_gridSamplePowerW` | Modbus (HA) |
| `gp` | Off-grid power | W | `g_offGridSamplePowerW` | code |
| `gv` | Grid voltage | ×0.1 V | `g_gridVoltage_0p1V` | Modbus (HA) |
| `gf` | Grid frequency | ×0.1 Hz | `g_gridFrequency_0p1Hz` | Modbus (HA) |
| `t1` | Temperature, internal (environment) | ×0.1 °C | `g_envTemp_0p1C` | Modbus (HA) |
| `t2` | Temperature, MOS (radiator) | ×0.1 °C | `g_radiatorTemp_0p1C` | Modbus (HA) |
| `t3` | Reserved | — | constant 0 | code |
| `wm` | User work mode byte (10 = RS485 control) | raw | `g_userWorkMode` | Modbus (HA) |
| `sv` | Work mode | raw | `g_workMode` (not the version: the same name in the `getDateInfo` URL is the Control version) | code |
| `gt` | Inverter state | raw | `g_invState` | code |
| `bs` | Power flow state (charge / discharge / idle) | raw | `BatteryParams_PowerFlowState_Get()` | code |
| `bu` | EPS (backup) reserve enabled | 0/1 | `g_epsReserveEnable` | code |
| `ct` | Meter data valid | 0/1 | `g_meterDataValidFlag` | code |
| `mt` | Meter data fresh | 0/1 | `g_meterDataFreshFlag` | code |
| `ty` | Meter source type | raw | `g_meterSourceType` | code |
| `cy` | Meter source type (second copy) | raw | `g_meterSourceType` | code |
| `ds` | Discharge cut-off SoC | raw | `g_dischargeCutoffSocRaw` | code |
| `ph` | Active MPPT channel | raw | `g_mpptActiveChannel` | code |
| `cw` | MPPT operation mode | raw | `g_mpptOpModeFlag` | code |
| `bt` | BLE connection state | raw | `g_bleConnState` | Modbus (HA) |
| `mq` | Cloud/MQTT link active | 0/1 | 1 if the Wi-Fi MQTT is ready or an MQTT session is active, else 0 | code |
| `ws` | Wi-Fi RSSI | raw | `g_wifiRssi` | code |
| `up` | API connection state | raw | `g_apiConnState` | code |
| `ap` | Local API enabled | 0/1 | `g_localApiEnable` | code |
| `nt` | Ethernet link active (CH395) | 0/1 | `g_ethLinkActive` | code |
| `bl` | BLE SDV enable | 0/1 | `g_bleSdvEnable` | code |
| `as` | Economy (TOU) task state | raw | `g_httpEconomyTaskState` | code |
| `dn` | Control firmware (a constant compiled into the format arguments: 0x97 = 151) | — | literal in the call | Modbus (HA) |
| `bm` | BMS firmware | — | `g_bmsRefBmsVer` | Modbus (HA) |
| `iv` | Inverter firmware | — | `g_canRegCbSoftVer` | Modbus (HA) |
| `mv` | MPPT firmware (high:low byte of the CAN data word) | — | `g_canMpptDataWord_verHi/_verLo` | Modbus (HA) |
| `eb` | Pack 1 error words 1 and 2 (hex, `%lx`) | hex | `g_bmsPack0Err2:g_bmsPack0Err1` | code |
| `e1` | Error codes 1 and 2 as 64 bit (hex, `%llx`) | hex | `g_errorCode1`, `g_errorCode2` (the 64-bit argument takes two stack slots) | code |
| `ea` | Error code bitmask (hex) | hex | `g_errCodeBitmask` | code |
| `et` | Warning code bitmask (hex) | hex | `g_warnCodeBitmask` | code |
| `me` | MPPT error (hex) | hex | `g_mpptError` | code |
| `ma` | MPPT warning (hex) | hex | `g_mpptWarning` | code |
| `pw` | Inverter power command | W | `g_canInvPowerCmdW` | code |
| `soh` | Pack 1 state of health | % | `g_bmsPack0Soh` (often 0 in the field) | code |
| `p_sta` | Power clamp state | raw | `g_powerClampConfig` (control 151 only) | code |
| `p_pow` | Power clamp power | W | `g_powerClampConfig_clampPowerW` (control 151 only) | code |
| `ival` | Upload interval | min | literal 5 in the call | code |
| `no` | Upload / record number | — | argument of `Cloud_UploadTelemetryData` | code |
| `dt` | Device clock | — | record date + `g_rtcCacheHour/Minute/Second` | Modbus (HA) |
| `ip` | Own IP address | — | from the CH395 local address or the network info | Modbus (HA) |
| `gy` | Reserved | — | constant 0 | code |

## 3. List keys

| Key | Items | Layout | Unit | Firmware source | Evidence |
|---|---|---|---|---|---|
| `vc` | 24 | 6 packs × 4 values: highest-cell index, lowest-cell index, highest cell voltage, lowest cell voltage | index; mV | `g_bmsCellStat{0..5}Max/MinCellIdx`, `…Max/MinCellVoltage` | Modbus (HA) + code |
| `tc` | 4 | NTC temperatures 1..4 of pack 1 | ×0.1 °C | `g_bmsPack0NtcTemp1..4` | Modbus (HA) |
| `cp` | 4 | CT (meter) power phase 1, 2, 3, total | W | `g_ctPowerCh1..3`, `g_ctPowerNow` | code |
| `ce` | 4 | MPPT income values A1, A2, B1, B2 | raw | `g_mpptIncomeValueA1/A2/B1/B2` | code |
| `ms` | 5 | MQTT connected, VID session state, 0, 0, XID valid | raw | `g_mqttConnectedFlag`, `g_mqttVidSessionState`, `g_mqttXidValidFlag` | code |
| `pv` | 18 | 4 PV inputs × (voltage, current, power, present) then PV energy today and this month; power is forced to 0 below 101, present means voltage > 0x8c | voltage/current raw; power W; energy 10 Wh | `g_mpptPv1..4Vol/Cur/Pow`, `g_mpptPvDayCap10Wh`, `g_mpptPvMonCap10Wh` | code |
| `sk` | 16 | working battery index, force-charge flag, pack count, online mask, pack 1..6 error word (hex), pack 1..6 BMS version | raw; hex; version | `g_pgn1802InfoWorkBatIdx`, `g_pgn1803InfoForceChrg`, `g_pgn1802InfoBatTotalNb`, `g_pgn1802InfoBatOnlineMask`, `g_bmsPack0..5Err1/2`, `g_bmsPack0..5BmsVer` | Modbus (HA) + code |
| `fu` | 2 | surplus feed-in enable, surplus feed-in enabled | 0/1 | `g_surplusFeedInEnable`, `g_surplusFeedInEnabled` | code |
| `im` | 11 | MQTT session active, CH395 MQTT init status, local API UDP enabled, CH395 UDP last receive length, CH395 chip version, BMS lock flag, inverter power command (W), inverter setpoint send count, MPPT voltage (×0.1 V), MPPT power (W), MPPT send count | mixed | `g_mqttSessionActive`, `g_ch395MqttInitStatus`, `g_localApiUdpEnable`, `g_ch395UdpLastRecvLen`, `CH395CMDGetVer()`, `g_pgn1804InfoLockFlag`, `g_canInvPowerCmdW`, `g_invSetpointSendCount`, `g_mpptVoltage_0p1V`, `g_mpptPowerW`, `g_mpptSendCount` | code |
| `bke` | 3 | grid export (energy category 4: sold to the grid): today, month, total | Wh | `g_energyCat4DayWh/MonthWh/TotalWh`, fed from `g_gridSellEnergyWh` | code |
| `bkd` | 3 | grid import (energy category 3: bought from the grid): today, month, total | Wh | `g_energyCat3DayWh/MonthWh/TotalWh`, fed from `g_gridBuyEnergyWh` | code |
| `bt_p` | 2 | pack 1 charge power, pack 1 discharge power | W | `g_bmsPack0ChargePower`, `g_bmsPack0DischargePower` | code |
| `hd` | 3 | **unknown** (three values; the firmware writes the buffer elsewhere, not found) | — | — | open |

In `vc` the first two values of each pack are the indices of the cells that currently have the highest and lowest voltage. Positions 2 and 3 are the voltages in mV (confirmed for packs 1..3 against the Modbus per-pack cell voltages). A pack that is not present reports zeros.

## 4. Differences between firmware versions

Format strings of the decompiled control firmware compared key by key:

| Firmware | Keys | Difference to VNSD 151 |
|---|---|---|
| VNSD 150, VNSA 150 | 74 | without `p_sta`, `p_pow` |
| VNSA 150.9, VNSA 151 | 76 | none |
| VNSE3 150 | 68 | without `pv, sk, mv, me, ma, fu, p_sta, p_pow` |
| VNSE3 151 | 70 | without `pv, sk, mv, me, ma, fu` |
| VNSD 147, 149, 149.2 | — | not compared (the function is not under this name in the exports) |

Key order is the same wherever a key exists. The meanings above were traced on **VNSD 151** only; VNSA and VNSE3 use the same function with the same names, but their individual sources were not traced.

## 5. Open

- `hd`: three values (`0,16,0` in a field sample); the buffer behind it follows `g_mqttTeleNetInverterMpptStr` and its writer was not found.
- Units for the code-only keys without a unit hint (PV voltage and current, `ce`, `ds`, `ws`, meter fields).
- Whether the VNSD 147/149 formats differ.

## 6. Use

The endpoint project decodes these keys (`decode.py`, and a measurements database with editable names and a per-firmware meaning) and marks every key as checked against Modbus or from the code. Firmware changes show up there because `dn`, `bm`, `iv` and `mv` are recorded with a timestamp.
