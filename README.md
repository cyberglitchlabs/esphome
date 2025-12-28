# ESPHome Configurations

This repository contains modular ESPHome configurations for my personal projects.

## Structure

*   `packages/`: Contains reusable configuration blocks.
    *   `core.yaml`: Common network and API settings.
    *   `sonoff_s31.yaml`: Hardware definition for Sonoff S31.
    *   `s31_base.yaml`: Bundle including both core and device configs for S31.
    *   `sonoff_ifan04.yaml`: Hardware definition for Sonoff iFan04 (Fan + Light + Remote).
    *   `ifan04_base.yaml`: Bundle including both core and device configs for iFan04.
    *   `owon_xdm_base.yaml`: Hardware definition for OWON XDM Remote (ESP32-C3 SCPI bridge).
    *   `owon_xdm_mqtt.yaml`: Optional MQTT integration for OWON XDM Remote.
    *   `owon_xdm.yaml`: Bundle with core and device configs (no MQTT).
    *   `owon_xdm_with_mqtt.yaml`: Bundle with core, device, and MQTT configs.
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

| Device | Package | Base YAML | MQTT Support |
| :--- | :--- | :--- | :--- |
| **Sonoff S31** | `packages/sonoff_s31.yaml` | `packages/s31_base.yaml` | N/A |
| **Sonoff iFan04** | `packages/sonoff_ifan04.yaml` | `packages/ifan04_base.yaml` | N/A |
| **OWON XDM Remote** | `packages/owon_xdm_base.yaml` | `packages/owon_xdm.yaml` | `packages/owon_xdm_with_mqtt.yaml` |

## Device-Specific Documentation

### OWON XDM Remote

**Description:** ESP32-C3 based SCPI bridge for OWON XDM bench multimeters (tested on XDM1041). Enables remote control and measurement retrieval via ESPHome API or optional MQTT integration for Home Assistant and Node-RED.

**Features:**
- SCPI command execution via ESPHome API or MQTT (optional)
- Fast sampling mode auto-configuration on startup
- WiFi quality and heartbeat monitoring (with MQTT)
- Device status tracking (online/offline, with MQTT)
- ESP32-C3 Super Mini Plus compatible

**Hardware Requirements:**
- ESP32-C3 Super Mini Plus (with external antenna recommended)
- Custom PCB replacing the original UART module (optional but recommended)
- See [original project](https://github.com/Elektroarzt/owon-xdm-remote) for PCB files and full hardware details

**Basic Configuration (ESPHome API only):**
```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/owon_xdm.yaml
    ref: main
```

**With MQTT Support:**
```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/owon_xdm_with_mqtt.yaml
    ref: main
```

When using MQTT, add these credentials to your `secrets.yaml`:
```yaml
mqtt_broker: "your.mqtt.broker.ip"
mqtt_port: 1883
mqtt_user: "your_mqtt_username"
mqtt_password: "your_mqtt_password"
```

**MQTT Topics** (when MQTT package is included):
- `xdm1041/cmd` - Send SCPI commands (publish)
- `xdm1041/resp` - Receive SCPI responses (subscribe)
- `xdm1041/status` - Device status: online/offline (subscribe, retained)
- `xdm1041/heartbeat` - Heartbeat every 60s (subscribe)
- `xdm1041/wifiquality` - WiFi quality 0-100% (subscribe, retained)

**SCPI Command Examples:**
```yaml
# Query measurement
MEAS1?

# Set function to DC voltage
FUNC VOLT

# Set range
CONF:VOLT:RANG 10
```

**Credits:** This ESPHome port is based on the [OWON XDM Remote MicroPython project](https://github.com/Elektroarzt/owon-xdm-remote) by Elektroarzt.

## Installation Notes

When you fork this repository, remember to update the `dashboard_import` URL in `packages/s31_base.yaml` and `packages/ifan04_base.yaml` to point to your own GitHub repository URL so that dashboard updates work correctly for your custom version.
