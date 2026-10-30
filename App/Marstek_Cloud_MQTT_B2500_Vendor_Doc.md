# Marstek cloud MQTT vendor documentation (B2500 family)

> 🇩🇪 Deutsche Version: [Marstek_Cloud_MQTT_B2500_Hersteller_Doku_de.md](Marstek_Cloud_MQTT_B2500_Hersteller_Doku_de.md)

Notes on the MQTT protocol page that Marstek publishes on its own static host. This document is a summary in our own words;
the vendor page itself is kept only locally (third-party material, not republished).

| | |
|---|---|
| Source | `https://static-eu.marstekenergy.com/ems/mqtt/index.html?version=<n>&type=<devType>&mac=<id>` |
| Retrieved | 2026-10-30 |
| Local copy | `Hersteller_Doku/Marstek_MQTT_Doku_B2500_2026-10-30.html` (SHA-256 `4cc72384…af2a`), not in any repo |
| Page title | "MQTT" (English, with a "Download PDF" button) |

## 1. What the page is, and what it is not

- **The URL parameters do almost nothing.** The page script reads `type` and `mac` and writes them into the topic placeholders
  (`{type}`, `{uid or mac}`). `version` is not evaluated at all. The same text is served for every product.
  The example link used `type=HMG-50`, but that is only a placeholder fill, not a statement about HMG-50.
- **The content is the B2500 family.** Notes in the command sections name first-generation B2500 (product codes `HMB-`)
  and second-generation B2500 (`HMA-`, `HMF-`, `HMK-`). Telemetry (two solar inputs, two outputs, two battery packs) matches that.
- **It is not proven to describe Venus.** The `cd` numbering is per product (see
  [Marstek_App_Command_Vocabulary_1.6.76.md](Marstek_App_Command_Vocabulary_1.6.76.md), section 5 for B2500, section 4 for Venus).
  The Venus D commands use other meanings for the same numbers (for example `cd=2` = work mode there). Do not transfer
  this page to Venus devices.
- The B2500 section of the app vocabulary also lists `cd=07` for B2500 (`setB2500State`, `setTimePower`), which fits the page. The other commands were not compared.

## 2. Transport

| Direction | Topic | Use |
|---|---|---|
| Device → cloud/app | `hame_energy/<type>/device/<id>/ctrl/#` (subscribe) | Telemetry as one line `key=value,key=value,...` |
| App → device | `hame_energy/<type>/App/<id>/ctrl` (publish) | Commands, payload starts with `cd=<n>` |

`cd=01` (publish, no other field) requests a fresh telemetry line.

## 3. Telemetry fields (subscribe payload)

Fields without a version note exist in all versions the page covers (212 to 220).

| Key | Meaning |
|---|---|
| `p1`, `p2` | Solar input status 1 / 2 |
| `w1`, `w2` | Solar input power 1 / 2 |
| `pe` | Battery percentage |
| `vv` | Device version number |
| `cs`, `cd` | Charging settings, discharge settings |
| `am` | "AM" (no explanation on the page) |
| `o1`, `o2` | Output state 1 / 2 |
| `do` | Depth of discharge |
| `lv` | Battery output threshold |
| `cj` | Scene |
| `kn` | Battery capacity |
| `g1`, `g2` | Output power 1 / 2 |
| `b1`, `b2` | Power pack 1 / 2 connected |
| `md` | Discharge setting mode |
| `d1`..`d3`, `e1`..`e3`, `f1`..`f3`, `h1`..`h3` | Time slots 1 to 3: enabled, start (`H:MM`), end, output value |
| `d4`, `d5`, `e4`.., `f4`.., `h4`.. | Time slots 4 and 5 (version 218.2 and later) |
| `sg`, `sp`, `st` | Sensor connected, automatic power of the monitor, power sent by the monitor |
| `tl`, `th` | Minimum / maximum cell temperature |
| `tc`, `tf` | Charging / discharge temperature alarm |
| `ts` | WiFi signal |
| `fc` | Version of the communication chip (FC4) |
| `id`, `a0`, `a1`, `a2`, `l0`, `l1` | Device ID, capacities of host / extra pack 1 / extra pack 2, status bits (212.17 and later) |
| `bc`, `bs`, `pt`, `it` | Daily totals: battery charge, battery discharge, PV charge, micro-inverter feed-in (218 and later) |
| `c0`, `c1`, `m0`..`m3` | CT001 channel, CT001 diagnosis state, CT001 clamp powers, micro-inverter real-time power (218.2 and later) |
| `lmo1`, `lmo2`, `lmi1`, `lmi2`, `lmf` | Rated output / input power of channel 1 and 2, "device is limited" flag (220.1 and later) |

**Errors in the vendor page** (noticed when comparing the sample payload with the table; not verified on a device):
the sample line has `pe-99` (probably `pe=99`), `02=1` (probably `o2=1`), `1v=200` (table says `lv`),
`t1=26` (table says `tl`), `2=0:0` (a lost `e2`/`f2` key) and `e3=0:0f3=0:0` (missing comma).

## 4. Commands (publish payload)

"Flash" = the setting is stored persistently on the device.

| Payload | Effect | Flash | Version |
|---|---|---|---|
| `cd=01` | Request telemetry | - | - |
| `cd=03,md=0` / `md=1` | Charge mode: charge and discharge at the same time / fully charge first, then discharge | yes | - |
| `cd=17,md=0` / `md=1` | same as `cd=03` | no | 214.1 and later |
| `cd=04,md=0..3` | Discharge mode: all outputs off / OUT1 / OUT2 / OUT1 and OUT2 (first-generation B2500 only, `HMB-`) | yes | - |
| `cd=18,md=0..3` | same as `cd=04` | no | 214.1 and later |
| `cd=05,md=<0-100>` | Depth of discharge in percent (example: `md=95`) | yes | - |
| `cd=19,md=<0-100>` | same as `cd=05` | no | 214.1 and later |
| `cd=06,md=<0-500>` | Battery output start threshold in W (example: `md=300`) | - | - |
| `cd=07,md=0,a1=..,b1=..,e1=..,v1=..,a2=..,...` | Three time slots: `aN` on/off, `bN` start, `eN` end, `vN` output value, range 80 to 800 W (second-generation B2500: `HMA-`, `HMF-`, `HMK-`) | yes | - |
| `cd=07,md=1` | Power follows the monitor automatically | yes | - |
| `cd=20,...` | same parameters as `cd=07` | no | 214.1 and later |
| `cd=08,wy=<offset>,yy=<year-1900>,mm=<0-11>,rr=<1-31>,hh=<0-23>,mn=<0-59>,ss=<0-59>` | Set the time (month counts from 0) | - | - |
| `cd=09,wy=<offset>` | Set the time zone offset (example: `wy=480`) | yes | - |
| `cd=10` | Restart | - | - |
| `cd=11` | Factory reset | - | - |

The page gives the time-zone offset `wy=480` as its example (480 is probably minutes east of UTC, +08:00; the page does not say).
Each time-slot command sends all three slots in one payload; the page shows the first, second and third period as separate examples.

## 5. Why this is kept

- It is a vendor-written description of a Marstek MQTT payload (we did not search for others). It shows the vendor's own
  field naming and the `key=value,...` payload style, which is the style the app uses for every product.
- For the **Venus D** work it is background only. Venus D MQTT commands are in the app vocabulary document, derived from the app code.
- Open: whether other products have an equivalent page. The path `/ems/mqtt/index.html` is public; other file names were not probed.

## 6. Provenance of this document

| Statement | Basis |
|---|---|
| Page content, fields, commands, version notes | read from the page |
| URL parameters only fill placeholders | read from the page script (three inline scripts, one of them 3639 characters) |
| Content describes B2500 | notes in the page itself (`HMB-`, `HMA-`, `HMF-`, `HMK-`) |
| Typos in the sample payload | comparison of the sample with the field table, not tested on a device |
| Nothing was sent to any device | read-only download of one public page |
