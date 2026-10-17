# MPPT Hardware — External Research Snapshot

> 🇩🇪 Deutsche Version: [MPPT_Hardware_External_Research_de.md](MPPT_Hardware_External_Research_de.md)

**Date:** 2026-10-17. **Method:** web search only (no device access). **This is a snapshot, not a conclusion** — forums, teardown blogs and patent filings change; re-run before relying on "nothing found" again, especially if months have passed.

## Why this exists

The MPPT sub-controller (separate chip, CAN-connected to Control, firmware version 104 per the CAN string `V150.116.104.118`, see [VNSD_Control_Charge_Power_PV_Curtailment_v150.md](../Control_FW/VNSD_Control_Charge_Power_PV_Curtailment_v150.md) §3.2) has no image in this project. Before planning a hardware approach (teardown, chip ID, SWD dump), it made sense to check whether anyone else already published that information.

## What was searched

Web search (English, German, Chinese) for: Venus D/E/A teardown, internal photos, PCB, serial console / UART access, MPPT chip identification, FCC ID filings, GitHub firmware dumps, and forum discussions (akkudoktor.net, community.simon42.com, Home Assistant community, Homey community, photovoltaikforum-adjacent terms).

## Result: nothing found

- **No teardown** with photos of the Venus D/A/E internals, in any language.
- **No serial-console or UART-pad discussion** for any Venus model — all forum hardware interest is about the documented external interfaces (RS485/Modbus, local UDP API, BLE), not internal debug access.
- **No MPPT chip identification** anywhere public.
- **No FCC ID** found for Marstek or Hamedata-branded Venus units (may be certified under a different name, or sold in markets without that requirement — not checked further).
- **No firmware-dump repository** on GitHub beyond the usual Home Assistant/Modbus integrations (which talk to the device's own documented interfaces, not its internals).

The public interest in these devices is entirely about **using** them (Modbus registers, local API, HACS integrations, cloud-relay add-ons like "Hame Relay") — not about analysing them. As far as this search can tell, this project is currently the only public reverse-engineering work on this hardware.

## The manufacturer, confirmed

Marstek is the consumer brand of **Shenzhen Huamei Xingtai Technology Co., Ltd.** (深圳市华美兴泰科技股份有限公司, trading as "Hamedata"/"Hame" — matches the `_HM` build suffix, `Hame147258` default Wi-Fi password and `hamedata.com` endpoints already found in the firmware). Hamedata's own marketing names BMS, PCS, EMS and MPPT as in-house core technologies, consistent with there being no separate third-party MPPT vendor to identify — the MPPT firmware is presumably their own, like the Control/BMS firmware.

## Patents (Google Patents, assignee search)

Assignee `Hamedata` / `深圳市华美兴泰科技` returns several hundred results (464 for the broader Chinese-entity-name search), almost all unrelated (battery cell chemistry/manufacturing, consumer electronics from an older product line). One hit is on-topic:

- **WO2025179896A1**, "Rapid MPPT method, system and apparatus based on unipolar inverter circuit" — assignee Hamedata Technology Co Ltd, filed 2024-10-12, published 2025-09-04, priority 2024-02-28. Describes an MPPT algorithm for single-stage unipolar (flyback-type) inverters: PV-side voltage/current sampling at 45° grid-phase steps, a power-compensation factor from cycle-to-cycle comparison, feeding a current-loop inverter control. It mentions "a microcontroller" generically for the sampling/compute step and gives **no chip part number, no CAN/UART detail, no register addresses**. Plausibly related to this product line's MPPT work, but it documents an algorithm, not the hardware.
- No patent found that names a specific MCU, a bootloader, a firmware-update mechanism, or a CAN/Modbus register map.

## Conclusion

Nothing external closes any of this project's open MPPT questions (chip identity, firmware image, bootloader). The patent search adds one data point (an MPPT algorithm patent consistent with Hamedata doing MPPT control development in-house) but no hardware specifics. The only paths that remain are the ones already in this project's own plan: physical teardown + chip ID, then SWD/JTAG if the chip supports it (see [VNSD_SWD_JTAG_Dump_Plan.md](VNSD_SWD_JTAG_Dump_Plan.md)), or capturing an MPPT firmware OTA transfer if Marstek ever ships one over a channel this project can observe.
