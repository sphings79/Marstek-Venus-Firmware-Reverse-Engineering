# VNSD-0 MPPT firmware v104 — findings from static analysis

> 🇩🇪 Deutsche Version: [VNSD_MPPT_Firmware_Findings_de.md](VNSD_MPPT_Firmware_Findings_de.md)

Static analysis only (disassembly and decompilation of the OTA image, TI C28x, loaded at word address `0x3E8000`; see [VNSD_MPPT_C28x_Image_Setup.md](VNSD_MPPT_C28x_Image_Setup.md)). Nothing here was verified on hardware. All 438 functions are named, but names carry a confidence level in the Ghidra function comment: 188 high, 250 medium, 0 low. Treat medium and low names as working labels, not as proof. All addresses are C28x **word** addresses.

## Program structure

| Item | Address | Note |
|---|---|---|
| Reset entry | `0x3E8000` | `LB 0x3F0BAE` |
| `c_int00` | `0x3F0BAE` | standard TI startup: disables the watchdog (`0x7029` = `0x68`), runs the copy table at `0x3F2B30`, calls `main` chain |
| Main background loop | `0x3EF4FA` | `Main_BackgroundLoop`; peripheral init sequence at `0x3EF4C9` |
| Periodic control step | `0x3E8348` | `Mppt_ControlStep_FourChannelTracker`, called from an interrupt handler (entry `0x3F05FB`), no other caller |
| Interrupt vector table | copied from `0x3F296A` (125 longs) to `0x0D06` by `0x3F07CE` | PIE enabled afterwards |
| Letter-shell console | code `0x3EB7BE`–`0x3EC3xx`, command table at `0x3F271A` | 8-word entries (attribute, name, handler, description) |
| Code island in the data tail | `0x3F2CA0`–`0x3F2D19` | four Q24 / float helpers called from real code |

The tail `0x3F0BE2`–`0x3F3FFF` otherwise holds strings (one 16-bit character per word), tables and the C runtime copy table.

## Hardware as seen from the code

- Four PWM channels: register blocks at `0x6800`, `0x6840`, `0x6880`, `0x68C0` (stride `0x40`), fixed-point (Q24) control per channel.
- ADC result registers `0x0B00`–`0x0B0B` are labelled by the firmware's own console output as `PV1_V`, `PV1_I`, … `PV4_V`, `PV4_I`, `CELL_V`, `CELL_I`, `VREF`, `PE_V`: four PV inputs with voltage and current, the battery-side cell voltage and current, a reference and a protective-earth voltage.
- An EEPROM sits on an I2C-like peripheral (`0x7900`–`0x7909`, `0x7921`, 400 kHz, slave address `0x50`, `0x51` above address `0xFFFF`). **Correction:** an earlier working note called `0x7900` a GPIO frame; the code behaves like I2C.
- A CAN controller with 32 mailboxes in an eCAN-style layout (`0x6000` / mailbox RAM `0x6100`). Mailboxes 0–15 transmit, 16–31 receive; the node address in RAM `0x8002` is part of the identifiers.
- Code outside the image is called: `0x3D7C80` (clock / init), `0x3D7E82`, `0x3D7E85` (temperature sensor calibration). This is consistent with factory-programmed code and calibration data in a chip region that is not part of the OTA image.
- Exact chip type: still unknown.

## Control and protection

- A four-channel tracker state machine on RAM `0x83C0` (0 wait/init, 1 input range check, 2 run, 3 shutdown/reset), 80-sample min/max windows and a 38-word context per channel at `0x8580`. The reference update looks like perturb-and-observe or incremental conductance; which of the two is not proven. A separate leaf function `Mppt_PerturbObserveStep` (`0x3EFA0F`) implements a perturb-and-observe step.
- Constant-power mode is flagged at `0x83CE` (the console prints `CP_CTL_MODE` or `MPPT_CTL_MODE_CONSTANT_STEP`).
- The protection routine `Prot_AdcScaleFilterAndCheckFaults` (`0x3E9333`) scales and IIR-filters the ADC values, compares them with thresholds and latches fault bits in `0x8748`/`0x8749`/`0x874A`/`0x874B` and `0x874C`/`0x874D`. The console fault texts name: PV1–4 overcurrent, cell overcurrent / over- and undervoltage / reflux, hardware cell overvoltage and reflux, chip temperature, PE overvoltage / warning, PV voltage over 58 V and PV diode current over 14.5 A.
- `Status_BuildTelemetryFromFilteredAdc` fills the CAN telemetry block `data_toCan` at `0x8400`–`0x8416`: PV voltage, current and power per channel, status, fault, warning, and daily / weekly / monthly / yearly energy.
- A heartbeat check (`0x3EAB55`) stops the PWM and forces a watchdog reset after 60 s without commands, unless RAM `0x8782` holds the magic `0xA5A5` ("master control", set by the console command `master`).
- Further `0xA5A5` values at `0x83C1` and `0x83C4` act as override keys inside the control step.

## CAN command handling (as named by the analysis)

`Can_ProcessRxCommandQueue` (`0x3EAB78`) reads an 8-word-entry receive ring at `0x8042`. Command bytes handled: `1`/`4` start/stop, `2` (control mode, power limit, voltage limit), `3` status, `5`, `6` (clear the energy slots in the EEPROM), `0xCB`, `0xCC` (date block, energy rollover), `0xCE`, `0xFA`–`0xFF` (serial number, offset calibration). It also sends a periodic frame with eight values. In shell mode the firmware ignores CAN commands.

## Console (letter-shell)

The image contains the open-source *letter-shell* command interpreter with users, commands, variables, keys, a password prompt and a five-entry history. The table at `0x3F271A` maps these commands to handlers (names as read from the table): `ctl_d`, `ctl_mode`, `bms_power`, `bms_cur`, `bms_vol`, `start`, `epwm_start`, `ss`, `offset_cal`, `offset_read`, `ele_read`, `ele_clear`, `clear_prt`, `master`, `help`, `clear`, and the display commands `dshow`, `adshow`, `adcshow`, `powershow`, `mpptshow`, `prtshow`, `canshow`, `snshow`, `vershow`. `vershow` prints the application version `104`.

`ctl_d` writes a Q24 duty value to the high-resolution duty registers of the four PWM blocks; a value of 10000 or more forces a trip. The I/O device of the console is named `uart` in the firmware; which pins and baud rate it uses is not established here. Whether the console is reachable on the shipped device was **not** examined.

## EEPROM contents (as seen from the code)

Version words `101` at `0x100` and `104` at `0x104`, a 20-character serial number at `0x109`, calibration offsets for PV and cell current, and five long values at `0x20`–`0x30` that are probably cumulative energy counters (not confirmed).

## C runtime in the image

TI runtime library functions are present and named by what they do: printf with `%f`/`%g`/`%a`, malloc / free / realloc / calloc on a small heap at `0x8A20`–`0x8B1E`, stdio (fopen, freopen, setvbuf, fputc), TI C I/O host functions, string and memory helpers, 32/64-bit division and float/double conversions. The 64-bit division helper at `0x3EF40A` returns the quotient (reviewed: equal operands give 1, a smaller dividend gives 0); the remainder entry is `0x3EF439`.

## Known weak spots

- No function is left at low confidence: the 55 low-confidence names of the first pass were re-reviewed once and raised to medium or high. The re-review was again done by reading code, not by testing on hardware. About 76 functions are identical 5-word stubs (`ASP`, `ESTOP0`, endless loop) in one run starting at `0x3EEFA8`; they look like unused trap/interrupt handlers and make up a sixth of the function count without carrying logic.
- The two-word instruction `0x763F 0x0080` occurs three times (at `0x3E80C0`, `0x3E9CF3`, `0x3EF4F9`). The Ghidra processor module did not know it, so it split the surrounding functions into pieces. It is `IACK #0x0080` (identified from context, **not confirmed against the TI manual**): each time it is followed by `RPT #3 || NOP`, which is the usual way to start a CLA task and wait four cycles, and the following code polls a CLA flag bit. A local patch adds the instruction to the module (see the setup document); the pieces were merged back into three functions: `Init_ResetControlStateAndTriggerCla` (`0x3E809F`), `Ctrl_DisableOutputsAndInitChannelParams` (`0x3E9CE6`) and `Main_InitSequenceAndBackgroundLoop` (`0x3EF4EF`). Starting CLA task 8 at these three points is an inference.
- The chip family is only a hint: register use looks like a TI F2806x/F2803x-class Piccolo (I2C at `0x7900`, GPIO bank B at `0x6F80`, ePWM at `0x6800`, a CLA program copied to `0x9000`), but the register names for clock, GPIO and PIE come from TI header knowledge of that family, not from this project, and are medium confidence.
- Q24 scale factors and the physical units of the PV and cell values were not derived.
- Names for RAM variables are limited to 106 locations with evidence from console strings or code; the rest of the RAM is unnamed.
