# Entwicklungsverlauf

Chronologische Zusammenfassung aller nummerierten Zwischenstufen. Stufen 01–13 sind
eine systematische Bisektion eines Boot-Absturzes; ab Stufe 14 sind es reine
Feature-Iterationen auf dem dabei gefundenen, sicheren Muster.

## Phase 1: Absturz-Bisektion (01–13)

Ausgangslage: eine erste, vollständige Version des Wecker-Funktionsumfangs stürzte
beim Booten ab ("Panic handler entered multiple times", kein Backtrace). Schrittweise
Eingrenzung durch minimale, aufeinander aufbauende Test-Configs:

| Stufe | Änderung | Ergebnis |
|---|---|---|
| 01 | Nur Hardware-Init + ein LVGL-Label ("Hello World") | ✅ funktioniert |
| 02 | + Uhrzeit/Datum-Anzeige, NTP-Status | ✅ funktioniert |
| 03 | + volle Weckzeit-Logik (Number+Roller+Switches+Checkboxen) | ❌ Absturz |
| 04 | Wie 03, minimiert auf nur Number+Roller | ❌ Absturz (reproduziert) |
| 05 | Stufe 02 + leere zweite Seite (Navigation) | ✅ funktioniert |
| 06 | + Number-Entities ohne on_value-Handler | ✅ funktioniert |
| 07 | + ein Roller (nur Roller→Number-Richtung) | ✅ funktioniert |
| 08 | + zweiter Roller | ✅ funktioniert |
| 09 | Roller-Modus NORMAL statt INFINITE getestet | ✅ funktioniert (kein Unterschied) |
| 10 | + ":"-Trennlabel zwischen den Rollern | ✅ funktioniert |
| 11 | + `number.on_value` → `lvgl.roller.update` (Number→Roller-Richtung) | ❌ Absturz reproduziert — **Ursache gefunden** |
| 12 | Fix-Versuch: `on_boot`-Sync mit `priority: -100` + Ready-Flag | ⚠️ behebt den Boot-Absturz, aber neuer verzögerter Laufzeit-Crash |
| 13 | Strukturfix: Roller-Wert wird nur beim Öffnen der Seite aus der Entity gezogen ("pull"), kein `number.on_value` mehr | ✅ **funktioniert, bestätigt** |

**Kernerkenntnis:** `number.on_value` (oder ähnliches), das ein LVGL-Widget aktualisiert,
race't mit der Komponenten-Setup-Reihenfolge — das Widget kann noch nicht existieren,
wenn ein wiederhergestellter (`restore_value`) Wert beim Booten sein `on_value` feuert.
Siehe [Docs/ESPHome-Lessons-Learned.md](Docs/ESPHome-Lessons-Learned.md) für die
volle technische Erklärung.

## Phase 2: Feature-Ausbau (14–26)

| Stufe | Änderung |
|---|---|
| 14 | Minute als zwei Ziffern-Roller (Zehner/Einer) statt 5er-Schritten; großes blinkendes "ALARM"-Label statt Ton |
| 15 | Deutsche Umlaute korrigiert (Schriftart-Glyphen); "OK"-Taste mit Aufblitz-Bestätigung; SNOOZE/STOP-Tasten verkleinert; Statuszeile "Wecker: HH:MM - Wochentage" |
| 16 | Dynamisches "Wecker aktiv/inaktiv"-Label (grün/grau); Stunde/Minute-Beschriftung repariert (war hinter den Rollern verdeckt); Checkboxen grau/grün je nach Zustand |
| 17 | Native Home-Assistant-Zeit-Entität "Weckzeit" (bidirektional mit den Rollern synchron); Gerätename vereinheitlicht |
| 18 | Roller-Endlos-Scroll (23→00) für alle drei Roller; größere Roller-Schrift; Stunde/Minute-Number-Entities aus HA ausgeblendet (`internal: true`); Hauptschalter umbenannt |
| 19 | Uhrzeit/Datum ca. 1,5× größer (eigene Schriftgrößen) |
| 20 | "Weckzeit"-Entität bekommt eigene, direkte Persistenz |
| 21 | Blinkendes "Synchronisiere Zeit" bis zum ersten NTP-Sync |
| 22 | Fix-Versuch für Weckzeit-Reset-Bug: `on_boot priority: -100` — **stellte sich später als nicht die Ursache heraus** |
| 23 | **Echter Fix** für den Reset-Bug: `preferences: flash_write_interval: 0s` (ESPHome schreibt Änderungen sonst erst nach bis zu einer Minute auf den Flash) |
| 24 | Zeit wird primär von Home Assistant abgefragt (native API), NTP als Fallback |
| 25 | Außen-/Zimmertemperatur oben in der Kopfzeile, per MQTT, Quelle frei aus Home Assistant wählbar (siehe README) |
| 26 | Temperaturzeile links statt zentriert ausgerichtet |

`smart-wecker-waveshare.yaml` (ohne Nummer) entspricht dem zuletzt bestätigten
Stand (Stufe 23). Die Stufen 24–26 (HA-Zeitquelle, MQTT-Temperaturanzeige,
Links-Ausrichtung) sind bereits gebaut, aber am Gerät noch nicht abschließend
bestätigt — nach erfolgreichem Test werden sie ebenfalls in die Hauptdatei
übernommen.

Die Zwischendateien der Stufen 01–23 und 25 wurden aus dem Repo entfernt, um es
übersichtlich zu halten, nachdem ihr Inhalt hier zusammengefasst und in
[Docs/ESPHome-Lessons-Learned.md](Docs/ESPHome-Lessons-Learned.md) dokumentiert
wurde (vollständig weiterhin in der Git-Historie enthalten). Stufe 24 und 26
liegen noch als Dateien vor, da sie den aktuell wartenden, noch zu bestätigenden
Stand markieren.
