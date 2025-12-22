# ESPHome Configurations

This repository contains modular ESPHome configurations for my personal projects.

## Structure

*   `packages/`: Contains reusable configuration blocks.
    *   `core.yaml`: Common network and API settings.
    *   `sonoff_s31.yaml`: Hardware definition for Sonoff S31.
    *   `s31_base.yaml`: Bundle including both core and device configs for S31.
    *   `sonoff_ifan04.yaml`: Hardware definition for Sonoff iFan04 (Fan + Light + Remote).
    *   `ifan04_base.yaml`: Bundle including both core and device configs for iFan04.
*   `examples/`: Example configurations for local usage.

## Usage

### Remote Usage (Recommended)

You can use these configurations in your local ESPHome instance without cloning the repo, by using the `remote_package` feature.

1.  Create a local YAML file (e.g., `living_room.yaml`).
2.  Define your substitutions (name, wifi).
3.  Include the remote package.

**Example `living_room.yaml`:**

```yaml
substitutions:
  name: "living-room-s31"
  friendly_name: "Living Room S31"
  wifi_ssid: !secret wifi_ssid  # Ensure you have a secrets.yaml locally
  wifi_password: !secret wifi_password

esphome:
  name: ${name}
  friendly_name: ${friendly_name}

packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/s31_base.yaml
    ref: main
```

**Note:** Ensure you replace `cyberglitchlabs/esphome` with the actual path to this repository.

### Secrets

This repository does **not** contain a `secrets.yaml` file.
*   The `core.yaml` package defines `wifi_ssid` and `wifi_password` substitutions.
*   You must provide values for these in your top-level configuration's `substitutions` block.
*   You can pass raw strings or reference your own local `!secret` definitions.

### Available Devices

| Device | Package | Base YAML |
| :--- | :--- | :--- |
| **Sonoff S31** | `packages/sonoff_s31.yaml` | `packages/s31_base.yaml` |
| **Sonoff iFan04** | `packages/sonoff_ifan04.yaml` | `packages/ifan04_base.yaml` |

## Installation Notes

When you fork this repository, remember to update the `dashboard_import` URL in `packages/s31_base.yaml` and `packages/ifan04_base.yaml` to point to your own GitHub repository URL so that dashboard updates work correctly for your custom version.
