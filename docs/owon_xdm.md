# OWON XDM Remote

## Description

ESP32-C3 based SCPI bridge for OWON XDM bench multimeters (tested on XDM1041). Enables remote control and measurement retrieval via ESPHome API or optional MQTT integration for Home Assistant and Node-RED.

## Features

* SCPI command execution via ESPHome API or MQTT (optional).
* Fast sampling mode auto-configuration on startup.
* WiFi quality and heartbeat monitoring (with MQTT).
* Device status tracking (online/offline, with MQTT).

## Configuration

### Basic (API Only)

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/owon_xdm.yaml
    ref: main
```

### With MQTT Support

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/owon_xdm_with_mqtt.yaml
    ref: main
```

When using MQTT, define your broker details in `secrets.yaml`.

## MQTT Topics

* `xdm1041/cmd` - Send SCPI commands (publish)
* `xdm1041/resp` - Receive SCPI responses (subscribe)
* `xdm1041/status` - Device status: online/offline (subscribe, retained)
* `xdm1041/heartbeat` - Heartbeat every 60s (subscribe)
* `xdm1041/wifiquality` - WiFi quality 0-100% (subscribe, retained)

## SCPI Examples

* `MEAS1?` - Query primary measurement.
* `FUNC VOLT` - Set function to DC voltage.
* `CONF:VOLT:RANG 10` - Set voltage range to 10V.

## Credits

This ESPHome port is based on the [OWON XDM Remote MicroPython project](https://github.com/Elektroarzt/owon-xdm-remote) by Elektroarzt.
