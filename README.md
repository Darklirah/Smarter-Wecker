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
`esphome`-CLI): `smart-wecker-waveshare.yaml` auswählen, kompilieren, per USB oder OTA
flashen.

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
Ein stufenweiser Verlauf aller 26 Entwicklungsschritte steht in
[CHANGELOG.md](CHANGELOG.md).

## Dateien in diesem Projekt

- `smart-wecker-waveshare.yaml` — finale, zuletzt bestätigte Haupt-ESPHome-Konfiguration
- `smart-wecker-waveshare-24.yaml`, `-26.yaml` — die beiden letzten, noch nicht final
  bestätigten Zwischenstufen (siehe [CHANGELOG.md](CHANGELOG.md) für den vollständigen
  Verlauf aller 26 Stufen; die Zwischendateien 01–23 und 25 wurden nach Abschluss der
  jeweiligen Schritte wieder entfernt, ihr Inhalt ist im CHANGELOG zusammengefasst und
  in der Git-Historie enthalten)
- `Waveshare_Swirch.yaml` — vom Nutzer beigesteuerte, verifizierte Hardware-Referenz
  ("ESPHome Designer"-Export)
- `BILDER/` — Fotos des laufenden Geräts
- `Docs/hardware-specs.md` — recherchierte Hardware-Spezifikationen mit Quellen
- `Docs/quellen.md` — Liste aller verwendeten Quellen
- `Docs/waveshare-esp32-s3-touch-lcd-7.yaml` — Kopie des ursprünglich eingebundenen
  (inzwischen nicht mehr genutzten) Community-Pakets, zur Referenz
- `Docs/Waveshare_Swirch_verifiziert.yaml` — Kopie der verifizierten Hardware-Referenz
- `Docs/ESPHome-Lessons-Learned.md` — projektübergreifende ESPHome-Lektionen
- `secrets.yaml` — deine lokalen Zugangsdaten (nicht Teil des Repos, siehe oben)
