# MPPT-Hardware — Momentaufnahme externer Recherche

> 🇬🇧 English version: [MPPT_Hardware_External_Research.md](MPPT_Hardware_External_Research.md)

**Datum:** 2026-10-17. **Methode:** nur Websuche (kein Gerätezugriff). **Das ist eine Momentaufnahme, keine abschließende Aussage** — Foren, Teardown-Blogs und Patentanmeldungen verändern sich; vor erneutem Verlassen auf „nichts gefunden“ neu suchen, besonders wenn Monate vergangen sind.

## Warum das hier steht

Der MPPT-Teilcontroller (separater Chip, per CAN mit dem Control verbunden, Firmware-Version 104 laut CAN-String `V150.116.104.118`, siehe [VNSD_Control_Charge_Power_PV_Curtailment_v150.md](../Control_FW/VNSD_Control_Charge_Power_PV_Curtailment_v150.md) §3.2) liegt diesem Projekt als Image nicht vor. Bevor ein Hardware-Ansatz (Teardown, Chip-Identifikation, SWD-Dump) geplant wird, war es sinnvoll zu prüfen, ob diese Information schon anderswo veröffentlicht wurde.

## Was gesucht wurde

Websuche (Deutsch, Englisch, Chinesisch) zu: Venus D/E/A Teardown, Innenaufnahmen, Platine, serielle Konsole/UART-Zugang, MPPT-Chip-Identifikation, FCC-ID-Einträge, GitHub-Firmware-Dumps und Forenbeiträge (akkudoktor.net, community.simon42.com, Home-Assistant-Community, Homey-Community, photovoltaikforum-nahe Begriffe).

## Ergebnis: nichts gefunden

- **Kein Teardown** mit Fotos vom Innenleben von Venus D/A/E, in keiner Sprache.
- **Keine Diskussion zu serieller Konsole oder UART-Pads** zu irgendeinem Venus-Modell — alles Hardware-Interesse in den Foren dreht sich um die dokumentierten externen Schnittstellen (RS485/Modbus, lokale UDP-API, BLE), nicht um internen Debug-Zugang.
- **Keine MPPT-Chip-Identifikation** irgendwo öffentlich.
- **Keine FCC-ID** für Marstek- oder Hamedata-gebrandete Venus-Geräte gefunden (möglich, dass unter anderem Namen zertifiziert, oder in Märkten ohne diese Pflicht vertrieben — nicht weiter verfolgt).
- **Kein Firmware-Dump-Repository** auf GitHub außer den üblichen Home-Assistant-/Modbus-Integrationen (die mit den dokumentierten Schnittstellen des Geräts sprechen, nicht mit dessen Innenleben).

Das öffentliche Interesse an diesen Geräten dreht sich vollständig um deren **Nutzung** (Modbus-Register, lokale API, HACS-Integrationen, Cloud-Relay-Addons wie „Hame Relay“) — nicht um deren Analyse. Soweit diese Suche es zeigen kann, ist dieses Projekt derzeit die einzige öffentliche Reverse-Engineering-Arbeit an dieser Hardware.

## Der Hersteller, bestätigt

Marstek ist die Endkundenmarke von **Shenzhen Huamei Xingtai Technology Co., Ltd.** (深圳市华美兴泰科技股份有限公司, firmiert als „Hamedata“/„Hame“ — passt zum `_HM`-Build-Suffix, dem Default-WLAN-Passwort `Hame147258` und den `hamedata.com`-Endpunkten, die schon in der Firmware gefunden wurden). Hamedatas eigenes Marketing nennt BMS, PCS, EMS und MPPT als eigene Kerntechnologien, passend dazu, dass kein separater Dritthersteller für MPPT zu identifizieren ist — die MPPT-Firmware stammt vermutlich ebenso von ihnen selbst wie Control- und BMS-Firmware.

## Patente (Google Patents, Anmelder-Suche)

Anmelder `Hamedata` / `深圳市华美兴泰科技` liefert mehrere hundert Treffer (464 bei der breiteren Suche über den chinesischen Firmennamen), fast alle themenfremd (Batteriezellen-Chemie/-Fertigung, Unterhaltungselektronik aus einer älteren Produktlinie). Ein Treffer ist einschlägig:

- **WO2025179896A1**, „Rapid MPPT method, system and apparatus based on unipolar inverter circuit“ — Anmelder Hamedata Technology Co Ltd, angemeldet 2024-10-12, veröffentlicht 2025-09-04, Priorität 2024-02-28. Beschreibt einen MPPT-Algorithmus für einstufige unipolare (Flyback-artige) Wechselrichter: PV-seitige Spannungs-/Strommessung in 45°-Netzphasenschritten, ein Leistungs-Korrekturfaktor aus dem Vergleich aufeinanderfolgender Zyklen, der eine Stromregelschleife des Wechselrichters füttert. Es erwähnt „einen Mikrocontroller“ nur allgemein für den Mess-/Rechenschritt und nennt **keine Chip-Typenbezeichnung, keine CAN-/UART-Details, keine Registeradressen**. Plausibel zur MPPT-Arbeit dieser Produktlinie passend, dokumentiert aber einen Algorithmus, keine Hardware.
- Kein Patent gefunden, das einen konkreten MCU-Typ, einen Bootloader, einen Firmware-Update-Mechanismus oder eine CAN-/Modbus-Registerkarte benennt.

## Fazit

Nichts Externes schließt eine der offenen MPPT-Fragen dieses Projekts (Chip-Identität, Firmware-Image, Bootloader). Die Patentsuche liefert einen Datenpunkt (ein MPPT-Algorithmus-Patent, das dazu passt, dass Hamedata MPPT-Regelungsentwicklung selbst betreibt), aber keine Hardware-Details. Es bleiben nur die Wege, die im Projekt schon geplant sind: physischer Teardown + Chip-Identifikation, danach SWD/JTAG, falls der Chip das unterstützt (siehe [VNSD_SWD_JTAG_Dump_Plan_de.md](VNSD_SWD_JTAG_Dump_Plan_de.md)), oder das Mitschneiden eines MPPT-Firmware-OTA-Transfers, falls Marstek je einen über einen für dieses Projekt beobachtbaren Kanal verteilt.
