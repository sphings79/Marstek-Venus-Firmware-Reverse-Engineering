# Marstek Venus D — Reverse Engineering: Methodology & Findings

> 🇩🇪 Deutsche Version: [Reverse_Engineering_Methodik_de.md](Reverse_Engineering_Methodik_de.md)

**Project:** Marstek Venus D (`VNSD-0`) reverse engineering  
**Reference:** Moved out of [VNSD_Control_FW_Analysis_app_1492_0702_142136.md](../Control_FW/VNSD_Control_FW_Analysis_app_1492_0702_142136.md) (formerly section 13)

---

## 1. Obtaining the Firmware

### Option A — Marstek FW Checker Tool (recommended)

The simplest method is the **Marstek FW Checker**, a community tool
by Remko Weijnen. An adapted, self-hosted variant is available as source code at:

```
https://github.com/sphings79/marstek-fw-checker
```

**How it works:**

1. Log in with Marstek cloud credentials (same as in the Marstek app)
2. All registered devices are listed
3. For each device: click **"Check Firmware"** — available versions on the Marstek server are displayed
4. A **"📥 Download" button** appears for each firmware component — it downloads the `.bin` file directly
5. In the **"Archive Status"** area, each component shows either:
   - **"✅ Archived"** — version already stored in the community archive
   - **"📥 Not Archived"** + button **"Submit for Archive"** — version not yet archived

**Donating firmware — why and how:**

Marstek occasionally deletes older firmware versions from the server. Anyone who clicks
**"Submit for Archive"** automatically creates a GitHub issue in the
archive repository. The firmware is then backed up and remains permanently available to the
community — even if Marstek removes it from the server.

> Please click **"Submit for Archive"** individually for each displayed component (BMS, Control, Inverter)
> if the status shows "Not Archived".

**Supported device types:**

| Device type | Description | Firmware components |
|---|---|---|
| **`VNSD-0`** | **Venus D** (this device) | Control, Micro (in OTA since July 2026) |
| `VNSA-0` | Venus A | BMS, Control, Inverter (micro) |
| `VNSE3-0` | Venus E V3 | BMS, Control |
| `HMG-50` | Venus E V1/V2 | BMS, Control |
| `HMG-25` | Venus E V1/V2 small | BMS, Control |
| `HME-4` / `HME-3` | AC-coupled devices | own OTA endpoint |

**What the tool delivers per component:**

| Firmware type | Description | Example VNSD-0 |
|---|---|---|
| Control (EMS) | Main controller firmware | v149 |
| Micro (Inverter) | Inverter firmware | v116 |
| BMS | Battery Management System | — |
| MPPT | Solar charge controller (if present) | — |

**Adaptations of this instance compared to the original:**

- Supports `VNSD-0` (Venus D) among others, as well as further Marstek device types
- Archiving via GitHub: https://github.com/sphings79/marstek-firmware-archiv
- Third download button for inverter firmware (micro) added
- Runs as a self-hosted Node.js service behind Apache (no Netlify)

> **Note:** The login logs the device out of the Marstek app — both
> sessions cannot be active at the same time. After the tool login,
> simply log back into the app.
>
> **Privacy & security:** The tool does not store any credentials —
> login data is held in the browser only for the duration of the session and forwarded directly to the Marstek server. The source code
> of the adapted version in use is publicly viewable:
> https://github.com/sphings79/marstek-fw-checker  
> If you still have concerns, you should change the Marstek password after using
> the tool.

> **VNSD-0 (Venus D):** Listed in the OTA system since July 2026 (control v149, micro v116).
> The FW Checker tool displays these components directly.

---

### Option B — Intercepting OTA Traffic (Wireshark)

If the device fetches an update or the OTA server is queried directly:

```bash
# Intercept OTA traffic via Wireshark (HTTP, unencrypted)
# Server: eu.hamedata.com
# Filter: http && tcp.port == 80

# Verify binary
xxd VNSD-0_app_1492_0702_142136.bin | head -5
# Line 1: 18 f3 01 20 71 4a 00 08 → IVT start (stack pointer + reset handler)
strings firmware.bin | grep -E "Task_|FreeRTOS|marstek"
```

### Option C — Direct OTA API Query

```bash
# Example: VNSD-0 firmware check (m=100 = retrieve all versions)
curl "https://eu.hamedata.com/ems/api/v2/checkSmallBalconyOTA\
?uid=<devid>&device_type=VNSD-0&m=100&sbv=0&mppt=0&inv=0\
&token=<auth-token>&mailbox=<email>" | python3 -m json.tool

# Extract download URL from the response:
# data.control.url → Control firmware
# data.micro.url   → Inverter firmware
```

## 2. Ghidra Configuration (mandatory)

```
Import: Raw Binary
Language: ARM → Cortex → 32 → little → default
Base Address: 0x08000000

Additionally add memory map:
  SRAM:   0x20000000  Size 0x20000  R+W
  PERIPH: 0x40000000  Size 0x10000000  R+W
  SCS:    0xE000E000  Size 0x1000  R+W

Auto-Analyze: enable Decompiler Switch Analysis + Scalar Operand References
```

## 3. Modbus Routing Scheme

```
Incoming request:
    FC code == 6 (Write Single)?
        uVar6 >= 40000  →  Write handler FUN_0804c83c
        uVar6 <  40000  →  Search descriptor table in FUN_0804b73c
    FC code == 3 (Read)?
        Iterate descriptor table, call serializer
```

## 4. Why the Descriptor Table Was Not Found in Flash

The table is built at runtime by init code. Attempts to find the source table:

| Strategy | Method | Result |
|---|---|---|
| Static flash search with SRAM ptr | v4 strategy A | 0 hits |
| Without SRAM ptr (monotonically increasing) | v4 strategy B | 0 hits |
| Startup literal pool | v6 | 0x08055d4c = MQTT strings |
| Follow MOVW #0x7530 | v5 | Leads to JSON parser (wrong!) |
| STR with offset 0x2F8 | v7 strategy A | 0 hits |
| MOVW #0x02F8 (table address) | v7 strategy B | Not found |
| 6-byte stride table | v4 strategy D | 0 hits |

**Conclusion:** The init code loads the table address indirectly:
```arm
LDR R0, [PC, #literal_pool_offset]  ; loads SRAM base address
ADD R0, R0, #0x2F8                  ; computes table address
; → neither MOVW #0x02F8 nor MOVT #0x2000 appears in the code
```
