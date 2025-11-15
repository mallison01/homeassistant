# Smart Room Lighting v5.6.0 - Dynamic Lighting Fix Implementation Guide

**Date:** November 14, 2025  
**Type:** Bug Fix - Critical Architecture Issue  
**Complexity:** High - Major Refactor  
**Estimated Tokens:** 60-80k for implementation

---

## Executive Summary

Dynamic lighting in v5.5.0 is completely broken due to blueprint variable caching. All dynamic calculations (sun elevation, lux sensor, time-based) are computed ONCE at automation start and never update. This guide provides complete specifications to fix the issue by moving calculations from top-level variables to inline variables blocks.

---

## Table of Contents

1. [Root Cause Analysis](#root-cause-analysis)
2. [The Fix Strategy](#the-fix-strategy)
3. [Files Needed](#files-needed)
4. [Pre-Implementation Checklist](#pre-implementation-checklist)
5. [Step-by-Step Implementation](#step-by-step-implementation)
6. [Verification Procedures](#verification-procedures)
7. [Testing Requirements](#testing-requirements)

---

## Root Cause Analysis

### The Problem

**Symptom:** Dynamic lighting doesn't work - brightness and color temperature never change based on sun elevation, lux, or time.

**Root Cause:** Blueprint variables are evaluated ONCE at automation start and then cached.

```yaml
# CURRENT BROKEN IMPLEMENTATION (v5.5.0)
variables:
  dynamic_brightness: >-   # ❌ Evaluated ONCE at automation start
    {% set sun_elev = state_attr('sun.sun', 'elevation') | float(0) %}
    # ... calculations using STALE sun elevation from hours ago ...
    
  light_on_data: >-
    {% set data = {} %}
    {% if 'use_brightness' in light_control_options %}
      {% set data = dict(data, brightness_pct=dynamic_brightness) %}  # ❌ Uses cached value
```

**Evidence:**
1. Same issue as v5.4.0 `expand()` caching bug
2. Sensor-light.yaml uses inline variables blocks (working)
3. Variables in top-level `variables:` section are cached
4. Variables in action-level `variables:` blocks are fresh

### What's Affected

All dynamic lighting calculations:
- ✅ Sun elevation mode (broken)
- ✅ Sun elevation inverted mode (broken)
- ✅ Lux sensor mode (broken)
- ✅ Lux inverted mode (broken)
- ✅ Time-based mode (broken)
- ✅ Dynamic color temperature (broken)

---

## The Fix Strategy

### Pattern: From Cached to Fresh

**Before (Broken):**
```yaml
variables:  # Top-level - cached
  dynamic_brightness: >-
    {% set sun_elev = state_attr('sun.sun', 'elevation') %}
    # ... calculations ...

action:
  - service: homeassistant.turn_on
    data: "{{ light_on_data }}"  # Uses cached dynamic_brightness
```

**After (Working):**
```yaml
# NO dynamic_brightness in top-level variables

action:
  - variables:  # Inline - fresh every time
      dynamic_brightness_pct: >-
        {% set sun_elev = state_attr('sun.sun', 'elevation') %}
        # ... calculations ...
      light_data: >-
        {% set data = {} %}
        {% set data = dict(data, brightness_pct=dynamic_brightness_pct) %}
        {{ data }}
  
  - service: homeassistant.turn_on
    data: "{{ light_data }}"  # Uses fresh values
```

### Changes Required

#### 1. Remove from Top-Level Variables (lines 2775-3000)

**Delete these variables:**
- `dynamic_brightness` (lines 2775-2852, ~78 lines)
- `dynamic_color_temp` (lines 2857-2889, ~33 lines)
- `light_on_data` (lines 2894-2914, ~21 lines)
- `override_light_on_data` (lines 2919-2950, ~32 lines)
- `night_light_on_data` (lines 3124-3155, ~32 lines)

**Keep these (they're simple references, not calculations):**
- All input references
- All `use_*_feature` flags
- All static configuration values
- Entity lists and simple checks

#### 2. Add Inline Calculations Before Each Service Call

**Locations to modify (estimated 15-20 places):**

**Simple Mode:**
- Initial light turn-on
- Entity state trigger turn-on
- Button trigger turn-on
- Sun elevation trigger turn-on
- Ambient light trigger turn-on
- Time trigger turn-on

**Occupancy Mode:**
- Motion detected turn-on
- Door open turn-on
- Presence detection turn-on

**Special Modes:**
- Override mode turn-on
- Night lights turn-on
- Night glow turn-on

**Each location needs:**
```yaml
- variables:
    dynamic_brightness_pct: >-
      [COMPLETE CALCULATION HERE]
    dynamic_color_temp_kelvin: >-
      [COMPLETE CALCULATION HERE]
    current_light_data: >-
      [BUILD DATA DICT HERE]

- service: homeassistant.turn_on
  target: !input light_switch
  data: "{{ current_light_data }}"
```

---

## Files Needed

### Required for Implementation

1. **smart-room-lighting-v5_5_0.yaml** (current version)
   - Start with this file
   - Will be modified to v5.6.0

2. **sensor-light.yaml** (reference implementation)
   - NOT needed if using templates below
   - Only needed for pattern verification

### Will Be Created

1. **smart-room-lighting-v5_6_0.yaml** (new version)
2. **SRLA_v5.6.0_ChangeLog.md** (minimal documentation)

---

## Pre-Implementation Checklist

- [ ] Token budget: 80k+ tokens available
- [ ] Have smart-room-lighting-v5_5_0.yaml file
- [ ] Understand blueprint variable caching issue
- [ ] Understand inline variables pattern
- [ ] Ready to make 15-20 similar modifications
- [ ] Time available: 30-45 minutes of focused work

---

## Step-by-Step Implementation

### Phase 1: Setup & Version Update

#### 1.1 Copy File
```bash
cp smart-room-lighting-v5_5_0.yaml smart-room-lighting-v5_6_0.yaml
```

#### 1.2 Update Metadata
**Line 2:**
```yaml
# Smart Room Lighting v5.6.0
```

**Line 13:**
```yaml
name: "Smart Room Lighting v5.6.0"
```

---

### Phase 2: Remove Cached Variables

**CRITICAL:** These must be removed completely, not just commented out.

#### 2.1 Delete dynamic_brightness (lines ~2775-2852)

**Find:**
```yaml
  dynamic_brightness: >-
    {% set clamp = namespace(value=0) %}
    {% if not use_dynamic_lighting_feature or brightness_mode_setting == 'fixed' %}
      {% set clamp.value = light_brightness_pct %}
    {% elif brightness_mode_setting in ['sun_elevation', 'sun_elevation_inverted'] %}
    [... full 78 lines ...]
    {{ ([1, clamp.value | int, 100] | sort)[1] }}
```

**Action:** Delete entire variable definition

#### 2.2 Delete dynamic_color_temp (lines ~2857-2889)

**Find:**
```yaml
  dynamic_color_temp: >-
    {% set clamp = namespace(value=0) %}
    {% if not use_dynamic_lighting_feature or color_temp_mode_setting == 'fixed' %}
      {% set clamp.value = light_color_temp %}
    {% elif color_temp_mode_setting == 'sun_elevation' %}
    [... full 33 lines ...]
    {{ ([2000, clamp.value | int, 8000] | sort)[1] }}
```

**Action:** Delete entire variable definition

#### 2.3 Delete light_on_data (lines ~2894-2914)

**Find:**
```yaml
  light_on_data: >-
    {% set data = {} %}
    {% if 'use_brightness' in light_control_options %}
      {% set data = dict(data, brightness_pct=dynamic_brightness) %}
    {% endif %}
    [... full 21 lines ...]
    {{ data }}
```

**Action:** Delete entire variable definition

#### 2.4 Delete override_light_on_data (lines ~2919-2950)

**Find:**
```yaml
  override_light_on_data: >-
    {% set data = {} %}
    {% if 'use_brightness' in light_control_options %}
      {% set data = dict(data, brightness_pct=override_mode_brightness_val) %}
    [... full 32 lines ...]
    {{ data }}
```

**Action:** Delete entire variable definition

#### 2.5 Delete night_light_on_data (lines ~3124-3155)

**Find:**
```yaml
  night_light_on_data: >-
    {% set data = {} %}
    {% if 'use_brightness' in night_light_control_options %}
      {% set data = dict(data, brightness_pct=night_light_brightness_pct) %}
    [... full 32 lines ...]
    {{ data }}
```

**Action:** Delete entire variable definition

**Verification after Phase 2:**
```bash
# These should return nothing:
grep -n "dynamic_brightness:" smart-room-lighting-v5_6_0.yaml
grep -n "dynamic_color_temp:" smart-room-lighting-v5_6_0.yaml
grep -n "light_on_data:" smart-room-lighting-v5_6_0.yaml
grep -n "override_light_on_data:" smart-room-lighting-v5_6_0.yaml
grep -n "night_light_on_data:" smart-room-lighting-v5_6_0.yaml
```

---

### Phase 3: Add Inline Calculations - Templates

Use these templates for each location. Copy the appropriate template and insert it **immediately before** each `homeassistant.turn_on` service call.

#### Template A: Normal Light Control (with Dynamic Lighting)

```yaml
- variables:
    current_dynamic_brightness: >-
      {% set clamp = namespace(value=0) %}
      {% if not use_dynamic_lighting_feature or brightness_mode_setting == 'fixed' %}
        {% set clamp.value = light_brightness_pct %}
      {% elif brightness_mode_setting in ['sun_elevation', 'sun_elevation_inverted'] %}
        {% set sun_elev = state_attr('sun.sun', 'elevation') | float(0) %}
        {% set min_elev = al_sunset_elevation_val | float(-6) %}
        {% set max_elev = al_noon_elevation_val | float(60) %}
        {% set min_bright = al_min_brightness_val | float(20) %}
        {% set max_bright = al_max_brightness_val | float(100) %}
        {% if brightness_mode_setting == 'sun_elevation' %}
          {% if sun_elev <= min_elev %}
            {% set clamp.value = min_bright %}
          {% elif sun_elev >= max_elev %}
            {% set clamp.value = max_bright %}
          {% else %}
            {% set ratio = (sun_elev - min_elev) / (max_elev - min_elev) %}
            {% set clamp.value = min_bright + (ratio * (max_bright - min_bright)) %}
          {% endif %}
        {% else %}
          {% if sun_elev <= min_elev %}
            {% set clamp.value = max_bright %}
          {% elif sun_elev >= max_elev %}
            {% set clamp.value = min_bright %}
          {% else %}
            {% set ratio = (sun_elev - min_elev) / (max_elev - min_elev) %}
            {% set clamp.value = max_bright - (ratio * (max_bright - min_bright)) %}
          {% endif %}
        {% endif %}
      {% elif brightness_mode_setting == 'time_based' %}
        {% set current_time = now().strftime('%H:%M:%S') %}
        {% if current_time >= late_night_time_val or current_time < morning_time_val %}
          {% set clamp.value = late_night_brightness_val %}
        {% elif current_time >= morning_time_val and current_time < day_time_val %}
          {% set clamp.value = morning_brightness_val %}
        {% elif current_time >= day_time_val and current_time < evening_time_val %}
          {% set clamp.value = day_brightness_val %}
        {% else %}
          {% set clamp.value = evening_brightness_val %}
        {% endif %}
      {% elif brightness_mode_setting in ['lux', 'lux_inverted'] %}
        {% if not use_illuminance_sensor or illuminance_entity is none or illuminance_entity == '' %}
          {% set clamp.value = light_brightness_pct %}
        {% else %}
          {% set lux_state = states(illuminance_entity) %}
          {% if lux_state in ['unknown', 'unavailable', 'none'] %}
            {% set clamp.value = light_brightness_pct %}
          {% else %}
            {% set current_lux = lux_state | float(0) %}
            {% set min_lux = lux_for_min_brightness_val | float(50) %}
            {% set max_lux = lux_for_max_brightness_val | float(300) %}
            {% set min_bright = lux_min_brightness_val | float(10) %}
            {% set max_bright = lux_max_brightness_val | float(100) %}
            {% if brightness_mode_setting == 'lux' %}
              {% if current_lux <= min_lux %}
                {% set clamp.value = min_bright %}
              {% elif current_lux >= max_lux %}
                {% set clamp.value = max_bright %}
              {% else %}
                {% set ratio = (current_lux - min_lux) / (max_lux - min_lux) %}
                {% set clamp.value = min_bright + (ratio * (max_bright - min_bright)) %}
              {% endif %}
            {% else %}
              {% if current_lux <= min_lux %}
                {% set clamp.value = max_bright %}
              {% elif current_lux >= max_lux %}
                {% set clamp.value = min_bright %}
              {% else %}
                {% set ratio = (current_lux - min_lux) / (max_lux - min_lux) %}
                {% set clamp.value = max_bright - (ratio * (max_bright - min_bright)) %}
              {% endif %}
            {% endif %}
          {% endif %}
        {% endif %}
      {% else %}
        {% set clamp.value = light_brightness_pct %}
      {% endif %}
      {{ ([1, clamp.value | int, 100] | sort)[1] }}
    
    current_dynamic_color_temp: >-
      {% set clamp = namespace(value=0) %}
      {% if not use_dynamic_lighting_feature or color_temp_mode_setting == 'fixed' %}
        {% set clamp.value = light_color_temp %}
      {% elif color_temp_mode_setting == 'sun_elevation' %}
        {% set sun_elev = state_attr('sun.sun', 'elevation') | float(0) %}
        {% set min_elev = al_sunset_elevation_val | float(-6) %}
        {% set max_elev = al_noon_elevation_val | float(60) %}
        {% set min_kelvin = al_min_kelvin_val | float(2700) %}
        {% set max_kelvin = al_max_kelvin_val | float(5000) %}
        {% if sun_elev <= min_elev %}
          {% set clamp.value = min_kelvin %}
        {% elif sun_elev >= max_elev %}
          {% set clamp.value = max_kelvin %}
        {% else %}
          {% set ratio = (sun_elev - min_elev) / (max_elev - min_elev) %}
          {% set clamp.value = min_kelvin + (ratio * (max_kelvin - min_kelvin)) %}
        {% endif %}
      {% elif color_temp_mode_setting == 'time_based' %}
        {% set current_time = now().strftime('%H:%M:%S') %}
        {% if current_time >= late_night_time_val or current_time < morning_time_val %}
          {% set clamp.value = late_night_kelvin_val %}
        {% elif current_time >= morning_time_val and current_time < day_time_val %}
          {% set clamp.value = morning_kelvin_val %}
        {% elif current_time >= day_time_val and current_time < evening_time_val %}
          {% set clamp.value = day_kelvin_val %}
        {% else %}
          {% set clamp.value = evening_kelvin_val %}
        {% endif %}
      {% else %}
        {% set clamp.value = light_color_temp %}
      {% endif %}
      {{ ([2000, clamp.value | int, 8000] | sort)[1] }}
    
    current_light_data: >-
      {% set data = {} %}
      {% if 'use_brightness' in light_control_options %}
        {% set data = dict(data, brightness_pct=current_dynamic_brightness) %}
      {% endif %}
      {% if 'use_transition' in light_control_options %}
        {% set data = dict(data, transition=light_transition_on_time) %}
      {% endif %}
      {% if 'use_effect' in light_control_options and light_effect not in [none, ''] %}
        {% set data = dict(data, effect=light_effect) %}
      {% endif %}
      {% if light_color_control == 'use_color_temperature' %}
        {% set data = dict(data, kelvin=current_dynamic_color_temp) %}
      {% elif light_color_control == 'use_rgb_color' %}
        {% set data = dict(data, rgb_color=light_rgb) %}
      {% elif light_color_control == 'use_rgbw_color' %}
        {% set data = dict(data, rgbw_color=light_rgbw) %}
      {% elif light_color_control == 'use_rgbww_color' %}
        {% set data = dict(data, rgbww_color=light_rgbww) %}
      {% endif %}
      {{ data }}

- service: homeassistant.turn_on
  target: !input light_switch
  data: "{{ current_light_data }}"
```

#### Template B: Override Mode Light Control

```yaml
- variables:
    current_override_light_data: >-
      {% set data = {} %}
      {% if 'use_brightness' in light_control_options %}
        {% set data = dict(data, brightness_pct=override_mode_brightness_val) %}
      {% endif %}
      {% if 'use_transition' in light_control_options %}
        {% set data = dict(data, transition=light_transition_on_time) %}
      {% endif %}
      {% if 'use_effect' in light_control_options and override_mode_effect_val not in [none, ''] %}
        {% set data = dict(data, effect=override_mode_effect_val) %}
      {% endif %}
      {% if light_color_control == 'use_color_temperature' %}
        {% set data = dict(data, kelvin=override_mode_color_temp_val) %}
      {% elif light_color_control == 'use_rgb_color' %}
        {% set data = dict(data, rgb_color=override_mode_rgb_val) %}
      {% elif light_color_control == 'use_rgbw_color' %}
        {% set data = dict(data, rgbw_color=override_mode_rgbw_val) %}
      {% elif light_color_control == 'use_rgbww_color' %}
        {% set data = dict(data, rgbww_color=override_mode_rgbww_val) %}
      {% endif %}
      {{ data }}

- service: homeassistant.turn_on
  target: !input light_switch
  data: "{{ current_override_light_data }}"
```

#### Template C: Night Light Control

```yaml
- variables:
    current_night_light_data: >-
      {% set data = {} %}
      {% if 'use_brightness' in night_light_control_options %}
        {% set data = dict(data, brightness_pct=night_light_brightness_pct) %}
      {% endif %}
      {% if 'use_transition' in night_light_control_options %}
        {% set data = dict(data, transition=night_light_transition_on_time) %}
      {% endif %}
      {% if 'use_effect' in night_light_control_options and night_light_effect not in [none, ''] %}
        {% set data = dict(data, effect=night_light_effect) %}
      {% endif %}
      {% if night_light_color_control == 'use_color_temperature' %}
        {% set data = dict(data, kelvin=night_light_color_temp) %}
      {% elif night_light_color_control == 'use_rgb_color' %}
        {% set data = dict(data, rgb_color=night_light_rgb) %}
      {% elif night_light_color_control == 'use_rgbw_color' %}
        {% set data = dict(data, rgbw_color=night_light_rgbw) %}
      {% elif night_light_color_control == 'use_rgbww_color' %}
        {% set data = dict(data, rgbww_color=night_light_rgbww) %}
      {% endif %}
      {{ data }}

- service: homeassistant.turn_on
  target: !input night_lights
  data: "{{ current_night_light_data }}"
```

#### Template D: Night Glow Control

```yaml
- variables:
    current_night_glow_data: >-
      {% set data = {} %}
      {% if 'use_brightness' in night_glow_control_options %}
        {% set data = dict(data, brightness_pct=night_glow_brightness_pct) %}
      {% endif %}
      {% if 'use_transition' in night_glow_control_options %}
        {% set data = dict(data, transition=night_glow_transition_on_time) %}
      {% endif %}
      {% if 'use_effect' in night_glow_control_options and night_glow_effect not in [none, ''] %}
        {% set data = dict(data, effect=night_glow_effect) %}
      {% endif %}
      {% if night_glow_color_control == 'use_color_temperature' %}
        {% set data = dict(data, kelvin=night_glow_color_temp) %}
      {% elif night_glow_color_control == 'use_rgb_color' %}
        {% set data = dict(data, rgb_color=night_glow_rgb) %}
      {% elif night_glow_color_control == 'use_rgbw_color' %}
        {% set data = dict(data, rgbw_color=night_glow_rgbw) %}
      {% elif night_glow_color_control == 'use_rgbww_color' %}
        {% set data = dict(data, rgbw color=night_glow_rgbww) %}
      {% endif %}
      {{ data }}

- service: homeassistant.turn_on
  target: !input night_glow_lights
  data: "{{ current_night_glow_data }}"
```

---

### Phase 4: Find and Replace All Service Calls

**CRITICAL:** Every place that calls `homeassistant.turn_on` for lights must be updated.

#### 4.1 Find All Service Call Locations

```bash
# Find all light turn-on service calls
grep -n "service: homeassistant.turn_on" smart-room-lighting-v5_6_0.yaml > service_calls.txt

# Find which ones use light_on_data (must be updated)
grep -B 3 "data: \"{{ light_on_data }}\"" smart-room-lighting-v5_6_0.yaml

# Find which ones use override_light_on_data
grep -B 3 "data: \"{{ override_light_on_data }}\"" smart-room-lighting-v5_6_0.yaml

# Find which ones use night_light_on_data
grep -B 3 "data: \"{{ night_light_on_data }}\"" smart-room-lighting-v5_6_0.yaml
```

#### 4.2 Systematic Replacement

For **each location** found above:

1. **View the context** (10 lines before the service call)
2. **Determine which template** to use (A, B, C, or D)
3. **Insert the variables block** immediately before service call
4. **Update the service call** to use new variable name:
   - `light_on_data` → `current_light_data`
   - `override_light_on_data` → `current_override_light_data`
   - `night_light_on_data` → `current_night_light_data`

**Example Replacement:**

**Before:**
```yaml
- if:
    - "{{ (light_control_type_var == 'devices' or light_control_type_var == 'both') and has_light_entity }}"
  then:
    - service: homeassistant.turn_on
      target: !input light_switch
      data: "{{ light_on_data }}"
```

**After:**
```yaml
- if:
    - "{{ (light_control_type_var == 'devices' or light_control_type_var == 'both') and has_light_entity }}"
  then:
    - variables:
        current_dynamic_brightness: >-
          [TEMPLATE A BRIGHTNESS CALCULATION]
        current_dynamic_color_temp: >-
          [TEMPLATE A COLOR TEMP CALCULATION]
        current_light_data: >-
          [TEMPLATE A DATA BUILDER]
    
    - service: homeassistant.turn_on
      target: !input light_switch
      data: "{{ current_light_data }}"
```

#### 4.3 Expected Modification Locations

**Estimate: 15-20 locations total**

**Simple Mode Triggers:**
1. Basic trigger_on (line ~3900)
2. Entity trigger_on (line ~4100)
3. Button trigger_on (line ~4200)
4. Sun elevation trigger_on (line ~4300)
5. Ambient light trigger_on (line ~4400)
6. Time trigger_on (line ~4500)

**Occupancy Mode:**
7. Motion detected (line ~4700)
8. Door opened (line ~4800)
9. Entering state (line ~4900)

**Override Mode:**
10. Override activated (line ~3775) - Use Template B

**Night Lights:**
11. Night mode time start (line ~5100) - Use Template C
12. Night mode entity ON (line ~5200) - Use Template C
13. Night lights turn-on sequence (line ~5300) - Use Template C

**Night Glow:**
14. Night glow turn-on (line ~5400) - Use Template D

**Other Locations:**
15-20. Additional turn-on calls in complex logic blocks

**Search Strategy:**
```bash
# Find every occurrence needing update
grep -n "data: \"{{.*_on_data" smart-room-lighting-v5_6_0.yaml
```

---

### Phase 5: Verification & Testing

#### 5.1 YAML Syntax Validation

```python
python3 << 'EOF'
import yaml

class HALoader(yaml.SafeLoader):
    pass

def input_constructor(loader, node):
    return '!input ' + loader.construct_scalar(node)

HALoader.add_constructor('!input', input_constructor)

try:
    with open('smart-room-lighting-v5_6_0.yaml', 'r') as f:
        yaml.load(f, Loader=HALoader)
    print('✓ YAML syntax valid')
except yaml.YAMLError as e:
    print(f'✗ YAML error: {e}')
EOF
```

#### 5.2 Verify Variable Removal

```bash
# These should all return "not found" or nothing:
grep -n "dynamic_brightness:" smart-room-lighting-v5_6_0.yaml
grep -n "dynamic_color_temp:" smart-room-lighting-v5_6_0.yaml
grep -n "light_on_data:" smart-room-lighting-v5_6_0.yaml
grep -n "override_light_on_data:" smart-room-lighting-v5_6_0.yaml
grep -n "night_light_on_data:" smart-room-lighting-v5_6_0.yaml
```

#### 5.3 Verify All Service Calls Updated

```bash
# These should return nothing (all old references removed):
grep "{{ light_on_data }}" smart-room-lighting-v5_6_0.yaml
grep "{{ override_light_on_data }}" smart-room-lighting-v5_6_0.yaml
grep "{{ night_light_on_data }}" smart-room-lighting-v5_6_0.yaml

# These should return matches (new variables used):
grep "{{ current_light_data }}" smart-room-lighting-v5_6_0.yaml
grep "{{ current_override_light_data }}" smart-room-lighting-v5_6_0.yaml
grep "{{ current_night_light_data }}" smart-room-lighting-v5_6_0.yaml
```

#### 5.4 Count Inline Variables Blocks

```bash
# Should find 15-20 occurrences:
grep -c "current_dynamic_brightness:" smart-room-lighting-v5_6_0.yaml
grep -c "current_light_data:" smart-room-lighting-v5_6_0.yaml
```

#### 5.5 Line Count Check

```bash
wc -l smart-room-lighting-v5_6_0.yaml

# Expected:
# Before: 5,742 lines (v5.5.0)
# Removed: ~196 lines (old variables)
# Added: ~2,400-3,200 lines (15-20 inline blocks × ~160 lines each)
# After: ~7,950-8,750 lines (v5.6.0)
```

---

## Testing Requirements

### Critical Test Scenarios

#### Test 1: Sun Elevation Dynamic Brightness

**Setup:**
- Enable dynamic lighting
- Set brightness mode to "sun_elevation"
- Configure min/max brightness and elevation values

**Test:**
1. Trigger lights at sunrise (low elevation)
2. Verify brightness matches min_brightness setting
3. Wait 2 hours (higher elevation)
4. Trigger lights again
5. Verify brightness has increased
6. Test at noon (max elevation)
7. Verify brightness matches max_brightness setting

**Expected:** Brightness changes based on current sun elevation, not cached value from hours ago.

#### Test 2: Time-Based Dynamic Brightness

**Setup:**
- Enable dynamic lighting
- Set brightness mode to "time_based"
- Configure morning/day/evening/late_night times and brightness values

**Test:**
1. Trigger lights in morning time window
2. Verify brightness matches morning_brightness setting
3. Change system time to day window
4. Trigger lights again
5. Verify brightness matches day_brightness setting

**Expected:** Brightness changes based on current time, not cached value.

#### Test 3: Lux Sensor Dynamic Brightness

**Setup:**
- Enable dynamic lighting
- Set brightness mode to "lux"
- Configure illuminance sensor and min/max lux values

**Test:**
1. Set room to low lux (e.g., 10 lux)
2. Trigger lights
3. Verify brightness matches min_brightness
4. Increase room lux (e.g., 200 lux)
5. Trigger lights again
6. Verify brightness has increased proportionally

**Expected:** Brightness changes based on current lux reading, not cached value.

#### Test 4: Dynamic Color Temperature

**Setup:**
- Enable dynamic lighting
- Set color temp mode to "sun_elevation"
- Configure min/max kelvin values

**Test:**
1. Trigger lights at different sun elevations
2. Verify color temperature changes
3. Confirm kelvin value changes from warm (2700K) to cool (5000K)

**Expected:** Color temperature updates based on fresh sun elevation readings.

#### Test 5: Override Mode (Should NOT Use Dynamic)

**Setup:**
- Enable override mode
- Set override brightness to 50%
- Enable dynamic lighting (sun elevation mode)

**Test:**
1. Activate override mode
2. Verify lights turn on at 50% (not dynamic value)
3. Deactivate and reactivate override
4. Verify still uses 50% fixed value

**Expected:** Override mode uses fixed values, ignores dynamic lighting.

#### Test 6: Night Lights (Should Use Fixed Values)

**Setup:**
- Enable night lights
- Set night light brightness to 10%
- Enable dynamic lighting for main lights

**Test:**
1. Trigger night mode
2. Verify night lights use 10% brightness
3. Verify night lights don't use dynamic calculations

**Expected:** Night lights use configured fixed values.

---

## Common Issues & Solutions

### Issue 1: YAML Indentation Errors

**Symptom:** YAML validation fails with indentation error

**Solution:** 
- Check that inline `variables:` block is indented correctly
- Should align with service call (same indentation level)
- Each variable in the block should be indented 2 spaces from `variables:`

### Issue 2: Variables Not Found

**Symptom:** Template error: variable 'current_dynamic_brightness' not defined

**Solution:**
- Ensure variables block is BEFORE the service call
- Check variable names match exactly in both definition and usage

### Issue 3: Missing Old Variable Reference

**Symptom:** Template error: variable 'light_on_data' not defined

**Solution:**
- You missed updating a service call
- Search for all old variable names and replace

### Issue 4: Lights Not Responding to Dynamic Changes

**Symptom:** Lights turn on but brightness doesn't change throughout day

**Solution:**
- Verify you're using the inline variables template
- Check that you haven't accidentally left old cached variable
- Test by viewing automation trace in HA Developer Tools

### Issue 5: File Too Large

**Symptom:** File is now 8,000+ lines and hard to navigate

**Solution:**
- This is expected - we're duplicating calculation logic 15-20 times
- Consider future refactor to use scripts/helpers for shared logic
- For now, accept this as necessary to fix the caching issue

---

## Token Usage Tracking

### Estimated Breakdown

- **Phase 1 (Setup):** 2-3k tokens
- **Phase 2 (Remove variables):** 5-8k tokens
- **Phase 3 (Templates):** 5k tokens (already provided above)
- **Phase 4 (Replacements):** 40-55k tokens (15-20 locations × 2-3k each)
- **Phase 5 (Verification):** 5-8k tokens
- **Documentation creation:** 5-8k tokens
- **Total:** 62-87k tokens

### Progress Checkpoints

After each phase, run:
```bash
echo "Tokens used: [current] / [available]"
echo "Phase X complete"
```

If approaching 85% token usage:
- Complete current phase
- Generate documentation for remaining work
- Start new chat with updated file and remaining instructions

---

## Post-Implementation

### Documentation to Create

**Minimal (per user request):**
1. **SRLA_v5.6.0_ChangeLog.md** - Change log only

**Include in ChangeLog:**
- Root cause explanation
- What was changed (removed cached variables, added inline calculations)
- Number of locations modified
- Testing performed
- Line count change
- Migration notes (seamless upgrade, no config changes)

### File Outputs

1. **smart-room-lighting-v5_6_0.yaml** - Updated blueprint
2. **SRLA_v5.6.0_ChangeLog.md** - Minimal documentation

### Verification Checklist

- [ ] YAML validates successfully
- [ ] All old variable references removed
- [ ] All service calls updated with inline variables
- [ ] Line count increased by ~2,000-3,000 lines
- [ ] Tested basic trigger in Home Assistant
- [ ] Dynamic lighting responds to changes
- [ ] Documentation created
- [ ] Files copied to /mnt/user-data/outputs/

---

## Backward Compatibility

✅ **Fully backward compatible**

- No user configuration changes required
- All inputs remain the same
- Behavior identical except dynamic lighting now WORKS
- Seamless upgrade from v5.5.0

---

## Migration from v5.5.0

**Required Actions:** None - seamless upgrade

1. Replace v5.5.0 blueprint file with v5.6.0
2. Reload automations in Home Assistant
3. Test dynamic lighting in your environment
4. Verify lights respond to sun elevation/lux/time changes

---

## Success Criteria

✅ **Implementation Complete When:**

1. All 5 cached variables removed from top-level variables section
2. 15-20 inline variables blocks added before service calls
3. All service calls updated to use new variable names
4. YAML validates without errors
5. No references to old variable names remain
6. File size increased appropriately (~2,000-3,000 lines)
7. Documentation created
8. Basic test in Home Assistant successful

---

## Need Help?

### Quick Diagnostics

```bash
# Check if old variables still exist (should be empty)
grep -E "(dynamic_brightness|dynamic_color_temp|light_on_data):" smart-room-lighting-v5_6_0.yaml

# Count inline variables blocks (should be 15-20)
grep -c "current_dynamic_brightness:" smart-room-lighting-v5_6_0.yaml

# Check for missed service calls (should be empty)
grep "{{ light_on_data }}" smart-room-lighting-v5_6_0.yaml
```

### Common Patterns

**Finding service calls:**
```bash
# Pattern 1: Direct service call
grep -A 3 "service: homeassistant.turn_on" file.yaml | grep "data:"

# Pattern 2: Inside then: block
grep -B 5 "data: \"{{ light_on_data }}\"" file.yaml
```

---

## End of Implementation Guide

**This guide provides complete specifications for implementing the dynamic lighting fix. Follow systematically, verify each phase, and test thoroughly.**

**Estimated Time:** 45-60 minutes  
**Estimated Tokens:** 60-85k  
**Complexity:** High  
**Risk:** Medium (extensive changes, but well-defined pattern)

**Good luck!** 🚀
