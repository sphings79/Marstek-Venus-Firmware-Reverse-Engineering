# VNSD – Battery Pack Recovery ("Akkupack-Wiederherstellung"): firmware side (Venus D)

> 🇩🇪 Deutsche Version: [VNSD_Akkupack_Wiederherstellung_Firmware_de.md](VNSD_Akkupack_Wiederherstellung_Firmware_de.md)

Status 2026-10-02: **firmware side AND app side complete. The app button 'Battery Pack Recovery' sends the BMS coding (pack address numbering) command, NOT a recovery charge** (proven by decompiling app 1.6.76 / 1.6.72 with Blutter, see §7). Related: register resolution
[`../Modbus_RS485_TCP/VNSD_Unclear_Registers_Resolution_2026-10-01.md`](../Modbus_RS485_TCP/VNSD_Unclear_Registers_Resolution_2026-10-01.md),
BLE command map [`../BLE/VNSD_BLE_Command_Map_v150.md`](../BLE/VNSD_BLE_Command_Map_v150.md).

Scope: Control 147/149/149.2/150/151 and BMS 117.7/118, text exports only. Marstek app 1.6.72 (`libapp.so`) and 1.6.76
(`libapp_new.so`): strings, plus a Dart AOT decompilation of 1.6.76 with Blutter (Dart AOT decompiler, §7). No live device was used.

## 0. Short answer

* **PROVEN (app decompilation, §7): the app button "Battery Pack Recovery" sends the BMS coding command (pack address numbering), not a recovery charge.**
  Over BLE it sends `73 06 23 40 00 16` (cmd 0x40, payload `00`); without BLE it sends over MQTT `cd=40,number_as` on topic
  `hame_energy/<type>/App/<id>/ctrl`. Internal names: `handleBmsCoding` / `setBMSCodeFunction`. In the firmware this reaches the BMS as CAN
  `0x100101AA` byte 3 (arms the pack-coding handshake `g_relayCommandPendingFlag`). There are no parameters and no follow-up command.
  Control 147–151 already handle it, so no newer firmware is needed.
* **The firmware has no command named or shaped like "pack recovery".** No BLE command, MQTT `cd=`, JSON-RPC method,
  UDP-API method or shell command in Control 147–151 is a dedicated "recover/reactivate pack" function.
* **Pack recovery does exist, but it runs on its own** in the BMS. The master BMS has a recovery-charge state machine
  (`PackRecovery_LowCell_Charge_SM`) and a low-voltage balancer (`Pack_LowVoltage_Balancer`). The balancer feeds a
  Control-side forced charge (PGN 0x1803 byte 7 bit 0 -> `Battery_Charge_Power_Limiter` + CAN `0x100401AA [00 04]`).
  The recovery SM gives up and shuts the pack down (pack mode 3) if no charge current of at least 1.1 A flows for
  about 300 s. That matches the app text "the device must be charged to activate it". No remote command starts or
  stops this logic.
* **What the app button does on a D** (remote paths that change BMS state, see §3): of the three candidates (unlock the BMS
  error lock, `0x100101AA` byte 8, on the master this also reboots the BMS; restart the BMS, byte 4; start
  pack address encoding, byte 3) the app sends **only the last one** (byte 3). The unlock and restart commands are never sent by the app (§3).
* **Modbus:** none of the unlock/restart commands has a Modbus write path. The nearest Modbus actions are
  45027 (master BMS reboot, FUNCTIONAL, dangerous), 45029 (pack encode, CONDITIONAL; the Modbus-side equivalent of the app button, §4) and 45028 (BMS factory reset,
  FUNCTIONAL, dangerous). 45031 would send byte 4 (restart), but on D it never gets that far (FUNCTIONLESS). 45030 is
  FUNCTIONLESS. 32112 shows the lock state (read). 32107 = 0 is the observable sign that recovery charging is running.
  42010 (force charge) can supply the charge current the recovery needs (indirect only).
* **Verdict for "Battery Pack Recovery" as a remote function on D:** the command is identified (BMS coding, cmd 0x40 /
  `cd=40,number_as`) and FUNCTIONAL on v151 (handler present, §1.1 byte 3), but it only arms the pack-numbering handshake.
  The underlying BMS recovery charge is FUNCTIONAL but stays automatic and is **not** started by the button.
  The newer `cd=` codes (67/72/90/91 …) are unrelated to it (no handler on v151, §6).

## 1. Every remote path that makes the Control send something to the BMS / change pack state

CAN transport: `Register_WriteValue` -> CAN1. The BMS accepts all IDs in hardware and routes them in software
(see `VNSD_Unclear_Registers_Resolution_2026-10-01.md`): if slot == 1 (master) -> `CAN_RX_Handler@0x08008240` (fixed IDs) + `CAN_CMD_Pack_Activation@0x0800df70`
(class 3); if slot != 1 -> `CAN_RX_System_Command_Handler@0x0800f0bc` + `CAN_RX_Command_Dispatcher@0x0800f128`.

### 1.1 `0x100101AA` byte commands (`Config_Apply_SingleReg@0x080317e8`)

| byte | BMS master (`CAN_CMD_01_Handler@0x08008040`) | BMS slot != 1 (`CAN_RX_System_Command_Handler`) | Control senders (v151) |
|---|---|---|---|
| 1 | `g_pendingSystemCmd=1` -> after 10 ticks of `BMS_ProtectionAndCommTick@0x080095ac` `g_packFaultModeState=3` (shutdown) | `g_pendingSystemCmd=1` only if slot 0xFF, **then always falls through to the factory reset** (every pack that handles the frame in this branch, i.e. every non-master pack, is factory-reset) (see `VNSD_Unclear_Registers_Resolution_2026-10-01.md`) | shell `vns_ate_test@0x08057000`; `Idle_Watchdog_AutoReboot@0x08032c80` (×2 before Control reboot) |
| 2 | `APP_Factory_Reset_And_Reboot` | same | **Modbus 45028**; `FactoryReset_NotifyAndReboot(1)@0x0800c11e` <- `Factory_Reset(1)` <- BLE 0x06 val 1, MQTT `cd=5,rs=1`, JSON-RPC/UDP `Reset.Factory {type:1}`, shell `restore_factory 1` |
| 3 | `g_relayCommandPendingFlag=1` (encode handshake; expires after 300 ticks unless the PB13 pulse pattern arrives, `PackSelect_RelayCommStateMachine@0x0800b304`) | same | `BLE_Apply_ConfigReg_Offset3(0)@0x08032ab8` <- BLE `0x40 [00]`, MQTT `cd=40,number_as` (= app button "Battery Pack Recovery"); `Board_ProtectionStage_Dispatch@0x08021e78` (+PB10 pulse train) <- Modbus 45029 / power key |
| 4 | `g_packFaultModeState=4` -> `PackMode_Dispatcher@0x080097e4` case 4: CAN broadcast (class 3, fn 2, node 0xF, payload 1), after 100 ticks `g_packRotationCommitPendingFlag=1` + **system reset** = BMS restart | – (ignored) | `BLE_Apply_ConfigReg_Offset3(1)` <- BLE `0x40 [01]`, MQTT `cd=40,bms_rst`; dev mode 4: BLE `0x0C 0A 0B 0C 04`, MQTT `cd=7,dm=4`; `Config_Graduated_Sync_Retry@0x0802e680`; Modbus 45031 only if PGN1804 b4-5 == 0xAA (never on D) |
| 7 | `g_packHandoffRequestFlag=1` | – | `Grid_Export_Limit_Periodic_StateMachine@0x0800acac` every 1 s while PGN 0x1803 byte 7 bit 1 is set (automatic) |
| 8 | if locked: `APP_SetErrorLockFlag(0)` + `g_packFaultModeState=4` (= unlock **and** restart) | if locked: `APP_SetErrorLockFlag(0)` | `Config_Apply_WorkModeReg(0)@0x08022280` <- BLE `0x43 [01 00]`, MQTT `cd=44,lock=0`, shell `lck_bms 0` |
| 9 | `APP_SetErrorLockFlag(1)` (EEPROM 0xFC00 = 0x86, MOSFETs blocked) | – | `Config_Apply_WorkModeReg(1)` <- BLE `0x43 [01 01]`, MQTT `cd=44,lock=1`, shell `lck_bms 1` |
| 0x0A | `Config_Graduated_Sync_Retry` (automatic) | – | – |
| 0x0B/0x0C/0xA0 | no case | no case | Modbus 45030 / 45031 (FUNCTIONLESS) |

### 1.2 Other BMS-directed frames

| Frame | BMS effect | Control sender / trigger |
|---|---|---|
| class 3 fn 0x0A node 1 (`0x0001310A`) | master: `CAN_Set_Notify_Flag_And_Send` -> (EEPROM 0x202=0xFF if runtime master) + immediate system reset | **Modbus 45027** (`CAN_SendTriggerCategory0x0A(1)@0x08032934`) only |
| `0x100401AA [00 04]` | `CAN_CMD_04_Handler@0x08007ea4` case 4: if flags bit0 and no pack fault -> SOC override, active pack = balancer's low pack | `Inverter_PVString_ChannelValue_PeriodicCheck_Save@0x080173b4`, every 500 ms while PGN 0x1803 b7 bit0 and (any PV > 14.0 V or grid permit), automatic |
| `0x100201AA [soc]` | `CAN_CMD_02_Handler` -> DoD (0x200028E0) | shell `change_soc` (`Config_Write_TargetSOC_Percent@0x0803180c`) only |
| `0x1801FFAA` | byte 2 = discharge-cutoff/DoD, byte 11 = enable (master startup) | `Grid_BuildStatusFlagsReg@0x08031888`, periodic |
| class 3 0xC4 node 0 | `g_pendingActivationFlag` (= CLI `g_u8ProhibitCommFlag`, silences CAN during OTA). Not an activation | `OTA_Upgrade_Timeout_Watchdog`, `System_ResetToDefaults` |
| class 3 0xCE | BMS OTA flash writer | OTA (`cd=53/54`, `Ota.Set`) |
| class 3 0xC1/0xC2/0xCF, `0x100601AA`, `0x180701AA` | no BMS handler | 45001, 45011, shell `debug_mode`, EPS hysteresis |
| CAN id field `0x210100` | BMS ISR: RAM error lock + outputs off | **no sender found** in Control 151 / Micro 116 |

JSON-RPC (`MQTT_JSON_RPC_Dispatcher@0x0801f4f8`, also used by the UDP local API via
`Cloud_PassiveMode_UDP_RecvAndDispatch@0x08015888`): methods `Marstek.GetDevice/GetData, Wifi/BLE/Bat/PV/ES/EM.GetStatus,
ES.SetMode/GetMode, DOD.SET, Ble.Adv, Led.Ctrl, Ota.Set, Set.Ver, Reset.Factory`. Only `Reset.Factory {type:1}` reaches the
BMS (byte 2). `DOD.SET` is Control-side cutoff 30–88 % (+`Config_Param0A_Relay`).

All of §1 is identical in Control 147, 149, 149.2, 150 and 151: same `Config_Apply_WorkModeReg` 8/9, `BLE_Apply_ConfigReg_Offset3`
3/4, MQTT cases 0x28/0x2C, BLE '@'/'C'/0x0C-case-4, `FactoryReset_NotifyAndReboot` byte 2, Grid_Export byte 7. BMS 117.7
`CAN_CMD_01_Handler` and `PackMode_Dispatcher` are textually identical to 118.

## 2. Firmware logic that really is "pack recovery / reactivation" (all automatic)

1. **Recovery charge, master BMS** — `Master_RunState_Dispatcher@0x080066e0`: at start-up (`Master_RunState_InitSelect@0x08006b94`)
   or in run state 2 (`PackRecovery_Request_Check@0x080075c4`), a pack with Protect1 bit 0 (cell undervoltage, min cell
   < 2500 mV / 2300 mV cold) sets `g_packAnyFaultFlag`. `PackRecovery_LowCell_Charge_SM@0x0801070c` then: all Dsg-MOS off ->
   lowest-cell pack charges alone -> after 30 ticks at ≥ 1.1 A also Dsg-MOS on -> done at min cell > 3100 mV and SOC > 13–14 %.
   **If < 1.1 A for 3000 ticks (~300 s), state 7 -> pack mode 3 (shutdown)**. While it runs, PF3 forces charge limit 25.0 A
   and discharge limit 0 (`CAN_TX_PF3_ChargeDischargeLimits@0x08007cd8` via `APP_Get_PackFaultStatus`).
   This needs `g_highestPackIndex != 0` (`Pack_Get_Protect1Bit0_Mask@0x08009534`), i.e. a multi-pack stack. Behaviour with a single pack is not proven.
2. **Low-voltage balancer + Control forced charge** — `Pack_LowVoltage_Balancer@0x08007f24` flags packs with SOC < 8 % or
   Protect1 bit 0 and keeps them until > 12 % and UV cleared. PF3 byte 7 bit 0 -> Control `g_pgn1803InfoForceChrg` bit 0 ->
   `Battery_Charge_Power_Limiter@0x08008bfc` forces the setpoint to charge at −(V × BMS charge-current limit)
   (later clamped by max charge power etc. in `Inverter_Power_Setpoint_Calc@0x0800a968`), and
   `Inverter_PVString_ChannelValue_PeriodicCheck_Save` sends `0x100401AA [00 04]` (switch to the low pack).
3. **Handoff gate** — PF3 byte 7 bit 1 (master run state 2 and aggregation not done) -> `Grid_Export_Limit_Periodic_StateMachine`
   (event 600, byte 7 every 1 s, charge setpoint 0).
4. **Error-lock clear** (remote only, §1.1 byte 8): the only remote command that turns a pack that can neither charge
   nor discharge back on (lock blocks `OwnPack_MOSFET_Driver*`, `PackSelect_Tick`, `Master_Startup_WaitForEnable`).
5. **Restart** (byte 4 / class-3 0x0A / BMS key PB13 long press 10 s -> mode 4).

"The device must be charged to activate it" matches mechanisms 1 and 2. The BMS wakes and recovers only when charge
current flows. This is also the classic ship-mode/deep-sleep wake condition (`System_Reset_And_Halt_WithCurrentCheck`
-> STOP mode, EXTI wake). No remote command drives it.

## 3. Candidates for what the app button sends (resolved)

| # | Command | BMS effect on D | Evidence for | Evidence against | Result |
|---|---|---|---|---|---|
| 1 | BLE `0x43 [01 00]` / MQTT `cd=44,lock=0` (byte 8) | unlock + master restart | only remote "make it work again" action; the restart fits "reactivate"; app 1.6.76 newly contains a bare `cd=44` string | neither app build has a `lock=` literal; the unlock does nothing for an undervolted pack | **refuted** (the app never sends it; no `lock=` in the app) |
| 2 | BLE `0x40 [01]` / MQTT `cd=40,bms_rst` (byte 4) | master BMS restart (slaves ignore byte 4) | "reactivate"; both app builds contain `number_as`, the sibling parameter of the same handler | no `bms_rst` string in either app build | **refuted** (the app never sends it; no `bms_rst` in the app) |
| 3 | BLE `0x40 [00]` / MQTT `cd=40,number_as` (byte 3) | arms the encode handshake only; probably NO_OP without the PB10 pulse train | `number_as` string present in both app builds | is pack addressing, not recovery | **PROVEN: this is what the app sends** (BLE `73 06 23 40 00 16`, MQTT `cd=40,number_as`, §7) |
| 4 | BLE `0x0C 0A 0B 0C 04` / `cd=7,dm=4` (byte 4) | same as #2 | – | developer-mode command, unlikely behind a user menu item | **refuted** (the app never sends it) |
| 5 | none / a newer `cd=` code | nothing on v151 | the D manual has no "Battery Pack Recovery" entry; new app codes 67/72/90/91 are unhandled | – | **refuted** (the button uses the existing code 0x40 / `cd=40`; the new codes belong to other functions, §6) |

PROVEN: everything in §1/§2 (code) and the app side in §7 (disassembly). The app wording "recover/reactivate" is marketing for BMS address
coding (internal names `handleBmsCoding`, `setBMSCodeFunction`, `icon_bms_coding`/`btn_bms_coding`, `check_get_bms_coding`, payload `number_as`).
The earlier remark on app-string ordering stays valid: the order of the strings is **not** evidence. In
both builds, `RepairSignPage` / `recovery current type` / the German "aufgeladen" text / `cd33=` sit next to the same
unrelated strings (Romanian i18n, `ECDHKeyDerivator`). That is a global string-pool order, not code locality. And
`cd=33` (0x21) has **no case** in any Control version (0x20–0x27 absent), so `cd33=` belongs to another product.
`RepairSignPage` comes from `pages/Common/repair/` (warranty repair), not battery recovery.

## 4. Modbus mapping

| Register | Relation | Status on D |
|---|---|---|
| 45027 | class-3 0x0A -> master BMS reboot (same effect class as candidate #2) | FUNCTIONAL, danger high, do not expose |
| 45028 | byte 2 -> all packs factory reset + reboot | FUNCTIONAL, danger high, never expose |
| 45029 | byte 3 + PB10 pulses -> encode / master claim; **Modbus-side equivalent of the app button** | CONDITIONAL |
| 45031 | would send byte 4 (restart) only after a 0xAA ack that BMS 117.7/118 never send; sends 0xA0 (unhandled), ~5 s stall, exc 3 | FUNCTIONLESS |
| 45030 | bytes 0x0B/0x0C, no handler | FUNCTIONLESS |
| 32112 | read: BMS error lock (PF4 b7) — the state that candidate #1 clears | FUNCTIONAL (read) |
| 32106/32107 (35111/35112) | read: BMS charge/discharge current limit; 250 / 0 = recovery SM active | FUNCTIONAL (read) |
| 42010 (+42000/42020) | RS485 force charge: can provide the ≥ 1.1 A that the recovery SM needs | FUNCTIONAL, indirect |
| 41200, 32112 write | no path to bytes 8/9/4 | – |

**45029 is the Modbus-side equivalent of the app button.** Like the button it sends byte 3, but it additionally sends the PB10 pulse train
that the handshake needs. The BLE/MQTT path of the app sends no pulse train, so on its own it probably times out (INFERRED, §7.3).
The Control-internal test reports success only with exactly two packs online (status in 32114, otherwise 3).

**No Modbus register sends byte 8 (unlock), byte 9 (lock) or byte 4 (restart) on D.** Write_Handler calls
`Config_Apply_SingleReg` only with 2, 4/0xA0, 0x0B, 0x0C, and `CAN_SendTriggerCategory0x0A(1)`. PF3 byte 7 (force flags)
is not mapped to any Modbus register (MQTT `force=`, shell `get_main_bms_data` only).

Suite: add 32112 as diagnostic binary_sensor "BMS error lock" and 32107 as diagnostic. Do **not** add a "pack recovery"
button: there is no safe Modbus equivalent, and 45027/45028 are destructive.

## 5. Optional confirmation

A BLE capture is no longer needed to identify the command (§7). If wanted as a cross-check: log the GATT writes while pressing
"Battery Pack Recovery". The capture shows the frame `73 06 23 40 00 16` and the reply `73 .. 23 40 ..` (HM frame `73 <len> 23 <cmd> <payload…> <xor>`).
Over Wi-Fi/cloud the same press shows `cd=40,number_as` and the reply `cd=40,ret=1`.

## 6. App 1.6.76 `cd=` codes vs. the v151 dispatcher

The v151 `MQTT_CtrlCommand_Dispatcher` handles decimal 0–31, 40–45, 50–59, 61, 63–66. 147 lacks 61/63–66; 149/149.2 have 61/62
(0x3d/0x3e). 150 has 61, 63–65.

| New app string | v151 handler | Meaning in app 1.6.76 (proven by decompilation) |
|---|---|---|
| `cd=12,op=0/1` (`http=<url>`) | 0x0C = cloud server type, parses only `ht=0..4`; `op=` ignored, replies result 0 -> **no handler for op** | A3600CE EV charger: server URL get/set (`ev_charger_connector.dart`, own topic `marstek_energy/A3600CE-A0/App/<encMac>/ctrl`, not the Venus) |
| `cd=16` | 0x10 max charge power (`cp`); handled (with `cp=`) | `cd=16,cp=` = `setAcCouplerMaxChargePower` (Venus); bare `cd=16` also CT bind query and HMH `cd=16,mod=` |
| `cd=18,ct_t=` / `meter=` | 0x12 -> `Cloud_HTTP_Response_Parser@0x08029700` knows `meter=`, `ct_dev=`, `dchrg=`, `mac=`, `shelly_port=`, `ctrl_rat*`, `rechrg_seq`; `meter=` ok, **`ct_t=` unhandled** | not analysed in the app decompilation |
| `cd=20,md=3,no=` | 0x14 = FC41D URL OTA (`len`,`url=`); **no matching handler** | not analysed in the app decompilation |
| `cd=30,data=` | 0x1E = local API (`api`,`port`); **`data=` unhandled** | not analysed in the app decompilation |
| `cd=44` / `cd=45` | 0x2C lock bms (`lock=`), handled only with `lock=`; 0x2D get mppt data, handled | V6000 power station FC41D module (`cd=44` version query, `cd=45,fc41d_url=` update); `cd=44,do=` / `cd=45,cv=` DoD / charge voltage of other products (BigPower, new-module devices `CMD_SET_DOD`); HMH `cd=45` = provider data. Not the Venus `lock=` command |
| `cd=46` / `cd=47,<k>=<v>` | none (46–49 absent in all versions) -> **no handler** | A3600CE EV charger: status query / charge control (own topic, not the Venus) |
| `cd=51,url=` | 0x33 = get OTA state (ignores params); **`url=` unhandled** | A3600CE EV charger: trigger OTA upgrade (own topic, not the Venus) |
| `cd=63,ct_chg_type=`, `cd=64,ups_p=`, `cd=65,lapi_en=,self_port=,remote_port=` | 0x40 = publish device IDs and 0x41 = get INV data ignore params -> **no setter**; v151 has other meanings for 63–65 | new-module Venus products (Venus X / E-mini): CT charge type, UPS charge power, local API settings |
| `cd=67` | none (max case 0x42 = 66) -> **no handler in any of 147–151** | `CMD_GET_SCAN_WIFI_LIST`: Wi-Fi scan list of newer Venus models |
| `cd=72,type?` / `cd=72,type=1,len=<n>,ip=<ip>,mac=<mac>` | none -> **no handler in any of 147–151** (BLE fallback cmd 0x61, also unhandled, v151 BLE max 0x5B) | binds an EV charger to the Venus (EV charger IP/MAC; `queryMacAndIp` / `setMacAndIp`, Venus topic). Needs newer firmware, unrelated to the pack |
| `cd=90`, `cd=91,type=<t>,…`, `cd=68` | none -> **no handler in any of 147–151** | HMH family (other product): Shelly scan / connect, system check |

BLE side: the v151 BLE dispatcher handles 0x02–0x06, 0x08–0x29, 0x40–0x43, 0x50–0x55, 0x59–0x5B. There is no extra
recovery id. JSON-RPC has the 16 methods listed in §1.2 (no recovery method). No A/E3 Control images are in this
tree to check.

## 7. App side (proven by decompilation)

Source: app `com.hamedata.marstek` 1.6.76 (Dart package `cross_power_x`, `libapp.so` arm64), decompiled with Blutter (Dart AOT decompiler).
Static analysis only: no device, no cloud, no login, nothing was triggered. The feature already exists in 1.6.72 (same strings).

### 7.1 Call chain (PROVEN from disassembly)

1. **Settings row** (`AccouplerSettingState::createContent`): built only if `isStack()`. That is true for the Venus types VNSD, VNSA, VNSACH, VEPRO,
   VEPROCH, VNSDCH, VNSD2, VEPRO2, VNSA2 and for VEPAC. The row is hidden for Venus E/C, E3, E4 and EMAX. No firmware-version check was found.
   Title "Battery Pack Recovery" (`tv_battery_recovery`), icon `icon_bms_coding`, button id `btn_bms_coding`.
2. **Tap:** requires device owner/admin (`hasAdminOfDevice`) and a connected device (`isDeviceConnected`), then a confirm dialog
   ("Preparing to recover the battery pack. Do you want to continue?").
3. **Confirm:** `VenusHelper::handleBmsCoding` -> `CommonCommand::setBMSCodeFunction`, wrapped in `retryOnTimeoutWithNoTag`: up to 3 sends with a 5 s timeout each.
4. **Transport:** if the current type is empty, nothing is sent. With a BLE link: frame `73 06 23 40 00 16` (Venus; for Jupiter HMM/HMN/JPLS the frame is
   `73 06 23 23 00 75` / `cd=20,number_as`). Otherwise, if the cloud is online: MQTT topic `hame_energy/<type>/App/<id>/ctrl`, payload `cd=40,number_as`.
   No parameters (no current type, temperature, version, MAC or options).
5. **Reply:** BLE: any answer frame with cmd 0x40 and length >= 5 counts as success (the echoed byte is not checked). MQTT: `cd=40,ret=1` = success.
   The UI then shows "Recovery successful" or "Recovery failed". No answer after 3 x 5 s = "Recovery failed".
6. **No follow-up command:** no status poll, no charge command, no second step.
7. **Firmware:** `MQTT_CtrlCommand_Dispatcher` case 0x28 (`number_as`) or `BLE_App_Command_Dispatcher` case 0x40 -> `BLE_Apply_ConfigReg_Offset3(0)` ->
   `Config_Apply_SingleReg(3)` -> CAN `0x100101AA` byte 3 (§1.1). The handler is identical in Control 147/149/149.2/150/151; no newer firmware is needed.
   A `bms_rst` payload (byte 4, restart) would exist in the handler, but the app never sends it.

### 7.2 Strings that turned out to be unrelated (PROVEN by location)

| String | Real use |
|---|---|
| `recovery current type/mac/version/temp` | debug logs of the CT pairing page (`SelectCtPageState::getSelectCtList`); they log and restore the device context after connecting to a CT. Nothing to do with the battery |
| `recovery_server` / `recoveryMqttSSL` | "Reset Server Connection" (Advanced functions, `FunctionCode.recoveryMqttSSL`) |
| "The device must be charged before activation." | B2500 OTA tip (`OTA_CheckB2500SuccessfulTip`) |
| `/ems/api/v1/setApplyVenusActivate` | `activateUpgradePower(devid, code)`: power-class activation code, not the battery |
| "Upgrade … recovery" texts | OTA-failure hints |

### 7.3 Risk assessment

* PROVEN: the button sends only cmd 0x40 `[00]` / `cd=40,number_as`. It has no parameters, `bms_rst` and `lock=` do not occur in the app, and no follow-up command exists.
* PROVEN: in the firmware this is only byte 3 of `0x100101AA` (arms `g_relayCommandPendingFlag`, expires after 300 ticks without the PB13 pulse pattern).
* INFERRED: the command does not start a forced charge and does not reset or restart the BMS. The only effect is "start pack numbering" on the CAN bus.
  Without the PB10 pulse train (which only Modbus 45029 / the power key add) the handshake probably times out and nothing changes.
* INFERRED: the meaning of `hasAdminOfDevice` / `isDeviceConnected` / `isVirtualExp` comes from the function names only (bodies not traced).

## 8. Open questions

1. Does the master recovery SM / balancer run with a single pack (`g_highestPackIndex == 0`)? Not proven.
2. Who sends CAN id field `0x210100` (BMS RAM lock + outputs off)? Not in Control 151 / Micro 116.
3. Side finding (already in the register resolution): `Idle_Watchdog_AutoReboot` sends byte 1 twice. On every non-master pack this falls through to
   the BMS factory reset.
