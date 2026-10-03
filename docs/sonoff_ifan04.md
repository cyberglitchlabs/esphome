# Sonoff iFan04

## Description

The Sonoff iFan04 is a Wi-Fi ceiling fan and light controller. It includes a 433MHz RF remote for local control.

## Features

* 3-speed fan control.
* Light toggle control.
* Native RF remote support.
* Buzzer feedback for speed changes.
* `RC Command` event entity: every RF remote button press is forwarded to Home Assistant
  (`light_toggle`, `mute_toggle`, `fan_off`, `fan_low`, `fan_mid`, `fan_high`, `rf_wifi`,
  `wifi_long`, `rf_long`) for use as an automation trigger.

## Configuration

### Usage

Include the base package in your local configuration:

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/ifan04_base.yaml
    ref: main
```

### Migration note

The former `RC button ...` binary sensors were replaced by the single `RC Command` event
entity. Update any Home Assistant automations that triggered on those binary sensors.

### Security

The example config enables API encryption and encrypted OTA (reusing the API key) and sets a
fallback AP password. The API key is deliberately shared with OTA, so only one key is needed per device. Use a different key on every device. The example expects `wifi_fallback_password` and `api_encryption_key` in `secrets.yaml`, and encrypted OTA needs ESPHome 2026.9.0 or newer.

## Hardware Notes

* **Chip**: ESP8285.
* **Pins**:
  * Fan Speed 1: GPIO14
  * Fan Speed 2: GPIO12
  * Fan Speed 3: GPIO15
  * Light: GPIO9
  * Buzzer: GPIO10
  * RF Receiver: GPIO3
