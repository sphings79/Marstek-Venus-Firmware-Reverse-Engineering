# Venus D — BMS 118: SOH, Zyklenzähler und Entladestromgrenze

> 🇬🇧 English version: [VNSD_BMS_SOH_Cycles_Discharge_Limit_v118.md](VNSD_BMS_SOH_Cycles_Discharge_Limit_v118.md)

> **Hinweis (2026-09-30):** Flash-Adressen in diesem Dokument stammen aus der alten Analyse mit falscher Basisadresse (0x2800 zu niedrig). Gesamtanalyse mit korrekten Adressen: [VNSD_BMS_FW_v118_Vollanalyse_de.md](VNSD_BMS_FW_v118_Vollanalyse_de.md).

**Stand:** 2026-09-24
**Gilt für:** Venus D, BMS 118. SOH- und Zyklenlogik sind in 117.7 bytegleich vorhanden.
**Quellen:** Ghidra `/VNSD/BMS/BMS_118_…`, Rohimage, Dekompilat-Export `Dekompilate/VNSD_BMS_118/` (lokal, nicht im Repo).
Control-Seite: `/VNSD/Control/VNSD-0_app_0150_…`.

Die 117.7-Doku (§4) beschreibt den SOC-Algorithmus. Zwei Teile davon waren falsch gedeutet: die
angebliche OCV-Korrektur im Ruhezustand und die „Quick-OCV-Tabelle“. Beide gehören zur
SOH-Berechnung (§6).

---

## 1. Kernaussagen

1. **SOH ist reine Alterung nach Zyklenzahl.** Es wird nicht gemessen, sondern aus einer
   Tabelle mit 7 Stützstellen über die Zyklenzahl interpoliert. Die Tabelle liegt im
   Kalibrier-Flash hinter dem Image-Ende und ist nicht lesbar.
2. **Zyklen = Energiedurchsatz ÷ (2 × 2,56 kWh).** Laden und Entladen zählen beide. Die
   Zyklenzahl ist abgerundet und erscheint pro Pack in Modbus **34x03**.
3. **SOH verlässt das BMS nicht.** Es gibt kein CAN-, RS485- oder Modbus-Feld dafür, nur die
   Debug-Shell (`Soh:`). Im Cloud-Upload des Control steht `soh=0`. Ein SOH in HA muss die
   Integration selbst rechnen.
4. **Das SOH steuert die Kapazität der Coulomb-Zählung:** Nennkapazität 50 Ah × SOH. Sinkt das
   SOH, bewegt sich der Pack-SOC pro Ah schneller.
5. **Die Entladestromgrenze hängt von der niedrigsten Zellspannung ab, nicht vom SOC.**
   Die Ladestromgrenze hängt dagegen vom angezeigten SOC ab.
6. **Register 32105** (`bat_energy`) ist die feste Nennenergie: Packanzahl × 50 Ah × 51,2 V,
   bei 7 Packs 17,92 kWh (`CAN_TX_PF2_Capacity`). Sie ist weder SOH-bereinigt noch die Restenergie.
7. **Die Entladung stoppt im Control am Gesamt-SOC**, sobald dieser die Backup-Reserve
   erreicht (Standard 12 %). Die Pack-SOCs spielen dafür keine Rolle.

---

## 2. SOC-Struktur (Basis `0x20004A38`)

Die Belegung ist aus der Debug-Ausgabe `CLI_Print_SOC_Status` (0x080138E0) und den Schreibstellen
abgeleitet. Sie korrigiert die Tabelle in `VNSD_BMS_FW_Analyse_v117.7_de.md` §4.5.

| Offset | Adresse | Feld | Einheit |
|---|---|---|---|
| +0x00 | `0x20004A38` | real_soc | 1e-4 % |
| +0x04 | `0x20004A3C` | show_soc (angezeigt, geht in Modbus 34x02) | 1e-4 % |
| +0x08 / +0x0C | | max_soc / min_soc | 1e-4 % |
| +0x10 | | target_soc | 1e-4 % |
| +0x24 | `0x20004A5C` | **Zyklenzahl** (u16) | 1 |
| +0x28 | `0x20004A60` | **SOH** (u32) | **1e-4 %**, 1.000.000 = 100 % |
| +0x2C | `0x20004A64` | **Energiedurchsatz** Laden + Entladen (u32) | **0,1 kWh** |
| +0x30 / +0x32 | | max. Lade- / Entladestrom | 0,1 A |
| +0x34 / +0x36 | | max. / min. Zellspannung | mV |
| +0x38 | | Temperatur | 0,1 °C |
| +0x3E | | Strom | 0,1 A |
| +0x40 | | Pack-Spannung („Tal Vol“) | 0,1 V |

**Korrekturen zu 117.7 §4.5:** Der Durchsatz ist dort als `total_capacity` (Kapazität in kWh)
geführt. Das SOH ist dort mit ÷10 angegeben, die Einheit ist aber 1e-4 %. Die Debug-Shell teilt
tatsächlich durch 10 und würde bei 100 % „100000.0“ ausgeben. Das ist ein Anzeigefehler der
Firmware, nicht der Wert.

---

## 3. Energiedurchsatz

`Coulomb_Counting` liefert alle 100 ms (SOC-Task: `SOC_Algorithm_Entry(100)`, danach
`vTaskDelay(100)`):

```
q  = I[0,1 A] × dt[ms] / 36                → µAh
dE = |q| × U[0,1 V] / 100                  → 10 µWh
```

`Charge_Energy_Calc` und `Discharge_Energy_Calc` summieren `dE` in je einem Akkumulator
(`0x200029F4` Laden, `0x200029FC` Entladen). Ganze 10⁷-Einheiten (= 0,1 kWh) wandern jeweils in
`+0x2C`, der Rest bleibt im Akkumulator. **Beide Richtungen erhöhen denselben Zähler.**

Gesichert wird `+0x2C` zusammen mit real/show/max/min-SOC in einem **EEPROM-Ringpuffer ab
`0x400`**, 50 Slots à 20 Byte (Verschleißausgleich; `SOC_EEPROM_LoadCalibration` sucht beim Start
den jüngsten Slot). `SOC_EEPROM_SaveIfChanged` schreibt, sobald sich real_soc um mehr als 1 %
oder der Durchsatz um mindestens 0,1 kWh geändert hat.

---

## 4. Zyklenzahl — `CycleCount_From_Throughput` (0x0800FAE4)

```c
zyklen = (durchsatz / 10.0) / (2.56 × 2.0);   // kWh / (2 × Nennenergie), abgerundet
```

Die 2,56 kWh sind als Float-Konstante im Image fest einkodiert (`0x0800FB34`), keine
Konfiguration. Ein Zyklus sind also 5,12 kWh Durchsatz, je 2,56 kWh rein und raus.

Die Zyklenzahl wird **nur beim Übergang in den Ruhezustand** neu berechnet (§6), zusätzlich
einmal beim Start. Sie ist eine ganze Zahl und springt deshalb stufenweise. Übertragen wird sie
über die Pack-Struktur (+0x24 „Cyc Cnt“) bis zu Modbus **34x03**.

Feldwerte am 24.09.2026: P1 9, P2 30, P3 28, P4 33, P5 32, P6 75, P7 69. Das entspricht grob
46 bis 384 kWh Durchsatz pro Pack.

---

## 5. SOH — `SOH_From_Throughput` (0x08010324) / `SOH_Table_Lookup_ByCycles` (0x08011534)

```c
zyklen = CycleCount_From_Throughput(durchsatz);
if (zyklen gültig) soh = SOH_Table_Lookup_ByCycles(zyklen);
soh = min(soh, 1.000.000);                    // höchstens 100 %
```

Die Tabelle hat 7 Einträge zu je 8 Byte, `{zyklen_schwelle u32, soh u32}`, bei **`0x0801B874`**.
Das ist hinter dem Image-Ende `0x08019FFF`, im Kalibrier-Flash. Zwischen den Stützstellen wird
linear interpoliert. Unter der ersten Stützstelle gilt deren Wert, über der letzten der Wert des
letzten Eintrags. Die Tabelle stammt vermutlich aus der Zellspezifikation des Herstellers.
Ohne SWD-Dump bleibt sie unbekannt.

Verwendung:

- `Coulomb_Counting`: `Kapazitätsfaktor = soh / 1e6`, also effektive Kapazität
  = 50 Ah × SOH. Bei Faktor 0 zählt die Funktion nicht.
- `SOC_Init_Capacity_Params`: legt beim Start SOH in % (u16, `0x20004ADE`) und daraus
  abgeleitete Kapazitätswerte an. `0x20004ADE` wird danach nie mehr aktualisiert und von keiner
  Sende-Funktion gelesen.

**Nicht nach außen gegeben:**

- `PerPack_Struct_Builder`, `CAN_TX_PerPack_10Msgs` und `RS485_Pack_Telemetry_Response`
  enthalten kein SOH.
- Die Pack-Struktur des Control hat bei +0x5A ein Feld `bat_soh` (Debug-Ausgabe
  `bat soh =%d`). Kein CAN-Frame befüllt es, und es ist keinem Modbus-Register zugeordnet
  (Pack 1: `0x2001500E`).
- Der Cloud-Upload hat den Schlüssel `soh`, im Mitschnitt mit dem Wert 0
  (`Control_FW/VNSD_Cloud_Upload_Payload_v150_de.md`).
- BLE: `BLE_Build_BMS_Data_Response` (Control 150, 0x0800B4F4) legt als Wort 5 genau dieses
  nie befüllte Pack-1-Feld +0x5A (`0x2001500E`) in die Antwort. Das BLE-SOH ist damit ebenfalls 0.
  Wort 11 derselben Antwort ist die **Restenergie** = SOC (32104, 0,1 %) × Nennenergie
  (32105, Wh) ÷ 10000 in 10 Wh. Auch sie ist nicht SOH-bereinigt; HA „gespeicherte Energie“
  13,08 kWh bei 73 % × 17,92 kWh passt dazu.

**Folge für die HA-Integration:** Ein SOH lässt sich aus Modbus nur nachbilden: Zyklen pro Pack
aus 34x03, dazu eine eigene Alterungskurve. Die 7 Stützstellen der Firmware kennen wir nicht.
Eine Kapazitätsmessung (entnommene Ah zwischen zwei Vollerkennungen) ist unabhängig davon und
wäre das bessere Maß, weil die Firmware selbst nichts misst.

---

## 6. Wann SOH und Zyklen neu berechnet werden

`SOC_Algorithm_Orchestrator`, Zustand 3 (Ruhe), nur beim **ersten** Durchlauf nach Laden oder
Entladen:

```c
if (vorheriger_zustand != 3) {
    RTC-Zeitstempel → EEPROM-Adresse 0xFC02 (10 Byte);
    soh    = SOH_From_Throughput(durchsatz);        // bis 2026-09-24 „OCV_Correction“
    zyklen = CycleCount_From_Throughput(durchsatz); // bis 2026-09-24 „OCV_Voltage_To_Index“
}
```

`SOC_OCV_Bounds_Check_And_Recalibrate` (0x08013E34), das der Orchestrator ebenfalls in jedem
Durchlauf aufruft, ist toter Code. Sein Auslöse-Flag `0x200029CA` wird nirgends gesetzt
(`VNSD_BMS_FW_Analyse_v118_de.md` §4.7).

Die echte Ruhe-Rekalibrierung ist davon getrennt. Nach 3.600.000 ms (1 h) Ruhe prüft der
Orchestrator min/max Zellspannung gegen Schwellen aus der Tabelle bei `0x0801B8D4` und ruft
gegebenenfalls `Full_Recalibration`. Das bleibt wie in 117.7 §4.4 beschrieben.

Beim Start: `SOC_Init_SOH_Cycles_From_Throughput` (0x0801351E, bis 2026-09-24
`SOC_Calculate_OCV_From_Voltage`) lädt den Durchsatz aus dem EEPROM-Ring und berechnet beides
einmal.

---

## 7. Strombegrenzung

### 7.1 Laden — `Max_Charge_Current_Calc` (0x0800FE60)

Zeile = Temperaturindex (18 Stufen, Schwellen `0x0801B97E`), Spalte = SOC-Zone (12 Zonen,
Schwellen `0x0801B9A2`), Matrix `0x0801B9BC`. Maßgeblich ist **show_soc** in ganzen Prozent.
Eintrag × 0,5 A, höchstens 50 A. Bei show_soc = 100 % → 0 A. Feldwerte:
`VNSD_BMS_Pack_Rotation_Taper_v118_de.md` §8.

### 7.2 Entladen — `Max_Discharge_Current_Calc` (0x0800FF58)

- **Zeile:** Temperaturindex wie beim Laden.
- **Spalte:** eine von 5 Stufen der **niedrigsten Zellspannung** (`+0x36`), nicht des SOC.
- Es gibt zwei Schwellentabellen: Temperaturindex < 5 → `0x0801BB76`, sonst `0x0801BB6C`.
- Matrix `0x0801BB80`, 5 Spalten je Zeile. Eintrag × 0,5 A, höchstens 50 A.
- Alle Tabellen liegen im Kalibrier-Flash, keine ist lesbar.

`SOC_Current_Bracket_Lookup` hat eine **Hysterese**: Nach unten wechselt die Stufe sofort, nach
oben erst, wenn die Zellspannung mindestens **100 mV** über der unteren Grenze der aktuellen
Stufe liegt. Unter der zweiten Schwelle gilt Stufe 0, vermutlich 0 A. Das ist die
spannungsbasierte Entladeuntergrenze. Die Temperaturachse (`Temperature_Index_Lookup`,
18 Stufen) hat ebenfalls eine Hysterese.

Für die Restzeit bis leer heißt das: Solange die Zellspannung nicht in die unteren Stufen
fällt, begrenzt das BMS die Entladung nicht (bis 50 A). Am Ende eines Packs kann die Leistung
spannungsabhängig abfallen. Im Feld ist das nicht gemessen.

`Master_Build_DischargeCurrentLimit` nimmt den Wert des aktiven Packs **ohne**
Minimum-Bildung (`VNSD_BMS_FW_Analyse_v118_de.md` §11.2). Entladespannungsgrenze fest 48,0 V.

---

## 7b. Voll- und Leererkennung (SOC-Sprünge)

`Charge_SOC_Integration` / `Discharge_SOC_Integration` (Zustand Laden bzw. Entladen). Maßgeblich
ist die **höchste bzw. niedrigste Zellspannung** des Packs, nicht der Coulomb-Zähler:

| Bedingung | Wirkung |
|---|---|
| Max Cell ≥ **3600 mV** (Temperatur < 0 °C: 3700 mV), `SOC_Is_Full_ByMaxCell` | SOC sofort **100 %** (`SOC_Set_Full_Charge_1000000`) |
| Max Cell 3501–3549 mV | SOC wird spannungsproportional Richtung **96 %** gezogen |
| Max Cell 3551–3599 mV | SOC wird spannungsproportional Richtung **100 %** gezogen |
| Max Cell < 3500 mV | Coulomb-Zählung; show_soc läuft einem abweichenden target_soc mit halber bzw. doppelter Geschwindigkeit nach |
| vor der Vollerkennung | show_soc höchstens **99,0 %** |
| Min Cell ≤ **2750 mV** (Temperatur < 0 °C: 2420 mV), `SOC_Is_Empty_ByMinCell` | SOC sofort **0 %** |
| Min Cell < 2900 mV | `SOC_Voltage_Drop_Compensator` zieht den SOC nach unten; vor der Leererkennung bei 1,0 % gehalten |

Das erklärt den Feldbefund vom 22.09.: 97,2 → 98,6 → 100 % in 30 s bei 56,8 V Pack-Spannung
(≈ 3,55 V je Zelle im Schnitt, höchste Zelle an der 3,6-V-Grenze). **Für eine
Restzeit-Anzeige heißt das:** Die letzten Prozent vor 100 % (und vor 0 %) laufen nicht linear
zur Energie. Das Ende bestimmt die Zellspannung, nicht die Ah-Bilanz.

Die davon unabhängigen Schutzgrenzen (Überspannung 3700–3900 mV → SOC 100, Unterspannung
2500/2300 mV → SOC 0) stehen in `VNSD_BMS_FW_Analyse_v118_de.md` §4.6.

`BMS_Check_FullCharge_Condition` (0x08006F34) war falsch benannt und heißt jetzt
`BMS_Measurements_Valid_Check`. Die Funktion setzt Bit 5 in `0x20004B28`, sobald 100
aufeinanderfolgende Messungen gültig waren (alle Zellen ≥ 1000 mV, NTC zwischen −40 und 120 °C).
Erst danach läuft der SOC-Algorithmus überhaupt.

**Anzeige-SOC nach Vollerkennung** (`SOC_Display_And_FullFlag_Update`): Das Voll-Flag hält die
Anzeige auf 100,0 %, bis entladen wird oder der SOC unter 99,5 % fällt.

## 7c. Zellbalancing — nur oben, nur beim Laden

`KA495XX_Cell_Balance_Decision` (0x08014C98) mit der Zellspannungsstruktur `0x20004068`:
Zellen ab +0x0C, min. Zelle +0x54, max. Zelle +0x58, alle in mV.

| Bedingung | Wirkung |
|---|---|
| max − min < **30 mV** oder `s32@0x20004070` ≤ 1000 | kein Balancing, Maske gelöscht |
| Zelle ≥ **3501 mV** und ≥ 30 mV über der niedrigsten (entprellt) | Zelle wird balanciert |
| Zelle < 3500 mV und < **20 mV** über der niedrigsten (entprellt) | Balancing der Zelle endet |
| alle Zellen würden balanciert | keine (Maske 0) |

`0x20004070` berechnet `KA495XX_Compute_Power_And_Energy` aus einem BMIC-Messwert. Vermutlich ist
es der Ladestrom in mA, also Bedingung > 1 A. Das ist nicht sicher belegt.

`KA495XX_Cell_Balance_StateMachine` (0x08014D90) schaltet **abwechselnd die ungeraden und die
geraden Zellen** (Maske `0x20003C70 + 0x3B4`, je 4 Durchläufe). Es sind nie zwei benachbarte
Zellen gleichzeitig aktiv. Die Balancing-Bits gehen in die BMIC-Register `0x15`/`0x16`,
gesteuert über Register `0x0C` und `0x17`. Die Funktion `APP_CAN_Send_With_Validation` ist dabei
**kein CAN-Versand**, sondern ein BMIC-Registerzugriff (Register, Wert, Maske). Der Name ist
falsch, aber nicht angefasst.

**Folge:** Balanciert wird nur, wenn eine Zelle über 3,5 V liegt. Das ist bei LFP nur ganz oben,
in den letzten Minuten vor 100 % (Feld 22.09.: ≈ 3,50–3,55 V ab ~97 %). Ein Pack, der selten
voll wird, balanciert praktisch nie. Regelmäßige Vollladungen jedes Packs sind die einzige
Stellschraube dafür.

Feldstand 24.09. bei ~70 % SOC: 2–3 mV Zelldifferenz **innerhalb** jedes Packs (HA
`zellspannungs_differenz`). Die 127 mV von Omnibattery (`cell_delta`) sind die Spanne **über alle
Packs** und bilden die unterschiedlichen Pack-SOCs ab, keine Zell-Unbalance. Bei 70 % sagt eine
kleine Zelldifferenz wenig; aussagekräftig ist sie erst oben, ab ~3,4 V.

## 8. Control: Backup-Reserve und Leistungsdeckel

`Inverter_Power_Setpoint_Calc` (Control 150, 0x08006168) wendet auf jeden Sollwert an:

| Bedingung | Wirkung |
|---|---|
| Sollwert > 0 (Entladen) und Reserve `u8@0x2000012C` ≥ Gesamt-SOC (`0x20014F94`, 32104) | Sollwert = 0 |
| Sollwert > `u16@0x2000012F` (44003) | auf 44003 gedeckelt |
| Sollwert < −`u16@0x2000012D` (44002) | auf −44002 gedeckelt |
| Sollwert < P_bms | auf P_bms (`Control_FW/VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md` §2) |
| Flag `0x200002BA` gesetzt und Sollwert > 0 | Sollwert = 0 |

Die Reserve ist `100 − App-DoD`, Standard 12 %, App-Bereich DoD 30–88 %
(`Modbus_RS485_TCP/VNSD_Register_Persistenz_RAM_vs_EEPROM_v150_de.md`). Ob die Grenze bei SOC =
Reserve genau greift, lässt der fp64-Vergleich im Dekompilat nicht sicher erkennen.

**Abgleich mit dem Feld (20.09.):** Omnibattery hat bei **Pack**-SOC 12,0 % abgeschaltet. Der
Gesamt-SOC lag da bei etwa 21 % (P3/P4/P6 bei 28,9 %). Die Control-Reserve kann es also nicht
gewesen sein. Das passt zu Omnibattery als Ursache.

---

## 9. Ghidra-Umbenennungen (2026-09-24)

| Programm | Adresse | vorher | nachher |
|---|---|---|---|
| 118 | `0x08010324` | `OCV_Correction` | `SOH_From_Throughput` |
| 118 | `0x0800fae4` | `OCV_Voltage_To_Index` | `CycleCount_From_Throughput` |
| 118 | `0x08011534` | `Quick_OCV_Lookup` | `SOH_Table_Lookup_ByCycles` |
| 118 | `0x0801351e` | `SOC_Calculate_OCV_From_Voltage` | `SOC_Init_SOH_Cycles_From_Throughput` |
| 118 | `0x080138e0` | `FUN_080138e0` | `CLI_Print_SOC_Status` |
| 117.7 | `0x080102b4` | `OCV_Correction` | `SOH_From_Throughput` |
| 117.7 | `0x0800fa74` | `OCV_Voltage_To_Index` | `CycleCount_From_Throughput` |
| 117.7 | `0x080114c4` | `Quick_OCV_Lookup` | `SOH_Table_Lookup_ByCycles` |
| 117.7 | `0x080134ae` | `SOC_Calculate_OCV_From_Voltage` | `SOC_Init_SOH_Cycles_From_Throughput` |
| 118 | `0x0800fd4c` | `SOC_Check_Max_Capacity_Valid` | `SOC_Is_Full_ByMaxCell` |
| 118 | `0x0800fd7c` | `SOC_Check_Min_Capacity_Valid` | `SOC_Is_Empty_ByMinCell` |
| 118 | `0x08006f34` | `BMS_Check_FullCharge_Condition` | `BMS_Measurements_Valid_Check` |
| 117.7 | `0x0800fcdc` | `SOC_Check_Max_Capacity_Valid` | `SOC_Is_Full_ByMaxCell` |
| 117.7 | `0x0800fd0c` | `SOC_Check_Min_Capacity_Valid` | `SOC_Is_Empty_ByMinCell` |

---

## 10. Offene Punkte

- SOH-Tabelle (`0x0801B874`), Entlade-Matrix (`0x0801BB6C`–`0x0801BB80` ff.) und
  Temperaturschwellen: nur per SWD-Dump.
- Welche Funktion den Cloud-Schlüssel `soh` befüllt (der Formatstring wird über eine Tabelle
  referenziert). Nach dem BLE-Befund vermutlich dasselbe leere Feld.
- Control-Pack +0x50/+0x54 (`bat_charge_power` / `bat_discharge_power`, u32) befüllt
  `CAN_Store_PerPack_GrpExt_0x50` aus einem Zusatz-Frame, den BMS 118 **nicht** sendet. Die Felder
  bleiben also 0, Energiezähler pro Pack gibt es nicht. Welche BMS-Version den Frame sendet,
  ist offen.
