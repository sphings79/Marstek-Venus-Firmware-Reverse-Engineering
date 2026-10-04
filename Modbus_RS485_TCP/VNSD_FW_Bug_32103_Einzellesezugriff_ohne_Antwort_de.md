# 32103 einzeln gelesen: keine Antwort, Session tot

> 🇬🇧 English version: [VNSD_FW_Bug_32103_Single_Read_No_Response.md](VNSD_FW_Bug_32103_Single_Read_No_Response.md)

**Gerät:** Marstek Venus D (VNSD-0), direkt, ohne Proxy
**Firmware:** Control/EMS **v150**
**Register:** 32103 `bat_sample_power_w2` — die obere Hälfte des 32-Bit-Werts ab 32102
**Datum:** 20. September 2026
**Status:** am Gerät reproduziert · Ursache im Code identifiziert · Wirkung: Verbindungsverlust

---

## 1. Kurzfassung

Ein FC03-Lesezugriff auf **Register 32103 mit Anzahl 1** wird vom Gerät **nicht beantwortet** —
weder mit einem Wert noch mit einer Exception. Die Modbus-Session ist danach tot: auch die
nächste Anfrage auf derselben Verbindung läuft in den Timeout.

Ein Blocklesezugriff, der dasselbe Register mit abdeckt, funktioniert dagegen einwandfrei.

**Ursache:** der Float-Zweig des `Read_Serializer` liest bei einem Zugriff auf die zweite
Hälfte eines `float32`-Eintrags von einer **nicht wortausgerichteten Adresse**. Auf Cortex-M
verlangt `VLDR` Wortausrichtung; die Ganzzahl-Zweige daneben nutzen `LDR` und verkraften
Schieflage, weshalb alle anderen 32-Bit-Register unauffällig sind.

---

## 2. Messung

Frische Verbindung je Zeile, Gerät im Leerlauf, eine Anfrage:

| Anfrage | Ergebnis | Zeit | Session danach |
|---|---|---|---|
| **32103, Anzahl 1** | **keine Antwort** | Timeout | **tot** — 32104 danach ebenfalls Timeout |
| 32103, Anzahl 1, Wiederholung | keine Antwort | Timeout | tot |
| 32102, Anzahl 2 (deckt 32103 ab) | `0 0` | 8,6 ms | in Ordnung |
| 32100, Anzahl 5 | `5269 0 0 0 23` | 9,2 ms | in Ordnung |
| 32102, Anzahl 1 | `0` | 4,4 ms | in Ordnung |
| 32104, Anzahl 1 | `23` | 4,5 ms | in Ordnung |

Das Gerät erholt sich nach einigen Sekunden und nimmt neue Verbindungen an. Es startet dabei
nicht neu — die Nachbarregister liefern anschließend unveränderte Werte.

Beim Vollscan aller 65535 Register war 32103 der **einzige** Timeout. Alle 64984 anderen nicht
belegten Adressen lieferten eine saubere Exception.

---

## 3. Warum ausgerechnet dieses Register

Von allen 32-Bit-Einträgen der Registerkarte ist 32102 der **einzige mit Typ-Code `0x24`
(float32)**:

| Register | Name | Typ-Code | 2. Hälfte einzeln lesbar |
|---|---|---|---|
| **32102** | bat_sample_power | **0x24 float32** | **nein — keine Antwort** |
| 32202 | grid_sample_power | 0x12 i16 | ja |
| 33000 / 33002 / 33004 / 33006 / 33008 / 33010 | Energiezähler | 0x04 u32 | ja |
| 36000 / 36100 / 36102 | warn/error codes | 0x04 u32 | ja |
| 37013 / 37015 / 37021 | error codes, PV-Jahresertrag | 0x04 u32 | ja |

Die Korrelation ist eindeutig, beruht aber auf genau einem Fall — es gibt nur diesen einen
float32-Eintrag.

---

## 4. Der Code

`Read_Serializer` @ `0x08055414` (v150, korrekte Basis 0x08004800; ursprünglich als `0x08050C14` notiert —
alte Ghidra-Analyse mit falscher Basis 0x08000000, Differenz 0x4800). Zuerst der Versatz innerhalb des
Eintrags, in Registern:

```c
uVar3 = param_2 - (uint)*param_4 & 0xffff;        // 32103 - 32102 = 1
```

Dann der Float-Zweig, `'$'` ist `0x24`:

```c
if ((char)param_4[4] == '$') {
    local_3c = *(float *)(uVar3 * 2 + iVar4 * 4 + *(int *)(param_4 + 2));
```

Mit `uVar3 = 1` und `iVar4 = 0` wird der Float von **Basis + 2** gelesen. Die Basis ist
`0x20014EB8` (Spalte `source_ptr_v150` der Registerkarte), gelesen wird also von
**`0x20014EBA`** — halbwort-, nicht wortausgerichtet.

Die Ganzzahl-Zweige rechnen mit derselben schiefen Adresse:

```c
else if (bVar2 == 4) {
    local_38 = *(uint *)(iVar4 * (param_4[4] & 0xf) + uVar3 * 2 + *(int *)(param_4 + 2));
}
```

Der Unterschied liegt nicht im Code, sondern in der erzeugten Instruktion: ein `uint`-Zugriff
wird zu `LDR`, und unausgerichtete `LDR` sind auf Cortex-M erlaubt, solange `UNALIGN_TRP` nicht
gesetzt ist. Der `float`-Zugriff wird zu `VLDR`, und **`VLDR` verlangt immer Wortausrichtung** —
sonst UsageFault.

Das erklärt beide Beobachtungen: das Ausbleiben jeder Antwort (die Task kommt nicht bis zum
Senden) und die Beschränkung auf genau diesen einen Eintrag.

**Nicht belegt** ist, was danach passiert — ob ein Fault-Handler greift, ein Watchdog die Task
neu startet oder die Verbindung anders verloren geht. Beobachtet ist nur, dass das Gerät nach
einigen Sekunden wieder Verbindungen annimmt.

---

## 5. Abgrenzung — was es nicht ist

Zwei naheliegende Erklärungen wurden geprüft und **widerlegt**:

**Es liegt nicht an der Lesemenge.** Die Kopierschleife am Ende des `FC03_Read_Handler` zählt
mit einem auf 8 Bit maskierten Zähler gegen eine 16-Bit-Schranke:

```c
for (uVar5 = 9; uVar5 <= uVar8; uVar5 = uVar5 + 1 & 0xff) { ... }
```

Bei `uVar8 > 255` liefe sie endlos, und `uVar8 = 9 + 2 × bediente Register` wäre ab 124
Registern erreicht. Am Gerät geprüft, Start 34000, Anzahlen 34/35/120/123/124/125/126: **alle
antworten sauber**, ab 126 mit Exception 3 (Prüfung `0x7d < uVar5`). Der Deskriptor-Lauf bricht
vorher an der ersten Lücke ab, `uVar8` wird nie groß genug. Die 8-Bit-Schleife bleibt ein
latenter Fehler, sie erklärt 32103 aber nicht.

**Es liegt nicht daran, dass mitten in einen Eintrag hineingelesen wird.** Alle anderen
`_w2`-Register sind ebenfalls zweite Hälften eines 4-Byte-Eintrags und lesen sich einzeln
problemlos (Abschnitt 3).

---

## 6. Auswirkung und Umgehung

Ein Client, der Register einzeln abfragt und 32103 dabei berührt, **verliert die Verbindung**.
Da das Gerät nur eine einzige Modbus-TCP-Session zulässt, trifft das auch alle anderen
Verbraucher, bis es sich erholt hat.

Umgehung, beide wirksam:

1. **32103 nie einzeln lesen.** Den 32-Bit-Wert immer ab 32102 mit Anzahl 2 holen. Das ist
   ohnehin die richtige Art, ihn zu lesen.
2. **32103 aus Registerlisten streichen.** Für einen Scanner oder ein Discovery-Verfahren ist
   das die sicherere Variante.

Für einen Herstellerbericht: der Float-Lesepfad muss den Versatz wortweise behandeln oder den
Wert über einen ausgerichteten Zwischenpuffer laden, wie es die Ganzzahl-Zweige faktisch tun.

---

## 6a. Status in v151 (Firmware 151, geprüft 2026-10-01)

**Geändert — Auswirkung behoben, Ursache im Code besteht latent fort.**

`Read_Serializer` selbst ist in v151 (`0x08055fe0`, 626 Byte) instruktionsidentisch zu v150 — der
Float-Zweig mit `VLDR` auf unausgerichteter Adresse ist unverändert vorhanden (siehe
`VNSD_FW_Bug_Report_EN_Read_Serializer.md`, Abschnitt „Status in v151"). Die Ursache selbst wurde also
**nicht** repariert.

Behoben ist stattdessen die **Auslösung**: der Descriptor-Eintrag für 32102 wurde von `float32` (Typ-Code
`0x24`, 4 Byte) auf `i16` (Typ-Code `0x12`, 2 Byte) geändert (`VNSD_Descriptor_Table_Diff_v150_vs_v151_de.md`,
Abschnitt 1). Damit endet der Eintrag jetzt bei Register 32103 statt bei 32104, und 32103 liegt für
`FC03_Read_Handler` (v150 `0x0802386c` → v151 `0x08023a00`, 382 Byte) außerhalb jedes Eintrags:

```c
if (start_offset <
    (uint)g_modbusRegDescTable[reply_write_offset].reg_addr +
    (uint)((int)(short)(g_modbusRegDescTable[reply_write_offset].type_code2 & 0xf) *
          (int)(short)(ushort)g_modbusRegDescTable[reply_write_offset].count) / 2) {
  if (start_offset < g_modbusRegDescTable[reply_write_offset].reg_addr) {
    return 2;                    // Exception 02, sauber beantwortet
  }
  ...
```

Ein Einzelzugriff auf 32103 durchläuft den Float-Zweig von `Read_Serializer` in v151 also gar nicht mehr —
statt keiner Antwort/toter Session liefert das Gerät jetzt eine reguläre Exception 02. Am Vollscan-Verhalten
(v150: einziger Timeout unter 65535 Adressen) ändert sich damit für 32103 konkret etwas; die zugrunde liegende
`VLDR`-Ausrichtungslücke im Serializer bleibt aber bestehen und würde bei jedem künftigen float32-Eintrag mit
ungeradem Byteversatz erneut zuschlagen. Nicht geprüft: ob ein solcher Eintrag in v151 existiert.

## 7. Querverweise

- `VNSD_FW_Bug_Report_EN_Read_Serializer.md` — Vorzeichenfehler im selben Serializer (32101)
- `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md` — falsches MBAP-Längenfeld bei Exceptions
- `VNSD_FW_Bug_44002_44003_Ruecklesewert_nach_Neustart_de.md` — Rücklesewert nach Neustart
- `VNSD_Netzabrisse_waehrend_Registerscan_2026-09-20_de.md` — davon unabhängige Verbindungsabrisse
- Messdaten: `Scan_Logs/register_scan_full_20260920_1154.csv` (lokal, Rohdaten), Zeile 32103
