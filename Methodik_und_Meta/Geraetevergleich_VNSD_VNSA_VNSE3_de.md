# Gerätevergleich Venus D / Venus A / Venus E3

> 🇬🇧 English version: [Device_Comparison_VNSD_VNSA_VNSE3.md](Device_Comparison_VNSD_VNSA_VNSE3.md)

**Stand:** 20. September 2026
**Zweck:** Festhalten, was zwischen den Modellen identisch ist, was abweicht und was ungeprüft
ist — damit ein Befund am D nicht stillschweigend auf A oder E3 übertragen wird und umgekehrt
nicht doppelt untersucht werden muss.

---

## 1. Codebasis — eine für alle Modelle

Ghidra Version Tracking, Quelle `VNSD-0_app_0150_0805_115146.bin` (D v150), Ziel
`150_control_VNSA-0_app_0150_0811_114724.bin` (A v150):

| | |
|---|---|
| Funktionen im D | 1884 |
| Funktionen im A (vor dem Lauf) | 955 |
| davon zugeordnet | **955 — alle** |
| im A ohne Gegenstück | **0** |
| im D ohne Gegenstück | 929 |
| Zuordnung über | *Exact Function Bytes Match* |
| Similarity | **durchgehend 1.0**, `sizeDelta 0` |

**Jede Funktion, die im A-Image disassembliert war, ist byte-identisch mit ihrem
D-Gegenstück.** Keine einzige zeigt eine Logikabweichung.

Die 620 Einträge, die das Diff als „changed" führt, sind ausschließlich Callee-Artefakte: die
aufgerufenen Funktionen waren im A-Image noch nicht erfasst, also erschien die Aufrufliste
kürzer. Betroffen sind durchweg Bibliotheksfunktionen — mbedTLS, FreeRTOS, llhttp, MQTT,
SysTick.

Die 929 im D ohne Gegenstück waren **keine fehlenden Funktionen**, sondern im A schlicht noch
nicht disassemblierter Code. Abschnitt 1d zeigt, wo sie liegen.

### Konsequenz

Ein Befund im **Control-Code** des D gilt bis zum Gegenbeweis auch für den A. Das betrifft
insbesondere die drei dokumentierten Firmware-Defekte — Exception-Frame, Read-Serializer,
Rücklesewert 44002/44003. Für den Exception-Frame ist es inzwischen belegt, siehe Abschnitt 4.

> **Wichtige Einschränkung, 21.09.2026.** Dieser Satz gilt **nur für den Control**. Für die
> Wechselrichter-Firmware („Micro", `*_inv_app_*`) ist er beim A nachweislich falsch — dort ist
> eine fremde Codebasis verbaut. Bei der E3 stimmt er wieder. Siehe Abschnitt 1e.
> Das ist mehr als eine Feinheit: die Fehlerwörter 36000/36100/36102 erzeugt **nicht** der
> Control, sondern der Wechselrichter. Die Integration hat deshalb eine Zeit lang die
> D-Fehlertexte am Venus A angezeigt, wo sie nichts bedeuten.

### 1b. Venus E3 gegen Venus D

Quelle D v150, Ziel `150_control_VNSE3-0_app_0150_0804_151249.bin`:

| | |
|---|---|
| Funktionen im D | 1884 |
| Funktionen im E3 | 1617 |
| zugeordnet | 1578 (447 identisch, 1131 „changed") |
| im E3 ohne Gegenstück | 39 |
| im D ohne Gegenstück | 306 |

Hier gibt es anders als beim A Nichtzuordnungen in beide Richtungen. **Sie sind aber kein
Beleg für fehlende Funktionalität** — siehe Abschnitt 5. In der Liste der 306 „nur im D"
stehen unter anderem `CAN_SendMessage`, `CH395_SPI_ReadByte` und der Config-Validator,
Funktionen die das E3 zweifelsfrei besitzt. Ghidra VT hat keinen Körper-Ähnlichkeits-
Korrelator; weicht eine Funktion geringfügig ab, fällt sie durch.

### 1c. Markup-Transfer aus dem D

Weil die Zuordnung byte-genau ist, wurde die Benennung aus dem D-Image übertragen:

| Ziel | übertragen | übersprungen | benannte Funktionen danach |
|---|---|---|---|
| A v150 | 945 | 135 | 954 von 955 |
| E3 v150 | 1556 | 303 | — |

Das benennt vorhandene Funktionen, **es erzeugt keine fehlenden**. Der Modbus-Pfad blieb im A
deshalb zunächst unsichtbar. Aufgelöst hat das erst der Lauf in Abschnitt 1d.

### 1d. Das A-Image ist derselbe Code, nur verschoben

Der entscheidende Befund. Von den 954 Funktionen, die nach dem Markup-Transfer in beiden
Images denselben Namen tragen, liegen nur **56 an derselben Adresse**. Alle übrigen sind um
einen von genau **21 verschiedenen Versätzen** verschoben:

| Versatz A − D | Funktionen |
|---|---|
| −1628 (−0x65C) | 603 |
| −1616 | 74 |
| ±0 | 56 |
| −1612 | 50 |
| −352 | 43 |
| −1624 | 31 |
| −1620 | 29 |
| −164 | 27 |
| übrige 13 Werte | zusammen ~40 |

Damit lässt sich jede D-Funktion im A gezielt suchen, statt blind nach Prologen zu scannen.
Verfahren:

1. Kandidatenadressen = D-Adresse plus jeder der 21 Versätze, **zusätzlich** jede Fundstelle
   der ersten 16 Bytes der D-Funktion im gesamten A-Image.
2. Bytevergleich, dabei **relokations-bewusst**: ein 4-Byte-Wort, das im A um einen der 21
   Versätze von dem im D abweicht, zählt als gleich (Literal-Pool-Zeiger). Ebenso ein
   Thumb-2-`bl`, dessen dekodiertes Sprungziel sich um einen dieser Versätze unterscheidet.
3. Annahme nur bei ≥ 85 % Übereinstimmung **und** ≥ 25 Prozentpunkten Abstand zum
   zweitbesten Kandidaten.

Ohne Schritt 2 liegen viele Funktionen bei 88–94 % und fallen durch; mit ihm springen sie auf
100 %. Von 926 gesuchten Funktionen wurden **774 zu exakt 100 %** wiedergefunden.

Ergebnis des Laufs:

| | |
|---|---|
| neu angelegt und benannt | 663 |
| umbenannt (trug vorher `FUN_…`) | 128 |
| Namenskonflikte (nicht angefasst) | 4 |
| fehlgeschlagen | 0 |
| mehrdeutig, übersprungen | 71 |
| nicht verortet | 60 |

**Stand des A-Images danach: 1831 Funktionen statt 955, Instruktionsabdeckung 73,4 % statt
35,5 %.** Der D liegt bei 1884 Funktionen.

Die 4 Konflikte sind Adressen, an denen eine bereits benannte A-Funktion steht und eine
*andere* D-Funktion byte-identisch passt — beides sind winzige Wrapper, die sich nicht
unterscheiden lassen:

| Adresse | Name im A | passt auch zu |
|---|---|---|
| 08006768 | `Delay_Adaptive` | `EEPROM_Mutex_Wait` |
| 080342da | `mbedTLS_AES_Setkey_Dec_Wrapper` | `mbedTLS_ECP_RNG_Wrapper` |
| 08046b74 | `mbedTLS_Cipher_Init` | `mbedTLS_SHA256_Init` |
| 080558fc | `CH395_Debug_SendCmdByte` | `SPI_WriteByte` |

Kein Fehler, sondern eine Grenze des Verfahrens: bei identischen Bytes entscheidet nur der
Aufrufkontext.

---

### 1e. Die Wechselrichter-Firmware zerfällt in zwei Familien

Nachtrag 21.09.2026. Der Vergleich oben betrifft ausschließlich den Control. Die
Wechselrichter-Images verhalten sich anders — Blockvergleich über alle 24-Byte-Fenster
(Schrittweite 4):

| | D-116 | E3-119 | A-119 |
|---|---|---|---|
| Datei | `vd_inv_app_0116` | `inv_app_0119` | `VA_inv_app_0119` |
| Größe | 115.712 B | 119.808 B | 146.432 B |
| Reset Handler | `0x080042AD` | **`0x080042AD`** | `0x08006435` |
| Initial SP | `0x20009A70` | `0x20009AC0` | `0x20002400` |
| Fehlerblock im SRAM | `0x200019F4` | `0x20001A28` | `0x20101650` |
| Basisadresse geladen über | Literal-Pool | Literal-Pool | `movw`/`movt` |
| gemeinsame Blöcke mit D-116 | — | **43,2 %** | **0,4 %** |

**D und E3 sind eine Familie, der A ist eine eigene.** 0,4 % ist Rauschen — von 73 gemeinsamen
Blöcken sind zwei triviale Füllmuster, der Rest zufällige kurze Befehlsfolgen. Zum Vergleich
teilen sich zwei Versionen derselben Familie rund 60 %. Auch die Codeerzeugung unterscheidet
sich systematisch: D und E3 holen Globals aus Literal-Pools, der A ausschließlich per
`movw`/`movt`-Paar.

Die Fehlerbits folgen dieser Trennung exakt. D und E3 haben dieselben err2-Bits mit denselben
Event-Codes (0, 1, 3, 4 über Event 0x36, Bit 7 über 0x13, Bit 8 über 0x31, Bit 11 über 0x35,
dazu Bit 16); der A hat die Bits 5, 6, 7, 8, 9, 11 und 13 mit anderen Events. Die
Hardware-Schutzlatch-Bits 0/1/3/4 des D existieren im A-Image überhaupt nicht.

Einziger bekannter Unterschied zwischen D und E3: die E3 setzt und löscht `warn` Bit 0, der D
beschreibt das Warnwort nie. Details in `VNS_Micro_Inverter/VNSD_Fehlercodes_Micro_FW_Analyse_de.md`,
Abschnitt 10, und für den A in `VNS_Micro_Inverter/VNSA_Fehlercodes_Micro_FW_Analyse_de.md`.

> Dass E3 und A beide eine Firmware „119" führen, ist Zufall: die Produktlinien zählen getrennt,
> die Prüfsummen unterscheiden sich, und der A steht laut Changelog längst bei 119.3 und 121.1.

### 1f. Die Modbus-Descriptor-Tabellen sind über die Modelle identisch

Ebenfalls 21.09.2026, entpackt mit `Scripts/marstek_descriptor_unpack.py` (lokal, nicht im Repo) aus den drei
Control-v150-Images:

| Vergleich | Einträge | Register nur in einem | Abweichungen bei Typ / Skala / Elementgröße / Anzahl |
|---|---|---|---|
| D ↔ E3 | 246 / 246 | keine | **0** |
| D ↔ A | 246 / 246 | keine | **0** |

Die einzigen Unterschiede sind SRAM-Quellzeiger: beim A **identisch** zum D, bei der E3 um −8
verschoben (212 Register), 29 unverändert, vier Einzelfälle.

**Folge:** Registernummern, Typen und Skalen müssen pro Modell nicht neu ermittelt werden. Was ein
Modell unterscheidet, ist ausschließlich, ob hinter einem Register Hardware steckt — keine PV
heißt, die MPPT-Register lesen 0; ein einzelner Speicher statt eines Stapels heißt, die Register
für Pack 2…7 lesen 0. Die vollständige D-Registerkarte ist damit die belastbare Vorlage für die
anderen Modelle, was für `e_v3.yaml` (bildet nur 82 von 410 bedienten Registern ab) unmittelbar
relevant ist.

## 2. Stand der Disassemblierung

| Image | Funktionen | Instruktionsabdeckung | Modbus-Code erfasst |
|---|---|---|---|
| D v150 | 1884 | — | ja |
| A v150 | **1831** (vorher 955) | **73,4 %** (vorher 35,5 %) | **ja** — siehe unten |
| A v1509 | 935 | ungeprüft | ungeprüft |
| E3 v150 | 1617 | — | ja, über den Markup-Transfer benannt |

### Der Modbus-Pfad im A

Über den Versatz aus Abschnitt 1d gefunden, disassembliert und benannt:

| Funktion | D | A | Versatz | Bytegleichheit |
|---|---|---|---|---|
| `Modbus_Dispatcher` | 0801ea04 | **0801e3ac** | −1624 | 94,1 % |
| `FC03_Read_Handler` | 0801f06c | **0801ea14** | −1624 | 98,4 % |
| `Read_Serializer` | 08050c14 | **080505b8** | −1628 | 98,6 % |
| `Write_Handler` | 08051d14 | **080516b8** | −1628 | 95,7 % |

Die fehlenden Prozente sind durchweg Literal-Pool-Adressen und Sprungziele, die sich mit dem
Layout mitverschieben.

### Korrektur zweier früherer Fehlschlussfolgerungen

**Erstens**, zu den Registernummern. In einer früheren Fassung stand, eine Suche nach `0xabe2`
(44002), `0xa424` (42020) und `0xa41b` (42011) liefere im A null Treffer, und daraus wurde
geschlossen, der Modbus-Pfad sei dort nicht erschlossen. Der Schluss trug nicht: **derselbe
Scan liefert im vollständig disassemblierten D-Image ebenfalls null Treffer.** Die
Registernummern stehen in keinem der Images als Immediate — sie werden berechnet
beziehungsweise über die Descriptor-Tabelle aufgelöst. Die Suche war für die Frage von
vornherein ungeeignet.

**Zweitens**, zu den Leistungsgrenzen, und das war der gravierendere Fehler. Es stand hier,
die Konstante 1500 komme in keinem der beiden Images vor, gesucht wurde als `0x5dd`.
**1500 ist `0x5DC`; `0x5DD` ist 1501.** Derselbe Versatz steckte in `0x9c5` für 2500 — richtig
ist `0x9C4`. Mit den korrekten Werten ergibt sich das genaue Gegenteil des damals
Behaupteten:

| Konstante | D | A |
|---|---|---|
| 1500 = `0x5DC` | 8× | **22×** |
| 2500 = `0x9C4` | **22×** | 8× |
| 2501 = `0x9C5` (Schranke für ≤2500) | 2× | 2× |
| 1200 = `0x4B0` | 29× | 29× |
| 2200 = `0x898` | 5× | 5× |

Die beiden Werte sind zwischen den Modellen schlicht **vertauscht**. Abschnitt 3a sagt, wo.

Die Lehre steht in Abschnitt 5: eine Negativsuche ist nur so gut wie die Umrechnung der
gesuchten Konstante.

---

## 3. Bekannte Unterschiede

### 3a. Maximale Lade- und Entladeleistung — aufgeklärt

| Modell | Grenze |
|---|---|
| Venus A | 1500 W |
| Venus D | 2500 W |

**Die Grenze steht im Code, an vier Stellen.** Die Leistungsklassen-Hypothese aus der
früheren Fassung wird dafür nicht gebraucht:

| Funktion | D | A |
|---|---|---|
| `FUN_08004b5c` — Config-Validator, hebt unplausible Werte auf Default | `0x9C4` = 2500 | **`0x5DC` = 1500** |
| `Config_Write_U16_0x202` — persistiert 44002 | Clamp 2500 | **Clamp 1500** |
| `Config_Write_U16` — persistiert 44003 | Clamp 2500 | **Clamp 1500** |
| `Write_Handler`, Zweig `uVar8 == 900` → **44002** | `param_4 < 0x9C5` (≤2500) | `param_4 < 0x9C5` (≤2500) |
| `Write_Handler`, Zweig `uVar8 == 0x385` → **44003** | `param_4 < 0x9C5` (≤2500) | **`param_4 < 0x5DD` (≤1500)** |

#### Die Asymmetrie

Die beiden Zwillingsregister werden auf dem A **unterschiedlich geprüft**, und das ist die
praktisch wichtigste Konsequenz:

- **44002 `max_charge_power`:** der `Write_Handler` nimmt weiterhin bis 2500 an und gibt 0
  zurück. Erst `Config_Write_U16_0x202` klemmt beim Speichern auf 1500. Ein Schreibzugriff
  mit 2000 wird also **angenommen und stillschweigend zu 1500**.
- **44003 `max_discharge_power`:** der `Write_Handler` lehnt alles über 1500 direkt ab und
  liefert `return 3`, also Exception 3 *illegal data value*. Der Clamp wird nie erreicht.

Am Gerät prüfbar, und die Vorhersagen sind gegenläufig genug, um aussagekräftig zu sein: 2000
nach 44002 schreiben → keine Exception, Wirkung 1500. Dasselbe nach 44003 → Exception 3.

**Folge für die Integration.** `a.yaml` sollte für 44002 und 44003 ein Maximum von 1500
führen, nicht 2500 — und beim Schreiben von 44002 ist ein Rücklesewert unter dem
geschriebenen Wert kein Fehler, sondern der Clamp. Das betrifft das offene Stable-Release
3.0.0 unmittelbar (Abschnitt 3c).

#### Wo die Leistungsklassen doch auftauchen

`BLE_Cmd_Dispatch` weist über den BLE-Pfad eine Klassennummer zu, und dort unterscheiden sich
die Modelle ebenfalls:

| | D | A |
|---|---|---|
| Zweig | `0x898` (2200) → Klasse **3** | `0x4B0` (1200) → Klasse **4** |
| Zweig | `0x9C4` (2500) → Klasse **0** | `0x5DC` (1500) → Klasse **5** |
| Bereichsprüfung | `299 < x < 0x9C5` | `299 < x < 0x5DD` |

Das ist das Klassen-Byte aus `Modbus_RS485_TCP/VNSD_Descriptor_Offene_Register_Ghidra_Befund_de.md`, Abschnitt 11. Es
existiert also, wird hier aber **geschrieben**, nicht gelesen — die wirksame Grenze kommt aus
den Konstanten oben, nicht aus dem Byte.

### 3b. PV und MPPT — im E3 vorhanden

Die Erwartung, dem E3 fehle der PV-Teil, trifft **nicht** zu. Das Image enthält den
vollständigen Satz:

```
PV1_Vol(0.1V)=%d|Cur(0.1A)=%d|Pow(0.1W)=%d   ... bis PV4
Mppt_State / Mppt_Error / Mppt_Temp / Mppt_Warning
mppt_power(W) / mppt_mode / mppt_ver(new) / mppt_ver(old)
PV_Day_Cap_10Wh / PV_Mon_Cap_10Wh / PV_Year_Cap_10Wh
update_mppt / get_mppt_data / "The MPPT will be updated soon"
&mod2=mppt&err2=%x   (Cloud-Report)
```

Nicht nur als Strings: der Markup-Transfer hat im E3 unter anderem `MPPT_Debug_Print`,
`AT_Copy_MPPT_Channel_Data`, `MQTT_Publish_VNS_MPPT_Data` und `Cloud_Report_FillPvData`
benannt — es sind echte Funktionen. Sogar der OTA-Pfad für die MPPT-Firmware ist enthalten.

Das passt zum Gesamtbild: eine Codebasis für alle Modelle, Unterschiede in Hardware und
Konfiguration. Dass ein Gerät keine PV-Eingänge besitzt, heißt nicht, dass seine Firmware sie
nicht kennt.

### 3c. Registerkarten der Integration

`a.yaml`, `d.yaml`, `e_v3.yaml` und `e_v12.yaml` unterscheiden sich. Der Venus-A-Teil gilt als
unbestätigt und blockiert das Stable-Release 3.0.0 der Integration.

Aus diesem Vergleich ergeben sich dafür drei belastbare Punkte, alle statisch aus dem Code
und keiner am Gerät geprüft:

1. **44002 und 44003 gehören in `a.yaml` auf max 1500**, nicht 2500 (Abschnitt 3a).
2. **44002 nimmt Werte bis 2500 an und klemmt still auf 1500.** Ein Rücklesewert unter dem
   geschriebenen Wert ist hier erwartetes Verhalten, kein Fehler.
3. **44003 lehnt über 1500 mit Exception 3 ab.** Eine Integration, die 2500 anbietet, erzeugt
   dort eine Fehlermeldung statt eines Clamps.

Ob die Descriptor-Tabelle im A weitere abweichende Register führt, ist weiterhin offen — jetzt
aber am Code beantwortbar, weil die Tabelle an der um den passenden Versatz verschobenen
Adresse liegt.

### 3d. RS485-Anbindung — anderer UART, keine Richtungssteuerung

Der zweite handfeste Unterschied, gefunden über die Restliste aus Abschnitt 1d.

| | Venus D | Venus A |
|---|---|---|
| UART | `0x40004400` = **USART2** | `0x40013800` = **USART1** |
| Richtungsleitung | GPIO `0x40011000` Bit 7 (**PC7**), gesetzt über BSRR `+0x10`, gelöscht über BRR `+0x14` | **keine** |
| Guard-Zeit vor dem Senden | `0xDAC` = 3500 Schleifendurchläufe | **keine** |
| Guard-Zeit vor dem Empfangen | `0xBB8` = 3000 Schleifendurchläufe | **keine** |
| Hilfsfunktionen | `RS485_Set_TX_Mode` @ 0x080072C8, `RS485_Set_RX_Mode` @ 0x080072E4 | **existieren nicht** |

Die A-Funktion bei `0x08007148` (hier `RS485_UART_Send_NoDirectionControl` benannt) ist
Instruktion für Instruktion das D-`RS485_UART_Send` @ `0x08007268`, nur ohne die beiden
Aufrufe am Anfang und am Ende. Dasselbe gilt für `0x080071A0` gegenüber
`RS485_UART_SendByte` @ `0x08007300`.

Dass die Funktionen wirklich fehlen und nicht nur woanders liegen, ist geprüft: die
20-Byte-Rümpfe beider Funktionen kommen im gesamten A-Image kein einziges Mal vor.

Die Aufruferliste ist auf beiden Modellen identisch — `RS485_FC03_ReadWrite_Handler`,
`RS485_FC06_WriteSingle_Handler`, `RS485_FC10_WriteMultiple_Handler`,
`RS485_RTU_Frame_Dispatcher`, `RS485_SendIfValid`, `Protocol_AA_RS485Forward`. Der A hat also
den vollständigen Modbus-RTU-Stack, treibt aber keine DE/RE-Leitung. Naheliegende Erklärung:
ein Transceiver mit automatischer Richtungsumschaltung oder eine andere physikalische
Schnittstelle. Nicht am Gerät geprüft.

### 3e. Weitere echte Codeunterschiede

`Grid_Export_Power_Limiter` (D `0x0801F200`, A `0x0801EBA8`), drei Abweichungen:

| | D | A |
|---|---|---|
| Obergrenze | `4000` | `0x960` = **2400** |
| Zustandsprüfung | `if (*x == 5)` | `if ((*x == 5) \|\| (*x == 1))` |
| Wechselrichter-Kommando | `Inverter_RS485_Command_Send(1,**1**,…)` | `Inverter_RS485_Command_Send(1,**3**,…)` |

Nicht weiter untersucht. Unverändert sind dagegen `Modbus_Dispatcher`, `FC03_Read_Handler`
und `Read_Serializer` — dort gibt es außer verschobenen Adressen keine einzige Abweichung.

Im D 150 heißt `Inverter_RS485_Command_Send` seit 2026-09-24 `Inverter_CAN_PowerCmd_Send`,
weil die Funktion per CAN sendet (s. `Control_FW/VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md`
§3.2). In den A-Programmen ist der alte Name geblieben.

### 3f. Ladelimit, Pack-Rotation und PV-Abregelung (Stand 2026-09-24)

Ausführlich für den D: `BMS/VNSD_BMS_Pack_Rotation_Taper_v118_de.md` und
`Control_FW/VNSD_Control_Ladeleistung_PV_Abregelung_v150_de.md`. Die Angaben hier stammen aus der
Analyse vom 24.09. und sind nicht so tief geprüft wie die D-Seite.

**Umrechnung Limit → Leistung** (`P_bms = Spannung × Ladestromlimit`) ist auf allen drei
Modellen gleich gebaut:

| Modell | Funktion | Aggregat-Struct |
|---|---|---|
| D 150 | `Power_Limit_Clamp` `0x08013234` | `0x20014F8E` |
| A 150 | `0x080130d4` | — |
| A 1509 | `0x080130ec` | `0x200177AE` |
| E3 150 | `0x080126cc` | `0x20014F86` |

**PV-Obergrenze bei Einspeisung** (`Grid_Export_Power_Limiter`):

| Version | Obergrenze |
|---|---|
| A 150 | `min(P_bms, 2500)`, wie D 149.2/150; Gesamtdeckel 2400 statt 4000 |
| A 1509 | `ac + P_bms`, wie D 147 (Byte-Vergleich bei `0x0801ecb4`) |

**BMS — Pack-Wechselschwelle beim Entladen (korrigiert 2026-10-11):** Die frühere Aussage, die
Entladeschwelle mit DoD-Untergrenze und die Modi 2/3 unter 11 % gebe es „nur im D 118“, war falsch.
Je BMS-Version (nur Mehrpack-Modelle; Image-Namen im Firmware-Archiv):

| Modell | BMS-FW | `PackSelect_DischargeStepThreshold` | Wechsel auf das vollste Pack bei (Cutoff 12 %) | `ChargeStepTarget`, Modi 1/2/3 | Beleg |
|---|---|---|---|---|---|
| D | 116 (kein Image im Archiv) | unbekannt | Packs stoppten bei **10,9 %** (beobachtet); passt zur 117.7-Formel | unbekannt | von Dritten an einem 4-Pack-Gerät gemessen, App-Gesamt-SOC 12 %; kein Code vorhanden |
| D | 117.7 (`1177`) | feste Stufentabelle `max(10, ⌊(soc% − 10) / 10⌋ · 10)`, **ignoriert** die eingestellte Entladegrenze | etwa **10,9 %**, unabhängig vom Cutoff | vorhanden | Dekompilat `0x08009354` |
| D | 118 | `L = 100 − DoD`, Schwelle `max(soc% − 10, L)`, bei `soc% ≤ L` → `L − 1` | **Cutoff + 0,9 %** (12,9 % bei Cutoff 12 %) | vorhanden | Dekompilat `0x080093bc`; am 10./11.10.2026 an einem Venus D (BMS 118) gemessen |
| A | 109 | Funktionskörper **Byte für Byte identisch** zu A 1105 (`0x08009384`) | **Cutoff + 0,9 %** | identisch zu 1105 (`ModeBySoc`, `Mode1/2/3`, `ChargeStepTarget`, `Discharge_StepRotate`) | Byte-Vergleich, nicht dekompiliert; von Dritten an einem Venus A mit 2 Packs gemessen (Cutoff 12 %): Ein Pack, das schon bei 12,1 % stand, wurde unter 12,0 % abgegeben (Schwelle `L − 1`), der endgültige Stopp kam bei Gesamt-SOC 12 %, das andere Pack stand bei 14,1 % |
| A | 1105 | gleiche Formel wie D 118, nur Adressen anders | **Cutoff + 0,9 %** | vorhanden | Dekompilat `0x0800942c`, nicht gemessen |
| A | 110 | identisch zu A 1105 (`0x08009384`); Pack-Auswahl unverändert, nur das Stromlimit weicht ab (40 A) | **Cutoff + 0,9 %** | identisch / nur Call-Ziele anders | Byte-Vergleich und Dekompilat, nicht gemessen |
| A | 115 | Schwellenfunktion Byte-identisch zu 1105 (`0x08009EB4`); **Stale Threshold behoben**, neuer Abstandswechsel (siehe unten) | **Cutoff + 0,9 %** | `ModeBySoc`, `Mode1/2/3` identisch; `ChargeStepTarget`, `Discharge_StepRotate`, `ModeDispatch` geändert | Dekompilat, nicht gemessen |

Anmerkungen zur Tabelle:

- `soc%` ist der SOC ×10 geteilt durch 10 als **Ganzzahl** (12,9 % → 12). Der Wechsel passiert deshalb,
  sobald das Pack unter `L + 1` fällt, und zeigt sich als `L + 0,9 %`, nicht als `L`.
- `L = 100 − DoD`; die Control begrenzt den Cutoff auf 12–70 % (App-Bereich DoD 30–88 %, Standard 12 %).
  Bis die Control ein DoD gesendet hat, gilt im BMS der Standard DoD 90 (`0x5A`), also `L = 10`.
- Die Schwelle wird nur bei einem Packwechsel neu berechnet; ein veralteter Wert lässt ein Pack unter
  `L` laufen (Einzelheiten: `BMS/VNSD_BMS_Pack_Rotation_Taper_v118_de.md` §5.3).
- Nur per Byte-Suche und Dekompilat geprüft. Beim A 109 nicht verglichen: der CAN-Heartbeat-Pfad, der
  `g_dischargeLimitPct` zur Laufzeit aktualisiert (in A 1105 und D 118 vorhanden, in D 117.7 nicht).
- **A 115 unterscheidet sich in zwei Punkten** (Einzelheiten: `BMS/VNSA_BMS_Diff_v110_v115_de.md`,
  Release-Note-Punkte 1 und 2):
  - `PackSelect_Discharge_StepRotate` berechnet die gespeicherte Schwelle neu, sobald sie mehr als 10
    Punkte unter dem SOC des aktiven Packs liegt (dynamische Zielkorrektur). Damit entfällt der Stale
    Threshold von D 118 / A 1105 (§5.3 in `BMS/VNSD_BMS_Pack_Rotation_Taper_v118_de.md`).
  - Neu ist `PackSelect_SocImbalanceSwitch` (`0x08007B98`, aus `PackSelect_ModeDispatch` im Modus 0
    aufgerufen): Beträgt der Abstand zwischen vollstem und leerstem Online-Pack ≥ 20 Punkte, wird nach
    einer leistungsabhängigen Verzögerung von 1200 / 1800 / 2400 / 3000 Ticks (> 1500 W / > 1000 W /
    > 500 W / sonst) auf das vollste Pack (Entladen) bzw. das leerste (Laden) gewechselt.
  - Die Schwelle selbst (`L = 100 − DoD`, Ganzzahlvergleich) ist unverändert, der Wechselpunkt bleibt bei
    Cutoff + 0,9 %.

**E3:** eigene BMS-Firmware. Das Archiv enthält die Images 106, 110, 112, 113 und 115; 115 ist
dekompiliert (`BMS/VNSE3_BMS_FW_v115_Vollanalyse_de.md`). Es meldet ein Pack und hat keine
Pack-Auswahl und keine Rotation, die Schwellentabelle oben gilt hier also nicht. Das Archiv enthält keinen
Taper-Code. Der E3-Taper ist deshalb unbekannt, und die Modbus-Adresse des Stromlimits beim E3 ist noch
nicht belegt.

## 4. Firmware-Defekte über die Modelle

### Exception-Frame — auf allen drei Modellen belegt

**Venus E3.** `Modbus_Dispatcher` liegt bei `0x0801d278` und ist mit 392 Byte exakt so groß
wie im D. Die Bytes ab `0x0801d358` sind Zeichen für Zeichen dieselben wie im D.

**Venus A.** `Modbus_Dispatcher` liegt bei `0x0801e3ac`, ebenfalls 392 Byte. Von den 392 Byte
unterscheiden sich 23 vom D, und zwar ausschließlich an Sprungoffsets und im Literal-Pool.
Das Fenster um die fehlerhafte Stelle ist identisch:

```
D 0801ea04+0xE0: 00 BF 00 BF 2C 48 00 78 80 B1 00 20 29 49 08 71 04 20 48 71 08 46 C0 79 80 30 C8 71 26 48 00 78
A 0801e3ac+0xE0: 00 BF 00 BF 2C 48 00 78 80 B1 00 20 29 49 08 71 04 20 48 71 08 46 C0 79 80 30 C8 71 26 48 00 78
                                                        ^^^^^  movs r0,#4
```

`movs r0,#4` steht in beiden Images an Offset `0xF0` des Dispatchers, im A also bei
`0x0801e49c`. Eine Exception-PDU ist immer zwei Byte lang, das MBAP-Längenfeld müsste
folglich 3 lauten.

**Damit tragen Venus D, Venus A und Venus E3 denselben Defekt, statisch nachgewiesen.**
Am Gerät geprüft ist bisher nur der D.

### Rücklesewert 44002/44003 — im A dieselbe Konstruktion

Der Lesepfad des A ist unverändert: beide Register werden aus einem Schattenwert bedient, den
nur ein Schreibzugriff oder ein CAN-Frame füllt. Der Zeiger `DAT_08052360` löst sich zu
**`0x20000156`** auf — **derselben SRAM-Adresse wie im D**. Der Defekt „liest nach einem
Neustart dauerhaft 0" gilt damit auch für den Venus A. Am Gerät geprüft ist nur der D.

### Read-Serializer

Auf A und E3 nicht einzeln geprüft. `Read_Serializer` ist im A erfasst (Abschnitt 2) und
zeigt im Dekompilat-Vergleich keine Abweichung zum D, der Vorzeichenfehler ist dort also mit
hoher Wahrscheinlichkeit ebenfalls vorhanden.

---

## 4b. Ungeprüft

- **A v1509 gegen A v150** — was die neuere A-Firmware geändert hat. Jetzt sinnvoll
  durchführbar, weil das A v150 zu 73,4 % erschlossen ist. Für v1509 empfiehlt sich derselbe
  Ankerlauf aus Abschnitt 1d.
- **Die Vorhersagen aus Abschnitt 3a am Venus A**: 2000 nach 44002 → angenommen, Wirkung
  1500; 2000 nach 44003 → Exception 3.
- **Ob der Venus A eine DE/RE-Leitung hat** (Abschnitt 3d) — am Gerät oder am Schaltplan.
- **`FUN_08015bd0`, `FUN_08059408`, `FUN_080025a4`, `FUN_08017f94`** und rund 20 weitere
  kleine Funktionen, für die im A an der erwarteten Stelle gar keine Funktion liegt. Diese
  Restliste ist noch nicht abgearbeitet.

---

## 5. Methodisches

Vier Fallstricke, die in diesem Vergleich aufgetreten sind:

**„Changed" im Diff heißt nicht geändert.** Bei ungleich weit disassemblierten Images
dominieren Callee-Artefakte die Rangliste. Erst `similarity`, `sizeDelta` und der verwendende
Korrelator zusammen sagen, ob eine echte Abweichung vorliegt.

**Eine Suche über ein halb disassembliertes Image liefert falsche Negative.** Dass die
Registerkonstanten im A nicht gefunden wurden, hieß nicht, dass es sie nicht gibt — und wie
Abschnitt 2 zeigt, gibt es sie auch im vollständigen D-Image nicht. Eine Negativsuche ist nur
dann aussagekräftig, wenn sie vorher an einem Image validiert wurde, in dem der gesuchte Code
sicher erfasst ist.

**Eine Negativsuche ist nur so gut wie die umgerechnete Konstante.** Hier wurde nach `0x5DD`
für 1500 und `0x9C5` für 2500 gesucht; richtig sind `0x5DC` und `0x9C4`. Beide Male ergab die
falsche Suche null Treffer, und aus diesen null Treffern wurde eine inhaltliche Aussage
abgeleitet — die sich als das genaue Gegenteil der Wahrheit herausstellte. Bei einer Suche,
deren Ergebnis „nichts gefunden" lautet, gehört die Umrechnung gegengeprüft, am besten an
einem Image, in dem der Wert sicher vorkommt.

**Ein Bytevergleich ohne Relokationsbewusstsein unterschätzt die Ähnlichkeit systematisch.**
Zwischen A und D sind Literal-Pool-Zeiger und `bl`-Sprungziele um den Layout-Versatz
verschoben. Roh verglichen landen identische Funktionen bei 88–94 % und fallen durch jede
vernünftige Schwelle; rechnet man den Versatz heraus, sind es 100 %.

**Und eine positive Beobachtung:** ein Dekompilat-Diff mit normalisierten Symbolnamen trennt
echte Logikunterschiede sauber vom Adressrauschen. `Grid_Export_Power_Limiter` sah im
Bytevergleich mit 53 % nach einer massiven Abweichung aus und hatte tatsächlich drei
geänderte Zeilen. Umgekehrt sah `Write_Handler` mit 95,7 % fast identisch aus und enthielt
genau die eine Konstante, auf die es ankam.
