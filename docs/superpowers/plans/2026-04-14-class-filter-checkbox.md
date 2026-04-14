# Class Filter Checkbox Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the class filter toggle button in the main GUI with a persistent checkbox in the Options panel that saves its state across sessions.

**Architecture:** `MyslotSettings.classFilterActive` (boolean) becomes the single source of truth, initialized in `options.lua`. `gui.lua` reads it directly instead of a local variable. The button block in `gui.lua` is removed entirely.

**Tech Stack:** Lua, WoW Classic AddOn API (`CreateFrame`, `UICheckButtonTemplate`, `UIDropDownMenu_Initialize`)

---

## File Map

| File | Change |
|------|--------|
| `options.lua` | Add `MyslotSettings.classFilterActive` default + checkbox UI |
| `gui.lua` | Remove local variable + button block; update condition to read from `MyslotSettings` |
| `locales.lua` | No change needed — metatable fallback returns key string automatically |

---

### Task 1: Persist default value and add checkbox to Options panel

**Files:**
- Modify: `options.lua:38-61`

- [ ] **Step 1: Add default initialization**

In `options.lua`, after line 38 (`MyslotSettings = MyslotSettings or {}`), add the default:

```lua
    MyslotSettings = MyslotSettings or {}
    MyslotSettings.classFilterActive = MyslotSettings.classFilterActive or false
```

- [ ] **Step 2: Add checkbox block after the Minimap checkbox block**

Insert after the closing `end` of the Minimap checkbox block (after line 61):

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

- [ ] **Step 3: Commit**

```bash
git add options.lua
git commit -m "feat: add class filter checkbox to options panel"
```

---

### Task 2: Remove local variable from `gui.lua`

**Files:**
- Modify: `gui.lua:754`

- [ ] **Step 1: Delete the local variable declaration**

Remove line 754 entirely:

```lua
-- DELETE this line:
local classFilterActive = false
```

- [ ] **Step 2: Update the condition in `initDropdown`**

At `gui.lua:769`, change the condition from the local variable to `MyslotSettings`:

```lua
-- Before:
if not classFilterActive then

-- After:
if not MyslotSettings.classFilterActive then
```

- [ ] **Step 3: Commit**

```bash
git add gui.lua
git commit -m "refactor: read classFilterActive from MyslotSettings"
```

---

### Task 3: Remove the filter button block from `gui.lua`

**Files:**
- Modify: `gui.lua:950-971`

- [ ] **Step 1: Delete the entire button `do` block**

Remove the following block from `gui.lua` (lines 950–971):

```lua
        do
            local b = CreateFrame("Button", nil, f, "GameMenuButtonTemplate")
            b:SetWidth(30)
            b:SetHeight(25)
            b:SetPoint("TOPLEFT", t, 615, 0)
            local icon = b:CreateTexture(nil, "ARTWORK")
            icon:SetPoint("CENTER", 0, 0)
            icon:SetSize(20, 20)
            local _, englishClass = UnitClass("player")
            if englishClass then
                icon:SetTexture("Interface\\Icons\\ClassIcon_" .. englishClass)
            end
            b:SetScript("OnClick", function()
                classFilterActive = not classFilterActive
                if classFilterActive then
                    b:LockHighlight()
                else
                    b:UnlockHighlight()
                end
                UIDropDownMenu_Initialize(t, initDropdown)
            end)
        end
```

- [ ] **Step 2: Commit**

```bash
git add gui.lua
git commit -m "feat: remove class filter button, replaced by options checkbox"
```

---

### Task 4: Manual verification in-game

- [ ] **Step 1: Reload WoW or type `/reload` in-game**

- [ ] **Step 2: Open Options panel (ESC → Myslot)**

Verify: Checkbox "Filter profiles by class" appears below "Minimap Icon" checkbox.

- [ ] **Step 3: Check default state**

Verify: Checkbox is unchecked by default (or matches last saved state on a second reload).

- [ ] **Step 4: Enable the filter**

Check the checkbox. Open the profile dropdown in the main MySlot window.
Verify: Profiles matching current class appear first (normal), non-matching profiles are greyed out below a separator.

- [ ] **Step 5: Persist across reload**

Type `/reload`. Open Options panel.
Verify: Checkbox is still checked.

Open the profile dropdown.
Verify: Filter is still active (class profiles first, others greyed out).

- [ ] **Step 6: Disable the filter**

Uncheck the checkbox. Open the profile dropdown.
Verify: All profiles shown in normal order, no separator.

- [ ] **Step 7: Verify button is gone**

Open the main MySlot window.
Verify: No class icon button appears next to the profile dropdown.
