# Control-FW v151 (Venus E3) — Vollanalyse und Unterschiede zur Venus D v151

> 🇬🇧 English version: [VNSE3_Control_FW_v151_Full_Analysis.md](VNSE3_Control_FW_v151_Full_Analysis.md)

**Stand:** abgeschlossen 2026-10-02. Alle Aussagen wurden am Code dieses Images geprüft; offene Punkte stehen in §12.
**Binary:** `151_control_VNSE3-0_ems_app_0151_0928_094421.bin` (Archiv `Marstek FW Archiv/firmwares/VNSE3-0/Control/151/`)
**Referenz:** Venus D Control v151 (`151_control_VNSD-0_ems_app_0151_0929_094333.bin`), siehe [VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md](VNSD_Control_FW_Analyse_ems_app_0151_0929_094333_de.md); zweiter Verwandter: [VNSA_Control_FW_v1509_Vollanalyse_de.md](VNSA_Control_FW_v1509_Vollanalyse_de.md)
**Ghidra:** `/VNSE3/Control/Control_151_VNSE3-0_ems_app_0151_0928_094421.bin` (Projekt `Marstek_Venus`)
**Dekompilat:** `Dekompilate/VNSE3_Control_v151/` (nur lokal, nicht im Repo): 1927 Funktionen, 0 Dekompilierfehler, + `listing/` (Flash-Listing, `globals_and_registers.txt`)
**Modbus-Deskriptortabelle:** [../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v151.csv](../Modbus_RS485_TCP/VNSE3_Descriptor_Table_Decoded_v151.csv)

---

## 0. Zusammenfassung

- **Gleiche Codebasis wie die Venus D, gebaut ohne PV/MPPT-, Stack- und Parallel-Funktionen.** 93,4 % des Images sind nach Maskierung
  von Pointern und BL-Offsets byteidentisch zu D v151 (410 Segmente; ohne BL-Maskierung 80,2 %). Von den 1956 D-Funktionen sind 1845
  in der E3 befehlsidentisch, 49 sind dieselbe Funktion mit Änderungen, 24 liegen an anderen Adressen (verschoben oder reduziert, siehe §3)
  und **31 sind herauskompiliert**. Die E3 hat eine kleine eigene Funktion (ein Dirty-Bit-Wrapper, §4.6).
- **RAM-Layout weicht von der D ab** (anders als bei der Venus A): `.data` 12 016 B (D 12 032 B), `.bss` endet bei 0x20021DF7
  (D 0x2002200F), externes SRAM `.xmc_bss` 113 752 B (D 158 456 B). Die meisten Globals liegen 16 Byte tiefer als auf der D; der große
  Sprung im externen SRAM (−44 640 B) ist der Telemetrie-Historienring (144 Datensätze à 0x227 B statt 0x35D B, PV/MPPT-Felder entfernt).
  Namen wurden über Code-Alignment übernommen, nicht nach Adresse (§11).
- **Modbus-Registerkarte identisch zu D v151.** Gleiche 246 Deskriptoreinträge (30000–38014), gleiche Typen/Skalierungen/Anzahlen;
  nur die RAM-Quellpointer unterscheiden sich. `Write_Handler`, `FC03_Read_Handler` und `Read_Serializer` sind befehlsidentisch (§5).
- **Leistungsgrenzen unverändert (2500 W)**, aber der „set version“-Leistungswähler bietet jetzt **600 W** (neu, Modus 2), 800 W (Modus 1)
  und 2500 W (Modus 0); die 2200 W (Modus 3) der D entfallen (BLE-, MQTT- und JSON-RPC-Pfad).
- **Kein PV/MPPT, kein Batterie-Stack, keine Parallel-Wechselrichter:** PV-Felder in BLE-/MQTT-/Cloud-Nutzdaten, der
  Einspeisebegrenzer, die Überschuss-Einspeiseprüfung, die MQTT-Befehle Stack-BMS/BMS-Daten/MPPT (cd 0x28–0x2D), BLE-Befehle
  0x40–0x43, der CAN-Datum/Uhrzeit-Broadcast (0xCC) und die 0xC4-Schreibzugriffe sind entfernt; die Funktionen zur Erkennung/Synchronisation
  paralleler Knoten sind leer.
- **Zwei Schreibweisen der Modellkennung:** Laufzeit-Modell-/Typstring `VNSE3-0` (MQTT-Topics, Cloud, Shell-Benutzer `VNSE3-0 v151`,
  BLE-Name `MST_VNSE3_`), aber die OTA-/Boot-Deskriptor-Prüfung verwendet **`VNSEE3-0`**, und der Image-Footer lautet `VNSEE3`.

## 0a. Checkliste (E3 151 verglichen mit D 151)

| # | Frage | Antwort E3 151 | Beleg |
|---|---|---|---|
| 1 | Modbus-Register anders? | **Nein.** Deskriptortabelle Eintrag für Eintrag identisch (246 Einträge); `Write_Handler` @ 0x08054254 befehlsidentisch (46000 = 0x5100 Debug-Hintertür, 45000-Block, 47001 wie auf der D). Register 30200 liest das Versionswort @ 0x08010000 = **151**. Register, die aus PV/MPPT oder aus Pack 2–8 gespeist werden, haben auf der E3 keine Datenquelle. | §5 |
| 2 | BLE anders? | Gerätename `MST_VNSE3_` + 4 ID-Bytes. Befehl 0x15 Leistungswähler 600/800/2500 W (D: 800/2200/2500 W). Befehle 0x40–0x43 tun nichts. Befehl 0x03 Statusantwort 0x8B Byte (D 0xA7), Befehl 0x04 Infostring ohne `mppt_v`. | §4.1, §4.3 |
| 3 | AC-Grenzen? | Max. Lade-/Entladeleistung **2500 W** (`Config_Write_U16_0x202`, `Config_Write_U16`, `Config_ClampDefaults_ParamBlock` identisch zur D); neuer Wählwert 600 W; kein Einspeisebegrenzer (D: 4000-W-Klemme). | §4.1 |
| 4 | Firmware-Update per BLE möglich? | Ja, gleicher OTA-Code; die Modellprüfungen vergleichen mit **`VNSEE3-0`** (`OTA_Is_VNSE3_Model`, `OTA_Slot_Config_Validate`, `OTA_FW_Verify_SetStatus`, Boot-Deskriptor in `ProcessFirmwareUpdateCommand`). | §4.5 |
| 5 | Cloud-Domains? | Unverändert gegenüber D 151 (`api-<region>.marstekcloud.com` getDateInfo, marstekcloud data-upload, hamedata schedulings/policies/backup-configs). `getDateInfo` ohne `&mpptv=%d`; Upload-JSON ohne `pv/sk/mv/me/ma/fu`. | §4.4 |
| 6 | Pack-Kapazität? | Die Control behält die D-Strukturen: `Protocol_AA_CommandDispatch` akzeptiert Units < 9, 8 Pack-Datensätze à 0x60 B (`g_bmsPack0..7`, `g_bmsPack<k>Voltage` @ 0x200177C4 + k·0x60), Modbus-Pack-Blöcke 34000–34699 unverändert. Alles, was mehrere Packs zusammenfasst, wurde entfernt; die MQTT-Telemetrie enthält nur die 16 Zellspannungen von Pack-Datensatz 0. | §6a |

## 1. Binary-Fingerabdruck

| | E3 151 | D 151 (Referenz) |
|---|---|---|
| Größe | 378 880 B (0x5C800) | 393 216 B |
| SHA-256 | `a2f566ebdf3f94a5b2b171e071bd65237d0cd0ae68035dab5fc34734cf250148` | — |
| Linkadresse | **0x08004800**–0x08060FFF (18-KB-Bootloader darunter) | 0x08004800–0x080647FF |
| MCU | Nations **N32G45x** mit XFMC (Cortex-M4F), 144 KB SRAM | gleich |
| Toolchain | Keil ARMCC (`__main`/`__scatterload`, LZ77-komprimiertes RW) | gleich |
| Build-String | `Sep 28 2026 09:39:27` | `Sep 29 2026 09:42:27` |
| Version | `VNSE3-0 v151` (Shell-Root); Versionswort 0x97 (151) @ 0x08010000, gefolgt von `VNSEE3-0` | 0x97, `VNSD-0` |
| Image-Footer | `VNSEE3` + Nullbytes @ 0x08060FE0, davor 0xFF-Füllung ab 0x08060C10 | `VNSD` |
| Gerätetypcode | `g_deviceTypeCode` = 0x97 (gleicher Wert wie D) | 0x97 |

Basisnachweis: Die Vektortabelle bei 0x08004800 löst genau wie auf der D auf (Keil-Reset-Muster; `SystemInit` @ 0x080307A0 schreibt
`SCB->VTOR = 0x08004800`); Scatter-Tabelle, String- und Funktionspointer lösen auf dieser Basis auf.

## 2. Speicherkarte (Ghidra)

| Block | Bereich | Inhalt |
|---|---|---|
| bootloader | 0x08000000–0x080047FF | nicht im Image; 0x08001000 Bootloader-Versionswort, 0x08004000 Boot-Parameterseite |
| flash | 0x08004800–0x08060FFF | Applikation |
| sysinfo | 0x1FFFF7E0–0x1FFFF7FF | 96-Bit-UID @ 0x1FFFF7F0 |
| .data | 0x20000000–0x20002EEF | RW-Initialwerte (LZ77 @ 0x0805FED4, 2492 B) |
| .bss | 0x20002EF0–0x20021DF7 | nullinitialisiert; `ucHeap` @ 0x20003134 (80 KB); initialer SP 0x20021DF8 |
| sram_unused | 0x20021DF8–0x20023FFF | bis zur 144-KB-Grenze des N32G45x |
| .xmc_bss | 0x60000000–0x6001BC57 | externes SRAM über XFMC (LZ77-Image @ 0x08060890, 896 B) |
| Peripherie | apb1/apb2/ahb/Bit-Band/ppb | Registernamen aus der Nations-N32G457-SVD (1:1 von D 151 kopiert, gleiche Hardware-Karte) |
| xmc_reg / extflash_ctrl | 0xA0000000 / 0xA0001000 | XFMC- und QSPI-Controller |

Scatter-Tabelle `Region$$Table$$Base` @ 0x0805F994: .data 0x0805FED4 → 0x20000000 (0x2EF0, `__decompress` @ 0x08005FBE),
XMC 0x08060890 → 0x60000000 (0x1BC58), .bss Nullinit 0x1EF08 @ 0x20002EF0.

## 3. Verhältnis zur Venus D v151 (Methode und Zahlen)

| Kategorie | Funktionen | Bedeutung |
|---|---|---|
| befehlsidentisch | 1845 | gleiche Bytes bis auf Aufruf-/Pointerziele; Namen, Prototypen, Plates, Locals aus D 151 übernommen |
| geändert | 49 | gleiche Funktion, andere Konstanten oder anderer Code (§4) |
| andere Adresse / reduziert | 24 | z. B. `ProcessFirmwareUpdateCommand`, `Inverter_Power_Setpoint_Calc`, `BLE_Build_DeviceStatus_Response`, `Cloud_UploadTelemetryData`, `Config_LoadOrInit_DeviceSettings`, `Factory_Reset`, `Energy_TOU_Counters_Update`, `MQTT_Telemetry_String_Formatter`, `Quectel_UART_Send_Data`, `TLS_SSL_Read_WithTimeout`, die Stubs; in der E3 leer: `CAN_Detect_Mismatched_Nodes`, `CAN_Parallel_Inverter_Sync`, `Inverter_ClearReg_0x03_Default` |
| fehlt in der E3 | 31 | siehe §4.6 |
| nur in der E3 | 1 | `Inverter_RegDirty_Mark_BleState` (+ 4 umbenannte Wrapper, §4.6) |

Sechs D-Namen, die die Offline-Paarung der falschen E3-Funktion zugeordnet hatte, wurden nach dem Review korrigiert (z. B. ist
0x0802B69C `Config_LoadOrInit_DeviceSettings` und nicht das entfernte `CAN_Send_ClearCmd_0xCB_DLC2`; 0x08009F48 sendet 0xFB, also
`Inverter_Set_Reg_0xFB`). Methode: Offline-Bytesegmente mit maskierten Pointern/BL-Offsets, Paarung über Einsprungadressen,
namensnormierte Befehlsdiffs und Dekompilat-Review aller geänderten und verschobenen Funktionen.

## 4. Funktionale Unterschiede zur Venus D v151

### 4.1 Leistungsgrenzen und Leistungswähler

| Stelle | E3 151 | D 151 |
|---|---|---|
| `Config_Write_U16_0x202` (max. Ladeleistung, EEPROM 0x202) | Klemme 2500 W | 2500 W |
| `Config_Write_U16` (max. Entladeleistung, EEPROM 0x204) | Klemme 2500 W | 2500 W |
| Defaults in `Config_LoadOrInit_DeviceSettings` | 2500 W | 2500 W |
| BLE-Befehl 0x15 / MQTT cd 0x0F / JSON-RPC „set version“ | **600 W → Modus 2 (neu)**, 800 W → Modus 1 (+ Zeitplan-Klemme auf 800 W), 2500 W → Modus 0 | 800 → 1, **2200 → 3**, 2500 → 0 |
| BLE 0x16 / 0x17 | 300–2500 W / ≤ 2500 W | gleich |
| Einspeisebegrenzer (`Grid_Export_Power_Limiter`, 4000-W-Klemme) | **nicht vorhanden** | vorhanden |
| `Inverter_Power_Setpoint_Calc` | kein Schritt `Battery_Charge_Power_Limiter`, keine PV-Überschuss-Addition, keine Zwangslade-Klemme; verbleibende Klemmen: Fehlerbit 0, max. Entladen / −max. Laden, Grenzen aus `Power_ScaledValue_Calc`, Entlade-Abschalt-SOC, EPS-Reserve, 3-fach-Wiederholungs-Entprellung | vollständige Kette |

### 4.2 Batterie / BMS

- Ein-Pack-Sicht: `BMS_CellVoltage_MinMax_PerPack`, `BMS_Version_Consistency_Check` (kopiert nur noch die Version von Pack 0),
  `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version` und die MQTT-Stack-Befehle fehlen (§6a).
- `CAN_CommLink_Timeout_DataClear` überwacht nur noch die BMS-Verbindung (1) und die Wechselrichter-Verbindung (2); die MPPT-Verbindung 0 wird übersprungen.

### 4.3 Unterschiede bei BLE- und MQTT-Befehlen

- **BLE** (`BLE_App_Command_Dispatcher` @ 0x0800C0E4): Befehle 0x40 (`BLE_Apply_ConfigReg_Offset3`), 0x41 (Überschussstrom),
  0x42 (BMS-Datenantwort) und 0x43 sind leer; Befehl 0x03 Statusantwort 0x8B Byte (`BLE_Build_DeviceStatus_Response` ohne PV1–4,
  PV-Kapazität, MPPT-Version und Pack-Anzahl-Felder); Befehl 0x04 Infostring ohne `,mppt_v=%d`; Entwicklermodus-Untercode 5 tut nichts.
  Es gibt in der E3 keinen Befehl, den die D nicht hat.
- **MQTT ctrl** (`MQTT_CtrlCommand_Dispatcher` @ 0x08022A20): cd 0x28 (Stack-BMS setzen), 0x29 (BMS-Version), 0x2A (BMS-Daten),
  0x2B (Überschussstrom), 0x2C (BMS sperren), 0x2D (MPPT-Daten) entfernt; `MQTT_Config_Command_Handler` kennt den Schlüssel `fu=` nicht mehr.
- **JSON-RPC** (`MQTT_JSON_RPC_Dispatcher`): Methode 5 (PV-Daten) antwortet mit „Method not found“.
- **Status/Telemetrie:** `MQTT_StatusReport_BuildAndPublish` lässt den Block `,mppt=…,pv1..pv4,…,pack=…,fu=…,em=%x` weg;
  `MQTT_Telemetry_String_Formatter` lässt PV-String, Pack-Fehlerübersicht und Einspeiseflags weg und sendet 16 Zellspannungen von
  Pack 0 statt der Min/Max-Statistik von 6 Packs der D; `MQTT_Publish_PowerSchedule` veröffentlicht p1/p2 mit bis zu 30 Einträgen
  (D: p1–p7, 15 Einträge); die 1-s-Leistungshistorien enthalten nur noch Netz- und Off-Grid-Leistung (je 29 Werte).

### 4.4 Cloud / HTTP

| Funktion | E3 151 | D 151 |
|---|---|---|
| `HTTP_GetUrlData`, `HTTP_Cloud_Reporting_Dispatcher` | `http://api-%s.marstekcloud.com/app/neng/getDateInfo%s.php?...&boot=%d&inv=%d` | gleich plus `&mpptv=%d` |
| `Cloud_UploadTelemetryData` (`{"d":"di=…"}`) | Schlüssel `pv, sk, mv, me, ma, fu` entfernt; Datensatz-Schrittweite 0x227 | Schrittweite 0x35D |
| `Cloud_Report_FillDeviceInfo` | Gerätename `VenusE 3.0` | `Venus D` |
| `Cloud_Report_FillGridData` / `FillPowerFlow` | PV-Leistungen entfernt (Feld wird 0 geschrieben), statt PV-Summe wird die Batterieleistung abgelegt | PV1–4 |
| `Energy_TOU_Counters_Update` | 2 Zähler (`"%d,%d"`) | 3 Zähler |

Domains, Pfade und TLS-Code sind unverändert; es gibt keinen E3-spezifischen Cloud-Endpunkt.

### 4.5 Identität und OTA

- Laufzeit-Strings: `VNSE3-0` (`g_deviceModelTag`/`g_deviceTypeStr` = `VNSE` + `3-0`), `VNSE3-0 v151`, `MST_VNSE3_`, `VenusE 3.0`,
  Geräte-Quell-ID `VNSE3-0-<BLE-MAC-ID>`.
- OTA / Boot: `OTA_Is_VNSE3_Model` (D `OTA_Is_VNSD_Model`), `OTA_Slot_Config_Validate` und `OTA_FW_Verify_SetStatus` akzeptieren ein Image,
  dessen Typstring ein **Teilstring von `VNSEE3-0`** ist (`strstr("VNSEE3-0", empfangen)`, Fehlercodes 0x401 CRC / 0x402 Gerätemaske /
  0x403 Typ unverändert). `ProcessFirmwareUpdateCommand` schreibt den Boot-Deskriptor mit dem Tag `VNSEE3-0` (Versions-Tag bis `-`
  kopiert, max. 9 Zeichen; D max. 7). Das ist die im E3-Changelog erwähnte „Typprüfung“; derselbe Mechanismus existiert auf D 151 mit `VNSD-0`.
  Der YMODEM-Update-Pfad (`YModem_*`, `CLI_FirmwareUpdate_Prepare`) ist befehlsidentisch zu D 151.
- „AI-Modus“: `AI_ScheduleQuery_Response_Send` (cd 58, nur bei `g_userWorkMode == 5`) ist vorhanden und identisch zu D 151.
- „CT 1,1 s“: Ein E3-spezifisches CT-Timing wurde nicht gefunden. Der CT-/Zähler-Code ist identisch zu D 151; die einzige Konstante
  von 1100 Ticks ist das Status-Telemetrie-Intervall in `Modem_/Ethernet_SendStatusTelemetry`, das D 151 ebenfalls hat.

### 4.6 In der E3 fehlende (herauskompilierte) Funktionen und E3-Wrapper

| D-151-Funktion(en) | Zweck auf der D |
|---|---|
| `Key_ProtectionMonitor`, `Board_ProtectionStage_Sequencer/Dispatch/Controller`, `Board_ProtectionBand_Monitor`, `Board_ProtectionLatch_TimedConfirm/ConfirmStep`, `Board_ProtectionFlag_Toggle`, `WDT_ProtectionTimeout_Reboot`, `CAN_ParallelInvLink_DebounceHold`, `Board_GPIO_Init_ControlPins` | Taster-/Schutzrelais-Kette an GPIOB.10/.11, GPIOE.15 (gleiche Menge wie bei der A fehlend) |
| `Grid_Export_Power_Limiter`, `Grid_Export_Limit_Periodic_StateMachine`, `Surplus_FeedIn_Enable_Check`, `Battery_Charge_Power_Limiter` | Einspeisebegrenzer und Überschuss-Einspeisung |
| `Inverter_PVString_ChannelValue_PeriodicCheck_Save`, `Cloud_Report_FillPvStringDetails`, `MQTT_BuildPvDataResponse`, `Grid_UpdateMpptFaultLatch`, `MQTT_Publish_Inverter_Telemetry` | PV/MPPT |
| `BMS_CellVoltage_MinMax_PerPack`, `MQTT_Publish_BMS_Data`, `MQTT_Publish_BMS_Version`, `BLE_Build_Cmd0x42_Response` | Mehr-Pack-Berichte |
| `CAN_Broadcast_DateTime`, `Config_Write_Category0xC4`, `Config_Write_ParamPair`, `CAN_Send_ClearCmd_0xCB_DLC2`, `Inverter_Set_Reg_0x04`, `Inverter_ClearReg_0x03`, `BLE_Apply_ConfigReg_Offset3` | CAN-Befehle, die die E3 nicht sendet (§6) |

Die „Dirty-Register“-Bitmap des Wechselrichters hat auf der E3 ein anderes Layout (0x1C Bits, 7 pro Byte, statt 0x18 Bits / 6 pro Byte,
`Inverter_RegDirty_Bitmap_SetClear`); die Feld-Wrapper wurden nach ihren E3-Aufrufern benannt: `Inverter_RegDirty_Mark_BleState`
(0x1A, nur E3), `_GridPermit` (Bit 2), `_MeterInvalid` (0x16), `_NetworkLink` (0x18), `_ChgVolt` (Bit 4, D 0x14).
`Grid_BuildStatusFlagsReg` sendet einen 8-Byte-Statusrahmen aus Nullen an CAN-ID 0x1801FFAA (D: MPPT-Fehler-Latch, PV-Überspannung,
Abschalt-SOC).

## 5. Modbus

- **Lesebereich:** `g_modbusRegDescTable` @ 0x20000340 (D 0x2000034C), 246 × 12 B; Registernummern, Typen, Skalierungen, Elementgrößen
  und Anzahlen sind identisch zu D 151; 213 Quellpointer liegen 16 Byte tiefer, die übrigen folgen den anderen RAM-Verschiebungen
  (−444/−532/−533, +44, Flash/keiner unverändert). Die dekodierte CSV nennt in der Spalte `notes` das RAM-Global des E3-Programms.
  `FC03_Read_Handler` @ 0x080221B4 und `Read_Serializer` @ 0x08053154 sind befehlsidentisch, die bekannten D-Fehler (Vorzeichen 32101,
  MBAP-Länge von Exception-Frames) gelten also auch hier.
- **Schreibbereich:** `Write_Handler` @ 0x08054254 befehlsidentisch (45000-Befehlsblock, 46000 = 0x5100 Debugmodus → Shell auf RS485 und
  TCP 8091, 46500–46544 Zeitplan (CAN 0x30–0x3E), 47001 (CAN 0x18)).
- **E3-spezifische Register:** keine. Anders ist nur die Datenquelle: Register, die die D vom Class-2-MPPT-Knoten füllt
  (z. B. 30028, 30036, MPPT-Version), haben auf der E3 keine Quelle (keine MPPT/PV-Hardware, die MPPT-Verbindung wird nicht mehr überwacht);
  Pack-Blöcke von Packs, die nie berichten, bleiben 0; Register 30200 liest 151.

## 6. Interner CAN-Bus (Control-Seite)

Arbitrierungs-ID: `CAN_Build_Arbitration_ID` @ 0x08034F60 (Funktionscode Bits 0–7, Knoten 8–11, Zielklasse 12–15). 51 Sendestellen
(D: 57). Empfangsseite identisch: `CAN_FrameDispatcher` @ 0x08031420 → Quellklasse 4 `Telemetry_Register_Dispatcher`,
Klasse 2 `Modbus_ResponseDispatch`, sonst `Protocol_AA_CommandDispatch` @ 0x08031D04.

| Von der E3-Control gebaute Funktionscodes | Hinweise |
|---|---|
| 0x01 Leistungssollwert (`Inverter_Power_Setpoint_Calc`, 4 B) und Ein/Aus, 0x02 Modus-/Leistungsbefehl, 0x03/0x04 u32-Schreibzugriffe, 0x05, 0x06, 0x07, 0x08, 0x0A, 0x10–0x14, 0x16, 0x17, 0x18, 0x30–0x3E Zeitplan (über `Write_Handler`), 0x40–0x43, 0x50–0x58, 0x60, 0xC1–0xC3, 0xCB (nur DLC4), 0xCE (Knoten 1 fest), 0xCF, 0xFB, 0xFE, 0xFF | gleiche Sendefunktionen wie D 151 |
| **nicht mehr gesendet** | 0xCC Datum/Uhrzeit-Broadcast, 0xC4-Kategorie-Schreibzugriffe, 0xCB DLC2, 0x04 DLC2 (`Inverter_Set_Reg_0x04`), 0x03 DLC2-Löschen |

Der E3-Micro (Micro 119, STM32G474, gleiche Familie wie der D-Micro) ist der Empfänger der Class-4-Befehle; welche Codes er tatsächlich
verarbeitet, bestätigt die E3-Micro-Analyse. Der CAN-Datum/Uhrzeit-Broadcast fehlt, der E3-Micro erhält also über CAN keine Uhrzeit von
der Control. `Register_WriteCategory0xCE` adressiert immer Knoten 1, und `CAN_ParallelInverterDataParser` hält pro Bank nur einen Wert
(kein Unit-Index).

### 6a. Pack-Kapazität (Standardprüfung 6)

| Punkt | E3 151 | D 151 |
|---|---|---|
| Unit-Grenze in `Protocol_AA_CommandDispatch` | `(id & 0xFF) < 9` (identischer Code) | `< 9` |
| Pack-Datensätze | 8 × 0x60 B (`g_bmsPack<k>Voltage` @ 0x200177C4 + k·0x60; der 0x300-Byte-Block wird bei BMS-Verbindungs-Timeout gelöscht) | 8 × 0x60 B (`g_bmsPack0Voltage` @ 0x200177D4) |
| Modbus-Pack-Blöcke | 34000–34699, identische Deskriptoren | gleich |
| Mehr-Pack-Auswertung | entfernt (Zell-Min/Max pro Pack, Versionskonsistenz, MQTT-Befehle BMS-Daten/Version/Stack, BLE-Befehl 0x42) | vorhanden |
| Zellen pro Pack | 16 (MQTT-Telemetrie gibt 16 Zellspannungen von Datensatz 0 aus) | 16 |

Die Control hat also weiterhin Platz für 8 Units, wertet aber nicht mehr als ein Pack aus; wie viele Packs das E3-BMS wirklich meldet
(Grenze in `CAN_TX_PerPack_10Msgs`), entscheidet die Analyse von E3 BMS 115.

## 7. Tasks, Shell, Protokolle

Bis auf Adressen identisch zu D 151: Task-Tabelle `g_taskDefTable` @ 0x080584B4 (14 Tasks, gleiche Prioritäten/Stacks),
`shellCommandTable` @ 0x0805F9C4 (81 Einträge, Benutzer `VNSE3-0 v151` mit leerem Passwort), `g_mqttRpcMethodTable` @ 0x0805B6DC
(16 Methoden), `mbedtls_cipher_definitions` @ 0x0805B834, CH395-Sockets (502 Modbus TCP, 8091 Shell, 8090 UDP).

## 8. EEPROM-Karte

Gleiche Adressen wie D 151 (`Config_LoadOrInit_DeviceSettings` liest/schreibt dieselben Blöcke: 1 Identität, 0x200, 0x300, 0x900,
0x36BD, 0x500, 0x395, 0x400 WLAN, 0x441 Servertyp, 0x484/0x488/0x492 TOU-Zähler). Der Identitätsblock enthält jetzt `VNSE3-0`.

## 9. Interessante Strings

`VNSE3-0 v151`, `VNSEE3-0` (OTA-Typ-Tag), `MST_VNSE3_`, `VenusE 3.0`, `Build:\tSep 28 2026 09:39:27`,
`dev_type=%s, bin_type=%s`, die `getDateInfo`-URL ohne `mpptv`, das gekürzte Upload-Format `{"d":"di=…"}`. Die WLAN-Standardzugangsdaten
des Herstellers (`SM471`, …) und `quit_debug_5100_` sind wie auf der D vorhanden.

## 10. Sicherheitshinweise

Alle Control-Befunde von D 151 gelten unverändert (identischer Code): Modbus 46000 = 0x5100 → nicht authentifizierte Shell auf RS485 und
TCP 8091, Shell-Benutzer mit leerem Passwort, WLAN-Standardzugangsdaten des Herstellers, `getDateInfo` über unverschlüsseltes HTTP.
Die OTA-Typprüfung verwendet `strstr("VNSEE3-0", empfangener_typ)`, d. h. jeder Teilstring der Konstante (auch ein leerer String) besteht;
dieselbe Argumentreihenfolge gibt es auf D 151.

## 11. Bearbeitungsstand in Ghidra

- Abdeckung: 0 undefinierte Bytes, 0 verwaiste Befehle im Flash-Block; 1927 Funktionen, alle benannt, alle mit Prototyp
  (Task-Einsprünge `void f(void *pvParameters)`); 13 Sprungtabellen-Overrides, keine nicht aufgelösten Sprungtabellen.
- `marstek_gates.py`: **alle Gates bestanden** (kein Code-als-Daten, keine Nur-Literal-Funktionen, keine FUN_/tmp_/Duplikate, 0 unbenannte
  referenzierte RAM-Globals, Tasknamen passen zur Task-Tabelle, No-Return-Flags plausibel); Dekompilat-Export 0 Fehler.
- Tags: `NAME_VERIFIED` an allen 1927 Funktionen (Kommentar „Name, prototype and comment verified against the VNSE3 Control 151 code“),
  davon 94 zusätzlich `NAME_LOW_CONFIDENCE`.
- RAM-Globals wurden über Befehls-Alignment gepaarter Funktionen (Mehrheitsentscheid) plus Interpolation zwischen Ankern übernommen und gegen
  eine Offline-Pointerkarte geprüft; Abweichungen wurden von Hand korrigiert (z. B. OTA-Fehlercode-Slots, Layout der Modell-/Typstrings,
  Felder der Telemetrie-Historie). Peripherie- und XFMC-Labels wurden 1:1 kopiert (gleiche Hardware-Karte). Kommentare mit D-spezifischem
  Inhalt (VNSD-Strings, D-Flash-/RAM-Adressen) wurden für die E3 umgeschrieben.

## 12. Offene Fragen

1. Welche Class-4-Befehle verarbeitet der E3-Micro (Micro 119), und erwartet er noch das CAN-Datum/Uhrzeit (0xCC), das die E3-Control nicht mehr sendet?
2. Wie viele Packs meldet das E3-BMS 115 (Grenze des Pack-Senders)? Die Control würde bis zu 8 Units annehmen.
3. Bedeutung der umsortierten Felder der Dirty-Bitmap (Bits 2, 0x16, 0x18, 0x1A): Die Namen folgen den E3-Aufrufern; das Register hinter jedem Bit braucht den E3-Micro.
4. Warum zwei Modell-Schreibweisen (`VNSE3-0` zur Laufzeit, `VNSEE3-0` für OTA/Boot)? Der eigene Footer dieses Images (`VNSEE3`) besteht die Prüfung,
   ein Image mit dem Tag `VNSE3-0` würde aber abgelehnt (`VNSE3-0` ist kein Teilstring von `VNSEE3-0`); ob alle anderen offiziellen E3-Images den
   Footer `VNSEE3` tragen, ist nicht geprüft.

## Anhang: Methode

Offline-Segmentkarte der beiden Images (maskierte Pointer und BL-Offsets), `ve3_layout_sync.py` / `ve3_port_segments.py` /
`ve3_port_signatures.py` / `ve3_copy_by_address.py` (nur Peripherie) / `ve3_port_refs.py` (nur Flash-Ziele) /
`ve3_port_globals_aligned.py` / `ve3_ram_interp_port.py`, um Layout, Namen, Prototypen, Referenzen und RAM-Globals von D 151 zu übernehmen,
danach Review aller geänderten und verschobenen Funktionen, Lückentypisierung und die Prüf-Gates. Hilfsskripte sind nur lokal vorhanden (nicht im Repo).
