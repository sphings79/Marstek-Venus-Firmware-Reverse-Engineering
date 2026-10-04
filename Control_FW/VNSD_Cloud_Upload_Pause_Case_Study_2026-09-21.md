# Case study: the 4-second pause every 5 minutes

> 🇩🇪 Deutsche Version: [VNSD_Cloud_Upload_Pause_Fallstudie_2026-09-21_de.md](VNSD_Cloud_Upload_Pause_Fallstudie_2026-09-21_de.md)

*Created 2026-09-21. Device: Venus D, Control firmware v150, LAN, behind
modbus-proxy. Offline endpoint active, so no more 30-minute reset.
Related: `VNSD_Cloud_Watchdog_Case_Study_2026-08-26.md` (the reset that the
endpoint eliminated) and `VNSD_Control_FW_Analysis_app_0150_0805_115146.md` §4a.*

The 30-minute reset was the coarse problem. Once it is gone, a finer one
shows up underneath: **the device does not respond for about four seconds
every five minutes.** This file shows where that comes from and what about it is not
a defect.

---

## 1. Observation

On the production Home Assistant, two integrations run through the same
proxy against the same battery: the Marstek Modbus Suite (all registers
enabled) and Omnibattery. A capture of both containers ran during the night.

In 57 minutes:

| | |
|---|---|
| Requests through the proxy | 18,281 |
| Timeouts of the Suite (3 s) | 4, each with a connection rebuild |
| `received pdu without a corresponding request` (pymodbus) | 5 |
| `transaction_id mismatch` | 6 |
| Errors in the proxy | 0 |
| lost values | 0 |

The outages were not scattered but fell on a grid of about five
minutes.

## 2. The response time is bimodal

Measured at the proxy, from `sending` to the device until `received`:

```
Mean              81 ms
under 100 ms    53.7 %
under 250 ms    46.3 %
250 ms to 2 s    0 requests
over 2 s        13 requests   (0.07 %)
```

Not a distribution with tails, but two separate worlds: either under
a quarter of a second, or two to four seconds. The thirteen outliers:

```
21:35:29  2.25 s     22:02:16  4.29 s     22:22:15  4.20 s
21:37:11  4.17 s     22:07:10  4.37 s     22:27:10  4.14 s
21:42:16  3.91 s     22:12:16  4.26 s     22:32:16  4.12 s
21:47:10  4.11 s     22:17:11  3.44 s
21:52:18  2.37 s     (all times UTC)
21:57:11  3.27 s
```

Intervals alternate between 294 and 306 seconds, averaging exactly 300.

## 3. The constant in the firmware

`HTTP_Economy_TOU_PeriodicHandler` @ `0x080169b4`, last block:

```c
if (((*(char *)(DAT_08016b20 + 0x11) == '\0') && (*(short *)(DAT_08016b20 + 2) != 0)) &&
   (iVar3 = Tick_Timer_Check_Elapsed(DAT_08016b94, 300), iVar3 != 0)) {
    *(undefined1 *)(DAT_08016b20 + 0x11) = 1;   // upload requested
    *(undefined1 *)(iVar2 + 0x10) = 3;          // three attempts
}
```

**300 seconds.** The unit is secured via `0x708`, which is evidenced as 1800 seconds in the same
firmware (§4a). The condition is a record in the
ring buffer (`+2 != 0`), created at the slot change via
`RTC_TimeOfDay_To5MinSlotIndex` @ `0x08014168`.

It is a **relative** timer, not a wall-clock grid: it is reset after every fire
and only polled when the state machine of the cloud task
comes around. Hence 294/306 instead of a rigid 300.

## 4. What takes four seconds in it

The chain from the flag to the wire:

| Step | Address |
|---|---|
| 300-s timer sets upload flag | `HTTP_Economy_TOU_PeriodicHandler` @ `0x080169b4` |
| Cloud task, `case '\x01'` | `FUN_08015bd0` |
| Prepare and send record | `Cloud_Report_MarstekCloud_Upload` @ `0x08016b98` |
| HTTPS over the Ethernet chip | `HTTPS_Request_Execute_CH395` @ `0x080174e8` |
| TLS session **from scratch** | `HTTPS_TLS_Session_Init` @ `0x080370a0` |

The session is rebuilt completely every time:

```c
HTTPS_TLS_Context_Reset();
mbedTLS_RSA_Alt_InitOnce();
mbedTLS_CTR_DRBG_Init(...);   mbedTLS_Entropy_Init(...);
mbedTLS_CTR_DRBG_Seed(...);   // collect entropy anew
mbedTLS_SSL_Setup(...);
mbedTLS_SSL_Handshake(...);   // full TLS 1.2 handshake
```

No session resumption, no keep-alive. Before that comes the socket setup with a
wait loop `vTaskDelay(10)` up to `500`. Which part of this
costs the four seconds is covered in §5a — measured, not estimated.

**Why Modbus hangs along with it:** same chip. `CH395_Init_Modbus_TCP_Socket` @
`0x080494d0` opens port `0x1f6` = 502 on the CH395, and every SPI command to
this chip runs through a mutex (`CH395_SPI_Cmd_WithResponse` @
`0x08002cd4`, `xQueueReceive(..., 5000)`). As long as the cloud path keeps the chip
busy, nobody reads out the Modbus sockets. The TCP connection stays up
in the meantime — the CH395 holds it —, it is just that nobody responds.

## 5. The proof from outside

The offline endpoint on the Raspberry Pi logs every request with an
ISO timestamp (`/data/requests-<datum>.jsonl`). On the left the measurement at the proxy,
on the right the POST at the endpoint:

```
Pause begins    silent  Device responds  Upload POST      Gap
   21:37:11    4.17 s     21:37:15.2      21:37:16.366      1.20 s
   21:42:16    3.91 s     21:42:19.9      21:42:21.022      1.11 s
   21:47:10    4.11 s     21:47:14.1      21:47:15.283      1.17 s
   21:52:18    2.37 s     21:52:20.4      21:52:21.278      0.91 s
   21:57:11    3.27 s     21:57:14.3      21:57:15.258      0.99 s
   22:02:16    4.29 s     22:02:20.3      22:02:21.297      1.01 s
   22:07:10    4.37 s     22:07:14.4      22:07:15.309      0.94 s
   22:12:16    4.26 s     22:12:20.3      22:12:21.200      0.94 s
   22:17:11    3.44 s     22:17:14.4      22:17:15.351      0.91 s
   22:22:15    4.20 s     22:22:19.2      22:22:20.145      0.95 s
   22:27:10    4.14 s     22:27:14.1      22:27:15.265      1.12 s
   22:32:16    4.12 s     22:32:20.1      22:32:21.303      1.18 s
   22:46:51    3.90 s     22:46:54.9      22:46:55.887      0.99 s
```

Thirteen events, thirteen uploads, the gap each time between 0.91 and
1.20 seconds. The upload intervals on the Pi's clock: 295, 305, 294, 306, 294,
306, 294, 306, 294, 305, 295, 306, 295, 305 — the same alternating pattern as
in the Modbus data.

## 5a. Where the four seconds lie

Capture on the Pi, headers only (`tcpdump -s 96`), filtered to the battery
and port 443. One connection, times relative to the SYN:

```
 +0.000 s  client  [S ]      0 B   SYN
 +0.000 s  server  [S.]      0 B   SYN-ACK          TCP up after 1 ms
 +0.041 s  client  [P.]     78 B   ClientHello
 +0.066 s  server  [. ]   1024 B   ServerHello, Certificate,
 +0.066 s  server  [P.]    315 B     ServerKeyExchange, HelloDone
 +0.300 s  server  [. ]   1024 B   rest of the server flight
 +0.516 s  client  [. ]      0 B   ACK
                                   -- 4.22 s silence --
 +4.740 s  client  [P.]     75 B   ClientKeyExchange
 +4.784 s  client  [P.]     51 B   ChangeCipherSpec, Finished
 +4.786 s  server  [P.]     51 B   ChangeCipherSpec, Finished
 +4.982 s  client  [P.]    988 B   POST (encrypted)
 +4.986 s  server  [P.]    511 B   response
 +25.06 s  client  [F.]      0 B   FIN
+294.02 s  client  [S ]      0 B   next upload
```

**The pause lies between the server flight and the ClientKeyExchange.** The
device has everything it needs and stays silent for 4.22 seconds — that is pure
computation time on the Cortex-M4: parsing the certificate, verifying the signature of the
ServerKeyExchange, generating the ECDHE key and computing the shared
secret. Network and server have no share in this; the Pi responds
in milliseconds.

Everything else is negligible: TCP 1 ms, ClientHello 41 ms, server flight
260 ms, from Finished to the response 250 ms.

Cross-check across both ends: this connection (start 01:12:16.341 local)
corresponds at the proxy to a Modbus pause of **4.32 s at 23:12:16 UTC**. The next
connection, 294 s later, yields the same computation pause and 3.27 s at the proxy —
shorter only because the Modbus request arrived only in the middle of the window.

Side finding: the connection stays open for another **20 seconds** after the response
before the device sends FIN. It is not reused — the next
session starts from scratch again.

## 6. The control: HTTP costs nothing

The same endpoint answers a time query every ten minutes, over **HTTP
on port 80, without TLS**:

```
21:45:58, 21:55:59, 22:06:00, 22:16:01, 22:26:02, 22:36:03, 22:46:04
```

At none of these points in time is there a pause. Same chip, same task,
same socket machinery, same mutex — just without a handshake, and nothing
is measurable.

This narrows the cause down to the setup of the TLS session, not to
the CH395, not to the network and not to the endpoint.

## 7. Consequence for the integration

The Modbus Suite waited 3 seconds and rebuilt the connection after every timeout:

```
00:07:13  Rebuilding the connection after a timeout on block read
00:07:14  Reconnected
```

Raised to **5 seconds** on 21.09.2026 at 00:43 (`DEFAULT_TIMEOUT`,
`const.py`; a config entry reload is not enough, Python has cached the module —
core restart required). The first pause afterwards, 00:46:51 with 3.90 s,
went through without a timeout and without a reconnect.

In the 25 minutes after that: five pauses (3.09 to 4.34 s), **zero timeouts, zero
reconnects**. Before, each of them would have cost one.

Five seconds covers everything measured so far; the largest value ever
was 4.77 s in a measurement from the previous evening. The margin is thus thin,
six seconds would be the calmer choice.

### Omnibattery is not affected — and that is no coincidence

One might expect that the connection rebuild of the Suite drags the second
integration along. The numbers say no: `received pdu without a
corresponding request` and `transaction_id mismatch` occur before the change
at 0.19 per minute and afterwards at 0.20 per minute — i.e. unchanged, even though
the Suite no longer reconnects. They are tied to the pauses, not to the
Suite: one message per pause.

Omnibattery is designed for exactly this case. `READ_TIMEOUT_S` is set to 3 seconds for the
v3 family, with two pymodbus-internal retries, and the
comment in the source code names the mechanism:

> occasionally stalls for several seconds and then flushes queued replies in a
> burst. Each pymodbus-internal retry re-sends the SAME transaction_id, so a
> reply arriving after a per-attempt timeout still matches the retry and is
> consumed. Keep v3 attempts short so the retries land inside the stall window.

The retry thus catches the delayed response. What then appears in the log as
`without a corresponding request` is the **second** response to the
same transaction — a duplicate, not a loss. Consistent with that: in 57 minutes
not a single incomplete poll and no missing value.

Two integrations, two answers to the same firmware behavior: short
attempts with the same transaction number (Omnibattery) or a timeout
above the pause (Suite). Both work; the second is quieter.

## 8. Open items

1. **The 2.25-s gap at 21:35:29** has no associated upload. It lies
   24 seconds after the start of the capture; presumably an artifact of the
   pairing at the start of the file. Not pursued further, not explained away.
2. **Three attempts.** The flag is set with `+0x10 = 3`. What happens if
   an upload fails and is retried twice has not been measured — the
   pause should then get longer.
3. **The 60-second throttle.** If more than three records pile up, `case '\x01'`
   allows one upload per 60 s (§6 of the watchdog case study). The
   5-minute dropout would become a one-minute cycle. Not observed so far, because the
   endpoint keeps the buffer empty.
