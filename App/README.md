# App – Marstek smartphone app analysis

> 🇩🇪 Deutsche Version: [README_de.md](README_de.md)

## What the app is

The Marstek app (`com.hamedata.marstek`, Dart package `cross_power_x`) is a Flutter app. Its logic lives in the Dart AOT snapshot `libapp.so`, not in Java/Kotlin code. One app serves many products: B2500 (HMA–HMJ), Venus, Jupiter, V6000/M5000, CT, HMH and the EV charger A3600CE. It talks to devices over two transports plus a cloud REST API:

- **BLE**: HM frame `73 LEN 23 CMD payload XOR`.
- **MQTT**: topic `hame_energy/<type>/App/<id>/ctrl`, payload `cd=<n>,k=v`.
- **Cloud REST**: account, device list, OTA queries.

The `cd` numbering is per product. Current analysed version: **1.6.76** (1.6.72 compared).

## Contents

| File | Description |
|---|---|
| [VNSD_Hamedata_App_API_Recon.md](VNSD_Hamedata_App_API_Recon.md) | Cloud API reconnaissance of the app, used to obtain the communication module OTA image |
| [Marstek_App_Decompilation_Method.md](Marstek_App_Decompilation_Method.md) | How the Dart AOT code was analysed (Blutter), reading tips, worked example "Battery Pack Recovery" |
| [Marstek_App_Command_Vocabulary_1.6.76.md](Marstek_App_Command_Vocabulary_1.6.76.md) | Command vocabulary of app 1.6.76 (BLE commands and MQTT `cd=` codes per product) |
| [Marstek_App_vs_Firmware_Command_Map.md](Marstek_App_vs_Firmware_Command_Map.md) | Mapping of the app commands to the Control firmware dispatchers |
| [Marstek_Cloud_MQTT_B2500_Vendor_Doc.md](Marstek_Cloud_MQTT_B2500_Vendor_Doc.md) | Summary of the vendor's own MQTT protocol page (B2500 family, versions 212 to 220): telemetry fields and `cd=` commands |

## Related documents

- [../BLE/VNSD_BLE_Command_Map_v150.md](../BLE/VNSD_BLE_Command_Map_v150.md): BLE command map of the Control firmware (firmware side of the BLE frames).
- [../BMS/VNSD_Battery_Pack_Recovery_Firmware.md](../BMS/VNSD_Battery_Pack_Recovery_Firmware.md): Battery Pack Recovery; the app side is section 7.
- Capture guides in [../HM_HIE_FC41D/](../HM_HIE_FC41D/): [Raspberry Pi 3B+](../HM_HIE_FC41D/RPi3Bplus_Marstek_Capture_Guide.md), [laptop access point](../HM_HIE_FC41D/Marstek_Traffic_Capture_Laptop_AP.md), [BLE module update](../HM_HIE_FC41D/VNSD_BLE_Comm_Module_Update_Capture_Guide.md).
