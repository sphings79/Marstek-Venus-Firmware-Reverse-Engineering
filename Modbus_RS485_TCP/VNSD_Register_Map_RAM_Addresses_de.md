# VNSD-Registerkarte – RAM-Adressen je Firmware-Version (und Schreibeffekte)

> 🇬🇧 English version: [VNSD_Register_Map_RAM_Addresses.md](VNSD_Register_Map_RAM_Addresses.md)

`VNSD_Register_Map_Final_generated.csv` enthält zu jedem Modbus-Register die RAM-Adresse dahinter („Quellzeiger"). Diese Adressen
wandern mit jedem Firmware-Build (RAM-Layout ändert sich: v149.2 hat den BMS-Pack-Record vergrößert, v151 hat den ganzen
Block um +0x2800 verschoben). Die frühere einzelne Spalte `source_ptr_v150` wurde deshalb auf alle VNSD-Control-Versionen erweitert:

| Spalte | Firmware |
|---|---|
| `source_ptr_v147` | Control 147 |
| `source_ptr_v149` | Control 149 |
| `source_ptr_v1492` | Control 149.2 (Beta) |
| `source_ptr_v150` | Control 150 (Werte unverändert) |
| `source_ptr_v151` | Control 151 |
| `source_ptr_basis` | wie die Adressen dieser Zeile ermittelt wurden |
| `status_D` | Funktionsstatus an der Venus D (Registeraufloesung 2026-10-01): **FUNCTIONAL** = hat die dokumentierte Wirkung · **CONDITIONAL** = nur in einem Sondermodus / nach einem anderen Schreibzugriff · **NO_OP** = wird angenommen, hat aber keine Wirkung · **FUNCTIONLESS** = Ziel existiert, wirkt aber nie (gefiltert, nie gelesen, kein Empfänger) · **UNKNOWN** = Ziel gefunden, Bedeutung nicht belegbar |
| `ziel_D` | Ziel des Registers (Gerät: was erreicht wird, oder `none`) |
| `leseeffekt_D` | Wirkung/Verhalten eines **Lesezugriffs** (z. B. Exception 2, hängende TCP-Verbindung) |
| `befund_D` | Befund samt Verifikationshinweisen |
| `gefahr_D` | Gefahreneinstufung und Grund (`none` / `low` / `medium` / `high`) |
| `suite_empfehlung_D` | Empfehlung für die Marstek-Modbus-Suite-Integration (z. B. nicht anbieten) |
| `befund_konfidenz_D` | Konfidenz des Befunds (`high` / `medium`) |
| `schreibeffekt` | was ein **Schreibzugriff** auf dieses Register bewirkt: gültige Werte, Nebenwirkungen (EEPROM, CAN-Frame an Wechselrichter/BMS, WLAN-Neuinitialisierung, Neustart/Werksreset, Moduswechsel), Exception-Fälle, Leseverhalten; für alle 170 schreibbaren Register (40000+) gefüllt, basierend auf dem v151-`Write_Handler` |

Zellwerte: `0x2001…` = Adresse; `—` = Register existiert in dieser Version nicht (z. B. 38000–38014 vor v149);
leer = nicht bestimmbar (siehe unten).

## Wie die Adressen ermittelt wurden
* **`descriptor`** (246 Register 30000–38014, in v147 231): direkt aus der Modbus-Descriptor-Tabelle im initialisierten
  `.data`-Abbild jeder Firmware dekodiert (12-Byte-Einträge: Register, Quellzeiger, Typ, Skalierung, Anzahl). Für v149.2, v150 und v151
  stimmt das 246/246 mit den vorhandenen Tabellen (`VNSD_Descriptor_Table_Decoded_*.csv`) überein; die v150-Spalte ist identisch
  mit der bisherigen Spalte.
* **`symbol+offset`** (Schreib-/Kommandoregister 40000+, 41xxx, 42xxx, 43xxx, 44xxx, 45xxx und die Zugangsdaten-Puffer): die v150-Adresse
  wurde im Ghidra-Projekt v150 auf das benannte Global (plus Byte-Offset) aufgelöst und in den anderen Versionen unter demselben Namen
  nachgeschlagen. Die Methode wurde an den Descriptor-Einträgen validiert: 245 von 246 Einträgen korrekt aufgelöst für v149.2 und v151 (ein Eintrag ohne Symbol), 235 von 246 für v149
  und 227 von 231 für v147 (der Rest hat dort keinen eindeutigen Symbolnamen); kein falsches Ergebnis.
* **`symbol(near)+offset`**: wie oben, aber die v150-Adresse liegt in einem Array/Struct, das nur ein Unter-Symbol hat; etwas unsicherer.
* **`table-base+offset`** (46500–46544): die 120-Byte-Tabelle, die in v150/v149.2/v149 `g_powerLimitTable` heißt, ist in v151
  `g_canScheduleRegCache` (ebenfalls 120 Byte). Der Name `g_powerLimitTable` ist ein veraltetes Label aus der alten Analyse und als
  CAN-Schedule-Cache zu lesen. In v147 konnte die Tabelle nicht gefunden werden (leere Zellen).
* Teilregister von Mehrregisterwerten (`teil_von` gesetzt, z. B. MAC-Zeichen) haben wie bisher in keiner Spalte einen eigenen Zeiger.

## Vorbehalte
* **Neuprüfung 2026-10-01 (v151-Code, ausgehend von der Registernummer):** die Zuordnungen aller Schreib-/Kommandoregister (40000+) und der
  Descriptor-Register mit Platzhalternamen wurden gegen den v151-`Write_Handler` (`0x080570e0`) und die Descriptor-Tabelle geprüft. Die Zeiger
  waren richtig; 113 Register-**Namen** nicht, sie wurden in der CSV korrigiert (alter Name in `alt_name` als `frueher: …`, Begründung in `notiz` als
  `KORR 2026-10-01`). Hauptbefunde: 41600–41631 ist der schreibgeschützte WLAN-**Passwort**-Puffer (nicht `schedule_power`); 46500–46544 ist der
  CAN-Schedule-Register-Cache (`g_canScheduleRegCache`), keine EEPROM-Struktur / Leistungsgrenzen-Tabelle; 40000 ist der Burn-in-Teststep (kein
  RS485-Unlock); 42000 = RS485-Steuermodus (teilt sich das Byte `g_userWorkMode` mit 43000); 37002/37003 sind nicht max. Lade-/Entladeleistung (das
  sind 44002/44003); 30110/32114 = `g_encodeFlag` (Status der Schlüssel-/Pairing-Sequenz); mehrere 45xxx-Kommandonamen waren falsch (siehe
  `VNSD_Command_Register_45000_Block_v150_de.md`, Abschnitte 7 und 8). Die Korrekturen in `notiz` sind englisch geschrieben, die alten „SRAM-Ziel"-Adressen
  darin wurden durch den Wert der v150-Spalte ersetzt. Nach der Registeraufloesung vom 2026-10-01 bleibt nur 1 Register (37021) `UNKNOWN`; 45021, 45028, 45030, 30035 und 47001 sind geklärt (47001 steht jetzt in der CSV). In diesem Durchgang wurden bei 170 Zeilen die Namen korrigiert (alter Name in `alt_name` als `frueher: <alt>`); Details in `VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md`.
* Die `v_*`-Spalten (Live-Scanwerte) und die Notizen beziehen sich weiter auf Firmware 150.
* `VNSD_Register_Map_v150_annotiert.csv` und `VNSD_Cross_Reference_Modbus_SRAM_BLE_v150.md` enthalten weiter nur v150-Adressen.

## Spalten der Registeraufloesung (2026-10-01)
Die Spalten `status_D`, `ziel_D`, `leseeffekt_D`, `befund_D`, `gefahr_D`, `suite_empfehlung_D` und `befund_konfidenz_D` enthalten das Ergebnis der vollständigen Codeanalyse der früher unklaren Register (179 Register gefüllt: FUNCTIONLESS 71, FUNCTIONAL 60, CONDITIONAL 39, NO_OP 8, UNKNOWN 1). Bedeutung der Statuswerte: siehe Zeile `status_D` in der Tabelle oben. Details: [VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md](VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md).

Neue Zeile **41001** (`unhandled_register_41001`, früher als `factory_reset (Suite button)` bezeichnet): **an der Venus D nicht implementiert** – in keiner Control-Version gibt es einen 41001-Case, das Register antwortet daher mit Exception 2 (das gleich nummerierte Micro-Register liegt in einem getrennten Registerraum). Ein echter Werksreset ist 41000 = `0xAA11`.

## Neues Register 47001
Register 47001 (`0xB799`, `can_cat0x18_command`) gibt es erst ab v151 (ältere Versionen antworten mit Exception 2); es steht mit `—` in allen Zeigerspalten in der CSV.
Hinweis zu gefährlichen Schreibzugriffen (siehe `schreibeffekt`): 45028 löst einen BMS-Werksreset + Neustart aus, 41000 = `0xAA11` einen Werksreset der Control-Einheit, 41000 = `0x55AA` einen Neustart. **Korrektur 2026-10-01:** 45001 löst **keinen** BMS-Reset aus (es ist ein Micro-EEPROM-Selbsttest); 45028 schon (Werksreset + Neustart aller Packs), und 45027 startet das BMS-Master-Pack neu.
