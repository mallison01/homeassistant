# 🌱 Smart Multi-Zone Irrigation for Home Assistant

A **complete smart irrigation system** for Home Assistant with support for up to **4 independent irrigation zones**, configurable schedules, pump control, soil and weather monitoring, water usage tracking, notifications, and per-zone abort logic.  
All components are modular, optional, and fully configurable via UI.

---

## ✨ Features

- ✅ Up to **4 irrigation zones** (switches/valves)  
- ✅ **Fixed schedules** (3 configurable times per day)  
- ✅ **Sun-based schedules** (sunrise/sunset with offsets)  
- ✅ **Schedule Helper integration** (optional advanced scheduling)  
- ✅ **Manual overrides** per zone (Start/Stop + custom duration)  
- ✅ **Pump control** with delay and water-level safety checks  
- ✅ **Soil moisture, rainfall, and rain detection** (optional abort conditions)  
- ✅ **Per-zone early abort logic** (stop zone if threshold not met)  
- ✅ **Water usage calculation & daily/weekly/monthly meters**  
- ✅ **Persistent notifications** for skip/abort/usage events  
- ✅ **Debug logging toggle**  
- ✅ **Auto-reset toggle** (reset manual selects after use)  

---

## 📂 Repository Structure

```
/config
├── blueprints/automation/irrigation/
│   └── advanced_irrigation.yaml    # Main irrigation blueprint
├── packages/
│   └── irrigation_helpers.yaml     # Input_booleans, input_numbers, sensors, utility_meters
├── dashboards/
│   └── irrigation_dashboard.yaml   # Lovelace view (zones, totals, graphs)
└── README.md                       # You are here
```

---

## 🚀 Installation

### 1. Copy Files
Place the files in your Home Assistant config directory:

- `advanced_irrigation.yaml` → `config/blueprints/automation/irrigation/`  
- `irrigation_helpers.yaml` → `config/packages/`  
- `irrigation_dashboard.yaml` → `config/dashboards/`  

### 2. Enable Packages
Add this to `configuration.yaml` if not already present:

```yaml
homeassistant:
  packages: !include_dir_merge_named packages
```

### 3. Reload
- Reload **Automations**  
- Reload **Helpers**  
- Or restart Home Assistant  

### 4. Import Dashboard
- Go to **Settings → Dashboards → Add Dashboard**  
- Open the **Raw Config Editor** and paste the contents of `irrigation_dashboard.yaml`  

---

## ⚙️ Setup

1. Create a new automation using the **Advanced Irrigation** blueprint  
2. Configure:
   - Zones (valves/switches, runtimes, manual selects)  
   - Schedules (fixed, sun, or helper)  
   - Optional: soil, rain, pump, level sensors  
   - Toggles: notifications, debug logging, auto-reset  
3. Save and enable  

---

## ✅ Validation Checklist

- [ ] Confirm **zones** turn on/off correctly when manually triggered  
- [ ] Confirm **fixed schedule** runs (set a near-future time for testing)  
- [ ] Confirm **sun schedule** (set offset to 0 and compare with actual sunrise/sunset)  
- [ ] Confirm **schedule helper** triggers irrigation when enabled  
- [ ] Confirm **pump** activates with delay (if configured)  
- [ ] Confirm **soil/rain aborts** stop watering when thresholds exceeded  
- [ ] Confirm **notifications** appear in HA sidebar  
- [ ] Confirm **water usage** values populate sensors and utility meters  
- [ ] Confirm **dashboard cards** show current status and history  

---

## 🛠️ Troubleshooting

- **Error: Missing inputs**  
  → Check that all required entities are assigned when creating the automation.  

- **Automation doesn’t trigger**  
  → Verify schedule booleans are enabled in blueprint config.  

- **Zones not turning off**  
  → Ensure each zone has valid switches/valves assigned.  

- **Water usage not showing**  
  → Confirm you set a non-zero `water_consumption_per_minute`.  

- **Abort not working**  
  → Check that your soil/rain sensor reports values in the correct unit (%, in, mm).  

---

## 📊 Example Dashboard

💡 Optional: Add a screenshot here for GitHub  
Place your PNG/JPG in `/docs/` and link it like this:

```
![Dashboard Preview](docs/dashboard_example.png)
```

---

## 🧩 Future Enhancements

- MQTT integration for external controllers  
- Weather forecast-based watering prediction  
- HA companion app push notifications  

---

## 📜 License

MIT License. Use freely and adapt for your system 🌍
