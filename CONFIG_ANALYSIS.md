# Home Assistant Configuration Analysis: Cross-Package Dependencies

## Executive Summary

The Home Assistant configuration uses a packages-based architecture where different functionality is split across separate YAML files. This analysis reveals **critical cross-package entity dependencies** that could cause configuration failures if not properly managed during validation and loading.

---

## 1. Entity References Analysis

### routines.yaml - Specific Lines Referenced

#### Line 17 (Variable Declaration)
```yaml
is_mode_guest: "{{ is_state('input_boolean.mode_guest', 'on') }}"
```
**Entity Reference:** `input_boolean.mode_guest`
**Defined In:** `/config/input_boolean/modes.yaml` (not in a package)
**Status:** Cross-configuration dependency

#### Lines 44, 65, 115, 166 (Template Conditions)
```yaml
value_template: "{{ not is_mode_guest }}"
```
**Uses:** Variable declared from line 17 (indirect dependency on `input_boolean.mode_guest`)

### Analysis Points

The routines.yaml script does NOT have entity references at line 69 or 119 that would directly call garage door entities. However, it DOES have cross-package dependencies on `input_boolean.mode_guest` which is defined outside the packages system.

---

## 2. Entity Definition Locations

### input_boolean.mode_guest
- **Defined:** `/config/input_boolean/modes.yaml`
- **File Type:** Standalone input_boolean config (NOT in a package)
- **Loading Method:** `input_boolean: !include_dir_merge_named input_boolean` in configuration.yaml
- **Used In:**
  - `packages/routines.yaml` (line 17)
  - `packages/presence.yaml` (lines 32, 54, 129)

### input_boolean.garage_door_notification_sent
- **Defined:** `packages/garage_door_monitoring.yaml` (lines 35-38)
- **Loading Method:** Package system (`packages: !include_dir_named packages`)
- **Used In:**
  - `packages/garage_door_monitoring.yaml` (internal to same package)

### input_text.garage_door_active_notification_tag
- **Defined:** `packages/garage_door_monitoring.yaml` (lines 302-306)
- **Loading Method:** Package system
- **Used In:**
  - `packages/garage_door_monitoring.yaml` (internal to same package)

---

## 3. Package Loading Order

### Configuration Loading Strategy

From `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages

input_boolean: !include_dir_merge_named input_boolean
```

### Loading Sequence

1. **First:** `homeassistant.packages` - Loads ALL packages via directory include
   - Home Assistant loads packages in **alphabetical order**
   - Alphabetical order: air_quality → cameras → chores → device_health → garage_door_monitoring → known_batteries → light_groups → notifications → presence → remotes → routines → security_lights → towner_notifications → weather

2. **Second:** `input_boolean: !include_dir_merge_named input_boolean` - Loads standalone input_boolean files
   - Directory contents:
     - brief_module_toggles.yaml
     - modes.yaml (contains mode_guest)

### Critical Timing Issue

**PROBLEM:** The `input_boolean.mode_guest` is defined in a standalone config file that loads AFTER packages are processed. However, `routines.yaml` (which loads as part of packages) references `input_boolean.mode_guest` which doesn't yet exist at package load time.

**Home Assistant's Behavior:** Home Assistant loads all configuration sections, validates templates and entity references, and typically defers resolution until runtime. This usually works because:
- Templates are evaluated lazily at execution time
- State checking happens at runtime, not parse time
- Entity references in service calls are validated at runtime

---

## 4. Cross-Package Dependencies Found

### Dependency Map

```
routines.yaml (package)
  ↓ References (line 17)
  input_boolean.mode_guest (standalone config)

routines.yaml (package)
  ↓ Also uses indirectly via variables
  presence.yaml (package)

presence.yaml (package)
  ↓ References (lines 32, 54, 129)
  input_boolean.mode_guest (standalone config)

garage_door_monitoring.yaml (package)
  ↓ Internal - all references self-contained
  - input_boolean.garage_door_notification_sent (lines 35-38, defined same file)
  - input_text.garage_door_active_notification_tag (lines 302-306, defined same file)
```

### Other Packages with input_boolean/input_text Definitions

Scanning all packages for entity definitions:

1. **air_quality.yaml** (line 25)
   - Defines: `input_boolean:` section
   - Status: Package-internal

2. **chores.yaml** (line 36)
   - Defines: `input_boolean:` section
   - Status: Package-internal

3. **device_health.yaml** (line 21)
   - Defines: `input_boolean:` section
   - Status: Package-internal

4. **garage_door_monitoring.yaml** (lines 29, 302)
   - Defines: `input_boolean:` and `input_text:` sections
   - Status: Package-internal

5. **remotes.yaml** (line 2)
   - Defines: `input_boolean:` section
   - Status: Package-internal

6. **security_lights.yaml** (line 11)
   - Defines: `input_boolean:` section
   - Status: Package-internal

7. **towner_notifications.yaml** (line 11)
   - Defines: `input_boolean:` section
   - Status: Package-internal

---

## 5. Configuration Loading Details

### configuration.yaml Structure

```yaml
homeassistant:
  packages: !include_dir_named packages  # Loads all packages

# ... integrations ...

automation: !include_dir_merge_list automation
input_boolean: !include_dir_merge_named input_boolean  # Standalone input_boolean files
scene: !include scenes.yaml
```

### Key Points

1. **Package Loading:** `!include_dir_named packages` loads each file as a separate entity and merges them
   - All domains (script, automation, sensor, input_boolean, etc.) are merged at the top level
   - Loading order is filesystem-based (typically alphabetical)

2. **Input Boolean Loading:** `!include_dir_merge_named input_boolean` merges all files under that directory
   - Processed AFTER packages section in the config
   - Includes files: modes.yaml, brief_module_toggles.yaml

3. **No Explicit Order Control:** Home Assistant's `!include_dir_named` and `!include_dir_merge_named` have no built-in mechanism to control load order beyond filesystem order

---

## 6. Potential Cross-Package Issues

### Issue 1: Late Binding of mode_guest

**Risk Level:** LOW (but architectural concern)

**Description:**
- `routines.yaml` references `input_boolean.mode_guest` which is defined in a separate config section
- This works because template evaluation happens at runtime
- However, it creates tight coupling across configuration boundaries

**Impact:**
- If `input_boolean/modes.yaml` is deleted or renamed, `routines.yaml` will fail at execution time
- Static validators won't catch this at configuration parse time
- Requires runtime testing to validate

**Mitigation:**
- Document cross-package dependencies
- Add validation rules to check for entity existence before script execution
- Consider moving mode_guest into a package to keep related logic together

### Issue 2: Garage Door Package Self-Containment

**Risk Level:** LOW

**Description:**
- `garage_door_monitoring.yaml` is well-contained
- All referenced entities are defined within the same package
- Uses anchors for internal references (best practice)

**Impact:**
- No external dependencies mean this package can be removed/reordered without affecting others
- Good separation of concerns

### Issue 3: Alphabetical Load Order Dependency

**Risk Level:** MEDIUM

**Description:**
- If `presence.yaml` is renamed to something alphabetically earlier (e.g., `alarm_presence.yaml`), it might load before garage_door_monitoring
- No explicit dependencies are encoded in configuration
- Filesystem order changes could break assumptions

**Impact:**
- Package reorganization could break assumptions about load order
- No runtime protection if a package is moved

---

## 7. Detailed Entity Reference Summary

### Table: All Cross-Package Entity References

| Entity | Package/File | Defined | Used In | Line(s) | Type |
|--------|-------------|---------|---------|---------|------|
| input_boolean.mode_guest | input_boolean/modes.yaml | Yes | routines.yaml, presence.yaml | 17, 32, 54, 129 | Variable + Condition |
| input_boolean.notification_camera_indoor | input_boolean/brief_module_toggles.yaml | Yes | presence.yaml | 107, 124, 132, 134, 151, 153 | Service Calls |
| input_boolean.notification_camera_outdoor | input_boolean/brief_module_toggles.yaml | Yes | presence.yaml | 108, 122 | Service Calls |
| input_boolean.garage_door_monitoring_enabled | garage_door_monitoring.yaml | Yes | garage_door_monitoring.yaml | 30-32, 52, 290 | Self-contained |
| input_boolean.garage_door_notification_sent | garage_door_monitoring.yaml | Yes | garage_door_monitoring.yaml | 35-37, 55, 60, 130, 216, 320 | Self-contained |
| input_text.garage_door_active_notification_tag | garage_door_monitoring.yaml | Yes | garage_door_monitoring.yaml | 303-305, 71, 136, 152, 220, 233, 258, 271, 325 | Self-contained |

---

## 8. Package Loading Verification

### Home Assistant `!include_dir_named` Behavior

From Home Assistant documentation:
- Loads all YAML files from a directory
- Creates a domain namespace for each file
- Merges all entities in that domain into a single list/dict
- **Load order:** Alphabetical by filename (filesystem dependent)
- **No recursion:** Does not load subdirectories (unless explicitly configured)

### Actual Load Order for Packages

Based on alphabetical filesystem order:
```
1. air_quality.yaml
2. cameras.yaml
3. chores.yaml
4. device_health.yaml
5. garage_door_monitoring.yaml
6. known_batteries.yaml
7. light_groups.yaml
8. notifications.yaml
9. presence.yaml
10. remotes.yaml
11. routines.yaml          <- Uses input_boolean.mode_guest
12. security_lights.yaml
13. towner_notifications.yaml
14. weather.yaml
```

**Then after all packages:**
```
15. input_boolean/ directory (modes.yaml and brief_module_toggles.yaml)
```

---

## 9. Recommendations for Improvement

### Short Term

1. **Document Dependencies:** Add comments to `routines.yaml` explaining that it depends on `input_boolean.mode_guest` from the standalone config

2. **Add Validation:** Create a validation script that checks all entity references at startup

3. **Add Comments:** Include entity definition locations as comments in files that reference them

### Medium Term

1. **Consolidate Related Entities:** Consider moving `input_boolean.mode_guest` into a separate package like `modes.yaml` or into `presence.yaml` where it's heavily used

2. **Create Dependency Map:** Maintain documentation of all cross-package dependencies

3. **Add Validation Tests:** Expand the validation suite to check:
   - All entity references resolve
   - No circular dependencies
   - Load order assumptions are documented

### Long Term

1. **Refactor Using Packages Fully:** Move standalone input_boolean and input_text entities into logical packages
   - `package.modes.yaml` - mode_guest and related
   - `package.notifications.yaml` - notification toggles and config

2. **Use Entity Naming Conventions:** Leverage the location-based naming to group related entities

3. **Add Integration Tests:** Validate that packages work correctly when loaded in any order

---

## 10. Summary of Findings

### Key Discoveries

1. **routines.yaml lines 69 and 119** do NOT contain direct garage door entity references
   - They reference `alarm_control_panel.home_alarm` (defined in presence.yaml)
   - These are valid Home Assistant service calls

2. **Primary Cross-Package Dependency:** `input_boolean.mode_guest`
   - Defined in `/config/input_boolean/modes.yaml` (standalone)
   - Used in `routines.yaml` (package) and `presence.yaml` (package)
   - Loaded AFTER packages due to load order

3. **Garage Door Package is Self-Contained**
   - All entities defined and used within `garage_door_monitoring.yaml`
   - Uses YAML anchors for internal references (best practice)
   - No external dependencies

4. **No Package Load Order Issues**
   - Home Assistant handles lazy template evaluation
   - Runtime resolution works correctly
   - However, it's an architectural smell - tight coupling across boundaries

### Risk Assessment

- **Configuration Stability:** HIGH - Will load and work correctly
- **Maintainability:** MEDIUM - Cross-file dependencies not explicitly documented
- **Refactor Safety:** MEDIUM - Moving/renaming files requires coordination
- **Validation Coverage:** MEDIUM - Static validators won't catch cross-package issues
