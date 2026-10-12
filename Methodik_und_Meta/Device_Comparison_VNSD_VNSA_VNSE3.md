# Device Comparison Venus D / Venus A / Venus E3

> 🇩🇪 Deutsche Version: [Geraetevergleich_VNSD_VNSA_VNSE3_de.md](Geraetevergleich_VNSD_VNSA_VNSE3_de.md)

**Status:** 20 September 2026
**Purpose:** Record what is identical between the models, what differs and what is unchecked
— so that a finding on the D is not silently transferred to the A or E3, and conversely does
not have to be investigated twice.

---

## 1. Codebase — One for All Models

Ghidra Version Tracking, source `VNSD-0_app_0150_0805_115146.bin` (D v150), target
`150_control_VNSA-0_app_0150_0811_114724.bin` (A v150):

| | |
|---|---|
| Functions in the D | 1884 |
| Functions in the A (before the run) | 955 |
| of which matched | **955 — all** |
| in the A without counterpart | **0** |
| in the D without counterpart | 929 |
| Matched via | *Exact Function Bytes Match* |
| Similarity | **uniformly 1.0**, `sizeDelta 0` |

**Every function that was disassembled in the A image is byte-identical to its
D counterpart.** Not a single one shows a logic deviation.

The 620 entries that the diff lists as "changed" are exclusively callee artifacts: the
called functions had not yet been captured in the A image, so the call list appeared
shorter. Library functions are affected throughout — mbedTLS, FreeRTOS, llhttp, MQTT,
SysTick.

The 929 in the D without counterpart were **not missing functions**, but simply code in the A that
had not been disassembled yet. Section 1d shows where they are located.

### Consequence

A finding in the **Control code** of the D also applies to the A until disproven. This applies
in particular to the three documented firmware bugs — exception frame, read serializer,
readback value 44002/44003. For the exception frame it has meanwhile been evidenced, see section 4.

> **Important qualification, 21.09.2026.** This sentence applies **only to the Control**. For the
> inverter firmware ("Micro", `*_inv_app_*`) it is demonstrably false for the A — a foreign
> codebase is installed there. For the E3 it holds again. See section 1e.
> This is more than a subtlety: the error words 36000/36100/36102 are generated **not by the
> Control**, but by the inverter. The integration therefore displayed the
> D error texts on the Venus A for a while, where they mean nothing.

### 1b. Venus E3 versus Venus D

Source D v150, target `150_control_VNSE3-0_app_0150_0804_151249.bin`:

| | |
|---|---|
| Functions in the D | 1884 |
| Functions in the E3 | 1617 |
| matched | 1578 (447 identical, 1131 "changed") |
| in the E3 without counterpart | 39 |
| in the D without counterpart | 306 |

Unlike with the A, there are unmatched functions in both directions here. **They are not
evidence of missing functionality**, however — see section 5. The list of the 306 "only in D"
contains, among others, `CAN_SendMessage`, `CH395_SPI_ReadByte` and the config validator,
functions that the E3 undoubtedly has. Ghidra VT has no body similarity
correlator; if a function deviates slightly, it falls through.

### 1c. Markup Transfer from the D

Because the matching is byte-exact, the naming was transferred from the D image:

| Target | transferred | skipped | named functions afterwards |
|---|---|---|---|
| A v150 | 945 | 135 | 954 of 955 |
| E3 v150 | 1556 | 303 | — |

This names existing functions, **it does not create missing ones**. The Modbus path therefore initially
remained invisible in the A. Only the run in section 1d resolved it.

### 1d. The A Image Is the Same Code, Just Shifted

The decisive finding. Of the 954 functions that carry the same name in both
images after the markup transfer, only **56 are at the same address**. All the others are shifted by
one of exactly **21 different offsets**:

| Offset A − D | Functions |
|---|---|
| −1628 (−0x65C) | 603 |
| −1616 | 74 |
| ±0 | 56 |
| −1612 | 50 |
| −352 | 43 |
| −1624 | 31 |
| −1620 | 29 |
| −164 | 27 |
| remaining 13 values | together ~40 |

This makes it possible to search for each D function in the A in a targeted way, instead of blindly scanning for prologues.
Procedure:

1. Candidate addresses = D address plus each of the 21 offsets, **additionally** every occurrence
   of the first 16 bytes of the D function in the entire A image.
2. Byte comparison, **relocation-aware**: a 4-byte word that differs in the A from the one in the D by one of the 21
   offsets counts as equal (literal pool pointer). The same applies to a
   Thumb-2 `bl` whose decoded branch target differs by one of these offsets.
3. Accept only at ≥ 85 % match **and** ≥ 25 percentage points gap to the
   second-best candidate.

Without step 2, many functions are at 88–94 % and fall through; with it they jump to
100 %. Of 926 functions searched, **774 were found again at exactly 100 %**.

Result of the run:

| | |
|---|---|
| newly created and named | 663 |
| renamed (previously carried `FUN_…`) | 128 |
| name conflicts (not touched) | 4 |
| failed | 0 |
| ambiguous, skipped | 71 |
| not located | 60 |

**Status of the A image afterwards: 1831 functions instead of 955, instruction coverage 73.4 % instead of
35.5 %.** The D stands at 1884 functions.

The 4 conflicts are addresses where an already named A function sits and a
*different* D function matches byte-identically — both are tiny wrappers that cannot be
distinguished:

| Address | Name in the A | also matches |
|---|---|---|
| 08006768 | `Delay_Adaptive` | `EEPROM_Mutex_Wait` |
| 080342da | `mbedTLS_AES_Setkey_Dec_Wrapper` | `mbedTLS_ECP_RNG_Wrapper` |
| 08046b74 | `mbedTLS_Cipher_Init` | `mbedTLS_SHA256_Init` |
| 080558fc | `CH395_Debug_SendCmdByte` | `SPI_WriteByte` |

Not an error, but a limit of the method: with identical bytes only the call context
decides.

---

### 1e. The Inverter Firmware Splits into Two Families

Addendum 21.09.2026. The comparison above concerns exclusively the Control. The
inverter images behave differently — block comparison over all 24-byte windows
(step size 4):

| | D-116 | E3-119 | A-119 |
|---|---|---|---|
| File | `vd_inv_app_0116` | `inv_app_0119` | `VA_inv_app_0119` |
| Size | 115,712 B | 119,808 B | 146,432 B |
| Reset handler | `0x080042AD` | **`0x080042AD`** | `0x08006435` |
| Initial SP | `0x20009A70` | `0x20009AC0` | `0x20002400` |
| Error block in SRAM | `0x200019F4` | `0x20001A28` | `0x20101650` |
| Base address loaded via | literal pool | literal pool | `movw`/`movt` |
| shared blocks with D-116 | — | **43.2 %** | **0.4 %** |

**D and E3 are one family, the A is its own.** 0.4 % is noise — of 73 shared blocks, two are trivial fill
patterns, the rest random short instruction sequences. For comparison, two versions of the same family
share around 60 %. The code generation also differs
systematically: D and E3 fetch globals from literal pools, the A exclusively via
`movw`/`movt` pair.

The error bits follow this separation exactly. D and E3 have the same err2 bits with the same
event codes (0, 1, 3, 4 via event 0x36, bit 7 via 0x13, bit 8 via 0x31, bit 11 via 0x35,
plus bit 16); the A has bits 5, 6, 7, 8, 9, 11 and 13 with different events. The
hardware protection latch bits 0/1/3/4 of the D do not exist in the A image at all.

The only known difference between D and E3: the E3 sets and clears `warn` bit 0, the D
never writes the warning word. Details in `VNS_Micro_Inverter/VNSD_Error_Codes_Micro_FW_Analysis.md`,
section 10, and for the A in `VNS_Micro_Inverter/VNSA_Error_Codes_Micro_FW_Analysis.md`.

> That E3 and A both carry a firmware "119" is coincidence: the product lines count separately,
> the checksums differ, and according to the changelog the A is long since at 119.3 and 121.1.

### 1f. The Modbus Descriptor Tables Are Identical Across the Models

Also 21.09.2026, unpacked with `Scripts/marstek_descriptor_unpack.py` (local, not in the repo) from the three
Control v150 images:

| Comparison | Entries | Registers only in one | Deviations in type / scale / element size / count |
|---|---|---|---|
| D ↔ E3 | 246 / 246 | none | **0** |
| D ↔ A | 246 / 246 | none | **0** |

The only differences are SRAM source pointers: on the A **identical** to the D, on the E3 shifted by −8
(212 registers), 29 unchanged, four individual cases.

**Consequence:** Register numbers, types and scales do not have to be determined anew per model. What distinguishes a
model is exclusively whether there is hardware behind a register — no PV
means the MPPT registers read 0; a single storage unit instead of a stack means the registers
for pack 2…7 read 0. The complete D register map is thus the reliable template for the
other models, which is directly relevant for `e_v3.yaml` (maps only 82 of 410 served registers).

## 2. Status of the Disassembly

| Image | Functions | Instruction coverage | Modbus code captured |
|---|---|---|---|
| D v150 | 1884 | — | yes |
| A v150 | **1831** (previously 955) | **73.4 %** (previously 35.5 %) | **yes** — see below |
| A v1509 | 935 | unchecked | unchecked |
| E3 v150 | 1617 | — | yes, named via the markup transfer |

### The Modbus Path in the A

Found via the offset from section 1d, disassembled and named:

| Function | D | A | Offset | Byte equality |
|---|---|---|---|---|
| `Modbus_Dispatcher` | 0801ea04 | **0801e3ac** | −1624 | 94.1 % |
| `FC03_Read_Handler` | 0801f06c | **0801ea14** | −1624 | 98.4 % |
| `Read_Serializer` | 08050c14 | **080505b8** | −1628 | 98.6 % |
| `Write_Handler` | 08051d14 | **080516b8** | −1628 | 95.7 % |

The missing percentages are throughout literal pool addresses and branch targets that shift along with the
layout.

### Correction of Two Earlier Wrong Conclusions

**First**, regarding the register numbers. An earlier version stated that a search for `0xabe2`
(44002), `0xa424` (42020) and `0xa41b` (42011) yields zero hits in the A, and from this it was
concluded that the Modbus path was not captured there. The conclusion did not hold: **the same
scan also yields zero hits in the fully disassembled D image.** The
register numbers appear as an immediate in none of the images — they are computed
or resolved via the descriptor table. The search was unsuitable for the question from the
outset.

**Second**, regarding the power limits, and that was the more serious error. It stated here
that the constant 1500 does not occur in either of the two images, searched as `0x5dd`.
**1500 is `0x5DC`; `0x5DD` is 1501.** The same off-by-one was in `0x9c5` for 2500 — the correct
value is `0x9C4`. With the correct values the exact opposite of what was
claimed at the time results:

| Constant | D | A |
|---|---|---|
| 1500 = `0x5DC` | 8× | **22×** |
| 2500 = `0x9C4` | **22×** | 8× |
| 2501 = `0x9C5` (bound for ≤2500) | 2× | 2× |
| 1200 = `0x4B0` | 29× | 29× |
| 2200 = `0x898` | 5× | 5× |

The two values are simply **swapped** between the models. Section 3a says where.

The lesson is in section 5: a negative search is only as good as the conversion of the
constant being searched for.

---

## 3. Known Differences

### 3a. Maximum Charge and Discharge Power — Resolved

| Model | Limit |
|---|---|
| Venus A | 1500 W |
| Venus D | 2500 W |

**The limit is in the code, at four places.** The power class hypothesis from the
earlier version is not needed for this:

| Function | D | A |
|---|---|---|
| `FUN_08004b5c` — config validator, resets implausible values to default | `0x9C4` = 2500 | **`0x5DC` = 1500** |
| `Config_Write_U16_0x202` — persists 44002 | clamp 2500 | **clamp 1500** |
| `Config_Write_U16` — persists 44003 | clamp 2500 | **clamp 1500** |
| `Write_Handler`, branch `uVar8 == 900` → **44002** | `param_4 < 0x9C5` (≤2500) | `param_4 < 0x9C5` (≤2500) |
| `Write_Handler`, branch `uVar8 == 0x385` → **44003** | `param_4 < 0x9C5` (≤2500) | **`param_4 < 0x5DD` (≤1500)** |

#### The Asymmetry

The two twin registers are **checked differently** on the A, and that is the
practically most important consequence:

- **44002 `max_charge_power`:** the `Write_Handler` still accepts up to 2500 and returns 0.
  Only `Config_Write_U16_0x202` clamps to 1500 when saving. A write access
  with 2000 is therefore **accepted and silently becomes 1500**.
- **44003 `max_discharge_power`:** the `Write_Handler` rejects anything above 1500 directly and
  returns `return 3`, i.e. exception 3 *illegal data value*. The clamp is never reached.

Verifiable on the device, and the predictions are opposed enough to be meaningful: writing 2000
to 44002 → no exception, effect 1500. The same to 44003 → exception 3.

**Consequence for the integration.** `a.yaml` should carry a maximum of 1500 for 44002 and 44003,
not 2500 — and when writing 44002 a readback value below the
written value is not an error, but the clamp. This directly affects the open stable release
3.0.0 (section 3c).

#### Where the Power Classes Do Appear After All

`BLE_Cmd_Dispatch` assigns a class number via the BLE path, and there the
models also differ:

| | D | A |
|---|---|---|
| Branch | `0x898` (2200) → class **3** | `0x4B0` (1200) → class **4** |
| Branch | `0x9C4` (2500) → class **0** | `0x5DC` (1500) → class **5** |
| Range check | `299 < x < 0x9C5` | `299 < x < 0x5DD` |

This is the class byte from `Modbus_RS485_TCP/VNSD_Descriptor_Open_Registers_Ghidra_Finding.md`, section 11. It
therefore exists, but is **written** here, not read — the effective limit comes from
the constants above, not from the byte.

### 3b. PV and MPPT — Present in the E3

The expectation that the E3 lacks the PV part does **not** hold. The image contains the
complete set:

```
PV1_Vol(0.1V)=%d|Cur(0.1A)=%d|Pow(0.1W)=%d   ... up to PV4
Mppt_State / Mppt_Error / Mppt_Temp / Mppt_Warning
mppt_power(W) / mppt_mode / mppt_ver(new) / mppt_ver(old)
PV_Day_Cap_10Wh / PV_Mon_Cap_10Wh / PV_Year_Cap_10Wh
update_mppt / get_mppt_data / "The MPPT will be updated soon"
&mod2=mppt&err2=%x   (cloud report)
```

Not just as strings: the markup transfer named, among others, `MPPT_Debug_Print`,
`AT_Copy_MPPT_Channel_Data`, `MQTT_Publish_VNS_MPPT_Data` and `Cloud_Report_FillPvData`
in the E3 — these are real functions. Even the OTA path for the MPPT firmware is included.

This fits the overall picture: one codebase for all models, differences in hardware and
configuration. That a device has no PV inputs does not mean that its firmware
does not know about them.

### 3c. Register Maps of the Integration

`a.yaml`, `d.yaml`, `e_v3.yaml` and `e_v12.yaml` differ. The Venus A part is regarded as
unconfirmed and blocks the stable release 3.0.0 of the integration.

This comparison yields three reliable points for it, all derived statically from the code
and none checked on the device:

1. **44002 and 44003 belong in `a.yaml` with a max of 1500**, not 2500 (section 3a).
2. **44002 accepts values up to 2500 and silently clamps to 1500.** A readback value below the
   written value is expected behavior here, not an error.
3. **44003 rejects above 1500 with exception 3.** An integration that offers 2500 produces
   an error message there instead of a clamp.

Whether the descriptor table in the A contains further deviating registers is still open — but now
answerable from the code, because the table sits at the address shifted by the matching offset.

### 3d. RS485 Connection — Different UART, No Direction Control

The second tangible difference, found via the remainder list from section 1d.

| | Venus D | Venus A |
|---|---|---|
| UART | `0x40004400` = **USART2** | `0x40013800` = **USART1** |
| Direction line | GPIO `0x40011000` bit 7 (**PC7**), set via BSRR `+0x10`, cleared via BRR `+0x14` | **none** |
| Guard time before sending | `0xDAC` = 3500 loop iterations | **none** |
| Guard time before receiving | `0xBB8` = 3000 loop iterations | **none** |
| Helper functions | `RS485_Set_TX_Mode` @ 0x080072C8, `RS485_Set_RX_Mode` @ 0x080072E4 | **do not exist** |

The A function at `0x08007148` (named `RS485_UART_Send_NoDirectionControl` here) is
instruction for instruction the D `RS485_UART_Send` @ `0x08007268`, just without the two
calls at the beginning and at the end. The same applies to `0x080071A0` compared to
`RS485_UART_SendByte` @ `0x08007300`.

That the functions really are missing and not just located elsewhere has been checked: the
20-byte bodies of both functions do not occur a single time in the entire A image.

The caller list is identical on both models — `RS485_FC03_ReadWrite_Handler`,
`RS485_FC06_WriteSingle_Handler`, `RS485_FC10_WriteMultiple_Handler`,
`RS485_RTU_Frame_Dispatcher`, `RS485_SendIfValid`, `Protocol_AA_RS485Forward`. So the A has
the complete Modbus RTU stack, but does not drive a DE/RE line. Obvious explanation:
a transceiver with automatic direction switching or a different physical
interface. Not checked on the device.

### 3e. Further Real Code Differences

`Grid_Export_Power_Limiter` (D `0x0801F200`, A `0x0801EBA8`), three deviations:

| | D | A |
|---|---|---|
| Upper limit | `4000` | `0x960` = **2400** |
| State check | `if (*x == 5)` | `if ((*x == 5) \|\| (*x == 1))` |
| Inverter command | `Inverter_RS485_Command_Send(1,**1**,…)` | `Inverter_RS485_Command_Send(1,**3**,…)` |

Not investigated further. In contrast, `Modbus_Dispatcher`, `FC03_Read_Handler`
and `Read_Serializer` are unchanged — there is not a single deviation there apart from shifted addresses.

In D 150, `Inverter_RS485_Command_Send` has been named `Inverter_CAN_PowerCmd_Send` since 2026-09-24,
because the function sends via CAN (see `Control_FW/VNSD_Control_Charge_Power_PV_Curtailment_v150.md`
§3.2). In the A programs the old name has remained.

### 3f. Charge Limit, Pack Rotation and PV Curtailment (status 2026-09-24)

In detail for the D: `BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md` and
`Control_FW/VNSD_Control_Charge_Power_PV_Curtailment_v150.md`. The information here comes from the
analysis of 24.09. and has not been checked as deeply as the D side.

**Conversion limit → power** (`P_bms = voltage × charge current limit`) is built identically
on all three models:

| Model | Function | Aggregate struct |
|---|---|---|
| D 150 | `Power_Limit_Clamp` `0x08013234` | `0x20014F8E` |
| A 150 | `0x080130d4` | — |
| A 1509 | `0x080130ec` | `0x200177AE` |
| E3 150 | `0x080126cc` | `0x20014F86` |

**PV upper limit on feed-in** (`Grid_Export_Power_Limiter`):

| Version | Upper limit |
|---|---|
| A 150 | `min(P_bms, 2500)`, like D 149.2/150; overall cap 2400 instead of 4000 |
| A 1509 | `ac + P_bms`, like D 147 (byte comparison at `0x0801ecb4`) |

**BMS — pack hand-off threshold on discharge (corrected 2026-10-11):** The earlier statement that the
discharge threshold with DoD lower limit and the modes 2/3 below 11 % exist "only in D 118" was wrong.
Per BMS version (multi-pack models only; image names in the firmware archive):

| Model | BMS FW | `PackSelect_DischargeStepThreshold` | Hand-off to the fullest pack at (cutoff 12 %) | `ChargeStepTarget`, modes 1/2/3 | Evidence |
|---|---|---|---|---|---|
| D | 116 (no image in the archive) | unknown | packs stopped at **10.9 %** (observed); matches the 117.7 formula | unknown | measured by a third party on a 4-pack unit, app overall SOC 12 %; no code available |
| D | 117.7 (`1177`) | fixed step table `max(10, ⌊(soc% − 10) / 10⌋ · 10)`, **ignores** the configured discharge limit | about **10.9 %**, independent of the cutoff | present | decompilation `0x08009354` |
| D | 118 | `L = 100 − DoD`, threshold `max(soc% − 10, L)`, at `soc% ≤ L` → `L − 1` | **cutoff + 0.9 %** (12.9 % at cutoff 12 %) | present | decompilation `0x080093bc`; measured on a Venus D (BMS 118) on 10/11 Oct 2026 |
| A | 109 | function body **byte-identical** to A 1105 (`0x08009384`) | **cutoff + 0.9 %** | byte-identical to 1105 (`ModeBySoc`, `Mode1/2/3`, `ChargeStepTarget`, `Discharge_StepRotate`) | byte comparison, not decompiled; measured by a third party on a Venus A with 2 packs (cutoff 12 %): a pack that was already at 12.1 % was handed over below 12.0 % (threshold `L − 1`), the final stop came at total SOC 12 % with the other pack at 14.1 % |
| A | 1105 | same formula as D 118, only addresses differ | **cutoff + 0.9 %** | present | decompilation `0x0800942c`, not measured |
| A | 110 | identical to A 1105 (`0x08009384`); pack-selection code unchanged, only the current limit differs (40 A) | **cutoff + 0.9 %** | identical / only call targets differ | byte comparison and decompilation, not measured |
| A | 115 | threshold function byte-identical to 1105 (`0x08009EB4`); **stale threshold fixed**, new spread switch (see below) | **cutoff + 0.9 %** | `ModeBySoc`, `Mode1/2/3` identical; `ChargeStepTarget`, `Discharge_StepRotate`, `ModeDispatch` changed | decompilation, not measured |

Notes to the table:

- `soc%` is the SOC ×10 divided by 10 as an **integer** (12.9 % → 12). The hand-off therefore happens as
  soon as the pack falls below `L + 1`, which shows up as `L + 0.9 %`, not as `L`.
- `L = 100 − DoD`; the Control limits the cutoff to 12–70 % (app range DoD 30–88 %, default 12 %).
  Until the Control has sent a DoD the BMS default is DoD 90 (`0x5A`), i.e. `L = 10`.
- The threshold is only recalculated at a pack change; a stale value lets a pack run below `L`
  (details: `BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md` §5.3).
- Verified by byte search and decompilation only. Not compared for A 109: the CAN heartbeat path that
  updates `g_dischargeLimitPct` at runtime (present in A 1105 and D 118, absent in D 117.7).
- **A 115 differs in two points** (details: `BMS/VNSA_BMS_Diff_v110_v115.md`, release note items 1 and 2):
  - `PackSelect_Discharge_StepRotate` recomputes the stored threshold as soon as it lies more than 10
    points below the SOC of the active pack (dynamic target correction). This removes the stale threshold
    of D 118 / A 1105 (§5.3 of `BMS/VNSD_BMS_Pack_Rotation_Taper_v118.md`).
  - New `PackSelect_SocImbalanceSwitch` (`0x08007B98`, called by `PackSelect_ModeDispatch` in mode 0): if
    the spread between the fullest and the emptiest online pack is ≥ 20 points, the load is handed to the
    fullest pack (discharging) or the emptiest (charging) after a power-dependent delay of 1200 / 1800 /
    2400 / 3000 ticks (> 1500 W / > 1000 W / > 500 W / otherwise).
  - The threshold itself (`L = 100 − DoD`, integer compare) is unchanged, so the hand-off point stays at
    cutoff + 0.9 %.

**E3:** own BMS firmware. The archive holds the images 106, 110, 112, 113 and 115; 115 is decompiled
(`BMS/VNSE3_BMS_FW_v115_Full_Analysis.md`). It reports one pack and has no pack selection or rotation, so
the threshold table above does not apply. The archive contains no taper code. The E3 taper is therefore
unknown, and the Modbus address of the current limit on the E3 is not yet evidenced.

## 4. Firmware Bugs Across the Models

### Exception Frame — Evidenced on All Three Models

**Venus E3.** `Modbus_Dispatcher` is at `0x0801d278` and, at 392 bytes, exactly as large
as in the D. The bytes from `0x0801d358` are character for character the same as in the D.

**Venus A.** `Modbus_Dispatcher` is at `0x0801e3ac`, likewise 392 bytes. Of the 392 bytes,
23 differ from the D, and exclusively at branch offsets and in the literal pool.
The window around the faulty spot is identical:

```
D 0801ea04+0xE0: 00 BF 00 BF 2C 48 00 78 80 B1 00 20 29 49 08 71 04 20 48 71 08 46 C0 79 80 30 C8 71 26 48 00 78
A 0801e3ac+0xE0: 00 BF 00 BF 2C 48 00 78 80 B1 00 20 29 49 08 71 04 20 48 71 08 46 C0 79 80 30 C8 71 26 48 00 78
                                                        ^^^^^  movs r0,#4
```

`movs r0,#4` is at offset `0xF0` of the dispatcher in both images, i.e. at
`0x0801e49c` in the A. An exception PDU is always two bytes long, so the MBAP length field
should therefore be 3.

**Thus Venus D, Venus A and Venus E3 carry the same bug, statically proven.**
So far only the D has been checked on the device.

### Readback Value 44002/44003 — Same Construction in the A

The read path of the A is unchanged: both registers are served from a shadow value that
only a write access or a CAN frame fills. The pointer `DAT_08052360` resolves to
**`0x20000156`** — **the same SRAM address as in the D**. The bug "reads permanently 0 after a
restart" therefore also applies to the Venus A. Only the D has been checked on the device.

### Read Serializer

Not checked individually on A and E3. `Read_Serializer` is captured in the A (section 2) and
shows no deviation from the D in the decompilation comparison, so the sign error is very probably
present there as well.

---

## 4b. Unchecked

- **A v1509 versus A v150** — what the newer A firmware changed. Now sensible to
  carry out, because A v150 is 73.4 % covered. For v1509 the same
  anchor run from section 1d is recommended.
- **The predictions from section 3a on the Venus A**: 2000 to 44002 → accepted, effect
  1500; 2000 to 44003 → exception 3.
- **Whether the Venus A has a DE/RE line** (section 3d) — on the device or from the schematic.
- **`FUN_08015bd0`, `FUN_08059408`, `FUN_080025a4`, `FUN_08017f94`** and around 20 further
  small functions for which no function at all sits at the expected place in the A. This
  remainder list has not been worked through yet.

---

## 5. Methodology

Four pitfalls that occurred in this comparison:

**"Changed" in the diff does not mean changed.** With images disassembled to unequal extents,
callee artifacts dominate the ranking. Only `similarity`, `sizeDelta` and the correlator used
together tell whether a real deviation exists.

**A search over a half-disassembled image yields false negatives.** That the
register constants were not found in the A did not mean that they do not exist — and as
section 2 shows, they do not exist in the complete D image either. A negative search is only
meaningful if it was validated beforehand on an image in which the code being searched for
is certainly captured.

**A negative search is only as good as the converted constant.** Here `0x5DD`
was searched for 1500 and `0x9C5` for 2500; the correct values are `0x5DC` and `0x9C4`. Both times the
wrong search produced zero hits, and from these zero hits a substantive statement was
derived — which turned out to be the exact opposite of the truth. For a search
whose result is "nothing found", the conversion should be cross-checked, ideally on
an image in which the value certainly occurs.

**A byte comparison without relocation awareness systematically underestimates the similarity.**
Between A and D, literal pool pointers and `bl` branch targets are shifted by the layout offset.
Compared raw, identical functions land at 88–94 % and fall through every
reasonable threshold; when the offset is factored out, it is 100 %.

**And a positive observation:** a decompilation diff with normalized symbol names cleanly separates
real logic differences from address noise. `Grid_Export_Power_Limiter` looked like a massive deviation
in the byte comparison at 53 % and actually had three
changed lines. Conversely, `Write_Handler` at 95.7 % looked almost identical and contained
exactly the one constant that mattered.
