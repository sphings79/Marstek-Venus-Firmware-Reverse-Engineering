# Marstek Venus D – Serial Number Structure

> 🇩🇪 Deutsche Version: [VNSD_Seriennummer_de.md](VNSD_Seriennummer_de.md)

Analyzed by means of Ghidra reverse engineering of the Control firmware  
(`Control_149.2_VNSD-0_app_1492_0702_142136.bin`)

---

## Format

```
MVD 1 0 4 25 42 00123
│   │ │ │  │  │   └── Sequence number within the batch (5 digits)
│   │ │ │  │  └────── Production week (2 digits, ISO week)
│   │ │ │  └───────── Production year, 2-digit (e.g. 25 = 2025)
│   │ │ └──────────── Plant/line code (1 digit)
│   │ └────────────── HW minor version (1 digit)
│   └────────────────  HW major version (1 digit)
└────────────────────  Model prefix (3 characters: Marstek Venus D)
```

**Total length: 15 characters**

---

## Examples

| Serial number     | HW rev | Plant | Year | CW | Seq. no. |
|-------------------|--------|------|------|----|-----------|
| MVD1042500123   | 1.0    | 4    | 2025 | 42 | 00123     |
| MVD1042502345   | 1.0    | 4    | 2025 | 42 | 02345     |

Both devices were produced in **CW 42 / 2025** (approx. 13–19 October 2025).  
The sequence numbers `00123` and `02345` are **xxxx units** apart.

---

## Field description

### Positions 1–3: Model prefix `MVD`
Stands for **M**arstek **V**enus **D**. Identifies the product line.  
The firmware internally uses `OTA_Is_VNSD_Model()` to check whether the model string stored in the EEPROM contains `"VNSD-0"` — this is exclusively an OTA compatibility check, not an SN validation.

### Position 4: HW major version
Only `1` observed so far.

### Position 5: HW minor version
Only `0` observed so far → hardware revision **1.0**.

### Position 6: Plant/line code
Only `4` observed so far. Presumably the production plant or manufacturing line.

### Positions 7–8: Production year
Two digits, without the century. `25` → **2025**.

### Positions 9–10: Production week
ISO calendar week. `42` → **CW 42**.

### Positions 11–15: Sequence number
Five-digit consecutive number within the batch (plant + year + week).

---

## SN in the firmware

The SN is **not validated** — it is read from the EEPROM and used unchanged:

- **Cloud telemetry**: as `sn=` in the query string, encrypted with AES-128-ECB + Base64,  
  sent to `http://<server>.hamedata.com/prod/api/v1/setVenusDReporting`
- **MQTT**: in the same telemetry format `di=%s&sn=%s&...&hd=%s&...`
- **BLE device name**: format `MST_VNSD_XXXX`, where the last 4 digits of the SN  
  are inserted (e.g. `MST_VNSD_1234` or `MST_VNSD_3456`)

The `hd=` field in the telemetry is a **separate hardware descriptor** from the EEPROM  
and is not derived from the SN.

---

## Analyzed firmware files

| File | Version | Type | Size |
|------|---------|-----|-------|
| `Control_149.2_VNSD-0_app_1492_0702_142136.bin` | 149.2 | Control (main processor) | 376 KB |
| `Control_147_202601281721320b2053125.bin` | 147 | Control | 492 KB |
| `Micro_VNS_116_vd_inv_app_0116_0702_ota_163439.bin` | 116 | Micro/inverter | 113 KB |
| `BMS_117.7_20251010135647565eb2036.bin` | 117.7 | BMS | 104 KB |

---

*Created with Ghidra analysis, July 2026*
