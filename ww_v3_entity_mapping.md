# ww_v3 – Entity-Mapping (v2 → v3)

Legende: **übernommen** = gleiche externe Quelle, nur Namensraum-Umbenennung;
**ersetzt** = Bosch-Cloud-Entity durch EMS-ESP-Entity ersetzt; **neu** = ohne
v2-Vorbild; **entfernt** = kein Ersatz mehr referenziert.

## Kern-Migration: Bosch Cloud → EMS-ESP

| Alte Referenz (v2) | Neue Referenz (v3) | Funktion | Ergebnis | Begründung |
|---|---|---|---|---|
| `water_heater.dhw1` (Attribut `current_temperature`) | `sensor.boiler_dhw_curtemp` | Ist-Temperatur-Quelle | **ersetzt** | Anwenderentscheidung (Frage 2): EMS-ESP wird alleinige Quelle. Numerischer Sensor statt String-Attribut, lokal statt Cloud |
| `switch.charge` | `switch.boiler_dhw_onetime` | Ladeauslöser | **ersetzt** | Anwenderentscheidung (Frage 3): boiler-seitiger One-Time-Charge-Schalter statt thermostat-seitigem `switch.thermostat_dhw_charge` |
| `number.charge_setpoint` | `number.boiler_dhw_seltempsingle` | Zieltemperatur für Ladung | **ersetzt** | Identischer Startwert (51 °C) im Snapshot, direkt zu `switch.boiler_dhw_onetime` gehoerig |
| `number.charge_duration` | – | Ladedauer-Begrenzung am Geraet | **entfernt** | Kein passendes EMS-ESP-Aequivalent fuer `switch.boiler_dhw_onetime`; ersetzt durch die zwei package-eigenen Sicherheitsmechanismen (siehe `ww_v3_migration_concept.md` 3.3) |
| – | `number.boiler_dhw_maxtemp` | Geraete-Maximaltemperatur (Sicherheitsgrenze) | **neu** | War in v2 unverifizierbar, jetzt aktiv als Kappungsgrenze genutzt |
| – | `sensor.boiler_dhw_curtemp2` | zweiter Temperaturfuehler (Diagnose) | **neu** | Nur Anzeige/Vergleich, keine Regelfunktion (Offene Frage 1) |
| – | `binary_sensor.boiler_dhw_charging` / `_recharging` | Hardware-Ladebestaetigung | **neu** | Unabhaengiges Signal, in v2 technisch nicht verfuegbar |
| – | `binary_sensor.boiler_dhw_tempok` | Geraete-eigenes Temperatur-OK-Flag | **neu** | Diagnose-Attribut |
| – | `switch.thermostat_pvenabledhw` | natives PV-Aufladen | **neu, wird gesperrt** | Anwenderentscheidung (Frage 4): ww_v3 haelt es deaktiviert, um Doppelsteuerung zu vermeiden |

## Unveraendert referenzierte externe Entities (kein EMS-ESP-Bezug)

Alle uebrigen externen Referenzen aus `ww_v2_entity_mapping.md` (Abschnitt
"Externe (fremde) Quell-Entities") bleiben **identisch**, da sie nicht Teil
des Heizungs-/Warmwassersystems sind: `sensor.wp_gesamtleistung`,
`sensor.home_aktueller_strompreis`, `sensor.home_preis_vorlaufend_24h`,
`binary_sensor.home_bestpreis_zeitraum`, `sensor.home_bestpreis_startet[_in]`,
`sensor.energy_production_*[_2]`, `sensor.stp10_..._pv_power_a/_b`,
`sensor.evu_leistung`, `sensor.gesamtleistung_haushalt`,
`sensor.marstek_venus_modbus_soc_batterie`, `notify.pushover`.

## Package-eigene Entities (v2 → v3, reine Umbenennung `ww_v2_` → `ww_v3_`)

Alle in `ww_v2_entity_mapping.md` (Abschnitt "Package-eigene Entities")
gelisteten Entities wurden 1:1 unter dem neuen Praefix `ww_v3_*` uebernommen,
ohne Verhaltensaenderung. Beispiele: `input_boolean.ww_v3_aktiv`,
`sensor.ww_v3_wp_energie_gesamt`, `automation.ww_v3_stop_ziel`,
`script.ww_v3_debug_log` usw. Vollstaendige Liste siehe `ww_v3_package.yaml`.

## Neue Entities ohne v2-Vorbild

| Neue Entity | Funktion |
|---|---|
| `binary_sensor.ww_v3_ladung_aktiv_hardware` | Hardware-bestaetigte Ladung (aus `binary_sensor.boiler_dhw_charging`/`_recharging`), unabhaengig vom eigenen Schalterbefehl |
| `binary_sensor.ww_v3_hygieneziel_ueber_geraetemaximum` | Warnt, wenn `input_number.ww_v3_hygiene_ziel` ueber `number.boiler_dhw_maxtemp` liegt |
| `automation.ww_v3_debug_ladung_ohne_hardware_reaktion` | Erkennt Befehl-ohne-Wirkung nach 10 Minuten |
| `automation.ww_v3_pvenabledhw_sperren` | Haelt `switch.thermostat_pvenabledhw` deaktiviert, solange `input_boolean.ww_v3_aktiv` an ist |

## Entfallene Funktionen (kein v3-Äquivalent)

| Funktion | Grund |
|---|---|
| Geraeteseitige Ladedauer-Begrenzung (`number.charge_duration`) | Kein passendes EMS-ESP-Aequivalent fuer den gewaehlten Schalter; durch package-eigene Sicherheitsmechanismen ersetzt (siehe oben) |

Alle uebrigen v1_8/v2-Funktionsentscheidungen (Urlaubsmodus entfaellt usw.)
bleiben unveraendert – siehe `ww_v2_entity_mapping.md`.
