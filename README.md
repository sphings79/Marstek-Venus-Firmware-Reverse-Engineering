# Marstek Venus — Firmware Reverse Engineering

> 🇩🇪 Deutsche Version: [README.de.md](README.de.md)

Static reverse-engineering analysis of the Marstek Venus firmware (models D, A and E3: Control/EMS,
micro-inverter/VNS, BMS) using Ghidra, as the basis for a Home Assistant integration via the devices' Modbus TCP interface.
All findings are based on static analysis of the firmware images, not on live debug
access (SWD/JTAG), unless noted otherwise.

**Models:** Venus D/E (`VNSD-0`), Venus A (`VNSA-0`), Venus E3 (`VNSEE3-0`); ARM Cortex-M MCUs, FreeRTOS
**Analyzed firmware versions:**
- Venus D: Control 151 / 150 / 149.2 / 149 / 147, Micro/VNS 116 / 115, BMS 118 / 117.7
- Venus A: Control 151 / 150 / 1509, Micro 1211, BMS 115 / 110 / 1105
- Venus E3: Control 151 / 150, Micro 119, BMS 115

The binary-fingerprint table below covers the **Venus D** images only (status as noted there).

## Binary fingerprint (Venus D firmware images)

Verified live from Ghidra (status 2026-07-15). The "functions named" share is purely the
Ghidra symbol status. BMS 118 was analyzed completely in terms of content (diff, struct/register layouts)
and the findings were written back as Ghidra symbol names on 2026-07-15 (526 via
diff markup transfer from 117.7, 24 new ones named individually) — hence 100 %. Control 147 was completely
re-mapped on the same day: 1530 functions via diff markup transfer from 149.2, 340 only-in-147 functions
named individually via subagent analysis (of these, 23 were marked as Ghidra function-boundary artifacts instead of
being given invented semantics) — details in `Control_FW/VNSD_Control_FW_Analysis_147_202601281721320b2053125.md`.

| Firmware | Version | File | Size | Flash range | Initial SP | Reset handler | Functions (named/total) | Strings |
|---|---|---|---|---|---|---|---|---|
| Control | **150 (current)** | `VNSD-0_app_0150_0805_115146.bin` | 389,120 B (380 KB) | `0x08000000–0x0805EFFF` | `0x2001F7E8` | `0x08004A71` | 1627 / 1884 (86.4 %) ¹ | 1804 ¹ |
| Control | 149.2 | `Control_149.2_VNSD-0_app_1492_0702_142136.bin` | 385,024 B (376 KB) | `0x08000000–0x0805DFFF` | `0x2001F7D8` | `0x08004A71` | 1618 / 1622 (99.8 %) | 1743 |
| Control | 147 | `Control_147_...0b2053125.bin` | 372,736 B (364 KB) | `0x08000000–0x0805AFFF` | `0x2001F3E0` | `0x08004A71` | 1870 / 1870 (100 %) | 1302 |
| Micro/VNS | **116 (current)** | `Micro_VNS_116_vd_inv_app_0116_0702_ota_163439.bin` | 115,712 B (113 KB) | `0x08000000–0x0801C3FF` | `0x20009A70` | `0x080042AD` | 392 / 445 (88.1 %) | 251 |
| Micro/VNS | 115 | `Micro_VNS_115_...0c0e30687.bin` | 115,712 B (113 KB) | `0x08000000–0x0801C3FF` | `0x20009A70` | `0x080042B1` | 13 / 190 (6.8 %, version diff against 116 only) | 102 |
| BMS | **118 (current)** | `BMS_118_20260119100535e43806957.bin` | 106,496 B (104 KB) | `0x08000000–0x08019FFF` | `0x2000CBA0` | `0x08002A6D` | 578 / 649 (89.1 %) ¹ | 263 ¹ |
| BMS | 117.7 | `BMS_117.7_20251010135647565eb2036.bin` | 106,496 B (104 KB) | `0x08000000–0x08019FFF` | `0x2000CB90` | `0x08002A6D` | 552 / 552 (100 %) | 260 |

¹ Status 2026-09-24, counted live from Ghidra. The numbers are higher than before because
more functions and strings have been defined since. The other rows are status 2026-07-15.

**Common properties of these seven Venus D images:** ARM Cortex-M4F, Thumb-2, little-endian, FreeRTOS
(`heap_4`, `ARM_CM4F` port), flash base `0x08000000`.

| Firmware | Compiler | Crypto | Cell monitoring | Communication |
|---|---|---|---|---|
| Control | GCC | mbedTLS 2.28.10 | — | WiFi + Ethernet (CH395) + RS485 + CAN |
| Micro/VNS | RVDS/Keil ARM | — (none) | — | CAN + RS485 |
| BMS | RVDS/Keil ARM | — (none) | KA495XX (BMIC) | CAN + RS485 |

Notable: Micro/VNS 115 and 116 as well as BMS 117.7 and 118 each occupy exactly the same
flash address range despite differing functionality — suggests a fixed OTA slot size
per firmware type rather than a variable image size.

## Folder structure

**File naming:** Documents that apply to only one device carry the device prefix `VNSD_`
(Venus D), `VNSA_` (Venus A) or `VNSE3_` (Venus E3). Only cross-device documents
(methodology, device comparison, diff workflow) have no prefix. Not renamed is the subfolder
`Modbus_RS485_TCP/Scan_Logs/`.

This directory is the root of the repository; all paths in the documents are
repo-relative (e.g. `Modbus_RS485_TCP/…`, `Control_FW/…`, `BMS/…`). What exists only locally
is listed in the section "Not in the repository (local only)" further below.

| Folder | Contents |
|---|---|
| `Control_FW/` | Control/EMS firmware (main processor). Venus D: full function analysis for v149.2 (name tracking table, ~1600 functions, batch history) and v147 (diff-based analysis against 149.2, 340 only-in-147 functions documented individually). For v150 additionally the cloud watchdog including the root-cause analysis of the 30-minute grid outages (§4a of the v150 analysis) and the case study `VNSD_Cloud_Watchdog_Case_Study_2026-08-26.md` — including the four refuted interim hypotheses and the methodology that settled them. Conversion of the BMS charge limit into power, PV curtailment and the 42011/force_mode coupling in `VNSD_Control_Charge_Power_PV_Curtailment_v150.md`; Venus A Control 1509 analysis `VNSA_Control_FW_v1509_Full_Analysis.md`; Venus A Control 151 analysis `VNSA_Control_FW_v151_Full_Analysis.md` with the diff tables `VNSA_Control_Diff_v1509_v151.md` and `VNSA_Control_Diff_v150_v151.md`; Venus A Control 150 analysis `VNSA_Control_FW_v150_Full_Analysis.md` with `VNSA_Control_Diff_v150_v1509.md`; Venus E3 Control 150 analysis `VNSE3_Control_FW_v150_Full_Analysis.md` (incl. verification of the HTTP OTA body-start selector) with the diff table `VNSE3_Control_Diff_v150_v151.md`; cross-device OTA HTTP body-boundary selector, OTA reader and OTA diagnostics reference `OTA_HTTP_Body_Boundary_and_Diagnostics.md`; Wi-Fi disconnect URC bug `VNSD_FW_Bug_WLAN_Disconnected_URC_2026-10-02.md` |
| `VNS_Micro_Inverter/` | Micro-inverter/VNS firmware: Venus D function analysis, register maps, version diff 115→116, error/warning codes. Files with a device prefix (`VNSA_…`) apply **only** to this model — the Venus A inverter is a separate code base (0.4 % shared code with the D), its error bits are documented separately in `VNSA_Error_Codes_Micro_FW_Analysis.md`; Venus A micro 1211 (Novosense NS800RT) analysis: `VNSA_Micro_FW_v1211_Full_Analysis.md` |
| `BMS/` | BMS firmware analysis (Venus D: v117.7, v118); pack rotation, discharge lower limit, special modes and measured charge taper in `VNSD_BMS_Pack_Rotation_Taper_v118.md`; SOH, cycle counter, full/empty detection, cell balancing and discharge current limit in `VNSD_BMS_SOH_Cycles_Discharge_Limit_v118.md`; battery pack recovery, firmware side (automatic recovery charge, remote BMS commands) in `VNSD_Battery_Pack_Recovery_Firmware.md`; Venus A BMS 1105 analysis: `VNSA_BMS_FW_v1105_Full_Analysis.md`; Venus A BMS 110 analysis `VNSA_BMS_FW_v110_Full_Analysis.md` (code base A 109 + 40 A cap) with the diff table `VNSA_BMS_Diff_v1105_v110.md`; Venus A BMS 115 analysis `VNSA_BMS_FW_v115_Full_Analysis.md` (merge of 110 and 110.5 plus SOC-imbalance pack switch, MOS recovery, RTC wake-up from STOP) with the diff table `VNSA_BMS_Diff_v110_v115.md` and the open points in depth (config checksum, unknown CAN ids, PE7/PD8) in `VNSA_BMS_FW_v115_Open_Points.md` |
| `Modbus_RS485_TCP/` | External Modbus interface of the Venus devices (TCP **and** RS485 RTU, parallel stack with a shared descriptor table): connection reference, protocol quirks, scale factors, RS485 unlocking, full register map (CSV), register hypotheses from scan evaluation, decoded descriptor tables (CSV) including version diffs, read/write handler register maps in `Scan_Logs/` (the raw scan logs themselves exist only locally), reconciliation of the BLE power values against Modbus (load test 600 W), firmware bug in the read serializer (German analysis + English vendor report), write register block 46500-46544, register resolution (functional status per register); complete code-verified register reference (all registers with status, origin/target and integration notes) in `VNSD_Register_Reference.md`; Venus A register map and reference (all registers, same-as-D comparison, version notes for Control 151 and BMS 110/115): `VNSA_Register_Reference.md` |
| `CAN_Bus/` | Internal CAN bus between the MCUs: structure of the arbitration ID, target classes (inverter/BMS/RS485 path), complete function code tables of all 46 send sites of the Control against the receiver dispatchers of Micro v116 and BMS v118 (Venus D), coverage gaps |
| `BLE/` | Bluetooth LE interface of the Control: command mapping (Venus D v150) and cross-reference BLE ↔ Modbus registers |
| `Methodik_und_Meta/` | Reverse-engineering methodology, Ghidra workflow/lessons learned, firmware diff workflow, serial number format, open questions. **Save Ghidra changes made via ReVa:** `ReVa_Saving_Changes.md` (saving in the project window is not sufficient) |
| `HM_HIE_FC41D/` | Communication module Quectel FC41D (Beken BK7231N): OTA/update analysis, software stack, version diff 202409 → 202512, SoftAP/update path, Hamedata customizations, instructions for traffic capture (Raspberry Pi, laptop AP, BLE update) |
| `App/` | Analysis of the Marstek smartphone app (`com.hamedata.marstek`): API recon, command vocabulary of app 1.6.76, mapping of the app commands to the firmware, decompilation method |
| `Hardware_Recherche/` | Internet research on components: MCU identification (Control/BMS: Nations N32G45x, Micro: STM32G474), teardown/photo research, approval and certificate documents (Markdown evaluations only, sources as links) |
| `SVD/` | Vendor SVD files (Nations N32G451/452/455/457, Artery AT32F403A, STM32G474) for naming the peripheral registers in Ghidra |

**Caution, terminology mix-up:** "RS485" appears in the firmware in two independent roles —
(1) the **external** RS485 RTU Modbus stack (47,400 baud, FC03/FC06/FC10, parallel access to
the same registers as Modbus TCP, documented in `Modbus_RS485_TCP/`) and (2) **internal**
RS485/UART connections between the MCUs (Control↔micro-inverter for power setpoints/modes, BMS
master↔slave packs for 96-byte struct transfers) — a proprietary struct protocol, **not Modbus**,
documented in `Control_FW/` and `BMS/` respectively.

## Not in the repository (local only)

The following belongs to the project but is deliberately not part of this repository. The documents refer to it
at the relevant places and mark it with "(local, not in the repo)":

- **Decompilation exports** (`Dekompilate/`): complete decompilations and listings per firmware. They can
  be reproduced with the documented Ghidra workflow, see `Methodik_und_Meta/Decompilation_Export.md`.
- **Own helper scripts** (Python): scan/monitoring tools for Modbus, export/save scripts for
  Ghidra, verification and diff helpers.
- **Raw scan logs** (CSV/log files of the Modbus scans of various operating states).
- **Third-party PDFs, screenshots and vendor headers**: datasheets, manuals, test reports,
  teardown screenshots and the vendor header `n32g45x.h` (copyrighted; sources as links
  in `Hardware_Recherche/`).
- **Decrypted vendor firmware** and archives of the communication module's update images.
- **Confidential security material** (`security/`), see next section.

## Important note on security findings

In the course of the analysis, security-relevant findings were made (among others a hardcoded AWS IoT client certificate
including private key shared across product lines, and a firmware update path without
bounds check). These were reported to the vendor confidentially via responsible disclosure and are
**deliberately not published in this repository** until a fix is available. Affected
passages in the documents above are marked accordingly and refer to internal,
non-public documentation.

## Use for the Home Assistant integration

Most relevant are `Modbus_RS485_TCP/` (connection setup, register map, scale factors,
pitfalls such as the 32-register batch limit) as well as the register/struct descriptions in `Control_FW/`
and `VNS_Micro_Inverter/`.
