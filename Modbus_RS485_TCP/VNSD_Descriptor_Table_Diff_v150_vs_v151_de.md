# Modbus-Descriptor-Tabelle — Diff v150 ↔ v151

> 🇬🇧 English version: [VNSD_Descriptor_Table_Diff_v150_vs_v151_de.md](VNSD_Descriptor_Table_Diff_v150_vs_v151_de.md)

**Gerät:** Marstek Venus D (VNSD-0), Control-Firmware
**Stand:** 2026-09-29
**Englische Fassung:** `VNSD_Descriptor_Table_Diff_v150_vs_v151_de.md`
**Daten:** `VNSD_Descriptor_Table_Decoded_v150.csv`, `VNSD_Descriptor_Table_Decoded_v151.csv`

## Kurzfassung

| Kennzahl | v150 | v151 |
|---|---|---|
| Einträge | 246 | 246 |
| Registerbereich | 30000–38014 | 30000–38014 |
| Register nur in einer Version | – | – |
| Typ-/Größen-Änderungen | – | **1 (32102)** |
| Skalen-/Count-Änderungen | – | 0 |
| Tabellenadresse (SRAM) | 0x20000354 | 0x2000034C |
| Quelle von 30010 | globale Variable 0x20000144 | Feld im Telemetrie-Struct (0x200176D4 = Basis + 0x38) |

## 1. Die einzige Typänderung: 32102

| | v150 | v151 |
|---|---|---|
| Typ | `float32` (0x24), Elementgröße 4 | `i16` (0x12), Elementgröße 2 |
| belegt | 32102 + 32103 | nur 32102 |
| Quelle | 0x20014EB8 (gleiche wie 30001) | 0x200176B8 (gleiche wie 30001) |

FC03-Handler und Read-Serializer sind **instruktionsidentisch** zu v150. Die Folgen ergeben sich allein aus der Tabelle:

| Abfrage | v150 | v151 |
|---|---|---|
| 32103 einzeln | UsageFault (`VLDR` auf 0x20014EBA, nicht ausgerichtet), Modbus-Session stirbt | Exception 02 (Adresse liegt in keinem Eintrag) |
| Block über 32103 (z. B. 32100–32109, bisheriger Workaround 32102–32104) | funktioniert | **ganzer Block wird mit Exception 02 abgelehnt** |
| 32102 | Float-Darstellung einer i16-Größe (sinnlos) | derselbe Wert wie 30001 |

Nach der Suchlogik des FC03-Handlers liefert jede Registeradresse zwischen zwei Einträgen Exception 02. In v151 gilt das nun auch für 32103.
Die HA-Integration (`d.yaml`) liest aus dem Bereich nur 32104 und ist nicht betroffen.

## 2. Unverändert fehlerhaft

- **32101** (i16, ÷10): Der Read-Serializer rechnet das Vorzeichen weiterhin vorzeichenlos (Code identisch zu v150), negative Ströme bleiben falsch.

## 3. Verschobene Quellzeiger (Layout, keine Semantikänderung)

| Delta | Register | Deutung |
|---|---|---|
| −8 | 30030, 30202–30203, 30205, 30211, 30213, 30300, 30303, 37001–37003, 38000–38014 | Kern-Globals um 8 Byte nach unten |
| ±0 | 30200–30201, 30212, 37000 | unverändert (u. a. Flash-Konstante 0x08010000) |
| +0x2800 | 30000–30007, 31000, 32102, 32200–32202, 32204, 32300–32302, 33000–33010 (gerade), 35000–35002, 35100, 36000, 36100, 36102, 37004, 37013, 37015 | Telemetrie-Struct verschoben (v150 0x20014EAx → v151 0x200176Ax) |
| +0x2820 | 177 Register (30020–30040, 30100–30110, 32100–32114, alle Pack-Blöcke 34x00–34x18, 35010–35112, 37005–37024 …) | zweiter Teil des Telemetrie-/BMS-Blocks |
| +0x2822/+0x2824/+0x2828 | 30301, 30304, 30350 / 30302 / 30400 | späte SRAM-Puffer |
| **+0x17590** | **30010** | **neue Quelle**, s. Kurzfassung |

Die Verschiebung um 0x2800 entspricht exakt der Heap-Vergrößerung (0x11800 → 0x14000). Das spricht dafür, dass der Heap im .bss vor diesen Variablen liegt (Schlussfolgerung, nicht einzeln belegt).

## 4. Methode

Die Tabelle ist ein Keil-RW-Global, das LZ77-komprimiert im Flash liegt (Scatter-Region 0 → 0x20000000, 0x2F00 Byte). v151 wurde mit einem Entpacker dekodiert, der den Firmware-Code `__decompress` (0x08005FBE) exakt nachbildet. Das ältere Skript `marstek_descriptor_unpack.py` (lokal, nicht im Repo) behandelt die Erweiterungsbytes (`(op & 7) == 0` bzw. `(op >> 4) == 0`) nicht, liefert für die Descriptor-Tabelle aber dasselbe Ergebnis (246/246 identisch).
