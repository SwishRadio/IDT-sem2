# IoT Health Monitoring System

An ESP32-based wearable health monitor that tracks heart rate, body temperature, and fall events in real time. All data is served through a self-hosted web dashboard accessible from any device on the same WiFi network. If an alert is raised and the patient does not respond within 3 minutes, the system automatically escalates to emergency services.

---

## Features

- Real-time heart rate monitoring with a two-stage noise filter
- Body temperature monitoring with outlier smoothing
- Three-phase fall detection: free-fall → impact → inactivity
- Web dashboard with live scrolling BPM and temperature charts
- Emergency alert banner with a 3-minute countdown before automatic escalation
- WiFi auto-reconnect with a non-blocking watchdog
- Serial debug output every 4 seconds

---

## Hardware

| Component | Purpose | Connection |
|---|---|---|
| ESP32 (any 30/38-pin variant) | Microcontroller | — |
| HW-827 heartbeat sensor | Heart rate | D0 → GPIO34 |
| DS18B20 temperature sensor | Body temperature | Data → GPIO4 |
| MPU6050 accelerometer/gyro | Fall detection | SDA → GPIO21, SCL → GPIO22 |
| 10 kΩ resistor | Pull-up on GPIO34 | GPIO34 ↔ 3.3V |
| 4.7 kΩ resistor | Pull-up for DS18B20 | GPIO4 ↔ 3.3V |

### Wiring notes

**HW-827 heartbeat sensor**

Wire the module's `D0` pin (digital output) to GPIO34 — not `A0`. The `A0` pin outputs a raw analog waveform that will produce hundreds of false interrupts per beat. GPIO34 on the ESP32 does not have a functional internal pull-up, so an external 10 kΩ resistor between GPIO34 and 3.3V is required. Without it the pin floats and reads noise as heartbeats.

```
3.3V ──┬── HW-827 VCC
       │
      [10kΩ]
       │
GPIO34 ┴── HW-827 D0

GND ──── HW-827 GND
```

After wiring, adjust the blue trimmer potentiometer on the HW-827 until the onboard LED blinks once per heartbeat. If the LED is always on or always off, the threshold is set wrong.

**DS18B20 temperature sensor**

```
3.3V ──┬── DS18B20 VDD
       │
      [4.7kΩ]
       │
GPIO4 ─┴── DS18B20 DATA

GND ──── DS18B20 GND
```

**MPU6050**

```
GPIO21 (SDA) ── MPU6050 SDA
GPIO22 (SCL) ── MPU6050 SCL
3.3V ────────── MPU6050 VCC
GND ─────────── MPU6050 GND
```

---

## Software setup

### Arduino IDE libraries

Install all of the following through **Sketch → Include Library → Manage Libraries**:

| Library | Install name |
|---|---|
| ESP32 board support | `esp32` by Espressif Systems (via Board Manager) |
| DallasTemperature | `DallasTemperature` by Miles Burton |
| OneWire | `OneWire` by Jim Studt et al. |
| MPU6050_tockn | `MPU6050_tockn` by tockn |
| ArduinoJson | `ArduinoJson` by Benoit Blanchon |

### Board settings

Go to **Tools** and set:

| Setting | Value |
|---|---|
| Board | ESP32 Dev Module |
| Upload Speed | 921600 |
| CPU Frequency | 240 MHz |
| Flash Size | 4MB |
| Partition Scheme | Default 4MB with spiffs |
| Port | whichever COM/tty port your ESP32 appears on |

### Configuration

Open `health_monitor_v3.ino` and update the two lines at the top of the file:

```cpp
const char* ssid     = "your_network_name";
const char* password = "your_password";
```

Upload the sketch. Open the Serial Monitor at **115200 baud**. After connecting to WiFi the monitor prints the dashboard URL:

```
[INFO] Dashboard: http://192.168.x.x
```

Open that address in any browser on the same network.

---

## Dashboard

The web interface is served directly from the ESP32 on port 80. It auto-refreshes every 1.2 seconds and requires no internet connection.

### Panels

**Heart rate** — current BPM with status badge (Normal / Low / High / No Signal) and a 60-second scrolling chart with danger threshold lines.

**Temperature** — current °C with a circular gauge and a 60-second scrolling chart.

**Fall detection** — current fall state with live X/Y/Z accelerometer bars.

**Sensor status** — online/offline indicator for each sensor and WiFi.

**Emergency banner** — appears when any alert threshold is breached. Shows a 3-minute countdown ring. The patient must tap one of two buttons:

- **I need help** — escalates immediately and calls emergency services
- **I am okay** — clears the alert and resets the countdown

If neither button is pressed within 3 minutes, `checkEscalation()` fires automatically.

### API endpoints

| Endpoint | Method | Description |
|---|---|---|
| `/` | GET | Dashboard HTML |
| `/api/data` | GET | Current sensor readings + escalation state as JSON |
| `/api/history` | GET | Last 60 seconds of BPM and temperature as JSON arrays |
| `/api/need_assistance` | POST | Immediately escalates the emergency |
| `/api/clear_emergency` | POST | Clears the active alert and resets the countdown |

---

## Thresholds

All thresholds are defined as named constants near the top of the sketch and can be changed without touching any other code.

| Constant | Default | Meaning |
|---|---|---|
| `HEART_RATE_MIN` | 45 BPM | Below this triggers a bradycardia alert |
| `HEART_RATE_MAX` | 130 BPM | Above this triggers a tachycardia alert |
| `TEMP_MIN` | 35.0 °C | Below this triggers a hypothermia alert |
| `TEMP_MAX` | 39.5 °C | Above this triggers a fever alert |
| `FREEFALL_THRESHOLD` | 3.0 m/s² | Acceleration below this signals free-fall |
| `IMPACT_THRESHOLD` | 28.0 m/s² | Acceleration above this signals impact |
| `INACTIVITY_THRESHOLD` | 10.5 m/s² | Below this after impact confirms a fall |
| `EMERGENCY_COOLDOWN` | 25 000 ms | Minimum gap between successive alerts |
| `ESCALATION_TIMEOUT_MS` | 180 000 ms | Time before auto-escalation (3 minutes) |

---

## Heart rate signal processing

The HW-827 is a resistive/optical sensor that produces noisy electrical signals. Raw interrupt counting produces readings of 200+ BPM even at rest because the sensor fires multiple pulses per actual heartbeat. Three layers of filtering are applied:

**ISR debounce** — the interrupt handler rejects any pulse arriving less than 273 ms after the previous one. This corresponds to 220 BPM and blocks most bounce pulses at the hardware level before they enter the software pipeline.

**Median filter (stage 1)** — the last 5 instantaneous BPM values are sorted and the middle value is returned. A median collapses isolated spikes without smearing the true reading the way a mean would.

**BPM jump guard** — if the median output differs from the last accepted value by more than 35 BPM, the reading is discarded as a glitch and the previous stable value is held. A genuine heart rate cannot change by 35 BPM between two consecutive beats.

**Moving average (stage 2)** — the last 8 post-median values are averaged to produce the final displayed BPM, smoothing beat-to-beat jitter.

If readings are still erratic after flashing the firmware, work through these hardware checks in order:

1. Confirm you are wired to `D0`, not `A0`
2. Confirm the 10 kΩ pull-up resistor is present between GPIO34 and 3.3V
3. Adjust the trimmer until the module LED blinks once per heartbeat
4. If still unstable, increase the ISR debounce floor from `273000UL` to `400000UL` (rejects anything faster than ~150 BPM per pulse)
5. Add a 100 nF ceramic capacitor across the sensor's VCC and GND pins to filter power supply noise from WiFi transmit bursts

---

## Emergency escalation

When a threshold is breached the system enters an alert state and starts a 3-minute countdown. If the patient does not tap a response button on the dashboard within that window, `checkEscalation()` calls `callEmergencyServices()`.

The default implementation prints to Serial. To connect it to a real escalation mechanism, find the clearly marked extension point in `callEmergencyServices()` and add your code there — an HTTP POST to a webhook, an AT command to a GSM module (SIM800L etc.), or an MQTT publish:

```cpp
// Example: HTTP POST
if (WiFi.status() == WL_CONNECTED) {
  HTTPClient http;
  http.begin("http://your-server.com/emergency");
  http.addHeader("Content-Type", "application/json");
  http.POST("{\"escalated\":true}");
  http.end();
}
```

---

## Project structure

```
health_monitor_v3.ino   — full sketch (single file)
README.md               — this file
```

---

## Disclaimer

This project is a learning exercise and is not a certified medical device. Do not rely on it as a substitute for professional medical equipment or emergency services.
