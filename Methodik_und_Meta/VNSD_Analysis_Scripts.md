# Marstek Venus D — Analysis scripts (overview)

> 🇩🇪 Deutsche Version: [VNSD_Analyse_Skripte_de.md](VNSD_Analyse_Skripte_de.md)

**Project:** Marstek Venus D (`VNSD-0`) reverse engineering
**Reference:** Split out from [VNSD_Control_FW_Analysis_app_1492_0702_142136.md](../Control_FW/VNSD_Control_FW_Analysis_app_1492_0702_142136.md) (formerly section 12)

**Status 2026-07-14:** This overview was checked against the actual folder contents and
corrected (the cross-check of 2026-07-10 had noted that several referenced scripts no longer
exist). Sections 1 and 5 list the scripts that **actually exist**; sections
2–4 document earlier one-off tools that have since been removed from the project folder
(results are preserved in the respective analysis documents).

**Note:** All scripts named here (own Python helper scripts) are local only and are not part of the repository.

---

## 1. Modbus TCP scan scripts (present in the project folder)

| Script | Description | Usage |
|---|---|---|
| `scan_registers.py` | Flexible register scanner (raw socket, no pymodbus) — single/range registers, watch mode, `--unknown-only` filter | `python3 scan_registers.py --host IP --regs 30000-30040,32200,34000-34033` |
| `scan_known_registers.py` | Targeted scan of **only** the registers already known in the register map (CSV) — ~30 instead of ~440 batches, approx. 30× faster than the full scan | `python3 scan_known_registers.py --host IP --out scan.csv` |
| `scan_continuous.py` | Continuous scan of known registers at a fixed interval, CSV with a new column per pass (Ctrl+C saves & exits) | `python3 scan_continuous.py --host IP --interval 10 --out monitor.csv` |
| `scan_powercycle.py` | Event scanner for state changes (power cycle, battery DC disconnection, off-grid/backup (EPS) power, force mode changes) — focused on unknown/so-far-always-0 registers | `python3 scan_powercycle.py --host IP --interval 2 --out event_scan.csv` |

**Note on batch performance** (applied to the original full-scan approach):

| Method | Requests for 65535 reg | Time (100ms/req) |
|---|---|---|
| Single | 65,535 | ~109 hours |
| Batch (32/req, as in the current scripts) | ~2,048 | **significantly faster, see `scan_known_registers.py` for targeted scan** |

*Correction compared with the earlier version of this file:* The `scan_modbus.py` /
`scan_modbus_batch.py` (125 registers/request) listed here earlier no longer exist in the project folder — the
four scripts above are the currently used successors. **Known open item (not part of this
cleanup run):** `Modbus_RS485_TCP/VNSD_Modbus_TCP_Connection_Reference.md` still references
`scan_modbus_batch.py` and `test_batch_limit.py` in sections 7/8 as well — both are likewise no longer
present. This would have to be corrected in that file in a separate pass.

## 2. Firmware analysis scripts (historical, scripts no longer present)

These six one-off scripts were used to locate the Modbus register table in the firmware image.
They no longer exist in the project folder; the respective results are documented in
`Control_FW/VNSD_Control_FW_Analysis_app_1492_0702_142136.md`.

| Script | Version | Method | Result |
|---|---|---|---|
| `find_registers_v2.py` | v2 | 6 strategies (strings, MOVW, cluster) | MOVW #0x7530 found at 0x0801295e |
| `find_registers_v3.py` | v3 | Targeted analysis of known addresses | String pool found at 0x08018000 |
| `find_registers_v4.py` | v4 | SRAM pointer validation | Table not found (0 hits) |
| `find_registers_v5.py` | v5 | ARM Thumb-2 disassembly | Init function region narrowed down |
| `find_registers_v6.py` | v6 | .data section mapping | 0x08055d4c = MQTT strings (wrong) |
| `find_registers_v7.py` | v7 | STR offset search + pointer analysis | 0 STR hits; SRAM pointer blocks found |

## 3. YAML generator (historical, script no longer present)

`generate_final_yaml.py` no longer exists in the project folder. Original invocation (to
document the workflow of that time):

```bash
python3 generate_final_yaml.py \
    --csv30 register_3000039999_matched.csv \
    --csv40 register_4000049999_matched_csv.csv \
    --out   marstek_venus_registers_final.yaml
```

## 4. Ghidra scripts (Jython 2.7, historical, no longer present)

| Script | Description |
|---|---|
| `MaerstekExtractor.py` | Writes descriptor entries via getReferencesFrom() (0 hits without SRAM block) |
| `MaerstekDecompiler.py` | Uses Ghidra decompiler + regex parser |
| `FindRouterCallers.py` | Finds callers of FUN_0801c088 → leads to init code |

## 5. Other existing scripts (not previously listed in this overview)

| Script | Location | Description | Docs |
|---|---|---|---|
| `shell_probe.py` | Project root | TCP shell probe (port 8091), tests text CLI format and binary BLE protocol (`[0x73][len][0x23][cmd][payload][XOR-CRC]`) | see `VNSD_Shell.md` |
| `marstek_rot_cipher.py` | Project root | ROT-N cipher tool for AES key extraction (reverses the ROT-6/7/9 obfuscation of the firmware) | already documented in `AES_Crypto_Stack_Analysis.md` (local, not in the repo) |
| `security/brute_force_certs.py`, `security/brute_force_venuse.py`, `security/brute_force_vnsa.py` | `security/` (local, not in the repo) | Replicates the `marstek-firmware-analyzer` brute-force algorithm (key and Base64 cert candidates × Caesar/ROT shifts) against Venus D/E/A firmware dumps | **not** referenced in any analysis file so far — thematically belongs to `AES_Crypto_Stack_Analysis.md`/`security_problems.md` §4, but is not part of this cleanup run (the scripts concern the security topic, not Modbus/register scanning) |
| `security/build_doc.py`, `security/merge.py` | `security/` | One-off merge helper scripts (Ghidra function dump ↔ function tracking doc), reference an old session path (`/sessions/laughing-focused-volta/...`) | no ongoing use discernible, not documented further |

*Remark:* The three `brute_force_*.py` scripts and `build_doc.py`/`merge.py` were found during this
check in the `security/` folder, but thematically do not belong to this
Modbus/firmware scan script overview. Since this is a cross-cutting topic (security tooling
vs. register scanning tooling), they were only listed here for information instead of being mixed into section 1
— whether a separate short doc makes sense for them should be decided by the user.

---

## Warning: missing transaction ID check in older scanners (2026-08-22)

`scan_continuous.py` and `scan_known_registers.py` do set a Modbus transaction ID in the MBAP header,
but **do not check it on receipt**. `parse()`
validates only the function code. If a late response to request N
arrives while request N+1 is open, it is read as the response to N+1 — the
values end up in the wrong register.

Only `scan_registers.py` validates (`recv_response(..., expect_tid=...)`).

### Proven case

Register `38003` showed the value `118` in three logs, which led to the hypothesis
that the CAN frame buffer is occasionally populated. In fact the 118 comes from
register `37012` (`bms_version`):

| Log | `37012` | `38003` |
|---|---|---|
| `entladen_lang.csv` (local, raw data) | 118 in 65/66 batches, **0 in exactly one** | 0 in 65/66, **118 in exactly the same one** |
| `entladen_dc.csv` | 118 in 3/4, 0 in one | 0 in 3/4, 118 in the same one |
| `unklar_watch discharge ...csv` | 118 in 7/8, 0 in one | 0 in 7/8, 118 in the same one |

The complementarity is exact: where one value disappears, it shows up in the
other register, in the same batch.

### Consequence for the evaluation

- **Isolated** deviating values in logs from these two scripts are
  generally suspect. Before drawing any conclusion, check whether another register
  lost its usual value in the same batch.
- Values that are **stable** over many batches are not affected.
- For new measurements, use `scan_registers.py` or
  `watch_can_frames_38000.py` — both validate the TID.

The bug has **not** yet been fixed in the two scripts.
