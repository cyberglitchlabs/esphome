# AGENTS.md — ESPHome Device Configuration Repository

This is a YAML-based ESPHome device configuration repository for managing ESP8266/ESP32
microcontrollers. It is **not** a Python package — there is no pytest, linting pipeline,
or build system beyond the `esphome` CLI.

---

## Repository Structure

```
packages/      # Reusable hardware-specific ESPHome config blocks (the source of truth)
examples/      # Minimal copy-paste starter configs for end users (reference remote packages)
devices/       # Real production device configs with secrets — GITIGNORED, never commit
tests/         # CI validation wrappers (use local !include to test in-repo packages)
docs/          # Hardware-specific guides and usage documentation
.agents/       # AI agent skill definitions — GITIGNORED
```

---

## Validation / Test Commands

ESPHome YAML validation is the equivalent of running tests:

```bash
# Install ESPHome
pip install esphome

# Validate a single test file (run a single "test")
esphome config tests/test_s31.yaml

# Validate any package or config file
esphome config packages/s31_base.yaml

# Compile (more thorough — requires toolchain)
esphome compile tests/test_s31.yaml

# Validate all CI-covered files
esphome config tests/test_s31.yaml
esphome config tests/test_ifan04.yaml
esphome config tests/test_owon_xdm.yaml
esphome config tests/test_wopr.yaml
esphome config tests/test_rv_tv.yaml
esphome config tests/test_plant_bot.yaml
esphome config tests/test_bermuda.yaml
esphome config tests/test_mochi.yaml
esphome config tests/test_current_affairs.yaml
esphome config tests/test_ikea_vindriktning.yaml
```

CI runs `esphome config <file>` for each file in the matrix defined in
`.github/workflows/validate.yml`. There are no unit test functions — all tests are
whole-file YAML validation passes.

**Note:** `tests/test_anavi_fume_extractor.yaml` exists but is not in the CI matrix
(it references missing local paths and should not be relied upon).

---

## Four-Layer Architecture

Every supported device follows a strict layered composition pattern:

| Layer | Location | Purpose |
|-------|----------|---------|
| **Package** | `packages/*_base.yaml` | Hardware-only config. No credentials. Exposes `substitutions` with sensible defaults. |
| **Example** | `examples/*_example.yaml` | Minimal device config a user copies. References package via remote GitHub URL. |
| **Device** | `devices/*.yaml` | Production config with `!secret` references. Gitignored. Never commit. |
| **Test** | `tests/test_*.yaml` | CI wrapper using `!include ../packages/...` (local path) to test in-repo version. |

---

## YAML Code Style

### Indentation and Formatting

- **2-space indentation** throughout all YAML files
- **4-space indentation** inside C++ lambda blocks (`lambda: |-`)
- Use `|-` block scalar for multi-line C++ lambdas (strips trailing newline)
- Inline `#` comments for section headers and non-obvious decisions

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| ESPHome node name (`esphome.name`) | lowercase-hyphen | `my-plug`, `dryer`, `bedroom-fan` |
| `friendly_name` | Title Case with spaces | `"My Plug"`, `"Jacob's Bedroom"` |
| Entity `name:` within a package | Short Title Case, NOT device-prefixed | `"Relay"`, `"WiFi Signal"`, `"Daily Energy"` |
| Internal IDs (`id:`) | `snake_case` | `relay`, `fan_comp`, `my_power` |
| `substitutions` keys | `snake_case` | `relay_restore_mode`, `ct_factor` |
| Script IDs | `snake_case` | `fan_speed_set`, `initialize_multimeter` |
| MQTT topics | `lowercase/slash_separated` | `xdm1041/cmd`, `xdm1041/resp` |
| Project name | `org.device_model` | `"cyberglitchlabs.sonoff_s31"` |

### Entity Categories

Always set `entity_category` to reduce clutter in Home Assistant:

```yaml
entity_category: diagnostic  # Status, uptime, WiFi signal, IP, version, MAC
entity_category: config       # Restart button, safe mode button, tunable switches
# (omit entity_category for primary user-facing entities like Relay, Power, etc.)
```

### Sensor Attributes

Every sensor must include all applicable metadata:

```yaml
sensor:
  - platform: cse7766
    power:
      name: "Power"
      accuracy_decimals: 2
      device_class: power          # Always set device_class
      state_class: measurement     # measurement | total_increasing | total
      unit_of_measurement: "W"     # Required for numeric sensors
```

- Use `state_class: measurement` for instantaneous values
- Use `state_class: total_increasing` for cumulative/energy values
- Use `filters: throttle_average:` to reduce HA state spam for high-frequency sensors

### Package File Header

Every package must begin with a comment block documenting requirements:

```yaml
# DeviceName Hardware Package
# This package provides hardware-specific configuration only.
# Users must define esphome, wifi, api, ota, and time in their device config.
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
#       files: [packages/xxx_base.yaml]
#       ref: main
```

### Substitutions

Packages declare substitutions with sensible defaults. Device configs override before the
`packages:` key:

```yaml
# In device yaml — overrides package default
substitutions:
  relay_restore_mode: "RESTORE_DEFAULT_ON"

packages:
  device:
    url: https://github.com/cyberglitchlabs/esphome
    files: [packages/s31_base.yaml]
    ref: main
```

---

## Required Platform Settings

All packages must set:

```yaml
api:
  reboot_timeout: 0s    # Prevents device reboot when HA connection is lost

logger:
  baud_rate: 0          # Required when UART is used for sensors (prevents conflicts)
```

---

## Standard Diagnostic Block

Every package must include the following diagnostic entities (copy verbatim, adjust
`entity_category` only if needed):

```yaml
binary_sensor:
  - platform: status
    name: "Status"
    entity_category: diagnostic

sensor:
  - platform: wifi_signal
    name: "WiFi Signal"
    update_interval: 60s
    entity_category: diagnostic
  - platform: uptime
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

---

## C++ Lambda Style

For inline lambdas use `lambda: |-` with 4-space indentation:

```yaml
on_turn_on:
  - lambda: |-
      std::string cmd = "CONF:MEAS:VOLT:AC\n";
      id(uart_bus).write_str(cmd.c_str());
      ESP_LOGI("xdm", "Sent command: %s", cmd.c_str());
```

- Use `id(component_id)` to reference ESPHome components
- Use `ESP_LOGI` / `ESP_LOGW` / `ESP_LOGE` for logging (not `printf`)
- Use `std::string` for string manipulation
- Terminate all statements with semicolons

---

## Script Modes

```yaml
script:
  - id: fan_speed_set
    mode: restart   # Cancel and restart if re-triggered mid-execution
  - id: initialize_multimeter
    mode: single    # Ignore re-triggers while running
```

---

## Python Utility Scripts

The only Python in this repo is `scripts/gen_esphome_noise_psk.py`. Follow this style:

- Type annotations on all function signatures
- `if __name__ == "__main__": raise SystemExit(main())` pattern
- `argparse` for CLI argument parsing
- Standard library only — no third-party dependencies

---

## Secrets and Security

- **Never commit** `secrets.yaml`, `devices/`, or `.agents/` (all gitignored)
- All production credentials use `!secret key_name` references
- Generate new noise_psk keys with: `python3 .agents/skills/home-assistant-esphome/scripts/gen_esphome_noise_psk.py --yaml`
- API encryption key and OTA password must be set per-device, never in packages

---

## Adding a New Device Package

1. Create `packages/<device>_base.yaml` with hardware-only config + header comment
2. Create `examples/<device>_example.yaml` referencing the package via remote GitHub URL
3. Create `tests/test_<device>.yaml` using `!include ../packages/<device>_base.yaml`
4. Add the test file to `.github/workflows/validate.yml` matrix
5. Create `docs/<device>.md` with hardware notes and wiring details
