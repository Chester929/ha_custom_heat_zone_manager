# Changelog

All notable changes to the Floor Heating Valve Manager blueprint will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned Features
- Advanced scheduling integration
- Presence-based zone control
- Weather-based temperature adjustments
- Energy optimization modes
- Multi-floor support with priority zones
- Historical performance analytics

## [2.0.1] - 2026-01-10

### Fixed
- **Frontend Trigger Description Error**
  - Fixed "Error in describing trigger: Cannot read properties of undefined (reading 'includes')" error
  - Removed all state triggers that referenced entity_ids with potential empty values
  - Removed 60 optional zone climate state triggers (zone entities default to `[]`)
  - Removed 4 MAIN thermostat state triggers (to avoid any entity_id issues)
  - Replaced with a single time_pattern trigger that runs every 10 seconds
  - Time-based interval checking ensures the automation only executes at user-configured frequency
  - Resolves GitHub issue about trigger description errors in the automation editor

- **Automation Mode Changed from `single` to `queued`**
  - Fixes issue where automation would stop with "only one execution allowed" error
  - Previous `mode: single` with `max_exceeded: silent` blocked new triggers when automation was running
  - Intermediate fix attempt using `mode: restart` caused time trigger filtering to not work (would trigger every minute even when disabled)
  - New `mode: queued` with `max: 10` allows new triggers to queue up and execute sequentially
  - Works seamlessly with duplicate prevention logic to maintain efficiency
  - Respects time trigger filtering - when set to "disabled", automation correctly stops periodic execution
  - Prevents automation from getting stuck or ignoring triggers
  - Time pattern and entity state triggers now work reliably without blocking or interference

### Added
- **Configurable Trigger Interval**
  - New `trigger_interval` input parameter allows users to control automation execution frequency
  - Range: 10 seconds (minimum, very responsive) to 900 seconds / 15 minutes (maximum, stable systems)
  - Default: 60 seconds (1 minute) provides good balance
  - Uses automation's `last_triggered` attribute to track time since last execution
  - Automation checks every 10 seconds but only executes actions when configured interval has elapsed
  - Works seamlessly with duplicate prevention logic for maximum efficiency
  - Benefits:
    - Users can optimize for their specific system characteristics
    - Fast-changing systems can use shorter intervals (10-30 seconds)
    - Stable systems can use longer intervals (5-15 minutes) to reduce overhead
    - No external helper entities required

- **Duplicate Trigger Prevention**
  - Prevents automation from re-triggering when its own actions cause state changes
  - Added state comparison logic to detect if changes are actually needed before executing
  - New variables: `current_main_target`, `main_temp_needs_change`, `current_valve_states`, `valves_need_change`, `any_changes_needed`
  - **MAIN thermostat temperature**: Only updated if difference >0.05°C from current value
  - **Valve states**: Only changed if not already in desired state (open/closed)
  - Early exit mechanism when no changes needed - logs debug message and stops execution
  - Significantly reduces unnecessary service calls and re-triggers
  - Improves system efficiency and reduces log spam
  - Benefits:
    - Eliminates infinite loops caused by automation's own state changes
    - Reduces Home Assistant system load
    - Cleaner logs with fewer redundant executions
    - Faster response times by avoiding unnecessary processing

## [2.0.0] - 2026-01-10

### Added
- **Configurable Trigger Time Interval**
  - New `trigger_time_interval` input parameter allows users to select periodic trigger frequency
  - Options: Disabled (state-change triggers only), Every 1, 2, 5, 10, 15, or 30 minutes
  - Default: Every 1 minute (maintains backward compatibility)
  - "Disabled" option completely disables periodic updates - automation relies only on state-change triggers
  - Optimized implementation: Single time_pattern trigger with modulo-based filtering reduces system overhead
  - Allows users to balance system responsiveness vs. resource usage
  - Combined with state-change triggers, users can set longer intervals or disable periodic updates entirely without sacrificing responsiveness

- **Comprehensive State-Change Triggers**
  - **Main Climate Entity Triggers (4 new triggers)**:
    - State change (on/off/heat/cool/etc.)
    - HVAC mode attribute change
    - Target temperature attribute change
    - Current temperature attribute change
  - **Zone Climate Entity Triggers (60 new zone triggers, 4 per zone × 15 zones; 64 triggers total including main climate)**:
    - State change for each zone
    - HVAC mode attribute change for each zone
    - Target temperature attribute change for each zone
    - Current temperature attribute change for each zone
  - **Benefits**:
    - Automation responds immediately (within 1-2 seconds) to any climate entity change
    - No need to wait for periodic trigger when user adjusts temperature
    - Valve states update instantly when climate entities change
    - Enables longer periodic intervals (e.g., 10-15 minutes) while maintaining quick response

- **Availability Tracking and Safety Features**
  - Zone data now includes `is_available` status for each zone
  - Tracks unavailable climate entities (state = 'unavailable' or 'unknown')
  - Monitors main climate entity availability
  - **Unavailable zones filtered from calculations** to prevent stale data from affecting valve decisions
  - New variables: `unavailable_zones` and `main_climate_available`

- **Safety Override for Unavailable Entities**
  - Enhanced valve opening logic with multi-level safety override
  - If calculated valve list is empty, fallback zones are forced open
  - If fallback zones is empty, first available zone is used as last resort
  - Ensures at least one valve remains open even when all climate entities are unavailable
  - Critical for preventing pump damage in edge cases
  - Safety check runs on every automation execution with multiple fallback levels

- **Warning Logging for Unavailable Entities**
  - Logs WARNING to Home Assistant system logs when climate entities are unavailable
  - Uses `system_log.write` service for proper warning-level logging
  - Improved message formatting with clear line breaks and conditional safety override indication
  - Warning message includes:
    - Which climate entities are unavailable (main and/or zones)
    - Which valve(s) are being kept open as safety measure
    - Whether safety override was activated
  - Enhanced logbook entries to indicate unavailable zones
  - Helps users quickly identify and troubleshoot connectivity issues

- **Configuration Validation**
  - Added validation to ensure at least one zone is configured
  - Automation will not run if no zones are set up
  - Clear error message directs users to configure at least one zone (zone 1-15)
  - Prevents automation from running without any zones to manage

### Improved
- **Optimized Periodic Trigger System**
  - Replaced 6 separate time_pattern triggers with single optimized trigger
  - Uses modulo-based filtering to check if current minute matches selected interval
  - Significantly reduces unnecessary trigger evaluations and system overhead
  - Simpler, more maintainable code structure

- **Enhanced Availability Checks**
  - Removed unnecessary string 'none' comparison from availability checks
  - Now correctly checks only for 'unavailable' and 'unknown' states
  - More accurate detection of entity availability issues

### Fixed
- **Time Pattern Trigger Validation Error**
  - Fixed invalid `seconds: "/60"` in time_pattern trigger that caused automation save errors
  - Changed to `minutes: "/1"` to comply with Home Assistant validation (seconds must be 0-59)
  - Error message was: "Message malformed: must be a value between 0 and 59 for dictionary value @ data['seconds']"
  - The automation now correctly triggers every 60 seconds (every 1 minute) without validation errors

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
- **Valve Control Logic** - Now uses the virtual switch pattern to prevent internal thermostat conflicts while preserving real climate target temperatures
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
- 10 example configurations for different scenarios (9 active + 1 deprecated reference)
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

---

## Version History

- **v2.0.1** (2026-01-10) - Bug fixes: trigger description error, automation mode, duplicate trigger prevention
- **v2.0.0** (2026-01-10) - Major update with configurable triggers, 15 zones, state-change triggers, availability tracking
- **v1.0.0** (2026-01-09) - Initial release with full feature set
