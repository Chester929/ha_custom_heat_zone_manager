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
4. Review trigger entities - ensure they're reporting state changes

### Valves don't open/close as expected

**Problem**: Valves don't respond or behave incorrectly.

**Solutions**:
1. Check temperature thresholds are appropriate for your system
   - Default: open at 0.5°C below, close at 0.2°C above
   - Adjust if needed based on sensor accuracy
2. Verify climate entities support the `climate.set_hvac_mode` service
3. If using manual valve overrides, ensure they support `homeassistant.turn_on/turn_off`
4. Check that valve entities are working independently

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

**Problem**: Automations triggers constantly, causing excessive wear.

**Solutions**:
1. Increase the update interval (default: 60 seconds)
2. Increase temperature thresholds to reduce sensitivity
3. Check for temperature sensor fluctuations
4. Consider using sensors with better accuracy/stability

## Debugging

### Enable detailed logging

Add this to your `configuration.yaml`:

```yaml
logger:
  default: info
  logs:
    homeassistant.components.automation: debug
```

Then restart Home Assistant.

### Check the Logbook

The blueprint logs important calculations:

1. Go to Settings → System → Logbook
2. Filter by "Floor Heating Valve Manager"
3. Review entries like:
   ```
   Recalculating valve states and MAIN temperature.
   Mode: Heating. Zones needing action: 2. All satisfied: False. Target MAIN temp: 25.0°C.
   ```

### Check automation traces

1. Go to Settings → Automations & Scenes
2. Find your Floor Heating automation
3. Click the three dots → Traces
4. Review recent executions to see:
   - Which triggers fired
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

**A**: The blueprint supports up to 5 zones. If you need more, you can:
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

### Optimize update interval

- **Fast response needed**: 30 seconds
- **Standard**: 60 seconds (recommended)
- **Conservative**: 120+ seconds

### Optimize temperature thresholds

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
