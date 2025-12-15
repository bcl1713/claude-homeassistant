# Home Assistant Configuration - Comprehensive Code Review

**Review Date:** December 14, 2025
**Scope:** All automations and packages
**Total Issues Found:** 20

---

## CRITICAL ISSUES

### 3. Weather Package - Race Condition in Template Sensors

**File:** `packages/weather.yaml:171-177`
**Severity:** 🔴 CRITICAL

**Problem:**

The forecast sensors read from MQTT sensor attributes that may not exist yet:

```yaml
state: >
  {% set forecast = state_attr('sensor.weather_hourly_forecast', 'forecast') %}
  {% if forecast is not none and forecast | length > 0 %}
    {{ forecast[0].condition }}
  {% else %}
    {{ states('weather.forecast_home') }}  # ❌ Fallback might be stale
  {% endif %}
```

If MQTT data hasn't published yet, you get stale weather data. No initial state
defined.

**Impact:** Incorrect weather conditions in briefings, especially on startup.

**Fix Required:**

- Add `initial_value: unknown` to MQTT sensors
- Add startup delay or wait logic before first briefing
- Use `is defined` checks for MQTT attributes

---

### 4. Device Health Package - Logic Error in Warning Count

**File:** `packages/device_health.yaml:165-172`
**Severity:** 🔴 CRITICAL

**Problem:**

The warning count calculation uses critical batteries list instead of filtering by
warning threshold:

```yaml
warning_count: >
  {% set devices = state_attr('sensor.brief_data_devices',
                              'critical_batteries') %}
  {% set warning_threshold = states('input_number.device_health_warning_threshold')
                            | int %}
  {% if devices %}
    {{ devices | selectattr('level', '<=', warning_threshold)
               | list | length }}
    # ❌ Using critical_batteries, should use all batteries
  {% endif %}
```

Should read from all batteries or get warning batteries list from data collector.

**Impact:** Warning count is wrong, users don't see accurate device health status.

**Fix Required:**

- Change to read from correct data source (not critical_batteries)
- Or have data collector provide separate warning_batteries list

---

### 5. Security Lights - 10-Minute Blocking Delay

**File:** `packages/security_lights.yaml:95-96`
**Severity:** 🔴 CRITICAL

**Problem:**

The automation has a hard 10-minute delay that blocks other automations:

```yaml
- delay: "00:10:00" # ❌ Blocking delay - can't react to new detection
```

If motion detected again during delay, it won't trigger. Should use `wait_trigger`
with timeout instead.

**Impact:** Security lights stay on longer than needed, or don't respond to
additional motion during cooldown.

**Fix Required:**

Replace with:

```yaml
- wait_for_trigger:
    - platform: state
      entity_id: binary_sensor.camera_front_drive_person_occupancy
      to: "off"
  timeout: "00:10:00"
  continue_on_timeout: true
```

---

## HIGH PRIORITY ISSUES

### 6. Garage Door - Missing Input Text Helper Definition

**File:** `packages/garage_door_monitoring.yaml:302-306`
**Severity:** 🟠 HIGH

**Problem:**

The package defines `input_text.garage_door_active_notification_tag` at the
bottom, but references it earlier (lines 69-73) without checking if it exists.
Initial value is empty string, which causes conditional logic to fail on first
run:

```yaml
if:
  - condition: template
    value_template: "{{ active_tag != 'unknown' and active_tag != '' }}"
```

Since `initial: ""`, first notification ever sent will have `active_tag == ''`, so
condition is false and notification won't dismiss properly.

**Impact:** First garage door notification may not dismiss correctly when door
closes.

**Fix Required:**

- Check config file order - ensure input_text defined before first use
- Or change initial value to a placeholder like `"unset"`
- Add extra check for initial state

---

### 7. Presence Automation - Unsafe State Conversion

**File:** `packages/presence.yaml:163-164`
**Severity:** 🟠 HIGH

**Problem:**

Template condition has unsafe logic:

```yaml
condition:
  - condition: template
    value_template: >
      {{ trigger.from_state.state | int <
         trigger.to_state.state | int }}
      # ❌ Crashes if zone state is non-numeric like 'unavailable'
```

If zone state is 'unavailable' or 'unknown', the `| int` filter will fail.

**Impact:** Automation may crash and fail to trigger light welcome-home.

**Fix Required:**

```yaml
value_template: >
  {{ (trigger.from_state.state | int(0)) <
     (trigger.to_state.state | int(0)) }}
```

Or better:

```yaml
value_template: >
  {{ trigger.from_state is not none and
     trigger.to_state is not none and
     (trigger.from_state.state | int(0)) <
     (trigger.to_state.state | int(0)) }}
```

---

### 8. Good Night Routine - Service Target Syntax Error

**File:** `packages/routines.yaml:28-30`
**Severity:** 🟠 HIGH

**Problem:**

Mixing YAML syntax styles inconsistently:

```yaml
# Line 28: Using old syntax (entity_id directly)
- service: light.turn_on
  entity_id: # ❌ Old style - should be under 'target:'
    - light.bedroom_table_lamp_brian

# Line 46: Using correct syntax
- service: media_player.turn_off
  target:
    entity_id: # ✅ Correct style
      - media_player.display_kitchen
```

**Impact:** Inconsistent behavior, potential future compatibility issues with HA
updates.

**Fix Required:**

Standardize all service calls to use `target:` syntax:

```yaml
- service: light.turn_on
  target:
    entity_id:
      - light.bedroom_table_lamp_brian
      - light.bedroom_table_lamp_hester
  data:
    brightness_pct: 50
    transition: 3
```

---

### 9. Orchestration - Potential Null Attribute Access

**File:** `packages/brief/orchestration_enhanced.yaml:521`
**Severity:** 🟠 HIGH

**Problem:**

Accessing dictionary key that might not exist without full validation:

```yaml
has_issues: >
  {{ devices_dict.get('has_issues', false)
     if devices_dict is mapping else false }}
```

The `devices_dict` comes from JSON parsing at line 472-478 that can fail. If
parsing returns a string or null, this could cause issues.

**Impact:** Brief prompt generation may fail or have undefined variables.

**Fix Required:**

Add explicit validation:

```yaml
devices_dict: >
  {%- set d = devices_data | from_json
       if devices_data is string and
       devices_data != '{}' else {} -%}
  {%- if d is mapping -%}
    {{ d }}
  {%- else -%}
    {}
  {%- endif -%}
```

---

### 10. Chores - Template Has HTML Comment in Output

**File:** `packages/chores.yaml:97`
**Severity:** 🟠 HIGH

**Problem:**

The macro output includes HTML comments in state value:

```yaml
state: >
  {% macro format_chore(assignee, chore_name, completed_state) -%}
  # {% if states(completed_state) == 'on' -%}
  # ❌ This creates "# (✓)" in state string
  (✓) {% endif -%}
```

The `#` character will appear in the state value, creating malformed output like:

```text
# (✓) Towner needs to empty the dishwasher
```

**Impact:** Chore summary sensor state looks broken in UI.

**Fix Required:**

Use proper Jinja2 line statements:

```yaml
state: >
  {% macro format_chore(assignee, chore_name, completed_state) -%}
  {% if states(completed_state) == 'on' -%}(✓) {% endif -%}
  {{ assignee }} needs to {{ chore_name }}
  {%- endmacro %}
```

---

## MEDIUM PRIORITY ISSUES

### 11. Weather Package - Inefficient Refresh Trigger

**File:** `packages/weather.yaml:81-88`
**Severity:** 🟡 MEDIUM

**Problem:**

Triggers on every state change of sun and weather entities even when no forecast
refresh is needed:

```yaml
trigger:
  - platform: time_pattern
    minutes: "/15"
  - platform: state
    entity_id: sun.sun # ❌ Triggers even for attribute changes
  - platform: state
    entity_id: weather.forecast_home
    attribute: weather_alert
```

The `sun.sun` state rarely changes meaningfully (just above/below horizon). But
entity also has attributes that change frequently. Every attribute change triggers
the automation, and then the condition checks if enough time passed. This is
wasteful.

**Impact:** Unnecessary API calls to weather service, increased template
evaluations.

**Fix Required:**

Remove sun.sun trigger, keep only:

```yaml
trigger:
  - platform: time_pattern
    minutes: "/15"
  - platform: state
    entity_id: weather.forecast_home
    attribute: weather_alert
    to: "true" # Only on alert state change
```

---

### 12. Notifications Package - Hardcoded Time Window

**File:** `packages/notifications.yaml:21-23`
**Severity:** 🟡 MEDIUM

**Problem:**

Bus notification only triggers during fixed 07:35-07:59 window. No flexibility
for different school schedules or holidays:

```yaml
condition:
  - condition: time
    after: 07:35:00
    before: 07:59:00 # ❌ Hardcoded, inflexible
```

**Impact:** Can't adjust for schedule changes without editing YAML. Triggers even
on non-school days.

**Fix Required:**

Add configuration helpers:

```yaml
input_number:
  bus_notification_start_time:
    name: "Bus Notification Start"
    min: 0
    max: 23.99
    step: 0.01
    unit_of_measurement: "hours"
    initial: 7.583 # 07:35

input_boolean:
  bus_notification_enabled:
    name: "Bus Notifications"
    initial: true
```

Then use in condition:

```yaml
condition:
  - condition: state
    entity_id: input_boolean.bus_notification_enabled
    state: "on"
  # Then use template for time check
```

---

### 13. Device Health - Multiple Threshold Inconsistencies

**File:** `packages/device_health.yaml`
**Severity:** 🟡 MEDIUM

**Problems:**

**Issue A (Line 111):** Warning threshold uses hardcoded default 25 instead of 30

```yaml
{
  % set threshold = states('input_number.device_health_warning_threshold')
  | int(25) %,
}
# ❌ Default should be 30 (from line 54)
```

**Issue B (Line 165-166):** References wrong data source

```yaml
warning_count: >
  {% set devices = state_attr('sensor.brief_data_devices',
                              'critical_batteries') %}
  # ❌ Using critical_batteries, should use all batteries
```

**Issue C (Lines 192-208):** Multiple duplicate calculations

- `Device Health Critical Devices` calculates critical threshold (lines 192-218)
- Similar logic repeated in attributes

**Impact:** Inconsistent thresholds, wrong warning counts, code duplication makes
maintenance harder.

**Fix Required:**

1. Standardize default values: `| int(30)` everywhere
2. Point warning count to correct data source
3. Consolidate duplicate threshold logic into helper templates

---

### 14. Garage Door - Notification Tag Collision Risk

**File:** `packages/garage_door_monitoring.yaml:66`
**Severity:** 🟡 MEDIUM

**Problem:**

Using `now().timestamp()` to create unique tags could theoretically collide if two
notifications generated in same second:

```yaml
notification_tag: "garage-door-open-{{ now().timestamp() | int }}"
# ❌ If two notifications in same second, both get same tag
```

**Impact:** Unlikely but possible - two garage notifications could overwrite each
other.

**Fix Required:**

Use higher precision or add randomness:

```yaml
notification_tag: "garage-door-open-{{ now().timestamp() * 1000 | int(0) }}-{{
  range(0, 999) | random }}"
```

Or simpler:

```yaml
notification_tag: "garage-door-open-{{ now().isoformat()
  | replace(':', '').replace('-', '') }}"
```

---

### 15. Presence Automations - Mode Guest Not Consistently Honored

**File:** `packages/presence.yaml:72-95`
**Severity:** 🟡 MEDIUM

**Problem:**

When someone returns home during away mode, the door unlocks automatically
regardless of guest mode setting:

```yaml
# Lines 80-91: Unlocks door when returning home
- if:
    condition: state
    entity_id: alarm_control_panel.home_alarm
    state: "armed_away"
  then:
    - service: lock.unlock
      target:
        entity_id: lock.front_door
      # ❌ No check for guest mode
```

**Impact:** If guests arrive home while system is in away mode, they'd get
auto-unlocked. May be security concern or just unexpected behavior.

**Fix Required:**

Add guest mode check:

```yaml
- if:
    - condition: state
      entity_id: alarm_control_panel.home_alarm
      state: "armed_away"
    - condition: state
      entity_id: input_boolean.mode_guest
      state: "off" # Only auto-unlock if not in guest mode
  then:
    - service: lock.unlock
      target:
        entity_id: lock.front_door
```

---

### 16. Alarm Mode Transitions - Missing State Handlers

**File:** `packages/presence.yaml`
**Severity:** 🟡 MEDIUM

**Problem:**

Only handles 3 alarm states, but Home Assistant manual alarm has 5 possible
states:

- `disarmed` ❌ Not handled
- `armed_home` ✅ Handled
- `armed_away` ✅ Handled
- `armed_night` ✅ Handled
- `triggered` ❌ Not handled

No handlers for `disarmed` or `triggered` states could leave system in undefined
behavior.

**Impact:** If alarm is disarmed or triggered, no corresponding actions taken
(lights, notifications, etc.).

**Fix Required:**

Add automations for missing states:

```yaml
- alias: "alarm_handle_disarmed"
  id: "alarm_handle_disarmed"
  description: "Actions when alarm is disarmed"
  trigger:
    - platform: state
      entity_id: alarm_control_panel.home_alarm
      to: "disarmed"
  action:
    # Handle disarmed state (e.g., turn off notification toggles)

- alias: "alarm_handle_triggered"
  id: "alarm_handle_triggered"
  description: "Actions when alarm is triggered"
  trigger:
    - platform: state
      entity_id: alarm_control_panel.home_alarm
      to: "triggered"
  action:
    # Handle triggered state (e.g., send urgent notification, turn on lights)
```

---

### 17. Orchestration - Parallel Collection May Timeout Unevenly

**File:** `packages/brief/orchestration_enhanced.yaml:63-174`
**Severity:** 🟡 MEDIUM

**Problem:**

All 8 collectors run in parallel with same timeout. If one is slow, the entire
orchestration appears to fail:

```yaml
- parallel:
    - if:
        - condition: template
          value_template: "{{ chores_enabled }}"
      then:
        - service: script.brief_collect_chores_safe
          data:
            enabled: true
            timeout_seconds: "{{ collector_timeout }}" # All use same 15s
        - service: script.brief_wait_for_mqtt_sensor
          data:
            sensor_name: "sensor.brief_data_chores"
            timeout_seconds: "{{ collector_timeout }}"
```

If any collector takes >15 seconds, that parallel branch fails, and subsequent
sections of the orchestration may not complete properly.

**Impact:** If one data source is slow, the entire daily brief fails.

**Fix Required:**

- Allow per-collector timeout configuration
- Add error recovery - continue with partial data even if one collector fails
- Add retry logic with exponential backoff

---

### 18. Chores Package - Weekend Detection May Be Inconsistent

**File:** `packages/chores.yaml:109-116`
**Severity:** 🟡 MEDIUM

**Problem:**

Bathroom chore should only appear on weekend. The template does this:

```yaml
{# Weekend chores #}
{% if is_state('binary_sensor.time_is_weekend', 'on') %}
  # show bathroom
{% endif %}
```

But `binary_sensor.time_is_weekend` is a template sensor that recalculates
whenever it changes. If evaluated during Friday 23:59:59 -> Saturday 00:00:00
transition, there could be brief inconsistency where sensor updates but template
doesn't. Better to check time directly.

**Impact:** Brief time window where weekend chore might appear or disappear
incorrectly.

**Fix Required:**

Use time-based condition instead of entity state:

```yaml
condition:
  - condition: time
    weekday:
      - sat
      - sun
```

Or always calculate directly:

```yaml
state: >
  {# Weekend chores #}
  {% if now().weekday() in [5,6] %}
    {{ format_chore(...) }}
  {% endif %}
```

---

### 19. Security Lights - Mixed Control Systems May Conflict

**File:** `packages/security_lights.yaml:81-93`
**Severity:** 🟡 MEDIUM

**Problem:**

Mixing `light.turn_on` and `select.select_option` for the same physical light
(doorbell):

```yaml
- service: light.turn_on
  target:
    entity_id: light.doorbell_front_ring
- service: select.select_option
  data:
    option: "On"
  target:
    entity_id: select.front_doorbell_security_light
```

These are likely two different entities controlling the same light. If one service
succeeds and the other fails, state will be inconsistent.

**Impact:** Doorbell light may be on in one system but off in another, creating
confusing UI state.

**Fix Required:**

- Choose one method (either light service OR select service)
- Verify both entities actually control the same light
- Add checks to ensure both succeed or both fail atomically

---

### 20. Good Night Routine - Wait Timeout Behavior Unclear

**File:** `packages/routines.yaml:100-101`
**Severity:** 🟡 MEDIUM

**Problem:**

When sensor unavailable, user gets 2-minute timeout to respond:

```yaml
wait_for_trigger:
  - platform: event
    event_type: mobile_app_notification_action
    event_data:
      action: continue
  - platform: event
    event_type: mobile_app_notification_action
    event_data:
      action: cancel
continue_on_timeout: true
timeout: "00:02:00"

# Then...
- if:
    - condition: template
      value_template: >
        {{ wait.trigger is defined and
           wait.trigger.event.data.action == 'cancel' }}
  then:
    - stop: "Good Night routine canceled by user"
```

What happens if timeout expires without user action? The `wait.trigger` will be
undefined, so the `if` condition is false, and the routine continues automatically.
This should be explicit.

**Impact:** Silent behavior - routine continues automatically after timeout without
clear indication to user.

**Fix Required:**

```yaml
- if:
    - condition: template
      value_template: "{{ wait.trigger is not defined }}" # Timeout occurred
  then:
    - service: logbook.log
      data:
        name: "Good Night Routine"
        message: "Timeout waiting for user response - proceeding automatically"

- if:
    - condition: template
      value_template: >
        {{ wait.trigger is defined and
           wait.trigger.event.data.action == 'cancel' }}
  then:
    - stop: "Good Night routine canceled by user"
```

---

## RECOMMENDATIONS

### Code Quality Improvements

1. **Standardize Service Target Syntax**
   - Use `target:` with nested `entity_id:` consistently
   - Replace all old-style `entity_id:` at service level
   - Affects: `packages/routines.yaml`

2. **Centralize Configuration**
   - Create `packages/config_helpers.yaml` for common input_number/input_boolean
   - Include: time windows, thresholds, enable/disable toggles
   - Reference from all packages

3. **Add Dependency Documentation**
   - Each package should declare required packages/entities
   - Use YAML comments to list entity dependencies
   - Create dependency matrix in main README

4. **Implement Error Handling Pattern** - Catch service call failures, log errors
   to persistent notification, don't silently fail

5. **Use Atomic Operations for State** - Don't split one logical operation across
   multiple entity updates; use MQTT topics or template sensors instead

6. **Add Diagnostic Sensors** - Create sensor reporting last successful execution
   of each automation and track collector health status

7. **Test Timeout/Error Scenarios** - Add test cases for entity unavailable,
   service call failures, timeouts, and invalid state transitions

8. **Use Script Templates for Complex Logic** - Convert large if/then/choose
   chains to reusable scripts with error handling

9. **Add Startup Delays** - Weather, brief, device health should wait for system
   startup before first execution

10. **Create Debug Dashboard** - Add YAML dashboard showing recent automation
    runs, pending notifications, collector status, and battery levels

---

## Summary by Severity

| Severity    | Count  | Status                     |
| ----------- | ------ | -------------------------- |
| 🔴 CRITICAL | 5      | Requires immediate fix     |
| 🟠 HIGH     | 5      | Should fix soon            |
| 🟡 MEDIUM   | 10     | Should fix when convenient |
| **Total**   | **20** | **Review complete**        |

---

## Implementation Priority

**Immediate (Before Next Deployment):**

- Issue #1: Chores auto-reset logic
- Issue #3: Weather MQTT race condition
- Issue #4: Device health warning count
- Issue #5: Security lights delay timing

**This Week:**

- Issue #2: Missing helper definitions
- Issue #6: Garage door notification state
- Issue #7: Unsafe int conversion
- Issue #8: Service target syntax
- Issue #10: Template syntax error

**This Month:**

- Remaining medium priority issues
- Code quality improvements
- Add diagnostic sensors
- Documentation updates

---
