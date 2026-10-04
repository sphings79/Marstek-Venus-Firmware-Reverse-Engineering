# SWD/JTAG Dump Plan — Closing the Last Wiring Questions

> 🇩🇪 Deutsche Version: [VNSD_SWD_JTAG_Dump_Plan_de.md](VNSD_SWD_JTAG_Dump_Plan_de.md)

**Date:** 16 August 2026
**Purpose:** Everything functional (handlers, tasks, field names, descriptor formats) has been reconstructed statically from the
OTA images. What remains open is only the **wiring/registration layer** and a few
tables that are either built in SRAM at runtime or reside in **non-distributed flash sectors**.
Both can be closed with device access — usually even without a complete flash dump.

> **Key statement:** For the *table contents*, a **live RAM read** is enough (device running, halt, read).
> A full **flash dump** is only needed for the *static code/strings* in the missing sectors.

---

## 1. What exactly is missing (target inventory)

### Control MCU (VNSD-0, Cortex-M, image 0x08000000–0x0805EFFF)

| Region | Address | Size | Type | Closes question |
|---|---|---|---|---|
| **0xAA handler table** | SRAM `0x2000018c` | 17×8 = 136 B | Runtime RAM | Which CAN group slot calls which handler (registrar) |
| **Descriptor table** | SRAM `0x20000354` | 246×12 = 2952 B | Runtime RAM | All register→SRAM source pointers + type/scale, without a live scan |
| **Missing flash tail** | `0x0805F000–0x08064000` | ~20 KB | Flash (sector 7) | Cloud format strings, init/registration code, shell command match table `0x08060EF8` |

### Micro MCU (VNS inverter, image 0x08000000–0x0801C3FF)

| Region | Address | Size | Type | Closes question |
|---|---|---|---|---|
| **Telemetry descriptor** | SRAM `0x200004C0` | ≤70×12 = 840 B | Runtime RAM | Micro register→SRAM source pointers (RS485/Modbus interface) |
| **0x10000000 region** | `0x10000000–0x10020000` | ~128 KB | mem-mapped | Descriptor **builder** code + config tables (not in the .bin) |

> **Important:** The register *meanings* are already clarified (via handlers, BMS `CAN_TX_PerPack_10Msgs`
> and debug prints). These dumps only provide the **wiring** (slot→handler, reg→source pointer) and the
> static match/string tables.

---

## 2. Two approaches — the easy one first

### Approach A (recommended first): Live RAM read of the runtime tables
No flash dump needed, no read-out protection problems (RAM is always readable when debug access is available).
Let the device boot normally, halt it with the debugger (`halt`), then read out the three SRAM tables:

- Control `0x2000018c` (136 B) → 17 entries `{u16 group_id, u16 pad, u32 handler_ptr}`
- Control `0x20000354` (2952 B) → 246 entries `{u16 reg, u16 pad?, u32 src_ptr, u8 type, u8 elem, u8 scale, u8 count}` (12 B; format see `Modbus_RS485_TCP/VNSD_Descriptor_Table_Unpack_Format.md`)
- Micro `0x200004C0` (840 B) → 70 entries `{u16 reg, u16 pad, u32 src_ptr, u8 type, u8 elem, u8 scale, u8 count}`

**This alone closes almost all open wiring questions.** Only when the *static* sectors
(cloud strings, shell command match values) are needed is Approach B necessary.

### Approach B: Full flash dump of the missing sectors
For the Control tail `0x0805F000+` and the Micro `0x10000000` region.

---

## 3. Prerequisites (hardware & software)

**Debug probe:** ST-Link V2/V3 (cheap, ideal for STM32) or SEGGER J-Link.

**SWD connection (4–5 wires)** to the respective MCU debug pins:
- `SWDIO`, `SWCLK`, `GND`, `VDD` (as reference only, do NOT feed power if the board is powered), optionally `NRST`.
- Look for debug pads on the PCB (often a row of 4 next to the MCU) or tap directly at the MCU pins.
- **Board must be powered** (own supply); probe only SWDIO/SWCLK/GND + VDD sense.

**Software (pick one):**
- **OpenOCD** (universal) — examples below.
- **pyOCD** (`pip install pyocd`) — convenient for STM32.
- **ST-Link Utility / STM32CubeProgrammer** (GUI, Windows/Linux/Mac).

### ⚠️ STM32 read-out protection (RDP) — check FIRST!
An STM32 can **completely lock** reading of the internal flash via the RDP level.
- **RDP level 0** = open, flash readable. **Level 1** = flash read via debugger locked (RAM still readable!).
  **Level 2** = debug completely dead.
- **Check** (option bytes / `FLASH_OPTR`): e.g. OpenOCD `stm32f4x option_read 0` or in CubeProgrammer in the Option Bytes tab.
- 🚫 **NEVER** try to lower RDP from level 1 to 0 "in order to read" — that **erases the entire flash** (mass erase). Only read, change nothing in the option bytes.
- If RDP≥1: **Approach A (RAM read) still works** — RAM is not RDP-protected. Use that.

---

## 4. Concrete commands (OpenOCD)

### 4.1 Connect & identify the chip
```bash
# Control (presumably STM32F4, 512 KB) — adjust target cfg if necessary
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg
# in a second terminal:
telnet localhost 4444
```
```
> reset halt
> mdw 0xE0042000 1        ;# DBGMCU_IDCODE -> verify the exact part
> stm32f4x option_read 0  ;# check RDP/option bytes (RDP must be 0xAA/level 0)
```
> For the **Micro** analogously with `target/stm32f3x.cfg` (or the part confirmed via IDCODE).

### 4.2 Approach A — read runtime tables from RAM
```
> halt
;# Control 0xAA handler table (136 B)
> dump_image ctrl_aa_table_2000018c.bin 0x2000018c 136
;# Control descriptor table (2952 B)
> dump_image ctrl_descriptor_20000354.bin 0x20000354 2952
;# Micro descriptor table (840 B) — on the Micro probe
> dump_image micro_descriptor_200004c0.bin 0x200004C0 840
```
> Tip: take several snapshots during operation (charging/discharging) to distinguish dynamic from static fields.

### 4.3 Approach B — dump the missing flash sectors
```
;# Control: back up the complete flash 0x08000000..0x08080000 (512 KB)
> dump_image ctrl_flash_full.bin 0x08000000 0x80000
;# ...or specifically only the missing tail:
> dump_image ctrl_tail_0805F000.bin 0x0805F000 0x9000     ;# up to 0x08068000

;# Micro: internal flash + the 0x10000000 region
> dump_image micro_flash_full.bin 0x08000000 0x40000
> dump_image micro_region_10000000.bin 0x10000000 0x20000  ;# 128 KB
```
> If `0x10000000` is not directly readable (no mem-mapping while halted): check whether it is CCM RAM
> (then readable after boot) or external QSPI/FMC (then let the controller initialize first — boot the device
> normally, THEN halt and read).

### pyOCD equivalent (if preferred)
```bash
pyocd cmd -t stm32f429xi        # adjust chip
# in the pyOCD shell:
savemem 0x2000018c 136 ctrl_aa_table.bin
savemem 0x20000354 2952 ctrl_descriptor.bin
savemem 0x0805F000 0x9000 ctrl_tail.bin
```

---

## 5. Importing into Ghidra

### 5.1 Add SRAM tables as an overlay block (closes the wiring)
So that the previously "unmapped" SRAM addresses get real data:
1. Ghidra → **Window ▸ Memory Map** ▸ **+ (Add Block)**
   - Name `sram_live`, start `0x20000000`, suitable length (e.g. 0x20000 = 128 KB), **Initialized: From File Import** → place the RAM dumps at the correct offset address.
   - More practical: one block per table (`aa_table` @0x2000018c, `descriptor` @0x20000354) with the respective `.bin` as content.
2. The 17 `handler_ptr` values in the 0xAA table are now readable flash pointers →
   **right-click ▸ Data ▸ Pointer**; Ghidra links them to `CAN_Store_PerPack_Grp2x_...` →
   **slot↔handler mapping finally evidenced** (instead of inferred).
3. Parse the descriptor table 0x20000354 with `marstek_descriptor_unpack.py` (local, not in the repo) or the 12-B format →
   complete register→source pointer list **without** a live scan.

### 5.2 Append the missing flash tail to the existing program
1. Memory Map ▸ Add Block: name `flash_tail`, start `0x0805F000`, **Initialized: From File Import** →
   `ctrl_tail_0805F000.bin`.
2. **Run auto-analysis again** (Analysis ▸ Auto Analyze) — now:
   - the cloud `snprintf` format strings are resolved (cloud JSON field names final),
   - the shell command match table `0x08060EF8` becomes visible → command IDs of the tool shell,
   - the previously missing init/registration code is disassembled → the `callerCount 0` functions
     (tasks/callbacks) get their callers/`xTaskCreate` registration.
3. For the **Micro**: attach `micro_region_10000000.bin` as a block @0x10000000, auto-analyze →
   the descriptor **builder** becomes visible (writes 0x200004C0).

> **Alternatively (no overlay import):** Import a second Ghidra program from the full dump
> (base 0x08000000) and take over the names from the OTA project via ReVa diff/markup transfer.

---

## 6. What each dump concretely closes (checklist)

- [ ] **Control 0x2000018c (RAM)** → registrar/slot→handler of the per-pack CAN table (last §13 remainder).
- [ ] **Control 0x20000354 (RAM)** → complete descriptor map without a live scan; verifies `all_register.csv`.
- [ ] **Control tail 0x0805F000+ (flash)** → cloud JSON field names, shell command match values (VNSD_Shell.md §8.1), `callerCount 0` callers (Serial/UART/Modbus/CH395 tasks).
- [ ] **Micro 0x200004C0 (RAM)** → Micro register→source pointers.
- [ ] **Micro 0x10000000 (mem)** → descriptor builder + config tables.

---

## 7. Safety/precautionary notes

- **Read only.** No `flash write`/`erase`/option byte write operations. An accidental
  mass erase (e.g. RDP downgrade) leaves the device empty.
- **Backup first:** right after establishing the connection, take a full flash dump (if RDP=0) before
  anything else happens.
- **Voltage:** probe VDD only as sense; power the board through its own power supply, no double feeding.
- **Live reads** are uncritical and the fastest way to the tables — start with that.

---

*References: `Modbus_RS485_TCP/VNSD_Descriptor_Open_Registers_Ghidra_Finding.md` (§11–§13),
`Modbus_RS485_TCP/VNSD_Descriptor_Table_Unpack_Format.md`, `Methodik_und_Meta/Docs_Audit_Open_Items_2026-08-16.md`,
`Methodik_und_Meta/VNSD_Shell.md` (§8).*
