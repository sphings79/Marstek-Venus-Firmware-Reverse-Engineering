# Cloud nicht erreichbar: der HTTP-Watchdog in Control v150 vs v151

> 🇬🇧 English version: [VNSD_Control_Cloud_Blocked_Watchdog_v150_v151.md](VNSD_Control_Cloud_Blocked_Watchdog_v150_v151.md)

**Datum:** 2026-10-01 · **Firmware:** Control v150 und v151 (Venus D) · **Funktion:** `HTTP_ClientTask_Dispatcher`
(v150 `0x0801A3D0`, v151 `0x0801A4A4`) · **Status:** aus dem dekompilierten Code, nicht am Gerät gemessen

**Anlass:** Ein Nutzerbericht (marstek_venus_modbus Issue 300, Venus E 3.0 mit V150) sagt, dass bei blockierter Cloud der
Netzwerkchip alle 30 Minuten zurückgesetzt wird und die Steuerung hängt. Diese Notiz erklärt den Code dahinter und wie
v151 ihn geändert hat.

## 1. Was der Watchdog beobachtet

- **Zeitbasis:** `Tick_Timer_Check_Elapsed` vergleicht einen 16-Bit-Zähler (`g_tick16b`). Er wird in
  `Runtime_Energy_Counter_Tick` (aus `SysTick_Handler`) bei jeweils 1000 Unter-Ticks zusammen mit `g_uptimeSeconds`
  erhöht. **Ein Tick = 1 Sekunde.**
- **Rückstau (`upload_cnt`):** das ist `g_telemetryHistoryRecordCount`, der Füllstand eines Ringpuffers mit 144 Plätzen
  (`Telemetry_History_Record_Push`, 0x35D Byte je Eintrag). Pro **5-Minuten-TOU-Slot** wird ein Eintrag abgelegt
  (`HTTP_Economy_TOU_PeriodicHandler`). Erfolgreiche Uploads entfernen Einträge (Logzeile „Delete data, upload_cnt = …").
  Bei toter Cloud wächst der Zähler also alle 5 Minuten um 1 (voll nach 12 h).
- **Testflag:** `g_oldStep` (Burn-in-Teststep, normal 0) sperrt die Aktion, solange ein Werks-/Alterungstest läuft.

## 2. v150

```
if (upload_cnt > 1 && Tick_Timer_Check_Elapsed(g_httpNoLinkRebootTick, 0x708 /*1800 s*/) && g_oldStep == 0) {
    CH395_Reset_And_Reinit(0);     // Soft-Reset des Ethernet-Chips, Log "[HTTP]ch395 reset!!!!"
    g_httpNoLinkRebootTick = 0;    // Timer neu gestartet
    g_httpClientJobState = 0;
}
```
Der Timer wird beim ersten Aufruf mit einem Rückstau von 2 oder mehr gestartet und beim Absinken des Rückstaus **nicht**
zurückgesetzt. Bei blockierter Cloud wird der Chip daher etwa **alle 30 Minuten** zurückgesetzt. Dabei werden jedes Mal alle
Modbus-TCP-Verbindungen getrennt (der Reset-Pfad enthält keine Wechselrichter-Aktion). Das passt zum Bericht.

## 3. v151

```
if (g_httpWatchdogSuppressFlag == 1)            g_httpNoLinkRebootTick = 0;   // unterdrücken
else if (upload_cnt < 5 || g_oldStep != 0)      g_httpNoLinkRebootTick = 0;   // Timer zurücksetzen
else if (Tick_Timer_Check_Elapsed(g_httpNoLinkRebootTick, 0x3840 /*14400 s*/)) {
    g_httpClientJobState = 0;
    System_Reboot();                             // kompletter MCU-Reset
}
```
- Die Bedingung muss **4 Stunden ohne Unterbrechung** gelten (jeder erfolgreiche Upload, der den Rückstau unter 5 bringt,
  startet den Timer neu). Bei toter Cloud wird das etwa 25 Minuten (Rückstau 5) + 4 h nach dem Cloud-Verlust erreicht.
- `g_httpWatchdogSuppressFlag` (`0x60025B1E`) wird im untersuchten Code nur **gelesen** (kein Schreiber gefunden), die
  Unterdrückung ist also praktisch nie aktiv.
- `System_Reboot()` ruft `Inverter_StopOutput()` auf, speichert die Laufzeitzähler und einen Neustart-Zeitstempel, wartet 50 ms
  und löst einen NVIC-Reset aus. **Der Wechselrichterausgang ist während des Neustarts aus** — auch im Notstrom-(EPS-)Betrieb.
- Der Watchdog prüft **nicht**, ob die Modbus-/LAN-Seite funktioniert oder ob die Batterie gerade Leistung liefert.

## 4. Folge

| | v150 | v151 |
|---|---|---|
| Cloud nicht erreichbar | CH395-Soft-Reset alle ~30 min | kein Chip-Reset, aber **kompletter Neustart nach ~4 h 25 min** (wiederholt sich) |
| Modbus TCP | Verbindungen alle 30 min getrennt | Verbindungen nur beim Neustart getrennt |
| Wechselrichter | keine Wechselrichter-Aktion im Reset-Pfad | Ausgang **aus** bei jedem Neustart |
| Kurzer Cloud-Ausfall (< 25 min) | nichts | nichts |

Für Aufbauten, die die Cloud absichtlich sperren (rein lokaler Betrieb, stromkritischer Notstrombetrieb), ersetzt v151 einen
häufigen, aber harmlosen Reset durch einen seltenen, aber störenden Neustart. Eine lokale Antwort auf die HTTP-Uploads (z. B. ein
Offline-Endpoint, der die Uploads quittiert) hält den Rückstau in beiden Versionen unter den Schwellen; ob der vorhandene
Offline-Endpoint das tut, wurde hier nicht geprüft.

## 5. Grenzen dieser Analyse

- Basiert auf dem dekompilierten Code von v150/v151; keine Messung am Gerät.
- Analysiert wurde der Pfad des **CH395-(Ethernet-)HTTP-Clients**. Ein reines WLAN-Gerät lädt über das Quectel-Modul hoch
  (`HTTP_Task_StateMachine_Handler`); dieser Pfad hat eigene Timeouts und wurde nicht auf einen Neustart-Auslöser untersucht.
- In beiden Versionen gibt es einen älteren, separaten Neustart: `Idle_Watchdog_AutoReboot` schaltet den Wechselrichter aus und
  startet nach einer Stunde vollständigem Leerlauf neu (kein PV, keine Netzfreigabe, Wechselrichterstatus < 2, alle Leistungen 0,
  keine Netzwerk-Kommandoaktivität).

## 6. Hält der Marstek-Offline-Endpoint den Rückstau auch unter v151 leer?

Geprüft durch Lesen des Server-Codes des Endpoints gegen den v151-Upload-Pfad (nicht gegen ein v151-Gerät gelaufen):

- **Upload-Pfad unverändert:** `Cloud_UploadTelemetryData` sendet per POST an `https://api-<region>.marstekcloud.com/data-upload/v1/venus/<id>`
  (CH395-Pfad). Ein Eintrag gilt als gesendet, wenn der HTTPS-POST abgeschlossen ist **und** `HTTP_ParseUploadResponseCode` 0 liefert.
  Dieser Parser ist in v150 und v151 identisch (`strstr("\"code\":")` + `atoi`; ein fehlendes `"code"` zählt ebenfalls als 0). Danach wird
  der Eintrag entfernt und `upload_cnt` sinkt um eins („Delete data, upload_cnt = …").
- **Der Endpoint antwortet genau das:** `POST /data-upload/v1/venus/*` wird mit `{"code":0,"message":"success","data":null}` beantwortet.
  Werden die Uploads quittiert, sinkt der Rückstau je Upload um einen Eintrag, bleibt weit unter 5, und die v151-Neustartbedingung
  (Rückstau ≥ 5 über 4 h) kann nicht eintreten.
- **DNS:** Das empfohlene Setup schreibt die ganzen Domains `marstekcloud.com` und `hamedata.com` inklusive Wildcards um. Das deckt auch die
  eine Host-Änderung in v151 ab: `getDateInfo` wanderte von `<region>.hamedata.com` zu `api-<region>.marstekcloud.com` (HTTP, Port 80).
  Die Zeit-Route des Endpoints reagiert auf `/getDateInfo`, unabhängig vom Hostnamen.
- **Nicht abgedeckt:** der WLAN-Uploadpfad (FC41D / Quectel). `Quectel_HTTP_POST_SendAndVerify` wurde in v151 umgeschrieben (Tick-Schleife mit
  ERROR-/`+QIURC`-Erkennung und exakter Byte-Zahl statt blinder Wartezeiten). Ob die Antworten des Endpoints die neue Schleife zufriedenstellen, wurde
  **nicht** geprüft. Die Dokumentation des Endpoints (30-Minuten-Reset, 15 Minuten bei WLAN) beschreibt nur v150; für v151 ist das Fehlerbild
  ohne Cloud der oben beschriebene Neustart.
