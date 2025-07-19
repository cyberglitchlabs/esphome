# ESPHome Multi-Device Project

## Structure

- `devices/` — Individual device YAMLs (per board/model)
- `templates/` — Board templates and factory configs
- `shared/` — Common sensors, packages, automations, templates
- `_template/` — Setup checklist and documentation
- `static/` — Site config and docs
- `README.md` — Main documentation

## Usage

- Add new device configs to `devices/`
- Add new board templates to `templates/`
- Place shared configs in `shared/` and use `!include` in device YAMLs

See `_template/setup-checklist.md` for setup steps.
