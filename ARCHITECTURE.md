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
  ├─ Get MAIN sensor (corridor) temperature
  │   ├─ Use override sensor if provided
  │   └─ Else use MAIN climate entity's sensor
  │
  ├─ Are there zones needing action?
  │   ├─ YES (Heating) → Use intelligent compensation algorithm:
  │   │   ├─ Base target = HIGHEST zone target
  │   │   ├─ Calculate max temperature deficit of zones
  │   │   ├─ If corridor temp > base target:
  │   │   │   └─ Add 50% of max deficit as compensation
  │   │   ├─ Else if corridor temp > coldest zone + 1°C:
  │   │   │   └─ Add 30% of temp gap as compensation
  │   │   └─ Else: Use base target (no compensation)
  │   │
  │   ├─ YES (Cooling) → Use intelligent compensation algorithm:
  │   │   ├─ Base target = LOWEST zone target
  │   │   ├─ If corridor temp < base target and zones need cooling:
  │   │   │   └─ Lower target by 50% of max deficit
  │   │   └─ Else: Use base target
  │   │
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

**Why This Matters:**

When the MAIN thermostat sensor (typically in a corridor) reads a different temperature than zones that need heating, the HVAC system may not heat adequately. For example:
- Corridor: 23°C
- Bedroom: 20°C, target 22°C

If we simply set MAIN target to 22°C, the HVAC sees the corridor is already at 23°C (above target) and won't heat the water sufficiently. By adding compensation based on the temperature deficit, we ensure the HVAC heats the water hot enough to warm the bedroom despite the corridor already being warm.

### 5. Critical Constraint Check & Valve Determination

```
Determine which valves to open (at least one MUST be open):
  │
  ├─ Are ALL zones overheated?
  │   ├─ YES → SAFETY MODE!
  │   │   ├─ Close all valves EXCEPT fallback zone(s)
  │   │   ├─ Set MAIN target to minimum temperature
  │   │   └─ Prevent further overheating while maintaining pump safety
  │   └─ NO → Continue evaluation
  │
  ├─ Are ALL zones satisfied (not overheated)?
  │   ├─ YES → Check fallback configuration
  │   │   ├─ Fallback zones configured?
  │   │   │   ├─ YES → Open ONLY fallback zone(s)
  │   │   │   └─ NO → Open ALL valves (legacy behavior)
  │   │   └─ Use "all satisfied" temperature calculation
  │   └─ NO → Continue evaluation
  │
  ├─ Are there zones needing action?
  │   ├─ YES → Open those zone valves
  │   │         └─ Use compensated temperature for MAIN
  │   └─ NO → ERROR/FALLBACK
  │           └─ Open fallback zone(s) only
  │
  └─ Calculate which valves to close
      └─ Any valve NOT in the "open" list gets closed
```

**Fallback Zones:**
- User-configurable zones that stay open when all zones are satisfied/overheated
- Critical for pump safety - ensures at least one valve always open
- Recommended: Corridor or least critical room
- Default: First zone if not configured

**Overheated Detection:**
- Zone is overheated when: `current_temp > (target_temp + overheated_threshold)`
- Default threshold: 1.0°C
- When all zones overheated: Close all except fallback, lower MAIN to minimum

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

### Scenario 4: Corridor Warmer Than Zones (Intelligent Compensation)

**State:**
- Corridor (MAIN sensor): 23°C
- Bedroom: 20°C / target 22°C → Needs heat ✗
- Bathroom: 21°C / target 21°C → Satisfied ✓
- Living: 22°C / target 22°C → Satisfied ✓

**Logic (OLD algorithm):**
```
Zones needing heat: Bedroom
Base MAIN target: 22°C (highest requesting = bedroom target)
Problem: Corridor is 23°C, already above 22°C target
Result: HVAC may not heat water sufficiently because corridor sensor shows temp above target
```

**Logic (NEW intelligent algorithm):**
```
Zones needing heat: Bedroom
Base MAIN target: 22°C (bedroom target)
Corridor temp: 23°C
Coldest zone needing heat: Bedroom at 20°C
Temperature deficit: 22°C - 20°C = 2°C

Calculation:
- Corridor (23°C) > Base target (22°C): TRUE
- Apply 50% compensation: 22°C + (2°C × 0.5) = 23°C
- Final MAIN target: 23°C

Result: HVAC heats water more aggressively because target matches corridor temp,
        ensuring sufficient heat reaches bedroom despite corridor already being warm
Valves:
  ✓ Bedroom: OPEN (needs heat)
  ✗ Bathroom: CLOSED (satisfied)
  ✗ Living: CLOSED (satisfied)
```

### Scenario 5: Large Temperature Difference (Maximum Compensation)

**State:**
- Corridor (MAIN sensor): 24°C
- Bedroom: 18°C / target 22°C → Needs heat ✗ (4°C deficit!)
- Bathroom: 19°C / target 23°C → Needs heat ✗ (4°C deficit)
- Living: 24°C / target 22°C → Satisfied ✓

**Logic (NEW intelligent algorithm):**
```
Zones needing heat: Bedroom, Bathroom
Base MAIN target: 23°C (highest requesting = bathroom target)
Corridor temp: 24°C
Coldest zone: Bedroom at 18°C
Max temperature deficit: 4°C

Calculation:
- Corridor (24°C) > Base target (23°C): TRUE
- Apply 50% compensation: 23°C + (4°C × 0.5) = 25°C
- Final MAIN target: 25°C (after min/max clamping)

Result: Higher MAIN target ensures HVAC heats water hot enough to overcome
        the large temperature gap between warm corridor and cold bedrooms
Valves:
  ✓ Bedroom: OPEN (needs heat)
  ✓ Bathroom: OPEN (needs heat)
  ✗ Living: CLOSED (satisfied)
```

### Scenario 6: All Zones Overheated (New Fallback Logic)

**State:**
- Fallback zones configured: Corridor (climate.corridor)
- Overheated threshold: 1.0°C
- Bedroom: 23°C / target 22°C → Overheated ✗
- Bathroom: 26°C / target 25°C → Overheated ✗
- Living: 23.5°C / target 22°C → Overheated ✗
- Corridor: 24°C / target 23°C → Overheated ✗

**Logic (NEW overheated protection):**
```
All zones overheated: TRUE
Fallback zones: [climate.corridor]

Action:
- Close all valves EXCEPT corridor (fallback)
- Set MAIN target to minimum (18°C)
- Prevent further heating while maintaining pump safety

Result: System cools down safely without closing all valves
Valves:
  ✗ Bedroom: CLOSED (overheated)
  ✗ Bathroom: CLOSED (overheated)
  ✗ Living: CLOSED (overheated)
  ✓ Corridor: OPEN (fallback - ensures pump safety)
MAIN target: 18°C (minimum to stop heating)
```

### Scenario 7: All Satisfied with Fallback Zones

**State:**
- Fallback zones configured: Living Room
- Bedroom: 22.1°C / target 22°C → Satisfied ✓
- Bathroom: 25.1°C / target 25°C → Satisfied ✓
- Living: 22.3°C / target 22°C → Satisfied ✓

**Logic (NEW fallback behavior):**
```
All zones satisfied: TRUE
All zones overheated: FALSE
Fallback zones configured: [climate.living_room]

Action:
- Open ONLY fallback zone (living room)
- Close other zones to prevent overheating
- Use "all satisfied" temperature mode

Result: Better temperature control, prevents unnecessary heating
Valves:
  ✗ Bedroom: CLOSED (satisfied, not fallback)
  ✗ Bathroom: CLOSED (satisfied, not fallback)
  ✓ Living: OPEN (fallback zone)
MAIN target: 23°C (based on slider at 50%)
```

**Note:** If no fallback zones configured, legacy behavior opens ALL valves when satisfied.

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
