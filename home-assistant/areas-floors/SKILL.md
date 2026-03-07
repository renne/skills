---
name: areas-floors
description: Home Assistant areas and floors for organizing devices by physical location, plus labels and zones. Use when creating or reorganizing rooms, assigning devices to areas, setting up multi-floor buildings, applying labels for cross-cutting organization, or managing geographical zones for presence detection.
---
# Home Assistant Areas & Floors

Source: https://www.home-assistant.io/docs/configuration/

## Overview

Home Assistant uses a four-level organizational hierarchy:

| Level | Concept | Example |
|-------|---------|---------|
| Building | Implicit (your home) | — |
| **Floor** | Named vertical level | Ground Floor, Upstairs, Basement |
| **Area** | Named room or space | Living Room, Kitchen, Garage |
| **Entity / Device** | Assigned to an area | `light.living_room`, `sensor.kitchen_temp` |

Areas enable room-based control in automations (`area_id` target), voice assistants, and dashboards. Floors group areas for multi-story homes.

---

## Areas

An area represents a physical room or space. Entities and devices are assigned to an area and can be controlled as a group.

### Creating areas

Via `ha_config_set_area` (or **Settings → Areas & Zones → Areas → Add Area**):

- `name` — display name (e.g., `"Living Room"`)
- `icon` — Material Design Icon (e.g., `"mdi:sofa"`)
- `floor_id` — optional floor assignment
- `aliases` — alternative names for voice assistant recognition

```
# Example (conceptual - use ha_config_set_area tool or UI)
name: Living Room
icon: mdi:sofa
floor_id: ground_floor
aliases: ["lounge", "family room"]
```

### Common area icons

| Room | Icon |
|------|------|
| Living room | `mdi:sofa` |
| Kitchen | `mdi:chef-hat` |
| Bedroom | `mdi:bed` |
| Bathroom | `mdi:shower` |
| Garage | `mdi:garage` |
| Office | `mdi:desk` |
| Garden / Yard | `mdi:flower` |
| Hallway | `mdi:door` |
| Basement | `mdi:stairs-down` |
| Attic | `mdi:home-roof` |

### Assigning entities and devices to areas

- **UI:** Settings → Devices → click a device → assign Area.
- **Automation target:** Use `area_id` to control all devices in a room at once:

```yaml
actions:
  - action: light.turn_off
    target:
      area_id: living_room   # turns off all lights in the area

  - action: light.turn_on
    target:
      area_id:
        - kitchen
        - hallway
    data:
      brightness_pct: 80
```

### Using areas in conditions

```yaml
conditions:
  - condition: state
    entity_id: binary_sensor.living_room_motion
    state: "on"
```

### Listing areas in templates

```yaml
# All entity IDs in an area
"{{ area_entities('living_room') }}"

# All device IDs in an area
"{{ area_devices('living_room') }}"

# Area name for an entity
"{{ area_name('light.living_room') }}"

# Area ID for an entity
"{{ area_id('light.living_room') }}"
```

---

## Floors

Floors group areas into vertical levels and allow floor-level control.

### Creating floors

Via `ha_config_set_floor` (or **Settings → Areas & Zones → Floors → Add Floor**):

- `name` — display name (e.g., `"Ground Floor"`)
- `level` — numeric sort order (`0` = ground, `1` = first, `-1` = basement)
- `icon` — e.g., `"mdi:home-floor-0"`
- `aliases` — voice assistant names (e.g., `["downstairs", "main level"]`)

### Recommended level conventions

| Level value | Meaning |
|------------|---------|
| `-2`, `-1` | Basement / sub-basement |
| `0` | Ground floor |
| `1`, `2`, … | First floor, second floor, … |

### Assigning areas to floors

Set `floor_id` when creating or updating an area. Areas not assigned to a floor still appear in the UI but are not grouped.

---

## Labels

Labels provide a **cross-cutting, tag-based** organizational layer that spans areas and floors.

Use labels when an entity belongs to multiple conceptual groups:

- `security` — all security-related entities regardless of room
- `vacation-mode` — entities to shut off when away
- `energy-monitor` — entities included in energy reports

### Managing labels

Use `ha_config_set_label`, `ha_config_get_label`, `ha_config_remove_label`, and `ha_manage_entity_labels`.

### Using labels in automations

```yaml
# Turn off all entities labeled "vacation-mode"
actions:
  - action: homeassistant.turn_off
    target:
      label_id: vacation_mode
```

### Labels vs Areas

| Feature | Area | Label |
|---------|------|-------|
| Physical location | ✅ Yes | ❌ No |
| One entity per group | One area only | Multiple labels |
| Voice control | ✅ "Turn off living room" | ❌ |
| Cross-cutting grouping | ❌ | ✅ |

Use **areas** for location-based grouping and **labels** for functional/thematic grouping.

---

## Zones

Zones are geographical areas (GPS coordinates + radius) used for **presence detection**.

```yaml
# YAML definition (also manageable via UI: Settings → Areas & Zones → Zones)
zone:
  - name: Home
    latitude: 52.3731
    longitude: 4.8922
    radius: 100
    icon: mdi:home

  - name: Office
    latitude: 52.3760
    longitude: 4.9010
    radius: 150
    icon: mdi:office-building
```

### Zone triggers

```yaml
triggers:
  - trigger: zone
    entity_id: person.alice
    zone: zone.office
    event: enter    # enter | leave
```

### Zone conditions

```yaml
conditions:
  - condition: zone
    entity_id: person.alice
    zone: zone.home
```

### Zone templates

```yaml
# Check if person is in a zone
"{{ is_state('person.alice', 'zone.home') }}"

# Distance from home zone (km)
"{{ distance('zone.home', 'person.alice') | round(1) }}"
```

> **Note:** The `Home` zone is special — it is the default zone for presence and cannot be deleted. Its coordinates come from **Settings → System → General**.

---

## Practical Workflow: Setting Up a Multi-Floor Home

1. **Create floors** with `ha_config_set_floor`:
   - `Ground Floor` (level 0)
   - `First Floor` (level 1)
   - `Basement` (level -1)

2. **Create areas** with `ha_config_set_area`, assigning each to the correct floor:
   - `Living Room` → Ground Floor
   - `Kitchen` → Ground Floor
   - `Master Bedroom` → First Floor
   - `Home Office` → First Floor
   - `Utility Room` → Basement

3. **Assign devices** to areas via Settings → Devices, or specify `area_id` when creating helpers.

4. **Add labels** for cross-cutting groups like `security`, `energy`, or `vacation-mode`.

5. **Use area targets** in automations for room-level control without listing individual entities.

---

## Tips

- Always assign newly added devices to an area immediately — it makes voice control and dashboard organization much easier.
- Use `aliases` on areas/floors to teach voice assistants informal names ("downstairs", "lounge").
- Prefer `area_id` targets in automations over individual `entity_id` lists — adding a new device to an area automatically includes it in all such automations.
- Labels complement areas; a security camera can be in the `Garage` area and also have a `security` label.

## References

- [Areas](https://www.home-assistant.io/docs/organizing/areas/)
- [Floors](https://www.home-assistant.io/docs/organizing/floors/)
- [Labels](https://www.home-assistant.io/docs/organizing/labels/)
- [Zones](https://www.home-assistant.io/integrations/zone/)
- [Presence Detection](https://www.home-assistant.io/docs/presence-detection/)
