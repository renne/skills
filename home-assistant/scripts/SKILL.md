---
name: scripts
description: Home Assistant scripts covering sequence actions, variables, execution modes, field parameters, and calling scripts from automations. Use when creating reusable action sequences, parameterizing scripts, or designing complex multi-step flows in Home Assistant.
---
# Home Assistant Scripts

Source: https://www.home-assistant.io/docs/scripts/

## Overview

Scripts are reusable, named sequences of actions in Home Assistant. Unlike automations, scripts are not triggered automatically — they must be called manually, from the UI, from an automation, or by another script. Scripts are defined in `scripts.yaml` (or included via `script: !include scripts.yaml` in `configuration.yaml`).

---

## Basic Structure

```yaml
# scripts.yaml
turn_on_living_room:
  alias: "Turn On Living Room"
  description: "Turns on all living room lights at full brightness"
  sequence:
    - action: light.turn_on
      target:
        area_id: living_room
      data:
        brightness_pct: 100
```

Each script is a top-level key. The `sequence` list is executed in order.

---

## Calling a Script

From an automation or another script:

```yaml
actions:
  - action: script.turn_on_living_room
```

Pass data/variables to a script:

```yaml
actions:
  - action: script.set_light_scene
    data:
      brightness: 60
      color_temp: 350
```

---

## Script Fields (Parameters)

Declare accepted input fields with `fields` to make scripts parameterizable:

```yaml
set_light_scene:
  alias: "Set Light Scene"
  fields:
    brightness:
      description: "Brightness percentage (0-100)"
      default: 50
      example: 80
      selector:
        number:
          min: 0
          max: 100
    color_temp:
      description: "Color temperature in mireds"
      selector:
        number:
          min: 153
          max: 500
  sequence:
    - action: light.turn_on
      target:
        entity_id: light.living_room
      data:
        brightness_pct: "{{ brightness }}"
        color_temp: "{{ color_temp }}"
```

---

## Variables

Use `variables` to pre-compute or name values used later in the sequence:

```yaml
welcome_home:
  sequence:
    - variables:
        greeting: "{{ 'Good morning' if now().hour < 12 else 'Good evening' }}"
        name: "Alex"
    - action: notify.mobile_app_my_phone
      data:
        message: "{{ greeting }}, {{ name }}!"
```

Variables can also be declared at the script level (shared across all script invocations):

```yaml
my_script:
  variables:
    default_brightness: 80
  sequence:
    - action: light.turn_on
      target:
        entity_id: light.kitchen
      data:
        brightness_pct: "{{ default_brightness }}"
```

---

## Script Execution Modes

Controls concurrent execution behavior:

| Mode | Behavior |
|------|----------|
| `single` (default) | New calls are ignored while script is running |
| `restart` | Running instance is stopped; script restarts from beginning |
| `queued` | New calls wait in a queue; run in order |
| `parallel` | Every call runs independently at the same time |

```yaml
flash_alert:
  mode: restart
  max: 5       # max queued or parallel instances (default: 10)
  sequence:
    - repeat:
        count: 5
        sequence:
          - action: light.toggle
            target:
              entity_id: light.alert
          - delay:
              milliseconds: 500
```

---

## All Available Sequence Actions

### Call a service / action

```yaml
- action: switch.turn_on
  target:
    entity_id: switch.garden_pump
```

### Delay

```yaml
- delay: "00:01:00"          # 1 minute
- delay:
    hours: 0
    minutes: 5
    seconds: 30
```

### Wait for a trigger

```yaml
- wait_for_trigger:
    - trigger: state
      entity_id: binary_sensor.door
      to: "off"
  timeout: "00:05:00"
  continue_on_timeout: true    # proceed even if timeout occurs
```

After the wait, `wait.completed` (boolean) and `wait.trigger` are available in templates.

### Wait for a template condition

```yaml
- wait_template: "{{ is_state('binary_sensor.motion', 'off') }}"
  timeout: "00:02:00"
```

### Fire an event

```yaml
- event: my_event
  event_data:
    message: "Script completed"
```

### Choose (if/elif/else)

```yaml
- choose:
    - conditions:
        - condition: state
          entity_id: input_select.mode
          state: "away"
      sequence:
        - action: alarm_control_panel.alarm_arm_away
          target:
            entity_id: alarm_control_panel.home
    - conditions:
        - condition: state
          entity_id: input_select.mode
          state: "night"
      sequence:
        - action: alarm_control_panel.alarm_arm_night
          target:
            entity_id: alarm_control_panel.home
  default:
    - action: alarm_control_panel.alarm_disarm
      target:
        entity_id: alarm_control_panel.home
```

### If/Then/Else (shorthand)

```yaml
- if:
    - condition: state
      entity_id: sun.sun
      state: "below_horizon"
  then:
    - action: light.turn_on
      target:
        entity_id: light.porch
  else:
    - action: light.turn_off
      target:
        entity_id: light.porch
```

### Repeat

```yaml
# Fixed count
- repeat:
    count: 3
    sequence:
      - action: script.beep

# While condition
- repeat:
    while:
      - condition: state
        entity_id: binary_sensor.motion
        state: "on"
    sequence:
      - action: light.toggle
        target:
          entity_id: light.alert
      - delay:
          seconds: 1

# Until condition
- repeat:
    until:
      - condition: numeric_state
        entity_id: sensor.temperature
        below: 25
    sequence:
      - action: fan.turn_on
        target:
          entity_id: fan.room
      - delay:
          minutes: 5

# For each (iterate a list)
- repeat:
    for_each:
      - "light.bedroom"
      - "light.kitchen"
      - "light.hallway"
    sequence:
      - action: light.turn_off
        target:
          entity_id: "{{ repeat.item }}"
```

### Parallel execution

```yaml
- parallel:
    - action: notify.mobile_app_alice
      data:
        message: "Doorbell rang"
    - action: notify.mobile_app_bob
      data:
        message: "Doorbell rang"
    - action: camera.snapshot
      target:
        entity_id: camera.doorbell
      data:
        filename: /config/www/doorbell.jpg
```

### Stop (early exit)

```yaml
- if:
    - condition: state
      entity_id: input_boolean.maintenance_mode
      state: "on"
  then:
    - stop: "Maintenance mode is active"

# Stop and return a value (when script is called as a function)
- stop: "Done"
  response_variable: result
```

### Set a variable

```yaml
- variables:
    temp: "{{ states('sensor.temperature') | float }}"
    message: "{{ 'Hot' if temp > 28 else 'Comfortable' }}"
```

---

## Script Response Variables

Scripts can return data to their caller using `response_variable`:

```yaml
calculate_duration:
  sequence:
    - variables:
        start_time: "{{ states('input_datetime.start') }}"
        end_time: "{{ states('input_datetime.end') }}"
        duration_seconds: >
          {{ (as_timestamp(end_time) - as_timestamp(start_time)) | int }}
    - stop: "Done"
      response_variable: duration_seconds
```

Calling automation can capture this with `response_variable`:

```yaml
- action: script.calculate_duration
  response_variable: result
- action: notify.notify
  data:
    message: "Duration: {{ result }} seconds"
```

---

## Practical Example: Morning Routine

```yaml
morning_routine:
  alias: "Morning Routine"
  mode: single
  sequence:
    - action: light.turn_on
      target:
        entity_id: light.bedroom
      data:
        brightness_pct: 30
        transition: 60
    - delay:
        minutes: 10
    - action: media_player.play_media
      target:
        entity_id: media_player.kitchen_speaker
      data:
        media_content_id: "https://stream.radio-station.com/live"
        media_content_type: music
    - action: climate.set_temperature
      target:
        entity_id: climate.home
      data:
        temperature: 21
    - action: notify.mobile_app_my_phone
      data:
        message: "Good morning! Routine activated."
```

---

## Tips

- Test scripts in **Developer Tools → Actions** by calling `script.<name>`.
- Use `mode: restart` for toggle-like scripts (e.g., flashing lights).
- Use `mode: queued` when script must run completely but order matters.
- Scripts with `fields` are callable from dashboards using the entity card.

## References

- [Script Syntax](https://www.home-assistant.io/docs/scripts/)
- [Script Integration](https://www.home-assistant.io/integrations/script/)
- [Script Conditions](https://www.home-assistant.io/docs/scripts/conditions/)
- [Performing Actions](https://www.home-assistant.io/docs/scripts/perform-actions/)
