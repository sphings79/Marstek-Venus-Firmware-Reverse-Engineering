# Marstek Venus D Teardown Photo Research – Findings

> 🇩🇪 Deutsche Version: [venus_d_findings_de.md](venus_d_findings_de.md)

**Search scope:** English, German, Chinese sources (photovoltaikforum, akkudoktor, reddit, GitHub, YouTube, Bilibili, Zhihu, repair forums, 1688)
**Search date:** 2026-09-30
**Update:** 2026-10-26 – own photos of an opened Venus D added; MCU table corrected
**Status:** No *public* teardown photos of an opened Venus D found. Own photos exist (see below).

## Own photos

Photos of an opened Venus D are in [`../Marstek Venus D/`](../Marstek%20Venus%20D/VNSD_Teardown_Photos.md).
The power board is fully potted; IC markings are not legible. Visible are connectors (J26 with `GND/485B/485A/5V`),
relays (Hongfa HF140FF-G, HF115F, HF118F), electrolytic capacitors, the radio module print `YDCFH0CA2AA_WIFI_X1`
and cable labels (`通讯线` = "communication line"). Details: [VNSD_Teardown_Photos.md](../Marstek%20Venus%20D/VNSD_Teardown_Photos.md).

## Key finding (public sources)

Marstek explicitly prohibits unauthorized disassembly in all user manuals. This likely explains the absence of public teardown
documentation. See: [Marstek Venus D User Manual](https://www.marstek.nl/wp-content/uploads/2025/12/Marstek-Venus-D-user-manual.pdf)

## Internal components (from firmware analysis)

| Component | Details | Source |
|-----------|---------|--------|
| **MCU – Control/EMS** | Nations N32G45x with XFMC (external memory controller) | Firmware analysis, see [MCU identification](../MCU_Identifikation/Control_MCU_Identification.md) |
| **MCU – BMS** | Nations N32G45x | Firmware analysis (ADC at 0x40020800, UID at 0x1FFFF7F0) |
| **MCU – Micro/VNS** | STMicroelectronics STM32G474 | Firmware analysis (HRTIM, FDCAN, CCM SRAM) |
| **WiFi/BLE module** | Quectel FC41D (Beken BK7231N) | Decrypted update image; the photos show the module print `YDCFH0CA2AA_WIFI_X1` (relation to the FC41D not verified) |
| **Ethernet IC** | CH395 | Control firmware (see main README) |
| **Battery management IC** | KA495XX BMIC (cell monitoring) | BMS firmware (task `RTOS_vTaskKA495XX`) |
| **Internal bus** | CAN + RS485 between the MCUs | Firmware analysis; J26 in the photos carries RS485 + 5 V |

The earlier version of this file listed "STM32F3 / Cortex-M4F" for all three MCUs. That was an unverified assumption and is wrong.

## Searched resources (no photos found)

| URL | Board type | Notes |
|-----|-----------|-------|
| photovoltaikforum.com/thread/247315 | Venus D discussion | Multiple threads; installation/setup photos only, no opened units |
| akkudoktor.net | Various Venus D issues | Troubleshooting; no internal photos |
| GitHub: sphings79/Marstek-Venus-Firmware-Reverse-Engineering | N/A – firmware only | Register/protocol mapping; no hardware photos |
| YouTube (4 unboxing/review videos) | Venus D setup | Packaging and external views; no internal disassembly |
| Heise.de Marstek Venus D test | Product review | External photos and app screenshots only |
| Marstek A1000 teardown (chongdiantou.com) | Portable power station | Different product; shows Marstek component selection but not Venus D |

## Related technical resources

- **Firmware reverse-engineering repo:** [sphings79/Marstek-Venus-Firmware-Reverse-Engineering](https://github.com/sphings79/Marstek-Venus-Firmware-Reverse-Engineering)
- **Community monitor PCB:** [stdioGangsta/MarstekMonitor](https://github.com/stdioGangsta/MarstekMonitor) (KiCAD files for an external monitoring device, not Venus D internals)
- **Modbus integration:** [ViperRNMC/marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus)

## Conclusion

No public teardown photos of the Venus D exist. Component information comes from firmware analysis and from the own photos,
which show a fully potted board and therefore confirm only connectors, relays and capacitors.
