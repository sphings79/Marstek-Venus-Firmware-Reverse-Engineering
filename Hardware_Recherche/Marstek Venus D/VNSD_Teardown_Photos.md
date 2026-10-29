# Venus D – internal photos (teardown)

> 🇩🇪 Deutsche Version: [VNSD_Teardown_Fotos_de.md](VNSD_Teardown_Fotos_de.md)

Photos of an opened Venus D (lid removed). EXIF/GPS metadata was removed and the rotation baked into the pixels.
No serial numbers, MAC addresses or labels with device IDs are legible in any photo.

## Main finding: the electronics are potted

The whole power board is covered with a black/grey potting compound. Components are only identifiable where they
protrude from it or where it was wiped off: connectors, relays, some electrolytic capacitors, cable labels.
Markings on ICs (MCU, MOSFETs, drivers) are **not** legible, so the MCU cannot be identified from photos.
Removing the compound would destroy the board; this was not attempted.

## What is visible

| Photo | Content |
|---|---|
| `VNSD_Teardown_10_potted_pcb_overview_wifi.jpg`, `VNSD_Teardown_11_potted_pcb_overview_battery.jpg`, `VNSD_Teardown_01_potted_pcb_rj45_corner.jpg` | Top-down overviews. Bottom edge: LED strip and the radio module with print `YDCFH0CA2AA_WIFI_X1`. Top right: two RJ45 sockets with ferrite sleeves and the J20/J26 board connectors. |
| `VNSD_Teardown_08_potted_pcb_angle_battery.jpg`, `VNSD_Teardown_09_potted_pcb_angle_rear.jpg` | Angled views across the potted board (magnetics, large black electrolytic capacitors, wiring from the PV side). |
| `VNSD_Teardown_02_battery_terminals_side.jpg`, `VNSD_Teardown_07_battery_terminals_button.jpg` | Battery terminals labelled `B+` / `B-` (two screw lugs, red/black heavy-gauge leads). Behind them two rows of 5 electrolytic capacitors (battery-side DC link, presumably). Bottom left: an illuminated pushbutton with blue ring, wired with 4 wires (red, yellow, blue, black). |
| `VNSD_Teardown_03_pv_input_mc4.jpg` | PV input: two MC4 connectors, 4 red + 4 black leads (cable marking `600V`, 10 AWG) to a terminal strip and four ferrite-sleeved leads; further ferrite chokes below. |
| `VNSD_Teardown_05_rs485_connector_j26.jpg` | Board connector **J26** with silkscreen `GND / 485B / 485A / 5V` (RS485 plus 5 V). Next to it connector J20. Both are fed by cables running to the two RJ45 sockets. |
| `VNSD_Teardown_04_comm_cable_label.jpg`, `VNSD_Teardown_15_cable_label_handwritten.jpg` | Cables with a sleeve printed `通讯线` ("communication line"), 1 red + 3 black wires; the second photo shows a handwritten sleeve (illegible). |
| `VNSD_Teardown_06_relays_hf140ff_hf115f.jpg` | Two Hongfa **HF140FF-G** relays (012-2HSWTF, 16 A 250 VAC) and a Hongfa HF115F. |
| `VNSD_Teardown_12_relay_hf118f.jpg` | Hongfa **HF118F** relay (012-1DS6T, 10 A 250 VAC). |
| `VNSD_Teardown_13_capacitor_hec_470uf.jpg` | HEC solid/hybrid electrolytic capacitor 470 µF / 25 V next to a yellow transformer (low-voltage supply section). |
| `VNSD_Teardown_14_capacitors_closeup.jpg` | Close-up of the black DC-link capacitors; blurry, little information. |

## Notes for the register/firmware work

- J26 confirms that the RJ45 sockets carry an RS485 pair; the pin order on the connector itself is `GND, 485B, 485A, 5V`.
  Mapping of J26/J20 pins to the RJ45 pins is not verified (cable colours are only partly visible).
- Relay part numbers are given as printed; their function (grid, bypass, EPS, precharge) is not determined from the photos.
- Which Venus D hardware revision these photos show is not recorded.
