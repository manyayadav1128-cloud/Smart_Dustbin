# Smart Dustbin 🗑️📶

An ESP8266-based automatic touchless dustbin that opens its lid using an IR sensor, monitors fill level with an ultrasonic sensor, displays status on an LCD, and sends live data plus full-bin alerts to the **Blynk** IoT app.

---

## Features

- **Touchless lid operation** — an IR proximity sensor detects a hand/object and automatically opens the lid via a servo motor, then closes it after a short delay
- **Fill-level monitoring** — an ultrasonic sensor measures how full the bin is and calculates a fill percentage
- **On-device display** — a 16x2 I2C LCD shows real-time lid status ("Opening...", "Closing...") and fill percentage
- **Remote monitoring via Blynk**
  - Live fill percentage sent to a virtual pin (V0) for dashboard gauges/charts
  - Push notification/event triggered automatically when the bin reaches 80% full
- **Debounced full-bin alert** — the "bin full" event is only sent once per full cycle, not repeatedly every loop

---

## Hardware Required

| Component | Notes |
|---|---|
| ESP8266 (NodeMCU / Wemos D1 Mini) | Main controller with built-in WiFi |
| IR Proximity Sensor | Detects hand/object to trigger lid opening — connected to D4 |
| Ultrasonic Sensor (HC-SR04) | Measures dustbin fill level — Trig on D6, Echo on D7 |
| SG90 (or similar) Servo Motor | Opens/closes the lid — connected to D5 |
| 16x2 I2C LCD Display | Shows status and fill percentage (I2C address 0x27) |
| Dustbin enclosure | Sized so sensor readings match `dustbinDepth` |
| Power supply | 5V supply recommended, especially for the servo |
| Jumper wires, breadboard, etc. | Wiring |

---

## Pin Configuration

| Function | ESP8266 Pin |
|---|---|
| IR Sensor (lid trigger) | D4 |
| Ultrasonic Trig | D6 |
| Ultrasonic Echo | D7 |
| Servo (lid) | D5 |
| LCD (I2C) | SDA/SCL (default I2C pins, e.g. D2/D1 on NodeMCU) |

> **Note:** The LCD uses the I2C bus, so its SDA/SCL pins depend on your board's default I2C mapping (commonly D2/D1 on NodeMCU) — connect according to your specific board's I2C pinout, not the digital pin table above.

---

## Blynk Setup

This project uses the **Blynk IoT platform** for remote monitoring and alerts.

### Before uploading, update these values in the code:

```cpp
#define BLYNK_TEMPLATE_ID   "templete name"   // Your Blynk Template ID
#define BLYNK_TEMPLATE_NAME "templete name"   // Your Blynk Template Name
#define BLYNK_AUTH_TOKEN    "authentication token"  // Your device Auth Token

char ssid[] = "ssid";   // Your WiFi network name
char pass[] = "pass";   // Your WiFi password
```

1. Create a free account and a new **Template** at [blynk.cloud](https://blynk.cloud).
2. Copy the **Template ID**, **Template Name**, and device **Auth Token** into the sketch.
3. Add a **Datastream** on **Virtual Pin V0** (type: Integer, range 0–100) to receive the fill percentage.
4. Add a **Widget** (e.g., Gauge or Level display) on your Blynk dashboard bound to V0.
5. Set up an **Automation/Event** named `dustbin_full` in the Blynk console so the `Blynk.logEvent()` call triggers a push notification or email when the bin is full.

---

## How It Works

### Lid Control (IR Sensor)

- When the IR sensor detects an object (`HIGH`), the LCD shows "Opening...", the servo rotates to 180°, holds for 3 seconds, then returns to 0°, and the LCD shows "Closing...".
- When no object is detected, the servo stays at 0° (closed) and the LCD continues showing "Closing...".

### Fill Level Monitoring (Ultrasonic Sensor)

- The ultrasonic sensor measures the distance from the sensor to the trash surface.
- `level = dustbinDepth - distance` converts that into how much of the bin's depth is filled, clamped between 0 and `dustbinDepth` (default: **25 cm**).
- This is converted into a percentage: `percentage = (level * 100) / dustbinDepth`.
- The percentage is shown on the LCD, printed to Serial, and pushed to Blynk virtual pin V0.
- If the ultrasonic sensor gets no echo within 30ms (e.g., open air or sensor fault), the reading is skipped for that cycle rather than reporting a false value.

### Full-Bin Alert

- Once fill percentage reaches **80% or higher**, the LCD displays "Bin Full!!!" and a Blynk event (`dustbin_full`) is sent **once** via the `eventSent` flag, preventing repeated notifications every loop cycle.
- The flag resets once the level drops back below 80%, allowing a new alert if it fills up again later.

---

## Setup Instructions

1. **Install ESP8266 board support** in the Arduino IDE (via Boards Manager, if not already installed).
2. **Install the required libraries**:
   - `Blynk` (Blynk IoT library)
   - `Servo`
   - `LiquidCrystal_I2C`
   - `Wire` (bundled with Arduino IDE)
3. **Wire the hardware** according to the pin configuration table above.
4. **Update WiFi and Blynk credentials** in the code (see Blynk Setup above).
5. **Set `dustbinDepth`** to match the actual internal depth of your dustbin in centimeters.
6. **Upload the sketch** to your ESP8266 board.
7. **Open the Serial Monitor** at 9600 baud to confirm WiFi/Blynk connection and view live readings.
8. **Check your Blynk dashboard** to see the live fill-level gauge and receive full-bin notifications.

---

## Calibration Notes

- **`dustbinDepth`**: Must match the real internal depth (in cm) from where the ultrasonic sensor is mounted down to the bottom of the bin, or fill percentages will be inaccurate.
- **Ultrasonic sensor placement**: Should be mounted at the top of the bin, facing straight down, with a clear line of sight (no obstructions) to get reliable readings.
- **IR sensor sensitivity/range**: Adjustable on most modules via an onboard potentiometer — tune so it reliably detects a hand/object at your desired opening distance.

---

## Possible Improvements

- Use `Blynk.virtualWrite()` for lid status too, so the app dashboard reflects "Opening/Closing" in addition to fill level.
- Replace blocking `delay(3000)` for the lid with a non-blocking `millis()` timer so ultrasonic readings and Blynk stay responsive while the lid is open.
- Add a battery-level or low-power indicator if running off battery power.
- Add a buzzer for a local audible full-bin alert in addition to the Blynk notification.
- Debounce the IR sensor read to avoid the lid re-triggering if a hand lingers near the sensor.

---

## License

Free to use and modify for personal, educational, or hobby projects.
