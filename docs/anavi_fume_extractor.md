# ANAVI Fume Extractor

## Description

ESP8266-based smart solder smoke absorber with Wi-Fi connectivity. Features an 80mm fan, mini OLED display, MQ-135 air quality sensor, and support for multiple I2C environmental sensors.

## Features

* 80mm fan control with physical button and relay.
* MQ-135 analog gas sensor for air quality monitoring.
* Mini OLED display (128x64) showing status and environmental data.
* Automatic discovery in Home Assistant.

## Configuration

### Usage

Include the base package in your local configuration:

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/anavi_fume_extractor.yaml
    ref: main
```

## I2C Sensor Support

* **HTU21D** (Temperature & Humidity) at 0x40.
* **BH1750** (Light) at 0x23.
* **BMP180** (Pressure & Temperature) at 0x77.
* **APDS9960** (Color, Gesture, Proximity) at 0x39.

## Air Quality Thresholds (MQ-135)

* Good: ≤18%
* Moderate: 19-36%
* Unhealthy: 37-54%
* Very Unhealthy: 55-72%
* Hazardous: >72%

## Credits

Based on the [ANAVI Fume Extractor](https://github.com/AnaviTechnology/anavi-fume-extractor-sw) open source hardware project.
