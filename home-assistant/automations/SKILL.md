---
name: automations
description: Home Assistant automations covering triggers, conditions, actions, modes, and blueprints. Use when creating or editing Home Assistant automations in YAML, troubleshooting automation logic, or designing event-driven home automation flows.
---
# Home Assistant Automations

Source: https://www.home-assistant.io/docs/automation/

## Overview

Automations in Home Assistant let your smart home react to events automatically. Every automation consists of three building blocks:

1. **Triggers** – what starts the automation.
2. **Conditions** (optional) – additional checks that must all pass before actions run.
3. **Actions** – what happens when the automation fires.

Automations are defined in `automations.yaml` (or any file included via `automation: !include automations.yaml` in `configuration.yaml`). They can also be created and edited through the UI.

---

## Basic Structure

```yaml
- id: "unique_id_string"
  alias: "Human-readable name"
  description: "Optional description"
  mode: single           # single | restart | queued | parallel
  triggers:
    - trigger: state
      entity_id: binary_sensor.motion
      to: "on"
  conditions:
    - condition: time
      after: "22:00:00"
      before: "06:00:00"
  actions:
    - action: light.turn_on
      target:
        entity_id: light.hallway
      data:
        brightness_pct: 20
```

> **Note:** The plural keys `triggers`, `conditions`, and `actions` are preferred for new automations. The singular forms (`trigger`, `condition`, `action`) remain valid for backward compatibility but plural forms are now preferred.

---

## Triggers

A trigger fires when its conditions are met. Multiple triggers are ORed – the automation fires when **any** trigger activates.

### State trigger

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.front_door
    to: "on"
    for:
      seconds: 5      # optional: state must hold for this duration
```

### Time trigger

```yaml
triggers:
  - trigger: time
    at: "07:30:00"
```

### Time pattern trigger

```yaml
triggers:
  - trigger: time_pattern
    minutes: "/15"    # every 15 minutes
```

### Sun trigger

```yaml
triggers:
  - trigger: sun
    event: sunset
    offset: "-00:30:00"   # 30 minutes before sunset
```

### Numeric state trigger

```yaml
triggers:
  - trigger: numeric_state
    entity_id: sensor.temperature
    above: 30
```

### Webhook trigger

```yaml
triggers:
  - trigger: webhook
    webhook_id: "my-secret-webhook-id"
```

### MQTT trigger

```yaml
triggers:
  - trigger: mqtt
    topic: "home/sensor/temperature"
    payload: "hot"
```

### Template trigger

```yaml
triggers:
  - trigger: template
    value_template: "{{ states('sensor.humidity') | float > 80 }}"
```

### Zone trigger

```yaml
triggers:
  - trigger: zone
    entity_id: person.alice
    zone: zone.home
    event: enter        # enter | leave
```

---

## Conditions

Conditions are all ANDed – **every** condition must be true for actions to proceed.

### State condition

```yaml
conditions:
  - condition: state
    entity_id: person.alice
    state: "home"
```

### Numeric state condition

```yaml
conditions:
  - condition: numeric_state
    entity_id: sensor.temperature
    below: 25
```

### Time condition

```yaml
conditions:
  - condition: time
    after: "18:00:00"
    before: "23:00:00"
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
```

### Sun condition

```yaml
conditions:
  - condition: sun
    after: sunset
    before: sunrise
```

### Template condition

```yaml
conditions:
  - condition: template
    value_template: "{{ is_state('input_boolean.guest_mode', 'off') }}"
```

### AND / OR / NOT conditions

```yaml
conditions:
  - condition: or
    conditions:
      - condition: state
        entity_id: input_boolean.away_mode
        state: "on"
      - condition: state
        entity_id: person.alice
        state: "not_home"
```

---

## Actions

Actions execute sequentially by default.

### Call a service / action

```yaml
actions:
  - action: light.turn_on
    target:
      entity_id: light.living_room
    data:
      brightness_pct: 80
      color_temp: 300
```

### Send a notification

```yaml
actions:
  - action: notify.mobile_app_my_phone
    data:
      title: "Alert"
      message: "Front door opened!"
```

### Delay

```yaml
actions:
  - delay:
      minutes: 5
```

### Wait for a state / template

```yaml
actions:
  - wait_for_trigger:
      - trigger: state
        entity_id: binary_sensor.motion
        to: "off"
    timeout:
      minutes: 10
    continue_on_timeout: false
```

### Choose (if/elif/else)

```yaml
actions:
  - choose:
      - conditions:
          - condition: state
            entity_id: sun.sun
            state: below_horizon
        sequence:
          - action: light.turn_on
            target:
              entity_id: light.porch
      - conditions:
          - condition: numeric_state
            entity_id: sensor.luminance
            below: 200
        sequence:
          - action: light.turn_on
            target:
              entity_id: light.porch
            data:
              brightness_pct: 50
    default:
      - action: light.turn_off
        target:
          entity_id: light.porch
```

### Repeat

```yaml
actions:
  - repeat:
      count: 3
      sequence:
        - action: light.toggle
          target:
            entity_id: light.alert
        - delay:
            milliseconds: 500
```

### Parallel execution

```yaml
actions:
  - parallel:
      - action: notify.mobile_app_alice
        data:
          message: "Motion detected!"
      - action: notify.mobile_app_bob
        data:
          message: "Motion detected!"
```

### Variables in actions

```yaml
actions:
  - variables:
      area_name: "living_room"
  - action: "light.turn_on"
    target:
      area_id: "{{ area_name }}"
```

### Fire an event

```yaml
actions:
  - event: my_custom_event
    event_data:
      message: "Automation fired"
```

---

## Automation Modes

Controls what happens when an automation is triggered while it is already running.

| Mode | Behavior |
|------|----------|
| `single` (default) | Ignores new trigger if already running |
| `restart` | Cancels current run and restarts |
| `queued` | Queues triggers; runs them one by one |
| `parallel` | Runs multiple instances simultaneously |

```yaml
mode: queued
max: 10       # max queued/parallel runs (default: 10)
```

---

## Automation Variables

Variables declared at the automation level are accessible throughout conditions and actions.

```yaml
- alias: "Adjust lights"
  variables:
    brightness: 80
  triggers:
    - trigger: time
      at: "20:00:00"
  actions:
    - action: light.turn_on
      target:
        entity_id: light.living_room
      data:
        brightness_pct: "{{ brightness }}"
```

---

## Trigger Variables

Each trigger type provides `trigger` data in templates:

```yaml
- alias: "Log state change"
  triggers:
    - trigger: state
      entity_id: binary_sensor.motion
  actions:
    - action: notify.notify
      data:
        message: >
          {{ trigger.entity_id }} changed from
          {{ trigger.from_state.state }} to {{ trigger.to_state.state }}
```

---

## Blueprints

Blueprints are reusable automation templates with configurable inputs:

```yaml
blueprint:
  name: "Motion-activated light"
  description: "Turn on a light when motion is detected."
  domain: automation
  input:
    motion_entity:
      name: Motion Sensor
      selector:
        entity:
          domain: binary_sensor
    light_entity:
      name: Light
      selector:
        entity:
          domain: light

triggers:
  - trigger: state
    entity_id: !input motion_entity
    to: "on"
actions:
  - action: light.turn_on
    target:
      entity_id: !input light_entity
```

Use blueprints via the UI or by referencing them in your automation:

```yaml
- alias: "Motion light - bedroom"
  use_blueprint:
    path: my_blueprints/motion_light.yaml
    input:
      motion_entity: binary_sensor.bedroom_motion
      light_entity: light.bedroom
```

---

## Tips

- Test automations with **Developer Tools → Actions** to call services manually.
- Trace an automation run with **Developer Tools → Automation Trace** to debug step-by-step.
- Use `id` fields on all automations to enable trace history and UI editing.
- Reload automations without a full restart: **Developer Tools → YAML → Reload Automations**.

## References

- [Automating Home Assistant](https://www.home-assistant.io/docs/automation/)
- [Automation Basics](https://www.home-assistant.io/docs/automation/basics/)
- [Automation Triggers](https://www.home-assistant.io/docs/automation/trigger/)
- [Automation Conditions](https://www.home-assistant.io/docs/scripts/conditions/)
- [Automation Actions](https://www.home-assistant.io/docs/automation/action/)
- [Automation Modes](https://www.home-assistant.io/docs/automation/modes/)
- [Automation YAML](https://www.home-assistant.io/docs/automation/yaml/)
- [Blueprints](https://www.home-assistant.io/docs/blueprint/)
