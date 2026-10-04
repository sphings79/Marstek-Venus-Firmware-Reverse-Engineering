# Documentation audit: "blocked" statements & open items

> 🇩🇪 Deutsche Version: [Doku_Audit_Offene_Punkte_2026-08-16_de.md](Doku_Audit_Offene_Punkte_2026-08-16_de.md)

**Date:** 16 August 2026
**Occasion:** Systematic review of the entire project documentation (37 MD files) for statements of the
form "not in the code / not in the bin / not possible / generated at runtime / only via SWD dump".
**Guiding thesis (confirmed):** Such statements were repeatedly too pessimistic. What runs on the device
is almost always located somewhere in flash — you just have to search thoroughly enough (and in the right image).

**Track record of this session:**
- "Pack field names only in the missing flash tail / only via SWD" → **refuted**: names are in the
  Control AND BMS image; debug function `BMS_Debug_PerPack_Detail_Print` uncovered after
  disassembly. (VNSD_Descriptor_Offene_Register §12)
- "0xAA handler table 0x2000018c can only be resolved via live RAM dump" → **functionally refuted**:
  all 8 per-pack store handlers found + named, registers mapped byte-accurately. (§13)

---

## Categories

- ✅ **REFUTED** — already resolved in this session; source documentation is outdated and should be corrected.
- 🔍 **RE-INVESTIGATE** — statement probably too pessimistic; attackable with the new techniques (disassembly
  runs of hidden functions, findBytes, cross-FW reconciliation).
- ⛔ **DATA-LIMIT** — bytes physically in NO available image (genuine availability blocker);
  but each case must be re-checked as to whether the information is located elsewhere (neighboring FW, live output).

---

## Findings (file : line)

### ✅ REFUTED — source documentation outdated, correction needed

| Location | Statement | Reality (2026-08-16) |
|---|---|---|
| `VNSD_Control_FW_Analyse_app_1492…md:1806` | "0xAA command table (RAM 0x2000018c) … no write xref … only via live RAM dump or CAN sniffing" | 8 handlers @0x0802f764.. found+named; groups→offsets→registers completely mapped (BMS CAN TX verified). Only the raw slot order in RAM remains open, cosmetically. |
| `…1492…md:2503 / 2520 / 2526` | "17-entry handler table … cannot be determined statically" | ditto — determined functionally. |
| `VNSD_Descriptor_Offene_Register…md:167/241/246/305/344-350/376` (sect. 1,6,7e,9) | "Pack field names in the missing flash tail, only via SWD dump" | **refuted** — see §12/§13 of the same file. Old sections need a correction note. |
| `…1492…md:4360` | "Register numbers computed at runtime, not findable as an array" | Descriptor unpack format was reconstructed (`Modbus_RS485_TCP/VNSD_Descriptor_Table_Unpack_Format.md:412`: "SWD dump no longer needed"). |

### 🔍 RE-INVESTIGATE — worth a thorough attack

| Location | Statement | Attack plan |
|---|---|---|
| `Micro_Inverter_FW…md:2684` | "Descriptor table is built by code in external flash (0x10000000+) … not extractable" | Check whether the builder code is in the analyzed binary after all; look for QSPI mapping/init; if needed compare neighboring Micro FW (VNSA/VNSE3). Disassembly run over undefined regions (analogous to BMS). |
| `Micro_Inverter_FW…md:2646` | "runtime-built descriptor table 0x200004C0" | As with Control: uncover the build/store code via prolog scan. |
| `…1492…md:1980 / 2061 / 2066 / 2154` | "callerCount 0 / presumably function pointer registered at runtime / caller not findable" | Exactly the pattern of the per-pack handlers. Disassemble undefined code regions, then re-check xrefs; look for callback registration. |
| `VNSD_Shell.md:354` | "4-byte command pattern in flash sector 7 (0x08060EF8), not in the binary" | 0x08060EF8 lies beyond the end of the OTA image — but the match values may be materialized in the using code (comparisons); check find-constants in the dispatcher. |
| `…1492…md:2294` | "No plain-text string could be reconstructed" (crypto/modulus) | Check context; if RSA/key: possibly derivable from certificates/neighboring FW (cf. TLS key success). |

### ⛔ DATA-LIMIT — genuine availability blocker (verified per case)

| Location | Statement | Status |
|---|---|---|
| `VNSD_Descriptor_Offene_Register…md` (cloud format strings 0x0805F000–0x08063000) | ~290 pointers point beyond the image end | Bytes in NONE of the 14 archive images. BUT: pack field names & cloud JSON keys ARE in the image after all (§12) — only the printing cloud function is missing. Irrelevant for the registers. |
| `Modbus_RS485_TCP/VNSD_Descriptor_Table_Unpack_Format.md:15-18`, `Reverse_Engineering_Methodology.md:137-139`, `VNSD_Control_FW_Analyse_app_0150…md:180/368` | Descriptor table 0x20000354 "built at runtime" | Register numbers can be enumerated via live scan; unpack format however reconstructed. No longer a blocker for the meaning, only for pure number enumeration without a device. |
| `BMS/VNSD_BMS_FW_Analysis_v117.7.md:328` | Reading out values only via SWD/debug shell | Concerns live values, not structure. |

### ℹ️ NOT blockers (false positives of the scan)

- `BLE/VNSD_BLE_Command_Map_v150.md:91` (meter IP auto-discovered) — feature description, not an RE blocker.
- `BMS/VNSD_BMS_FW_Analysis_v118.md:167` ("not present in 117.7") — version diff statement, correct.
- `Micro…md:41` (WiFi not present), `Fehlercodes_Micro…md:213` (isolation test mechanical) — factual.
- `Control…1492…md:1071/1276/1526/1527` — register not present in the scan = hardware-dependent (MPPT/DCDC not fitted), no code blocker.

---

## Recommended order for the next pass

1. **Micro descriptor 0x200004C0 / "external flash"** — largest open "not extractable" claim;
   tackle it with the BMS disassembly recipe.
2. **"callerCount 0 / runtime function pointer" in Control** — systematically disassemble all undefined
   code regions (as begun in §12: Control 1630→1884), then re-evaluate xrefs.
3. **VNSD_Shell.md command matcher (sector 7)** — pull match values from the using comparison code.
4. **Source documentation corrections** for the ✅ REFUTED entries (references to §12/§13 or new findings).

*Created after a full-text grep over 37 MD files + targeted Ghidra verification. The ✅ entries
are evidenced; the 🔍 entries have not yet been conclusively checked, but according to experience so far
are highly likely to be solvable.*

---

## Systematic pass 2026-08-16 — result & unifying insight

All 🔍 items thoroughly checked via Ghidra (incl. additional disassembly runs). Result:

### What IS in the code and was reconstructed (thesis confirmed)
- **All functional handlers/tasks/functions** are in the image and analyzable — proven by the
  8 per-pack CAN store handlers (byte-accurate register mapping) and the debug print functions.
- **Micro descriptor format 0x200004C0**: fully decoded from `modbus_read_register_block`
  (12-byte entry: reg@+0, source pointer@+4, type@+8, elem_size@+9, scale@+0xa, count@+0xb;
  scale codes 0x01=×10 … 0x04=×0.01). Contents can be reconstructed via live scan + debug prints (as with Control).

### The ONE common, verified blocker
Several "who registers/calls X" questions (0xAA table registrar, `Serial_Command_Dispatch`,
`UART_Packet_Receive_Parse`, `Modbus_Dispatcher`/`CH395_UDP_ServerTask` task creation) all end the
same way: **the addresses of these functions appear NOWHERE as a 32-bit value in the available flash**
(checked exhaustively via findBytes for even+thumb; method verified — 0x20014FB4 yields 22 hits,
0x2000018c exactly 1). Since the image also uses no `movt` and `adr` cannot reach the scattered targets,
the **registration/wiring layer demonstrably lies in the flash region
that is contained in NO OTA image**:
- **Control:** tail from `0x0805F000` (sector 7, incl. the shell command match table 0x08060EF8).
- **Micro:** separate region `0x10000000+` (~128 KB, builder of the descriptor table; FileBytes=Mapped, no appendix).

**Consequence:** This is a **genuine, verified data availability blocker** — not a
thoroughness problem. The *meaning* of all registers is nevertheless resolved (via handlers + debug prints
+ live scan); only the *wiring* (which slot calls which handler, where which task starts)
needs the missing sectors. **The only real solution: full-chip dump via SWD/JTAG** of the running
device (or the external 0x10000000 image of the Micro).

### Non-blockers (resolved/recategorized)
- "Point #12 / no plain-text string" (Control 2294): already recognized as **binary register buffer init**
  (`Inverter_Register_Buffer_Init`), no string obfuscation → not a blocker.

### Conclusion
Your motto holds for **everything functional**: it is in the code, you just have to search thoroughly (often in
non-disassembled code). The remaining rest is not "not found",
but "physically not in the distributed image" — and is now precisely localized and justified.
