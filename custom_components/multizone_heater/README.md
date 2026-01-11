# Multizone Heater Custom Component

This directory contains the Multizone Heater custom component for Home Assistant. This is a Python-based integration that efficiently manages multizone heating/cooling systems with individual zone valves, temperature sensors, and optional zone climate entities.

## Installation

### Option 1: Copy to Custom Components Directory
1. Copy the entire `custom_components/multizone_heater` directory to your Home Assistant's `custom_components` directory
2. Restart Home Assistant
3. Go to **Settings** → **Devices & Services** → **Add Integration**
4. Search for "Multizone Heater" and follow the configuration steps

### Option 2: HACS (Future)
This integration will be available through HACS in the future.

## Features

- **Async Operation**: All valve control operations are executed asynchronously in parallel for maximum performance
- **Multiple Zone Support**: Configure unlimited heating/cooling zones
- **Zone Climate Entities**: Support for zone climate entities with virtual switch pattern
- **Temperature Aggregation**: Choose how to aggregate zone temperatures (average, minimum, maximum, or percentage-based)
- **Cooling Mode Support**: Automatically detects and supports cooling if main climate entity has cooling capability
- **Safety Features**: Ensures a minimum number of valves remain open to protect heating systems
- **Fallback Zones**: Mandatory fallback zones ensure pump safety during cooling and when all zones are satisfied

## Configuration

The integration is configured through the Home Assistant UI:

1. **Main Settings**
   - Main Climate Entity (optional)
   - Temperature Aggregation Method
   - Minimum Valves Open (default: 1)

2. **Add Zones**
   - Zone Name
   - Zone Climate Entity (optional)
   - Temperature Sensor Override (optional)
   - Physical Valve Switch (optional)
   - Virtual Switch (optional)
   - Opening/Closing Temperature Offsets

3. **Fallback Zones**
   - Select at least one zone to keep open when all zones are satisfied

## Bug Fixes

### v1.0.1 - TypeError Fix
Fixed a critical issue where `min_valves_open` configuration value could be passed as a float, causing a TypeError when used with `range()` function. The fix ensures the value is properly converted to an integer at initialization.

See [FIX_SUMMARY.md](../../FIX_SUMMARY.md) for detailed information about this fix.

## Documentation

For more information about the Multizone Heater integration, see:
- [GitHub Repository](https://github.com/Chester929/ha_multizone_heater) - Original repository
- [Home Assistant Blueprint](https://github.com/Chester929/ha_custom_heat_zone_manager) - Blueprint version

## Support

If you encounter any issues, please report them on the [GitHub Issues page](https://github.com/Chester929/ha_custom_heat_zone_manager/issues).
