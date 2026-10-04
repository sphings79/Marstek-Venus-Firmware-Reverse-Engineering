# Marstek-App 1.6.76: Befehlsvokabular

> 🇬🇧 English version: [Marstek_App_Command_Vocabulary_1.6.76.md](Marstek_App_Command_Vocabulary_1.6.76.md)

Stand 2026-10-02. Statische Analyse der arm64-`libapp.so` (Dart 3.8.1) der Marstek-App (Paket `com.hamedata.marstek`) Version 1.6.76,
mit Blutter disassembliert und mit den Strings von 1.6.72 verglichen. Kein Gerät, kein Cloud-Login, nichts wurde ausgelöst. Das Vokabular listet, was die
App **senden** kann (MQTT-`cd=`-Payloads und HM-BLE-Frames). Welche davon die Venus-D-Firmware (Control 147-151) behandelt, steht in
[Marstek_App_vs_Firmware_Befehlszuordnung_de.md](Marstek_App_vs_Firmware_Befehlszuordnung_de.md).

## 1. Lesehilfe

* Die `cd`-Nummerierung gilt **pro Produkt**. Dieselbe Nummer bedeutet bei verschiedenen Produkten etwas anderes. Der Quellort in der App (Spalte „App-Funktion(en)“) zeigt, zu welchem Produkt eine Vorlage gehört.
* `<v>` = Laufzeitwert, `<n>` = die `cd`-Nummer ist eine Variable und kein Literal in dieser Funktion, `<HH:MM>` = Uhrzeit mit führender Null. `...` = weitere Parameter, die die Extraktion nicht erfasst hat.
* „Von Hand geprüft“ = von Hand durch das Disassembly nachverfolgt. Alle übrigen Zeilen sind **automatisch extrahierte** Sendevorlagen; ihre Bedeutung stammt nur aus dem Funktionsnamen.
* „Neu in 1.6.76“ (NEU) = String, der in 1.6.72 nicht vorkommt.
* Behandelt wird nur die Sendeseite. Parser-, Antwort- und Mock-Funktionen wurden herausgefiltert.
* † in einer BLE-Spalte = die Zuordnung zum `cd=`-Befehl beruht nur auf dem Konstantennamen (nicht nachverfolgt).

## 2. Transportformate

**MQTT.** Topic `hame_energy/<devType>/App/<devId>/ctrl`, Payload ASCII `cd=<n>,k=v,k=v,...`, zum Beispiel `cd=40,number_as` oder `cd=16,cp=<W>`.
Das E-Auto-Ladegerät A3600CE nutzt ein eigenes Topic: `marstek_energy/A3600CE-A0/App/<encMac>/ctrl`.

**BLE (HM-Frame).** `73 | LEN | 23 | CMD | Payload... | XOR`.

* `73` Magic, `23` fest, `LEN` = Gesamtlänge des Frames inklusive XOR-Byte, `CMD` = Befehls-ID, `XOR` = XOR über alle vorangehenden Bytes (`calCrcValueListWithMutableList`).
* Beispiel: `73 06 23 40 00 16` = Cmd 0x40, Payload `00`, XOR 0x16 (0x73 ^ 0x06 ^ 0x23 ^ 0x40 ^ 0x00).

**Transportwahl** (`CommonCommand._doOperate`, geprüft): Ist der aktuelle Gerätetyp leer, wird nichts gesendet; bei BLE-Verbindung läuft die BLE-Closure;
sonst veröffentlicht die MQTT-Closure, wenn die Cloud-Verbindung online ist. Befehle in `retryOnTimeoutWithNoTag` werden bis zu 3-mal mit 5 s Timeout gesendet.

## 3. Produktfamilien und Quellbereiche

| Familie | App-Quellbereiche | Geräte / Hinweis |
|---|---|---|
| B2500 / HMA-HMJ | `BatterB2500/`, `newBatterB2500/` | Speicherfamilie B2500 |
| Venus | `Ac_Coupler/`, Venus functions in `common/common_command` | VNSD, VNSA, VNSE3, VNSE4, EMAX, VEPRO, VEPAC (+ HMG*) |
| Jupiter | no own source area in the extraction | HMM, HMN, JPLS (nur die BMS-Codierungs-Variante ist zugeordnet) |
| V6000 / M5000 / other power stations | `ColorScreenPower/`, `Home/BigPower/`, `Home/mars_se_module/`, `Widget/base_setting_page` | V6000, M5000, BigPower, Mars SE (Modulnamen in der App) |
| CT | `ct_view_model`, `modules/devices/entity/ct/`, CT functions in `common/common_command` | Zähler CT002 / CT003 |
| HMH (incl. heat pump, EV, generator) | `Home/new_home/view/tool/new_home_MqttTool` | HMH-Familie |
| EV charger A3600CE | `Ac_Coupler/evCharger/connectors/ev_charger_connector` | eigenes Topic `marstek_energy/A3600CE-A0/App/<encMac>/ctrl` |
| New modules (Venus X / E-mini / G) | `modules/devices/...` | Modul-Framework mit `CMD_*`-Entity-Methoden |
| Sun / inverter monitor | `All_Big_litle_Sun/`, `InverterMonitor/` | Produktzuordnung nicht verifiziert |
| Shared / unattributed | generic pages, `utils`, parts of `common/common_command` | keine Produktzuordnung möglich |

## 4. Venus (VNSD / VNSA / VNSE3 / VNSE4 / EMAX / VEPRO / VEPAC)

### 4.1 Gerätetyp-Freigabe (geprüft)

* `DeviceInfo.isVNSD` ist wahr für die Venus-Typen VNSD, VNSA, VNSACH, VEPRO, VEPROCH, VNSDCH, VNSD2, VEPRO2, VNSA2.
* `DeviceInfo.isStackDev` = `isVNSD` oder VEPAC. Die Zeile „Battery Pack Recovery“ der Einstellungsseite wird nur für Stack-Geräte gebaut (nicht für Venus E/C, E3, E4, EMAX, VDAC). Eine Firmware-Versionsprüfung wurde nicht gefunden.
* `DeviceInfo.isVenus` = HMG1 / HMG25 / HMG50, VNSE3*, die `isVNSD`-Familie, VDAC, VNSE4, VNSEMAX, VEPAC.
* Venus E-mini / X / G nutzen das getrennte Modul-Framework (Abschnitt 11) mit anderer `cd`-Nummerierung.

### 4.2 Befehle

Zeilen mit „ja“ in der Spalte „Von Hand geprüft“ stammen aus Abschnitt A der Extraktion (von Hand geprüft), der Rest ist extrahiert.

| cd | Parameter | BLE-Befehl | Bedeutung | App-Funktion(en) | Von Hand geprüft | Neu in 1.6.76 |
|---|---|---|---|---|---|---|
| `cd=1` | - | - | Status / Gerätedaten anfordern (Init der Betriebsart-Seite, Geräteinfo des OTA-Dienstes) | `ac_workMode_setting::initState`, `venus_http_ota_progress_service::_requestDeviceInfo` |  |  |
| `cd=2` | `md=0` | 0x09 Payload `00` (`ACCOUPLE_WORK_MODE_AUTO`) † | Auto-Modus | `CommonCommand::setAcAutoMode`, `setAutoPower` | ja |  |
| `cd=2` | `md=1` | - | Manueller Modus | `CommonCommand::setAcManualMode` | ja |  |
| `cd=2` | `md=5,nl=1` | - | Sparmodus (Economy) | `CommonCommand::setAcEconomyMode` | ja |  |
| `cd=3` | `md=1,nm=<slot>,bt=<HH:MM>,et=<HH:MM>,...` | 0x09 Payload `01`, Frame-Anfang `73 0F 23 09 01` (`ACCOUPLE_SEND_MANUAL_SET_POWER`) † | Manueller Zeitslot (Leistungsplan-Eintrag) | `venus_helper::setTimePower`, `power_setting_page::setTimePower` |  |  |
| `cd=3` | `md=2,id=<v>,in=<v>,on=<v>` | 0x09 Payload `02`, Frame-Anfang `73 10 23 09 02` (`ACCOUPLE_SEND_TRADE_PRICE`) † | Handelspreis und Gebiet | `trade_setting_page::setTradePriceAndArea` |  |  |
| `cd=5` | `rs=1` | 0x06 Payload `01` (`POWER_SEND_RESET`) † | Werksreset, Variante 1 | `AccouplerSettingState::createContent` | ja |  |
| `cd=5` | `rs=2` | 0x06 Payload `02` (`ACCOUPLE_FACTORY_RECOVERY`) | Werksreset, Variante 2 | `AccouplerSettingState::createContent` | ja |  |
| `cd=9` | `ot=0` | - | Aktion der Einstellungsseite (Schaltfläche nicht identifiziert) | `AccouplerSettingState::createContent` |  |  |
| `cd=10` | - | - | Abfrage (Funktionsname `getMqttCurrentTimeState`) | `venus_page_presenter::getMqttCurrentTimeState` |  |  |
| `cd=11` | `bc=<0\|1>` | - | Backup (EPS) aktivieren | `CommonCommand::setAcCouplerBackupEnable` |  |  |
| `cd=13` | - | - | Timer-Info (Funktionsname `_bleTimerInfo`) | `venus_page_presenter::_bleTimerInfo` |  |  |
| `cd=15` | `vs=<v>` | 0x15, Frame-Anfang `73 07 23 15 00` (`ACCOUPLE_SET_DISCHARGE_POWER_VERSION`) † | Leistungsversion (Entladeleistungsklasse) | `common_command::setAcCouplerPowerVersion` |  |  |
| `cd=16` | `cp=<W>` | 0x16 | Maximale Ladeleistung | `CommonCommand::setAcCouplerMaxChargePower` | ja |  |
| `cd=19` | - | 0x1A, Frame `73 08 23 1A 0A` (`ACCOUPLE_GET_CT_POWER_INFO`) † | CT-/Zählerleistung lesen | `common_command::getCtPowerInfo` |  |  |
| `cd=20` | `len=<n>,url=<URL>` | - | Modul-(FC4-)Firmwareupdate per URL | `CommonCommand::updateAcCouplerFc4Firmware` |  |  |
| `cd=22` | - | - | Upgrade der Leistungsversion melden | `CommonCommand::notifyAcCouplerUpgradePowerVersion` |  |  |
| `cd=30` | `api=<0\|1>,port=<p>` | 0x28 | Lokale API aktivieren + Port | `CommonCommand::setLocalApiEnable` | ja |  |
| `cd=40` | `number_as` | 0x40, `73 06 23 40 00 16` | Akkupack-Wiederherstellung = BMS-Codierung (Pack-Adressnummerierung); nur Stack-Typen | `CommonCommand::setBMSCodeFunction` | ja |  |
| `cd=42` | `bms_idx=255` | - | Akkupack-Details, alle Packs | `CommonCommand::getBatteryPackageDetail` |  |  |
| `cd=51` | - | - | HTTP-OTA-Status | `CommonCommand::getHttpOtaState` |  |  |
| `cd=52` | `clr=<v>,cnt=<v>` | 0x38 (Builder `httpOtaPrepare`) | HTTP-OTA vorbereiten | `common_command::mqttHttpOtaPrepare` |  |  |
| `cd=53` | `num=<v>,<per-module entries>` | - | HTTP-OTA-Info (Modul-Einträge) | `common_command::mqttHttpOtaInfo` |  |  |
| `cd=54` | `start` | 0x3A (Builder `startHttpOta`) | HTTP-OTA starten | `CommonCommand::startHttpOta` |  |  |
| `cd=56` | `dod=<v>` | - | Entladetiefe (DoD) | `CommonCommand::setBatteryDischargeDepth` |  |  |
| `cd=63` | `... (parameters not recovered)` | 0x29 (Builder `setVenusPeakShaving`) | Peak Shaving (Spitzenkappung) | `CommonCommand::setVenusPeakShaving` |  |  |
| `cd=72` | `type?` | 0x61 Payload `0B` | E-Auto-Lader-Bindung: Abfrage | `VenusEvChargerController::queryMacAndIp` | ja | NEU |
| `cd=72` | `type=1,len=<n>,ip=<IP>,mac=<MAC>` | 0x61 Payload `0A` + ip `,` mac | E-Auto-Lader-Bindung: setzen | `VenusEvChargerController::setMacAndIp` | ja | NEU |
| `cd=<n>` | (none) | - | P1-Zähler-IP setzen (cd-Literal ist eine Variable) | `CommonCommand::setP1MeterIp` |  |  |
| `cd=<n>` | `dchrg=<v>` | - | Nachlademodus (Power Replenishment) | `CommonCommand::setPowerReplenishmentMode` |  |  |
| `cd=<n>` | `len=<v> / num=<v>` | - | Aktuelle Daten anfordern | `CommonCommand::requestNowData` |  |  |
| `cd=<n>` | `meter=<v>` | - | CT-Typ | `common_command::setCtType` |  |  |
| `cd=<n>` | `seq_check` | - | Sequenzprüfung | `CommonCommand::setSeqCheck` |  |  |
| `cd=<n>` | `yy=<v>,mm=<v>,rr=<v>,hh=<v>,mn=<v>` | - | Zeit setzen | `CommonCommand::setTime` |  |  |
| `cd=<v>` | (none) | - | Geräte-IP abfragen | `CommonCommand::getDeviceIp` |  |  |

Reine BLE-Builder ohne `cd=`-Gegenstück in der Extraktion: `setVid` / `requestVid` = Cmd 0x50 / 0x51.
Die BLE-Zuordnungen mit † stammen aus den Konstantennamen (`ACCOUPLE_*`); alle Frames stehen in Anhang A.

### 4.3 Akkupack-Wiederherstellung (geprüft)

Die Schaltfläche „Battery Pack Recovery“ der Einstellungen sendet nur den BMS-Codierungsbefehl: BLE `73 06 23 40 00 16` oder MQTT `cd=40,number_as`, ohne Parameter,
ohne Folgebefehl. Jupiter-Geräte (HMM/HMN/JPLS) erhalten stattdessen `cd=20,number_as` / `73 06 23 23 00 75`. Die Firmware-Seite steht in
[../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md](../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md).

## 5. B2500 / HMA-HMJ

| cd | Parameter | App-Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=01` | - | `batter_pack_home::build`, `batter_pack_home::initState` (+41) |  |
| `cd=03` | `md=0` | `pack_charging_setting::getBody`, `pack_discharging_settingHelp::sendOrderForMonitor` |  |
| `cd=03` | `md=1` | `pack_charging_setting::getBody`, `pack_discharging_settingHelp::sendOrderForMonitor` |  |
| `cd=04` | `md=0` | `pack_discharging_setting::setDisChargingSetting` |  |
| `cd=04` | `md=1` | `pack_discharging_setting::setDisChargingSetting` |  |
| `cd=04` | `md=2` | `pack_discharging_setting::setDisChargingSetting` |  |
| `cd=04` | `md=3` | `pack_discharging_setting::setDisChargingSetting` |  |
| `cd=05` | `md=<v>` | `battery_pack_setting::getBody` |  |
| `cd=06` | `md=<v>` | `battery_pack_setting::getBody` |  |
| `cd=07` | `md=0` | `newB2500_NewModeSetting::setB2500State` |  |
| `cd=07` | `md=0,a1=<v>,b1=<HH:MM>,e1=<HH:MM>,v1=<v>,a2=...(repeated per slot)` | `pack_discharging_setting216::setTimeDischarge` |  |
| `cd=07` | `md=01` | `newB2500_NewModeSetting::setB2500State`, `newB2500_NewModeSetting::setAutoPower`, `pack_discharging_setting216::setAdativeDischarge` (+1) |  |
| `cd=07` | `md=<v>,` | `newB2500_NewModeSetting::setTimePower` |  |
| `cd=08` | `wy=<v>,yy=<v>,mm=<v>,rr=<v>,hh=<v>,mn=<v>,ss=<v>` | `batter_pack_home::setTimeZoneToDevices`, `new_Batter_Home::setTimeZoneToDevices` |  |
| `cd=09` | `wy=<v>` | `new_Batter_Home::setTimeZoneToDevices` |  |
| `cd=10` | - | `battery_pack_setting::getBody` |  |
| `cd=11` | - | `battery_pack_setting::getBody` |  |
| `cd=15` | - | `batter_pack_home::_onRefresh`, `new_Batter_Home::_onRefresh` |  |
| `cd=18` | `md=01` | `B2500CT_old_setingWidget::showCustomAlertDialog` |  |
| `cd=22` | `md=01` | `pack_discharging_setting216::build` |  |
| `cd=22` | `md=255` | `B2500CT_old_setingWidget::showCustomAlertDialog` |  |
| `cd=29` | `single_mode=0,b2500_num=2` | `new_Batter_Home::sendOrderForGroupComb` |  |
| `cd=29` | `single_mode=1,b2500_num=2` | `new_Batter_Home::sendOrderForGroupComb` |  |
| `cd=31` | `touchuan_disa=<v>` | `battery_setting_fullHelp::sendOrderForSetFullState` |  |

## 6. Jupiter (HMM / HMN / JPLS)

Die Extraktion ordnet nur einen Jupiter-spezifischen Befehl zu: BMS-Codierung `cd=20,number_as` (BLE `73 06 23 23 00 75`), siehe Abschnitt 4.3.
Weitere Jupiter-Befehle sind nicht von den gemeinsamen Buildern (Abschnitt 13) getrennt und wurden nicht zugeordnet.

## 7. V6000 / M5000 / weitere Powerstations

Befehle aus den Bereichen `ColorScreenPower`, `BigPower` und `mars_se_module`. Die `cd`-Nummern weichen hier von Venus ab (zum Beispiel `cd=37` / `cd=38` / `cd=39` Ladeplan, `cd=41` Lademodus).

| cd | Parameter | App-Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=02` | `md=<v>` | `v6000_view_model::setworkmodeSettingSE` |  |
| `cd=05` | - | `bigPowerSystemSetting_page::getBody`, `mars_se_setting_page::createContent` |  |
| `cd=20` | `md=7,va=<v>` | `outdoor_power_setting_page::createContent` |  |
| `cd=21` | `md=1,va=<v>` | `outdoor_power_setting_page::createContent`, `power_color_setting::getBody` |  |
| `cd=21` | `md=2,va=<v>` | `outdoor_power_setting_page::createContent`, `power_color_setting::getBody` |  |
| `cd=22` | `startTime=<v>,endTime=<v>,logPaths=<v>` | `mars_export_view_model::exportLogs` |  |
| `cd=33` | `y=<v>,m=<v>,d=<v>,h=<v>,min=<v>` | `outdoor_M5000_device_home::_setTime` |  |
| `cd=34` | `cxm=<v>` | `outdoor_m5000_setting_page::_showBottomSheet` |  |
| `cd=35` | - | `outdoor_m5000_setting_page::createContent`, `mars_se_setting_page::createContent` |  |
| `cd=36` | `pl=<v>` | `developer_mode_setting::_showBottomSheet` |  |
| `cd=37` | `dod=<v>` | `system_setting_dod_page::_onConfirmPressed` |  |
| `cd=37` | `md=0` | `outd_chargesetting_view_model::getChargePower`, `outdoor_power_device_home::getCurrentChargePower` |  |
| `cd=37` | `md=1,num=<v>` | `outd_chargesetting_view_model::setChargePower` |  |
| `cd=38` | `bjm=<v>` | `parallel_State_help::setDevice_Parallel_State`, `m5000_illegal_page::sendJapanMode` |  |
| `cd=38` | `type=del,id=<v>` | `outd_chargesetting_view_model::deleteTimePeriod` |  |
| `cd=38` | `type=del,id=all` | `outd_chargesetting_view_model::clearChargePlan` |  |
| `cd=38` | `type=get` | `outd_chargesetting_view_model::getChargePlan` |  |
| `cd=38` | `type=set,id=<v>,en=<v>,cycle=<v>,start=<HH:MM>\|<HH:MM>,end=<HH:MM>...` | `outd_chargesetting_view_model::addTimePeriod`, `outd_chargesetting_view_model::changeTimePeriod` |  |
| `cd=39` | `bjd=<v>` | `parallel_State_help::setDevice_Host_State`, `m5000_illegal_page::sendJapanMode` |  |
| `cd=39` | `md=0` | `outd_chargesetting_view_model::getChargePlan` |  |
| `cd=39` | `md=1,chg_mode=<v>` | `outd_chargesetting_view_model::setChargeMode` |  |
| `cd=40` | `otv=0` | `m5000_illegal_page::sendJapanMode` |  |
| `cd=41` | `eco_mode=<v>,soc_t1=<v>,soc_t2=<v>` | `v6000_charge_setting_page::body`, `base_setting_page::buildChargeModeCardWidget` |  |
| `cd=44` | `do=<v>` | `BigPowerFunctions_Settings::getBody`, `BigPowerSOC_Settings::getBody` |  |
| `cd=44` | `dod=<v>` | `system_setting_dod_page::_onConfirmPressed`, `mars_se_setting_page::toSetDOD` |  |
| `cd=45` | `cv=<v>` | `BigPowerFunctions_Settings::getBody`, `BigPowerSOC_Settings::getBody` |  |
| `cd=48` | - | `outdoor_power_device_home::checkErrorMqttCode` |  |
| `cd=49` | - | `outdoor_power_device_home::checkErrorMqttCode` |  |
| `cd=50` | `eco_en=<v>,eco_dod=500,eco_lsoc=6000,eco_hsoc=10000` | `base_setting_page::buildChargeModeCardWidget` |  |
| `cd=54` | `am=<v>,aw=<v>,ap=<v>` | `v6000_view_model::antiRevSettingmodeSe` |  |
| `cd=<v>` | - | `v6000_mqtt_tool::getMqttBase` |  |

Mit `getMqttBase(cd, params)` gebaute Befehle in den V6000-Tools (das `cd` ist ein separates Literal):

| cd | Parameter | Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=1` | - | `v6000_grid_setting_helper (closure in V6000GridSettingHelper::_send)`, `v6000_setting_page (closure in V6000SettingState::_buildV6000ChargeModeCardWidget)` (+6) |  |
| `cd=1` | `ms` | `ct_view_model (closure in V6000CTViewModel::_sendOrderForDevice)` |  |
| `cd=33` | `d=, h=, min=, s=, wy=` | `v6000_view_model::_setTime` |  |
| `cd=41` | `eco_mode=, soc_t1=, soc_t2=` | `v6000_hybrid_grid_mode_page::_sendEconomicMode`, `v6000_setting_page (closure in V6000SettingState::_buildV6000ChargeModeCardWidget)` |  |
| `cd=44` | - | `common_command::queryV6000Fc41dVersion` | NEU |
| `cd=45` | `fc41d_url=` | `common_command::updateV6000Fc41dFirmware` | NEU |
| `cd=56` | - | `v6000_grid_setting_helper (closure in V6000GridSettingHelper::getManualRules)` |  |
| `cd=62` | - | `v6000_http_ota_mqtt::queryStatus` |  |
| `cd=63` | `clr=, cnt=` | `v6000_http_ota_mqtt::prepare` |  |
| `cd=64` | `len=, url=` | `v6000_http_ota_mqtt::sendUpgradeInfo` |  |
| `cd=65` | `start` | `v6000_http_ota_mqtt::startUpgrade` |  |
| `?` | - | `ct_view_model::_sendOrderForDevice`, `v6000_grid_setting_helper::setManualRule` (+1) |  |
| `?` | `turn_s` | `ct_view_model::_sendOrderForDevice` |  |

## 8. CT (Zähler CT002 / CT003)

| cd | Parameter | App-Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=4` | `p1=0` | `dev_ct::CMD_GET_CONNECTED_DEVICES` |  |
| `cd=4` | `p1=<v>` | `common_command::getCT002DeviceInfo` |  |
| `cd=5` | `dir=<v>` | `common_command::setPhaseCurrentDirection` |  |
| `cd=5` | `p1=<v>` | `common_command::setPhaseCurrentDirection` |  |
| `cd=5` | `p1=<v>,p2=<v>,p3=<v>,p4=<v>` | `common_command::setCt003Meter` |  |
| `cd=8` | - | `common_command::hardwareResetCT002` |  |
| `cd=11` | - | `common_command::resetCT002` |  |
| `cd=14` | - | `common_command::getCTBindInverterInfo` |  |
| `cd=16` | - | `common_command::getCTBindInverterDetailInfo` |  |
| `cd=17` | - | `common_command::getCtPowerInfo` |  |
| `cd=19` | - | `common_command::getCtPowerInfo` |  |
| `cd=28` | - | `common_command::getCtPowerInfo` |  |
| `cd=42` | `ammeter_ip=<v>` | `ct_view_model::_setDevice_Sn` |  |
| `cd=43` | `turn_s=<v>` | `ct_view_model::_setDevice_ReverseEnergy` |  |
| `cd=<n>` | `ammeter_ip=<v>` | `ct_view_model::_sendOrderForDevice` |  |
| `cd=<n>` | `turn_s=<v>` | `ct_view_model::_sendOrderForDevice` |  |

## 9. HMH (inklusive Wärmepumpen-, E-Auto-Lader- und Generator-Seiten)

Die HMH-Familie sendet über `new_home_MqttTool` mit `getMqttBase(cd, params)`:

| cd | Parameter | Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=1` | - | `new_home_MqttTool::getDeviceBaseData` |  |
| `cd=2` | - | `new_home_MqttTool::getTimeInvater` |  |
| `cd=4` | - | `new_home_MqttTool::getHttpSeverIPAndPort` |  |
| `cd=5` | - | `new_home_MqttTool::getDeviceVersion` |  |
| `cd=6` | `mod=3, mod=2, mod=1` | `new_home_MqttTool::setDevieMqttinOTA` |  |
| `cd=7` | - | `new_home_MqttTool::getOTAProgress` |  |
| `cd=9` | `regs=` | `new_home_MqttTool::setParamentForDevice` |  |
| `cd=10` | `regs=<register-name list>` | `new_home_MqttTool::getDeviceBaseData` |  |
| `cd=13` | - | `new_home_MqttTool::getCTdata` |  |
| `cd=14` | - | `new_home_MqttTool::getNowBatteryInfo` |  |
| `cd=15` | - | `new_home_MqttTool::getNowData` |  |
| `cd=16` | `mod=` | `new_home_MqttTool::setSilentMqttinOTA` | NEU |
| `cd=17` | - | `new_home_MqttTool::getErrorCode` |  |
| `cd=18` | - | `new_home_MqttTool::getHeatPumpInfo` |  |
| `cd=21` | - | `new_home_MqttTool::HeatPumpStop` |  |
| `cd=24` | - | `new_home_MqttTool::getScanCTList` |  |
| `cd=25` | `mac=` | `new_home_MqttTool::newConnectCT` |  |
| `cd=26` | - | `new_home_MqttTool::getNewCTNow` |  |
| `cd=28` | `add=` | `new_home_MqttTool::setOtherCTSN` |  |
| `cd=29` | - | `new_home_MqttTool::getOtherCTConnected` |  |
| `cd=30` | `enable=, enable=, id=` | `new_home_MqttTool::setParallelAddressDevice` |  |
| `cd=31` | - | `new_home_MqttTool::CompactDownload` |  |
| `cd=32` | - | `new_home_MqttTool::checkDownloadState` |  |
| `cd=36` | `dp=` | `new_home_MqttTool::setDevicedodWork` |  |
| `cd=41` | - | `new_home_MqttTool::setTimeInvater` |  |
| `cd=42` | `x=` | `new_home_MqttTool::setGridStandard` |  |
| `cd=43` | `x=` | `new_home_MqttTool::setMaxPower` |  |
| `cd=45` | - | `new_home_MqttTool::getProviderNowData` | NEU |
| `cd=46` | - | `new_home_MqttTool::getEVChargerInfo` | NEU |
| `cd=47` | - | `new_home_MqttTool::sendEVChargerSettings` | NEU |
| `cd=49` | - | `new_home_MqttTool::setEVChargerTimer` |  |
| `cd=50` | - | `new_home_MqttTool::getEVChargerTimer` |  |
| `cd=51` | - | `new_home_MqttTool::evChargerStartOTA` | NEU |
| `cd=52` | `ip=` | `new_home_MqttTool::sendIPForEVCharger` |  |
| `cd=55` | `gn=` | `new_home_MqttTool::evChargerGetErrorCodeInfo` |  |
| `cd=57` | - | `new_home_MqttTool::getEVChargerWorkModeInfo` |  |
| `cd=58` | - | `new_home_MqttTool::setEVChargerWorkModes` |  |
| `cd=60` | - | `new_home_MqttTool::getScanShellyList` |  |
| `cd=61` | - | `new_home_MqttTool::connectShellRPC` |  |
| `cd=65` | - | `new_home_MqttTool::sendEVChargermarsSetValue` |  |
| `cd=68` | - | `new_home_MqttTool::setSystemCheck` | NEU |
| `cd=90` | - | `new_home_MqttTool::getScanShellyList` | NEU |
| `cd=91` | `type=` | `new_home_MqttTool::connectShellRPC` | NEU |
| `?` | `19, 20` | `new_home_MqttTool::HeatPumpSettingsInfo` |  |

## 10. E-Auto-Ladegerät A3600CE

Eigenes Topic `marstek_energy/A3600CE-A0/App/<encMac>/ctrl`. Diese Vorlagen sind **neu in 1.6.76**. Auf der HMH-Seite gibt es passende Builder (`cd=46`, `cd=47`, `cd=51`, Abschnitt 9).

| cd | Parameter | App-Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=12` | `op=0` | `ev_charger_connector::queryServerUrl` | NEU |
| `cd=12` | `op=1,http=<URL>` | `ev_charger_connector::setServerUrl` | NEU |
| `cd=46` | - | `ev_charger_connector::queryDeviceStatus` | NEU |
| `cd=47` | `<key>=<value>` | `ev_charger_connector::setChargeState` | NEU |
| `cd=51` | `url=<URL>` | `ev_charger_connector::triggerOtaUpgrade` | NEU |

## 11. Neue Module (Venus X / E-mini / G)

Das Modul-Framework (`modules/devices/...`) modelliert Befehle als `CMD_*`-Entity-Methoden. Seine `cd`-Nummern folgen **nicht** der älteren Venus-D-Nummerierung
(zum Beispiel 44 = DoD hier, aber 0x2C `lock` bei einem Venus D). Zeilen mit `?` sind anonyme Closures in den OTA-Entities.

| cd | Parameter | App-Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=01` | - | `module_http_ota_progress_service::_readAllTypeVersion` |  |
| `cd=02` | - | `dev_data::CMD_SET_WORK_MODE`, `dev_venus_x::CMD_SET_WORK_MODE` |  |
| `cd=03` | - | `dev_data::CMD_GET_NET_INFO`, `dev_network_entity::CMD_GET_NET_INFO`, `dev_venus_x::CMD_GET_NET_INFO` |  |
| `cd=04` | - | `dev_data::CMD_GET_ERROR_CODE`, `dev_venus_x::CMD_GET_ERROR_CODE` |  |
| `cd=18` | `ct_t=<v>,mac=<MAC>` | `dev_venus_b::CMD_SET_CT` | NEU |
| `cd=18` | `meter=<v>,mac=<MAC>` | `dev_venus_g::CMD_SET_CT`, `dev_venus_x::CMD_SET_CT` | NEU |
| `cd=19` | - | `dev_venus_x::CMD_GET_CT_POWER` |  |
| `cd=33` | - | `dev_data::CMD_SET_TIME`, `dev_venus_x::CMD_SET_TIME` |  |
| `cd=38` | - | `dev_venus_x::CMD_SET_GRID_POWER_STANDARD` |  |
| `cd=44` | - | `dev_data::CMD_SET_DOD`, `dev_venus_x::CMD_SET_DOD` | NEU |
| `cd=46` | - | `dev_data::CMD_SET_ACCESS_POWER`, `dev_venus_x::CMD_SET_ACCESS_POWER` |  |
| `cd=54` | - | `dev_data::CMD_SET_ANTL_REVERSE`, `dev_venus_x::CMD_SET_ANTL_REVERSE` |  |
| `cd=55` | - | `dev_data::CMD_SET_BLUETOOTH_STATE`, `dev_venus_b::CMD_SET_BLUETOOTH_STATE`, `dev_venus_x::CMD_SET_BLUETOOTH_STATE` |  |
| `cd=56` | - | `dev_venus_x::CMD_SET_LED_STATE` |  |
| `cd=57` | `on_grid_dsg / on_grid keys (layout not recovered)` | `dev_venus_x::CMD_SET_MAX_GRID_POWER` |  |
| `cd=58` | - | `dev_venus_x::CMD_SET_CT_PHASE_AUTO_CHECK` |  |
| `cd=59` | - | `dev_venus_b::CMD_GET_NOW_POWER`, `dev_venus_x::CMD_GET_NOW_POWER` |  |
| `cd=60` | - | `dev_venus_b::CMD_SET_SERVER`, `dev_venus_x::CMD_SET_SERVER` |  |
| `cd=61` | - | `dev_data::CMD_SET_REBOOT`, `dev_venus_x::CMD_SET_REBOOT` |  |
| `cd=62` | - | `ota_type_entity::?`, `dev_venus_g::CMD_GET_BATTERY_PACK_INFO`, `ota_model::getUpgradeStatus` (+1) |  |
| `cd=62` | `shelly_pro=<v>` | `dev_venus_x::CMD_SET_SHELLY_PRO_3EM_PORT` |  |
| `cd=63` | - | `ota_type_entity::?` |  |
| `cd=63` | `clr=<v>,cnt=<v>` | `ota_model::setUpgradePrepareHttp` |  |
| `cd=63` | `ct_chg_type=<v>` | `dev_venus_b::CMD_SET_CT_CHARGE_TYPE` |  |
| `cd=64` | `num=<v>` | `ota_type_entity::CMD_SET_UPGRADE_INFO_HTTP` |  |
| `cd=64` | `ups_p=<v>` | `dev_data::CMD_SET_UPS_CHARGE_POWER`, `dev_venus_b::CMD_SET_UPS_CHARGE_POWER`, `dev_venus_x::CMD_SET_UPS_CHARGE_POWER` | NEU |
| `cd=65` | - | `ota_type_entity::?` |  |
| `cd=65` | `lapi_en=<v>,self_port=<v>,remote_port=<v>` | `dev_data::CMD_SET_LOCAL_API`, `dev_venus_x::CMD_SET_LOCAL_API` | NEU |
| `cd=65` | `start` | `ota_model::startUpgradeHttp` |  |
| `cd=66` | - | `vg_ota_type_entity::?`, `ota_model::getUpgradeStatus` |  |
| `cd=67` | - | `dev_data::CMD_GET_SCAN_WIFI_LIST`, `dev_venus_x::CMD_GET_SCAN_WIFI_LIST` | NEU |
| `cd=<v>` | - | `dev_data::CMD_SET_MODE_MANUAL_INFO` |  |
| `cd=A0` | - | `ota_model::getUpgradeStatus`, `vb_ota_type_entity::?` |  |
| `cd=A1` | - | `vb_ota_type_entity::?` |  |
| `cd=A1` | `cnt=<v>` | `ota_model::setUpgradePrepareHttp` |  |
| `cd=A2` | - | `vb_ota_type_entity::CMD_SET_UPGRADE_INFO_HTTP` |  |
| `cd=A2` | `num=<v>` | `vb_ota_type_entity::CMD_SET_UPGRADE_INFO_HTTP` |  |
| `cd=A3` | - | `vb_ota_type_entity::?` |  |
| `cd=A3` | `start` | `ota_model::startUpgradeHttp` |  |

## 12. Sun- / Inverter-Monitor-Gruppe

Die Produktzuordnung dieser Gruppe wurde nicht verifiziert.

| cd | Parameter | App-Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=1` | - | `All_Little_Sun_Home::createBody`, `All_Little_Sun_Home::initState`, `All_Little_Sun_Home::_onRefresh` (+1) |  |
| `cd=2` | `md=1` | `All_little_Sun_NewSetTimeMode::setAutoPower` |  |
| `cd=3` | `md=2,nm=<slot>,bt=<HH:MM>,et=<HH:MM>,...` | `All_little_Sun_NewSetTimeMode::setTimePower` |  |
| `cd=5` | - | `All_Little_Sun_Setting::getBody` |  |
| `cd=8` | `p1=<v>` | `Inverter_power_limit::sendOrderForSetFullState` |  |
| `cd=10` | - | `FC4Helper::_sendQueryFC4ByMqtt_`, `FC4Helper::FC4Helper` |  |
| `cd=11` | - | `All_Little_Sun_Setting::clickFC4Upgrade` |  |
| `cd=11` | `p1=0` | `Inverter_monitor_ModeSettingHelp::sendOrderForMonitor` |  |
| `cd=11` | `p1=1` | `Inverter_monitor_ModeSettingHelp::sendOrderForMonitor` |  |
| `cd=11` | `p1=2` | `Inverter_monitor_ModeSettingHelp::sendOrderForMonitor` |  |
| `cd=11` | `url=<URL>` | `common_command::updateSunFc4ForUrl` |  |
| `cd=12` | - | `All_Little_Sun_Home::initState`, `Inverter_Monitor_Home::initState` |  |
| `cd=13` | `full_d=0` | `SunHelper::_sendFullByMqtt_`, `SunHelper::SunHelper` |  |
| `cd=13` | `full_d=1` | `SunHelper::_sendFullByMqtt_`, `SunHelper::SunHelper` |  |
| `cd=14` | - | `All_Little_Sun_Home::initState` |  |
| `cd=20` | `le=<n>,url=<URL>` | `Inverter_Monitor_Setting::sendFc4order` |  |
| `cd=21` | - | `Inverter_Monitor_Home::getMqttCurrentTimeState`, `Inverter_Monitor_Setting::initState` |  |
| `cd=22` | `p1=<v>` | `Inverter_set_tool::sendGridNoAllowed_order` |  |
| `cd=23` | `id=<v>,s=<v>,c=<v>,i=<v>,p=<v>` | `newSetinverter_page::setSelect`, `addDevice_setInverter_widget::setSelect` |  |

## 13. Gemeinsame und nicht zugeordnete Befehle

| cd | Parameter | App-Funktion(en) | Neu in 1.6.76 |
|---|---|---|---|
| `cd=02` | `md=1,tm=<1\|2\|3>,ab=<0\|1>,cb=<v>,ce=<v>,db=<v>,de=<v>` | `system_setting_workmodeset::sentModeTime` |  |
| `cd=02` | `md=<v>` | `system_setting_workmodeset::getAppBar` |  |
| `cd=05` | - | `my_item2::getBody`, `my_item::getBody` |  |
| `cd=06` | `do=<v>` | `my_item::getBody` |  |
| `cd=09` | `wy=<v>` | `setTimeZone_page::setTimeZoneToDevices` |  |
| `cd=12` | `ht=<v>` | `utils::syncServiceToDevice` |  |
| `cd=20` | `md=1,no=<v>` | `common_command::toggleAcOutput` |  |
| `cd=20` | `md=2,no=<v>` | `common_command::toggleDcOutput` |  |
| `cd=20` | `md=3,no=<v>` | `common_command::toggleLedLight` | NEU |
| `cd=29` | `single_mode=1,b2500_num=0` | `add_Group_selectDevice::sendCombModelForDeviceInfo` |  |
| `cd=32` | `es=<v>` | `system_setting_function::getBody` |  |
| `cd=35` | `ar=<v>` | `system_setting_function::getBody` |  |
| `cd=37 / cd=44` | `dod=<v> / do=<v>` | `system_setting_function::getBody` |  |
| `cd=45` | `cv=<v>` | `system_setting_function::getBody` |  |
| `cd=48` | - | `common_command::getErrorCodefromDevice` |  |
| `cd=51` | - | `device_home::createBody`, `device_home::didChangeDependencies`, `device_home::_onRefresh` |  |
| `cd=pull_log` | `taskId=<v>` | `remote_log_pull_helper::_publishResult` |  |

`getMqttBase` selbst baut `cd=<n>,<params>` / `cd=<v>`. `pull_log` ist der Remote-Log-Pull-Helper.

## 14. Neu in 1.6.76 gegenüber 1.6.72

| String | Produkt / Funktion | Bedeutung |
|---|---|---|
| `cd=12,op=0` / `cd=12,op=1,http=<URL>` | EV charger: `EvChargerConnector.queryServerUrl/setServerUrl` | Server-URL lesen/setzen |
| `cd=46`, `cd=47,<key>=<value>`, `cd=51,url=<URL>` | EV charger: `queryDeviceStatus/setChargeState/triggerOtaUpgrade` (HMH: `getEVChargerInfo` 46, `sendEVChargerSettings` 47, `evChargerStartOTA` 51) | Status, Laderegelung, OTA |
| `cd=72,...` | Venus: `VenusEvChargerController` | E-Auto-Lader-IP/MAC-Bindung |
| `cd=67` | New modules: `CMD_GET_SCAN_WIFI_LIST` | WLAN-Scanliste |
| `cd=44` / `cd=45` (bare) | V6000: `queryV6000Fc41dVersion` / `updateV6000Fc41dFirmware` (`cd=45,fc41d_url=<URL>`); HMH: `getProviderNowData` = 45; new modules: `CMD_SET_DOD` = 44 | FC41D-Version / OTA; Provider-Daten; DoD |
| `cd=90` / `cd=91,type=<t>` | HMH: `getScanShellyList` / `connectShellRPC` | Shelly-Scan / Verbindung (neue Varianten von 60/61) |
| `cd=68` | HMH: `setSystemCheck` | Systemprüfung |
| `cd=16` | CT: `getCTBindInverterDetailInfoByMqtt`; HMH: `setSilentMqttinOTA` (`cd=16,mod=`) | CT-Bindungsdetail / stiller OTA-Modus |
| `cd=18,ct_t=<v>,mac=<MAC>` / `cd=18,meter=<v>` | Venus E-mini: `CMD_SET_CT`; `CommonCommand::setCtType` | CT-Typ |
| `cd=20,md=3,no=<v>` | `CommonCommand.toggleLedLight` | LED umschalten (neben md=1 AC- / md=2 DC-Ausgang) |
| `cd=30,data=<v>` | Parsed in `new_home_MqttTool.handMqttMessage` (response side) | Antwortfeld, keine Sendevorlage |
| `cd=64,ups_p=<v>` / `cd=65,lapi_en=<v>,self_port=<v>,remote_port=<v>` | New modules (dev_data, venus_x, venus_e_mini): `CMD_SET_UPS_CHARGE_POWER` / `CMD_SET_LOCAL_API` | UPS-Ladeleistung / lokale API |

## Anhang A: Statische HM-BLE-Frames

Automatisch extrahierte Konstanten aus den BLE-Hilfsklassen (`Blecommon*`). Gezeigt werden nur die ersten 5 Bytes (`73 LEN 23 CMD P0`); Byte 4 kann bei dynamisch
gebauten Listen fehlen oder falsch sein, CMD ist verlässlich. Dynamische Builder in `common_command` sind in dieser Tabelle nicht vollständig:
`setBMSCodeFunction` 0x40, `setLocalApiEnable` 0x28, `setAcCouplerMaxChargePower` 0x16, `setVenusPeakShaving` 0x29, HTTP-OTA vorbereiten 0x38, HTTP-OTA starten 0x3A, `setVid` / `requestVid` 0x50 / 0x51.
Der Namenspräfix-Hinweis ist eine Vermutung aus dem Konstantennamen, keine verifizierte Produktzuordnung.

| CMD | Konstanten (erste 5 Bytes `73 LEN 23 CMD P0`) | Hinweis aus Namenspräfix |
|---|---|---|
| `0x01` | `POWER_SEND_OPEN_OTA_BT_M2200N` `73 06 23 01 01` [M2200N] | - |
| `0x02` | `MARSISE_WORK_MODE_AI` `73 06 23 02 03`<br>`MARSISE_WORK_MODE_AUTO` `73 06 23 02 00`<br>`MARSISE_WORK_MODE_MANUAL` `73 06 23 02 02`<br>`POWER_SEND_Mode_Battery` `73 06 23 02 02`<br>`POWER_SEND_Mode_Pink2` `73 0F 23 02 01`<br>`POWER_SEND_Mode_SELFUSE` `73 06 23 02 00`<br>`POWER_SEND_SET_SERVICE` `73 06 23 02 00` | power station |
| `0x03` | `POWER_SEND_GET_DEVICEINFO` `73 06 23 03 01` | - |
| `0x04` | `POWER_SEND_GET_TYPEMACDEVICEID` `73 06 23 04 01` | - |
| `0x06` | `ACCOUPLE_FACTORY_RECOVERY` `73 06 23 06 02`<br>`POWER_SEND_RESET` `73 06 23 06 01` | Venus (AC coupler) |
| `0x07` | `POWER_SEND_DeepDisCharge` `73 06 23 07 01` | - |
| `0x08` | `POWER_SEND_READWIFINAME` `73 06 23 08 01` | - |
| `0x09` | `ACCOUPLE_SEND_MANUAL_SET_POWER` `73 0F 23 09 01`<br>`ACCOUPLE_SEND_TRADE_PRICE` `73 10 23 09 02`<br>`ACCOUPLE_WORK_MODE_AUTO` `73 06 23 09 00`<br>`POWER_SEND_GET_DEVICEINFO2` `73 06 23 09 01`<br>`POWER_SEND_HANDEMODE_SET_POWER` `73 06 23 09 02`<br>`POWER_SEND_MANUAL_SET_POWER` `73 0F 23 09 02` | Venus (AC coupler) |
| `0x0A` | `POWER_SEND_GET_LittleSun` `73 06 23 0A 01` | Sun |
| `0x0B` | `POWER_SEND_DeepB2500` `73 06 23 0B 01`<br>`POWER_SEND_Standby_Timeout` `73 06 23 0B 00` | - |
| `0x0C` | `POWER_SEND_Set_Frequency` `73 06 23 0C 00`<br>`SEND_SetPower_Max_CMD` `73 07 23 0C 00` | - |
| `0x0D` | `CHARGING_SEND_SET_Full_SET` `73 06 23 0D 01`<br>`CHARGING_SEND_SET_HALF_Charge` `73 06 23 0D 00`<br>`COLOR_SEND_SET_ACTIME` `73 06 23 0D 00` | power station |
| `0x0E` | `COLOR_SEND_SET_DCTIME` `73 06 23 0E 00`<br>`DISCHARGING_SEND_SET_00` `73 06 23 0E 00`<br>`DISCHARGING_SEND_SET_01` `73 06 23 0E 01`<br>`DISCHARGING_SEND_SET_10` `73 06 23 0E 02`<br>`DISCHARGING_SEND_SET_11` `73 06 23 0E 03` | power station |
| `0x0F` | `POWER_SEND_GET_DianXinInfo` `73 06 23 0F 01` | - |
| `0x10` | `CT002_ZERO_POINT_CALIBRATION` `73 06 23 10 01`<br>`FC4_START_SELFOTA` `73 09 23 10 0A`<br>`POWER_SEND_GET_FC4_LittleSun` `73 08 23 10 0A` | CT002, Sun |
| `0x11` | `COLOR_SEND_SET_EN_MODE_CHARGE` `73 08 23 11 00`<br>`FC4_CHECK_SELFOTA` `73 08 23 11 0A`<br>`POWER_SEND_SET_ADAPTIVE_MODE` `73 06 23 11 01` | power station, Sun |
| `0x12` | `FC4_LITTLE_SUN_SELFOTA` `73 08 23 12 0A`<br>`POWER_SEND_SET_TIME_MODE` `73 28 23 12 00` | Sun |
| `0x13` | `ERROR_CODE_GET_LittleSun` `73 08 23 13 0A`<br>`POWER_SEND_GET_DisCharageInfo` `73 06 23 13 01` | Sun |
| `0x14` | `CT002_VOLTAGE_FREQUENCY_DETECTION` `73 06 23 14 01`<br>`POWER_SEND_Time` `73 0D 23 14 00`<br>`SUBDEV_RUNINFO_GET_LittleSun` `73 08 23 14 0A`<br>`V6000_FC41D_CHECK_VERSION` `73 05 23 14 41` | CT002, Sun, power station |
| `0x15` | `ACCOUPLE_SET_DISCHARGE_POWER_VERSION` `73 07 23 15 00`<br>`CT002_VOLTAGE_PHASE_ANGLE_DETECTION` `73 06 23 15 01`<br>`V6000_FC41D_START_OTA` `73 06 23 15 AA` | Venus (AC coupler), CT002, power station |
| `0x16` | `CT002_ACTIVE_POWER_DETECTION` `73 06 23 16 01` | CT002 |
| `0x17` | `CT002_CURRENT_MEASUREMENT_DIRECTION_REVERSE` `73 06 23 17 00`<br>`LITTLE_SUN_GET_CT_POWER_INFO` `73 08 23 17 0A` | CT002, Sun |
| `0x1A` | `ACCOUPLE_GET_CT_POWER_INFO` `73 08 23 1A 0A` | Venus (AC coupler) |
| `0x22` | `POWER_SEND_SET_FULL_CLOSE_LittleSun` `73 06 23 22 00`<br>`POWER_SEND_SET_FULL_OPEN_LittleSun` `73 06 23 22 01` | Sun |
| `0x25` | `POWER_SEND_ReStartDevice` `73 06 23 25 01` | - |
| `0x26` | `SEND_SetPower_reSetting` `73 06 23 26 01` | - |
| `0x2A` | `POWER_SEND_RE_DIAGNOSE` `73 06 23 2A FF`<br>`POWER_SEND_RE_DIAGNOSE_OLD` `73 06 23 2A 01` | - |
| `0x2B` | `REQUIRE_SWITCH_BATTERY_STATE` `73 06 23 2B 01` | - |
| `0x30` | `POWER_SEND_GET_ErrorInfo` `73 06 23 30 01` | - |
| `0x31` | `POWER_SEND_GET_BMS_ErrorInfo` `73 06 23 31 01` | - |
| `0x32` | `COLOR_SEND_SET_V3000_ECO_Close` `73 0E 23 32 55`<br>`COLOR_SEND_SET_V3000_ECO_Open` `73 0E 23 32 55`<br>`SEND_SetEPS_CMD` `73 06 23 32 00` | power station |
| `0x33` | `POWER_SEND_GET_CTPowerInfo` `73 06 23 33 01`<br>`POWER_SEND_GET_V3000_SN` `73 06 23 33 00` | - |
| `0x35` | `SEND_SetAntl_Reverse_CMD` `73 06 23 35 00` | - |
| `0x44` | `SEND_SetDisChargeDeep_CMD` `73 06 23 44 00` | - |
| `0x45` | `SEND_SetCharge_Current_CMD` `73 06 23 45 00` | - |
| `0xB9` | `POWER_SEND_MODE_SET_AntiReverse` `73 06 23 B9 02`<br>`POWER_SEND_MODE_SET_B2500` `73 06 23 B9 01`<br>`POWER_SEND_MODE_SET_DEFAULT` `73 06 23 B9 00` | - |
| `0xBA` | `ERROR_CODE_GET_Inverter` `73 06 23 BA 01` | - |

## Anhang B: Grenzen dieses Vokabulars

* Nur statische Analyse; es wurde kein Frame mitgeschnitten und kein Gerät angesprochen.
* Zeilen ohne „Von Hand geprüft“ stammen aus automatischer Literal-Extraktion. Builder, die die `cd`-Nummer oder Parameter erst zur Laufzeit zusammensetzen (`cd=<n>`), sind unvollständig.
* Die Bedeutung nicht von Hand geprüfter Zeilen stammt nur aus dem Funktionsnamen.
* BLE-Konstanten sind keinen Produkten zugeordnet; es gibt nur die in Abschnitt 4.2 markierten Zuordnungen.
