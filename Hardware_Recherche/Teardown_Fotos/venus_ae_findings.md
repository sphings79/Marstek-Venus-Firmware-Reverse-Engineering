# Marstek Venus A/E Teardown Image Search — Results

> 🇩🇪 Deutsche Version: [venus_ae_findings_de.md](venus_ae_findings_de.md)

**Date:** 2026-09-30  
**Search strategy:** Web, GitHub, forums (DE/EN/CN), YouTube, Bilibili

## Summary

**No publicly available teardown photos of the Marstek Venus A or Venus E found.**

> Update 2026-10-28: own photos of the Venus A BMS board now exist, see [VNSA_BMS_Board_Photos.md](../Marstek%20Venus%20A/VNSA_BMS_Board_Photos.md).

### Sources searched

| Source | Status | Result |
|--------|--------|--------|
| photovoltaikforum.com | Searched | BMS/fault threads without photos; a user mentions firmware analysis |
| akkudoktor.net | Searched | 9 threads on Venus E/A; discussions are text only, no hardware images |
| mikrocontroller.net | Searched | General Marstek discussions, no teardown photos |
| GitHub | Searched | Firmware RE (rweijnen), code analysis, but no hardware teardown |
| YouTube | Searched | No disassembly videos indexed |
| Bilibili | Searched | No Marstek Venus teardown videos |
| Reddit | Searched | No hits |

---

## Findings

### 1. Manufacturer requirement: disassembly prohibited
- All manuals (Venus A, D, E v1–v3): "Do not dismantle the MARSTEK VENUS without authorization"
- Repairs only by authorized Marstek service
- Conclusion: no manufacturer-approved teardown has been published

### 2. Community resources
- **Akkudoktor, Photovoltaikforum:** Fault reports (BMS faults, FW versions), but no interior views
- **One user mentions opening the device himself** (Photovoltaikforum): "opening the storage box, taking out the board…" — but **shared no photos**

### 3. Firmware reverse engineering
| Repo | Content | Hardware images |
|------|---------|-----------------|
| rweijnen/marstek-venus-e-firmware-notes | Firmware analysis, code, extracted/ | shell.png (RS485 interface only) |
| sphings79/Marstek-Venus-Firmware-Reverse-Engineering | Ghidra analysis, Modbus registers | No hardware photos |
| sphings79/marstek-ble-control | BLE control panel | No Venus teardown images |

### 4. Hardware chips (known from firmware RE)
- **Control:** STM32F3 (or comparable)
- **BMS:** Version 117–148 (BMS firmware known, hardware markings not public)
- **WiFi/BLE:** Quectel FC41D (mentioned in firmware)
- **Ethernet:** CH395 (Photovoltaikforum post mentions boot loop)

---

## Conclusion

**For hardware images (boards, chip markings):**
1. Open and photograph a defective unit locally (e.g. RMA devices)
2. Contact Marstek/distributor directly for a service manual

**No public teardown available online.**

---

## Sources searched (web archives)
- https://github.com/rweijnen/marstek-venus-e-firmware-notes
- https://github.com/sphings79/Marstek-Venus-Firmware-Reverse-Engineering
- https://www.photovoltaikforum.com/board/247-marstek-venus-series/
- https://akkudoktor.net/t/marstek-venus-a-erfahrungen/35044
- https://www.manualslib.com/manual/4100411/Marstek-Venus-E.html
