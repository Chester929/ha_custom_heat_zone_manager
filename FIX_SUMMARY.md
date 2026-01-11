# Fix for TypeError: 'float' object cannot be interpreted as an integer

## Problem Statement
The multizone_heater custom component was raising a TypeError when attempting to use the `range()` function with a float value:

```
TypeError: 'float' object cannot be interpreted as an integer
  File "/config/custom_components/multizone_heater/climate.py", line 514, in _async_control_valves
    for i in range(min(needed, len(available_valves))):
```

## Root Cause
The `min_valves_open` configuration parameter was being passed as a float (e.g., `2.0`) from Home Assistant's NumberSelector in the config flow. When this float value was used in calculations:

```python
needed = self._min_valves_open - len(valves_to_turn_on)  # Results in float if min_valves_open is float
```

The result would be a float, which cannot be used with Python's `range()` function.

## Solution
The fix ensures `min_valves_open` is always converted to an integer at two critical points:

### 1. In `async_setup_entry()` (climate.py, line 73)
```python
min_valves_open = int(config.get(CONF_MIN_VALVES_OPEN, DEFAULT_MIN_VALVES_OPEN))
```

### 2. In `__init__()` (climate.py, line 124)
```python
self._min_valves_open = int(min_valves_open)
```

This defensive approach ensures that:
- The value is converted to int when retrieved from config
- The value is converted to int again when stored in the instance (double safety)
- All subsequent uses of `self._min_valves_open` work correctly with operations requiring integers

### Additional Fix: Options Flow Support
The `__init__.py` file was also updated to properly merge options flow updates with entry data:

```python
# Merge options with data, giving priority to options
config = {**entry.data, **entry.options}
hass.data[DOMAIN][entry.entry_id] = config
```

## Testing
A verification script was created and successfully tested the fix:
- ✓ `range()` function works correctly with int conversion
- ✓ List slicing works correctly with int conversion
- ✓ No TypeError raised when min_valves_open is initially a float

## Files Changed
1. `custom_components/multizone_heater/__init__.py` - Added options merge
2. `custom_components/multizone_heater/climate.py` - Added int conversions at entry point and initialization
3. `custom_components/multizone_heater/const.py` - New file
4. `custom_components/multizone_heater/config_flow.py` - New file
5. `custom_components/multizone_heater/manifest.json` - New file
6. `custom_components/multizone_heater/strings.json` - New file

## Code Review & Security
- ✅ Code review completed - all feedback addressed
- ✅ CodeQL security scan completed - no vulnerabilities found

## Impact
This fix resolves the critical issue where the climate entity would crash when trying to control valves, making the integration unusable. The fix is minimal, surgical, and ensures backward compatibility while preventing future occurrences of this error.
