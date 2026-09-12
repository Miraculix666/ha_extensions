# MASTER TASK FOR JULES: Modernization & Integration of ESPHome ATM90E32 6-Channel Power Monitor

**Assigned Agent:** Jules (`jules.google.com`)  
**Scope:** `ha_config` (`/GitHub/ha_config/esphome/power-monitor.yaml`, `/GitHub/ha_config/homeassistant/`), `ha_extensions`, `agents_and_prompts`  
**Standard:** ESPHome 2024+ Syntax, Zero Deprecation, Clean Entity Naming, GitOps Single-Source-of-Truth  
**Authorization Mode:** Fast-Track Global Auto-Acceptance (Rule 1 & Rule 15: All Jules branches/PRs are automatically approved and merged).

---

## 1. System & Hardware Context

- **Hardware:** CircuitSetup Expandable 6-Channel ESP32 Energy Meter (ESP32 NodeMCU-32s).
- **Energy Meter IC:** Microchip ATM90E32 (SPI interface: `clk: 18`, `miso: 19`, `mosi: 23`, `cs: 5`).
- **Wiring Standard (Altbau Installation):**
  - **L1** = Blau
  - **L2** = Grau (**Referenzphase** – hier ist der 9V AC-AC Netzteil-Adapter angeschlossen)
  - **L3** = Schwarz
  - **PE / Erde** = Rot
- **Active Calibration Parameters (EFK-geprüft via Wallbox-Referenz):**
  - `voltage_cal: 14240`
  - `l1_current_cal: 36700` with `l1_boost: 2.495` (Kompensation zur Umgehung des 16-Bit Register-Limits 65535)
  - `l2_current_cal: 25852` (Referenzphase L2)
  - `l3_current_cal: 27265`
- **Current File Location:** `/GitHub/ha_config/esphome/power-monitor.yaml` (gespiegelt in `/GitHub/ha_config/homeassistant/esphome/power-monitor.yaml`).
- **Core Directive:** "shelly gibt es nicht" – Alle primären Energiemessungen des Haushalts basieren ausschließlich auf diesem ESPHome ATM90E32 Monitor.

---

## 2. Work Packages for Jules

### Work Package 1: ESPHome 2024+ Hardening & Zero-Deprecation Refactoring

1. **Entity Naming De-duplication (Critical Bugfix):**
   - **Problem:** Die Konfiguration definiert `friendly_name: "Stromzähler"` im Header und benennt Sensoren wie `name: "${friendly_name} L1 Spannung"`. Home Assistant hängt den Device-Friendly-Name automatisch voran, wodurch unschöne doppelte Entity-IDs wie `sensor.stromzahler_stromzahler_l1_spannung` erzeugt wurden.
   - **Soll-Zustand:** Bereinigung der Sensornamen auf `name: "L1 Spannung"`, `name: "L1 Strom"`, `name: "Netzleistung Gesamt"` etc., sodass HA saubere IDs wie `sensor.stromzahler_l1_spannung` und `sensor.stromzahler_netzleistung_gesamt` generiert.
2. **Modern ESPHome 2024+ Component Syntax:**
   - Bereinigung veralteter Syntaxelemente im `esp32:`-Block, `sensor:`-Filtern und Template-Sensoren.
   - Sicherstellen korrekter `state_class: measurement` (oder `total_increasing` für Energie) und `device_class` (`voltage`, `current`, `power`, `energy`, `power_factor`, `frequency`).
   - Phasen-Mapping (L1=Phase A, L2=Phase B / Referenz, L3=Phase C) klar dokumentieren und stabilisieren.
3. **Robustes SPI & Update-Timing:**
   - Beibehaltung stabiler SPI-Taktung für ATM90E32.
   - Konsistentes Update-Intervall (z. B. 2s für Echtzeit-Wirkleistung, 60s für WLAN-Diagnose).

---

### Work Package 2: Mathematisch Korrekte Tibber-Kostenberechnung & HA-Integration

1. **Analyse des bestehenden Berechnungsfehlers:**
   - In `power-monitor.yaml` wird `daily_energy * tibber_current_price` im ESPHome-Template gerechnet.
   - **Fehler:** Ein dynamischer Tarif ändert sich stündlich. Multipliziert man die gesamte Tagesenergie (`daily_energy`) um 18:00 Uhr mit dem aktuellen Stundenpreis, wird der gesamte historische Tagesverbrauch rückwirkend mit dem teuren Spitzenpreis verrechnet!
2. **Architektur-Sollzustand:**
   - ESPHome liefert die unverfälschte physikalische Messung:
     - `sensor.stromzahler_netzleistung_gesamt` ($P_{tot}$ in W)
     - `sensor.stromzahler_tagesverbrauch` (kWh, `total_increasing`)
     - Neuer Lifetime-Bezugszähler `sensor.stromzahler_energie_bezug_total` (kWh, `total_increasing` ohne täglichen Reset) für das native Home Assistant Energy Dashboard.
   - **Tibber Entity Resolution:**
     - Korrekte Referenzierung des real existierenden Tibber-Sensors in HA: `sensor.electricity_price_im_beisen_12b` (bzw. Fallback `sensor.tibber_prices`).
   - **Kostenintegration in Home Assistant:**
     - Konfiguration für das Home Assistant Energy Dashboard (Grid Ingestion mit dynamischem Preisentität `sensor.electricity_price_im_beisen_12b`).
     - Optionales HA Package / Template-Sensor für die kontinuierliche Integration $\int (P(t) \cdot \text{Preis}(t)) \, dt$ (z. B. via Riemann-Summe oder Helper-Automation), damit Tageskosten auch bei dynamischen Tarifen auf den Cent genau stimmen.

---

### Work Package 3: Dediziertes Lovelace 3-Phasen Energy Card Package

Jules soll ein modulares Lovelace Dashboard / Card Package erstellen (z. B. als YAML-Include unter `/GitHub/ha_config/homeassistant/dashboards/energy_atm90e32.yaml` oder Package-Snippet):

1. **3-Phasen-Matrix (L1, L2, L3):**
   - **Spannung:** $V_{L1}, V_{L2}, V_{L3}$ (V) mit Normalbereichs-Anzeige (~230 V).
   - **Strom:** $I_{L1}, I_{L2}, I_{L3}$ (A).
   - **Wirkleistung:** $P_{L1}, P_{L2}, P_{L3}$ (W).
   - **Leistungsfaktor:** $\cos\varphi$ (oder PF in %) für jede Phase zur Erkennung induktiver/kapazitiver Blindleistung.
   - **Netzfrequenz:** $f_{Grid}$ (Hz, Soll: 50,0 Hz).
2. **Netz-Gesamtansicht & Kosten:**
   - Gesamte Wirkleistung (W) als Echtzeit-Wert.
   - Heutiger Gesamtverbrauch (kWh).
   - Aktueller Tibber-Strompreis (ct/kWh oder €/kWh) mit Trend.
   - Bisher aufgelaufene Tageskosten (€).
3. **Design-Standard:**
   - Nutze bevorzugt moderne native Home Assistant Lovelace Cards (`grid`, `tile`, `gauge`, `entities`) und optional populäre Cards wie `custom:apexcharts-card` oder `custom:power-flow-card-plus`, sofern in HACS vorhanden.

---


### Work Package 4: Industrial Smart Meter Emulation (Modbus TCP SDM630 & go-e Wallbox Communication)

Der ESPHome Power Monitor soll so erweitert werden, dass er wie ein industrieller Standard-Smart-Meter (z.B. Eastron SDM630) direkt mit der Wallbox (**go-eCharger Gemini Flex / V4**, SN 302240) und PV-Überschuss-Managern (EVCC/openWB) kommuniziert:

1. **Modbus TCP Server (Port 502) - Eastron SDM630 Register-Emulation:**
   - Bereitstellung eines leichtgewichtigen Modbus TCP Servers auf dem ESP32 (z.B. via Arduino ModbusIP oder C++ Custom Include).
   - Abbildung der ATM90E32 Messwerte auf standardisierte SDM630 Input/Holding Register (IEEE 754 Float32 Big-Endian):
     - `30001 - 30002`: Spannung L1 ($V_{L1}$)
     - `30003 - 30004`: Spannung L2 ($V_{L2}$)
     - `30005 - 30006`: Spannung L3 ($V_{L3}$)
     - `30007 - 30008`: Strom L1 ($I_{L1}$)
     - `30009 - 30010`: Strom L2 ($I_{L2}$)
     - `30011 - 30012`: Strom L3 ($I_{L3}$)
     - `30013 - 30014`: Wirkleistung L1 ($P_{L1}$)
     - `30015 - 30016`: Wirkleistung L2 ($P_{L2}$)
     - `30017 - 30018`: Wirkleistung L3 ($P_{L3}$)
     - `30053 - 30054`: Gesamt-Wirkleistung ($P_{tot}$)
     - `30071 - 30072`: Netzfrequenz ($f_{Grid} = 50.0 \text{ Hz}$)
     - `30343 - 30344`: Gesamtbezug Wirkenergie ($E_{imp} \text{ in kWh}$)
   - **Vorteil:** Die go-e Wallbox, EVCC oder openWB können den ESP32 direkt per IP und Port 502 als echten physischen "Eastron SDM630" Smart Meter ansteuern – völlig unabhängig von Home Assistant Ausfällen!

2. **Direkte go-eCharger API v2 Push-Option (Zero-Latency PV-Überschuss):**
   - Optionale HTTP-Request Routine in `power-monitor.yaml`:
     - Sende alle 2-5 Sekunden bei Leistungsänderung den aktuellen Netzübergabewert ($P_{grid}$) per POST/GET an die go-eCharger API (`http://<wallbox_ip>/api/set?ids={"pgrid": <grid_power_watts>}`).
     - Dadurch regelt die Wallbox den Ladestrom (6A bis 16A, 1-/3-phasig) in Echtzeit exakt nach PV-Überschuss.

---

## 3. Ziel-Dateien & Output-Pfade

1. **ESPHome Firmware Config:**
   - `/GitHub/ha_config/esphome/power-monitor.yaml` (und Spiegel `/GitHub/ha_config/homeassistant/esphome/power-monitor.yaml`).
2. **Home Assistant Package / Sensoren:**
   - `/GitHub/ha_config/homeassistant/packages/power_monitor_atm90e32.yaml`.
3. **Lovelace Dashboard / Card:**
   - `/GitHub/ha_config/homeassistant/dashboards/power_monitor_card.yaml` (oder äquivalente Dashboard-Einbindung).

---

## 4. Validierungskriterien für Jules

- [ ] `esphome compile /GitHub/ha_config/esphome/power-monitor.yaml` läuft fehlerfrei durch (oder valide ESPHome 2024+ Syntax ohne Warnungen).
- [ ] Keine doppelten Entity-Prefixes (`sensor.stromzahler_stromzahler_*` eliminiert).
- [ ] Alle 3 Phasen (L1 Blau, L2 Grau Referenz, L3 Schwarz) korrekt abgebildet und kalibriert.
- [ ] Tibber-Kostenberechnung ist dynamisch tarifkonform und referenziert `sensor.electricity_price_im_beisen_12b`.
- [ ] Lovelace Energy Card Package ist syntaktisch valide und sofort einsatzbereit.
- [ ] Saubere Dokumentation im Header der Dateien.
