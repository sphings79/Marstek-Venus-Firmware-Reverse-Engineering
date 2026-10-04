# VNSD FC41D — Modul-Update-Wege und SoftAP-Aktivierung

> 🇬🇧 English version: [VNSD_FC41D_Update_and_SoftAP.md](VNSD_FC41D_Update_and_SoftAP.md)

**Stand:** 2026-09-30.

**Quellen:**
- FC41D-Modulfirmware 202512040647 (`FC41DAAR18A05M02_HM`), Dekompilate (5205 Funktionen):
  `<project>/Dekompilate/FC41D_202512040647_app.bin/` (lokal, nicht im Repo)
- Strings mit Referenzfunktionen (neu/alt) und String-Diff:
  `.../scratchpad/strings_FC41D_202512040647_app.json`, `strings_FC41D_202409090159_app.json`, `fc41d_strings_diff.json` (lokal, nicht im Repo)
- Control-Firmware v151, Dekompilate: `<project>/Dekompilate/VNSD_Control_v151/`
- Vorberichte: `VNSD_FC41D_Hamedata_Anpassungen_de.md`, `VNSD_FC41D_Comm_Modul_OTA_Analyse_de.md`,
  `VNSD_BLE_Comm_Modul_Update_Mitschnitt_Anleitung_de.md`, `security/VNSD_Firmware_Schluessel_und_Entpacken_de.md`,
  `security/WLAN_Credentials_ueber_Modbus_41500_de.md` (beide `security/`-Dateien lokal, nicht im Repo; alle im selben Projektordner)

Methodik: Offline-Analyse der Ghidra-Dekompilate beider Firmwarestände sowie gezielte Cross-Reference-Suche
(`grep` über die exportierten `.c`-Dateien) für Aufrufer/Aufgerufene. Jede Aussage ist mit Funktionsname/Adresse
belegt; unverifizierte Punkte stehen unter „Offene Punkte“.

---

## A. Wie das FC41D-Modul aktualisiert werden kann

### A.1 Update-Einstiegspunkte im Modul selbst

Im rekonstruierten AT-Kommandotable (siehe `VNSD_FC41D_Hamedata_Anpassungen_de.md`, §2.4) gibt es **zwei** Wege,
die tatsächlich Schreibzugriffe auf die OTA-Partition auslösen, plus einen reinen Query-Befehl:

| Befehl | Handler (Adresse) | Akzeptiert | Signaturprüfung |
|---|---|---|---|
| `AT+QWLANOTA=<url>[,<port>]` (intern „http_ota“) | `atcmd_QWLANOTA` @ `0x00067de0` → `FUN_0004bc04` @ `0x0004bc04` | HTTP **und** HTTPS (Präfixprüfung auf „https://“, Default-Port 80/443); URL-Länge ≤ 0x10000 wenn ein dritter Parameter übergeben wird | **Ja** – zwingend, s. u. |
| `AT+QFOTA=("ota/init"\|"ota/write"\|"ota/done")[,len]` | `atcmd_QFOTA` @ `0x00076ab8` | rohe Bytes, chunkweise **über die serielle Schnittstelle** (kein URL/Download); max. 0xc00 (3072) Byte je „ota/write“-Chunk | **Nein** – kein Aufruf einer Verify-Funktion in diesem Pfad |
| `AT+QGMR`, `AT+QVERSION` (Versionquery) | `atcmd_QGMR` @ `0x00067980`, `FUN_00075b60` @ `0x00075b60` | – (nur Lesen) | entfällt |

**`AT+QWLANOTA`-Ablauf (Signaturzwang, neu in 202512):**
`FUN_0004bc04` (@`0x0004bc04`) lädt die Datei per HTTP(S)-GET (`FUN_00025ad4` @ `0x00025ad4`) in einen
Flash-Staging-Bereich und ruft danach unmittelbar `FUN_0003eba4` (**`ql_secfota_verify_signature`**) auf.
Diese Funktion:
1. sucht per `FUN_0003e890` (**`ql_secfota_analys_file`** @ `0x0003e890`) im Staging-Bereich nach dem
   13-Byte-Magic **„QLSECFOTA!!!“** (Literal @ `0x0003e890`-Funktion selbst) und einer angehängten Signatur
   (≤ 255 Byte, sonst „signature len error“);
2. prüft die Signatur per `func_0x0003eb14` gegen einen **fest im Image einkompilierten NIST-P-384-Public-Key**
   (PEM-Literal in `FUN_0003eba4` @ `0x0003eba4`);
3. bei Erfolg: `FUN_0003ea90` (**`ql_secfota_move_file`**) kopiert den Payload in die eigentliche OTA-Partition;
4. bei Misserfolg: `FUN_0003e85c(0, 0x1000)` **löscht den Staging-Bereich wieder**, Log „FOTA Signature
   verification failed“.

Der auf diese Weise geladene Container muss also im **QLSECFOTA-Format** (Magic + ECDSA-P384/SHA-384-Signatur)
vorliegen. Das ist ein anderes Format als der von Marstek/Hamedata tatsächlich ausgelieferte **`.rbl`-Container**
(Magic „RBL\0“, AES-256-CBC-verschlüsselter Payload, siehe `security/VNSD_Firmware_Schluessel_und_Entpacken_de.md`,
§1) — eine `.rbl`-Datei enthält keine „QLSECFOTA!!!“-Signatur und würde diese Prüfung **nicht bestehen**
(siehe A.3).

`AT+QFOTA` dagegen schreibt die per „ota/write“ übergebenen Rohbytes direkt über `FUN_00039a7c` (Partitions-
Schreibfunktion, @`0x00039a7c`, aufgerufen aus `atcmd_QFOTA`) in die Partition; beim „ota/done“-Schritt wird nur
`FUN_00076980` (@`0x00076980`, reine Flash-Buchhaltung: Puffer schließen, Zeiger nullen) aufgerufen – **keine**
Signatur- oder Formatprüfung. Dieser Pfad kommt aber, wie unten gezeigt, in der aktuellen Control-Firmware v151
an keiner Stelle vor.

### A.2 Wie die Control-Firmware ein Modul-Update auslöst

In Control v151 gibt es **zwei** fest benannte Funktionen, die exakt das tun:

- **`Quectel_OTA_StartUpdate_AndReboot(uint32 use_local_ap)`** @ `0x08010074`: baut je nach Parameter
  `AT+QWLANOTA=http://www.hamedata.com/app/download/neng/HM_HIE_FC41D_remote_ota.rbl` (Parameter 0, Timeout
  180 s) oder `AT+QWLANOTA=http://192.168.137.1/FC41D_OTA.rbl` (Parameter 1, „lokaler AP“, Timeout 90 s), sendet
  das per `Quectel_AT_SendAndVerify(...,"OK",...)` und rebootet den Control-MCU danach hart
  (`System_Reboot()` @ Ende der Funktion) — unabhängig vom Ergebnis.
- **`Quectel_OTA_StartUpdate_FromURL_AndReboot(char *url, uint32 url_len, uint32 port)`** @ `0x08010170`: baut
  `AT+QWLANOTA=<url>` bzw. `AT+QWLANOTA=<url>,<port>` aus einer **frei übergebenen** URL/Port-Kombination,
  wartet bis zu 60 s auf das URC-Ergebnis (`g_otaFc41dResultState`) und rebootet danach ebenfalls hart.

Beide Funktionen sind **nur** über `AT+QWLANOTA` an das Modul angebunden. `AT+QFOTA` kommt in keiner der 1958
dekompilierten Control-v151-Funktionen vor (per `grep` über alle `.c`-Dateien geprüft) — der unauthentifizierte
Chunk-Upload-Pfad des Moduls wird von Control **nicht genutzt**.

**Auslöser (Aufrufer-Cross-Reference):**

| Kanal | Fundstelle | Auslösung |
|---|---|---|
| **BLE-App-Kommando `0x10`** | `BLE_App_Command_Dispatcher` @ `0x0800c730`, Case `'\x10'` | Erwartet Bestätigungsbytes `local_8[4..6] == 0x0a,0x0b,0x0c` und `g_networkLinkUp != false`; danach `local_8[7]==0` → `Quectel_OTA_StartUpdate_AndReboot(0)` (Cloud-URL), `local_8[7]==1` → `Quectel_OTA_StartUpdate_AndReboot(1)` (Lokal-AP-URL) |
| **BLE-App-Kommando `0x1b`** | `BLE_App_Command_Dispatcher` @ `0x0800c730`, Case `'\x1b'`, plus `BLE_Cmd_SetURL_Handler` @ `0x0800f25c` und `BLE_Pending_Commands_Process` @ `0x08015260` | Byte `local_8[4]` unterscheidet „URL ohne Port“ / „URL mit Port“ (Port aus zwei Bytes, `local_8[len-3..len-2]`); die URL wird in `g_otaUrlBuf` gepuffert, `g_otaUrlStartPendingFlag` gesetzt und im Haupt-Loop `BLE_Pending_Commands_Process` per `Quectel_OTA_StartUpdate_FromURL_AndReboot(g_otaUrlBuf, len, port)` verarbeitet – **freie URL, vom Handy vorgegeben** |
| **Cloud/MQTT-Kommando Typ `9`** | `MQTT_CtrlCommand_Dispatcher` @ `0x080244e0` | Feld `ot=0` → `Quectel_OTA_StartUpdate_AndReboot(0)`, `ot=1` → `Quectel_OTA_StartUpdate_AndReboot(1)` — dieselbe Cloud-URL bzw. Lokal-AP-URL wie beim BLE-Kommando `0x10` |
| Versionsabfrage | `Quectel_AT_QVERSION_Query` @ `0x08010370`: sendet `AT+QVERSION` (bis zu 3 Versuche) | liest die Modul-Firmwareversion (Antwortstring inkl. `_HM`-Suffix, s. Hamedata-Bericht §2.3) — kein `AT+QGMR` |

Ein direkter „Control reicht per BLE empfangene Rohbytes an das Modul weiter“-Mechanismus (Firmware-Bytes über
BLE-GATT direkt durchschleifen) findet sich in Control v151 **nicht**: Die einzigen BLE-Pfade zum Modul-Update
sind die beiden oben genannten Kommandos, die ausschließlich eine **URL** übergeben, die Control dann per
`AT+QWLANOTA` an das Modul weiterreicht. Das Modul lädt die Datei selbst per HTTP(S) — es fließen keine
Firmware-Bytes über die BLE-Verbindung zum Handy.

### A.3 Warum der bisherige BLE-Update-Weg mit der neuen Modul-Firmware (202512) vermutlich scheitert

Belegt:
- Die 202409-Firmware enthält **keinen einzigen** `ql_secfota_*`-String und keinen ECDSA-Public-Key
  (`fc41d_strings_diff.json`: gesamte `ql_secfota`-Funktionsfamilie ist „nur in neu“). D. h. in 202409 gab es in
  `AT+QWLANOTA` **keine** Signaturprüfung — ein heruntergeladener `.rbl` wurde offenbar ungeprüft in die
  Partition übernommen (Analysepfad des alten Images liegt hier nicht in vergleichbarer Dekompilat-Tiefe vor,
  s. „Offene Punkte“ Nr. 1).
- In 202512 ruft **derselbe** `AT+QWLANOTA`-Pfad jetzt zwingend `ql_secfota_verify_signature` auf (A.1). Der
  echte Marstek/Hamedata-`.rbl`-Container (Magic „RBL\0“, AES-256-CBC, siehe
  `security/VNSD_Firmware_Schluessel_und_Entpacken_de.md`) enthält **keine** „QLSECFOTA!!!“-Signatur — die Prüfung
  in `FUN_0003e890` würde das Magic im Staging-Bereich nicht finden, die Funktion mit einem Fehlercode
  zurückkehren, `FUN_0003eba4` den Staging-Bereich löschen und `atcmd_QWLANOTA` „http_ota download failed.“
  melden statt „OK“.
- Damit würden **beide** von Control angebotenen `AT+QWLANOTA`-Auslöser (Cloud-URL `www.hamedata.com/…rbl`,
  Lokal-AP-URL `192.168.137.1/FC41D_OTA.rbl`, sowie die freie BLE-Kommando-`0x1b`-URL) mit der 202512-
  Modulfirmware **an derselben Signaturprüfung scheitern**, sofern die dort abgelegte Datei ein normaler
  `.rbl`-Container ist und kein QLSECFOTA-signiertes Paket.
- Diese Erklärung ist eine **sehr wahrscheinliche, aber nicht am Gerät nachvollzogene** Ursache: Sie erklärt
  konkret, warum ein App-seitiger BLE-Update-Trigger, der intern genau diesen `AT+QWLANOTA`-Weg nutzt, mit dem
  neuen Modul plötzlich fehlschlägt, obwohl er mit 202409 funktionierte. Nicht geprüft wurde, ob die App
  stattdessen (zusätzlich oder alternativ) einen rohen BLE-GATT-Chunk-Transfer nutzt, wie ihn
  `VNSD_BLE_Comm_Modul_Update_Mitschnitt_Anleitung_de.md` als Möglichkeit beschreibt (Charakteristik `FF06`,
  Handle `0x0018`) — ein solcher Pfad kommt in Control v151 nicht vor (s. o.) und würde ohnehin nur über
  `AT+QFOTA` beim Modul ankommen können, wozu Control keinen Aufruf besitzt.
- Andere im Code sichtbare Verschärfungen, die zusätzlich relevant sein könnten: `AT+HMIOPEN` ist neu (nur
  202512, s. Hamedata-Bericht §2.1) und der komplette `AT+QBLE*`-Befehlssatz sowie die BLE-Security-Strings
  (`GAPC_TK_EXCH`/`GAPC_LTK_EXCH`) existieren laut String-Diff ebenfalls erst seit 202512 — ob sich dadurch das
  BLE-Pairing-Verhalten des Moduls selbst (nicht die Control-Firmware) geändert hat, lässt sich aus den
  vorliegenden Exporten nicht abschließend klären (s. „Offene Punkte“ Nr. 2).

### A.4 Praktische Optionen für den Besitzer

Aus dem Code ergeben sich zwei konkrete, im Gerät bereits vorhandene Mechanismen, um Control zu einem eigenen
`AT+QWLANOTA`-Aufruf zu bewegen — **ihr Erfolg hängt aber davon ab, ob die bereitgestellte Datei die
QLSECFOTA-Signaturprüfung besteht** (A.3), was für eine gewöhnliche `.rbl`-Datei nach aktuellem Kenntnisstand
**nicht** der Fall ist:

1. **BLE-Kommando `0x1b` (freie URL):** Frame `0x73 | LEN | 0x23 | 0x1b | ...` an Charakteristik `FF01`
   (Handle `0x0012`, siehe `VNSD_BLE_Comm_Modul_Update_Mitschnitt_Anleitung_de.md` §7) senden, mit eingebetteter
   URL (und optional 2 Portbytes, `local_8[4]` steuert „mit/ohne Port“, s. `BLE_App_Command_Dispatcher` Case
   `0x1b`). Ergebnis: Control sendet `AT+QWLANOTA=<eigene URL>[,<port>]`.
2. **BLE-Kommando `0x10` mit Parameter 1:** setzt fest `AT+QWLANOTA=http://192.168.137.1/FC41D_OTA.rbl` ab
   (Windows-Mobile-Hotspot-Standardgateway) — praktisch nutzbar, wenn das Modul selbst (als Station) mit einem
   Hotspot verbunden ist, der genau diese Adresse vergibt, und dort ein Webserver `FC41D_OTA.rbl` bereitstellt.
3. **Cloud/MQTT-Kommando Typ 9 (`ot=0`/`ot=1`):** identische Auslösung wie Kommando `0x10`, aber über die
   AWS-IoT-MQTT-Verbindung statt BLE (setzt Cloud-Zugriff voraus).

**Ergebnis für den Besitzer:** Ohne den privaten ECDSA-P384-Schlüssel, mit dem Hamedata ein QLSECFOTA-Paket
signieren würde, führt keiner dieser drei Wege zu einer erfolgreichen Übernahme einer normalen `.rbl`-Datei auf
der 202512-Modulfirmware. Der einzige im Code sichtbare Pfad **ohne** Signaturprüfung ist `AT+QFOTA`
(chunkweises Schreiben über UART) — dafür besitzt Control v151 aber **keinen** Aufrufer; ein Update über diesen
Weg müsste extern direkt per AT-Befehl über die UART-Verbindung Control↔Modul erfolgen (physischer/serieller
Zugriff auf diese Leitung nötig, z. B. durch Auflöten/Abgreifen), nicht über BLE, Modbus oder Cloud.

---

## B. SoftAP (eigener Access Point des Moduls)

### B.1 Modul-Seite: `AT+QSOFTAP`, `AT+QAPCFG`, `AT+QAPSTATIC`

| Befehl | Handler | Funktion |
|---|---|---|
| `AT+QSOFTAP=<ssid>[,<key>]` bzw. `AT+QSOFTAP=OPEN` | `atcmd_QSOFTAP` @ `0x00068464` → `FUN_00074ecc` @ `0x00074ecc` | Startet die SoftAP mit gegebener SSID (≤ 32 Byte) und Key (8–63 Byte, WPA2); ohne Key: **offenes Netz** und Log/URC `+QSOFTAP: WARNING_NO_PASSWORD` |
| `AT+QAPCFG` | `atcmd_QAPCFG` @ `0x00067a64` | Subparameter `linknum` (max. 1–2 gleichzeitige Clients) sowie ein zweiter, nicht abschließend geklärter Modus-Flag-Vergleich (Aufruf über Funktionszeiger `DAT_004006d8`, s. „Offene Punkte“ Nr. 3) |
| `AT+QAPSTATIC=<ip>,<mask>,<gw>[,<dns>]` | `atcmd_QAPSTATIC` @ `0x000683a0` → `FUN_00074fcc` @ `0x00074fcc` | Setzt IP/Maske/Gateway/DNS der SoftAP in RAM-Puffer `DAT_004013f5`/`…405`/`…415`/`…425`; diese Werte werden von `AT+QSOFTAP` beim Start ausgelesen (`FUN_00074ecc`) |

Es gibt **keine** im Handler `atcmd_QSOFTAP` fest einkompilierte Default-SSID/PW – beides muss der Aufrufer
übergeben. Für die IP-Konfiguration gilt dasselbe: `DAT_004013f5` u. a. sind reine RAM-Puffer ohne
Initialwert im Flash-Image; ob eine separate Boot-Routine sie mit einem Default (z. B. 192.168.0.1, s. u.)
vorbelegt, ist aus dem Export nicht zu klären (Offene Punkte Nr. 4).

**Gefundener, aber nicht sicher als aktiv erkannter Zweitpfad:** `FUN_0003a3c8` (@`0x0003a3c8`, geloggt als
„SOFTAP_COMMAND“) ist funktional identisch zu `atcmd_QSOFTAP`, verwendet aber **hart kodiert**
SSID/Key (übergeben) mit fester IP-Konfiguration `192.168.0.1` / Maske `255.255.255.0` / Gateway `192.168.0.1`
/ DNS `192.168.0.1` (`FUN_000140c8` @ `0x000140c8`, Literale direkt im Code). Diese Funktion hat im Export
**keinen ermittelbaren Aufrufer** aus der AT-Kommandotabelle oder aus Control – vermutlich Teil einer separaten
Debug-/Shell-Befehlsliste (Name „softap“, Stringtreffer @ `0x000dda2e`), die im vorliegenden Auszug nicht
vollständig rekonstruiert werden konnte.

**IP 192.168.19.1 / alte Weboberfläche (alt vs. neu):** Der String `192.168.19.1` (@`0x000faf79` in
202409090159, referenziert aus `FUN_00072498`) sowie `/index.shtml`, `/404.shtml` kommen in der **neuen**
Firmware 202512040647 **nicht mehr vor** (per String-Diff und gezielter Suche in beiden Stringlisten
bestätigt: null Treffer für „192.168.19“ und „shtml“ in der neuen Firmware). Die alte lwIP-httpd-
Weboberfläche für die Erstkonfiguration per Browser auf der SoftAP ist damit **bestätigt entfernt**, ebenso
das zugehörige AirKiss-Tencent-Smart-Config-Protokoll (dessen komplette Funktions- und String-Familie
`airkiss_*` nur in 202409 existiert). Auf der neuen Firmware ist über die SoftAP nach aktuellem Kenntnisstand
**kein** eigenständiger Web-/TCP-Konfigurationsdienst mehr aktiv; die einzigen noch vorhandenen Restspuren des
alten Airkiss-Mechanismus sind Log-Strings ohne zugehörige `airkiss_*`-Kernfunktionen (s. Hamedata-Bericht
§4).

**„12345678“ ist kein SoftAP/UI-Passwort:** Wie bereits in `VNSD_FC41D_Hamedata_Anpassungen_de.md` (§2.2) belegt,
sind SSID „hellow“ und Passwort „12345678“ **Default-WLAN-Station-Zugangsdaten** (nicht SoftAP!), die
`set_def_config` (`FUN_000766c4` @ `0x000766c4`) als Sentinel für „nicht konfiguriert“ behandelt (Log „not used
ssid and psk!“) und `save_user_info_tab_to_flash`/`_to_backup_flash` (`FUN_00076228`/`FUN_000767b8`) als
Werksvorgabe ins Flash schreiben. Es handelt sich um einen Werks-/Reset-Platzhalter für die
Stations-WLAN-Verbindung des Moduls, **nicht** um ein SoftAP-Passwort und **nicht** um ein UI-/App-Passwort.
Ein davon unabhängiger zweiter Default-Zugangsdatensatz existiert auf der Control-Seite
(„SM471…Hame147258…“, EEPROM `0x400`, Modbus-Register 41500 ff. — Station-WLAN-Default des Control-Boards,
siehe `security/WLAN_Credentials_ueber_Modbus_41500_de.md`); beide Wertepaare sind voneinander unabhängige
Werksdefaults unterschiedlicher Firmwarekomponenten.

### B.2 Control-Seite: Wird `AT+QSOFTAP` überhaupt gesendet?

**Nein.** Eine `grep`-Suche über sämtliche 1958 dekompilierten Control-v151-Funktionen nach den Strings
„QSOFTAP“, „QAPCFG“, „QAPSTATIC“, „softap“/„SoftAP“ ergibt **keinen Treffer**. Control v151 sendet ausschließlich
`AT+QSTAAPINFO`/`AT+QSTAAPINFODEF` (Station-Modus: das Modul verbindet sich **als Client** mit dem Heim-WLAN,
siehe `Quectel_WiFi_SetAPConfig` @ `0x08016a90` und `Quectel_WiFi_SetSTAInfo_NoSave` @ `0x08016bc0` — der Name
„AAPINFO“ bezeichnet hier die Ziel-AP-Information für die Station-Verbindung, **nicht** die eigene SoftAP des
Moduls). Die eigentliche SoftAP-Funktion des FC41D (`AT+QSOFTAP`) wird von der aktuell untersuchten
Control-Firmware **an keiner Stelle genutzt** — es gibt darin keinen Button-Trigger, kein Modbus-Register, kein
BLE-Kommando und keinen Cloud-Befehl, der sie aktiviert. Ein „Kein-WLAN-Fallback“ über die SoftAP (Modul öffnet
eigenen Hotspot, wenn Station-Verbindung fehlschlägt) ist im Code nicht vorhanden.

**Praktische Konsequenz:** Um die SoftAP des Moduls zu aktivieren, müsste `AT+QSOFTAP=<ssid>,<key>` (optional
vorher `AT+QAPSTATIC=<ip>,<mask>,<gw>`) **direkt per AT-Befehl über die UART-Leitung Control↔Modul** gesendet
werden — es gibt dafür keinen von Control, BLE-App, Modbus oder Cloud erreichbaren Auslöser. Ein lokaler
Konfigurations-Zugriff über eine vom Modul selbst gehostete Oberfläche (wie früher bei 192.168.19.1) ist mit der
aktuellen Kombination aus Control v151 und Modulfirmware 202512 nach diesem Befund **nicht vorgesehen** und auch
softwareseitig im Modul (fehlende `/index.shtml`-Weboberfläche) nicht mehr vorhanden.

---

## Offene Punkte

1. Die Analyse „`AT+QWLANOTA` prüfte 202409 keine Signatur“ stützt sich auf den vollständigen String-Diff
   (keine `ql_secfota_*`-Strings/Key in 202409) und die Codepfad-Analyse der Vergleichsversion, **nicht** auf
   eine ebenso tiefe Dekompilierung von 202409 selbst (dort liegen nur Strings vor, keine vergleichbare
   Funktionsdekompilation) — die Schlussfolgerung ist plausibel, aber nicht auf Codeebene der Altversion
   verifiziert.
2. Ob sich durch die 202512 neu hinzugekommene BLE-Security (`AT+QBLE*`-Kommandosatz, `GAPC_TK_EXCH`/
   `GAPC_LTK_EXCH`) das Pairing-Verhalten des Moduls gegenüber der App geändert hat (z. B. jetzt zwingendes
   Bonding, wo vorher offen gepaart wurde), lässt sich aus den vorliegenden Exporten nicht klären – der
   zugehörige Code (`app_sec.c`-Pfad) liegt außerhalb der hier untersuchten AT-Kommando-Handler.
3. `AT+QAPCFG`-Subparameter `0xb` (Vergleich gegen `&DAT_004006a4`, Aufruf über Funktionszeiger
   `DAT_004006d8`) konnte nicht auf seine genaue Bedeutung (vermutlich SSID-Sichtbarkeit „hidden“/„visible“)
   zurückgeführt werden – die referenzierten Datenadressen liegen außerhalb der als Strings exportierten
   Bereiche.
4. Ob die SoftAP-IP-Puffer (`DAT_004013f5` ff.) durch eine Boot-Init-Routine mit einem Default (vermutlich
   192.168.0.1, analog zu `FUN_000140c8`) vorbelegt werden, oder ob `AT+QSOFTAP` ohne vorherigen
   `AT+QAPSTATIC`-Aufruf mit 0.0.0.0 starten würde, ist aus dem Export nicht zu klären.
5. `FUN_0003a3c8` („SOFTAP_COMMAND“, feste IP 192.168.0.1) hat keinen im Export auffindbaren Aufrufer; die
   vermutete Zugehörigkeit zu einer separaten Debug-/Shell-Befehlsliste (getrennt von der AT-Kommandotabelle)
   konnte nicht verifiziert werden.
6. Ob die Marstek-App aktuell (noch) einen rohen BLE-GATT-Firmware-Chunk-Transfer nutzt (Charakteristik `FF06`,
   wie in `VNSD_BLE_Comm_Modul_Update_Mitschnitt_Anleitung_de.md` beschrieben) und dieser am Modul auf `AT+QFOTA`
   abgebildet würde, ließ sich mangels eines echten Update-Mitschnitts mit der 202512-Firmware nicht prüfen;
   in Control v151 existiert dafür jedenfalls kein Aufrufer.
