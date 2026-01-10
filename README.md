# Floor Heating Valve Manager for Home Assistant

[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Blueprint-blue.svg)](https://www.home-assistant.io/)
[![Version](https://img.shields.io/badge/version-2.0.0-green.svg)](https://github.com/Chester929/ha_custom_heat_zone_manager)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

A comprehensive Home Assistant Blueprint for intelligently managing floor heating valves alongside a MAIN thermostat that controls your HVAC water heating system.

## 📚 Documentation

- **[Quick Start Guide](QUICKSTART.md)** - Get running in 5 minutes!
- **[Installation Guide](INSTALLATION.md)** - Detailed setup instructions
- **[Troubleshooting](TROUBLESHOOTING.md)** - Common issues and solutions
- **[Architecture](ARCHITECTURE.md)** - How it works internally
- **[Examples](examples/)** - Example configurations
- **[Contributing](CONTRIBUTING.md)** - How to contribute
- **[Changelog](CHANGELOG.md)** - Version history

## 🎯 Purpose

This blueprint solves a common challenge in floor heating systems: coordinating multiple zone valves with a central HVAC unit that cannot directly control the water pump. It ensures:

- **Never all valves closed**: Critical constraint to prevent pump issues
- **Intelligent temperature management**: Dynamically adjusts MAIN thermostat based on zone demands
- **Flexible zone control**: Supports multiple zones with individual thermostats and optional manual overrides
- **Efficient heating/cooling**: Opens only necessary valves while maintaining system stability

## 🏠 Use Case

Your HVAC unit (e.g., De Dietrich heat pump) heats water based on outdoor and indoor temperature sensors. Individual rooms have thermostats controlling floor heating valves. However:

- You **cannot** directly control the water pump
- The HVAC's indoor sensor is in a corridor (may differ from room temperatures)
- Each room has its own temperature sensor and valve
- You need to coordinate valve states and MAIN thermostat temperature for optimal heating

This blueprint manages the entire system automatically!

## ✨ Features

### Core Functionality
- ✅ Manages up to 15 heating/cooling zones (organized in 3 groups of 5)
- ✅ **Guarantees at least one valve is always open** (critical!)
- ✅ Dynamically calculates MAIN thermostat target temperature
- ✅ **Intelligent temperature compensation** when MAIN sensor location differs from zones
- ✅ Supports both heating and cooling modes
- ✅ Configurable temperature thresholds for valve control

### Zone Configuration
- ✅ Each zone can use a climate entity
- ✅ Optional manual overrides for temperature sensors
- ✅ Optional manual overrides for valve entities
- ✅ Optional MAIN sensor override for accurate corridor temperature
- ✅ Flexible: configure 1-15 zones as needed (organized in 3 groups)

### Temperature Management
- ✅ Adjustable open/close temperature thresholds
- ✅ **Smart compensation algorithm** accounts for temperature differences between corridor and zones
- ✅ Configurable "all zones satisfied" behavior:
  - 0% = Use lowest zone target
  - 50% = Use average of all targets
  - 100% = Use highest zone target
- ✅ Min/max limits for MAIN thermostat
- ✅ Dual-trigger system: instant response (1-2s) + configurable periodic updates

### Safety & Reliability
- ✅ **Availability tracking** - monitors climate entity health
- ✅ **Multi-level safety override** - ensures at least one valve open even if entities unavailable
- ✅ **Configuration validation** - validates zone setup on startup
- ✅ **Warning logging** - alerts when entities become unavailable
- ✅ **Fallback zones** - configurable zones to keep open when all satisfied/overheated
- ✅ **Overheated protection** - closes unnecessary valves when zones too hot

## 📦 Installation

### Method 1: Direct Download

1. Copy the `heat_zone_manager.yaml` file to your Home Assistant configuration:
   ```
   <config>/blueprints/automation/chester929/heat_zone_manager.yaml
   ```

2. Restart Home Assistant or reload automations

3. Go to **Settings** → **Automations & Scenes** → **Blueprints**

4. Click **Import Blueprint** and select "Floor Heating Valve Manager"

### Method 2: Import via URL

1. In Home Assistant, go to **Settings** → **Automations & Scenes** → **Blueprints**

2. Click the import button and paste this URL:
   ```
   https://github.com/Chester929/ha_custom_heat_zone_manager/blob/main/heat_zone_manager.yaml
   ```

## 🔧 Configuration

### ⚠️ CRITICAL: Virtual Switch Pattern REQUIRED

**All zones MUST use the virtual switch pattern - this is mandatory, not optional.**

**REQUIRED Configuration for Each Zone:**

For each zone you configure, you must specify **BOTH** parameters:

- ✅ **Physical Valve Entity** - REQUIRED
- ✅ **Virtual Switch Entity** - REQUIRED  
- ❌ **Only one specified** → INVALID (blueprint will log error)
- ℹ️ **Don't need zone** → Leave climate entity empty to skip that zone

**Why This Is Mandatory:**

Generic Thermostat climate entities MUST have a heater/valve configured. There's no way to "disable" their automatic valve control. The ONLY conflict-free solution is the virtual switch pattern where:

- **Generic Thermostat** controls a virtual switch (based on temperature)
- **Blueprint** controls the physical valve (based on virtual switch + coordination logic)

This ensures clean separation with no conflicts while preserving real target temperatures.

**The Virtual Switch Pattern:**

```
Generic Thermostat → Virtual Switch → Blueprint → Physical Valve
```

1. **Create virtual/helper switches** for each zone (input_boolean or switch helper)
2. **Configure Generic Thermostat** to control the virtual switch (not physical valve)
3. **Configure blueprint** with **BOTH** parameters (REQUIRED):
   - `zone_climate`: Your Generic Thermostat (for temperature reading)
   - `zone_virtual_switch`: The virtual switch (blueprint monitors this) - REQUIRED
   - `zone_valve`: The physical valve switch (blueprint controls this) - REQUIRED

**Benefits:**
- ✅ Climate entities keep real target temperatures (you can adjust them normally)
- ✅ No conflicts - climate controls virtual switch, blueprint controls physical valve
- ✅ Blueprint coordinates across zones while respecting individual requests
- ✅ Clean separation and easy debugging

**Quick Setup:**
```yaml
# 1. Create virtual switch (Settings → Helpers → Toggle)
input_boolean.bedroom_virtual_valve

# 2. Configure Generic Thermostat
climate:
  - platform: generic_thermostat
    heater: input_boolean.bedroom_virtual_valve  # Virtual!
    target_sensor: sensor.bedroom_temperature

# 3. Configure blueprint with BOTH parameters (REQUIRED)
zone1_climate: climate.bedroom_thermostat
zone1_valve: switch.bedroom_physical_valve       # Physical valve - REQUIRED
zone1_virtual_switch: input_boolean.bedroom_virtual_valve  # REQUIRED
```

**See detailed setup instructions**: [TROUBLESHOOTING.md - Virtual Switch Pattern](TROUBLESHOOTING.md#climate-entity-conflicts-with-blueprint-valve-control)

### Basic Setup

1. **Select MAIN Thermostat**: Your primary HVAC climate entity

2. **Configure Zones**: Add your room thermostats (up to 15, organized in 3 groups)
   - Each zone requires a climate entity
   - Optionally override with custom temperature sensor
   - **REQUIRED**: Specify BOTH physical valve AND virtual switch for each zone

3. **Set Temperature Thresholds**:
   - **Open Threshold**: How many degrees below target triggers valve opening (default: 0.5°C)
   - **Close Threshold**: How many degrees above target triggers valve closing (default: 0.2°C)

4. **Configure "All Satisfied" Behavior**:
   - Use slider to set how MAIN temperature is calculated when all zones are satisfied
   - 0% = lowest, 50% = average, 100% = highest

5. **Optional: Set MAIN Temperature Sensor**:
   - If your MAIN thermostat sensor is in a location (e.g., corridor) that differs from your zones
   - Providing this enables intelligent temperature compensation
   - The blueprint will adjust MAIN target to overcome temperature differences

### Example Configuration

**Scenario**: 
- MAIN thermostat in corridor (corridor temp: 23°C)
- Bedroom: target 22°C, current 20°C (needs heating)
- Bathroom: target 25°C, current 25°C (satisfied)

**Blueprint Behavior (with intelligent compensation)**:
1. Opens bedroom valve (20°C < 22°C - 0.5°C)
2. Closes bathroom valve (25°C > 25°C + 0.2°C)
3. Calculates smart MAIN target:
   - Base target: 22°C (bedroom target)
   - Corridor is 23°C (warmer than target)
   - Temperature deficit: 2°C (22°C - 20°C)
   - Applies 50% compensation: 22°C + (2°C × 0.5) = 23°C
   - **Sets MAIN thermostat to 23°C** (ensures adequate heating despite warm corridor)
4. When bedroom reaches target:
   - Opens ALL valves (prevent all-closed scenario)
   - Sets MAIN to calculated temperature (e.g., average = 23.5°C if slider at 50%)

## ⚙️ Configuration Parameters

### Required
| Parameter | Description |
|-----------|-------------|
| **MAIN Thermostat** | Primary climate entity controlling HVAC |

### Zones (at least 1 required)
| Parameter | Description | Default |
|-----------|-------------|---------|
| **Zone N - Climate Entity** | Climate entity for zone | - |
| **Zone N - Temperature Sensor** | Optional temp sensor override | - |
| **Zone N - Physical Valve Entity** | **REQUIRED** if zone configured - Physical valve switch | - |
| **Zone N - Virtual Switch** | **REQUIRED** if zone configured - Virtual switch controlled by climate entity | - |

### Temperature Management
| Parameter | Description | Default |
|-----------|-------------|---------|
| **Temperature Difference to Open Valve** | Degrees below target to open valve | 0.5°C |
| **Temperature Difference to Close Valve** | Degrees above target to close valve | 0.2°C |
| **All Zones Satisfied - Temperature Mode** | How to calculate MAIN temp when all satisfied (0-100%) | 50% |
| **Minimum MAIN Temperature** | Minimum MAIN thermostat temperature | 18.0°C |
| **Maximum MAIN Temperature** | Maximum MAIN thermostat temperature | 28.0°C |
| **MAIN Temperature Sensor** | Optional sensor for MAIN location (enables intelligent compensation) | - |

### Safety & Fallback
| Parameter | Description | Default |
|-----------|-------------|---------|
| **Fallback Zone(s)** | Zone(s) to keep open when all satisfied/overheated (ensures pump safety) | First zone |
| **Overheated Threshold** | Degrees above target that indicates zone is overheated | 1.0°C |
| **Fallback Temperature** | Temperature used if sensor fails | 20.0°C |
| **Valve Transition Delay** | Time to wait after opening new valve before closing old valve | 5 seconds |

### Advanced
| Parameter | Description | Default |
|-----------|-------------|---------|
| **Enable Cooling Mode Support** | Invert logic for cooling | false |
| **Trigger Time Interval** | How often to run periodic updates (Disabled, 1, 2, 5, 10, 15, or 30 minutes) | Every 1 minute |

**Trigger System:** The blueprint uses a dual-trigger approach for optimal responsiveness:
1. **State-Change Triggers** - Responds immediately (1-2 seconds) when any climate entity changes (state, HVAC mode, temperature, or target temperature). This includes the MAIN thermostat and all 15 zones (64 triggers total).
2. **Periodic Updates** - Runs at the configured interval (default: every 1 minute) to ensure regular recalculation even if no changes detected. Can be disabled to rely solely on state-change triggers.

This design allows you to set longer intervals (e.g., 10-15 minutes) or disable periodic updates entirely while maintaining instant response to user temperature adjustments.

**Valve Transition Delay:** When switching valves (e.g., closing Valve 1 and opening Valve 2), the blueprint first opens the new valve, waits for the specified delay to allow it to fully open, then closes the old valve. This ensures at least one valve is always fully open during transitions, preventing water pump issues. Range: 0-180 seconds (0-3 minutes). Recommended: 5-10 seconds for fast motorized valves, 60-120 seconds for slow valves.

## 🧠 Logic Explanation

### Heating Mode (Default)

1. **Zone needs heating** if: `current_temp < (target_temp - open_threshold)`
   - Valve opens
   - MAIN target = highest requesting zone's target

2. **Zone is satisfied** if: `current_temp >= (target_temp + close_threshold)`
   - Valve closes (if other zones need heating)

3. **All zones satisfied**:
   - ALL valves open (critical constraint!)
   - MAIN target = calculated from slider (low/avg/high)

### Cooling Mode (if enabled)

Inverse logic:
1. **Zone needs cooling** if: `current_temp > (target_temp + open_threshold)`
2. **Zone is satisfied** if: `current_temp <= (target_temp - close_threshold)`
3. MAIN target = **lowest** requesting zone's target (for cooling)

## 📊 Example Scenarios

### Scenario 1: One Zone Needs Heating

| Zone | Target | Current | State | Valve |
|------|--------|---------|-------|-------|
| Bedroom | 20°C | 21°C | Satisfied | Closed |
| Bathroom | 25°C | 22°C | Needs heating | **Open** |
| Living Room | 22°C | 22.5°C | Satisfied | Closed |

**MAIN Target**: 25°C (highest requesting)

### Scenario 2: All Zones Satisfied

| Zone | Target | Current | State | Valve |
|------|--------|---------|-------|-------|
| Bedroom | 20°C | 21°C | Satisfied | **Open** |
| Bathroom | 25°C | 25.5°C | Satisfied | **Open** |
| Living Room | 22°C | 23°C | Satisfied | **Open** |

**MAIN Target**: 22°C (average of 20, 25, 22 if slider at 50%)

### Scenario 3: Multiple Zones Need Heating

| Zone | Target | Current | State | Valve |
|------|--------|---------|-------|-------|
| Bedroom | 20°C | 18°C | Needs heating | **Open** |
| Bathroom | 25°C | 23°C | Needs heating | **Open** |
| Living Room | 22°C | 22.5°C | Satisfied | Closed |

**MAIN Target**: 25°C (highest requesting)

## 🔍 Debugging

The blueprint logs actions to the Home Assistant Logbook:

```
Floor Heating Valve Manager: Recalculating valve states and MAIN temperature.
Mode: Heating. Zones needing action: 2. All satisfied: False. Target MAIN temp: 25.0°C.
```

Check the logbook for detailed information about each calculation cycle.

## 🤝 Contributing

Issues and pull requests are welcome! Please visit the [GitHub repository](https://github.com/Chester929/ha_custom_heat_zone_manager).

## 📄 License

This project is open source and available under the MIT License.

## 🙏 Acknowledgments

- Inspired by the [NSPanel_HA_Blueprint](https://github.com/Blackymas/NSPanel_HA_Blueprint) structure
- Designed for De Dietrich Strateo heat pumps and similar HVAC systems
- Built for the Home Assistant community

## 💡 Support

If you find this blueprint useful, please ⭐ star the repository!

For issues or questions, please open an issue on [GitHub](https://github.com/Chester929/ha_custom_heat_zone_manager/issues).
