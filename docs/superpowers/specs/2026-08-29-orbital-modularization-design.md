# Orbital Package Modularization — Design

## Goal

Split `packages/orbital_base.yaml` (1437 lines, one monolithic file) into
focused files under `packages/`, and collapse the Presence/Custom sets'
5x-duplicated per-display blocks into a single template each. Both
maintainability and flash headroom are goals; this design covers the
maintainability half plus one confirmed flash win (dropping `web_server`).
Further flash-specific trims (icon size, logger verbosity) are explicitly
deferred to follow-up work once this split makes single-file measurement
easy (see Out of Scope).

## Baseline (measured before this change)

- `esphome compile tests/test_orbital.yaml` produces `firmware.factory.bin`
  at 1,337,392 bytes against a 1,835,008-byte OTA app partition — **72.9%
  full, 486 KB free**.
- Confirmed via `esphome config`: `packages:` merging is non-destructive —
  dicts merge key-by-key, lists of components merge by declared `id:`,
  other lists concatenate. The "one top-level key per file" limitation
  noted in earlier orbital docs describes the separate `<<: !include`
  shallow-merge mechanism, not `packages:` — it does not apply here.
- Confirmed via spike (`/private/tmp/.../scratchpad/merge_spike`, throwaway,
  not committed): a second package file can append a `pages:` entry to an
  `lvgl:` instance defined in another file using `id: !extend lvgl_1`.
  Plain re-declaration of the same id across files fails validation
  ("ID lvgl_1 redefined"). `!extend` is required specifically for the
  nested `lvgl.pages` case; every other top-level list in this config
  (`binary_sensor`, `sensor`, `text_sensor`, `button`, `script`,
  `interval`) splits across files by plain concatenation, since no two
  files contribute conflicting ids there.
- Confirmed via spike: `!include`'s `vars:` correctly resolves a `${...}`
  reference to an outer substitution (e.g. `vars: {entity:
  "${orbital_person1_entity}"}` resolves to the substitution's value).
  This is what lets the Presence/Custom templates keep reading from the
  existing `orbital_person*`/`orbital_custom*` substitutions — end users
  customize the device exactly as they do today.

## File Layout

`packages/orbital_base.yaml` remains the only file external consumers
reference (`examples/orbital_example.yaml`, `tests/test_orbital.yaml`
both do `files: [packages/orbital_base.yaml]` and are unaffected).

**`packages/orbital_base.yaml`** (root/composition, ~150 lines)
- `substitutions:` — unchanged, all of it (every sub-file reads from
  this shared set regardless of which file it lives in)
- `esphome:` (incl. `on_boot` status LED hook), `esp32:`, `api:`,
  `wifi:` (incl. `on_connect`/`on_disconnect` status LED hooks),
  `logger:`, `time:`, `sun:`
- `web_server:` — **removed** (confirmed unused; local web UI not needed,
  device is controlled via HA/API)
- `packages:` block importing the 8 files below

**`orbital_hardware.yaml`** — `spi:`; 5x `display:`; status `light:`;
the `lvgl:` list's base declarations (`id`, `displays`, `buffer_size`,
`default_font`, `theme` on `lvgl_1` only, `pages: []`) — screen files
below each `!extend` one page onto these.

*Implementation note:* the spike only confirmed `!extend` appending a
second page to a base that already had one page (`part_a.yaml` declared
`pages: [page_a]`, `part_b.yaml` `!extend`ed in `page_b`) — an empty
`pages: []` base was not itself spike-tested. If that doesn't extend
cleanly, fall back to having `orbital_screens_clock.yaml` establish each
`lvgl_N` entry's first page (`clock_N`) directly instead of `hardware.yaml`
declaring an empty list, and have weather/presence/custom `!extend` from
there. Also note: `packages:` in `orbital_base.yaml` must list
`hardware` (or whichever file establishes the base `lvgl:` entries)
*before* any file that `!extend`s them — package merge order follows
declaration order in the `packages:` mapping.

**`orbital_fonts.yaml`** — the MDI glyph subset (`font:`), unchanged.

**`orbital_screens_clock.yaml`** — `!extend`s `clock_1..5` pages onto the
5 lvgl instances (analog clock, calendar, waste pickup, sunrise/sunset,
indoor temp); the 1s (clock hands) and 60s (calendar/waste/sun) `interval:`
entries; `update_sun_screen`/`update_waste_screen` scripts; the HA entity
bindings that feed this set (`indoor_temp_num` sensor, `waste_desc_ts`/
`waste_start_ts` text_sensors).

**`orbital_screens_weather.yaml`** — `!extend`s `weather_1..5` pages;
the weather entity bindings (`weather_temp_num`, `weather_humidity_num`,
`weather_wind_num` sensors; `weather_condition_ts`, `weather_forecast_ts`
text_sensors).

**`orbital_screen_presence.yaml`** — template, included 5x from root's
`packages:` block with `vars: {index, entity, label}`. Contains one
`!extend`ed `presence_${index}` page plus the one `presence_person${index}_ts`
text_sensor that drives it. Replaces 5 copy-pasted ~29-line blocks with
1 template + 5 one-line inclusions.

**`orbital_screen_custom.yaml`** — same pattern as presence, `vars:
{index, entity, label, icon, unit}`, for the 5 custom slots.

**`orbital_navigation.yaml`** — `globals:` (`current_set`,
`autocycle_enabled`); the 3 nav buttons (`binary_sensor: button_left/
button_ok/button_right`, including their `on_click` behavior); the
autocycle `interval:` entry; the `show_current_set` script.

**`orbital_diagnostics.yaml`** — `binary_sensor: status`; `sensor:
wifi_signal_sensor/uptime_sensor`; `text_sensor: version/wifi_info`;
`button: restart/safe_mode`.

### Example: presence template

```yaml
# orbital_screen_presence.yaml
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

```yaml
# in orbital_base.yaml's packages: block
packages:
  presence_1: !include
    file: orbital_screen_presence.yaml
    vars: {index: "1", entity: "${orbital_person1_entity}", label: "${orbital_person1_label}"}
  presence_2: !include
    file: orbital_screen_presence.yaml
    vars: {index: "2", entity: "${orbital_person2_entity}", label: "${orbital_person2_label}"}
  # ... 3, 4, 5
```

## Testing / Verification Plan

1. Before touching anything: capture `esphome config packages/orbital_base.yaml`
   output as a baseline (already-substituted defaults from the package
   itself) and `esphome compile tests/test_orbital.yaml`'s
   `firmware.factory.bin` size (baseline: 1,337,392 bytes).
2. After the split: `esphome config packages/orbital_base.yaml` again and
   diff against the baseline. Expect it identical except the `web_server:`
   block being gone — this refactor is behavior-preserving otherwise.
3. `esphome compile tests/test_orbital.yaml` must succeed. Compare the new
   `firmware.factory.bin` size against baseline — expect roughly equal or
   slightly smaller (web_server removal), not larger.
4. Confirm `esphome config` on `examples/orbital_example.yaml` (with
   dummy secrets) and `tests/test_orbital.yaml` both still validate.

## Out of Scope (this change)

- Icon font size reduction, logger verbosity trims, or any other
  flash-size cut beyond removing `web_server` — these are real levers
  but change visual/diagnostic behavior and should be measured
  individually, not bundled here.
- Clock and Weather sets are **not** templated — each of their 5 displays
  shows genuinely different content (analog clock vs. calendar vs. waste
  pickup vs. sunrise/sunset vs. indoor temp), so there's no duplication
  to collapse there.
- Any new feature (menu screen, flight-overhead screen, per-screen
  independent paging) — this is a pure refactor, no new behavior.
