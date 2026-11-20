---
description: Making an ESO addon run on console (PlayStation 5 / Xbox Series X|S) — the .addon manifest, PC/Console special folders, PCDependsOn/ConsoleDependsOn, detecting the platform at runtime, gamepad-only UI and console settings menus, the shared add-on memory and per-frame CPU budget, the SavedVariables disk quota, testing with ForceConsoleFlow, and publishing through Bethesda.net. Use when porting a PC addon to console, writing a cross-platform addon, or debugging an addon that works on PC but fails on console.
---

# ESO addons on console

Since Update 46 (June 2025), players on PlayStation 5 and Xbox Series X|S can install add-ons from an in-game browser. Last-gen consoles (PS4, Xbox One) are excluded. Console add-ons use the same Lua/XML API as PC, but they run in a stricter environment, are distributed through a different channel, and only get the gamepad UI.

This skill covers what changes. For exact function names, constants and signatures, use `api-lookup`. The `zo_platformutils.lua` and `addoncompatibilityaliases/` folders in the source mirror are the relevant starting points.

## 1. Should this addon go to console at all?

Check this first. Some addon designs don't survive the trip:

| Design | On console |
|---|---|
| **External-tool sync via SavedVariables** (TamrielTradeCentre-style uploader reading the `.lua` file) | **Not possible.** Console players have no file access; on Xbox the SavedVariables live in cloud storage. |
| **Custom keybindings** (`bindings.xml` actions the user binds) | **Not available.** The Lua functions for binding keys are private on console. See §5. |
| **Keyboard-UI windows** (custom XML built on keyboard templates) | **Not loaded.** You need a gamepad UI. See §5. |
| **Scanning whole data sets** (every item ID, every ability, every collectible) | **Dangerous.** The game loads the definitions on first touch and never unloads them, and that memory counts against the add-on pool. See §6. |
| **Huge static data tables** (HarvestMap-scale node databases) | Only with lazy, per-zone loading. See §6. |
| **Slash-command-only interface** | Typing on a controller is painful. Every action needs a settings-menu or keybind-strip entry. |

If the addon is mostly event-driven logic plus map pins, tooltips or a settings menu, it ports well. LoreBooks, Dolgubon's Lazy Writ Crafter and most of the `Lib*` packages in the libraries skill already ship console builds.

## 2. The manifest: `.addon`, not `.txt`

- **Use `MyAddon.addon`** for any addon that should run on console. On console and in ForceConsoleFlow, `.txt` manifests are **ignored entirely**. On PC, `.addon` is the preferred extension and `.txt` is deprecated.
- **Keep PC-only addons on `.txt`.** Otherwise they load in ForceConsoleFlow, waste memory and throw errors.
- **Never ship both files.** On PC the one with the higher `AddOnVersion` wins, and if the versions are equal, which one loads is undefined.
- **Split dependencies by platform:**
  ```
  ## DependsOn: LibLazyCrafting>=4035
  ## PCDependsOn: LibAddonMenu-2.0>=41
  ## ConsoleDependsOn: LibHarvensAddonSettings
  ## OptionalDependsOn: LibDebugLogger
  ```
  This is from `DolgubonsLazyWritCreator.addon`. Plain `DependsOn` applies everywhere.
- **No embedded libraries on console.** PC scans up to three folder levels deep for nested manifests. Console only loads the top-level `.addon` in each addon folder, so a library bundled inside your folder never loads there. Declare it as a dependency instead; the console browser installs it as its own add-on.
- **Filenames are case-sensitive on PlayStation.** `Console/Settings.lua` in the manifest does not match `console/settings.lua` on disk. Keep manifest paths and texture paths exactly matching the files.
- **500 files per upload**, counting everything in the package, not only what the manifest loads. Strip dev files, tests and `.git` from the upload.

The other manifest rules (UTF-8 without BOM, 301-byte line limit, `$(language)` expansion) are the same as on PC. See `core`.

## 3. Splitting PC and console code

The game skips files in specially named folders based on platform. This lets you split code without any Lua checks:

| Folder name | Loads on |
|---|---|
| `PC/` | Windows and Mac. **Not** in ForceConsoleFlow. |
| `Console/` | Consoles **and** ForceConsoleFlow. |
| `Keyboard/` | PC only. Not on console, not in ForceConsoleFlow. |
| `Gamepad/` | Everywhere. |
| `Playstation/`, `XB1/` | Only that console family. |

You list every file in the manifest as usual, and the game drops the ones that don't apply. The real addons show three layouts, from the most separation to the least:

**A. Shared data, separate implementations** (LoreBooks):
```
Constants.lua
Language\$(language)\$(language).lua
BookshelfData.lua           -- shared static data
PC/Settings.lua             -- LAM panel
PC/LoreBooks.lua            -- full-featured PC build (~2,150 lines)
PC/LoreBooks.xml
console/LoreBooks.lua       -- trimmed console build (~600 lines)
```
Pick this when the console version is a deliberately reduced product. The cost is maintaining two copies of the logic.

**B. One code base, split UI layer** (Dolgubon's Lazy Writ Crafter):
```
Keyboard/WritCreater.xml
Console/ConsoleWritCreater.xml
WritCreater.lua             -- shared logic, with runtime checks where behaviour differs
SettingsMenu.lua            -- LAM panel
Console/ConsoleSettingsMenu.lua   -- LibHarvensAddonSettings panel
```
Pick this when the features are the same and only the presentation differs. Most cross-platform addons should start here.

**C. Runtime checks only.** Gate code with the predicates in §4, or check that the thing you need exists (`if SOME_PC_CONTROL then … end`). Use this for small differences inside otherwise shared files, not as the main split. A file full of `if console then … else … end` blocks becomes layout B done badly.

In every layout, **expect PC-only controls and globals to be missing** on console. ESO on console doesn't load any of the keyboard UI code, so `ZO_SomeKeyboardWindow` is `nil`. Hooks and `SecurePostHook` targets that point into keyboard UI must be guarded too.

## 4. Detecting the platform at runtime

Several predicates exist, and each answers a different question:

| You want to know… | Use |
|---|---|
| "Is the keyboard UI available?" (can I build a keyboard window, register a LAM panel?) | `IsKeyboardUISupported()` |
| "Am I in the console environment?" (resource limits, console-private functions, console SavedVariables) | `ZO_IsConsoleOrGameCoreUI()` |
| "Am I on a physical console?" (rarely needed) | `IsConsoleUI()`. Also true in ForceConsoleFlow. |
| "Which console family?" (platform-specific text or icons) | `ZO_IsPlaystationPlatform()` or `GetUIPlatform()` |
| "Is this PC player using a controller right now?" | `IsInGamepadPreferredMode()`. This is a UI-mode question, **not** a platform one. |

The trap: the Microsoft Store PC build is really the Xbox client ("GameCore"). On it, `IsConsoleUI()` returns **false**, yet the keyboard UI isn't loaded and console limits apply. Code that uses `IsConsoleUI()` to decide "should I show keyboard UI" breaks there. **Ask the capability question** (`IsKeyboardUISupported()`) for UI decisions, and `ZO_IsConsoleOrGameCoreUI()` for environment decisions. LibAddonMenu does this: `RegisterAddonPanel` branches on `IsKeyboardUISupported()` (`LibAddonMenu-2.0.lua:1042`). Dolgubon's Lazy Writ Crafter uses `ZO_IsConsoleOrGameCoreUI()` throughout.

Don't confuse the two gamepad cases. A PC player in gamepad mode still has the keyboard UI loaded and no resource limits. A console player has only the gamepad UI, always.

## 5. UI on console

- **Gamepad UI only.** Anything visible has to be built on gamepad templates, scenes and fragments. If your PC addon has no gamepad mode yet, that's the bulk of the port. Use gamepad fonts; not every keyboard font exists on console.
- **Labels are capped at 10,000 characters** on console. Long help text or changelogs need paging or splitting.
- **Settings menus: LibHarvensAddonSettings (LHAS).** It's the console settings library, and it works on PC too. Two routes:
  - **Native LHAS panel** (Dolgubon's Lazy Writ Crafter, `Console/ConsoleSettingsMenu.lua`): `LibHarvensAddonSettings:AddAddon(name, options)`, then `settings:AddSetting{ type = LHA.ST_CHECKBOX, … }`. Use this when you want the console menu to feel native.
  - **LAM auto-conversion.** Recent LibAddonMenu builds (r43+) ship a `.addon` manifest, take LHAS as an optional dependency, and turn a LAM panel into an LHAS panel when the keyboard UI is missing. Only some control types convert: slider, header, checkbox, colorpicker, button, editbox, dropdown, submenu, description and divider. `custom`, `texture` and `iconpicker` controls are dropped. Mark any option that should stay PC-only with `isPCOnly = true` and the converter skips it. This is fine for simple panels; test every control in ForceConsoleFlow.
  Either way, check for the library before using it (`if not LibHarvensAddonSettings then return end`) and declare it in `ConsoleDependsOn`.
- **Context menus and actions.** LibCustomMenu is keyboard-only. Its console counterpart is **LibConsoleDialogs**, which adds menus and keybind-strip actions. **LibRadialMenu** adds a radial wheel the user binds through LHAS.
- **Keybindings.** You can't add entries to the custom keybinding system on console, so `bindings.xml` actions are unreachable there. Expose actions through a **keybind-strip descriptor** that reuses the game's existing gamepad keybinds, or through LibConsoleDialogs or LibRadialMenu.
- **Display names.** Console players see platform names, not `@names`, and crossplay is moving the API toward separate crossplay and platform display names. Run any name you show through `ZO_FormatUserFacingDisplayName` rather than printing the raw string.

## 6. The resource budget (the part that bites)

All add-ons on a console share **one memory pool and one per-frame CPU budget**. When the memory pool is exhausted, the game **disables every add-on**, not just the guilty one, and shows a notification. At launch the pool was about 100 MB and the CPU budget about 1 s of add-on time per frame. ZOS has adjusted these since, so **don't hardcode them**; read them at runtime:

```lua
-- Memory: shared pool, all add-ons combined
local usedMB, capMB = GetTotalUserAddOnMemoryPoolUsageMB(), GetTotalUserAddOnMemoryPoolCapacityMB()
-- CPU: shared per-frame budget
local usedMS, availMS = GetTotalUserAddOnCPUTimeUsedNowMS(), GetTotalUserAddOnCPUTimeAvailableEachFrameMS()
```

Patterns that follow from this:

- **Yield on the shared budget, not just your own.** Add-on chunking already targets a per-chunk time (see `async`). On console, also stop a chunk early when the *shared* frame budget is running low, so you don't push everyone over:
  ```lua
  local function ShouldYield(chunkStartMS, myBudgetMS)
    if GetGameTimeMilliseconds() - chunkStartMS >= myBudgetMS then return true end
    if ZO_IsConsoleOrGameCoreUI() then
      -- leave headroom for every other add-on sharing the frame
      return GetTotalUserAddOnCPUTimeUsedNowMS() >= GetTotalUserAddOnCPUTimeAvailableEachFrameMS() * 0.5
    end
    return false
  end
  ```
- **Never sweep an entire data set.** A loop like `for itemId = 1, 250000 do GetItemLinkName(...)` makes the client load every item definition. That's roughly twice the whole pool, it's permanent until a client restart, and it's billed to add-ons. Work from what the player actually has (bags, journal, collections API) or from a precomputed ID list shipped with the addon. This is the console version of the inventory skill's "iterate what exists" rule.
- **Load static data lazily.** Large shipped tables (pin coordinates, set data, recipe lists) count against the pool as soon as their file loads. Partition by zone or category and build the table on first use, as HarvestMap does with zones. Or ship a trimmed console data set in `Console/`, as LoreBooks does.
- **Cap what you accumulate.** Fight logs, chat history and caches need hard limits on console. CombatMetrics' fight-history cap is the model.
- **Measure, don't guess.** Profile with ESOProfiler in ForceConsoleFlow and watch the pool numbers above while you play. The base game also has a toggleable HUD readout of add-on memory use (`ZO_AddOnMemoryDisplay` in `hudwarnings.lua`).

## 7. SavedVariables on console

The `ZO_SavedVars` API is the same (see `savedvars`), but the storage behaves differently:

- **There's a separate disk quota for all add-ons' SavedVariables.** `GetTotalUserAddOnSavedVariablesDiskCapacityMB()`, `GetTotalUserAddOnSavedVariablesDiskUsageMB()` and `GetUserAddOnSavedVariablesDiskUsageMB(addOnIndex)` report it. When it's full, saves **fail** with a "not enough disk space" notification, so data written since the last successful save is lost. Prune old data on load, and keep per-character tables from growing without bound.
- **Files are minified and may live in the cloud.** Xbox stores them in cloud storage, with a local cache under randomised file names. Nobody, not the player or a companion app, can read or edit them. That rules out external-tool sync and "edit your SavedVariables file to fix X" support advice.
- **No data crosses platforms.** A PC player's SavedVariables don't follow them to console. Even ForceConsoleFlow on PC writes to a separate `live/SavedVariablesUser/` folder. Defaults must produce a working addon on first run.
- **Players can wipe data for uninstalled add-ons.** The console add-on manager offers to clear unused SavedVariables to free quota, so don't rely on data surviving an uninstall/reinstall cycle.

## 8. Other things that work on PC and break on console

- **Old API aliases are missing.** ZOS keeps renamed functions and constants alive for old addons, but the PC alias layer (`esoui/ingame/addoncompatibilityaliases/pc/`) isn't loaded on console. Only the much smaller shared layer is. Code still calling a deprecated name errors on console. Grep the PC alias file in the source mirror for any old-style name you use.
- **Some public functions are private on console.** `SetSetting` and `StartChatInput` are known examples: calling them from add-on code raises an insecure-code error on console. There's no list; find them by testing in ForceConsoleFlow.
- **`GetWorldName()` returns console realm names** (`XB1live`, `PS4live-eu`, …), not the PC megaserver names. Data keyed by hardcoded PC world names never matches.
- **`GetAddOnRootDirectoryPath()` returns a different shape** (`user:/AddOnsManaged/<guid>/MyAddon/`) on a real console than in ForceConsoleFlow. Don't parse it or build paths from it.

## 9. Testing without a console

1. **ForceConsoleFlow.** Set `ForceConsoleFlow.2` to `1` in `UserSettings.txt`, or in game run `/script SetCVar("ForceConsoleFlow.2", "1")`, then reload. It loads `Console/` folders, skips `PC/` and `Keyboard/` folders, ignores `.txt` manifests, simulates the resource limits and uses the gamepad UI. A dev-only toggle is handy:
   ```lua
   if GetDisplayName() == "@YourAccount" then
     SLASH_COMMANDS["/console"] = function()
       SetCVar("ForceConsoleFlow.2", IsConsoleUI() and "0" or "1")
     end
   end
   ```
   Without a controller, bind the simulated gamepad buttons in Controls. **Alt** opens the system menu, where you can toggle add-ons. **Turn it off before switching servers**: the UI may show EU while your character data stays NA.
2. **Xbox Play Anywhere build** (the Microsoft Store client). It's the real Xbox client on PC and connects to Xbox servers, with the real add-on manager and gamertags. Add-ons go in `%LocalAppData%\Elder Scrolls Online\pccert\CachedData\AddOnsManaged`. Here `IsConsoleUI()` is false and `IsGameCoreUI()` is true (see §4), which makes it a good check that your platform predicates are right.
3. **A real console** is the only place to catch PlayStation filename-casing bugs and true realm and path values.

## 10. Publishing

- Console add-ons are distributed **only through Bethesda.net** (`mods.bethesda.net`). ESOUI and Minion can't install them. Players install through the in-game add-on browser.
- You need a **Bethesda.net account** and the **ZOS console add-on uploader tool** (linked from elderscrollsonline.com). Do the first upload with the tool. Later updates can be scripted; the community GitHub Action `m00nyONE/bnet-upload` is the usual route.
- The description is mostly read in the in-game browser, which uses the game's own markup (`|cRRGGBB…|r` works there). The Markdown-like formatting the website shows doesn't all carry over.
- **Community etiquette** (enforced by reports to Bethesda): don't port someone else's PC addon or library without asking the author first, don't reuse its name, and credit any code you adapt in the description. If you upload a private test build of a library, gate it so it only runs for you (`if GetDisplayName() ~= "@You" then return end`), and remove the upload afterwards.

## 11. Porting checklist

- [ ] Rename the manifest to `.addon`. Delete the `.txt`.
- [ ] Move PC-only dependencies to `## PCDependsOn`, add console ones to `## ConsoleDependsOn`. Replace embedded libraries with dependencies.
- [ ] Move keyboard XML and LAM settings into `PC/` or `Keyboard/`. Add `Console/` equivalents (gamepad UI, LHAS panel, or LAM auto-conversion).
- [ ] Replace `IsConsoleUI()` UI checks with `IsKeyboardUISupported()`, and environment checks with `ZO_IsConsoleOrGameCoreUI()`.
- [ ] Guard every hook and global that points into keyboard UI.
- [ ] Replace custom keybinds with keybind-strip, LibConsoleDialogs or LibRadialMenu actions.
- [ ] Remove deprecated API names (check against the PC alias file).
- [ ] Audit memory: no whole-data-set sweeps, lazy static data, capped caches. Profile in ForceConsoleFlow.
- [ ] Audit SavedVariables size and pruning. Make defaults work on a fresh install.
- [ ] Check filename casing across the manifest and texture paths. Keep the package under 500 files.
- [ ] Test in ForceConsoleFlow, ideally also in the Xbox Play Anywhere build.
- [ ] Ask the original author before publishing a port of someone else's addon.
