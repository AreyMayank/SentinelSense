# CareVoice Edge

## Purpose

CareVoice Edge is a wearable patient-safety prototype based on the Seeed
Studio XIAO ESP32-C3. It reads patient sensors, detects possible falls,
supports manual SOS, displays local information, provides
caretaker-configured reminders, and exposes a monitoring dashboard
**hosted directly by the ESP32**.

## Non-negotiable requirement

**All patient data must be visible on the webpage served by the ESP32.**
The ESP32 hosts the HTML/CSS/JavaScript and local API. Do not make cloud
services, Firebase, a separate server, or a remote database required for
the core product. The browser connects to the ESP32 on the same local
network or through the ESP32's own Wi-Fi access point. Local hosting
does not automatically mean internet-wide access.

## Hardware

-   XIAO ESP32-C3
-   MAX30102 for estimated heart rate and SpO2
-   MPU6050 for motion and suspected-fall detection
-   1.3-inch I2C OLED, likely SH1106 128x64, previously found at `0x3C`
-   MAX98357A I2S amplifier and 8-ohm speaker (later stage)
-   SOS and OK/Cancel buttons
-   Status LED and resistor
-   3.7 V, 700 mAh LiPo, slide switch and battery divider (later stage)

## Current status

-   XIAO ESP32-C3 selected and used with Arduino IDE.
-   OLED basic text output tested with U8g2; I2C scan found `0x3C`.
-   MAX30102 detected and raw red/IR readings obtained. Stable HR/SpO2
    readings remain under validation.
-   MPU6050, combined operation, fall logic, ESP32-hosted webpage,
    buttons, audio and battery operation remain to be implemented or
    validated.

## Documentation

-   `PROJECT_SPEC.md`: requirements, features and acceptance criteria.
-   `ARCHITECTURE.md`: firmware structure, data flow and ESP32-hosted
    web design.
-   `HARDWARE_WIRING.md`: pin map, connections and electrical cautions.
-   `AI_AGENT_GUIDE.md`: instructions for AI coding agents.

## Principles

1.  Test one component at a time.
2.  Keep code modular and compile after each stage.
3.  All patient dashboard data comes from the ESP32.
4.  Never fabricate sensor readings; mark invalid/stale values clearly.
5.  HR and SpO2 are prototype estimates, not medical-grade readings.
6.  Do not connect the LiPo until the power path and protection are
    verified.
7.  Do not call a feature complete until its acceptance test passes.
