# BMS-FW V115 (Venus A) — Offene Punkte im Detail

> 🇬🇧 English version: [VNSA_BMS_FW_v115_Open_Points.md](VNSA_BMS_FW_v115_Open_Points.md)

**Stand:** 2026-10-11. **Übergeordnetes Dokument:** [VNSA_BMS_FW_v115_Vollanalyse_de.md](VNSA_BMS_FW_v115_Vollanalyse_de.md) (§13).
**Quellen:** Dekompilat und Listing `Dekompilate/VNSA_BMS_115/` (nur lokal), abgeglichen mit den Exporten von A BMS 110 / 110.5,
D BMS 117.7 / 118, E3 BMS 115, A Control 150.9 / 151, D Control 147–151, E3 Control 151, A Micro 1211 und E3 Micro 119.
Nichts in diesem Dokument wurde an einem Gerät gemessen.

**Verwendete Beweisstufen**

| Kennung | Bedeutung |
|---|---|
| **[C]** | Durch den Code von A BMS 115 belegt (Funktion/Adresse angegeben) |
| **[X]** | Durch den Code eines anderen Firmware-Images im Bestand belegt (Image genannt) |
| **[I]** | Schlussfolgerung aus Code-Fakten; plausibel, nicht bewiesen |
| **[U]** | Unbekannt; aus der vorhandenen Firmware nicht entscheidbar |

## 0. Kontext: Dateiname `VEPRO_BMS`

- Dateinamen der Venus-A-BMS-Images: A 110 `…_VABMS_APP_110_…`, A 110.5 `…_VA50A_APP_V1105_…`, A 115
  `…_VEPRO_BMS_V115_APP_20260901_…`. Der OTA-Eintrag ist unter dem Gerätetyp `VNSA-0` abgelegt, die Release-Note lautet „VNSA115
  Official Edition".
- Der Code ist **Venus-A-Code**: maskierte Segmentabdeckung 82,4 % gegen A 110.5 (81,4 % gegen A 110, 65,7 % gegen D 118, 12,6 %
  gegen E3 115); der String-Satz entspricht A 110/110.5 bis auf Build-/Versions-Strings; der String `VNSA` ist wie in 110/110.5
  vorhanden; der String `VEPRO` kommt im Image **nicht** vor **[C]**.
- Das App-Vokabular (App 1.6.76, [../App/Marstek_App_Befehlsvokabular_1.6.76_de.md](../App/Marstek_App_Befehlsvokabular_1.6.76_de.md))
  führt die Venus-Typcodes `VEPRO`, `VEPROCH` und `VEPRO2` neben `VNSA`/`VNSA2` **[X]**.
- **Annahme (nur aus dem Namen, von der Firmware nicht bestätigt):** `VEPRO` steht für eine geplante „Venus E Pro", eventuell in
  zwei Modellen (`VEPRO`/`VEPRO2`), die die BMS-Codebasis der Venus A nutzt. Nichts im BMS-Image verweist auf ein solches Produkt.

## 1. Konfigurations-Prüfsumme umfasst ihr eigenes Wort

### 1.1 Code-Fakten

| Punkt | A 115 |
|---|---|
| Block | RAM `g_cfgMagicByte` `0x20003F60` … `0x20004037`, EEPROM-Adresse 600 (0x258), **0xD8 Bytes** |
| Nutzdaten | Offsets 0x00–0xCF, jedes Byte gehört zu einem benannten Feld (zuletzt `g_cfgTailReg20` 0xCC, `g_cfgTailReg19` 0xCE) **[C]** |
| Prüfsummenwort | `g_cfgChecksum` (int32) an Offset **0xD0** (`0x20004030`) |
| Neues Feld | `g_cfgPackUvShutdownTimeout` (uint32) an Offset 0xD4, Default 120 — nicht von der Prüfsumme erfasst |
| Prüfsummenfunktion | `LIB_Checksum_Ones_Complement` `0x0800BE28`: `-1 - Σ data[i]` (Bytesumme, int32) |
| Speichern | `APP_Config_Checksum_And_Save` `0x0800B7E4`: `movs r1,#0xd4; bl 0x0800BE28; str.w r0,[r1,#0xd0]; movs r2,#0xd8` → Prüfsumme über **0x00–0xD3**, abgelegt an 0xD0, 0xD8 Bytes geschrieben **[C]** |
| Laden | `BMS_Config_Params_Load_Or_Default` `0x0800A840` (einmal aus `System_Init_Main`): 0xD8 Bytes lesen, Prüfsumme über **0x00–0xD3** (`movs r1,#0xd4` bei `0x0800A854`), Vergleich mit dem Wort an 0xD0 (`0x0800A86A`), Vergleich der Kennung mit `'s'` (0x73). Bei Lesefehler, Abweichung oder falscher Kennung: alle Felder = Defaults, danach `APP_Config_Checksum_And_Save` **[C]** |
| Weitere Schreiber | `RS485_Register_Write_Handler` `0x0800B10C` (RS485-Opcode 0x12, Gruppen 0x01/0x02/0x04/0x08/0x10/0x40/0x100/0x200/0x400), `RS485_RegisterWrite_DefaultGroup` `0x0800B212` (+ Thunk `0x0800B20C`): gleiche Inline-Folge (0xD4 / 0xD8); `RS485_Dispatcher`-Opcodes 0x1E (`g_cfgUserFlag1`) und 0x20 (`g_cfgTailReg19`) sowie `BMS_Config_Params_Factory_Reset` (RS485-Opcode 10 über `APP_Factory_Reset_And_Reboot`) rufen `APP_Config_Checksum_And_Save`. Kein CAN-Pfad schreibt den Block **[C]** |

### 1.2 Rechnung

Sei `S` = Bytesumme der Offsets 0x00–0xCF, `c_alt` = das int32 an 0xD0 **im RAM vor dem Speichern**, `b(c)` = Bytesumme der vier
Bytes eines Worts. Beim Speichern wird berechnet

`c_neu = -1 - (S + b(c_alt)) = ~(S + b(c_alt))`

Da `S + b(c) ≤ 0xD4 × 255 = 54060 < 65536`, sind die beiden oberen Bytes jedes berechneten Worts `0xFF 0xFF`, und mit
`T = S + b(c_alt) = 256·hi + lo` gilt: `b(c_neu) = 1020 − lo − hi`.

Beim nächsten Start liest das Laden `c_neu` und `S` zurück und berechnet `~(S + b(c_neu))`. Die Prüfung besteht nur, wenn

`~(S + b(c_alt)) == ~(S + b(c_neu))`  ⇔  **`b(c_neu) == b(c_alt)`**

d. h. **der gespeicherte Block ist nur gültig, wenn die Bytesumme des neuen Prüfsummenworts gleich der Bytesumme des Prüfsummenworts
ist, das vor dem Speichern im RAM stand** **[C]** (reine Rechnung auf dem obigen Code).

Bei fehlgeschlagener Prüfung werden die Defaults in die Nutzdaten geschrieben, `g_cfgChecksum` behält aber den gerade aus dem EEPROM
gelesenen Wert; das Speichern iteriert daher `x_{k+1} = g(x_k) = 1020 − lo(S+x_k) − hi(S+x_k)` mit `x = b(c)`. Innerhalb eines
256er-Blocks ohne Übertrag gilt `g(x) = K − x`, die Folge **springt also zwischen zwei Werten hin und her** und kommt nie zur Ruhe,
außer sie trifft den Fixpunkt.

**Zahlenbeispiele für den Default-Block** (Defaults aus `BMS_Config_Params_Load_Or_Default`, `S_def = 16215 = 0x3F57`) **[C]**:

| Start (Wort im RAM vor dem ersten Speichern) | Gespeicherte Wörter bei aufeinanderfolgenden Starts | Ergebnis |
|---|---|---|
| Gelöschtes EEPROM (`0xFFFFFFFF`, b = 1020) | `0xFFFFBCAC`, `0xFFFFBD42`, `0xFFFFBDAB`, `0xFFFFBD42`, `0xFFFFBDAB`, … | scheitert bei jedem Start |
| Von A 110 mit Defaults geschriebener Block (Annahme: 110-Defaults = 115-Defaults bis auf Kennung `'n'`; Wort `0xFFFFC0AD`, b = 875) | `0xFFFFBD3D`, `0xFFFFBDB0`, `0xFFFFBD3D`, … | scheitert bei jedem Start |
| Lesefehler (RAM-Wort 0 aus `.bss`, b = 0) | nach dem ersten Schritt wie gelöschtes EEPROM | scheitert bei jedem Start |
| Wort mit b = 690 | `0xFFFFBDF6`, danach dauerhaft `0xFFFFBCF7` | besteht ab dem dritten Start |
| Wort mit b = 945 (`0xFFFFBCF7`) | `0xFFFFBCF7` | besteht |

Für `S_def` ist b = 945 (Wort `0xFFFFBCF7`) der einzige Fixpunkt; Start-Bytesummen, die ihn je erreichen, sind 180, 435, 690 und
945. Ein berechnetes Wort hat immer b ≥ 510, aus von der Firmware geschriebenen Blöcken sind also nur 690/945 erreichbar. Über alle
`S` und alle Startwerte 510…1020 bestehen nur ~0,4 % der Kombinationen innerhalb von 40 Starts je die Prüfung (Simulation der
obigen Formel) **[C]**.

Ein geänderter Parameter (RS485-Opcode 0x12/0x1E/0x20) ändert `S`; selbst im stabilen Zustand b = 945 bleibt 945 bei einer
Nutzdatenänderung um +1, +2, +10 oder +100 kein Fixpunkt, der nächste Start scheitert also und lädt die Defaults **[C]**.

### 1.3 Folgen

- **Über RS485 geschriebene Parameter (Schutzschwellen, Zeiten, `g_cfgUserFlag1`, `g_cfgTailReg19`) überleben praktisch nie einen
  Neustart**; beim nächsten Start werden die Defaults geladen und erneut gespeichert **[C]**.
- Packs, die nie umparametriert wurden, merken keinen funktionalen Unterschied: sie liefen ohnehin mit den Defaults. Nebeneffekt: ein
  0xD8-Byte-EEPROM-Schreibvorgang bei fast jedem Start **[C]**; ob das für die EEPROM-Lebensdauer relevant ist, ist **[U]**
  (EEPROM-Typ nicht bestimmt).
- Nach dem Update von 110/110.5 werden die Defaults ohnehin einmal geladen (Kennung `'n'` → `'s'`, Blocklänge 0xD4 → 0xD8, siehe
  Vollanalyse §11) **[C]**.
- Wahrscheinliche Ursache: der Block wuchs um 4 Bytes, und das Längenargument der Prüfsumme wurde zusammen mit der Blocklänge von 0xD0
  auf 0xD4 erhöht, statt beim Prüfsummen-Offset zu bleiben **[I]**.

### 1.4 Andere Images

| Image | Block | Prüfsummenbereich | Wort an | Korrekt? |
|---|---|---|---|---|
| A 110 / A 110.5 | 0xD4, Kennung `'n'` | 0x00–0xCF (`0xd0`) | 0xD0 | ja **[X]** |
| D 117.7 / D 118 | 0xD4, Kennungen `'u'` / `'v'` | 0x00–0xCF | 0xD0 | ja **[X]** |
| E3 115 | 0xA8, Kennung `'s'` | 0x00–0xA3 (`0xa4`) | 0xA4 | ja **[X]** |
| **A 115** | 0xD8, Kennung `'s'` | **0x00–0xD3** | 0xD0 | **nein** **[C]** |

Der Fehler existiert nur in A 115.

### 1.5 Test an einem echten Pack

1. **Passiv, ohne Schreibzugriff des Testers (bevorzugt):** Logikanalysator am per Bit-Banging betriebenen EEPROM-Bus (PB10/PB11,
   Geräteadresse 0xA0, 16-Bit-Wortadresse; `INIT_I2C_EEPROM_GPIO`, `I2C_EEPROM_Read`). Pack mehrmals aus- und einschalten. Erwartung
   laut Code: bei jedem Start ein Lesen von 0xD8 Bytes ab 0x0258, gefolgt von einem **Schreiben** nach 0x0258; die Bytes an den
   Block-Offsets 0xD0–0xD3 wechseln zwischen zwei Wörtern (z. B. `3D BD FF FF` / `B0 BD FF FF` nach einem Update von 110 mit
   Default-Konfiguration). Kein Schreiben nach dem Lesen = Prüfung bestanden (Fixpunkt erreicht).
2. **Offline, nur lesend:** Pack getrennt, MCU im Reset gehalten, EEPROM 0x0258–0x032F mit einem Klemm-Programmer auslesen.
   `S` = Σ Bytes 0x00–0xCF, `x` = Σ Bytes 0xD0–0xD3; der Block besteht beim nächsten Start genau dann, wenn
   `Wort(0xD0) == 0xFFFFFFFF − (S + x)`. Nach einem Start erneut auslesen: ein geändertes Wort ohne Parameteränderung bestätigt den
   Fehler.
3. **Aktiv (schreibend, nur wenn vertretbar):** RS485-Opcode 0x11 eine Parametergruppe lesen, Opcode 0x12 einen harmlosen Wert
   schreiben (z. B. eine Release-Zeit), mit 0x11 zurücklesen, Neustart (Aus/Ein oder Opcode 2), erneut 0x11. Erwartung: der
   Default-Wert ist zurück. Im seltenen Fixpunkt-Fall bleibt der geänderte Wert — dann von Hand zurücksetzen.
   CAN hilft nicht: kein CAN-Frame überträgt diese Parameter oder die Prüfsumme.

### 1.6 Nicht bekannt

- Ob Marsteks Service-Werkzeuge diese Parameter bei A-Packs überhaupt schreiben, d. h. ob der Fehler praktisch wirkt **[U]**.
- Ob Bootloader oder Fertigung den Block anfassen **[U]** (Bootloader nicht vorhanden).
- Ob eine spätere Version ihn behebt **[U]**.

## 2. Unbekannte CAN-IDs

### 2.0 Gemeinsame Fakten

- Alle vier eingehenden IDs werden nur in `CAN_RX_Handler` `0x08008BD4` dekodiert, den `APP_CAN_RX_Task_Process` `0x080035D0`
  **nur auf Slot 1 (Master)** aufruft; andere Slots nutzen `CAN_RX_Command_Dispatcher`/`CAN_RX_System_Command_Handler`, die keine
  dieser ID-Konstanten enthalten. Nutzdatenbyte n = Frame-Offset `0xB + n` **[C]**.
- **ID-Aufbau des internen Busses** (A Control `CAN_Build_Arbitration_ID` `0x08037798` in Control 151, A Micro 1211
  `CAN_TxQueue_PushReply` `0x0801A420` / `CAN_TxQueue_PushMultiFrame` `0x0801A5B0`) **[X]**: Anfrage
  `0x00 | 0x01<<16 | (Klasse<<4 | Knoten)<<8 | Funktion`; der Micro antwortet mit vertauschter Quelle/Ziel und der Frame-Nummer in
  Bit 24–28: Klasse 4 → `idx<<24 | 0x41<<16 | 0x01<<8 | Funktion`, Klasse 2 → `idx<<24 | 0x21<<16 | 0x01<<8 | Funktion`.
- Der A-Micro nimmt BMS-Frames `(id & 0x1FF0FFFF) == 0x1800AA01` an (`CAN_RxDispatch` `0x08019360`), Micro und BMS hängen also am
  selben Bus, und das BMS sieht die Antworten des Micros **[X]**.
- Die A Control 151 fragt Klasse 2 Funktion 0x03 (`Inverter_ClearReg_0x03_Default` `0x0802CFEC`, periodischer Timer ≈ 510 ms) und
  Klasse 4 Funktion 0x10 (`Inverter_ClearReg_0x10_Default` `0x0802D06A`, ≈ 520 ms) ab **[X]**.
- Dokumentation des A-Micros: [../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Vollanalyse_de.md](../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Vollanalyse_de.md) §4.1/§4.2.

### 2.1 Eingehend `0x01210103`

| Punkt | Befund |
|---|---|
| Handler | `CAN_RX_Handler`, Zweig `0x08008D5C` **[C]** |
| Auswertung | `g_can01210103Flag` (`0x20002AA5`) = (data[0] >> 4) != 0; DLC nicht geprüft; das Flag wird nur vom nächsten solchen Frame neu geschrieben (kein Timeout) **[C]** |
| Verwendung | `PackSelect_RelayCommStateMachine` `0x0800BEE8` (gelesen bei `0x0800BF10`): bei `g_emsHeartbeatModeByte != 2` gilt `g_relayCommEscalationInhibit` = Flag ODER \|`g_sysTotalCurrent_x10`\| ≥ 11 (1,1 A). Die Sperre unterdrückt die Eskalation nach 300 Ticks in Fehlermodus 3 und nach 600 Ticks in Fehlermodus 2 sowie die LED-Phasen der Relais-Kommunikation (`LED_StatusPatternUpdate`); die Eskalation nach 1000 Ticks in Modus 4 wird nicht gesperrt **[C]** |
| Sender | **A Micro 1211**: Frame-Nummer 1 der Klasse-2-Antwort auf Funktion 0x03 (`CAN_RxDispatch` Klasse 2, Funktion 3 → `Mppt_BuildStatusReport`, `CAN_TxQueue_PushReply(3,1,8,…)`); data[0] = `g_mpptRptStateByte`, Bit 0–3 MPPT-Ladezustand, **Bit 4–7 = Lauf-Flags der PV-Kanäle 4/3/2/1**; data[2..3] Fehlercode, data[4..5] Chiptemperatur, data[6..7] Warncode **[X]** |
| Bedeutung | Flag = „mindestens ein PV-Kanal der MPPT-Stufe des A-Micros läuft" **[X]**; das BMS nutzt PV-Aktivität, um die Eskalation der Relais-Kommunikations-Zustandsmaschine zu verhindern **[I]** |
| Sicherheit | Hoch für Sender und Feld (Code auf beiden Seiten); nicht am Bus beobachtet |

### 2.2 Eingehend `0x04410110`

| Punkt | Befund |
|---|---|
| Handler | `CAN_RX_Handler`, Zweig `0x08008CFE` **[C]** |
| Auswertung | `g_can04410110Ratio` (`0x20002AA8`, float) = int16(data[4..5]) × 10000,0 / uint16(data[6..7]) (Soft-Float, `0x08008CFE`–`0x08008D56`; keine Prüfung auf Divisor 0) **[C]** |
| Verwendung | **keine**: die einzige Literal-Pool-Referenz ist das Speichern bei `0x08008D54`; kein Basis+Offset-Zugriff gefunden **[C]** |
| Sender | **A Micro 1211**: Frame-Nummer 4 der Klasse-4-Telemetrieantwort auf Funktion 0x10 = Telemetriebytes 24–31: Netzleistung W (0x18), Off-Grid-Leistung W (0x1A), **Batterieleistung W (0x1C, int16)**, **Batteriespannung × 10 (0x1E, uint16)** **[X]** |
| Bedeutung | W × 10000 / (V × 10) = **Batteriestrom in mA** aus Sicht des Micros **[X]** (Rechnung) |
| Gegenprobe | E3 BMS 115 dekodiert denselben Frame mit derselben Formel in `g_vnsCurrentMa` und nutzt ihn in der SOC-Integration; E3 Micro 119 sendet ihn als Frame 4 seiner Telemetrie ([../VNS_Micro_Inverter/VNSE3_Micro_FW_v119_Vollanalyse_de.md](../VNS_Micro_Inverter/VNSE3_Micro_FW_v119_Vollanalyse_de.md) §4.1.5) **[X]**. In A 115 wurde der Empfangscode ohne Verbraucher übernommen **[I]** |
| Sicherheit | Hoch |

### 2.3 Eingehend `0x03410110`, Byte 6

`g_pvRescueRequestFlag` (`0x20002AA4`) = (data[1] > 8) ODER (data[6] == 1) (`0x08008CE2`) **[C]**. Die ID ist Frame-Nummer 3 der
Funktion-0x10-Telemetrie des A-Micros = Bytes 16–23: Netzspannung × 10 (0x10), Netzfrequenz × 10 (0x12), Off-Grid-Spannung × 10
(0x14), `g_gridPermit` (0x16, u16) **[X]**. data[1] > 8 bedeutet also Netzspannung ≥ 230,4 V (wie für 110.5 bereits dokumentiert),
und **data[6] == 1 bedeutet Low-Byte von `g_gridPermit` == 1** **[X]**. Mit einem A-Micro 1211 bedeutet das in diesem Projekt
„PV-Rettung" genannte Flag daher „Netzspannung ≥ 230,4 V oder Netzfreigabe gesetzt" **[I]**; was die BMS-Entwickler beabsichtigten,
ist **[U]**.

### 2.4 Eingehend `0x1801FFAA`, Byte 3 == 2

| Punkt | Befund |
|---|---|
| Handler | `CAN_RX_Handler`, Zweig `id − 0x100501AA == 0x07FCFE00`: zusätzlich zu den bisherigen Feldern `g_emsHeartbeatModeByte` (`0x20002A29`) = data[3] **[C]** |
| Leser | `PackHandoff_Tick` `0x08007150`: Wert 2 **und** `g_pvRescueRequestFlag` → `PackHandoff_Charge_SocSwitch_SM` `0x08006B5C` (Strom des aktiven Packs ≥ 0) bzw. `PackHandoff_Discharge_SocSwitch_SM` `0x08006D0C` statt `PackHandoff_StateMachine` (Schaltreihenfolge siehe Vollanalyse §6); `PackSelect_RelayCommStateMachine`: Wert 2 → Sperre = nur `g_pvRescueRequestFlag` (0x01210103-Flag und Strom werden dann ignoriert) **[C]** |
| Geprüfte Sender | `Grid_BuildStatusFlagsReg` in A Control 150.9 / 151 und D Control 147 / 149 / 149.2 / 150 / 151 schreibt nur Bytes 0–2 (Byte 3 = 0 aus dem genullten Puffer); E3 Control 151 sendet acht Nullbytes. Kein anderes Image im Bestand sendet diese ID oder liest Byte 3 (A 110/110.5, D 118, E3 115 BMS nicht) **[X]** |
| Schluss | Mit jeder vorhandenen Control ist Byte 3 = 0 → die Übergabe-Zustandsmaschinen und die Modus-2-Sperre sind auf heutigen Systemen toter Code **[X]**. Sender des Werts 2: **[U]**; eine spätere Control-Version oder ein anderes Produkt (siehe §0) ist möglich **[I]** |
| Sicherheit | Hoch, dass keine vorhandene Control 2 sendet; Bedeutung von „Modus 2" unbekannt |

### 2.5 Ausgehend `0x1805AA01`

| Punkt | Befund |
|---|---|
| Sender | `APP_IdleKeepaliveBroadcast_Tick` `0x0800F144`, aufgerufen aus dem CAN-RX-Task (10 ms) auf jedem Pack: sobald `g_idleKeepaliveTimerMs` ≥ 50 ms, stellt jeder Aufruf den Frame ein (DLC 1); der Timer wird nur im Master-Pfad zurückgesetzt, Nicht-Master-Packs senden ihn also bei jedem 10-ms-Aufruf **[C]** |
| Nutzdaten | data[0] = `g_mosRecoveryReport` (`0x20002A25`): 0 nach dem Start, 1 Befreiung läuft (Zustand 2/3), 2 beendet; nach einer Befreiung nie wieder auf 0 gesetzt **[C]** |
| Geprüfte Empfänger | A Micro 1211: die ID passiert die Maske `0x1800AA01`, aber Frame-Nummer 5 liegt außerhalb 1..4 und wird verworfen **[X]**. A Control 151: `CAN_FrameDispatcher` → `Protocol_AA_CommandDispatch`, Tabelle `g_canAaDispatchTable` mit 17 Einträgen (initialisiertes `.data`; im komprimierten Ladeabbild lesbare Sub-IDs: 0x1801–0x1804, 0x1820–0x1823, 0x1830–0x1833, 0x1840/0x1841, 0x1007, zwei 0x185x-Einträge) — **kein 0x1805**; `CAN_ParallelInverterDataParser` trifft nicht zu **[X]**. Kein BMS-Image im Bestand empfängt ihn **[X]** |
| Schluss | Kein Empfänger in der vorhandenen Firmware; eine künftige Control ist möglich **[I]** |
| Sicherheit | Hoch für „kein Empfänger im Bestand" (die A-Control-Tabelle wurde aus dem komprimierten Ladeabbild gelesen, nicht aus einer dekomprimierten Kopie) |

## 3. Pins PE7 und PD8

### 3.1 GPIO-Hilfsfunktionen waren vertauscht benannt (Befund, korrigiert)

`GPIO_Init_Pin` `0x08009C3C` ist die STM32F1-artige Init: Deskriptor-Byte 3 = Modus; ist Bit 4 gesetzt, ist es ein Ausgang und die
Geschwindigkeit (Byte 2) wird eingeodert. `GPIO_Init_Output_PP_50MHz` `0x080042F4` übergibt Modus **0x04 = schwebender Eingang**,
`GPIO_Init_Input_Floating` `0x080043B4` übergibt Modus **0x10 + Geschwindigkeit 3 = Push-Pull-Ausgang 50 MHz** **[C]**. Die Namen
waren vertauscht (korrigiert, siehe unten). Das passt zur Verwendung: PB13 und PE9 (gelesen mit `HAL_GPIO_ReadInputBit_Wrapper`) werden mit der
„Output"-Funktion eingerichtet, die Schaltausgänge PC5/PB0/PB2 mit der „Input"-Funktion (Namen vor der Korrektur). Dieselbe Vertauschung in A 110, A 110.5,
D 117.7, D 118; korrekt in E3 115 **[X]**. **Korrigiert:** Die beiden Namen sind in den Ghidra-Programmen von A 110, A 110.5, A 115, D 117.7 und D 118 inzwischen zurückgetauscht (`GPIO_Init_Input_Floating` = Modus 0x04, `GPIO_Init_Output_PP_50MHz` = Modus 0x10 + Geschwindigkeit 3), die Plate-Kommentare der Aufrufer wurden angepasst. Die oben genannten Adressen gehören zu den Namen vor der Korrektur; danach ist `0x080042F4` `GPIO_Init_Input_Floating` und `0x080043B4` `GPIO_Init_Output_PP_50MHz`.

Tatsächliche Einrichtung in `GPIO_Init_All_Pins` `0x08004148` von A 115 **[C]**: Ausgänge PC8, PE6, PE15, PE8, PB5, PB12, PD7, PD15,
PD14, PA4, PB14, PB0, PB2, PC5; Eingänge PC4, PD3, PB13, PE10, PE12, PB1, PE9, PC1, PD13. `GPIO_DeInit_And_CAN_NVIC_Init` (vor STOP)
macht PD7, PD14, PD15, PE8, PB13, PD3, PA4, PB14, PB0, PB2, PC5, PC9, PE15, PA12 zu schwebenden Eingängen.

### 3.2 PE7 — MOS-Befreiung

| Punkt | Befund |
|---|---|
| Treiber | `HAL_GPIO_Control_Switch` `0x08006890` Aktionen 7/8 = PE7 high/low **[C]** |
| Start | `OwnPack_MOSFET_Driver_MasterVariant` `0x080078C8` (Einzel-Master über `BMS_Role_Dispatcher` sowie `PackSelect_Mode3_IdleTick`): in den beiden Zweigen „Laden erlaubt" liest er statt `Control_Switch(1)` (PC5 ein) wie in 110.5 das **Ausgangs-Latch** von PC5; ist es 0 und `g_mosRecoveryState == 0` → Zustand 1. Die Befreiung läuft also bei **jedem Aus→Ein-Übergang der Ladefreigabe** in diesen Rollen, nicht nur nach einem Fehler **[C]**. Slave-Packs (`OwnPack_MOSFET_Driver`) und `Master_MultiPack_Tick` nutzen sie nicht **[C]** |
| Ablauf | `OwnPack_MosRecovery_SM` `0x08005BA4`, alle 100 ms nach der Vorlade-SM: Zustand 1 PE7 ein → Zustand 2 PE7 ein, warten bis `g_pack0Voltage_x10 > 849` → Meldung 1 → Zustand 3: nach 20 Ticks PE7 aus, **PC5 ein**, Meldung 2, Zustand 0. Minimale PE7-Einschaltzeit ≈ 2,1–2,2 s **[C]** |
| 8,49 V | `g_pack0Voltage_x10` ist trotz des Namens in **10 mV** (`PerPack_Struct_Builder`: `g_sh36735PackVoltageMv / 10`; gleiche Skala wie die Schwellen 36,00 V/35,00 V/20,00 V), 849 = **8,49 V**. Quelle: AFE-Register 0x93 (`SH36735_ConvertPackVoltage`, Shell `VBAT2`), nach dem Start 0 (`.bss`) **[C]**. Bei einem 13S-Pack im Normalbereich (≈ 32,5–47,5 V) ist die Bedingung sofort erfüllt, Zustand 2 dauert also einen Tick, außer der Wert ist noch 0 oder veraltet (ein AFE-Lesefehler überspringt die Aktualisierung) **[I]**. Mäße Register 0x93 einen Knoten hinter den MOSFETs, hieße 8,49 V „Ausgangsseite vorgeladen" — **[U]**, der AFE-Knoten ist nicht bestimmbar |
| Kein Timeout | Bleibt der Wert ≤ 8,49 V, hält Zustand 2 PE7 unbegrenzt ein und PC5 bleibt aus **[C]** |
| Zustände 4/5 | Zustand 4 (neuer Versuch nach 1 s, max. 2) und 5 (setzt `g_protState1` Bit 2) existieren, aber **kein Code schreibt 4**: die einzigen Schreiber von `g_mosRecoveryState` sind die SM selbst (0/1/2/3/5, 5 nur aus 4) und der Master-Treiber (0 → 1) **[C]**. In diesem Image unerreichbar |
| Wechselwirkung | Tick-Reihenfolge: Schutz-Tick, dann Rollen-Dispatcher (Treiber → Vorlade-SM → Befreiungs-SM). In Zustand 2 wird ein Schutzzweig, der PE7 ausschaltet, im selben Tick überschrieben. Am Ende von Zustand 3 wird PC5 ohne erneute Schutzprüfung eingeschaltet; ein noch aktiver Schutz schaltet es im nächsten Tick (≤ 100 ms später) wieder aus **[C]**; elektrische Wirkung **[U]** |
| Meldung | Nur über `0x1805AA01` (§2.5); Statusbit 7 von `g_pack0StatusFlags` = PE7-Latch, aber Bit 6/7 gelangen in kein CAN- oder RS485-Statusbyte (`APP_Get_Protection_Status_Byte` packt Bit 0–5, `CAN_TX_PerPack_10Msgs` Bit 4/5) **[C]** |
| Pin-Modus | **PE7 wird nie konfiguriert**: kein Init-Aufruf mit Port E Maske 0x80, kein direkter Zugriff auf `GPIOE_PL_CFG`/`PH_CFG` außerhalb der Init-Hilfsfunktionen **[C]**. Nach dem Reset ist ein STM32F1-kompatibler GPIO-Pin ein schwebender Eingang, Schreiben ins Ausgangs-Latch würde den Pin also nicht treiben **[I]** (Verhalten laut Referenzhandbuch). Ob der Bootloader ihn konfiguriert, ist **[U]** |

### 3.3 PD8 — Ausgang unter 20,00 V

| Punkt | Befund |
|---|---|
| Funktion | `Pack_LowVoltageOutput_PD8_Update` `0x0800A330`, aufgerufen aus `BMS_ProtectionMasterTick` `0x0800C78C` (Schutz-Task, 100 ms, alle Rollen): PD8-Latch = 1, solange `g_pack0Voltage_x10 < 2000` (20,00 V), sonst 0 **[C]** |
| Verwandt | `HW_OvercurrentAndSensorProtection` `0x0800BC70` löscht und überspringt unterhalb derselben 20,00 V die Prüfungen der NTC-Anzahl und der MOS-Temperatur **[C]** |
| Wann wahr | Gesunder 13S-Pack: nie (20 V ≈ 1,54 V/Zelle); beim Start bis zur ersten AFE-Wandlung (Wert 0) und bei einem 0/veralteten Messwert **[I]**. Die 20-V-Schwelle wirkt wie eine Erkennung „Messung ungültig / Platine nicht an den Zellen" **[I]** |
| Pin-Modus | **PD8 wird nie konfiguriert**: `GPIO_Init_All_Pins` und `GPIO_DeInit_And_CAN_NVIC_Init` nutzen Port D Maske 8 (= **PD3**, nicht PD8) sowie 0x80/0x2000/0x4000/0x8000; die KA495xx/SPI-Pins sind PD10–PD12 (0x1C00) **[C]**. Gleiche Folge wie bei PE7 **[I]**; Bootloader **[U]** |
| Platinenfunktion | **[U]** |

### 3.4 Vergleich mit anderen Images

| Pin | A 110 | A 110.5 | D 118 | E3 115 | A 115 |
|---|---|---|---|---|---|
| PE7 | nicht benutzt | nicht benutzt | nicht benutzt | als Push-Pull-Ausgang eingerichtet, beim Init low, vor STOP schwebender Eingang; kein weiterer Zugriff **[X]** | von der MOS-Befreiung geschrieben, nie konfiguriert |
| PD8 | nicht benutzt | nicht benutzt | nicht benutzt | nicht benutzt | alle 100 ms geschrieben, nie konfiguriert |

(Suche nach Port E Maske 0x80 und Port D Maske 0x100 in allen Exporten.) Die E3-Platine hat PE7 als Ausgang, der nur low gehalten
wird; ob es dasselbe Netz wie auf der Venus-A-Platine ist, ist **[U]** (andere Hardware).

### 3.5 Was bewiesen ist und was unbekannt

- Bewiesen: wann die Latches geschrieben werden, die genauen Bedingungen, die unerreichbaren Zustände, die fehlende
  Pin-Modus-Einrichtung in der Anwendung.
- Unbekannt: die Platinennetze von PE7 und PD8, ob der Bootloader ihren Modus setzt, welchen AFE-Knoten Register 0x93 misst, und daher,
  ob die neue Befreiung und das PD8-Signal auf der aktuellen Venus-A-Platine überhaupt elektrisch wirken. Mögliche Lesart: beide
  Ausgänge bereiten Hardware einer anderen Platinenrevision vor (vgl. §0) **[I]**.
