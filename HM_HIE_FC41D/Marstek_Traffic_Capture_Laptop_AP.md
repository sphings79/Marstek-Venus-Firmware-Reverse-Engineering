# Capturing Marstek Traffic Permanently — Laptop as WLAN Access Point

> 🇩🇪 Deutsche Version: [Marstek_Traffic_Capture_Laptop_AP_de.md](Marstek_Traffic_Capture_Laptop_AP_de.md)

**Goal:** The Marstek connects to an old laptop (its own WLAN AP), the laptop forwards
to the internet and records all traffic in **daily rotating files**. Runs unattended for weeks.
Captures DNS (which host), HTTP downloads (full URL + file) and MQTT timing.

> **Why this is enough:** The module firmware download (FC41D `.rbl`) runs over **plaintext HTTP** —
> the complete URL and the file are in the clear in the capture, no matter which host the URL
> pushed via MQTT points to. No TLS break needed. DNS via the laptop additionally shows every host.

---

## 0. Hardware

- **Old laptop running Linux** (Ubuntu Server 24.04 or similar). Keep the power supply connected, disable suspend on lid close.
- **Upstream to the internet via Ethernet cable** laptop → router. This is the most stable and leaves the
  WLAN free for the AP.
- **WLAN adapter in AP mode.** The built-in card often cannot do this cleanly. Reliable and cheap:
  a **USB stick with Atheros AR9271** (driver `ath9k_htc`, 2.4 GHz) — exactly right, because the
  FC41D module only transmits on **2.4 GHz**. Alternatives: MT7612U, RTL8812AU (with driver).
- Some storage: an idle Marstek generates roughly 20–60 MB/day; a few GB are enough for weeks.

Check whether the adapter supports AP mode:
```bash
iw list | grep -A10 "Supported interface modes"   # must contain "AP"
```

---

## 1. Install software

```bash
sudo apt update
sudo apt install -y hostapd dnsmasq tcpdump tshark iw
sudo systemctl unmask hostapd
```

Note the interface names (e.g. upstream `eth0`/`enp1s0`, AP WLAN `wlan0`):
```bash
ip link
```

---

## 2. Give the AP interface a fixed IP

`/etc/systemd/network/10-ap.network` (or via netplan). Example subnet 192.168.50.0/24:
```bash
sudo ip addr add 192.168.50.1/24 dev wlan0
sudo ip link set wlan0 up
```

---

## 3. hostapd — set up the access point

`/etc/hostapd/hostapd.conf`:
```
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
```
Start test (foreground, Ctrl+C to stop):
```bash
sudo hostapd /etc/hostapd/hostapd.conf
```

---

## 4. dnsmasq — DHCP + DNS logging

`/etc/dnsmasq.conf` (comment out existing lines, put this at the end):
```
interface=wlan0
bind-interfaces
dhcp-range=192.168.50.50,192.168.50.150,12h
dhcp-option=3,192.168.50.1      # Gateway = laptop
dhcp-option=6,192.168.50.1      # DNS   = laptop (dnsmasq then logs every query)
log-queries
log-facility=/var/log/marstek-dns.log
```
```bash
sudo systemctl restart dnsmasq
```
This puts **every** domain queried by the Marstek into `/var/log/marstek-dns.log` — so the
host detection runs along the way, independent of the MQTT URL.

---

## 5. Forwarding + NAT (so the Marstek gets onto the internet)

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```
(`eth0` = adjust to your upstream interface.) Make it permanent with `iptables-persistent`.

Now set up the WLAN **MarstekCap** in the Marstek via the app.

---

## 6. The capture — daily rotating files

Exactly what you want — one file per day, named automatically:
```bash
sudo mkdir -p /var/captures
sudo tcpdump -i wlan0 -n -s 0 \
     -G 86400 -w '/var/captures/marstek_%Y-%m-%d_%H%M%S.pcap' -Z root
```
- `-G 86400` → a new file every 24 h (86400 s = 1 day).
- `%Y-%m-%d_%H%M%S` → the date is part of the file name (`marstek_2026-08-20_000000.pcap`).
- `-s 0` → full packets (required to get the firmware file completely).
- `-Z root` → drops privileges after opening.

**Hourly** instead of daily: `-G 3600`. **By size** (e.g. 200 MB): `-C 200` instead of `-G`.

Only relevant traffic (smaller files) — DNS, HTTP, MQTT:
```bash
... 'udp port 53 or tcp port 80 or tcp port 1883 or tcp port 8883'
```
(The telemetry on 443 is then dropped; the FC41D download runs on 80, however, and stays in.)

---

## 7. Run permanently as a service

`/etc/systemd/system/marstek-capture.service`:
```
[Unit]
Description=Marstek traffic capture
After=network.target

[Service]
ExecStart=/usr/bin/tcpdump -i wlan0 -n -s 0 -G 86400 -w /var/captures/marstek_%%Y-%%m-%%d_%%H%%M%%S.pcap -Z root
Restart=always

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now marstek-capture
```
(The `%` must be doubled in the systemd file — already done above.)

Clean up old files automatically (e.g. older than 30 days), daily via cron:
```bash
echo '0 4 * * * root find /var/captures -name "marstek_*.pcap" -mtime +30 -delete' \
  | sudo tee /etc/cron.d/marstek-cleanup
```

---

## 8. Analysis — find the update and extract the file

**Quickly check whether there was an OTA download on a given day** (HTTP GETs for .rbl/.bin):
```bash
tshark -r marstek_2026-08-20_000000.pcap -Y 'http.request' \
  -T fields -e ip.dst -e http.host -e http.request.full_uri | grep -iE '\.rbl|\.bin|ota'
```

**Export the firmware file directly from the capture:**
```bash
mkdir out && tshark -r <datei>.pcap --export-objects http,out/
ls -l out/            # the .rbl / .bin ends up here
```

**New/suspicious domains over time** (from the DNS log):
```bash
grep query /var/log/marstek-dns.log | awk '{print $6}' | sort | uniq -c | sort -rn
```

As soon as you have an `.rbl` or `.bin`: put it into the project folder — then read out the header, build date,
module ID and version (as with the previous FC41D file) and compare it with the previous status.

---

## Cheat sheet for the update moment

1. Capture is running (step 7 as a service) — just let it run for days/weeks.
2. When the app offers the module update: **trigger it** while the laptop is capturing.
3. Then analyze the daily `.pcap` with step 8 → URL + file are in there.
4. Check the DNS log in parallel in case the URL points to an unexpected host.

---

## Appendix: Raspberry Pi 3B instead of a laptop (recommended for continuous operation)

The Pi 3B is better suited for unattended long-term capture than a laptop:
silent, ~3 W, headless, always on. And it needs **no USB WLAN stick**, because it has two
separate interfaces.

**Roles:**
- `eth0` (Ethernet) = upstream to the router
- `wlan0` (built-in WLAN, 2.4 GHz only → matches the FC41D) = access point for the Marstek

**Setup:** Flash Raspberry Pi OS **Lite** (headless), enable SSH. After that the instructions
above apply **unchanged** — same packages (`hostapd dnsmasq tcpdump tshark`), same configs, same
interface names (`eth0` upstream, `wlan0` AP). Use `eth0` as upstream in `iptables`.

**Important — do not write to the SD card:**
Continuous pcap writing wears out the SD card and space is tight. Put the capture on a
**USB stick / a USB SSD**:
```bash
# Mount the USB medium (example):
sudo mkdir -p /mnt/cap
sudo mount /dev/sda1 /mnt/cap          # check the device name with 'lsblk'
# add permanently to /etc/fstab
```
In the tcpdump command or in the systemd service, then use `/mnt/cap/...` as the target path instead of `/var/captures/...`.

**Minor notes:**
- Onboard WLAN in AP mode is occasionally finicky under load — with a single device practically
  never an issue. If it is: plug in an AR9271 USB stick as `wlan1` and use that as the AP
  (`interface=wlan1` in hostapd.conf); the onboard WLAN stays unused.
- Do the heavy analysis (`tshark --export-objects` over large pcaps) on the Mac instead —
  the Pi 3B with an SD card is slow for that. Just pull the pcaps over with `scp`.
- Use a power supply with enough current (officially 2.5 A), otherwise there are brownouts
  under WLAN load (lightning-bolt symbol / undervoltage in `dmesg`).
