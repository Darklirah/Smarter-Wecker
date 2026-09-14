# Smart-Wecker – Waveshare-Variante

> Dieses Projekt wurde mit Unterstützung von [Claude Code](https://claude.com/claude-code) entwickelt.

Ein smarter Wecker auf Basis eines ESP32-S3-Touch-Displays, gebaut mit
[ESPHome](https://esphome.io/) und in [Home Assistant](https://www.home-assistant.io/)
integriert. Uhrzeit, Datum, eine frei einstellbare Weckzeit mit Wochentagen und eine
Außen-/Zimmertemperatur-Anzeige laufen direkt auf dem Gerät; die Weckzeit,
Wochentage und Temperaturquelle lassen sich zusätzlich bequem aus Home Assistant
heraus steuern — ganz ohne den ESP32 dafür neu zu flashen.

Dieses Board hat **keine Audioausgabe**, der Wecker weckt daher rein visuell: ein
großes, blinkendes rotes "ALARM"-Label plus zwei große Touchflächen zum Stoppen bzw.
9 Minuten Schlummern. Da das Display keinen nativen Audioanschluss besitzt, ist das
hilfsweise so gelöst, um das Alarmereignis zu visualisieren. Im weiteren Ausbau wird
die Hardware um eine Audioausgabe ergänzt und das Projekt dann entsprechend angepasst.

**Hinweis:** Dies stellt keine voll funktionsfähige, rocksolide Version dar, sondern
ein Proof of Concept — also eher Work in Progress.

## Fotos

Fotos des laufenden Geräts im Ordner [BILDER/](BILDER/): Uhrzeit-Seite, Einstellungsseite
(Weckzeit/Wochentage) und die Alarm-Seite.

## Hardware

| Merkmal | Wert |
|---|---|
| Board | Waveshare ESP32-S3-Touch-LCD-7 (**Basisversion mit CH422G-IO-Expander** — nicht die "-7B"-Variante mit CH32V003) |
| Chip | ESP32-S3, Dual-Core Xtensa LX7, bis 240 MHz |
| Flash | 16 MB |
| PSRAM | 8 MB Octal (mit reduzierter Taktrate 80 MHz betrieben, siehe Lessons-Learned) |
| Display | 7", 800×480, RGB-Parallel-Interface, 65K Farben |
| Touch | Kapazitiv, 5-Punkt, GT911 (I2C, mit Interrupt-Pin) |
| Backlight | Nur Ein/Aus (kein PWM/Dimmen — Hardware-Eigenschaft dieses Boards) |
| Audio | Keins — kein Onboard-Verstärker/Lautsprecheranschluss, keine sicher nutzbaren freien GPIOs dafür |
| Physischer Taster | Keiner — dafür zwei große Touchflächen (260×260px) auf der Alarm-Seite |
| Funk | Wi-Fi 2.4 GHz, Bluetooth 5 (LE, ungenutzt) |
| Stromversorgung | **5V / mindestens 2A, siehe Warnung unten** — mit einem schwächeren Netzteil flackert das Display |

Ausführliche Pinbelegung, Quellen und Recherche-Details: [Docs/hardware-specs.md](Docs/hardware-specs.md),
[Docs/quellen.md](Docs/quellen.md).

## Funktionsumfang

- **Uhrzeit-Seite:** große Uhrzeit + Datum, Außen-/Zimmertemperatur oben links (per
  MQTT, Quelle frei aus HA wählbar), blinkendes "Synchronisiere Zeit" bis zur ersten
  erfolgreichen Zeitsynchronisation, NTP-Status-Punkt, Statuszeile unten
  ("Wecker: HH:MM - Wochentage")
- **Einstellungsseite:** Stunden-Roller + zwei Minuten-Ziffern-Roller (Zehner/Einer,
  jede Minute frei wählbar, Endlos-Scroll), Hauptschalter mit dynamischem
  "Wecker aktiv/inaktiv"-Label (grün/grau), 7 Wochentag-Checkboxen (grau = aus,
  grün = an), "OK"-Taste quittiert mit kurzem Aufblitzen der eingestellten Zeit
- **Alarm-Seite:** blinkendes rotes "ALARM"-Label + blinkender Hintergrund (rein
  visuell, kein Ton möglich), zwei große Touchflächen (SNOOZE 9 Min / STOP)
- **Home-Assistant-Entitäten:** native Zeit-Entität "Weckzeit", Schalter
  "Wecker aktiv" + 7 Wochentag-Schalter, alle persistent über Neustarts hinweg
  (sofort auf Flash geschrieben)
- **Zeitquelle:** primär über die native Home-Assistant-API, NTP als Fallback

## Einrichtung

### 1. `secrets.yaml` anlegen

Diese Datei existiert nur lokal bei dir und wird **nie** committet
(`.gitignore` schließt sie aus). Lege sie im Projektordner neu an mit folgendem
Inhalt, jeweils mit deinen echten Werten:

| Schlüssel | Bedeutung |
|---|---|
| `wifi_ssid` | Name deines WLAN-Netzwerks |
| `wifi_password` | WLAN-Passwort |
| `OTA_Secret` | frei wählbares Passwort für Over-The-Air-Firmware-Updates |
| `api_encryption_key` | Verschlüsselungscode für die native ESPHome-API (Home-Assistant-Verbindung) — 32 Byte, Base64-kodiert, z.B. per `esphome secrets` generieren |
| `MQTT-Broker` | Host/IP deines MQTT-Brokers (für die Temperaturanzeige) |
| `MQTT-user` | MQTT-Benutzername |
| `MQTT-Passwort` | MQTT-Passwort |

Beispiel (Platzhalterwerte ersetzen):
```yaml
wifi_ssid: "Dein-WLAN-Name"
wifi_password: "Dein-WLAN-Passwort"
OTA_Secret: "ein-selbst-gewaehltes-ota-passwort"
api_encryption_key: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx="
MQTT-Broker: "192.168.1.xxx"
MQTT-user: "dein-mqtt-benutzername"
MQTT-Passwort: "dein-mqtt-passwort"
```

### 2. Kompilieren & Flashen

Über die ESPHome-Oberfläche (z.B. das ESPHome-Add-on in Home Assistant, oder die
`esphome`-CLI): `Smart-Wecker-Waveshare_MQTT.yaml` auswählen (aktueller, empfohlener
Stand inkl. Temperaturanzeige — siehe [Dateien unten](#dateien-in-diesem-projekt)),
kompilieren, per USB oder OTA flashen. Falls dabei Probleme auftreten, ist
`smart-wecker-waveshare-24.yaml` der zuletzt vollständig bestätigte, stabile
Stand ohne Temperaturanzeige.

### 3. Gerät in Home Assistant hinzufügen

Nach dem ersten Boot meldet sich das Gerät per mDNS bei Home Assistant
(Einstellungen → Geräte & Dienste → "Entdeckt"). Beim Hinzufügen wird der
Verschlüsselungscode aus `secrets.yaml` (`api_encryption_key`) abgefragt. Danach
erscheinen automatisch alle Entitäten des Geräts (Weckzeit, Wecker aktiv, die 7
Wochentage, LCD Backlight, …).

### 4. Home-Assistant-Helfer für die Temperaturanzeige anlegen

Die Temperaturzeile bezieht ihre Werte per MQTT aus **beliebigen** bestehenden
Home-Assistant-Sensoren — die Auswahl, welcher Sensor das sein soll, passiert
komplett in Home Assistant, nicht in der ESPHome-Konfiguration. Dafür müssen in
Home Assistant einmalig folgende Objekte angelegt werden (sie sind bewusst nicht Teil
dieses Repos, da sie zu deiner HA-Instanz gehören, nicht zum Gerät):

**Zwei Text-Helfer** (Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen →
Text):
- **"Wecker Außentemperatur-Quelle"** — hier die Entity-ID deines Außentemperatur-Sensors eintragen (z.B. `sensor.aussentemperatur`)
- **"Wecker Zimmertemperatur-Quelle"** — analog für die Zimmertemperatur

**Eine Automatisierung**, die die referenzierten Sensoren regelmäßig ausliest und per
MQTT (retained) veröffentlicht:
- **Auslöser:** alle 1 Minute (`time_pattern`, `minutes: "/1"`) **plus** bei Änderung
  von einem der beiden Text-Helfer (für sofortige Aktualisierung nach dem Umstellen)
- **Aktion:** für jeden der beiden Helfer, falls der referenzierte Sensor einen
  gültigen Zustand hat: `mqtt.publish` mit `topic: smartwecker/aussentemperatur` bzw.
  `smartwecker/zimmertemperatur`, `payload` = aktueller Zustand des referenzierten
  Sensors, `retain: true`

Der Schalter **"Wecker Temperaturzeile ausblenden falls leer"** (erscheint automatisch
als Geräte-Entität, sobald das Gerät zu HA hinzugefügt wurde) steuert, ob bei
fehlendem Sensorwert "n/v" angezeigt oder die ganze Zeile ausgeblendet wird.

Benötigt einen MQTT-Broker (z.B. Mosquitto) im lokalen Netz.

## Wichtige Lektionen aus der Entwicklung

Eine ausführliche, 13-stufige Bisektion (`smart-wecker-waveshare-01.yaml` bis
`-13.yaml`) hat einen anfänglichen Boot-Absturz auf ein einziges Muster
zurückgeführt: **jede direkte Rückkopplung "Entity ändert sich → LVGL-Widget wird
automatisch aktualisiert"** (z.B. `number.on_value` → `lvgl.roller.update`) führt zu
Abstürzen, weil die Reihenfolge, in der ESPHome-Komponenten beim Booten eingerichtet
werden, das betroffene Widget zu diesem Zeitpunkt noch nicht existieren lässt.
Der Fix: Widget-Zustände werden nur **beim tatsächlichen Öffnen** einer Seite aus dem
Entity-Zustand gezogen ("pull"), nie automatisch bei jeder Entity-Änderung
zurückgeschrieben.

Weitere, beim Ausbau der Funktionen entdeckte ESPHome-Fallstricke (Schriftart-Glyphen,
`on_boot`-Prioritäten, verzögertes Flash-Schreiben von persistenten Werten) sind in
[Docs/ESPHome-Lessons-Learned.md](Docs/ESPHome-Lessons-Learned.md) dokumentiert.
Ein stufenweiser Verlauf aller Entwicklungsschritte steht in
[CHANGELOG.md](CHANGELOG.md).

## ⚠️ Stromversorgung: ein ausreichend starkes 5V-Netzteil ist Pflicht

**Das Board braucht ein 5V-Netzteil mit mindestens 2A.** Ein schwächeres Netzteil
(oder ein normaler USB-Port am PC/Hub) zeigt sich nicht als Absturz, sondern als
**unregelmäßiges kurzes Zucken/Blitzen des Bildschirms** — ein Fehlerbild, das sehr
nach einem Software- oder Timing-Problem aussieht.

Genau das hat bei diesem Projekt eine ausgedehnte Fehlersuche in die falsche Richtung
gelenkt (siehe [CHANGELOG.md](CHANGELOG.md), Phase 3). Reihenweise Software-Ursachen
wurden untersucht und wieder ausgeschlossen:

- periodisches Nachziehen von Schalter-Widgets (daraufhin auf rein ereignisgesteuerte
  Updates umgebaut — an sich sinnvoll, aber nicht die Ursache),
- die Bildwiederholrate des Displays (`update_interval`, 1s bis 30s getestet, kein Effekt),
- der WLAN-Energiesparmodus (`power_save_mode: none`, kein eindeutiger Effekt),
- der Pixeltakt `pclk_frequency` (von 16MHz auf 14MHz gesenkt — schien zu helfen, war
  aber offenbar kein echter Effekt),
- vermutete PSRAM-Bandbreiten-Konkurrenz zwischen RGB-Display-DMA und CPU.

**Die tatsächliche Lösung war ein Netzteilwechsel:** mit einem Raspberry-Pi-Netzteil
(5V / 2A) ist das Flackern vollständig verschwunden. Wer dieses Board nachbaut und
Flackern sieht, sollte deshalb **zuerst das Netzteil tauschen**, bevor er anfängt, in
der YAML-Konfiguration zu suchen.

Anmerkung zu `pclk_frequency`: der Wert steht in der Konfiguration weiterhin auf
`14MHZ` statt auf dem Hardware-Referenzwert `16MHZ`. Das ist jetzt nicht mehr nötig und
kostet etwas Bildwiederholrate (~34Hz statt ~39Hz) — bei stabiler Stromversorgung kann
er wieder auf `16MHZ` gestellt werden. Falls doch jemals ein niedrigerer Takt getestet
wird: **nicht unter 14MHz gehen** — sowohl 10MHz als auch 8,2MHz haben bei diesem Panel
zu komplettem Synchronisationsverlust geführt (wechselnde Farbflächen statt normaler
Anzeige), das Panel hat also kaum Spielraum nach unten.

## Offene Probleme

Aktuell keine bekannten.

Das kurze Reißen des Bildes beim Umschalten auf die Einstellungsseite ist kein Fehler,
sondern eine Eigenschaft dieser Panel-Bauart — die Erklärung dazu steht am Ende dieser
Datei unter
[Warum das Bild beim Seitenwechsel kurz „zuckt" (Tearing)](#warum-das-bild-beim-seitenwechsel-kurz-zuckt-tearing).

## Dateien in diesem Projekt

- `Smart-Wecker-Waveshare_MQTT.yaml` — **aktueller, empfohlener Stand**: alles aus
  `-24.yaml` (siehe unten) PLUS die Außen-/Zimmertemperatur-Zeile (per MQTT, Quelle
  frei aus Home Assistant wählbar, siehe
  [Temperaturanzeige einrichten](#4-home-assistant-helfer-für-die-temperaturanzeige-anlegen)),
  links ausgerichtet, mit einem Schalter zum Ausblenden bei fehlendem Wert, sowie dem
  Bugfix, dass ferngesteuerte Änderungen (z.B. über ein HA-Dashboard) am Hauptschalter
  und den Wochentag-Checkboxen jetzt live am Display ankommen statt erst beim nächsten
  Öffnen der Einstellungsseite.
- `smart-wecker-waveshare-24.yaml` — zuletzt **vollständig bestätigter, stabiler**
  Meilenstein-Stand: Zeitsynchronisation primär über Home Assistant (NTP als
  Fallback), aber noch **ohne** Temperaturanzeige. Als Rückfalloption aufgehoben.
  Die Zwischendateien aller anderen Entwicklungsschritte wurden nach Abschluss wieder
  entfernt, um das Repo übersichtlich zu halten — ihr Inhalt ist im
  [CHANGELOG.md](CHANGELOG.md) zusammengefasst und in der Git-Historie enthalten.
  Neue Experimente/Zwischenstufen landen ab jetzt im lokalen, nicht committeten Ordner
  `Entwuerfe/` (siehe `.gitignore`).
- `BILDER/` — Fotos des laufenden Geräts
- `Docs/hardware-specs.md` — recherchierte Hardware-Spezifikationen mit Quellen
- `Docs/quellen.md` — Liste aller verwendeten Quellen
- `Docs/waveshare-esp32-s3-touch-lcd-7.yaml` — Kopie des ursprünglich eingebundenen
  (inzwischen nicht mehr genutzten) Community-Pakets, zur Referenz
- `Docs/Waveshare_Swirch_verifiziert.yaml` — Kopie der verifizierten Hardware-Referenz
- `Docs/ESPHome-Lessons-Learned.md` — projektübergreifende ESPHome-Lektionen
- `secrets.yaml` — deine lokalen Zugangsdaten (nicht Teil des Repos, siehe oben)

## Warum das Bild beim Seitenwechsel kurz „zuckt" (Tearing)

Wenn man auf die Einstellungstaste tippt, sieht man manchmal für den Bruchteil einer
Sekunde eine Reißkante im Bild. **Das ist kein Fehler dieser Konfiguration, sondern
eine Eigenschaft der Hardware bzw. des ESPHome-Treibers** — und es ist ein anderes
Phänomen als das weiter oben beschriebene Netzteil-Flackern.

**Ursache.** Ein paralleles RGB-Panel hat keinen eigenen Bildspeicher. Der ESP32-S3
schiebt den Bildinhalt per DMA fortlaufend aus einem Framebuffer im PSRAM zum Panel
hinaus — bei 14MHz Pixeltakt etwa 34-mal pro Sekunde, also alle ~29ms einmal komplett
von oben nach unten. Im ESPHome-Treiber `mipi_rgb` ist dabei fest verdrahtet:

- `config.num_fbs = 1` — es gibt **genau einen** Framebuffer, kein Double Buffering.
- `write_to_display_()` ruft `esp_lcd_panel_draw_bitmap()` **sofort** auf, sobald LVGL
  etwas neu gezeichnet hat. Eine Synchronisation auf den Bildanfang (VSYNC) findet
  nicht statt; eine `on_vsync`-Callback-Registrierung
  (`esp_lcd_rgb_panel_register_event_callbacks`) kommt im Treiber überhaupt nicht vor.

LVGL schreibt also mitten in genau das Bild hinein, das gerade zur Anzeige
hinausgeschoben wird. Ist der Wechsel auf die Einstellungsseite halb fertig, während
der Bildstrahl schon in der Bildmitte steht, sieht man oben die neue und unten noch die
alte Seite — die sichtbare Reißkante. Bei einem Seitenwechsel wird das gesamte Bild
(800×480) auf einen Schlag neu aufgebaut, deshalb fällt es genau dort am meisten auf.

**Was nicht die Ursache ist:** eine Umschaltanimation. Der Seitenwechsel läuft bereits
ohne (`LV_SCR_LOAD_ANIM_NONE`), ist also ein einziger Neuaufbau und nicht viele
hintereinander. Über die YAML-Konfiguration ist an dieser Stelle nichts zu holen.

**Warum es hier nicht behoben wurde.** Technisch ginge es, aber nur mit einer
gepatchten Kopie der `mipi_rgb`-Komponente als lokale `external_components` — mit der
Folge, dass man den Patch bei jedem ESPHome-Update nachpflegen muss. Zwei Ansätze
wären denkbar:

1. **Auf VSYNC warten, dann kopieren** (~20 Zeilen): `on_vsync`-Callback registrieren,
   der ein Semaphor freigibt, und in `write_to_display_` vor dem Kopieren darauf
   warten. Kein zusätzlicher Speicherbedarf, kostet bis zu ~29ms Latenz pro
   Aktualisierung. Das ist allerdings ein Wettrennen mit dem Bildstrahl und keine
   Garantie: 768 KB in den PSRAM zu kopieren dauert einen erheblichen Teil eines
   Frames. Für kleine Aktualisierungen (Uhrzeit-Ziffern) wäre es eine echte Lösung,
   ausgerechnet für den Vollbild-Seitenwechsel eher nicht.
2. **Echtes Double Buffering** (`num_fbs = 2`, 2 × 768 KB PSRAM von 8 MB verfügbar):
   konstruktionsbedingt reißfrei. Haken: mit zwei Framebuffern schreibt
   `esp_lcd_panel_draw_bitmap` in den hinteren Puffer und tauscht, hält die beiden aber
   **nicht** synchron. Teilaktualisierungen würden zwischen zwei Bildständen
   hin- und herspringen, solange LVGL nicht jedes Mal das ganze Bild neu zeichnet
   (`full_refresh = 1`) — das setzt die ESPHome-LVGL-Komponente nicht. Man müsste also
   **zwei** Komponenten patchen.

Ein einzelner Riss bei einem bewusst ausgelösten Seitenwechsel ist für ein RGB-Panel
mit einem Framebuffer schlicht normal. Der Aufwand und das Risiko der beiden Patches
standen dafür nicht im Verhältnis.

**Kleine, kostenlose Milderung:** `pclk_frequency` auf dem Hardware-Referenzwert
`16MHZ` belassen. Das verkürzt einen Bildaufbau von ~29ms auf ~25,6ms, das Zeitfenster
für einen Riss wird entsprechend kleiner.

**Am Rande, für spätere Fehlersuche:** ESPHome ruft in `MipiRgb::loop()` bei *jedem*
Durchlauf `esp_lcd_rgb_panel_restart()` auf — eine eingebaute Selbstheilung gegen
dauerhaft verrutschte Bilder. Wer deshalb auf die Idee kommt, zusätzlich die
IDF-Option `CONFIG_LCD_RGB_RESTART_IN_VSYNC` zu setzen, sollte das vorher genau prüfen:
die beiden Mechanismen greifen sich vermutlich gegenseitig ins Lenkrad.
