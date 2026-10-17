# Cloud Upload Payload v150 — Field Reference

> 🇩🇪 Deutsche Version: [VNSD_Cloud_Upload_Payload_v150_de.md](VNSD_Cloud_Upload_Payload_v150_de.md)

What the Venus D sends under firmware v150 to
`https://api-eu.marstekcloud.com/data-upload/v1/venus/<uid>`.

**Collection method:** no capture against the real cloud, but a local
endpoint that the device points to via DNS redirection and that
answers with `{"code":0}` (see `VNSD_Control_FW_Analysis_app_0150_0805_115146.md` §4a as well as
<https://github.com/sphings79/Marstek-offline-endpoint>). Recorded on
2026-08-25 on a Venus D, Control 150 / BMS 118 / Inverter 116 / MPPT 104.

**Region code confirmed:** `eu`. The device requested
`http://eu.hamedata.com/app/neng/getDateInfoeu.php?…` — both `%s` in the
firmware template are filled with the same code. This means `api-eu.marstekcloud.com`
is no longer merely inferred but measured.

## Transport

| | |
|---|---|
| Method | `POST` |
| Path | `/data-upload/v1/venus/<uid>` — `uid` identical to the parameter from `getDateInfo` |
| Body | **one** form field `d`, containing a URL-encoded query string |
| Size | ~770 bytes |
| Interval | irregular during the test period, triggered via the state machine in `FUN_08015bd0` |

## Raw example

`di`, `sn` and `ip` are replaced by placeholders.

```
d=di=<uid>&sn=<mac>&to=1561&td=1933&ed=0&em=1561&gd=241&gm=1933&wm=10&gy=0&gp=0
&go=591&gt=3&gf=500&gv=2424&ct=0&bs=3&bp=299&eb=0&dn=150&bu=0&t1=268&t2=273&t3=0
&vc=13,14,3248,3245,5,14,3187,3183,7,4,3276,3272,1,6,3281,3279,5,0,3247,3245,3,1,3264,3262
&tc=249,254,252,253&dt=2026-08-25%2004:01:14&no=48&e1=0&ws=39&mt=0&ty=4&sv=0
&mc=2500&md=2500&sc=19&ci=500&ri=500&bv=5247&bi=-121&pb=-634&ds=12&ph=0&bt=1&mq=0
&cy=4&cw=0&cp=0,0,0,0&bm=118&ce=0,0,0,0&up=0&ap=0&nt=1&iv=116&et=4&ea=800
&pv=99,0,0,0,99,0,0,0,99,0,0,0,99,0,0,0,0,0&sk=2,0,6,63,0,0,0,0,0,0,118,118,118,118,118,118
&mv=104&me=0&ma=0&fu=0,0&ms=0,1,0,0,0&im=0,-3,0,0,74,0,590,241,576,2500,57&hd=0,0,0
&pw=600&bl=1&as=0&bke=0,0,0&bkd=0,0,0&ip=<ip>&bt_p=0,0&ival=5&soh=0
```

## Confirmed fields

Confirmed means: the same value was read in the same period from a second
source — via Modbus (Home Assistant) or from the status message of the
device. Mere plausibility does not count here.

| Key | Meaning | Scaling | Evidence |
|---|---|---|---|
| `di` | Device UID | — | identical to `uid` in `getDateInfo` and the upload path |
| `sn` | Serial number/MAC | — | form `aabbccddee01` = MAC notation |
| `ip` | own IP | — | matches the source IP of the request |
| `dt` | Device clock | — | URL-encoded, `YYYY-MM-DD hh:mm:ss` |
| `wm` | **Mode byte** | — | `10` = `0x0A`; identical to the byte behind register 42000/43000 |
| `sc` | State of charge | % | 19 against status message "SoC 19 %" |
| `pb` | Battery power | W | −634 against Modbus −629 (time offset) |
| `bv` | Battery voltage | ×0.01 V | 5247 → 52.47 V |
| `bi` | Battery current | ×0.1 A | −121 → −12.1 A |
| `go` | AC/grid power | W | 591 against status message "AC 591 W · Netz 591 W" |
| `gv` | Grid voltage | ×0.1 V | 2424 → 242.4 V against Modbus 242.6 V |
| `gf` | Grid frequency | ×0.1 Hz | 500 → 50.0 Hz against Modbus 49.9 Hz |
| `mc` | max. charge power | W | 2500 against register 44002 |
| `md` | max. discharge power | W | 2500 against register 44003 |
| `t1` | Temperature internal | ×0.1 °C | 268 → 26.8 °C |
| `t2` | Temperature MOS | ×0.1 °C | 273 → 27.3 °C |
| `tc` | Cell temperatures | ×0.1 °C | 254 → 25.4 °C against status message "Zelle 25,4 °C" |
| `dn` | Control firmware | — | 150 |
| `bm` | BMS firmware | — | 118 |
| `iv` | Inverter firmware | — | 116 |
| `mv` | MPPT firmware | — | 104 |

**`wm` is the most interesting entry.** It is the same mode byte that registers 42000 and 43000
share (see
`Modbus_RS485_TCP/VNSD_Register_Persistence_RAM_vs_EEPROM_v150.md` §6a). If it falls
back on the device side, this is from now on also visible in the upload — with a
timestamp and without polling.

## Not yet assigned

> **Update 2026-10-15:** this list is resolved (all but `hd`) in
> [VNSD_Cloud_Upload_Key_Table_v151.md](VNSD_Cloud_Upload_Key_Table_v151.md), read from the Control v151
> code. The text below is the state of 2026-08-25.

31 keys, deliberately left here without interpretation:

```
bp bs bt bu ci ct cy ds e1 eb ed em gd gm gp gt gy ival mq mt no ph ri soh
sv t3 td to ty vc ws
```

plus the list fields `cp ce fu ms im hd pv sk bke bkd bt_p`.

Two warnings on this. `sv` is `0` in the payload, whereas the same
parameter name in the `getDateInfo` URL carries the Control version `150` — **the
names are not consistent across the endpoints.** And `vc` looks like
cell voltages in mV (values around 3250), but the grouping is unresolved.

## The time URL `getDateInfoeu.php` — parameters are version statuses

*(Field capture 2026-08-25 on the own offline endpoint. Unlike the upload,
this request goes over **plain HTTP on port 80** to `eu.hamedata.com`.)*

```
GET /app/neng/getDateInfoeu.php?uid=<device-id>&fcv=202409090159&aid=VNSD-0
    &sv=150&sbv=118&mv=101&cert=0&boot=18&inv=116&mpptv=104
```

| Parameter | Meaning | Register | Evidence |
|---|---|---|---|
| `uid` | Device ID (24 digits) | — | identical to the path segment of the upload |
| `aid` | Model identifier `VNSD-0` | — | corresponds to the image name `VNSD-0_app_0150_…` |
| `fcv` | Communication module firmware | — | date form `202409090159` |
| `sv` | Control app version = 150 | **30200** `ems_ver` | register map, confidence high |
| `boot` | EMS/Control **bootloader** = 18 | **30201** `ems_boot_ver` | `0x20000038`, version debug print `0x08036F52`; live scans show 18 |
| `inv` | Micro inverter (VNS) = 116 | **30202** `vns_ver` | `0x2000015E` |
| `sbv` | BMS = 118 | **30204** `bms_ver` | `0x20014FD2` |
| `mpptv` | MPPT = 104 | **30205** `mppt_ver` | `0x20000188` |
| `cert` | 0 | — | consistent with `Authmode 0` in `HTTPS_TLS_Session_Init` |

**Open:** `mv=101`. In the **upload payload**, `mv` stands for the MPPT version (there
`mv=104`, see field table above) — in this URL it cannot be that, because
`mpptv=104` is already next to it. Which version status 101 refers to is not
evidenced. `vns_boot_ver` (30203, value 106) and the pack versions do not appear in the
URL.

**Correction:** `boot` was initially read as a boot counter. That is wrong —
all the other parameters in the series are version statuses, and the value matches
register 30201.

## Method for further work

Change a DEV register's value, then look for exactly
that value in the next payload. If a single key remains after two or three changes,
the assignment is certain. A decoder script for this is in the endpoint repo
(`decode.py`, local, not in the repo).

**This does not work for every register, though.** Counterexample from
practice: 37021/37022 (inverter struct `0x20014F4C` +0x2c) does not appear in the upload.
The function that reads the value is called `Cloud_Report_FillGridData`,
but has exactly one caller — `MQTT_BuildEsDataResponse`. The value therefore goes
to **MQTT**, not to the HTTP upload. Anyone who wants to see everything the device emits
cannot get around the MQTT path, and that is secured with `Authmode 2` including a
client certificate.

The upload record itself is built by `MQTT_Telemetry_Struct_Builder` (despite the
name), called immediately before `Telemetry_History_Record_Push`. That is the
entry point if the field assignment is to come from the code instead of empirically.
