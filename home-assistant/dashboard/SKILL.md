---
name: dashboard
description: Home Assistant dashboard (Lovelace) covering views, cards, YAML mode, custom cards, and configuration. Use when creating or customizing Home Assistant dashboards, adding Lovelace cards via YAML, switching to YAML mode, or building dynamic and conditional UI layouts.
---
# Home Assistant Dashboard (Lovelace)

Source: https://www.home-assistant.io/docs/dashboard/

## Overview

Home Assistant's UI is built on the **Lovelace** dashboard framework. Dashboards consist of **views** (tabs/pages), each containing **cards** (widgets). Dashboards can be managed through the visual editor or configured entirely in YAML for full control.

---

## Dashboard Modes

### Storage mode (default)
Dashboards are stored in Home Assistant's database and edited through the UI. Convenient for most users.

### YAML mode
Dashboards are defined in YAML files. Ideal for version control, reproducibility, and advanced configuration.

Enable YAML mode in `configuration.yaml`:

```yaml
lovelace:
  mode: yaml
```

Then create `ui-lovelace.yaml` in the config directory.

### Mixed: multiple dashboards

```yaml
lovelace:
  mode: storage          # default dashboard uses storage
  dashboards:
    my-custom-dash:
      mode: yaml
      filename: dashboards/custom.yaml
      title: "Custom Dashboard"
      icon: mdi:home-automation
      show_in_sidebar: true
      require_admin: false
```

---

## YAML Dashboard Structure

```yaml
# ui-lovelace.yaml
title: My Home
views:
  - title: Home
    path: home
    icon: mdi:home
    cards:
      - type: entities
        title: Living Room
        entities:
          - light.living_room
          - switch.tv
          - climate.downstairs

  - title: Energy
    path: energy
    icon: mdi:lightning-bolt
    cards:
      - type: energy-distribution
```

---

## Built-in Card Types

### Entities card

Display and control multiple entities:

```yaml
- type: entities
  title: Bedroom Controls
  show_header_toggle: true
  entities:
    - entity: light.bedroom
      name: Main Light
    - entity: switch.fan
      name: Fan
    - type: divider
    - entity: input_number.temperature_setpoint
      name: Temperature Setpoint
```

### Glance card

Compact overview grid:

```yaml
- type: glance
  title: Status Overview
  columns: 4
  entities:
    - entity: binary_sensor.front_door
      name: Front Door
    - entity: binary_sensor.back_door
      name: Back Door
    - entity: lock.front_door
      name: Lock
    - entity: alarm_control_panel.home
      name: Alarm
```

### Button card

Single-entity button:

```yaml
- type: button
  entity: light.porch
  name: Porch Light
  icon: mdi:outdoor-lamp
  tap_action:
    action: toggle
```

### Tile card

Modern compact card with optional features:

```yaml
- type: tile
  entity: light.living_room
  name: Living Room
  features:
    - type: light-brightness
```

### Map card

Show entities on a map:

```yaml
- type: map
  entities:
    - entity: person.alice
    - entity: person.bob
    - zone: zone.home
  hours_to_show: 24
```

### History graph card

Display historical state graph:

```yaml
- type: history-graph
  title: Temperature History
  hours_to_show: 24
  entities:
    - entity: sensor.living_room_temperature
      name: Living Room
    - entity: sensor.outdoor_temperature
      name: Outdoor
```

### Statistic card

Show statistics (min, max, mean):

```yaml
- type: statistics-graph
  title: Energy Usage
  period:
    calendar:
      period: month
  stat_types:
    - sum
  entities:
    - sensor.electricity_consumed
```

### Media control card

Media player widget:

```yaml
- type: media-control
  entity: media_player.living_room_speaker
```

### Thermostat card

Climate control:

```yaml
- type: thermostat
  entity: climate.downstairs
  features:
    - type: climate-hvac-modes
      hvac_modes:
        - heat
        - cool
        - auto
        - "off"
```

### Weather forecast card

```yaml
- type: weather-forecast
  entity: weather.home
  forecast_type: daily
```

### Gauge card

Display a value as a gauge:

```yaml
- type: gauge
  entity: sensor.cpu_usage
  name: CPU Usage
  unit: "%"
  min: 0
  max: 100
  severity:
    green: 0
    yellow: 60
    red: 85
```

### Conditional card

Show a card only when a condition is met:

```yaml
- type: conditional
  conditions:
    - condition: state
      entity: binary_sensor.motion_hallway
      state: "on"
  card:
    type: entities
    title: Motion Detected!
    entities:
      - binary_sensor.motion_hallway
```

### Vertical stack / Horizontal stack

Group cards together:

```yaml
- type: vertical-stack
  cards:
    - type: glance
      entities:
        - light.kitchen
    - type: entities
      entities:
        - switch.kitchen_exhaust

- type: horizontal-stack
  cards:
    - type: button
      entity: script.morning_routine
    - type: button
      entity: script.night_routine
```

### Grid card

Custom grid layout:

```yaml
- type: grid
  columns: 3
  square: false
  cards:
    - type: tile
      entity: light.room1
    - type: tile
      entity: light.room2
    - type: tile
      entity: light.room3
```

### Markdown card

Display text with Markdown and templates:

```yaml
- type: markdown
  title: Welcome
  content: >
    ## Hello, {{ user }}!

    The time is **{{ now().strftime('%H:%M') }}**.
    The temperature outside is {{ states('sensor.outdoor_temp') }}°C.
```

### Picture elements card

Overlay controls on a floorplan image:

```yaml
- type: picture-elements
  image: /local/floorplan.png
  elements:
    - type: state-icon
      entity: light.living_room
      style:
        top: 40%
        left: 30%
    - type: state-label
      entity: sensor.living_room_temperature
      style:
        top: 45%
        left: 30%
        color: white
```

### Energy distribution card

```yaml
- type: energy-distribution
```

---

## Card Actions

Most cards support `tap_action`, `hold_action`, and `double_tap_action`:

```yaml
- type: button
  entity: light.living_room
  tap_action:
    action: toggle
  hold_action:
    action: more-info
  double_tap_action:
    action: call-service
    service: light.turn_on
    data:
      brightness_pct: 100
```

Action types: `toggle`, `turn-on`, `turn-off`, `more-info`, `navigate`, `url`, `call-service`, `assist`, `none`.

---

## Custom Cards (HACS / Manual)

1. Place custom card JS in `/config/www/` (accessible as `/local/`).
2. Add as a resource:

```yaml
# dashboards/custom.yaml or in lovelace configuration
resources:
  - url: /local/button-card.js
    type: module
```

3. Use with `type: custom:<card-name>`:

```yaml
- type: custom:button-card
  entity: light.living_room
  name: Living Room
```

---

## View Types

| Type | Description |
|------|-------------|
| `masonry` (default) | Columns layout, cards automatically stacked |
| `sidebar` | Fixed sidebar with main area |
| `panel` | Single full-width card |
| `sections` | Sections with headers and badge rows (new layout system) |

```yaml
views:
  - title: Dashboard
    type: sections
    path: home
    sections:
      - title: Lights
        type: grid
        cards:
          - type: tile
            entity: light.living_room
```

---

## Badges

Small indicators shown at the top of a view:

```yaml
views:
  - title: Home
    badges:
      - entity: person.alice
      - entity: person.bob
      - type: entity
        entity: sensor.outdoor_temperature
        name: Outside
    cards: []
```

---

## Tips

- Use **Edit Dashboard → Raw Configuration Editor** to toggle to YAML in storage mode.
- Press `c` to open the quick search/command palette in the dashboard.
- Use `!include` in YAML mode dashboards to split view definitions into separate files.
- Test template values in **Developer Tools → Template** before using them in `markdown` cards.

## References

- [Dashboard Overview](https://www.home-assistant.io/docs/dashboard/)
- [Dashboard Cards](https://www.home-assistant.io/dashboards/cards/)
- [Lovelace YAML Mode](https://www.home-assistant.io/dashboards/dashboards/#yaml-mode)
- [Custom Cards on HACS](https://hacs.xyz/)
