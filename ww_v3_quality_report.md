# ww_v3 – Qualitätsbericht

**Wichtiger Hinweis zur Prüftiefe:** Wie bereits bei v2 steht in dieser
Umgebung keine laufende Home-Assistant-Instanz zur Verfügung. `ha core
check_config` sowie ein echtes Auslösen von Automationen/Templates in
Developer Tools konnten daher **nicht** durchgeführt werden. Alle Prüfungen
unten sind **statische Code-/Text-Analysen** (grep-Suchen, manuelle
Zeile-für-Zeile-Durchsicht). Vor Produktivbetrieb ist Abschnitt 1 der
`ww_v3_installation.md` ("Konfiguration überprüfen") zwingend zusätzlich in
der echten HA-Instanz durchzuführen.

## 1. Checkliste (Delta gegenüber ww_v2_quality_report.md)

| Prüfpunkt | Methode | Ergebnis |
|---|---|---|
| Keine Bosch-Referenzen (jetzt: **keine Ausnahme mehr**) | `grep -rni bosch ww_v3_package.yaml ww_v3_dashboard.yaml` | Alle Treffer ausschliesslich in erlaeuternden Kommentarzeilen (6 Treffer, siehe Abschnitt 4), **keine einzige tatsaechliche Entity-Referenz** – anders als v2, das `water_heater.dhw1` noch als bestaetigte Ausnahme referenzierte |
| Keine Bosch-Cloud-Entity-IDs referenziert | `grep -nE "water_heater\.dhw1\|switch\.charge\b\|number\.charge_setpoint\|number\.charge_duration" ww_v3_package.yaml ww_v3_dashboard.yaml` | Alle Treffer ausschliesslich in erlaeuternden Kommentarzeilen (Migrationsdokumentation im Dateikopf/an den geaenderten Stellen), keine aktive Referenz |
| Alle referenzierten EMS-ESP-Entities existieren nachweislich | Abgleich jeder neu referenzierten `sensor.boiler_*`/`switch.boiler_*`/`number.boiler_*`/`switch.thermostat_*`/`select.*`/`binary_sensor.boiler_*`-Entity gegen `CSV/harvest_2026-09-08-20-33-39.csv` | Alle 14 direkt referenzierten EMS-ESP-Entity-IDs wurden einzeln gegen die CSV geprueft und **bestaetigt vorhanden** (siehe Liste in `ww_v3_analysis.md` Abschnitt 2) |
| Keine doppelten Unique IDs | `grep -o "unique_id: ..." ww_v3_package.yaml \| sort \| uniq -d` | leer, keine Duplikate |
| Keine doppelten Automations-IDs | `grep -o "id: ww_v3_..." ww_v3_package.yaml \| sort \| uniq -d` | leer, keine Duplikate |
| Alle Dashboard-Entities definiert | jede im Dashboard referenzierte `ww_v3_*`-Entity gegen `ww_v3_package.yaml` abgeglichen, jede direkt referenzierte EMS-ESP-Entity gegen die CSV | vollstaendig abgeglichen |
| Templates gegen `unknown`/`unavailable` abgesichert | jedes neue/geaenderte `state:`-Template geprueft | `sensor.ww_v3_warmwassertemperatur` hat weiterhin ein explizites `availability:`-Gate (jetzt auf `sensor.boiler_dhw_curtemp` statt `water_heater.dhw1`); `binary_sensor.ww_v3_pflichtdaten_gueltig` prueft alle drei EMS-ESP-Pflicht-Entities einzeln |
| Sichere Division | Divisionsstellen erneut geprueft (`/ 60`, `/ 100`, `/ 4`) | keine neuen Divisionsstellen hinzugekommen, weiterhin nur durch Konstanten |
| Geraeteseitige Obergrenze wird respektiert | Codepfad `script.ww_v3_ladung_starten` geprueft | `ziel = [ziel_gewuenscht, geraet_max] | min` – kann rechnerisch nie ueber `number.boiler_dhw_maxtemp` hinausgehen; Kappung wird geloggt (severity warning) |
| Charge wird nicht mehrfach unnoetig ausgeloest | `script.ww_v3_ladung_starten` frueher `stop`, wenn `switch.boiler_dhw_onetime` bereits `on` | bestaetigt, identisches Verhalten wie v2, nur auf neuer Entity |
| Neustartverhalten sicher | `automation.ww_v3_sicherheitsabschaltung_maxdauer` unveraendert vorhanden, zusaetzlich `automation.ww_v3_pvenabledhw_sperren` reagiert auch auf `homeassistant: event: start` | bestaetigt |
| Keine offensichtlichen Race Conditions | Prioritaetenkette P0 vs. wirtschaftliche Ladung erneut geprueft (jetzt gegen `switch.boiler_dhw_onetime`) | unveraendert korrekt, gleiche Absicherung wie v2 |
| Neue Automation `ww_v3_pvenabledhw_sperren` kann keine Bedien-Schleife erzeugen | Trigger `to: "on"` + Aktion `turn_off` desselben Schalters | Der `turn_off`-Service loest **keinen** erneuten `to: "on"`-Trigger aus (Zustandswechsel ist `on`→`off`), daher keine Endlosschleife. Einzige Wiederholung waere ein erneutes externes Einschalten (z. B. am Thermostat), was korrekt erneut erkannt und zurueckgesetzt wird |
| `entity_category: diagnostic` korrekt verwendet | Abgleich, dass nur reine Status-/Diagnose-Entities markiert wurden, keine fuer die Regelentscheidung direkt relevanten Sensoren (z. B. `ww_v3_warmwassertemperatur`, `ww_v3_kritischer_bedarf`, `ww_v3_effektives_ziel` bleiben undiagnostisch/primaer) | bestaetigt, siehe `ww_v3_package.yaml` |

Alle uebrigen Pruefpunkte aus `ww_v2_quality_report.md` Abschnitt 1 (YAML-
Parsebarkeit, Einrueckung, Automationsmodi, Recorder-Last, Dashboard-
Kartentypen usw.) wurden erneut durchgesehen und sind unveraendert gueltig,
da die entsprechenden Codeabschnitte nicht durch die Migration beruehrt
wurden.

## 2. Design-Entscheidung: Bedienung vs. Status bei `switch.boiler_dhw_onetime`

Identische Design-Entscheidung wie in v2 (siehe `ww_v2_quality_report.md`
Abschnitt 2): `switch.boiler_dhw_onetime` wird in der Uebersicht/Steuerung
nicht als bedienbarer Schalter gezeigt. Neu in v3: die Debug-Seite zeigt
jetzt **zwei** unabhaengige Statuswerte statt einem – den eigenen Befehl
(`binary_sensor.ww_v3_ladung_aktiv`) und die Hardware-Bestaetigung
(`binary_sensor.ww_v3_ladung_aktiv_hardware`). Das war mit der
Bosch-Cloud-Anbindung nicht moeglich und ist eine echte Transparenz-
Verbesserung.

## 3. Dokumentierte technische Einschraenkungen (unveraendert gueltig)

Alle drei in `ww_v2_quality_report.md` Abschnitt 3 dokumentierten
Einschraenkungen (keine echte Pushover-Zustellbestaetigung, kein
Erfolgs-Log bei fehlgeschlagenem `switch.turn_on`, "veraltete Temperatur"
ist nur Anzeige) gelten unveraendert fuer v3 – sie betreffen die
Debug-/Pushover-Infrastruktur bzw. HA-Plattformgrenzen, nicht die
EMS-ESP-Migration selbst.

**Neu hinzugekommene Einschraenkung:**

4. **Keine Garantie, dass `switch.boiler_dhw_onetime` sich nach
   Ladeabschluss selbst zuruecksetzt.** EMS-ESP-Firmware-Verhalten hierzu
   ist aus der vorliegenden CSV nicht ableitbar (nur ein Zustands-Snapshot,
   keine Verhaltensdokumentation). ww_v3 verlaesst sich **nicht** auf ein
   eventuelles automatisches Zuruecksetzen, sondern schaltet den Schalter in
   `automation.ww_v3_stop_ziel` und `automation.ww_v3_
   sicherheitsabschaltung_maxdauer` immer explizit aus – falls die Firmware
   den Schalter bereits selbst zurueckgesetzt hat, ist der `switch.turn_off`-
   Aufruf ein wirkungsloser, unschaedlicher No-Op. Siehe
   `ww_v3_open_questions.md`, Offene Frage 2.

## 4. Bosch-Volltextsuche (wiederholt für v3)

```
grep -rni "bosch" ww_v3_package.yaml ww_v3_dashboard.yaml
→ 6 Treffer, alle in erlaeuternden Kommentarzeilen (Migrationsdokumentation:
  "Migration von ww_v2 (Bosch-Cloud-Anbindung ...)"), keine Entity-Referenz

grep -nE "water_heater\.dhw1|switch\.charge\b|number\.charge_setpoint|
          number\.charge_duration" ww_v3_package.yaml ww_v3_dashboard.yaml
→ alle Treffer in Kommentarzeilen, die die Migration dokumentieren
  (z. B. "Ersetzt water_heater.dhw1/..."), keine aktive Referenz

grep -niE "hotwater_temp|dhw1_temperature|climate\.hc1|holiday_dhw|
           holiday_hc|holiday_mode|ww_bosch|actual_supply_temp|
           outdoor_temperature|return_temp|chimney|pool_temperature|
           burner_power|system_pressure|numberofstarts|health_status|
           notifications|start_time|supply_temp_setpoint|
           total_system_uptime|chimneysweeper|flamestatus|
           bosch_thermostat" ww_v3_package.yaml ww_v3_dashboard.yaml
→ keine Treffer
```

**Ergebnis: ww_v3 enthaelt erstmals seit Beginn dieses Projekts (v1_8 → v2 →
v3) keine einzige Bosch-Referenz mehr – auch nicht als dokumentierte
Ausnahme.** Alle Kern-Entities stammen jetzt aus der lokalen EMS-ESP-
Integration.
