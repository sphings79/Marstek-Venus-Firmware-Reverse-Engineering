# Capturing Marstek Traffic — Raspberry Pi 3 B+ (Tested Version)

> 🇩🇪 Deutsche Version: [RPi3Bplus_Marstek_Capture_Anleitung_de.md](RPi3Bplus_Marstek_Capture_Anleitung_de.md)

**Setup:** The Pi 3 B+ acts as its own WiFi access point. The Marstek connects only to it,
the Pi forwards via LAN cable to the router and captures all traffic, **rotating daily**.
It runs unattended for weeks; status is available at any time via SSH using `marstek-status.sh`. An
**auto-extractor** (systemd timer, every 10 min) additionally pulls every file transferred via HTTP
out of the captures on its own — so the new firmware ends up on disk as a file by itself,
without you having to search through pcaps.

- `eth0` (Ethernet) = upstream to the router
- `wlan0` (built-in 2.4 GHz WiFi) = access point for the Marstek (the FC41D only transmits on 2.4 GHz)

> **Confirmed live (2026-08-20):** The module firmware check runs over **plaintext HTTP** to
> `eu.hamedata.com` (port 80). URL + file are in the open in the capture, regardless of where an MQTT URL
> points. **MITM is not needed for the `.rbl`.** Details in `VNSD_FC41D_Comm_Module_OTA_Analysis.md` §6.
> For the MQTT push channel (AWS IoT, TLS:8883) see optional Appendix B (mitmproxy).

### Important rules during setup (otherwise you get today's errors)
1. **Paste each `sudo tee …` block in full in one go** and afterwards verify with the shown
   `cat`/`head` that the content is really in the file and **not** the `tee` line.
2. In **systemd unit files**, `%%` must be used (systemd turns it into `%`). Only there, not in the script.
3. `iptables` has to be installed separately (Bookworm no longer ships it).

---

## 0. Image with Raspberry Pi Imager

- **Raspberry Pi OS Lite (64-bit)**.
- In the settings (gear icon): **enable SSH**, user + password, time zone, country.
- **Leave the WiFi connection (SSID/password) EMPTY** — `wlan0` becomes our AP, not a client.
- Set **"Wireless LAN Country" = DE** (if possible; otherwise later via raspi-config).
- Upstream via **LAN cable** Pi → router. Log in via SSH.

---

## 1. Packages + Unblock WiFi

```bash
sudo apt update
sudo apt install -y hostapd dnsmasq tcpdump tshark iw rfkill iptables
sudo systemctl unmask hostapd
sudo rfkill unblock wlan
# Set the country if not done in the Imager:
sudo raspi-config nonint do_wifi_country DE
```
> Be sure to install `tshark` as well — the auto-extractor (section 9) needs it. For the question
> "Should non-superusers be able to capture packets?", **No** is sufficient.

Verify (for "Wireless LAN" it must say **Soft blocked: no**):
```bash
rfkill list
```

---

## 2. Take wlan0 Out of NetworkManager

```bash
sudo tee /etc/NetworkManager/conf.d/99-marstek.conf >/dev/null <<'NM_EOF'
[keyfile]
unmanaged-devices=interface-name:wlan0
NM_EOF
sudo systemctl restart NetworkManager
```
**Verify** — for `wlan0` it must say `unmanaged`:
```bash
nmcli device status
```

---

## 3. IP Forwarding, Persistent

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-marstek.conf
sudo sysctl --system >/dev/null
```

---

## 4. AP Setup Script (Sets IP + NAT) + Service

Create the script:
```bash
sudo tee /usr/local/sbin/marstek-ap-up.sh >/dev/null <<'AP_EOF'
#!/usr/bin/env bash
set -e
ip addr flush dev wlan0
ip addr add 192.168.50.1/24 dev wlan0
ip link set wlan0 up
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -C POSTROUTING -o eth0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -C FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -C FORWARD -i wlan0 -o eth0 -j ACCEPT 2>/dev/null || iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
AP_EOF
sudo chmod +x /usr/local/sbin/marstek-ap-up.sh
```
**Verify** — it must begin with `#!/usr/bin/env bash`, NOT with `sudo tee`:
```bash
cat /usr/local/sbin/marstek-ap-up.sh
```
Create the service:
```bash
sudo tee /etc/systemd/system/marstek-ap.service >/dev/null <<'APSVC_EOF'
[Unit]
Description=Marstek AP network setup
After=network.target
Before=hostapd.service dnsmasq.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/marstek-ap-up.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
APSVC_EOF
```

---

## 5. hostapd (the Access Point)

```bash
sudo tee /etc/hostapd/hostapd.conf >/dev/null <<'HOSTAPD_EOF'
interface=wlan0
driver=nl80211
ssid=MarstekCap
hw_mode=g
channel=6
country_code=DE
ieee80211d=1
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=EinLangesPasswort123
HOSTAPD_EOF
echo 'DAEMON_CONF="/etc/hostapd/hostapd.conf"' | sudo tee /etc/default/hostapd
```
**Verify** — it must begin with `interface=wlan0`:
```bash
head -3 /etc/hostapd/hostapd.conf
```

---

## 6. dnsmasq (DHCP + DNS Logging)

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig 2>/dev/null || true
sudo tee /etc/dnsmasq.conf >/dev/null <<'DNSMASQ_EOF'
interface=wlan0
bind-interfaces
dhcp-range=192.168.50.50,192.168.50.150,12h
dhcp-option=3,192.168.50.1
dhcp-option=6,192.168.50.1
log-queries
log-facility=/var/log/marstek-dns.log
DNSMASQ_EOF
```
**Verify** — it must begin with `interface=wlan0`:
```bash
head -3 /etc/dnsmasq.conf
```

---

## 7. Start the AP + Verify

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now marstek-ap.service
sudo systemctl enable --now hostapd
sudo systemctl enable --now dnsmasq
```
**Check** — all three must be running, wlan0 must have the IP:
```bash
systemctl is-active marstek-ap.service hostapd dnsmasq   # -> 3x active
ip addr show wlan0                                        # -> inet 192.168.50.1/24
```
If `marstek-ap.service` fails → `sudo bash -x /usr/local/sbin/marstek-ap-up.sh` shows the
line. Most common cause: `iptables` not installed (repeat step 1).

---

## 8. Capture Service (Default: SD Card)

```bash
sudo mkdir -p /var/captures
sudo tee /etc/systemd/system/marstek-capture.service >/dev/null <<'CAP_EOF'
[Unit]
Description=Marstek traffic capture
After=marstek-ap.service
Requires=marstek-ap.service

[Service]
ExecStart=/usr/bin/tcpdump -i wlan0 -n -s 0 -G 86400 -w /var/captures/marstek_%%Y-%%m-%%d_%%H%%M%%S.pcap -Z root
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
CAP_EOF
sudo systemctl daemon-reload
sudo systemctl enable --now marstek-capture
```
> **Remember:** The unit contains `%%Y-%%m-%%d_%%H%%M%%S` (double `%`). systemd turns it into `%Y-…`,
> which tcpdump substitutes as a date. A single `%` leads to broken paths (today's error).

**Verify:**
```bash
systemctl is-active marstek-capture     # -> active
ls -l /var/captures/                    # file marstek_YYYY-MM-DD_*.pcap (grows with traffic)
```
> **Cleanup of the SD card** is handled by the OTA scanner (section 9): it deletes raw pcaps after
> 21 days, **but keeps the extracted objects**. No separate cron needed.

---

## 9. OTA Auto-Extractor (Pulls the Firmware Out by Itself)

This script searches the captures every 10 min, exports every file transferred via HTTP
to `/var/captures/http_objects/`, logs firmware-sized objects + suspicious URLs to
`/var/log/marstek-ota.log` and cleans up old pcaps (>21 days).

```bash
sudo tee /usr/local/sbin/marstek-ota-scan.sh >/dev/null <<'OTA_EOF'
#!/bin/bash
set -u
CAPDIR=/var/captures
OUTDIR=/var/captures/http_objects
LOG=/var/log/marstek-ota.log
mkdir -p "$OUTDIR"
for f in "$CAPDIR"/marstek_*.pcap; do
  [ -e "$f" ] || continue
  tshark -r "$f" --export-objects http,"$OUTDIR" >/dev/null 2>&1
  tshark -r "$f" -Y 'http.request' -T fields -e http.host -e http.request.uri 2>/dev/null
done | sort -u > /var/captures/http_urls.txt
BIG=$(find "$OUTDIR" -type f -size +100k 2>/dev/null)
if [ -n "$BIG" ]; then
  echo "[$(date -Is)] MOEGLICHES FIRMWARE-OBJEKT:" >> "$LOG"
  ls -la $BIG >> "$LOG"
fi
grep -Ei 'rbl|ota|upgrade|firmware|\.bin' /var/captures/http_urls.txt >> "$LOG" 2>/dev/null
# delete raw pcaps older than 21 days (extracted objects are kept)
find "$CAPDIR" -maxdepth 1 -name 'marstek_*.pcap' -mtime +21 -delete 2>/dev/null
OTA_EOF
sudo chmod +x /usr/local/sbin/marstek-ota-scan.sh
```
**Verify** — it must begin with `#!/bin/bash`:
```bash
head -1 /usr/local/sbin/marstek-ota-scan.sh
```
Create and enable the timer + service:
```bash
sudo tee /etc/systemd/system/marstek-ota-scan.service >/dev/null <<'OTASVC_EOF'
[Unit]
Description=Scan Marstek captures for OTA firmware objects

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/marstek-ota-scan.sh
OTASVC_EOF

sudo tee /etc/systemd/system/marstek-ota-scan.timer >/dev/null <<'OTATIM_EOF'
[Unit]
Description=Run Marstek OTA scan every 10 min

[Timer]
OnBootSec=2min
OnUnitActiveSec=10min

[Install]
WantedBy=timers.target
OTATIM_EOF

sudo systemctl daemon-reload
sudo systemctl enable --now marstek-ota-scan.timer
sudo /usr/local/sbin/marstek-ota-scan.sh    # one trial run
```
**Verify** — timer scheduled, first objects present:
```bash
systemctl list-timers marstek-ota-scan.timer --no-pager
ls -la /var/captures/http_objects
```
As long as no update is pending, only the tiny 29-byte response of `getDateInfoeu.php` is found here
(the time sync `_YYYY_MM_DD_…_0_0_0`). When an OTA arrives, a **~680 KB object** (the `.rbl`)
appears and the log entry "MOEGLICHES FIRMWARE-OBJEKT" triggers.

**Look in at any time** (the `.rbl` would be at the very top, sorted by size):
```bash
tail -n 20 /var/log/marstek-ota.log; echo '--- Objekte (groesste zuerst) ---'; ls -laS /var/captures/http_objects | head
```

---

## 10. Status Script

```bash
sudo tee /usr/local/sbin/marstek-status.sh >/dev/null <<'STATUS_EOF'
#!/usr/bin/env bash
CAPDIR=/var/captures
echo "=== AP ======================================================"
echo "hostapd:  $(systemctl is-active hostapd)"
echo "dnsmasq:  $(systemctl is-active dnsmasq)"
echo "ap-setup: $(systemctl is-active marstek-ap.service)"
echo "wlan0-IP: $(ip -4 -o addr show wlan0 | awk '{print $4}')"
echo "=== Verbundene Geraete ======================================"
out=$(iw dev wlan0 station dump)
if [ -z "$out" ]; then echo "NICHT verbunden"; else
  echo "$out" | grep -E "Station|signal:|connected time:|rx bytes:|tx bytes:"; fi
echo "=== DHCP-Leases ============================================="
cat /var/lib/misc/dnsmasq.leases 2>/dev/null || echo "keine"
echo "=== Mitschnitt =============================================="
echo "capture: $(systemctl is-active marstek-capture)"
f=$(ls -1t "$CAPDIR"/marstek_*.pcap 2>/dev/null | head -1)
if [ -n "$f" ]; then
  s1=$(stat -c%s "$f"); sleep 2; s2=$(stat -c%s "$f")
  printf "Datei  : %s\nGroesse: %s (waechst: %s Bytes/2s)\n" "$(basename "$f")" "$(numfmt --to=iec "$s2")" "$((s2-s1))"
else echo "noch keine Datei"; fi
echo "=== OTA-Scanner ============================================="
echo "timer  : $(systemctl is-active marstek-ota-scan.timer)"
nobj=$(ls -1 "$CAPDIR"/http_objects 2>/dev/null | wc -l)
big=$(find "$CAPDIR"/http_objects -type f -size +100k 2>/dev/null)
echo "Objekte: $nobj  (Firmware-Verdacht: ${big:-keiner})"
tail -1 /var/log/marstek-ota.log 2>/dev/null
echo "=== letzte 5 DNS-Anfragen ==================================="
grep query /var/log/marstek-dns.log 2>/dev/null | tail -5
STATUS_EOF
sudo chmod +x /usr/local/sbin/marstek-status.sh
```
Invocation: `sudo marstek-status.sh`  ·  Continuous monitoring: `watch -n5 sudo marstek-status.sh`

---

## 11. Connect the Marstek + Verify

1. Test with a **phone** first: connect to WiFi **MarstekCap** → internet must work. `sudo marstek-status.sh`
   shows the phone as `Station`. This proves the whole chain.
2. Then set up the same WiFi **MarstekCap** in the **Marstek via the app**.
3. Read the Marstek IP from `dnsmasq.leases` (the FC41D module reports with hostname `wlan0`),
   then check live what it is doing:
   ```bash
   sudo tcpdump -i wlan0 -nn 'host <MARSTEK-IP> and not arp and not port 67 and not port 68 and not port 5353'
   ```

**What a healthy cloud contact looks like (really observed 2026-08-20):**
```
DNS  eu.hamedata.com -> 3.122.27.237          (AWS Frankfurt)
GET  http://eu.hamedata.com/app/neng/getDateInfoeu.php?...&fcv=...&cert=0...   (port 80, plaintext)
     Response HTTP 200:  _YYYY_MM_DD_HH_MM_SS_<weekday>_0_0_0   (0_0_0 = no update)
MQTT a40nr6osvmmaw-ats.iot.eu-west-3.amazonaws.com -> 8883/TLS   (AWS Paris, push channel)
UDP  <marstek>:22222 -> ...255:12345                              (local app discovery)
```
- The Marstek uses the **DNS of the Pi** (192.168.50.1) — the query for `eu.hamedata.com` is
  therefore in the DNS log. (Some IoT modules hard-code `8.8.8.8`; then nothing shows in the DNS log, but the
  pcap still captures everything.)
- **Right after connecting it is often silent for a few minutes** and only polls `getDateInfo`
  in bursts. "No traffic immediately" therefore does not necessarily mean something is broken.
- **Force a reconnect without pulling the power** (handy when the Marstek is on backup power):
  ```bash
  sudo systemctl restart hostapd     # kicks all WiFi clients -> Marstek re-registers completely
  ```
  Only DHCP + gratuitous ARP and permanent silence afterwards = the Marstek is still in the old WiFi / is not
  set to MarstekCap in the app.

---

## 12. Evaluate the Update

**Normal case: nothing to do** — the auto-extractor (section 9) has already placed the firmware as a file in
`/var/captures/http_objects/` and noted it in the log. Copy the large object via `scp` to the
Mac and into the project folder → read out header, build date, module ID, version and compare with the currently
installed `202409090159`.

Pull it manually from a pcap (if you want to look yourself):
```bash
# Were there any OTA-suspicious URLs?
tshark -r marstek_2026-08-20_000000.pcap -Y 'http.request' \
  -T fields -e http.host -e http.request.full_uri | grep -iE '\.rbl|\.bin|ota'
# Extract all files transferred via HTTP:
mkdir out && tshark -r marstek_2026-08-20_000000.pcap --export-objects http,out/
ls -lS out/
```

---

## 13. Troubleshooting (the Real Errors)

| Symptom | Cause | Fix |
|---|---|---|
| `iptables: command not found` (status 127) | Bookworm without iptables | `sudo apt install -y iptables` |
| Capture path becomes `…/etc/systemd/system-…` garbage | single `%` in the unit | use `%%` (step 8) |
| File/script contains the `sudo tee` line instead of the content | block slipped when pasting | paste the block again in full, verify with `cat`/`head` |
| `marstek-ap.service` failed | `wlan0` down/rfkill/NM | `rfkill list`, `nmcli device status` (unmanaged?), `bash -x` of the script |
| hostapd does not start | country not set | `sudo raspi-config nonint do_wifi_country DE`, `sudo rfkill unblock wlan` |
| capture `inactive`/`failed`, "repeated too quickly" | faulty unit in restart loop | fix the unit, then `sudo systemctl reset-failed marstek-capture && sudo systemctl restart marstek-capture` |
| 0-byte file | simply little traffic | generate traffic with a client, run `ls -l /var/captures/` again |
| Marstek connected, but only DHCP/ARP, no cloud traffic | still in the old WiFi / poll interval | `sudo systemctl restart hostapd` (force re-registration), otherwise wait |
| `marstek-ota-scan` finds nothing | `tshark` missing | `sudo apt install -y tshark`, then `sudo /usr/local/sbin/marstek-ota-scan.sh` |

**Reboot test:** `sudo reboot`, after boot-up run `sudo marstek-status.sh` — everything must be
`active` again by itself (all services + the OTA timer are `enabled`).

---

## Appendix A — Switch to USB Stick/SSD Later

```bash
sudo apt install -y exfatprogs           # only if exFAT
sudo mkdir -p /mnt/cap
lsblk -f                                 # find the stick (e.g. /dev/sda1) + FSTYPE
sudo blkid /dev/sda1                     # read the UUID
echo 'UUID=DEINE-UUID /mnt/cap exfat defaults,nofail,uid=root,gid=root 0 0' | sudo tee -a /etc/fstab
sudo mount -a && df -h /mnt/cap
```
Then change the path in the capture unit, status script **and OTA scanner**:
```bash
sudo sed -i 's#/var/captures#/mnt/cap#g' \
  /etc/systemd/system/marstek-capture.service \
  /usr/local/sbin/marstek-status.sh \
  /usr/local/sbin/marstek-ota-scan.sh
sudo systemctl daemon-reload && sudo systemctl restart marstek-capture
```
To change the retention: adjust the value `-mtime +21` in the OTA scanner (e.g. `+30` for 30 days).

---

## Appendix B — Breaking TLS (mitmproxy), for MQTT Push + HTTPS Downloads

**Optional only.** The `.rbl` itself arrives via HTTP and is captured by the auto-extractor anyway —
you only need mitmproxy if the download URL came **exclusively** via the MQTT channel (8883/TLS)
or an STM32 image is loaded via HTTPS.

**Basis:** The v150 analysis evidences `MBEDTLS_SSL_VERIFY_NONE` (server certificate is not
verified) → transparent TLS proxy possible. The live check additionally confirms this (`cert=0` in the
HTTP request). The device's client certificate is available extracted (`security/VNSD_Certs/`, local, not in the repo).

> The verify level of the Quectel/FC41D path is still unconfirmed. The attempt is the test: if flows
> arrive in mitmproxy → verification off. Reset/timeout → verification on, then roll back (B4). Fully reversible.

**B1 — Install + client cert:**
```bash
sudo apt install -y mitmproxy
cat marstek_device_client_cert.pem marstek_device_private_key.pem | sudo tee /root/marstek_client.pem >/dev/null
sudo chmod 600 /root/marstek_client.pem
# Copy the addon marstek_mitm_addon.py (local, not in the repo) from the project folder to /root/
```
**B2 — Redirect only for the Marstek IP** (adjust M):
```bash
M=192.168.50.90
sudo sysctl -w net.ipv4.conf.all.send_redirects=0
sudo iptables -t nat -A PREROUTING -s $M -p tcp --dport 443  -j REDIRECT --to-ports 8080
sudo iptables -t nat -A PREROUTING -s $M -p tcp --dport 8883 -j REDIRECT --to-ports 8080
```
**B3 — Start the proxy:**
```bash
sudo mitmdump --mode transparent --showhost --ssl-insecure --tcp-hosts '.*' \
     --set client_certs=/root/marstek_client.pem \
     -s /root/marstek_mitm_addon.py -w /var/captures/mitm_flows.mitm
```
Firmware ends up in `/var/captures/objects/`, OTA URL hits in `/var/captures/mitm_hits.log`.
**B4 — Rollback (always after the test):**
```bash
M=192.168.50.90
sudo iptables -t nat -D PREROUTING -s $M -p tcp --dport 443  -j REDIRECT --to-ports 8080
sudo iptables -t nat -D PREROUTING -s $M -p tcp --dport 8883 -j REDIRECT --to-ports 8080
# stop mitmdump with Ctrl+C
```
