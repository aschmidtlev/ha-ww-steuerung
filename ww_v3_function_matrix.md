# ww_v3 – Funktionsmatrix (Delta gegenüber ww_v2)

Diese Matrix listet nur Funktionen, die sich durch die EMS-ESP-Migration
inhaltlich geaendert haben, sowie alle neuen Funktionen. Fuer alle 34
Funktionen aus v2 (F1–F34), die **unveraendert** uebernommen wurden (reine
`ww_v2_` → `ww_v3_`-Umbenennung, identisches Verhalten), siehe
`ww_v2_function_matrix.md` – das betrifft insbesondere die komplette
Preis-/PV-/Batteriespeicher-/Lern-/Kosten-/Debug-Logik (F2–F16, F18, F20–F34).

| # | Funktion | v2 | v3 | Aenderung | Testmethode |
|---|---|---|---|---|---|
| F1 | Ist-Temperatur-Ermittlung | `water_heater.dhw1`/`current_temperature`, `is not number`-Workaround fuer String-Zahl | `sensor.boiler_dhw_curtemp`, direkter numerischer Sensor | **geaendert**: einfacherer, robusterer Template-Code (kein Attribut-Zugriff mehr), lokal statt Cloud | Testplan Szenario 1-2, 5-7 (angepasst) |
| F17 | Ladung starten | `number.set_value` (Zieltemp + feste 180-min-Dauer) + `switch.turn_on` auf Bosch-Cloud-Entities | `number.set_value` (Zieltemp, auf Geraetemaximum gekappt) + `switch.turn_on` auf EMS-ESP-Entities, kein Dauer-Wert mehr gesetzt | **geaendert**: siehe `ww_v3_migration_concept.md` 3.3 | Testplan Szenario 24-25, 32-33 (angepasst) |
| F33 | Pflichtdaten-Pruefung vor jeder Aktion | prueft `water_heater.dhw1`/`switch.charge` | prueft `sensor.boiler_dhw_curtemp`/`switch.boiler_dhw_onetime`/`number.boiler_dhw_seltempsingle` | **geaendert**: dritte Pflicht-Entity ergaenzt (die Zieltemperatur-Number wird jetzt direkt vom Skript beschrieben) | Testplan Szenario 5-7, 24 |
| F34 | Diagnose fehlender Pflicht-Entities | Liste mit `water_heater.dhw1`/`switch.charge` | Liste mit den drei EMS-ESP-Entities | **geaendert** | Testplan Szenario 34-36 |
| F35 | Geraete-Maximaltemperatur pruefen | **nicht vorhanden** (unverifizierbar, siehe `ww_v2_open_questions.md`) | `number.boiler_dhw_maxtemp` wird vor jeder Ladung gelesen, Ziel wird bei Ueberschreitung gekappt, Warnung geloggt | **neu** | Testplan Szenario "Geraetemaximum" (neu) |
| F36 | Hardware-Ladebestaetigung | **nicht vorhanden** (kein unabhaengiges Signal in der Bosch-Cloud-Anbindung verfuegbar) | `binary_sensor.ww_v3_ladung_aktiv_hardware` aus `binary_sensor.boiler_dhw_charging`/`_recharging`; Debug-Warnung nach 10 min Befehl-ohne-Wirkung | **neu** | Testplan Szenario "Hardware-Reaktion" (neu) |
| F37 | Sperre natives PV-Aufladen | **nicht vorhanden** (Funktion existierte in der Bosch-Cloud-Anbindung nicht) | `automation.ww_v3_pvenabledhw_sperren` haelt `switch.thermostat_pvenabledhw` aus, solange `ww_v3_aktiv` an ist | **neu** | Testplan Szenario "PV-Aufladen-Sperre" (neu) |
| F38 | Hygieneziel-Erreichbarkeit pruefen | **nicht vorhanden** | `binary_sensor.ww_v3_hygieneziel_ueber_geraetemaximum` vergleicht `input_number.ww_v3_hygiene_ziel` mit `number.boiler_dhw_maxtemp`, sichtbar im Dashboard | **neu** | manuelle Pruefung vor Aktivierung des Hygienezyklus (siehe `ww_v3_installation.md`) |

## Entfallene Funktion

| Funktion | Grund |
|---|---|
| Setzen einer Geraete-Ladedauer (`number.charge_duration`) | Kein passendes EMS-ESP-Aequivalent fuer `switch.boiler_dhw_onetime`; durch F17-Aenderung (siehe oben) und die bereits bestehenden Sicherheitsmechanismen (F19-Aequivalent `ww_v3_sicherheitsabschaltung_maxdauer`, weiterhin unveraendert vorhanden) vollstaendig kompensiert |
