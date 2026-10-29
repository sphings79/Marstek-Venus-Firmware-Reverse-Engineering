# Venus D – Innenfotos (Teardown)

> 🇬🇧 English version: [VNSD_Teardown_Photos.md](VNSD_Teardown_Photos.md)

Fotos eines geöffneten Venus D (Deckel abgenommen). EXIF-/GPS-Metadaten wurden entfernt und die Drehung in die Pixel eingebrannt.
In keinem Foto sind Seriennummern, MAC-Adressen oder Aufkleber mit Geräte-IDs lesbar.

## Hauptbefund: Die Elektronik ist vergossen

Die gesamte Leistungsplatine ist mit schwarzer/grauer Vergussmasse bedeckt. Bauteile sind nur dort erkennbar, wo sie
herausragen oder die Masse abgewischt wurde: Stecker, Relais, einige Elkos, Kabelbeschriftungen.
Aufdrucke auf ICs (MCU, MOSFETs, Treiber) sind **nicht** lesbar, die MCU lässt sich daher nicht anhand der Fotos identifizieren.
Das Entfernen der Masse würde die Platine zerstören; es wurde nicht versucht.

## Was zu sehen ist

| Foto | Inhalt |
|---|---|
| `VNSD_Teardown_10_potted_pcb_overview_wifi.jpg`, `VNSD_Teardown_11_potted_pcb_overview_battery.jpg`, `VNSD_Teardown_01_potted_pcb_rj45_corner.jpg` | Draufsichten. Unterer Rand: LED-Leiste und das Funkmodul mit Aufdruck `YDCFH0CA2AA_WIFI_X1`. Oben rechts: zwei RJ45-Buchsen mit Ferrit-Hülsen und die Platinenstecker J20/J26. |
| `VNSD_Teardown_08_potted_pcb_angle_battery.jpg`, `VNSD_Teardown_09_potted_pcb_angle_rear.jpg` | Schrägansichten über die vergossene Platine (Magnetik, große schwarze Elkos, Verdrahtung von der PV-Seite). |
| `VNSD_Teardown_02_battery_terminals_side.jpg`, `VNSD_Teardown_07_battery_terminals_button.jpg` | Batterieanschlüsse mit Beschriftung `B+` / `B-` (zwei Schraubösen, rote/schwarze dicke Leitungen). Dahinter zwei Reihen mit je 5 Elkos (vermutlich Zwischenkreis auf der Batterieseite). Unten links: beleuchteter Taster mit blauem Ring, 4 Adern (rot, gelb, blau, schwarz). |
| `VNSD_Teardown_03_pv_input_mc4.jpg` | PV-Eingang: zwei MC4-Stecker, 4 rote + 4 schwarze Leitungen (Aufdruck `600V`, 10 AWG) auf eine Klemmleiste und vier Leitungen mit Ferrit-Hülsen; darunter weitere Ferrit-Drosseln. |
| `VNSD_Teardown_05_rs485_connector_j26.jpg` | Platinenstecker **J26** mit Siebdruck `GND / 485B / 485A / 5V` (RS485 plus 5 V). Daneben Stecker J20. Beide werden über Kabel zu den zwei RJ45-Buchsen gespeist. |
| `VNSD_Teardown_04_comm_cable_label.jpg`, `VNSD_Teardown_15_cable_label_handwritten.jpg` | Kabel mit Hülse `通讯线` („Kommunikationsleitung“), 1 rote + 3 schwarze Adern; das zweite Foto zeigt eine handbeschriebene Hülse (nicht lesbar). |
| `VNSD_Teardown_06_relays_hf140ff_hf115f.jpg` | Zwei Hongfa-Relais **HF140FF-G** (012-2HSWTF, 16 A 250 VAC) und ein Hongfa HF115F. |
| `VNSD_Teardown_12_relay_hf118f.jpg` | Hongfa-Relais **HF118F** (012-1DS6T, 10 A 250 VAC). |
| `VNSD_Teardown_13_capacitor_hec_470uf.jpg` | HEC-Festkörper-/Hybrid-Elko 470 µF / 25 V neben einem gelben Übertrager (Niederspannungsversorgung). |
| `VNSD_Teardown_14_capacitors_closeup.jpg` | Nahaufnahme der schwarzen Zwischenkreis-Elkos; unscharf, wenig Information. |

## Hinweise für die Register-/Firmware-Arbeit

- J26 bestätigt, dass die RJ45-Buchsen ein RS485-Paar führen; die Pinreihenfolge am Stecker selbst ist `GND, 485B, 485A, 5V`.
  Die Zuordnung der J26/J20-Pins zu den RJ45-Pins ist nicht geprüft (Aderfarben nur teilweise sichtbar).
- Relais-Typen sind wie aufgedruckt angegeben; ihre Funktion (Netz, Bypass, EPS, Vorladung) ist aus den Fotos nicht bestimmt.
- Welche Hardware-Revision des Venus D die Fotos zeigen, ist nicht festgehalten.
