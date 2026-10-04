# Marstek app: decompilation method (Dart AOT)

> 🇩🇪 Deutsche Version: [Marstek_App_Dekompilierung_Methode_de.md](Marstek_App_Dekompilierung_Methode_de.md)

How the logic of the Marstek app (`com.hamedata.marstek`, Dart package `cross_power_x`, version 1.6.76) was analysed. Static analysis only: no device, no cloud, no login, nothing is triggered. Overview of the app: [README.md](README.md).

## 1. Why strings alone mislead

The app is a Flutter app. All logic is in the Dart AOT snapshot `libapp.so`; there is no readable Java/Kotlin code. A plain `strings` search finds texts, but **the order of the global string pool is not code locality**: neighbouring strings do not belong to neighbouring code.

Example: searching for "recovery" found debug prints `ACC======recovery current type/mac/version/temp=====` next to other recovery strings. They belong to the CT/meter pairing page (`select_ct_page.dart`) and have nothing to do with the battery. Other false friends: `recovery_server` (l10n "Reset Server Connection"), `recoveryMqttSSL` (enum of the same item), `/ems/api/v1/setApplyVenusActivate` (power-class activation code). A string only counts as evidence once its **use site** (the function that references it) is known.

## 2. Obtaining the app

- The vendor offers no EU APK download, and third-party APK sites must not be used (unknown modifications).
- Pull the installed splits from your own phone:
  ```
  adb shell pm path com.hamedata.marstek
  adb pull <path printed above> .      # once per split
  ```
- `libapp.so` is in the ABI split `split_config.arm64_v8a.apk`, as `lib/arm64-v8a/libapp.so`. `libflutter.so` is in the same split.
- Analysed here: version 1.6.76 (Dart 3.8.1 stable, flags `product … arm64 android compressed-pointers`); 1.6.72 was compared.

## 3. Blutter

[Blutter](https://github.com/worawit/blutter) is an open-source decompiler/analyser for Dart AOT snapshots. It reads `libapp.so` and writes annotated assembly with resolved class, method and string names.

Requirements: cmake, ninja, pkg-config, capstone, a C++ compiler, git, Python 3 (venv with `pyelftools`, `requests`). The matching Dart SDK (3.8.1) is fetched automatically on the first run.

```
python3 blutter.py <dir with libapp.so and libflutter.so> <output dir>
```

Build and analysis take a few minutes. Outputs:

- `asm/`: annotated assembly per Dart library/class (about 530 MB).
- `pp.txt`: object pool (strings, constants, references).
- `objs.txt`: dumped objects (class names, enum values).

Including the Dart SDK checkout and build directories the whole set is about 840 MB; all of it can be deleted afterwards (the `dartsdk/` and `build/` folders are only needed for building).

## 4. Reading tips

- **Smi constants** show the raw tagged immediate, so the real value is **half**: `230` means 0x73, `70` means 0x23, `128` means 0x40.
- In Dart 3.8 compressed-pointer mode `true` = NULL+0x20 and `false` = NULL+0x30. Hence `tbnz w0,#4,X` jumps when the value is **false**, and `tbz w0,#4,X` jumps when it is **true**.
- Addresses printed by Blutter are offsets in `libapp.so`.
- Closures appear as `AnonymousClosure` with their own address; follow the address, not the name.
- Separate **proven** (traced in the disassembly) from **inferred** (function names only).

## 5. Following a feature

Typical chain for a settings function:

1. Settings row (`createContent` of the settings page state): visibility gate (device type), label string, `onTap` closure.
2. `onTap` closure: permission and connection checks, confirm dialog.
3. Confirm closure: callback bundle (start, result, error) and helper call.
4. Helper (e.g. `VenusHelper`): demo-device shortcut, wrapper with retry and timeout (`retryOnTimeoutWithNoTag`, 3 tries of 5 s).
5. `CommonCommand::<function>`: `_doOperate(bleFn, mqttFn)`. If the current device type is empty, nothing is sent; with a BLE link the BLE closure runs; otherwise, if MQTT is available, the MQTT closure.
6. BLE closure: builds the byte list `73 LEN 23 CMD payload`, appends the XOR checksum of all bytes (`calCrcValueListWithMutableList`), sends via `BleManager::sentOrderToDevice`. MQTT closure: builds topic `hame_energy/<type>/App/<id>/ctrl` and payload `cd=<n>,...`, publishes via `MqttUtil::publishMessage`.
7. Response parser: BLE notify closure (match on `frame[3]`) and `MqttAccoupler::analyseMqttData` (match on `cd`), then an event to the UI closure.

## 6. Mapping to the firmware

Take the command byte (BLE) or `cd` number (MQTT) and the payload from step 6, then look up the Control firmware handler:

- MQTT: `MQTT_CtrlCommand_Dispatcher`, case = `cd` number (e.g. `cd=40` is case 0x28).
- BLE: `BLE_App_Command_Dispatcher`, case = HM command byte (e.g. 0x40 is `'@'`).

Check that the firmware version really has a handler for that code; app and firmware numbering are per product and a newer app can send codes an older firmware ignores. BLE side of the firmware: [../BLE/VNSD_BLE_Command_Map_v150.md](../BLE/VNSD_BLE_Command_Map_v150.md); full app-to-firmware table: [Marstek_App_vs_Firmware_Command_Map.md](Marstek_App_vs_Firmware_Command_Map.md).

## 7. Worked example: "Battery Pack Recovery"

Question: what does the app button "Battery Pack Recovery" send? Result: **the BMS coding (pack address numbering) command, not a recovery charge.**

| Step | Finding |
|---|---|
| Settings row | `AccouplerSettingState::createContent` builds the row only if `isStack()` is true (Venus D/A/Pro and variants, VEPAC). Label `tv_battery_recovery`, icon/button `icon_bms_coding` / `btn_bms_coding`. |
| Tap | Closure checks admin rights and connection, shows the confirm dialog "Preparing to recover the battery pack. Do you want to continue?". |
| Confirm | Calls `VenusHelper::handleBmsCoding`, which wraps `CommonCommand::setBMSCodeFunction` in `retryOnTimeoutWithNoTag` (3 x 5 s). |
| BLE closure | List `[0x73, 0x06, 0x23]`, Venus appends `0x40`, payload `0x00`, XOR: **`73 06 23 40 00 16`** (Jupiter: `73 06 23 23 00 75`). |
| MQTT closure | Topic `hame_energy/<type>/App/<id>/ctrl`, payload **`cd=40,number_as`** (Jupiter: `cd=20,number_as`). |
| Response | BLE: any `0x40` answer of length 5 or more counts as success. MQTT: `cd=40,ret=1` is success. No answer after 3 x 5 s shows "Recovery failed". |
| Follow-up | None: no status poll, no charge command. |
| Firmware | MQTT: `MQTT_CtrlCommand_Dispatcher` case 0x28 with `strstr("number_as")`. BLE: `BLE_App_Command_Dispatcher` case `'@'`. Both end in `BLE_Apply_ConfigReg_Offset3(0)` and CAN byte 3 to the BMS (arm the pack-encode handshake). |

The strings "recovery/reactivate" are marketing wording; the internal names are `handleBmsCoding`, `setBMSCodeFunction`, `check_get_bms_coding` and the payload `number_as`. Details on the firmware and BMS side: [../BMS/VNSD_Battery_Pack_Recovery_Firmware.md](../BMS/VNSD_Battery_Pack_Recovery_Firmware.md), app side in section 7 there.
