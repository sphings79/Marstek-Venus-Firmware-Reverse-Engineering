# Marstek Venus D — CLI Shell: Architecture, Commands, Access Paths

> 🇩🇪 Deutsche Version: [VNSD_Shell_de.md](VNSD_Shell_de.md)

**Firmware:** v149.2 (`Control_149.2_VNSD-0_app_1492_0702_142136.bin`)  
**Reference:** [VNSD_Ghidra_Analysis_Findings.md](VNSD_Ghidra_Analysis_Findings.md)

---

## 0. Practical conclusion (corrected 2026-10-17 from Control v151)

> **Correction.** The statements of the July 2026 live tests ("shell silent, output only on the UART", "command table not in the binary") do not hold for v151:
>
> - The shell output **does** go over the TCP socket (`Protocol_PutChar` → `Shell_CH395SendData`, socket 1) as soon as a client is connected to port 8091.
> - The shell is only served when `g_debugMode == 1` (EEPROM 0x900). In the default mode 0, `Task_rs485` runs Modbus RTU and nobody reads port 8091. That is why the tests were silent.
> - The command table is in the image (`shellCommandTable` @ 0x08062cc4, 81 entries). Commands are matched by **name string**, not by 4-byte patterns; the 4-byte entries are the 10 key bindings.
> - How to enable the shell (Modbus write `0x5100` to register 46000), the command list and the risks: [VNSD_Shell_Command_Reference_v151.md](VNSD_Shell_Command_Reference_v151.md).
>
> The rest of this document is the v149.2 analysis and is kept as written; sections 3, 4, 5 and 8 contain statements that the v151 reference supersedes (marked there).
>
> **Interface overview:**
>
> | Port | Protocol | Status | Conclusion |
> |------|-----------|--------|-------|
> | 502 | Modbus TCP | ✅ active | HA integration, fully usable |
> | 8091 | TCP CLI shell | served only in `g_debugMode` 1 | silent in mode 0; see the v151 reference |
> | 30000 | **UDP** JSON-RPC | ⛔ do not enable | permanently destroys Modbus registers |
> | BLE | Binary VenusPacket | ✅ active | `[0x73][len][0x23][cmd][payload][xor]` |

---

## 1. Overview: two separate command systems

The device has TWO completely independent command systems that must NOT be confused:

| System | Function | Protocol | Channel |
|--------|----------|-----------|-------|
| **Binary BLE protocol** | `BLE_Recv_Cmd_Dispatcher @ 0x08007F58` | `[0x73][len][0x23][cmd][payload][xor]` | BLE (Quectel FC41D, UUIDs ff01/ff02) |
| **Text CLI shell** | `BLE_GATT_DispatchCommandByte @ 0x0804C4B4` | 4-byte pattern, character by character | TCP port 8091 (output on UART!) |

This document deals exclusively with the **text CLI shell system**.

---

## 2. Data path: how bytes reach the shell

```
┌─────────────────────────────────────────────────────────────┐
│  INPUT SOURCES                                              │
│                                                             │
│  CH395 TCP Socket 1 (Port 8091)    BLE GATT (Quectel FC41D)│
│         ↓                                    ↓             │
│  CH395_ReadRecvBuf(socket=1)         Queue @ 0x200000F0    │
│         └────────────────┬───────────────────┘             │
│                          ↓                                  │
│         Network_ReceiveAndDispatchData @ 0x0804D4E8         │
│              (184 bytes, dual-path dispatcher)              │
│                          ↓  (byte by byte)                  │
│           Special handling: byte 0x03 → log_SetModeAndLevel(0) │
│                          ↓                                  │
│         BLE_GATT_DispatchCommandByte @ 0x0804C4B4           │
│              (232 bytes, 4-byte pattern matching)           │
│                          ↓                                  │
│           Command table @ 0x08060EF8 (78 entries)           │
│                          ↓                                  │
│              Command handler function(ctx)                  │
└─────────────────────────────────────────────────────────────┘
```

**Dual-path logic in `Network_ReceiveAndDispatchData`:**
```c
if (RAM[0x20000132] == 1) {
    // TCP path: read data from CH395 socket 1 (port 8091)
    CH395_ReadRecvBuf(1, buf, len);
    for each byte in buf:
        if (byte == 0x03) log_SetModeAndLevel(0);
        BLE_GATT_DispatchCommandByte(ctx, byte);
} else {
    // Queue path: read BLE/RS485 queue
    Modbus_Response_Builder(queue, buf, timeout=10000);
    if (buf[0] == 0x03) log_SetModeAndLevel(0);
    BLE_GATT_DispatchCommandByte(ctx, buf[0]);
}
```

**RAM addresses:**
- `0x20000132` — socket type selector (1 = TCP mode)
- `0x2000010C` — TCP queue handle
- `0x200000F0` — BLE/RS485 queue handle
- `0x20014A70` — CLI context structure (RAM, dynamically allocated)

---

## 3. Command table format (16 bytes per entry)

**Address:** 0x08060EF8 – 0x080613D8 (78 entries × 16 bytes = 1248 bytes)

```
Offset  Size   Meaning
+0x00   1      Flags/permissions
+0x01   1      Type byte  (low nibble == 9 → active command)
+0x02   2      Padding
+0x04   4      4-byte match pattern (big-endian, first byte = MSB)
+0x08   4      Pointer to handler function (called as handler(ctx))
+0x0C   4      Pointer to display name string (→ strings at 0x08055xxx)
```

**Recognized by decompiling `BLE_GATT_DispatchCommandByte`:**
```c
// Loop over all 78 entries (stride 0x10):
if ((entry[+1] & 0xF) == 9 &&                          // type check
    BLE_GATT_EntryFilter(ctx, entry) == 0 &&            // permission check
    (entry[+4] & accumulated_mask) == ctx->accumulator && // prefix matches
    (entry[+4] & byte_mask) == new_byte << shift) {     // new byte matches
    
    ctx->accumulator |= new_byte << shift;
    if (shift == 0 || entry[+4].next_byte == 0) {
        entry[+8]();  // call handler
        ctx->accumulator = 0;
    }
}
```

The pattern is accumulated **byte by byte from MSB to LSB**. 4 bytes yield a complete match.

**Important:** The command table itself resides in **flash sector 7 (0x08060000–0x0807FFFF)**, which is **not** contained in the distributed binary (`_app_1492_...bin`, 385 024 bytes). Only the display name strings at 0x08055xxx are present in the binary (recognizable by `referenceCount=0` — they are referenced exclusively from the missing table).

---

> **Superseded (v151):** the "4-byte patterns" below are the key-binding entries of the table; commands are matched by name. The table is contained in the v151 image. The handler assignments in §4 (e.g. `err_code`, `ext_info`) were guessed from strings; the correct ones are in the reference. See [VNSD_Shell_Command_Reference_v151.md](VNSD_Shell_Command_Reference_v151.md) §1.

## 4. Known commands (from string analysis)

The following strings are located at **0x08055xxx** in the binary with `referenceCount=0`. This is reliable proof that they are CLI command names — the pointers from the table at 0x08060EF8 are missing in the Ghidra model.

### 4.1 Debug & diagnostics

| Command name | Flash address | Handler function | Description |
|-------------|---------------|------------------|--------------|
| `err_code` | 0x080559CA | `Debug_PrintErrorAndEventLog(0)` @ 0x0804D2D4 | Error code table (20 entries × 14 bytes) |
| `get_log` | 0x08055853 | `Debug_PrintErrorAndEventLog(1)` @ 0x0804D2D4 | Event log (20 entries × 9 bytes) |
| `wifi_info` | 0x0805589E | `Debug_PrintWifiStatus` @ 0x0804D498 | WLAN signal strength + connection status |
| `show_modbus` | 0x08055834 | `Debug_PrintModbusAddress` @ 0x0804D7E4 | Print Modbus register addresses |
| `ext_info` | 0x080559E9 | `SSL_PrintCertInfo` @ 0x0804D21C | SSL/TLS certificate info |
| `g_ext_info` | 0x080559F8 | `SSL_PrintCertVersion` @ 0x0804D25C (medium-high) | Global extended info — located directly after the `ext_info` handler in the same block, prints `"SSL cert version: %d"` |
| `rtos_status` | 0x080554CC | — | FreeRTOS task status |
| `reset_reason` | 0x08055A03 | — | Last reset reason (watchdog/POR/etc.) |
| `energy_show` | 0x080559A5 | `Debug_PrintPowerStatistics` @ 0x08034974 (high) | Energy statistics — prints all/mon/day_charge_power + all/mon/day_discharge_power |
| `get_time` | 0x0805592C | — | Read RTC time |
| `rtc_show` | 0x080554BA | — | RTC detail info |
| `get_ota_log` | 0x08055C34 | — | OTA update log |
| `power_check` | 0x08055C0F | `PowerPercent_WriteCallback` @ 0x0804D204 | Power check |
| `debug_fc` | 0x08055343 | — | FC41D BLE module debug |
| `trace_debug` | 0x080557EA | — | Trace output |
| `mars_debug` | 0x08032BA8 | — | Marstek debug mode |

### 4.2 Configuration & control

| Command name | Flash address | Handler function | Description |
|-------------|---------------|------------------|--------------|
| `set_level` | 0x0805530C | `Log_SetModeAndLevelCallback` @ 0x0804D210 | Set log level |
| `api_port` | 0x080558F4 | `Config_SetLocalApiPort` @ 0x0804D6EC | Change local API port (writes EEPROM @ 0x372) |
| `api_set` | 0x080558B9 | — | API settings |
| `set_wifi` | 0x08055772 | — | Set WLAN configuration |
| `debug_mode` | 0x080555EA | `Inverter_SetWorkMode` @ 0x0804D76C | Debug/operating mode |
| `modbus` | 0x08055820 | — | Modbus control |
| `mac_ble` | 0x0805578C | — | Show BLE MAC address |
| `full_dischrg` | 0x08055B6C | — | Force full discharge |
| `change_soc` | 0x08055418 | — | Change battery SOC |
| `button_ctrl` | 0x08055A23 | — | Button mapping / button control |
| `key_state` | 0x08055AE8 | — | Show key state |
| `reset` | 0x0805542D | — | Reset device |
| `venus_rfd` | 0x08055645 | — | Venus RFD (Recovery/Factory Default?) |
| `get_ver` | 0x080553C1 | — | Firmware version |
| `set_old` | 0x080553E2 | — | Enable legacy mode |
| `get_old` | 0x080553FD | — | Read legacy value |

### 4.3 OTA updates

| Command name | Flash address | Description |
|-------------|---------------|--------------|
| `update_ems` | 0x080554E3 | OTA update: EMS control board |
| `update_vns` | 0x08055517 | OTA update: Venus D (this device) |
| `update_bms` | 0x0805552D | OTA update: BMS (battery management) |
| `update_mppt` | 0x08055543 | OTA update: MPPT/solar charge controller |

### 4.4 Production/manufacturing tests

| Command name | Flash address | Handler function | Description |
|-------------|---------------|-------------------|--------------|
| `check_periph` | 0x08055386 | `EEPROM_ReadWrite_Test` @ 0x08006FB4 (medium) | Peripheral self-test — tests only EEPROM (name does not fit perfectly, possibly part of a larger test sequence) |
| `xfmc_test` | 0x080553A6 | — | XFMC memory test |
| `flash_test` | 0x08055440 | `Flash_ReadWriteErase_SelfTest` @ 0x08034D08 (high) | Flash memory test — interactive write/read test with address parameter, `"Write and read OK"` |
| `vns_ate_test` | 0x08055948 | ATE production test (Automated Test Equipment) |
| `code_test` | 0x08055AFF | Code/logic test |

### 4.5 Terminal control (special keys)

These names are located at **0x08056790** and probably represent terminal special-key handlers:

| Name | Flash address |
|------|---------------|
| `right` | 0x08056790 |
| `backspace` | 0x0805679F |
| `delete` | 0x080567B3 |
| `enter` | 0x080567BA |
| `clear` | 0x080567E9 |

---

## 5. Access paths to the shell

### 5.1 TCP port 8091 (primary access)

**Setup:** `CH395_Init_TCPServer_Socket @ 0x08032C0C`

```c
CH395_SPI_CmdWaitReady(1);
socket_descriptor.protocol = 2;   // TCP
socket_descriptor.socket   = 1;   // CH395 socket 1
socket_descriptor.src_port = 0x1F9B;  // 8091
socket_descriptor.dst_port = 0x1F9B;  // 8091
socket_descriptor.mode     = 2;   // Server (listen)
CH395_Socket_Open_ByDescriptor(socket_descriptor);
```

**Connection:**
```
telnet <device-ip> 8091
nc <device-ip> 8091
```

**Particulars:**
- Activation: `g_debugMode` (RAM `0x20000132`, EEPROM 0x900) must be 1; the socket is read only then (v151 reference §1.1)
- Byte `0x03` (Ctrl+C/ETX) resets the log mode: `log_SetModeAndLevel(0)`
- Connection setup runs via `CH395_Recv_Buffer_Setup @ 0x080195A4`, which also calls `CLI_InitSession`

### 5.2 BLE GATT (second access path — very likely usable)

**Registration:** `CLI_InitSession @ 0x0804C708`, called from `CH395_Recv_Buffer_Setup`:

```c
void CLI_InitSession(ctx, param_2, param_3=0x200) {
    // ... initialize context ...
    ctx[0x16] = 0x08060EF8;          // pointer to command table
    ctx[0x17] = 78;                   // number of entries
    BLE_ServiceSlot_Register(ctx);    // ← register CLI as BLE service
    entry = BLE_GATT_FindEntryByName(ctx, "VNSD-0 v1492", ctx[0x16], 0);
    BLE_GATT_SelectEntry(ctx, entry);
}
```

**BLE hardware:** Quectel FC41D (UART interface, AT commands)  
**AT command patterns** (from IDA analysis of a related firmware):
- GATT server: `AT+QBLEGATTSSRV=...`
- Characteristic: `AT+QBLEGATTSCHAR=...`
- Notification: `AT+QBLEGATTSNTFY=ff02,...`

**Assessment:** The CLI is officially registered as a BLE service (`BLE_ServiceSlot_Register`). The queue path in `Network_ReceiveAndDispatchData` routes BLE bytes through exactly the same `BLE_GATT_DispatchCommandByte` call as the TCP path. The BLE characteristic UUID is probably `ff02` (from the GATT notification command pattern).

**Access steps (experimental, to be verified):**
1. Establish a BLE connection with the FC41D (device name: "VNSD-0 v1492" or similar)
2. Write characteristic `ff02` (service UUID: `ff00` or similar)
3. Bytes are enqueued into the BLE queue (`0x200000F0`)
4. The queue path in `Network_ReceiveAndDispatchData` processes them

### 5.3 RS485 / queue path (theoretical)

The queue path (`else` branch in `Network_ReceiveAndDispatchData`) reads from `queue @ 0x200000F0`. In principle this queue can also be filled via RS485 if the corresponding Modbus task passes bytes on. In practice this is unlikely for the shell (RS485 is reserved for Modbus RTU).

---

## 6. Known handler decompilations

### `Debug_PrintErrorAndEventLog @ 0x0804D2D4` (230 bytes)

```c
void Debug_PrintErrorAndEventLog(int param_1) {
    if (param_1 == 0) {
        // "err_code": 20 error entries (14 bytes each)
        printf("Error code info:\n");
        for (uint i = 0; i < 20; i++) {
            printf("err %d: %d %d %d %d %d %lld\n",
                   i, error_table[i].code, error_table[i].f2,
                   error_table[i].f3, error_table[i].f4, error_table[i].f5);
        }
    } else {
        // "get_log": 20 event entries (9 bytes each)
        printf("Event info:\n");
        for (uint i = 0; i < 20; i++) {
            printf("event %d: %d %d %d %d %d %d %d\n",
                   i, event_table[i*9], ...);
        }
    }
}
// Error table: DAT_0804d3d0, stride 14 bytes
// Event table: DAT_0804d400, stride  9 bytes
```

### `Debug_PrintWifiStatus @ 0x0804D498` (24 bytes)

```c
void Debug_PrintWifiStatus(void) {
    printf("wifi_strength: %d\n", wifi_strength_var);
    printf("wifi_connect_status: %d\n", wifi_connect_status_var);
}
```

### `Config_SetLocalApiPort @ 0x0804D6EC` (66 bytes)

```c
void Config_SetLocalApiPort(uint16_t new_port) {
    config_base[0x72] = new_port;             // write to RAM
    EEPROM_Write(0x372, &config_base[0x72], 2); // persist to EEPROM
    if (work_mode != 3 && work_mode != 4)
        work_mode = 5;                        // set restart mode
    printf("Set local api port: %d, read port: %d, mode: %d\n",
           new_port, config_base[0x72], work_mode);
}
```

### `SSL_PrintCertInfo @ 0x0804D21C` (34 bytes)

```c
undefined8 SSL_PrintCertInfo(void) {
    // Prints SSL certificate metadata (7 fields via printf)
    printf(format_string, cert_field[0], cert_field[1], ..., cert_field[6]);
}
```

---

## 7. CLI context structure (reconstructed)

Base: `param_1` in `CLI_InitSession` and `BLE_GATT_DispatchCommandByte`

```c
struct CLI_Context {
    // +0x00: uint32   - flags/status
    // +0x03: uint16   - field (init to 0)
    // +0x0E: uint16   - field (init to 0)
    // +0x10: uint32   - param_2 (buffer pointer)  [= param_1[4]]
    // +0x34: uint8    - flags (bit 0 set in init)  [param_1+0x18]
    // +0x38: uint32   - 4-byte accumulator for pattern comparison ← important
    // +0x52: uint16   - field (init to 0)
    // +0x54: uint16   - field (= param_3/6, e.g. 0x200/6 ≈ 85)
    // +0x58: ptr32    - pointer to command table (= 0x08060EF8)  [param_1[0x16]]
    // +0x5C: uint16   - number of entries (= 78)  [param_1[0x17]]
    // +0x3C - 0x53: 5× uint32 - buffer offsets (computed)
};
```

---

> **Update 2026-10-17 (v151):** items 1, 3 and 4 are resolved: commands are matched by name (no patterns needed), the table is in the v151 image, parameters are plain tokens after the name. Open points are listed in [VNSD_Shell_Command_Reference_v151.md](VNSD_Shell_Command_Reference_v151.md) §4.

## 8. Open questions / next steps

1. **4-byte command patterns**: The exact match values (entry+4) reside in flash sector 7 (0x08060EF8), which is not contained in the binary. Cannot be determined without a full flash dump (incl. sector 7). → Possible sources: OTA update package, full flash dump via JTAG/SWD.

2. **Verify BLE access**: Actually establish a BLE connection and write characteristic `ff02`, then check whether shell output comes back.

3. **Shell prompt**: Unknown whether the shell outputs a prompt/echo or is silent.

4. **Command format with parameters**: `Config_SetLocalApiPort` takes a parameter (`new_port`). How parameter passing works over the byte interface is not fully clear (parameter bytes possibly follow after the 4-byte pattern match).

5. **Missing handlers**: **2026-07-10 update** — via static Ghidra analysis (orphan function heuristic: `callerCount=0`, since the only caller is the missing table in sector 7) 4 further handlers were identified: `energy_show`→`Debug_PrintPowerStatistics` (high), `flash_test`→`Flash_ReadWriteErase_SelfTest` (high), `g_ext_info`→`SSL_PrintCertVersion` (medium-high), `check_periph`→`EEPROM_ReadWrite_Test` (medium, name does not fit perfectly). Refuted: `set_wifi` is **not** an orphan (`Quectel_WiFi_SetSTAInfo_NoSave` @ 0x08012068 has an active caller in the WiFi reconnect code, does not belong to the CLI). For the remaining ~26 commands (`rtos_status`, `reset_reason`, `get_time`, `rtc_show`, `get_ota_log`, `debug_fc`, `trace_debug`, `api_set`, `set_wifi`, `modbus`, `mac_ble`, `full_dischrg`, `change_soc`, `button_ctrl`, `key_state`, `reset`, `venus_rfd`, `get_ver`, `set_old`, `get_old`, `update_ems/_vns/_bms/_mppt`, `xfmc_test`, `vns_ate_test`, `code_test`) the mapping remained uncertain despite keyword/similarity search — **blocked, needs live test** (either pure wrappers around already wired functions or the code resides in the missing flash sector 7).

---

## 9. Cheat sheet: testing shell access

```bash
# Test TCP access (prerequisite: device on the network, IP known)
nc -v <device-ip> 8091

# With timeout and logging
nc -v -w 10 <device-ip> 8091 | xxd

# NMAP port check
nmap -p 8091 <device-ip>

# Ctrl+C reset: send byte 0x03
printf '\x03' | nc -w 1 <device-ip> 8091
```

---

*Status: v149.2 analysis, July 2026*  
*Analysis tool: Ghidra + manual BL scan*
