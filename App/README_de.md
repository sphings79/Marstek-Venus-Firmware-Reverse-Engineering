# App – Analyse der Marstek-Smartphone-App

> 🇬🇧 English version: [README.md](README.md)

## Was die App ist

Die Marstek-App (`com.hamedata.marstek`, Dart-Paket `cross_power_x`) ist eine Flutter-App. Ihre Logik steckt im Dart-AOT-Snapshot `libapp.so`, nicht in Java-/Kotlin-Code. Eine App bedient viele Produkte: B2500 (HMA–HMJ), Venus, Jupiter, V6000/M5000, CT, HMH und die Wallbox A3600CE. Sie spricht mit den Geräten über zwei Transporte plus eine Cloud-REST-API:

- **BLE**: HM-Frame `73 LEN 23 CMD payload XOR`.
- **MQTT**: Topic `hame_energy/<type>/App/<id>/ctrl`, Payload `cd=<n>,k=v`.
- **Cloud-REST**: Konto, Geräteliste, OTA-Abfragen.

Die `cd`-Nummerierung gilt pro Produkt. Aktuell analysierte Version: **1.6.76** (1.6.72 verglichen).

## Inhalt

| Datei | Beschreibung |
|---|---|
| [VNSD_Hamedata_App_API_Recon_de.md](VNSD_Hamedata_App_API_Recon_de.md) | Cloud-API-Recherche der App, genutzt um das OTA-Image des Kommunikationsmoduls zu beschaffen |
| [Marstek_App_Dekompilierung_Methode_de.md](Marstek_App_Dekompilierung_Methode_de.md) | Wie der Dart-AOT-Code analysiert wurde (Blutter), Lesetipps, durchgespieltes Beispiel „Akkupack-Wiederherstellung“ |
| [Marstek_App_Befehlsvokabular_1.6.76_de.md](Marstek_App_Befehlsvokabular_1.6.76_de.md) | Befehlsvokabular der App 1.6.76 (BLE-Befehle und MQTT-`cd=`-Codes je Produkt) |
| [Marstek_App_vs_Firmware_Befehlszuordnung_de.md](Marstek_App_vs_Firmware_Befehlszuordnung_de.md) | Zuordnung der App-Befehle zu den Dispatchern der Control-Firmware |

## Verwandte Dokumente

- [../BLE/VNSD_BLE_Command_Map_v150_de.md](../BLE/VNSD_BLE_Command_Map_v150_de.md): BLE-Befehlsabbildung der Control-Firmware (Firmware-Seite der BLE-Frames).
- [../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md](../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md): Akkupack-Wiederherstellung; die App-Seite steht in Abschnitt 7.
- Mitschnitt-Anleitungen in [../HM_HIE_FC41D/](../HM_HIE_FC41D/): [Raspberry Pi 3B+](../HM_HIE_FC41D/RPi3Bplus_Marstek_Capture_Anleitung_de.md), [Laptop-Access-Point](../HM_HIE_FC41D/Marstek_Traffic_Capture_Laptop_AP_de.md), [BLE-Modul-Update](../HM_HIE_FC41D/VNSD_BLE_Comm_Modul_Update_Mitschnitt_Anleitung_de.md).
