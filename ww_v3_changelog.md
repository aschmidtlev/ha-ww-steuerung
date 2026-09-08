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

## Nachtrag: drei weitere EMS-ESP-Funktionen integriert (2026-09-08)

- **Neu**: `automation.ww_v3_debug_fehlercode_erkannt` alarmiert (severity
  critical, unabhaengig vom Debug-Modus lokal protokolliert) bei jeder
  Aenderung von `sensor.boiler_lastcode`/`sensor.thermostat_lastcode`.
  `sensor.boiler_servicecode`/`_servicecodenumber` werden bewusst nur zur
  Anzeige aufgenommen, nicht automatisch interpretiert (Bedeutung der
  Codes nicht verifiziert, siehe `ww_v3_open_questions.md` Punkt B7).
- **Neu**: `sensor.ww_v3_dhw_cop_lebenszeit` und `sensor.ww_v3_dhw_
  zusatzheizung_anteil` berechnen erstmals eine echte Effizienzkennzahl
  (Lebenszeit-Durchschnitt) aus den EMS-ESP-Energiezaehlern, bewusst nur
  fuer den DHW-Zweig (nicht ueber das mehrdeutig benannte `sensor.boiler_
  nrgsupptotal`, siehe Offene Frage B7). Beide neu im Bereich "Effizienz"
  der Energie-und-PV-Ansicht.
- **Neu**: Heizkreis hc1 (`climate.thermostat_hc1` und vier zugehoerige
  Sensoren) ist jetzt rein informativ im Dashboard sichtbar (eigene Karte
  "Heizkreis hc1"), wird aber weiterhin **nicht** von ww_v3 gesteuert oder
  in Ladeentscheidungen einbezogen – bleibt ausserhalb des Funktionsumfangs
  "Warmwassersteuerung".

## Nachtrag: Zeitzonen-Bug in Zeitdifferenz-Berechnungen behoben (2026-09-09)

Live-Beobachtung des Anwenders: `binary_sensor.ww_v3_antitakt_frei` und
`binary_sensor.ww_v3_mindestlaufzeit_erreicht` zeigten nach dem ersten
Ladezyklus dauerhaft "Nicht verfügbar" statt ein/aus. Ursache (aus dem
Muster erschlossen, nicht live verifiziert): beide Templates berechneten
Zeitdifferenzen mit `(now() - as_datetime(zeitstempel)).total_seconds()`.
`now()` ist zeitzonen-bewusst, `as_datetime()` liefert auf manchen
HA-Versionen ein zeitzonen-naives Objekt zurueck – die Subtraktion wirft
dann eine Ausnahme, die HA als `unavailable` anzeigt. Betraf zusaetzlich
still (ohne UI-Anzeige als "unavailable", da es sich um eine Automations-
*Bedingung* statt eine Entity handelt) die Bedingung von
`automation.ww_v3_sicherheitsabschaltung_maxdauer` – dort haette die
Sicherheitsabschaltung im Zweifel gar nicht ausgeloest.

**Fix**: alle drei Stellen verwenden jetzt `as_timestamp(now()) -
as_timestamp(zeitstempel)` (reiner Unix-Zeitstempel-Vergleich, keine
Datetime-Objekt-Subtraktion) – identisches Muster wie bereits im
Pushover-Dedup in `script.ww_v3_debug_log` erfolgreich im Einsatz.

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
