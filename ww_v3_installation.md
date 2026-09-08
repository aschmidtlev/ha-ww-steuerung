# ww_v3 – Installationsanleitung und Rollback

## 1. Voraussetzungen prüfen

Vor der Installation in Home Assistant Developer Tools → Zustände folgende
Entities aufrufen und bestätigen, dass sie existieren und plausible Werte
liefern (siehe `ww_v3_analysis.md` für Details):

**EMS-ESP-Gateway (neu, Pflicht):**
- `sensor.boiler_dhw_curtemp` (numerischer Wert, °C)
- `switch.boiler_dhw_onetime`, `number.boiler_dhw_seltempsingle`,
  `number.boiler_dhw_maxtemp`
- `binary_sensor.boiler_dhw_charging`, `binary_sensor.boiler_dhw_recharging`,
  `binary_sensor.boiler_dhw_tempok`
- `switch.thermostat_pvenabledhw`
- Optional zur Diagnose: `sensor.boiler_dhw_curtemp2`,
  `binary_sensor.boiler_dhw_3wayvalve`, `select.thermostat_dhw_mode`,
  `switch.boiler_dhw_dhwprio`, `select.boiler_dhw_comfort1`,
  `select.boiler_dhw_circmode`

**Unverändert aus ww_v2 (weiterhin Pflicht, kein EMS-ESP-Bezug):**
- `sensor.stp10_0_3av_40_040_pv_power_a` / `_pv_power_b`
- `sensor.energy_production_today[_2]`, `_today_remaining[_2]`,
  `_tomorrow[_2]`, `_next_hour[_2]`
- `sensor.evu_leistung`, `sensor.gesamtleistung_haushalt`,
  `sensor.wp_gesamtleistung`
- `sensor.home_aktueller_strompreis`, `sensor.home_preis_vorlaufend_24h`,
  `sensor.home_bestpreis_startet[_in]`, `binary_sensor.home_bestpreis_zeitraum`
- Pushover-Notify-Service `notify.pushover`

Falls eine dieser Entities aktuell nicht existiert oder umbenannt wurde,
**vor** der Aktivierung `ww_v3_package.yaml` entsprechend anpassen (nicht
blind aktivieren).

**Zusaetzlich vor Aktivierung des optionalen Hygienezyklus pruefen:**
`input_number.ww_v3_hygiene_ziel` (Standard 60 °C) darf `number.boiler_dhw_
maxtemp` (im CSV-Snapshot 56 °C) nicht ueberschreiten, sonst wird das Ziel
real nie erreicht (siehe `ww_v3_testplan.md` Szenario 38). Entweder
`ww_v3_hygiene_ziel` absenken oder `number.boiler_dhw_maxtemp` am Geraet
anheben (fachlich pruefen lassen, siehe Sicherheitshinweis im Dateikopf von
`ww_v3_package.yaml`).

## 2. Ablageort

```
<config>/packages/ww_v3_package.yaml
<config>/packages/ww_v3_dashboard.yaml   (optional, falls Dashboards als YAML verwaltet werden)
```

`ww_v3_package.yaml` ist in sich geschlossen (keine weiteren `!include`
nötig).

## 3. Einbindung in `configuration.yaml`

Identisch zu `ww_v2_installation.md` Abschnitt 3 (`homeassistant: packages:
!include_dir_named packages`).

## 4. Dashboard-Einbindung

Identisch zu `ww_v2_installation.md` Abschnitt 4, mit `packages/ww_v3_
dashboard.yaml` statt `ww_v2_dashboard.yaml` und eigenem Dashboard-Key
(z. B. `ww-v3` statt `ww-v2`), damit beide Dashboards parallel existieren
koennen.

## 5. Benoetigte Reloads / Neustart

Identisch zu `ww_v2_installation.md` Abschnitt 5. **"Konfiguration
überprüfen" ist vor jeder Aktivierung zwingend manuell durchzufuehren** –
konnte in dieser Umgebung nicht ausgefuehrt werden.

## 6. Kontrollierte Erstaktivierung (empfohlener Ablauf)

1. **Vor dem ersten Neuladen sicherstellen, dass `ww_v2_package.yaml` nicht
   mehr aktiv ist** (siehe Abschnitt 7 – v2 und v3 duerfen wegen des
   gemeinsamen physischen Warmwasserspeichers nicht gleichzeitig aktiv sein,
   sobald sowohl Bosch Home Connect als auch EMS-ESP parallel Zugriff auf
   das Geraet haben).
2. `input_boolean.ww_v3_aktiv` zunächst auf `off` lassen (im YAML ist
   `initial: true` gesetzt – vor dem ersten Neustart im Code auf `false`
   ändern, falls ein rein passiver erster Test gewünscht ist, oder direkt
   nach dem Start über das Dashboard ausschalten).
3. `input_boolean.ww_v3_debug_mode` auf `on` setzen.
4. Über `input_button.ww_v3_debug_test` eine Pushover-Testnachricht auslösen
   und Empfang prüfen.
5. Debug-Seite im Dashboard beobachten: `sensor.ww_v3_diagnose_
   pflichtentities` sollte „0 nicht verfuegbar“ anzeigen. Falls nicht: Liste
   der fehlenden Entities prüfen und beheben, bevor die Automatik aktiviert
   wird.
6. `sensor.ww_v3_warmwassertemperatur` (Wert sollte plausibel zu
   `sensor.boiler_dhw_curtemp` am Geraet passen), `sensor.ww_v3_pv_leistung_
   ost/west`, `sensor.ww_v3_effektives_ziel`, `number.boiler_dhw_maxtemp`
   auf Plausibilität prüfen.
7. Pruefen, dass `switch.thermostat_pvenabledhw` nach kurzer Zeit auf `off`
   steht (durch `automation.ww_v3_pvenabledhw_sperren`), sofern
   `ww_v3_aktiv` bereits auf `on` gesetzt wurde.
8. Erst danach `input_boolean.ww_v3_aktiv` auf `on` setzen.
9. Bei einer manuellen Testladung beobachten, ob `binary_sensor.ww_v3_
   ladung_aktiv_hardware` innerhalb weniger Minuten nach `binary_sensor.
   ww_v3_ladung_aktiv` (Befehl) ebenfalls auf `on` wechselt – falls nicht,
   siehe `ww_v3_testplan.md` Szenario 39.
10. Debug-Modus für mindestens einen vollen Tag (idealerweise inkl. einer
    PV-Phase und einer Tibber-Bestpreis-Phase) aktiv lassen und die
    Pushover-Meldungen sowie die Debug-Seite beobachten.
11. Danach `input_boolean.ww_v3_debug_mode` nach Bedarf wieder auf `off`
    setzen (die Regelung selbst läuft unabhängig vom Debug-Modus weiter).

## 7. Umstellung von ww_v2 auf ww_v3

Anders als beim Uebergang v1_8 → v2 (dort war v1_8 im Live-System bereits
inaktiv) muss hier aktiv gehandelt werden, da ww_v2 vermutlich noch laeuft:

1. `input_boolean.ww_v2_aktiv` auf `off` setzen (stoppt die v2-Regellogik,
   ohne die Datei zu entfernen).
2. `ww_v2_package.yaml` aus dem `packages`-Include-Pfad herausnehmen
   (umbenennen auf z. B. `.disabled`-Endung oder in einen nicht
   eingebundenen Unterordner verschieben) – **wichtig**, da sowohl
   `switch.charge` (Bosch Cloud) als auch `switch.boiler_dhw_onetime`
   (EMS-ESP) denselben physischen Warmwasserspeicher schalten und ein
   gleichzeitiger Betrieb beider Packages zu widerspruechlichen
   Ladebefehlen fuehren koennte.
3. Konfiguration prüfen, neu laden.
4. `ww_v3_package.yaml`/`ww_v3_dashboard.yaml` wie in Abschnitt 6
   beschrieben aktivieren.

## 8. Rollback auf ww_v2

1. `input_boolean.ww_v3_aktiv` auf `off` setzen (stoppt v3 sofort, ohne
   Dateien zu entfernen).
2. Schritte aus Abschnitt 7 rueckgaengig machen: `ww_v2_package.yaml`
   wieder einbinden, `input_boolean.ww_v2_aktiv` auf `on`.
3. `ww_v3_package.yaml`/`ww_v3_dashboard.yaml` aus dem Include-Pfad nehmen
   (analog zu Schritt 7.2), um denselben Doppelsteuerungs-Konflikt in die
   andere Richtung zu vermeiden.
4. Konfiguration prüfen, neu laden/neu starten.

## 9. Vollständige Entfernung von ww_v3

Identisch zu `ww_v2_installation.md` Abschnitt 9, mit `ww_v3_*`-Dateien und
-Entities statt `ww_v2_*`.

## 10. Umgang mit Recorder- und Historiedaten

Identisch zu `ww_v2_installation.md` Abschnitt 10, mit `ww_v3_status_*`
statt `ww_v2_status_*` in der empfohlenen Recorder-Ausschlussliste:

```yaml
recorder:
  exclude:
    entities:
      - input_text.ww_v3_status_action
      - input_text.ww_v3_status_reason
      - input_text.ww_v3_status_result
      - input_text.ww_v3_status_error
      - input_text.ww_v3_status_pushover
      - input_text.ww_v3_letzte_debug_nachricht
```

## 11. Bekannte offene Punkte vor Produktivbetrieb

Siehe `ww_v3_open_questions.md` für die verbliebenen, nicht blockierenden
Punkte (u. a. Wahl `sensor.boiler_dhw_curtemp` vs. `_curtemp2`, Verhalten von
`switch.boiler_dhw_onetime` nach Ladeabschluss) sowie `ww_v3_quality_
report.md` für dokumentierte technische Einschränkungen.
