# Home Assistant Sensor Light Blueprint - Occupancy Sensor Modification

This document provides step-by-step instructions for adding occupancy sensor support to the popular Home Assistant "Sensor Light" blueprint by Blackshome.

## What This Modification Adds

This modification adds an optional occupancy sensor feature that:
- **Keeps lights on** while occupancy is detected (does not turn lights on by itself)
- **Restarts automation timers** when occupancy changes from off to on
- **Includes a cooldown period** after occupancy ends before allowing lights to turn off
- **Works alongside existing motion sensors** without interfering with normal operation

## Prerequisites

- Existing "Sensor Light" blueprint (version 8.4) installed in Home Assistant
- Text editor to modify YAML files
- Basic understanding of Home Assistant YAML structure

## Installation Instructions

### 1. Add Input Section

**Find this section** (end of device_tracker_settings):
```yaml
        people:
          name: Device Tracker - People
          description: >
            Select the people you would like to track in the zone selected above. You must enter a zone above for this option to work.
          default: []
          selector:
            entity:
              multiple: true
              filter:
                domain:
                  - person
```

**Add this new section immediately after** (before `night_lights_trigger_settings:`):
```yaml
    occupancy_settings:
      name: "Occupancy Sensor"
      icon: mdi:motion-sensor
      collapsed: true
      input:
        include_occupancy:
          name: Use Occupancy Sensor (Optional)
          description: >
            Enable occupancy sensor to keep lights on when occupancy is detected.
            The occupancy sensor will not turn lights on, but will prevent them from turning off while occupied.
            When occupancy sensor turns on, it will restart the automation timers like the main trigger sensor.
          default: occupancy_disabled
          selector:
            select:
              options:
                - label: Enable occupancy sensor
                  value: "occupancy_enabled"
                - label: Disable occupancy sensor
                  value: "occupancy_disabled"
        occupancy_sensor:
          name: Occupancy Sensor
          description: >
            Select the occupancy sensors. These sensors will keep lights on when occupied but will not turn lights on.
            When using multiple occupancy sensors, it's advisable to group them together using a group helper.
          default: []
          selector:
            entity:
              filter:
                domain: binary_sensor
              multiple: true
        occupancy_cooldown:
          name: Occupancy Cooldown
          description: >
            Time to wait after occupancy sensor goes off before considering the space unoccupied.
            This prevents the lights from turning off too quickly if the occupancy sensor briefly goes off.
          default: 2
          selector:
            number:
              min: 0
              max: 10
              step: 0.5
              unit_of_measurement: minutes
```

### 2. Add Variables

**Find this section** (end of variables):
```yaml
  night_glow_light_rgbw_colour: !input night_glow_light_rgbw_colour
  night_glow_light_rgbww_colour: !input night_glow_light_rgbww_colour

# If motion sensor turns ON again within the time delay, it will restart the script.
```

**Replace with**:
```yaml
  night_glow_light_rgbw_colour: !input night_glow_light_rgbw_colour
  night_glow_light_rgbww_colour: !input night_glow_light_rgbww_colour
  include_occupancy: !input include_occupancy
  occupancy_sensor: !input occupancy_sensor
  occupancy_cooldown: !input occupancy_cooldown

# If motion sensor turns ON again within the time delay, it will restart the script.
```

### 3. Add Trigger

**Find this section** (end of triggers):
```yaml
  - trigger: homeassistant
    id: "t19"
    event: start

# All Conditions
```

**Replace with**:
```yaml
  - trigger: homeassistant
    id: "t19"
    event: start
  - trigger: state
    id: "t20"
    entity_id: !input occupancy_sensor
    from: "off"
    to: "on"

# All Conditions
```

### 4. Add Condition

**Find this section** (in the main OR conditions block):
```yaml
      - condition: and # trigger by HA Restart & check if by-pass auto off is enabled and any by-passes are on
        conditions:
          - condition: trigger
            id: 't19'
          - "{{ ('bypass_auto_off_enabled_on' in include_bypass_auto_off) or ('bypass_auto_off_enabled_off' in include_bypass_auto_off) or ('bypass_auto_off_enabled_stop' in include_bypass_auto_off) }}"
          - condition: or
            conditions:
              - condition: state
                entity_id: !input motion_bypass_lights_on
                match: any
                state: 'on'
              - condition: state
                entity_id: !input motion_bypass_lights_off
                match: any
                state: 'on'
              - condition: state
                entity_id: !input motion_bypass_lights_stop
                match: any
                state: 'on'

# Check Motion Sensor Manual By-pass
```

**Add this condition before the comment**:
```yaml
      - condition: and # trigger by occupancy sensor turning on, but only if lights are already on
        conditions:
          - condition: trigger
            id: 't20'
          - "{{ include_occupancy == 'occupancy_enabled' }}"
          - condition: state
            entity_id: !input occupancy_sensor
            match: any
            state: 'on'
          - condition: or
            conditions:
              - "{{ (expand(light_switch.entity_id) | selectattr('state', '==', 'on') | list | count > 0) }}"
              - "{{ (include_night_lights == 'night_lights_enabled') and (expand(night_lights.entity_id) | selectattr('state', '==', 'on') | list | count > 0) }}"
              - "{{ (include_night_lights == 'night_lights_enabled') and (include_night_glow == 'night_glow_enabled') and (expand(night_glow_lights.entity_id) | selectattr('state', '==', 'on') | list | count > 0) }}"
              - condition: template
                value_template: >-
                  {% if boolean_scenes_scripts != [] %}
                    {{ is_state(boolean_scenes_scripts, 'on') }}
                  {% endif %}
              - condition: template
                value_template: >-
                  {% if night_boolean_scenes_scripts != [] %}
                    {{ is_state(night_boolean_scenes_scripts, 'on') }}
                  {% endif %}
              - condition: template
                value_template: >-
                  {% if dynamic_lighting_boolean != [] %}
                    {{ is_state(dynamic_lighting_boolean, 'on') }}
                  {% endif %}

# Check Motion Sensor Manual By-pass
```

### 5. Update Wait Logic (Two Instances)

#### First Instance - Night Lights Section

**Find this code block**:
```yaml
          - choose:
              - alias: "Check if the trigger is on and wait for it to go off"
                conditions:
                  - condition: state
                    entity_id: !input motion_trigger
                    state: 'on'
                    match: any
                sequence:
                  - alias: "Wait until motion sensor is off"
                    wait_for_trigger:
                      - trigger: state
                        entity_id: !input motion_trigger
                        from: "on"
                        to: "off"
          - alias: "Wait the number of minutes set in the night lights time delay"
```

**Replace with**:
```yaml
          - choose:
              - alias: "Check if the triggers are on and wait for them to go off"
                conditions:
                  - condition: or
                    conditions:
                      - condition: state
                        entity_id: !input motion_trigger
                        state: 'on'
                        match: any
                      - condition: and
                        conditions:
                          - "{{ include_occupancy == 'occupancy_enabled' }}"
                          - condition: state
                            entity_id: !input occupancy_sensor
                            state: 'on'
                            match: any
                sequence:
                  - alias: "Wait until motion sensor is off"
                    wait_for_trigger:
                      - trigger: state
                        entity_id: !input motion_trigger
                        from: "on"
                        to: "off"
                  - choose:
                    - alias: "If occupancy is enabled, wait for occupancy to be clear"
                      conditions:
                        - "{{ include_occupancy == 'occupancy_enabled' }}"
                      sequence:
                        - alias: "Wait for occupancy sensors to be off for cooldown period"
                          wait_template: >
                            {{ expand(occupancy_sensor) | selectattr('state', 'eq', 'off') | list | count == (occupancy_sensor | count) }}
                          timeout: "01:00:00"
                        - delay:
                            minutes: !input occupancy_cooldown
          - alias: "Wait the number of minutes set in the night lights time delay"
```

#### Second Instance - Default Section

**Find this code block**:
```yaml
              - choose:
                - alias: "Check if the trigger is on and wait for it to go off"
                  conditions:
                    - condition: state
                      entity_id: !input motion_trigger
                      state: 'on'
                      match: any
                  sequence:
                    - alias: "Wait until motion sensor is off"
                      wait_for_trigger:
                        - trigger: state
                          entity_id: !input motion_trigger
                          from: "on"
                          to: "off"
              - alias: "Wait the number of minutes set in the normal lights time delay"
```

**Replace with**:
```yaml
              - choose:
                - alias: "Check if the triggers are on and wait for them to go off"
                  conditions:
                    - condition: or
                      conditions:
                        - condition: state
                          entity_id: !input motion_trigger
                          state: 'on'
                          match: any
                        - condition: and
                          conditions:
                            - "{{ include_occupancy == 'occupancy_enabled' }}"
                            - condition: state
                              entity_id: !input occupancy_sensor
                              state: 'on'
                              match: any
                  sequence:
                    - alias: "Wait until motion sensor is off"
                      wait_for_trigger:
                        - trigger: state
                          entity_id: !input motion_trigger
                          from: "on"
                          to: "off"
                    - choose:
                      - alias: "If occupancy is enabled, wait for occupancy to be clear"
                        conditions:
                          - "{{ include_occupancy == 'occupancy_enabled' }}"
                        sequence:
                          - alias: "Wait for occupancy sensors to be off for cooldown period"
                            wait_template: >
                              {{ expand(occupancy_sensor) | selectattr('state', 'eq', 'off') | list | count == (occupancy_sensor | count) }}
                            timeout: "01:00:00"
                          - delay:
                              minutes: !input occupancy_cooldown
              - alias: "Wait the number of minutes set in the normal lights time delay"
```

## What This Modification Accomplishes

After implementing these changes, your Sensor Light automation will:

1. **Show a new "Occupancy Sensor" section** in the blueprint configuration UI
2. **Allow selection of occupancy binary sensors** (supports multiple sensors)
3. **Keep lights on** while any occupancy sensor is active
4. **Restart timers** when occupancy sensor goes from off to on (if lights are already on)
5. **Wait for cooldown period** after occupancy ends before allowing lights to turn off
6. **Work seamlessly** with all existing motion sensor functionality

## Usage Notes

- The occupancy sensor **will not turn lights on** - it only keeps them on
- Motion sensors still function normally and can turn lights on
- Multiple occupancy sensors are supported
- Consider using a group helper if you have multiple occupancy sensors
- The cooldown period prevents lights from turning off too quickly if occupancy briefly goes off
- All existing blueprint features (dynamic lighting, bypass, night lights, etc.) continue to work unchanged

## Troubleshooting

- Ensure your occupancy sensors are binary sensors in Home Assistant
- Check that the blueprint reloads successfully after making changes
- Verify occupancy sensors show "on/off" states correctly
- Test with a short cooldown period first to ensure proper operation

## Contributing

Feel free to submit issues or improvements to enhance this modification.
