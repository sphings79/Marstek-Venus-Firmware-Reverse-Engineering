# VNSD-0 MPPT firmware v104 — findings from static analysis

> 🇩🇪 Deutsche Version: [VNSD_MPPT_Firmware_Findings_de.md](VNSD_MPPT_Firmware_Findings_de.md)

Static analysis only (disassembly and decompilation of the OTA image, TI C28x, loaded at word address `0x3E8000`; see [VNSD_MPPT_C28x_Image_Setup.md](VNSD_MPPT_C28x_Image_Setup.md)). Nothing here was verified on hardware. All 454 functions are named; every name carries a confidence level in the Ghidra function comment: 408 high, 46 medium, 0 low. Treat medium names as working labels, not as proof. All addresses are C28x **word** addresses.

Register layouts of the F2803x peripherals (ADC, clock control, PLL, GPIO, CLA registers) were in part taken from knowledge of the TI header files, not from a retrieved reference manual (the F2803x technical reference manual could not be fetched as text). Names that rest on this are flagged in the summary below.

## Chip

| Item | Result | Evidence |
|---|---|---|
| Family | **TI TMS320F2803x** (Piccolo), most likely **F28035** | PIE table, LIN-A block, CLA, no FPU instructions |
| Not an F2806x | the earlier working assumption was wrong | the console UART at `0x6C00` is the LIN-A block, which F2806x does not have; only 7 ePWM interrupt slots are populated; no FPU (LVF/LUF) use |
| Why F28035 | CLA present and flash of at least 48 K words | F28033 has 32 K flash, F28034 has no CLA |
| Package / Q1 variant | **open** | needs `PARTID` (`0x3D7E80`, never read by the firmware) or the pin count |

Flash start `0x3E8000` and the CLA RAM at `0x9000` do not discriminate between F2803x and F2806x. The F2805x family was not checked.

## Program structure

| Item | Address | Note |
|---|---|---|
| Reset entry | `0x3E8000` | `LB 0x3F0BAE` |
| `c_int00` | `0x3F0BAE` | standard TI startup, writes `WDCR = 0x68` (watchdog off), runs the copy table at `0x3F2B30`, calls the `main` chain |
| Flash set-up routine | `0x3E8080` | `Init_FlashWaitStatesAndPipelineRamFunc`: writes the flash registers `0x0A80`–`0x0A87`; copied to RAM `0x400` and executed there by `Init_StartupCopyRamAndPieSetup` (`0x3EF49B`) |
| RAM-function delay loop | `0x3E809B` | flash load image of a delay loop that runs from RAM; never called at its flash address |
| Peripheral init | `0x3EF4C9` | `Init_ConfigureAllPeripherals` |
| Main and background loop | `0x3EF4EF` | `Main_InitSequenceAndBackgroundLoop` |
| Periodic control step | `0x3E8348` | `Mppt_ControlStep_FourChannelTracker`, called from the Timer 1 interrupt |
| Letter-shell console | code `0x3EB7BE`–`0x3EC3xx`, command table `0x3F271A` | 8-word entries (attribute, name, handler, description) |
| Code island in the data tail | `0x3F2CA0`–`0x3F2D19` | four Q24 / float helpers called from real code |
| Initial PIE vector table | `0x3F296A` | 125 long entries, copied to RAM `0x0D06` by `Init_CopyPieVectorTable` (`0x3F07CE`) |

The tail `0x3F0BE2`–`0x3F3FFF` otherwise holds strings (one 16-bit character per word), tables, the C runtime copy table and padding (see "Image layout").

## Interrupts

The initial vector table contains only default trap stubs (`ASP`, `ESTOP0`, endless loop); the real handlers are written into the PIE table in RAM at run time. 110 of the 112 PIE slots match the F2803x name table (unique stubs only where the datasheet names a peripheral); ADCINT1/2 (INT10.1/10.2 in the datasheet's numbering) are the exception. Layout sources: TI F2803x datasheet SPRS584Q table 7-19, F2806x datasheet SPRS698J table 8-17 (CPU-level order from a TI F2806x header file).

| Vector | Source | Handler | Role |
|---|---|---|---|
| `0x0D1A` | INT13, CPU Timer 1 | `0x3F05FB` | services the watchdog (`Wdt_ServiceKeys_55_AA`, keys `0x55` then `0xAA`) and runs the control step `0x3E8348`; the timer tick is 1 ms (inferred from the timer arguments) |
| `0x0D1C` | INT14, CPU Timer 2 | `0x3F0A66` | 32-bit tick counter at `0x88C6` and a flag |
| `0x0D42` | INT1.2, ADCINT2 | `0x3F05DF` | calls `Prot_AdcScaleFilterAndCheckFaults`, acknowledges the PIE |
| `0x0DC8` | INT9.5, ECAN0INTA | `0x3ED7E1` | re-initialises the CAN controller on an error flag or clears the receive-pending bits; does not queue data |
| `0x0DCA` | INT9.6, ECAN1INTA | `0x3ED814` | receive only: copies each pending mailbox (31 down to 16) into the 20-entry ring at `0x8042` |

`IER` ends up as `0x3101`. The CLA completion interrupts (PIE group 11) are never enabled; the eight PIEACK stubs at `0x3F043A`–`0x3F0479` are unused. About 76 identical 5-word trap stubs are named after their vector. Any unexpected interrupt therefore ends in an endless loop; the watchdog is then no longer serviced and resets the chip.

## Watchdog

`WDCR` (`0x7029`): `0x68` = disabled, `0x2D` = enabled (check bits 101, prescaler 5), `0x00` = invalid check bits, forces an immediate reset. The watchdog is disabled at reset entry and around two calibration routines and enabled at `0x3EF528` after the initialisation. The keys are written by `Wdt_ServiceKeys_55_AA` (`0x3EF13E`), called from the Timer 1 interrupt and a few init paths. A heartbeat routine (`Comm_CheckHeartbeatTimeout_StopReset`) counts 60 calls without CAN activity, then stops the PWM and writes `WDCR = 0`, which resets the chip; the magic value `0xA5A5` in RAM `0x8782` ("master control", set by the console command `master`) suppresses this. At start-up the firmware reads `WDCR` and prints "Reset by Watchdog" if bit 7 is set. The bit field layout was not confirmed against the F2803x manual; the timeout is unknown because the oscillator frequency is not proven.

## Hardware as seen from the code

- Four PWM channels: register blocks at `0x6800`, `0x6840`, `0x6880`, `0x68C0` (stride `0x40`). Period `TBPRD = 0x4AF` (1200 counts), dead band 30. The duty is written as `CMPA = D × 1199` plus a high-resolution part (`CMPAHR`, scale 111, probably the HRPWM MEP steps per clock). Fixed-point (Q24) control per channel.
- ADC result registers `0x0B00`–`0x0B0B` are labelled by the firmware's own console output as `PV1_V`, `PV1_I`, … `PV4_V`, `PV4_I`, `CELL_V`, `CELL_I`, `VREF`, `PE_V`: four PV inputs, battery-side cell voltage and current, a reference and a protective-earth voltage. An offset self-calibration (`Adc_OffsetSelfCal`) and a 256-sample averaging sweep are present.
- An EEPROM sits on the I2C block (`0x7900`, 400 kHz, slave address `0x50`, `0x51` above address `0xFFFF`).
- A CAN controller with 32 mailboxes in the eCAN layout (`0x6000`, mailbox RAM `0x6100`). Mailboxes 0–15 transmit, 16–31 receive; the node address in RAM `0x8002` is part of the identifiers. The bit rate is not established.
- The console UART is the LIN-A block at `0x6C00` used as an SCI (baud rate not established).
- The CLA (control law accelerator) is initialised: program copy to RAM `0x9000`, vector `MVECT8 = 0xE96` (task 8 code at `0x9E96`), `MIER` bit 7 set. `IACK #0x80` starts CLA task 8 at three places; the main sequence then waits on `MIRUN` bit 7. The task code itself is not part of the analysed program space.
- Code outside the image is called: `0x3D7C80` (clock calibration), `0x3D7E82` and `0x3D7E85` (temperature sensor calibration). This is the TI-programmed OTP area of the device.
- Time base (inferred, medium): 60 MHz system clock, ADC/PWM interrupt at 50 kHz, tracker at 1 kHz.

## Scaling and units

All values are fixed-point Q24 where **1.0 equals ADC full scale** (the ADC interrupt converts each result with `raw << 12`, so 4096 counts = 3.3 V = `0x1000000`). Float constants are stored with the high word first (verified with round values such as 60.0, 1000.0, 25000.0).

| Quantity | Full scale (derived) |
|---|---|
| PV voltage | about 68.04 V |
| PV current | 44.0 A |
| PV power | about 2993.81 W |
| Cell voltage | about 73.52 V |
| Cell power | about 7889.17 W |
| PE voltage | about 132.7 V |

The cell current is `(VREF − I) / 0.0205 A`, i.e. a 20.5 mV/A sensor. The implied voltage dividers are about 20.6:1 (PV), 22.3:1 (cell), 40.2:1 (PE) for a 3.3 V reference. The real resistor values are not in the code. The IIR filters have the coefficients `0x202B7F` (fast) and `0x83C` (slow) divided by 2^24, which correspond to about 1 kHz and 1 Hz with a 20 µs sample period.

Telemetry is reported in steps of 0.1 V, 0.1 A and 0.1 W. In the telemetry block (`data_toCan`, `0x8400`) the channel order is **reversed**: index 0 holds PV4 and index 3 holds PV1; the status run bits and the "below 5 W is 0" rule use the normal order. Whether this is intentional is unknown.

## Control and protection

- The tracker is **incremental conductance** (it compares `I/V` with `−dI/dV` with a fixed step), a four-channel state machine on RAM `0x83C0` (0 wait/init, 1 input range check, 2 run, 3 shutdown/reset) with a 38-word context per channel at `0x8580`. The power stage looks like a boost converter. `Mppt_IncCondTrackerStep` (`0x3EFA0F`) is a second incremental-conductance implementation without a caller (dead code); an earlier working name called it perturb-and-observe, which was wrong.
- PV start window 16 V to 56 V, stop below 13 V or above 60 V.
- Constant-power mode is flagged at `0x83CE`. Per-channel power cap 1041.67 W, total 4166.7 W, battery cap 4000 W.
- `Prot_AdcScaleFilterAndCheckFaults` (`0x3E9333`) scales and IIR-filters the ADC values, compares them with thresholds and latches fault bits in `0x8748`–`0x874D`. Decoded trip points (from the constants; round values confirm the scale): PV overvoltage 63.0 V (debounce of 25000 samples), PV current 32.0 A (debounced) and 40.0 A, current offset limit 1.0 A, cell overvoltage 60 V, undervoltage 15 V, overcurrent 120 A, reflux −30 A, PE trip 60 V, chip temperature raw limit corresponds to 95 °C.
- **Text and code disagree:** the console fault texts say "PV voltage over 58 V", "diode current over 14.5 A" and "temperature over 90 °C"; the decoded trip points are 63.0 V, 32.0 A and 95 °C. The messages may be stale.
- `Status_BuildTelemetryFromFilteredAdc` (`0x3E98E8`) fills `data_toCan` (`0x8400`–`0x8420`): PV voltage, current and power per channel, status, fault, warning, energy slots (daily, weekly, monthly, yearly).
- The heartbeat check described under "Watchdog" is really a timeout on CAN silence.

## CAN

`Can_ProcessRxCommandQueue` (`0x3EAB78`) reads the 20-entry receive ring at `0x8042` (8 words per entry; write index `0x80E3` from the ECAN1 interrupt, read index `0x80E2`). Command bytes handled: `1`/`4` start/stop, `2` (control mode, power limit, voltage limit), `3` status (answered with eight frames, 56 bytes: PV1..4 V/I/P, state, error, temperature, warning, energies, battery/base/PE voltage), `5`, `6` (clear the energy slots in the EEPROM), `0xCB` (version request), `0xCC` (date block, energy rollover), `0xCE` (enter loader, see below), `0xFA`–`0xFF` (serial number, offset calibration). In shell mode the firmware ignores CAN commands. The only unsolicited frame is an 8-byte fault snapshot sent every 100 ms while `0x83D1` is non-zero (whole volts of PV1..4, then whole amps of PV1..4). The identifiers are 29-bit with the command byte, address nibbles and a 5-bit frame index; the nibble roles are not bit-exact.

## Firmware update path (as seen from the Control firmware)

The Control firmware sends the MPPT image over **CAN** with command byte `0xCE` (identifiers `0x000121CE`, `0x010121CE`, `0x020121CE`, 8 data bytes per frame): sub-code 0 announces the total length and a CRC16 of the whole image, sub-code 1 carries data blocks of 1024 bytes (96 blocks) each followed by a block CRC16 and a sequence number, sub-code 2 reports the block status. The MPPT answers 1 (next block), 2 (repeat block) or 3 (restart); more than 25 retries is a failure. Entry points on the Control are BLE (target type 2, staged at external flash `0x100000`), the cloud, and the console YModem command `update_mppt`. The image is sent raw, including its 32-byte footer.

The MPPT application contains **no loader**: its own `0xCE` handler resets the PWM state, sets EEPROM byte 0 to 1 and forces a watchdog reset without replying; the normal start clears that byte again. It has no erase or program code and never sends `0xCE`, although the Control expects `0xCE` answers. The loader that receives the image therefore lives outside the analysed program space (plausibly flash below the image or the flash end at `0x3F4000`–`0x3F7FFF`); its CAN bit rate, ID decoding, footer or CRC check and start procedure are unknown. The Control checks no minimum MPPT version; the version arrives through a periodic `0xCB` poll and is only displayed.

## Image layout

| Range | Content |
|---|---|
| `0x3E8000`–`0x3F3E9D` | code, strings and tables |
| `0x3F3E9E`–`0x3F3FEF` | padding `0xFFFF` (338 words) |
| `0x3F3FF0`–`0x3F3FFF` | 16-word footer: `VNSD`, zeros, `0xFFFF` padding, `0x7CE8`, **`0x2222`** |

`0x2222` is the device mask of the MPPT in the Control's update table (BMS and Micro use `0x3333` and `0x4444`); the Control checks it when validating an update. `0x7CE8` is not read by the Control or the MPPT application, and it matches no tested checksum (CRC-16 variants, byte and word sums, CRC32 over the whole file, sub-ranges and all prefixes). On the BLE path the Control overwrites the mask bytes of the last chunk with `0xFFFF` before the CRC and the flash write, so an image staged and forwarded over BLE ends in `7CE8 FFFF`; why is unknown.

## Console (letter-shell)

The image contains the open-source *letter-shell* command interpreter with users, commands, variables, keys, a password prompt and a five-entry history. The table at `0x3F271A` maps these commands to handlers: `ctl_d`, `ctl_mode`, `bms_power`, `bms_cur`, `bms_vol`, `start`, `epwm_start`, `ss`, `offset_cal`, `offset_read`, `ele_read`, `ele_clear`, `clear_prt`, `master`, `help`, `clear`, and the display commands `dshow`, `adshow`, `adcshow`, `powershow`, `mpptshow`, `prtshow`, `canshow`, `snshow`, `vershow`. `vershow` prints the application version `104`. `ctl_d` writes a Q24 duty value to the high-resolution duty registers of the four PWM blocks; a value of 10000 or more forces a trip. The console device is registered as `uart` in the TI C runtime. Whether the console is reachable on the shipped device was **not** examined.

## EEPROM contents (as seen from the code)

Version words `101` at `0x100` and `104` at `0x104`, a 20-character serial number at `0x109`, five long values at `0x20`–`0x30` that are the current-offset calibration values (PV1..4 and cell), and slots at `0x10`–`0x1C` that are the cumulative energy totals. The meaning of the date-block fields (year, month, day) is not decoded.

## C runtime in the image

TI runtime library functions are present and named by what they do: printf with `%e`/`%f`/`%g`, malloc / free / realloc / calloc on a small heap at `0x8A20`–`0x8B1E`, stdio, the TI C I/O host functions, string and memory helpers, 16/32/64-bit division and float/double conversions. The 64-bit division helper at `0x3EF40A` returns the quotient; the remainder entry is `0x3EF439`.

## Known weak spots

- Names are static-analysis labels. 46 functions are at medium confidence: the GPIO20/21 and GPIO44 routines, the comparator routines (register layout not verified), the password and user parts of the shell, several printf helpers, the EEPROM date-block reader/writer and the heartbeat pair.
- Peripheral register names for ADC, clock control, PLL, GPIO and the CLA registers (`MIER` at `0x1425`, `MIRUN` at `0x1426`) come from knowledge of the TI headers for this family and were not checked against a retrieved manual.
- The two-word instruction `0x763F 0x0080` occurs three times (at `0x3E80C0`, `0x3E9CF3`, `0x3EF4F9`). The Ghidra processor module did not know it and split the surrounding functions. It is `IACK #0x0080`; the encoding (`0x763F` plus a 16-bit immediate) is confirmed in the TI instruction set reference (SPRU430F), and the task-bit mapping comes from a TI CLA FAQ. A local patch adds the instruction to the module (see the setup document).
- 16 functions (interrupt handlers and uncalled routines) were missing from the first function list because nothing calls them; they were added later. Any code that is reached only indirectly could still hide the same way; the check "no decoded instruction outside a function" is clean now.
- Hardware divider and shunt values, the CAN bit rate, the energy counter unit (about 10.4 Wh per count if the tracker runs at exactly 1 kHz), the ADC start-of-conversion trigger and the watchdog timeout are not established.
- RAM variable names exist for 188 locations with evidence from console strings or code; the rest of the RAM is unnamed. Structure types (`ChanCtx`, `DataToCan`, `FaultSnapshot`, `CanFrameEntry`, `TimerDrv`, `AdcPipeline`) are defined in the Ghidra project; their field meanings are medium or low confidence in places.
