# ww_v2 – Installationsanleitung und Rollback

## 1. Voraussetzungen prüfen

Vor der Installation in Home Assistant Developer Tools → Zustände folgende
Entities aufrufen und bestätigen, dass sie existieren und plausible Werte
liefern (siehe `ww_v2_entity_catalog.md` für Details):

- `water_heater.dhw1` (Attribut `current_temperature` muss einen Zahlenwert
  liefern, z. B. wie bestätigt 52.2)
- `switch.charge`, `number.charge_setpoint`, `number.charge_duration`
- `sensor.stp10_0_3av_40_040_pv_power_a` / `_pv_power_b`
- `sensor.energy_production_today[_2]`, `_today_remaining[_2]`, `_tomorrow[_2]`, `_next_hour[_2]`
- `sensor.evu_leistung`, `sensor.gesamtleistung_haushalt`, `sensor.wp_gesamtleistung`
- `sensor.home_aktueller_strompreis`, `sensor.home_preis_vorlaufend_24h`,
  `sensor.home_bestpreis_startet[_in]`, `binary_sensor.home_bestpreis_zeitraum`
- Pushover-Notify-Service `notify.pushover` (Einstellungen → Geräte & Dienste)

Falls eine dieser Entities aktuell nicht existiert oder umbenannt wurde,
**vor** der Aktivierung `ww_v2_package.yaml` entsprechend anpassen (nicht
blind aktivieren).

## 2. Ablageort

```
<config>/packages/ww_v2_package.yaml
<config>/packages/ww_v2_dashboard.yaml   (optional, falls Dashboards als YAML verwaltet werden)
```

Falls kein `packages`-Ordner existiert, diesen anlegen. Die Datei
`ww_v2_package.yaml` ist in sich geschlossen (keine weiteren `!include`
nötig).

## 3. Einbindung in `configuration.yaml`

Falls `packages:` noch nicht aktiviert ist:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Falls bereits ein anderer Mechanismus verwendet wird (z. B. einzelne
`!include`-Zeilen), `ww_v2_package.yaml` entsprechend dort ergänzen. **Die
v1_8-Datei bleibt unverändert und wird nicht referenziert** – v1_8 und v2
können parallel in `packages/` liegen, ohne sich zu stören, solange v1_8
weiterhin über ihren eigenen `ww_v18_*`-Namensraum läuft (siehe Abschnitt 6
„Deaktivierung von v1_8“ für den empfohlenen Ablauf).

## 4. Dashboard-Einbindung

Zwei Optionen:

**Option A – als eigenes YAML-verwaltetes Dashboard:**
In `configuration.yaml`:
```yaml
lovelace:
  dashboards:
    ww-v2:
      mode: yaml
      title: Warmwasser V2
      icon: mdi:water-boiler
      filename: packages/ww_v2_dashboard.yaml
```

**Option B – Inhalt manuell in ein bestehendes Dashboard übernehmen:**
Die fünf `views:` aus `ww_v2_dashboard.yaml` per UI-Editor (Raw-Konfiguration
bearbeiten) in ein vorhandenes Dashboard kopieren.

## 5. Benötigte Reloads / Neustart

Nach dem Ablegen der Datei(en):

1. **Konfiguration prüfen**: Einstellungen → System → Steuerung →
   „Konfiguration überprüfen“ (entspricht `ha core check_config`). Dieser
   Schritt konnte in der vorliegenden Umgebung **nicht** ausgeführt werden
   (keine laufende Home-Assistant-Instanz verfügbar) – er ist daher **vor
   jeder Aktivierung zwingend manuell durchzuführen**.
2. Bei erfolgreicher Prüfung: „YAML-Konfiguration neu laden“ →
   Helfer, Automationen, Skripte (oder vollständiger Neustart, falls neue
   `sensor:`-Plattformen wie `integration`/`statistics`/`utility_meter` zum
   ersten Mal geladen werden – diese benötigen oft einen vollständigen
   Neustart statt nur „Neu laden“).
3. Dashboard: bei YAML-Modus wird es automatisch geladen; bei manueller
   Übernahme UI-Dashboard speichern.

## 6. Kontrollierte Erstaktivierung (empfohlener Ablauf)

1. `input_boolean.ww_v2_aktiv` **zunächst auf `off` lassen** (im YAML ist
   `initial: true` gesetzt – vor dem ersten Neustart im Code auf `false`
   ändern, falls ein rein passiver erster Test gewünscht ist, oder direkt
   nach dem Start über das Dashboard ausschalten).
2. `input_boolean.ww_v2_debug_mode` auf `on` setzen.
3. Über `input_button.ww_v2_debug_test` eine Pushover-Testnachricht auslösen
   und Empfang prüfen (Test-Szenario 26/27 im Testplan).
4. Debug-Seite im Dashboard beobachten: `sensor.ww_v2_diagnose_pflichtentities`
   sollte „0 nicht verfuegbar“ anzeigen. Falls nicht: Liste der fehlenden
   Entities prüfen und beheben, bevor die Automatik aktiviert wird.
5. `sensor.ww_v2_warmwassertemperatur`, `sensor.ww_v2_pv_leistung_ost/west`,
   `sensor.ww_v2_effektives_ziel` auf Plausibilität prüfen.
6. Erst danach `input_boolean.ww_v2_aktiv` auf `on` setzen.
7. Debug-Modus für mindestens einen vollen Tag (idealerweise inkl. einer
   PV-Phase und einer Tibber-Bestpreis-Phase) aktiv lassen und die
   Pushover-Meldungen sowie die Debug-Seite (Aktionen/Entscheidungen/
   Diagnose) beobachten.
8. Danach `input_boolean.ww_v2_debug_mode` nach Bedarf wieder auf `off`
   setzen (die Regelung selbst läuft unabhängig vom Debug-Modus weiter).

## 7. Deaktivierung von v1_8

Da v1_8 im Live-System aktuell bereits inaktiv ist (alle `ww_v1_8_*`-Entities
`unavailable`, siehe `ww_v2_analysis.md`), ist vermutlich keine gesonderte
Deaktivierung mehr nötig. Falls v1_8 doch wieder geladen werden sollte
(z. B. nach Behebung der Ursache):

1. `input_boolean.ww_v18_aktiv` auf `off` setzen (stoppt die v1_8-Regellogik,
   ohne die Datei zu entfernen).
2. `warmwasser_v1_8_full_package (2).yaml` NICHT löschen, sondern optional
   aus dem `packages`-Include-Pfad herausnehmen (umbenennen auf z. B.
   `.disabled`-Endung oder in einen nicht eingebundenen Unterordner
   verschieben), damit v1_8 und v2 nicht gleichzeitig auf `switch.charge`
   zugreifen.
3. Konfiguration erneut prüfen und neu laden.

## 8. Rollback auf v1_8

1. `input_boolean.ww_v2_aktiv` auf `off` setzen (stoppt v2 sofort, ohne
   Dateien zu entfernen).
2. Falls v1_8 aus Schritt 7 deaktiviert wurde: die dortigen Schritte
   rückgängig machen (Datei wieder einbinden, `ww_v18_aktiv` auf `on`).
3. Konfiguration prüfen, neu laden/neu starten.
4. `ww_v2_package.yaml` und `ww_v2_dashboard.yaml` können bei Bedarf
   vollständig aus dem `packages`-Ordner entfernt werden (siehe Abschnitt 9).

## 9. Vollständige Entfernung von v2

1. `input_boolean.ww_v2_aktiv` auf `off`.
2. `packages/ww_v2_package.yaml` und ggf. `packages/ww_v2_dashboard.yaml`
   löschen bzw. aus dem Include-Pfad entfernen.
3. Dashboard-Eintrag aus `configuration.yaml` (`lovelace: dashboards:`)
   entfernen, falls Option A verwendet wurde.
4. Konfiguration prüfen, neu laden/neu starten.
5. Alle `ww_v2_*`-Entities werden danach `unavailable` und verschwinden nach
   einiger Zeit aus der Entity-Registry (oder können manuell über
   Einstellungen → Geräte & Dienste → Entities gelöscht werden).

## 10. Umgang mit Recorder- und Historiedaten

- Es werden keine bestehenden Recorder-Einstellungen durch dieses Package
  verändert (bewusst kein eigener `recorder:`-Block, um Konflikte mit
  bestehender Konfiguration zu vermeiden).
- Empfehlung: Die häufig aktualisierten internen Roh-Helfer
  (`input_text.ww_v2_status_*`, `input_text.ww_v2_letzte_debug_nachricht`)
  in der bestehenden `recorder:`-Konfiguration des Anwenders von der
  Historie ausschliessen, falls die Recorder-Datenbank spürbar wächst:

```yaml
recorder:
  exclude:
    entities:
      - input_text.ww_v2_status_action
      - input_text.ww_v2_status_reason
      - input_text.ww_v2_status_result
      - input_text.ww_v2_status_error
      - input_text.ww_v2_status_pushover
      - input_text.ww_v2_letzte_debug_nachricht
```

  Diese Zeilen müssen in die **bestehende** `recorder:`-Konfiguration des
  Anwenders eingefügt werden (nicht als zweiter, separater `recorder:`-Block
  im Package, da mehrere `recorder:`-Schlüssel über Packages nicht
  zuverlässig zusammengeführt werden).
- Historie von v1_8 (`sensor.ww_v1_8_*` usw.) bleibt in der Datenbank
  erhalten und wird durch v2 nicht berührt oder gelöscht.
- `utility_meter.ww_v2_wp_energie_woche` / `_monat` benötigen wie jeder
  `utility_meter` eine funktionierende Recorder-Historie der Quelle
  (`sensor.ww_v2_wp_energie_gesamt`), um nach einem Neustart korrekt
  fortzusetzen – Standardverhalten von Home Assistant, keine zusätzliche
  Konfiguration nötig.

## 11. Bekannte offene Punkte vor Produktivbetrieb

Siehe `ww_v2_open_questions.md` für die verbliebenen kleineren, nicht
blockierenden Punkte (z. B. Umgang mit veralteter, aber numerisch gültiger
Temperatur) sowie `ww_v2_quality_report.md` für die dokumentierten
technischen Einschränkungen (z. B. keine echte Zustellbestätigung von
Pushover-Nachrichten).
