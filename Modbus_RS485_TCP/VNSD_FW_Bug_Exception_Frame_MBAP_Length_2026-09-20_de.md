# FW-Bug: Exception-Frame deklariert eine Länge zu viel (MBAP off-by-one)

> 🇬🇧 English version: [VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md](VNSD_FW_Bug_Exception_Frame_MBAP_Length_2026-09-20_de.md)

**Gerät:** Marstek Venus D (VNSD-0), Nations N32G45x (Cortex-M4F) — nicht AT32F403A, siehe Abschnitt 10
**Firmware:** Control/EMS **v150** (`VNSD-0_app_0150_0805_115146.bin`)
**Pfad:** Modbus **TCP** (CH395), nicht RS485-RTU
**Betroffene Funktion:** `Modbus_Dispatcher` @ `0x08023204` (korrekte Basis 0x08004800), Exception-Zweig ab
`0x080232e8` — ursprünglich als `0x0801ea04` / `0x0801eae8` notiert (alte Ghidra-Analyse mit falscher Basis
0x08000000, Differenz durchgängig 0x4800)
**Datum:** 20. September 2026
**Status:** Statik byte-genau belegt · **am Gerät bestätigt** (Abschnitt 5)

---

## 1. Kurzfassung

Die Firmware baut für abgelehnte Requests einen Modbus-TCP-Exception-Frame und sendet ihn.
Der Frame ist 9 Byte lang und bis auf ein Feld korrekt: das **MBAP-Längenfeld enthält 4,
richtig wäre 3**. Ein konformer Client wartet dadurch auf ein Byte, das nie kommt, und läuft
in seinen Timeout. Die Exception ist für ihn nicht von „Gerät antwortet nicht" zu unterscheiden.

Praktische Folge: das Lesen eines nicht implementierten Registers kostet einen vollen
Client-Timeout statt einer sofortigen Absage.

---

## 2. Der Code

Exception-Zweig in `Modbus_Dispatcher`, erreicht wenn ein FC-Handler einen Fehlercode ≠ 0
zurückgibt:

```c
else {
  DAT_0801eb98[4] = 0;              /* MBAP-Länge Highbyte */
  puVar1[5] = 4;                    /* MBAP-Länge Lowbyte  <-- FEHLER, muss 3 sein */
  puVar1[7] = puVar1[7] + -0x80;    /* FC | 0x80, in 8 Bit korrekt: 0x03 -> 0x83 */
  puVar1[8] = *DAT_0801eb9c;        /* Exception-Code */
  CH395_SPI_Send_Data(0,puVar1,9);  /* 9 Byte */
}
```

Assembler, byte-genau aus dem Image gelesen (kein Decompiler-Artefakt), korrekte Basis 0x08004800
(ursprünglich mit falscher Basis 0x08000000 als `0x0801eaee` ff. notiert, hier durchgängig +0x4800):

```
080232ee  00 20        movs r0, #0
080232f0  29 49        ldr  r1, [pc, #0xa4]
080232f2  08 71        strb r0, [r1, #4]     ; Länge Highbyte = 0
080232f4  04 20        movs r0, #4           ; <-- die 4
080232f6  48 71        strb r0, [r1, #5]     ; Länge Lowbyte = 4
080232f8  08 46        mov  r0, r1
080232fa  c0 79        ldrb r0, [r0, #7]
080232fc  80 30        adds r0, #0x80        ; FC | 0x80
080232fe  c8 71        strb r0, [r1, #7]
08023300  26 48        ldr  r0, [pc, #0x98]
08023302  00 78        ldrb r0, [r0, #0]
08023304  08 72        strb r0, [r1, #8]     ; Exception-Code
08023306  09 22        movs r2, #9           ; <-- 9 Byte gesendet
0802330a  e4 f7 51 fd  bl   CH395_SPI_Send_Data
```

### Warum 3 und nicht 4

Der Frame:

| Offset | Inhalt |
|---|---|
| 0–1 | Transaction ID |
| 2–3 | Protocol ID |
| 4–5 | **Length** |
| 6 | Unit ID |
| 7 | FC \| 0x80 |
| 8 | Exception-Code |

Das Längenfeld zählt die Bytes **nach** sich selbst: Unit + FC + Code = **3**. Gesendet werden
9 Byte, deklariert werden 6 + 4 = **10**. Der Frame verspricht ein Byte mehr, als er liefert.

### Gegenprobe an der eigenen Konvention der Firmware

`FC03_Read_Handler` @ `0x0802386c` (korrekte Basis; alt `0x0801f06c`) rechnet im Erfolgsfall:

```c
iVar1  = uVar5 * 2;      /* Bytecount = Anzahl * 2 */
iVar10 = iVar1 + 3;      /* Länge = Bytecount + 3  (Unit + FC + Bytecount-Byte) */
```

Bei einem Register: Bytecount 2, Länge 5, Frame 11 Byte — korrekt und live bestätigt
(`12 34 00 00 00 05 01 03 02 00 96`). Dieselbe Firmware schreibt im Fehlerfall eine 4, wo
nach ihrer eigenen Rechnung eine 3 stehen müsste. Es ist also ein Off-by-one, kein anderes
Framing-Schema.

---

## 3. Wann der Zweig erreicht wird

`FC03_Read_Handler` hat zwei Fehlerausgänge, beide landen hier:

| Rückgabe | Exception | Auslöser |
|---|---|---|
| 2 | Illegal Data Address | Register liegt in einer **Lücke zwischen zwei Descriptor-Einträgen** |
| 3 | Illegal Data Value | Register **hinter dem letzten Eintrag**, oder Anzahl 0 bzw. > 125 |

Zusätzlich setzt der Dispatcher Code 1 (Illegal Function) für jeden FC außer 03/06/10.

Die Descriptor-Tabelle hat 246 Einträge (Stride 0xC) ab SRAM `0x20000354`. Die Suchschleife
bricht beim ersten Eintrag ab, dessen Ende über der gesuchten Adresse liegt; liegt die Adresse
davor, ist sie in einer Lücke → Code 2.

---

## 4. Auswirkung

- **Jedes nicht implementierte Register kostet einen vollen Client-Timeout** statt einer
  sofortigen Absage. Bei 3 s Timeout sind das 3 s pro Fehlgriff.
- **Gap-Bridging wird unbezahlbar — aber nur wegen dieses Fehlers.** Am Gerät lehnt ein Block,
  der ein nicht belegtes Register überspannt, sofort mit Exception 2 ab; das kostet dieselben
  ~126 ms wie jeder andere Request. Kann der Client die Exception nicht lesen, werden daraus
  3 s. Damit kippt die Rechnung: mit lesbarer Exception spart `max_gap=4` 19 % der Zykluszeit,
  ohne sie wäre dieselbe Einstellung rund 9 s langsamer als lückenloses Lesen. Messwerte in
  Abschnitt 5d.
- **Register-Map-Erkundung ist langsam.** Ein Scan über unbekannte Adressen läuft mit
  Timeout-Geschwindigkeit statt mit Antwortgeschwindigkeit. Für die noch offene Venus-A-Map
  ist das der Unterschied zwischen Minuten und Stunden.
- **Ein unbelegtes Register kostet zwei Reads, nicht einen.** Die 9 Byte bleiben im
  Empfangspuffer; der Parser zieht das fehlende zehnte Byte aus der Folgeantwort und verwirft
  beide Frames. Gemessen: 6,1 s und zwei verlorene Reads (Abschnitt 5b).

---

## 5. Messungen

### 5a. Direkt am Gerät — der Nachweis

`192.168.1.50:502`, Proxy gestoppt, Rohmitschnitt auf dem Socket:

| Anfrage | Antwort | Code |
|---|---|---|
| 30200 (belegt) | `15 F8 00 00 00 05 01 03 02 00 96` — 11 B, Länge 5 ✓ | — |
| 39000 | `18 58 00 00 00 04 01 83 03` — **9 B, Länge 4** | 3 |
| 45000 | `1F C8 00 00 00 04 01 83 02` — **9 B, Länge 4** | 2 |
| 31500 | `1B 0C 00 00 00 04 01 83 02` — **9 B, Länge 4** | 2 |
| 35000+12 | `18 B8 00 00 00 04 01 83 02` — **9 B, Länge 4** | 2 |

Das Gerät antwortet **sofort** und mit einem wohlgeformten Exception-Frame — bis auf das
Längenfeld. Die Codes decken sich exakt mit den beiden Rückgabepfaden aus Abschnitt 3:
39000 liegt hinter dem letzten Descriptor-Eintrag → Code 3, die übrigen in einer Lücke bzw.
≥ 40000 → Code 2. Damit ist die Statik aus Abschnitt 2 end-to-end bestätigt.

### 5b. Kosten beim Client — gemessen

Realer Client (tmodbus 0.6.2, `MarstekModbusClient`) auf **einer** Verbindung, ohne Reconnect:

```
1) gutes Register        -> 150    (0.01s)
2) unbelegt, 1 Versuch   -> None   (3.05s)
   tmodbus: Received unexpected response with Transaction ID: 2.
            Discarding bytes: 00 02 00 00 00 04 01 83 03 00
3) gut, gleiche Conn     -> None   (3.05s)   <-- verloren
4) gut, nochmal          -> 150    (0.05s)
```

Die verworfenen zehn Bytes sind die neun Exception-Bytes **plus das erste Byte der
Folgeantwort**. Der Parser wartet auf das deklarierte zehnte Byte, bekommt es aus dem nächsten
Frame, verwirft beide und resynchronisiert. **Ein unbelegtes Register kostet damit zwei
fehlgeschlagene Reads und 6,1 s.** Der Reconnect nach einem Timeout ist das, was den zweiten
Verlust verhindert — er ist keine Optimierungsreserve, sondern die Resynchronisation.

### 5c. Über den Proxy — warum es wie Stille aussieht

Dieselben Register über `192.168.1.60:1502`, jeder Versuch auf frischem Socket:

| Anfrage | Ergebnis |
|---|---|
| 30200+1, 30200+4, 34002+7 (belegt) | OK, 5–154 ms |
| 30200+6 (überspannt 30204) | OK, 4 ms |
| 30205+10, 34002+32, 35000+12, 42000+12 (Lücke) | **Stille**, 4 s Timeout |
| 39000, 45000, 60000, 31500 (unbelegt) | **Stille**, 6 s ohne ein einziges Byte |
| 31500, 35000+12 (Rohmitschnitt) | Gegenstelle **schließt** nach ~3,1 s |
| 49999 | Connection Reset |

**Über den Proxy kommt kein einziges Byte an.** Nicht neun, sondern null. Der Proxy ist selbst
ein Modbus-Parser: er wartet auf das deklarierte zehnte Byte, läuft in seinen eigenen Timeout
und schließt. Von hinter dem Proxy ist der Defekt deshalb nicht als Protokollfehler sichtbar,
sondern nur als schweigendes Gerät.

Nebenbefund: `192.168.1.50:502` weist Verbindungen aktiv ab (RST, nicht Timeout), solange
der Proxy läuft, während der Proxy selbst drei Clients gleichzeitig annimmt. Der Venus erlaubt
offenbar genau **eine** Modbus-TCP-Session — der Grund, warum der Proxy existiert, und der
Grund, warum dieser Befund so lange unentdeckt blieb.

---

### 5d. Gap-Bridging-Benchmark am Gerät

`Scripts/gap_benchmark.py` (lokal, nicht im Repo), Proxy gestoppt, komplette `d.yaml` (455 Definitionen,
482 Adressen) einmal je Strategie, 20 ms Pace, FC03 only. Der Client liest die defekten
Exception-Frames korrekt, indem er bei gesetztem Bit 7 fest zwei PDU-Bytes liest und das
Längenfeld ignoriert.

| max_gap | Blöcke | Requests | Wall | vs. Basis |
|---|---|---|---|---|
| 0 (heutige Implementierung) | 60 | 60 | 8,92 s | — |
| 1 | 51 | 52 | 7,91 s | −1,01 s |
| 2 | 48 | 49 | 7,35 s | −1,57 s |
| **4** | **41** | **48** | **7,25 s** | **−1,67 s (−19 %)** |
| 8 | 39 | 50 | 7,56 s | −1,36 s |
| 10 | 35 | 51 | 7,71 s | −1,21 s |

**310 Requests, null Timeouts, null Resets.** Jede Ablehnung kam als sofortige Exception 2.

Zwei Beobachtungen:

1. **Die Latenz hängt nicht an der Blockgröße.** Median 126 ms, egal ob 1 oder 40 Register,
   Maximum 180 ms. Die Kosten sind reiner Pro-Request-Overhead; zusätzlich gelesene Register
   sind praktisch gratis. Deshalb schlägt jede eingesparte Anfrage linear durch.
2. **Das Optimum liegt bei `max_gap=4`, nicht höher.** Ab 8 fressen die Ablehnungen und ihre
   Rückfall-Requests den Gewinn wieder auf.

Nicht überbrückbare Lücken (vom Gerät abgelehnt): 30001–30010, 30200–30214, 35000–35011,
35100–35110, 42000–42021. Mit einem Gedächtnis für schlechte Lücken (`bad_gaps`, wie im alten
`dev_tmodbus_blockread`) zahlt man jede nur einmal; im eingeschwungenen Zustand landen sowohl
`max_gap=4` als auch `=10` bei **45 statt 60 Requests** (−25 %).

Messartefakt: das Testset enthält 41000/41001 (`reset_device`, `factory_reset`) — reine
Kommandoregister, nicht lesbar, daher in jeder Strategie eine Exception. Die Integration pollt
sie nicht; im Live-Log kommen sie in keinem Block vor.

**Bedingung:** der Gewinn hängt vollständig daran, dass der Client die Exception lesen kann.
Mit 3 s Timeout je Ablehnung statt 126 ms wäre `max_gap=4` rund 9 s **langsamer** als
lückenloses Lesen. Gap-Bridging und die Frame-Korrektur sind ein Paket.

---

## 6. Vorgeschlagene Korrektur (Firmware)

```c
puVar1[5] = 3;   /* Unit + FC + Exception-Code */
```

Eine Konstante. Der Rest des Frames ist korrekt, auch die FC-Verknüpfung
(`0x03 + 0x80` ergibt in 8 Bit `0x83`).

---

## 7. Korrektur auf unserer Seite

Der Defekt lässt sich abfangen, ohne auf eine Firmware zu warten. Eine Exception-PDU ist per
Modbus-Definition **immer genau 2 Byte** (FC + Code) — das deklarierte Längenfeld wird für sie
schlicht nicht gebraucht. Wer bei gesetztem Bit 7 im Funktionscode die Länge ignoriert und
fest 2 Byte liest, liegt immer richtig, auch nach einem künftigen Firmware-Fix.

| Ort | Aufwand | Wirkung |
|---|---|---|
| **Im Proxy** | gering | Korrigiert den Frame einmal für *alle* Clients (HA, Omnibattery, Scans). Empfohlene Stelle. |
| **In tmodbus** | mittel, fragil | Das Framing liegt in `ModbusTcpProtocol.data_received` (`tmodbus/transport/async_tcp.py`), einem Bibliotheks-Internal. Monkeypatch überlebt kein Minor-Release; sauber wäre ein Upstream-Vorschlag, formuliert als allgemeine Robustheitsregel statt als Marstek-Sonderfall. |
| **Kürzerer Probe-Timeout** | gering | Umgeht das Problem statt es zu lösen. Gesunde Antworten kommen in 5–154 ms; 500 ms Timeout für Block-Probes macht einen Fehlgriff billig. Preis: ein echter Stall sieht aus wie ein totes Register — erst nach mehreren Fehlversuchen in Folge als tot führen. |

Gewinn bei gelöster Variante 1 oder 2: ein unbelegtes Register kostet statt 6,1 s und zwei
verlorener Reads etwa 10 ms und liefert eine **eindeutige** Aussage. Damit wird zweierlei
praktikabel, was heute ausscheidet: Gap-Bridging (rechnerisch 55 % weniger Requests) und das
schnelle Erkunden einer unbekannten Register-Map — relevant für die noch offene Venus-A-Karte.

## 8. Korrektur am eigenen Kenntnisstand

Zwei frühere Annahmen sind damit widerlegt:

1. **„Das Gerät antwortet auf nicht existierende Register mit Schweigen."** Falsch. Das Gerät
   antwortet sofort und korrekt mit Exception 2 bzw. 3. Das Schweigen entsteht erst dadurch,
   dass das Längenfeld um eins danebenliegt und jeder konforme Parser — Proxy wie Client —
   auf ein Byte wartet, das nie kommt.
2. **„Der Reconnect nach einem Timeout ist auf tmodbus womöglich überflüssig."** Das Gegenteil
   ist gemessen: er ist die einzige Resynchronisation, die den zweiten verlorenen Read
   verhindert. Nicht entfernen.

---

## 9. Querverweise

- `VNSD_FW_Bug_Read_Serializer_Vorzeichen_2026-08-22_de.md` — der andere Serializer-Defekt (32101)
- `VNSD_FW_Bug_Report_EN_Exception_Frame_Length.md` — herstellerseitiger Report zu diesem Befund
- `VNSD_Descriptor_Offene_Register_Ghidra_Befund_de.md` — Descriptor-Tabelle und Registerzuordnung
- Ghidra: durabler Pre-Kommentar auf `0x080232f4` (korrekte Basis; alte Notiz `0x0801eaf4`) in
  `/VNSD/Control/VNSD-0_app_0150_0805_115146.bin`

## 10. MCU-Korrektur

Die MCU ist ein **Nations N32G45x mit XFMC** (Cortex-M4F), keine Artery/AT32F403A — die frühere Zuordnung war
falsch (siehe `Control_FW`-Hauptdokumentation, Abschnitt „MCU identification"). Für den hier beschriebenen
Fehler ist das ohne Belang, da er reines Anwendungscode-Verhalten betrifft, keine MCU-spezifische Eigenschaft.

## 11. Status in v151 (Firmware 151, geprüft 2026-10-01)

**Unverändert.** `Modbus_Dispatcher` liegt in v151 bei `0x08023398` (392 Byte, gleiche Größe wie v150). Der
Exception-Zweig ist byte-identisch zum v150-Fehler:

```
08023488  0420             movs r0,#0x4      ; MBAP-Länge Lowbyte weiterhin 4 statt 3
0802349a  0922             movs r2,#0x9      ; weiterhin 9 Byte gesendet
```

Der Decompiler zeigt an derselben Stelle unverändert `g_modbusTcpTxFrame_lengthLo = '\x04';` gefolgt von
`CH395SendData()`. Der Off-by-one im MBAP-Längenfeld für Exception-Frames besteht in v151 unverändert fort —
mit denselben Folgen für Clients (Timeout statt sofortiger Fehlerauswertung).
