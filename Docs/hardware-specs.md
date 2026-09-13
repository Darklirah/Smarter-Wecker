# Hardware-Recherche: Waveshare ESP32-S3-Touch-LCD-7 (Basisversion, CH422G)

Stand: 16.09.2026. Alle Angaben mit Quelle, nichts geraten.

## Kernspezifikationen

| Merkmal | Wert | Quelle |
|---|---|---|
| Modul | ESP32-S3-WROOM / ESP32-S3N16R8 | [Waveshare Docs](https://docs.waveshare.com/ESP32-S3-Touch-LCD-7) |
| SRAM | 512 KB (intern) | Waveshare Docs |
| ROM | 384 KB | Waveshare Docs |
| Flash | 16 MB (gestapelt) | Waveshare Docs |
| PSRAM | 8 MB (Octal) | Waveshare Docs |
| CPU | Xtensa LX7 Dual-Core, bis 240 MHz | Waveshare Docs |
| Funk | Wi-Fi 2.4 GHz (802.11 b/g/n), Bluetooth 5 (LE) | Waveshare Docs |
| Display | 7" 800×480, 65K Farben, RGB-Parallel-Interface | Waveshare Docs |
| Touch | Kapazitiv, 5-Punkt, GT911, mit Interrupt-Unterstützung | Waveshare Docs |
| IO-Expander | **CH422G** (steuert Backlight-Enable, Touch-Reset, TF-Karten-Enable, USB/CAN-Auswahl) | [Waveshare Wiki](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-7), [ESPHome-Paket-README](https://github.com/inytar/waveshare-esp32-s3-touch-lcd-7-esphome) |

> **Wichtig zur Versionsfrage:** Es gibt neben dieser Basisversion auch eine **"-7B"-Variante**,
> die den CH422G durch einen **CH32V003**-Mikrocontroller ersetzt (andere Pin-/Backlight-
> Ansteuerung, eigene Konfiguration nötig, hier NICHT abgedeckt). Dieses Projekt ist für die
> **CH422G-Basisversion** gebaut (vom Nutzer am 16.09.2026 bestätigt).

## Kein Onboard-Audio

Diese Basisversion hat **keinen** Audio-Codec, keinen Lautsprecheranschluss, keinen
Verstärker. Nur **1 zuverlässig freier GPIO** (GPIO6, über den 3-Pin-Sensor-Header) — für
I2S-Audio bräuchte man normalerweise 3 Signal-Pins. Laut einem Waveshare-Moderator gibt es
3 weitere freie IOs auf den SD-Karten-Pads des ESP32-Moduls, aber nur durch Löten auf kleine
Modul-Pads erreichbar, exakte GPIO-Nummern nicht dokumentiert.

Quelle: [HA-Community-Forum-Thread](https://community.home-assistant.io/t/waveshare-esp32-s3-7-lcd-touch-screen-using-gpio-pins-that-may-be-pre-assigned/831578)

→ Für dieses Projekt bewusst **ohne Audioausgabe** gebaut (Nutzerentscheidung), siehe README.

## Backlight ist NUR Ein/Aus, nicht dimmbar

Das ESPHome-Community-Paket (siehe unten) implementiert die Backlight-Steuerung als
`light: platform: binary` über den CH422G-Ausgang — **kein PWM, keine Helligkeitsregelung**.
Das ist eine Hardware-/Paket-Eigenschaft dieser Platine (Backlight-Enable ist ein simples
Ein/Aus-Signal über den IO-Expander), keine Einschränkung, die ich willkürlich eingeführt
habe. Ein sanftes Aufwachlicht über die Displayhelligkeit ist mit dieser Hardware also nicht
direkt möglich.

## I2C / GT911 / CH422G Pinbelegung (aus dem verifizierten ESPHome-Paket)

| Funktion | Pin |
|---|---|
| I2C SDA | GPIO8 |
| I2C SCL | GPIO9 |
| Touch-Interrupt | GPIO4 |
| Touch-Reset | über CH422G, Pin-Nummer 1 |
| Backlight-Enable | über CH422G, Pin-Nummer 2 |
| Display-Reset | über CH422G, Pin-Nummer 3 |
| Display DE | GPIO5 |
| Display HSYNC | GPIO46 |
| Display VSYNC | GPIO3 |
| Display PCLK | GPIO7 |
| Rot-Datenpins | 1, 2, 42, 41, 40 |
| Blau-Datenpins | 14, 38, 18, 17, 10 |
| Grün-Datenpins | 39, 0, 45, 48, 47, 21 |

Quelle: [inytar/waveshare-esp32-s3-touch-lcd-7-esphome](https://github.com/inytar/waveshare-esp32-s3-touch-lcd-7-esphome/blob/main/waveshare-esp32-s3-touch-lcd-7.yaml)
(vollständige Kopie liegt auch in diesem Docs-Ordner: `waveshare-esp32-s3-touch-lcd-7.yaml`)
