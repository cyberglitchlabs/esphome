# Orbital Package Modularization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split `packages/orbital_base.yaml` (1437 lines) into 9 focused files under `packages/`, collapsing the Presence and Custom sets' 5x-duplicated per-display blocks into one templated file each, with zero behavior change other than removing the unused `web_server` component.

**Architecture:** `packages/orbital_base.yaml` becomes a thin composition root (substitutions + core device settings + a `packages:` block). Everything else moves into 8 new files, reassembled via ESPHome's `packages:` deep-merge: dicts merge key-by-key, lists of components merge by declared `id:` (via `!extend`), other lists concatenate. Presence and Custom become `!include`+`vars` templates included 5x each instead of 5 hand-copied blocks.

**Tech Stack:** ESPHome 2026.8.1 YAML config (no application code). Verification is `esphome config` (schema/merge validation) and `esphome compile` (full build + binary size).

**Spec:** `docs/superpowers/specs/2026-08-29-orbital-modularization-design.md`

## Global Constraints

- External consumer contract is unchanged: `examples/orbital_example.yaml` and `tests/test_orbital.yaml` must keep working with `files: [packages/orbital_base.yaml]` and no other changes.
- In `orbital_base.yaml`'s `packages:` mapping, `hardware:` (which declares the base `lvgl:` instances) must appear before `screens_clock:`, `screens_weather:`, `presence_*:`, and `custom_*:` (which each `!extend` those instances) — ESPHome processes `packages:` entries in declaration order.
- Baseline firmware size is 1,337,392 bytes (`firmware.factory.bin` from `esphome compile tests/test_orbital.yaml`). Final size must be equal or smaller — never larger.
- This is a pure refactor: no new features, no behavior change, except removing `web_server:` (confirmed unused).
- Clock and Weather sets are NOT templated — each display shows genuinely different content. Only Presence and Custom use the `!include`+`vars` template pattern.
- `!extend` is only needed for `lvgl:` (the one place two files touch the same id). Every other top-level list (`binary_sensor`, `sensor`, `text_sensor`, `button`, `script`, `interval`) splits across files by plain concatenation — no `!extend` needed there, since no two files declare the same id in those lists.

---

### Task 1: Capture baseline

**Files:**
- Create (scratch, not committed): baseline config/size snapshots

**Interfaces:**
- Produces: baseline `firmware.factory.bin` byte count and a saved `esphome config` dump, used by Task 7's regression diff.

- [ ] **Step 1: Confirm working tree is clean**

```bash
cd /Users/jsievert/personal/esphome && git status --short
```
Expected: no output (only the untracked plan/spec files from prior work, if any remain — otherwise clean).

- [ ] **Step 2: Save the fully-resolved baseline config**

```bash
cd /Users/jsievert/personal/esphome
esphome config packages/orbital_base.yaml > /tmp/orbital_baseline_config.txt
wc -l /tmp/orbital_baseline_config.txt
```
Expected: succeeds, prints a non-zero line count.

- [ ] **Step 3: Rebuild and record the baseline firmware size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: build succeeds; printed size is 1337392 (matches the spec's recorded baseline — if it differs, note the new number and use it as the baseline for Task 7 instead of 1337392).

No commit — this task produces no repo changes.

---

### Task 2: Remove `web_server`

**Files:**
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: nothing new.
- Produces: nothing new (pure deletion).

- [ ] **Step 1: Remove the `web_server:` block**

In `packages/orbital_base.yaml`, find:

```yaml
  - id: autocycle_enabled
    type: bool
    restore_value: false
    initial_value: "false"

web_server:
  port: 80

# ---------------------------------------------------------------------------
# Hardware: Displays (5x round GC9A01A, shared SPI bus)
# ---------------------------------------------------------------------------
```

Replace with:

```yaml
  - id: autocycle_enabled
    type: bool
    restore_value: false
    initial_value: "false"

# ---------------------------------------------------------------------------
# Hardware: Displays (5x round GC9A01A, shared SPI bus)
# ---------------------------------------------------------------------------
```

- [ ] **Step 2: Validate and measure**

```bash
cd /Users/jsievert/personal/esphome
esphome config packages/orbital_base.yaml > /tmp/orbital_after_webserver.txt
diff /tmp/orbital_baseline_config.txt /tmp/orbital_after_webserver.txt
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: `diff` shows only the `web_server` block missing (nothing else). Compile succeeds. Size is less than 1337392.

- [ ] **Step 3: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_base.yaml
git commit -m "Drop unused web_server component from orbital package

Confirmed unused (device is controlled via HA/API, not the local
web UI) — pure flash reclaim ahead of the file split."
```

---

### Task 3: Extract `orbital_fonts.yaml`

**Files:**
- Create: `packages/orbital_fonts.yaml`
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: nothing.
- Produces: `font.weather_icons` (id), used by every screen file's labels.

- [ ] **Step 1: Create `packages/orbital_fonts.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Fonts: Material Design Icons, subset to the weather condition glyphs HA's
# weather integration actually reports (clear-night/cloudy/exceptional/fog/
# hail/lightning/lightning-rainy/partlycloudy/pouring/rainy/snowy/
# snowy-rainy/sunny/windy/windy-variant, plus a sunny-off fallback for an
# unrecognized condition string). Fetched from jsDelivr (pinned to a specific
# @mdi/font release, not @latest) rather than committed locally — font file:
# paths resolve relative to whatever top-level config includes this package,
# so a local path would need duplicating into every consumer's directory.
# Source: https://github.com/Templarian/MaterialDesign-Webfont (SIL OFL 1.1).
# ---------------------------------------------------------------------------

font:
  - file: "https://cdn.jsdelivr.net/npm/@mdi/font@7.4.47/fonts/materialdesignicons-webfont.ttf"
    id: weather_icons
    size: 64
    bpp: 4
    glyphs: [
      "\U000F0594", # clear-night
      "\U000F0590", # cloudy
      "\U000F0F2F", # exceptional
      "\U000F0591", # fog
      "\U000F0592", # hail
      "\U000F0593", # lightning
      "\U000F067E", # lightning-rainy
      "\U000F0595", # partlycloudy
      "\U000F0596", # pouring
      "\U000F0597", # rainy
      "\U000F0598", # snowy
      "\U000F067F", # snowy-rainy
      "\U000F0599", # sunny
      "\U000F059D", # windy
      "\U000F059E", # windy-variant
      "\U000F14E4", # sunny-off (fallback for unknown conditions)
      # Measurement icons for the weather screens that show a number
      "\U000F050F", # thermometer  -> temperature
      "\U000F058E", # water-percent -> humidity
      "\U000F0E17", # calendar-month -> forecast
      # Clock-set icons
      "\U000F0A79", # trash-can     -> waste pickup
      "\U000F044C", # recycle       -> recycling included this week
      "\U000F059C", # weather-sunset-up   -> next event is sunrise
      "\U000F059B", # weather-sunset-down -> next event is sunset
      # Presence state icons
      "\U000F02DC", # home        -> person is home
      "\U000F0583", # walk        -> person is away
      "\U000F02D7", # help-circle -> unknown / unavailable
      # General-purpose palette for the custom slots. This list is static on
      # purpose: ESPHome rejects a font whose glyph list contains duplicates,
      # so interpolating the five orbital_custom*_icon substitutions here
      # would break as soon as two slots picked the same icon (two power
      # readings both wanting mdi:flash, say). Baking a fixed palette instead
      # means a substitution only selects a glyph that is already present, and
      # slots may freely repeat. To use an icon that is not listed, add its
      # codepoint here as well as setting the substitution.
      "\U000F029A", # gauge
      "\U000F0241", # flash
      "\U000F140B", # lightning-bolt
      "\U000F01C1", # currency-usd
      "\U000F0079", # battery
      "\U000F05A9", # wifi
      "\U000F012A", # chart-line
      "\U000F0150", # clock-outline
      "\U000F032A", # leaf
      "\U000F010B", # car
      "\U000F0F54", # home-thermometer
      "\U000F058C", # water
      "\U000F042B", # printer-3d
      "\U000F048D", # server-network
      "\U000F009A", # bell
      "\U000F03D6", # package-variant
      ]
```

- [ ] **Step 2: Remove the font block from `orbital_base.yaml` and add the `packages:` block**

Find:

```yaml
  - id: autocycle_enabled
    type: bool
    restore_value: false
    initial_value: "false"

# ---------------------------------------------------------------------------
# Hardware: Displays (5x round GC9A01A, shared SPI bus)
# ---------------------------------------------------------------------------
```

Replace with:

```yaml
  - id: autocycle_enabled
    type: bool
    restore_value: false
    initial_value: "false"

packages:
  fonts: !include orbital_fonts.yaml

# ---------------------------------------------------------------------------
# Hardware: Displays (5x round GC9A01A, shared SPI bus)
# ---------------------------------------------------------------------------
```

Then find (later in the same file) the font block itself:

```yaml
# ---------------------------------------------------------------------------
# Fonts: Material Design Icons, subset to the weather condition glyphs HA's
# weather integration actually reports (clear-night/cloudy/exceptional/fog/
# hail/lightning/lightning-rainy/partlycloudy/pouring/rainy/snowy/
# snowy-rainy/sunny/windy/windy-variant, plus a sunny-off fallback for an
# unrecognized condition string). Fetched from jsDelivr (pinned to a specific
# @mdi/font release, not @latest) rather than committed locally — font file:
# paths resolve relative to whatever top-level config includes this package,
# so a local path would need duplicating into every consumer's directory.
# Source: https://github.com/Templarian/MaterialDesign-Webfont (SIL OFL 1.1).
# ---------------------------------------------------------------------------

font:
  - file: "https://cdn.jsdelivr.net/npm/@mdi/font@7.4.47/fonts/materialdesignicons-webfont.ttf"
    id: weather_icons
    size: 64
    bpp: 4
    glyphs: [
```

... through the closing `]` of the glyph list, up to and including the blank line before:

```yaml
# ---------------------------------------------------------------------------
# LVGL: one instance per display, four pages each (Clock/Weather/Presence/Custom)
# ---------------------------------------------------------------------------
```

Delete that entire font block (comment header through the glyph list's closing `]` and its trailing blank line), leaving the `# LVGL:` comment directly following the end of the `display:` block.

- [ ] **Step 3: Validate**

```bash
cd /Users/jsievert/personal/esphome
esphome config packages/orbital_base.yaml > /tmp/orbital_after_fonts.txt
diff /tmp/orbital_after_webserver.txt /tmp/orbital_after_fonts.txt
```
Expected: no diff (purely structural move, resolved config identical).

- [ ] **Step 4: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_base.yaml packages/orbital_fonts.yaml
git commit -m "Extract font definitions into orbital_fonts.yaml"
```

---

### Task 4: Extract `orbital_diagnostics.yaml`

**Files:**
- Create: `packages/orbital_diagnostics.yaml`
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing consumed elsewhere (diagnostic entities are leaves).

- [ ] **Step 1: Create `packages/orbital_diagnostics.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Diagnostic entities: status, WiFi signal, uptime, restart/safe-mode
# buttons, firmware version, WiFi info. Not wired to any screen.
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
    # No longer drives a screen -- kept as an HA diagnostic entity, which is
    # where signal strength is actually useful.
  - platform: uptime
    id: uptime_sensor
    name: "Uptime"
    update_interval: 60s
    entity_category: diagnostic
    # No longer drives a screen -- kept as an HA diagnostic entity.

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

- [ ] **Step 2: Add the `diagnostics:` package entry**

Find:

```yaml
packages:
  fonts: !include orbital_fonts.yaml
```

Replace with:

```yaml
packages:
  fonts: !include orbital_fonts.yaml
  diagnostics: !include orbital_diagnostics.yaml
```

- [ ] **Step 3: Remove the diagnostic blocks from `orbital_base.yaml`**

Find the `binary_sensor:` block:

```yaml
binary_sensor:
  - platform: status
    name: "Status"
    entity_category: diagnostic
  - platform: gpio
    id: button_left
```

Replace with (dropping just the `status` platform entry):

```yaml
binary_sensor:
  - platform: gpio
    id: button_left
```

Find the `sensor:` block's diagnostic entries:

```yaml
sensor:
  - platform: wifi_signal
    id: wifi_signal_sensor
    name: "WiFi Signal"
    update_interval: 60s
    entity_category: diagnostic
    # No longer drives a screen -- kept as an HA diagnostic entity, which is
    # where signal strength is actually useful.
  - platform: uptime
    id: uptime_sensor
    name: "Uptime"
    update_interval: 60s
    entity_category: diagnostic
    # No longer drives a screen -- kept as an HA diagnostic entity.

  - platform: homeassistant
    id: indoor_temp_num
```

Replace with:

```yaml
sensor:
  - platform: homeassistant
    id: indoor_temp_num
```

Find the `button:` block:

```yaml
button:
  - platform: restart
    name: "Restart"
    entity_category: config
  - platform: safe_mode
    name: "Safe Mode"
    entity_category: config

# ---------------------------------------------------------------------------
# Home Assistant data: Weather / Presence / Custom sets
# ---------------------------------------------------------------------------
```

Replace with:

```yaml
# ---------------------------------------------------------------------------
# Home Assistant data: Weather / Presence / Custom sets
# ---------------------------------------------------------------------------
```

Find the trailing `text_sensor:` diagnostic entries (near the end of the `text_sensor:` list, right before the `light:` section):

```yaml
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

# ---------------------------------------------------------------------------
# Hardware: onboard RGB status LED — diagnostic only, not user-controllable
# ---------------------------------------------------------------------------
```

Replace with:

```yaml
# ---------------------------------------------------------------------------
# Hardware: onboard RGB status LED — diagnostic only, not user-controllable
# ---------------------------------------------------------------------------
```

- [ ] **Step 4: Validate**

```bash
cd /Users/jsievert/personal/esphome
esphome config packages/orbital_base.yaml > /tmp/orbital_after_diagnostics.txt
diff /tmp/orbital_after_fonts.txt /tmp/orbital_after_diagnostics.txt
```
Expected: no diff.

- [ ] **Step 5: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_base.yaml packages/orbital_diagnostics.yaml
git commit -m "Extract diagnostic entities into orbital_diagnostics.yaml"
```

---

### Task 5: Rebuild display content — `orbital_hardware.yaml`, `orbital_screens_clock.yaml`, `orbital_screens_weather.yaml`, `orbital_screen_presence.yaml`, `orbital_screen_custom.yaml`

This is one task because these five files are not independently valid: `hardware.yaml`'s `lvgl:` base has no pages until the four screen files `!extend` it, and none of the four screen files can exist without `hardware.yaml`'s base already declared. They must land together.

**Files:**
- Create: `packages/orbital_hardware.yaml`
- Create: `packages/orbital_screens_clock.yaml`
- Create: `packages/orbital_screens_weather.yaml`
- Create: `packages/orbital_screen_presence.yaml`
- Create: `packages/orbital_screen_custom.yaml`
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: `font.weather_icons` (Task 3), substitutions `orbital_person1..5_entity/_label`, `orbital_custom1..5_entity/_label/_icon/_unit` (root, unchanged).
- Produces: `lvgl` instances `lvgl_1..5` each with all 4 pages (`clock_N`, `weather_N`, `presence_N`, `custom_N`); ids `minute_hand`, `hour_hand`, `sun_time_4`, `sun_label_4`, `sun_icon_4`, `waste_recycle_3`, `waste_when_3` used by scripts still living in root at this point (`update_sun_screen`, `update_waste_screen` move into this task too — see Step 3); page ids `clock_1..5`/`weather_1..5`/`presence_1..5`/`custom_1..5` consumed by `show_current_set` (still in root, moves in Task 6).

- [ ] **Step 1: Create `packages/orbital_hardware.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Hardware: Displays (5x round GC9A01A, shared SPI bus)
# ---------------------------------------------------------------------------

spi:
  id: spi_bus
  clk_pin: ${orbital_sclk_pin}
  mosi_pin: ${orbital_mosi_pin}

display:
  - platform: mipi_spi
    model: GC9A01A
    id: display_1
    spi_id: spi_bus
    cs_pin: ${orbital_cs1_pin}
    dc_pin:
      number: ${orbital_dc_pin}
      allow_other_uses: true
    reset_pin: ${orbital_reset_pin}
    invert_colors: true
    auto_clear_enabled: false
    update_interval: never  # LVGL drives redraws; no polling needed
  - platform: mipi_spi
    model: GC9A01A
    id: display_2
    spi_id: spi_bus
    cs_pin: ${orbital_cs2_pin}
    dc_pin:
      number: ${orbital_dc_pin}
      allow_other_uses: true
    invert_colors: true
    auto_clear_enabled: false
    update_interval: never
  - platform: mipi_spi
    model: GC9A01A
    id: display_3
    spi_id: spi_bus
    cs_pin: ${orbital_cs3_pin}
    dc_pin:
      number: ${orbital_dc_pin}
      allow_other_uses: true
    invert_colors: true
    auto_clear_enabled: false
    update_interval: never
  - platform: mipi_spi
    model: GC9A01A
    id: display_4
    spi_id: spi_bus
    cs_pin: ${orbital_cs4_pin}
    dc_pin:
      number: ${orbital_dc_pin}
      allow_other_uses: true
    invert_colors: true
    auto_clear_enabled: false
    update_interval: never
  - platform: mipi_spi
    model: GC9A01A
    id: display_5
    spi_id: spi_bus
    cs_pin: ${orbital_cs5_pin}
    dc_pin:
      number: ${orbital_dc_pin}
      allow_other_uses: true
    invert_colors: true
    auto_clear_enabled: false
    update_interval: never

# ---------------------------------------------------------------------------
# LVGL: one instance per display. Base declaration only -- no pages: key
# here. orbital_screens_clock.yaml, orbital_screens_weather.yaml,
# orbital_screen_presence.yaml, and orbital_screen_custom.yaml each
# `!extend` these instances to add their one page apiece. Those files'
# entries in orbital_base.yaml's `packages:` block must come after this
# file's `hardware` entry -- ESPHome processes packages in declaration
# order, and `!extend` needs something to extend.
# ---------------------------------------------------------------------------

lvgl:
  - id: lvgl_1
    displays:
      - display_1
    buffer_size: 10%
    default_font: montserrat_28
    # dark_mode applies globally across all 5 LVGL instances even though it's
    # only declared here — ESPHome only allows `theme:` on one instance when
    # multiple LVGL instances are in play (it's one shared define, not
    # per-instance), so don't copy this block onto lvgl_2..lvgl_5.
    theme:
      dark_mode: true
  - id: lvgl_2
    displays:
      - display_2
    buffer_size: 10%
    default_font: montserrat_28
  - id: lvgl_3
    displays:
      - display_3
    buffer_size: 10%
    default_font: montserrat_28
  - id: lvgl_4
    displays:
      - display_4
    buffer_size: 10%
    default_font: montserrat_28
  - id: lvgl_5
    displays:
      - display_5
    buffer_size: 10%
    default_font: montserrat_28

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

- [ ] **Step 2: Create `packages/orbital_screens_clock.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Clock set: analog clock (display 1), calendar (display 2), waste pickup
# (display 3), sunrise/sunset (display 4), indoor temperature (display 5).
# Each display shows genuinely different content here, so unlike Presence
# and Custom this set is not a template -- five distinct pages.
# ---------------------------------------------------------------------------

lvgl:
  - id: !extend lvgl_1
    pages:
      - id: clock_1
        widgets:
          - obj:
              width: 240
              height: 240
              align: CENTER
              pad_all: 0
              border_width: 0
              bg_color: 0x000000
              widgets:
                - meter:
                    id: clock_meter_1
                    height: 220
                    width: 220
                    align: CENTER
                    bg_opa: TRANSP
                    border_width: 0
                    scales:
                      - range_from: 0 # minutes scale (outer ring of ticks)
                        range_to: 60
                        angle_range: 360
                        rotation: 270
                        ticks:
                          width: 1
                          count: 61
                          length: 6
                          color: 0x444444
                        indicators:
                          - line:
                              id: minute_hand
                              width: 3
                              color: 0x00D9FF
                              length: 90%
                              value: 0
                      - range_from: 1 # hour scale (numbered 1-12 major ticks)
                        range_to: 12
                        angle_range: 330
                        rotation: 300
                        ticks:
                          width: 1
                          count: 12
                          length: 1
                          major:
                            stride: 1
                            width: 3
                            length: 14
                            color: 0xAAAAAA
                            label_gap: 10
                      - range_from: 0 # hi-res hours scale, drives the hour hand
                        range_to: 720
                        angle_range: 360
                        rotation: 270
                        indicators:
                          - line:
                              id: hour_hand
                              width: 5
                              color: 0xFF9500
                              length: 65%
                              value: 0

  - id: !extend lvgl_2
    pages:
      - id: clock_2
        widgets:
          # Month and weekday are content here, not captions, so they use
          # montserrat_28 rather than the montserrat_14 used for captions
          # elsewhere. 28 is already compiled in as default_font, so this
          # costs no flash -- a new size would have.
          #
          # Vertical budget on the 240px circle: month spans -70..-42, the
          # day number -32..+16, weekday +30..+58. The widest weekday,
          # WEDNESDAY, is ~144px at this size; the circle is 210px wide at
          # y=+58, so it clears the rim.
          - label:
              id: clock_month_2
              align: CENTER
              y: -56
              text_font: montserrat_28
              text_color: 0xB6BEC9
              text: "---"
          - label:
              id: clock_date_2
              align: CENTER
              y: -8
              text_font: montserrat_48
              text: "00"
          - label:
              id: clock_dow_2
              align: CENTER
              y: 44
              text_font: montserrat_28
              text_color: 0xB6BEC9
              text: "---"

  - id: !extend lvgl_3
    pages:
      - id: clock_3
        widgets:
          # Both icons are always present in the same place; only the recycle
          # symbol's colour changes. Position never moving is what makes it
          # readable at a glance instead of something you have to parse.
          - label:
              id: waste_trash_3
              align: CENTER
              x: -34
              y: -30
              text_font: weather_icons
              text_color: 0xC9D1D9
              text: "\U000F0A79"
          - label:
              id: waste_recycle_3
              align: CENTER
              x: 34
              y: -30
              text_font: weather_icons
              text_color: 0x22272E
              text: "\U000F044C"
          - label:
              id: waste_when_3
              align: CENTER
              y: 38
              text: "-"

  - id: !extend lvgl_4
    pages:
      - id: clock_4
        widgets:
          - label:
              id: sun_icon_4
              align: CENTER
              y: -44
              text_font: weather_icons
              text_color: 0xFFC64B
              text: "\U000F059C"
          - label:
              id: sun_time_4
              align: CENTER
              y: 4
              text: "--:--"
          - label:
              id: sun_label_4
              align: CENTER
              y: 38
              text_font: montserrat_14
              text_color: 0x6B7380
              text: "---"

  - id: !extend lvgl_5
    pages:
      - id: clock_5
        widgets:
          - label:
              id: indoor_icon_5
              align: CENTER
              y: -44
              text_font: weather_icons
              text_color: 0xFF7A33
              text: "\U000F050F"
          - label:
              id: indoor_label_5
              align: CENTER
              y: 4
              text: "-"
          - label:
              id: indoor_name_5
              align: CENTER
              y: 38
              text_font: montserrat_14
              text_color: 0x6B7380
              text: "INDOOR"

interval:
  - interval: 1s
    then:
      - lvgl.indicator.update:
          id: minute_hand
          value: !lambda "return id(orbital_time).now().minute;"
      - lvgl.indicator.update:
          id: hour_hand
          value: !lambda |-
            auto t = id(orbital_time).now();
            return (t.hour % 12) * 60 + t.minute;
  - interval: 60s
    then:
      # Calendar page. strftime's day-of-month is zero-padded (%d) or
      # space-padded (%e) and the no-pad form is a GNU extension, so the
      # number comes from the field directly instead.
      - lvgl.label.update:
          id: clock_date_2
          text: !lambda |-
            char buf[4];
            snprintf(buf, sizeof(buf), "%d", id(orbital_time).now().day_of_month);
            return std::string(buf);
      - lvgl.label.update:
          id: clock_month_2
          text: !lambda |-
            std::string s = id(orbital_time).now().strftime("%b");
            for (auto &c : s) c = toupper(c);
            return s;
      - lvgl.label.update:
          id: clock_dow_2
          text: !lambda |-
            std::string s = id(orbital_time).now().strftime("%A");
            for (auto &c : s) c = toupper(c);
            return s;
      # Sunrise/sunset: show whichever comes next.
      - script.execute: update_sun_screen

sensor:
  - platform: homeassistant
    id: indoor_temp_num
    entity_id: ${orbital_indoor_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: indoor_label_5
          text:
            format: "%.1f${orbital_indoor_unit}"
            args: [x]

text_sensor:
  # Clock screen 3: the waste calendar's next event. `description` carries
  # the combined text for the whole pickup; `message` would give only the
  # first of several same-day events.
  - platform: homeassistant
    id: waste_desc_ts
    entity_id: ${orbital_waste_calendar_entity}
    attribute: description
    internal: true
    on_value:
      - script.execute: update_waste_screen
  - platform: homeassistant
    id: waste_start_ts
    entity_id: ${orbital_waste_calendar_entity}
    attribute: start_time
    internal: true
    on_value:
      - script.execute: update_waste_screen

script:
  - id: update_sun_screen
    mode: restart
    then:
      - lambda: |-
          // -0.833 deg is the standard refraction-corrected horizon.
          auto sunrise = id(orbital_sun).sunrise(-0.833);
          auto sunset = id(orbital_sun).sunset(-0.833);
          const bool day = id(orbital_sun).is_above_horizon();
          auto next = day ? sunset : sunrise;
          if (!next.has_value())
            return;   // no fix yet (clock not synced, or polar day/night)
          lv_label_set_text(id(sun_time_4), next->strftime("%H:%M").c_str());
          lv_label_set_text(id(sun_label_4), day ? "SUNSET" : "SUNRISE");
          lv_label_set_text(id(sun_icon_4),
                            day ? "\U000F059B" : "\U000F059C");

  - id: update_waste_screen
    mode: restart
    then:
      - lambda: |-
          // Recycling is the only stream that varies week to week here -
          // garbage and yard waste ride along on every pickup - so the
          // recycle glyph lights up and everything else stays put.
          std::string d = id(waste_desc_ts).state;
          for (auto &c : d) c = tolower(c);
          const bool recycling = d.find("recycl") != std::string::npos;
          lv_obj_set_style_text_color(
              id(waste_recycle_3),
              recycling ? lv_color_make(52, 199, 89) : lv_color_make(34, 39, 53),
              LV_PART_MAIN);

          // "2026-08-25 00:00:00" -> TODAY / TOMORROW / a weekday.
          ESPTime pickup{};
          if (!ESPTime::strptime(id(waste_start_ts).state, pickup)) {
            lv_label_set_text(id(waste_when_3), "-");
            return;
          }
          // strptime fills the calendar fields but not day_of_week, so round
          // trip through the epoch to get a fully populated struct.
          pickup.recalc_timestamp_utc(false);
          pickup = ESPTime::from_epoch_utc(pickup.timestamp);

          auto now = id(orbital_time).now();
          auto tomorrow = now;
          tomorrow.increment_day();

          const char *when;
          if (pickup.year == now.year && pickup.month == now.month &&
              pickup.day_of_month == now.day_of_month) {
            when = "TODAY";
          } else if (pickup.year == tomorrow.year &&
                     pickup.month == tomorrow.month &&
                     pickup.day_of_month == tomorrow.day_of_month) {
            when = "TOMORROW";
          } else {
            static const char *const DOW[] = {"SUN", "MON", "TUE", "WED",
                                              "THU", "FRI", "SAT"};
            const int idx = pickup.day_of_week - 1;   // sunday == 1
            when = (idx >= 0 && idx < 7) ? DOW[idx] : "-";
          }
          lv_label_set_text(id(waste_when_3), when);
```

- [ ] **Step 3: Create `packages/orbital_screens_weather.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Weather set: screens 1/2/3/4 all read from a SINGLE HA weather entity
# (temperature, humidity, wind_speed off its attributes; condition off its
# plain state). Screen 5 is a precomputed forecast string.
# ---------------------------------------------------------------------------

lvgl:
  - id: !extend lvgl_1
    pages:
      - id: weather_1
        widgets:
          - label:
              id: weather_icon_1
              align: CENTER
              y: -34
              text_font: weather_icons
              text_color: 0xFF7A33
              text: "\U000F050F"
          - label:
              id: weather_label_1
              align: CENTER
              y: 18
              text: "-"

  - id: !extend lvgl_2
    pages:
      - id: weather_2
        widgets:
          - label:
              id: weather_icon_2
              align: CENTER
              y: -34
              text_font: weather_icons
              text_color: 0x00B4D8
              text: "\U000F058E"
          - label:
              id: weather_label_2
              align: CENTER
              y: 18
              text: "-"

  - id: !extend lvgl_3
    pages:
      - id: weather_3
        widgets:
          - label:
              id: weather_label_3
              align: CENTER
              text_font: weather_icons
              text: ""

  - id: !extend lvgl_4
    pages:
      - id: weather_4
        widgets:
          - label:
              id: weather_icon_4
              align: CENTER
              y: -34
              text_font: weather_icons
              text_color: 0x9AA5B1
              text: "\U000F059D"
          - label:
              id: weather_label_4
              align: CENTER
              y: 18
              text: "-"

  - id: !extend lvgl_5
    pages:
      - id: weather_5
        widgets:
          - label:
              id: weather_icon_5
              align: CENTER
              y: -34
              text_font: weather_icons
              text_color: 0x8B7BE8
              text: "\U000F0E17"
          - label:
              id: weather_label_5
              align: CENTER
              y: 18
              text: "-"

# Weather numerics, read as attributes off the single weather entity. Using
# the numeric platform (rather than text_sensor) means x is a float, so the
# value can be rounded for display — HA's raw state string would give us
# "8.780000000000001" for wind speed.
sensor:
  - platform: homeassistant
    id: weather_temp_num
    entity_id: ${orbital_weather_entity}
    attribute: temperature
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_1
          text:
            format: "%.0f${orbital_weather_temp_unit}"
            args: [x]
  - platform: homeassistant
    id: weather_humidity_num
    entity_id: ${orbital_weather_entity}
    attribute: humidity
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_2
          text:
            format: "%.0f%%"
            args: [x]
  - platform: homeassistant
    id: weather_wind_num
    entity_id: ${orbital_weather_entity}
    attribute: wind_speed
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_4
          text:
            format: "%.0f${orbital_weather_wind_unit}"
            args: [x]

text_sensor:
  # Screen 3: the weather entity's own state, mapped to a condition glyph.
  - platform: homeassistant
    id: weather_condition_ts
    entity_id: ${orbital_weather_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_3
          text: !lambda |-
            if (x == "clear-night") return std::string("\U000F0594");
            if (x == "cloudy") return std::string("\U000F0590");
            if (x == "fog") return std::string("\U000F0591");
            if (x == "hail") return std::string("\U000F0592");
            if (x == "lightning") return std::string("\U000F0593");
            if (x == "lightning-rainy") return std::string("\U000F067E");
            if (x == "partlycloudy") return std::string("\U000F0595");
            if (x == "pouring") return std::string("\U000F0596");
            if (x == "rainy") return std::string("\U000F0597");
            if (x == "snowy") return std::string("\U000F0598");
            if (x == "snowy-rainy") return std::string("\U000F067F");
            if (x == "sunny") return std::string("\U000F0599");
            if (x == "windy") return std::string("\U000F059D");
            if (x == "windy-variant") return std::string("\U000F059E");
            if (x == "exceptional") return std::string("\U000F0F2F");
            return std::string("\U000F14E4"); // unrecognized condition

  # Screen 5: a short forecast string precomputed in HA. Truncated because the
  # round display fits roughly ten characters at montserrat_28.
  - platform: homeassistant
    id: weather_forecast_ts
    entity_id: ${orbital_weather_forecast_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: weather_label_5
          text: !lambda "return x.substr(0, 10);"
```

- [ ] **Step 4: Create `packages/orbital_screen_presence.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Presence set: template, included once per display from orbital_base.yaml's
# packages: block with vars: {index, entity, label}. Replaces what used to
# be 5 hand-copied ~29-line blocks (one per person).
# ---------------------------------------------------------------------------

lvgl:
  - id: !extend lvgl_${index}
    pages:
      - id: presence_${index}
        widgets:
          - label:
              id: presence_icon_${index}
              align: CENTER
              y: -44
              text_font: weather_icons
              text_color: 0x5A6472
              text: "\U000F02D7"
          - label:
              id: presence_label_${index}
              align: CENTER
              y: 4
              text: "-"
          - label:
              id: presence_name_${index}
              align: CENTER
              y: 38
              text_font: montserrat_14
              text_color: 0x6B7380
              text: "${label}"

text_sensor:
  - platform: homeassistant
    id: presence_person${index}_ts
    entity_id: ${entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: presence_label_${index}
          text: !lambda |-
            if (x == "home") return std::string("Home");
            if (x == "not_home") return std::string("Away");
            return x.substr(0, 10);
      - lvgl.label.update:
          id: presence_icon_${index}
          text: !lambda |-
            if (x == "home") return std::string("\U000F02DC");
            if (x == "not_home") return std::string("\U000F0583");
            return std::string("\U000F02D7");
          text_color: !lambda |-
            if (x == "home") return lv_color_make(52, 199, 89);
            if (x == "not_home") return lv_color_make(255, 149, 0);
            return lv_color_make(90, 100, 114);
```

- [ ] **Step 5: Create `packages/orbital_screen_custom.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Custom set: template, included once per display from orbital_base.yaml's
# packages: block with vars: {index, entity, label, icon, unit}. Replaces
# what used to be 5 hand-copied ~29-line blocks.
# ---------------------------------------------------------------------------

lvgl:
  - id: !extend lvgl_${index}
    pages:
      - id: custom_${index}
        widgets:
          - label:
              id: custom_icon_${index}
              align: CENTER
              y: -44
              text_font: weather_icons
              text_color: 0x00D9FF
              text: "${icon}"
          - label:
              id: custom_label_${index}
              align: CENTER
              y: 4
              text: "-"
          - label:
              id: custom_name_${index}
              align: CENTER
              y: 38
              text_font: montserrat_14
              text_color: 0x6B7380
              text: "${label}"

text_sensor:
  - platform: homeassistant
    id: custom_sensor${index}_ts
    entity_id: ${entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: custom_label_${index}
          text: !lambda |-
            char *endptr;
            float val = strtof(x.c_str(), &endptr);
            if (endptr != x.c_str() && *endptr == '\0') {
              char buf[24];
              snprintf(buf, sizeof(buf), "%.1f${unit}", val);
              return std::string(buf);
            }
            return x.substr(0, 10);
```

- [ ] **Step 6: Add the 11 new `packages:` entries in `orbital_base.yaml`**

Find:

```yaml
packages:
  fonts: !include orbital_fonts.yaml
  diagnostics: !include orbital_diagnostics.yaml
```

Replace with:

```yaml
packages:
  hardware: !include orbital_hardware.yaml
  fonts: !include orbital_fonts.yaml
  diagnostics: !include orbital_diagnostics.yaml
  screens_clock: !include orbital_screens_clock.yaml
  screens_weather: !include orbital_screens_weather.yaml
  presence_1: !include
    file: orbital_screen_presence.yaml
    vars:
      index: "1"
      entity: "${orbital_person1_entity}"
      label: "${orbital_person1_label}"
  presence_2: !include
    file: orbital_screen_presence.yaml
    vars:
      index: "2"
      entity: "${orbital_person2_entity}"
      label: "${orbital_person2_label}"
  presence_3: !include
    file: orbital_screen_presence.yaml
    vars:
      index: "3"
      entity: "${orbital_person3_entity}"
      label: "${orbital_person3_label}"
  presence_4: !include
    file: orbital_screen_presence.yaml
    vars:
      index: "4"
      entity: "${orbital_person4_entity}"
      label: "${orbital_person4_label}"
  presence_5: !include
    file: orbital_screen_presence.yaml
    vars:
      index: "5"
      entity: "${orbital_person5_entity}"
      label: "${orbital_person5_label}"
  custom_1: !include
    file: orbital_screen_custom.yaml
    vars:
      index: "1"
      entity: "${orbital_custom1_entity}"
      label: "${orbital_custom1_label}"
      icon: "${orbital_custom1_icon}"
      unit: "${orbital_custom1_unit}"
  custom_2: !include
    file: orbital_screen_custom.yaml
    vars:
      index: "2"
      entity: "${orbital_custom2_entity}"
      label: "${orbital_custom2_label}"
      icon: "${orbital_custom2_icon}"
      unit: "${orbital_custom2_unit}"
  custom_3: !include
    file: orbital_screen_custom.yaml
    vars:
      index: "3"
      entity: "${orbital_custom3_entity}"
      label: "${orbital_custom3_label}"
      icon: "${orbital_custom3_icon}"
      unit: "${orbital_custom3_unit}"
  custom_4: !include
    file: orbital_screen_custom.yaml
    vars:
      index: "4"
      entity: "${orbital_custom4_entity}"
      label: "${orbital_custom4_label}"
      icon: "${orbital_custom4_icon}"
      unit: "${orbital_custom4_unit}"
  custom_5: !include
    file: orbital_screen_custom.yaml
    vars:
      index: "5"
      entity: "${orbital_custom5_entity}"
      label: "${orbital_custom5_label}"
      icon: "${orbital_custom5_icon}"
      unit: "${orbital_custom5_unit}"
```

- [ ] **Step 7: Remove the now-duplicated content from `orbital_base.yaml`**

Remove the `spi:` block, the 5 `display:` entries, and their section comment (everything from `# --- Hardware: Displays ---` through the end of `display_5`'s block) — this content now lives in `orbital_hardware.yaml`.

Remove the entire `lvgl:` block (all 5 instances, all 20 pages — `# --- LVGL: ... ---` comment through the end of `custom_5`'s page) — this content is now split across `orbital_hardware.yaml` (base) and the 4 screen files (pages).

Remove the `interval:` block's first two entries (the `1s` and `60s` ones) — keep the third (`${orbital_autocycle_interval}`) entry and its own `interval:` key, since that one moves in Task 6. The `interval:` block should read, after this step:

```yaml
interval:
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

Remove the `sensor:` block entirely (it now only contains `indoor_temp_num`, `weather_temp_num`, `weather_humidity_num`, `weather_wind_num`, all moved out) — delete the `sensor:` top-level key.

Remove the `text_sensor:` block's `weather_condition_ts`, `waste_desc_ts`/`waste_start_ts`, `weather_forecast_ts`, `presence_person1..5_ts`, and `custom_sensor1..5_ts` entries — this empties `text_sensor:` entirely (diagnostics' entries already left in Task 4). Delete the `text_sensor:` top-level key.

Remove the `light:` block (status LED) — moved into `orbital_hardware.yaml`.

Remove the `script:` block's `update_sun_screen` and `update_waste_screen` entries — keep `show_current_set` (moves in Task 6). The `script:` block should read, after this step:

```yaml
script:
  - id: show_current_set
    mode: restart
    then:
      - if:
          condition:
            lambda: "return id(current_set) == 0;"
          then:
            - lvgl.page.show:
                id: clock_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: clock_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: clock_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: clock_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: clock_5
                lvgl_id: lvgl_5
      - if:
          condition:
            lambda: "return id(current_set) == 1;"
          then:
            - lvgl.page.show:
                id: weather_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: weather_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: weather_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: weather_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: weather_5
                lvgl_id: lvgl_5
      - if:
          condition:
            lambda: "return id(current_set) == 2;"
          then:
            - lvgl.page.show:
                id: presence_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: presence_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: presence_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: presence_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: presence_5
                lvgl_id: lvgl_5
      - if:
          condition:
            lambda: "return id(current_set) == 3;"
          then:
            - lvgl.page.show:
                id: custom_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: custom_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: custom_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: custom_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: custom_5
                lvgl_id: lvgl_5
```

(This is unchanged from the original file — call it out explicitly since it's easy to mis-delete while removing the two scripts above it.)

- [ ] **Step 8: Validate**

```bash
cd /Users/jsievert/personal/esphome
esphome config packages/orbital_base.yaml > /tmp/orbital_after_screens.txt
diff /tmp/orbital_after_diagnostics.txt /tmp/orbital_after_screens.txt
```
Expected: no diff — this is the step that proves `!extend` and the presence/custom templates reproduced the original merged config exactly. If there IS a diff, do not proceed: compare the two lvgl entries side by side (likely culprits: `!extend` base ordering, or a missing `pages:` on `lvgl_1..5` in `orbital_hardware.yaml` not accepting a later `!extend` — see the spec's documented fallback of having `orbital_screens_clock.yaml` establish the base page instead of `orbital_hardware.yaml` declaring none).

- [ ] **Step 9: Compile and check size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: succeeds; size still less than the original 1337392 baseline (should match Task 2's post-web_server size closely, since this task is a pure reorganization).

- [ ] **Step 10: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_hardware.yaml packages/orbital_screens_clock.yaml \
        packages/orbital_screens_weather.yaml packages/orbital_screen_presence.yaml \
        packages/orbital_screen_custom.yaml packages/orbital_base.yaml
git commit -m "Split display/screen content into per-concern package files

Presence and Custom sets now use an !include+vars template (one file,
included 5x) instead of 5 hand-copied blocks each. Clock and Weather
stay as explicit per-display pages since each display shows different
content in those sets."
```

---

### Task 6: Extract `orbital_navigation.yaml`

By this point every page id (`clock_1..5`, `weather_1..5`, `presence_1..5`, `custom_1..5`) exists again in its final location, so `show_current_set` can move safely.

**Files:**
- Create: `packages/orbital_navigation.yaml`
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: page ids from Task 5's files; `orbital_button_left_pin`/`orbital_button_ok_pin`/`orbital_button_right_pin`/`orbital_autocycle_interval` substitutions (root, unchanged).
- Produces: nothing consumed elsewhere.

- [ ] **Step 1: Create `packages/orbital_navigation.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Navigation: state (current set, autocycle), the 3 hardware buttons, and
# the script that flips all five displays to the active set.
# ---------------------------------------------------------------------------

globals:
  - id: current_set
    type: int
    restore_value: false
    initial_value: "0"
  - id: autocycle_enabled
    type: bool
    restore_value: false
    initial_value: "false"

binary_sensor:
  - platform: gpio
    id: button_left
    pin:
      number: ${orbital_button_left_pin}
      mode: INPUT_PULLDOWN
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
      mode: INPUT_PULLDOWN
    internal: true
    on_click:
      min_length: 50ms
      max_length: 500ms
      then:
        - lambda: |-
            id(current_set) = (id(current_set) + 1) % 4;
        - script.execute: show_current_set
  - platform: gpio
    id: button_ok
    pin:
      number: ${orbital_button_ok_pin}
      mode: INPUT_PULLDOWN
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

interval:
  - interval: ${orbital_autocycle_interval}
    then:
      - if:
          condition:
            lambda: "return id(autocycle_enabled);"
          then:
            - lambda: |-
                id(current_set) = (id(current_set) + 1) % 4;
            - script.execute: show_current_set

script:
  - id: show_current_set
    mode: restart
    then:
      - if:
          condition:
            lambda: "return id(current_set) == 0;"
          then:
            - lvgl.page.show:
                id: clock_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: clock_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: clock_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: clock_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: clock_5
                lvgl_id: lvgl_5
      - if:
          condition:
            lambda: "return id(current_set) == 1;"
          then:
            - lvgl.page.show:
                id: weather_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: weather_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: weather_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: weather_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: weather_5
                lvgl_id: lvgl_5
      - if:
          condition:
            lambda: "return id(current_set) == 2;"
          then:
            - lvgl.page.show:
                id: presence_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: presence_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: presence_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: presence_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: presence_5
                lvgl_id: lvgl_5
      - if:
          condition:
            lambda: "return id(current_set) == 3;"
          then:
            - lvgl.page.show:
                id: custom_1
                lvgl_id: lvgl_1
            - lvgl.page.show:
                id: custom_2
                lvgl_id: lvgl_2
            - lvgl.page.show:
                id: custom_3
                lvgl_id: lvgl_3
            - lvgl.page.show:
                id: custom_4
                lvgl_id: lvgl_4
            - lvgl.page.show:
                id: custom_5
                lvgl_id: lvgl_5
```

- [ ] **Step 2: Add the `navigation:` package entry**

Find:

```yaml
  custom_5: !include
    file: orbital_screen_custom.yaml
    vars:
      index: "5"
      entity: "${orbital_custom5_entity}"
      label: "${orbital_custom5_label}"
      icon: "${orbital_custom5_icon}"
      unit: "${orbital_custom5_unit}"
```

Replace with:

```yaml
  custom_5: !include
    file: orbital_screen_custom.yaml
    vars:
      index: "5"
      entity: "${orbital_custom5_entity}"
      label: "${orbital_custom5_label}"
      icon: "${orbital_custom5_icon}"
      unit: "${orbital_custom5_unit}"
  navigation: !include orbital_navigation.yaml
```

- [ ] **Step 3: Remove `globals:`, `binary_sensor:`, the remaining `interval:`, and `script:` from `orbital_base.yaml`**

Delete the entire `globals:` block (it now only exists in `orbital_navigation.yaml`).

Delete the entire `binary_sensor:` block (the 3 buttons — the `status` platform entry was already removed in Task 4).

Delete the entire `interval:` block (only the autocycle entry remains, per Task 5 Step 7).

Delete the entire `script:` block (only `show_current_set` remains, per Task 5 Step 7).

After this step, `packages/orbital_base.yaml` should contain only: `substitutions:`, `esphome:`, `esp32:`, `api:`, `wifi:`, `logger:`, `time:`, `sun:`, and `packages:`.

- [ ] **Step 4: Validate**

```bash
cd /Users/jsievert/personal/esphome
esphome config packages/orbital_base.yaml > /tmp/orbital_after_navigation.txt
diff /tmp/orbital_after_screens.txt /tmp/orbital_after_navigation.txt
```
Expected: no diff.

- [ ] **Step 5: Compile and check size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: succeeds; same size as Task 5's Step 9 result.

- [ ] **Step 6: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_navigation.yaml packages/orbital_base.yaml
git commit -m "Extract navigation (globals/buttons/autocycle/set-switching) into orbital_navigation.yaml

orbital_base.yaml is now a thin composition root: substitutions, core
device settings, and the packages: block. All screen/hardware/nav/
diagnostic content lives in its own file under packages/."
```

---

### Task 7: Final verification

**Files:**
- None (verification only, plus a possible size-comparison note if you want one recorded)

- [ ] **Step 1: Full config diff against the very first baseline**

```bash
cd /Users/jsievert/personal/esphome
esphome config packages/orbital_base.yaml > /tmp/orbital_final_config.txt
diff /tmp/orbital_baseline_config.txt /tmp/orbital_final_config.txt
```
Expected: the only difference is the absence of the `web_server:` section (and its dependent internal config). No sensor, entity, page, or script content differs.

- [ ] **Step 2: Compile and confirm final size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: succeeds; size ≤ 1337392 (the Task 1 baseline), matching Task 6's result.

- [ ] **Step 3: Validate both external consumer files still work**

```bash
cd /Users/jsievert/personal/esphome
esphome config tests/test_orbital.yaml
esphome config examples/orbital_example.yaml
```
Expected: both succeed. (`examples/orbital_example.yaml` references `!secret` values that may not resolve outside a real ESPHome secrets context — if it fails only on missing secrets, not on the package structure, that's expected and fine; if it fails on anything related to the package files themselves, that's a real regression.)

- [ ] **Step 4: Confirm final file layout**

```bash
cd /Users/jsievert/personal/esphome
ls packages/orbital_*.yaml
```
Expected: `orbital_base.yaml`, `orbital_diagnostics.yaml`, `orbital_fonts.yaml`, `orbital_hardware.yaml`, `orbital_navigation.yaml`, `orbital_screen_custom.yaml`, `orbital_screen_presence.yaml`, `orbital_screens_clock.yaml`, `orbital_screens_weather.yaml` — 9 files.

- [ ] **Step 5: Report the final numbers**

No commit needed if Steps 1–4 all pass cleanly (nothing left to change). If anything surfaced a real discrepancy, fix it, re-run Steps 1–4, and commit the fix before considering the plan complete.
