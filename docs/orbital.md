# HackerBox 0129 Orbital

Hardware guide for the ESP32-S3 five-display Orbital package (`orbital_base.yaml`).

## Overview

This device drives five round 1.28" GC9A01A displays (240x240 each) over a
shared SPI bus using LVGL, with three buttons for navigation and an onboard
addressable RGB LED as a status indicator. It:

- Shows one of four "sets" across all five screens at once: Clock, Weather,
  Presence, Custom
- Pulls all non-local content from Home Assistant entities — no API keys or
  third-party HTTP calls in firmware
- Navigates via Left/Right (change set) and OK (short-press: redraw;
  long-press: toggle auto-cycle)
- Indicates WiFi/HA connection state via the onboard RGB LED

**Pinout note:** the GPIO assignments below come from the HackerBox 0129
Instructables guide (Step 5), not the board silkscreen, and differ from the
upstream [info-orbs](https://github.com/brettdottech/info-orbs) project (which
targets a plain ESP32, not the S3 SuperMini used in this kit). Double-check
against your physical unit before flashing — if it differs, override the
relevant pin substitution rather than editing the package.

## GPIO Pinout

| GPIO | Function | Substitution |
|---|---|---|
| GPIO11 | SPI MOSI | `orbital_mosi_pin` |
| GPIO12 | SPI SCLK | `orbital_sclk_pin` |
| GPIO9 | Shared DC (all displays) | `orbital_dc_pin` |
| GPIO13 | Shared RST (all displays) | `orbital_reset_pin` |
| GPIO4 | Display 1 CS | `orbital_cs1_pin` |
| GPIO5 | Display 2 CS | `orbital_cs2_pin` |
| GPIO6 | Display 3 CS | `orbital_cs3_pin` |
| GPIO7 | Display 4 CS | `orbital_cs4_pin` |
| GPIO8 | Display 5 CS | `orbital_cs5_pin` |
| GPIO2 | Button Left | `orbital_button_left_pin` |
| GPIO1 | Button OK | `orbital_button_ok_pin` |
| GPIO3 | Button Right | `orbital_button_right_pin` |
| GPIO48 | Onboard RGB LED (WS2812) | `orbital_led_pin` |

MISO is unused — the displays are write-only.

## Wiring

All five displays share one SPI clock/data line and DC/RST line, and are
individually addressed by their own chip-select pin. If you rewire a display
to a different CS pin, override the matching `orbital_csN_pin` substitution —
don't renumber which screen shows which content, since screen position (1–5)
is fixed per set in the package.

## Display Sets

Each set puts one piece of content on each of the five screens simultaneously.
Screens are numbered 1–5 left-to-right as laid out on the Orbital PCB.

| Set | Screen 1 | Screen 2 | Screen 3 | Screen 4 | Screen 5 |
|---|---|---|---|---|---|
| Clock | Time | Date | Day of week | Uptime | WiFi signal |
| Weather | Current temp | Humidity | Condition | Wind | Forecast |
| Presence | Person 1 | Person 2 | Person 3 | Person 4 | Person 5 |
| Custom | HA sensor 1 | HA sensor 2 | HA sensor 3 | HA sensor 4 | HA sensor 5 |

Weather, Presence, and Custom are entirely HA-fed — point the substitutions
below at real entities in your Home Assistant instance. The device just
displays whatever state string HA sends; it doesn't parse or format it beyond
what HA already gives it.

## Navigation

| Button | Action |
|---|---|
| Left | Previous set (wraps Clock → Custom) |
| Right | Next set (wraps Clock → Weather → Presence → Custom → Clock) |
| OK (short press, 50–500ms) | Force a redraw of the current set |
| OK (long press, 500ms–5s) | Toggle auto-cycle on/off |

While auto-cycle is on, the set advances automatically every
`orbital_autocycle_interval` (default `30s`).

## Status Light

The onboard RGB LED (GPIO48) is a fixed diagnostic indicator, not a
user-controllable light entity:

| Color | Meaning |
|---|---|
| Red | WiFi disconnected |
| Yellow | WiFi connected, Home Assistant not linked |
| Green | Home Assistant linked |

## Known Constraint: No PSRAM

The ESP32-S3 SuperMini in this kit typically has no PSRAM. Holding a
full-resolution LVGL buffer for all five 240x240 displays at once would
exceed available SRAM, so each of the five `lvgl:` blocks sets
`buffer_size: 10%`. If you see boot-time out-of-memory
errors, lower this further in your device config (it's not currently exposed
as its own substitution — override it per-display via a package merge, or ask
in an issue if you need it promoted to a substitution).

## Home Assistant Entities

| Entity | Type | Category |
|---|---|---|
| Status | Binary Sensor | diagnostic |
| WiFi Signal | Sensor | diagnostic |
| Uptime | Sensor | diagnostic |
| Restart | Button | config |
| Safe Mode | Button | config |
| ESPHome Version | Text Sensor | diagnostic |
| WiFi IP Address / SSID / MAC | Text Sensor | diagnostic |

The five displays, three buttons, and status light are internal-only (not
exposed as separate HA entities) — they're driven entirely by the package's
own automations.

## Quick Start

Copy `examples/orbital_example.yaml`, set your `wifi_ssid`, `wifi_password`,
`api_encryption_key`, and `ota_password` in `secrets.yaml`, override any
`orbital_*_entity` substitutions to point at your real HA entities, then flash:

```bash
esphome run examples/orbital_example.yaml
```

## Substitutions Reference

| Key | Default | Description |
|---|---|---|
| `name` | `orbital` | Device name |
| `friendly_name` | `Orbital` | Device friendly name |
| `orbital_mosi_pin` | GPIO11 | SPI MOSI |
| `orbital_sclk_pin` | GPIO12 | SPI SCLK |
| `orbital_dc_pin` | GPIO9 | Shared DC pin |
| `orbital_reset_pin` | GPIO13 | Shared RST pin |
| `orbital_cs1_pin`..`orbital_cs5_pin` | GPIO4/5/6/7/8 | Per-display chip-select |
| `orbital_button_left_pin` | GPIO2 | Left button |
| `orbital_button_ok_pin` | GPIO1 | OK button |
| `orbital_button_right_pin` | GPIO3 | Right button |
| `orbital_led_pin` | GPIO48 | Onboard RGB LED |
| `orbital_autocycle_interval` | `30s` | Auto-cycle timer while toggled on |
| `orbital_weather_temp_entity` | `sensor.outdoor_temperature` | Weather screen 1 |
| `orbital_weather_humidity_entity` | `sensor.outdoor_humidity` | Weather screen 2 |
| `orbital_weather_condition_entity` | `weather.home` | Weather screen 3 |
| `orbital_weather_wind_entity` | `sensor.outdoor_wind_speed` | Weather screen 4 |
| `orbital_weather_forecast_entity` | `sensor.weather_forecast_summary` | Weather screen 5 |
| `orbital_person1_entity`..`orbital_person5_entity` | `person.person_1`..`person.person_5` | Presence screens 1–5 |
| `orbital_custom1_entity`..`orbital_custom5_entity` | `sensor.custom_1`..`sensor.custom_5` | Custom screens 1–5 |
