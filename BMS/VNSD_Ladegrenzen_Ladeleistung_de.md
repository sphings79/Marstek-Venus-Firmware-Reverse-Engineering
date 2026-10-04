# Venus D — Ladegrenzen der Batterie (Ladestrom / Ladeleistung)

> 🇬🇧 English version: [VNSD_Charge_Limits_Charge_Power.md](VNSD_Charge_Limits_Charge_Power.md)

**Stand:** 2026-09-30.
**Firmware-Stände:** BMS v118, Control (EMS) v151, Micro/VNS v116.
**Quellen:** Ghidra-Programme mit korrekter Basisadresse (BMS `0x08002800`, Control `0x08004800`, Micro `0x08004000`)
und die Exporte in `Dekompilate/VNSD_BMS_118/` (lokal, nicht im Repo), `Dekompilate/VNSD_Control_v151/`, `Dekompilate/VNSD_Micro_116/`.
Alle Adressen in diesem Dokument sind echte Flash-Adressen dieser Programme.

## Frage

Der Venus D hat 4 PV-Eingänge à 1000 W und kann per AC mit maximal 2500 W laden. Auf welchen Ladestrom bzw. welche
Ladeleistung ist das Laden der Batterien in der Firmware begrenzt? Kann man die Batterien mit 4000 W laden?

## Kurzantwort

**Nein, 4000 W Ladeleistung sind mit dieser Firmware nicht möglich.** Es gibt zwei unabhängige Grenzen:

1. **BMS: maximal 50,0 A Ladestrom, und es lädt immer nur ein Pack.** Das ergibt höchstens etwa 2,7 kW
   (50 A × ~54 V), rechnerisch maximal 2,88 kW bei der Ladeschlussspannung 57,6 V. Diese Grenze gilt unabhängig
   davon, woher die Leistung kommt (AC oder PV).
2. **Control und Micro/VNS: maximal 2500 W.** Beide Firmwares deckeln die Ladeleistung hart auf 2500 W (`0x9C4`).

Unter 90 % SoC (bei 20–40 °C Zellentemperatur) begrenzen die 2500 W von Control/Micro, oberhalb von 90 % SoC die Stromtabelle des BMS.

## 1. BMS (v118)

### 1.1 Ladestrom pro Pack: Tabelle Temperatur × SoC, Obergrenze 50 A

Funktion `FUN_08012660` (@`0x08012660`, im Ghidra-Programm benannt) berechnet den erlaubten Ladestrom des Packs:

- SoC = 100 % → 0 A.
- Sonst: Temperatur-Stufe aus `g_tempAxisTable` (@`0x0801B97E`, 18 Stützstellen −25 … 60 °C) über
  `BMS_TempZoneIndexLookup` (@`0x08012B90`, mit Hysterese), SoC-Stufe aus der SoC-Achse @`0x0801B9A2`
  (0, 5, 10, 20, 30, 40, 50, 60, 70, 80, 90, 95, 100 %).
- Wert aus `g_chargeCurrentDerateMatrix` (@`0x0801B9BC`, 18 × 12 × uint16) = Prozent von 50 A;
  Ergebnis in 0,1 A: `Prozent × 5` → 100 % = 500 = **50,0 A**. Werte über 100 ergeben ebenfalls 500.

`Master_Build_ChargeCurrentLimit` (@`0x0800BBFC`) übernimmt den Wert des aktiven Packs (während eines Pack-Wechsels
das Minimum der beiden beteiligten Packs, **nie eine Summe**) und begrenzt ihn nochmals auf **500 = 50,0 A**.
Die Ladeschlussspannung setzt dieselbe Funktion fest auf `0x240` = 576 → **57,6 V**. Gesendet wird beides per CAN
(`CAN_TX_PF3_ChargeDischargeLimits` @`0x08007CD8`; die Spannung wird dort zusätzlich auf höchstens 60,9 V begrenzt).

Die Tabelle aus dem Image (Wert = % von 50 A; Zeile = Temperaturbereich von … bis):

| Zellentemperatur | 0–5 % | 5–10 % | 10–20 % | 20–90 % | 90–95 % | 95–100 % |
|---|---|---|---|---|---|---|
| bis −20 °C | 0 | 0 | 0 | 0 | 0 | 0 |
| −20 … −15 °C | 5 | 5 | 5 | 5 | 5 | 5 |
| −15 … −5 °C | 10 | 10 | 10 | 10 | 10 | 10 |
| −5 … 0 °C | 15 | 15 | 15 | 15 | 10 | 10 |
| 0 … 5 °C | 25 | 25 | 30 | 30 | 10 | 10 |
| 5 … 10 °C | 30 | 30 | 40 | 40 | 15 | 15 |
| 10 … 15 °C | 50 | 50 | 50 | 50 | 15 | 15 |
| 15 … 20 °C | 50 | 50 | 80 | 80 | 40 | 15 |
| 20 … 40 °C | 100 | 100 | 100 | 100 | 50 | 20 |
| 40 … 45 °C | 80 | 80 | 80 | 80 | 50 | 20 |
| 45 … 50 °C | 40 | 60 | 60 | 60 | 40 | 20 |
| 50 … 55 °C | 20 | 40 | 40 | 40 | 40 | 20 |
| 55 … 60 °C | 20 | 20 | 20 | 20 | 20 | 20 |
| über 60 °C | 0 | 0 | 0 | 0 | 0 | 0 |

(Die Spalten 20–30 % bis 80–90 % sind in allen Zeilen identisch und hier zusammengefasst. Die Zeilenbeschriftung
folgt der Indexierung in `BMS_TempZoneIndexLookup`: Stufe *i* = Temperatur größer als Stützstelle *i* bis
einschließlich Stützstelle *i+1*.)

Im typischen Bereich 20–40 °C heißt das: **50 A bis 90 % SoC, 25 A von 90 bis 95 %, 10 A von 95 bis 100 %.**

### 1.2 Immer nur ein Pack lädt — im Code erzwungen

- `PackHandoff_StateMachine` (@`0x08006740`) setzt die Lade-MOSFET-Maske `DAT_20002a17` nur auf 0 oder auf genau
  ein Bit (`1 << pack_idx`), nie auf mehrere.
- `OwnPack_MOSFET_Driver` (@`0x08006BC4`) schaltet den Lade-MOSFET eines Packs nur ein, wenn dessen Bit in dieser
  Maske gesetzt ist.

Damit kann physisch immer nur ein Pack seinen Lade-MOSFET eingeschaltet haben. Die Ladegrenze des aktiven Packs ist
gleichzeitig die Ladegrenze des gesamten Systems — egal wie viele Packs verbaut sind.

### 1.3 Pack-Spannung

Ladeschluss 57,6 V (`Master_Build_ChargeCurrentLimit`) und Entladeschluss 48,0 V (`0x1E0`,
`Master_Build_DischargeCurrentLimit` @`0x0800BCC4`) passen zu 16 LFP-Zellen in Reihe (3,60 V bzw. 3,00 V je Zelle,
51,2 V nominal). Eine Zellzahl-Variable gibt es dafür nicht; das ist aus den Konstanten abgeleitet.

## 2. Control / EMS (v151)

- `g_maxChargePowerW` (EEPROM `0x202`): Voreinstellung und zugleich Obergrenze **2500 W**
  (`Config_ClampDefaults_ParamBlock` @`0x0800935C`: Wert 0 oder > `0x9C4` → `0x9C4`).
- App/BLE-Kommando `0x16` akzeptiert nur 300–2500 W (`BLE_App_Command_Dispatcher` @`0x0800C730`).
- Zwangsladen und Fernsteuerung/Modbus-Ladebefehle sind ebenfalls auf ±2500 W begrenzt
  (`Battery_Charge_Power_Limiter` @`0x08008BFC`, `Remote_Power_Command_Execute` @`0x08023538`).
- Zusätzlich gilt ein dynamisches Limit aus Packspannung × Ladestromgrenze des BMS (`g_pgn1801InfoBatVolt` ×
  `g_pgn1803InfoChrgCurr`, `Power_ScaledValue_Calc_Negated` @`0x08017AD8`), benutzt in `Battery_Charge_Power_Limiter`
  und `Inverter_Power_Setpoint_Calc` @`0x0800A968`. Die Werte pro Pack (`g_bmsPack0..7MaxChargeCurrent`) werden nur
  für Telemetrie gelesen, **nie summiert**.
- Ergebnis: Ladesollwert = min(angeforderte Leistung, BMS-Limit, `g_maxChargePowerW` ≤ 2500 W).

**PV/MPPT:** Die 4 PV-Kanäle kommen von einer separaten MPPT-Baugruppe über ein eigenes ASCII-Protokoll per
Ethernet/UDP (`MPPT_ParseATResponse` @`0x08033D28`, `AT_Copy_MPPT_Channel_Data` @`0x080089A0`). Im Control-Code ist nur
dieser Empfangsweg zu finden; einen Befehl zurück an die MPPT-Baugruppe gibt es auf diesem Weg nicht. Die MPPT-Firmware
selbst liegt nicht vor.

## 3. Micro / VNS (v116)

- Lade- und Entladeleistung sind nur in Watt begrenzt: `g_powerLimitChargeW` / `g_powerLimitDischargeW`
  (EEPROM `0x202`/`0x204`, Modbus 44002/44003), CAN-Kommandos `0x03`/`0x04` in `EMS_Inverter_CAN_Dispatcher`
  lehnen Werte ≥ 2501 ab → **harte Obergrenze 2500 W**. Ebenso die Echtzeit-Begrenzung (±2500 W) und die übrigen
  Konfigurations-/Rampenwerte (seit v116 durchgängig 2500 W, in v115 teils 2200 W).
- Den Ladestrom-/Spannungswert des BMS empfängt der Micro (`BMS_CAN_Parser` → `g_bmsChargeVoltLimit`, enthält
  Spannung und Ladestromgrenze).
- Im Micro gibt es **keinen MPPT-/PV-Code**.

## 4. Abgleich mit Messwerten

Messungen ohne Einfluss von Omnibattery (7 Packs, AC-Limit 2500 W): < 90 % SoC ≈ 2310 W; 90 → 95 % ≈ 1260–1290 W;
95 → 100 % ≈ 480–490 W; zwischen den Packs 15–30 s mit 0 W.

- 90–95 %: 25 A × ~53–54 V ≈ 1,3–1,35 kW auf der Batterieseite → passt zu ~1270 W.
- 95–100 %: 10 A × ~54 V ≈ 0,54 kW → passt zu ~485 W.
- < 90 %: 50 A × ~53 V wären ~2,65 kW; es greift vorher die 2500-W-Grenze von Control/Micro.

Die Abstufung kommt also genau aus der BMS-Tabelle (Spalten 90–95 % und 95–100 %, Zeile 20–40 °C).
Die Messwerte liegen etwas unter dem rechnerischen Wert; wie viel davon Wandlerverluste und wie viel die tatsächliche
Packspannung ausmachen, lässt sich aus der Firmware nicht bestimmen.

## 5. Offene Punkte

- Ob PV physisch über den DC/DC-Wandler (LLC) des Micro in die Batterie geht oder direkt an der Batterie hängt, ist
  ohne die MPPT-Firmware nicht sicher zu klären. Für die Antwort spielt es keine Rolle: Die 50-A-Grenze des BMS gilt
  in jedem Fall.
- Nicht geprüft: ob das BMS bei mehr als 50 A aktiv abschaltet (Überstromschwelle im KA495xx-AFE) oder die 50 A nur
  als Grenze an Control und Micro meldet.
- `Grid_Export_Power_Limiter` (@`0x08023B94`) im Control hat einen harten Deckel von **4000 W**. Er betrifft die
  Einspeise-/Exportrichtung und einen CAN-Befehl mit „mppt“-Feldern an Knoten 1, nicht die Ladeleistung. Was dieser
  Befehl genau ansteuert, lässt sich aus dem Control allein nicht bestimmen.
