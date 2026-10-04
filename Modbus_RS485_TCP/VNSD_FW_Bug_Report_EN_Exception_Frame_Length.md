# Bug report — Modbus TCP exception responses declare one byte more than they contain

**Product:** Marstek Venus D (VNSD-0)
**Firmware:** Control / EMS **v150** (`VNSD-0_app_0150_0805_115146.bin`)
**Interface:** Modbus TCP, port 502
**Affected:** every exception response the device produces (illegal function, illegal data
address, illegal data value)
**Severity:** the device cannot signal a rejected request; clients see a timeout instead

---

## Summary

When the device rejects a request it builds a Modbus TCP exception response and sends it.
The frame is nine bytes and correct in every field but one: the **MBAP length field is set
to 4 where it must be 3**.

A conforming client reads the seven-byte MBAP header, then waits for `length - 1 = 3` bytes
of PDU. Only two arrive — the function code and the exception code. The client blocks on a
byte the device never sends and eventually times out, so a deliberate, well-formed rejection
is indistinguishable from a device that stopped answering.

The same firmware computes the length correctly on the success path, so this is an off-by-one
against its own convention rather than a different framing scheme.

## The frame

```
offset  0..1   transaction id
        2..3   protocol id
        4..5   length          <-- written as 4, must be 3
        6      unit id
        7      function code | 0x80
        8      exception code
```

The MBAP length field counts the bytes that follow it: unit id + function code + exception
code = 3. The device sends nine bytes in total but declares `6 + 4 = 10`.

## Where it is

`Modbus_Dispatcher` at `0x08023204` (correct image base 0x08004800), exception branch from `0x080232e8`.
(Originally cited as `0x0801ea04` / `0x0801eae8` — an older Ghidra analysis used the wrong base 0x08000000,
off by a constant 0x4800; the same offset applies below.)

```
080232f2  strb r0, [r1, #4]     ; length, high byte = 0
080232f4  movs r0, #4           ; <-- the defect
080232f6  strb r0, [r1, #5]     ; length, low byte = 4
080232fc  adds r0, #0x80        ; function code | 0x80  (correct)
080232fe  strb r0, [r1, #7]
08023304  strb r0, [r1, #8]     ; exception code
08023306  movs r2, #9           ; nine bytes are sent
0802330a  bl   CH395_SPI_Send_Data
```

For comparison, the success path in `FC03_Read_Handler` at `0x0802386c` (correct base; originally `0x0801f06c`) computes
`length = quantity * 2 + 3`, which accounts for unit id, function code and byte count. A
one-register read therefore declares length 5 and sends eleven bytes, which is correct and
which we confirmed on a live device:

```
request   12 34 00 00 00 06 01 03 75 F8 00 01
response  12 34 00 00 00 05 01 03 02 00 96      (11 bytes, length 5)  OK
```

## Which requests reach the branch

| Handler result | Exception code | Condition |
|---|---|---|
| 2 | Illegal data address | requested register falls in a gap between descriptor entries |
| 3 | Illegal data value | register past the last descriptor entry, or quantity 0 or > 125 |
| 1 | Illegal function | any function code other than 03, 06, 10 |

## Impact

- **A read of an unimplemented register costs a full client timeout.** With a three second
  timeout, every wrong guess costs three seconds instead of a few milliseconds.
- **Block reads spanning unimplemented registers cannot be used.** A block that covers a
  single unimplemented address is rejected as a whole, and the rejection is unreadable. On
  our unit this removes an otherwise available optimisation: merging read blocks across gaps
  of up to ten registers would cut a full poll cycle from 76 requests to 34, but each wrong
  merge would cost a timeout rather than an immediate error.
- **Discovering a register map is slow.** Probing unknown addresses runs at timeout speed
  rather than at response speed.
- A client that leaves the nine bytes in its receive buffer and reuses the connection may
  also lose the *next* response, because its parser takes the missing tenth byte from the
  following frame and is misaligned afterwards.

## Suggested fix

```c
buf[5] = 3;   /* unit id + function code + exception code */
```

A single constant. The rest of the frame, including the `| 0x80` on the function code, is
already correct.

## How to reproduce

1. Connect to the device over Modbus TCP on port 502.
2. Read a register that the firmware does not implement, for example **39000** or **45000**
   (FC03, quantity 1).
3. A compliant client reports a timeout rather than an exception.
4. Capture the raw bytes on the socket: nine bytes ending in `01 83 02`, with the length
   field reading `00 04`. Note that a proxy or gateway in the path will hide this — see the
   note below.

## Captured frames

Read straight from the device on port 502, one register, function code 03:

| Requested | Response on the wire | Exception |
|---|---|---|
| 30200 (implemented) | `15 F8 00 00 00 05 01 03 02 00 96` — 11 bytes, length 5 | — |
| 39000 | `18 58 00 00 00 04 01 83 03` — **9 bytes, length 4** | 3 |
| 45000 | `1F C8 00 00 00 04 01 83 02` — **9 bytes, length 4** | 2 |
| 31500 | `1B 0C 00 00 00 04 01 83 02` — **9 bytes, length 4** | 2 |
| 35000, quantity 12 | `18 B8 00 00 00 04 01 83 02` — **9 bytes, length 4** | 2 |

The device answers immediately and the frame is well formed apart from the length field.

## Measured client cost

With a standard client library on a persistent connection, one read of an unimplemented
register costs **two** failed reads, not one:

```
1) implemented register   -> 150    (0.01 s)
2) unimplemented, 1 try   -> None   (3.05 s)   client library logs:
   "Received unexpected response with Transaction ID: 2.
    Discarding bytes: 00 02 00 00 00 04 01 83 03 00"
3) implemented, same conn -> None   (3.05 s)   lost
4) implemented, again     -> 150    (0.05 s)   recovered
```

The ten discarded bytes are the nine bytes of the exception plus the first byte of the
*following* response: the parser waits for the tenth byte the header promised, takes it from
the next frame, and discards both before resynchronising.

## Note for anyone reproducing this behind a proxy

The defect is invisible from behind a Modbus proxy or gateway. A proxy is itself a parser, so
it stalls on the same missing byte and simply closes the connection after its own timeout —
we measured no bytes at all through ours, and only saw the real frames once it was stopped.
The device also accepts only one Modbus TCP session at a time: port 502 refuses further
connections while another client holds it.

We are happy to share the raw captures and the register dumps behind these measurements on
request.

## Other models

We verified this on a Venus D running Control/EMS v150. We have not verified Venus A or
Venus E v3, which share this firmware base, but we would expect the same behaviour there and
suggest the fix be applied across the shared codebase rather than to the Venus D build alone.

## Related

We reported a second, independent defect in the same read path: FC03 applies its scale
divisions with unsigned arithmetic, which corrupts signed registers with a divide scale
(register 32101, BMS battery current). See `VNSD_FW_Bug_Report_EN_Read_Serializer.md`.

## Status in v151 (checked 2026-10-01)

**Unchanged.** `Modbus_Dispatcher` is at `0x08023398` in v151 (392 bytes, same size as v150). The exception
branch is byte-identical to the v150 defect:

```
08023488  movs r0,#0x4      ; MBAP length low byte still 4, not 3
0802349a  movs r2,#0x9      ; still nine bytes sent
```

The decompiled source shows the same `g_modbusTcpTxFrame_lengthLo = '\x04';` followed by `CH395SendData()`
at that spot. The MBAP length off-by-one on exception frames is present unchanged in v151, with the same
consequence for clients (timeout instead of an immediate, readable rejection).

## MCU correction

The Control/EMS MCU is a **Nations N32G45x with XFMC** (Cortex-M4F), not an Artery AT32F403A as earlier
documentation assumed. This has no bearing on the defect itself, which is pure application-level logic.
