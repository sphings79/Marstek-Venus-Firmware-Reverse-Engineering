# VNSD FC41D – Software-Stack des Moduls (aus Ghidra-Analyse 202512040647 / 202409090159)

> 🇬🇧 English version: [VNSD_FC41D_Software_Stack_de.md](VNSD_FC41D_Software_Stack_de.md)

Quectel FC41D (Beken BK7231N, ARM968E-S). Es gibt keine Symbole; alle Namen stammen aus Log-/Assert-Strings,
Konstanten (z. B. mbedTLS-Fehlercodes), Struct-Layouts und Aufrufbeziehungen. Zweifelhafte Namen tragen im
Plate-Kommentar „(medium confidence)".

| Schicht | Inhalt |
|---|---|
| Beken SDK `beken378` | WLAN-MAC (`rwnx_*`, `txl_*`, `nxmac_*`), RF-Kalibrierung, Flash-/UART-/GPIO-Treiber, `ddev_open`, `bk_flash_*` |
| RTOS | FreeRTOS (`xQueueGenericSend/Receive`, `xTaskCreate`, `prvTimerTask`, `uxTaskGetSystemState`, Heap-Implementierung) |
| IP-Stack | lwIP (`tcp_input/process/write/output`, `udp_input`, `ip_input`, `dhcp_*`, `dns_recv`, `lwip_select/recvfrom/accept/getaddrinfo`) |
| TLS | mbedTLS 2.x (`mbedtls_ssl_*`, `x509_crt_*`, `rsa`, `gcm`, `sha1/256/512`, `ctr_drbg`) |
| WLAN-Sicherheit | hostapd + wpa_supplicant (WPA2/SAE) mit **wolfSSL/TomsFastMath** als Bignum-/ECC-Basis (0x46218–0x4a5ce: `fp_*`, `ecc_*`, `wc_ecc_*`) – getrennt von der mbedTLS-Kopie bei 0x4e000–0x53000 |
| BLE | Beken BLE-5 RivieraWaves-Stack (`lld/llc/llm/sch/hci/gapm/gapc/l2cc`, `app_sec.c`) – größtenteils nur über Assert-Dateinamen identifiziert |
| libc | newlib (`printf/scanf`-Kerne, `strtod`, `qsort`, `gmtime_r`, `mktime`) + libgcc Soft-Float (`__adddf3` …) |
| Quectel-Schicht | AT-Server (`atsvr_*`, AT-Tabelle), `ql_socket_*`, `ql_mqtt_client_work_thread`, `ql_httpc_work_thread`, `ql_secfota_*` (ECDSA-P384) |
| Hamedata | `hame_tcp_task` @0x000416ec (verbindet TCP, `+HMIOPEN SUCCESS/FAIL`, `lwip_select`/`recv`-Schleife → `hame_msg_handle`), AT+HMIOPEN @0x000419f8 |

Weitere App-nahe Funktionen: `dhcp_server_main` (0x33268, SoftAP-DHCP), `cli_main` (0x3bef0, UART-CLI),
`atsvr_handle_input` (0x757b8), `ql_socket_ssl_session_setup` (0x3f17c).

## Offene / unsichere Stellen
* RF-Kalibrierung 0x1dcf0, 0x1e2e8, 0x1fcb4: reine Register-Zugriffe, ohne Strings nicht benennbar.
* BLE-Link-Layer: überwiegend `FUN_*`, Assert-Strings liefern nur den Dateinamen.
* `0x77350` (Heap): von Durchgang 1 als `prvHeapInit` benannt, verhält sich aber wie malloc/realloc → `pvPortRealloc` (medium confidence).
* `mbedtls_ssl_derive_keys` @0x5a29c war nicht belegt → `ssl_derive_keys_wrapper`; die eigentliche Schlüsselableitung ist `ssl_derive_keys_core` @0x5bc48.
* Noch ca. 5000 kleine `FUN_*`-Funktionen (<200 Byte) unbenannt.
