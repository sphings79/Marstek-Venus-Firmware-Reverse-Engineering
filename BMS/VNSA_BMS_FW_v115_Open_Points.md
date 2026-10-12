# BMS-FW V115 (Venus A) — Open points in depth

> 🇩🇪 Deutsche Version: [VNSA_BMS_FW_v115_Offene_Punkte_de.md](VNSA_BMS_FW_v115_Offene_Punkte_de.md)

**Status:** 2026-10-11. **Parent document:** [VNSA_BMS_FW_v115_Full_Analysis.md](VNSA_BMS_FW_v115_Full_Analysis.md) (§13).
**Sources:** decompilation and listing `Dekompilate/VNSA_BMS_115/` (local only), cross-checked against the exports of A BMS 110 /
110.5, D BMS 117.7 / 118, E3 BMS 115, A Control 150.9 / 151, D Control 147–151, E3 Control 151, A Micro 1211 and E3 Micro 119.
Nothing in this document was measured on a device.

**Proof levels used below**

| Mark | Meaning |
|---|---|
| **[C]** | Proven by the code of A BMS 115 (function/address given) |
| **[X]** | Proven by the code of another firmware image in the set (image named) |
| **[I]** | Inference from code facts; plausible, not proven |
| **[U]** | Unknown; not decidable from the available firmware |

## 0. Context: file name `VEPRO_BMS`

- File names of the Venus A BMS images: A 110 `…_VABMS_APP_110_…`, A 110.5 `…_VA50A_APP_V1105_…`, A 115
  `…_VEPRO_BMS_V115_APP_20260901_…`. The OTA record is filed under device type `VNSA-0`, the release note reads "VNSA115 Official
  Edition".
- The code is **Venus A code**: masked segment coverage 82.4 % against A 110.5 (81.4 % against A 110, 65.7 % against D 118,
  12.6 % against E3 115); the string set equals A 110/110.5 apart from build/version strings; the string `VNSA` is present as in
  110/110.5; the string `VEPRO` does **not** occur in the image **[C]**.
- The app vocabulary (app 1.6.76, [../App/Marstek_App_Command_Vocabulary_1.6.76.md](../App/Marstek_App_Command_Vocabulary_1.6.76.md))
  lists the Venus type codes `VEPRO`, `VEPROCH` and `VEPRO2` next to `VNSA`/`VNSA2` **[X]**.
- **Assumption (from the name only, not confirmed by the firmware):** `VEPRO` stands for a planned "Venus E Pro", possibly in two
  models (`VEPRO`/`VEPRO2`), that reuses the Venus A BMS code base. Nothing in the BMS image refers to such a product.

## 1. Configuration checksum covers its own word

### 1.1 Code facts

| Item | A 115 |
|---|---|
| Block | RAM `g_cfgMagicByte` `0x20003F60` … `0x20004037`, EEPROM address 600 (0x258), **0xD8 bytes** |
| Payload | offsets 0x00–0xCF, every byte belongs to a named field (last: `g_cfgTailReg20` 0xCC, `g_cfgTailReg19` 0xCE) **[C]** |
| Checksum word | `g_cfgChecksum` (int32) at offset **0xD0** (`0x20004030`) |
| New field | `g_cfgPackUvShutdownTimeout` (uint32) at offset 0xD4, default 120 — not covered by the checksum |
| Checksum function | `LIB_Checksum_Ones_Complement` `0x0800BE28`: `-1 - Σ data[i]` (byte sum, int32) |
| Save | `APP_Config_Checksum_And_Save` `0x0800B7E4`: `movs r1,#0xd4; bl 0x0800BE28; str.w r0,[r1,#0xd0]; movs r2,#0xd8` → checksum over **0x00–0xD3**, stored at 0xD0, 0xD8 bytes written **[C]** |
| Load | `BMS_Config_Params_Load_Or_Default` `0x0800A840` (called once from `System_Init_Main`): read 0xD8 bytes, checksum over **0x00–0xD3** (`movs r1,#0xd4` at `0x0800A854`), compare with the word at 0xD0 (`0x0800A86A`), compare tag with `'s'` (0x73). On read error, mismatch or wrong tag: all fields = defaults, then `APP_Config_Checksum_And_Save` **[C]** |
| Other writers | `RS485_Register_Write_Handler` `0x0800B10C` (RS485 opcode 0x12, groups 0x01/0x02/0x04/0x08/0x10/0x40/0x100/0x200/0x400), `RS485_RegisterWrite_DefaultGroup` `0x0800B212` (+ thunk `0x0800B20C`): same inline sequence (0xD4 / 0xD8); `RS485_Dispatcher` opcodes 0x1E (`g_cfgUserFlag1`) and 0x20 (`g_cfgTailReg19`) and `BMS_Config_Params_Factory_Reset` (RS485 opcode 10 via `APP_Factory_Reset_And_Reboot`) call `APP_Config_Checksum_And_Save`. No CAN path writes the block **[C]** |

### 1.2 Arithmetic

Let `S` = byte sum of offsets 0x00–0xCF, `c_old` = the int32 at 0xD0 **in RAM before the save**, `b(c)` = byte sum of the four
bytes of a word. The save computes

`c_new = -1 - (S + b(c_old)) = ~(S + b(c_old))`

Because `S + b(c) ≤ 0xD4 × 255 = 54060 < 65536`, the two upper bytes of every computed word are `0xFF 0xFF`, and with
`T = S + b(c_old) = 256·hi + lo`: `b(c_new) = 1020 − lo − hi`.

At the next boot the load reads `c_new` and `S` back and computes `~(S + b(c_new))`. It passes only if

`~(S + b(c_old)) == ~(S + b(c_new))`  ⇔  **`b(c_new) == b(c_old)`**

i.e. **the stored block verifies only if the byte sum of the new checksum word equals the byte sum of the checksum word that was in
RAM before the save** **[C]** (pure arithmetic on the code above).

On a failed check the defaults are written into the payload, but `g_cfgChecksum` keeps the value just read from the EEPROM; the save
therefore iterates `x_{k+1} = g(x_k) = 1020 − lo(S+x_k) − hi(S+x_k)` with `x = b(c)`. Within a 256-block without carry
`g(x) = K − x`, so the sequence **alternates between two values** and never settles, except when it hits the fixed point.

**Worked numbers for the default block** (defaults of `BMS_Config_Params_Load_Or_Default`, `S_def = 16215 = 0x3F57`) **[C]**:

| Start (word in RAM before the first save) | Stored words on successive boots | Result |
|---|---|---|
| Erased EEPROM (`0xFFFFFFFF`, b = 1020) | `0xFFFFBCAC`, `0xFFFFBD42`, `0xFFFFBDAB`, `0xFFFFBD42`, `0xFFFFBDAB`, … | fails on every boot |
| Block written by A 110 with defaults (assumes 110 defaults = 115 defaults except tag `'n'`; word `0xFFFFC0AD`, b = 875) | `0xFFFFBD3D`, `0xFFFFBDB0`, `0xFFFFBD3D`, … | fails on every boot |
| Read error (RAM word 0 from `.bss`, b = 0) | as erased EEPROM after the first step | fails on every boot |
| Word with b = 690 | `0xFFFFBDF6`, then `0xFFFFBCF7` constantly | passes from the third boot on |
| Word with b = 945 (`0xFFFFBCF7`) | `0xFFFFBCF7` | passes |

For `S_def` the only fixed point is b = 945 (word `0xFFFFBCF7`); starting byte sums that ever reach it are 180, 435, 690 and 945.
A computed word always has b ≥ 510, so only 690/945 are reachable from blocks written by the firmware. Over all `S` and all start
values 510…1020, only ~0.4 % of the combinations ever verify within 40 boots (simulation of the formula above) **[C]**.

A changed parameter (RS485 opcode 0x12/0x1E/0x20) changes `S`; even when the block was in the stable state b = 945, a payload change
by +1, +2, +10 or +100 does not keep 945 as a fixed point, so the next boot fails and reloads the defaults **[C]**.

### 1.3 Consequences

- **Parameters written over RS485 (protection thresholds, timeouts, `g_cfgUserFlag1`, `g_cfgTailReg19`) do not survive a reboot**
  in practically all cases; at the next boot the defaults are loaded and saved again **[C]**.
- Packs that were never re-parameterised see no functional difference: they would run on the defaults anyway. Side effect: a
  0xD8-byte EEPROM write at almost every boot **[C]**; whether that matters for EEPROM endurance is **[U]** (EEPROM type not identified).
- After the update from 110/110.5 the defaults are loaded once in any case (tag `'n'` → `'s'`, block length 0xD4 → 0xD8, see
  Full Analysis §11) **[C]**.
- Probable origin: the block grew by 4 bytes and the length argument of the checksum was raised from 0xD0 to 0xD4 together with the
  block length instead of staying at the checksum offset **[I]**.

### 1.4 Other images

| Image | Block | Checksum range | Word at | Correct? |
|---|---|---|---|---|
| A 110 / A 110.5 | 0xD4, tag `'n'` | 0x00–0xCF (`0xd0`) | 0xD0 | yes **[X]** |
| D 117.7 / D 118 | 0xD4, tags `'u'` / `'v'` | 0x00–0xCF | 0xD0 | yes **[X]** |
| E3 115 | 0xA8, tag `'s'` | 0x00–0xA3 (`0xa4`) | 0xA4 | yes **[X]** |
| **A 115** | 0xD8, tag `'s'` | **0x00–0xD3** | 0xD0 | **no** **[C]** |

The defect exists only in A 115.

### 1.5 How to test on a real pack

1. **Passive, no write by the tester (preferred):** logic analyser on the bit-banged EEPROM bus (PB10/PB11, device address 0xA0,
   16-bit word address; `INIT_I2C_EEPROM_GPIO`, `I2C_EEPROM_Read`). Power-cycle the pack a few times. Expected per code: at every
   boot a read of 0xD8 bytes at 0x0258 followed by a **write** to 0x0258; the bytes at block offsets 0xD0–0xD3 alternate between two
   words (e.g. `3D BD FF FF` / `B0 BD FF FF` after an update from 110 with default config). No write after the read = the check
   passed (fixed point reached).
2. **Offline, read-only:** with the pack disconnected and the MCU held in reset, read EEPROM 0x0258–0x032F with a clip programmer.
   Compute `S` = Σ bytes 0x00–0xCF, `x` = Σ bytes 0xD0–0xD3; the block will verify at the next boot iff
   `word(0xD0) == 0xFFFFFFFF − (S + x)`. Dump again after one boot: a changed word without any parameter change confirms the defect.
3. **Active (writes, only if acceptable):** RS485 opcode 0x11 read of one parameter group, opcode 0x12 write of a harmless value
   (e.g. a release timeout), opcode 0x11 read-back, reboot (power cycle or opcode 2), opcode 0x11 again. Expected: the default value
   is back. In the rare fixed-point case the changed value stays — restore it manually.
   CAN does not help: no CAN frame carries these parameters or the checksum.

### 1.6 Not known

- Whether Marstek's service tooling ever writes these parameters on A packs, i.e. whether the defect has a practical effect **[U]**.
- Whether the bootloader or production touches the block **[U]** (bootloader not available).
- Whether a later version fixes it **[U]**.

## 2. Unknown CAN ids

### 2.0 Common facts

- All four inbound ids are decoded only in `CAN_RX_Handler` `0x08008BD4`, which `APP_CAN_RX_Task_Process` `0x080035D0` calls
  **only on slot 1 (master)**; other slots use `CAN_RX_Command_Dispatcher`/`CAN_RX_System_Command_Handler`, which contain none of
  these id constants. Payload byte n = frame offset `0xB + n` **[C]**.
- **Id structure of the internal bus** (A Control `CAN_Build_Arbitration_ID` `0x08037798` in Control 151, A Micro 1211
  `CAN_TxQueue_PushReply` `0x0801A420` / `CAN_TxQueue_PushMultiFrame` `0x0801A5B0`) **[X]**: request
  `0x00 | 0x01<<16 | (class<<4 | node)<<8 | func`; the micro replies with source and destination swapped and the frame index in
  bits 24–28: class 4 → `idx<<24 | 0x41<<16 | 0x01<<8 | func`, class 2 → `idx<<24 | 0x21<<16 | 0x01<<8 | func`.
- The A micro accepts BMS frames `(id & 0x1FF0FFFF) == 0x1800AA01` (`CAN_RxDispatch` `0x08019360`), so micro and BMS share the bus
  and the BMS sees the micro's replies **[X]**.
- The A Control 151 polls class-2 function 0x03 (`Inverter_ClearReg_0x03_Default` `0x0802CFEC`, periodic timer ≈ 510 ms) and
  class-4 function 0x10 (`Inverter_ClearReg_0x10_Default` `0x0802D06A`, ≈ 520 ms) **[X]**.
- The A micro's documentation: [../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Full_Analysis.md](../VNS_Micro_Inverter/VNSA_Micro_FW_v1211_Full_Analysis.md) §4.1/§4.2.

### 2.1 Inbound `0x01210103`

| Item | Finding |
|---|---|
| Handler | `CAN_RX_Handler`, branch `0x08008D5C` **[C]** |
| Parsing | `g_can01210103Flag` (`0x20002AA5`) = (data[0] >> 4) != 0; DLC not checked; the flag is only rewritten by the next such frame (no timeout) **[C]** |
| Use | `PackSelect_RelayCommStateMachine` `0x0800BEE8` (read at `0x0800BF10`): if `g_emsHeartbeatModeByte != 2`, `g_relayCommEscalationInhibit` = flag OR \|`g_sysTotalCurrent_x10`\| ≥ 11 (1.1 A). The inhibit suppresses the 300-tick escalation to fault mode 3 and the 600-tick escalation to fault mode 2 and the LED relay-comm phases (`LED_StatusPatternUpdate`); the 1000-tick escalation to mode 4 is not inhibited **[C]** |
| Sender | **A Micro 1211**: frame index 1 of the class-2 reply to function 0x03 (`CAN_RxDispatch` class 2, func 3 → `Mppt_BuildStatusReport`, `CAN_TxQueue_PushReply(3,1,8,…)`); data[0] = `g_mpptRptStateByte`, bits 0–3 MPPT charge state, **bits 4–7 = run flags of PV channels 4/3/2/1**; data[2..3] fault code, data[4..5] chip temperature, data[6..7] warning code **[X]** |
| Meaning | flag = "at least one PV channel of the A micro's MPPT stage is running" **[X]**; the BMS uses PV activity to keep the relay-comm state machine from escalating **[I]** |
| Confidence | High for sender and field (code on both sides); not observed on the bus |

### 2.2 Inbound `0x04410110`

| Item | Finding |
|---|---|
| Handler | `CAN_RX_Handler`, branch `0x08008CFE` **[C]** |
| Parsing | `g_can04410110Ratio` (`0x20002AA8`, float) = int16(data[4..5]) × 10000.0 / uint16(data[6..7]) (soft-float, `0x08008CFE`–`0x08008D56`; no check for a zero divisor) **[C]** |
| Use | **none**: the only literal-pool reference is the store at `0x08008D54`; no base+offset access found **[C]** |
| Sender | **A Micro 1211**: frame index 4 of the class-4 func-0x10 telemetry reply = telemetry bytes 24–31: grid power W (0x18), off-grid power W (0x1A), **battery power W (0x1C, int16)**, **battery voltage × 10 (0x1E, uint16)** **[X]** |
| Meaning | W × 10000 / (V × 10) = **battery current in mA** as seen by the micro **[X]** (arithmetic) |
| Cross-check | E3 BMS 115 decodes the same frame with the same formula into `g_vnsCurrentMa` and uses it in the SOC integration; E3 Micro 119 sends it as frame 4 of its telemetry ([../VNS_Micro_Inverter/VNSE3_Micro_FW_v119_Full_Analysis.md](../VNS_Micro_Inverter/VNSE3_Micro_FW_v119_Full_Analysis.md) §4.1.5) **[X]**. In A 115 the receive code was taken over without a consumer **[I]** |
| Confidence | High |

### 2.3 Inbound `0x03410110`, byte 6

`g_pvRescueRequestFlag` (`0x20002AA4`) = (data[1] > 8) OR (data[6] == 1) (`0x08008CE2`) **[C]**. The id is frame index 3 of the A
micro's func-0x10 telemetry = bytes 16–23: grid voltage × 10 (0x10), grid frequency × 10 (0x12), off-grid voltage × 10 (0x14),
`g_gridPermit` (0x16, u16) **[X]**. So data[1] > 8 means grid voltage ≥ 230.4 V (as already documented for 110.5) and **data[6] == 1
means low byte of `g_gridPermit` == 1** **[X]**. With an A micro 1211 the flag named "PV rescue" in this project therefore means
"grid voltage ≥ 230.4 V or grid permit set" **[I]**; what the BMS developers intended is **[U]**.

### 2.4 Inbound `0x1801FFAA`, byte 3 == 2

| Item | Finding |
|---|---|
| Handler | `CAN_RX_Handler`, branch `id − 0x100501AA == 0x07FCFE00`: besides the existing fields, `g_emsHeartbeatModeByte` (`0x20002A29`) = data[3] **[C]** |
| Readers | `PackHandoff_Tick` `0x08007150`: value 2 **and** `g_pvRescueRequestFlag` → `PackHandoff_Charge_SocSwitch_SM` `0x08006B5C` (active pack current ≥ 0) or `PackHandoff_Discharge_SocSwitch_SM` `0x08006D0C` instead of `PackHandoff_StateMachine` (switch order see Full Analysis §6); `PackSelect_RelayCommStateMachine`: value 2 → inhibit = `g_pvRescueRequestFlag` only (the 0x01210103 flag and the current are then ignored) **[C]** |
| Senders checked | `Grid_BuildStatusFlagsReg` in A Control 150.9 / 151 and D Control 147 / 149 / 149.2 / 150 / 151 writes bytes 0–2 only (byte 3 = 0 from the zero-initialised buffer); E3 Control 151 sends eight zero bytes. No other image in the set sends this id or reads byte 3 (A 110/110.5, D 118, E3 115 BMS do not) **[X]** |
| Conclusion | With every available Control, byte 3 is 0 → the handoff state machines and the mode-2 inhibit are dead code on today's systems **[X]**. Sender of value 2: **[U]**; a later Control version or a different product (see §0) is a possibility **[I]** |
| Confidence | High that no available Control sends 2; meaning of "mode 2" unknown |

### 2.5 Outbound `0x1805AA01`

| Item | Finding |
|---|---|
| Sender | `APP_IdleKeepaliveBroadcast_Tick` `0x0800F144`, called from the CAN RX task (10 ms) on every pack: once `g_idleKeepaliveTimerMs` ≥ 50 ms, every call queues the frame (DLC 1); the timer is only reset on the master path, so non-master packs send it on every 10 ms call **[C]** |
| Payload | data[0] = `g_mosRecoveryReport` (`0x20002A25`): 0 after boot, 1 recovery running (state 2/3), 2 finished; never reset to 0 after a recovery **[C]** |
| Receivers checked | A Micro 1211: id passes the mask `0x1800AA01`, but frame number 5 is outside 1..4 and is dropped **[X]**. A Control 151: `CAN_FrameDispatcher` → `Protocol_AA_CommandDispatch`, 17-entry table `g_canAaDispatchTable` (initialised `.data`; sub-ids readable in the compressed load image: 0x1801–0x1804, 0x1820–0x1823, 0x1830–0x1833, 0x1840/0x1841, 0x1007, two 0x185x entries) — **no 0x1805**; `CAN_ParallelInverterDataParser` does not match it **[X]**. No BMS image in the set receives it **[X]** |
| Conclusion | No receiver in the available firmware; a future Control is a possibility **[I]** |
| Confidence | High for "no receiver in the set" (the A Control table was read from the compressed load image, not from a decompressed copy) |

## 3. Pins PE7 and PD8

### 3.1 GPIO helper names were swapped (finding, corrected)

`GPIO_Init_Pin` `0x08009C3C` is the STM32F1-style init: descriptor byte 3 = mode; if bit 4 is set it is an output and the speed
(byte 2) is ORed in. `GPIO_Init_Output_PP_50MHz` `0x080042F4` passes mode **0x04 = floating input**, `GPIO_Init_Input_Floating`
`0x080043B4` passes mode **0x10 + speed 3 = push-pull output 50 MHz** **[C]**. The names were swapped (corrected, see below). This matches the usage: PB13 and
PE9 (read with `HAL_GPIO_ReadInputBit_Wrapper`) are set up with the "Output" helper, the switch outputs PC5/PB0/PB2 with the "Input"
helper (pre-correction names). Same swap in A 110, A 110.5, D 117.7, D 118; correct in E3 115 **[X]**. **Corrected:** the two names are now swapped back in the Ghidra programs of A 110, A 110.5, A 115, D 117.7 and D 118 (`GPIO_Init_Input_Floating` = mode 0x04, `GPIO_Init_Output_PP_50MHz` = mode 0x10 + speed 3), and the plate comments of their callers were adjusted. The addresses quoted above are the pre-correction names; after the correction `0x080042F4` is `GPIO_Init_Input_Floating` and `0x080043B4` is `GPIO_Init_Output_PP_50MHz`.

Effective setup in `GPIO_Init_All_Pins` `0x08004148` of A 115 **[C]**: outputs PC8, PE6, PE15, PE8, PB5, PB12, PD7, PD15, PD14, PA4,
PB14, PB0, PB2, PC5; inputs PC4, PD3, PB13, PE10, PE12, PB1, PE9, PC1, PD13. `GPIO_DeInit_And_CAN_NVIC_Init` (before STOP) turns
PD7, PD14, PD15, PE8, PB13, PD3, PA4, PB14, PB0, PB2, PC5, PC9, PE15, PA12 into floating inputs.

### 3.2 PE7 — MOS recovery

| Item | Finding |
|---|---|
| Driver | `HAL_GPIO_Control_Switch` `0x08006890` actions 7/8 = PE7 high/low **[C]** |
| Start | `OwnPack_MOSFET_Driver_MasterVariant` `0x080078C8` (single master via `BMS_Role_Dispatcher`, and `PackSelect_Mode3_IdleTick`): in the two "charge allowed" branches, instead of `Control_Switch(1)` (PC5 on) as in 110.5, it reads the **output latch** of PC5; if it is 0 and `g_mosRecoveryState == 0` → state 1. So the recovery runs on **every off→on transition of the charge enable** in these roles, not only after a fault **[C]**. Slave packs (`OwnPack_MOSFET_Driver`) and `Master_MultiPack_Tick` do not use it **[C]** |
| Sequence | `OwnPack_MosRecovery_SM` `0x08005BA4`, ticked every 100 ms after the precharge SM: state 1 PE7 on → state 2 PE7 on, wait until `g_pack0Voltage_x10 > 849` → report 1 → state 3: after 20 ticks PE7 off, **PC5 on**, report 2, state 0. Minimum PE7-on time ≈ 2.1–2.2 s **[C]** |
| 8.49 V | `g_pack0Voltage_x10` is in **10 mV** despite its name (`PerPack_Struct_Builder`: `g_sh36735PackVoltageMv / 10`; same scale as the 36.00 V/35.00 V/20.00 V thresholds), so 849 = **8.49 V**. Source: AFE register 0x93 (`SH36735_ConvertPackVoltage`, shell `VBAT2`), 0 after boot (`.bss`) **[C]**. For a 13S pack in normal range (≈ 32.5–47.5 V) the condition is true at once, so state 2 lasts one tick unless the value is still 0 or stale (AFE read failure skips the update) **[I]**. If register 0x93 measured a node behind the MOSFETs, 8.49 V would read as "output side pre-charged" — **[U]**, the AFE node is not identifiable |
| No timeout | If the value stays ≤ 8.49 V, state 2 holds PE7 on indefinitely and PC5 stays off **[C]** |
| States 4/5 | State 4 (retry after 1 s, max 2 retries) and 5 (set `g_protState1` bit 2) exist, but **no code writes 4**: the only writers of `g_mosRecoveryState` are the SM itself (0/1/2/3/5, 5 only from 4) and the master driver (0 → 1) **[C]**. Unreachable in this image |
| Interaction | Tick order: protection tick, then role dispatcher (driver → precharge SM → recovery SM). In state 2 a protection branch switching PE7 off is overridden in the same tick. At the end of state 3 PC5 is switched on without re-checking protection; a still-active protection switches it off again on the next tick (≤ 100 ms later) **[C]**; electrical effect **[U]** |
| Reporting | Only via `0x1805AA01` (§2.5); status bit 7 of `g_pack0StatusFlags` = PE7 latch, but bits 6/7 are not put into any CAN or RS485 status byte (`APP_Get_Protection_Status_Byte` packs bits 0–5, `CAN_TX_PerPack_10Msgs` bits 4/5) **[C]** |
| Pin mode | **PE7 is never configured**: no init call with port E mask 0x80, no direct access to `GPIOE_PL_CFG`/`PH_CFG` outside the init helpers **[C]**. After reset an STM32F1-compatible GPIO pin is a floating input, so writes to the output latch would not drive the pin **[I]** (reference-manual behaviour). Whether the bootloader configures it is **[U]** |

### 3.3 PD8 — output below 20.00 V

| Item | Finding |
|---|---|
| Function | `Pack_LowVoltageOutput_PD8_Update` `0x0800A330`, called from `BMS_ProtectionMasterTick` `0x0800C78C` (protect task, 100 ms, all roles): PD8 latch = 1 while `g_pack0Voltage_x10 < 2000` (20.00 V), else 0 **[C]** |
| Related | `HW_OvercurrentAndSensorProtection` `0x0800BC70` clears and skips the NTC-count and MOS-temperature checks below the same 20.00 V **[C]** |
| When true | Healthy 13S pack: never (20 V ≈ 1.54 V/cell); at boot until the first AFE conversion (value 0) and with a 0/stale reading **[I]**. The 20 V gate looks like a "measurement not valid / board not on the cells" detection **[I]** |
| Pin mode | **PD8 is never configured**: `GPIO_Init_All_Pins` and `GPIO_DeInit_And_CAN_NVIC_Init` use port D mask 8 (= **PD3**, not PD8) and 0x80/0x2000/0x4000/0x8000; the KA495xx/SPI pins are PD10–PD12 (0x1C00) **[C]**. Same consequence as for PE7 **[I]**; bootloader **[U]** |
| Board function | **[U]** |

### 3.4 Comparison with other images

| Pin | A 110 | A 110.5 | D 118 | E3 115 | A 115 |
|---|---|---|---|---|---|
| PE7 | not used | not used | not used | configured as push-pull output, low at init, floating input before STOP; no other access **[X]** | written by MOS recovery, never configured |
| PD8 | not used | not used | not used | not used | written every 100 ms, never configured |

(Search for port E mask 0x80 and port D mask 0x100 in all exports.) The E3 board has PE7 as an output that is only held low; whether
it is the same net as on the Venus A board is **[U]** (different hardware).

### 3.5 What is proven vs. unknown

- Proven: when the latches are written, the exact conditions, the unreachable states, the missing pin-mode setup in the application.
- Unknown: the board nets of PE7 and PD8, whether the bootloader sets their mode, which AFE node register 0x93 measures, and therefore
  whether the new recovery and the PD8 signal have any electrical effect on the current Venus A board. Possible reading: both
  outputs prepare hardware of a different board revision (cf. §0) **[I]**.
