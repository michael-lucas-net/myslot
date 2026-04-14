# Class Filter Checkbox in Options Panel

## Overview

Replace the existing class filter toggle button (in the main GUI frame) with a persistent checkbox in the Options panel. When checked, the profile dropdown filters profiles by the current character's class. The setting is saved across sessions.

## Requirements

- Checkbox in Options panel (ESC → Myslot), below the existing Minimap Icon checkbox
- Label: `L["Filter profiles by class"]`
- Default: unchecked (false)
- State persisted in `MyslotSettings.classFilterActive` (boolean)
- When checked: dropdown groups matching class profiles first, non-matching greyed out below a separator
- When unchecked: dropdown shows all profiles in normal order
- Existing class-filter logic in `initDropdown` unchanged

## Design

### Approach

Ansatz A — `MyslotSettings.classFilterActive` as single source of truth. No local variable, no sync needed. `gui.lua` reads directly from `MyslotSettings`.

### 1. State Initialization (`options.lua`)

In the existing `ADDON_LOADED` handler where `MyslotSettings` is already initialized:

```lua
MyslotSettings.classFilterActive = MyslotSettings.classFilterActive or false
```

### 2. Checkbox (`options.lua`)

Added below the Minimap Icon checkbox at `TOPLEFT, f, 15, -140`:

```lua
do
    local b = CreateFrame("CheckButton", nil, f, "UICheckButtonTemplate")
    b:SetPoint("TOPLEFT", f, 15, -140)
    b.text = b:CreateFontString(nil, "OVERLAY", "GameFontNormal")
    b.text:SetPoint("LEFT", b, "RIGHT", 0, 1)
    b.text:SetText(L["Filter profiles by class"])
    b:SetChecked(MyslotSettings.classFilterActive)
    b:SetScript("OnClick", function()
        MyslotSettings.classFilterActive = b:GetChecked()
    end)
end
```

### 3. Remove Button (`gui.lua`)

- Delete `local classFilterActive = false` (line 754)
- Delete entire `do` block for the filter button (lines 950–971)

### 4. Update Dropdown (`gui.lua`)

Replace all references to the local `classFilterActive` with `MyslotSettings.classFilterActive`:

```lua
-- Before:
if not classFilterActive then
-- After:
if not MyslotSettings.classFilterActive then
```

### 5. Localization (`locales.lua`)

Add new key:

```lua
L["Filter profiles by class"] = "Filter profiles by class"
```

(plus translations for any other locales present)

## Edge Cases

- **First load / no saved settings:** `MyslotSettings.classFilterActive` defaults to `false` — checkbox unchecked, normal dropdown behavior.
- **Settings loaded before GUI built:** `ADDON_LOADED` initializes `MyslotSettings` before the dropdown is ever rendered — safe to read.
- **No class info in profile:** treated as non-matching (greyed out) — unchanged from existing behavior.

## Files Changed

| File | Change |
|------|--------|
| `options.lua` | Add default init + checkbox |
| `gui.lua` | Remove button block, remove local variable, update condition |
| `locales.lua` | Add `L["Filter profiles by class"]` |

## Not Changed

- Filter logic in `initDropdown` (matching, separator, greyed-out)
- Storage format
- Export/Import logic
