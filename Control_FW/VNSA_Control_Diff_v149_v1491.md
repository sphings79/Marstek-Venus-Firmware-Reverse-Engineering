# Diff Table Control v149 → v1491 (Venus A)

> 🇩🇪 Deutsche Version: [VNSA_Control_Diff_v149_v1491_de.md](VNSA_Control_Diff_v149_v1491_de.md)

**As of:** 2026-11-05. Image `149_control_VNSA-0_app_0149_0528_102448.bin` ("A Control 149", build string `May 28 2026 10:23:00`) compared with `1491_control_VNSA-0_app_1491_0528_145327.bin` ("A Control 1491", build string `May 28 2026 14:52:08`). Both are Venus A (`VNSA-0`) images, both 382,976 B, both loaded in Ghidra with base 0x08004800 (1913 functions each).
**Related documents:** [VNSA_Control_FW_v1491_Full_Analysis.md](VNSA_Control_FW_v1491_Full_Analysis.md), [VNSA_Control_FW_v149_Full_Analysis.md](VNSA_Control_FW_v149_Full_Analysis.md), [VNSA_Control_Diff_v149_v150.md](VNSA_Control_Diff_v149_v150.md)

## Summary

**Direction:** 149 is the older build, 1491 the newer one, built on the same day (28 May 2026, about 4.5 hours later). "1491 has X" means: X is in 1491 and not in 149.

- **The only functional change is the firmware version constant: 149 (0x95) → 1491 (0x5d3).** It is loaded in five places (five functions) and stored at the version word 0x08010000. No other opcode, string pointer, format string, called function or data item differs. The shell root entry string changes from `VNSA-0 v149` to `VNSA-0 v1491`, the build string from `May 28 2026 10:23:00` to `May 28 2026 14:52:08`.
- The 16-bit `movs rN,#0x95` became a 32-bit `movw rN,#0x5d3` (+2 B per site), so the code behind each site shifts and the literal pools move; in addition the string `v149` became `v1491` (+1 B) and alignment padding changed. The image size is unchanged (382,976 B) because the padding in front of the compressed data blobs absorbs the growth (see "Layout").
- The version constant is reported to the outside (`ems_ver` in the shell, device type code in the cloud report, BLE device status and MQTT telemetry). That 0x5d3 is the firmware version number follows from the numeric match with the file name and from these uses (see below); the code itself carries no further meaning for it.

## Method (how "only the version constant changes" was verified)

1. **Byte level.** Both files have the same size; SHA-256 differs (`f14fede5...` vs `271c6026...`). 99.39 % of the 1491 bytes (380,646 of 382,976) are identical to 149 in 45 anchor blocks after masking addresses and call operands; the rest is the shift region around the five sites.
2. **Instruction level.** The flash listings exported from Ghidra (123,591 vs 123,592 items) were compared item by item after masking all address operands (`adr`, `bl`, `ldr` literals, pointer words). Result: 15 difference blocks, and every one is accounted for:

| # | Block | Meaning |
|---|---|---|
| 1 | 08010000 `uint 0x95` → `uint 0x5d3` | version word |
| 2-6 | `movs rN,#0x95` → `movw rN,#0x5d3` at 149 08011d00 / 080190e0 / 0801ad76 / 0802c8f4 / 0803a50e (1491: 08011d00 / 080190e4 / 0801ad7e / 0802c8fc / 0803a516) | the five version loads (registers r3, r3, r3, r0, r1) |
| 7-10 | five `Alignment align(2)` items appear / disappear next to the literal pools of these functions | padding moves with the 2 B growth (typing, no code) |
| 11 | `Alignment align(3)` at 08050939 | padding after the longer string `VNSA-0 v1491` |
| 12-14 | strings "left", "help" typed as `string` vs `TerminatedCString`; `byte[3]` pad at 0805a991 | Ghidra typing of the same bytes, no content change |

   No other mnemonic or operand differs once addresses are masked.
3. **Decompiler level.** The 1913 exported functions pair 1:1 by name. The five functions with the version load are the only ones with a different instruction count or opcode (same instruction counts 234/234, 225/225, 751/751, 369/369, 35/35 and the same control flow). The remaining 1908 differ only by address operands, because the code after each site moved by +2, +4 or +8 bytes (entry points shift, e.g. `HTTP_GetUrlData` 08019048 → 0801904c, `Cloud_UploadTelemetryData` 0801a8d0 → 0801a8d8, `Config_LoadOrInit_DeviceSettings` 0802c6ec → 0802c6f4, `ShellCmd_get_ver` 0803a4f4 → 0803a4fc; later functions by +8).
4. **Pair check in Ghidra.** Anchors 1908 identical, 5 partial, 0 unmapped (of 1913); call-target check over 9115 calls: 0 findings; RAM layout identical to 149 (all 1429 synchronised labels have the same address and name; the sets of referencing functions of 89 further globals are identical).

## Diff table

| Area | 149 | 1491 | Function / address (1491) | Evidence |
|---|---|---|---|---|
| Version word | `uint 0x95` | `uint 0x5d3` | 08010000 (followed by `VNSA-0` at 08010004) | bytes `95 00 00 00` → `d3 05 00 00`; no code reference in either version |
| Shell `get_ver` | `ems_ver=\t%d` prints 0x95 | prints 0x5d3 | `ShellCmd_get_ver` 0803a4fc, `movw r1,#0x5d3` at 0803a516 | listing diff; decompile `printf("ems_ver=\t%d\r\n",0x5d3)` |
| Build string | `May 28 2026 10:23:00` @ 0803a557 | `May 28 2026 14:52:08` @ 0803a55f | `ShellCmd_get_ver` | byte search |
| Device type code | `g_deviceTypeCode = 0x95` | `= 0x5d3` | `Config_LoadOrInit_DeviceSettings` 0802c6f4, `movw r0,#0x5d3` at 0802c8fc (next to `g_deviceTypeStr = "VNSA-0"`) | listing diff; decompile |
| Cloud report version | 0x95 | 0x5d3 | `HTTP_Cloud_Reporting_Dispatcher` 08011c60 (`movw r3` at 08011d00) | listing diff |
| HTTP URL data version (`sv=`) | 0x95 | 0x5d3 | `HTTP_GetUrlData` 0801904c (`movw r3` at 080190e4) | listing diff |
| Telemetry upload version | 0x95 | 0x5d3 | `Cloud_UploadTelemetryData` 0801a8d8 (`movw r3` at 0801ad7e) | listing diff |
| Shell root entry | `VNSA-0 v149` @ 08050924 / 0805a8f5 | `VNSA-0 v1491` @ 0805092c / 0805a908 | `CLI_InitSession` | byte search; decompile `CLI_FindEntryByName(session,"VNSA-0 v1491",...)` |
| Consumers of the constant (code unchanged) | | | `g_deviceTypeCode` is read by `Cloud_Report_FillDeviceInfo`, `BLE_Build_DeviceStatus_Response`, `BLE_App_Command_Dispatcher` (formatted device info), `MQTT_Telemetry_Struct_Builder` (`g_telemetryDeviceVersion`), `MQTT_StatusReport_BuildAndPublish` | decompiled exports: the device reports 1491 where it reported 149 |

### Layout (not functional)

| Item | 149 | 1491 |
|---|---|---|
| Image size | 382,976 B | 382,976 B |
| Scatter table `Region$$Table$$Base` | 08060840 | 08060850 |
| `ShellCommand[78]` | 08060870 | 08060880 |
| `.data` load image (2480 B, `Load$$RW_IRAM1$$Base`) | 08060D50 | 08060D60 |
| XMC load image (1164 B, `Load$$RW_XMC$$Base`) | 08061700 | 08061710 |
| 0xFF fill before the footer | 1108 B from 08061B8C | 1092 B from 08061B9C |
| Footer `VNSA` | 08061FE0 | 08061FE0 |

The code growth (5 x 2 B for the `movw`, 1 B for `v1491`, alignment) moves everything behind the first site up by 4 to 16 B; the footer stays at 08061FE0 because the 0xFF fill shrinks by 16 B.

## Verification checklist (standing checks)

| Question | Result 149 vs 1491 |
|---|---|
| Modbus registers | **Unchanged.** `Write_Handler`, `FC03_Read_Handler`, `RS485_Modbus_RegisterMap_Init` are identical pairs (identical after masking addresses); `g_modbusRegDescTable` is at the same RAM address with the same type and size (246 x 12 B). No added or removed register. Register contents and status classification were not re-derived (as in 149). |
| BLE | **Unchanged** except the reported version. Name prefix `MST_VNSA_`; `BLE_App_Command_Dispatcher` is an identical pair, so command table and response layouts are unchanged. The device status response (`BLE_Build_DeviceStatus_Response`) now carries 1491 instead of 149 through `g_deviceTypeCode`. |
| AC feed-in limit | **Unchanged.** `Grid_Export_Power_Limiter` identical pair: start clamp 2500 W (0x9c4), upper clamp 2400 W (0x960); `Write_Handler` text for 44003 (1500 W) identical. |
| Update path (BLE OTA) | **Unchanged.** `BLE_Cmd_OTA_Validate`, `BLE_Cmd_OTA_Init`, `BLE_Cmd_OTA_WriteSetup`, `BLE_OTA_WriteDataChunk`, `OTA_Is_VNSA_Model` (tests `VNSA-0`), `OTA_Slot_Config_Validate`, `YModem_ReceiveFile` and `HTTP_FindHeaderBodySeparator` are identical pairs. BLE OTA is possible as far as the code shows (**not tested on a device**). A version-number comparison for update offers (if any) happens outside the image, **not checked**. |
| Cloud endpoints | **Unchanged.** `hamedata.com` only, upload via `setVenusDReporting` (AES-128-ECB + Base64Url), no `marstekcloud.com`; all endpoint strings are the same. Changed: the version value (`sv=`, device info, telemetry upload) now reads 1491. How the cloud treats this value: **not checked**. |
| Pack capacity | **Unchanged.** `BMS_CellVoltage_MinMax_PerPack` identical pair (6 packs `< 6`, 13 cells `< 0xd`), same 0x58 B pack record and 0x35b B history slot. The BMS-side bound (`CAN_TX_PerPack_10Msgs`) is not part of the Control image: **not checked**. |
| Non-functional registers (re-check) | **No change possible from this image**, because the register code and table are unchanged; the 149 state carries over (A register classification itself: not checked). |
| Diff table | Above. |

## Open points

- The word at 08010000 (0x95 in 149, 0x5d3 in 1491, followed by `VNSA-0`) has no code reference in either image and no literal-pool word pointing at it. Its consumer is unknown; a bootloader or the update tooling reading the application header is **inferred, not shown**.
- Whether the changed version value triggers different behaviour on the cloud or app side is not established (it is only reported).
- Not checked: the comparison of 1491 against 150, 151 and the betas (1508, 1509); 1491 vs 149 is the only comparison in this document.
