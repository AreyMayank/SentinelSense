# CareVoice Edge: Technical Project Specification

## 1. Product definition

CareVoice Edge is an ESP32-C3 patient-monitoring and safety prototype.
It collects sensor data, evaluates motion for possible falls, provides
local feedback, and makes patient data available through a webpage
hosted by the ESP32 itself.

## 2. Mandatory webpage requirement

**Every patient-data view must be available on the webpage hosted by the
ESP32-C3.** The ESP32 serves the frontend and the local API. No external
cloud service or separate server is required.

The webpage must show: - Heart rate in BPM, or `Unavailable` when
invalid - SpO2 percentage, or `Unavailable` when invalid - Sensor
validity/status and last-update time - Fall state and recent fall
events - SOS state and recent SOS events - Wi-Fi/device status - Battery
voltage/estimated level when implemented - Caretaker reminder text and
schedules - Caretaker messages for OLED/voice playback - Clear
stale/disconnected indicators

Do not show hard-coded demo values in production firmware. If demo mode
is used during UI development, label it clearly and disable it in the
real device build.

## 3. Features

### Vital-sign estimates

Read MAX30102 red/infrared data and estimate HR and SpO2. Reject invalid
results; never substitute plausible-looking numbers. Publish validity
and freshness. These are prototype estimates, not medical-grade
measurements.

### Suspected-fall detection

Read MPU6050 accelerometer and gyroscope data. Use configurable logic
considering impact, orientation/motion change and post-event inactivity.
Treat it as a suspected fall, not a guaranteed diagnosis. Trigger a
local warning and a configurable 15-second cancellation window. If
cancelled, record a cancelled event; otherwise record an alert event on
the ESP32-hosted webpage. Manual SOS bypasses the fall timer.

### Patient interface

OLED shows readings, sensor status, Wi-Fi, battery, alerts and caretaker
messages as space permits. OK/Cancel confirms reminders or cancels a
pending fall alert. SOS triggers a manual emergency event. Define status
LED patterns for boot, connected, alert and fault states.

### Reminders and messages

Caretaker configures text and schedules through the ESP32-hosted
webpage. Store locally where feasible and persist across reboot if
implemented. Display messages on OLED. MAX98357A voice playback is a
later stage. Provide acknowledgement status where practical.

### Local dashboard

Serve HTML/CSS/JavaScript from ESP32 and expose a local API for
readings, status, events, reminders and messages. Use one update
mechanism initially: polling, WebSocket or Server-Sent Events. Data must
come from live firmware state. Do not send patient data to cloud
services by default.

## 4. Network modes

Choose and document one initial mode: - **Station mode:** ESP32 joins an
existing Wi-Fi network; browser opens its local IP on the same
network. - **Access-point mode:** ESP32 creates its own Wi-Fi network;
browser connects directly and opens the configured local address.

AP mode is a practical initial choice when credentials are not known. Do
not expose credentials in logs. Remote internet access, port forwarding
and cloud access are out of scope unless approved later.

## 5. Implementation stages

1.  Verify board and serial output.
2.  Verify OLED alone.
3.  Verify MAX30102 raw data, then HR/SpO2.
4.  Verify MPU6050 alone.
5.  Combine I2C devices and verify addresses.
6.  Implement state model and button handling.
7.  Implement fall state machine and cancellation window.
8.  Host a minimal webpage and status endpoint on ESP32.
9.  Connect actual sensor state to the webpage.
10. Add local reminders/messages and persistence.
11. Add audio output.
12. Add battery monitoring and safe portable power.
13. Run integration and failure-mode tests.

## 6. Acceptance criteria

-   Firmware compiles for XIAO ESP32-C3 with dependencies documented.
-   OLED and sensors can be tested independently.
-   I2C addresses are verified and compatible.
-   Invalid HR/SpO2 are shown as unavailable.
-   Fall cancellation and manual SOS pass controlled tests.
-   ESP32 serves the dashboard without an external backend.
-   Webpage shows actual device data and stale/invalid states.
-   Webpage-entered messages/reminders reach local firmware state.
-   Wi-Fi loss does not stop local sensor sampling or warning logic.
-   Battery operation is not declared complete until safety and runtime
    tests pass.

## 7. Safety and privacy

This is a student prototype, not a certified medical device or
emergency-response service. Sensor values can be inaccurate, and fall
detection can miss events or generate false alarms. Do not claim an
alert reached a caretaker unless the relevant channel confirms it. Keep
data local by default; add authentication if untrusted users may access
the webpage. Do not expose the ESP32 dashboard to the public internet by
default.
