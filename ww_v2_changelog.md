# ww_v2 – Changelog gegenüber v1_8

## Hotfix nach erster Live-Prüfung durch den Anwender

- **Behoben**: `sensor.ww_v2_warmwassertemperatur` wurde fälschlich
  `unavailable` (Lovelace-Gauge zeigte „Entität ist nicht-numerisch“), weil
  die `availability`-Prüfung mit dem Jinja-Test `is number` arbeitete. Dieser
  Test ist nur für native Python-Zahlentypen `true` – liefert die zugrunde
  liegende Integration `current_temperature` als numerischen **String**
  (z. B. `"52.2"` statt `52.2`), schlägt `is number` fehl, obwohl der Wert
  gültig und umwandelbar ist. Ersetzt durch `(t | float(none)) is not none`,
  was sowohl native Zahlen als auch numerische Strings korrekt als gültig
  erkennt, aber `none`/nicht-umwandelbare Werte weiterhin zuverlässig
  ablehnt (keine Aufweichung der Sicherheitsregel „kein Fantasiewert bei
  ungültigen Daten“).

## Sicherheit / Korrektheit

- **Entfernt**: Fallback auf `sensor.hotwater_temp` (Bosch-Entity, laut
  Vorgabe verboten) und `sensor.dhw1_temperature` (existiert im System
  nicht) für die Warmwasser-Ist-Temperatur. `sensor.ww_v2_warmwassertemperatur`
  nutzt jetzt ausschliesslich `water_heater.dhw1`/`current_temperature` und
  wird bei ungültigen Daten `unavailable` statt einen Fantasiewert zu liefern.
- **Neu**: `binary_sensor.ww_v2_pflichtdaten_gueltig` als zentrale
  Vorprüfung (Priorität 1) in allen ladeauslösenden Automationen.
- **Neu**: `script.ww_v2_ladung_starten` bricht jetzt explizit und mit
  Protokollierung ab, wenn `switch.charge` nicht verfügbar ist oder die
  Temperatur ungültig ist, statt stillschweigend HA-interne Fehler zu
  erzeugen.
- **Neu**: `automation.ww_v2_sicherheitsabschaltung_maxdauer` schliesst die
  in `ww_v2_analysis.md` dokumentierte Lücke, dass eine Ladung nach einem
  Neustart während eines `wait_template` (Hygienezyklus) theoretisch
  unbegrenzt hätte weiterlaufen können.
- **Entfernt**: Prüfung auf `state_attr('number.charge_setpoint','max')` im
  Hygienezyklus (Attribut in der vorliegenden CSV nicht verifizierbar).
  Ersetzt durch die ohnehin bestehende `input_number`-Begrenzung
  (60–65 °C) von `ww_v2_hygiene_ziel`.

## PV Ost/West (neu)

- **Neu**: `sensor.ww_v2_pv_leistung_ost` / `_west` / `_gesamt` bilden erstmals
  die aktuelle Momentanleistung beider PV-Strings ab (vorher nur Forecast,
  keine Ist-Leistung je Anlage).
- **Neu**: `sensor.ww_v2_pv_forecast_rest_heute_gesamt`,
  `_morgen_gesamt`, `_naechste_stunde_gesamt` fassen die Forecast.Solar-
  Sensoren beider Anlagen korrekt zusammen (v1_8 nutzte für die
  Zielanpassung ausschliesslich die Prognose der Anlage 1/Ost, Anlage 2/West
  floss nirgends ein). Beide vorgefundenen externen Summen-Sensoren
  (`sensor.pv_forecast_heute_gesamt`/`_morgen_gesamt`) werden laut
  Nutzerangabe nicht mehr verwendet, da sie fehlerhaft sind.
- **Neu**: Ausfall einer einzelnen PV-Anlage wird erkannt
  (`binary_sensor.ww_v2_pv_ost_verfuegbar`/`_west_verfuegbar`) und die
  Regelung arbeitet automatisch mit der verbleibenden Anlage weiter, statt
  bei Teilausfall komplett auszufallen.

## Netz / Kosten

- **Neu**: `sensor.ww_v2_netzbezug_watt`, `_netzeinspeisung_watt`,
  `_pv_ueberschuss_watt` als vorzeichenkorrekt aufgeschlüsselte, eigenständig
  anzeigbare Momentanwerte (vorher nur ein einzelner Rohwert
  `sensor.evu_leistung` ohne Aufschlüsselung im Dashboard).
- **Neu**: `utility_meter.ww_v2_wp_energie_woche` / `_monat` sowie
  `sensor.ww_v2_kosten_woche_geschaetzt` / `_monat_geschaetzt` – v1_8 kannte
  nur Tageswerte.
- Alle Kostenwerte im Dashboard sind jetzt explizit als „gemessen“,
  „berechnet“ oder „geschätzt“ gekennzeichnet.

## Tibber / Bestpreis

- **Ersetzt**: Die fragile Attribut-Ratekette in
  `sensor.ww_v1_8_naechster_bestpreis_start` (bis zu 5 mögliche
  Attributnamen plus zwei nicht existierende Fallback-Sensoren) durch die
  direkten, bestätigt vorhandenen Sensoren `sensor.home_bestpreis_startet`
  und `sensor.home_bestpreis_startet_in`.

## Debug / Pushover (grundlegend überarbeitet)

- **Neu**: zentrales `script.ww_v2_debug_log` statt zweier separater
  Automationen mit direktem `notify.pushover`-Aufruf. Wird jetzt von jedem
  relevanten Aktionspunkt aufgerufen (Ladung starten/stoppen, Hygienezyklus,
  Sicherheitsabschaltung, Kostenabschluss, Blockaden, Sensorausfälle,
  Strategiewechsel).
- **Neu**: Deduplizierung identischer Meldungen mit konfigurierbarem
  Mindestabstand (`input_number.ww_v2_pushover_mindestabstand_min`).
- **Neu**: `continue_on_error: true` beim Pushover-Versand – ein
  fehlgeschlagener Versand blockiert die Regelung nicht (vorher implizit
  durch HA-Standardverhalten mehr oder weniger gegeben, jetzt explizit und
  dokumentiert abgesichert).
- **Neu**: eigenständiges `script.ww_v2_pushover_test`, unabhängig vom
  Debug-Modus und von der Deduplizierung, für gezielte Funktionstests.
- **Neu**: `counter.ww_v2_aktionen_gesamt`, `_blockierte_aktionen`,
  `_fehler_gesamt` sowie `sensor.ww_v2_last_action/_reason/_result/_error/
  _pushover_status` für eine kompakte, dashboard-taugliche Statusübersicht.
- **Neu**: `automation.ww_v2_regelungsdurchlauf_heartbeat` protokolliert
  erstmals „letzter erfolgreicher/übersprungener Regelungsdurchlauf“ inkl.
  Grund.

## Entfallen

- **Urlaubsmodus**: nicht übernommen (auf Nutzerwunsch; die einzige im
  System vorhandene Implementierung war Bosch-gebunden und daher ohnehin
  nicht nutzbar).
- **`input_text.ww_v18_tibber_leading24_entity` /
  `_tibber_next_best_start_entity`**: ersatzlos entfernt – diese beiden
  Helfer wurden im v1_8-Dashboard angezeigt, aber im v1_8-Package nie
  definiert (Fehler in v1_8) und sind durch die feste Referenzierung der
  Tibber-Sensoren in v2 nicht mehr nötig.

## Unverändert übernommen

- Anti-Takt-Sperre (20 min), Mindestlaufzeit (30 min), Start-Hysterese
  (2 °C), Lernlogik für Nachtverlust und Verbrauch morgens/abends,
  Marstek-Integration (3-Stufen-Logik), optionaler Hygienezyklus
  (standardmässig deaktiviert, gleicher Sicherheitshinweis), Tagesreset,
  WP-Energiezähler per Riemann-Integration aus `sensor.wp_gesamtleistung`.
