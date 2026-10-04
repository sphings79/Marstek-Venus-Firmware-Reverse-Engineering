# Reading register 32103 on its own stops the Venus D answering

**Found and measured on:** Marstek Venus D (VNSD-0), Control / EMS **v150**
(`VNSD-0_app_0150_0805_115146.bin`, build Aug 5 2026)
**Interface:** Modbus TCP, port 502, function code 03
**Date:** 20 September 2026

This is a follow-up to the three defects we sent earlier today, and to the read serializer
report we sent some time before that. It arrives separately only because we found it after
those had already gone out.

It is the same component as defect 2 of that set — `Read_Serializer` — but a different fault
with a different cause, so it needs its own correction.

**Please pass this to whoever maintains the Modbus stack.** As with the others, a short
confirmation that it reached engineering is all we need. We would be glad to test a candidate
build on our unit.

---

## Symptom

A read of **register 32103, quantity 1** is not answered at all. Not with a value, not with an
exception — the device simply stops responding, and the session is dead afterwards: the next
request on the same connection also times out. After a few seconds the device accepts new
connections again and behaves normally.

A block read that covers the same register works perfectly.

Each line below was measured on a fresh connection, device idle:

| Request | Result | Time | Session afterwards |
|---|---|---|---|
| **32103, quantity 1** | **no response** | timeout | **dead** — a following read of 32104 also times out |
| 32103, quantity 1, repeated | no response | timeout | dead |
| 32102, quantity 2 (covers 32103) | `0 0` | 8.6 ms | fine |
| 32100, quantity 5 | `5269 0 0 0 23` | 9.2 ms | fine |
| 32102, quantity 1 | `0` | 4.4 ms | fine |
| 32104, quantity 1 | `23` | 4.5 ms | fine |

We scanned all 65535 register addresses to check whether anything else behaves this way.
**32103 was the only one.** Every other unimplemented address — 64984 of them — returned a
clean exception.

The device does not restart. Neighbouring registers hold their previous values afterwards.

---

## Where it is

`Read_Serializer` at `0x08055414` (v150, correct image base 0x08004800; originally cited as `0x08050C14`
from an older Ghidra analysis with the wrong base 0x08000000, off by 0x4800). It first works out the offset of the requested register
inside the descriptor entry, counted in registers:

```c
uVar3 = param_2 - (uint)*param_4 & 0xffff;        /* 32103 - 32102 = 1 */
```

Then the branch for type code `0x24`, which the serializer's own type table gives as
IEEE-754 float (`'$'` is `0x24`):

```c
if ((char)param_4[4] == '$') {
    local_3c = *(float *)(uVar3 * 2 + iVar4 * 4 + *(int *)(param_4 + 2));
```

With `uVar3 = 1` the float is loaded from **base + 2**. The entry for 32102
(`bat_sample_power`) has its data at `0x20014EB8`, so the load address is **`0x20014EBA`** —
halfword aligned, not word aligned.

That is the whole fault. A word-aligned `VLDR` is architecturally required on Cortex-M; an
unaligned one raises a UsageFault. Unlike the integer case, this is not affected by
`CCR.UNALIGN_TRP` — the floating-point load traps either way.

The integer branches a few lines below compute the same skewed address:

```c
else if (bVar2 == 4) {
    local_38 = *(uint *)(iVar4 * (param_4[4] & 0xf) + uVar3 * 2 + *(int *)(param_4 + 2));
}
```

but they compile to ordinary `LDR`, and an unaligned `LDR` is permitted. That is why every
other 32-bit register in the map reads correctly from its upper half and only this one does
not.

### Why only this register

Of all the 32-bit entries we have mapped, 32102 is the only one with type code `0x24`:

| Register | Field | Type code | Upper half readable on its own |
|---|---|---|---|
| **32102** | bat_sample_power | **`0x24` float32** | **no — no response** |
| 32202 | grid_sample_power | `0x12` i16 | yes |
| 33000 / 33002 / 33004 / 33006 / 33008 / 33010 | energy counters | `0x04` u32 | yes |
| 36000 / 36100 / 36102 | warning and error codes | `0x04` u32 | yes |
| 37013 / 37015 / 37021 | error codes, PV yearly yield | `0x04` u32 | yes |

If there are further float32 entries elsewhere in the descriptor table, they will have the
same problem.

---

## What we ruled out

Two plausible explanations that turned out to be wrong, so that nobody spends time on them
again:

**It is not the request size.** The copy loop at the end of `FC03_Read_Handler` counts with an
8-bit masked counter against a 16-bit bound:

```c
for (uVar5 = 9; uVar5 <= uVar8; uVar5 = uVar5 + 1 & 0xff) { ... }
```

This would loop forever once `uVar8` exceeds 255, which `uVar8 = 9 + 2 × registers served`
would reach at 124 registers. We tested it on the device — start 34000, quantities 34, 35, 120,
123, 124, 125 and 126 — and all of them answered correctly, 126 with exception 3 from the
quantity check. The descriptor walk stops at the first gap, so `uVar8` never gets large enough
in practice.

We mention it because the loop is still wrong as written, and a future map with a longer
contiguous run would hit it. It is a one-character fix, but it is not this defect.

**It is not reading into the middle of an entry as such.** Every other `_w2` register is also
the upper half of a four-byte entry and reads fine on its own.

---

## Impact

A client that reads registers individually and touches 32103 loses its connection. Because the
device accepts only one Modbus TCP session at a time, that also takes down whatever else was
waiting for it until the device recovers.

It affects register discovery in particular: any tool that walks the address space one register
at a time will hit this and conclude the device has gone offline.

---

## Suggested fix

Load the float through an aligned access rather than a direct `*(float *)` on a computed
address — for example copy four bytes into a local `float` (or a `uint32_t` union) and use
that, which is effectively what the integer branches already do.

Rejecting a read that starts at an odd offset inside a float entry with exception 2 would also
be correct behaviour and is a smaller change, though returning the upper half as the other
types do is friendlier to clients.

Either way the device should never answer a well-formed request with silence.

---

## Status in v151 (checked 2026-10-01)

**Changed — the trigger is gone, the underlying defect is not.**

`Read_Serializer` itself is instruction-identical in v151 (`0x08055fe0`, 626 bytes) — the float branch still
loads via `VLDR` from a possibly unaligned address (see our read-serializer report, "Status in v151"
section). That root cause was not fixed.

What changed is the descriptor entry for 32102: it moved from `float32` (type code `0x24`, 4 bytes) to `i16`
(type code `0x12`, 2 bytes). The entry now ends at register 32103 instead of 32104, so 32103 falls outside
every descriptor entry. `FC03_Read_Handler` (v150 `0x0802386c` → v151 `0x08023a00`, 382 bytes) then takes the
"address not covered by any entry" branch instead of ever reaching the float code:

```c
if (start_offset <
    (uint)g_modbusRegDescTable[reply_write_offset].reg_addr +
    (uint)((int)(short)(g_modbusRegDescTable[reply_write_offset].type_code2 & 0xf) *
          (int)(short)(ushort)g_modbusRegDescTable[reply_write_offset].count) / 2) {
  if (start_offset < g_modbusRegDescTable[reply_write_offset].reg_addr) {
    return 2;                    /* clean Exception 02 */
  }
  ...
```

So a solo read of 32103 in v151 no longer hits the unaligned `VLDR` at all — it gets a normal Exception 02
instead of silence and a dead session. The alignment bug in `Read_Serializer` itself is still there and would
reappear for any future float32 entry with an odd register offset; we have not checked whether one exists in
v151's descriptor table.

## Reproducing

1. Connect over Modbus TCP on port 502. The device accepts only one session at a time.
2. Read register **32103, function code 03, quantity 1**. No response arrives.
3. On the same connection, read 32104, quantity 1. It also times out — the session is gone.
4. Reconnect, then read **32102, quantity 2**. It returns two registers immediately.
5. Reconnect and read 32102 alone, then 32104 alone. Both work.

We can provide the socket captures, the full 65535-address scan behind the "only one" claim,
and the timings on request.

Regards,
Reporter
