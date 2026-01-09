# Changelog

All notable changes to the Floor Heating Valve Manager blueprint will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Valve Transition Delay** parameter (default: 5 seconds, range: 0-180 seconds)
  - Configurable delay between opening new valves and closing old valves
  - Prevents brief periods where all valves are in transition
  - Ensures at least one valve is fully open during valve switching
  - Supports slow motorized valves (up to 3 minutes opening/closing time)
  - Recommended: 5-10 seconds for fast valves, 60-120 seconds for slow valves

### Changed
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
- **Zone Flexibility**: Configure 1-5 zones, each with optional overrides
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
