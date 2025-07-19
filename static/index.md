
# ESPHome Multi-Device Project

Welcome to the CyberGlitchLabs ESPHome repository!

## Features
- Modular configuration for multiple devices (ANAVI Fume Extractor, Sonoff S31, and more)
- Shared sensor, output, and automation packages for easy reuse
- Versioning and substitutions for maintainable configs
- Automatic documentation publishing via GitHub Pages
- Continuous Integration for YAML linting and ESPHome config validation

## Usage: Import into ESPHome Dashboard
This project is designed to be imported into your ESPHome dashboard. You do not need to clone the repo directly.

### Import Instructions
1. In your ESPHome dashboard, click **"Import"**.
2. Paste the URL for your device:
   - **ANAVI Fume Extractor:**
     ```
     github://cyberglitchlabs/esphome/devices/anavi-fume-extractor-esp32-s3.yaml@main
     ```
   - **Sonoff S31:**
     ```
     github://cyberglitchlabs/esphome/devices/sonoff-s31.yaml@main
     ```
   - **Other templates:**
     ```
     github://cyberglitchlabs/esphome/devices/project-template-esp32.yaml@main
     github://cyberglitchlabs/esphome/devices/project-template-esp32-c3.yaml@main
     github://cyberglitchlabs/esphome/devices/project-template-esp32-c6.yaml@main
     github://cyberglitchlabs/esphome/devices/project-template-esp32-s3.yaml@main
     ```
3. Follow the prompts to configure and install the firmware for your device.

All device YAMLs are in the `devices/` folder. You can import any supported device using its GitHub URL.

## Modular Structure
- Device configs in `devices/`
- Shared packages in `shared/`
- Templates in `templates/`
- Documentation in `static/`

## Documentation
- Published from the `static/` folder
- Main site: [GitHub Pages](https://cyberglitchlabs.github.io/esphome/)
- Add docs by creating Markdown files in `static/` (e.g., `hardware.md`, `setup.md`, `faq.md`)

## Continuous Integration
- All YAML files are linted and validated on every pull request and nightly
- ESPHome config validation runs on all device YAMLs in `devices/`

## Example Device
See [`devices/anavi-fume-extractor-esp32-s3.yaml`](../devices/anavi-fume-extractor-esp32-s3.yaml) for a complete example.

## Contributing
- Fork and submit pull requests for improvements
- Open issues for bugs or feature requests

---

For more info, see the [ESPHome documentation](https://esphome.io/).
