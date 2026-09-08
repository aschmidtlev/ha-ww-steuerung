# ww_v2 – Analyse der bestehenden Warmwassersteuerung v1_8

Stand: 2026-08-31. Analysierte Quellen (Ordner `Copilot/`, da kein Ordner `ha` existiert):

- `Copilot/warmwasser_v1_8_full_package (2).yaml` (1056 Zeilen)
- `Copilot/warmwasser_v1_8_full_dashboard (2).yaml` (275 Zeilen)
- `Copilot/harvest_2026-08-29-22-35-33.csv` (2081 Zeilen, Entity-Snapshot vom 30.08.2026)

Keine dieser Dateien wurde verändert.

## 1. Wichtigster Befund: v1_8 ist im Live-System aktuell nicht funktionsfähig

Alle Entities mit Präfix `ww_v1_8_` / `ww_v18_` (Package-eigene `sensor`, `binary_sensor`,
`input_*`, `counter`, `script`, `automation`) tragen in der CSV exakt denselben
`Last updated`-Zeitstempel **30/08/2026, 00:05:53** und den State **`unavailable`**.
Andere, unabhängige Integrationen (z. B. `sensor.evu_leistung`, `sensor.wp_gesamtleistung`)
aktualisieren sich dagegen bis 00:35 Uhr weiter.

**Interpretation:** Um 00:05:53 Uhr gab es einen Reload/Neustart, nach dem das gesamte
Package v1_8 nicht mehr geladen wurde (YAML-Fehler, entfernte Include-Datei oder
deaktiviertes Package). Seitdem liefert v1_8 keine Steuerungsentscheidung mehr, keine
Ladung, keine Kostenerfassung. Das ist unabhängig von der inhaltlichen Qualität des
Packages ein Betriebsausfall.

**Mögliche Ursache (aus dem Code ableitbar, nicht bewiesen):** Das Template
`sensor.ww_v1_8_warmwassertemperatur` referenziert `sensor.dhw1_temperature` als
dritten Fallback. Diese Entity existiert in der aktuellen CSV **nicht** (nur
`sensor.dhw1_water_flow` und `sensor.dhw1_working_time` sind vorhanden). Das allein
verursacht bei Home Assistant keinen Ladefehler (Templates werfen bei fehlenden
Entities i. d. R. `unknown`, keinen Package-Absturz) – die genaue Ursache des
Ladefehlers ist aus den vorliegenden Dateien nicht abschließend feststellbar und wird
als offene Frage geführt.

**Konsequenz für ww_v2:** ww_v2 wird komplett neu und unabhängig aufgebaut. Vor der
Aktivierung muss der Anwender die neuen Dateien separat auf YAML-Gültigkeit prüfen
(`ha core check_config`), da der aktuelle Live-Zustand keine Rückschlüsse auf
funktionierende Syntax zulässt.

## 2. Reste einer noch älteren Version (vor v1_8)

In der CSV existieren zusätzlich, unabhängig vom gelieferten Package:

- `input_boolean.ww_debug` ("WW Debug (Pushover)") – State `off`, **funktionsfähig**
- `input_boolean.ww_debug_mode` ("WW Debug-Modus") – `unavailable`
- `input_button.ww_debug_test` ("WW Debug-Nachricht testen") – `unavailable`
- `script.ww_debug_notify` ("WW Debug - Pushover senden") – `unavailable`

Diese Namen (`ww_debug*`, ohne Versionsnummer) kommen in keiner der beiden
gelieferten Dateien vor. Es handelte sich vermutlich um eine noch ältere
Debug-Implementierung. Sie werden in ww_v2 nicht wiederverwendet, aber im
Entity-Katalog als „gefunden, nicht referenziert von v1_8, Herkunft unklar“ geführt.

## 3. YAML- und Strukturprüfung (Abschnitt 9.1)

- Beide Dateien sind syntaktisch sauber lesbar, konsistente Einrückung (2 Leerzeichen),
  keine offensichtlichen doppelten Schlüssel.
- Alle `unique_id`s im Package sind auf den ersten Blick eindeutig und folgen dem
  Schema `ww_v18_*`.
- Package enthält keine `!include`-Direktiven – alles ist in einer Datei, daher keine
  fehlenden Include-Dateien zu prüfen.
- Dashboard nutzt ausschließlich native Lovelace-Karten (`entities`, `gauge`,
  `history-graph`, `logbook`, `conditional`, `markdown`) – keine HACS-Abhängigkeit.
- `input_text.ww_v18_wp_energy_entity` hat `initial: sensor.ww_v1_8_wp_energie_gesamt`
  – das ist eine **Selbstreferenz-Falle**: Der WP-Energiezähler, den das Package selbst
  aus `sensor.wp_gesamtleistung` per `integration`-Plattform bildet, wird über einen
  Helfer wieder referenziert. Funktioniert technisch, ist aber eine unnötige
  Indirektion ohne erkennbaren Nutzen (der Wert steht bereits fest im Code).

## 4. Entities: Risiken und Lücken (Abschnitt 9.2)

| Befund | Bewertung |
|---|---|
| `sensor.dhw1_temperature` (3. Fallback für Warmwassertemperatur) existiert nicht in der CSV | Fallback ist tot; verschleiert im Fehlerfall echten Datenausfall |
| `sensor.hotwater_temp` (2. Fallback) hat Friendly Name „Bosch sensors“, State `unknown` | Es handelt sich fachlich um eine **Bosch-Entity** (siehe Abschnitt 6) – laut Vorgabe in ww_v2 nicht verwendbar |
| `sensor.home_leading_price_average` (Fallback für Leading24) existiert nicht in CSV | Toter Fallback, wie oben |
| `sensor.home_nachster_bestpreis_zeitraum_start` / `sensor.home_next_best_price_period_start` (Fallbacks) existieren nicht in CSV | Toter Fallback |
| `binary_sensor.home_bestpreis_zeitraum`-Attribute (`next_start` etc.) nicht verifizierbar (CSV erfasst keine Attribute) | Die gesamte Attribut-Rateschleife in `ww_v18_naechster_bestpreis_start` ist fragil |
| Es existieren bereits fertige, direkte Sensoren `sensor.home_bestpreis_startet`, `sensor.home_bestpreis_startet_in`, `sensor.home_bestpreis_endet`, `sensor.home_bestpreis_verbleibend` | Deutlich robusterer Ersatz für die Attribut-Ratelogik – wird für v2 empfohlen |
| `input_boolean.ww_v18_marstek_nutzen` steuert `binary_sensor.ww_v1_8_marstek_verfuegbar`, aber Marstek-Sensor (`sensor.marstek_venus_modbus_soc_batterie`) ist aktuell **`unavailable`** | Marstek-Anbindung ist derzeit ausgefallen (Modbus-Verbindung `binary_sensor.marstek_venus_modbus_modbus_verbindung` = `off`) – Fallback-Verhalten bei Marstek-Ausfall im Code vorhanden (State-Check `not in unknown/unavailable`), das ist korrekt |
| `number.charge_setpoint` / `number.charge_duration` – `min`/`max`-Attribute nicht in CSV erfasst | `ww_v18_hygiene_pruefung` prüft `state_attr('number.charge_setpoint','max')`, kann nicht verifiziert werden, ob das Attribut existiert bzw. ≥ 60 °C zulässt |
| CSV erfasst für `water_heater.dhw1` keinerlei Attribute (auch `current_temperature` nicht) | Zentrale Annahme von v1_8 („Ist-Temperatur steht in `current_temperature`“) ist aus der CSV **nicht verifizierbar** – siehe offene Frage 1 |

## 5. Bosch-Referenzen (vollständige Suche, Abschnitt 4)

### 5.1 Direkt im Package/Dashboard (`warmwasser_v1_8_full_*.yaml`)

Keine Entity-ID mit `bosch` im Namen wird in den beiden gelieferten v1_8-Dateien
referenziert. **Aber:** `sensor.hotwater_temp` (Fallback #2 für die Warmwassertemperatur,
Zeile 411/419 im Package) ist laut CSV ein Bosch-Sensor (Friendly Name „Bosch sensors“).
Das ist eine **indirekte Bosch-Abhängigkeit**, die im Code nicht als solche erkennbar ist.

| Datei | Zeile | Entity | Funktion | Abhängigkeit | Ersatz in v2 | Status |
|---|---|---|---|---|---|---|
| package.yaml | 407, 411, 419 | `sensor.hotwater_temp` | 2. Fallback Warmwassertemperatur | indirekt (Bosch-Integration) | entfällt ersatzlos, nur `water_heater.dhw1` bleibt Quelle | entfernt |
| package.yaml | 412, 415, 420 | `sensor.dhw1_temperature` | 3. Fallback Warmwassertemperatur | Entity existiert nicht (siehe oben), Herkunft unklar | entfällt ersatzlos | entfernt |

### 5.2 Im Gesamtsystem (CSV), die potenziell für v2 relevant wären, aber ausgeschlossen werden

| Entity | Domain | Friendly Name | Grund für Bosch-Klassifizierung |
|---|---|---|---|
| `climate.hc1` | climate | Heating circuit hc1 hc1 | Bosch-Heizkreis (Home Connect / Bosch-Integration) |
| `sensor.hotwater_temp` | sensor | Bosch sensors | Friendly-Name-Gruppe „Bosch sensors“ |
| `sensor.actual_supply_temp`, `sensor.actual_supply_temperature`, `sensor.outdoor_temperature`, `sensor.return_temp`, `sensor.switch_temp`, `sensor.pool_temperature`, `sensor.energy_consumption`, `sensor.actual_power`, `sensor.actual_modulation`, `sensor.actual_heating_pump_modulation`, `sensor.chimney_temp`, `sensor.burner_power_setpoint`, `sensor.system_pressure`, `sensor.numberofstarts`, `sensor.health_status`, `sensor.notifications`, `sensor.start_time`, `sensor.supply_temp_setpoint`, `sensor.total_system_uptime` | sensor | „Bosch sensors“ | gleiche Integration/Gerätegruppe |
| `binary_sensor.chimneysweeper`, `binary_sensor.flamestatus` | binary_sensor | „Bosch sensors …“ | gleiche Integration |
| `select.holiday_dhw_mode_hm1…5`, `select.holiday_hc_mode_hm1…5`, `select.holiday_mode_assigned_to_hm1…5` | select | „Bosch selects …“ | Bosch-Urlaubsmodus – **einzige im System vorhandene Urlaubsmodus-Funktion ist Bosch-gebunden** (siehe offene Frage 9) |
| `sensor.ww_bosch_betriebsmodus`, `sensor.ww_bosch_status` | sensor | explizit „Bosch“ im Namen, State `unavailable` | bereits verwaister Rest einer früheren Bosch-Anbindungsidee |
| `update.bosch_thermostat_update` | update | Bosch thermostat Update | Bosch-Integration |

**Ergebnis:** `water_heater.dhw1` und `switch.charge` selbst tragen keinen „Bosch“-Namen
und werden vom Anwender in Abschnitt 3 ausdrücklich bestätigt – sie werden daher
weiter verwendet. Alle anderen oben gelisteten Bosch-Entities fließen **nicht** in
ww_v2 ein. Nach Erstellung der neuen Dateien erfolgt ein erneuter Volltext-Scan auf
„bosch“ (Groß-/Kleinschreibung ignorierend) über alle `ww_v2_*`-Dateien; das Ergebnis
wird im Qualitätsbericht (`ww_v2_quality_report.md`) dokumentiert.

## 6. Templates (Abschnitt 9.3)

- Temperaturfilter verwenden überwiegend `| float(99)` bzw. `| float(0)` als Default.
  Das ist gefährlich: `float(99)` bei „Kritischer Bedarf“ verhindert zwar einen
  Fehlalarm (99 °C < Minimum ist falsch), aber `float(0)` in `ww_v18_stop_ziel`
  (`states(...) | float(0) >= ziel`) hätte bei ungültiger Temperatur **keine** Aktion
  zur Folge (0 ≥ 51 ist falsch) – hier zufällig sicher, aber nicht durch Konzept,
  sondern durch Zufall der Vergleichsrichtung.
- Keine der Templates verwendet ein echtes `availability:`-Template konsequent für
  *alle* abhängigen Sensoren (z. B. `ww_v1_8_ladebedarf`, `ww_v1_8_dynamisches_ziel`
  haben keine `availability`), obwohl sie von potenziell ungültigen Werten abhängen.
- `now()` wird mehrfach in `state`-Templates verwendet (`ww_v1_8_verbrauchsprognose`,
  `ww_v1_8_antitakt_frei`, `ww_v1_8_mindestlaufzeit_erreicht`) ohne Zeitzonenproblem
  (HA löst `now()` immer in der konfigurierten Zeitzone auf) – unkritisch, aber
  `now()`-Templates aktualisieren sich nicht automatisch bei reiner Zeitänderung ohne
  weiteren Trigger; die Antitakt-/Mindestlaufzeit-Sensoren werden daher nur bei
  Abfrage (z. B. durch die `/5`-Minuten-Automation) neu berechnet – das ist so vom
  Design her beabsichtigt und in Ordnung, aber nicht dokumentiert.
- Division durch Null: keine gefunden (keine Templates dividieren durch eine
  Sensor-abhängige Variable ohne Konstante).
- Kein Template begrenzt die Textlänge von `input_text`-Werten aktiv (max ist über
  `input_text`-Konfiguration selbst begrenzt, z. B. 120 Zeichen für
  `ww_v18_letzter_grund`) – ausreichend.

## 7. Automationen (Abschnitt 9.4)

- Alle Automationen laufen mit `mode: single` (außer den beiden Debug-Automationen:
  `mode: queued`, und den Lern-Snapshot-Automationen ohne explizites `mode`, Default
  `single`). Für die kritischen Lade-/Stop-Automationen ist `single` richtig, um
  Doppelausführungen zu verhindern.
- `ww_v18_wirtschaftliche_ladung` und `ww_v18_p0_kritisch` prüfen beide
  `switch.charge: state: off` als Bedingung – das verhindert Mehrfachauslösung von
  `switch.charge`, solange sich der Schalter-State zuverlässig sofort auf `on`
  aktualisiert. Bei einer Race Condition zwischen zwei gleichzeitig auslösenden
  Automationen (`time_pattern: /5` UND ein State-Trigger im selben Moment) ist
  `mode: single` zusammen mit HA's synchroner Bedingungsauswertung ausreichend, aber
  nicht hundertprozentig race-frei, da HA Automationen mit unterschiedlicher
  `automation_id` unabhängig voneinander in Parallelität starten kann. **Für v2 wird
  ein zentrales „Charge-Lock“-Helferkonzept vorgeschlagen** (siehe Funktionsmatrix).
- `ww_v18_hygiene_pruefung` nutzt `wait_template` mit `timeout: 02:30:00` und
  `continue_on_timeout: true` – bei einem HA-Neustart während dieses Wartens geht der
  Skriptzustand verloren; `switch.charge` bliebe dann ggf. dauerhaft `on`, ohne dass
  eine der übrigen Automationen (`ww_v18_stop_ziel` würde greifen, sobald Zieltemperatur
  erreicht ist – das fängt den Fall ab, sofern die Zieltemperatur überhaupt erreicht
  wird). Restrisiko: Wird das Ziel wegen eines Sensorausfalls nie erreicht, bleibt die
  Ladung nach einem Neustart unbegrenzt aktiv. Das wird in v2 durch eine
  Not-Aus-/Maximaldauer-Bedingung auf Basis eines Zeitstempels geschlossen.
- Kein `mode: restart` wird verwendet, wo es unter Umständen sinnvoll wäre (z. B. bei
  `ww_v18_stop_ziel`, dessen Trigger-Template mehrfach parallel auslösen könnte); da
  aber `single` bereits Doppelausführung verhindert (neue Ausführung wird verworfen,
  nicht verzögert), ist das für einen Stop-Befehl unkritisch.
- Anti-Takt-Sperre (`ww_v18_antitakt_sperre`, 20 min) und Mindestlaufzeit
  (`ww_v18_mindestlaufzeit`, 30 min) sind vorhanden und werden korrekt in den
  Bedingungen der Start- bzw. Stop-Automation geprüft. Diese Werte werden für v2
  übernommen (siehe Abschnitt 26 der Aufgabenstellung).
- Priorität zwischen `ww_v18_p0_kritisch` und `ww_v18_wirtschaftliche_ladung` ist
  implizit über die Bedingung `binary_sensor.ww_v1_8_kritischer_bedarf: off`
  **nicht** in der wirtschaftlichen Automation abgesichert – d. h. beide Automationen
  könnten theoretisch im selben Fünf-Minuten-Takt feuern, wenn kritischer Bedarf UND
  Ladebedarf gleichzeitig wahr sind. Da aber `switch.charge: off` in beiden eine
  Bedingung ist und die zuerst ausgeführte Automation `switch.charge` einschaltet,
  scheitert die zweite an der `off`-Bedingung. Funktioniert, ist aber implizit statt
  explizit priorisiert. v2 macht die Priorität explizit (siehe
  `ww_v2_function_matrix.md`, Prioritätenkette).

## 8. Dashboard (Abschnitt 9.5)

- Nur native Karten, keine HACS-Abhängigkeit – positiv.
- Bedienung und Statusanzeige sind nur teilweise getrennt: Der Bereich „Bedienung“
  enthält korrekt nur `input_boolean`-Schalter. Reine Status-Informationen
  (z. B. „Aktive Strategie“, „Ampel“) werden aber als normale `entities`-Zeilen ohne
  Farbcodierung dargestellt, nicht als klar erkennbare Status-Badges.
- Ampel-Logik (`sensor.ww_v1_8_ampel`) unterscheidet nur rot/orange/grün als reinen
  Text-State, keine visuelle Gauge/Badge-Farbgebung im Dashboard selbst (die
  `conditional`-Karten mit Markdown sind eine funktionierende, aber technisch
  unelegante Lösung).
- Keine mobile-spezifische Anpassung (kein `type: sections` mit Spaltensteuerung);
  bei `masonry` sollte das auf dem Smartphone dennoch funktionieren (automatisches
  Stapeln), wurde aber nicht live getestet, da keine laufende HA-Instanz zur
  Verfügung steht.
- Zwei `history-graph`-Karten mit `hours_to_show: 24` ohne weitere Performance-Regler
  – bei Recorder-Last unkritisch für zwei Karten.

## 9. Zusammenfassung wichtigster Handlungsfelder für ww_v2

1. Zentrale Ist-Temperatur nur noch aus `water_heater.dhw1` (Attribut, sobald
   bestätigt) ableiten – keine Bosch-Fallbacks, keine toten Fallback-Entities.
2. Bestpreis-Zeitfenster über die bereits vorhandenen dedizierten Sensoren
   (`sensor.home_bestpreis_startet`, `_endet`, `_startet_in`, `_verbleibend`) statt
   fragiler Attribut-Ratelogik lesen.
3. Zwei getrennte Forecast.Solar-Instanzen (`sensor.energy_*` / `sensor.energy_*_2`)
   sauber Ost/West zuordnen (Rückfrage) statt sie blind zu addieren.
4. Charge-Sperren (Anti-Takt, Mindestlaufzeit, „läuft bereits“) beibehalten und um
   ein explizites Lock-Konzept ergänzen.
5. Debug/Pushover zentralisieren (`script.ww_v2_debug_log`), Deduplizierung und
   Fehlertrennung von der Regelungslogik einbauen.
6. Betriebszustand nach Neustart robust machen (keine verwaisten `wait_template`
   ohne Zeit-Obergrenze mit Fallback).
