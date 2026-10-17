# Cloud Upload: Which Variable Feeds Which Key, Compared Across Control Versions

> 🇩🇪 Deutsche Version: [Cloud_Upload_Schluessel_Quellen_Versionsvergleich_de.md](Cloud_Upload_Schluessel_Quellen_Versionsvergleich_de.md)

**Status:** 2026-10-15. Extends `VNSD_Cloud_Upload_Key_Table_v151.md` (what each key means on Venus D control 151) to all decompiled control firmwares.

## 1. Scope and method

Ten control firmwares: VNSD 147, 149, 1492, 150, 151; VNSA 150, 1509, 151; VNSE3 150, 151.
In every version the upload is one `sprintf` in `Cloud_UploadTelemetryData`. For each version the arguments were recovered from the assembler listing (the decompiler shows only the first ones), joined to the offset inside the telemetry history record, and then to the code that fills that field: `MQTT_Telemetry_Struct_Builder` for scalars, `MQTT_Telemetry_String_Formatter` for the comma-separated lists. Result per key: the live variable, its width and its sign. The sources were then compared with Venus D 151, the version the meanings were verified on.

## 2. Result in one sentence

**Almost every key reads the same variable with the same conversion in all ten versions and on all three models.** What a key means therefore carries over; only the set of keys and a few fields differ.

## 3. Keys sent per version

| Version | Keys | Missing compared with VNSD 151 |
|---|---|---|
| VNSD 151, VNSA 1509, VNSA 151 | 76 | none |
| VNSD 1492, VNSD 150, VNSA 150 | 74 | `p_sta`, `p_pow` |
| VNSD 149 | 71 | `p_sta`, `p_pow`, `bt_p`, `ival`, `soh` |
| VNSD 147 | 69 | as 149, plus `bkd`, `ip` |
| VNSE3 151 | 70 | `pv`, `sk`, `mv`, `me`, `ma`, `fu` (no MPPT/pack block) |
| VNSE3 150 | 68 | as VNSE3 151, plus `p_sta`, `p_pow` |

`dn` is a literal in every version and equals the firmware version (147, 149, 1492, 150, 151, 1509).

## 4. Keys whose source differs

| Key | Where | Difference |
|---|---|---|
| `vc` | VNSE3 150, 151 | Not the max/min statistics of six packs. It is a list of **16 values: the voltage of cell 1 to 16 of pack 1** (`g_bmsPack0CellVoltageArray[0..15]`). Unit assumed to be mV like the other cell voltages (not checked on a device). |
| `mq` | VNSD 147 | The live flag `g_quectelWifiMqttReadyFlag`. From control 149 on the builder stores the constant 1, so `mq` is always 1 there and says nothing about the link. |
| `bke` | VNSD 147 | One value only: `g_energyCat3MonthWh` (monthly grid import in the meaning of 151's `bkd`). From 149 on: `bke` = category 4 day/month/total, `bkd` = category 3 day/month/total. |
| `mt` | VNSD 147, 1492, 150 | Source is named `g_meterSourceChangeFlag`; named `g_meterDataFreshFlag` in 149 and 151. Same position in the record and same neighbours, so very likely one variable renamed during analysis, not a change in the code. |

Everything else in the table of `VNSD_Cloud_Upload_Key_Table_v151.md` (all scalar keys, `tc`, `cp`, `ce`, `ms`, `pv`, `sk`, `fu`, `im`, `bkd`, `bt_p`) has the same source in every version that sends the key. Differences in local variable names of the decompiler (`uVar1` vs. `uVar2`) are not differences in the code.

## 5. Limits

- `pw`, `bl`, `as` could not be traced in VNSD 147 (the compiled function is laid out differently there); they are present in the other nine versions with the same source as 151.
- This compares the code, not live data. Units and scales were verified on Venus D 151 only; for the other models they follow from the identical conversion in the code.
- `hd` is still unassigned.
- The binaries of further versions in the firmware archive (VNSA 1508/149/1487/148, VNSE3 149/148/1476/147/144, HMG-50 153–156) have no decompilation yet and are not covered.
