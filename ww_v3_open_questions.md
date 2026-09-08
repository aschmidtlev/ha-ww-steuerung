# ww_v3 – Offene Fragen: Status

## Teil A – Migrationsentscheidungen (beantwortet, in Session vom 2026-09-08)

Diese vier Fragen wurden vor Beginn der Implementierung gestellt, um zu
verhindern, dass an der falschen Architektur (Python-Integration statt
YAML-Package) oder mit falschen Sicherheitsannahmen gearbeitet wird. Die
Antworten sind vollstaendig in `ww_v3_package.yaml`, `ww_v3_dashboard.yaml`
und der uebrigen v3-Dokumentation umgesetzt.

| # | Frage (Kurzform) | Antwort des Anwenders | Umsetzung |
|---|---|---|---|
| 1 | Deliverable-Format: YAML-Package statt Python-Integration | Ja, an YAML-Package anpassen | `ww_v3_package.yaml`/`ww_v3_dashboard.yaml` statt Python-Code |
| 2 | EMS-ESP als alleinige Quelle statt nur Ergaenzung | Ja, vollstaendig ersetzen | `water_heater.dhw1`/`switch.charge`/`number.charge_setpoint`/`_duration` vollstaendig entfernt |
| 3 | Welcher EMS-ESP-Schalter fuer "jetzt laden" | `switch.boiler_dhw_onetime` (boiler-seitig) statt `switch.thermostat_dhw_charge` (thermostat-seitig) | In allen Automationen/Skripten verwendet |
| 4 | Umgang mit nativem EMS-ESP-PV-Aufladen | Deaktiviert halten, ww_v3 hat alleinige Kontrolle | `automation.ww_v3_pvenabledhw_sperren` |

## Teil B – Neue, nicht blockierende Punkte (v3-spezifisch)

Diese Punkte verhindern **nicht** die Aktivierung von ww_v3, sollten aber
vom Anwender zur Kenntnis genommen bzw. bei Gelegenheit entschieden werden.

**B1. `sensor.boiler_dhw_curtemp` (intern) vs. `sensor.boiler_dhw_curtemp2`
(extern) als Regelungsquelle.**
ww_v3 verwendet ausschliesslich `curtemp` ("Current intern temperature") als
Quelle fuer `sensor.ww_v3_warmwassertemperatur` und damit fuer alle
Ladeentscheidungen. Begruendung: der Snapshot-Wert (50.2 °C) lag naeher am
bisherigen `water_heater.dhw1`-Wert aus der v2-Analyse (52.2 °C) und am
Standard-Ladeziel (51 °C) als `curtemp2` (43.4 °C, vermutlich ein
Zapfstellen-/Ausgangsfuehler, der nach Warmwasserentnahme kurzfristig
abweicht). Diese Zuordnung ist jedoch **nicht durch Herstellerdokumentation
verifiziert**, sondern eine plausible Annahme aus den Snapshot-Werten.
→ Falls gewuenscht: bitte ueber einen laengeren Zeitraum beobachten, welcher
der beiden Fuehler staerker mit dem tatsaechlich am Wasserhahn spuerbaren
Komfort korreliert, und bei Bedarf `sensor.ww_v3_warmwassertemperatur` auf
`curtemp2` umstellen (eine Zeile in `ww_v3_package.yaml`).

**B2. Verhalten von `switch.boiler_dhw_onetime` nach Ladeabschluss ist nicht
dokumentiert verifizierbar.**
Es ist aus der vorliegenden CSV (reiner Zustands-Snapshot) nicht ableitbar,
ob sich dieser Schalter nach einem abgeschlossenen Ladezyklus selbststaendig
zuruecksetzt (typisches Verhalten fuer "One-Time"-Funktionen) oder dauerhaft
`on` bleibt, bis er aktiv ausgeschaltet wird. ww_v3 verlaesst sich nicht auf
automatisches Zuruecksetzen (siehe `ww_v3_quality_report.md` Abschnitt 3,
Punkt 4) und schaltet ihn in jedem Fall selbst aus. Falls sich der Schalter
tatsaechlich selbst zuruecksetzt, hat das keine negativen Auswirkungen
(unschaedlicher Doppel-`turn_off`).
→ Waehrend der Testbetriebsphase (siehe `ww_v3_installation.md` Abschnitt 6,
Schritt 10) beobachten und bei Gelegenheit dokumentieren.

**B3. `switch.boiler_dhw_dhwprio` (DHW-Prioritaet) wird nicht automatisiert.**
Koennte theoretisch genutzt werden, um waehrend einer preis-/PV-optimierten
Ladung den Heizkreis kurzzeitig zurueckzustellen und die Ladung zu
beschleunigen (kuerzere Verdichterlaufzeit, weniger Netzbezug waehrend eines
teuren Preisfensters). Wurde bewusst **nicht** implementiert, da dies den
Heizkreis (ausserhalb des Funktionsumfangs "Warmwassersteuerung") aktiv
beeinflussen wuerde und nicht Teil der urspruenglichen Aufgabenstellung ist.
→ Falls gewuenscht, als separate, explizit angeforderte Erweiterung
umsetzbar.

**B4. Geraete-Maximaltemperatur koennte sich aendern.**
`number.boiler_dhw_maxtemp` wird bei jedem Ladestart live gelesen (kein
gecachter Wert), daher wirkt sich eine spaetere Aenderung am Geraet
automatisch auf die Kappungslogik aus. Keine Aktion noetig.

**B5. Kein `ha core check_config` in dieser Umgebung möglich.**
Identisch zu `ww_v2_open_questions.md` Punkt B5 – es stand keine laufende
Home-Assistant-Instanz zur Verfügung. Die in `ww_v3_quality_report.md`
beschriebenen statischen Prüfungen (grep-Suchen, manuelle Durchsicht,
Existenzabgleich gegen die CSV) ersetzen dies nicht vollständig. **Vor der
ersten Aktivierung zwingend „Konfiguration überprüfen“ in Home Assistant
selbst ausführen.**

**B6. `sensor.gesamtleistung_haushalt` wurde durch einen selbst berechneten
Hausverbrauchs-Sensor ersetzt.**
Ein Plausibilitaetsvergleich anhand der CSV vom 30.08.2026, 00:35 Uhr (PV=0,
Waermepumpe im Standby mit 9,8 W) zeigte: `sensor.evu_leistung` (Netzbezug)
= 699,9 W, aber `sensor.gesamtleistung_haushalt` = nur 193,7 W – ein
Unterschied von ~506 W, der durch nichts in den vorliegenden Daten erklaert
wird. Vermutung: die Entity misst nur einen Teil-Stromkreis, nicht das
gesamte Haus. `sensor.ww_v3_hausverbrauch_berechnet` ersetzt sie jetzt
ueberall im Dashboard und in der Pflichtdaten-Diagnose durch eine
Energiebilanz-Berechnung: PV gesamt + Netzbezug − Netzeinspeisung (+
Batterieleistung, falls Marstek verfuegbar). Zwei Einschraenkungen dieser
neuen Berechnung:
- Die Vorzeichenkonvention von `sensor.marstek_venus_modbus_batterieleistung`
  ist nicht herstellerseitig verifiziert (Annahme: positiv = Entladung).
  Da Marstek aktuell hardwareseitig ausgefallen ist, hat das derzeit keine
  praktische Auswirkung (Beitrag = 0), sollte aber geprueft werden, sobald
  die Modbus-Verbindung wiederhergestellt ist.
- Die Berechnung geht implizit davon aus, dass PV-Erzeugung entweder lokal
  verbraucht oder eingespeist wird (keine weiteren Verbraucher/Erzeuger
  zwischen Wechselrichter und Netzzaehler ausser der Marstek-Batterie). Der
  urspruengliche Rohwert von `sensor.gesamtleistung_haushalt` bleibt als
  Attribut `alter_rohwert_sensor_gesamtleistung_haushalt` an
  `sensor.ww_v3_hausverbrauch_berechnet` sichtbar, falls ein Vergleich
  gewuenscht ist. Die Entity selbst wird nirgends mehr geloescht oder
  veraendert – nur nicht mehr referenziert.
→ Unabhaengig davon lohnt sich eine Pruefung, welchen Stromkreis
`sensor.gesamtleistung_haushalt` in der Quell-Integration tatsaechlich
misst (siehe Geraete & Dienste), da das auf ein Konfigurationsproblem
ausserhalb dieses Packages hindeuten koennte.

## Teil C – Aus ww_v2 übernommene, weiterhin offene Punkte

Die Punkte B1–B4 aus `ww_v2_open_questions.md` (veraltete Temperatur als
Sperrkriterium, `switch`-Anzeige auf Debug-Seite, differenzierte Behandlung
fehlgeschlagener `switch.turn_on`-Aufrufe, Recorder-Ausschlussliste) sind
weiterhin gültig und wurden durch die EMS-ESP-Migration nicht berührt – sie
gelten für v3 in gleicher Form, nur mit den migrierten Entity-Namen.
