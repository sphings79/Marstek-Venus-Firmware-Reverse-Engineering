# Case study: the 30-minute reset — from observation to root cause

> 🇩🇪 Deutsche Version: [VNSD_Cloud_Watchdog_Fallstudie_2026-08-26_de.md](VNSD_Cloud_Watchdog_Fallstudie_2026-08-26_de.md)

*Created 2026-08-26. Device: Venus D, Control firmware v150, LAN.
Related analysis: `VNSD_Control_FW_Analysis_app_0150_0805_115146.md` §4a.*

This file records the **path**, not just the result. Four explanations
persisted for an entire day, each with a real mechanism in the code
and a number that matched the measurement. All four were wrong. A single control experiment
disposed of them in ten minutes. The methodological error
is more transferable than the finding.

---

## 1. Starting point

The mechanism was known (`FUN_08015bd0`, §4a):

```c
if (((1 < *(ushort *)(DAT_08015fcc + 2)) &&
     (Tick_Timer_Check_Elapsed(DAT_0801600c, 0x708) != 0)) &&
    (*_DAT_08016010 == 0)) {
    CH395_Reset_And_Reinit(0);
    log_printf(3, 1, "[HTTP]ch395 reset!!!!");
    *DAT_0801600c = 0;
}
```

More than one buffered telemetry record, 1800 s elapsed, status register
37001 equal to zero → hardware reset of the Ethernet bridge. 2–3 s without network, without
Modbus, without ICMP.

The conclusion was obvious: answer the upload, the buffer empties, the condition is never
met. A container was built that does exactly that, and the uploads demonstrably
arrived.

**The resets continued.** Twelve of them in five and a half hours, mean
1829 s, scatter 43 s.

---

## 2. Four refuted hypotheses

### 2.1 The time response was faulty

It really was — truncated and in local time instead of the original's format.
The correction was necessary and right.

**It changed nothing.** The next reset was 34 s off a prediction from the
previous interval, clearly within the scatter of 43 s. The cycle even ran
through a complete container restart, during which nothing at all
was answering.

*Lesson:* Fixing a real defect does not prove it was **the** defect.

### 2.2 A cold start clears the counter

The counter lives at `0x2000100C` in regular SRAM. No code path restores it
from non-volatile memory — only three functions touch the
ring buffer header at all.

That part was right, and the measurement proved it: after the cold start the first
interval was **2133 s** versus 1824 s in steady state. The additional
309 s is the counter climbing back above its threshold —
`Tick_Timer_Check_Elapsed` stays true once elapsed, so the reset fires
as soon as the *last* open condition becomes true.

**But it climbed back within 35 minutes, and the cycle returned.** A
restart is not a solution.

### 2.3 Node's keep-alive blocks the device

Re-measured against the real server file: with HTTP/1.1 Node kept the connection
open, and a test client waited until its own timeout. The device speaks
HTTP/1.1 (later logged from `req.httpVersion`). The theory fit.

`Connection: close` was added and it was verified that the socket
closes immediately. **The groups of four stayed unchanged.**

### 2.4 The receive loop needs CR LF at the end

This condition is real and is in the code exactly like this
(`HTTPS_POST_ReceiveResponseData` @ `0x08015744`):

```c
} while ( ( (total == 0) || (idle < 0x15) ||
            (buf[total-1] != '\n') || (buf[total-2] != '\r') )
          && !elapsed(timer, timeout) && total < maxlen );
```

Early exit only if data is present, 21 idle polls have passed
**and** the last two bytes are CR LF. A response ending in `0`
runs into the full timeout — and that is `Cloud_Report_URL_Builder(1, 0x14)`,
i.e. 20 s, exactly the observed retry interval.

The endpoint was changed to a CR-LF ending, then to chunked with a
terminating chunk, byte-identical to a raw capture of the real cloud.

**Still four attempts. Still 20 seconds apart.**

---

## 3. Why all four persisted for so long

Each had a mechanism in the code, evidence and a matching number. And every
test consisted of **changing something**. A change that does not help says
almost nothing — it can be right and still insufficient.

What was missing was a **control**: the same procedure with a known good
counterpart, with only one variable changed.

---

## 4. The decisive experiment

The container got a pass-through mode: the time request is forwarded to the real
cloud and its response is returned **verbatim**, written directly to
the socket so that Node does not reformat anything. The telemetry upload stayed
local.

The result came within a ten-minute window:

| Response | Behavior |
|---|---|
| self-built, same body, same chunked framing | four requests, 20 s apart, then give up |
| real, passed through | **one** request, accepted |

The difference lay solely in the header lines:

```
ours    HTTP/1.1 200 OK · Content-Type · Date · Connection · Keep-Alive · Transfer-Encoding
real    HTTP/1.1 200 OK · Date · Content-Type · Transfer-Encoding · Connection · Trace-Id
```

Node appends `Keep-Alive: timeout=5`, which the original does not send, and
orders the headers differently. Which detail upsets the firmware is **not narrowed down**;
the endpoint has since reproduced the entire captured response.

### The upload host

A different machine behind a Kong API gateway, with seven additional
header lines: `vary`, `Access-Control-Allow-Credentials`, `X-Kong-Upstream-Latency`,
`X-Kong-Proxy-Latency`, `Via: 1.1 kong/3.9.1`, `X-Kong-Request-Id`,
`Strict-Transport-Security`.

Captured **without** telemetry: a POST with `{}` as the body is rejected with
`{"code":51,"message":"The d field is required"}` — the framing is
what mattered.

---

## 5. Call chain and error codes (for the next investigation)

```
FUN_08015bd0 case 0 (time)          FUN_08015bd0 case 1 (upload)
  Cloud_Report_URL_Builder(1,0x14)    Cloud_Report_MarstekCloud_Upload
    @0x080151AC                         @0x08016B98
  HTTPS_POST_Request @0x08015094      HTTPS_Request_Execute_CH395 @0x080174E8
    HTTPS_POST_ReceiveResponseData      mbedTLS_SSL_Recv_WithRetry @0x08015914
      @0x08015744  (CR-LF condition)      (only close_notify or timeout)
    HTTP_Response_Parse @0x080273F0     HTTPS_Response_Parse @0x08027564
      llhttp, type 2                      llhttp, type 2
```

`HTTPS_POST_Request` / `HTTPS_Request_Execute_CH395`, return `0` = success:

| Code | Cause |
|---|---|
| −1 | Arguments |
| −2 | `HTTP_URL_ExtractPath` |
| −3 | Request buffer |
| −4 | DNS / CH395 command |
| −5 | SPI send or TCP connect timeout |
| −6 | TLS session init |
| −8 | SSL send |
| −9 | Receive returned 0 bytes |
| **−10** | **llhttp parse failed** |

`Cloud_Report_MarstekCloud_Upload` maps every nonzero value to `0`, and
`FUN_08015bd0` takes the error branch on `0` — **`count` is never
decremented there**, only the retry counter `+0x10`.

### The TLS receive drops data

```c
if ((int)uVar1 < 1) {
    if (uVar1 == 0xffff8780) { ... }                       // close_notify: clean
    if ((uVar1 != 0xffff9700) && (uVar1 != 0xffff9780)) {
        return uVar1;        // returns the ERROR as the result
    }
}
```

The caller passes this value on as the **length**. A hard connection drop after six seconds therefore made llhttp parse tens of thousands of bytes from
a 3 KB buffer — a guaranteed parse error, code −10, error branch, no
decrement.

The real cloud responds with `keep-alive` and **never** closes. The device
therefore waits out its full 20-second timeout on every upload. That is
normal.

---

## 6. The 86-second cycle

Over an entire day the uploads came exactly 86 s apart, read as a
retry rhythm. It is a throttle (`case 1`, line 102):

```c
if ((3 < count) && (Tick_Timer_Check_Elapsed(DAT_0801608c, 0x3c) == 0)) { state++; break; }
```

From four buffered records on: at most one upload per 60 s. The
**disappearance** of this pattern is thus a direct indicator of a
backlog below four — and was the first signal that the fix takes effect.

---

## 7. Result

| Quantity | before | after |
|---|---|---|
| Time requests | 4 per trigger, 20 s apart | 1 per ~600 s |
| Upload interval | rigid 86 s | ~300 s, following the 5-minute push |
| CH395 resets | every 1804–1836 s | none in seven hours |
| Register 42000 | dropped back several times a day | unchanged since 02:13 |

One upload per 300 s corresponds to the push interval from
`HTTP_Economy_TOU_PeriodicHandler` — one upload per record, counter between 0
and 1. `1 < count` is thus permanently false.

After the fix, the backlog took around 45 minutes to drain.

---

## 8. Open items

1. **Which header** the firmware rejects — not narrowed down.
2. ~~**A second reset trigger.**~~ **Closed 2026-08-27.** The two
   outages with 1293 s and 1052 s spacing fell into the phase of several
   container restarts; one was four minutes after one of them. Over more than
   24 h without intervention there was no further one. It was our own disturbance. The remaining
   callers of `CH395_Reset_And_Reinit` (`BLE_Cmd_Dispatch` case 6,
   `FUN_0801fa3c` case 6, the unreferenced wrappers `FUN_0802e1ac` and
   `FUN_0804c364`) remain unobserved — they are command paths.
3. **Transferability.** One device, one firmware version, LAN.
4. **Whether the framing of the upload host is just as strict** as that of the
   time endpoint. The Kong headers were added on the reasoning
   that an equally harmless-looking difference had been decisive once before.
   That is a trade-off, not a measurement.

---

## 9. Methodological lesson

**A change that does not help is not a test.** It can be right and
insufficient at the same time. Four hypotheses survived a day because they were checked
only this way.

What disposed of them was a comparison against a **known good counterpart**
with a single changed variable. Effort: one hour of building, ten minutes
of measuring.

If an imitation is not accepted and all differences in content have been
ruled out, the cause is in what one does **not** compare — here, in
what the own library adds on its own.
