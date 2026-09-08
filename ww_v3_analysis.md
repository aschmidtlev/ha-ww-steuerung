# ww_v3 – Gap-Analyse: ww_v2 vs. EMS-ESP-Gateway

Stand: 2026-09-08. Analysierte Quellen:

- `ww_v2_package.yaml` (1761 Zeilen), `ww_v2_dashboard.yaml` (603 Zeilen) sowie
  die vollstaendige ww_v2-Dokumentation (`ww_v2_analysis.md`,
  `ww_v2_entity_catalog.md`, `ww_v2_entity_mapping.md`,
  `ww_v2_function_matrix.md`, `ww_v2_quality_report.md`, `ww_v2_testplan.md`,
  `ww_v2_installation.md`, `ww_v2_open_questions.md`, `ww_v2_changelog.md`)
- `CSV/harvest_2026-09-08-20-33-39.csv` (258 Zeilen, Entity-Snapshot des neu
  installierten EMS-ESP-Gateways vom 08.09.2026, 22:29-22:30 Uhr)

Keine dieser Dateien wurde veraendert. Alle nachfolgend beschriebenen
Entscheidungen wurden vom Anwender im Rahmen der Aufgabenstellung explizit
bestaetigt (siehe `ww_v3_open_questions.md`, Teil A).

## 1. Ausgangslage: ww_v2 und seine Bosch-Cloud-Abhaengigkeit

ww_v2 (siehe `ww_v2_analysis.md`) wurde am 31.08.2026 vollstaendig neu gegen
v1_8 aufgebaut und referenzierte fuer die zentrale Warmwasser-Ist-Temperatur
und die Ladesteuerung ausschliesslich `water_heater.dhw1` (Attribut
`current_temperature`), `switch.charge`, `number.charge_setpoint` und
`number.charge_duration`. Diese vier Entities stammen technisch aus der
**Bosch Home Connect Cloud-Integration** (erkennbar u. a. am Attribut
`bosch_state` auf `water_heater.dhw1`, siehe `ww_v2_open_questions.md` Frage
1). Sie wurden in v2 als einzige Ausnahme vom generellen Bosch-Ausschluss
verwendet, weil der Anwender sie zum damaligen Zeitpunkt explizit als einzige
verfuegbare Quelle bestaetigt hatte (kein lokales Gateway vorhanden).

Am 08.09.2026 wurde ein **EMS-ESP-Gateway** installiert (offene-Source-
Firmware fuer ESP8266/ESP32, liest den EMS-Bus des Bosch/Buderus/Junkers-
Heizsystems direkt und lokal aus, ohne Cloud-Anbindung). Es stellt ueber die
offizielle EMS-ESP-HA-Integration 258 native Home-Assistant-Entities bereit
(Domainverteilung: 101 sensor, 73 number, 31 switch, 27 select,
19 binary_sensor, 5 text, 1 climate, 1 device_tracker). Damit entfaellt die
Notwendigkeit, ueberhaupt noch eine Bosch-Cloud-Entity zu verwenden.

## 2. Gegenueberstellung: bisherige Bosch-Cloud-Entities vs. EMS-ESP-Aequivalente

| Funktion | v2 (Bosch Cloud) | EMS-ESP-Aequivalent | Bewertung |
|---|---|---|---|
| Warmwasser-Ist-Temperatur | `water_heater.dhw1` / Attribut `current_temperature` (String-codierte Zahl, benoetigte `is not number`-Workaround in v2, siehe `ww_v2_changelog.md`) | `sensor.boiler_dhw_curtemp` ("Boiler dhw Current intern temperature", 50.2 °C im Snapshot) – eigenstaendiger numerischer Sensor | **klare Verbesserung**: kein Attribut-Zugriff mehr noetig, direkter Sensor mit eigener `last_updated`, lokal ohne Cloud-Latenz |
| Zweiter Temperaturfuehler (bisher nicht vorhanden) | – | `sensor.boiler_dhw_curtemp2` ("Current extern temperature", 43.4 °C im Snapshot) | **neu**: als Diagnose-/Vergleichswert aufgenommen, nicht fuer Regelentscheidungen (siehe Offene Frage 1) |
| Ladeauslöser | `switch.charge` | `switch.boiler_dhw_onetime` ("Boiler dhw One time charging") – vom Anwender explizit als zu verwendende Entity bestaetigt (statt der Alternative `switch.thermostat_dhw_charge`) | uebernommen |
| Ladezieltemperatur | `number.charge_setpoint` (Startwert 51.0 °C) | `number.boiler_dhw_seltempsingle` ("Single charge temperature", Startwert 51 °C im Snapshot – **identischer Defaultwert**) | uebernommen, sehr passendes Aequivalent |
| Ladedauer-Begrenzung | `number.charge_duration` (v2 setzte fest 180 min, um die eigene Sicherheitsabschaltung nicht durch das Geraet vorzeitig limitieren zu lassen) | keine 1:1 passende Entity fuer `switch.boiler_dhw_onetime`; `number.thermostat_dhw_chargeduration` gehoert zum getrennten `switch.thermostat_dhw_charge` | **entfaellt** – ww_v3 verlaesst sich ausschliesslich auf die eigenen zwei Sicherheitsmechanismen (Stop-am-Ziel, Sicherheitsabschaltung-Maxdauer), siehe `ww_v3_changelog.md` |
| Geraete-Maximaltemperatur (bisher unbekannt/unverifizierbar) | – (v2 konnte `state_attr('number.charge_setpoint','max')` nicht verifizieren, siehe `ww_v2_open_questions.md` B-Teil) | `number.boiler_dhw_maxtemp` (56 °C im Snapshot) – **eigene, direkt lesbare Entity** | **neu, schliesst eine dokumentierte Luecke aus v2**: ww_v3 kappt Ladeziele aktiv auf diesen Wert (siehe Skript `ww_v3_ladung_starten`) |
| Hardware-Ladebestaetigung (bisher nicht vorhanden) | – (v2 kannte nur den eigenen Schalterbefehl, keine unabhaengige Bestaetigung) | `binary_sensor.boiler_dhw_charging`, `binary_sensor.boiler_dhw_recharging` | **neu**: ermoeglicht Erkennung von Befehl-ohne-Wirkung (siehe Automation `ww_v3_debug_ladung_ohne_hardware_reaktion`) |
| Temperatur-OK-Flag des Geraets (bisher nicht vorhanden) | – | `binary_sensor.boiler_dhw_tempok` | als Diagnose-Attribut aufgenommen |
| Natives PV-Aufladen (bisher nicht vorhanden) | – | `switch.thermostat_pvenabledhw` ("Enable raise dhw", im Snapshot **an**) | siehe Abschnitt 4 – wird von ww_v3 aktiv deaktiviert gehalten |

## 3. Weitere EMS-ESP-Funde ohne Aenderung an ww_v3 (bewusst nicht automatisiert)

Diese Entities existieren im EMS-ESP-Gateway, werden aber **nicht** in die
automatische Ladelogik integriert, um den Funktionsumfang nicht ueber die
Aufgabenstellung hinaus zu erweitern. Sie werden lediglich als manuell
bedienbare/beobachtbare Zusatzfunktionen im Dashboard (Bereich "EMS-ESP
native Zusatzfunktionen") aufgenommen:

| Entity | Bedeutung | Grund fuer Nicht-Automatisierung |
|---|---|---|
| `switch.boiler_dhw_dhwprio` | Warmwasser-Prioritaet gegenueber Heizkreis | Wuerde das Verhalten des Heizkreises (nicht Teil dieses Packages) beeinflussen – Entscheidung bewusst beim Anwender belassen |
| `select.boiler_dhw_comfort1` | Komfortmodus ("high comfort" im Snapshot) | Geraeteseitige Grundeinstellung, keine Preis-/PV-Abhaengigkeit vorgesehen |
| `select.boiler_dhw_circmode` | Zirkulationspumpen-Modus | Nicht Teil der urspruenglichen Aufgabenstellung (Komfortwasserzirkulation, kein Ladethema) |
| `select.thermostat_dhw_mode` | Thermostat-Betriebsmodus ("auto" im Snapshot) | Steuert grundsaetzliche Betriebsart des Geraets, nicht preisabhaengig zu automatisieren ohne weitere Rueckfrage |
| `climate.thermostat_hc1` | Heizkreis-Thermostat | Ausserhalb des Funktionsumfangs "Warmwassersteuerung" |

Diese Entities werden nur **read-only bzw. optional manuell bedienbar** im
Dashboard angezeigt (siehe `ww_v3_dashboard.yaml`, View "Steuerung").

## 4. Sonderfall: natives PV-Aufladen (`switch.thermostat_pvenabledhw`)

Das EMS-ESP-Gateway (genauer: der angeschlossene Thermostat) bietet ein
eigenes Feature "PV laedt Warmwasser auf", das im Snapshot **aktiv** war.
Wuerde dieses parallel zur ww_v3-eigenen PV-/Preislogik laufen, koennten zwei
unabhaengige Systeme gleichzeitig und unkoordiniert Ladeentscheidungen
treffen (z. B. das Geraet startet eine Ladung, waehrend ww_v3 aus
Preisgruenden gerade wartet, oder umgekehrt beide gleichzeitig). Der
Anwender hat sich hierzu entschieden: **ww_v3 uebernimmt die alleinige
Kontrolle**, das native Feature wird aktiv deaktiviert gehalten (Automation
`ww_v3_pvenabledhw_sperren`, siehe `ww_v3_package.yaml`). Die Sperre greift
nur, solange `input_boolean.ww_v3_aktiv` an ist (bei manueller
Uebersteuerung darf der Anwender frei entscheiden, siehe Dashboard-Hinweis).

## 5. Nicht in v2 vorhanden gewesene, jetzt verfuegbare Diagnosedaten

Zusaetzlich zu den in Abschnitt 2 genannten Kern-Entities liefert EMS-ESP
umfangreiche weitere Diagnosedaten (Energiezaehler Verdichter/Zusatzheizung,
Betriebsstunden, Start-Zaehler, Spuelventil-Status usw.). Diese wurden
durchgesehen, aber **nicht** in ww_v3 aufgenommen, da sie keinen erkennbaren
Mehrwert fuer die Warmwasser-Preis-/PV-Optimierung bieten und die
Aufgabenstellung explizit vor unnoetiger Funktionsausweitung warnt
("Avoid creating duplicate entities" / "Prefer maintainability"). Sie bleiben
als eigene, native EMS-ESP-Entities in Home Assistant jederzeit verfuegbar
und koennen bei Bedarf direkt (ohne Umweg ueber dieses Package) in einem
Dashboard verwendet werden.

## 6. Zusammenfassung

Die Migration auf EMS-ESP ist eine **strikte Verbesserung** gegenueber v2:
lokale statt Cloud-Anbindung, ein numerischer Sensor statt Attribut-Zugriff,
eine bislang unverifizierbare Geraete-Obergrenze wird jetzt aktiv geprueft,
und es gibt erstmals eine unabhaengige Hardware-Bestaetigung der Ladung.
Gleichzeitig bleibt die vollstaendige Preis-/PV-/Batteriespeicher-Logik von
v2 unveraendert (kein EMS-ESP-Bezug) und wird nicht neu erfunden. Details zur
Umsetzung siehe `ww_v3_migration_concept.md`, `ww_v3_entity_mapping.md` und
`ww_v3_function_matrix.md`.
