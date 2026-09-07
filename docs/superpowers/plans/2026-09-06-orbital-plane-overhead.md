# Orbital Plane-Overhead Popup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a pop-up screen to the Orbital device that interrupts all five displays with detail about the nearest airborne aircraft whenever one is within a small radius of the house, then automatically restores whatever was showing before.

**Architecture:** Two new HA template entities (`sensor.orbital_nearest_aircraft`, `binary_sensor.orbital_plane_overhead`) derive from the already-installed FlightRadar24 integration's `sensor.flightradar24_current_in_area`. ESPHome gains one new package file (`orbital_screen_plane.yaml`, 5 pages `!extend`ing the existing `lvgl_1..5` instances) plus a hook in `orbital_navigation.yaml`: the overhead binary_sensor's `on_press` shows the plane pages, its `on_release` just re-invokes the existing `show_current_set` script — no new state variables, since the popup never touches the `current_set` global.

**Tech Stack:** ESPHome 2026.8.2 YAML config + Home Assistant YAML templates (modern `template:` platform, state-based — no `trigger:` block needed since these templates recompute automatically whenever the `sensor.flightradar24_current_in_area` entity they reference changes). Verification is `esphome config`/`esphome compile` for the firmware side, and Home Assistant's own template evaluation (Developer Tools or live entity state) for the HA side.

**Spec:** `docs/superpowers/specs/2026-09-06-orbital-plane-overhead-design.md`

## Global Constraints

- External consumer contract preserved: `examples/orbital_example.yaml` and `tests/test_orbital.yaml` must keep validating with no changes to either file — the two new substitutions (`orbital_plane_entity`, `orbital_plane_overhead_entity`) get sensible non-secret defaults directly in `orbital_base.yaml`.
- No direct third-party API calls in firmware — all aircraft data reaches ESPHome exclusively through the two HA template entities defined in Task 5, exactly like every other HA-sourced screen in this device.
- The popup must never modify the `current_set` global. `on_release` always calls `show_current_set` unmodified — this is the mechanism that makes "restore whatever was showing" work with zero new state.
- Button presses are deliberately NOT guarded against the popup — pressing a button while a plane is overhead switches sets immediately, overriding the popup. This is intentional, not a gap to fix.
- Font glyph codepoints are exact values confirmed against the pinned `@mdi/font@7.4.47` CSS (the same version `orbital_fonts.yaml` already pins) during this plan's authoring: `mdi:airplane` = `\U000F001D`, `mdi:helicopter` = `\U000F0AC2`. Do not substitute different codepoints.
- `esphome config packages/orbital_base.yaml` does NOT validate standalone — always validate via `esphome config tests/test_orbital.yaml`.
- `!extend` requires the base `lvgl_1..5` ids to already exist — `orbital_screen_plane.yaml`'s `packages:` entry must be declared after `hardware` in `orbital_base.yaml`'s `packages:` map (declaration order no longer functionally matters per the 2026-09-06 correction to the modularization plan/spec, but keeping it after `hardware` for readability is still good practice).
- Screens 2 and 3 (`aircraft_model`, `airline`) use LVGL wrapping labels — a new technique for this device; every other screen so far uses single-line truncation. Exact wrap width is a starting value (190px) to be confirmed visually once flashed to real hardware; there is no simulator for this.

---

### Task 1: Capture ESPHome baseline

**Files:**
- Create (scratch, not committed): baseline config/size snapshots

**Interfaces:**
- Produces: baseline `firmware.factory.bin` byte count and a saved `esphome config` dump, used by Task 6's regression diff.

- [ ] **Step 1: Confirm working tree is clean**

```bash
cd /Users/jsievert/personal/esphome && git status --short
```
Expected: no output (the two 2026-09-06 commits from the prior code-review fix and the new spec should already be committed — if `git log --oneline -3` doesn't show them, stop and check with the user before proceeding).

- [ ] **Step 2: Save the fully-resolved baseline config**

```bash
cd /Users/jsievert/personal/esphome
esphome config tests/test_orbital.yaml > /tmp/orbital_plane_baseline_config.txt
wc -l /tmp/orbital_plane_baseline_config.txt
```
Expected: succeeds, prints a non-zero line count.

- [ ] **Step 3: Rebuild and record the baseline firmware size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: build succeeds; note the printed byte count as the baseline for Task 6.

No commit — this task produces no repo changes.

---

### Task 2: Add plane-overhead icon glyphs

**Files:**
- Modify: `packages/orbital_fonts.yaml`

**Interfaces:**
- Consumes: nothing new.
- Produces: two new codepoints in the `weather_icons` font's glyph list — `\U000F001D` (airplane) and `\U000F0AC2` (helicopter) — consumed by Task 3's icon-selection lambda.

- [ ] **Step 1: Add the two glyphs to the palette**

In `packages/orbital_fonts.yaml`, find the end of the general-purpose glyph palette:

```yaml
      "\U000F009A", # bell
      "\U000F03D6", # package-variant
      ]
```

Replace with:

```yaml
      "\U000F009A", # bell
      "\U000F03D6", # package-variant
      # Plane-overhead icons (selected by aircraft_category)
      "\U000F001D", # airplane
      "\U000F0AC2", # helicopter
      ]
```

- [ ] **Step 2: Validate and measure**

```bash
cd /Users/jsievert/personal/esphome
esphome config tests/test_orbital.yaml > /tmp/orbital_plane_after_glyphs.txt
diff /tmp/orbital_plane_baseline_config.txt /tmp/orbital_plane_after_glyphs.txt
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: `diff` shows only the two new glyph entries in the resolved `font:` block's glyph list — nothing else differs. Compile succeeds. Size increases slightly (two more glyphs baked into the font bitmap) — a few hundred bytes to a couple KB is normal; note the new number.

- [ ] **Step 3: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_fonts.yaml
git commit -m "$(cat <<'EOF'
Add airplane/helicopter glyphs for the plane-overhead popup

Codepoints (mdi:airplane U+F001D, mdi:helicopter U+F0AC2) confirmed
against the pinned @mdi/font@7.4.47 CSS, not guessed.
EOF
)"
```

---

### Task 3: Add `orbital_screen_plane.yaml` and wire it into `orbital_base.yaml`

**Files:**
- Create: `packages/orbital_screen_plane.yaml`
- Modify: `packages/orbital_base.yaml`

**Interfaces:**
- Consumes: `font.weather_icons` (Task 2's glyphs), `lvgl_1..5` base instances (from `orbital_hardware.yaml`, unchanged).
- Produces: page ids `plane_1..plane_5` (consumed by Task 4's navigation hook); substitutions `orbital_plane_entity`/`orbital_plane_overhead_entity` (consumed by this task's own bindings and Task 4's binary_sensor).

- [ ] **Step 1: Create `packages/orbital_screen_plane.yaml`**

```yaml
# ---------------------------------------------------------------------------
# Plane-overhead popup: 5 facets of the single nearest airborne aircraft,
# sourced from orbital_plane_entity (identity as state, aircraft_model/
# airline/route/altitude as attributes). Included once from orbital_base.yaml's
# packages: block -- these are 5 facets of ONE entity, like the Weather set,
# not a per-slot template like Presence/Custom. The popup interrupt itself
# (showing plane_1..5, restoring afterward) lives in orbital_navigation.yaml,
# not here -- this file only owns the pages and their content bindings.
# ---------------------------------------------------------------------------

lvgl:
  - id: !extend lvgl_1
    pages:
      - id: plane_1
        widgets:
          - label:
              id: plane_icon_1
              align: CENTER
              y: -44
              text_font: weather_icons
              text_color: 0x00D9FF
              text: "\U000F001D"
          - label:
              id: plane_label_1
              align: CENTER
              y: 4
              text: "-"

  - id: !extend lvgl_2
    pages:
      - id: plane_2
        widgets:
          # aircraft_model can run long ("Mitsubishi CRJ-900LR") -- wrapped
          # at montserrat_14 (already compiled in for Presence/Custom
          # captions, so this costs no extra flash) rather than truncated.
          - label:
              id: plane_label_2
              align: CENTER
              width: 190
              long_mode: WRAP
              text_align: CENTER
              text_font: montserrat_14
              text: "-"

  - id: !extend lvgl_3
    pages:
      - id: plane_3
        widgets:
          # airline/operator can also run long ("Thunderbird Aviation").
          - label:
              id: plane_label_3
              align: CENTER
              width: 190
              long_mode: WRAP
              text_align: CENTER
              text_font: montserrat_14
              text: "-"

  - id: !extend lvgl_4
    pages:
      - id: plane_4
        widgets:
          - label:
              id: plane_label_4
              align: CENTER
              text: "-"

  - id: !extend lvgl_5
    pages:
      - id: plane_5
        widgets:
          - label:
              id: plane_label_5
              align: CENTER
              text: "-"

text_sensor:
  # Screen 1: identity (flight_number if present, else callsign/tail
  # number -- orbital_plane_entity's own state already resolves that
  # fallback in HA) and its icon, chosen from the aircraft_category
  # attribute.
  - platform: homeassistant
    id: plane_identity_ts
    entity_id: ${orbital_plane_entity}
    internal: true
    on_value:
      - lvgl.label.update:
          id: plane_label_1
          text: !lambda "return x.substr(0, 10);"
  - platform: homeassistant
    id: plane_category_ts
    entity_id: ${orbital_plane_entity}
    attribute: aircraft_category
    internal: true
    on_value:
      - lvgl.label.update:
          id: plane_icon_1
          text: !lambda |-
            if (x == "Airplane") return std::string("\U000F001D");
            if (x == "Helicopter") return std::string("\U000F0AC2");
            return std::string("\U000F001D"); // unrecognized category

  # Screen 2: aircraft model.
  - platform: homeassistant
    id: plane_model_ts
    entity_id: ${orbital_plane_entity}
    attribute: aircraft_model
    internal: true
    on_value:
      - lvgl.label.update:
          id: plane_label_2
          text: !lambda "return x;"

  # Screen 3: operator/airline.
  - platform: homeassistant
    id: plane_airline_ts
    entity_id: ${orbital_plane_entity}
    attribute: airline
    internal: true
    on_value:
      - lvgl.label.update:
          id: plane_label_3
          text: !lambda "return x;"

  # Screen 4: route, precomputed as one "ORIGIN → DEST" string in HA so
  # ESPHome doesn't need to combine two attributes with a null fallback.
  - platform: homeassistant
    id: plane_route_ts
    entity_id: ${orbital_plane_entity}
    attribute: route
    internal: true
    on_value:
      - lvgl.label.update:
          id: plane_label_4
          text: !lambda "return x;"

sensor:
  # Screen 5: altitude, read as a numeric attribute (not text_sensor) so it
  # can be formatted -- same technique as the Weather set's temperature/
  # humidity/wind_speed attributes.
  - platform: homeassistant
    id: plane_altitude_num
    entity_id: ${orbital_plane_entity}
    attribute: altitude
    internal: true
    on_value:
      - lvgl.label.update:
          id: plane_label_5
          text:
            format: "%.0fft"
            args: [x]
```

- [ ] **Step 2: Add the two new substitutions**

In `packages/orbital_base.yaml`, find the end of the substitutions block:

```yaml
  orbital_custom5_entity: sensor.custom_5
  orbital_custom5_icon: "\U000F029A"  # mdi:gauge
  orbital_custom5_label: "Custom 5"
  orbital_custom5_unit: ""

esphome:
```

Replace with:

```yaml
  orbital_custom5_entity: sensor.custom_5
  orbital_custom5_icon: "\U000F029A"  # mdi:gauge
  orbital_custom5_label: "Custom 5"
  orbital_custom5_unit: ""

  # Plane-overhead popup -- sourced from two HA template entities derived
  # from the FlightRadar24 integration (see
  # docs/superpowers/plans/2026-09-06-orbital-plane-overhead.md Task 5 for
  # their exact definitions). orbital_plane_entity carries the nearest
  # airborne aircraft's identity as its state and aircraft_model/airline/
  # route/altitude/aircraft_category as attributes; orbital_plane_overhead_entity
  # is true only while an airborne aircraft is within range.
  orbital_plane_entity: sensor.orbital_nearest_aircraft
  orbital_plane_overhead_entity: binary_sensor.orbital_plane_overhead

esphome:
```

- [ ] **Step 3: Add the `planes:` package entry**

Find:

```yaml
  navigation: !include orbital_navigation.yaml
```

Replace with:

```yaml
  planes: !include orbital_screen_plane.yaml
  navigation: !include orbital_navigation.yaml
```

- [ ] **Step 4: Validate**

```bash
cd /Users/jsievert/personal/esphome
esphome config tests/test_orbital.yaml > /tmp/orbital_plane_after_screen.txt
```
Expected: succeeds (no error). This is an additive change (new pages, never shown by anything yet), so no diff comparison against the previous dump is meaningful here — just confirm it validates.

- [ ] **Step 5: Compile and check size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: succeeds. Size increases versus Task 2's result (new pages, sensors, lambdas) — note the new number; expect an increase in the same rough range as one of the existing single-entity screen sets (Weather's screens 1-4, say), not a dramatic jump.

- [ ] **Step 6: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_screen_plane.yaml packages/orbital_base.yaml
git commit -m "$(cat <<'EOF'
Add plane-overhead screen content (orbital_screen_plane.yaml)

5 facets of the single nearest airborne aircraft: icon+identity,
aircraft model, operator/airline, route, altitude. Pages are dormant
until orbital_navigation.yaml's popup hook (next commit) shows them --
this commit only adds the content and its HA bindings.
EOF
)"
```

---

### Task 4: Wire the popup interrupt into `orbital_navigation.yaml`

**Files:**
- Modify: `packages/orbital_navigation.yaml`

**Interfaces:**
- Consumes: page ids `plane_1..plane_5` (Task 3); substitution `orbital_plane_overhead_entity` (Task 3).
- Produces: nothing consumed elsewhere — this is the top of the interrupt chain.

- [ ] **Step 1: Add the overhead binary_sensor and guard the autocycle interval**

In `packages/orbital_navigation.yaml`, find:

```yaml
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
```

Replace with:

```yaml
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
  # Plane-overhead popup. Deliberately does not touch current_set: on_press
  # shows the plane pages directly; on_release just re-invokes
  # show_current_set, which restores whatever current_set still says since
  # this interrupt never changed it. Button presses are NOT guarded against
  # this -- a button press always wins over the popup.
  - platform: homeassistant
    id: plane_overhead
    entity_id: ${orbital_plane_overhead_entity}
    internal: true
    on_press:
      - lvgl.page.show:
          id: plane_1
          lvgl_id: lvgl_1
      - lvgl.page.show:
          id: plane_2
          lvgl_id: lvgl_2
      - lvgl.page.show:
          id: plane_3
          lvgl_id: lvgl_3
      - lvgl.page.show:
          id: plane_4
          lvgl_id: lvgl_4
      - lvgl.page.show:
          id: plane_5
          lvgl_id: lvgl_5
    on_release:
      - script.execute: show_current_set

interval:
  - interval: ${orbital_autocycle_interval}
    then:
      - if:
          condition:
            # Guarded so autocycle doesn't yank the plane pages away
            # mid-flyover -- it just skips its tick while the popup is up.
            lambda: "return id(autocycle_enabled) && !id(plane_overhead).state;"
          then:
            - lambda: |-
                id(current_set) = (id(current_set) + 1) % 4;
            - script.execute: show_current_set
```

- [ ] **Step 2: Validate**

```bash
cd /Users/jsievert/personal/esphome
esphome config tests/test_orbital.yaml > /tmp/orbital_plane_after_nav.txt
```
Expected: succeeds.

- [ ] **Step 3: Compile and check size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: succeeds; a small increase over Task 3's result (one more binary_sensor + its automations).

- [ ] **Step 4: Commit**

```bash
cd /Users/jsievert/personal/esphome
git add packages/orbital_navigation.yaml
git commit -m "$(cat <<'EOF'
Wire plane-overhead popup into navigation

on_press shows plane_1..5 on all five displays; on_release re-invokes
show_current_set, restoring whatever was showing since the popup never
touches current_set. Autocycle is guarded so it won't interrupt an
active popup; button presses are deliberately left unguarded.
EOF
)"
```

---

### Task 5: Add the HA-side template entities

This is a manual step in the user's live Home Assistant configuration — there is no MCP tool available in this environment that can write a `template:` entry with custom `attributes:` into HA's YAML config (the UI "Template" helper flow only supports a single state template, not custom attributes). The ESPHome-side tasks above do not depend on this being done first — `platform: homeassistant` components validate and compile regardless of whether the referenced entity exists yet, since that's resolved at runtime over the API connection, not at compile time.

**Files:**
- Modify (user's Home Assistant config, not this repo): `configuration.yaml`, or wherever the user keeps their own `template:` entries (e.g. a packages file, if they use one).

**Interfaces:**
- Produces: `sensor.orbital_nearest_aircraft` and `binary_sensor.orbital_plane_overhead`, consumed by Task 3/4's ESPHome bindings via `orbital_plane_entity`/`orbital_plane_overhead_entity`.

- [ ] **Step 1: Add this block to the user's HA configuration**

```yaml
template:
  - sensor:
      - name: "Orbital Nearest Aircraft"
        unique_id: orbital_nearest_aircraft
        state: >
          {% set flights = state_attr('sensor.flightradar24_current_in_area', 'flights') | default([], true) | selectattr('on_ground', 'equalto', 0) | list %}
          {% if flights | count == 0 %}
            none
          {% else %}
            {% set f = flights | sort(attribute='distance') | first %}
            {{ f.flight_number or f.callsign }}
          {% endif %}
        attributes:
          aircraft_model: >
            {% set flights = state_attr('sensor.flightradar24_current_in_area', 'flights') | default([], true) | selectattr('on_ground', 'equalto', 0) | list %}
            {{ (flights | sort(attribute='distance') | first).aircraft_model if flights | count > 0 else '?' }}
          airline: >
            {% set flights = state_attr('sensor.flightradar24_current_in_area', 'flights') | default([], true) | selectattr('on_ground', 'equalto', 0) | list %}
            {% if flights | count == 0 %}
              ?
            {% else %}
              {% set f = flights | sort(attribute='distance') | first %}
              {{ f.airline or '?' }}
            {% endif %}
          route: >
            {% set flights = state_attr('sensor.flightradar24_current_in_area', 'flights') | default([], true) | selectattr('on_ground', 'equalto', 0) | list %}
            {% if flights | count == 0 %}
              ?
            {% else %}
              {% set f = flights | sort(attribute='distance') | first %}
              {{ f.airport_origin_code_iata or '?' }} -> {{ f.airport_destination_code_iata or '?' }}
            {% endif %}
          altitude: >
            {% set flights = state_attr('sensor.flightradar24_current_in_area', 'flights') | default([], true) | selectattr('on_ground', 'equalto', 0) | list %}
            {{ (flights | sort(attribute='distance') | first).altitude if flights | count > 0 else 0 }}
          aircraft_category: >
            {% set flights = state_attr('sensor.flightradar24_current_in_area', 'flights') | default([], true) | selectattr('on_ground', 'equalto', 0) | list %}
            {{ (flights | sort(attribute='distance') | first).aircraft_category if flights | count > 0 else '' }}
  - binary_sensor:
      - name: "Orbital Plane Overhead"
        unique_id: orbital_plane_overhead
        state: >
          {{ state_attr('sensor.flightradar24_current_in_area', 'flights') | default([], true) | selectattr('on_ground', 'equalto', 0) | list | count > 0 }}
```

If the user already has a top-level `template:` key elsewhere in their config, merge this `sensor:`/`binary_sensor:` content into the existing list rather than adding a second `template:` key (HA allows multiple `template:` blocks via packages, but within a single file only one top-level `template:` key is valid YAML).

- [ ] **Step 2: Restart Home Assistant to pick up the new template entities**

New top-level `template:` entries require a full restart (not just a YAML reload) the first time they're added to a file HA didn't already have loaded as a `template:` platform.

- [ ] **Step 3: Verify both entities exist and resolve correctly**

Ask the user to confirm the restart is complete, then check:

```
ha_get_state(["sensor.orbital_nearest_aircraft", "binary_sensor.orbital_plane_overhead"])
```

Expected: both entities exist. If no aircraft is currently overhead (likely, given the small radius), `sensor.orbital_nearest_aircraft` reads `none` and `binary_sensor.orbital_plane_overhead` reads `off` — that's correct, not a failure. If either entity is missing or shows an error/unknown state, re-check the template YAML for a syntax mistake before proceeding (Home Assistant logs template errors under Settings → System → Logs).

No commit — this step modifies the user's HA config, not this repo.

---

### Task 6: Final verification

**Files:**
- None (verification only)

- [ ] **Step 1: Full config diff against the very first baseline**

```bash
cd /Users/jsievert/personal/esphome
esphome config tests/test_orbital.yaml > /tmp/orbital_plane_final_config.txt
diff /tmp/orbital_plane_baseline_config.txt /tmp/orbital_plane_final_config.txt
```
Expected: differences are limited to the new glyphs, the new `plane_1..5` pages, the new `plane_overhead` binary_sensor, the guarded autocycle condition, and the two new substitutions. Nothing about the existing Clock/Weather/Presence/Custom pages or their behavior should differ.

- [ ] **Step 2: Compile and confirm final size**

```bash
cd /Users/jsievert/personal/esphome
esphome compile tests/test_orbital.yaml
stat -f "%z" tests/.esphome/build/orbital-test/build/firmware.factory.bin
```
Expected: succeeds. Report the final size and the total delta versus Task 1's baseline in the task's completion notes — there is no hard ceiling for this feature (unlike the prior modularization work), but a large, unexplained jump is worth a second look before considering this done.

- [ ] **Step 3: Validate both external consumer files still work**

```bash
cd /Users/jsievert/personal/esphome
esphome config tests/test_orbital.yaml
esphome config examples/orbital_example.yaml
```
Expected: both succeed with no changes needed to either file (the new substitutions have defaults in `orbital_base.yaml`). If `examples/orbital_example.yaml` fails only on missing `!secret` values unrelated to this change, that's expected and pre-existing, not a regression.

- [ ] **Step 4: Confirm live HA entities one more time**

```
ha_get_state(["sensor.orbital_nearest_aircraft", "binary_sensor.orbital_plane_overhead"])
```
Expected: same as Task 5 Step 3 — both resolve without error.

- [ ] **Step 5: Flash and observe on real hardware**

There is no simulator for LVGL layout on this device. Ask the user to flash the updated firmware and confirm:
- The device behaves identically to before when no plane is overhead (this is the common case at the real configured radius).
- If a real flyover happens to occur during testing (or the radius is temporarily widened the same way it was during design, per the spec's noted testing constraint), all five displays switch to the plane pages, screens 2/3 wrap legibly within the round display rather than overflowing it, and the displays revert to whatever was showing before once the aircraft leaves the area.
- If the wrap width (190px) clips or overflows on real hardware, adjust `width:` on `plane_label_2`/`plane_label_3` in `orbital_screen_plane.yaml` and re-flash — this is expected tuning, not a sign anything upstream is wrong.

No commit needed if everything above passes cleanly. If real-hardware observation surfaces a layout or logic issue, fix it, re-run Steps 1–4, and commit the fix before considering the plan complete.
