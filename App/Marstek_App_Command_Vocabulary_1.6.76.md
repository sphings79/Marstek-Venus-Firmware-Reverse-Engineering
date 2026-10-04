# Marstek App 1.6.76: Command Vocabulary

> 🇩🇪 Deutsche Version: [Marstek_App_Befehlsvokabular_1.6.76_de.md](Marstek_App_Befehlsvokabular_1.6.76_de.md)

Status 2026-10-02. Static analysis of the arm64 `libapp.so` (Dart 3.8.1) of the Marstek app (package `com.hamedata.marstek`) version 1.6.76,
disassembled with Blutter and compared with the strings of 1.6.72. No device, no cloud login, nothing was triggered. The vocabulary lists what the
app can **send** (MQTT `cd=` payloads and HM BLE frames). Which of it the Venus D firmware (Control 147-151) handles is in
[Marstek_App_vs_Firmware_Command_Map.md](Marstek_App_vs_Firmware_Command_Map.md).

## 1. How to read this document

* The `cd` numbering is **per product**. The same number means different things for different products. The source location in the app (column "App function(s)") tells which product a template belongs to.
* `<v>` = runtime value, `<n>` = the `cd` number is a variable and not a literal in that function, `<HH:MM>` = zero-padded time. `...` = more parameters that the extraction did not recover.
* "Hand-verified" = traced through the disassembly by hand. All other rows are **automatically extracted** send templates; their meaning is taken from the app function name only.
* "New in 1.6.76" (NEW) = string that does not occur in 1.6.72.
* Only the send side is covered. Parser, response and mock functions were filtered out.
* † in a BLE column = the pairing with the `cd=` command is by constant name only (not traced).

## 2. Transport formats

**MQTT.** Topic `hame_energy/<devType>/App/<devId>/ctrl`, payload ASCII `cd=<n>,k=v,k=v,...`, for example `cd=40,number_as` or `cd=16,cp=<W>`.
The EV charger A3600CE uses its own topic: `marstek_energy/A3600CE-A0/App/<encMac>/ctrl`.

**BLE (HM frame).** `73 | LEN | 23 | CMD | payload... | XOR`.

* `73` magic, `23` fixed, `LEN` = total frame length including the XOR byte, `CMD` = command id, `XOR` = XOR over all preceding bytes (`calCrcValueListWithMutableList`).
* Example: `73 06 23 40 00 16` = cmd 0x40, payload `00`, XOR 0x16 (0x73 ^ 0x06 ^ 0x23 ^ 0x40 ^ 0x00).

**Transport choice** (`CommonCommand._doOperate`, verified): if the current device type is empty nothing is sent; with a BLE link the BLE closure runs;
otherwise, if the cloud connection is online, the MQTT closure publishes. Commands wrapped in `retryOnTimeoutWithNoTag` are sent up to 3 times with a 5 s timeout.

## 3. Product families and source areas

| Family | App source areas | Devices / note |
|---|---|---|
| B2500 / HMA-HMJ | `BatterB2500/`, `newBatterB2500/` | B2500 storage family |
| Venus | `Ac_Coupler/`, Venus functions in `common/common_command` | VNSD, VNSA, VNSE3, VNSE4, EMAX, VEPRO, VEPAC (+ HMG*) |
| Jupiter | no own source area in the extraction | HMM, HMN, JPLS (only the BMS-coding variant is attributed) |
| V6000 / M5000 / other power stations | `ColorScreenPower/`, `Home/BigPower/`, `Home/mars_se_module/`, `Widget/base_setting_page` | V6000, M5000, BigPower, Mars SE (module names in the app) |
| CT | `ct_view_model`, `modules/devices/entity/ct/`, CT functions in `common/common_command` | CT002 / CT003 meters |
| HMH (incl. heat pump, EV, generator) | `Home/new_home/view/tool/new_home_MqttTool` | HMH family |
| EV charger A3600CE | `Ac_Coupler/evCharger/connectors/ev_charger_connector` | own topic `marstek_energy/A3600CE-A0/App/<encMac>/ctrl` |
| New modules (Venus X / E-mini / G) | `modules/devices/...` | module framework with `CMD_*` entity methods |
| Sun / inverter monitor | `All_Big_litle_Sun/`, `InverterMonitor/` | product identity not verified |
| Shared / unattributed | generic pages, `utils`, parts of `common/common_command` | no product attribution possible |

## 4. Venus (VNSD / VNSA / VNSE3 / VNSE4 / EMAX / VEPRO / VEPAC)

### 4.1 Device-type gating (verified)

* `DeviceInfo.isVNSD` is true for the Venus types VNSD, VNSA, VNSACH, VEPRO, VEPROCH, VNSDCH, VNSD2, VEPRO2, VNSA2.
* `DeviceInfo.isStackDev` = `isVNSD` or VEPAC. The "Battery Pack Recovery" row of the settings page is built only for stack devices (not for Venus E/C, E3, E4, EMAX, VDAC). No firmware-version gate was found.
* `DeviceInfo.isVenus` = HMG1 / HMG25 / HMG50, VNSE3*, the `isVNSD` family, VDAC, VNSE4, VNSEMAX, VEPAC.
* Venus E-mini / X / G devices use the separate module framework (section 11) with a different `cd` numbering.

### 4.2 Commands

Rows marked "yes" in the "Hand-verified" column are section A of the extraction (verified by hand), the rest is extracted.

| cd | Parameters | BLE command | Meaning | App function(s) | Hand-verified | New in 1.6.76 |
|---|---|---|---|---|---|---|
| `cd=1` | - | - | Request status / device data (work-mode page init, OTA service device info) | `ac_workMode_setting::initState`, `venus_http_ota_progress_service::_requestDeviceInfo` |  |  |
| `cd=2` | `md=0` | 0x09 payload `00` (`ACCOUPLE_WORK_MODE_AUTO`) † | Auto mode | `CommonCommand::setAcAutoMode`, `setAutoPower` | yes |  |
| `cd=2` | `md=1` | - | Manual mode | `CommonCommand::setAcManualMode` | yes |  |
| `cd=2` | `md=5,nl=1` | - | Economy mode | `CommonCommand::setAcEconomyMode` | yes |  |
| `cd=3` | `md=1,nm=<slot>,bt=<HH:MM>,et=<HH:MM>,...` | 0x09 payload `01`, frame head `73 0F 23 09 01` (`ACCOUPLE_SEND_MANUAL_SET_POWER`) † | Manual time slot (power schedule entry) | `venus_helper::setTimePower`, `power_setting_page::setTimePower` |  |  |
| `cd=3` | `md=2,id=<v>,in=<v>,on=<v>` | 0x09 payload `02`, frame head `73 10 23 09 02` (`ACCOUPLE_SEND_TRADE_PRICE`) † | Trade price and area | `trade_setting_page::setTradePriceAndArea` |  |  |
| `cd=5` | `rs=1` | 0x06 payload `01` (`POWER_SEND_RESET`) † | Factory reset, variant 1 | `AccouplerSettingState::createContent` | yes |  |
| `cd=5` | `rs=2` | 0x06 payload `02` (`ACCOUPLE_FACTORY_RECOVERY`) | Factory reset, variant 2 | `AccouplerSettingState::createContent` | yes |  |
| `cd=9` | `ot=0` | - | Settings-page action (button not identified) | `AccouplerSettingState::createContent` |  |  |
| `cd=10` | - | - | Query (function name `getMqttCurrentTimeState`) | `venus_page_presenter::getMqttCurrentTimeState` |  |  |
| `cd=11` | `bc=<0\|1>` | - | Backup (EPS) enable | `CommonCommand::setAcCouplerBackupEnable` |  |  |
| `cd=13` | - | - | Timer info (function name `_bleTimerInfo`) | `venus_page_presenter::_bleTimerInfo` |  |  |
| `cd=15` | `vs=<v>` | 0x15, frame head `73 07 23 15 00` (`ACCOUPLE_SET_DISCHARGE_POWER_VERSION`) † | Power version (discharge power class) | `common_command::setAcCouplerPowerVersion` |  |  |
| `cd=16` | `cp=<W>` | 0x16 | Max charge power | `CommonCommand::setAcCouplerMaxChargePower` | yes |  |
| `cd=19` | - | 0x1A, frame `73 08 23 1A 0A` (`ACCOUPLE_GET_CT_POWER_INFO`) † | Read CT / meter power | `common_command::getCtPowerInfo` |  |  |
| `cd=20` | `len=<n>,url=<URL>` | - | Module (FC4) firmware update by URL | `CommonCommand::updateAcCouplerFc4Firmware` |  |  |
| `cd=22` | - | - | Notify power-version upgrade | `CommonCommand::notifyAcCouplerUpgradePowerVersion` |  |  |
| `cd=30` | `api=<0\|1>,port=<p>` | 0x28 | Local API enable + port | `CommonCommand::setLocalApiEnable` | yes |  |
| `cd=40` | `number_as` | 0x40, `73 06 23 40 00 16` | Battery Pack Recovery = BMS coding (pack address numbering); only for stack types | `CommonCommand::setBMSCodeFunction` | yes |  |
| `cd=42` | `bms_idx=255` | - | Battery pack detail, all packs | `CommonCommand::getBatteryPackageDetail` |  |  |
| `cd=51` | - | - | HTTP OTA state | `CommonCommand::getHttpOtaState` |  |  |
| `cd=52` | `clr=<v>,cnt=<v>` | 0x38 (builder `httpOtaPrepare`) | HTTP OTA prepare | `common_command::mqttHttpOtaPrepare` |  |  |
| `cd=53` | `num=<v>,<per-module entries>` | - | HTTP OTA info (module entries) | `common_command::mqttHttpOtaInfo` |  |  |
| `cd=54` | `start` | 0x3A (builder `startHttpOta`) | Start HTTP OTA | `CommonCommand::startHttpOta` |  |  |
| `cd=56` | `dod=<v>` | - | Discharge depth (DoD) | `CommonCommand::setBatteryDischargeDepth` |  |  |
| `cd=63` | `... (parameters not recovered)` | 0x29 (builder `setVenusPeakShaving`) | Peak shaving | `CommonCommand::setVenusPeakShaving` |  |  |
| `cd=72` | `type?` | 0x61 payload `0B` | EV charger binding: query | `VenusEvChargerController::queryMacAndIp` | yes | NEW |
| `cd=72` | `type=1,len=<n>,ip=<IP>,mac=<MAC>` | 0x61 payload `0A` + ip `,` mac | EV charger binding: set | `VenusEvChargerController::setMacAndIp` | yes | NEW |
| `cd=<n>` | (none) | - | Set P1 meter IP (cd literal is a variable) | `CommonCommand::setP1MeterIp` |  |  |
| `cd=<n>` | `dchrg=<v>` | - | Power replenishment mode | `CommonCommand::setPowerReplenishmentMode` |  |  |
| `cd=<n>` | `len=<v> / num=<v>` | - | Request now-data | `CommonCommand::requestNowData` |  |  |
| `cd=<n>` | `meter=<v>` | - | CT type | `common_command::setCtType` |  |  |
| `cd=<n>` | `seq_check` | - | Sequence check | `CommonCommand::setSeqCheck` |  |  |
| `cd=<n>` | `yy=<v>,mm=<v>,rr=<v>,hh=<v>,mn=<v>` | - | Set time | `CommonCommand::setTime` |  |  |
| `cd=<v>` | (none) | - | Device IP query | `CommonCommand::getDeviceIp` |  |  |

BLE-only builders without `cd=` twin in the extraction: `setVid` / `requestVid` = cmd 0x50 / 0x51.
The BLE pairings with † come from the constant names (`ACCOUPLE_*`); see appendix A for all frames.

### 4.3 Battery Pack Recovery (verified)

The settings button "Battery Pack Recovery" sends only the BMS-coding command: BLE `73 06 23 40 00 16` or MQTT `cd=40,number_as`, no parameters,
no follow-up command. Jupiter devices (HMM/HMN/JPLS) get `cd=20,number_as` / `73 06 23 23 00 75` instead. The firmware side is described in
[../BMS/VNSD_Battery_Pack_Recovery_Firmware.md](../BMS/VNSD_Battery_Pack_Recovery_Firmware.md).

## 5. B2500 / HMA-HMJ

| cd | Parameters | App function(s) | New in 1.6.76 |
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

The extraction attributes only one Jupiter-specific command: BMS coding `cd=20,number_as` (BLE `73 06 23 23 00 75`), see section 4.3.
Other Jupiter commands are not separated from the shared builders (section 13) and were not attributed.

## 7. V6000 / M5000 / other power stations

Commands from the `ColorScreenPower`, `BigPower` and `mars_se_module` areas. The `cd` numbers here differ from Venus (for example `cd=37` / `cd=38` / `cd=39` charge plan, `cd=41` charge mode).

| cd | Parameters | App function(s) | New in 1.6.76 |
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

Commands built with `getMqttBase(cd, params)` in the V6000 tools (the `cd` is a separate literal):

| cd | Parameters | Function(s) | New in 1.6.76 |
|---|---|---|---|
| `cd=1` | - | `v6000_grid_setting_helper (closure in V6000GridSettingHelper::_send)`, `v6000_setting_page (closure in V6000SettingState::_buildV6000ChargeModeCardWidget)` (+6) |  |
| `cd=1` | `ms` | `ct_view_model (closure in V6000CTViewModel::_sendOrderForDevice)` |  |
| `cd=33` | `d=, h=, min=, s=, wy=` | `v6000_view_model::_setTime` |  |
| `cd=41` | `eco_mode=, soc_t1=, soc_t2=` | `v6000_hybrid_grid_mode_page::_sendEconomicMode`, `v6000_setting_page (closure in V6000SettingState::_buildV6000ChargeModeCardWidget)` |  |
| `cd=44` | - | `common_command::queryV6000Fc41dVersion` | NEW |
| `cd=45` | `fc41d_url=` | `common_command::updateV6000Fc41dFirmware` | NEW |
| `cd=56` | - | `v6000_grid_setting_helper (closure in V6000GridSettingHelper::getManualRules)` |  |
| `cd=62` | - | `v6000_http_ota_mqtt::queryStatus` |  |
| `cd=63` | `clr=, cnt=` | `v6000_http_ota_mqtt::prepare` |  |
| `cd=64` | `len=, url=` | `v6000_http_ota_mqtt::sendUpgradeInfo` |  |
| `cd=65` | `start` | `v6000_http_ota_mqtt::startUpgrade` |  |
| `?` | - | `ct_view_model::_sendOrderForDevice`, `v6000_grid_setting_helper::setManualRule` (+1) |  |
| `?` | `turn_s` | `ct_view_model::_sendOrderForDevice` |  |

## 8. CT (CT002 / CT003 meters)

| cd | Parameters | App function(s) | New in 1.6.76 |
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

## 9. HMH (including heat pump, EV charger and generator pages)

The HMH family sends through `new_home_MqttTool` with `getMqttBase(cd, params)`:

| cd | Parameters | Function(s) | New in 1.6.76 |
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
| `cd=16` | `mod=` | `new_home_MqttTool::setSilentMqttinOTA` | NEW |
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
| `cd=45` | - | `new_home_MqttTool::getProviderNowData` | NEW |
| `cd=46` | - | `new_home_MqttTool::getEVChargerInfo` | NEW |
| `cd=47` | - | `new_home_MqttTool::sendEVChargerSettings` | NEW |
| `cd=49` | - | `new_home_MqttTool::setEVChargerTimer` |  |
| `cd=50` | - | `new_home_MqttTool::getEVChargerTimer` |  |
| `cd=51` | - | `new_home_MqttTool::evChargerStartOTA` | NEW |
| `cd=52` | `ip=` | `new_home_MqttTool::sendIPForEVCharger` |  |
| `cd=55` | `gn=` | `new_home_MqttTool::evChargerGetErrorCodeInfo` |  |
| `cd=57` | - | `new_home_MqttTool::getEVChargerWorkModeInfo` |  |
| `cd=58` | - | `new_home_MqttTool::setEVChargerWorkModes` |  |
| `cd=60` | - | `new_home_MqttTool::getScanShellyList` |  |
| `cd=61` | - | `new_home_MqttTool::connectShellRPC` |  |
| `cd=65` | - | `new_home_MqttTool::sendEVChargermarsSetValue` |  |
| `cd=68` | - | `new_home_MqttTool::setSystemCheck` | NEW |
| `cd=90` | - | `new_home_MqttTool::getScanShellyList` | NEW |
| `cd=91` | `type=` | `new_home_MqttTool::connectShellRPC` | NEW |
| `?` | `19, 20` | `new_home_MqttTool::HeatPumpSettingsInfo` |  |

## 10. EV charger A3600CE

Own topic `marstek_energy/A3600CE-A0/App/<encMac>/ctrl`. These templates are **new in 1.6.76**. The HMH side has matching builders (`cd=46`, `cd=47`, `cd=51`, section 9).

| cd | Parameters | App function(s) | New in 1.6.76 |
|---|---|---|---|
| `cd=12` | `op=0` | `ev_charger_connector::queryServerUrl` | NEW |
| `cd=12` | `op=1,http=<URL>` | `ev_charger_connector::setServerUrl` | NEW |
| `cd=46` | - | `ev_charger_connector::queryDeviceStatus` | NEW |
| `cd=47` | `<key>=<value>` | `ev_charger_connector::setChargeState` | NEW |
| `cd=51` | `url=<URL>` | `ev_charger_connector::triggerOtaUpgrade` | NEW |

## 11. New modules (Venus X / E-mini / G)

The module framework (`modules/devices/...`) models commands as `CMD_*` entity methods. Its `cd` numbers do **not** follow the older Venus D numbering
(for example 44 = DoD here, but 0x2C `lock` on a Venus D). Rows ending in `?` are anonymous closures inside the OTA entities.

| cd | Parameters | App function(s) | New in 1.6.76 |
|---|---|---|---|
| `cd=01` | - | `module_http_ota_progress_service::_readAllTypeVersion` |  |
| `cd=02` | - | `dev_data::CMD_SET_WORK_MODE`, `dev_venus_x::CMD_SET_WORK_MODE` |  |
| `cd=03` | - | `dev_data::CMD_GET_NET_INFO`, `dev_network_entity::CMD_GET_NET_INFO`, `dev_venus_x::CMD_GET_NET_INFO` |  |
| `cd=04` | - | `dev_data::CMD_GET_ERROR_CODE`, `dev_venus_x::CMD_GET_ERROR_CODE` |  |
| `cd=18` | `ct_t=<v>,mac=<MAC>` | `dev_venus_b::CMD_SET_CT` | NEW |
| `cd=18` | `meter=<v>,mac=<MAC>` | `dev_venus_g::CMD_SET_CT`, `dev_venus_x::CMD_SET_CT` | NEW |
| `cd=19` | - | `dev_venus_x::CMD_GET_CT_POWER` |  |
| `cd=33` | - | `dev_data::CMD_SET_TIME`, `dev_venus_x::CMD_SET_TIME` |  |
| `cd=38` | - | `dev_venus_x::CMD_SET_GRID_POWER_STANDARD` |  |
| `cd=44` | - | `dev_data::CMD_SET_DOD`, `dev_venus_x::CMD_SET_DOD` | NEW |
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
| `cd=64` | `ups_p=<v>` | `dev_data::CMD_SET_UPS_CHARGE_POWER`, `dev_venus_b::CMD_SET_UPS_CHARGE_POWER`, `dev_venus_x::CMD_SET_UPS_CHARGE_POWER` | NEW |
| `cd=65` | - | `ota_type_entity::?` |  |
| `cd=65` | `lapi_en=<v>,self_port=<v>,remote_port=<v>` | `dev_data::CMD_SET_LOCAL_API`, `dev_venus_x::CMD_SET_LOCAL_API` | NEW |
| `cd=65` | `start` | `ota_model::startUpgradeHttp` |  |
| `cd=66` | - | `vg_ota_type_entity::?`, `ota_model::getUpgradeStatus` |  |
| `cd=67` | - | `dev_data::CMD_GET_SCAN_WIFI_LIST`, `dev_venus_x::CMD_GET_SCAN_WIFI_LIST` | NEW |
| `cd=<v>` | - | `dev_data::CMD_SET_MODE_MANUAL_INFO` |  |
| `cd=A0` | - | `ota_model::getUpgradeStatus`, `vb_ota_type_entity::?` |  |
| `cd=A1` | - | `vb_ota_type_entity::?` |  |
| `cd=A1` | `cnt=<v>` | `ota_model::setUpgradePrepareHttp` |  |
| `cd=A2` | - | `vb_ota_type_entity::CMD_SET_UPGRADE_INFO_HTTP` |  |
| `cd=A2` | `num=<v>` | `vb_ota_type_entity::CMD_SET_UPGRADE_INFO_HTTP` |  |
| `cd=A3` | - | `vb_ota_type_entity::?` |  |
| `cd=A3` | `start` | `ota_model::startUpgradeHttp` |  |

## 12. Sun / inverter monitor group

Product identity of this group was not verified.

| cd | Parameters | App function(s) | New in 1.6.76 |
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

## 13. Shared and unattributed commands

| cd | Parameters | App function(s) | New in 1.6.76 |
|---|---|---|---|
| `cd=02` | `md=1,tm=<1\|2\|3>,ab=<0\|1>,cb=<v>,ce=<v>,db=<v>,de=<v>` | `system_setting_workmodeset::sentModeTime` |  |
| `cd=02` | `md=<v>` | `system_setting_workmodeset::getAppBar` |  |
| `cd=05` | - | `my_item2::getBody`, `my_item::getBody` |  |
| `cd=06` | `do=<v>` | `my_item::getBody` |  |
| `cd=09` | `wy=<v>` | `setTimeZone_page::setTimeZoneToDevices` |  |
| `cd=12` | `ht=<v>` | `utils::syncServiceToDevice` |  |
| `cd=20` | `md=1,no=<v>` | `common_command::toggleAcOutput` |  |
| `cd=20` | `md=2,no=<v>` | `common_command::toggleDcOutput` |  |
| `cd=20` | `md=3,no=<v>` | `common_command::toggleLedLight` | NEW |
| `cd=29` | `single_mode=1,b2500_num=0` | `add_Group_selectDevice::sendCombModelForDeviceInfo` |  |
| `cd=32` | `es=<v>` | `system_setting_function::getBody` |  |
| `cd=35` | `ar=<v>` | `system_setting_function::getBody` |  |
| `cd=37 / cd=44` | `dod=<v> / do=<v>` | `system_setting_function::getBody` |  |
| `cd=45` | `cv=<v>` | `system_setting_function::getBody` |  |
| `cd=48` | - | `common_command::getErrorCodefromDevice` |  |
| `cd=51` | - | `device_home::createBody`, `device_home::didChangeDependencies`, `device_home::_onRefresh` |  |
| `cd=pull_log` | `taskId=<v>` | `remote_log_pull_helper::_publishResult` |  |

`getMqttBase` itself builds `cd=<n>,<params>` / `cd=<v>`. `pull_log` is the remote log pull helper.

## 14. New in 1.6.76 compared with 1.6.72

| String | Product / function | Meaning |
|---|---|---|
| `cd=12,op=0` / `cd=12,op=1,http=<URL>` | EV charger: `EvChargerConnector.queryServerUrl/setServerUrl` | Server URL get/set |
| `cd=46`, `cd=47,<key>=<value>`, `cd=51,url=<URL>` | EV charger: `queryDeviceStatus/setChargeState/triggerOtaUpgrade` (HMH: `getEVChargerInfo` 46, `sendEVChargerSettings` 47, `evChargerStartOTA` 51) | Status, charge control, OTA |
| `cd=72,...` | Venus: `VenusEvChargerController` | EV charger IP/MAC binding |
| `cd=67` | New modules: `CMD_GET_SCAN_WIFI_LIST` | Wi-Fi scan list |
| `cd=44` / `cd=45` (bare) | V6000: `queryV6000Fc41dVersion` / `updateV6000Fc41dFirmware` (`cd=45,fc41d_url=<URL>`); HMH: `getProviderNowData` = 45; new modules: `CMD_SET_DOD` = 44 | FC41D version / OTA; provider data; DoD |
| `cd=90` / `cd=91,type=<t>` | HMH: `getScanShellyList` / `connectShellRPC` | Shelly scan / connect (new variants of 60/61) |
| `cd=68` | HMH: `setSystemCheck` | System check |
| `cd=16` | CT: `getCTBindInverterDetailInfoByMqtt`; HMH: `setSilentMqttinOTA` (`cd=16,mod=`) | CT bind detail / silent OTA mode |
| `cd=18,ct_t=<v>,mac=<MAC>` / `cd=18,meter=<v>` | Venus E-mini: `CMD_SET_CT`; `CommonCommand::setCtType` | CT type |
| `cd=20,md=3,no=<v>` | `CommonCommand.toggleLedLight` | LED toggle (next to md=1 AC / md=2 DC output) |
| `cd=30,data=<v>` | Parsed in `new_home_MqttTool.handMqttMessage` (response side) | Response field, not a send template |
| `cd=64,ups_p=<v>` / `cd=65,lapi_en=<v>,self_port=<v>,remote_port=<v>` | New modules (dev_data, venus_x, venus_e_mini): `CMD_SET_UPS_CHARGE_POWER` / `CMD_SET_LOCAL_API` | UPS charge power / local API |

## Appendix A: Static HM BLE frames

Automatically extracted constants from the BLE helper classes (`Blecommon*`). Only the first 5 bytes (`73 LEN 23 CMD P0`) are shown; byte 4 can be
missing or wrong when the list is built dynamically, CMD is reliable. Dynamic builders in `common_command` are not complete in this table:
`setBMSCodeFunction` 0x40, `setLocalApiEnable` 0x28, `setAcCouplerMaxChargePower` 0x16, `setVenusPeakShaving` 0x29, HTTP-OTA prepare 0x38, start HTTP OTA 0x3A, `setVid` / `requestVid` 0x50 / 0x51.
The name-prefix hint is a guess from the constant name, not a verified product attribution.

| CMD | Constants (first 5 bytes `73 LEN 23 CMD P0`) | Name-prefix hint |
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

## Appendix B: Limits of this vocabulary

* Static analysis only; no frame was captured and no device was addressed.
* Rows other than "Hand-verified" come from automatic literal extraction. Builders that assemble the `cd` number or parameters at runtime (`cd=<n>`) are incomplete.
* The meaning of rows without hand verification comes from the function name only.
* BLE constants are not attributed to products; only the pairings marked in section 4.2 exist.
