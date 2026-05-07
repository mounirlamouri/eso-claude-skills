---
description: Building or modifying the UI of an ESO addon — settings panels with LibAddonMenu-2.0, custom XML controls, scene/fragment integration, and tooltip extensions. Use when designing an addon's settings UI, adding controls, hooking tooltips, or working with .xml files in an addon.
---

# ESO addon UI

ESO's UI is declarative XML + Lua handlers. Most addons need a settings panel and a few custom controls; the standard answer for both is to lean on libraries (especially **LibAddonMenu-2.0**) instead of building from raw XML.

## 1. Settings panel: LibAddonMenu-2.0 (almost always)

Roughly a third of installed addons use LibAddonMenu-2.0 (LAM). It gives you a settings panel that integrates with the in-game menu and a declarative control vocabulary. Use it unless you have a specific reason not to.

**Registration:**

```lua
local LAM = LibAddonMenu2
local panelData = {
  type = "panel",
  name = "My Addon",
  displayName = "My Addon",
  author = "Me",
  version = "1.0",
  slashCommand = "/myaddon",   -- opens this panel
  registerForRefresh = true,   -- re-eval getFunc when SVs change
  registerForDefaults = true,  -- ESC restores from defaults table
}
LAM:RegisterAddonPanel("MyAddon_OptionsPanel", panelData)

local controls = {
  { type = "header", name = "General" },
  { type = "checkbox",
    name = "Enable feature X",
    getFunc = function() return db.featureX end,
    setFunc = function(v) db.featureX = v end,
    default = defaults.featureX },
}
LAM:RegisterOptionControls("MyAddon_OptionsPanel", controls)
```

**Control vocabulary (most-used):**

| Type | Use |
|---|---|
| `header` | Section break; just a name. |
| `description` | Block of explanatory text. |
| `checkbox` | Boolean. |
| `dropdown` | One-of-many; supply `choices` (display) and optionally `choicesValues` (stored). |
| `slider` | Integer with `min`, `max`, `step`. |
| `editbox` | Free text; multi-line via `isMultiline`. |
| `colorpicker` | RGBA. |
| `iconpicker` | Choose from a list of textures. |
| `button` | Action; `func = function() ... end`. |
| `submenu` | Nested group of controls under a foldout. |
| `divider` | Visual separator. |
| `custom` | Drop in any control you've built; you handle layout. |

Each control supports `disabled = function() return ... end` for conditional greying, `warning = "..."` for a hover note, and `tooltip = "..."` for the help bubble. Use `disabled` instead of conditionally hiding — users get confused when controls disappear.

**Refreshing the world after a setting change.** Common pattern in `setFunc`:

```lua
setFunc = function(v)
  db.pinSize = v
  LMP:RefreshPins(MY_PIN_TYPE)   -- or whatever your addon needs to re-render
end
```

When the setting only matters on next reload, use a `warning = "Reload UI to apply"` and don't call refresh.

## 2. When to drop to custom XML

Custom XML is for things LAM can't express:
- A persistent in-world HUD (combat meter, minimap, timer).
- A dialog with non-trivial layout (HarvestMap's import dialog, MasterMerchant's history table).
- A control bound to game scenes (replacing or augmenting a ZOS panel).

Don't write XML for "I want a settings checkbox in a custom place" — wrap a LAM `custom` control instead.

## 3. XML basics

A minimal XML file:

```xml
<GuiXml>
  <Controls>
    <TopLevelControl name="MyAddon_HUD" mouseEnabled="true" movable="true">
      <Dimensions x="200" y="50" />
      <Anchor point="CENTER" relativeTo="GuiRoot" relativePoint="CENTER" />
      <Backdrop edgeColor="000000" centerColor="000000DD" />
      <Controls>
        <Label name="$(parent)Label" font="ZoFontGame" verticalAlignment="CENTER">
          <Anchor point="CENTER" />
        </Label>
      </Controls>
      <OnUpdate>
        MyAddon:UpdateHud(self)
      </OnUpdate>
    </TopLevelControl>
  </Controls>
</GuiXml>
```

Key things to know:
- `$(parent)` is the textual name of the enclosing control. Use it for child names so they're scoped.
- Anchors are everything. A control with no anchor has undefined position. Always anchor at least once.
- `<OnUpdate>` runs every frame. Don't put real work there — see the `async` skill. Throttle with `RegisterForUpdate` instead, even from XML.
- Templates (`<TopLevelControl name="MyAddon_Template" virtual="true">`) let you stamp out repeated controls. Define once, instantiate via `CreateControlFromVirtual(name, parent, template)`.

## 4. Tooltip extension

Adding price info / collected status / extra lines to item tooltips is one of the most common addon features. Two viable patterns:

**a) Override the tooltip method (TamrielTradeCentre style):**

```lua
local function override(toolTipControl, methodName, getLink)
  local base = toolTipControl[methodName]
  toolTipControl[methodName] = function(control, ...)
    base(control, ...)
    if MyAddon.enabled then
      MyAddon:Append(control, getLink(...))
    end
  end
end

override(ItemTooltip, "SetBagItem",      function(b, s) return GetItemLink(b, s) end)
override(ItemTooltip, "SetBuybackItem",  function(i)    return GetBuybackItemLink(i) end)
override(ItemTooltip, "SetQuestReward",  function(q, i) return GetQuestRewardItemLink(q, i) end)
-- etc.
```

You'll need an override per tooltip-feeding method — there are about a dozen. ZO_Tooltip helpers (`ZO_Tooltip_AddDivider`, `tooltip:AddLine(text, font, r, g, b)`) format the appended lines.

**b) `SecurePostHook` for tooltips fed by secure code.** Same shape, safer for any tooltip path that crosses secure code (see `events-and-hooks` for taint rules).

## 5. Scenes and fragments

ESO organises UI as **scenes** (Player Menu, Inventory, Bank, Map) composed of **fragments** (panels that show/hide together). To add your panel to an existing scene:

```lua
local fragment = ZO_FadeSceneFragment:New(MyAddon_Panel)
SCENE_MANAGER:GetScene("inventory"):AddFragment(fragment)
```

To replace existing UI parts, find the fragment via `SCENE_MANAGER:GetScene(name).fragments` and remove/swap. Don't reach into ZOS scene internals from outside `EVENT_PLAYER_ACTIVATED` — scene state isn't fully populated until then.

Common scene names: `"hud"`, `"hudui"`, `"inventory"`, `"bank"`, `"interact"`, `"mainMenu"`, `"map"`. Look up the rest in the live source via `api-lookup`.

## 6. Console vs PC split

Some libraries (notably LibAddonMenu) don't load on the console client. The standard pattern is to declare two manifest dependency sections and split your code into `PC/` and `console/` folders:

```
## PCDependsOn: LibAddonMenu-2.0>=41
## ConsoleDependsOn: ...

PC/MyAddon.lua
PC/Settings.lua
console/MyAddon.lua
```

Then in your manifest list `IsConsoleUI() and "console/..." or "PC/..."` blocks separately. LoreBooks does this cleanly.

If you don't care about console support, ignore this entirely.

## 7. UI traps to avoid

- **Dynamic panel rebuilds bloat memory.** Each call to `RegisterOptionControls` allocates fresh; if you rebuild on every setting change you'll leak. If a control list depends on data, use `disabled` / `warning` to gate, or use a `custom` control whose body re-renders without re-registering.
- **`OnUpdate` runs every frame.** Move work behind `RegisterForUpdate` with a sane interval (40–100 ms is fine for HUD).
- **Don't taint inventory or action UI.** Tooltip overrides for inventory rows and ability bars are notoriously easy to taint. Prefer `SecurePostHook`; never call secure functions from inside a hook (see `events-and-hooks`).
- **Don't anchor to ZOS controls that may not exist.** Some controls are created on first scene activation. Anchor to `GuiRoot` for safety, or anchor lazily inside an `EVENT_PLAYER_ACTIVATED` handler.
- **Localize control text via `GetString(SI_*)`** — see the `core` skill's localization section. Hardcoded English strings in XML look fine until a German user files an issue.
