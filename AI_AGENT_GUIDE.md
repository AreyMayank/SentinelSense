# CareVoice Edge: Instructions for AI Coding Agents

## 1. Role

You are contributing firmware, embedded web UI and documentation for
CareVoice Edge, a student prototype based on the Seeed Studio XIAO
ESP32-C3. Treat this file as project constraints.

## 2. Absolute architecture requirement

**All patient data must appear on a webpage hosted directly by the
ESP32-C3.**

The ESP32 serves the HTML/CSS/JavaScript dashboard and a local API. Do
not make a cloud server, Firebase, external database, laptop server or
third-party dashboard a required part of the product. Do not send
patient data to external services by default. The browser accesses the
ESP32 on the same local network or through the ESP32's own access point.

## 3. Work incrementally

-   Inspect the repository and preserve working code.
-   Do not rewrite unrelated modules.
-   Implement one stage at a time and keep each stage compiling.
-   State exact files changed and build/test steps.
-   Never claim hardware was tested without evidence.
-   If a detail is unknown, state the assumption and choose a
    conservative documented default.
-   Never present placeholder sensor values as real.

## 4. Pin constraints

Use `HARDWARE_WIRING.md` as the planned map: - I2C SDA D4/GPIO6 - I2C
SCL D5/GPIO7 - Battery ADC D1/GPIO3 through a divider - SOS D2/GPIO4,
button to GND, `INPUT_PULLUP` - OK/Cancel D3/GPIO5, button to GND,
`INPUT_PULLUP` - LED D10/GPIO10 with series resistor - Planned I2S: BCLK
D0/GPIO2, WS D7/GPIO20, DIN D8/GPIO8. Audio is later stage; check
strapping-pin implications.

Do not connect the LiPo until the power circuit is reviewed. OLED was
previously found at `0x3C`. MAX30102 raw readings have been observed;
stable vital-sign estimates remain under test.

## 5. Code quality

-   Keep sensor acquisition, fall logic, device state, display, web
    server, reminders and audio separate.
-   Avoid long blocking delays in normal operation.
-   Use non-blocking timers for reminders and fall cancellation.
-   Debounce physical buttons.
-   Protect shared state if asynchronous callbacks or multiple execution
    contexts are used.
-   Check sensor initialization and return values.
-   Mark disconnected sensors and stale values explicitly.
-   Make thresholds configurable and explain their basis.
-   Bound event history, message length, reminder count and HTTP request
    sizes.
-   Do not write every high-frequency sample to flash.

## 6. MAX30102 requirements

-   Prefer the installed SparkFun MAX3010x library unless a tested
    alternative already exists.
-   `spo2_algorithm.h` defines a `BUFFER_SIZE` macro. Do not declare a
    sketch variable with the same name; use `SAMPLE_BUFFER_SIZE`.
-   Check the library's actual sample-rate and averaging behavior before
    calculating sample timestamps. Do not assume FIFO cadence without
    verification.
-   Never fabricate HR/SpO2 when the algorithm reports invalid data.
-   Publish validity and update times to the webpage.
-   These readings are prototype estimates, not medical-grade outputs.

## 7. ESP32-hosted webpage

-   Serve static assets from firmware flash or another documented
    ESP32-local storage method.
-   Implement a documented JSON API; suggested routes are in
    `ARCHITECTURE.md`.
-   Choose one update method initially: polling, WebSocket or
    Server-Sent Events.
-   API responses must reflect actual firmware state.
-   Display `Unavailable` for invalid readings and stale indicators for
    old readings.
-   Show last-update time and network status.
-   Make the UI responsive on mobile and desktop.
-   Sanitize text rendered in the browser and validate requests.
-   Use meaningful HTTP status codes.
-   Report a POST action as successful only after the ESP32 responds
    successfully.
-   Never expose Wi-Fi passwords in frontend source or logs.
-   Do not imply internet-wide access; local hosting is the default.

## 8. Authoritative data model

Use one device-state model for both OLED and webpage. Include: - HR
value, validity and update time - SpO2 value, validity and update time -
MAX30102 status - MPU6050 status and motion values - fall state and
recent events - SOS state and recent events - Wi-Fi status and IP
address - battery value/validity when implemented - current message -
reminder definitions and acknowledgement state - last update time

Represent invalid readings using validity flags/null fields, not
plausible substitute numbers.

## 9. Fall workflow

Use a state machine, not a single threshold-only `if`: - `NORMAL` -
`SUSPECTED_FALL` - `CANCELLED` - `ALERT_SENT` or equivalent

A suspected fall starts a configurable 15-second cancellation window and
local warning. OK/Cancel cancels during the window. SOS bypasses the
fall timer and records an immediate local SOS event. The webpage shows
events once recorded by the ESP32. Do not claim a caretaker was notified
outside the device unless a real notification channel is separately
implemented.

## 10. Reminders and messages

Caretaker input comes from the ESP32-hosted webpage. Validate schedule
and text length. Store configuration locally where feasible. Display
messages on OLED when that module is active. Audio playback is optional
until the MAX98357A stage. Track queued/displayed/acknowledged states
where supported.

## 11. Required response for coding tasks

For each implementation task, report: 1. Feature being implemented. 2.
Files created/modified. 3. Wiring/configuration assumptions. 4. Complete
code for changed files when requested. 5. Library dependencies and
installation notes. 6. Exact compile/upload/test steps. 7. Expected
serial output or browser behavior. 8. Known limitations and next test.
9. Which acceptance criteria passed versus remain untested.

## 12. Definition of done

A feature is complete only when code compiles for the selected board,
has a reproducible test, handles failure paths, has updated
documentation, and displays actual patient state on the ESP32-hosted
webpage where relevant. No external cloud dependency may be silently
introduced.
