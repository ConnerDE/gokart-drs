# Gokart Elektronik (ESP32-S3)

Elektronik-Firmware für den Unterbau-Controller eines selbstgebauten Gokarts. Der ESP32-S3 steuert Antrieb, Getriebe, Sicherheit, Beleuchtung, Telemetrie und ein DRS-System (Drag Reduction System) für einen verstellbaren Heckflügel.

Schulprojekt zu zweit: Hardware (Aufbau, Verkabelung, Mechanik) von meinem Projektpartner (Louis), Software (dieser Code) von mir (Conner).

**Meine Rolle (Conner) :** Fehleranalyse & -behebung des bestehenden Codes sowie Neu-Implementierung des DRS-Systems und diverser weiterer Features (Gaspedal-ADC, Stromsensor, CAN-Lenkwinkelabgleich, u.a.).

## Hardware

- **MCU:** ESP32-S3 DevKit C1 (N16R8)
- I/O-Expansion: MCP23X17 (2x)
- ADC: ADS1115 (Gaspedal-Poti, Lenkwinkel-Backup, Stromsensor ACS770 30A)
- Getriebeaktor: TMC2209 + NEMA17 Schrittmotor
- Temperatur: DS18B20 (OneWire), MAX31855 (EGT), AHT/BMP280
- Beleuchtung: WS2812B (FastLED) – Front, Heck, Spoiler
- Display: SSD1306 OLED
- Kommunikation: CAN-Bus (TWAI, Lenkung-ESP als zweiter Knoten), BLE (Telemetrie & Kalibration)
- Aktuatoren: Servos (Gas, Auspuff), hydraulischer DRS-Aktuator

Vollständige Pinbelegung: [`a_Unterbau/HARDWARE_PINOUT_COMPLETE.txt`](a_Unterbau/HARDWARE_PINOUT_COMPLETE.txt)

## Projektstruktur

```
a_Unterbau/
├── a_Unterbau.ino          # Globale Includes, Objekte
├── z_SETUP.ino              # Setup-Routine
├── z2_LOOP.ino              # Hauptloop
├── zb_global.ino            # Globale Instanzen
├── b_PIN_Def.ino            # Pindefinitionen
├── c_LED_Config.ino         # WS2812B Konfiguration
├── c2_ADS1115_Driver.ino    # ADC-Treiber (Gas, Lenkwinkel-Backup, Strom)
├── c3_CURRENT_SENSOR.ino    # Stromsensor ACS770
├── c4_EGT_MAX31855.ino      # Abgastemperatur
├── c5_STEERING.ino          # Lenkwinkel (lokal + CAN-Abgleich)
├── d_TMC2209_Config.ino     # Schrittmotortreiber Getriebe
├── f_MCP23017_Mapping.ino   # I/O-Expander Pin-Mapping
├── g_RPM_Counter.ino        # Drehzahlmessung (Hall-Sensor)
├── h_CAN_Receiver.ino       # CAN-Bus Empfang (u.a. Lenkwinkel)
├── i_GAS_SERVO.ino          # Gaspedal → Servo
├── j_SAFETY_STARTSTOP.ino   # Sicherheitslogik, Start/Stop
├── k_GEARBOX.ino            # Getriebesteuerung
├── l_EXHAUST-SERVO.ino      # Auspuffklappen-Servo
├── m_LIGHTS.ino             # Beleuchtungslogik
├── n_BLE.ino                # BLE-Telemetrie & Kommandos
├── o_DISPLAY.ino            # OLED-Anzeige
├── p_DRS.ino                # DRS-Zustandsmaschine
└── q_DRS_Config.ino         # DRS-Parameter & Bedingungslogik
```

Zusätzliche Doku: [`FEATURE_IMPLEMENTATION.txt`](a_Unterbau/FEATURE_IMPLEMENTATION.txt), [`IMPLEMENTATION_CHECKLIST.txt`](a_Unterbau/IMPLEMENTATION_CHECKLIST.txt), [`VALIDATION_CHECKLIST.txt`](a_Unterbau/VALIDATION_CHECKLIST.txt)

## Features

### DRS (Drag Reduction System)
- Zustandsmaschine: `Disabled → Armed → Active`
- Manuelle Aktivierung per Lenkradtaster (nur wenn armed)
- Sicherheitsbedingungen: Speed, RPM, Gas, Öltemperatur, Spannung, Bremse
- Show-Mode per BLE (bypasst alle Bedingungen für Standvorführungen)

### Gaspedal
- Umstellung von EC11-Encoder auf 10k-Poti über ADS1115 (Kanal 0)
- Lineare Kalibrierung (Preferences-Speicher), Multi-Sample-Auslesung gegen Rauschen

### Stromsensor
- ACS770 (30A) über ADS1115 (Kanal 3)
- Strom-, Leistungsberechnung, BLE-Telemetrie

### Lenkwinkel-Abgleich
- Duale Messung: lokaler ADC (GPIO 3) + CAN-Bus vom Lenkungs-ESP
- Gewichteter Durchschnitt (70 % lokal / 30 % CAN), Fallback bei fehlendem CAN-Signal
- Plausibilitätswarnung bei Abweichung > 20 %

### Weitere Systeme
- Getriebesteuerung mit Schrittmotor & Neutral-Sensor
- Sicherheits-Interlocks (Bremse, Öldruck, Zündunterbrechung)
- WS2812B-Beleuchtung (Front/Heck/Spoiler) inkl. Blinker, Fernlicht, Bremslicht
- OLED-Statusanzeige
- BLE-Telemetrie & Fernkalibration

## BLE Commands

### Kalibration
| Command | Funktion |
|---|---|
| `CAL:GAS_MIN` | Gaspedalposition als Minimum speichern |
| `CAL:GAS_MAX` | Gaspedalposition als Maximum speichern |
| `SET:SRV_GAS_MIN:<wert>` | Servo Gas Minimalwinkel setzen |
| `SET:SRV_GAS_MAX:<wert>` | Servo Gas Maximalwinkel setzen |
| `SET:SRV_EXH_MIN:<wert>` | Servo Auspuff Minimalwinkel setzen |
| `SET:SRV_EXH_MAX:<wert>` | Servo Auspuff Maximalwinkel setzen |

### Reset
| Command | Funktion |
|---|---|
| `RESET_HOURS` | Betriebsstunden auf 0 zurücksetzen |
| `RESET_CHAIN` | Kettenschaltzähler auf 0 zurücksetzen |

### Sonstiges
| Command | Funktion |
|---|---|
| `SPD:<wert>` | Geschwindigkeit manuell setzen (float) |
| `DRS:SHOW_ON` | DRS Show-Mode aktivieren (bypasst alle Bedingungen) |
| `DRS:SHOW_OFF` | DRS Show-Mode deaktivieren |

## Abhängigkeiten (Arduino Libraries)

- `Adafruit_MCP23X17`, `Adafruit_AHTX0`, `Adafruit_BMP280`, `Adafruit_ADS1X15`
- `OneWire`, `DallasTemperature`
- `FastLED`
- `Adafruit_GFX`, `Adafruit_SSD1306`
- `BLEDevice` / `BLEServer` / `BLEUtils` / `BLE2902`
- `ArduinoJson`
- `TMCStepper`
- ESP-IDF: `driver/pulse_cnt.h`, `driver/ledc.h`, `driver/twai.h`, `esp_task_wdt.h`, `Preferences`

## Bekannte offene Punkte

- Kanal 1 des ADS1115 ist noch frei für Erweiterungen
- CAN-Message für Lenkwinkel könnte als eigene, dedizierte Message definiert werden
- Gaspedal: Kalman-Filter / Hysterese als mögliche Verbesserung

## Lizenz

Proprietär – alle Rechte vorbehalten.
