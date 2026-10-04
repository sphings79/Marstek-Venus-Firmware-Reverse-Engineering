# Firmware-Fehler: vorzeichenbehaftete Register mit Divisions-Skala

> 🇬🇧 English version: [VNSD_FW_Bug_Read_Serializer_Sign_2026-08-22.md](VNSD_FW_Bug_Read_Serializer_Sign_2026-08-22.md)

**Datum:** 22. August 2026
**Firmware:** Control v150 (`VNSD-0_app_0150_0805_115146.bin`), Venus D
**Betroffene Funktion:** `Read_Serializer` @ `0x08055414` (v150, korrekte Basis 0x08004800; ursprünglich als `0x08050c14` notiert — alte Ghidra-Analyse mit falscher Basis 0x08000000, Differenz 0x4800)
**Status:** Ursache im Dekompilat belegt, an zwei Hardware-Messungen exakt reproduziert
**Auswirkung heute:** genau **ein** Register (32101), der Defekt liegt aber im gemeinsamen Pfad

---

## 1. Der Fehler

`Read_Serializer` legt den gelesenen Wert in eine lokale Variable vom Typ `uint` und
wendet darauf den Skalencode an:

```c
uint local_38;
...
else if (bVar2 == 0x12) {                       // i16
    local_38 = (uint)*(short *)(quelle);        // Vorzeichenerweiterung nach uint
}
...
switch ((char)param_4[5]) {
    case 1: local_38 = local_38 * 10;  break;   // ok
    case 2: local_38 = local_38 * 100; break;   // ok
    case 3: local_38 = local_38 / 10;  break;   // FEHLER bei negativen Werten
    case 4: local_38 = local_38 / 100; break;   // FEHLER bei negativen Werten
    case 5: local_38 = -local_38;      break;   // ok
}
```

Die Vorzeichenerweiterung ist korrekt — aus `-122` wird `0xFFFFFF86`. Multiplikation und
Negation arbeiten im Zweierkomplement richtig weiter. Die **Division nicht**: `local_38`
ist vorzeichenlos, also teilt der Compiler 4.294.967.174 statt −122.

Anschließend nimmt `Byte_Swap_Copy` die unteren zwei Byte des Ergebnisses. Übrig bleibt
ein Wert ohne Bezug zum Messwert.

## 2. Reproduktion

`32101` (`bms_battery_current`, i16, Skalencode 3) an zwei Betriebspunkten:

| Betriebszustand | SRAM `0x20014F90` | Modbus liefert | Erwartet |
|---|---|---|---|
| Leerlauf, ca. −0,3 A | `-3` | **39321** (`0x9999`) | `0` |
| 600 W Entladung, −12,2 A | `-122` | **39309** (`0x998D`) | `-12` |
| **600 W Ladung, +10,3 A** | `+103` | **10** | `10` — korrekt |

Nachrechnung des Fehlerpfads:

```python
def fw_read(sram_i16):
    u = sram_i16 & 0xFFFFFFFF     # (uint)(short)x
    u = u // 10                   # vorzeichenlos
    return u & 0xFFFF             # Byte_Swap_Copy, 2 Byte

fw_read(-3)    ->  39321
fw_read(-122)  ->  39309
```

Beide Messwerte exakt getroffen. Der SRAM-Wert stammt aus dem Doppelscan vom 22.08.
(`Scan_Logs/idle_bypass_backup48w_2026-08-22.csv` (lokal, Rohdaten) und
`discharge600w_backup49w_2026-08-22.csv` (lokal, Rohdaten)); als Gegenprobe lag zeitgleich `34301`
(Pack-4-Strom, kein Skalencode) bei genau `-3` bzw. `-122`.

Positive Werte passieren den Pfad korrekt — und das ist inzwischen **gemessen**, nicht nur
gerechnet: beim Laden mit 600 W lieferte dasselbe Register `10` bei einem Pack-Strom von
`103`. Der Fehler tritt also **nur bei negativen Werten** auf; bei einem Batteriespeicher
heißt das: bei jeder Entladung.

Nebenbefund desselben Scans: auch beim **Laden** ist nur ein Pack aktiv (Pack 4, `+103`,
alle anderen `0`). Die Aussage „immer nur ein Pack" gilt damit in beide Richtungen.

## 3. Wirkungsbereich in v150

Betroffen ist jede Kombination aus **vorzeichenbehaftetem Typ** (`0x11` i8, `0x12` i16,
`0x14` i32) und **Divisions-Skalencode** (3 = ÷10, 4 = ÷100).

Auswertung der 246 Descriptor-Einträge:

| Typ | Skala | Einträge | |
|---|---|---|---|
| i16 | ÷10 | **1** | betroffen — 32101 |
| i16 | ×1 | 38 | unauffällig |
| i16 | ×10 | 2 | unauffällig |
| u16 | ÷10 | 2 | vorzeichenlos, korrekt |
| u32 | ÷10 | 6 | vorzeichenlos, korrekt |
| übrige | ×1 | 197 | keine Skalierung |

**Heute also ein einziges Register.** Der Defekt sitzt aber im Serializer, den alle
FC03-Lesezugriffe durchlaufen — jedes künftige Register mit dieser Typ/Skala-Kombination
erbt ihn. Für Venus A und E v3 (gleiche Firmware-Basis) ist die Zählung nicht geprüft,
deren Descriptor-Tabellen liegen nicht dekodiert vor.

## 4. Folge für die Auswertung

`32101` ist **nicht** als Strommesswert verwendbar. Der Wert lässt sich zwar näherungsweise
zurückrechnen — `Strom ≈ (Rohwert − 39321) A` — aber nur mit 1-A-Quantisierung, weil die
Division vor der Kürzung auf 16 Bit stattfindet, und asymmetrisch zwischen Laden und
Entladen. Für Messzwecke unbrauchbar.

**Ersatz:** die Pack-Stromregister `34x01` tragen keinen Skalencode und sind unversehrt.
Da der Venus D immer nur ein Pack entlädt, liefert deren Summe denselben Strom, den auch
die Firmware intern verwendet. Genau so ist es in der HA-Integration umgesetzt
(`battery_power_bms`).

## 5. Einordnung

Die zurückgerechnete „Auflösung von rund 1 A je Zählschritt", die am 22.08. aus dem Delta
zweier Messungen abgeleitet wurde, war kein Merkmal des Registers, sondern die
Quantisierung dieses Fehlers. Die entsprechende Notiz in der Register-Map ist damit
überholt.

Ein englischer Text zur Meldung an den Hersteller liegt in
`VNSD_FW_Bug_Report_EN_Read_Serializer.md`.

## 6. Status in v151 (Firmware 151, geprüft 2026-10-01)

**Unverändert.** `Read_Serializer` liegt in v151 bei `0x08055fe0` (626 Byte, identische Größe) und ist bis auf
Decompiler-Variablennamen (`bVar1`/`bVar2` vertauscht zu `bVar2`/`cVar1`) instruktionsidentisch zu v150. Die
`switch`-Fälle 3/4 rechnen weiterhin auf der vorzeichenlosen Variable `int_val`:

```c
switch(*(undefined1 *)((int)field_desc + 10)) {
case 3:
  int_val = int_val / 10;   // weiterhin FEHLER bei negativen Werten
  break;
case 4:
  int_val = int_val / 100;  // weiterhin FEHLER bei negativen Werten
  break;
```

Register 32101 bleibt in v151 nach derselben Auswertung (Typ i16, Skalencode ÷10) das einzig betroffene Register
(siehe `VNSD_Descriptor_Table_Diff_v150_vs_v151_de.md`, Abschnitt „Unverändert fehlerhaft“). Quelle: Dekompilat
`Dekompilate/VNSD_Control_v151/08055fe0_Read_Serializer.c` (lokal, nicht im Repo).
