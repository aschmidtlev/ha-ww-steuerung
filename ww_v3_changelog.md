# ww_v3 – Changelog gegenüber ww_v2

## Kern-Migration: Bosch Home Connect Cloud → EMS-ESP (lokal)

- **Ersetzt**: `water_heater.dhw1`/`current_temperature` (Bosch Cloud) durch
  `sensor.boiler_dhw_curtemp` (EMS-ESP, lokal). Der Template-Code fuer
  `sensor.ww_v3_warmwassertemperatur` ist dadurch einfacher als in v2 (kein
  Attribut-Zugriff, kein String-Zahl-Workaround mehr noetig).
- **Ersetzt**: `switch.charge` durch `switch.boiler_dhw_onetime` als
  Ladeauslöser in allen Automationen und Skripten.
- **Ersetzt**: `number.charge_setpoint` durch `number.boiler_dhw_seltempsingle`
  als Ziel des `number.set_value`-Aufrufs beim Ladestart.
- **Entfernt**: `number.charge_duration` wird nicht mehr gesetzt (siehe
  `ww_v3_migration_concept.md` Abschnitt 3.3 fuer die Begruendung) – die
  Ladedauer wird ausschliesslich durch `automation.ww_v3_stop_ziel` und
  `automation.ww_v3_sicherheitsabschaltung_maxdauer` begrenzt (beide
  unveraendert aus v2 uebernommen).
- **Ergebnis**: Es wird keine einzige Bosch-Cloud-Entity mehr referenziert
  (in v2 war `water_heater.dhw1` die einzige dokumentierte Ausnahme vom
  Bosch-Ausschluss – diese Ausnahme entfaellt jetzt vollstaendig, siehe
  `ww_v3_quality_report.md` Abschnitt 4).

## Neue Sicherheits-/Diagnosefunktionen (nur durch EMS-ESP moeglich)

- **Neu**: Ladeziele werden vor jedem Start gegen `number.boiler_dhw_maxtemp`
  (Geraete-eigene Obergrenze) gekappt; eine Kappung wird mit Warnung
  geloggt. Schliesst die in `ww_v2_open_questions.md` dokumentierte,
  ungeloeste Luecke ("`number.charge_setpoint`-Max war nicht verifizierbar").
- **Neu**: `binary_sensor.ww_v3_ladung_aktiv_hardware` zeigt die tatsaechliche
  Geraete-Ladebestaetigung (`binary_sensor.boiler_dhw_charging`/`_recharging`),
  unabhaengig vom eigenen Schalterbefehl `switch.ww_v3_ladung_aktiv`.
- **Neu**: `automation.ww_v3_debug_ladung_ohne_hardware_reaktion` warnt, wenn
  `switch.boiler_dhw_onetime` seit 10 Minuten an ist, aber keine Hardware-
  Ladebestaetigung vorliegt (moegliche Geraetestoerung).
- **Neu**: `binary_sensor.ww_v3_hygieneziel_ueber_geraetemaximum` warnt, falls
  das konfigurierte Hygieneziel ueber der Geraete-Maximaltemperatur liegt.
- **Neu**: `automation.ww_v3_pvenabledhw_sperren` haelt das native EMS-ESP-
  Feature "PV laedt Warmwasser auf" (`switch.thermostat_pvenabledhw`)
  deaktiviert, solange `input_boolean.ww_v3_aktiv` an ist, um eine
  Doppelsteuerung zu vermeiden (Anwenderentscheidung, siehe
  `ww_v3_open_questions.md`).
- **Neu**: `sensor.boiler_dhw_curtemp2` (zweiter, externer Temperaturfuehler)
  wird als Vergleichs-/Diagnosewert im Attribut `temperatur_extern` von
  `sensor.ww_v3_warmwassertemperatur` sowie auf der Debug-Seite angezeigt –
  fliesst **nicht** in Regelentscheidungen ein (siehe
  `ww_v3_open_questions.md`, Offene Frage 1).
- **Neu**: Dashboard-Bereich "EMS-ESP native Zusatzfunktionen" (Steuerung-
  Ansicht) zeigt DHW-Prioritaet, Komfortmodus, Zirkulationsmodus und
  Thermostat-Betriebsart als manuell bedienbare/beobachtbare Zusatzinfos,
  ohne sie zu automatisieren.

## Nachtrag: Hausverbrauch neu berechnet statt Rohwert (2026-09-08)

- **Ersetzt**: `sensor.gesamtleistung_haushalt` wird nicht mehr direkt im
  Dashboard oder in der Pflichtdaten-Diagnose referenziert. Grund: ein
  Plausibilitaetsvergleich zeigte, dass dieser externe Sensor bei PV=0 und
  inaktiver Waermepumpe nur ~28 % des gleichzeitigen Netzbezugs anzeigte
  (193,7 W vs. 699,9 W) – die Entity misst vermutlich nur einen
  Teil-Stromkreis, nicht das gesamte Haus.
- **Neu**: `sensor.ww_v3_hausverbrauch_berechnet` berechnet den Hausverbrauch
  stattdessen aus der Energiebilanz (PV gesamt + Netzbezug − Netzeinspeisung
  + Batterieleistung, falls Marstek verfuegbar) und ersetzt den alten Sensor
  in allen drei Dashboard-Ansichten (Uebersicht, Energie, Debug) sowie in
  der Pflichtdaten-Diagnoseliste. Der alte Rohwert bleibt als
  Vergleichsattribut erhalten. Details und dokumentierte Annahmen (u. a.
  unverifizierte Batterie-Vorzeichenkonvention) siehe
  `ww_v3_open_questions.md`, Punkt B6.

## HA-Coding-Standards (Qualitaetsverbesserung, im Rahmen der Migration)

- **Neu**: `entity_category: diagnostic` wurde fuer alle reinen
  Diagnose-/Status-Entities ergaenzt (u. a. `ww_v3_system_aktiv`,
  `ww_v3_manuelle_uebersteuerung`, `ww_v3_pflichtdaten_gueltig`,
  `ww_v3_diagnose_pflichtentities`, `ww_v3_last_*`, `ww_v3_pv_*_verfuegbar`,
  `ww_v3_marstek_verfuegbar`, `ww_v3_ladung_aktiv_hardware`,
  `ww_v3_hygieneziel_ueber_geraetemaximum`). War in v2 an keiner Stelle
  gesetzt.

## Unveraendert uebernommen (reine Namensraum-Umbenennung `ww_v2_` → `ww_v3_`)

Die gesamte Preis-/PV-/Batteriespeicher-/Anti-Takt-/Mindestlaufzeit-/Lern-/
Kosten-/Debug-Pushover-Logik aus v2 wurde **inhaltlich unveraendert**
uebernommen, nur der Entity-Namensraum wurde umbenannt. Siehe
`ww_v2_changelog.md` fuer die vollstaendige Historie dieser Funktionen
gegenueber v1_8.

## Rollback

`ww_v2_package.yaml`/`ww_v2_dashboard.yaml` bleiben unveraendert im
Verzeichnis liegen und koennen jederzeit wieder aktiviert werden (siehe
`ww_v3_installation.md` Abschnitt 8). Ein produktiver **Parallelbetrieb**
von ww_v2 und ww_v3 wird jedoch **nicht** unterstuetzt, sobald sowohl die
Bosch-Home-Connect-Integration als auch EMS-ESP gleichzeitig aktiv sind,
da sonst zwei unabhaengige Systeme denselben physischen Warmwasserspeicher
ueber unterschiedliche Schalter (`switch.charge` vs.
`switch.boiler_dhw_onetime`) steuern koennten.
