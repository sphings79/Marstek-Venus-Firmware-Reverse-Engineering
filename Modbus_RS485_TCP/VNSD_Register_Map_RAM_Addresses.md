# VNSD register map – RAM addresses per firmware version (and write effects)

> 🇩🇪 Deutsche Version: [VNSD_Register_Map_RAM_Addresses_de.md](VNSD_Register_Map_RAM_Addresses_de.md)

`VNSD_Register_Map_Final_generated.csv` carries the RAM address behind every Modbus register ("source pointer").
These addresses move with every firmware build (RAM layout changes: v149.2 resized the BMS pack record, v151 moved the
whole block by +0x2800), so the single former column `source_ptr_v150` was extended to all VNSD Control versions:

| Column | Firmware |
|---|---|
| `source_ptr_v147` | Control 147 |
| `source_ptr_v149` | Control 149 |
| `source_ptr_v1492` | Control 149.2 (beta) |
| `source_ptr_v150` | Control 150 (unchanged values) |
| `source_ptr_v151` | Control 151 |
| `source_ptr_basis` | how the addresses of that row were obtained |
| `status_D` | functional status on the Venus D (register resolution 2026-10-01): **FUNCTIONAL** = has the documented effect · **CONDITIONAL** = only in a special mode / after another write · **NO_OP** = accepted but no effect · **FUNCTIONLESS** = target exists but never acts (filtered, never read, no receiver) · **UNKNOWN** = target found, meaning not provable |
| `ziel_D` | target of the register (device: what it reaches, or `none`) |
| `leseeffekt_D` | effect/behaviour of a **read** (e.g. exception 2, stalled TCP connection) |
| `befund_D` | finding, with verification notes |
| `gefahr_D` | danger rating and reason (`none` / `low` / `medium` / `high`) |
| `suite_empfehlung_D` | recommendation for the Marstek Modbus Suite integration (e.g. do not expose) |
| `befund_konfidenz_D` | confidence of the finding (`high` / `medium`) |
| `schreibeffekt` | (German) what a **write** to this register does: accepted values, side effects (EEPROM, CAN frame to the inverter/BMS, Wi-Fi re-init, reboot/factory reset, mode change), exception cases, read behaviour; filled for all 170 writable registers (40000+), based on the v151 `Write_Handler` |

Cell values: `0x2001…` = address; `—` = register does not exist in that version (e.g. 38000–38014 before v149);
empty = not determinable (see below).

## How the addresses were obtained
* **`descriptor`** (246 registers 30000–38014, 231 in v147): decoded directly from the Modbus descriptor table in the
  initialized `.data` image of each firmware (12-byte entries: register, source pointer, type, scale, count). For v149.2,
  v150 and v151 this matches the existing decoded tables (`VNSD_Descriptor_Table_Decoded_*.csv`) 246/246; the v150 column is
  identical to the previous column.
* **`symbol+offset`** (write/command registers 40000+, 41xxx, 42xxx, 43xxx, 44xxx, 45xxx and the credential buffers): the v150
  address was resolved to the named global (plus byte offset) in the v150 Ghidra project and looked up under the same name in the other
  versions. The method was validated against the descriptor entries: 245 of 246 entries resolved correctly for v149.2 and v151 (one entry has no symbol), 235 of 246 for v149
  and 227 of 231 for v147 (the rest has no unique symbol name there); no wrong result.
* **`symbol(near)+offset`**: as above, but the v150 address lies inside an array/struct that only has a sub-symbol; slightly less certain.
* **`table-base+offset`** (46500–46544): the 120-byte table that v150/v149.2/v149 call `g_powerLimitTable` is `g_canScheduleRegCache`
  (also 120 bytes) in v151. The name `g_powerLimitTable` is a stale label from the old analysis and should be read as the CAN schedule cache.
  In v147 the table could not be located (empty cells).
* Sub-registers of multi-register values (`teil_von` set, e.g. MAC characters) have no own pointer in any column (as before).

## Caveats
* **Re-check 2026-10-01 (v151 code, starting from the register number):** the associations of all write/command registers (40000+) and of
  the descriptor registers with placeholder names were verified against the v151 `Write_Handler` (`0x080570e0`) and the descriptor table. Pointers
  were correct; 113 register **names** were not and were corrected in the CSV (old name kept in `alt_name` as `frueher: …`, reason in `notiz` as
  `KORR 2026-10-01`). Main findings: 41600–41631 is the write-only Wi-Fi **password** buffer (not `schedule_power`); 46500–46544 is the CAN
  schedule register cache (`g_canScheduleRegCache`), not an EEPROM struct / power-limit table; 40000 is the burn-in test step (not an RS485 unlock);
  42000 = RS485 control mode (shares the byte `g_userWorkMode` with 43000); 37002/37003 are not max charge/discharge power (that is 44002/44003);
  30110/32114 = `g_encodeFlag` (key/pairing sequence status); several 45xxx command names were wrong (see `VNSD_Command_Register_45000_Block_v150.md`, sections 7 and 8).
  The `notiz` texts were English-only for the corrections and the old "SRAM-Ziel" addresses in them were replaced by the v150 column value.
  After the register resolution of 2026-10-01 only 1 register (37021) remains `UNKNOWN`; 45021, 45028, 45030, 30035 and 47001 are resolved (47001 is now in the CSV). In that pass 170 rows had their names corrected (old name kept in `alt_name` as `frueher: <old>`); the details are in `VNSD_Unclear_Registers_Resolution_2026-10-01.md`.
* `v_*` columns (live scan values) and the notes still refer to firmware 150.
* `VNSD_Register_Map_v150_annotiert.csv` and `VNSD_Cross_Reference_Modbus_SRAM_BLE_v150.md` still contain v150 addresses only.

## Register resolution columns (2026-10-01)
The columns `status_D`, `ziel_D`, `leseeffekt_D`, `befund_D`, `gefahr_D`, `suite_empfehlung_D` and `befund_konfidenz_D` hold the result of the complete code analysis of the formerly unclear registers (179 registers filled: FUNCTIONLESS 71, FUNCTIONAL 60, CONDITIONAL 39, NO_OP 8, UNKNOWN 1). Meaning of the status values: see the `status_D` row in the table at the top. Details: [VNSD_Unclear_Registers_Resolution_2026-10-01.md](VNSD_Unclear_Registers_Resolution_2026-10-01.md).

New row **41001** (`unhandled_register_41001`, formerly labelled `factory_reset (Suite button)`): **not implemented on the Venus D** – no 41001 case exists in any Control version, so the register answers with exception 2 (the identically numbered Micro register lives in a separate register space). A real factory reset is 41000 = `0xAA11`.

## New register 47001
Register 47001 (`0xB799`, `can_cat0x18_command`) exists only from v151 (earlier versions answer exception 2); it is included in the CSV with `—` in all pointer columns.
Note on dangerous writes (see `schreibeffekt`): 45028 triggers a BMS factory reset + reboot, 41000 = `0xAA11` a factory reset of the Control unit, 41000 = `0x55AA` a reboot. **Correction 2026-10-01:** 45001 does **not** trigger a BMS reset (it is a Micro EEPROM self-test); 45028 does (factory reset + reboot of all packs), and 45027 reboots the BMS master pack.
