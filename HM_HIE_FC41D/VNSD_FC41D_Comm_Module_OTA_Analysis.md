# FC41D Communication Module — OTA Firmware Analysis

> 🇩🇪 Deutsche Version: [VNSD_FC41D_Comm_Modul_OTA_Analyse_de.md](VNSD_FC41D_Comm_Modul_OTA_Analyse_de.md)

Analysis of the OTA `.rbl` files of the Wi-Fi/communication module (Quectel/Realtek **FC41D**) of the
Marstek Venus D. Two firmware versions are available:

| Label | Version | File | Size | SHA-256 (start) |
|--------|---------|-------|-------|------------------|
| **OLD** (installed) | 202409090159 | `fc41d_archive/HM_HIE_FC41D_remote_ota_20260819T194315Z_0d10d2a4d467.rbl` (local, not in the repo) | 679,696 B | `0d10d2a4d467…` |
| **NEW** (rollout update) | 202512040647 | `fc41d_archive/fc41d_new_202512040647.rbl` | 682,928 B | `63c2f353130634ce…` |

- **OLD** comes from the static base URL `http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl`
  (server header: Last-Modified 2025-03-10, ETag `67ce80f4-a5f10`) — this is the **already installed** version.
- **NEW** was obtained via the app cloud API (`getCheckWifiOta`, see [`VNSD_Hamedata_App_API_Recon.md`](../App/VNSD_Hamedata_App_API_Recon.md))
  and is hosted on Marstek's CDN: `https://static-eu.marstekenergy.com/uploads/ota/20251227/202512271054507d95a7957.rbl`.

---

## 1. RBL header (plaintext) — format & comparison

Both files begin with an **unencrypted Realtek bootloader header** (`RBL`), followed by the
encrypted payload. Header layout:

```
0x00  "RBL\0"                         Magic (Realtek/RTL8710 family)
0x04  02 01 00 00                     Format version (v1.2)
0x08  u32 (LE)                        Build timestamp (Unix, UTC)  — = version number
0x0c  "app\0"                         Image/segment name
0x10  00 00 00 00 00 00 00 00
0x18  u32                             differs per build → presumably CRC/checksum of the payload
0x1c  "FC41DAAR18A0xM02_HMVxx\0"      Module/HW variant identifier
0x34  "0000000000000…"               Reserved/padding (UID placeholder, empty for a generic image)
~0x60 …                              AES-encrypted payload (entropy ~8.0)
```

### Direct comparison OLD ↔ NEW
| Field | OLD (installed) | NEW (update) |
|------|-------------------|--------------|
| Version | 202409090159 | **202512040647** |
| Build @0x08 (UTC) | 2024-09-09 02:00:01 | **2025-12-04 06:47:58** |
| Magic / format | `RBL\0` `02 01 00 00` | `RBL\0` `02 01 00 00` |
| Image name @0x0c | `app\0` | `app\0` |
| @0x18 (CRC?) | `6337c7ea` | `63c779c3` |
| **Module ID @0x1c** | `FC41DAAR18`**`A04M02`**`_`**`HMV02`** | `FC41DAAR18`**`A05M02`**`_`**`HMV01`** |
| Payload entropy | 7.99976 | 7.99973 |

**Key observations:**
- **The build timestamp IS the version number.** OLD: 2024-09-09 01:59/02:00 = `202409090159`.
  NEW: 2025-12-04 06:47 = `202512040647`. Both files are thus confirmed as authentic.
- **The module ID has changed:** `A04M02→A05M02` (firmware/HW revision) and `HMV02→HMV01`.
  `HMVxx` is presumably the hardware variant code against which the server matches compatible images.
- Same wrapper, same structure → the FC41D uses the Realtek RBL container throughout.

---

## 2. How the module update is actually delivered

Confirmed (2026-08-20): The **actual update path** is NOT the static www.hamedata.com file,
but the app cloud API:

1. The app queries `GET /ems/api/v1/getCheckWifiOta` with the device's `devid`.
2. If the device is **enabled** for an update (rollout whitelist), the server returns
   `{"version":…,"url":"https://static-eu.marstekenergy.com/uploads/ota/…/<hash>.rbl"}`.
3. The new `.rbl` is downloaded from this CDN and then transferred to the module via **BLE** (app→module) or via
   a **MQTT-pushed URL** (`fc41d_url=`).

Details + reproducible call: [**`VNSD_Hamedata_App_API_Recon.md`**](../App/VNSD_Hamedata_App_API_Recon.md).

---

## 3. Why the earlier firmware checker did not see the module

- The **device** endpoint `getDateInfoeu.php` (plaintext HTTP, see §6) does know `fcv` (FC41D version),
  but is purely **time sync + update flags** and does not return a download URL.
- The initially suspected `checkSmallBalconyOTA` is the **system firmware** catalog (control/bms/micro/
  mppt) and contains **no comm module**.
- Only `getCheckWifiOta` is the correct comm module endpoint.
- In addition: The module version is a **12-digit timestamp**, not an integer like 150/118/116 —
  a naive "newer than" comparison fails.

---

## 4. Payload encryption

Everything from the header onward (~offset 0x60, payload) is **highly entropic, encrypted** — in OLD as well as NEW:

- Entropy **7.9997 bit/byte** (8.0 = perfectly random)
- No repeated 16-byte blocks → if it is a block cipher, **no ECB pattern**
- Payload size divisible by 16 (block-cipher compatible)
- Chi-square in the range of a true uniform distribution

**The known Control key `hamedatahamedata` (AES-128-ECB) does NOT decrypt the payload** —
tested at several start offsets, plus candidates from the module ID and null/FF keys; entropy remains
~8.0. The FC41D (its own Realtek Wi-Fi SoC with its own bootloader chain) uses a **different key
and/or a different scheme** than the STM32 Control telemetry stack.

---

## 5. Can the key be found?

**Pure brute force: no.** With AES-128 the key space is 2^128 — not attackable with any conceivable hardware.
A detection criterion (the decrypted `app` payload would have to contain ARM/Xtensa code + strings)
does exist, but does not help with a key space this size.

**Only key *extraction* is realistic** — exactly as with the Control AES key `hamedatahamedata`,
which was found by a raw byte scan in the image, not by a computational attack:

1. **Pull the key from the firmware/loader.** The decryption code runs in the module itself or in its
   bootloader. Whoever preprocesses the `.rbl` has the key.
2. **Use the Realtek RBL structure.** The format is documented (Realtek Ameba / RTL8710). If only a
   standard Realtek image is packed into a crypto wrapper, the *wrapper* layer is the point of attack.
3. **Look for a weak derivation.** Worthwhile only if the key is short/derivable (e.g. from the module ID or
   serial number via a simple KDF) → dictionary/structure attack over a few thousand candidates. The
   `hamedatahamedata` find shows: Marstek tends toward weak keys.

**New as an attack surface:** We now have **two** firmware versions. The plaintext headers confirm
the structure; a payload diff only makes sense after decryption, but two ciphertexts of the same
scheme can help with the key search.

---

## 6. Live traffic confirmed (RPi3B+ capture, 2026-08-20)

The Marstek was routed through a dedicated Raspberry Pi AP (`MarstekCap`). This empirically evidences
the **device's** cloud endpoints:

### 6.1 Version/time check — plaintext HTTP (port 80)
```
DNS:  eu.hamedata.com  ->  3.122.27.237   (AWS eu-central-1, Frankfurt)
GET   http://eu.hamedata.com/app/neng/getDateInfoeu.php
      ?uid=<REDACTED>&fcv=202409090159&aid=VNSD-0&sv=150&sbv=118
      &mv=101&cert=0&boot=18&inv=116&mpptv=104
User-Agent: quectel-fc41d
```
| Field | Value | Meaning |
|------|------|-----------|
| fcv | 202409090159 | Comm module FW (= .rbl build 2024-09-09) |
| sv / sbv / inv / mpptv | 150 / 118 / 116 / 104 | Control / BMS / Micro / MPPT |
| boot | 18 | Bootloader |
| **cert** | **0** | **Certificate check OFF → confirms VERIFY_NONE** |

Response (HTTP 200, chunked): `_2026_08_20_00_12_20_04_0_0_0`
= `_YYYY_MM_DD_HH_MM_SS_<weekday>_<flag1>_<flag2>_<flag3>`. Pure **time sync**; the three flags
`0_0_0` = no update. Returns **no** download URL (getCheckWifiOta is responsible for that, §2).

### 6.2 Push channel — AWS IoT MQTT over TLS (port 8883)
```
a40nr6osvmmaw-ats.iot.eu-west-3.amazonaws.com -> 13.37.135.170 (AWS Paris), :8883 TLS
```
Persistent MQTT connection; the cloud pushes commands/the OTA trigger (`fc41d_url=`) over this.

### 6.3 Local discovery
```
UDP  <marstek>:22222 -> <broadcast>:12345   (len 50, app discovery on the LAN)
```

---

## 7. Status & next steps

**Done:**
- Both firmware versions (OLD installed, NEW rollout) obtained and archived.
- OTA delivery path fully understood (getCheckWifiOta → static-eu CDN → BLE/MQTT to the module).
- Header decoded, authenticity + module ID change documented.

**Open:**
- Payload is AES-encrypted; the module key is missing for plaintext code (key extraction from loader/
  module FW or weak derivation — §5).

**Ongoing:**
- The RPi continuous capture + BLE HCI snoop automatically catch a real update assigned to the device
  (`VNSD_BLE_Comm_Module_Update_Capture_Guide.md`, `RPi3Bplus_Marstek_Capture_Guide.md`).
