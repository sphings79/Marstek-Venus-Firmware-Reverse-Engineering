# Cleanup of shifted scans (TID fault), 2026-08-22

> 🇩🇪 Deutsche Version: [BEREINIGUNG_TID_2026-08-22_de.md](BEREINIGUNG_TID_2026-08-22_de.md)

Scan cycles were removed in which values ended up in the wrong register because of a
misassigned Modbus response.

**The originals are kept unchanged in `_backup_vor_tid_bereinigung_20260822/`.**

## Cause

`scan_continuous.py` (local, not in the repo), `scan_known_registers.py` (local, not in the repo) and `scan_powercycle.py` (local, not in the repo) set a transaction ID in the
MBAP header but did not check it on receipt. A late
response to request N was therefore read as the response to N+1. All three scripts have been
fixed since 2026-08-22; `scan_registers.py` (local, not in the repo) was already correct.

## Detection and decision

A hit requires: carrier register stable (mode share >= 90 %, at least 6 measurements),
the conspicuous value occurs there at most 2x in the log, and another stable
register loses exactly this value in the same cycle.

Hits count as **hard** if the foreign value is far enough from the carrier's normal value
(distance > max(5, 25 % of the normal value)) — a real frame offset
produces a completely foreign value. Otherwise **weak**: that can also be measurement noise
or a real state change.

**All hit cycles were removed in logs that contain at least one hard hit** —
including the weak ones. Reason: a desynchronization spans
several consecutive requests; once it is proven for a log, its
weak hits are no longer trustworthy either.

Logs **without** a hard hit remain unchanged. Only
`discharge_w_steps1.csv` (local, raw data) is affected: there pack 1 switches its MOSFETs from 3 to 0 and pack 2
simultaneously from 0 to 3, two cycles 30 s apart — a real
state change, not a shift. The same applies to a swap of adjacent cell temperatures
(233/234).

## Validation of the method

`watch_update.csv` (local, raw data) (10,249 cycles, recorded with TID check) is clean,
`watch_update_vor_tid_fix.csv` (local, raw data) from the same measurement series is not. The hits also occur
in *consecutive* cycles — the expected behavior of a
desynchronization that only locks back in after a timeout.

## discharge_under_dod.csv (local, raw data)

- Format: wide, 548 cycles, of which **2 removed** (2 columns)
- Hard hits in: 10:14:04, 10:15:23

| Cycle | Register | wrong value | belongs to | otherwise shows | hard |
|---|---|---|---|---|---|
| 10:14:04 | 34007 | 7 | 34003 | 0 | yes |
| 10:14:04 | 34410 | 116 | 37012 | 0 | yes |
| 10:14:04 | 34502 | 116 | 37012 | 0 | yes |
| 10:14:04 | 34503 | 45 | 34303 | 0 | yes |
| 10:14:04 | 34510 | 116 | 37012 | 0 | yes |
| 10:14:04 | 37000 | 0 | 44002 | 1 | no |
| 10:14:04 | 37002 | 1 | 41100 | 0 | no |
| 10:14:04 | 37003 | 1 | 41100 | 0 | no |
| 10:14:04 | 37012 | 0 | 44002 | 116 | yes |
| 10:14:04 | 37015 | 116 | 37012 | 0 | yes |
| 10:14:04 | 41100 | 0 | 44002 | 1 | no |
| 10:14:04 | 43100 | 1 | 41100 | 127 | yes |
| 10:14:04 | 44002 | 127 | 43100 | 0 | yes |
| 10:15:23 | 30200 | 0 | 41200 | 147 | yes |

## entladen_lang.csv (local, raw data)

- Format: long, 669 cycles, of which **6 removed** (54 rows)
- Hard hits in: 14:16:34
- Additionally removed (weak hits only): 14:16:33, 14:16:35, 14:24:17, 14:42:15, 14:42:16

| Cycle | Register | wrong value | belongs to | otherwise shows | hard |
|---|---|---|---|---|---|
| 14:16:33 | 37000 | 0 | 36000 | 1 | no |
| 14:16:34 | 37009 | 1 | 41100 | 0 | no |
| 14:16:34 | 37012 | 0 | 38003 | 118 | yes |
| 14:16:34 | 38003 | 118 | 37012 | 0 | yes |
| 14:16:34 | 41100 | 0 | 38003 | 1 | no |
| 14:16:35 | 42010 | 1 | 43000 | 2 | no |
| 14:16:35 | 42020 | 2 | 42010 | 0 | no |
| 14:16:35 | 43000 | 0 | 43101 | 1 | no |
| 14:16:35 | 43101 | 1 | 43000 | 0 | no |
| 14:24:17 | 42010 | 1 | 43000 | 2 | no |
| 14:24:17 | 42020 | 2 | 42010 | 0 | no |
| 14:24:17 | 43000 | 0 | 43101 | 1 | no |
| 14:24:17 | 43101 | 1 | 43000 | 0 | no |
| 14:42:15 | 30212 | 0 | 30110 | 5 | no |
| 14:42:16 | 30300 | 0 | 32102 | 1 | no |
| 14:42:16 | 30301 | 0 | 32102 | 1 | no |
| 14:42:16 | 32102 | 5 | 31003 | 0 | no |

## laden_lang.csv (local, raw data)

- Format: long, 124 cycles, of which **3 removed** (27 rows)
- Hard hits in: 15:44:07, 15:44:09
- Additionally removed (weak hits only): 15:44:06

| Cycle | Register | wrong value | belongs to | otherwise shows | hard |
|---|---|---|---|---|---|
| 15:44:06 | 30212 | 0 | 30110 | 5 | no |
| 15:44:07 | 30300 | 0 | 32102 | 1 | no |
| 15:44:07 | 30301 | 0 | 32102 | 1 | no |
| 15:44:07 | 32101 | 0 | 32102 | 42 | yes |
| 15:44:07 | 32102 | 5 | 31003 | 0 | no |
| 15:44:09 | 34502 | 0 | 36000 | 601 | yes |
| 15:44:09 | 36000 | 601 | 34502 | 0 | yes |

## watch_update_vor_tid_fix.csv

- Format: long, 124 cycles, of which **5 removed** (46 rows)
- Hard hits in: 2026-08-21 16:14:36, 2026-08-21 16:22:28, 2026-08-21 16:22:29, 2026-08-21 16:22:30
- Additionally removed (weak hits only): 2026-08-21 16:14:39

| Cycle | Register | wrong value | belongs to | otherwise shows | hard |
|---|---|---|---|---|---|
| 2026-08-21 16:14:36 | 34000 | 117 | 30204 | 5256 | yes |
| 2026-08-21 16:14:36 | 34002 | 12848 | 30350 | 477 | yes |
| 2026-08-21 16:14:39 | 34503 | 62 | 34403 | 56 | no |
| 2026-08-21 16:22:28 | 34210 | 0 | 34308 | 118 | yes |
| 2026-08-21 16:22:28 | 34310 | 0 | 34308 | 118 | yes |
| 2026-08-21 16:22:29 | 34407 | 695 | 34502 | 0 | yes |
| 2026-08-21 16:22:29 | 34410 | 0 | 34508 | 118 | yes |
| 2026-08-21 16:22:29 | 34507 | 695 | 34502 | 0 | yes |
| 2026-08-21 16:22:30 | 34510 | 0 | 37023 | 118 | yes |

