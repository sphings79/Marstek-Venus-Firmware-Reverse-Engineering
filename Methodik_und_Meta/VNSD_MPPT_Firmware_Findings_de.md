# VNSD-0 MPPT-Firmware v104 — Befunde der statischen Analyse

> 🇬🇧 English version: [VNSD_MPPT_Firmware_Findings.md](VNSD_MPPT_Firmware_Findings.md)

Nur statische Analyse (Disassembly und Dekompilation des OTA-Images, TI C28x, geladen auf Wortadresse `0x3E8000`; siehe [VNSD_MPPT_C28x_Image_Setup_de.md](VNSD_MPPT_C28x_Image_Setup_de.md)). Nichts davon wurde an Hardware geprüft. Alle 454 Funktionen sind benannt; jeder Name trägt eine Konfidenz im Ghidra-Funktionskommentar: 408 hoch, 46 mittel, 0 niedrig. Mittlere Namen sind Arbeitsbezeichnungen, kein Beweis. Alle Adressen sind C28x-**Wort**adressen.

Die Registerlayouts der F2803x-Peripherie (ADC, Taktsteuerung, PLL, GPIO, CLA-Register) stammen zum Teil aus dem Wissen über die TI-Headerdateien, nicht aus einem abgerufenen Referenzhandbuch (das F2803x-Referenzhandbuch war nicht als Text abrufbar). Namen, die darauf beruhen, sind in der Zusammenfassung unten gekennzeichnet.

## Chip

| Punkt | Ergebnis | Beleg |
|---|---|---|
| Familie | **TI TMS320F2803x** (Piccolo), am wahrscheinlichsten **F28035** | PIE-Tabelle, LIN-A-Block, CLA, keine FPU-Befehle |
| Kein F2806x | die frühere Arbeitsannahme war falsch | die Konsolen-UART bei `0x6C00` ist der LIN-A-Block, den der F2806x nicht hat; nur 7 ePWM-Interruptslots sind belegt; keine FPU-Nutzung (LVF/LUF) |
| Warum F28035 | CLA vorhanden und mindestens 48 K Worte Flash | F28033 hat 32 K Flash, F28034 hat keine CLA |
| Gehäuse- / Q1-Variante | **offen** | braucht `PARTID` (`0x3D7E80`, wird von der Firmware nie gelesen) oder die Pinzahl |

Flashbeginn `0x3E8000` und das CLA-RAM bei `0x9000` unterscheiden F2803x und F2806x nicht. Die F2805x-Familie wurde nicht geprüft.

## Programmaufbau

| Punkt | Adresse | Hinweis |
|---|---|---|
| Reset-Einsprung | `0x3E8000` | `LB 0x3F0BAE` |
| `c_int00` | `0x3F0BAE` | Standard-TI-Startup, schreibt `WDCR = 0x68` (Watchdog aus), arbeitet die Kopiertabelle bei `0x3F2B30` ab, ruft die `main`-Kette |
| Flash-Setup-Routine | `0x3E8080` | `Init_FlashWaitStatesAndPipelineRamFunc`: schreibt die Flash-Register `0x0A80`–`0x0A87`; wird von `Init_StartupCopyRamAndPieSetup` (`0x3EF49B`) nach RAM `0x400` kopiert und dort ausgeführt |
| Delay-Schleife für RAM-Ausführung | `0x3E809B` | Flash-Ladeabbild einer Delay-Schleife, die aus dem RAM läuft; wird an ihrer Flash-Adresse nie aufgerufen |
| Peripherie-Init | `0x3EF4C9` | `Init_ConfigureAllPeripherals` |
| Main und Hintergrundschleife | `0x3EF4EF` | `Main_InitSequenceAndBackgroundLoop` |
| Periodischer Regelschritt | `0x3E8348` | `Mppt_ControlStep_FourChannelTracker`, aus dem Timer-1-Interrupt gerufen |
| Letter-shell-Konsole | Code `0x3EB7BE`–`0x3EC3xx`, Befehlstabelle `0x3F271A` | Einträge zu 8 Worten (Attribut, Name, Handler, Beschreibung) |
| Code-Insel im Datenbereich | `0x3F2CA0`–`0x3F2D19` | vier Q24- / Float-Hilfsfunktionen, von echtem Code gerufen |
| Initiale PIE-Vektortabelle | `0x3F296A` | 125 Langworte, von `Init_CopyPieVectorTable` (`0x3F07CE`) nach RAM `0x0D06` kopiert |

Der Rest von `0x3F0BE2`–`0x3F3FFF` enthält Strings (ein 16-Bit-Zeichen je Wort), Tabellen, die Kopiertabelle der C-Laufzeit und Füllung (siehe „Image-Aufbau").

## Interrupts

Die initiale Vektortabelle enthält nur Standard-Trap-Stubs (`ASP`, `ESTOP0`, Endlosschleife); die echten Handler werden zur Laufzeit in die PIE-Tabelle im RAM geschrieben. 110 der 112 PIE-Slots passen zur F2803x-Namenstabelle (eindeutige Stubs nur dort, wo das Datenblatt eine Peripherie nennt); ADCINT1/2 (INT10.1/10.2 in der Nummerierung des Datenblatts) sind die Ausnahme. Quellen für das Layout: TI-F2803x-Datenblatt SPRS584Q Tabelle 7-19, F2806x-Datenblatt SPRS698J Tabelle 8-17 (Reihenfolge der CPU-Ebene aus einer TI-F2806x-Headerdatei).

| Vektor | Quelle | Handler | Aufgabe |
|---|---|---|---|
| `0x0D1A` | INT13, CPU Timer 1 | `0x3F05FB` | bedient den Watchdog (`Wdt_ServiceKeys_55_AA`, Schlüssel `0x55` dann `0xAA`) und führt den Regelschritt `0x3E8348` aus; der Timertakt ist 1 ms (aus den Timer-Argumenten erschlossen) |
| `0x0D1C` | INT14, CPU Timer 2 | `0x3F0A66` | 32-Bit-Tickzähler bei `0x88C6` und ein Flag |
| `0x0D42` | INT1.2, ADCINT2 | `0x3F05DF` | ruft `Prot_AdcScaleFilterAndCheckFaults`, quittiert das PIE |
| `0x0DC8` | INT9.5, ECAN0INTA | `0x3ED7E1` | initialisiert den CAN-Controller bei einem Fehlerflag neu oder löscht die Empfangs-Pending-Bits; legt keine Daten in die Queue |
| `0x0DCA` | INT9.6, ECAN1INTA | `0x3ED814` | nur Empfang: kopiert jede anstehende Mailbox (31 abwärts bis 16) in den 20-Einträge-Ring bei `0x8042` |

`IER` ist am Ende `0x3101`. Die CLA-Abschlussinterrupts (PIE-Gruppe 11) werden nie freigegeben; die acht PIEACK-Stubs bei `0x3F043A`–`0x3F0479` sind ungenutzt. Etwa 76 identische 5-Wort-Trap-Stubs sind nach ihrem Vektor benannt. Ein unerwarteter Interrupt endet deshalb in einer Endlosschleife; der Watchdog wird dann nicht mehr bedient und setzt den Chip zurück.

## Watchdog

`WDCR` (`0x7029`): `0x68` = aus, `0x2D` = ein (Prüfbits 101, Teiler 5), `0x00` = ungültige Prüfbits, erzwingt sofort einen Reset. Der Watchdog ist am Reset-Einsprung und rund um zwei Kalibrierroutinen abgeschaltet und wird bei `0x3EF528` nach der Initialisierung eingeschaltet. Die Schlüssel schreibt `Wdt_ServiceKeys_55_AA` (`0x3EF13E`), gerufen aus dem Timer-1-Interrupt und aus einigen Init-Pfaden. Eine Heartbeat-Routine (`Comm_CheckHeartbeatTimeout_StopReset`) zählt 60 Aufrufe ohne CAN-Aktivität, stoppt dann die PWM und schreibt `WDCR = 0`, was den Chip zurücksetzt; die Magic `0xA5A5` in RAM `0x8782` („Master control“, gesetzt durch den Konsolenbefehl `master`) unterdrückt das. Beim Start liest die Firmware `WDCR` und gibt „Reset by Watchdog“ aus, wenn Bit 7 gesetzt ist. Das Bitfeld-Layout wurde nicht gegen das F2803x-Handbuch bestätigt; das Timeout ist unbekannt, weil die Oszillatorfrequenz nicht belegt ist.

## Hardware, soweit aus dem Code erkennbar

- Vier PWM-Kanäle: Registerblöcke bei `0x6800`, `0x6840`, `0x6880`, `0x68C0` (Abstand `0x40`). Periode `TBPRD = 0x4AF` (1200 Zählschritte), Totzeit 30. Der Tastgrad wird als `CMPA = D × 1199` plus hochauflösender Anteil (`CMPAHR`, Skala 111, wahrscheinlich die HRPWM-MEP-Schritte je Takt) geschrieben. Festkomma-Regelung (Q24) je Kanal.
- Die ADC-Ergebnisregister `0x0B00`–`0x0B0B` werden von der Konsolenausgabe der Firmware selbst benannt: `PV1_V`, `PV1_I`, … `PV4_V`, `PV4_I`, `CELL_V`, `CELL_I`, `VREF`, `PE_V`: vier PV-Eingänge, Spannung und Strom auf der Akkuseite, eine Referenz und eine Schutzerde-Spannung. Eine Offset-Selbstkalibrierung (`Adc_OffsetSelfCal`) und ein Mittelungsdurchlauf über 256 Werte sind vorhanden.
- Ein EEPROM hängt am I2C-Block (`0x7900`, 400 kHz, Slave-Adresse `0x50`, `0x51` oberhalb von Adresse `0xFFFF`).
- Ein CAN-Controller mit 32 Mailboxen im eCAN-Layout (`0x6000`, Mailbox-RAM `0x6100`). Mailboxen 0–15 senden, 16–31 empfangen; die Knotenadresse im RAM `0x8002` geht in die Identifier ein. Die Bitrate ist nicht ermittelt.
- Die Konsolen-UART ist der LIN-A-Block bei `0x6C00`, als SCI genutzt (Baudrate nicht ermittelt).
- Die CLA (Control Law Accelerator) wird initialisiert: Programmkopie nach RAM `0x9000`, Vektor `MVECT8 = 0xE96` (Task-8-Code bei `0x9E96`), `MIER` Bit 7 gesetzt. `IACK #0x80` startet CLA-Task 8 an drei Stellen; die Hauptsequenz wartet danach auf `MIRUN` Bit 7. Der Task-Code selbst gehört nicht zum analysierten Programmraum.
- Es wird Code außerhalb des Images aufgerufen: `0x3D7C80` (Taktkalibrierung), `0x3D7E82` und `0x3D7E85` (Kalibrierung des Temperatursensors). Das ist der von TI programmierte OTP-Bereich des Bausteins.
- Zeitbasis (erschlossen, mittel): 60 MHz Systemtakt, ADC-/PWM-Interrupt mit 50 kHz, Tracker mit 1 kHz.

## Skalierung und Einheiten

Alle Werte sind Festkomma Q24, wobei **1,0 dem ADC-Vollausschlag entspricht** (der ADC-Interrupt wandelt jedes Ergebnis mit `raw << 12`, also 4096 Counts = 3,3 V = `0x1000000`). Float-Konstanten liegen mit dem High-Wort zuerst (geprüft an runden Werten wie 60,0, 1000,0, 25000,0).

| Größe | Vollausschlag (abgeleitet) |
|---|---|
| PV-Spannung | etwa 68,04 V |
| PV-Strom | 44,0 A |
| PV-Leistung | etwa 2993,81 W |
| Zellspannung | etwa 73,52 V |
| Zellleistung | etwa 7889,17 W |
| PE-Spannung | etwa 132,7 V |

Der Zellstrom ist `(VREF − I) / 0,0205 A`, also ein Sensor mit 20,5 mV/A. Die abgeleiteten Spannungsteiler sind etwa 20,6:1 (PV), 22,3:1 (Zelle), 40,2:1 (PE) bei 3,3 V Referenz. Die echten Widerstandswerte stehen nicht im Code. Die IIR-Filter haben die Koeffizienten `0x202B7F` (schnell) und `0x83C` (langsam) geteilt durch 2^24, was bei 20 µs Abtastzeit etwa 1 kHz und 1 Hz entspricht.

Die Telemetrie wird in Schritten von 0,1 V, 0,1 A und 0,1 W gemeldet. Im Telemetrieblock (`data_toCan`, `0x8400`) ist die Kanalreihenfolge **umgekehrt**: Index 0 enthält PV4 und Index 3 PV1; die Status-Laufbits und die Regel „unter 5 W ist 0“ verwenden die normale Reihenfolge. Ob das Absicht ist, ist unbekannt.

## Regelung und Schutz

- Der Tracker ist **Incremental Conductance** (er vergleicht `I/V` mit `−dI/dV` bei fester Schrittweite), eine Vierkanal-Zustandsmaschine auf RAM `0x83C0` (0 Warten/Init, 1 Eingangsbereichsprüfung, 2 Lauf, 3 Abschaltung/Reset) mit einem 38-Wort-Kontext je Kanal bei `0x8580`. Die Leistungsstufe sieht nach einem Boost-Wandler aus. `Mppt_IncCondTrackerStep` (`0x3EFA0F`) ist eine zweite Incremental-Conductance-Umsetzung ohne Aufrufer (toter Code); ein früherer Arbeitsname nannte sie Perturb-and-Observe, das war falsch.
- PV-Startfenster 16 V bis 56 V, Stopp unter 13 V oder über 60 V.
- Der Konstantleistungsmodus wird bei `0x83CE` gekennzeichnet. Leistungsgrenze je Kanal 1041,67 W, gesamt 4166,7 W, Akkugrenze 4000 W.
- `Prot_AdcScaleFilterAndCheckFaults` (`0x3E9333`) skaliert und IIR-filtert die ADC-Werte, vergleicht sie mit Schwellen und hält Fehlerbits in `0x8748`–`0x874D` fest. Entschlüsselte Auslösepunkte (aus den Konstanten; runde Werte bestätigen die Skala): PV-Überspannung 63,0 V (Entprellung 25000 Samples), PV-Strom 32,0 A (entprellt) und 40,0 A, Stromoffset-Grenze 1,0 A, Zell-Überspannung 60 V, Unterspannung 15 V, Überstrom 120 A, Rückstrom −30 A, PE-Auslösung 60 V, die Rohgrenze der Chiptemperatur entspricht 95 °C.
- **Text und Code widersprechen sich:** Die Fehlertexte der Konsole sagen „PV voltage over 58 V“, „diode current over 14.5 A“ und „temperature over 90 °C“; die entschlüsselten Auslösepunkte sind 63,0 V, 32,0 A und 95 °C. Die Meldungen sind möglicherweise veraltet.
- `Status_BuildTelemetryFromFilteredAdc` (`0x3E98E8`) füllt `data_toCan` (`0x8400`–`0x8420`): PV-Spannung, -Strom und -Leistung je Kanal, Status, Fehler, Warnung, Energieslots (Tag, Woche, Monat, Jahr).
- Die unter „Watchdog“ beschriebene Heartbeat-Prüfung ist in Wahrheit ein Timeout bei CAN-Stille.

## CAN

`Can_ProcessRxCommandQueue` (`0x3EAB78`) liest den 20-Einträge-Empfangsring bei `0x8042` (8 Worte je Eintrag; Schreibindex `0x80E3` aus dem ECAN1-Interrupt, Leseindex `0x80E2`). Behandelte Befehlsbytes: `1`/`4` Start/Stop, `2` (Regelmodus, Leistungs- und Spannungsgrenze), `3` Status (beantwortet mit acht Rahmen, 56 Bytes: PV1..4 V/I/P, Zustand, Fehler, Temperatur, Warnung, Energien, Akku-/Basis-/PE-Spannung), `5`, `6` (Energieslots im EEPROM löschen), `0xCB` (Versionsabfrage), `0xCC` (Datumsblock, Energie-Rollover), `0xCE` (Loader betreten, siehe unten), `0xFA`–`0xFF` (Seriennummer, Offset-Kalibrierung). Im Shell-Modus ignoriert die Firmware CAN-Befehle. Der einzige unaufgeforderte Rahmen ist ein 8-Byte-Fehlerschnappschuss alle 100 ms, solange `0x83D1` ungleich null ist (ganze Volt von PV1..4, dann ganze Ampere von PV1..4). Die Identifier sind 29 Bit mit Befehlsbyte, Adress-Nibbles und 5-Bit-Rahmenindex; die Nibble-Rollen sind nicht bitgenau.

## Update-Weg (aus Sicht der Control-Firmware)

Die Control-Firmware sendet das MPPT-Image über **CAN** mit dem Befehlsbyte `0xCE` (Identifier `0x000121CE`, `0x010121CE`, `0x020121CE`, 8 Datenbytes je Rahmen): Subcode 0 kündigt die Gesamtlänge und eine CRC16 über das ganze Image an, Subcode 1 trägt Datenblöcke zu 1024 Byte (96 Blöcke) mit je einer Block-CRC16 und einer Sequenznummer, Subcode 2 meldet den Blockstatus. Der MPPT antwortet 1 (nächster Block), 2 (Block wiederholen) oder 3 (neu starten); mehr als 25 Wiederholungen sind ein Fehler. Einstiegspunkte am Control sind BLE (Zieltyp 2, zwischengespeichert im externen Flash `0x100000`), die Cloud und der Konsolen-YModem-Befehl `update_mppt`. Das Image wird roh gesendet, einschließlich seines 32-Byte-Footers.

Die MPPT-Anwendung enthält **keinen Loader**: Ihr eigener `0xCE`-Handler setzt den PWM-Zustand zurück, setzt EEPROM-Byte 0 auf 1 und erzwingt einen Watchdog-Reset, ohne zu antworten; der normale Start löscht das Byte wieder. Er hat keinen Lösch- oder Programmiercode und sendet nie `0xCE`, obwohl das Control `0xCE`-Antworten erwartet. Der Loader, der das Image empfängt, liegt deshalb außerhalb des analysierten Programmraums (plausibel Flash unterhalb des Images oder das Flash-Ende `0x3F4000`–`0x3F7FFF`); seine CAN-Bitrate, ID-Dekodierung, Footer- oder CRC-Prüfung und sein Startvorgang sind unbekannt. Das Control prüft keine Mindestversion des MPPT; die Version kommt über eine periodische `0xCB`-Abfrage und wird nur angezeigt.

## Image-Aufbau

| Bereich | Inhalt |
|---|---|
| `0x3E8000`–`0x3F3E9D` | Code, Strings und Tabellen |
| `0x3F3E9E`–`0x3F3FEF` | Füllung `0xFFFF` (338 Worte) |
| `0x3F3FF0`–`0x3F3FFF` | 16-Wort-Footer: `VNSD`, Nullen, `0xFFFF`-Füllung, `0x7CE8`, **`0x2222`** |

`0x2222` ist die Gerätemaske des MPPT in der Update-Tabelle des Control (BMS und Micro nutzen `0x3333` und `0x4444`); das Control prüft sie bei der Validierung eines Updates. `0x7CE8` wird weder vom Control noch von der MPPT-Anwendung gelesen und passt zu keiner getesteten Prüfsumme (CRC-16-Varianten, Byte- und Wortsummen, CRC32 über die ganze Datei, Teilbereiche und alle Präfixe). Auf dem BLE-Weg überschreibt das Control die Maskenbytes des letzten Chunks vor CRC und Flash-Schreiben mit `0xFFFF`; ein über BLE zwischengespeichertes und weitergeleitetes Image endet daher auf `7CE8 FFFF`; warum, ist unbekannt.

## Konsole (letter-shell)

Das Image enthält den Open-Source-Befehlsinterpreter *letter-shell* mit Benutzern, Befehlen, Variablen, Tasten, einer Passwortabfrage und einem Verlauf mit fünf Einträgen. Die Tabelle bei `0x3F271A` ordnet diese Befehle Handlern zu: `ctl_d`, `ctl_mode`, `bms_power`, `bms_cur`, `bms_vol`, `start`, `epwm_start`, `ss`, `offset_cal`, `offset_read`, `ele_read`, `ele_clear`, `clear_prt`, `master`, `help`, `clear` und die Anzeigebefehle `dshow`, `adshow`, `adcshow`, `powershow`, `mpptshow`, `prtshow`, `canshow`, `snshow`, `vershow`. `vershow` gibt die Anwendungsversion `104` aus. `ctl_d` schreibt einen Q24-Tastgrad in die hochauflösenden Tastgradregister der vier PWM-Blöcke; ab einem Wert von 10000 wird eine Abschaltung erzwungen. Das Konsolengerät ist in der TI-C-Laufzeit als `uart` registriert. Ob die Konsole am ausgelieferten Gerät erreichbar ist, wurde **nicht** untersucht.

## EEPROM-Inhalt (aus dem Code erkennbar)

Versionsworte `101` bei `0x100` und `104` bei `0x104`, eine 20-stellige Seriennummer bei `0x109`, fünf Langworte bei `0x20`–`0x30`, die die Stromoffset-Kalibrierwerte sind (PV1..4 und Zelle), und Slots bei `0x10`–`0x1C`, die die kumulierten Energiesummen sind. Die Bedeutung der Felder des Datumsblocks (Jahr, Monat, Tag) ist nicht entschlüsselt.

## C-Laufzeit im Image

Funktionen der TI-Laufzeitbibliothek sind vorhanden und nach ihrer Wirkung benannt: printf mit `%e`/`%f`/`%g`, malloc / free / realloc / calloc auf einem kleinen Heap bei `0x8A20`–`0x8B1E`, stdio, die TI-C-I/O-Hostfunktionen, String- und Speicherhilfen, 16/32/64-Bit-Division und Float/Double-Umwandlungen. Die 64-Bit-Divisionshilfe bei `0x3EF40A` liefert den Quotienten; der Resteinstieg ist `0x3EF439`.

## Bekannte Schwachstellen

- Namen sind Beschriftungen aus der statischen Analyse. 46 Funktionen haben mittlere Konfidenz: die GPIO20/21- und GPIO44-Routinen, die Komparator-Routinen (Registerlayout nicht geprüft), die Passwort- und Benutzerteile der Shell, mehrere printf-Hilfen, der EEPROM-Datumsblock-Leser/-Schreiber und das Heartbeat-Paar.
- Die Namen der Peripherieregister für ADC, Taktsteuerung, PLL, GPIO und der CLA-Register (`MIER` bei `0x1425`, `MIRUN` bei `0x1426`) stammen aus dem Wissen über die TI-Header dieser Familie und wurden nicht gegen ein abgerufenes Handbuch geprüft.
- Der Zwei-Wort-Befehl `0x763F 0x0080` kommt dreimal vor (bei `0x3E80C0`, `0x3E9CF3`, `0x3EF4F9`). Das Ghidra-Prozessormodul kannte ihn nicht und hat die umgebenden Funktionen zerlegt. Es ist `IACK #0x0080`; die Kodierung (`0x763F` plus 16-Bit-Wert) ist im TI-Befehlssatzhandbuch (SPRU430F) bestätigt, die Zuordnung der Task-Bits stammt aus einer TI-CLA-FAQ. Ein lokaler Patch ergänzt den Befehl im Modul (siehe Setup-Dokument).
- 16 Funktionen (Interrupt-Handler und nicht aufgerufene Routinen) fehlten in der ersten Funktionsliste, weil sie niemand aufruft; sie wurden nachträglich ergänzt. Code, der nur indirekt erreicht wird, könnte sich genauso verstecken; die Prüfung „kein dekodierter Befehl außerhalb einer Funktion“ ist jetzt sauber.
- Spannungsteiler- und Shunt-Werte, die CAN-Bitrate, die Einheit des Energiezählers (etwa 10,4 Wh je Zählschritt, wenn der Tracker genau mit 1 kHz läuft), der ADC-Konvertierungs-Trigger und das Watchdog-Timeout sind nicht ermittelt.
- RAM-Variablennamen gibt es für 188 Stellen mit Beleg aus Konsolenstrings oder Code; der übrige RAM ist unbenannt. Strukturtypen (`ChanCtx`, `DataToCan`, `FaultSnapshot`, `CanFrameEntry`, `TimerDrv`, `AdcPipeline`) sind im Ghidra-Projekt definiert; die Feldbedeutungen haben stellenweise mittlere oder niedrige Konfidenz.
