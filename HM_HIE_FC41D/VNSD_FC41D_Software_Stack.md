# VNSD FC41D – Software Stack of the Module (from Ghidra analysis 202512040647 / 202409090159)

> 🇩🇪 Deutsche Version: [VNSD_FC41D_Software_Stack_de.md](VNSD_FC41D_Software_Stack_de.md)

Quectel FC41D (Beken BK7231N, ARM968E-S). There are no symbols; all names come from log/assert strings,
constants (e.g. mbedTLS error codes), struct layouts and call relationships. Doubtful names carry
"(medium confidence)" in the plate comment.

| Layer | Content |
|---|---|
| Beken SDK `beken378` | WLAN MAC (`rwnx_*`, `txl_*`, `nxmac_*`), RF calibration, flash/UART/GPIO drivers, `ddev_open`, `bk_flash_*` |
| RTOS | FreeRTOS (`xQueueGenericSend/Receive`, `xTaskCreate`, `prvTimerTask`, `uxTaskGetSystemState`, heap implementation) |
| IP stack | lwIP (`tcp_input/process/write/output`, `udp_input`, `ip_input`, `dhcp_*`, `dns_recv`, `lwip_select/recvfrom/accept/getaddrinfo`) |
| TLS | mbedTLS 2.x (`mbedtls_ssl_*`, `x509_crt_*`, `rsa`, `gcm`, `sha1/256/512`, `ctr_drbg`) |
| WLAN security | hostapd + wpa_supplicant (WPA2/SAE) with **wolfSSL/TomsFastMath** as bignum/ECC basis (0x46218–0x4a5ce: `fp_*`, `ecc_*`, `wc_ecc_*`) – separate from the mbedTLS copy at 0x4e000–0x53000 |
| BLE | Beken BLE 5 RivieraWaves stack (`lld/llc/llm/sch/hci/gapm/gapc/l2cc`, `app_sec.c`) – mostly identified only via assert file names |
| libc | newlib (`printf/scanf` cores, `strtod`, `qsort`, `gmtime_r`, `mktime`) + libgcc soft-float (`__adddf3` …) |
| Quectel layer | AT server (`atsvr_*`, AT table), `ql_socket_*`, `ql_mqtt_client_work_thread`, `ql_httpc_work_thread`, `ql_secfota_*` (ECDSA-P384) |
| Hamedata | `hame_tcp_task` @0x000416ec (connects TCP, `+HMIOPEN SUCCESS/FAIL`, `lwip_select`/`recv` loop → `hame_msg_handle`), AT+HMIOPEN @0x000419f8 |

Further app-related functions: `dhcp_server_main` (0x33268, SoftAP DHCP), `cli_main` (0x3bef0, UART CLI),
`atsvr_handle_input` (0x757b8), `ql_socket_ssl_session_setup` (0x3f17c).

## Open / uncertain spots
* RF calibration 0x1dcf0, 0x1e2e8, 0x1fcb4: pure register accesses, cannot be named without strings.
* BLE link layer: mostly `FUN_*`, assert strings provide only the file name.
* `0x77350` (heap): named `prvHeapInit` in pass 1, but behaves like malloc/realloc → `pvPortRealloc` (medium confidence).
* `mbedtls_ssl_derive_keys` @0x5a29c was not evidenced → `ssl_derive_keys_wrapper`; the actual key derivation is `ssl_derive_keys_core` @0x5bc48.
* Still about 5000 small `FUN_*` functions (<200 bytes) unnamed.
