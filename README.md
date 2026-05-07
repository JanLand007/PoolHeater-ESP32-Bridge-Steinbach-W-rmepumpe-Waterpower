# PoolHeater-ESP32-Bridge-Steinbach-W-rmepumpe-Waterpower 🏊‍♂️
Reverse Engineering und Home Assistant Integration für Pool-Wärmepumpen mit serieller Kommunikation via ESP32/ESPHome


Dieses Projekt ermöglicht die intelligente Steuerung und Überwachung einer Standard-Pool-Wärmepumpe von Steinbach und ähnlichen Geräten über einen ESP32 und ESPHome. Es ist die perfekte Lösung, wenn das originale Display defekt ist oder die Pumpe einfach smart gemacht werden soll.

---

## 🌟 Features
- **Echtzeit-Monitoring:** Wassertemperatur, Außentemperatur und interne Kompressortemperatur direkt in Home Assistant.
- **Vollständige Kontrolle:** Ein- und Ausschalten sowie (bald) Einstellen der Zieltemperatur.
- **Native Home Assistant Integration:** Erscheint als Thermostat/Klima-Entität im Dashboard.
- **Robustes Design:** Inklusive Schutzbeschaltung (Diode) und Pufferkondensator für maximale WLAN-Stabilität.

---

## 🛠 Hardware Setup

### Benötigte Komponenten
- **ESP32** (z. B. NodeMCU oder Wroom)
- **Bi-direktionaler Level Shifter** (3.3V <-> 12V/5V)
- **Step-Down Konverter** (12V -> 5V)
- **Diode** (z. B. 1N5817) für den TX-Pfad
- **Elektrolytkondensator (1000µF)** zur Stabilisierung der 5V Schiene
- **Kabel & Gehäuse**

### Schaltplan & Verkabelung
Die Kommunikation erfolgt über ein 3-adriges Kabel (VCC, GND, VPP/DATA). Da das System Half-Duplex arbeitet (Senden und Empfangen auf einer Leitung), nutzen wir eine Schutzdiode am TX-Pin.

#### Anschlussplan
1. **Stromversorgung:**
   - **12V (Braun)** von der Pumpe -> Step-Down **IN+**.
   - **GND (Gelb)** von der Pumpe -> Common Ground (Step-Down IN-, ESP GND, Level Shifter GND).
   - **Step-Down Out (5V)** -> ESP32 **VIN**, Level Shifter **HV**, 1000µF Kondensator (+).
2. **Datenleitung:**
   - **Data (Blau)** von der Pumpe -> Level Shifter **HV1**.
   - **ESP Pin 18 (RX)** -> Level Shifter **LV1**.
   - **ESP Pin 19 (TX)** -> Anode (+) der Diode; Kathode (-) der Diode -> Level Shifter **LV1**.

> [!TIP]
> Der 1000µF Kondensator an der 5V Leitung verhindert Reboots des ESP32 bei WLAN-Lastspitzen.

---

## 🔍 Das Protokoll
Die Pumpe kommuniziert mit 23-Byte-Paketen (Pulsweiten-moduliert). Ein Paket beginnt immer mit `0xDD`.

| Byte (Index) | Funktion | Beschreibung |
| :--- | :--- | :--- |
| 1 (0) | Header | Immer `0xDD` |
| 4 (3) | Ist-Temp | Wassertemperatur in °C |
| 5 (4) | Soll-Temp | Zieltemperatur in °C |
| 6 (5) | Power | `0x01` = AN, `0x00` = AUS |
| 17 (16) | Umwelt | Außentemperatur |
| 18 (17) | Intern | Interne Gerätetemperatur (Kompressor) |
| 23 (22) | Checksumme | Summe der Bytes 1-21 (Modulo 256) |

---

## 💻 Software (ESPHome)
Die Integration erfolgt über ein Custom-Lambda in ESPHome, welches die Pulsweiten (1ms Mark/Space) dekodiert und die Checksumme für ausgehende Befehle live berechnet.

### Beispiel: Power-Schalter Template
```yaml
switch:
  - platform: template
    name: "Poolheizung Power"
    id: pool_power_switch
    icon: "mdi:pool-thermometer"
    lambda: |-
      return id(pool_power_status).state;
    turn_on_action:
      - button.press: pumpe_ein_test
    turn_off_action:
      - button.press: pumpe_aus_test
