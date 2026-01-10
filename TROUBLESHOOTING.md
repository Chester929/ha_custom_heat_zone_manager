# Troubleshooting Guide

This guide helps resolve common issues when using the Floor Heating Valve Manager blueprint.

## Table of Contents
- [Installation Issues](#installation-issues)
- [Configuration Issues](#configuration-issues)
- [Runtime Issues](#runtime-issues)
- [Debugging](#debugging)
- [FAQ](#faq)

## Installation Issues

### Blueprint doesn't appear after import

**Problem**: After importing the blueprint, it doesn't show up in the Blueprints list.

**Solutions**:
1. Ensure the file is in the correct location: `<config>/blueprints/automation/chester929/heat_zone_manager.yaml`
2. Restart Home Assistant or reload automations:
   - Go to Developer Tools → YAML → Automations
3. Check the Home Assistant logs for YAML errors:
   - Settings → System → Logs

### Import URL fails

**Problem**: Importing via URL shows an error.

**Solutions**:
1. Ensure you have internet connectivity
2. Use the raw GitHub URL:
   ```
   https://raw.githubusercontent.com/Chester929/ha_custom_heat_zone_manager/main/heat_zone_manager.yaml
   ```
3. Alternatively, manually download and place the file in the blueprints folder

## Configuration Issues

### "Entity not found" error

**Problem**: Configuration shows errors about missing entities.

**Solutions**:
1. Verify entity IDs in Developer Tools → States
2. Ensure entities exist before configuring the automation
3. Check that entity domains match (e.g., `climate.` for climate entities)
4. Leave unused zone fields empty (don't delete them)

### Automation doesn't trigger

**Problem**: The automation never runs or doesn't respond to changes.

**Solutions**:
1. Check that at least one zone is configured
2. Verify the MAIN thermostat entity is correct
3. Check automation is enabled:
   - Settings → Automations & Scenes
   - Find your automation and ensure it's toggled ON
4. Review the automation trace to confirm it's running every minute

### Automation stops immediately with "only one execution allowed"

**Problem**: The automation trace shows it stopped with "Stopped because only one execution is allowed" or similar message, especially when triggered by time pattern.

**Root Cause**: This issue occurred in older versions (v2.0.0 and earlier) due to `mode: single` blocking new triggers while the automation was running.

**Solution**: Update to the latest version which uses `mode: queued`:
1. Re-import the blueprint from GitHub
2. The automation will now use `mode: queued` with `max: 10`
3. This allows new triggers to queue up and execute sequentially instead of being blocked
4. The built-in duplicate prevention logic ensures efficiency is maintained

**Technical Details**:
- **Old behavior**: `mode: single` + `max_exceeded: silent` → New triggers blocked and silently ignored
- **New behavior**: `mode: queued` + `max: 10` → New triggers queue up and execute in order
- The duplicate prevention logic (early exit) ensures unnecessary work is still avoided

### Valves don't open/close as expected

**Problem**: Valves don't respond or behave incorrectly.

**Solutions**:
1. Check temperature thresholds are appropriate for your system
   - Default: open at 0.5°C below, close at 0.2°C above
   - Adjust if needed based on sensor accuracy
2. Verify climate entities support the `climate.set_hvac_mode` service
3. If using manual valve overrides, ensure they support `homeassistant.turn_on/turn_off`
4. Check that valve entities are working independently

### Climate entity conflicts with blueprint valve control

**Problem**: Zone climate entities (e.g., Generic Thermostat) have their own internal logic that controls valves based on current vs target temperature. This **WILL conflict** with the blueprint's valve control decisions.

**THE SOLUTION: Virtual Switch Pattern (MANDATORY)**

**⚠️ REQUIRED CONFIGURATION**

For each zone you configure, you MUST specify **BOTH** the physical valve AND virtual switch parameters:

- ✅ **BOTH specified** → Virtual switch pattern (REQUIRED for all zones)
- ❌ **Only one specified** → INVALID configuration (blueprint will log error)
- ℹ️ **Zone not used** → Leave zone climate entity empty to skip that zone

**Why Both Are Required:**

Generic Thermostat MUST have a heater entity configured. If you only specify the physical valve in the blueprint, the Generic Thermostat would still try to control it directly, causing conflicts. The virtual switch pattern ensures clean separation:

- Generic Thermostat → Controls virtual switch (based on temperature)
- Blueprint → Controls physical valve (based on virtual switch + coordination)

Use a virtual/helper switch as an intermediary between the climate entity and the physical valve:

```
Climate Entity (Generic Thermostat)
    ↓ (controls based on temp)
Virtual Switch (input_boolean or switch helper)
    ↓ (monitored by blueprint)
Blueprint Logic (considers virtual switch + coordination)
    ↓ (final control)
Physical Valve (actual heating valve)
```

**How It Works:**

1. **Generic Thermostat** controls a **virtual switch** based on temperature
   - Virtual switch is just a helper entity in Home Assistant (not a physical device)
   - Climate entity keeps its real target temperature
   - Climate entity's logic determines when virtual switch should be on/off

2. **Blueprint monitors the virtual switch** to understand what the climate entity wants
   - Reads virtual switch state as "zone request"
   - Combines this with temperature data and coordination logic
   - Makes final decision about physical valve

3. **Blueprint controls physical valve** directly
   - Has final authority over actual heating
   - Can override climate entity when needed for coordination
   - Ensures at least one valve always open (pump safety)

**Benefits:**
- ✅ Climate entities keep real target temperatures (you can see and adjust them)
- ✅ No conflicts - climate entity controls virtual switch, blueprint controls physical valve
- ✅ Blueprint can coordinate across zones
- ✅ Climate entity's "wants" are respected but not blindly followed
- ✅ Clean architecture and easy to debug

**Setup Instructions:**

**Step 1: Create Virtual Switches**

For each zone, create a helper switch in Home Assistant:

```yaml
# configuration.yaml
input_boolean:
  bedroom_virtual_valve:
    name: "Bedroom Virtual Valve"
    icon: mdi:valve
  
  bathroom_virtual_valve:
    name: "Bathroom Virtual Valve"
    icon: mdi:valve
```

Or use UI: Settings → Devices & Services → Helpers → Create Helper → Toggle

**Step 2: Configure Generic Thermostat to Control Virtual Switch**

```yaml
# configuration.yaml
climate:
  - platform: generic_thermostat
    name: Bedroom Thermostat
    heater: input_boolean.bedroom_virtual_valve  # Virtual switch, not physical!
    target_sensor: sensor.bedroom_temperature
    min_temp: 15
    max_temp: 25
    
  - platform: generic_thermostat
    name: Bathroom Thermostat
    heater: input_boolean.bathroom_virtual_valve  # Virtual switch, not physical!
    target_sensor: sensor.bathroom_temperature
    min_temp: 15
    max_temp: 25
```

**Step 3: Configure Blueprint**

```yaml
# Blueprint configuration
zone1_climate: climate.bedroom_thermostat        # Reads temp and target
zone1_temp_sensor: sensor.bedroom_temperature    # Optional override
zone1_valve: switch.bedroom_physical_valve       # Controls PHYSICAL valve
zone1_virtual_switch: input_boolean.bedroom_virtual_valve  # Monitors climate entity's wants

zone2_climate: climate.bathroom_thermostat
zone2_temp_sensor: sensor.bathroom_temperature
zone2_valve: switch.bathroom_physical_valve      # Controls PHYSICAL valve
zone2_virtual_switch: input_boolean.bathroom_virtual_valve
```

**How the Blueprint Makes Decisions:**

```
For each zone:
  1. Check temperature: Does it need heating? (current < target - threshold)
  2. Check virtual switch: Does climate entity want heating? (virtual switch = on)
  3. Combine: Zone needs action if BOTH temperature needs it AND virtual switch is on
  4. Apply coordination: Consider all zones, safety constraints
  5. Control physical valve: Turn on/off based on final decision
```

**Example Scenario:**

```
Bedroom: 20°C, target 22°C
  → Generic Thermostat: "I need heating" → Turns ON virtual switch
  → Blueprint sees: temp needs heating (20<22-0.5) AND virtual switch ON
  → Blueprint opens physical bedroom valve

Bathroom: 25°C, target 25°C  
  → Generic Thermostat: "I'm satisfied" → Turns OFF virtual switch
  → Blueprint sees: temp satisfied AND virtual switch OFF
  → Blueprint closes physical bathroom valve (unless needed for pump safety)

All zones satisfied:
  → All virtual switches OFF
  → Blueprint: Opens fallback zone's physical valve (pump safety)
  → Coordination overrides individual zone requests when necessary
```

**Without Virtual Switch (UNSUPPORTED CONFIGURATION)**

Virtual switches are **mandatory** for this blueprint. Running the blueprint without virtual switches is **not supported** and will lead to conflicts between the climate entities and the physical valves.

If the blueprint is misconfigured without a virtual switch, you will experience one of the following unsupported scenarios:

1. **Physical valve override only** (no virtual switch configured):
   - Only the physical valve parameter is configured
   - Blueprint controls the physical valve based on temperature only
   - Climate entity will also try to control the same valve → **CONFLICT**
   - Result: Unpredictable behavior, valve fighting between controllers

2. **Climate entity controlled directly** (no valve override and no virtual switch configured):
   - No valve or virtual switch is specified  
   - Blueprint attempts to set the climate entity to heat/cool/off modes
   - Climate entity controls its own valve independently
   - Result: Climate entity may override the blueprint's coordinated decisions → **CONFLICT**

**Solution**: Update your configuration to include BOTH a virtual switch and physical valve for each zone, as described in the installation and configuration sections. The virtual switch pattern is the only supported configuration mode.

## Runtime Issues

### All valves close (should never happen!)

**Problem**: The critical constraint is violated - all valves are closed.

**Solutions**:
1. This should never happen if the blueprint is working correctly
2. Check the logbook for error messages
3. Verify all zone climate entities are responding
4. Ensure the blueprint automation is enabled
5. Check for conflicting automations that might override valve states

**Emergency workaround**:
Create a separate automation to force at least one valve open if all are closed.

### MAIN thermostat doesn't update

**Problem**: The MAIN thermostat temperature doesn't change.

**Solutions**:
1. Verify the MAIN thermostat supports `climate.set_temperature` service
2. Check min/max temperature limits aren't too restrictive
3. Ensure MAIN thermostat is not in a mode that prevents temperature changes
4. Review logbook entries to see calculated temperatures

### Temperature calculations seem wrong

**Problem**: The calculated MAIN temperature doesn't match expectations.

**Solutions**:
1. Review the "All Zones Satisfied" slider setting
   - 0% = lowest target
   - 50% = average
   - 100% = highest target
2. Check that zone target temperatures are set correctly
3. Verify temperature sensors are reporting accurate values
4. Review the heating/cooling mode setting

### Automation runs too frequently

**Problem**: Automation seems to execute too often.

**Note**: The automation runs every minute by design (time-pattern trigger). This is appropriate for HVAC control and provides:
- Predictable execution intervals
- Lower system load than state-change monitors
- Sufficient responsiveness for heating/cooling (thermal systems respond slowly)

**If concerned about execution frequency**:
1. Check automation traces to verify it's running once per minute (expected behavior)
2. The automation includes built-in checks to skip unnecessary API calls (e.g., MAIN thermostat only updated when temperature difference exceeds 0.1°C)
3. Valve operations already check current state before opening/closing

**Note**: One execution per minute is normal and expected for time-based operation.

## Debugging

### Enable detailed logging

Add this to your `configuration.yaml`:

```yaml
logger:
  default: info
  logs:
    homeassistant.components.automation: debug
    blueprints.floor_heating_valve_manager: debug
```

Then restart Home Assistant.

### Check the Logbook

The blueprint logs important calculations:

1. Go to Settings → System → Logbook
2. Filter by "Floor Heating Valve Manager"
3. Review entries like:
   ```
   Evaluating valve states and MAIN temperature.
   Mode: Heating. Zones needing action: 2. All satisfied: False. Target MAIN temp: 25.0°C (current: 24.5°C, needs change: true).
   ```

### Check automation traces

1. Go to Settings → Automations & Scenes
2. Find your Floor Heating automation
3. Click the three dots → Traces
4. Review recent executions to see:
   - Trigger source (time_pattern trigger every minute)
   - Variable calculations
   - Variable calculations
   - Actions taken

### Test with developer tools

Test individual components:

```yaml
# Test MAIN thermostat
service: climate.set_temperature
target:
  entity_id: climate.main_hvac
data:
  temperature: 22

# Test valve control
service: climate.set_hvac_mode
target:
  entity_id: climate.bedroom_thermostat
data:
  hvac_mode: heat
```

## FAQ

### Q: How many zones can I configure?

**A**: The blueprint supports up to 15 zones (organized in 3 groups of 5). If you need more, you can:
1. Create multiple instances of the automation
2. Modify the blueprint to add more zones
3. Group some zones together

### Q: Can I use this with non-floor heating systems?

**A**: Yes! The blueprint works with any heating/cooling system that:
- Has a MAIN thermostat controlling the heat source
- Has zone thermostats or valve controls
- Requires at least one valve to be open

### Q: What if my zones don't have climate entities?

**A**: You can use the manual override options:
- Create a generic thermostat integration for each zone
- Use the temperature sensor override for accurate readings
- Use the valve override to control actual physical valves

### Q: Does this work with Google Nest or Ecobee?

**A**: Yes, as long as:
- The MAIN thermostat is integrated with Home Assistant
- You can control zone thermostats via HA
- The thermostats support the standard climate entity interface

### Q: Can I use this for radiator heating?

**A**: Absolutely! The same logic applies:
- MAIN thermostat controls the boiler
- Zone thermostats control radiator valves (TRVs)
- Prevents all valves from closing

### Q: What about mixed heating and cooling?

**A**: The blueprint supports either heating OR cooling mode (not both simultaneously). For mixed mode:
- Create two automation instances (one for heating, one for cooling)
- Add conditions to enable/disable based on season
- Use a helper to switch between modes

### Q: How do I handle zones that don't need heating?

**A**: Simply leave those zone configurations empty in the blueprint. The automation will only manage configured zones.

### Q: Can I override the automation manually?

**A**: Yes, you can:
1. Disable the automation temporarily
2. Manually control thermostats and valves
3. Re-enable when ready to resume automatic control

### Q: Does this affect my existing automations?

**A**: The blueprint controls:
- MAIN thermostat target temperature
- Zone climate entity HVAC modes
- Zone valve states (if overridden)

Other automations that modify these may conflict. Ensure you don't have competing automations.

## Getting Help

If you're still experiencing issues:

1. **Check the logs**: Settings → System → Logs
2. **Review automation traces**: Settings → Automations → Your automation → Traces
3. **Search existing issues**: [GitHub Issues](https://github.com/Chester929/ha_custom_heat_zone_manager/issues)
4. **Open a new issue**: Include:
   - Home Assistant version
   - Blueprint version
   - Relevant log entries
   - Your configuration (remove sensitive data)
   - Description of the problem
   - Steps to reproduce

## Performance Tips

### Optimize temperature thresholds

The blueprint uses time-pattern triggers for reliable, predictable operation.

**Response Time:** The time-pattern trigger evaluates every minute (60 seconds), providing appropriate responsiveness for HVAC control. Heating systems have thermal inertia and respond over minutes, not seconds.

**Temperature threshold settings:**

- **Precise control**: 0.2-0.3°C difference
- **Standard**: 0.5°C open, 0.2°C close (recommended)
- **Stable operation**: 1.0°C or higher

### Reduce automation complexity

- Only configure zones you need
- Remove unused overrides
- Use built-in climate entity sensors when possible

## Advanced Scenarios

### Multiple heating zones with priority

Create separate automations with conditions:
- Priority zones: Lower temperature thresholds
- Secondary zones: Higher temperature thresholds

### Time-based heating schedules

Combine with schedule helpers:
- Use conditions to enable/disable zones by time
- Adjust temperature targets based on schedule

### Presence-based control

Add conditions based on occupancy:
- Disable zones when rooms are unoccupied
- Reduce targets in vacant areas

### Weather-based adjustments

Use weather conditions to:
- Adjust temperature thresholds
- Modify the all-satisfied temperature calculation
- Enable/disable zones based on forecast
