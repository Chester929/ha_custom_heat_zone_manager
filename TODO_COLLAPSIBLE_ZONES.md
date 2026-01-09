# TODO: Complete Collapsible Groups for Zones 6-15

## Status

✅ **Completed:**
- Zones 1-5 are properly nested in `zones_group_1` collapsible group
- All parameters have correct indentation (8 spaces for param names, 10 for properties)
- Group has icon, description, and `collapsed: true`

❌ **Remaining Work:**
- Zones 6-10 need to be converted from `zones_group_2_header` (text field) to proper `zones_group_2` collapsible group
- Zones 11-15 need to be converted from `zones_group_3_header` (text field) to proper `zones_group_3` collapsible group

## Current Implementation (Zones 6-10 and 11-15)

Currently using simple text headers:
```yaml
zones_group_2_header:
  name: "━━━━━━━━━━━━━ ZONES GROUP 2 (Zones 6-10) ━━━━━━━━━━━━━"
  description: Configure zones 6 through 10 below
  default: ""
  selector:
    text:

# Then zones 6-10 defined at root level (4 spaces)
zone6_climate:
  name: Zone 6 - Climate Entity
  ...
```

## Target Implementation

Should match zones_group_1 pattern:
```yaml
zones_group_2:
  name: "🏠 ZONES GROUP 2 (Zones 6-10)"
  icon: mdi:home-group
  description: Configure heating zones 6 through 10. Each zone requires climate entity, physical valve, and virtual switch.
  collapsed: true
  input:
    # Zone 6 Configuration
    zone6_climate:  # 8 spaces
      name: Zone 6 - Climate Entity  # 10 spaces
      description: ...  # 10 spaces
      default: []  # 10 spaces
      selector:  # 10 spaces
        entity:  # 12 spaces
          domain: climate  # 14 spaces
          multiple: false  # 14 spaces
    
    zone6_temp_sensor:
      # Same pattern...
    
    zone6_valve:
      # Same pattern...
    
    zone6_virtual_switch:
      # Same pattern...
    
    # Zones 7-10 follow same pattern
```

## Implementation Steps

For each group (zones_group_2 and zones_group_3):

1. Replace the `zones_group_N_header` text field with proper collapsible group structure
2. Add 4 spaces to all zone parameter definitions (move from 4 to 8 spaces)
3. Add 4 spaces to all parameter properties (move from 6 to 10 spaces)
4. Add 4 spaces to all selector properties (move from 8 to 12 spaces, etc.)
5. Ensure the closing of the group is at the correct indentation level

## Estimated Effort

- Zones 6-10: ~200 lines to adjust (5 zones × 4 parameters × ~10 lines each)
- Zones 11-15: ~200 lines to adjust (5 zones × 4 parameters × ~10 lines each)
- Total: ~400 lines need indentation adjustment
- Time estimate: 30-45 minutes of careful editing

## Testing Required

After making changes:
1. Validate YAML syntax
2. Test blueprint loads in Home Assistant
3. Verify all 15 zones appear in UI
4. Confirm groups 2 and 3 are collapsible
5. Test that zone parameter references in variables section still work

## Alternative Approach

Could use a Python script to automate the indentation changes:
1. Identify zones 6-15 parameter blocks
2. Add 4 spaces to each line
3. Wrap in proper group structure
4. Validate result

## References

- NSPanel blueprint pattern: Uses this exact collapsible group structure
- BLUEPRINT_RESTRUCTURING_GUIDE.md: Contains detailed syntax examples
- zones_group_1 (lines 81-316): Working example to copy from
