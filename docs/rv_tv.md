# RV TV Lift Controller

Hardware guide for the ESP32-based RV TV motor lift controller package (`rv_tv_base.yaml`).

## Overview

This device controls a motorized TV lift mechanism in an RV using an ESP32. It:

- Drives a DC motor via two relays configured as an H-bridge (up/down)
- Reads two ADC-based limit sensors (raised/lowered end-stops via voltage divider)
- Controls a TV power relay (auto-on when lowered, auto-off when raised)
- Responds to a physical momentary push button
- Exposes position and controls to Home Assistant

## GPIO Pinout

| GPIO  | Function              | Default substitution  |
|-------|-----------------------|-----------------------|
| GPIO26 | TV power relay       | `tv_power_pin`        |
| GPIO32 | Motor UP relay       | `motor_up_pin`        |
| GPIO33 | Motor DOWN relay     | `motor_down_pin`      |
| GPIO19 | Lift button (INPUT_PULLUP) | `button_pin`     |
| GPIO35 | ADC — lowered limit  | `adc_lowered_pin`     |
| GPIO34 | ADC — raised limit   | `adc_raised_pin`      |

## Wiring

### Motor Relays

Use two SPDT or SPST relays wired as an H-bridge to reverse motor polarity:

- **Relay UP (GPIO32)**: Closes to apply 12 V in the "raise" direction
- **Relay DOWN (GPIO33)**: Closes to apply 12 V in the "lower" direction
- Both relays use ESPHome `interlock` to prevent simultaneous activation

### Limit Sensors

Each end-stop is read via a resistive voltage divider into an ADC-capable pin (GPIO34/35
are input-only, 0–3.9 V with `attenuation: auto`):

```
12V ──┬── R1 (e.g. 10kΩ) ──┬── GPIO3x
      │                     │
      └── (switch) ─────────┴── GND
```

When the mechanical limit switch closes, the divider output rises above `adc_threshold`
(default 2.5 V). The firmware inverts this signal so `true` = switch triggered.

Adjust `adc_threshold` in your substitutions if your divider uses different resistor values.

### TV Power Relay

- GPIO26 controls a relay that switches 12 V (or mains via opto-isolated relay) to the TV
- Automatically turned **on** when the TV reaches the lowered position
- Automatically turned **off** when the TV reaches the raised position

### Push Button

- Normally-open momentary button between GPIO19 and GND (internal pull-up enabled)
- Single press toggles movement; press again while moving to stop mid-travel
- If stopped mid-travel, next press reverses direction from last movement

## Button Logic

| TV State         | Button press result                         |
|------------------|---------------------------------------------|
| Fully lowered    | Start raising                               |
| Fully raised     | Start lowering                              |
| Moving (any dir) | Stop                                        |
| Stopped mid-way  | Reverse from last direction                 |

## Home Assistant Entities

| Entity                     | Type          | Category    |
|----------------------------|---------------|-------------|
| TV Power                   | Switch        | —           |
| TV Position                | Sensor (%)    | —           |
| TV Lowered                 | Binary Sensor | —           |
| TV Raised                  | Binary Sensor | —           |
| Motor Up                   | Switch        | config      |
| Motor Down                 | Switch        | config      |
| TV Lift Button             | Binary Sensor | diagnostic  |
| TV Lowered Voltage         | Sensor (V)    | diagnostic  |
| TV Raised Voltage          | Sensor (V)    | diagnostic  |
| WiFi Signal / Uptime / etc.| Sensors       | diagnostic  |

## Quick Start

Copy `examples/rv_tv_example.yaml`, set your `wifi_ssid`, `wifi_password`,
`api_encryption_key`, and `ota_password` in `secrets.yaml`, then flash:

```bash
esphome run examples/rv_tv_example.yaml
```

## Substitutions Reference

| Key                  | Default    | Description                              |
|----------------------|------------|------------------------------------------|
| `tv_power_pin`       | GPIO26     | TV power relay GPIO                      |
| `motor_up_pin`       | GPIO32     | Motor up relay GPIO                      |
| `motor_down_pin`     | GPIO33     | Motor down relay GPIO                    |
| `button_pin`         | GPIO19     | Physical button GPIO                     |
| `adc_lowered_pin`    | GPIO35     | ADC pin for lowered limit sensor         |
| `adc_raised_pin`     | GPIO34     | ADC pin for raised limit sensor          |
| `adc_threshold`      | `"2.5"`    | Voltage threshold (V) for limit detection|
| `adc_update_interval`| `50ms`     | ADC polling rate                         |
