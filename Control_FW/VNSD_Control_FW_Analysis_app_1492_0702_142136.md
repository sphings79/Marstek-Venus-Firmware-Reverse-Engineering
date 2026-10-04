# Marstek Venus D — Modbus Reverse Engineering Documentation

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_Analyse_app_1492_0702_142136_de.md](VNSD_Control_FW_Analyse_app_1492_0702_142136_de.md)
## Full Analysis Documentation — Venus D (VNSD-0) v3 (July 2026)

> **Superseded for code statements by `VNSD_Control_FW_v1492_Full_Analysis.md` (2026-09-30).** The static analysis
> in this document is based on the wrong image base 0x08000000: **all flash addresses here +0x4800**
> (real address = cited address + 0x4800); function names derived from flash strings are partly wrong.
> The live scan results of the Modbus registers (measurements on the device) remain valid.

**Device:** Marstek Venus D (`VNSD-0`), STM32 ARM Cortex-M, FreeRTOS, 6 battery packs  
**Firmware (current):** `VNSD-0_app_1492_0702_142136.bin` — Version **149.2** (July 2026)  
**Goal:** Complete Modbus register map for the Home Assistant integration `marstek_venus_modbus`  
**Status:** Register map of the device completely live-scanned (89 read + 41 write, address space 0–65535). Firmware 149.2 statically analyzed — Modbus handler, descriptor structure and entry format reconstructed (**246 descriptor entries**). Cloud API & OTA infrastructure analyzed (9 MCU components). Descriptor table is built at runtime → register numbers can only be enumerated by live scan (proven from the firmware side, see section 0.3). **Inverter telemetry bridge fully decoded** — all 20 fields of the 48-byte block with plain-text names, units and SRAM addresses of both MCUs (see section 0.6). **Dual-MCU architecture**, DC bus topology, pack rotation and PV behavior documented (see section 0.7). **Cloud telemetry field names** of all 9 report types decoded from FW strings (see section 11.8). WiFi/BLE module identified as **Quectel FC41D**. **RS485 RTU Modbus interface** discovered as a complete parallel stack (CRC16, slave address check, FC03/FC06/FC10+broadcast, see section 2.1). **Write handler statically decompiled** — ~50 write registers extracted from code, many of them **write-only** (invisible in the FC03 batch scan, see section 7.6).

> **Focus:** This documentation covers exclusively the **Venus D (VNSD-0)**.

---

## Table of Contents

1. [Hardware & Firmware Architecture](#1-hardware--firmware-architecture)
2. [Modbus Protocol Details](#2-modbus-protocol-details)
3. [Descriptor Table (Internal Structure)](#3-descriptor-table-internal-structure)
4. [Known Flash Addresses (Ghidra)](#4-known-flash-addresses-ghidra)
5. [Known SRAM Addresses](#5-known-sram-addresses)
6. [Confirmed Read Registers (30000–39999)](#6-bestätigte-read-register-30000-39999)
7. [Confirmed Write Registers (40000–49999)](#7-bestätigte-write-register-40000-49999)
8. [Critical Peculiarities](#8-critical-peculiarities)
9. [MQTT Field Names → Register Mapping](#9-mqtt-field-names--register-mapping)
10. [Marstek Local API — Parallel Protocol & Cross-Validation](#10-marstek-local-api--parallel-protocol--cross-validation)
11. [Marstek Cloud API & OTA Infrastructure](#11-marstek-cloud-api--ota-infrastructure)
12. [Open Items & Next Steps](#12-offene-punkte--nachste-schritte)
13. [Ghidra Detail Analyses: Interesting Findings](#13-ghidra-detailed-analyses-interesting-findings)

> **Externalized documentation:**
> - [VNSD_Analysis_Scripts.md](../Methodik_und_Meta/VNSD_Analysis_Scripts.md) — Scan, FW analysis and Ghidra scripts
> - [Reverse_Engineering_Methodology.md](../Methodik_und_Meta/Reverse_Engineering_Methodology.md) — Obtaining the FW, Ghidra setup, methodology
> - [VNSD_Ghidra_Analysis_Findings.md](../Methodik_und_Meta/VNSD_Ghidra_Analysis_Findings.md) — Wrongly/correctly identified functions, false leads

---

> **Note:** This documentation has been anonymized for publication.
> Device-specific values (UID, MAC, serial number, IP) have been replaced by placeholders.
> The technical content (register map, protocol details, firmware analysis) is complete.

## Binary Fingerprint

Verified live from Ghidra (as of 2026-07-15). A comparison with the other five analyzed
firmware images (Control 147, VNS 116/115, BMS 118/117.7) can be found in the project `README.md`.

| Property | Value |
|---|---|
| File | `VNSD-0_app_1492_0702_142136.bin` |
| Version | 149.2 (current) |
| Size | 385,024 B (0x5E000, 376 KB) |
| Architecture | ARM Cortex-M4F, Thumb-2, Little-Endian |
| Flash range | `0x08000000–0x0805DFFF` |
| Initial SP | `0x2001F7D8` (~128 KB SRAM) |
| Reset Handler | `0x08004A71` |
| Functions | 1618 / 1622 named (99.8 %) |
| Strings | 1743 |
| Compiler | GCC |
| RTOS | FreeRTOS (`heap_4`, `ARM_CM4F` port) |
| Crypto | mbedTLS 2.28.10 |
| Communication | WiFi + Ethernet (CH395) + RS485 + CAN |

---

## 0. Firmware 149.2 — Current Status (July 2026)

> This section summarizes the static analysis of the **current** firmware
> `VNSD-0_app_1492_0702_142136.bin` (149.2). Sections 1–12 and the
> appendices below contain the **complete accumulated RE analysis** of the device
> (protocol, register map, Local API, cloud/OTA, local update procedure).
> Scripts, methodology and Ghidra findings are externalized into separate documents
> (see table of contents). Where firmware-specific addresses/numbers are given, this section 0
> is authoritative for 149.2.

### 0.1 Firmware Key Data

```
File:       VNSD-0_app_1492_0702_142136.bin
Size:       385,024 bytes
Flash:      0x08000000 – 0x0805DFFF
Reset-Vec:  0x08004A70   (IVT @0x08000004 = 0x08004A71 → proves link base 0x08000000)
Initial-SP: 0x2001F7D8   (IVT @0x08000000)
Ghidra:     ARM:LE:32:Cortex, 1,615 functions (as of July 2026)
```

> **Mandatory Ghidra import step:** By default, the image is loaded at `0x00000000` — then
> no absolute pointers resolve. **Before the analysis: set the image base to `0x08000000`
> (Memory Map → Set Image Base) and run the auto-analysis again.**

### 0.2 Modbus Handler & Descriptor Table (149.2)

| Function (149.2) | Role |
|---|---|
| `FUN_0801e43c` | Dispatcher (FC selection; FC `0x03` → read handler) |
| `FUN_0801eaa4` | FC03 read handler (descriptor iteration) |
| `FUN_0804fe20` | Read serializer (type/scale conversion) |
| `FUN_08050f20` | Write handler (register ≥ 40000) |

```
Descriptor base:   0x20000354 (SRAM)   Entries: 246 (0xF6)   Stride: 12 bytes
Table pointer:     DAT_0801EC30 (=0x20000354); 2nd reference @0x08029F50 (RS485 router)
Read/write split:  Register 40000        Max. count: 0x7D = 125 (transport: 32-register batch)
Dual interface:    TCP (:502) + RS485 RTU — both use the same serializer/write handler
```

**Entry format (12 bytes, verified from serializer `FUN_0804fe20`):**
`+0` u16 register (direct) · `+4` u32 live pointer · `+8` type · `+9` low nibble = element size ·
`+10` scale · `+11` count. Register range per entry: `reg_start + (byte9 & 0xF) × byte11 / 2`.

**Type codes:** `01`=u8, `02`=u16, `04`=u32 · high nibble `1`=signed (`11`=i8, `12`=i16, `14`=i32) ·
`24`=float (IEEE-754) · `31`=ASCII (memcpy direct, no scale).

**Scale codes (integer types):** `0`=×1 · `1`=×10 · `2`=×100 · `3`=÷10 · `4`=÷100 · `5`=negate.
**Scale codes (float types):** `0`=×1 · `1`=×10 · `2`=×100 · `3`=×0.1 · `4`=×0.01
(float constants @ `0x0805009C`/`A0`/`A4`; conversion via `VectorFloatToUnsigned`).

**FC03 read flow (149.2):**
1. `FUN_0801eaa4` (FC03_Read_Handler) checks `reg < 40000` → descriptor table lookup
2. Iterates 246 entries (0xF6, stride 0xC) starting from `LiteralPool_DescriptorBasePtr` (0x20000354)
3. Matching entry → `FUN_0804fe20` (Read_Serializer) converts SRAM data → Modbus response
4. `reg >= 40000` → delegation to the write handler with `param3=0` (read mode)
5. Special case: access to 38000–39014 sets a flag at SRAM 0x20000EE5

> **See also:** `Modbus_RS485_TCP/Scan_Logs/Read_Handler_Register_Map.csv` — complete read register map (194 registers,
> consolidated from scan data, FW verification and pack pattern extrapolation).

> **New in 149.2:** A read access in the range **38000–39014** (`0x9867`) sets a
> state flag at SRAM `0x20000EE5`.

### 0.3 Proof: The Descriptor Table Is Built at Runtime

The 246 register numbers are **not statically enumerable** from flash (exhaustive search):
(1) no ascending register array in flash (only false positive `0x08036C10`),
(2) no dense register `MOVW` immediates (leader: write handler `FUN_08050f20`, only 27 constants),
(3) only **2** code references to the base `0x20000354` — both readers, **no** builder,
(4) no `.data` template (pointer signature @+2/+4/+6/+8 in CCM+SRAM negative; `_sidata+0x354` negative),
(5) 0xF6 (246) as a literal only in the 2 readers, not in the builder,
(6) 36 `add rx, ry, ry, lsl #1` candidates (stride-3×4 pattern) analyzed — no matches,
(7) init table at `0x08055158` (24 B entries) references only SRAM 0x20000000–0x20000028,
(8) cloud telemetry format strings are located at `0x0805EC04` — **outside the loaded binary**
    (flash ends at 0x0805DFFF), i.e. in a separate flash partition.

⇒ The builder receives the table base **as a parameter** (no literal pool access) and the
register numbers/SRAM pointers come from a **separate flash data partition** beyond the
application image. Enumeration only by **live scan** or **emulation**.

### 0.4 Crypto, Cloud & Telemetry (149.2)

- **mbedTLS 2.28.10** (SDK paths `..\SDK\mbedtls_2.28.10\library\...`) — for AES key extraction
  of the `setVenusDReporting` payload, trace the `mbedtls_aes_setkey_enc` call sites.
- Reporting endpoint: `http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s`.
- Payload schema (AES plaintext) 70+ fields, including `…&bke=%s&bkd=%s&ip=%s&bt_p=%s&ival=%d&soh=%d`.
- 16 cells confirmed (`b_vo1…b_vo16`); BMS via J1939 PGN (`pgn_1801/1802`); SOC scale 0.1
  (`bat_soc(0.1)=%d`); 4 PV strings.

### 0.5 OTA Status Update & Micro Component

> **Update relative to the earlier status:** The VNSD-0 is now **listed in the OTA system**
> (the earlier empty response is obsolete). Available: **`control` v149** and **`micro` v116**.

| Component | Role | File | Status |
|---|---|---|---|
| `control` | EMS / main MCU | `VNSD-0_app_1492_0702_142136.bin` (149.2) | ✅ analyzed |
| `micro` | Inverter co-processor (`vd_inv_app`) | `vd_inv_app_0116_0702_ota_163439.bin` (116) | ✅ analyzed (own documentation) |

The `micro` component is a **separate** inverter controller (ARM Cortex-M4F, RVDS/Keil,
FreeRTOS, ~40 KB SRAM). Full analysis in a separate document:
`VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analysis_vd_inv_app_0116.md`.

### 0.6 Inverter Telemetry — CAN/RS485 Bridge to the Modbus Registers

> **New in v2.** The complete data flow from the inverter sensors to the Modbus registers
> was reconstructed from both firmwares.

**Architecture:**
```
Micro-MCU ADC sensors
    ↓ (FP calibration, RMS calculation)
Micro-MCU SRAM (0x200018xx float → int16 × scale)
    ↓ (build_telemetry_block @ FUN_08001560)
48-byte telemetry block (Micro SRAM 0x200038E8)
    ↓ (CAN CMD 0x10 / RS485)
Control-MCU SRAM: 0x20014E90  (inverter data struct)
    ↓ (Modbus descriptor table, pointer → 0x20014E90+offset)
Modbus TCP port 502 (FC03 read)
```

**Debug dump:** `FUN_08035ffc` in the Control FW prints all 20 telemetry fields with plain-text names.
Three further blocks (version @ `0x2000015C`, status @ `0x20000144`) are exposed as well.

#### Full Telemetry Block Mapping (48 Bytes)

| Offset | Bytes | Control-FW field name | Unit | Type | Micro-FW source SRAM | Chinese |
|--------|-------|---------------------|---------|-----|---------------------|------------|
| 0x00 | 1 | **inv_state** | — | byte | `0x2000046D` (ctl_status) | — |
| 0x01 | 1 | **buz_state** | — | byte | (always 0) | — |
| 0x02 | 1 | **chrg_flag** | — | byte | `0x200002B3` (grid_connected) | — |
| 0x03 | 1 | **back_func** | — | byte | `0x20003E03` (backup_mode) | — |
| 0x04 | 4 | **warn_code** | — | u32 | `0x200019FC` (war1) | — |
| 0x08 | 4 | **error_code1** | — | u32 | `0x200019F4` (err1) | — |
| 0x0C | 4 | **error_code2** | — | u32 | `0x200019F8` (err2) | — |
| 0x10 | 2 | **grid_volt** | 0.1 V | u16 | `0x200018E4` (grid_voltage_rms) | 电网电压有效值 |
| 0x12 | 2 | **grid_pf** | 0.1 Hz | u16 | `0x200002F0` (grid_frequency) | 捕获频率 |
| 0x14 | 2 | **off_grid_volt** | 0.1 V | u16 | `0x200018BC` (offgrid_voltage_rms) | 离网电压有效值 |
| 0x16 | 2 | **grid_permit** | — | u16 | `0x200002AB` | — |
| 0x18 | 2 | **grid_sample_power** | 1 W | i16 | `0x200018F0` (actual_power) | 实际功率 |
| 0x1A | 2 | **off_grid_power** | 1 W | i16 | `0x200018C8` (output_power) | 发出功率 |
| 0x1C | 2 | **bat_sample_power** | 1 W | i16 | `0x20000E00` | — |
| 0x1E | 2 | **bat_sample_volt** | 0.1 V | i16 | `0x20000E10` (bat_vol_avg) | — |
| 0x20 | 2 | **env_temp** | 0.1 °C | i16 | `0x20000E30` (ntc_inv) | — |
| 0x22 | 2 | **radiator_temp** | 0.1 °C | i16 | `0x20000E34` (ntc_mppt) | — |
| 0x24 | 2 | **max_power** | W | i16 | `0x2000030C` | — |
| 0x26 | 2 | **min_power** | W | i16 | `0x20000310` | — |
| 0x28 | 4 | **chrg_energy** | 1 Wh | u32 | `0x20003E83` (daily_charge) | 日充电 |
| 0x2C | 4 | **dischrg_energy** | 1 Wh | u32 | `0x20003E87` (daily_discharge) | 日放电 |

#### Control-FW SRAM Mapping (Pointer → Modbus)

Base: `0x20014E90` (34 references in the Control FW). All Modbus descriptor entries
for inverter data point directly into this struct as their live-value pointer:

| Control SRAM | = Base + | Telemetry field | Presumed Modbus register |
|---|---|---|---|
| `0x20014EA0` | +0x10 | grid_volt (0.1V) | **32200** |
| `0x20014EA2` | +0x12 | grid_pf (0.1Hz) | **32204** |
| `0x20014EA4` | +0x14 | off_grid_volt (0.1V) | **32300** |
| `0x20014EA8` | +0x18 | grid_sample_power (W) | **32100** (?) |
| `0x20014EAA` | +0x1A | off_grid_volt_mirror (0.1V) | **32301** — **Correction:** voltage duplicate of 32300, not power, see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md |
| `0x20014EAC` | +0x1C | bat_sample_power (W) | **32102** (?) |
| `0x20014EAE` | +0x1E | bat_sample_volt (0.1V) | in scan as 34000 range? |
| `0x20014EB0` | +0x20 | env_temp (0.1°C) | **35000–35002** |
| `0x20014EB2` | +0x22 | radiator_temp (0.1°C) | **35003–35005** |
| `0x20014EB8` | +0x28 | chrg_energy (Wh) | **33000** (u32, 2 registers) |
| `0x20014EBC` | +0x2C | dischrg_energy (Wh) | **33004** (u32, 2 registers) |

> **Note:** The exact Modbus register numbers are determined by the runtime descriptor table.
> The "presumed" numbers come from the live scan (`scan_komplett.csv`, local, raw data) and the
> marstek2mqtt reference project. A definitive confirmation requires either a targeted
> scan of the individual registers or emulation of the descriptor builder routine.

#### Status Block (8 Bytes, `0x20000144`)

| Offset | Type | Name | Description |
|---|---|---|---|
| 0x00 | byte | work_mode | 0=Normal, 1=Test, 4=Update, 0xC=Factory |
| 0x01 | byte | sleep_flag | Sleep mode flag |
| 0x02 | byte | bat_mode | Battery mode |
| 0x04 | u16 | max_charge_power | Max. charge power (≤2500 W) |
| 0x06 | u16 | max_discharge_power | Max. discharge power (≤2500 W) |

#### Version Block (4 Bytes, `0x2000015C`)

| Offset | Type | Name |
|---|---|---|
| 0x00 | byte | hard_ver |
| 0x01 | byte | soft_ver (= 116 for vd_inv v116) |
| 0x02 | byte | boot_ver |
| 0x03 | byte | dev_state |

### 0.7 System Architecture & Operating Behavior (New in v3)

> **Cross-FW findings** from the analysis of all three firmwares (Control, Micro, BMS).
> Details in the respective FW-specific documentation.

#### Dual-MCU Architecture

```
┌─────────────────────────────┐    RS485/UART     ┌─────────────────────────────┐
│  CONTROL-MCU (EMS)          │◄═════════════════►│  MICRO-MCU (Inverter)       │
│  Nations N32G45x, 144KB SRAM│                   │  STM32F4, ~40KB SRAM        │
│  GCC, FreeRTOS              │                   │  RVDS/Keil, FreeRTOS        │
│                             │                   │                             │
│  WiFi (FC41D) + Ethernet    │                   │  LLC converter (DC-DC)      │
│  Modbus TCP :502            │                   │  H-bridge (DC-AC)           │
│  MQTT Cloud                 │                   │  MPPT (PV controller)       │
│  BLE                        │                   │  CAN → BMS packs            │
│  mbedTLS                    │                   │                             │
└─────────────────────────────┘                   └──────────────┬──────────────┘
                                                                 │ CAN bus
                                                  ┌──────────────▼──────────────┐
                                                  │  BMS master (pack 1)        │
                                                  │  STM32F3, ~52KB SRAM        │
                                                  │  RVDS/Keil, FreeRTOS        │
                                                  │  KA495XX cell monitoring    │
                                                  │  RS485 → slave packs 2-7   │
                                                  └─────────────────────────────┘
```

**Communication chain:**
- Control → Micro: RS485/UART (power setpoint, modes)
- Micro → BMS: CAN bus (`0x40xx/41xx` internal + `0x180xAA01` BMS protocol)
- BMS master → slaves: RS485 (96-byte struct transfer, CMD 0x29)
- Control → Cloud: MQTT over WiFi/Ethernet (TLS, mbedTLS)
- Control → HA: Modbus TCP port 502

#### DC Bus Topology (One Inverter, One Power Path)

```
PV panels ──→ MPPT ──→ DC bus (~400V) ←── LLC converter ←── Battery packs
                            │                                 (parallel)
                       H-bridge (DC-AC)
                            │
                       AC grid (household grid)
```

**Critical limitation:** The Venus D has only **one bidirectional power path**.
There is only one `set_inverter_power(float)` call (bootloader function `0x10000160`):
- **Negative** → charging (PV/grid → battery)
- **Positive** → discharging (battery → grid, PV also flows into the grid)
- **Zero** → inverter off (PV is throttled completely)

This is NOT a true hybrid inverter (like Fronius/SMA with separate PV→grid and
PV→battery paths). Consequence: PV cannot charge AND feed into the grid at the same time.

#### Observed PV Throttling Behavior

As the SOC rises, the BMS current limit matrix (`0x0801B884`, 2D: temperature × SOC)
progressively reduces the permitted charge current. Since only one power path exists, PV is throttled:

```
SOC  50% → charge_limit high    → Inverter: -2500W → PV fully used
SOC  90% → charge_limit drops   → Inverter: -1000W → PV partially throttled
SOC  99% → charge_limit minimal → Inverter: -100W  → PV almost completely throttled
SOC 100% → charge_limit = 0     → Inverter: 0W     → PV completely throttled
→ Direction change → discharging → PV flows into the grid again
```

#### Pack Rotation (Round-Robin)

The Micro MCU orchestrates pack selection via CAN commands to the BMS packs:
- **CAN CMD 6** → activates a pack (address-based, MOSFET on)
- **CAN CMD 3** → handoff to the next pack (data = next pack address)

Timers (from Micro FW mode 3):
- Main cycle: **3600s (60 min)** — SOC-based direction decision
- Sub-timer: **600s (10 min)** — CAN update to BMS

The Control MCU only specifies the total power setpoint (CAN CMD 0x01);
pack selection in normal mode (mode 0/1/2) is presumably also controlled in the
Control FW — this scheduler has not been identified yet.

#### Separate FW Documentation

| Firmware | Document | Key contents |
|---|---|---|
| Control v149.2 | (this document) | Modbus handler, descriptor table, telemetry reception |
| Micro/Inverter v116 | `VNS_Micro_Inverter/VNSD_Micro_Inverter_FW_Analysis_vd_inv_app_0116.md` | CAN protocol, telemetry builder, charge/discharge control, DC bus |
| BMS v117.7 | `BMS/VNSD_BMS_FW_Analysis_v117.7.md` | SOC algorithm, cell struct, protect bitmasks, RS485, round-robin |

---

## 1. Hardware & Firmware Architecture

### MCU & Operating System

```
MCU:        Nations N32G45x with XFMC (ARM Cortex-M4F, Thumb-2, Little-Endian)
            -- still listed generically as "STM32" in this document version; for the binding
               identification see section "MCU Identification" below
OS:         FreeRTOS
Flash:      0x08000000 – 0x0805DFFF  (385,024 bytes)
SRAM:       0x20000000 – 0x2001FFFF  (144 KB)
Ext-RAM:    0x60000000 – 0x6001FFFF  (CH395Q or similar, for network buffers)
```

### MCU Identification: Nations N32G45x, not Artery AT32F403A

- The Control/EMS MCU is a **Nations N32G45x with XFMC** (External Memory Controller).
- Evidence: QSPI controller at 0xA0001000 with N32 QSPI_STS bits; RCC PLL multiplier in bits 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG used; SAC crypto coprocessor at 0x40024000 (previously wrongly called "RNGX"); 96-bit UID @ 0x1FFFF7F0; 102 vectors.
- External SRAM runs via XFMC, control registers at 0xA0000000: the official Nations header `n32g45x.h` (local, not in the repo) defines `XFMC_REG_BASE 0xA0000000` and `QSPI_BASE = XFMC_REG_BASE + 0x1000`; the Nations driver library contains `n32g45x_xfmc.c`; the firmware itself has the shell command `xfmc_test`.
- Details/header copy: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

### Firmware binary properties

```
File:       VNSD-0_app_1492_0702_142136.bin
Size:       385.024 bytes
IVT:        0x08000000  (interrupt vector table, ARM standard)
Reset-Vec:  0x08004A70  (from IVT+4: 0x08004A71)
Initial-SP: 0x2001F7D8  (from IVT+0)
```

### Subsystems

| Component | Description |
|---|---|
| CH395Q | Ethernet controller (SPI → Modbus TCP port 502) |
| FC41D | WiFi 802.11b/g/n + BLE 5.0 (UART AT commands) |
| RS485 | 47.400 baud, Modbus RTU → inverter/EMS |
| mbedTLS | Version **2.28.10** — AES-128/256, RSA, ECDH secp256r1 (cloud TLS + telemetry encryption) |
| MQTT | Device communicates with `eu.hamedata.com` |

### Communication channels (complete)

The device exposes **five protocol stacks** (Modbus TCP and RS485 RTU share
the backend handler). For the HA integration, **only Modbus TCP** is used:

| Channel | Port / medium | Protocol | Status for HA |
|---|---|---|---|
| **Modbus TCP** | :502 (Ethernet/WiFi) | Modbus FC03/FC06/FC16 | ✅ Use actively |
| **RS485 RTU** | UART 47400 baud | Modbus RTU FC03/FC06/FC10 | ℹ️ Parallel stack (see 2.1) |
| **Local API** | UDP :30000 (WiFi) | JSON-RPC | ⛔ Do NOT enable (see section 10) |
| **BLE** | Bluetooth LE | HM protocol (0x23 frames) | ℹ️ Monitoring tool rweijnen only |
| **Cloud/MQTT** | TCP :8883 TLS | MQTT → `eu.hamedata.com` | ❌ No local access |

### FreeRTOS tasks (from string analysis)

```
Task_Modbus_tcp    - Modbus TCP server (port 502)
ch395              - Ethernet controller
fc41d              - WiFi/BLE module (Quectel FC41D, confirmed via OTA string "FC41D_OTA.rbl")
ch_mqtts           - MQTT client (cloud)
rs485              - RS485/Modbus RTU
ems                - Energy Management System
udp                - UDP listener → Local API (port 30000, JSON-RPC)
```

> **Note on the `udp` task:** Previously recorded as "discovery on port 8899".
> After analysis of the official Local API documentation (REV 0.5), the default port is 30000.
> This task implements the Local API JSON-RPC daemon.
> **Enabling it causes Modbus errors — see section 10.**

---

## 2. Modbus protocol details

### Connection

```
Protocol:     Modbus TCP
Port:         502
Slave ID:     1
Max. request: 32 registers per read request (Modbus standard)
```

### Register addressing

**Important:** Marstek uses **direct addressing** — the Modbus PDU address corresponds exactly to the visible register number. There is no offset in either direction.

```
User register 30000  →  PDU address 30000  (0x7530)   ← no -1
User register 34002  →  PDU address 34002  (0x84D2)   ← no -1
Read/write boundary: 40000 (0x9C40)
```

> **Proof from TCPRouter.c (decompilation):**
> `uVar6 = CONCAT11(param_1[0], param_1[1])` → raw PDU address, unchanged.
> `if (uVar6 < descriptor.base_addr)` → direct comparison, no offset.
> `Serializer.c`: `offset = PDU_addr - base_addr` → for reg 30000 and `base_addr=30000` the offset is 0. ✅
>
> **Difference from standard Modbus:** In the classic Modbus standard,
> "register 1 = PDU address 0" (user number minus 1). Marstek ignores this
> convention completely. pymodbus call: `read_holding_registers(30000)` → correct.

### Register ranges

| Range | Type | Description | Status |
|---|---|---|---|
| 0 – 29.999 | – | Empty | ✅ Scanned — **no registers present** |
| 30.000 – 39.999 | Read | Sensor/status data | ✅ 89 registers confirmed |
| 40.000 – 49.999 | Write | Control registers | ✅ 41 registers confirmed |
| 50.000 – 65.535 | – | Empty | ✅ Scanned — **no registers present** |

> **Conclusion:** The complete Modbus address space (0–65535) was scanned.
> All registers of the device are located exclusively in the range 30000–49999.
> The ranges 0–29999 and 50000–65535 return no responses.

### Routing logic (from decompilation, 149.2)

```c
// Dispatcher FUN_0801e43c → FC03 read handler FUN_0801eaa4:
if (uVar6 >= 40000)  →  Write-Handler  (FUN_08050f20)
if (uVar6 < 40000)   →  Read-Serializer (FUN_0804fe20)

// Table iteration (FUN_0801eaa4):
for (uVar4 = 0; uVar4 < 246; uVar4++) {
    // Compares requested address with table entries
    (&DAT_20000354)[uVar4 * 6]  // ushort* access = 12 bytes/entry
}
```

### 2.1 RS485 RTU Modbus interface (parallel stack)

> **New in v3.1.** Besides the TCP stack there is a complete **RS485 RTU Modbus stack**
> that uses the same descriptor table and the same write handler.

**Discovery:** The RS485 router was found via cross-references to the second descriptor table reference
(`DAT_08029F50` at `0x08029F50`). This stack
is completely independent of the TCP server and runs on the FreeRTOS task `rs485`.

**Architecture:**

```
UART receive (FUN_080541f8)
    ↓
RS485_RTU_Frame_Dispatcher (0x0801e6b0)
    ├─ CRC16 check (Modbus_CRC16_Calculate @ 0x0801e678)
    ├─ Match slave address (inline, no separate call)
    └─ FC routing:
         FC03 → RS485_RTU_Modbus_Router (0x08029df8)
                 ├─ Reg < 40000 → Read_Serializer (0x0804fe20)  ← identical to TCP
                 └─ Reg ≥ 40000 → Write_Handler (0x08050f20)    ← identical to TCP
         FC06 → RS485_FC06_WriteSingle_Handler (0x08029f54)
         FC10 → RS485_FC10_WriteMultiple_Handler (0x0802a028)
         Broadcast (Addr 0):
              FC06 → RS485_Broadcast_FC06_WriteSingle (0x08029fe0)
              FC10 → RS485_Broadcast_FC10_WriteMultiple (0x0802a108)
    ↓ Response
RS485_UART_Send (0x080072a0)
```

**Special features:**

- **Shared backend:** RS485 and TCP use the same read serializer (`FUN_0804fe20`)
  and write handler (`FUN_08050f20`). All registers reachable via TCP are
  also reachable via RS485 and vice versa.
- **Broadcast support:** Slave address 0 is recognized as broadcast — FC06/FC10
  are executed, but no response is sent.
- **CRC16:** Standard Modbus CRC16 in `Modbus_CRC16_Calculate` (`0x0801e678`).
- **Special flags:** In the dispatcher:
  - Flag `0x01` at `DAT_0801e87c` → `System_Reboot` (`0x08029c78`)
  - Flag `0x02` → `Factory_Reset` (`0x0800b724`)
  - `FUN_08050524(5)` is called on certain fault states
- **Special range 38000–39014:** A read access via RS485 in this range
  likewise sets the flag at `DAT_08029f4c` (analogous to the TCP flag at `0x20000EE5`).

**TCP dispatcher (comparison):**

The TCP dispatcher (`FUN_0801e43c`) likewise has dedicated FC06/FC10 handlers:

| Address | Function |
|---|---|
| `0x0801e880` | `TCP_FC06_WriteSingle_Handler` |
| `0x0801e904` | `TCP_FC06_WriteSingle_Handler_B` |
| `0x0801e94c` | `TCP_FC10_WriteMultiple_Handler` |
| `0x0801ea0c` | `TCP_FC10_WriteMultiple_Handler_B` |

---

## 3. Descriptor table (internal structure)

The table is **built at runtime in SRAM** and does not exist as a static flash structure.

### SRAM location

```
Address:     0x20000354 (SRAM)
Entries:     246 (0xF6)
Size:        246 × 12 = 2.952 bytes (up to 0x20000EDC)
```

### Entry format (12 bytes)

```c
struct RegisterDescriptor {     // 12 bytes per entry
    uint16_t  base_addr;        // [0:2]  Direct PDU address (= user register number, no offset)
    uint16_t  padding;          // [2:4]  unknown/padding
    uint32_t* data_ptr;         // [4:8]  SRAM pointer to raw data
    uint8_t   data_type;        // [8]    Data type
    uint8_t   reg_size;         // [9]    Size in registers (& 0x0F)
    uint8_t   scale;            // [10]   Scaling code
    uint8_t   count;            // [11]   Number of elements
};
```

### Data type codes

| Code | Type | Size |
|---|---|---|
| 0x01 | uint8 | 1 byte |
| 0x02 | uint16 | 2 bytes |
| 0x04 | uint32 | 4 bytes |
| 0x11 | int8 | 1 byte |
| 0x12 | int16 | 2 bytes |
| 0x14 | int32 | 4 bytes |
| 0x24 | float32 | 4 bytes |
| 0x31 | ASCII | variable |

### Scaling codes

| Code | Meaning | Example |
|---|---|---|
| 0x00 | × 1 | Raw value directly |
| 0x01 | × 10 | Raw value × 10 |
| 0x02 | × 100 | Raw value × 100 |
| 0x03 | ÷ 10 (= × 0.1) | Raw value 3116 → 311.6 |
| 0x04 | ÷ 100 (= × 0.01) | Raw value 5012 → 50.12 V |
| 0x05 | negate | Raw value × −1 |

### Serializer logic (from Serializer.c)

```c
// Float type ($=0x24):
local_3c = *(float *)(data_ptr + element_index * 4);
// Apply scaling...
// Other types:
case 0x02: local_38 = *(ushort*)(data_ptr + offset);
case 0x12: local_38 = *(short*)(data_ptr + offset);
case 0x14: local_38 = *(uint*)(data_ptr + offset);
// Scale switch:
case 0x03: local_38 = local_38 / 10;
case 0x04: local_38 = local_38 / 100;
case 0x05: local_38 = -local_38;
```

---

## 4. Known flash addresses (Ghidra)

| Flash address | Function / label | Description |
|---|---|---|
| `0x08000000` | IVT | Interrupt vector table |
| `0x08004A70` | Reset-Vec | Reset handler (149.2) |
| **Modbus TCP** | | |
| `0x0801e43c` | Modbus_Dispatcher | **TCP dispatcher** (FC routing, 149.2) |
| `0x0801eaa4` | FC03_Read_Handler | **FC03 read handler** (descriptor iteration) |
| `0x0804fe20` | Read_Serializer | **Read serializer** (type/scale conversion) |
| `0x08050f20` | Write_Handler | **Write handler** (registers ≥ 40000, 3958 bytes, 942 lines) |
| `0x0801e880` | TCP_FC06_WriteSingle_Handler | TCP FC06 Write Single |
| `0x0801e904` | TCP_FC06_WriteSingle_Handler_B | TCP FC06 Write Single (variant B) |
| `0x0801e94c` | TCP_FC10_WriteMultiple_Handler | TCP FC10 Write Multiple |
| `0x0801ea0c` | TCP_FC10_WriteMultiple_Handler_B | TCP FC10 Write Multiple (variant B) |
| **RS485 RTU** | | |
| `0x0801e6b0` | RS485_RTU_Frame_Dispatcher | UART→CRC→slave addr→FC routing |
| `0x0801e678` | Modbus_CRC16_Calculate | CRC16 calculation/check |
| `0x08029df8` | RS485_RTU_Modbus_Router | RS485 read/write dispatcher (246 entries) |
| `0x08029f54` | RS485_FC06_WriteSingle_Handler | RS485 FC06 Write Single |
| `0x0802a028` | RS485_FC10_WriteMultiple_Handler | RS485 FC10 Write Multiple |
| `0x08029fe0` | RS485_Broadcast_FC06_WriteSingle | RS485 broadcast FC06 (Addr 0) |
| `0x0802a108` | RS485_Broadcast_FC10_WriteMultiple | RS485 broadcast FC10 (Addr 0) |
| `0x080072a0` | RS485_UART_Send | Send RS485 UART response |
| **System** | | |
| `0x08029c78` | System_Reboot | NVIC reset via SCB_AIRCR (50ms delay) |
| `0x0800b724` | Factory_Reset | Factory reset |
| **Descriptor / scale** | | |
| `0x0801EC30` | Literal-Pool | Table base pointer → `0x20000354` |
| `0x08029F50` | Literal-Pool | 2nd descriptor pointer (RS485 router) |
| `0x0805009C` | Scale-Konst. | IEEE-754: 100.0 (scale code 2) |
| `0x080500A0` | Scale-Konst. | IEEE-754: 0.1 (scale code 3) |
| `0x080500A4` | Scale-Konst. | IEEE-754: 0.01 (scale code 4) |

### Important misidentifications

> ⚠️ `FUN_080128a0` (contains MOVW #0x7530) is **not** a descriptor init!
> It is a JSON config parser. The value 0x7530 = 30.000 there is a **default API port**, not a register address.

> ⚠️ `FUN_0801bcb0` is the **RS485/RTU packet handler**, not the descriptor initialization.

---

## 5. Known SRAM addresses

Decompiled from WriteHandler.c:

| SRAM address | Variable | Meaning |
|---|---|---|
| `0x20000133` | `DAT_20000133` | Modbus slave address |
| `0x20000156` | `_DAT_20000156` | Max discharge power (short) |
| `0x20000158` | `DAT_20000158` | Max charge power (short) |
| `0x2000027c` | `DAT_2000027c` | RS485 control active (0/1/2) |
| `0x200002ec` | `DAT_200002ec` | Reset command status |
| `0x200002ed` | `DAT_200002ed` | Operating mode (work_mode) |
| `0x200002ee` | `DAT_200002ee` | Charge target SOC |
| `0x200002f0` | `DAT_200002f0` | Discharge power setpoint (short) |
| `0x200002f2` | `DAT_200002f2` | Charge power setpoint (short) |
| `0x20000354` | `DAT_20000354` | **Descriptor table base** |
| `0x20014b6c` | `DAT_20014b6c` | Backup/UPS function active |
| `0x20014b6d` | `DAT_20014b6d` | Operating mode internal |
| `0x20014b6f..` | Schedule data | Schedule enable bytes |
| `0x20014c25..` | Schedule times | Schedule start/end/power |

---

## 6. Confirmed read registers (30000–39999)

**Scan date:** May 2026 | **Device:** Marstek Venus D, 6 battery packs  
**Condition:** PV not connected (MPPT values = idle ~9.9V)

### 6.1 Power & AC grid

| Register | Name | Type | Scale | Unit | Scan value | Interpreted |
|---|---|---|---|---|---|---|
| 30001 | battery_power | int16 | 1 | W | 65525 | −11 W (discharging) |
| 30006 | ac_power | int16 | 1 | W | 0 | 0 W |
| 32200 | ac_voltage | uint16 | 0.1 | V | 2398 | 239.8 V |
| 32204 | ac_frequency | int16 | 0.1 | Hz | 499 | 49.9 Hz |
| 32300 | ac_offgrid_voltage | uint16 | 0.1 | V | 3 | 0.3 V (off-grid off) |
| 32301 | ac_offgrid_voltage_2 | uint16 | 0.1 | V | 3 | 0.3 V — **Correction:** is voltage, not current (duplicate/mirror of 32300, identical in all 28 scans, see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md) |
| 32302 | ac_offgrid_power | int32 | 1 | W | 0 | 0 W (2 registers) |
| 37004 | grid_power_setpoint | int16 | 1 | W | 65038 | −498 W — **Correction:** not a current register; nearly equal in value to 30006 (watt magnitude), reacts about 1 scan cycle faster on setpoint change (presumably an internal target/control value), see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md |

### 6.2 MPPT / PV inputs

| Register | Name | Type | Scale | Unit | Note |
|---|---|---|---|---|---|
| 30020 | mppt1_voltage | uint16 | 0.1 | V | Idle = 9.9V with PV not connected |
| 30021 | mppt2_voltage | uint16 | 0.1 | V | ditto |
| 30022 | mppt3_voltage | uint16 | 0.1 | V | ditto |
| 30023 | mppt4_voltage | uint16 | 0.1 | V | ditto |
| 30024 | mppt1_current | uint16 | 0.1 | A | |
| 30025 | mppt2_current | uint16 | 0.1 | A | |
| 30026 | mppt3_current | uint16 | 0.1 | A | |
| 30027 | mppt4_current | uint16 | 0.1 | A | |
| 30037 | mppt1_power | uint16 | 0.1 | W | |
| 30038 | mppt2_power | uint16 | 0.1 | W | |
| 30039 | mppt3_power | uint16 | 0.1 | W | |
| 30040 | mppt4_power | uint16 | 0.1 | W | |

### 6.3 Battery pack 1

| Register | Name | Type | Scale | Unit | Scan value | Interpreted |
|---|---|---|---|---|---|---|
| 30100 | battery_voltage | uint16 | 0.01 | V | 5012 | 50.12 V |
| 30101 | battery_current | int16 | 0.1 | A | 0 | 0.0 A |
| 32105 | battery_total_energy | uint16 | 0.001 | kWh | 5120 | 5.120 kWh |
| 34002 | battery_soc | uint16 | **0.1** | % | 109 | **10.9 %** ⚠️ |
| 34003 | battery_cycle_count | uint16 | 1 | – | 14 | 14 cycles |
| 34010 | battery_1_bms_version | uint16 | 1 | – | 116 | **v116** (BMS version per pack) |
| 34018 | battery_1_cell_1_voltage | int16 | 0.001 | V | 3116 | 3.116 V |
| 34019 | battery_1_cell_2_voltage | int16 | 0.001 | V | 3114 | 3.114 V |
| 34020 | battery_1_cell_3_voltage | int16 | 0.001 | V | 3113 | 3.113 V |
| 34021 | battery_1_cell_4_voltage | int16 | 0.001 | V | 3114 | 3.114 V |
| 34022 | battery_1_cell_5_voltage | int16 | 0.001 | V | 3113 | 3.113 V |
| 34023 | battery_1_cell_6_voltage | int16 | 0.001 | V | 3114 | 3.114 V |
| 34024 | battery_1_cell_7_voltage | int16 | 0.001 | V | 3114 | 3.114 V |
| 34025 | battery_1_cell_8_voltage | int16 | 0.001 | V | 3119 | 3.119 V |
| 34026 | battery_1_cell_9_voltage | int16 | 0.001 | V | 3112 | 3.112 V |
| 34027 | battery_1_cell_10_voltage | int16 | 0.001 | V | 3116 | 3.116 V |
| 34028 | battery_1_cell_11_voltage | int16 | 0.001 | V | 3112 | 3.112 V |
| 34029 | battery_1_cell_12_voltage | int16 | 0.001 | V | 3117 | 3.117 V |
| 34030 | battery_1_cell_13_voltage | int16 | 0.001 | V | 3112 | 3.112 V |
| 34031 | battery_1_cell_14_voltage | int16 | 0.001 | V | 3112 | 3.112 V |
| 34032 | battery_1_cell_15_voltage | int16 | 0.001 | V | 3114 | 3.114 V |
| 34033 | battery_1_cell_16_voltage | int16 | 0.001 | V | 3113 | 3.113 V |

### 6.4 Battery pack 2 (inferred, +100 block)

| Register | Name | Type | Scale | Unit | Scan value | Interpreted |
|---|---|---|---|---|---|---|
| 34100 | battery_2_voltage | uint16 | 0.01 | V | 5125 | 51.25 V |
| 34102 | battery_2_soc | uint16 | 0.1 | % | 120 | 12.0 % |
| 34103 | battery_2_cycle_count | uint16 | 1 | – | 13 | 13 |
| 34105 | battery_2_max_cell_voltage | uint16 | 0.001 | V | 3205 | 3.205 V |
| 34106 | battery_2_min_cell_voltage | uint16 | 0.001 | V | 3203 | 3.203 V |
| 34110 | battery_2_bms_version | uint16 | 1 | – | 116 | **v116** (BMS version, not temperature!) |
| 34111 | battery_2_max_cell_temp | int16 | 0.1 | °C | 235 | 23.5 °C |
| 34118–34133 | battery_2_cell_1..16_voltage | int16 | 0.001 | V | ~3203–3205 | ~3.2 V |

### 6.5 Temperatures

| Register | Name | Type | Scale | Unit | Scan value | Interpreted |
|---|---|---|---|---|---|---|
| 35000 | internal_temperature | int16 | 0.1 | °C | 300 | 30.0 °C |
| 35001 | internal_mos1_temperature | int16 | 0.1 | °C | 313 | 31.3 °C |
| 35002 | internal_mos2_temperature | int16 | 0.1 | °C | 312 | 31.2 °C |
| 35010 | max_cell_temperature | int16 | 0.1 | °C | 216 | 21.6 °C |

### 6.6 Status & alarms

| Register | Name | Type | Scale | Unit | Scan value | Interpreted |
|---|---|---|---|---|---|---|
| 35100 | inverter_state | uint16 | 1 | – | 2 | State code 2 |
| 36000 | alarm_status | uint32 | 1 | Bitmask | 0 | No alarm (2 registers) |
| 36100 | fault_status | uint64 | 1 | Bitmask | 2388 | Fault bits (4 registers) |
| 37007 | max_cell_voltage | uint16 | 0.001 | V | 3339 | 3.339 V |
| 37008 | min_cell_voltage | uint16 | 0.001 | V | 3337 | 3.337 V |
| 37012 | bms_version | uint16 | 1 | – | 116 | **v116** (BMS version, system level) |

### 6.7 Energy counters

| Register | Name | Type | Scale | Unit | Scan value | Interpreted |
|---|---|---|---|---|---|---|
| 33000 | total_charging_energy | uint32 | 0.01 | kWh | (2 registers) | 63.62 kWh |
| 33002 | total_discharging_energy | int32 | 0.01 | kWh | 0 | 0 kWh |
| 33004 | total_daily_charging_energy | uint32 | 0.01 | kWh | 0 | 0 kWh |
| 33006 | total_daily_discharging_energy | int32 | 0.01 | kWh | 0 | 0 kWh |
| 33008 | total_monthly_charging_energy | uint32 | 0.01 | kWh | 0 | 0 kWh |
| 33010 | total_monthly_discharging_energy | int32 | 0.01 | kWh | 0 | 0 kWh |

### 6.8 Firmware & device information

| Register | Name | Type | Scale | Unit | Scan value | Interpreted |
|---|---|---|---|---|---|---|
| 30200 | ems_version | uint16 | 1 | – | 147 | v147 |
| 30202 | vns_version | uint16 | 1 | – | 115 | v115 (**Correction:** OTA-confirmed VNS, not VMS) |
| 30204 | bms_version | uint16 | 1 | – | 116 | v116 |
| 30300 | wifi_status | uint16 | 1 | – | 1 | Connected |
| 30301 | active_inverter_state | uint16 | 1 | – | 3 | **Correction:** NOT bluetooth_status (refuted by backup test behavior) — changes 1→2→3→2→3 during the backup test, loosely correlates with pack rotation; exact function still unclear, see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md |
| 30302 | cloud_status | uint16 | 1 | – | 0 | Not connected |
| 30303 | wifi_signal_strength | uint16 | −1 | dBm | 50 | −50 dBm |
| 30304–30309 | mac_address | ASCII | – | – | – | AA:BB:CC:DD:EE:FF |
| 30350–30355 | comm_module_firmware | ASCII | – | – | – | 20240909_0159 |
| 31000–31009 | device_name | ASCII | – | – | – | VNSD-0 |

---

## 7. Confirmed write registers (40000–49999)

### 7.1 Basic control

| Register | Name | Type | Description | Scan value |
|---|---|---|---|---|
| 41000 | reset_device | uint16 | Device reset (0x55AA=BMS reset, 0xAA11=deeper reset) | 0 |
| 41100 | modbus_address | uint16 | Modbus slave address (1–127) | 1 |
| 41200 | backup_function | uint16 | Backup/UPS function (0=off, 1=on) | 1 |

### 7.2 RS485 control mode

> ⚠️ **Mandatory:** Register 42000 = 0x55AA (21930) must be written first before 42010–42021 take effect!

| Register | Name | Type | Description | Scan value |
|---|---|---|---|---|
| 42000 | rs485_control_mode | uint16 | 0x55AA = RS485 control active | 21930 (active!) |
| 42010 | force_mode | uint16 | 0=no setpoint, 1=charge, 2=discharge | 0 |
| 42011 | charge_to_soc | uint16 | Target SOC (0–100%) | 100 |
| 42020 | set_charge_power | uint16 | Charge power setpoint (W, max 2500) | 0 |
| 42021 | set_discharge_power / inverter_power_limit | uint16 | Discharge power setpoint (W, max 2500); in the backup load test consistently constant at **2200W observed = maximum inverter output power** (explains why 30006 never exceeds ~2209W), see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md | 0 |

### 7.3 Operating mode

| Register | Name | Type | Values | Scan value |
|---|---|---|---|---|
| 43000 | user_work_mode | uint16 | 0=self-consumption, 1=anti-feed-in, 2=trading | 1 |

### 7.4 Scheduling (6 slots of 5 registers each)

Each slot (1–6) occupies registers `43100 + (slot-1)*5` to `43104 + (slot-1)*5`:

| Offset | Name | Type | Description |
|---|---|---|---|
| +0 | schedule_N_days | uint16 | Weekday bitmask (Bit0=Mon, Bit6=Sun) |
| +1 | schedule_N_start | uint16 | Start time as HHMM (e.g. 0800 = 08:00) |
| +2 | schedule_N_end | uint16 | End time as HHMM |
| +3 | schedule_N_mode | int16 | Power in W (−2500 to +2500; negative=charging) |
| +4 | schedule_N_enabled | uint16 | 0=disabled, 1=active |

**Slot addresses:**

| Slot | Days | Start | End | Mode | Enable |
|---|---|---|---|---|---|
| 1 | 43100 | 43101 | 43102 | 43103 | 43104 |
| 2 | 43105 | 43106 | 43107 | 43108 | 43109 |
| 3 | 43110 | 43111 | 43112 | 43113 | 43114 |
| 4 | 43115 | 43116 | 43117 | 43118 | 43119 |
| 5 | 43120 | 43121 | 43122 | 43123 | 43124 |
| 6 | 43125 | 43126 | 43127 | 43128 | 43129 |

### 7.5 Hardware limits (read-only in write range)

| Register | Name | Type | Unit | Scan value |
|---|---|---|---|---|
| 44002 | max_discharge_power | uint16 | W | 2500 |
| 44003 | max_charge_power | uint16 | W | 2500 |

**Correction:** The previous version of this table had 44002/44003 swapped. The
assignment follows the write handler decompilation (section 7.6) and the SRAM table
(section 5): 44002 → SRAM `0x20000156` → max_discharge_power, 44003 → SRAM
`0x20000158` → max_charge_power.

### 7.6 Write handler — complete static analysis (new in v3.1)

> **Source:** Decompilation of `FUN_08050f20` (Write_Handler), 3958 bytes, 942 lines of decompiler output.
> Many of these registers are **write-only** — they return error code 2 on an FC03 read attempt
> (`param_3 == 0 → return 2`). This explains why the FC03 batch scan found only 41 write registers,
> although the descriptor table has 246 entries.

**Overview of all write registers extracted from the code:**

| Register | SRAM target | Function | Write-only? | Persistence |
|---|---|---|---|---|
| **40000** | `DAT_080517a4` | RS485 unlock (0x55AA=enable, 0x55A1–A6=alt modes) | ✅ Yes | SRAM |
| **41000** | `DAT_080517a8` | Command (0x55AA→1, 0xAA11→2) | ✅ Yes | SRAM |
| **41100** | `0x20000133` | Set Modbus slave address | No | SRAM |
| **41200** | `0x20014b6c` | Backup/UPS enable | No | EEPROM 0x300 |
| **41500–41515** | Array | Schedule time settings (u16 array) | ✅ Yes | SRAM |
| **41600–41631** | Array | Schedule power settings (u16 array) | ✅ Yes | SRAM |
| **42011** | `0x200002ee` | charge_target_soc (13–100%) | No | SRAM |
| **42020** | `0x200002f0` | discharge_power_limit (max 2500W) | No | SRAM |
| **43100–43129** | `0x20014b6f+` | Schedule slots (6×5 reg, 10-byte struct) | No | EEPROM 0x36BF+ |
| **43492** | Status flag | OTA/update status (0x55AA/0x55BB/0x55FF) | ✅ Yes | SRAM |
| **43502** | `0x200002ed` | work_mode (0=self-consumption, 1=Anti, 2=Trading) | No | SRAM |
| **43513** | `0x200002f2` | charge_power_limit (max 2500W) | No | SRAM |
| **44002** | `0x20000156` | max_discharge_power (max 2500W) | No | SRAM |
| **44003** | `0x20000158` | max_charge_power (max 2500W) | No | SRAM |
| **44492** | EEPROM 0x301 | Inverter mode (0/1/2) | ✅ Yes | EEPROM 0x301 |
| **45000–45022** | various | Command registers: reboot, reset, SMR, power set, GPIO | ✅ Yes | SRAM |
| **45023–45029** | GPIO | HW debug (GPIO, SMR, relay, WiFi reset) | ✅ Yes | SRAM |
| **45030** | Flag | BLE enable/disable | ✅ Yes | SRAM |
| **45031** | SMR relay | SMR relay control with retry | ✅ Yes | SRAM |
| **45539–45541** | ADC | Read BCD version number (ADC measurement) | ✅ Yes | SRAM |
| **45597** | Status | Read-only status query | No | SRAM |
| **46000** | EEPROM | OTA command (0x5100→EEPROM, various modes) | ✅ Yes | EEPROM |
| **46500–46544** | EEPROM | 8-byte structs, EEPROM-persistent, 3 reg/entry | ✅ Yes | EEPROM |
| **47400** | Debug | Debug register (R: 0xAABB, W: debug_printf) | No | SRAM |

> **⚠️ Discrepancy reg 40000 vs. 42000 (RS485 unlock):**
> The decompiled write handler shows **reg 40000** as the RS485 unlock handler
> (0x55AA check). The previous documentation and live tests use **reg 42000**.
> Possible explanation: reg 40000 is write-only (error 2 on FC03) and was therefore
> never found in the batch scan. In practice reg 42000 works — possibly
> a second code path exists (e.g. via unrecovered jump tables in the write handler),
> or the live documentation refers to a different register with the same
> unlock semantics. **Both variants must be verified by live test.**

**Schedule struct (10 bytes, 6 slots @ reg 43100–43129):**

```c
struct ScheduleSlot {    // 10 bytes, SRAM from 0x20014b6f
    uint8_t  enable;     // +0: 0=disabled, 1=active
    uint16_t start_time; // +1: start time (HHMM)
    uint16_t end_time;   // +3: end time (HHMM)
    int16_t  power;      // +5: power in W (negative=charging)
    uint8_t  mode;       // +7: mode
    uint8_t  reserved[2];// +8: padding
};
// EEPROM persistence from offset 0x36BF
```

---

## 8. Critical peculiarities

### 8.1 battery_soc scale factor (reg 34002)

```
⚠️ Scale = 0.1  (NOT 1!)
Raw value 109 → 10.9%  (NOT 109%)
```

### 8.2 Multi-register values

```python
# read uint32 (energy counter): combine 2 registers
value = (reg_hi << 16) | reg_lo
kWh = value * 0.01

# read uint64 (fault_status): combine 4 registers
value = (r0 << 48) | (r1 << 32) | (r2 << 16) | r3

# read ASCII (mac_address, device_name):
# each register = 2 ASCII characters (big-endian)
char1 = raw >> 8
char2 = raw & 0xFF
```

| Register | Count | Type | Combination |
|---|---|---|---|
| 30304–30309 | 6 | ASCII | MAC address (12 characters) |
| 30350–30355 | 6 | ASCII | Module firmware |
| 31000–31009 | 10 | ASCII | Device name (20 characters) |
| 32302–32303 | 2 | int32 | Off-grid power |
| 33000–33001 | 2 | uint32 | Total charge energy |
| 33002–33003 | 2 | int32 | Total discharge energy |
| 36000–36001 | 2 | uint32 | Alarm status bits |
| 36100–36103 | 4 | uint64 | Fault status bits |

### 8.3 RS485 control sequence

```python
# Order is MANDATORY:
# 1. Enable RS485 control
client.write_register(42000, 0x55AA, slave=1)  # = 21930

# 2. Set mode (wait 1 second)
import time; time.sleep(1)
client.write_register(42010, 1, slave=1)  # force charging

# 3. Set power
client.write_register(42020, 2000, slave=1)  # charge at 2000W
```

### 8.4 Sign conventions

| Register | Positive | Negative |
|---|---|---|
| 30001 battery_power | Charging | Discharging |
| 37004 grid_power_setpoint | Feed-in | Grid import |
| 30303 wifi_signal | Negate raw! (raw×−1) | – |

---

## 9. MQTT field names → register mapping

Found in the flash string pool at `0x08018000`.
The *Local API* column shows the equivalent field from the official
Marstek Local API REV 0.5 — as an additional verification source:

| MQTT field | Register | Description | Local API equivalent |
|---|---|---|---|
| `bat_soc` | 34002 | Battery SOC (scale 0.1!) | `ES.GetStatus.bat_soc` / `Bat.GetStatus.soc` |
| `bat_temp` | 35000 | Battery temperature | `Bat.GetStatus.bat_temp` |
| `bat_cap` / `bat_capacity` | – | Battery capacity | `Bat.GetStatus.bat_capacity` (current) / `rated_capacity` (rated) |
| `ongrid_power` | 30006 | Grid power (AC) | `ES.GetStatus.ongrid_power` |
| `offgrid_power` | 32302 | Off-grid power | `ES.GetStatus.ogrid_power` |
| `pv_power` | MPPT sum | Total PV power | `ES.GetStatus.pv_power` / `PV.GetStatus.pv_power` |
| `total_pv_energy` | – (open) | Total PV energy | `ES.GetStatus.total_pv_energy` |
| `total_grid_output_energy` | 33002–33003 | Total feed-in | `ES.GetStatus.total_grid_output_energy` |
| `total_grid_input_energy` | 33000–33001 | Total grid import | `ES.GetStatus.total_grid_input_energy` |
| `total_load_energy` | – (open) | Load energy | `ES.GetStatus.total_load_energy` |
| `work_mode` | 43000 | Operating mode | `ES.GetMode.mode` ("Auto"/"Manual"/"AI"/"Passive") |
| `tol_charge` | 33000 | Total charge energy | `ES.GetStatus.total_grid_input_energy` |
| `tol_discharge` | 33002 | Total discharge energy | `ES.GetStatus.total_grid_output_energy` |
| `connect` / `disconnect` | 30300/30302 | Connection status | `Wi.GetStatus` / `BLE.GetStatus` |
| `ble_mac` | 30304 | Bluetooth MAC | `Marstek.GetDevice.ble_mac` |
| `wifi_mac` | 30304 | WiFi MAC | `Marstek.GetDevice.wi_mac` |
| `wifi_name` | – | SSID | `Marstek.GetDevice.wi_name` / `Wi.GetStatus.ssid` |

---

## 10. Marstek Local API — parallel protocol & cross-validation

> **⛔ WARNING — DO NOT ACTIVATE:**
> Several user reports prove that activating the Marstek Local API
> leads to permanently wrong or frozen Modbus register values.
> The fault does not go away even after a device reset or deactivation of the API.
> The cause is a shared internal state (mutex / SRAM pointer) between
> the Modbus TCP task and the UDP JSON task in the FreeRTOS scheduler.
> The official document itself warns: *"some native functions of the device
> may be disabled to avoid command conflicts".*
>
> **The Local API is not used for the HA integration.**
> It serves here exclusively as a reference source for verifying
> our Modbus register meanings.

### 10.1 Protocol overview (reference)

| Property | Value |
|---|---|
| Document | Marstek Device Open API REV 0.5 (Draft, 04.07.2025) |
| Transport | UDP, unicast and broadcast |
| Port | Default 30000, configurable (recommendation: 49152–65535) |
| Format | JSON-RPC (method + params → result) |
| Activation | Marstek APP or BLE command 0x28 — **avoid both** |
| Authentication | None (purely local, no token) |

### 10.2 Available methods (reference)

| Method | Function | Type |
|---|---|---|
| `Marstek.GetDevice` | Discover device in the LAN | Read |
| `Wi.GetStatus` | WiFi status and network info | Read |
| `BLE.GetStatus` | Bluetooth connection status | Read |
| `Bat.GetStatus` | Battery SOC, temp, capacity, error code | Read |
| `PV.GetStatus` | PV power, voltage, current | Read |
| `ES.GetStatus` | Energy system status (powers, counters) | Read |
| `ES.GetMode` | Read current operating mode | Read |
| `ES.SetMode` | Set operating mode (Auto/AI/Manual/Passive) | **Write** |

### 10.3 Cross-validation: Local API ↔ Modbus registers

The following links were determined by reconciling the API example values
(REV 0.5) with our Modbus scan results. All matches
confirm the correctness of our RE results.

| Local API field | Modbus reg. | Scale | Verification |
|---|---|---|---|
| `ES.GetStatus.bat_soc` = 98 | **34002** battery_soc | ×0.1 → raw=980 | ✅ **Scale 0.1 officially confirmed** |
| `ES.GetStatus.bat_power` = 501 W | **30001** battery_power | ×1 (int16) | ✅ Meaning confirmed |
| `ES.GetStatus.ongrid_power` = 100 W | **30006** ac_power | ×1 (int16) | ✅ Meaning confirmed |
| `ES.GetStatus.ogrid_power` = 0 W | **32302–32303** offgrid_power | int32 | ✅ Meaning confirmed |
| `ES.GetStatus.pv_power` = 0 W | **30037–30040** mppt*_power | ×0.1, sum | ✅ (PV not connected) |
| `ES.GetStatus.total_grid_input_energy` = 3273 Wh | **33000–33001** uint32 | ×0.01 kWh | ✅ Unit Wh confirmed |
| `ES.GetStatus.total_grid_output_energy` = 2548 Wh | **33002–33003** uint32 | ×0.01 kWh | ✅ Unit Wh confirmed |
| `Bat.GetStatus.soc` = 90 | **34002** battery_soc | ×0.1 → raw=900 | ✅ Second confirmation of scale 0.1 |
| `Bat.GetStatus.bat_temp` = 25.0 °C | **35000** internal_temperature | ×0.1 → raw=250 | ✅ Scale 0.1 for temp confirmed |
| `Bat.GetStatus.rated_capacity` = 5120 Wh | **35110 / 35111** (candidate) | ×1 Wh | ⚠️ Register not yet unambiguously mapped |
| `Bat.GetStatus.error_code` = "0x430" | **36100–36103** fault_status | bitmask | ⚠️ Bit mapping still open |
| `PV.GetStatus.pv_voltage` = 40.0 V | **30020–30023** mppt*_voltage | ×0.1 → raw=400 | ✅ Scale 0.1 confirmed |

### 10.4 Mode mapping: API ↔ Modbus

| Local API `ES.SetMode.mode` | Modbus reg. 43000 | Modbus control register | Meaning |
|---|---|---|---|
| `"Auto"` | `0` (self-consumption) | – | Automatic self-consumption |
| `"AI"` | no direct equivalent | – | AI-controlled mode (cloud function) |
| `"Manual"` + `manual_cfg` | `1` (anti-feed-in) + schedules | 43100–43129 | Time-controlled charge/discharge scheduling |
| `"Passive"` + `power` + `cd_time` | force_mode 42010=1 or 2 + 42020/42021 | – | One-off charging/discharging with time limit |

> **Note on "Passive":** The parameter `cd_time` (countdown in seconds) has
> no direct Modbus equivalent in our scanned registers.
> It is presumably managed internally by the EMS task and cannot be read/set
> via Modbus. For HA automations, force_mode control via Modbus (42010 + 42020/42021)
> is fully sufficient.

### 10.5 New register candidates from API fields

These API fields do not yet have a confirmed Modbus equivalent:

| API field | Type | Candidate register | Rationale |
|---|---|---|---|
| `ES.GetStatus.total_pv_energy` | Wh | 33004–33005? | Analogous to the grid_input/output pattern |
| `ES.GetStatus.total_load_energy` | Wh | 33006–33007? | Another uint32 counter in the 33000 block |
| `Bat.GetStatus.rated_capacity` | Wh (5120) | 35110 or 35111 | raw=576→57.6? No. Or directly 5120? |
| `Bat.GetStatus.bat_capacity` | Wh (current) | 34008? | Remaining capacity in Wh |
| `ES.GetStatus.bat_cap` | Wh total | 30003 or 30004? | 30004=2364 does not fit; 5120 not yet seen |

---

## 11. Marstek Cloud API & OTA infrastructure

### 11.1 Analysis tool

The open-source tool **marstek-fw-checker** (by Remko Weijnen, GitHub: `rweijnen/marstek-fw-checker`)
was used to analyze the Marstek cloud API. It runs as a static web page
with Netlify functions as a CORS proxy and allows login, device query and OTA check.

```bash
# Local start (no Netlify needed for basic function):
cd marstek-fw-checker-master/
python -m http.server 8000
```

### 11.2 Cloud API endpoints (from script analysis)

| Endpoint | Path | Purpose |
|---|---|---|
| Authentication | `/app/Solar/v2_get_device.php` | Login + device list (MD5 password) |
| Device list (detail) | `/ems/api/v1/getDeviceList` | Detailed device data incl. versions |
| OTA standard | `/ems/api/v2/checkSmallBalconyOTA` | Firmware check for Venus devices |
| OTA CT devices | `/ems/api/v1/checkAcCoupleOta` | Firmware check for HME-3/HME-4 |
| Settings | `/ems/api/v1/getAdvance` | Advanced device settings |

**Base URL (EU):** `https://eu.hamedata.com`

### 11.3 OTA request parameters (for VNSD-0)

The tool sends `m=100` as the baseline version in order to obtain all available
firmware entries. The complete request for the Venus D:

```
GET https://eu.hamedata.com/ems/api/v2/checkSmallBalconyOTA
    ?uid=<devid>
    &device_type=VNSD-0
    &m=100
    &sbv=0
    &mppt=0
    &inv=0
    &click=false
    &lang=English
    &is_fourDigit={"control":false,"bms":false,"micro":false,"mppt":false}
    &token=<auth-token>
    &mailbox=<email>
```

**Result for VNSD-0 (status July 2026):** VNSD-0 is now **listed in the OTA system**.
Available: **`control` v149** and **`micro` v116**. Earlier documentation (May 2026) reported
an empty response — the server entries for VNSD-0 have been added since then.

**Staged rollout:** OTA distribution happens in waves. Even for
supported device types, not all devices receive firmware updates at the
same time — depending on device age, region and
possibly `dlock` status.

### 11.4 Complete MCU component matrix (OTA API)

The `/ems/api/v2/checkSmallBalconyOTA` response contains **9 component fields**.
The complete response for VNSD-0 (18.05.2026) reads:

```json
{
  "code": 1,
  "show": 0,
  "msg": "success",
  "data": {
    "control":   "",
    "bms":       "",
    "mppt":      "",
    "micro":     "",
    "dcdc":      "",
    "bms_pack1": "",
    "bms_pack2": "",
    "led":       "",
    "charger":   ""
  }
}
```

> **New compared with Rev 7:** The fields `dcdc`, `led` and `charger` were previously
> undocumented. They were identified by evaluating the complete raw API response.

For VNSD-0, `control` (v149) and `micro` (v116) are available; the remaining fields stay empty.

#### Complete component table of all 9 API fields

| API field | MCU / subsystem | Description | VNSD-0 (July 2026) |
|---|---|---|---|
| `control` | EMS main controller | Energy management, Modbus TCP, WiFi | **v149** |
| `bms` | BMS main controller | Battery management system (all packs) | – (empty) |
| `micro` | Inverter MCU | Inverter controller | **v116** |
| `mppt` | MPPT controller | Solar charge controller | – (empty) |
| `dcdc` | DC/DC converter MCU | Bidirectional DC/DC converter | – (empty) |
| `bms_pack1` | Pack 1 BMS | Individual pack 1 firmware | – (empty) |
| `bms_pack2` | Pack 2 BMS | Individual pack 2 firmware | – (empty) |
| `led` | LED controller | Status display MCU / LED control | – (empty) |
| `charger` | Charge controller | AC charger / grid charger | – (empty) |

> **Note on `bms_pack1`/`bms_pack2`:** Pack-specific OTA fields are not populated
> server-side, even though the device supports multiple packs.

#### `is_fourDigit` parameter — meaning for version registers

The request contains:
```json
"is_fourDigit": {"control": false, "bms": false, "micro": false, "mppt": false}
```

This flag shows whether the respective component has a 4-digit version number.
Currently the device reports `false` everywhere (3-digit versions: 147, 116, 115).
When a new firmware generation switches to 4-digit numbers (e.g. 1001),
this flag becomes `true` — useful as an indicator of major updates.

> **Security note:** The auth token (`token=`) and email address are in
> plaintext in the OTA request URL. Logs containing this URL should be cleaned up before
> being passed on.

### 11.5 Cloud device record — Marstek Venus D

Complete API response for the analyzed device (`/ems/api/v1/getDeviceList`):

```json
{
  "devid":        "<DEVICE_UID>",
  "name":         "MST_VNSD_xxxx",
  "sn":           "MVDxxxxxxxxxx",
  "mac":          "xxxxxxxxxxxx",
  "type":         "VNSD-0",
  "access":       "1",
  "bluetooth_name": "MST_VNSD_xxxx",
  "date":         "2026-04-21 11:20:08",
  "id":           <CLOUD_ID>,
  "dlock":        "Y",
  "timeZone":     "Europe/Berlin",
  "version":      "147",
  "soc":          80,
  "battery_num":  1,
  "status":       1,
  "isVppDevice":  "N"
}
```

### 11.7 Analysis of the device record

| Field | Value | Meaning / peculiarity |
|---|---|---|
| `type` | `VNSD-0` | Confirms device type |
| `version` | `"147"` | **Control/EMS firmware** (not BMS) — identical to reg 30200 |
| `soc` | `80` | Cloud SoC at query time (reported by the device) |
| `battery_num` | `1` | ⚠️ Cloud reports only 1 pack, although 2 are installed |
| `dlock` | `"Y"` | **Device lock active** — could block OTA update |
| `salt` | `"d1f1900057185ec9,d1f1900057185ec9"` | Two identical values, comma-separated — possibly one salt per pack |
| `date` | `2026-04-21` | Registration date (device relatively new) |
| `isVppDevice` | `"N"` | No virtual power plant / aggregator |
| `status` | `1` | Online |

**On `battery_num: 1`:** The cloud sees only one pack, although both packs
can be read via Modbus (registers 34000 and 34100). The device probably reports
only the primary pack in the MQTT heartbeat — or the cloud logic
counts the device itself as "battery unit 1".

**On `dlock: "Y"`:** Device lock is an unknown flag that is presumably
set server-side by Marstek. Possible meanings: support hold,
known incompatibility of the current firmware with the update,
or a general block for unofficial device types in the OTA system.

---

### 11.8 OTA components → Modbus version register mapping

The 9 API components can be partly traced back to Modbus registers.
The pattern in the 30200 block is: one register pair per MCU (main + sub version).

#### Confirmed assignments

| Register | Value | API field | Description |
|---|---|---|---|
| **30200** | 147 | `control` | EMS firmware version (main version) ✅ confirmed |
| 30201 | 18 | `control`? | EMS sub/patch version (right next to 30200) |
| **30202** | 115 | — | VNS firmware version (no dedicated API field) — **correction:** OTA-confirmed VNS, not VMS |
| 30203 | 106 | — | VNS sub-version |
| **30204** | 116 | `bms` | BMS firmware version ✅ confirmed |
| **30205** | 104 | `mppt`? | **mppt_version** (MPPT firmware version) ✅ OTA-confirmed — **correction:** previously wrongly listed as `bms_version_sub`; OTA screenshot shows 4 FW components (EMS/VNS/MPPT/BMS), 30205=104 corresponds to MPPT:104, see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md |

#### Missing components in the scan

Registers 30206–30209 do **not** exist in the scan (the device delivered no data there
— either not present or value = 0 and no response).

| Register (hypothetical) | Possible API field | Status |
|---|---|---|
| 30206/30207 | `mppt` | Not in the scan — MPPT probably not installed |
| 30208/30209 | `dcdc` / `micro` | Not in the scan |

> **Conclusion:** `dcdc`, `led` and `charger` are either:
> (a) sub-components without dedicated Modbus version registers, or
> (b) their registers lie outside the 30200 block (e.g. in the 37000 range).
> **Open:** Register 37006 = 192 is unexplained — could be the DCDC version.
> **Close the scan gap:** read 30206–30209 manually (single register,
> timeout 500 ms) to confirm or rule out the `mppt`/`dcdc` registers.

#### LED control via Modbus

The `led` entry in the OTA API corresponds to known
Modbus write functions from the binary analysis:

| Function (from binary) | Chinese | Modbus equivalent |
|---|---|---|
| `ctrl_led_light` | 控制led灯亮度 | Write command (40000 range, unknown address) |
| `led_continue_ctrl` | 控制类型 | LED permanent: 0=on; 1=off + duration ms |
| `Led.Ctrl` | — | Higher-level MQTT topic |

> The exact Modbus addresses for LED control have not yet been identified in the 40000 scan
> — probably in the unscanned range 44004–49999.

### 11.8 Cloud telemetry field names (from FW strings, Ghidra)

The FW contains in its data area (0x0805A000+) all format strings for the cloud reports.
The main format string for `setVenusDReporting` is at 0x0805EC04 (beyond the
loaded binary — probably extended flash data). The remaining format strings
are present in the binary and fully decoded:

#### Report type 1: Main telemetry (`setVenusDReporting`)
Format string at 0x0805a404 (485 bytes):
```
di=%s       → device_id (String)
sn=%s       → serial_number (String)
to=%d       → total_output (Energy)
td=%d       → total_discharge
ed=%d       → energy_daily
em=%d       → energy_monthly
gd=%d       → grid_daily
gm=%d       → grid_monthly
wm=%d       → work_mode
gy=%d       → grid_yield
gp=%d       → grid_power
go=%d       → grid_offset/output
gt=%d       → grid_total
gf=%d       → grid_frequency
gv=%d       → grid_voltage
ct=%d       → ct_value (Current Transformer)
bs=%d       → battery_soc
bp=%d       → battery_power
eb=%lx      → energy_battery (hex u32)
dn=%d       → device_number
bu=%d       → battery_usage
t1=%d       → temp_1 (Inverter NTC)
t2=%d       → temp_2
t3=%d       → temp_3
vc=%s       → version_control (String)
tc=%s       → time_config (String)
dt=%d-%02d-%02d %02d:%02d:%02d → datetime
no=%d       → number/count
e1=%llx     → energy_1 (64-bit hex)
ws=%d       → wifi_status
mt=%d       → meter_type
ty=%d       → type
sv=%d       → software_version
mc=%d       → meter_config
md=%d       → mode
sc=%d       → state_code
ci=%d       → charge_current
ri=%d       → rated_current
bv=%d       → battery_voltage
bi=%d       → battery_current
pb=%d       → pv_battery (power)
ds=%d       → discharge_status
ph=%d       → phase
bt=%d       → battery_temp
mq=%d       → mqtt_status
cy=%d       → cycle_count
cw=%d       → charge_watt
cp=%s       → charge_plan (String)
bm=%d       → bms_mode
ce=%s       → cert_info (String)
up=%d       → uptime
ap=%d       → ap_mode
nt=%d       → network_type
iv=%d       → inverter_version
et=%lx      → energy_total (hex u32)
ea=%lx      → energy_accumulated (hex u32)
pv=%s       → pv_info (String)
sk=%s       → secret_key (String)
mv=%d       → micro_version
me=%lx      → micro_energy (hex u32)
ma=%lx      → micro_accumulated (hex u32)
fu=%s       → firmware_url (String)
ms=%s       → mac_string (String)
im=%s       → imei (String)
hd=%s       → hardware_id (String)
pw=%d       → power_watt
bl=%d       → backup_level
as=%d       → auto_switch
bke=%s      → backup_energy (String)
bkd=%s      → backup_data (String)
ip=%s       → ip_address (String)
bt_p=%s     → battery_pack_info (String)
ival=%d     → interval
soh=%d      → state_of_health
```

#### Report type 2: BMS per-pack (0x0805a9d4, 354 bytes)
```
cd=%d       → command/report_code
b_ver=%d    → BMS Version
b_chv=%d    → charge_voltage (max)
b_rci=%d    → rated_charge_current
b_rdi=%d    → rated_discharge_current
b_soc=%d    → SOC (÷10!)
b_soh=%d    → SOH
b_cap=%d    → capacity
b_vol=%d    → voltage
b_cur=%d    → current
b_tem=%d    → temperature
b_chf=%d    → charge_flag
b_slf=%d    → sleep_flag
b_cpc=%d    → charge_protect_count (cycles)
b_err=%d    → error_bitmask
b_war=%d    → warning_bitmask
b_ret=%d    → return_energy (charge total)
b_ent=%d    → energy_total (discharge)
b_mot=%d    → mos_temperature
b_tp1..4=%d → cell_temp_1..4
b_vo1..16=%d → cell_voltage_1..16
```

#### Report type 3: EMS status (0x0805ac80, 845 bytes)
```
cd, tot_i, tot_o, ele_d, ele_m, grd_d, grd_m, inc_d, inc_m,
grd_f, grd_o, grd_t, gct_s, cel_s, cel_p, cel_c,
err_t (hex), err_a (hex), dev_n, grd_y, wor_m,
tim_0..9 (7 pipe-separated values each = 10 time slots × 7 parameters),
cts_m, bac_u, tra_a, tra_i, tra_o, htt_p,
prf_s, inc_a, set_v, mcp_w, mdp_w, ct_t, phase_t, dchrg_t,
bms_v, fc_v (String), wifi_n (String), seq_s, ctrl_r,
par, gen, ble, shelly_p, c_ratio, udp, api, net, port,
inv_v, id (5 pipe-sep.), lk, bp, ei (hex64), eb (hex32),
rp, gp, vp, bl, dod, bl_p, led, as, lf, pl, soh
```

#### Report type 4: BMS multi-pack summary (0x0805ab38, 226 bytes)
```
cd, BMS: num, mask, idx, charge_pow, discharge_pow,
soc1..6, state1..6, temp1..6 (6 packs × 3 fields)
```

#### Report type 5: Meter (0x0805ac1c, 96 bytes)
```
cd, meter: type, real_tol_power, real_power1..3, err_flag
```

#### Report type 6: RS485/inverter (0x0805afd0, 107 bytes)
```
cd, 485_a, grd_f, grd_v, grd_a, temp_1..5, 485_c, ssl
```

#### Report type 7: MPPT/PV (0x0805b1d0, 209 bytes)
```
cd, state, err, war, temp,
pv1v/pv1c/pv1p .. pv4v/pv4c/pv4p (4 MPPT strings × V/C/P),
pve, pvs, pow, capd, capm, capy, batv, batc, basev, pev
```

#### Report type 8: OTA URL info (0x0805b360, 237 bytes)
```
cd, type1..4, mod1..4, size1..4, crc1..4, ver1..4, len1..4, url1..4
(4 firmware slots with type/mod/size/crc/ver/url_len/url each)
```

#### Report type 9: Extended MPPT append (0x0805b450, 93 bytes)
```
%.*s (preceding report), mppt, pv1..4 (V|P each),
pack (4 pipe-sep.), pv (2 pipe-sep.), fu (2 pipe-sep.), em (hex)
```

> **Insight:** The format strings contain NO JSON structure but
> URL query parameters (`key=value&key=value` or `key=%d,key=%d`).
> The AES encryption in `setVenusDReporting` applies to the entire
> payload (after Base64 encoding).

---

## 11b. Ghidra — Complete function list

> **Note (2026-07-09):** Until July 2026 this section contained an early partial function list
> (168 of 1615 functions, status "FW v148" — even the label was already outdated compared to 149.2).
> It was **removed** because it was no longer in sync with the now complete and continuously corrected
> function tracking documentation and partly contained contradictory address/name assignments
> (among others, `0x0802a4ec` was still wrongly listed here as `cJSON_CreateObject`, although
> the function is actually `cJSON_InitHooks` — see batch 18 in
> [VNSD_Ghidra_Analysis_Findings.md](../Methodik_und_Meta/VNSD_Ghidra_Analysis_Findings.md) and
> [VNSD_Control_FW_Naming_Batch_History.md](VNSD_Control_FW_Naming_Batch_History.md)).
>
> **The authoritative source for function names/addresses is exclusively
> [VNSD_Control_FW_Function_Tracking_new.md](VNSD_Control_FW_Function_Tracking_new.md)**
> (ordered by topic; the earlier chronological batch log was deleted on 2026-07-11, see
> [VNSD_Control_FW_Naming_Batch_History.md](VNSD_Control_FW_Naming_Batch_History.md)).

## 12. Open items & next steps

### 12.1 Register address space — status

| Range | FC03 scan | Static analysis | Status |
|---|---|---|---|
| 0 – 29,999 | ✅ empty | — | completed |
| 30,000 – 39,999 | ✅ 89 registers | — | completed |
| 40,000 – 49,999 | 41 via FC03 | **~50+ from write handler** | ⚠️ extended |
| 50,000 – 65,535 | ✅ empty | — | completed |

> **Important:** The FC03 batch scan found only 41 write registers because many registers
> are **write-only** (error code 2 on FC03 read access). The static analysis of the
> write handler (`FUN_08050f20`) uncovered ~50+ additional registers
> (see section 7.6), including command registers (45000–45031), OTA commands (46000),
> EEPROM structs (46500–46544) and a debug register (47400).

### 12.2 Open registers from Local API reconciliation (priority high)

The following API fields still have no confirmed Modbus register.
Since the complete address space has been scanned, these
registers — if present — are located in the known range 30000–49999:

| API field | Expected value | Candidate register | Next step |
|---|---|---|---|
| `ES.GetStatus.total_pv_energy` | Wh counter | 33004–33005 | Check during PV operation |
| `ES.GetStatus.total_load_energy` | Wh counter | 33006–33007 | Check |
| `Bat.GetStatus.bat_capacity` | currently available Wh | 34008? 34009? | Targeted single read |
| `ES.GetStatus.bat_cap` | Nominal capacity (5120 Wh) | 35110 or 35111 | Raw value 576 ≠ 5120 → different scale? |
| `Bat.GetStatus.error_code` | "0x430" as string | 36100–36103 | Bitmask mapping missing |
| `ES.GetMode.mode` string | "Auto"/"Manual" etc. | 43000 | Verify enum mapping 0→Auto, 1→? |

### 12.2a OTA components without Modbus mapping (priority medium)

Three newly identified OTA fields (`dcdc`, `led`, `charger`) still have
no confirmed Modbus version registers:

| OTA field | Hypothesis | Candidate register | Next step |
|---|---|---|---|
| `dcdc` | DC/DC converter firmware version | 30206/30207? | Single read; not present in the scan |
| `led` | LED controller firmware version | 30208/30209? | Single read; not present in the scan |
| `charger` | AC charger firmware version | unknown | No candidate known |
| LED write command | `ctrl_led_light` from binary | 44xxx? | Catch up the 40000s scan from 44004 |
| DCDC status register | Power / efficiency | 37006 = 192? | Verify with single read |

```python
# Close scan gap 30206-30209 (single-register, timeout 500ms)
from pymodbus.client import ModbusTcpClient
import time

c = ModbusTcpClient("192.168.X.X", port=502, timeout=0.5)
c.connect()
for reg in [30206, 30207, 30208, 30209, 37006]:
    try:
        r = c.read_holding_registers(reg, 1, slave=1)
        if not r.isError():
            print(f"Reg {reg}: {r.registers[0]}")
        else:
            print(f"Reg {reg}: Error (not present)")
    except Exception as e:
        print(f"Reg {reg}: Exception → {e}")
    time.sleep(0.1)
c.close()
```

### 12.3 Descriptor table (246 entries) — not yet extracted

The table at SRAM `0x20000354` was derived empirically live through the batch scan
(413 registers). For the exact data type and
scale information of all 246 entries, a direct extraction would be useful:

```
Possible approaches:
1. JTAG/SWD: read 2952 bytes from 0x20000354 on the running device
2. Unicorn emulation: emulate the binary until SRAM is initialized
3. Ghidra: find router callers via References-to-DAT_20000354
```

Ghidra instructions:
```
1. Open FUN_0801c088 (router) in Ghidra (G → 1c088)
2. F5 (decompiler)
3. Right-click on 'DAT_20000354'
4. References → Show References to Address
5. WRITE references → that is the init function
```

### 12.4 Known register gaps

From the descriptor table schema (246 entries), 413 scanned registers
(in 89 read + 41 write groups) and ~50+ statically extracted write registers
(see 7.6), the following open items remain:

- **33004–33011**: Further energy counters (total_pv_energy, total_load_energy)
- **34008–34009**: Current battery capacity in Wh (bat_capacity)
- **30000–30010**: First registers not yet fully decoded (30000=509 unexplained)
- Extended alarm/fault codes with bit definitions

### 12.5a Open verifications from static analysis (priority high)

| Topic | Details | Next step |
|---|---|---|
| **Reg 40000 vs. 42000** | Code shows 40000 as RS485 unlock (0x55AA), documentation/live uses 42000 | Test FC06 write to 40000 with 0x55AA |
| **Reg 41500–41631** | Schedule arrays (previously undocumented) | Set a schedule, then read the registers |
| **Reg 45000–45031** | Command registers (reboot, SMR, GPIO) — **dangerous!** | Test only after backup |
| **Reg 46000** | OTA command register | **DO NOT test** without an OTA image |
| **Reg 46500–46544** | EEPROM structs (function unknown) | Read/write after backup |
| **Reg 47400** | Debug (reads 0xAABB, writes debug_printf) | Harmless, test reading |

### 12.5 Verification code for critical registers

```python
from pymodbus.client import ModbusTcpClient

c = ModbusTcpClient("192.168.1.XXX", port=502)
c.connect()

# SOC (scale 0.1!) — confirmed by Local API REV 0.5
raw = c.read_holding_registers(34002, 1, slave=1).registers[0]
print(f"SOC Pack1: {raw * 0.1:.1f}%")

# SOC Pack2
raw2 = c.read_holding_registers(34102, 1, slave=1).registers[0]
print(f"SOC Pack2: {raw2 * 0.1:.1f}%")

# Grid voltage
v = c.read_holding_registers(32200, 1, slave=1).registers[0]
print(f"Grid: {v * 0.1:.1f} V")

# uint32 charge energy (total_grid_input_energy)
hi = c.read_holding_registers(33000, 1, slave=1).registers[0]
lo = c.read_holding_registers(33001, 1, slave=1).registers[0]
print(f"Total grid import: {((hi << 16) | lo) * 0.01:.2f} kWh")

# uint32 discharge energy (total_grid_output_energy)
hi2 = c.read_holding_registers(33002, 1, slave=1).registers[0]
lo2 = c.read_holding_registers(33003, 1, slave=1).registers[0]
print(f"Total feed-in: {((hi2 << 16) | lo2) * 0.01:.2f} kWh")

c.close()
```

---

## 13. Ghidra detailed analyses: interesting findings

In-depth analyses of individual items from the "Interesting findings" list in
`VNSD_Control_FW_Function_Tracking_new.md`. Each subsection goes beyond the coarse
batch naming (cross-references, data structures, security assessment).

### 13.1 CAN bus protocol (item #4)

**Functions:** `CAN_ReadMailbox` (0x08002a20), `CAN_SetupTxMailbox` (0x08002b10),
`CAN_Detect_Mismatched_Nodes` (0x08004418), `CAN_Update_StateMachine` (0x080045dc)

Additionally identified, related functions:

| Address | Name |
|---|---|
| `0x0802dfd4` | `CAN_RxMailbox_Handler` (ISR-near receive routine) |
| `0x08002000` | `USART_SendData` (**misnamed** — internally only calls `CAN_SetupTxMailbox`, no USART access whatsoever) |
| `0x080056b4` | `CAN_Select_Master_Node` |
| `0x08005418` | `CAN_Node_Data_Reset` |
| `0x0800557c` | `CAN_Parallel_Inverter_Sync` |
| `0x08027224` | `CAN_Update_Init` (state 0) |
| `0x080272e0` | `CAN_Update_Erase` (state 1) |
| `0x08027074` | `CAN_Update_SendData` (state 2) |
| `0x08026f08` | `CAN_Update_HandleResponse` (state 3) |
| `0x080271a4` | `CAN_Update_Success` (state 4) |
| `0x080271e4` | `CAN_Update_Failed` (state 5) |
| `0x080049a4` | `CAN_Update_Check_Retry_Limit` |
| `0x080358ec` | `CAN_Battery_Telemetry_Debug_Print` (PGN 1801–1804, 1007) |

**Purpose:** Second communication channel (STM32 bxCAN1, peripheral base `0x40006400`,
register offsets confirmed 1:1 from the STM32 reference manual) alongside RS485/Modbus.
Used for (1) parallel operation synchronization of multiple battery/inverter nodes
(`CAN_Select_Master_Node`, `CAN_Parallel_Inverter_Sync`, J1939-style telemetry with
PGN 1801–1804/1007) and (2) a remote firmware update protocol for CAN nodes
(`CAN_Update_StateMachine`).

**Frame format:** `CAN_ReadMailbox`/`CAN_SetupTxMailbox` are generic bxCAN HAL routines
(no proprietary bit layout), supporting standard (11 bit) and extended ID (29 bit).
Common message structure (stack, 0x14 bytes):

```c
struct CAN_Message {
    uint32_t std_id;   // +0x00, valid when IDE==0
    uint32_t ext_id;   // +0x04, valid when IDE==1
    uint8_t  ide;       // +0x08
    uint8_t  rtr;        // +0x09
    uint8_t  dlc;         // +0x0A (0-15)
    uint8_t  data[8];      // +0x0B..0x12
};
```

`CAN_RxMailbox_Handler` reads via FIFO0 and passes messages to a FreeRTOS queue
(`*DAT_0802e000`); the consumer task of this queue was not identified (open question).

**State machine (`CAN_Update_StateMachine`, runs every main loop tick via
`App_MainLoopDispatcher`):** Session struct at RAM `0x20015320`. Guard: the FSM only runs
if the master node index (`0x20000282`) < 3.

| State | Function | Meaning |
|---|---|---|
| 0 | `CAN_Update_Init` | Zero buffers, select target firmware, **`Inverter_StopOutput()`**, start log |
| 1 | `CAN_Update_Erase` | Send erase info, 10-tick delay |
| 2 | `CAN_Update_SendData` | Read FW block from QSPI, compute CRC, send block by block |
| 3 | `CAN_Update_HandleResponse` | Evaluate node response (1=continue, 2=retry, 3=error→state 1, 0=poll) |
| 4 | `CAN_Update_Success` | Log, reset to master index `0xFF` |
| 5 | `CAN_Update_Failed` | Log, reset to master index `0xFF` |

A 20-tick timeout forces state 5 immediately on expiry (regardless of the current state); retry
> 25 likewise forces state 5. A second, 5-tick timeout additionally exists in state 3
for the "no response" branch.

**Node mismatch detection:** Compares up to N node addresses (stride 0x60 bytes) against the
own reference address. Expects exactly 1 deviating peer (normal case); 0 or >1
deviating peers count as an error. Trigger flag (`DAT_0800447c`) and consumer of the
error result not found.

**Security assessment:** No authentication/signature verification at CAN level; the CRC in the
update path only protects the transmission, not the authenticity of the sender. The
update cycle calls `Inverter_StopOutput()` in state 0 — with physical CAN bus access,
forged status responses (faking success, forcing retry loops, blocking the
update permanently/DoS) appear plausible. Full code execution via this path
could not be demonstrated from the analyzed functions. The trust model assumes
a physically secured, internal CAN network.

**Anomaly:** `Modbus_SendResponse` (0x0802c060) calls `USART_SendData` (0x08002000),
which despite its name exclusively fills CAN mailboxes — no USART registers
are touched. Historical misnaming or a deliberate dual-transport architecture (Modbus
additionally mirrored onto CAN)? Unresolved, see also 13.2/13.3.

**Open questions:** Trigger of `DAT_0800447c`; consumer of the mismatch error flag;
FreeRTOS queue consumer of the CAN RX data; exact field meaning of the node array entries
(`0x200152F0`); origin/trigger of the firmware update cycle.

**Addendum (deep dive):**

1. **Trigger `DAT_0800447c`:** Not a standalone RAM flag, but a flash literal pool
word in `CAN_Detect_Mismatched_Nodes` that simply contains the pointer `0x20000282` —
the same master node index that was already known as the FSM guard. The code dereferences
it (`*DAT_0800447c` == `*0x20000282`); the mismatch check only runs if the
master node index is exactly `1` (BMS slot).

2. **Consumer of the mismatch error flag** (structure `0x2000027a`, written by
`CAN_Detect_Mismatched_Nodes` — `+0x00` error byte, `+0x01` index of last
mismatch, `+0x02` node count, `+0x06` reference ID): **`CAN_Parallel_Inverter_Sync`** reads
byte 0. At `0` (exactly 1 mismatch = normal case) fast path: node index (`+0x01`) indexes
directly into table `0x20014e3c`, values are taken over to `0x20014e24+8/+0xc`. At `!=0`
(0 or >1 mismatches) the already documented maximum fallback runs over the byte+6 fields
of `0x20014e3c`, controlled by the update session state (`0x20015320+0x12`: state 2/3 from
`CAN_Update_StateMachine`). Second reader: `Register_WriteCategory0xCE` (0x080265f0) uses
byte 0 as a condition for a sequence number reset during the CAN update register write.

3. **FreeRTOS queue consumer identified:** `CAN_RxQueue_DrainAndDispatch` (0x080292d4) —
reads the queue handle from RAM `0x200000e8` (via xQueueReceive, internally misnamed as
`Modbus_Response_Builder`, 1000-tick timeout), takes up to 64 frames per call, dispatches
each via `CAN_FrameDispatcher`, then `vTaskDelay(1)`. Its own caller (task entry point /
xTaskCreate site) cannot be resolved by Ghidra (0 xrefs) — presumably registered via function pointer,
analogous to other unresolvable task assignments (see 13.5).

4. **Node array `0x200152F0`** (stride 0x10, 3 slots mppt/bms/venus): `+0x00`/`+0x01` =
status values (valid only in the range 2–4); `+0x02` = online flag (≠0); `+0x04` (read only for
slot 1/BMS, as 16-bit) = reference/comparison ID, compared in
`CAN_Detect_Mismatched_Nodes` against the device table `0x20014fa8` (`+0x1e` per slot). If not
satisfied, the complete 0x10-byte slot is zeroed (`CAN_Select_Master_Node`). The writer
of the online/status bytes itself is still not located.

5. **Origin/trigger of the update cycle:** No explicit "start command" found. The FSM
(`CAN_Update_StateMachine`) runs unconditionally every tick via `App_MainLoopDispatcher` and itself first calls
`CAN_Select_Master_Node()` — which checks all 3 node slots on every tick
and automatically takes over the one with a valid online flag + status values 2–4 as master
(`0x20000282` = slot index). The cycle thus "starts" data-driven as soon as a peer reports
itself online, not through a separate command. Termination/reset happens via
`CAN_Node_Data_Reset` (sets index to `0xFF`) from state 4/5.

6. **`USART_SendData`/`CAN_SendMessage`** (0x08002000) fully decompiled (14 bytes
total size): The function consists of exactly one call `CAN_SetupTxMailbox(DAT_08002010,
param_1)` — no conditional branch, no USART register accesses, no room for a second
code path. Clearly a historical misnaming (in the current
Ghidra state the function already carries the name `CAN_SendMessage`), **not** a deliberate
dual-transport architecture.

### 13.1b CAN message ID structure (addendum 2026-07-10)

Static addendum to the open item "Fully document CAN bus message IDs and arbitration"
from `AES_Crypto_Stack_Analysis.md` (local, not in the repo). All internally evaluated messages are
**extended frames** (29-bit ID) — the 11-bit standard ID is not filtered/used anywhere.

**Bit layout of the extended ID** (struct offset +0x04 in the CAN_Message):

| Bit range | Field name | Meaning |
|---|---|---|
| [7:0] | `RegKat` | Register category byte of the internal register protocol (0x01, 0x03, 0x06, 0x07, 0x10–0x14, 0x40–0x43, 0x54, 0xC1, 0xC2, 0xC3, 0xCB, 0xCE, 0xFE, 0xFF) |
| [11:8] | `Node` | Node/channel index |
| [15:12] | `Sub` | Sub-field (for CAN update: 0=EraseInfo, 1=Data, 2=Status) |
| [19:16] | — | only for class 3 + RegKat 0xCE: table index 1–8 for `CAN_ParallelInverterDataParser` |
| [23:20] | `Klasse` | 2=Modbus response, 3=parallel inverter/update data, 4=telemetry register, 0xA (combined with bits[15:8]==0xAA) = 0xAA inter-device protocol |
| [28:24] | — | Additional discriminator (table column selection in `CAN_ParallelInverterDataParser`) |

**RX dispatch** (`CAN_FrameDispatcher`, 0x0802e698): class 4 + sub 0 → `Telemetry_Register_Dispatcher` (0x0802f1c0); class 2 + sub 0 → `Modbus_ResponseDispatch` (0x0802ea54); otherwise → `Protocol_AA_CommandDispatch` (0x0802efac).

| Class | RegKat | Handler |
|---|---|---|
| 4 | 0x10–0x14 | `Protocol_AA_SetChannelData`/`_SetSystemParams`/`_SetDeviceParams`/`_EnqueueCommand`/`_SetExtendedParams` |
| 4 | 0x40–0x43 | `Telemetry_Store_EnergyCounters` |
| 4 | 0xC1/0xCB/0xCE/0x54 | `Telemetry_Store_RegC1`/`_RegCB`/`_RegCE_ByChannel`/`_Reg54` |
| 4 | 0xC3 | `Protocol_AA_RS485Forward` |
| 2 | 0x03/0xCB/0xCE/0xFE/0xFF | `Modbus_StoreRegisterSlot`/`_StoreWithHandshake`/`_StoreDualSlot`/`_StoreValue16`/`_StorePairSlot` |
| 3 | 0xCE, idx 1–8 | `CAN_ParallelInverterDataParser` → table 0x20014e3c |
| 3 | 0x06, sub 3 | `CAN_ParallelInverterDataParser` → flag `*0x20000f10=1` |
| 0xA (0xAA protocol) | Byte0 < 9 | `Protocol_AA_CommandDispatch` → RAM table 0x2000018c (17×8 bytes) |

**TX assembly:** `Register_PackDescriptor(RegKat, Node, Sub, x)` (0x08032348, 43 callers) builds `Bits[7:0]|Node<<8|Sub<<12`; used among others by `Register_WriteCategory0xCE` (cat 0xCE, sub 0/1/2 from `CAN_Update_WriteEraseInfo/Data/Status`).

**Additional finding:** `CAN_SendWorkModeFrame` (0x0802c5b0) and `CAN_SyncChangedRegisters` (0x0802c5d8) internally call only `I2C_BitBang_WriteBytes` despite their CAN names, no bxCAN access — further evidence for "name ≠ actual transport", analogous to §13.6.

**Remains blocked (cannot be resolved statically, only live):**
1. **0xAA command table** (RAM 0x2000018c, 17×8 bytes): the Ghidra project maps only the flash range, RAM is not mapped, no write xref found in the image — command IDs/handler pointers can only be clarified via a live RAM dump or CAN sniffing.

   > **CORRECTION 2026-08-16:** Functionally solved. The 8 per-pack CAN store handlers (0x0802f764–0x0802f8b8) were found + named and the registers mapped byte-exactly (verified against BMS `CAN_TX_PerPack_10Msgs`). Group↔offset↔register are known. Only the *raw* handler pointers appear nowhere in flash as a literal (the registrar lies in the undistributed sector 7 / 0x0805F000+) → for the pure RAM slot order a live dump is still needed, but functionally irrelevant. See `Modbus_RS485_TCP/VNSD_Descriptor_Open_Registers_Ghidra_Finding.md` §13 + `Methodik_und_Meta/Docs_Audit_Open_Items_2026-08-16.md`.
2. **Composition of bits[23:20] ("class") on the TX side:** `Register_PackDescriptor` does not set this field; the consumer of the TX queue (RAM 0x200000ec) was not found (0 xrefs).
3. **Arbitration priority on bus conflict:** purely a bxCAN hardware property (lowest ID wins), no software priority logic in the code — verifiable only by bus measurement.

### 13.2 Network protocol dispatcher (item #5)

**Function:** `Network_ProtocolDispatcher` (0x08003d40, 132 bytes, 4 callees)

**Central result — verified dead code:** The "0 callers" observation from the
coarse analysis was confirmed and deepened. The complete flash image (0x08000000–0x0805dfff)
was searched byte by byte for the target address — as a little-endian pointer (`40 3D 00 08`),
as a pointer with the Thumb bit set (`41 3D 00 08`) and as a MOVW/MOVT immediate pair
(`0x3d40`/`0x3d41`): **0 hits in all three cases.** No fallthrough from the
previous function (`CH395_HardwareReset` ends with a real `pop {r4,pc}` return). The
function also has no parameter, which also speaks against a FreeRTOS task entry point.
The 3 dispatched handlers each also have exactly one caller: the
dispatcher itself. **The entire chain is statically unreachable in firmware build v149.2.**

**Dispatcher logic:** Checks CH395 link status (`0x2001adb0+6/+7`) and a model/work mode
byte (`0x20014cfc+0x68`, used system-wide e.g. also by `MQTT_JSON_RPC_Dispatcher` and
`WorkMode_State_Machine`), excludes certain device models, then dispatches
by mode byte (`0x2001a82c+1`):

| Mode | Handler | Reconstructed purpose |
|---|---|---|
| 0 | `FUN_08017dc0` | Configure CH395 UDP socket (source port 22222, destination port 12345 or model-dependent) including broadcast address calculation — local discovery/query for an energy meter |
| 1 | `Network_HeartbeatHandler` (0x08026114) | Open socket; success → mode 2, failure → error counter +10 |
| 2 | `Ethernet_SendStatusTelemetry` (0x0802c438) | Sends **JSON-RPC request** `{"id":%d,"method":"EM.GetStatus"/"EM1.GetStatus","params":{"id":0}}` to a local device — **not** a cloud telemetry report despite its name |

Error counter at `0x2001a82c+2`; if it exceeds 60 (0x3c), the mode is reset to 0
(cycle restart). The meaning of the CH395 link gate byte (`+6/+7`) and the location
that writes it were not found.

**Noteworthy:** A structurally identical sister function `Modem_SendStatusTelemetry`
(0x0802c29c) for the Quectel/AT command path also exists and **also has 0
callers** — the entire "EM status poll" feature (Ethernet *and* modem variant) is completely unused/unreachable
in v149.2.

**Security assessment:** The CH395 receive buffer is protected against overflow
(1023-byte limit check before `CH395_ReadRecvBuf`). `AT_Response_Parser` masks the copied
length with `& 0xff` (fits in a 256-byte stack buffer) — appears more accidental than a deliberate
bounds check, should be examined in more depth at some point. Since the entire path is dead
code, there is currently **no active attack surface**.

**Name suggestions (not applied):**

| Old | Suggestion | Rationale |
|---|---|---|
| `FUN_08017dc0` | `CH395_UDP_MeterQuerySocket_Setup` | UDP socket setup for local EM query |
| `Ethernet_SendStatusTelemetry` | `Ethernet_EM_PollStatus` | Sends JSON-RPC `EM.GetStatus`, no cloud telemetry |
| `Modem_SendStatusTelemetry` | `Modem_EM_PollStatus` | Analogous function, AT path, also 0 callers |
| `Network_ProtocolDispatcher` | Comment `[DEAD CODE — 0 callers verified, status v149.2]` | Name remains factually correct |

**Open questions:** Actual call mechanism in older/future FW variants with
CH395 population; meaning of the link gate byte; whether `0x20014cfc+0x68` is exactly the
device model code; robustness of `AT_Response_Parser` (0x0802e35c) against manipulated
responses.

**Addendum (deep dive):**

**1) Link gate byte (`0x2001adb0+6/+7`):** Via cross-reference analysis (`find-cross-
references`) identified as a sub-range of a CH395 network status block at RAM
`0x2001adb0`. Field `+1` is a connection type/IP-valid flag, written among others by
`CH395_PHY_StatusHandler` (0x08032650) — on PHY event "disconnected"
(`param_1==1`), `+0`=1 (status) and `+1`=0 are set (`s_PHY_DISCONN` log). Field
`+5..+8` is a 4-byte value that is formatted in `Cloud_Report_FillDeviceInfo` (0x0801c398) and
`Cloud_Reporting_setVenusDReporting` (0x08016764) as `"%02d-%02d-%02d-%02d"` (MAC/serial number or IP-like value) and that is
set completely to 0 in `CH395_Reset_And_Reinit`
(0x08029964) on every CH395 reset with `memset(base+5, 0x14)`
(together with `+1`=0). The "link gate" check in `Network_ProtocolDispatcher`
(`+6 != 0 || +7 != 0`) is thus described more precisely as: **"are 2 of the 4 IP/serial number bytes
non-zero"** — a heuristic check whether a valid
address/identifier has already been assigned to the CH395 module, not a pure PHY link-up flag. The exact
writing location for `+5..+8` (i.e. who actually fills the IP/identifier, apart from the
reset clearing) was still not found — presumably part of the CH395 DHCP/
socket setup chain, which was not followed completely here.

**2) `0x20014cfc+0x68` = device model code — verified:** `find-cross-references` on
RAM `0x20014d64` (= `0x20014cfc+0x68`) shows as one of the readers the function
`Config_Get_DeviceModelCode` (0x08006990), which queries **exactly this address** three times:
```
uVar1 = 0x3f2;
if (*(char *)(DAT_080069d8 + 0x6d) == -0x56) { uVar1 = *(undefined2 *)(DAT_080069d8 + 0x6e); }
else if (*(char *)(DAT_080069d8 + 0x68) == '\x01') { uVar1 = 0x3f2; }
else if (*(char *)(DAT_080069d8 + 0x68) == '\x05') { uVar1 = 0x8ae; }
else if (*(char *)(DAT_080069d8 + 0x68) == '\x06') { uVar1 = 0x8af; }
```
(`DAT_080069d8` verified via `read-memory` == `0x20014cfc`). This confirms: byte
`+0x68` is the model selection byte from which `Config_Get_DeviceModelCode` derives a
16-bit model code (0x3f2/0x8ae/0x8af, possibly overridden by an override byte at
`+0x6d`==0xAA) — the designation as "device model code byte" in the
coarse analysis is correct, more precisely: it is the **raw index** from which the actual
16-bit model code only arises via lookup.

**3) `AT_Response_Parser` (0x0802e35c) robustness — finding with caveat:** Full
decompilation (159 lines) analyzed. The function uses a 256-byte stack buffer
(`auStack_1b0`) and a 32-element array (`local_b0`, 128 bytes) for up to 32
token pointers. Critical spot:
```
uVar7 = (iVar6 - iVar4) + 3U & 0xff;               // total length, masked to 0-255
...
local_2c = iVar6 - iVar4 & 0xff;                   // offset of the '|' separator, likewise 0-255
strncpy(auStack_1b0, iVar6, (uVar7 - local_2c) + -3);
```
Both values are individually limited to 0–255 (`& 0xff`), **but the difference `uVar7 -
local_2c` is not checked for underflow before masking.** If an input (controllable in the
AT response from the modem/network operator) produces `local_2c > uVar7 - 3`,
the `uint` subtraction underflows and yields a value close to `0xFFFFFFFF`, which is
passed directly as the copy length to `strncpy` — a potential
**stack buffer overflow through integer underflow**, not prevented by the presumed
accidental safeguard via `& 0xff`. **Practical attack surface is currently not given, however:**
`AT_Response_Parser` is called exclusively via `Modem_Response_Dispatch`
(0x08026b40), whose only two callers are in turn `Modem_SendStatusTelemetry`
(0x0802c29c) and `Ethernet_SendStatusTelemetry` (0x0802c438) — exactly the EM status poll functions from this section already identified as
**0-caller dead code**.
The potential overflow is thus statically unreachable in firmware v149.2, but should be
fixed before release if this code path is reactivated (e.g. in variants with active modem status polling).

### 13.3 UART Packet Parser (item #6)

**Functions:** `UART_Packet_Receive_Parse` (0x0800468c), `Serial_Packet_Validate`
(0x08004844), additionally analyzed: `Serial_Command_Dispatch` (0x08025eb2),
`FUN_08013f24` (XOR checksum helper), `BLE_SendFramedNotification` (0x08025f94),
`BLE_Cmd_SystemReboot` (0x0802eacc), `BLE_Cmd_OTA_Init` (0x0802e86c),
`BLE_Cmd_OTA_WriteSetup` (0x0802e900), `BLE_OTA_WriteDataChunk` (0x0802ed04),
`BLE_Cmd_OTA_Validate` (0x0802e78c).

**Correction to the rough analysis:** `UART_Packet_Receive_Parse` is **not a UART driver**. The
function works exclusively via `CH395_SPI_WriteCmd`, `CH395_GetRecvLen`,
`CH395_ReadRecvBuf` — driver functions of the WCH CH395 SPI Ethernet controller. It is
a network receive parser over SPI, not a physical UART parser. The
actual "Serial" protocol with the XOR checksum is the chain
`Serial_Command_Dispatch` → `Serial_Packet_Validate` → `BLE_Cmd_*`.

**A) CH395 receive buffer parser (`UART_Packet_Receive_Parse`)**

Compacting linear buffer (RAM `0x2001af8c`, 0x802 bytes, not a classic ring buffer):
new data is appended at `length`, overflow (`0x800-length < new_bytes`) discards
the entire buffer + CH395 SPI reset. After frame extraction, the remaining bytes are moved
to the front via memcpy. Idle counter in the last byte (0x64 = "dead", connection
treated as terminated).

| Type byte | Meaning | Effect |
|---|---|---|
| `0x18` (twice) | Abort/cancel | Signal -1 to caller, buffer cleared |
| `0x01` | Small packet, 128B payload | Frame extracted, redundancy check (inverted byte pair); **no statically findable downstream handler** |
| `0x02` | Large packet, 1024B payload | like type 1, fits the pattern config chunk vs. firmware chunk |
| `0x04` | No-op/acknowledgment | Buffer cleared, return 0 |
| `0x41`/`0x61` | Mode switch/trigger | Buffer cleared, return 1 |

Redundancy check for type 1/2: only `hdr_a == (hdr_b ^ 0xFF)` (two header bytes), **no**
XOR sum over the entire payload data despite the contrary initial assessment.

**B) Serial command protocol (`Serial_Packet_Validate` / `Serial_Command_Dispatch`)**

Frame: `['s'][len:2 BE][cmd:1][0x10][payload...][xor_checksum:1]`, max. 138 bytes
total length. Checksum = XOR over all bytes excluding the checksum byte itself
(`FUN_08013f24`, generic XOR helper, also used by `AT_Response_Parser` among others).

| Cmd | Meaning | Handler | Effect |
|---|---|---|---|
| `0x23` | System reboot | `BLE_Cmd_SystemReboot` | `Inverter_BeginShutdown(1)`, stop timers, set reboot flag — **triggers a device-wide reboot** |
| `0x3a` | OTA init/slot selection | `BLE_Cmd_OTA_Init` | Init OTA status structure, take over target offset/model |
| `0x50` | OTA write setup | `BLE_Cmd_OTA_WriteSetup` | Take over target address/size, prepare flash |
| `0x51` | OTA data block | `BLE_OTA_WriteDataChunk` | **Write 128 bytes of payload directly into external SPI flash via `Flash_Write_Protected`** |
| `0x52` | OTA validate/commit | `BLE_Cmd_OTA_Validate` | Check model ID/length/CRC, mark image as valid |

All handlers respond via `BLE_SendFramedNotification` → GATT notify, which points to a
**BLE connection** as the transport path. On a validation error: the packet is silently
discarded, no logging, no rate limiting.

**Callers/physical layer:** For both functions no statically resolvable caller
was found (`callerCount: 0`) — presumably a function pointer registered at runtime
(CH395 socket callback or BLE GATT write callback respectively). `Comm_Status_Flags_Reset`, which
`Serial_Command_Dispatch` calls on every valid packet, is also used by
`CAN_Update_StateMachine` (13.1) — the Serial/BLE protocol and the CAN update share
the same OTA watchdog/status mechanism. Relationship to the separate function
`BLE_Recv_Cmd_Dispatcher` (0x08007f58, 6732 bytes, also 0 callers) unclear — possibly a
parallel/older protocol generation.

**Security assessment:** No authentication in either path. The XOR checksum protects
only against transmission errors, not against deliberate manipulation. **Reboot (`0x23`) and
the complete firmware flash sequence (`0x3a`→`0x50`→`0x51`→`0x52`) are protected solely by
magic byte + XOR checksum + CRC integrity check — no
authenticity check.** If the underlying BLE GATT channel is reachable without pairing/bonding,
this is a critical attack vector (unauthenticated reboot and
firmware flash). The exact protection level depends on the BLE security configuration
(not analyzed here).

**Open questions:** Actual call mechanism/callback registration; downstream processing
of the type 1/2 packets in `UART_Packet_Receive_Parse`; relationship to `BLE_Recv_Cmd_Dispatcher`;
UART bridge vs. directly integrated BLE stack; BLE pairing/bonding configuration; bounds
checks of the target flash address in `BLE_Cmd_OTA_WriteSetup`.

**Addendum (deep dive):**

**CRITICAL SECURITY FINDING — no bounds check of the target flash address in the entire
BLE OTA write path.** Full chain analysis (`get-decompilation`, complete functions)
of `BLE_Cmd_OTA_WriteSetup` (0x0802e900, 68 bytes) → `OTA_Flash_Prepare_ByTarget`
(0x0802f730) → `BLE_OTA_WriteDataChunk` (0x0802ed04, 240 bytes) → `Flash_Write_Protected`
(0x0802b774) → `SPI_Flash_QuadPageProgram`:

- All three OTA handlers (`BLE_Cmd_OTA_Init` 0x0802e86c, `BLE_Cmd_OTA_WriteSetup`
  0x0802e900, `BLE_OTA_WriteDataChunk` 0x0802ed04) reference the **same** global
  OTA status block, verified via `read-memory`: all four `DAT_` literals (0x0802e8fc,
  0x0802e948, 0x0802edf8, 0x0802f7ac) identically contain `63 9D 01 20` = RAM address
  `0x20019d63`.
- `BLE_Cmd_OTA_WriteSetup`: `*(struct+0xc) = *(param_1+5)` (target offset) and
  `*(struct+8) = *(param_1+9)` (size) are taken **unchecked** from the BLE packet —
  no range check whatsoever against slot size or valid address range.
- `OTA_Flash_Prepare_ByTarget` only sets the slot **base** offset depending on the model byte
  (`struct+1`): EMS→`0x80000`, MPPT→`0x100000`, BMS→`0x180000`, VNS→`0x200000`, and calls
  `Flash_ReadWrite_Transaction(base, 0x7d000)` (presumably erase) — **only once per
  session** (`if (struct+0x14 == 0)`).
- `BLE_OTA_WriteDataChunk`, core lines:
  ```
  *(struct+0x14) = *(param_1+5);                         // chunk offset, attacker-controlled
  if ((*(struct+0x18) == *(struct+0x14)) || (*(struct+0x18) == 0)) {
      ...
      Flash_Write_Protected(param_1+9, *(struct+0x10) + *(struct+0x14), 0x80);
      *(struct+0x18) = *(struct+0x18) + 0x80;
  }
  ```
  The only "check" is a soft sequence reconciliation against the internal running counter
  `struct+0x18` — **no bounds check against the slot size `0x7d000` or a valid
  address range.** The OR condition `struct+0x18 == 0` completely defeats even this sequence check:
  `struct+0x18` is reset to 0 by `BLE_Cmd_OTA_Init` on every call with a
  new model ID/version (`memset(struct, 0x3c)` in
  `BLE_Cmd_OTA_Init`, 0x0802e86c). An attacker can therefore send `0x3a`
  (init, with a changing model ID) **any number of times**, each followed by exactly one `0x51` packet (data block) with a
  freely chosen 32-bit offset in `param_1+5`, and in doing so write 128 bytes of
  payload to `slot_base + arbitrary_offset` every time.
- `Flash_Write_Protected` (0x0802b774) and `SPI_Flash_QuadPageProgram` **likewise perform no**
  address/bounds validation — only mutex locking and page boundary splitting
  for the SPI write operation itself.

**Result:** The complete path `0x3a`→`0x51` (BLE command, see frame format in part B)
constitutes an **arbitrary-write primitive to practically any address in the external
SPI flash chip** (target = fixed model base offset + fully attacker-controlled
offset, no upper/lower bound check, no sign check). Potentially affected are
all four firmware/model slots (EMS/MPPT/BMS/VNS) as well as adjacent flash areas
outside the slot nominally prepared (erased) with `0x7d000` bytes. Since according to the
security assessment in part B there is **no authentication** on the Serial/BLE command path
and the protection consists only of magic byte + XOR checksum + a later CRC check at
commit (`0x52`), this is, with a reachable BLE GATT channel (without pairing/bonding),
a **critical, remotely exploitable flash corruption/firmware manipulation
attack vector** — regardless of whether the CRC commit check (`0x52`) at the end takes effect, since
the actual write has already happened before and can overwrite arbitrary flash contents
(including outside the OTA image area, e.g. configuration data).

On the remaining open questions: `UART_Packet_Receive_Parse` (CH395 SPI receive parser) still has,
according to `find-cross-references`, **0 statically resolvable callers** — the call
demonstrably does not occur via a direct function call findable in the flash image,
but presumably via a function pointer registered at runtime (socket Rx callback
of the CH395 driver infrastructure), analogous to the BLE handlers already described in 13.3 part B. For type 1/2 packets still **no** downstream processing code
could be identified that accesses the extracted frame contents — the function
extracts and compacts the buffer but passes no frame pointer to a recognizable
handler; most likely the return value/frame is processed further by the (unfindable) caller. `BLE_Recv_Cmd_Dispatcher` (0x08007f58) remains isolated from `UART_Packet_Receive_
Parse` and `Serial_Command_Dispatch` (no cross references found) — the two paths
share no common code except `Flash_Write_Protected` and `Comm_Status_Flags_
Reset`, which points to **two parallel, independent protocol generations** instead of a
UART→BLE bridge layer: `UART_Packet_Receive_Parse` is a pure CH395/SPI
network parser (see correction to the rough analysis above), while `Serial_Command_Dispatch`/
`BLE_Cmd_*` and `BLE_Recv_Cmd_Dispatcher` are two separate BLE-related command paths without a
recognizable mutual call relationship.

**Security assessment (supplement):** The existing assessment "reboot and firmware flash
protected solely by magic byte + checksum" is aggravated by the bounds check finding:
it is not just missing authenticity but a missing
**integrity/range check of the target address**, i.e. even an in itself "expected"
command can cause damage outside the intended OTA slot through a single manipulated field value (offset).

### 13.4 SOC charge limiting (item #7)

**Function:** `Battery_Charge_Power_Limiter` (0x08004490, 1 caller)

**Correction to the initial hypothesis:** **No SOC reference (% state of charge)** can be demonstrated in the code.
Instead the function combines (a) four raw PV string measurement values from the structure
`0x20014f40` (offsets `0xC/0x12/0x18/0x1E`, threshold 101 for the validity check) and (b) a
hardware limit `P = -(U×I)` computed from the central BMS structure `0x20014f82` (offset `0x00`=pack voltage ×0.01,
offset `0x24`=BMS charge current limit ×0.1) with a fixed
lower bound of **-2500 W** (`Power_Limit_Clamp`, 0x08012f80). Only with a negative input value
(charge direction) and if at least one PV string value is ≥101, the requested
charge power is capped to `min(param, PV_sum×0.1 + HW_limit)`; a mode flag
(`0x20014f82+0x17` bit0) switches to the pure hardware limit.

`Power_Limit_Clamp` is a generic building block that is also used by `Grid_Export_Power_Limiter`
(0x0801ec38) and `Inverter_Power_Setpoint_Calc` (0x080061fc) — not
battery-charge-specific code.

**Caller chain:** `Inverter_Power_Setpoint_Calc` → several further clamping stages (PV yield,
feed-in block, fixed limits) → the total setpoint is written via `Register_PackDescriptor(1,·,4,0)`
+ `Register_WriteValue` into the internal register system (plausibly Modbus-readable
via `Modbus_StoreRegisterSlot`, concrete register address not resolved).

**HA relevance:** External remote control paths (`Remote_Power_Command_Execute`, MQTT passive
mode) can specify a target charge power; `Battery_Charge_Power_Limiter` then acts as a
hard safety limit — no bypass/override visible in the code. For an HA display of the
"currently permitted charge power", the BMS telemetry structure `0x20014f82` (already processed
via MQTT/BLE) is the more relevant source.

**Open questions:** Exact field meaning of `0x20014f82+0x00/+0x24`; meaning of flag bit0/bit1 at
`+0x17`; concrete Modbus register address of the final setpoint; sign convention
(negative = charging or discharging?) — an existing code comment at `Inverter_Power_Setpoint_Calc`
contradicts the logic observed here and is presumably wrong.

**Recommendation:** Rename item #7 in the findings list to "PV-yield/BMS-current-limit-
based charge power limiting" (not an SOC algorithm).

### 13.5 CH395 mutex blocking (item #8)

**Functions:** `CH395_SPI_Cmd_WithData` (0x08002c38), `CH395_SPI_ReadByte` (0x08002e10),
`CH395_SPI_WriteCmd` (0x08002ea4), `CH395_Socket_SendData` (0x0802d12c), additionally
`CH395_SPI_Send_Data` (0x08003644), `CH395_SPI_CmdWaitReady` (0x08002f38),
`CT_SyncTransfer` (0x0802d484), `Delay_Ms` (0x0803e304).

**Central finding:** All CH395 driver functions secure SPI access via **a single
global FreeRTOS queue of length 1 (RAM `0x20000120`)** as a mutex substitute — verified via
identical literal pool values at the four `DAT_*` reference sites. It is not a real
`xSemaphoreCreateMutex` (no priority inheritance) but `xQueueReceive`/`xQueueSend`
with a timeout literal **5000** (≈5s at a 1kHz tick, tick rate assumed, not verified).
On timeout: silent failure (return 0), no retry.

**Busy-wait locations:**

| Function | Timeout/iterations | Blocking? |
|---|---|---|
| `CT_SyncTransfer` (per SPI byte) | 2× 4096 count loops | true busy spin, no yield |
| `CH395_Socket_SendData` tail loop | 20000 downwards, one poll every 500 | pure CPU spin |
| `CH395_SPI_CmdWaitReady` | up to 201× `Delay_Ms(5)` ≈ **~1.0s worst case**, **holds the global mutex while doing so** | cooperative towards the scheduler, but the mutex stays occupied for up to 1s |

**Cascade risk confirmed:** `Modbus_Dispatcher` (0x0801e43c) directly calls
`CH395_SPI_WriteCmd`/`CH395_SPI_Send_Data`; `RS485_UART_Send` (0x080072a0, from
`RS485_RTU_Frame_Dispatcher`, FC06/FC10 handlers, `Protocol_AA_RS485Forward`) directly calls
`CH395_Socket_SendData` — synchronously, without queue decoupling. Since all CH395 functions
share the same global lock, a network-side operation (chip init/reset,
`CH395_UDP_ServerTask`, `HTTPS_POST_Request`) that holds the lock for up to ~1s (for
`CH395_Reset_And_Reinit` possibly significantly more due to two consecutive init phases,
approximately towards the 5s timeout) can delay a concurrent Modbus/RS485 send operation for
that duration. With the ≤1000ms timeout of many Modbus masters this is plausibly sufficient
to be perceived as a master-side timeout.

**Open questions:** Tick rate not directly verified; task/priority assignment of
`Modbus_Dispatcher` vs. `CH395_UDP_ServerTask` cannot be determined (no resolvable callers,
presumably an `xTaskCreate` table); activation status of the RS485→Ethernet forwarding flag
in the default configuration unclear.

### 13.6 I2C bit-bang + HW I2C (item #9)

**Functions:** `I2C_BitBang_Delay/WriteByte/Start/Stop/ReadBit/WriteBytes`,
`I2C_Init_Configure` (0x080028ec), `CAN_SendWorkModeFrame` (0x0802c5b0),
`CAN_SyncChangedRegisters` (0x0802c5d8, newly identified).

**GPIO assignment (corrected):** The bit-bang runs over **GPIOC** (base `0x40011000`) —
**SCL=PC0, SDA=PC1** (BSRR/BRR/IDR accesses verified). The earlier note "SDA=pin2,
SCL=pin1" referred to bit masks, not to physical pin numbers.

**Hardware I2C peripheral — misnaming uncovered:** `I2C_Init_Configure` is **not an
I2C init but a bit-exact copy of `CAN_Init()` from the STM32 StdPeriph lib**
(CAN_MCR/CAN_MSR/CAN_BTR bit fields exactly demonstrated). 0 callers verified — dead
StdPeriph leftover code or an indication of a second, unused CAN interface. Recommendation:
rename to `CAN_StdPeriph_Init_UNUSED`.

**I2C target device (`CAN_SendWorkModeFrame`):** Sends 2 bytes via bit-bang: byte0=`0x48` (fixed),
byte1=`(WorkMode<<4|0x81)`. Two callers identified, **both from the inverter subsystem**:
`Inverter_Sync_Init` (periodic, 3000-tick timer) and `Inverter_Apply_BatteryParams`.
`I2C_BitBang_WriteBytes` additionally has a second, previously undocumented caller:
`CAN_SyncChangedRegisters` (compares 4 byte registers, sends 2-byte frames on deviation) —
likewise called exclusively from the inverter subsystem.

**Hypothesis target device:** `0x48` as a classic 7-bit I2C address is unusual (no known
standard chip). Together with the finding that all callers without exception originate from the
inverter sync subsystem, the more plausible interpretation is a **proprietary
point-to-point protocol to an internal coprocessor/MCU in the inverter/power section**
(frame/command IDs instead of real multi-slave I2C addressing) — not a standard I2C chip
(display/EEPROM/RTC). Remains a hypothesis.

**Open questions:** Whether a supposed 4-byte address table at `0x0805af35` is real or
a linker/string deduplication artifact (suspiciously overlaps with the debug string
"udp=%d,..."); exact meaning of the WorkMode enum; reason for the dead CAN_Init leftover code.

**Addendum (deep dive):**

7. **`0x0805af35` is not a real array but a debug string overlap** — confirmed.
Ghidra's own symbol assignment at this address is
`s_udp=%d,api=%d,net=%d,port=%d,inv_0805ac80+0x2b5`, i.e. the address lies exactly byte-precisely
(offset `+0x2B5`) within the 848-byte debug format string starting at `0x0805ac80`
("cd=%d,tot_i=%d,...soh=%d"). The literal pool value in `CAN_SyncChangedRegisters`
(`DAT_0802c628` = `0x0805AF35`) reads 4 bytes from there as "byte1" send values for the
I2C bit-bang transfer — these are exactly the ASCII bytes `'u','d','p','='`
(0x75,0x64,0x70,0x3D) of the string. Not an independent, deliberately designed data array but
a compiler/linker placement artifact (probably missing/incorrect
memory reservation for a 4-byte array originally intended in the source code, which the
linker instead let overlap with existing .rodata bytes).

8. **WorkMode address corrected:** Contrary to the earlier assumption in 13.2, the actual WorkMode byte is not at
`0x20014cfc+0x68` but at **`0x20014cfc+0x74`**
(`0x20014d70`) — written by `Config_Write_WorkingMode`/`Config_Set_WorkMode_Validated`
(value range hard-validated: `< 8`, otherwise discarded), read (unchecked) by
`CAN_SendWorkModeFrame` (`WorkMode<<4|0x81`). `0x20014cfc+0x68`, in contrast, is used by numerous
other functions including `Config_Get_DeviceModelCode` — presumably the
device model byte, not WorkMode; the earlier assignment in 13.2 was imprecise. The individual
enum values 0–7 could not be fully reconstructed in terms of meaning; only
the already documented special cases remain known: WorkMode `10` = "Remote" (13.12, reachable only via RS485,
bypasses the `<8` check) as well as a **separate** sub-flag at `+0x69`
(`5` = "Force Charge", used in `WorkMode_State_Machine`) — this `+0x69` field is **not**
identical to the main WorkMode byte `+0x74` (possible confusion in an earlier
code commenting).

9. **Dead CAN_Init leftover code = `I2C_Init_Configure` itself** (0x080028ec): 0 callers
confirmed, AND the function sits in the flash image directly before the real, actively used
CAN driver functions `CAN_ReadMailbox` (0x08002a20) and `CAN_SetupTxMailbox` (0x08002b10) —
matching the thesis that a second bxCAN peripheral instance was originally initialized here,
whose init call was removed when switching to the I2C bit-bang solution, while the
compiled function body (object file linking without function-level dead code elimination /
`--gc-sections`) remained unused in the image.

### 13.7 CAN parallel inverter (item #11)

**Functions:** `CAN_Parallel_Inverter_Sync` (0x0800557c), `CAN_Select_Master_Node`
(0x080056b4), `CAN_Detect_Mismatched_Nodes` (0x08004418), newly brought in:
`CAN_ParallelInverterDataParser` (0x0802e6e4).

**Correction to the initial hypothesis:** `CAN_Select_Master_Node` contains **no SOC comparison**.
Instead the function selects the first slot marked "online" (byte `+0x02≠0`) with
plausible status bytes (range 2–4) from the node array `0x200152F0` (3×0x10 bytes). According to the `CAN_Update_Init` name table, the
three slots correspond to **mppt/bms/venus** — presumably
different subsystems for the CAN-based firmware distribution, not three equal-ranking
parallel inverters.

A real **maximum comparison** ("highest value wins") does exist, but in
`CAN_Parallel_Inverter_Sync` (fallback path for an ambiguous node situation), operating on a
byte+6 field of a separate table (`0x20014e3c`) — whether this field is SOC could **not be
verified** (speculation).

**Version compatibility check** (`CAN_Detect_Mismatched_Nodes`): compares a
reference ID from the BMS slot against a device table (`0x20014fa8`); exactly 1 deviation =
normal case, 0 or >1 = error flag set (structure `0x2000027a`). No log entry,
only controls the control flow of `CAN_Parallel_Inverter_Sync`.

**Data source unchecked:** `CAN_ParallelInverterDataParser` (called from
`Protocol_AA_CommandDispatch`, the 0xAA inter-device protocol) takes payload data
**unchecked directly from CAN extended ID frames** (class `bits[23:20]==3`, byte0=`0xce`) into
the table `0x20014e3c` — no sender authentication, no CRC/signature check
(consistent with finding #4).

**Caller context:** `CAN_Select_Master_Node` and `CAN_Parallel_Inverter_Sync` both run
exclusively from `CAN_Update_StateMachine`, i.e. once per main loop tick
(polling, not interrupt-driven).

**Security assessment:** An attacker with CAN access can write arbitrary values into
`0x20014e3c` via suitable extended IDs and thereby deliberately influence both the fast path (raw
takeover of foreign data) and the maximum fallback — plausible as a
targeted manipulation of the update target selection. Who writes `0x200152F0` (online/status bytes per slot)
could not be located (open).

**Open questions:** Writer of `0x200152F0`/`0x20014fa8`; meaning of byte+6 in `0x20014e3c`;
trigger of `*0x20000282=0` (update start); whether "parallel operation of multiple Venus units"
or "multi-MCU update (MPPT/BMS)" is the more accurate interpretation — the names
`CAN_Select_Master_Node`/`CAN_Parallel_Inverter_Sync` may be misleading
(alternative suggestion: `CAN_Select_Update_Target_Slot`).

### 13.8 Flash string obfuscation (item #12)

**Function:** `Flash_Obfuscated_String_Decode` (0x080058cc)

**Result: initial hypothesis refuted.** There is no evidence of hidden
plaintext strings (credentials, URLs, debug commands). Findings:

- The supposed "obfuscated input" comes from `Flash_ReadWords` — a generic
  hardware flash controller routine with busy-wait polling, whose return value `0x5a5a5a5a`
  is a **success status code**, not a string byte/terminator.
- The supposed lookup table (`0x0805D915`) contains only the monotonic pattern
  `01 FF 01 FF …` over at least 256 bytes — unsuitable as a character substitution table.
- Of the 2 call contexts found, one (`sscanf_Format_Parser`, 12 call sites)
  is the generic "get next char" callback of the firmware-wide `sscanf()` implementation
  (no individual string addresses per site); the other
  (`Inverter_Register_Buffer_Init` → `MQTT_Client_Init`) fills a 37-byte
  **binary** register buffer, not text.
- The calling convention (`r0`/`r1` expected vs. actually used `r4/r5/r6/r8/r9/r10`) suggests
  that the **Ghidra function boundary at `0x080058cc` is set incorrectly**
  (presumably merged code of two different routines).

**No plaintext string could be reconstructed** — deliberately no speculation, since the modulus/
table contents cannot be verified.

**Recommendation:** Recategorize item #12 as "refuted/unclear — presumably register buffer init, no
string obfuscation", no security risk.

**Open questions:** Correct the function boundary in Ghidra and re-analyze; actual
meaning of `DAT_080010ec`/supposed sscanf callback; tracing back `r9`/`r10`
would require emulation.

**Addendum (deep dive):** Function boundary confirmed incorrect — verified by linear disassembly
(0x08005860–0x08005900, Ghidra script). The real, contiguous function
starts at **0x08005860** (`push {r3,r4,r5,r6,r7,r8,r9,r10,r11,lr}`) and ends at
**0x080058d9** (122/0x7a bytes total size), only caller `MQTT_Connect_And_Subscribe`
(0x080180de via `Inverter_Register_Buffer_Init`). Ghidra's separate function entry at
`0x080058cc` (`Flash_Obfuscated_String_Decode`) lies exactly on the instruction `cmp r8,r9`
in the middle of this function's loop condition — reached only via a `b` tail jump from
`0x080058a2` in the function start, never as a valid standalone jump target. The 12
`COMPUTED_CALL` xrefs from `sscanf_Format_Parser` are thus to be classified as a Ghidra misinterpretation:
an indirect call via the sscanf callback table (convention `r0`/`r1`)
cannot sensibly point to an address whose code works exclusively with inherited
`r4/r5/r6/r8/r9/r10` context from the caller.

Real function logic (confirmed by disassembly): parameter validation, `memset`+`memcpy`,
then a loop over `Flash_ReadWords()` (0x08000294 — decompilation verifies a generic
flash controller routine with busy-poll on status bit 0x80 via `DAT_080004ac[3]`; return value
`0x5a5a5a5a` exclusively as a success status code after a completed word-read loop). On
success, `(word_read mod 64)` is used as an index into a table referenced via the literal-pool pointer
`DAT_080058dc` (memory content: `15 D9 05 08` = `0x0805D915`);
the result byte is written to the target buffer. Memory dump of the first 64
bytes from `0x0805D915`: strictly alternating pattern `FF 01 FF 01 …` — **only 2 possible
output values** (parity-dependent 0x01/0xFF). This is structurally incapable of producing arbitrary
plaintext characters, and supports the classification already made as binary
register buffer initialization (marker/flag bytes per register slot for `MQTT_Client_Init`),
not as string decoding. Item #12 thus remains refuted — now with the corrected
function boundary and fully clarified actual semantics.

### 13.9 Debug backdoor (item #13)

**Function:** `Debug_Mode_Set` (0x08005fb8); `System_SetDebugMode`/`_Wrapper` (0x08050dd8)
are **the same function** (duplicate entry in the tracking, should be merged).

**Mechanism:** `0xC2` is not a simple comparison constant but a
**register group ID** in the internal CAN/RS485 register protocol
(`Register_PackDescriptor(0xC2,…)` → `Register_WriteValue` → FreeRTOS queue to a
downstream inverter/DSP subsystem, same mechanism as in findings #4/#11). Group
0xC2 is generated only here, no receiver handler found. In addition, the
function sets a local RAM flag `0x20000132 = 2` on activation.

`0x08050dd8` is a CLI-style wrapper (`printf("debug_mode=0/1")` + call of
`Debug_Mode_Set` with log level 4=DEBUG).

**Activation channel: statically unreachable.** `Debug_Mode_Set` has only one caller
(`0x08050dd8`); for `0x08050dd8` itself, **neither code nor data references nor
function pointer literals** were found (cross-reference and constant search negative). The
path is **dead code** in the current firmware image — subject to an untyped
jump table that Ghidra would not recognize as an xref.

**Related, actually active risk:** The same RAM/EEPROM flag (`0x20000132`/EEPROM
`0x900`) can be set independently of `Debug_Mode_Set` via **Modbus register 46000 (0xB3B0,
"OTA command") with magic value `0x5100`** (`Write_Handler`, 0x08051f7a) — via TCP
or RS485-RTU, **without authentication** (only check: register number ≥40000). This
persistently activates (EEPROM backup) extended debug log output and survives a reboot.
The value (`2`) set by `Debug_Mode_Set` additionally enables RS485 passthrough forwarding
in `Protocol_AA_RS485Forward` (0x0802f0d0) on incoming register group 0xC3 — but this path
is only relevant if `Debug_Mode_Set` does become reachable.

**Security assessment:** The 0xC2/Debug_Mode_Set path originally classified as a "backdoor"
is **inactive** in FW 149.2 and currently poses no risk. The actually exploitable
risk lies in the unprotected Modbus write access to register 46000, which abuses the same RAM cell
and is related to the OTA topic complex (finding #14/#25).

**Open questions:** Why is `0x08050dd8` unreachable (remnant of a removed CLI parser)?
Consumer of the register group 0xC2 queue; caller of `EEPROM_ClearSetting_0x900`
(0x0804d5b0, also 0 xrefs); meaning of further flag values besides 1/2.

**Addendum (deep dive):**

*Why `0x08050dd8` is unreachable:* The strings `"debug_mode=1\r\n"`/`"debug_mode=0\r\n"`
are referenced exclusively by `0x08050dd8` itself (no external dispatcher). In the
memory range `0x080555a0`–`0x08055620` there is additionally a contiguous block of
setting names with Chinese-language description texts — among others `"debug_mode"`
(0x080555ea, directly followed by `"开关逆变器调试功能"` = "inverter debug function
on/off") and `"venus_poweroff"` (0x08055611, followed by a sleep mode description).
**Both name strings have 0 references in the entire firmware image** (checked via
`find-cross-references`) — so this is not just a single removed
command, but an entire orphaned CLI/service menu parameter table (several settings
incl. `debug_mode` and `venus_poweroff`), whose consuming dispatcher was compiled out of FW
149.2 entirely; the strings remained only as bloat in the `.rodata` area.
This supports the hypothesis "remnant of a removed CLI parser" and extends it: not only the
function, but also the associated name entries are orphaned.

*Consumer of the register group 0xC2 queue:* `Register_WriteValue` → `Modem_QueueSendMessage`
→ `xQueueSend(*(struct@0x200000e8 + 4), …)` — the queue handle is located in RAM `0x200000ec`
(field+4 of a larger control structure, whose field+0 (`0x200000e8`) is used separately as the CAN-RX
mailbox handle by `CAN_RxMailbox_Handler`/`CAN_RxQueue_DrainAndDispatch` — active
and independent of the 0xC2 path). `Modbus_SendResponse` (0x0802c060) was identified as the only plausible consumer for exactly this
queue handle (`0x200000ec`): it calls
`Modbus_Response_Builder(*(0x200000e8+4), buf, 1000)` — a function that works directly against
FreeRTOS queue internals (`prvCopyDataFromQueue`, message counter at struct offset
`+0x38`), de facto its own `xQueueReceive` implementation — and on success forwards via
`CAN_SendMessage(buf)` to the CAN bus. **`Modbus_SendResponse` itself, however, has 0
incoming references** (`find-cross-references` negative in both directions; a
constant search for the Thumb address `0x0802c061` also yields no hit, no recognizable
task creation site). Thus not only the producer (`Debug_Mode_Set` via `0x08050dd8`),
but also the only consumer path for group 0xC2 findable in the image is statically
unreachable — the entire producer→queue→consumer chain is inactive in FW 149.2.

*Caller of `EEPROM_ClearSetting_0x900`:* Checked again specifically — `find-cross-references`
(both directions, incl. data references) still yields **0 hits**; additionally
`find-constant-uses` for both the plain address `0x0804d5b0` and the Thumb-bit variant
`0x0804d5b1` (possible function pointer table entries) also yields **0 hits**.
No function pointer table, no indirect call found. The function is thus a
third confirmed example of dead code besides `0x08050dd8` and `Modbus_SendResponse` — fits
into the overall picture of a partially disabled/removed debug/service subsystem in FW 149.2.

*Further flag values:* Complete writers/readers of RAM flag `0x20000132` determined.
Writers: `Debug_Mode_Set` (0 on function entry / 2 on activation), `EEPROM_
ClearSetting_0x900` (0), `Write_Handler` 0x5100 path at reg. 46000 (writes **1**, together
with `EEPROM_Write(0x900,…)` — this is the already documented persistent activation path).
Readers: `Protocol_AA_RS485Forward` (only comparison site, checks exclusively `== 2`),
`log_printf` (only as a log output argument, no comparison). Thus **0/1/2 are the only values
actually produced or compared** — no `switch`/no further comparison chain
found for this flag. For delimitation: The values `0x4d2`/`0x929`/`0xd80`/`0x11d7` for register 46000 documented in the `Write_Handler` code comment set a **different** flag
(`*DAT_08052000`, values 1–4, presumably OTA/reset mode selection) and must not be confused with the
0x20000132 debug mode flag. Side finding: `CH395_Init_TCPServer_Socket`
references the same base address `0x20000132` only coincidentally as the base pointer of a
socket descriptor structure (offset `+0x40` ff., TCP port `0x1f9b`=8091) — address coincidence,
since the debug mode flag sits at the start of a larger global control block, no
substantive connection.

### 13.10 OTA flash pipeline & dispatcher (#14, #25, #43, #52, #93)

End-to-end chain: `App_MainLoopDispatcher` → `OTA_Update_Dispatcher` (0x080151c8, in Ghidra
still `FUN_080151c8` — rename still to be done) → `OTA_Process_Pending_Updates` →
`ProcessFirmwareUpdateCommand` → `Flash_EraseRegion`/`Flash_WriteRegion` (internal STM32 flash,
only a 40-byte **slot descriptor**, not the firmware image itself) or
`OTA_Firmware_Download_Init` → `Flash_ReadWrite_Transaction` (external QSPI flash). Final
check in `OTA_FW_Verify_And_Apply`: CRC16 + model string `"VNSD-0"` + `dev_mask` comparison —
**no cryptographic signature check at any of the four examined levels.**

**#43 4-slot layout confirmed:** EMS=0x80000, MPPT=0x100000, BMS=0x180000, VNS=0x200000, each
512 KB, via `OTA_Flash_Prepare_ByTarget`.

**#52 llhttp:** Is **not** used for OTA downloads — `llhttp_execute` has 0 callers,
completely dead library. The active HTTP client (`HTTPS_POST_Request`) serves exclusively
cloud telemetry POSTs; a prepared OTA branch in its response reader is not activated by any
caller (`param_5=0`). The actual network OTA trigger instead runs via the Quectel AT command
`AT+QWLANOTA` (see §13.11).

**Security:** Anyone with BLE access can use the BLE OTA chain (§13.3) to write an arbitrary image
with correct CRC16 + `"VNSD-0"` + `dev_mask` into one of the four 512 KB slots — both
values are static/easily computable. No signature protection at any level.

### 13.11 BLE OTA, WiFi OTA & WiFi provisioning (#18, #41, #95)

**#18 WiFi provisioning:** Delimiter corrected: `<.,.>` (not `<,>`). SSID/PW are applied via
`WiFi_Set_Credentials` → `Quectel_WiFi_SetAPConfig` (`AT+QSTAAPINFODEF`) and
additionally mirrored **in plaintext** to EEPROM 0x400/0x420 (I2C writer transforms nothing).
No auth gate on BLE command 5.

**#41 BLE OTA slot selection:** No real slot arbitration — a single global
0x3C-byte session struct, the target address is chosen purely **client-side** via a 1-byte model field
(EMS/MPPT/BMS/VNS), no server-side plausibility check. **No rollback:** Every
128-byte chunk is written to flash unconditionally **before** any check; if the
CRC/model check fails, the possibly corrupt region simply stays, no backup/A-B.

**#95 ⚠️ most critical single finding of this group:** `BLE_WiFi_OTA_WithURL` (0x0800b90c) accepts an
**arbitrary URL specified by the BLE client** and passes it 1:1 as
`AT+QWLANOTA=<url>[,<port>]` on to the Quectel modem — no whitelist, no protocol enforcement,
the only check is a length limit (≤255 characters). In addition there is a
cloud-independent fixed mode (`AT+QWLANOTA=http://192.168.137.1/FC41D_OTA.rbl`, classic
hotspot gateway IP). The actual download/flash process runs entirely inside the Quectel module,
the Control MCU has no visibility of the content whatsoever. After every trigger, a
`System_Reboot()` follows unconditionally — DoS side effect even for an ineffective attack.

### 13.12 RS485 register map & remote power control (#15, #28, #46)

All `Inverter_Set_*`/`Inverter_RS485_Cmd_*` functions run via
`Register_PackDescriptor→Register_WriteValue→Queue` and send **synchronously and immediately**, independent
of the "24 dirty bit" system. The dirty bit array (`Timeslot_Bitmap_SetClear`) is only read within
its own setter function — **no RS485 transmission actually depends on it**;
the initial hypothesis on the dirty bit transmission mechanism is thus refuted.

**No auth mechanism** in `Write_Handler` (Modbus TCP as well as RS485-RTU) — the only barrier is
the address range `≥40000`. The "register 40000 unlock" (0x55AA) does **not** gate on/off, power setpoint
and WorkMode — these run independently of it. Reg 45012
(`Inverter_Power_Setpoint_Apply`) additionally bypasses the safety clamps that the regular path
(`Inverter_Power_Setpoint_Calc`) applies.

**#28 Remote power control:** `Remote_Power_Command_Execute` (mode 0=zero/1=discharge/2=charge/
3=SOC threshold) fires only at WorkMode==10 ("Remote"). This mode is **not reachable** via the regular
Modbus WorkMode registers (hard-limited to <6/<8 there) — it is set primarily via
**MQTT/cloud JSON** (`work_mode` field **without value range check**) or BLE. Protection
depends entirely on the TLS/broker trust model, not on in-firmware authorization.

**#46 WorkMode mode 1** ("CH395Reset" according to the old name) is **not a hardware reset** — only a
connection flag reset + `TimePlan_Evaluate_Setpoint` trigger (name correction).

### 13.13 TLV parser & 0xAA inter-device protocol (#16, #42)

**#16 Name correction:** `TLV_Record_Parse` is in truth a **DNS resource record parser**
(RFC 1035, already renamed to `DNS_ResourceRecord_Parse` on 2026-07-09); `TLV_Record_Skip`
should consistently become `DNS_ResourceRecord_Skip`. No relation to OTA metadata. Bug found:
RR types SOA/HINFO/MX/TXT do not skip RDATA — desync risk with multiple answer records
(DNS responses are spoofable, since UDP/port 53 without DNSSEC).

**#42 0xAA protocol:** Transport is **CAN** (extended ID, magic in ID bits[15:8]=0xAA), not
RS485. The 17-entry handler table resides in RAM at runtime (`0x2000018c`) — **no
initialization code found**, content remains unknown (open question). Important naming finding:
The 6 "known handlers" (`Protocol_AA_SetChannelData` etc.) are **not** attached to the 0xAA gate,
but to a separate `Telemetry_Register_Dispatcher` (CAN ID class 4, 11 registers,
completely unauthenticated). Most dangerous function: `Protocol_AA_RS485Forward` (reg 0xC3)
bridges an unchecked 8-byte CAN payload directly onto the RS485 bus **or** a TCP/UDP socket —
classic trust boundary violation between the CAN and RS485/network domains.

**Addendum (deep dive):** The only access to the table in the entire image is
`Protocol_AA_CommandDispatch` (0x0802efac) itself — it reads, via the flash constant pointer
`DAT_0802f00c` (value `0x2000018c`), 17 entries of 8 bytes each (2-byte ID + 4-byte function pointer,
+2 pad) and calls the handler on an ID match. Three independent search methods found **no**
further reference to this value: `find-cross-references` on `0x2000018c` (0 hits),
full-text search across all decompilations for `2000018c` (0 hits) and `find-constant-uses` for
the immediate `0x2000018c` (0 hits, also searches all instruction immediates beyond
symbolically resolved pointers). The RAM address additionally lies outside the only memory block
mapped by Ghidra (`ram`, `0x08000000`–`0x0805dfff`, flash only) — a direct `get-data`
on `0x2000018c` is therefore not possible, and the hypothesis "preset .data init value" cannot
be verified for lack of a reachable flash source offset (no generic data segment copy loop
found in the statically analyzed code). Conclusion: The table is either written by
code outside the examined app image (e.g. bootloader or another
firmware part), or the initialization happens via a mechanism that triggers none of the three
search approaches (e.g. purely register-relative addressing without a literal pool constant). The
table content thus remains **statically indeterminable** — confirms the previous finding,
but now rules out initialization by known app code with high confidence.

### 13.14 Config/EEPROM layout & factory reset (#17, #20)

Three factory reset modes (1=full incl. cloud certificates, 2=config reset, 3=cert wipe only),
reachable via **four** channels without additional auth: Modbus TCP, RS485-RTU, MQTT JSON-RPC
(method 0x11) and BLE (cmd 6) — **no physical button path demonstrable in the app image**.
Mode 1 deletes EEPROM validity, statistics, **and cloud TLS certificates** (CA/user cert/key via
`Quectel_SSL_Certificate_Manage`, see §13.16).

Important naming finding: 6 functions (`Config_Read_U8/U16/U32/String/Block`) are named "Read", but
actually call `EEPROM_Write` — misnaming, should become `Config_Set_*`.
EEPROM access layer (`EEPROM_Write`/`EEPROM_Read`) has **no CRC, no wear leveling** —
only change detection for cycle reduction.

### 13.15 BMS cell data via BLE + internal (#19, #89)

`BLE_Build_BMS_Data_Response` (cmd 0x14, no auth gate) and `BMS_CellVoltage_MinMax_Finder`
(#89, still `FUN_08013060` in Ghidra — rename not performed) read the same
source (`0x20014f82`/`0x20014fa8`). Confirmed: **6 channels × 16 cells** (plausibly 6
packs/modules of 16S each). Retrievable via BLE without authorization: pack voltage/current/temperature/SOC,
charge/discharge limits, fault flags (partially) and 16 individual cell voltages of pack 0 — privacy-
relevant (allows conclusions about cell aging). No balancing logic found in the Control FW, only
reporting.

**Addendum (deep dive):** Four targeted follow-up questions on struct field meaning clarified or
confirmed as unresolvable:

1. **Field `0x20014f82+0x00`:** **Pack/system voltage, unit 10 mV (unsigned short)** —
   evidenced by the (unwired, but layout-identical) debug printer
   `CAN_Battery_Telemetry_Debug_Print` (`0x080358ec`), whose first output labels exactly this field
   with the format string `"pgn_1801_info.bat_volt(10mv)=%d"` (verified via
   `find-cross-references` on `0x080358f0`→`DAT_20014f82`, immediately followed by the string access
   at `0x080359d0`). Consistent with `Power_Limit_Clamp` (`0x08012f80`), which scales `*DAT_08012fe0`
   (=`0x20014f82`) by `0.01` (10 mV raw value → volts) and multiplies it with the field at `+0x12`
   (`chrg_curr`, 100 mA unit, same debug string table) into a power clamp limit
   — i.e. this field is the central voltage quantity referenced in practically
   the entire charge/discharge control chain (`Power_Limit_Clamp`, `WorkMode_State_Machine`,
   `Grid_Power_Dynamic_Adjust`, `Inverter_Power_Setpoint_Calc`, `Inverter_Apply_BatteryParams`).
   Field `0x20014f82+0x24` (`0x20014fa6`), by contrast, has **not a single
   static reader/writer** (empty `find-cross-references` result) — it lies in a
   2-byte gap between the last widely used field `+0x22`/`+0x23` (referenced among others by
   `MQTT_Telemetry_Struct_Builder`, `Cloud_Report_URL_Builder`, `BLE_RuntimeInfo_Builder`,
   `BLE_Recv_Cmd_Dispatcher`) and the start of the 6-channel cell structure at `0x20014fa8`.
   Most likely explanation: padding/reserve byte, alternatively a field addressed only dynamically (not
   constant-propagatable) — remains open.

2. **Flag bit0/bit1:** The tested byte sits at `0x20014f82+0x16` (`0x20014f98`) and, according to
   the same debug string table, bears the name **`chrg_flag`** (`"pgn_1803_info_chrg_flag=%d"`,
   one byte further follows a separate `force_chrg` flag at `+0x17`). `WorkMode_State_Machine`
   (`0x0802c784`) tests both bits to override the normal SOC hysteresis for charge start/stop
   (start < 51 %, stop > 50 %, see existing documentation):
   `bVar6 < 0x33 || (chrg_flag>>1 & 1)` (bit1 set ⇒ charge start is forced even above 51 % SOC)
   AND `0x32 < bVar6 || (chrg_flag & 1)` (bit0 set ⇒ charge mode stays active even below 50 % SOC).
   The field is additionally read by `Battery_Forced_Charge_Check`
   (`0x08005344`), which supports the interpretation as a BMS-side "forced charging" request flag.
   A writer was not located (plausible: CAN RX receive path from the BMS,
   analogous to item 3).

3. **Writers of `0x200152F0` and `0x20014fa8`:** Despite an exhaustive search (all known
   flash literal pool entries pointing to these RAM addresses searched via
   `find-cross-references`/`find-constant-uses`), **only readers** were
   found — no writer locatable in the firmware image; the "open question" from §13.1 is
   thereby confirmed as **actually unresolvable** (not a mere search gap). Also confirmed:
   **two different memory regions**, no address alias. `0x200152F0` is the
   3×0x10-byte node array (readers: `CAN_Select_Master_Node` `0x080056b4`, `CAN_Update_Init`
   `0x08027224`) for the firmware update peer status per subsystem slot (mppt/bms/venus, see
   §13.1). `0x20014fa8` is the separate 6-channel BMS cell structure (6×0x60 bytes; readers:
   `BMS_CellVoltage_MinMax_Finder`/`FUN_08013060`, `BLE_Build_BMS_Data_Response`,
   `CAN_Detect_Mismatched_Nodes` `0x08004418`, which detects channel mismatch via field `+0x1e` per channel).
   Both follow directly after one another in RAM (node array ends at `0x20015320`, where according to
   `CAN_Update_Init` the update state structure begins) or directly follow the
   pack sum structure `0x20014f82` (`0x20014fa8` = `0x20014f82+0x26`) — both point to
   the same CAN RX receive path as the most likely (but unverified) writer,
   presumably via a PGN dispatcher mechanism that is not captured by Ghidra's static
   constant propagation (similar to the indirect `Register_PackDescriptor`
   pattern from §13.18).

4. **Byte+6 in `0x20014e3c`:** The struct is an array of 8-byte entries (base `0x20014e3c`,
   indexed `uVar3*8`, count in `0x2000027a+2`), processed in `CAN_Parallel_Inverter_Sync`
   (`0x0800557c`). Byte+6 of each entry is a **priority/rank value for the "maximum fallback"
   master election** between parallel Venus units: value `0` marks an entry as "no data yet"
   and aborts the election (`if (*(entry+6) == 0) { reset; return; }`); otherwise
   the entry with the highest byte+6 value across all entries wins (comparison/max tracking
   in `0x2000027a+3`, winner index in `0x2000027a+4`). This directly confirms the
   "maximum fallback" mechanism described in §13.1 (mode byte `0x20015320+0x12 == 3`, while
   mode `2` takes the direct "fast path" to the first entry) and thus also the security
   assessment there: An attacker with CAN access can inject arbitrary byte+6 values via forged extended-ID frames
   and thereby deliberately determine which (simulated) node wins the
   master role.

**Still open:** exact field meaning of `0x20014f82+0x24` (presumably padding); writer
of `0x200152F0`/`0x20014fa8` (CAN RX path plausible, but unverified); writer of the
`chrg_flag`/`force_chrg` byte at `+0x16`/`+0x17`.

### 13.16 Quectel/cellular subsystem (#21, #22, #49)

**#21** `Quectel_SSL_Certificate_Manage` manages 3 named slots ("CA"/"User Cert"/"User Key")
for mTLS — but in the running firmware image **only the delete branch is reachable** (from
factory reset/BLE reset); upload/query/list have 0 callers. Actual certificate material
is not located in the Control FW image (see also §13.27 private key finding).

**#22** `Quectel_URC_JsonFrameParser`: clean 511-byte overflow protection, no infinite loop.
But a resync wait pattern (`"id=%s&latest=true"`) looks like a format string artifact
that is unlikely to appear literally in real modem responses — **suspected parser
hang after the first frame** (not runtime-verified, open question).

**#49** `Modem_ParseThreePhaseActivePower` reads the same buffer as #22 (JSON pipeline
UART→brace assembler→key/value extraction), no signature/plausibility check of the values.
Whether these flow unfiltered into `CT_PowerSetpoint_Compute` (§13.21) could not be
verified without gaps (queue consumer not located).

**Addendum (deep dive):**

**On #22 (parser hang):** Suspicion **confirmed** — bug found. Full decompilation of
`Quectel_URC_JsonFrameParser` (0x08010eb0) shows a byte-by-byte state machine parser with
two modes via the flag `*DAT_08010ff4`: mode 0 = normal JSON frame collection (until `}\r\n` is detected at
brace depth 0 → flag is set to 1); mode 1 = "resync wait", which compares incoming
bytes against a 16-byte reference pattern (`DAT_08010ff8` → `0x0805c65e`). This pattern
is not a standalone constant but a substring (offset 74–89) of the HTTP URL
`"http://%s.hamedata.com/external-services/api/v1/devices/backup-configs?devid=%s&latest=true"`,
specifically the 16 characters `"id=%s&latest=tru"` — including the literal `%s` placeholder, which
practically never occurs as a byte sequence in real modem URCs. On a byte mismatch the
collection buffer is cleared (`memset` + length counter to 0), **but `*DAT_08010ff4` is not
reset** — only the single success path (16/16 bytes matched, lines 53–57 of the decompilation)
resets the mode to 0. Since this pattern effectively never occurs exactly in normal modem traffic,
the parser stays permanently stuck in the resync-wait state after the first successfully parsed JSON frame
and discards every further character from then on, without ever recognizing another
JSON frame — a reproducible parser hang/DoS after frame 1 (runtime-verified
only by code path analysis, not by hardware test).

**On #49 (queue consumer):** Located. `Modem_ParseThreePhaseActivePower` itself writes to
no queue — it fills a stack structure directly (4×4 bytes: total/phase A/phase B/
phase C active power) in the caller `Modem_Response_Dispatch` (0x08026b40). On success, the latter sends the
structure via `xQueueSend(*DAT_08026bf0, &local_30, 0, 2)` into a queue (handle
in RAM `0x200000f8`). Consumer: `CT_GridPower_Controller` (0x0802baa4) reads exactly the same
queue — `Modbus_Response_Builder(*DAT_0802bc64, &local_38, 0x1e)`, where `DAT_0802bc64` also
references `0x200000f8`. Despite its name, `Modbus_Response_Builder` is the actual
FreeRTOS `xQueueReceive` implementation (error strings reference `.../SDK/FreeRTOS/src/
queue.c`, uses `vTaskEnterCritical` — another naming error analogous to the known
batch naming problem). The received values are taken over into `DAT_0802bc6c[0..3]`, of which
one value (minus a fixed offset `*(short*)(DAT_0802bc70+0x83)`) flows via
`Power_Direction_Change_Check`/`Power_Delta_Detect` directly into `CT_PowerSetpoint_Compute(local_10,
local_14)` (line 72). **Confirmed:** The modem-parsed power values reach
`CT_PowerSetpoint_Compute` unfiltered (no range/plausibility check, only direction change/delta gating)
— the open question from #49 is thereby resolved.

### 13.17 MQTT protocol (#23, #24, #79)

**#23** MQTT v3.1 ("MQIsdp") confirmed hardwired, the v3.1.1 branch is dead
code in the default case. Credentials in the CONNECT packet unencrypted (protection only by TLS). **Confirmed
critical, see addendum at the end of this section:** The code path in `MQTT_Session_Init` that
skips `mbedTLS_SSL_Connection_Init` is real and reachable via unauthenticated BLE
provisioning — it activates a completely TLS-free "custom broker" mode,
credentials and payload data go out in plaintext.

**#24 Passive mode:** Cloud remote control (`work_mode=3`), power ±2500 (unit
unconfirmed) via JSON-RPC `"Passive"`, **no message authentication beyond TLS**.
The countdown watchdog forces power=0 on timeout (failsafe), but does not leave passive mode
automatically.

**#79** `MQTT_Config_ParseScheduleEntries` is not a standalone schedule mechanism, but
a second (text `key=value` instead of JSON) write path into the same 10-slot table
(`0x20014cfe`), which presumably is also read by `TimePlan_Evaluate_Setpoint` (§13.22) — relationship
between cloud schedule, local RAM array and RS485 register schedule (`Inverter_Set_Schedule_Reg`)
not conclusively clarified (possibly three parallel schedule stores).

**Addendum (deep dive, checked for security criticality):** `MQTT_Session_Init` (0x080187fc,
only caller: `CH395_MQTT_Init_And_CertSetup` @0x08024e32, `MQTT_Session_Init(7)`) contains
two occurrences of the same condition — identical at 0x08018840-0x08018852 (target host selection) and
0x080188a8-0x080188ba (TLS init skip):

```
if ((*(char *)(cfg + 0xe4) == 1) && (strlen(cfg + 1) > 5)) {
    // line 22f: uses cfg+0x22 (server string) instead of the default host as the CH395 target host
} ...
if ((*(char *)(cfg + 0xe4) != 1) || (strlen(cfg + 1) < 6)) {
    mbedTLS_SSL_Connection_Init();   // line 37f, 0x080188bc -> 0x08018364
}
// -> if flag==1 && strlen(Xid)>5, mbedTLS_SSL_Connection_Init is skipped ENTIRELY
```

`cfg` is a globally shared configuration block (0xe5 bytes, loaded from flash/EEPROM via
`Flash_Read_Protected(cfg, ..., 0xe5)`), whose layout was verified via
log format strings (`0x08059994`, `0x08059b1c`, `0x08059b7c`) and the BLE write routine:

| Offset | Field | Size |
|---|---|---|
| 0x00 | ID/type byte | 1 |
| 0x01 | **Xid** (device/topic identity, see `MQTT_Topic_Builder`) | 0x21 |
| 0x22 | **Server/Url** (custom broker hostname, used as CH395 target host when the flag is active) | 0x40 |
| 0x62 | Port | 2 |
| 0x64 | User (MQTT username) | 0x40 |
| 0xa4 | Pwd (MQTT password) | 0x40 |
| 0xe4 | **Flag** ("custom broker valid/active") | 1 |

**Writers of the flag (`find-cross-references`, 22 hits in total, of which 8 WRITE):**
- `MQTT_Config_VID_Init` (0x0801f394, 0x0801f3c6) — called unconditionally as the **very first step** of
  `MQTT_Session_Init` (line 21) and **derives** the flag anew on every
  session setup: reloads the config block from flash via `Flash_Read_Protected` if needed
  and sets `cfg[0xe4] = 1` as soon as `strlen(cfg+1)` (Xid) > 5 — regardless of which
  raw value a previous writer had set.
- `BLE_Recv_Cmd_Dispatcher`, command **0x50, subcode 0x0C** ("Write xid info", 0x080097e2ff.):
  parses the complete raw payload from the BLE packet via `BLE_WiFiConfig_Parse` in the format
  `<field,field,...>` directly into `cfg` (incl. Xid, server, port, user, pwd, flag), then writes
  **persistently to flash** via `Flash_Write_Protected(cfg, ..., 0xe6)` and subsequently
  implicitly triggers (via the next `MQTT_Session_Init`) the TLS skip path as soon as Xid is
  ≥6 characters long. According to the dispatcher's comment header ("0x50-0x51: VID/XID provisioning")
  this is a **regular, documented provisioning command without additional authentication**
  (no magic byte protection as with command 0x0C "Develop Mode", no password) — any client
  that can establish a BLE connection to the device can set server/user/pwd/Xid arbitrarily.
- `BLE_MQTT_Command_Execute` (0x080253ae, case `param_1==3`, "Xid err" path): also sets the flag
  directly to 1 if the already stored Xid has ≥6 characters — called from
  `BLE_Pending_Commands_Process`, so also reachable via the BLE command channel.
- Further writers (`MQTT_Session_BLE_Notify`, `CH395_MQTT_Init_And_CertSetup`,
  `MQTT_Connect_And_Subscribe`) set/read the flag in the normal session lifecycle, but are
  not independent external triggers.

**Assessment — path is really reachable, not debug dead code:** The TLS skip is not an
unused leftover, but the flip side of a **real product feature** ("own/
self-hosted MQTT broker" via Xid+server+port+user+pwd, provisionable via app/BLE command
0x50/0x0C). Exactly when this feature is actively used (flag==1 through a valid
Xid), the firmware **completely** forgoes `mbedTLS_SSL_Connection_Init` — no
TLS context is set up, `MQTT_Connect_And_Subscribe` runs directly over the raw CH395 TCP socket.
MQTT CONNECT credentials (user/pwd) and the complete pub/sub traffic to the
custom broker therefore go out in plaintext — without a warning in the code. Since server/user/pwd can be freely overwritten via the same
unprotected BLE command, any BLE client in range can additionally redirect the
device to an arbitrary broker (confirmation/aggravation of #45/#54 on
unprotected BLE write paths, §13.23). **Security assessment updated:** formerly "urgent
open question" → now **confirmed, really reachable risk** (not merely a debug flag).

### 13.18 CRC16 implementations & register system (#26, #50)

All three CRC16 variants are **byte-identical** (poly 0xA001, reflected) — the redundancy is
purely API-related (pointer vs. value init), no functional difference. `CRC16_Modbus_Incremental`
has **0 callers** (dead code). Despite the name, **none** of the three runs through the
Modbus RTU/TCP dispatcher — used exclusively in OTA/config persistence; Modbus itself uses
a different, separate CRC16 routine (`CRC16_Calculate`, dual lookup table).

`Register_PackDescriptor` (8/4/4/5-bit category/subindex/field/flags) is verified as a bidirectional
address layer: TX via `Register_WriteValue`→queue (consumer not located,
presumably CAN TX task), RX via `CAN_FrameDispatcher`→`Modbus_ResponseDispatch`, which unpacks exactly
the same bit scheme from CAN response frames and stores it in Modbus-readable slot tables —
thus internally connecting CAN communication with externally readable Modbus registers. 5 bits (19:16) in the
descriptor are evaluated by no known decoder (open).

**Addendum (deep dive):** Consumer of the `Register_WriteValue` queue located.
`Register_WriteValue` (`0x08026598`) calls `Modem_QueueSendMessage` (`0x08027394`), which writes via
`xQueueSend` into a queue handle that is read from a shared RAM structure at `0x200000e8`
(field `+4` = `0x200000ec`; the pointer value `0x200000e8` exists as a flash literal
duplicated several times: `0x080273b0`, `0x0801d028`, `0x0802c08c`). The same queue is additionally
filled by `WiFi_HardwareResetSequence` (`0x0801cf1c`, case 0) — so it is not an exclusive
register queue, but a shared output queue. The consumer (reader) is `Modbus_SendResponse`
(`0x0802c060`, **0 callers and no data reference** — analogous to `App_MainLoopDispatcher` in
§13.24 presumably registered via an `xTaskCreate` function pointer, i.e. the CAN TX task we were looking for):
it fetches from the queue with a 1000-tick timeout and on success sends via `CAN_SendMessage` +
`vTaskDelay(1)`. **Name correction needed:** The function `Modbus_Response_Builder` (`0x080541f8`) called internally there
is, despite its name, **not Modbus-specific code**,
but the generic FreeRTOS queue implementation (`xQueueReceive`/`xQueueGenericReceive` —
verified via embedded error strings `.../SDK/FreeRTOS/src/queue.c` as well as internal
function names `prvCopyDataFromQueue`/`prvUnlockQueue`); it is used by 9 different callers
for different queues (among others `CAN_RxQueue_DrainAndDispatch`,
`Quectel_SendCmd_WaitResponse`, `Write_Handler`, `RS485_RTU_Frame_Dispatcher`) and should be
renamed (e.g. `FreeRTOS_xQueueReceive`) to avoid misinterpretations — case
for the duplicate check after batch naming.

### 13.19 Cloud report pipeline & connection watchdog (#27, #34)

**Correction:** The 8 (+1) `Cloud_Report_Fill*` functions are **not periodic
cloud telemetry**, but fill the response of an **unencrypted local
JSON-RPC-over-UDP query/response mechanism** (`MQTT_JSON_RPC_Dispatcher`, event-driven on
incoming UDP packet, no timer). Header (34 bytes, not "0x22B") = echo of the JSON-RPC request ID.
Separate from the already documented AES-128-ECB cloud telemetry pipeline
(`AES_Crypto_Stack_Analysis.md`) — no overlap, RPC responses are plain plaintext JSON.
Data exposed without recognizable authentication: serial number, MAC, 4 IP addresses,
time zone, real-time battery/PV/grid data — a strong device/location fingerprint for every
client in the same network segment.

`Cloud_EdgeDetectAndWatchdog`: 12000-tick watchdog (~2 min) without backoff/upper limit, triggers
`Cloud_Response_Action` on a connection problem (reset state flags + power setpoint
to 0 as failsafe) — no hard CH395 chip reset.

### 13.20 Debug/logging infrastructure & cJSON naming errors (#29, #35, #87)

**#29 Correction:** The documented addresses were assigned incorrectly. In fact there exist
**13 dedicated debug print functions** (battery, MPPT, inverter, CAN PGN incl.
lock/factory test flag, SSL certificate info, event/error log dump, WiFi, Modbus address), **all
with 0 callers** — compiled but unwired diagnostic code, no active
information disclosure risk in FW 149.2.

**#87 refuted:** `Task_AppendStatusToJSON` collects real FreeRTOS stack watermarks, but demonstrably
**does not** end up in cloud reports — only in a local `printf` debug output that is
itself unreachable (its only caller block is without references).

**#35 confirmed done:** cJSON name correction (`cJSON_InitHooks` vs. `cJSON_NewObject`)
already set correctly in Ghidra, no action needed anymore.

### 13.21 Relay staged timing & CT power control loop (#30, #32)

**#30** GPIO bit15 = **PE15** (GPIOE, STM32F1 standard registers BSRR/BRR/ODR verified),
direct contactor drive. 3-stage sequence (310/610/1010 ticks) corresponds to a classic
precharge→main relay close→confirmation/debounce sequence; closing is tied to an external
ready flag (not closing on time alone). Notable: stage 3 uses an
**XOR toggle** instead of set/clear on the ODR register — in case of state desync (e.g. reset
midway) the physical pin state may potentially deviate from the assumed software state.

**#32** Control chain confirmed: CT measurement → WorkMode/remote processing →
`Voltage_Stability_Check` (debounce against grid fluctuations) → `Power_Direction_Change_Check` →
`Power_Delta_Detect` → `CT_PowerSetpoint_Compute` (P controller with configurable 30–100% gain,
4-fold clamp, dead band **exactly 11 W or 15 W** depending on channel/mode selector, confirmed). On
grid problems the controller defensively goes to 0 W.

### 13.22 Event log system & schedule engine (#31, #37, #91, #92)

**#31 Correction:** The dedup window is **31 minutes**, not 31 seconds (the calculation in the code uses
hours×60+minutes difference). Both logs (DisplayError 14B/EEPROM 0x1100, SystemEvent
9B/EEPROM 0x2000, 20 slots each) are **write-only** from the Control FW's point of view — no
readout function found in this firmware (presumably debug backdoor or another MCU).

**#37** 10-slot schedule (`TimePlan_Evaluate_Setpoint`) confirmed: weekday bitmask + packed
HH:MM start/end time + power setpoint (special value -1 = standby/grid follow). Priority = slot
index (lowest index wins, no explicit priority logic). Trigger every 300 ticks or
on flag. Writer of the RAM array (`0x20014cfe`) not located — relationship to
`MQTT_Config_ParseScheduleEntries` (§13.17) open.

**#91/#92** `RTC_TimeToFiveMinSlot`: `(hour*60+minute)/5`. The RTC is set via **three** channels:
HTTP (with `DateTime_Validate`), MQTT (**without recognizable validation** — most conspicuous
single finding), BLE. The schedule engine continues to run independently of cloud reachability with the last
set/possibly drifting RTC time, no "time invalid" lock state.

**Addendum (deep dive):** Schedule store relationship (§13.17 #79) clarified — there are **not
three independent stores**, but two functionally separate systems. Since `0x20014cfe` itself
is never referenced as a static address (only reachable via per-function duplicated flash literals with
the same value), the writers were verified via the literal values instead of a direct
cross-reference to the address:

1) The local 10-slot RAM array `0x20014cfe` is a **single shared store** with
   at least four writers:
   - `MQTT_Config_ParseScheduleEntries` (`0x08049674`, §13.17 #79): parses text `key=value` into
     a staging buffer (`0x20019091`) and copies all 100 bytes to `0x20014cfe` via `memcpy` (call at `0x08049806`).
   - `MQTT_JSON_RPC_Dispatcher` (`0x0801a5a4`, "Manual" branch, approx. lines 383–396): writes
     directly slot by slot into the same table (base value `0x20014cfe`, verified as a 4-byte literal at
     `0x0801b344` by raw memory reading — wrongly typed there as a 1-byte scalar by Ghidra's auto-analysis).
   - `Write_Handler` (`0x08050f20`, register range `0xa85c`–`0xa85e`, shared by Modbus TCP **and**
     RS485-RTU FC06/FC10): writes directly into the same table (base literal at
     `0x08051bc4` = `0x20014cfe`).
   - `BLE_Recv_Cmd_Dispatcher` (`0x08007f58`): writes into the same table via a base literal
     `0x20014cfc` (= table − 2 bytes, presumably 2-byte header/count before the 10-slot array).

   The table is read by `TimePlan_Evaluate_Setpoint` (`0x0802d018`, base literal
   `0x0802d0e4` = `0x20014cfe`, thereby confirming #37) as well as by `BLE_Build_Settings_Response`
   (`0x0800ae5c`, base literal `0x0800af0c` = `0x20014cfe`) for reporting the current
   schedule back to the app.

2) `Inverter_Set_Schedule_Reg` (`0x08005d74`) is called by `Write_Handler` from a **different**
   register range (`0x30`–`0x3e`, not `0xa85c`+) and is in fact a
   **standalone, independent third path**: It does **not** write to `0x20014cfe`, but
   persists to EEPROM (offset `0x36bf + n*8`) and forwards the value via
   `Register_PackDescriptor`+`Register_WriteValue` to the same CAN TX queue described in
   §13.18 (addendum) — a mirror/sync write to external
   (inverter-side) registers, independent of the local RAM schedule engine.

   Result: **two** functionally separate schedule mechanisms — one shared local
   RAM array with four writers/two readers (MQTT text, MQTT JSON-RPC, Modbus/RS485 register,
   BLE), plus an independent CAN register mirror path to the inverter — not three parallel
   stores as suspected in §13.17 #79.

### 13.23 Network transport dispatch & CH395 server/reset (#33, #36, #45, #54, #74, #81)

**#33** Transport 1 (previously "unknown") identified: native Quectel modem MQTT via
`AT+QMTPUB` (modem-internal stack, parallel to the device's own software MQTT/TLS in transport 3).

**#36** `CH395_Reset_And_Reinit` via BLE (cmd 0x0C, case 6, "protected" only by a public 3-byte magic
prefix) has **no rate limiting** — plausible DoS vector against the entire
network connectivity (Ethernet + Modbus/TCP/broadcast services depending on it).

**#45/#54 confirmed critical:** Port 8090 (UDP broadcast, `255.255.255.255`) is effectively a
**cloud command interface without a cloud** — any device on the LAN can trigger the same full
`MQTT_JSON_RPC_Dispatcher` method set by broadcast (WiFi config, setpoints, factory reset), the only
"check" is an IP≠0 check. Port 8091 (TCP) runs via a real `CH395_TCPListen`, runtime handler
not located.

**#74** Contradiction to the previous documentation: Modbus port 502 uses the same mode=2/`TCPListen` path
as the TCP server (port 8091) — points to **TCP instead of UDP**, confidence lowered to "medium",
datasheet reconciliation as a TODO. Surprising finding: The RS485 Modbus register map is
initialized on CH395 reset **via the same CH395 SPI bus** (`RS485_Modbus_MapRead/WriteRegister`),
without its own mutex — only safe because the sole caller already holds the global CH395 lock.

**#81** `CH395_SPI_SendCmd`, 26 callers: 24 classic CH395 driver functions + the two
RS485 Modbus register map functions from #74.

**Overall picture:** Local network attack surface considerably larger than "one Modbus port" — port
8090 alone suffices for complete remote configuration without cloud and without auth.

**Addendum (deep dive):** Both open items from #45/#54/#74 clarified.

*Port 8091 runtime handler found:* `Network_ReceiveAndDispatchData` (`0x0804d4e8`, 0 findable
callers — presumably its own FreeRTOS task). Waits via `xQueueReceive` for a queue signal,
then reads up to 256 bytes explicitly via socket descriptor **1** (`CH395_GetRecvLen()`/`CH395_ReadRecvBuf(1,...)`
— exactly the descriptor that `CH395_Init_TCPServer_Socket` assigns for port 8091)
and passes each byte individually to `BLE_GATT_DispatchCommandByte` — **the same
byte state machine handler as the BLE GATT command channel**. Port 8091 is thus functionally a
TCP tunnel into the BLE GATT command dispatcher, without any additional
authentication recognizable in the code. For comparison: the separate `Modbus_Dispatcher` (`0x0801e43c`) serves,
cleanly separated, socket descriptor **0** (`CH395_GetRecvLen(0)`/`CH395_ReadRecvBuf(0,...)`) —
both handlers run independently over the same CH395 TCP driver.

*Modbus port 502 transport confirmed (TCP, confidence now high):* `CH395_Init_Modbus_TCP_Socket`
(`0x08048754`, only caller `CH395_Reset_And_Reinit`) sets source **and** destination port to
`0x1f6` = **502 (dec.)**, mode byte = **2**, socket descriptor **0**. `CH395_Init_TCPServer_Socket`
(`0x08032c0c`) analogously sets port `0x1f9b` = **8091 (dec.)**, also mode 2, socket descriptor
1. In `CH395_Socket_Open_ByDescriptor` the mode 2 branch leads in both cases directly into a
call of `CH395_TCPListen()` — an operation that exists in the CH395 driver only for TCP sockets
(no UDP equivalent). Modbus port 502 is thus confirmed **with high confidence as TCP**
(not UDP); the datasheet reconciliation left open in #74 is replaced by this code evidence.

### 13.24 Shutdown/Watchdog/MainLoop/FreeRTOS kernel (#38, #40, #47, #55, #80, #82, #83, #86, #94)

**Address conflict #86/#94 resolved:** `0x08053ba4` is `FreeRTOS_StartScheduler` (standard
portable layer), **not** the SysTick handler. The real SysTick ISR is a different, previously
undocumented function (`Periodic_Tick_Handler`, `0x0802d9ca`), which in addition to the
FreeRTOS tick also increments an app-specific energy counter — correct #94 in the tracking document to
this address.

**#55** `App_MainLoopDispatcher` (0 findable callers — presumably an `xTaskCreate` table):
the actual order is **shutdown → OTA update dispatcher → CAN update → periodic tasks →
cloud watchdog** (the OTA step was missing from the original short description).

**#47 Correction:** `Comm_Watchdog_CheckTimeout` triggers **no** reboot/shutdown — only
`OTA_Set_SlotStatus(..., failed)` + an internal retry counter reset. It is an OTA-download-specific
timeout, not a general system watchdog.

**#40 Correction:** `Shutdown_Sequence_Handler` does not suspend 7 tasks, but stops 7
software **timers**. There are **four** distinct, partly overlapping timer-stop locations
(`Shutdown_Sequence_Handler`, `OTA_PrepareShutdown`, `System_StopAllTimers` for BLE reboot,
others) — functionally redundant, but not faulty, since there is always an NVIC system reset
at the end.

**#38 Idle watchdog:** Reboot after 3600 ticks without PV yield AND without charge/discharge activity —
can be **triggered by legitimate nightly standby** (not a pure hang detector).
Practically most relevant finding: the device might restart recurrently at night.

### 13.25 Heap/memory allocator (#48, #76, #77)

**Correction:** There are **two independent heaps**, not a variant of the same system — Heap A
(`malloc`, 1.276 B pool, only for `calloc`/`realloc`) and Heap B (FreeRTOS standard `heap_4`,
71.680 B confirmed, for practically everything including the complete TLS/X.509/RSA/ECC stack via
`mbedTLS_Calloc`). OOM behavior is clean (NULL return, checked by callers).

**#77 Heap canaries confirmed missing:** Neither `heap_Free_Coalesce` nor `Heap_InsertFreeBlock`
check magic bytes/guard values before linking — every header value is trusted unchecked.
`pvPortFree` has only a double-free plausibility check (no independent canary), on
violation only a log, no halt. Additional finding: `calloc`/`Heap_Calloc` multiply
size parameters without an overflow check (CWE-190). Network-controlled allocation sizes go through
the mbedTLS/TLS certificate path (5 identified functions) — real exploitability depends on
an overflow bug in a parser that has not yet been demonstrated (no directly exploitable finding,
structural risk).

### 13.26 Custom crypto & mbedTLS core primitives (#51, #53, #58–62, #66, #67, #75)

All constant-time properties checked (#58 MemCompare, #59 RSA-PKCS1v15 unpad/Bleichenbacher
protection, #62 GCM zeroize-before-return, #67/#75 MPI comparisons) **confirmed correct**, no
deviation from the mbedTLS 2.28 reference. #60: only SECP256R1 enabled (build decision,
not a weakness per se). #61 `mbedTLS_ECP_Check_PubPriv`: 0 callers, dead code. #66 already
corrected in batch 18, no longer an open item.

**#53 Correction:** `GCM_Setup_Hash_Subkey`/`GCM_GHASH_Multiply` are **not** an independent
AES-GCM implementation, but internal mbedTLS `gcm.c` helpers (`gcm_gen_table`/`gcm_mult`) without the
`mbedTLS_` prefix — the hypothesized BLE use is not evidenced by the call graph.

**#51** Software AES confirmed (S-box/T-table generation, no STM32 CRYP accesses) —
a performance disadvantage, but on a Cortex-M without D-cache practically no cache-timing risk
(assessment).

### 13.27 mbedTLS PKI/certificates incl. private key question (#63, #64, #65, #71, #72, #78, #85)

**🔑 Private key finding — addendum 2026-07-10, SUCCESSFULLY EXTRACTED:** The previous assessment
(key resides in an external, memory-mapped QSPI region and is not statically extractable) is
obsolete. A raw byte scan of the flash image (instead of following decompiler pointers) found a
ROT-obfuscated AES-128 key candidate statically in flash. This allowed the AWS IoT root CA,
device certificate and **private key** to be successfully extracted from the flash dump and cryptographically
verified (`openssl rsa -check` passed, modulus reconciliation with the device certificate exactly
identical). The private key is thus very much **statically in the flash image** (encrypted with
AES-128, not in an external QSPI region) — the "unproven hypothesis" is thereby
refuted/clarified. Purpose of use still confirmed: mTLS client authentication for the
MQTT connection (effectively AWS IoT Core), `VERIFY_REQUIRED` set.

> **Security-relevant finding — deliberately not documented publicly.** The key, the
> complete extraction methodology and the extracted credentials themselves are **not part of this
> public repo**. It is a hard-coded AWS IoT client certificate including private key, shared across
> product lines — already reported confidentially to the manufacturer via responsible disclosure, details are withheld until
> it is fixed. Internal, non-public documentation on this is kept locally in the `security/` folder (not part of this repo).

**#63** Encrypted PEM support confirmed missing. **#65** only RSA/ECKEY/ECKEY_DH. **#71** RSA-
PKCS1v15 verify: **0 callers, entirely dead code** — supports the already known
signature-less OTA (§13.10/§13.11). **#72** Sign-with-blinding + verify-after-sign confirmed
(fault-attack protection correct). **#78** Max. 2048-bit RSA confirmed (hard rejection of larger
keys). **#85/#90** are **the same function** (address duplicate in the tracking document, Ghidra name
still `FUN_080528c8` despite documented renaming) — complete X.509 pipeline
confirmed actively used in the TLS handshake.

### 13.28 mbedTLS TLS/SSL handshake & config (#68, #69, #70, #73, #84)

**#68** AEAD-only confirmed — no CBC code path present, hence no padding-oracle surface.
**#69** TLS 1.2 hard-enforced (min=max=version 3.3) confirmed. **#73** 16-state handshake
corresponds 1:1 to the standard mbedTLS client, no skipped certificate check; naming error
found: `mbedTLS_SSL_Parse_ServerKex` (0x0804f284) is actually the
ClientHello **write** function. **#84** HMAC-SHA256 PRF confirmed, no MD5/SHA1 fallback.

**#70 ⚠️ confirmed critical:** `mbedTLS_SSL_Conf_CA_Chain` is called with a **hard-wired literal
`NULL`** as the CRL parameter — the only caller in the entire program, no
OCSP substitute. Certificate revocation is structurally never checked. In combination with an otherwise strong
TLS configuration (1.2-only, AEAD-only, client mTLS, hostname check), the missing
revocation check remains the only reliable weak point — relevant as soon as a server certificate
ever has to be rotated/revoked.

### 13.29 MPPT/GPIO data structures (#57, #88)

**#57** Complete MPPT struct layout at RAM `0x20014F40` (56 bytes) reconstructed from debug format strings:
Mppt_State/Error/Temp/Warning, PV1–4 each V(0.1V)/I(0.1A)/P(0.1W), daily/monthly/
yearly capacity (10 Wh), battery/ground/PE voltage — documented in an HA-suitable form. Writer of the
structure not conclusively located; most likely candidate a UART AT command path
to the MPPT sub-MCU (not CAN, as originally assumed).

**#88** Names `GPIO_Pin_Clear`/`GPIO_Pin_Set` **not applied** in Ghidra (still
`Generic_StructField_Set_0x14`/`FUN_08012d90`) — content (STM32F1 BSRR/BRR) confirmed. Largest
user group (~25 callers): CH395 SPI chip select. Other categories: I2C bit-bang SDA/SCL,
main relay GPIOE15, BLE/WiFi module reset, I2C bus recovery — including a direct GPIO access
via Modbus registers 45023–45029 (diagnostic/factory test interface according to the Write_Handler
comment).

**Addendum (deep dive):** Writer of the MPPT structure (`0x20014F40`) located — path is
**CAN**, not UART AT as last assumed (the original CAN hypothesis is thus confirmed again).
Chain: `CAN_RxQueue_DrainAndDispatch` (`0x080292d4`, presumably its own FreeRTOS task) reads
CAN frames from the receive queue → `CAN_FrameDispatcher` (`0x0802e698`) branches based on
bit fields of the CAN ID to, among others, `Telemetry_Register_Dispatcher` (class 4),
`Modbus_ResponseDispatch` (class 2) and `Protocol_AA_CommandDispatch` (0xAA protocol, §13.13).
For class 2, `Modbus_ResponseDispatch` (`0x0802ea54`) branches further based on a command byte
embedded in the CAN status word; on value 3 it calls `Modbus_StoreRegisterSlot` (`0x0802e94c`),
which extracts a 3-bit slot index (value range 1–7) from the status word and uses it to write an
8-byte slot starting at base `0x20014F40` (verified via `get-data`) — **7 slots × 8 bytes =
exactly the 56-byte structure size from #57**, which additionally confirms the layout. Remark: The
function names `Modbus_ResponseDispatch`/`Modbus_StoreRegisterSlot` are presumably a
naming artifact from an earlier batch — the real code processes CAN payloads, not the
Modbus PDU format (candidate for correction in Function_Tracking.md, see also §13.18 on the
related `Register_PackDescriptor` bit scheme). No UART receive path with write access to
`0x20014F40` found — the AT command hypothesis is thus considered refuted.

### 13.30 Re-audit Modbus/RS485 + inverter/register/energy cluster (2026-07-14)

**Context:** Targeted re-audit of the two topic clusters most important for the HA integration in
`VNSD_Control_FW_Function_Tracking_new.md` (Modbus/RS485: 33 functions, inverter/register/energy:
89 functions) against fresh Ghidra decompilation (including caller/callee context), analogous to the
batch 18 re-audit of the mbedTLS cluster. A total of 85 functions checked (33/33 Modbus + 52/89
inverter/register, prioritized by missing description, medium/low confidence and central
dispatchers). Error rate significantly lower than in the mbedTLS cluster (batch 18: 62 misnamings):
Modbus cluster 2 of 33 questionable names (94% correct), inverter/register cluster 1 of 52
questionable name (98% correct). All detailed results (renaming proposals, added
descriptions) maintained directly in the respective table rows of `VNSD_Control_FW_Function_Tracking_new.md`
(source column "Doc (re-audit 2026-07-14)").

**Confirmation of an existing observation:** The finding in §13.29, suspected to be a naming artifact,
on `Modbus_ResponseDispatch`/`Modbus_StoreRegisterSlot` (CAN payload instead of Modbus PDU) was independently reproduced
during the re-audit and extended by further functions of the same family:
`Modbus_StoreWithHandshake` (0xCB), `Modbus_StoreDualSlot` (0xCE), `Modbus_StoreValue16` (0xFE),
`Modbus_StorePairSlot` (0xFF) all hang off the same `Modbus_ResponseDispatch` entry point, which is
reached exclusively from `CAN_FrameDispatcher` (class 2, CAN ID pattern
`(id&0xffff)>>0xc==0 && (id&0xffffff)>>0x14==2`) — no call from the RS485 or
TCP Modbus path. These five functions thus form a standalone **"CAN-tunneled
pseudo-Modbus protocol"** for parallel operation/multi-pack communication, which has nothing to do
with the actual RS485/TCP Modbus server (section 2 above) except for the (presumably historically
caused) naming.

**New finding — `Modbus_SendResponse` (`0x0802c060`) sends via CAN, not via USART:** The
decompilation shows a call path via `CAN_SendMessage`/`CAN_SetupTxMailbox`, no
USART access. Currently 0 callers findable in Ghidra (possibly a function-pointer target or dead code).
Fits thematically with the CAN pseudo-Modbus family above — possibly the response counterpart
function to it, but not conclusively bound to a caller.

**New finding — `Modbus_Response_Builder` (`0x080541f8`) is not a Modbus function:** Decompilation
shows a generic, blocking FreeRTOS queue-receive implementation (timeout via
`vTaskPlaceOnEventList`/`xTaskCheckForTimeOut`, functionally analogous to `xQueueReceive`), which is used
across the whole code base: `RS485_RTU_Frame_Dispatcher` (UART byte reception), `CAN_RxQueue_DrainAndDispatch`,
`Quectel_SendCmd_WaitResponse` (modem AT commands), `Write_Handler`, `CT_GridPower_Controller`,
`Network_ReceiveAndDispatchData`, `WorkMode_Modbus_ResponseCache_Refresh`. At no point does it build
a Modbus frame. The name is a misinterpretation from an earlier batch — renaming
proposed (e.g. `Queue_Receive_WithTimeout`), but not yet applied (see convention:
check renamings centrally for duplicates).

**Shared BatteryParams struct (SRAM `0x20014F82`):** Numerous functions in the
inverter/register cluster (`Power_Limit_Clamp`, `Inverter_Power_Setpoint_ScaleFactor_Calc`,
`Inverter_PowerSetpoint_DeadbandClamp`, `WorkMode_State_Machine`, `Telemetry_Timestamp_Get`,
`Grid_Power_Dynamic_Adjust`, as well as cloud/BLE/MQTT builders) according to
`find-cross-references` (38 reference locations) all access the same global struct base pointer
`0x20014F82`. The numerous `DAT_*` literal pool addresses in the code (`DAT_08012fe0`,
`DAT_0801304c`, `DAT_08013df4`, `DAT_0802f2ec` and others) are all pointers to the same structure.
A worthwhile target for a future struct retype action in Ghidra (field offsets at `0x0`,
`+0xC`, `+0x12`, `+0x18`, `+0x1E`, `+0x24`, `+0x28`, `+0xF` among others can already be
reconstructed from several functions).

**Naming suspicion — `Telemetry_Timestamp_Get` (`0x0802f2b4`) — resolved, see §13.46:** No RTC/time access in the
code. Reads a disable flag (offset `+0xF`) and a scaled
value (offset `+2`, divided by 10) from the above-mentioned BatteryParams struct and returns a status code 0–3 (0 = disabled). Is stored, among others,
in `Cloud_Report_FillPowerFlow` directly next to an SOC byte — suggests a
charge/capacity class code rather than a timestamp. **Update 2026-07-15:** renamed to
`BatteryParams_PowerFlowState_Get` (confidence `medium`, exact physical unit of offset+2 not
conclusively proven — details see §13.46).

**Confirmation of TCP/RS485 symmetry:** The 1:1 mirroring between the TCP and RS485 paths already documented in section 2.1
was confirmed during the re-audit; the only functional divergence is the
broadcast behavior (RS485 broadcast to address 0x00 deliberately sends no response, TCP has no
broadcast concept and always responds) — already correctly documented in section 2.1.

The remaining 82 checked functions of both clusters (including all central dispatchers such as
`Register_WriteValue`, `Register_PackDescriptor`, `CT_GridPower_Controller`,
`WorkMode_ChangeHandler`, `RS485_Modbus_RegisterMap_Init`) were verified and are correctly
named and described — no further corrections necessary.

### 13.31 Re-audit CAN / BLE / config EEPROM / OTA flash (2026-07-14)

Four topic clusters of the Control FW tracking table, which had never been checked individually against fresh decompilation
since the uncontrolled initial assignment
(batch 1-17), were re-verified by
sub-audit (4 parallel agents, 133 of 194 functions checked). For details/tables
see `VNSD_Control_FW_Function_Tracking_new.md`, sections "CAN bus", "BLE / GATT",
"Config / EEPROM", "OTA / Flash". Most important findings:

**CAN bus — CAN1 peripheral confirmed, but two functions misclassified:**
- CAN1 base address `0x40006400` (STM32 CAN1) verified in `CAN_SetupTxMailbox`; TX mailbox registers
  (TIxR/TDTxR/TDLxR/TDHxR at offset `0x180 + mailbox*0x10`) confirm real HW CAN use.
- `CAN_FrameDispatcher` distinguishes frames by `(id>>0xc)&0xffff`/`(id>>0x14)&0xffffff`: value `4` →
  `Telemetry_Register_Dispatcher`, value `2` → `Modbus_ResponseDispatch`, otherwise →
  `Protocol_AA_CommandDispatch` — a previously undocumented CAN frame ID bit field scheme.
- Modbus responses can be transmitted via CAN (`Modbus_SendResponse` → `CAN_SendMessage`),
  not only via RS485/TCP.
- `CAN_SendWorkModeFrame` (`0x0802c5b0`) and `CAN_SyncChangedRegisters` (`0x0802c5d8`) are, despite their
  names, **not CAN functions**: They exclusively call `I2C_BitBang_WriteBytes` with the fixed
  I2C slave address `0x48` (classic I2C bit-bang protocol: start/WriteByte/ReadBit-ACK/stop).
  This shows that the inverter board coupling partly runs via I2C, separate from the actual
  CAN bus for parallel operation/BMS updates. Renaming proposal (not yet applied, see
  tracking table): `I2C_SendWorkModeFrame` / `I2C_SyncChangedRegisters`.

**BLE — cmd 0x50/0x51 substructure refined, GATT characteristic handle found, major
misclassification finding:**
- Real GATT notify characteristic handle `0xFF02` confirmed (`AT+QBLEGATTSNTFY=ff02,<len>` in
  `BLE_GATT_Notify_Send`).
- Cmd 0x50/0x51 sub-selector `0x0A` = VID provisioning (string, flash `0x22`B), sub-selector `0x0C`
  = XID provisioning (structure type/2×31-63B strings/port/2 further strings, `0xE6`B total, flash address
  `DAT_0800999c`). The semantics of the individual config fields (host/topic prefix or similar) remain
  **unclear/need further checking**.
- **Most important finding:** The 18 functions in the address range `0x0804bd58`–`0x0804cc40` (so far all named with
  `BLE_`/`BLE_GATT_` prefix) are, according to the call chain, **not BLE functionality**, but a
  generic, transport-independent CLI/AT command engine: 5 parallel sessions, 16-byte
  command table entries (visibility flag, type code, match mask, value/handler pointer),
  `$name` variable substitution, 5-entry history ring buffer, tab completion. Initialized via
  `CLI_InitSession`, called from `CH395_Recv_Buffer_Setup` (CH395 = external Ethernet/SPI chip);
  the input dispatcher is called exclusively from `Network_ReceiveAndDispatchData` (CH395/Modbus TCP)
  — no BLE code path. Fits cmd 0x28 "Local API enable + port (Modbus TCP)". All
  existing neighboring functions are already correctly named `CLI_*`/`ATCmd_*`/`Util_*`,
  so the `BLE_GATT_*` prefixes are also stylistically inconsistent. Renaming proposals (18
  functions, `CLI_*` prefix) are available, see tracking table — **question to the user needed** whether
  this block should be kept as its own cluster "CLI/AT command engine" (cross-cutting topic,
  cf. documentation structure convention).

**Config/EEPROM — consolidated EEPROM address map (from comments + cross-reference analysis):**

```
0x160        DateTime (8B) — production date/reboot state
0x201        Discharge cutoff SOC (1B, complement)
0x202/0x204  U16 value each, max 2500 (0x9c4)
0x300        EPS (comment)
0x301        work_mode (comment) — contradiction with 0x374, see below
0x302        time_slots (100B = 10×10B)
0x366        auto_mode (comment)
0x367        1-byte flag/enum (<2)
0x369        1-byte value (0-4)
0x36b        1-byte value (threshold-validated)
0x36d/0x36e  0xAA magic + 2B value (validity pattern)
0x370        Capacity factor byte (30-100%)
0x371/0x372  api_enable / api_port (comment)
0x374        WorkingMode (0-7)
0x375        Bool flag (feature unclear)
0x376        String (13B)
0x383        Signed 16-bit power offset/calibration
0x388        String (13B, max. 12 characters+NUL)
0x441        server_type (0-4)
0x484/0x488/0x492  power_stats (4B each)
0x500        Runtime counter block (52B)
0x900/0x901  Modbus address + 0xAA validity
0x2000+n*9   Event log entries (9B/entry)
0x3500       meter_ip (16B)
0x36b7/0x36b9/0x36bb  2B SSL certificate check values each (CH395_MQTT_Init_And_CertSetup)
0x36bd       1-byte flag (BLE "sdv_en")
0x4000       User data block (36B) — internal comment calls it "net_cfg" (contradiction to the
             previous name "User data block", see below)
```

Open contradictions (not resolved, need further checking): (1) EEPROM `0x301` is internally commented as
"work_mode", while `Config_Write_WorkingMode` demonstrably writes to `0x374` —
unclear whether these are two separate fields or an outdated comment. (2) `Config_Save_UserDataBlock`
(EEPROM `0x4000`) — internal comment calls the area "net_cfg" instead of "user data"; callers
(`Config_SaveWithCRC`, `OTA_InitSlotConfig`) fit network/OTA configuration better.

Additional structural finding: The generic helper family `Config_Read_U8/U16/U32/String/Block`
is >80 % misnamed — 5 of 6 functions actually only call `EEPROM_Write` (not a
single `EEPROM_Read` call), so they are write rather than read functions. Presumably the result of a
single faulty batch naming run that assigned "Read/Write" across the board instead of by actual
access type. Renaming proposals see tracking table.

**OTA/Flash — two separate flash subsystems, QSPI command set, memory layout confirmed:**
- Internal MCU flash (STM32 flash controller: `Flash_ErasePage`/`Flash_ProgramWord`/`Flash_Lock`/
  `Flash_Unlock`/`Flash_WaitReady`) vs. external QSPI NOR flash (memory-mapped base `0x70000000`,
  offset conversion `addr - 0x70000000` confirmed in several functions).
- QSPI command set verified: `0x06` Write Enable, `0x20` Sector Erase (4KB), `0x32` Quad Page
  Program (256B pages), `0x6B` Quad Output Fast Read, `0xFF` dummy/release byte after
  read/CRC/checksum operations. Status register offset `+0x28` (bit 0 BUSY, bit 1/bit 3 transfer flags),
  data register offset `+0x60`.
- Flash memory layout per target system confirmed: EMS `0x80000`, MPPT `0x100000`, BMS `0x180000`,
  VNS `0x200000`, 512KB spacing each; download staging size constant `0x7D000` (500 000 bytes).
- **CRC algorithm correction:** `OTA_CRC_Verify`/`QSPI_Flash_CalculateCRC` use, contrary to previous
  documentation, **CRC-16 (Modbus)**, not CRC-32 (`CRC16_Calculate`, init `0xFFFF`, comparison `& 0xffff`).
- OTA flow confirmed: `OTA_Update_Dispatcher` → `OTA_Process_Pending_Updates` →
  `OTA_Slot_Config_Validate` → `OTA_Slot_Config_Summary_Build` → `OTA_Firmware_Download_Init` →
  `OTA_PrepareShutdown` → `OTA_Download_Retry_Handler` (max. 2 retries/slot, 4 slots). Chunks land
  via `OTA_WriteDataToRingBuffer` in a 5-slot ring buffer (`0x800`B/slot), `OTA_Flash_Page_Writer`
  writes 2KB at a time to flash, afterwards only verify + status code (`OTA_FW_Verify_And_Apply` writes
  nothing itself, contrary to its name — renaming proposal `OTA_FW_Verify_SetStatus`, see
  tracking table).
- `0x080273dc` (previously `SPI_Flash_ResetGPIO`) demonstrably only accesses `RCC_AHBPeriphResetCmd`,
  not a single GPIO register access — renaming proposal `QSPI_Controller_Reset`.
- `0x0802b8cc` (previously `Flash_ReadWrite_Transaction`) performs exclusively `SPI_Flash_SectorErase`
  calls (no read/write) — renaming proposal `Flash_EraseAddressRange`.

**Overall result of the re-audit:** 133 of 194 functions (69 %) of the four clusters checked. Of these approx. 88 %
correct, approx. 12 % with naming or description errors (details/renaming proposals in
`VNSD_Control_FW_Function_Tracking_new.md`, no Ghidra renaming performed without a central duplicate check).

### 13.32 Re-audit MQTT cluster (2026-07-14)

The MQTT cluster (`## MQTT — Client/Protocol/Payload`, 101 functions) had never been checked individually
against fresh decompilation since the initial assignment (exception: the duplicate resolution
`MQTT_Decode_RemainingLength` vs. `..._ViaCallback` from batch 19). 65 of the 101 functions were
checked (target 55-65 per the assignment reached): all with an incomplete description/missing
confidence value (15), all central dispatchers/handlers along the connect/subscribe/
publish/receive paths (25 — including `MQTT_Session_Init`, `MQTT_Connect_And_Subscribe`,
`MQTT_Connect`, `MQTT_Subscribe_Impl`, `MQTT_Process_IncomingPacket`,
`MQTT_Dispatch_PublishCallback`, `MQTT_ReceivePacket`, `MQTT_Transport_ReceiveAll`,
`MQTT_Client_SendAndReceive`, all serialize/deserialize/encode/decode helpers, `MQTT_TopicFilter_Match`,
`MQTT_KeepAlive_SendPing`, `MQTT_Next_PacketId`), plus a representative sample of the
JSON-RPC response builders, Publish_* variants and config parsers (25).

**Result: 63 of 65 functions (97 %) correctly named and described** — the MQTT core
(connect/subscribe/publish/receive/encode/decode, the complete JSON-RPC dispatcher ecosystem including the
`MQTT_Build*Response` family) is clean and consistent with the actual code. Two
misnamings found (error rate 3 %, significantly lower than in the previously audited
clusters — CAN/BLE/Config/OTA were at ~12 %, the mbedTLS cluster had 62 corrections in batch 18
for approx. 260 functions, i.e. ~24 %):

- **`MQTT_Subscribe_Handler` (`0x080109d8`) has nothing to do with MQTT subscribe.** The code
  iterates over 10 schedule slots (stride 10B, power field at offset+2 as int16) and clamps values
  >800 to 800, then `Config_Notify_Change(slot)` for each changed slot. Callers are
  `MQTT_JSON_RPC_Dispatcher` (directly after `EEPROM_Write(0x90,...)`) and `BLE_Recv_Cmd_Dispatcher` —
  in both cases as a config post-processing step after writing schedule/setpoint data,
  no SUBSCRIBE packet, no topic reference. Presumably derived wrongly from the
  MQTT call context at first naming instead of from the actual function body.
- **`MQTT_Topic_Build` (`0x0802fa18`) does not build a topic string.** The function takes an index
  (1-4) and returns one of several flash/RAM addresses (`0x310000` literal or one of three
  `DAT_` pointers); all four callers (`OTA_ValidateUrlSlots`, `BLE_Recv_Cmd_Dispatcher`,
  `Quectel_AT_Response_Parser`, `MQTT_JSON_RPC_Dispatcher`) pass the return result directly to
  `Flash_EraseAddressRange(..., 0xfb)` or `0x04` — the magnitude 0xfb (251 bytes) fits a
  URL/config slot, not an MQTT topic buffer. More of a generic slot address selector
  (presumably for OTA/server URL configuration slots) than a topic builder.

Both findings are pure naming/classification errors without security relevance; renaming proposals
are available (see tracking table, column "Naming suspicion"), but, as in all previous
re-audits, were **not applied in Ghidra** until the central duplicate check has been done.

**New findings (topic/payload structure):**
- `MQTT_Topic_Builder` (`0x080058e0`, to be distinguished from the above `MQTT_Topic_Build`) builds four
  topic strings via `Cloud_Telemetry_JSON_Builder`: `marstek/{xid}/server/{sn}/ctrl` and
  `marstek/{xid}/device/{sn}/ctrl` (only if a custom xid ≥6 characters is active, see §13.17) as well as
  always `marstek_energy/{sn}/device/{sn}/ctrl` and `marstek_energy/{sn}/App/{sn}/ctrl` — i.e.
  there are **two parallel topic namespaces** (`marstek/...` for the optional custom broker via
  xid, `marstek_energy/...` as the default cloud topic), both with the same `.../ctrl` suffix scheme.
- All `MQTT_Publish_Telemetry`/`MQTT_Send_Data_Buffer`/`MQTT_Publish_AI_Data`/etc. functions send
  via the same pattern: either directly via `Quectel_Modem_DataSend` (transport mode 1, TCP socket)
  or as AT command `AT+QMTPUB=0,0,0,0,"marstek_energy/{xid}/device/{sn}/ctrl",{len},{payload}`
  (Quectel modem AT interface) — so the firmware supports two redundant transport paths to the
  same cloud endpoint, controlled via a single mode parameter per publish call.
- Response payload field names confirmed (from the `MQTT_Build*Response` family, cJSON object names read directly
  from the code): `Bat_data`, `Ble_data`, `Device_data`, `Em_data`, `Es_mode_data`, `Es_data`,
  `Marstek_data`, `Pv_data`, `Wifi_data`, `Err_data` (error envelope), `Result_data` (set result) —
  supplements the existing table in §9 (MQTT field names → register mapping) with the generic
  JSON-RPC envelope mechanism (`MQTT_JSON_CreateResponseEnvelope` → root+ID+result array,
  `MQTT_JSON_AddTypedValue` supports only type 8=int and type 0x10=string).

### 13.33 Re-audit Config/EEPROM + OTA/Flash — continuation/deep dive (2026-07-14)

Continuation of §13.31 (there, 133/194 functions of the four clusters CAN/BLE/Config-EEPROM/
OTA-Flash were checked). This pass specifically re-checked the previously still open functions of the two
clusters **Config/EEPROM** and **OTA/Flash/SPI-Flash/QSPI**, with a focus on (a) all
functions without a description and (b) all remaining functions named `Read`/`Get` (because of
the error pattern found in §13.31 that several `Config_Read_*` were in truth write functions).

**Config/EEPROM:** 22 additional functions checked (13 without description, 5 named `Get`/`Read`,
4 from the `Config_Param5x` family), which together with the 8 from §13.31 makes **30 of 57 functions
(53 %)** checked. All 22 newly checked functions confirmed name and (where present)
description — **no further read/write mix-ups found**. In particular the remaining
`Config_Get_Capacity_Factor`, `Config_Get_WorkMode`, `Config_Get_DeviceModelCode`,
`Config_Read_String_0x388`, `Config_Read_ProductionDate` demonstrably only read (via `EEPROM_Read`
or direct struct access) — the error pattern found in §13.31 was apparently limited to the
`Config_Read_U8/U16/U32/String/Block` functions already identified there and does not extend
across the whole cluster. The two core primitives `EEPROM_Read`/`EEPROM_Write` (mutex/
queue-protected I2C access, previously without description) as well as `EEPROM_Mutex_Wait` and
`EEPROM_Config_Factory_Write` were documented.

**OTA/Flash/SPI-Flash/QSPI:** 21 additional functions checked (13 without description, 8 previously only
marked by name match without decompilation review), which together with the 6 from §13.31 makes
**27 of 61 functions (44 %)** checked. Here, too, **no naming/description errors found** —
the complete QSPI low-level set (`QSPI_ReadDataRegister`/`QSPI_WriteDataRegister`/
`QSPI_TransferWords`/`QSPI_SendAndReceive`/`QSPI_SendCommandIrqSafe`/`QSPI_ConfigureMode`/
`QSPI_ApplyRegisterConfig`) as well as the SPI flash GPIO/clock configuration
(`SPI_Flash_PeripheralEnable`/`QSPI_Controller_Reset`/`SPI_Flash_ConfigGPIO`/
`SPI_Flash_ClockGateControl`/`SPI_Flash_WriteEnableCheck`) and the command functions
(`SPI_Flash_SectorErase`, `SPI_Flash_QuadPageProgram`, `QSPI_Flash_QuadRead`,
`QSPI_Flash_PollStatusReady`, `Flash_SelfTest`) are all implemented in line with their names.
`OTA_FW_Verify_SetStatus` (previously without description) exactly confirms the verify logic already
documented in §13.31 (CRC-16 + dev_mask + model string, status codes 0x401-0x403).
`SPI_Flash_MutexTransaction` and `Flash_EraseAddressRange` (the latter was already corrected in content
in §13.31, now fully documented with a caller list) were likewise brought up to date.

**Error rate of this continuation: 0 of 43 checked functions (0 %)** — significantly lower than the
~12 % from §13.31 (first pass). Interpretation: The misclassifications from §13.31 were concentrated
in specific, already identified function families (`Config_Read_*` helpers,
`SPI_Flash_ResetGPIO`, `Flash_ReadWrite_Transaction`, `OTA_FW_Verify_And_Apply`); the other functions checked in
this pass (primarily low-level hardware primitives without complex
naming semantics) were already correctly named.

**Remaining stock:** Config/EEPROM 27 of 57 functions still unchecked (primarily the
`Config_Write_*`/`Config_Set_*`/`Config_Apply_*` family with names that clearly express write
semantics — low error risk based on the pattern found). OTA/Flash 34 of 61
functions still unchecked (primarily high-level OTA orchestration: `OTA_Update_Dispatcher`,
`OTA_Process_Pending_Updates`, `OTA_Slot_Config_Summary_Build`, `OTA_Firmware_Download_Init`,
`OTA_InitSlotConfig`, `OTA_Download_Retry_Handler`, `OTA_Flash_Prepare_ByTarget`,
`OTA_Flash_Page_Writer`, internal MCU flash functions `Flash_ReadWords`/`Flash_ReadWithECC`/
`Flash_ErasePage`/`Flash_Lock`/`Flash_Unlock`/`Flash_ProgramWord`/`Flash_WaitReady`/
`Flash_EraseRegion`/`Flash_WriteRegion` as well as various batch 20 functions). **Both clusters are
thus not yet 100 % re-audited** — continuation in a further session recommended if
full coverage is desired.

### 13.34 Re-audit cJSON vendor lib + Quectel modem/WiFi/AT cluster (2026-07-14)

First individual re-audit of the two clusters `cJSON — Vendor lib` (30 functions) and
`Quectel modem / WiFi / AT commands` (65 functions), previously never checked individually. Performed via
4 parallel partial audits (all 30 cJSON functions; 42 non-batch-20 Quectel functions in two
halves; sample of 8 of the 23 batch 20 Quectel functions), each via
`get-decompilation(includeCallers, includeCallees)`.

**cJSON cluster (30/30 checked, 0 naming errors requiring action, 11 descriptions corrected):**
The firmware uses a **modified cJSON variant**: `cJSON_New_Item(type, name)` already takes the
name at node allocation, so that `Create*` functions systematically have more
parameters than the real Dave Gamble cJSON API (e.g. `cJSON_CreateBool`/`_CreateString` with
name+value instead of only value; `cJSON_CreateDouble` with 3 instead of 1 parameter). This is consistent,
firmware-specific design and not a misnaming. The only real naming oddity:
`cJSON_AddItemToObject` (`0x0802a290`) takes only 2 parameters (name, item) and **always creates a
new** named wrapper node instead of mutating an existing object — does not correspond to the
3-parameter semantics of the real API `cJSON_AddItemToObject(object, string, item)`; the actual
linking into the parent object happens externally via `cJSON_AddItemToArray` in all 11 callers.
Proposal (not applied): `cJSON_CreateNamedItem`. `cJSON_NewObject` (`0x0802a32c`) corresponds
exactly to `cJSON_CreateObject(void)` — renaming optional, not an error. `cJSON_GetValue`/
`cJSON_FindChildByName` are not duplicates, but different internal helpers (value pointer
accessor vs. name search). Notable: `0x08013e04 JSON_ExtractFieldValue` does **not** belong
to the cJSON lib in terms of content (no recursive parser, only a strstr/strncpy scanner for an HTTP JSON API field
`device_id`), but stays in the cluster for consistency reasons (already correctly named, no
move necessary).

**Quectel cluster (50/65 checked = 77 %, of which all 42 non-batch-20 + sample 8/23 batch 20;
7 naming errors, 15 description corrections):**

1. `0x08022b4c Quectel_AT_Response_Parser` — actually parses **MQTT OTA metadata** (id/type/
   size/crc/url per device slot) and writes it via Flash_Write/Erase into flash, not generic
   AT reply parsing. Proposal: `Quectel_MQTT_OTA_Info_Parser`.
2. `0x0804d018–0x0804d1d4` (5 functions: `ATCmd_OutputLineTruncated`, `ATCmd_PrintCommandInfo`,
   `ATCmd_PrintPrompt`, `ATCmd_PrintReturnValue`, `ATCmd_WriteString`) — **do not belong to the
   Quectel modem**, but are output primitives of the generic CLI/AT command engine
   (`0x0804bd58`–`0x0804cc40`), which in the previous re-audit of 2026-07-14 was already corrected from `BLE_GATT_*`
   to `CLI_*` (see §13.31/batch 19 history). All 5 functions are called exclusively
   from there (`CLI_PrintCommandEntry`, `CLI_HelpOrInfoDispatch`, `CLI_InsertChar`/
   `CLI_TabComplete`/`CLI_ConfirmAndExecute`, `CLI_ExecuteCommand`, 13× various `CLI_*`). The
   name part "ATCmd" wrongly suggests a Quectel AT modem reference. Proposal: `CLI_*` prefix
   (`CLI_OutputLineTruncated`, `CLI_PrintCommandInfo`, `CLI_PrintPrompt`, `CLI_PrintReturnValue`,
   `CLI_WriteString`). With this, the already noted "open item for the user" (separate
   CLI/AT command engine cluster?) grows by a further address range outside the originally
   found 18-function block.
3. `0x08011008 Quectel_TCP_SendData` — the format string branch contains both the pure TCP form
   (`"AT+QISEND=%d,%d"`) and the UDP destination address form (`"AT+QISEND=%d,%d,\"%s\",\"%s\",%d"`), the caller
   is the protocol-neutral `Network_TransportDispatch` — not a TCP special case. Proposal:
   `Quectel_QISEND_SendData_TCP_UDP`.
4. `0x0801285c Quectel_SignalQuality_PeriodicCheck_Save` (from batch 20!) — **no modem/AT/
   UART reference**, operates instead on inverter/PV string telemetry structs (`0x20014f40`/
   `0x20014e90`), the caller environment is purely energy/inverter related
   (`Grid_Export_Limit_Periodic_StateMachine`, `Battery_Forced_Charge_Check`, `Inverter_Sync_Init`).
   Presumably a misclassification *within* batch 20 (function landed in the wrong cluster/with the
   wrong naming pattern). Proposal: `Inverter_PVString_ChannelValue_PeriodicCheck_Save`. This is
   the first confirmed error in the batch 20 sample of this cluster (1 of 8 = 12.5 %) — the
   previous assumption "batch 20 more reliable" thus does not hold without exception.

Further findings without a need for renaming: Several helper functions named "TCP_"
(`Quectel_TCP_ConnectionState_Query`, `Quectel_TCP_SendAndVerify`) are demonstrably also used by
`Quectel_UDP_CommStateMachine`/`Quectel_UDP_OpenSocket`, since Quectel AT+QISTATE/AT+QISEND
are connectID-based and protocol-independent (descriptions added, names left). The
suspected risk of confusion between 5 AT send variants (`Quectel_SendCmd_WaitResponse`,
`Quectel_AT_SendAndVerify`, `Quectel_AT_SendAndPollResponse`, `Quectel_AT_SendRaw_WithMutex`,
`Quectel_TCP_SendAndVerify`) was **not** confirmed — they are in fact 5 different
implementations (byte assembler+timer / notify+strstr / tick polling / raw without verify / own
buffer). The 5 GPIO reset functions (`WiFi_Module_RestartStateMachine`,
`WiFi_HardwareResetSequence`, `WiFi_ModuleResetDispatcher`, `WiFi_ResetWithRecoveryWait`,
`WiFi_PowerCycleSequence`) are likewise cleanly distinguished from one another, no mix-up.

**Error rates:** cJSON 0/30 real naming errors (0 %, 1 optional alternative); Quectel 7/50 naming errors
(14 %, of which 5 from a single misclassified address block). All renaming proposals are
**not** applied in Ghidra — central duplicate check still pending (open item).

### 13.35 Re-audit completion BLE/GATT + inverter/register/energy cluster (2026-07-14)

Final re-audit of the two previously partially checked clusters `BLE / GATT` (50 functions) and
`Inverter / Register / Energy logic` (89 functions). All remaining rows not yet
marked "Re-audit 2026-07-14" were checked (28 in the BLE cluster excluding the functions already renamed to `CLI_*` earlier,
64 in the inverter cluster) via 5 parallel partial audits, each via
`get-decompilation(includeCallers, includeCallees)`. With this, **both clusters are now fully
(100 %) re-audited**: BLE/GATT 34/34 relevant rows (50 minus 16 functions already corrected to `CLI_*`),
inverter/register/energy logic 89/89.

**BLE/GATT cluster (28/28 checked, 1 naming error, 1 structure anomaly, 6 description corrections/additions):**

1. `0x0800a740 BLE_CRC_Calculate` — actually computes only a **XOR checksum** (`uVar2 ^=
   *(byte*)(param_1+i)` in a loop), not a real CRC with polynomial/table. Proposal (not
   applied): `BLE_XOR_Checksum_Calculate`.
2. `0x08008000 BLE_Command_Handler` — **structure anomaly, not a naming error — resolved, see §13.46**: The
   code body was identical in content to a case block from `BLE_Recv_Cmd_Dispatcher` (0x08007f58,
   located directly before it) and contained unresolvable `in_stack_...` parameters in the decompilation —
   a typical symptom of a wrongly set Ghidra function boundary in the middle of a
   switch statement. **Update 2026-07-15:** Root cause confirmed (fully overlapping
   ghost definition inside `BLE_Recv_Cmd_Dispatcher`, all references `PARAM` type instead of
   `CALL`/`JUMP`), function definition removed via `FunctionManager.removeFunction()`, doc row
   removed, cluster header BLE/GATT (36→35) corrected. Details see §13.46.
3. `0x08007f58 BLE_Recv_Cmd_Dispatcher`, `0x0800a764 BLE_Send_Response`, `0x08026bfc
   BLE_XidServerConfig_Parse` — had no description, now added based on the code
   (dispatcher: frame validation + switch on >50 cmd handlers; Send_Response: central
   response frame construction, >80 callers; XidServerConfig_Parse: 6-field parser for
   VID/XID provisioning from BLE cmd 0x0C).
4. `0x0800b430 BLE_Build_BMS_Data_Response` — is **not only used for BLE**, contrary to the previous
   description, but the prepared buffer is also reused by `MQTT_Publish_BMS_Full_Data`
   for the MQTT channel.
5. `0x08010718 BLE_Pending_Commands_Process` — functionally as documented (WiFi setup/OTA/
   socket ctrl dispatch), but according to `find-cross-references` has **0 callers in the current
   code path** (possibly reachable only via an unresolved function pointer) — added as a note.

**Inverter/register/energy cluster (64/64 checked, 4 naming errors/suspicions, 8
description corrections/additions, 1 factual correction):**

1. `0x08005860 Inverter_Register_Buffer_Init` — **clear misidentification**: the only caller is
   `MQTT_Connect_And_Subscribe`, no RS485/inverter reference in the code. The function does
   memset+memcpy and calls `Flash_Obfuscated_String_Decode()` — decodes an obfuscated
   flash string (presumably an MQTT/TLS credential) into a buffer for MQTT client setup.
   Potentially relevant for the ongoing TLS/AWS IoT credential project (see memory entry
   "Control FW TLS certificate extraction"). Proposal: `MQTT_Credential_Buffer_Decode` (name
   provisional, exact semantics of the decoded string still to be clarified).
2. `0x08005adc Inverter_Sync_Init` — contrary to its name, is called on **every** pass from
   `MainLoop_Periodic_Tasks`, not just once; "Init" only fits the first-run branch
   (MarkAll on empty flag), afterwards continuously Timeslot_ApplyConfigOnSync + Inverter_Apply_BatteryParams
   + 3000-tick WorkMode frame + I2C sync. Proposal: `Inverter_Sync_Periodic` or similar.
3. `0x08005d00 Timeslot_Bitmap_Set_Slot2` and `0x08005dd4 Timeslot_Bitmap_SetClear` — both carry
   a misleading "Timeslot" name, although they serve as a generic register dirty bit field for
   various inverter registers (WorkMode, ChgVolt, DischgVolt, PowerSetpoint, GridPower,
   BatteryParams) and **none** of their 9 callers in total has a schedule/time-of-day reference.
   Proposals: `Inverter_RegDirty_Mark_BatteryParams` and `Inverter_RegDirty_Bitmap_SetClear` respectively.
   (2026-10-02: the "inverter registers" are front-panel LED segments of a TM1650-type driver; in v151 the wrapper is `Inverter_RegDirty_Mark_PvVoltage`.)
   Indirectly also affects the name family `Inverter_BatteryParams_Timeslot2_SetClear`
   (0x08006530, calls `Timeslot_Bitmap_SetClear`) — there "Timeslot2" is likewise technically
   imprecise in the context of battery parameters, but was checked without its own rename proposal.
4. `0x08032348 Register_PackDescriptor` — factual correction: actually **36 callers** (not 35 as
   in the previous description), verified via the complete caller list.
5. Several smaller description refinements: `Inverter_RegDirty_MarkAll`/`_Mark_ChgVolt`/
   `_Mark_DischgVolt` set or clear depending on the parameter (not only "set");
   `Inverter_Power_Value_Scale` additionally applies `Inverter_PowerSetpoint_DeadbandClamp` (was missing
   from the description); `Inverter_Apply_BatteryParams` additionally calls
   `Grid_Power_Dynamic_Adjust` at the end and contains an init gate + Timeslot2 bitmap logic (was missing from the
   description); `0x0801e290 WorkMode_Flag_Reset_And_TriggerSetpointEval` had a **factually
   wrong** description ("clear connection flag, trigger disconnect") — actually, when
   the broadcast flag is set or the 300ms timer has expired, `TimePlan_Evaluate_Setpoint()`
   (setpoint recalculation) is triggered, no connection/disconnect reference in the code.
6. Three debug print functions (`Battery_Config_Debug_Print`, `MPPT_Debug_Print`,
   `Inverter_PowerSetpoint_Apply_Wrapper`) have **0 callers/references** in the entire binary —
   added as a note, not a misnaming (fits the already known pattern of dead
   debug/wrapper functions).

**Error rates:** BLE/GATT 1/28 clear naming error (3.6 %) + 1 structure anomaly without a
renaming recommendation; inverter/register/energy 4/64 naming errors/suspicions (6.3 %, of which 1
clear misidentification and 3 misleading "Timeslot" names in the same function family). All
renaming proposals are **not** applied in Ghidra — central duplicate check still
pending (open item, consistent with §13.34).

### 13.36 Re-audit hardware/HAL cluster (GPIO/ADC/SPI/I2C/USART/RCC/RTC, 81 functions, 2026-07-14)

First individual re-audit of the "Hardware / HAL" cluster (81 functions, previously never
checked individually). Performed via 4 parallel partial audits (RCC/SysTick/NVIC/FPU 21, GPIO/ADC/SPI 17,
I2C bit-bang/EEPROM_I2C 21, RTC/UART/USART 20) plus 3 reference verifications done myself
(RCC_APB2PeriphClockCmd, SPI_DeInit, USART_Init) — **all 81 functions checked 100 %**, each
via `get-decompilation(includeCallers, includeCallees)` plus `get-data` to resolve referenced
`DAT_` peripheral base addresses.

**Verified STM32F1 peripheral base address map** (confirmed by direct Ghidra analysis, values
read from real `DAT_` constants in the program — not just taken over from reference literature):

| Peripheral | Base address | Bus | Evidence function |
|---|---|---|---|
| RCC | 0x40021000 | — | RCC_APB2PeriphClockCmd (`DAT_08028014`) |
| FLASH interface | 0x40022000 | — | formerly RCC_EnablePeripheralClock (`DAT_080077f0`) |
| AFIO | 0x40010000 | APB2 | formerly STM32_RCC_Clock_Config (`DAT_08012c4c`) |
| SPI1 | 0x40013000 | APB2 | SPI_DeInit (`DAT_0802b9dc`) |
| SPI2 | 0x40003800 | APB1 | SPI_DeInit (`DAT_0802b9e0`), ADC_Peripheral_Init (`DAT_0801796c`) |
| SPI3 | 0x40003C00 | APB1 | SPI_DeInit (`DAT_0802b9e4`) |
| USART1 | 0x40013800 | APB2 | USART_Init (`DAT_0802dfb4`) |
| I2C1 | 0x40005400 | APB1 | EEPROM_I2C_* cluster (14 functions) |
| GPIOB | 0x40010C00 | APB2 | SPI_Timer_Peripheral_Init/EEPROM_I2C_GPIO_ClockPulseRecovery |
| GPIOC | 0x40011000 | APB2 | I2C_BitBang_* cluster (5 functions) |
| GPIOD | 0x40011400 | APB2 | GPIOD_Pin9_Write |
| GPIOE | 0x40011800 | APB2 | SPI_Timer_GPIO_ChipSelect_Init, Peripheral_SetBit15 |

**13 naming errors found with code evidence** (proposals, **not** applied in Ghidra — user
decision pending):

| Address | Old name | New name (proposal) | Rationale (short version) |
|---|---|---|---|
| 0x080077e4 | RCC_EnablePeripheralClock | Flash_SR_ClearFlags | Base address 0x40022000 = FLASH interface (not RCC 0x40021000), offset+0xC=FLASH_SR; only callers Flash_ErasePage/Flash_ProgramWord |
| 0x08012a04 | STM32_RCC_Clock_Config | AFIO_PinRemapConfig | Base address 0x40010000 = AFIO (not RCC); register offsets fit MAPR/MAPR2, not clock config |
| 0x08028044 | RCC_SetClockBypass | RCC_LSICmd | Bit-band alias back-calculation → RCC_CSR (0x40021024) bit 0 = LSION, not HSEBYP/LSEBYP |
| 0x08028050 | RCC_SetClockOutputFlag | RCC_RTCCLKCmd | Bit-band alias back-calculation → RCC_BDCR (0x40021020) bit 15 = RTCEN, not MCO/clock output |
| 0x08001fb6 | GPIO_PulsePin | RCC_BackupDomainReset_Pulse | Only calls GPIO_WritePinValue(1)/(0), target is RCC_BDCR.BDRST (see below), no GPIO |
| 0x08028038 | GPIO_WritePinValue | RCC_BDCR_BDRST_Write | Bit-band alias 0x42420440 → 0x40021020 (RCC_BDCR) bit 16 = BDRST, no GPIO register involved (back-calculation verified) |
| 0x080178bc | ADC_Peripheral_Init | SPI2_Peripheral_Init | Base address 0x40003800 = SPI2 (not ADC1=0x40012400); RCC_APB1(0x4000=SPI2EN)/APB2(GPIOB) fit SPI2 |
| 0x0802b6d4 | ADC_SetControlBit0x40 | SPI_Cmd | Sets/clears SPI_CR1 bit 6 (SPE) — standard SPL SPI_Cmd() |
| 0x0802ba04 | ADC_ApplyChannelConfig | SPI_Init | Writes SPI_CR1 + clears I2SCFGR.I2SMOD — standard SPL SPI_Init() |
| 0x0802ba40 | ADC_SetScanMode | SPI_SSOutputCmd | Sets/clears SPI_CR2 bit 2 (SSOE) — standard SPL SPI_SSOutputCmd() |
| 0x08017ed4 | SPI_Timer_Peripheral_Init | EEPROM_I2C_Peripheral_Init | Base address 0x40005400 = I2C1 (not SPI/timer); GPIOB8/9 AF-OD (I2C1 remap SCL/SDA); callees exclusively EEPROM_I2C_* |
| 0x08028628 | RTC_GetTime | RTC_GetDate | Reads RTC_DR (0x40002804, offset+0x04), decodes year/month/day/weekday; caller semantics in RTC_GetDateTime confirm the swap |
| 0x08028674 | RTC_GetDate | RTC_GetTime | Reads RTC_TR (0x40002800, offset+0x00), decodes hour/minute/second |
| 0x08028750 | RTC_SetTime | RTC_SetDate | Write target = WPR−0x20 = RTC_DR; parameter mapping year/month/day/weekday |
| 0x08028404 | RTC_SetDate | RTC_SetTime | Write target = WPR−0x8 = RTC_TR; parameter mapping hour/minute/second/PM |
| 0x0800468c | UART_Packet_Receive_Parse | CH395_Packet_Receive_Parse | No UART reference; exclusively calls CH395_ReadRecvBuf/CH395_GetRecvLen/CH395_SPI_WriteCmd (CH395 SPI Ethernet controller) |

In addition 2 consistency notes without a hard error (not counted in the above table):
`RCC_PeriphBitControl` (0x08028018, actually RCC_APB2RSTR — functionally correct, but
inconsistent with the naming scheme of the sister functions RCC_APB1/APB2PeriphResetCmd) and
`Peripheral_SetBit15` (0x08032324, confirms GPIOE pin 15 — name not wrong, but less precise than
the sister pair GPIOD_Pin9_Write suggests).

**I2C↔CAN confusion suspicion NOT confirmed:** The case feared in the assignment
(I2C bit-bang functions wrongly named CAN_*, as happened with other functions in an earlier session) was
**refuted** for the complete I2C bit-bang/EEPROM_I2C cluster (21 functions)
— all base addresses (I2C1=0x40005400 for 13 EEPROM_I2C functions, GPIOC=0x40011000
for the 6 I2C_BitBang core functions, GPIOB=0x40010C00 for the bus recovery) match the
names exactly. Only finding: the tracking doc named an outdated caller "CAN_SendWorkModeFrame" for
`I2C_BitBang_WriteBytes` (0x0802fdb8) — in the current Ghidra state this function is already called
`I2C_SendWorkModeFrame` (0x0802c5b0, which in turn calls `Inverter_Sync_Init`/
`Inverter_Apply_BatteryParams`) — a pure doc sync error, no Ghidra fix necessary, corrected in the
tracking table.

**Resolved — third/fourth UART instance with atypical address:** `USART_Init` (0x0802df0c) checks
`param_1` against three base address constants: `DAT_0802dfb4=0x40013800` (USART1, APB2/PCLK2), as well as
`DAT_0802dfb8=0x40015000` and `DAT_0802dfbc=0x40015400` — **both do not correspond** to the
STM32F103 standard addresses for USART2 (0x40004400) or USART3 (0x40004800), since the MCU is not an STM32F103.
With the MCU identification as **Nations N32G45x** (see section "MCU identification" above), both addresses
are resolved: according to the official `n32g45x.h`, `APB2PERIPH_BASE + 0x5000 = 0x40015000` =
`UART6_BASE` and `APB2PERIPH_BASE + 0x5400 = 0x40015400` = `UART7_BASE` — both APB2-clocked, matching
the observed PCLK2 baud rate calculation. UART6 is also named independently in the firmware code as
`UART6_IRQHandler` (formerly wrongly `USBFS_MAPL_IRQHandler`). Only affected the internal branching logic of
`USART_Init`, not its name (the function itself remains correctly named, since it applies
generically to any USART instance).

**7 description corrections applied directly in VNSD_Control_FW_Function_Tracking_new.md** (name
unchanged): GPIO_ConfigPinAsInput, RCC_AHBPeriphResetCmd, RCC_APB1PeriphResetCmd,
RCC_PeriphBitControl, RCC_GetFlagStatus (missing description added in each case), GPIOD_Pin9_Write
(wrong description "MPU/Flash region protection" replaced — the name was correct, only the description
was wrong) and I2C_BitBang_WriteBytes (outdated CAN caller note corrected).

**Error rate:** 13/81 clear naming errors (16.0 %) + 2 consistency notes without a need for renaming.
This puts this cluster above the previous project error rate (avg. approx. 6–14 % in the other
clusters re-audited on 2026-07-14), which confirms the focus suspected in the assignment,
"peripheral base address mix-up" for HAL-near functions — however primarily as an
RCC↔FLASH/AFIO and ADC↔SPI mix-up, not, as feared, as an I2C↔CAN mix-up. All
renaming proposals are **not** applied in Ghidra — central duplicate check and
user decision are still pending (open item, consistent with §13.34/§13.35).

### 13.37 Re-audit completion Config/EEPROM + OTA/flash cluster (2026-07-14)

Continuation/completion of the re-audit campaign started in §13.33: the remaining 27 of 57
functions in the "Config / EEPROM" cluster as well as 34 of 61 functions in the "OTA / Flash /
SPI flash / QSPI" cluster have now also been checked individually via `get-decompilation(includeCallers,
includeCallees)`. **Both clusters are thus at 100 % "Re-Audit 2026-07-14"**
(57/57 and 61/61 respectively).

**1 naming error found with code evidence** (suggestion, **not** applied in Ghidra):

| Address | Old name | New name (suggestion) | Reasoning |
|---|---|---|---|
| `0x08006f98` | EEPROM_Clear_RebootState | EEPROM_Save_RebootTimestamp (or EEPROM_Write_RebootDateTime) | The function does initialize a local 8-byte buffer with 0, but immediately overwrites it with `RTC_GetDateTime(&local_10)` and then writes this (non-zeroed) buffer via `EEPROM_Write(0x160, &local_10, 8)`. So the current RTC date/time is persisted before the reboot, nothing is cleared. The sole caller is `System_Reboot` (directly before the actual reboot trigger, after `Config_Save_RuntimeCounters`). Noteworthy: `Config_Read_ProductionDate` (0x08006b62) reads the same EEPROM offset 0x160 as an 8-byte DateTime — so the field is originally the production date, but is overwritten with the current timestamp before every reboot (presumably misused as a "last clean reboot" marker). If this is correct, `Config_Read_ProductionDate` might also be better named `Config_Read_LastRebootOrProductionDate` — but no rename suggestion here, since without further code evidence (e.g. the write path for the initial factory production date) it cannot be reliably distinguished. |

**1 description correction** (stale cross-reference, name unchanged): `OTA_Flash_Page_Writer`
(0x0802fdec) referenced in the documentation description a function "OTA_FW_Verify_And_Apply" that no longer
exists in the current Ghidra state — the actual callee on a complete
page transfer is `OTA_FW_Verify_SetStatus` (0x08004efc, already re-audited in §13.33). Corrected.

**33 further descriptions supplemented/refined** (caller context, exact register/EEPROM offsets,
byte sizes resolved from hex literals) without a substantive error finding — names each confirmed,
among others for the complete QSPI low-level chain (`QSPI_Flash_IsBusy`, `QSPI_TransferReady_Poll`,
`QSPI_WriteComplete_Check`, `OTA_QSPI_ReadRegion_CalcCRC`, `SPI_Flash_DataChecksum_Calc`), the
internal flash driver (`Flash_ReadWords`, `Flash_ReadWithECC`, `Flash_ErasePage`,
`Flash_GetStatusFromFlags`, `Flash_Lock`/`Flash_Unlock`, `Flash_ProgramWord`, `Flash_WaitReady`,
`Flash_EraseRegion`, `Flash_WriteRegion`) as well as the OTA process dispatcher
(`OTA_Update_Dispatcher` → `OTA_Process_Pending_Updates`, `OTA_Slot_Config_Validate`,
`OTA_Slot_Config_Summary_Build`, `OTA_Firmware_Download_Init`, `OTA_PrepareShutdown`,
`OTA_Download_Retry_Handler`, `OTA_StopAllTimers`, `OTA_InitSlotConfig`, `OTA_Flash_Prepare_ByTarget`,
`OTA_Set_SlotStatus`, `OTA_FlashPageWrite_Counter_Increment`) as well as the remaining config cluster
(various `Config_Write_*`/`Config_Set_*`/`Config_Apply_*` wrappers and terminal functions — for all
functions named `Set_`/`Apply_`, a targeted check was made for the Get_/Set_ mix-up feared in the
task description; **not a single one found**, all of them actually write).

**Error rate of this sub-round:** 1/61 clear naming error (1.6 %) + 1 description stale reference.
Markedly below the error rate of the hardware/HAL cluster (§13.36, 16.0 %) and at the lower end of the
clusters so far — plausible, since Config/EEPROM and OTA/Flash had already caught the most
conspicuous `Config_Read_*`-actually-write cases in the first sub-round
(§13.33), and the remaining functions are predominantly simple, easily readable wrappers/register writers.

**Both clusters thus completely (100 %) re-audited: Config/EEPROM 57/57, OTA/Flash 61/61.**

### 13.38 Re-audit CH395 cluster (Ethernet controller, 54/54 functions, 2026-07-14)

Complete individual check of all 54 functions of the "CH395 — Ethernet Controller" cluster via
`get-decompilation(includeCallers, includeCallees)` — so far this cluster had only been touched by
the duplicate fix in batch 19 (`CH395_Socket_SendData` vs. `_ViaSocketPtr`), never checked individually
against fresh decompilation. The cluster is now at **100 % "Re-Audit 2026-07-14"**
(54/54).

**Result: 0 naming errors.** All 54 functions are correctly classified as `CH395_*` — the
feared misclassification (e.g. pure socket buffer management without HW access incorrectly named as
CH395, or conversely real SPI accesses given a generic name) was **not** found. The
complete SPI low-level chain (`CH395_SPI_Cmd_WithData`, `_SPI_ReadByte`, `_SPI_WriteCmd`,
`_SPI_CmdWaitReady`, `_SPI_Send_Data`) consistently accesses the chip via `SPI_BeginCommand`/`SPI_WriteByte`/
`SPI_ReadByte` (generic SPI HAL functions, not part of this cluster), with
CH395 command opcodes as the first byte (0x2C version, 0x3C RecvBuf, 0x3B RecvLen, 0x30 SocketStatus,
0x35/0x37/0x38/0x36 Open/Connect/Disconnect/Listen, 0x39 SendData, 0x58/0x56/0x57/0x59/0x31-34
set register) — all opcode values in the documentation description match the code. The
higher orchestration layers (`CH395_Socket_Open_ByDescriptor`, `CH395_Reset_And_Reinit`,
`CH395_MQTT_Init_And_CertSetup`, `CH395_UDP_ServerTask`/`_DataHandler`) consistently call into the
SPI chain or each other, no misassignment to other peripherals (I2C/CAN/UART)
found — unlike the CAN cluster (§13.31, where 2 I2C bit-bang functions were incorrectly
named as CAN), this pattern did not occur here.

**4 description corrections/supplements** (names unchanged/confirmed in each case):

- `0x08003644` `CH395_SPI_Send_Data` — had no description; added: CMD 0x39, core send function
  with socket busy poll (max. 10×3ms) and mutex, byte-by-byte transmission via `SPI_WriteByte`. Central
  function called by practically all other CH395 send paths (9 callers: DNS query,
  Modbus TCP handler FC03/06/10, `CH395_ConfigAndSendSocket`, `CH395_SPI_SendData_Verified`,
  `CH395_Socket_SendSafe`, `CH395_Socket_SendData_ViaSocketPtr`).
- `0x08024d80` `CH395_MQTT_Init_And_CertSetup` — had no description; added based on an existing
  Ghidra comment (already present in the decompilation, presumably from an earlier session): orchestrates
  TLS certificate decryption (`TLS_Cert_Decrypt_All`), MQTT session init, CRC16 of the certificates
  towards EEPROM 0x36B7-0x36BB. 686 bytes in size, 0 callers found in the current state (presumably
  called from the main loop/state machine via a function pointer, not directly referenced).
- `0x080329b2` `CH395_Socket_SendData_ViaSocketPtr` — had no description (batch 19 duplicate);
  added: thin wrapper, dereferences a socket pointer (byte 0 = socket index) and directly calls
  `CH395_SPI_Send_Data`. 0 callers in the current state.
- `0x080195a4` `CH395_Recv_Buffer_Setup` — the description was **factually wrong** and was corrected:
  the documentation claimed "initialize DMA receive buffer (512B)"; in fact the function at
  no point accesses SPI/CH395 hardware and there is no DMA usage. It sets a field in the
  CH395 socket descriptor and then calls `CLI_InitSession(buf, size, 0x200)` — this is the same
  `CLI_InitSession` that was already identified in §13.31 as the init function of the (there newly
  recognized) generic CLI/AT command engine (18 functions in the range `0x0804bd58`-`0x0804cc40`,
  originally incorrectly named `BLE_*`). The 512 (0x200) is thus not a DMA buffer size
  but the session buffer size, which is internally subdivided into 5 slices; the session lookup is done via
  the version string "VNSD_0_v1492". Additional finding: **the function has 0 callers/cross-references**
  (`find-cross-references` returns 0 `to` references) — presumably dead code or only reachable via a
  function pointer not resolved by Ghidra. The name `CH395_Recv_Buffer_Setup` remains
  unchanged (no rename suggestion), since the function conceptually still belongs to the CH395
  receive path (call context according to §13.31 from the CH395/Modbus TCP receive path), even though it itself
  performs no SPI operation — pure description correction.

**No rename suggestions from this cluster.** The only remaining open item is the question already
posed to the user in §13.31 about the CLI/AT command engine (with which `CH395_Recv_Buffer_Setup`
is linked via `CLI_InitSession`).

**Error rate:** 0/54 naming errors (0 %) — by far the lowest error rate of all
clusters re-audited so far (cf. hardware/HAL 16.0 % in §13.36, Config/EEPROM+OTA 1.6 % in §13.37).
Plausible: the cluster was originally named very systematically according to CH395 command opcodes
(direct reference to the CH395 datasheet recognizable), and the SPI low-level functions are structurally
uniform (mutex → SPI_BeginCommand → opcode → optional parameters → Generic_StructField_Set_0x10 →
mutex release), which makes naming errors unlikely.

---

### 13.39 Re-audit FreeRTOS kernel cluster (task/queue/timer/heap, 85 functions, 2026-07-15)

All 44 functions from the initial assignment (batches 1-17, never individually decompiled)
plus a sample of 11 of the 41 batch-20 additions (2026-07-09) were checked — a total of 55/85 (65 %) individually
checked via `get-decompilation(includeCallers, includeCallees)`.

**Core finding: 0 naming errors.** The cluster is exceptionally cleanly named. Reason: FreeRTOS is
a well-known open-source RTOS library, and the compiler embedded in this firmware (debug build with
`configASSERT`) the **original source file paths as string literals**, e.g.
`____SDK_FreeRTOS_src_tasks_c`, `____SDK_FreeRTOS_src_queue_c`, `____SDK_FreeRTOS_src_timers_c`,
`____SDK_FreeRTOS_portable_RVDS_ARM_CM4F` (assert error messages `"Error: %s: %d"` with file name +
line number from the `assert` macro). These strings directly confirm that functions such as
`FreeRTOS_eTaskGetState`, `prvAddCurrentTaskToDelayedList`, `prvAddNewTaskToReadyList`,
`prvCheckForValidListAndQueue`, `prvCopyDataFromQueue`, `prvCopyDataToQueue`, `prvDeleteTCB`,
`prvGetDisinheritPriorityAfterTimeout`, `prvQueueSend_CopyAndNotify`, `prvResetNextTaskUnblockTime`,
`prvSwitchTimerLists`, `prvTaskIsTaskSuspended`, `prvUnlockQueue`, `vTaskEnterCritical`,
`vTaskExitCritical`, `vTaskDelay`, `xQueueSend`, `xQueueReceive`, `xTimerStop_Internal`,
`vTaskSwitchContext`, `vTaskPlaceOnEventListRestricted`, `FreeRTOS_xTaskIncrementTick`,
`FreeRTOS_xQueueGenericSend` and `FreeRTOS_xTimerCreateTimerTask` actually correspond 1:1 to the real
FreeRTOS internal function names from `tasks.c`/`queue.c`/`timers.c`/`portable/RVDS/ARM_CM4F` or
`portable/MemMang/heap_4.c` — not a guess, but verified facts.

**The only `medium`-confidence function** (`FreeRTOS_Timer_InsertIntoActiveList`, 0x0804aae0,
already corrected on 2026-07-09 in batch 18, see memory) was checked again: 0 callers confirmed
(unchanged since the batch-18 correction), logic exactly matches `prvInsertTimerInActiveList` from
FreeRTOS `timers.c` (comparison of `xNextExpireTime` against the current tick count including the overflow case,
insertion into the active or overflow list via `vListInsert`). Name and description remain correct,
confidence stays at `medium` for lack of runtime observation (0 callers in the current build — presumably
because auto-reload timers reach this path via `prvSwitchTimerLists` rather than directly).

**6 missing descriptions added** (names unchanged, all correct): `Task_Init_CreateAll`
(0x0801947c, creates 14 tasks from a table + starts the scheduler), `prvQueueSend_CopyAndNotify`
(0x0804abdc), `FreeRTOS_vQueueAddToRegistry` (0x080504ac, queue registry entry),
`vTaskDelay` (0x08050524, 90 callers FW-wide), `xQueueSend` (0x08053eb0, 77 callers),
`xQueueReceive` (0x08054370, 61 callers). All six are central, extremely widely used
FreeRTOS API functions — their absence in the documentation was a pure documentation gap, not a naming error.

**Side finding (no rename, observation only):** The cluster contains two structurally almost identical
first-fit allocators with block splitting: `pvPortMalloc`/`Heap_Init`/`Heap_InsertFreeBlock`
(FreeRTOS's own heap_4 pool, 71,680 bytes, used exclusively by FreeRTOS kernel functions)
and `Heap_AllocFromFreeList`/`Heap_Calloc` (separate pool, callers are `malloc`/`heap_Realloc` from
the utility/libc corner, not part of the FreeRTOS kernel). Both heaps coexist independently
of each other — plausible cause: newlib `malloc` wrapper vs. FreeRTOS `pvPortMalloc`, no
naming error, but an indication that `Heap_AllocFromFreeList`/`Heap_Calloc` actually belong better in the
libc/heap allocator cluster (§13.25, in progress) than in the FreeRTOS kernel cluster —
reassignment left to the responsible cluster owner, no change made unilaterally.

**Error rate of this sub-round: 0/55 naming errors (0 %).** By far the cleanest sub-round of the
entire re-audit campaign — consistent with the expectation that a well-known, well-documented
open-source library (FreeRTOS) is significantly less error-prone when initially classified than
firmware-own business logic clusters.

### 13.40 Re-audit CLI/debug output/logging + system/reset/shutdown/watchdog (62 functions, 2026-07-15)

Both clusters were fully checked (45 + 17 = 62/62, 100 %), each function individually via
`get-decompilation(includeCallers, includeCallees)`.

**Main finding: systematic cluster misclassification, analogous to the known BLE/Quectel→CLI pattern.**
28 of the 45 functions in the "CLI / Debug Output / Logging" cluster (`0x0804bd80`–`0x0804d008`,
`CLI_DeleteCharForward` to `CLI_PutChar`) have **nothing to do** with debug output/logging — they are
the complete line editor/tokenizer/command dispatcher of the CLI engine: cursor movement
(`CLI_CursorForward`, `CLI_CursorLeftRepeat`), character insertion/deletion (`CLI_InsertChar`,
`CLI_DeleteChar`, `CLI_Backspace`), tokenizer (`CLI_TokenizeLine`, `CLI_TokenizeInput`,
`CLI_StripQuotes`), argument parser (`CLI_ParseNumber`, `CLI_ParseString`, `CLI_ParseEscapeChar`,
`CLI_ParseArgValue`, `CLI_DetectNumberFormat`), tab complete (`CLI_TabComplete`,
`CLI_CommonPrefixLength`), command execution (`CLI_ExecuteCommand`, `CLI_ProcessInputLine`,
`CLI_ConfirmAndExecute`), output formatting (`CLI_ListCommands`, `CLI_PrintCommandEntry`,
`CLI_ShowHelp`) as well as low-level I/O (`CLI_PutChar`). Direct call-graph evidence for the membership
in the already existing "CLI / AT command engine" cluster: `CLI_ProcessInputLine` calls
`CLI_Entry_MatchAndInit` and `CLI_FindEntryByName` (both there), `CLI_HistoryNavigate` (there) calls
`CLI_ClearToEndOfLine` (here), `CLI_InitSession` (here) calls `CLI_Session_Register` (there),
`CLI_ExecuteCommand` (here) calls `CLI_InvokeCommandHandler` (there). Both groups together
form exactly one single coherent subunit (address range `0x0804bd58`–`0x0804d7e4`,
gapless). Recommendation: move the 28 functions into "CLI / AT command engine" during a central
cleanup; the remaining 17 functions (`Debug_Mode_Set`, `EventLog_Record_*`, `debug_printf`,
`log_printf`/`log_SetLevel`/`log_SetEnabled`/`log_SetModeAndLevel`, `Debug_Print*`) are indeed
debug/logging and correctly stay in the cluster. However, all 28 CLI function names themselves are correct and
match the actual behavior — a pure cluster assignment question, no rename needed.
Not moved by this round itself (instruction: central review before cluster move).

**One real naming error in the system/reset cluster:** `Reset_Command_Wrapper` (`0x08004838`) has
no relation whatsoever to system reset. The function is a 1-line wrapper around `CH395_SPI_Cmd_WithData`
and is called exclusively by `CH395_Debug_PrintVersion` with an arbitrary SPI command byte
to query the CH395 Ethernet chip version via debug output — no
register/peripheral reset semantics, as the existing name suggests. Probable
origin of the name: `SPI_BeginCommand(6)` inside the called `CH395_SPI_Cmd_WithData` may have been
confused with a CH395 reset opcode, but is a hardcoded generic
read command, not a reset. Rename and cluster move suggestion see final answer/table;
not renamed by this round itself.

**Minor description correction:** `MainLoop_Periodic_Tasks` (`0x0802d268`) had the
misleading note "(13 callers)" — in fact the function has only **1** caller
(`App_MainLoopDispatcher`) and in turn calls 13 subsystem functions. Description refined to
"(13 called subsystems; 1 caller)".

**8 missing descriptions added** (names unchanged, all correctly verified): `debug_printf`,
`log_printf`, `Log_SetModeAndLevelCallback`, `Debug_PrintErrorAndEventLog`, `Debug_PrintWifiStatus`,
`Debug_PrintModbusAddress` (cluster CLI/Debug), as well as `Factory_Reset` and `System_Reboot`
(the latter already had a helpful Ghidra function comment about the NVIC AIRCR reset, which was
carried over into the documentation description) in the system/reset cluster.

**Error rate of this sub-round:** 1 real naming error among 62 checked functions (1.6 %) plus
1 structural cluster misclassification finding affecting 28 functions (62 % of the
CLI/debug cluster, 45 % of the total volume of this round) — the names themselves are correct for all 28,
only the thematic assignment is not. This finding thus falls consistently in line with the pattern already
known from the BLE/Quectel→CLI separation: the CLI engine was repeatedly distributed into the wrong
neighboring clusters during initial classification.

### 13.41 Re-audit completion MQTT cluster + Quectel modem/WiFi/AT cluster (2026-07-15)

Both previously only partially checked clusters were brought to 100 %: "## MQTT —
Client/Protocol/Payload" (101/101, previously 14/101 marked "Re-Audit 2026-07-14") and
"## Quectel Modem / WiFi / AT Commands" (60/60, previously 14/60 marked). A total of **133 previously
unchecked lines** (87 MQTT + 46 Quectel) were verified against name/description via `get-decompilation(includeCallers,
includeCallees)` — significantly more than the originally estimated
36 (MQTT) or 10-15 (Quectel) open lines; the estimate apparently was based on a mix-up
between "description present" and "individually re-verified against the code".

**Core finding: 0 naming errors in both clusters.** Not a single rename suggestion out of 133 checked
functions — both clusters were already cleanly named before this round (the MQTT protocol engine and
the Quectel AT command layer appear to have been named with particular care in earlier batches).

**4 description corrections/supplements:**
- `MQTT_BuildPvDataResponse` (0x0801bf08): "4 strings V/I/P/State" refined to "4 PV strings (PV1-4),
  each power/voltage/current/state as int" — cJSON_CreateInt, not text strings.
- `MQTT_Clear_SubscriptionSlots` (0x0801d342): refined — with stride 8B the code clears only the first
  4B field (topic pointer) per slot, not the full 8B slot.
- `MQTT_Calc_Remaining_Length` (0x0801dac8): the description was factually wrong ("remaining length for
  PUBLISH/PUBACK"). The actual sole caller is `MQTT_Serialize_Connect` — the function calculates
  the remaining length for the CONNECT packet (protocol name/version + client ID + optional
  will topic/message + username/password via `MQTT_String_ResolveLength`).
- `Quectel_MQTT_OTA_Info_Parser` (0x08022b4c): had NO description at all. Newly added: parses a
  "key=value" string received via MQTT (strtok) for up to 4 device OTA slots (mod/type/size/crc/
  url_len/url, URL assembled from multiple tokens up to 0xE6 bytes) and, after a complete
  parse pass, writes the OTA target data per slot into flash via `Config_URLSlot_AddressSelect` +
  `Flash_EraseAddressRange` + `Flash_Write_Protected`, with read-back verification via
  `Flash_Read_Protected`. 0 callers in the disassembly (presumably a callback from the MQTT JSON-RPC dispatch).
  The name remains factually defensible despite the flash-write portion (parser as the core function of the process).

**Side findings without documentation change:**
- `MQTT_Topic_StringCompare`: internally calls a function labeled `atoi` with 3 parameters
  (ptr, ptr, len) — a Ghidra thunk mislabel, actual behavior corresponds to memcmp
  (0 on equality after length comparison). The existing description "strlen+memcmp" was already correct.
- `MQTT_KeepAlive_SendPing` does not itself trigger a disconnect, but only sets error status -1 for an
  unanswered ping; the actual disconnect happens in the caller `MQTT_Process_IncomingPacket`.
- `Quectel_HTTP_GET_Request`/`Quectel_HTTP_ReadResponse`/`Quectel_HTTP_Config_SSLCtxId` internally call
  a function named `HTTP_POST_Request` (0x0800bcc4) — a generic AT executor despite the
  POST-specific name, noted for a later HTTP app layer re-audit round (task #27).
- Caller counts from the existing documentation confirmed multiple times (exactly recounted, no deviation):
  `MQTT_BuildErrorResponse` 42, `MQTT_RPC_BuildSetResult` 16, `MQTT_JSON_CreateResponseEnvelope`/
  `MQTT_JSON_SerializeAndSend` 11 each, `MQTT_JSON_AddTypedValue` 10, `MQTT_ParseRpcParams` 9,
  `MQTT_Transport_ReceiveAll` 5, `MQTT_Encode_RemainingLength` 6, `MQTT_CalcPacketSize` 3.

**Error rate of this sub-round: 0/133 naming errors (0 %).** Both clusters are thus fully
(101/101 MQTT, 60/60 Quectel) and duplicate-free checked — a central duplicate scan across both clusters
after completion yielded 0 duplicate function names.

---

### 13.42 Re-audit Utility/Byte-Helpers cluster + Miscellaneous/Not-yet-categorized (66 functions, 2026-07-15)

Both the `## Utility / Byte-Helpers / Timer-Helpers` cluster (43 functions, never individually checked before) and
`## Miscellaneous / Not yet categorized` (23 functions) were fully verified via `get-decompilation`
(incl. caller/callee) — 66/66, split across 3 parallel sub-checks (22/22/22).

**Result: 2 real naming errors, 25 description corrections/additions (already applied), 23
cluster re-sorting proposals for the Miscellaneous cluster.**

**Proposed renamings (NOT applied, central duplicate check still pending):**

| Address | Old name | New name | Rationale |
|---|---|---|---|
| `0x0800685a` | TLV_Record_Skip | DNS_QuestionRecord_Skip | Not a generic TLV skip: calls `DNS_Name_Decompress` directly and skips name+TYPE+CLASS (+4 bytes), no RDLENGTH/TTL as for DNS answer records. Only caller `DNS_ParseResponseHeader` (question-section loop) — function is DNS-specific. |
| `0x0802dd5e` | DataStream_ReadNext | Timeout_RefreshAndGetTicksRemaining | Does not read stream bytes, but instead (like `Timeout_IsExpired`) calls `FreeRTOS_xTaskCheckForTimeOut` on the same timeout struct type and returns the updated "ticks remaining" count. In `MQTT_ReceivePacket`/`MQTT_Transport_ReceiveAll` the same timeout pointer is called alternately with `Timeout_IsExpired` AND this function — the result serves as the timeout parameter for the socket read callback, not a read data byte. |

**Suspected bug (documented, not fixed):** `Parse_TimeString_To_HourMinute` (0x0802dccc), according to
the disassembly, does NOT encode `hour<<8|min` as previously documented, but rather `hour << (min+8)` (a variable shift amount
instead of a fixed byte pack) — for larger minute values the hour can potentially be lost through bit overflow/shifting.
This looks like a real firmware bug, not a pure documentation issue; the description was corrected
accordingly, no code change was made (project convention: no speculative fixes without further
verification of the impact).

**25 description corrections applied directly in `VNSD_Control_FW_Function_Tracking_new.md`** (source column
set to "Re-Audit 2026-07-14"), of which 14 in the Utility cluster (among others `LZ77_Decompress` — the third mode,
zero-run/RLE, was not documented; `Periodic_RTC_Display_Format` — description misleadingly referred to
"sensor data" instead of the RTC date; `Util_CalcHalfSizeCapped` — "max 48" suggested a result cap,
when it is actually an input limit with a fallback of 0; `RTOS_Delay_Ms` — "ISR vs. task" was imprecise,
actually scheduler-state dependent via `xTaskGetSchedulerState`) and 11 in the Miscellaneous cluster (most important
finding: `DateTime_Validate_NoNull` had a description that claimed the EXACT OPPOSITE of the code —
"with null-pointer check" instead of the correct "without").

**Cluster re-sorting proposals for "Miscellaneous" (23/23 functions could be assigned to an existing topic
— the cluster is, after this review, content-wise fully resolvable, not implemented, only
a proposal):**

| Address | Name | Proposed cluster |
|---|---|---|
| `0x08000268` | Get_ActiveIRQn | Hardware/HAL (Cortex-M system registers, only caller is the FreeRTOS port layer) |
| `0x08001814` | ProcessFirmwareUpdateCommand | OTA/Flash |
| `0x08003dd0` | CRC16_Calculate | CRC16/checksum system |
| `0x08004844` | Serial_Packet_Validate | BLE (only caller distributes only BLE_Cmd_*) |
| `0x08004888` | Device_Network_Info_Init | Network/DNS |
| `0x080049ec` | Voltage_Stability_Check | Relay/CT power control |
| `0x08004b24` | Status_Bitfield_Update | Config/EEPROM |
| `0x08004c7c` | DateTime_Validate | Hardware/HAL (RTC) |
| `0x08005230` | DateTime_Validate_NoNull | Hardware/HAL (RTC) / Config/EEPROM |
| `0x080054e0` | Retry_Until_Success_Or_Limit | OTA/Flash (QSPI polling) |
| `0x08005500` | Standby_Wakeup_Debounce | Relay/CT power control |
| `0x080068ac` | Stats_Clear_Counters | Config/EEPROM |
| `0x080071c4` | Display_Cycle_DataSources | Event log/schedule engine |
| `0x08012d8c` | Generic_StructField_Set_0x14 | Utility/Byte-Helpers (17 callers across completely different subsystems) |
| `0x08012d90` | Generic_StructField_Set_0x10 | Utility/Byte-Helpers (44 callers, same pattern) |
| `0x0801d084` | Relay_StagedTimingControl | Relay/CT power control |
| `0x08025eac` | Event_Params_Pack | MQTT |
| `0x08025eb2` | Serial_Command_Dispatch | BLE |
| `0x08026344` | DeviceInfo_BuildStatusString | Network/HTTP app layer/cloud reporting |
| `0x0802d00c` | Unused_FuncCall_Wrapper | Debug/logging |
| `0x0802d018` | TimePlan_Evaluate_Setpoint | Event log/schedule engine |
| `0x0804fe20` | Read_Serializer | Modbus/RS485 |
| `0x08050f20` | Write_Handler | Modbus/RS485 |

**Error rate of this sub-round: 2/66 naming errors (≈3%), 25/66 lines needing description work (≈38%,
of which 5 had factually misleading/wrong rather than merely missing content).** No duplicate function name
found among the 66 addresses checked.

---

### 13.43 Re-audit Network/DNS/Sockets + HTTP/HTTPS app layer + Cloud Reporting (70 functions, 2026-07-15)

All 70 functions of the three thematically related clusters `Network / DNS / Sockets (general)` (29),
`HTTP/HTTPS — App Layer` (18) and `Cloud Reporting` (23) were checked individually via `get-decompilation`
(incl. caller/callee). The focus, per the task, was on transport layer confusions
(TCP/UDP/DNS), analogous to the known Quectel error pattern from earlier rounds.

**Core finding: a generic sprintf wrapper mislabeled as "Cloud_Telemetry_JSON_Builder".**
The function `Cloud_Telemetry_JSON_Builder` (`0x080303bc`) is a pure sprintf/vsprintf wrapper
(internally calls only `printf_Format_Engine` + `sprintf_Output_Char`, with no cloud or
JSON connection whatsoever). It has **39 callers spread across practically every cluster of the firmware**: BLE GATT names
(`BLE_Set_Device_Name`), MQTT topic construction (`MQTT_Topic_Builder`), Quectel AT commands
(`Quectel_HTTP_GET_Request`, `Quectel_MQTT_Subscribe`, `Quectel_WiFi_SetSTAInfo_NoSave`,
`Quectel_SSL_Certificate_Manage`, and many others), cJSON serialization (`cJSON_PrintNumber`,
`cJSON_PrintStringPtr`), DNS/broadcast strings (`Network_BroadcastAddr_ComputeAndLog`), TCP close
(`TCP_Socket_Close`, `Cloud_Close_TCP_Connection`) as well as actual cloud reporting sites. The
name suggests a function specialized for cloud telemetry JSON, but it is in fact the
central generic string-formatting primitive of the entire firmware — comparable to the
already known `snprintf` (`0x08030388`), just without a length limit. Renaming proposal:
`sprintf` (fits the existing naming convention alongside `snprintf`).

**Follow-up finding: BLE GATT notify hex-encoder chain incorrectly filed under "Cloud_".** The chain
`Cloud_Format_And_Send_Byte` (`0x08004808`) → `Cloud_Format_And_Send_Buffer` (`0x0800b7dc`) →
`Cloud_Transmit_Byte_Blocking` (`0x0802fa58`) has **exclusively** `BLE_GATT_Notify_WithData` and
`BLE_GATT_Notify_Send` as (indirect) callers. The functions encode a byte as 2
ASCII hex characters (via the sprintf wrapper mentioned above) and send them in a blocking manner via UART
(`USART_SendData` + status poll) — pure BLE GATT notify payload transmission to the Quectel module,
no connection to cloud reporting. Presumably misfiled historically because the generic
formatting helper was incorrectly considered a "Cloud" function and the naming passed
this error on.

**Follow-up finding (already noted in advance by a parallel session in 13.41, deepened here):
`HTTP_POST_Request` is a generic Quectel AT executor, not POST-specific.**
`HTTP_POST_Request` (`0x0800bcc4`) sends an arbitrary AT command (param_1) via UART, waits for a
notify (max. 400 ticks) + delay, and checks the response for 1–2 substrings. It is called by
`Quectel_HTTP_GET_Request` (HTTP **GET**!), `Quectel_HTTP_ReadResponse` (AT+QHTTPREAD),
`Quectel_HTTP_Config_SSLCtxId` (AT+QHTTPCFG, SSL configuration) AND by the actual
POST reporting sites (`HTTP_Cloud_Reporting_Dispatcher`, `Cloud_Reporting_setVenusDReporting`).
The name "POST_Request" thus only covers part of the actual usage. Clearly
distinct from this is `HTTPS_POST_Request` (`0x08014dc0`) — a completely separate, correctly named
path that actually opens a TCP socket via CH395 Ethernet (not Quectel AT), builds an
HTTP request buffer and sends it; no confusion between the two functions
found.

**All remaining 66 functions** (Network/DNS cluster complete: `Network_ProtocolDispatcher`,
`DNS_ResourceRecord_Parse`, `DNS_Name_Decompress`, `DNS_Query_Build`/`_Send`,
`DNS_ParseResponseHeader`, `TCP_Socket_Close`, `Network_TransportDispatch`,
the `Protocol_AA_*` family, among others; HTTP/HTTPS cluster: all parsers/builders for P1 meter, EcoTracker,
StormOne meter, Economy/TOU, llhttp response parsing; Cloud Reporting: all `Cloud_Report_Fill*`,
`Cloud_Config_Apply`, `Cloud_Response_Action`, `Cloud_EdgeDetectAndWatchdog`,
`Cloud_HTTP_Response_Parser`, `Cloud_Report_URL_Builder`) were found to behave consistently with their name/description
— no transport layer confusions (TCP/UDP/DNS) found as in the Quectel cluster. A
minor thematic (not naming) note: `Protocol_AA_CommandDispatch` and the
`Protocol_AA_Set*`/`Protocol_AA_EnqueueCommand`/`Protocol_AA_RS485Forward` family are
called exclusively from the CAN bus cluster (`CAN_FrameDispatcher`) and belong,
content-wise, more to CAN bus/parallel operation than to network/sockets — the function names themselves
are correct though, only the cluster assignment in the documentation structure has grown historically.

**8 missing descriptions added** (names left unchanged, only the description column):
`HTTP_Cloud_Reporting_Dispatcher`, `HTTP_Economy_TOU_Parser`, `HTTPS_POST_Request`,
`HTTP_Response_Parse` (HTTP/HTTPS cluster); `Network_HeartbeatHandler`,
`Network_ReceiveAndDispatchData` (network cluster); `Cloud_Report_URL_Builder`,
`Cloud_Reporting_setVenusDReporting`, `Cloud_HTTP_Response_Parser`, `Cloud_Config_Apply`,
`Cloud_Response_Action` (Cloud Reporting cluster). For `HTTP_POST_Request`, `Cloud_Format_And_Send_Byte`
and `Cloud_Format_And_Send_Buffer` the description was likewise added, but with a
naming-suspicion note (see above), since name and behavior diverge here.

**Renaming proposals (to be applied after the central duplicate check, not carried out ourselves):**

| Address | Old name | New name proposal | Rationale |
|---|---|---|---|
| `0x080303bc` | Cloud_Telemetry_JSON_Builder | `sprintf` | Pure sprintf/vsprintf wrapper (printf_Format_Engine + sprintf_Output_Char), 39 callers across all clusters, no cloud/JSON connection; name fits the existing `snprintf` convention |
| `0x08004808` | Cloud_Format_And_Send_Byte | `BLE_GATT_HexByte_Send` (or similar) | Formats byte as 2 hex ASCII characters, sends via UART; only caller path is BLE GATT notify, no cloud connection |
| `0x0800b7dc` | Cloud_Format_And_Send_Buffer | `BLE_GATT_HexBuffer_Send` (or similar) | Loop over the above function, used exclusively from BLE_GATT_Notify_WithData/Send |
| `0x0802fa58` | Cloud_Transmit_Byte_Blocking | `UART_TransmitByte_Blocking` (or similar) | Generic blocking USART byte transmission with no cloud connection whatsoever; only caller is the above BLE hex-encoder chain |
| `0x0800bcc4` | HTTP_POST_Request | `Quectel_HTTP_AT_SendAndVerify` (or similar) | Generic Quectel AT executor (send command + notify/delay + substring check), used equally for GET/READ/SSL config AND POST — "POST" only covers part of the callers |

**Error rate of this round:** 5 naming suspicion cases among 70 functions checked (7.1%), of which
1 case with particularly large impact (39 callers spread across the entire firmware image). In addition
1 purely structural cluster-assignment note (Protocol_AA family, names correct) without
renaming need. No TCP/UDP/DNS transport confusions in the narrower sense found — the pattern
known from the Quectel cluster does not repeat 1:1 here, instead
generic helper functions dominate, named after their (non-representative) original use case.

---

### 13.44 Re-audit libc/standard library + fixed-point math (fp64)/dtoa + heap allocator (78 functions, 2026-07-15)

All 78 functions of the three thematically related clusters `libc / Standard Library / Memory Utilities`
(45), `Fixed-Point Math (fp64) / dtoa / Float Formatting` (28) and `Heap Allocator (internal)` (5) were
checked individually via `get-decompilation` (incl. caller/callee) and compared, for libc functions, against the known
standard C API signature, and for fp64/dtoa against the expected IEEE 754 procedure.

**Core finding 1: Two swapped libc names in the "atoi" cluster.**
- `0x080009c0` is called `atoi`, but implements a pure 3-parameter byte comparison
  (`for (i=0;i<n && s1[i]==s2[i] && s1[i]!=0; i++)`), without any number conversion — this is `strncmp`,
  not `atoi`. Callers like `CLI_FindEntryByName`, `MQTT_Topic_StringCompare`, `cJSON_Parse_Value`
  use it exactly as an equality check (`if (iVar2 == 0)`), consistent with strncmp.
- `0x08000cc6` is called `atoi_u16`, but implements exactly standard `atoi` behavior: calls
  `strtol(s,NULL,10)` and then restores errno — no 16-bit specifics whatsoever. 73
  callers (among others `HTTP_ParseContentLengthHeader`, `Meter_ExtractValue_ByKey`, `AT_Response_Parser`)
  use it generically for arbitrary int conversion.
- The two misnamings are presumably historically linked (an earlier batch presumably
  tried to assign "atoi" and in doing so targeted the wrong function, or renamed the real
  atoi function as a precaution to avoid the naming conflict).

**Core finding 2: `strncat` is actually `strchr`.**
`0x08000956` is called `strncat`, but has only 2 parameters `(ptr, char)` instead of the `(dest, src, n)`
required for strncat. The code searches linearly for the given single character in the string and returns
a pointer to it if found, otherwise NULL — exactly `strchr(s, c)`. All 11 callers (`DNS_Query_Build`,
`HTTP_URL_ExtractPath`, `MQTT_Config_ExtractStringValue`, `MQTT_JSON_Payload_EnsureBracePrefix`, among others)
pass single-byte separators like `.`, `/`, `;`, `{` — consistent with strchr, not strncat.

**Core finding 3 (largest impact): `dtoa_Float_To_String` is actually `pow()`, not a float→string
conversion.** The 2900-byte function `0x080305b8` has so far been classified as "dtoa" (double-to-ASCII),
but implements x^y for two fp64 operands: special cases for 0^y/x^0/x^1/x^-1, a sqrt
shortcut (y=0.5, calls `fp64_sqrt_WithExceptionCheck`), a squaring shortcut (y=2, calls
`dtoa_Square_Constant_A/B`), an overflow path (`fp_Set_Exception(2)` + `dtoa_Generate_Infinity`), and for
the general case a classic range reduction with table lookup + Horner polynomial evaluation
(`fp64_Polynomial_Eval`) — the standard structure of a software `pow(x,y) = exp(y·log(x))` implementation.
The **only caller** is `cJSON_Parse_Number` (JSON number parser), which calls the function as
`pow(10, exponent)` to scale the mantissa when parsing exponential notation ("1.5e10") —
not to convert a double into a string. This also means the three
helper functions `dtoa_Generate_Infinity`, `dtoa_Square_Constant_A` and `dtoa_Square_Constant_B`
are misclassified: they belong to the overflow path of `pow()`, not to a dtoa routine. The
actual "double → decimal string" functionality of the firmware still resides in
`printf_Float_To_Digits` (unchanged, name correct) — a separate, already correctly named
function. The cluster title "Fixed-Point Math (fp64) / dtoa / Float Formatting" is thus
misleading in that this cluster contains **no real dtoa implementation** — only a
pow() function mislabeled as such.

**Side note (not a naming error, but a decompilation limitation):** `fp64_div` (`0x08000fc0`) and
`fp64_cmp_ge` (`0x08000ff0`) show identical, only 9-line decompiled code in Ghidra despite
different addresses and, according to the call context, different tasks (callers each pass 4
parameters, while Ghidra only recognizes 2) — presumably ARM assembly with conditional execution (IT block)
or VFP coprocessor instructions that the simple decompilation does not resolve. The names remain
plausible according to the call context (division and comparison operations respectively at the expected
locations in `fp64_floor`, `printf_Float_To_Digits`, `cJSON_Parse_Number`), but were left only at "medium"
confidence; a complete manual disassembly check would be needed to close the
signature gap — outside the scope of this round.

**24 missing descriptions added** (names left unchanged, only the description column):
`memcpy`, `memset`, `strstr`, `strncpy`, `strlen`, `strcmp`, `memcmp`, `strtok`, `sscanf_Format_Parser`,
`printf`, `snprintf`, `putchar`, `printf_Format_Engine` (libc cluster); `fp64_add`, `fp64_sub`,
`fp64_neg`, `fp64_mul`, `int_to_fp64`, `uint_to_fp64`, `fp64_to_int`, `fp64_to_uint`, `fp64_div`,
`fp64_cmp_ge`, `fp64_abs_cmp`, `fp64_normalize`, `fp64_sqrt_WithExceptionCheck` (fp64 cluster). For
`atoi`, `strncat`, `atoi_u16`, `dtoa_Float_To_String`, `dtoa_Generate_Infinity`,
`dtoa_Square_Constant_A/B` the description column was annotated with "name wrong" or a reference to the
renaming proposal instead of a regular description.

**Renaming proposals (to be applied after the central duplicate check, not carried out ourselves):**

| Address | Old name | New name proposal | Rationale |
|---|---|---|---|
| `0x080009c0` | atoi | `strncmp` | 3 parameters (s1,s2,n), pure byte comparison until mismatch/null terminator, no number conversion — corresponds to strncmp |
| `0x08000cc6` | atoi_u16 | `atoi` | Calls strtol(s,NULL,10), restores errno — exact standard atoi behavior. Only possible once 0x080009c0 is renamed (name collision) |
| `0x08000956` | strncat | `strchr` | 2 parameters (ptr,char) instead of (dest,src,n); searches for a single character in the string, returns a pointer or NULL — corresponds to strchr(s,c) |
| `0x080305b8` | dtoa_Float_To_String | `fp64_pow` | Implements x^y (special cases 0^y/x^0/x^-1, sqrt via y=0.5, squaring via y=2, range reduction+polynomial for the general case) — standard C pow(), not a float→string conversion. Only caller cJSON_Parse_Number uses it as pow(10,exponent) |
| `0x08031350` | dtoa_Generate_Infinity | `fp64_pow_OverflowInfinity` (or similar) | Overflow return path of pow() (0x080305b8), called after fp_Set_Exception(2) — depends on the above rename |
| `0x080313b8` | dtoa_Square_Constant_A | `fp64_pow_OverflowConst_A` (or similar) | Squares an overflow constant to produce a correctly flagged +Inf in the pow() overflow path — depends on the above rename |
| `0x080313d8` | dtoa_Square_Constant_B | `fp64_pow_OverflowConst_B` (or similar) | Like 0x080313b8, second variant — depends on the above rename |

**Error rate of this round:** 7 naming errors among 78 functions checked (9.0%) — of which 3 are direct
API confusions with high caller reach (atoi↔strncmp, strncat→strchr, 90+ callers
affected in total) and 4 related misclassifications in the supposed "dtoa" area that turned out to be
parts of a `pow()` implementation. Cluster `Heap Allocator (internal)` (5/5) error-free.
Of the 28 fp64 functions, 21 were already correctly named and documented, 3 (Generate_Infinity,
Square_Constant_A/B) were thematically misclassified (name correctly describes the mechanics, but wrong
functional area/cluster), 1 (Float_To_String) completely misclassified. Of the 45 libc functions,
42 were correct, 3 faulty (atoi, atoi_u16, strncat).

### 13.45 Re-audit llhttp cluster (HTTP parser, vendor lib, 96 functions, 2026-07-15)

All 96 functions of the `llhttp — HTTP Parser (Vendor Lib)` cluster were individually checked for the first time via
`get-decompilation` (incl. caller/callee) — previously only a duplicate fix
(8 bit-test helpers in batch 19) had been done in this cluster, never an individual content check. All 51
functions without a description (confidence "-"), all 7 functions with "medium" confidence, as well as a
generous sample of central dispatcher/parser callbacks from the already documented
"high" pool were checked — a total of **75 of 96 functions** individually verified.

**Core finding: the cluster is ~100% correctly named — not a single misnaming case found.**
This clearly distinguishes this cluster from the previous re-audit rounds (7–63 errors per cluster
in sections 13.30–13.44). Reason: The names of the internal helpers (`llhttp__internal__c_or_flags*`,
`c_test_flags*`, `c_store_*`, `c_update_*`) are **not Ghidra reconstructions**, but correspond
exactly to the naming scheme that the real llhttp code generator (nodejs/llhttp) uses for its C output
— presumably they were taken over from debug strings/symbol remnants in the binary. As a result, the
initial naming of this cluster was very reliable from the outset.

**Bit-by-bit verification of the flags helpers:** All `OrFlags_*`/`c_or_flags*` functions (0x08036a26–
0x08036be8) were checked against their actual bitmasks on the flags word @offset 0x32 (or lenient flags
@0x2e) — each sets exactly the bit implied by its name (F_CHUNKED=0x01,
F_CONNECTION_KEEP_ALIVE=0x02, F_CONNECTION_CLOSE=0x04, F_CONNECTION_UPGRADE=0x08, F_UPGRADE=0x10,
F_CONTENT_LENGTH=0x20, F_SKIPBODY=0x40, F_TRAILING=0x80, F_TRANSFER_ENCODING=0x200). Likewise, all
`c_test_flags*`/`c_test_lenient_flags*` bit-test helpers were verified against their masks/shifts — the
`_Off32`/`_Off2e` suffixes assigned in batch 19 are correctly mapped to the two struct fields.

**Minor special case (no rename, only a documentation note):** `llhttp__internal__c_or_flags_4`
(`0x08036be8`) does follow the "c_or_flags" name family, but actually performs a direct
byte assignment (`*(byte*)(param_1+0x30) = 1`) instead of a bit-OR operation. Functionally equivalent
(the field @0x30 apparently holds only this one flag), but technically not a real OR operation — noted
in the description column, no rename proposal, since the name remains meaningful and consistent with
the generated llhttp family.

**Central dispatchers confirmed:** `llhttp_Parser_Execute` (0x08036bf4), at 22,470 bytes, is the huge
generated state machine (corresponds to `llhttp__internal__run` in real llhttp) and dispatches across all
the helpers mentioned above as well as all settings callbacks. `llhttp_Parse_Wrapper` (0x0803d538) is the
outer wrapper (corresponds to `llhttp__internal_execute`): caches the error status @0xc, calls
Parser_Execute and manages pause/reexec via a state-handler function pointer stored in the
parser struct @0x08. `llhttp_execute`, `llhttp_init`, `llhttp_settings_init`,
`llhttp_set_error_reason` (9 callers, all verified), `llhttp_ShouldKeepAlive`,
`llhttp_message_needs_eof` and `llhttp_errno_name` (440B switch/jump table) were each checked against their
decompilation and match their description exactly.

**Ghidra instability during the check:** 6 functions (`c_load_upgrade`, `c_test_lenient_flags`,
`c_test_flags_4`, `ClearHeaderState`, `GetHeaderState`, `MatchSequence`) repeatedly produced timeouts
on full decompilation (presumably server load from other re-audit sessions on other clusters running in parallel,
see the memory entry on the parallel campaign) — for these, signature, size and
caller context were successfully retrieved and compared against the pattern of the respective neighboring
(fully verified) sibling functions; confidence for these was set to "medium" with the note "partial verification" instead of
"high", since the actual function body could not be inspected.

**51 missing descriptions added** (names consistently confirmed, no change needed): all
`Load*`/`MulAdd*`/`OrFlags_*`/`c_or_flags*`/`c_store_*`/`c_test_flags*`/`c_test_lenient_flags*`/
`c_update_*`/`Set*`/`Clear*` helpers in the address range 0x0803686e–0x08036bdc, as well as `Parser_Execute` and
`Parse_Wrapper`. In addition, 24 already documented lines (source previously "Doku"/"Batch 20") were updated, after
individual re-verification, to source "Re-Audit 2026-07-14", of which 5 were upgraded from medium→high
(`OnChunkComplete`, `OnChunkHeader`, `OnHeaderFieldComplete`, `OnReset`, `ParserInternalInit`
— all confirmed exactly as previously documented in batch 18).

**Error rate of this round:** 0 naming errors among 75 individually checked functions (0%) — the cleanest
cluster of the entire re-audit campaign so far. No renaming proposals. The remaining 21 not
individually checked functions are exclusively already "high"-documented span/lifecycle callbacks
with an identical, repeatedly verified pattern (settings-pointer null check + conditional call) and are considered
sufficiently assured.

### 13.46 Deep dive: 0x0802f2b4 (BatteryParams struct) & 0x08008000 (Ghidra function-boundary artifact) (2026-07-15)

Two suspect cases specifically named by the user were examined and closed.

**1) `0x0802f2b4`, formerly `Telemetry_Timestamp_Get`.** The name was demonstrably wrong — the
decompilation contains no RTC/time access whatsoever:

```c
undefined4 Telemetry_Timestamp_Get(void)
{
  undefined4 uVar1 = 0;
  if (*(char *)(DAT_0802f2ec + 0xf) != '\x01') {           // offset+0xF: disable flag
    if ((int)*(short *)(DAT_0802f2ec + 2) / 10 < 1) {       // offset+2: signed short, /10
      uVar1 = (*(short *)(DAT_0802f2ec + 2) / 10 < 0) ? 3 : 1;
    } else {
      uVar1 = 2;
    }
  }
  return uVar1;
}
```

`DAT_0802f2ec` is the literal-pool copy of the BatteryParams struct base `0x20014F82` (the same base
that is referenced by, among others, `Power_Limit_Clamp`, `Inverter_Power_Setpoint_Calc`, `Inverter_Power_Setpoint_ScaleFactor_Calc`
and `Inverter_Apply_BatteryParams`, see the offset table below). The function returns a
4-state code: `0` = disabled (flag @+0xF == 1), `1` = value near zero (`0 <= value/10 < 1`), `2` = positive
(`value/10 >= 1`), `3` = negative (`value/10 < 0`).

All three callers embed the result as a single status byte in larger telemetry/report structs:

- `MQTT_Telemetry_Struct_Builder` (0x080124a4): result to `puVar1[1]` or `DAT_0801274c[2]`.
- `Cloud_Report_FillPowerFlow` (0x0801c748): result to `param_1+0x23`, directly next to a
  scaling/percent byte (`param_1+0x22`, from an `fp64_mul` computation) and before several
  difference/sum fields made of four percentage values (offsets 0xC/0x12/0x18/0x1E of the same struct family,
  clamped to 0 for values ≥101 — a classic SOC/percent validation pattern). The function name
  "FillPowerFlow" and the neighboring fields support a charge/discharge/power-flow semantic.
- `BLE_RuntimeInfo_Builder` (0x0800af78): result to `DAT_0800b388+4`, one byte in a generic
  runtime-info struct.

**Assessment:** The proof that the function reads **no timestamp** is complete
(no RTC access, no tick counter, no time library function in the call path). The exact physical
meaning of offset+2 (presumably a net power or current quantity in 0.1 units, whose sign
indicates the charge/discharge direction) could not be proven conclusively for lack of a found **write access** to exactly offset+2
— `Inverter_Apply_BatteryParams` (the most likely setter for the struct)
demonstrably does *not* write to offset+2, but only reads other offsets (0x8/0xE/0x14/0x1A on a
second struct instance) for dirty-flag purposes — the actual writer of offset+2 was not located.
The new name therefore conservatively describes **what the function does** (derive a status code from a BatteryParams field),
not an unproven physical unit. Renamed to **`BatteryParams_PowerFlowState_Get`**,
confidence `medium` (behavior 100 % proven, exact physical meaning of offset+2 not). Ghidra rename
and plate comment with full justification applied.

**2) `0x08008000`, formerly `BLE_Command_Handler` — confirmed Ghidra function-boundary artifact.**
Root-cause analysis confirmed the suspicion: the function was a fully overlapping ghost definition
inside the real function body of `BLE_Recv_Cmd_Dispatcher` (`0x08007f58`–`0x0800a26f`, 6732 bytes).
The bogus function claimed `0x08008000`–`0x0800a277` (lies completely inside plus 8 bytes beyond the real
end). Evidence: (a) the first instruction `lsls r5,r3,#0xb` is not a valid function prologue; (b) the
decompilation body contained a `pop {pc}` at `0x08007f76` — an address *below* the claimed
entry point, which is impossible for a real function; (c) all 5 incoming "references" were
`PARAM` type (data-constant hits), not real `CALL`/`JUMP` references. The function definition was
removed via `FunctionManager.removeFunction()`, a plate comment with the justification was placed
at `0x08008000`, and `BLE_Recv_Cmd_Dispatcher` was verified as unchanged after the cleanup (still 6732 bytes, correctly
bounded). The corresponding row was removed from `VNSD_Control_FW_Function_Tracking_new.md`,
cluster header "BLE / GATT" corrected from (36) to (35). Live function count in the program: 1623 → 1622.

---

## Appendix A: Scan context (device state during the scan)

```
Date:           May 2026
Device:         Marstek Venus D (VNSD-0)
Battery packs:  2
SOC Pack1:      10.9% (raw 109, scale 0.1)
SOC Pack2:      12.0% (raw 120, scale 0.1)
Voltage Pack1:  50.12V
Voltage Pack2:  51.25V
AC grid:        239.8V / 49.9Hz
AC current:     −4.98A (feed-in)
PV:             NOT connected (MPPT = idle ~9.9V)
Mode:           Anti-feed-in (work_mode=1)
RS485:          Control ACTIVE (42000=0x55AA)
Backup/UPS:     Active (41200=1)
```

## Appendix B: Tools & dependencies

```bash
# Python dependencies
pip3 install pymodbus pandas

# Ghidra (macOS)
brew install ghidra
brew install --cask temurin@21  # Java 21

# Ghidra scripts: Jython 2.7 (built in, no extra install)
# Important: # -*- coding: utf-8 -*- in line 1, no umlauts!
```

---

*Created: May 2026, updated July 2026 | Firmware: VNSD-0 v149.2 | Device: Marstek Venus D*  
*Analysis tools: Ghidra + PyGhidra + ReVa-MCP + pymodbus + marstek-fw-checker*  
*v1: Firmware 149.2 analyzed, purely Venus-D-focused, OTA status updated*

---

## Appendix C: Extended Ghidra analysis (supplement)

### C.1 Newly found SRAM variables

From `FUN_08004bd8` (max power initialization):

| SRAM address | Variable | Meaning |
|---|---|---|
| `0x20000136` | `DAT_20000136` | Hardware version (version_num) |
| `0x2000012f` | `DAT_2000012f` | Max charge power (dynamic, version-dependent) |
| `0x2000012d` | `DAT_2000012d` | Max discharge power |
| `0x20000260` | `DAT_20000260` | Current power setpoint (EMS) |
| `0x2000026f` | `DAT_2000026f` | RS485 table index |
| `0x20000270` | `DAT_20000270` | Modbus slave ID (low byte) |
| `0x20000271` | `DAT_20000271` | Modbus slave ID (high byte) |
| `0x200002b2` | `DAT_200002b2` | Timer/hysteresis counter |
| `0x20000269` | `DAT_20000269` | EMS control state |

Hardware version → max power mapping:
```
version_num = 0  →  max_power = 2500 W  (standard Venus D)
version_num = 1  →  max_power =  800 W  (smaller variant)
version_num = 2  →  max_power =  600 W  (smallest variant)
```

### C.2 Second descriptor table discovered

In `FUN_08004b20`, an identical access pattern to a second table was found:

```c
// Main table (TCP Modbus):
(&DAT_20000354)[uVar4 * 6]          // base: 0x20000354, 246 entries

// Second table (RS485/EMS):
*(ushort *)(DAT_2000026f * 6 + 0x600172b8)  // base: 0x600172b8, ~96 entries
```

| Property | TCP table | RS485 table |
|---|---|---|
| Base | `0x20000354` (SRAM) | `0x600172b8` (ext. RAM / CH395Q) |
| Entries | 246 (0xF6) | ~96 (0x5F+1) |
| Stride | 12 bytes | 12 bytes (identical) |
| Purpose | Modbus TCP read registers | RS485/EMS internal registers |

### C.3 Final analysis results

**Fully decompiled functions (149.2 equivalents):**

| Function (149.2) | Role | Status |
|---|---|---|
| `FUN_0801e43c` | Modbus_Dispatcher | ✅ Complete |
| `FUN_0801eaa4` | FC03_ReadHandler | ✅ Complete |
| `FUN_0804fe20` | Read_Serializer | ✅ Complete |
| `FUN_08050f20` | Write_Handler | ✅ Complete |

**Descriptor init: conclusion**

The init function for the descriptor table (`0x20000354`) cannot be found by static analysis
— the register numbers are **computed** at runtime (not stored as an array
or immediates). Four independent checks prove this (see section 0.3).

Possible approaches:
1. **Batch scan** (recommended): `scan_modbus_batch.py` → 0-65535 in ~52 min
2. **JTAG/SWD**: read 2952 bytes from `0x20000354` on a running device
3. **Unicorn emulation**: emulate the binary until SRAM is initialized

---

## Appendix D: Full scan — 413 registers (May 2026)

### D.1 Scan result overview

```
Scan date:         May 2026
Method:            Raw TCP socket, batch=32, direct addressing (PDU addr = register no.)
Scanner:           scan_modbus_batch.py v7
Duration:          2:30h (batch=32, fallback to single reads, skip empty blocks)
Found:             413 registers (before: 130)
Address space:     0 – 65535 (fully scanned ✅)

COMPLETED:
  Registers 0–29999:     EMPTY — not a single response
  Registers 30000–39999: 89 read registers confirmed
  Registers 40000–49999: 41 write registers confirmed
  Registers 50000–65535: EMPTY — not a single response
```

> **Significance:** The Modbus address space is thus fully mapped.
> There are no further unknown ranges. All 413 registers found
> are located exclusively between 30000 and 49999.

### D.2 Newly discovered register clusters

| Cluster | Registers | Content |
|---|---|---|
| **NEW** | 30000–30010 | New power/status registers |
| Known | 30020–30040 | MPPT / PV |
| **NEW** | 30100–30110 | Pack 1 BMS (extended) |
| **NEW** | 30200–30214 | Firmware versions (extended) |
| **NEW** | 30400–30403 | IP addresses (device + gateway) |
| Known | 31000–31009 | Device name |
| Known | 33000–33011 | Energy counters |
| **NEW** | 34000–34033 | Pack 1 (detailed, incl. individual temps) |
| **NEW** | 34100–34133 | Pack 2 (detailed) |
| **NEW** | 34200–34233 | Pack 3 slot (empty = not installed) |
| **NEW** | 34300–34333 | Pack 4 slot (empty) |
| **NEW** | 34400–34433 | Pack 5 slot (empty) |
| **NEW** | 34500–34533 | Pack 6 slot (empty) |
| **NEW** | 34600–34633 | Pack 7 slot (empty) |
| **NEW** | 35000–35012 | Temperatures (extended) |
| **NEW** | 35110–35112 | Power limits (new) |
| Known | 36000–36103 | Alarm / fault |
| **NEW** | 37000–37024 | Combined grid + sensor values |

### D.3 Newly decoded registers (full list)

#### New registers 30000–30010

| Register | Raw value | Interpreted | Status |
|---|---|---|---|
| 30000 | 509 | ? (maybe grid_power or combined_pv_power) | Unknown |
| 30001 | −10 | battery_power = −10 W (discharge) | Known |
| 30002 | 179 | ? | Unknown |
| 30003 | 181 | ? | Unknown |
| 30004 | 2410 | 241.0 V (maybe ac_voltage 2nd measurement) | Plausible |
| 30005 | 2 | backup_voltage (backup output voltage, scale 0.1V) — see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md, confidence: high | Determined |
| 30006 | 0 | ac_power = 0 W | Known |
| 30007 | 0 | backup_power (backup output power, W) — see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md, confidence: high | Determined |
| 30010 | 0 | ? | Unknown |
| 30028 | 513 | 0x0201 = ? (same value as 37022) | Unknown |
| 30029 | 65535 | battery_power_factor (power index, not linear, int16) — see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md, confidence: medium | Determined |
| 30036 | 219 | hardware_version (constant 219 across all scans/2+h long-term, no sensor quantity) — see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md, confidence: medium | Determined |

#### New network registers (30400–30403)

```
30400–30401:  device_ip   = 192.168.x.x  (the device's own IP)
30402–30403:  gateway_ip  = 192.168.x.1
```

Encoding: each register = 2 bytes of the IPv4 address (big-endian)
```python
ip_hi = raw_uint16 >> 8    # first octet
ip_lo = raw_uint16 & 0xFF  # second octet
```

#### Extended pack 1 registers (34000–34017)

| Register | Raw value | Interpreted |
|---|---|---|
| 34000 | 5114 | pack1_voltage = 51.14 V |
| 34001 | 0 | pack1_current = 0.0 A |
| 34002 | 146 | **pack1_soc = 14.6 %** (scale 0.1!) |
| 34003 | 19 | pack1_cycle_count = 19 |
| 34004 | 3 | packX_charge_status (0=idle, 3=actively charging/discharging) — see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md, confidence: high |
| 34005 | 3199 | pack1_max_cell_voltage = 3.199 V |
| 34006 | 3195 | pack1_min_cell_voltage = 3.195 V |
| 34010 | 116 | **pack1_bms_version = 116** (not temperature — raw 116 = BMS version like reg 30204!) |
| 34011 | 261 | **pack1_temp_max = 26.1 °C** |
| 34012 | 198 | pack1_temp_? = 19.8 °C (NEW) |
| 34013 | 193 | **pack1_temp_min = 19.3 °C** (NEW) |
| 34014 | 194 | pack1_temp_? = 19.4 °C (NEW) |
| 34015 | 192 | pack1_temp_? = 19.2 °C (NEW) |
| 34016 | 191 | pack1_temp_? = 19.1 °C (NEW) |

#### Battery pack slots (firmware supports 7 packs!)

> **Note:** This scan dates from May 2026 with 2 packs. The device has since
> **6 battery packs** — the 149.2 live scan will update the table.

| Pack | Base register | Voltage | SOC | Cycles | Status |
|---|---|---|---|---|---|
| Pack 1 | 34000 | 51.14 V | 14.6 % | 19 | ✅ Installed |
| Pack 2 | 34100 | 50.24 V | 10.9 % | 17 | ✅ Installed |
| Pack 3 | 34200 | — | — | — | ✅ Installed (since expansion) |
| Pack 4 | 34300 | — | — | — | ✅ Installed (since expansion) |
| Pack 5 | 34400 | — | — | — | ✅ Installed (since expansion) |
| Pack 6 | 34500 | — | — | — | ✅ Installed (since expansion) |
| Pack 7 | 34600 | — | — | — | ❌ Empty |

#### Registers 37000–37024 (new, combined system values)

| Register | Raw value | Interpreted |
|---|---|---|
| 37000 | 1 | system_status = 1 |
| 37004 | 0 | grid_power_setpoint = 0 W (setpoint/control value, ≈30006, reacts ~1 scan cycle faster — see Modbus_RS485_TCP/VNSD_Register_Hypotheses_Analysis.md) |
| 37005 | 14 | ? (1.4) |
| 37006 | 192 | ? (19.2 °C or 192 W?) |
| 37007 | 3198 | **max_cell_v_all_packs = 3.198 V** |
| 37008 | 3195 | **min_cell_v_all_packs = 3.195 V** |
| 37012 | 116 | **bms_version = 116** (not temperature — same value as 30204 and 34010/34110) |
| 37016 | 2416 | ac_voltage = 241.6 V |
| 37022 | 513 | 0x0201 = ? (protocol version 2.01?) |

#### Registers 35110–35112 (new)

| Register | Raw value | Probable meaning |
|---|---|---|
| 35110 | 576 | max_charge_power_current = 576 W? |
| 35111 | 500 | rated_charge_power = 500 W? |
| 35112 | 500 | rated_discharge_power = 500 W? |

### D.4 Device state at the time of the scan

```
Pack 1:  51.14 V,  SOC 14.6%,  19 cycles,  BMS version 116  (reg 34010 — NOT temperature!)
Pack 2:  50.24 V,  SOC 10.9%,  17 cycles,  BMS version 116  (reg 34110 — NOT temperature!)
Grid:    241.6 V,  current 0 A (no import/export)
Mode:    Anti-feed-in (work_mode=1)
RS485:   Control active (0x55AA)
Backup:  Active (=1)
Alarm:   0 (no alarm)
Fault:   0x0963 (fault_bits, not 0!)
```

> **⚠️ Correction:** Reg 34010 (pack 1) and 34110 (pack 2) contain **BMS firmware version 116**,
> identical to reg 30204. The raw value 116 was mistakenly interpreted as 11.6 °C (scale 0.1)
> — a value that seemed plausible for the time of year.
> Actual pack temperatures are found in reg 34011–34016 (pack 1) and 34111 (pack 2) respectively.

### D.5 Technical insights

**Addressing:** Marstek uses **direct addressing** — PDU address = user register number (no offset, no -1). Proof: TCPRouter.c compares the raw PDU address directly with `descriptor.base_addr` without any correction. pymodbus: `read_holding_registers(30000)` → reads register 30000.

**Maximum batch size:** 32 registers per FC03 request.
With batch > 32 the device returns no response (timeout, no exception code).

**Gaps in the register map:** Many addresses are unoccupied.
A batch request across a gap → exception code 2.
Solution: fall back to single reads on exception.

**Pack expandability:** The firmware supports up to 7 battery packs
(register blocks 34000, 34100, 34200, ..., 34600).
The BMS firmware internally allows up to **9 packs** (`param_1 < 10` in the CAN sender),
Marstek officially limits to 6. Registers for pack 7 (34600) exist as placeholders.
Unconnected packs respond with 0 for all registers.

**Fault bits 0x0963:** From the BMS FW analysis (v117.7) the protect bitmasks are
fully decoded. Protect1 has 12 bits (cell OVP/UVP, overcurrent L2,
temperature protection, hardware SCP), Protect2 has 5 bits (NTC fault, comm error,
error lock). Full bit definitions: see `BMS/VNSD_BMS_FW_Analysis_v117.7.md`, section 5.2/5.3.

---

*Created: May 2026 | Firmware: VNSD-0 v149.2 / Micro v116 / BMS v117.7 | Device: Marstek Venus D*  
*Analysis tools: Ghidra + ReVa MCP + Python 3 + pymodbus*  
*v1: Modbus handler, descriptor table, cloud API*  
*v2: Telemetry bridge (48-byte block, 20 fields, SRAM 0x20014E90)*  
*v3: System architecture, DC bus topology, PV behavior, pack rotation, cross-FW references*
