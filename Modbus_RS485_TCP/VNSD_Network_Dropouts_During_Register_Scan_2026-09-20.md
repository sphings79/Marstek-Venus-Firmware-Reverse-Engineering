# Network dropouts during the register scan

> 🇩🇪 Deutsche Version: [VNSD_Netzabrisse_waehrend_Registerscan_2026-09-20_de.md](VNSD_Netzabrisse_waehrend_Registerscan_2026-09-20_de.md)

**Device:** Marstek Venus D (VNSD-0), `192.168.1.50:502`, direct, without proxy
**Firmware:** Control/EMS v150
**Date:** 20 September 2026
**Status:** observed and narrowed down · **cause open** · counter-test at low load pending

---

## 1. What happened

During a full scan of all holding registers (function code 03, one address per request,
about 6.5 requests per second), the device lost the connection three times. Twice following
the same pattern:

```
43668  exception  illegal_data_address  150.0 ms   <- last response from the device
43669  error      BrokenPipeError         0.1 ms   <- send() fails
43670  —          No route to host                 <- device no longer reachable
```

`BrokenPipeError` on **sending** means: the peer had already closed the TCP connection
before the next request went out. So the device actively hung up, it is
neither a timeout nor packet loss.

Afterwards it was not only silent on port 502, but gone at the IP level: `EHOSTUNREACH`. A
device that had merely lost its Modbus server would instead return `ECONNREFUSED`
or keep answering ICMP.

## 2. The three disruptions

| No. | Register | Type | Last response before | Outage duration |
|---|---|---|---|---|
| 1 | **32103** | Timeout, 10 s, no response | 32102, exception, 150 ms | short, afterwards session occupied |
| 2 | **40897** | `BrokenPipeError`, then `EHOSTUNREACH` | 40896, exception, 150 ms | ~2 h 30 min (13:52 – 16:21) |
| 3 | **43669** | `BrokenPipeError`, then `EHOSTUNREACH` | 43668, exception, 150 ms | ~1 h (16:30 – 17:30) |

Disruption 1 is probably of a different nature: before it, a scan process had been killed hard,
and the device presumably still held the old session. The device allows **exactly one** Modbus TCP
session. It is listed here only for completeness.

Trigger in the sense of "the request that was still answered":

```
40896 = 0x9FC0        43668 = 0xAA94        distance 2772
```

Both still returned a regular exception 2 *illegal data address*.

### Requests between the disruptions

| Section | Requests | of which exceptions | Values |
|---|---|---|---|
| 1 – 32102 | 32102 | 32017 | 85 |
| 32103 – 40896 | 7779 | 7469 | 309 |
| 40897 – 43668 | 2772 | 2684 | 87 |

The sections get shorter. For a fixed threshold — such as "after N exceptions a
buffer is full" — that is not enough: 32017, 7469 and 2684 are too far apart.

### Response times

Every rejected address is answered in **exactly 150.0 ms**, across tens of thousands of requests,
until immediately before the dropout. No degradation, no increase, no jitter. The
device shows no symptoms of load beforehand.

---

## 3. What the firmware says about it — nothing

Checked in the D image v150:

**The two trigger registers are not special.** The read path delegates everything from 40000 onward
to `Write_Handler(buf, reg, 0)` (`FC03_Read_Handler` @ 0x0801F06C, line 62 ff.). Its
range boundaries for `param_2` are:

```
40000 41000 41100 41200 41201 41500 41501 41502 41503 41511 41512
41600 41604 41610 41611 41631 42011 42020 43100 43101 43102 43103
43118 45000 45603 45604 46499 46500 46545 47400
```

**40896 lies between 40000 and 41000, 43668 between 43118 and 45000** — both in the middle of
an unassigned gap, at no boundary, in no special branch. In the same gaps,
thousands of neighboring addresses returned the same exception without any incident.

**The receive path calculates correctly.** `Modbus_Dispatcher` reads 7 bytes of MBAP, takes the
length field, reads `len − 1` further bytes and subtracts `len + 6` from the fill level — that is
consistent, nothing runs out of control here.

**The only known special case in the read path is a different range.** Line 30 of the
`FC03_Read_Handler`:

```c
if (((37999 < uVar7) && (uVar7 < 0x9867)) && (*DAT_0801f1ec == '\0')) {
    *DAT_0801f1ec = '\x01';
}
```

38000 – 39014 sets a flag. This range is **skipped** by the scanner and is not a candidate
explanation.

There is thus **no firmware finding that singles out the two addresses.** If a
connection exists, it is cumulative via the number of exception frames sent, not via
the specific register number. These frames are those whose MBAP length field is one too large
(`VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20.md`) — an obvious suspicion, but the
section lengths above do not support it.

---

## 4. What speaks against a pure environment problem, and what for it

**For:** the measuring Mac has **two interfaces in the same subnet**, `en0 = 192.168.1.71`
and `en1 = 192.168.1.70`, both with an ARP entry for the device. Dual attachment to the same
L2 segment produces asymmetric routing and is a classic source of sporadic
`EHOSTUNREACH`. That needs to be cleaned up before blaming the device.

**Against:** in the healthy state none of this is visible. Counter-check on 20.09. at 17:27,
while the scanner was still reporting `No route to host`:

- ICMP: 10 of 10 packets, 0.3 ms, no loss
- TCP connect to port 502, four attempts each **from each of the two source addresses**: 12 of 12
  successful

And the dropout itself begins every time with a `BrokenPipeError` — that is, with the
**device** closing the existing connection. That does not explain a routing problem of the measuring computer.

---

## 5. Open, and how it could be clarified

The observation is reliable, the cause is not. Three hypotheses, none confirmed:

1. **Cumulative exhaustion in the network stack** under continuous load — CH395 socket, buffers,
   descriptors. Fits the active closing and the long outage duration, does not fit the
   strongly fluctuating section lengths.
2. **Register-dependent trigger.** Severely weakened by section 3: in the firmware there is nothing at
   the two addresses.
3. **Environment** (section 4). Weakened by the counter-check, but not ruled out.

### Planned counter-test

After the full scan is complete: **40891 – 40901 and 43663 – 43673**, i.e. ±5 registers around both
triggers, at **low load** — one request every few seconds instead of 6.5 per second, over
several runs.

- If the dropout occurs there too, it is register-dependent and hypothesis 2 is back in play.
- If it does not occur, that supports hypothesis 1, and the next step would be a run test
  at a high rate on an **innocuous** address, to separate the load from the register.

Sensible beforehand: switch off the second IP in the subnet on the measuring computer.

### Restart or not — open, and the indicators are no good

The shadow value behind 44002/44003 was set under control to 2500 at 12:00 and reads
0 again at 17:38:

| Time | 44002 | 44003 | Source |
|---|---|---|---|
| 11:46 | 0 | 0 | after the restart triggered by the user |
| ~12:00 | 2500 | 2500 | end of the A/B test, readback value under control |
| 17:38 | 0 | 0 | running full scan |

From this it was initially concluded here that the device had restarted. **The conclusion does not
hold.** The shadow at `0x20000156` is not only zeroed at startup, but is also filled by
`Protocol_AA_SetExtendedParams` from a CAN frame, and the call chain leading there is

```
CAN_FrameDispatcher → Telemetry_Register_Dispatcher (command 0x14) → Protocol_AA_SetExtendedParams
```

A CAN frame with zeros during a disruption — when the inverter or the Control restarts — is not
unlikely, but to be expected. Restart and CAN zeroing are **indistinguishable** at this
register.

Two further candidates were checked and are likewise ruled out:

- **Home Assistant** has no data. The integration has been idle since 13:30, i.e. since the
  proxy was switched off for the scan — and thus before the first dropout at 13:52.
- **Register 42000** now reads `0x55BB` instead of the `0x55AA` written at 12:00. The
  underlying state, however, is the work mode byte at `0x20014D01`, and that has many
  writers: `MQTT_Config_Command_Handler`, `MQTT_JSON_RPC_Dispatcher`, `MQTT_SetWorkMode`,
  `MQTT_PassiveModeHandler`, `BLE_Cmd_Dispatch`, the config loader and the `Write_Handler`.
  Evidential value zero.

  *Side finding:* this branch (`uVar8 == 0x186` at base `0xA28A`) belongs to **42000**, not
  to 43492, as the register map in the function header of the `Write_Handler` claims. 43492 returns
  an exception in the scan.

**So it remains open whether the device restarted.**

### A canary that would decide it

What is needed is a register in BSS that is set exclusively by a Modbus write access. The
cross-references to the SRAM block `0x20000348` deliver exactly that:

| Register | Address | writing | reading |
|---|---|---|---|
| 42010 force_mode | 0x20000348 | `Write_Handler`, `Modbus_Dispatcher`, `RS485_RTU_Frame_Dispatcher` | the same |
| 42011 charge_target_soc | 0x2000034A | **only `Write_Handler`** | `Remote_Power_Command_Execute` |
| 42020 charge_power_limit | 0x2000034C | **only `Write_Handler`** | `Remote_Power_Command_Execute` |
| **42021 discharge_power_limit** | **0x2000034E** | **only `Write_Handler`** | `Remote_Power_Command_Execute` |

No CAN, no MQTT, no BLE, no restore from the EEPROM. If a value written into it falls
to 0, it can **only** have been the BSS initialization at startup.

`42021` is the safest of these: `Remote_Power_Command_Execute` evaluates it only when
`force_mode` is active. If 42010 is at 0 (standby), the write access is without consequence, and
unlike 44002 it touches neither the EEPROM nor the inverter.

That is why `Scripts/dropout_retest.py` (local, not in the repo) uses **42021** as the indicator, not 44002.

---

## 6. Impact on the scanner

`Scripts/register_scan.py` (local, not in the repo) was subsequently improved twice:

1. The reconnect in the timeout branch was unprotected — a refused connection ended the
   entire run. Now with backoff and caught.
2. A lost session makes it retry the same address instead of rattling through the rest of the scan as
   errors. The wait time grows from 30 s to 5 min, up to 60 attempts, i.e.
   roughly four hours of patience. The CSV remains valid at all times.
