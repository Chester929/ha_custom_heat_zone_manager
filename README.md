# Floor Heating Valve Manager for Home Assistant

[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Blueprint-blue.svg)](https://www.home-assistant.io/)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](https://github.com/Chester929/ha_custom_heat_zone_manager)

A comprehensive Home Assistant Blueprint for intelligently managing floor heating valves alongside a MAIN thermostat that controls your HVAC water heating system.

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
- ✅ Manages up to 5 heating/cooling zones
- ✅ **Guarantees at least one valve is always open** (critical!)
- ✅ Dynamically calculates MAIN thermostat target temperature
- ✅ Supports both heating and cooling modes
- ✅ Configurable temperature thresholds for valve control

### Zone Configuration
- ✅ Each zone can use a climate entity
- ✅ Optional manual overrides for temperature sensors
- ✅ Optional manual overrides for valve entities
- ✅ Flexible: add 1-5 zones as needed

### Temperature Management
- ✅ Adjustable open/close temperature thresholds
- ✅ Configurable "all zones satisfied" behavior:
  - 0% = Use lowest zone target
  - 50% = Use average of all targets
  - 100% = Use highest zone target
- ✅ Min/max limits for MAIN thermostat
- ✅ Automatic periodic updates

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

### Basic Setup

1. **Select MAIN Thermostat**: Your primary HVAC climate entity

2. **Configure Zones**: Add your room thermostats (up to 5)
   - Each zone requires a climate entity
   - Optionally override with custom temperature sensor
   - Optionally override with custom valve entity

3. **Set Temperature Thresholds**:
   - **Open Threshold**: How many degrees below target triggers valve opening (default: 0.5°C)
   - **Close Threshold**: How many degrees above target triggers valve closing (default: 0.2°C)

4. **Configure "All Satisfied" Behavior**:
   - Use slider to set how MAIN temperature is calculated when all zones are satisfied
   - 0% = lowest, 50% = average, 100% = highest

### Example Configuration

**Scenario**: 
- MAIN thermostat in corridor
- Bedroom: target 20°C, current 21°C (satisfied)
- Bathroom: target 25°C, current 22°C (needs heating)

**Blueprint Behavior**:
1. Opens bathroom valve (22°C < 25°C - 0.5°C)
2. Closes bedroom valve (21°C > 20°C + 0.2°C)
3. Sets MAIN thermostat to 25°C (highest requesting zone)
4. When bathroom reaches 25°C:
   - Opens ALL valves (prevent all-closed scenario)
   - Sets MAIN to calculated temperature (e.g., average = 22.5°C if slider at 50%)

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
| **Zone N - Valve Entity** | Optional valve entity override | - |

### Temperature Management
| Parameter | Description | Default |
|-----------|-------------|---------|
| **Temperature Difference to Open Valve** | Degrees below target to open valve | 0.5°C |
| **Temperature Difference to Close Valve** | Degrees above target to close valve | 0.2°C |
| **All Zones Satisfied - Temperature Mode** | How to calculate MAIN temp when all satisfied (0-100%) | 50% |
| **Minimum MAIN Temperature** | Minimum MAIN thermostat temperature | 18.0°C |
| **Maximum MAIN Temperature** | Maximum MAIN thermostat temperature | 28.0°C |

### Advanced
| Parameter | Description | Default |
|-----------|-------------|---------|
| **Enable Cooling Mode Support** | Invert logic for cooling | false |
| **Update Interval** | How often to recalculate (seconds) | 60 |

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
