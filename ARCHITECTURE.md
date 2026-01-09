# System Architecture and Logic Flow

This document explains how the Floor Heating Valve Manager blueprint works internally.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Floor Heating System                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────┐         ┌─────────────────────┐    │
│  │  MAIN Thermostat   │         │  Blueprint Manager  │    │
│  │  (HVAC Control)    │◄────────┤  (This Blueprint)   │    │
│  └────────────────────┘         └─────────────────────┘    │
│           │                              ▲                   │
│           │ Controls                     │                   │
│           │ Water Temp                   │ Monitors &        │
│           ▼                              │ Manages           │
│  ┌────────────────────┐                  │                   │
│  │   HVAC/Boiler      │                  │                   │
│  │  (Water Heating)   │                  │                   │
│  └────────────────────┘                  │                   │
│           │                              │                   │
│           │ Heated Water                 │                   │
│           ▼                              │                   │
│  ┌──────────────────────────────────────┴────────┐         │
│  │              Water Distribution                │         │
│  │           (Floor Heating Pipes)                │         │
│  └──────────────────────────────────────┬────────┘         │
│                     │                                        │
│        ┌────────────┼────────────┬──────────────┐          │
│        ▼            ▼            ▼              ▼          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌──────────┐    │
│  │  Zone 1  │ │  Zone 2  │ │  Zone 3  │  │  Zone 4  │    │
│  │ Bedroom  │ │ Bathroom │ │  Living  │  │ Kitchen  │    │
│  ├──────────┤ ├──────────┤ ├──────────┤  ├──────────┤    │
│  │ Valve ●  │ │ Valve ●  │ │ Valve ●  │  │ Valve ●  │    │
│  │ Temp 📊  │ │ Temp 📊  │ │ Temp 📊  │  │ Temp 📊  │    │
│  │ Target🎯 │ │ Target🎯 │ │ Target🎯 │  │ Target🎯 │    │
│  └──────────┘ └──────────┘ └──────────┘  └──────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Logic Flow

### 1. Trigger Phase

```
Trigger Events:
  ├─ Zone climate entity state change
  ├─ Zone temperature sensor change
  ├─ MAIN thermostat state change
  └─ Periodic timer (every X seconds)
       │
       ▼
  Blueprint Activates
```

### 2. Data Collection Phase

```
For each configured zone:
  ├─ Get current temperature
  │   ├─ Use override sensor if provided
  │   └─ Else use climate entity's sensor
  ├─ Get target temperature from climate entity
  └─ Get zone HVAC mode
```

### 3. Decision Logic Phase

#### Heating Mode (Default)

```
For each zone:
  │
  ├─ Is current_temp < (target_temp - open_threshold)?
  │   ├─ YES → Zone needs heating
  │   │         └─ Mark valve to OPEN
  │   └─ NO  → Check if satisfied
  │
  └─ Is current_temp >= (target_temp + close_threshold)?
      ├─ YES → Zone is satisfied
      │         └─ Mark valve to CLOSE (if others need heat)
      └─ NO  → Zone is in acceptable range
                └─ Keep current state
```

#### Cooling Mode (if enabled)

```
For each zone:
  │
  ├─ Is current_temp > (target_temp + open_threshold)?
  │   ├─ YES → Zone needs cooling
  │   │         └─ Mark valve to OPEN
  │   └─ NO  → Check if satisfied
  │
  └─ Is current_temp <= (target_temp - close_threshold)?
      ├─ YES → Zone is satisfied
      │         └─ Mark valve to CLOSE (if others need cooling)
      └─ NO  → Zone is in acceptable range
                └─ Keep current state
```

### 4. MAIN Thermostat Calculation

```
Calculate MAIN target temperature:
  │
  ├─ Are there zones needing action?
  │   ├─ YES (Heating) → MAIN target = HIGHEST zone target
  │   ├─ YES (Cooling) → MAIN target = LOWEST zone target
  │   └─ NO → Go to "All Satisfied" logic
  │
  └─ All zones satisfied?
      └─ YES → Calculate based on slider:
          ├─ 0%   → MAIN target = LOWEST zone target
          ├─ 50%  → MAIN target = AVERAGE zone targets
          ├─ 100% → MAIN target = HIGHEST zone target
          └─ Other → Linear interpolation between min and max
  │
  └─ Apply min/max limits
      └─ Final MAIN target = clamp(calculated, min_temp, max_temp)
```

### 5. Critical Constraint Check

```
Before applying valve states:
  │
  └─ Are ALL valves about to close?
      ├─ YES → OVERRIDE! Open ALL valves
      │         └─ Use "all satisfied" temperature
      └─ NO  → Proceed with calculated states
```

### 6. Action Phase

```
Actions performed:
  ├─ Set MAIN thermostat target temperature
  │
  ├─ PHASE 1: Open new valves
  │   └─ For each zone that needs to open:
  │       ├─ Manual valve override?
  │       │   ├─ YES → Turn ON valve entity
  │       │   └─ NO  → Set climate to heat/cool mode
  │
  ├─ Wait for valve transition delay (if > 0)
  │   └─ Allows valves to fully open before closing others
  │
  ├─ PHASE 2: Close old valves
  │   └─ For each zone that needs to close:
  │       ├─ Manual valve override?
  │       │   ├─ YES → Turn OFF valve entity
  │       │   └─ NO  → Set climate to off mode
  │
  └─ Log action to Home Assistant logbook
```

**Valve Transition Logic:**
The two-phase approach ensures at least one valve is always fully open:
1. **Phase 1**: Open all valves that need to be opened
2. **Delay**: Wait for configured time (default: 5 seconds) to allow motorized valves to fully open
3. **Phase 2**: Close all valves that need to be closed

This prevents the scenario where all valves are simultaneously in transition (partially open/closed), which could briefly interrupt water flow and cause pump issues.

## Example Scenarios

### Scenario 1: One Zone Needs Heat

**State:**
- Bedroom: 21°C / target 20°C → Satisfied ✓
- Bathroom: 22°C / target 25°C → Needs heat ✗
- Living: 22°C / target 22°C → Satisfied ✓

**Logic:**
```
Zones needing heat: Bathroom (22 < 25-0.5)
MAIN target: 25°C (highest requesting)
Valves:
  ✗ Bedroom: CLOSED (satisfied)
  ✓ Bathroom: OPEN (needs heat)
  ✗ Living: CLOSED (satisfied)
```

### Scenario 2: All Zones Satisfied

**State:**
- Bedroom: 21°C / target 20°C → Satisfied ✓
- Bathroom: 25.5°C / target 25°C → Satisfied ✓
- Living: 23°C / target 22°C → Satisfied ✓

**Logic:**
```
Zones needing heat: None
All satisfied: TRUE
MAIN target: 22°C (average of 20,25,22 if slider at 50%)
Valves:
  ✓ Bedroom: OPEN (prevent all closed!)
  ✓ Bathroom: OPEN (prevent all closed!)
  ✓ Living: OPEN (prevent all closed!)
```

### Scenario 3: Multiple Zones Need Heat

**State:**
- Bedroom: 18°C / target 20°C → Needs heat ✗
- Bathroom: 23°C / target 25°C → Needs heat ✗
- Living: 23°C / target 22°C → Satisfied ✓

**Logic:**
```
Zones needing heat: Bedroom, Bathroom
MAIN target: 25°C (highest requesting)
Valves:
  ✓ Bedroom: OPEN (needs heat)
  ✓ Bathroom: OPEN (needs heat)
  ✗ Living: CLOSED (satisfied)
```

## Temperature Calculation Examples

### All Satisfied Mode (Slider Effect)

Given zone targets: Bedroom=20°C, Bathroom=25°C, Living=22°C

**Slider at 0% (Lowest):**
```
MAIN target = 20°C
```

**Slider at 25%:**
```
Range: 20°C to 25°C (5°C range)
Position: 25% of range = 1.25°C
MAIN target = 20°C + 1.25°C = 21.25°C
```

**Slider at 50% (Average):**
```
MAIN target = (20 + 25 + 22) / 3 = 22.33°C
```

**Slider at 75%:**
```
Range: 20°C to 25°C (5°C range)
Position: 75% of range = 3.75°C
MAIN target = 20°C + 3.75°C = 23.75°C
```

**Slider at 100% (Highest):**
```
MAIN target = 25°C
```

## Edge Cases Handled

### 1. No Zones Configured
```
Result: Automation doesn't execute (no active zones)
```

### 2. Only One Zone Configured
```
Result: That zone's valve always stays OPEN
        MAIN target = that zone's target
```

### 3. Temperature Sensor Failure
```
Fallback: Use default value (20°C)
Warning: May cause incorrect behavior
```

### 4. Climate Entity Unavailable
```
Result: Skip that zone in calculations
        Other zones continue to work
```

### 5. MAIN Thermostat Unavailable
```
Result: Valve control continues
        Target temperature not updated
```

### 6. Conflicting Temperature Ranges
```
Example: Zone A wants 18°C, Zone B wants 25°C
Result: MAIN set to 25°C (highest)
        Zone A valve CLOSES
        Zone B valve OPENS
```

## Performance Considerations

**Update Frequency:**
- Default: Every 60 seconds
- Minimum: 15 seconds
- Maximum: 300 seconds (5 minutes)

**Calculation Complexity:**
- O(n) where n = number of zones
- Negligible impact up to 10+ zones

**Automation Execution:**
- Mode: Single (one execution at a time)
- Exceeded: Silent (skip if already running)

## Safety Mechanisms

1. **All Valves Closed Prevention:**
   - Explicit check before applying states
   - Opens all valves if none would be open
   - Sets MAIN to "all satisfied" temperature

2. **Temperature Limits:**
   - Min/max bounds on MAIN thermostat
   - Prevents extreme temperatures
   - Configurable per installation

3. **Fallback Values:**
   - Default temperatures if sensors fail
   - Graceful degradation
   - Continues operation with reduced data

4. **Automation Mode:**
   - Single mode prevents race conditions
   - Silent on exceeded prevents log spam
   - Ensures consistent state

## Debugging

**Logbook Entries:**
```
Floor Heating Valve Manager: Recalculating valve states and MAIN temperature.
Mode: Heating. Zones needing action: 2. All satisfied: False. Target MAIN temp: 25.0°C.
```

**Automation Traces:**
- View in Settings → Automations → Your Automation → Traces
- See all variable calculations
- Track decision logic
- Identify issues
