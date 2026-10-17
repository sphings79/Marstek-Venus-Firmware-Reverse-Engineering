# Cloud-Upload: Welche Variable speist welchen Schluessel, Vergleich ueber die Control-Versionen

> 🇬🇧 English version: [Cloud_Upload_Key_Sources_Version_Comparison.md](Cloud_Upload_Key_Sources_Version_Comparison.md)

**Stand:** 2026-10-15. Erweitert `VNSD_Cloud_Upload_Schluesseltabelle_v151_de.md` (Bedeutung der Schluessel auf Venus D Control 151) auf alle dekompilierten Control-Firmwares.

## 1. Umfang und Methode

Zehn Control-Firmwares: VNSD 147, 149, 1492, 150, 151; VNSA 150, 1509, 151; VNSE3 150, 151.
In jeder Version ist der Upload ein einziger `sprintf` in `Cloud_UploadTelemetryData`. Die Argumente wurden je Version aus dem Assembler-Listing rekonstruiert (der Decompiler zeigt nur die ersten), dem Offset im Telemetrie-Verlaufssatz zugeordnet und dann dem Code, der das Feld fuellt: `MQTT_Telemetry_Struct_Builder` fuer Einzelwerte, `MQTT_Telemetry_String_Formatter` fuer die Kommalisten. Ergebnis je Schluessel: die Live-Variable, ihre Breite und ihr Vorzeichen. Danach Vergleich mit Venus D 151, der Version, auf der die Bedeutungen geprueft wurden.

## 2. Ergebnis in einem Satz

**Fast jeder Schluessel liest in allen zehn Versionen und auf allen drei Modellen dieselbe Variable mit derselben Umrechnung.** Die Bedeutung uebertraegt sich also; abweichend sind nur die Menge der Schluessel und wenige Felder.

## 3. Gesendete Schluessel je Version

| Version | Schluessel | Fehlt gegenueber VNSD 151 |
|---|---|---|
| VNSD 151, VNSA 1509, VNSA 151 | 76 | nichts |
| VNSD 1492, VNSD 150, VNSA 150 | 74 | `p_sta`, `p_pow` |
| VNSD 149 | 71 | `p_sta`, `p_pow`, `bt_p`, `ival`, `soh` |
| VNSD 147 | 69 | wie 149, dazu `bkd`, `ip` |
| VNSE3 151 | 70 | `pv`, `sk`, `mv`, `me`, `ma`, `fu` (kein MPPT-/Pack-Block) |
| VNSE3 150 | 68 | wie VNSE3 151, dazu `p_sta`, `p_pow` |

`dn` ist in jeder Version ein Literal und entspricht der Firmware-Version (147, 149, 1492, 150, 151, 1509).

## 4. Schluessel mit anderer Quelle

| Schluessel | Wo | Unterschied |
|---|---|---|
| `vc` | VNSE3 150, 151 | Nicht die Max/Min-Statistik von sechs Packs, sondern eine Liste mit **16 Werten: Spannung von Zelle 1 bis 16 des Packs 1** (`g_bmsPack0CellVoltageArray[0..15]`). Einheit als mV angenommen wie bei den uebrigen Zellspannungen (am Geraet nicht geprueft). |
| `mq` | VNSD 147 | Das echte Flag `g_quectelWifiMqttReadyFlag`. Ab Control 149 schreibt der Builder die Konstante 1, `mq` ist dort immer 1 und sagt nichts ueber die Verbindung. |
| `bke` | VNSD 147 | Nur ein Wert: `g_energyCat3MonthWh` (Netzbezug des Monats, im Sinn von `bkd` aus 151). Ab 149: `bke` = Kategorie 4 Tag/Monat/Gesamt, `bkd` = Kategorie 3 Tag/Monat/Gesamt. |
| `mt` | VNSD 147, 1492, 150 | Quelle heisst `g_meterSourceChangeFlag`; in 149 und 151 `g_meterDataFreshFlag`. Gleiche Stelle im Verlaufssatz, gleiche Nachbarn: sehr wahrscheinlich eine bei der Analyse umbenannte Variable, keine Aenderung im Code. |

Alles andere aus der Tabelle `VNSD_Cloud_Upload_Schluesseltabelle_v151_de.md` (alle Einzelwerte, `tc`, `cp`, `ce`, `ms`, `pv`, `sk`, `fu`, `im`, `bkd`, `bt_p`) hat in jeder Version, die den Schluessel sendet, dieselbe Quelle. Unterschiede in lokalen Variablennamen des Decompilers (`uVar1` gegen `uVar2`) sind keine Unterschiede im Code.

## 5. Grenzen

- `pw`, `bl`, `as` liessen sich in VNSD 147 nicht verfolgen (die Funktion ist dort anders uebersetzt); in den anderen neun Versionen haben sie dieselbe Quelle wie in 151.
- Verglichen wurde der Code, keine Live-Daten. Einheiten und Skalierungen sind nur auf Venus D 151 geprueft; bei den anderen Modellen folgen sie aus der gleichen Umrechnung im Code.
- `hd` ist weiter nicht zugeordnet.
- Die Binaries weiterer Versionen im Firmware-Archiv (VNSA 1508/149/1487/148, VNSE3 149/148/1476/147/144, HMG-50 153–156) sind noch nicht dekompiliert und nicht erfasst.
