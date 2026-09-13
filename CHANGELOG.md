# Entwicklungsverlauf

Chronologische Zusammenfassung aller nummerierten Zwischenstufen (01–27). Stufen
01–13 sind eine systematische Bisektion eines Boot-Absturzes; ab Stufe 14 sind es
reine Feature-Iterationen auf dem dabei gefundenen, sicheren Muster. Mit Stufe 27
endet die nummerierte Dateikonvention für diesen Funktionsstrang -- der
zusammengeführte Endstand heißt `Smart-Wecker-Waveshare_MQTT.yaml` (siehe unten).

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
| 27 | Bugfix: Hauptschalter-Label und Wochentag-Checkboxen zeigten eine Fernänderung (z.B. über ein HA-Dashboard) erst nach erneutem Öffnen der Einstellungsseite an — jetzt periodisch (alle 500ms) live nachgezogen |

`smart-wecker-waveshare-24.yaml` bleibt als zuletzt vollständig bestätigter,
stabiler Meilenstein (noch ohne Temperaturanzeige) im Repo erhalten.

Die Stufen 25–27 (MQTT-Temperaturanzeige, Links-Ausrichtung, Live-Sync-Bugfix)
wurden zusammengeführt und liegen als **`Smart-Wecker-Waveshare_MQTT.yaml`** vor —
das ist der aktuelle, empfohlene Stand. Die numerierte Zwischenstufen-Datei-Konvention
wird für diesen Funktionsstrang damit durch einen benannten, feststehenden Dateinamen
abgelöst; neue Experimente/Zwischenstufen landen ab jetzt im lokalen, nicht
committeten Ordner `Entwuerfe/` statt als weitere nummerierte Dateien im Repo.

Die Zwischendateien der Stufen 01–23, 25, 26 und 27 wurden aus dem Repo entfernt, um
es übersichtlich zu halten, nachdem ihr Inhalt hier zusammengefasst und in
[Docs/ESPHome-Lessons-Learned.md](Docs/ESPHome-Lessons-Learned.md) dokumentiert
wurde (vollständig weiterhin in der Git-Historie enthalten).

## Phase 3: Ereignisgesteuerte Live-Sync + Bildschirm-Zucken (29–38)

Ausgangspunkt: der Nutzer bemerkte ein regelmäßiges kurzes Zucken/Blitzen der Anzeige
und vermutete das in Stufe 27 eingeführte 500ms-Polling (Hauptschalter+Checkboxen bei
jedem Tick nachziehen, unabhängig ob sich etwas geändert hat) als Ursache. Eine
schrittweise Fehlersuche über mehrere Zwischenstufen (nur noch lokal in `Entwuerfe/`,
nicht mehr einzeln committet):

| Stufe | Änderung | Ergebnis |
|---|---|---|
| 29 | 500ms-Polling durch ereignisgesteuerte `on_turn_on`/`on_turn_off`-Handler ersetzt (mit `ui_ready`-Gate gegen das bekannte Boot-Absturzmuster) | Zucken weiterhin da; zusätzlich Regression: HA→Gerät-Sync brach komplett ab |
| 30 | Diagnose-Logging in allen Schalter-Handlern + 3s-Fallback-Polling als Absicherung | Log bewies: ereignisgesteuerter Weg feuert korrekt zeitgleich zu HA-Änderungen — das 3s-Polling war nie nötig |
| 31 | `display: update_interval` testweise 1s→5s | Zucken seltener, aber nicht weg (Test war durch das gleichzeitige 3s-Polling nicht sauber isoliert) |
| 32 | 3s-Fallback-Polling entfernt; Weckzeit-Roller bekommen dieselbe ereignisgesteuerte Live-Sync (neues `roller_driven_update`-Flag verhindert Selbstüberschreiben während des eigenen Scrollens) | Weckzeit-Sync aus HA funktioniert jetzt sofort |
| 33 | `update_interval` testweise 1s→30s (sauber isolierter Test) | Zucken **unverändert** — `update_interval` damit als Ursache widerlegt |
| 34 | `wifi: power_save_mode: none` (bekannter Kandidat für periodische CPU-Stocker) | Kein eindeutiger Effekt, aber beibehalten |
| 35 | `pclk_frequency` 16MHz→8,2MHz (Nutzerwunsch: 20Hz Bildwiederholrate reicht) | **Fehlgeschlagen** — kompletter Sync-Verlust des Panels (wechselnde Farbflächen) |
| 36 | Rollback auf letzten funktionierenden Stand (= Stufe 34) | Display wieder normal, Zucken (seltener/unregelmäßig) weiterhin da |
| 37 | `pclk_frequency` 16MHz→10MHz (vorsichtigerer Test) | Ebenfalls Sync-Verlust |
| 38 | `pclk_frequency` 16MHz→14MHz | **Vom Nutzer am Gerät bestätigt: deutliche Besserung** |

**Kernerkenntnisse:**
- Der ESPHome-`mipi_rgb`-Treiber bietet keinen direkten Bounce-/Doppelpuffer-Parameter
  über YAML (im Quellcode geprüft) — der einzige wirksame Hebel gegen vermutete
  PSRAM-Bandbreiten-Konkurrenz ist `pclk_frequency`.
- Dieses Panel hat sehr wenig Spielraum unter dem Referenztakt: 14MHz funktioniert,
  10MHz und darunter führen zu komplettem Sync-Verlust (nicht nur Zucken).
- Das ursprünglich für Roller bewusst ausgeschlossene Live-Sync-Muster (Stufe 27) lässt
  sich sicher nachrüsten, wenn ein zusätzliches "wurde die Änderung gerade vom Widget
  selbst ausgelöst"-Flag (`roller_driven_update`) einen Selbstüberschreib-Rücklauf
  während aktiver Nutzer-Interaktion verhindert.

Die Zwischendateien der Stufen 29–37 liegen nicht im Repo (nur lokal in `Entwuerfe/`,
Stufe 38 wurde direkt zusammengeführt) — der Endstand ist in
`Smart-Wecker-Waveshare_MQTT.yaml`.
