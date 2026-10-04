# Control FW — Naming: Process History & Lessons

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_Naming_Batch_History_de.md](VNSD_Control_FW_Naming_Batch_History_de.md)

Archive of the batch-by-batch procedure notes from the now-deleted `Control_FW_Function_Tracking.md`
(chronological batch log). The pure address→name data is complete and more current in
[VNSD_Control_FW_Function_Tracking_new.md](VNSD_Control_FW_Function_Tracking_new.md) — this file
keeps only the **process reflections** ("Lesson:" sections) and the context of the larger correction batches,
in case the methodology becomes relevant for future naming sessions.

---

## Batches 1–17 (2026-07-07 to 2026-07-08): Initial naming

All 1,615 `FUN_` functions that were unnamed at the time were named in 17 batches of ~50–108 functions by parallel
sub-agents, with little cross-checking between the agents. Batch 5 found that 20 of the 50
requested addresses in the range `0x08012xxx`–`0x08013xxx` were already named, mid-function offsets or
string data (effectively only 30 new names) — an early indication that address lists should be checked against the current Ghidra status
before naming.

## Batch 18 — Verification correction pass (2026-07-09)

**Trigger:** A hint that functions in Control FW 149.2 might be misidentified/mis-decompiled,
since batches 1–17 were produced by many parallel sub-agents with little cross-checking.

**Method:** Direct Ghidra function comparison (fresh decompilation, not just the documentation description)
for the complete `mbedTLS_MPI_*`/`ECP_*`/`RSA_*`/`PK_`/`OID_`/`Cipher_`/`SSL_`/`X509_` cluster
(142+ functions) as well as all 153 non-crypto functions marked with `medium`/`low` confidence.
Focus: duplicate names (a strong error signal — real mbedTLS sources never have two functions
with the same name), parameter count/order against the real mbedTLS 2.28 API, behavior vs.
claimed name.

**Result:** 62 misnamed functions found and corrected directly in Ghidra (not just in the
documentation). The trigger was a naming conflict (`0x08042c2e`) that had been documented earlier but never actually fixed,
and that turned out to be a cascade of 3 shifted MPI function names; the systematic
review of the entire cluster then found 58 further such misnamings, partly in longer
shift chains (e.g. Add_Abs/Grow/Cmp_Abs/Lset/Add_Int in the MPI cluster).

**Lesson:** A duplicate name was the most reliable signal of a misnaming in practically 100% of the cases examined
— real mbedTLS functions never collide in name.

## Batch 19 — Complete duplicate-name resolution (2026-07-09)

**Trigger:** While building `VNSD_Control_FW_Function_Tracking_new.md` (thematic reorganization), in addition
to the address/name reconciliation, a complete name-duplicate check was run across all 1,417 functions then
named (not just a sample as in batch 18). Result: beyond the
6 duplicates already known, **8 more** were found in the llhttp internals. In total
14 duplicate names were resolved in this round (6 app/library functions + 8 llhttp bit-test helpers,
named by struct offset instead of presumed bit semantics, since the code was structurally identical).

**Side finding:** The total number of named functions rose from 1411 to 1417 between two Ghidra queries
(presumably ongoing background analysis in Ghidra, not attributable to the session).

**Result:** 0 remaining name duplicates (verified by a full scan across all 1417 functions).

**Lesson:** The duplicate check from batch 18 was applied only to a sample, not as an exhaustive scan across *all* named
functions — only the full scan in batch 19 uncovered the
additional 8 llhttp duplicates. **For future batches: always run the duplicate check as a full
scan (`getFunctions()` + name frequency count), not just as a sample.** (See also the
memory entry `feedback_batch-naming-duplicate-check`.)

## Batch 20 — Identification of the last 206 unnamed FUN_ functions (2026-07-09)

**Trigger:** Task to analyze the last remaining unidentified functions using subagents for
parallel processing.

**Starting point:** After batch 19, 1417 of 1623 functions were named — 206 still carried `FUN_0xxxxxxx`.
A check of the memory regions revealed only a single block (`0x08000000`–`0x0805dfff`, a flat-mapped executable image) — no separate
"unidentified memory regions" beyond these 206 functions. In addition, 66 `LAB_*` labels (jump targets without their own function header) were found,
classified as lower priority and not processed.

**Procedure:** The 206 addresses were assigned in 6 chunks of ~35 functions to 6 parallel subagents
(each with full context on naming convention, project structure, neighboring functions). Each agent fully decompiled each
function (incl. callers/callees) and proposed a name with confidence + rationale,
but did NOT rename anything itself — the 202 proposals were checked centrally for duplicates
(among themselves AND against all 1417 names already assigned) (0 collisions) and then applied together
in a single Ghidra transaction.

**4 functions deliberately NOT renamed** (padding/stubs without meaningful semantics): `0x0800bcc0` (4 bytes,
constant-1 return as a function-pointer callback), `0x0802da14` (1 byte, probably an incorrectly delimited fragment), `0x0802db54` (1 byte, ditto), `0x0802dc50` (2 bytes, pure `return;` stub).

**Result:** 202 of 206 functions renamed. Status afterwards: 1619 of 1623 functions named (only
4 deliberately deferred mini-stubs remaining). Full scan confirms: 0 name duplicates.

**Lesson:** Subagent parallelization worked well for this volume (206 functions in 6 chunks)
— each agent got enough context to produce consistent, convention-conforming names even without a handwritten reference (e.g. mbedTLS
X.509/ASN.1 cluster, FreeRTOS kernel cluster).
The central duplicate check before applying remained important: even though 0 collisions occurred this time,
it could easily have missed some with several agents working in parallel, had it not
been performed.

---

*Deleted on 2026-07-11: `Control_FW_Function_Tracking.md` (chronological batch log, 3,649 lines).
Reason: redundant to `VNSD_Control_FW_Function_Tracking_new.md` (thematic order, requested by the user as the only
authoritative source) — confirmed by diff check that all address→name pairs of the old file
are contained in the new one (58 addresses found only in the old file were pure address drift,
correctly captured in the new file under the current address via name match).*
