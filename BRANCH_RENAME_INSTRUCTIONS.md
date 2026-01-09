# Branch Rename Instructions

## Issue
The `heat_zone_manager.yaml` file is currently accessible on the `master` branch but needs to be accessible on the `main` branch. The blueprint's `source_url` and raw GitHub URL reference the `main` branch.

## Current State
- ✅ `heat_zone_manager.yaml` exists in the `master` branch (commit d5960967c1a53cf093aedfc844d3618c0536404f)
- ✅ File has correct content and structure
- ✅ File has correct `source_url` pointing to `main` branch
- ❌ `main` branch does not exist on GitHub remote
- ❌ Raw URL returns 404: `https://raw.githubusercontent.com/Chester929/ha_custom_heat_zone_manager/main/heat_zone_manager.yaml`

## Solution: Rename master to main

To resolve this issue, the `master` branch needs to be renamed to `main` on GitHub. This requires repository admin access.

### Steps to Rename Branch on GitHub:

1. **Go to Repository Settings**
   - Navigate to: https://github.com/Chester929/ha_custom_heat_zone_manager/settings

2. **Access Branches Section**
   - Click on "Branches" in the left sidebar

3. **Rename Default Branch**
   - Find the "Default branch" section
   - Click the pencil/edit icon next to `master`
   - Type `main` as the new branch name
   - Click "Rename branch"
   - Confirm the rename operation

4. **Verify the Change**
   - Check that the default branch is now `main`
   - Verify the file is accessible at: `https://raw.githubusercontent.com/Chester929/ha_custom_heat_zone_manager/main/heat_zone_manager.yaml`
   - Verify the blueprint import URL works: `https://github.com/Chester929/ha_custom_heat_zone_manager/blob/main/heat_zone_manager.yaml`

### Alternative: Using GitHub CLI (requires authentication)
```bash
# If you have GitHub CLI installed and authenticated:
gh api repos/Chester929/ha_custom_heat_zone_manager/branches/master/rename -f new_name=main
```

### Alternative: Using Git Commands (requires push permissions)
```bash
# Rename local branch
git branch -m master main

# Push main branch to remote
git push -u origin main

# Delete old master branch from remote (optional, after updating default branch on GitHub)
git push origin --delete master
```

## After Rename
Once the branch is renamed:
- ✅ The `source_url` in `heat_zone_manager.yaml` will correctly point to the existing file
- ✅ Home Assistant blueprint import will work correctly
- ✅ Raw GitHub URL will return the file content instead of 404
- ✅ All existing tags and commit history will be preserved

## Notes
- The file content does NOT need to be changed - it already has the correct `source_url`
- No code modifications are required
- This is purely a repository configuration change
