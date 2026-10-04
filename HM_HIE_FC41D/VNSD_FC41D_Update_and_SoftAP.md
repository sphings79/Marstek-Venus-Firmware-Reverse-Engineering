# VNSD FC41D — Module Update Paths and SoftAP Activation

> 🇩🇪 Deutsche Version: [VNSD_FC41D_Update_und_SoftAP_de.md](VNSD_FC41D_Update_und_SoftAP_de.md)

**Status:** 2026-09-30.

**Sources:**
- FC41D module firmware 202512040647 (`FC41DAAR18A05M02_HM`), decompilations (5205 functions):
  `<project>/Dekompilate/FC41D_202512040647_app.bin/` (local, not in the repo)
- Strings with referencing functions (new/old) and string diff:
  `.../scratchpad/strings_FC41D_202512040647_app.json`, `strings_FC41D_202409090159_app.json`, `fc41d_strings_diff.json` (local, not in the repo)
- Control firmware v151, decompilations: `<project>/Dekompilate/VNSD_Control_v151/`
- Preliminary reports: `VNSD_FC41D_Hamedata_Customizations.md`, `VNSD_FC41D_Comm_Module_OTA_Analysis.md`,
  `VNSD_BLE_Comm_Module_Update_Capture_Guide.md`, `security/VNSD_Firmware_Keys_and_Unpacking.md`,
  `security/WLAN_Credentials_via_Modbus_41500.md` (both `security/` files local, not in the repo; all in the same project folder)

Methodology: offline analysis of the Ghidra decompilations of both firmware versions as well as targeted cross-reference search
(`grep` over the exported `.c` files) for callers/callees. Every statement is evidenced
with a function name/address; unverified points are listed under "Open items".

---

## A. How the FC41D module can be updated

### A.1 Update entry points in the module itself

In the reconstructed AT command table (see `VNSD_FC41D_Hamedata_Customizations.md`, §2.4) there are **two** paths
that actually trigger write accesses to the OTA partition, plus a pure query command:

| Command | Handler (address) | Accepts | Signature check |
|---|---|---|---|
| `AT+QWLANOTA=<url>[,<port>]` (internally "http_ota") | `atcmd_QWLANOTA` @ `0x00067de0` → `FUN_0004bc04` @ `0x0004bc04` | HTTP **and** HTTPS (prefix check for "https://", default port 80/443); URL length ≤ 0x10000 if a third parameter is passed | **Yes** – mandatory, see below |
| `AT+QFOTA=("ota/init"\|"ota/write"\|"ota/done")[,len]` | `atcmd_QFOTA` @ `0x00076ab8` | raw bytes, chunk by chunk **over the serial interface** (no URL/download); max. 0xc00 (3072) bytes per "ota/write" chunk | **No** – no call to a verify function in this path |
| `AT+QGMR`, `AT+QVERSION` (version query) | `atcmd_QGMR` @ `0x00067980`, `FUN_00075b60` @ `0x00075b60` | – (read only) | not applicable |

**`AT+QWLANOTA` flow (signature enforcement, new in 202512):**
`FUN_0004bc04` (@`0x0004bc04`) loads the file via HTTP(S) GET (`FUN_00025ad4` @ `0x00025ad4`) into a
flash staging area and then immediately calls `FUN_0003eba4` (**`ql_secfota_verify_signature`**).
This function:
1. searches via `FUN_0003e890` (**`ql_secfota_analys_file`** @ `0x0003e890`) in the staging area for the
   13-byte magic **"QLSECFOTA!!!"** (literal @ the `0x0003e890` function itself) and an appended signature
   (≤ 255 bytes, otherwise "signature len error");
2. verifies the signature via `func_0x0003eb14` against a **NIST P-384 public key compiled fixed into the image**
   (PEM literal in `FUN_0003eba4` @ `0x0003eba4`);
3. on success: `FUN_0003ea90` (**`ql_secfota_move_file`**) copies the payload into the actual OTA partition;
4. on failure: `FUN_0003e85c(0, 0x1000)` **erases the staging area again**, log "FOTA Signature
   verification failed".

The container loaded this way must therefore be in **QLSECFOTA format** (magic + ECDSA-P384/SHA-384 signature).
That is a different format from the **`.rbl` container** actually shipped by Marstek/Hamedata
(magic "RBL\0", AES-256-CBC-encrypted payload, see `security/VNSD_Firmware_Keys_and_Unpacking.md`,
§1) — an `.rbl` file contains no "QLSECFOTA!!!" signature and would **not pass** this check
(see A.3).

`AT+QFOTA`, on the other hand, writes the raw bytes passed via "ota/write" directly into the partition via `FUN_00039a7c` (partition
write function, @`0x00039a7c`, called from `atcmd_QFOTA`); at the "ota/done" step only
`FUN_00076980` (@`0x00076980`, pure flash bookkeeping: close buffer, null pointers) is called – **no**
signature or format check. However, as shown below, this path does not occur anywhere in the current Control firmware v151.

### A.2 How the Control firmware triggers a module update

In Control v151 there are **two** explicitly named functions that do exactly this:

- **`Quectel_OTA_StartUpdate_AndReboot(uint32 use_local_ap)`** @ `0x08010074`: depending on the parameter, builds
  `AT+QWLANOTA=http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl` (parameter 0, timeout
  180 s) or `AT+QWLANOTA=http://192.168.137.1/FC41D_OTA.rbl` (parameter 1, "local AP", timeout 90 s), sends
  it via `Quectel_AT_SendAndVerify(...,"OK",...)` and afterwards hard-reboots the Control MCU
  (`System_Reboot()` @ end of the function) — regardless of the result.
- **`Quectel_OTA_StartUpdate_FromURL_AndReboot(char *url, uint32 url_len, uint32 port)`** @ `0x08010170`: builds
  `AT+QWLANOTA=<url>` or `AT+QWLANOTA=<url>,<port>` from a **freely passed** URL/port combination,
  waits up to 60 s for the URC result (`g_otaFc41dResultState`) and then likewise hard-reboots.

Both functions are connected to the module **only** via `AT+QWLANOTA`. `AT+QFOTA` does not occur in any of the 1958
decompiled Control v151 functions (checked via `grep` over all `.c` files) — the unauthenticated
chunk upload path of the module is **not used** by Control.

**Triggers (caller cross-reference):**

| Channel | Location | Trigger |
|---|---|---|
| **BLE app command `0x10`** | `BLE_App_Command_Dispatcher` @ `0x0800c730`, case `'\x10'` | Expects confirmation bytes `local_8[4..6] == 0x0a,0x0b,0x0c` and `g_networkLinkUp != false`; then `local_8[7]==0` → `Quectel_OTA_StartUpdate_AndReboot(0)` (cloud URL), `local_8[7]==1` → `Quectel_OTA_StartUpdate_AndReboot(1)` (local AP URL) |
| **BLE app command `0x1b`** | `BLE_App_Command_Dispatcher` @ `0x0800c730`, case `'\x1b'`, plus `BLE_Cmd_SetURL_Handler` @ `0x0800f25c` and `BLE_Pending_Commands_Process` @ `0x08015260` | Byte `local_8[4]` distinguishes "URL without port" / "URL with port" (port from two bytes, `local_8[len-3..len-2]`); the URL is buffered in `g_otaUrlBuf`, `g_otaUrlStartPendingFlag` is set and it is processed in the main loop `BLE_Pending_Commands_Process` via `Quectel_OTA_StartUpdate_FromURL_AndReboot(g_otaUrlBuf, len, port)` – **free URL, specified by the phone** |
| **Cloud/MQTT command type `9`** | `MQTT_CtrlCommand_Dispatcher` @ `0x080244e0` | Field `ot=0` → `Quectel_OTA_StartUpdate_AndReboot(0)`, `ot=1` → `Quectel_OTA_StartUpdate_AndReboot(1)` — the same cloud URL or local AP URL as with BLE command `0x10` |
| Version query | `Quectel_AT_QVERSION_Query` @ `0x08010370`: sends `AT+QVERSION` (up to 3 attempts) | reads the module firmware version (response string incl. `_HM` suffix, see Hamedata report §2.3) — no `AT+QGMR` |

A direct "Control forwards raw bytes received via BLE to the module" mechanism (looping firmware bytes through
BLE GATT directly) is **not** found in Control v151: the only BLE paths to the module update
are the two commands named above, which exclusively pass a **URL** that Control then forwards to the module via
`AT+QWLANOTA`. The module downloads the file itself via HTTP(S) — no firmware bytes flow over the BLE connection to the phone.

### A.3 Why the previous BLE update path probably fails with the new module firmware (202512)

Evidenced:
- The 202409 firmware contains **not a single** `ql_secfota_*` string and no ECDSA public key
  (`fc41d_strings_diff.json`: the entire `ql_secfota` function family is "only in new"). I.e. in 202409 there was
  **no** signature check in `AT+QWLANOTA` — a downloaded `.rbl` was apparently accepted into the
  partition unchecked (the analysis path of the old image is not available here at comparable decompilation depth,
  see "Open items" no. 1).
- In 202512 the **same** `AT+QWLANOTA` path now mandatorily calls `ql_secfota_verify_signature` (A.1). The
  real Marstek/Hamedata `.rbl` container (magic "RBL\0", AES-256-CBC, see
  `security/VNSD_Firmware_Keys_and_Unpacking.md`) contains **no** "QLSECFOTA!!!" signature — the check
  in `FUN_0003e890` would not find the magic in the staging area, the function would return with an error code,
  `FUN_0003eba4` would erase the staging area and `atcmd_QWLANOTA` would report "http_ota download failed."
  instead of "OK".
- Thus **both** `AT+QWLANOTA` triggers offered by Control (cloud URL `www.hamedata.com/…rbl`,
  local AP URL `192.168.137.1/FC41D_OTA.rbl`, as well as the free BLE command `0x1b` URL) would **fail at the same
  signature check** with the 202512 module firmware, provided that the file placed there is a normal
  `.rbl` container and not a QLSECFOTA-signed package.
- This explanation is a **very probable, but not device-verified** cause: it explains
  specifically why an app-side BLE update trigger that internally uses exactly this `AT+QWLANOTA` path suddenly fails with the
  new module although it worked with 202409. It was not checked whether the app
  instead (additionally or alternatively) uses a raw BLE GATT chunk transfer, as
  `VNSD_BLE_Comm_Module_Update_Capture_Guide.md` describes as a possibility (characteristic `FF06`,
  handle `0x0018`) — such a path does not occur in Control v151 (see above) and could in any case only reach the module via
  `AT+QFOTA`, for which Control has no call.
- Other tightenings visible in the code that might additionally be relevant: `AT+HMIOPEN` is new (202512 only,
  see Hamedata report §2.1) and the complete `AT+QBLE*` command set as well as the BLE security strings
  (`GAPC_TK_EXCH`/`GAPC_LTK_EXCH`) also exist only since 202512 according to the string diff — whether this changed the
  BLE pairing behavior of the module itself (not the Control firmware) cannot be conclusively clarified from the
  available exports (see "Open items" no. 2).

### A.4 Practical options for the owner

The code yields two concrete mechanisms already present in the device to get Control to issue its own
`AT+QWLANOTA` call — **but their success depends on whether the provided file passes the
QLSECFOTA signature check** (A.3), which according to current knowledge is **not** the case for an ordinary `.rbl` file:

1. **BLE command `0x1b` (free URL):** send frame `0x73 | LEN | 0x23 | 0x1b | ...` to characteristic `FF01`
   (handle `0x0012`, see `VNSD_BLE_Comm_Module_Update_Capture_Guide.md` §7), with embedded
   URL (and optionally 2 port bytes, `local_8[4]` controls "with/without port", see `BLE_App_Command_Dispatcher` case
   `0x1b`). Result: Control sends `AT+QWLANOTA=<own URL>[,<port>]`.
2. **BLE command `0x10` with parameter 1:** fixedly issues `AT+QWLANOTA=http://192.168.137.1/FC41D_OTA.rbl`
   (Windows mobile hotspot default gateway) — practically usable if the module itself (as a station) is connected to a
   hotspot that assigns exactly this address and a web server there provides `FC41D_OTA.rbl`.
3. **Cloud/MQTT command type 9 (`ot=0`/`ot=1`):** identical trigger to command `0x10`, but via the
   AWS IoT MQTT connection instead of BLE (requires cloud access).

**Result for the owner:** Without the private ECDSA-P384 key with which Hamedata would sign a QLSECFOTA package, none of these three paths leads to a successful acceptance of a normal `.rbl` file on
the 202512 module firmware. The only path visible in the code **without** signature check is `AT+QFOTA`
(chunk-wise writing over UART) — but Control v151 has **no** caller for it; an update via this
path would have to be done externally, directly via AT command over the UART connection Control↔module
(physical/serial access to this line required, e.g. by soldering/tapping), not via BLE, Modbus or cloud.

---

## B. SoftAP (the module's own access point)

### B.1 Module side: `AT+QSOFTAP`, `AT+QAPCFG`, `AT+QAPSTATIC`

| Command | Handler | Function |
|---|---|---|
| `AT+QSOFTAP=<ssid>[,<key>]` or `AT+QSOFTAP=OPEN` | `atcmd_QSOFTAP` @ `0x00068464` → `FUN_00074ecc` @ `0x00074ecc` | Starts the SoftAP with the given SSID (≤ 32 bytes) and key (8–63 bytes, WPA2); without key: **open network** and log/URC `+QSOFTAP: WARNING_NO_PASSWORD` |
| `AT+QAPCFG` | `atcmd_QAPCFG` @ `0x00067a64` | Subparameter `linknum` (max. 1–2 simultaneous clients) as well as a second mode flag comparison that has not been conclusively clarified (called via function pointer `DAT_004006d8`, see "Open items" no. 3) |
| `AT+QAPSTATIC=<ip>,<mask>,<gw>[,<dns>]` | `atcmd_QAPSTATIC` @ `0x000683a0` → `FUN_00074fcc` @ `0x00074fcc` | Sets IP/mask/gateway/DNS of the SoftAP in RAM buffers `DAT_004013f5`/`…405`/`…415`/`…425`; these values are read by `AT+QSOFTAP` at startup (`FUN_00074ecc`) |

There is **no** default SSID/PW compiled fixed into the handler `atcmd_QSOFTAP` – the caller must pass both.
The same applies to the IP configuration: `DAT_004013f5` etc. are pure RAM buffers with no
initial value in the flash image; whether a separate boot routine pre-populates them with a default (e.g. 192.168.0.1, see below)
cannot be determined from the export (Open items no. 4).

**Found, but not reliably identified as active second path:** `FUN_0003a3c8` (@`0x0003a3c8`, logged as
"SOFTAP_COMMAND") is functionally identical to `atcmd_QSOFTAP`, but uses a **hard-coded**
SSID/key (passed in) with a fixed IP configuration `192.168.0.1` / mask `255.255.255.0` / gateway `192.168.0.1`
/ DNS `192.168.0.1` (`FUN_000140c8` @ `0x000140c8`, literals directly in the code). This function has
**no determinable caller** in the export, neither from the AT command table nor from Control – presumably part of a separate
debug/shell command list (name "softap", string hit @ `0x000dda2e`) that could not be
fully reconstructed in the present excerpt.

**IP 192.168.19.1 / old web interface (old vs. new):** The string `192.168.19.1` (@`0x000faf79` in
202409090159, referenced from `FUN_00072498`) as well as `/index.shtml`, `/404.shtml` **no longer occur** in the **new**
firmware 202512040647 (confirmed via string diff and targeted search in both string lists:
zero hits for "192.168.19" and "shtml" in the new firmware). The old lwIP httpd
web interface for initial configuration via browser on the SoftAP is thus **confirmed removed**, as is
the associated AirKiss Tencent Smart Config protocol (whose complete function and string family
`airkiss_*` exists only in 202409). On the new firmware, according to current knowledge, **no** standalone web/TCP configuration service is
active on the SoftAP anymore; the only remaining traces of the
old Airkiss mechanism are log strings without associated `airkiss_*` core functions (see Hamedata report
§4).

**"12345678" is not a SoftAP/UI password:** As already evidenced in `VNSD_FC41D_Hamedata_Customizations.md` (§2.2),
SSID "hellow" and password "12345678" are **default WLAN station credentials** (not SoftAP!), which
`set_def_config` (`FUN_000766c4` @ `0x000766c4`) treats as a sentinel for "not configured" (log "not used
ssid and psk!") and which `save_user_info_tab_to_flash`/`_to_backup_flash` (`FUN_00076228`/`FUN_000767b8`) write
to flash as factory defaults. It is a factory/reset placeholder for the module's
station WLAN connection, **not** a SoftAP password and **not** a UI/app password.
A second, independent default credential set exists on the Control side
("SM471…Hame147258…", EEPROM `0x400`, Modbus register 41500 ff. — station WLAN default of the Control board,
see `security/WLAN_Credentials_via_Modbus_41500.md`); both value pairs are independent
factory defaults of different firmware components.

### B.2 Control side: Is `AT+QSOFTAP` sent at all?

**No.** A `grep` search over all 1958 decompiled Control v151 functions for the strings
"QSOFTAP", "QAPCFG", "QAPSTATIC", "softap"/"SoftAP" yields **no hit**. Control v151 sends exclusively
`AT+QSTAAPINFO`/`AT+QSTAAPINFODEF` (station mode: the module connects **as a client** to the home WLAN,
see `Quectel_WiFi_SetAPConfig` @ `0x08016a90` and `Quectel_WiFi_SetSTAInfo_NoSave` @ `0x08016bc0` — the name
"AAPINFO" here denotes the target AP information for the station connection, **not** the module's own SoftAP).
The actual SoftAP function of the FC41D (`AT+QSOFTAP`) is **not used anywhere** by the currently examined
Control firmware — there is no button trigger, no Modbus register, no
BLE command and no cloud command in it that activates it. A "no-WLAN fallback" via the SoftAP (module opens
its own hotspot if the station connection fails) does not exist in the code.

**Practical consequence:** To activate the module's SoftAP, `AT+QSOFTAP=<ssid>,<key>` (optionally
beforehand `AT+QAPSTATIC=<ip>,<mask>,<gw>`) would have to be sent **directly as an AT command over the UART line Control↔module**
— there is no trigger reachable from Control, the BLE app, Modbus or cloud. Local
configuration access via an interface hosted by the module itself (as formerly at 192.168.19.1) is, with the
current combination of Control v151 and module firmware 202512, **not provided for** according to this finding and is also no longer
present on the software side in the module (missing `/index.shtml` web interface).

---

## Open items

1. The analysis "`AT+QWLANOTA` did not check a signature in 202409" rests on the complete string diff
   (no `ql_secfota_*` strings/key in 202409) and the code path analysis of the comparison version, **not** on
   an equally deep decompilation of 202409 itself (only strings are available there, no comparable
   function decompilation) — the conclusion is plausible, but not verified at the code level of the old version.
2. Whether the BLE security newly added in 202512 (`AT+QBLE*` command set, `GAPC_TK_EXCH`/
   `GAPC_LTK_EXCH`) changed the module's pairing behavior toward the app (e.g. now mandatory
   bonding where previously pairing was open) cannot be clarified from the available exports – the
   associated code (`app_sec.c` path) lies outside the AT command handlers examined here.
3. `AT+QAPCFG` subparameter `0xb` (comparison against `&DAT_004006a4`, called via function pointer
   `DAT_004006d8`) could not be traced back to its exact meaning (presumably SSID visibility "hidden"/"visible")
   – the referenced data addresses lie outside the ranges exported as strings.
4. Whether the SoftAP IP buffers (`DAT_004013f5` ff.) are pre-populated by a boot init routine with a default (presumably
   192.168.0.1, analogous to `FUN_000140c8`), or whether `AT+QSOFTAP` without a prior
   `AT+QAPSTATIC` call would start with 0.0.0.0, cannot be determined from the export.
5. `FUN_0003a3c8` ("SOFTAP_COMMAND", fixed IP 192.168.0.1) has no caller that can be found in the export; the
   suspected membership of a separate debug/shell command list (separate from the AT command table)
   could not be verified.
6. Whether the Marstek app currently (still) uses a raw BLE GATT firmware chunk transfer (characteristic `FF06`,
   as described in `VNSD_BLE_Comm_Module_Update_Capture_Guide.md`) and whether this would be mapped to `AT+QFOTA` on the module
   could not be checked for lack of a real update capture with the 202512 firmware;
   in Control v151 there is in any case no caller for it.
