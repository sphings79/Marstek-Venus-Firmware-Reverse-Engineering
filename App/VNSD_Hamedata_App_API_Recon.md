# Hamedata App API Recon — Marstek Venus D Communication Module OTA

> 🇩🇪 Deutsche Version: [VNSD_Hamedata_App_API_Recon_de.md](VNSD_Hamedata_App_API_Recon_de.md)

Reverse engineering of the Marstek app to pull the **communication module (FC41D) firmware** directly from the
cloud — without having to wait for the device-specific rollout. **Result: solved.**

> **Anonymization:** In this document, real identifiers are replaced by placeholders.
> `<TOKEN>` = account access token · `<UID>` = account user ID · `<DEVID>` = device ID ·
> `<MAILBOX>` = account email address. The real values do not belong in the documentation.

---

## 1. The App

- Package `com.hamedata.marstek` (internally `com.hamedata.powerx.cross_power_x`) — **Flutter** app.
- Business logic in the **Dart AOT snapshot** `lib/arm64-v8a/libapp.so` (plaintext string literals).
- Archived: `fc41d_archive/` (local, not in the repo) → `marstek_base.apk` (Flutter shell, DEX),
  `marstek_arm64.apk` (contains `libapp.so`), `btsnoop_ble_parser.py`.
- API hosts: `eu.hamedata.com` (prod), `eu-dev.hamedata.com`, `eu-staging.hamedata.com`,
  `www.hamedata.com` (legacy endpoints). Firmware CDN: `static-eu.marstekenergy.com`.

---

## 2. Authentication (Empirical)

- **The token MUST be passed as a query parameter** `?token=<TOKEN>`:
  - Token in the `Authorization: Bearer` header → `{"code":8,"msg":"Unauthorized"}`
  - Token in the POST body → `{"code":8,"msg":"无访问权限"}` (no permission)
  - Token as `?token=` → **gets through**
- The token is **NOT bound to the account**: A valid token also accepts foreign `uid`/`devid`.
  → You can query with the `devid` of an **enabled** device and receive its
  assigned firmware URL, regardless of who owns the token.
- The app additionally sends the headers `X-User-ID`, `X-Client-Type`, `X-Client-Version`, `X-App-Version`,
  `X-Device-Id` — but they are not mandatory for the OTA endpoints.

---

## 3. Solved: Obtaining the Communication Module OTA URL

### The Correct Endpoint
`GET https://eu.hamedata.com/ems/api/v1/getCheckWifiOta`
("WifiOta" = WiFi/communication module = FC41D). The initially assumed `check_fc4_ota.php` was
wrong — see §5.

### Mandatory Parameters
The server reports missing parameters one at a time as `Undefined array key "<name>"`:

| Parameter | Value | Remark |
|-----------|-------|-----------|
| `token` | `<TOKEN>` | in the query (not Bearer/POST) |
| `uid` | `<UID>` | account user ID |
| `devid` | `<DEVID>` | **selects the device** — decisive |
| `version` | e.g. `202001010000` | value **irrelevant** (gating is device-bound, see below) |
| `device_type` | `VNSD-0` | model code Venus D |
| `lang` | `English` | |
| `mailbox` | `<MAILBOX>` | account email |
| `click` | `false` | **read-only** — `true` would trigger the real update (do not use) |

### Working Call
```bash
MAIL='<MAILBOX>'
curl -sk -G 'https://eu.hamedata.com/ems/api/v1/getCheckWifiOta' \
  --data-urlencode 'uid=<UID>' \
  --data-urlencode 'devid=<DEVID>' \
  --data-urlencode 'lang=English' \
  --data-urlencode 'token=<TOKEN>' \
  --data-urlencode 'device_type=VNSD-0' \
  --data-urlencode "mailbox=$MAIL" \
  --data-urlencode 'click=false' \
  --data-urlencode 'version=202001010000'; echo
```

### Response (devid of an Enabled Device)
```json
{"code":1,"show":0,"msg":"ok","data":{
  "version":"202512040647",
  "url":"https://static-eu.marstekenergy.com/uploads/ota/20251227/202512271054507d95a7957.rbl"}}
```
→ **New FC41D firmware `202512040647` (build 2025-12-04)** vs. installed `202409090159`.
The CDN download is **unencrypted and without auth**. Saved as
`fc41d_archive/fc41d_new_202512040647.rbl`. Header/payload analysis: [`VNSD_FC41D_Comm_Module_OTA_Analysis.md`](../HM_HIE_FC41D/VNSD_FC41D_Comm_Module_OTA_Analysis.md).

---

## 4. Key Finding: Gating Is Device-Bound, Not Version-Driven

Proven by counter-tests:

| Test | Result |
|------|----------|
| `getCheckWifiOta` with **own** (not enabled) devid | `{"code":0,"show":1,"msg":"固件已经最新"}` (= up to date, no update) |
| `getCheckWifiOta` with **enabled** devid | returns `version` + `url` (see above) |
| `checkSmallBalconyOTA` with **own** uid, `version=0` | empty list (`data.control=""`, …) |
| `checkSmallBalconyOTA` with **enabled** uid, `version=0` | full list (control/bms/micro URLs) |

**The `version` parameter forces nothing.** The server knows the target firmware **assigned to the device**
(rollout whitelist) and returns a URL only for enabled devices. That is why
a device that is not enabled never gets a response — no matter which version you specify.
**Consequence:** You have to query with the `devid` of an already **enabled** device.

---

## 5. Dead Ends (Documented So They Are Not Repeated)

- **`/app/neng/check_fc4_ota.php`** — supposed comm module check. Auth via `?token=` works,
  but the endpoint demands an unknown mandatory parameter (`{"code":"0","msg":"参数为空"}` =
  parameter empty) for ALL guessed names (devid, mac, snn, fc4V, fcv, fc41d_ver, …). Wrong path —
  `getCheckWifiOta` is the right one.
- **`getDateInfoeu.php`** (plaintext HTTP on the device) — only time sync + update flags, returns no URL
  and ignores the reported version.
- **MITM of the phone** — Android 7+ does not trust user CA certificates → mitmproxy fails against the
  app (without root/repacking).
- **logcat** — the release build prints no Dart logs (only Android system noise).

---

## 6. System Firmware Catalog (for Comparison)

`GET https://eu.hamedata.com/ems/api/v2/checkSmallBalconyOTA` — the **system FW** counterpart
(control/bms/micro/mppt), **not** the comm module.

Parameters: `uid`, `lang`, `token`, `device_type=VNSD-0`, `mailbox`, `click=false`,
`is_fourDigit={"control":false,"bms":false,"micro":false,"mppt":false}`, `m`, `sbv`, `mppt`, `inv`.

Response structure:
`data.{control,bms,micro,mppt,dcdc,bms_pack1,bms_pack2,led,charger}` →
each `{version, url, crc, size, force_update, is_boot, is_http, remark, …}`.
For an enabled device it returns the `control/bms/micro` `.bin` URLs on
`static-eu.marstekenergy.com`.

> **⚠️ Do NOT use `click=true`** — corresponds to the real "Update" click and can trigger an OTA on the
> target device. For pure research always use `click=false`.

---

## 7. Further OTA Endpoints (Found in libapp.so, for Later)

```
/ems/api/v1/getCheckWifiOta        ← comm module (FC41D)         [solved]
/ems/api/v2/checkSmallBalconyOTA   ← system FW (Venus D)         [solved]
/ems/api/v1/checkMicroDeviceOTA    ← micro inverter
/ems/api/v1/checkAcCoupleOta       ← AC coupler
/ems/api/v2/getScreenOtaUpdate     ← display/screen
/ems/api/v2/getCommonOtaUpdate     ← generic (needs 'version')
/ems/api/v1/checkDeviceVersion, /ems/api/v1/getDeviceVersion
/ems/api/v1/getCheckAppUpdate      ← app update
```

---

## 8. BLE Fallback (Device-Independent)

During the real comm module update assigned to the device, the firmware can also be captured **without** the cloud API:
HCI snoop of the BLE transfer, GATT handle `0x0012` (FF01) + `0x0018` (FF06), split frames by
`73/LEN/23/CMD/…/XOR` → firmware blob. See [`VNSD_BLE_Comm_Module_Update_Capture_Guide.md`](../HM_HIE_FC41D/VNSD_BLE_Comm_Module_Update_Capture_Guide.md).
