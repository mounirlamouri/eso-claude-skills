---
description: Looking up exact ESO API specifics — function signatures, event names and payloads, constant values, scene and fragment names, control templates, or any precise game-engine detail. Use when you need an authoritative answer about what the game exposes, not just patterns or idioms. Directs to the official esoui/esoui source mirror.
---

# Looking up ESO API specifics

The other skills in this plugin teach **patterns**. When you need a *specific* answer — exact arguments to `GetItemInfo`, the parameter list of `EVENT_COMBAT_EVENT`, the value of `SPECIALIZED_ITEMTYPE_TROPHY_SURVEY_REPORT`, the names of all scenes the inventory uses — go to the source.

## The single authoritative source

[**`github.com/esoui/esoui`**](https://github.com/esoui/esoui) is the official mirror of ZeniMax's UI Lua source code. It is refreshed every patch (default branch `live`). Everything the game exposes to addons is in there: API function definitions, event constants, all enum values, scenes, fragments, control templates, the lot.

**Do not** consult the ESOUI wiki (`wiki.esoui.com`) or forum threads programmatically:
- The wiki returns **HTTP 403** to programmatic fetches — it can't be reached from a tool call.
- The forum patch dumps contain the same content as the source mirror — they're already in `esoui/esoui`.

The wiki and forums are valuable to humans browsing in a real browser, but skills should not try to fetch them.

## Setup: clone the source once

The fast, offline, deterministic path is a local clone:

```bash
git clone https://github.com/esoui/esoui.git
```

Put it somewhere stable (e.g. `~/src/esoui` or `C:\src\esoui`). The clone is ~250 MB and takes a minute. After that, every lookup is a local grep.

**If the user hasn't cloned yet**, ask them to. That's a one-time cost; it pays back on every subsequent lookup. Suggested prompt:

> I'd like to look up an ESO API specific. The fast path is a local clone of the official source mirror. Run:
> `git clone https://github.com/esoui/esoui.git`
> somewhere convenient (e.g. `~/src/esoui`), then tell me the path.

Once they share the path, remember it within the current session and grep there.

## Repo layout

Inside the clone, the directories you'll grep:

| Path | Contents |
|---|---|
| `esoui/ingame/` | Game-side UI: inventory, combat, crafting, scenes, fragments, action bar, group, guild. Most addon-relevant code is here. |
| `esoui/publicallingames/` | Code shared between PC and console. Tooltip helpers, common scenes, base controls. |
| `esoui/libraries/` | Foundational utilities ZOS itself uses — `zo_callLater`, `ZO_Object`, `ZO_PreHook`, `SecurePostHook`, scene manager internals. |
| `esoui/pregame/` | Pre-character-select UI. Rarely needed by addons. |
| `esoui/internalingame/` | Misc shared UI. |

## Lookup recipes

**"What arguments does `GetItemInfo` take and return?"**
```
grep -rn "function GetItemInfo" /path/to/esoui/clone
# or, since GetItemInfo is engine-side, find its callers:
grep -rn "GetItemInfo(" /path/to/esoui/clone | head -20
```
Engine functions (`GetItemInfo`, `GetUnitName`, etc.) aren't *defined* in the clone — they're C functions exposed to Lua. But every callsite shows you exactly what arguments they pass and what they unpack from the return.

**"What does `EVENT_COMBAT_EVENT` deliver?"**
```
grep -rn "EVENT_COMBAT_EVENT" /path/to/esoui/clone | head
```
Look at any `RegisterForEvent(..., EVENT_COMBAT_EVENT, callback)` and read the callback's signature.

**"What's the value of `SPECIALIZED_ITEMTYPE_TROPHY_SURVEY_REPORT`?"**
```
grep -rn "SPECIALIZED_ITEMTYPE_TROPHY_SURVEY_REPORT" /path/to/esoui/clone
```
Constants are usually defined in `esoui/ingame/globals/` or `esoui/publicallingames/globals/`. The first match in a `_globals.lua` or `_constants.lua` file is the definition.

**"What scenes exist?"**
```
grep -rn "ZO_Scene:New(" /path/to/esoui/clone
# or
grep -rn "SCENE_MANAGER:GetScene" /path/to/esoui/clone | head -30
```

**"What's the current live (or PTS) APIVersion?"**
The repo uses two long-lived branches: `live` and `pts`. Each branch's `README.md` declares the version, in the form `version X.Y.Z (API NNNNNN)`. Easiest path:
```
git -C /path/to/esoui/clone fetch origin live pts
git -C /path/to/esoui/clone show origin/live:README.md | grep -E 'API [0-9]{6}'
git -C /path/to/esoui/clone show origin/pts:README.md  | grep -E 'API [0-9]{6}'
```
Or fetch directly:
- `https://raw.githubusercontent.com/esoui/esoui/live/README.md`
- `https://raw.githubusercontent.com/esoui/esoui/pts/README.md`

For an interactive lookup that also recommends the right `## APIVersion:` manifest line, use the `/eso-addon-dev:check-api-version` slash command.

**"What's the right SecurePostHook target for X?"**
Find ZO_*'s definition, look at the function name, hook that.
```
grep -rn "function ZO_InventorySlot" /path/to/esoui/clone
```

## Fallback when there's no local clone

If the user doesn't want to clone, fall back to fetching individual files from GitHub:

- Raw file URL: `https://raw.githubusercontent.com/esoui/esoui/live/esoui/ingame/inventory/inventory.lua`
- GitHub API for searching: `gh api -X GET 'search/code' -f q='SPECIALIZED_ITEMTYPE_TROPHY_SURVEY_REPORT repo:esoui/esoui'`
- Or browse the tree: `https://github.com/esoui/esoui/tree/live/esoui`

This works but is rate-limited and slower than a local clone. Push for the clone when the user is doing more than one or two lookups in a session.

**Don't** use the WebFetch tool against the wiki — it returns 403. Don't use it against the forum threads either; their content lives in the source mirror.

## What the clone *doesn't* cover

- **Engine-side C functions** (everything that's not declared `function` in Lua). The clone shows you their callsites, not their implementations. For these, infer behaviour from how ZOS itself uses them.
- **Per-patch deltas as a single diff.** If the user asks "what changed in U41 for inventory?", a `git log live --oneline -- esoui/ingame/inventory/` against the clone gives an answer; the [forum APIVersion threads](https://www.esoui.com/forums/forumdisplay.php?f=172) frame the same change as patch notes for human reading.
- **Localized strings.** The `SI_*` string IDs are declared but the localized values come from the game client's data files, not Lua. Use `GetString(SI_*)` at runtime.

## Scope of this skill

Use this skill when the question is "what *exactly* does X look like in ESO?" — signatures, names, values, layouts. For "how do I idiomatically do Y?" — that's the job of the other skills (`core`, `ui`, `events-and-hooks`, `inventory`, `savedvars`, `async`, `libraries`).
