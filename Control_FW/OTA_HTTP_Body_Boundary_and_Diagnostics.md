# OTA HTTP Body Boundary and OTA Diagnostics (Control FW, Venus D / A / E3)

> 🇩🇪 Deutsche Version: [OTA_HTTP_Body_Boundary_und_Diagnose_de.md](OTA_HTTP_Body_Boundary_und_Diagnose_de.md)

**Status:** 2026-10-14. Cross-device reference. All addresses are in the respective Control image (base 0x08004800). Everything below is derived from code listings and decompilations; nothing was tested on hardware.

The starting point was an external issue (external reporter) that compared E3 Control 148 and 150. The reporter's claims were re-checked and extended to D and A.

## 1. Summary

- Older Control images select the start of the HTTP body in the OTA download with `strstr(response, "Connection:") + 21`. This is only correct if `Connection: close` is the last header. Newer images use `strstr(response, "\r\n\r\n") + 4`.
- The change point differs per family: **E3 149 -> 150**, **D 149 -> 149.2**, **A: already the new form in 150** (older A versions not analysed).
- The OTA reader (ota_mode = 1) evaluates the headers only on the first socket read, never accumulates a split header and does not check the selector result for NULL. This is identical in every version checked, old and new selector alike. Real server header order and read boundaries are not known.
- BLE command 0x13 returns a 20 x 14 byte persistent error history. It is not the component-OTA result. Component-OTA diagnostics live in separate 8-byte slots (codes 0x400..0x404, console `get_ota_log`).
- The footer model check (0x403) is `strstr(model_literal, footer_copy)` (arguments in this order). Real footers pass only because of this order.
- CAN category 0x18 exists only in the 151 line (D 151, E3 151, A 1509 and A 151). All checked receivers ignore it. Purpose unknown.

## 2. HTTP body-start selector (function `HTTP_FindHeaderBodySeparator`)

Literal / advance: **old** = `"Connection:"` / +0x15 (21), error log `[HTTP] get bin_start error .`; **new** = `"\r\n\r\n"` / +4, error log `[HTTP] Not found split mark \r\n\r\n, header incomplete`. On a miss both return NULL; a NULL input also returns NULL. The only caller is `HTTPS_POST_ReceiveResponseData`.

| Family | Version | Selector address | Literal / advance | Evidence |
|---|---|---|---|---|
| E3 | 147 | 0x08016EF0 | old | raw Thumb bytes |
| E3 | 148 | 0x08017570 | old | raw Thumb bytes |
| E3 | 149 | 0x08017A58 | old | raw Thumb bytes |
| E3 | 150 | 0x08018238 (42 B) | new | Ghidra, raw listing, literal pool |
| E3 | 151 | 0x080182E0 | new | Ghidra, same code as 150 |
| D | 147 | 0x08017C34 (44 B) | old | Ghidra (live) |
| D | 149 | 0x08018744 (44 B) | old | Ghidra (live) |
| D | 149.2 | 0x08018C58 (42 B) | new | Ghidra (live) |
| D | 150 | 0x08018F2C (42 B) | new | Ghidra (live) |
| D | 151 | 0x08018FD8 (42 B) | new | Ghidra (live) |
| A | 150 | 0x08018DCC (42 B) | new | Ghidra (xrefs, literal at 0x08018DF8) |
| A | 1509 | 0x08018DEC | new | Ghidra (live) |
| A | 151 | 0x08018E78 | new | Ghidra (live) |

Change point per family: E3 149 -> 150; D 149 -> 149.2; A already new in 150 (no older A image analysed).
**Not analysed:** A 148 / 1487 / 149 / 1508 (no programs at the correct base) and E3 144 / 1476.

Corresponding reader (`HTTPS_POST_ReceiveResponseData`, 348 B in D and A; 0x08019240 -> 0x08019310 in E3 150 -> 151): E3 148 0x08018560, E3 150 0x08019240, E3 151 0x08019310, D 147 0x08018BFC, D 149 0x08019744, D 149.2 0x08019C74, D 150 0x08019F44, D 151 0x0801A018, A 150 0x08019DE4, A 1509 0x08019E08, A 151 0x08019EB8.

Pointer model of the old selector: with `Connection: close` as last header the result is correct; with another last header (for example `keep-alive`) it points 5 bytes too early; with `Connection:` before another header it points into the next header; without a `Connection:` header it returns NULL. These are results of the pointer arithmetic, not observed server responses.

## 3. OTA reader first-read behaviour

Chain: `Task_ch_http` -> HTTP client dispatcher (OTA branch) -> `HTTPS_POST_Request(6, 0x14, url_slot, len, 1)` -> `HTTPS_POST_ReceiveResponseData(..., ota_mode = 1)` -> {`HTTP_ParseContentLengthHeader`, selector, `OTA_WriteDataToRingBuffer`}. ota_mode = 1 is set only by the OTA branch of the dispatcher; `HTTP_GetUrlData` and the telemetry upload use ota_mode = 0.

OTA mode (ota_mode = 1):
- Every `CH395GetRecvData` writes to the **start** of the receive buffer; there is no header accumulation.
- On the first read only (flag cleared immediately): `HTTP_ParseContentLengthHeader(buf)`, `p = selector(buf)`, `n = read_len - (p - buf)`, `OTA_WriteDataToRingBuffer(p, n, 0)`, progress += n.
- **No NULL check on `p`.** If the selector returns NULL (no delimiter in the first read): `n = read_len + buf` (a pointer value is added to the progress counter), `OTA_WriteDataToRingBuffer(NULL, ...)` returns -1 (E3 150; not re-read for D/A), and every following read is treated as body data. The header is never retried; a header split across reads is never reassembled.
- Loop exit (D / A): 180-tick timer, 500 idle polls, or progress >= Content-Length. Returns 1 if progress != 0, else -1.
- With the old selector the same path is taken when `Connection:` is missing from the first read.

Non-OTA mode (ota_mode = 0): appends at `buf + offset`, ends on a trailing `\r\n` after more than 20 idle polls, on timeout or when the buffer is full; the selector is not used.

Identity across versions: the first-read sequence is byte-for-byte the same in E3 148 and 150 apart from BL offsets (raw comparison); E3 150 and 151 are identical after address masking; D 147 / 149 / 149.2 and D 150 / 151 and A 150 / 1509 / 151 are identical after address and variable-name normalisation.

Evidence for the missing NULL check: E3 150 and 151 from the raw Thumb listing plus decompilation; E3 148 from hand-decoded raw bytes; D and A from the decompiler output only (no separate instruction-level reading).

**Not known:** the real server header order and the real socket read boundaries (whether the first read ever lacks the delimiter), chunked transfer, and what a failed body-start does to the downstream verification. No hardware test was performed.

## 4. BLE 0x13 error history vs component-OTA diagnostic slots

### 4.1 BLE 0x13 (general persistent error history)

`BLE_App_Command_Dispatcher` case 0x13 (log `[BLE] Get err code info.`) -> `BLE_Send_Response(0x13, g_displayErrorLog, 0x118)`. 0x118 = 20 records x 14 bytes:

| Offset | Size | Field |
|---|---|---|
| 0 | u16 | year (little endian) |
| 2 / 3 / 4 / 5 | u8 each | month / day / hour / minute |
| 6 | u32 | code1 |
| 10 | u32 | code2 |

Stored in EEPROM at 0x1100 + n x 14 (20-entry ring). The loader validates each record (year 2000..0x898, month <= 12, day <= 31, hour <= 24, minute <= 60), otherwise the record is reset. The writer `EventLog_Record_DisplayError(code1, code2)` ignores (0,0) and a repeat of the last pair, suppresses identical codes on the same month/day within 31 minutes and otherwise replaces the oldest record. A byte inside code1/code2 has no OTA-stage meaning on its own.

### 4.2 Component-OTA diagnostic slots

`OTA_Set_SlotStatus(slot, status, detail)`: `slot < 4`; 16-bit status at `base + slot*8`, 32-bit detail at `base + slot*8 + 4` (detail written only if non-zero). The block is also persisted via `Config_SaveWithCRC` (A: EEPROM 0x4000, 0x24 B, CRC16 over 0x22 B).

| Code | Meaning | Writer |
|---|---|---|
| 0x400 | communication watchdog timeout (miss count > 0x3C while slot state 0) | `Comm_Watchdog_CheckTimeout` |
| 0x401 | Modbus CRC16 mismatch, detail = CRC | `OTA_FW_Verify_SetStatus` |
| 0x402 | footer device mask mismatch, detail = mask | `OTA_FW_Verify_SetStatus` |
| 0x403 | footer model string mismatch, detail = first 4 footer bytes | `OTA_FW_Verify_SetStatus` |
| 0x404 | CAN update failure (error flag set; slots 2/3/0 by result code), 0 on success | `CAN_UpdateResultHandler` |
| 0 | slot cleared | `OTA_Firmware_Download_Init` |

Console: `ShellCmd_get_ota_log` prints crc, `ota_flag`, `fail_flag` and `err_code%d` / `err_data%d` for the four slots. No other writer of 0x400..0x404 exists in the A version checked by xref.

### 4.3 Per-version addresses

| Item | E3 150 | E3 151 | D 147 | D 149 | D 149.2 | D 150 | D 151 | A 150 | A 1509 | A 151 |
|---|---|---|---|---|---|---|---|---|---|---|
| BLE dispatcher (0x13) | 0x0800C0D4 | not re-read | 0x0800C194 | 0x0800C31C | 0x0800C758 | 0x0800C720 | 0x0800C730 | 0x0800C5B8 | 0x0800C5C8 | 0x0800C5C8 |
| Error loader | 0x080172A4 | 0x08017344 | 0x080170B8 | 0x080177A8 | 0x08017CBC | 0x08017F70 | 0x08018014 | 0x08017E10 | 0x08017E28 | 0x08017EB4 |
| Error writer | 0x0802B8CC | 0x0802BD14 | 0x0802B99C | 0x0802D33C | 0x0802D7B0 | 0x0802E308 | 0x0802E800 | 0x0802DCB8 | 0x0802E098 | 0x0802E1A8 |
| `OTA_Set_SlotStatus` | 0x0802BBD4 | 0x0802C01C | 0x0802BCA4 | 0x0802D644 | 0x0802DAB8 | 0x0802E610 | 0x0802EB08 | 0x0802DFC0 | 0x0802E3A0 | 0x0802E4B0 |
| Slot array (code / detail) | 0x20017E28 / 0x20017E2C | 0x2001A640 | 0x20017B78 / 0x20017B7C | 0x20017DC8 / 0x20017DCC | 0x20017E24 / 0x20017E28 | 0x20017E30 / 0x20017E34 | 0x2001A650 / 0x2001A654 | 0x20017E30 / 0x20017E34 | 0x2001A650 / 0x2001A654 | 0x2001A650 / 0x2001A654 |
| 0x401/0x402/0x403 (`OTA_FW_Verify_SetStatus`) | 0x080093B8 | 0x080093B8 | 0x080092F4 | 0x08009328 | 0x080096FC | 0x08009668 | 0x08009668 | 0x080095C4 | 0x080095C4 | 0x080095C4 |
| 0x404 (`CAN_UpdateResultHandler`) | 0x0802F898 | 0x0802FDD8 | 0x0802F7AC | 0x0803173C | 0x0803178C | 0x08032368 | 0x08032958 | 0x08031D18 | 0x080321E0 | 0x08032300 |
| 0x400 (`Comm_Watchdog_CheckTimeout`) | 0x080322A0 | not listed | 0x080322F8 | 0x0803444C | 0x080344A4 | 0x08035080 | 0x08035698 | 0x08034A2C | 0x08034F1C | 0x0803503C |
| Console `get_ota_log` | 0x08038554 | 0x08038B74 | 0x080384D4 | 0x0803A714 | 0x0803A75C | 0x0803B334 | 0x0803BA04 | 0x0803ACD8 | 0x0803B280 | 0x0803B3A0 |
| Error log RAM (`g_displayErrorLog`) | 0x60013630 | - | 0x6001B670 | 0x6001E370 | 0x6001E490 | 0x6001E490 | 0x6001E490 | 0x6001E490 | 0x6001E490 | 0x6001E490 |

"not re-read" / "not listed" / "-": not covered by the checks. The external issue lists E3 148 anchors (BLE 0x0800bc18, loader 0x08016600, writer 0x08029d90, OTA writer 0x0802a098, slots 0x20017ba0, console 0x08036728); these were **not** checked in the raw bytes of 148. Their 150 counterparts are the E3 150 column above. Note: the D 150 export comments still name the 151 slot address; the live symbol is 0x20017E30.

## 5. Verification chain 0x401 / 0x402 / 0x403

`OTA_FW_Verify_SetStatus` checks the staged image in this order: 0x401, 0x402, 0x403. The 32-byte footer is the last 0x20 bytes of the received image, read with `Flash_Read_Protected` into a zeroed buffer.

1. **0x401 CRC16.** Modbus CRC16 of the staged image against the expected value; mismatch -> slot status 0x401, detail = CRC.
2. **0x402 mask.** Footer bytes 30..31 (u16, little endian) are compared for **exact equality** with the u16 at `ctx + 0x16`. That value is set in `OTA_Firmware_Download_Init` from a static 4-entry table (stride 12 B: id u8 at +0, flash destination u32 at +4, mask u16 at +8), selected by `id == ctx[0]`:

   | id | flash dest | mask |
   |---|---|---|
   | 1 | 0x200000 | 0x4444 |
   | 2 | 0x080000 | 0xFFFF |
   | 3 | 0x100000 | 0x2222 |
   | 4 | 0x180000 | 0x3333 |

   The mask is per component type, not per model (D, A, E3 share the table). The archive footers match it (Control FFFF, Micro 4444, BMS 3333); 0x2222 has no footer in the archive. There is no wildcard. If the flash read fails (mutex timeout), the buffer stays zero, the mask is 0, and 0x402 triggers before 0x403.
3. **0x403 model.** The call is `strstr(haystack = model literal, needle = 16-byte footer copy)`, i.e. `r0` = literal, `r1` = footer copy. Consequences, derived from the listing:
   - The needle is the first 16 footer bytes copied to the stack with **no terminator written**.
   - Pass = `strstr` returns non-NULL, so the check is "footer string is a contiguous substring of the model literal". An empty footer string (first byte 0) passes, as does any substring (for example `VNS` on every family).
   - A footer longer than the literal (for example `VNSD-0X`), or 16 non-NUL bytes, does not match.
   - Real footers are `VNSD`, `VNSA` and `VNSEE3`; they pass only because the literal carries the `-0` suffix and sits in the haystack position. With the more usual order `strstr(footer, literal)` these footers would not match, because `VNSD` does not contain `VNSD-0`.
   - Model literals (exact, NUL-terminated): D `VNSD-0`, A `VNSA-0`, E3 `VNSEE3-0` (not `VNSE3-0`). The literal is the only difference between the D, A and E3 functions; the code is otherwise identical in all ten builds checked.
   - The E3 BMS 110 footer is `VNSE3`, which is not a substring of `VNSEE3-0`; it would not pass if it were fed through this check. E3 BMS 112/113/115 carry `VNSEE3`.

| Build | `OTA_FW_Verify_SetStatus` | `strstr` call | Literal |
|---|---|---|---|
| D 147 | 0x080092F4 | 0x0800941E | `VNSD-0` |
| D 149 | 0x08009328 | 0x08009452 | `VNSD-0` |
| D 149.2 | 0x080096FC | 0x08009826 | `VNSD-0` |
| D 150 / 151 | 0x08009668 | 0x08009792 | `VNSD-0` |
| E3 148 (raw) | 0x0800905C..0x080091DC | 0x08009186 | `VNSEE3-0` |
| E3 150 / 151 | 0x080093B8 | 0x080094E2 | `VNSEE3-0` |
| A 150 / 1509 / 151 | 0x080095C4 | 0x080096EE | `VNSA-0` |

This describes the logic as it appears in the code. It was derived from the listing (E3 148: hand-decoded raw bytes; all others: Ghidra listing, A 150 additionally decompiled) and has **not been tested on hardware**. Whether the order is intentional (accepting abbreviated footers) is open. Footer bytes 28..29 in Micro/BMS images are not used by this function; whether Micro/BMS verify the footer themselves was not looked at.

## 6. CAN category 0x18

- **Present only in the 151 line:** D 151 `CAN_Write_Category0x18` @0x08032004, E3 151 @0x0802F484, A 151 @0x080319AC, A 1509 @0x0803188C. Absent in D 147 / 149 / 149.2 / 150, E3 150 and A 150 (call-site enumeration of `CAN_Build_Arbitration_ID`; A 150: no literal 0x14118 either). E3 147 / 148 / 149 not checked for this.
- **Frame:** `CAN_Build_Arbitration_ID(0x18, 1, 4, 0)` = 0x00014118, extended frame, DLC 8, byte 0 = value, bytes 1..7 = 0. Sent via `Register_WriteValue` -> `Modem_QueueSendMessage` -> CAN1 TX queue.
- **Triggers:** Modbus write to 46000 with 1234 / 2345 / 3456 / 4567 (also sets `g_otaTargetDevice` 1 / 4 / 3 / 2; payload byte 0 = low byte of the value) and to 47001 (0xB799; byte 0 = low byte of the value); MQTT `cd=0x42` "Get INV DEBUG info" with `debug=3` (payload 0x01) and `debug=4` (payload 0x00, `g_maintModeState` = 0). In versions without the sender the 46000 block only sets `g_otaTargetDevice` and returns status 2.
- **Receivers, all ignore the frame:**
  - E3 Micro 119 / D Micro 116 (STM32G474, FDCAN1): the frame passes the hardware filter (ExtID 0x4100, mask 0xFF00) and is queued; `EMS_Inverter_CAN_Dispatcher` has no case 0x18, nothing is stored, no reply.
  - A Micro 1211 (FlexCAN): matches mailboxes MB16-23, falls into an empty entry of the class-4 jump table (entry for 0x18 branches to the common exit). The common exit clears a link-flag bit and so refreshes the CAN link watchdog, as for every frame that passes the filter; the content is not consumed.
  - BMS D 118, E3 115, A 1105 / 115 / 110 (N32G45x, bxCAN1): accept-all filter (derived from the register writes in the code, not from the datasheet); no handler matches the id.
  - Control: no handler for an incoming 0x18.
- **Purpose unknown.** The MQTT pairing (3 -> 0x01, 4 -> 0x00) suggests an on/off toggle; this is an inference only.

## 7. Open points / not checked

1. A 148 / 1487 / 149 / 1508 and E3 144 / 1476: not analysed (selector, reader, diagnostics).
2. E3 147 / 148 / 149: only the selector shape, and for 148 the reader and writer, were read from raw bytes; no decompilation. E3 148 diagnostic anchors from the external issue not checked.
3. Real server response headers, TCP segmentation and the read boundaries of the first read; chunked transfer. Whether a split first read occurs in practice.
4. The NULL-check absence for D and A rests on decompiler output.
5. `OTA_WriteDataToRingBuffer(NULL, ...)` return value: read for E3 150 only.
6. Whether the model-check argument order is intentional; behaviour on hardware of any footer other than the shipped ones.
7. Purpose of category 0x18; consumers on other nodes (FC41D, other hardware families, later Micro/BMS); whether Control and BMS share one physical bus.
8. BMS and Micro filter semantics (accept-all, RXIMR "1 = compare") are derived from code and convention, not from reference manuals.
9. E3 BMS 110 against the 0x403 check: not tested.
10. Which component-update step succeeded or failed after a full network download is not identifiable from the download alone; the exact runtime failure of any real update failure is not established.

## 8. Method and evidence boundary

- Method: Ghidra analysis of the Control images (base 0x08004800), Thumb instruction listings with literal pools, exported decompilations, and for E3 147 / 148 / 149 hand-decoded raw image bytes (no disassembler). Cross-version equality by address-normalised diffs. Receivers read from the Micro and BMS images.
- Evidence markers: selector addresses and literals are live-checked in all Ghidra programs listed; reader, diagnostics and CAN triggers for D 147 / 149 / 149.2 and large parts of A come from exported decompilations that agreed with every live spot check.
- The synthetic pointer replays in the external issue model the selector arithmetic only. They are consistent with the code but are no evidence for runtime behaviour; a missing marker in a model is not safe NULL handling by the C caller.
- Image identity (E3, from the issue and re-matched): `148_control_VNSE3-0_app_0148_0331_093751.bin` and the 150 image match the SHA-256 values given by the reporter. CRC agreement shows consistency, not authenticity.
- No hardware test, no network capture, no vendor server observation. All statements describe what the code does given the inputs described, not what a real device or server does.
