# ww_v2 – Funktionsmatrix

Alle Rückfragen aus der Analysephase sind beantwortet (siehe `ww_v2_open_questions.md`
für die protokollierten Antworten). Diese Matrix dokumentiert jede Funktion aus v1_8,
ihre Bewertung und die Umsetzung in v2.

| # | Funktion | Fachliches Ziel | Bisherige Umsetzung (v1_8) | Trigger | Entscheidung v2 | Neue Umsetzung | Testmethode |
|---|---|---|---|---|---|---|---|
| F1 | Ist-Temperatur-Ermittlung | Validierte, sichere Ist-Temperatur ohne Fantasiewerte | 3 Fallbacks (`current_temperature`, `sensor.hotwater_temp`[Bosch], `sensor.dhw1_temperature`[existiert nicht]) | Template | **geändert**: nur noch `water_heater.dhw1`/`current_temperature`, bestätigt gültig (52.2 °C) | `sensor.ww_v2_warmwassertemperatur` mit `availability:`-Gate, keine Bosch-/Fantasiewerte, Altersattribut | Test 5–7 |
| F2 | Kritischer Bedarf (P0) | Sicherheitsuntergrenze | Vergleich Temp < Minimum, Default `float(99)` | State + 5-Min-Pattern | übernommen | `binary_sensor.ww_v2_kritischer_bedarf`, availability-gegated | Test 1–2 |
| F3 | Komfortbedarf | Komfortschwelle | wie F2 | – | übernommen | analog | Test 3–4 |
| F4 | Ladebedarf | Startschwelle mit Hysterese | Vergleich zu effektivem Ziel − Hysterese | – | übernommen | analog | Test 23 |
| F5 | PV-Überschuss (aktuell) | Erkennung Netzeinspeisung über Schwelle | `evu_leistung <= -Schwelle`, delay_on 3 min/delay_off 2 min | State + 5-Min-Pattern | **bestätigt unverändert** (Nutzerantwort Q6: aus `evu_leistung` ableiten) | `binary_sensor.ww_v2_pv_ueberschuss` + numerischer `sensor.ww_v2_pv_ueberschuss_watt` | Test 8–10 |
| F6 | Netzbezug/-einspeisung (Momentanwert) | Getrennte Anzeige | nicht vorhanden | – | **neu** (Nutzerantwort Q5: `evu_leistung` positiv=Bezug, negativ=Einspeisung) | `sensor.ww_v2_netzbezug_watt`, `sensor.ww_v2_netzeinspeisung_watt` | Test 34–36 |
| F7 | PV-Leistung Ost/West getrennt | Transparenz je String | nicht vorhanden (nur Forecast, keine Ist-Leistung je Anlage) | – | **neu** (Nutzerantwort Q3: A=Ost, B=West) | `sensor.ww_v2_pv_leistung_ost` (`…pv_power_a`), `…_west` (`…pv_power_b`), `…_gesamt` mit Ausfall-Fallback | Test 11–12 |
| F8 | PV-Forecast kombiniert (heute Rest, morgen, nächste Stunde) | Ost+West korrekt addieren | v1_8 nutzte nur unbenannten Forecast (Anlage 1) für Zielanpassung, Anlage 2 unbenutzt; externe „Gesamt“-Sensoren (`pv_forecast_*_gesamt`) fehlerhaft (Nutzerantwort Q4) | Template | **korrigiert**: eigene Summenbildung aus den 2×6 Forecast.Solar-Sensoren, mit Verfügbarkeits-Fallback je Anlage | `sensor.ww_v2_pv_forecast_rest_heute_gesamt`, `_morgen_gesamt`, `_naechste_stunde_gesamt` | Test 13–17 |
| F9 | Eigenverbrauchsvorteil / wirtschaftliche PV-Nutzung | ct/kWh-Vorteil ggü. Einspeisung | Preis − Einspeisevergütung ≥ Mindestvorteil | Template | übernommen | `sensor.ww_v2_eigenverbrauchsvorteil`, `binary_sensor.ww_v2_eigenverbrauch_wirtschaftlich` | Test 18 |
| F10 | Tibber Leading24 / Bestpreis-Fenster | Günstiges 24h-Fenster ohne fragile Attribut-Suche | Attribut-Ratekette über bis zu 5 mögliche Attributnamen, tote Fallback-Sensoren | Template | **korrigiert**: direkte, bestätigt vorhandene Sensoren `sensor.home_bestpreis_startet/_endet/_startet_in/_verbleibend` statt Attribut-Raten | `sensor.ww_v2_naechster_bestpreis_start`, `_in` | Test 18–20 |
| F11 | Morgige Preise fehlen (Fallback) | Sicherer Umgang mit `unknown` | `float(999)`/`float(0)`-Defaults ohne explizite Prüfung | Template | übernommen, mit explizitem availability-Check statt Silent-Default | `binary_sensor.ww_v2_tibber_wirtschaftlich` mit Fallback auf aktuellen Preis | Test 20 |
| F12 | Marstek-Batteriespeicher-Berücksichtigung | Ladefreigabe/-sperre je nach SoC | 3-Stufen-Logik (Freigabe/Eingeschränkt/Schutz), toleriert Ausfall (`unavailable`) | Template | übernommen unverändert (Marstek ist aktuell hardwareseitig ausgefallen, Fallback-Logik bereits korrekt) | `sensor.ww_v2_marstek_stufe` etc. | Test 24 (analog) |
| F13 | Anti-Takt-Sperre | Verdichterschutz | 20 min seit letztem Ladeende | Template | übernommen | `binary_sensor.ww_v2_antitakt_frei` | Test 22 |
| F14 | Mindestlaufzeit | Verdichterschutz | 30 min seit letzter Ladung | Template | übernommen | `binary_sensor.ww_v2_mindestlaufzeit_erreicht` | Test 22 |
| F15 | Dynamisches Ziel (gelernter Bedarf + PV-Prognose) | Zieltemperatur adaptiv anheben/absenken | Nutzte nur Anlage-1-Forecast (`energy_production_tomorrow`) | Template | **korrigiert**: nutzt `sensor.ww_v2_pv_forecast_morgen_gesamt` (Ost+West) | `sensor.ww_v2_dynamisches_ziel` | Test 13–14 |
| F16 | Effektives Ziel (PV-Wärmespeicher-Modus) | Bei PV-Überschuss auf Maximaltemperatur laden | wie F15, aufbauend | Template | übernommen | `sensor.ww_v2_effektives_ziel` | Test 8 |
| F17 | Ladung starten (Zieltemperatur+Dauer setzen, Schalter ein) | Ausführende Aktion | `number.set_value` ×2 + `switch.turn_on`, kein Idempotenz-Schutz gegen Doppelklick innerhalb des Skripts | Skriptaufruf | **verbessert**: Skript prüft zusätzlich intern `switch.charge` State und Verfügbarkeit, bricht bei bereits laufender Ladung/Nichtverfügbarkeit ab und protokolliert | `script.ww_v2_ladung_starten`, `mode: single` | Test 24–25, 32–33 |
| F18 | Ladung stoppen bei Zielerreichung | Verdichterschutz + Komfort | Template-Trigger + Mindestlaufzeit-Bedingung | Template + 2-Min-Pattern | übernommen | `automation.ww_v2_stop_ziel` | Test 3 |
| F19 | Sicherheitsabschaltung bei Maximaldauer | Schutz vor endlos laufender Ladung nach Neustart während `wait_template` | **fehlte in v1_8** (Risiko in Analyse Abschnitt 7 dokumentiert) | – | **neu** | `automation.ww_v2_sicherheitsabschaltung_maxdauer`, `input_number.ww_v2_max_ladedauer_minuten` | Test 28 |
| F20 | Nachtgarantie 04:00 | Fallback ohne PV/Preis-Fenster | Zeit-Trigger + Komfortbedarf-Bedingung | Zeit | übernommen | `automation.ww_v2_nachtgarantie` | Test 1, 3 |
| F21 | Optionaler Hygienezyklus | Legionellenschutz, standardmäßig deaktiviert | Sonntags 02:00, Sicherheitshinweis im Kommentar, Prüfung auf `number.charge_setpoint`-Max-Attribut (unverifiziert) | Zeit + Bedingung | **übernommen** (Nutzerantwort Q10: ja), Prüfung auf unverifiziertes Attribut entfernt, stattdessen Begrenzung durch `input_number`-Grenzen (60–65 °C) | `automation.ww_v2_hygiene_pruefung` | – (bleibt standardmäßig aus, manueller Test bei Bedarf) |
| F22 | Urlaubsmodus | Reduzierter Betrieb im Urlaub | nicht vorhanden (nur Bosch-Urlaubsmodus im System, nicht nutzbar) | – | **entfällt** (Nutzerantwort Q9: weglassen) | – | – |
| F23 | Nachtverlust lernen | Adaptive Zielanpassung | Snapshot 22:00, Lernen 05:25, EMA 0.7/0.3 | Zeit | übernommen | `automation.ww_v2_lernen_nacht*` | – (Langzeitverhalten, kein Einzeltest) |
| F24 | Verbrauch morgens/abends lernen | Adaptive Zielanpassung | Snapshot+Lernen je Tageshälfte, EMA 0.75/0.25 | Zeit | übernommen | `automation.ww_v2_verbrauch_*` | – |
| F25 | Kostenabschluss je Ladevorgang | Tageskosten/-einsparung, PV-/Netzenergie-Split | Energiezähler-Differenz × Preis bzw. Einspeisevergütung | State (`switch.charge` on→off) | übernommen | `automation.ww_v2_kostenabschluss` | Test „Kosten“ (siehe Testplan) |
| F26 | Wochen-/Monatskosten | Längerfristige Kostenübersicht | **fehlte in v1_8** | – | **neu** (Nutzerantwort Q12: ja) | `utility_meter.ww_v2_wp_energie_woche/_monat` (gemessen) + `sensor.ww_v2_kosten_woche_geschaetzt/_monat_geschaetzt` (geschätzt, klar gekennzeichnet) | – |
| F27 | Tagesreset | Zähler/Kosten um Mitternacht zurücksetzen | Zeit-Trigger 00:00:05 | Zeit | übernommen | `automation.ww_v2_reset` | – |
| F28 | Ampel-Status | Schnelle Systemdiagnose | Text-State rot/orange/grün + Klartext-Attribut | Template | übernommen, PV-Prognose-Verfügbarkeit jetzt für beide Anlagen geprüft | `sensor.ww_v2_ampel` | Test 15–17 |
| F29 | Debug-Modus + Pushover | Zentrale, deduplizierte Entscheidungsprotokollierung | 2 einzelne Automationen mit direktem `notify.pushover`-Aufruf, keine Deduplizierung, kein zentrales Skript | State-Trigger | **grundlegend neu** (siehe Abschnitt 15–18 der Aufgabenstellung) | `script.ww_v2_debug_log`, zentral aus allen Aktionspunkten aufgerufen, mit Deduplizierung und `continue_on_error` | Test 26–27, 30–33 |
| F30 | WP-Energiezähler | Kumulierte elektrische Energie für Kostenrechnung | `platform: integration` (Riemann) aus `sensor.wp_gesamtleistung` | – | **bestätigt unverändert** (Nutzerantwort Q2: Option a) | `sensor.ww_v2_wp_energie_gesamt` | – |
| F31 | Temperatur-Maximum 7 Tage (Hygiene-Nachweis) | Nachweis, ob 60 °C in 7 Tagen erreicht wurde | `platform: statistics` | – | übernommen | `sensor.ww_v2_temperatur_maximum_7_tage` | – |
| F32 | Dynamisch konfigurierbare Quell-Entity-Helfer | Flexibilität bei Entity-Umbenennung | `input_text.ww_v18_wp_energy_entity` (genutzt), `ww_v18_tibber_leading24_entity`/`_next_best_start_entity` (im Dashboard erwartet, im Package nie definiert – Fehler) | – | **bereinigt** (Nutzerantwort Q11: Konzept beibehalten, aber Inkonsistenz beheben) | nur `input_text.ww_v2_wp_energy_entity` bleibt als konfigurierbarer Helfer; die beiden nicht existierenden Tibber-Helfer entfallen ersatzlos, da Tibber-Sensoren jetzt fest und eindeutig referenziert werden (kein Attribut-Rateverfahren mehr nötig) | Test „YAML/Entity-Konsistenz“ |
| F33 | Pflichtdaten-Prüfung vor jeder Aktion | Verhindert Aktionen bei ungültigen Kerndaten | nicht explizit vorhanden (nur implizit über einzelne `availability`-Templates) | – | **neu** (Abschnitt 12, Priorität 1 der Aufgabenstellung) | `binary_sensor.ww_v2_pflichtdaten_gueltig`, als Bedingung 1 in allen ladeauslösenden Automationen | Test 5–7, 24 |
| F34 | Diagnose fehlender Pflicht-Entities | Übersicht, welche Kern-Entity aktuell nicht verfügbar ist | nicht vorhanden | – | **neu** | `sensor.ww_v2_diagnose_pflichtentities` | Test 24, 34–36 |

## Entfernte/geänderte Funktionen – Begründung im Detail

- **`sensor.hotwater_temp`-Fallback entfernt**: Bosch-Entity, laut Vorgabe verboten.
  Auswirkung: Bei Ausfall von `water_heater.dhw1` gibt es keinen Fallback mehr –
  das ist beabsichtigt, da laut Sicherheitsregel 3 kein erfundener/Fremdwert eine
  Aktion auslösen darf. Migration: keine, da der Fallback ohnehin nie sauber nutzbar
  war (Bosch-Ausschluss).
- **`sensor.dhw1_temperature`-Fallback entfernt**: Entity existiert nicht im System.
- **Attribut-Ratekette für „nächster Bestpreis-Start“ entfernt**: ersetzt durch die
  bestätigt vorhandenen, direkten Sensoren. Auswirkung: robuster, weniger Codezeilen,
  kein Rateverfahren über 5 mögliche Attributnamen mehr.
- **`state_attr('number.charge_setpoint','max')`-Prüfung im Hygienezyklus entfernt**:
  Das Attribut ist nicht verifizierbar. Ersatz: `input_number.ww_v2_hygiene_ziel` ist
  ohnehin durch `min: 60, max: 65` begrenzt – ausreichende Absicherung ohne
  unbestätigte Attributabfrage.
- **Zwei nicht existierende Dashboard-Helfer (`ww_v18_tibber_leading24_entity`,
  `ww_v18_tibber_next_best_start_entity`) entfernt**: Sie wurden im Dashboard anzeigt,
  aber nie im Package definiert (Fehler in v1_8). Da die Tibber-Sensoren in v2 fest
  referenziert werden, entfällt der Bedarf für konfigurierbare Entity-ID-Helfer an
  dieser Stelle vollständig.
