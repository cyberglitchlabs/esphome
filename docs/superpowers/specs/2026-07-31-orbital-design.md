# HackerBox 0129 "Orbital" — ESPHome Package Design

## Summary

Add a new ESPHome package (`packages/orbital_base.yaml`) for the HackerBox #0129
"Orbital" kit: an ESP32-S3 SuperMini driving five round 1.28" GC9A01A SPI displays
(240×240 each) plus three navigation buttons and an onboard addressable RGB LED.
The package renders four cyclable "sets" of Home Assistant data across all five
screens simultaneously, using only first-party ESPHome components (`ili9xxx`,
`lvgl`, `light`, `homeassistant` sensors) — no `external_components:`, no custom
C++ libraries, no direct third-party API calls from firmware.

This is a from-scratch ESPHome port of the concept behind the open-source
[info-orbs](https://github.com/brettdottech/info-orbs) project (Arduino/PlatformIO
+ TFT_eSPI), adapted to this repo's package conventions and to the HackerBox kit's
actual ESP32-S3 wiring, not a port of its firmware.

## Hardware

Pinout confirmed from the HackerBox 0129 Instructables guide (Step 5), which
differs from the upstream info-orbs project (that project targets a plain ESP32
with different GPIO numbers):

| Function | GPIO | Substitution |
|---|---|---|
| SPI MOSI | 11 | `orbital_mosi_pin` |
| SPI SCLK | 12 | `orbital_sclk_pin` |
| SPI MISO | unused (-1) | — |
| Shared DC | 9 | `orbital_dc_pin` |
| Shared RST | 13 | `orbital_reset_pin` |
| Display 1 CS | 4 | `orbital_cs1_pin` |
| Display 2 CS | 5 | `orbital_cs2_pin` |
| Display 3 CS | 6 | `orbital_cs3_pin` |
| Display 4 CS | 7 | `orbital_cs4_pin` |
| Display 5 CS | 8 | `orbital_cs5_pin` |
| Button Left | 2 | `orbital_button_left_pin` |
| Button OK | 1 | `orbital_button_ok_pin` |
| Button Right | 3 | `orbital_button_right_pin` |
| Onboard RGB LED (WS2812) | 48 | `orbital_led_pin` |

All pins are substitutions with the above as defaults, so a user can override them
if their physical unit differs. The docs will flag that these came from the
Instructables guide (not board silkscreen) and should be double-checked against
the physical unit before flashing.

**Known constraint:** the ESP32-S3 SuperMini typically ships without PSRAM. LVGL
does not strictly require it, but a 100% buffer for a single 240×240 RGB565
display is 115,200 bytes, and five of those held at once would exceed
available SRAM by itself, before accounting for WiFi stack and application
code. LVGL's `buffer_size` is expressed as a percentage of display size, and
ESPHome falls back to a smaller allocation automatically if a full buffer
can't be allocated — but relying on the automatic fallback across five
simultaneous LVGL instances is a real risk, so the package sets
`buffer_size: 10%` explicitly on each of the five `lvgl:` blocks (~11.5KB each,
~57.6KB combined) and keeps widgets simple (labels, arcs — no heavy animation)
to stay within budget. This value is a starting point, not a guarantee — Task
3 in the implementation plan calls out watching for boot-time allocation
failures and lowering it further if needed. If a user's board does have
PSRAM, they can raise `buffer_size` via
substitution.

## Architecture

- One shared `spi:` bus (MOSI/SCLK), five `display: platform: ili9xxx` entries
  (model `GC9A01A`, first-class named model — no manual init sequence needed),
  differing only by `cs_pin`, sharing `dc_pin`/`reset_pin`.
- Five `lvgl:` blocks, one per display (`displays: [display_N]`), each defining
  the same four `pages:` (one page per "set" below) so all five screens change
  together.
- One `globals: current_set` int (0–3) is the single source of truth for which
  set is active. Advancing it triggers `lvgl.page.show` on all five LVGL
  instances in lockstep.
- Three `binary_sensor: platform: gpio` entries for Left/OK/Right, using
  `on_click` with `min_length`/`max_length` to distinguish short vs. long press
  on OK.
- One `light:` entity (native ESP32 RGB/WS2812 platform, GPIO48) used only as a
  diagnostic status indicator (WiFi/HA connection state) — not a user-content
  screen.

## The Four Sets

Each set maps one Home Assistant entity substitution to each of the five
screens. All content is HA-fed (`platform: homeassistant` sensor/text_sensor) —
no API keys, no HTTP calls, no JSON parsing in firmware. This is a deliberate
departure from upstream info-orbs (which calls Visual Crossing/Twelve Data
directly from the ESP32): Home Assistant already has mature integrations and
templating for weather, stocks, energy, etc., so the device just displays
whatever entity state HA gives it.

| Set | Screen 1 | Screen 2 | Screen 3 | Screen 4 | Screen 5 |
|---|---|---|---|---|---|
| Clock | Time | Date | Day of week | Uptime | WiFi signal |
| Weather | Current temp | Humidity | Condition | Wind | Forecast hi/lo |
| Presence | Person 1 | Person 2 | Person 3 | Person 4 | Person 5 |
| Custom | HA sensor 1 | HA sensor 2 | HA sensor 3 | HA sensor 4 | HA sensor 5 |

- **Clock** uses local `time: homeassistant` (no substitution needed) plus the
  device's own uptime/WiFi-signal sensors — same pattern as the diagnostic block
  used across every package in this repo.
- **Weather** substitutions default to `weather.home` (HA's default weather
  entity ID) for condition/forecast, and `sensor.outdoor_temperature` /
  `sensor.outdoor_humidity` style placeholders for temp/humidity/wind;
  documented as overridable to any HA weather integration's actual entity IDs.
- **Presence** substitutions default to generic placeholders
  `person.person_1` .. `person.person_5`. Note: `bermuda_base.yaml` is a bare
  BLE proxy — it doesn't expose person/presence entities itself, it only feeds
  data to the Bermuda HA integration, which creates its own device_tracker/
  distance entities under names Bermuda controls, not this repo. So there's no
  fixed naming convention to default to; the docs will note that if a user runs
  Bermuda, they should point these substitutions at whatever entity IDs the
  Bermuda integration actually created in their HA instance.
- **Custom** substitutions default to generic placeholders
  `sensor.custom_1` .. `sensor.custom_5`, letting a user point them at anything
  HA already tracks (energy, stocks, whatever) without the package needing to
  know what it is.

## Navigation

- **Left** — previous set (wraps 0↔3)
- **Right** — next set (wraps 0↔3)
- **OK short press** — force a manual refresh of the current set's entities
- **OK long press (≥600ms)** — toggle auto-cycle mode, which advances the set
  automatically every `orbital_autocycle_interval` (substitution, default 30s,
  off by default)

## Error Handling

If an HA entity is unavailable or the API connection drops, the affected LVGL
label shows ESPHome's default unavailable-state rendering ("—" / blank) rather
than any custom retry logic — consistent with how `homeassistant:` sensors
already behave elsewhere in this repo (e.g. `weather_entity` in
`wopr_base.yaml`). No bespoke reconnect/backoff logic is needed; ESPHome's API
client already reconnects automatically, and `api: reboot_timeout: 0s` (required
platform setting per this repo's conventions) prevents the device from rebooting
while HA is unreachable.

## Testing

`tests/test_orbital.yaml` will `!include ../packages/orbital_base.yaml`
(local path, per this repo's four-layer pattern) and be added to the CI matrix
in `.github/workflows/validate.yml`. Validation is `esphome config
tests/test_orbital.yaml` — there's no unit-test layer in this repo; whole-file
YAML/config validation is the equivalent of a test, same as every other package.

## Files to Add

- `packages/orbital_base.yaml` — hardware package (this design's core deliverable)
- `examples/orbital_example.yaml` — minimal user-facing config referencing the
  package via `remote_package`
- `tests/test_orbital.yaml` — CI validation wrapper
- `docs/orbital.md` — hardware/wiring guide, pin table, substitutions reference,
  entity table (same shape as `docs/rv_tv.md`)
- Update `.github/workflows/validate.yml` to add the new test file to the matrix
- Update root `README.md` device table with the new package/docs links

## Out of Scope (v1)

- Direct third-party API integration (weather/stock APIs) in firmware — always
  routed through HA instead
- Touch input (displays are SPI-only in this kit; no touch controller)
- Per-screen independent paging (all five screens always show the same set)
- User-configurable RGB LED behavior/effects (fixed diagnostic use only)
