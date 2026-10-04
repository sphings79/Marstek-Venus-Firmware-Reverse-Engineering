# Control (EMS) Board: AT32F403A vs. N32G45x — Offline-Analyse (Control v151)

> 🇬🇧 English version: [Control_MCU_Identification.md](Control_MCU_Identification.md)

Quelle: `Dekompilate/VNSD_Control_v151/` (lokal, nicht im Repo) (Export, keine Ghidra/ReVa-Live-Tools verwendet).

## 1. Register bei 0xA0000000 / 0xA0001000 und RCC/CRM-AHBEN

**Board_XMC_SRAM_Init @ 0x0801e094** (aufgerufen aus **SystemInit @ 0x080333e0**, Zeile `Board_XMC_SRAM_Init();` — also echter, ausgeführter Boot-Pfad, kein toter Code):
- ruft `RCC_AHBPeriphClockCmd(0x100, 1)` (0x0801e09a/0x0801e09c), dann `Board_XMC_ConfigGPIO_AddressBus @ 0801e100`, `Board_XMC_ConfigGPIO_DataBus @ 0801e240`, `Board_XMC_ConfigGPIO_ControlBus @ 0801e288`.
- baut danach einen Registerblock auf dem Stack, `bank_regs[0]=0xa0000000` (BCR1), ruft `Board_XMC_BuildControlAndTimingRegs @ 080358ce` (OR-verknüpft Timing-Felder → BCR1/BTR1/BWTR1-Muster, klassisches ST/AT32-FSMC/XMC-Modell) und `Board_XMC_SetBankEnable @ 080358b8` (setzt/löscht nur Bit 0 = Bank-Enable).

**Befund AT32F403Axx_v2.svd**: Peripherie `XMC` genau bei `0xA0000000`, Größe `0x1000`, mit Registern `BK1CTRL1` (offset 0x0), `BK1TMGWR1`/`BK1TMGWR4` (0x104/0x11C) etc. — das ist exakt das Register-Layout, das der Code beschreibt (BCR/BTR/BWTR, Bit0=Enable).

**Befund N32G457.svd / N32G455.svd / N32G452.svd**: **kein** Peripherie-Eintrag bei `0xA0000000`. Einziger Eintrag oberhalb `0x90000000` ist `QSPI` bei `0xA0001000` (Größe 0x400). Laut N32G457-Datenblatt (S. 15, "Figure 2-1 Memory map") ist `0xA000_0000–0xA000_0FFF` explizit **"Reserved"**; ebenso `0x6000_0000–0x8FFF_FFFF` ("Reserved", kein externer Speicher-Controller) — das offizielle "QSPI Bank"-XIP-Fenster liegt stattdessen bei `0x9000_0000–0x9FFF_FFFF`.

→ Die BCR/BTR/BWTR-Schreibzugriffe bei 0xA0000000 passen bitgenau zu **AT32** (XMC), sind auf **N32G45x laut Datenblatt Reserved-Adressraum**.

**0xA0001000** wird separat genutzt (`QSPI_Flash_IsBusy @ 08017914`, `QSPI_TransferReady_Poll @ 0801792c`, `QSPI_WriteComplete_Check @ 08017944`, alle über `PTR_EXTFLASH_REG_000` = 0xa0001000, Statuswort bei Offset +0x28):
- Bit0 wird als „busy" geprüft, Bit3 als „transfer ready", Bit1 (invertiert) als „write complete".

**Befund N32G457/455/452.svd**: `QSPI_STS` @ Offset `0x28`, Bits: `BUSY`=0, `TXFNF`=1, `RXFNE`=3, `RXFF`=4, `DC_ERR`=6 — **bitgenauer Treffer** für alle drei verwendeten Bits (0,1,3).
**Befund AT32F403Axx_v2.svd**: **keine QSPI-Peripherie überhaupt** vorhanden (grep liefert 0 Treffer) — 0xA0001000 wäre auf AT32 unbenutzter/undokumentierter Adressraum direkt hinter dem 0x1000-großen XMC-Block.

→ Für 0xA0001000 ist **nur N32G45x** überhaupt eine gültige Interpretation; AT32 hat dort schlicht keine Hardware.

**RCC/CRM AHBEN (0x40021014)**: Aufruf `RCC_AHBPeriphClockCmd(0x100,1)` in `Board_XMC_SRAM_Init` setzt Bit 8 des AHB-Clock-Enable-Registers — bei AT32 CRM_AHBEN ist das exakt `XMCEN`; bei N32 RCC_AHBPCLKEN müsste dort ein anderes/kein Feature sitzen (im vorliegenden Kontext ohne dokumentiertes SMC nicht sinnvoll benannt) — dieser Punkt ist für sich wenig aussagekräftig, da Bit-Namen aus der (AT32-)SVD-Zuordnung stammen.

## 2. Nutzung von 0x60000000..0x60026AF7

Scatter/Linker-Region `.xmc_bss` (`60000000-60026af7`, 235 referenzierte Adressen) wird wie **echtes, beschreibbares RAM** benutzt, kein XIP-Fenster:
- `Board_ExternalSRAM_SelfTest @ 0x08035980` (Shell-Kommando **"xfmc_test"**, String-Literal bei `0x0805b58e` im Image vorhanden, also real im Firmware-Image, kein Analyse-Artefakt): liest/schreibt Bytes bei `0x60000000+idx`, füllt mit Inkrement-Pattern bzw. 0xFF, alles über `printf`-Ausgabe verifiziert — klassischer externer-SRAM-Selbsttest.
- Weitere Nutzung als persistente Zähler/Puffer (`DAT_600238c8`, JSON-RPC-Empfangspuffer ab `DAT_60025518`, etc. laut `globals_and_registers.txt`).

Es handelt sich also um denselben externen-SRAM-Adressraum, den `Board_XMC_SRAM_Init` über die BCR/BTR-Register bei 0xA0000000 konfiguriert (Bank1 NOR/SRAM-Fenster, klassisches ST/AT32-Layout: Steuerregister @0xA000_0000, Datenfenster Bank1 @0x6000_0000). Laut N32G457-Datenblatt ist genau dieser Speicherbereich (0x6000_0000–0x8FFF_FFFF) **"Reserved"** — es gibt dort keinen dokumentierten externen Speicher-Controller.

## 3. Weitere unterscheidende Peripherie

- **GPIOF/GPIOG** (0x40011C00 / 0x40012000): `Board_XMC_ConfigGPIO_AddressBus @ 0801e100` konfiguriert laut Kommentar "GPIOF/GPIOG (A0-A9,A10-A15) und GPIOD (A16-A18)" als AF-Push-Pull-50MHz-Pins für den Adressbus — **tatsächlich benutzte, funktionale GPIO-Ports**.
  - AT32F403Axx_v2.svd: nur `GPIOA..GPIOE` (0x40010800–0x40011800) vorhanden, **kein GPIOF/GPIOG**.
  - N32G457.svd: `GPIOA..GPIOG` inkl. `GPIOF@0x40011C00`, `GPIOG@0x40012000` vorhanden.
  - → GPIOF/G-Nutzung ist auf AT32F403A (lt. dieser SVD/RM) **nicht möglich**, auf N32G45x **exakt passend**.

- **RCC/CRM_CFG PLL-Multiplikator-Feld** (`RCC_GetClocksFreq @ 0802d864`): Maske `0x83c0000` = Bits {18,19,20,21,27}; wenn Bit 27 gesetzt: `mult = feld - 0x1ef` (Bereich 17–32), sonst `mult = feld + 2` (Bereich 2–17).
  - **N32G457.svd RCC_CFG**: `PLLMULFCT` Bits 21:18 (4 Bit) **+ PLLMULFCT_H Bit 27** (1 Bit) — **bitgenauer Treffer**, inkl. exakt derselben Bitposition 27 für das High-Bit.
  - **AT32F403Axx_v2.svd CRM_CFG**: `PLLMULT3_0` Bits 21:18, aber die Erweiterung liegt bei **PLLMULT5_4 (Bits 30:29)**, **PLLRANGE bei Bit 31**, Bit 27 ist dort `USBDIV2` — passt nicht zur Maske des Codes.
  - Der resultierende Multiplikatorbereich (max. ~32, mit 8-MHz-Quarz/HSE-Vorteiler ⇒ max. Systemtakt ~144 MHz) passt zur dokumentierten N32G457-Obergrenze, nicht zu AT32F403A (bis 240 MHz, dort wäre ein 6-Bit-Multiplikatorfeld mit Bits 29/30 nötig).
  - → Klarer, bitgenauer Treffer für **N32G45x**, Gegenbeweis gegen AT32.

- Funktionsnamen der Clock-Treiber (`RCC_HSEConfig`, `RCC_AHBPeriphClockCmd`, `RCC_APB1PeriphClockCmd`, `RCC_APB2PeriphClockCmd`, `RCC_GetClocksFreq`, `RCC_WaitForHSEStartUp`) folgen der klassischen ST/N32-SPL-Namenskonvention (`RCC_...`), nicht AT32s `crm_...`-Firmware-Library-Konvention. Nur als unterstützendes Indiz (Namen könnten aus einer geteilten/portierten SDK-Basis stammen), nicht als Beweis gewertet.

- UID-Auslesung `Board_GetUniqueID_HexString @ 0801xxxx` (siehe Vorgabe) liest bei `0x1FFFF7F0` — passt zu N32G45x-Dokumentation (AT32/STM32F1: `0x1FFFF7E8`). Bestätigt hier nur wiederholt, nicht neu geprüft.

- Keine Hinweise im Export auf AT32-exklusive Peripherie wie SDIO2 (0x40023400), ACC (0x40015800), UART7/8 (0x40016400/0x40016800) gefunden (kein Treffer für diese Adressen in `globals_and_registers.txt`).

## 4. Fazit

**N32G45x**, mit hoher Konfidenz — wahrscheinlich **N32G457** (oder ein eng verwandtes Geschwistermodell G455/G452 mit identischem QSPI/RCC-Kern; alle drei SVDs zeigen dieselbe QSPI-Basisadresse/-Register).

**Dafür (bitgenau, primäre Quellen):**
1. QSPI-Statusregister bei `0xA0001000+0x28`: benutzte Bits 0/1/3 stimmen exakt mit N32-`QSPI_STS` (BUSY/TXFNF/RXFNE) überein; AT32F403A besitzt überhaupt keine QSPI-Peripherie.
2. RCC_CFG-PLL-Multiplikator-Dekodierung (Bits 18-21 + Bit 27) stimmt exakt mit N32G457 `PLLMULFCT`/`PLLMULFCT_H` überein; AT32 CRM_CFG hat die Erweiterung an anderer Bitposition (29/30).
3. Firmware konfiguriert und nutzt GPIOF und GPIOG — auf AT32F403A laut SVD/RM nicht vorhanden, auf N32G45x vorhanden.
4. UID-Adresse 0x1FFFF7F0 (aus Vorgabe) passt zu N32, nicht zu AT32/STM32F1.
5. Konsistent mit dem bereits für das BMS-Board derselben Plattform bewiesenen N32G45x (ADC @ 0x40020800).

**Dagegen / ungeklärt:**
- Der komplette "XMC"-Codepfad (`Board_XMC_SRAM_Init`, BCR/BTR/BWTR-Register bei 0xA0000000, externes SRAM-Fenster 0x60000000–0x60026AF7, Selbsttest-Shell-Befehl "xfmc_test") entspricht bitgenau AT32s XMC-Peripherie und wird tatsächlich aus `SystemInit` heraus ausgeführt — dieser Adressraum ist aber laut N32G457-Datenblatt (S.15) explizit "Reserved" (kein Speicher-Controller dokumentiert). Plausibelste Erklärung: geteilte/portierte Referenz-BSP (evtl. ursprünglich für einen AT32-Referenzentwurf geschrieben und unverändert in die N32-Firmware übernommen); der Code liefe auf echtem N32G45x-Silizium ins Leere (Reserved-Bus-Bereich, vermutlich kein Hardfault, aber auch kein funktionierendes externes SRAM) oder deutet auf eine im öffentlichen Datenblatt nicht dokumentierte interne Testerweiterung hin. Dies bleibt ein ungeklärter Rest, ändert aber angesichts der drei unabhängigen bitgenauen Treffer (QSPI, PLL, GPIOF/G) nichts an der Gesamteinschätzung.

**Nicht abschließend geklärt:** genaue N32-Variante (452 vs. 455 vs. 457) — dafür wäre ein Abgleich von Pinzahl/Package-spezifischer Peripherie (z. B. CAN2, USB, DAC, zusätzliche UARTs) nötig, was im Rahmen dieser Aufgabe nicht mehr geprüft wurde.
