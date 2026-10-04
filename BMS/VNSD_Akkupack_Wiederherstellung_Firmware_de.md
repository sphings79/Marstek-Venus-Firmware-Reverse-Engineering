# VNSD – Battery Pack Recovery („Akkupack-Wiederherstellung“): Firmware-Seite (Venus D)

> 🇬🇧 English version: [VNSD_Battery_Pack_Recovery_Firmware.md](VNSD_Battery_Pack_Recovery_Firmware.md)

Stand 2026-10-02: **Firmware-Seite UND App-Seite abgeschlossen. Die App-Schaltfläche „Battery Pack Recovery“ sendet den BMS-Kodierungsbefehl (Nummerierung der Pack-Adressen), KEIN Wiederherstellungsladen** (bewiesen durch Dekompilierung der App 1.6.76 / 1.6.72 mit Blutter, siehe §7). Siehe auch: Registeraufschlüsselung
[`../Modbus_RS485_TCP/VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md`](../Modbus_RS485_TCP/VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md),
BLE-Befehlskarte [`../BLE/VNSD_BLE_Command_Map_v150_de.md`](../BLE/VNSD_BLE_Command_Map_v150_de.md).

Umfang: Control 147/149/149.2/150/151 und BMS 117.7/118, nur Textexporte. Marstek-App 1.6.72 (`libapp.so`) und 1.6.76
(`libapp_new.so`): Strings, dazu eine Dart-AOT-Dekompilierung von 1.6.76 mit Blutter (Dart-AOT-Dekompiler, §7). Es wurde kein Live-Gerät verwendet.

## 0. Kurzantwort

* **BEWIESEN (App-Dekompilierung, §7): Die App-Schaltfläche „Battery Pack Recovery“ sendet den BMS-Kodierungsbefehl (Nummerierung der Pack-Adressen), kein Wiederherstellungsladen.**
  Über BLE sendet sie `73 06 23 40 00 16` (cmd 0x40, Nutzlast `00`); ohne BLE sendet sie über MQTT `cd=40,number_as` auf dem Topic
  `hame_energy/<type>/App/<id>/ctrl`. Interne Namen: `handleBmsCoding` / `setBMSCodeFunction`. In der Firmware erreicht dies das BMS als CAN
  `0x100101AA` Byte 3 (schaltet den Pack-Kodierungs-Handshake `g_relayCommandPendingFlag` scharf). Es gibt keine Parameter und keinen Folgebefehl.
  Control 147–151 behandeln ihn bereits, es ist keine neuere Firmware nötig.
* **Die Firmware hat keinen Befehl, der „Pack Recovery“ heißt oder so aufgebaut ist.** Kein BLE-Befehl, MQTT-`cd=`, JSON-RPC-Methode,
  UDP-API-Methode oder Shell-Befehl in Control 147–151 ist eine dedizierte Funktion „Pack wiederherstellen/reaktivieren“.
* **Die Pack-Wiederherstellung existiert, läuft aber eigenständig** im BMS. Das Master-BMS hat eine Wiederherstellungs-Ladezustandsmaschine
  (`PackRecovery_LowCell_Charge_SM`) und einen Niederspannungs-Balancer (`Pack_LowVoltage_Balancer`). Der Balancer speist ein
  Control-seitiges erzwungenes Laden (PGN 0x1803 Byte 7 Bit 0 -> `Battery_Charge_Power_Limiter` + CAN `0x100401AA [00 04]`).
  Die Wiederherstellungs-SM gibt auf und fährt das Pack herunter (Pack-Modus 3), wenn etwa 300 s lang kein Ladestrom von mindestens 1,1 A fließt.
  Das passt zum App-Text „das Gerät muss geladen werden, um es zu aktivieren“. Kein Fernbefehl startet oder
  stoppt diese Logik.
* **Was die App-Schaltfläche bei einem D tut** (Fernpfade, die den BMS-Zustand ändern, siehe §3): von den drei Kandidaten (die BMS-Fehlersperre aufheben,
  `0x100101AA` Byte 8, beim Master startet dies außerdem das BMS neu; das BMS neu starten, Byte 4; die Pack-Adresskodierung
  starten, Byte 3) sendet die App **nur den letzten** (Byte 3). Die Entsperr- und Neustart-Befehle sendet die App nie (§3).
* **Modbus:** Keiner der Entsperr-/Neustart-Befehle hat einen Modbus-Schreibpfad. Die nächstliegenden Modbus-Aktionen sind
  45027 (Master-BMS-Neustart, FUNCTIONAL, gefährlich), 45029 (Pack-Kodierung, CONDITIONAL; das Modbus-seitige Äquivalent der App-Schaltfläche, §4) und 45028 (BMS-Werksreset,
  FUNCTIONAL, gefährlich). 45031 würde Byte 4 (Neustart) senden, kommt beim D aber nie so weit (FUNCTIONLESS). 45030 ist
  FUNCTIONLESS. 32112 zeigt den Sperrzustand (Lesen). 32107 = 0 ist das beobachtbare Zeichen, dass das Wiederherstellungsladen läuft.
  42010 (erzwungenes Laden) kann den Ladestrom liefern, den die Wiederherstellung braucht (nur indirekt).
* **Urteil zu „Battery Pack Recovery“ als Fernfunktion beim D:** Der Befehl ist identifiziert (BMS-Kodierung, cmd 0x40 /
  `cd=40,number_as`) und auf v151 FUNCTIONAL (Handler vorhanden, §1.1 Byte 3), schaltet aber nur den Pack-Nummerierungs-Handshake scharf.
  Das zugrunde liegende BMS-Wiederherstellungsladen bleibt automatisch und wird von der Schaltfläche **nicht** gestartet.
  Die neueren `cd=`-Codes (67/72/90/91 …) haben damit nichts zu tun (kein Handler auf v151, §6).

## 1. Jeder Fernpfad, über den das Control etwas an das BMS sendet / den Pack-Zustand ändert

CAN-Transport: `Register_WriteValue` -> CAN1. Das BMS akzeptiert in Hardware alle IDs und routet sie in Software
(siehe `VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md`): wenn Slot == 1 (Master) -> `CAN_RX_Handler@0x08008240` (feste IDs) + `CAN_CMD_Pack_Activation@0x0800df70`
(Klasse 3); wenn Slot != 1 -> `CAN_RX_System_Command_Handler@0x0800f0bc` + `CAN_RX_Command_Dispatcher@0x0800f128`.

### 1.1 `0x100101AA`-Byte-Befehle (`Config_Apply_SingleReg@0x080317e8`)

| Byte | BMS-Master (`CAN_CMD_01_Handler@0x08008040`) | BMS Slot != 1 (`CAN_RX_System_Command_Handler`) | Control-Sender (v151) |
|---|---|---|---|
| 1 | `g_pendingSystemCmd=1` -> nach 10 Ticks von `BMS_ProtectionAndCommTick@0x080095ac` `g_packFaultModeState=3` (Abschaltung) | `g_pendingSystemCmd=1` nur bei Slot 0xFF, **fällt dann immer in den Werksreset durch** (jedes Pack, das den Frame in diesem Zweig verarbeitet, also jedes Nicht-Master-Pack, wird auf Werkseinstellungen zurückgesetzt) (siehe `VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md`) | Shell `vns_ate_test@0x08057000`; `Idle_Watchdog_AutoReboot@0x08032c80` (×2 vor dem Control-Neustart) |
| 2 | `APP_Factory_Reset_And_Reboot` | gleich | **Modbus 45028**; `FactoryReset_NotifyAndReboot(1)@0x0800c11e` <- `Factory_Reset(1)` <- BLE 0x06 Wert 1, MQTT `cd=5,rs=1`, JSON-RPC/UDP `Reset.Factory {type:1}`, Shell `restore_factory 1` |
| 3 | `g_relayCommandPendingFlag=1` (Kodierungs-Handshake; läuft nach 300 Ticks ab, sofern das PB13-Impulsmuster nicht eintrifft, `PackSelect_RelayCommStateMachine@0x0800b304`) | gleich | `BLE_Apply_ConfigReg_Offset3(0)@0x08032ab8` <- BLE `0x40 [00]`, MQTT `cd=40,number_as` (= App-Schaltfläche „Battery Pack Recovery“); `Board_ProtectionStage_Dispatch@0x08021e78` (+PB10-Impulsfolge) <- Modbus 45029 / Power-Taste |
| 4 | `g_packFaultModeState=4` -> `PackMode_Dispatcher@0x080097e4` Fall 4: CAN-Broadcast (Klasse 3, fn 2, Node 0xF, Nutzlast 1), nach 100 Ticks `g_packRotationCommitPendingFlag=1` + **System-Reset** = BMS-Neustart | – (ignoriert) | `BLE_Apply_ConfigReg_Offset3(1)` <- BLE `0x40 [01]`, MQTT `cd=40,bms_rst`; Entwicklermodus 4: BLE `0x0C 0A 0B 0C 04`, MQTT `cd=7,dm=4`; `Config_Graduated_Sync_Retry@0x0802e680`; Modbus 45031 nur wenn PGN1804 b4-5 == 0xAA (beim D nie) |
| 7 | `g_packHandoffRequestFlag=1` | – | `Grid_Export_Limit_Periodic_StateMachine@0x0800acac` jede 1 s, solange PGN 0x1803 Byte 7 Bit 1 gesetzt ist (automatisch) |
| 8 | falls gesperrt: `APP_SetErrorLockFlag(0)` + `g_packFaultModeState=4` (= Entsperren **und** Neustart) | falls gesperrt: `APP_SetErrorLockFlag(0)` | `Config_Apply_WorkModeReg(0)@0x08022280` <- BLE `0x43 [01 00]`, MQTT `cd=44,lock=0`, Shell `lck_bms 0` |
| 9 | `APP_SetErrorLockFlag(1)` (EEPROM 0xFC00 = 0x86, MOSFETs gesperrt) | – | `Config_Apply_WorkModeReg(1)` <- BLE `0x43 [01 01]`, MQTT `cd=44,lock=1`, Shell `lck_bms 1` |
| 0x0A | `Config_Graduated_Sync_Retry` (automatisch) | – | – |
| 0x0B/0x0C/0xA0 | kein Fall | kein Fall | Modbus 45030 / 45031 (FUNCTIONLESS) |

### 1.2 Weitere an das BMS gerichtete Frames

| Frame | BMS-Wirkung | Control-Sender / Auslöser |
|---|---|---|
| Klasse 3 fn 0x0A Node 1 (`0x0001310A`) | Master: `CAN_Set_Notify_Flag_And_Send` -> (EEPROM 0x202=0xFF, falls Laufzeit-Master) + sofortiger System-Reset | nur **Modbus 45027** (`CAN_SendTriggerCategory0x0A(1)@0x08032934`) |
| `0x100401AA [00 04]` | `CAN_CMD_04_Handler@0x08007ea4` Fall 4: wenn Flags Bit 0 und kein Pack-Fehler -> SOC-Überschreibung, aktives Pack = Low-Pack des Balancers | `Inverter_PVString_ChannelValue_PeriodicCheck_Save@0x080173b4`, alle 500 ms, solange PGN 0x1803 b7 Bit 0 und (beliebige PV > 14,0 V oder Netzfreigabe), automatisch |
| `0x100201AA [soc]` | `CAN_CMD_02_Handler` -> DoD (0x200028E0) | nur Shell `change_soc` (`Config_Write_TargetSOC_Percent@0x0803180c`) |
| `0x1801FFAA` | Byte 2 = Entlade-Abschaltschwelle/DoD, Byte 11 = Freigabe (Master-Start) | `Grid_BuildStatusFlagsReg@0x08031888`, periodisch |
| Klasse 3 0xC4 Node 0 | `g_pendingActivationFlag` (= CLI `g_u8ProhibitCommFlag`, schaltet CAN während OTA stumm). Keine Aktivierung | `OTA_Upgrade_Timeout_Watchdog`, `System_ResetToDefaults` |
| Klasse 3 0xCE | BMS-OTA-Flash-Schreiber | OTA (`cd=53/54`, `Ota.Set`) |
| Klasse 3 0xC1/0xC2/0xCF, `0x100601AA`, `0x180701AA` | kein BMS-Handler | 45001, 45011, Shell `debug_mode`, EPS-Hysterese |
| CAN-ID-Feld `0x210100` | BMS-ISR: RAM-Fehlersperre + Ausgänge aus | **kein Sender gefunden** in Control 151 / Micro 116 |

JSON-RPC (`MQTT_JSON_RPC_Dispatcher@0x0801f4f8`, über `Cloud_PassiveMode_UDP_RecvAndDispatch@0x08015888` auch von der lokalen UDP-API genutzt):
Methoden `Marstek.GetDevice/GetData, Wifi/BLE/Bat/PV/ES/EM.GetStatus,
ES.SetMode/GetMode, DOD.SET, Ble.Adv, Led.Ctrl, Ota.Set, Set.Ver, Reset.Factory`. Nur `Reset.Factory {type:1}` erreicht das
BMS (Byte 2). `DOD.SET` ist die Control-seitige Abschaltschwelle 30–88 % (+`Config_Param0A_Relay`).

Alles in §1 ist in Control 147, 149, 149.2, 150 und 151 identisch: gleiche `Config_Apply_WorkModeReg` 8/9, `BLE_Apply_ConfigReg_Offset3`
3/4, MQTT-Fälle 0x28/0x2C, BLE '@'/'C'/0x0C-Fall-4, `FactoryReset_NotifyAndReboot` Byte 2, Grid_Export Byte 7. BMS 117.7
`CAN_CMD_01_Handler` und `PackMode_Dispatcher` sind textlich identisch mit 118.

## 2. Firmware-Logik, die tatsächlich „Pack-Wiederherstellung / Reaktivierung“ ist (alles automatisch)

1. **Wiederherstellungsladen, Master-BMS** — `Master_RunState_Dispatcher@0x080066e0`: beim Start (`Master_RunState_InitSelect@0x08006b94`)
   oder im Run-State 2 (`PackRecovery_Request_Check@0x080075c4`) setzt ein Pack mit Protect1 Bit 0 (Zellunterspannung, min. Zelle
   < 2500 mV / 2300 mV kalt) `g_packAnyFaultFlag`. `PackRecovery_LowCell_Charge_SM@0x0801070c` dann: alle Dsg-MOS aus ->
   das Pack mit der niedrigsten Zelle lädt allein -> nach 30 Ticks bei ≥ 1,1 A auch Dsg-MOS ein -> fertig bei min. Zelle > 3100 mV und SOC > 13–14 %.
   **Bei < 1,1 A für 3000 Ticks (~300 s) Zustand 7 -> Pack-Modus 3 (Abschaltung)**. Solange sie läuft, erzwingt PF3 ein Ladelimit von 25,0 A
   und ein Entladelimit von 0 (`CAN_TX_PF3_ChargeDischargeLimits@0x08007cd8` über `APP_Get_PackFaultStatus`).
   Das setzt `g_highestPackIndex != 0` voraus (`Pack_Get_Protect1Bit0_Mask@0x08009534`), also einen Multi-Pack-Stapel. Das Verhalten mit nur einem Pack ist nicht belegt.
2. **Niederspannungs-Balancer + erzwungenes Laden durch das Control** — `Pack_LowVoltage_Balancer@0x08007f24` markiert Packs mit SOC < 8 % oder
   Protect1 Bit 0 und hält sie markiert, bis > 12 % und UV aufgehoben. PF3 Byte 7 Bit 0 -> Control `g_pgn1803InfoForceChrg` Bit 0 ->
   `Battery_Charge_Power_Limiter@0x08008bfc` zwingt den Sollwert zum Laden mit −(V × BMS-Ladestromlimit)
   (später durch max. Ladeleistung usw. in `Inverter_Power_Setpoint_Calc@0x0800a968` begrenzt), und
   `Inverter_PVString_ChannelValue_PeriodicCheck_Save` sendet `0x100401AA [00 04]` (Umschalten auf das Low-Pack).
3. **Übergabe-Gate (Handoff)** — PF3 Byte 7 Bit 1 (Master-Run-State 2 und Aggregation nicht abgeschlossen) -> `Grid_Export_Limit_Periodic_StateMachine`
   (Ereignis 600, Byte 7 jede 1 s, Ladesollwert 0).
4. **Fehlersperre aufheben** (nur per Fernzugriff, §1.1 Byte 8): der einzige Fernbefehl, der ein Pack, das weder laden noch
   entladen kann, wieder einschaltet (die Sperre blockiert `OwnPack_MOSFET_Driver*`, `PackSelect_Tick`, `Master_Startup_WaitForEnable`).
5. **Neustart** (Byte 4 / Klasse-3 0x0A / BMS-Taste PB13 langer Druck 10 s -> Modus 4).

„Das Gerät muss geladen werden, um es zu aktivieren“ passt zu den Mechanismen 1 und 2. Das BMS wacht nur auf und erholt sich, wenn Ladestrom
fließt. Das ist auch die klassische Aufwachbedingung aus Ship-Mode/Tiefschlaf (`System_Reset_And_Halt_WithCurrentCheck`
-> STOP-Modus, EXTI-Wake). Kein Fernbefehl steuert dies.

## 3. Kandidaten dafür, was die App-Schaltfläche sendet (aufgelöst)

| # | Befehl | BMS-Wirkung beim D | Belege dafür | Belege dagegen | Ergebnis |
|---|---|---|---|---|---|
| 1 | BLE `0x43 [01 00]` / MQTT `cd=44,lock=0` (Byte 8) | Entsperren + Master-Neustart | einzige Fernaktion „wieder zum Laufen bringen“; der Neustart passt zu „reaktivieren“; App 1.6.76 enthält neu einen nackten `cd=44`-String | keiner der beiden App-Builds hat ein `lock=`-Literal; das Entsperren bewirkt bei einem unterspannten Pack nichts | **widerlegt** (die App sendet ihn nie; kein `lock=` in der App) |
| 2 | BLE `0x40 [01]` / MQTT `cd=40,bms_rst` (Byte 4) | Master-BMS-Neustart (Slaves ignorieren Byte 4) | „reaktivieren“; beide App-Builds enthalten `number_as`, den Geschwisterparameter desselben Handlers | kein `bms_rst`-String in einem der beiden App-Builds | **widerlegt** (die App sendet ihn nie; kein `bms_rst` in der App) |
| 3 | BLE `0x40 [00]` / MQTT `cd=40,number_as` (Byte 3) | schaltet nur den Kodierungs-Handshake scharf; ohne die PB10-Impulsfolge wahrscheinlich NO_OP | `number_as`-String in beiden App-Builds vorhanden | ist Pack-Adressierung, keine Wiederherstellung | **BEWIESEN: Das sendet die App** (BLE `73 06 23 40 00 16`, MQTT `cd=40,number_as`, §7) |
| 4 | BLE `0x0C 0A 0B 0C 04` / `cd=7,dm=4` (Byte 4) | wie #2 | – | Entwicklermodus-Befehl, hinter einem Benutzermenüpunkt unwahrscheinlich | **widerlegt** (die App sendet ihn nie) |
| 5 | keiner / ein neuerer `cd=`-Code | nichts auf v151 | das D-Handbuch hat keinen Eintrag „Battery Pack Recovery“; neue App-Codes 67/72/90/91 sind unbehandelt | – | **widerlegt** (die Schaltfläche nutzt den vorhandenen Code 0x40 / `cd=40`; die neuen Codes gehören zu anderen Funktionen, §6) |

BEWIESEN: alles in §1/§2 (Code) und die App-Seite in §7 (Disassemblierung). Die App-Formulierung „wiederherstellen/reaktivieren“ ist Marketing für die BMS-Adress-
kodierung (interne Namen `handleBmsCoding`, `setBMSCodeFunction`, `icon_bms_coding`/`btn_bms_coding`, `check_get_bms_coding`, Nutzlast `number_as`).
Die frühere Anmerkung zur Reihenfolge der App-Strings bleibt gültig: Die Reihenfolge der Strings ist **kein** Beleg. In
beiden Builds stehen `RepairSignPage` / `recovery current type` / der deutsche Text „aufgeladen“ / `cd33=` neben denselben
nicht zusammenhängenden Strings (rumänische i18n, `ECDHKeyDerivator`). Das ist die Reihenfolge des globalen String-Pools, keine Code-Nähe. Und
`cd=33` (0x21) hat in keiner Control-Version einen **Fall** (0x20–0x27 fehlen), also gehört `cd33=` zu einem anderen Produkt.
`RepairSignPage` stammt aus `pages/Common/repair/` (Garantiereparatur), nicht aus der Akku-Wiederherstellung.

## 4. Modbus-Zuordnung

| Register | Bezug | Status beim D |
|---|---|---|
| 45027 | Klasse-3 0x0A -> Master-BMS-Neustart (gleiche Wirkungsklasse wie Kandidat #2) | FUNCTIONAL, Gefahr hoch, nicht freigeben |
| 45028 | Byte 2 -> Werksreset aller Packs + Neustart | FUNCTIONAL, Gefahr hoch, niemals freigeben |
| 45029 | Byte 3 + PB10-Impulse -> Kodierung / Master-Übernahme; **Modbus-seitiges Äquivalent der App-Schaltfläche** | CONDITIONAL |
| 45031 | würde Byte 4 (Neustart) erst nach einer 0xAA-Quittung senden, die BMS 117.7/118 nie senden; sendet 0xA0 (unbehandelt), ~5 s Stillstand, Exc 3 | FUNCTIONLESS |
| 45030 | Bytes 0x0B/0x0C, kein Handler | FUNCTIONLESS |
| 32112 | Lesen: BMS-Fehlersperre (PF4 b7) — der Zustand, den Kandidat #1 aufhebt | FUNCTIONAL (Lesen) |
| 32106/32107 (35111/35112) | Lesen: BMS-Lade-/Entladestromlimit; 250 / 0 = Wiederherstellungs-SM aktiv | FUNCTIONAL (Lesen) |
| 42010 (+42000/42020) | RS485-erzwungenes Laden: kann die ≥ 1,1 A liefern, die die Wiederherstellungs-SM braucht | FUNCTIONAL, indirekt |
| 41200, 32112 schreiben | kein Pfad zu den Bytes 8/9/4 | – |

**45029 ist das Modbus-seitige Äquivalent der App-Schaltfläche.** Wie die Schaltfläche sendet es Byte 3, schickt aber zusätzlich die PB10-Impulsfolge,
die der Handshake braucht. Der BLE/MQTT-Pfad der App sendet keine Impulsfolge, daher läuft er für sich allein wahrscheinlich in ein Timeout (INFERRED, §7.3).
Der Control-interne Test meldet nur dann Erfolg, wenn genau zwei Packs online sind (Status in 32114, sonst 3).

**Kein Modbus-Register sendet beim D Byte 8 (Entsperren), Byte 9 (Sperren) oder Byte 4 (Neustart).** Write_Handler ruft
`Config_Apply_SingleReg` nur mit 2, 4/0xA0, 0x0B, 0x0C und `CAN_SendTriggerCategory0x0A(1)` auf. PF3 Byte 7 (Erzwingungs-Flags)
ist auf kein Modbus-Register abgebildet (nur MQTT `force=`, Shell `get_main_bms_data`).

Suite: 32112 als diagnostischen binary_sensor „BMS error lock“ und 32107 als Diagnose ergänzen. **Keine** Schaltfläche „Pack Recovery“
hinzufügen: Es gibt kein sicheres Modbus-Äquivalent, und 45027/45028 sind destruktiv.

## 5. Optionale Bestätigung

Ein BLE-Mitschnitt ist zur Identifikation des Befehls nicht mehr nötig (§7). Als Gegenprobe: die GATT-Schreibzugriffe protokollieren, während
„Battery Pack Recovery“ gedrückt wird. Der Mitschnitt zeigt den Frame `73 06 23 40 00 16` und die Antwort `73 .. 23 40 ..` (HM-Frame `73 <len> 23 <cmd> <payload…> <xor>`).
Über WLAN/Cloud zeigt derselbe Druck `cd=40,number_as` und die Antwort `cd=40,ret=1`.

## 6. App-1.6.76-`cd=`-Codes vs. der v151-Dispatcher

Der v151-`MQTT_CtrlCommand_Dispatcher` behandelt dezimal 0–31, 40–45, 50–59, 61, 63–66. 147 hat 61/63–66 nicht; 149/149.2 haben 61/62
(0x3d/0x3e). 150 hat 61, 63–65.

| Neuer App-String | v151-Handler | Bedeutung in App 1.6.76 (durch Dekompilierung bewiesen) |
|---|---|---|
| `cd=12,op=0/1` (`http=<url>`) | 0x0C = Cloud-Servertyp, parst nur `ht=0..4`; `op=` ignoriert, antwortet mit Ergebnis 0 -> **kein Handler für op** | A3600CE-Wallbox: Server-URL lesen/setzen (`ev_charger_connector.dart`, eigenes Topic `marstek_energy/A3600CE-A0/App/<encMac>/ctrl`, nicht der Venus) |
| `cd=16` | 0x10 max. Ladeleistung (`cp`); behandelt (mit `cp=`) | `cd=16,cp=` = `setAcCouplerMaxChargePower` (Venus); nacktes `cd=16` auch CT-Bind-Abfrage und HMH `cd=16,mod=` |
| `cd=18,ct_t=` / `meter=` | 0x12 -> `Cloud_HTTP_Response_Parser@0x08029700` kennt `meter=`, `ct_dev=`, `dchrg=`, `mac=`, `shelly_port=`, `ctrl_rat*`, `rechrg_seq`; `meter=` ok, **`ct_t=` unbehandelt** | in der App-Dekompilierung nicht untersucht |
| `cd=20,md=3,no=` | 0x14 = FC41D-URL-OTA (`len`,`url=`); **kein passender Handler** | in der App-Dekompilierung nicht untersucht |
| `cd=30,data=` | 0x1E = lokale API (`api`,`port`); **`data=` unbehandelt** | in der App-Dekompilierung nicht untersucht |
| `cd=44` / `cd=45` | 0x2C lock bms (`lock=`), nur mit `lock=` behandelt; 0x2D get mppt data, behandelt | V6000-Powerstation, FC41D-Modul (`cd=44` Versionsabfrage, `cd=45,fc41d_url=` Update); `cd=44,do=` / `cd=45,cv=` DoD / Ladespannung anderer Produkte (BigPower, Neumodul-Geräte `CMD_SET_DOD`); HMH `cd=45` = Provider-Daten. Nicht der Venus-Befehl `lock=` |
| `cd=46` / `cd=47,<k>=<v>` | keiner (46–49 in allen Versionen nicht vorhanden) -> **kein Handler** | A3600CE-Wallbox: Statusabfrage / Ladesteuerung (eigenes Topic, nicht der Venus) |
| `cd=51,url=` | 0x33 = get OTA state (ignoriert Parameter); **`url=` unbehandelt** | A3600CE-Wallbox: OTA-Update auslösen (eigenes Topic, nicht der Venus) |
| `cd=63,ct_chg_type=`, `cd=64,ups_p=`, `cd=65,lapi_en=,self_port=,remote_port=` | 0x40 = publish device IDs und 0x41 = get INV data ignorieren Parameter -> **kein Setter**; v151 hat für 63–65 andere Bedeutungen | Neumodul-Venus-Produkte (Venus X / E-mini): CT-Ladetyp, UPS-Ladeleistung, Local-API-Einstellungen |
| `cd=67` | keiner (max. Fall 0x42 = 66) -> **kein Handler in 147–151** | `CMD_GET_SCAN_WIFI_LIST`: WLAN-Scanliste neuerer Venus-Modelle |
| `cd=72,type?` / `cd=72,type=1,len=<n>,ip=<ip>,mac=<mac>` | keiner -> **kein Handler in 147–151** (BLE-Rückfall cmd 0x61, ebenfalls unbehandelt, v151-BLE max. 0x5B) | bindet eine Wallbox an den Venus (Wallbox-IP/MAC; `queryMacAndIp` / `setMacAndIp`, Venus-Topic). Braucht neuere Firmware, ohne Bezug zum Pack |
| `cd=90`, `cd=91,type=<t>,…`, `cd=68` | keiner -> **kein Handler in 147–151** | HMH-Familie (anderes Produkt): Shelly-Scan / Verbinden, Systemcheck |

BLE-Seite: Der v151-BLE-Dispatcher behandelt 0x02–0x06, 0x08–0x29, 0x40–0x43, 0x50–0x55, 0x59–0x5B. Es gibt keine zusätzliche
Recovery-ID. JSON-RPC hat die 16 in §1.2 aufgeführten Methoden (keine Recovery-Methode). In diesem Baum liegen keine A/E3-Control-Images zur Prüfung.

## 7. App-Seite (durch Dekompilierung bewiesen)

Quelle: App `com.hamedata.marstek` 1.6.76 (Dart-Paket `cross_power_x`, `libapp.so` arm64), dekompiliert mit Blutter (Dart-AOT-Dekompiler).
Nur statische Analyse: kein Gerät, keine Cloud, kein Login, nichts wurde ausgelöst. Die Funktion existiert bereits in 1.6.72 (gleiche Strings).

### 7.1 Aufrufkette (BEWIESEN aus der Disassemblierung)

1. **Einstellungszeile** (`AccouplerSettingState::createContent`): wird nur gebaut, wenn `isStack()`. Das gilt für die Venus-Typen VNSD, VNSA, VNSACH, VEPRO,
   VEPROCH, VNSDCH, VNSD2, VEPRO2, VNSA2 und für VEPAC. Für Venus E/C, E3, E4 und EMAX ist die Zeile ausgeblendet. Eine Prüfung der Firmware-Version wurde nicht gefunden.
   Titel „Battery Pack Recovery“ (`tv_battery_recovery`), Icon `icon_bms_coding`, Button-ID `btn_bms_coding`.
2. **Antippen:** setzt Geräte-Eigentümer/Admin (`hasAdminOfDevice`) und ein verbundenes Gerät (`isDeviceConnected`) voraus, danach ein Bestätigungsdialog
   („Preparing to recover the battery pack. Do you want to continue?“).
3. **Bestätigen:** `VenusHelper::handleBmsCoding` -> `CommonCommand::setBMSCodeFunction`, eingepackt in `retryOnTimeoutWithNoTag`: bis zu 3 Sendungen mit je 5 s Timeout.
4. **Transport:** ist der aktuelle Typ leer, wird nichts gesendet. Mit BLE-Verbindung: Frame `73 06 23 40 00 16` (Venus; bei Jupiter HMM/HMN/JPLS lautet der Frame
   `73 06 23 23 00 75` / `cd=20,number_as`). Sonst, wenn die Cloud online ist: MQTT-Topic `hame_energy/<type>/App/<id>/ctrl`, Nutzlast `cd=40,number_as`.
   Keine Parameter (kein Stromtyp, keine Temperatur, keine Version, keine MAC, keine Optionen).
5. **Antwort:** BLE: jeder Antwort-Frame mit cmd 0x40 und Länge >= 5 zählt als Erfolg (das zurückgespiegelte Byte wird nicht geprüft). MQTT: `cd=40,ret=1` = Erfolg.
   Die UI zeigt dann „Recovery successful“ oder „Recovery failed“. Keine Antwort nach 3 x 5 s = „Recovery failed“.
6. **Kein Folgebefehl:** keine Statusabfrage, kein Ladebefehl, kein zweiter Schritt.
7. **Firmware:** `MQTT_CtrlCommand_Dispatcher` Fall 0x28 (`number_as`) oder `BLE_App_Command_Dispatcher` Fall 0x40 -> `BLE_Apply_ConfigReg_Offset3(0)` ->
   `Config_Apply_SingleReg(3)` -> CAN `0x100101AA` Byte 3 (§1.1). Der Handler ist in Control 147/149/149.2/150/151 identisch; es ist keine neuere Firmware nötig.
   Eine `bms_rst`-Nutzlast (Byte 4, Neustart) gäbe es im Handler, die App sendet sie aber nie.

### 7.2 Strings, die sich als unbeteiligt herausstellten (BEWIESEN durch die Fundstelle)

| String | Tatsächliche Verwendung |
|---|---|
| `recovery current type/mac/version/temp` | Debug-Logs der CT-Kopplungsseite (`SelectCtPageState::getSelectCtList`); sie protokollieren und stellen den Geräte-Kontext nach dem Verbinden mit einem CT wieder her. Nichts mit dem Akku zu tun |
| `recovery_server` / `recoveryMqttSSL` | „Reset Server Connection“ (Erweiterte Funktionen, `FunctionCode.recoveryMqttSSL`) |
| „The device must be charged before activation.“ | B2500-OTA-Hinweis (`OTA_CheckB2500SuccessfulTip`) |
| `/ems/api/v1/setApplyVenusActivate` | `activateUpgradePower(devid, code)`: Aktivierungscode für die Leistungsklasse, nicht der Akku |
| „Upgrade … recovery“-Texte | Hinweise bei OTA-Fehlern |

### 7.3 Risikobewertung

* BEWIESEN: Die Schaltfläche sendet nur cmd 0x40 `[00]` / `cd=40,number_as`. Es gibt keine Parameter, `bms_rst` und `lock=` kommen in der App nicht vor, und es existiert kein Folgebefehl.
* BEWIESEN: In der Firmware ist das nur Byte 3 von `0x100101AA` (schaltet `g_relayCommandPendingFlag` scharf, läuft nach 300 Ticks ohne das PB13-Impulsmuster ab).
* INFERRED: Der Befehl startet kein erzwungenes Laden und setzt das BMS weder zurück noch startet er es neu. Die einzige Wirkung ist „Pack-Nummerierung starten“ auf dem CAN-Bus.
  Ohne die PB10-Impulsfolge (die nur Modbus 45029 / die Power-Taste ergänzen) läuft der Handshake wahrscheinlich in ein Timeout und nichts ändert sich.
* INFERRED: Die Bedeutung von `hasAdminOfDevice` / `isDeviceConnected` / `isVirtualExp` ergibt sich nur aus den Funktionsnamen (Rümpfe nicht nachverfolgt).

## 8. Offene Fragen

1. Laufen die Master-Wiederherstellungs-SM / der Balancer mit nur einem Pack (`g_highestPackIndex == 0`)? Nicht belegt.
2. Wer sendet das CAN-ID-Feld `0x210100` (BMS-RAM-Sperre + Ausgänge aus)? Nicht in Control 151 / Micro 116.
3. Nebenbefund (bereits in der Registeraufschlüsselung): `Idle_Watchdog_AutoReboot` sendet Byte 1 zweimal. Bei jedem Nicht-Master-Pack fällt dies in
   den BMS-Werksreset durch.
