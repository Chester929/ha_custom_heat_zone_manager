# Duplicate Trigger Prevention - Testing Guide

This document describes how to test and verify the duplicate trigger prevention feature.

## Overview

The duplicate trigger prevention feature prevents the automation from re-triggering when its own actions cause state changes. This is achieved by:

1. **State Comparison**: Comparing current state with desired state before making changes
2. **Conditional Execution**: Only executing service calls when changes are actually needed
3. **Early Exit**: Stopping execution entirely when no changes are needed

## What Gets Prevented

The automation previously would trigger itself in these scenarios:

### Scenario 1: MAIN Thermostat Temperature Change
**Before**: 
1. User adjusts zone temperature → Automation calculates new MAIN target (25.5°C)
2. Automation sets MAIN thermostat to 25.5°C → Triggers automation again
3. Automation recalculates (still 25.5°C) → Sets MAIN thermostat to 25.5°C again
4. Unnecessary re-execution

**After (with prevention)**:
1. User adjusts zone temperature → Automation calculates new MAIN target (25.5°C)
2. Automation sets MAIN thermostat to 25.5°C → Triggers automation again
3. Automation recalculates (still 25.5°C) → Compares: current 25.5°C == target 25.5°C
4. **Skips execution** with debug log: "Skipping execution - no changes needed"

### Scenario 2: Valve State Changes
**Before**:
1. Temperature drops → Automation opens valve → Triggers automation again
2. Automation recalculates → Opens valve again (already open)
3. Unnecessary service call

**After (with prevention)**:
1. Temperature drops → Automation opens valve → Triggers automation again
2. Automation recalculates → Checks if valve already open → **Skips service call**

### Scenario 3: Zone Climate HVAC Mode Change
**Before**:
1. Blueprint sets zone climate to "heat" mode → Triggers automation
2. Automation recalculates → Sets zone climate to "heat" mode again
3. Unnecessary re-execution

**After (with prevention)**:
1. Blueprint sets zone climate to "heat" mode → Triggers automation
2. Automation checks current state is already "heat" → **Skips service call**

## Testing Procedure

### Test 1: MAIN Thermostat Temperature (Manual Test)

1. **Setup**: Enable debug logging in `configuration.yaml`:
   ```yaml
   logger:
     logs:
       blueprints.floor_heating_valve_manager: debug
   ```

2. **Action**: Manually change a zone's target temperature

3. **Expected Behavior**:
   - First execution: Log shows "Applying changes to valve states and MAIN temperature" with new target
   - Second execution (triggered by MAIN temp change): Log shows "Skipping execution - no changes needed"

4. **Verification**: Check Home Assistant logs for the debug message:
   ```
   Skipping execution - no changes needed. 
   Current MAIN target: 25.5°C, 
   Calculated MAIN target: 25.5°C.
   ```

### Test 2: Valve State (Manual Test)

1. **Setup**: Monitor automation traces in Home Assistant

2. **Action**: Change room temperature to trigger valve opening

3. **Expected Behavior**:
   - First execution: Valve opens
   - Second execution (triggered by valve state change): Skips because valve already open

4. **Verification**: Check automation trace to see the condition check that skips the service call

### Test 3: No Changes Needed (Automatic)

This test verifies the early exit mechanism works correctly.

1. **Setup**: System in steady state (all zones satisfied, no temperature changes)

2. **Trigger**: Periodic trigger fires every minute

3. **Expected Behavior**:
   - Automation calculates states
   - Compares with current states
   - Finds no changes needed
   - Logs debug message and stops

4. **Verification**: Look for consecutive executions in the logbook where only the second shows "Skipping execution"

## Verification Checklist

- [ ] Debug logging shows "Skipping execution - no changes needed" messages
- [ ] MAIN thermostat temperature only changes when actually different (>0.05°C)
- [ ] Valves only receive turn_on/turn_off commands when state differs
- [ ] Climate entities only receive HVAC mode changes when mode differs
- [ ] Automation traces show conditional checks preventing redundant service calls
- [ ] System logs show reduced automation execution frequency
- [ ] No infinite loops or rapid re-triggering observed

## Implementation Details

### Key Variables

```yaml
# Current state variables
current_main_target: Current MAIN thermostat temperature setting
current_valve_states: Dictionary with 'open' and 'closed' lists of climate entities

# Decision variables
main_temp_needs_change: Boolean - True if |final_main_target - current_main_target| > 0.05°C
valves_need_change: Boolean - True if any valve needs to open or close
any_changes_needed: Boolean - True if ANY changes are needed (MAIN temp OR valves)
```

### Logic Flow

```
1. Calculate desired states (final_main_target, valves_to_open, valves_to_close)
2. Get current states (current_main_target, current_valve_states)
3. Compare: Do any changes need to be made?
   - Is MAIN temp different by >0.05°C?
   - Are any valves in wrong state?
4. If NO changes needed → Log debug message and STOP
5. If changes needed → Proceed with service calls:
   - Set MAIN temp (only if different)
   - Open valves (only if not already open)
   - Close valves (only if not already closed)
```

### Tolerance Values

- **Temperature**: 0.05°C (smaller than typical sensor precision, prevents floating-point comparison issues)
- **Valve States**: Exact match (binary on/off state)

## Expected Log Patterns

### Normal Operation (Changes Needed)
```
[blueprints.floor_heating_valve_manager] Applying changes to valve states and MAIN temperature. 
Mode: Heating. MAIN sensor temp: 22.3°C. Zones needing action: 2. 
Target MAIN temp: 25.5°C (current: 24.0°C).
```

### Duplicate Trigger Prevention Active
```
[blueprints.floor_heating_valve_manager] Skipping execution - no changes needed. 
Current MAIN target: 25.5°C, Calculated MAIN target: 25.5°C. 
Valve states already match desired configuration.
This prevents duplicate triggers from the automation's own actions.
```

## Performance Impact

Expected improvements:
- **Reduced service calls**: 40-60% fewer climate.set_temperature and homeassistant.turn_on/turn_off calls
- **Faster response**: Less system overhead from redundant executions
- **Cleaner logs**: Fewer duplicate logbook entries
- **Lower CPU usage**: Early exit prevents unnecessary template calculations

## Troubleshooting

### Issue: No "Skipping execution" messages in logs

**Possible Causes**:
1. Debug logging not enabled → Enable `blueprints.floor_heating_valve_manager: debug`
2. State always changing → Check for sensor fluctuations or external automations
3. Temperature tolerance too tight → 0.05°C should catch most duplicates

### Issue: Automation still runs too frequently

**Possible Causes**:
1. Legitimate state changes → Normal behavior, not duplicate triggers
2. Increase periodic trigger interval → Set to "Every 5 minutes" or "Disabled"
3. External automations changing states → Review other automations affecting same entities

### Issue: Changes not being applied

**Possible Causes**:
1. Tolerance too loose → 0.05°C is correct for temperature
2. State comparison logic error → Review automation trace to see variable values
3. Service call failing → Check Home Assistant logs for service call errors
