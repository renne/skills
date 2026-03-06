---
name: scenes
description: Home Assistant scenes covering scene definition, activating scenes with transitions, applying ad-hoc scenes, and using scenes in automations and scripts. Use when defining preset states for multiple entities, creating ambiance lighting setups, or activating scenes from automations and dashboards.
---
# Home Assistant Scenes

Source: https://www.home-assistant.io/docs/scene/

## Overview

A **scene** captures the desired state of one or more entities at a single point in time. Activating a scene sets all listed entities to their configured states simultaneously. Scenes are defined in `scenes.yaml` (or included via `scene: !include scenes.yaml` in `configuration.yaml`).

---

## Basic Scene Definition

```yaml
# scenes.yaml
- name: "Evening Relaxation"
  entities:
    light.living_room:
      state: "on"
      brightness: 120
      color_temp: 400
    light.floor_lamp:
      state: "on"
      brightness: 60
    media_player.tv:
      state: "on"
      source: "Netflix"
    switch.reading_lamp: "on"

- name: "Movie Night"
  entities:
    light.living_room:
      state: "on"
      brightness: 30
      rgb_color: [10, 10, 60]
    light.floor_lamp: "off"
    media_player.projector:
      state: "on"
      source: "HDMI 1"
    cover.blinds: "closed"
```

**Key points:**
- `name`: Human-readable name. The entity ID will be `scene.<name_lowercase_with_underscores>`.
- `entities`: A dict mapping entity IDs to their desired states.
- Use a simple string (`"on"`, `"off"`) for binary states, or a dict for full attribute control.

---

## Scene ID and Icon

Optionally assign a persistent `id` (required for UI editing) and an icon:

```yaml
- id: "evening_relaxation"
  name: "Evening Relaxation"
  icon: mdi:sofa
  entities:
    light.living_room:
      state: "on"
      brightness: 120
```

---

## Entity State Options

### Lights

```yaml
light.bedroom:
  state: "on"
  brightness: 200          # 0-255
  brightness_pct: 80       # 0-100 (alternative to brightness)
  color_temp: 350          # mireds (153-500)
  color_temp_kelvin: 3000  # Kelvin (alternative)
  rgb_color: [255, 120, 0] # R, G, B (0-255)
  hs_color: [30, 100]      # Hue (0-360), Saturation (0-100)
  xy_color: [0.55, 0.38]   # CIE 1931 xy coordinates
  transition: 2            # transition time in seconds
```

### Climate

```yaml
climate.downstairs:
  state: "heat"
  temperature: 21.5
```

### Media player

```yaml
media_player.living_room:
  state: "on"
  source: "Spotify"
  volume_level: 0.3
```

### Cover (blinds, shutters)

```yaml
cover.living_room_blinds:
  state: "open"
  position: 70    # 0-100 (percentage open)
```

### Input helpers

```yaml
input_boolean.guest_mode: "on"
input_select.home_mode: "Evening"
input_number.brightness_level: 75
```

---

## Activating a Scene

### From the UI

Click the scene's card on your dashboard or via **Settings → Automations & Scenes → Scenes**.

### From an automation or script

```yaml
actions:
  - action: scene.turn_on
    target:
      entity_id: scene.evening_relaxation
```

### With a transition (for lights that support it)

```yaml
actions:
  - action: scene.turn_on
    target:
      entity_id: scene.movie_night
    data:
      transition: 3    # seconds
```

---

## Apply a Scene Ad-hoc (Without Pre-defining It)

Use `scene.apply` to set entity states on-the-fly without a pre-defined scene:

```yaml
actions:
  - action: scene.apply
    data:
      entities:
        light.living_room:
          state: "on"
          brightness: 200
          color_temp: 250
        light.floor_lamp: "off"
      transition: 2
```

---

## Create a Scene from Current States

Capture the current state of a set of entities and create a scene dynamically:

```yaml
actions:
  - action: scene.create
    data:
      scene_id: "my_captured_scene"
      snapshot_entities:
        - light.living_room
        - light.floor_lamp
        - switch.reading_lamp
```

Later restore it:

```yaml
actions:
  - action: scene.turn_on
    target:
      entity_id: scene.my_captured_scene
```

---

## Practical Examples

### Day/Night lighting

```yaml
- name: "Daytime"
  entities:
    light.living_room:
      state: "on"
      brightness_pct: 100
      color_temp_kelvin: 5000
    light.kitchen:
      state: "on"
      brightness_pct: 100

- name: "Night"
  entities:
    light.living_room:
      state: "on"
      brightness_pct: 20
      color_temp_kelvin: 2700
    light.kitchen: "off"
    light.hallway:
      state: "on"
      brightness_pct: 10
```

### Party mode

```yaml
- name: "Party"
  entities:
    light.living_room:
      state: "on"
      effect: "colorloop"
      brightness_pct: 100
    media_player.living_room_speaker:
      state: "on"
      volume_level: 0.7
      source: "Party Playlist"
```

### Good night

```yaml
- name: "Good Night"
  entities:
    light.living_room: "off"
    light.kitchen: "off"
    light.hallway:
      state: "on"
      brightness_pct: 5
    lock.front_door: "locked"
    switch.outdoor_lights: "off"
    climate.home:
      state: "heat"
      temperature: 18
```

---

## Using Scenes in Automations

Scenes are often used in time-based automations:

```yaml
- alias: "Evening lighting at sunset"
  triggers:
    - trigger: sun
      event: sunset
      offset: "-00:15:00"
  actions:
    - action: scene.turn_on
      target:
        entity_id: scene.evening_relaxation
      data:
        transition: 10

- alias: "Activate scene by input_select"
  triggers:
    - trigger: state
      entity_id: input_select.home_mode
  actions:
    - action: scene.turn_on
      target:
        entity_id: >
          scene.{{ trigger.to_state.state | lower | replace(' ', '_') }}
```

---

## Tips

- Always add an `id` to scenes you want to edit from the UI — scenes without IDs cannot be modified via the editor.
- Use `scene.create` + `scene.turn_on` to implement "restore previous state" logic before applying a new scene.
- `scene.apply` is ideal for one-off state changes without cluttering your scene list.
- Reload scenes without restart: **Developer Tools → YAML → Reload Scenes** or call `scene.reload`.

## References

- [Scenes Documentation](https://www.home-assistant.io/docs/scene/)
- [Scene Integration](https://www.home-assistant.io/integrations/scene/)
