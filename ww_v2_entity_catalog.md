# ww_v2 – Entity-Katalog (aus CSV-Analyse)

Quelle: `Copilot/harvest_2026-08-29-22-35-33.csv` (BOM UTF-8, Trennzeichen `,`,
Felder in Anführungszeichen, 12 Spalten: Domain, Entity ID, Name, State, Unit,
Device class, Last changed, Last updated, attr_id, attr_last_triggered, attr_mode,
attr_current). **Wichtige Einschränkung:** Die CSV enthält nur die vier generischen
Attributspalten `attr_id / attr_last_triggered / attr_mode / attr_current` – sie
erfasst NICHT die vollständigen Attribute jeder Entity (z. B. fehlen
`current_temperature`, `min`, `max`, `next_start` u. ä. komplett). Aussagen zu
Attributen einzelner Entities, die hier nicht in dieser CSV stehen, sind daher nicht
aus der CSV verifizierbar und werden entsprechend gekennzeichnet.

Domainverteilung (Auszug, gesamt 2080 Entities): sensor 1202, device_tracker 198,
switch 144, binary_sensor 134, update 69, button 62, select 45, input_number 35,
media_player 33, automation 31, input_boolean 14, notify 13, number 11,
input_text 8, input_datetime 3, script 2, input_button 2, counter 1, climate 1,
water_heater 1.

## 1. Warmwasser

| Entity-ID | Friendly Name | Domain | State (Snapshot) | Unit | Device class | Verwendung v1_8 | Geplant v2 | Status |
|---|---|---|---|---|---|---|---|---|
| `water_heater.dhw1` | Water heater dhw1 dhw1 | water_heater | `performance` | – | – | Quelle Ist-Temperatur via Attribut `current_temperature` (Annahme) | zentrale Ist-Temperatur-Quelle (Attribut-Existenz muss noch verifiziert werden) | **unklar – Rückfrage 1** |
| `sensor.hotwater_temp` | **Bosch sensors** | sensor | `unknown` | – | temperature | Fallback #2 Ist-Temperatur | **ausgeschlossen (Bosch)** | entfernt |
| `sensor.dhw1_temperature` | – (existiert nicht) | – | – | – | – | Fallback #3 Ist-Temperatur | entfällt (Entity nicht vorhanden) | entfernt |
| `sensor.dhw1_water_flow` | Water heater dhw1 | sensor | `0.0` | l/min | – | nicht verwendet | nicht benötigt | unklar/nicht genutzt |
| `sensor.dhw1_working_time` | Water heater dhw1 | sensor | `0` | mins | – | nicht verwendet | nicht benötigt | unklar/nicht genutzt |
| `switch.charge` | Water heater dhw1 charge | switch | `off` | – | – | Ladeauslöser | Ladeauslöser (bestätigt) | übernommen |
| `number.charge_setpoint` | Water heater dhw1 Charge setpoint | number | `51.0` | °C | – | Zieltemperatur für Ladevorgang | übernehmen | übernommen |
| `number.charge_duration` | Water heater dhw1 Charge duration | number | `45.0` | mins | – | Ladedauer-Begrenzung | übernehmen | übernommen |
| `sensor.ww_bosch_betriebsmodus` | WW Bosch Betriebsmodus | sensor | `unavailable` | – | – | nicht referenziert | **ausgeschlossen (Bosch)** | entfernt |
| `sensor.ww_bosch_status` | WW Bosch Status | sensor | `unavailable` | – | – | nicht referenziert | **ausgeschlossen (Bosch)** | entfernt |

## 2. Wärmepumpe / elektrische Energie DHW

| Entity-ID | Friendly Name | Unit | Device class | Verwendung v1_8 | Geplant v2 | Status |
|---|---|---|---|---|---|---|
| `sensor.wp_gesamtleistung` | WP Gesamtleistung | W | power | Quelle für Riemann-Integration (`sensor.ww_v1_8_wp_energie_gesamt`) | Kandidat für Momentanleistung | Rückfrage 2 |
| `sensor.warmepumpe_device_energy` | Wärmepumpe Device Energy | kWh | energy | nicht verwendet | Kandidat als **direkter** kumulierter Energiezähler (keine Integration-Plattform nötig) | Rückfrage 2 |
| `sensor.warmepumpe_device_power` | Wärmepumpe Device Power | W | – | nicht verwendet | alternative Momentanleistung | Rückfrage 2 |
| `switch.warmepumpe` | Wärmepumpe | – | – | nicht verwendet | nicht benötigt | unklar/nicht genutzt |
| `binary_sensor.warmepumpe_uberlast` | Wärmepumpe Überlast | – | problem | nicht verwendet | ggf. Sicherheitsindikator | unklar |
| `sensor.rcompressor` | Recording sensors rcompressor | – | energy | nicht verwendet | ungeklärter Zweck, State `unknown` | unklar |
| `sensor.home_primare_heizquelle` | Home Primäre Heizquelle | – | enum | nicht verwendet | informativ | unklar |

## 3. Charge (siehe auch Abschnitt 1)

Abgedeckt oben. Kein weiterer eigenständiger „Charge“-Sensor-Block gefunden.

## 4. Strompreis / Tibber

| Entity-ID | Friendly Name | Unit | Device class | Verwendung v1_8 | Geplant v2 | Status |
|---|---|---|---|---|---|---|
| `sensor.home_aktueller_strompreis` | Home Aktueller Strompreis | ct/kWh | monetary | aktueller Preis (bestätigte Quelle) | übernehmen | übernommen |
| `sensor.home_aktueller_strompreis_energie_dashboard` | Home Aktueller Strompreis (Energie-Dashboard) | €/kWh | monetary | nicht verwendet | nicht benötigt (Redundanz) | nicht genutzt |
| `sensor.home_preis_vorlaufend_24h` | Home ⌀ Preis vorlaufend 24h | ct/kWh | monetary | Leading24-Durchschnitt (primäre Quelle) | übernehmen als alleinige Leading24-Quelle | übernommen |
| `sensor.home_leading_price_average` | – (existiert nicht) | – | – | Fallback für Leading24 | entfällt | entfernt |
| `binary_sensor.home_bestpreis_zeitraum` | Home Bestpreis-Zeitraum | – | – | Trigger + Attribut-Ratelogik für Bestpreis-Start | Trigger/Bedingung übernehmen, Attribut-Ratelogik ersetzen | übernommen (Nutzung geändert) |
| `sensor.home_bestpreis_startet` | Home Bestpreis startet | – | timestamp | nicht verwendet | **Ersatz** für `ww_v1_8_naechster_bestpreis_start` | neu übernommen |
| `sensor.home_bestpreis_startet_in` | Home Bestpreis startet in | min | duration | nicht verwendet | Anzeige „verbleibende Zeit bis Bestpreis“ | neu übernommen |
| `sensor.home_bestpreis_endet` | Home Bestpreis endet | – | timestamp | nicht verwendet | Anzeige | neu übernommen |
| `sensor.home_bestpreis_verbleibend` | Home Bestpreis verbleibend | min | duration | nicht verwendet | Anzeige | neu übernommen |
| `sensor.home_spitzenpreis_zeitraum` (binary_sensor) / `_startet` / `_endet` / `_verbleibend` | Home Spitzenpreis … | – | – | nicht verwendet | ggf. für „hoher Strompreis“-Anzeige | Kandidat, optional |
| `sensor.home_preis_morgen` | Home ⌀ Preis morgen | ct/kWh | monetary | nicht direkt verwendet | Diagnose „morgige Preise verfügbar?“ | neu übernommen (Diagnose) |
| `sensor.home_hochstpreis_morgen`, `sensor.home_mindestpreis_morgen`, `sensor.home_preisniveau_morgen` | – | ct/kWh / – | monetary/enum | nicht verwendet | optional Diagnose | Kandidat |
| `sensor.home_preis_nachste_1h` … `_5h` | Home ⌀ Preis nächste Xh | ct/kWh | monetary | nicht verwendet | Kandidat für kurzfristige Preisbewertung | Kandidat |
| `sensor.home_aktuelle_preisphase`, `home_preisniveau_heute`, `home_aktuelles_preisniveau` | – | – | enum | nicht verwendet | informativ | Kandidat |
| `input_number.ww_v18_einspeiseverguetung` | WW V1.8 Einspeiseverguetung | ct/kWh | – | Einspeisevergütung, aktuell 8 | ersetzen durch fest `0,08 EUR/kWh` gem. Vorgabe (weiterhin als Helfer, Startwert 8) | übernommen |

## 5. PV – Forecast (zwei Anlagen, Muster `sensor.energy_*` / `sensor.energy_*_2`)

Beide Reihen stammen erkennbar von zwei separaten Forecast.Solar-Konfigurationen
(identischer Friendly-Name-Text „Solar production forecast …“ für beide Reihen –
**keine Unterscheidung nach Ost/West im Namen möglich**).

| Entity-ID (Anlage 1) | Entity-ID (Anlage 2) | Bedeutung | Unit | Device class |
|---|---|---|---|---|
| `sensor.power_production_now` | `sensor.power_production_now_2` | aktuelle Erzeugungsleistung (Forecast-Modell) | W | power |
| `sensor.energy_current_hour` | `sensor.energy_current_hour_2` | geschätzte Energie diese Stunde | kWh | energy |
| `sensor.energy_next_hour` | `sensor.energy_next_hour_2` | geschätzte Energie nächste Stunde | kWh | energy |
| `sensor.energy_production_today` | `sensor.energy_production_today_2` | Tagesprognose gesamt | kWh | energy |
| `sensor.energy_production_today_remaining` | `sensor.energy_production_today_remaining_2` | Restprognose heute | kWh | energy |
| `sensor.energy_production_tomorrow` | `sensor.energy_production_tomorrow_2` | Prognose morgen | kWh | energy |

Snapshot-Werte: Anlage 1 heute 10,183 kWh / morgen 13,562 kWh; Anlage 2 heute
7,881 kWh / morgen 11,525 kWh. Beide Reihen sind Energie in kWh bzw. Leistung in W –
fachlich gleiche Größen, gleiche Einheit, daher **grundsätzlich additionsfähig**,
sofern beide denselben Zeitraum/dieselbe Aktualität haben (beide `Last updated`
identisch 00:01:03 → passt).

Zusätzlich gefunden, aber **nicht in den beiden gelieferten v1_8-Dateien referenziert**
und aus keiner anderen vorliegenden Datei erklärbar:

| Entity-ID | Friendly Name | State | Unit | Bemerkung |
|---|---|---|---|---|
| `sensor.pv_forecast_heute_gesamt` | PV Forecast Heute Gesamt | 18,064 | kWh | ≈ Summe Anlage1+2 heute (10,183+7,881=18,064) – vermutlich bereits ein kombinierter Summen-Sensor aus einem nicht vorliegenden Package |
| `sensor.pv_forecast_morgen_gesamt` | PV Forecast Morgen Gesamt | 25,087 | kWh | ≈ Summe Anlage1+2 morgen (13,562+11,525=25,087) |

→ Rückfrage 3/4: Existiert dieses Summen-Package bereits produktiv und soll es
weiterverwendet werden, oder soll ww_v2 die Summenbildung selbst neu übernehmen?

## 6. PV – aktuelle Leistung / Wechselrichter

| Entity-ID | Friendly Name | Unit | Device class | Bemerkung |
|---|---|---|---|---|
| `sensor.stp10_0_3av_40_040_pv_power` | STP10.0-3AV-40 040 PV Power | W | power | Gesamtleistung des (einzigen gefundenen) SMA-Wechselrichters STP10.0 |
| `sensor.stp10_0_3av_40_040_pv_power_a` | STP10.0-3AV-40 040 PV Power A | W | power | String A – Kandidat für Anlage 1 |
| `sensor.stp10_0_3av_40_040_pv_power_b` | STP10.0-3AV-40 040 PV Power B | W | power | String B – Kandidat für Anlage 2 |
| `sensor.stp10_0_3av_40_040_inverter_power_limit` | … Inverter Power Limit | W | power | 10000 W, informativ |
| `sensor.stp10_0_3av_40_040_inverter_condition` | … Inverter Condition | – | – | `Ok` |
| `sensor.pv_ueberschuss_aktuell` | PV Ueberschuss Aktuell | W | – | 0 – vorgefertigter Überschuss-Sensor, nicht von v1_8 genutzt (v1_8 leitet PV-Überschuss stattdessen aus `sensor.evu_leistung` ab) |
| `sensor.pv_garage_*` (5 Diagnose-Entities) | PV Garage … | – | – | alle `unavailable`, reine MQTT/WLAN-Diagnose eines Geräts „PV Garage“, keine Leistungswerte |

**Wichtiger Befund:** Es gibt nur **einen** physischen Wechselrichter (SMA
STP10.0-3AV-40) mit zwei MPPT-Strings (A/B) – keine zwei getrennten Wechselrichter.
Die „zwei PV-Anlagen“ des Nutzers sind vermutlich diese zwei Strings, abgebildet durch
zwei parallele Forecast.Solar-Konfigurationen. Das deckt sich mit der Vorgabe
„Ost-West-Ausrichtung“ (ein Dreiphasen-Wechselrichter mit Ost- und West-String ist
eine sehr verbreitete Konfiguration). Zuordnung A↔Anlage1 bzw. B↔Anlage2 sowie
Ost/West ist aus den Daten selbst nicht ableitbar → Rückfrage 3.

## 7. Netzbezug / Netzeinspeisung / Hausverbrauch

| Entity-ID | Friendly Name | State | Unit | Device class | Bemerkung |
|---|---|---|---|---|---|
| `sensor.evu_leistung` | EVU Leistung | 699,871 | W | power | von v1_8 als Basis für PV-Überschuss verwendet (negativ = Einspeisung, Annahme aus Code) |
| `sensor.netzbezug_aktuell` | Netzbezug Aktuell | 699,871 | W | – | identischer Wert wie `evu_leistung` im Snapshot – vermutlich Duplikat/Alias |
| `sensor.evu_scheinleistung` | EVU Scheinleistung | 1355,262 | VA | apparent_power | nicht benötigt für Wirkleistungs-Logik |
| `sensor.evu_energie` | EVU Energie | 3937,681 | kWh | energy | kumulierter Netzbezug (Zähler) |
| `sensor.evu_energieeinspeisung` | EVU Energieeinspeisung | 5325,123 | kWh | energy | kumulierte Einspeisung (Zähler) |
| `sensor.tibber_pulse_home_stromerzeugung` | Tibber Pulse Home **Einspeiseleistung** | 0 | W | power | Name/Friendly-Name-Inkonsistenz (Entity-ID sagt „Stromerzeugung“, Friendly Name sagt „Einspeiseleistung“) – Kandidat für Momentan-Einspeiseleistung, aber unklar ob identisch zu `-evu_leistung` bei negativem Vorzeichen |
| `sensor.gesamtleistung_haushalt` | Gesamtleistung Haushalt | 193,7 | W | power | Kandidat für „Hausverbrauch aktuell“ |
| `sensor.energy_import_daily`, `_monthly`, `_sum` | Energy Import … | 0 / 0 / 19600,03 | kWh | energy | Netzbezug, Tages-/Monats-/Summenwert |
| `sensor.energy_export_daily`, `_monthly`, `_sum` | Energy Export … | 0 / 0 / 11965,01 | kWh | energy | Einspeisung, Tages-/Monats-/Summenwert |
| `sensor.energy_import_daily_cost` | – | `unavailable` | EUR | monetary | vorgesehener Kostensensor, aktuell nicht verfügbar |
| `sensor.energy_export_daily_compensation` | – | `unavailable` | EUR | monetary | vorgesehener Vergütungssensor, aktuell nicht verfügbar |

## 8. Batteriespeicher (Marstek)

| Entity-ID | Friendly Name | State | Unit | Device class | Bemerkung |
|---|---|---|---|---|---|
| `sensor.marstek_venus_modbus_soc_batterie` | Marstek Venus Modbus SoC Batterie | `unavailable` | % | battery | aktuell ausgefallen |
| `binary_sensor.marstek_venus_modbus_modbus_verbindung` | … Modbus-Verbindung | `off` | – | connectivity | Verbindung derzeit getrennt |
| `sensor.marstek_venus_modbus_batterieleistung` | … Batterieleistung | `unavailable` | W | power | Lade-/Entladeleistung (Vorzeichen vermutlich +/-) |
| `sensor.marstek_venus_modbus_gespeicherte_energie` | … Gespeicherte Energie | `unknown` | kWh | energy | – |
| weitere `marstek_venus_modbus_*` (Spannung, Strom, Zyklen, Effizienz, Firmware) | – | überwiegend `unavailable`/`unknown` | – | – | Diagnosewerte, für Regelung nicht benötigt |

## 9. Wetter / Sonne

| Entity-ID | Domain | Bemerkung |
|---|---|---|
| `sun.sun` | sun | Standard-Sonnenstand, in v1_8 nicht verwendet |
| 2× `weather.*` | weather | in v1_8 nicht verwendet |

## 10. Notify / Pushover

| Entity/Service | Typ | Bemerkung |
|---|---|---|
| `notify.pushover` | Service (kein State-Entity, daher nicht in CSV) | **Direkt referenziert in `warmwasser_v1_8_full_package (2).yaml`**, Zeilen 1002 und 1029 (`service: notify.pushover`). Das ist der stärkste verfügbare Beleg für den produktiv genutzten Pushover-Service. |
| `notify.tibber` | notify (Entity vorhanden, State `unknown`) | eigenständiger Tibber-App-Notify-Ziel, nicht Pushover |
| `notify.andreas_iphone`, `notify.iphone_von_andreas`, `notify.andreas_iphone_14`, `notify.andreas_iphone_home`, `notify.iphone8_von_andreas`, `notify.iphone_von_nadine`, `notify.ipad`, `notify.andreass_ipad`, `notify.firetab`, `notify.kftrwi`, `notify.kftrwi_2`, `notify.sm_x216b` | notify (Mobile-App-Ziele) | keine Pushover-Ziele, sondern HA-Companion-App-Geräte |
| `script.ww_debug_notify` | script | „WW Debug - Pushover senden“, State `unavailable` – vermutlich Vorgänger-Implementierung vor v1_8, nicht mehr in Verwendung |
| `input_boolean.ww_debug` | input_boolean | „WW Debug (Pushover)“, State `off` – vermutlich Vorgänger-Debug-Schalter, nicht Teil von v1_8 |

→ **Empfehlung:** `notify.pushover` als Zielservice für ww_v2 verwenden (Beleg aus
produktivem Code), aber siehe Rückfrage 8 zur expliziten Bestätigung.

## 11. Statistik / Utility Meter

- Kein `utility_meter:` in v1_8 gefunden.
- Eine `platform: statistics`-Sensor-Definition in v1_8:
  `sensor.ww_v1_8_temperatur_maximum_7_tage` (max_age 7 Tage, Quelle
  `sensor.ww_v1_8_warmwassertemperatur`) – wird für die Legionellen-/Hygienefunktion
  benötigt und in v2 übernommen.
- Eine `platform: integration`-Sensor-Definition:
  `sensor.ww_v1_8_wp_energie_gesamt` (Riemann-Summe aus `sensor.wp_gesamtleistung`) –
  möglicherweise durch den direkten Zähler `sensor.warmepumpe_device_energy` ersetzbar
  (Rückfrage 2).

## 12. Kosten (bereits in v1_8 vorhanden)

`input_number.ww_v18_kosten_heute`, `ww_v18_einsparung_heute`,
`ww_v18_pv_energie_heute`, `ww_v18_netz_energie_heute` – tagesbasierte, im Package
selbst mitgeführte Kostenwerte (kein `utility_meter`, keine `recorder`-Statistik).
Werden fachlich für v2 übernommen; Wochen-/Monatswerte fehlen bisher komplett und
müssten neu (z. B. über `utility_meter` oder Statistics-Sensor) ergänzt werden.

## 13. Helfer, die in v1_8 referenziert, aber im Dashboard fälschlich unter anderem Namen erwartet werden

Das Dashboard (`Diagnose`-View) referenziert:

- `input_text.ww_v18_tibber_leading24_entity`
- `input_text.ww_v18_tibber_next_best_start_entity`

Diese beiden Helfer sind **nicht** im Package definiert (das Package definiert nur
`input_text.ww_v18_wp_energy_entity`, `ww_v18_letzter_grund`,
`ww_v18_aktive_strategie`). Das Dashboard referenziert damit zwei nicht existierende
Entities – ein konkreter Fehler in v1_8 (Dashboard und Package sind inkonsistent).

## 14. Sonstiges / nicht eindeutig zuordenbar (offene Kategorie)

- `sensor.home_geschatzter_jahresverbrauch`, `sensor.home_netzbetreiber`,
  `sensor.home_abonnementstatus` u. Ä. – informative Tibber-Metadaten, für Regelung
  nicht relevant.
- `sensor.home_volatilitat_heute`, `sensor.home_preismuster_heute`,
  `sensor.home_aktueller_preistrend` – optionale Zusatzinformationen für erweiterte
  Preisstrategien, aktuell nicht in v1_8 verwendet, nicht zwingend für v2 nötig.
