---
name: templates
description: Home Assistant Jinja2 templating covering states, attributes, filters, math, date/time functions, and template sensors/binary sensors. Use when writing dynamic values in automations, scripts, conditions, notifications, or template entities in Home Assistant.
---
# Home Assistant Templates

Source: https://www.home-assistant.io/docs/configuration/templating/

## Overview

Home Assistant uses [Jinja2](https://jinja.palletsprojects.com/) for templating. Templates let you insert dynamic values anywhere a value can appear in YAML configuration — automations, scripts, notifications, template sensors, and more.

---

## Template Delimiters

| Syntax | Purpose |
|--------|---------|
| `{{ ... }}` | Expression – outputs a value |
| `{% ... %}` | Statement – control flow (if, for, set) |
| `{# ... #}` | Comment – ignored at runtime |

---

## Accessing Entity States

### Get an entity's state as a string

```jinja
{{ states('sensor.temperature') }}
```

### Check if an entity is in a specific state

```jinja
{{ is_state('binary_sensor.motion', 'on') }}
```

### Get an attribute value

```jinja
{{ state_attr('climate.thermostat', 'current_temperature') }}
{{ state_attr('light.living_room', 'brightness') }}
```

### Check if an attribute is a specific value

```jinja
{{ is_state_attr('media_player.tv', 'source', 'Netflix') }}
```

### Get last changed / updated time

```jinja
{{ states.sensor.temperature.last_changed }}
{{ (now() - states.binary_sensor.motion.last_changed).total_seconds() }}
```

---

## Numeric Conversions and Math

Always convert state strings to numbers before math operations:

```jinja
{{ states('sensor.temperature') | float }}
{{ states('sensor.count') | int }}
{{ states('sensor.temperature') | float * 9 / 5 + 32 }}   {# Celsius to Fahrenheit #}
{{ states('sensor.power') | float | round(2) }}
```

---

## String Filters

```jinja
{{ states('sensor.name') | upper }}
{{ states('sensor.name') | lower }}
{{ states('sensor.name') | title }}
{{ states('sensor.name') | replace('_', ' ') }}
{{ states('sensor.name') | truncate(30) }}
```

---

## Default Values and Error Handling

```jinja
{{ states('sensor.missing') | default('unknown') }}
{{ state_attr('sensor.x', 'unit') | default('°C') }}
{{ states('sensor.temp') | float(default=0) }}
```

---

## Conditional Expressions

### Inline if (ternary)

```jinja
{{ 'on' if is_state('switch.pump', 'on') else 'off' }}
{{ 'Hot' if states('sensor.temp') | float > 30 else 'Cool' }}
```

### Multi-line if/elif/else block

```jinja
{% if is_state('person.alice', 'home') %}
  Alice is home
{% elif is_state('person.alice', 'not_home') %}
  Alice is away
{% else %}
  Alice's location is unknown
{% endif %}
```

---

## Date and Time

### Current time

```jinja
{{ now() }}                    {# datetime object #}
{{ now().hour }}               {# current hour (0-23) #}
{{ now().minute }}
{{ now().weekday() }}          {# 0=Monday, 6=Sunday #}
{{ now().strftime('%H:%M') }}  {# formatted time string #}
```

### Today's date comparisons

```jinja
{{ now().date() == today_at('00:00').date() }}
{{ now() > today_at('18:00') }}
```

### Time since last state change

```jinja
{{ (now() - states.binary_sensor.door.last_changed).total_seconds() > 300 }}
```

### Timestamp conversion

```jinja
{{ as_timestamp(states('sensor.last_seen')) }}
{{ as_datetime(states('sensor.timestamp')) }}
```

---

## Loops

```jinja
{% for entity in expand('group.lights') %}
  {{ entity.name }}: {{ entity.state }}
{% endfor %}
```

### Expand entities in a domain/group/area

```jinja
{{ expand('light') | selectattr('state', 'eq', 'on') | list | count }}
```

---

## Useful Built-in Functions

| Function | Description |
|----------|-------------|
| `states(entity_id)` | Returns state as string |
| `state_attr(entity_id, attr)` | Returns attribute value |
| `is_state(entity_id, state)` | Returns `true` if entity is in state |
| `is_state_attr(entity_id, attr, value)` | Returns `true` if attribute equals value |
| `expand(entity_or_group)` | Returns list of entity state objects |
| `now()` | Current local datetime |
| `utcnow()` | Current UTC datetime |
| `today_at(time_str)` | Today at a specific time (e.g., `today_at('08:00')`) |
| `as_timestamp(dt)` | Converts to Unix timestamp |
| `as_datetime(ts)` | Converts timestamp to datetime |
| `timedelta(hours=1)` | Creates a time duration |
| `distance(lat1, lon1, lat2, lon2)` | Distance between two coordinates |
| `closest(lat, lon, entities)` | Closest entity to a point |
| `area_id(entity_id)` | Area ID of entity |
| `area_name(area_id)` | Name of an area |
| `area_entities(area_id)` | All entities in an area |
| `device_id(entity_id)` | Device ID for an entity |
| `device_attr(device_id, attr)` | Attribute of a device |
| `has_value(entity_id)` | `true` if entity has a valid (non-unknown/unavailable) state |
| `iif(condition, if_true, if_false)` | Inline if shorthand |
| `log(value, base)` | Logarithm |
| `sin(x)`, `cos(x)`, `tan(x)` | Trigonometric functions |
| `max(list)`, `min(list)` | Maximum / minimum value |

---

## Jinja2 Filters (selection)

```jinja
{{ value | int }}                  {# to integer #}
{{ value | float }}                {# to float #}
{{ value | round(2) }}             {# round to 2 decimal places #}
{{ value | abs }}                  {# absolute value #}
{{ list | join(', ') }}            {# join list to string #}
{{ list | sort }}                  {# sort list #}
{{ list | unique }}                {# remove duplicates #}
{{ list | select('eq', 'on') }}    {# filter by equality #}
{{ list | selectattr('state', 'eq', 'on') }}  {# filter objects by attr #}
{{ list | map(attribute='name') | list }}      {# extract attribute #}
{{ list | count }}                 {# number of items #}
{{ value | regex_match('pattern') }}
{{ value | regex_replace('old', 'new') }}
{{ dict | items }}                 {# dict to list of (key, value) pairs #}
{{ dict | dictsort }}              {# sort dict by key #}
```

---

## Template Sensors

Define virtual sensors whose value is computed from a template:

```yaml
template:
  - sensor:
      - name: "Outdoor Temperature Fahrenheit"
        unit_of_measurement: "°F"
        state: "{{ (states('sensor.outdoor_temp') | float * 9/5 + 32) | round(1) }}"
        state_class: measurement
        device_class: temperature

      - name: "Lights On Count"
        state: "{{ expand('light') | selectattr('state', 'eq', 'on') | list | count }}"

      - name: "Total Power Usage"
        unit_of_measurement: "W"
        state: >
          {{ states('sensor.fridge_power') | float(0)
           + states('sensor.oven_power') | float(0)
           + states('sensor.washing_machine_power') | float(0) }}
```

---

## Template Binary Sensors

```yaml
template:
  - binary_sensor:
      - name: "Someone Home"
        state: >
          {{ is_state('person.alice', 'home')
             or is_state('person.bob', 'home') }}
        device_class: presence

      - name: "Window Open While Heating"
        state: >
          {{ is_state('binary_sensor.window', 'on')
             and is_state('climate.home', 'heat') }}
```

---

## Template Triggers

Use template triggers to fire automations on computed conditions:

```yaml
triggers:
  - trigger: template
    value_template: >
      {{ states('sensor.temperature') | float > 30
         and is_state('climate.ac', 'off') }}
```

---

## Multi-line Templates

Use YAML block scalars for multi-line templates:

```yaml
message: >
  The temperature is {{ states('sensor.temp') | float | round(1) }}°C.
  {% if states('sensor.temp') | float > 30 %}
  It is too hot!
  {% endif %}
```

---

## Debugging Templates

Use **Developer Tools → Template** in the Home Assistant UI to write and test templates with live feedback. Enter any Jinja2 template and see the rendered output immediately.

---

## Tips

- Always use `| float(default=0)` instead of `| float` when state might be `unavailable` or `unknown`.
- Quote single-line templates in YAML: `"{{ expression }}"`.
- Templates in `trigger:` context can reference `trigger` variable for trigger data.
- The `{{ this }}` variable provides access to the current entity (in template sensors/binary sensors).

## References

- [Templating - Home Assistant](https://www.home-assistant.io/docs/configuration/templating/)
- [Template Integration](https://www.home-assistant.io/integrations/template/)
- [Jinja2 Template Designer Docs](https://jinja.palletsprojects.com/en/stable/templates/)
