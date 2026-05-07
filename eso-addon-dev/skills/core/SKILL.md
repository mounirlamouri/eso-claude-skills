---
description: Writing or modifying an Elder Scrolls Online (ESO) addon — manifest format (.txt directives), project layout conventions, namespace setup, the EVENT_ADD_ON_LOADED initialization lifecycle, slash commands and keybinds, and localization basics. Loads when working with .txt addon manifests, .lua files inside an ESO AddOns directory, or starting a new ESO addon.
---

# ESO addon development — core

This is the always-relevant skill for ESO addon work. It covers the manifest, the project layout, the boot lifecycle, namespace conventions, slash commands, and localization. For any topic deeper than a one-paragraph mention, it points to one of the focused skills.

## 1. The manifest (`AddonName.txt`)

Every addon has a manifest with the same name as its folder. It's a simple directive-per-line format:

```
## Title: My Addon
## Author: Me
## Version: 1.2.3
## AddOnVersion: 010203
## APIVersion: 101049 101050
## Description: One-line summary that shows in the addon list.
## SavedVariables: MyAddon_AccountSavedVars
## DependsOn: LibAddonMenu-2.0>=41
## OptionalDependsOn: LibDebugLogger
## IsLibrary: false

# Files load in this order:
MyAddon_Init.lua
MyAddon.xml
Localization/$(language).lua
MyAddon.lua
MyAddon_Settings.lua
```

Key directives:

| Directive | Notes |
|---|---|
| `## Title` | Display name. Can include color codes via `|cFFFFFF...|r`. |
| `## Author` | Display author. |
| `## Version` | Human-readable. |
| `## AddOnVersion` | Numeric for compatibility checks (e.g. `010203` = 1.2.3). |
| `## APIVersion` | Space-separated list of supported API versions. **Update on every patch** — see below. |
| `## SavedVariables` | One or more global table names ESO will persist. See the `savedvars` skill. |
| `## DependsOn` / `## OptionalDependsOn` / `## PCDependsOn` / `## ConsoleDependsOn` | Library and addon dependencies. See the `libraries` skill. |
| `## IsLibrary: true` | Marks this addon as a library so the addon manager treats it as such. |
| `## DisableSavedVariablesAutoSaving: 1` | Stops ESO's automatic save; use only when managing saves manually (HarvestMap-style). See `savedvars`. |
| `# `-prefixed lines | Comments. |

**File loading.** Files are loaded in the order they appear. Constants and shared modules first; main code last. The `$(language)` macro expands to the user's language code (`en`, `de`, `fr`, …).

**APIVersion.** Bumped by ZOS every patch (e.g. `101049` → `101050`). Listing both old and new in `## APIVersion` lets your addon load on both for the transition period; once the next patch hits, drop the old version. Players see "out of date" warnings if your single listed version doesn't match. Keep the list short; don't accumulate years of old versions.

To find the current live and PTS API versions on demand, run the slash command **`/eso-addon-dev:check-api-version`**. It looks up both numbers from the official `esoui/esoui` source mirror and recommends the right `## APIVersion:` line for your manifest.

## 2. Project layout

Small addon (single feature, no settings):
```
MyAddon/
├── MyAddon.txt
├── MyAddon.lua
└── Localization/
    ├── en.lua
    └── de.lua
```

Medium addon (settings panel, custom UI, libs):
```
MyAddon/
├── MyAddon.txt
├── MyAddon.lua             -- entry: bootstrap + event wiring
├── MyAddon.xml             -- UI templates
├── Constants.lua           -- enums, defaults, namespace setup
├── Settings.lua            -- LAM panel
├── Localization/
│   ├── en.lua              -- ZO_CreateStringId or SafeAddString calls
│   └── (other languages)
└── (data tables, helper modules)
```

Large addon (HarvestMap, FCOItemSaver, IIfA): organized into `src/` with subdirectories per concern (`src/Events/`, `src/Settings/`, `src/Pins/`, `src/BackupMigration/`, etc.). Adopt this once your single-file layout exceeds a few thousand lines.

**Console support.** If you target the console client, split client-specific code into `PC/` and `console/` directories and gate via `## PCDependsOn` / `## ConsoleDependsOn`. LoreBooks is a clean example.

## 3. Initialization lifecycle

The reliable boot pattern. Every ESO addon should look something like this:

```lua
local ADDON_NAME = "MyAddon"
local MyAddon = {}
_G[ADDON_NAME] = MyAddon            -- expose to other addons / for debugging

local function OnAddOnLoaded(_, name)
  if name ~= ADDON_NAME then return end
  EVENT_MANAGER:UnregisterForEvent(ADDON_NAME, EVENT_ADD_ON_LOADED)

  -- 1. SavedVariables (see savedvars skill)
  MyAddon.db = ZO_SavedVars:NewAccountWide("MyAddon_AccountSavedVars", 1, nil, MyAddon.defaults)

  -- 2. Settings panel (see ui skill)
  MyAddon:InitSettings()

  -- 3. Slash commands
  SLASH_COMMANDS["/myaddon"] = function() MyAddon:Toggle() end

  -- 4. Game events (see events-and-hooks skill)
  EVENT_MANAGER:RegisterForEvent(ADDON_NAME, EVENT_PLAYER_ACTIVATED, OnPlayerActivated)
end

EVENT_MANAGER:RegisterForEvent(ADDON_NAME, EVENT_ADD_ON_LOADED, OnAddOnLoaded)
```

The two non-negotiables:

1. **Filter by addon name** in your `EVENT_ADD_ON_LOADED` handler. The event fires for every addon, including yours.
2. **Unregister immediately** after your name fires. Otherwise you keep getting called and waste cycles.

If your init needs the player in the world (zone known, abilities loaded, inventory readable), defer that part to `EVENT_PLAYER_ACTIVATED` — it fires on every zone load including the first.

## 4. Namespace conventions

The minimum: one global table named after your addon.

```lua
MyAddon = {}
```

A common upgrade for medium-to-large addons is the **public/internal split** — one table for the API other addons can poke at, one for internals:

```lua
local MyAddon = {}
local internal = {}
_G["MyAddon"] = MyAddon
_G["MyAddon_Internal"] = internal
```

LoreBooks uses this. It signals intent (don't reach into `_Internal` from outside) and gives debug tooling a clean handle.

**Library aliasing at the top of each file** keeps your code short and lets you swap implementations:

```lua
local LMP = LibMapPins
local LAM = LibAddonMenu2
```

## 5. Slash commands

```lua
SLASH_COMMANDS["/myaddon"] = function(args) MyAddon:HandleSlash(args) end
SLASH_COMMANDS["/myaddon-debug"] = function() MyAddon.debug = not MyAddon.debug end
```

`args` is the string after the command. Parse it yourself if you need subcommands.

For commands users will discover via the LAM settings panel, set `slashCommand = "/myaddon"` in your panel data — opening the panel becomes part of the same command surface.

## 6. Keybinds

Declare keybinds in a separate `bindings.xml` file:

```xml
<Bindings>
  <Layer name="MyAddonKeybinds">
    <Category name="My Addon">
      <Action name="MYADDON_TOGGLE">
        <Down>MyAddon:Toggle()</Down>
      </Action>
    </Category>
  </Layer>
</Bindings>
```

List `bindings.xml` in your manifest. The action becomes user-bindable in the in-game keybind menu. Translate the action display name via the `SI_BINDING_NAME_*` string ID (e.g. `SI_BINDING_NAME_MYADDON_TOGGLE`).

## 7. Localization

ESO uses string IDs. The manifest loads `Localization/$(language).lua`, which calls `ZO_CreateStringId` once per string:

```lua
ZO_CreateStringId("SI_MYADDON_TITLE", "My Addon")
ZO_CreateStringId("SI_MYADDON_RESET_BUTTON", "Reset")
ZO_CreateStringId("SI_BINDING_NAME_MYADDON_TOGGLE", "Toggle My Addon")
```

In code, retrieve via `GetString(SI_MYADDON_TITLE)`. The string ID is available globally as soon as the language file has loaded.

**Always provide an `en.lua` fallback.** ESO falls back to English when the user's language file is missing a key.

For dynamic strings, use `zo_strformat`:

```lua
ZO_CreateStringId("SI_MYADDON_FOUND", "Found <<1>> books in <<2>>")
local msg = zo_strformat(SI_MYADDON_FOUND, count, zoneName)
```

`<<1>>`, `<<2>>` are positional placeholders; `<<C:1>>` capitalises (locale-aware).

For a full localization deep-dive (gendered articles, plural rules) consult the live source via the `api-lookup` skill.

## 8. Where to go from here

| Question | Skill |
|---|---|
| How do I design my SavedVariables? Migration? Large data? | `savedvars` |
| How do I build a settings panel? Custom UI? Tooltips? | `ui` |
| Hooking ZOS functions safely? Avoiding taint? Event filtering? | `events-and-hooks` |
| Iterating bags? Identifying items reliably? | `inventory` |
| Long-running work without freezing? Chunking? | `async` |
| Which library should I depend on for X? | `libraries` |
| What's the exact signature/value of Y? | `api-lookup` |

Each skill auto-loads when its description matches what you're doing; you can also invoke them explicitly as `/eso-addon-dev:<name>`.
