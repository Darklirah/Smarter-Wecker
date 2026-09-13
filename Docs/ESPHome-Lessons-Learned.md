# ESPHome / ESP32 – Lessons Learned

> Übernommen aus einem projektübergreifenden Notizen-Ordner, der auch eine zweite
> Hardware-Variante (CrowPanel, mit Audioausgabe) desselben Smart-Wecker-Projekts
> abdeckt. Diese Kopie ist Teil des Waveshare-Repos, da alle hier dokumentierten
> Erkenntnisse an diesem Board entdeckt wurden; einzelne Verweise auf die
> CrowPanel-Variante (`Smart-Wecker/...`) betreffen ein nicht in diesem Repo
> enthaltenes Schwesterprojekt.

Projektübergreifende technische Notizen aus den Smart-Wecker-Projekten (CrowPanel,
Waveshare), die bei künftigen ESP32/ESPHome-Projekten Zeit sparen sollen.

## PSRAM-Geschwindigkeit vs. "MSPI Timing: flash model not verified" (16.09.2026)

**Symptom:** Gerät bootet, dann sofortiger Absturz:
```
E MSPI Timing: The flash model has not been verified support this feature, please contact espressif business support
E (644) cpu_start: init function ... has failed (0x106), aborting
abort() was called at PC 0x...
```
Endlosschleife aus Boot → Absturz → Reboot.

**Ursache:** Bei **Octal-PSRAM mit hoher Taktrate (z.B. `speed: 120MHz`)** prüft ESP-IDF beim
Start den tatsächlich verbauten Flash-Chip gegen eine interne Liste bekannter, für dieses
Timing verifizierter Modelle. Ist der auf dem konkreten Board-Exemplar verbaute Flash-Chip
nicht in dieser Liste (variiert je nach Fertigungscharge, nicht vom Nutzer beeinflussbar),
bricht der Boot ab statt mit unsicherem Timing weiterzulaufen.

**Fix:** PSRAM-Geschwindigkeit reduzieren, z.B. auf `80MHz`:
```yaml
psram:
  mode: octal   # unveraendert lassen, falls von einem Community-Paket vorgegeben
  speed: 80MHz  # von z.B. 120MHz reduziert
```
Wenn ein per `packages:` eingebundenes Community-Paket bereits einen `psram:`-Block mit
`speed: 120MHz` mitbringt: einfach im eigenen Config-Teil erneut `psram: speed: 80MHz`
setzen — ESPHome merged das (eigener Wert überschreibt den Wert aus dem Paket für denselben
Schlüssel, andere Schlüssel wie `mode:` bleiben vom Paket erhalten).

**Aufgetreten bei:** Waveshare ESP32-S3-Touch-LCD-7 (Community-ESPHome-Paket von
`inytar/waveshare-esp32-s3-touch-lcd-7-esphome`, das dort `speed: 120MHz` vorgibt).

**Allgemeine Lektion:** Bei JEDEM ESP32-S3-Board mit Octal-PSRAM, das direkt nach
`I (xxx) boot: Disabling RNG early entropy source...` mit einer MSPI-Timing-Fehlermeldung
abstürzt, zuerst die PSRAM-Geschwindigkeit reduzieren, bevor an anderen Stellen gesucht wird.

---

## AUFGELÖST (16.09.2026): Der echte Absturz-Grund war NIE die I2C-Bus-Recovery

Nach systematischer Stufen-Bisektion (`smart-wecker-waveshare-01.yaml`
bis `-11.yaml`) gefunden: **`number.on_value` → `lvgl.roller.update`** (Rückkopplung
Number→Roller-Widget, für das Anzeigen eines wiederhergestellten Weckzeit-Werts) crasht
zuverlässig, weil:

1. `restore_value: true` beim `number:`-Helper laedt beim Booten den gespeicherten Wert
2. Das loest `on_value` aus -- **noch waehrend `setup()`**, sehr frueh
3. `on_value` ruft `lvgl.roller.update` auf -- aber **LVGL/der Roller ist zu diesem
   Zeitpunkt noch nicht initialisiert** (ESPHome setzt `number:` vor `lvgl:` auf)
4. Ergebnis: Zugriff auf ein nicht-existierendes LVGL-Objekt -> Speicherkorruption/Crash

Die Log-Zeile `[I][i2c.idf:205]: Performing bus recovery` gefolgt von
`[C][ch422g:025]: Initialization complete` war **immer nur die letzte real ausgefuehrte
Log-Ausgabe vor dem Crash**, weil i2c/ch422g in der Komponenten-Reihenfolge kurz vor
number/lvgl kommen -- keine tatsaechliche Verbindung zum I2C-Treiber. Die urspruengliche
Vermutung (offener ESPHome-Bug esphome/esphome#15356, generelle I2C-Bus-Recovery-
Instabilitaet) war eine Fehlspur, mangels vollstaendigem Backtrace zunaechst plausibel,
aber durch die Bisektion widerlegt.

**Erster Fix-Versuch (Stufe 12, nur TEILWEISE erfolgreich):** `esphome: on_boot: -
priority: -100` + ein Global-Bool `ui_ready`, der `number.on_value` erst nach LVGL-Setup
scharf schaltet. Das behob zwar den sofortigen `setup()`-Crash, aber ~30s spaeter (genau
beim NTP-Sync) trat ein NEUER Crash mit echtem Backtrace (`Guru Meditation ...
LoadProhibited`) auf -- vermutlich weil `lvgl.roller.update` beim programmatischen Setzen
selbst ein `on_value`-Event am Roller ausloest, das wieder `number.set` aufruft, was
(sobald `ui_ready` true ist) erneut `lvgl.roller.update` ausloest -> verzoegerte
Rueckkopplungsschleife. Zeigt: das Zurueckschreiben von `number` nach `lvgl.roller` im
laufenden Betrieb ist grundsaetzlich fragil, nicht nur beim fruehen `setup()`.

**Finaler, bestaetigter Fix (Stufe 13):** Die Rueckkopplung Number->Roller komplett
weglassen. Der `number:`-Helper ist nur noch ein reiner Wertespeicher (`restore_value:
true`, KEIN `on_value`-Handler). Der Roller wird stattdessen nur EINMALIG dann per
`lvgl.roller.update` aus dem aktuellen `number`-Zustand gesetzt, wenn die Seite mit dem
Roller tatsaechlich geoeffnet wird -- z.B. im `on_click:` des Buttons, der zur
Einstellungsseite navigiert, VOR dem `lvgl.page.show:`:
```yaml
on_click:
  then:
    - lvgl.roller.update:
        id: roller_hour
        selected_index: !lambda "return (int) id(alarm_hour).state;"
    - lvgl.roller.update:
        id: roller_minute
        selected_index: !lambda "return (int) (id(alarm_minute).state / 5);"
    - lvgl.page.show: settings_page
```
Die andere Richtung (Roller -> `number.set` bei `roller.on_value`) bleibt unveraendert und
war nie das Problem. Getestet und vom Nutzer bestaetigt: kein Crash mehr, Aenderungen am
Roller funktionieren, und beim Oeffnen der Einstellungsseite wird korrekt der
gespeicherte/aktuelle Wert angezeigt.

**Allgemeine Lektion:** `number.on_value` (oder aehnliche restaurierbare Helper) sollte
NIE direkt ein `lvgl.*.update` aufrufen, weder ungeschuetzt (crasht in `setup()`) noch mit
einem Ready-Guard (kann zu einer verzoegerten Rueckkopplungsschleife im laufenden Betrieb
fuehren). Stattdessen den Widget-Zustand nur "on demand" beim tatsaechlichen Anzeigen
(Seitenwechsel/Button-Klick) aus dem `number`-Zustand ziehen -- pull statt push.

**Betraf vermutlich auch die CrowPanel-Variante** (separates Schwesterprojekt, nicht Teil
dieses Repos), die denselben `number.on_value` -> `lvgl.roller.update`-Muster verwendete.

---

## ESPHome-Schrift: `glyphs:` ohne `glyphsets:` kann den kompletten Basis-Zeichensatz loeschen (16.09.2026)

**Symptom:** Nach dem Hinzufuegen deutscher Umlaute zu einer `gfonts://`-Schrift per
```yaml
font:
  - file: "gfonts://Roboto"
    id: font_large
    size: 40
    glyphs: "äöüÄÖÜß"
```
zeigt das Display ploetzlich GAR KEINEN Text mehr an -- alle Labels/Buttons erscheinen nur
noch als leere, umrandete Kaestchen (Rahmen/Hintergrund der Widgets sind da, aber jeglicher
Text fehlt, auch normale Buchstaben/Ziffern wie die Uhrzeit).

**Ursache:** Die ESPHome-Doku beschreibt `glyphs:` als "in addition to the characters
defined by the glyphsets option" (Standard-glyphsets: `GF_Latin_Kernel`, nur englische
Basis-Zeichen). In der Praxis hat sich gezeigt, dass ein alleinstehendes `glyphs:` (ohne
explizit gesetztes `glyphsets:`) dazu fuehrt, dass die Schrift am Ende NUR die manuell
angegebenen Zeichen enthaelt -- der komplette Standard-Zeichensatz (Buchstaben, Ziffern,
Satzzeichen) fehlt dann. Nicht mit "expected a dictionary" o.ae. abgefangen, kompiliert
anstandslos durch, faellt erst am Geraet auf.

**Fix:** Statt `glyphs:` zu ergaenzen, explizit ein groesseres `glyphsets:` setzen, das die
benoetigten Sonderzeichen bereits einschliesst -- fuer Deutsch/Europa:
```yaml
font:
  - file: "gfonts://Roboto"
    id: font_large
    size: 40
    glyphsets:
      - GF_Latin_Core   # deckt europaeische Sprachen inkl. Umlauten/ß ab
```
Das ist auch der von ESPHome selbst dokumentierte Weg (Beispiel `roboto_european_core`).
`GF_Latin_Core` ist ein Superset von `GF_Latin_Kernel` und enthaelt weiterhin alle
englischen Basis-Zeichen PLUS Umlaute -- kein zusaetzliches `glyphs:` noetig.

**Allgemeine Lektion:** Bei jedem ESPHome-LVGL-Projekt mit nicht-englischen Sonderzeichen
(Umlaute, Akzente, kyrillisch, etc.) direkt `glyphsets:` mit einem passenden Set
(`GF_Latin_Core`, `GF_Cyrillic_Core`, ...) verwenden statt `glyphs:` alleinstehend zu
ergaenzen. `glyphs:` eignet sich eher fuer einzelne zusaetzliche Symbole (z.B. ein Icon aus
einer Icon-Schriftart via `extras:`), nicht fuer ganze Sprach-Zeichensaetze.

---

## KORRIGIERT (16.09.2026): Weckzeit-Reset war NICHT die on_boot-Prioritaet, sondern `flash_write_interval`

Der folgende Abschnitt (on_boot-Prioritaets-Gleichstand) beschreibt eine echte,
plausible ESPHome-Falle und der Fix (priority: -100) ist weiterhin sinnvoll -- er war
hier aber NICHT die Ursache des beobachteten Problems (nach dem Fix trat der 00:00-
Reset weiter auf).

**Tatsaechliche Ursache:** ESPHome schreibt Aenderungen an `restore_value`-Entities
nicht sofort auf den Flash, sondern sammelt sie und schreibt sie erst nach
`flash_write_interval` (Standard: **1 Minute**) -- um den Flash zu schonen (siehe
https://esphome.io/components/preferences.html). Wurde das Geraet zum Testen kurz
nach dem Einstellen der Weckzeit neu gestartet (ueblich beim Testen), war die
Aenderung schlicht noch nicht auf dem Flash gelandet -- die Wiederherstellung beim
naechsten Boot lieferte zwangslaeufig den alten/0-Wert, komplett unabhaengig von
jeglicher Komponenten- oder on_boot-Reihenfolge.

**Fix:**
```yaml
preferences:
  flash_write_interval: 0s   # sofort schreiben statt bis zu 1 Minute warten
```
Fuer selten geaenderte Werte (z.B. eine Weckzeit, ein paar Mal am Tag) ist der
zusaetzliche Flash-Verschleiss vernachlaessigbar -- die Standard-Verzoegerung ist
primaer fuer haeufig wechselnde Werte (Sensoren, Lichthelligkeit, etc.) gedacht.

**Allgemeine Lektion:** Wenn ein `restore_value`/`restore_mode`-Wert nach einem
Neustart "manchmal" oder "immer, wenn kurz vorher geaendert" auf einen alten/Default-
Wert zurueckfaellt, zuerst `flash_write_interval` pruefen/setzen, BEVOR an
Komponenten-Reihenfolge oder on_boot-Prioritaeten gesucht wird -- besonders wenn der
Test-Ablauf "Wert aendern -> sofort neu starten" ist.

---

## `esphome.on_boot` mit Standard-Prioritaet kann VOR der Wiederherstellung persistenter Werte laufen (16.09.2026)

**Symptom:** Ein `esphome: on_boot:`-Block, der beim Start einen Wert aus einer
`restore_value:true`-Entity (number/datetime/...) liest und in eine andere Entity
schreibt, sieht dort NICHT den wiederhergestellten Wert, sondern den C++-Standardwert
(z.B. 0) -- und ueberschreibt damit ggf. sogar die eigentlich schon korrekt
wiederhergestellte Quelle wieder mit 0, wenn diese Ziel-Entity ihrerseits per
`on_value` zurueckschreibt. Ergebnis: ein Wert, der eigentlich persistent sein sollte
(z.B. eine Weckzeit), faellt nach JEDEM Neustart auf einen Default-Wert zurueck --
obwohl `restore_value: true` uebrall korrekt gesetzt ist.

**Ursache:** `esphome.on_boot` hat standardmaessig `priority: 600` (siehe
https://esphome.io/components/esphome.html#on-boot). Das ist EXAKT dieselbe
Prioritaets-Stufe, mit der die meisten "normalen" Komponenten (u.a. `number:`,
`datetime:`) waehrend `setup()` eingerichtet werden -- inklusive dem Laden ihres
gespeicherten Werts aus dem Flash. Bei einem Prioritaets-**Gleichstand** ist die
Ausfuehrungsreihenfolge zwischen dem eigenen `on_boot`-Trigger und der `setup()` dieser
Komponenten NICHT garantiert. Der eigene `on_boot`-Code kann also vor oder nach der
Wiederherstellung laufen -- reine Glueckssache je nach Deklarationsreihenfolge/Build.

**Fix:** Fuer jeden `on_boot`-Block, der auf den bereits wiederhergestellten Zustand
anderer Komponenten angewiesen ist, explizit eine SEHR NIEDRIGE Prioritaet setzen:
```yaml
esphome:
  on_boot:
    priority: -100   # laeuft garantiert NACH allen anderen Komponenten (Doku: "-100.0:
                      # At this priority, pretty much everything should already be
                      # initialized.")
    then:
      - ...
```
Allgemeine Regel: Jeder `on_boot`-Block, der `id(irgendwas).state` von einer
`restore_value`/`restore_mode`-Entity liest, sollte eine niedrige (negative)
Prioritaet bekommen, es sei denn, es ist explizit gewuenscht, VOR der
Wiederherstellung zu laufen.

**Warum das nicht sofort auffiel:** Der Bug zeigt sich nicht bei jedem Boot
deterministisch als Compile- oder Laufzeitfehler, sondern nur als "der Wert ist
komischerweise nach dem Neustart weg" -- leicht mit einem generellen
Persistenz-Problem zu verwechseln, obwohl `restore_value: true` an der Entity selbst
technisch voellig korrekt war.

**How to apply:** Bei jedem ESPHome-Projekt mit eigenem `esphome.on_boot`, das auf
`id(entity).state` anderer, persistenter Entities zugreift: Prioritaet explizit auf
einen niedrigen/negativen Wert setzen (z.B. `-100`), nicht auf dem 600er-Standard
belassen.

---

## Historische, jetzt widerlegte Annahme (Verlauf, nicht mehr aktuell)

Der folgende Abschnitt beschreibt den fruehen (falschen) Verdacht "I2C Bus Recovery in
ESPHome 2026.8.2 / ESP-IDF 5.5.5" -- als Dokumentation des Untersuchungsverlaufs stehen
gelassen, siehe Korrektur weiter oben ("Der echte Absturz-Grund war NIE die
I2C-Bus-Recovery").

**Symptom:** `[I][i2c.idf:205]: Performing bus recovery` im Log, direkt danach entweder
- ein Absturz mit vollem Backtrace/Guru-Meditation-Dump, ODER
- ein Watchdog-Timeout-Hänger, ODER
- ein sofortiger Doppel-Crash **ohne jeden Backtrace** ("Panic handler entered multiple
  times. Abort panic handling.")

**Ursprünglich vermutet:** nur ein Problem der GPIO19/20-Pins (die beim ESP32-S3 mit
USB_SERIAL_JTAG geteilt sind) — siehe [esphome/esphome#15356](https://github.com/esphome/esphome/issues/15356).

**Widerlegt/erweitert:** Derselbe Fehler (`Performing bus recovery` → Crash) trat auch
auf einem **komplett anderen Board mit I2C auf GPIO8/9** auf (Waveshare
ESP32-S3-Touch-LCD-7, keinerlei USB_SERIAL_JTAG-Pin-Überschneidung möglich) — letztlich
durch die Bisektion oben endgültig als Fehlspur entlarvt (die eigentliche Ursache war
die Number→Roller-Rückkopplung, siehe oben).
