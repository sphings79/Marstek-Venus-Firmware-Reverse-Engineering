# Bluetooth capture of the communication module update (FC41D) — Android + Pi

> 🇩🇪 Deutsche Version: [VNSD_BLE_Comm_Modul_Update_Mitschnitt_Anleitung_de.md](VNSD_BLE_Comm_Modul_Update_Mitschnitt_Anleitung_de.md)

**Starting point:** The communication module update does NOT run via the cloud/WLAN
(`getDateInfo` permanently reports "no update"), but via a **dedicated button in the app**
that pushes the firmware from the phone into the FC41D module over **Bluetooth (BLE)**. In airplane mode it failed
because the app first has to download the `.rbl` from the network → **perform the update with internet ON.**

**Method:** Android's built-in **Bluetooth HCI snoop log** records at the HCI level — i.e.
*behind* the BLE radio encryption. We see the GATT writes in plain text. The app payload itself
can still be the encrypted `.rbl`; but we reliably get the transport path + blob.

Analysis is done headless on the Pi 3 B+ with `adb` + `tshark` (tshark is already installed).

---

## 1. Prepare Android (one-time)

1. **Unlock developer options:** Settings → "About phone" → tap **Build number** 7×.
2. Open **Developer options**:
   - Enable **USB debugging**.
   - Enable **"Enable Bluetooth HCI snoop log"** (on some phones a dropdown:
     then choose **"Enabled" / "Full" / "All"**, not "Filtered").
3. **Important:** For the snoop log to actually take effect, **turn Bluetooth off and on once**
   (on some devices even reboot). The log starts fresh in the process.

---

## 2. Prepare the Pi

```bash
sudo apt update
sudo apt install -y adb
```
Connect the phone to the Pi by **USB cable**. The phone shows "Allow USB debugging?" → **Allow**
(tick "Always allow from this computer"). Then:
```bash
adb devices          # must show the device as "device" (not "unauthorized")
```
Cross-check the snoop status (should be 1/"true"/"Full"):
```bash
adb shell settings get secure bluetooth_hci_log 2>/dev/null
adb shell dumpsys bluetooth_manager | grep -i snoop
```

---

## 3. Capture the update

1. **Airplane mode OFF** (internet on — the app must be able to download the firmware).
2. Optional, but recommended: additionally connect the **phone to the WLAN `MarstekCap`**. Then the
   Pi's continuous capture also records the **app's download traffic** in parallel (URL/DNS, possibly the file).
3. **Briefly turn Bluetooth off/on on the phone** (fresh snoop log).
4. In the Marstek app, press the **"Update communication module" button** and let it run through
   completely. Do not disconnect midway.

---

## 4. Pull the log to the Pi

Direct pull (only works with root on the phone, just try it):
```bash
adb pull /data/misc/bluetooth/logs/btsnoop_hci.log ~/btsnoop_hci.log
```
If "permission denied" → **bugreport route** (works without root):
```bash
cd ~
adb bugreport btbug          # creates btbug.zip (takes 1-2 min)
unzip -o btbug.zip 'FS/data/misc/bluetooth/logs/*' -d btbug
find btbug -iname 'btsnoop_hci*'
```
The file found (e.g. `btbug/FS/data/misc/bluetooth/logs/btsnoop_hci.log`) is our capture.

---

## 5. Extract the firmware from the GATT writes

Set the log file variable:
```bash
L=~/btsnoop_hci.log      # or the path from step 4
```
**a) Overview — which GATT handles are written to, and how often?**
```bash
tshark -r "$L" -Y 'btatt.opcode.method==0x12 or btatt.opcode.method==0x52' \
  -T fields -e btatt.handle 2>/dev/null | sort | uniq -c | sort -rn
```
The handle with **the most writes** (hundreds/thousands) is the firmware channel.

**b) Assemble this handle completely into a binary file** (adjust H, e.g. `0x0012`):
```bash
H=0x0012
tshark -r "$L" -Y "btatt.handle==$H and (btatt.opcode.method==0x12 or btatt.opcode.method==0x52)" \
  -T fields -e btatt.value 2>/dev/null | tr -d ' \n:' | xxd -r -p > ~/fc41d_ble_fw.bin
ls -l ~/fc41d_ble_fw.bin
```
**c) First assessment:**
```bash
xxd ~/fc41d_ble_fw.bin | head            # Magic? "RBL" at the beginning?
python3 - <<'PY'
import math,collections
d=open('/root/fc41d_ble_fw.bin','rb').read() if False else open(__import__('os').path.expanduser('~/fc41d_ble_fw.bin'),'rb').read()
if d:
    c=collections.Counter(d); H=-sum(v/len(d)*math.log2(v/len(d)) for v in c.values())
    print("Bytes:",len(d),"Entropie:",round(H,4))
PY
```
Entropy ~8.0 = encrypted (like the known `.rbl`). Magic `RBL\0` at the beginning = Realtek RBL.

**d) Inspect control packets / protocol** (small writes, notifications):
```bash
tshark -r "$L" -Y 'btatt' -T fields -e frame.number -e btatt.opcode -e btatt.handle -e btatt.value \
  2>/dev/null | head -60
```
Shows the handshake before the transfer (often: length/CRC/version command) — important for recognizing chunk offsets
and possibly a header/key exchange.

---

## 6. Secure the result

- Put `fc41d_ble_fw.bin` + the `btsnoop_hci.log` into the project folder `HM_HIE_FC41D/fc41d_archive/` (local, not in the repo).
- Compare header/build date/module ID/version with the installed `202409090159`.
- In parallel, search the Pi WLAN capture for the app's download of the `.rbl` (section 12 of the
  RPi guide / `marstek-ota-scan.sh`).

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `adb devices` shows `unauthorized` | confirm the USB debugging dialog on the phone; otherwise `adb kill-server && adb start-server` |
| `adb devices` empty | different USB cable (data cable!), set USB mode on the phone to "File transfer" |
| Snoop log empty/old | really set snoop to "Full", then BT off/on or reboot the phone, then repeat the update |
| `pull` permission denied | bugreport route (step 4) |
| no `btatt` in the log | possibly "Filtered" chosen instead of "Full", or wrong snoop → step 1.2 |
| many handles about equally frequent | additionally filter for large `btatt.value` lengths (the transfer uses max. MTU) |

---

## 7. Decoded BLE protocol (from capture 2026-08-20, "no update")

**Device:** BLE name `MST_VNSD_ee01` (= MAC aa:bb:cc:dd:ee:01 of the FC41D). Connection practically
unencrypted (1 btsmp frame) → HCI snoop shows everything in plain text.

### GATT structure (vendor service 0xFF00)
| Handle | Characteristic | Props | Role |
|--------|----------------|-------|------|
| 0x0012 | **0xFF01** | 0x14 (WriteNoResp+Notify) | App → device (commands) |
| 0x0015 | **0xFF02** | 0x14 | Device → app (replies/notify) |
| 0x0018 | **0xFF06** | 0x14 | unused → **firmware channel candidate for the real OTA** |

### Frame format
```
73 | LEN | 23 | CMD | [payload...] | XOR
```
- `0x73` ('s') = start magic, `0x23` ('#') = constant
- `LEN` = total length of the frame (incl. header + checksum)
- `CMD` = command (request and response use the same number)
- last byte = **XOR** of all preceding bytes (verified in all frames)

### Observed commands
| cmd | Meaning | Response (example) |
|-----|-----------|--------------------|
| 0x03 | Live telemetry (continuous poll) | 172-byte binary block (BMS/inverter values) |
| 0x0a | Statistics/energy | 85 bytes (here ~0) |
| **0x04** | **Identity + versions (ASCII)** | `type=VNSD-0,id=000000000000000000000000,mac=aabbccddee01,dev_ver=150,bms_ver=118,fc_ver=202409090159,inv_ver=116,mppt_v=104` |
| **0x11** | **Comm module version alone (ASCII)** | `202409090159` |
| 0x24 | Config/status | 6 / 25 bytes |
| 0x42 | Status | 42 bytes |
| 0x51 | (unclear) | 29 bytes `M05VnVC0TVi0rVe0JVBVBVA0` |

### "No update" mechanism (important)
The app reads the comm module version `202409090159` via BLE (cmd 0x04/0x11), sends it to the
**cloud** and gets the update decision from there. That is why the attempt failed in
**airplane mode** ("connection problem"): without internet there is no cloud response. So the update logic lives
**server-side**. → To be offered an update at all, the cloud has to release a newer version
for exactly this `id`/`fc_ver` (a friend was offered it, this device not yet).

### Capturing for the real update
1. Same HCI snoop setup (sections 1–4).
2. Start the update with **internet on**.
3. Afterwards, evaluate **FF01 (0x0012) AND FF06 (0x0018)** in the btsnoop — there, instead of ~115
   small frames, **thousands** of writes with the firmware chunks will run.
4. Split the chunks by `73/LEN/23/CMD/…/XOR`, concatenate the payloads → this yields the module firmware (presumably
   AES-encrypted like the `.rbl`).

The parser is located at: `fc41d_archive/btsnoop_ble_parser.py` (btsnoop → HCI → L2CAP → ATT, maps handles
to UUIDs and decodes the 0xFFxx frames).
