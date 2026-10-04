# VNSD FC41D Kommunikationsmodul – Hamedata/Marstek-Anpassungen

> 🇬🇧 English version: [VNSD_FC41D_Hamedata_Customizations.md](VNSD_FC41D_Hamedata_Customizations.md)

**Stand:** 2026-09-30.

**Firmware-Version:** 202512040647, Build-String `FC41DAAR18A05M02_HM.bin` (Quectel FC41D, Beken BK7231N, ARM968E-S)
**Quelle (Dekompilate):** `<project>/Dekompilate/FC41D_202512040647_app.bin/` (lokal, nicht im Repo)
**Rohdaten:** `<project>/HM_HIE_FC41D/fc41d_decrypted/fc41d_202512040647_app_decrypted.bin` (lokal, nicht im Repo)
**Vergleichsversion:** 202409090159, Build `FC41DAAR18A04M02_HM.bin` (String-Diff `fc41d_strings_diff.json`)

Methodik: Offline-Analyse der Dekompilate (Ghidra-Auto-Analyse, 4666 Funktionen), der extrahierten Strings mit Referenzfunktionen sowie gezielter Byte-Pattern-Suche im Rohimage (Little-Endian-Zeigersuche), um Tabellen/Records zu rekonstruieren, die Ghidra nicht als eigene Funktionen erkannt hat. Für jede Aussage ist die Quelladresse angegeben; Lücken sind unter „Offene Punkte" benannt.

---

## 1. Zusammenfassung

Die Firmware ist zu weit über 95 % Quectels Standard-AT-Firmware für das FC41D (Beken-SDK, lwIP, mbedTLS, FreeRTOS, BLE-5-Stack, Quectel-AT-Befehlssatz für WLAN/BLE/TCP/SSL/MQTT/HTTP/FOTA). Der komplette AT-Befehlsdispatch-Table (siehe §2.4) enthält **genau einen** Befehl, der nicht dem Quectel-Namensschema „AT+Q…" folgt: **`AT+HMIOPEN`**. Daneben finden sich eine Handvoll Hamedata-eigener Ergänzungen (eigene Flash-Konfigurationsschicht mit Entwickler-Debug-Tags „lyy debug"/„[cx]", ein „_HM"-Build-Suffix, Default-WLAN-Zugangsdaten, ein BLE-„connect_key"). Der Rest der Unterschiede zur Vorversion 202409 ist ein Versionssprung der zugrunde liegenden Quectel-SDK-Basis (neue Standardbefehle für BLE/MQTT/SSL/HTTP/Secure-FOTA), keine Hamedata-Eigenentwicklung.

---

## 2. Hamedata/Marstek-spezifischer Code

### 2.1 `AT+HMIOPEN` und die `hame_tcp`-Task

Fundstellen (Datenreferenzen im Rohimage, Adresse = Dateioffset + 0x10000):

- AT-Kommandotabelle, Eintrag `AT+HMIOPEN`: Tabellen-Slot bei **0x000f7460** (Name-Pointer → String „AT+HMIOPEN" @0x000f8098; Hilfetext-Pointer @0x000f7464 → „AT+HMIOPEN=1/0,ip,remote,local" @0x000f80a3; Handler-Pointer @0x000f7468 = **0x000419f9** (Thumb, realer Funktionsstart 0x000419f8)).
- Zugehörige String-Gruppe (alle unreferenziert von dekompiliertem Code, nur per Byte-Pattern im Rohimage lokalisiert, referenzierender Code liegt im Adressbereich 0x419f8, der in diesem Export **nicht** als eigene Funktion vorliegt – siehe „Offene Punkte"):
  - „hame_tcp_connect_server_command" @0x000ed22f
  - „hame_msg_handle ret: %d\r\n" @0x000ed2ab
  - „%s+HMIOPEN SUCCESS %s" @0x000ed272 / „%s+HMIOPEN FAIL" @0x000ed313 (URCs Richtung Control-Board/Host)
  - „hm_task_enable = %d ,enable = %d\r\n" @0x000ed333, „hm_task_enable = %d \r\n" @0x000ed356
  - Task-Name „hame_tcp" @0x000ed3cc

**Funktionsweise (Nachtrag 2026-09-30, nach Anlegen der fehlenden Funktionen in Ghidra verifiziert):**

- Handler `atcmd_HMIOPEN` (@0x000419f8): erwartet `AT+HMIOPEN=<enable>,<ip>,<remote_port>,<local_port>`. `enable` 0/1 wird in `hm_task_enable` (@0x003f1ec0) gespeichert; die IP wird per `inet_aton` in @0x003f1ec4 abgelegt, Remote-Port in @0x003f1ec8, lokaler Port in @0x003f1eca. Bei `enable=1` wird die FreeRTOS-Task **„hame_tcp“** (Entry `FUN_000416ec`, Stack 0x800, Priorität 7) gestartet.
- Task `FUN_000416ec` („hame_tcp“): legt einen TCP-Socket an, **bindet ihn an den lokalen Port** und **verbindet aktiv zur angegebenen IP/Remote-Port** (nicht blockierend mit `select`, 10 s Timeout). Bei Erfolg geht die URC `+HMIOPEN SUCCESS` an den Host (Control-Board), sonst `+HMIOPEN FAIL`. Danach empfängt die Task in einer Schleife bis zu 100 Byte je Paket (solange `hm_task_enable` = 1), gibt sie als Hex im Log aus und reicht sie an `hame_msg_handle` weiter.
- `hame_msg_handle` (`FUN_0004163c`): Rahmenformat **`AA <len> <cmd> <daten…> <prüfsumme>`**, Prüfsumme = 8-Bit-Summe der Bytes ab `<len>` (`FUN_000415ee`). Befehle:
  - `cmd 0x01` → `FUN_00041558`: **schaltet einen GPIO des Moduls** (Byte 3: Pin 0/1 → Beken-GPIO 6 bzw. 7; Byte 4: 0 = Eingang, 1 = Ausgang; Byte 5: Pegel, gesetzt über `bk_gpio_output`).
  - `cmd 0x02` → sucht Byte 3 in einer 9-Einträge-Tabelle (@0x00400504, Wertepaare) und ruft mit dem gefundenen Wert `FUN_000415d4` → `bk_uart_initialize(0)` auf, d. h. **UART-Neuinitialisierung mit anderem Parametersatz** (vermutlich Baudrate; Tabelleninhalt liegt in `.data` und ist ohne entpacktes RAM-Abbild nicht gelesen).
  - Antwort an die Gegenstelle: `AA 01 <ret> <ret+1>` (`ret` 0 = OK, 1–6 = Fehlercodes).
- **Aufrufer:** Keine der Control-Firmwares 147–151 sendet `AT+HMIOPEN` (Suche in allen fünf Dekompilaten und Listings: 0 Treffer). Der Befehl ist damit derzeit ungenutzt – vermutlich ein Werks-/Service-Werkzeug oder für eine künftige Control-Version.
- **Einordnung:** Eine von außen aufgebaute TCP-Verbindung, über die eine Gegenstelle zwei Modul-GPIOs schalten und die UART umkonfigurieren kann, ist ein typisches Fernwartungs-/Recovery-Muster (z. B. BOOT/RESET-Leitungen eines angeschlossenen Mikrocontrollers). Welche Leitungen GPIO 6/7 auf der Venus-D-Platine tatsächlich treiben, ist ohne Schaltplan/Hardware nicht feststellbar.

### 2.2 Eigene Flash-Konfigurationsschicht (Entwickler-Fingerabdruck)

Auf der Standard-Quectel-KV-Flash-API (`FUN_00039be4` Schreiben, Item-IDs 8/9) hat Hamedata eine eigene Config-Verwaltung aufgesetzt, erkennbar an durchgängigen Debug-Tags, die nicht dem Quectel-Stil entsprechen:

- „lyy debug" – 22 Fundstellen in 13 Funktionen, u. a. `FUN_00076228` (@0x00076228, `save_user_info_tab_to_flash`), `FUN_000767b8` (@0x000767b8, `save_user_info_tab_to_backup_flash`), `FUN_000766c4` (@0x000766c4, `set_def_config`).
- „[cx]" – 8 Funktionen, u. a. ebenfalls `FUN_000767b8`.

`FUN_00076228`/`FUN_000767b8` schreiben einen Config-Block mit Feldern SSID (Tag `0x11111104`), PSK (Tag `0x11111105`) und Gerätename „Quec_BT" (Tag `0x11111107`) nach Flash/Backup-Flash. Als **Default-Werte sind fest einkompiliert: SSID „hellow", PSK „12345678"** (@0x00076228 Z. 30/54/68 sowie @0x000767b8 Z. 32/56/70; PSK wird vor dem Schreiben über `FUN_0003ecc0` – vermutlich AES – verschlüsselt abgelegt). `FUN_000766c4` (`set_def_config`, @0x000766c4) liest diese Felder beim Boot zurück und behandelt SSID=„hellow"/PSK=„12345678" explizit als „nicht konfiguriert" (Log „not used ssid and psk!"), verbindet sich sonst mit den gespeicherten Zugangsdaten. Es handelt sich also um einen Werks-/Reset-Platzhalter, kein aktives Geheimnis – aber ein hartkodiertes Default-Credential-Paar.

### 2.3 Weitere Hamedata-Fundstellen (nicht abschließend aufklärbar)

Die folgenden Strings wurden zunächst nur über Datenreferenzen gefunden; die zugehörigen Handler sind inzwischen als Funktionen angelegt (siehe `atcmd_*` im aktuellen Export), aber noch nicht im Detail ausgewertet:

- „user ble passcode" @0x000e5358 – Hinweis auf einen zusätzlichen, nutzerseitig gesetzten BLE-Pairing-Passcode oberhalb der Standard-BLE-Security (siehe unten).
- „connect_key= %s , g_connect_key = %s\n" @0x000f7b86 – referenziert von Code @0x00068040/0x000682b0; deutet auf eine globale Variable `g_connect_key`, die mit einem übergebenen Wert verglichen wird (Zugriffs-/Pairing-Schlüssel für BLE oder den HMIOPEN-Tunnel). Wert und Vergleichslogik nicht rekonstruierbar (Handler nicht im Export).
- Feste NTP-Hosts „ntp1.aliyun.com" @0x000f7ab4 und „ntp.aliyun.com" @0x001073b8, sowie eine IP-Literal „210.72.145.44" @0x000f7ac4 unmittelbar benachbart zur ntp1.aliyun.com-Referenz (@0x00068040/0x00068044) – vermutlich Fallback-IP für den NTP-Host. Fest einkompilierte Zeitserver eines chinesischen Cloud-Anbieters (Alibaba) in einem Gerät, das auch in der EU vertrieben wird; ob sie tatsächlich angesprochen werden (oder nur Altcode sind) ist aus dem Export nicht zu klären.
- BLE-Security selbst ist **Standard**: Quellpfad-Kommentar „beken378/driver/ble_5_x_rw/ble_pub/app/src/app_sec.c" @0x000f649f sowie Strings „GAPC_TK_EXCH"/„GAPC_LTK_EXCH" @0x000f643d/0x000f64e5 zeigen den unveränderten Beken/RivieraWaves-BLE-5-Stack. Der „user ble passcode"/„connect_key"-Mechanismus liegt oberhalb (zusätzlich zu) dieser Standard-Pairing-Logik.
- Versions-/Build-Kennung: `AT+QVERSION`-Handler `FUN_00075b60` (@0x00075b60) liefert den Build-String „FC41DAAR18A05M02_HM.bin" (@0x000e63a2) zurück – Standard-Quectel-Mechanismus, nur der „_HM"-Suffix ist Hamedata-Branding.
- Standard-BLE-Werbename „Quec_BLE"/„Quec_BT" (@0x000e61dc, referenziert aus `FUN_0003b5c0` @0x0003b5c0, dem Standard-Quectel-„AT+QBLE…"-Testkommando-Dispatcher) ist **nicht** von Hamedata umbenannt – der für Nutzer sichtbare Gerätename wird laut Control-FW-Doku zur Laufzeit vom Control-Board per `AT+QBLENAME=MST_VNSD_xxxx` gesetzt.

### 2.4 Rekonstruierter AT-Befehlstabellen-Ausschnitt

Per Zeigersuche im Rohimage wurde der zusammenhängende AT-Kommando-Dispatchtable (Records zu je 3 Pointern: Name, Hilfetext, Handler) von 0x000f72bc bis mindestens 0x000f8788 rekonstruiert (≈95 Einträge: QGMR, CSUB, QREDDAVN, QRST, QSETBAND, QSETBAUD, QWLANOTA, QSLEEPMODE, QPING, QTESTFLASH, QURCCFG, QWLMAC, QSTADHCP(_DEF), QSTAAPINFO(_DEF), QSTASTOP, QSTASTATIC, QSTAST, QWSCAN, QSECCFG, QSOFTAP(STOP), QAPSTATIC, QAPSTATE, QAPSTAINFO, QGETIP, QGETWIFISTATE, QSETSTARETRY, QLOWPOWER, QDEEPSLEEP, QAPGETLIST, QTIME, QAPCFG, QGPIOCFG, **HMIOPEN**, QSTAINFORAW, alle QBLE*-Befehle, QIOPEN/QISEND/QIRD/QIACCEPT/QISTATE/QISWTMD/QICLOSE/QIGETERROR/QICFG, QMTCFG/QMTOPEN/QMTCONN/QMTSUB/QMTUNS/QMTPUB(RAW)/QMTDISC/QMTCLOSE/QMTRECV, QSSLCERT/QSSLCFG/QSSLOPEN/QSSLSTATE/QSSLSEND/QSSLRECV/QSSLCLOSE, QHTTPCFG/QHTTPGET/QHTTPPOST/QHTTPREAD, QNTP, QIDNSGIP, QFOTA, QWDG). **`AT+HMIOPEN` (Tabellenslot @0x000f7460) ist der einzige Eintrag, dessen Name nicht mit „AT+Q" beginnt.**

Abgleich mit dem öffentlichen Handbuch „Quectel_FC41D_AT_Commands_Manual_V1.0" (quectel.com, abgerufen 2026-09-30): `AT+HMIOPEN` kommt darin nicht vor. `AT+QWLANOTA` ist dort dokumentiert („Start OTA Upgrade", Syntax `AT+QWLANOTA=<URL>`). `AT+QREDDAVN`, `AT+QFOTA`, `AT+QSECCFG`, `AT+QDBGCFG` fehlen in diesem Handbuch-Stand ebenfalls, tauchen aber – anders als HMIOPEN – im Namensschema und Funktionscode klar als Quectel-eigene Erweiterungen auf; `AT+QREDDAVN` ist laut einer Quectel-Release-Note für andere Module (BG96) eine Abfrage einer sicherheitszertifizierten Versionsnummer („version number certified by R02A07_01") – vermutlich ein plattformweites Quectel-Feature einer neueren SDK-Generation, nicht Hamedata-spezifisch (Rückgabewert hier: fester String „SRCI0885TREDDAVN001" @0x000f77ab, Handler @0x00067a18 – nicht im Export enthalten).

---

## 3. Sicherheit: Secure-FOTA-Pfad (`ql_secfota_*`)

Namensschema (`ql_` = Quectel) und Code zeigen: Dies ist **Quectels eigener** Secure-FOTA-Mechanismus, keine Hamedata-Eigenentwicklung.

- `FUN_0003e890` (@0x0003e890, „ql_secfota_analys_file"): Parst einen Container mit 13-Byte-Magic „QLSECFOTA!!!" (@0x000ec765), extrahiert eine angehängte Signatur (Länge ≤ 255 Byte, sonst Fehler „signature len error"), kopiert die Nutzdaten und ermittelt Signaturlänge/Offset.
- `FUN_0003eba4` (@0x0003eba4, „ql_secfota_verify_signature"): Enthält **hartkodiert als String-Literal** einen NIST-P-384-(secp384r1-)Public-Key im PEM-Format:
  ```
  -----BEGIN PUBLIC KEY-----
  MHYwEAYHKoZIzj0CAQYFK4EEACIDYgAEn38D9dJtf9k2ReCLAccOacZ5GOKIS47r
  JTAIKmUszTn2B+OmfceEUSpubPNn9vGwA7v1nQKIJK2yBeNNYEEnLHMNHa7TEX3o
  peIHLV6UrKezJjasPNszcbnrhoz5+S3I
  -----END PUBLIC KEY-----
  ```
  (Adresse des Literals @0x0003eba4-Funktion, referenziert als Pointer in `func_0x0003eb14`, welcher die eigentliche ECDSA/SHA-384-Prüfung durchführt – Funktionsnamen „ql_secfota_verify_ecdsa384_signature"/„ql_secfota_calculate_file_sha384" @0x000ec9cb/0x000eca0c, beide nicht separat referenziert, also inline/statisch verwendet). Es gibt **keinen** separaten Key-Storage – der Vertrauensanker ist fest im App-Image einkompiliert.
- Bei Erfolg: `FUN_0003ea90` (@0x0003ea90, „ql_secfota_move_file") kopiert das verifizierte Image in einen Flash-Puffer und schreibt es. Bei Fehlschlag: `FUN_0003e85c(0, 0x1000)` löscht den Staging-Bereich, Meldung „FOTA Signature verification failed" (@0x000ec89e) – das Paket wird **nicht** übernommen.
- Aufrufer: `FUN_0004bc04` (@0x0004bc04) lädt eine Datei per HTTP/HTTPS-GET von einer als Parameter übergebenen URL (Default-Port 80 bzw. 443) und ruft danach unmittelbar `FUN_0003eba4` auf. Das ist der plausible Implementierungsort hinter `AT+QFOTA`/`AT+QWLANOTA` (beide Standard-Quectel-Befehle, gleiche AT-Tabelle wie HMIOPEN, s. §2.4).

**Zentraler Befund zum tatsächlich genutzten Update-Weg (korrigiert 2026-09-30, im Code verifiziert):** Das Control-Board aktualisiert das Modul ausschließlich per `AT+QWLANOTA=<url>` – auch mit den `.rbl`-Paketen von hamedata.com (Details: `VNSD_FC41D_Update_und_SoftAP_de.md`). In 202512 führt `atcmd_QWLANOTA` (@0x00067de0) den Download über `FUN_0004bc04` aus, und diese Funktion ruft **nach jedem erfolgreichen Download zwingend** `ql_secfota_verify_signature` (`FUN_0003eba4`) auf: Der Staging-Bereich muss mit „QLSECFOTA!!!“ beginnen und eine gültige ECDSA-P384/SHA-384-Signatur tragen, sonst wird der erste Sektor gelöscht und „verify fail“ gemeldet. Die bisher ausgelieferten Marstek-Pakete beginnen mit „RBL\0“ und tragen keine Signatur – **ein Modul mit 202512 nimmt sie nicht mehr an.** Das Format ist ein Aufsatz: Nach erfolgreicher Prüfung schreibt `ql_secfota_move_file` die ursprünglichen ersten Bytes (am Dateiende angehängt) zurück an den Anfang, sodass wieder ein normales `.rbl`-Paket im Staging liegt, das der Bootloader (mit dem AES-Standardschlüssel) wie bisher verarbeitet. Die alte Modul-Firmware 202409 lädt dagegen per einfachem HTTP (ohne TLS) und prüft keine Signatur.

Weitere hartkodierte Zugangsdaten: siehe §2.2 (WLAN-Default „hellow"/„12345678") und §2.3 (`g_connect_key`, unaufgeklärt).

---

## 4. Änderungen 202409 → 202512

Basis: Nur-in-neu/Nur-in-alt-Diff der String-Tabellen (`fc41d_strings_diff.json`, 141 nur alt / 212 nur neu).

**Neu in 202512 (Auswahl, eingeordnet):**

- *Quectel-SDK-Versionssprung (Standard, keine Hamedata-Arbeit):* kompletter BLE-5-AT-Befehlssatz (`AT+QBLE*`), MQTT-Client (`AT+QMT*`), SSL/TLS-Client (`AT+QSSL*`, mbedTLS-x509/ECDSA-Fehlermeldungen), HTTP-Client (`AT+QHTTPGET/POST/PUT`), `AT+QSECCFG`, `AT+QDBGCFG`, `AT+QREDDAVN` (s. §2.4), `AT+QSTAINFORAW`, `AT+QAPCFG`. Diese Funktionsgruppen existierten 2024 (202409) im Modul schlicht noch nicht.
- *Secure FOTA (Quectel-Standard, s. §3):* komplette `ql_secfota_*`-Familie inkl. eingebettetem ECDSA-P384-Public-Key – in 202409 gibt es dafür **keine** Entsprechung (keine der Strings „QLSECFOTA", „ql_secfota_*", kein PEM-Key im alten Image). D. h. **die 202409-Firmware hatte für keinen ihrer OTA-Pfade eine kryptographische Signaturprüfung.**
- *Hamedata-eigen:* `AT+HMIOPEN` (s. §2.1) – existiert ausschließlich in 202512, nicht in 202409.
- *Hamedata-eigen, neu:* Default-Zugangsdaten „12345678"/„hellow" (s. §2.2), `connect_key`/`g_connect_key` und „user ble passcode" (s. §2.3) – alle nur in 202512 vorhanden, in 202409 nicht.
- NTP-Fallback-IP „210.72.145.44" neben „ntp1.aliyun.com" (s. §2.3) – neu in 202512.
- „AT+QSOFTAP: WARNING_NO_PASSWORD" – neue Warnmeldung, wenn der Softap-Modus ohne Passwort betrieben wird (Standard-Hinweis, keine Funktionsänderung).

**Entfernt seit 202409 (nur in der alten Version vorhanden):** AirKiss (Tencent-WLAN-Smart-Config-Protokoll) sowie die lokalen lwIP-httpd-Weboberflächen-Seiten (`/index.shtml`, feste AP-IP `192.168.19.1`) für die Erstkonfiguration per Browser. Das alte „Modul öffnet eigenes AP + Weboberfläche"-Onboarding wurde ersatzlos gestrichen, vermutlich zugunsten der neuen BLE-Provisionierung (`AT+QBLEINIT=3`, dokumentiert im öffentlichen Handbuch: SSID/Passwort werden per BLE-GATT-Write mit eingebettetem `AT+QSTAAPINFO=...`-Befehl gesetzt). Auch das ist eine Quectel-SDK-Modernisierung, keine Hamedata-spezifische Entscheidung.

**Sicherheitsrelevante Einordnung des Diffs (korrigiert 2026-09-30):** Mit 202512 greift die Signaturprüfung genau auf dem von Marstek genutzten Weg (`AT+QWLANOTA`). Ein Modul mit 202409 akzeptiert jedes `.rbl` ohne Prüfung (auch Downgrades und selbst erstellte Pakete); ein Modul mit 202512 nur noch Pakete mit gültiger Quectel/Hamedata-Signatur. Der Schritt 202409 → 202512 ist damit praktisch eine Einbahnstraße. Der einzige Weg ohne Signaturprüfung bleibt `AT+QFOTA` (Übertragung über die UART), den das Control-Board nirgends verwendet.

---

## 5. Standard vs. custom – Kurzfassung

| Bereich | Einordnung |
|---|---|
| Basis-AT-Firmware (WLAN/BLE/TCP/SSL/MQTT/HTTP, Quectel-SDK auf Beken BK7231N) | Standard Quectel/Beken |
| Secure-FOTA-Mechanismus `ql_secfota_*` + ECDSA-P384-Key | Standard Quectel (Namensschema, keine Hamedata-Spuren) |
| `AT+QREDDAVN` | Vermutlich Quectel-Standard (plattformweites Feature, nicht in diesem Handbuch-Stand dokumentiert) |
| BLE-Pairing-Basis (`app_sec.c`, GAPC_TK/LTK_EXCH) | Standard Beken/RivieraWaves |
| `AT+HMIOPEN` + `hame_tcp`/`hame_msg_handle`/`hm_task_enable` | **Hamedata-eigen** |
| Eigene Flash-Konfig-Schicht („lyy debug", „[cx]", `save_user_info_tab_to_(backup_)flash`, `set_def_config`) | **Hamedata-eigen** |
| Default-WLAN-Zugangsdaten „hellow"/„12345678" | **Hamedata-eigen** (Werks-/Reset-Platzhalter) |
| `_HM`-Build-Suffix in `AT+QVERSION`-Antwort | **Hamedata-Branding** (kosmetisch) |
| `g_connect_key`/„user ble passcode", NTP-Hosts/IP | Vermutlich Hamedata-eigen, Details nicht rekonstruierbar (siehe unten) |

---

## 6. Offene Punkte

1. *(erledigt 2026-09-30)* Die fehlenden Funktionen (u. a. alle AT-Handler, `atcmd_HMIOPEN`, `hame_tcp`) sind in Ghidra angelegt; der aktuelle Export enthält 5205 Funktionen. Offen bleibt die Detailauswertung von `connect_key`/`g_connect_key` und des NTP-Codes.
2. *(teilweise erledigt)* `AT+HMIOPEN` wird von keiner Control-Version 147–151 gesendet; ob ein anderes Gerät/Werkzeug es nutzt, ist offen. Welche Signale GPIO 6/7 auf der Platine treiben, ist ohne Hardware nicht feststellbar.
3. *(korrigiert)* Die frühere Annahme „`.rbl`-Weg ohne Signaturschutz“ war für 202512 falsch (siehe §3). Der Bootloader selbst liegt nicht vor.
4. `AT+QREDDAVN`/„SRCI0885TREDDAVN001" konnte nur über eine Quectel-Release-Note zu einem anderen Modul (BG96) plausibilisiert werden, nicht über eine FC41D-spezifische Quelle – Einordnung „Quectel-Standard" ist wahrscheinlich, aber nicht letztgültig bestätigt.
5. Genaue Bedeutung der IP „210.72.145.44" (NTP-Fallback oder anderer Zweck) nicht verifiziert.
