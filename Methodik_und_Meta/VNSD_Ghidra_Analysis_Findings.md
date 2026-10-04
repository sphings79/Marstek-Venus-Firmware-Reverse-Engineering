# Marstek Venus D — Ghidra Analysis Findings

> 🇩🇪 Deutsche Version: [VNSD_Ghidra_Analyse_Erkenntnisse_de.md](VNSD_Ghidra_Analyse_Erkenntnisse_de.md)

**Project:** Marstek Venus D (`VNSD-0`) reverse engineering  
**Reference:** Split out of [VNSD_Control_FW_Analysis_app_1492_0702_142136.md](../Control_FW/VNSD_Control_FW_Analysis_app_1492_0702_142136.md) (formerly sections 14 + 15)

---

## 1. Misidentified function #1

```
Address: FUN_080128a0 (contains MOVW #0x7530 at 0x0801295e)
NOT:     descriptor init function
IS:      JSON config parser

Evidence: decompiled code shows
  DAT_20018b6b = 30000;  // ← 0x7530 here is a default API port!
  FUN_08045340(param_1, &DAT_08012b04, &DAT_20018b68);  // JSON parser
```

## 2. Misidentified function #2

```
Address: FUN_0801bcb0
NOT:     descriptor init function
IS:      RS485/RTU packet handler

Evidence: processes Modbus RTU frames (CRC check, FC03/FC06/FC10 routing)
  DAT_200151ec  = RS485 receive buffer
  FUN_08025f3c  = Modbus FC03 handler
  FUN_08026080  = Modbus FC06 handler
  FUN_08026154  = Modbus FC16 handler
```

## 3. Correctly identified functions

| Function | Status | Source |
|---|---|---|
| `FUN_0801c088` = router | ✅ Confirmed | TCPRouter.c decompiled |
| `FUN_0804b73c` = serializer | ✅ Confirmed | Serializer.c decompiled |
| `FUN_0804c83c` = write handler | ✅ Confirmed | WriteHandler.c decompiled |

## 4. Ghidra script problems

- `getReferencesFrom()` does not find **computed** SRAM addresses
- Ghidra tracks only direct literal references
- Solution: add SRAM block + re-analysis → then use the decompiler
- Jython 2.7: no umlauts/special characters without `# -*- coding: utf-8 -*-`

---

## 5. Network architecture (CH395 Ethernet, CLI shell, Modbus TCP)

### 5.1 CH395 Ethernet chip: socket assignment

Initialization takes place in `CH395_Reset_And_Reinit @ 0x08029964` (144 bytes).
Initialization order: HardwareReset → Init_Basic → RS485_Modbus_RegisterMap_Init → Init_Full → Init_Modbus_TCP_Socket → Init_BroadcastListener_Socket → **Init_TCPServer_Socket**.

| Socket | Port | Function | Init function |
|--------|------|----------|---------------|
| 0 | **502** (0x1F6) | Modbus TCP | `CH395_Init_Modbus_TCP_Socket @ 0x08048754` |
| 1 | **8091** (0x1F9B) | TCP debug/CLI shell | `CH395_Init_TCPServer_Socket @ 0x08032C0C` |

**`CH395_Init_TCPServer_Socket` (66 bytes @ 0x08032C0C):**
```c
void CH395_Init_TCPServer_Socket(void) {
  CH395_SPI_CmdWaitReady(1);
  *(uint8*)(descriptor + 0x40) = 2;    // protocol = TCP
  *(uint8*)(descriptor + 0x41) = 1;    // socket = 1
  *(uint16*)(descriptor + 0x48) = 0x1F9B; // src port = 8091
  *(uint16*)(descriptor + 0x4A) = 0x1F9B; // dst port = 8091
  *(uint8*)(descriptor + 0x42) = 2;    // mode = server/listen
  CH395_Socket_Open_ByDescriptor(descriptor + 0x40);
}
```

**`CH395_Init_Modbus_TCP_Socket` (50 bytes @ 0x08048754):**
```c
void CH395_Init_Modbus_TCP_Socket(void) {
  *(uint8*)(descriptor + 0x1C) = 2;    // protocol = TCP
  *(uint8*)(descriptor + 0x1D) = 0;    // socket = 0
  *(uint16*)(descriptor + 0x24) = 0x1F6; // src port = 502
  *(uint16*)(descriptor + 0x26) = 0x1F6; // dst port = 502
  CH395_Socket_Open_ByDescriptor(descriptor + 0x1C);
}
```

### 5.2 CLI architecture: data path TCP → dispatcher

```
TCP Socket 1 (Port 8091)
    ↓  [CH395 receives data]
RAM flag @ 0x20000132 = 1  (socket-type selector)
    ↓  [dispatch loop at ~0x0802DACC]
Network_ReceiveAndDispatchData @ 0x0804D4E8  (via BL @ 0x0802DBE2)
    ↓  [reads CH395 data or queue]
BLE_GATT_DispatchCommandByte @ 0x0804C4B4  (byte-by-byte CLI dispatch)
    ↓
CLI_TabComplete / CLI_ListCommands / command handler
```

**Callers of `Network_ReceiveAndDispatchData`:**
- No caller in Ghidra's reference manager (BL was not recognized)
- Manual BL scan: BL **@ 0x0802DBE2** → 0x0804D4E8
- Condition: `LDRB R0, [0x20000132]; CMP R0, #1; BEQ → BL`

**Caller context at 0x0802DBE2 (dispatch loop, function not recognized by Ghidra):**
```c
// ~0x0802DBD0: dispatch block (socket-type selector)
if (RAM[0x20000270] != 0) {
    SomeFunctionA();           // socket type 0 path
} else if (RAM[0x20000132] == 1) {
    Network_ReceiveAndDispatchData();  // TCP socket 1 path → CLI port 8091
} else if (RAM[0x20000132] == 2) {
    SomeFunctionB();           // socket type 2 path
}
```

**`Network_ReceiveAndDispatchData @ 0x0804D4E8` (184 bytes) — dual input path:**
- Reads data from CH395 TCP socket 1 (port 8091) **or** from a queue (BLE/RS485 path)
- Forwards bytes to `BLE_GATT_DispatchCommandByte` (byte-by-byte CLI parser)
- Callers from Ghidra: 0x0804D552 + 0x0804D598 in itself → both inside the function

### 5.3 Unrecognized main task function (~0x0802DA0C)

The code range **0x0802DA0C–0x0802DC20** was not recognized as a function by Ghidra (all attempts to create functions there failed or produced 1-byte stubs). Contents (reconstructed from raw bytes):

- **GPIO/hardware init**: bit set on 0x40021000 (GPIOE) → probably RS485 transceiver enable
- **Inner loop at 0x0802DACC**: checks state, calls `vTaskDelay(100)`, loops back → FreeRTOS task main loop
- **Dispatch block**: reads RAM flags, calls either the CLI path or other handlers
- Also contains `BL Modbus_Drain_RxBuffer @ 0x080292D4` → Modbus buffer draining

**`App_MainLoopDispatcher @ 0x08034812` (24 bytes)** — probably called from this range (0 Ghidra callers, no pointer in flash):
```c
void App_MainLoopDispatcher(void) {
  Shutdown_Sequence_Handler();
  FUN_080151c8();
  CAN_Update_StateMachine();
  MainLoop_Periodic_Tasks();
  Cloud_EdgeDetectAndWatchdog();
}
```

### 5.4 FreeRTOS xTaskCreate — task table problem (unsolved)

`FUN_080545fc @ 0x080545FC` = **xTaskCreate** (confirmed by pvPortMalloc×2 + Task_InitNewTask + prvAddNewTaskToReadyList).

`Task_Init_CreateAll @ 0x0801947C` creates 14 tasks via a loop:
```c
for (i = 0; i < 14; i++) {
    xTaskCreate(TABLE[i].pvTaskCode,  // entry + 0x00
                TABLE[i].pcName,     // entry + 0x04
                TABLE[i].stackDepth, // entry + 0x08 (uint16)
                TABLE[i].pvParam,    // entry + 0x0C
                TABLE[i].priority,   // entry + 0x10
                TABLE[i].handle);    // entry + 0x14
}
```

- **TABLE_BASE** = `Mem[0x08019518]` = **0x08059958**
- **Stride** = 24 bytes (0x18) ← confirmed by the instruction `ADD.W R0, R4, R4, LSL#1; ADD.W R0, TABLE, R0, LSL#3`
- **Problem**: 0x08059958 contains ASCII text in flash (`"[BLE] URL count sequence error!"`) — no valid function pointers
- **Hypothesis**: the task table resides in the `.data` section (RAM VMA ≠ flash LMA) or the function is called with a RAM pointer as a parameter that is not visible in the flash analysis

**xTaskCreate call sites (manual BL scan):**
| Address | Function | Task |
|---------|----------|------|
| 0x080194CE | Task_Init_CreateAll | 14 app tasks (table) |
| 0x08050A8A | (FreeRTOS internal) | IDLE task (prio 0, stack 128) |
| 0x08054FBC | (FreeRTOS internal) | Timer task (`FUN_0804f910`) |

---

## 7. Red herrings & lessons

### 7.1 MOVW #0x7530 is not a reliable anchor

The value 0x7530 = 30,000 appears several times in the code with different meanings:
- **API port** (30000 as default port in the JSON config parser)
- **Comparison value** (range check: `if reg < 30000`)
- Possibly also as the first register address in the init code

### 7.2 0x9C40 (40,000) serves as a range check

In WriteHandler.c and TCPRouter.c, 0x9C40 appears as the boundary between read and write registers. That makes it unsuitable as an anchor for the descriptor init.

### 7.3 MOVW with "table offset values"

75 MOVW instructions with values 0x02F8–0x0DCC were found. However, these are all **power limits** (0x09C4 = 2500W, 0x0BB8 = 3000W), not table offsets.

### 7.4 Startup literal pool at 0x08055c8c

Looked promising as the .data section source. Actually contains only MQTT debug strings, no register descriptor data.

### 7.5 Register 34010 / 34110 / 37012 misinterpreted as temperature

**Symptom:** Raw value 116 was interpreted with scale 0.1 as 11.6 °C —
a value that seemed plausible for the season (May, cool battery).

**Correction:** All three registers contain the **BMS firmware version (116)**,
identical to reg 30204 (`bms_version = 116`).

| Register | Wrong | Correct |
|---|---|---|
| 34010 | pack1_temp_avg = 11.6 °C | **pack1_bms_version = 116** |
| 34110 | battery_2_temperature = 11.6 °C | **battery_2_bms_version = 116** |
| 37012 | max_temp_all_packs = 11.6 °C | **bms_version (system level) = 116** |

**Lesson:** If a raw value happens to make a temperature interpretation plausible,
always cross-check against known values with the same meaning (here reg 30204).
The triple match with 30204 should have been noticed earlier.
The actual pack average temperature is probably at 34012–34016
(raw values 192–198 → 19.2–19.8 °C), which fits an indoor-operated battery better.

## 8. Batch 18 — Systematic verification pass (2026-07-09)

The user pointed out that an earlier session had already mentioned misidentified/
misdecompiled functions. A spot check showed: the already documented
naming conflict #66 (`mbedTLS_MPI_Mul_MPI`/`Exp_Mod`, originally from batch 13) had
**never actually been fixed in Ghidra** — only the doc line "Interesting
finds" mentioned it. A complete re-check of the entire `mbedTLS_MPI_*`/`ECP_*`/`RSA_*`/
`PK_`/`SSL_`/`X509_` cluster (142+ functions) as well as all 153 medium/low-confidence
non-crypto functions (parallel Ghidra re-decompilation per address) found **62 incorrectly
named functions** in total, several of them in multi-stage shift chains
(e.g. Div_MPI ↔ Exp_Mod ↔ Inv_Mod; Add_Abs ↔ Grow ↔ Cmp_Abs ↔ Lset in the MPI cluster;
SSL Conf_Authmode ↔ Derive_Keys ↔ SessionFree ↔ TransformFree). For 8 addresses the name
stated in the docs had never actually been set in Ghidra (the function was still called
`FUN_0xxxxxxx` there) — docs and Ghidra status had already diverged beforehand.

All 62 were renamed in Ghidra. The current names are in
[VNSD_Control_FW_Function_Tracking_new.md](../Control_FW/VNSD_Control_FW_Function_Tracking_new.md); a summary
of the batch 18 procedure (the full old/new table no longer exists, see below) can be found in
[VNSD_Control_FW_Naming_Batch_History.md](../Control_FW/VNSD_Control_FW_Naming_Batch_History.md).

**Lesson (addendum to section 7):** A function name that occurs at **two different addresses**
was practically always an error signal in this check — in every case found,
exactly one of the two name holders was wrong. In future batch runs, a check for name duplicates
across the entire function inventory should run automatically at the end, instead of discovering
this only through a later full check. In addition, an "named in docs but
not applied in Ghidra" check should be added (occurred for 8 functions in this pass).

## 9. Batch 19 — Complete duplicate name resolution (2026-07-09)

While building the thematically regrouped [VNSD_Control_FW_Function_Tracking_new.md](../Control_FW/VNSD_Control_FW_Function_Tracking_new.md),
a **complete** name duplicate scan was performed this time over all 1417 Ghidra functions named
at that point (not just a spot check as in batch 18). Result:
besides the 6 duplicates already known (MQTT_Decode_RemainingLength, CH395_Socket_SendData,
mbedTLS_ASN1_Get_Int, mbedTLS_ECP_Group_Load, prvCopyDataToQueue, CLI_Backspace), **8 more** were found
in the llhttp internals (`llhttp__internal__c_test_flags` and `_1`/`_2`/`_3`, each at
two addresses with identical bit-test code but a different struct field offset).

All 14 were decompiled, compared and disambiguated: in each app/library pair,
one instance corresponds to the "canonical" function (e.g. the real mbedTLS public API signature,
or the actually referenced FreeRTOS static function behavior) and keeps the name,
while the other instance gets a clarifying suffix (`_ViaCallback`, `_RawParams`,
`_Core`, `_AndNotify`, `_EchoOnly`, `_ViaSocketPtr`). The llhttp pairs were named after the
distinguishing fact the decompilation yielded (struct offset `_Off32`/`_Off2e`),
instead of guessing unverifiable bit semantics.

Full table with rationales: [VNSD_Control_FW_Function_Tracking_new.md](../Control_FW/VNSD_Control_FW_Function_Tracking_new.md),
section "Batch 19". Result: 0 remaining name duplicates in the whole project (verified).

**Side finding:** The total number of named functions rose between the two Ghidra queries
in this session from 1411 to 1417 (6 new: `OTA_Update_Dispatcher`, `vTaskEnterCritical`,
`vTaskExitCritical`, `xTimerStop_Internal`, `FreeRTOS_StartScheduler`, `FreeRTOS_SysTick_TaskUnblock`) —
presumably Ghidra's background analysis kept running during the session. No cause for concern,
but a hint that function snapshots in active Ghidra projects should not be treated as
static/final.

**Lesson (addendum to section 8):** A duplicate check is only reliable if it runs over
*all* named functions, not just over a suspected risk area (in batch 18 the
focus was on the crypto cluster + medium/low-confidence functions, which let the llhttp internals
slip through). For future audits: pull `getFunctions()` in full, group by name,
disambiguate every group with >1 entry individually — regardless of how "safe" the respective
code area is considered to be.

## 10. Batch 20 — Identification of the last 206 unnamed functions (2026-07-09)

**Trigger:** The user asked to analyze the last remaining unidentified functions or
memory regions and to use subagents for parallel processing.

**Pre-check of the "memory regions":** `get-memory-blocks` shows only a single block `ram`
(`0x08000000`–`0x0805dfff`) — the firmware is a flat-mapped image, so there are no
separate unidentified memory regions beyond the 206 still unnamed functions.
`get-undefined-function-candidates` additionally returned 66 `LAB_*` jump targets without their own
function head (data references/jump-table-like targets) — classified as lower priority and
not processed in this batch.

**Methodology:** The 206 `FUN_0xxxxxxx` addresses were sorted by address, split into 6 chunks of ~35
and assigned to 6 parallel subagents. Each agent received the same project and
naming-convention context (examples from already named neighboring functions) plus its
specific address list, decompiled each function completely (incl. callers/callees) and proposed
a name with confidence + rationale — renames were NOT performed by the subagents themselves
but collected centrally.

Before applying, all 202 proposals were checked automatically: (1) for duplicates among themselves,
(2) for collisions with the 1417 names already assigned. Both checks produced 0 hits. The
renames were then applied in a single Ghidra transaction.

4 functions were deliberately not renamed (too trivial for a meaningful name):
`0x0800bcc0` (constant-return callback, 4 bytes), `0x0802da14`/`0x0802db54` (1-byte fragments,
probably faulty function boundaries) and `0x0802dc50` (2-byte `return;` stub).

**Result:** 202/206 functions successfully renamed. New project status: 1619 of 1623
functions named (99.75%), 0 name duplicates (verified by full scan). The newly identified
functions cover, among others, the complete mbedTLS X.509/ASN.1 certificate verification cluster
(33 functions, exact reconciliation against mbedtls 2.28 `x509_crt.c`/`asn1parse.c`), the complete
FreeRTOS kernel cluster (41 functions: task/queue/timer/list primitives such as `vTaskSwitchContext`,
`xQueueGenericSend`, `vListInsert`), the EEPROM I2C bit driver (14 functions) as well as various
Quectel AT command handlers (23 functions). Full table with all 202 renames by
topic: [VNSD_Control_FW_Function_Tracking_new.md](../Control_FW/VNSD_Control_FW_Function_Tracking_new.md), section
"Batch 20".

**Lesson:** For large, clearly delimitable remainders (here: all remaining `FUN_` functions),
subagent parallelization works well if (a) each agent gets enough context to
continue the established naming conventions correctly, and (b) the duplicate check runs centrally AFTER
collecting all proposals, not per agent — otherwise two agents can independently
propose the same name for different functions without either of
them noticing.

## 11. Review of LAB_ candidates + feasibility of TLS certificate extraction (2026-07-10)

**Trigger:** The user asked whether there is more to analyze and whether the 3 TLS credentials
stored in the device (CA certificate, user/client certificate, user private key) can be extracted from the
firmware.

**Part A — LAB_ candidates:** `get-undefined-function-candidates` queried again (the 66 candidates
deferred from batch 20). All 66 have `hasDataReference:true` and
`hasCallReference:false` — they are referenced exclusively as data addresses (jump-table
targets, constant pool entries, literal pool values inside existing functions), never as a
function call. This confirms the classification from batch 20: none of these 66 is a real
overlooked function. That completes all meaningfully identifiable functions/memory regions in
this firmware dump.

**Part B — TLS certificate extraction:** The decryption path was traced completely
(`CH395_MQTT_Init_And_CertSetup` → `TLS_Cert_Decrypt_All` (0x08024b54), `KeyDerive_CopyAndROT` +
`AES128_ECB_Decrypt`, custom AES-128-ECB implementation, NOT mbedTLS). The static extraction
of the 3 TLS credentials (CA root, device certificate, private key) was successful.

> **Security-relevant finding — deliberately not documented publicly.** The key required for decryption as well as the complete extraction methodology and the extracted credentials are
> **not part of this public repo**. The user reported the finding (a hardcoded AWS IoT client certificate
> including private key shared across product lines) confidentially to the vendor as part of a responsible
> disclosure; the details are withheld until a fix is available. Internal, non-public documentation on this is stored locally in the `security/` folder
> (not part of this repo).

**Lesson (general, without reference to the concrete key data):** With heavily nested/densely
packed constant pools, Ghidra's decompiler pointer resolution is not always trustworthy. For
byte-exact full copies of large binary data from Ghidra: have `run-script` (PyGhidra, `jpype.JArray`) write directly
to a file, do NOT go through several subagents via hex text retyping — the latter has no
reliable error detection against individual incorrectly reproduced bytes.
