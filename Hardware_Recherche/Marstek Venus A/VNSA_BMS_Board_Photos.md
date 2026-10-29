# Venus A – BMS board photos

> 🇩🇪 Deutsche Version: [VNSA_BMS_Board_Fotos_de.md](VNSA_BMS_Board_Fotos_de.md)

43 photos of a Venus A battery pack and its connectors (EXIF/GPS metadata removed). The BMS board is **not potted**, so components and markings are readable,
unlike the Venus D (see [VNSD_Teardown_Photos.md](../Marstek%20Venus%20D/VNSD_Teardown_Photos.md)).
No MAC addresses or device IDs are visible. The one serial-number label that was visible (case sticker in photo 28) is blacked out.

| Files in this folder | Count | Content |
|---|---|---|
| `VNSA_BMS_Board_01…27_<content>.jpg` | 27 | BMS board, both sides |
| `VNSA_BMS_Board_28…38_<content>.jpg` | 11 | More close-ups (MCU, AFE, switch/LED connector J7, J1, TVS parts), contributed via GitHub issue #4 |
| `VNSA_Head_Unit_01_<content>.jpg` | 1 | CAN connector J14 on the head-unit board, issue #4 |
| `VNSA_Pack_Connector_01…04_<content>.jpg` | 4 | 12-pin housing connector between pack and head unit/expansion, issue #5 |

## Board identification

Silkscreen: `HAME` logo, `13S50AH_BMS_V1.1 venus A`, `2025.09.08`; PCB base material marking `E336650`, date code `2538`.
"13S" matches the 13 cell taps (B0…B13). In this document "MCU side" is the side with the MCU, connectors and the large screw terminals,
"power side" the opposite side with the pads, shunts, MOSFETs and fuse-like parts. Both sides are populated.

A public forum thread about an opened Venus A (photovoltaikforum.com, thread 265074) shows the same board type `13S50AH_BMS_V1.1 venus A` with the
silkscreen date `2025.08.22` on another pack, so at least two build dates of the V1.1 board exist (`2025.08.22`, `2025.09.08`).
The forum states that the pack board of the main unit and of the expansion battery is identical, but the firmware differs.

## Identified components

| Ref. | Part | Evidence | Photos |
|---|---|---|---|
| U3 | **Nations N32G455VEL7** (MCU, ~100-pin LQFP, lot code `TBE12014`) | Print read at full resolution: `Nation` + logo, `N32G455`, `VEL7`. Confirms the N32G45x family for the BMS (until now only derived from the firmware). The suffix was not decoded. The print is partly rubbed off in photos 29/30, so no further digits could be read. | 11, 12, 29, 30 |
| U5 | **Sinowealth `SH36735?4U`** (cell-monitoring AFE, 48-pin LQFP, date code `2507`) | Print (Sinowealth logo and name, `SH36735…`, `2507-…`) read at full resolution; the last digits stay ambiguous in all five photos. The firmware calls this AFE `SH36735` (SPI1, 13 cells). | 13–16, 31–33 |
| Y1 | Crystal 8.000 MHz | Print `8.000` | 11, 12, 30 |
| T1 | Small isolation transformer (gold-coloured housing, four pins per side) | Sits on the isolation line next to connector J1; print on top is mirrored/illegible | 21, 22, 36, 37 |
| U8, PC1 | Optocouplers, print `817B` | Print | 20, 24, 25 |
| U20, U22 | SOT-223 regulators | Footprint only, print not legible | 24, 25 |
| U2, U4, U21, U15, U16, U17, U11 | SOIC-8 chips | **Prints not legible**; type unknown (U21 sits between J7 and T1 with R213…R216 and C102…C105; print visible but not decodable in photos 36, 37). U2/U4 next to the MCU are possibly the I²C EEPROM mentioned in the firmware (unverified). | 11, 12, 21, 22, 36, 37 |
| – | Gold-coloured backup element with white tab next to the MCU | Presumably a supercap/backup cell (unverified) | 1, 11 |
| SW1 | Tactile push button | Function open, see [open points](#open-points) | 8, 10 |
| D1 | SMD LED (translucent body) next to the MCU, with `R14` (`102` = 1 kΩ) and `R9` (`103` = 10 kΩ) | Identified by its body only; its function (status LED vs. something else) is inferred | 29, 30 |
| R25… | Row of six small resistors between U5 and U3: five marked `101` (100 Ω), one marked `103` (10 kΩ) | Print; presumably series/pull resistors on the AFE–MCU lines (inferred, traces not followed) | 31, 32, 33 |
| Q1…Q17 | Small transistors in a row next to 13 resistors | Balancing network for 13 cells, with `NTC`/`B` taps | 17, 19, 20 |
| RS1…RS10 | Ten SMD shunt resistors in a row | Current measurement in the B- path (power side) | 23–27 |
| FU1…FU3 | Three larger black SMD parts in the P- path | Silkscreen "FU" (fuse designator); type not readable | 24, 26 |
| Q2, Q4, Q9, Q18, Q21 | Five MOSFET-like SMD parts in the power path | Footprints; count not exhaustive | 23–27 |
| TVS1, TVS10, TVS12, TVS16, TVS17 | Large black SMB/SMC-type diodes | Print partly visible (`5?GM`, `02`), type and voltage not determined; TVS12/TVS16 sit next to the cell-tap pad groups `B18`, `B38`, `B39`, TVS10 next to a `B+` pad | 32, 33, 34, 35, 37 |
| R212 | Large SMD resistor marked `100` (= 10 Ω), between a `B+` pad and D20/TVS10/C43/C58 | Print; presumably a series element in the supply path from `B+` (inferred) | 34, 37 |

## Connectors and pinouts (silkscreen)

| Connector | Pins (as printed) | Remark |
|---|---|---|
| **J5** | `B0, B2, B4, B6, B8, B10, B12`, two unlabelled, `GND, NTC2, GND, NTC4` | Cell taps (even cells) + two NTCs, 14-pin flat connector |
| **J14** (BMS board) | `B1, B3, B5, B7, B9, B11, B13`, unlabelled, `GND, NTC3, GND, NTC1` | Cell taps (odd cells) + two NTCs, 14-pin flat connector. Not to be confused with `J14` on the head-unit board (below). |
| **J7** | `LED, 3.3V, GND, ADD-IN` | 4 pins. **Power switch with LED:** the cable to this connector is labelled `ON/OFF` (photo 28), and the reporter states that the top 4-pin connector is the power-switch connector. The expansion battery leaves it unused (forum). `ADD-IN` is then presumably the switch/wake input (inferred). |
| **J1** | `GND, ADD-IN, IN, OUT, (2 unlabelled), H, L, AGND` | 9 pins. Above the isolation line (printed next to T1): `GND/ADD-IN/IN/OUT`; below it: `H`, `L`, `AGND`, i.e. a separate ground. The reporter states that the top four pins (`GND/ADD-IN/IN/OUT`) carry the communication to the main unit/top module. `H`/`L` are most likely the CAN pair (firmware: CAN1; confirmed on the head-unit side by `CAN_H`/`CAN_L`, see below). The meaning of `ADD-IN/IN/OUT` (pack addressing/wake chain?) is not verified. |
| **J2** | `VCC, DIO, CLK, GND` | 2.54 mm through-hole, 4 pins, **not populated** – SWD |
| **J3** | `VCC, TX, RX, GND` | 2.54 mm through-hole, 4 pins, **not populated** – UART |
| Power | `B+` ×12 pads (three groups), `B-` ×4 pads, `P-` ×8 pads, three larger terminals on the MCU side | Battery side (`B+`/`B-`) and output side (`P-`); the cables are labelled `B-`, `B+`, `PACK+` (photo 28) |
| Test points | `TP1…TP20` | Along the power-side edge, e.g. TP13/TP14 near the cell taps, TP15–TP17 near the NTC pads |

## Pack ↔ head unit wiring

**Head-unit board, connector `J14`** (photo "Head_Unit_01", board covered with potting compound): a 5-pin wafer connector. Silkscreen below it:
`CAN_H`, `CAN_L`, `Button_S`, `GND`. The reporter states that only the top two of the five pins are wired, `CAN_H` and `CAN_L`. This matches the forum
finding that head unit and battery communicate over a two-wire cable, and the firmware (BMS and Control talk CAN). `Button_S` is presumably the switch signal
that belongs to the `ON/OFF` switch (inferred); it is not wired on the photographed unit.

**Housing connector pack ↔ head unit/expansion** (photos "Pack_Connector_04" front, "01…03" rear): 12 contacts. Four large contacts are numbered 1–4.
The eight small contacts are numbered 5–12 in a triangular block: top row `9, 10, 11, 5`, middle row `8, 12, 6`, bottom `7`.

Reporter's pin assignment (their own words: "as far as I expect", not measured here):

| Contact | Function (reporter) |
|---|---|
| 1 | Earth (PE) |
| 2 | Minus |
| 3 | Plus |
| 4 | Not used |
| 5, 9, 10, 11 | Communication |
| 6, 7, 8, 12 | Not used |

Visible on the rear photos: heavy red and black conductors, a green/yellow conductor, and a bundle of thin wires (black, red, magenta) soldered to the small contacts.
Which thin wire goes to which small contact cannot be read from the photos. Open: how the four communication contacts map to `J1` (`GND/ADD-IN/IN/OUT`, `H/L/AGND`)
and `J7`; the reporter's count (four communication contacts) does not match the eight signal-type pins on `J1` plus `J7`.

## Serial console

- **Found:** the unpopulated 2×4 through-hole footprint **J3/J2** next to the MCU (photos 11, 12, 29, 30; visible from the other side on photos 23–25).
  Silkscreen: `VCC / TX / RX / GND` (J3) and `VCC / DIO / CLK / GND` (J2). The square pad is pin 1 of each column (`VCC`).
  Photo 30 shows empty plated holes with solder residue, no pins.
  The same convention (UART `3.3V/TX/RX/GND`, SWD `3.3V/DIO/CLK/GND`) was seen on the Venus A main board (see `../Teardown_Fotos/`).
- **Not determined:**
  - Which MCU pins J3 TX/RX are connected to. Following the traces from the photos is not possible with certainty.
    A continuity test between J3 and the MCU pins is needed. Candidate from the firmware: USART1 on PA9/PA10 (the only UART in the A BMS image).
  - The baud rate of a console. The firmware documents 9600 baud only for the RS485 on USART1.
  - Whether `VCC` is 3.3 V. Measure before connecting a USB-UART adapter (3.3 V logic only).
- Firmware context (BMS 115, letter-shell): user `BLUE`, empty password, about 39 commands; see
  [VNSA_BMS_FW_v115_Full_Analysis.md](../../BMS/VNSA_BMS_FW_v115_Full_Analysis.md) §4.3. The firmware notes also say that the shell runs on the RS485 debug path;
  whether J3 carries the same shell or a separate one is open.
- J2 `DIO/CLK` is most likely SWD for the N32G455 (to be confirmed by measurement).

## Open points

1. Types of the SOIC-8 chips (U2, U4, U11, U15, U16, U17, U21) and of the TVS diodes: sharp macro photos with oblique light needed (photos 36, 37 did not make U21 legible).
2. Pin assignment of J3/J2 to MCU pins (continuity test).
3. Function of `ADD-IN/IN/OUT` on J1 and of the two unlabelled J1 pins. Firmware candidate: in A 115 the BMS reads **PB13** as the pack-select acknowledge input,
   and Control drives a pack-encode line (PB10 → first BMS PB13, see [VNSD_Unclear_Registers_Resolution_2026-10-01.md](../../Modbus_RS485_TCP/VNSD_Unclear_Registers_Resolution_2026-10-01.md), register 45025).
   Whether PB13 is wired to `ADD-IN`, `IN` or the button is **not** shown by the photos (continuity test needed).
4. Meaning of `FU1…FU3` (type, rating).
5. Which hardware revision and firmware version the photographed units run (the board exists with at least the build dates `2025.08.22` and `2025.09.08`).
6. Function of SW1: reset button or user key. A forum user reports that pressing it during a BMS hang (SoC stuck at 1.4 %) had no visible effect; only a BMS update fixed it.
   In the D/E3 BMS firmware PB13 is the power key, in A 115 it is the pack-select acknowledge input; which net SW1 is on is not known.
7. Mapping of the 12 housing contacts (1–12) to `J1`/`J7` and to the head-unit connector; the reporter's list is an expectation, not a measurement.
