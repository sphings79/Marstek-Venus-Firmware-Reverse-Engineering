# 44002 / 44003 lesen nach einem Neustart dauerhaft 0

> 🇬🇧 English version: [VNSD_FW_Bug_44002_44003_Readback_After_Restart.md](VNSD_FW_Bug_44002_44003_Readback_After_Restart.md)

**Gerät:** Marstek Venus D (VNSD-0)
**Firmware:** Control/EMS **v150** (`VNSD-0_app_0150_0805_115146.bin`)
**Register:** 44002 `max_charge_power`, 44003 `max_discharge_power`
**Datum:** 20. September 2026
**Status:** Ursache gefunden · am Gerät belegt · **Auswirkung auf den Betrieb ausgeschlossen** (Abschnitt 2b)

**Hinweis zu Adressen:** Alle unten genannten Flash-Adressen wurden für diese Fassung auf die korrekte
Ghidra-Basis 0x08004800 umgerechnet (ursprünglich mit falscher Basis 0x08000000 notiert, Differenz
durchgängig 0x4800). Alte Adresse jeweils in Klammern vermerkt.

---

## 1. Kurzfassung

Nach einem Neustart des Control liefern 44002 und 44003 den Wert **0** und behalten ihn,
bis etwas sie schreibt.

**Ursache:** Es gibt zwei Speicherstellen, und der Lesepfad greift auf die falsche zu.

| | Adresse | wer füllt sie | Rolle |
|---|---|---|---|
| **Config-Block** | `0x2000012B` (+2 charge, +4 discharge) | beim Start aus EEPROM `0x200` geladen und validiert | **das wirksame Limit** |
| **Schattenwert** | `0x20000156` (+0 charge, +2 discharge) | nur bei einem Schreibzugriff | **was 44002/44003 zurückgeben** |

Der Schattenwert wird nirgends initialisiert. Nach einem Neustart steht er auf seinem
BSS-Nullwert, während der Config-Block korrekt geladen ist — deshalb liest man 0, während das
Gerät weiter richtig begrenzt.

**Es ist ein reiner Rücklesefehler, kein Funktionsverlust.** Für einen Client heißt das:
**eine 0 in diesen beiden Registern bedeutet „unbekannt", nicht „null Watt".**

---

## 2. Messung

### 2a. Verlauf über einen Neustart

Direkt am Gerät mitgeschnitten (`192.168.1.50:502`, Proxy gestoppt), Sekundentakt:

| Zeit | 44002 | 44003 | bat_power | SoC | ems_ver |
|---|---|---|---|---|---|
| 11:43:18 | 2500 | 2500 | −12 W | 23 | 150 |
| 11:43:29 | — Gerät nicht erreichbar — | | | | |
| 11:43:59 | **0** | **0** | −12 W | 0 | 150 |
| 11:44:52 | **0** | **0** | −13 W | 24 | 150 |

Der SoC erholt sich von 0 auf 24 und `ems_ver` steht wieder auf 150 — das Gerät ist also
vollständig hochgefahren. Nur die beiden Maximalwerte bleiben auf 0. Auch Minuten später
ändert sich daran nichts; erst ein Schreibzugriff stellt sie wieder her.

### 2b. Lädt das Gerät trotzdem? — A/B-Test

Gleiches Ladekommando, einmal mit `max_charge = 0` und einmal mit 2500. Aufbau:
42000 = `0x55AA` (RS485-Freigabe, nach Neustart nötig), 42020 = 400 W, 42010 = 1 (Laden).
Gemessen wurde Register 30001 über je 30 Sekunden:

| | gemessene Ladeleistung |
|---|---|
| **A: max_charge = 0** | 342–349 W |
| **B: max_charge = 2500** | 347–348 W |

Identisch. Die 0 verhindert das Laden nicht. Vorzeichen zur Einordnung: 30001 steht im
Leerlauf bei −12/−13 W (Eigenverbrauch) und während des Ladens bei +348 W.

---

## 3. Widerlegte Vermutung — bitte nicht wiederholen

Beim ersten Durchgang wurde aus `Inverter_Power_Setpoint_Calc` (0x0800a968, alt 0x08006168) geschlossen,
eine 0 müsse jeden Sollwert klemmen und damit die Batterie lahmlegen. Die dortige Klemmung
liest `DAT_080063d8 + 2` und `+ 4` (die alte, noch nicht vollständig aufgelöste Notation; im aktuellen
Dekompilat sind die beiden Felder bereits direkt als benannte Globals aufgelöst), und `DAT_080063d8` löst
sich zu `0x2000012B` auf — dem Config-Block, aus dem auch 44002/44003 bedient werden — ohne erkennbare
Null-Prüfung:

```c
if (*(ushort *)(DAT_080063d8 + 4) < uVar6)  uVar6 = *(ushort *)(DAT_080063d8 + 4);
if (... uVar6 <= -*(ushort *)(DAT_080063d8 + 2))  uVar6 = -*(ushort *)(DAT_080063d8 + 2);
```

Aktuelles Dekompilat (`Inverter_Power_Setpoint_Calc`, dieselbe Klemmung, vollständig symbolisiert):

```c
if ((int)(uint)g_maxDischargePowerW < (int)power_setpoint) {
    power_setpoint = (uint)g_maxDischargePowerW;
}
if (-power_setpoint != (uint)g_maxChargePowerW &&
    (int)power_setpoint <= (int)-(uint)g_maxChargePowerW) {
    power_setpoint = -(uint)g_maxChargePowerW;
}
```

`g_maxChargePowerW` (0x2000012D) und `g_maxDischargePowerW` (0x2000012F) sind exakt die Config-Block-Felder
aus Abschnitt 4a — die Klemmung ist bestätigt dieselbe, per direktem Zugriff statt über den alten `DAT_`-Zeiger.

**Der A/B-Test in Abschnitt 2b widerlegt diesen Schluss.** Mit `max_charge = 0` lädt das
Gerät unverändert. Entweder greift dieser Zweig im `force_mode`-Pfad nicht, oder `uVar6`
trägt an dieser Stelle nicht den Sollwert. Wo genau die Fehlinterpretation liegt, ist offen —
die Messung ist jedenfalls maßgeblich.

Die Lehre daraus ist allgemeiner Natur: eine Klemmung im Decompilat zu sehen heißt nicht,
dass sie im betrachteten Betriebsfall auch durchlaufen wird.

**Nachtrag — jetzt aufgelöst.** Der Zweig wird sehr wohl durchlaufen, er liest nur nie eine 0:
`DAT_080063d8` ist der Config-Block `0x2000012B`, und der ist nach dem Start korrekt geladen.
Die 0 steckt ausschließlich im Schattenwert `0x20000156`, den die Klemmung gar nicht anfasst
(Abschnitt 4). Der A/B-Test hat also nicht die Klemmung widerlegt, sondern gezeigt, dass
Klemmung und Modbus-Rückgabe aus verschiedenen Quellen stammen — was der eigentliche Befund
dieses Dokuments ist.

---

## 4. Die beiden Speicherstellen

### 4a. Der Config-Block `0x2000012B` — das wirksame Limit

Wird beim Start geladen. `Board_SystemStartup` (0x0801e194, alt `FUN_08019994` @ 0x08019994) ist die oberste
Init-Sequenz — IRQs aus, Init-Kette, IRQs an, CH395-Setup — und ruft darin `Config_LoadOrInit_DeviceSettings`
(0x0802dcbc, alt `FUN_080294bc` @ 0x080294bc) auf, den Config-Loader:

```c
iVar2 = EEPROM_Read(0x200, &g_chargeCutoffSoc, 7);    // g_chargeCutoffSoc @ 0x2000012B
```

(alte Notation: `DAT_0802987c`, noch nicht auf den benannten Global aufgelöst)

Sieben Byte ab EEPROM `0x200`, also genau der Block:

| Offset | SRAM | Feld | EEPROM |
|---|---|---|---|
| +0 | 0x2000012B | charge_cutoff_soc | 0x200 |
| +1 | 0x2000012C | discharge_cutoff_soc | 0x201 |
| +2 | 0x2000012D | **max_charge_power** | **0x202** |
| +4 | 0x2000012F | **max_discharge_power** | **0x204** |
| +6 | 0x20000131 | grid_standard | 0x206 |

Am Ende von `Config_LoadOrInit_DeviceSettings` läuft **unbedingt** der Validator
`Config_ClampDefaults_ParamBlock` (0x0800935c, alt `FUN_08004b5c` @ 0x08004b5c), der
unplausible Werte auf Defaults hebt:

| Feld | bei 0 oder außerhalb des Bereichs |
|---|---|
| charge_cutoff_soc | → 100 |
| discharge_cutoff_soc | → 12 |
| **max_charge_power** | → **2500** |
| **max_discharge_power** | → **800** |

Nach dem Start steht in diesem Block also in jedem Fall ein brauchbarer Wert. Die
Leistungsbegrenzung liest ihn — `Inverter_Power_Setpoint_Calc`, `CT_PowerSetpoint_Compute`,
`Grid_Export_Power_Limiter`, `WorkMode_State_Machine` — und funktioniert entsprechend weiter.

Am Rande: der Default für max_discharge_power ist **800**, nicht 2500. Wer dort einmal 800
vorfindet, hat nicht „800 eingestellt", sondern einen verworfenen EEPROM-Wert.

### 4b. Der Schattenwert `0x20000156` — das, was 44002/44003 zurückgeben

Ein Schreibzugriff auf 44002 (`Write_Handler` @ 0x08056514, alt 0x08051D14) macht **drei** Dinge — aktuelles
Dekompilat, vollständig benannt:

```c
Inverter_Write_Reg_0x04_U32Value(1, value);        // per CAN an den Micro-Inverter
Config_Write_U16_0x202(max_charge_power_w);        // Config-Block 0x2000012D + EEPROM 0x202
_g_maxChargePowerRegCache = max_charge_power_w;    // Schattenwert 0x20000156
```

Der Lesepfad (`is_write == 0`) gibt aber **ausschließlich den Schattenwert** zurück:

```c
if (is_write == 0) {
    local_34 = (uint)_g_maxChargePowerRegCache;    // 0x20000156, nicht 0x2000012D
    Byte_Swap_Copy(rsp, &local_34, 2);
    return 0;
}
```

(alte Notation: `*DAT_080529bc = uVar2` bzw. `DAT_080529bc[1]` — dieselben Zugriffe, noch nicht auf die
benannten Globals `g_maxChargePowerRegCache`/`g_maxDischargePowerRegCache` aufgelöst)

`_g_maxChargePowerRegCache` löst sich zu **0x20000156** auf, nicht zu 0x2000012D; 44003 verwendet analog
`g_maxDischargePowerRegCache` @ **0x20000158**, nicht 0x2000012F. Eine Kreuzreferenz auf 0x20000156 liefert
**genau vier Treffer**, und keiner davon ist ein Laden:

| Von | Art |
|---|---|
| `Write_Handler` 0x0805700c (alt 0x0805280c) | WRITE — Modbus-Schreibzugriff |
| `Write_Handler` 0x0805701a (alt 0x0805281a) | READ — Modbus-Lesepfad |
| `Protocol_AA_SetExtendedParams` 0x0803449c (alt 0x0802fc9c) | WRITE — aus einem CAN-Frame kopiert |

Kein `EEPROM_Read`, keine Init, kein Restore. Nach einem Neustart steht dort der BSS-Nullwert.

### 4c. Warum der Wert manchmal von selbst zurückkommt

`Protocol_AA_SetExtendedParams` (0x08034494, alt 0x0802fc94) kopiert sechs Byte aus einem CAN-Frame in den
Schatten:

```c
*DAT_0802fca8 = *param_2;                        // 4 Byte: 44002 und 44003
*(undefined2 *)(puVar1 + 1) = *(undefined2 *)(param_2 + 1);
```

Schickt ein CAN-Knoten diesen Frame, füllt sich der Schatten ohne Zutun eines Modbus-Clients.
Das erklärt Beobachtungen, bei denen der Rücklesewert nach einer Weile wieder stimmte.

### 4d. Korrektur an der bestehenden Doku

`VNSD_Register_Persistenz_RAM_vs_EEPROM_v150_de.md` und der Funktionskopf von `Write_Handler` führen
für 44002 einen „SRAM-Mirror 0x2000012D" und für 44003 „0x2000012F". Das beschreibt den
**Schreibpfad** richtig, den **Lesepfad** aber falsch — der liest 0x20000156/0x20000158.

---

## 5. Workaround

Ein einmaliger Schreibzugriff nach dem Neustart stellt den Rücklesewert her. Der Wert im
EEPROM wird dabei nur angefasst, wenn er sich ändert (Change-Guard in `Config_Write_U16*`),
ein Zurückschreiben desselben Werts kostet also keinen EEPROM-Zyklus:

```python
client.write_register(44002, 2500)   # eigenen Wert einsetzen
client.write_register(44003, 2500)
```

Für eine Integration bietet sich an, den zuletzt bekannten Wert zu merken und nach einem
erkannten Neustart — etwa an einem SoC-Sprung auf 0 oder am Verbindungsabriss — einmal
zurückzuschreiben. Alternativ die 0 als „unbekannt" darstellen statt als Zahl.

---

## 6. Für einen Herstellerbericht

Belastbar und knapp formulierbar:

1. Die wirksame Konfiguration liegt in `0x2000012B` und wird beim Start korrekt aus EEPROM
   `0x200` geladen und validiert. Daran fehlt nichts.
2. Der Lesepfad von FC03 auf 44002/44003 liest aber nicht diesen Block, sondern einen
   separaten Schattenwert bei `0x20000156`, den nur ein Schreibzugriff oder ein CAN-Frame
   füllt.
3. Folge: FC03 auf 44002/44003 liefert nach jedem Neustart 0, obwohl die Begrenzung aktiv und
   korrekt ist.
4. Kein Sicherheitsproblem — das Gerät begrenzt weiter. Es ist ein Anzeigefehler, der aber
   jeden Client in die Irre führt, der den Wert ausliest oder darauf regelt.
5. Die Korrektur ist klein und es gibt zwei saubere Varianten: entweder den Lesepfad direkt
   aus dem Config-Block bedienen (`0x2000012D` / `0x2000012F` statt `0x20000156`), oder den
   Schatten beim Start einmal aus dem Config-Block initialisieren.

---

## 6b. Venus A — gleicher Defekt, zusätzlich abweichende Grenzen

Statisch aus `150_control_VNSA-0_app_0150_0811_114724.bin`, nicht am Gerät geprüft.

**Der Rücklesefehler ist identisch.** Der Lesepfad des A-`Write_Handler` (@ `0x080516B8`)
bedient 44002 und 44003 aus demselben Schattenwert, und der Zeiger `DAT_08052360` löst sich zu
**`0x20000156`** auf — derselben SRAM-Adresse wie im D. Auch dort wird nichts beim Start
geladen. Ein Venus A liest nach einem Neustart also ebenfalls 0.

**Die Grenzen unterscheiden sich, und zwar asymmetrisch:**

| | Venus D | Venus A |
|---|---|---|
| `Write_Handler`, 44002 (`uVar8 == 900`) | `param_4 < 0x9C5` → ≤2500 | `param_4 < 0x9C5` → **≤2500** |
| `Config_Write_U16_0x202` (speichert 44002) | Clamp `0x9C4` = 2500 | Clamp **`0x5DC` = 1500** |
| `Write_Handler`, 44003 (`uVar8 == 0x385`) | `param_4 < 0x9C5` → ≤2500 | **`param_4 < 0x5DD` → ≤1500** |
| `Config_Write_U16` (speichert 44003) | Clamp 2500 | Clamp **1500** |
| `FUN_08004b5c` Validator-Default | 2500 | **1500** |

Daraus folgt für einen Client am Venus A:

- **44002:** ein Schreibzugriff mit 2000 wird **angenommen** (Rückgabe 0, keine Exception) und
  beim Speichern still auf 1500 geklemmt. Wer den Erfolg am Rückgabewert misst, merkt nichts.
- **44003:** derselbe Schreibzugriff wird **abgelehnt**, `return 3`, also Exception 3
  *illegal data value*.

Zwei Zwillingsregister, zwei verschiedene Prüfungen. Für einen Herstellerbericht ist das ein
eigener, kleiner Punkt: die Eingabevalidierung von 44002 passt auf dem A nicht zur Grenze, die
beim Speichern tatsächlich angewendet wird.

Der Workaround aus Abschnitt 5 gilt unverändert, nur mit 1500 statt 2500 als sinnvollem Wert.

---

## 6c. Status in v151 (Firmware 151, geprüft 2026-10-01)

**Unverändert.** Alle beteiligten Funktionen existieren in v151 mit identischer Größe/Logik, nur an anderen
Adressen (Heap-Vergrößerung 0x11800 → 0x14000 verschiebt nachfolgenden Code):

| Funktion | v150 | v151 |
|---|---|---|
| `Write_Handler` | 0x08056514 | 0x080570e0 |
| `Config_LoadOrInit_DeviceSettings` | 0x0802dcbc | 0x0802e17c |
| `Config_ClampDefaults_ParamBlock` | 0x0800935c | 0x0800935c (unverändert, gleiche Adresse) |
| `Board_SystemStartup` | 0x0801e194 | 0x0801e324 |
| `Protocol_AA_SetExtendedParams` | 0x08034494 | 0x08034aa8 |

Der Lesepfad für 44002 in `Write_Handler` (v151, Register-Offset `uVar3 == 900`) gibt weiterhin nur den
Schattenwert zurück:

```c
if (uVar3 == 900) {
  if (is_write == 0) {
    bcd_reply_value = (uint)_g_maxChargePowerRegCache;   // weiterhin 0x20000156, nicht der Config-Block
    Byte_Swap_Copy(rsp,&bcd_reply_value,2);
    return 0;
  }
  ...
  _g_maxChargePowerRegCache = max_charge_power_w;
  return 0;
}
```

Für 44003 (Registeroffset `uVar6 == 0xabe3`) ist es identisch mit `g_maxDischargePowerRegCache`
(weiterhin 0x20000158). `Config_LoadOrInit_DeviceSettings` initialisiert in v151 zusätzlich den Servertyp
beim Boot (siehe `Control_FW/VNSD_Control_Diff_v150_v151_de.md`), aber **nicht** die beiden Schattenwerte — sie bleiben
beim Start auf ihrem BSS-Nullwert. Der Rücklesefehler nach einem Neustart besteht in v151 unverändert fort.

## 7. Querverweise

- `VNSD_Register_Persistenz_RAM_vs_EEPROM_v150_de.md` — Schreibpfade, EEPROM-Offsets, Change-Guard
- `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md` — der andere offene Firmware-Befund
- `VNSD_FW_Bug_Report_EN_Read_Serializer.md` — Vorzeichenfehler im Read-Serializer (32101)
- Mitschnitt des Neustarts und des A/B-Tests: siehe `Scan_Logs/` (lokal, Rohdaten)
