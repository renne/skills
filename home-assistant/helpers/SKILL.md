---
name: helpers
description: Home Assistant helper entities covering all types (input_boolean, input_number, input_text, input_select, input_datetime, input_button, counter, timer, schedule) with guidance on choosing the right type. Use when creating or managing Home Assistant helpers, selecting the correct helper type for a use case, or setting helper values from automations and scripts.
---
# Home Assistant Helpers

Source: https://www.home-assistant.io/docs/configuration/

## Overview

Helpers are virtual entities that store state or act as inputs for automations, scripts, and dashboards. They live in Home Assistant's storage and can be created via the UI (**Settings → Devices & Services → Helpers**), via `ha_config_set_helper`, or in YAML.

> **Always prefer the correct native helper over a workaround.** Using `input_boolean` is cleaner than an `input_text` storing `"true"/"false"`. Templates that recompute a fixed choice belong in `input_select`, not a Jinja2 expression.

---

## Choosing the Right Helper Type

| Use case | Correct type |
|----------|-------------|
| On/off flag (guest mode, away mode, maintenance) | `input_boolean` |
| Numeric setpoint, slider, threshold | `input_number` |
| Pick one from a fixed list (Home/Away/Night) | `input_select` |
| Free-form text, custom message | `input_text` |
| Date, time, or date+time picker | `input_datetime` |
| Press-to-trigger (doorbell, test button) | `input_button` |
| Event count (how many times something happened) | `counter` |
| Countdown (egg timer, cool-down delay) | `timer` |
| Weekly on/off schedule with time blocks | `schedule` |

---

## input_boolean

A simple on/off toggle — the most used helper.

```yaml
input_boolean:
  guest_mode:
    name: Guest Mode
    initial: false
    icon: mdi:account-multiple

  maintenance_mode:
    name: Maintenance Mode
    icon: mdi:wrench
```

**Services:** `input_boolean.turn_on`, `input_boolean.turn_off`, `input_boolean.toggle`

```yaml
actions:
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.guest_mode
```

**Condition check:**

```yaml
conditions:
  - condition: state
    entity_id: input_boolean.guest_mode
    state: "on"
```

---

## input_number

Stores a numeric value within a defined range.

```yaml
input_number:
  target_temperature:
    name: Target Temperature
    min: 15
    max: 30
    step: 0.5
    unit_of_measurement: "°C"
    mode: slider     # slider | box

  alarm_delay_minutes:
    name: Alarm Delay
    min: 0
    max: 60
    step: 1
    unit_of_measurement: min
    mode: box
```

**Services:** `input_number.set_value`, `input_number.increment`, `input_number.decrement`

```yaml
actions:
  - action: input_number.set_value
    target:
      entity_id: input_number.target_temperature
    data:
      value: 21
```

**Reading the value in a template:**

```yaml
"{{ states('input_number.target_temperature') | float }}"
```

---

## input_select

Stores one selection from a fixed list of options.

```yaml
input_select:
  home_mode:
    name: Home Mode
    options:
      - Home
      - Away
      - Night
      - Vacation
    initial: Home
    icon: mdi:home

  notify_channel:
    name: Notification Channel
    options:
      - SMS
      - Email
      - Push
```

**Services:** `input_select.select_option`, `input_select.select_next`, `input_select.select_previous`

```yaml
actions:
  - action: input_select.select_option
    target:
      entity_id: input_select.home_mode
    data:
      option: Away
```

**Condition check:**

```yaml
conditions:
  - condition: state
    entity_id: input_select.home_mode
    state: Night
```

---

## input_text

Stores an arbitrary text string.

```yaml
input_text:
  welcome_message:
    name: Welcome Message
    initial: "Welcome home!"
    max: 255

  last_triggered_by:
    name: Last Triggered By
    max: 100
    mode: text    # text | password
```

**Service:** `input_text.set_value`

```yaml
actions:
  - action: input_text.set_value
    target:
      entity_id: input_text.welcome_message
    data:
      value: "Good evening, everyone!"
```

---

## input_datetime

Stores a date, a time, or both.

```yaml
input_datetime:
  alarm_time:
    name: Alarm Time
    has_date: false
    has_time: true
    initial: "07:00:00"

  vacation_start:
    name: Vacation Start
    has_date: true
    has_time: false

  reminder_at:
    name: Reminder At
    has_date: true
    has_time: true
```

**Service:** `input_datetime.set_datetime`

```yaml
actions:
  - action: input_datetime.set_datetime
    target:
      entity_id: input_datetime.alarm_time
    data:
      time: "06:30:00"

  - action: input_datetime.set_datetime
    target:
      entity_id: input_datetime.vacation_start
    data:
      date: "2025-07-14"
```

**Trigger automation at the stored time:**

```yaml
triggers:
  - trigger: time
    at: input_datetime.alarm_time
```

---

## input_button

A stateless virtual button. Pressing it fires a `state_changed` event; automations can react to it.

```yaml
input_button:
  ring_doorbell:
    name: Ring Doorbell
    icon: mdi:doorbell
```

**Service:** `input_button.press`

```yaml
actions:
  - action: input_button.press
    target:
      entity_id: input_button.ring_doorbell
```

**Trigger on button press:**

```yaml
triggers:
  - trigger: state
    entity_id: input_button.ring_doorbell
```

---

## counter

An integer counter that can be incremented, decremented, and reset.

```yaml
counter:
  door_open_count:
    name: Door Opens Today
    initial: 0
    step: 1
    minimum: 0
    maximum: 9999
    icon: mdi:counter
```

**Services:** `counter.increment`, `counter.decrement`, `counter.reset`, `counter.set_value`

```yaml
actions:
  - action: counter.increment
    target:
      entity_id: counter.door_open_count

  # Reset every day at midnight
  - action: counter.reset
    target:
      entity_id: counter.door_open_count
```

---

## timer

A countdown timer that fires events when it finishes.

```yaml
timer:
  fan_shutoff:
    name: Fan Shutoff
    duration: "00:30:00"
    icon: mdi:fan
    restore: true
```

**Services:** `timer.start`, `timer.pause`, `timer.cancel`, `timer.finish`

```yaml
# Start with default duration
actions:
  - action: timer.start
    target:
      entity_id: timer.fan_shutoff

# Start with a custom duration
  - action: timer.start
    target:
      entity_id: timer.fan_shutoff
    data:
      duration: "00:15:00"
```

**Trigger when timer finishes:**

```yaml
triggers:
  - trigger: event
    event_type: timer.finished
    event_data:
      entity_id: timer.fan_shutoff
```

**State values:** `idle`, `active`, `paused`

---

## schedule

A weekly schedule helper that is `on` during configured time blocks and `off` otherwise.

```yaml
schedule:
  work_hours:
    name: Work Hours
    monday:
      - from: "09:00:00"
        to: "17:00:00"
    tuesday:
      - from: "09:00:00"
        to: "17:00:00"
    wednesday:
      - from: "09:00:00"
        to: "17:00:00"
    thursday:
      - from: "09:00:00"
        to: "17:00:00"
    friday:
      - from: "09:00:00"
        to: "17:00:00"
```

**Use in a condition:**

```yaml
conditions:
  - condition: state
    entity_id: schedule.work_hours
    state: "on"
```

> **Tip:** Prefer `schedule` over multiple `time` conditions when you have repeating weekly blocks. It makes the schedule visible and editable from the UI without touching automation YAML.

---

## Storage vs YAML Helpers

| Method | Pros | Cons |
|--------|------|------|
| UI / Storage | Editable from UI; survives config reload | Not in version control |
| YAML | Version-controlled; portable | Requires restart/reload; cannot be edited from UI |

- Use `ha_config_set_helper` (or the UI) for most helpers.
- YAML helpers (`input_boolean:`, `counter:`, etc. in `configuration.yaml` or a package) are useful for infrastructure-level helpers you want to track in git.

> **Note:** `ha_config_list_helpers` only returns **storage-based** helpers (created via UI/API). YAML-defined helpers appear in entity lists but cannot be managed via the helper API.

---

## Common Patterns

### Guard flag (prevent re-triggering)

```yaml
input_boolean:
  routine_running:
    name: Routine Running

# In the automation:
conditions:
  - condition: state
    entity_id: input_boolean.routine_running
    state: "off"
actions:
  - action: input_boolean.turn_on
    target:
      entity_id: input_boolean.routine_running
  # ... do the work ...
  - action: input_boolean.turn_off
    target:
      entity_id: input_boolean.routine_running
```

### Parameterized automation using helpers

```yaml
# User sets input_number.notification_delay from dashboard
# Automation reads the value at runtime
actions:
  - delay:
      minutes: "{{ states('input_number.notification_delay') | int }}"
  - action: notify.notify
    data:
      message: "Delayed notification fired"
```

### Cycle through modes with input_select

```yaml
# Morning button press cycles through Home/Away/Night
triggers:
  - trigger: state
    entity_id: input_button.mode_cycle
actions:
  - action: input_select.select_next
    target:
      entity_id: input_select.home_mode
```

---

## Tips

- Give helpers a clear `name` and an appropriate `icon` (`mdi:*`) so they are recognizable in dashboards and the logbook.
- For `input_number` used as a threshold in a `numeric_state` trigger, set `step` to match the sensor's precision.
- Use `timer` instead of a `delay` in an automation when the countdown must survive an HA restart (`restore: true`).
- A `schedule` helper is always preferable to a hardcoded `time` condition block when the schedule may change.

## References

- [Input Boolean](https://www.home-assistant.io/integrations/input_boolean/)
- [Input Number](https://www.home-assistant.io/integrations/input_number/)
- [Input Select](https://www.home-assistant.io/integrations/input_select/)
- [Input Text](https://www.home-assistant.io/integrations/input_text/)
- [Input Datetime](https://www.home-assistant.io/integrations/input_datetime/)
- [Input Button](https://www.home-assistant.io/integrations/input_button/)
- [Counter](https://www.home-assistant.io/integrations/counter/)
- [Timer](https://www.home-assistant.io/integrations/timer/)
- [Schedule](https://www.home-assistant.io/integrations/schedule/)
- [Helpers Overview](https://www.home-assistant.io/docs/configuration/helpers/)
