# Control FW — Function Tracking (by Topic)

> 🇩🇪 Deutsche Version: [VNSD_Control_FW_Function_Tracking_new_de.md](VNSD_Control_FW_Function_Tracking_new_de.md)

**Binary:** `VNSD-0_app_1492_0702_142136.bin` (v149.2)  
**Total named functions (Ghidra, as of 2026-07-09 after Batch 20):** 1619

**This is the authoritative reference for function names/addresses of the Control FW** — grouped by
topic area (mbedTLS cluster, MQTT, BLE, hardware, ...) instead of by chronological batch. Addresses and
names are pulled directly from Ghidra (status after Batch 19/20, 2026-07-09), including the corrections from
Batch 18 (62 mbedTLS/miscellaneous renamings) and Batch 19 (14 resolved duplicate names, see below).
Descriptions were taken over from the former batch docs where they still match the current address/name;
where that was not the case, the description column is empty.

> **Note (2026-07-11):** The original chronological batch log (`Control_FW_Function_Tracking.md`)
> was deleted because it was redundant to this file (same address→name pairs, but ordered by batch instead of
> by topic) and, at the user's request, is not to be maintained any further — new name assignments
> will be maintained directly here from now on. The process notes/lessons from the batches are archived in
> [VNSD_Control_FW_Naming_Batch_History.md](VNSD_Control_FW_Naming_Batch_History.md).

## Batch 19 — Resolution of All Duplicate Names (2026-07-09)

While building this document, in addition to the address reconciliation check, a full scan for duplicate
function names was run on the *current* Ghidra status (not only for the 6 previously known). Result: **14**
functions carried duplicate names; all were renamed in Ghidra and updated in both tracking documents:

| Address | Old name | New name | Rationale |
|---|---|---|---|
| `0x0801d6cc` | MQTT_Decode_RemainingLength | MQTT_Decode_RemainingLength_ViaCallback | Reads bytes via a passed function pointer (callback), separate from the transport object path; only 1 caller (wrapper) |
| `0x08033044` | MQTT_Decode_RemainingLength | MQTT_Decode_RemainingLength (unchanged) | Canonical variant: reads via transport vtable call in MQTT_ReceivePacket — keeps the name |
| `0x080329b2` | CH395_Socket_SendData | CH395_Socket_SendData_ViaSocketPtr | Thin wrapper, dereferences the socket pointer and directly calls CH395_SPI_Send_Data, no mutex/polling, 0 callers |
| `0x0802d12c` | CH395_Socket_SendData | CH395_Socket_SendData (unchanged) | Full implementation with mutex queue, byte-by-byte transmission and status polling — keeps the name |
| `0x080320ba` | mbedTLS_ASN1_Get_Int | mbedTLS_ASN1_Get_TaggedInt_Core | Generic internal helper, tag as a parameter (not hard-coded) — used by Get_Int AND presumably Get_Bool |
| `0x0803e8c8` | mbedTLS_ASN1_Get_Int | mbedTLS_ASN1_Get_Int (unchanged) | 2-line wrapper, calls Core with hard-coded tag=2 (INTEGER) — corresponds exactly to the real mbedtls public API |
| `0x08033434` | mbedTLS_ECP_Group_Load | mbedTLS_ECP_Group_Load (unchanged) | 2 parameters (grp, id), calls Group_SetById — corresponds to the real mbedtls_ecp_group_load(grp,id) public API |
| `0x08033db8` | mbedTLS_ECP_Group_Load | mbedTLS_ECP_Group_Load_RawParams | 13 parameters (P/A/B/Gx/Gy/N each as ptr+len) — corresponds to the internal static ecp_group_load() from ecp_curves.c |
| `0x0804a6fa` | prvCopyDataToQueue | prvCopyDataToQueue (unchanged) | Pure memcpy/wraparound logic on the queue memory block — corresponds to the real FreeRTOS-internal prvCopyDataToQueue |
| `0x0804abdc` | prvCopyDataToQueue | prvQueueSend_CopyAndNotify | Calls the real prvCopyDataToQueue itself and additionally handles lock-count/notify logic — presumably a part of xQueueGenericSend/-Receive outlined by the compiler |
| 0x08036a8e/0x08036b18 | llhttp__internal__c_test_flags (×2) | …_Off32 / …_Off2e | Identical bit-test code, but on two different struct fields (offset 0x32 and 0x2e respectively) — the suffix names the tested field |
| 0x08036a9a/0x08036ade, 0x08036aa6/0x08036ad2, 0x08036ab2/0x08036af6 | llhttp__internal__c_test_flags_{1,2,3} (each ×2) | …_Off32 / …_Off2e | Same pattern as above, once each for field @0x32 and @0x2e |

This resolves **all** duplicate names known at the time this document was created in the Ghidra project
(0 remaining, verified by a full scan over all 1417 named functions).

In addition, 6 functions became visible when querying Ghidra again that had not been captured before (or only under a
default name) — presumably due to auto-analysis continuing in the background: `OTA_Update_Dispatcher`,
`vTaskEnterCritical`, `vTaskExitCritical`, `xTimerStop_Internal`, `FreeRTOS_StartScheduler`,
`FreeRTOS_SysTick_TaskUnblock`. These are already included in this document, but still without a description.

## Batch 20 — Identification of the Last FUN_ Functions (2026-07-09)

Of the 206 `FUN_` functions remaining at that point, 202 could be identified and renamed in Ghidra by 6 subagents working in parallel. A subsequent full scan found no duplicate function names, neither before nor after applying the renamings (0 duplicates). The remaining 4 functions (`0x0800bcc0`, `0x0802da14`, `0x0802db54`, `0x0802dc50`) were classified as padding/stubs and deliberately left as `FUN_`. The full details on all 202 renamings can be found in the section "Batch 20" of `Control_FW_Function_Tracking.md`.

## Notes on the Data Basis

- **992** functions: description taken 1:1 from the existing docs (address *and* name match).
- **58** functions: description matched via the **name**, not the address — i.e. the old docs
  recorded a different address for this function name (address drift, see the memory entry on the topic).
- **165** functions: **no description found** (neither via address nor via name) — column
  stays empty. Mostly llhttp internals, further Quectel/CH395 helpers, and the 6 newly appeared FreeRTOS functions.
- **202** functions: Batch 20 (2026-07-09) — newly identified and named, description derived from decompilation by subagents.

**Re-audit 2026-07-14 (autonomous session):** Full address-drift check (all 1619 names compared against a live Ghidra dump
by script — 0 real drift cases, 0 missing live mappings; the previously feared "201 stale
addresses" do not exist in this file, since live-resolved addresses were already used when it was
built) as well as targeted re-verification of 218 functions in clusters never individually audited before
(Modbus/RS485, inverter/register/energy, CAN, BLE, config/EEPROM, OTA/flash — see the respective tables) against
fresh decompilation. Result: 30 faulty names corrected (address + old/new name see below in the
respective cluster tables, source column "Re-audit 2026-07-14"); among other things, a completely
misfiled 18-function block (`0x0804bd58`–`0x0804cc40`) was corrected from "BLE_GATT_*" to "CLI_*" — it
is a generic CLI/AT-command engine (reachable via CH395/Modbus TCP), not
BLE GATT handling. These functions still appear in the BLE section below (renamed, but no
section move made); **open item for the user:** whether a separate cluster
"CLI/AT-Command-Engine" should be created for them. After applying all renamings: full scan still confirms
0 duplicate function names (1619/1623 named, unchanged). Details and complete proposal lists (including
unapplied ones, e.g. `0x0802f2b4` and `0x08008000` — both deliberately left unchanged, see rationale in the
session log) in the project memory. **Update 2026-07-15: both completed** — `0x0802f2b4` renamed to
`BatteryParams_PowerFlowState_Get`, `0x08008000` identified as a Ghidra function-boundary artifact and removed from
Ghidra (details: VNSD_Control_FW_Analysis_app_1492_0702_142136.md §13.46).

**Re-audit 2026-07-14, part 2 (cluster "Hardware / HAL", 81 functions, never individually checked before):**
All 81 functions checked individually via decompilation (including callers/callees) against the STM32F1 standard peripheral
base address map (RCC=0x40021000, SPI1=0x40013000/APB2, SPI2=0x40003800/APB1, SPI3=0x40003C00/APB1,
USART1=0x40013800/APB2, I2C1=0x40005400, GPIOA–E=0x40010800–0x40011800, FLASH=0x40022000). Result: **13
clear naming errors found** (not applied so far, see proposal table in the session log/memory), among others
a block of four `ADC_*`/`SPI_Timer_*` functions that actually address SPI2 and I2C1 peripherals respectively (no ADC, no
SPI/timer registers), an RTC get/set swapped pair (RTC_GetTime↔RTC_GetDate, RTC_SetTime↔RTC_SetDate,
target address and caller semantics consistently confirm the swap), two functions (`RCC_EnablePeripheralClock`,
`STM32_RCC_Clock_Config`) that actually address FLASH and AFIO registers respectively instead of RCC, as well as two
bit-band alias cases (`GPIO_PulsePin`/`GPIO_WritePinValue`) that write RCC_BDCR.BDRST instead of a GPIO pin.
The feared I2C↔CAN mix-up in the I2C bit-bang/EEPROM_I2C cluster (21 functions) was
**not confirmed** — all base addresses (I2C1=0x40005400, GPIOB/GPIOC for bit-bang) match the names;
only an outdated caller hint ("CAN_SendWorkModeFrame" instead of the correct "I2C_SendWorkModeFrame")
in the description of `I2C_BitBang_WriteBytes` was corrected. In addition, 7 missing/wrong
descriptions corrected directly (GPIO_ConfigPinAsInput, RCC_AHBPeriphResetCmd, RCC_APB1PeriphResetCmd,
RCC_PeriphBitControl, RCC_GetFlagStatus, GPIOD_Pin9_Write, I2C_BitBang_WriteBytes — source column "Ghidra
(Re-audit 2026-07-14)"). Clarified: besides USART1 (0x40013800, correct), `USART_Init` also checks two
constants `0x40015000`/`0x40015400` that do **not** correspond to the STM32F103 addresses for USART2/3
(0x40004400/0x40004800), because the MCU is not an STM32F103 but a **Nations N32G45x**:
according to the official `n32g45x.h` (local, not in the repo) these are `UART6_BASE` (`APB2PERIPH_BASE`+0x5000) and `UART7_BASE`
(`APB2PERIPH_BASE`+0x5400), both APB2-clocked — consistent with the observed PCLK2 baud rate calculation. See
VNSD_Control_FW_Analysis_app_1492_0702_142136.md, section "MCU identification" (formerly §13.35, "not
conclusively clarified"). All renaming proposals see reply/memory of this
session — not yet applied in Ghidra (user decision pending).

---

## mbedTLS — MPI / Bignum (54)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08031cce` | mbedTLS_MPI_Add_Sub_Signed | MPI signed add/sub with sign logic | high | Docs |
| `0x08033ee4` | mbedTLS_MPI_Init_Static | MPI from static ROM buffer | high | Docs |
| `0x0803f16a` | mbedTLS_MPI_CT_CondAssign | Conditional MPI limb copy with mask | high | Docs |
| `0x080408a2` | mbedTLS_MPI_Int_Div_Int | 64-bit division with remainder, overflow check | high | Docs |
| `0x0804247c` | mbedTLS_MPI_Add_Abs | Limb addition with carry (real implementation) | high | Docs |
| `0x08042584` | mbedTLS_MPI_Add_Int | Construct MPI from signed integer  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x08042612` | mbedTLS_MPI_Add_MPI | *(no description — see notes above)* | - | — |
| `0x08042628` | mbedTLS_MPI_Bitlen | *(no description — see notes above)* | - | — |
| `0x08042660` | mbedTLS_MPI_Cmp_Abs | Leading-zero trim + MSB-first comparison | high | Docs |
| `0x080426f2` | mbedTLS_MPI_Cmp_Int | *(no description — see notes above)* | - | — |
| `0x08042726` | mbedTLS_MPI_Cmp_MPI | *(no description — see notes above)* | - | — |
| `0x080427da` | mbedTLS_MPI_Copy | *(no description — see notes above)* | - | — |
| `0x08042866` | mbedTLS_MPI_Div_MPI | *(no description — see notes above)* | - | — |
| `0x08042c2e` | mbedTLS_MPI_Exp_Mod | *(no description — see notes above)* | - | — |
| `0x080430e4` | mbedTLS_MPI_Fill_Random_Sized | Byte→limb conversion + grow  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0804313c` | mbedTLS_MPI_Free | *(no description — see notes above)* | - | — |
| `0x08043164` | mbedTLS_MPI_GCD | Stein's binary GCD algorithm | high | Docs |
| `0x080432cc` | mbedTLS_MPI_Get_Bit | Extract bit position from MPI | high | Docs |
| `0x080432f2` | mbedTLS_MPI_Mont_Compute_RR | R² mod N for Montgomery setup | high | Docs |
| `0x0804334e` | mbedTLS_MPI_Grow | *(no description — see notes above)* | - | — |
| `0x080433a4` | mbedTLS_MPI_Init | *(no description — see notes above)* | - | — |
| `0x080433b4` | mbedTLS_MPI_Inv_Mod | *(no description — see notes above)* | - | — |
| `0x080436f2` | mbedTLS_MPI_LSB | Count trailing-zero bits | high | Docs |
| `0x0804372a` | mbedTLS_MPI_Lset | *(no description — see notes above)* | - | — |
| `0x08043772` | mbedTLS_MPI_Lt_MPI_CT | **CT** less-than comparison (side-channel safe) | high | Docs |
| `0x0804381e` | mbedTLS_MPI_Mod_MPI | *(no description — see notes above)* | - | — |
| `0x080438ac` | mbedTLS_MPI_Mont_Mul | Montgomery multiplication (CT CondAssign) | high | Docs |
| `0x0804396a` | mbedTLS_MPI_Mont_Init | mm = -N⁻¹ mod 2³² (Newton iteration) | high | Docs |
| `0x08043994` | mbedTLS_MPI_Mul_Int | *(no description — see notes above)* | - | — |
| `0x08043a08` | mbedTLS_MPI_Mul_Mod | *(no description — see notes above)* | - | — |
| `0x08043a4e` | mbedTLS_MPI_Mul_MPI | Real schoolbook multiplication (8 callers) — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08043b68` | mbedTLS_MPI_Random | Rejection sampling [min, N), max 250 retries | high | Docs |
| `0x08043c7c` | mbedTLS_MPI_Read_Binary | Big-endian bytes → MPI (6 callers) | high | Docs |
| `0x08043cdc` | mbedTLS_MPI_Grow_Clear | Resize + zero (data not preserved) | high | Docs |
| `0x08043d14` | mbedTLS_MPI_Safe_Cond_Assign | **CT** conditional copy (ECP side-channel) | high | Docs |
| `0x08043d82` | mbedTLS_MPI_Shift_Left | Left shift: word+bit with carry | high | Docs |
| `0x08043e96` | mbedTLS_MPI_Shift_R | *(no description — see notes above)* | - | — |
| `0x08043f40` | mbedTLS_MPI_Shrink_Realloc | Shrink allocation (10001-limb limit) | high | Docs |
| `0x08043fc2` | mbedTLS_MPI_Size | Byte count: (bitlen+7)>>3 | high | Docs |
| `0x08043fd2` | mbedTLS_MPI_Sub_Abs_Core | Unsigned subtraction with borrow | high | Docs |
| `0x080440ba` | mbedTLS_MPI_Sub_Int | Subtract integer from MPI | medium | Docs |
| `0x080440f6` | mbedTLS_MPI_Sub_Mod | *(no description — see notes above)* | - | — |
| `0x0804414e` | mbedTLS_MPI_Sub_MPI | *(no description — see notes above)* | - | — |
| `0x08044166` | mbedTLS_MPI_Write_Binary | MPI → big-endian bytes (5 callers) | high | Docs |
| `0x080441e8` | mbedTLS_MPI_Write_Binary_LE | MPI → little-endian bytes | high | Docs |
| `0x08044252` | mbedTLS_MPI_Zeroize | Zeroize limb array (security) | high | Docs |
| `0x08048794` | mbedTLS_MPI_Reverse_Words | In-place word array reversal (endianness) | high | Docs |
| `0x080487cc` | mbedTLS_MPI_Fill_Random | Fill limbs via RNG, padding, reversal | high | Docs |
| `0x0804883a` | mbedTLS_MPI_Mont_Mul_One | Construct MPI=1 → Mont_Mul (Montgomery identity) | high | Docs |
| `0x08048866` | mbedTLS_MPI_Mul_Helper | *(no description — see notes above)* | - | — |
| `0x08048e92` | mbedTLS_MPI_CT_Select | **Constant-time** window table lookup (side-channel resistant) | high | Docs |
| `0x08048edc` | mbedTLS_MPI_Uint_Abs | Simple absolute value | high | Docs |
| `0x08048eea` | mbedTLS_MPI_Sub_Helper | Multi-precision subtraction with borrow propagation | high | Docs |
| `0x0804baec` | mbedTLS_MPI_Montgomery_Convert | Montgomery domain conversion, compute RR  — **corrected 2026-07-09**, see Batch 18 | high | Docs |

## mbedTLS — ECP / ECDH (67)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08033200` | mbedTLS_ECP_Write_Reduced_Coordinate | Point → binary (compressed/uncompressed)  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08033298` | mbedTLS_ECDH_Compute_Shared_Restartable | Modular reduction with temp MPI  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0803330c` | mbedTLS_ECP_Group_Free | Free ECP group struct | high | Docs |
| `0x08033338` | mbedTLS_ECDH_Gen_Public_Restartable | Normalize Jacobian (Z^-1)  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08033394` | mbedTLS_ECP_Group_Init | Initialize ECP group struct | high | Docs |
| `0x080333c0` | mbedTLS_ECDH_Make_Public | Binary → ECP point  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08033418` | mbedTLS_ECP_TLS_Read_Point_Alt | Write TLS wire-format point  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08033434` | mbedTLS_ECP_Group_Load | Load curve parameters | high | Docs |
| `0x080334dc` | mbedTLS_ECP_Add_Mixed | *(no description — see notes above)* | - | — |
| `0x080337d0` | mbedTLS_ECP_Check_Point_On_Curve | Verify y²=x³+ax+b mod P | high | Docs |
| `0x080338fc` | mbedTLS_ECP_Comb_Recode_Core | Comb method: extract scalar bits | high | Docs |
| `0x0803398c` | mbedTLS_ECP_Comb_Recode_Scalar | Prepare scalar for comb multiplication | high | Docs |
| `0x08033a40` | mbedTLS_ECP_Point_Zeroize | Set point = infinity  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x08033a54` | mbedTLS_ECP_Double_Jac | *(no description — see notes above)* | - | — |
| `0x08033d52` | mbedTLS_ECP_RNG_Wrapper | MPI read wrapper for point  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x08033d68` | mbedTLS_ECP_DRBG_Seed_From_Scalar | Private key from binary (max 32B)  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08033db8` | mbedTLS_ECP_Group_Load_RawParams | *(no description — see notes above)* | - | — |
| `0x08033e44` | mbedTLS_ECP_Modp | Modular reduction (fast path or MPI_Mod) | high | Docs |
| `0x08033ef0` | mbedTLS_ECP_Mul_Restartable | Main scalar multiplication (comb method) | high | Docs |
| `0x08034074` | mbedTLS_ECP_Mul_Comb | Comb method orchestrator | high | Docs |
| `0x08034126` | mbedTLS_ECP_Mul_Comb_Core | Inner comb loop (double+add) | high | Docs |
| `0x080341f0` | mbedTLS_ECP_Normalize_Jac | *(no description — see notes above)* | - | — |
| `0x080342b4` | mbedTLS_ECP_Normalize_Jac_Many | Batch normalization (Montgomery's trick) | high | Docs |
| `0x080344d4` | mbedTLS_ECP_PickWindowSize | Window 4 or 5 by bit length | high | Docs |
| `0x080344fc` | mbedTLS_ECP_PrecomputeComb | Precompute comb table | high | Docs |
| `0x08034640` | mbedTLS_ECP_RandomizeJac | Compute Z^-1, X/Z², Y/Z³  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08034718` | mbedTLS_ECP_SafeInvertJac | Conditionally negate Y (constant-time) | high | Docs |
| `0x0803477c` | mbedTLS_ECP_SelectComb | *(no description — see notes above)* | - | — |
| `0x0803f638` | mbedTLS_ECP_TLS_Write_Point | TLS point serialization (named curve check) | high | Docs |
| `0x0803f67c` | mbedTLS_ECP_Mod_Reduce_NoRestart | Wrapper → ModReduce(restart=0) | high | Docs |
| `0x0803f6b4` | mbedTLS_ECP_KeyInfo_Free | KeyInfo wrapper free (Marstek-specific, 0xe0B) | high | Docs |
| `0x0803f6da` | mbedTLS_ECP_Normalize_Jac_NoRestart | Wrapper → NormalizeJac(restart=0) | high | Docs |
| `0x0803f710` | mbedTLS_ECP_KeyInfo_Init | KeyInfo memset(0, 0xe0) | high | Docs |
| `0x0803f728` | mbedTLS_ECP_KeyInfo_Read_Point | Read binary point via KeyInfo wrapper | high | Docs |
| `0x0803f770` | mbedTLS_ECP_TLS_Read_GroupAndPoint | Serialize TLS group ID + point  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0803f7cc` | mbedTLS_ECP_KeyInfo_LoadGroup | Load curve ID → group + set flag | high | Docs |
| `0x0803f7f4` | mbedTLS_ECP_Check_PrivKey | Validate key >= 1 && key < N | high | Docs |
| `0x0803f83c` | mbedTLS_ECP_Check_PubPriv | **0 callers** — pub/priv consistency check | high | Docs |
| `0x0803f900` | mbedTLS_ECP_Check_PubKey | Z==1, group valid, point on curve | high | Docs |
| `0x0803f940` | mbedTLS_ECP_Point_Copy | 3 MPI copies (X, Y, Z) | high | Docs |
| `0x0803f996` | mbedTLS_ECP_CurveInfo_FromGrpId | Table lookup by group ID | high | Docs |
| `0x0803f9b8` | mbedTLS_ECP_CurveInfo_FromTlsId | Table lookup by TLS named curve ID | high | Docs |
| `0x0803f9dc` | mbedTLS_ECP_CurveInfo_GetList | 4B stub → static table | high | Docs |
| `0x0803f9e4` | mbedTLS_ECP_Gen_Privkey | Mod reduction dispatcher (short Weierstrass)  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0803fa20` | mbedTLS_ECP_Gen_Privkey_ShortWeierstrass | mpi_mod_mpi with error remapping  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0803fa50` | mbedTLS_ECP_GetType | Curve type: 0=none, 1=SW, 2=Montgomery | high | Docs |
| `0x0803fa66` | mbedTLS_ECP_Group_SetFromInfo | Thin wrapper → Group_SetById | high | Docs |
| `0x0803fa7e` | mbedTLS_ECP_Group_FreeFields | Free all MPIs + T-table, 0x7C zeroize | high | Docs |
| `0x0803fae2` | mbedTLS_ECP_Group_InitFields | Initialize all group fields | high | Docs |
| `0x0803fb2c` | mbedTLS_ECP_Group_SetById | **Only SECP256R1 (ID=3)** — no fallback | high | Docs |
| `0x0803fb98` | mbedTLS_ECP_BuildSupportedCurvesList | Lazy-init cached curve list | high | Docs |
| `0x0803fbd4` | mbedTLS_ECP_Point_IsZero | Z==0 → point at infinity | high | Docs |
| `0x0803fbf0` | mbedTLS_ECP_Keypair_Free | Free group + MPI d + point Q | high | Docs |
| `0x0803fc12` | mbedTLS_ECP_Keypair_Init | Keypair init (0 callers → function pointer) | high | Docs |
| `0x0803fc32` | mbedTLS_ECP_Mul | Wrapper → Mul_Checked(restart=0) | high | Docs |
| `0x0803fc6c` | mbedTLS_ECP_Mul_Checked | Validate curve type + point, → Mul_Restartable | high | Docs |
| `0x0803fcf4` | mbedTLS_ECP_Point_Free | Free 3 MPIs (X, Y, Z), 7 callers | high | Docs |
| `0x0803fd16` | mbedTLS_ECP_Point_Init | Initialize 3 MPIs (X, Y, Z), 7 callers | high | Docs |
| `0x0803fd38` | mbedTLS_ECP_Point_ReadBinary | 0x00=infinity, 0x04=uncompressed | high | Docs |
| `0x0803fdf0` | mbedTLS_ECP_Point_WriteBinary | Uncompressed (0x04) + compressed (0x02/0x03) | high | Docs |
| `0x0803fee4` | mbedTLS_ECP_Point_SetZero | X=1, Y=1, Z=0 (Jacobian infinity) | high | Docs |
| `0x0803ff30` | mbedTLS_ECP_TLS_ReadGroup | Read TLS NamedCurve (tag 0x03, 2B ID) | high | Docs |
| `0x0803ff94` | mbedTLS_ECP_TLS_Read_Point | Length-prefixed point from TLS buffer | high | Docs |
| `0x0803ffec` | mbedTLS_ECP_TLS_Write_Point_Inner | Write point + length prefix into TLS buffer | high | Docs |
| `0x080425c0` | mbedTLS_ECP_Mod_Reduce | MPI grow + subtraction in loop until < P | high | Docs |
| `0x08043e4c` | mbedTLS_ECP_Mod_Reduce_ShiftSub | *(no description — see notes above)* | - | — |
| `0x0804f7dc` | mbedTLS_ECP_SupportedCurves_Write | Writes supported-curves extension into TLS ClientHello | high | Docs (name match) |

## mbedTLS — RSA (36)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x0803f1b2` | mbedTLS_RSA_PKCS1v15_Unpad | **SECURITY** CT PKCS#1 v1.5 unpadding (Bleichenbacher protection) | high | Docs |
| `0x08044bb8` | mbedTLS_RSA_Alt_SetCallbacks | Set alternative RSA callbacks | medium | Docs |
| `0x08044bd0` | mbedTLS_RSA_Alt_InitOnce | One-shot guard → SetCallbacks | medium | Docs |
| `0x08044c14` | mbedTLS_RSA_CheckPrivkey | Validate P, Q, D, DP, DQ, QP | high | Docs |
| `0x08044c84` | mbedTLS_RSA_CheckPubPriv | Compare N and E of pub/priv | high | Docs |
| `0x08044cd0` | mbedTLS_RSA_CheckPubkey | N>127bit, E odd, E>1, E<N | high | Docs |
| `0x08044d2c` | mbedTLS_RSA_Complete | Complete key from partial components | high | Docs |
| `0x08044f78` | mbedTLS_RSA_DeduceCrt | CRT: compute DP, DQ, QP | high | Docs |
| `0x08045018` | mbedTLS_RSA_DeducePrimes | Reconstruct P, Q from N, E, D (540B) | high | Docs |
| `0x0804526c` | mbedTLS_RSA_DeduceD | D = E⁻¹ mod LCM(P-1, Q-1) | high | Docs |
| `0x08045360` | mbedTLS_RSA_Export | Export N/E/D/P/Q via MPI_Copy | high | Docs |
| `0x08045440` | mbedTLS_RSA_Free | Free 13 MPI fields (0xAC context) | high | Docs |
| `0x080454b4` | mbedTLS_RSA_Get_Len | Key length in bytes from ctx+4 | high | Docs |
| `0x080454bc` | mbedTLS_RSA_Import | Import MPI values into RSA context | high | Docs |
| `0x08045584` | mbedTLS_RSA_Import_Raw | Raw binary → RSA context | high | Docs |
| `0x08045680` | mbedTLS_RSA_Init | Zero the 0xAC context (0 callers) | high | Docs |
| `0x080456a4` | mbedTLS_RSA_PKCS1_Encrypt | Padding mode check → RSAES dispatch | high | Docs |
| `0x080456f0` | mbedTLS_RSA_PKCS1_Decrypt | Padding mode check → decrypt dispatch | high | Docs |
| `0x08045738` | mbedTLS_RSA_PKCS1v15_Sign | Padding mode check → RSASSA dispatch | high | Docs |
| `0x08045780` | mbedTLS_RSA_PKCS1v15_Verify | **0 callers** — dead code or pointer table | high | Docs |
| `0x080457c8` | mbedTLS_RSA_Private | *(no description — see notes above)* | - | — |
| `0x08045b18` | mbedTLS_RSA_Public | RSA public-key op (MPI exp mod), 4 callers | high | Docs |
| `0x08045bf8` | mbedTLS_RSA_RSAES_PKCS1v15_Decrypt | Decrypt + unpad + zeroize | high | Docs |
| `0x08045c98` | mbedTLS_RSA_RSAES_PKCS1v15_Encrypt | Padding (0x02 random / 0x01 0xFF) + RSA op | high | Docs |
| `0x08045dcc` | mbedTLS_RSA_RSASSA_PKCS1v15_Sign | **Sign with blinding + verify-after-sign** (fault-attack protection) | high | Docs |
| `0x08045edc` | mbedTLS_RSA_RSASSA_PKCS1v15_Verify | Verify: encode + RSA public + CT compare | high | Docs |
| `0x08045fbc` | mbedTLS_RSA_SetPadding | Padding mode (+0xa4) + hash ID (+0xa8) | high | Docs |
| `0x08045fd0` | mbedTLS_RSA_CheckCRT | Check DP, DQ, QP consistency | high | Docs |
| `0x08046120` | mbedTLS_RSA_ValidateParams | N==P*Q, D*E mod LCM check (452B) | high | Docs |
| `0x0804b610` | mbedTLS_RSA_Check_Context | **Max 2048-bit** RSA (N ≤ 256B), 5 callers — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0804b6dc` | mbedTLS_RSA_CheckPubPriv_Wrapper | Thin wrapper, 0 callers (fn pointer) | high | Docs |
| `0x0804b7a8` | mbedTLS_RSA_Context_Free | RSA_Free + Platform_Free, 0 callers (fn pointer) | high | Docs |
| `0x0804b7ba` | mbedTLS_RSA_Get_Bitlen | N export → bitlen, fallback Get_Len*8 | high | Docs |
| `0x0804b7fc` | mbedTLS_RSA_Prepare_Blinding | Modular reduction with retry loop (max 10x)  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0804b96c` | mbedTLS_RSA_PKCS1v15_Encode_DigestInfo | PKCS#1 v1.5 signature encoding (00 01 FF..FF 00 DI hash) | high | Docs |
| `0x0804bab0` | mbedTLS_RSA_Sign_PKCS1v15_Wrapper | Wrapper → Sign(mode=1, private), 0 callers | high | Docs |

## mbedTLS — SSL/TLS Handshake (137)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080128c4` | mbedTLS_SSL_Recv_WithTimeout | SSL receive with timeout | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08012924` | mbedTLS_SSL_Send_WithTimeout | SSL send with timeout | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08018364` | mbedTLS_SSL_Connection_Init | *(no description — see notes above)* | - | — |
| `0x08032e14` | SSL_RecordLen_ToHighWord | 6B shift helper for SSL record | medium | Docs |
| `0x0803658e` | mbedTLS_CheckCompressionMethod | Check param != 0 (TLS ServerHello) | high | Docs |
| `0x0804664e` | mbedTLS_SSL_CheckSigHash | Check allowed sig hashes (config+0x58) | high | Docs |
| `0x0804667a` | mbedTLS_SSL_CheckCurve | Check allowed curves (config+0x54) | high | Docs |
| `0x080466a6` | mbedTLS_SSL_CheckTimer | Fetch timer callback | high | Docs |
| `0x080466c8` | mbedTLS_SSL_BoundsCheck | Buffer bounds check (11 callers) | high | Docs |
| `0x080466dc` | mbedTLS_Ciphersuite_Uses_SrvCert | Key exchange → server cert needed? | high | Docs |
| `0x08046708` | mbedTLS_SSL_Read_RecordLen | *(no description — see notes above)* | - | — |
| `0x08046728` | mbedTLS_Ciphersuite_Uses_EC | Key exchange → EC needed? | high | Docs |
| `0x08046752` | mbedTLS_Ciphersuite_CertReq_Allowed | CertRequest allowed? | high | Docs |
| `0x0804677e` | mbedTLS_SSL_Conf_Authmode | Set authmode (+6) | high | Docs |
| `0x08046782` | mbedTLS_SSL_Conf_CA_Chain | Set CA chain + CRL (+0x4c/+0x50) | high | Docs |
| `0x08046788` | mbedTLS_SSL_Conf_Ciphersuites | 4× ciphersuite list (+0x10-+0x1c) | high | Docs |
| `0x08046792` | mbedTLS_SSL_Conf_Curves | Curve list (+0x58) | medium | Docs |
| `0x08046796` | mbedTLS_SSL_Conf_Endpoint | Client/server (+4) | medium | Docs |
| `0x0804679a` | mbedTLS_SSL_Conf_MaxVersion | Max TLS version (+0,+1) | high | Docs |
| `0x080467a0` | mbedTLS_SSL_Conf_MinVersion | Min TLS version (+2,+3) | high | Docs |
| `0x080467a6` | mbedTLS_SSL_Conf_OwnCert | Register cert+key | medium | Docs |
| `0x080467bc` | mbedTLS_SSL_Conf_Rng | Set RNG callback (+0x28/+0x2c) | high | Docs |
| `0x080467c2` | mbedTLS_SSL_Conf_SigHashes | Sig-hash list (+0x54) | medium | Docs |
| `0x080467c6` | mbedTLS_SSL_Conf_Transport | Authmode (+5)  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x080467cc` | mbedTLS_SSL_Config_Defaults | **TLS 1.2 forced** (all version bytes = 3) | high | Docs |
| `0x0804685c` | mbedTLS_SSL_Config_Free | Zeroize 0x5c config | high | Docs |
| `0x08046870` | mbedTLS_SSL_Config_Init | memset(0, 0x5c) | high | Docs |
| `0x08046880` | mbedTLS_SSL_Decrypt_Buf | **AEAD-only** (GCM/CCM/ChaCha20, no CBC!) | high | Docs |
| `0x080469d4` | mbedTLS_SSL_Derive_Keys | Master secret + transform setup | high | Docs |
| `0x08046a90` | mbedTLS_SSL_Encrypt_Buf | AEAD encrypt + explicit IV | high | Docs |
| `0x08046c14` | mbedTLS_SSL_Fetch_Input | f_recv loop, max 0x182f bytes | high | Docs |
| `0x08046ce0` | mbedTLS_SSL_Flush_Output | f_send loop, out_left tracking | high | Docs |
| `0x08046d78` | mbedTLS_SSL_Free | 5 fields + 0xC4 zeroize  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x08046e20` | mbedTLS_SSL_FreeHostname | Hostname zeroize+free | high | Docs |
| `0x08046e5c` | mbedTLS_SSL_GetKeyExchangeType | Ciphersuite → KE type (1/2/4) | high | Docs |
| `0x08046e8c` | mbedTLS_SSL_GetHostname | Hostname from +0xbc | high | Docs |
| `0x08046ea8` | mbedTLS_SSL_CalcVerifyHash | Compute handshake hash | high | Docs |
| `0x08046f50` | mbedTLS_SSL_GetOutputMaxFragLen | Max fragment = 0x800 (2048B) | high | Docs |
| `0x08046f6a` | mbedTLS_SSL_GetMaxFragLenInternal | Minimum over 3 transforms | high | Docs |
| `0x08046fb0` | mbedTLS_SSL_CheckRecordType | 0x16=HS, 0x14=CCS, 0x15=Alert | high | Docs |
| `0x08047060` | mbedTLS_SSL_Handshake | Loop until state 0x10 (OVER) | high | Docs |
| `0x08047098` | mbedTLS_SSL_HandshakeClientStep | **16-case client state machine** | high | Docs |
| `0x08047184` | mbedTLS_SSL_HandshakeParamsFree | ECP + PK + 0x1DC zeroize | high | Docs |
| `0x080471bc` | ssl_HandshakeStepDispatch | Context validation + dispatch | high | Docs |
| `0x080471e4` | ssl_HandshakeWrapup | Rotate session, SNI callback | high | Docs |
| `0x08047238` | ssl_HandshakeWrapupFreeTransform | Free transform, rotate buffers | high | Docs |
| `0x0804726e` | mbedtls_ssl_check_pending | Buffered data? (offset 0xBC) | high | Docs |
| `0x0804727e` | ssl_HashFromMdAlg | SHA224→3, SHA256→4 | high | Docs |
| `0x08047296` | ssl_HandshakeParamsInit | memset(0, 0xC4) | high | Docs |
| `0x080472a4` | ssl_BuildSupportedCipherList | Lazy-init filtered cipher list | medium | Docs |
| `0x080472f4` | ssl_MdAlgFromHash | 3→SHA224, 4→SHA256 | high | Docs |
| `0x0804730c` | ssl_ApplyEncryptThenMac | Set ETM extension flag | medium | Docs |
| `0x08047328` | ssl_GetRemainingPayloadLen | Remaining space in output buffer | high | Docs |
| `0x08047336` | ssl_GetOwnPrivateKey | PK key from session or config | high | Docs |
| `0x0804735c` | mbedTLS_SSL_ProcessCertificateKeyExchange | Parse server cert + key exchange | high | Docs |
| `0x08047450` | mbedTLS_SSL_ParseChangeCipherSpec | Validate CCS (0x14, len=1) | high | Docs |
| `0x080474b4` | mbedTLS_SSL_ParseFinishedMessage | **CT compare** 12B verify data | high | Docs |
| `0x08047588` | mbedTLS_SSL_IsDTLS | Transport==1? | high | Docs |
| `0x08047594` | mbedTLS_SSL_ValidateHandshakeHeader | Length check (>3, total_len+4 <= record) | high | Docs |
| `0x080475dc` | mbedTLS_SSL_Read | Read app data + alert/HS dispatch | high | Docs |
| `0x08047754` | mbedTLS_SSL_ReadRecord | Core record reader (8 callers) | high | Docs |
| `0x080477ea` | mbedTLS_SSL_ReadVersion | 2 bytes → major/minor | high | Docs |
| `0x080477f6` | mbedTLS_SSL_UpdateRecordPointers | In/out buffer +8 | medium | Docs |
| `0x0804781c` | mbedTLS_SSL_SendAlertMessage | Send alert (15 callers!) | high | Docs |
| `0x08047880` | mbedTLS_SSL_SessionFree | Cipher + 0x70 zeroize  — **corrected 2026-07-09**, see Batch 18 | high | Docs |
| `0x0804789a` | mbedTLS_SSL_HandshakeParams_Init | memset(0, 0x70) | high | Docs |
| `0x080478a8` | mbedTLS_SSL_Set_Transport_Params | Set 4 transport fields | high | Docs |
| `0x080478b8` | mbedTLS_SSL_Set_Hostname | Set hostname (max 256 characters) | high | Docs |
| `0x08047920` | mbedTLS_SSL_Debug_Callback_Invoke | Debug callback (+0x50) | high | Docs |
| `0x08047940` | mbedTLS_SSL_Setup | Alloc in/out buffer (0x182f/0x82d) | high | Docs |
| `0x080479dc` | mbedTLS_SSL_PK_Can_Do_RSA | PK_Can_Do(ctx, RSA) wrapper | high | Docs |
| `0x080479f2` | mbedTLS_SSL_Verify_Flags_Init | Set 2 bytes equal | medium | Docs |
| `0x080479f8` | mbedTLS_SSL_Transform_Free | 2× cipher free + 0xa8 zeroize | high | Docs |
| `0x08047a1c` | mbedTLS_SSL_Transform_Init | 0xa8 memset + 2× cipher init | high | Docs |
| `0x08047a3a` | mbedTLS_SSL_Flight_Retransmit | DTLS flight retransmit | high | Docs |
| `0x08047a58` | mbedTLS_SSL_Reset_Record_Pointers | Header/content/payload offsets | high | Docs |
| `0x08047a70` | mbedTLS_SSL_SetOutMsgPointers | Compute output buffer offsets | high | Docs |
| `0x08047aa4` | mbedTLS_SSL_Write | Write app data (state 0x10 check) | high | Docs |
| `0x08047af4` | mbedTLS_SSL_WriteClientCertificate | Serialize cert chain (type 0x0b) | high | Docs |
| `0x08047c04` | mbedTLS_SSL_WriteChangeCipherSpec | Send CCS (0x14, 0x01) | high | Docs |
| `0x08047c42` | mbedTLS_SSL_WriteFinished | Send Finished (type 0x14) | high | Docs |
| `0x08047cd8` | mbedTLS_SSL_WriteRecord | Send record (6 callers) | high | Docs |
| `0x08047d84` | mbedTLS_SSL_FlushOutput | Version + encrypt + MAC + send | high | Docs |
| `0x08047f14` | mbedTLS_SSL_WriteVersion | major/minor → 2 bytes | high | Docs |
| `0x0804d21c` | SSL_PrintCertInfo | *(no description — see notes above)* | - | — |
| `0x0804d25c` | SSL_PrintCertVersion | *(no description — see notes above)* | - | — |
| `0x0804da38` | mbedTLS_KeyCert_AppendToList | *(no description — see notes above)* | - | — |
| `0x0804da7c` | mbedTLS_BuildAeadNonce | *(no description — see notes above)* | - | — |
| `0x0804dabc` | mbedTLS_ComputeFinishedVerifyData | *(no description — see notes above)* | - | — |
| `0x0804db54` | mbedTLS_CalcHandshakeHash | *(no description — see notes above)* | - | — |
| `0x0804db94` | mbedTLS_ValidateContentType | *(no description — see notes above)* | - | — |
| `0x0804dbb4` | mbedTLS_SSL_CheckCurveAndSigHash | *(no description — see notes above)* | - | — |
| `0x0804dbf0` | mbedTLS_SSL_FreeSessionBuffer | *(no description — see notes above)* | - | — |
| `0x0804dc0c` | mbedTLS_SSL_DeriveMasterSecret | *(no description — see notes above)* | - | — |
| `0x0804dc80` | mbedTLS_SSL_UpdateRecordBuffer | *(no description — see notes above)* | - | — |
| `0x0804dce0` | mbedTLS_SSL_BuildAeadNonce | *(no description — see notes above)* | - | — |
| `0x0804dd16` | mbedTLS_SSL_GeneratePremasterRandom | *(no description — see notes above)* | - | — |
| `0x0804dd52` | mbedTLS_SSL_GetHandshakeLength | *(no description — see notes above)* | - | — |
| `0x0804dd6a` | mbedTLS_SSL_ReadRecordLayer | *(no description — see notes above)* | - | — |
| `0x0804de04` | mbedTLS_SSL_AllocSessionStructs | *(no description — see notes above)* | - | — |
| `0x0804de9c` | mbedTLS_SSL_HandshakeInit | *(no description — see notes above)* | - | — |
| `0x0804dee4` | mbedTLS_SSL_FreeLinkedList | *(no description — see notes above)* | - | — |
| `0x0804defc` | mbedTLS_SSL_MflCodeToLength | *(no description — see notes above)* | - | — |
| `0x0804df30` | mbedTLS_SSL_ParseCertificateChain | *(no description — see notes above)* | - | — |
| `0x0804e08c` | mbedTLS_SSL_CrtReqNotAllowed | *(no description — see notes above)* | - | — |
| `0x0804e0a8` | mbedTLS_TLS_CertificateRequest_Parse | *(no description — see notes above)* | - | — |
| `0x0804e210` | mbedTLS_SSL_VerifyCertificateChain | *(no description — see notes above)* | - | — |
| `0x0804e3fc` | mbedTLS_SSL_ParseMaxFragLenExt | *(no description — see notes above)* | - | — |
| `0x0804e434` | mbedTLS_TLS_ServerHello_ParseHeader | *(no description — see notes above)* | - | — |
| `0x0804e4f8` | mbedTLS_SSL_ParseExtendedMsExt | *(no description — see notes above)* | - | — |
| `0x0804e528` | mbedTLS_SSL_ReadServerEcdhParams | *(no description — see notes above)* | - | — |
| `0x0804e56c` | mbedTLS_TLS_ServerHello_Parse | *(no description — see notes above)* | - | — |
| `0x0804e95c` | mbedTLS_TLS_ServerHelloDone_Parse | *(no description — see notes above)* | - | — |
| `0x0804e9c8` | mbedTLS_TLS_ServerKeyExchange_Parse | *(no description — see notes above)* | - | — |
| `0x0804ec18` | ssl_ParseSignHashAlg | *(no description — see notes above)* | - | — |
| `0x0804ec9c` | ssl_ParseEcPointFormatsExt | *(no description — see notes above)* | - | — |
| `0x0804ed04` | ssl_PopulateCipherTransform | *(no description — see notes above)* | - | — |
| `0x0804eee0` | ssl_DecryptVerifyRecord | *(no description — see notes above)* | - | — |
| `0x0804efb0` | ssl_IsHandshakeActive | *(no description — see notes above)* | - | — |
| `0x0804efc0` | ssl_CalcCertificateHash | *(no description — see notes above)* | - | — |
| `0x0804f024` | ssl_ParsePeerPublicKey | *(no description — see notes above)* | - | — |
| `0x0804f05c` | ssl_SetPrfFunction | *(no description — see notes above)* | - | — |
| `0x0804f088` | ssl_TransformUsesExplicitIv | *(no description — see notes above)* | - | — |
| `0x0804f09a` | ssl_GetExplicitIvLen | *(no description — see notes above)* | - | — |
| `0x0804f0b0` | ssl_UpdateHandshakeHash | *(no description — see notes above)* | - | — |
| `0x0804f0c8` | mbedTLS_SSL_UpdateChecksum_SHA256 | Update handshake checksum with SHA-256 | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f0e0` | mbedTLS_SSL_Check_ECDH_Param_Bitlen | *(no description — see notes above)* | - | — |
| `0x0804f100` | mbedTLS_SSL_Write_CertificateVerify | Write CertificateVerify message | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f284` | mbedTLS_SSL_Parse_ServerKex | *(no description — see notes above)* | - | — |
| `0x0804f574` | mbedTLS_SSL_Write_ClientKeyExchange | Write ClientKeyExchange message | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f644` | mbedTLS_SSL_Write_MaxFragmentLengthExt | Write max-fragment-length extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f6a4` | mbedTLS_SSL_Write_Real | Actual writing of the SSL payload data | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f714` | mbedTLS_SSL_Write_SignatureAlgorithmsExt | Write signature-algorithms extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f8b4` | mbedTLS_SSL_Write_SupportedPointFormatsExt | Write supported-point-formats extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f910` | mbedTLS_Util_ReallocViaCallbacks | Reallocate memory via configured callback functions | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804f948` | mbedTLS_SSL_TLS_PRF_Generic | Generic TLS PRF implementation | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0804fabc` | mbedTLS_SSL_TLS_PRF_SHA256 | TLS PRF with SHA-256 | Batch 20 | Ghidra (Batch 20, 2026-07-09) |

## mbedTLS — X.509 / ASN.1 / PEM / OID / PK (102)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08032048` | mbedTLS_ASN1_Get_MPI | Read ASN.1 big integer (8 callers in RSA key parsing) | very high | Docs |
| `0x0803207c` | mbedTLS_ASN1_Sequence_Append | 16B linked-list nodes, ALLOC_FAILED=-106 | high | Docs |
| `0x080320ba` | mbedTLS_ASN1_Get_TaggedInt_Core | *(no description — see notes above)* | - | — |
| `0x0803e728` | mbedTLS_ASN1_Get_Alg | SEQUENCE → OID + optional parameters | high | Docs |
| `0x0803e7da` | mbedTLS_ASN1_Get_Bitstring | Parse BIT STRING (tag 0x03), extract unused bits | high | Docs |
| `0x0803e83e` | mbedTLS_ASN1_Get_Bitstring_Null | BIT STRING with 0 unused bits (byte-aligned) | high | Docs |
| `0x0803e884` | mbedTLS_ASN1_Get_Bool | Parse BOOLEAN (tag 0x01), exactly 1 byte | high | Docs |
| `0x0803e8c8` | mbedTLS_ASN1_Get_Int | Wrapper: Get_Tag with tag=2 (INTEGER) | high | Docs |
| `0x0803e8de` | mbedTLS_ASN1_Get_Len | Core DER length decoder (short/long form, 1-4 bytes) | high | Docs |
| `0x0803e9dc` | mbedTLS_ASN1_Get_Tagged_Data | Read INTEGER tag + raw bytes for MPI import | medium | Docs |
| `0x0803ea14` | mbedTLS_ASN1_Get_Sequence_Of | Traverse SEQUENCE with tag filter (e.g. OID) | high | Docs |
| `0x0803ea50` | mbedTLS_ASN1_Get_Tag | Core tag check + Get_Len (25 callers) | high | Docs |
| `0x0803ea88` | mbedTLS_ASN1_Traverse_Sequence_Of | Iterate SEQUENCE elements with callback | high | Docs |
| `0x08044262` | mbedTLS_OID_Get_Md_Alg | OID → md_type lookup | medium | Docs |
| `0x08044280` | mbedTLS_OID_Get_OID_By_Md | Reverse lookup: md_type → OID | medium | Docs |
| `0x080442b0` | mbedTLS_OID_Get_Pk_Alg | OID → pk_type lookup | medium | Docs |
| `0x080442ce` | mbedTLS_OID_Get_Sig_Alg | OID → md_type + pk_type | high | Docs |
| `0x080442f6` | mbedTLS_OID_Get_Ec_Grp | OID → ecp_group_id | medium | Docs |
| `0x08044314` | mbedTLS_PEM_Free | Free PEM context + zeroize | high | Docs |
| `0x0804433a` | mbedTLS_PEM_Init | Zero PEM context (12B) | high | Docs |
| `0x08044344` | mbedTLS_PEM_Read_Buffer | Parse PEM, Base64 decode. **Encrypted keys not supported** | high | Docs |
| `0x080444d8` | mbedTLS_PK_Can_Do | Vtable 0x0c: check algorithm support | high | Docs |
| `0x080444f4` | mbedTLS_PK_Check_Pair | Vtable 0x20: pub/priv key match | high | Docs |
| `0x08044550` | mbedTLS_PK_EC | EC context accessor (type 2/3/4) | high | Docs |
| `0x08044572` | mbedTLS_PK_Free | Vtable ctx_free + 8B zeroize (7 callers) | high | Docs |
| `0x08044592` | mbedTLS_PK_Get_Bitlen | Vtable 0x08: key size in bits | high | Docs |
| `0x080445aa` | mbedTLS_PK_Get_Type | pk_type enum from pk_info[0] | high | Docs |
| `0x080445bc` | mbedTLS_PK_Info_From_Type | Type→pk_info: only RSA(1), ECKEY(2), ECKEY_DH(3) | high | Docs |
| `0x080445e8` | mbedTLS_PK_Init | pk_info=0, pk_ctx=0 | high | Docs |
| `0x080445f4` | mbedTLS_PK_Parse_Key | Parse private key (RSA/EC PEM, PKCS8, DER) | high | Docs |
| `0x080448b4` | mbedTLS_PK_Parse_SubPubKey | Parse SubjectPublicKeyInfo ASN.1 | high | Docs |
| `0x08044a30` | mbedTLS_PK_RSA | RSA context accessor (type==1) | high | Docs |
| `0x08044a44` | mbedTLS_PK_Setup | Set pk_info + ctx_alloc via vtable | high | Docs |
| `0x08044a74` | mbedTLS_PK_Sign | Vtable 0x14: sign | high | Docs |
| `0x08044ad0` | mbedTLS_PK_Verify | Wrapper → Verify_Restartable(NULL) | high | Docs |
| `0x08044af8` | mbedTLS_PK_Verify_Ext | Extended verify with algorithm check | medium | Docs |
| `0x08044b5c` | mbedTLS_PK_Verify_Restartable | Core verify via vtable 0x10 | high | Docs |
| `0x08047f1c` | mbedTLS_X509_CRT_Free | Free cert chain (0x154B structs) | high | Docs |
| `0x08048008` | mbedTLS_X509_CRT_Init | memset(0, 0x154) | high | Docs |
| `0x08048018` | mbedTLS_X509_CRT_Parse | PEM vs DER detection, loop over PEM blocks | high | Docs |
| `0x08048150` | mbedTLS_X509_CRT_Parse_DER | Wrapper → _Internal(flag=1, make_copy) | high | Docs |
| `0x0804816c` | mbedTLS_X509_CRT_Parse_DER_Internal | Core chain parser, allocate 0x154B cert nodes | high | Docs |
| `0x0804820c` | mbedTLS_X509_CRT_Parse_DER_NoCopy | Wrapper → _Internal(flag=0, no buffer copy) | high | Docs |
| `0x08048228` | mbedTLS_X509_CRT_Verify | Wrapper → verify_restartable(NULL,NULL) | high | Docs |
| `0x08048254` | mbedTLS_X509_Get_Alg | ASN1_Get_Alg + X.509 error layer | high | Docs |
| `0x080482b8` | mbedTLS_X509_Get_Ext | Context-specific tag 0xA0, v3 extensions | high | Docs |
| `0x08048370` | mbedTLS_X509_Get_Name | Distinguished name → linked list (0x20B nodes) | high | Docs |
| `0x08048460` | mbedTLS_X509_Get_Serial | Serial number (tag 0x82/0x02) | high | Docs |
| `0x0804850c` | mbedTLS_X509_Get_Sig | Signature value via Bitstring_Null | high | Docs |
| `0x0804859c` | mbedTLS_X509_Get_Sig_Alg | OID → sig algorithm, param validation | high | Docs |
| `0x08048628` | mbedTLS_X509_Get_Time | UTCTime (0x17) / GeneralizedTime (0x18) | high | Docs |
| `0x080486dc` | mbedTLS_X509_Name_Parse_Der | DER name → 40B structs, encoding-type bitmask | high | Docs |
| `0x08048748` | mbedTLS_X509_Check_Cert_Validity | Cert validity check (+0x90) | medium | Docs |
| `0x0804874e` | mbedTLS_X509_Check_Cert_Extensions | Extensions check (+0xa8) | medium | Docs |
| `0x08048fc4` | mbedTLS_OID_SearchMdAlgTable | OID → MD algorithm | high | Docs |
| `0x08049000` | mbedTLS_OID_SearchPkAlgTable | OID → PK algorithm | high | Docs |
| `0x0804903c` | mbedTLS_OID_SearchSigAlgTable | OID → sig algorithm | high | Docs |
| `0x08049078` | mbedTLS_OID_SearchEcGrpTable | OID → EC group | high | Docs |
| `0x080498d4` | mbedTLS_PK_Get_ECParams | EC params OID from ASN.1 (tag 0x06) | high | Docs |
| `0x0804998c` | mbedTLS_PK_Get_ECPubKey | EC public key → ReadBinary + Check_PubKey | high | Docs |
| `0x080499c4` | mbedTLS_PK_Get_PkAlg | AlgorithmIdentifier → PK algorithm | high | Docs |
| `0x08049a60` | mbedTLS_PK_Get_RSAPubKey | RSA public key (N,E) from DER SEQUENCE | high | Docs |
| `0x08049ba8` | mbedTLS_PK_HashLenHelper | Validate hash length vs MD type | high | Docs |
| `0x08049be8` | mbedTLS_PK_ParseKey_PKCS1_DER | PKCS#1 RSAPrivateKey (8x ASN1_Get_MPI) | high | Docs |
| `0x08049df8` | mbedTLS_PK_Parse_Key_PKCS8_Unencrypted_DER | PKCS#8 → RSA(type1) / EC(type2/3) | high | Docs |
| `0x08049f88` | mbedTLS_PK_Parse_Key_SEC1_DER | SEC1 EC private key (d scalar, optional pubkey) | high | Docs |
| `0x0804a1b8` | mbedTLS_PK_Use_ECParams | Set EC params OID → group ID | high | Docs |
| `0x08052004` | mbedTLS_X509_CheckWildcard | Check wildcard match for X.509 hostnames | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052078` | mbedTLS_X509_Crt_CheckCN | Check certificate common name | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080520aa` | mbedTLS_X509_Crt_CheckSelfSignedLoop | Detect self-signature loop in certificate | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080520ec` | mbedTLS_X509_Crt_CheckParent | Check whether parent certificate matches | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805212a` | mbedTLS_X509_Crt_CheckSAN_DNSName | Check DNS name in Subject Alternative Name | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052152` | mbedTLS_X509_Crt_CheckSignature | Check certificate signature | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080521c8` | mbedTLS_X509_Crt_FindParent | Find parent certificate in chain | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805222e` | mbedTLS_X509_Crt_FindParentIn | Find parent certificate in given list | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080522dc` | mbedTLS_X509_Crt_ApplyVerifyCallback | Invoke application-side verify callback | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052328` | mbedTLS_ASN1_X509_Parse | *(no description — see notes above)* | - | — |
| `0x080526e8` | mbedTLS_X509_Crt_VerifyChain | Verify certificate chain | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052818` | mbedTLS_X509_Crt_VerifyChain_Init | Initialize verification context for certificate chain | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052838` | mbedTLS_X509_Crt_VerifyCN | Verify common name against expected hostname | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080528c8` | mbedTLS_X509_Crt_VerifyRestartable | Interruptible certificate verification | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052998` | mbedTLS_X509_DateIsValid | Check certificate validity date | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052a7c` | mbedTLS_X509_Get_AttrTypeValue | Read AttributeTypeAndValue from ASN.1 | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052bf4` | mbedTLS_X509_Get_BasicConstraints | Read BasicConstraints extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052d10` | mbedTLS_X509_Get_CertificatePolicies | Read CertificatePolicies extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08052ee8` | mbedTLS_X509_Get_CrtExt | Read certificate extensions | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805319c` | mbedTLS_X509_Get_Dates | Read validity period (NotBefore/NotAfter) | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053244` | mbedTLS_X509_Get_ExtKeyUsage | Read ExtendedKeyUsage extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080532c0` | mbedTLS_X509_Get_KeyUsage | Read KeyUsage extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053364` | mbedTLS_X509_Get_NsCertType | Read Netscape cert-type extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080533e8` | mbedTLS_X509_Get_OtherName | Read OtherName field from SAN | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080535a8` | mbedTLS_X509_Get_SubjectAltName | Read SubjectAltName extension | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805372c` | mbedTLS_X509_Get_UID | Read issuer/subject unique ID | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080537bc` | mbedTLS_X509_Get_Version | Read certificate version | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053868` | mbedTLS_X509_NameCompare_CaseInsensitive | Compare names case-insensitively | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080538b4` | mbedTLS_X509_Name_Compare | Compare X.509 names | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805391c` | mbedTLS_ASN1_ParseDecimalDigits | Parse decimal digits from ASN.1 time string | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053964` | mbedTLS_X509_ParseTime_Internal | Internal parsing of ASN.1 time fields | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053a7c` | mbedTLS_X509_Profile_CheckPubkeyAlg | Check public-key algorithm against X.509 profile | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053b04` | mbedTLS_X509_Profile_CheckMdAlg | Check hash algorithm against X.509 profile | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053b26` | mbedTLS_X509_Profile_CheckPkAlg | Check signature algorithm against X.509 profile | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053b48` | mbedTLS_X509_Name_CompareAttribute | Compare single name attribute | Batch 20 | Ghidra (Batch 20, 2026-07-09) |

## mbedTLS — Symmetric Crypto / Hash / RNG (91)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08031d68` | mbedTLS_AES_Crypt_ECB_Wrapper | cipher_info_t function-pointer target | high | Doku |
| `0x08031d84` | mbedTLS_AES_Context_Alloc | Allocate + initialize 280B AES context | high | Doku |
| `0x08031da2` | mbedTLS_AES_Context_Free | AES context cleanup + free | high | Doku |
| `0x08031db4` | mbedTLS_AES_Gen_Tables | Generate 574B S-box/T-tables in software (no HW AES!) | very high | Doku |
| `0x08032020` | mbedTLS_AES_Setkey_Dec_Wrapper | Decrypt key-schedule wrapper | high | Doku |
| `0x08032034` | mbedTLS_AES_Setkey_Enc_Wrapper | Encrypt key-schedule wrapper | high | Doku |
| `0x08032194` | mbedTLS_AES_CMAC_PRF_KDF | CMAC-based KDF, 3×16=48B output | high | Doku |
| `0x08032e30` | mbedTLS_CTR_DRBG_Update | Increment counter, AES encrypt, XOR | high | Doku |
| `0x08034a60` | mbedTLS_Entropy_Gather | Query registered entropy sources | high | Doku |
| `0x08034b0e` | mbedTLS_Entropy_Update | SHA-256 hash + accumulator update | high | Doku |
| `0x08034fd6` | GCM_Setup_Hash_Subkey | Build GF(2^128) multiplication tables | high | Doku |
| `0x08035168` | GCM_GHASH_Multiply | 4-bit nibble table-lookup GF multiplication (3 callers) | high | Doku |
| `0x0803e3a0` | mbedTLS_AES_Crypt_ECB | ECB encrypt/decrypt dispatcher | high | Doku |
| `0x0803e3da` | mbedTLS_AES_Free | Zeroize 0x118B AES context + NULL check | high | Doku |
| `0x0803e3f0` | mbedTLS_AES_Init | Zero 0x118B AES context | high | Doku |
| `0x0803e404` | mbedTLS_AES_RoundKey_Offset | Compute round-key table base offset | high | Doku |
| `0x0803e40c` | mbedTLS_AES_Setkey_Dec | Decrypt key schedule + InvMixColumns | high | Doku |
| `0x0803e500` | mbedTLS_AES_Setkey_Enc | Encrypt key schedule (128/192/256-bit, RCON+S-box) | high | Doku |
| `0x0803eb2a` | mbedTLS_Base64_Decode | Full Base64 decoder (364B), input validation | high | Doku |
| `0x0803ec98` | mbedTLS_Calloc | Heap allocator via function pointer (22 callers) | high | Doku |
| `0x0803ecb0` | mbedTLS_Cipher_AuthDecryptUpdate | Inner AEAD decrypt via vtable | high | Doku |
| `0x0803ed18` | mbedTLS_Cipher_AuthEncryptUpdate | Inner AEAD encrypt via vtable | high | Doku |
| `0x0803ed64` | mbedTLS_Cipher_AuthDecryptExt | Outer wrapper with buffer-size validation | high | Doku |
| `0x0803edcc` | mbedTLS_Cipher_AuthEncryptExt | Outer wrapper with buffer-size validation | high | Doku |
| `0x0803ee34` | mbedTLS_Cipher_Free | Vtable free + zeroize 0x38B context | high | Doku |
| `0x0803ee56` | mbedTLS_Cipher_GetType | Cipher-type enum from cipher_info+1 | high | Doku |
| `0x0803ee6c` | mbedTLS_Cipher_InfoFromSuiteId | Table lookup by suite ID (8B entries) | high | Doku |
| `0x0803ee8c` | mbedTLS_Cipher_InfoFromValues | 3-field match (cipher_id, key_bits, mode) | high | Doku |
| `0x0803eec4` | mbedTLS_Cipher_Init | Context memset(0, 0x38) | high | Doku |
| `0x0803eed8` | mbedTLS_Cipher_SetKey | Key via vtable, mode 3/4/5 → always encrypt | high | Doku |
| `0x0803ef5c` | mbedTLS_Cipher_Setup | Alloc + ctx_alloc via vtable | high | Doku |
| `0x0803ef90` | mbedTLS_Cipher_ProcessBlock | ECB/stream dispatch via vtable | high | Doku |
| `0x0803f054` | mbedTLS_CT_CountLeadingZeros | CLZ loop with 0x80000000 mask | high | Doku |
| `0x0803f072` | mbedTLS_Base64_DecodeChar | Constant-time char→6-bit value (A-Z, a-z, 0-9, +, /) | high | Doku |
| `0x0803f0d6` | mbedTLS_CT_MemMoveToLeft | Constant-time left shift (RSA unpadding) | high | Doku |
| `0x0803f134` | mbedTLS_CT_MemCompare | **SECURITY** timing-safe compare (XOR-OR, 4 callers) | high | Doku |
| `0x0803f18e` | mbedTLS_CT_MpiUintLt | Constant-time "a < b" for MPI limbs | high | Doku |
| `0x0803f1a6` | mbedTLS_CT_IsNonZero | `(-x\|x)>>31` → -1 or 0 | high | Doku |
| `0x0803f308` | mbedTLS_CT_UintEqual | `(x^y\|-(x^y))>>31^1` | high | Doku |
| `0x0803f31e` | mbedTLS_CT_UintLt | `(a-b)>>31` simple variant | high | Doku |
| `0x0803f326` | mbedTLS_ConstantTime_InRange | Branchless range check for Base64 | high | Doku |
| `0x0803f33c` | mbedTLS_ConstantTime_Select | Conditional select via IsNonZero mask | high | Doku |
| `0x0803f356` | mbedTLS_ConstantTime_IsNonZero | Second IsNonZero variant (12B) | high | Doku |
| `0x0803f362` | mbedTLS_CTR_DRBG_Free | AES_Free + zeroize 0x140B, reseed=10000 | high | Doku |
| `0x0803f38c` | mbedTLS_CTR_DRBG_Init | memset(0, 0x140), reseed_interval=10000 | high | Doku |
| `0x0803f3a8` | mbedTLS_CTR_DRBG_Random | Public wrapper → Random_WithAdd(0,0) | high | Doku |
| `0x0803f3d0` | mbedTLS_CTR_DRBG_Random_WithAdd | Core DRBG: AES-256-ECB counter, auto-reseed | high | Doku |
| `0x0803f4ce` | mbedTLS_CTR_DRBG_Reseed | Wrapper → Reseed_Internal(nonce=0) | high | Doku |
| `0x0803f4e4` | mbedTLS_CTR_DRBG_Reseed_Internal | Entropy via callback, state update | high | Doku |
| `0x0803f5b8` | mbedTLS_CTR_DRBG_Seed | Seed: entropy pointer + AES-256 init + reseed | high | Doku |
| `0x0804004c` | mbedTLS_Entropy_Add_Source | Source array (max 20, stride 0x14) | high | Doku |
| `0x0804009e` | mbedTLS_Entropy_Free | SHA free + zeroize 400B | high | Doku |
| `0x080400ca` | mbedTLS_Entropy_Func | Main entropy gathering (0 callers → DRBG pointer) | high | Doku |
| `0x08040206` | mbedTLS_Entropy_Init | State=0, SHA init | high | Doku |
| `0x08040226` | mbedTLS_Error_Add_PK | `return p1+p2` for the PK module | high | Doku |
| `0x0804022e` | mbedTLS_Error_Add_PEM | `return p1+p2` for the PEM module (7 callers) | high | Doku |
| `0x08040236` | mbedTLS_Error_Add_RSA | `return p1+p2` for the RSA module | high | Doku |
| `0x0804023e` | mbedTLS_Error_Add_X509 | `return p1+p2` for X.509 (8 callers) | high | Doku |
| `0x08040246` | mbedTLS_Error_Add_X509_Parse | `return p1+p2` for X.509 parsing (12 callers) | high | Doku |
| `0x08040250` | mbedTLS_Platform_Free | Free via function pointer (28 callers) | high | Doku |
| `0x08040264` | mbedTLS_GCM_Auth_Decrypt | Decrypt + tag verify, zeroize on mismatch | high | Doku |
| `0x080402e8` | mbedTLS_GCM_Crypt_And_Tag | Orchestrator: Starts→Update→Finish | high | Doku |
| `0x0804035e` | mbedTLS_GCM_Finish | Final tag: AAD/CT bit length + GHASH + J0 XOR | high | Doku |
| `0x080404e8` | mbedTLS_GCM_Free | cipher_free + zeroize 0x180B | high | Doku |
| `0x08040504` | mbedTLS_GCM_Init | memset(0, 0x180) — 384B context | high | Doku |
| `0x08040518` | mbedTLS_GCM_SetKey | Compute cipher info + key + GHASH subkey H | high | Doku |
| `0x08040598` | mbedTLS_GCM_Starts | IV processing (12B direct / GHASH) + AAD | high | Doku |
| `0x08040766` | mbedTLS_GCM_Update | Core CTR-mode loop + GHASH | high | Doku |
| `0x08041224` | SHA256_Internal_Transform | *(no description — see notes above)* | - | — |
| `0x080420e8` | mbedTLS_MD_Digest | One-shot hash (SHA-224/256 dispatch) | high | Doku |
| `0x0804212c` | mbedTLS_MD_Finish | Finalize hash → digest buffer | high | Doku |
| `0x08042160` | mbedTLS_MD_Free | Free MD context + zeroize | high | Doku |
| `0x080421b4` | mbedTLS_MD_GetSize | Digest size from md_info+5 | high | Doku |
| `0x080421c0` | mbedTLS_MD_HMAC_Finish | HMAC: inner hash → opad → outer hash | high | Doku |
| `0x08042238` | mbedTLS_MD_HMAC_Reset | HMAC reset: restart state + ipad | high | Doku |
| `0x08042270` | mbedTLS_MD_HMAC_Starts | HMAC key setup (0x36/0x5c ipad/opad) | high | Doku |
| `0x08042348` | mbedTLS_MD_HMAC_Update | Wrapper → MD_Update | high | Doku |
| `0x08042370` | mbedTLS_MD_InfoFromType | Type enum → md_info struct (SHA-224/256 only) | high | Doku |
| `0x08042390` | mbedTLS_MD_Init | Zero 3 context fields | high | Doku |
| `0x0804239c` | mbedTLS_MD_Setup | Alloc 0x6c + optional HMAC pads | high | Doku |
| `0x0804240c` | mbedTLS_MD_Starts | Start digest engine (type 5/6 dispatch) | high | Doku |
| `0x08042444` | mbedTLS_MD_Update | Feed data into the digest (5 callers) | high | Doku |
| `0x08044bf4` | mbedTLS_Platform_Zeroize | *(no description — see notes above)* | - | — |
| `0x080462ec` | mbedTLS_SHA256_Clone | memcpy 0x6c (context clone) | high | Doku |
| `0x08046306` | mbedTLS_SHA256_Finish | Padding + 64-bit length + final transform | high | Doku |
| `0x08046480` | mbedTLS_SHA256_Free | Zeroize 0x6c | high | Doku |
| `0x08046494` | mbedTLS_SHA256_Init | memset(0, 0x6c) | high | Doku |
| `0x080464a6` | mbedTLS_SHA256_Digest | One-shot: Init→Starts→Update→Finish→Free | high | Doku |
| `0x0804650c` | mbedTLS_SHA256_Starts | Load 8 IV constants (SHA-224 vs SHA-256) | high | Doku |
| `0x080465a8` | mbedTLS_SHA256_Update | 64B block buffering + transform | high | Doku |
| `0x08046646` | mbedTLS_Cipher_SetupNoop | Null-cipher stub (return 0) | medium | Doku |

## App Crypto (non-mbedTLS: AES/RC4/Base64/ROT) (28)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080004d4` | RC4_KSA_Init | RC4-style key scheduling algorithm (identity fill + Fisher-Yates swap) | high | Doku |
| `0x080018f0` | AES_SubBytes | *(no description — see notes above)* | - | — |
| `0x0800192c` | AES128_ECB_Decrypt_Raw | *(no description — see notes above)* | - | — |
| `0x08001a2c` | AES128_ECB_Encrypt | *(no description — see notes above)* | - | — |
| `0x08001b5c` | AES_InvMixColumns | AES InvMixColumns (decrypt side) | high | Doku |
| `0x08001bec` | AES_InvSBoxLookup | Inverse S-box byte lookup (decrypt) | high | Doku |
| `0x08001bf8` | AES_InvShiftRows | Inverse ShiftRows (decrypt) | high | Doku |
| `0x08001c58` | AES_InvSubBytes | Inverse S-box over the entire state matrix (decrypt) | high | Doku |
| `0x08001c90` | AES_KeyExpansion | *(no description — see notes above)* | - | — |
| `0x08001d6c` | AES_MixColumns_Fwd | MixColumns (encrypt side) | high | Doku |
| `0x08001dfc` | AES_GF256_Multiply | GF(2^8) multiplication for MixColumns | high | Doku |
| `0x08001e90` | AES_RotWord | Rotate a 32-bit word 8 bits left (key expansion) | high | Doku |
| `0x08001e9c` | AES_SBoxLookup | S-box byte lookup (encrypt) | high | Doku |
| `0x08001ea8` | AES_ShiftRows | ShiftRows (encrypt) | high | Doku |
| `0x08001f00` | AES_SubBytes_Fwd | S-box over the entire state matrix (encrypt) | high | Doku |
| `0x08001f38` | AES_SubWord | S-box on 4 bytes of a 32-bit word (key expansion) | high | Doku |
| `0x08001f6c` | AES_ByteSwap32 | Reverse 32-bit byte order (key expansion) | high | Doku |
| `0x08001f86` | AES_GF256_xtime | GF(2^8) xtime: left shift with XOR 0x1b | high | Doku |
| `0x08001f9a` | AES_GF256_Pow2 | Iterates xtime N times for x^(2^N) in GF(2^8) | high | Doku |
| `0x08002118` | Base64_Decode | *(no description — see notes above)* | - | — |
| `0x0800221c` | Base64_Encode | *(no description — see notes above)* | - | — |
| `0x08003e10` | ROT_N_PrintableASCII | *(no description — see notes above)* | - | — |
| `0x0800b630` | KeyDerive_CopyAndROT | *(no description — see notes above)* | - | — |
| `0x08024b54` | TLS_Cert_Decrypt_All | *(no description — see notes above)* | - | — |
| `0x0802d4d0` | AES128_ECB_Decrypt | *(no description — see notes above)* | - | — |
| `0x080358b0` | Crypto_Validate_And_Dispatch | Validate TLS session | medium | Doku |
| `0x0804090c` | AES_Encrypt_Block | *(no description — see notes above)* | - | — |
| `0x08040d98` | AES_Decrypt_Block | *(no description — see notes above)* | - | — |

## llhttp — HTTP Parser (Vendor Lib) (96)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080366a8` | llhttp__AfterHeadersComplete | Determine body-parsing mode (chunked/content-length/EOF); verified by decompilation (256B, calls message_needs_eof) | high | Re-Audit 2026-07-14 |
| `0x080367a8` | llhttp__AfterMessageComplete | Calls ShouldKeepAlive, then resets the finish byte @0x31 and flags word @0x32 for the next message | high | Re-Audit 2026-07-14 |
| `0x080367c4` | llhttp__BeforeHeadersComplete | Sets the upgrade-candidate byte @0x30 (Method==CONNECT or status 101/204 special case) | high | Re-Audit 2026-07-14 |
| `0x0803680a` | llhttp__ClearChunkedFlag | Clear bit 3 (F_CHUNKED) in flags | high | Doku |
| `0x08036818` | llhttp__IsContentLengthZero | Check whether 64-bit content length (@0x20/0x24) == 0 | high | Re-Audit 2026-07-14 |
| `0x08036834` | llhttp__IsMethodConnect | Check method byte @0x29 == 0x05 (HTTP_CONNECT) | high | Re-Audit 2026-07-14 |
| `0x08036846` | llhttp__IsUpgrade | Check upgrade-candidate byte @0x30 == 1 | high | Re-Audit 2026-07-14 |
| `0x08036856` | llhttp__GetHeaderState | Read header-state byte @0x2c (only signature/size 8B verified, timeout on full decompilation) | high | Re-Audit 2026-07-14 (partial verification) |
| `0x0803685e` | llhttp__GetHttpMajor | Read HTTP major version | high | Doku |
| `0x08036866` | llhttp__GetHttpMinor | Read HTTP minor version | high | Doku |
| `0x0803686e` | llhttp__LoadInitialMessageCompleted | Reads byte @0x36 (message_completed flag) | high | Re-Audit 2026-07-14 |
| `0x08036876` | llhttp__LoadMethod | Reads byte @0x29 (method field) | high | Re-Audit 2026-07-14 |
| `0x0803687e` | llhttp__LoadType | Reads byte @0x28 (type: request/response) | high | Re-Audit 2026-07-14 |
| `0x08036886` | llhttp__MulAddContentLengthHex | 64-bit content length ×16 + hex digit, with overflow check | high | Re-Audit 2026-07-14 |
| `0x0803690c` | llhttp__MulAddContentLength | 64-bit content length ×10 + decimal digit, with overflow check | high | Re-Audit 2026-07-14 |
| `0x08036998` | llhttp__MulAddStatusCode | 16-bit status code ×10 + digit, with overflow check | high | Re-Audit 2026-07-14 |
| `0x080369e0` | llhttp__OrFlags_Trailing | Set bit 0x80 in flags @0x32 (F_TRAILING) | high | Re-Audit 2026-07-14 |
| `0x080369ee` | llhttp__OrFlags_SkipBody | Set bit 0x40 in flags @0x32 (F_SKIPBODY) | high | Re-Audit 2026-07-14 |
| `0x080369fc` | llhttp__OrFlags_ContentLength | Set bit 0x20 in flags @0x32 (F_CONTENT_LENGTH) | high | Re-Audit 2026-07-14 |
| `0x08036a0a` | llhttp__OrFlags_TransferEncoding | Set bit 0x200 in flags @0x32 (F_TRANSFER_ENCODING) | high | Re-Audit 2026-07-14 |
| `0x08036a18` | llhttp__OrFlags_Upgrade | Set bit 0x10 in flags @0x32 (F_UPGRADE) | high | Re-Audit 2026-07-14 |
| `0x08036a26` | llhttp__internal__c_or_flags | Set bit 0x01 in flags @0x32 (F_CHUNKED) | high | Re-Audit 2026-07-14 |
| `0x08036a34` | llhttp__internal__c_or_flags_1 | Set bit 0x02 in flags @0x32 (F_CONNECTION_KEEP_ALIVE) | high | Re-Audit 2026-07-14 |
| `0x08036a42` | llhttp__internal__c_or_flags_2 | Set bit 0x04 in flags @0x32 (F_CONNECTION_CLOSE) | high | Re-Audit 2026-07-14 |
| `0x08036a50` | llhttp__internal__c_or_flags_3 | Set bit 0x08 in flags @0x32 (F_CONNECTION_UPGRADE) | high | Re-Audit 2026-07-14 |
| `0x08036a5e` | llhttp__internal__c_store_header_state | Byte @0x2c = parameter (set header_state) | high | Re-Audit 2026-07-14 |
| `0x08036a6a` | llhttp__internal__c_store_http_major | Byte @0x2a = parameter (set http_major) | high | Re-Audit 2026-07-14 |
| `0x08036a76` | llhttp__internal__c_store_http_minor | Byte @0x2b = parameter (set http_minor) | high | Re-Audit 2026-07-14 |
| `0x08036a82` | llhttp__internal__c_store_method | Byte @0x29 = parameter (set method) | high | Re-Audit 2026-07-14 |
| `0x08036a8e` | llhttp__internal__c_test_flags_Off32 | Test bit 7 (MSB) of flags byte @0x32 | high | Re-Audit 2026-07-14 |
| `0x08036a9a` | llhttp__internal__c_test_flags_1_Off32 | Test bit 5 (masked 0x3f) of flags @0x32 | high | Re-Audit 2026-07-14 |
| `0x08036aa6` | llhttp__internal__c_test_flags_2_Off32 | Test bit 3 (masked 0xf) of flags @0x32 | high | Re-Audit 2026-07-14 |
| `0x08036ab2` | llhttp__internal__c_test_flags_3_Off32 | Test bit 9 (16-bit, masked 0x3ff) of flags @0x32 | high | Re-Audit 2026-07-14 |
| `0x08036abc` | llhttp__internal__c_load_upgrade | Presumably reads the upgrade field — Ghidra timeout on full decompilation, only signature/size (12B) verified, name/pattern matches neighboring functions | medium | Re-Audit 2026-07-14 (partial verification) |
| `0x08036ac8` | llhttp__internal__c_test_lenient_flags | Presumably tests a lenient-flags bit — Ghidra timeout on full decompilation, only signature/size (10B) verified, name/pattern matches neighboring functions | medium | Re-Audit 2026-07-14 (partial verification) |
| `0x08036ad2` | llhttp__internal__c_test_flags_2_Off2e | Test bit 2 (masked 7) of flags @0x2e | high | Re-Audit 2026-07-14 |
| `0x08036ade` | llhttp__internal__c_test_flags_1_Off2e | Test bit 3 (masked 0xf) of flags @0x2e | high | Re-Audit 2026-07-14 |
| `0x08036aea` | llhttp__internal__c_test_flags_6 | Test bit 1 (masked 3) of flags @0x2e | high | Re-Audit 2026-07-14 |
| `0x08036af6` | llhttp__internal__c_test_flags_3_Off2e | Test bit 4 (masked 0x1f) of flags @0x2e | high | Re-Audit 2026-07-14 |
| `0x08036b02` | llhttp__internal__c_test_flags_5 | Test bit 5 (masked 0x3f) of flags @0x2e | high | Re-Audit 2026-07-14 |
| `0x08036b0e` | llhttp__internal__c_test_lenient_flags_1 | Test bit 9 (16-bit, masked 0x3ff) of flags @0x2e | high | Re-Audit 2026-07-14 |
| `0x08036b18` | llhttp__internal__c_test_flags_Off2e | Test bit 7 (MSB) of flags byte @0x2e | high | Re-Audit 2026-07-14 |
| `0x08036b24` | llhttp__internal__c_test_flags_4 | Presumably a bit test on flags @0x2e — Ghidra timeout on full decompilation, only signature/size (12B) verified, name/pattern matches neighboring functions | medium | Re-Audit 2026-07-14 (partial verification) |
| `0x08036b30` | llhttp__internal__c_update_content_length | Reset 64-bit content length @0x20/0x24 to 0 | high | Re-Audit 2026-07-14 |
| `0x08036b3e` | llhttp__internal__c_update_finish | Byte @0x31 = 2 (finish state) | high | Re-Audit 2026-07-14 |
| `0x08036b4a` | llhttp__internal__c_update_finish_safe | Byte @0x31 = 0 (reset finish state) | high | Re-Audit 2026-07-14 |
| `0x08036b54` | llhttp__internal__c_update_finish_1 | Byte @0x31 = 1 (finish state) | high | Re-Audit 2026-07-14 |
| `0x08036b60` | llhttp__internal__c_update_header_state | Byte @0x2c = 1 (header_state) | high | Re-Audit 2026-07-14 |
| `0x08036b6c` | llhttp_ClearHeaderState | Presumably byte @0x2c = 0 — Ghidra timeout on full decompilation, only signature/size (10B) verified, matches the set/clear family @0x2c | medium | Re-Audit 2026-07-14 (partial verification) |
| `0x08036b76` | llhttp_SetHeaderState_ConnClose | Byte @0x2c = 6 (header_state = connection: close) | high | Re-Audit 2026-07-14 |
| `0x08036b82` | llhttp_SetHeaderState_ConnKeepAlive | Byte @0x2c = 5 (header_state = connection: keep-alive) | high | Re-Audit 2026-07-14 |
| `0x08036b8e` | llhttp_SetHeaderState_ConnUpgrade | Byte @0x2c = 7 (header_state = connection: upgrade) | high | Re-Audit 2026-07-14 |
| `0x08036b9a` | llhttp_SetHeaderState_TEChunked | Byte @0x2c = 8 (header_state = transfer-encoding: chunked) | high | Re-Audit 2026-07-14 |
| `0x08036ba6` | llhttp_ClearHttpMajor | Byte @0x2a = 0 (reset http_major, HTTP/0.9 special case) | high | Re-Audit 2026-07-14 |
| `0x08036bb0` | llhttp_SetHttpMinor9 | Byte @0x2b = 9 (http_minor=9, HTTP/0.9 simple-request special case) | high | Re-Audit 2026-07-14 |
| `0x08036bbc` | llhttp_SetInitialMessageCompleted | Byte @0x36 = 1 (set message_completed flag) | high | Re-Audit 2026-07-14 |
| `0x08036bc8` | llhttp_ClearStatusCode | Word @0x34 = 0 (reset status_code) | high | Re-Audit 2026-07-14 |
| `0x08036bd0` | llhttp_SetTypeRequest | Byte @0x28 = 1 (type = HTTP_REQUEST) | high | Re-Audit 2026-07-14 |
| `0x08036bdc` | llhttp_SetTypeResponse | Byte @0x28 = 2 (type = HTTP_RESPONSE) | high | Re-Audit 2026-07-14 |
| `0x08036be8` | llhttp__internal__c_or_flags_4 | Byte @0x30 = 1 (skip-body context flag) — **re-audit note:** direct assignment instead of bit-OR, despite the name family not a real OR operation (the field holds only this one flag) | high | Re-Audit 2026-07-14 |
| `0x08036bf4` | llhttp_Parser_Execute | Main state machine of the generated llhttp parser (22,470 bytes!) — dispatches by state byte, calls all internal helpers (c_*, Or/Test/Store/Update flags) as well as all settings callbacks | high | Re-Audit 2026-07-14 |
| `0x0803d538` | llhttp_Parse_Wrapper | Outer wrapper: caches error status (@0xc), calls Parser_Execute, manages pause/reexec via state-handler function pointer @0x08 (corresponds to llhttp__internal_execute) | high | Re-Audit 2026-07-14 |
| `0x0803d588` | llhttp_ParserInternalInit | Zero 0x40B struct (memclr intrinsic), set state 0xec — verified, description correct | high | Re-Audit 2026-07-14 |
| `0x0803d59c` | llhttp_InvokeCallback_OnBody | settings+0x28, span (passes length as param_3-param_2) | high | Re-Audit 2026-07-14 |
| `0x0803d5fc` | llhttp_InvokeCallback_OnChunkComplete | settings+0x54, lifecycle — **corrected 2026-07-09**, see batch 18; re-verified by decompilation | high | Re-Audit 2026-07-14 |
| `0x0803d628` | llhttp_InvokeCallback_OnChunkExtName | settings+0x1c, span | high | Doku |
| `0x0803d698` | llhttp_InvokeCallback_OnChunkExtNameComplete | settings+0x48, lifecycle — **corrected 2026-07-09**, see batch 18 | medium | Doku |
| `0x0803d6c4` | llhttp_InvokeCallback_OnChunkExtValue | settings+0x20, span | high | Doku |
| `0x0803d734` | llhttp_InvokeCallback_OnChunkExtValueComplete | settings+0x4c, lifecycle — **corrected 2026-07-09**, see batch 18 | medium | Doku |
| `0x0803d75e` | llhttp_InvokeCallback_OnChunkHeader | settings+0x50, lifecycle — **corrected 2026-07-09**, see batch 18; re-verified by decompilation | high | Re-Audit 2026-07-14 |
| `0x0803d788` | llhttp_InvokeCallback_OnHeaderField | settings+0x14, span | high | Doku |
| `0x0803d7f0` | llhttp_InvokeCallback_OnHeaderFieldComplete | settings+0x40, lifecycle; verified by decompilation | high | Re-Audit 2026-07-14 |
| `0x0803d81c` | llhttp__on_header_value | settings+0x18, span | high | Doku |
| `0x0803d884` | llhttp__on_header_value_complete | settings+0x44, lifecycle | high | Doku |
| `0x0803d8ae` | llhttp__on_headers_complete | settings+0x24, lifecycle | high | Re-Audit 2026-07-14 |
| `0x0803d8d8` | llhttp__on_message_begin | settings+0x00, lifecycle | high | Re-Audit 2026-07-14 |
| `0x0803d902` | llhttp__on_message_complete | settings+0x2c, lifecycle | high | Doku |
| `0x0803d92c` | llhttp__on_method | settings+0x0c, span | high | Doku |
| `0x0803d990` | llhttp__on_method_complete | settings+0x38, lifecycle | high | Doku |
| `0x0803d9ba` | llhttp_InvokeCallback_OnReset | settings+0x58, lifecycle — **corrected 2026-07-09**, see batch 18; re-verified by decompilation | high | Re-Audit 2026-07-14 |
| `0x0803d9e4` | llhttp__on_status | settings+0x08, span | high | Doku |
| `0x0803da48` | llhttp__on_status_complete | settings+0x34, lifecycle | high | Doku |
| `0x0803da74` | llhttp__InvokeOnUrl | settings+0x10, span | high | Doku |
| `0x0803dad4` | llhttp__InvokeOnUrlComplete | settings+0x30, lifecycle | high | Doku |
| `0x0803db00` | llhttp__InvokeOnVersion | settings+0x04, span | high | Doku |
| `0x0803db64` | llhttp__InvokeOnVersionComplete | settings+0x3c, lifecycle | high | Doku |
| `0x0803db90` | llhttp_errno_name | HPE_* error codes (0x00-0x23) → strings, 440B switch | high | Re-Audit 2026-07-14 |
| `0x0803df88` | llhttp_execute | Public API: parser+data+len → Parse_Wrapper | high | Re-Audit 2026-07-14 |
| `0x0803dfa2` | llhttp_init | Calls ParserInternalInit, sets type @0x28 and settings pointer @0x38 | high | Re-Audit 2026-07-14 |
| `0x0803dfb8` | llhttp_message_needs_eof | Body ends at EOF? (1xx/204/304, content-length, chunked) | high | Re-Audit 2026-07-14 |
| `0x0803e016` | llhttp_set_error_reason | Store error string in parser+0x10 (9 callers, all confirmed) | high | Re-Audit 2026-07-14 |
| `0x0803e01a` | llhttp_settings_init | settings memset(0, 0x5c) — 23 callback pointers | high | Re-Audit 2026-07-14 |
| `0x0803e028` | llhttp_ShouldKeepAlive | HTTP version + connection flags → keep-alive?; calls message_needs_eof | high | Re-Audit 2026-07-14 |
| `0x0803e068` | llhttp_MatchSequence | Case-sensitive token matcher (96B, only signature/size verified, timeout on full decompilation) | high | Re-Audit 2026-07-14 (partial verification) |
| `0x0803e0c8` | llhttp_MatchSequenceToLower | Case-insensitive (with A-Z range check) | high | Doku |
| `0x0803e14a` | llhttp_MatchSequenceToLowerUnsafe | Case-insensitive (without range check) | high | Doku |

## HTTP/HTTPS — App layer (19)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x0800bcc4` | Quectel_HTTP_AT_SendAndVerify | Name corrected (2026-07-15, formerly HTTP_POST_Request): generic Quectel AT executor, no POST-specific logic: sends command (param_1) via UART, waits for notify (max. 400 ticks) + delay (param_4), checks the response buffer for substring(s) param_2/param_3. Used equally for HTTP POST (cloud reporting), HTTP GET (Quectel_HTTP_GET_Request), HTTP READ (Quectel_HTTP_ReadResponse) AND SSL config (Quectel_HTTP_Config_SSLCtxId) | medium | Re-Audit 2026-07-15 |
| `0x0800dac8` | HTTP_Cloud_Reporting_Dispatcher | Cloud reporting dispatcher: reads server type from EEPROM 0x441 (0-4, various cloud subdomains), param_1 selects report type via switch (1=full telemetry, 2=short status, 3-7 further report types), builds JSON via Cloud_Telemetry_JSON_Builder (=generic sprintf wrapper), sends via HTTP_POST_Request (7 call sites) | high | Re-Audit 2026-07-15 |
| `0x08013678` | HTTP_URL_ExtractPath | Extract path portion from URL | Batch 20 | Re-Audit 2026-07-15 |
| `0x080139ac` | Meter_ExtractValue_ByKey | Extract meter value from response by key | Batch 20 | Re-Audit 2026-07-15 |
| `0x080143c4` | HTTP_ParseContentLengthHeader | Parse Content-Length header | Batch 20 | Re-Audit 2026-07-15 |
| `0x08014458` | HTTP_FindHeaderBodySeparator | Find separator between HTTP header and body | Batch 20 | Re-Audit 2026-07-15 |
| `0x080148f8` | HTTP_EcoTracker_ParsePower | Parse EcoTracker power value from HTTP response | Batch 20 | Re-Audit 2026-07-15 |
| `0x080149f8` | HTTP_Economy_TOU_Parser | Parses Economy/Time-of-Use HTTP response (electricity price time windows): extracts values via strstr/atoi_u16 from the response string, error counter on "ERROR" string (after 4 failed attempts reset+skip) | high | Re-Audit 2026-07-15 |
| `0x08014dc0` | HTTPS_POST_Request | Complete CH395/Ethernet TCP HTTP(S) request (independent path alongside the Quectel AT-based HTTP_POST_Request @0x0800bcc4): extracts path from URL, builds request buffer, opens TCP socket port 80 via CH395, sends request via SPI, receives response via HTTPS_POST_ReceiveResponseData, optional llhttp response parse | high | Re-Audit 2026-07-15 |
| `0x080152a0` | HTTP_P1Meter_ParseActivePower | Parse P1 meter active power from HTTP response | Batch 20 | Re-Audit 2026-07-15 |
| `0x08015474` | HTTPS_POST_ReceiveResponseData | Receive HTTPS POST response data | Batch 20 | Re-Audit 2026-07-15 |
| `0x080155d0` | HTTP_ParseUploadDecryptResultFlag | Parse decryption result flag on upload | Batch 20 | Re-Audit 2026-07-15 |
| `0x08015668` | HTTP_StormOneMeter_ParseData | Parse StormOne meter data from HTTP response | Batch 20 | Re-Audit 2026-07-15 |
| `0x08016444` | HTTP_ParseServerDateTime_UpdateRTC | Parse server date/time and update RTC | Batch 20 | Re-Audit 2026-07-15 |
| `0x08016580` | HTTP_Economy_TOU_PeriodicHandler | Periodic handler for Economy/Time-of-Use HTTP data | Batch 20 | Re-Audit 2026-07-15 |
| `0x08017010` | HTTP_ExternalMeter_TimeoutMonitor | Timeout monitoring for external HTTP meter | Batch 20 | Re-Audit 2026-07-15 |
| `0x08026644` | HTTP_BuildRequestBuffer | Build HTTP request buffer | high | Re-Audit 2026-07-15 |
| `0x08026930` | HTTP_Response_Parse | Initializes the llhttp parser (response mode) and parses the received HTTP response via llhttp_execute; logs the llhttp error name on parse error. Only caller: HTTPS_POST_Request | high | Re-Audit 2026-07-15 |
| `0x08026344` | DeviceInfo_BuildStatusString | Builds formatted device status string (serial number, FW version, manufacturing date) incl. XOR checksum for telemetry (Ethernet/modem) | high | Re-Audit 2026-07-14 |

## cJSON — Vendor lib (30)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08013e04` | JSON_ExtractFieldValue | Standalone text scanner, NOT part of cJSON: searches for the field name via strstr in the raw JSON text, copies the value up to `,`/`}`/`"`/`\0` via strncpy into the output buffer, no real nesting/escape handling, special case field `device_id` (offset +10); only for HTTP_StormOneMeter_ParseData | high | Re-Audit 2026-07-14 |
| `0x0802a1a8` | cJSON_CreateBool | Creates a bool node with 2 parameters (name string, bool value); name is duplicated via cJSON_strdup; type=4 (false) / 0x104 (true) | high | Re-Audit 2026-07-14 |
| `0x0802a1f4` | cJSON_CreateDouble | Creates a double node with 3 parameters (val_lo, val_hi as 64-bit via CONCAT44, name string); name is strdup'd; type=8 | high | Re-Audit 2026-07-14 |
| `0x0802a248` | cJSON_CreateInt | Integer node: type 0x208, 10 callers | high | Docs |
| `0x0802a290` | cJSON_AddItemToObject | Append named item with child node, 11 callers | high | Docs |
| `0x0802a32c` | cJSON_NewObject | Wrapper: cJSON_New_Item(0x40, 0), actual object creation | high | Docs |
| `0x0802a338` | cJSON_CreateString | Creates a string node with 2 parameters (name, value); both are duplicated separately via cJSON_strdup; type=0x10 | high | Re-Audit 2026-07-14 |
| `0x0802a3ac` | cJSON_Delete | Recursive release of a cJSON node: for array/object (0x20/0x40) recursive delete of the child list, for string (0x10) release of the value string, for name flag (0x400) release of the name string, then release of the node itself — corresponds to cJSON_Delete(cJSON*) | high | Re-Audit 2026-07-14 |
| `0x0802a420` | cJSON_FindChildByName | Search child via strcmp in linked list | high | Docs |
| `0x0802a46c` | cJSON_GetObjectItem | Searches child node by name in the object via cJSON_FindChildByName. Decompiled with 4 parameters; param_3/param_4 implement a potential path chaining, but are left unused (=0) by all known call sites — effectively identical to cJSON_GetObjectItem(object,name) | high | Re-Audit 2026-07-14 |
| `0x0802a4be` | cJSON_GetValue | Internal low-level accessor without a public API counterpart: returns pointer to the value/child field of a node, takes the name flag (0x400) into account for the offset shift; returns 0 for NULL/True/False types. Used by practically all other cJSON_* functions for reading/writing | high | Re-Audit 2026-07-14 |
| `0x0802a4ec` | cJSON_InitHooks | Initializes 3 global function pointers directly as parameters (malloc_fn, free_fn, realloc_fn) instead of via a cJSON_Hooks struct pointer as in the original; 3rd parameter optional (0=keep default) | high | Re-Audit 2026-07-14 |
| `0x0802a520` | cJSON_AddItemToArray | Append item to array/object, 93 callers (!) | high | Docs |
| `0x0802a598` | cJSON_New_Item | Internal node allocator with 2 parameters (type bitmask, optional name pointer). Size is calculated from the type bitmask; if name≠0, the 0x400 flag is set, size +4 bytes, name stored directly in the node | high | Re-Audit 2026-07-14 |
| `0x0802a610` | cJSON_Parse_Array | '[' parser, recursive, max depth 30000 | high | Docs |
| `0x0802a7b2` | cJSON_ParseHex4 | 4 hex chars → uint16 for \uXXXX escapes | high | Docs |
| `0x0802a874` | cJSON_Parse_Number | Recursive number parser: manual parsing of sign/integer/fraction/exponent (no strtod), accumulation via fp64 helper; creates integer node (0x208) or double node (8) depending on value range/decimal places | high | Re-Audit 2026-07-14 |
| `0x0802ab9c` | cJSON_Parse_Object | '{' parser, key-value pairs, recursive | high | Docs |
| `0x0802adfc` | cJSON_Parse | Entry point: skips whitespace, calls cJSON_Parse_Value, optionally checks for trailing garbage/NUL termination. 4 parameters (text, length, require-null-terminated, optional return-parse-end) — functionally closer to cJSON_ParseWithOpts+length than to the 1-parameter original; the only caller passes just 3 arguments | high | Re-Audit 2026-07-14 |
| `0x0802aec8` | cJSON_Parse_String | '"' parser, extract string and create node | high | Docs |
| `0x0802af10` | cJSON_Parse_Value | Top-level dispatcher: string/object/array/number/literal | high | Docs |
| `0x0802b038` | cJSON_PrintBuffered | Top-level serializer: allocate buffer, PrintValue, null-terminate | high | Docs |
| `0x0802b0cc` | cJSON_PrintArray | Serialize array with comma separation | high | Docs |
| `0x0802b254` | cJSON_PrintNumber | Format number (adaptive precision, strip trailing zeros) | high | Docs |
| `0x0802b410` | cJSON_PrintObject | Serialize object with key:value pairs | high | Docs |
| `0x0802b580` | cJSON_PrintValue | Central type dispatcher for serialization | high | Docs |
| `0x0802b6a0` | cJSON_strdup | strlen+malloc+memcpy string duplication | high | Docs |
| `0x08034bd0` | cJSON_EnsureBuffer | Check/reallocate print buffer capacity (18 callers) | high | Docs |
| `0x08049310` | cJSON_ParseStringValue | JSON string parser with escape + UTF-16 surrogate pairs | high | Docs |
| `0x0804a2e0` | cJSON_PrintStringPtr | JSON string printer with escape + \uXXXX | high | Docs |

## MQTT — Client/protocol/payload (102)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080058e0` | MQTT_Topic_Builder | Assemble MQTT topic string from parameters | high | Re-Audit 2026-07-14 |
| `0x0800be50` | MQTT_Telemetry_EnergyCounter_Update | Update energy counters in MQTT telemetry | Batch 20 | Re-Audit 2026-07-14 |
| `0x080109d8` | Config_ScheduleEntries_ClampPower800W | **Name suspicion (Re-Audit 2026-07-14):** the code has NOTHING to do with subscribe — clamps 10 schedule slots (power field, offset+2) to max. 800 and calls Config_Notify_Change() for each changed slot; called from BLE_Recv_Cmd_Dispatcher and MQTT_JSON_RPC_Dispatcher directly after EEPROM_Write(0x90,...) as config post-processing. For a rename proposal see the session result (not applied, duplicate check pending) | medium | Re-Audit 2026-07-14 |
| `0x08010a28` | MQTT_DeviceID_Encrypt | Encrypts device ID (max 33B) with a secondary AES-128-ECB key (SRAM key, ROT(-9)) for MQTT topic construction; output as hex string, callers: BLE_MQTT_Command_Execute, MQTT_Config_VID_Init | high | Re-Audit 2026-07-14 |
| `0x080124a4` | MQTT_Telemetry_Struct_Builder | Collects the current device state (energy counters ÷10, WorkMode, voltage/frequency/power, pack SOC/cell voltages) into a flat telemetry struct for cloud/HTTP reporting; caller: HTTP_Economy_TOU_PeriodicHandler | high | Re-Audit 2026-07-14 |
| `0x080127c4` | MQTT_JSON_Payload_EnsureBracePrefix | Ensure that the JSON payload begins with '{' | Batch 20 | Re-Audit 2026-07-14 |
| `0x08012984` | MQTT_TLS_Context_Reset | Reset of the mbedTLS/SSL context for MQTT: SSL_Free, Config_Free, 2× X509_CRT_Free, PK_Free, CTR_DRBG_Free, Entropy_Free + memset(0x16ec); callers: CH395_MQTT_Socket_Close_And_TLS_Reset, mbedTLS_SSL_Connection_Init | high | Re-Audit 2026-07-14 |
| `0x08013fa4` | MQTT_Config_Command_Handler | Parses JSON cloud config payload: work_mode, ble_status (adv enable), local_api_en, api_port, version_num etc. via MQTT_Config_Extract* helpers, calls MQTT_Config_ParseScheduleEntries when work_mode==1 | high | Re-Audit 2026-07-14 |
| `0x080180c8` | MQTT_Connect_And_Subscribe | MQTT connect sequence: topics, broker, subscribe | high | Re-Audit 2026-07-14 |
| `0x080187fc` | MQTT_Session_Init | Top level: DNS→socket→TLS→connect→subscribe | high | Re-Audit 2026-07-14 |
| `0x080197a4` | MQTT_JSON_AddTypedValue | cJSON add number/string dispatch (10 callers) | high | Re-Audit 2026-07-14 |
| `0x080197f4` | MQTT_SetWorkMode | EEPROM 0x301: Auto=0, AI=5 | high | Re-Audit 2026-07-14 |
| `0x08019860` | MQTT_ParseTypedField | Extract payload field by type (8 callers) | high | Re-Audit 2026-07-14 |
| `0x08019898` | MQTT_BuildBatteryDataResponse | "Bat_data": SOC, temp, capacity, flags | high | Re-Audit 2026-07-14 |
| `0x08019a48` | MQTT_BuildBleDataResponse | "Ble_data": state + MAC | high | Re-Audit 2026-07-14 |
| `0x08019b2c` | MQTT_JSON_CreateResponseEnvelope | JSON-RPC envelope: root + ID + result (11 callers) | high | Re-Audit 2026-07-14 |
| `0x08019b90` | MQTT_JSON_SerializeAndSend | cJSON→string→MQTT publish→free (11 callers) | high | Re-Audit 2026-07-14 |
| `0x08019c00` | MQTT_BuildDeviceDataResponse | "Device_data": name, MAC, WiFi | high | Re-Audit 2026-07-14 |
| `0x08019d2c` | MQTT_BuildEnergyMeterResponse | "Em_data": CT state, phase power, energy | high | Re-Audit 2026-07-14 |
| `0x08019e9c` | MQTT_BuildErrorResponse | JSON-RPC error (42 callers!) | high | Re-Audit 2026-07-14 |
| `0x08019f64` | MQTT_BuildEsModeDataResponse | "Es_mode_data": on/off-grid, SOC, CT | high | Re-Audit 2026-07-14 |
| `0x0801a1b8` | MQTT_BuildEsDataResponse | "Es_data": SOC, PV, grid, load energy | high | Re-Audit 2026-07-14 |
| `0x0801a3a0` | MQTT_BuildMarstekDataResponse | "Marstek_data": full status report | high | Re-Audit 2026-07-14 |
| `0x0801a5a4` | MQTT_JSON_RPC_Dispatcher | JSON-RPC command dispatcher for cloud/MQTT (5024B, 40 callees); method lookup table, cases incl. ble_mac query, device control, config (Auto/AI/Manual mode); error codes 0x191-0x198; callers: Quectel_UDP_DataReceive, CH395_UDP_DataHandler | high | Re-Audit 2026-07-14 |
| `0x0801bc64` | MQTT_ParseRpcParams | JSON-RPC param extractor (9 case handlers) | high | Re-Audit 2026-07-14 |
| `0x0801bda4` | MQTT_PassiveModeHandler | Callback: WorkMode 3 + power setpoint | high | Re-Audit 2026-07-14 |
| `0x0801be60` | MQTT_PassiveModeStart | Activate passive mode with timeout | high | Re-Audit 2026-07-14 |
| `0x0801becc` | MQTT_PassiveModeTimeoutCheck | Watchdog: timeout → callback(1) | high | Re-Audit 2026-07-14 |
| `0x0801bf08` | MQTT_BuildPvDataResponse | "Pv_data": 4 PV strings (PV1-4), each power/voltage/current/state as int | high | Re-Audit 2026-07-14 |
| `0x0801c1b8` | MQTT_RPC_BuildSetResult | "set_result" response (16 callers) | high | Re-Audit 2026-07-14 |
| `0x0801cca0` | MQTT_ResetTimeslotConfig | Reset timeslot config to default | high | Re-Audit 2026-07-14 |
| `0x0801cd1c` | MQTT_BuildWifiDataResponse | "Wifi_data": MAC, IP, gateway, DNS | high | Re-Audit 2026-07-14 |
| `0x0801d342` | MQTT_Clear_SubscriptionSlots | 5 subscription slots (stride 8B): first 4B field (topic pointer) of each set to 0 | high | Re-Audit 2026-07-14 |
| `0x0801d35c` | MQTT_Client_Init | Client context: buffer, semaphores, mutex | high | Re-Audit 2026-07-14 |
| `0x0801d3c2` | MQTT_Reset_ConnectionState | Clear connected flag + subscriptions | high | Re-Audit 2026-07-14 |
| `0x0801d3d8` | MQTT_Connect_Wrapper | Thin wrapper → MQTT_Connect | high | Re-Audit 2026-07-14 |
| `0x0801d3ec` | MQTT_Connect | Build CONNECT packet, send, parse CONNACK | high | Re-Audit 2026-07-14 |
| `0x0801d4b0` | MQTT_Parse_FixedHeader | Fixed header + variable-length + packet ID | high | Re-Audit 2026-07-14 |
| `0x0801d51c` | MQTT_Parse_ConnAck | CONNACK: session present + return code | high | Re-Audit 2026-07-14 |
| `0x0801d592` | MQTT_Deserialize_Publish | PUBLISH: DUP/QoS/retain, topic, payload | high | Re-Audit 2026-07-14 |
| `0x0801d63a` | MQTT_Deserialize_Suback | SUBACK: packet ID + return codes | high | Re-Audit 2026-07-14 |
| `0x0801d6cc` | MQTT_Decode_RemainingLength_ViaCallback | Decodes MQTT variable-length integer (max 4 bytes) via a passed read function pointer instead of a transport object; only caller MQTT_Decode_RemainingLength_Wrapper | high | Re-Audit 2026-07-14 |
| `0x0801d724` | MQTT_Decode_RemainingLength_Wrapper | Indirection with read function pointer | high | Re-Audit 2026-07-14 |
| `0x0801d740` | MQTT_Encode_RemainingLength | Variable-length encoder (6 callers) | high | Re-Audit 2026-07-14 |
| `0x0801d776` | MQTT_Topic_StringCompare | Topic string comparison (strlen+memcmp) | medium | Re-Audit 2026-07-14 |
| `0x0801d7b8` | MQTT_CalcPacketSize | Packet size: payload + header + VarLen | high | Re-Audit 2026-07-14 |
| `0x0801d7e0` | MQTT_Client_SendAndReceive | Mutex, serialize, send, wait for PUBACK | high | Re-Audit 2026-07-14 |
| `0x0801d8ec` | MQTT_Serialize_Ack | PUBACK/PUBREC/PUBREL/PUBCOMP (types 4-7) | high | Re-Audit 2026-07-14 |
| `0x0801d960` | MQTT_Serialize_Connect | CONNECT: "MQIsdp" v3.1, credentials, will | high | Re-Audit 2026-07-14 |
| `0x0801dac8` | MQTT_Calc_Remaining_Length | Remaining length for CONNECT (not PUBLISH/PUBACK): protocol name+version (10/12B) + client ID + optional will topic/message + username/password via MQTT_String_ResolveLength; only caller MQTT_Serialize_Connect | high | Re-Audit 2026-07-14 |
| `0x0801db3e` | MQTT_Build_PingReq | PINGREQ (0xC0) via simple packet | high | Re-Audit 2026-07-14 |
| `0x0801db50` | MQTT_Build_Publish_Packet | PUBLISH: header+topic+payload, QoS check | high | Re-Audit 2026-07-14 |
| `0x0801dc00` | MQTT_Calc_Publish_Size | Precompute PUBLISH packet size | high | Re-Audit 2026-07-14 |
| `0x0801dc22` | MQTT_Build_Subscribe_Packet | SUBSCRIBE (0x82): topics + QoS | high | Re-Audit 2026-07-14 |
| `0x0801dcd2` | MQTT_Calc_Subscribe_Size | Precompute SUBSCRIBE packet size | high | Re-Audit 2026-07-14 |
| `0x0801dd00` | MQTT_Build_Simple_Packet | PINGREQ/DISCONNECT (fixed header only) | high | Re-Audit 2026-07-14 |
| `0x0801dd4c` | MQTT_Manage_Topic_Handler | 5-slot topic handler table (add/remove) | high | Re-Audit 2026-07-14 |
| `0x0801ddd6` | MQTT_Subscribe | Public API wrapper for subscribe | high | Re-Audit 2026-07-14 |
| `0x0801ddf6` | MQTT_Subscribe_Impl | Subscribe flow: mutex→build→send→SUBACK→register | high | Re-Audit 2026-07-14 |
| `0x0801df2c` | MQTT_String_ResolveLength | String pointer → (ptr, len) pair | high | Re-Audit 2026-07-14 |
| `0x0801eeb8` | MQTT_Publish_AI_Data | Publish analog input data to cloud | high | Re-Audit 2026-07-14 |
| `0x0801f0d0` | MQTT_Publish_BMS_Full_Data | Full BMS data (cells, temp, SOC) | high | Re-Audit 2026-07-14 |
| `0x0801f2a0` | MQTT_Config_VID_Init | Initializes topic ID (VID) from flash; if device ID <6 characters → MQTT_DeviceID_Encrypt fallback, otherwise taken over directly; caller: MQTT_Session_Init | high | Re-Audit 2026-07-14 |
| `0x080224c8` | MQTT_Publish_Event_Log | Send event log entries to cloud | high | Re-Audit 2026-07-14 |
| `0x08022a38` | MQTT_Publish_Telemetry | Publishes telemetry via Quectel modem: param_2==1 directly via Quectel_Modem_DataSend, otherwise AT+QMTPUB to topic "marstek_energy/{xid}/device/{sn}/ctrl" (format "cd=%d&data=%s"); caller: Cloud_Reporting_setVenusDReporting | high | Re-Audit 2026-07-14 |
| `0x0802340c` | MQTT_Publish_Network_Status | Network status data | high | Re-Audit 2026-07-14 |
| `0x08023618` | MQTT_Send_Data_Buffer | Generic publish helper: sends byte buffer via Quectel_Modem_DataSend or AT+QMTPUB depending on mode; callers: Cloud_HTTP_Response_Parser etc. | high | Re-Audit 2026-07-14 |
| `0x08023dac` | MQTT_Publish_BMS_Version | BMS version information | high | Re-Audit 2026-07-14 |
| `0x08023ee4` | MQTT_Publish_Grid_Power | Publish grid power data | high | Re-Audit 2026-07-14 |
| `0x08023ffc` | MQTT_Publish_VNS_MPPT_Data | MPPT solar tracker data | high | Re-Audit 2026-07-14 |
| `0x080244a8` | MQTT_Publish_Inverter_Telemetry | Inverter operating data | high | Re-Audit 2026-07-14 |
| `0x0802545c` | MQTT_Session_BLE_Notify | Send MQTT session BLE notification | high | Re-Audit 2026-07-14 |
| `0x08025cbc` | MQTT_Mutex_Create | Create FreeRTOS mutex for MQTT session | high | Re-Audit 2026-07-14 |
| `0x08025cca` | MQTT_Mutex_Take | Acquire MQTT mutex (blocking) | high | Re-Audit 2026-07-14 |
| `0x08025cda` | MQTT_Mutex_Give | Release MQTT mutex | high | Re-Audit 2026-07-14 |
| `0x08025e5c` | MQTT_Server_Config_Select | Selects broker configuration (custom server vs. default) depending on flag +0xe4 and writes host/port fields into the client struct; caller: MQTT_Connect_And_Subscribe | high | Re-Audit 2026-07-14 |
| `0x0802735c` | MQTT_DataPublish | Thin wrapper: builds a small stack struct and calls MQTT_Client_SendAndReceive; caller: Network_TransportDispatch (transport type 3) | high | Re-Audit 2026-07-14 |
| `0x0802f2f0` | MQTT_Telemetry_String_Formatter | Formats IP addresses and cell voltage strings via 12× snprintf for the telemetry struct; filters cell voltages <101 as invalid; caller: MQTT_Telemetry_Struct_Builder | high | Re-Audit 2026-07-14 |
| `0x0802fa18` | Config_URLSlot_AddressSelect | **Name suspicion (Re-Audit 2026-07-14):** No MQTT topic relation — depending on the index (1-4) returns one of several flash/RAM addresses, which are then erased with Flash_EraseAddressRange(…,0xfb) as an OTA/config URL slot (callers: OTA_ValidateUrlSlots, BLE_Recv_Cmd_Dispatcher, Quectel_AT_Response_Parser, MQTT_JSON_RPC_Dispatcher). More of a slot address selector than a topic builder. For a rename proposal see the session result (not applied, duplicate check pending) | medium | Re-Audit 2026-07-14 |
| `0x08032edc` | MQTT_Process_IncomingPacket | PUBLISH/PUBREC/PUBREL/PINGRESP/DISCONNECT handler | high | Re-Audit 2026-07-14 |
| `0x08033044` | MQTT_Decode_RemainingLength | Variable-length integer (max 4 bytes) | high | Re-Audit 2026-07-14 |
| `0x080330ae` | MQTT_Dispatch_PublishCallback | 5 subscription slots, wildcard match, callback | high | Re-Audit 2026-07-14 |
| `0x080353ac` | MQTT_Next_PacketId | ID++ with wrap 0xFFFF→1 (never 0, per MQTT spec) | high | Re-Audit 2026-07-14 |
| `0x08036518` | MQTT_TopicFilter_Match | '+' and '#' wildcard support, '/' separator | high | Re-Audit 2026-07-14 |
| `0x08036598` | MQTT_KeepAlive_SendPing | Send PINGREQ, disconnect on unanswered ping | high | Re-Audit 2026-07-14 |
| `0x080492bc` | MQTT_Config_ExtractFields | Iterate 6 config entries (stride 12B) | high | Re-Audit 2026-07-14 |
| `0x0804959c` | MQTT_Config_ExtractStringValue | "key=value;" parser with length limit | high | Re-Audit 2026-07-14 |
| `0x08049674` | MQTT_Config_ParseScheduleEntries | **10 schedule entries** (nm_0..nm_9): start/end, mode, power | high | Re-Audit 2026-07-14 |
| `0x08049844` | MQTT_Config_ExtractU16Value | strstr+atoi → uint16 | high | Re-Audit 2026-07-14 |
| `0x08049874` | MQTT_Config_ExtractU32Value | strstr+atoi → uint32 | high | Re-Audit 2026-07-14 |
| `0x080498a4` | MQTT_Config_ExtractU8Value | strstr+atoi → uint8 | high | Re-Audit 2026-07-14 |
| `0x0804b3f0` | MQTT_ReadByte | Read 1 byte, cursor++ | high | Re-Audit 2026-07-14 |
| `0x0804b3fe` | MQTT_ReadUint16 | Read 2 bytes big-endian | high | Re-Audit 2026-07-14 |
| `0x0804b418` | MQTT_ReadStringField | uint16 length + pointer+len store | high | Re-Audit 2026-07-14 |
| `0x0804b452` | MQTT_ReceivePacket | Read header + remaining length + payload | high | Re-Audit 2026-07-14 |
| `0x0804bcf8` | MQTT_Transport_ReceiveAll | Blocking receive loop, **5 callers**, keepalive reset | high | Re-Audit 2026-07-14 |
| `0x08050e4a` | MQTT_WaitForPacketType | Wait for a specific MQTT packet type | Batch 20 | Re-Audit 2026-07-14 |
| `0x08050e7e` | MQTT_Encode_String | Encode string into MQTT packet | Batch 20 | Re-Audit 2026-07-14 |
| `0x08050ea6` | MQTT_Encode_Byte | Encode single byte into MQTT packet | Batch 20 | Re-Audit 2026-07-14 |
| `0x08050eb2` | MQTT_Encode_Uint16 | Encode 16-bit value into MQTT packet | Batch 20 | Re-Audit 2026-07-14 |
| `0x08050ee2` | MQTT_Encode_Field | Encode generic field into MQTT packet | Batch 20 | Re-Audit 2026-07-14 |
| `0x08025eac` | Event_Params_Pack | Packs topic/payload pointers into an event struct for MQTT_Dispatch_PublishCallback (only caller) | high | Re-Audit 2026-07-14 |

## Cloud-Reporting (23)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08004808` | BLE_GATT_HexByte_Send | Name corrected (2026-07-15, formerly Cloud_Format_And_Send_Byte): nothing to do with "Cloud" — formats 1 byte as 2 ASCII hex characters (via sprintf) and sends both characters via UART_TransmitByte_Blocking; only caller is BLE_GATT_HexBuffer_Send, used exclusively from BLE_GATT_Notify_WithData/Send (BLE GATT notify payload hex encoding) | medium | Re-Audit 2026-07-15 |
| `0x08005494` | Cloud_Close_TCP_Connection | Builds "AT+QICLOSE=%d" via sprintf and sends it via Quectel_SendCmd_WaitResponse (waits for "+QISTATE"); 0 callers in the current Ghidra status (possibly only indirect/function pointer) | high | Re-Audit 2026-07-15 |
| `0x0800b7dc` | BLE_GATT_HexBuffer_Send | Name corrected (2026-07-15, formerly Cloud_Format_And_Send_Buffer): loop over buffer, calls BLE_GATT_HexByte_Send for each byte; used exclusively from BLE_GATT_Notify_WithData/Send (BLE context, no cloud relation) | medium | Re-Audit 2026-07-15 |
| `0x08014ed8` | Cloud_Report_URL_Builder | Builds report URL depending on param_1 (report type 1/2/4/5/6/8) and server type (EEPROM 0x441 = 0/1/2), calls HTTPS_POST_Request(6,...) with the built URL/payload (6 call sites in the function body) | high | Re-Audit 2026-07-15 |
| `0x08016764` | Cloud_Reporting_setVenusDReporting | Builds cloud telemetry JSON (via Cloud_Telemetry_JSON_Builder=sprintf), encrypts with AES-128-ECB (own implementation, SRAM key ROT(-6)), Base64-encodes, sends via HTTP_POST_Request to hamedata.com/setVenusDReporting; additionally publishes via MQTT (AT+QMTPUB) depending on cloud connection type (1886 bytes, 14 callees) | high | Re-Audit 2026-07-15 |
| `0x0801c278` | Cloud_Report_FillPvData | PV: voltage (fp64), SOC, power, watts | high | Re-Audit 2026-07-15 |
| `0x0801c32c` | Cloud_Report_FillBatteryStatus | Charge status (0x02=Charging) + parameters | high | Re-Audit 2026-07-15 |
| `0x0801c398` | Cloud_Report_FillDeviceInfo | "Venus_D", serial, MAC, FW version | high | Re-Audit 2026-07-15 |
| `0x0801c478` | Cloud_Report_FillEnergyCounters | Mode + 4 energy words + import/export | medium | Re-Audit 2026-07-15 |
| `0x0801c4ec` | Cloud_Report_FillGridData | AC: cell voltage, current, frequency, energy | high | Re-Audit 2026-07-15 |
| `0x0801c5e0` | Cloud_Report_FillOperatingMode | "Manual"/"Passive" + energy counters | high | Re-Audit 2026-07-15 |
| `0x0801c748` | Cloud_Report_FillPowerFlow | Real-time: battery/grid/PV power + timestamp | high | Re-Audit 2026-07-15 |
| `0x0801c88c` | Cloud_Report_FillPvStringDetails | 4 PV strings: V/I fp64-scaled, active flag | high | Re-Audit 2026-07-15 |
| `0x0801cafc` | Cloud_Report_FillNetworkConfig | Serial, MAC, 4 IPs, timezone | high | Re-Audit 2026-07-15 |
| `0x08023274` | Cloud_Handle_SelfCtl_Power | Parses "selfctl_power=" from cloud command, writes PowerOffset | high | Re-Audit 2026-07-15 |
| `0x08024128` | Cloud_HTTP_Response_Parser | Parses server response after telemetry upload: searches key=value pairs (incl. seq_check, meter, dchrg) via strstr/atoi_u16, writes various config values (Config_Write_*), calls Cloud_Response_Action on certain responses (714 bytes, 19× strstr) | high | Re-Audit 2026-07-15 |
| `0x08026aa4` | Cloud_Config_Apply | Generic key=value parser: searches param_2 in param_1 (strstr), extracts the value after it up to ','/end, filters only printable characters + dot via ctype table, max. 16 bytes output into param_3; despite the "Cloud" name widely used (IP/network config, device info, BLE dispatcher, Quectel UDP state machine — 8 callers across several clusters) | high | Re-Audit 2026-07-15 |
| `0x08029b2c` | Cloud_Response_Action | Resets power setpoint flags and calls Inverter_Power_Setpoint_Calc(1,0); param_1 controls a "was-already-connected" flag. Callers: Cloud_HTTP_Response_Parser, Cloud_EdgeDetectAndWatchdog, BLE_Recv_Cmd_Dispatcher (9 call sites) | high | Re-Audit 2026-07-15 |
| `0x0802caf4` | Cloud_EdgeDetectAndWatchdog | 2 edge detectors + 12000-tick connection watchdog | medium | Re-Audit 2026-07-15 |
| `0x0802f800` | Cloud_Telemetry_Snapshot_Build | Build JSON with "VenusD", then log | high | Re-Audit 2026-07-15 |
| `0x0802fa58` | UART_TransmitByte_Blocking | Generic blocking UART byte send (USART_SendData + 5000-iteration TX status poll); only caller is Cloud_Format_And_Send_Byte (BLE context) — name "Cloud_" misleading, see naming hypothesis at 0x08004808 | medium | Re-Audit 2026-07-15 |
| `0x080303bc` | sprintf | Name corrected (2026-07-15, formerly Cloud_Telemetry_JSON_Builder): generic sprintf/vsprintf wrapper (calls printf_Format_Engine + sprintf_Output_Char), NOT JSON- or cloud-specific — 39 callers across practically all clusters | high | Re-Audit 2026-07-15 |
| `0x0805063c` | Cloud_Telemetry_BuildTaskListJSON | Cloud telemetry: build task list as JSON | Batch 20 | Re-Audit 2026-07-15 |

## Modbus / RS485 (35)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080072a0` | RS485_UART_Send | Sends byte buffer over RS485 (TX mode + USART polling, switches back to RX afterwards) or alternatively over a CH395 TCP socket if network mode is active (flag at DAT_080072f8+3) | high | Docs (Re-Audit 2026-07-14) |
| `0x08007300` | RS485_Set_TX_Mode | Enable TX direction + settle wait | high | Docs |
| `0x0800731c` | RS485_Set_RX_Mode | Enable RX direction after transmission | high | Docs |
| `0x08007338` | RS485_UART_SendByte | Send single byte over RS485 (TX/RX switching) | high | Docs |
| `0x0801e37c` | CRC16_Modbus_Calc | Standard CRC16: init 0xFFFF, poly 0xA001 | high | Docs |
| `0x0801e3c0` | CRC16_Modbus_Incremental | CRC16 incremental (running-value update) | high | Docs |
| `0x0801e406` | CRC16_Modbus_WithSeed | CRC16 with variable seed value | high | Docs |
| `0x0801e43c` | Modbus_Dispatcher | Modbus TCP dispatcher over the CH395 Ethernet chip: reads frame from socket receive buffer, routes by FC (0x03/0x06/0x10) to TCP handlers, sends response via CH395_SPI_Send_Data; additionally processes reboot/factory-reset flags | high | Docs (Re-Audit 2026-07-14) |
| `0x0801e678` | Modbus_CRC16_Calculate | Standard CRC16 (init 0xFFFF, poly 0xA001), same algorithm as CRC16_Modbus_Calc; used not only for Modbus RTU frames but also for certificate integrity check in CH395_MQTT_Init_And_CertSetup (block sizes up to 0x6b4 bytes) | high | Docs (Re-Audit 2026-07-14) |
| `0x0801e6b0` | RS485_RTU_Frame_Dispatcher | UART receive loop (byte by byte via queue receive), CRC16 check, slave address comparison, FC routing (03→RS485_RTU_Modbus_Router, 06→RS485_FC06_WriteSingle_Handler, 10→RS485_FC10_WriteMultiple_Handler); broadcast (address 0) handled separately for FC06/FC10; processes reboot/factory-reset flags | high | Docs (Re-Audit 2026-07-14) |
| `0x0801e880` | TCP_FC06_WriteSingle_Handler | FC06 write single register for the TCP/CH395 path (reg must be >=40000, otherwise error code 2); on success echo response via CH395_SPI_Send_Data or Quectel modem as fallback | high | Docs (Re-Audit 2026-07-14) |
| `0x0801e904` | TCP_FC06_WriteSingle_Handler_B | Second TCP FC06 instance for a parallel second socket/slot (own context DAT_0801e948), same check logic as TCP_FC06_WriteSingle_Handler, but without its own response sending | high | Docs (Re-Audit 2026-07-14) |
| `0x0801e94c` | TCP_FC10_WriteMultiple_Handler | FC10 write multiple registers for the TCP/CH395 path, reg>=40000, 1-125 registers, byte-swapped values via Byte_Swap_Copy; sends response via CH395_SPI_Send_Data | high | Docs (Re-Audit 2026-07-14) |
| `0x0801ea0c` | TCP_FC10_WriteMultiple_Handler_B | Second TCP FC10 instance for a parallel second socket/slot, same check logic as TCP_FC10_WriteMultiple_Handler, but without its own response sending | high | Docs (Re-Audit 2026-07-14) |
| `0x0801eaa4` | FC03_Read_Handler | FC03 read handler for the TCP path: reg<40000 → lookup in 246-entry descriptor table (base 0x20000354, stride 0xC) + Read_Serializer; reg>=40000 → delegation to Write_Handler(param3=0) in read mode; special case reg 38000-39014 sets a flag at SRAM 0x20000ee5 | high | Docs (Re-Audit 2026-07-14) |
| `0x08028ee0` | Modbus_Parse_Response_Frame | FC01/02/04/18/41/61 response parser, address XOR validation | high | Docs |
| `0x08029b88` | RS485_Modbus_RegisterMap_Init | Initialize 8 read/write register groups | high | Docs |
| `0x08029c18` | RS485_Modbus_MapReadRegister | Build read register entry (FC 0x52) | high | Docs |
| `0x08029c48` | RS485_Modbus_MapWriteRegister | Build write register entry (FC 0x53) | high | Docs |
| `0x08029df8` | RS485_FC03_ReadWrite_Handler | FC03 handler for RS485 RTU (counterpart to FC03_Read_Handler in the TCP path), not a cross-FC router: reg<40000 reads via descriptor table+Read_Serializer, reg>=40000 via Write_Handler(param3=0); sends CRC16-protected response over RS485_UART_Send. Renamed from "RS485_RTU_Modbus_Router" on 2026-07-14 (old name wrongly implied a cross-FC router) | high | Re-Audit 2026-07-14 |
| `0x08029f54` | RS485_FC06_WriteSingle_Handler | FC06 write single register for the RS485 RTU path, reg>=40000; on success CRC16-protected echo response over RS485_UART_Send | high | Docs (Re-Audit 2026-07-14) |
| `0x08029fe0` | RS485_Broadcast_FC06_WriteSingle | FC06 write single as RS485 broadcast (slave address 0x00), reg>=40000; sends NO response (Modbus broadcast convention) | high | Docs (Re-Audit 2026-07-14) |
| `0x0802a028` | RS485_FC10_WriteMultiple_Handler | FC10 write multiple registers for the RS485 RTU path, reg>=40000, 1-125 registers, byte swap via Byte_Swap_Copy; sends CRC16-protected response over RS485_UART_Send | high | Docs (Re-Audit 2026-07-14) |
| `0x0802a108` | RS485_Broadcast_FC10_WriteMultiple | FC10 write multiple as RS485 broadcast (slave address 0x00), reg>=40000; sends NO response | high | Docs (Re-Audit 2026-07-14) |
| `0x0802c060` | Modbus_SendResponse | Takes finished response frame from queue (via queue receive function, timeout 1000 ticks) and sends it over CAN (CAN_SendMessage/CAN_SetupTxMailbox) — NOT over USART as previously documented; currently 0 direct callers in Ghidra | high | Docs (Re-Audit 2026-07-14, corrected) |
| `0x0802e94c` | Modbus_StoreRegisterSlot | FC 0x03: 8B into slot (1-7) | medium | Docs |
| `0x0802e988` | Modbus_StoreWithHandshake | FC 0xCB: 2-phase handshake | medium | Docs |
| `0x0802e9d4` | Modbus_StoreDualSlot | FC 0xCE: 4B into slot 0/1 | medium | Docs |
| `0x0802ea04` | Modbus_StoreValue16 | FC 0xFE: 16-bit single value | high | Docs |
| `0x0802ea18` | Modbus_StorePairSlot | FC 0xFF: 8B into slot (1-2) | medium | Docs |
| `0x0802ea54` | Modbus_ResponseDispatch | FC-based dispatcher: 0x03/0xCB/0xCE/0xFE/0xFF | high | Docs |
| `0x0802fc60` | RS485_SendIfValid | Guard wrapper around RS485_UART_Send: sends only if buffer pointer != 0, otherwise returns 0; currently 0 direct callers in Ghidra (possibly a function pointer target) | high | Docs (Re-Audit 2026-07-14) |
| `0x080541f8` | Queue_Receive_WithTimeout | Generic blocking FreeRTOS queue receive function (timeout via vTaskPlaceOnEventList/xTaskCheckForTimeOut, analogous to xQueueReceive); widely used by RS485_RTU_Frame_Dispatcher, CAN_RxQueue_DrainAndDispatch, Quectel_SendCmd_WaitResponse, Write_Handler, CT_GridPower_Controller and others — not a Modbus-specific function. Renamed from "Modbus_Response_Builder" on 2026-07-14 (old name wrongly implied Modbus frame building) | high | Re-Audit 2026-07-14 |
| `0x0804fe20` | Read_Serializer | Converts SRAM values via descriptor table, with correct type/scaling, into a Modbus read response (TCP FC03 + RS485 FC03) | high | Re-Audit 2026-07-14 |
| `0x08050f20` | Write_Handler | Central Modbus register write dispatcher (FC06/FC10) for TCP and RS485, full register map documented in the code comment | high | Re-Audit 2026-07-14 |

## CAN Bus / Parallel Operation (33)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08002000` | CAN_SendMessage | CAN send wrapper: calls CAN_SetupTxMailbox(CAN1_BASE=0x40006400); only caller is Modbus_SendResponse (Modbus response via CAN) — **corrected 2026-07-14**: old description "USART wrapper" was wrong, real CAN1 peripheral confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08002a20` | CAN_ReadMailbox | Read CAN RX mailbox (ID/DLC/data), set release bit | high | Docs |
| `0x08002b10` | CAN_SetupTxMailbox | Fill CAN TX mailbox (TME check, set TXRQ) | high | Docs |
| `0x08004418` | CAN_Detect_Mismatched_Nodes | Scan CAN nodes for address mismatch, error if >1 | medium | Docs |
| `0x080045dc` | CAN_Update_StateMachine | 6-state machine for CAN update with 20-tick timeout | high | Docs |
| `0x080049a4` | CAN_Update_Check_Retry_Limit | Check CAN update retry counter against limit | medium | Docs |
| `0x08005418` | CAN_Node_Data_Reset | Reset CAN node data structure to 0 | high | Docs |
| `0x0800557c` | CAN_Parallel_Inverter_Sync | Multi-unit SOC/version sync over CAN | high | Docs |
| `0x080056b4` | CAN_Select_Master_Node | Node validation: first node with a valid version/compatibility byte (range 2-4) becomes the active node, then calls CAN_Detect_Mismatched_Nodes — **corrected 2026-07-14**: no SOC comparison in the code (the SOC max search actually takes place in CAN_Parallel_Inverter_Sync) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08026f08` | CAN_Update_HandleResponse | Process CAN update response | high | Docs (name match) |
| `0x08027074` | CAN_Update_SendData | Transfer firmware data blocks over CAN | high | Docs (name match) |
| `0x080271a4` | CAN_Update_Success | Complete CAN update successfully | high | Docs (name match) |
| `0x080271e4` | CAN_Update_Failed | Handle CAN update error state | high | Docs (name match) |
| `0x08027224` | CAN_Update_Init | CAN update initialization | high | Docs (name match) |
| `0x080272e0` | CAN_Update_Erase | Send flash erase command over CAN | high | Docs (name match) |
| `0x080292d4` | CAN_RxQueue_DrainAndDispatch | Dispatch up to 64 messages from RX queue  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x0802c090` | CAN_Update_WriteStatus | Write status data to register cat. 0xCE/sub 2 | high | Docs |
| `0x0802c0c2` | CAN_Update_WriteData | Flash data block to register cat. 0xCE/sub 1 | high | Docs |
| `0x0802c0de` | CAN_Update_WriteEraseInfo | Erase metadata to register cat. 0xCE/sub 0 | high | Docs |
| `0x0802c5b0` | I2C_SendWorkModeFrame | WorkMode as 2-byte CAN frame (0x81\|mode<<4, addr 0x48) | high | Docs |
| `0x0802c5d8` | I2C_SyncChangedRegisters | Delta sync: send only changed registers over CAN (5 callers) | high | Docs |
| `0x0802cf8c` | CAN_UpdateResultHandler | Map CAN node ID → device type, OTA_Set_SlotStatus | high | Docs |
| `0x0802dfd4` | CAN_RxMailbox_Handler | CAN_ReadMailbox + forward, ISR/callback | high | Docs |
| `0x0802e698` | CAN_FrameDispatcher | Dispatch CAN frames by ID bits (3 handlers) | high | Docs |
| `0x0802e6e4` | CAN_ParallelInverterDataParser | Parse CAN frame ID, fill 8-entry table | medium | Docs |
| `0x080358ec` | CAN_Battery_Telemetry_Debug_Print | 21 CAN PGN fields (1801-1804, 1007) | high | Docs |
| `0x0802efac` | Protocol_AA_CommandDispatch | Validate 0xAA header, 17-entry function table. Caller is CAN_FrameDispatcher — name itself correct | high | Re-Audit 2026-07-15 |
| `0x0802f010` | Protocol_AA_SetChannelData | Cmd 0x10: 8B into channel slot (1-7) | high | Re-Audit 2026-07-15 |
| `0x0802f04c` | Protocol_AA_SetSystemParams | Copy 8B into system struct | medium | Re-Audit 2026-07-15 |
| `0x0802f064` | Protocol_AA_SetDeviceParams | Copy 6B into device struct | medium | Re-Audit 2026-07-15 |
| `0x0802f07c` | Protocol_AA_EnqueueCommand | Cmd 0x13: 4B+2B into two FreeRTOS queues | high | Re-Audit 2026-07-15 |
| `0x0802f0b8` | Protocol_AA_SetExtendedParams | Cmd 0x14: 6B into extended struct | medium | Re-Audit 2026-07-15 |
| `0x0802f0d0` | Protocol_AA_RS485Forward | Cmd 0xC3: RS485 passthrough only in mode 0x02 | high | Re-Audit 2026-07-15 |

## BLE / GATT (35)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08002674` | BLE_Handle_SetDischargeCutoff | Read discharge cutoff relay value from BLE payload, apply via Config_Set_DischargeCutoff_WithRelay, send WorkMode back as confirmation (Cmd 0x54) — **corrected 2026-07-14**: old description "read and apply WorkMode" was wrong, WorkMode is only read out for the reply | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080026a4` | BLE_ConnState_Timer_Control | Depending on BLE connection state (offsets +0x20e/+0x210) start/reset/delete the FreeRTOS timer "BLE_TIMER" via BLE_Timer_Manage, called from BLE_ConnectionStateManager at state 2/3 — **corrected 2026-07-14**: old description "charging mode" was wrong, no charging relation in the code | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08007cf4` | BLE_AdvertisingControl | AT+QBLEADVSTART/STOP with 3 retries | high | Docs (Re-Audit 2026-07-14) |
| `0x08007d68` | BLE_ConnectionStateManager | BLE state machine: advertising, connect, disconnect | medium | Docs (Re-Audit 2026-07-14) |
| `0x08007ef4` | BLE_Set_AdvParams | AT+QBLEADVPARAM with retry | high | Docs (Re-Audit 2026-07-14) |
| `0x08007f58` | BLE_Recv_Cmd_Dispatcher | BLE receive command dispatcher (~6.7 KB of code): validates frame format ([Header][Len]['#'][Cmd][Payload][XOR-Checksum]) via BLE_CRC_Calculate, dispatches by switch on the cmd byte to >50 handlers (server type, work status, device info, config R/W, factory reset, WLAN setup, operating mode, date/time, develop mode, BMS data, OTA, VID/XID provisioning etc.), reply via BLE_Send_Response | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800a740` | BLE_XOR_Checksum_Calculate | Name corrected (2026-07-15, formerly BLE_CRC_Calculate): computes XOR checksum (no real CRC/polynomial) over param_2 bytes starting at param_1 — used by BLE_Send_Response and BLE_Recv_Cmd_Dispatcher for frame validation | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800a764` | BLE_Send_Response | Builds BLE response frame (header 0x73, length=payload+5, '#', cmd byte, payload, XOR checksum via BLE_CRC_Calculate) and sends via BLE_GATT_Notify_Send; only for payload!=0 and <0x1FC bytes. Central response function of the BLE protocol (>80 callers) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800a7c0` | BLE_Poll_Connection_State | Poll AT+QBLESTAT, 6 states, 9×500ms | high | Docs (Re-Audit 2026-07-14) |
| `0x0800a9b8` | BLE_Recv_Extract_URL | Extracts URL string (from offset 5) from BLE packet into global buffer; two modes: param_2=0 URL only, param_2=1 URL incl. port field (for cloud URL or cloud URL+port respectively) | high | Docs (Re-Audit 2026-07-14) |
| `0x0800aa2c` | BLE_Recv_Parse_WiFi_Credentials | Parse SSID+password from BLE, `<,>` delimiter | high | Docs (Re-Audit 2026-07-14) |
| `0x0800ab14` | BLE_Module_Init | AT+QBLEINIT with retry | high | Docs (Re-Audit 2026-07-14) |
| `0x0800abb8` | BLE_Set_Device_Name | AT+QBLENAME=MST_VNSD_xxxx (serial number) | high | Docs (Re-Audit 2026-07-14) |
| `0x0800acc4` | BLE_GATT_Notify_Send | Waits for queue release, sends AT+QBLEGATTSNTFY=ff02,<len> via Quectel AT over UART to the BLE module (real GATT notify, characteristic handle 0xFF02) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800ad40` | BLE_GATT_Notify_WithData | Send GATT notify with data payload | high | Docs (Re-Audit 2026-07-14) |
| `0x0800ae5c` | BLE_Build_Settings_Response | Prepare 10 settings entries with byte swap | medium | Docs (Re-Audit 2026-07-14) |
| `0x0800af78` | BLE_RuntimeInfo_Builder | Builds work status response (0xA4 bytes, Cmd 0x03) from meter/CT/telemetry data incl. fp64 power calculations | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800b430` | BLE_Build_BMS_Data_Response | Prepare BMS data (V, I, SOC, temp, cells, fp64 power calculation) in a shared buffer — used both for the BLE response (BLE_Recv_Cmd_Dispatcher) and for MQTT publish (MQTT_Publish_BMS_Full_Data) — corrected/extended 2026-07-14 | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800b6c0` | BLE_Build_DevelopModeInfo_Response | BLE response: assemble developer mode info | Batch 20 | Ghidra (Batch 20, Re-Audit 2026-07-14) |
| `0x08010518` | BLE_Build_Telemetry_Response | Inverter telemetry for BLE (fp64 power calculation) | high | Docs (Re-Audit 2026-07-14) |
| `0x08010718` | BLE_Pending_Commands_Process | Deferred BLE commands: WiFi setup, OTA, socket control (calls WiFi_Set_Credentials, Quectel_OTA_StartUpdate_FromURL_AndReboot, BLE_ConnectionStateManager, BLE_MQTT_Command_Execute). Note: 0 callers/references found in the current code path (possibly reachable only via an unresolved function pointer) — added 2026-07-14 | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08012e04` | BLE_RuntimeInfo_ChannelDelta_Calc | BLE runtime info: compute channel delta | Batch 20 | Ghidra (Batch 20, Re-Audit 2026-07-14) |
| `0x0801e08c` | BLE_Timer_Manage | FreeRTOS timer: create/reset/delete "BLE_TIMER" | high | Docs (Re-Audit 2026-07-14) |
| `0x08025204` | BLE_MQTT_Command_Execute | BLE-triggered MQTT actions: VID/XID subscribe+publish with ack wait time, device ID encryption; caller BLE_Pending_Commands_Process | high | Ghidra (Re-Audit 2026-07-14, Docs confirmed) |
| `0x08025f94` | BLE_SendFramedNotification | Builds reply frame (header 0x73, length, cmd byte, payload, XOR checksum via Util_XOR_Checksum_Calc) and sends via BLE_GATT_Notify_WithData; used by OTA cmds 0x3A/0x50/0x51/0x52 and reboot cmd 0x23 | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08026bfc` | BLE_XidServerConfig_Parse | Parses XID/server config string (delimiter `<,>`) from BLE command 0x0C (VID/XID provisioning) into 6 fields: ID (1B), host (32B), topic (64B), port (2B), user (64B), password (64B); return 1=success/0=error. Caller: BLE_Recv_Cmd_Dispatcher | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802d0ec` | BLE_Apply_ConfigReg_Offset3 | Map BLE index 0/1 → config register 3/4 | high | Docs (Re-Audit 2026-07-14) |
| `0x0802e78c` | BLE_Cmd_OTA_Validate | Cmd 0x52: validate model/magic/size/CRC | high | Docs (Re-Audit 2026-07-14) |
| `0x0802e86c` | BLE_Cmd_OTA_Init | Cmd 0x3a: initialize OTA context (0x3C bytes) | high | Docs (Re-Audit 2026-07-14) |
| `0x0802e900` | BLE_Cmd_OTA_WriteSetup | Cmd 0x50: set flash address/size, erase flash if needed | high | Docs (Re-Audit 2026-07-14) |
| `0x0802eacc` | BLE_Cmd_SystemReboot | Cmd 0x23: shutdown + reboot, BLE response 0xAA01 | high | Docs (Re-Audit 2026-07-14) |
| `0x0802ed04` | BLE_OTA_WriteDataChunk | Cmd 0x51: receive 128B chunks, CRC16, write flash | high | Docs (Re-Audit 2026-07-14) |
| `0x0803216c` | BLE_Set_Mode_Persist | BLE OFF/Ready/ON → EEPROM 0x36a | very high | Docs (Re-Audit 2026-07-14) |
| `0x08004844` | Serial_Packet_Validate | Magic 's'+0x10, max 138B, single-byte XOR checksum | high | Docs |
| `0x08025eb2` | Serial_Command_Dispatch | Distributes BLE module commands (OTA init/write/validate, reboot) over the serial interface; 0 direct callers (presumably UART RX callback) | medium | Re-Audit 2026-07-14 |

## CH395 — Ethernet controller (55)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08002c38` | CH395_SPI_Cmd_WithData | Mutex-protected SPI command with data | high | Re-audit 2026-07-14 |
| `0x08002e10` | CH395_SPI_ReadByte | Mutex-protected single-byte SPI read | high | Re-audit 2026-07-14 |
| `0x08002ea4` | CH395_SPI_WriteCmd | Mutex-protected SPI write (cmd 0x2E) | high | Re-audit 2026-07-14 |
| `0x08002f38` | CH395_SPI_CmdWaitReady | SPI cmd + busy poll (201×5ms, 0xFA on timeout) | high | Re-audit 2026-07-14 |
| `0x08003008` | CH395_GetICVersion | CMD 0x2C: read chip version | high | Re-audit 2026-07-14 |
| `0x08003164` | CH395_ReadRecvBuf | CMD 0x3C: read receive buffer (socket+length→buffer) | high | Re-audit 2026-07-14 |
| `0x0800322c` | CH395_GetRecvLen | CMD 0x3B: query receive buffer length (16-bit) | high | Re-audit 2026-07-14 |
| `0x080032d4` | CH395_GetSocketStatus | CMD 0x30: read socket status | high | Re-audit 2026-07-14 |
| `0x08003378` | CH395_SetProtoType | CMD 0x58: set protocol type | high | Re-audit 2026-07-14 |
| `0x0800340c` | CH395_SetRetranPeriod | CMD 0x56: retransmission timeout (32-bit ms) | high | Re-audit 2026-07-14 |
| `0x080034b8` | CH395_SetRetranCount | CMD 0x57: retransmission count (32-bit) | high | Re-audit 2026-07-14 |
| `0x08003564` | CH395_OpenSocket | CMD 0x35: open socket + busy poll | high | Re-audit 2026-07-14 |
| `0x08003644` | CH395_SPI_Send_Data | CMD 0x39: core send function, polls the socket busy flag (max. 10×3ms), mutex-protected, sends byte by byte via SPI_WriteByte; called by almost all other CH395 send paths (Send_Data_Verified, Socket_SendSafe, Socket_SendData_ViaSocketPtr, DNS/Modbus/TCP handlers) | high | Re-audit 2026-07-14 |
| `0x08003740` | CH395_SetKeepAlive | CMD 0x59: keep-alive on/off | medium | Re-audit 2026-07-14 |
| `0x080037dc` | CH395_SetDestIP | CMD 0x31: set destination IP (4 bytes) | high | Re-audit 2026-07-14 |
| `0x08003894` | CH395_SetSocketSrcPort | CMD 0x32: set source port (16-bit lo/hi) | high | Re-audit 2026-07-14 |
| `0x08003938` | CH395_SetSocketProtocol | CMD 0x34: socket protocol (2=TCP, 3=UDP) | high | Re-audit 2026-07-14 |
| `0x080039d4` | CH395_SetSocketDstPort | CMD 0x33: set destination port (16-bit lo/hi) | high | Re-audit 2026-07-14 |
| `0x08003a78` | CH395_TCPConnect | CMD 0x37: initiate TCP connection + poll | high | Re-audit 2026-07-14 |
| `0x08003b48` | CH395_TCPDisconnect | CMD 0x38: disconnect TCP connection + poll | high | Re-audit 2026-07-14 |
| `0x08003c18` | CH395_TCPListen | CMD 0x36: TCP server listen + poll | high | Re-audit 2026-07-14 |
| `0x08003ce8` | CH395_ConfigAndSendSocket | Orchestrator: DestIP+SrcPort+send for socket | high | Re-audit 2026-07-14 |
| `0x08003d18` | CH395_HardwareReset | GPIO reset pulse (100ms low, 50ms high) | high | Re-audit 2026-07-14 |
| `0x080066be` | CH395_MQTT_Socket_Close_And_TLS_Reset | Close CH395 MQTT socket and reset TLS context | high | Re-audit 2026-07-14 |
| `0x080067b4` | CH395_Wait_Command_Complete | CH395 command completion with 5s timeout | high | Re-audit 2026-07-14 |
| `0x080074e4` | CH395_SendData_WaitResponse | Send data over CH395 socket, wait for response | high | Re-audit 2026-07-14 |
| `0x08015638` | CH395_SPI_SendData_Verified | CH395 SPI data transmission with verification | high | Re-audit 2026-07-14 |
| `0x08017974` | CH395_UDP_Socket_Init | Initialize CH395 UDP socket | high | Re-audit 2026-07-14 |
| `0x08017c50` | CH395_TCP_Socket_Open_HTTPPort80 | Open CH395 TCP socket on port 80 (HTTP) | high | Re-audit 2026-07-14 |
| `0x08017e64` | CH395_UDP_Socket_OpenForDNS | Open CH395 UDP socket for DNS queries | high | Re-audit 2026-07-14 |
| `0x0801828c` | CH395_Socket_Dest_Config | Configure socket with IP + port 8883 (MQTTS) | high | Re-audit 2026-07-14 |
| `0x08019560` | CH395_UDP_Socket_Config | CH395 socket: protocol=UDP, index=4 | medium | Re-audit 2026-07-14 |
| `0x080195a4` | CH395_Recv_Buffer_Setup | No SPI/HW access: sets a field in the CH395 socket descriptor and uses it to initialize a CLI/AT command session (`CLI_InitSession`, 512B buffer in 5 slices, lookup via version string "VNSD_0_v1492") — see the section on the CLI/AT command engine (0x0804bd58-0x0804cc40). **0 callers/xrefs found (dead code or referenced only indirectly)**; the description "DMA Receive Buffer" was wrong, there is no DMA usage | medium | Re-audit 2026-07-14 |
| `0x0801ded4` | CH395_Command_SendAndWait | CH395 cmd + poll with vTaskDelay(10) | high | Re-audit 2026-07-14 |
| `0x08024d80` | CH395_MQTT_Init_And_CertSetup | Orchestrates MQTT init via CH395: TLS certificate decryption (TLS_Cert_Decrypt_All), MQTT session setup, CRC16 of the certificates in EEPROM 0x36B7-0x36BB (per existing Ghidra comment); calls, among others, CH395_Command_SendAndWait and CH395_MQTT_Socket_Close_And_TLS_Reset | high | Re-audit 2026-07-14 |
| `0x08029314` | CH395_Receive_WithRetry | 6× poll (100ms), 512B buffer, return 0/-1/-2 | high | Re-audit 2026-07-14 |
| `0x08029918` | CH395_Init_Full | Mode 0x41, 4K clear, IC version poll (201×20ms) | high | Re-audit 2026-07-14 |
| `0x08029964` | CH395_Reset_And_Reinit | FreeRTOS queue sync, HW reset, basic→full init | high | Re-audit 2026-07-14 |
| `0x08029a24` | CH395_Init_Basic | Mode 0x27, IC version poll (soft init) | high | Re-audit 2026-07-14 |
| `0x0802d12c` | CH395_Socket_SendData | 0x39 header, mutex-protected, 20000-iteration timeout | high | Re-audit 2026-07-14 |
| `0x0802e004` | CH395_UDP_DataHandler | Parse UDP packet, validate IP, forward to JSON-RPC | high | Re-audit 2026-07-14 |
| `0x0802e190` | CH395_UDP_ServerTask | 3-state UDP server: start→receive→close | high | Re-audit 2026-07-14 |
| `0x08032634` | CH395_ConfigTcpKeepalive_Defaults | UDP: RetranPeriod=10s, RetranCount=3000 | high | Re-audit 2026-07-14 |
| `0x08032650` | CH395_PHY_StatusHandler | Link status: 10M/100M full/half, auto-neg | high | Re-audit 2026-07-14 |
| `0x08032978` | CH395_Socket_WaitReady | Poll socket ready via SPI | high | Re-audit 2026-07-14 |
| `0x08032984` | CH395_Socket_ReadData | Non-blocking receive with length limit | high | Re-audit 2026-07-14 |
| `0x080329b2` | CH395_Socket_SendData_ViaSocketPtr | Thin wrapper: dereferences the socket pointer (byte 0 = socket index) and calls CH395_SPI_Send_Data directly, no mutex/polling of its own (the mutex is inside Send_Data itself); 0 callers in the current status (see batch 19 duplicate fix) | high | Re-audit 2026-07-14 |
| `0x080329cc` | CH395_Socket_RecvBlocking | Blocking receive, 1s timeout | high | Re-audit 2026-07-14 |
| `0x08032a44` | CH395_Socket_SendSafe | Guarded send with NULL/0 check | high | Re-audit 2026-07-14 |
| `0x08032c0c` | CH395_Init_TCPServer_Socket | Port 8091, type=TCP listen | high | Re-audit 2026-07-14 |
| `0x08032c58` | CH395_Socket_Open_ByDescriptor | **8 callers** — central socket opener (UDP/TCP client/server) | high | Re-audit 2026-07-14 |
| `0x08032d78` | CH395_Debug_PrintVersion | Read and print IC version (debug) | high | Re-audit 2026-07-14 |
| `0x08032dd4` | CH395_Init_BroadcastListener_Socket | Port 8090, dest=255.255.255.255 | high | Re-audit 2026-07-14 |
| `0x08048754` | CH395_Init_Modbus_TCP_Socket | UDP socket port 502, buffer 0x80, **located between mbedTLS code** | high | Re-audit 2026-07-14 |
| `0x08004838` | CH395_Debug_SendCmdByte | Only calls CH395_SPI_Cmd_WithData(param_1); sole caller is CH395_Debug_PrintVersion (CH395 chip version query) | high | Re-audit 2026-07-14 |

## Network / DNS / sockets (general) (23)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08003d40` | Network_ProtocolDispatcher | Dispatches 3 handlers (mode 0/1/2), checks CH395 link, 60s timeout | medium | Re-audit 2026-07-15 |
| `0x0800670c` | DNS_ResourceRecord_Parse | Decode TLV record, type switch (1-16), recursive  — **corrected 2026-07-09**, see batch 18 | medium | Re-audit 2026-07-15 |
| `0x0800b5bc` | TCP_Socket_Close | AT+QICLOSE with socket ID validation (Quectel AT path, 3 retries via Quectel_TCP_SendAndVerify); sole caller is Quectel_UDP_CommStateMachine | high | Re-audit 2026-07-15 |
| `0x08011258` | Network_ConnectionStatus_Fill | Fill network status struct (type, signal, mode) | medium | Re-audit 2026-07-15 |
| `0x080130f4` | Network_BroadcastAddr_ComputeAndLog | Compute and log broadcast address | Batch 20 | Re-audit 2026-07-15 |
| `0x080131b0` | Network_BroadcastAddr_ComputeInPlace | Compute broadcast address in place | Batch 20 | Re-audit 2026-07-15 |
| `0x080136e0` | Network_ExtractIPv4_BetweenTags | Extract IPv4 address between text tags | Batch 20 | Re-audit 2026-07-15 |
| `0x0801379c` | Modem_ConnInfo_UpdateMeterIP | Modem connection info: update meter IP | Batch 20 | Re-audit 2026-07-15 |
| `0x080138c8` | Ethernet_ConnInfo_UpdateMeterIP | Ethernet connection info: update meter IP | Batch 20 | Re-audit 2026-07-15 |
| `0x08017dc0` | Network_LANDiscovery_SocketInit | Initialize socket for LAN discovery | Batch 20 | Re-audit 2026-07-15 |
| `0x080195c8` | Network_Filter_Rule_Init | 16B filter struct: address, mask, type, 0xFFFF | medium | Re-audit 2026-07-15 |
| `0x0801d2f0` | Protocol_Set_ResponseBytes | Cmd byte + 2 status bytes into response | medium | Re-audit 2026-07-15 |
| `0x0801df8c` | DNS_Query_Build | DNS wire format: label split on dots, 512B | high | Re-audit 2026-07-15 |
| `0x08026114` | Network_HeartbeatHandler | Opens CH395 socket via descriptor (keep-alive/heartbeat); increments error counter (+10) on failure, on success sets state=2 and resets the error counter. Caller: Network_ProtocolDispatcher (mode 1) | high | Re-audit 2026-07-15 |
| `0x080267e0` | DNS_Name_Decompress | DNS name decompression (label pointer) | high | Re-audit 2026-07-15 |
| `0x0802c120` | DNS_Query_Send | Build DNS query and send via CH395 SPI | high | Re-audit 2026-07-15 |
| `0x0802c438` | Ethernet_SendStatusTelemetry | Send JSON status via CH395 Ethernet | high | Re-audit 2026-07-15 |
| `0x0802c52c` | Network_TransportDispatch | Router: 0=Quectel, 1=unknown, 2=CH395, 3=MQTT | high | Re-audit 2026-07-15 |
| `0x08034f20` | Protocol_Reset_Counter | Clear 16-bit field at offset 0x100; sole caller is CH395_Init_TCPServer_Socket | medium | Re-audit 2026-07-15 |
| `0x080491c4` | DNS_ParseResponseHeader | 12B header + QD skip + AN parse, 0 callers (fn pointer) | high | Re-audit 2026-07-15 |
| `0x0804b1e6` | DNS_WriteUint16BE | 16-bit big-endian write, 8× from DNS_Query_Build | high | Re-audit 2026-07-15 |
| `0x0804d4e8` | Network_ReceiveAndDispatchData | Receives bytes via CH395 Ethernet (SPI, mode flag=1) or FreeRTOS queue (mode flag=0) and forwards them byte by byte to CLI_DispatchInputByte — network-based CLI/console interface; detects Ctrl-C (0x03) to reset the log level | high | Re-audit 2026-07-15 |
| `0x08004888` | Device_Network_Info_Init | Initialize network broadcast info, ASCII validation (0 direct callers according to Ghidra — presumably via function table/init sequence) | medium | Re-audit 2026-07-14 |

## Quectel modem / WiFi / AT commands (60)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08004234` | AT_Copy_MPPT_Channel_Data | Copy MPPT channel data (SOC/power/energy) into AT response | high | Re-audit 2026-07-14 |
| `0x0800736c` | Quectel_SendCmd_WaitResponse | Send AT command, mutex, match response pattern | high | Re-audit 2026-07-14 |
| `0x08007968` | Quectel_AT_QISEND_HexData | Formats payload data as hex ASCII (%02X) into an AT+QISEND=&lt;id&gt;,&lt;len&gt; command, sends via Quectel_AT_SendRaw_WithMutex with up to 3 retries (used by the Modbus TCP FC03/FC06 handlers for the cellular response) | high | Re-audit 2026-07-14 |
| `0x08007a18` | Quectel_AT_SendAndVerify | Core AT executor (19 callers), strstr verify | high | Re-audit 2026-07-14 |
| `0x08007b5c` | Quectel_AT_SendAndPollResponse | AT executor with tick-based polling timeout | high | Re-audit 2026-07-14 |
| `0x0800b810` | Quectel_OTA_StartUpdate_AndReboot | Start modem OTA update and reboot | high | Re-audit 2026-07-14 |
| `0x0800b90c` | Quectel_OTA_StartUpdate_FromURL_AndReboot | Start modem OTA update from URL and reboot | high | Re-audit 2026-07-14 |
| `0x0800babc` | Quectel_AT_QVERSION_Query | AT+QVERSION firmware version query | high | Re-audit 2026-07-14 |
| `0x0800bb30` | Quectel_Modem_HardwareReset_Handler | Perform hardware reset of the Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800be00` | Quectel_HTTP_GET_Request | Send HTTP GET request via Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800bfdc` | Quectel_UART_WriteCommand_Guarded | Send AT command guarded via UART | high | Re-audit 2026-07-14 |
| `0x0800c060` | Quectel_HTTP_ReadResponse | Read HTTP response from the Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800c184` | Quectel_HTTP_Config_SSLCtxId | Configure SSL context ID for Quectel HTTP | high | Re-audit 2026-07-14 |
| `0x0800df04` | Quectel_MQTT_Close_Connection | Close MQTT connection via Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800e598` | Quectel_MQTT_Connect | Establish MQTT connection via Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800e684` | Quectel_MQTT_Config_DataType | Configure MQTT data type on the Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800e6f4` | Quectel_MQTT_Unsubscribe | End MQTT topic subscription via Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800ebfc` | Quectel_MQTT_Publish_AndWaitAck | MQTT publish via Quectel modem, wait for ACK | high | Re-audit 2026-07-14 |
| `0x0800ed24` | Quectel_MQTT_Publish_ViaTransportDispatch | MQTT publish via transport dispatch layer | high | Re-audit 2026-07-14 |
| `0x0800eea0` | Quectel_MQTT_Subscribe | Subscribe to MQTT topic via Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800f040` | Quectel_MQTT_WaitForAck | Wait for MQTT ACK from the Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800f098` | Quectel_MQTT_Config_Version | Configure MQTT protocol version on the Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800f108` | Quectel_UDP_OpenSocket | Open UDP socket via Quectel modem | high | Re-audit 2026-07-14 |
| `0x0800fadc` | Quectel_UDP_ParseRecvURC_IPPort | Parse IP/port from UDP receive URC | high | Re-audit 2026-07-14 |
| `0x0800fc3c` | WiFi_Set_Credentials | Set WiFi SSID+PW via Quectel, EEPROM 0x400/0x420 | high | Re-audit 2026-07-14 |
| `0x0800fd2c` | Quectel_SSL_Certificate_Manage | AT+QSSLCERT: upload/delete/query/list certificates | high | Re-audit 2026-07-14 |
| `0x0800fe6c` | Quectel_Send_CertData_WithMutex | Sends a string with queue mutex protection via Quectel_AT_Send_Cmd, followed by delay(param_3). Generic send wrapper; cert relation evidenced only via the debug string "Cert info: %d, %s", no direct caller found in the current disassembly (0 callers) | medium | Re-audit 2026-07-14 |
| `0x0800fed4` | Quectel_SSL_Set_CipherSuite | AT+QSSLCFG configure cipher suite | high | Re-audit 2026-07-14 |
| `0x080106dc` | Quectel_UART_Send_Data | Thin wrapper for HAL UART transmit | medium | Re-audit 2026-07-14 |
| `0x080106f4` | Quectel_AT_Send_Cmd | Generic UART send function (strlen check + UART_TransmitBytes), without mutex/response handling. Base building block for AT commands, MQTT publish strings and BLE GATT notify (16 callers, including all MQTT_Publish_* functions) | high | Re-audit 2026-07-14 |
| `0x08010834` | Quectel_AT_SendRaw_WithMutex | Send raw AT command with queue mutex | high | Re-audit 2026-07-14 |
| `0x08010904` | Quectel_UART_RxByte_Parser | Byte parser for UART receive, \r\n termination | high | Re-audit 2026-07-14 |
| `0x08010b84` | Quectel_TCP_ConnectionState_Query | Query AT+QISTATE=&lt;id&gt;, check response for "+QISTATE: 3" (connect-ID-based). Also used by Quectel_UDP_CommStateMachine to query the UDP session (ID 3) — not TCP-exclusive | high | Re-audit 2026-07-14 |
| `0x08010c04` | Quectel_TCP_SendAndVerify | Second generic AT send+verify core (own mutex/buffer, notify-based, strstr verify on 2 patterns). 4 callers confirmed, one of them Quectel_UDP_OpenSocket — used for both TCP and UDP socket setup | high | Re-audit 2026-07-14 |
| `0x08010d40` | Quectel_UDP_DataReceive | Parse UDP data, dispatch to MQTT JSON RPC | high | Re-audit 2026-07-14 |
| `0x08010eb0` | Quectel_URC_JsonFrameParser | JSON frame assembler with brace-depth tracking | high | Re-audit 2026-07-14 |
| `0x08011008` | Quectel_QISEND_SendData_TCP_UDP | Format AT+QISEND, 3 retries | high | Re-audit 2026-07-14 |
| `0x080110cc` | Quectel_UDP_CommStateMachine | 4-state UDP lifecycle (open→verify→recv→close) | high | Re-audit 2026-07-14 |
| `0x080112d4` | Quectel_URC_LineParser | Collects URC lines introduced by "+" up to \n. After detecting "+Q", if the third character is 'I', 'W' or 'G' (i.e. +QIURC/+QWIFI.../+QGxxx) the buffer is discarded — these URCs are handled by dedicated parsers; only other +Q* URCs are collected completely here | high | Re-audit 2026-07-14 |
| `0x080113b0` | Quectel_AT_SetURCConfig | Sends "AT+QURCCFG=%d" (single integer parameter, no quoted urcport/uart string syntax as with the standard QURCCFG). No caller found in the disassembly (0 callers) — the concrete meaning of param_1 cannot be verified conclusively | medium | Re-audit 2026-07-14 |
| `0x08011c94` | Quectel_UART_WriteCommand | Core UART write: bytes + \r\n (6 callers) | high | Re-audit 2026-07-14 |
| `0x08011f38` | Quectel_WiFi_SetAPConfig | AT+QSTAAPINFODEF, SSID/PW validation | high | Re-audit 2026-07-14 |
| `0x08012068` | Quectel_WiFi_SetSTAInfo_NoSave | Set WiFi STA info, without saving | Batch 20 | Re-audit 2026-07-14 |
| `0x0801219c` | Quectel_WiFi_QueryConnectionState | Query WiFi connection status | Batch 20 | Re-audit 2026-07-14 |
| `0x0801285c` | Inverter_PVString_ChannelValue_PeriodicCheck_Save | Name corrected (2026-07-15, formerly Quectel_SignalQuality_PeriodicCheck_Save) — no AT/UART/modem relation; periodically (500ms) checks four PV string/MPPT channel values and a status field of the inverter telemetry (struct 0x20014f40/0x20014e90) against a threshold and saves a configuration parameter on change (Config_Write_ParamPair). Sole caller: MainLoop_Periodic_Tasks, in the vicinity of purely energy/inverter functions | Batch 20 | Re-audit 2026-07-14 |
| `0x08013b18` | Quectel_AT_Recv_ByteAssembler | Assemble AT response byte by byte | Batch 20 | Re-audit 2026-07-14 |
| `0x0801ce8c` | WiFi_Module_RestartStateMachine | 3-state: trigger→cmd→completion (999 tick timeout) | medium | Re-audit 2026-07-14 |
| `0x0801cf1c` | WiFi_HardwareResetSequence | 5-state GPIO toggle (pin 0x400, 30-tick intervals) | medium | Re-audit 2026-07-14 |
| `0x0801d034` | WiFi_ModuleResetDispatcher | State dispatcher: GPIO clear/HW reset/power cycle | medium | Re-audit 2026-07-14 |
| `0x0801d168` | WiFi_ResetWithRecoveryWait | 2-phase GPIO toggle (0x400) + 300-tick recovery with enable gate; on completion possibly triggers state 4 in a linked state structure. Note: no caller found in the static call graph | medium | Re-audit 2026-07-14 |
| `0x0801d238` | WiFi_PowerCycleSequence | GPIO 0x400 power cycle + 300 tick wait | medium | Re-audit 2026-07-14 |
| `0x08022b4c` | Quectel_MQTT_OTA_Info_Parser | Parses a "key=value" string received via MQTT (strtok tokenizer) for up to 4 device slots (index via embedded "d" field, 1..4) with fields mod/type/size/crc/url_len/url (URL up to 0xE6 bytes, possibly assembled across several tokens via strncat). After a complete parse pass (strtok returns NULL), the OTA target data for each filled slot is written to flash via Config_URLSlot_AddressSelect + Flash_EraseAddressRange + Flash_Write_Protected and read back via Flash_Read_Protected for verification. No caller found in the disassembly (0 callers), presumably a callback/function pointer from the MQTT JSON-RPC dispatch | high | Re-audit 2026-07-14 |
| `0x08024624` | Quectel_AT_Command_Builder | Builds the MQTT payload in "cd;p1…p7" format and sends it either as raw data (Quectel_Modem_DataSend) or as an AT+QMTPUB command (Quectel_AT_Send_Cmd) via the Quectel modem | high | Re-audit 2026-07-14 |
| `0x08026b40` | Modem_Response_Dispatch | Evaluates the meter type and routes to Modem_ParseThreePhaseActivePower / Modem_ParseActivePower / AT_Response_Parser. Used by both the modem path (Modem_SendStatusTelemetry) and the Ethernet path (Ethernet_SendStatusTelemetry) — transport-independent | high | Re-audit 2026-07-14 |
| `0x08027324` | Quectel_Modem_DataSend | Sends raw data via the Quectel modem (MQTT_Client_SendAndReceive); central send function for all 13 MQTT_Publish_* callers | high | Re-audit 2026-07-14 |
| `0x08027394` | Modem_QueueSendMessage | Packs 4 parameters into a queue message and sends it to the modem send queue; consumed by Quectel_AT_Command_Builder via xQueueReceive. Caller: Register_WriteValue | high | Re-audit 2026-07-14 |
| `0x0802c29c` | Modem_SendStatusTelemetry | Send JSON status via Quectel 4G modem TCP | high | Re-audit 2026-07-14 |
| `0x0802e35c` | AT_Response_Parser | Validates the checksum (XOR) of a device response, extracts device ID/FW version/MPPT channel data and writes them into the output structure. Default branch of Modem_Response_Dispatch (meter type ≠ EM/EM1) | high | Re-audit 2026-07-14 |
| `0x0802edfc` | Modem_ParseActivePower | Parse "act_power" from modem response (single-phase) | high | Re-audit 2026-07-14 |
| `0x0802ee7c` | Modem_ParseThreePhaseActivePower | Parse a/b/c/total_act_power (three-phase) | high | Re-audit 2026-07-14 |

## Inverter / register / energy logic (94)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08003e54` | SMR_TIC_SelfUse_PowerController | 6-state control loop for self-consumption optimization: CT power vs. reference value, threshold 40W (0x28), damping /2, states INIT→MEASURE→ADJUST→STABLE→CHECK→SAMPLE; calls Inverter_Power_Setpoint_Calc. Sole caller: CT_PowerSetpoint_Compute at WorkMode 0x0A | high | Docs (re-audit 2026-07-14) |
| `0x0800434c` | Power_Direction_Change_Check | Charge/discharge direction change with 10-tick debounce | medium | Docs (re-audit 2026-07-14) |
| `0x08004490` | Battery_Charge_Power_Limiter | Charge power limiting based on PV yield/BMS current limit (no SOC reference in the code, fp64, Power_Limit_Clamp) — details: analysis §13.4 | high | Docs (re-audit 2026-07-14) |
| `0x080051bc` | Register_Address_Range_Check | Check register address against valid range | medium | Docs (re-audit 2026-07-14) |
| `0x08005344` | Battery_Forced_Charge_Check | Check whether forced charging should be active | high | Docs (re-audit 2026-07-14) |
| `0x08005860` | MQTT_Credential_Buffer_Decode | Name corrected (2026-07-15, formerly Inverter_Register_Buffer_Init): sole caller is MQTT_Connect_And_Subscribe (no RS485/register relation). The function does memset+memcpy and calls Flash_Obfuscated_String_Decode() — decodes an obfuscated flash string (presumably MQTT/TLS credential) into a buffer for the MQTT client setup | low | Ghidra (re-audit 2026-07-14) |
| `0x08005a7c` | Inverter_RegDirty_MarkAll | Sets or clears (depending on the parameter) all 24 dirty bits in the bitfield via Timeslot_Bitmap_SetClear(0,0x18,param_1) + I2C sync — refined 2026-07-14 | high | Docs (re-audit 2026-07-14) |
| `0x08005a90` | Inverter_Power_Setpoint_Apply | Write power setpoint to register + mark dirty | high | Docs (re-audit 2026-07-14) |
| `0x08005abc` | Inverter_RegDirty_Mark_ChgVolt | Set or clear dirty bit for charge voltage (depending on the parameter) — refined 2026-07-14 | medium | Docs (re-audit 2026-07-14) |
| `0x08005acc` | Inverter_RegDirty_Mark_DischgVolt | Set or clear dirty bit for discharge voltage (depending on the parameter) — refined 2026-07-14 (2026-10-02, v151: the bitmap is the front-panel LED bitmap of a TM1650-type driver — bit-banged I2C to registers 0x68/0x6A/0x6C/0x6E — not inverter registers; renamed `Inverter_RegDirty_Mark_GridPermit`, driven by g_gridPermit == 0) | medium | Docs (re-audit 2026-07-14) |
| `0x08005adc` | Inverter_Sync_Periodic | Name corrected (2026-07-15, formerly Inverter_Sync_Init): called on every pass from MainLoop_Periodic_Tasks (not a pure one-time init): on first run MarkAll(1), afterwards continuously Timeslot_ApplyConfigOnSync + Inverter_Apply_BatteryParams + 3000-tick WorkMode frame + I2C sync | medium | Docs (re-audit 2026-07-14) |
| `0x08005b7c` | Inverter_RS485_Cmd_WorkMode | Set RS485 register 0x06 (work mode) | high | Docs (re-audit 2026-07-14) |
| `0x08005ba0` | Inverter_RS485_Cmd_Reset | Set RS485 register 0xFF (reset) | high | Docs (re-audit 2026-07-14) |
| `0x08005bc4` | Inverter_Set_Reg_0xFE | Write RS485 register 0xFE | medium | Docs (re-audit 2026-07-14) |
| `0x08005be8` | Inverter_RS485_Command_Send | Sends RS485 command to the inverter: writes register cat.2/subindex 0 (4B, on-off) and cat.2/subindex 1 (8B, power+flags) via Register_PackDescriptor/Register_WriteValue. Callers: Grid_Export_Power_Limiter (4x), Write_Handler | high | Docs (re-audit 2026-07-14) |
| `0x08005c84` | Inverter_Set_Reg_0x04 | Write RS485 register 0x04 | medium | Docs (re-audit 2026-07-14) |
| `0x08005ca8` | Inverter_Set_Reg_0xFB | Write RS485 register 0xFB | medium | Docs (re-audit 2026-07-14) |
| `0x08005ccc` | Inverter_Set_OnOff_Reg_0x01 | Inverter on/off via RS485 register 0x01 | high | Docs (re-audit 2026-07-14) |
| `0x08005d00` | Inverter_RegDirty_Mark_BatteryParams | Name corrected (2026-07-15, formerly Timeslot_Bitmap_Set_Slot2): sets/clears bit 2/3 via Inverter_RegDirty_Bitmap_SetClear(2,2,param_1). Sole caller Inverter_Apply_BatteryParams (battery parameter dirty flag) — no schedule relation despite the former name "Timeslot" (2026-10-02, v151: front-panel LED bitmap of a TM1650-type driver, not inverter registers; renamed `Inverter_RegDirty_Mark_PvVoltage`, set while all four PV string voltages are < 14.1 V) | medium | Docs (re-audit 2026-07-14) |
| `0x08005d10` | Inverter_Power_Value_Scale | Scale power value for register format: in addition to the scaling, Inverter_PowerSetpoint_DeadbandClamp is applied to a min/max table, followed by fixed-point→float conversion (VectorSignedToFloat) and back — added 2026-07-14 | medium | Docs (re-audit 2026-07-14) |
| `0x08005d74` | Inverter_Set_Schedule_Reg | Write schedule registers 0x30-0x3E | high | Docs (re-audit 2026-07-14) |
| `0x08005dd4` | Inverter_RegDirty_Bitmap_SetClear | Name corrected (2026-07-15, formerly Timeslot_Bitmap_SetClear): generic bitfield setter/clearer (arbitrary register dirty bit); 8 callers — no schedule/time-of-day relation despite the former name "Timeslot" | medium | Docs (re-audit 2026-07-14) |
| `0x08005e2c` | Inverter_Set_PowerLimit_Reg_0x58 | Power limiting via register 0x58 | high | Docs (re-audit 2026-07-14) |
| `0x08005e68` | Inverter_Set_Flag_Reg_0x57 | Feature flag via register 0x57 | medium | Docs (re-audit 2026-07-14) |
| `0x080060d6` | Inverter_Write_Reg_0x04_U32Value | Write 32-bit value to register 0x04 (Register_PackDescriptor cat. 4 confirmed) | high | Docs (re-audit 2026-07-14) |
| `0x08006114` | Inverter_Write_Reg_0x03_U32Value | Write 32-bit value to register 0x03 (Register_PackDescriptor cat. 3 confirmed) | high | Docs (re-audit 2026-07-14) |
| `0x08006152` | Inverter_Clear_Reg_0x13 | Clear register 0x13 (Register_PackDescriptor cat. 0x13, writes 0, confirmed) | high | Docs (re-audit 2026-07-14) |
| `0x080061a0` | Inverter_Set_WorkMode_Reg_0x60 | Set work mode register 0x60 (Register_PackDescriptor cat. 0x60 confirmed) | high | Docs (re-audit 2026-07-14) |
| `0x080061d0` | Inverter_Set_BLEMode_Reg_0x08 | Set BLE mode register 0x08 (Register_PackDescriptor cat. 8 confirmed, caller BLE_Set_Mode_Persist) | high | Docs (re-audit 2026-07-14) |
| `0x080061fc` | Inverter_Power_Setpoint_Calc | Central power setpoint calculation (12 callers, 28 reference sites). Basis Battery_Charge_Power_Limiter, optional fp64 sum of 4 struct fields (offsets 0xC/0x12/0x18/0x1E), clamp via Power_Limit_Clamp/Inverter_Power_Setpoint_ScaleFactor_Calc, writes register cat. 1 | high | Docs (re-audit 2026-07-14) |
| `0x08006488` | Inverter_Set_Flag_Reg_0x06 | Set flag register 0x06 (Register_PackDescriptor cat. 6 confirmed; callers Idle_Watchdog_AutoReboot, Write_Handler, Register_SetFlag_0x06) | high | Docs (re-audit 2026-07-14) |
| `0x080064ba` | Inverter_Clear_Reg_0x12 | Clear register 0x12 (Register_PackDescriptor cat. 0x12, writes 0, confirmed) | high | Docs (re-audit 2026-07-14) |
| `0x080064dc` | Inverter_Set_Flag_Reg_0x05 | Set flag register 0x05 | Batch 20 | Ghidra (batch 20, re-audit 2026-07-14) |
| `0x08006506` | Inverter_Write_Reg_0x07_Value | Write value to register 0x07 | Batch 20 | Ghidra (batch 20, re-audit 2026-07-14) |
| `0x08006530` | Inverter_BatteryParams_Timeslot2_SetClear | Set/clear battery parameter time window 2 | Batch 20 | Ghidra (batch 20, re-audit 2026-07-14) |
| `0x08006540` | Grid_Export_Limit_Periodic_StateMachine | State machine for periodic grid feed-in limit: flag byte (offset+0x17 bit 7), 3 states (0/1/2), calls Inverter_Power_Setpoint_Calc + Grid_Export_Power_Limiter, EventLog_Record_SystemEvent(6,600), 1000-tick Config_Apply_SingleReg(7). Caller: MainLoop_Periodic_Tasks | high | Docs (re-audit 2026-07-14) |
| `0x0800c0b0` | Telemetry_History_Record_Push | Append telemetry history entry | Batch 20 | Ghidra (batch 20, re-audit 2026-07-14) |
| `0x08012f80` | Power_Limit_Clamp | Computes fp64 power limit from the BatteryParams struct (base SRAM 0x20014F82, offset 0 × offset+0x24, sign inverted). Callers: Battery_Charge_Power_Limiter, Grid_Export_Power_Limiter, Inverter_Power_Setpoint_Calc | high | Docs (re-audit 2026-07-14) |
| `0x08012ff4` | Inverter_Power_Setpoint_ScaleFactor_Calc | Compute scaling factor for power setpoint (fp64 multiplication from BatteryParams struct offset 0 and offset+0x28, 3 callers in Inverter_Power_Setpoint_Calc) | high | Docs (re-audit 2026-07-14) |
| `0x08013060` | BMS_CellVoltage_MinMax_PerPack | Compute min/max per schedule group | Batch 20 | Ghidra (batch 20, re-audit 2026-07-14) |
| `0x0801321c` | Meter_CT_GetActiveType | Determine active CT transformer type | Batch 20 | Ghidra (batch 20, re-audit 2026-07-14) |
| `0x08013254` | Meter_CT_GetStabilizeDelay | Determine stabilization delay for CT transformer | Batch 20 | Ghidra (batch 20, re-audit 2026-07-14) |
| `0x08013920` | WorkMode_Modbus_ResponseCache_Refresh | Refresh Modbus response cache for work mode (Modbus_Response_Builder, 4×4B cache). Sole caller: WorkMode_ChangeHandler (at the end of each branch) | high | Docs (re-audit 2026-07-14) |
| `0x08013cc8` | Inverter_PowerSetpoint_DeadbandClamp | Limit power setpoint with deadband (fp64 percentage from BatteryParams struct, sign/range logic). Callers: Remote_Power_Setpoint_Process, Inverter_Power_Value_Scale | high | Docs (re-audit 2026-07-14) |
| `0x0801d2a8` | Timeslot_ApplyConfigOnSync | Semaphore → bitmap apply → release | high | Docs (re-audit 2026-07-14) |
| `0x0801d310` | Register_ToggleBit15 | 16-bit register XOR 0x8000 | high | Docs (re-audit 2026-07-14) |
| `0x0801e290` | WorkMode_Flag_Reset_And_TriggerSetpointEval | Resets pending flag; if the broadcast flag is set OR the 300ms timer has expired, TimePlan_Evaluate_Setpoint() (setpoint recalculation) is triggered — corrected 2026-07-14: the old description "clear connection flag, trigger disconnect" was wrong, no connection/disconnect relation in the code | medium | Ghidra (re-audit 2026-07-14) |
| `0x0801e5dc` | Remote_Power_Command_Execute | Mode 0-3: zero/discharge/charge/SOC threshold | high | Docs (re-audit 2026-07-14) |
| `0x0801ec38` | Grid_Export_Power_Limiter | Limits grid feed-in power; 4 modes via DAT_0801eea4 (0/1/3, incl. special case CT type 5); sends the result via up to 4 calls of Inverter_RS485_Command_Send. Callers: MainLoop_Periodic_Tasks, Grid_Export_Limit_Periodic_StateMachine | high | Docs (re-audit 2026-07-14) |
| `0x08025d74` | Power_Delta_Detect | Detect power change (threshold-based) | high | Docs (name match, re-audit 2026-07-14) |
| `0x08026598` | Register_WriteValue | Central register write access (41 callers verified) | high | Docs (name match, re-audit 2026-07-14) |
| `0x080265f0` | Register_WriteCategory0xCE | Writes register category 0xCE (subindex param_4) via Register_PackDescriptor/Register_WriteValue; special case subindex 3 with incremental flag byte. Callers: CAN_Update_WriteData/WriteStatus/WriteEraseInfo | high | Docs (re-audit 2026-07-14) |
| `0x08028ebc` | Inverter_Default_Init_Once | One-time inverter initialization (flag-protected) | high | Docs (re-audit 2026-07-14) |
| `0x08029394` | Inverter_Apply_BatteryParams | Check battery cell voltages, set charge/discharge flags; additionally init gate (I2C_SendWorkModeFrame/I2C_SyncChangedRegisters + 15-tick timer), timeslot 2 bitmap set/clear depending on the cell voltage flag, and final call of Grid_Power_Dynamic_Adjust (150ms control cycle) — added 2026-07-14 | high | Docs (re-audit 2026-07-14) |
| `0x08029458` | Remote_Power_Setpoint_Process | Protocol cmd 0x05, register lookup, float scaling | high | Docs (re-audit 2026-07-14) |
| `0x080295a4` | Grid_Power_Dynamic_Adjust | 150ms timer, fp64 scaling, frequency thresholds ±15 | high | Docs (re-audit 2026-07-14) |
| `0x080297cc` | Runtime_Energy_Counter_Tick | ms-resolution energy measurement (999 wrap → kWh; the unit "kWh" cannot be verified in the code itself, but no counter-evidence found) | high | Docs (re-audit 2026-07-14) |
| `0x0802b9e8` | CT_StatusFlag_Test | Test CT struct bitmask flag (offset +8) | high | Docs (re-audit 2026-07-14) |
| `0x0802b9fa` | CT_GetResultValue | CT struct getter (offset +0xC) | high | Docs (re-audit 2026-07-14) |
| `0x0802ba00` | CT_SetResultValue | CT struct setter (offset +0xC) | high | Docs (re-audit 2026-07-14) |
| `0x0802baa4` | CT_GridPower_Controller | Main control loop: CT→WorkMode→Voltage→Power→Setpoint | high | Docs (re-audit 2026-07-14) |
| `0x0802bce8` | CT_PowerSetpoint_Compute | Proportional control, min/max clamp, dead band (11W/15W) | high | Docs (re-audit 2026-07-14) |
| `0x0802bec8` | WorkMode_Register_Write | Validate work mode (<11), write via Register_WriteValue | high | Docs (re-audit 2026-07-14) |
| `0x0802bf18` | PowerPercent_Register_Write | Validate power percent (<101), write via Register_WriteValue | high | Docs (re-audit 2026-07-14) |
| `0x0802c784` | WorkMode_State_Machine | Charge/discharge control: SOC thresholds 0x33=51%/0x32=50%, force charge (mode 5), 60s timer intervals, calls Inverter_Power_Setpoint_Calc 6× for different states. Sole caller: CT_GridPower_Controller | high | Docs (re-audit 2026-07-14) |
| `0x0802cf68` | Register_ClearOnWrite | Clear register to 0 on write access | medium | Docs (re-audit 2026-07-14) |
| `0x0802d484` | CT_SyncTransfer | Synchronous CT register access with 0x1000-iteration timeout | high | Docs (re-audit 2026-07-14) |
| `0x0802d53c` | Inverter_SendOnCommand | Inverter_Set_OnOff_Reg_0x01(1) 2× for reliability | high | Docs (re-audit 2026-07-14) |
| `0x0802d6d0` | Inverter_BeginShutdown | Activate shutdown state, call Inverter_StopOutput | high | Docs (re-audit 2026-07-14) |
| `0x0802d714` | Inverter_StopOutput | Power=0 + off — core shutdown primitive (4 callers) | high | Docs (re-audit 2026-07-14) |
| `0x0802d91c` | Register_Write_PackedAsciiValue_Group0xCC | Writes the RTC_GetDateTime() result + parameter bytes into register cat. 0xCC (subindex 0+1); only 1 byte is actually encoded as an ASCII digit (+'0'), the rest are raw bytes — the description "packed ASCII value" is only partially accurate, see re-audit report 2026-07-14. No callers in the current code path | medium | Docs (re-audit 2026-07-14) |
| `0x0802f0ec` | Telemetry_Store_RegCB | 8B for cmd 0xCB | medium | Docs (re-audit 2026-07-14) |
| `0x0802f104` | Telemetry_Store_Reg54 | 1B for cmd 0x54 | medium | Docs (re-audit 2026-07-14) |
| `0x0802f118` | Telemetry_Store_RegC1 | 1B for cmd 0xC1 | medium | Docs (re-audit 2026-07-14) |
| `0x0802f12c` | Telemetry_Store_RegCE_ByChannel | 4B in channel 0/2 for cmd 0xCE | medium | Docs (re-audit 2026-07-14) |
| `0x0802f15c` | Telemetry_Store_EnergyCounters | Cmd 0x40-0x43: 4 energy phases | high | Docs (re-audit 2026-07-14) |
| `0x0802f1c0` | Telemetry_Register_Dispatcher | Central dispatcher (11 callees), cmd 0x10-0xCE | high | Docs (re-audit 2026-07-14) |
| `0x0802f2b4` | BatteryParams_PowerFlowState_Get | Renamed 2026-07-15 (was Telemetry_Timestamp_Get, demonstrably wrong — no RTC/time access). Reads disable flag (BatteryParams struct base 0x20014F82, offset+0xF) and signed short (offset+2, scaled /10), returns 4-state code: 0=disabled, 1=~0 (<1), 2=positive (>=1), 3=negative (<0). Callers: MQTT_Telemetry_Struct_Builder, Cloud_Report_FillPowerFlow, BLE_RuntimeInfo_Builder — each embedded as a status byte in telemetry/report/BLE, consistent with charge/discharge/power-flow status. Physical unit of offset+2 not conclusively proven. See VNSD_Control_FW_Analysis_app_1492_0702_142136.md §13.46 | medium | Deep dive 2026-07-15 |
| `0x0802f840` | Energy_Stats_Accumulate_And_Save | Accumulate 4 sources, date change reset, flash persist every 600 ticks | high | Docs (re-audit 2026-07-14) |
| `0x0802fb9c` | WorkMode_ChangeHandler | Dispatch 6 modes (CT/CH395/remote/etc.), heartbeat reset on change | high | Docs (re-audit 2026-07-14) |
| `0x08032348` | Register_PackDescriptor | **36 callers** (corrected 2026-07-14, previously "35 callers!")! Packs category/subindex/field/flags into 32 bits | high | Ghidra (re-audit 2026-07-14) |
| `0x08034c24` | Battery_Config_Debug_Print | SOC cutoffs, max power, grid standard (debug print, 0 callers in the current code path — presumably a dead debug feature) | high | Docs (re-audit 2026-07-14) |
| `0x08035cac` | MPPT_Debug_Print | PV1-4 V/I/P, fault, temp, day/month/year capacity (debug print, 0 callers in the current code path) | high | Docs (re-audit 2026-07-14) |
| `0x08035ffc` | Inverter_Telemetry_Debug_Print | Prints 20 fields of the inverter telemetry block (base SRAM 0x20014E90/0x20036128, 48B) via printf: inv_state, grid_volt, grid_pf, off_grid_volt, bat_sample_power, chrg/dischrg energy, hard/soft/boot version, work_mode etc. No callers in the current code path (debug only) | high | Docs (re-audit 2026-07-14) |
| `0x08036698` | Inverter_PowerSetpoint_Apply_Wrapper | Thin wrapper → Inverter_Power_Setpoint_Apply. 0 callers/references found in the entire binary (neither call nor data pointer) — presumably dead code or an unresolved indirect call — added 2026-07-14 | high | Docs (re-audit 2026-07-14) |
| `0x0804d204` | PowerPercent_WriteCallback | Thin wrapper → PowerPercent_Register_Write(param_1). No callers in the current code path (presumably callback table) | high | Docs (re-audit 2026-07-14) |
| `0x0804d76c` | Inverter_SetWorkMode | Sets work mode (parameter <6): Inverter_RS485_Cmd_WorkMode(0) + Inverter_Set_WorkMode_Reg_0x60(param_1). No callers in the current code path (presumably callback/API) | high | Docs (re-audit 2026-07-14) |
| `0x08050e24` | Register_SetFlag_0x06 | Thin wrapper → Inverter_Set_Flag_Reg_0x06(1,param_1). No callers in the current code path | high | Docs (re-audit 2026-07-14) |
| `0x08050e32` | Register_SetValue_0x07 | Thin wrapper → Inverter_Write_Reg_0x07_Value(1,param_1). No callers in the current code path | high | Docs (re-audit 2026-07-14) |
| `0x080049ec` | Voltage_Stability_Check | Checks CT meter voltage for stability (delta<31) with timer debounce, controls the enable for CT_GridPower_Controller | medium | Re-audit 2026-07-14 |
| `0x08005500` | Standby_Wakeup_Debounce | Wake-up debounce from standby mode | medium | Docs |
| `0x080071c4` | Display_Cycle_DataSources | 4-state round robin over 4 fault/status values, reports non-zero values via EventLog_Record_DisplayError | medium | Re-audit 2026-07-14 |
| `0x0801d084` | Relay_StagedTimingControl | Relay/contactor GPIO bit 15, 3 timing stages | medium | Docs |
| `0x0802d018` | TimePlan_Evaluate_Setpoint | 10-slot schedule: weekday bitmask, time window, power or standby | high | Docs |

## Config / EEPROM (59)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x0800545c` | EEPROM_Config_Factory_Write | Factory reset routine: write EEPROM block 0x3532 (0x22B) with factory settings (0xE6 pattern), erase flash region (Flash_EraseAddressRange) and reprogram (Flash_Write_Protected); caller Factory_Reset | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08005e8a` | Config_Param51_Reset | Reset parameter 0x51 to 0 (Register_PackDescriptor+WriteValue); caller Write_Handler — write access confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08005eac` | Config_WorkMode_Set | Configure work mode via param system (Register_PackDescriptor(0x52)+WriteValue); caller Write_Handler (3x) — write access confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08005ece` | Config_PowerSetpoint_Write | Write power setpoint into param store (Register_PackDescriptor(0x56)+WriteValue); caller Write_Handler (2x) — write access confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08005f0a` | Config_Param53_Activate | Set parameter 0x53 to 1 (Register_PackDescriptor+WriteValue); caller Write_Handler — write access confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08005f2c` | Config_Param55_Set | Set parameter 0x55 (param_1, Register_PackDescriptor+WriteValue); caller Write_Handler(1) — write access confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08005f50` | Config_Counters_Reset | Reset counter parameter 0xC1 (2× register write access subindex 3+4, 4B=0 each); caller Write_Handler — write access confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08005f94` | Config_Feature_Enable | Feature activation via param 0x50 (0x55EE/0x55BB); caller Write_Handler — write access confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006074` | Config_Conditional_Write | Conditional param write via 0xC3 (only if flag DAT_080060a4 is set and param_2<9); 0 callers in the current code path — write access confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x080060ac` | Config_Param0A_Relay | Forward parameter 0x0A (Register_PackDescriptor(10)+WriteValue); sole caller Config_Set_DischargeCutoff_WithRelay — write access confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08006174` | Config_Mode_Apply | Set EPS enable/disable via register 2 (param1=index, param2=bool); caller context "BLE: Set eps_enable/disable" | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006890` | Config_Factory_Reset | Erase EEPROM addr 0, clear stats (if param_1≠0), before reboot; sole caller FactoryReset_NotifyAndReboot — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800691c` | EEPROM_Mutex_Wait | Wait function after I2C transfer: busy-wait loop (iteration count = DAT_08006940 × param_1) while the FreeRTOS scheduler is running, otherwise vTaskDelay(param_1); caller EEPROM_Read/EEPROM_Write | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006944` | Config_Get_Capacity_Factor | Battery capacity factor from config (byte 30-100%, fallback 100 if out of range), as a float scaled with DAT_0800697c; sole caller CT_PowerSetpoint_Compute — read access confirmed, no read/write error | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08006980` | Config_Get_WorkMode | Work mode = 'd'(100) - config_byte; read access confirmed, no read/write error | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08006990` | Config_Get_DeviceModelCode | Device model code: 0xAA magic byte → stored value, otherwise hardcoded codes depending on type byte (0x3F2/0x8AE/0x8AF); read access confirmed, no read/write error | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080069dc` | EEPROM_Read | Low-level EEPROM read function (I2C, mutex/queue-protected): param_1=address, param_2=target buffer, param_3=size; central primitive for all Config_Read_*/Get_* functions (13+ callers) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006b40` | Config_Read_String_0x388 | Read string from EEPROM offset 0x388, null termination enforced; read access confirmed, no read/write error | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006b62` | Config_Read_ProductionDate | Read production date from EEPROM 0x160 (8B DateTime), year correction (-2000), validation via DateTime_Validate_NoNull; read access confirmed, no read/write error. No callers found in the current code path | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08006ba8` | Config_Notify_Change | Persist time slot config (time slot, index<10, 10B each) or the complete 100B block (index≥10) to EEPROM 0x302 | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006be0` | Config_Save_RuntimeCounters | Persist 52B (0x34) runtime statistics to EEPROM 0x500; 3 callers (System_Reboot, Energy_Stats_Accumulate_And_Save, Stats_Clear_Counters) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006bf4` | EEPROM_Write | Low-level EEPROM write function (I2C, mutex/queue-protected): param_1=address, param_2=source buffer, param_3=size; central primitive for all Config_Write_*/Save_* functions (35+ callers) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006c9c` | Config_Write_U16_0x202 | Write a 16-bit value (max. 0x9C4/2500) to EEPROM 0x202 on change; caller Write_Handler, BLE_Recv_Cmd_Dispatcher (range check 300-2500) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006cd0` | Config_Write_Byte_0x36b | Write a 1-byte value to EEPROM 0x36b, only if ≤ limit from config struct+8; caller Cloud_HTTP_Response_Parser, BLE_Recv_Cmd_Dispatcher | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006d00` | Config_Write_CapacityFactor | Writes 1 byte (not U32), counterpart to `Config_Get_Capacity_Factor`. Renamed on 2026-07-14 from "Config_Read_U32" (the old name wrongly implied a read access/U32 width) | high | Re-Audit 2026-07-14 |
| `0x08006d2c` | Config_Write_U8 | Write a 1-byte value (0-4) to EEPROM 0x369, including an error flag reset on a previous error status | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006d68` | Config_Write_U16 | Write a 16-bit value (max. 2500/0x9c4) to EEPROM 0x204 on change | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006d9c` | Config_Write_DischargeCutoffSOC | Discharge cutoff SOC (30-88%), stored as a complement ('d' value or 0x0c if 0); caller MQTT_Config_Command_Handler, Config_Set_DischargeCutoff_WithRelay — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006dd0` | Config_Write_BoolFlag_0x375 | Bool flag at EEPROM 0x375 (feature unclear); sole caller BLE_Recv_Cmd_Dispatcher — write access confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08006df8` | Config_Save_UserDataBlock | Save 36B (0x24) user data block to EEPROM 0x4000; caller Config_SaveWithCRC, OTA_InitSlotConfig — confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08006e0c` | Config_Write_WorkingMode | Write work mode (0-7) to EEPROM 0x374, only on change; sole caller Config_Set_WorkMode_Validated — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006e3c` | Config_Write_String_0x388 | Write a string (max. 12 characters + terminator) to EEPROM 0x388; caller Cloud_HTTP_Response_Parser, BLE_Recv_Cmd_Dispatcher | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006e80` | Config_Write_String | Write a string (max. 12 characters) to EEPROM 0x376, only with length check <13 and on change (strcmp); caller Cloud_HTTP_Response_Parser, BLE_Recv_Cmd_Dispatcher | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006ebc` | Config_Write_ModbusAddr | Modbus slave address to EEPROM 0x901 + 0xAA validity marker to EEPROM 0x902 (only if not already set); sole caller Write_Handler — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006ef0` | Config_Write_Flag_0x367 | Write bool flag (0/1) to EEPROM 0x367; caller Cloud_HTTP_Response_Parser, BLE_Recv_Cmd_Dispatcher | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006f18` | Config_Write_PowerOffset | Signed 16-bit power offset/calibration (range-limited) to EEPROM 899/0x383; caller Cloud_Handle_SelfCtl_Power, BLE_Recv_Cmd_Dispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006f64` | Config_Write_ValidatedU16_0x36e | Sets validity marker 0xAA at EEPROM 0x36d and a 16-bit value at EEPROM 0x36e (two EEPROM_Write calls); caller Cloud_HTTP_Response_Parser, BLE_Recv_Cmd_Dispatcher | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006f98` | EEPROM_Save_RebootTimestamp | Name corrected (2026-07-15, formerly EEPROM_Clear_RebootState): actually writes the current RTC date/time (8B, via RTC_GetDateTime) to EEPROM 0x160 before system reboot — the local zero initialization is immediately overwritten by the RTC_GetDateTime call, nothing is cleared; sole caller System_Reboot | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08006fb4` | EEPROM_ReadWrite_Test | EEPROM R/W test: write/read 0xFF fill pattern + index sequence pattern, hex dump output; no callers in the current code path (debug function) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800b660` | EEPROM_Clear_Range | Erases Quectel SSL certificates/keys (3× Quectel_SSL_Certificate_Manage with 500ms delays: CA cert, user cert, user key) and one additional config byte; caller Factory_Reset, BLE_Recv_Cmd_Dispatcher. Name "EEPROM" slightly misleading (actually Quectel certificate store erasure, not an EEPROM region), but consistent in the factory reset context, no rename suggestion | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0801d324` | Config_Apply_WorkModeReg | Map mode index → reg 8/9 (via Config_Apply_SingleReg); caller Config_WorkMode_Apply_Wrapper, BLE_Recv_Cmd_Dispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08026338` | Config_SetUpdateFlag | Set update flag in RTC config structure; sole caller RTC_ConfigClockSource — cluster assignment to Config/EEPROM questionable (possibly rather hardware/RTC cluster) | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08027e44` | Config_PostWriteCommit | Reset flag + write parameter 0x54 to 0 (commit signal after a write sequence, same family as Config_Param51/53/55) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08028e30` | Config_Graduated_Sync_Retry | Escalating timer (5s→30s), config sync with retry via Config_Apply_SingleReg(10)/(4); sole caller Status_Bitfield_Update — confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x0802bef4` | Config_Apply_SingleReg | Write a single byte code value (command/status code, e.g. 2/4/7/8/9/0xB/0xC/0xA0) via a fixed predetermined register | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802bf44` | Config_Write_ParamPair | Concatenate two byte params into 16-bit, write 8B via Register_WriteValue; sole caller Inverter_PVString_ChannelValue_PeriodicCheck_Save — confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x0802bf70` | Config_Write_ResetFlag | Write constant 1 into register (config reset signal); sole caller Write_Handler — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802c034` | Config_Write_Category0xC4 | Get register handle via 0xC4 and write byte; sole caller System_ResetToDefaults — confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x0802cfc8` | Config_Set_WorkMode_Validated | Validate WorkMode (<8) → Config_Write_WorkingMode, otherwise printf debug output; sole caller Config_Set_WorkMode_Wrapper — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802d108` | Config_Set_DischargeCutoff_WithRelay | Write discharge cutoff SOC (Config_Write_DischargeCutoffSOC), trigger relay update on success (Config_Param0A_Relay); caller MQTT_JSON_RPC_Dispatcher, BLE_Handle_SetDischargeCutoff — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802fd64` | Config_SaveWithCRC | Calculate CRC16 (Modbus) over 0x22 bytes, then Config_Save_UserDataBlock; sole caller HTTP_Cloud_Reporting_Dispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08032ed0` | Config_Set_WorkMode_Wrapper | Thin wrapper → Config_Set_WorkMode_Validated; 0 callers in the current code path | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0803668c` | Config_WorkMode_Apply_Wrapper | Thin wrapper → Config_Apply_WorkModeReg; 0 callers in the current code path | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0804bc68` | EEPROM_I2C_BusRecovery | I2C bus recovery (clock toggling), "IIC Restore Slave By Clock" log message; caller EEPROM_I2C_WriteBytes, EEPROM_I2C_ReadBytes — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0804d5b0` | EEPROM_ClearSetting_0x900 | Sets byte to 0 and writes it to EEPROM 0x900. No callers found in the current code path | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0804d6ec` | Config_SetLocalApiPort | Write local API port (16-bit) to EEPROM 0x372, sets status byte to 5 unless already 3/4, debug printf. No callers found in the current code path | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08050e40` | Config_Apply_SingleReg_0x01 | Apply single register 0x01 from config (thin wrapper → Config_Apply_SingleReg(1)); 0 callers in the current code path | Batch 20 | Ghidra (Re-Audit 2026-07-14) |
| `0x08004b24` | Status_Bitfield_Update | Sets 3 status bits from error/state flags and triggers Config_Graduated_Sync_Retry | medium | Re-Audit 2026-07-14 |
| `0x080068ac` | Stats_Clear_Counters | Clear 0x34B statistics structure (3 modes) | high | Docs |

## OTA / Flash / SPI Flash / QSPI (63)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08000294` | Flash_ReadWords | Configure flash controller, read words from internal flash (return 0x5A5A5A5A sentinel on success); caller Flash_ReadWithECC (1x direct + callee) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08000358` | Flash_ReadWithECC | Flash read with ECC check (Flash_ReadWords/Mem_CopyWords_Aligned depending on ECC mode) and XOR validation of the ECC words; 0 callers in the current code path | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08004918` | OTA_Is_VNSD_Model | Checks via strstr whether the model string contains "VNSD_0"; sole caller BLE_Cmd_OTA_Validate — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08004da4` | OTA_CRC_Verify | **CRC-16** (Modbus) check of an FW block — **corrected 2026-07-14**: previously wrongly documented as CRC-32; calls QSPI_Flash_CalculateCRC → CRC16_Calculate (init 0xFFFF), comparison masked with & 0xffff | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08004e04` | OTA_Slot_Config_Validate | Validate 4 OTA slots of 251B (0xfb) each: model string (strstr "VNSD_0"), type 1-4, CRC/address mandatory fields; return 0=OK, otherwise error code; sole caller OTA_Update_Dispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08004efc` | OTA_FW_Verify_SetStatus | OTA firmware verification: CRC check (Modbus CRC via OTA_QSPI_ReadRegion_CalcCRC) + dev_mask validation + device type string comparison ("VNSD-0" via strstr); sets slot status 0x401=CRC error/0x402=dev_mask mismatch/0x403=success via OTA_Set_SlotStatus; caller OTA_Flash_Page_Writer | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0800577c` | OTA_Process_Pending_Updates | Process pending slots (status 0x02) (ProcessFirmwareUpdateCommand per slot) and then stop all timers (OTA_StopAllTimers); sole caller OTA_Update_Dispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080058cc` | Flash_Obfuscated_String_Decode | Decode obfuscated strings from internal flash (modular LUT via DAT_080058dc, Flash_ReadWords as sentinel source); caller sscanf_Format_Parser (12x as a getc callback), MQTT_Credential_Buffer_Decode | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080077f4` | Flash_ErasePage | Erase a flash page (PER+STRT bits, 2x Flash_WaitReady with 0xb0000 timeout); sole caller Flash_EraseRegion — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08007848` | Flash_GetStatusFromFlags | Decode FLASH_SR bits (offset+0xc) into status codes 1/3/4/5/6/7; caller Flash_WaitReady (2x) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08007898` | Flash_Lock | Lock flash controller (set LOCK bit 0x80); caller Flash_EraseRegion, Flash_WriteRegion (2x each) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080078ac` | Flash_ProgramWord | Program a 32-bit word into internal flash (4-byte alignment check, PG bit, 2x Flash_WaitReady); sole caller Flash_WriteRegion — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08007908` | Flash_Unlock | Unlock flash controller (2-key sequence in CR register); caller Flash_EraseRegion, Flash_WriteRegion — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08007920` | Flash_WaitReady | Poll flash status (Flash_GetStatusFromFlags) with a configurable timeout counter; caller Flash_ErasePage (2x), Flash_ProgramWord (2x) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08012d94` | QSPI_Flash_QuadRead_ByteLoop | Byte-wise read subroutine of QSPI_Flash_QuadRead: polls transfer ready and reads the data register byte by byte into the target buffer | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08012dbc` | QSPI_Flash_IsBusy | Check QSPI flash busy status (peripheral offset+0x28, bit 0); sole caller QSPI_WaitBusyDone — confirmed | Batch 20 | Ghidra (Re-Audit 2026-07-14) |
| `0x08012dd4` | QSPI_TransferReady_Poll | Query QSPI transfer readiness (peripheral offset+0x28, bit 3); 4 callers (QSPI_Flash_QuadRead_ByteLoop, QSPI_TransferWords, QSPI_SendAndReceive, Retry_Until_Success_Or_Limit) — confirmed | Batch 20 | Ghidra (Re-Audit 2026-07-14) |
| `0x08012dec` | QSPI_WriteComplete_Check | Check QSPI write operation for completion (peripheral offset+0x28, bit 1 cleared); sole caller SPI_Flash_QuadPageProgram — confirmed | Batch 20 | Ghidra (Re-Audit 2026-07-14) |
| `0x08012f3c` | OTA_QSPI_ReadRegion_CalcCRC | Read a QSPI region: calculates CRC16 (QSPI_Flash_CalculateCRC) AND a separate checksum (SPI_Flash_MutexTransaction) over the same region, both values in output parameters; sole caller OTA_FW_Verify_SetStatus — confirmed | Batch 20 | Ghidra (Re-Audit 2026-07-14) |
| `0x08013f42` | SPI_Flash_DataChecksum_Calc | Calculate a simple byte sum checksum (+1) over SPI flash data; sole caller SPI_Flash_MutexTransaction — confirmed | Batch 20 | Ghidra (Re-Audit 2026-07-14) |
| `0x080151c8` | OTA_Update_Dispatcher | OTA flow: process pending updates, validate slot, build summary, start firmware download, prepare shutdown, watchdog+retry handler; sole caller App_MainLoopDispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080189f8` | OTA_Slot_Config_Summary_Build | Copy 4 slots config (0xfb B each) into a compact summary buffer (0x14 B each), including target address lookup via a 4-entry table; sole caller OTA_Update_Dispatcher — confirmed | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08018ad4` | OTA_Firmware_Download_Init | OTA download: search for the next open/faulty slot, initialize runtime buffer, determine target flash address, Flash_EraseAddressRange over 0x7D000; caller OTA_Update_Dispatcher, OTA_Download_Retry_Handler (2x) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08019650` | Flash_EraseRegion | Erase flash pages (0x800B/page, sector-wise via Flash_ErasePage), no signature check, only address range>=0x8000000; sole caller ProcessFirmwareUpdateCommand — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080196f8` | Flash_WriteRegion | Write flash word by word via Flash_ProgramWord (byte→word assembly via CONCAT), only address range>=0x8000000; sole caller ProcessFirmwareUpdateCommand — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08026140` | OTA_ValidateUrlSlots | Checks 4 OTA URL slots (251B) for validity (mandatory fields offset 0/9/0xa/0xb/0xd/0x11/0x15); on invalid data, flash re-init (read→erase→write). No caller found in the binary — usage unclear | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x08026704` | OTA_WriteDataToRingBuffer | Write OTA data into a 5-slot ring buffer (0x800B each), splits large payloads across slot boundaries, manages write index/slot-full flags; caller HTTPS_POST_ReceiveResponseData (3x) — name confirmed, no read/write error | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080273b4` | SPI_Flash_PeripheralEnable | Set/clear SPI/QSPI peripheral enable bit (2 registers); caller QSPI_ConfigureMode (4x) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080273dc` | QSPI_Controller_Reset | QSPI controller reset via RCC_AHBPeriphResetCmd (assert+deassert); caller QSPI_ConfigureMode (4x) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080273f4` | SPI_Flash_ConfigGPIO | Configure GPIO pins for SPI/QSPI mode (4 modes 0-3: single/dual/quad variants), including RCC clock enable and pin mode (input/AF); caller QSPI_ConfigureMode (4x) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802760c` | SPI_Flash_ClockGateControl | Set/clear QSPI clock gate bit; caller QSPI_ConfigureMode | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027630` | SPI_Flash_WriteEnableCheck | Check/set write enable latch bit in the flash status register (cmd sequence via QSPI_TransferWords), polls until ready via QSPI_Flash_PollStatusReady; caller SPI_Flash_QuadPageProgram | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027678` | SPI_Flash_SectorErase | Erase flash sector: cmd 0x06 (write enable) + cmd 0x20 (sector erase, 24-bit address) via QSPI_TransferWords, busy wait + retry + status poll; sole caller Flash_EraseAddressRange (sector loop) — name confirmed, no read/write error | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080276c0` | SPI_Flash_QuadPageProgram | Quad page program (cmd 0x32, max. 256B/call): write enable check, send address+data byte by byte via QSPI_WriteComplete_Check polling; sole caller Flash_Write_Protected (9x, for page splitting) — name confirmed, no read/write error | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080277a4` | QSPI_Flash_QuadRead | Quad read (cmd 0x6B) from external flash: send address, 0x100 ticks delay, byte-wise read via QSPI_Flash_QuadRead_ByteLoop; sole caller Flash_Read_Protected — name confirmed, no read/write error | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080277e0` | QSPI_Flash_PollStatusReady | Poll the flash status register until ready (max. 200× with 1ms delay via QSPI_TransferWords); caller SPI_Flash_SectorErase, SPI_Flash_WriteEnableCheck, SPI_Flash_QuadPageProgram | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802781c` | QSPI_SendCommandIrqSafe | Sends a 1-byte command + data word via an IRQ-locked QSPI_SendAndReceive transaction, then polls busy status; caller SPI_Flash_SectorErase, SPI_Flash_QuadPageProgram | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027834` | QSPI_ConfigureMode | Configure QSPI mode (parameter set per mode 0-3+ for write/read/CRC/quad), builds a local register config structure and calls QSPI_Controller_Reset, SPI_Flash_ConfigGPIO, QSPI_ApplyRegisterConfig, SPI_Flash_PeripheralEnable etc.; 6 callers (SPI_Flash_SectorErase, QSPI_Flash_CalculateCRC, SPI_Flash_MutexTransaction, Flash_Read_Protected, QSPI_Flash_QuadRead, SPI_Flash_QuadPageProgram) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027a44` | QSPI_ApplyRegisterConfig | Apply QSPI register configuration: ORs struct fields (timing/mode/address size/data size) into the QSPI peripheral registers, special path for extended config at value 0x400000/0x800000; sole caller QSPI_ConfigureMode (4x) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027b94` | QSPI_ReadDataRegister | Reads the QSPI data register (peripheral offset +0x60), 1-line function; 4 callers (QSPI_TransferWords, QSPI_SendAndReceive, Retry_Until_Success_Or_Limit, QSPI_Flash_QuadRead_ByteLoop) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027ba0` | QSPI_TransferWords | Writes param_3 words into the QSPI data register, polls transfer ready (200 iter.) + word counter register, then reads the response words back into param_2; caller SPI_Flash_WriteEnableCheck, SPI_Flash_SectorErase, QSPI_Flash_PollStatusReady | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027bfc` | QSPI_WriteDataRegister | Writes the QSPI data register (peripheral offset +0x60), 1-line function; 6 callers (QSPI_Flash_CalculateCRC, QSPI_TransferWords, QSPI_SendAndReceive, Flash_Read_Protected, QSPI_Flash_QuadRead, SPI_Flash_MutexTransaction) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027c08` | QSPI_SendAndReceive | Sends 1 word via QSPI_WriteDataRegister, immediately reads the response; if param_3≠0 additionally polls for transfer ready (200 iter.) and a second read response; return 1=success/0=timeout; sole caller QSPI_SendCommandIrqSafe | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027c44` | QSPI_Flash_CalculateCRC | Mutex-protected: calculates CRC-16 (Modbus, via CRC16_Calculate) over a QSPI flash region — **corrected 2026-07-14**: algorithm is CRC-16, not generically "CRC" | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027cf0` | SPI_Flash_MutexTransaction | Mutex-protected checksum transaction over a QSPI flash region: queue lock (1000 ticks timeout), QSPI_ConfigureMode(3), 3ms delay, SPI_Flash_DataChecksum_Calc, switch back to QSPI mode 2 + dummy write 0xFF, unlock; sole caller OTA_QSPI_ReadRegion_CalcCRC | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08027d9c` | Flash_SelfTest | Flash self-test at fixed address 0x3F0000: erase→pattern write (0-0x1D)→read→byte-wise comparison→final erase; return 0=OK/1=error. No callers found in the current code path | high | Ghidra (Re-Audit 2026-07-14) |
| `0x080292b8` | OTA_Set_SlotStatus | Set status per slot (max 4), codes 0x400-0x404, optional additional detail word; 4 callers (Comm_Watchdog_CheckTimeout, OTA_FW_Verify_SetStatus 3x, CAN_UpdateResultHandler 2x, OTA_Firmware_Download_Init) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08029594` | OTA_FlashPageWrite_Counter_Increment | Increment counter at offset +8 — **corrected 2026-07-09**, see batch 18; sole caller OTA_Flash_Page_Writer — re-audit confirms unchanged | medium | Ghidra (Re-Audit 2026-07-14) |
| `0x0802b6ec` | Flash_Read_Protected | Mutex-protected read from external QSPI flash (via QSPI_Flash_QuadRead, cmd 0x6B), 1000 ticks timeout on mutex acquisition | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802b774` | Flash_Write_Protected | Mutex-protected write into external QSPI flash; automatically splits data at 256B page boundaries (cmd 0x32 via SPI_Flash_QuadPageProgram) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802b8cc` | Flash_EraseAddressRange | Erases flash address range [param_1, param_1+param_2) sector by sector (via SPI_Flash_SectorErase, 0x1000 boundaries from Flash_AddressToPage), mutex-protected (queue, 1000 ticks timeout); only valid for addresses <0x400000; 9 callers including EEPROM_Config_Factory_Write, OTA_Firmware_Download_Init, Flash_SelfTest | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802b984` | Flash_AddressToPage | Flash address → page number: (addr & 0x0FFFFFFF) >> 12 (4KB pages); caller Flash_EraseAddressRange (2x) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802d550` | OTA_StopAllTimers | Stop 10 software timers (xTimerStop_Internal), then set shutdown flag (Shutdown_SetPendingFlag); sole caller OTA_Process_Pending_Updates — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802d5ec` | OTA_PrepareShutdown | Initiate inverter shutdown (Inverter_BeginShutdown), configure OTA state byte, depending on param_1 either queue sync (xQueueReceive/Send) or direct timer stop, stop 3 further timers; sole caller OTA_Update_Dispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802d6a8` | OTA_InitSlotConfig | One-time OTA config initialization (0x24B memset + Config_Save_UserDataBlock, flag-protected); sole caller OTA_Firmware_Download_Init — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802d86c` | OTA_Download_Retry_Handler | 4 slots, max 2 retries, calls OTA_Firmware_Download_Init when needed (2x in the code); sole caller OTA_Update_Dispatcher — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802f730` | OTA_Flash_Prepare_ByTarget | 4 targets: EMS(0x80000, byte value 0), MPPT(0x100000, byte value 2), BMS(0x180000, byte value 3), VNS(0x200000, byte value 4), 512KB each (0x7D000 erase length); sole caller BLE_Cmd_OTA_WriteSetup — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802fc78` | QSPI_WaitBusyDone | 20× poll (initially 5ms, then 50ms each) over QSPI_Flash_IsBusy, 4 callers (SPI_Flash_WriteEnableCheck 2x, QSPI_SendCommandIrqSafe, SPI_Flash_SectorErase, QSPI_Flash_QuadRead) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x0802fdec` | OTA_Flash_Page_Writer | 2KB pages from RAM ring buffer→external flash (Flash_Write_Protected), 5-slot circular, calls **OTA_FW_Verify_SetStatus** on a complete transfer (description corrected 2026-07-14: the old function name "OTA_FW_Verify_And_Apply" no longer exists/was stale); 0 callers in the current code path (presumably a timer callback) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08034cfc` | Flash_Address_To_PageIndex | (addr & 0x7FFFFFF) >> 11 (2KB pages); caller Flash_EraseRegion (2x) — confirmed | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08034d08` | Flash_ReadWriteErase_SelfTest | Flash write→read→verify→erase self-test (0x1e bytes pattern, hex dump per step); 0 callers in the current code path (debug function) | high | Ghidra (Re-Audit 2026-07-14) |
| `0x08001814` | ProcessFirmwareUpdateCommand | FW update commands (type 1-4): flash write, metadata, reboot | medium | Docs |
| `0x080054e0` | Retry_Until_Success_Or_Limit | Retry loop until success or limit reached | medium | Docs |

## Hardware / HAL (GPIO,ADC,SPI,I2C,USART,RCC,RTC,...) (84)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080001f8` | FPU_EnableCoprocessorAccess | Set SCB->AIRCR priority grouping field  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x08001fb6` | RCC_BackupDomainReset_Pulse | Pin set+clear (pulse) via wrapper | low | Docs |
| `0x08002014` | I2C_BitBang_Delay | Busy-wait 150 iterations (timing for bit-bang) | high | Docs |
| `0x08002024` | I2C_BitBang_WriteByte | Send MSB-first byte via GPIO (SDA=pin 2, SCL=pin 1) | high | Docs |
| `0x0800207c` | I2C_BitBang_Start | Generate I2C START condition | high | Docs |
| `0x080020b0` | I2C_BitBang_Stop | Generate I2C STOP condition | high | Docs |
| `0x080020d8` | I2C_BitBang_ReadBit | SDA high, toggle SCL, read SDA bit | high | Docs |
| `0x080028ec` | I2C_Init_Configure | Hardware I2C init (CR, CCR registers), 0 callers | high | Docs |
| `0x0800468c` | CH395_Packet_Receive_Parse | 2KB ring buffer, 6 packet types, XOR checksum, no auth | high | Docs |
| `0x080077e4` | Flash_SR_ClearFlags | Set RCC register bits for peripheral clocks | high | Docs |
| `0x08012a04` | AFIO_PinRemapConfig | *(no description — see notes above)* | - | — |
| `0x08012c54` | GPIO_ConfigPin_ModeCNF | Configure GPIO pin mode/CNF bits | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08012ed0` | RTC_GetDateTime | Read date/time from RTC | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08013e94` | RTC_TimeOfDay_To5MinSlotIndex | Convert time of day to 5-minute time slot index | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080172a0` | EEPROM_I2C_CheckStatusFlags | Check I2C status flags for EEPROM access | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080172ca` | EEPROM_I2C_SetAckEnable | Enable I2C ACK bit | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080172e4` | EEPROM_I2C_PeripheralReset | Reset I2C peripheral | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0801731c` | EEPROM_I2C_SetPeripheralEnable | Enable I2C peripheral | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08017334` | EEPROM_I2C_SetStartCondition | Set I2C start condition | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0801734c` | EEPROM_I2C_SetStopCondition | Set I2C stop condition | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08017364` | EEPROM_I2C_ConfigureClockTiming | Configure I2C clock timing | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08017450` | EEPROM_I2C_ReadDataRegister | Read I2C data register | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08017458` | EEPROM_I2C_SendSlaveAddress | Send I2C slave address | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0801746a` | EEPROM_I2C_WriteDataRegister | Write I2C data register | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08017470` | EEPROM_I2C_ReadBytes | Read multiple bytes from EEPROM via I2C | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080175f0` | EEPROM_I2C_WriteBytes | Write multiple bytes to EEPROM via I2C | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0801771c` | EEPROM_I2C_ClockDisable_Reset | Disable and reset I2C clock | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0801773c` | EEPROM_I2C_GPIO_ClockPulseRecovery | I2C bus recovery via GPIO clock pulses | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08017888` | SPI_Timer_GPIO_ChipSelect_Init | Initialize SPI chip-select GPIO/timer | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080178bc` | SPI2_Peripheral_Init | Initialize ADC peripheral | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08017ed4` | EEPROM_I2C_Peripheral_Init | Configure SPI GPIO + timer prescaler/period | medium | Docs |
| `0x080275ec` | GPIO_ConfigPinAsInput | Wrapper around GPIO_ConfigPin_ModeCNF; configures a pin fixed as input (CNF=3, MODE=0) — **Re-audit 2026-07-14** | high | Ghidra (Re-audit 2026-07-14) |
| `0x08027edc` | RCC_HSEConfig | Configure high-speed external oscillator | high | Docs (name match) |
| `0x08027f28` | RCC_LSEConfig | Configure low-speed external oscillator | high | Docs (name match) |
| `0x08027f5c` | RCC_RTCClkSourceConfig | Select RTC clock source | high | Docs (name match) |
| `0x08027f78` | RCC_AHBPeriphClockCmd | AHB peripheral clock enable/disable | high | Docs (name match) |
| `0x08027f98` | RCC_AHBPeriphResetCmd | AHB peripheral reset (RCC_AHBRSTR, bit set/clear depending on param_2) — **Re-audit 2026-07-14** | high | Ghidra (Re-audit 2026-07-14) |
| `0x08027fb8` | RCC_APB1PeriphClockCmd | APB1 peripheral clock enable/disable | high | Docs (name match) |
| `0x08027fd8` | RCC_APB1PeriphResetCmd | APB1 peripheral reset (RCC_APB1RSTR, bit set/clear depending on param_2) — **Re-audit 2026-07-14** | high | Ghidra (Re-audit 2026-07-14) |
| `0x08027ff8` | RCC_APB2PeriphClockCmd | APB2 peripheral clock enable/disable | high | Docs (name match) |
| `0x08028018` | RCC_PeriphBitControl | APB2 peripheral reset (RCC_APB2RSTR, offset +0xC, bit set/clear) — functionally analogous to RCC_APB1PeriphResetCmd, used for SPI1RST among others — **Re-audit 2026-07-14** | high | Ghidra (Re-audit 2026-07-14) |
| `0x08028038` | RCC_BDCR_BDRST_Write | *(no description — see notes above)* | - | — |
| `0x08028044` | RCC_LSICmd | *(no description — see notes above)* | - | — |
| `0x08028050` | RCC_RTCCLKCmd | *(no description — see notes above)* | - | — |
| `0x0802805c` | RCC_GetClocksFreq | Read out system clock frequencies | high | Docs (name match) |
| `0x08028168` | RCC_GetFlagStatus | Reads RCC status flag (CR/BDCR/CSR depending on flag encoding), verified against standard constants HSERDY(0x31)/LSERDY(0x41)/LSIRDY(0x61) — **Re-audit 2026-07-14** | high | Ghidra (Re-audit 2026-07-14) |
| `0x080281a4` | RCC_WaitForHSEStartUp | Wait for HSE startup with timeout | high | Docs (name match) |
| `0x080281dc` | RTC_BcdToByte | BCD → binary conversion | high | Docs (name match) |
| `0x080281f2` | RTC_ByteToBcd | Binary → BCD conversion | high | Docs (name match) |
| `0x08028210` | RTC_ConfigClockSource | Configure RTC clock source | high | Docs (name match) |
| `0x08028404` | RTC_SetTime | Set RTC date (WPR unlock 0xCA/0x53) | high | Docs (name match) |
| `0x080285c4` | RTC_EnterInitMode | Enter RTC init mode | high | Docs (name match) |
| `0x08028614` | RTC_ExitInitMode | Exit RTC init mode | high | Docs (name match) |
| `0x08028628` | RTC_GetDate | Read out RTC time | high | Docs (name match) |
| `0x08028674` | RTC_GetTime | Read out RTC date | high | Docs (name match) |
| `0x080286c4` | RTC_WriteAlarmRegisters | Write RTC alarm registers | high | Docs (name match) |
| `0x08028728` | RTC_InitAlarm | Initialize RTC alarm | high | Docs (name match) |
| `0x08028750` | RTC_SetDate | Set RTC time | high | Docs (name match) |
| `0x080288f0` | RTC_WaitForSynchro | Wait for RTC synchronization | high | Docs (name match) |
| `0x0802b6d4` | SPI_Cmd | Set/clear const flag (bit 0x40)  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x0802b990` | SPI_DeInit | RCC reset for 3 peripheral base addresses  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x0802ba04` | SPI_Init | OR 8 channels, preserve bits 0x3040, clear 0x0800 | medium | Docs |
| `0x0802ba40` | SPI_SSOutputCmd | Set/clear bit 2 (single/scan mode) | medium | Docs |
| `0x0802ced8` | RTC_SetDateTime | Validate date/time, HAL_RTC_Set*, mutex-protected | high | Docs |
| `0x0802d9b4` | SysTick_Enable_ClockSource | Set NVIC SysTick bit 2 (CLKSOURCE) | high | Docs |
| `0x0802d9e0` | GPIO_TogglePin_Periodic | Toggle GPIO pin periodically | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0802de9e` | UART_CheckStatusFlag | Status register bitmask test (4 callers, mask 0x80=TXE) | high | Docs |
| `0x0802deb8` | NVIC_GetIRQEnableStatus | Read IRQ enable bit from ISER register | high | Docs |
| `0x0802df0c` | USART_Init | CR1/CR2/CR3 + BRR calculation, standard STM32 | high | Docs |
| `0x0802dfca` | USART_SendData | Write 9-bit value into USART_DR (4 callers) | high | Docs |
| `0x0802fa84` | UART_TransmitBytes | Byte-wise UART send with 5000-tick TX poll | high | Docs |
| `0x0802facc` | SysTick_ComputeTickPeriod | Compute reload value from clock divider | high | Docs |
| `0x0802fb04` | SysTick_WaitTicks | Poll SysTick register until COUNTFLAG | high | Docs |
| `0x0802fb50` | SysTick_DelayMs | Chunked ms delay via SysTick_WaitTicks | high | Docs |
| `0x0802fdb8` | I2C_BitBang_WriteBytes | Complete I2C write transaction (Start/WriteByte×N/ReadBit×N/Stop) via GPIOC bit-bang; callers are `I2C_SendWorkModeFrame` and `I2C_SyncChangedRegisters` (0x0802c5b0). **Description corrected 2026-07-14** (old caller entry "CAN_SendWorkModeFrame" was outdated/wrong — the function is already called I2C_SendWorkModeFrame in the current Ghidra status, no CAN relation) | high | Ghidra (Re-audit 2026-07-14) |
| `0x08032324` | Peripheral_SetBit15 | Set/clear bit 0x8000 on peripheral register | medium | Docs |
| `0x080364a4` | GPIOD_Pin9_Write | Sets/clears GPIOD pin 9 (0x40011400, bit 9) via BSRR/BRR — pure GPIO pin write, name correct. **Description corrected 2026-07-14** (old description "MPU/Flash region protection" was wrong, no relation to MPU/flash registers) | high | Ghidra (Re-audit 2026-07-14) |
| `0x0804bc00` | RTC_CalcDayOfWeek | Arithmetic weekday calculation (1-7, 7=Sun) | high | Docs |
| `0x0805452c` | SPI_ReadByte | Read single byte via SPI | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08055120` | SPI_BeginCommand | Start SPI command transfer | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805514c` | SPI_WriteByte | Write single byte via SPI | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08000268` | Get_ActiveIRQn | Reads active exception/IRQ number from IPSR, only in privileged mode (otherwise 0), masked to 5 bits — only caller is the FreeRTOS port layer (vPortValidateInterruptPriority) | high | Re-audit 2026-07-14 |
| `0x08004c7c` | DateTime_Validate | Check date/time fields for validity | high | Docs |
| `0x08005230` | DateTime_Validate_NoNull | DateTime validation WITHOUT null pointer check (counterpart to DateTime_Validate) — callers guarantee a valid pointer; the old description wrongly claimed the opposite | high | Re-audit 2026-07-14 |

## FreeRTOS Kernel (Task/Queue/Timer/Heap) (83)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x0801947c` | Task_Init_CreateAll | Creates 14 tasks from table DAT_08019518 via FreeRTOS_xTaskCreate (with log on failure), then starts the scheduler | high | Re-audit 2026-07-14 |
| `0x08029d84` | RTOS_ResumeSuspendedTasks | Check 14 tasks, suspended→resume with logging | high | Docs |
| `0x0803313c` | FreeRTOS_eTaskGetState | Task state: running/ready/blocked/suspended/deleted | high | Docs |
| `0x0804a46c` | prvAddCurrentTaskToDelayedList | Task → delayed list with tick overflow handling | high | Docs |
| `0x0804a538` | prvAddNewTaskToReadyList | Task count++, update TCB, trigger PendSV | high | Docs |
| `0x0804a624` | prvCheckForValidListAndQueue | Timer lists + service task lazy init (prio 5) | high | Docs |
| `0x0804a6d0` | prvCopyDataFromQueue | Circular buffer read with pointer wrap | high | Docs |
| `0x0804a6fa` | prvCopyDataToQueue | Queue back/front copy, semaphore special case | high | Docs |
| `0x0804a778` | prvDeleteTCB | Free stack storage (+0x30) + TCB | high | Docs |
| `0x0804a78a` | prvGetDisinheritPriorityAfterTimeout | Mutex priority disinheritance | high | Docs |
| `0x0804a7c8` | Heap_Init | heap_4 pool: **71,680 bytes** (0x11800), 8B-aligned | high | Docs |
| `0x0804a860` | Event_GroupInit | Zero event group fields, xQueueSend signal | high | Docs |
| `0x0804a87e` | Task_InitTcb | TCB init: stack ptr, fn (+0x3C), priority (+0x40) | high | Docs |
| `0x0804a8ac` | Task_InitNewTask | Stack 0xA5 fill, copy name (max 16), prio clamp 0xF | high | Docs |
| `0x0804a990` | Timer_InitStruct | Timer CB: period, callback, auto-reload (bit 0x04) | high | Docs |
| `0x0804aa10` | Scheduler_InitReadyLists | 16 priority ready lists + 5 system lists | high | Docs |
| `0x0804aa78` | Heap_InsertFreeBlock | Free list insert with coalescing, **no heap canaries** | high | Docs |
| `0x0804aae0` | FreeRTOS_Timer_InsertIntoActiveList | Timer expiry with wrap-around, 0 callers  — **corrected 2026-07-09**, see Batch 18 | medium | Docs |
| `0x0804ab38` | Queue_IsEmpty | Critical section check (offset 0x38) | high | Docs |
| `0x0804ab52` | Queue_IsFull | Count vs max length (0x38 vs 0x3C) | high | Docs |
| `0x0804ab70` | Scheduler_CopyReadyTasksToArray | 5 categories × list → flat array | high | Docs |
| `0x0804abdc` | prvQueueSend_CopyAndNotify | FreeRTOS-internal (queue.c, name confirmed via error string): copies data into queue and notifies waiting task or semaphore special case | high | Re-audit 2026-07-14 |
| `0x0804aee4` | prvResetNextTaskUnblockTime | Delayed list head → global timer | high | Docs |
| `0x0804af14` | prvSampleTimeNow | Tick count, overflow detect → list switch | high | Docs |
| `0x0804af40` | prvSwitchTimerLists | Expired timer callbacks + list swap | high | Docs |
| `0x0804b008` | prvTaskCheckFreeStackSpace | Count 0xA5 bytes → free words (uint16) | high | Docs |
| `0x0804b094` | prvTaskIsTaskSuspended | TCB check: suspended list + event pending | high | Docs |
| `0x0804b12a` | prvUnlockQueue | Drain send/receive notifications, lock reset 0xFF | high | Docs |
| `0x0804b1bc` | prvWriteNameToBuffer | Task name → 15 chars padded | high | Docs |
| `0x0804b1fc` | pvPortMalloc | heap_4 allocator, first-fit, block splitting, **6 callers** | high | Docs |
| `0x0804b350` | pvTaskIncrementMutexHeldCount | Mutex counter TCB+0x50 | high | Docs |
| `0x0804b370` | xTimerGetTimerID | Timer ID (+0x1c) with critical section | high | Docs |
| `0x0804b3c4` | pxPortInitialiseStack | ARM Cortex-M stack frame (xPSR Thumb, EXC_RETURN) | high | Docs |
| `0x0804fc94` | ulTaskGenericNotifyTake | Generic notify take for task | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080500e8` | uxListRemove | Remove element from linked list | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050110` | uxTaskGetSystemState | Query system state of all tasks | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080501c0` | vListInitialise | Initialize list | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080501da` | vListInitialiseItem | Initialize list item | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080501e0` | vListInsert | Insert element into list in sorted order | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050214` | vListInsertEnd | Insert element at end of list | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805022c` | vTaskEnterCritical | Enter FreeRTOS critical section, set base priority, counter++ | high | Docs |
| `0x080502b8` | vTaskExitCritical | Exit FreeRTOS critical section, counter--, restore base priority | high | Docs |
| `0x0805032c` | vPortFree | Free memory via port layer | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080503e0` | vPortSetupTimerInterrupt | Set up timer interrupt for scheduler | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050408` | vPortValidateInterruptPriority | Validate interrupt priority against kernel requirements | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080504ac` | FreeRTOS_vQueueAddToRegistry | Registers queue/mutex with name in 15-slot queue registry (debug/trace) | high | Re-audit 2026-07-14 |
| `0x080504d8` | prvQueueReceive_LockAndBlockIfEmpty | Queue receive: lock and block if queue is empty | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050524` | vTaskDelay | FreeRTOS vTaskDelay: block task for n ticks via prvAddCurrentTaskToDelayedList, 90 callers across the FW | high | Re-audit 2026-07-14 |
| `0x080505a0` | vTaskGetInfo | Retrieve task information | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050624` | vTaskInternalSetTimeOutState | Set internal timeout state | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050718` | vTaskMissedYield | Mark missed yield | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050724` | vTaskPlaceOnEventList | Place task on event list | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050780` | vTaskPlaceOnEventListRestricted | Place task on event list with restrictions | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050804` | vTaskPriorityDisinheritAfterTimeout | Cancel priority inheritance after timeout | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050934` | vTaskResume | Resume task | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050a1c` | vTaskSetTimeOutState | Set timeout state for task | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050a78` | vTaskStartScheduler | Start FreeRTOS scheduler | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050b34` | xTimerStop_Internal | Remove timer from active timer list, scheduler yield if needed | high | Docs |
| `0x08050c58` | vTaskSuspendAll | Suspend scheduler (all tasks) | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050c68` | vTaskSwitchContext | Context switch to the next task | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053ba4` | FreeRTOS_StartScheduler | Start FreeRTOS scheduler: HW checks, SysTick init, SoftReset | high | Docs |
| `0x08053d24` | FreeRTOS_SysTick_TaskUnblock | From ISR: tick-based waking of tasks via PendSV | high | Docs (name match) |
| `0x08053d58` | FreeRTOS_xQueueCreateMutex | Create mutex queue | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053d7c` | FreeRTOS_xQueueGenericCreate | Generic queue creation | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053df8` | FreeRTOS_xQueueGenericReset | Reset queue to initial state | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08053eb0` | xQueueSend | FreeRTOS xQueueSend (base function for send variants): copy data into queue, timeout handling, task notify at receiver, 77 callers | high | Re-audit 2026-07-14 |
| `0x0805408c` | FreeRTOS_xQueueGenericSend | Generic send to queue | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080541a4` | FreeRTOS_xQueueIsQueueFullFromISR | Queue-full check from ISR context | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054370` | xQueueReceive | FreeRTOS xQueueReceive: copy data from queue, mutex priority inheritance handling, timeout, 61 callers | high | Re-audit 2026-07-14 |
| `0x08054538` | FreeRTOS_xTaskCheckForTimeOut | Check task timeout | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080545fc` | FreeRTOS_xTaskCreate | Create new task | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054664` | FreeRTOS_xTaskGenericNotify | Send generic task notification | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x0805488c` | FreeRTOS_xTaskGetSchedulerState | Query scheduler state | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080548ac` | FreeRTOS_xTaskGetTickCount | Read current tick counter | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080548b8` | FreeRTOS_xTaskIncrementTick | Increment SysTick, check delayed list | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054a94` | FreeRTOS_vTaskPriorityDisinherit | Undo mutex priority inheritance | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054b9c` | FreeRTOS_vTaskPriorityDisinheritAfterTimeout | Reset priority inheritance after timeout | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054c88` | FreeRTOS_xTaskRemoveFromEventList | Remove task from event list | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054dcc` | FreeRTOS_xTaskResumeAll | Resume scheduler/all tasks | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054f64` | FreeRTOS_xTimerCreate | Create software timer | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08054f9c` | FreeRTOS_xTimerCreateTimerTask | Create timer service task | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08055018` | FreeRTOS_xTimerGenericCommand | Send generic timer command to timer queue | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080550c0` | FreeRTOS_xTimerIsTimerActive | Check whether timer is active | Batch 20 | Ghidra (Batch 20, 2026-07-09) |

## CLI / AT-Command-Engine (49)

Standalone command-line interpreter (token parser, command dispatch, history, output formatting), reachable via the CH395/Modbus TCP receive path (`Network_ReceiveAndDispatchData` → `CLI_DispatchInputByte`). Originally misnamed as `BLE_GATT_*` (16 functions) and `ATCmd_*` (5 functions) respectively — in both cases the names suggested a relation to BLE GATT or to the Quectel AT modem interpreter, which the code does not have. Corrected and split out as its own cluster on 2026-07-14 (re-audit session), at the user's decision (deliberately against "leave in the BLE section" or "file into the CLI/debug cluster").

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x0804bd58` | CLI_Session_Register | Service in 5-slot array, references "VNSD_0_v1492" | medium | Docs |
| `0x0804bd90` | CLI_Entry_MatchAndInit | strcmp match → set flag | medium | Docs |
| `0x0804bdc8` | CLI_Entry_VisibilityFilter | 4 callers, attribute visibility filter | medium | Docs |
| `0x0804c24c` | CLI_ResolveVariableRef | *(no description — see notes above)* | - | — |
| `0x0804c270` | CLI_InvokeCommandHandler | *(no description — see notes above)* | - | — |
| `0x0804c3a6` | CLI_GetEntryDisplayName | *(no description — see notes above)* | - | — |
| `0x0804c3d8` | CLI_GetEntryValueString | *(no description — see notes above)* | - | — |
| `0x0804c428` | CLI_FindActiveSession | *(no description — see notes above)* | - | — |
| `0x0804c45c` | CLI_ReadEntryValue | *(no description — see notes above)* | - | — |
| `0x0804c4b4` | CLI_DispatchInputByte | *(no description — see notes above)* | - | — |
| `0x0804c5a0` | CLI_HelpOrInfoDispatch | *(no description — see notes above)* | - | — |
| `0x0804c5cc` | CLI_HistoryNavigate | *(no description — see notes above)* | - | — |
| `0x0804c68e` | CLI_HistoryPush | *(no description — see notes above)* | - | — |
| `0x0804cb7e` | CLI_FindEntryByName | *(no description — see notes above)* | - | — |
| `0x0804cbf8` | CLI_SelectEntry | *(no description — see notes above)* | - | — |
| `0x0804cc40` | CLI_FormatEntryValue | *(no description — see notes above)* | - | — |
| `0x0804d018` | CLI_OutputLineTruncated | *(no description — see notes above)* | - | — |
| `0x0804d074` | CLI_PrintCommandInfo | *(no description — see notes above)* | - | — |
| `0x0804d0e0` | CLI_PrintPrompt | *(no description — see notes above)* | - | — |
| `0x0804d148` | CLI_PrintReturnValue | *(no description — see notes above)* | - | — |
| `0x0804d1d4` | CLI_WriteString | *(no description — see notes above)* | - | — |
| `0x0804bd80` | CLI_DeleteCharForward | Wrapper around CLI_DeleteChar(param,1) — forward delete/Del key in the line editor. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804be20` | CLI_ClearToEndOfLine | Clears the rest of the edit line by overwriting with spaces + cursor back. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804be48` | CLI_Backspace | Wrapper around CLI_DeleteChar(param,-1) — backspace in the line editor. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804be58` | CLI_DeleteChar | Core logic for deleting a character (forward/backward) including buffer shift + terminal echo. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804bf34` | CLI_CursorLeftRepeat | Cursor N times to the left (send backspace escape sequence). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804bf64` | CLI_ConfirmAndExecute | Enter key: processes input line + prints new prompt. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804bf78` | CLI_ProcessInputLine | Line termination: entry match, history push, tokenize, command lookup + execute. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804bfec` | CLI_DetectNumberFormat | Detects number format from prefix (0x/0b/0-octal/float). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c044` | CLI_ParseEscapeChar | Converts escape sequence (\n,\t,\r,\b,\0) to character. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c098` | CLI_ParseNumber | Parses number (bin/oct/hex/float) from string to float. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c184` | CLI_ParseArgValue | Dispatches argument parsing depending on prefix ('=escape, digit/-=number, $=variable, otherwise string). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c202` | CLI_ParseString | Strips quotation marks + resolves escapes in string (in-place). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c708` | CLI_InitSession | Initializes CLI session structure (buffers, history slots) + registers session + selects root entry "VNSD_0_v1492". (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c7a8` | CLI_InsertChar | Inserts character at cursor position (insert/overwrite depending on mode flag) + terminal echo. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c8a0` | CLI_Backspace_EchoOnly | Only cursor position--/terminal echo (0x08), no buffer manipulation — different CLI context (echo without edit buffer). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c8ba` | CLI_ShowHelp | Wrapper around CLI_ListCommands. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c8c8` | CLI_ListCommands | Iterates all visible command entries + prints them. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804c914` | CLI_PrintCommandEntry | Prints a command entry line (value, type flags, name). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804ca10` | CLI_InsertCharPlain | Wrapper around CLI_InsertChar with plaintext echo flag cleared. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804ca2c` | CLI_TokenizeInput | Wrapper around CLI_TokenizeLine with session parameters (separator space, max. 8 tokens). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804ca46` | CLI_StripQuotes | Removes leading/trailing quotation marks from all tokens. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804caae` | CLI_CursorForward | Cursor one character to the right (re-output the character). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804cacc` | CLI_ExecuteCommand | Executes command entry depending on type (direct handler, InvokeCommandHandler with args, select, FormatEntryValue). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804cd00` | CLI_TokenizeLine | Core tokenizer: splits line at separators, respects quotation marks + backslash escapes. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804ce18` | CLI_CommonPrefixLength | Computes common prefix length of two strings (for tab complete). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804ce5c` | CLI_TabComplete | Tab-complete logic: finds matching entries by prefix match, shows candidates or completes a unique match. (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804cffa` | CLI_MoveForward | Wrapper around CLI_HistoryNavigate(1) — history forward (arrow down). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |
| `0x0804d008` | CLI_PutChar | Low-level character output via session output function pointer (offset +0x68). (2026-07-15 moved from CLI/debug output/logging to CLI/AT-Command-Engine) | high | Re-audit 2026-07-14 |

## CLI / debug output / logging (18)

> **Update (2026-07-15):** The 28 line editor / tokenizer / command dispatch functions previously misclassified here
> (`CLI_DeleteCharForward` through `CLI_PutChar`) were moved to the cluster "CLI / AT-Command-Engine"
> (user decision). Remaining: pure debug output/logging functions.

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08005fb8` | Debug_Mode_Set | Enable/disable debug mode via param 0xC2 | high | Re-Audit 2026-07-14 |
| `0x08028fb0` | EventLog_Record_DisplayError | 14B entries, EEPROM 0x1100, 20-slot ring buffer, 31s deduplication | high | Re-Audit 2026-07-14 |
| `0x08029124` | EventLog_Record_SystemEvent | 9B entries, EEPROM 0x2000, 20 slots, events 0x191/0xcb/0x19b/0x259/600 | high | Re-Audit 2026-07-14 |
| `0x0802dd70` | debug_printf | Conditional debug printf: only outputs if the debug flag (offset+0x38b) is set, then calls putchar-based vararg output | high | Re-Audit 2026-07-14 |
| `0x0803238c` | Log_WarnCode0xE8 | Warning log with code 0xE8 | high | Re-Audit 2026-07-14 |
| `0x08034974` | Debug_PrintPowerStatistics | 6 power values (all/monthly/daily charge/discharge) | high | Re-Audit 2026-07-14 |
| `0x08034b94` | Debug_PrintErrorCodes | err_code + warn_code as hex | high | Re-Audit 2026-07-14 |
| `0x08035c8c` | Debug_Print_MeterMac | Print meter MAC address | high | Re-Audit 2026-07-14 |
| `0x0803e1b0` | log_SetEnabled | Switch logging on/off (bool flag) | high | Re-Audit 2026-07-14 |
| `0x0803e1c4` | log_printf | Central log output with level/module filter (enabled flag, max level, module filter), prefix [module][ERROR/WARN/INFO/DEBUG/UNKWN]; 386 callers project-wide | high | Re-Audit 2026-07-14 |
| `0x0803e2d0` | log_SetLevel | Log level 0-4 (TRACE/DEBUG/INFO/WARN/ERROR) | high | Re-Audit 2026-07-14 |
| `0x0803e2e4` | log_SetModeAndLevel | Set enable + level combined (ETX signal) | medium | Re-Audit 2026-07-14 |
| `0x0804d210` | Log_SetModeAndLevelCallback | Thin wrapper, calls log_SetModeAndLevel(param) — 0 callers, presumably a function pointer callback | medium | Re-Audit 2026-07-14 |
| `0x0804d2d4` | Debug_PrintErrorAndEventLog | Debug dump: param=0 outputs 20 error log slots, param=1 outputs 20 system event log slots (EventLog_Record_SystemEvent buffer) | high | Re-Audit 2026-07-14 |
| `0x0804d498` | Debug_PrintWifiStatus | Outputs WiFi signal strength and connection status (2 printf) | high | Re-Audit 2026-07-14 |
| `0x0804d7e4` | Debug_PrintModbusAddress | Outputs the configured Modbus address | high | Re-Audit 2026-07-14 |
| `0x08050dd8` | Debug_Mode_ToggleWithLog | Toggle debug mode with log output | high | Re-Audit 2026-07-14 |
| `0x0802d00c` | Unused_FuncCall_Wrapper | 0 callers, 12B passthrough wrapper | medium | Docs |

## System / reset / shutdown / watchdog (16)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080001d0` | System_SoftReset | Reset MSP from VTOR, re-enable IRQs, SVC #0 soft reset | high | Re-Audit 2026-07-14 |
| `0x08007946` | FactoryReset_NotifyAndReboot | Cloud notification + reboot after factory reset | high | Re-Audit 2026-07-14 |
| `0x0800b724` | Factory_Reset | Dispatcher: param=1 EEPROM clear+factory write+notify+reboot, param=2 notify without EEPROM clear, param=3 EEPROM clear+2s delay+reboot without notify | high | Re-Audit 2026-07-14 |
| `0x08013eb4` | System_IsOperationBusy_Flag | Check whether the operation is marked as 'busy' | high | Re-Audit 2026-07-14 |
| `0x08013f68` | System_ResetRuntimeBuffers_Init | Re-initialize runtime buffers on reset | high | Re-Audit 2026-07-14 |
| `0x0801d204` | ConditionalSystemReboot | Guard-checked reboot (counter 5-301) | high | Re-Audit 2026-07-14 |
| `0x08029900` | Comm_Status_Flags_Reset | Status flags reset (4 callers: serial, CAN, OTA) | high | Re-Audit 2026-07-14 |
| `0x08029c78` | System_Reboot | Stops inverter output, saves runtime counter + reboot timestamp, prints "system will reboot", 50ms delay, then NVIC reset via AIRCR register (infinite loop) | high | Re-Audit 2026-07-14 |
| `0x0802d268` | MainLoop_Periodic_Tasks | Central periodic dispatcher (13 subsystems called; 1 caller: App_MainLoopDispatcher) | high | Re-Audit 2026-07-14 |
| `0x0802d2b4` | Idle_Watchdog_AutoReboot | 3600-tick idle → config apply → inverter off → reboot | high | Re-Audit 2026-07-14 |
| `0x0802d364` | Shutdown_Sequence_Handler | Suspend 7 RTOS tasks, register dirty, 2000ms wait, reboot | high | Re-Audit 2026-07-14 |
| `0x0802d6fc` | Shutdown_SetPendingFlag | Set pending flag (offset+2) to 1 | medium | Re-Audit 2026-07-14 |
| `0x0802d728` | System_ResetToDefaults | Write config, resume RTOS tasks, clear state | medium | Re-Audit 2026-07-14 |
| `0x0802fca4` | Comm_Watchdog_CheckTimeout | 500ms poll, 60 fails (~30s) → OTA slot failed + reset | high | Re-Audit 2026-07-14 |
| `0x08034812` | App_MainLoopDispatcher | 5 subsystems: shutdown, OTA update, CAN update, periodic tasks, cloud watchdog | high | Re-Audit 2026-07-14 |
| `0x0804916c` | System_StopAllTimers | Stop **7 timer handles** before reboot | high | Re-Audit 2026-07-14 |

## libc / standard library / memory utilities (45)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x0800050a` | Mem_CopyBytes | Byte-wise memory copy | high | Docs |
| `0x08000522` | Mem_CopyWords_Aligned | Word-aligned memory copy (4 words/iteration, fallback to byte copy) | high | Docs |
| `0x08000574` | Mem_XorBytes | XOR two byte buffers together | high | Docs |
| `0x080005c8` | Mem_ZeroWords | Zero memory in 4-word chunks | high | Docs |
| `0x080006a0` | CompareWordArrays | Compare two word arrays, 0=equal, 1=different | high | Docs |
| `0x08000786` | FillMemoryFromWord | Fill buffer with repeated 32-bit value | medium | Docs |
| `0x08000820` | UInt64Div | 64-bit unsigned division (shift-and-subtract, for printf) | high | Docs |
| `0x08000884` | GetCtypeTable | Return pointer to the ctype classification table | high | Docs |
| `0x0800088c` | IsSpace | Whitespace check via ctype table | high | Docs |
| `0x0800089e` | memcpy | Standard memcpy, word-wise with byte remainder; detects overlap (destination>source) and then copies backwards (memmove-safe) | high | Re-Audit 2026-07-14 |
| `0x080008de` | MemFill | Core memset implementation (fill loop) | high | Docs |
| `0x080008ec` | memset | Only 2 parameters (ptr,len), calls MemFill(ptr,len,0) — val fixed to 0, no real 3-param C API (no rename needed, memset is the most fitting available name despite the differing signature) | medium | Re-Audit 2026-07-14 |
| `0x080008f0` | MemsetStdcall | Standard C `memset(ptr, val, len)` wrapper | high | Docs |
| `0x08000902` | StrCat | Standard `strcat` | high | Docs |
| `0x0800091a` | strstr | Standard strstr (naive substring search) | high | Re-Audit 2026-07-14 |
| `0x0800093e` | strncpy | Standard strncpy with null padding up to n | high | Re-Audit 2026-07-14 |
| `0x08000956` | strchr | Name corrected (2026-07-15, formerly strncat): 2 parameters (ptr,char), searches for a character in the string, corresponds to strchr | - | Re-Audit 2026-07-14 |
| `0x0800096a` | strlen | Standard strlen | high | Re-Audit 2026-07-14 |
| `0x08000978` | strcmp | Standard strcmp | high | Re-Audit 2026-07-14 |
| `0x08000994` | memcmp | Byte-wise memory comparison up to n or first difference (memcmp) | high | Re-Audit 2026-07-14 |
| `0x080009ae` | strcpy | Standard `strcpy` | high | Docs |
| `0x080009c0` | strncmp | Name corrected (2026-07-15, formerly atoi): 3 parameters (s1,s2,n), pure byte comparison, corresponds to strncmp | - | Re-Audit 2026-07-14 |
| `0x080009e0` | strtok | Core of strtok with explicit save pointer (strtok_r-like); strtok_wrapper wraps this as the classic 2-parameter strtok(str,delim) | high | Re-Audit 2026-07-14 |
| `0x08000a1c` | strtok_wrapper | Thin wrapper around strtok | high | Docs |
| `0x08000a24` | strpbrk | Find first occurrence of a character from a set | high | Docs |
| `0x08000a44` | calloc | malloc + memset(0) | high | Docs |
| `0x08000a60` | sscanf | Variadic sscanf (setup + delegation to parser) | high | Docs |
| `0x08000a98` | sscanf_ReadInteger | Integer conversion engine for sscanf | high | Docs |
| `0x08000be4` | strtol | String-to-long with sign and overflow clamping | high | Docs |
| `0x08000c54` | strtol_SaveErrno | strtol variant that saves/restores errno | high | Docs |
| `0x08000cc6` | atoi | Name corrected (2026-07-15, formerly atoi_u16): calls strtol(s,NULL,10) and restores errno — exact standard atoi behavior, no 16-bit specifics | - | Re-Audit 2026-07-14 |
| `0x08001020` | uint32_div | Software 32-bit unsigned division | high | Docs |
| `0x0800104c` | uint64_shl | 64-bit left shift (lo/hi pair) | high | Docs |
| `0x080010d8` | sscanf_Format_Parser | Format string parser for sscanf (% directives), calls sscanf_ReadInteger for number conversion | high | Re-Audit 2026-07-14 |
| `0x08001134` | strtoul_internal | Core of strtoul (base detection, accumulation) | high | Docs |
| `0x08030368` | printf | Variadic printf, setup + delegation to printf_Format_Engine | high | Re-Audit 2026-07-14 |
| `0x08030388` | snprintf | Variadic snprintf with buffer size limit, delegation to printf_Format_Engine | high | Re-Audit 2026-07-14 |
| `0x080303e4` | putchar | putchar via printf_Format_Engine (single character as a format string call) | high | Re-Audit 2026-07-14 |
| `0x08030424` | __errno_location | Pointer to the global errno variable | very high | Docs |
| `0x08031300` | malloc | Heap allocator with free list | very high | Docs |
| `0x08031424` | printf_Float_To_Digits | Float→decimal digits via power of 10 + UInt64Div | high | Docs |
| `0x080315a8` | printf_Format_Engine | Core engine for printf/snprintf/putchar: parses format string, handles %d/%s/%f/... specifiers, padding, float digit generation | high | Re-Audit 2026-07-14 |
| `0x08031c5c` | printf_Pad_Trailing_Spaces | Trailing space padding (left-align '-') | high | Docs |
| `0x08031c80` | printf_Pad_Leading | Leading '0'/space padding (right-align) | high | Docs |
| `0x08031cc4` | sprintf_Output_Char | Putchar callback: write byte, pointer++ | high | Docs |

## Fixed-point math (fp64) / dtoa / float formatting (28)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x08000ce0` | fp64_add | Software IEEE754 addition of two fp64 values (mantissa alignment via shift, rounding) | high | Re-Audit 2026-07-14 |
| `0x08000e22` | fp64_sub | Software fp64 subtraction (tail call to fp64_add with inverted sign bit of operand B, set in assembly before the jump) | medium | Re-Audit 2026-07-14 |
| `0x08000e28` | fp64_neg | Negates sign bit of operand A, adds operand B (fp64_add with XOR 0x80000000); Ghidra recognizes only 2 of 4 parameters | medium | Re-Audit 2026-07-14 |
| `0x08000e2e` | fp64_mul | Software IEEE754 multiplication of two fp64 values (mantissa multiplication + exponent addition via fp64_normalize) | high | Re-Audit 2026-07-14 |
| `0x08000f12` | int_to_fp64 | Signed int32 to fp64 (form absolute value, fp64_normalize with exponent bias 0x433) | high | Re-Audit 2026-07-14 |
| `0x08000f34` | uint_to_fp64 | Unsigned int32 to fp64 (fp64_normalize with exponent bias 0x433, without sign handling) | high | Re-Audit 2026-07-14 |
| `0x08000f4e` | fp64_to_int | fp64 to signed int32 (exponent range check, shift via fp64_abs_cmp, apply sign) | high | Re-Audit 2026-07-14 |
| `0x08000f8c` | fp64_to_uint | fp64 to unsigned int32 (analogous to fp64_to_int, without applying the sign) | high | Re-Audit 2026-07-14 |
| `0x08000fc0` | fp64_div | Software fp64 division; Ghidra decompilation incomplete (only 2 of presumably 4 parameters visible, callers pass 4) — name plausible according to call context (division operations in cJSON_PrintNumber/SMR_TIC) | medium | Re-Audit 2026-07-14 |
| `0x08000ff0` | fp64_cmp_ge | fp64 greater-or-equal comparison; same decompilation limitation as fp64_div (identical visible code, actual logic not fully decompiled) — name plausible according to call context (branch conditions) | medium | Re-Audit 2026-07-14 |
| `0x0800106a` | fp64_abs_cmp | 64-bit right shift of (param1:param2) by param3 bits; helper function for mantissa alignment in fp64_normalize/fp64_to_int/fp64_to_uint/fp64_sqrt | high | Re-Audit 2026-07-14 |
| `0x0800108a` | fp64_ArithmeticShiftRight | 64-bit arithmetic right shift with sign extension | high | Docs |
| `0x080011f0` | fp64_normalize | Core normalization for fp64 results: leading zero count, mantissa shift to 53 bits, exponent bias adjustment, rounding (round-to-nearest-even) | high | Re-Audit 2026-07-14 |
| `0x0800128c` | fp64_div_impl | Software FP64 division (bit by bit, IEEE 754 rounding) | high | Docs |
| `0x0800136a` | fp64_ScaleByPowerOf2 | Adjust the exponent of an FP64 value | high | Docs |
| `0x08001398` | fp64_ToInteger | Truncate FP64 toward zero (mask fraction bits) | medium | Docs |
| `0x0800171c` | fp64_sqrt | Software FP64 square root (53 iterations) | high | Docs |
| `0x080303f4` | fp64_Classify | Double classification (zero/subnormal/inf/NaN) | high | Docs |
| `0x08030488` | fp64_Low_Word_Identity | Identity function for soft-float low word | medium | Docs |
| `0x080304a0` | fp64_floor | IEEE 754 floor() implementation | high | Docs |
| `0x080305b8` | fp64_pow | Name corrected (2026-07-15, formerly dtoa_Float_To_String): implements x^y (2900 bytes, range reduction+polynomial evaluation, special cases sqrt/squaring/overflow), no float→string conversion; only caller cJSON_Parse_Number uses it as pow(10,exponent) when parsing JSON numbers. Corresponds to standard C pow() | - | Re-Audit 2026-07-14 |
| `0x08031208` | fp64_Polynomial_Eval | Horner scheme polynomial evaluation | very high | Docs |
| `0x08031350` | fp64_pow_OverflowInfinity | Name corrected (2026-07-15, formerly dtoa_Generate_Infinity): part of the overflow path of fp64_pow, called after fp_Set_Exception(2), not a dtoa function | medium | Re-Audit 2026-07-14 |
| `0x080313b8` | fp64_pow_OverflowConst_A | Name corrected (2026-07-15, formerly dtoa_Square_Constant_A): squares a fixed overflow constant to produce a correctly flagged +Inf result in the pow() overflow path, not a dtoa function | high | Re-Audit 2026-07-14 |
| `0x080313d8` | fp64_pow_OverflowConst_B | Name corrected (2026-07-15, formerly dtoa_Square_Constant_B): like fp64_pow_OverflowConst_A, second variant — part of the pow() overflow path, not a dtoa function | high | Re-Audit 2026-07-14 |
| `0x08031418` | fp_Set_Exception | Store inexact/invalid exception code | high | Docs |
| `0x08034ce4` | Double_Fabs | Clear sign bit (& 0x7FFFFFFFFFFFFFFF) | high | Docs |
| `0x0804d9c8` | fp64_sqrt_WithExceptionCheck | sqrt with exception flag setting on overflow edge case (calls fp64_sqrt, checks result exponent against input exponent, sets fp_Set_Exception(1) on inexact) | high | Re-Audit 2026-07-14 |

## Heap allocator (internal) (7)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x0803042c` | heap_Free_Coalesce | Free list with block coalescing (core of free()) | high | Docs |
| `0x0804b500` | heap_Realloc | Coalesce + re-alloc, fallback restore | high | Docs |
| `0x08050d18` | Heap_TrackedFree | Free with tracking/counting | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050d4c` | Heap_TrackedMalloc | Malloc with tracking/counting | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x08050d88` | Heap_TrackedRealloc | Realloc with tracking/counting | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080364c8` | Heap_AllocFromFreeList | First-fit allocator, block splitting, called by malloc | high | Docs |
| `0x08048f64` | Heap_Calloc | calloc: malloc + memset(0), 0 callers (fn pointer) | high | Docs |

## Utility / byte helpers / timer helpers (46)

| Address | Name | Description | Conf. | Source |
|---|---|---|---|---|
| `0x080005f6` | ByteSwap32Array | Endian swap of a 32-bit word array | high | Docs |
| `0x0800068a` | ByteSwap32ArrayWrapper | Wrapper around ByteSwap32Array, returns 0 | high | Docs |
| `0x080010ae` | CharToDigitValue | ASCII → digit value for a given base | high | Docs |
| `0x080017be` | LZ77_Decompress | LZ77-style decompression: literal runs, back references AND zero runs (RLE for zero bytes, when control byte bit 0x08 is not set) | high | Re-Audit 2026-07-14 |
| `0x08001fc8` | LookupTable_Read16 | Read 16-bit value from lookup table | medium | Docs |
| `0x08004cd0` | Parse_IP_Address_String | Parse IP address from string (dotted notation) | high | Docs |
| `0x08005644` | Version_Compare_Fields | Compare version numbers field by field (range 2-4) | high | Docs |
| `0x08005758` | Byte_Swap_Copy | Copies param3 bytes from param2 to param1 in reverse byte order (generic variable-length endian swap); used e.g. for Modbus registers in TCP/RS485 handlers (6 callers) | medium | Re-Audit 2026-07-14 |
| `0x080066cc` | Delay_BusyWait_Ms | Blocking busy-wait delay, ms scale | high | Docs |
| `0x080066ec` | Delay_BusyWait_Us | Blocking busy-wait delay, µs scale | medium | Docs |
| `0x0800685a` | DNS_QuestionRecord_Skip | Skip TLV record, advance pointer | high | Docs |
| `0x080068f4` | Delay_Adaptive | vTaskDelay if the scheduler is running, otherwise busy-wait | high | Docs |
| `0x08013f24` | Util_XOR_Checksum_Calc | Calculate XOR checksum | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
| `0x080195f4` | Uint32_To_DecString | uint32 → ASCII decimal string conversion | high | Docs |
| `0x0801df44` | HexChar_To_TimeOffsetIndex | Hex char → numeric % 5 (NTP offset) | medium | Docs |
| `0x08025f34` | Checksum_Accumulate | Accumulates byte sum into *param_1 over param_3 bytes starting at param_2; if param_4≠0 the result is inverted (one's complement) — used for BLE OTA chunk checksums | medium | Re-Audit 2026-07-14 |
| `0x08025f60` | Timer_CheckElapsed | Check timer expiry (ms-based) | high | Docs (name match) |
| `0x08029370` | Heartbeat_Timer_Reset | Flag=1, timer clear, start 3000ms timer | medium | Docs |
| `0x08029818` | Periodic_RTC_Display_Format | Every 5000ms: reads the date via RTC_GetDateTime(), formats one field as an ASCII digit, 3 further bytes copied raw into the display buffer (RTC date, not generic "sensor" data) | medium | Re-Audit 2026-07-14 |
| `0x0802c174` | Timer_Defaults_Init | Initialize 13 timer/limit defaults (500-60000ms) | high | Docs |
| `0x0802d768` | ParseIntegerWithSuffix | 0x/decimal + k/M suffix (×1024/×1M), 0 callers | high | Docs |
| `0x0802d9ca` | Periodic_Tick_Handler | SysTick ISR handler: checks scheduler status, calls FreeRTOS_SysTick_TaskUnblock() if applicable, then always Runtime_Energy_Counter_Tick() | medium | Re-Audit 2026-07-14 |
| `0x0802dc24` | Tick_Timer_Check_Elapsed | Tick-based elapsed timer (19 callers) | high | Docs |
| `0x0802dc58` | Timer_Periodic_RunCallback | Periodic timer with callback (13× from Timer_Defaults_Init) | high | Docs |
| `0x0802dc98` | Tick_Timer_Expired | Tick-based elapsed timer, identical structure to Tick_Timer_Check_Elapsed, own global tick counter (18 caller functions / 45 call sites) | high | Re-Audit 2026-07-14 |
| `0x0802dccc` | Parse_TimeString_To_HourMinute | Parses "HH.MM" via sscanf, validates HH<24/MM<60; per disassembly the result is NOT hour<<8\|min but hour<<(min+8) — potential bit loss for larger minute values (suspected FW bug) | high | Re-Audit 2026-07-14 |
| `0x0802dd18` | Timeout_Start_Seconds | Convert seconds→ms, then Timeout_Start | high | Docs |
| `0x0802dd2e` | Timeout_Start | Store duration, capture start tick (6 callers) | high | Docs |
| `0x0802dd3e` | Timeout_Init | Zero 12-byte timeout struct (6 callers) | high | Docs |
| `0x0802dd48` | Timeout_IsExpired | Check whether timeout has expired (4 callers) | high | Docs |
| `0x0802dd5e` | Timeout_RefreshAndGetTicksRemaining | Process ptr+1, return *ptr | medium | Docs |
| `0x080322fc` | Stream_ReadBytesAdvance | Read bytes from stream pointer + advance | medium | Docs |
| `0x08035394` | Read_BigEndian_UInt16 | 2 bytes big-endian → uint16 | high | Docs (name match) |
| `0x08036494` | Util_CalcHalfSizeCapped | ceil(param/2), but only if param<48 — for param≥48 fallback to 0 (input limit with fallback, no result cap) | medium | Re-Audit 2026-07-14 |
| `0x0803e304` | Delay_Ms | Scheduler-aware: vTaskDelay or busy-wait (9 callers) | high | Docs |
| `0x0803e340` | Delay_BusyWaitUs | Calibrated µs busy-wait from system clock | high | Docs |
| `0x0803e360` | RTOS_Delay_Ms | Scheduler-state-dependent delay (counterpart to Delay_Ms, uses Delay_BusyWait_Ms): busy-wait when the scheduler is halted/not started (xTaskGetSchedulerState), otherwise vTaskDelay | high | Re-Audit 2026-07-14 |
| `0x08048f3c` | ByteSwap32 | Wrapper → ByteSwap32_Internal | high | Docs |
| `0x08048f48` | ByteSwap32_Internal | 4 bytes big-endian → uint32 | high | Docs |
| `0x0804c370` | Util_HexCharToNibble | Converts ASCII hex character (0-9, a-f, A-F) to nibble value 0-15; invalid characters yield 0 | high | Re-Audit 2026-07-14 |
| `0x0804ce44` | Util_StrCopy | Copies null-terminated string (strcpy equivalent, no length limit/bounds check) | high | Re-Audit 2026-07-14 |
| `0x0804cf6c` | Util_IntToDecStr | Converts signed integer to decimal ASCII (backwards into 12-byte buffer), incl. sign/zero special case; returns start index | high | Re-Audit 2026-07-14 |
| `0x0804cfc2` | Util_IntToHexStr | Converts unsigned integer to hex ASCII (lowercase, backwards into 9-byte buffer); returns start index | high | Re-Audit 2026-07-14 |
| `0x08003dd0` | CRC16_Calculate | CRC-16 with dual lookup tables, init 0xFFFF | high | Docs |
| `0x08012d8c` | Generic_StructField_Set_0x14 | Set generic struct field at offset 0x14 (counterpart to Generic_StructField_Set_0x10); for GPIO structures, offset 0x14 corresponds to the BRR register (pin reset) | medium | Re-Audit 2026-07-14 |
| `0x08012d90` | Generic_StructField_Set_0x10 | Set generic struct field at offset 0x10 | Batch 20 | Ghidra (Batch 20, 2026-07-09) |
