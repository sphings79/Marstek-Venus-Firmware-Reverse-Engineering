# Hardware research Marstek Venus (status 2026-09-30)

> 🇩🇪 Deutsche Version: [README_de.md](README_de.md)

The original PDFs and screenshots are copyrighted and are kept locally only; the source links are in the Markdown files.

Collection of the internet research on components, approval documents and internal photos. The firmware analyses themselves
are in `Control_FW/`, `VNS_Micro_Inverter/`, `BMS/`.

## MCU identification (`MCU_Identifikation/`)

| Board | MCU | Evidence |
|---|---|---|
| Control/EMS | **Nations N32G45x with XFMC** | QSPI status bits at 0xA0001028, RCC PLL bits 18–21 + 27, GPIOF/G, SAC at 0x40024000, UID at 0x1FFFF7F0; external memory controller at 0xA0000000 with SRAM from 0x60000000 (shell command `xfmc_test`). The official Nations header `n32g45x.h` (copy local, not in the repo) defines `XFMC_REG_BASE 0xA0000000` and `QSPI_BASE = XFMC_REG_BASE + 0x1000`; the Nations driver library contains `n32g45x_xfmc.c`. The current English user manual, on the other hand, marks 0x6000_0000/0xA000_0000 as "Reserved". |
| BMS | **Nations N32G45x** | ADC1/ADC2 at 0x40020800/0x40020C00, UID at 0x1FFFF7F0, 102 vectors |
| Micro/VNS | STMicroelectronics STM32G474 | HRTIM, FDCAN, CCM SRAM, flash size register (see Micro documentation) |
| Communication module | Quectel FC41D (Beken BK7231N) | decrypted update image, strings `beken378/…` |

- `Hardware_Recherche/MCU_Identifikation/Control_MCU_Identification.md` – bit-exact reconciliation of Control firmware ↔ N32G457/AT32F403A.
- `Hardware_Recherche/MCU_Identifikation/Nations_Families.md` – review of all Nations families (N32G45x, N32G4FR, N32WB452, N32H47x …).
- `Hardware_Recherche/MCU_Identifikation/Other_Vendors.md` – feature comparison with Artery AT32F403A/407 and GigaDevice GD32F30x.
- Note: The two research reports still assume that the N32G45x has no external memory controller.
  This is refuted by the header `n32g45x.h` (see table).

## Internal photos / teardowns (`Teardown_Fotos/`)

- There are no public teardown photos of the Venus D (search in forums, YouTube, Bilibili, Reddit, GitHub).
- Own photos of an opened Venus D (board fully potted, ICs not legible): `Hardware_Recherche/Marstek Venus D/` – description in [VNSD_Teardown_Photos.md](Marstek%20Venus%20D/VNSD_Teardown_Photos.md).
- The only source with PCB photos: YouTube video „Ремонт Marstek Venus A, не вмикається!“ from the channel **AMP Service**
  (Vinnytsia), https://www.youtube.com/watch?v=ujSDFebJVoE, and Instagram reel from `amp_service_vin`.
  Board marking **HAME-POWER-H1200_V4.0, 2025-09-25** (Venus A). Three QFP processors (markings not legible
  through the potting compound), pin headers J15 `3.3V/DIO/CLK/GND` (SWD) and J16 `3.3V/TX/RX/GND` (UART), another with `GND/TMS/3.3V`.
  Components shown in the video: Infineon BSC026N08NS5 (80 V MOSFET), Silan SVS60R190 (600 V MOSFET).
- The two PNG files are screenshots from this video.
- Own photos of the Venus A **BMS board** (not potted, MCU Nations N32G455VEL7, AFE Sinowealth SH36735, unpopulated UART/SWD header J3/J2; plus CAN connector of the head unit and the 12-pin pack housing connector, 43 photos in total): `Hardware_Recherche/Marstek Venus A/` – see [VNSA_BMS_Board_Photos.md](Marstek%20Venus%20A/VNSA_BMS_Board_Photos.md).
- `Hardware_Recherche/Teardown_Fotos/*_findings.md` – search logs of the three search passes (contain the outdated assumption "STM32F3" for the Control).

## Approval documents (`Zertifikate/`)

- FCC: Hamedata (grantee 2ADOW) has registered only power stations/power banks, no Venus.
- Found (all Venus E, models MST-BIE5/2.5-2500/0800): unit certificates VDE-AR-N 4105 (Bureau Veritas, Kiwa),
  VDE-0124-100 test report (231 pp., exterior photos only), IEC 62040-1 report, UN38.3 report (cell GSP50160119F,
  3.2 V/100 Ah, 16S), IP65 report, WEEE recycling test (internal photos without legible components).
- `germany_grid_shopify.pdf` and `venus-e3_vde0124_reichelt.pdf` (both local, not in the repo) are the same file (same size, two sources).
- None of the documents names component types (MCU, AFE, radio module).
- Details and URLs: `Hardware_Recherche/Zertifikate/Certificates_Registry_Research.md`.
