# Netzabrisse während des Registerscans

> 🇬🇧 English version: [VNSD_Network_Dropouts_During_Register_Scan_2026-09-20.md](VNSD_Network_Dropouts_During_Register_Scan_2026-09-20.md)

**Gerät:** Marstek Venus D (VNSD-0), `192.168.1.50:502`, direkt, ohne Proxy
**Firmware:** Control/EMS v150
**Datum:** 20. September 2026
**Status:** beobachtet und eingegrenzt · **Ursache offen** · Gegentest bei geringer Last steht aus

---

## 1. Was passiert ist

Während eines Vollscans aller Halteregister (Funktionscode 03, eine Adresse je Anfrage,
rund 6,5 Anfragen pro Sekunde) hat das Gerät dreimal die Verbindung verloren. Zweimal nach
demselben Muster:

```
43668  exception  illegal_data_address  150,0 ms   <- letzte Antwort des Geräts
43669  error      BrokenPipeError         0,1 ms   <- send() scheitert
43670  —          No route to host                 <- Gerät nicht mehr erreichbar
```

`BrokenPipeError` beim **Senden** heißt: die Gegenstelle hatte die TCP-Verbindung bereits
geschlossen, bevor die nächste Anfrage rausging. Das Gerät hat also aktiv aufgelegt, es ist
kein Timeout und kein Paketverlust.

Danach war es nicht nur auf Port 502 still, sondern auf IP-Ebene weg: `EHOSTUNREACH`. Ein
Gerät, das nur seinen Modbus-Server verloren hätte, würde stattdessen `ECONNREFUSED` liefern
oder weiter auf ICMP antworten.

## 2. Die drei Störungen

| Nr. | Register | Art | Letzte Antwort davor | Ausfalldauer |
|---|---|---|---|---|
| 1 | **32103** | Timeout, 10 s, keine Antwort | 32102, Exception, 150 ms | kurz, danach Session belegt |
| 2 | **40897** | `BrokenPipeError`, dann `EHOSTUNREACH` | 40896, Exception, 150 ms | ~2 h 30 min (13:52 – 16:21) |
| 3 | **43669** | `BrokenPipeError`, dann `EHOSTUNREACH` | 43668, Exception, 150 ms | ~1 h (16:30 – 17:30) |

Störung 1 ist wahrscheinlich anderer Natur: davor war ein Scan-Prozess hart beendet worden,
das Gerät hielt vermutlich noch die alte Session. Das Gerät lässt **genau eine** Modbus-TCP-
Session zu. Sie ist hier nur der Vollständigkeit halber aufgeführt.

Auslöser im Sinne von „die Anfrage, die noch beantwortet wurde":

```
40896 = 0x9FC0        43668 = 0xAA94        Abstand 2772
```

Beide lieferten noch eine reguläre Exception 2 *illegal data address*.

### Anfragen zwischen den Störungen

| Abschnitt | Anfragen | davon Exceptions | Werte |
|---|---|---|---|
| 1 – 32102 | 32102 | 32017 | 85 |
| 32103 – 40896 | 7779 | 7469 | 309 |
| 40897 – 43668 | 2772 | 2684 | 87 |

Die Abschnitte werden kürzer. Für einen festen Schwellwert — etwa „nach N Exceptions ist ein
Puffer voll" — reicht das nicht: 32017, 7469 und 2684 liegen zu weit auseinander.

### Antwortzeiten

Jede abgelehnte Adresse wird auf **exakt 150,0 ms** beantwortet, über zehntausende Anfragen
hinweg, bis unmittelbar vor dem Abriss. Keine Verschlechterung, kein Anstieg, kein Jitter. Das
Gerät zeigt vorher keinerlei Belastungssymptome.

---

## 3. Was die Firmware dazu sagt — nichts

Geprüft im D-Image v150:

**Die beiden Auslöseregister sind nicht besonders.** Der Lesepfad delegiert für alles ab 40000
an `Write_Handler(buf, reg, 0)` (`FC03_Read_Handler` @ 0x0801F06C, Zeile 62 ff.). Dessen
Bereichsgrenzen für `param_2` lauten:

```
40000 41000 41100 41200 41201 41500 41501 41502 41503 41511 41512
41600 41604 41610 41611 41631 42011 42020 43100 43101 43102 43103
43118 45000 45603 45604 46499 46500 46545 47400
```

**40896 liegt zwischen 40000 und 41000, 43668 zwischen 43118 und 45000** — beide mitten in
einer unbelegten Lücke, an keiner Grenze, in keinem Sonderzweig. In denselben Lücken haben
tausende Nachbaradressen dieselbe Exception ohne jeden Zwischenfall geliefert.

**Der Empfangspfad rechnet korrekt.** `Modbus_Dispatcher` liest 7 Byte MBAP, entnimmt das
Längenfeld, liest `len − 1` weitere Byte und zieht `len + 6` vom Füllstand ab — das ist
stimmig, hier läuft nichts aus dem Ruder.

**Der einzige bekannte Sonderfall im Lesepfad ist ein anderer Bereich.** Zeile 30 des
`FC03_Read_Handler`:

```c
if (((37999 < uVar7) && (uVar7 < 0x9867)) && (*DAT_0801f1ec == '\0')) {
    *DAT_0801f1ec = '\x01';
}
```

38000 – 39014 setzt ein Flag. Dieser Bereich wird vom Scanner **ausgelassen** und kommt als
Erklärung nicht in Frage.

Es gibt damit **keinen Firmware-Befund, der die beiden Adressen auszeichnet.** Wenn ein
Zusammenhang besteht, dann kumulativ über die Zahl der gesendeten Exception-Frames, nicht über
die konkrete Registernummer. Diese Frames sind die, deren MBAP-Längenfeld um eins zu groß ist
(`VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md`) — ein naheliegender Verdacht, aber die
Abschnittslängen oben stützen ihn nicht.

---

## 4. Was gegen ein reines Umgebungsproblem spricht, und was dafür

**Dafür:** der messende Mac hat **zwei Interfaces im selben Subnetz**, `en0 = 192.168.1.71`
und `en1 = 192.168.1.70`, beide mit ARP-Eintrag für das Gerät. Doppelte Anbindung an dasselbe
L2-Segment erzeugt asymmetrisches Routing und ist eine klassische Quelle sporadischer
`EHOSTUNREACH`. Das gehört bereinigt, bevor man dem Gerät etwas anlastet.

**Dagegen:** im gesunden Zustand ist nichts davon zu sehen. Gegenprobe am 20.09. um 17:27,
während der Scanner noch `No route to host` meldete:

- ICMP: 10 von 10 Paketen, 0,3 ms, kein Verlust
- TCP-Connect auf Port 502, je vier Versuche **von jeder der beiden Quelladressen**: 12 von 12
  erfolgreich

Und der Abriss selbst beginnt jedes Mal mit einem `BrokenPipeError` — also damit, dass das
**Gerät** die bestehende Verbindung schließt. Das erklärt kein Routing-Problem des Messrechners.

---

## 5. Offen, und wie es zu klären wäre

Die Beobachtung ist belastbar, die Ursache nicht. Drei Hypothesen, keine bestätigt:

1. **Kumulative Erschöpfung im Netzwerk-Stack** unter Dauerlast — CH395-Socket, Puffer,
   Deskriptoren. Passt zum aktiven Schließen und zur langen Ausfalldauer, passt nicht zu den
   stark schwankenden Abschnittslängen.
2. **Registerabhängiger Auslöser.** Durch Abschnitt 3 stark geschwächt: in der Firmware ist an
   den beiden Adressen nichts.
3. **Umgebung** (Abschnitt 4). Durch die Gegenprobe geschwächt, aber nicht ausgeschlossen.

### Geplanter Gegentest

Nach Abschluss des Vollscans: **40891 – 40901 und 43663 – 43673**, also ±5 Register um beide
Auslöser, bei **geringer Last** — eine Anfrage alle paar Sekunden statt 6,5 pro Sekunde, über
mehrere Durchläufe.

- Tritt der Abriss auch dort auf, ist es registerabhängig und Hypothese 2 lebt wieder.
- Tritt er nicht auf, spricht das für Hypothese 1, und der nächste Schritt wäre ein Lauftest
  mit hoher Rate auf einer **unverfänglichen** Adresse, um die Last vom Register zu trennen.

Vorher sinnvoll: die zweite IP im Subnetz am Messrechner abschalten.

### Neustart oder nicht — offen, und die Indikatoren taugen nicht

Der Schattenwert hinter 44002/44003 lag um 12:00 kontrolliert auf 2500 und liest um 17:38
wieder 0:

| Zeit | 44002 | 44003 | Quelle |
|---|---|---|---|
| 11:46 | 0 | 0 | nach dem vom Nutzer ausgelösten Neustart |
| ~12:00 | 2500 | 2500 | Ende des A/B-Tests, Rücklesewert kontrolliert |
| 17:38 | 0 | 0 | laufender Vollscan |

Daraus wurde hier zunächst geschlossen, das Gerät habe neu gestartet. **Der Schluss trägt
nicht.** Der Schatten bei `0x20000156` wird nicht nur beim Start genullt, sondern auch von
`Protocol_AA_SetExtendedParams` aus einem CAN-Frame gefüllt, und die Aufrufkette dorthin ist

```
CAN_FrameDispatcher → Telemetry_Register_Dispatcher (Kommando 0x14) → Protocol_AA_SetExtendedParams
```

Ein CAN-Frame mit Nullen ist genau in einer Störung — wenn der Wechselrichter oder das Control
durchstartet — nicht unwahrscheinlich, sondern erwartbar. Restart und CAN-Nullung sind an
diesem Register **nicht unterscheidbar**.

Zwei weitere Kandidaten wurden geprüft und fallen ebenfalls aus:

- **Home Assistant** hat keine Daten. Die Integration steht seit 13:30 still, also seit der
  Proxy für den Scan abgeschaltet wurde — und damit vor dem ersten Abriss um 13:52.
- **Register 42000** liest jetzt `0x55BB` statt des um 12:00 geschriebenen `0x55AA`. Der
  dahinterliegende Zustand ist aber das Work-Mode-Byte bei `0x20014D01`, und das hat viele
  Schreiber: `MQTT_Config_Command_Handler`, `MQTT_JSON_RPC_Dispatcher`, `MQTT_SetWorkMode`,
  `MQTT_PassiveModeHandler`, `BLE_Cmd_Dispatch`, der Config-Loader und der `Write_Handler`.
  Beweiskraft null.

  *Nebenbefund:* dieser Zweig (`uVar8 == 0x186` bei Basis `0xA28A`) gehört zu **42000**, nicht
  zu 43492, wie die Registerkarte im Funktionskopf des `Write_Handler` behauptet. 43492 liefert
  im Scan eine Exception.

**Es ist also weiterhin offen, ob das Gerät neu gestartet ist.**

### Ein Kanarienvogel, der es entscheiden würde

Gesucht ist ein Register im BSS, das ausschließlich ein Modbus-Schreibzugriff setzt. Die
Kreuzreferenzen auf den SRAM-Block `0x20000348` liefern genau das:

| Register | Adresse | schreibend | lesend |
|---|---|---|---|
| 42010 force_mode | 0x20000348 | `Write_Handler`, `Modbus_Dispatcher`, `RS485_RTU_Frame_Dispatcher` | dieselben |
| 42011 charge_target_soc | 0x2000034A | **nur `Write_Handler`** | `Remote_Power_Command_Execute` |
| 42020 charge_power_limit | 0x2000034C | **nur `Write_Handler`** | `Remote_Power_Command_Execute` |
| **42021 discharge_power_limit** | **0x2000034E** | **nur `Write_Handler`** | `Remote_Power_Command_Execute` |

Kein CAN, kein MQTT, kein BLE, kein Restore aus dem EEPROM. Fällt ein hineingeschriebener Wert
auf 0, kann das **nur** die BSS-Initialisierung beim Start gewesen sein.

`42021` ist dabei der unbedenklichste: `Remote_Power_Command_Execute` wertet ihn nur bei
aktivem `force_mode` aus. Steht 42010 auf 0 (Standby), ist der Schreibzugriff folgenlos, und
anders als 44002 fasst er weder das EEPROM noch den Wechselrichter an.

Deshalb verwendet `Scripts/dropout_retest.py` (lokal, nicht im Repo) **42021** als Indikator, nicht 44002.

---

## 6. Auswirkung auf den Scanner

`Scripts/register_scan.py` (lokal, nicht im Repo) wurde daraufhin zweimal nachgebessert:

1. Der Reconnect im Timeout-Zweig war ungesichert — eine abgelehnte Verbindung hat den
   gesamten Lauf beendet. Jetzt mit Backoff und abgefangen.
2. Eine verlorene Session lässt dieselbe Adresse erneut versuchen, statt den Rest des Scans als
   Fehler durchzurattern. Die Wartezeit wächst von 30 s auf 5 min, bis zu 60 Versuche, also
   rund vier Stunden Geduld. Die CSV bleibt dabei jederzeit gültig.
