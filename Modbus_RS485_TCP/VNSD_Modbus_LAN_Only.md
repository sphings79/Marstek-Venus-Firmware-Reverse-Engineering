# Modbus TCP only exists via the LAN jack, the discovery beacon via both

> 🇩🇪 Deutsche Version: [VNSD_Modbus_nur_ueber_LAN_de.md](VNSD_Modbus_nur_ueber_LAN_de.md)

**Device:** Marstek Venus D (VNSD-0), communication module HME-3
**Firmware:** Control/EMS v150
**Path:** Modbus **TCP** (CH395) and UDP beacon 22222 → 12345
**Date:** September 22, 2026
**Status:** measured on the device, four network states incl. cold start · firmware rationale from
`Methodik_und_Meta/VNSD_Ghidra_Analysis_Findings.md` §5.1

**Scope:** measured exclusively on the Venus D. Whether Venus A and E v3 behave the
same way is **not** checked — hence the prefix in the file name.

---

## 1. Summary

A Venus D with a network cable has two IP addresses: one on the Ethernet chip CH395 and one on
the radio module HME-3. The two behave differently:

- **Modbus TCP only responds on the LAN address.** On the Wi-Fi address, port 502 is
  refused — during normal operation, after link loss and also after a **cold start
  without a cable**. The server does not follow along.
- **The discovery beacon, on the other hand, does follow along.** It always comes from exactly one
  interface, preferring LAN, and switches within about half a minute after link loss or link
  recovery. Never from both at once.

Practical consequence: autodiscovery via the beacon also finds a device **without** a cable
and in that case returns an address behind which no Modbus is listening.

---

## 2. The addresses

| | LAN | Wi-Fi |
|---|---|---|
| IP | 192.168.1.50 | 192.168.1.51 |
| MAC | `aa:bb:cc:dd:ee:02` | `aa:bb:cc:dd:ee:03` |
| DHCP hostname | `CH395` | — |
| Chip | CH395 (Ethernet) | HME-3 / FC41D (radio) |

The DHCP hostname of the LAN side literally reads `CH395` — directly visible in the
UniFi controller and thus a usable filter if someone wants to build DHCP-based detection.

The MAC `aa:bb:cc:dd:ee:01` carried in the beacon shares the OUI block `aa:bb:cc` with
the Wi-Fi MAC. The payload thus originates from the radio module, but is sent over whichever
interface is currently active — with the cable plugged in, via the CH395.

---

## 3. The measurement protocol

Passively listened on UDP 12345 with no deduplication at all, every single packet counted.
The port checks are pure TCP connects without a Modbus PDU.

### 3.1 Both interfaces up

```
Beacons, 15 s:   192.168.1.50  x30
                 192.168.1.51  x0
192.168.1.50:502  OPEN
192.168.1.51:502  ConnectionRefusedError
```

### 3.2 LAN link lost during operation

```
Beacons, 90 s:   192.168.1.51  x229
192.168.1.51:502  ConnectionRefusedError   (5 measurements over 84 s)
192.168.1.50      100 % packet loss
```

### 3.3 Cold start without LAN

```
Beacons, 60 s:   192.168.1.51  x166
192.168.1.51:502  ConnectionRefusedError   (3 measurements)
```

This rules out the possibility that the Modbus server is only absent from the Wi-Fi stack
because it was already bound to the CH395 at boot time.

### 3.4 LAN comes back

```
[  0s] Beacon source -> 192.168.1.51
[ 26s] Beacon source -> 192.168.1.50
Total over 5 min: {'192.168.1.51': 15, '192.168.1.50': 538}
```

A single switch, none after that. No overlap, no phase with both sources.

### 3.5 Counter-check with a guaranteed free slot

The Venus accepts exactly **one** Modbus connection. All previous measurements ran against
a device whose connection was held by a `modbus-proxy`. For the counter-check, the proxy
was stopped:

```
192.168.1.50:502  OPEN                    (2x, immediately)
192.168.1.51:502  ConnectionRefusedError  (2x)

FC03 against 192.168.1.50:
  30400 device_ip   -> 192.168.1.50
  30402 gateway_ip  -> 192.168.1.1
  32104             -> 54
```

Real FC03 responses, not just an open port. **Register 30400 reports the LAN address** —
the device itself names the address under which it speaks Modbus.

---

## 4. Why: the socket allocation of the CH395

From `Methodik_und_Meta/VNSD_Ghidra_Analysis_Findings.md` §5.1, initialization in
`CH395_Reset_And_Reinit @ 0x08029964`:

| Socket | Port | Function | Init |
|---|---|---|---|
| 0 | 502 | **Modbus TCP** | `CH395_Init_Modbus_TCP_Socket @ 0x08048754` |
| 1 | 8091 | TCP debug/CLI shell | `CH395_Init_TCPServer_Socket @ 0x08032C0C` |

The Modbus server is firmly bound to the Ethernet chip. On the Quectel/AT path of
the radio module there is no equivalent. This matches the measurement and also explains
why a cold start without a cable changes nothing about it.

**The beacon, on the other hand, is not explained by this.** The obvious candidate function
`FUN_08017dc0` (source port 22222, destination port 12345, CH395 UDP socket, see
`Control_FW/VNSD_Control_FW_Analysis_app_1492_0702_142136.md` line 1834) belongs to the
EM status poll, which sends `EM.GetStatus` to a local energy meter — and this
feature has, in v149.2, in **both** variants, Ethernet and Quectel, zero callers and is
unreachable. Who actually sends the beacon remains open.

---

## 5. Methodological pitfall: `ConnectionRefused` is ambiguous

An RST on port 502 can mean two completely different things:

1. **No listener** — the address belongs to the wrong interface.
2. **Slot occupied** — the device accepts exactly one connection, and a proxy or a
   second integration is already holding it.

Both produce the same packet. During this measurement series, the **LAN** address at times returned
`ConnectionRefused`, even though Home Assistant successfully read from it in the same second — case 2.

That is why the claim in this document does not rest on the port scan alone, but on two
independent states with a demonstrably free slot: §3.2/§3.3 (LAN unreachable, so
nobody could have been holding the connection) and §3.5 (proxy stopped).

For anyone measuring this in the future: read against the `modbus-proxy` instead of against the device. It
allows multiple clients and does not disturb the ongoing poll.

---

## 6. Side finding: register 30303 (`wifi_signal_strength`)

According to `BLE/VNSD_BLE_Modbus_CrossReference.md` line 42, 30303 is the field `wifiRssi` (u8, source
`DAT_0800b38c[1]`), displayed as the raw value × −1.

The opposite of what was expected was observed:

| State | 30303 |
|---|---|
| Wi-Fi **not** connected, device had been running on the cable for a long time | 40 → −40 dBm |
| Wi-Fi connected, device on the cable after a restart | 0, stable over 3 reads |

**Working hypothesis:** the value is only updated when the radio module is the active uplink
path. On the cable it stays at the last known value and, after a
restart, simply was never populated. The −40 before the restart would then be a stale value from
an earlier Wi-Fi phase.

The hypothesis **cannot be checked** with Modbus alone: to read 30303 you need LAN —
and with LAN, the radio module is not the uplink. A catch-22. It could be checked via BLE
(field 0x3D) on a device without a cable.

Practical consequence: on a wired Venus D, the `wifi_signal_strength` sensor returns
either 0 or a stale value. It is currently `enabled_by_default: true`.

---

## 7. Consequences for the HA integration

The autodiscovery of the Marstek Modbus Suite listens on the same beacon. From the measurements:

- **With the cable plugged in, the address found is always the correct one.** Only the
  LAN interface transmits, and that is exactly the one carrying Modbus. Actively probing several
  found addresses would be pointless — and harmful, because every connect attempt goes against the one
  slot the device has.
- **Without a cable, the device is found but cannot be set up.** The dialog offers an
  address that refuses port 502. Handled in `config_flow.py` via
  `async_is_connection_refused`: after a failed connection attempt, a single check is made
  whether the device is refusing rather than staying silent, and the error key
  `connection_refused` then names both possible causes from §5.
- **There are not two beacons for the same device**, not even during the switchover. The
  deduplication in `helpers/discovery.py` nevertheless runs on `(MAC, Host)` rather than on the MAC
  alone: the MAC belongs to the radio module and is the same on both interfaces, so a
  MAC-only key would, with changed firmware, silently keep the address that arrived first.

---

## 8. Open

- Who sends the beacon? It is not `FUN_08017dc0` (§4).
- Do §1–§4 also apply to Venus A and Venus E v3? Both share the Control firmware family,
  but only the D was measured.
- Meaning of register 30304 (stable at 13880 across all reads).
- Confirmation of the hypothesis on 30303 via BLE field 0x3D.
