# VNSD – Firmware-Fehler: Das URC für getrenntes WLAN wird nie erkannt (`WLAN DISCONNECTED` gegen `WLAN_DISCONNECTED`)

> 🇬🇧 English version: [VNSD_FW_Bug_WLAN_Disconnected_URC_2026-10-02.md](VNSD_FW_Bug_WLAN_Disconnected_URC_2026-10-02.md)

## Zusammenfassung

Der Control sucht den URC-Text `+QSTASTAT:WLAN DISCONNECTED` (mit **Leerzeichen**), das WLAN-Modul FC41D sendet aber `+QSTASTAT:WLAN_DISCONNECTED` (mit **Unterstrich**). Die Behandlung eines verlorenen WLAN-Links läuft für dieses URC deshalb nie.

## Belege

- Control `Quectel_URC_Dispatcher` (v151 `0x08015f60`): `strstr(buf, "+QSTASTAT:WLAN DISCONNECTED")`; dasselbe Literal steht in den Exporten von Control 147, 149, 149.2, 150 und 151 sowie im Venus-A-Control 1509.
- FC41D `wlan_event_handler` (202512 bei `0x00075404`): sendet `+QSTASTAT:WLAN_DISCONNECTED\r\n` (außerdem `WLAN_CONNECTED`, `GOT_IP`, `SCAN_NO_AP`, `AP_CONNECT`, `AP_DISCONNECT`); der String steht auch im entschlüsselten 202409-Image.
- Der Control-Zweig für einen verlorenen Link (Logzeile, `g_networkLinkUp = false`, `g_quectelInitStage = 2`, Ereignislog-Eintrag 4/0x195) wird in der Praxis nur über `+QSTASTAT:SCAN_NO_AP` erreicht.

## Auswirkung

Nach einem WLAN-Abriss kann der Control `g_networkLinkUp` bis zu einer späteren periodischen Statusprüfung gesetzt lassen (laut Venus-A-Analyse etwa 60 s; nicht live gemessen). Modbus 30300 (`wifi_connected`) kann in dieser Zeit 1 bleiben. Noch nicht geprüft: die Venus E3.

## Korrektur

Ein Zeichen im Control: gegen `WLAN_DISCONNECTED` vergleichen.
