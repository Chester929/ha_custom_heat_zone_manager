# Changelog

All notable changes to the Floor Heating Valve Manager blueprint will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Expanded Zone Support: 15 Zones in 3 Groups**
  - Increased maximum zones from 5 to 15
  - Zones organized into 3 groups of 5 for better UI organization:
    - Group 1: Zones 1-5
    - Group 2: Zones 6-10
    - Group 3: Zones 11-15
  - Each group has a visual header separator in the UI
  - All zones follow the same configuration pattern (climate, temp_sensor, valve, virtual_switch)
- **Virtual Switch Pattern for Generic Thermostat (REQUIRED)**
  - New `zoneN_virtual_switch` parameters for all 15 zones
  - Allows Generic Thermostat climate entities to control virtual/helper switches
  - Blueprint monitors virtual switch state to understand what climate entity wants
  - Blueprint controls physical valve directly based on both virtual switch state AND coordinated logic
  - **MANDATORY REQUIREMENT**: Physical valve and virtual switch MUST BOTH be configured for each zone
    - ✅ BOTH specified → Virtual switch pattern (REQUIRED)
    - ❌ Only one specified → INVALID configuration (blueprint logs error)
    - ℹ️ Zone not needed → Leave climate entity empty
  - **Rationale**: Generic Thermostat MUST have a heater configured. Virtual switch pattern is the only conflict-free solution.
  - **Configuration validation**: Blueprint validates zone configuration and logs errors if only one parameter is specified
  - **Benefits**:
    - Climate entities keep real target temperatures (no overriding with extreme values)
    - No conflicts between climate entity and blueprint
    - Blueprint has final control over physical valves
    - Can coordinate across zones while respecting individual zone requests
    - Clean separation: Climate entity → Virtual switch, Blueprint → Physical valve
  - **Setup**: Create helper switch (input_boolean or switch helper) for each zone
    - Configure Generic Thermostat to control the virtual switch
    - Configure blueprint with BOTH virtual_switch AND physical valve parameters (REQUIRED)
    - Blueprint monitors virtual switch and controls physical valve accordingly
- **Documentation for Virtual Switch Pattern**
  - Updated TROUBLESHOOTING with complete explanation and setup instructions
  - Added configuration examples showing the virtual switch pattern
  - Updated README emphasizing mandatory requirement
  - Documented that both parameters must be specified together
- **Intelligent Temperature Compensation Algorithm**
  - New optional `main_temp_sensor` parameter for MAIN thermostat location (e.g., corridor)
  - Automatically compensates when MAIN sensor location differs from zones needing heating
  - Accounts for temperature deficits between zones and corridor
  - Applies 50% compensation when corridor is warmer than target zones
  - Applies 30% compensation when corridor is significantly warmer than coldest zone
  - Ensures HVAC heats water adequately despite warm corridor sensor reading
  - Example: Corridor 23°C, Bedroom 20°C targeting 22°C → MAIN target compensated to 23°C
- **Fallback Zones Configuration**
  - New optional `fallback_zones` parameter to select which zone(s) stay open when all satisfied/overheated
  - Ensures at least one valve is always open (critical pump safety constraint)
  - Prevents overheating by closing unnecessary valves when all zones satisfied
  - Recommended: Select corridor or least critical room
  - Supports multiple zones for redundancy
  - Defaults to first zone if not configured (backward compatible)
- **Overheated Protection**
  - New `overheated_threshold` parameter (default: 1.0°C)
  - Detects when zones are too hot (current temp > target + threshold)
  - When all zones overheated: Closes all valves EXCEPT fallback zone(s)
  - Automatically lowers MAIN target to minimum temperature
  - Prevents system from continuing to heat when all zones are too hot
  - Maintains pump safety by keeping fallback zone(s) open
- Enhanced logbook entries now include:
  - MAIN sensor temperature
  - All overheated status
  - Number of valves to open
- Added temperature deficit calculation for each zone
- Added overheated status tracking for each zone

### Changed
- **Valve Control Logic** - Now sets climate entity target temperatures to prevent internal thermostat conflicts
- MAIN thermostat temperature calculation now uses intelligent weighted algorithm
- Improved heating effectiveness when MAIN sensor is in a different location than zones
- Valve open/close logic now supports three modes:
  1. **All overheated**: Open only fallback zone(s), MAIN to minimum
  2. **All satisfied**: Open only fallback zone(s) if configured, otherwise all zones (legacy)
  3. **Normal operation**: Open zones needing action
- Updated documentation with new compensation algorithm explanation
- Added new example configuration (Example 7) demonstrating corridor compensation
- Updated ARCHITECTURE.md with detailed algorithm flow and scenarios (6 & 7)
- Updated README.md and QUICKSTART.md to highlight intelligent compensation and fallback features
- Improved condition checks to use `not in [none, '', []]` pattern

### Technical
- Zone data structure now includes `temp_deficit` and `overheated` fields
- New variable `main_current_temp` captures MAIN sensor reading
- New variable `all_zones_overheated` detects overheating condition
- New variable `fallback_zones` determines which zones to keep open
- Backward compatible: works without `main_temp_sensor` and `fallback_zones` parameters
- Cooling mode compensation uses explicit subtraction for clarity

### Documentation
- Added Scenario 4, 5, 6, & 7 to ARCHITECTURE.md explaining new features
- Updated example configurations with corridor sensor and fallback zone examples
- Enhanced README with detailed compensation and fallback examples
- Added fallback zones section to configuration parameters

---

### Added (from previous unreleased)
- **Valve Transition Delay** parameter (default: 5 seconds, range: 0-180 seconds)
  - Configurable delay between opening new valves and closing old valves
  - Prevents brief periods where all valves are in transition
  - Ensures at least one valve is fully open during valve switching
  - Supports slow motorized valves (up to 3 minutes opening/closing time)
  - Recommended: 5-10 seconds for fast valves, 60-120 seconds for slow valves

### Changed (from previous unreleased)
- Valve control logic now uses two-phase approach:
  - Phase 1: Open all valves that need to be opened
  - Delay: Wait for configured transition time (up to 180 seconds)
  - Phase 2: Close all valves that need to be closed
- Updated documentation to explain valve transition behavior
- Increased maximum valve transition delay from 60 to 180 seconds to support slow valves

## [1.0.0] - 2026-01-09

### Added
- Initial release of Floor Heating Valve Manager blueprint
- Support for up to 5 heating/cooling zones
- Dynamic MAIN thermostat target temperature calculation
- Manual overrides for temperature sensors per zone
- Manual overrides for valve entities per zone
- Configurable temperature thresholds (open/close valves)
- "All zones satisfied" mode with configurable calculation:
  - 0% = use lowest zone target temperature
  - 50% = use average of all zone targets
  - 100% = use highest zone target temperature
- Min/max limits for MAIN thermostat temperature
- Cooling mode support with inverse logic
- Periodic update interval (configurable)
- Critical constraint enforcement: at least one valve always open
- Comprehensive logging to Home Assistant logbook
- Complete documentation with examples
- 6 example configurations for different scenarios
- Troubleshooting guide

### Features
- **Heating Mode**: Opens valves when zones need heat, closes when satisfied
- **Cooling Mode**: Inverse logic for air conditioning systems
- **Zone Flexibility**: Configure 1-15 zones, each with optional overrides
- **Temperature Management**: Intelligent calculation based on zone demands
- **Safety**: Guarantees at least one valve is always open (prevents pump issues)
- **Logging**: Detailed logbook entries for debugging

### Documentation
- README.md with comprehensive feature documentation
- TROUBLESHOOTING.md with common issues and solutions
- examples/example_configuration.yaml with 6 example scenarios
- LICENSE (MIT)

### Technical Details
- Compatible with Home Assistant 2024.1.0+
- Uses Home Assistant Blueprint system
- Supports standard climate entities
- Works with any HVAC system that requires zone coordination

## [Unreleased]

### Planned Features
- Support for more than 5 zones
- Advanced scheduling integration
- Presence-based zone control
- Weather-based temperature adjustments
- Energy optimization modes
- Multi-floor support with priority zones

---

## Version History

- **v1.0.0** (2026-01-09) - Initial release with full feature set
