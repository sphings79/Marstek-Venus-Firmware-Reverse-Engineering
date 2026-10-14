# OTA-HTTP-Body-Grenze und OTA-Diagnose (Control-FW, Venus D / A / E3)

> 🇬🇧 English version: [OTA_HTTP_Body_Boundary_and_Diagnostics.md](OTA_HTTP_Body_Boundary_and_Diagnostics.md)

**Stand:** 2026-10-14. Geräteübergreifende Referenz. Alle Adressen gelten im jeweiligen Control-Image (Basis 0x08004800). Alles Folgende ist aus Code-Listings und Dekompilaten abgeleitet; nichts wurde auf Hardware getestet.

Ausgangspunkt war ein externes Issue (externer Melder), das E3-Control 148 und 150 vergleicht. Die Aussagen des Melders wurden nachgeprüft und auf D und A erweitert.

## 1. Zusammenfassung

- Ältere Control-Images bestimmen den Beginn des HTTP-Bodys im OTA-Download mit `strstr(response, "Connection:") + 21`. Das stimmt nur, wenn `Connection: close` der letzte Header ist. Neuere Images verwenden `strstr(response, "\r\n\r\n") + 4`.
- Der Wechselpunkt unterscheidet sich je Familie: **E3 149 -> 150**, **D 149 -> 149.2**, **A: in 150 bereits die neue Form** (ältere A-Versionen nicht analysiert).
- Der OTA-Reader (ota_mode = 1) wertet die Header nur beim ersten Socket-Read aus, sammelt einen geteilten Header nie auf und prüft das Selector-Ergebnis nicht auf NULL. Das ist in allen geprüften Versionen gleich, mit altem wie mit neuem Selector. Reale Header-Reihenfolge des Servers und Read-Grenzen sind unbekannt.
- BLE-Befehl 0x13 liefert eine persistente Fehlerhistorie aus 20 x 14 Byte. Das ist nicht das Komponenten-OTA-Ergebnis. Die Komponenten-OTA-Diagnose liegt in eigenen 8-Byte-Slots (Codes 0x400..0x404, Konsole `get_ota_log`).
- Die Footer-Modellprüfung (0x403) ist `strstr(Modell_Literal, Footer_Kopie)` (Argumente in dieser Reihenfolge). Echte Footer bestehen nur wegen dieser Reihenfolge.
- CAN-Kategorie 0x18 gibt es nur in der 151-Linie (D 151, E3 151, A 1509 und A 151). Alle geprüften Empfänger ignorieren sie. Zweck unbekannt.

## 2. HTTP-Body-Start-Selector (Funktion `HTTP_FindHeaderBodySeparator`)

Literal / Vorschub: **alt** = `"Connection:"` / +0x15 (21), Fehlerlog `[HTTP] get bin_start error .`; **neu** = `"\r\n\r\n"` / +4, Fehlerlog `[HTTP] Not found split mark \r\n\r\n, header incomplete`. Bei einem Fehlschlag liefern beide NULL; auch eine NULL-Eingabe liefert NULL. Einziger Aufrufer ist `HTTPS_POST_ReceiveResponseData`.

| Familie | Version | Selector-Adresse | Literal / Vorschub | Beleg |
|---|---|---|---|---|
| E3 | 147 | 0x08016EF0 | alt | rohe Thumb-Bytes |
| E3 | 148 | 0x08017570 | alt | rohe Thumb-Bytes |
| E3 | 149 | 0x08017A58 | alt | rohe Thumb-Bytes |
| E3 | 150 | 0x08018238 (42 B) | neu | Ghidra, rohes Listing, Literal-Pool |
| E3 | 151 | 0x080182E0 | neu | Ghidra, gleicher Code wie 150 |
| D | 147 | 0x08017C34 (44 B) | alt | Ghidra (live) |
| D | 149 | 0x08018744 (44 B) | alt | Ghidra (live) |
| D | 149.2 | 0x08018C58 (42 B) | neu | Ghidra (live) |
| D | 150 | 0x08018F2C (42 B) | neu | Ghidra (live) |
| D | 151 | 0x08018FD8 (42 B) | neu | Ghidra (live) |
| A | 150 | 0x08018DCC (42 B) | neu | Ghidra (Xrefs, Literal bei 0x08018DF8) |
| A | 1509 | 0x08018DEC | neu | Ghidra (live) |
| A | 151 | 0x08018E78 | neu | Ghidra (live) |

Wechselpunkt je Familie: E3 149 -> 150; D 149 -> 149.2; A in 150 bereits neu (kein älteres A-Image analysiert).
**Nicht analysiert:** A 148 / 1487 / 149 / 1508 (keine Programme mit korrekter Basis) und E3 144 / 1476.

Zugehöriger Reader (`HTTPS_POST_ReceiveResponseData`, 348 B bei D und A; 0x08019240 -> 0x08019310 bei E3 150 -> 151): E3 148 0x08018560, E3 150 0x08019240, E3 151 0x08019310, D 147 0x08018BFC, D 149 0x08019744, D 149.2 0x08019C74, D 150 0x08019F44, D 151 0x0801A018, A 150 0x08019DE4, A 1509 0x08019E08, A 151 0x08019EB8.

Zeigermodell des alten Selectors: Mit `Connection: close` als letztem Header stimmt das Ergebnis; bei einem anderen letzten Header (z. B. `keep-alive`) zeigt es 5 Byte zu früh; steht `Connection:` vor einem weiteren Header, zeigt es in den nächsten Header; ohne `Connection:`-Header liefert es NULL. Das sind Ergebnisse der Zeigerarithmetik, keine beobachteten Server-Antworten.

## 3. Verhalten des OTA-Readers beim ersten Read

Kette: `Task_ch_http` -> HTTP-Client-Dispatcher (OTA-Zweig) -> `HTTPS_POST_Request(6, 0x14, url_slot, len, 1)` -> `HTTPS_POST_ReceiveResponseData(..., ota_mode = 1)` -> {`HTTP_ParseContentLengthHeader`, Selector, `OTA_WriteDataToRingBuffer`}. ota_mode = 1 setzt nur der OTA-Zweig des Dispatchers; `HTTP_GetUrlData` und der Telemetrie-Upload nutzen ota_mode = 0.

OTA-Modus (ota_mode = 1):
- Jedes `CH395GetRecvData` schreibt an den **Anfang** des Empfangspuffers; es gibt keine Header-Akkumulation.
- Nur beim ersten Read (Flag wird sofort gelöscht): `HTTP_ParseContentLengthHeader(buf)`, `p = selector(buf)`, `n = read_len - (p - buf)`, `OTA_WriteDataToRingBuffer(p, n, 0)`, progress += n.
- **Keine NULL-Prüfung auf `p`.** Liefert der Selector NULL (kein Trenner im ersten Read): `n = read_len + buf` (ein Zeigerwert wird auf den Fortschrittszähler addiert), `OTA_WriteDataToRingBuffer(NULL, ...)` liefert -1 (E3 150; für D/A nicht erneut gelesen), und jeder weitere Read gilt als Body-Daten. Der Header wird nie erneut versucht; ein auf mehrere Reads verteilter Header wird nie zusammengesetzt.
- Schleifenende (D / A): 180-Tick-Timer, 500 Leerabfragen oder progress >= Content-Length. Rückgabe 1 bei progress != 0, sonst -1.
- Mit dem alten Selector wird derselbe Pfad genommen, wenn `Connection:` im ersten Read fehlt.

Nicht-OTA-Modus (ota_mode = 0): hängt bei `buf + offset` an, endet bei einem abschließenden `\r\n` nach mehr als 20 Leerabfragen, bei Timeout oder vollem Puffer; der Selector wird nicht benutzt.

Gleichheit über Versionen: Die Sequenz des ersten Reads ist in E3 148 und 150 bis auf BL-Offsets Byte für Byte gleich (Rohvergleich); E3 150 und 151 sind nach Adressmaskierung identisch; D 147 / 149 / 149.2 sowie D 150 / 151 und A 150 / 1509 / 151 sind nach Normalisierung von Adressen und Variablennamen identisch.

Beleg für die fehlende NULL-Prüfung: E3 150 und 151 aus dem rohen Thumb-Listing plus Dekompilat; E3 148 aus von Hand dekodierten Rohbytes; D und A nur aus der Dekompilat-Ausgabe (keine getrennte Lektüre auf Instruktionsebene).

**Unbekannt:** die reale Header-Reihenfolge des Servers und die realen Socket-Read-Grenzen (ob dem ersten Read der Trenner je fehlt), Chunked Transfer, und was ein fehlgeschlagener Body-Start für die nachgelagerte Verifikation bedeutet. Es wurde kein Hardware-Test durchgeführt.

## 4. BLE-0x13-Fehlerhistorie und Komponenten-OTA-Diagnose-Slots

### 4.1 BLE 0x13 (allgemeine persistente Fehlerhistorie)

`BLE_App_Command_Dispatcher` Fall 0x13 (Log `[BLE] Get err code info.`) -> `BLE_Send_Response(0x13, g_displayErrorLog, 0x118)`. 0x118 = 20 Einträge x 14 Byte:

| Offset | Größe | Feld |
|---|---|---|
| 0 | u16 | Jahr (Little Endian) |
| 2 / 3 / 4 / 5 | je u8 | Monat / Tag / Stunde / Minute |
| 6 | u32 | code1 |
| 10 | u32 | code2 |

Abgelegt im EEPROM bei 0x1100 + n x 14 (Ring aus 20 Einträgen). Der Loader prüft jeden Eintrag (Jahr 2000..0x898, Monat <= 12, Tag <= 31, Stunde <= 24, Minute <= 60), sonst wird der Eintrag zurückgesetzt. Der Writer `EventLog_Record_DisplayError(code1, code2)` ignoriert (0,0) und die Wiederholung des letzten Paares, unterdrückt gleiche Codes am selben Monat/Tag innerhalb von 31 Minuten und ersetzt sonst den ältesten Eintrag. Ein Byte innerhalb von code1/code2 hat für sich allein keine OTA-Stufen-Bedeutung.

### 4.2 Komponenten-OTA-Diagnose-Slots

`OTA_Set_SlotStatus(slot, status, detail)`: `slot < 4`; 16-Bit-Status bei `base + slot*8`, 32-Bit-Detail bei `base + slot*8 + 4` (Detail nur geschrieben, wenn ungleich null). Der Block wird zusätzlich über `Config_SaveWithCRC` persistiert (A: EEPROM 0x4000, 0x24 B, CRC16 über 0x22 B).

| Code | Bedeutung | Schreiber |
|---|---|---|
| 0x400 | Kommunikations-Watchdog-Timeout (Miss-Zähler > 0x3C bei Slot-Zustand 0) | `Comm_Watchdog_CheckTimeout` |
| 0x401 | Modbus-CRC16 passt nicht, Detail = CRC | `OTA_FW_Verify_SetStatus` |
| 0x402 | Footer-Gerätemaske passt nicht, Detail = Maske | `OTA_FW_Verify_SetStatus` |
| 0x403 | Footer-Modellstring passt nicht, Detail = erste 4 Footer-Bytes | `OTA_FW_Verify_SetStatus` |
| 0x404 | CAN-Update-Fehler (Fehlerflag gesetzt; Slots 2/3/0 nach Ergebniscode), 0 bei Erfolg | `CAN_UpdateResultHandler` |
| 0 | Slot gelöscht | `OTA_Firmware_Download_Init` |

Konsole: `ShellCmd_get_ota_log` gibt crc, `ota_flag`, `fail_flag` und `err_code%d` / `err_data%d` der vier Slots aus. In der per Xref geprüften A-Version gibt es keinen weiteren Schreiber von 0x400..0x404.

### 4.3 Adressen je Version

| Element | E3 150 | E3 151 | D 147 | D 149 | D 149.2 | D 150 | D 151 | A 150 | A 1509 | A 151 |
|---|---|---|---|---|---|---|---|---|---|---|
| BLE-Dispatcher (0x13) | 0x0800C0D4 | nicht erneut gelesen | 0x0800C194 | 0x0800C31C | 0x0800C758 | 0x0800C720 | 0x0800C730 | 0x0800C5B8 | 0x0800C5C8 | 0x0800C5C8 |
| Fehler-Loader | 0x080172A4 | 0x08017344 | 0x080170B8 | 0x080177A8 | 0x08017CBC | 0x08017F70 | 0x08018014 | 0x08017E10 | 0x08017E28 | 0x08017EB4 |
| Fehler-Writer | 0x0802B8CC | 0x0802BD14 | 0x0802B99C | 0x0802D33C | 0x0802D7B0 | 0x0802E308 | 0x0802E800 | 0x0802DCB8 | 0x0802E098 | 0x0802E1A8 |
| `OTA_Set_SlotStatus` | 0x0802BBD4 | 0x0802C01C | 0x0802BCA4 | 0x0802D644 | 0x0802DAB8 | 0x0802E610 | 0x0802EB08 | 0x0802DFC0 | 0x0802E3A0 | 0x0802E4B0 |
| Slot-Array (Code / Detail) | 0x20017E28 / 0x20017E2C | 0x2001A640 | 0x20017B78 / 0x20017B7C | 0x20017DC8 / 0x20017DCC | 0x20017E24 / 0x20017E28 | 0x20017E30 / 0x20017E34 | 0x2001A650 / 0x2001A654 | 0x20017E30 / 0x20017E34 | 0x2001A650 / 0x2001A654 | 0x2001A650 / 0x2001A654 |
| 0x401/0x402/0x403 (`OTA_FW_Verify_SetStatus`) | 0x080093B8 | 0x080093B8 | 0x080092F4 | 0x08009328 | 0x080096FC | 0x08009668 | 0x08009668 | 0x080095C4 | 0x080095C4 | 0x080095C4 |
| 0x404 (`CAN_UpdateResultHandler`) | 0x0802F898 | 0x0802FDD8 | 0x0802F7AC | 0x0803173C | 0x0803178C | 0x08032368 | 0x08032958 | 0x08031D18 | 0x080321E0 | 0x08032300 |
| 0x400 (`Comm_Watchdog_CheckTimeout`) | 0x080322A0 | nicht aufgeführt | 0x080322F8 | 0x0803444C | 0x080344A4 | 0x08035080 | 0x08035698 | 0x08034A2C | 0x08034F1C | 0x0803503C |
| Konsole `get_ota_log` | 0x08038554 | 0x08038B74 | 0x080384D4 | 0x0803A714 | 0x0803A75C | 0x0803B334 | 0x0803BA04 | 0x0803ACD8 | 0x0803B280 | 0x0803B3A0 |
| Fehlerlog-RAM (`g_displayErrorLog`) | 0x60013630 | - | 0x6001B670 | 0x6001E370 | 0x6001E490 | 0x6001E490 | 0x6001E490 | 0x6001E490 | 0x6001E490 | 0x6001E490 |

"nicht erneut gelesen" / "nicht aufgeführt" / "-": von den Prüfungen nicht abgedeckt. Das externe Issue nennt E3-148-Anker (BLE 0x0800bc18, Loader 0x08016600, Writer 0x08029d90, OTA-Writer 0x0802a098, Slots 0x20017ba0, Konsole 0x08036728); diese wurden in den Rohbytes von 148 **nicht** geprüft. Ihre 150-Entsprechungen stehen in der Spalte E3 150. Hinweis: Die Export-Kommentare von D 150 nennen noch die Slot-Adresse von 151; das Live-Symbol liegt bei 0x20017E30.

## 5. Verifikationskette 0x401 / 0x402 / 0x403

`OTA_FW_Verify_SetStatus` prüft das bereitgestellte Image in dieser Reihenfolge: 0x401, 0x402, 0x403. Der 32-Byte-Footer sind die letzten 0x20 Byte des empfangenen Images, gelesen mit `Flash_Read_Protected` in einen genullten Puffer.

1. **0x401 CRC16.** Modbus-CRC16 des Images gegen den erwarteten Wert; Abweichung -> Slot-Status 0x401, Detail = CRC.
2. **0x402 Maske.** Footer-Bytes 30..31 (u16, Little Endian) werden auf **exakte Gleichheit** mit dem u16 bei `ctx + 0x16` geprüft. Dieser Wert wird in `OTA_Firmware_Download_Init` aus einer statischen Tabelle mit 4 Einträgen gesetzt (Schrittweite 12 B: id u8 bei +0, Flash-Ziel u32 bei +4, Maske u16 bei +8), gewählt über `id == ctx[0]`:

   | id | Flash-Ziel | Maske |
   |---|---|---|
   | 1 | 0x200000 | 0x4444 |
   | 2 | 0x080000 | 0xFFFF |
   | 3 | 0x100000 | 0x2222 |
   | 4 | 0x180000 | 0x3333 |

   Die Maske gilt je Komponententyp, nicht je Modell (D, A, E3 teilen die Tabelle). Die Archiv-Footer passen dazu (Control FFFF, Micro 4444, BMS 3333); 0x2222 hat im Archiv keinen Footer. Es gibt keinen Platzhalter. Schlägt das Flash-Lesen fehl (Mutex-Timeout), bleibt der Puffer null, die Maske ist 0, und 0x402 löst vor 0x403 aus.
3. **0x403 Modell.** Der Aufruf ist `strstr(Heuhaufen = Modell-Literal, Nadel = 16-Byte-Footer-Kopie)`, d. h. `r0` = Literal, `r1` = Footer-Kopie. Folgen, aus dem Listing abgeleitet:
   - Die Nadel sind die ersten 16 Footer-Bytes, auf den Stack kopiert, **ohne geschriebenen Terminator**.
   - Bestanden = `strstr` liefert nicht-NULL; die Prüfung lautet also „Footer-String ist zusammenhängender Teilstring des Modell-Literals“. Ein leerer Footer-String (erstes Byte 0) besteht, ebenso jeder Teilstring (z. B. `VNS` bei allen Familien).
   - Ein Footer, der länger als das Literal ist (z. B. `VNSD-0X`), oder 16 Nicht-NUL-Bytes passen nicht.
   - Echte Footer sind `VNSD`, `VNSA` und `VNSEE3`; sie bestehen nur, weil das Literal das Suffix `-0` trägt und an der Heuhaufen-Position steht. Mit der üblicheren Reihenfolge `strstr(footer, literal)` würden diese Footer nicht passen, weil `VNSD` nicht `VNSD-0` enthält.
   - Modell-Literale (exakt, NUL-terminiert): D `VNSD-0`, A `VNSA-0`, E3 `VNSEE3-0` (nicht `VNSE3-0`). Das Literal ist der einzige Unterschied zwischen den D-, A- und E3-Funktionen; der Code ist in allen zehn geprüften Builds sonst identisch.
   - Der Footer von E3-BMS 110 ist `VNSE3`, kein Teilstring von `VNSEE3-0`; er würde diese Prüfung nicht bestehen, wenn er hindurchgeschickt würde. E3-BMS 112/113/115 tragen `VNSEE3`.

| Build | `OTA_FW_Verify_SetStatus` | `strstr`-Aufruf | Literal |
|---|---|---|---|
| D 147 | 0x080092F4 | 0x0800941E | `VNSD-0` |
| D 149 | 0x08009328 | 0x08009452 | `VNSD-0` |
| D 149.2 | 0x080096FC | 0x08009826 | `VNSD-0` |
| D 150 / 151 | 0x08009668 | 0x08009792 | `VNSD-0` |
| E3 148 (roh) | 0x0800905C..0x080091DC | 0x08009186 | `VNSEE3-0` |
| E3 150 / 151 | 0x080093B8 | 0x080094E2 | `VNSEE3-0` |
| A 150 / 1509 / 151 | 0x080095C4 | 0x080096EE | `VNSA-0` |

Das beschreibt die Logik, wie sie im Code steht. Sie wurde aus dem Listing abgeleitet (E3 148: von Hand dekodierte Rohbytes; alle anderen: Ghidra-Listing, A 150 zusätzlich dekompiliert) und **nicht auf Hardware getestet**. Ob die Reihenfolge Absicht ist (abgekürzte Footer akzeptieren), ist offen. Die Footer-Bytes 28..29 in Micro-/BMS-Images werden von dieser Funktion nicht benutzt; ob Micro/BMS den Footer selbst prüfen, wurde nicht untersucht.

## 6. CAN-Kategorie 0x18

- **Nur in der 151-Linie vorhanden:** D 151 `CAN_Write_Category0x18` @0x08032004, E3 151 @0x0802F484, A 151 @0x080319AC, A 1509 @0x0803188C. Nicht vorhanden in D 147 / 149 / 149.2 / 150, E3 150 und A 150 (Aufrufstellen-Aufzählung von `CAN_Build_Arbitration_ID`; A 150: auch kein Literal 0x14118). E3 147 / 148 / 149 wurden darauf nicht geprüft.
- **Frame:** `CAN_Build_Arbitration_ID(0x18, 1, 4, 0)` = 0x00014118, Extended Frame, DLC 8, Byte 0 = Wert, Bytes 1..7 = 0. Gesendet über `Register_WriteValue` -> `Modem_QueueSendMessage` -> CAN1-TX-Queue.
- **Auslöser:** Modbus-Schreiben auf 46000 mit 1234 / 2345 / 3456 / 4567 (setzt außerdem `g_otaTargetDevice` 1 / 4 / 3 / 2; Payload-Byte 0 = Low-Byte des Werts) und auf 47001 (0xB799; Byte 0 = Low-Byte des Werts); MQTT `cd=0x42` „Get INV DEBUG info“ mit `debug=3` (Payload 0x01) und `debug=4` (Payload 0x00, `g_maintModeState` = 0). In Versionen ohne den Sender setzt der 46000-Block nur `g_otaTargetDevice` und liefert Status 2.
- **Empfänger, alle ignorieren den Frame:**
  - E3-Micro 119 / D-Micro 116 (STM32G474, FDCAN1): Der Frame passiert den Hardware-Filter (ExtID 0x4100, Maske 0xFF00) und wird eingereiht; `EMS_Inverter_CAN_Dispatcher` hat keinen Fall 0x18, nichts wird gespeichert, keine Antwort.
  - A-Micro 1211 (FlexCAN): trifft die Mailboxen MB16-23, landet in einem leeren Eintrag der Klasse-4-Sprungtabelle (der Eintrag für 0x18 verzweigt zum gemeinsamen Ausgang). Der gemeinsame Ausgang löscht ein Link-Flag-Bit und frischt so den CAN-Link-Watchdog auf, wie bei jedem Frame, der den Filter passiert; der Inhalt wird nicht verarbeitet.
  - BMS D 118, E3 115, A 1105 / 115 / 110 (N32G45x, bxCAN1): Accept-all-Filter (aus den Registerschreibzugriffen im Code abgeleitet, nicht aus dem Datenblatt); kein Handler passt auf die ID.
  - Control: kein Handler für ein eingehendes 0x18.
- **Zweck unbekannt.** Die MQTT-Paarung (3 -> 0x01, 4 -> 0x00) deutet auf einen Ein/Aus-Schalter hin; das ist nur eine Schlussfolgerung.

## 7. Offene Punkte / nicht geprüft

1. A 148 / 1487 / 149 / 1508 und E3 144 / 1476: nicht analysiert (Selector, Reader, Diagnose).
2. E3 147 / 148 / 149: nur die Selector-Form, bei 148 zusätzlich Reader und Writer, aus Rohbytes gelesen; keine Dekompilation. E3-148-Diagnose-Anker aus dem externen Issue nicht geprüft.
3. Reale Server-Antwort-Header, TCP-Segmentierung und die Grenzen des ersten Reads; Chunked Transfer. Ob ein geteilter erster Read in der Praxis vorkommt.
4. Die fehlende NULL-Prüfung bei D und A beruht auf der Dekompilat-Ausgabe.
5. Rückgabewert von `OTA_WriteDataToRingBuffer(NULL, ...)`: nur für E3 150 gelesen.
6. Ob die Argumentreihenfolge der Modellprüfung Absicht ist; Hardware-Verhalten bei anderen als den ausgelieferten Footern.
7. Zweck der Kategorie 0x18; Verbraucher auf anderen Knoten (FC41D, andere Hardwarefamilien, spätere Micro/BMS); ob Control und BMS einen physischen Bus teilen.
8. Filter-Semantik von BMS und Micro (Accept-all, RXIMR „1 = vergleichen“) ist aus Code und Konvention abgeleitet, nicht aus Referenzhandbüchern.
9. E3-BMS 110 gegen die 0x403-Prüfung: nicht getestet.
10. Welcher Komponenten-Update-Schritt nach einem vollständigen Netzwerk-Download gelang oder fehlschlug, lässt sich aus dem Download allein nicht erkennen; der genaue Laufzeitfehler eines realen Update-Fehlschlags ist nicht geklärt.

## 8. Methode und Beweisgrenze

- Methode: Ghidra-Analyse der Control-Images (Basis 0x08004800), Thumb-Instruktionslistings mit Literal-Pools, exportierte Dekompilate und für E3 147 / 148 / 149 von Hand dekodierte Roh-Image-Bytes (ohne Disassembler). Versionsgleichheit über adressnormalisierte Diffs. Empfänger aus den Micro- und BMS-Images gelesen.
- Beleg-Markierungen: Selector-Adressen und Literale sind in allen genannten Ghidra-Programmen live geprüft; Reader, Diagnose und CAN-Auslöser für D 147 / 149 / 149.2 und große Teile von A stammen aus exportierten Dekompilaten, die mit jeder Live-Stichprobe übereinstimmten.
- Die synthetischen Zeiger-Replays im externen Issue bilden nur die Selector-Arithmetik nach. Sie sind mit dem Code vereinbar, aber kein Beleg für das Laufzeitverhalten; ein fehlender Marker im Modell ist keine sichere NULL-Behandlung durch den C-Aufrufer.
- Image-Identität (E3, aus dem Issue und erneut abgeglichen): `148_control_VNSE3-0_app_0148_0331_093751.bin` und das 150-Image stimmen mit den vom Melder genannten SHA-256-Werten überein. CRC-Übereinstimmung zeigt Konsistenz, nicht Echtheit.
- Kein Hardware-Test, kein Netzwerkmitschnitt, keine Beobachtung des Hersteller-Servers. Alle Aussagen beschreiben, was der Code bei den beschriebenen Eingaben tut, nicht was ein reales Gerät oder ein realer Server tut.
