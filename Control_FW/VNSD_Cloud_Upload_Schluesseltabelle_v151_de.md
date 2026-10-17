# Cloud-Upload-Schluesseltabelle (Control v151, Venus D)

> 🇬🇧 English version: [VNSD_Cloud_Upload_Key_Table_v151.md](VNSD_Cloud_Upload_Key_Table_v151.md)

**Stand:** 2026-10-15. Abgeleitet aus Listing und Dekompilat von VNSD Control v151; geprueft an Live-Uploads von drei Venus D (Control 151) und an den Modbus-Werten, die Home Assistant fuer dieselben Geraete gelesen hat. Ueber diesen Vergleich hinaus nicht an Hardware getestet.

Das ergaenzt `VNSD_Cloud_Upload_Payload_v150_de.md`, deren Liste „Noch nicht zugeordnet“ hier bis auf einen Schluessel (`hd`) aufgeloest ist. Der Upload-Body ist `d=<Query-String>`; die Firmware 151 sendet 76 Schluessel: `di` und `sn` (Geraete-ID und Seriennummer, unten nicht aufgefuehrt) und die 74 Schluessel in den Tabellen unten.

Vergleich der Schluesselquellen ueber alle zehn dekompilierten Control-Versionen: [Cloud_Upload_Schluessel_Quellen_Versionsvergleich_de.md](Cloud_Upload_Schluessel_Quellen_Versionsvergleich_de.md)

## 1. Zusammenfassung

- Der Upload ist **ein einziges `sprintf`** in `Cloud_UploadTelemetryData` (0x0801B460). Jedes Argument ist eine Kopie einer benannten Live-Variable oder eines Feldes eines Telemetrie-Datensatzes, die Bedeutung eines Schluessels folgt also aus seiner Quellvariable. Kein Schluessel musste aus seinem Wert geraten werden.
- Zwei Builder fuellen den Datensatz: `MQTT_Telemetry_Struct_Builder` (0x08016FFC) schreibt die Skalarfelder aus den Live-Variablen, `MQTT_Telemetry_String_Formatter` (0x08034CE0) die kommagetrennten Listenfelder. Der Datensatz geht mit `Telemetry_History_Record_Push` (0x08010B7C) in einen Ring aus 144 Slots, und der Upload liest einen Slot (`upload_slot_index`).
- **Hex-Felder:** `eb, e1, ea, et, me, ma` und `sk[4..9]` werden mit `%lx` / `%llx` ausgegeben. Eine `10` bedeutet dort 0x10, nicht zehn.
- **`dn`** ist ein Literal im Aufruf (0x97 = 151), keine Variable: Es zeigt immer die Firmware, die die Nachricht gebaut hat.
- **`sv` ist hier nicht die Version.** In der `getDateInfo`-URL ist `sv` die Control-Version; im Upload ist es `g_workMode`.
- Der Dekompiler zeigt nur die ersten Argumente des Aufrufs. Die uebrigen wurden aus dem Assembler-Listing gelesen (Stack-Slots `sp+0x000 … sp+0x140`, 79 Slots; das 64-Bit-`e1` belegt zwei) und ueber die globalen Adressen den Struct-Feldern zugeordnet. Die Ghidra-Namen mancher Struct-Felder sind falsch (das Feld hinter `t1` heisst „GridExportRaw1“, ist aber die Innentemperatur); verlaesslich ist die Quellvariable im Builder.

Beleg: **Modbus (HA)** = der Wert stimmte mit dem Modbus-Register oder einem Home-Assistant-Sensor desselben Geraets genau (Zaehler, Stufen) oder innerhalb der Abfrageverzoegerung ueberein. **Code** = Bedeutung nur aus der Quellvariable; die Einheit steht dort, wo der Code sie zeigt (etwa ein Suffix `_0p1V` oder ein `/ 10`), sonst gilt der Rohwert.

## 2. Skalarfelder

| Key | Bedeutung | Einheit / Skalierung | Firmware-Quelle | Beleg |
|---|---|---|---|---|
| `to` | Ladeenergie gesamt | ×0.01 kWh | `g_totalChargeEnergyWh` / 10 | Modbus (HA) |
| `td` | Entladeenergie gesamt | ×0.01 kWh | `g_totalDischargeEnergyWh` / 10 | Modbus (HA) |
| `em` | Ladeenergie Monat | ×0.01 kWh | `g_monthChargeEnergyWh` / 10 | Modbus (HA) |
| `gm` | Entladeenergie Monat | ×0.01 kWh | `g_monthDischargeEnergyWh` / 10 | Modbus (HA) |
| `ed` | Ladeenergie heute | ×0.01 kWh | `g_dayChargeEnergyWh` / 10 | Modbus (HA) |
| `gd` | Entladeenergie heute | ×0.01 kWh | `g_dayDischargeEnergyWh` / 10 | Modbus (HA) |
| `bp` | Gespeicherte Energie (SoC × Pack-Energie) | ×0.01 kWh | `g_pgn1801InfoBatSoc * g_pgn1802InfoBatEnergy / 10000` | Modbus (HA) |
| `sc` | Ladezustand | % | `g_pgn1801InfoBatSoc` × 0.1 | Modbus (HA) |
| `pb` | Batterieleistung (negativ = Entladen) | W | `bv × 0.01 × bi × 0.1`, computed | Modbus (HA) |
| `bv` | Batteriespannung | ×0.01 V | `g_pgn1801InfoBatVolt` | Modbus (HA) |
| `bi` | Batteriestrom | ×0.1 A | `g_pgn1801InfoBatCurr` | Modbus (HA) |
| `ci` | BMS-Ladestromlimit | ×0.1 A | `g_pgn1803InfoChrgCurr` | Modbus (HA) |
| `ri` | BMS-Entladestromlimit | ×0.1 A | `g_pgn1803InfoDischrgCurr` | Modbus (HA) |
| `mc` | Max. Ladeleistung | W | `g_maxChargePowerW` | Modbus (HA) |
| `md` | Max. Entladeleistung | W | `g_maxDischargePowerW` | Modbus (HA) |
| `go` | AC-/Netzleistung | W | `g_gridSamplePowerW` | Modbus (HA) |
| `gp` | Off-Grid-Leistung | W | `g_offGridSamplePowerW` | Code |
| `gv` | Netzspannung | ×0.1 V | `g_gridVoltage_0p1V` | Modbus (HA) |
| `gf` | Netzfrequenz | ×0.1 Hz | `g_gridFrequency_0p1Hz` | Modbus (HA) |
| `t1` | Temperatur innen (Umgebung) | ×0.1 °C | `g_envTemp_0p1C` | Modbus (HA) |
| `t2` | Temperatur MOS (Kuehlkoerper) | ×0.1 °C | `g_radiatorTemp_0p1C` | Modbus (HA) |
| `t3` | Reserviert | — | constant 0 | Code |
| `wm` | Arbeitsmodus-Byte (10 = RS485-Steuerung) | raw | `g_userWorkMode` | Modbus (HA) |
| `sv` | Arbeitsmodus | raw | `g_workMode` (not the version: the same name in the `getDateInfo` URL is the Control version) | Code |
| `gt` | Wechselrichter-Status | raw | `g_invState` | Code |
| `bs` | Leistungsfluss-Status (Laden / Entladen / Leerlauf) | raw | `BatteryParams_PowerFlowState_Get()` | Code |
| `bu` | EPS-(Notstrom-)Reserve an | 0/1 | `g_epsReserveEnable` | Code |
| `ct` | Zaehlerdaten gueltig | 0/1 | `g_meterDataValidFlag` | Code |
| `mt` | Zaehlerdaten frisch | 0/1 | `g_meterDataFreshFlag` | Code |
| `ty` | Zaehler-Quelle | raw | `g_meterSourceType` | Code |
| `cy` | Zaehler-Quelle (zweite Kopie) | raw | `g_meterSourceType` | Code |
| `ds` | Entlade-Abschalt-SoC | raw | `g_dischargeCutoffSocRaw` | Code |
| `ph` | Aktiver MPPT-Kanal | raw | `g_mpptActiveChannel` | Code |
| `cw` | MPPT-Betriebsart | raw | `g_mpptOpModeFlag` | Code |
| `bt` | BLE-Verbindungsstatus | raw | `g_bleConnState` | Modbus (HA) |
| `mq` | Cloud-/MQTT-Verbindung aktiv | 0/1 | 1 if the Wi-Fi MQTT is ready or an MQTT session is active, else 0 | Code |
| `ws` | WLAN-RSSI | raw | `g_wifiRssi` | Code |
| `up` | API-Verbindungsstatus | raw | `g_apiConnState` | Code |
| `ap` | Lokale API aktiv | 0/1 | `g_localApiEnable` | Code |
| `nt` | Ethernet-Link aktiv (CH395) | 0/1 | `g_ethLinkActive` | Code |
| `bl` | BLE-SDV freigegeben | 0/1 | `g_bleSdvEnable` | Code |
| `as` | Economy-(TOU-)Task-Status | raw | `g_httpEconomyTaskState` | Code |
| `dn` | Control-Firmware (Konstante in den Format-Argumenten: 0x97 = 151) | — | literal in the call | Modbus (HA) |
| `bm` | BMS-Firmware | — | `g_bmsRefBmsVer` | Modbus (HA) |
| `iv` | Wechselrichter-Firmware | — | `g_canRegCbSoftVer` | Modbus (HA) |
| `mv` | MPPT-Firmware (High:Low-Byte des CAN-Datenworts) | — | `g_canMpptDataWord_verHi/_verLo` | Modbus (HA) |
| `eb` | Pack-1-Fehlerworte 1 und 2 (hex, `%lx`) | hex | `g_bmsPack0Err2:g_bmsPack0Err1` | Code |
| `e1` | Fehlercodes 1 und 2 als 64 Bit (hex, `%llx`) | hex | `g_errorCode1`, `g_errorCode2` (the 64-bit argument takes two stack slots) | Code |
| `ea` | Fehlercode-Bitmaske (hex) | hex | `g_errCodeBitmask` | Code |
| `et` | Warncode-Bitmaske (hex) | hex | `g_warnCodeBitmask` | Code |
| `me` | MPPT-Fehler (hex) | hex | `g_mpptError` | Code |
| `ma` | MPPT-Warnung (hex) | hex | `g_mpptWarning` | Code |
| `pw` | Wechselrichter-Leistungsbefehl | W | `g_canInvPowerCmdW` | Code |
| `soh` | Pack-1-Gesundheit (SOH) | % | `g_bmsPack0Soh` (often 0 in the field) | Code |
| `p_sta` | Leistungsbegrenzung Status | raw | `g_powerClampConfig` (control 151 only) | Code |
| `p_pow` | Leistungsbegrenzung Leistung | W | `g_powerClampConfig_clampPowerW` (control 151 only) | Code |
| `ival` | Upload-Intervall | min | literal 5 in the call | Code |
| `no` | Upload-/Datensatznummer | — | argument of `Cloud_UploadTelemetryData` | Code |
| `dt` | Geraeteuhr | — | record date + `g_rtcCacheHour/Minute/Second` | Modbus (HA) |
| `ip` | Eigene IP-Adresse | — | from the CH395 local address or the network info | Modbus (HA) |
| `gy` | Reserviert | — | constant 0 | Code |

## 3. Listenfelder

| Key | Anzahl | Aufbau | Einheit | Firmware-Quelle | Beleg |
|---|---|---|---|---|---|
| `vc` | 24 | 6 Packs × 4 Werte: Index hoechste Zelle, Index niedrigste Zelle, hoechste Zellspannung, niedrigste Zellspannung | index; mV | `g_bmsCellStat{0..5}Max/MinCellIdx`, `…Max/MinCellVoltage` | Modbus (HA) + Code |
| `tc` | 4 | NTC-Temperaturen 1..4 von Pack 1 | ×0.1 °C | `g_bmsPack0NtcTemp1..4` | Modbus (HA) |
| `cp` | 4 | CT-(Zaehler-)Leistung Phase 1, 2, 3, gesamt | W | `g_ctPowerCh1..3`, `g_ctPowerNow` | Code |
| `ce` | 4 | MPPT-Einkommenswerte A1, A2, B1, B2 | raw | `g_mpptIncomeValueA1/A2/B1/B2` | Code |
| `ms` | 5 | MQTT verbunden, VID-Sitzungsstatus, 0, 0, XID gueltig | raw | `g_mqttConnectedFlag`, `g_mqttVidSessionState`, `g_mqttXidValidFlag` | Code |
| `pv` | 18 | 4 PV-Eingaenge × (Spannung, Strom, Leistung, vorhanden), dann PV-Energie heute und Monat; Leistung wird unter 101 auf 0 gesetzt, vorhanden heisst Spannung > 0x8c | voltage/current raw; power W; energy 10 Wh | `g_mpptPv1..4Vol/Cur/Pow`, `g_mpptPvDayCap10Wh`, `g_mpptPvMonCap10Wh` | Code |
| `sk` | 16 | Arbeits-Pack-Index, Zwangslade-Flag, Pack-Anzahl, Online-Maske, Pack 1..6 Fehlerwort (hex), Pack 1..6 BMS-Version | raw; hex; version | `g_pgn1802InfoWorkBatIdx`, `g_pgn1803InfoForceChrg`, `g_pgn1802InfoBatTotalNb`, `g_pgn1802InfoBatOnlineMask`, `g_bmsPack0..5Err1/2`, `g_bmsPack0..5BmsVer` | Modbus (HA) + Code |
| `fu` | 2 | Ueberschusseinspeisung freigegeben, aktiv | 0/1 | `g_surplusFeedInEnable`, `g_surplusFeedInEnabled` | Code |
| `im` | 11 | MQTT-Sitzung aktiv, CH395-MQTT-Init-Status, lokale UDP-API aktiv, CH395 UDP letzte Empfangslaenge, CH395-Chipversion, BMS-Sperrflag, Wechselrichter-Leistungsbefehl (W), Sendezaehler Sollwert, MPPT-Spannung (×0,1 V), MPPT-Leistung (W), MPPT-Sendezaehler | mixed | `g_mqttSessionActive`, `g_ch395MqttInitStatus`, `g_localApiUdpEnable`, `g_ch395UdpLastRecvLen`, `CH395CMDGetVer()`, `g_pgn1804InfoLockFlag`, `g_canInvPowerCmdW`, `g_invSetpointSendCount`, `g_mpptVoltage_0p1V`, `g_mpptPowerW`, `g_mpptSendCount` | Code |
| `bke` | 3 | Netzeinspeisung (Energiekategorie 4: ans Netz verkauft): heute, Monat, gesamt | Wh | `g_energyCat4DayWh/MonthWh/TotalWh`, fed from `g_gridSellEnergyWh` | Code |
| `bkd` | 3 | Netzbezug (Energiekategorie 3: vom Netz gekauft): heute, Monat, gesamt | Wh | `g_energyCat3DayWh/MonthWh/TotalWh`, fed from `g_gridBuyEnergyWh` | Code |
| `bt_p` | 2 | Pack-1-Ladeleistung, Pack-1-Entladeleistung | W | `g_bmsPack0ChargePower`, `g_bmsPack0DischargePower` | Code |
| `hd` | 3 | **unbekannt** (drei Werte; die Firmware schreibt den Puffer an anderer Stelle, nicht gefunden) | — | — | offen |

In `vc` sind die ersten zwei Werte je Pack die Indizes der Zellen mit der aktuell hoechsten und niedrigsten Spannung. Die Positionen 2 und 3 sind die Spannungen in mV (fuer Pack 1..3 gegen die Modbus-Zellspannungen je Pack bestaetigt). Ein nicht vorhandenes Pack meldet Nullen.

## 4. Unterschiede zwischen Firmware-Versionen

Format-Strings der dekompilierten Control-Firmware, Schluessel fuer Schluessel verglichen:

| Firmware | Schluessel | Unterschied zu VNSD 151 |
|---|---|---|
| VNSD 150, VNSA 150 | 74 | ohne `p_sta`, `p_pow` |
| VNSA 150.9, VNSA 151 | 76 | keiner |
| VNSE3 150 | 68 | ohne `pv, sk, mv, me, ma, fu, p_sta, p_pow` |
| VNSE3 151 | 70 | ohne `pv, sk, mv, me, ma, fu` |
| VNSD 147, 149, 149.2 | — | nicht verglichen (die Funktion liegt in den Exporten nicht unter diesem Namen) |

Die Reihenfolge ist ueberall gleich, wo ein Schluessel vorkommt. Die Bedeutungen oben wurden nur an **VNSD 151** nachverfolgt; VNSA und VNSE3 nutzen dieselbe Funktion mit denselben Namen, ihre einzelnen Quellen wurden aber nicht nachverfolgt.

## 5. Offen

- `hd`: drei Werte (`0,16,0` in einer Feldprobe); der Puffer dahinter folgt auf `g_mqttTeleNetInverterMpptStr`, sein Schreiber wurde nicht gefunden.
- Einheiten der reinen Code-Schluessel ohne Einheiten-Hinweis (PV-Spannung und -Strom, `ce`, `ds`, `ws`, Zaehlerfelder).
- Ob die Formate von VNSD 147/149 abweichen.

## 6. Verwendung

Das Endpoint-Projekt entschluesselt diese Schluessel (`decode.py` und eine Messwerte-Datenbank mit aenderbaren Namen und einer Bedeutung je Firmware) und kennzeichnet jeden Schluessel als gegen Modbus geprueft oder aus dem Code. Firmware-Wechsel werden dort sichtbar, weil `dn`, `bm`, `iv` und `mv` mit Zeitstempel aufgezeichnet werden.
