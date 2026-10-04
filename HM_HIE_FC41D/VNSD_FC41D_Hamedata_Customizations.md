# VNSD FC41D communication module – Hamedata/Marstek customizations

> 🇩🇪 Deutsche Version: [VNSD_FC41D_Hamedata_Anpassungen_de.md](VNSD_FC41D_Hamedata_Anpassungen_de.md)

**Status:** 2026-09-30.

**Firmware version:** 202512040647, build string `FC41DAAR18A05M02_HM.bin` (Quectel FC41D, Beken BK7231N, ARM968E-S)
**Source (decompilations):** `<project>/Dekompilate/FC41D_202512040647_app.bin/` (local, not in the repo)
**Raw data:** `<project>/HM_HIE_FC41D/fc41d_decrypted/fc41d_202512040647_app_decrypted.bin` (local, not in the repo)
**Comparison version:** 202409090159, build `FC41DAAR18A04M02_HM.bin` (string diff `fc41d_strings_diff.json`)

Methodology: offline analysis of the decompilations (Ghidra auto-analysis, 4666 functions), of the extracted strings with referencing functions, and of targeted byte-pattern search in the raw image (little-endian pointer search) in order to reconstruct tables/records that Ghidra did not recognize as separate functions. The source address is given for every statement; gaps are named under "Open items".

---

## 1. Summary

The firmware is, to well over 95 %, Quectel's standard AT firmware for the FC41D (Beken SDK, lwIP, mbedTLS, FreeRTOS, BLE 5 stack, Quectel AT command set for WLAN/BLE/TCP/SSL/MQTT/HTTP/FOTA). The complete AT command dispatch table (see §2.4) contains **exactly one** command that does not follow the Quectel naming scheme "AT+Q…": **`AT+HMIOPEN`**. Besides that there are a handful of Hamedata-specific additions (own flash configuration layer with developer debug tags "lyy debug"/"[cx]", an "_HM" build suffix, default WLAN credentials, a BLE "connect_key"). The rest of the differences from the previous version 202409 is a version jump of the underlying Quectel SDK base (new standard commands for BLE/MQTT/SSL/HTTP/secure FOTA), not Hamedata's own development.

---

## 2. Hamedata/Marstek-specific code

### 2.1 `AT+HMIOPEN` and the `hame_tcp` task

Locations (data references in the raw image, address = file offset + 0x10000):

- AT command table, entry `AT+HMIOPEN`: table slot at **0x000f7460** (name pointer → string "AT+HMIOPEN" @0x000f8098; help text pointer @0x000f7464 → "AT+HMIOPEN=1/0,ip,remote,local" @0x000f80a3; handler pointer @0x000f7468 = **0x000419f9** (Thumb, real function start 0x000419f8)).
- Associated string group (all unreferenced by decompiled code, located only via byte pattern in the raw image, the referencing code lies in the address range 0x419f8, which is **not** present as a separate function in this export – see "Open items"):
  - "hame_tcp_connect_server_command" @0x000ed22f
  - "hame_msg_handle ret: %d\r\n" @0x000ed2ab
  - "%s+HMIOPEN SUCCESS %s" @0x000ed272 / "%s+HMIOPEN FAIL" @0x000ed313 (URCs toward control board/host)
  - "hm_task_enable = %d ,enable = %d\r\n" @0x000ed333, "hm_task_enable = %d \r\n" @0x000ed356
  - Task name "hame_tcp" @0x000ed3cc

**How it works (addendum 2026-09-30, verified after creating the missing functions in Ghidra):**

- Handler `atcmd_HMIOPEN` (@0x000419f8): expects `AT+HMIOPEN=<enable>,<ip>,<remote_port>,<local_port>`. `enable` 0/1 is stored in `hm_task_enable` (@0x003f1ec0); the IP is stored via `inet_aton` at @0x003f1ec4, the remote port at @0x003f1ec8, the local port at @0x003f1eca. With `enable=1` the FreeRTOS task **"hame_tcp"** (entry `FUN_000416ec`, stack 0x800, priority 7) is started.
- Task `FUN_000416ec` ("hame_tcp"): creates a TCP socket, **binds it to the local port** and **actively connects to the given IP/remote port** (non-blocking with `select`, 10 s timeout). On success the URC `+HMIOPEN SUCCESS` goes to the host (control board), otherwise `+HMIOPEN FAIL`. After that the task receives in a loop up to 100 bytes per packet (as long as `hm_task_enable` = 1), prints them as hex in the log and passes them on to `hame_msg_handle`.
- `hame_msg_handle` (`FUN_0004163c`): frame format **`AA <len> <cmd> <data…> <checksum>`**, checksum = 8-bit sum of the bytes from `<len>` onward (`FUN_000415ee`). Commands:
  - `cmd 0x01` → `FUN_00041558`: **switches a GPIO of the module** (byte 3: pin 0/1 → Beken GPIO 6 or 7; byte 4: 0 = input, 1 = output; byte 5: level, set via `bk_gpio_output`).
  - `cmd 0x02` → looks up byte 3 in a 9-entry table (@0x00400504, value pairs) and calls `FUN_000415d4` → `bk_uart_initialize(0)` with the value found, i.e. **UART reinitialization with a different parameter set** (presumably baud rate; the table contents are in `.data` and were not read without an unpacked RAM image).
  - Response to the peer: `AA 01 <ret> <ret+1>` (`ret` 0 = OK, 1–6 = error codes).
- **Callers:** None of the Control firmwares 147–151 sends `AT+HMIOPEN` (search in all five decompilations and listings: 0 hits). The command is thus currently unused – presumably a factory/service tool or intended for a future Control version.
- **Classification:** A TCP connection established from outside, over which a peer can switch two module GPIOs and reconfigure the UART, is a typical remote maintenance/recovery pattern (e.g. BOOT/RESET lines of a connected microcontroller). Which lines GPIO 6/7 actually drive on the Venus D board cannot be determined without a schematic/hardware.

### 2.2 Own flash configuration layer (developer fingerprint)

On top of the standard Quectel KV flash API (`FUN_00039be4` write, item IDs 8/9) Hamedata has built its own config management, recognizable by consistent debug tags that do not match the Quectel style:

- "lyy debug" – 22 locations in 13 functions, including `FUN_00076228` (@0x00076228, `save_user_info_tab_to_flash`), `FUN_000767b8` (@0x000767b8, `save_user_info_tab_to_backup_flash`), `FUN_000766c4` (@0x000766c4, `set_def_config`).
- "[cx]" – 8 functions, including likewise `FUN_000767b8`.

`FUN_00076228`/`FUN_000767b8` write a config block with fields SSID (tag `0x11111104`), PSK (tag `0x11111105`) and device name "Quec_BT" (tag `0x11111107`) to flash/backup flash. **Compiled in as default values: SSID "hellow", PSK "12345678"** (@0x00076228 l. 30/54/68 and @0x000767b8 l. 32/56/70; the PSK is stored encrypted via `FUN_0003ecc0` – presumably AES – before writing). `FUN_000766c4` (`set_def_config`, @0x000766c4) reads these fields back at boot and explicitly treats SSID="hellow"/PSK="12345678" as "not configured" (log "not used ssid and psk!"), and otherwise connects with the stored credentials. So this is a factory/reset placeholder, not an active secret – but a hard-coded default credential pair.

### 2.3 Further Hamedata findings (cannot be fully resolved)

The following strings were initially found only via data references; the associated handlers have since been created as functions (see `atcmd_*` in the current export), but have not yet been evaluated in detail:

- "user ble passcode" @0x000e5358 – indication of an additional, user-set BLE pairing passcode on top of the standard BLE security (see below).
- "connect_key= %s , g_connect_key = %s\n" @0x000f7b86 – referenced by code @0x00068040/0x000682b0; points to a global variable `g_connect_key` that is compared with a passed value (access/pairing key for BLE or the HMIOPEN tunnel). Value and comparison logic cannot be reconstructed (handler not in the export).
- Fixed NTP hosts "ntp1.aliyun.com" @0x000f7ab4 and "ntp.aliyun.com" @0x001073b8, as well as an IP literal "210.72.145.44" @0x000f7ac4 immediately adjacent to the ntp1.aliyun.com reference (@0x00068040/0x00068044) – presumably a fallback IP for the NTP host. Compiled-in time servers of a Chinese cloud provider (Alibaba) in a device that is also sold in the EU; whether they are actually contacted (or are just legacy code) cannot be determined from the export.
- BLE security itself is **standard**: source path comment "beken378/driver/ble_5_x_rw/ble_pub/app/src/app_sec.c" @0x000f649f as well as strings "GAPC_TK_EXCH"/"GAPC_LTK_EXCH" @0x000f643d/0x000f64e5 show the unmodified Beken/RivieraWaves BLE 5 stack. The "user ble passcode"/"connect_key" mechanism sits above (in addition to) this standard pairing logic.
- Version/build identifier: `AT+QVERSION` handler `FUN_00075b60` (@0x00075b60) returns the build string "FC41DAAR18A05M02_HM.bin" (@0x000e63a2) – standard Quectel mechanism, only the "_HM" suffix is Hamedata branding.
- Standard BLE advertising name "Quec_BLE"/"Quec_BT" (@0x000e61dc, referenced from `FUN_0003b5c0` @0x0003b5c0, the standard Quectel "AT+QBLE…" test command dispatcher) is **not** renamed by Hamedata – according to the Control FW documentation, the device name visible to users is set at runtime by the control board via `AT+QBLENAME=MST_VNSD_xxxx`.

### 2.4 Reconstructed excerpt of the AT command table

Via pointer search in the raw image, the contiguous AT command dispatch table (records of 3 pointers each: name, help text, handler) was reconstructed from 0x000f72bc to at least 0x000f8788 (≈95 entries: QGMR, CSUB, QREDDAVN, QRST, QSETBAND, QSETBAUD, QWLANOTA, QSLEEPMODE, QPING, QTESTFLASH, QURCCFG, QWLMAC, QSTADHCP(_DEF), QSTAAPINFO(_DEF), QSTASTOP, QSTASTATIC, QSTAST, QWSCAN, QSECCFG, QSOFTAP(STOP), QAPSTATIC, QAPSTATE, QAPSTAINFO, QGETIP, QGETWIFISTATE, QSETSTARETRY, QLOWPOWER, QDEEPSLEEP, QAPGETLIST, QTIME, QAPCFG, QGPIOCFG, **HMIOPEN**, QSTAINFORAW, all QBLE* commands, QIOPEN/QISEND/QIRD/QIACCEPT/QISTATE/QISWTMD/QICLOSE/QIGETERROR/QICFG, QMTCFG/QMTOPEN/QMTCONN/QMTSUB/QMTUNS/QMTPUB(RAW)/QMTDISC/QMTCLOSE/QMTRECV, QSSLCERT/QSSLCFG/QSSLOPEN/QSSLSTATE/QSSLSEND/QSSLRECV/QSSLCLOSE, QHTTPCFG/QHTTPGET/QHTTPPOST/QHTTPREAD, QNTP, QIDNSGIP, QFOTA, QWDG). **`AT+HMIOPEN` (table slot @0x000f7460) is the only entry whose name does not begin with "AT+Q".**

Reconciliation with the public manual "Quectel_FC41D_AT_Commands_Manual_V1.0" (quectel.com, retrieved 2026-09-30): `AT+HMIOPEN` does not appear in it. `AT+QWLANOTA` is documented there ("Start OTA Upgrade", syntax `AT+QWLANOTA=<URL>`). `AT+QREDDAVN`, `AT+QFOTA`, `AT+QSECCFG`, `AT+QDBGCFG` are also missing in this manual status, but – unlike HMIOPEN – clearly show up in the naming scheme and function code as Quectel's own extensions; `AT+QREDDAVN` is, according to a Quectel release note for other modules (BG96), a query of a security-certified version number ("version number certified by R02A07_01") – presumably a platform-wide Quectel feature of a newer SDK generation, not Hamedata-specific (return value here: fixed string "SRCI0885TREDDAVN001" @0x000f77ab, handler @0x00067a18 – not contained in the export).

---

## 3. Security: secure FOTA path (`ql_secfota_*`)

The naming scheme (`ql_` = Quectel) and code show: this is **Quectel's own** secure FOTA mechanism, not a Hamedata in-house development.

- `FUN_0003e890` (@0x0003e890, "ql_secfota_analys_file"): parses a container with the 13-byte magic "QLSECFOTA!!!" (@0x000ec765), extracts an appended signature (length ≤ 255 bytes, otherwise error "signature len error"), copies the payload data and determines signature length/offset.
- `FUN_0003eba4` (@0x0003eba4, "ql_secfota_verify_signature"): contains, **hard-coded as a string literal**, a NIST P-384 (secp384r1) public key in PEM format:
  ```
  -----BEGIN PUBLIC KEY-----
  MHYwEAYHKoZIzj0CAQYFK4EEACIDYgAEn38D9dJtf9k2ReCLAccOacZ5GOKIS47r
  JTAIKmUszTn2B+OmfceEUSpubPNn9vGwA7v1nQKIJK2yBeNNYEEnLHMNHa7TEX3o
  peIHLV6UrKezJjasPNszcbnrhoz5+S3I
  -----END PUBLIC KEY-----
  ```
  (Address of the literal @0x0003eba4 function, referenced as a pointer in `func_0x0003eb14`, which performs the actual ECDSA/SHA-384 verification – function names "ql_secfota_verify_ecdsa384_signature"/"ql_secfota_calculate_file_sha384" @0x000ec9cb/0x000eca0c, both not separately referenced, so used inline/statically). There is **no** separate key storage – the trust anchor is compiled directly into the app image.
- On success: `FUN_0003ea90` (@0x0003ea90, "ql_secfota_move_file") copies the verified image into a flash buffer and writes it. On failure: `FUN_0003e85c(0, 0x1000)` erases the staging area, message "FOTA Signature verification failed" (@0x000ec89e) – the package is **not** accepted.
- Caller: `FUN_0004bc04` (@0x0004bc04) downloads a file via HTTP/HTTPS GET from a URL passed as a parameter (default port 80 or 443) and then immediately calls `FUN_0003eba4`. This is the plausible implementation location behind `AT+QFOTA`/`AT+QWLANOTA` (both standard Quectel commands, same AT table as HMIOPEN, see §2.4).

**Key finding on the update path actually used (corrected 2026-09-30, verified in code):** The control board updates the module exclusively via `AT+QWLANOTA=<url>` – also with the `.rbl` packages from hamedata.com (details: `VNSD_FC41D_Update_and_SoftAP.md`). In 202512, `atcmd_QWLANOTA` (@0x00067de0) performs the download via `FUN_0004bc04`, and this function **mandatorily** calls `ql_secfota_verify_signature` (`FUN_0003eba4`) **after every successful download**: the staging area must begin with "QLSECFOTA!!!" and carry a valid ECDSA-P384/SHA-384 signature, otherwise the first sector is erased and "verify fail" is reported. The Marstek packages delivered so far begin with "RBL\0" and carry no signature – **a module with 202512 no longer accepts them.** The format is an overlay: after successful verification, `ql_secfota_move_file` writes the original first bytes (appended at the end of the file) back to the start, so that a normal `.rbl` package lies in staging again, which the bootloader (with the AES default key) processes as before. The old module firmware 202409, by contrast, downloads via plain HTTP (without TLS) and checks no signature.

Further hard-coded credentials: see §2.2 (WLAN default "hellow"/"12345678") and §2.3 (`g_connect_key`, unresolved).

---

## 4. Changes 202409 → 202512

Basis: only-in-new/only-in-old diff of the string tables (`fc41d_strings_diff.json`, 141 only old / 212 only new).

**New in 202512 (selection, classified):**

- *Quectel SDK version jump (standard, no Hamedata work):* complete BLE 5 AT command set (`AT+QBLE*`), MQTT client (`AT+QMT*`), SSL/TLS client (`AT+QSSL*`, mbedTLS x509/ECDSA error messages), HTTP client (`AT+QHTTPGET/POST/PUT`), `AT+QSECCFG`, `AT+QDBGCFG`, `AT+QREDDAVN` (see §2.4), `AT+QSTAINFORAW`, `AT+QAPCFG`. These function groups simply did not yet exist in the module in 2024 (202409).
- *Secure FOTA (Quectel standard, see §3):* complete `ql_secfota_*` family including embedded ECDSA-P384 public key – in 202409 there is **no** counterpart for it (none of the strings "QLSECFOTA", "ql_secfota_*", no PEM key in the old image). I.e. **the 202409 firmware had no cryptographic signature verification for any of its OTA paths.**
- *Hamedata-specific:* `AT+HMIOPEN` (see §2.1) – exists only in 202512, not in 202409.
- *Hamedata-specific, new:* default credentials "12345678"/"hellow" (see §2.2), `connect_key`/`g_connect_key` and "user ble passcode" (see §2.3) – all present only in 202512, not in 202409.
- NTP fallback IP "210.72.145.44" next to "ntp1.aliyun.com" (see §2.3) – new in 202512.
- "AT+QSOFTAP: WARNING_NO_PASSWORD" – new warning message when the softap mode is operated without a password (standard notice, no functional change).

**Removed since 202409 (present only in the old version):** AirKiss (Tencent WLAN smart-config protocol) as well as the local lwIP httpd web interface pages (`/index.shtml`, fixed AP IP `192.168.19.1`) for initial configuration via browser. The old "module opens its own AP + web interface" onboarding was removed without replacement, presumably in favor of the new BLE provisioning (`AT+QBLEINIT=3`, documented in the public manual: SSID/password are set via BLE GATT write with an embedded `AT+QSTAAPINFO=...` command). This too is a Quectel SDK modernization, not a Hamedata-specific decision.

**Security classification of the diff (corrected 2026-09-30):** With 202512 the signature verification applies exactly on the path used by Marstek (`AT+QWLANOTA`). A module with 202409 accepts any `.rbl` without verification (including downgrades and self-built packages); a module with 202512 only packages with a valid Quectel/Hamedata signature. The step 202409 → 202512 is thus practically a one-way street. The only path without signature verification remains `AT+QFOTA` (transfer over the UART), which the control board uses nowhere.

---

## 5. Standard vs. custom – short version

| Area | Classification |
|---|---|
| Base AT firmware (WLAN/BLE/TCP/SSL/MQTT/HTTP, Quectel SDK on Beken BK7231N) | Standard Quectel/Beken |
| Secure FOTA mechanism `ql_secfota_*` + ECDSA-P384 key | Standard Quectel (naming scheme, no Hamedata traces) |
| `AT+QREDDAVN` | Presumably Quectel standard (platform-wide feature, not documented in this manual status) |
| BLE pairing base (`app_sec.c`, GAPC_TK/LTK_EXCH) | Standard Beken/RivieraWaves |
| `AT+HMIOPEN` + `hame_tcp`/`hame_msg_handle`/`hm_task_enable` | **Hamedata-specific** |
| Own flash config layer ("lyy debug", "[cx]", `save_user_info_tab_to_(backup_)flash`, `set_def_config`) | **Hamedata-specific** |
| Default WLAN credentials "hellow"/"12345678" | **Hamedata-specific** (factory/reset placeholder) |
| `_HM` build suffix in `AT+QVERSION` response | **Hamedata branding** (cosmetic) |
| `g_connect_key`/"user ble passcode", NTP hosts/IP | Presumably Hamedata-specific, details cannot be reconstructed (see below) |

---

## 6. Open items

1. *(done 2026-09-30)* The missing functions (including all AT handlers, `atcmd_HMIOPEN`, `hame_tcp`) have been created in Ghidra; the current export contains 5205 functions. Still open is the detailed evaluation of `connect_key`/`g_connect_key` and of the NTP code.
2. *(partly done)* `AT+HMIOPEN` is not sent by any Control version 147–151; whether another device/tool uses it is open. Which signals GPIO 6/7 drive on the board cannot be determined without hardware.
3. *(corrected)* The earlier assumption "`.rbl` path without signature protection" was wrong for 202512 (see §3). The bootloader itself is not available.
4. `AT+QREDDAVN`/"SRCI0885TREDDAVN001" could only be made plausible via a Quectel release note for another module (BG96), not via an FC41D-specific source – the classification "Quectel standard" is probable, but not conclusively confirmed.
5. Exact meaning of the IP "210.72.145.44" (NTP fallback or another purpose) not verified.
