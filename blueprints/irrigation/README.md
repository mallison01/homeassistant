🌱 Advanced Multi-Zone Irrigation System

Full-featured irrigation automation for Home Assistant.
Supports 4 zones with schedules, sunrise/sunset offsets, early-abort sensors, consumption tracking, notifications, debug logging, and a ready-to-use dashboard.

📂 Files Included

Blueprint
config/blueprints/automation/advanced_irrigation.yaml
Main automation logic.

Helpers + Sensors Package
config/packages/irrigation_helpers.yaml
Provides all required input_text, input_number, template, and utility_meter entities.

Dashboard (optional)
config/dashboards/irrigation_dashboard.yaml
Lovelace dashboard with per-zone info, totals, and debug log.

README
config/blueprints/automation/irrigation/README.md (this file)

⚙️ Installation

Enable packages in configuration.yaml (if not already):

homeassistant:
  packages: !include_dir_named packages


Copy files:

Place helpers: config/packages/irrigation_helpers.yaml

Place blueprint: config/blueprints/automation/advanced_irrigation.yaml

(Optional) dashboard: config/dashboards/irrigation_dashboard.yaml

Restart Home Assistant.

🌐 Setup

Go to Settings → Automations & Scenes → Blueprints.

Import from Advanced Multi-Zone Irrigation Controller.

Configure:

Zone switches (4 max)

Runtimes (per zone)

Flow rates (per zone, gal/min)

Abort sensors (optional, per zone)

Schedules (fixed times, sunrise/sunset offsets, or helper entity)

Toggles: notifications + auto-reset

📊 Dashboard

Add irrigation dashboard with Raw Config Editor → paste this snippet:

views:
  - title: Irrigation
    path: irrigation
    icon: mdi:sprinkler-variant
    cards:

      - type: entities
        title: Debug
        entities:
          - entity: input_text.irrigation_debug_log

      - type: entities
        title: Zone 1
        entities:
          - input_text.zone_1_last_run
          - input_text.zone_1_last_status
          - input_number.zone_1_last_duration
          - input_number.zone_1_consumption

      - type: entities
        title: Zone 2
        entities:
          - input_text.zone_2_last_run
          - input_text.zone_2_last_status
          - input_number.zone_2_last_duration
          - input_number.zone_2_consumption

      - type: entities
        title: Zone 3
        entities:
          - input_text.zone_3_last_run
          - input_text.zone_3_last_status
          - input_number.zone_3_last_duration
          - input_number.zone_3_consumption

      - type: entities
        title: Zone 4
        entities:
          - input_text.zone_4_last_run
          - input_text.zone_4_last_status
          - input_number.zone_4_last_duration
          - input_number.zone_4_consumption

      - type: glance
        title: Totals
        entities:
          - sensor.irrigation_total_consumption
          - sensor.irrigation_daily
          - sensor.irrigation_weekly
          - sensor.irrigation_monthly

✅ Features

4 zones with independent runtimes & flow rates

Early-abort logic per zone (rain/soil sensors)

Schedules: fixed time, sunrise/sunset, or helper entity

Auto-reset toggle per zone

Notifications toggle

Per-zone last run, status, duration

Per-zone + total consumption tracking (gal)

Daily, weekly, monthly utility meters

Debug log helper for troubleshooting

🧪 Validation Checklist

 Helpers & sensors created

 Automation created from blueprint

 Zones run for configured duration

 Abort sensors skip zones when active

 Consumption sensors update correctly

 Utility meters increment

 Notifications send when enabled

 Debug log updates at each step

 Dashboard reflects all zone + total data

🔧 Troubleshooting

Check input_text.irrigation_debug_log for last debug message

Verify helpers loaded (Developer Tools → States)

If automation doesn’t trigger, confirm at least one schedule is enabled

If consumption doesn’t increment, check zone flow rates are set correctly
