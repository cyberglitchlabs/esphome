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

**Button polarity note:** the three navigation buttons are wired active-high
(pressing connects the pin to 3.3V; confirmed with a multimeter), not the
active-low pattern you might expect. `orbital_base.yaml` configures them with
`mode: INPUT_PULLDOWN` (idle reads low, press reads high) rather than
`INPUT_PULLUP` + `inverted: true`. This matches the original Arduino
reference firmware's `BUTTON_MODE INPUT_PULLDOWN` constant. If you rewire a
button to a different pin or a different Orbital unit turns out to be wired
active-low instead, adjust the pin's `mode`/`inverted` settings accordingly.

**Reset pin note:** only `display_1`'s config carries `reset_pin` in
`orbital_base.yaml`. Since GPIO13 is physically wired to all five panels,
toggling it resets every panel at once — if each display's `mipi_spi` config
declared its own `reset_pin`, every subsequent display's setup would re-reset
the panels that already finished initializing, leaving only the
last-initialized display actually showing anything (the classic symptom:
one screen lit, the other four blank). This applies regardless of display
driver — both `ili9xxx` (the driver this package used previously) and its
successor `mipi_spi` toggle `reset_pin` per-instance during `setup()`. Keep
the reset wired to a single display only when editing this package.

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
| Clock | Analog clock face | Date | Day of week | Uptime | WiFi signal |
| Weather | Current temp | Humidity | Condition (icon) | Wind | Forecast |
| Presence | Person 1 | Person 2 | Person 3 | Person 4 | Person 5 |
| Custom | HA sensor 1 | HA sensor 2 | HA sensor 3 | HA sensor 4 | HA sensor 5 |

Weather, Presence, and Custom are entirely HA-fed — point the substitutions
below at real entities in your Home Assistant instance.

Every screen in these three sets uses the same layout: a Material Design Icons
glyph on top, the value beneath it with its unit attached, and — on Presence
and Custom — a caption naming what is being shown. Weather screens need no
caption because the icon plus a value like `76°F` already says everything. The
unit is welded to the value rather than shown on its own line, so a wind
reading always renders as `9 mph`, never a bare `9`.

**Clock screen 1** is an LVGL `meter` widget (analog hour/minute hands), not
digital text — it's driven every second by an interval that reads
`id(orbital_time).now()` and updates the `minute_hand`/`hour_hand`
indicators directly, rather than a `lvgl.label.update`.

**Weather screens 1–4 all read from one entity.** Set `orbital_weather_entity`
to any HA `weather.*` entity and the package pulls `temperature`, `humidity`
and `wind_speed` off its *attributes*, while its plain state drives the
condition icon. Reading attributes rather than four standalone `sensor.*`
entities means there is a single thing to configure, and — because the numeric
`homeassistant` platform hands ESPHome a float instead of a state string — the
values can be rounded for display. HA's raw state for wind speed is often
something like `8.780000000000001`; the screen shows `9 mph`.

Because attributes arrive as bare numbers with no unit attached, set
`orbital_weather_temp_unit` / `orbital_weather_wind_unit` to match what your
weather entity actually reports in. Check its `temperature_unit` and
`wind_speed_unit` attributes — they default to `°F` and ` mph`.

**Weather screen 3** (condition) renders as an icon, not text. The
`weather_condition_ts` text sensor maps Home Assistant's `weather.*` entity
condition strings (`sunny`, `cloudy`, `partlycloudy`, `rainy`, etc.) to glyphs
from a Material Design Icons font subset (`weather_icons`, fetched from
jsDelivr's `@mdi/font` package — see the `font:` block in
`orbital_base.yaml`). An unrecognized condition string falls back to a
"sunny-off" glyph.

**Presence screens** show an icon that tracks state: a green house when the
person is `home`, an amber walking figure when `not_home`, and a grey question
mark for anything else (`unknown`, `unavailable`, or a named zone). The state
text is shortened to `Home`/`Away` and truncated to ten characters, and the
person's name comes from `orbital_personN_label`.

**Custom screens** take four substitutions each — the entity, an icon, a
caption, and a unit appended to numeric states. Numeric states are rounded to
one decimal; anything non-numeric is truncated to ten characters. The `_icon`
value must be one of the codepoints already baked into the `weather_icons`
font, because ESPHome subsets fonts at build time and rejects a glyph list
containing duplicates — so the palette is fixed and slots may freely reuse the
same icon. The baked palette covers gauge, flash, lightning-bolt, currency,
battery, wifi, chart-line, clock, leaf, car, home-thermometer, water,
printer-3d, server-network, bell and package. To use anything else, add its
codepoint to the `font:` glyph list as well as setting the substitution.

**Dark theme:** all five displays use LVGL's `dark_mode` theme (dark
backgrounds, light text) for readability. It's declared once, on `lvgl_1`
only — ESPHome only allows one `theme:` block when multiple LVGL instances
share a device (it's a single global define, not per-instance), so don't
copy it onto `lvgl_2`..`lvgl_5` if you're editing this package.

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

## Forecast Setup

Weather screen 5 shows a forecast, and it is the one screen that needs setup
on the Home Assistant side rather than just an entity name.

Home Assistant removed the `forecast` attribute from `weather.*` entities in
release 2024.4. Forecast data now comes only from the `weather.get_forecasts`
action, which returns a *response* — and ESPHome's `homeassistant` platforms
can only subscribe to entity states and attributes, never invoke an action or
read its response. So the forecast has to be flattened into a plain entity in
HA first, and `orbital_weather_forecast_entity` pointed at that.

Create an `input_text` helper (Settings → Devices & Services → Helpers →
Create Helper → Text) named so it lands at
`input_text.orbital_forecast_summary`, with a maximum length of 32. Then add
an automation that keeps it filled:

```yaml
alias: Orbital forecast summary
mode: single
triggers:
  - trigger: time_pattern
    minutes: "/30"
  - trigger: homeassistant
    event: start
actions:
  - action: weather.get_forecasts
    target:
      entity_id: weather.home     # your weather entity
    data:
      type: daily
    response_variable: fc
  - action: input_text.set_value
    target:
      entity_id: input_text.orbital_forecast_summary
    data:
      value: >-
        {% set f = fc['weather.home'].forecast %}
        {% if f %}{{ f[0].temperature | round | int }}/{{ f[0].templow | round | int }}{% else %}--{% endif %}
```

That writes today's high and low as `77/52`. Keep whatever you render under
about ten characters — the value is truncated to ten before it reaches the
display, and a 240px round screen has no room for more at this font size.

The `{% if f %}` guard matters: `get_forecasts` returns an empty list when the
weather integration is still starting up or has lost its upstream, and
indexing `f[0]` unguarded would fail the automation every time that happened.

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
| `orbital_weather_entity` | `weather.home` | Feeds Weather screens 1–4 via its attributes |
| `orbital_weather_forecast_entity` | `input_text.orbital_forecast_summary` | Weather screen 5 — see [Forecast Setup](#forecast-setup) |
| `orbital_weather_temp_unit` | `°F` | Appended to the temperature value |
| `orbital_weather_wind_unit` | ` mph` | Appended to the wind value |
| `orbital_person1_entity`..`orbital_person5_entity` | `person.person_1`..`person.person_5` | Presence screens 1–5 |
| `orbital_person1_label`..`orbital_person5_label` | `Person 1`..`Person 5` | Caption naming each presence screen |
| `orbital_custom1_entity`..`orbital_custom5_entity` | `sensor.custom_1`..`sensor.custom_5` | Custom screens 1–5 |
| `orbital_custom1_icon`..`orbital_custom5_icon` | `\U000F029A` (mdi:gauge) | Icon for each custom screen |
| `orbital_custom1_label`..`orbital_custom5_label` | `Custom 1`..`Custom 5` | Caption for each custom screen |
| `orbital_custom1_unit`..`orbital_custom5_unit` | `""` | Appended to numeric custom values |

### Migrating from the pre-icon-hero package

`orbital_weather_temp_entity`, `orbital_weather_humidity_entity`,
`orbital_weather_condition_entity` and `orbital_weather_wind_entity` have been
**removed**. Replace all four with a single `orbital_weather_entity` pointing
at a `weather.*` entity. Supporting both forms was considered and rejected:
ESPHome cannot conditionally choose between reading a sensor's state and
reading an entity's attribute, so it would have meant shipping two parallel
component trees for the same four screens.

If you relied on standalone `sensor.*` entities that have no parent weather
entity, point the Custom set at them instead — that set is unchanged apart
from its new icon/label/unit keys.
