# FW Bug: Exception Frame Declares One Length Too Many (MBAP Off-by-One)

> 🇩🇪 Deutsche Version: [VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md](VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md)

**Device:** Marstek Venus D (VNSD-0), Nations N32G45x (Cortex-M4F) — not AT32F403A, see section 10
**Firmware:** Control/EMS **v150** (`VNSD-0_app_0150_0805_115146.bin`)
**Path:** Modbus **TCP** (CH395), not RS485 RTU
**Affected function:** `Modbus_Dispatcher` @ `0x08023204` (correct base 0x08004800), exception branch from
`0x080232e8` — originally noted as `0x0801ea04` / `0x0801eae8` (old Ghidra analysis with wrong base
0x08000000, difference consistently 0x4800)
**Date:** 20 September 2026
**Status:** Static analysis proven byte-exact · **confirmed on the device** (section 5)

---

## 1. Summary

For rejected requests, the firmware builds a Modbus TCP exception frame and sends it.
The frame is 9 bytes long and correct except for one field: the **MBAP length field contains 4,
the correct value would be 3**. As a result, a compliant client waits for a byte that never arrives and runs
into its timeout. For the client, the exception is indistinguishable from "device does not respond".

Practical consequence: reading an unimplemented register costs a full
client timeout instead of an immediate rejection.

---

## 2. The Code

Exception branch in `Modbus_Dispatcher`, reached when an FC handler returns an error code ≠ 0:

```c
else {
  DAT_0801eb98[4] = 0;              /* MBAP length high byte */
  puVar1[5] = 4;                    /* MBAP length low byte  <-- BUG, must be 3 */
  puVar1[7] = puVar1[7] + -0x80;    /* FC | 0x80, correct in 8 bit: 0x03 -> 0x83 */
  puVar1[8] = *DAT_0801eb9c;        /* exception code */
  CH395_SPI_Send_Data(0,puVar1,9);  /* 9 bytes */
}
```

Assembler, read byte-exact from the image (not a decompiler artifact), correct base 0x08004800
(originally noted with wrong base 0x08000000 as `0x0801eaee` ff., here consistently +0x4800):

```
080232ee  00 20        movs r0, #0
080232f0  29 49        ldr  r1, [pc, #0xa4]
080232f2  08 71        strb r0, [r1, #4]     ; length high byte = 0
080232f4  04 20        movs r0, #4           ; <-- the 4
080232f6  48 71        strb r0, [r1, #5]     ; length low byte = 4
080232f8  08 46        mov  r0, r1
080232fa  c0 79        ldrb r0, [r0, #7]
080232fc  80 30        adds r0, #0x80        ; FC | 0x80
080232fe  c8 71        strb r0, [r1, #7]
08023300  26 48        ldr  r0, [pc, #0x98]
08023302  00 78        ldrb r0, [r0, #0]
08023304  08 72        strb r0, [r1, #8]     ; exception code
08023306  09 22        movs r2, #9           ; <-- 9 bytes sent
0802330a  e4 f7 51 fd  bl   CH395_SPI_Send_Data
```

### Why 3 and Not 4

The frame:

| Offset | Content |
|---|---|
| 0–1 | Transaction ID |
| 2–3 | Protocol ID |
| 4–5 | **Length** |
| 6 | Unit ID |
| 7 | FC \| 0x80 |
| 8 | Exception code |

The length field counts the bytes **after** itself: unit + FC + code = **3**. 9 bytes are sent,
6 + 4 = **10** are declared. The frame promises one byte more than it delivers.

### Cross-Check Against the Firmware's Own Convention

`FC03_Read_Handler` @ `0x0802386c` (correct base; old `0x0801f06c`) calculates in the success case:

```c
iVar1  = uVar5 * 2;      /* byte count = count * 2 */
iVar10 = iVar1 + 3;      /* length = byte count + 3  (unit + FC + byte-count byte) */
```

For one register: byte count 2, length 5, frame 11 bytes — correct and confirmed live
(`12 34 00 00 00 05 01 03 02 00 96`). The same firmware writes a 4 in the error case, where
by its own calculation a 3 should be. So this is an off-by-one, not a different
framing scheme.

---

## 3. When the Branch Is Reached

`FC03_Read_Handler` has two error exits, both end up here:

| Return value | Exception | Trigger |
|---|---|---|
| 2 | Illegal Data Address | Register lies in a **gap between two descriptor entries** |
| 3 | Illegal Data Value | Register **beyond the last entry**, or count 0 or > 125 |

Additionally, the dispatcher sets code 1 (Illegal Function) for every FC except 03/06/10.

The descriptor table has 246 entries (stride 0xC) starting at SRAM `0x20000354`. The search loop
aborts at the first entry whose end lies above the searched address; if the address lies
before it, it is in a gap → code 2.

---

## 4. Impact

- **Every unimplemented register costs a full client timeout** instead of an
  immediate rejection. With a 3 s timeout that is 3 s per miss.
- **Gap bridging becomes unaffordable — but only because of this bug.** On the device, a block
  that spans an unoccupied register is rejected immediately with exception 2; that costs the same
  ~126 ms as any other request. If the client cannot read the exception, this turns into
  3 s. That flips the calculation: with a readable exception, `max_gap=4` saves 19 % of the cycle time,
  without it the same setting would be about 9 s slower than gapless reading. Measured values in
  section 5d.
- **Register map exploration is slow.** A scan over unknown addresses runs at
  timeout speed instead of response speed. For the still-open Venus A map
  this is the difference between minutes and hours.
- **An unoccupied register costs two reads, not one.** The 9 bytes remain in the
  receive buffer; the parser pulls the missing tenth byte from the next response and discards
  both frames. Measured: 6.1 s and two lost reads (section 5b).

---

## 5. Measurements

### 5a. Directly on the Device — the Proof

`192.168.1.50:502`, proxy stopped, raw capture on the socket:

| Request | Response | Code |
|---|---|---|
| 30200 (occupied) | `15 F8 00 00 00 05 01 03 02 00 96` — 11 B, length 5 ✓ | — |
| 39000 | `18 58 00 00 00 04 01 83 03` — **9 B, length 4** | 3 |
| 45000 | `1F C8 00 00 00 04 01 83 02` — **9 B, length 4** | 2 |
| 31500 | `1B 0C 00 00 00 04 01 83 02` — **9 B, length 4** | 2 |
| 35000+12 | `18 B8 00 00 00 04 01 83 02` — **9 B, length 4** | 2 |

The device responds **immediately** and with a well-formed exception frame — except for the
length field. The codes match exactly the two return paths from section 3:
39000 lies beyond the last descriptor entry → code 3, the others in a gap or
≥ 40000 → code 2. This confirms the static analysis from section 2 end-to-end.

### 5b. Cost at the Client — Measured

Real client (tmodbus 0.6.2, `MarstekModbusClient`) on **one** connection, without reconnect:

```
1) good register         -> 150    (0.01s)
2) unoccupied, 1 attempt -> None   (3.05s)
   tmodbus: Received unexpected response with Transaction ID: 2.
            Discarding bytes: 00 02 00 00 00 04 01 83 03 00
3) good, same conn       -> None   (3.05s)   <-- lost
4) good, again           -> 150    (0.05s)
```

The discarded ten bytes are the nine exception bytes **plus the first byte of the
following response**. The parser waits for the declared tenth byte, gets it from the next
frame, discards both and resynchronizes. **An unoccupied register thus costs two
failed reads and 6.1 s.** The reconnect after a timeout is what prevents the second
loss — it is not an optimization reserve, but the resynchronization.

### 5c. Via the Proxy — Why It Looks Like Silence

The same registers via `192.168.1.60:1502`, each attempt on a fresh socket:

| Request | Result |
|---|---|
| 30200+1, 30200+4, 34002+7 (occupied) | OK, 5–154 ms |
| 30200+6 (spans 30204) | OK, 4 ms |
| 30205+10, 34002+32, 35000+12, 42000+12 (gap) | **Silence**, 4 s timeout |
| 39000, 45000, 60000, 31500 (unoccupied) | **Silence**, 6 s without a single byte |
| 31500, 35000+12 (raw capture) | Remote end **closes** after ~3.1 s |
| 49999 | Connection reset |

**Not a single byte arrives via the proxy.** Not nine, but zero. The proxy is itself
a Modbus parser: it waits for the declared tenth byte, runs into its own timeout
and closes. From behind the proxy, the defect is therefore not visible as a protocol error,
but only as a silent device.

Side finding: `192.168.1.50:502` actively rejects connections (RST, not timeout) as long as
the proxy is running, while the proxy itself accepts three clients simultaneously. The Venus apparently allows exactly
**one** Modbus TCP session — the reason the proxy exists, and the
reason this finding remained undiscovered for so long.

---

### 5d. Gap Bridging Benchmark on the Device

`Scripts/gap_benchmark.py` (local, not in the repo), proxy stopped, complete `d.yaml` (455 definitions,
482 addresses) once per strategy, 20 ms pace, FC03 only. The client reads the defective
exception frames correctly by reading a fixed two PDU bytes when bit 7 is set and ignoring the
length field.

| max_gap | Blocks | Requests | Wall | vs. baseline |
|---|---|---|---|---|
| 0 (current implementation) | 60 | 60 | 8.92 s | — |
| 1 | 51 | 52 | 7.91 s | −1.01 s |
| 2 | 48 | 49 | 7.35 s | −1.57 s |
| **4** | **41** | **48** | **7.25 s** | **−1.67 s (−19 %)** |
| 8 | 39 | 50 | 7.56 s | −1.36 s |
| 10 | 35 | 51 | 7.71 s | −1.21 s |

**310 requests, zero timeouts, zero resets.** Every rejection came as an immediate exception 2.

Two observations:

1. **Latency does not depend on block size.** Median 126 ms, regardless of whether 1 or 40 registers,
   maximum 180 ms. The cost is pure per-request overhead; additionally read registers
   are practically free. That is why every saved request translates linearly.
2. **The optimum is at `max_gap=4`, not higher.** From 8 on, the rejections and their
   fallback requests eat up the gain again.

Gaps that cannot be bridged (rejected by the device): 30001–30010, 30200–30214, 35000–35011,
35100–35110, 42000–42021. With a memory for bad gaps (`bad_gaps`, as in the old
`dev_tmodbus_blockread`) each one is paid only once; in steady state both
`max_gap=4` and `=10` end up at **45 instead of 60 requests** (−25 %).

Measurement artifact: the test set contains 41000/41001 (`reset_device`, `factory_reset`) — pure
command registers, not readable, hence an exception in every strategy. The integration does not poll
them; in the live log they do not occur in any block.

**Condition:** the gain depends entirely on the client being able to read the exception.
With a 3 s timeout per rejection instead of 126 ms, `max_gap=4` would be about 9 s **slower** than
gapless reading. Gap bridging and the frame correction are a package.

---

## 6. Proposed Fix (Firmware)

```c
puVar1[5] = 3;   /* unit + FC + exception code */
```

One constant. The rest of the frame is correct, including the FC combination
(`0x03 + 0x80` gives `0x83` in 8 bit).

---

## 7. Fix on Our Side

The defect can be worked around without waiting for a firmware update. An exception PDU is by
Modbus definition **always exactly 2 bytes** (FC + code) — the declared length field is simply
not needed for it. Anyone who ignores the length when bit 7 is set in the function code and
reads a fixed 2 bytes is always right, even after a future firmware fix.

| Location | Effort | Effect |
|---|---|---|
| **In the proxy** | low | Corrects the frame once for *all* clients (HA, Omnibattery, scans). Recommended place. |
| **In tmodbus** | medium, fragile | The framing is in `ModbusTcpProtocol.data_received` (`tmodbus/transport/async_tcp.py`), a library internal. A monkeypatch does not survive a minor release; the clean way would be an upstream proposal, phrased as a general robustness rule instead of a Marstek special case. |
| **Shorter probe timeout** | low | Works around the problem instead of solving it. Healthy responses arrive in 5–154 ms; a 500 ms timeout for block probes makes a miss cheap. Price: a real stall looks like a dead register — only treat it as dead after several failed attempts in a row. |

Gain with option 1 or 2 solved: instead of 6.1 s and two lost reads, an unoccupied register costs
about 10 ms and yields an **unambiguous** result. This makes two things
practical that are ruled out today: gap bridging (calculated 55 % fewer requests) and the
fast exploration of an unknown register map — relevant for the still-open Venus A map.

## 8. Correction to Our Own Knowledge

Two earlier assumptions are thereby refuted:

1. **"The device responds to non-existent registers with silence."** Wrong. The device
   responds immediately and correctly with exception 2 or 3. The silence only arises because
   the length field is off by one and every compliant parser — proxy as well as client —
   waits for a byte that never arrives.
2. **"The reconnect after a timeout may be superfluous on tmodbus."** The opposite
   was measured: it is the only resynchronization that prevents the second lost read.
   Do not remove it.

---

## 9. Cross-References

- `VNSD_FW_Bug_Read_Serializer_Sign_2026-08-22.md` — the other serializer defect (32101)
- `VNSD_FW_Bug_Report_EN_Exception_Frame_Length.md` — vendor-side report on this finding
- `VNSD_Descriptor_Open_Registers_Ghidra_Finding.md` — descriptor table and register assignment
- Ghidra: durable pre-comment at `0x080232f4` (correct base; old note `0x0801eaf4`) in
  `/VNSD/Control/VNSD-0_app_0150_0805_115146.bin`

## 10. MCU Correction

The MCU is a **Nations N32G45x with XFMC** (Cortex-M4F), not an Artery/AT32F403A — the earlier assignment was
wrong (see `Control_FW` main documentation, section "MCU identification"). For the
bug described here this is irrelevant, as it concerns pure application code behavior, not an MCU-specific property.

## 11. Status in v151 (Firmware 151, checked 2026-10-01)

**Unchanged.** `Modbus_Dispatcher` is located at `0x08023398` in v151 (392 bytes, same size as v150). The
exception branch is byte-identical to the v150 bug:

```
08023488  0420             movs r0,#0x4      ; MBAP length low byte still 4 instead of 3
0802349a  0922             movs r2,#0x9      ; still 9 bytes sent
```

At the same location, the decompiler shows unchanged `g_modbusTcpTxFrame_lengthLo = '\x04';` followed by
`CH395SendData()`. The off-by-one in the MBAP length field for exception frames persists unchanged in v151 —
with the same consequences for clients (timeout instead of immediate error evaluation).
