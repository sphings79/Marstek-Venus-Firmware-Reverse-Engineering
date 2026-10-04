# VNSD FC41D – Diff Firmware 202409090159 (A04) ↔ 202512040647 (A05)

> 🇬🇧 English version: [VNSD_FC41D_Diff_202409_vs_202512_de.md](VNSD_FC41D_Diff_202409_vs_202512_de.md)

Basis: Ghidra-Versionstracking (`/VNSD/FC41D/…`) + String-Diff beider entschlüsselter App-Images
(`fc41d_decrypted/`, lokal, nicht im Repo). Beide Images: Quectel FC41D / Beken BK7231N, Image-Base 0x00010000.

| | 202409090159 | 202512040647 |
|---|---|---|
| Build-Name | `FC41DAAR18A04M02_HM` | `FC41DAAR18A05M02_HM` |
| Build-Zeit | Sep 9 2024 01:59 | Dec 4 2025 06:47 |
| Funktionen (Ghidra) | 5915 | 5895 |
| VT-Korrelation | 3653 identisch, 180 geändert | 2071 nur alt / 2051 nur neu (Adressverschiebung, daher geringe Aussagekraft) |

## 1. Entfernt in 202512

| Was | Details |
|---|---|
| **AirKiss** (WeChat-Smartconfig) | komplette `airkiss_*`-Routinen + Lib `airkiss-2.0.0-25360`, AT+QAIRKISS, URC `+QAIRKISS:SSID=…,PSK=…` |
| **Web-Konfig-Seite** (SoftAP-Webserver) | AT+QWEBCFG, `192.168.19.1`, HTML-Formular „FC41D smart config" (SSID, Passwort, UART-Baudrate), `/index.htm`, `/not_connect.html`, `/connect_succes.html` – Webserver/Captive-Portal ist raus |
| Flash-Init-Debugausgaben | `[Flash]id:…`, `write status reg`, `choose default config` |
| diverse Debug-Strings | `sctrl_sta_ps_init`, `bk_misc_init_start_type`, `+QSTAST:STATION_DOWN` |

→ Ab 202512 kann das Modul **nicht mehr** per Smartconfig/Web-Seite konfiguriert werden; nur noch AT-Kommandos / Control über UART.

## 2. Neu in 202512

| Was | Details |
|---|---|
| **Signierte FOTA** | `QLSECFOTA!!!`-Header, `ql_secfota_*` (move_file, analys_file, verify_ecdsa384_signature, calculate_file_sha384), eingebetteter **ECDSA-P384-Public-Key** (PEM @0x000ec8c2), „FOTA Signature verification succeeded/failed". Alte, unsignierte RBL-Pakete werden damit abgelehnt. |
| Neue AT-Kommandos (Quectel-Standard) | `AT+QREDDAVN`, `AT+QAPCFG="linknum"`, `AT+QSECCFG="log_ctl"`, `AT+QSTAINFORAW`, `AT+QBLEMTUMODE`, `AT+QHTTPGET/PUT`-Timeout-Range (10–65535 statt 60–65535), `AT+QSSLCERT` mit Länge+Checksumme |
| SSL-HAL | `HAL_SSL_Establish/Read/Write` (mbedTLS-Client), `ql_socket_ssl_session_setup`, `ql_cert_data_is_exist/init` |
| BLE | `app_sec.c` (GAPC bond/pairing/LTK/TK-Handler), `app_ble_set_le_pkt_size`, MTU-Mode-Konfiguration, erweiterte `+QBLEPEERINFO`/`QL BLE:…mtu` Ausgaben |
| SNTP | Timer-Fehlerbehandlung, Server `pool.ntp.org`, `ntp1.aliyun.com`, `210.72.145.44` (Fallback-Liste) |
| Persistente User-Config | `usr_config_read/write_id_flash`, `usr_config_search_tab`, `save_user_info_tab_to_flash` (@0x00076228), Flash-Partitionen 7/8/9: TLV-Tabelle (Baudrate Default 115200 = 0x1c200, SSID, PSK **AES-verschlüsselt**, Flags), CRC/Checksumme, **Backup-Kopie**, „redflag"-Marker |
| Dev-Debug (`lyy debug`, `[cx]`) | Kürzel der Quectel/Hame-Entwickler, in 202512 deutlich mehr Stellen (SSID-Repeat-Check, `g_my_ssid`, `g_connect_key`, `uart_band`, `sleep_mode`, `flash_len`, `calcuCrc`) |
| Sonstiges | `AT+HMIOPEN`-Usage-Text zeigt jetzt `AT+HMIOPEN=1/0,ip,remote,local`; Shell-Kommandos `echo`, `memdump`, `memp`; MQTT-Logging erweitert |

## 3. Unverändert (Hamedata-Anpassungen)
`AT+HMIOPEN` / `hame_tcp` / `hame_tcp_connect_server_command`, `connect_key`, Default-WLAN `hellow` / `12345678`,
UART-Reinit + GPIO6/7-Toggle sind in **beiden** Versionen vorhanden (siehe `VNSD_FC41D_Hamedata_Anpassungen_de.md`).

## 4. Bewertung
* Sicherheitsrelevant: 202512 = **Einbahnstraße** (ECDSA-Signaturprüfung, kein Downgrade auf A04 via QWLANOTA).
* Wegfall von Web-Config/AirKiss verkleinert die Angriffsfläche des Moduls (kein lokaler HTTP-Server mehr).
* Zukünftige Images bleiben entschlüsselbar (gleicher AES-RBL-Wrapper, Signatur nur umhüllend) – siehe `security/` (lokal, nicht im Repo).
