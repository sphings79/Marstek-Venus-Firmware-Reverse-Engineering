# Venus A – BMS-Platine, Fotos

> 🇬🇧 English version: [VNSA_BMS_Board_Photos.md](VNSA_BMS_Board_Photos.md)

43 Fotos eines Venus-A-Akkupacks und seiner Stecker (EXIF-/GPS-Metadaten entfernt). Die BMS-Platine ist **nicht vergossen**, Bauteile und Aufdrucke sind also lesbar,
anders als beim Venus D (siehe [VNSD_Teardown_Fotos_de.md](../Marstek%20Venus%20D/VNSD_Teardown_Fotos_de.md)).
MAC-Adressen und Geräte-IDs sind nicht zu sehen. Das einzige sichtbare Seriennummern-Etikett (Gehäuse-Aufkleber auf Foto 28) ist geschwärzt.

| Dateien in diesem Ordner | Anzahl | Inhalt |
|---|---|---|
| `VNSA_BMS_Board_01…27_<Inhalt>.jpg` | 27 | BMS-Platine, beide Seiten |
| `VNSA_BMS_Board_28…38_<Inhalt>.jpg` | 11 | Weitere Nahaufnahmen (MCU, AFE, Schalter-/LED-Stecker J7, J1, TVS-Bauteile), per GitHub-Issue #4 beigesteuert |
| `VNSA_Head_Unit_01_<Inhalt>.jpg` | 1 | CAN-Stecker J14 auf der Platine des Kopfteils, Issue #4 |
| `VNSA_Pack_Connector_01…04_<Inhalt>.jpg` | 4 | 12-poliger Gehäusestecker zwischen Pack und Kopfteil/Erweiterung, Issue #5 |

## Platinen-Identifikation

Siebdruck: `HAME`-Logo, `13S50AH_BMS_V1.1 venus A`, `2025.09.08`; Basismaterial-Aufdruck `E336650`, Datumscode `2538`.
„13S“ passt zu den 13 Zellabgriffen (B0…B13). In diesem Dokument heißt „MCU-Seite“ die Seite mit MCU, Steckern und den großen Schraubklemmen,
„Leistungsseite“ die Gegenseite mit Pads, Shunts, MOSFETs und sicherungsartigen Bauteilen. Beide Seiten sind bestückt.

Ein öffentlicher Forenthread über ein geöffnetes Venus A (photovoltaikforum.com, Thread 265074) zeigt denselben Platinentyp `13S50AH_BMS_V1.1 venus A` auf einem anderen Pack mit
dem Siebdruck-Datum `2025.08.22`; es gibt also mindestens zwei Baudaten der V1.1-Platine (`2025.08.22`, `2025.09.08`).
Laut Forum ist die Pack-Platine von Hauptgerät und Zusatzakku identisch, die Firmware aber unterschiedlich.

## Identifizierte Bauteile

| Ref. | Bauteil | Beleg | Fotos |
|---|---|---|---|
| U3 | **Nations N32G455VEL7** (MCU, ca. 100-Pin-LQFP, Lot `TBE12014`) | Aufdruck in voller Auflösung gelesen: `Nation` + Logo, `N32G455`, `VEL7`. Bestätigt die N32G45x-Familie im BMS (bisher nur aus der Firmware abgeleitet). Das Suffix wurde nicht entschlüsselt. Auf den Fotos 29/30 ist der Aufdruck teilweise abgerieben, weitere Stellen waren nicht lesbar. | 11, 12, 29, 30 |
| U5 | **Sinowealth `SH36735?4U`** (Zellüberwachungs-AFE, 48-Pin-LQFP, Datumscode `2507`) | Aufdruck (Sinowealth-Logo und -Name, `SH36735…`, `2507-…`) in voller Auflösung gelesen; die letzten Stellen bleiben auf allen fünf Fotos uneindeutig. Die Firmware nennt diesen AFE `SH36735` (SPI1, 13 Zellen). | 13–16, 31–33 |
| Y1 | Quarz 8,000 MHz | Aufdruck `8.000` | 11, 12, 30 |
| T1 | Kleiner Trennübertrager (goldfarbenes Gehäuse, vier Pins pro Seite) | Sitzt an der Isolationslinie neben dem Stecker J1; Aufdruck oben gespiegelt/unleserlich | 21, 22, 36, 37 |
| U8, PC1 | Optokoppler, Aufdruck `817B` | Aufdruck | 20, 24, 25 |
| U20, U22 | SOT-223-Regler | Nur Bauform, Aufdruck nicht lesbar | 24, 25 |
| U2, U4, U21, U15, U16, U17, U11 | SOIC-8-Chips | **Aufdrucke nicht lesbar**, Typ unbekannt (U21 sitzt zwischen J7 und T1 bei R213…R216 und C102…C105; der Aufdruck ist auf den Fotos 36, 37 sichtbar, aber nicht entzifferbar). U2/U4 neben der MCU sind möglicherweise das in der Firmware genannte I²C-EEPROM (ungeprüft). | 11, 12, 21, 22, 36, 37 |
| – | Goldfarbenes Pufferelement mit weißer Lasche neben der MCU | Vermutlich Supercap/Pufferzelle (ungeprüft) | 1, 11 |
| SW1 | Taster | Funktion offen, siehe [Offene Punkte](#offene-punkte) | 8, 10 |
| D1 | SMD-LED (durchscheinender Körper) neben der MCU, mit `R14` (`102` = 1 kΩ) und `R9` (`103` = 10 kΩ) | Nur am Körper erkannt; die Funktion (Status-LED oder etwas anderes) ist abgeleitet | 29, 30 |
| R25… | Reihe aus sechs kleinen Widerständen zwischen U5 und U3: fünf mit Aufdruck `101` (100 Ω), einer mit `103` (10 kΩ) | Aufdruck; vermutlich Serien-/Pull-Widerstände auf den AFE–MCU-Leitungen (abgeleitet, Leiterbahnen nicht verfolgt) | 31, 32, 33 |
| Q1…Q17 | Kleine Transistoren in einer Reihe neben 13 Widerständen | Balancer-Netzwerk für 13 Zellen, mit `NTC`-/`B`-Abgriffen | 17, 19, 20 |
| RS1…RS10 | Zehn SMD-Shunt-Widerstände in einer Reihe | Strommessung im B--Pfad (Leistungsseite) | 23–27 |
| FU1…FU3 | Drei größere schwarze SMD-Bauteile im P--Pfad | Siebdruck „FU“ (Sicherungs-Kennung); Typ nicht lesbar | 24, 26 |
| Q2, Q4, Q9, Q18, Q21 | Fünf MOSFET-artige SMD-Bauteile im Leistungspfad | Bauform; Zahl nicht abschließend | 23–27 |
| TVS1, TVS10, TVS12, TVS16, TVS17 | Große schwarze Dioden vom Typ SMB/SMC | Aufdruck teilweise sichtbar (`5?GM`, `02`), Typ und Spannung nicht bestimmt; TVS12/TVS16 sitzen bei den Zellabgriff-Pad-Gruppen `B18`, `B38`, `B39`, TVS10 bei einem `B+`-Pad | 32, 33, 34, 35, 37 |
| R212 | Großer SMD-Widerstand mit Aufdruck `100` (= 10 Ω), zwischen einem `B+`-Pad und D20/TVS10/C43/C58 | Aufdruck; vermutlich ein Serienelement im Versorgungspfad ab `B+` (abgeleitet) | 34, 37 |

## Stecker und Belegung (Siebdruck)

| Stecker | Pins (wie aufgedruckt) | Anmerkung |
|---|---|---|
| **J5** | `B0, B2, B4, B6, B8, B10, B12`, zwei unbeschriftet, `GND, NTC2, GND, NTC4` | Zellabgriffe (gerade Zellen) + zwei NTCs, 14-poliger Flachstecker |
| **J14** (BMS-Platine) | `B1, B3, B5, B7, B9, B11, B13`, einer unbeschriftet, `GND, NTC3, GND, NTC1` | Zellabgriffe (ungerade Zellen) + zwei NTCs, 14-poliger Flachstecker. Nicht zu verwechseln mit `J14` auf der Platine des Kopfteils (unten). |
| **J7** | `LED, 3.3V, GND, ADD-IN` | 4 Pins. **Ein/Aus-Schalter mit LED:** Das Kabel zu diesem Stecker trägt das Etikett `ON/OFF` (Foto 28), und der Einsender nennt den obersten 4-Pin-Stecker den Anschluss des Ein/Aus-Schalters. Beim Zusatzakku bleibt er unbenutzt (Forum). `ADD-IN` ist dann vermutlich der Schalter-/Weckeingang (abgeleitet). |
| **J1** | `GND, ADD-IN, IN, OUT, (2 unbeschriftet), H, L, AGND` | 9 Pins. Oberhalb der Isolationslinie (neben T1 aufgedruckt): `GND/ADD-IN/IN/OUT`; darunter `H`, `L`, `AGND`, also eigene Masse. Der Einsender nennt die obersten vier Pins (`GND/ADD-IN/IN/OUT`) als Kommunikation zum Hauptgerät/Top-Modul. `H`/`L` sind sehr wahrscheinlich das CAN-Paar (Firmware: CAN1; auf der Kopfteil-Seite durch `CAN_H`/`CAN_L` bestätigt, siehe unten). Die Bedeutung von `ADD-IN/IN/OUT` (Pack-Adressierung/Weckkette?) ist nicht geprüft. |
| **J2** | `VCC, DIO, CLK, GND` | 2,54 mm Durchsteck, 4 Pins, **nicht bestückt** – SWD |
| **J3** | `VCC, TX, RX, GND` | 2,54 mm Durchsteck, 4 Pins, **nicht bestückt** – UART |
| Leistung | `B+` ×12 Pads (drei Gruppen), `B-` ×4 Pads, `P-` ×8 Pads, drei größere Klemmen auf der MCU-Seite | Batterieseite (`B+`/`B-`) und Ausgangsseite (`P-`); die Kabel sind mit `B-`, `B+`, `PACK+` beschriftet (Foto 28) |
| Testpunkte | `TP1…TP20` | Am Rand der Leistungsseite, z. B. TP13/TP14 bei den Zellabgriffen, TP15–TP17 bei den NTC-Pads |

## Verdrahtung Pack ↔ Kopfteil

**Platine des Kopfteils, Stecker `J14`** (Foto „Head_Unit_01“, Platine mit Vergussmasse bedeckt): ein 5-poliger Wafer-Stecker. Siebdruck darunter:
`CAN_H`, `CAN_L`, `Button_S`, `GND`. Der Einsender nennt nur die oberen zwei der fünf Pins als belegt, `CAN_H` und `CAN_L`. Das passt zum Forenbefund,
dass Kopfteil und Akku über ein Zweidrahtkabel kommunizieren, und zur Firmware (BMS und Control sprechen CAN). `Button_S` ist vermutlich das Schaltersignal
des `ON/OFF`-Schalters (abgeleitet); am fotografierten Gerät ist es nicht angeschlossen.

**Gehäusestecker Pack ↔ Kopfteil/Erweiterung** (Fotos „Pack_Connector_04“ Vorderseite, „01…03“ Rückseite): 12 Kontakte. Vier große Kontakte sind mit 1–4 nummeriert.
Die acht kleinen Kontakte sind in einem dreieckigen Block mit 5–12 nummeriert: obere Reihe `9, 10, 11, 5`, mittlere Reihe `8, 12, 6`, unten `7`.

Belegung laut Einsender (Wortlaut: „as far as I expect“, hier nicht gemessen):

| Kontakt | Funktion (Einsender) |
|---|---|
| 1 | Erdung (PE) |
| 2 | Minus |
| 3 | Plus |
| 4 | Unbelegt |
| 5, 9, 10, 11 | Kommunikation |
| 6, 7, 8, 12 | Unbelegt |

Auf den Rückseitenfotos sichtbar: dicke rote und schwarze Leiter, ein grün-gelber Leiter und ein Bündel dünner Adern (schwarz, rot, magenta), die an die kleinen Kontakte gelötet sind.
Welche dünne Ader zu welchem kleinen Kontakt gehört, ist auf den Fotos nicht lesbar. Offen: wie die vier Kommunikationskontakte auf `J1` (`GND/ADD-IN/IN/OUT`, `H/L/AGND`)
und `J7` abgebildet werden; die Zahl des Einsenders (vier Kommunikationskontakte) passt nicht zu den acht signalartigen Pins an `J1` plus `J7`.

## Serielle Konsole

- **Gefunden:** die unbestückte 2×4-Durchsteck-Leiste **J3/J2** neben der MCU (Fotos 11, 12, 29, 30; von der Gegenseite auf den Fotos 23–25 sichtbar).
  Siebdruck: `VCC / TX / RX / GND` (J3) und `VCC / DIO / CLK / GND` (J2). Das quadratische Pad ist Pin 1 jeder Spalte (`VCC`).
  Foto 30 zeigt leere durchkontaktierte Löcher mit Lötresten, keine Stifte.
  Dieselbe Konvention (UART `3.3V/TX/RX/GND`, SWD `3.3V/DIO/CLK/GND`) wurde auf der Venus-A-Hauptplatine gesehen (siehe `../Teardown_Fotos/`).
- **Nicht bestimmt:**
  - Mit welchen MCU-Pins J3 TX/RX verbunden sind. Den Leiterbahnen ist auf den Fotos nicht sicher zu folgen.
    Nötig ist ein Durchgangstest zwischen J3 und den MCU-Pins. Kandidat aus der Firmware: USART1 an PA9/PA10 (der einzige UART im A-BMS-Image).
  - Die Baudrate einer Konsole. Die Firmware nennt 9600 Baud nur für das RS485 an USART1.
  - Ob `VCC` 3,3 V ist. Vor dem Anschluss eines USB-UART-Adapters nachmessen (nur 3,3-V-Pegel).
- Firmware-Kontext (BMS 115, letter-shell): Benutzer `BLUE`, leeres Passwort, etwa 39 Befehle; siehe
  [VNSA_BMS_FW_v115_Vollanalyse_de.md](../../BMS/VNSA_BMS_FW_v115_Vollanalyse_de.md) §4.3. Die Firmware-Notizen sagen außerdem, dass die Shell auf dem RS485-Debugpfad läuft;
  ob J3 dieselbe oder eine eigene Shell führt, ist offen.
- J2 `DIO/CLK` ist sehr wahrscheinlich SWD für den N32G455 (durch Messung zu bestätigen).

## Offene Punkte

1. Typen der SOIC-8-Chips (U2, U4, U11, U15, U16, U17, U21) und der TVS-Dioden: scharfe Makrofotos mit schrägem Licht nötig (die Fotos 36, 37 machen U21 nicht lesbar).
2. Zuordnung der J3/J2-Pins zu MCU-Pins (Durchgangstest).
3. Funktion von `ADD-IN/IN/OUT` an J1 und der zwei unbeschrifteten J1-Pins. Kandidat aus der Firmware: In A 115 liest das BMS **PB13** als Eingang für die Packauswahl-Quittung,
   und Control treibt eine Pack-Kodierleitung (PB10 → erstes BMS-PB13, siehe [VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md](../../Modbus_RS485_TCP/VNSD_Unklare_Register_Aufloesung_2026-10-01_de.md), Register 45025).
   Ob PB13 an `ADD-IN`, `IN` oder am Taster hängt, zeigen die Fotos **nicht** (Durchgangstest nötig).
4. Bedeutung von `FU1…FU3` (Typ, Nennwert).
5. Welche Hardware-Revision und Firmware-Version die fotografierten Geräte haben (die Platine gibt es mindestens mit den Baudaten `2025.08.22` und `2025.09.08`).
6. Funktion von SW1: Reset-Taster oder Benutzertaste. Ein Forennutzer berichtet, dass das Drücken bei einem BMS-Hänger (SoC bei 1,4 % festgehalten) keine sichtbare Wirkung hatte; erst ein BMS-Update half.
   In der D-/E3-BMS-Firmware ist PB13 die Power-Taste, in A 115 der Eingang für die Packauswahl-Quittung; an welchem Netz SW1 hängt, ist nicht bekannt.
7. Zuordnung der 12 Gehäusekontakte (1–12) zu `J1`/`J7` und zum Kopfteil-Stecker; die Liste des Einsenders ist eine Erwartung, keine Messung.
