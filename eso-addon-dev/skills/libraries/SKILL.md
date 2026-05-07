---
description: Choosing the right ESO addon library for a task — picking among LibAddonMenu, LibCustomMenu, LibMapPins, LibCombat, LibLazyCrafting, LibAsync, LibSavedVars, LibDebugLogger, LibChatMessage, LibMediaProvider, LibFilters, LibPrice, LibHistoire, LibGPS — and writing the right ## DependsOn / ## OptionalDependsOn / ## PCDependsOn manifest lines. Use when deciding what to depend on.
---

# ESO addon libraries — what to use when

The ESO addon ecosystem has dozens of `Lib*` packages. Pulling in the right one saves weeks of fighting the API and inherits battle-tested fixes for game quirks. Pulling in the wrong one (or the right one at the wrong version) is one of the most common reasons addons break on patch days.

This skill is a decision guide. For each library: **what problem it solves, when to reach for it, when not to.**

## 1. Decision shorthand

| You want to… | Use |
|---|---|
| A settings panel | **LibAddonMenu-2.0** (LAM) |
| A right-click context menu on inventory rows / chat / etc. | **LibCustomMenu** |
| Persistent debug output you can read after the fact | **LibDebugLogger** + DebugLogViewer addon |
| Print to chat with a colored prefix and addon name | **LibChatMessage** |
| Add custom map pins | **LibMapPins-1.0** |
| Add custom compass pins | `COMPASS_PINS` (built-in) or **CustomCompassPins** addon |
| Aggregate combat events without doing raw filtering | **LibCombat** |
| Queue crafting actions (writs, refining, deconning) | **LibLazyCrafting** |
| Run long jobs without freezing the UI | **LibAsync** |
| Track guild history reliably across sessions | **LibHistoire** |
| Convert between map and global coordinates | **LibGPS** (3.x) |
| Get and set "media" fonts/textures users have configured | **LibMediaProvider** |
| Item market prices for tooltips | **LibPrice** (queries multiple price addons via a uniform API) |
| Filter default ESO inventory/bank UI panels | **LibFilters-3.0** |
| Account-wide-with-per-character-fallback SavedVars boilerplate | **LibSavedVars** |
| Sort tables with stable predicates | **LibSort** |
| Display a notification ("you got X!") | **LibNotification** |

## 2. Per-library notes

**LibAddonMenu-2.0** — Almost universal for settings UI. See the `ui` skill for control vocabulary. Does **not** load on the console client; gate with `## PCDependsOn` if you support both. Version constraint: pick something modern (`>=41` is current as of 2026).

**LibCustomMenu** — Adds custom items to the right-click context menu on inventory slots, chat names, etc. Easier than building your own and it handles taint correctly. Don't try to do this with raw `ZO_PreHook` — use this lib.

**LibDebugLogger** — Persistent log to SavedVariables; pair with `DebugLogViewer` (a separate addon a user installs to read your logs). Use throughout development; in release builds, gate verbose output behind a setting. Optional dep — graceful fallback if missing:
```lua
local logger = LibDebugLogger and LibDebugLogger:Create("MyAddon") or { Info = function() end, ... }
```

**LibChatMessage** — Provides `RegisterChannel` for namespaced chat output. Lets users disable your chat noise without disabling your addon. Worth pulling in once you produce more than one chat line per session.

**LibMapPins-1.0** — The standard for adding pins to the world map. Two-step pattern: register a *pin type* with a layout once, then `CreatePin` for individual pins. Handles filter UI on the map. LoreBooks and SkyShards both use it.

**COMPASS_PINS** (global) and **CustomCompassPins** (addon) — Compass pin support. The global is built into recent game versions for some pin classes; the addon is needed for arbitrary custom pins. Most map-pin addons also register a compass pin layer.

**LibCombat** — The right way to consume combat events. Handles event filtering, source/target classification, ability translation. CombatMetrics depends on it; you should too if you're processing combat at any scale. Don't register raw `EVENT_COMBAT_EVENT` if you can use this.

**LibLazyCrafting** — Queue interface for all crafting stations. Submit "craft me one of X with these materials" and get a callback when done. Handles the ugly parts (station interaction states, mat counting, autocraft toggles). DolgubonsLazyWritCreator is built on it.

**LibAsync** — Task scheduling for long jobs. See the `async` skill — you can roll your own with `RegisterForUpdate`, but LibAsync gives cancellation, error handling, composition. Worth it once you have more than one concurrent async pipeline.

**LibHistoire** — Guild history retrieval that works around ZOS's flaky guild-history pagination. If you need to walk guild trade or roster history, use this — don't try direct `RequestGuild*` calls.

**LibGPS** — Coordinate conversion (map ↔ world ↔ global). Necessary if your addon needs to reason about positions across maps. Map pin addons depend on it.

**LibMediaProvider** — A media registry users populate with fonts, sounds, and textures. Lets your addon's font dropdowns show every font installed across the user's other media-aware addons. Pull in if you have a font/texture/sound setting.

**LibPrice** — A uniform price query API across MasterMerchant, ArkadiusTradeTools, TamrielTradeCentre. Use this in tooltip code if you want to show market prices without forcing the user to use a specific price-tracking addon.

**LibFilters-3.0** — Filter the default inventory, bank, guild bank, and crafting UIs. Adds tabs and predicates without you reimplementing the panels. Use this for "hide quest items in my bank view" and similar.

**LibSavedVars** — A wrapper over `ZO_SavedVars` that handles "account-wide unless the user opts to override per-character" cleanly. Optional. The pattern is simple enough to write by hand (DolgubonsLazyWritCreator does), but pull this in if you want it solved once.

**LibSort** — Stable, predicate-driven sort. ESO's `table.sort` is unstable. If sort order matters and you have ties, use this.

**LibNotification** — Push a notification badge into the in-game notifications UI. Niche but the right answer when you need it.

## 3. Manifest dependency directives

Three flavors. Pick by what happens if the dep is missing:

| Directive | Behavior |
|---|---|
| `## DependsOn:` | Required. Addon won't load if missing. |
| `## OptionalDependsOn:` | Loaded *before* your addon if present, but not required. Your code must check. |
| `## PCDependsOn:` | PC-only required dep. Use for LibAddonMenu and other libs that don't ship on console. |
| `## ConsoleDependsOn:` | Console-only required dep. |

Version constraints with `>=`:

```
## DependsOn: LibAddonMenu-2.0>=41 LibMapPins-1.0>=10047
## OptionalDependsOn: LibDebugLogger
## PCDependsOn: LibCustomMenu>=730
```

**Don't pin exact versions** (`==`). Libraries fix bugs. Pin only the minimum that contains the API you call.

## 4. Optional dep with graceful fallback

Most addons can degrade if a debug or convenience lib is missing. Pattern:

```lua
local logger
if LibDebugLogger then
  logger = LibDebugLogger:Create(ADDON_NAME)
else
  logger = { Info = function() end, Warn = function() end, Error = function() end }
end
```

Now you can `logger:Info(...)` everywhere without guarding each call site.

## 5. Bundling vs depending

You will see addons that ship a `libs/` folder containing copies of LibAddonMenu and friends. **Don't do this.** Pros: zero install friction. Cons: if your bundled version is older than another addon's, the other addon may load yours and break. Always declare via `## DependsOn` and let the user (or Minion / a similar addon manager) install the lib.

The exception: a lib that's truly internal — you forked it, modified it, no one else uses it. Then bundle it under a unique name so it can't collide.

## 6. Circular dependency trap

If addon A `## DependsOn` addon B and B `## OptionalDependsOn` A, you have a cycle that may or may not load deterministically. The convention to avoid this: a *better-with* relationship goes in a code comment, not in `OptionalDependsOn`. FCOItemSaver's manifest documents this explicitly. Reach for `OptionalDependsOn` only when you actually need the other addon's table to be populated before yours loads.
