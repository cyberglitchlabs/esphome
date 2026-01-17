# Current Affairs (HackerBox 0120)

## Description

8-channel AC current monitoring system based on [HackerBox 0120: Current Affairs](https://hackerboxes.com/collections/past-hackerboxes/products/hackerbox-0120-current-affairs). Uses four ADS1115 16-bit ADCs to measure differential voltage from current transformers (CTs).

## Features

* 8 high-precision AC current monitoring channels.
* Native ESPHome `ct_clamp` integration for RMS calculation.
* 860 samples per second per channel.
* Configurable current transformation factors.
* **Energy Dashboard Ready**: Automatically calculates Power (W) and Daily Energy (kWh).

## Configuration

### Basic Usage

Include the package in your local configuration:

```yaml
packages:
  remote_package:
    url: github://cyberglitchlabs/esphome/packages/current_affairs.yaml
    ref: main
```

### Nominal Voltage

By default, the package assumes **120V** for power calculations. You can override this in your `substitutions` block:

```yaml
substitutions:
  rms_voltage: "230"  # Set to your local mains voltage
```

## Measuring 240V Circuits (US Split-Phase)

In a US split-phase system, 240V circuits use two 120V legs. For the most accurate measurement (especially with unbalanced loads like dryers), use one CT on each leg and sum them.

If you use **J1** and **J2** for a single 240V breaker, add this to your local YAML to create a combined energy entity:

```yaml
sensor:
  - platform: template
    name: "Dryer Power"
    id: dryer_power
    unit_of_measurement: "W"
    device_class: power
    state_class: measurement
    lambda: return (id(j1_current).state + id(j2_current).state) * ${rms_voltage};
    update_interval: ${update_interval}

  - platform: total_daily_energy
    name: "Dryer Energy"
    sensor: dryer_power
    unit_of_measurement: "kWh"
    device_class: energy
    state_class: total_increasing
    filters:
      - multiply: 0.001
```

## Pin Configuration

* I2C SDA: GPIO4
* I2C SCL: GPIO5
* ADS1115 Addresses: 0x48, 0x49, 0x4A, 0x4B

## Credits

Inspired by the HackerBox 0120 "Current Affairs" project.
