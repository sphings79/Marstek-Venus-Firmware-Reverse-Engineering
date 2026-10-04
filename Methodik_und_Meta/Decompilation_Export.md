# Decompilation Export — Search Index Alongside Ghidra

> 🇩🇪 Deutsche Version: [Dekompilat_Export_de.md](Dekompilat_Export_de.md)

## Why

Ghidra's own search has a gap that cost us time. `find-constant-uses`
only finds constants if they appear as an immediate in the instruction. If a value is loaded via
a literal pool — on Thumb-2 the normal case for anything above 8 bits —
it does not show up there.

Concretely: the search for `1800` or `0x708` returned **zero hits**, even though the
constant occurs four times in the Control code. Only a full decompilation of all 1884
functions brought it to light, and a `grep` over the export even found a
fourth location that the regex search over the decompilation had missed.

The export turns such questions into a `grep`.

## Running it

Via ReVa, once per program:

```
run-script(programPath="/VNSD/Control/VNSD-0_app_0150_0805_115146.bin",
           scriptName="MarstekExportDecompilation.py")
```

`~/ghidra_scripts/MarstekExportDecompilation.py` is only a loader — ReVa does not run
scripts outside registered script directories. The project version is maintained
under `Scripts/export_decompilation.py` (local, not in the repo).

Duration: about 35 s for 1884 functions, 7 s for the smaller images.

Before writing, the script deletes all existing `.c` files in the target folder.
This is intentional: otherwise files of renamed functions remain under their old
name and you end up searching through corpses later.

## Target folders

| Program | Export |
|---|---|
| Control v150 | `Dekompilate/VNSD_Control_v150/` (local, not in the repo) |
| Micro/VNS 116 | `Dekompilate/VNSD_Micro_116/` |
| BMS 118 | `Dekompilate/VNSD_BMS_118/` |

Assignment is done by **substring match against `currentProgram.getName()`** — beware,
the Ghidra program name is not the project path: the Micro image is called
`vd_inv_app_0116_0702_ota_163439.bin` there, not `Micro_VNS_116_...`. A prefix
match missed exactly that and stored the export under the file name.

**Device prefix (since 2026-09-24).** The target folders are named `VNSD_…` so that exports of
Venus A and E3 can sit alongside them. The keys in the script therefore contain the device
(`VNSD-0_app_0150` instead of `app_0150`) — the short key also matched the
Control 150 images of A and E3 and would have written their export over the D export.

**No timestamp subfolders.** Several snapshots side by side
multiply every `grep` and invite reading outdated code. The
separating axis is the firmware version.

The file name is `<address>_<function name>.c`. The address comes first so that
sorting follows the memory layout and a rename does not
scramble the order. Each file carries name, address and size at the top as a comment.

`_index.md` lists all functions with address, size, number of callers and
file name, plus the creation date and symbol status.

**Don't forget to save.** The export reads the ReVa copy of the program. Changes made
via ReVa are therefore not saved yet. Saving in the project window
is not sufficient for that, see `ReVa_Saving_Changes.md`.

## The rule

**The export is a snapshot. The source of truth remains the
Ghidra project.**

After each of these changes it is outdated and must be regenerated:

- a function renamed
- a variable or a type set (visibly changes the decompilation)
- a new function defined or one deleted
- data converted to code or vice versa
- firmware version switched or markup transferred

Best to run it right after an analysis session, then the
tree stays honest. The creation date at the top of `_index.md` is the check: if it is
older than the last Ghidra work, the export is no longer reliable.

## What the export is **not** suited for

- **Not as evidence in the documentation.** Quotes belong backed by an address, and the
  address is valid in Ghidra. The export can be outdated without anyone noticing.
- **Not for regions that Ghidra treats as data.** Whatever was never disassembled
  is also not in the export. The range `0x0802E684`–`0x0802E730`, for example,
  contains FreeRTOS task stubs that were marked as raw data — they appeared
  in no decompilation although they are executed.
- **Not as a diff basis between firmware versions**, as long as the
  symbol statuses diverge. First transfer markup, then export both sides,
  then `diff -r`.

## Side effect during cleanup

Anyone who wants to make code visible in an undefined region has to delete data there
and disassemble. **Caution:** `clearCodeUnits` over a range also removes
existing instructions and their references. On 2026-08-25 the
calls at `0x0802E6E0` and `0x0802E6F4` were lost together with their references and
had to be restored by a targeted `ArmDisassembleCommand` starting from the known instruction boundary.

Rule derived from this: **never clear a whole range**, but
disassemble from an address known to be an instruction boundary — for example
because a reference points to it.
