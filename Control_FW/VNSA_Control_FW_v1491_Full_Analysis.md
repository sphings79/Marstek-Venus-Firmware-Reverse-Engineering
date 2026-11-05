# Control FW v1491 (Venus A) — Full Analysis (port from 149)

> 🇩🇪 Deutsche Version: [VNSA_Control_FW_v1491_Vollanalyse_de.md](VNSA_Control_FW_v1491_Vollanalyse_de.md)

**Status:** completed 2026-11-05. The image was analysed by porting the finished v149 analysis and verifying the port against the 1491 code. 1491 is **functionally identical to 149**; only the firmware version constant differs.
**Binary:** `1491_control_VNSA-0_app_1491_0528_145327.bin` (archive `Marstek FW Archiv/firmwares/VNSA-0/Control/1491/`, "A Control 1491")
**Port source:** Venus A Control v149, see [VNSA_Control_FW_v149_Full_Analysis.md](VNSA_Control_FW_v149_Full_Analysis.md). Everything 1491 shares with 149 (method of the port from 150, RAM layout, device identity, differences to 150, shared topics) is documented there and is **not repeated** here.
**Differences 149 → 1491:** [VNSA_Control_Diff_v149_v1491.md](VNSA_Control_Diff_v149_v1491.md)
**Ghidra:** `/VNSA/Control/Control_1491_VNSA-0_app_1491_0528_145327.bin` (project `Marstek_Venus`)
**Decompilation:** private working export (not part of this repository): 1913 functions, 0 decompilation failures, plus a flash listing and `globals_and_registers.txt` (2589 named RAM/peripheral symbols), same layout as for 149.

---

## 0. Summary

- **Same code as 149.** After masking addresses and call operands, 99.39 % of the bytes (380,646 of 382,976) are identical; the instruction-level comparison of the flash listings finds only the version constant and its side effects (diff table). Every statement in the 149 analysis about code, Modbus, BLE, OTA, cloud, packs and the comparison with 150 therefore applies to 1491 unchanged.
- **1491 is Venus A:** `VNSA-0` / `Venus A` / BLE prefix `MST_VNSA_`, build string `May 28 2026 14:52:08` (149: `May 28 2026 10:23:00`, the same day).
- **What differs:** the firmware version constant 0x95 (149) → 0x5d3 (1491) at five `movw` sites (`HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `Config_LoadOrInit_DeviceSettings`, `ShellCmd_get_ver`), the version word at 0x08010000, the shell root entry `VNSA-0 v149` → `VNSA-0 v1491`, the build string; layout shifts by 4 to 16 B behind the first site.
- **Verification:** all gates pass (see §3); tags identical to 149 (1786 `NAME_VERIFIED`, 127 `NAME_LOW_CONFIDENCE`).

## 1. Binary fingerprint

| | A 1491 | A 149 (reference) |
|---|---|---|
| Size | 382,976 B (0x5D800) | 382,976 B |
| SHA-256 | `271c60265478cf92eba21824814632c185d9344244112a842b547fd1e65b6a16` | `f14fede53a07462ecbba7c37c0c1b24ac910a5962206a54f2ab21c62034ea07e` |
| Link address | **0x08004800**–0x08061FFF | same |
| MCU / toolchain | Nations **N32G45x** (Cortex-M4F), Keil ARMCC | same |
| Build string (`ShellCmd_get_ver`) | `May 28 2026 14:52:08` @ 0x0803A55F | `May 28 2026 10:23:00` @ 0x0803A557 |
| Version word | 0x5d3 (1491) @ 0x08010000, followed by `VNSA-0` | 0x95 (149) |
| Shell root entry (`CLI_InitSession`) | `VNSA-0 v1491` @ 0x0805092C, 0x0805A908 | `VNSA-0 v149` |
| Image footer | `VNSA` @ 0x08061FE0, 0xFF fill (1092 B) from 0x08061B9C | 1108 B from 0x08061B8C |
| Functions in Ghidra | 1913 | 1913 |

Base proof as in 149: initial stack pointer and reset vector resolve at base 0x08004800. Memory map, scatter table (here `Region$$Table$$Base` @ 0x08060850, `.data` load image @ 0x08060D60, XMC load image @ 0x08061710, `ShellCommand[78]` @ 0x08060880; sizes unchanged) and RAM layout are those of [149, §2 and §3.3](VNSA_Control_FW_v149_Full_Analysis.md); the RAM layout is identical to 149 (only flash addresses shift).

## 2. What differs from 149

Details with addresses and evidence: [VNSA_Control_Diff_v149_v1491.md](VNSA_Control_Diff_v149_v1491.md).

- Five functions load the constant: `movs rN,#0x95` became `movw rN,#0x5d3` (+2 B each); the literal pools behind them move by +4 / +8 B and entry points shift (e.g. `Config_LoadOrInit_DeviceSettings` 0802c6ec → 0802c6f4).
- The constant goes out as `ems_ver` (shell `get_ver`), as device type code `g_deviceTypeCode` (cloud report, BLE device status, MQTT telemetry) and as the version in the HTTP URL data (`sv=`) and the telemetry upload.
- No other opcode, string, format string or called function differs; the remaining 1908 function pairs differ only by address operands.

## 3. Processing status in Ghidra

- Import 1638 functions (auto-analysis), port from 149 via anchors: 1908 identical pairs, 5 partial, 0 unmapped; layout (1913 functions), segments, signatures, flash references, flow overrides (28 applied, 2 reset), 64 TBB/TBH jump tables typed (24 decompiler overrides, case labels versus tables: 0 mismatches), RAM labels (1429 from the name sync, 89 more taken over after an equal-reference-set check, 94 `<object>_off<hex>`), 1336 RAM/XMC data types, 977 peripheral labels, 26 plates.
- Gates (all 0): undefined runs, orphan runs, flow into data, function pointers to non-code, literal-only code, `FUN_*` names, duplicate names/labels, placeholders, shell-command plates, default prototypes, task-table mismatches, noreturn; unnamed referenced RAM globals 0. Call-target check over 9115 calls: 0 findings.
- Plates that mention version or build time were corrected for 1491 (`ShellCmd_get_ver`, `Config_LoadOrInit_DeviceSettings`, `HTTP_Cloud_Reporting_Dispatcher`, `HTTP_GetUrlData`, `Cloud_UploadTelemetryData`, `CLI_InitSession`).
- Export: 1913 decompiled functions with 0 failures, flash listing, `globals_and_registers.txt` (2589 symbols). Name quality is as in 149: `<object>_off<hex>` and layout-derived names are address-relation names (low confidence).

## 4. Standing checks

All unchanged against 149 (code identical): Modbus registers, BLE, AC feed-in limit (2500 W start clamp / 2400 W upper clamp, 1500 W text for 44003), BLE-OTA path, cloud endpoints (`hamedata.com` only), pack capacity (Control side 6 packs / 13 cells). Per-item evidence and the limits of the check: [VNSA_Control_Diff_v149_v1491.md](VNSA_Control_Diff_v149_v1491.md), "Verification checklist". The reported version value is the only visible change.

## 5. Open points and limits

- **Word at 0x08010000** (0x95 in 149, 0x5d3 in 1491, before `VNSA-0`): no code reference and no literal-pool word pointing at it in either image. Consumer unknown; probably the bootloader or the update tooling reading the application header (**inferred, not shown**). Left as an unnamed `uint`.
- That 0x5d3 is the firmware version number is inferred from the numeric match with the file name and from its uses (`ems_ver=%d`, `g_deviceTypeCode`, `sv=`); no symbol or documentation names it.
- How the cloud or the app reacts to version 1491 instead of 149: not checked.
- Everything the 149 analysis lists as unchecked (register table entry by entry, BMS-side pack bound, cloud TLS, status of non-functional registers) is unchecked for 1491 as well.
- Not part of this document: comparison of 1491 with 150, 151 and the betas.
