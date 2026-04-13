# Class Filter for Profile Dropdown

## Overview

Add a toggle button to the MySlot GUI that groups/filters saved profiles by the currently logged-in character's class. When active, matching profiles appear first (normal), followed by a separator, then non-matching profiles (greyed out but still clickable).

## Requirements

- Toggle button (off by default) next to the existing "Sort" button
- 50px wide button with class icon (`Interface\Icons\ClassIcon_<CLASS>`)
- When active: profiles matching current class shown first, rest greyed out below a separator
- Class is parsed from the `# Class: <name>` comment line in the profile's export text
- Profiles without class info are treated as non-matching (greyed out)
- "Before Last Import" entry always stays at the top regardless of filter state
- Greyed-out profiles remain clickable (selectable)

## Design

### File Changes

Only `gui.lua` is modified. No changes to Myslot.lua, options.lua, locales.lua, or the storage format.

### 1. Helper Function: GetProfileClass

```lua
local function GetProfileClass(value)
    if not value then return nil end
    return value:match("# " .. CLASS .. ": ([^\r\n]+)")
end
```

Parses the localized class name from the export text header. Uses the WoW global `CLASS` as label, matching the export code in `Myslot.lua:443` which uses `UnitClass("player")`.

### 2. Filter State

```lua
local classFilterActive = false
```

Local boolean in `gui.lua`, not persisted across sessions.

### 3. Filter Button

- Position: next to "Sort" button, at `TOPLEFT, t, 615, 0`
- Size: 50x25
- Icon: `Interface\Icons\ClassIcon_<englishClass>` where `englishClass` comes from `select(2, UnitClass("player"))` (uppercase English name)
- On click: toggles `classFilterActive`, calls `UIDropDownMenu_Initialize(t, initDropdown)` to rebuild dropdown

### 4. Modified initDropdown

When `classFilterActive == true`:

1. Add "Before Last Import" entry (unchanged)
2. First pass: iterate `exports`, add profiles where `GetProfileClass(profile.value) == UnitClass("player")` (normal style, with class-colored check icon)
3. Add separator (disabled info entry)
4. Second pass: iterate `exports`, add remaining profiles with `colorCode = "|cff888888"` (greyed out, still clickable)

When `classFilterActive == false`: unchanged behavior (current code).

### Edge Cases

- **No class info in profile:** treated as non-matching, greyed out when filter active
- **Empty profile (no value):** treated as non-matching
- **Class name matching:** uses localized `UnitClass("player")` for comparison, same as export writes it — language-independent correctness
- **Fewer than 2 profiles:** filter still works, just no visual difference

### Not Changed

- Storage format (no migration needed)
- Export/Import logic
- Existing profiles work retroactively (class parsed from existing comment header)
- Sort functionality (sort order preserved within each group)
