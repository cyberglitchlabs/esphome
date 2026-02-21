# Plant-Bot

Plant-Bot is an ESP32-C3-based smart plant watering device by [elektroThing](https://www.tindie.com/stores/elektrothing/), available on Tindie.

## Hardware

| Component | Part | Details |
|-----------|------|---------|
| MCU | ESP32-C3-MINI | WiFi 802.11b/g/n, BLE 5.0, 4MB flash, CP2102 USB-UART |
| Temp/Humidity | SHT20 | ±0.3°C, ±3%RH, I2C address 0x40 |
| Pump driver | DRV8837CDSGR | H-bridge, 1.2A peak, powered from 5V/VUSB |
| LED | WS2812B | 1× NeoPixel |
| Moisture | Capacitive | Analog output, NE555 oscillator circuit |
| Power | 700mA 3.3V LDO | Micro USB |

## Pin Assignments

| GPIO | Function |
|------|----------|
| GPIO0 | Pump DRV8837C IN1 |
| GPIO1 | Pump DRV8837C IN2 |
| GPIO2 | Capacitive oscillator enable (active HIGH) |
| GPIO3 | Status LED (indicator) |
| GPIO4 | Soil moisture ADC input |
| GPIO10 | WS2812B NeoPixel data |
| GPIO18 | I2C SCL (SHT20) |
| GPIO19 | I2C SDA (SHT20) |

**Pump wiring**: IN1=HIGH, IN2=LOW drives the pump forward. The package always keeps IN2 LOW and controls the pump by toggling IN1 only.

## Moisture Sensor Calibration

The capacitive moisture sensor outputs a raw ADC value. Two calibration points are needed:

1. **Dry reading** (`moisture_raw_max`): Hold the probe in open air, note the ADC value (typically ~2700)
2. **Wet reading** (`moisture_raw_min`): Submerge the probe in water up to the fill line, note the ADC value (typically ~1800)

Set these in your device config:

```yaml
substitutions:
  moisture_raw_min: "1800"  # raw ADC in water (100%)
  moisture_raw_max: "2700"  # raw ADC in dry air (0%)
```

Do not submerge the PCB — only the probe section below the soil line should contact water.

## Pump Duration

The `pump_duration` substitution controls how long the pump runs each time "Water Now" is pressed. The pump will always stop automatically after this duration.

```yaml
substitutions:
  pump_duration: "5s"  # Default: 5 seconds per watering cycle
```

## Home Assistant Entities

| Entity | Type | Description |
|--------|------|-------------|
| Temperature | Sensor | SHT20 temperature in °C |
| Humidity | Sensor | SHT20 relative humidity % |
| Soil Moisture | Sensor | Calibrated moisture 0–100% |
| LED | Light | WS2812B NeoPixel (RGB) |
| Water Now | Button | Triggers a timed pump cycle |
| Status | Binary Sensor | Connection status |
| WiFi Signal | Sensor | RSSI in dBm |
| Uptime | Sensor | Device uptime in seconds |
| Restart | Button | Reboot device |
| Safe Mode | Button | Boot into safe mode |
| ESPHome Version | Text Sensor | Firmware version |
| WiFi IP / SSID / MAC | Text Sensor | Network diagnostics |

## Flashing

The Plant-Bot uses a CH340 USB-UART chip. Flash via USB with:

```bash
esphome run devices/my-plant-bot.yaml
```

Or compile and upload via the ESPHome dashboard.

## Home Assistant Integration with Plant Monitor + OpenPlantbook

For a full plant-care experience, combine Plant-Bot with two HACS integrations:

```
Plant-Bot (ESPHome)  →  Plant Monitor (HACS)  →  OpenPlantbook (HACS)
  sensor readings          plant health                species thresholds
```

### Step 1 — Install HACS integrations

Install both via HACS → Integrations:

- **[Plant Monitor](https://github.com/Olen/homeassistant-plant)** — creates a plant "device" in HA, tracks sensor health against thresholds
- **[OpenPlantbook](https://github.com/Olen/home-assistant-openplantbook)** — fetches species-specific care thresholds from [open.plantbook.io](https://open.plantbook.io)

OpenPlantbook requires a free API account at open.plantbook.io. Note your `client_id` and `secret` during setup.

### Step 2 — Add a plant in Plant Monitor

1. **Settings → Devices & Services → Add Integration → Plant Monitor**
2. Enter a name (e.g. `Monstera`)
3. Search for your species — OpenPlantbook will auto-fill min/max thresholds for moisture, temperature, and humidity
4. Assign your Plant-Bot sensors:
   - **Moisture** → `sensor.<your_device>_soil_moisture`
   - **Temperature** → `sensor.<your_device>_temperature`
   - **Humidity** → `sensor.<your_device>_humidity`
   - Conductivity and illuminance can be left unassigned (Plant-Bot has neither)

Plant Monitor creates a `plant.<your_plant_name>` entity with state `ok` or `problem`.

### Step 3 — Auto-watering automation

Wire the plant health state to the "Water Now" button:

```yaml
automation:
  - alias: "Water plant when soil is too dry"
    trigger:
      - platform: state
        entity_id: plant.monstera
        to: "problem"
    condition:
      # Only water if moisture is specifically the problem
      - condition: template
        value_template: >
          {{ state_attr('plant.monstera', 'moisture') == 'Low' }}
      # Avoid watering at night
      - condition: time
        after: "07:00:00"
        before: "22:00:00"
    action:
      - service: button.press
        target:
          entity_id: button.my_plant_bot_water_now
```

### Step 4 — Lovelace Flower Card (optional)

Install the **[Flower Card](https://github.com/Olen/lovelace-flower-card)** via HACS → Frontend for a purpose-built plant status card:

```yaml
type: custom:flower-card
entity: plant.monstera
show_bars:
  - moisture
  - temperature
  - humidity
```

### What OpenPlantbook provides automatically

When you search for a species, the following thresholds are fetched and set on your plant entity:

| Threshold | Example (Monstera deliciosa) |
|-----------|------------------------------|
| Min soil moisture | 20% |
| Max soil moisture | 60% |
| Min temperature | 15°C |
| Max temperature | 30°C |
| Min humidity | 40% |
| Max humidity | 80% |

These become individual HA entities and can be adjusted at any time from the UI.

### Note on missing sensors

Plant Monitor also supports **conductivity** (soil nutrients) and **illuminance** sensors.
Plant-Bot does not include these. You can leave them unassigned — Plant Monitor simply
disables those thresholds and does not count them toward the `problem` state.

## NeoPixel LED as Plant Health Indicator

The WS2812B NeoPixel (`light.my_plant_bot_led`) can mirror the Plant Monitor health
state, giving you an at-a-glance status without opening the HA app.

### Option 1 — Status color (with Plant Monitor)

Green = healthy, yellow = warning (approaching threshold), red = problem.

```yaml
automation:
  - alias: "Plant Bot LED — health status color"
    mode: restart
    trigger:
      - platform: state
        entity_id: plant.monstera
      - platform: state
        entity_id: sensor.my_plant_bot_soil_moisture
    action:
      - choose:
          # Problem — red
          - conditions:
              - condition: state
                entity_id: plant.monstera
                state: "problem"
            sequence:
              - service: light.turn_on
                target:
                  entity_id: light.my_plant_bot_led
                data:
                  rgb_color: [255, 0, 0]
                  brightness: 128
          # Getting dry (below 40%) — yellow
          - conditions:
              - condition: numeric_state
                entity_id: sensor.my_plant_bot_soil_moisture
                below: 40
            sequence:
              - service: light.turn_on
                target:
                  entity_id: light.my_plant_bot_led
                data:
                  rgb_color: [255, 180, 0]
                  brightness: 80
        # Default — green (healthy)
        default:
          - service: light.turn_on
            target:
              entity_id: light.my_plant_bot_led
            data:
              rgb_color: [0, 200, 0]
              brightness: 50
```

### Option 2 — Moisture gradient color (no Plant Monitor required)

Maps soil moisture 0–100% to a red→yellow→blue gradient:
dry (red) → moderate (yellow) → wet (blue).

```yaml
automation:
  - alias: "Plant Bot LED — moisture gradient"
    mode: restart
    trigger:
      - platform: state
        entity_id: sensor.my_plant_bot_soil_moisture
    action:
      - service: light.turn_on
        target:
          entity_id: light.my_plant_bot_led
        data:
          # Template blends red→yellow at 0–50%, yellow→blue at 50–100%
          rgb_color: >
            {% set m = states('sensor.my_plant_bot_soil_moisture') | float(0) %}
            {% if m <= 50 %}
              {% set r = 255 %}
              {% set g = (m / 50 * 180) | int %}
              {% set b = 0 %}
            {% else %}
              {% set r = ((100 - m) / 50 * 255) | int %}
              {% set g = ((100 - m) / 50 * 180) | int %}
              {% set b = ((m - 50) / 50 * 255) | int %}
            {% endif %}
            [{{ r }}, {{ g }}, {{ b }}]
          brightness: 80
```

### Option 3 — Pulse when watering

Flash white while the pump is running, then return to the status color.
Trigger off the `script.pump_timed_run` state:

```yaml
automation:
  - alias: "Plant Bot LED — pulse white while watering"
    mode: single
    trigger:
      - platform: state
        entity_id: script.my_plant_bot_pump_timed_run
        to: "on"
    action:
      - repeat:
          while:
            - condition: state
              entity_id: script.my_plant_bot_pump_timed_run
              state: "on"
          sequence:
            - service: light.turn_on
              target:
                entity_id: light.my_plant_bot_led
              data:
                rgb_color: [0, 100, 255]
                brightness: 255
            - delay: "00:00:00.4"
            - service: light.turn_on
              target:
                entity_id: light.my_plant_bot_led
              data:
                brightness: 20
            - delay: "00:00:00.4"
```

> **Tip:** Combine Options 1 and 3 by restoring the status color after the pump script finishes — add a second trigger on `script.my_plant_bot_pump_timed_run` transitioning `to: "off"` that calls the status color automation.

---

## Simple automation (without Plant Monitor)

If you prefer not to use Plant Monitor, a basic moisture-threshold automation works too:

```yaml
automation:
  - alias: "Auto Water Plant"
    trigger:
      - platform: numeric_state
        entity_id: sensor.my_plant_bot_soil_moisture
        below: 30
    action:
      - service: button.press
        target:
          entity_id: button.my_plant_bot_water_now
```
