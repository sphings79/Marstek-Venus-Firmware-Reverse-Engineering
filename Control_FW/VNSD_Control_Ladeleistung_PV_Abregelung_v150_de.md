# Venus D — Control 150: Ladeleistung aus dem BMS-Limit und PV-Abregelung

> 🇬🇧 English version: [VNSD_Control_Charge_Power_PV_Curtailment_v150.md](VNSD_Control_Charge_Power_PV_Curtailment_v150.md)

**Stand:** 2026-09-24
**Gilt für:** Venus D (VNSD-0), Control/EMS 150 (`VNSD-0_app_0150_0805_115146.bin`).
Abweichungen zu D 147/149.2 stehen in §5, Venus A und E3 in
`Methodik_und_Meta/Geraetevergleich_VNSD_VNSA_VNSE3_de.md` §3f.
**Gegenstück BMS-seitig:** `BMS/VNSD_BMS_Pack_Rotation_Taper_v118_de.md`.

Quellenlage: §1–§4 in dieser Sitzung gegen das Ghidra-Projekt und das Rohimage geprüft (Literale
aufgelöst). Die Versionsvergleiche in §5 und die Aussage zu `charge_cutoff_soc` stammen aus der
Analyse vom selben Tag und sind hier **nicht erneut** geprüft.

---

## 1. Kernaussagen

1. Der Control rechnet das Ladestromlimit des BMS in eine Leistung um:
   **P_bms = 32100 × 32106** (Spannung × Stromlimit). Beim Laden begrenzt dieser Wert jeden
   Sollwert, auch die Zwangsladung per 42020.
2. Eine eigene Pack-Auswahl nach SOC gibt es im Control nicht. Die macht das BMS.
3. **Bei positiver AC-Leistung regelt der Control die PV auf das Ladelimit des BMS ab**
   (`min(P_bms, 2500 W)`). Im Top-of-Charge-Taper (≈ 1270 W bzw. ≈ 500 W, s. BMS-Doku §8)
   liegt die PV-Obergrenze damit entsprechend niedrig, solange „Überschuss einspeisen“ nicht
   aktiv ist. Wie viel PV das im Feld kostet, ist nicht gemessen.
4. Wer 42011 schreibt, setzt damit **auch 42010 (force_mode) auf 3**. Modus 3 fährt dann mit
   festen ±2500 W auf den Gesamt-SOC 32104 zu.

---

## 2. Umrechnung Limit → Leistung

### 2.1 `Power_Limit_Clamp` (0x08013234)

```c
return −( u16@0x20014F8E × 0.01   // 32100 bat_volt, 10 mV
        × u16@0x20014FA0 × 0.1 ); // 32106 bzw. 35111 chrg_curr_limit, 0,1 A
```

Negativ bedeutet Laden (Vorzeichenkonvention des Sollwerts). Das Stromlimit ist direkt der Wert,
den das BMS aus `Master_Build_ChargeCurrentLimit` per CAN PF3 schickt (Pack-Tabelle, höchstens
50 A, s. BMS-Doku §8).

### 2.2 `Battery_Charge_Power_Limiter` (0x080043FC)

Aufgerufen aus `Inverter_Power_Setpoint_Calc` (0x08006168, Aufruf bei 0x08006176). `param` ist
der Sollwert (negativ = Laden).

```c
if (bms.flags@+0x17 & 1)                  // force_chrg vom BMS
    return max(param, P_bms);              // Ladung höchstens P_bms
if (param < 0 && irgendein MPPT ≥ 10,1 W) {  // 0x20014F58/5E/64/6A = 30037–30040, 0,1 W
    pv    = Summe MPPT1–4;
    bound = pv + max(P_bms, −2500);
    if (bound < 0) param = max(param, bound);   // Netzanteil = |P_bms| − PV
    else           param = 0;                   // PV deckt das Limit: kein Netzbezug
}
return param;
```

Unabhängig davon klemmt `Inverter_Power_Setpoint_Calc` das Ergebnis danach noch einmal nach
unten: `if (soll < P_bms) soll = P_bms`. Diese Grenze gilt immer, mit und ohne PV.

Effektiv lädt die Batterie mit `min(PV + Netzanteil, |P_bms|)`. Das gilt auch bei Zwangsladung
über 42020, denn beide Begrenzungen sitzen hinter jeder Sollwertquelle.

---

## 3. PV-Obergrenze — `Grid_Export_Power_Limiter` (0x0801F200)

Läuft alle 4 Ticks, unabhängig vom Betriebsmodus. Rechnet die PV-Obergrenze und schickt sie per
`Inverter_CAN_PowerCmd_Send` an den Wechselrichter (§3.2).

### 3.1 Berechnung

```c
lim  = min(−P_bms, 2500);
cap  = −P_bms;                             // nochmals, wenn BMS +0x17 Bit 1
if (!(bms.flags@+0x16 & 1)) cap = 0;       // BMS gibt Laden nicht frei
ac   = s16@0x20014EB4;                     // 30006, AC-Leistung, > 0 = Einspeisung

if (surplus_flag@0x20000285)               // „Überschuss einspeisen“ aktiv
    out = u16@0x2000012F;                  // 44003
else if (ac ≥ 1)                           // Einspeisung
    out = lim;                             // ← PV auf das Ladelimit abgeregelt
else
    out = min(cap, ac + lim + 300);
out = min(out, 4000);
```

### 3.2 Versand

`Inverter_CAN_PowerCmd_Send` (0x08005B54, bis 2026-09-24 `Inverter_RS485_Command_Send`) baut
die ID mit `CAN_Build_Arbitration_ID(2, 1, 2, 0/1)` und schreibt 4 + 8 Byte. Das ist ein
**CAN**-Versand an Zielklasse 2, kein RS485.

| Zustandsbyte `0x200002B4` (Modbus 40000/45000) | gesendet |
|---|---|
| 0 | `(1, 0, out, …)` |
| 1 | `(1, 1, max(0, −30001 − P_bms), …)`, nur wenn `0x2000014C` (37001) == 5, sonst `(1, 1, 0, …)` |
| 3 | `(1, 1, min(out, u16@0x200002BE), …)` |
| andere | nichts |

Die Kurzfassung vom 24.09. sprach von „jedem Modus“. Genau genommen senden die Zustände 0 und 3
die PV-Obergrenze, 1 schickt einen anderen Wert, und 2 sendet nichts. Was die Zustände fachlich
bedeuten, ist hier nicht untersucht.

Der Empfänger ist nach bisheriger Lage ein eigener MPPT-Controller ohne verfügbares Image
(MPPT-Version 104 laut Firmware-String `V150.116.104.118`). Im Micro 116 findet sich keine
PV-Abregelung, er setzt nur Wechselrichterleistung und BMS-Strom-/Spannungsgrenzen um.

### 3.3 „Überschuss einspeisen“ — `Surplus_FeedIn_Enable_Check` (0x080052B0)

Bis 2026-09-24 hieß die Funktion `Battery_Forced_Charge_Check`, mit Zwangsladung hat sie nichts
zu tun. Alle 300 Ticks werden geprüft:

- `full_en` (`0x20014D00 + 0x75`, EEPROM `0x375`) gesetzt,
- Netzfreigabe (`0x20014E9C + 0x16`) ≠ 0,
- mindestens ein PV-String ≥ 14,1 V,
- SOC ≥ 95 %.

Fehlt eine Bedingung, werden beide Flags gelöscht. Bei **SOC ≥ 100 %** und gelöschtem
`chrg_flag` (BMS `+0x16` Bit 0) wird `0x20000285` gesetzt und bleibt es, bis der SOC unter 95 %
fällt. `0x20000286` kommt zusätzlich hinzu, sobald die AC-Leistung 44003 − 200 W erreicht.

---

## 4. `charge_to_soc` (42011) und force_mode

### 4.1 Schreibpfad

`Write_Handler` (0x08051D14), Zweig `0xA41B`:

```c
if (13 ≤ wert ≤ 100) {
    *(u8*)0x2000034A = wert;   // 42011, nur RAM
    *(u8*)0x20000349 = 3;      // 42010 force_mode ← Nebenwirkung
}
```

Die Zeile `puVar3[1] = 3` zeigt über `puVar3 = 0x20000348` auf `0x20000349`. Jeder gültige
Schreibzugriff auf 42011 startet also Modus 3, auch wenn 42010 vorher 0 war. Das deckt sich
mit dem Feldbefund in `Modbus_RS485_TCP/VNSD_Register_Persistenz_RAM_vs_EEPROM_v150_de.md`
(„Setzen auf 70 löst sofort eine Entladung aus“).

### 4.2 Modus 3 — `Remote_Power_Command_Execute` (0x0801EBA4)

```c
case 3:
    soc = u16@0x20014F94 / 10;             // 32104, GESAMT-SOC
    if      (charge_to_soc < soc) Setpoint(1, +2500);  // entladen
    else if (charge_to_soc > soc) Setpoint(1, −2500);  // laden
    else    force_mode = 0;                           // Ziel erreicht → aus
```

- Vergleichsbasis ist der Gesamt-SOC, nicht der SOC des aktiven Packs.
- Die Leistung ist fest 2500 W. 42020/42021 wirken hier nicht, begrenzt wird nur durch §2.2.
- Bei Gleichstand setzt der Control force_mode selbst auf 0 zurück.

`charge_cutoff_soc` (EEPROM `0x200`) wirkt in D 150 nicht (aus der Analyse vom 24.09.
übernommen).

---

## 5. Versionsunterschiede (aus der Analyse vom 24.09., nicht erneut geprüft)

PV-Obergrenze bei Einspeisung (`ac ≥ 1`):

| Version | Obergrenze | Bemerkung |
|---|---|---|
| D 147 | `ac + P_bms`, ohne 2500-W-Grenze | zweistufige Behandlung bei voller Batterie |
| D 149.2 | `min(P_bms, 2500)` | |
| D 150 | `min(P_bms, 2500)` | |

Die Abregelung auf das Ladelimit kam also mit 149.2. Unter 147 durfte die PV bei Einspeisung um
den Betrag der Einspeisung über das Ladelimit hinaus.

---

## 6. Ghidra-Umbenennungen (2026-09-24, nur Programm D 150)

| Adresse | vorher | nachher |
|---|---|---|
| `0x080052b0` | `Battery_Forced_Charge_Check` | `Surplus_FeedIn_Enable_Check` |
| `0x08005b54` | `Inverter_RS485_Command_Send` | `Inverter_CAN_PowerCmd_Send` |

In 149.2, 147 und den A/E3-Programmen tragen dieselben Funktionen noch die alten Namen. Ältere
Dokumente (`VNSD_Control_FW_Function_Tracking_new_de.md`, `Modbus_RS485_TCP/VNSD_Command_Register_45000_Block_v150_de.md`,
Gerätevergleich §3e) nennen ebenfalls die alten Namen. Dekompilat-Export
`Dekompilate/VNSD_Control_v150/` (lokal, nicht im Repo) ist neu erzeugt.

---

## 7. Offene Punkte

- Bedeutung der Zustände von `0x200002B4` im Limiter und des Werts `0x200002BE` (Zustand 3).
- Modbus-Adresse des Stromlimits beim E3 (s. Gerätevergleich §3f).
- Feldmessung zu §1.3 fehlt: MPPT-Leistung (30037–30040) gegen AC-Leistung und 32106 oberhalb
  90 % SOC mit PV-Überschuss loggen. Erst dann ist klar, ob und wie viel PV verloren geht.
  „Überschuss einspeisen“ (`full_en`) greift laut Code erst ab 100 % SOC.
