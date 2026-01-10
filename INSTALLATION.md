# Installation Guide

Step-by-step instructions for installing and setting up the Floor Heating Valve Manager blueprint.

## Prerequisites

Before installing, ensure you have:

- ✅ Home Assistant 2024.1.0 or newer
- ✅ A MAIN thermostat integrated with Home Assistant (climate entity)
- ✅ At least one zone thermostat or valve control (climate entity or switch)
- ✅ Basic understanding of Home Assistant automations

## Installation Methods

### Method 1: Import via URL (Recommended)

This is the easiest method and ensures you always get updates.

1. **Open Home Assistant**
   - Go to **Settings** → **Automations & Scenes**
   
2. **Navigate to Blueprints**
   - Click the **Blueprints** tab
   
3. **Import Blueprint**
   - Click the blue **Import Blueprint** button (bottom right)
   
4. **Paste URL**
   ```
   https://github.com/Chester929/ha_custom_heat_zone_manager/blob/main/heat_zone_manager.yaml
   ```
   
5. **Preview and Import**
   - Review the blueprint preview
   - Click **Import Blueprint**

6. **Success!**
   - The blueprint is now available for use

### Method 2: Manual File Installation

Use this method if you want to customize the blueprint or don't have internet access.

1. **Download the Blueprint**
   - Download `heat_zone_manager.yaml` from the [repository](https://github.com/Chester929/ha_custom_heat_zone_manager)
   
2. **Create Directory Structure**
   ```bash
   cd /config  # or wherever your Home Assistant config is
   mkdir -p blueprints/automation/chester929
   ```
   
3. **Copy the File**
   ```bash
   cp heat_zone_manager.yaml blueprints/automation/chester929/
   ```
   
4. **Reload Automations**
   - Go to **Developer Tools** → **YAML**
   - Click **Automations** under "Configuration validation"
   
5. **Verify Installation**
   - Go to **Settings** → **Automations & Scenes** → **Blueprints**
   - You should see "Floor Heating Valve Manager"

### Method 3: Using SSH/File Editor Add-on

If you use the SSH or File Editor add-on:

1. **Open File Editor** or SSH into Home Assistant
   
2. **Navigate to blueprints folder**
   ```
   /config/blueprints/automation/
   ```
   
3. **Create folder**
   ```
   chester929/
   ```
   
4. **Create file**
   - Create `heat_zone_manager.yaml` in the folder
   - Paste the blueprint content from GitHub
   
5. **Save and reload**
   - Save the file
   - Developer Tools → YAML → Reload Automations

## Setting Up Your First Automation

After installation, create your first automation:

### Step 1: Create Automation from Blueprint

1. Go to **Settings** → **Automations & Scenes**
2. Click **Create Automation**
3. Click **Use a blueprint**
4. Find and select **Floor Heating Valve Manager**

### Step 2: Configure MAIN Thermostat

1. **MAIN Thermostat (REQUIRED)**
   - Select your primary HVAC climate entity
   - Example: `climate.main_hvac` or `climate.de_dietrich_main`

### Step 3: Configure Zones

⚠️ **IMPORTANT: Virtual Switch Pattern REQUIRED**

For each zone you configure, you **MUST** specify **BOTH** the physical valve AND virtual switch parameters. This is mandatory to prevent conflicts between the climate entity and the blueprint.

**Setup Process for Each Zone:**

1. **Create Virtual Switch** (if not already created)
   - Go to **Settings** → **Devices & Services** → **Helpers**
   - Click **Create Helper** → **Toggle**
   - Name it: `bedroom_virtual_valve` (or similar for your zone)

2. **Configure Generic Thermostat** (in configuration.yaml)
   ```yaml
   climate:
     - platform: generic_thermostat
       name: Bedroom Thermostat
       heater: input_boolean.bedroom_virtual_valve  # VIRTUAL switch, not physical!
       target_sensor: sensor.bedroom_temperature
   ```

3. **Configure Zone in Blueprint:**
   - **Zone 1 - Climate Entity**: `climate.bedroom_thermostat`
   - **Zone 1 - Temperature Sensor** (Optional): `sensor.bedroom_temperature` or leave empty
   - **Zone 1 - Physical Valve Entity** (**REQUIRED**): `switch.bedroom_physical_valve`
   - **Zone 1 - Virtual Switch** (**REQUIRED**): `input_boolean.bedroom_virtual_valve`

**Why Both Are Required:**
- Generic Thermostat entities MUST have a heater configured (cannot be disabled)
- To prevent conflicts, the climate entity controls the virtual switch
- The blueprint monitors the virtual switch and controls the physical valve
- This ensures clean separation and proper coordination across zones

Repeat for additional zones (up to 15 zones available, organized in 3 collapsible UI groups for easier navigation).

### Step 4: Configure Temperature Settings

**Recommended starting values:**

1. **Temperature Difference to Open Valve**: `0.5°C`
   - Valve opens when current temp is 0.5°C below target

2. **Temperature Difference to Close Valve**: `0.2°C`
   - Valve closes when current temp is 0.2°C above target

3. **All Zones Satisfied - Temperature Mode**: `50%`
   - When all zones satisfied, use average temperature
   - 0% = lowest, 50% = average, 100% = highest

4. **Minimum MAIN Temperature**: `18.0°C`

5. **Maximum MAIN Temperature**: `28.0°C`

### Step 5: Advanced Settings (Optional)

1. **Enable Cooling Mode Support**: `Off`
   - Turn on only if your system supports cooling

### Step 6: Save and Enable

1. **Name your automation**
   - Example: "Floor Heating - Automatic Valve Manager"

2. **Click Save**

3. **Automation is now active!**

## Verifying Installation

### Check Logbook

1. Go to **Settings** → **System** → **Logbook**
2. Look for entries from "Floor Heating Valve Manager"
3. You should see messages like:
   ```
   Recalculating valve states and MAIN temperature.
   Mode: Heating. Zones needing action: 2. All satisfied: False. Target MAIN temp: 25.0°C.
   ```

### Check Automation Traces

1. Go to **Settings** → **Automations & Scenes**
2. Find your Floor Heating automation
3. Click the three dots → **Traces**
4. You should see recent executions

### Test a Zone Change

1. Manually adjust a zone thermostat target temperature
2. The automation responds immediately via state-change trigger (1-2 seconds)
3. Check that:
   - The zone's valve state changed (if needed)
   - The MAIN thermostat target was updated
   - A logbook entry was created

## Example Configuration

Here's a complete basic example with virtual switch pattern:

```yaml
# First, create virtual switches in configuration.yaml:
input_boolean:
  bedroom_virtual_valve:
    name: "Bedroom Virtual Valve"
  bathroom_virtual_valve:
    name: "Bathroom Virtual Valve"

# Configure Generic Thermostats to use virtual switches:
climate:
  - platform: generic_thermostat
    name: Bedroom Thermostat
    heater: input_boolean.bedroom_virtual_valve  # VIRTUAL!
    target_sensor: sensor.bedroom_temperature
  - platform: generic_thermostat
    name: Bathroom Thermostat
    heater: input_boolean.bathroom_virtual_valve  # VIRTUAL!
    target_sensor: sensor.bathroom_temperature

# Then configure the blueprint automation:
automation:
  - alias: Floor Heating - My House
    use_blueprint:
      path: chester929/heat_zone_manager.yaml
      input:
        # MAIN thermostat
        main_thermostat: climate.main_hvac
        
        # Zone 1: Bedroom (BOTH valve and virtual_switch REQUIRED)
        zone1_climate: climate.bedroom_thermostat
        zone1_valve: switch.bedroom_physical_valve
        zone1_virtual_switch: input_boolean.bedroom_virtual_valve
        
        # Zone 2: Bathroom (BOTH valve and virtual_switch REQUIRED)
        zone2_climate: climate.bathroom_thermostat
        zone2_temp_sensor: sensor.bathroom_temperature
        zone2_valve: switch.bathroom_physical_valve
        zone2_virtual_switch: input_boolean.bathroom_virtual_valve
        
        # Settings
        temp_difference_open: 0.5
        temp_difference_close: 0.2
        all_satisfied_temp_mode: 50
        min_main_temp: 18.0
        max_main_temp: 28.0
```

## Next Steps

1. **Monitor operation** for a few days to ensure it works as expected
2. **Adjust thresholds** if needed based on your system's behavior
3. **Fine-tune** the "all satisfied" temperature mode
4. **Read the [Troubleshooting Guide](TROUBLESHOOTING.md)** if you encounter issues
5. **Check [example configurations](examples/example_configuration.yaml)** for more advanced setups

## Common First-Time Issues

### Blueprint doesn't show up
- Reload automations via Developer Tools → YAML
- Restart Home Assistant
- Check file permissions

### "Entity not found" errors
- Verify entity IDs in Developer Tools → States
- Ensure entities exist before configuring
- Check spelling and domain prefixes (climate., sensor., etc.)

### Automation doesn't trigger
- Ensure automation is enabled (toggle switch should be ON)
- Check that zones are configured
- Verify MAIN thermostat is accessible

## Getting Help

If you need assistance:

1. Check the [Troubleshooting Guide](TROUBLESHOOTING.md)
2. Review [example configurations](examples/example_configuration.yaml)
3. Read the [full documentation](README.md)
4. Open an issue on [GitHub](https://github.com/Chester929/ha_custom_heat_zone_manager/issues)

---

**Congratulations!** Your Floor Heating Valve Manager is now installed and running. 🎉
