# VNSD-0 MPPT-Firmware — Architektur und Ladeadresse

> 🇬🇧 English version: [VNSD_MPPT_C28x_Image_Setup.md](VNSD_MPPT_C28x_Image_Setup.md)

Ergebnis der ersten Analyse des MPPT-Images (v104, 98304 Bytes, SHA-256 `4237eb5e8216a5ac3caf6c5adaafdf6bee31ee78843a488206239991c1a4091d`). Bis dahin hatte das Projekt kein MPPT-Image (siehe [MPPT_Hardware_External_Research_de.md](MPPT_Hardware_External_Research_de.md)).

## Zusammenfassung

| Eigenschaft | Wert | Beleg |
|---|---|---|
| Kern | **TI TMS320C28x** (C2000, Piccolo-Klasse) | Opcode-Statistik, Startup-Code (unten) |
| Genauer Chip | **unbekannt** (Platine vergossen, Aufschrift nicht lesbar) | — |
| Wortadressiert | ja, 1 Adresse = 16 Bit | Architektur |
| Image-Größe | 0xC000 Worte (48 K Worte) | Dateigröße / 2 |
| Ladeadresse | **0x3E8000** (Wortadresse) | Einsprungsprung + Startup-Code (unten) |
| Einsprungpunkt | `0x3F0BAE` (`c_int00`) | erstes Wort des Images |

Datei-Wort `i` liegt auf Wortadresse `0x3E8000 + i`; der Byte-Offset in der Datei ist `2·i`.

## Wie die Architektur erkannt wurde

Das Image ist kein Cortex-M (keine Vektortabelle, kein Stackpointer in den ersten Worten). Die Opcode-Statistik im Codebereich zeigte die C28x-Signatur: 1548 × `0x761F` (`MOVW DP,#imm`), 369 × `0x0006` (`LRETR`), 699 × `0x8F00`. Die `MOVW DP`-Operanden (Datenseiten zu 64 Worten) zeigen auf Peripherie-Bereiche, wie sie bei einem Piccolo-Baustein üblich sind: `0x0B00` (ADC-Ergebnisse), `0x6800` (ePWM), `0x7000`/`0x7100` (Systemsteuerung / ADC), `0x7900` (GPIO), `0x8000+` (RAM). Das passt zu einem MPPT-Wandler, identifiziert aber **nicht** das genaue Derivat (F2803x und F2806x haben ein sehr ähnliches Layout).

## Wie die Ladeadresse abgeleitet wurde

Die ersten beiden Worte (`0x007F 0x0BAE`) dekodieren als `LB 0x3F0BAE`. Mit Image-Start `0x3E8000` gilt:

- Das Ziel liegt auf Datei-Wort `0x8BAE`, davor steht ein `LRETR`, das Ziel ist also ein Funktionsanfang.
- Der Code dort ist ein lehrbuchmäßiges `c_int00`:

```
3F0BAE  SETC  Objmode
3F0BAF  EALLOW
3F0BB0  MOVZ  DP,#0x1c0          ; Seite 0x7000 (Systemsteuerung)
3F0BB1  MOV   @0x29,#0x68        ; WDCR = 0x68 -> Watchdog aus
3F0BB3  EDIS
3F0BB4  LB    0x3F0347           ; weiter mit dem C-Runtime-Setup
```

- Vor `0x3F0347` steht ebenfalls ein `LRETR`. Dort werden Stack und Statusbits gesetzt (`MOV @SP,#0x50`, `SETC Objmode`, `CLRC AMODE`, `SETC M0M1MAP`, `ASP`), danach folgt der Aufruf der Startup-/`main`-Kette.

Beide Sprungziele treffen nur bei dieser Basisadresse auf Funktionsanfänge. Die Basis ist damit gut gestützt, aber nicht durch ein Datenblatt bewiesen. Die genaue Flash-Größe des Chips ist unbekannt, deshalb wird das Image nur über seine eigene Länge abgebildet.

## Ghidra-Einrichtung

Ghidra 12.1.1 hat keinen C28x-Prozessor. Verwendet wurde das Community-Modul [ghidra-tms320c28x](https://github.com/outlandnish/ghidra-tms320c28x) (Apache-2.0, inoffiziell, gegen den F28377D entworfen, noch in Arbeit).

1. Repository klonen und die Specs mit Ghidras eigenem Compiler übersetzen:
   `support/sleigh data/languages/tms320c28x.slaspec` und `support/sleigh data/languages/tms320c28x_cla.slaspec`
2. `data/`, `ghidra_scripts/`, `Module.manifest`, `extension.properties`, `LICENSE`, `NOTICE` nach `~/Library/ghidra/<Ghidra-Version>/Extensions/ghidra-tms320c28x/` kopieren und `version=` in `extension.properties` auf die installierte Ghidra-Version setzen (im Repository steht 12.1.2).
3. Ghidra neu starten (Sprachen werden beim Start eingelesen).
4. Die `.bin` mit Sprache `TMS320C28x:LE:32:default`, Loader *Raw Binary* und Basisadresse `0x3E8000` importieren.

Hinweise:
- **Lokaler Patch am Modul:** Das Modul (Stand des Upstream-Commits vom 2026-09-28) dekodiert `IACK #imm16` nicht (Opcode `0x763F` + imm16). Dieses Image nutzt ihn dreimal, und ohne ihn zerlegt Ghidra die Funktionen. In `data/languages/tms320c28x.sinc` wurde ein Konstruktor ergänzt (markiert „LOCAL PATCH“) und `tms320c28x.slaspec` neu kompiliert; das Original liegt daneben als `tms320c28x.sinc.orig_upstream`. Die Opcode-Zuordnung stammt aus dem Zusammenhang und ist nicht gegen SPRU430 bestätigt.
- Adressen im wortadressierten Raum: In Skripten liefert `Address.getOffset()` einen **Byte**-Offset (Wort × 2); `toAddr("3E8000")` nimmt die Wortadresse als String, `toAddr(0x3E8000)` nicht.
- Das Modul zielt auf den F28377D. Ein Piccolo-Baustein hat einen kleineren Befehlssatz, die Standardsprache sollte ihn also dekodieren; Peripherie-Registernamen aus den Setup-Skripten des Moduls passen nicht.
- Das Modul ist kein offizielles TI-/NSA-Produkt; zweifelhafte Dekodierungen gegen das TI-Referenzhandbuch (SPRU430) prüfen.

## Analysestand (Ghidra-Projekt `/VNSD/MPPT`)

| Punkt | Stand |
|---|---|
| Codebereich | Wort `0x3E8000`–`0x3F0BE1` (35810 Worte), zu 99,6 % dekodiert (der Rest ist Header-Füllung) |
| Funktionen | 438, alle benannt (188 hoch, 250 mittel, 0 niedrig; die Konfidenz steht im Kommentar jeder Funktion). Befunde: [VNSD_MPPT_Firmware_Findings_de.md](VNSD_MPPT_Firmware_Findings_de.md) |
| Datenbereich am Ende | Wort `0x3F0BE2`–`0x3F3FFF`: Strings (16-Bit-Zeichen, ein Zeichen je Wort), Konstantentabellen und die Kopiertabelle der C-Laufzeit (`.cinit`, von `c_int00` bei `0x3F2B30` gelesen); als Worte / Strings typisiert. Funktionen gibt es dort nur vier Hilfsfunktionen in einer kleinen Code-Insel `0x3F2CA0`–`0x3F2D19` |
| RAM / Peripherie | drei Blöcke sind zum Beschriften abgebildet (`0x0000`–`0x1FFF`, `0x6000`–`0x7FFF`, `0x8000`–`0x9FFF`); 106 Globale sind anhand von Konsolenstrings und Code benannt |
| Dekompilation | alle 438 Funktionen dekompilieren fehlerfrei; lokaler Export unter `Dekompilate/VNSD_MPPT_104/` (nur privates Repo) |

Wie die Funktionen gefunden wurden: Der Einsprung und jede nicht dekodierte Stelle im Codebereich wurden als möglicher Funktionsanfang behandelt und verfolgt, bis kein neuer Code mehr hinzukam. Funktionen ohne Aufrufer (ISRs, indirekte Aufrufe) sind damit plausibel, aber nicht verifiziert; ein erster Versuch hatte auch den Datenbereich erfasst und dort etwa 340 falsche Funktionen erzeugt, die entfernt wurden (die letzte aufgerufene Funktion endet bei `0x3F0BE2`).

Der erste Versuch, die falschen Funktionen im Datenbereich zu entfernen, hat auch die oben genannte Code-Insel entfernt; sie wurde wiederhergestellt, nachdem die Analyse zeigte, dass echter Code dorthin springt.

Wie die Namen entstanden sind: Die Funktionen wurden von unten nach oben in drei Wellen entlang des Aufrufgraphen benannt (Blattfunktionen zuerst), jeweils von einem eigenen Prüfer, der Dekompilat, Strings und die Namen der Aufgerufenen las; jeder Name trägt eine Konfidenz. Die Namen sind Beschriftungen aus der statischen Analyse, nicht an Hardware geprüft.

Noch nicht gemacht: Peripherie-Registerlabels über die 106 benannten Globalen hinaus, Strukturtypen für den Kanalkontext und den Telemetrieblock sowie die Prüfung der Ziele der Interrupt-Vektortabelle (die Tabelle wird von `0x3F07CE` aus `0x3F296A` kopiert; sie wurde nicht dekodiert).

## Offen

- Genauer Chiptyp und Flash-Größe (braucht die Aufschrift, einen Datenblatt-Abgleich der Peripherie-Karte oder eine andere Quelle).
- Bedeutung der letzten beiden Worte `0x7CE8 0x2222` des Images (Prüfsumme oder Marker, unklar).
- Wie das Control das Image an den MPPT schickt (der Control-Code hat einen YModem-Empfänger und ein OTA-Zielflag für den MPPT; eine MPPT-seitige Zieladresse wurde nicht gefunden, der Übertragungsweg wurde aber nicht zu Ende analysiert).
