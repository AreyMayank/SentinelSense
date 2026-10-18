# 🩺 SentinelSense

> **A self-contained patient-monitoring and safety prototype built on the Seeed Studio XIAO ESP32-C3.**  
> Everything — sensors, logic, dashboard, and API — runs directly on the microcontroller. No cloud. No external server. No database.

---

## 📌 What Is This?

**SentinelSense** is a wearable patient-safety prototype designed for **at-home or clinical care settings**. It reads biometric sensors, detects potential falls, supports manual SOS alerts, displays real-time readings on an OLED screen, plays audio reminders via a speaker, and serves a **live monitoring dashboard hosted directly by the ESP32**.

The core philosophy: the patient and caretaker should not depend on internet connectivity, cloud subscriptions, or external infrastructure for basic safety monitoring.

> ⚠️ **Disclaimer:** This is a **student prototype**, not a certified medical device or emergency-response service. Sensor readings are estimates. Fall detection may miss events or produce false alarms. Do not rely on this for clinical diagnosis or life-safety decisions.

---

## 🔧 Hardware

| Component | Purpose |
|---|---|
| **Seeed Studio XIAO ESP32-C3** | Main controller, web host, and Wi-Fi interface |
| **MAX30102** | Optical heart rate & SpO2 (blood oxygen) sensor |
| **MPU6050** | 6-axis IMU for motion sensing and fall detection |
| **1.3" I2C OLED (SH1106, 128×64)** | Local display for readings, alerts, and messages |
| **MAX98357A I2S Amplifier + 8Ω Speaker** | Voice/audio reminder playback *(later stage)* |
| **SOS Button** | Manually trigger an emergency event (D2 / GPIO4) |
| **OK/Cancel Button** | Confirm reminders or cancel a pending fall alert (D3 / GPIO5) |
| **Status LED** | Boot, connected, alert, and fault visual states (D10 / GPIO10) |
| **3.7V 700mAh LiPo + Slide Switch** | Portable power source *(later stage, after safety review)* |
| **Voltage Divider (2× 100kΩ)** | Safe battery voltage measurement via ADC (D1 / GPIO3) |

---

## 🗺️ Pin Map

| Function | XIAO Pin | ESP32-C3 GPIO | Notes |
|---|---|---|---|
| I2C SDA (shared bus) | D4 | GPIO6 | MAX30102, MPU6050, OLED |
| I2C SCL (shared bus) | D5 | GPIO7 | MAX30102, MPU6050, OLED |
| Battery ADC | D1 | GPIO3 | Through voltage divider only |
| SOS Button | D2 | GPIO4 | Button to GND, INPUT_PULLUP |
| OK/Cancel Button | D3 | GPIO5 | Button to GND, INPUT_PULLUP |
| Status LED | D10 | GPIO10 | 220Ω series resistor required |
| I2S BCLK | D0 | GPIO2 | Audio — later stage (strapping pin) |
| I2S LRC/WS | D7 | GPIO20 | Audio — later stage |
| I2S DIN | D8 | GPIO8 | Audio — later stage (strapping pin) |
| Reserved | D6 | GPIO21 | Unallocated |
| Avoid | D9 | GPIO9 | Boot-strapping concerns |

> All I2C devices share the same SDA/SCL lines at **3.3V logic**. OLED confirmed at address `0x3C`.

---

## ✅ Features

### 🫀 Vital Sign Monitoring
- Reads **MAX30102** optical sensor via shared I2C bus
- Estimates **Heart Rate (BPM)** and **SpO2 (%)** using the SparkFun MAX3010x library
- Invalid or unstable readings are **never fabricated** — shown as `Unavailable`
- Validity flags and timestamps are published to both the OLED and dashboard
- Raw IR/Red values available for diagnostics

### 🤸 Fall Detection
- Reads **MPU6050** 6-axis accelerometer and gyroscope data
- Uses a **configurable state machine** with four states:
  - `NORMAL` → `SUSPECTED_FALL` → `CANCELLED` or `ALERT_SENT`
- On suspected fall: triggers a **15-second cancellation window** with local warning
- Press **OK/Cancel** during the window to cancel and log a `CANCELLED` event
- If uncancelled: records an **`ALERT`** event on the local dashboard
- **Manual SOS** bypasses the fall timer and logs an immediate emergency event

### 🖥️ OLED Local Display
- Shows: Heart rate, SpO2, sensor status, Wi-Fi status, battery level, active alerts, and caretaker messages
- Updated from the same authoritative device state model used by the web dashboard
- Displays `Unavailable` for missing/invalid readings and stale indicators for outdated data

### 🌐 ESP32-Hosted Web Dashboard

The ESP32 serves its own HTML/CSS/JavaScript dashboard and REST API — **no external server needed**.

**Dashboard displays:**
- Live HR (BPM) and SpO2 (%), or `Unavailable`
- Sensor validity status and last-update timestamps
- Fall state and recent fall event log
- SOS state and recent SOS event log
- Wi-Fi / device status and IP address
- Battery voltage and estimated level *(when implemented)*
- Caretaker reminder schedules and text
- Caretaker messages queued for OLED / voice playback
- Clear stale/disconnected indicators

**REST API routes (local network only):**

| Method | Route | Description |
|---|---|---|
| `GET` | `/api/status` | Current readings, validity, device/network/battery state |
| `GET` | `/api/events` | Recent fall, SOS, and reminder events |
| `GET` | `/api/reminders` | Configured reminders list |
| `POST` | `/api/reminders` | Create or update a reminder |
| `POST` | `/api/messages` | Submit an OLED message, optionally queue audio |
| `POST` | `/api/fall/cancel` | Cancel a pending fall alert |
| `POST` | `/api/reminders/ack` | Acknowledge a reminder |
| `GET` | `/health` | Basic server/device heartbeat |

### 🔔 Reminders & Caretaker Messages
- Caretaker enters reminder text and schedules via the web dashboard
- Reminders displayed on OLED when active
- Acknowledgement state tracked per reminder
- Configuration persisted in flash where feasible (survives reboot)
- Audio playback via MAX98357A is a **later stage** feature

### 🔋 Battery Monitoring *(later stage)*
- 3.7V LiPo via a resistor voltage-divider read on GPIO3
- Firmware reconstructs and calibrates battery voltage from ADC readings
- Battery level shown on OLED and dashboard
- **LiPo will not be connected until power path, protection, and switch are independently reviewed**

### 🔊 Audio Output *(later stage)*
- MAX98357A I2S amplifier connected to GPIO2 (BCLK), GPIO20 (WS), GPIO8 (DIN)
- Speaker for voice reminder playback
- Strapping-pin implications for GPIO2 and GPIO8 must be verified before adoption

---

## 🏗️ Architecture

```
MAX30102 ─┐
          ├─ I2C → Sensor drivers → Validated measurements ─┐
MPU6050 ──┘                                                 │
OLED ←──────────────────────────────────────────────────────┤
Buttons → Input manager                                     │
Reminder scheduler ─────────────────────────────────────────┤
Fall state machine ─────────────────────────────────────────┤
                                                            ▼
                                                  Device State Model
                                                            │
                                                            ▼
                                              ESP32-hosted HTTP server
                                               ├─ HTML / CSS / JavaScript
                                               ├─ GET status/readings API
                                               ├─ GET events API
                                               ├─ POST reminder/message API
                                               └─ POST acknowledgement/action API
                                                            │
                                                            ▼
                                                Browser on local network
```

**Key design rule:** The OLED and the web API both read from **one single authoritative device state model** — they never calculate values independently.

---

## 🗂️ Firmware Modules

| Module | Responsibility |
|---|---|
| `main.ino` | Setup, loop, and coordination |
| `config.h` | Pins, thresholds, and intervals |
| `SensorManager` | I2C device initialization and sample acquisition |
| `Max30102Manager` | Raw readings, HR/SpO2 algorithm, validity, timestamps |
| `Mpu6050Manager` | Accelerometer and gyroscope readings |
| `FallDetector` | Fall candidate state machine and timeout |
| `ButtonManager` | Debounced SOS and OK/Cancel events |
| `DisplayManager` | OLED layout and updates |
| `ReminderManager` | Schedules and acknowledgements |
| `AudioManager` | I2S voice playback *(later stage)* |
| `DeviceState` | Authoritative current readings and event state |
| `WebServerManager` | Serve dashboard and REST API |
| `StorageManager` | Optional flash-based persistence |
| `web/index.html` / `web/style.css` / `web/app.js` | Dashboard frontend assets |

---

## 📡 Network Modes

| Mode | Description |
|---|---|
| **Station Mode** | ESP32 joins an existing Wi-Fi network; browser accesses it by local IP |
| **Access Point Mode** | ESP32 creates its own Wi-Fi network; browser connects directly |

- **AP Mode is the practical default** when Wi-Fi credentials are not known ahead of time
- Sensor sampling and fall logic **continue during Wi-Fi loss**
- Wi-Fi credentials are never logged or exposed in frontend source
- The dashboard is not reachable from the public internet by default

---

## 🔒 Safety & Privacy

- This is a **student prototype** — not a certified medical device or emergency-response service
- Sensor values (HR, SpO2) are estimates; fall detection may miss events or false-alarm
- **Never claim a caretaker was notified** unless a verified notification channel confirms it
- Patient data is kept **local by default** — nothing is sent to external services
- Add authentication if the device will be used on untrusted networks
- Do **not** expose the dashboard to the public internet by default
- Do **not** commit Wi-Fi credentials or patient data to source control

---

## 🚀 Implementation Stages

| Stage | Task | Status |
|---|---|---|
| 1 | Verify XIAO ESP32-C3 and serial output | ✅ Done |
| 2 | Verify OLED alone | ✅ Done — U8g2, SH1106, I2C `0x3C` |
| 3 | Verify MAX30102 raw data, then HR/SpO2 | 🔄 Raw readings obtained; HR/SpO2 under validation |
| 4 | Verify MPU6050 alone | ⏳ Pending |
| 5 | Combine all I2C devices, verify addresses | ⏳ Pending |
| 6 | Implement state model and button handling | ⏳ Pending |
| 7 | Implement fall state machine and cancellation window | ⏳ Pending |
| 8 | Host minimal webpage and status endpoint on ESP32 | ⏳ Pending |
| 9 | Connect live sensor state to the webpage | ⏳ Pending |
| 10 | Add local reminders/messages and persistence | ⏳ Pending |
| 11 | Add audio output (MAX98357A) | ⏳ Pending |
| 12 | Add battery monitoring and portable power | ⏳ Pending |
| 13 | Integration and failure-mode tests | ⏳ Pending |

---

## ✔️ Acceptance Criteria

- [ ] Firmware compiles for XIAO ESP32-C3 with all dependencies documented
- [ ] OLED and sensors can be tested independently
- [ ] I2C addresses verified and compatible (no conflicts)
- [ ] Invalid HR/SpO2 shown as `Unavailable` — never fabricated
- [ ] Fall cancellation and manual SOS pass controlled tests
- [ ] ESP32 serves the dashboard without any external backend
- [ ] Webpage shows actual live device data with stale/invalid state indicators
- [ ] Caretaker messages/reminders entered via webpage reach local firmware state
- [ ] Wi-Fi loss does not stop local sensor sampling or warning logic
- [ ] Battery operation declared complete only after safety and runtime tests pass

---

## 📚 Project Documentation

| File | Contents |
|---|---|
| [`PROJECT_SPEC.md`](PROJECT_SPEC.md) | Full requirements, features, and acceptance criteria |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Firmware structure, data flow, API design, and web behavior |
| [`HARDWARE_WIRING.md`](HARDWARE_WIRING.md) | Pin map, wiring tables, and electrical safety notes |
| [`AI_AGENT_GUIDE.md`](AI_AGENT_GUIDE.md) | Constraints and instructions for AI coding agents |

---

## 🛠️ Development Principles

1. **Test one component at a time** — follow the staged test order
2. **Keep modules small and separate** — sensor, fall logic, display, web, audio are distinct modules
3. **Compile after every stage** — never accumulate breaking changes
4. **All dashboard data comes from the ESP32** — no cloud shortcut
5. **Never fabricate sensor readings** — mark invalid and stale values explicitly
6. **HR and SpO2 are prototype estimates** — not medical-grade measurements
7. **Do not connect the LiPo** until power path, protection, and charging are reviewed
8. **A feature is done only when** it compiles, has a reproducible test, handles failures, and shows real patient state on the dashboard

---

## 🔩 Safe Wiring & Test Order

> ⚠️ **Always disconnect USB before changing wiring. Never modify wiring while the device is powered.**

1. XIAO alone over USB
2. OLED alone
3. MAX30102 alone
4. MPU6050 alone
5. All I2C devices together (scan for address conflicts)
6. Buttons and LED
7. ESP32-hosted webpage and Wi-Fi
8. Amplifier and speaker
9. Battery divider and LiPo — **only after full electrical review**

---

## 📦 Libraries & Toolchain

| Tool / Library | Purpose |
|---|---|
| **Arduino IDE** | Firmware development environment |
| **XIAO ESP32-C3 board target** | Board support package for Seeed XIAO |
| **SparkFun MAX3010x** | MAX30102 heart rate and SpO2 algorithm |
| **U8g2** | OLED display driver (SH1106 / SSD1306 compatible) |
| **Wire.h** | I2C communication (built-in) |
| **MPU6050 library** | Accelerometer and gyroscope data |
| **ESP32 WebServer / AsyncWebServer** | ESP32-hosted HTTP server and REST API |

---

*SentinelSense — built for care, designed for the edge.*
