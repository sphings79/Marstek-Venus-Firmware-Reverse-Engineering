# Device power class "Grid power" 800 / 2200 / 2500 W (Control FW v150)

> 🇩🇪 Deutsche Version: [VNSD_Netzleistung_Geraeteklasse_800_2200_2500_v150_de.md](VNSD_Netzleistung_Geraeteklasse_800_2200_2500_v150_de.md)

> **Firmware:** `VNSD-0_app_0150_0805_115146.bin` (v150)
> **Source:** `MQTT_JSON_RPC_Dispatcher` @0x0801ab6c (case 0x10) + `Config_Write_U16` @0x08006d30
> **Status:** 2026-08-14

---

## 1. What it is

The app selector "Grid power / device power" is **not a free 0–2500 W slider**,
but a **device power class** with exactly three permitted levels: **800 / 2200 / 2500 W**.
It acts on the **discharge/feed-in side** (German 800 W feed-in cap) and is set via the
**cloud (MQTT/JSON-RPC)** — not via a dedicated Modbus register.

## 2. The command (cloud / MQTT, case 0x10)

JSON-RPC command with field `version`. Only three values are valid, otherwise error `0x1AA`:

| `version` | hex | Tier flag → EEPROM 0x90 | Power value → EEPROM 0x204 | Additional action |
|---|---|---|---|---|
| 800 | 0x320 | `1` | 800 | **`Config_ScheduleEntries_ClampPower800W()`** — clamps all schedule slot powers to 800 W |
| 2200 | 0x898 | `3` | 2200 | — |
| 2500 | 0x9C4 | `0` | 2500 | — |

Sequence per level (decompiled):

```c
case 0x10:                                  // "set device power class"
  version = json["version"];
  if (version == 800)  { flag=1; Config_Write_U16(800);   EEPROM_Write(0x90,flag,1);
                         Config_ScheduleEntries_ClampPower800W(); }
  else if (version==0x898){flag=3; Config_Write_U16(0x898); EEPROM_Write(0x90,flag,1); }
  else if (version==0x9c4){flag=0; Config_Write_U16(0x9c4); EEPROM_Write(0x90,flag,1); }
  else  -> Error 0x1AA;
```

`Config_Write_U16(v)` (@0x08006d30): clamps `v` to max. 2500, on change writes
**EEPROM 0x204** (2 bytes) and into the SRAM config struct (+4).

## 3. Two persisted quantities

| What | Location | Meaning |
|---|---|---|
| Power value (800/2200/2500) | **EEPROM 0x204** | = backend of Modbus register **44003** (discharge/feed-in max) |
| Class/tier flag (1/3/0) | **EEPROM 0x90** | "selected class", which the app reads back |

## 4. Reproduction via Modbus (Home Assistant)

The exact cloud command is **not** available as a Modbus register. The effect can,
however, be reproduced in practice:

1. Unlock RS485: `42000 = 0x55AA` (FC06)
2. Set feed-in/discharge max: **`44003 = 800 | 2200 | 2500`** (FC06)
   - hits the same EEPROM 0x204 and pushes the value immediately to the inverter via **CAN reg 0x03**
   - (charge counterpart: `44002` → CAN reg 0x04, EEPROM 0x202)

**What a Modbus write to 44003 does NOT do:**

- The class flag in **EEPROM 0x90** is not set (the app may keep displaying the class unchanged).
- The schedule slots are **not** automatically clamped to 800 W. If needed, set the slot
  powers (43103/43108/43113/43118/43123/43128 or the 41600 array) to ≤800 yourself.

## 5. Direction note (charging vs. discharging) — code-proven

This class acts on the **discharge side** (44003 / EEPROM 0x204 / CAN func 0x03).
**Proven (v150 decompilation, 2026-08-14):**

- `Config_ScheduleEntries_ClampPower800W` clamps, in each schedule slot, only values
  `> +800` (field +2, signed). Since schedule power is signed (**positive = discharging/
  feeding in, negative = charging**), only the **discharge side** is capped →
  the 800 W selector is a feed-in/discharge limit.
- The same app command writes `Config_Write_U16` → **EEPROM 0x204 = register 44003** →
  **44003 = discharging**, by exclusion **44002 = charging**.
- Cross-check force_mode (`Remote_Power_Command_Execute`): `force_mode=1` (charging) uses
  SRAM field +4 = **reg 42020**, `force_mode=2` (discharging) field +6 = **reg 42021**.

This establishes with certainty: **42020/44002 = charging, 42021/44003 = discharging** and
**CAN func 0x03 = discharging, 0x04 = charging**. The earlier `Modbus_RS485_TCP/Scan_Logs/Write_Handler_Register_Map.csv`
had swapped both pairs — **corrected on 2026-08-14**.

## 6. References

- `MQTT_JSON_RPC_Dispatcher` @0x0801ab6c, case 0x10 (v150)
- `Config_Write_U16` @0x08006d30 (EEPROM 0x204), `Config_ScheduleEntries_ClampPower800W`
- `Config_Write_U16_0x202` @0x08006c64 (EEPROM 0x202 = charge max / 44002)
- Modbus 44002/44003 → Inverter_Write_Reg_0x04/0x03 (CAN reg 0x04/0x03)
