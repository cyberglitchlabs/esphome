# HackerBox 0129 "Orbital" ESPHome Package Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a pure-ESPHome package for the HackerBox #0129 "Orbital" kit (ESP32-S3 SuperMini + five round GC9A01A displays + three buttons + onboard RGB LED) that cycles through four Home-Assistant-fed display sets, following this repo's four-layer package convention.

**Architecture:** One shared SPI bus feeds five `ili9xxx`/GC9A01A displays, each driven by its own `lvgl:` instance holding four pages (Clock/Weather/Presence/Custom). A single `globals: current_set` int is the source of truth for which page is active everywhere; a shared script flips all five displays to the matching page. Left/Right buttons change the set, OK short-presses force a redraw and long-presses toggle auto-cycling. All non-local content comes from `text_sensor: platform: homeassistant` entities — no HTTP calls, no API keys in firmware.

**Tech Stack:** ESPHome (esp-idf framework), `ili9xxx` display platform (model `GC9A01A`), `lvgl` component, `esp32_rmt_led_strip` light platform, `homeassistant` text_sensor platform. No `external_components:`, no custom C++.

## Global Constraints

- Design spec: `docs/superpowers/specs/2026-07-31-orbital-design.md` — read it before starting; every task below implements a section of it.
- No `external_components:` and no custom C++ libraries — first-party ESPHome components only.
- No API keys or third-party HTTP calls in firmware — all non-local data comes through `homeassistant:` sensor/text_sensor entities.
- 2-space indentation throughout YAML; 4-space indentation inside `lambda: |-` blocks (per `AGENTS.md`).
- `api: reboot_timeout: 0s` is required (prevents reboot on HA disconnect).
- Every package file must begin with the standard header comment block documenting requirements (per `AGENTS.md`).
- Entity `name:` values are Short Title Case, not device-prefixed (e.g. `"Status"`, not `"Orbital Status"`).
- `entity_category: diagnostic` on status/uptime/WiFi/version entities; `entity_category: config` on restart/safe-mode buttons.
- All hardware pins are substitutions with the confirmed Instructables-guide defaults from the spec's pinout table — never hardcoded inline.
- Validate every task with `esphome config packages/orbital_base.yaml` (install via `pip install esphome` first if not already installed) — this is this repo's equivalent of running tests; there is no other test runner.

---

### Task 1: Package skeleton and diagnostics

**Files:**
- Create: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: nothing (first task)
- Produces: substitutions `name` (default `"orbital"`), `friendly_name` (default `"Orbital"`), `orbital_mosi_pin` (GPIO11), `orbital_sclk_pin` (GPIO12), `orbital_dc_pin` (GPIO9), `orbital_reset_pin` (GPIO13), `orbital_cs1_pin`..`orbital_cs5_pin` (GPIO4/5/6/7/8), `orbital_button_left_pin` (GPIO2), `orbital_button_ok_pin` (GPIO1), `orbital_button_right_pin` (GPIO3), `orbital_led_pin` (GPIO48), `orbital_autocycle_interval` (default `30s`); time component id `orbital_time`; sensor ids `wifi_signal_sensor`, `uptime_sensor`.

- [ ] **Step 1: Install ESPHome if not already available**

Run: `pip install esphome`
Expected: installs cleanly, `esphome version` prints a version number.

- [ ] **Step 2: Write the package skeleton**

Create `packages/orbital_base.yaml` with this content:

```yaml
# HackerBox #0129 Orbital Hardware Package
# This package provides hardware-specific configuration for the Orbital
# ESP32-S3 SuperMini device driving five round 1.28" GC9A01A displays
# (240x240 each) via LVGL, three navigation buttons, and an onboard
# addressable RGB status LED. Pinout confirmed from the HackerBox 0129
# Instructables guide (Step 5) — differs from the upstream info-orbs
# project, which targets a plain ESP32.
#
# Required in your device yaml:
#   esphome:
#     name: your-device-name
#     friendly_name: "Your Device Name"
#
# Usage:
#   packages:
#     device:
#       url: https://github.com/cyberglitchlabs/esphome
#       files: [packages/orbital_base.yaml]
#       ref: main

substitutions:
  name: "orbital"
  friendly_name: "Orbital"

  # SPI bus (shared by all five displays)
  orbital_mosi_pin: GPIO11
  orbital_sclk_pin: GPIO12
  orbital_dc_pin: GPIO9
  orbital_reset_pin: GPIO13

  # Per-display chip-select pins
  orbital_cs1_pin: GPIO4
  orbital_cs2_pin: GPIO5
  orbital_cs3_pin: GPIO6
  orbital_cs4_pin: GPIO7
  orbital_cs5_pin: GPIO8

  # Buttons
  orbital_button_left_pin: GPIO2
  orbital_button_ok_pin: GPIO1
  orbital_button_right_pin: GPIO3

  # Onboard addressable RGB LED
  orbital_led_pin: GPIO48

  # Auto-cycle timing (only used while auto-cycle is toggled on via long-press OK)
  orbital_autocycle_interval: 30s

esphome:
  name: "${name}"
  friendly_name: "${friendly_name}"
  project:
    name: cyberglitchlabs.hackerbox_orbital
    version: "1.0.0"

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  framework:
    type: esp-idf

api:
  reboot_timeout: 0s

logger:

time:
  - platform: homeassistant
    id: orbital_time

web_server:
  port: 80

# ---------------------------------------------------------------------------
# Diagnostic / management entities
# ---------------------------------------------------------------------------

binary_sensor:
  - platform: status
    name: "Status"
    entity_category: diagnostic

sensor:
  - platform: wifi_signal
    id: wifi_signal_sensor
    name: "WiFi Signal"
    update_interval: 60s
    entity_category: diagnostic
  - platform: uptime
    id: uptime_sensor
    name: "Uptime"
    update_interval: 60s
    entity_category: diagnostic

button:
  - platform: restart
    name: "Restart"
    entity_category: config
  - platform: safe_mode
    name: "Safe Mode"
    entity_category: config

text_sensor:
  - platform: version
    name: "ESPHome Version"
    entity_category: diagnostic
  - platform: wifi_info
    ip_address:
      name: WiFi IP Address
      entity_category: diagnostic
    ssid:
      name: WiFi SSID
      entity_category: diagnostic
    mac_address:
      name: WiFi MAC Address
      entity_category: diagnostic
```

- [ ] **Step 3: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS, prints the fully resolved config with no errors.

- [ ] **Step 4: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add Orbital package skeleton with diagnostics"
```

---

### Task 2: SPI bus and five GC9A01A displays

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: substitutions `orbital_mosi_pin`, `orbital_sclk_pin`, `orbital_dc_pin`, `orbital_reset_pin`, `orbital_cs1_pin`..`orbital_cs5_pin` (Task 1)
- Produces: SPI bus id `spi_bus`; display ids `display_1`..`display_5`

- [ ] **Step 1: Add the SPI bus and display entries**

Insert this block after the `web_server:` section and before the diagnostic entities:

```yaml
# ---------------------------------------------------------------------------
# Hardware: Displays (5x round GC9A01A, shared SPI bus)
# ---------------------------------------------------------------------------

spi:
  id: spi_bus
  clk_pin: ${orbital_sclk_pin}
  mosi_pin: ${orbital_mosi_pin}

display:
  - platform: ili9xxx
    model: GC9A01A
    id: display_1
    spi_id: spi_bus
    cs_pin: ${orbital_cs1_pin}
    dc_pin: ${orbital_dc_pin}
    reset_pin: ${orbital_reset_pin}
    auto_clear_enabled: true
    update_interval: never  # LVGL drives redraws; no polling needed
  - platform: ili9xxx
    model: GC9A01A
    id: display_2
    spi_id: spi_bus
    cs_pin: ${orbital_cs2_pin}
    dc_pin: ${orbital_dc_pin}
    reset_pin: ${orbital_reset_pin}
    auto_clear_enabled: true
    update_interval: never
  - platform: ili9xxx
    model: GC9A01A
    id: display_3
    spi_id: spi_bus
    cs_pin: ${orbital_cs3_pin}
    dc_pin: ${orbital_dc_pin}
    reset_pin: ${orbital_reset_pin}
    auto_clear_enabled: true
    update_interval: never
  - platform: ili9xxx
    model: GC9A01A
    id: display_4
    spi_id: spi_bus
    cs_pin: ${orbital_cs4_pin}
    dc_pin: ${orbital_dc_pin}
    reset_pin: ${orbital_reset_pin}
    auto_clear_enabled: true
    update_interval: never
  - platform: ili9xxx
    model: GC9A01A
    id: display_5
    spi_id: spi_bus
    cs_pin: ${orbital_cs5_pin}
    dc_pin: ${orbital_dc_pin}
    reset_pin: ${orbital_reset_pin}
    auto_clear_enabled: true
    update_interval: never
```

- [ ] **Step 2: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS. If it fails on `model: GC9A01A` not being recognized, run `esphome update-all` / check the installed ESPHome version supports it (added as a named `ili9xxx` model in relatively recent ESPHome releases) and upgrade if needed.

- [ ] **Step 3: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add SPI bus and five GC9A01A displays to Orbital package"
```

---

### Task 3: LVGL wiring and the Clock set

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: display ids `display_1`..`display_5` (Task 2); time id `orbital_time`, sensor ids `wifi_signal_sensor`/`uptime_sensor` (Task 1)
- Produces: global `current_set` (int, 0=Clock/1=Weather/2=Presence/3=Custom); lvgl ids `lvgl_1`..`lvgl_5`; page ids `clock_1`..`clock_5`, plus **empty** page shells `weather_1`..`weather_5`, `presence_1`..`presence_5`, `custom_1`..`custom_5` (a single blank label each, filled in by Tasks 4–6); label ids `clock_time_1`, `clock_date_2`, `clock_day_3`, `clock_uptime_4`, `clock_wifi_5`

- [ ] **Step 1: Add the global page index**

Insert after the `time:` section:

```yaml
globals:
  - id: current_set
    type: int
    restore_value: false
    initial_value: "0"
```

- [ ] **Step 2: Add the five LVGL instances with all four pages (Clock filled in, others as blank shells)**

Insert this block right after the `display:` section from Task 2:

```yaml
# ---------------------------------------------------------------------------
# LVGL: one instance per display, four pages each (Clock/Weather/Presence/Custom)
# ---------------------------------------------------------------------------

lvgl:
  - id: lvgl_1
    displays:
      - display_1
    buffer_size: 10%
    pages:
      - id: clock_1
        widgets:
          - label:
              id: clock_time_1
              align: CENTER
              text: "00:00:00"
      - id: weather_1
        widgets:
          - label:
              id: weather_label_1
              align: CENTER
              text: "—"
      - id: presence_1
        widgets:
          - label:
              id: presence_label_1
              align: CENTER
              text: "—"
      - id: custom_1
        widgets:
          - label:
              id: custom_label_1
              align: CENTER
              text: "—"

  - id: lvgl_2
    displays:
      - display_2
    buffer_size: 10%
    pages:
      - id: clock_2
        widgets:
          - label:
              id: clock_date_2
              align: CENTER
              text: "0000-00-00"
      - id: weather_2
        widgets:
          - label:
              id: weather_label_2
              align: CENTER
              text: "—"
      - id: presence_2
        widgets:
          - label:
              id: presence_label_2
              align: CENTER
              text: "—"
      - id: custom_2
        widgets:
          - label:
              id: custom_label_2
              align: CENTER
              text: "—"

  - id: lvgl_3
    displays:
      - display_3
    buffer_size: 10%
    pages:
      - id: clock_3
        widgets:
          - label:
              id: clock_day_3
              align: CENTER
              text: "-"
      - id: weather_3
        widgets:
          - label:
              id: weather_label_3
              align: CENTER
              text: "—"
      - id: presence_3
        widgets:
          - label:
              id: presence_label_3
              align: CENTER
              text: "—"
      - id: custom_3
        widgets:
          - label:
              id: custom_label_3
              align: CENTER
              text: "—"

  - id: lvgl_4
    displays:
      - display_4
    buffer_size: 10%
    pages:
      - id: clock_4
        widgets:
          - label:
              id: clock_uptime_4
              align: CENTER
              text: "0s"
      - id: weather_4
        widgets:
          - label:
              id: weather_label_4
              align: CENTER
              text: "—"
      - id: presence_4
        widgets:
          - label:
              id: presence_label_4
              align: CENTER
              text: "—"
      - id: custom_4
        widgets:
          - label:
              id: custom_label_4
              align: CENTER
              text: "—"

  - id: lvgl_5
    displays:
      - display_5
    buffer_size: 10%
    pages:
      - id: clock_5
        widgets:
          - label:
              id: clock_wifi_5
              align: CENTER
              text: "0 dB"
      - id: weather_5
        widgets:
          - label:
              id: weather_label_5
              align: CENTER
              text: "—"
      - id: presence_5
        widgets:
          - label:
              id: presence_label_5
              align: CENTER
              text: "—"
      - id: custom_5
        widgets:
          - label:
              id: custom_label_5
              align: CENTER
              text: "—"
```

- [ ] **Step 3: Drive the Clock labels from time and diagnostic sensors**

Add an `interval:` block (after the `lvgl:` block) to refresh the time/date/day labels, and add `on_value:` triggers to the existing `wifi_signal_sensor`/`uptime_sensor` (Task 1) to push straight to their labels.

Add this interval block:

```yaml
interval:
  - interval: 1s
    then:
      - lvgl.label.update:
          id: clock_time_1
          text:
            time_format: "%H:%M:%S"
            time: orbital_time
  - interval: 60s
    then:
      - lvgl.label.update:
          id: clock_date_2
          text:
            time_format: "%Y-%m-%d"
            time: orbital_time
      - lvgl.label.update:
          id: clock_day_3
          text:
            time_format: "%A"
            time: orbital_time
```

Now modify the `sensor:` block from Task 1 so `wifi_signal_sensor` and `uptime_sensor` each gain an `on_value:` trigger:

```yaml
sensor:
  - platform: wifi_signal
    id: wifi_signal_sensor
    name: "WiFi Signal"
    update_interval: 60s
    entity_category: diagnostic
    on_value:
      - lvgl.label.update:
          id: clock_wifi_5
          text:
            format: "%.0f dB"
            args: [x]
  - platform: uptime
    id: uptime_sensor
    name: "Uptime"
    update_interval: 60s
    entity_category: diagnostic
    on_value:
      - lvgl.label.update:
          id: clock_uptime_4
          text:
            format: "%.0f s"
            args: [x]
```

- [ ] **Step 4: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS.

- [ ] **Step 5: Watch for LVGL allocation problems (compile, not just config-check)**

This step needs a real board to fully confirm, but note it now: if `esphome compile packages/orbital_base.yaml` (or a later on-device test) fails with an out-of-memory error during LVGL buffer allocation, lower `buffer_size` from `10%` to `5%` on all five `lvgl:` blocks and recheck. Document whatever value ends up working in Task 11 (`docs/orbital.md`).

- [ ] **Step 6: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add LVGL multi-display wiring and Clock set to Orbital package"
```

---

### Task 4: Weather set

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: LVGL label ids `weather_label_1`..`weather_label_5` (Task 3, currently blank shells)
- Produces: substitutions `orbital_weather_temp_entity` (default `sensor.outdoor_temperature`), `orbital_weather_humidity_entity` (default `sensor.outdoor_humidity`), `orbital_weather_condition_entity` (default `weather.home`), `orbital_weather_wind_entity` (default `sensor.outdoor_wind_speed`), `orbital_weather_forecast_entity` (default `sensor.weather_forecast_summary`); text_sensor ids `weather_temp_ts`..`weather_forecast_ts`

- [ ] **Step 1: Add the weather substitutions**

Add these to the `substitutions:` block from Task 1 (near the other substitutions):

```yaml
  # Weather set — point these at any HA weather-related entities you have
  orbital_weather_temp_entity: sensor.outdoor_temperature
  orbital_weather_humidity_entity: sensor.outdoor_humidity
  orbital_weather_condition_entity: weather.home
  orbital_weather_wind_entity: sensor.outdoor_wind_speed
  orbital_weather_forecast_entity: sensor.weather_forecast_summary
```

- [ ] **Step 2: Add the HA text_sensors that feed the Weather page**

Add this block after the `interval:` section from Task 3:

```yaml
# ---------------------------------------------------------------------------
# Home Assistant data: Weather set
# ---------------------------------------------------------------------------

text_sensor:
  - platform: homeassistant
    id: weather_temp_ts
    entity_id: ${orbital_weather_temp_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_1
          text: !lambda "return x;"
  - platform: homeassistant
    id: weather_humidity_ts
    entity_id: ${orbital_weather_humidity_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_2
          text: !lambda "return x;"
  - platform: homeassistant
    id: weather_condition_ts
    entity_id: ${orbital_weather_condition_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_3
          text: !lambda "return x;"
  - platform: homeassistant
    id: weather_wind_ts
    entity_id: ${orbital_weather_wind_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_4
          text: !lambda "return x;"
  - platform: homeassistant
    id: weather_forecast_ts
    entity_id: ${orbital_weather_forecast_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_5
          text: !lambda "return x;"
  - platform: version
    name: "ESPHome Version"
    entity_category: diagnostic
  - platform: wifi_info
    ip_address:
      name: WiFi IP Address
      entity_category: diagnostic
    ssid:
      name: WiFi SSID
      entity_category: diagnostic
    mac_address:
      name: WiFi MAC Address
      entity_category: diagnostic
```

Note: this repo allows only one top-level `text_sensor:` key per file, so this step's block **replaces** the `text_sensor:` block from Task 1 — the `version`/`wifi_info` entries are repeated here rather than duplicated as a second `text_sensor:` key (ESPHome would merge/overwrite, not append, if split across two keys in the same file).

- [ ] **Step 3: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS, and the resolved config shows `weather_temp_ts` etc. reading from the placeholder entity IDs.

- [ ] **Step 4: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add Weather set to Orbital package"
```

---

### Task 5: Presence set

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: LVGL label ids `presence_label_1`..`presence_label_5` (Task 3, currently blank shells); merges into the single `text_sensor:` block established in Task 4
- Produces: substitutions `orbital_person1_entity`..`orbital_person5_entity` (defaults `person.person_1`..`person.person_5`); text_sensor ids `presence_person1_ts`..`presence_person5_ts`

- [ ] **Step 1: Add the presence substitutions**

Add to `substitutions:`:

```yaml
  # Presence set — point these at your HA person.* entities.
  # Note: this repo's bermuda_base.yaml package is a bare BLE proxy with no
  # person entities of its own; if you use Bermuda for presence, point these
  # at whatever entities the Bermuda HA integration created for you.
  orbital_person1_entity: person.person_1
  orbital_person2_entity: person.person_2
  orbital_person3_entity: person.person_3
  orbital_person4_entity: person.person_4
  orbital_person5_entity: person.person_5
```

- [ ] **Step 2: Add the five presence text_sensors**

Insert these entries into the `text_sensor:` block created in Task 4, directly after the `weather_forecast_ts` entry and before the `platform: version` entry:

```yaml
  - platform: homeassistant
    id: presence_person1_ts
    entity_id: ${orbital_person1_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: presence_label_1
          text: !lambda "return x;"
  - platform: homeassistant
    id: presence_person2_ts
    entity_id: ${orbital_person2_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: presence_label_2
          text: !lambda "return x;"
  - platform: homeassistant
    id: presence_person3_ts
    entity_id: ${orbital_person3_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: presence_label_3
          text: !lambda "return x;"
  - platform: homeassistant
    id: presence_person4_ts
    entity_id: ${orbital_person4_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: presence_label_4
          text: !lambda "return x;"
  - platform: homeassistant
    id: presence_person5_ts
    entity_id: ${orbital_person5_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: presence_label_5
          text: !lambda "return x;"
```

- [ ] **Step 3: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add Presence set to Orbital package"
```

---

### Task 6: Custom set

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: LVGL label ids `custom_label_1`..`custom_label_5` (Task 3, currently blank shells); merges into the `text_sensor:` block
- Produces: substitutions `orbital_custom1_entity`..`orbital_custom5_entity` (defaults `sensor.custom_1`..`sensor.custom_5`); text_sensor ids `custom_sensor1_ts`..`custom_sensor5_ts`

- [ ] **Step 1: Add the custom substitutions**

Add to `substitutions:`:

```yaml
  # Custom set — point these at absolutely anything HA already tracks
  # (energy, stocks, whatever). The device just displays the entity's state.
  orbital_custom1_entity: sensor.custom_1
  orbital_custom2_entity: sensor.custom_2
  orbital_custom3_entity: sensor.custom_3
  orbital_custom4_entity: sensor.custom_4
  orbital_custom5_entity: sensor.custom_5
```

- [ ] **Step 2: Add the five custom text_sensors**

Insert into the `text_sensor:` block, after `presence_person5_ts` and before `platform: version`:

```yaml
  - platform: homeassistant
    id: custom_sensor1_ts
    entity_id: ${orbital_custom1_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: custom_label_1
          text: !lambda "return x;"
  - platform: homeassistant
    id: custom_sensor2_ts
    entity_id: ${orbital_custom2_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: custom_label_2
          text: !lambda "return x;"
  - platform: homeassistant
    id: custom_sensor3_ts
    entity_id: ${orbital_custom3_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: custom_label_3
          text: !lambda "return x;"
  - platform: homeassistant
    id: custom_sensor4_ts
    entity_id: ${orbital_custom4_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: custom_label_4
          text: !lambda "return x;"
  - platform: homeassistant
    id: custom_sensor5_ts
    entity_id: ${orbital_custom5_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: custom_label_5
          text: !lambda "return x;"
```

- [ ] **Step 3: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS. All 20 pages (5 displays × 4 sets) and 20 labels now exist with real content sources.

- [ ] **Step 4: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add Custom set to Orbital package"
```

---

### Task 7: Left/Right navigation

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: global `current_set` (Task 3); page ids `clock_1..5`/`weather_1..5`/`presence_1..5`/`custom_1..5` (Tasks 3–6); substitutions `orbital_button_left_pin`, `orbital_button_right_pin` (Task 1)
- Produces: script id `show_current_set`; binary_sensor ids `button_left`, `button_right`

- [ ] **Step 1: Add the shared page-switching script**

Add this after the `text_sensor:` block:

```yaml
# ---------------------------------------------------------------------------
# Navigation: shared script flips all five displays to the active set
# ---------------------------------------------------------------------------

script:
  - id: show_current_set
    mode: restart
    then:
      - if:
          condition:
            lambda: "return id(current_set) == 0;"
          then:
            - lvgl.page.show: clock_1
            - lvgl.page.show: clock_2
            - lvgl.page.show: clock_3
            - lvgl.page.show: clock_4
            - lvgl.page.show: clock_5
      - if:
          condition:
            lambda: "return id(current_set) == 1;"
          then:
            - lvgl.page.show: weather_1
            - lvgl.page.show: weather_2
            - lvgl.page.show: weather_3
            - lvgl.page.show: weather_4
            - lvgl.page.show: weather_5
      - if:
          condition:
            lambda: "return id(current_set) == 2;"
          then:
            - lvgl.page.show: presence_1
            - lvgl.page.show: presence_2
            - lvgl.page.show: presence_3
            - lvgl.page.show: presence_4
            - lvgl.page.show: presence_5
      - if:
          condition:
            lambda: "return id(current_set) == 3;"
          then:
            - lvgl.page.show: custom_1
            - lvgl.page.show: custom_2
            - lvgl.page.show: custom_3
            - lvgl.page.show: custom_4
            - lvgl.page.show: custom_5
```

- [ ] **Step 2: Add the Left and Right buttons**

Add this after the `script:` block:

```yaml
# ---------------------------------------------------------------------------
# Hardware: Buttons (Left/Right navigation)
# ---------------------------------------------------------------------------

binary_sensor:
  - platform: status
    name: "Status"
    entity_category: diagnostic
  - platform: gpio
    id: button_left
    pin:
      number: ${orbital_button_left_pin}
      mode: INPUT_PULLUP
      inverted: true
    internal: true
    on_click:
      min_length: 50ms
      max_length: 500ms
      then:
        - lambda: |-
            id(current_set) = (id(current_set) + 3) % 4;
        - script.execute: show_current_set
  - platform: gpio
    id: button_right
    pin:
      number: ${orbital_button_right_pin}
      mode: INPUT_PULLUP
      inverted: true
    internal: true
    on_click:
      min_length: 50ms
      max_length: 500ms
      then:
        - lambda: |-
            id(current_set) = (id(current_set) + 1) % 4;
        - script.execute: show_current_set
```

Note: this **replaces** the `binary_sensor:` block from Task 1 (repeating the `Status` entry) for the same reason as Task 4's `text_sensor:` merge — one `binary_sensor:` key per file.

- [ ] **Step 3: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add Left/Right navigation buttons to Orbital package"
```

---

### Task 8: OK button (refresh + auto-cycle toggle)

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: script id `show_current_set` (Task 7); global `current_set` (Task 3); substitutions `orbital_button_ok_pin` (Task 1), `orbital_autocycle_interval` (Task 1)
- Produces: global `autocycle_enabled` (bool); binary_sensor id `button_ok`

- [ ] **Step 1: Add the auto-cycle flag**

Modify the `globals:` block from Task 3 to add a second entry:

```yaml
globals:
  - id: current_set
    type: int
    restore_value: false
    initial_value: "0"
  - id: autocycle_enabled
    type: bool
    restore_value: false
    initial_value: "false"
```

- [ ] **Step 2: Add the OK button**

Add this entry to the `binary_sensor:` block from Task 7, after `button_right`:

```yaml
  - platform: gpio
    id: button_ok
    pin:
      number: ${orbital_button_ok_pin}
      mode: INPUT_PULLUP
      inverted: true
    internal: true
    on_click:
      - min_length: 50ms
        max_length: 500ms
        then:
          - script.execute: show_current_set
      - min_length: 500ms
        max_length: 5000ms
        then:
          - lambda: |-
              id(autocycle_enabled) = !id(autocycle_enabled);
```

- [ ] **Step 3: Add the auto-cycle interval**

Add this after the `binary_sensor:` block:

```yaml
# ---------------------------------------------------------------------------
# Auto-cycle: advances the set on a timer while toggled on (long-press OK)
# ---------------------------------------------------------------------------

interval:
  - interval: 1s
    then:
      - lvgl.label.update:
          id: clock_time_1
          text:
            time_format: "%H:%M:%S"
            time: orbital_time
  - interval: 60s
    then:
      - lvgl.label.update:
          id: clock_date_2
          text:
            time_format: "%Y-%m-%d"
            time: orbital_time
      - lvgl.label.update:
          id: clock_day_3
          text:
            time_format: "%A"
            time: orbital_time
  - interval: ${orbital_autocycle_interval}
    then:
      - if:
          condition:
            lambda: "return id(autocycle_enabled);"
          then:
            - lambda: |-
                id(current_set) = (id(current_set) + 1) % 4;
            - script.execute: show_current_set
```

Note: this **replaces** the `interval:` block from Task 3 (repeating the two time-label intervals) for the same one-key-per-file reason as Tasks 4 and 7.

- [ ] **Step 4: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add OK button (refresh + auto-cycle toggle) to Orbital package"
```

---

### Task 9: RGB status light

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: substitution `orbital_led_pin` (Task 1); existing `api:` block (Task 1)
- Produces: light id `status_light`; package-level `wifi:` automation block (merges with the device's own `wifi:` config, which supplies `ssid`/`password`)

- [ ] **Step 1: Add the status light**

Add this block after the diagnostic `text_sensor:`/`button:` sections:

```yaml
# ---------------------------------------------------------------------------
# Hardware: onboard RGB status LED — diagnostic only, not user-controllable
# ---------------------------------------------------------------------------

light:
  - platform: esp32_rmt_led_strip
    id: status_light
    pin: ${orbital_led_pin}
    num_leds: 1
    chipset: ws2812
    rgb_order: GRB
    internal: true
    default_transition_length: 0s
```

- [ ] **Step 2: Wire WiFi connect/disconnect to red/yellow**

Add this block (merges with the device config's own `wifi:` block, which supplies `ssid`/`password`):

```yaml
wifi:
  on_connect:
    - light.turn_on:
        id: status_light
        red: 100%
        green: 80%
        blue: 0%
  on_disconnect:
    - light.turn_on:
        id: status_light
        red: 100%
        green: 0%
        blue: 0%
```

- [ ] **Step 3: Wire HA API connect/disconnect to green/yellow**

Modify the `api:` block from Task 1:

```yaml
api:
  reboot_timeout: 0s
  on_client_connected:
    - light.turn_on:
        id: status_light
        red: 0%
        green: 100%
        blue: 0%
  on_client_disconnected:
    - light.turn_on:
        id: status_light
        red: 100%
        green: 80%
        blue: 0%
```

- [ ] **Step 4: Validate**

Run: `esphome config packages/orbital_base.yaml`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/orbital_base.yaml
git commit -m "Add RGB status light with WiFi/API connection state to Orbital package"
```

---

### Task 10: Example config, CI test, and workflow matrix

**Files:**
- Create: `examples/orbital_example.yaml`
- Create: `tests/test_orbital.yaml`
- Modify: `.github/workflows/validate.yml`

**Interfaces:**
- Consumes: `packages/orbital_base.yaml` (Tasks 1–9) as a whole
- Produces: nothing consumed by later tasks

- [ ] **Step 1: Write the example config**

Create `examples/orbital_example.yaml`, mirroring `examples/wopr_example.yaml`'s shape:

```yaml
# Example configuration for HackerBox #0129 Orbital
# Copy this into your ESPHome device configuration and customize.
#
# Hardware: ESP32-S3 SuperMini + 5x round GC9A01A 1.28" displays (240x240) +
# 3 buttons + onboard addressable RGB LED
# Product: https://hackerboxes.com/products/hackerbox-0129-orbital

substitutions:
  friendly_name: "Orbital"
  # Optional overrides — point these at your own HA entities:
  # orbital_weather_temp_entity: sensor.outdoor_temperature
  # orbital_person1_entity: person.jane
  # orbital_custom1_entity: sensor.grid_power

esphome:
  name: orbital
  friendly_name: "${friendly_name}"

packages:
  device:
    url: https://github.com/cyberglitchlabs/esphome
    files: [packages/orbital_base.yaml]
    ref: main
    refresh: 1d

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  min_auth_mode: WPA2
  ap:
    ssid: "${friendly_name} Fallback"
    password: ""

api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password
```

- [ ] **Step 2: Write the CI test wrapper**

Create `tests/test_orbital.yaml`, mirroring `tests/test_wopr.yaml`'s shape:

```yaml
substitutions:
  name: orbital-test
  friendly_name: "Orbital Test"
  wifi_ssid: test_ssid
  wifi_password: test_password
  orbital_weather_temp_entity: sensor.test_temperature
  orbital_person1_entity: person.test_person

esphome:
  name: ${name}
  friendly_name: "${friendly_name}"

esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  framework:
    type: esp-idf

wifi:
  ssid: ${wifi_ssid}
  password: ${wifi_password}

api:

ota:
  - platform: esphome

packages:
  base: !include ../packages/orbital_base.yaml
```

- [ ] **Step 3: Validate the test file**

Run: `esphome config tests/test_orbital.yaml`
Expected: PASS.

- [ ] **Step 4: Add the test to the CI matrix**

Modify `.github/workflows/validate.yml` — add a line to the `matrix.file` list:

```yaml
        file:
          - tests/test_s31.yaml
          - tests/test_ifan04.yaml
          - tests/test_owon_xdm.yaml
          - tests/test_wopr.yaml
          - tests/test_rv_tv.yaml
          - tests/test_plant_bot.yaml
          - tests/test_bermuda.yaml
          - tests/test_mochi.yaml
          - tests/test_orbital.yaml
```

- [ ] **Step 5: Commit**

```bash
git add examples/orbital_example.yaml tests/test_orbital.yaml .github/workflows/validate.yml
git commit -m "Add Orbital example config, CI test, and workflow matrix entry"
```

---

### Task 11: Documentation

**Files:**
- Create: `docs/orbital.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: the finished `packages/orbital_base.yaml` (Tasks 1–9) — pin table, substitutions, and entity list must match the actual file exactly
- Produces: nothing consumed by later tasks (final task)

- [ ] **Step 1: Write the hardware/usage doc**

Create `docs/orbital.md` with this content. The one bracketed placeholder
(`[CONFIRMED-OR-LOWERED-VALUE]`) must be filled in with whatever `buffer_size`
actually worked from Task 3 Step 5 — replace it, don't leave the brackets in:

```markdown
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
| Left | Previous set (wraps Custom → Clock) |
| Right | Next set (wraps Clock → Weather → Presence → Custom) |
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
`buffer_size: [CONFIRMED-OR-LOWERED-VALUE]`. If you see boot-time out-of-memory
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
```

- [ ] **Step 2: Update the root README device table**

Modify `README.md`'s device table to add:

```markdown
| **HackerBox 0129 Orbital** | [Docs](docs/orbital.md) | `packages/orbital_base.yaml` |
```

- [ ] **Step 3: Cross-check completeness**

Read back `docs/orbital.md` next to the final `packages/orbital_base.yaml` and confirm every substitution defined in the package appears in the docs' substitutions table, and every HA-fed entity substitution states which of the 5 screens it feeds. Fix any gaps found.

- [ ] **Step 4: Commit**

```bash
git add docs/orbital.md README.md
git commit -m "Add Orbital hardware/usage documentation"
```
