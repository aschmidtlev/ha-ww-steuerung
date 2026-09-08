# ww_v3 – Testplan (Delta gegenüber ww_v2)

**Wichtiger Hinweis zur Testmethode:** identisch zu `ww_v2_testplan.md` –
keine laufende Home-Assistant-Instanz verfuegbar, alle Szenarien wurden
**statisch anhand des Codes** in `ww_v3_package.yaml` geprueft. Die
sicherheitsrelevanten Szenarien muessen vor Produktivbetrieb zusaetzlich in
Home Assistant selbst nachgestellt werden.

Referenzwerte unveraendert gegenueber v2 (kritisches Minimum 44 °C,
Komfortminimum 47 °C, Normalziel 51 °C, Maximaltemperatur 55 °C,
Start-Hysterese 2 °C, Mindestlaufzeit 30 min, Anti-Takt-Sperre 20 min, max.
Ladedauer 150 min, Einspeisevergütung 8 ct/kWh). Neu: Geraete-
Maximaltemperatur 56 °C (`number.boiler_dhw_maxtemp`, Snapshot-Wert).

Alle 36 Szenarien aus `ww_v2_testplan.md` bleiben inhaltlich gueltig, nur
mit den migrierten Entity-Namen (z. B. Szenario 1 "Warmwasser deutlich zu
kalt" bezieht sich jetzt auf `sensor.boiler_dhw_curtemp` statt
`water_heater.dhw1`, Szenario 24 "switch nicht verfuegbar" jetzt auf
`switch.boiler_dhw_onetime`). Nachfolgend nur die inhaltlich **neuen bzw.
veraenderten** Szenarien.

| # | Szenario | Ausgangszustand | Erwartete Entscheidung/Aktion | Testergebnis |
|---|---|---|---|---|
| 37 | Ladeziel über Geraetemaximum | `sensor.ww_v3_effektives_ziel` = 55 °C (PV-Waermespeicher-Modus), `number.boiler_dhw_maxtemp` = 50 °C (Beispiel unter dem sonst ueblichen 56 °C) | `script.ww_v3_ladung_starten` kappt `ziel` auf 50 °C, setzt `number.boiler_dhw_seltempsingle` auf 50, loggt "Ziel begrenzt" mit severity warning, startet Ladung trotzdem mit dem gekappten Wert | statisch geprüft: korrekt (F35) |
| 38 | Hygieneziel über Geraetemaximum | `input_number.ww_v3_hygiene_ziel` = 65 °C, `number.boiler_dhw_maxtemp` = 56 °C | `binary_sensor.ww_v3_hygieneziel_ueber_geraetemaximum` = on, sichtbar im Dashboard (Hygiene-Monitoring-Karte); ein trotzdem gestarteter Hygienezyklus wuerde durch dieselbe Kappungslogik wie Szenario 37 auf 56 °C begrenzt, das konfigurierte 65-°C-Ziel wird real nie erreicht, `wait_template` laeuft in den 2:30h-Timeout | statisch geprüft: **Einschraenkung dokumentiert** – Anwender sollte `ww_v3_hygiene_ziel` vor Aktivierung des Hygienezyklus auf ≤ `number.boiler_dhw_maxtemp` pruefen (siehe `ww_v3_installation.md`) |
| 39 | Befehl gesetzt, Geraet reagiert nicht | `switch.boiler_dhw_onetime` = on seit > 10 min, `binary_sensor.boiler_dhw_charging` = off, `binary_sensor.boiler_dhw_recharging` = off | `automation.ww_v3_debug_ladung_ohne_hardware_reaktion` loggt Warnung "Moegliche Geraetestoerung erkannt", **keine automatische Korrekturaktion** (bewusst, um kein ungetestetes automatisches Fehlerverhalten einzufuehren) | statisch geprüft: korrekt (F36) |
| 40 | Natives PV-Aufladen wird eingeschaltet | `switch.thermostat_pvenabledhw` wechselt von off auf on (z. B. durch Bedienung am Geraet), `input_boolean.ww_v3_aktiv` = on | `automation.ww_v3_pvenabledhw_sperren` schaltet den Schalter sofort wieder aus, loggt info (ohne Pushover, `send_pushover: false`) | statisch geprüft: korrekt (F37) |
| 41 | Natives PV-Aufladen bei manueller Uebersteuerung | wie #40, aber `input_boolean.ww_v3_aktiv` = off | Automation greift **nicht** (Bedingung `ww_v3_aktiv: on` erfuellt nicht) – Anwender kann das native Feature bei deaktivierter Automatik frei nutzen | statisch geprüft: korrekt, bewusste Design-Entscheidung |
| 42 | HA-Neustart mit noch aktivem nativen PV-Aufladen | `switch.thermostat_pvenabledhw` = on vor Neustart | `platform: homeassistant, event: start`-Trigger schaltet es nach dem Start aus (sofern `ww_v3_aktiv` on ist) | statisch geprüft: korrekt |
| 43 | Zwei Temperaturfuehler weichen stark voneinander ab | `sensor.boiler_dhw_curtemp` = 50 °C, `sensor.boiler_dhw_curtemp2` = 35 °C | Keine automatische Reaktion (bewusste Design-Entscheidung, curtemp2 ist reiner Diagnosewert); Abweichung ist ueber das Attribut `temperatur_extern` auf der Debug-Seite sichtbar und muss vom Anwender manuell bewertet werden | statisch geprüft: **bewusste Design-Entscheidung**, dokumentiert (siehe Offene Frage 1) |
| 44 (ersetzt v2 #24) | `switch.boiler_dhw_onetime` nicht verfügbar | `switch.boiler_dhw_onetime` = unavailable | `ww_v3_ladung_starten` bricht mit `stop` ab, bevor irgendein Service aufgerufen wird; `sensor.ww_v3_diagnose_pflichtentities` listet die Entity | statisch geprüft: korrekt (identisches Verhalten wie v2 #24, nur neue Entity) |

## Zusammenfassung neuer dokumentierter Einschraenkungen

1. **Szenario 38** – ein Hygieneziel oberhalb der Geraete-Maximaltemperatur
   wird zwar erkannt und angezeigt (`ww_v3_hygieneziel_ueber_
   geraetemaximum`), aber nicht automatisch verhindert (der Hygienezyklus
   ist ohnehin standardmaessig deaktiviert und muss vom Anwender bewusst
   freigegeben werden). Siehe `ww_v3_open_questions.md`.
2. **Szenario 39** – Erkennung einer moeglichen Geraetestoerung ist rein
   informativ (Pushover-Warnung), keine automatische Korrektur- oder
   Abschaltaktion wird ausgeloest.
