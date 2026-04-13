# Class Filter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a toggle button to the MySlot profile dropdown that groups profiles by the logged-in character's class — matching profiles shown first, others greyed out but clickable.

**Architecture:** Single-file change to `gui.lua`. A helper function parses the class from the profile export text's `# Class: <name>` comment header. A 50px icon button toggles the filter. When active, `initDropdown` does two passes: matching profiles first (normal), then a separator, then the rest (greyed out).

**Tech Stack:** WoW Lua API, UIDropDownMenu framework

---

## File Structure

- **Modify:** `gui.lua` — all changes go here
  - Add `GetProfileClass` helper function (top of file, after constants)
  - Add `classFilterActive` local variable (inside ADDON_LOADED, alongside dropdown state)
  - Modify `initDropdown` function to support grouped display
  - Add class filter icon button after the Sort button

No new files created.

---

### Task 1: Add GetProfileClass helper function

**Files:**
- Modify: `gui.lua:1-7` (after constants, before frame creation)

- [ ] **Step 1: Add the helper function after the constants block**

Insert after line 7 (after `local IMPORT_BACKUP_COUNT = 1` and its blank line):

```lua
local function GetProfileClass(value)
    if not value then return nil end
    return value:match("# " .. CLASS .. ": ([^\r\n]+)")
end
```

This parses the localized class name from the export text header. `CLASS` is a WoW global string (e.g. "Class" in English, "Klasse" in German). The export code in `Myslot.lua:443` writes `# <CLASS>: <UnitClass("player")>`, so this pattern matches it exactly across all locales.

- [ ] **Step 2: Commit**

```bash
git add gui.lua
git commit -m "feat: add GetProfileClass helper to parse class from profile text"
```

---

### Task 2: Add classFilterActive state and filter button

**Files:**
- Modify: `gui.lua:866-898` (after the Sort button block)

- [ ] **Step 1: Add classFilterActive variable and the icon button**

Insert a new `do ... end` block immediately after the Sort button's closing `end` on line 898 (before the `end` that closes the dropdown section on line 900):

```lua
        do
            local classFilterActive = false
            local _, englishClass = UnitClass("player")
            local b = CreateFrame("Button", nil, f, "GameMenuButtonTemplate")
            b:SetWidth(50)
            b:SetHeight(25)
            b:SetPoint("TOPLEFT", t, 615, 0)
            do
                local icon = b:CreateTexture(nil, 'ARTWORK')
                icon:SetTexture("Interface\\Icons\\ClassIcon_" .. englishClass)
                icon:SetPoint('CENTER', 0, 0)
                icon:SetSize(20, 20)
            end
            b:SetScript("OnClick", function()
                classFilterActive = not classFilterActive
                UIDropDownMenu_Initialize(t, initDropdown)
            end)
        end
```

**Problem:** `classFilterActive` is scoped inside this `do` block, but `initDropdown` (defined earlier at line 746) needs to read it. We need to hoist the variable. Move it to a shared scope.

- [ ] **Step 2: Hoist classFilterActive to shared scope**

Instead of declaring `classFilterActive` inside the `do` block, declare it right before the `initDropdown` function definition. Insert before line 746 (`local initDropdown = function()`):

```lua
        local classFilterActive = false
```

And remove the `local classFilterActive = false` from the button's `do` block (from Step 1). The button block becomes:

```lua
        do
            local _, englishClass = UnitClass("player")
            local b = CreateFrame("Button", nil, f, "GameMenuButtonTemplate")
            b:SetWidth(50)
            b:SetHeight(25)
            b:SetPoint("TOPLEFT", t, 615, 0)
            do
                local icon = b:CreateTexture(nil, 'ARTWORK')
                icon:SetTexture("Interface\\Icons\\ClassIcon_" .. englishClass)
                icon:SetPoint('CENTER', 0, 0)
                icon:SetSize(20, 20)
            end
            b:SetScript("OnClick", function()
                classFilterActive = not classFilterActive
                UIDropDownMenu_Initialize(t, initDropdown)
            end)
        end
```

- [ ] **Step 3: Commit**

```bash
git add gui.lua
git commit -m "feat: add class filter toggle button with class icon"
```

---

### Task 3: Modify initDropdown to support class grouping

**Files:**
- Modify: `gui.lua:746-769` (the `initDropdown` function)

- [ ] **Step 1: Replace the initDropdown function body**

Replace the entire `initDropdown` function (lines 746-769) with:

```lua
        local initDropdown = function()
            local info = UIDropDownMenu_CreateInfo()
            info.text = L["Before Last Import"]
            info.customCheckIconTexture = "Interface\\Icons\\inv_scroll_04"
            info.func = function()
                local b = backups[1] -- only 1 backup now, will support more later
                if b then
                    exportEditbox:SetText(b)
                    infolabel:SetText("")
                    UIDropDownMenu_SetText(t, "")
                end
            end
            UIDropDownMenu_AddButton(info)

            if not classFilterActive then
                -- Normal mode: show all profiles in order
                for i, txt in pairs(exports) do
                    local itemInfo = UIDropDownMenu_CreateInfo()
                    itemInfo.text = txt.name
                    itemInfo.value = i
                    itemInfo.func = onclick
                    itemInfo.customCheckIconTexture = "Interface\\Icons\\inv_scroll_03"
                    UIDropDownMenu_AddButton(itemInfo)
                end
            else
                -- Filtered mode: matching class first, then separator, then rest greyed out
                local playerClass = UnitClass("player")

                -- First pass: matching profiles
                for i, txt in pairs(exports) do
                    local profileClass = GetProfileClass(txt.value)
                    if profileClass == playerClass then
                        local itemInfo = UIDropDownMenu_CreateInfo()
                        itemInfo.text = txt.name
                        itemInfo.value = i
                        itemInfo.func = onclick
                        itemInfo.customCheckIconTexture = "Interface\\Icons\\inv_scroll_03"
                        UIDropDownMenu_AddButton(itemInfo)
                    end
                end

                -- Separator
                local sepInfo = UIDropDownMenu_CreateInfo()
                sepInfo.isTitle = true
                sepInfo.notCheckable = true
                sepInfo.text = " "
                UIDropDownMenu_AddButton(sepInfo)

                -- Second pass: non-matching profiles (greyed out)
                for i, txt in pairs(exports) do
                    local profileClass = GetProfileClass(txt.value)
                    if profileClass ~= playerClass then
                        local itemInfo = UIDropDownMenu_CreateInfo()
                        itemInfo.text = txt.name
                        itemInfo.value = i
                        itemInfo.func = onclick
                        itemInfo.colorCode = "|cff888888"
                        itemInfo.customCheckIconTexture = "Interface\\Icons\\inv_scroll_03"
                        UIDropDownMenu_AddButton(itemInfo)
                    end
                end
            end
        end
```

Key details:
- `UnitClass("player")` returns the localized class name as first value (e.g. "Priester" in German)
- `GetProfileClass` also returns the localized name because the export writes it that way
- `isTitle = true` with `notCheckable = true` creates a non-clickable separator line
- `colorCode = "|cff888888"` makes the text grey but the button remains clickable
- The `value = i` mapping is preserved so selecting a greyed-out profile still works correctly with `onclick`

- [ ] **Step 2: Commit**

```bash
git add gui.lua
git commit -m "feat: implement class-grouped dropdown when filter is active"
```

---

### Task 4: Manual testing in WoW

- [ ] **Step 1: Verify normal mode (filter off)**

1. Log into WoW with any character
2. Open MySlot (`/myslot`)
3. Click the profile dropdown — all profiles should appear in normal order
4. The class icon button should be visible next to "Sort"
5. Select a profile — it should load into the text box as before

- [ ] **Step 2: Verify filtered mode (filter on)**

1. Click the class icon button to activate the filter
2. Open the dropdown:
   - Profiles matching your class should appear first (normal color)
   - A separator line should appear
   - Non-matching and unknown-class profiles should appear greyed out below
3. Click a greyed-out profile — it should still be selectable and load into the text box

- [ ] **Step 3: Verify toggle off**

1. Click the class icon button again to deactivate
2. Open the dropdown — should return to normal unfiltered view

- [ ] **Step 4: Verify Sort + Filter interaction**

1. Click "Sort" to sort profiles
2. Activate the class filter
3. Open dropdown — profiles within each group should maintain their sorted order

- [ ] **Step 5: Commit final state if any adjustments were needed**

```bash
git add gui.lua
git commit -m "fix: adjustments from manual testing"
```
