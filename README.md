# ESPHome Configurations

This repository contains modular ESPHome configurations for my personal projects.

## Structure

* `packages/`: Contains reusable configuration blocks.
* `examples/`: Example configurations for local usage.
* `docs/`: Detailed hardware and setup guides for each device.

## Usage

### Remote Usage (Recommended)

You can use these configurations in your local ESPHome instance without cloning the repo, by using the `remote_package` feature.

**Example `living_room.yaml`:**

```yaml
substitutions:
  name: "living-room-s31"
  friendly_name: "Living Room S31"
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password

esphome:
  name: ${name}
  friendly_name: ${friendly_name}

esp8266:
  board: esp01_1m

packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/s31_base.yaml
    ref: main
```

### Available Devices

| Device | Documentation | Package |
| :--- | :--- | :--- |
| **Sonoff S31** | [Docs](docs/sonoff_s31.md) | `packages/s31_base.yaml` |
| **Sonoff iFan04** | [Docs](docs/sonoff_ifan04.md) | `packages/ifan04_base.yaml` |
| **OWON XDM Remote** | [Docs](docs/owon_xdm.md) | `packages/owon_xdm.yaml` |
| **ANAVI Fume Extractor** | [Docs](docs/anavi_fume_extractor.md) | `packages/anavi_fume_extractor.yaml` |
| **Current Affairs** | [Docs](docs/current_affairs.md) | `packages/current_affairs.yaml` |
| **IKEA VINDRIKTNING** | [Docs](docs/ikea_vindriktning.md) | `packages/ikea_vindriktning.yaml` |

## Installation Notes

When you fork this repository, remember to update the `dashboard_import` URL in your packages to point to your own GitHub repository URL so that dashboard updates work correctly for your custom version.
