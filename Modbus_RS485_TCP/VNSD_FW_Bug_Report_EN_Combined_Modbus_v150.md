# Three Modbus defects in the Venus Control firmware v150

**Found and measured on:** Marstek Venus D (VNSD-0), Control / EMS **v150**
(`VNSD-0_app_0150_0805_115146.bin`, build Aug 5 2026)
**Also present in:** Venus E3 v150 — confirmed byte-identical for defect 1, see below
**Interfaces:** Modbus TCP (port 502) and Modbus RTU over RS485
**Date:** 20 September 2026

**Note on addresses:** all Venus D flash addresses below use the correct Ghidra image base
0x08004800. An earlier analysis pass used the wrong base 0x08000000 (a constant 0x4800 too low);
each corrected address below carries a note with the original one. The Venus E3 address in defect
1 is from a separate image and analysis and was not re-checked here.

All three were found by static analysis of the firmware image and then confirmed on a running
six-pack Venus D. Each section gives the code location, what we measured on the device, and a
suggested correction. We are happy to supply the raw captures behind any of it.

A note on why these arrive together: we reported the second defect on its own earlier and have
had no reply to date, so rather than send further single reports we have collected what we
have. None of the three is a safety issue. The first is the one we would prioritise: it
affects every client, it is the cheapest to fix, and it makes the device look unresponsive
when it is in fact answering correctly.

**Please pass this to whoever maintains the Modbus stack.** It is not a support case: none of
the three can be resolved by configuration, a reset or a replacement unit, and each one points
at a specific location in the v150 image with a suggested correction. We are not asking for
support, and we would be glad to test a candidate build on our unit.

A short confirmation that it reached engineering is all we need — we do not expect individual
updates or a timeline.

| | Defect | Effect | Suggested fix |
|---|---|---|---|
| 1 | Exception replies declare one byte too many | Clients time out instead of seeing the rejection; each one costs two failed reads | one constant |
| 2 | FC03 scales signed values with unsigned arithmetic | Register 32101 unusable while discharging | signed local variable |
| 3 | 44002 / 44003 are read from an uninitialised copy | Both read 0 after every restart although the limit is active | read the config block, or initialise the copy |

---

# 1. Exception replies declare one byte more than they contain

**Severity:** every client is affected; a rejected register costs two failed reads and blocks
a shared connection for the duration of its timeout.

## Symptom

A Modbus exception PDU is always exactly two bytes: the function code with bit 7 set, followed
by the exception code. The MBAP length field must therefore read 3 (unit id + function code +
exception code). The firmware writes **4** while sending a nine byte frame.

Captured directly from the device with no proxy in the path:

```
18 58 00 00 00 04 01 83 03    read of register 39000 -> exception 3
1F C8 00 00 00 04 01 83 02    read of register 45000 -> exception 2
1B 0C 00 00 00 04 01 83 02    read of register 31500 -> exception 2
18 B8 00 00 00 04 01 83 02    read of 35000, quantity 12 -> exception 2
```

A conforming client reads the seven byte header and then waits for `length - 1 = 3` PDU bytes.
Only two arrive, so it blocks on a byte that is never sent and eventually times out. The
device answered correctly and in about 100 ms, but the client cannot tell that apart from a
device that stopped responding.

## Where it is

`Modbus_Dispatcher` at `0x08023204` (was `0x0801ea04`), exception branch from `0x080232e8`
(was `0x0801eae8`):

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

The same firmware computes the success path correctly in `FC03_Read_Handler` at `0x0802386c`
(was `0x0801f06c`) as `quantity * 2 + 3`, so a one register read declares 5 and sends eleven bytes. This is an
off-by-one against the firmware's own convention, not a different framing scheme.

Both error paths of `FC03_Read_Handler` reach this branch: `return 2` for a register in a gap
between descriptor entries, `return 3` for a register past the last entry or a quantity of 0
or above 125. Any function code other than 03, 06 and 10 also lands here with code 1.

**This is not specific to the Venus D.** In the Venus E3 image
(`150_control_VNSE3-0_app_0150_0804_151249.bin`) `Modbus_Dispatcher` sits at `0x0801d278`,
has the same size of 392 bytes, and the bytes at the corresponding offset are identical
including the `movs r0, #4`. We correlated the Venus A image against the Venus D with Ghidra
Version Tracking: all 955 functions that are disassembled there match byte for byte, so we
would expect the same there, although the Modbus path in that image is not yet disassembled
and we have not confirmed it directly.

## Measured impact on a client

With a current Modbus library on a persistent connection, one read of an unimplemented
register costs **two** failed reads:

```
1) implemented register   -> 150    (0.01 s)
2) unimplemented, 1 try   -> None   (3.05 s)
   library log: "Received unexpected response with Transaction ID: 2.
                 Discarding bytes: 00 02 00 00 00 04 01 83 03 00"
3) implemented, same conn -> None   (3.05 s)   lost
4) implemented, again     -> 150    (0.05 s)   recovered
```

The ten discarded bytes are the nine of the exception plus the first byte of the following
response: the parser takes the byte the header promised out of the next frame, then discards
both and resynchronises.

The effect is worse behind a Modbus proxy, which many installations use because the device
accepts only one TCP session. A proxy is itself a parser, so it stalls on the same missing
byte and gives up after its own timeout, and no client behind it ever learns that the device
answered. In our case a proxy configured with a ten second timeout and one retry blocked its
device lock for up to twenty seconds per rejected register, for every client at once.

## Suggested fix

```c
buf[5] = 3;   /* unit id + function code + exception code */
```

One constant. The rest of the frame, including the `| 0x80` on the function code, is correct.

## Status in v151 (checked 2026-10-01)

**Unchanged.** `Modbus_Dispatcher` is at `0x08023398` in v151 (392 bytes, same size). The exception
branch is byte-identical to the v150 defect:

```
08023488  movs r0,#0x4      ; MBAP length low byte still 4, not 3
0802349a  movs r2,#0x9      ; still nine bytes sent
```

The decompiled source shows the same `g_modbusTcpTxFrame_lengthLo = '\x04';` at that spot, still
followed by `CH395SendData()`. The off-by-one is present unchanged in v151.

---

# 2. FC03 scales signed values with unsigned arithmetic

**Severity:** the affected register cannot be used as a measurement whenever the battery
discharges.

**Already reported.** We sent this one on its own earlier and have had no reply to date. It is
repeated here so that all three are in one place. The finding and the measurements are the
same as before; the only addition is the address of the function in the image, which we had
not included the first time.

## Symptom

Register **32101** is the BMS battery current, `int16`, unit 0.1 A. Reading it while the
battery discharges returns values that correspond to no current at all:

| Battery state | Actual current | Expected | Returned |
|---|---|---|---|
| Idle, about −0.3 A | −0.3 A | `0` | **39321** (`0x9999`) |
| Discharging 600 W, −12.2 A | −12.2 A | `-12` | **39309** (`0x998D`) |
| Charging 600 W, +10.3 A | +10.3 A | `10` | `10` — correct |

Charging is unaffected, so the defect is asymmetric.

The actual current was established independently: at the same moment register **34301**
(pack 4 current, same quantity, no scale code applied) returned `-3` and `-122`, and the
device's own Bluetooth interface reported battery power of −16 W and −651 W, which matches
those currents at the measured pack voltage of 53.4 V.

## Where it is

`Read_Serializer` at `0x08055414` (was `0x08050c14`). The value is placed in an **unsigned** local and the
descriptor's scale code is applied to it:

```c
uint value;

value = (uint) *(short *) source;   /* sign extends: -122 becomes 0xFFFFFF86 */

switch (scale_code) {
    case 1: value = value * 10;   break;   /* correct */
    case 2: value = value * 100;  break;   /* correct */
    case 3: value = value / 10;   break;   /* wrong for negative values */
    case 4: value = value / 100;  break;   /* wrong for negative values */
    case 5: value = -value;       break;   /* correct */
}
```

The sign extension is right, and multiplication and negation continue to work in two's
complement. Division does not: the compiler divides 4294967174 rather than −122. The low 16
bits of that result are copied into the response, which reproduces both observed values
exactly:

```python
def serialize(sram_int16):
    v = sram_int16 & 0xFFFFFFFF   # (uint)(short)x
    v = v // 10                   # unsigned division
    return v & 0xFFFF

serialize(-3)    ->  39321
serialize(-122)  ->  39309
```

In v150 exactly one register combines a signed type with a divide scale, so only 32101 is
affected today. The defect itself sits in the shared serialiser that every FC03 read passes
through, so any future register with that combination inherits it.

## Suggested fix

Scale on a signed variable, or branch on the type's signedness before dividing:

```c
int32_t value = (int32_t) *(int16_t *) source;

switch (scale_code) {
    case 3: value = value / 10;  break;
    case 4: value = value / 100; break;
    ...
}
```

Signed division rounds toward zero in C, which matches the intent of the scale codes.

## Impact

Register 32101 cannot be used as a current measurement. The value can be approximately
inverted as `current ≈ (raw − 39321) A`, but only at 1 A resolution, because the division
happens before the truncation to 16 bits, and the behaviour is asymmetric between charging and
discharging. Any client that reads 32101 and interprets it as documented will report a large
positive current while the battery is in fact discharging.

The defect is latent for every other register: it lives in the shared read serialiser, not in
the descriptor entry for 32101.

We have not verified whether Venus A and Venus E v3, which share this firmware base, expose
additional registers with the same type and scale combination.

## Status in v151 (checked 2026-10-01)

**Unchanged.** `Read_Serializer` is at `0x08055fe0` in v151 (626 bytes, same size) and
instruction-identical to v150 apart from decompiler variable naming. The scale-code switch still
divides the unsigned local:

```c
case 3: value = value / 10;   /* still wrong for negative values */
case 4: value = value / 100;  /* still wrong for negative values */
```

Register 32101 remains the only register hit by this combination (int16, divide-by-10 scale) in
v151 as well.

---

# 3. Registers 44002 and 44003 are read from a copy that is never initialised

**Severity:** readback only. The limit itself keeps working, but every client that reads or
displays it is misled after a restart.

## Symptom

After a restart of the Control, FC03 on **44002** (`max_charge_power`) and **44003**
(`max_discharge_power`) returns **0** and keeps returning 0, although the configured limit is
stored and remains in force.

Captured directly at the device, one sample per second:

| Time | 44002 | 44003 | battery power | SoC | ems_ver |
|---|---|---|---|---|---|
| 11:43:18 | 2500 | 2500 | −12 W | 23 | 150 |
| 11:43:29 | device unreachable (restart) | | | | |
| 11:43:59 | **0** | **0** | −12 W | 0 | 150 |
| 11:44:52 | **0** | **0** | −13 W | 24 | 150 |

The SoC recovers and `ems_ver` reads 150 again, so the device is fully up. Only these two
registers stay at 0, and a single write restores them immediately.

We verified that the limit is unaffected. With both registers reading 0 we commanded a charge
(42000 = `0x55AA`, 42020 = 400 W, 42010 = 1) and measured the battery power over 30 seconds,
then wrote 2500 back and repeated the same command:

| | measured charge power |
|---|---|
| 44002 reading 0 | 342–349 W |
| 44002 reading 2500 | 347–348 W |

Identical. The zero is a readback artefact, not an active limit.

## Cause

There are two locations, and the read path uses the wrong one.

**The effective configuration** lives in the block at `0x2000012B`:

| Offset | SRAM | Field | EEPROM |
|---|---|---|---|
| +0 | 0x2000012B | charge_cutoff_soc | 0x200 |
| +1 | 0x2000012C | discharge_cutoff_soc | 0x201 |
| +2 | 0x2000012D | **max_charge_power** | **0x202** |
| +4 | 0x2000012F | **max_discharge_power** | **0x204** |
| +6 | 0x20000131 | grid_standard | 0x206 |

`Board_SystemStartup` at `0x0801e194` (was `FUN_08019994` at `0x08019994`) is the top level init
sequence and calls the configuration loader `Config_LoadOrInit_DeviceSettings` at `0x0802dcbc`
(was `FUN_080294bc` at `0x080294bc`), which reads seven bytes from EEPROM `0x200` into that
block and then runs the validator `Config_ClampDefaults_ParamBlock` at `0x0800935c` (was
`FUN_08004b5c`) unconditionally. The validator raises an implausible value to a default
(max_charge_power to 2500, max_discharge_power to 800), so after startup this block always
holds a usable value. The power limiter reads it, which is why the device keeps limiting
correctly.

**The Modbus read path uses something else.** In `Write_Handler` at `0x08056514` (was
`0x08051d14`) the read branch for these registers returns `g_maxChargePowerRegCache` /
`g_maxDischargePowerRegCache` (previously seen as the unresolved pointer `DAT_080529bc`), which
resolves to **`0x20000156`** / **`0x20000158`**, a separate shadow. Cross references to
`0x20000156` give exactly four hits and none of them is a load:

| From | Kind |
|---|---|
| `Write_Handler` 0x0805700c (was 0x0805280c) | WRITE — a Modbus write to 44002/44003 |
| `Write_Handler` 0x0805701a (was 0x0805281a) | READ — the Modbus read path |
| `Protocol_AA_SetExtendedParams` 0x0803449c (was 0x0802fc9c) | WRITE — copied out of a CAN frame |

No `EEPROM_Read`, no startup initialisation. After a restart the shadow holds its BSS zeros.

A write to 44002 updates three places — the Micro inverter over CAN, the config block plus
EEPROM `0x202` through `Config_Write_U16_0x202`, and the shadow — which is why writing the
register makes it read correctly again.

## Suggested fix

Either serve the read path from the configuration block:

```c
/* in Write_Handler, read branch for 44002 / 44003 */
value = *(uint16_t *)(0x2000012D);   /* and 0x2000012F for 44003 */
```

or initialise the shadow from that block once during startup, next to the existing
`Config_ClampDefaults_ParamBlock` call. Either removes the discrepancy without touching the
write path.

## Status in v151 (checked 2026-10-01)

**Unchanged.** All functions above exist in v151 at the same size, moved only by the heap-growth
offset (0x11800 → 0x14000):

| Function | v150 | v151 |
|---|---|---|
| `Board_SystemStartup` | 0x0801e194 | 0x0801e324 |
| `Config_LoadOrInit_DeviceSettings` | 0x0802dcbc | 0x0802e17c |
| `Config_ClampDefaults_ParamBlock` | 0x0800935c | 0x0800935c (unchanged) |
| `Write_Handler` | 0x08056514 | 0x080570e0 |
| `Protocol_AA_SetExtendedParams` | 0x08034494 | 0x08034aa8 |

The v151 read branch for 44002 (`Write_Handler`, register offset `uVar3 == 900`) still returns
only the shadow:

```c
if (uVar3 == 900) {
  if (is_write == 0) {
    bcd_reply_value = (uint)_g_maxChargePowerRegCache;   /* still 0x20000156 */
    Byte_Swap_Copy(rsp,&bcd_reply_value,2);
    return 0;
  }
  ...
```

44003 is identical with `g_maxDischargePowerRegCache` (still `0x20000158`). v151's
`Config_LoadOrInit_DeviceSettings` gained a new boot-time step (loading the cloud server type,
unrelated to this defect) but still does not initialise either shadow. Both registers read 0
after a restart in v151 exactly as in v150.

---

# Reproducing

1. Connect over Modbus TCP on port 502. Note that the device accepts only one session at a
   time.
2. **Defect 1:** read an unimplemented register, for example 39000 or 45000, function code 03,
   quantity 1. A compliant client reports a timeout. Capture the raw bytes on the socket:
   nine bytes ending in `01 83 02`, with the length field reading `00 04`. A proxy or gateway
   in the path hides this — capture with a direct connection.
3. **Defect 2:** put the battery into a discharge state of a few hundred watts, then read
   32101 and 34301 in the same pass. 34301 returns the pack current correctly signed, 32101
   a large positive number that does not correspond to it. Repeat while charging: both agree.
4. **Defect 3:** note the values of 44002 and 44003, restart the device, and read them again.
   Both return 0. Command a charge and confirm that the configured limit still applies.

We can provide the socket captures, the register dumps behind the measurements, and the
before-and-after traces of the restart on request.

Regards,
Reporter
