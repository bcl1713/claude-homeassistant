# Detailed Analysis: routines.yaml Lines 69 and 119

## Question
What entities are referenced at lines 69 and 119 in routines.yaml, and where are they defined?

## Answer

### Line 69 Content
```yaml
              - service: light.turn_off
                entity_id: light.downstairs_lights
```

**Analysis:**
- This is NOT directly referencing an entity at line 69
- Line 69 is part of the `entity_id:` parameter for a `light.turn_off` service call
- The entity being referenced is `light.downstairs_lights`

**Definition Status:** 
- `light.downstairs_lights` is not defined in the visible codebase
- Could be:
  1. A light defined in Home Assistant UI (not in config files)
  2. A group defined elsewhere
  3. Created by another integration
  4. Part of the standard Home Assistant entities

### Line 119 Content
```yaml
              - service: alarm_control_panel.alarm_arm_night
                entity_id: alarm_control_panel.home_alarm
```

**Analysis:**
- This line references `alarm_control_panel.home_alarm`
- This is the entity to act upon in the service call

**Definition Status:**
- **FOUND AND CORRECTLY DEFINED**
- Defined in: `packages/presence.yaml` lines 2-6
- Definition:
  ```yaml
  alarm_control_panel:
    - platform: manual
      name: Home Alarm
      code_arm_required: false
      arming_time: 10
      disarm_after_trigger: false
  ```
- Entity created as: `alarm_control_panel.home_alarm` (derived from name)
- Uses: Valid Service Call ✓

---

## Important Clarification

### The Real Cross-Package Issue

While lines 69 and 119 don't have problematic references, the **routines.yaml script does have a critical cross-package dependency on line 17:**

```yaml
is_mode_guest: "{{ is_state('input_boolean.mode_guest', 'on') }}"
```

This references `input_boolean.mode_guest` which is defined in:
- **File:** `/config/input_boolean/modes.yaml` (standalone, NOT in packages)
- **Lines:** 1-4

### The Load Order Issue

1. Configuration loads `packages: !include_dir_named packages`
   - This loads all package files in alphabetical order
   - `routines.yaml` is the 11th package to load

2. THEN configuration loads `input_boolean: !include_dir_merge_named input_boolean`
   - This loads the standalone input_boolean files
   - `modes.yaml` loads AFTER `routines.yaml` is already parsed

3. **Why It Still Works:**
   - Home Assistant doesn't validate template references at parse time
   - Templates are evaluated at **runtime** when the script executes
   - By runtime, `input_boolean.mode_guest` is already loaded
   - This is "late binding" - perfectly valid in Home Assistant

---

## Complete routines.yaml Structure

```
SCRIPT: good_night
├── Triggers: None (called explicitly)
├── Variables declared at line 16-24:
│   ├── is_mode_guest (references input_boolean.mode_guest) ⚠️
│   ├── dishwasher_state
│   ├── is_dishwasher_available
│   └── is_dishwasher_running
│
├── ACTION SEQUENCES:
│   ├── Lines 27-33: Turn on bedside lamps
│   ├── Lines 36-39: Turn off outside lights
│   ├── Lines 42-53: Conditional - turn off media if not guest mode
│   │                (Uses is_mode_guest variable)
│   │
│   └── Lines 56-173: Choose block with multiple branches
│       ├── Branch 1 (Lines 58-71):
│       │  ├── Condition: dishwasher is running
│       │  └── Line 69: Turn off downstairs lights (light.downstairs_lights)
│       │  └── Line 70-71: Arm alarm (alarm_control_panel.home_alarm) ✓
│       │
│       ├── Branch 2 (Lines 74-122):
│       │  ├── Condition: dishwasher unavailable
│       │  ├── Send notification and wait for response
│       │  ├── Check user response
│       │  ├── Line 119: Arm alarm (alarm_control_panel.home_alarm) ✓
│       │  └── Other light/alarm operations
│       │
│       └── Default (Lines 124-173):
│           ├── Notify and wait for dishwasher to start
│           ├── Handle wait trigger
│           └── Final alarm arming

ENTITIES REFERENCED IN routines.yaml:
├── light.bedroom_table_lamp_brian (Turn on)
├── light.bedroom_table_lamp_hester (Turn on)
├── light.outdoor_front_yard (Turn off)
├── light.outdoor_back_yard (Turn off)
├── media_player.display_kitchen (Turn off)
├── media_player.speaker_living_room (Turn off)
├── media_player.speaker_music_room (Turn off)
├── remote.tv_living_room (Turn off)
├── light.downstairs_lights (Turn off - LINE 69)
├── sensor.dishwasher_state (Read state for logic)
├── input_boolean.mode_guest (Check state - CROSS-PACKAGE)
├── alarm_control_panel.home_alarm (Arm night - LINES 70, 120) ✓
└── zone.home (Used in sensor reading)

CROSS-PACKAGE DEPENDENCIES:
├── input_boolean.mode_guest
│   └── Defined in: /config/input_boolean/modes.yaml
│   └── Why it works: Loaded after packages via lazy template eval
│
└── alarm_control_panel.home_alarm
    └── Defined in: packages/presence.yaml
    └── Why it works: presence.yaml loads earlier alphabetically

LIGHT REFERENCES:
├── light.downstairs_lights
│   └── Status: NOT FOUND in codebase
│   └── Possible: UI-created entity or other integration
│   └── No issue: Will fail gracefully if not found
```

---

## Summary: Lines 69 and 119

| Line | Content | Entity | Defined In | Status |
|------|---------|--------|-----------|--------|
| 69 | `entity_id: light.downstairs_lights` | light.downstairs_lights | Unknown (UI or other?) | ⚠️ Not in YAML |
| 70-71 | `service: alarm_control_panel.alarm_arm_night`<br/>`entity_id: alarm_control_panel.home_alarm` | alarm_control_panel.home_alarm | packages/presence.yaml L2-6 | ✓ OK |
| 119 | (Same as line 70-71) | alarm_control_panel.home_alarm | packages/presence.yaml L2-6 | ✓ OK |

---

## Validation Results

### Valid References
- `alarm_control_panel.home_alarm` - Correctly defined ✓

### Unknown References
- `light.downstairs_lights` - Not found in YAML configs
  - Likely created in Home Assistant UI
  - Will show warning if entity doesn't exist
  - Not a configuration error

### Cross-Package But Working
- `input_boolean.mode_guest` - Defined outside packages
  - Works via lazy template evaluation
  - Architectural concern only

---

## Recommendations

### For Light References
1. **Check Home Assistant UI** to verify `light.downstairs_lights` exists
2. **Add comment** above line 69 explaining this is a group light
3. **Option:** Define as a light group in configuration if missing

### For Alarm References  
1. **No changes needed** - Correctly referenced and defined
2. **Already properly configured** in presence.yaml

### For mode_guest Dependency
1. **Short term:** Add comment explaining cross-package dependency
2. **Medium term:** Consider moving to a package file
3. **Long term:** Document all cross-package dependencies

