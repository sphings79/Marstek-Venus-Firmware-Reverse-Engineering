# Modbus TCP gibt es nur über die LAN-Buchse, der Discovery-Beacon über beide

> 🇬🇧 English version: [VNSD_Modbus_LAN_Only.md](VNSD_Modbus_LAN_Only.md)

**Gerät:** Marstek Venus D (VNSD-0), Kommunikationsmodul HME-3
**Firmware:** Control/EMS v150
**Pfad:** Modbus **TCP** (CH395) und UDP-Beacon 22222 → 12345
**Datum:** 22. September 2026
**Status:** am Gerät gemessen, vier Netzzustände inkl. Kaltstart · Firmware-Begründung aus
`Methodik_und_Meta/VNSD_Ghidra_Analyse_Erkenntnisse_de.md` §5.1

**Geltungsbereich:** ausschließlich am Venus D gemessen. Ob Venus A und E v3 sich genauso
verhalten, ist **nicht** geprüft — daher das Präfix im Dateinamen.

---

## 1. Kurzfassung

Ein Venus D mit Netzwerkkabel hat zwei IP-Adressen: eine am Ethernet-Chip CH395 und eine am
Funkmodul HME-3. Die beiden verhalten sich unterschiedlich:

- **Modbus TCP antwortet nur auf der LAN-Adresse.** Auf der WLAN-Adresse wird Port 502
  abgelehnt — im laufenden Betrieb, nach Verlust des Links und auch nach einem **Kaltstart
  ohne Kabel**. Der Server wandert nicht mit.
- **Der Discovery-Beacon wandert sehr wohl mit.** Er kommt immer aus genau einem Interface,
  bevorzugt LAN, und wechselt bei Linkverlust bzw. Linkgewinn innerhalb von rund einer halben
  Minute. Nie aus beiden gleichzeitig.

Praktische Folge: eine Autodiscovery über den Beacon findet auch ein Gerät **ohne** Kabel und
liefert dafür eine Adresse, hinter der kein Modbus horcht.

---

## 2. Die Adressen

| | LAN | WLAN |
|---|---|---|
| IP | 192.168.1.50 | 192.168.1.51 |
| MAC | `aa:bb:cc:dd:ee:02` | `aa:bb:cc:dd:ee:03` |
| DHCP-Hostname | `CH395` | — |
| Chip | CH395 (Ethernet) | HME-3 / FC41D (Funk) |

Der DHCP-Hostname der LAN-Seite lautet wörtlich `CH395` — im UniFi-Controller direkt ablesbar
und damit ein brauchbarer Filter, falls jemand eine DHCP-basierte Erkennung bauen will.

Die im Beacon transportierte MAC `aa:bb:cc:dd:ee:01` teilt sich den OUI-Block `aa:bb:cc` mit
der WLAN-MAC. Die Nutzlast stammt also vom Funkmodul, wird aber über das gerade aktive
Interface verschickt — bei gestecktem Kabel über den CH395.

---

## 3. Das Messprotokoll

Passiv mitgehört wurde auf UDP 12345 ohne jede Entdopplung, jedes einzelne Paket gezählt.
Die Port-Prüfungen sind reine TCP-Connects ohne Modbus-PDU.

### 3.1 Beide Interfaces oben

```
Beacons, 15 s:   192.168.1.50  x30
                 192.168.1.51  x0
192.168.1.50:502  OPEN
192.168.1.51:502  ConnectionRefusedError
```

### 3.2 LAN-Link im Betrieb verloren

```
Beacons, 90 s:   192.168.1.51  x229
192.168.1.51:502  ConnectionRefusedError   (5 Messungen über 84 s)
192.168.1.50      100 % Paketverlust
```

### 3.3 Kaltstart ohne LAN

```
Beacons, 60 s:   192.168.1.51  x166
192.168.1.51:502  ConnectionRefusedError   (3 Messungen)
```

Damit ist ausgeschlossen, dass der Modbus-Server nur deshalb nicht auf dem WLAN-Stack liegt,
weil er beim Booten schon an den CH395 gebunden war.

### 3.4 LAN kommt zurück

```
[  0s] Beacon-Quelle -> 192.168.1.51
[ 26s] Beacon-Quelle -> 192.168.1.50
Gesamt über 5 min: {'192.168.1.51': 15, '192.168.1.50': 538}
```

Ein einziger Wechsel, danach keiner mehr. Keine Überlappung, keine Phase mit beiden Quellen.

### 3.5 Gegenprobe mit garantiert freiem Steckplatz

Der Venus nimmt genau **eine** Modbus-Verbindung an. Alle vorherigen Messungen liefen gegen
ein Gerät, dessen Verbindung ein `modbus-proxy` hielt. Für die Gegenprobe wurde der Proxy
gestoppt:

```
192.168.1.50:502  OPEN                    (2x, sofort)
192.168.1.51:502  ConnectionRefusedError  (2x)

FC03 gegen 192.168.1.50:
  30400 device_ip   -> 192.168.1.50
  30402 gateway_ip  -> 192.168.1.1
  32104             -> 54
```

Echte FC03-Antworten, nicht nur ein offener Port. **Register 30400 meldet die LAN-Adresse** —
das Gerät nennt selbst die Adresse, unter der es Modbus spricht.

---

## 4. Warum: die Socket-Belegung des CH395

Aus `Methodik_und_Meta/VNSD_Ghidra_Analyse_Erkenntnisse_de.md` §5.1, Initialisierung in
`CH395_Reset_And_Reinit @ 0x08029964`:

| Socket | Port | Funktion | Init |
|---|---|---|---|
| 0 | 502 | **Modbus TCP** | `CH395_Init_Modbus_TCP_Socket @ 0x08048754` |
| 1 | 8091 | TCP Debug/CLI Shell | `CH395_Init_TCPServer_Socket @ 0x08032C0C` |

Der Modbus-Server ist fest an den Ethernet-Chip gebunden. Auf dem Quectel-/AT-Pfad des
Funkmoduls existiert keine Entsprechung. Das deckt sich mit der Messung und erklärt auch,
warum ein Kaltstart ohne Kabel daran nichts ändert.

**Der Beacon dagegen ist damit nicht erklärt.** Die naheliegende Kandidatenfunktion
`FUN_08017dc0` (Quellport 22222, Zielport 12345, CH395-UDP-Socket, siehe
`Control_FW/VNSD_Control_FW_Analyse_app_1492_0702_142136_de.md` Zeile 1834) gehört zum
EM-Status-Poll, der `EM.GetStatus` an ein lokales Energiemessgerät schickt — und dieses
Feature hat in v149.2 in **beiden** Varianten, Ethernet und Quectel, null Caller und ist
unerreichbar. Wer den Beacon tatsächlich sendet, ist offen.

---

## 5. Methodischer Fallstrick: `ConnectionRefused` ist zweideutig

Ein RST auf Port 502 kann zwei völlig verschiedene Dinge bedeuten:

1. **Kein Listener** — die Adresse gehört zum falschen Interface.
2. **Steckplatz belegt** — das Gerät nimmt genau eine Verbindung an, und ein Proxy oder eine
   zweite Integration hält sie bereits.

Beide erzeugen dasselbe Paket. Während dieser Messreihe lieferte die **LAN**-Adresse zeitweise
`ConnectionRefused`, obwohl Home Assistant in derselben Sekunde erfolgreich las — Fall 2.

Deshalb trägt die Aussage in diesem Dokument nicht am Port-Scan allein, sondern an zwei
unabhängigen Zuständen mit nachweislich freiem Steckplatz: §3.2/§3.3 (LAN unerreichbar, also
konnte niemand die Verbindung halten) und §3.5 (Proxy gestoppt).

Wer künftig so misst: gegen den `modbus-proxy` lesen statt gegen das Gerät. Der lässt mehrere
Clients zu und stört die laufende Abfrage nicht.

---

## 6. Nebenbefund: Register 30303 (`wifi_signal_strength`)

Laut `BLE/VNSD_BLE_Modbus_CrossReference_de.md` Zeile 42 ist 30303 das Feld `wifiRssi` (u8, Quelle
`DAT_0800b38c[1]`), Anzeige als Rohwert × −1.

Beobachtet wurde das Gegenteil des Erwarteten:

| Zustand | 30303 |
|---|---|
| WLAN **nicht** verbunden, Gerät lief lange am Kabel | 40 → −40 dBm |
| WLAN verbunden, Gerät nach Neustart am Kabel | 0, stabil über 3 Lesungen |

**Arbeitshypothese:** der Wert wird nur nachgeführt, wenn das Funkmodul die aktive Uplink-
Strecke ist. Am Kabel bleibt er auf dem letzten bekannten Stand stehen und ist nach einem
Neustart schlicht nie befüllt worden. Die −40 vor dem Neustart wären dann ein Altwert aus
einer früheren WLAN-Phase.

Die Hypothese ist mit Modbus allein **nicht prüfbar**: um 30303 zu lesen, braucht man LAN —
und mit LAN ist das Funkmodul nicht der Uplink. Ein Zangengriff. Prüfbar wäre es über BLE
(Feld 0x3D) an einem Gerät ohne Kabel.

Praktische Folge: der Sensor `wifi_signal_strength` liefert an einem verkabelten Venus D
entweder 0 oder einen Altwert. Er ist derzeit `enabled_by_default: true`.

---

## 7. Folgen für die HA-Integration

Die Autodiscovery der Marstek Modbus Suite lauscht auf denselben Beacon. Aus den Messungen:

- **Bei gestecktem Kabel ist die gefundene Adresse immer die richtige.** Es funkt nur das
  LAN-Interface, und genau das trägt Modbus. Ein aktives Abklopfen mehrerer gefundener
  Adressen wäre wirkungslos — und schädlich, weil jeder Connect gegen den einen Steckplatz
  des Geräts geht.
- **Ohne Kabel wird das Gerät gefunden, ist aber nicht einrichtbar.** Der Dialog bietet eine
  Adresse an, die Port 502 ablehnt. Behandelt in `config_flow.py` über
  `async_is_connection_refused`: nach einem gescheiterten Verbindungsversuch wird einmal
  geprüft, ob das Gerät ablehnt statt zu schweigen, und der Fehlerschlüssel
  `connection_refused` nennt dann beide möglichen Ursachen aus §5.
- **Zwei Beacons für dasselbe Gerät gibt es nicht**, auch nicht während des Umschaltens. Die
  Entdopplung in `helpers/discovery.py` läuft trotzdem über `(MAC, Host)` statt über die MAC
  allein: die MAC gehört dem Funkmodul und ist auf beiden Interfaces dieselbe, sodass ein
  MAC-Schlüssel bei geänderter Firmware still die zuerst eingetroffene Adresse behielte.

---

## 8. Offen

- Wer sendet den Beacon? `FUN_08017dc0` ist es nicht (§4).
- Gelten §1–§4 auch für Venus A und Venus E v3? Beide teilen die Control-Firmware-Familie,
  gemessen wurde aber nur der D.
- Bedeutung von Register 30304 (stabil 13880 über alle Lesungen).
- Bestätigung der Hypothese zu 30303 über BLE-Feld 0x3D.
