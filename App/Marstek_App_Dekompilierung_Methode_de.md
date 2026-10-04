# Marstek-App: Dekompilierungsmethode (Dart AOT)

> 🇬🇧 English version: [Marstek_App_Decompilation_Method.md](Marstek_App_Decompilation_Method.md)

Wie die Logik der Marstek-App (`com.hamedata.marstek`, Dart-Paket `cross_power_x`, Version 1.6.76) analysiert wurde. Nur statische Analyse: kein Gerät, keine Cloud, kein Login, nichts wird ausgelöst. Überblick über die App: [README_de.md](README_de.md).

## 1. Warum Strings allein in die Irre führen

Die App ist eine Flutter-App. Die gesamte Logik steckt im Dart-AOT-Snapshot `libapp.so`; es gibt keinen lesbaren Java-/Kotlin-Code. Eine einfache `strings`-Suche findet Texte, aber **die Reihenfolge des globalen String-Pools sagt nichts über die Code-Nachbarschaft**: benachbarte Strings gehören nicht zu benachbartem Code.

Beispiel: Die Suche nach „recovery“ fand Debug-Ausgaben `ACC======recovery current type/mac/version/temp=====` neben anderen Recovery-Strings. Sie gehören zur CT-/Zähler-Kopplungsseite (`select_ct_page.dart`) und haben nichts mit dem Akku zu tun. Weitere Fehltreffer: `recovery_server` (l10n „Reset Server Connection“), `recoveryMqttSSL` (Enum desselben Eintrags), `/ems/api/v1/setApplyVenusActivate` (Aktivierungscode der Leistungsklasse). Ein String zählt erst als Beleg, wenn seine **Verwendungsstelle** (die Funktion, die ihn referenziert) bekannt ist.

## 2. Die App beschaffen

- Der Hersteller bietet keinen EU-APK-Download an, und APK-Seiten von Drittanbietern dürfen nicht verwendet werden (unbekannte Veränderungen).
- Die installierten Splits vom eigenen Telefon ziehen:
  ```
  adb shell pm path com.hamedata.marstek
  adb pull <oben ausgegebener Pfad> .      # einmal je Split
  ```
- `libapp.so` liegt im ABI-Split `split_config.arm64_v8a.apk`, als `lib/arm64-v8a/libapp.so`. `libflutter.so` liegt im selben Split.
- Hier analysiert: Version 1.6.76 (Dart 3.8.1 stable, Flags `product … arm64 android compressed-pointers`); 1.6.72 wurde verglichen.

## 3. Blutter

[Blutter](https://github.com/worawit/blutter) ist ein quelloffener Dekompilierer/Analysator für Dart-AOT-Snapshots. Er liest `libapp.so` und schreibt kommentierten Assembler mit aufgelösten Klassen-, Methoden- und String-Namen.

Voraussetzungen: cmake, ninja, pkg-config, capstone, ein C++-Compiler, git, Python 3 (venv mit `pyelftools`, `requests`). Das passende Dart-SDK (3.8.1) wird beim ersten Lauf automatisch geholt.

```
python3 blutter.py <Ordner mit libapp.so und libflutter.so> <Ausgabeordner>
```

Bauen und Analyse dauern einige Minuten. Ausgaben:

- `asm/`: kommentierter Assembler je Dart-Bibliothek/Klasse (etwa 530 MB).
- `pp.txt`: Objekt-Pool (Strings, Konstanten, Referenzen).
- `objs.txt`: ausgegebene Objekte (Klassennamen, Enum-Werte).

Mit Dart-SDK-Checkout und Build-Ordnern sind es zusammen etwa 840 MB; alles kann danach gelöscht werden (`dartsdk/` und `build/` werden nur zum Bauen gebraucht).

## 4. Lesetipps

- **Smi-Konstanten** zeigen den rohen getaggten Immediate, der echte Wert ist die **Hälfte**: `230` bedeutet 0x73, `70` bedeutet 0x23, `128` bedeutet 0x40.
- Im komprimierten Zeigermodus von Dart 3.8 gilt `true` = NULL+0x20 und `false` = NULL+0x30. Deshalb springt `tbnz w0,#4,X`, wenn der Wert **false** ist, und `tbz w0,#4,X`, wenn er **true** ist.
- Die von Blutter ausgegebenen Adressen sind Offsets in `libapp.so`.
- Closures erscheinen als `AnonymousClosure` mit eigener Adresse; der Adresse folgen, nicht dem Namen.
- **Bewiesen** (im Disassembly verfolgt) von **abgeleitet** (nur aus Funktionsnamen) trennen.

## 5. Einer Funktion folgen

Typische Kette für eine Einstellungsfunktion:

1. Einstellungszeile (`createContent` des Settings-Page-State): Sichtbarkeitsbedingung (Gerätetyp), Beschriftungs-String, `onTap`-Closure.
2. `onTap`-Closure: Berechtigungs- und Verbindungsprüfung, Bestätigungsdialog.
3. Bestätigungs-Closure: Callback-Bündel (Start, Ergebnis, Fehler) und Helper-Aufruf.
4. Helper (z. B. `VenusHelper`): Abkürzung für Demo-Geräte, Wrapper mit Wiederholung und Timeout (`retryOnTimeoutWithNoTag`, 3 Versuche à 5 s).
5. `CommonCommand::<Funktion>`: `_doOperate(bleFn, mqttFn)`. Ist der aktuelle Gerätetyp leer, wird nichts gesendet; mit BLE-Verbindung läuft die BLE-Closure; sonst, wenn MQTT verfügbar ist, die MQTT-Closure.
6. BLE-Closure: baut die Byte-Liste `73 LEN 23 CMD payload`, hängt die XOR-Prüfsumme aller Bytes an (`calCrcValueListWithMutableList`), sendet über `BleManager::sentOrderToDevice`. MQTT-Closure: baut Topic `hame_energy/<type>/App/<id>/ctrl` und Payload `cd=<n>,...`, veröffentlicht über `MqttUtil::publishMessage`.
7. Antwort-Parser: BLE-Notify-Closure (Vergleich auf `frame[3]`) und `MqttAccoupler::analyseMqttData` (Vergleich auf `cd`), danach ein Event an die UI-Closure.

## 6. Zuordnung zur Firmware

Befehlsbyte (BLE) bzw. `cd`-Nummer (MQTT) und Payload aus Schritt 6 nehmen, dann den Handler der Control-Firmware nachschlagen:

- MQTT: `MQTT_CtrlCommand_Dispatcher`, Case = `cd`-Nummer (z. B. `cd=40` ist Case 0x28).
- BLE: `BLE_App_Command_Dispatcher`, Case = HM-Befehlsbyte (z. B. 0x40 ist `'@'`).

Prüfen, ob die Firmware-Version für diesen Code wirklich einen Handler hat; App- und Firmware-Nummerierung gelten pro Produkt, und eine neuere App kann Codes senden, die eine ältere Firmware ignoriert. Firmware-Seite von BLE: [../BLE/VNSD_BLE_Command_Map_v150_de.md](../BLE/VNSD_BLE_Command_Map_v150_de.md); vollständige Zuordnungstabelle App zu Firmware: [Marstek_App_vs_Firmware_Befehlszuordnung_de.md](Marstek_App_vs_Firmware_Befehlszuordnung_de.md).

## 7. Durchgespieltes Beispiel: „Akkupack-Wiederherstellung“

Frage: Was sendet der App-Button „Battery Pack Recovery“? Ergebnis: **der Befehl „BMS-Codierung“ (Adressnummerierung der Packs), keine Wiederherstellungsladung.**

| Schritt | Befund |
|---|---|
| Einstellungszeile | `AccouplerSettingState::createContent` baut die Zeile nur, wenn `isStack()` wahr ist (Venus D/A/Pro und Varianten, VEPAC). Label `tv_battery_recovery`, Icon/Button `icon_bms_coding` / `btn_bms_coding`. |
| Tippen | Closure prüft Admin-Recht und Verbindung, zeigt den Bestätigungsdialog „Preparing to recover the battery pack. Do you want to continue?“. |
| Bestätigen | Ruft `VenusHelper::handleBmsCoding` auf, das `CommonCommand::setBMSCodeFunction` in `retryOnTimeoutWithNoTag` (3 x 5 s) einpackt. |
| BLE-Closure | Liste `[0x73, 0x06, 0x23]`, Venus hängt `0x40` an, Payload `0x00`, XOR: **`73 06 23 40 00 16`** (Jupiter: `73 06 23 23 00 75`). |
| MQTT-Closure | Topic `hame_energy/<type>/App/<id>/ctrl`, Payload **`cd=40,number_as`** (Jupiter: `cd=20,number_as`). |
| Antwort | BLE: jede `0x40`-Antwort mit Länge 5 oder mehr gilt als Erfolg. MQTT: `cd=40,ret=1` ist Erfolg. Keine Antwort nach 3 x 5 s zeigt „Recovery failed“. |
| Folgebefehl | Keiner: keine Statusabfrage, kein Ladebefehl. |
| Firmware | MQTT: `MQTT_CtrlCommand_Dispatcher` Case 0x28 mit `strstr("number_as")`. BLE: `BLE_App_Command_Dispatcher` Case `'@'`. Beide enden in `BLE_Apply_ConfigReg_Offset3(0)` und CAN-Byte 3 an das BMS (Pack-Encode-Handshake scharf schalten). |

Die Begriffe „recovery/reactivate“ sind Marketing-Wortlaut; die internen Namen sind `handleBmsCoding`, `setBMSCodeFunction`, `check_get_bms_coding` und die Payload `number_as`. Details zur Firmware- und BMS-Seite: [../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md](../BMS/VNSD_Akkupack_Wiederherstellung_Firmware_de.md), App-Seite dort in Abschnitt 7.
