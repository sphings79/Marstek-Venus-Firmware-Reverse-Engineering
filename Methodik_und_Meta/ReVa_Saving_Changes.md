# Saving ReVa changes in Ghidra

> 🇩🇪 Deutsche Version: [ReVa_Aenderungen_speichern_de.md](ReVa_Aenderungen_speichern_de.md)

**Status:** 2026-09-24, tested on the project (BMS 117.7, BMS 118, Control 150).

## The problem

Renamings, comments and types set via ReVa (`run-script`, `set-comment`, …)
initially end up only in working memory. Two ways that do **not** work:

1. **Saving in the Ghidra project window.** ReVa opens the programs as its own "consumer".
   These copies do not show up in any tool window, and saving in the project window does not capture
   them. Proven: after saving in the project window,
   `getDomainFile().isChanged()` still reported `True`.
2. **`df.save(monitor)` directly in the script.** Fails with
   `IOException: Unable to lock due to active transaction`. The lock comes from the script wrapper
   itself: `run-script` wraps each script in a transaction
   (`getCurrentTransactionInfo()` shows `PyGhidraGhidraScript`). As long as the script is running, the
   program cannot be saved.

## The solution

The script starts a background thread and returns immediately. The thread waits until the script's
transaction has ended and then saves.

The project version `Scripts/save_program.py` is maintained (local, not in the repo). The loader
`~/ghidra_scripts/MarstekSaveProgram.py` reads and runs it, the same pattern as for the
decompilation export (ReVa only runs scripts from registered script directories).

## Procedure (for every session)

After every session in which something was changed via ReVa, **for each changed program**:

```
# 1. Trigger the save
run-script(programPath="/VNSD/BMS/BMS_118_20260119100535e43806957.bin",
           scriptName="MarstekSaveProgram.py")
# Output: "...: save scheduled (check isChanged() in a second call)"
#     or "...: no unsaved changes"

# 2. Check (separate call, not in the same script)
run-script(programPath="/VNSD/BMS/BMS_118_20260119100535e43806957.bin",
           code='print(currentProgram.getDomainFile().isChanged())')
# expected: False
```

Only when step 2 returns `False` is the change saved. Do not report "saved" to the user
before that.

Which programs are unsaved is shown by step 2 for each program individually. A
total list via `state.getProject()` does not exist under ReVa (`state.getProject()` is there
`None`).

## Pitfalls

- **One script per program.** `currentProgram` is always the one chosen via `programPath`.
- **Takes effect only after the script ends.** Even `isChanged()` is not meaningful within the same script:
  A change in the running script only becomes visible when the wrapper transaction ends. Always check in the next call.
- **Hold a reference.** The script stores the runnable on `__main__`. Otherwise Python can
  collect the proxy object before the Java thread has finished.
- **Versioned programs** (e.g. BMS 118, checked out) are only saved to the local checkout.
  Checking in is a separate step (`checkin-program`) and only on request.
- **Saving in the GUI remains necessary for GUI changes.** What was changed by hand in the CodeBrowser
  is saved there as usual. The script only affects the ReVa copy.
- **Order relative to the export.** The decompilation export (`Decompilation_Export.md`) reads the
  ReVa copy and also works unsaved. Still do not forget to save afterwards,
  otherwise the export and the saved project will not match.
