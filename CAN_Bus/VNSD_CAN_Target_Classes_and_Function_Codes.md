# Internal CAN Bus: Target Classes and Function Codes

> 🇩🇪 Deutsche Version: [VNSD_CAN_Zielklassen_und_Funktionscodes_de.md](VNSD_CAN_Zielklassen_und_Funktionscodes_de.md)

**Status 2026-08-22.** Static analysis of three images:
`VNSD-0_app_0150_0805_115146.bin` (Control v150),
`Micro_VNS_116_vd_inv_app_0116_0702_ota_163439.bin` (inverter v116),
`BMS_118_20260119100535e43806957.bin` (BMS v118).

This document describes the **internal** CAN bus between the MCUs. Not to be
confused with the external Modbus (TCP/RS485) or the internal RS485/UART
links — see the terminology warning in the project README.

---

## 1. Arbitration ID Layout

All send sites in the Control go through `CAN_Build_Arbitration_ID`
(`0x08032F20`):

```c
return (param_4 & 0xfff0ffff) + 0x10000 & 0xe00f0000 | param_1 & 0xff |
       (param_2 & 0xf) << 8 | (param_3 & 0xf) << 0xc | (param_4 & 0x1f) << 0x18;
```

| Bits | Field | Meaning |
|---|---|---|
| 0–7 | `param_1` | **Function code** |
| 8–11 | `param_2` | Source / node index |
| 12–15 | `param_3` | **Target class** |
| 16–19 | derived from `param_4` | Sequence (`+0x10000`, masked with `0xe00f0000`) |
| 24–28 | `param_4 & 0x1f` | Counter / flags |

Even with `param_4 = 0`, bit 16 stays set: `(0 + 0x10000) & 0xe00f0000 =
0x10000`. Every ID built this way is therefore **> 11 bits**, i.e. an extended frame.

Example `Inverter_Set_Schedule_Reg` with `(0x30+idx, 1, 4, 0)`:

```
ID = 0x10000 | (4 << 12) | (1 << 8) | (0x30+idx) = 0x00014130 + idx
```

The receiver independently confirms the layout: the BMS dispatcher checks
`(ushort)id >> 0xc == 3` (target class) and `(id & 0xfff) >> 8` (node index).

---

## 2. Target Classes

46 callers of `CAN_Build_Arbitration_ID` in Control v150:

| Target class | Node | Send sites | Evidence |
|---|---|---|---|
| **4** | Inverter (Micro/VNS) | 34 | `EMS_Inverter_CAN_Dispatcher` handles exactly this code group |
| **3** | BMS | 5 | `CAN_RX_Command_Dispatcher` requires target field `== 3` |
| **2** | RS485 path to the inverter | 13 | exclusively `Inverter_RS485_*` functions |
| **1** | Control itself | 1 | only `Debug_Mode_Set(0xC2, 0, 1)` |

`Debug_Mode_Set` sends `0xC2` to all four classes (1, 2, 3, 4) — a
broadcast pattern that additionally supports the class assignment.

---

## 3. Function Codes per Target Class

### Target 4 — Inverter

Sent by the Control:

| Code | Send site |
|---|---|
| `0x01` | `Inverter_Power_Setpoint_Calc` |
| `0x02` | `Config_Mode_Apply` |
| `0x03` | `Inverter_Write_Reg_0x03_U32Value` |
| `0x04` | `Inverter_Write_Reg_0x04_U32Value` |
| `0x05` | `Inverter_Set_Flag_Reg_0x05` |
| `0x06` | `Inverter_Set_Flag_Reg_0x06` |
| `0x07` | `Inverter_Write_Reg_0x07_Value` |
| `0x08` | `Inverter_Set_BLEMode_Reg_0x08` |
| `0x0A` | `Config_Param0A_Relay` |
| `0x10` | `FUN_080289DA` |
| `0x11` | `FUN_080289B8` |
| `0x12` | `Inverter_Clear_Reg_0x12` |
| `0x13` | `Inverter_Clear_Reg_0x13` |
| `0x14` | `FUN_08029964` |
| `0x16` | `FUN_0802D208` |
| `0x17` | `FUN_0802D27C` |
| `0x30`–`0x3E` | `Inverter_Set_Schedule_Reg` (register block 46500–46544) |
| `0x40`–`0x43` | `FUN_0802D2F4` |
| `0x50` | `Config_Feature_Enable` |
| `0x51` | `Config_Param51_Reset` |
| `0x52` | `Config_WorkMode_Set` |
| `0x53` | `Config_Param53_Activate` |
| `0x54` | `Config_PostWriteCommit` |
| `0x55` | `Config_Param55_Set` |
| `0x56` | `Config_PowerSetpoint_Write` |
| `0x57` | `Inverter_Set_Flag_Reg_0x57` |
| `0x58` | `Inverter_Set_PowerLimit_Reg_0x58` |
| `0x60` | `Inverter_Set_WorkMode_Reg_0x60` |
| `0xC1` | `Config_Counters_Reset` |
| `0xC2` | `Debug_Mode_Set` |
| `0xCB` | `FUN_08029940` |

Handled by the inverter (`EMS_Inverter_CAN_Dispatcher`, `0x08001940`,
27 codes):

```
0x01 0x02 0x03 0x04 0x05 0x06 0x07
0x10 0x11 0x12 0x13 0x16
0x50 0x51 0x52 0x53 0x54 0x55 0x56 0x57 0x58 0x60
0xC1 0xC2 0xC3 0xCB 0xCE
```

The dispatcher reads the code as `param_1 & 0xff` — the same bit position the
builder fills. There is **no default branch**: an unknown code falls out the
bottom without any reaction, without an error and without an ACK.

### Target 3 — BMS

Sent by the Control: `0x0A` (`Register_ClearOnWrite`), `0xC1`
(`Config_Counters_Reset`), `0xC2` (`Debug_Mode_Set`), `0xC4`
(`Config_Write_Category0xC4`), `0xCF` (`FUN_0800653C`).

Handled by the BMS (`CAN_RX_Command_Dispatcher`, `0x0800C928`, 8 codes):

```
0x01 0x02 0x03 0x04 0x05 0x0A 0xC4 0xCE
```

Order of the checks in the BMS — target field first:

```c
if ((ushort)local_10 >> 0xc == 3) {              // 1. target class must be 3
    bVar1 = (byte)local_10;                      // 2. only then the function code
    if (((local_10 & 0xfff) >> 8 == *DAT_0800CA74) ||   // 3. own node index
        ((local_10 & 0xfff) >> 8 == 0xf) ||             //    or broadcast 0xF
        ((local_10 & 0xfff) >> 8 == 0)) { ... }         //    or 0
```

A message with the wrong target class is discarded **before** the function
code is even read.

### Second BMS Receive Path — Fixed ID Scheme

`CAN_RX_Handler` (`0x08005A40`) uses a different, hard-coded scheme and
compares full IDs instead of fields:

| ID | Effect |
|---|---|
| `0x100101AA` | `CAN_CMD_01_Handler` |
| `0x100201AA` | `CAN_CMD_02_Handler` |
| `0x100301AA` | Balancing register (`KA495XX_Write_Balancing_Register`) |
| `0x100401AA` | `CAN_CMD_04_Handler` |
| `0x180102AA` | Reference ID, offsets `+0x100/+0x200/+0x300/+0x400/+0xFD00` |

The low half-word is fixed (`0x01AA` / `0x02AA`), the command number sits
in bits 16–19. This scheme is **not** that of the Control builder and belongs
to the communication between the battery packs. Both paths hang off
`APP_CAN_RX_Task_Process`.

---

## 4. Coverage Gaps

The reconciliation of sender ↔ receiver does **not** match completely.

### Sent, but not handled by the receiver

| Target | Codes | Note |
|---|---|---|
| 4 | `0x08`, `0x0A`, `0x14`, `0x17` | Single commands without a branch in Micro v116 |
| 4 | `0x30`–`0x3E` | Schedule block, source: Modbus registers 46500–46544 |
| 4 | `0x40`–`0x43` | `FUN_0802D2F4`. The Control has receive buffers for them, they stayed empty (see below) |
| 3 | `0xC1`, `0xC2`, `0xCF` | BMS dispatcher knows `0xCE`, but not `0xCF` |

**23 of the codes sent to target 4 have no receiver in v116.** This is
not an edge case but a consistent pattern.

### Handled, but never sent by the Control

| Target | Codes | Likely explanation |
|---|---|---|
| 4 | `0xC3`, `0xCE` | `0xCE` is sent by `Register_WriteCategory0xCE` with a variable target class |
| 3 | `0x01`–`0x05`, `0xCE` | Pack-to-pack traffic: the sender is the master pack, not the Control |

### Opposite Direction: `0x40`–`0x43` Arrive at the Control

The Control is also the **receiver** for these codes. `CAN_FrameDispatcher` →
`Telemetry_Register_Dispatcher` (`0x0802FD9C`) reads the code as `id & 0xff`
and passes it on to `Telemetry_Store_EnergyCounters` (`0x0802FD38`):

```c
if      (param_1 == '@') { /* 0x40 -> 8 bytes to 0x20000168 */ }
else if (param_1 == 'A') { /* 0x41 -> 6 bytes to 0x20000170 */ }
else if (param_1 == 'B') { /* 0x42 -> 8 bytes to 0x20000176 */ }
else if (param_1 == 'C') { /* 0x43 -> 8 bytes to 0x2000017E */ }
```

These buffers are readable via Modbus registers `38000`–`38014`, and a
cross-reference run confirms: `Telemetry_Store_EnergyCounters` is the
**only** writer. This makes it possible to read directly from the register value whether a
frame has arrived.

**No frame was ever observed.** The entire block `38000`–`38014` was zero in
all measurements across eleven logs.

An earlier noted outlier — `38003` showed the value `118` three times — turned
out to be a **scanner artifact**, not telemetry:

| Log | `37012` (`bms_version`) | `38003` |
|---|---|---|
| `entladen_lang` | 118 in 65/66, **0 in exactly one batch** | 0 in 65/66, **118 in exactly the same batch** |
| `entladen_dc` | 118 in 3/4, 0 in one | 0 in 3/4, 118 in the same one |
| `discharge` watch | 118 in 7/8, 0 in one | 0 in 7/8, 118 in the same one |

The 118 is the BMS version from register `37012`. Cause:
`scan_continuous.py` (local, not in the repo) and `scan_known_registers.py` (local, not in the repo) do set a
Modbus transaction ID, **but do not check it on receipt**. A delayed
response to request N is thereby read as the response to request N+1. Only the
newer `scan_registers.py` (local, not in the repo) validates the TID (`expect_tid`).

For the send side this means: there is **no evidence** that anyone generates
frames `0x40`–`0x43`. The Control has receive buffers for them, but they
stayed empty. The code group thus remains unconfirmed on both sides
— Micro v116 neither sends nor handles them.

### How to Read This

A gap does **not** automatically mean "dead function". Possible explanations:

- The receiver is a different firmware version or hardware variant. The
  codes exist in the Control because the same code base serves Venus A, D and E v3.
- The sender is a different node than the Control (evidenced for the
  BMS side).
- The function is laid out in the Control but not implemented in the target device.

All that is evidenced: **In this specific firmware pairing (Control v150 ↔
Micro v116 ↔ BMS v118) there is no receiver for these codes.**

---

## 5. Practical Consequence for the Home Assistant Integration

Modbus registers `46500`–`46544` write into this bus via `Inverter_Set_Schedule_Reg`. A
write costs one EEPROM cycle and generates a
CAN message that demonstrably nobody evaluates. **Do not include in the integration.** Details in
`../Modbus_RS485_TCP/VNSD_Write_Register_46500_Block_v150.md`.

---

## 6. Methodology

- 46 callers of `CAN_Build_Arbitration_ID` via a decompiler scan over the
  Control image, arguments extracted per call site.
- Function codes of the receivers by reading out the comparison branches
  completely (`uVar14 ==`, `case`, `bVar1 ==`), not by spot checks.
- Cross-check in the BMS: scan over all 649 functions for a comparison against
  `0x30`–`0x3E`. Five hits, all unrelated (I²C EEPROM, printf
  digit parser, EEPROM log slots, CLI escape, SoC calibration).
- Fixed IDs in the BMS resolved from the literal pool, not estimated.

### Open

- Target class 2 (RS485 path) has not been checked against a receiver.
- Purpose of `0x40`–`0x43` (`FUN_0802D2F4`) unknown.
- Bits 16–19 and 24–28 of the ID (sequence/counter) have not been
  traced functionally.
