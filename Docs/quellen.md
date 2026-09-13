# Quellen (Waveshare ESP32-S3-Touch-LCD-7 Projekt)

- [Waveshare Dokumentation](https://docs.waveshare.com/ESP32-S3-Touch-LCD-7) — Modul, Speicher, Peripherie-Übersicht
- [Waveshare Wiki](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-7) — CH422G-Rolle, Pinbelegung
- [Waveshare Produktseite](https://www.waveshare.com/esp32-s3-touch-lcd-7.htm)
- [HA-Community-Forum: freie/vorbelegte GPIOs](https://community.home-assistant.io/t/waveshare-esp32-s3-7-lcd-touch-screen-using-gpio-pins-that-may-be-pre-assigned/831578) — welche Pins wirklich frei sind (GPIO6, SD-Karten-Pads)
- [inytar/waveshare-esp32-s3-touch-lcd-7-esphome (GitHub)](https://github.com/inytar/waveshare-esp32-s3-touch-lcd-7-esphome) — fertiges, gepflegtes ESPHome-Paket für dieses Board (Display, Touch, Backlight, Anti-Burn-Schutz), min. ESPHome 2025.8.0/2026.8.2
  - Vollständige Kopie der Paket-YAML liegt in diesem Ordner: `waveshare-esp32-s3-touch-lcd-7.yaml` (Stand 16.09.2026, direkt von GitHub geladen)
- [ESPHome LVGL Widgets-Doku](https://esphome.io/components/lvgl/widgets/) — Roller/Switch/Checkbox/Button-Trigger- und Actions-Namen (wichtig: `on_value` statt `on_value_changed`, `on_click` als generischer Interaktions-Trigger)
- [ESPHome LVGL Switch-Integration](https://esphome.io/components/switch/lvgl.html) — `switch: platform: lvgl` für bidirektionale Bindung von Checkbox/Switch-Widgets an ESPHome-Switches (kein manuelles Event-Verdrahten nötig)
- [ESPHome LVGL Cookbook — Uhr-Beispiel](https://esphome.io/cookbook/lvgl.html#an-analog-clock) — Vorlage für die eigene digitale Uhrzeit-/Datumsanzeige per Script + `lvgl.label.update`
