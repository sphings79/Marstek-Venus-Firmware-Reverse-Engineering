# Cloud unreachable: the HTTP watchdog in Control v150 vs v151

> 🇩🇪 Deutsche Version: [VNSD_Control_Cloud_Blocked_Watchdog_v150_v151_de.md](VNSD_Control_Cloud_Blocked_Watchdog_v150_v151_de.md)

**Date:** 2026-10-01 · **Firmware:** Control v150 and v151 (Venus D) · **Function:** `HTTP_ClientTask_Dispatcher`
(v150 `0x0801A3D0`, v151 `0x0801A4A4`) · **Status:** from the decompiled code, not measured on a device

**Trigger:** a user report (marstek_venus_modbus issue 300, Venus E 3.0 on V150) says that with the cloud blocked the
network chip is reset every 30 minutes, and that the control hangs. This note explains the code behind it and how
v151 changed it.

## 1. What the watchdog watches

- **Time base:** `Tick_Timer_Check_Elapsed` compares a 16-bit counter (`g_tick16b`). It is incremented in
  `Runtime_Energy_Counter_Tick` (called from `SysTick_Handler`) once per 1000 sub-ticks, together with
  `g_uptimeSeconds`. **One tick = 1 second.**
- **Backlog (`upload_cnt`):** it is `g_telemetryHistoryRecordCount`, the fill level of a 144-slot ring buffer
  (`Telemetry_History_Record_Push`, 0x35D bytes per record). One record is pushed per **5-minute TOU slot**
  (`HTTP_Economy_TOU_PeriodicHandler`). Successful uploads remove records (log line "Delete data, upload_cnt = …").
  With a dead cloud the counter therefore grows by 1 every 5 minutes (full after 12 h).
- **Test flag:** `g_oldStep` (burn-in test step, normally 0) disables the action while a factory/aging test runs.

## 2. v150

```
if (upload_cnt > 1 && Tick_Timer_Check_Elapsed(g_httpNoLinkRebootTick, 0x708 /*1800 s*/) && g_oldStep == 0) {
    CH395_Reset_And_Reinit(0);     // soft reset of the Ethernet chip, log "[HTTP]ch395 reset!!!!"
    g_httpNoLinkRebootTick = 0;    // timer re-armed
    g_httpClientJobState = 0;
}
```
The timer is armed on the first call with a backlog of 2 or more and is **not** reset when the backlog drops.
With the cloud blocked the chip is therefore reset about **every 30 minutes**. All Modbus TCP connections are cut
each time (the reset path contains no inverter action). This matches the report.

## 3. v151

```
if (g_httpWatchdogSuppressFlag == 1)            g_httpNoLinkRebootTick = 0;   // suppress
else if (upload_cnt < 5 || g_oldStep != 0)      g_httpNoLinkRebootTick = 0;   // reset the timer
else if (Tick_Timer_Check_Elapsed(g_httpNoLinkRebootTick, 0x3840 /*14400 s*/)) {
    g_httpClientJobState = 0;
    System_Reboot();                             // full MCU reset
}
```
- The condition must hold **without interruption for 4 hours** (any successful upload that brings the backlog below 5
  restarts the timer). With a dead cloud this is reached about 25 minutes (backlog 5) + 4 h after the cloud was lost.
- `g_httpWatchdogSuppressFlag` (`0x60025B1E`) is only **read** in the code that was searched (no writer found), so the
  suppression is effectively never active.
- `System_Reboot()` calls `Inverter_StopOutput()`, saves the runtime counters and a reboot timestamp, waits 50 ms and
  triggers an NVIC reset. **The inverter output is switched off during the reboot** — also in backup (EPS) operation.
- The watchdog does **not** look at whether the Modbus/LAN side works or whether the battery is delivering power.

## 4. Consequence

| | v150 | v151 |
|---|---|---|
| Cloud unreachable | CH395 soft reset every ~30 min | no chip reset, but **full reboot after ~4 h 25 min** (repeats) |
| Modbus TCP | connections dropped every 30 min | connections drop only with the reboot |
| Inverter | no inverter action in the reset path | output **off** during every reboot |
| Short cloud outage (< 25 min) | nothing | nothing |

For setups that block the cloud on purpose (local-only operation, power-sensitive backup use) v151 replaces a
frequent but harmless reset by a rare but disruptive reboot. A local answer for the HTTP uploads (for example an
offline endpoint that acknowledges the uploads) keeps the backlog below the thresholds in both versions; whether the
existing offline endpoint does so was not checked here.

## 5. Limits of this analysis

- Based on the decompiled code of v150/v151; no measurement on a device.
- The path analysed is the **CH395 (Ethernet) HTTP client**. A Wi-Fi-only device uploads through the Quectel module
  (`HTTP_Task_StateMachine_Handler`); that path has its own timeouts and was not analysed for a reboot trigger.
- A separate, older reboot exists in both versions: `Idle_Watchdog_AutoReboot` switches the inverter off and reboots
  after one hour of complete idle (no PV, no grid permit, inverter state < 2, zero powers, no network command activity).

## 6. Does the Marstek offline endpoint still keep the backlog empty on v151?

Checked by reading the endpoint's server code against the v151 upload path (not run against a v151 device):

- **Upload path unchanged:** `Cloud_UploadTelemetryData` posts to `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`
  (CH395 path). A record counts as sent when the HTTPS POST completes **and** `HTTP_ParseUploadResponseCode` returns 0. That
  parser is identical in v150 and v151 (`strstr("\"code\":")` + `atoi`; a missing `"code"` also counts as 0). The record is then
  removed and `upload_cnt` drops by one ("Delete data, upload_cnt = …").
- **The endpoint answers exactly that:** `POST /data-upload/v1/venus/*` is answered with `{"code":0,"message":"success","data":null}`.
  With the uploads acknowledged the backlog drains by one record per upload, stays far below 5, and the v151 reboot
  condition (backlog ≥ 5 for 4 h) cannot become true.
- **DNS:** the recommended setup rewrites the whole domains `marstekcloud.com` and `hamedata.com` including wildcards. That also
  covers the one host change in v151: `getDateInfo` moved from `<region>.hamedata.com` to `api-<region>.marstekcloud.com` (HTTP, port 80).
  The endpoint's time route matches `/getDateInfo` regardless of the host name.
- **Not covered:** the Wi-Fi (FC41D / Quectel) upload path. `Quectel_HTTP_POST_SendAndVerify` was rewritten in v151 (tick loop with
  ERROR/`+QIURC` detection and exact byte count instead of blind delays). Whether the endpoint's replies satisfy the new loop was
  **not** checked. The endpoint documentation (30-minute reset, 15 minutes on Wi-Fi) describes v150 only; for v151 the failure mode
  without a cloud is the reboot described above.
