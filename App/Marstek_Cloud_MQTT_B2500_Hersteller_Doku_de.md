# Marstek-Cloud-MQTT-Herstellerdoku (B2500-Familie)

> 🇬🇧 English version: [Marstek_Cloud_MQTT_B2500_Vendor_Doc.md](Marstek_Cloud_MQTT_B2500_Vendor_Doc.md)

Notizen zur MQTT-Protokollseite, die Marstek auf seinem eigenen Static-Host veröffentlicht. Dieses Dokument ist eine Zusammenfassung
in eigenen Worten; die Herstellerseite selbst liegt nur lokal (Fremdmaterial, nicht erneut veröffentlicht).

| | |
|---|---|
| Quelle | `https://static-eu.marstekenergy.com/ems/mqtt/index.html?version=<n>&type=<devType>&mac=<id>` |
| Abgerufen | 2026-10-30 |
| Lokale Kopie | `Hersteller_Doku/Marstek_MQTT_Doku_B2500_2026-10-30.html` (SHA-256 `4cc72384…af2a`), in keinem Repo |
| Seitentitel | „MQTT“ (englisch, mit Schaltfläche „Download PDF“) |

## 1. Was die Seite ist und was nicht

- **Die URL-Parameter bewirken fast nichts.** Das Seitenskript liest `type` und `mac` und schreibt sie in die Topic-Platzhalter
  (`{type}`, `{uid or mac}`). `version` wird gar nicht ausgewertet. Für jedes Produkt kommt derselbe Text.
  Der Beispiel-Link nutzte `type=HMG-50`; das ist nur eine Platzhalter-Füllung, keine Aussage über HMG-50.
- **Der Inhalt ist die B2500-Familie.** Hinweise in den Befehlsabschnitten nennen den B2500 der ersten Generation (Produktcodes `HMB-`)
  und der zweiten Generation (`HMA-`, `HMF-`, `HMK-`). Die Telemetrie (zwei Solareingänge, zwei Ausgänge, zwei Akkupacks) passt dazu.
- **Dass sie Venus beschreibt, ist nicht belegt.** Die `cd`-Nummerierung gilt je Produkt (siehe
  [Marstek_App_Befehlsvokabular_1.6.76_de.md](Marstek_App_Befehlsvokabular_1.6.76_de.md), Abschnitt 5 für B2500, Abschnitt 4 für Venus).
  Die Venus-D-Befehle belegen dieselben Nummern mit anderer Bedeutung (z. B. `cd=2` = Arbeitsmodus). Die Seite nicht auf Venus-Geräte übertragen.
- Der B2500-Abschnitt des App-Vokabulars führt ebenfalls `cd=07` für B2500 (`setB2500State`, `setTimePower`), das passt zur Seite.
  Die übrigen Befehle wurden nicht verglichen.

## 2. Transport

| Richtung | Topic | Zweck |
|---|---|---|
| Gerät → Cloud/App | `hame_energy/<type>/device/<id>/ctrl/#` (subscribe) | Telemetrie als eine Zeile `key=value,key=value,...` |
| App → Gerät | `hame_energy/<type>/App/<id>/ctrl` (publish) | Befehle, Payload beginnt mit `cd=<n>` |

`cd=01` (publish, ohne weiteres Feld) fordert eine frische Telemetriezeile an.

## 3. Telemetriefelder (Subscribe-Payload)

Felder ohne Versionshinweis gibt es in allen Versionen, die die Seite abdeckt (212 bis 220).

| Schlüssel | Bedeutung |
|---|---|
| `p1`, `p2` | Solareingang 1 / 2, Status |
| `w1`, `w2` | Solareingang 1 / 2, Leistung |
| `pe` | Akkustand in Prozent |
| `vv` | Geräteversion |
| `cs`, `cd` | Ladeeinstellung, Entladeeinstellung |
| `am` | „AM“ (die Seite erklärt es nicht) |
| `o1`, `o2` | Ausgangszustand 1 / 2 |
| `do` | Entladetiefe |
| `lv` | Schwelle für die Akkuausgabe |
| `cj` | Szene |
| `kn` | Akkukapazität |
| `g1`, `g2` | Ausgangsleistung 1 / 2 |
| `b1`, `b2` | Akkupack 1 / 2 angeschlossen |
| `md` | Modus der Entladeeinstellung |
| `d1`..`d3`, `e1`..`e3`, `f1`..`f3`, `h1`..`h3` | Zeitfenster 1 bis 3: aktiv, Start (`H:MM`), Ende, Ausgabewert |
| `d4`, `d5`, `e4`.., `f4`.., `h4`.. | Zeitfenster 4 und 5 (ab Version 218.2) |
| `sg`, `sp`, `st` | Sensor angeschlossen, automatische Leistung des Monitors, vom Monitor gesendete Leistung |
| `tl`, `th` | Minimale / maximale Zelltemperatur |
| `tc`, `tf` | Alarm Ladetemperatur / Entladetemperatur |
| `ts` | WLAN-Signal |
| `fc` | Version des Kommunikationschips (FC4) |
| `id`, `a0`, `a1`, `a2`, `l0`, `l1` | Geräte-ID, Kapazität Host / Zusatzpack 1 / Zusatzpack 2, Statusbits (ab 212.17) |
| `bc`, `bs`, `pt`, `it` | Tagessummen: Akku laden, Akku entladen, PV-Ladung, Einspeisung Mikrowechselrichter (ab 218) |
| `c0`, `c1`, `m0`..`m3` | CT001-Kanal, CT001-Diagnosestand, CT001-Zangenleistungen, Echtzeitleistung Mikrowechselrichter (ab 218.2) |
| `lmo1`, `lmo2`, `lmi1`, `lmi2`, `lmf` | Nennleistung Ausgang / Eingang Kanal 1 und 2, Kennzeichen „Gerät ist begrenzt“ (ab 220.1) |

**Fehler in der Herstellerseite** (beim Vergleich der Beispielzeile mit der Tabelle aufgefallen; am Gerät nicht geprüft):
Die Beispielzeile hat `pe-99` (wohl `pe=99`), `02=1` (wohl `o2=1`), `1v=200` (Tabelle: `lv`),
`t1=26` (Tabelle: `tl`), `2=0:0` (ein verlorener `e2`/`f2`-Schlüssel) und `e3=0:0f3=0:0` (fehlendes Komma).

## 4. Befehle (Publish-Payload)

„Flash“ = die Einstellung wird dauerhaft im Gerät gespeichert.

| Payload | Wirkung | Flash | Version |
|---|---|---|---|
| `cd=01` | Telemetrie anfordern | - | - |
| `cd=03,md=0` / `md=1` | Lademodus: gleichzeitig laden und entladen / erst voll laden, dann entladen | ja | - |
| `cd=17,md=0` / `md=1` | wie `cd=03` | nein | ab 214.1 |
| `cd=04,md=0..3` | Entlademodus: alle Ausgänge aus / OUT1 / OUT2 / OUT1 und OUT2 (nur B2500 erster Generation, `HMB-`) | ja | - |
| `cd=18,md=0..3` | wie `cd=04` | nein | ab 214.1 |
| `cd=05,md=<0-100>` | Entladetiefe in Prozent (Beispiel: `md=95`) | ja | - |
| `cd=19,md=<0-100>` | wie `cd=05` | nein | ab 214.1 |
| `cd=06,md=<0-500>` | Startschwelle der Akkuausgabe in W (Beispiel: `md=300`) | - | - |
| `cd=07,md=0,a1=..,b1=..,e1=..,v1=..,a2=..,...` | Drei Zeitfenster: `aN` an/aus, `bN` Start, `eN` Ende, `vN` Ausgabewert, Bereich 80 bis 800 W (B2500 zweiter Generation: `HMA-`, `HMF-`, `HMK-`) | ja | - |
| `cd=07,md=1` | Leistung folgt automatisch dem Monitor | ja | - |
| `cd=20,...` | gleiche Parameter wie `cd=07` | nein | ab 214.1 |
| `cd=08,wy=<Offset>,yy=<Jahr-1900>,mm=<0-11>,rr=<1-31>,hh=<0-23>,mn=<0-59>,ss=<0-59>` | Zeit setzen (der Monat zählt ab 0) | - | - |
| `cd=09,wy=<Offset>` | Zeitzonen-Offset setzen (Beispiel: `wy=480`) | ja | - |
| `cd=10` | Neustart | - | - |
| `cd=11` | Werkseinstellung | - | - |

Die Seite nennt den Zeitzonen-Offset `wy=480` als Beispiel (480 sind wohl Minuten östlich von UTC, also +08:00; die Seite sagt es nicht).
Jeder Zeitfenster-Befehl sendet alle drei Fenster in einer Payload; die Seite zeigt die erste, zweite und dritte Periode als getrennte Beispiele.

## 5. Warum das aufgehoben wird

- Es ist eine vom Hersteller geschriebene Beschreibung einer Marstek-MQTT-Payload (andere haben wir nicht gesucht). Sie zeigt die
  Feldbenennung des Herstellers und den Payload-Stil `key=value,...`, den die App für jedes Produkt nutzt.
- Für die **Venus-D**-Arbeit ist sie nur Hintergrund. Die MQTT-Befehle der Venus D stehen im App-Vokabular, abgeleitet aus dem App-Code.
- Offen: ob es für andere Produkte eine gleichwertige Seite gibt. Der Pfad `/ems/mqtt/index.html` ist öffentlich; andere Dateinamen wurden nicht abgefragt.

## 6. Herkunft dieses Dokuments

| Aussage | Grundlage |
|---|---|
| Seiteninhalt, Felder, Befehle, Versionshinweise | von der Seite gelesen |
| URL-Parameter füllen nur Platzhalter | aus dem Seitenskript gelesen (drei Inline-Skripte, eines mit 3639 Zeichen) |
| Inhalt beschreibt B2500 | Hinweise auf der Seite selbst (`HMB-`, `HMA-`, `HMF-`, `HMK-`) |
| Tippfehler in der Beispielzeile | Vergleich der Beispielzeile mit der Feldtabelle, am Gerät nicht getestet |
| Es wurde nichts an ein Gerät gesendet | nur lesender Abruf einer öffentlichen Seite |
