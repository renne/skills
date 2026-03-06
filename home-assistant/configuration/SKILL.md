---
name: configuration
description: Home Assistant configuration covering configuration.yaml structure, splitting config with !include, secrets management, packages, and validating YAML. Use when setting up or restructuring a Home Assistant configuration, managing sensitive credentials, or organizing large configurations.
---
# Home Assistant Configuration

Source: https://www.home-assistant.io/docs/configuration/

## Overview

Home Assistant's configuration lives in the **config directory** (typically `/config/` in Home Assistant OS/Supervised or `~/.homeassistant/` in standalone installs). The main entry point is `configuration.yaml`.

---

## configuration.yaml Structure

```yaml
# Core settings
homeassistant:
  name: "My Home"
  latitude: 52.3731
  longitude: 4.8922
  elevation: 5
  unit_system: metric           # metric | us_customary
  currency: USD
  country: US
  time_zone: America/New_York
  packages: !include_dir_named packages

# HTTP settings
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.30.33.0/24

# Integrations configured via YAML
mqtt:
  broker: 192.168.1.100
  username: homeassistant
  password: !secret mqtt_password

# Split large sections into separate files
automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml
```

> Many integrations are now configured via the UI (Settings → Devices & Services) and do **not** require `configuration.yaml` entries.

---

## Secrets Management

Never hard-code passwords or API keys. Use `secrets.yaml` instead.

### secrets.yaml

```yaml
# /config/secrets.yaml
mqtt_password: "my_mqtt_password"
db_url: "postgresql://user:pass@localhost/ha_db"
google_api_key: "AIzaSyXXXXXXXXXXXXXXXXXXX"
latitude: 52.3731
longitude: 4.8922
```

### Referencing a secret

```yaml
# configuration.yaml
mqtt:
  password: !secret mqtt_password

homeassistant:
  latitude: !secret latitude
  longitude: !secret longitude
```

- `secrets.yaml` must reside in the same directory as `configuration.yaml`.
- Never commit `secrets.yaml` to version control — add it to `.gitignore`.

---

## Splitting Configuration with !include

Keep `configuration.yaml` tidy by splitting large sections into their own files.

### Include a single file

```yaml
automation: !include automations.yaml
```

### Include all YAML files in a directory as a list

```yaml
sensor: !include_dir_merge_list sensors/
```

### Include all YAML files in a directory as a named mapping

```yaml
homeassistant:
  packages: !include_dir_named packages/
```

### Include a directory merging all mappings

```yaml
script: !include_dir_merge_named scripts/
```

### Comparison of !include variants

| Directive | Returns | Use case |
|-----------|---------|----------|
| `!include file.yaml` | Contents of file | Single-file inclusion |
| `!include_dir_list dir/` | List of all file contents | `sensor:`, `binary_sensor:` lists |
| `!include_dir_named dir/` | Dict of filename → contents | `packages:` |
| `!include_dir_merge_list dir/` | Merged list from all files | `automation:`, `script:` lists |
| `!include_dir_merge_named dir/` | Merged dict from all files | `script:` as a mapping |

---

## Recommended Directory Structure

```
/config/
├── configuration.yaml
├── secrets.yaml
├── automations.yaml
├── scripts.yaml
├── scenes.yaml
├── packages/
│   ├── living_room.yaml
│   ├── kitchen.yaml
│   └── security.yaml
├── sensors/
│   ├── weather.yaml
│   └── energy.yaml
├── custom_components/
└── www/
```

---

## Packages

Packages let you group all related configuration (sensors, automations, scripts) for a feature or room into a single file.

```yaml
# configuration.yaml
homeassistant:
  packages: !include_dir_named packages/
```

```yaml
# packages/security.yaml
input_boolean:
  alarm_enabled:
    name: Alarm Enabled
    initial: true
    icon: mdi:alarm-light

binary_sensor:
  - platform: template
    sensors:
      any_door_open:
        value_template: >
          {{ is_state('binary_sensor.front_door', 'on')
             or is_state('binary_sensor.back_door', 'on') }}

automation:
  - alias: "Trigger alarm when armed and door opens"
    triggers:
      - trigger: state
        entity_id: binary_sensor.any_door_open
        to: "on"
    conditions:
      - condition: state
        entity_id: input_boolean.alarm_enabled
        state: "on"
    actions:
      - action: alarm_control_panel.alarm_trigger
        target:
          entity_id: alarm_control_panel.home
```

---

## Common Integration YAML Snippets

### Template sensor

```yaml
template:
  - sensor:
      - name: "Feels Like Temperature"
        unit_of_measurement: "°C"
        state: "{{ states('sensor.outside_temp') | float | round(1) }}"
```

### Input helpers (helpers defined in YAML)

```yaml
input_boolean:
  guest_mode:
    name: Guest Mode
    icon: mdi:account-multiple

input_number:
  target_temperature:
    name: Target Temperature
    min: 15
    max: 30
    step: 0.5
    unit_of_measurement: "°C"

input_select:
  home_mode:
    name: Home Mode
    options:
      - Home
      - Away
      - Night
      - Vacation
    initial: Home

input_text:
  custom_message:
    name: Custom Message
    max: 255
```

### Groups

```yaml
group:
  all_lights:
    name: All Lights
    entities:
      - light.living_room
      - light.kitchen
      - light.bedroom
```

### Notify platform

```yaml
notify:
  - name: family_phones
    platform: group
    services:
      - action: notify.mobile_app_alice
      - action: notify.mobile_app_bob
```

---

## Validating Configuration

Before restarting, always check for YAML errors:

1. **Developer Tools → YAML → Check Configuration** – validates syntax and integration config.
2. **Command line** (Home Assistant OS):
   ```bash
   ha core check
   ```
3. **Restart safely**: If validation passes, **Developer Tools → YAML → Restart** or `ha core restart`.

### Reload without restart

Many integrations support partial reload (no full restart needed):

- **Developer Tools → YAML → Reload Automations / Scripts / Scenes / Template Entities**
- Via service call: `homeassistant.reload_config_entry`, `automation.reload`, `script.reload`

---

## Common YAML Gotchas

- **Indentation**: Use spaces, never tabs. Two or four spaces consistently.
- **Colons in strings**: Wrap values containing `:` in quotes: `name: "Time: now"`.
- **Boolean values**: `true`/`false` (not `yes`/`no` for modern YAML).
- **Multiline strings**: Use `|` (literal block) or `>` (folded block):
  ```yaml
  message: |
    Line one
    Line two
  message: >
    This long sentence
    will be folded into one line.
  ```
- **Anchors and aliases** (YAML built-in):
  ```yaml
  common_target: &common_target
    area_id: living_room

  actions:
    - action: light.turn_on
      target: *common_target
  ```

---

## Tips

- Keep `configuration.yaml` minimal — use it mainly for `!include` directives and core settings.
- Store all credentials in `secrets.yaml`; never in files you track with version control.
- Use packages to co-locate everything belonging to one feature/room in a single file.
- After editing YAML files, always run **Check Configuration** before restarting.

## References

- [Configuration](https://www.home-assistant.io/docs/configuration/)
- [Secrets](https://www.home-assistant.io/docs/configuration/secrets/)
- [Splitting configuration](https://www.home-assistant.io/docs/configuration/splitting_configuration/)
- [Packages](https://www.home-assistant.io/docs/configuration/packages/)
- [YAML](https://www.home-assistant.io/docs/configuration/yaml/)
