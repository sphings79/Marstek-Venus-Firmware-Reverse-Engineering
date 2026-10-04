# 32103 read individually: no response, session dead

> 🇩🇪 Deutsche Version: [VNSD_FW_Bug_32103_Einzellesezugriff_ohne_Antwort_de.md](VNSD_FW_Bug_32103_Einzellesezugriff_ohne_Antwort_de.md)

**Device:** Marstek Venus D (VNSD-0), direct, without proxy
**Firmware:** Control/EMS **v150**
**Register:** 32103 `bat_sample_power_w2` — the upper half of the 32-bit value starting at 32102
**Date:** 20 September 2026
**Status:** reproduced on the device · cause identified in the code · effect: connection loss

---

## 1. Summary

An FC03 read access to **register 32103 with count 1** is **not answered** by the device —
neither with a value nor with an exception. The Modbus session is dead afterwards: even the
next request on the same connection runs into a timeout.

A block read that also covers the same register, on the other hand, works flawlessly.

**Cause:** the float branch of the `Read_Serializer` reads from a **non-word-aligned address**
when the second half of a `float32` entry is accessed. On Cortex-M,
`VLDR` requires word alignment; the integer branches next to it use `LDR` and tolerate
misalignment, which is why all other 32-bit registers are unremarkable.

---

## 2. Measurement

Fresh connection per row, device idle, one request:

| Request | Result | Time | Session afterwards |
|---|---|---|---|
| **32103, count 1** | **no response** | timeout | **dead** — 32104 afterwards also times out |
| 32103, count 1, repeated | no response | timeout | dead |
| 32102, count 2 (covers 32103) | `0 0` | 8.6 ms | fine |
| 32100, count 5 | `5269 0 0 0 23` | 9.2 ms | fine |
| 32102, count 1 | `0` | 4.4 ms | fine |
| 32104, count 1 | `23` | 4.5 ms | fine |

The device recovers after a few seconds and accepts new connections. It does not restart
in the process — the neighboring registers subsequently return unchanged values.

In the full scan of all 65535 registers, 32103 was the **only** timeout. All 64984 other unassigned
addresses returned a clean exception.

---

## 3. Why this register in particular

Of all 32-bit entries in the register map, 32102 is the **only one with type code `0x24`
(float32)**:

| Register | Name | Type code | 2nd half readable individually |
|---|---|---|---|
| **32102** | bat_sample_power | **0x24 float32** | **no — no response** |
| 32202 | grid_sample_power | 0x12 i16 | yes |
| 33000 / 33002 / 33004 / 33006 / 33008 / 33010 | Energy counters | 0x04 u32 | yes |
| 36000 / 36100 / 36102 | warn/error codes | 0x04 u32 | yes |
| 37013 / 37015 / 37021 | error codes, PV annual yield | 0x04 u32 | yes |

The correlation is unambiguous, but rests on exactly one case — there is only this one
float32 entry.

---

## 4. The code

`Read_Serializer` @ `0x08055414` (v150, correct base 0x08004800; originally noted as `0x08050C14` —
old Ghidra analysis with the wrong base 0x08000000, difference 0x4800). First the offset within the
entry, in registers:

```c
uVar3 = param_2 - (uint)*param_4 & 0xffff;        // 32103 - 32102 = 1
```

Then the float branch, `'$'` is `0x24`:

```c
if ((char)param_4[4] == '$') {
    local_3c = *(float *)(uVar3 * 2 + iVar4 * 4 + *(int *)(param_4 + 2));
```

With `uVar3 = 1` and `iVar4 = 0`, the float is read from **base + 2**. The base is
`0x20014EB8` (column `source_ptr_v150` of the register map), so the read is from
**`0x20014EBA`** — halfword-aligned, not word-aligned.

The integer branches calculate with the same misaligned address:

```c
else if (bVar2 == 4) {
    local_38 = *(uint *)(iVar4 * (param_4[4] & 0xf) + uVar3 * 2 + *(int *)(param_4 + 2));
}
```

The difference lies not in the code, but in the generated instruction: a `uint` access
becomes `LDR`, and unaligned `LDR` is allowed on Cortex-M as long as `UNALIGN_TRP` is not
set. The `float` access becomes `VLDR`, and **`VLDR` always requires word alignment** —
otherwise UsageFault.

This explains both observations: the absence of any response (the task does not get as far as
sending) and the restriction to exactly this one entry.

**Not evidenced** is what happens afterwards — whether a fault handler kicks in, a watchdog
restarts the task, or the connection is lost some other way. All that is observed is that the device accepts
connections again after a few seconds.

---

## 5. Delimitation — what it is not

Two obvious explanations were checked and **refuted**:

**It is not due to the read quantity.** The copy loop at the end of the `FC03_Read_Handler` counts
with a counter masked to 8 bits against a 16-bit bound:

```c
for (uVar5 = 9; uVar5 <= uVar8; uVar5 = uVar5 + 1 & 0xff) { ... }
```

For `uVar8 > 255` it would run endlessly, and `uVar8 = 9 + 2 × served registers` would be reached from 124
registers. Checked on the device, start 34000, counts 34/35/120/123/124/125/126: **all
respond cleanly**, from 126 with exception 3 (check `0x7d < uVar5`). The descriptor run aborts
earlier at the first gap, `uVar8` never becomes large enough. The 8-bit loop remains a
latent bug, but it does not explain 32103.

**It is not because reading starts in the middle of an entry.** All other
`_w2` registers are likewise second halves of a 4-byte entry and read individually
without problems (section 3).

---

## 6. Impact and workaround

A client that queries registers individually and touches 32103 in the process **loses the connection**.
Since the device allows only a single Modbus TCP session, this also affects all other
consumers until it has recovered.

Workaround, both effective:

1. **Never read 32103 individually.** Always fetch the 32-bit value starting at 32102 with count 2. That is
   the right way to read it anyway.
2. **Remove 32103 from register lists.** For a scanner or a discovery procedure, this
   is the safer variant.

For a manufacturer report: the float read path must handle the offset word-wise or load the
value via an aligned intermediate buffer, as the integer branches effectively do.

---

## 6a. Status in v151 (firmware 151, checked 2026-10-01)

**Changed — impact fixed, cause in the code persists latently.**

`Read_Serializer` itself is instruction-identical to v150 in v151 (`0x08055fe0`, 626 bytes) — the
float branch with `VLDR` on an unaligned address is still present unchanged (see
`VNSD_FW_Bug_Report_EN_Read_Serializer.md`, section "Status in v151"). The cause itself was thus
**not** repaired.

What was fixed instead is the **trigger**: the descriptor entry for 32102 was changed from `float32` (type code
`0x24`, 4 bytes) to `i16` (type code `0x12`, 2 bytes) (`VNSD_Descriptor_Table_Diff_v150_vs_v151.md`,
section 1). As a result the entry now ends at register 32103 instead of 32104, and 32103 lies outside every entry
for `FC03_Read_Handler` (v150 `0x0802386c` → v151 `0x08023a00`, 382 bytes):

```c
if (start_offset <
    (uint)g_modbusRegDescTable[reply_write_offset].reg_addr +
    (uint)((int)(short)(g_modbusRegDescTable[reply_write_offset].type_code2 & 0xf) *
          (int)(short)(ushort)g_modbusRegDescTable[reply_write_offset].count) / 2) {
  if (start_offset < g_modbusRegDescTable[reply_write_offset].reg_addr) {
    return 2;                    // Exception 02, cleanly answered
  }
  ...
```

A single access to 32103 therefore no longer passes through the float branch of `Read_Serializer` in v151 at all —
instead of no response/dead session, the device now returns a regular exception 02. For the full scan behavior
(v150: only timeout among 65535 addresses), something concretely changes for 32103; the underlying
`VLDR` alignment gap in the serializer remains, however, and would strike again with any future float32 entry
with an odd byte offset. Not checked: whether such an entry exists in v151.

## 7. Cross-references

- `VNSD_FW_Bug_Report_EN_Read_Serializer.md` — sign error in the same serializer (32101)
- `VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20.md` — wrong MBAP length field on exceptions
- `VNSD_FW_Bug_44002_44003_Readback_After_Restart.md` — readback value after restart
- `VNSD_Network_Dropouts_During_Register_Scan_2026-09-20.md` — connection drops independent of this
- Measurement data: `Scan_Logs/register_scan_full_20260920_1154.csv` (local, raw data), row 32103
