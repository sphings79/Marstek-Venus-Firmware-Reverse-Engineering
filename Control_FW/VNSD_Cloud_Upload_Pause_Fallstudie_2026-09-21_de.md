# Fallstudie: die 4-Sekunden-Pause alle 5 Minuten

> 🇬🇧 English version: [VNSD_Cloud_Upload_Pause_Case_Study_2026-09-21.md](VNSD_Cloud_Upload_Pause_Case_Study_2026-09-21.md)

*Erstellt 2026-09-21. Geraet: Venus D, Control-Firmware v150, LAN, hinter
modbus-proxy. Offline-Endpoint aktiv, also kein 30-Minuten-Reset mehr.
Zugehoerig: `VNSD_Cloud_Watchdog_Fallstudie_2026-08-26_de.md` (der Reset, den der
Endpoint beseitigt hat) und `VNSD_Control_FW_Analyse_app_0150_0805_115146_de.md` §4a.*

Der 30-Minuten-Reset war das grobe Problem. Ist er weg, kommt darunter ein
feineres zum Vorschein: **das Geraet antwortet alle fuenf Minuten rund vier
Sekunden lang nicht.** Diese Datei zeigt, woher das kommt, und was daran kein
Defekt ist.

---

## 1. Beobachtung

Auf der produktiven Home Assistant laufen zwei Integrationen ueber denselben
Proxy gegen dieselbe Batterie: die Marstek Modbus Suite (alle Register
aktiviert) und Omnibattery. In der Nacht lief ein Mitschnitt beider Container.

In 57 Minuten:

| | |
|---|---|
| Requests ueber den Proxy | 18.281 |
| Timeouts der Suite (3 s) | 4, jeder mit Verbindungsneuaufbau |
| `received pdu without a corresponding request` (pymodbus) | 5 |
| `transaction_id mismatch` | 6 |
| Fehler im Proxy | 0 |
| verlorene Werte | 0 |

Die Ausfaelle lagen nicht verstreut, sondern auf einem Raster von rund fuenf
Minuten.

## 2. Die Antwortzeit ist zweigeteilt

Am Proxy gemessen, von `sending` an das Geraet bis `received`:

```
Mittel            81 ms
unter 100 ms    53,7 %
unter 250 ms    46,3 %
250 ms bis 2 s   0 Requests
ueber 2 s       13 Requests   (0,07 %)
```

Keine Verteilung mit Ausläufern, sondern zwei getrennte Welten: entweder unter
einer Viertelsekunde, oder zwei bis vier Sekunden. Die dreizehn Ausreisser:

```
21:35:29  2,25 s     22:02:16  4,29 s     22:22:15  4,20 s
21:37:11  4,17 s     22:07:10  4,37 s     22:27:10  4,14 s
21:42:16  3,91 s     22:12:16  4,26 s     22:32:16  4,12 s
21:47:10  4,11 s     22:17:11  3,44 s
21:52:18  2,37 s     (alle Zeiten UTC)
21:57:11  3,27 s
```

Abstaende abwechselnd 294 und 306 Sekunden, im Mittel exakt 300.

## 3. Die Konstante in der Firmware

`HTTP_Economy_TOU_PeriodicHandler` @ `0x080169b4`, letzter Block:

```c
if (((*(char *)(DAT_08016b20 + 0x11) == '\0') && (*(short *)(DAT_08016b20 + 2) != 0)) &&
   (iVar3 = Tick_Timer_Check_Elapsed(DAT_08016b94, 300), iVar3 != 0)) {
    *(undefined1 *)(DAT_08016b20 + 0x11) = 1;   // Upload angefordert
    *(undefined1 *)(iVar2 + 0x10) = 3;          // drei Versuche
}
```

**300 Sekunden.** Die Einheit ist ueber `0x708` gesichert, das in derselben
Firmware als 1800 Sekunden belegt ist (§4a). Bedingung ist ein Datensatz im
Ringpuffer (`+2 != 0`), erzeugt beim Slotwechsel ueber
`RTC_TimeOfDay_To5MinSlotIndex` @ `0x08014168`.

Es ist ein **relativer** Timer, kein Wanduhr-Raster: er wird nach jedem Feuern
neu gestellt und nur abgefragt, wenn die Zustandsmaschine des Cloud-Tasks
vorbeikommt. Daher 294/306 statt starr 300.

## 4. Was daran vier Sekunden dauert

Die Kette vom Flag bis zum Draht:

| Schritt | Adresse |
|---|---|
| 300-s-Timer setzt Upload-Flag | `HTTP_Economy_TOU_PeriodicHandler` @ `0x080169b4` |
| Cloud-Task, `case '\x01'` | `FUN_08015bd0` |
| Datensatz aufbereiten und senden | `Cloud_Report_MarstekCloud_Upload` @ `0x08016b98` |
| HTTPS ueber den Ethernet-Chip | `HTTPS_Request_Execute_CH395` @ `0x080174e8` |
| TLS-Sitzung **von Null** | `HTTPS_TLS_Session_Init` @ `0x080370a0` |

Die Sitzung wird jedes Mal komplett neu gebaut:

```c
HTTPS_TLS_Context_Reset();
mbedTLS_RSA_Alt_InitOnce();
mbedTLS_CTR_DRBG_Init(...);   mbedTLS_Entropy_Init(...);
mbedTLS_CTR_DRBG_Seed(...);   // Entropie neu sammeln
mbedTLS_SSL_Setup(...);
mbedTLS_SSL_Handshake(...);   // voller TLS-1.2-Handshake
```

Kein Session-Resume, kein Keep-Alive. Davor noch der Socket-Aufbau mit einer
Warteschleife `vTaskDelay(10)` bis `500`. Welcher Teil davon die vier Sekunden
kostet, steht in §5a — gemessen, nicht geschaetzt.

**Warum Modbus mit haengt:** derselbe Chip. `CH395_Init_Modbus_TCP_Socket` @
`0x080494d0` oeffnet Port `0x1f6` = 502 auf dem CH395, und jedes SPI-Kommando zu
diesem Chip laeuft durch einen Mutex (`CH395_SPI_Cmd_WithResponse` @
`0x08002cd4`, `xQueueReceive(..., 5000)`). Solange der Cloud-Pfad den Chip
beschaeftigt, liest niemand die Modbus-Sockets aus. Die TCP-Verbindung bleibt
dabei stehen — der CH395 haelt sie —, es antwortet nur keiner.

## 5. Der Beweis von aussen

Der Offline-Endpoint auf dem Raspberry Pi protokolliert jeden Request mit
ISO-Zeitstempel (`/data/requests-<datum>.jsonl`). Links die Messung am Proxy,
rechts der POST am Endpoint:

```
Pause beginnt   still   Geraet antwortet   Upload-POST      Abstand
   21:37:11    4,17 s     21:37:15,2      21:37:16,366      1,20 s
   21:42:16    3,91 s     21:42:19,9      21:42:21,022      1,11 s
   21:47:10    4,11 s     21:47:14,1      21:47:15,283      1,17 s
   21:52:18    2,37 s     21:52:20,4      21:52:21,278      0,91 s
   21:57:11    3,27 s     21:57:14,3      21:57:15,258      0,99 s
   22:02:16    4,29 s     22:02:20,3      22:02:21,297      1,01 s
   22:07:10    4,37 s     22:07:14,4      22:07:15,309      0,94 s
   22:12:16    4,26 s     22:12:20,3      22:12:21,200      0,94 s
   22:17:11    3,44 s     22:17:14,4      22:17:15,351      0,91 s
   22:22:15    4,20 s     22:22:19,2      22:22:20,145      0,95 s
   22:27:10    4,14 s     22:27:14,1      22:27:15,265      1,12 s
   22:32:16    4,12 s     22:32:20,1      22:32:21,303      1,18 s
   22:46:51    3,90 s     22:46:54,9      22:46:55,887      0,99 s
```

Dreizehn Ereignisse, dreizehn Uploads, Abstand jedes Mal zwischen 0,91 und
1,20 Sekunden. Die Upload-Abstaende an der Uhr des Pi: 295, 305, 294, 306, 294,
306, 294, 306, 294, 305, 295, 306, 295, 305 — dasselbe alternierende Muster wie
in den Modbus-Daten.

## 5a. Wo die vier Sekunden liegen

Mitschnitt auf dem Pi, nur Header (`tcpdump -s 96`), gefiltert auf die Batterie
und Port 443. Eine Verbindung, Zeiten relativ zum SYN:

```
 +0,000 s  client  [S ]      0 B   SYN
 +0,000 s  server  [S.]      0 B   SYN-ACK          TCP steht nach 1 ms
 +0,041 s  client  [P.]     78 B   ClientHello
 +0,066 s  server  [. ]   1024 B   ServerHello, Certificate,
 +0,066 s  server  [P.]    315 B     ServerKeyExchange, HelloDone
 +0,300 s  server  [. ]   1024 B   Rest der Serverflug
 +0,516 s  client  [. ]      0 B   ACK
                                   -- 4,22 s Stille --
 +4,740 s  client  [P.]     75 B   ClientKeyExchange
 +4,784 s  client  [P.]     51 B   ChangeCipherSpec, Finished
 +4,786 s  server  [P.]     51 B   ChangeCipherSpec, Finished
 +4,982 s  client  [P.]    988 B   POST (verschluesselt)
 +4,986 s  server  [P.]    511 B   Antwort
 +25,06 s  client  [F.]      0 B   FIN
+294,02 s  client  [S ]      0 B   naechster Upload
```

**Die Pause liegt zwischen der Serverflug und dem ClientKeyExchange.** Das
Geraet hat alles, was es braucht, und schweigt 4,22 Sekunden — das ist reine
Rechenzeit auf dem Cortex-M4: Zertifikat parsen, Signatur der
ServerKeyExchange pruefen, ECDHE-Schluessel erzeugen und das gemeinsame
Geheimnis rechnen. Netz und Server haben daran keinen Anteil; der Pi antwortet
in Millisekunden.

Alles andere ist vernachlaessigbar: TCP 1 ms, ClientHello 41 ms, Serverflug
260 ms, ab Finished bis zur Antwort 250 ms.

Gegenprobe ueber beide Enden: zu dieser Verbindung (Start 01:12:16,341 lokal)
gehoert am Proxy eine Modbus-Pause von **4,32 s um 23:12:16 UTC**. Die naechste
Verbindung, 294 s spaeter, liefert dieselbe Rechenpause und am Proxy 3,27 s —
kuerzer nur deshalb, weil der Modbus-Request erst mitten im Fenster ankam.

Nebenbefund: die Verbindung bleibt nach der Antwort noch **20 Sekunden** offen,
bevor das Geraet FIN schickt. Wiederverwendet wird sie nicht — die naechste
Sitzung faengt wieder bei Null an.

## 6. Die Kontrolle: HTTP kostet nichts

Derselbe Endpoint beantwortet alle zehn Minuten eine Zeitabfrage, ueber **HTTP
auf Port 80, ohne TLS**:

```
21:45:58, 21:55:59, 22:06:00, 22:16:01, 22:26:02, 22:36:03, 22:46:04
```

Zu keinem dieser Zeitpunkte gibt es eine Pause. Gleicher Chip, gleiche Task,
gleiche Socket-Maschinerie, gleicher Mutex — nur ohne Handshake, und es ist
nichts messbar.

Damit ist die Ursache eingegrenzt auf das Aufsetzen der TLS-Sitzung, nicht auf
den CH395, nicht auf das Netz und nicht auf den Endpoint.

## 7. Konsequenz fuer die Integration

Die Modbus Suite wartete 3 Sekunden und baute nach jedem Timeout die Verbindung
neu auf:

```
00:07:13  Rebuilding the connection after a timeout on block read
00:07:14  Reconnected
```

Am 21.09.2026 um 00:43 auf **5 Sekunden** angehoben (`DEFAULT_TIMEOUT`,
`const.py`; ein Config-Entry-Reload genuegt nicht, Python hat das Modul
gecacht — Core-Neustart noetig). Die erste Pause danach, 00:46:51 mit 3,90 s,
lief ohne Timeout und ohne Reconnect durch.

In den 25 Minuten danach: fuenf Pausen (3,09 bis 4,34 s), **null Timeouts, null
Reconnects**. Vorher haette jede davon einen gekostet.

Fuenf Sekunden decken alles bisher Gemessene ab; der bisher groesste Wert
ueberhaupt war 4,77 s in einer Messung vom Vorabend. Der Rand ist also duenn,
sechs Sekunden waeren die ruhigere Wahl.

### Omnibattery ist nicht betroffen — und das ist kein Zufall

Naheliegend waere, dass der Verbindungsneuaufbau der Suite die zweite
Integration mitreisst. Die Zahlen sagen nein: `received pdu without a
corresponding request` und `transaction_id mismatch` kommen vor der Aenderung
mit 0,19 pro Minute und danach mit 0,20 pro Minute — also unveraendert, obwohl
die Suite nicht mehr neu verbindet. Sie haengen an den Pausen, nicht an der
Suite: eine Meldung pro Pause.

Omnibattery ist auf genau diesen Fall ausgelegt. `READ_TIMEOUT_S` steht fuer die
v3-Familie auf 3 Sekunden, mit zwei pymodbus-internen Wiederholungen, und der
Kommentar im Quelltext benennt den Mechanismus:

> occasionally stalls for several seconds and then flushes queued replies in a
> burst. Each pymodbus-internal retry re-sends the SAME transaction_id, so a
> reply arriving after a per-attempt timeout still matches the retry and is
> consumed. Keep v3 attempts short so the retries land inside the stall window.

Die Wiederholung faengt die verspaetete Antwort also auf. Was danach als
`without a corresponding request` im Log steht, ist die **zweite** Antwort auf
dieselbe Transaktion — ein Duplikat, kein Verlust. Passend dazu: in 57 Minuten
kein einziger unvollstaendiger Poll und kein fehlender Wert.

Zwei Integrationen, zwei Antworten auf dasselbe Firmware-Verhalten: kurze
Versuche mit gleicher Transaktionsnummer (Omnibattery) oder ein Timeout
oberhalb der Pause (Suite). Beides funktioniert; das zweite ist leiser.

## 8. Offene Punkte

1. **Die 2,25-s-Luecke um 21:35:29** hat keinen zugehoerigen Upload. Sie liegt
   24 Sekunden nach Beginn des Mitschnitts; vermutlich ein Artefakt der
   Paarbildung am Dateianfang. Nicht weiter verfolgt, nicht wegerklaert.
2. **Drei Versuche.** Das Flag wird mit `+0x10 = 3` gesetzt. Was passiert, wenn
   ein Upload scheitert und zweimal nachgelegt wird, ist nicht gemessen — die
   Pause duerfte sich dann verlaengern.
3. **Die 60-Sekunden-Drossel.** Stauen sich mehr als drei Datensaetze, erlaubt
   `case '\x01'` einen Upload pro 60 s (§6 der Watchdog-Fallstudie). Aus dem
   5-Minuten-Aussetzer wuerde ein Minutentakt. Bisher nicht beobachtet, weil der
   Endpoint den Puffer leer haelt.
