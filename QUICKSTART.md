# Quick Start Guide

Get your Floor Heating Valve Manager up and running in 5 minutes!

## 🚀 Quick Install

### Step 1: Import Blueprint (1 minute)

1. Go to **Settings** → **Automations & Scenes** → **Blueprints**
2. Click **Import Blueprint** (bottom right)
3. Paste this URL:
   ```
   https://github.com/Chester929/ha_custom_heat_zone_manager/blob/main/heat_zone_manager.yaml
   ```
4. Click **Import**

### Step 2: Create Virtual Switches (2 minutes)

⚠️ **REQUIRED STEP - Do NOT skip!**

For each zone, you need a virtual switch (helper):

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle**
3. Create switches for your zones:
   - Name: `bedroom_virtual_valve`
   - Name: `bathroom_virtual_valve`
   - (Add more as needed)

### Step 3: Configure Generic Thermostats (3 minutes)

Add this to your `configuration.yaml`:

```yaml
input_boolean:
  bedroom_virtual_valve:
    name: "Bedroom Virtual Valve"
  bathroom_virtual_valve:
    name: "Bathroom Virtual Valve"

climate:
  - platform: generic_thermostat
    name: Bedroom Thermostat
    heater: input_boolean.bedroom_virtual_valve  # Use VIRTUAL switch!
    target_sensor: sensor.bedroom_temperature
  - platform: generic_thermostat
    name: Bathroom Thermostat
    heater: input_boolean.bathroom_virtual_valve  # Use VIRTUAL switch!
    target_sensor: sensor.bathroom_temperature
```

Restart Home Assistant to apply changes.

### Step 4: Create Automation (2 minutes)

1. Click **Create Automation**
2. Select **Use a blueprint**
3. Choose **Floor Heating Valve Manager**
4. Fill in the configuration:

   **MAIN Thermostat (REQUIRED):**
   - Select your main HVAC climate entity
   - Example: `climate.main_thermostat`

   **Zone 1 - Climate Entity:**
   - Select first room's climate entity
   - Example: `climate.bedroom`

   **Zone 1 - Physical Valve Entity (REQUIRED):**
   - Select the physical valve switch
   - Example: `switch.bedroom_valve`

   **Zone 1 - Virtual Switch (REQUIRED):**
   - Select the virtual switch you created
   - Example: `input_boolean.bedroom_virtual_valve`

   **Zone 2 - Climate Entity:**
   - Select second room's climate entity
   - Example: `climate.bathroom`

   **Zone 2 - Physical Valve Entity (REQUIRED):**
   - Example: `switch.bathroom_valve`

   **Zone 2 - Virtual Switch (REQUIRED):**
   - Example: `input_boolean.bathroom_virtual_valve`

5. Leave other settings at defaults for now
6. Name it: "Floor Heating - Auto Manager"
7. Click **Save**

### Step 5: Done! (1 minute)

✅ Your automation is now running!

**Verify it's working:**
1. Go to **Settings** → **System** → **Logbook**
2. Look for "Floor Heating Valve Manager" entries
3. Adjust a zone thermostat and watch it respond

## 📊 Basic Configuration

Use these recommended starting values:

| Setting | Value | Why |
|---------|-------|-----|
| Open Valve Threshold | 0.5°C | Opens valve when 0.5°C below target |
| Close Valve Threshold | 0.2°C | Closes valve when 0.2°C above target |
| All Satisfied Mode | 50% | Use average temperature |
| Min MAIN Temperature | 18°C | Safe minimum |
| Max MAIN Temperature | 28°C | Safe maximum |
| Fallback Temperature | 20°C | Default if sensor fails |
| Valve Transition Delay | 5-120 sec | Time for valves to fully open before closing others (adjust for your valve speed) |

**Trigger System:** The blueprint responds immediately to temperature changes via state-change triggers (1-2 second response).

## 🎯 Common Setups

### 2-Zone House (Bedroom + Bathroom)

```yaml
# In configuration.yaml:
input_boolean:
  bedroom_virtual_valve:
    name: "Bedroom Virtual Valve"
  bathroom_virtual_valve:
    name: "Bathroom Virtual Valve"

climate:
  - platform: generic_thermostat
    name: Bedroom
    heater: input_boolean.bedroom_virtual_valve
    target_sensor: sensor.bedroom_temperature
  - platform: generic_thermostat
    name: Bathroom
    heater: input_boolean.bathroom_virtual_valve
    target_sensor: sensor.bathroom_temperature

# In blueprint configuration:
MAIN Thermostat: climate.main_hvac
Zone 1 Climate: climate.bedroom
Zone 1 Valve: switch.bedroom_valve
Zone 1 Virtual Switch: input_boolean.bedroom_virtual_valve
Zone 2 Climate: climate.bathroom
Zone 2 Valve: switch.bathroom_valve
Zone 2 Virtual Switch: input_boolean.bathroom_virtual_valve
All others: Leave empty
```

### 3-Zone House with Corridor Compensation (RECOMMENDED)

**Best for:** MAIN sensor in corridor, bedrooms often cooler than corridor

```yaml
# In configuration.yaml:
input_boolean:
  bedroom1_virtual_valve:
    name: "Bedroom 1 Virtual Valve"
  bedroom2_virtual_valve:
    name: "Bedroom 2 Virtual Valve"
  bathroom_virtual_valve:
    name: "Bathroom Virtual Valve"

climate:
  - platform: generic_thermostat
    name: Bedroom 1
    heater: input_boolean.bedroom1_virtual_valve
    target_sensor: sensor.bedroom1_temperature
  - platform: generic_thermostat
    name: Bedroom 2
    heater: input_boolean.bedroom2_virtual_valve
    target_sensor: sensor.bedroom2_temperature
  - platform: generic_thermostat
    name: Bathroom
    heater: input_boolean.bathroom_virtual_valve
    target_sensor: sensor.bathroom_temperature

# In blueprint configuration:
MAIN Thermostat: climate.main_hvac
MAIN Temp Sensor: sensor.corridor_temperature  # KEY: Enables intelligent compensation
Zone 1 Climate: climate.bedroom1
Zone 1 Temp Sensor: sensor.bedroom1_temperature
Zone 1 Valve: switch.bedroom1_valve
Zone 1 Virtual Switch: input_boolean.bedroom1_virtual_valve
Zone 2 Climate: climate.bedroom2
Zone 2 Temp Sensor: sensor.bedroom2_temperature
Zone 2 Valve: switch.bedroom2_valve
Zone 2 Virtual Switch: input_boolean.bedroom2_virtual_valve
Zone 3 Climate: climate.bathroom
Zone 3 Valve: switch.bathroom_valve
Zone 3 Virtual Switch: input_boolean.bathroom_virtual_valve
```

**Why this works better:**
When corridor is 23°C and bedroom needs 22°C but is at 20°C, the blueprint automatically compensates by setting MAIN target higher (e.g., 23°C) to ensure adequate heating despite the warm corridor.

## 🔧 Quick Tweaks

### More responsive behavior
- Reduce **Open Valve Threshold** to `0.3°C`
- Reduce **Close Valve Threshold** to `0.1°C`

### More stable operation  
- Increase **Open Valve Threshold** to `1.0°C`
- Increase **Close Valve Threshold** to `0.5°C`

### Use more conservative heating
- Set **All Satisfied Mode** to `0%` (lowest)
- Increase **Min MAIN Temperature** to `19°C`

### Use more aggressive heating
- Set **All Satisfied Mode** to `100%` (highest)
- Decrease **Close Valve Threshold** to `0.1°C`

## ❓ Quick Troubleshooting

### Problem: Automation doesn't trigger
**Solution:** Check that zones are configured and automation is enabled (toggle should be ON)

### Problem: MAIN thermostat doesn't change
**Solution:** Verify MAIN thermostat entity supports `climate.set_temperature` service

### Problem: Valves don't open/close
**Solution:** Check temperature thresholds - they might be too large or too small

### Problem: Too many valve switches
**Solution:** Increase the **Open Valve Threshold** and **Close Valve Threshold** values to reduce sensitivity and prevent frequent valve switching.

## 📖 Next Steps

Once you have it working:

1. **Monitor for a few days** - Check logbook entries
2. **Fine-tune thresholds** - Adjust based on your system's behavior
3. **Read full docs** - Check [README.md](README.md) for advanced features
4. **Try examples** - See [examples/example_configuration.yaml](examples/example_configuration.yaml)

## 🆘 Need Help?

- 📘 [Installation Guide](INSTALLATION.md) - Detailed setup
- 🔧 [Troubleshooting](TROUBLESHOOTING.md) - Common issues
- 🏗️ [Architecture](ARCHITECTURE.md) - How it works
- 💬 [GitHub Issues](https://github.com/Chester929/ha_custom_heat_zone_manager/issues) - Ask questions

## 🎉 You're All Set!

Your floor heating system is now being managed automatically. Enjoy your perfectly heated home! 🏠🔥
