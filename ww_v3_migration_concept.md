# ww_v3 – Migrationskonzept

Dieses Dokument beschreibt die Strategie fuer die Migration von ww_v2
(Bosch-Cloud-Anbindung) auf ww_v3 (EMS-ESP-Gateway, lokal). Details/Belege
siehe `ww_v3_analysis.md`; die vollstaendige Entity-fuer-Entity-Zuordnung
steht in `ww_v3_entity_mapping.md`.

## 1. Leitprinzip

> Das EMS-ESP-Gateway wird die alleinige Quelle der Wahrheit fuer
> Warmwasser-Ist-Temperatur und Ladesteuerung. Die Bosch-Cloud-Entities
> (`water_heater.dhw1`, `switch.charge`, `number.charge_setpoint`,
> `number.charge_duration`) werden vollstaendig entfernt, nicht nur ergaenzt.

Diese Entscheidung wurde vom Anwender explizit bestaetigt (siehe
`ww_v3_open_questions.md`, Teil A, Frage 2: "Yes, fully replace"). Ein
Parallelbetrieb (EMS-ESP fuer Anzeige, Bosch-Cloud weiter fuer die Steuerung)
wurde bewusst **nicht** gewaehlt, um keine zwei potenziell widerspruechlichen
Temperaturquellen im selben System zu haben.

## 2. Vorgehen: Neue Version statt Inline-Aenderung

Wie beim Uebergang v1_8 → v2 wird **keine bestehende Datei veraendert**,
sondern eine neue, in sich geschlossene Version erzeugt:

- `ww_v3_package.yaml` (neu, ersetzt `ww_v2_package.yaml` im Betrieb)
- `ww_v3_dashboard.yaml` (neu, ersetzt `ww_v2_dashboard.yaml` im Betrieb)
- `ww_v2_package.yaml` / `ww_v2_dashboard.yaml` bleiben unveraendert liegen
  (Rollback-Option, siehe `ww_v3_installation.md` Abschnitt 8/9)

Alle package-internen Entities wurden vom Namensraum `ww_v2_*` auf `ww_v3_*`
umbenannt (analog zur v1_8→v2-Umbenennung `ww_v18_*`/`ww_v1_8_*` → `ww_v2_*`).
Das verhindert Unique-ID-Kollisionen bei parallelem Vorhalten beider
Packages und macht in der Entity-Registry sofort erkennbar, welche Version
eine gegebene Entity erzeugt hat.

## 3. Drei Kategorien von Aenderungen

### 3.1 Reine Umbenennung (kein Verhaltensunterschied)

Alle `ww_v2_*` Package-eigenen Entities (input_number, input_boolean,
sensor, binary_sensor, automation, script, counter, utility_meter) →
identisches Verhalten unter `ww_v3_*`. Betrifft die komplette Preis-/PV-/
Batteriespeicher-/Lern-/Kosten-Logik – **nichts davon wurde durch die
EMS-ESP-Migration inhaltlich veraendert**.

### 3.2 Direkter Entity-Ersatz (Bosch Cloud → EMS-ESP, gleiche Funktion)

Siehe `ww_v3_analysis.md` Abschnitt 2 fuer die vollstaendige Tabelle. Kurz:

| Alt (Bosch Cloud) | Neu (EMS-ESP) |
|---|---|
| `water_heater.dhw1` (Attribut) | `sensor.boiler_dhw_curtemp` |
| `switch.charge` | `switch.boiler_dhw_onetime` |
| `number.charge_setpoint` | `number.boiler_dhw_seltempsingle` |
| `number.charge_duration` | entfaellt (siehe 3.3) |

### 3.3 Funktionale Verbesserungen, durch die neuen Daten erst moeglich

Diese Punkte sind **keine reine Migration**, sondern echte
Qualitaetsverbesserungen, die die Aufgabenstellung ausdruecklich vorsieht
("Add all meaningful entities", "missing diagnostics"). Sie wurden bewusst
eng am Migrationsthema gehalten (kein Funktionsumfang-Wildwuchs):

1. **Geraete-Maximaltemperatur wird aktiv geprueft** (`number.boiler_dhw_maxtemp`):
   `script.ww_v3_ladung_starten` kappt jedes Ladeziel auf diesen Wert und
   loggt eine Warnung, falls gekappt wurde. Schliesst eine in v2
   dokumentierte, ungeloeste Luecke (`ww_v2_open_questions.md`, Punkt zu
   `number.charge_setpoint`-Max).
2. **Unabhaengige Hardware-Bestaetigung der Ladung**
   (`binary_sensor.boiler_dhw_charging`/`_recharging`): neue Automation
   `ww_v3_debug_ladung_ohne_hardware_reaktion` erkennt "Befehl gesetzt, aber
   Geraet reagiert nicht" nach 10 Minuten. War mit der Bosch-Cloud-Anbindung
   technisch nicht moeglich (kein unabhaengiges Signal vorhanden).
3. **Vereinfachte, robustere Ladedauer-Begrenzung**: kein Geraete-Duration-
   Wert mehr noetig (siehe `ww_v3_analysis.md` Tabelle) – die zwei
   bestehenden Sicherheitsmechanismen (Stop-am-Ziel,
   Sicherheitsabschaltung-Maxdauer) reichen aus und sind unabhaengig vom
   verwendeten Geraet.
4. **Sperre des nativen EMS-ESP-PV-Aufladens** (`switch.thermostat_pvenabledhw`):
   verhindert Doppelsteuerung (siehe `ww_v3_analysis.md` Abschnitt 4).
5. **Zweiter Temperaturfuehler als Diagnosewert** (`sensor.boiler_dhw_curtemp2`):
   nur zur Anzeige/zum Vergleich, fliesst nicht in Regelentscheidungen ein
   (siehe `ww_v3_open_questions.md`, Offene Frage 1).

### 3.4 Nicht automatisierte EMS-ESP-Funktionen

Siehe `ww_v3_analysis.md` Abschnitt 3 (DHW-Prioritaet, Komfortmodus,
Zirkulationsmodus, Thermostat-Betriebsart). Werden nur als optionale, manuell
bedienbare Zeilen im Dashboard angezeigt, nicht automatisiert.

## 4. Vermeidung von Doppel-Entities

Es wird an keiner Stelle eine EMS-ESP-Entity in eine neue `ww_v3_*`-Entity
"eingepackt", wenn sie bereits direkt (ohne Transformation) sinnvoll ist.
Beispiele: `sensor.boiler_dhw_curtemp2`, `binary_sensor.boiler_dhw_tempok`,
`binary_sensor.boiler_dhw_3wayvalve`, `number.boiler_dhw_maxtemp`,
`select.thermostat_dhw_mode` werden **direkt** im Dashboard referenziert,
genau wie v2 dies bereits fuer externe Werte wie `sensor.gesamtleistung_
haushalt` oder `sensor.wp_gesamtleistung` getan hat. Ein `ww_v3_*`-Wrapper
wird nur dort erzeugt, wo tatsaechlich Mehrwert entsteht: Einheitliche
Benennung/Rundung (z. B. `ww_v3_tibber_preis`), Verfuegbarkeits-Gate und
Kombination mehrerer Quellen (z. B. `ww_v3_warmwassertemperatur`,
`ww_v3_pv_leistung_gesamt`) oder eigene Entscheidungslogik
(`ww_v3_ladebedarf` usw.).

## 5. Migrationsreihenfolge (empfohlen fuer den Anwender)

Siehe `ww_v3_installation.md` fuer die vollstaendige Anleitung. Kurzfassung:

1. EMS-ESP-Pflicht-Entities in Home Assistant pruefen (siehe Installation
   Abschnitt 1).
2. `ww_v3_package.yaml`/`ww_v3_dashboard.yaml` ablegen, **`ww_v2_package.yaml`
   parallel deaktivieren** (nicht loeschen) – beide duerfen nicht
   gleichzeitig aktiv sein, da sie sonst unabhaengig voneinander auf
   unterschiedliche Ladeschalter (`switch.charge` vs.
   `switch.boiler_dhw_onetime`) zugreifen und der Warmwasserspeicher
   theoretisch von zwei Systemen gleichzeitig gesteuert werden koennte,
   solange beide Bosch-Cloud- und EMS-ESP-Integration parallel laufen.
3. Kontrollierte Erstaktivierung mit Debug-Modus (identisches Vorgehen wie
   v1_8→v2, siehe Installation Abschnitt 6).
4. Nach stabilem Betrieb: `ww_v2_package.yaml`/`ww_v2_dashboard.yaml` und
   die Bosch-Cloud-Integration (Home Connect) koennen bei Bedarf entfernt
   werden (nicht Teil dieses Packages, liegt beim Anwender).
