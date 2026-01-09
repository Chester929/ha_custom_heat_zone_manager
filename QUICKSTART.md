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

### Step 2: Create Automation (2 minutes)

1. Click **Create Automation**
2. Select **Use a blueprint**
3. Choose **Floor Heating Valve Manager**
4. Fill in the basics:

   **MAIN Thermostat (REQUIRED):**
   - Select your main HVAC climate entity
   - Example: `climate.main_thermostat`

   **Zone 1 - Climate Entity:**
   - Select first room's climate entity
   - Example: `climate.bedroom`

   **Zone 2 - Climate Entity:**
   - Select second room's climate entity
   - Example: `climate.bathroom`

5. Leave other settings at defaults for now
6. Name it: "Floor Heating - Auto Manager"
7. Click **Save**

### Step 3: Done! (1 minute)

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
| Valve Transition Delay | 5 sec | Time for valves to fully open before closing others |

**Note:** The blueprint automatically recalculates every 60 seconds.

## 🎯 Common Setups

### 2-Zone House (Bedroom + Bathroom)

```yaml
MAIN Thermostat: climate.main_hvac
Zone 1: climate.bedroom
Zone 2: climate.bathroom
All others: Leave empty
```

### 3-Zone House with Override Sensor

```yaml
MAIN Thermostat: climate.main_hvac
Zone 1: climate.bedroom
Zone 2: climate.bathroom
Zone 2 Temp Sensor: sensor.bathroom_temperature_accurate
Zone 3: climate.living_room
```

### 4-Zone House with Manual Valves

```yaml
MAIN Thermostat: climate.main_hvac
Zone 1: climate.bedroom
Zone 1 Valve: switch.bedroom_floor_valve
Zone 2: climate.bathroom
Zone 2 Valve: switch.bathroom_floor_valve
Zone 3: climate.living_room
Zone 4: climate.kitchen
```

## 🔧 Quick Tweaks

### More responsive behavior
- Reduce **Open Valve Threshold** to `0.3°C`
- Reduce **Close Valve Threshold** to `0.1°C`

### More stable operation  
- Increase **Open Valve Threshold** to `1.0°C`
- Increase **Close Valve Threshold** to `0.5°C`

**Note:** The blueprint runs automatically every 60 seconds to recalculate valve states.

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
**Solution:** Increase update interval to 120+ seconds

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
