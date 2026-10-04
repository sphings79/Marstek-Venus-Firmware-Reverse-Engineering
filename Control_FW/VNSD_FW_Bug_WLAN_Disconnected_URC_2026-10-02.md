# VNSD – Firmware bug: Wi-Fi disconnect URC is never matched (`WLAN DISCONNECTED` vs `WLAN_DISCONNECTED`)

> 🇩🇪 Deutsche Version: [VNSD_FW_Bug_WLAN_Disconnected_URC_2026-10-02_de.md](VNSD_FW_Bug_WLAN_Disconnected_URC_2026-10-02_de.md)

## Summary

The Control looks for the URC text `+QSTASTAT:WLAN DISCONNECTED` (with a **space**), but the FC41D Wi-Fi module sends `+QSTASTAT:WLAN_DISCONNECTED` (with an **underscore**). The handler that reacts to a lost Wi-Fi link therefore never runs for this URC.

## Evidence

- Control `Quectel_URC_Dispatcher` (v151 `0x08015f60`): `strstr(buf, "+QSTASTAT:WLAN DISCONNECTED")`; the same literal is present in the Control 147, 149, 149.2, 150 and 151 exports and in the Venus A Control 1509.
- FC41D `wlan_event_handler` (202512 at `0x00075404`): emits `+QSTASTAT:WLAN_DISCONNECTED\r\n` (also the other states `WLAN_CONNECTED`, `GOT_IP`, `SCAN_NO_AP`, `AP_CONNECT`, `AP_DISCONNECT`); the string is also present in the decrypted 202409 image.
- The Control branch that handles a lost link (log line, `g_networkLinkUp = false`, `g_quectelInitStage = 2`, event log entry 4/0x195) is reached only through `+QSTASTAT:SCAN_NO_AP` in practice.

## Effect

After a Wi-Fi drop the Control may keep `g_networkLinkUp` set until a later periodic state check (about 60 s according to the Venus A analysis; not measured live). Modbus 30300 (`wifi_connected`) can therefore stay 1 for that time. Not yet checked: the Venus E3.

## Fix

One character in the Control: compare with `WLAN_DISCONNECTED`.
