# Marstek App vs Control Firmware: Command Map (Venus D, Control 147-151)

> 🇩🇪 Deutsche Version: [Marstek_App_vs_Firmware_Befehlszuordnung_de.md](Marstek_App_vs_Firmware_Befehlszuordnung_de.md)

Status 2026-10-02. Maps the Venus-relevant commands of the Marstek app 1.6.76 ([vocabulary](Marstek_App_Command_Vocabulary_1.6.76.md)) to the handlers of the Venus D
Control firmware. Every row was checked against the decompiled dispatchers of Control v151: `MQTT_CtrlCommand_Dispatcher@0x080244e0`,
`BLE_App_Command_Dispatcher@0x0800c730` and `MQTT_JSON_RPC_Dispatcher@0x0801f4f8`. Version differences were checked in the dispatchers of
Control 147, 149, 149.2, 150 (base 0x4800) and 151. Static analysis only; nothing was sent to a device.
Related: [BLE command map v150](../BLE/VNSD_BLE_Command_Map_v150.md), [battery pack recovery, firmware side](../BMS/VNSD_Battery_Pack_Recovery_Firmware.md).

## 1. Legend

* **Case** = the hex case label of the dispatcher switch. For MQTT it is the hex of the decimal `cd` (`cd=40` = 0x28, `cd=44` = 0x2c).
* **Status**: yes = case exists and the parameters the app sends are parsed; partial = case exists but something the app sends is ignored or only matched by key name; none = no handler.
* **Since** = first Control version with the handler (versions: 147, 149, 149.2, 150, 151).
* Replies use `MQTT_Publish_CommandResult(cd, result)` = `cd=<n>,ret=<result>` unless another format is shown.

## 2. How the dispatchers work (verified in v151)

* **MQTT:** the handler looks for `cd=` in the payload, takes `atoi(...) & 0xff` and switches on it (so `cd=300` would alias `cd=44`). A payload without `cd=` goes to the JSON-RPC dispatcher. The `/App/<id>/` part of the topic is compared with the device id (`g_mqttTopicId`) and the group id (`g_mqttGroupIdStr`); the result is kept as an id type and a mismatch alone does not stop the `cd=` dispatch in this function. Replies from `MQTT_Publish_CommandResult` are published on `marstek_energy/<devType>/device/<devId>/ctrl`.
* **BLE:** `packet[2]` must be 0x23 and the XOR over the first `LEN - 1` bytes must equal the last byte; `packet[3]` is the command id. Replies come from `BLE_Send_Response(cmd, data, len)`.
* **JSON-RPC:** used for payloads without `cd=` (MQTT/cloud) and by the UDP local API. The id-to-name table has 16 entries (id 12 is unused).

## 3. MQTT: app commands mapped to the firmware

| App command | Case | Handler / effect | Parameters parsed | Reply | Status | Since |
|---|---|---|---|---|---|---|
| `cd=1` (status request) | 0x01 | `MQTT_StatusReport_BuildAndPublish(1)`: full status report | none | status report payload | yes | 147 |
| `cd=2,md=0` / `md=1` / `md=5,nl=1` | 0x02 | sets `g_userWorkMode` (EEPROM 0x301); `md=1` also clears `g_workModeUpsSubFlag` | `md=0`, `md=1`, `md=5`; `nl` is **not** parsed here | `cd=2,md=<mode>` | yes | 147 |
| `cd=3,md=1,nm=<slot>,bt,et,...` (`setTimePower`) | 0x03 | manual time slot: writes the `g_touSlot*` arrays, then `Config_Save_TimeSlotEntries(nm)` | `md=1`, `nm` (slot < 10), `bt`, `et` (`HH:MM`, zero-padded), `wk`, `vv`, `as`; all required, in this order | `cd=3,md=<mode>`; `cd=3,ret=0` if `nm` >= 10 (not in 147) | yes | 147 |
| `cd=3,md=2,id,in,on` (`setTradePriceAndArea`) | 0x03 | no `md=2` branch: the command falls through, nothing is stored | - | no reply | none | - |
| `cd=<n>,yy,mm,rr,hh,mn` (`setTime`) | 0x04 | `RTC_SetDateTime` (year - 2000), then reply with the RTC; `read` only reads | `read`; `yy`, `mm`, `rr`, `hh`, `mn` in this order; a missing key aborts without reply | `cd=4,<Y>-<M>-<D> <h>:<m>:<s>` | yes (key match; cd literal not extracted) | 147 |
| `cd=5,rs=1` / `rs=2` | 0x05 | `Factory_Reset(1\|2\|3)` after the reply; no `rs` -> `ret=0` | `rs=1`, `rs=2`, `rs=3` | `cd=5,ret=1` | yes | 147 |
| `cd=9,ot=0` | 0x09 | `Quectel_OTA_StartUpdate_AndReboot(0)`: starts the FC41D module OTA and reboots | `ot=0`, `ot=1` | `cd=9,ret=1` before the start; `ret=0` otherwise | yes | 147 |
| `cd=10` (`getMqttCurrentTimeState`) | 0x0a | `MQTT_Publish_WiFi_Version`: reports the FC41D version | none | version string | partial (the app function name suggests a different purpose) | 147 |
| `cd=11,bc=<0\|1>` | 0x0b | EPS/backup: `Config_Mode_Apply(1,bc)`, `g_epsReserveEnable` (EEPROM 0x300) | `bc=0`, `bc=1` | `cd=11,ret=1` (0 if the stored flag differs); no reply for other `bc` | yes | 147 |
| `cd=12,ht=<v>` (`syncServiceToDevice`) | 0x0c | HTTP/cloud server type, EEPROM 0x441 | `ht=0..4` (`op=`, `http=` are ignored) | `cd=12,ret=1`, `ret=0` without valid `ht` | yes | 147 |
| `cd=13` (`_bleTimerInfo`) | 0x0d | `MQTT_Publish_ErrorEventLog`: error log | none | error log payload | partial (the app function name suggests a different purpose) | 147 |
| `cd=15,vs=<W>` (`setAcCouplerPowerVersion`) | 0x0f | power class: 800 -> `g_workMode`=1 (+ `Config_ScheduleEntries_ClampPower800W`), 2200 -> 3, 2500 -> 0; `Config_Write_U16`, EEPROM 0x90 | `vs` in {800, 2200, 2500} | `cd=15,ret=1\|0` | yes | 147 |
| `cd=16,cp=<W>` | 0x10 | `Config_Write_U16_0x202` (max charge power) | `cp` 300..2500 | `cd=16,ret=1\|0` | yes | 147 |
| `cd=<n>,meter=<v>[,mac=<MAC>]` (`setCtType`) | 0x12 | `Cloud_HTTP_Response_Parser`: sets the meter source type, then `mac=` (serial) and `ct_dev=` | `meter` < 13; `mac=`, `ct_dev=` | echoes the key only: `cd=18,meter` | yes (key match; cd literal not extracted) | 147 |
| `cd=<n>,dchrg=<v>` (`setPowerReplenishmentMode`) | 0x12 | same parser: `dchrg` 0/1 -> `Config_Write_Flag_0x367` (MPPT channel sum flag) | `dchrg` | echoes the key only | yes (key match; cd literal not extracted) | 147 |
| `cd=<n>,seq_check` (`setSeqCheck`) | 0x12 | same parser: `Cloud_Response_Action(1)` | `seq_check` | echoes the key only | yes (key match; cd literal not extracted) | 147 |
| `cd=18,ct_t=<v>,mac=<MAC>` (E-mini) | 0x12 | `ct_t` is not a key of the parser: no action | - | echo `cd=18,ct_t` | none | - |
| `cd=19` (`getCtPowerInfo`) | 0x13 | `MQTT_Publish_Grid_Power`; with `meter_data` also `MQTT_Publish_Meter_Data`; `debug` -> `MQTT_ConnectionStatus_Report_Send` (151 only) | `meter_data`, `debug` | CT power payload | yes | 147 |
| `cd=20,len=<n>,url=<URL>` (`updateAcCouplerFc4Firmware`) | 0x14 | FC41D OTA by URL: with `port=` immediately `Quectel_OTA_StartUpdate_FromURL_AndReboot`, otherwise `g_otaUrlStartPendingFlag=1` | `len` (< 255), `url=`, optional `port=`, in this order | `cd=20,ret=1` (sent before the start); `ret=0` if `len` >= 255 | yes | 147 |
| `cd=22` (`notifyAcCouplerUpgradePowerVersion`) | 0x16 | `g_activeUpgradeInfoFlag = 1` | none | `cd=22,ret=1` | yes | 147 |
| `cd=<n>` + `len`/`ip=` (`setP1MeterIp`) | 0x18 | P1/meter IP: set (`len`, `ip=`) or query (`ip?`), EEPROM 0x3500; stored only (the BLE twin 0x21 is rated not effective in the BLE map) | `ip?`; `len` (<= 16), `ip=` | `cd=24,ip=<IP>` | yes (key match; cd literal not extracted) | 147 |
| `cd=<v>` (`getDeviceIp`) | 0x1a | `MQTT_Publish_Network_Info` | `net?` | network info | yes (cd literal not extracted) | 147 |
| `cd=<n>,num=<v>` (`requestNowData`) | 0x1d | `MQTT_Publish_PowerSchedule(num)` | `num=` (`len=` is ignored) | schedule payload | partial (cd literal not extracted) | 147 |
| `cd=30,api=<0\|1>,port=<p>` | 0x1e | local API: `g_localApiEnable` (EEPROM 0x371), `g_localApiUdpEnable`, `g_apiConnState` (1 / 4), port EEPROM 0x372 | `api` (0/1) before `port`; `port` is searched after `api`, so both keys are needed in this order | `cd=30,ret=<enable>,port=<port>` | yes | 147 |
| `cd=40,number_as` | 0x28 | `BLE_Apply_ConfigReg_Offset3(0)` -> `Config_Apply_SingleReg(3)` -> CAN `0x100101AA` byte 3 (pack-coding handshake); `bms_rst` -> index 1 (byte 4, BMS restart; never sent by the app) | `number_as`, `bms_rst` | `cd=40,ret=1`; `ret=0` otherwise | yes | 147 |
| `cd=42,bms_idx=255` | 0x2a | `MQTT_Publish_BMS_Data(idx)` | `bms_idx` 0..8 or 255 (other values become 0) | BMS data payload | yes | 147 |
| `cd=51` (`getHttpOtaState`) | 0x33 | `MQTT_Publish_Network_Status`: OTA state | none (`url=` of the EV-charger variant is ignored) | OTA state payload | yes | 147 |
| `cd=52,clr=<v>,cnt=<v>` | 0x34 | `clr=1` erases the 4 URL slots (flash); `cnt` -> `g_otaModuleCount` (max 4) | `clr`, then `cnt` | `cd=52,ret=<cnt<5>`; no reply without `cnt` | yes | 147 |
| `cd=53,num=<v>,<entries>` | 0x35 | `Quectel_MQTT_OTA_Info_Parser` (comma tokens `mod`, `type`, `size`, `crc`, `ver`, url); `check?` -> `OTA_BuildAndSend_StatusReport` | `check?` or the entries | `cd=53,ret=<n>` | yes | 147 |
| `cd=54,start` | 0x36 | `g_otaUpdatePendingFlag = true` (starts the pending OTA) | `start` | `cd=54,ret=1` | yes | 147 |
| `cd=56,dod=<v>` | 0x38 | `Cloud_DOD_Command_Handler`: `Config_Set_DischargeCutoff_WithRelay` | `dod` (reply ret=1 for 30..88) | `cd=56,ret=1\|0` (sent by the handler itself) | yes | 147 |
| `cd=63,...` (`setVenusPeakShaving`) | 0x3f | peak shaving: `Config_Write_PowerClampMode_0x394(1, vv)` and `Config_Write_PowerOffset(vv)`; `as=0` switches off and writes offset 0 | `as` (0/1), then `vv` (W, signed) | `cd=63,ret=1\|0` | yes | 150 |
| `cd=72,type?` / `type=1,len,ip,mac` (EV charger binding) | - | no case: the highest case is 0x42 (66) | - | no reply | none | - |

## 4. MQTT cases without a Venus app counterpart in the vocabulary

The app extraction contains no Venus sender for these cases. Builders with a runtime `cd` number (`cd=<n>`) cannot be excluded.

| cd | Case | Function | Parameters | Reply | Since |
|---|---|---|---|---|---|
| 0 | 0x00 | upload target mask | `set=1\|2\|3` | `cd=0,ret=<mask>` | 147 |
| 6 | 0x06 | developer data: `rd=1` uid, `rd=2` bootloader version, `rd=3` http count / slot / time, otherwise develop-mode info | `rd=1..3` | - | 147 |
| 7 | 0x07 | developer mode `dm`: 0 modem hardware reset, 1 reboot, 2 delete TLS user cert + reboot, 3 `Inverter_Set_Flag_Reg_0x05(0)`, **4 `Config_Apply_SingleReg(4)` = BMS restart (byte 4)**, 5 `Inverter_Set_Reg_0x04(0)`, 6 `CH395_Reset_And_Reinit` | `dm=0..6` | `cd=7,ret=1\|0` | 147 |
| 8 | 0x08 | work-mode auto change, EEPROM 0x366 | `dt=0\|1` | `cd=8,ret=1\|0` | 147 |
| 14 | 0x0e | BMS data info (`BMS_StatusQuery_Response_Send`) | - | - | 147 |
| 17 | 0x11 | max discharge power (<= 2500, <= 800 on the 800 W class) | `dp` | `cd=17,ret=1\|0` | 147 |
| 21 | 0x15 | event log (`MQTT_Publish_Event_Log`) | - | - | 147 |
| 23 | 0x17 | parallel machine mode | `pm=0\|1\|2` | `cd=23,ret=<mode>` | 147 |
| 25 | 0x19 | generator flag (RAM only) | `gen=0\|1` | `cd=25,ret=<flag>` | 147 |
| 27 | 0x1b | jump control, EEPROM 0x92; `jp=0` also forces mode 1 | `jp=0\|1` | `cd=27,jp=<n>` | 147 |
| 28 | 0x1c | **reboot** after the reply and a 1 s delay | - | `cd=28,ret=1` | 147 |
| 31 | 0x1f | vid info | `topic=1` | `cd=31,vid=...` | 147 |
| 41 | 0x29 | BMS version (`MQTT_Publish_BMS_Version`) | - | - | 147 |
| 43 | 0x2b | surplus feed-in, `Config_Write_BoolFlag_0x375` | `full_d=0\|1` | `cd=43,ret=<flag>` | 147 |
| 44 | 0x2c | BMS error lock: `Config_Apply_WorkModeReg(lock)` -> byte 8 (unlock + master restart) / byte 9 (lock) | `lock=0\|1` | `cd=44,ret=1` | 147 |
| 45 | 0x2d | MPPT data (`MQTT_Publish_Inverter_Telemetry`) | - | - | 147 |
| 50 | 0x32 | VNS MPPT data (log text says "Set ems ctrl info"; the handler only publishes) | - | - | 147 |
| 55 | 0x37 | BLE advertising enable | `adv=0..3` | `cd=55,ret=<flag>` | 147 |
| 57 | 0x39 | self-control power offset (`Cloud_Handle_SelfCtl_Power`) | `selfctl_power=` | `cd=57,selfctl_power=<W>` | 147 |
| 58 | 0x3a | AI strategy query | - | - | 147 |
| 59 | 0x3b | LED state (EEPROM 0x395 only since 151) | `led=0\|1` | `cd=59,ret=<led>` | 147 |
| 61 | 0x3d | `g_backupDataRequestFlag` | `bk_en=` | `cd=61,ret=<n>` | 149 |
| 62 | 0x3e | device IDs (`MQTT_Publish_DeviceIDs`); only 149 / 149.2 | - | - | 149 |
| 64 | 0x40 | device IDs (`MQTT_Publish_DeviceIDs`) | - | - | 150 |
| 65 | 0x41 | INV/grid status (`MQTT_Publish_Grid_Status`) | - | - | 150 |
| 66 | 0x42 | INV debug: `debug=1` maintenance state 2, `2` work-mode config, `3` / `4` `CAN_Write_Category0x18(1/0)` (4 also clears the maintenance state) | `debug=1..4` | `cd=66,ret=...` | 151 |

## 5. App commands without a Venus D handler (or only partly handled)

| App command | Product | Result on a Venus D (Control 147-151) | Note |
|---|---|---|---|
| `cd=72,type?` / `cd=72,type=1,len=<n>,ip=<IP>,mac=<MAC>` | Venus: EV charger binding | no case in 147-151 (highest case 66) | needs newer firmware |
| BLE cmd 0x61 (`0B` / `0A`+ip`,`mac) | Venus: EV charger binding, BLE fallback | no BLE case (highest 0x5B) | needs newer firmware |
| `cd=3,md=2,id=<v>,in=<v>,on=<v>`; BLE 0x09 payload `02` | Venus: trade price and area | case 3 / BLE 0x09 have no mode-2 branch; ignored (BLE still answers with the unchanged mode) | not supported in 147-151 |
| `cd=2,md=5,nl=1` | Venus: economy mode | case 2 sets mode 5 but does not parse `nl` (only case 3 reads `nl`, and only echoes it) | `nl` has no effect |
| `cd=18,ct_t=<v>,mac=<MAC>` | Venus E-mini: CT type | case 0x12 knows `meter=`, not `ct_t=`; echo without action | other product |
| `cd=12,op=0` / `cd=12,op=1,http=<URL>` | EV charger A3600CE: server URL | case 0x0c parses `ht=` only; without it the reply is `cd=12,ret=0` | other product |
| `cd=46`, `cd=47,<k>=<v>` | EV charger / HMH | no case (46-49 absent in all versions) | other product |
| `cd=51,url=<URL>` | EV charger: OTA trigger | case 0x33 ignores all parameters and only reports the OTA state; no OTA is started | other product |
| `cd=67` | New modules: Wi-Fi scan list | no case (highest 66) | needs newer firmware or other product |
| `cd=44` (bare), `cd=45` (bare) | V6000 / HMH / new modules | 0x2c needs `lock=` (no reply otherwise); 0x2d reports MPPT data and ignores parameters | other product |
| `cd=63,ct_chg_type=`, `cd=64,ups_p=`, `cd=65,lapi_en=,self_port=,remote_port=` | New modules (Venus X / E-mini) | 63 = peak shaving (needs `as=`; else `ret=0`), 64 = device IDs, 65 = grid status: no setter for these parameters; 147 / 149 / 149.2 have no case 63-65 (149 / 149.2 only 62) | other product |
| `cd=90`, `cd=91,type=<t>`, `cd=68` | HMH family | no case | other product |
| `cd=09,wy=<v>` | generic time-zone page (shared) | case 0x09 needs `ot=`; without it the reply is `cd=9,ret=0`, nothing else happens | no time-zone setter in 147-151 (time is set with `cd=4` / BLE 0x0B) |
| `cd=30,data=<v>` | response field in the app | not a send template; `data=` is not parsed by case 0x1e | n/a |
| BLE builders 0x38 (`httpOtaPrepare`) and 0x3A (`startHttpOta`) | Venus: HTTP OTA | no BLE case 0x38 / 0x3A; the firmware BLE OTA path is case 0x52 with sub commands | not supported over BLE in 147-151 |

## 6. Same `cd` number, different product

The new-module framework (Venus X / E-mini / G) and other products reuse small `cd` numbers with other meanings. What a Venus D v151 does if such a command arrives (only for information; the app sends these to the other products' own topics):

| `cd` | App function (new modules / other) | Case in v151 | What the Venus D does |
|---|---|---|---|
| `2` | `CMD_SET_WORK_MODE` | 0x02 | without `md` nothing changes; the reply carries the current mode |
| `3` | `CMD_GET_NET_INFO` | 0x03 | without `md=1` / `md=5` nothing happens, no reply (network info is `cd=26` + `net?` here) |
| `4` | `CMD_GET_ERROR_CODE` | 0x04 | returns the RTC time (the error log is `cd=13` here) |
| `19` | `CMD_GET_CT_POWER` | 0x13 | same meaning: CT power |
| `33, 38, 46, 60, 67` | `CMD_SET_TIME`, `CMD_SET_GRID_POWER_STANDARD`, `CMD_SET_ACCESS_POWER`, `CMD_SET_SERVER`, `CMD_GET_SCAN_WIFI_LIST` | - | no case in any of 147-151 |
| `62` | `CMD_GET_BATTERY_PACK_INFO`, `CMD_SET_SHELLY_PRO_3EM_PORT` | 0x3e (149 / 149.2 only) | device IDs, read-only; no case in 147 / 150 / 151 |
| `44` | `CMD_SET_DOD` | 0x2c | needs `lock=`; bare `cd=44` does nothing and does not reply (DoD is `cd=56,dod=` here) |
| `54` | `CMD_SET_ANTL_REVERSE` | 0x36 | needs the substring `start`; otherwise nothing |
| `55` | `CMD_SET_BLUETOOTH_STATE` | 0x37 | needs `adv=`; otherwise nothing |
| `56` | `CMD_SET_LED_STATE` | 0x38 | needs `dod=`; otherwise nothing (LED is `cd=59,led=` here) |
| `57` | `CMD_SET_MAX_GRID_POWER` | 0x39 | needs `selfctl_power=`; otherwise nothing |
| `58` | `CMD_SET_CT_PHASE_AUTO_CHECK` | 0x3a | read-only: AI strategy query |
| `59` | `CMD_GET_NOW_POWER` | 0x3b | `led=` is read without a NULL check (see section 10) |
| `61` | `CMD_SET_REBOOT` | 0x3d | `bk_en=` is read without a NULL check (see section 10); not a reboot |
| `63, 64, 65` | `CMD_SET_CT_CHARGE_TYPE`, `CMD_SET_UPS_CHARGE_POWER`, `CMD_SET_LOCAL_API` | 0x3f / 0x40 / 0x41 | peak shaving (needs `as=`) / device IDs / grid status; no setter |
| `66` | `getUpgradeStatus` (OTA status query) | 0x42 | `debug=` is read without a NULL check (see section 10) |
| `28` | `getCtPowerInfo` (CT variant) | 0x1c | **reboots the Control**; never send the CT variant to a Venus D |
| `3 (md=1)` | B2500 `cd=03,md=1` | 0x03 | sets manual mode 1 without a slot |

## 7. BLE: app frames mapped to `BLE_App_Command_Dispatcher`

Frame format `73 | LEN | 23 | CMD | payload | XOR`. Only the app commands that are Venus-related in the vocabulary are listed; the complete firmware BLE table is in the [BLE command map v150](../BLE/VNSD_BLE_Command_Map_v150.md).

| App BLE cmd | App builder / constant | Case | Handler / effect | Since | Status |
|---|---|---|---|---|---|
| 0x06 | `ACCOUPLE_FACTORY_RECOVERY` (`02`), `POWER_SEND_RESET` (`01`) | 0x06 | `Factory_Reset(1\|2\|3)` for byte 4 = 1/2/3; reply echoes the byte first | 147 | yes |
| 0x09 `00` | `ACCOUPLE_WORK_MODE_AUTO` | 0x09 | byte 4 = 0: `g_userWorkMode = 0`, EEPROM 0x301, reply = mode | 147 | yes |
| 0x09 `01` (LEN 0x0F) | `ACCOUPLE_SEND_MANUAL_SET_POWER` | 0x09 | byte 4 = 1: slot [5] (< 10), start [6..7] = hh, mm, end [8..9], weekday mask [10], power int16 LE [11..12], enable [13]; `Config_Save_TimeSlotEntries` | 147 | yes |
| 0x09 `02` (LEN 0x10) | `ACCOUPLE_SEND_TRADE_PRICE` | 0x09 | byte 4 = 2: no branch; the reply is the unchanged mode (the same value is written back to EEPROM 0x301) | - | none |
| 0x15 (LEN 7) | `ACCOUPLE_SET_DISCHARGE_POWER_VERSION` | 0x15 | u16 LE in {800, 2200, 2500} -> power class (`g_workMode` 1 / 3 / 0, EEPROM 0x90); other values are ignored without a reply | 147 | yes |
| 0x16 | builder `setAcCouplerMaxChargePower` | 0x16 | u16 LE 300..2500 -> `Config_Write_U16_0x202`; reply = the stored u16 | 147 | yes |
| 0x1A | `ACCOUPLE_GET_CT_POWER_INFO` | 0x1a | payload ignored; returns 18 bytes of CT values (`g_ctPowerCh1..3`, `g_ctPowerNow`, grid sample) | 147 | yes |
| 0x28 | builder `setLocalApiEnable` | 0x28 | byte 4 = 0/1 enable; port u16 LE [5..6] is written in any case (EEPROM 0x372); `g_apiConnState` = 3 / 4 (MQTT sets 1 / 4) | 147 | yes |
| 0x29 | builder `setVenusPeakShaving` | 0x29 | byte 4 = 1/0 peak state, int16 LE [5..6] = W; `Config_Write_PowerClampMode_0x394` and `Config_Write_PowerOffset` | 150 | yes |
| 0x40 `00` | builder `setBMSCodeFunction` | 0x40 | `BLE_Apply_ConfigReg_Offset3(byte 4)`: 0 -> CAN byte 3 (pack coding, what the app sends); 1 -> byte 4 (BMS restart, never sent by the app); reply echoes byte 4 | 147 | yes |
| 0x50 / 0x51 | builders `setVid` / `requestVid` | 0x50 / 0x51 | write (sub `0x0A`, string up to 32 bytes) / read vid and xid info | 147 | yes |
| 0x38 / 0x3A | builders `httpOtaPrepare` / `startHttpOta` | - | no case; the BLE OTA path is 0x52 (sub commands 0x0A-0x0D) | - | none |
| 0x61 `0B` / `0A`... | EV charger binding (`cd=72` twin) | - | no case (highest 0x5B) | - | none |

## 8. JSON-RPC methods (firmware only)

The app extraction has no JSON-RPC; the methods are used by the local UDP API and the MQTT JSON path.

| Id | Method | Parameters | Builder / result | Since |
|---|---|---|---|---|
| 1 | `Marstek.GetDevice` | `params.ble_mac` = `"0"` or the device's own BLE id; otherwise error `Invalid params` | `MQTT_BuildDeviceDataResponse` | 147 |
| 2-6, 8-10 | `Wifi.GetStatus`, `BLE.GetStatus`, `Bat.GetStatus`, `PV.GetStatus`, `ES.GetStatus`, `ES.GetMode`, `EM.GetStatus`, `Marstek.GetData` | `params.id` (checked by `MQTT_ParseRpcParams`) | builders `MQTT_BuildWifi/Ble/Battery/Pv/Es/EsMode/EnergyMeter/MarstekDataResponse` | 147 |
| 7 | `ES.SetMode` | `config.mode`: `Auto` (`auto_cfg.enable`=1 -> `MQTT_SetWorkMode(1)`), `AI` (`ai_cfg.enable`=1 -> `MQTT_SetWorkMode(2)`), `Manual` (`manual_cfg`: `time_num` <= 9, `start_time`, `end_time` as `HH:MM`, `week_set` <= 0x7f, `power` +-2500, `enable` 0/1), `Passive` (`passive_cfg.power`, `cd_time` != 0 -> `MQTT_PassiveModeStart`), `UPS` (`ups_cfg.enable` 1: manual mode + EPS on + UPS sub flag; 0: mode 0, EPS off) | `MQTT_RPC_BuildSetResult` | 147 |
| 11 | `DOD.SET` | `params.value` 30..88 -> `Config_Set_DischargeCutoff_WithRelay` | set result | 147 |
| 13 | `Ble.Adv` | `params.enable` 0/1 -> EEPROM 0x36bd | set result | 149 |
| 14 | `Led.Ctrl` | `params.state` 0/1 -> `g_selLedEnable` (EEPROM 0x395 only since 151) | set result | 149 |
| 15 | `Ota.Set` | `params.config`: `device_type` (one of 4 slot names), `Mod` 1..4, `Version`, `URL`, `URL_LEN` (must equal the URL length), `SIZE`, `CRC` -> URL slot in flash, `g_otaUpdatePendingFlag = true` | set result | 149 |
| 16 | `Set.Ver` | `params.version` in {800, 2200, 2500} | set result | 149 |
| 17 | `Reset.Factory` | `params.type` 1 / 2 -> `Factory_Reset`; 3 -> result only, no action; other -> `Invalid params` | set result | 149 |

## 9. Version differences (Control 147 / 149 / 149.2 / 150 / 151)

The MQTT dispatcher handles decimal 0-31, 40-45, 50-59 in all versions; 61 from 149, 62 only in 149 / 149.2, 63-65 from 150, 66 only in 151 (147 lacks 61 and 63-66; 149 / 149.2 have 61 and 62; 150 has 61 and 63-65; 151 has 61 and 63-66). `x` = present, `-` = absent.

| Item | 147 | 149 | 149.2 | 150 | 151 |
|---|---|---|---|---|---|
| MQTT cases 0-31, 40-45, 50-59 | x | x | x | x | x |
| MQTT 61 (`bk_en=`) | - | x | x | x | x |
| MQTT 62 (device IDs) | - | x | x | - | - |
| MQTT 63 (peak shaving) | - | - | - | x | x |
| MQTT 64 (device IDs), 65 (grid status) | - | - | - | x | x |
| MQTT 66 (`debug=`) | - | - | - | - | x |
| MQTT 19 option `debug` | - | - | - | - | x |
| MQTT 3 error reply `cd=3,ret=0` (`nm` >= 10) | - | x | x | x | x |
| MQTT 40 (`number_as` / `bms_rst`), 44 (`lock=`), 5, 16, 30, 52-54, 56 (same strings and calls) | x | x | x | x | x |
| BLE 0x29 (peak shaving) | - | - | - | x | x |
| BLE 0x5A, 0x5B | - | x | x | x | x |
| BLE 0x40 (`@`), 0x43 (`C`), 0x28, 0x16, 0x15, 0x1A, 0x50, 0x51 | x | x | x | x | x |
| JSON-RPC ids 1-11 (without 12) | x | x | x | x | x |
| JSON-RPC ids 13-17 (`Ble.Adv`, `Led.Ctrl`, `Ota.Set`, `Set.Ver`, `Reset.Factory`) | - | x | x | x | x |
| LED setting persisted (EEPROM 0x395): MQTT 59, BLE 0x59, `Led.Ctrl` | - | - | - | - | x |

## 10. Observations and open points

1. **No NULL check** in MQTT cases 0x3b (`led=`), 0x3d (`bk_en=`) and 0x42 (`debug=`): the result of `strstr` is passed to `atoi` as `ptr + 4/6/6`. A bare `cd=59`, `cd=61` or `cd=66` (the new-module codes) therefore reads from a near-zero address. The effect on a live device was **not tested**; for 0x3b it can overwrite `g_selLedEnable` (and, since 151, EEPROM 0x395). Reported, not changed.
2. MQTT case 0x1e (`cd=30`) searches `port` with `strstr(ptr, ...)` after the `api` search without checking the first result; a payload without `api` passes NULL to `strstr`. The app always sends `api` first, so it does not trigger this.
3. Several handlers search their keys one after the other, each `strstr` starting at the previous match, so the **key order matters** (case 3: `nm`, `bt`, `et`, `wk`, `vv`, `as`; case 4: `yy`, `mm`, `rr`, `hh`, `mn`; case 30: `api`, `port`; case 52: `clr`, `cnt`; case 20: `len`, `url`, `port`; case 63: `as`, `vv`). The app templates keep this order. Times are read with `atoi(p + 3)` and `atoi(p + 6)`, so `HH:MM` must be zero-padded.
4. Case 3 writes `bt`, `et`, `wk`, ... into RAM arrays one by one; if a later key is missing the slot is **not** saved and no reply is sent, but the earlier fields are already changed in RAM.
5. `cd=56` (DoD): `Cloud_DOD_Command_Handler` calls `Config_Set_DischargeCutoff_WithRelay` **before** the reply range check (ret=1 only for 30..88). Whether the setter validates internally was not traced. JSON-RPC `DOD.SET` validates 30..88 before the call.
6. App rows marked "cd literal not extracted" (`setTime`, `setCtType`, `setPowerReplenishmentMode`, `setSeqCheck`, `setP1MeterIp`, `getDeviceIp`, `requestNowData`): the app builds the `cd` number at runtime; the pairing is by parameter names only.
7. `cd=10` and `cd=13`: the app function names (`getMqttCurrentTimeState`, `_bleTimerInfo`) do not match the firmware handlers (FC41D version, error log); what the app wants from them was not traced.
8. The JSON-RPC id-to-name pairing is taken from the raw decode of `g_mqttRpcMethodTable` in the v150 analysis (the v151 table was not decoded again); the dispatcher case numbers and parameters were read from the v151 source. The app extraction contains no JSON-RPC (only `cd=` and HM BLE), so the methods have no app counterpart here.
9. Not checked: Control images for Venus A (1509) and E3 (151), live behaviour on a device, and the replies on the wire (the reply formats come from the `snprintf` strings in the handlers).
10. The version columns were derived from the case lists and the string/call content of the dispatcher sources of 147, 149, 149.2, 150 (base 0x4800) and 151; parameter literals are sometimes rendered differently by the decompiler (for example `cp` missing in the `strstr` argument of some versions) without a code difference.

For what the BMS side does with `cd=40` (CAN `0x100101AA` byte 3) see [VNSD_Battery_Pack_Recovery_Firmware.md](../BMS/VNSD_Battery_Pack_Recovery_Firmware.md).
