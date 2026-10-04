# Marstek/Hamedata Venus D/A/E – Certificate and Register Research

> 🇩🇪 Deutsche Version: [Zertifikate_Register_Recherche_de.md](Zertifikate_Register_Recherche_de.md)

Status: 2026-09-30. All documents were downloaded as PDF to
`cert_search/` in the local working directory.
US FCC (grantee 2ADOW) was already checked and contains no Venus products (not checked again, as instructed).

## Hit table

| Source / location | URL | Model(s) | Document type | Date | Content |
|---|---|---|---|---|---|
| Kiwa Primara GmbH (DAkkS-accredited) | https://cdn-reichelt.de/documents/datenblatt/D400/MRST_VENUS-E3_EINHEITEN.pdf | MST-BIE5-2500 / MST-BIE5-0800 (Venus E 3.0) | VDE-AR-N 4105 unit certificate no. 25-412-00 | Issued 2025-09-28, test report 2025-09-26 | Pure table of values (PEmax, SEmax, rated voltage/current, HW V34/SW 113), harmonic/interharmonic measurement values, flicker coefficients. No internal photos, no component datasheet. Test lab Lyns-tci Technology Guangdong (A2LA 5200.02). Limited to ≤4.6 kVA. |
| Bureau Veritas Consumer Products Services Germany (DAkkS) | https://balkonkraftwerk-sued.de/wp-content/uploads/2025/07/Marstek-Venus-E-Einheitszertifikat.pdf (identical to https://cdn.tepto.de/media/.../Einheitenzertifikat_Marstek_Venus.pdf) | MST-BIE2.5-2500, MST-BIE2.5-0800, MST-BIE5-2500, MST-BIE5-0800 (Venus E, older generation) | VDE-AR-N 4105 / DIN VDE V 0124-100 unit certificate no. U24-1239 | Issued 2025-01-09, test report BWDO-ESH-P24111897, measurement period 2024-07-05 to 2024-11-30 | Firmware V137. Tables: DC voltage range 44.8–58.4 V, max. charge/discharge current 45–60 A, AC 230/400 V, Ik" 14.2 A, active/apparent power. Short description of the design: "DC and grid-side EMC filter, galvanic isolation DC/AC (HF transformer), single-fault-safe disconnection via inverter bridge + relay per phase." No photos, no parts list. |
| Lyns-tci Technology Guangdong (A2LA 5200.02) | Manufacturer's Shopify CDN: https://cdn.shopify.com/s/files/1/0955/3657/8903/files/Germany_Grid_Connection.pdf | MST-BIE5-2500 / MST-BIE5-0800 (Venus E, firmware BMS 106 / SW 113) | Full test report DIN VDE V 0124-100 / VDE-AR-N 4105:2018-11, report no. LS2A25082503EGDE01, 231 pages | Issued 2025-11-26 | **Annex 2 "Pictures of the unit" (pp. 225–231): photos of enclosure front/rear/side views incl. close-ups of the rating plate (no internal views/PCB).** Contains complete test curves (flicker, grid support, LV protection, island detection etc.) and a copy of the rating plate with all technical data (charge current 60 A, IP65, isolated topology, protection class I). |
| Hwa-Hsing (Dongguan) Testing / Lyns-tci | https://cdn.shopify.com/s/files/1/0955/3657/8903/files/LVD_IEC62040_Test_Report.pdf | MST-BIE5-800 / MST-BIE5-2500 (Venus-E) | IEC 62040-1 (UPS safety) test report no. 25090404001-SF-LD, 112 pages | Issued 2025-11-11 | Ratings table: battery LiFePO4 51.2 V, max. charge current 120 A / discharge current 200 A (system limits, not the BMS cell value), backup/EPS max. 3.5 kVA. Annex 2 "Pictures of the unit" p. 100 (content not checked; the page count suggests enclosure photos only, as in the VDE report). |
| Shenzhen CCJC Technology (CNAS L9856) | https://cdn.shopify.com/s/files/1/0955/3657/8903/files/UN38.3_Certificate.pdf | MST-BIE5-2500 (Venus-E), cell GSP50160119F | UN38.3 test report no. CCJC2514572R01, 18 pages | Test 2025-05-19 to 2025-06-23 | Cell/pack data: 3.2 V / 100 Ah per cell, pack 51.2 V / 100 Ah / 5120 Wh, 16S1P, end-of-charge voltage 57.6 V, end-of-discharge voltage 48.0 V. All 8 UN38.3 tests passed (altitude simulation, thermal, vibration, shock, short circuit, crush, overcharge, forced discharge). No internal photos. |
| CCIC Southern Testing Co. (CNAS) | https://cdn.shopify.com/s/files/1/0955/3657/8903/files/IP65_Test_Report.pdf | MST-BIE5-2500 (representative also for MST-BIE5-0800) | IEC 60529 IP test report no. 20250910W23238, 10 pages | Test 2025-09-04 to 2025-09-11 | IP6X (dust) + IPX5 (water jet) passed. Contains test photos (dust chamber, water jet test) and 2 product photos (outer enclosure, black/high-gloss variant with "MARSTEK" lettering, RJ45 connectors visible). No internal view. |
| Shen Zhen Webb Testing Technology | https://cdn.shopify.com/s/files/1/0955/3657/8903/files/WEEE_Certificate.pdf | MST-BIE5-2500 / MST-BIE5-0800 (Venus-E) | WEEE recyclability test report no. WEB250827011RC (Directive 2012/19/EU Art. 11 / Annex V), 9 pages | Sep. 2025 | **Contains "Disassembly Tree" with internal photos**: A1 enclosure interior (wiring, display), A2 enclosure rear wall inside, A3 shredded plastic granulate, **A4 green printed circuit board (PCB) photographed – no component markings legible/no bill of materials**, A5 wiring harness, A6 16 prismatic battery cells (LFP, visible in 2 groups of 8). Total weight 58.80 kg, of which battery 45.5 kg, PCB 2.80 kg. Reuse/recycling 82.4%, recovery 88.1% (pass). No MCU/AFE chip designations legible. |
| Marktstammdatenregister (MaStR) | (Marktstammdatenregister entry, unit number removed) | Venus E | Unit registration (operator data, no manufacturer certificates) | – | Only system data (5.12 kWh usable capacity, 2.5 kW inverter power), no technical internal details. |
| ZEREZ (FGW) | zerez.net | – | – | – | No publicly findable Marstek entries; the ZEREZ full-text search cannot be searched without access/login, web search returns no hits. Cannot be conclusively verified without registration (outside the permitted research scope, as login is required). |

## Checked, but without usable hit

- **TÜV Rheinland Certipedia** (certipedia.com): No Marstek/Hamedata entries found via web search; according to the documents found, the Venus series certifications go through Bureau Veritas Germany and Kiwa Primara, not TÜV Rheinland.
- **TÜV SÜD, Intertek Directory, CB Scheme (IECEE)**: No Marstek/Hamedata hits found in the publicly searchable databases.
- **Clean Energy Council Australia** (approved batteries/inverters): No confirmation of a Marstek Venus entry via web search; would have to be checked directly in the CEC database (login/full-text search required).
- **Japan TELEC/技適 (tele.soumu.go.jp)**: No Marstek hits; the database can only be searched via form search (not findable by web crawler).
- **Korea KC/RRA (rra.go.kr)**: No Marstek-specific hits; database only via form search.
- **Brazil Anatel (Mosaico)**: No Marstek hits.
- **China SRRC/CCC**: No publicly indexed Marstek certificates found.
- **EU EPREL**: No Marstek Venus entries findable (EPREL covers product groups subject to energy labelling; home storage systems currently do not necessarily fall under it).
- **Bluetooth SIG Qualified Product Listing**: No direct confirmation that Quectel FC41D (or a Marstek-specific combination) is listed in Launch Studio – the database cannot be searched without a targeted QDID search/login.
- **Wi-Fi Alliance**: No Marstek-specific entries findable.
- **UKCA**: No separate UKCA declaration of conformity publicly findable (probably part of the manuals, but no separate PDF found).
- **marstek.de/pages/downloads**: 404, download area under a different URL structure; not identified within the time budget.

## Key findings on the original question (MCU/AFE/radio module evidence)

None of the public certificates/test reports found names concrete component part numbers (Nations N32G45x, KA495xx AFE, Quectel FC41D, CH395). The only find with real internal photos is the **WEEE recycling test from Shen Zhen Webb Testing** (see table) – it shows a green printed circuit board (photo A4) and the cell configuration (16 prismatic LFP cells, photo A6), but without legible chip markings or a bill of materials. For component identification (MCU/AFE/radio module), our own Ghidra firmware analysis or a physical teardown photo series (e.g. from forums/YouTube teardowns) remains the more productive source than the certification registers checked.
