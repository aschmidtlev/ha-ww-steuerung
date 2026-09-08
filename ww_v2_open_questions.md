# ww_v2 – Offene Fragen: Status

## Teil A – Ursprüngliche Rückfragen (beantwortet)

Diese Fragen wurden in der Analysephase gestellt und vom Nutzer beantwortet;
die Antworten sind bereits vollständig in `ww_v2_package.yaml`,
`ww_v2_entity_mapping.md` und `ww_v2_function_matrix.md` umgesetzt.

| # | Frage (Kurzform) | Antwort des Nutzers | Umsetzung |
|---|---|---|---|
| 1 | Ist-Temperatur-Attribut `water_heater.dhw1` | `current_temperature: 52.2`, `min_temp: 0`, `max_temp: 100`, `operation_list: off, on, eco, performance, high_demand`, zusätzlich Attribute `bosch_state`, `switchPoint`, `supported_features: 2` per Screenshot bestätigt | `sensor.ww_v2_warmwassertemperatur` nutzt genau dieses Attribut |
| 2 | Energiezähler Wärmepumpe | Option a: `sensor.wp_gesamtleistung` + eigene Riemann-Integration (unverändert wie v1_8) | `sensor.ww_v2_wp_energie_gesamt` unverändert übernommen |
| 3 | Ost/West-Zuordnung | A = Ost, B = West (`sensor.stp10_..._pv_power_a/_b`, entsprechend `sensor.energy_production_*` ohne Suffix = Ost, mit `_2` = West) | `sensor.ww_v2_pv_leistung_ost/_west`, alle Forecast-Kombinationen |
| 4 | Vorhandene Summen-Sensoren `pv_forecast_*_gesamt` | Funktionieren nicht, müssen aus den Solar-Production-Forecast-Sensoren neu gebaut werden | `sensor.ww_v2_pv_forecast_rest_heute_gesamt`/`_morgen_gesamt`/`_naechste_stunde_gesamt` neu berechnet, alte Sensoren nicht referenziert |
| 5 | Netzbezug/-einspeisung | `sensor.evu_leistung`: positiv = Bezug, negativ = Einspeisung | `sensor.ww_v2_netzbezug_watt`/`_netzeinspeisung_watt` |
| 6 | Aktueller PV-Überschuss | Aus `sensor.evu_leistung` ableiten (nicht aus `sensor.pv_ueberschuss_aktuell`) | `binary_sensor.ww_v2_pv_ueberschuss`, `sensor.ww_v2_pv_ueberschuss_watt` |
| 7 | Hausverbrauch | `sensor.gesamtleistung_haushalt` bestätigt | in Dashboard/Diagnose übernommen |
| 8 | Pushover-Service | `notify.pushover` bestätigt | zentral in `script.ww_v2_debug_log`/`script.ww_v2_pushover_test` |
| 9 | Urlaubsmodus | Weglassen | Funktion entfällt vollständig (Funktionsmatrix F22) |
| 10 | Legionellen-/Hygienefunktion | Übernehmen (ja) | `automation.ww_v2_hygiene_pruefung`, standardmässig deaktiviert |
| 11 | Dashboard-Inkonsistenz / konfigurierbare Entity-ID-Helfer | Ja (Konzept beibehalten) | `input_text.ww_v2_wp_energy_entity` bleibt konfigurierbar; die zwei fehlerhaften, nie definierten Tibber-Helfer aus v1_8 wurden nicht neu angelegt, da die Tibber-Quellen jetzt fest und robust referenziert sind (kein Bedarf mehr) |
| 12 | Wochen-/Monatskosten | Ja | `utility_meter.ww_v2_wp_energie_woche/_monat`, `sensor.ww_v2_kosten_woche_geschaetzt/_monat_geschaetzt` |

## Teil B – Verbleibende, nicht blockierende Punkte

Diese Punkte verhindern **nicht** die Aktivierung von ww_v2, sollten aber vom
Anwender zur Kenntnis genommen bzw. bei Gelegenheit entschieden werden.

**B1. Soll eine veraltete (aber numerisch gültige) Warmwassertemperatur
Ladeaktionen blockieren?**
Aktuell markiert `sensor.ww_v2_warmwassertemperatur` einen Wert nur als
`veraltet: true` (Attribut, sichtbar in Diagnose), wenn `water_heater.dhw1`
seit über 30 Minuten nicht aktualisiert wurde – blockiert aber keine Aktion.
Unklar: Wie häufig aktualisiert sich `water_heater.dhw1` im Normalbetrieb
tatsächlich? Falls die Entity z. B. nur alle 15–20 Minuten aktualisiert,
wäre eine 30-Minuten-Schwelle sinnvoll und ungefährlich als zusätzliches
Sperrkriterium nutzbar. Falls sie sich nur alle 45+ Minuten aktualisiert,
würde eine solche Sperre den Normalbetrieb stören.
→ Falls gewünscht: bitte kurz beobachten, wie oft sich
`state_last_updated` von `water_heater.dhw1` im Alltag ändert, dann kann die
Schwelle final festgelegt und als echtes Sperrkriterium ergänzt werden.

**B2. `switch.charge` auf der Debug-Seite – Status oder Bedienung?**
Aktuell wird `switch.charge` auf der Debug-Seite als normale
`entities`-Zeile gezeigt, was technisch weiterhin einen Kippschalter
darstellt (native HA-Karten haben keine reine Read-only-Option für
Switch-Entities). Der tatsächliche Ladezustand ist redundant auch über den
schreibgeschützten `binary_sensor.ww_v2_ladung_aktiv` sichtbar.
→ Falls eine strikt unbedienbare Debug-Seite gewünscht ist: die
`switch.charge`-Zeile aus der Debug-Karte „Aktueller Steuerungsstatus“
entfernen und ausschliesslich `binary_sensor.ww_v2_ladung_aktiv` anzeigen.
Aktuell bewusst beide gezeigt, da `switch.charge` auf der Debug-Seite als
Notfall-Zugriffsmöglichkeit nützlich sein kann.

**B3. Soll ein fehlgeschlagener `switch.turn_on`/`switch.turn_off`-Aufruf
selbst (nicht nur Pushover) künftig differenzierter behandelt werden?**
Siehe `ww_v2_quality_report.md`, Abschnitt 3, Punkt 2 – aktuell dokumentierte
Einschränkung, kein zusätzlicher Automatismus ergänzt, da nicht explizit
angefordert.

**B4. Recorder-Ausschlussliste**
In `ww_v2_installation.md` wird empfohlen, die internen `input_text.ww_v2_status_*`-
Helfer vom Recorder auszuschliessen. Das wurde nicht automatisch in eine
`recorder:`-Konfiguration im Package geschrieben, da mehrere `recorder:`-
Blöcke über Packages hinweg nicht zuverlässig zusammengeführt werden und ein
Konflikt mit einer eventuell bereits bestehenden `recorder:`-Konfiguration
des Anwenders vermieden werden sollte.

**B5. Kein `ha core check_config` in dieser Umgebung möglich**
Es stand keine laufende Home-Assistant-Instanz und kein Python/YAML-Parser
zur Verfügung, um die Dateien automatisiert zu validieren. Die in
`ww_v2_quality_report.md` beschriebenen statischen Prüfungen (Tab-Suche,
Klammer-/Schlüssel-Bilanz, manuelle Durchsicht) ersetzen dies nicht
vollständig. **Vor der ersten Aktivierung zwingend „Konfiguration
überprüfen“ in Home Assistant selbst ausführen.**
