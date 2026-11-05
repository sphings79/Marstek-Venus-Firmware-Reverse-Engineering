# VNSD-0 MPPT-Firmware v104 — Befunde der statischen Analyse

> 🇬🇧 English version: [VNSD_MPPT_Firmware_Findings.md](VNSD_MPPT_Firmware_Findings.md)

Nur statische Analyse (Disassembly und Dekompilation des OTA-Images, TI C28x, geladen auf Wortadresse `0x3E8000`; siehe [VNSD_MPPT_C28x_Image_Setup_de.md](VNSD_MPPT_C28x_Image_Setup_de.md)). Nichts davon wurde an Hardware geprüft. Alle 438 Funktionen sind benannt, die Namen tragen aber eine Konfidenz im Ghidra-Funktionskommentar: 188 hoch, 250 mittel, 0 niedrig. Mittlere und niedrige Namen sind Arbeitsbezeichnungen, kein Beweis. Alle Adressen sind C28x-**Wort**adressen.

## Programmaufbau

| Punkt | Adresse | Hinweis |
|---|---|---|
| Reset-Einsprung | `0x3E8000` | `LB 0x3F0BAE` |
| `c_int00` | `0x3F0BAE` | Standard-TI-Startup: schaltet den Watchdog ab (`0x7029` = `0x68`), arbeitet die Kopiertabelle bei `0x3F2B30` ab, ruft die `main`-Kette |
| Hauptschleife | `0x3EF4FA` | `Main_BackgroundLoop`; Peripherie-Init-Folge bei `0x3EF4C9` |
| Periodischer Regelschritt | `0x3E8348` | `Mppt_ControlStep_FourChannelTracker`, aus einem Interrupt-Handler gerufen (Eintritt `0x3F05FB`), kein anderer Aufrufer |
| Interrupt-Vektortabelle | von `0x3F296A` (125 Langworte) nach `0x0D06` kopiert durch `0x3F07CE` | danach wird PIE freigegeben |
| Letter-shell-Konsole | Code `0x3EB7BE`–`0x3EC3xx`, Befehlstabelle bei `0x3F271A` | Einträge zu 8 Worten (Attribut, Name, Handler, Beschreibung) |
| Code-Insel im Datenbereich | `0x3F2CA0`–`0x3F2D19` | vier Q24- / Float-Hilfsfunktionen, von echtem Code gerufen |

Der Rest von `0x3F0BE2`–`0x3F3FFF` enthält Strings (ein 16-Bit-Zeichen je Wort), Tabellen und die Kopiertabelle der C-Laufzeit.

## Hardware, soweit aus dem Code erkennbar

- Vier PWM-Kanäle: Registerblöcke bei `0x6800`, `0x6840`, `0x6880`, `0x68C0` (Abstand `0x40`), Festkomma-Regelung (Q24) je Kanal.
- Die ADC-Ergebnisregister `0x0B00`–`0x0B0B` werden von der Konsolenausgabe der Firmware selbst benannt: `PV1_V`, `PV1_I`, … `PV4_V`, `PV4_I`, `CELL_V`, `CELL_I`, `VREF`, `PE_V`. Das sind vier PV-Eingänge mit Spannung und Strom, Spannung und Strom auf der Akkuseite, eine Referenz und eine Schutzerde-Spannung.
- Ein EEPROM hängt an einer I2C-artigen Peripherie (`0x7900`–`0x7909`, `0x7921`, 400 kHz, Slave-Adresse `0x50`, `0x51` oberhalb von Adresse `0xFFFF`). **Korrektur:** Eine frühere Arbeitsnotiz bezeichnete `0x7900` als GPIO-Block; der Code verhält sich wie I2C.
- Ein CAN-Controller mit 32 Mailboxen im eCAN-artigen Layout (`0x6000` / Mailbox-RAM `0x6100`). Mailboxen 0–15 senden, 16–31 empfangen; die Knotenadresse im RAM `0x8002` geht in die Identifier ein.
- Es wird Code außerhalb des Images aufgerufen: `0x3D7C80` (Takt / Init), `0x3D7E82`, `0x3D7E85` (Kalibrierung des Temperatursensors). Das passt zu werkseitig programmiertem Code und Kalibrierdaten in einem Chipbereich, der nicht Teil des OTA-Images ist.
- Genauer Chiptyp: weiterhin unbekannt.

## Regelung und Schutz

- Eine Vierkanal-Zustandsmaschine auf RAM `0x83C0` (0 Warten/Init, 1 Eingangsbereichsprüfung, 2 Lauf, 3 Abschaltung/Reset), 80-Sample-Min/Max-Fenster und ein 38-Wort-Kontext je Kanal bei `0x8580`. Die Referenzaktualisierung sieht nach Perturb-and-Observe oder Inkrementalleitwert aus; welches von beiden, ist nicht bewiesen. Eine eigene Blattfunktion `Mppt_PerturbObserveStep` (`0x3EFA0F`) setzt einen Perturb-and-Observe-Schritt um.
- Konstantleistungsmodus wird bei `0x83CE` gekennzeichnet (die Konsole gibt `CP_CTL_MODE` oder `MPPT_CTL_MODE_CONSTANT_STEP` aus).
- Die Schutzroutine `Prot_AdcScaleFilterAndCheckFaults` (`0x3E9333`) skaliert und IIR-filtert die ADC-Werte, vergleicht sie mit Schwellen und hält Fehlerbits in `0x8748`/`0x8749`/`0x874A`/`0x874B` und `0x874C`/`0x874D` fest. Die Fehlertexte der Konsole nennen: PV1–4-Überstrom, Zell-Überstrom / Über- und Unterspannung / Rückstrom, Hardware-Zellüberspannung und -Rückstrom, Chiptemperatur, PE-Überspannung / -Warnung, PV-Spannung über 58 V und PV-Diodenstrom über 14,5 A.
- `Status_BuildTelemetryFromFilteredAdc` füllt den CAN-Telemetrieblock `data_toCan` bei `0x8400`–`0x8416`: PV-Spannung, -Strom und -Leistung je Kanal, Status, Fehler, Warnung sowie Tages-, Wochen-, Monats- und Jahresenergie.
- Eine Heartbeat-Prüfung (`0x3EAB55`) stoppt die PWM und erzwingt einen Watchdog-Reset nach 60 s ohne Befehle, außer in RAM `0x8782` steht die Magic `0xA5A5` („Master control“, gesetzt durch den Konsolenbefehl `master`).
- Weitere `0xA5A5`-Werte bei `0x83C1` und `0x83C4` wirken im Regelschritt als Override-Schlüssel.

## CAN-Befehlsbehandlung (wie von der Analyse benannt)

`Can_ProcessRxCommandQueue` (`0x3EAB78`) liest einen Empfangsring mit 8-Wort-Einträgen bei `0x8042`. Behandelte Befehlsbytes: `1`/`4` Start/Stop, `2` (Regelmodus, Leistungs- und Spannungsgrenze), `3` Status, `5`, `6` (Energieslots im EEPROM löschen), `0xCB`, `0xCC` (Datumsblock, Energie-Rollover), `0xCE`, `0xFA`–`0xFF` (Seriennummer, Offset-Kalibrierung). Außerdem sendet sie periodisch einen Rahmen mit acht Werten. Im Shell-Modus ignoriert die Firmware CAN-Befehle.

## Konsole (letter-shell)

Das Image enthält den Open-Source-Befehlsinterpreter *letter-shell* mit Benutzern, Befehlen, Variablen, Tasten, einer Passwortabfrage und einem Verlauf mit fünf Einträgen. Die Tabelle bei `0x3F271A` ordnet diese Befehle Handlern zu (Namen, wie sie aus der Tabelle gelesen wurden): `ctl_d`, `ctl_mode`, `bms_power`, `bms_cur`, `bms_vol`, `start`, `epwm_start`, `ss`, `offset_cal`, `offset_read`, `ele_read`, `ele_clear`, `clear_prt`, `master`, `help`, `clear` und die Anzeigebefehle `dshow`, `adshow`, `adcshow`, `powershow`, `mpptshow`, `prtshow`, `canshow`, `snshow`, `vershow`. `vershow` gibt die Anwendungsversion `104` aus.

`ctl_d` schreibt einen Q24-Tastgrad in die hochauflösenden Tastgradregister der vier PWM-Blöcke; ab einem Wert von 10000 wird eine Abschaltung erzwungen. Das Konsolengerät heißt in der Firmware `uart`; welche Pins und welche Baudrate es nutzt, ist hier nicht geklärt. Ob die Konsole am ausgelieferten Gerät erreichbar ist, wurde **nicht** untersucht.

## EEPROM-Inhalt (aus dem Code erkennbar)

Versionsworte `101` bei `0x100` und `104` bei `0x104`, eine 20-stellige Seriennummer bei `0x109`, Kalibrier-Offsets für PV- und Zellstrom und fünf Langworte bei `0x20`–`0x30`, die wahrscheinlich kumulierte Energiezähler sind (nicht bestätigt).

## C-Laufzeit im Image

Funktionen der TI-Laufzeitbibliothek sind vorhanden und nach ihrer Wirkung benannt: printf mit `%f`/`%g`/`%a`, malloc / free / realloc / calloc auf einem kleinen Heap bei `0x8A20`–`0x8B1E`, stdio (fopen, freopen, setvbuf, fputc), TI-C-I/O-Hostfunktionen, String- und Speicherhilfen, 32/64-Bit-Division und Float/Double-Umwandlungen. Die 64-Bit-Divisionshilfe bei `0x3EF40A` liefert den Quotienten (geprüft: gleiche Operanden ergeben 1, ein kleineres Dividend 0); der Resteinstieg ist `0x3EF439`.

## Bekannte Schwachstellen

- Keine Funktion hat mehr niedrige Konfidenz: Die 55 Namen mit niedriger Konfidenz aus dem ersten Durchgang wurden einmal nachgeprüft und auf mittel oder hoch gehoben. Auch die Nachprüfung beruht auf dem Lesen des Codes, nicht auf Tests an Hardware. Etwa 76 Funktionen sind identische 5-Wort-Stubs (`ASP`, `ESTOP0`, Endlosschleife) in einer Reihe ab `0x3EEFA8`; sie sehen nach ungenutzten Trap-/Interrupt-Handlern aus und machen ein Sechstel der Funktionszahl aus, ohne Logik zu tragen.
- Der Zwei-Wort-Befehl `0x763F 0x0080` kommt dreimal vor (bei `0x3E80C0`, `0x3E9CF3`, `0x3EF4F9`). Das Ghidra-Prozessormodul kannte ihn nicht und hat die umgebenden Funktionen in Stücke zerlegt. Es ist `IACK #0x0080` (aus dem Zusammenhang erschlossen, **nicht gegen das TI-Handbuch bestätigt**): Jedes Mal folgt `RPT #3 || NOP`, die übliche Folge, um einen CLA-Task zu starten und vier Takte zu warten, und der folgende Code fragt ein CLA-Flagbit ab. Ein lokaler Patch ergänzt den Befehl im Modul (siehe Setup-Dokument); die Stücke wurden zu drei Funktionen zusammengeführt: `Init_ResetControlStateAndTriggerCla` (`0x3E809F`), `Ctrl_DisableOutputsAndInitChannelParams` (`0x3E9CE6`) und `Main_InitSequenceAndBackgroundLoop` (`0x3EF4EF`). Dass an diesen drei Stellen CLA-Task 8 gestartet wird, ist ein Schluss.
- Die Chipfamilie ist nur ein Hinweis: Die Registernutzung sieht nach einem Piccolo der Klasse TI F2806x/F2803x aus (I2C bei `0x7900`, GPIO-Bank B bei `0x6F80`, ePWM bei `0x6800`, ein CLA-Programm, das nach `0x9000` kopiert wird). Die Registernamen für Takt, GPIO und PIE stammen aber aus dem TI-Headerwissen dieser Familie und nicht aus diesem Projekt und haben mittlere Konfidenz.
- Q24-Skalierungsfaktoren und die physikalischen Einheiten der PV- und Zellwerte wurden nicht abgeleitet.
- Namen für RAM-Variablen gibt es nur an 106 Stellen mit Beleg aus Konsolenstrings oder Code; der übrige RAM ist unbenannt.
