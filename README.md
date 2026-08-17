# Home-Solar-Bat-Inv-Tibber

Preisoptimierte Nulleinspeisung für einen Soyosource-GTN-Batteriewechsel-
richter (ESPHome + [`esphome-soyosource-gtn-virtual-meter`](https://github.com/syssi/esphome-soyosource-gtn-virtual-meter),
JK-BMS über [`esphome-jk-bms`](https://github.com/syssi/esphome-jk-bms)) mit
dynamischem Tibber-Stromtarif.

## Ausgangslage

- Die Batterie kann **ausschließlich von den Solarpanelen** geladen werden –
  es gibt kein Laden aus dem Netz.
- Der Wechselrichter darf **nicht ins Netz exportieren**
  ("Nulleinspeisung"): Der `soyosource_virtual_meter` gleicht seine Ausgabe
  laufend am Hausnetz-Leistungssensor (Tibber Pulse, `powermeter0` im
  ESPHome) aus, mit einem `buffer`-Wert (W) als Sicherheitsmarge gegen
  versehentlichen Export. Die maximale Einspeiseleistung begrenzt der
  Wechselrichter (`max_power_demand`) bereits selbst – dieses Repo verwaltet
  diesen Wert nicht.
- Die Inverter-Einstellungen (`manual_mode`, `buffer`) werden intern im
  Flash/EEPROM des ESP32 gespeichert (`restore_value: true`) – **zu
  häufiges Umschalten beschädigt das EEPROM**. Diese Werte dürfen also nur
  sparsam und mit Mindestabständen verändert werden, keinesfalls bei jedem
  Preis-Tick.
- Ziel: Strompreis-Kosten minimieren UND möglichst wenig Energie
  verschwenden (Akku soll nie 100 % SoC erreichen).

## Warum "billig laden, teuer entladen" hier anders funktioniert

Beim klassischen Batteriemanagement mit Netzladefähigkeit (wie im
Schwesterprojekt [`twizy-ladeberechnung`](https://github.com/DasPoseidon/twizy-ladeberechnung)
für die Twizy-Ladesteuerung) kauft man Energie in günstigen Stunden ein.
Das geht hier nicht – die Batterie füllt sich ausschließlich über die Sonne,
unabhängig vom Strompreis. Wirtschaftlich sinnvoll ist daher nicht "wann
laden", sondern **"wann die ohnehin vorhandene, tagsüber nachwachsende
Solarenergie einsetzen, um möglichst teure Netzstunden zu verdrängen"** –
eine begrenzte Energiemenge auf die teuersten Stunden verteilen, ohne dabei
den Akku bei niedrigem Füllstand vorzeitig leerzuräumen und ohne ihn bei
hohem Füllstand sinnlos voll (und damit überlaufend/verschwendend) werden
zu lassen.

## Architektur

Bewusst ohne Blueprint (anders als beim Twizy-Projekt): Diese
Automatisierungen sind für genau **einen** Wechselrichter gedacht, keine
Mehrfachinstanziierung nötig – daher ein einziges Home-Assistant-**Package**
mit festen Entity-IDs, das Helper, Energie-Sensoren und Automatisierungen in
einer Datei bündelt (Standard-HA-Bordmittel, keine zusätzliche
Laufzeitumgebung wie AppDaemon/pyscript/NodeRED nötig).

```
┌─────────────────┐
│ Tibber-Preise    │──tibber.get_prices──┐
└─────────────────┘                      ▼
┌─────────────────┐        ┌───────────────────────────────────┐
│ Akku-Spannung     │──────▶│ automation:                        │
│ (JK-BMS)          │        │ Nulleinspeisung: Preis-/SoC-        │
└─────────────────┘        │ Steuerung (alle 10 Min.)            │
┌─────────────────┐        │                                      │
│ Solarprognose     │──────▶│ 0) SoC = Fake-SoC aus Spannung       │
│ (forecast.solar)  │        │    (BMS-SoC wird nicht genutzt)     │
└─────────────────┘        │ 1) Preis-Perzentil (0-100%), Zeit-  │
                            │    raster (h/15min) autom. erkannt  │
                            │ 2) effektiver SoC = SoC + Prognose- │
                            │    Zuschlag (gedeckelt)             │
                            │ 3) Zielschwelle = 100 - eff. SoC    │
                            │    (5..95 begrenzt)                 │
                            │ 4) Hysterese + Mindeststandzeit +   │
                            │    Tages-Umschaltlimit (EEPROM)     │
                            │ 5) SoC-Überlauf-/Tiefentladeschutz  │
                            │    (echter SoC) übersteuert Preis   │
                            └───────────────┬──────────────────────┘
                                             │ switch.turn_on/off
                                             ▼
                            manual_mode-Schalter des Wechselrichters
                            (aus = automatische Nulleinspeisung
                             des soyosource_virtual_meter aktiv)
                                             │
                                             ▼
                            buffer-Number des Wechselrichters
                            (normal/aggressiv, eigene Hysterese,
                             eigene Mindeststandzeit)
```

Die tatsächlichen Entity-IDs (mit dem ESPHome-Geräteprefix `esp_twizygarage_`,
siehe unten) stehen im Detail-Abschnitt und im Kopf von
`packages/nulleinspeisung.yaml`.

Die maximale Einspeiseleistung (`number.esp_twizygarage_inverter_max_power_demand`)
wird von diesem Repo bewusst **nicht** verwaltet – das begrenzt der
Wechselrichter/`soyosource_virtual_meter` bereits selbst; der Wert ist
einfach einmalig manuell zu setzen und bleibt dank `restore_value: true` im
ESPHome dauerhaft erhalten.

## Funktionsweise im Detail

### SoC-Berechnung: Fake-SoC statt BMS-SoC (`input_number.fakesoc`)

Als "SoC" verwendet die gesamte Steuerung **nicht** den vom JK-BMS gemeldeten
Wert (`sensor.esp_twizygarage_akku_state_of_charge`), sondern einen aus der
Akkuspannung berechneten **Fake-SoC** (`input_number.fakesoc`), den die
Automatisierung bei jedem Durchlauf (alle 10 min) neu berechnet und
schreibt:

```
fakesoc = floor((Spannung_Akku − 46) × 8,6956522)   # 46 V = 0 %, 57,5 V = 100 %
```

linear zwischen 46 V (0 %) und 57,5 V (100 %), auf 0–100 geklemmt. Als
Spannungsquelle dient `sensor.esp_twizygarage_akku_total_voltage` (JK-BMS).
Der BMS-SoC bleibt als eigener Sensor weiterhin verfügbar (z. B. zum
Vergleich im Dashboard), fließt aber in keine Entscheidung mehr ein.

### Ein/Aus der Nulleinspeisung (`switch.esp_twizygarage_inverter_manual_mode`)

Der `soyosource_virtual_meter` hat zwei Betriebsarten:
- **`manual_mode` aus** → automatische Nulleinspeisungs-Regelung aktiv: Der
  Wechselrichter gleicht seine Ausgabe laufend an den Hausverbrauch an
  (`power_demand` minus `buffer`), um Netzexport zu vermeiden.
- **`manual_mode` an** → feste Ausgabe (`manual_power_demand`, hier auf 0 W
  gehalten) unabhängig vom Hausverbrauch → effektiv "aus".

Der aktuelle Tibber-Preis wird als **Perzentil** (0–100 %) unter allen
bekannten Preisen von heute+morgen eingeordnet (100 % = teuerster bekannter
Zeitabschnitt), analog zur Preisabfrage im Twizy-Projekt per
`tibber.get_prices` (kein eigener Preis-Cache nötig, die Tibber-Integration
ruft die eigentliche API ohnehin nur ca. 1x/Tag intern ab).

**Preis-Zeitraster (Stunden- oder Viertelstundenpreise):** Manche
Tibber-Verträge rechnen viertelstündlich statt stündlich ab (PTU). Das
Zeitraster wird deshalb bei jedem Durchlauf automatisch aus dem Abstand
zwischen den von `tibber.get_prices` gelieferten Preiseinträgen erkannt
(900 s = Viertelstundenpreise, 3600 s = Stundenpreise) – exakt das gleiche
Verfahren wie im Schwesterprojekt
[`twizy-ladeberechnung`](https://github.com/DasPoseidon/twizy-ladeberechnung)
("Auto-detect Tibber price resolution"). Keine Einstellung nötig, und ein
späterer Wechsel des Abrechnungszeitrasters durch Tibber wird automatisch
mitgezogen. Ohne diese Erkennung würde ein hart auf Stunden fest verdrahtetes
Bucketing bei einem Viertelstunden-Vertrag vier 15-Minuten-Preise in einen
Stunden-Bucket zusammenfassen und für "der aktuelle Preis" immer nur den
ersten der vier Werte liefern statt den tatsächlich gerade gültigen. Das
erkannte Zeitraster ist zur Kontrolle im Status-Helfer
(`input_text.nulleinspeisung_status`) sichtbar ("Preisraster …min").

Die **Zielschwelle** ergibt sich aus dem aktuellen Akkustand:
`Zielschwelle = 100 − effektiver SoC` (auf 5–95 begrenzt). Beispiele:
- SoC 20 % → Schwelle 80 % → nur in den teuersten ~20 % der Stunden wird
  entladen, der Rest wird für wirklich teure Stunden aufgespart.
- SoC 80 % → Schwelle 20 % → in ~80 % der Stunden wird entladen, damit der
  Akku nicht sinnlos voll bleibt.

Damit regelt sich das System selbst ein: Ein hoher SoC macht das System
"großzügiger" beim Entladen (Verschwendungsvermeidung), ein niedriger SoC
"geiziger" (Ersparnis maximieren).

**Solarprognose (`forecast.solar`) als Zuschlag auf den SoC:** Der reine
Akku-SoC allein behandelt zwei Situationen mit gleichem Füllstand gleich,
obwohl sie wirtschaftlich unterschiedlich sind – an einem sonnigen
Vormittag füllt sich der Akku ohnehin bald weiter nach, an einem
bewölkten Tag ist der aktuelle SoC praktisch das gesamte Tagesbudget.
Deshalb wird der **effektive SoC** wie folgt berechnet:

```
Prognose-Zuschlag (%pt) = min(100 × Prognose-Rest-heute (kWh) / Batteriekapazität (kWh),
                               max. Einfluss (Default 40 %pt))
effektiver SoC = min(SoC + Prognose-Zuschlag, 100)
```

- Nur der **für den Rest des heutigen Tages** prognostizierte Ertrag fließt
  ein (z. B. `sensor.energy_production_today_remaining` von forecast.solar,
  Entity-ID über `input_text.nulleinspeisung_solarprognose_entity_id`
  einstellbar – der tatsächliche Name hängt von der forecast.solar-
  Instanzkonfiguration ab, unter Entwicklerwerkzeuge → Zustände prüfen).
  Die Prognose für **morgen** wird bewusst NICHT einbezogen: Wetterprognosen
  werden mit zunehmendem Zeithorizont unzuverlässiger, und morgen wird die
  Zielschwelle ohnehin mit einer dann aktuelleren Prognose neu berechnet.
- Der Zuschlag ist **gedeckelt** (Default max. 40 Prozentpunkte,
  `input_number.nulleinspeisung_prognose_max_einfluss_prozentpunkte`), damit
  eine untypisch hohe Prognose (z. B. Sensor-/API-Ausreißer) nicht sofort
  zu maximaler Entladebereitschaft führt.
- Der Zuschlag kann die Entladebereitschaft nur **erhöhen**, nie verringern
  – bei fehlender/wenig Prognose (z. B. abends) verhält sich das System wie
  zuvor rein SoC-basiert.
- Über `input_boolean.nulleinspeisung_solarprognose_nutzen` komplett
  abschaltbar.
- **Wichtig:** Der Überlauf- und der Tiefentladeschutz (siehe unten)
  arbeiten bewusst mit dem **echten** SoC, nicht mit dem prognose-erhöhten
  effektiven Wert – eine zu optimistische Prognose darf diese
  Sicherheitsschwellen nicht verwässern.

**EEPROM-Schonung:** `manual_mode` wird nur umgeschaltet, wenn
- seit der letzten Änderung die konfigurierte Mindeststandzeit vergangen ist
  (Default 20 min, `input_number.nulleinspeisung_min_standzeit_minuten`),
- das Tages-Umschaltlimit noch nicht erreicht ist (Default 8x/Tag,
  `input_number.nulleinspeisung_max_toggles_pro_tag`),
- und die Zielrichtung außerhalb einer Hysterese-Zone um die Zielschwelle
  liegt (Default ±4 Perzentil-Punkte,
  `input_number.nulleinspeisung_hysterese_prozentpunkte`) – innerhalb der
  Zone bleibt der aktuelle Zustand einfach bestehen, statt bei jeder
  kleinen Preisschwankung zu kippen.

Zwei Schutzmechanismen dürfen Standzeit **und** Tageslimit übersteuern
(werden aber trotzdem gezählt, sichtbar im Dashboard):
- **Überlaufschutz** (Default ab 95 % SoC): erzwingt Nulleinspeisung EIN,
  unabhängig vom Preis.
- **Tiefentladeschutz** (Default ab 8 % SoC): erzwingt Nulleinspeisung AUS,
  unabhängig vom Preis. (Ergänzt, überschneidet sich aber nicht mit den
  hardwareseitigen JK-BMS-Schutzschwellen – die bleiben die eigentliche
  letzte Sicherheitsebene.)

### Puffer (`number.esp_twizygarage_inverter_buffer`)

Der Puffer ist primär eine Sicherheitsmarge gegen Netzexport, kein direkter
Preis-Hebel – er wird deshalb bewusst NICHT mit dem Preis, sondern nur mit
dem SoC verknüpft, mit eigener Hysterese und eigener Mindeststandzeit
(Default ebenfalls 20 min): Ab einer SoC-Schwelle (Default 95 %,
`input_number.nulleinspeisung_puffer_soc_ein`) wird die Sicherheitsmarge auf
einen "aggressiven" Wert reduziert (Default 0 W statt 50 W) – die
Nulleinspeisungsregelung fährt dann näher an die exakte Zielleistung heran,
statt konservativ Reserve zu lassen, um den Akku vor dem Überlaufen noch
etwas stärker zu entlasten. Sobald der SoC wieder unter eine niedrigere
Schwelle fällt (Default 85 %, `input_number.nulleinspeisung_puffer_soc_aus`),
wird auf den normalen Puffer zurückgestellt. **Bewusst kein negativer
Puffer** (der laut Komponenten-Doku aktiv Netzexport erzeugen würde) – das
widerspräche der Nulleinspeisungs-Vorgabe.

### Bekannte physikalische Grenze

Wenn der Hausverbrauch über längere Zeit sehr niedrig ist (z. B. niemand
zuhause), kann die Batterie trotz maximal aggressiver Entladeeinstellung
Richtung 100 % SoC laufen – die Nulleinspeisung darf per Definition nicht
mehr als den Hausverbrauch abdecken, ein tatsächlicher Export ins Netz ist
ja gerade nicht gewollt. Das ist eine physikalische Grenze, keine
Automatisierungslücke: Eine echte Lösung dafür (Solar-Ertrag aktiv drosseln
oder eine Dump-Load) läge außerhalb dieses ESPHome/dieser Automatisierung.

## Energie-Dashboard-Sensoren

Zwei neue Sensoren (Riemann-Summe/Integration der vorhandenen
Leistungssensoren, in kWh) für den Home-Assistant-**Energie**-Bereich unter
"Batteriesysteme":
- `sensor.akku_energie_geladen` – integriert `sensor.esp_twizygarage_akku_charging_power`
- `sensor.akku_energie_entladen` – integriert `sensor.esp_twizygarage_akku_discharging_power`

Der Netzbezug wird bereits über die vorhandene Tibber-Pulse-/Smartmeter-
Integration (`sensor.ltibber_0100100700ff`) im Energie-Dashboard erfasst und
muss dort nur einmal als Netzbezugssensor eingetragen werden – dieses
Package fügt keinen zusätzlichen Netzsensor hinzu. Eine echte
Solarertrags-Aufschlüsselung (getrennt von "was in die Batterie floss") ist
mit den vorhandenen ESPHome-Sensoren nicht möglich, da kein eigener
PV-Ertragssensor vorhanden ist – nur ein zusätzlicher PV-Zähler könnte das
liefern.

Falls Home Assistant die Gerätklasse der beiden neuen Sensoren nicht
automatisch als "Energie" erkennt, im Entity-Einstellungsdialog (Zahnrad-
Symbol → Gerätklasse) einmalig manuell auf "Energie" setzen, bevor sie im
Energie-Dashboard ausgewählt werden können.

## Installation

1. Voraussetzung: offizielle **Tibber**-Integration eingerichtet. Optional,
   aber empfohlen: **forecast.solar**-Integration eingerichtet (für den
   Solarprognose-Zuschlag – ohne sie verhält sich das System weiterhin rein
   SoC-basiert, siehe oben).
2. `packages/nulleinspeisung.yaml` nach `<config>/packages/` kopieren.
   Sicherstellen, dass Packages aktiviert sind:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. Die angenommenen Entity-IDs im Package (siehe Kopf der Datei) mit den
   tatsächlichen aus **Entwicklerwerkzeuge → Zustände** abgleichen und bei
   Abweichung in `packages/nulleinspeisung.yaml` anpassen (Suchen & Ersetzen
   reicht, da alle Entity-IDs dort direkt und ausschließlich verwendet
   werden – keine indirekten Helper-Referenzen wie im Twizy-Projekt, da
   keine Mehrfachinstanziierung nötig ist).
4. Home Assistant **komplett neu starten** (Einstellungen → System →
   Neu starten). Ein reines "YAML-Konfiguration neu laden" reicht NICHT
   zuverlässig: Domains mit eigenem Reload-Service (`automation`,
   `input_number`, `input_boolean`, `input_text`, `input_datetime`) werden
   damit zwar meist erfasst, die beiden neuen `sensor: platform: integration`-
   Energie-Sensoren (Riemann-Summe, siehe unten) sind aber plattform-basierte
   YAML-Sensoren, die i. d. R. nur bei einem echten Neustart neu eingerichtet
   werden. Fehlen sie danach immer noch unter Entwicklerwerkzeuge →
   Zustände, unter Einstellungen → System → Protokolle nach Fehlern zur
   Package-Datei suchen.
5. Optional: `dashboards/nulleinspeisung_dashboard.yaml` als eigenes
   Dashboard einbinden (Einstellungen → Dashboards → "+ Dashboard
   hinzufügen" → "Neues Dashboard von Grund auf erstellen" → ⋮ → Dashboard
   bearbeiten → ⋮ → Raw-Konfigurationseditor → Inhalt einfügen).
6. Unter **Einstellungen → Energie** im Bereich "Batteriesysteme"
   `sensor.akku_energie_geladen` (wird geladen) und
   `sensor.akku_energie_entladen` (wird entladen) eintragen.
7. Falls forecast.solar genutzt wird: im Dashboard unter "Einstellungen –
   Solarprognose" die tatsächliche Entity-ID des forecast.solar-
   Prognosesensors eintragen (Default-Vermutung
   `sensor.energy_production_today_remaining` – unter Entwicklerwerkzeuge →
   Zustände nach "remaining"/"heute" suchen, falls abweichend) und die
   nutzbare Batteriekapazität in kWh eintragen.
8. `number.esp_twizygarage_inverter_max_power_demand` (max. Einspeiseleistung)
   einmalig manuell auf den gewünschten Wert setzen – wird von diesem Repo
   nicht verwaltet, siehe "Architektur" oben.

## Voreinstellungen anpassen

Alle Schwellwerte sind über `input_number`-Helper im Dashboard einstellbar
(siehe `dashboards/nulleinspeisung_dashboard.yaml`, Bereich
"Einstellungen"). Sinnvolle Startpunkte sind bereits als Defaults gesetzt;
insbesondere die Mindeststandzeiten und Tages-Umschaltlimits sollten nicht
zu aggressiv verkleinert werden, um das EEPROM zu schonen.

## Repository-Struktur

```
esphome/
  esp-twizygarage.yaml            # ESPHome-Konfiguration (Referenz, unverändert)
packages/
  nulleinspeisung.yaml            # Helper, Energie-Sensoren, Automatisierungen
dashboards/
  nulleinspeisung_dashboard.yaml  # fertiges Dashboard (nur eingebaute Karten)
```

## Bekannte Vereinfachungen

- Der Fake-SoC ist eine lineare Spannungs-Näherung (46–57,5 V), keine echte
  Coulomb-Zählung – bei LiFePO4-Zellen ist die Spannungskurve im mittleren
  SoC-Bereich sehr flach, der Fake-SoC dort also ungenauer als am oberen/
  unteren Ende. Die genauen Eckwerte sind auf die konkrete Batterie
  kalibriert; bei einer anderen Zellenzahl/-chemie müssen die beiden
  Konstanten (46 V/57,5 V) in `packages/nulleinspeisung.yaml` angepasst
  werden.
- Die Preis-Perzentil-Berechnung ist ein einfacher, robuster Ansatz ohne
  echte Optimierung über den gesamten Tagesverlauf (kein Solver, keine
  PV-Ertragsprognose) – bewusst so gewählt, um mit Bordmitteln
  (Template-Automatisierung) auszukommen, wie im Schwesterprojekt.
- Die Zielschwelle `100 − effektiver SoC` ist eine lineare Heuristik, keine
  mathematisch optimale Lösung für "maximale Kostenersparnis bei nie 100 %
  SoC" – in der Praxis aber ein robuster Kompromiss, der beide Ziele in die
  richtige Richtung steuert.
- Der Solarprognose-Zuschlag nutzt nur die Rest-heute-Prognose, keine
  stundenaufgelöste Verteilung (kein Abgleich "wann genau heute kommt wie
  viel Ertrag") – dafür müsste man die stündliche `wh_hours`-Vorhersage von
  forecast.solar auswerten, was die Templates deutlich komplexer machen
  würde, ohne die Kernentscheidung (Ein/Aus jetzt) wesentlich zu verbessern.
- Kein Kosten-/Ersparnis-Sensor fürs Energie-Dashboard: Home Assistant
  unterstützt für Batteriesysteme im Energie-Dashboard nur einen statischen
  Preis oder eine feste Kosten-Entity, keine stündlich wechselnden
  Tibber-Preise – ein selbstgebauter Ersparnis-Sensor würde das nur
  näherungsweise nachbilden und wurde bewusst weggelassen, um keine
  irreführenden Zahlen zu liefern.
