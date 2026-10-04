# Marstek Venus D — Modbus TCP Connection Reference

> 🇩🇪 Deutsche Version: [VNSD_Modbus_TCP_Verbindungsreferenz_de.md](VNSD_Modbus_TCP_Verbindungsreferenz_de.md)

**Purpose:** Technical reference for all scanner scripts and the HA integration.
Documents the connection parameters, protocol quirks and known pitfalls.

---

## 1. Connection parameters

| Parameter | Value | Note |
|---|---|---|
| **Protocol** | Modbus TCP (not RTU!) | Directly over Ethernet/Wi-Fi |
| **IP address** | `192.168.1.100` | Statically configured, readable from reg 30400–30403 |
| **Port** | `502` | Standard Modbus TCP |
| **Slave/unit ID** | `1` | Fixed, readable from reg 41100 |
| **Timeout** | 3–5 seconds | Device normally responds in <500ms |
| **Function code** | `0x03` (Read Holding) | Read all registers |
| **Function code** | `0x06` (Write Single) | Write a single register |
| **Function code** | `0x10` (Write Multiple) | Write multiple registers |

## 2. MBAP header (Modbus TCP frame)

```
Request:
┌─────────────┬──────────────┬────────────┬──────────┬─────────┬──────────────┬───────────┐
│ TID (2B)    │ Protocol (2B)│ Length (2B)│ Unit (1B)│ FC (1B) │ Reg Addr (2B)│ Count (2B)│
│ 0x0001      │ 0x0000       │ 0x0006     │ 0x01     │ 0x03    │ e.g. 0x7532  │ 0x0001    │
└─────────────┴──────────────┴────────────┴──────────┴─────────┴──────────────┴───────────┘

Response:
┌─────────────┬──────────────┬────────────┬──────────┬─────────┬────────────┬─────────────┐
│ TID (2B)    │ Protocol (2B)│ Length (2B)│ Unit (1B)│ FC (1B) │ ByteCnt(1B)│ Data (N×2B) │
└─────────────┴──────────────┴────────────┴──────────┴─────────┴────────────┴─────────────┘

Total response length = 6 + Length field  (NOT 9 + Length - 1!)
```

**MBAP bug fix (from scanner v7):** The correct formula for the total length of a
Modbus TCP response is `6 + Length field`. Older implementations had incorrectly
used `9 + Length - 1`, which led to parsing errors.

### Python implementation (raw socket, NO pymodbus)

```python
import socket, struct, select, time

def build_req(tid, slave, reg, count):
    """Builds a Modbus TCP FC03 request."""
    pdu  = struct.pack(">BBHH", slave, 0x03, reg, count)
    mbap = struct.pack(">HHH", tid & 0xFFFF, 0, len(pdu))
    return mbap + pdu

def recv_response(sock, timeout=3.0):
    """Receives a complete Modbus TCP response."""
    resp = b""
    deadline = time.time() + timeout
    while True:
        remaining = deadline - time.time()
        if remaining <= 0:
            return None
        ready = select.select([sock], [], [], min(remaining, 1.0))
        if not ready[0]:
            if time.time() >= deadline:
                return None
            continue
        chunk = sock.recv(512)
        if not chunk:
            return None  # Connection closed!
        resp += chunk
        if len(resp) >= 6:
            length_field = struct.unpack(">H", resp[4:6])[0]
            total = 6 + length_field   # CORRECT formula
            if len(resp) >= total:
                return resp[:total]

def parse(data, count):
    """Parses an FC03 response into a list of uint16 values."""
    if not data or len(data) < 9:
        return None
    fc = data[7]
    if fc & 0x80:        # Exception response
        return None
    if fc != 0x03:
        return None
    byte_count = data[8]
    n = byte_count // 2
    if n < 1 or len(data) < 9 + byte_count:
        return None
    return list(struct.unpack(f">{n}H", data[9:9 + byte_count]))
```

### Why raw socket instead of pymodbus?

- **pymodbus API incompatibility:** v2.x uses `unit=`, v3.x uses `slave=`, newer versions
  change the API again. On macOS/Homebrew the installed version is unpredictable.
- **No dependency:** raw socket only needs the Python standard library
- **Full control:** TID management, timeout handling, MBAP parsing exactly as needed
- **Tested:** scanner v7 with raw socket is the only reliably working variant

## 3. Critical device quirks

### 3.1 Batch-size hard limit: 32 registers

```
FC03 with count ≤ 32:  ✅ Normal response
FC03 with count > 32:  ❌ NO response (timeout, no exception code!)
```

When exceeded, the device returns neither an error nor an exception —
it simply does **not** respond. The TCP connection stays open, but the request
is silently ignored.

**Recommendation:** `BATCH_SIZE = 32` for all scanners.

### 3.2 Connection drop after exception

```
FC03 on an invalid register → exception response (FC 0x83, code 0x02)
→ device CLOSES the TCP connection!
→ reconnect required for the next request
```

**Strategy:** On an exception response, immediately `socket.close()` + reconnect with a 1.5–2s pause.
Do not attempt to send further requests on the same socket.

### 3.3 Register addressing: DIRECT (no offset)

```
PDU address = register number (NO zero-based conversion!)

Register 30000 → PDU addr = 30000 (0x7530)
Register 34002 → PDU addr = 34002 (0x84D2)
Register 42000 → PDU addr = 42000 (0xA410)
```

**Proof:** The TCPRouter in the Control FW (`FUN_0801c088`) compares the raw PDU address
directly with `descriptor.base_addr` without any correction.

In Python (raw socket): `build_req(tid, 1, 34002, 1)` reads register 34002.
In pymodbus: `read_holding_registers(34002, 1)` — NO `-1` or `-40001`!

### 3.4 Gaps in register ranges

The address space is NOT contiguously populated. A batch read across a gap results in
an exception. Solution: fall back to single reads on exception.

```
30000-30040:  partially populated (gaps at 30008-30009, 30011-30019, etc.)
32000-32999:  only 6 registers populated (32105, 32200, 32204, 32300-32302)
34000-34033:  contiguously populated (pack 1)
37000-37025:  partially populated
```

### 3.7 BMS pack layout (34000 + pack×100 + offset)

Each battery pack occupies a block of 100 (pack 1 = 34000, pack 2 = 34100, …,
pack 7 = 34600). Within the block:

| Offset | Register (pack N) | Field | Scale |
|---|---|---|---|
| +0 | 34N00 | bat_volt | ×0.01 V |
| +1 | 34N01 | bat_curr (signed) | ×0.1 A |
| +2 | 34N02 | bat_soc | **×0.1 %** |
| +3 | 34N03 | cycle_count | ×1 |
| +4 | 34N04 | **charge_status** (0=idle, 3=actively charging) | ×1 |
| +5 | 34N05 | **max_cell_v** | ×0.001 V |
| +6 | 34N06 | min_cell_v | ×0.001 V |
| +7 | 34N07 | max_ntc | ×0.1 °C |
| +8/+9 | 34N08/34N09 | protect1 / protect2 (bitmask) | hex |
| +10 | 34N10 | bms_version | ×1 |
| +11…+14 | 34N11–34N14 | ntc0–ntc3 | ×0.1 °C |
| +15/+16/+17 | 34N15–34N17 | mos_ntc / env_ntc / avg_ntc | ×0.1 °C |
| +18…+33 | 34N18–34N33 | cell1–cell16 | ×0.001 V |

> **Correction (2026-08-14):** `max_cell_v` is at **offset +5** (e.g. 34005),
> not +4. Offset +4 is the **charge_status** (values 0/3). Older scanner versions
> incorrectly placed `max_cell_v` at +4 and listed +5 as "unknown" — in the
> live scan (`control_150_vns_116.csv` (local, raw data)) the real max cell value is clearly at
> +5 (34005=3332 ≈ 3.33 V), while +4 = 0 or 3. Applies to all packs 1–7.
> **Packs 5 and 6 have been fully confirmed since the v150/VNS116 full scan.**

### 3.8 ASCII/string registers (device identity)

The firmware provides exactly **three** ASCII blocks via the read handler
(2 characters per register each, big-endian / high byte first). Source: Ghidra analysis
of the read handler (`Scan_Logs/Read_Handler_Register_Map.csv`).

| Register | Field | Length | Example decoding |
|---|---|---|---|
| 30304–30309 | **MAC address** | 6 reg | `AABBCCDDEE01` → `AA:BB:CC:DD:EE:01` |
| 30350–30355 | **Communication module FW version** | 6 reg | `202409090159` (build 2024-09-09 / 0159) ✅ fully read |
| 31000–31009 | **Device name** | 10 reg | `VNSD-0` + null padding |

Decoding in Python: `chr((raw>>8)&0xFF) + chr(raw&0xFF)` per register, concatenated
in register order.

> **Confirmed (2026-08-14):** The 30350 block was fully read via a targeted read
> (`Scan_Logs/regs_30350-30355.csv` (local, raw data)) and yields, without gaps,
> `20`+`24`+`09`+`09`+`01`+`59` = **`202409090159`**. In the v150 full scan, the
> isolated 30352 capture was shifted by one register (showed "24" instead of "09") —
> the targeted read is authoritative. Note: for single hits in an otherwise
> gappy batch region, it is better to re-read the whole block in a targeted way
> (`scan_registers.py --regs 30300-30355` (local, not in the repo)).

> **NOT readable via Modbus:** The 24-digit **device ID** (e.g.
> `000000000000000000000000`, cloud/app field `di=`) is **not** a Modbus register.
> The read handler offers only MAC and device name as device-specific identifiers;
> the long device ID is transmitted exclusively in the cloud/MQTT telemetry
> (`di=%s&sn=%s&…`) and is only reachable via the app/BLE or the cloud path.

### 3.5 Reconnect timing

| Situation | Wait time | Retry |
|---|---|---|
| Exception response | 1.5–2.0s | Reconnect immediately |
| Timeout (no response) | 2.0–3.0s | 1× retry, then skip |
| Socket error | 2.0s, then 3.0s | 2 attempts |
| Request interval | 150ms minimum | Too fast → timeout |

### 3.6 Request rate

```
Recommended: 150ms between requests (DELAY_S = 0.15)
Minimum:     100ms (works, but occasional timeouts)
Maximum:     unlimited (device has no upper rate limit)
```

## 4. RS485 write access (register ≥ 42000)

Before writing to control registers (42010–42021), RS485 mode
must be activated with an **unlock command**:

```
Register 42000 = 0x55AA  → RS485 control ACTIVE
Register 42000 = 0x55BB  → alternative activation
Register 42000 = 0x55EE  → RS485 control INACTIVE
```

**Writable afterward:**

| Register | Description | Value range |
|---|---|---|
| 42010 | force_mode | 0=None, 1=Charge, 2=Discharge |
| 42011 | charge_to_soc | 0–100 (%) |
| 42020 | set_charge_power | 0–2500 (W) |
| 42021 | set_discharge_power | 0–2500 (W) |

**Warning:** Write commands use FC06 (Write Single Register), NOT FC03.

## 5. Important scale factors

| Register | Raw value example | Scale | Result | Source |
|---|---|---|---|---|
| 34002 (SOC) | 146 | **×0.1** | 14.6% | BMS show_soc, FW-verified |
| 34005 (max_cell_v) | 3332 | ×0.001 | 3.332V | BMS, offset +5 (NOT +4!) |
| 34006 (min_cell_v) | 3330 | ×0.001 | 3.330V | BMS, offset +6 |
| 30205 (mppt_version) | 104 | ×1 | v104 | Micro/MPPT FW, v150 scan confirmed |
| 30100 (bat_volt) | 5114 | ×0.01 | 51.14V | BMS struct 0x4A |
| 30101 (bat_curr) | -11 (signed!) | ×0.1 | -1.1A | BMS struct 0x4C |
| 32200 (ac_volt) | 2398 | ×0.1 | 239.8V | Telemetry grid_volt |
| 32204 (ac_freq) | 499 | ×0.1 | 49.9Hz | Telemetry grid_pf |
| 34018-34033 (cells) | 3116 | ×0.001 | 3.116V | BMS Cell Volt[n] |
| 35000 (env_temp) | 300 | ×0.1 | 30.0°C | Micro ntc_inv |
| 33000-33001 (energy) | u32 | ×0.01 | kWh | Telemetry chrg_energy |
| 37004 (ac_current) | -498 (signed!) | ×0.01 | -4.98A | Micro grid_cur |

**Signed/unsigned warning:** Registers with current (A), power (W) or temperature (°C)
can have negative values. Raw value > 32767 → `value = raw - 65536`.

## 6. Local API — DO NOT ENABLE

```
⛔ UDP port 30000 (Local API) → DO NOT enable!
   Documented reports of permanently corrupted Modbus registers
   after activation. Also affects values that are not reset
   after a device reset.
```

## 7. Available scanner scripts

| Script | Purpose | Dependency |
|---|---|---|
| `scan_registers.py` | Single/flexible registers (also a full scan via `--regs 0-65535`), watch mode, CSV export, **live output + progress** | None (raw socket) |
| `scan_known_registers.py` (local, not in the repo) | Scans only registers from the known register map CSV (`--regmap`) | None (raw socket) |
| `scan_continuous.py` (local, not in the repo) | Continuous loop over known registers (`--interval`, default 10s) | None (raw socket) |
| `scan_powercycle.py` (local, not in the repo) | Scan across power-cycle events | None (raw socket) |

**Note (2026-07-15):** The `scan_modbus_batch.py` and `test_batch_limit.py` scripts previously listed here
no longer exist in the project folder — presumably early prototype names that were
replaced by the scripts above. The batch-size limit (32 registers, see 3.1) is already documented as a
fact and no longer needs a dedicated test script.

### 7.1 `scan_registers.py` — output behavior (as of 2026-08-14)

- **Live output:** Every value found is printed to `stdout` **immediately upon being read**
  (previously only after the complete scan — for large ranges this meant
  no output at all for minutes). In addition, a single-line progress indicator runs on
  `stderr` (`Batch x/y @ from-to  found=n`), which keeps running even during
  timeout gaps. Can be disabled with `--no-progress`.
- **Timestamp:** The `timestamp` column in the CSV now contains the point in time
  at which the respective register was **actually queried** (captured per batch) —
  no longer a single timestamp from the end or abort of the run. For long
  scans, the time therefore increases across the rows.
- **Redirecting:** For a clean live log without the `stderr` progress:
  `python3 scan_registers.py … 2>/dev/null` or with `--no-progress`. For a
  complete log: `python3 scan_registers.py … | tee scan.log`.

### 7.2 `scan_registers.py` — names, tiers & full decoding (as of 2026-08-14b)

- **Full names from the register map:** The script automatically loads
  `VNSD_Register_Map_Final_generated.csv` at startup (searches the
  script folder, `../Modbus_RS485_TCP/` and CWD; override with `--regmap PATH`) and
  uses it to label **all** documented registers — no longer just the hard-
  wired fallback set. Without the map it still runs, but then only names the
  built-in KNOWN registers.
- **Confidence tiers in the output:** known/verified registers appear with a
  clear name, **suspected ones with the prefix `Verm:`**, **unknown ones with `Unb:`**
  (derived from the confidence column of the map: ✅/pack pattern → known,
  🔍/📊/scan → Verm, ❓/🆕/`unknown_*` → Unb). The header shows the tier count of the
  range. `--unknown-only` now shows only Verm + Unb.
- **Host/port/slave configurable:** `--port` (default 502), `--host`, `--slave`.
  All three also have an env default so you do not have to type them on
  every call: `MARSTEK_HOST`, `MARSTEK_PORT`, `MARSTEK_SLAVE`. A CLI flag
  overrides the env variable. Example: `export MARSTEK_HOST=192.168.1.50`
  then `python3 scan_registers.py --tiers verm,unb --out unklar.csv` is sufficient.
- **Scan specifically by tier (`--tiers`):** selects the registers directly from the map,
  without specifying a register range. `--tiers unb` = only the unknown ones, `--tiers
  verm,unb` = all not-finally-confirmed ones (currently ~130), `--tiers verm` = only the
  suspected ones. Can be combined with `--regs`/`--file` (union). Ideal for observing
  exactly the open registers — best with `--watch` under changing load.
- **Full decoding for guessing (all FW types):** For every value, all
  types that the FW's `Read_Serializer` also knows are computed —
  `u8(hi/lo) · i8(hi/lo) · u16 · i16 · hex · bin · ASCII(BE/LE) · BCD · ÷10 ÷100 ÷1000`
  as well as, **with the following register**, the 32-bit combos `u32 · i32 · float32` in both
  word orders (`*_next_be` = this register as the high word, `*_next_le` =
  the following register as the high word). The `f32` columns filter out implausible bit patterns
  (NaN/inf/absurd magnitude) as `-`, so that a real float pair
  stands out immediately with a sensible value. In the terminal, the decode line appears by
  default for `Verm:`/`Unb:` registers (`--decode-all` for all, `--no-decode`
  to turn it off); **the CSV always contains all decode columns**. This makes it possible to
  recognize the pattern directly for unknown registers: `u32_next_le` at 33000/33001 = kWh counter,
  `f32_next_le` for a register pair = an IEEE754 measurement, `ascii_be` = a string.

> **Note on the firmware (2026-08-14):** The Modbus FC03 read is served via a
> descriptor table built at runtime (SRAM 0x20000354) and `Read_Serializer`;
> each register has a type there (u8/u16/u32/i8/i16/i32/float/ASCII) +
> scale + source pointer, and the serializer **copies only the register width** — wide/
> float sources are thus truncated to 16 bits (this explains, e.g., the discharge value
> at 32101). The table is unpacked at boot from an encoded flash init structure
> (~0x0805dcb0); the exact register→type mapping is not
> trivially extractable statically (it would only exist ready-made in the live SRAM table). This is why
> the scanner does full decoding: it replaces the missing type info by "show all variants".

## 8. Quick start for a new scan

```bash
# 1. Full scan with the current FW status
python3 scan_registers.py --host 192.168.1.100 --regs 0-65535 --out scan_v149.2.csv

# 2. Check specific registers
python3 scan_registers.py --host 192.168.1.100 --regs 30000-30040,32200,34002

# 3. Investigate unknown registers
python3 scan_registers.py --host 192.168.1.100 --regs 30002-30005,30028-30036

# 4. Live monitoring (e.g. observe SOC while charging)
python3 scan_registers.py --host 192.168.1.100 --regs 34002,30001,32200 --watch 5

# 5. Scan a new register range (e.g. the 32000s) specifically
python3 scan_registers.py --host 192.168.1.100 --regs 32000-32999 --out scan_32k.csv

# 6. ONLY the unknown + suspected registers (selected from the map) — with full decoding
python3 scan_registers.py --host 192.168.1.50 --tiers verm,unb --out unklar.csv

# 7. Only the completely unknown ones, watched live under load (which values change = interpretable)
python3 scan_registers.py --host 192.168.1.50 --tiers unb --watch 10 --out unklar_watch.csv
```

---

*As of: 2026-08-14 | Firmware Control v150 / VNS v116 / BMS v118 | Device: Marstek Venus D (VNSD-0)*

*Changes 2026-08-14: BMS pack layout corrected (max_cell_v at offset +5, charge_status at +4),
packs 5+6 confirmed, 30205 = mppt_version confirmed, `scan_registers.py` with live output + real
query timestamps.*

---

## Maximum FC03 block size (measured 2026-08-22)

The scan scripts use `BATCH_SIZE = 32`, the Home Assistant integration
groups up to 125. The range in between had never been tested. Measurement on
Control v150 via `Scripts/test_block_size.py` (local, not in the repo), range starting at register 46501,
three runs each with a liveness probe on 37012 after every attempt:

| Registers in block | Result | Average |
|---|---|---|
| 8 | 3× ok | 15 ms |
| 16 | 3× ok | 6 ms |
| 24 | 3× ok | 26 ms |
| 30 | 3× ok | 6 ms |
| 32 | 3× ok | 8 ms |
| 36 | 3× ok | 5 ms |
| 40 | 3× ok | 26 ms |
| 44 | 3× ok | 7 ms |

**No error, no connection drop up to 44 registers.**

Notable: the response time does **not** grow with the block size — 44 registers
take 7 ms, 8 registers 15 ms. The outliers at 24 and 40 (26 ms each) are
network jitter, not a size effect. The limiting factor is the
round-trip time, not the payload.

`BATCH_SIZE = 32` in the scan scripts is thus a precaution, not a
device limit. For the integration, grouping at 125 remains safe;
larger blocks than 44, however, remain untested.
