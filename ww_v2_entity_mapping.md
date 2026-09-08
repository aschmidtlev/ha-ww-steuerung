# ww_v2 – Entity-Mapping (alt → neu)

Legende Ergebnis: **übernommen** = gleiche externe Quelle, neuer eigener Name;
**ersetzt** = andere/bessere Quelle für dieselbe Funktion; **entfernt** = Funktion
oder Fallback entfällt; **offen** = weiterhin ungeklärt (siehe `ww_v2_open_questions.md`).

## Externe (fremde) Quell-Entities – unverändert referenziert

| Alte Referenz (v1_8) | Neue Referenz (v2) | Funktion | Bosch-Abhängigkeit | Ergebnis | Begründung |
|---|---|---|---|---|---|
| `water_heater.dhw1` (Attribut `current_temperature`) | unverändert | Ist-Temperatur-Quelle | technisch Bosch-Integration (`bosch_state`-Attribut vorhanden), aber vom Nutzer in Abschnitt 3.1 explizit bestätigt | übernommen | Nutzer-Bestätigung hat Vorrang; Attribut verifiziert (52,2 °C, °C-Bereich 0–100) |
| `switch.charge` | unverändert | Ladeauslöser | nein (kein „bosch“ im Namen, generischer Schalter) | übernommen | vom Nutzer bestätigt |
| `number.charge_setpoint` | unverändert | Zieltemperatur für Ladung | nein | übernommen | vom Nutzer implizit bestätigt (Teil des Ladevorgangs) |
| `number.charge_duration` | unverändert | Ladedauer-Begrenzung | nein | übernommen | wie oben |
| `sensor.hotwater_temp` | – | Fallback #2 Ist-Temperatur | **ja** (Friendly Name „Bosch sensors“) | **entfernt** | Bosch-Verbot |
| `sensor.dhw1_temperature` | – | Fallback #3 Ist-Temperatur | unklar | **entfernt** | Entity existiert nicht |
| `sensor.wp_gesamtleistung` | unverändert | Momentanleistung WP für Riemann-Integration | nein | übernommen | Nutzerantwort Q2 = a |
| `sensor.home_aktueller_strompreis` | unverändert | aktueller Strompreis | nein | übernommen | bestätigte Quelle |
| `sensor.home_preis_vorlaufend_24h` | unverändert | Leading24-Durchschnittspreis | nein | übernommen | einzige vorhandene Quelle |
| `sensor.home_leading_price_average` | – | Fallback Leading24 | – | entfernt | Entity existiert nicht |
| `binary_sensor.home_bestpreis_zeitraum` | unverändert | Bestpreis-Fenster aktiv (Trigger/Bedingung) | nein | übernommen | – |
| `state_attr('binary_sensor.home_bestpreis_zeitraum', 'next_start' u. a.)` | `sensor.home_bestpreis_startet` | nächster Bestpreis-Start | nein | **ersetzt** | robuste direkte Entity statt Attribut-Ratekette |
| – | `sensor.home_bestpreis_startet_in` | Restzeit bis Bestpreis | nein | neu übernommen | zusätzliche Diagnose |
| `sensor.energy_production_today` | unverändert (= Anlage 1 / **Ost**) | PV-Forecast heute, Anlage Ost | nein | übernommen | Nutzerantwort Q3 |
| `sensor.energy_production_today_2` | unverändert (= Anlage 2 / **West**) | PV-Forecast heute, Anlage West | nein | übernommen | Nutzerantwort Q3 |
| `sensor.energy_production_today_remaining[_2]` | unverändert | Restprognose heute je Anlage | nein | übernommen | – |
| `sensor.energy_production_tomorrow[_2]` | unverändert | Prognose morgen je Anlage | nein | übernommen | – |
| `sensor.energy_current_hour[_2]` / `sensor.energy_next_hour[_2]` | unverändert | kurzfristiger Forecast je Anlage | nein | übernommen | für kombinierten Kurzfrist-Forecast |
| `sensor.pv_forecast_heute_gesamt` / `_morgen_gesamt` | – | vorgefertigte Summen-Sensoren | nein | **entfernt/ersetzt** | laut Nutzerantwort Q4 funktionsuntüchtig – v2 bildet die Summe selbst |
| `sensor.stp10_0_3av_40_040_pv_power_a` | Basis für `sensor.ww_v2_pv_leistung_ost` | aktuelle Leistung Ost-String | nein | übernommen | Nutzerantwort Q3 |
| `sensor.stp10_0_3av_40_040_pv_power_b` | Basis für `sensor.ww_v2_pv_leistung_west` | aktuelle Leistung West-String | nein | übernommen | Nutzerantwort Q3 |
| `sensor.evu_leistung` | unverändert, Basis mehrerer neuer Sensoren | Netzbezug (positiv)/-einspeisung (negativ), PV-Überschuss | nein | übernommen | Nutzerantworten Q5, Q6 |
| `sensor.gesamtleistung_haushalt` | unverändert | Hausverbrauch aktuell | nein | übernommen | Nutzerantwort Q7 |
| `sensor.marstek_venus_modbus_soc_batterie` | unverändert | Batterie-SoC | nein | übernommen | unverändert aus v1_8, aktuell hardwareseitig ausgefallen (toleriert) |
| `input_number.ww_v18_einspeiseverguetung` | `input_number.ww_v2_einspeiseverguetung` | Einspeisevergütung, Vorgabe 0,08 EUR/kWh | nein | übernommen | Startwert 8 ct/kWh gemäß Vorgabe |
| `notify.pushover` | unverändert | Pushover-Zielservice | nein | übernommen | Nutzerantwort Q8, Beleg aus produktivem v1_8-Code |
| `climate.hc1`, `sensor.actual_*`, `sensor.outdoor_temperature`, `select.holiday_*`, `sensor.ww_bosch_*`, alle „Bosch sensors“-Gruppe | – | diverse Bosch-Funktionen | **ja** | **entfernt** | Bosch-Verbot (Abschnitt 4 der Aufgabenstellung) |

## Package-eigene Entities (v1_8 → v2, reine Umbenennung ww_v18_/ww_v1_8_ → ww_v2_)

| Alt | Neu |
|---|---|
| `input_boolean.ww_v18_aktiv` | `input_boolean.ww_v2_aktiv` |
| `input_boolean.ww_v18_debug` | `input_boolean.ww_v2_debug_mode` (Name laut Vorgabe Abschnitt 1 exakt so) |
| `input_boolean.ww_v18_marstek_nutzen` | `input_boolean.ww_v2_marstek_nutzen` |
| `input_boolean.ww_v18_pv_vorrang` | `input_boolean.ww_v2_pv_vorrang` |
| `input_boolean.ww_v18_hygiene_aktiv` | `input_boolean.ww_v2_hygiene_aktiv` |
| `input_boolean.ww_v18_hygiene_erfolgt` | `input_boolean.ww_v2_hygiene_erfolgt` |
| `input_boolean.ww_v18_nachtladung_erfolgt` | `input_boolean.ww_v2_nachtladung_erfolgt` |
| `input_button.ww_v18_debug_test` | `input_button.ww_v2_debug_test` |
| `input_text.ww_v18_letzter_grund` | `input_text.ww_v2_letzter_grund` |
| `input_text.ww_v18_aktive_strategie` | `input_text.ww_v2_aktive_strategie` |
| `input_text.ww_v18_wp_energy_entity` | `input_text.ww_v2_wp_energy_entity` |
| `input_text.ww_v18_tibber_leading24_entity` | – (entfernt, Begründung: Funktionsmatrix F32) |
| `input_text.ww_v18_tibber_next_best_start_entity` | – (entfernt, Begründung: Funktionsmatrix F32) |
| `input_datetime.ww_v18_letzte_ladung` | `input_datetime.ww_v2_letzte_ladung` |
| `input_datetime.ww_v18_letztes_ladeende` | `input_datetime.ww_v2_letztes_ladeende` |
| `input_datetime.ww_v18_letzter_hygienezyklus` | `input_datetime.ww_v2_letzter_hygienezyklus` |
| `counter.ww_v18_ladungen_heute` | `counter.ww_v2_ladungen_heute` |
| alle `input_number.ww_v18_*` | `input_number.ww_v2_*` (identische Grenzwerte/Startwerte) |
| `sensor.ww_v1_8_wp_energie_gesamt` | `sensor.ww_v2_wp_energie_gesamt` |
| `sensor.ww_v1_8_temperatur_maximum_7_tage` | `sensor.ww_v2_temperatur_maximum_7_tage` |
| alle `binary_sensor.ww_v1_8_*` | `binary_sensor.ww_v2_*` |
| alle `sensor.ww_v1_8_*` | `sensor.ww_v2_*` |
| `script.ww_v18_ladung_starten` | `script.ww_v2_ladung_starten` |
| alle `automation.ww_v1_8_*` | `automation.ww_v2_*` |

## Neue Entities ohne v1_8-Vorbild

| Neue Entity | Funktion |
|---|---|
| `binary_sensor.ww_v2_pflichtdaten_gueltig` | Sammel-Verfügbarkeitsprüfung Kern-Entities vor jeder Aktion |
| `sensor.ww_v2_diagnose_pflichtentities` | Liste/Anzahl nicht verfügbarer Pflicht-Entities |
| `sensor.ww_v2_pv_leistung_ost` / `_west` / `_gesamt` | aktuelle PV-Leistung je Anlage + kombiniert mit Ausfall-Fallback |
| `sensor.ww_v2_pv_ueberschuss_watt` | numerischer PV-Überschuss in W |
| `sensor.ww_v2_netzbezug_watt` / `_netzeinspeisung_watt` | vorzeichenkorrekt aus `evu_leistung` abgeleitet |
| `sensor.ww_v2_pv_forecast_rest_heute_gesamt` / `_morgen_gesamt` / `_naechste_stunde_gesamt` | kombinierte Ost+West-Prognosen mit Ausfall-Fallback |
| `binary_sensor.ww_v2_pv_ost_verfuegbar` / `_west_verfuegbar` | Diagnose je Anlage |
| `script.ww_v2_debug_log` | zentrales Debug-/Pushover-Skript |
| `input_button.ww_v2_debug_test` → Automation | löst Testnachricht aus |
| `input_number.ww_v2_pushover_mindestabstand_min` | Deduplizierungsintervall |
| `input_number.ww_v2_max_ladedauer_minuten` | Sicherheitsabschaltung (Funktionsmatrix F19) |
| `automation.ww_v2_sicherheitsabschaltung_maxdauer` | neue Sicherheitsfunktion |
| `input_text.ww_v2_letzte_debug_nachricht` | Deduplizierungs-Signatur |
| `input_text.ww_v2_letzter_fehler` / `_letzte_pushover_nachricht` | lokale Diagnose-Anzeige |
| `input_datetime.ww_v2_letzte_pushover_zeit` / `_letzter_fehler_zeit` / `_letzter_erfolgreicher_durchlauf` / `_letzter_uebersprungener_durchlauf` | Zeitstempel für Diagnose |
| `counter.ww_v2_aktionen_gesamt` / `_blockierte_aktionen` / `_fehler_gesamt` | Zählwerke für Diagnose |
| `utility_meter.ww_v2_wp_energie_woche` / `_monat` | gemessene Wochen-/Monatsenergie (Nutzerantwort Q12) |
| `sensor.ww_v2_kosten_woche_geschaetzt` / `_monat_geschaetzt` | geschätzte Wochen-/Monatskosten, explizit als „geschätzt“ gekennzeichnet |
| `sensor.ww_v2_last_action` / `_last_reason` / `_last_result` / `_last_error` / `_last_pushover_status` | zentrale, kompakte Statusanzeige für Debug-Seite |

## Entfallene Funktionen (kein v2-Äquivalent)

| Funktion | Grund |
|---|---|
| Urlaubsmodus | Nutzerantwort Q9: weglassen (einzige vorhandene Implementierung war Bosch-gebunden) |
