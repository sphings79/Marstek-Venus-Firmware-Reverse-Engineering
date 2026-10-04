# Control FW v150 — Analysis and Diff against v149.2

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_Analyse_app_0150_0805_115146_de.md](VNSD_Control_FW_Analyse_app_0150_0805_115146_de.md)

> **Superseded by `VNSD_Control_FW_v150_Full_Analysis.md` (2026-09-30).** This version is based on the wrong
> image base 0x08000000: **all flash addresses here +0x4800** (real address = quoted address + 0x4800), and
> function names derived from flash strings are partly wrong. Field measurement results remain valid.

**Binary:** `VNSD-0_app_0150_0805_115146.bin`
**Reference:** `Control_149.2_VNSD-0_app_1492_0702_142136.bin` (v149.2)
**Analysis date:** 2026-08-13
**Tool:** Ghidra 12.1.1 (pyghidra) + ReVa MCP, Version Tracking diff

---

## 1. Binary fingerprint

| | v149.2 | v150 |
|---|---|---|
| File | `Control_149.2_VNSD-0_app_1492_0702_142136.bin` | `VNSD-0_app_0150_0805_115146.bin` |
| MD5 | `59c0c9f15c43e442c2861e592aafd11a` | `0c21645b9dc622a08e7de5ac1d269bd5` |
| Size | 385,024 B | 389,120 B (**+4,096 B**) |
| Flash base | `0x08000000` | `0x08000000` |
| Address range | `0x08000000`–`0x0805DFFF` | `0x08000000`–`0x0805EFFF` |
| Language | ARM:LE:32:Cortex (Thumb-2) | ARM:LE:32:Cortex (Thumb-2) |
| Initial SP | — | `0x2001F7E8` (SRAM from `0x20000000`) |
| Reset vector | — | `0x08004A70` (Thumb) |
| Functions | 1,622 | **1,618** |
| Named | 1,618 | **1,618 (100 %)** |
| Symbols | 4,381 | 4,200 |
| Version string | `VNSD-0 v1492` | `VNSD-0 v150` |
| Build stamp | `Jul  2 2026 14:20:03` | **`Aug  5 2026 11:51:21`** |

### MCU identification: Nations N32G45x, not Artery AT32F403A

- The Control/EMS MCU is a **Nations N32G45x with XFMC** (External Memory Controller). This version of the document did not yet name the MCU family explicitly; for the authoritative identification see `VNSD_Control_FW_v150_Full_Analysis.md` §1a.
- Evidence: QSPI controller at 0xA0001000 with N32 QSPI_STS bits; RCC PLL multiplier in bits 18–21 + 27 (N32 PLLMULFCT/PLLMULFCT_H); GPIOF/GPIOG used; SAC crypto coprocessor at 0x40024000 (previously wrongly called "RNGX"); 96-bit UID @ 0x1FFFF7F0; 102 vectors.
- External SRAM runs via XFMC, control registers at 0xA0000000: the official Nations header `n32g45x.h` (local, not in the repo) defines `XFMC_REG_BASE 0xA0000000` and `QSPI_BASE = XFMC_REG_BASE + 0x1000`; the Nations driver library contains `n32g45x_xfmc.c`; the firmware itself has the shell command `xfmc_test`.
- Details/header copy: `../Hardware_Recherche/README.md`, `../Hardware_Recherche/MCU_Identifikation/n32g45x.h`.

### Ghidra preparation

The image was originally imported with base `0x00000000`. Corrected via
`setImageBase(0x08000000, commit=True)`; the vector table check confirms the base
(initial SP `0x2001F7E8`, reset `0x08004A71`, all exception vectors in the range `0x080xxxxx`).
Afterwards full analysis (1,029 → 1,618 functions).

**Version Tracking diff** (source = v149.2, destination = v150), correlators:
`symbol-name`, `exact-bytes`, `exact-instructions`, `exact-mnemonics`,
`duplicate-instructions`, `function-reference` and additionally `combined-reference`.

| Result | Count |
|---|---|
| Matched | 1,567 |
| of which identical | 1,511 |
| of which changed | 56 |
| Only in v149.2 (removed/moved) | 55 |
| Only in v150 (new/unmatched) | 51 |
| Markup transferred automatically | 1,566 functions |
| Renamed manually afterwards | 52 |
| **Name duplicates after completion** | **0** |
| Plate comments in v150 | 89 |

---

## 2. Key message — what has changed?

v150 is **not a bugfix release**, but brings three substantial innovations:

1. **Cloud migration from `hamedata.com` (HTTP) to `marstekcloud.com` (HTTPS).**
2. **A completely new HTTPS/TLS stack over the CH395 Ethernet chip** (mbedTLS,
   dedicated socket, request-building and parsing functions).
3. **New "Peak Shaving" feature (load peak capping)**, configurable via BLE
   *and* MQTT, with two new telemetry fields.

In addition there are numerous smaller changes (EEPROM layout, clamping logic of the
power setpoint, buffer sizes) and the activation of code that was already in flash in v149.2
but not referenced by any function (dead code).

---

## 3. Cloud reporting: hamedata → marstekcloud

### Removed (v149.2)

```
http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s
AT+QHTTPCFG="url","http://%s.hamedata.com/prod/api/v1/setVenusDReporting?v=%s"
[URL Base64 data (%d byte)]: %s.
[HTTP] URL err, need %d bytes(all:%zu)
[HTTP] Upload data_len : %d, buf_size: %d, data: %s, key: %s.
"de":                                   (upload decrypt result flag)
```

### New (v150)

```
https://api-%s.marstekcloud.com/data-upload/v1/venus/%s
AT+QHTTPCFG="url","https://api-%s.marstekcloud.com/data-upload/v1/venus/%s"
AT+QHTTPCFG="header","Content-Type","application/json"
AT+QHTTPPOST=%d,60,60
{"d":"di=%s&sn=%s&to=%d&td=%d&ed=%d&…&soh=%d"}
```

**Meaning:** In v149.2 the telemetry upload ran as an **HTTP GET with an
AES-128-ECB-encrypted, Base64-encoded query** against `*.hamedata.com`
(function `Cloud_Reporting_setVenusDReporting`). In v150 an
**HTTPS POST with a JSON body** (`{"d":"<key=value&…>"}`) is sent instead to
`api-<region>.marstekcloud.com/data-upload/v1/venus/<sn>`.
The Base64/AES path for the URL is dropped without replacement, as is the evaluation of the
server flag `"de":` (upload decrypt result).

New central function: **`Cloud_Report_MarstekCloud_Upload` @ `0x08016B98`** (1,548 B).

Also new for the modem path:

* `Quectel_SSL_Set_SNI` @ `0x08010684` — `AT+QSSLCFG="sni",<ctx>,<en>`
* String `"CA",1` — CA configuration for the modem TLS session

---

> **What is actually uploaded there** has been measured since 2026-08-25 and is documented in
> `VNSD_Cloud_Upload_Payload_v150.md` — including the confirmation that
> the region abbreviation is `eu`.

## 4. New HTTPS stack over CH395 (Ethernet)

In v150 there is a second, complete HTTP path that runs **TLS in software
(mbedTLS) directly over the CH395 TCP socket** — in parallel to the existing
HTTP path. The firmware even contains the original source function names as
strings: `Init_ch395_https`, `Pack_https_post_req`, `Parse_https_data`, `on_bodys_cb`.

| Address | Name (newly assigned) | Size | Function |
|---|---|---|---|
| `0x080174E8` | `HTTPS_Request_Execute_CH395` | 398 B | Orchestrator: socket → TLS → request → parse |
| `0x080370A0` | `HTTPS_TLS_Session_Init` | 264 B | mbedTLS session, **authmode 0 = no certificate verification**, TLS 1.2, SNI |
| `0x080158D0` | `HTTPS_TLS_Context_Reset` | 52 B | Context reset of the HTTPS path |
| `0x080474BC` | `mbedTLS_SSL_CloseNotify` | 58 B | TLS `close_notify` |
| `0x0801821C` | `CH395_TCP_Socket_Open_HTTPS` | 150 B | Twin of `CH395_TCP_Socket_Open_HTTPPort80` |
| `0x080270E8` | `HTTPS_BuildRequestBuffer` | 174 B | Twin of `HTTP_BuildRequestBuffer` |
| `0x08027564` | `HTTPS_Response_Parse` | 80 B | Byte-identical twin of `HTTP_Response_Parse` (llhttp) |
| `0x08027478` | `HTTPS_Response_Marker_Matcher` | 218 B | Pattern matching in the response stream (confidence: low) |
| `0x08015914` | `mbedTLS_SSL_Recv_WithRetry` | 114 B | `mbedTLS_SSL_Read` with retry |
| `0x0801598C` | `mbedTLS_SSL_Send_WithRetry` | 118 B | `mbedTLS_SSL_Write` with retry |

New request templates:

```
POST %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\nUser-Agent: CH395-Client\r\n
Content-Type: application/json\r\nContent-Length: %d\r\n\r\n%s

POST %s HTTP/1.1\r\nHost: %s\r\nConnection: keep-alive\r\nUser-Agent: CH395-Client\r\n…
```

New log strings: `[HTTPS] Url(%d) : %s.`, `[HTTPS] TCP connect timeout, state=0x%02X`,
`[HTTPS] SSL send failed: %d`, `[HTTPS] SSL recv failed: %d`,
`[HTTPS] Parse http data failed`, `SSL send error: -0x%04X`,
`SSL send timeout, written=%d/%d`, `ssl setup fail -0x%04X`.

> **Security-relevant:** `HTTPS_TLS_Session_Init` calls
> `mbedTLS_SSL_Conf_Authmode(conf, 0)` — i.e. `MBEDTLS_SSL_VERIFY_NONE`.
> The new HTTPS path does **not** verify the server certificate. The MQTT session
> (`mbedTLS_SSL_Connection_Init` @ `0x08018934`) continues to use mutual
> authentication with a client certificate and CA chain.

---

## 4a. The 30-minute reset of the Ethernet bridge — mechanism

*(2026-08-25. Trigger: field reports of Modbus dropouts at an exact
30-minute interval on Venus E v3 and Venus D under v150. The log string
`[HTTP]ch395 reset!!!!` had already been noted in §6 as a "new recovery path"; here
is when it fires.)*

`FUN_08015bd0` (HTTP/cloud task, 2048 B) contains in line 44 of the decompilation:

```c
if (((1 < *(ushort *)(DAT_08015fcc + 2)) &&
     (Tick_Timer_Check_Elapsed(DAT_0801600c, 0x708) != 0)) &&
    (*_DAT_08016010 == 0)) {
    CH395_Reset_And_Reinit(0);
    log_printf(3, 1, "[HTTP]ch395 reset!!!!");
    *DAT_0801600c = 0;
    *DAT_08015fc0 = '\0';
}
```

`0x708` = **1800 seconds**. `CH395_Reset_And_Reinit` pulls the reset line of the
chip (100 ms low, 50 ms after release), reinitializes it and afterwards reopens the
Modbus TCP socket, broadcast listener and TCP server. During the
reset the bridge is completely gone — **including ICMP**, which field measurements via ping
over LAN cable confirm.

### The three conditions

| Condition | Address | Finding |
|---|---|---|
| `1 < count` | u16 @ **0x2000100C** | Number of buffered telemetry records |
| 1800 s elapsed | Tick @ 0x20001FD6 | Fixed timer, zeroed after the reset |
| `flag == 0` | u16 @ **0x2000014C** | = Modbus register **37001** (`status_block_offset8`) |

**The counter.** `DAT_08015fcc`, `DAT_0800c3b8` (in `Telemetry_History_Record_Push`)
and `DAT_08016b20` (in `HTTP_Economy_TOU_PeriodicHandler`) all three contain
`0x2000100A` — it is the same ring buffer header. `Telemetry_History_Record_Push`
@ `0x0800C2F0` stores records of `0x35D` bytes each (max. `0x90` = 144) and
increments `+2` in the process.

**Records are only invalidated on successful upload.** In `FUN_08015bd0`:

```c
iVar6 = FUN_0801774c(_DAT_080160c0);        // evaluate response body
if (iVar6 == 0) {                            // accepted
    ...
    memcpy(record[count-1], record[count], 0x35d);
    *(short *)(base + 2) = *(short *)(base + 2) + -1;   // counter minus 1
}
```

`FUN_0801774c` @ `0x0801774C` is remarkably simple:

```c
iVar2 = strstr(param_1, "\"code\":");     // string @ 0x08017784, verified
if (iVar2 == 0) return 0;
return atoi(iVar2 + 7);                     // log: "[HTTP] Upload decrypt result = %d."
```

So the server must return `"code":0` in the body for the record to count as
delivered. If the upload fails, the counter stays put and grows
with every further record.

**The third flag is practically ineffective.** `0x2000014C` is, according to
`Modbus_RS485_TCP/VNSD_Register_Map_Final_generated.csv`, register
**37001** (status block `0x20000144` +8, writer
`Protocol_AA_SetDeviceParams` @ `0x0802FC48`, reader `Grid_Export_Power_Limiter`).
In **all** existing scan logs — `langzeit2`, `pack1_bms_fault_2026-08-21`,
`unter_dod_backup…` — 37001 is consistently **0**, across all
operating states. The condition is therefore always satisfied on this device;
what the field means semantically remains open (confidence in the register map:
low).

### The WLAN twin: `FUN_0800c420` — 900 s instead of 1800 s

The same mechanism exists a second time for the FC41D/Quectel path. The
task `FUN_0800c420` (called from `LAB_0802E76E`) is the twin of
`FUN_08015bd0` (from `LAB_0802E6F4`) — same structure, same ring buffer,
different transport (`Quectel_HTTP_GET_StateMachine` instead of CH395). Line 127:

```c
if ((*(short *)(DAT_0800c814 + 2) != 0) &&
    (Tick_Timer_Check_Elapsed(_DAT_0800c920, 900) != 0)) {
    Quectel_Modem_HardwareReset_Handler(0);
    log_printf(3, 1, "[HTTP]fc41d reset!!!!");   // string @ 0x0800C923
    *_DAT_0800c920 = 0;
    *DAT_0800c818 = 0;
}
```

`DAT_0800c814` contains `0x2000100A` — **the same ring buffer header** as on the
Ethernet side (verified). The comparison:

| | Ethernet (CH395) | WLAN (FC41D/Quectel) |
|---|---|---|
| Task | `FUN_08015bd0` | `FUN_0800c420` |
| Reset | `CH395_Reset_And_Reinit` | `Quectel_Modem_HardwareReset_Handler` |
| Timer | **1800 s** (`0x708`) | **900 s** |
| Backlog | `> 1` record | `!= 0` — a single record suffices |
| Additional condition | reg. 37001 == 0 | none |

**WLAN is thus the harsher variant**, not the milder one: twice as often and
with a lower threshold. Switching the connection is not a workaround.

The remaining lines 134–1086 of `FUN_0800c420` are the Quectel AT state
machine (cases 0–0x23) with short protocol timeouts (300/600/5000 ms); case 9
contains the invalidation path including a bulk deletion
(`[HTTP] Delete data upload_cnt`). Two further modem resets in lines 67–78 are
counter-driven (`> 6`, `> 10`), not time-driven. `CH395_Reset_And_Reinit`
does not occur in this task. **Read completely 2026-08-25.**

**Not checked:** whether both tasks run in parallel or only the one matching the
active link.

### Consequence

**The 30-minute reset is a cloud upload watchdog.** As long as the telemetry
is not successfully delivered to `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s`,
the buffer grows, and the firmware resets the Ethernet chip every 1800 s
— on the assumption that the network path is broken. It is not;
only the cloud does not respond.

That explains the field observations completely:

- **why it does not hit everyone** — if the cloud is reachable, the buffer is emptied,
  the counter stays ≤ 1, the condition never triggers
- **why newer communication module firmware changes nothing** — the reset comes from
  the Control firmware, the module merely carries it out
- **why the interval is exact** — it is a timer, not a fault case

Correlation, not proven: on an affected Venus D,
`cloud_status` (register 30302) is at 0 with an intact WLAN/LAN connection. Whether
30302 maps exactly this upload path has not been verified.

### Why the interval persists — counter dynamics (2026-08-25, field test)

*(This subsection corrects an earlier version. The prediction formulated there
"a cold start ends the interval" was **refuted** in the field; the
reasoning for it rested on an unverified assumption about the
request flags. The sequence is corrected and evidenced below.)*

#### The field test

Offline endpoint responds correctly (`{"code":0,...}`, time response in UTC in the
original format). Venus D via LAN, cold start at 12:11:51. Ping at one-second intervals:

| Event | Time | Interval |
|---|---|---|
| Cold start (device gone) | 12:11:51 | — |
| Reset 1 | 12:47:24 | **2133 s** |
| Reset 2 | 13:17:47 | 1823 s |
| Reset 3 | 13:48:11 | 1824 s |
| Reset 4 | 14:18:35 | 1824 s |

The interval survives the cold start. **But the first interval is 309 s longer
than the following ones**, and that is not noise: `Tick_Timer_Check_Elapsed` stays
true once the 1800 s are up — the tick is only zeroed by the reset itself
(`FUN_08015bd0`, line 50). The reset thus fires as soon as the *last* still-open
condition becomes true. After the cold start that was `1 < count`.

**Thereby evidenced:** `count` is actually set to 0 at boot (consistent
with `.bss`), rises above 1 again within the first minutes and then stays up.
The steady-state period is 1823–1824 s, i.e. 1800 s plus ~24 s
loop latency.

#### Who creates records and who requests uploads

Both are done by `HTTP_Economy_TOU_PeriodicHandler` @ `0x080169B4`, called in
`FUN_08015bd0` line 34 on every task pass.

**Record** (lines 22–30): as soon as the 5-minute slot index
(`RTC_TimeOfDay_To5MinSlotIndex`) differs from the last stored one and 20 s
have elapsed → `Telemetry_History_Record_Push` → `count + 1`.

**Upload request** — two paths:

```c
iVar3 = HexChar_To_TimeOffsetIndex();
if (iVar3 == 0) { f_0x10 = 3; f_0x0f = 1; }        // immediately
else FreeRTOS_xTimerCreate("HTTP_TIMER", iVar3*60000, 0 /*one-shot*/, 1, cb);
```

and additionally lines 52–56, independently of that:

```c
if (f_0x11 == 0 && count != 0 && elapsed(DAT_08016b94, 300)) {
    f_0x11 = 1; f_0x10 = 3;
}
```

`HexChar_To_TimeOffsetIndex` @ `0x0801E50C` reads a **single character** at
`0x20018959` (= `*0x0801E550` + 0xB = `0x2001894E` + 0xB), decodes it as a
hex digit (0-9, A-F, a-f) and returns `value % 5`, i.e. **0..4 minutes**. That is
a device-dependent offset — presumably for load distribution of the cloud.

**Not resolved:** which buffer is located at `0x2001894E`. There are no
Ghidra references to it because the address is reached via a Thumb-2 literal pool
(known limitation, see `Methodik_und_Meta`). Candidates would be the
serial number or device ID. **If the offset is 0 or 1..4
depending on the device, that would be a candidate for why only some of the devices
are affected** — that is a hypothesis, not a finding.

#### Correction: the buffer *can* drain

The earlier version claimed "one upload per record, the counter cannot
decrease". That is wrong. The entry gate of `case 1` reads:

```c
if (f_0x11 == 0 && f_0x0f == 0 && f_0x10 == 0) { state++; break; }
```

After a successful upload only `f_0x11` and `f_0x0f` are cleared (lines
133–134). **`f_0x10` stays set** — it is decremented exclusively on failures
(lines 123 and 174) and cleared completely when `count == 0` is
reached (lines 91–94). Since the periodic handler sets `f_0x10 = 3`, the
gate stays open and the state machine keeps uploading in consecutive
passes until the buffer is empty.

From `count > 3` a throttle of 60 s per upload additionally applies
(lines 102–106).

**Consequence for the diagnosis:** If the buffer *can* drain but in the field
permanently stays above 1, then uploads are either not requested or
not accepted. Exactly three failed counts (`f_0x10 = 3`) close the gate
again.

**Next step:** Measure the upload frequency at the endpoint. The expectation in
balanced operation is 12 uploads/h (one 5-minute slot per record).
Significantly fewer proves that uploads fail or fail to occur; significantly more
would be incompatible with a stuck counter.

### The receive loop requires CR LF at the end (2026-08-25)

*(This is the root of the field observations above. Found after the own
endpoint responded correctly and the 30-minute interval persisted anyway.)*

The call chain of a cloud query is:

```
FUN_08015bd0 case 0
  -> Cloud_Report_URL_Builder(1, 0x14)        @ 0x080151AC   (0x14 = 20 s timeout)
     -> HTTPS_POST_Request(6, 20, url, len, 0) @ 0x08015094
        -> HTTPS_POST_ReceiveResponseData(...) @ 0x08015744
        -> HTTP_Response_Parse()               @ 0x080273F0   (llhttp, type 2)
```

Error codes of `HTTPS_POST_Request`: `-2` path, `-3` request buffer, `-4`
DNS/CH395 command, `-5` SPI send, **`-6` receive returned 0 bytes**,
**`-7` llhttp parse failed**. `0` = success.

The receive loop (`param_6 == 0`, the non-OTA branch):

```c
do {
    vTaskDelay(0x32);
    len = CH395_GetRecvLen(sock);
    if (len == 0) idle++;
    else { idle = 0; len = CH395_ReadRecvBuf(sock, buf + total, len); total += len; }
} while ( ( (total == 0) || (idle < 0x15) ||
            (buf[total-1] != '\n') || (buf[total-2] != '\r') )
          && (Tick_Timer_Check_Elapsed(&t, timeout) == 0) && total < maxlen );
```

**The early exit requires three things simultaneously:** data has
arrived, `0x15` = 21 consecutive empty polls (with `vTaskDelay(0x32)`
about one second of silence), **and the last two bytes in the buffer are
`CR LF`**. Otherwise the loop runs into the full timeout.

#### How the real cloud responds

Raw capture from `eu.hamedata.com` (port 80, plain HTTP), 2026-08-25:

```
Content-Type: text/html; charset=utf-8\r\n
Transfer-Encoding: chunked\r\n
Connection: keep-alive\r\n
Trace-Id: f93b4d8dba0ecf18bac995270755bf2e\r\n
\r\n
1d\r\n
_2026_08_25_15_16_11_04_0_0_0\r\n
0\r\n
\r\n
```

**Chunked, not `Content-Length`.** The final chunk `0\r\n\r\n` ensures
that the response ends in `CR LF` — the condition is thus satisfied without
the body itself carrying a line break. `Connection: keep-alive`, so **no**
connection close by the server.

#### Field evidence

An endpoint that responds with `Content-Length` and whose body ends in `0`
runs into every timeout. Measurement at the own endpoint:

```
12:31:48  time   +620 s      <- 600-s trigger
12:32:08  time   + 20 s
12:32:28  time   + 20 s
12:32:48  time   + 20 s      <- four attempts, then give up (retry > 3)
```

The interval is exactly the timeout parameter `0x14`. And between the four
attempts **no** upload runs — the state machine hangs in `case 0` for the whole minute
instead of stepping through its cases. On success the code makes exactly **one**
query (lines 61–68 of `FUN_08015bd0`).

`Connection: close` alone does not fix it: re-measured on 2026-08-25, the device
speaks HTTP/1.1, the server closed immediately afterwards — the groups of four remained.
It is the line ending, not the connection.

#### SOLVED 2026-08-26: it was the header lines

The CR LF condition above is in the code as stated and is described correctly — but it was
**not** the cause. Aligning the framing (chunked, keep-alive, no
`Content-Length`, byte-identical to the real cloud) changed nothing: still
groups of four at 20-second intervals.

A **pass-through experiment** was decisive: the own endpoint forwarded the
time query to the real cloud and returned its response verbatim,
written directly to the socket.

| Response | Behavior of the device |
|---|---|
| self-built, same body, same chunked framing | four requests, 20 s interval, then give up |
| real, passed through | **one** request, accepted |

The difference lay exclusively in the header lines:

```
ours    HTTP/1.1 200 OK · Content-Type · Date · Connection · Keep-Alive · Transfer-Encoding
real    HTTP/1.1 200 OK · Date · Content-Type · Transfer-Encoding · Connection · Trace-Id
```

Node appends `Keep-Alive: timeout=5`, which the real endpoint does not send, and
orders them differently. **Which of the two details disturbs the firmware is open** —
the endpoint has since replicated the captured response completely.

The upload host required the same treatment. It sits behind a
Kong API gateway and sends seven header lines that the time endpoint does not send
(`vary`, `Access-Control-Allow-Credentials`, two `X-Kong-*-Latency`, `Via`,
`X-Kong-Request-Id`, HSTS). Captured with a POST without payload, which the
gateway rejects with `{"code":51,"message":"The d field is required"}`.

#### The 86-second interval was a throttle, not a timeout

Over large parts of 2026-08-25 the uploads came exactly 86 seconds apart,
which was read as a retry rhythm. It is line 102 from `case 1`:

```c
if ((3 < count) && (Tick_Timer_Check_Elapsed(DAT_0801608c, 0x3c) == 0)) { state++; break; }
```

From four buffered records on, the firmware throttles to one upload per
60 seconds. **The disappearance of this pattern is thus the direct indicator
that the backlog has fallen below four** — and was the first signal that the
fix takes effect, even before the ping measurement caught up.

#### Confirmation in the field

After deploying the corrected endpoint at 00:03:

| Quantity | before | after |
|---|---|---|
| Time queries | 4 per trigger, 20 s interval | **1** per ~600 s |
| Upload interval | rigid 86 s | **~300 s**, following the 5-minute grid |
| CH395 resets | every 1804–1836 s | **none in seven hours** |
| RS485 control mode (42000) | dropped several times daily | unchanged since 02:13 |

The ping series:

```
21:55:14           23:56:43   +1804
22:25:39   +1825   00:27:08   +1825
22:56:03   +1824   ——— corrected endpoint ———
23:26:39   +1836   nothing more
```

One upload per 300 s corresponds exactly to the push interval from
`HTTP_Economy_TOU_PeriodicHandler` — one upload per record, i.e. a counter
oscillating between 0 and 1. Thus `1 < count` is permanently false and the
reset condition unfulfillable.

#### Still open

1. **Which header line exactly** the firmware rejects. Not narrowed down.
2. ~~**A second reset trigger.**~~ **Resolved 2026-08-27.** Two outages
   shortly after the fix were 1293 s and 1052 s apart, i.e. below the
   lower bound of 1800 s that this mechanism must observe. Both fell into
   the phase in which the container was restarted several times — one of them four
   minutes afterwards. Since then, over more than 24 h without intervention, **not a single
   outage**. It was our own restarts, not a second trigger.
   Further callers of `CH395_Reset_And_Reinit` exist (`BLE_Cmd_Dispatch`
   case 6, `FUN_0801fa3c` case 6, as well as the unreferenced wrappers `FUN_0802e1ac`
   and `FUN_0804c364`), but they are command paths and not observed in the field.
3. **Transferability.** One device, one firmware version, LAN. Reports from
   Venus E v3 fit the picture, but no other build has been decompiled.

Complete case study with all refuted interim hypotheses:
`VNSD_Cloud_Watchdog_Case_Study_2026-08-26.md`.

### The second price of v150: four seconds of Modbus per upload

*(Confirmed 2026-08-27 on a second, third-party device — Venus D, v150, LAN,
11.7 h capture.)*

Independent of the watchdog, v150 costs roughly four seconds on **every** telemetry upload,
during which the device does not serve its Modbus server.

**Cause: v150 switched the upload from plain-text HTTP to TLS.**

| | v149.2 | v150 |
|---|---|---|
| Endpoint | `http://%s.hamedata.com/prod/api/v1/setVenusDReporting` | `https://api-%s.marstekcloud.com/data-upload/v1/venus/%s` |
| Transport | Plain text | TLS, `HTTPS_Request_Execute_CH395` @ `0x080174E8` |
| TLS session | — | `HTTPS_TLS_Session_Init` @ `0x080370A0`, *"not present in v149.2"* |
| Upload function | — | `Cloud_Report_MarstekCloud_Upload` @ `0x08016B98`, new |

Before v150, an upload cost practically no compute time. Since v150, each
upload incurs a key exchange, and on this Cortex-M it takes several seconds in software.

#### Measurement

Raw capture of a third-party device, 11.7 h after a cold start, with
a responding offline endpoint and MQTT switched off:

| Quantity | Value |
|---|---|
| Modbus responses | 27 627 |
| Normal interval (median) | 2.44 s |
| Gaps > 3.5 s | **141** = 12.0/h |
| TLS ClientHellos | **141** |
| Gaps that coincide with a handshake | **141 of 141** |
| Longest gaps | 6.7–6.9 s |

**Not a single unexplained outage.** Twelve per hour corresponds exactly to the
5-minute push interval — one handshake per data record.

In the packet capture the compute effort is directly visible: between
`Server Hello Done` and `Client Key Exchange`, 4.4 s elapse, during which the
device keeps answering ARP and UDP, but not Modbus.

#### Distinction from the watchdog

Both look like "Modbus drops out" from the outside, but they are different:

| | Watchdog (§4a) | TLS compute pause |
|---|---|---|
| Duration | 2–5 s | 4–7 s |
| ICMP/ARP during that time | **gone** | keeps running |
| Signature | DHCP re-request, multiple ARP probes | none |
| Cycle | 1800 s | per upload, i.e. ~300 s |
| Fixable | yes, keep the buffer empty | **no** |

The device's own ARP probe cycle (one per minute) is suitable as a detector:
If it breaks off, it was a stack restart; if it keeps running, it was compute time.

#### Consequence for Modbus clients

A response timeout below roughly **8 seconds** inevitably produces an error on v150
every five minutes, regardless of what the network looks like. This affects every
v150 Venus, even without an offline endpoint and with the cloud reachable.

### Consequence for Open Item 5 (§12): Authmode 0 is also the solution

Because `HTTPS_TLS_Session_Init` works with `MBEDTLS_SSL_VERIFY_NONE`, without a
CA chain and without a client certificate, the upload can be redirected to a **custom
endpoint**:

| Step | Evidenced by |
|---|---|
| DNS override for the api hostname | String @ `0x08017214` |
| TLS on **port 443**, any certificate | `CH395_TCP_Socket_Open_HTTPS`: `desc+8 = 0x1BB`; Authmode 0 |
| Reply with `{"code":0}` | `FUN_0801774c` |

Then the buffer empties, the reset never fires, and no data record leaves the
local network. **Not covered:** the MQTT path
(`mbedTLS_SSL_Connection_Init`, Authmode 2 with client certificate) — it keeps
running independently.

The region placeholder `%s` in the hostname is filled from a pointer table at
`0x20000FD0` (pointer is at `DAT_08017250`), index from EEPROM `0x441`
(`Cloud_Report_MarstekCloud_Upload`, lines 22–32). The table is located in **RAM**
and cannot be resolved from the image — see Open Item 7.

---

## 5. New feature: Peak Shaving

Completely new in v150, controllable via two paths:

**BLE:**
```
[BLE] Set Peak shaving, power = %d, peak_state = %d.
```

**MQTT:**
```
[MQTT] Type: %d, Set peak status = %d, peak power = %d.
[MQTT] Set peak shaving err.
[MQTT] Type: %d, Set peak status = %d.
[MQTT] Set peak power err.
[MQTT] Type: %d, Get INV data info...
```

**Telemetry:** The large reporting string was extended by two fields:

```
… ,soh=%d,peak_status=%d,peak_power=%d
```

Removed in exchange is the MQTT command
`[MQTT] Type: %d, Set manual mode or economy mode info...` together with the payload `cd=%d,md=%d`.

> **Relevant for the HA integration:** `peak_status` and `peak_power` are new
> controllable quantities. Whether they are additionally reachable via Modbus (RS485/TCP)
> has not yet been verified — the register descriptor table is built at runtime
> and has not yet been scanned for v150. **Open item.**

---

## 6. Other telemetry changes

| String | Change |
|---|---|
| BMS block `cd=%d,b_ver=…` | **+ `self_check=%d,mos=%d`** |
| Main block `cd=%d,tot_i=…` | **+ `peak_status=%d,peak_power=%d`** |
| `g_st=%d,g_flag=%d,g_bau=%d,…,g_minpw=%d,g_maxpw=%d` | **new** — generator/grid parameter block |
| `bms_ver=\t%d`, `vns_ver=\t%d` | **new** (shell/debug output) |
| `bat_mode=%d` | removed |
| `[HTTP] Read Eco-Tracker ip: %s.` | moved (`0x0800D817` → `0x0800DA40`) |
| `[HTTP]fc41d reset!!!!`, `[HTTP]ch395 reset!!!!` | **new** — recovery paths |
| `[FC41D] HTTP P1 meter data: %s.` | now referenced |
| `+QMTSUB: 0,1,0,0`, `+QMTPUB: 0,0,0`, `+QHTTPGET: 9` | removed |
| `Tmr Svc` | removed; **new** `FreeRtos_mem` |

---

## 7. Functions with changed content (decompiler diff verified)

| Function | v149.2 | v150 | Sim. | Δ bytes | Change |
|---|---|---|---|---|---|
| `MQTT_Credential_Buffer_Decode` | `0x08005860` | `0x080057CC` | 0.58 | +54 | **Method changed** (see below) |
| `Quectel_MQTT_Unsubscribe` | `0x0800E6F4` | `0x0800C21C` | 0.58 | −24 | `log_printf`/`memset`/`sprintf` removed |
| `Quectel_HTTP_AT_SendAndVerify` | `0x0800BCC4` | `0x0800BEF4` | 0.62 | +6 | Timing/check logic slightly reworked |
| `Write_Handler` | `0x08050F20` | `0x08051D14` | 0.63 | 0 | FreeRTOS write handler reworked |
| `Modem_ConnInfo_UpdateMeterIP` | `0x0801379C` | `0x08013A50` | 0.65 | 0 | Meter IP adoption reworked |
| `Inverter_PowerSetpoint_DeadbandClamp` | `0x08013CC8` | `0x08013F7C` | 0.69 | +30 | **new clamping direction** (see below) |
| `mbedTLS_SSL_Connection_Init` | `0x08018364` | `0x08018934` | 0.73 | −6 | no longer calls `MQTT_TLS_Context_Reset` |
| `HTTP_Response_Parse` | `0x08026930` | `0x080273F0` | 0.74 | +2 | minimal |
| `Parse_IP_Address_String` | `0x08004CD0` | `0x08004C3C` | 0.81 | 0 | only source line number in the log (0x182 → 0x2C9) |
| `HTTPS_POST_Request` | `0x08014DC0` | `0x08015094` | 0.83 | 0 | **Buffer 0x1958 → 0x1DC0 B** (6488 → 7616) |
| `MQTT_TLS_Context_Reset` | `0x08012984` | `0x08012C38` | 0.84 | 0 | — |
| `Config_Write_String_0x388` | `0x08006E3C` | `0x08006E04` | 0.86 | 0 | **EEPROM 0x388 → 0x387** |

### 7.1 `Inverter_PowerSetpoint_DeadbandClamp` — new clamping direction

```c
      else if (*(char *)(configBase + 0x85) == '\x01') {   // NEW in v150
        if (param_3 < local_1e) { param_3 = local_1e; }    // lower limit
      }
      else if (local_1e <= param_3) {                      // as in v149.2
        param_3 = local_1e;                                // upper limit
      }
```

A new configuration byte at **struct offset `+0x85`** (SRAM base `0x20014D00`)
reverses the clamping direction of the power setpoint. It is written by the
likewise new function **`Config_Write_PowerClampMode_0x394` @ `0x08006B84`**,
which sets byte `+0x85` and halfword `+0x87` and persists **6 bytes starting at EEPROM address `0x394`**.

> Very likely the persistence slot of the Peak Shaving feature
> (`peak_state` + `peak_power` = 1 byte + 2 bytes). **Still to be verified.**

### 7.2 `MQTT_Credential_Buffer_Decode` — deobfuscation replaced

v149.2 called a fixed routine `Flash_Obfuscated_String_Decode`. In v150 this
function is **completely gone** (the second caller
`sscanf_Format_Parser` no longer calls it either). Instead:

```c
    local_28 = 0;
    for (i = 0; i < ((len - off) - 1 & 0xffff); i++) {
      r = Flash_ReadWords(&local_28, 1);
      if (r != 0x5a5a5a5a) return;                     // abort on error
      buf[off + i] = alphabet64[local_28 % 0x40];      // 64-character table
    }
    buf[len - 1] = 0;
```

The rest of the credential buffer is filled from a 64-character table
(`DAT_08005848`), indexed by a value read from flash
(magic `0x5A5A5A5A` as validity check).

---

## 8. Dead code in v149.2 — activated in v150

These functions or string copies were already in flash in v149.2, but were
**not referenced by any function** there. In v150 they are active:

| Address (v150) | Name | Evidence |
|---|---|---|
| `0x0801479C` | `Quectel_HTTP_GET_StateMachine` | `[HTTP] GET success/timeout/retry`, `[HTTP] Data read OK.` |
| `0x0800AE48` | `BLE_GATT_Add_Service` | `AT+QBLEGATTSSRV=%s` |
| `0x0800AFC0` | `BLE_GATT_Add_Characteristic` | `AT+QBLEGATTSCHAR=%s` |
| `0x0800AD08` | `BLE_Query_Device_Name` | `AT+QBLENAME?`, `MST_VNSD_` |
| `0x0800221C` | `Timer_Callback_BLE_Modem_Supervisor` | `*** xTimerStart end 1..........` |
| `0x0802A898` | `CLI_Cmd_FactoryReset` | `Reset, clear all/cert/part...` |
| `0x08002CD4` | `CH395_SPI_Cmd_WithResponse` | second copy of the mutex error string |

Newly added in addition: **`BLE_Stack_Init_Sequence` @ `0x0800A4DC`** (678 B) — orchestrates
`BLE_Module_Init`, `BLE_Set_Device_Name`, `BLE_Query_Device_Name`,
`BLE_Set_AdvParams`, `BLE_AdvertisingControl` and `Quectel_AT_QVERSION_Query`.

> The BLE GATT server configuration (service + characteristic) is thus actually
> issued for the first time in v150. Relevant for BLE-based integrations.

---

## 9. Code duplicates / twins

v150 contains several functions twice — identical instruction sequence, but
its own data/context reference. The cause is apparently the duplication of the
HTTP module for the HTTPS path.

| Original | Twin | Role of the twin |
|---|---|---|
| `HTTP_Response_Parse` `0x080273F0` | `HTTPS_Response_Parse` `0x08027564` | byte-identical, HTTPS path |
| `HTTP_BuildRequestBuffer` `0x08027028` | `HTTPS_BuildRequestBuffer` `0x080270E8` | HTTPS path |
| `CH395_TCP_Socket_Open_HTTPPort80` `0x08018164` | `CH395_TCP_Socket_Open_HTTPS` `0x0801821C` | HTTPS path |
| `MQTT_TLS_Context_Reset` `0x08012C38` | `HTTPS_TLS_Context_Reset` `0x080158D0` | HTTPS path |
| `RCC_LSICmd` `0x08017D84` | `RCC_LSICmd_Alias` `0x08028B8C` | used by `RTC_ConfigClockSource` |
| `RCC_RTCCLKCmd` `0x08017D90` | `RCC_RTCCLKCmd_Alias` `0x08028B98` | used by `RTC_ConfigClockSource` |
| `fp_Set_Exception` `0x08028B80` | `fp_Set_Exception_Alias` `0x08031FF0` | used by `fp64_pow`/`fp64_sqrt_…` |
| `Quectel_HTTP_AT_SendAndVerify` `0x0800BEF4` | `…_Unused` `0x0800BD98` | **0 callers — dead code** |

> **Pitfall in version tracking:** The `combined-reference` correlator initially
> matched three of these twin pairs the wrong way round
> (`MQTT_TLS_Context_Reset`, `mbedTLS_SSL_Connection_Init`,
> `Quectel_HTTP_AT_SendAndVerify`). The matching was corrected based on
> string references and caller chains; the affected functions
> carry a corresponding plate comment in Ghidra.

---

## 10. Other newly named functions (selection)

| Address | Name | Origin |
|---|---|---|
| `0x08003DC0` | `SMR_TIC_SelfUse_PowerController` | v149.2 `0x08003E54` |
| `0x080074AC` | `CH395_SendData_WaitResponse` | v149.2 `0x080074E4` (+54 B) |
| `0x0800584C` | `MQTT_Topic_Builder` | v149.2 `0x080058E0` (−24 B) |
| `0x080058CC` | `MQTT_Topic_Log_XidDevice` | log helper split out |
| `0x0800DCF4` | `HTTP_Cloud_Reporting_Dispatcher` | v149.2 `0x0800DAC8` |
| `0x080151AC` | `Cloud_Report_URL_Builder` | v149.2 `0x08014ED8` |
| `0x08018698` | `MQTT_Connect_And_Subscribe` | v149.2 `0x080180C8` |
| `0x0801F698` | `MQTT_Publish_BMS_Full_Data` | v149.2 `0x0801F0D0` (−98 B) |
| `0x08022A14` | `MQTT_Publish_DevelopModeInfo` | newly named |
| `0x08022B34` | `MQTT_Publish_ErrorEventLog` | `err%d=%d|…` via modem |
| `0x08026C70` | `DeviceInfo_BuildStatusString` | v149.2 `0x08026344` (**+136 B**) |
| `0x08029FB0` | `Remote_Power_Setpoint_Process` | v149.2 `0x08029458` (**+62 B**) |
| `0x0802D360` | `WorkMode_State_Machine` | v149.2 `0x0802C784` (−18 B) |
| `0x0802D6BC` | `Cloud_EdgeDetectAndWatchdog` | v149.2 `0x0802CAF4` (+18 B) |
| `0x0802E874` | `Tick_Timer_Check_Elapsed` | role swap with `Tick_Timer_Expired` |
| `0x0802FECC` | `MQTT_Telemetry_String_Formatter` | v149.2 `0x0802F2F0` (**852 → 370 B**) |
| `0x0800B4F4` | `BLE_Build_BMS_Data_Response` | v149.2 `0x0800B430` (+10 B) |
| `0x0800C090` | `Energy_TOU_Counters_Load` | read path from v149.2 `0x0800BE50` |
| `0x08017D64` | `IWDG_Start` | writes `0xCCCC` to `IWDG_KR` (`0x40003000`) |
| `0x08017D74` | `IWDG_ReloadCounter` | writes `0xAAAA` to `IWDG_KR` |
| `0x0802A888` / `0x0802E1B8` / `0x08013920` | `Http_BusyFlag_Clear` / `_Set` / `_Get` | byte `+0x39` at `0x20017D8C` |
| `0x08002750` | `RCC_PeripheralReset_Pulse` | `RCC_APB1PeriphResetCmd` |
| `0x0800B7F4` | `Stub_Return_One` | stub, 0 callers |

---

## 11. Removed functions (no longer findable in v150)

`Base64_Encode`, `Flash_Obfuscated_String_Decode`, `Cloud_Reporting_setVenusDReporting`,
`HTTP_ParseUploadDecryptResultFlag`, `MQTT_Telemetry_EnergyCounter_Update`
(only the read path survives), `BLE_Recv_Cmd_Dispatcher` in its old form,
`Task_Init_CreateAll`, `WiFi_*` reset family (`WiFi_Module_RestartStateMachine`,
`WiFi_HardwareResetSequence`, `WiFi_ModuleResetDispatcher`, `WiFi_ResetWithRecoveryWait`,
`WiFi_PowerCycleSequence`), `Relay_StagedTimingControl`, `Register_ToggleBit15`,
`ConditionalSystemReboot`, `Standby_Wakeup_Debounce`, `Config_Get_WorkMode`,
`Config_Read_ProductionDate`, `Debug_PrintModbusAddress`,
`Register_Write_PackedAsciiValue_Group0xCC`, `GPIO_TogglePin_Periodic`,
`System_SoftReset`, `FPU_EnableCoprocessorAccess`, `vPortSetupTimerInterrupt`,
`vTaskStartScheduler`, `FreeRTOS_StartScheduler`, `FreeRTOS_xTimerCreateTimerTask`.

> Some of these entries are **not a real removal**, but a non-match by
> version tracking (Ghidra VT has no body-similarity correlator). The FreeRTOS scheduler functions and the
> WiFi reset family are, with high probability, still present,
> just at a different address and already captured in v150 under other names.
> **Open item for a follow-up session:** targeted cross-check of these 25 names.

---

## 12. Open items

1. **Modbus register map for v150 not scanned.** Whether `peak_status` / `peak_power`
   are reachable via Modbus TCP/RS485 (and under which registers) is open.
   The descriptor table is built at runtime — a live scan against a
   device with v150 is required.
2. **EEPROM layout shift `0x388 → 0x387`** and the new block starting at `0x394`
   (6 bytes) have not yet been fully reconciled against the existing EEPROM map.
   *(Partially done 2026-08-14: The config block `0x200`–`0x206` is evidenced —
   see `Modbus_RS485_TCP/VNSD_Register_Persistence_RAM_vs_EEPROM_v150.md`.)*
3. **`HTTPS_Response_Marker_Matcher` @ `0x08027478`** — 0 callers, purpose only
   hypothesized (confidence: low).
4. **Cross-check 25 functions listed as "removed"** (see §11).
5. **`Authmode 0` in the new HTTPS path** — assess practical impact (MITM risk for the
   cloud upload); include in the security documentation if appropriate.
   *(Assessed 2026-08-25, see §4a: the upload can be redirected to an arbitrary
   endpoint without any certificate. The flip side of the same coin — exactly this
   is the only way to switch off the 30-minute reset without a firmware patch.)*
6. **Semantics of register 37001 / `0x2000014C`** open. It gates the
   CH395 reset (§4a), is 0 in all scan logs, and the register map lists
   it with confidence "low". Writer `Protocol_AA_SetDeviceParams`, reader
   `Grid_Export_Power_Limiter`.
7. **Region table at `DAT_08017250`** cannot be resolved from the image.
   `0x08017250` contains the pointer `0x20000FD0` — so the table is located in **RAM**
   and is filled at runtime; there are no cross-references to `0x20000FD0`.
   The concrete api hostname per region is therefore not statically provable. Ways:
   capture the device's DNS request, or SWD dump (see
   `Methodik_und_Meta/VNSD_SWD_JTAG_Dump_Plan.md`).

---

## 13. Reproduction

```
# 1. Correct image base
setImageBase(0x08000000, commit=True)

# 2. Full analysis
analyze-program  programPath=/VNSD-0_app_0150_0805_115146.bin  forceFullAnalysis=true

# 3. Diff session
diff-create-session
  source=/Control_149.2_VNSD-0_app_1492_0702_142136.bin
  destination=/VNSD-0_app_0150_0805_115146.bin
diff-add-correlator  correlator=combined-reference

# 4. Transfer markup
diff-transfer-markup  confidence=0.5

# 5. Match remaining functions manually (string references + caller chains)
```
