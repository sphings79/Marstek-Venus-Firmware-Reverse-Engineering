# Bug report — the device stops answering Modbus TCP for four seconds every five minutes

**Product:** Marstek Venus D (VNSD-0)
**Firmware:** Control / EMS **v150** (`VNSD-0_app_0150_0805_115146.bin`)
**Interface:** Modbus TCP, port 502, over Ethernet (CH395)
**Affected:** every Modbus TCP client, continuously, on a device with an intact
cloud connection
**Severity:** a Modbus server that is unreachable for four seconds out of every
three hundred. Most client libraries default to a three-second timeout, so this
appears to them as a dead device, not as a slow one.

**Note on addresses:** all flash addresses below are given with the correct Ghidra image base
0x08004800. An earlier pass used the wrong base 0x08000000 (a constant 0x4800 too low); the
original address is noted alongside each corrected one.

---

## Summary

Every 300 seconds the firmware uploads one telemetry record to the cloud over
HTTPS. It builds the TLS session from scratch for each upload: new context, new
DRBG seed, full TLS 1.2 handshake. On this MCU the client side of that handshake
takes **4.2 seconds of uninterrupted computation**, and while it runs, nothing
services the Modbus TCP socket on the same network chip.

The TCP connection stays open throughout — the CH395 holds it — so the client
sees no disconnect, no reset and no error. It sees a request that is simply not
answered for four seconds.

Nothing is wrong with the network, the cloud endpoint or the Modbus stack. The
cost is the handshake itself, and the device pays it 288 times a day.

## What a Modbus client sees

Measured over 18,281 requests in 57 minutes, timed at a Modbus proxy directly in
front of the device:

```
mean response time        81 ms
under 100 ms            53.7 %
under 250 ms            46.3 %
between 250 ms and 2 s     0 requests
over 2 s                  13 requests   (0.07 %)
```

There is no tail to this distribution. A request is either answered within a
quarter of a second or it waits two to four seconds. The thirteen slow ones:

```
21:35:29  2.25 s     22:02:16  4.29 s     22:22:15  4.20 s
21:37:11  4.17 s     22:07:10  4.37 s     22:27:10  4.14 s
21:42:16  3.91 s     22:12:16  4.26 s     22:32:16  4.12 s
21:47:10  4.11 s     22:17:11  3.44 s
21:52:18  2.37 s                          (UTC)
21:57:11  3.27 s
```

Spaced alternately 294 and 306 seconds apart, averaging exactly 300. The spread
between 2.2 and 4.4 seconds is not the stall varying; it is where in the stall
the request happened to arrive.

## Where the four seconds go

Packet capture at the receiving server, headers only, one upload, times relative
to the SYN:

```
 +0.000 s  device  [S ]      0 B   SYN
 +0.000 s  server  [S.]      0 B   SYN-ACK            TCP established in 1 ms
 +0.041 s  device  [P.]     78 B   ClientHello
 +0.066 s  server  [. ]   1024 B   ServerHello, Certificate,
 +0.066 s  server  [P.]    315 B     ServerKeyExchange, ServerHelloDone
 +0.300 s  server  [. ]   1024 B   remainder of the server flight
 +0.516 s  device  [. ]      0 B   ACK
                                   -- 4.22 s of silence --
 +4.740 s  device  [P.]     75 B   ClientKeyExchange
 +4.784 s  device  [P.]     51 B   ChangeCipherSpec, Finished
 +4.786 s  server  [P.]     51 B   ChangeCipherSpec, Finished
 +4.982 s  device  [P.]    988 B   POST (encrypted)
 +4.986 s  server  [P.]    511 B   response
+25.06  s  device  [F.]      0 B   FIN
+294.02 s  device  [S ]      0 B   next upload, new connection
```

The gap sits between the server's flight and the ClientKeyExchange. At that
point the device has everything it needs and is computing: parsing the
certificate, verifying the ServerKeyExchange signature, generating the ECDHE
key pair and deriving the shared secret. The server answers in milliseconds and
TCP is established in one; neither contributes.

The same four-second window, seen from the Modbus side of the same device at the
same moment, is a 4.32 s response time on port 502.

## Where it comes from in the firmware

| Step | Location |
|---|---|
| 300-second timer sets the upload flag | `HTTP_Economy_TOU_PeriodicHandler` @ `0x0801b1b4` (was `0x080169b4`) |
| cloud task, upload branch | `HTTP_ClientTask_Dispatcher` @ `0x0801a3d0` (was `0x08015bd0`), `case 1` |
| build and send the record | `Cloud_UploadTelemetryData` @ `0x0801b398` (was `0x08016b98`) |
| HTTPS over the CH395 | `HTTPS_FetchUrlData` @ `0x0801bce8` (was `0x080174e8`) |
| TLS session, built from scratch | `HTTPS_TLSSessionInit` @ `0x0803b8a0` (was `0x080370a0`) |

The timer, at `0x0801b1b4`:

```c
if (((*(char *)(state + 0x11) == '\0') && (*(short *)(state + 2) != 0)) &&
    (Tick_Timer_Check_Elapsed(timer, 300) != 0)) {
    *(undefined1 *)(state + 0x11) = 1;   // upload requested
    *(undefined1 *)(state + 0x10) = 3;   // three attempts
}
```

And the session setup at `0x0803b8a0` (`HTTPS_TLSSessionInit`), in full:

```c
HTTPS_TLS_Context_Reset();
mbedtls_rsa_alt_init_once();
mbedtls_ctr_drbg_init(...);   mbedtls_entropy_init(...);
mbedtls_ctr_drbg_seed(...);
mbedtls_ssl_setup(...);
mbedtls_ssl_handshake(...);
```

No session cache, no resumption, no reuse. Note that the connection from the
previous upload stayed open for twenty seconds after its response before the
device closed it — the socket was available, only the session was thrown away.

## Why it blocks Modbus

Modbus TCP and the cloud upload share one chip. `CH395_Init_Modbus_TCP_Socket` @ `0x0804dcd0`
(was `0x080494d0`) opens port `0x1f6` = 502 on the CH395, and every SPI command to that chip is
serialised behind a single mutex, `g_ch395CmdMutex` (representative call site:
`CH395CMDGetSocketStatus` @ `0x080074d4`, was cited as `0x08002cd4`), `xQueueSemaphoreTake(...,
5000)`. While the upload path owns the chip and the CPU, nothing reads the Modbus socket.

## The control experiment

The same device fetches the time every ten minutes from the same server, over
plain HTTP on port 80, through the same chip, the same task and the same mutex:

```
21:45:58, 21:55:59, 22:06:00, 22:16:01, 22:26:02, 22:36:03, 22:46:04
```

Not one of those causes a measurable delay on port 502. The difference between
the two paths is the TLS handshake and nothing else.

## Three ways to fix it, cheapest first

**1. Reuse the session.** The connection already lingers for twenty seconds
after the upload. Keeping it, or caching the session and resuming
(`mbedtls_ssl_session_save` / `_load`), removes the handshake from 287 of every
288 uploads.

**2. Make the handshake yield.** Your build already contains the restartable
ECC implementation — `mbedtls_ecdh_compute_shared_restartable` at `0x08038670`
(was `0x08033e70`) and `mbedtls_ecdh_gen_public_restartable` at `0x08038710`
(was `0x08033f10`) — but it is driven
without a budget, so the computation runs to completion in one call. Enabling
`MBEDTLS_ECP_RESTARTABLE` with `mbedtls_ecp_set_max_ops()` lets the handshake
return between operations, and the Modbus socket gets serviced in the gaps.
This is a configuration change to code you already ship.

**3. Do not hold the chip.** Release the CH395 mutex across the cryptographic
work, or run the cloud task below the network service task in priority.

Any one of the three removes the symptom. The first is the smallest change and
also saves the device a full handshake every five minutes.

## Note for anyone reproducing this

Poll any implemented holding register once per second and record the response
times for fifteen minutes. Three stalls will appear, about five minutes apart.
A client timeout above five seconds turns them into slow reads rather than
errors, which is how we confirmed that nothing is actually lost.

The behaviour needs a reachable upload endpoint. With the upload failing, the
firmware follows a different path — see the 1800-tick network-chip reset in
`HTTP_ClientTask_Dispatcher` (`0x0801a3d0`, was cited as `FUN_08015bd0`), which is a separate and
more disruptive problem (see `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20.md`'s
cross-references and the v150→v151 diff, where this same watchdog was restructured — v151 changes
the backlog threshold from >1 to ≥5, the delay from 1800 to 14400 ticks, and adds a
`System_Reboot()` call, see "Status in v151" below).

## Other models

Verified on a Venus D running Control/EMS v150. Venus A and Venus E v3 share
this firmware base and we would expect the same behaviour, so we suggest fixing
it in the shared codebase rather than in the Venus D build alone.

## Status in v151 (checked 2026-10-01)

**Unchanged.** Every function in the chain exists in v151 at the same size, moved only by the
heap growth (0x11800 → 0x14000 bytes) that shifts everything after it:

| Function | v150 | v151 |
|---|---|---|
| `HTTP_Economy_TOU_PeriodicHandler` | 0x0801b1b4 | 0x0801b27c |
| `HTTP_ClientTask_Dispatcher` | 0x0801a3d0 | 0x0801a4a4 |
| `Cloud_UploadTelemetryData` | 0x0801b398 | 0x0801b460 |
| `HTTPS_FetchUrlData` | 0x0801bce8 | 0x0801be7c |
| `HTTPS_TLSSessionInit` | 0x0803b8a0 | 0x0803bf70 |
| `CH395_Init_Modbus_TCP_Socket` | 0x0804dcd0 | 0x0804e3a0 |
| `CH395CMDGetSocketStatus` | 0x080074d4 | 0x080074d4 (unchanged, same address) |
| `mbedtls_ecdh_compute_shared_restartable` | 0x08038670 | 0x08038d40 |
| `mbedtls_ecdh_gen_public_restartable` | 0x08038710 | 0x08038de0 |

`HTTPS_TLSSessionInit` is instruction-identical between v150 and v151 (only local-variable
symbolization differs — v151 resolves `g_httpsSslConfig` etc. to named globals where v150 shows
raw addresses like `0x60021e40`, a cosmetic decompiler difference, not a code change). It still
builds a fresh `mbedtls_ssl_context`/`mbedtls_ctr_drbg`/entropy pool and runs one blocking
`mbedtls_ssl_handshake()` call per upload — no session cache, no `mbedtls_ssl_session_save`/`_load`.
Neither v150 nor v151 calls `mbedtls_ecp_set_max_ops()` anywhere in the image (checked across both
full decompiled trees), so the restartable ECC entry points listed above remain present but unused
in v151 too — suggested fix 2 has not been applied. `HTTP_Economy_TOU_PeriodicHandler` still fires
the upload every 300 ticks. The four-second Modbus stall on every cloud upload is present unchanged
in v151.

The related watchdog in `HTTP_ClientTask_Dispatcher` **did** change (see
`Control_FW/VNSD_Control_Diff_v150_v151.md`): the CH395-hang backlog threshold moved from `> 1` to `>= 5`, the
wait from 1800 to 14400 ticks, and a new `System_Reboot()` plus a suppression flag were added. That
is a different code path (the chip-hang recovery, triggered when the upload endpoint is
unreachable) from the TLS-handshake stall described here, which fires on every successful upload
and is unaffected by that change.

## Related

We reported an independent defect in the Modbus read path — exception responses
declare one byte more than they contain (`VNSD_FW_Bug_Report_EN_Exception_Frame_Length.md`),
which makes every rejected request look like a timeout. We have had no response
to that report. The two are unrelated in cause but compound each other in
effect: a client that waits out a rejection is also waiting through these
stalls.

Raw captures, the register dumps and the firmware analysis behind these
measurements are available on request.
