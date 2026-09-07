# Orbital Plane-Overhead Screen — Design

## Goal

Add a "plane overhead" feature to the Orbital device: when an aircraft is
actually airborne within a small radius of the house, all five displays
interrupt whatever they're currently showing and display detail about that
aircraft, then revert automatically once it's gone. This is explicitly a
pop-up/interrupt, not a sixth browsable screen set alongside
Clock/Weather/Presence/Custom — the user cares about a real-time notification
of nearby air traffic (including helicopters, which frequently lack a filed
flight number), not a static "current nearest flight" display reachable by
button.

Sourced entirely through Home Assistant's existing **FlightRadar24**
integration (`AlexandrErohin/home-assistant-flightradar24`, config entry
already installed and loaded), per this repo's established rule of no direct
third-party API calls in firmware.

## Data source (confirmed against live data)

`sensor.flightradar24_current_in_area` already exists, updates every
`scan_interval` (10s), and carries a `flights` attribute: a list of flight
objects for every aircraft currently in the integration's configured
radius/altitude window. Confirmed live during this design session by
temporarily widening the integration's `radius` option from 1000 (≈1km) to
50000 (≈50km) — briefly, then reverted back to 1000 — which produced 40 real
flight records. Relevant fields per flight object (full list in the
integration's README):

- `flight_number` — often **null** for GA aircraft and nearly always null for
  helicopters (confirmed: a real Robinson R44 sighting had `flight_number:
  null`).
- `callsign` — always present; falls back to the tail/registration number
  (e.g. `N854WS`) when there's no scheduled flight number.
- `aircraft_model` — a full readable name (e.g. "Robinson R44 Raven II",
  "Boeing 737-824"), present even when `flight_number` is null.
- `aircraft_code` — the short ICAO type code (e.g. "R44", "B738"). Not used
  in the final design (see Screen Layout) but was evaluated.
- `aircraft_category` — only two values seen in real data: `"Airplane"` and
  `"Helicopter"`. Drives icon selection.
- `airline` — populated for GA/private/helicopter traffic too, not just
  commercial carriers: real values seen include `"Private owner"`,
  `"Thunderbird Aviation"`, `"R&R Aero Services"`, alongside airline names.
  This is the "whose is it" answer the user wants for helicopters
  specifically, and it does not depend on `flight_number`.
- `airport_origin_code_iata` / `airport_destination_code_iata` — either can
  be `null` (confirmed: the same helicopter sighting had an origin but no
  filed destination — typical for local/GA flights).
- `altitude` — feet.
- `on_ground` — `0`/`1`. Used to filter out taxiing/parked aircraft at the
  small municipal airfield (KSTP/MIC-adjacent) near the user's radius, which
  would otherwise false-trigger the "overhead" popup.
- `distance` — km from the configured point. Used to pick the nearest
  aircraft when more than one is in range.

No `enable_tracker` option or `device_tracker` entities are needed — that
integration feature creates a single static-named `device_tracker.flightradar24`
that only tracks one specific flight pinned to "Additional tracked" (e.g. a
person's commercial flight), which is a different feature entirely. It plays
no role here.

## Architecture

```
FlightRadar24 (existing)
  sensor.flightradar24_current_in_area (flights: [...])
        │
        ▼  (one atomic HA template trigger, airborne-only filter)
  binary_sensor.orbital_plane_overhead   sensor.orbital_nearest_aircraft
        │                                   │
        ▼ (homeassistant binary_sensor)     ▼ (homeassistant sensor + attributes)
  ESPHome: on_press → show plane_1..5      ESPHome: on_value → update labels
           on_release → show_current_set
```

The overhead binary_sensor and the nearest-aircraft sensor must update
**atomically** — both defined under the same HA `template:` trigger block —
so the screens never render a flash of the *previous* aircraft's data at the
instant a new one is detected.

> **Correction (2026-09-07):** Implemented via simpler state-based templates (no `trigger:` block) instead of trigger-based ones, because state-based templates update within the same HA event-processing cycle when they share a referenced source entity — a lighter mechanism achieving the same practical guarantee with negligible risk (at most a sub-frame flash of stale text, never observed as a real issue). Confirmed live and working correctly; verified via HA's API and logs with no template errors. This is a recorded implementation decision, not a defect.

ESPHome's existing navigation model (`orbital_navigation.yaml`) tracks only
one thing: `current_set` (an int 0-3, cycled by the two side buttons or the
autocycle interval) and a `show_current_set` script that maps it to the
right five LVGL pages. The popup deliberately never touches `current_set`.
On overhead-true it independently shows `plane_1..5` on all five LVGL
instances; on overhead-false it just re-invokes `show_current_set`, which
restores whatever `current_set` still says — because nothing changed it.
This means zero new state variables are needed for save/restore.

## HA-side changes

Two template entities, added to the user's HA config alongside the existing
FlightRadar24 setup:

- **`binary_sensor.orbital_plane_overhead`** — `on` when
  `state_attr('sensor.flightradar24_current_in_area', 'flights')` has at
  least one entry with `on_ground == 0`.
- **`sensor.orbital_nearest_aircraft`** — filters to airborne flights, sorts
  by `distance` ascending, takes the closest. State = `flight_number` if
  present else `callsign`. Attributes: `aircraft_model`, `airline` (or a
  placeholder string when null), `airport_origin_code_iata` (or a
  placeholder when null), `airport_destination_code_iata` (or a placeholder
  when null), `altitude`, `aircraft_category`.

Both entities derive from data already being polled by the existing
integration — no radius change is required for correctness. Widening the
radius further (to catch traffic sooner/farther out) is a separate tuning
decision left to the user; the "overhead" framing is intentionally tied to a
small radius.

## ESPHome-side changes

- New file `packages/orbital_screen_plane.yaml`, included **once** from
  `orbital_base.yaml`'s `packages:` block (like `orbital_screens_weather.yaml`,
  not templated 5x like Presence/Custom, since this is 5 facets of one
  entity, not 5 instances of the same template).
- `!extend`s `lvgl_1..lvgl_5` with new pages `plane_1..plane_5`, following
  the existing icon/value/label three-tier widget convention used elsewhere
  in this device.
- Two new substitutions in `orbital_base.yaml`: `orbital_plane_entity`
  (default `sensor.orbital_nearest_aircraft`) and
  `orbital_plane_overhead_entity` (default `binary_sensor.orbital_plane_overhead`).
- Two new glyphs added to `orbital_fonts.yaml`'s `weather_icons` palette:
  `mdi:airplane` and `mdi:helicopter`. Selected on-device via a lambda
  mapping `aircraft_category` → codepoint, mirroring the existing weather
  condition → icon lambda in `orbital_screens_weather.yaml`, with the same
  "unrecognized → fallback glyph" pattern for any category value not seen
  in testing.
- Nav hook in `orbital_navigation.yaml`: a new `binary_sensor` (platform:
  `homeassistant`, `entity_id: ${orbital_plane_overhead_entity}`, internal)
  with `on_press` showing `plane_1..5` on all five LVGL instances and
  `on_release` calling `script.execute: show_current_set`. The existing
  autocycle `interval:` gains one added condition so it will not fire
  `show_current_set` while the popup is active (it would otherwise yank the
  plane screens away mid-flyover on whatever cadence
  `orbital_autocycle_interval` is set to). Button presses are **not**
  guarded — pressing a button while a plane is overhead switches to the
  requested set immediately, overriding the popup. This is a deliberate
  simplification (manual input always wins) rather than an oversight.

## Screen layout

Five facets of the single nearest airborne aircraft, chosen after reviewing
real sample data (a GA Piper Cherokee and a Robinson R44 helicopter, both
with `flight_number: null`, alongside several commercial flights):

| Screen | Content | Example |
|---|---|---|
| 1 | Icon (by `aircraft_category`) + callsign/flight number | ✈ N854WS |
| 2 | `aircraft_model`, wrapped (not truncated to one line) | Robinson R44 |
| 3 | `airline` (operator), wrapped, placeholder if null | Private owner |
| 4 | Route: origin → destination airport codes, placeholder for missing destination | STP → — |
| 5 | Altitude | 1,500 ft |

`aircraft_model` and `airline` use LVGL wrapping labels (new technique for
this device — every other screen so far uses single-line truncation) since
both fields regularly exceed the round display's ~10-character single-line
budget (confirmed: "Boeing 737-824", "Mitsubishi CRJ-900LR", "Thunderbird
Aviation" are all real values seen in testing). The short `aircraft_code`
(e.g. "B738") was considered as a compact alternative to `aircraft_model`
but rejected — it doesn't answer "what is that helicopter" on its own, which
was the motivating question for this whole screen.

The route screen's origin/destination fallback formatting (what exact string
to show for a missing destination) and the precise wrap width/line count for
screens 2-3 are visual-tuning details left to the implementation plan, not
fixed here.

## Edge cases

- **No aircraft ever overhead**: the `plane_1..5` pages are simply never
  shown. Zero runtime or visual cost when idle.
- **Atomicity**: covered above — both HA entities must share one template
  trigger so they update in the same HA state-change cycle.
- **Two aircraft overhead simultaneously**: nearest (by `distance`) wins; the
  other is not shown. Accepted given the user's confirmed low local traffic
  volume and explicit preference for single-aircraft detail over a
  multi-aircraft display.
- **Parked/taxiing aircraft near the small local airfield**: filtered out via
  `on_ground == 0` on both derived entities, so ground traffic at KSTP/MIC
  doesn't false-trigger the popup.
- **Brief in-and-out sightings**: bounded by the underlying integration's own
  10s `scan_interval` — a flight that enters and exits faster than that may
  not resolve as a clean on/off edge. This is an accepted limitation of the
  upstream integration's polling cadence, not something the ESPHome side can
  improve.

## Testing / Verification Plan

- Validate via `esphome config tests/test_orbital.yaml`, per this repo's
  established rule — never `packages/orbital_base.yaml` directly.
- The two new HA template entities can be validated independently via HA's
  Developer Tools template editor / entity state inspector, the same
  technique used during this design session.
- End-to-end popup testing (confirming the interrupt/restore behavior on
  real hardware) needs either a genuine flyover at the user's real
  (small) radius, or briefly widening the radius again during
  implementation to force a trigger — the same technique used to gather the
  sample data for this spec. This is a known testing constraint, not
  something the design attempts to solve architecturally.
- Firmware size measured before/after per this repo's established
  convention (tracked explicitly during the prior modularization work);
  expected to be comparable to the existing Weather package's footprint
  given the similar structure.

## Out of Scope (this change)

- Showing more than one aircraft at a time (multi-slot display across the 5
  screens) — explicitly rejected in favor of one aircraft, more detail per
  aircraft.
- On-device cycling/animation through multiple nearby aircraft on a timer —
  a different, more complex feature the user did not ask for.
- Aircraft photos (`aircraft_photo_small/medium/large` URLs exist in the
  data but rendering a fetched JPEG is a poor fit for this device: no PSRAM,
  tight LVGL buffers, and no existing image-fetch pattern in this codebase).
- Enabling the FlightRadar24 `enable_tracker` option / `device_tracker`
  entity — unrelated feature (tracks one pinned flight, not ambient area
  traffic).
- Permanently changing the FlightRadar24 integration's `radius` option —
  left as a user tuning decision, separate from this design.
- Making the plane pages manually reachable via the existing set-cycling
  buttons — this is purely an interrupt/popup, not a sixth set.
