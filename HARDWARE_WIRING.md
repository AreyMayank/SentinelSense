# SentinelSense: Hardware Connections and Electrical Notes

## 1. Controller

Board: Seeed Studio XIAO ESP32-C3. Use Arduino IDE and the correct XIAO
ESP32-C3 board target.

## 2. Current planned pin map

  Function           XIAO pin     ESP32-C3 GPIO Notes
  ------------------ ---------- --------------- -------------------------------------------
  Shared I2C SDA     D4                   GPIO6 MAX30102, MPU6050, OLED
  Shared I2C SCL     D5                   GPIO7 MAX30102, MPU6050, OLED
  Battery ADC        D1                   GPIO3 Through divider only
  SOS button         D2                   GPIO4 Button to GND, INPUT_PULLUP
  OK/Cancel button   D3                   GPIO5 Button to GND, INPUT_PULLUP
  Status LED         D10                 GPIO10 Use series resistor
  I2S BCLK           D0                   GPIO2 Planned; check strapping-pin implications
  I2S LRC/WS         D7                  GPIO20 Planned
  I2S DIN            D8                   GPIO8 Planned; check strapping-pin implications
  Unallocated        D6                  GPIO21 Reserve unless needed
  Avoid for now      D9                   GPIO9 Boot-strapping concerns

This is the current project plan. Confirm the exact board pinout and
module electrical requirements before final assembly. Avoid assigning
multiple functions to one GPIO.

## 3. Shared I2C bus

MAX30102, MPU6050 and OLED are intended to share SDA D4/GPIO6 and SCL
D5/GPIO7. Connect SDA-to-SDA and SCL-to-SCL with common ground. Scan
addresses before integration. The OLED was previously detected at 7-bit
address `0x3C`; some scanners or documentation may show shifted address
`0x78`.

The planned bus uses 3.3 V logic. Check each breakout's input-voltage
limits, regulator and pull-ups. Multiple modules may each contain I2C
pull-ups; too many parallel pull-ups can cause bus issues.

## 4. MAX30102

  -----------------------------------------------------------------------
  Module pin                          Connection
  ----------------------------------- -----------------------------------
  VIN                                 XIAO 3V3, if supported by this
                                      breakout

  GND                                 XIAO GND

  SDA                                 D4 / GPIO6

  SCL                                 D5 / GPIO7

  INT                                 Leave disconnected for polling

  RD, IRD, extra GND                  Follow the specific breakout
                                      documentation; leave unused unless
                                      needed
  -----------------------------------------------------------------------

Raw red/IR readings have previously been observed. Raw data alone does
not establish accurate HR/SpO2. Test with steady finger placement.

## 5. MPU6050

  Module pin   Connection
  ------------ ---------------------------------------------
  VCC          Supply supported by the specific breakout
  GND          XIAO GND
  SDA          D4 / GPIO6
  SCL          D5 / GPIO7
  AD0          Default state unless changing I2C address
  INT          Leave disconnected for initial polling test
  XDA, XCL     Auxiliary I2C; leave unused

First scan the bus and read raw acceleration/gyroscope values. Develop
fall logic only after raw readings are stable.

## 6. OLED

  OLED pin   Connection
  ---------- ------------------------
  VCC        XIAO 3V3, if supported
  GND        XIAO GND
  SDA        D4 / GPIO6
  SCL        D5 / GPIO7

The tested display used U8g2 with an SH1106 128x64 configuration and was
detected at `0x3C`. Confirm controller type before choosing a U8g2
constructor. Install U8g2 if `U8g2lib.h` is missing.

## 7. Buttons

For each button, connect one terminal to its assigned GPIO and the other
to GND. Configure as `INPUT_PULLUP`; pressed reads LOW. Implement
software debouncing.

## 8. Status LED

Suggested: D10/GPIO10 -\> 220-ohm resistor -\> LED anode; LED cathode
-\> GND. Verify polarity and current.

## 9. Battery-voltage monitoring

Proposed divider: - Battery positive -\> 100 kΩ -\> ADC node - ADC node
-\> 100 kΩ -\> GND - ADC node -\> D1/GPIO3 - Optional 0.1 µF capacitor
from ADC node to GND

Equal resistors make the ADC node approximately half the battery
voltage. Firmware must reconstruct battery voltage and calibrate ADC
readings. Confirm ADC input limits. Never connect battery positive
directly to an ADC pin.

A 3.7 V LiPo requires a suitable charging/protection and power
arrangement. Do not assume the board revision supports direct LiPo
connection or charging. Remain USB-powered until the battery circuit is
verified.

## 10. MAX98357A and speaker (later)

-   BCLK -\> D0/GPIO2
-   LRC/WS -\> D7/GPIO20
-   DIN -\> D8/GPIO8
-   VIN/GND -\> supply supported by the exact module
-   Speaker connects across SPK+ and SPK−

Do not connect SPK− to GND. The speaker output is bridge-tied. Check
supply requirements and peak current. GPIO2 and GPIO8 are strapping
pins; verify boot behavior before adopting this plan.

## 11. Safe staged test order

1.  XIAO alone over USB.
2.  OLED alone.
3.  MAX30102 alone.
4.  MPU6050 alone.
5.  All I2C devices together.
6.  Buttons and LED.
7.  ESP32-hosted webpage and Wi-Fi.
8.  Amplifier/speaker.
9.  Battery divider and power system after electrical review.

Disconnect USB before changing wiring. Never change wiring while
powered. Do not attach the LiPo until charging, protection, switch and
power path have been reviewed.
