# Quick Reference: Cross-Package Entity Dependencies

## Critical Finding

Lines 69 and 119 in `routines.yaml` do NOT reference garage door entities. They reference `alarm_control_panel.home_alarm` which is correctly defined.

---

## Primary Cross-Package Dependency

### input_boolean.mode_guest

**Location:** `/config/input_boolean/modes.yaml` (standalone, NOT in packages)

**Used in:**
- `packages/routines.yaml` - Line 17 (variable declaration)
- `packages/presence.yaml` - Lines 32, 54, 129 (conditions)

**Loading Issue:** 
- `input_boolean/` files load AFTER packages in configuration
- But Home Assistant uses lazy template evaluation, so this works at runtime
- However, it's an architectural coupling issue

**Fix Recommendation:**
- Move `mode_guest` into a package file like `packages/modes.yaml`
- This keeps related functionality together
- Eliminates cross-boundary dependencies

---

## Garage Door Package Analysis

### Package: garage_door_monitoring.yaml

**All entities are self-contained:**

#### input_boolean Defined & Used in Same File:
- `input_boolean.garage_door_monitoring_enabled` (lines 30-33)
- `input_boolean.garage_door_notification_sent` (lines 35-38)

#### input_text Defined & Used in Same File:
- `input_text.garage_door_active_notification_tag` (lines 302-306)

**Status:** NO external dependencies - Package can be moved/reordered safely

---

## Package Load Order (Alphabetical)

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
11. routines.yaml
12. security_lights.yaml
13. towner_notifications.yaml
14. weather.yaml
---
15. input_boolean/ directory (standalone, loads AFTER packages)
```

---

## All Cross-Package References (Summary)

| Entity | Defined In | Used In |
|--------|-----------|---------|
| input_boolean.mode_guest | `input_boolean/modes.yaml` | routines.yaml (L17), presence.yaml (L32,54,129) |
| input_boolean.notification_camera_indoor | `input_boolean/brief_module_toggles.yaml` | presence.yaml |
| input_boolean.notification_camera_outdoor | `input_boolean/brief_module_toggles.yaml` | presence.yaml |

---

## Risk Assessment

| Aspect | Risk | Notes |
|--------|------|-------|
| **Configuration Loads** | LOW | Home Assistant handles late binding correctly |
| **Runtime Execution** | LOW | All entities resolve properly at runtime |
| **Maintainability** | MEDIUM | Dependencies not explicitly documented |
| **Refactoring** | MEDIUM | Moving/renaming files needs coordination |
| **Validation** | MEDIUM | Static validators won't catch these |

---

## Action Items

### Immediate (Do Nothing - Currently Working)
- Configuration loads and runs correctly
- No urgent issues to fix

### Short Term (Recommended)
1. Add comment in `routines.yaml` explaining mode_guest dependency
2. Create a dependency map document (DONE - see CONFIG_ANALYSIS.md)
3. Consider moving mode_guest into a package

### Medium Term (Nice to Have)
1. Add runtime validation for entity references
2. Move all input_boolean/input_text entities into packages
3. Document all cross-package dependencies

### Long Term (Optimization)
1. Refactor to fully use packages pattern
2. Eliminate all cross-package dependencies
3. Use configuration validation at CI/CD level

