# SentinelSense: Firmware and Web Architecture

## 1. Architecture rule

The **XIAO ESP32-C3 is both controller and web host**. It serves the
webpage and API. The browser displays data supplied by the same firmware
that reads the sensors. Do not require a cloud backend, Firebase,
external database or separate computer/server.

## 2. Data flow

``` text
MAX30102 ─┐
          ├─ I2C -> Sensor drivers -> Validated measurements ─┐
MPU6050 ──┘                                                  │
OLED <───────────────────────────────────────────────────────┤
Buttons -> Input manager                                    │
Reminder scheduler ------------------------------------------┤
Fall state machine ------------------------------------------┤
                                                            v
                                                  Device state model
                                                            |
                                                            v
                                               ESP32-hosted HTTP server
                                                ├─ HTML/CSS/JavaScript
                                                ├─ GET status/readings API
                                                ├─ GET events API
                                                ├─ POST reminder/message API
                                                └─ POST acknowledgement/action API
                                                            |
                                                            v
                                                  Browser on local network
```

This is the intended design. Implement and test it incrementally.

## 3. Suggested firmware modules

Keep modules small; names are suggestions: - `main.ino`: setup, loop and
coordination - `config.h`: pins, thresholds and intervals -
`SensorManager`: initialize I2C devices and acquire samples -
`Max30102Manager`: raw readings, HR/SpO2, validity and timestamps -
`Mpu6050Manager`: accelerometer and gyroscope readings - `FallDetector`:
fall candidate state machine and timeout - `ButtonManager`: debounced
SOS and OK/Cancel events - `DisplayManager`: OLED layout and updates -
`ReminderManager`: schedules and acknowledgements - `AudioManager`:
later-stage I2S playback - `DeviceState`: authoritative current readings
and event state - `WebServerManager`: serve webpage and API -
`StorageManager`: optional local persistence - `web/index.html`,
`web/style.css`, `web/app.js`: dashboard assets embedded or served from
ESP32 flash

Do not create every module at once. Start with the smallest compiling
stage.

## 4. Single device state

OLED and API should read from one authoritative state rather than
independently inventing/recalculating values. Suggested fields: - HR
value, validity and update time - SpO2 value, validity and update time -
raw IR/red values for diagnostics - MAX30102 status - MPU6050 status and
motion values - fall state and recent events - SOS state and recent
events - Wi-Fi status and IP address - battery value/validity when
implemented - current message - reminders and acknowledgement state -
last update time

Use explicit validity flags/null fields. Zero must not ambiguously mean
both a valid reading and a missing reading.

## 5. Suggested HTTP API

These are suggested local routes; document the chosen JSON schema: -
`GET /api/status`: current readings, validity, device/network/battery
state - `GET /api/events`: recent fall, SOS and reminder events -
`GET /api/reminders`: configured reminders - `POST /api/reminders`:
create/update reminder - `POST /api/messages`: submit OLED message and
optionally queue audio - `POST /api/fall/cancel`: cancel pending fall
where allowed - `POST /api/reminders/ack`: acknowledge a reminder -
`GET /health`: basic server/device status

Validate request bodies, cap message length, reminder count and event
history, and use meaningful HTTP status codes.

## 6. Frontend behavior

-   Fetch actual state from the ESP32 API.
-   Choose one update mechanism: polling, WebSocket or Server-Sent
    Events.
-   Display `Unavailable` for invalid readings and stale indicators for
    outdated readings.
-   Show last-update time, device status, and recent events.
-   Show fall/SOS alerts prominently.
-   Send changes to the ESP32 and report success only after the server
    responds.
-   Make the UI usable on mobile and desktop.
-   Do not hard-code production patient values.

## 7. Networking

Document whether station mode, AP mode, or both are supported. In
station mode, the browser normally must be on the same LAN and use the
ESP32's IP. In AP mode, connect directly to the ESP32 network. Sensor
acquisition and local warning logic must continue during Wi-Fi loss.
Avoid long blocking delays. A local web server is not automatically
reachable from the public internet.

## 8. Persistence and privacy

Current sensor values can remain in RAM. Reminder configuration and a
small event log may be stored in flash if implemented. Avoid writing
every high-frequency sensor sample to flash. Do not send patient data
externally by default. For networks with untrusted users, add
authentication and consider the limitations of plain HTTP. Never commit
credentials or patient information to source control.
