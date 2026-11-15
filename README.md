# X4 HA Remote

**Xteink X4 als E-Ink-Fernbedienung für Home Assistant – mit ESPHome.**

*English summary: ESPHome firmware and a Home Assistant template sensor that turn the Xteink X4 e-reader (ESP32-C3, 4.26" e-paper) into a battery-friendly remote: browse rooms, switch lights and plugs, check windows, control music and watch your solar output. Rooms and devices are discovered automatically from Home Assistant areas – no entity IDs to configure. Documentation is in German.*

---

## Was es kann

Drei Ebenen, mit der Back-Taste durchschaltbar:

- **Räume**
  - Fenster-Übersicht: alle Räume mit Fenster- oder Türsensoren auf einen Blick
  - pro Raum: Temperatur, Feuchte, Fensterstatus, Lampen und Steckdosen schalten (Steckdosen mit Leistung in Watt)
- **Medien:** Titel, Interpret, Play/Pause, Titel vor/zurück, Lautstärke
- **Energie:** PV-Leistung gesamt und je Modul, Tagesertrag, Akkustand

Der X4 wacht nur per Power-Taste auf und schläft nach 5 Minuten ohne Tastendruck wieder ein. Das E-Ink-Display behält im Schlaf den letzten Stand, mit Uhrzeit.

Entity-IDs musst du nicht eintragen. Räume, Lampen, Steckdosen, Sensoren und Player werden aus den **Bereichen (Areas)** in Home Assistant übernommen. Feinsteuerung läuft über **Labels**.

## Inhalt des Repositorys

| Datei | Zweck |
| --- | --- |
| `homeassistant/ha_x4_remote.yaml` | Template-Sensor `sensor.x4_remote`, sammelt alle Daten als JSON |
| `esphome/esphome_x4_remote.yaml` | ESPHome-Firmware für den X4 |
| `esphome/secrets.yaml.example` | Vorlage für WLAN-Daten und API-Schlüssel |
| `docker/docker-compose.yml` | ESPHome Device Builder als Docker-Container (für Home Assistant Container) |

## Voraussetzungen

- **Xteink X4** (ESP32-C3, mit Tasten). Der X4 Pro (ESP32-S3, Touch) wird nicht unterstützt.
- **Home Assistant** (OS oder Container). Geräte müssen Bereichen zugeordnet sein.
- **ESPHome Device Builder**: bei Home Assistant OS als Add-on, sonst per Docker (siehe `docker/`).
- **USB-C-Datenkabel** für das erste Flashen.

## Installation

### 1. Home Assistant

1. `homeassistant/ha_x4_remote.yaml` in den Home-Assistant-Konfigurationsordner kopieren (neben `configuration.yaml`).
2. In `configuration.yaml`, ganz links:
   ```yaml
   template: !include ha_x4_remote.yaml
   ```
   Gibt es schon eine Zeile `template:`, den Inhalt stattdessen an die vorhandene Template-Datei anhängen.
3. **Developer tools → YAML → Check configuration**, dann Home Assistant neu starten.
4. Unter **Developer tools → States** prüfen: `sensor.x4_remote` hat die Attribute `rooms`, `media` und `pv`.

### 2. ESPHome Device Builder

- **Home Assistant OS:** Add-on *ESPHome Device Builder* installieren.
- **Home Assistant Container:** `docker/docker-compose.yml` in einen Ordner kopieren, daneben einen Ordner `config` anlegen, dann `docker compose up -d`. Weboberfläche: `http://SERVER:6052`.

In den Konfigurationsordner des Builders kommen:

- `esphome/esphome_x4_remote.yaml`
- `secrets.yaml`, aus `secrets.yaml.example` kopiert und ausgefüllt. Den Schlüssel erzeugst du mit `openssl rand -base64 32`.

### 3. Flashen

1. X4 per USB anschließen und wach halten. Im Reader die automatische Abschaltung hochsetzen. Sonst Download-Modus: X4 aus, **Leiser + Power** halten, bei der Warnung loslassen, **Lauter** drücken.
2. Optional: Backup
   ```bash
   docker exec -it esphome python -m esptool --chip esp32c3 --port /dev/ttyACM0 read_flash 0 0x1000000 /config/x4_backup.bin
   ```
3. Im Builder: **X4 Remote → ⋮ → Install → Plug into the computer running ESPHome Device Builder**.
   Alternativ: **Manual download → Factory format** und dann mit Chrome über [web.esphome.io](https://web.esphome.io) flashen.
4. Spätere Updates: **Install → Wirelessly**. Vorher den X4 wecken oder in Home Assistant den Schalter *Wach bleiben (für Updates)* einschalten.

### 4. In Home Assistant einbinden

1. **Settings → Devices & services**: *X4 Remote* hinzufügen, den API-Schlüssel eingeben.
2. **ESPHome → X4 Remote → Configure** → **Allow the device to perform Home Assistant actions** einschalten. Ohne diese Freigabe schalten die Tasten nichts.

## Labels

| Label | Wirkung |
| --- | --- |
| `x4_ausblenden` | Entität nicht auf dem X4 zeigen |
| `x4` | nur diese Media-Player zeigen (ohne Label: alle erreichbaren) |
| `x4_pv_modul` | Leistung eines PV-Moduls bzw. Solar-Eingangs (W oder kW) |
| `x4_pv_gesamt` | PV-Gesamtleistung (optional, sonst Summe der Module) |
| `x4_pv_heute` | Tagesertrag (Umrechnung über `faktor_heute`, Standard Wh → kWh) |
| `x4_pv_akku` | Akkustand in % (optional) |

Automatisch erkannt werden pro Bereich:
- Temperatur- und Feuchtesensoren (Geräteklasse `temperature` und `humidity`)
- Fenster- und Türkontakte (`window`, `door`, `opening`, `garage_door`)
- alle Lampen und Schalter
- die Leistung einer Steckdose, über einen Sensor mit Geräteklasse `power` am selben Gerät

## Bedienung

Fronttasten von links nach rechts: **Back · OK · Links · Rechts**. Rechte Seite: **Power**, darunter **Hoch** und **Runter**. Unten zeigt das Display über jeder Taste, was sie gerade tut.

| Seite | OK | Links / Rechts | Hoch / Runter |
| --- | --- | --- | --- |
| Fenster-Übersicht | Raum öffnen | zum letzten / ersten Raum | Raum wählen |
| Raum | schalten; lang: alles im Raum aus | Raum wechseln | Gerät wählen |
| Medien | Play/Pause | Titel zurück/weiter; lang: Player wechseln | Lautstärke ±5 % |
| Energie | neu zeichnen | – | – |

**Power:** kurz drücken weckt den X4 bzw. zeichnet neu, 1 s halten schickt ihn schlafen.

## Anpassen

Alles Wichtige steht unter `substitutions:` am Anfang von `esphome_x4_remote.yaml`.

| Was | Einstellung |
| --- | --- |
| Bild steht auf dem Kopf | `rotation: 90` ↔ `270` |
| Bild gespiegelt | `mirror_x: true` ↔ `false` |
| Lage der Tasten-Markierungen | `power_y`, `up_y`, `down_y` (Pixel von oben, 1 mm ≈ 8,6 px) |
| Schlaf nach | `idle_timeout` |
| Balkenlänge PV-Modul | `pv_module_peak` (W) |
| Tasten-Schwellwerte | ADC-Lambdas unter `sensor:`; Rohwerte im Log (`x4btn: … raw=…`) |

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| `sensor.x4_remote` fehlt | Einbindung in `configuration.yaml` prüfen, Check configuration, neu starten |
| `rooms` ist `J[]` | Geräte Bereichen zuordnen |
| `/dev/ttyACM0` verschwindet nach 1 s | X4 schläft ein: Abschaltung im Reader hochsetzen oder Download-Modus |
| Tasten blättern, schalten aber nicht | *Allow the device to perform Home Assistant actions* einschalten |
| Kein Liedtitel | Player mit `media_title` wählen und ihm das Label `x4` geben (z. B. HEOS statt Denon AVR/DLNA) |
| Tagesertrag 1000-fach zu hoch | `faktor_heute` in `ha_x4_remote.yaml` prüfen |
| X4 wacht nicht mehr auf | Die GPIO13-Hold-Zeilen in `go_sleep` dürfen nicht fehlen |

**Zurück zum E-Reader:** [xteink.dve.al](https://xteink.dve.al/) in Chrome öffnen und *Flash CrossPoint firmware* wählen. Oder das eigene Backup zurückspielen:

```bash
esptool --chip esp32c3 write_flash 0 x4_backup.bin
```

## Hinweise

- Das Flashen ersetzt die Reader-Firmware. Du machst es auf eigenes Risiko.
- Gemessener Akku-Verbrauch im Deep Sleep: etwa 4 mA (laut jumpit.eu). Die Laufzeit hängt stark davon ab, wie oft der X4 geweckt wird.
- `secrets.yaml` und Firmware-Backups gehören nicht ins Repository. Die `.gitignore` schließt sie aus.

## Quellen und Dank

- [Adafruit: CircuitPython on the Xteink X4](https://learn.adafruit.com/circuitpython-on-the-xteink-x4-ereader/pinouts): Pinbelegung
- [CrazyCoder/xteink-x4-sample](https://github.com/CrazyCoder/xteink-x4-sample): ADC-Werte der Tasten
- [jumpit.eu](https://jumpit.eu/en/practice/xteink-x4-home-assistant-display/): ESPHome auf dem X4, GPIO13-Hold für Deep Sleep
- [ESPHome epaper_spi](https://esphome.io/components/display/epaper_spi/): Displaytreiber SSD1677
- [CrossPoint / CrossInk](https://upd.dev/uxjulia/CrossInk): offene Reader-Firmware

## Lizenz

MIT, siehe [LICENSE](LICENSE).
