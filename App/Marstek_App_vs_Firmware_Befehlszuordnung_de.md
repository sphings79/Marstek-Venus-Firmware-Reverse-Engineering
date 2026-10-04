# Marstek-App vs Control-Firmware: Befehlszuordnung (Venus D, Control 147-151)

> 🇬🇧 English version: [Marstek_App_vs_Firmware_Command_Map.md](Marstek_App_vs_Firmware_Command_Map.md)

Stand 2026-10-02. Ordnet die Venus-relevanten Befehle der Marstek-App 1.6.76 ([Vokabular](Marstek_App_Befehlsvokabular_1.6.76_de.md)) den Handlern der Venus-D-
Control-Firmware zu. Jede Zeile wurde gegen die dekompilierten Dispatcher von Control v151 geprüft: `MQTT_CtrlCommand_Dispatcher@0x080244e0`,
`BLE_App_Command_Dispatcher@0x0800c730` und `MQTT_JSON_RPC_Dispatcher@0x0801f4f8`. Versionsunterschiede wurden in den Dispatchern von
Control 147, 149, 149.2, 150 (Basis 0x4800) und 151 geprüft. Nur statische Analyse; es wurde nichts an ein Gerät gesendet.
Verwandt: [BLE-Befehlsmap v150](../BLE/VNSD_BLE_Command_Map_v150_de.md), [Akkupack-Wiederherstellung, Firmware-Seite](../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md).

## 1. Legende

* **Case** = das Hex-Case-Label des Dispatcher-Switch. Bei MQTT ist es das Hex der dezimalen `cd` (`cd=40` = 0x28, `cd=44` = 0x2c).
* **Status**: ja = Case existiert und die von der App gesendeten Parameter werden geparst; teilweise = Case existiert, aber etwas von der App Gesendetes wird ignoriert oder nur über den Schlüsselnamen zugeordnet; nein = kein Handler.
* **Seit** = erste Control-Version mit dem Handler (Versionen: 147, 149, 149.2, 150, 151).
* Antworten nutzen `MQTT_Publish_CommandResult(cd, result)` = `cd=<n>,ret=<result>`, sofern kein anderes Format angegeben ist.

## 2. Funktionsweise der Dispatcher (in v151 geprüft)

* **MQTT:** Der Handler sucht `cd=` im Payload, nimmt `atoi(...) & 0xff` und schaltet darauf um (`cd=300` würde also auf `cd=44` abgebildet). Ein Payload ohne `cd=` geht an den JSON-RPC-Dispatcher. Der Teil `/App/<id>/` des Topics wird mit der Geräte-ID (`g_mqttTopicId`) und der Gruppen-ID (`g_mqttGroupIdStr`) verglichen; das Ergebnis bleibt als ID-Typ erhalten und eine Abweichung allein stoppt den `cd=`-Dispatch in dieser Funktion nicht. Antworten von `MQTT_Publish_CommandResult` werden auf `marstek_energy/<devType>/device/<devId>/ctrl` veröffentlicht.
* **BLE:** `packet[2]` muss 0x23 sein und das XOR über die ersten `LEN - 1` Bytes muss dem letzten Byte entsprechen; `packet[3]` ist die Befehls-ID. Antworten kommen von `BLE_Send_Response(cmd, data, len)`.
* **JSON-RPC:** wird für Payloads ohne `cd=` (MQTT/Cloud) und von der lokalen UDP-API genutzt. Die Tabelle Id-zu-Name hat 16 Einträge (Id 12 ist unbenutzt).

## 3. MQTT: App-Befehle der Firmware zugeordnet

| App-Befehl | Case | Handler / Wirkung | Geparste Parameter | Antwort | Status | Seit |
|---|---|---|---|---|---|---|
| `cd=1` (status request) | 0x01 | `MQTT_StatusReport_BuildAndPublish(1)`: vollständiger Statusbericht | keine | Statusbericht | ja | 147 |
| `cd=2,md=0` / `md=1` / `md=5,nl=1` | 0x02 | setzt `g_userWorkMode` (EEPROM 0x301); `md=1` löscht zusätzlich `g_workModeUpsSubFlag` | `md=0`, `md=1`, `md=5`; `nl` wird hier **nicht** geparst | `cd=2,md=<mode>` | ja | 147 |
| `cd=3,md=1,nm=<slot>,bt,et,...` (`setTimePower`) | 0x03 | manueller Zeitslot: schreibt die `g_touSlot*`-Arrays, dann `Config_Save_TimeSlotEntries(nm)` | `md=1`, `nm` (Slot < 10), `bt`, `et` (`HH:MM`, mit führender Null), `wk`, `vv`, `as`; alle Pflicht, in dieser Reihenfolge | `cd=3,md=<mode>`; `cd=3,ret=0` bei `nm` >= 10 (nicht in 147) | ja | 147 |
| `cd=3,md=2,id,in,on` (`setTradePriceAndArea`) | 0x03 | kein `md=2`-Zweig: der Befehl läuft durch, nichts wird gespeichert | - | keine Antwort | nein | - |
| `cd=<n>,yy,mm,rr,hh,mn` (`setTime`) | 0x04 | `RTC_SetDateTime` (Jahr - 2000), danach Antwort mit der RTC; `read` liest nur | `read`; `yy`, `mm`, `rr`, `hh`, `mn` in dieser Reihenfolge; ein fehlender Schlüssel bricht ohne Antwort ab | `cd=4,<Y>-<M>-<D> <h>:<m>:<s>` | ja (Schlüssel passen; cd-Literal nicht extrahiert) | 147 |
| `cd=5,rs=1` / `rs=2` | 0x05 | `Factory_Reset(1\|2\|3)` nach der Antwort; ohne `rs` -> `ret=0` | `rs=1`, `rs=2`, `rs=3` | `cd=5,ret=1` | ja | 147 |
| `cd=9,ot=0` | 0x09 | `Quectel_OTA_StartUpdate_AndReboot(0)`: startet das FC41D-Modul-OTA und bootet neu | `ot=0`, `ot=1` | `cd=9,ret=1` vor dem Start; sonst `ret=0` | ja | 147 |
| `cd=10` (`getMqttCurrentTimeState`) | 0x0a | `MQTT_Publish_WiFi_Version`: meldet die FC41D-Version | keine | Versionsstring | teilweise (der App-Funktionsname deutet auf einen anderen Zweck) | 147 |
| `cd=11,bc=<0\|1>` | 0x0b | EPS/Backup: `Config_Mode_Apply(1,bc)`, `g_epsReserveEnable` (EEPROM 0x300) | `bc=0`, `bc=1` | `cd=11,ret=1` (0 wenn das gespeicherte Flag abweicht); keine Antwort bei anderem `bc` | ja | 147 |
| `cd=12,ht=<v>` (`syncServiceToDevice`) | 0x0c | HTTP-/Cloud-Servertyp, EEPROM 0x441 | `ht=0..4` (`op=`, `http=` werden ignoriert) | `cd=12,ret=1`, `ret=0` without valid `ht` | ja | 147 |
| `cd=13` (`_bleTimerInfo`) | 0x0d | `MQTT_Publish_ErrorEventLog`: Fehlerlog | keine | Fehlerlog | teilweise (der App-Funktionsname deutet auf einen anderen Zweck) | 147 |
| `cd=15,vs=<W>` (`setAcCouplerPowerVersion`) | 0x0f | Leistungsklasse: 800 -> `g_workMode`=1 (+ `Config_ScheduleEntries_ClampPower800W`), 2200 -> 3, 2500 -> 0; `Config_Write_U16`, EEPROM 0x90 | `vs` in {800, 2200, 2500} | `cd=15,ret=1\|0` | ja | 147 |
| `cd=16,cp=<W>` | 0x10 | `Config_Write_U16_0x202` (maximale Ladeleistung) | `cp` 300..2500 | `cd=16,ret=1\|0` | ja | 147 |
| `cd=<n>,meter=<v>[,mac=<MAC>]` (`setCtType`) | 0x12 | `Cloud_HTTP_Response_Parser`: setzt den Zählerquellentyp, danach `mac=` (Seriennummer) und `ct_dev=` | `meter` < 13; `mac=`, `ct_dev=` | echot nur den Schlüssel: `cd=18,meter` | ja (Schlüssel passen; cd-Literal nicht extrahiert) | 147 |
| `cd=<n>,dchrg=<v>` (`setPowerReplenishmentMode`) | 0x12 | gleicher Parser: `dchrg` 0/1 -> `Config_Write_Flag_0x367` (MPPT-Kanalsummen-Flag) | `dchrg` | echot nur den Schlüssel | ja (Schlüssel passen; cd-Literal nicht extrahiert) | 147 |
| `cd=<n>,seq_check` (`setSeqCheck`) | 0x12 | gleicher Parser: `Cloud_Response_Action(1)` | `seq_check` | echot nur den Schlüssel | ja (Schlüssel passen; cd-Literal nicht extrahiert) | 147 |
| `cd=18,ct_t=<v>,mac=<MAC>` (E-mini) | 0x12 | `ct_t` ist kein Schlüssel des Parsers: keine Aktion | - | Echo `cd=18,ct_t` | nein | - |
| `cd=19` (`getCtPowerInfo`) | 0x13 | `MQTT_Publish_Grid_Power`; mit `meter_data` zusätzlich `MQTT_Publish_Meter_Data`; `debug` -> `MQTT_ConnectionStatus_Report_Send` (nur 151) | `meter_data`, `debug` | CT-Leistungs-Payload | ja | 147 |
| `cd=20,len=<n>,url=<URL>` (`updateAcCouplerFc4Firmware`) | 0x14 | FC41D-OTA per URL: mit `port=` sofort `Quectel_OTA_StartUpdate_FromURL_AndReboot`, sonst `g_otaUrlStartPendingFlag=1` | `len` (< 255), `url=`, optional `port=`, in dieser Reihenfolge | `cd=20,ret=1` (vor dem Start gesendet); `ret=0` bei `len` >= 255 | ja | 147 |
| `cd=22` (`notifyAcCouplerUpgradePowerVersion`) | 0x16 | `g_activeUpgradeInfoFlag = 1` | keine | `cd=22,ret=1` | ja | 147 |
| `cd=<n>` + `len`/`ip=` (`setP1MeterIp`) | 0x18 | P1-/Zähler-IP: setzen (`len`, `ip=`) oder abfragen (`ip?`), EEPROM 0x3500; nur gespeichert (der BLE-Zwilling 0x21 gilt in der BLE-Map als wirkungslos) | `ip?`; `len` (<= 16), `ip=` | `cd=24,ip=<IP>` | ja (Schlüssel passen; cd-Literal nicht extrahiert) | 147 |
| `cd=<v>` (`getDeviceIp`) | 0x1a | `MQTT_Publish_Network_Info` | `net?` | Netzwerkinfo | ja (cd-Literal nicht extrahiert) | 147 |
| `cd=<n>,num=<v>` (`requestNowData`) | 0x1d | `MQTT_Publish_PowerSchedule(num)` | `num=` (`len=` wird ignoriert) | Plan-Payload | teilweise (cd-Literal nicht extrahiert) | 147 |
| `cd=30,api=<0\|1>,port=<p>` | 0x1e | lokale API: `g_localApiEnable` (EEPROM 0x371), `g_localApiUdpEnable`, `g_apiConnState` (1 / 4), Port EEPROM 0x372 | `api` (0/1) vor `port`; `port` wird nach `api` gesucht, also beide Schlüssel in dieser Reihenfolge nötig | `cd=30,ret=<enable>,port=<port>` | ja | 147 |
| `cd=40,number_as` | 0x28 | `BLE_Apply_ConfigReg_Offset3(0)` -> `Config_Apply_SingleReg(3)` -> CAN `0x100101AA` Byte 3 (Pack-Codierungs-Handshake); `bms_rst` -> Index 1 (Byte 4, BMS-Neustart; die App sendet das nie) | `number_as`, `bms_rst` | `cd=40,ret=1`; `ret=0` otherwise | ja | 147 |
| `cd=42,bms_idx=255` | 0x2a | `MQTT_Publish_BMS_Data(idx)` | `bms_idx` 0..8 oder 255 (andere Werte werden zu 0) | BMS-Daten-Payload | ja | 147 |
| `cd=51` (`getHttpOtaState`) | 0x33 | `MQTT_Publish_Network_Status`: OTA-Status | keine (`url=` der E-Auto-Lader-Variante wird ignoriert) | OTA-Status-Payload | ja | 147 |
| `cd=52,clr=<v>,cnt=<v>` | 0x34 | `clr=1` löscht die 4 URL-Slots (Flash); `cnt` -> `g_otaModuleCount` (max. 4) | `clr`, danach `cnt` | `cd=52,ret=<cnt<5>`; keine Antwort ohne `cnt` | ja | 147 |
| `cd=53,num=<v>,<entries>` | 0x35 | `Quectel_MQTT_OTA_Info_Parser` (Komma-Token `mod`, `type`, `size`, `crc`, `ver`, URL); `check?` -> `OTA_BuildAndSend_StatusReport` | `check?` or the entries | `cd=53,ret=<n>` | ja | 147 |
| `cd=54,start` | 0x36 | `g_otaUpdatePendingFlag = true` (startet das ausstehende OTA) | `start` | `cd=54,ret=1` | ja | 147 |
| `cd=56,dod=<v>` | 0x38 | `Cloud_DOD_Command_Handler`: `Config_Set_DischargeCutoff_WithRelay` | `dod` (reply ret=1 for 30..88) | `cd=56,ret=1\|0` (vom Handler selbst gesendet) | ja | 147 |
| `cd=63,...` (`setVenusPeakShaving`) | 0x3f | Peak Shaving: `Config_Write_PowerClampMode_0x394(1, vv)` und `Config_Write_PowerOffset(vv)`; `as=0` schaltet aus und schreibt Offset 0 | `as` (0/1), danach `vv` (W, vorzeichenbehaftet) | `cd=63,ret=1\|0` | ja | 150 |
| `cd=72,type?` / `type=1,len,ip,mac` (EV charger binding) | - | kein Case: der höchste Case ist 0x42 (66) | - | keine Antwort | nein | - |

## 4. MQTT-Cases ohne Venus-App-Gegenstück im Vokabular

Die App-Extraktion enthält für diese Cases keinen Venus-Sender. Builder mit zur Laufzeit gebauter `cd`-Nummer (`cd=<n>`) lassen sich nicht ausschließen.

| cd | Case | Funktion | Parameter | Antwort | Seit |
|---|---|---|---|---|---|
| 0 | 0x00 | Upload-Zielmaske | `set=1\|2\|3` | `cd=0,ret=<mask>` | 147 |
| 6 | 0x06 | Entwicklerdaten: `rd=1` uid, `rd=2` Bootloader-Version, `rd=3` HTTP-Zähler / Slot / Zeit, sonst Develop-Mode-Info | `rd=1..3` | - | 147 |
| 7 | 0x07 | Developer-Modus `dm`: 0 Modem-Hardwarereset, 1 Reboot, 2 TLS-Nutzerzertifikat löschen + Reboot, 3 `Inverter_Set_Flag_Reg_0x05(0)`, **4 `Config_Apply_SingleReg(4)` = BMS-Neustart (Byte 4)**, 5 `Inverter_Set_Reg_0x04(0)`, 6 `CH395_Reset_And_Reinit` | `dm=0..6` | `cd=7,ret=1\|0` | 147 |
| 8 | 0x08 | Auto-Wechsel der Betriebsart, EEPROM 0x366 | `dt=0\|1` | `cd=8,ret=1\|0` | 147 |
| 14 | 0x0e | BMS-Dateninfo (`BMS_StatusQuery_Response_Send`) | - | - | 147 |
| 17 | 0x11 | maximale Entladeleistung (<= 2500, <= 800 bei der 800-W-Klasse) | `dp` | `cd=17,ret=1\|0` | 147 |
| 21 | 0x15 | Ereignislog (`MQTT_Publish_Event_Log`) | - | - | 147 |
| 23 | 0x17 | Parallelbetrieb | `pm=0\|1\|2` | `cd=23,ret=<mode>` | 147 |
| 25 | 0x19 | Generator-Flag (nur RAM) | `gen=0\|1` | `cd=25,ret=<flag>` | 147 |
| 27 | 0x1b | Jump-Steuerung, EEPROM 0x92; `jp=0` erzwingt zusätzlich Modus 1 | `jp=0\|1` | `cd=27,jp=<n>` | 147 |
| 28 | 0x1c | **Neustart** nach der Antwort und 1 s Verzögerung | - | `cd=28,ret=1` | 147 |
| 31 | 0x1f | Vid-Info | `topic=1` | `cd=31,vid=...` | 147 |
| 41 | 0x29 | BMS-Version (`MQTT_Publish_BMS_Version`) | - | - | 147 |
| 43 | 0x2b | Überschusseinspeisung, `Config_Write_BoolFlag_0x375` | `full_d=0\|1` | `cd=43,ret=<flag>` | 147 |
| 44 | 0x2c | BMS-Fehlersperre: `Config_Apply_WorkModeReg(lock)` -> Byte 8 (Entsperren + Master-Neustart) / Byte 9 (Sperren) | `lock=0\|1` | `cd=44,ret=1` | 147 |
| 45 | 0x2d | MPPT-Daten (`MQTT_Publish_Inverter_Telemetry`) | - | - | 147 |
| 50 | 0x32 | VNS-MPPT-Daten (Logtext "Set ems ctrl info"; der Handler veröffentlicht nur) | - | - | 147 |
| 55 | 0x37 | BLE-Advertising aktivieren | `adv=0..3` | `cd=55,ret=<flag>` | 147 |
| 57 | 0x39 | Self-Control-Leistungsoffset (`Cloud_Handle_SelfCtl_Power`) | `selfctl_power=` | `cd=57,selfctl_power=<W>` | 147 |
| 58 | 0x3a | KI-Strategie-Abfrage | - | - | 147 |
| 59 | 0x3b | LED-Zustand (EEPROM 0x395 erst seit 151) | `led=0\|1` | `cd=59,ret=<led>` | 147 |
| 61 | 0x3d | `g_backupDataRequestFlag` | `bk_en=` | `cd=61,ret=<n>` | 149 |
| 62 | 0x3e | Geräte-IDs (`MQTT_Publish_DeviceIDs`); nur 149 / 149.2 | - | - | 149 |
| 64 | 0x40 | Geräte-IDs (`MQTT_Publish_DeviceIDs`) | - | - | 150 |
| 65 | 0x41 | INV-/Netzstatus (`MQTT_Publish_Grid_Status`) | - | - | 150 |
| 66 | 0x42 | INV-Debug: `debug=1` Wartungszustand 2, `2` Betriebsart-Konfig, `3` / `4` `CAN_Write_Category0x18(1/0)` (4 löscht zusätzlich den Wartungszustand) | `debug=1..4` | `cd=66,ret=...` | 151 |

## 5. App-Befehle ohne Venus-D-Handler (oder nur teilweise behandelt)

| App-Befehl | Produkt | Ergebnis an einem Venus D (Control 147-151) | Hinweis |
|---|---|---|---|
| `cd=72,type?` / `cd=72,type=1,len=<n>,ip=<IP>,mac=<MAC>` | Venus: E-Auto-Lader-Bindung | kein Case in 147-151 (höchster Case 66) | braucht neuere Firmware |
| BLE cmd 0x61 (`0B` / `0A`+ip`,`mac) | Venus: E-Auto-Lader-Bindung, BLE-Fallback | kein BLE-Case (höchster 0x5B) | braucht neuere Firmware |
| `cd=3,md=2,id=<v>,in=<v>,on=<v>`; BLE 0x09 payload `02` | Venus: Handelspreis und Gebiet | Case 3 / BLE 0x09 haben keinen Modus-2-Zweig; ignoriert (BLE antwortet trotzdem mit dem unveränderten Modus) | in 147-151 nicht unterstützt |
| `cd=2,md=5,nl=1` | Venus: Sparmodus | Case 2 setzt Modus 5, parst aber `nl` nicht (nur Case 3 liest `nl` und echot es nur) | `nl` ohne Wirkung |
| `cd=18,ct_t=<v>,mac=<MAC>` | Venus E-mini: CT-Typ | Case 0x12 kennt `meter=`, nicht `ct_t=`; Echo ohne Aktion | anderes Produkt |
| `cd=12,op=0` / `cd=12,op=1,http=<URL>` | E-Auto-Lader A3600CE: Server-URL | Case 0x0c parst nur `ht=`; ohne das lautet die Antwort `cd=12,ret=0` | anderes Produkt |
| `cd=46`, `cd=47,<k>=<v>` | E-Auto-Lader / HMH | kein Case (46-49 in allen Versionen nicht vorhanden) | anderes Produkt |
| `cd=51,url=<URL>` | E-Auto-Lader: OTA-Auslöser | Case 0x33 ignoriert alle Parameter und meldet nur den OTA-Status; es wird kein OTA gestartet | anderes Produkt |
| `cd=67` | Neue Module: WLAN-Scanliste | kein Case (höchster 66) | braucht neuere Firmware oder anderes Produkt |
| `cd=44` (bare), `cd=45` (bare) | V6000 / HMH / neue Module | 0x2c braucht `lock=` (sonst keine Antwort); 0x2d meldet MPPT-Daten und ignoriert Parameter | anderes Produkt |
| `cd=63,ct_chg_type=`, `cd=64,ups_p=`, `cd=65,lapi_en=,self_port=,remote_port=` | Neue Module (Venus X / E-mini) | 63 = Peak Shaving (braucht `as=`; sonst `ret=0`), 64 = Geräte-IDs, 65 = Netzstatus: kein Setter für diese Parameter; 147 / 149 / 149.2 haben keinen Case 63-65 (149 / 149.2 nur 62) | anderes Produkt |
| `cd=90`, `cd=91,type=<t>`, `cd=68` | HMH-Familie | kein Case | anderes Produkt |
| `cd=09,wy=<v>` | generische Zeitzonen-Seite (gemeinsam) | Case 0x09 braucht `ot=`; ohne das lautet die Antwort `cd=9,ret=0`, sonst passiert nichts | kein Zeitzonen-Setter in 147-151 (die Zeit wird mit `cd=4` / BLE 0x0B gesetzt) |
| `cd=30,data=<v>` | Antwortfeld in der App | keine Sendevorlage; `data=` wird von Case 0x1e nicht geparst | entfällt |
| BLE builders 0x38 (`httpOtaPrepare`) and 0x3A (`startHttpOta`) | Venus: HTTP-OTA | kein BLE-Case 0x38 / 0x3A; der BLE-OTA-Pfad der Firmware ist Case 0x52 mit Unterbefehlen | über BLE in 147-151 nicht unterstützt |

## 6. Gleiche `cd`-Nummer, anderes Produkt

Das Neue-Module-Framework (Venus X / E-mini / G) und andere Produkte verwenden kleine `cd`-Nummern mit anderer Bedeutung. Was ein Venus D v151 tut, wenn so ein Befehl ankommt (nur zur Information; die App sendet diese an die eigenen Topics der anderen Produkte):

| `cd` | App-Funktion (neue Module / andere) | Case in v151 | Was das Venus D tut |
|---|---|---|---|
| `2` | `CMD_SET_WORK_MODE` | 0x02 | ohne `md` ändert sich nichts; die Antwort enthält den aktuellen Modus |
| `3` | `CMD_GET_NET_INFO` | 0x03 | ohne `md=1` / `md=5` passiert nichts, keine Antwort (Netzwerkinfo ist hier `cd=26` + `net?`) |
| `4` | `CMD_GET_ERROR_CODE` | 0x04 | liefert die RTC-Zeit (das Fehlerlog ist hier `cd=13`) |
| `19` | `CMD_GET_CT_POWER` | 0x13 | gleiche Bedeutung: CT-Leistung |
| `33, 38, 46, 60, 67` | `CMD_SET_TIME`, `CMD_SET_GRID_POWER_STANDARD`, `CMD_SET_ACCESS_POWER`, `CMD_SET_SERVER`, `CMD_GET_SCAN_WIFI_LIST` | - | kein Case in 147-151 |
| `62` | `CMD_GET_BATTERY_PACK_INFO`, `CMD_SET_SHELLY_PRO_3EM_PORT` | 0x3e (149 / 149.2 only) | Geräte-IDs, nur lesend; kein Case in 147 / 150 / 151 |
| `44` | `CMD_SET_DOD` | 0x2c | braucht `lock=`; ein nacktes `cd=44` tut nichts und antwortet nicht (DoD ist hier `cd=56,dod=`) |
| `54` | `CMD_SET_ANTL_REVERSE` | 0x36 | braucht den Teilstring `start`; sonst nichts |
| `55` | `CMD_SET_BLUETOOTH_STATE` | 0x37 | braucht `adv=`; sonst nichts |
| `56` | `CMD_SET_LED_STATE` | 0x38 | braucht `dod=`; sonst nichts (LED ist hier `cd=59,led=`) |
| `57` | `CMD_SET_MAX_GRID_POWER` | 0x39 | braucht `selfctl_power=`; sonst nichts |
| `58` | `CMD_SET_CT_PHASE_AUTO_CHECK` | 0x3a | nur lesend: KI-Strategie-Abfrage |
| `59` | `CMD_GET_NOW_POWER` | 0x3b | `led=` wird ohne NULL-Prüfung gelesen (siehe Abschnitt 10) |
| `61` | `CMD_SET_REBOOT` | 0x3d | `bk_en=` wird ohne NULL-Prüfung gelesen (siehe Abschnitt 10); kein Reboot |
| `63, 64, 65` | `CMD_SET_CT_CHARGE_TYPE`, `CMD_SET_UPS_CHARGE_POWER`, `CMD_SET_LOCAL_API` | 0x3f / 0x40 / 0x41 | Peak Shaving (braucht `as=`) / Geräte-IDs / Netzstatus; kein Setter |
| `66` | `getUpgradeStatus` (OTA status query) | 0x42 | `debug=` wird ohne NULL-Prüfung gelesen (siehe Abschnitt 10) |
| `28` | `getCtPowerInfo` (CT variant) | 0x1c | **startet die Control neu**; die CT-Variante nie an ein Venus D senden |
| `3 (md=1)` | B2500 `cd=03,md=1` | 0x03 | setzt manuellen Modus 1 ohne Slot |

## 7. BLE: App-Frames dem `BLE_App_Command_Dispatcher` zugeordnet

Frameformat `73 | LEN | 23 | CMD | Payload | XOR`. Aufgeführt sind nur die im Vokabular Venus-bezogenen App-Befehle; die vollständige BLE-Tabelle der Firmware steht in der [BLE-Befehlsmap v150](../BLE/VNSD_BLE_Command_Map_v150_de.md).

| App-BLE-Cmd | App-Builder / Konstante | Case | Handler / Wirkung | Seit | Status |
|---|---|---|---|---|---|
| 0x06 | `ACCOUPLE_FACTORY_RECOVERY` (`02`), `POWER_SEND_RESET` (`01`) | 0x06 | `Factory_Reset(1\|2\|3)` für Byte 4 = 1/2/3; die Antwort echot zuerst das Byte | 147 | ja |
| 0x09 `00` | `ACCOUPLE_WORK_MODE_AUTO` | 0x09 | Byte 4 = 0: `g_userWorkMode = 0`, EEPROM 0x301, Antwort = Modus | 147 | ja |
| 0x09 `01` (LEN 0x0F) | `ACCOUPLE_SEND_MANUAL_SET_POWER` | 0x09 | Byte 4 = 1: Slot [5] (< 10), Start [6..7] = hh, mm, Ende [8..9], Wochentagsmaske [10], Leistung int16 LE [11..12], Enable [13]; `Config_Save_TimeSlotEntries` | 147 | ja |
| 0x09 `02` (LEN 0x10) | `ACCOUPLE_SEND_TRADE_PRICE` | 0x09 | Byte 4 = 2: kein Zweig; die Antwort ist der unveränderte Modus (derselbe Wert wird wieder in EEPROM 0x301 geschrieben) | - | nein |
| 0x15 (LEN 7) | `ACCOUPLE_SET_DISCHARGE_POWER_VERSION` | 0x15 | u16 LE in {800, 2200, 2500} -> Leistungsklasse (`g_workMode` 1 / 3 / 0, EEPROM 0x90); andere Werte werden ohne Antwort ignoriert | 147 | ja |
| 0x16 | builder `setAcCouplerMaxChargePower` | 0x16 | u16 LE 300..2500 -> `Config_Write_U16_0x202`; Antwort = gespeichertes u16 | 147 | ja |
| 0x1A | `ACCOUPLE_GET_CT_POWER_INFO` | 0x1a | Payload ignoriert; liefert 18 Bytes CT-Werte (`g_ctPowerCh1..3`, `g_ctPowerNow`, Netz-Sample) | 147 | ja |
| 0x28 | builder `setLocalApiEnable` | 0x28 | Byte 4 = 0/1 Enable; Port u16 LE [5..6] wird in jedem Fall geschrieben (EEPROM 0x372); `g_apiConnState` = 3 / 4 (MQTT setzt 1 / 4) | 147 | ja |
| 0x29 | builder `setVenusPeakShaving` | 0x29 | Byte 4 = 1/0 Peak-Zustand, int16 LE [5..6] = W; `Config_Write_PowerClampMode_0x394` und `Config_Write_PowerOffset` | 150 | ja |
| 0x40 `00` | builder `setBMSCodeFunction` | 0x40 | `BLE_Apply_ConfigReg_Offset3(Byte 4)`: 0 -> CAN-Byte 3 (Pack-Codierung, das sendet die App); 1 -> Byte 4 (BMS-Neustart, sendet die App nie); die Antwort echot Byte 4 | 147 | ja |
| 0x50 / 0x51 | builders `setVid` / `requestVid` | 0x50 / 0x51 | Vid- und Xid-Info schreiben (Sub `0x0A`, String bis 32 Byte) / lesen | 147 | ja |
| 0x38 / 0x3A | builders `httpOtaPrepare` / `startHttpOta` | - | kein Case; der BLE-OTA-Pfad ist 0x52 (Unterbefehle 0x0A-0x0D) | - | nein |
| 0x61 `0B` / `0A`... | EV charger binding (`cd=72` twin) | - | kein Case (höchster 0x5B) | - | nein |

## 8. JSON-RPC-Methoden (nur Firmware)

Die App-Extraktion enthält kein JSON-RPC; die Methoden nutzen die lokale UDP-API und der MQTT-JSON-Pfad.

| Id | Methode | Parameter | Builder / Ergebnis | Seit |
|---|---|---|---|---|
| 1 | `Marstek.GetDevice` | `params.ble_mac` = `"0"` oder die eigene BLE-Id des Geräts; sonst Fehler `Invalid params` | `MQTT_BuildDeviceDataResponse` | 147 |
| 2-6, 8-10 | `Wifi.GetStatus`, `BLE.GetStatus`, `Bat.GetStatus`, `PV.GetStatus`, `ES.GetStatus`, `ES.GetMode`, `EM.GetStatus`, `Marstek.GetData` | `params.id` (von `MQTT_ParseRpcParams` geprüft) | Builder `MQTT_BuildWifi/Ble/Battery/Pv/Es/EsMode/EnergyMeter/MarstekDataResponse` | 147 |
| 7 | `ES.SetMode` | `config.mode`: `Auto` (`auto_cfg.enable`=1 -> `MQTT_SetWorkMode(1)`), `AI` (`ai_cfg.enable`=1 -> `MQTT_SetWorkMode(2)`), `Manual` (`manual_cfg`: `time_num` <= 9, `start_time`, `end_time` als `HH:MM`, `week_set` <= 0x7f, `power` +-2500, `enable` 0/1), `Passive` (`passive_cfg.power`, `cd_time` != 0 -> `MQTT_PassiveModeStart`), `UPS` (`ups_cfg.enable` 1: manueller Modus + EPS an + UPS-Subflag; 0: Modus 0, EPS aus) | `MQTT_RPC_BuildSetResult` | 147 |
| 11 | `DOD.SET` | `params.value` 30..88 -> `Config_Set_DischargeCutoff_WithRelay` | set result | 147 |
| 13 | `Ble.Adv` | `params.enable` 0/1 -> EEPROM 0x36bd | set result | 149 |
| 14 | `Led.Ctrl` | `params.state` 0/1 -> `g_selLedEnable` (EEPROM 0x395 erst seit 151) | set result | 149 |
| 15 | `Ota.Set` | `params.config`: `device_type` (einer von 4 Slot-Namen), `Mod` 1..4, `Version`, `URL`, `URL_LEN` (muss der URL-Länge entsprechen), `SIZE`, `CRC` -> URL-Slot im Flash, `g_otaUpdatePendingFlag = true` | set result | 149 |
| 16 | `Set.Ver` | `params.version` in {800, 2200, 2500} | set result | 149 |
| 17 | `Reset.Factory` | `params.type` 1 / 2 -> `Factory_Reset`; 3 -> nur Ergebnis, keine Aktion; sonst -> `Invalid params` | set result | 149 |

## 9. Versionsunterschiede (Control 147 / 149 / 149.2 / 150 / 151)

Der MQTT-Dispatcher behandelt dezimal 0-31, 40-45, 50-59 in allen Versionen; 61 ab 149, 62 nur in 149 / 149.2, 63-65 ab 150, 66 nur in 151 (147 hat weder 61 noch 63-66; 149 / 149.2 haben 61 und 62; 150 hat 61 und 63-65; 151 hat 61 und 63-66). `x` = vorhanden, `-` = fehlt.

| Merkmal | 147 | 149 | 149.2 | 150 | 151 |
|---|---|---|---|---|---|
| MQTT-Cases 0-31, 40-45, 50-59 | x | x | x | x | x |
| MQTT 61 (`bk_en=`) | - | x | x | x | x |
| MQTT 62 (Geräte-IDs) | - | x | x | - | - |
| MQTT 63 (Peak Shaving) | - | - | - | x | x |
| MQTT 64 (Geräte-IDs), 65 (Netzstatus) | - | - | - | x | x |
| MQTT 66 (`debug=`) | - | - | - | - | x |
| MQTT 19 Option `debug` | - | - | - | - | x |
| MQTT 3 Fehlerantwort `cd=3,ret=0` (`nm` >= 10) | - | x | x | x | x |
| MQTT 40 (`number_as` / `bms_rst`), 44 (`lock=`), 5, 16, 30, 52-54, 56 (gleiche Strings und Aufrufe) | x | x | x | x | x |
| BLE 0x29 (Peak Shaving) | - | - | - | x | x |
| BLE 0x5A, 0x5B | - | x | x | x | x |
| BLE 0x40 (`@`), 0x43 (`C`), 0x28, 0x16, 0x15, 0x1A, 0x50, 0x51 | x | x | x | x | x |
| JSON-RPC-Ids 1-11 (ohne 12) | x | x | x | x | x |
| JSON-RPC-Ids 13-17 (`Ble.Adv`, `Led.Ctrl`, `Ota.Set`, `Set.Ver`, `Reset.Factory`) | - | x | x | x | x |
| LED-Einstellung gespeichert (EEPROM 0x395): MQTT 59, BLE 0x59, `Led.Ctrl` | - | - | - | - | x |

## 10. Beobachtungen und offene Punkte

1. **Keine NULL-Prüfung** in den MQTT-Cases 0x3b (`led=`), 0x3d (`bk_en=`) und 0x42 (`debug=`): das Ergebnis von `strstr` geht als `ptr + 4/6/6` an `atoi`. Ein nacktes `cd=59`, `cd=61` oder `cd=66` (die Modul-Codes) liest daher von einer Adresse nahe null. Die Wirkung am echten Gerät wurde **nicht getestet**; bei 0x3b kann es `g_selLedEnable` (und seit 151 EEPROM 0x395) überschreiben. Gemeldet, nicht geändert.
2. MQTT-Case 0x1e (`cd=30`) sucht `port` per `strstr(ptr, ...)` nach der `api`-Suche, ohne das erste Ergebnis zu prüfen; ein Payload ohne `api` gibt NULL an `strstr`. Die App sendet immer zuerst `api`, löst das also nicht aus.
3. Mehrere Handler suchen ihre Schlüssel nacheinander, jedes `strstr` beginnt beim vorherigen Treffer, daher **zählt die Schlüsselreihenfolge** (Case 3: `nm`, `bt`, `et`, `wk`, `vv`, `as`; Case 4: `yy`, `mm`, `rr`, `hh`, `mn`; Case 30: `api`, `port`; Case 52: `clr`, `cnt`; Case 20: `len`, `url`, `port`; Case 63: `as`, `vv`). Die App-Vorlagen halten diese Reihenfolge ein. Zeiten werden mit `atoi(p + 3)` und `atoi(p + 6)` gelesen, daher muss `HH:MM` mit führender Null stehen.
4. Case 3 schreibt `bt`, `et`, `wk`, ... einzeln in RAM-Arrays; fehlt ein späterer Schlüssel, wird der Slot **nicht** gespeichert und es kommt keine Antwort, aber die früheren Felder sind im RAM bereits geändert.
5. `cd=56` (DoD): `Cloud_DOD_Command_Handler` ruft `Config_Set_DischargeCutoff_WithRelay` **vor** der Bereichsprüfung der Antwort auf (ret=1 nur für 30..88). Ob der Setter intern prüft, wurde nicht nachverfolgt. JSON-RPC `DOD.SET` prüft 30..88 vor dem Aufruf.
6. App-Zeilen mit „cd-Literal nicht extrahiert“ (`setTime`, `setCtType`, `setPowerReplenishmentMode`, `setSeqCheck`, `setP1MeterIp`, `getDeviceIp`, `requestNowData`): die App baut die `cd`-Nummer zur Laufzeit; die Zuordnung beruht nur auf Parameternamen.
7. `cd=10` und `cd=13`: die App-Funktionsnamen (`getMqttCurrentTimeState`, `_bleTimerInfo`) passen nicht zu den Firmware-Handlern (FC41D-Version, Fehlerlog); was die App damit will, wurde nicht nachverfolgt.
8. Die Zuordnung JSON-RPC-Id zu Name stammt aus dem Roh-Decode von `g_mqttRpcMethodTable` in der v150-Analyse (die v151-Tabelle wurde nicht erneut dekodiert); Case-Nummern und Parameter des Dispatchers wurden aus dem v151-Quelltext gelesen. Die App-Extraktion enthält kein JSON-RPC (nur `cd=` und HM-BLE), die Methoden haben hier daher kein App-Gegenstück.
9. Nicht geprüft: Control-Images für Venus A (1509) und E3 (151), Livebetrieb an einem Gerät und die Antworten auf dem Draht (die Antwortformate stammen aus den `snprintf`-Strings der Handler).
10. Die Versionsspalten wurden aus den Case-Listen und dem String-/Aufrufinhalt der Dispatcher-Quellen von 147, 149, 149.2, 150 (Basis 0x4800) und 151 abgeleitet; Parameter-Literale rendert der Decompiler teils unterschiedlich (zum Beispiel fehlt `cp` im `strstr`-Argument mancher Versionen), ohne Codeunterschied.

Was die BMS-Seite mit `cd=40` macht (CAN `0x100101AA` Byte 3), steht in [VNSD_Akkupack_Wiederherstellung_Firmware_de.md](../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md).
