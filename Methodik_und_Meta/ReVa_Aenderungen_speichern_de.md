# ReVa-Änderungen in Ghidra speichern

> 🇬🇧 English version: [ReVa_Saving_Changes.md](ReVa_Saving_Changes.md)

**Stand:** 2026-09-24, am Projekt getestet (BMS 117.7, BMS 118, Control 150).

## Das Problem

Umbenennungen, Kommentare und Typen, die über ReVa (`run-script`, `set-comment`, …) gesetzt
werden, landen zunächst nur im Arbeitsspeicher. Zwei Wege, die **nicht** funktionieren:

1. **Speichern im Ghidra-Projektfenster.** ReVa öffnet die Programme als eigener „Consumer“.
   Diese Kopien tauchen in keinem Tool-Fenster auf, und das Speichern im Projektfenster erfasst
   sie nicht. Nachgewiesen: Nach dem Speichern im Projektfenster meldete
   `getDomainFile().isChanged()` weiterhin `True`.
2. **`df.save(monitor)` direkt im Skript.** Scheitert mit
   `IOException: Unable to lock due to active transaction`. Die Sperre kommt vom Skript-Rahmen
   selbst: `run-script` legt um jedes Skript eine Transaktion
   (`getCurrentTransactionInfo()` zeigt `PyGhidraGhidraScript`). Solange das Skript läuft, lässt
   sich das Programm nicht speichern.

## Die Lösung

Das Skript startet einen Hintergrund-Thread und kehrt sofort zurück. Der Thread wartet, bis die
Transaktion des Skripts beendet ist, und speichert dann.

Gepflegt wird die Projektfassung `Scripts/save_program.py` (lokal, nicht im Repo). Der Loader
`~/ghidra_scripts/MarstekSaveProgram.py` liest und führt sie aus, dasselbe Muster wie beim
Dekompilat-Export (ReVa führt nur Skripte aus registrierten Skriptverzeichnissen aus).

## Ablauf (für jede Session)

Nach jeder Sitzung, in der über ReVa etwas geändert wurde, **je geändertem Programm**:

```
# 1. Speichern anstoßen
run-script(programPath="/VNSD/BMS/BMS_118_20260119100535e43806957.bin",
           scriptName="MarstekSaveProgram.py")
# Ausgabe: "...: save scheduled (check isChanged() in a second call)"
#      oder "...: no unsaved changes"

# 2. Prüfen (eigener Aufruf, nicht im selben Skript)
run-script(programPath="/VNSD/BMS/BMS_118_20260119100535e43806957.bin",
           code='print(currentProgram.getDomainFile().isChanged())')
# erwartet: False
```

Erst wenn Schritt 2 `False` liefert, ist die Änderung gespeichert. Dem Nutzer vorher nicht
„gespeichert“ melden.

Welche Programme ungespeichert sind, zeigt Schritt 2 für jedes Programm einzeln. Eine
Gesamtliste über `state.getProject()` gibt es unter ReVa nicht (`state.getProject()` ist dort
`None`).

## Fallstricke

- **Ein Skript je Programm.** `currentProgram` ist immer das über `programPath` gewählte.
- **Wirkung erst nach Skriptende.** Auch `isChanged()` ist innerhalb desselben Skripts nicht
  aussagekräftig: Eine Änderung im laufenden Skript wird erst beim Ende der Rahmen-Transaktion
  sichtbar. Immer im nächsten Aufruf prüfen.
- **Referenz halten.** Das Skript legt das Runnable auf `__main__` ab. Sonst kann Python das
  Proxy-Objekt einsammeln, bevor der Java-Thread fertig ist.
- **Versionierte Programme** (z. B. BMS 118, ausgecheckt) werden nur in den lokalen Checkout
  gespeichert. Einchecken ist ein eigener Schritt (`checkin-program`) und nur auf Wunsch.
- **Speichern im GUI bleibt nötig für GUI-Änderungen.** Was im CodeBrowser von Hand geändert
  wurde, speichert man wie gewohnt dort. Das Skript betrifft nur die ReVa-Kopie.
- **Reihenfolge mit dem Export.** Der Dekompilat-Export (`Dekompilat_Export_de.md`) liest die
  ReVa-Kopie und funktioniert auch ungespeichert. Speichern trotzdem danach nicht vergessen,
  sonst passen Export und gespeichertes Projekt nicht zusammen.
