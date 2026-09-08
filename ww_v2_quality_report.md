# ww_v2 – Qualitätsbericht

**Wichtiger Hinweis zur Prüftiefe:** In dieser Umgebung steht keine laufende
Home-Assistant-Instanz zur Verfügung. `ha core check_config` sowie ein
echtes Auslösen von Automationen/Templates in Developer Tools konnten daher
**nicht** durchgeführt werden. Alle Prüfungen unten sind **statische
Code-/Text-Analysen** (grep-Suchen, manuelle Zeile-für-Zeile-Durchsicht,
Zählung von Jinja-Klammerpaaren). Vor Produktivbetrieb ist Abschnitt 1 der
`ww_v2_installation.md` („Konfiguration überprüfen“) zwingend zusätzlich in
der echten HA-Instanz durchzuführen.

## 1. Checkliste gemäß Aufgabenstellung Abschnitt 27

| Prüfpunkt | Methode | Ergebnis |
|---|---|---|
| YAML ist parsebar | Keine YAML-Bibliothek/Interpreter in dieser Umgebung verfügbar (kein Python, kein Node, `Install-Module powershell-yaml` durch fehlenden interaktiven Modus nicht möglich); ersatzweise manuelle Einrückungsprüfung + Tab-Suche | **Keine Tabs gefunden**, Einrückung stichprobenartig und an allen komplexen Stellen (verschachtelte `if/then`, mehrzeilige Templates) manuell verifiziert. **Nicht mit einem echten Parser bestätigt** – vor Aktivierung zwingend „Konfiguration überprüfen“ in HA ausführen |
| Keine doppelten Schlüssel | grep auf `unique_id:` und Automations-`id:` | Keine Duplikate gefunden (`grep -o "unique_id: ..." \| sort \| uniq -d` → leer; gleiches für Automations-IDs) |
| Korrekte Einrückungen | manuelle Durchsicht aller Mehrzeilenblöcke (`>-`-Templates, `if/then/else`) | Konsistent 2-Leerzeichen-Schema wie in v1_8, keine Auffälligkeiten gefunden |
| Keine ungültigen Schlüssel | Abgleich aller verwendeten HA-Schlüssel (`availability`, `state_class`, `device_class`, `wait_template`, `continue_on_timeout`, `stop`, `continue_on_error`) gegen dokumentierte Home-Assistant-Standardfunktionen | Alle verwendeten Schlüssel sind dokumentierte HA-Kernfunktionen, keine erfundenen Optionen |
| Keine Bosch-Referenzen | `grep -rni bosch` über beide neuen Dateien, zusätzlich gezielte Suche nach allen 20 einzeln identifizierten Bosch-Entity-IDs aus `ww_v2_analysis.md` | **Bestätigt sauber**: „bosch“ kommt nur in 4 erklärenden Kommentarzeilen vor („Keine Bosch-Entity wird referenziert“ u. ä.), keine tatsächliche Entity-Referenz. Keine der 20 identifizierten Bosch-Entity-IDs wird referenziert |
| Alle Entities vorhanden oder ausdrücklich bestätigt | Abgleich jeder in `ww_v2_package.yaml`/`ww_v2_dashboard.yaml` referenzierten externen Entity gegen CSV bzw. Nutzerantworten | Alle externen Referenzen stammen aus der CSV oder wurden vom Nutzer bestätigt (siehe `ww_v2_entity_mapping.md`). Einzige nicht per CSV-Attribut verifizierbare Annahme (`number.charge_setpoint`-Grenzen) wurde bewusst **entfernt**, nicht verwendet |
| Alle Services vorhanden oder bestätigt | Liste aller `service:`-Aufrufe geprüft | `switch.turn_on/off`, `number.set_value`, `input_*.set_value/turn_on/turn_off`, `counter.increment/reset`, `notify.pushover`, `logbook.log`, `script.*` – ausschliesslich dokumentierte HA-Kernservices bzw. der im v1_8-Code produktiv referenzierte Pushover-Service |
| Alle Dashboard-Entities definiert | jede im Dashboard referenzierte `ww_v2_*`-Entity gegen `ww_v2_package.yaml` abgeglichen | Vollständig abgeglichen; keine „hängenden“ Referenzen wie beim v1_8-Dashboard-Fehler (dort zwei nicht definierte `input_text`-Helfer) |
| Keine nicht definierten Helfer | s.o. | bestätigt |
| Keine doppelten Unique IDs | s.o. (Tabelle oben) | bestätigt |
| Templates gegen `unknown` abgesichert | jedes `state:`-Template mit externer Entity-Abhängigkeit auf `availability:` oder expliziten `in ['unknown','unavailable']`-Check geprüft | Kern-Templates (Warmwassertemperatur, Kritischer Bedarf, Komfortbedarf, Ladebedarf, PV-Ueberschuss, PV-Leistung/-Forecast Ost/West, Netzbezug/-einspeisung, Diagnose) haben explizite `availability`-Gates oder Verfügbarkeits-Flags |
| Templates gegen `unavailable` abgesichert | s.o. | bestätigt, gleiche Prüfung deckt beide Zustände ab |
| Sichere mathematische Berechnungen | Divisionsstellen gesucht (`/`) | Divisionen: `/ 60` (Minutenumrechnung, Konstante), `/ 100` (ct→EUR, Konstante), `/ 4` (Test 13-Schwelle, Konstante) – **keine Division durch eine variable, potenziell nullwertige Grösse** |
| Keine Division durch null | s.o. | bestätigt |
| Forecast-Einheiten kompatibel | Prüfung in `ww_v2_analysis.md`/`ww_v2_entity_catalog.md` | Beide Forecast.Solar-Reihen liefern kWh (Energie) bzw. W (Leistung) in identischem Format, gleicher Zeitstempel – Addition ist fachlich zulässig |
| Forecast-Zeiträume kompatibel | s.o. | bestätigt (heute/morgen/nächste Stunde je Anlage decken denselben Zeitraum ab) |
| Korrekte Zeitzonen | `now()`/`as_datetime()`-Verwendung geprüft | Home Assistant löst `now()` grundsätzlich in der konfigurierten Systemzeitzone auf; keine manuelle Zeitzonen-Umrechnung im Code, daher kein Fehlerpotenzial durch abweichende Interpretation |
| Pushover-Ausfall blockiert keine Steuerung | Codepfad-Analyse `script.ww_v2_debug_log`, `script.ww_v2_pushover_test` | `continue_on_error: true` auf dem `notify.pushover`-Aufruf; kein `service: notify.pushover`-Aufruf befindet sich im kritischen Pfad einer Lade-Automation selbst |
| Debug aus sendet keine Debug-Nachrichten | Bedingungsprüfung in `script.ww_v2_debug_log` und den drei Debug-Automationen | Pushover-Versand ist strikt an `input_boolean.ww_v2_debug_mode: on` gebunden; die drei Debug-Automationen (`ww_v2_debug_blockade`, `_strategiewechsel`, `_sensorfehler`) haben zusätzlich `debug_mode: on` als Automations-Bedingung |
| Debug an protokolliert Aktionen | Aufrufliste von `script.ww_v2_debug_log` je Aktionspunkt | Aufgerufen bei: Ladung Start/Abbruch/Übersprungen, Ladung Stop, Sicherheitsabschaltung, Hygienezyklus Start/Erfolg/Abbruch, Kostenabschluss, Blockade (Anti-Takt), Strategiewechsel, Sensorausfall |
| Keine Meldung bei unveränderten Prüfläufen | Dedup-Logik in `script.ww_v2_debug_log` | Signatur-Vergleich + Mindestabstand verhindert wiederholte identische Pushover-Nachrichten (Testszenarien 32/33) |
| Charge wird nicht mehrfach unnötig ausgelöst | `script.ww_v2_ladung_starten` Frühzeitiger `stop`, wenn `switch.charge` bereits `on` | bestätigt, zusätzlich `mode: single` auf Skript und allen Lade-Automationen |
| Neustartverhalten ist sicher | Analyse der `wait_template`/wiederkehrenden `time_pattern`-Trigger | `ww_v2_sicherheitsabschaltung_maxdauer` (neue Funktion) fängt den einzigen identifizierten Restrisiko-Fall (Neustart während Hygiene-`wait_template`) ab |
| Automationsmodi sind passend | Durchsicht aller `mode:`-Angaben | ladeauslösende/-stoppende Automationen: `single` (verhindert Doppelausführung); Debug-Automationen: `queued` (keine Meldung geht verloren, auch bei schneller Abfolge) |
| Keine offensichtlichen Race Conditions | Prüfung der Prioritätenkette P0 vs. wirtschaftliche Ladung | Beide Automationen prüfen `switch.charge: off` als Bedingung; zusätzlich verhindert `script.ww_v2_ladung_starten` durch den `bereits_an`-Check am Skriptanfang eine doppelte Ausführung, selbst wenn beide Automationen im selben Tick auslösen sollten |
| Keine Endlosschleifen | Prüfung aller `wait_template`/Trigger-Ketten auf Selbstauslösung | `ww_v2_hygiene_pruefung` hat `timeout` + `continue_on_timeout`; keine Automation triggert direkt oder indirekt sich selbst erneut ohne Zustandsänderung |
| Vertretbare Recorder-Belastung | Update-Frequenz der Template-Sensoren geprüft | Die meisten Sensoren aktualisieren sich nur bei tatsächlicher Zustandsänderung ihrer Quell-Entities (event-getrieben); zusätzlich `time_pattern: /5`-Automationen für Heartbeat/Sicherheitscheck (12×/h, unkritisch). Empfehlung zum Ausschluss der `input_text.ww_v2_status_*`-Helfer aus dem Recorder in `ww_v2_installation.md` dokumentiert |
| Native Dashboard-Version funktioniert ohne nicht bestätigte HACS-Karten | Kartentypen-Liste geprüft | Ausschliesslich `entities`, `gauge`, `history-graph`, `logbook`, `conditional`, `markdown` – alles native HA-Kernkarten, keine HACS-Abhängigkeit im gesamten Dashboard |

## 2. Design-Entscheidung: Bedienung vs. Status bei `switch.charge`

`switch.charge` wird im Dashboard **nirgends als bedienbarer Schalter**
angezeigt (weder Übersicht noch Steuerung noch Debug), da der Nutzer keine
manuelle Bedienung dieses Schalters ausdrücklich angefordert hat (Abschnitt 22
der Aufgabenstellung erlaubt „Charge nur dann, wenn eine manuelle Bedienung
ausdrücklich gewünscht und abgesichert ist“). Stattdessen zeigt
`binary_sensor.ww_v2_ladung_aktiv` (schreibgeschützt) den Ladezustand an. Der
rohe `switch.charge`-Zustand wird zusätzlich auf der Debug-Seite über die
`entities`-Karte angezeigt – da native `entities`-Karten Switch-Entities
grundsätzlich als Kippschalter darstellen, ist dies technisch weiterhin
bedienbar. Falls eine strikt read-only-Darstellung gewünscht wird, empfiehlt
sich, diese eine Zeile durch `binary_sensor.ww_v2_ladung_aktiv` zu ersetzen
(siehe `ww_v2_open_questions.md`).

## 3. Dokumentierte technische Einschränkungen (keine Fehler, sondern Grenzen der Plattform)

1. **Keine echte Pushover-Zustellbestätigung**: Native `notify`-Services
   liefern der aufrufenden Automation/dem Skript in der Regel keinen
   auswertbaren Erfolgs-/Fehler-Rückgabewert. `sensor.ww_v2_last_pushover_status`
   zeigt daher „gesendet“ (Versand angestossen), nicht „zugestellt“. Ein
   echtes Zustellungs-Tracking wäre nur mit einer nicht bestätigten
   zusätzlichen Integration möglich und wurde daher nicht erfunden.
2. **Kein Erfolgs-Log, wenn `switch.turn_on` selbst fehlschlägt** (Testszenario
   25): Schlägt der Geräte-Service mitten im `ww_v2_ladung_starten`-Skript
   fehl, bricht HA die Skriptausführung ab, bevor der abschliessende
   Erfolgs-Log-Aufruf erreicht wird. Ein Abfangen wäre nur mit einem
   `continue_on_error`+Ergebnisprüfung auf `switch.turn_on` selbst möglich;
   dies wurde bewusst nicht ergänzt, da ein fehlgeschlagener Ladebefehl ein
   Gerätefehler ist, der ohnehin im Home-Assistant-Systemprotokoll sichtbar
   wird, und ein zusätzlicher Automatismus hier über die Aufgabenstellung
   hinausgehen würde. Dokumentiert als bekannte Lücke.
3. **„Veraltete Temperatur“ ist nur Anzeige, kein Sperrkriterium** (Testszenario 7):
   `sensor.ww_v2_warmwassertemperatur` markiert einen Wert als `veraltet: true`,
   wenn `water_heater.dhw1` seit über 30 Minuten nicht aktualisiert wurde,
   nutzt dies aber aktuell nicht, um Ladeaktionen zu blockieren (der Wert
   selbst ist ja weiterhin numerisch vorhanden). Dies wurde nicht automatisch
   ergänzt, da unklar ist, ob 30 Minuten Update-Stille bei `water_heater.dhw1`
   im Normalbetrieb überhaupt ungewöhnlich sind oder ob die Entity generell
   selten aktualisiert wird – siehe `ww_v2_open_questions.md`.

## 4. Erneute Bosch-Volltextsuche (Abschnitt 4 der Aufgabenstellung)

```
grep -rni "bosch" ww_v2_package.yaml ww_v2_dashboard.yaml
→ nur 4 Treffer, alle in erläuternden Kommentarzeilen, keine Entity-Referenz
grep -niE "hotwater_temp|dhw1_temperature|climate.hc1|holiday_dhw|holiday_hc|
           holiday_mode|ww_bosch|actual_supply_temp|outdoor_temperature|
           return_temp|chimney|pool_temperature|burner_power|system_pressure|
           numberofstarts|health_status|notifications|start_time|
           supply_temp_setpoint|total_system_uptime|chimneysweeper|
           flamestatus|bosch_thermostat" ww_v2_package.yaml ww_v2_dashboard.yaml
→ keine Treffer
```

**Ergebnis bestätigt: Die neuen ww_v2-Dateien enthalten keine Bosch-Referenz.**
