# Sonoff iFan04

## Description

The Sonoff iFan04 is a Wi-Fi ceiling fan and light controller. It includes a 433MHz RF remote for local control.

## Features

* 3-speed fan control.
* Light toggle control.
* Native RF remote support.
* Buzzer feedback for speed changes.

## Configuration

### Usage

Include the base package in your local configuration:

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/ifan04_base.yaml
    ref: main
```

## Hardware Notes

* **Chip**: ESP8285.
* **Pins**:
  * Fan Speed 1: GPIO14
  * Fan Speed 2: GPIO12
  * Fan Speed 3: GPIO15
  * Light: GPIO9
  * Buzzer: GPIO10
  * RF Receiver: GPIO3
