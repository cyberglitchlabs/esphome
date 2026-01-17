# IKEA VINDRIKTNING

## Description

The IKEA VINDRIKTNING is a budget-friendly particulate matter (PM2.5) sensor. It uses an internal PM1006 sensor. By adding an ESP8266 (like a Wemos D1 Mini or an ESP-01), you can integrate it into Home Assistant via ESPHome.

## Features

* Real-time PM2.5 monitoring.
* Native Home Assistant integration.
* Non-destructive mod (can be hidden inside the case).

## Configuration

### Usage

Include the base package in your local configuration:

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/ikea_vindriktning.yaml
    ref: main
```

## Hardware Notes

* **Sensor**: PM1006.
* **Communication**: UART (9600 baud).
* **Wiring**:
  * **IKEA PCB (REST)** -> **ESP8266 RX (GPIO3)**
  * **IKEA PCB (5V)** -> **ESP8266 5V/VIN**
  * **IKEA PCB (GND)** -> **ESP8266 GND**

> [!IMPORTANT]
> You only need to connect the RX pin because the ESP8266 only listens to the data transmitted by the IKEA sensor's internal controller.

## Credits

Based on the [ESPHome PM1006 Component](https://esphome.io/components/sensor/pm1006.html).
