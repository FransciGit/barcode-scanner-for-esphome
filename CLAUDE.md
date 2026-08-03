# Barcode-Scanner für ESPHome

ESPHome-Konfiguration für einen Barcode-Scanner, der gescannte Produkte auf die
Einkaufsliste in Home Assistant setzt. Zu jeder EAN werden Hersteller und
Produktname über OpenFoodFacts bzw. OpengtinDB ermittelt und als Sensoren
zurückgeliefert.

Dies ist ein Fork von `SmartHome-yourself/barcode-scanner-for-esphome`.

## Hardware

- D1 Mini ESP32 (`board: wemos_d1_mini32`)
- Barcode-Scanner GM60 (UART, TX `GPIO09` / RX `GPIO10`)
- OLED-Display SSD1306 0,96" (I2C, SDA `GPIO21` / SCL `GPIO22`, Adresse `0x3C`)
- Aktiver Piezo-Buzzer (`GPIO16`)

## Die beiden Dateien

| Datei | Zweck |
|---|---|
| `barcode.yaml` | Die vollständige Konfiguration. Wird zur Laufzeit über GitHub eingebunden — auch von meinem eigenen Gerät. |
| `barcode_template.yaml` | Vorlage für das kurze Stück, das in ESPHome angelegt wird. Bindet `barcode.yaml` als Package ein. |

Die tatsächlich benutzte Gerätedatei liegt im ESPHome-Ordner der
Home-Assistant-Installation, nicht hier. Sie ist aus `barcode_template.yaml`
entstanden, zeigt aber auf diesen Fork (siehe unten).

## ⚠️ Wichtigste Regel: `main` ist live

Die echte Gerätekonfiguration liegt **nicht in diesem Repository**, sondern im
ESPHome-Ordner der Home-Assistant-Installation. Sie bindet `barcode.yaml` so ein:

```yaml
packages:
  smarthomeyourself.barcodescanner:
    url: https://github.com/FransciGit/barcode-scanner-for-esphome
    ref: main          # ← der Live-Branch
    files: [barcode.yaml]
    refresh: 0s        # ← bei jedem Build neu geholt
```

**Daraus folgt:**

- `refresh: 0s` heißt: ESPHome holt die Datei bei **jedem** Build frisch. Ein Push
  auf `main` ist damit beim nächsten Build sofort auf dem Gerät.
- Ein fehlerhafter Commit auf `main` = fehlgeschlagener Build beim nächsten
  Flashen. Es gibt keine Versionierung als Sicherheitsnetz.
- Änderungen deshalb immer auf einem Branch entwickeln und erst nach bewusster
  Entscheidung nach `main` mergen.
- Nach dem Merge muss in ESPHome neu gebaut und geflasht werden. Das passiert
  nicht von allein.

**Offener Verbesserungsvorschlag:** Statt `ref: main` einen festen Tag verwenden
(`ref: v1.1`). Dann landet nicht mehr jeder Commit automatisch auf der Hardware,
sondern Updates werden zur bewussten Entscheidung.

## OTA: nicht am `ota:`-Block sparen

Seit ESPHome 2024.6 ist `ota:` eine **Liste von Verfahren**. Ein leeres `ota:`
richtet keines mehr ein — das Hochladen scheitert dann mit
`Cannot upload via web_server OTA`. Der Block muss lauten:

```yaml
ota:
  - platform: esphome
```

Wird das je entfernt, lässt sich das Gerät nur noch **per USB-Kabel** flashen: Der
OTA-Empfänger fehlt dann in der laufenden Firmware, und er lässt sich nicht per
Funk nachrüsten, wenn genau er fehlt. Symptom in dem Fall:
`Connection refused` auf Port 3232.

Fürs Flashen per Kabel das **Factory-Format** (`firmware.factory.bin`) verwenden,
nicht die `.ota.bin`.

## Abhängigkeiten in Home Assistant

Die Automation **„Barcode-Scan auf Alexa-Einkaufsliste"** hängt an den Sensoren
dieses Geräts. Vor Umbenennungen oder Änderungen an den Sensoren prüfen, ob sie
davon betroffen ist.

Entitäten des Geräts:

- `sensor.barcode_scanner_9809d0_barcode_scanner_ean` — die gescannte EAN
  (nach dem Booten: `Scan Barcode`)
- `sensor.barcode_scanner_9809d0_barcode_scanner_brand` — Hersteller
- `sensor.barcode_scanner_9809d0_barcode_scanner_product` — Produktname
- `switch.barcode_scanner_9809d0_barcode_scanner_buzzer` — Buzzer

### Nebenbefund: veralteter Verweis in `barcode.yaml`

Zeile 40 enthält noch die Adresse des Originalprojekts:

```yaml
dashboard_import:
  package_import_url: github://SmartHome-yourself/barcode-scanner-for-esphome/barcode.yaml@main
```

Das betrifft nur den Übernahme-Dialog von ESPHome („Gerät adoptieren"), nicht die
laufende Konfiguration — harmlos, aber inkonsistent. Ein weiteres Gerät, das über
diesen Weg eingerichtet wird, bekäme die Version des Originals.

## Weitere Regeln

- `substitutions` sind die Schnittstelle zur Gerätekonfiguration. Namen **nicht**
  umbenennen oder entfernen — das bricht die Datei in Home Assistant. Neue
  Substitutions immer mit sinnvollem Standardwert versehen.
- Bei Änderungen an den Sensoren prüfen, ob Home-Assistant-Automationen davon
  abhängen könnten.
- Nach einem Push muss in ESPHome neu gebaut und geflasht werden. Das passiert
  nicht von allein.

## APIs

1. **OpenFoodFacts** wird zuerst gefragt. Kostenlos, nur Lebensmittel, bessere Daten.
2. **OpengtinDB** als Rückfallebene. Braucht eine QueryID.
   Der Standardwert `400000000` ist die öffentliche Demo-ID — die teilen sich alle
   Nutzer, begrenzt auf 500 Abfragen pro Tag. Diesen Wert nicht durch eine echte
   ID ersetzen und einchecken.

## Home-Assistant-Dienst

`esphome.[GERÄTENAME]_request_ean` löst dieselbe Abfrage aus wie ein echter Scan.
Parameter: `eancode`.

## Prüfen vor dem Commit

```bash
esphome config barcode.yaml
```

Das validiert die Konfiguration, ohne zu flashen. Bei Änderungen an der Logik
zusätzlich auf echter Hardware testen — YAML-Validierung findet keine Denkfehler
in Lambdas oder Skripten.

## Konventionen

- Kommentare und Dokumentation auf Deutsch, passend zum bestehenden Bestand.
- Der Block bis `# ---- CONFIG END ----` in `barcode.yaml` ist der Teil, den
  Nutzer anpassen. Alles darunter ist Innenleben.
