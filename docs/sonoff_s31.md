# Sonoff S31

## Description

The Sonoff S31 is a compact US-plug Wi-Fi smart plug with power monitoring. This configuration utilizes the internal CSE7766 power monitoring chip.

## Features

* Remote relay control.
* Real-time power, voltage, and current monitoring.
* Native integration with Home Assistant via the ESPHome API.
* Optimized for stability on the ESP12E chip.

## Configuration

### Usage

Include the base package in your local configuration:

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/s31_base.yaml
    ref: main
```

## Hardware Notes

* **Chip**: ESP8266 (ESP12E).
* **Power Monitor**: CSE7766.
* **Internal Pins**:
  * Relay: GPIO12
  * Button: GPIO0
  * LED: GPIO13
