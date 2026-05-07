---
description: Persisting ESO addon data with ZO_SavedVars, designing SavedVariables migrations, choosing account-wide vs per-character vs per-server storage, or managing large persisted datasets and external-tool sync. Use when working with the ## SavedVariables manifest directive, ZO_SavedVars constructors, or migration logic.
---

# ESO SavedVariables — patterns and pitfalls

ESO addons can't do file or network I/O. **All persistence happens through SavedVariables tables**, which the game serializes to `<account>/SavedVariables/<AddonName>.lua` on logout. Get this layer wrong and you lose user data on a patch.

## 1. Pick the right constructor

`ZO_SavedVars` has several factory methods. Choose by the question "who owns this data?":

| Question | Use |
|---|---|
| Same on every character? (UI prefs, account-wide unlocks) | `ZO_SavedVars:NewAccountWide(name, version, namespace, defaults)` |
| Per-character (build, gear preferences, character-specific notes) | `ZO_SavedVars:NewCharacterIdSettings(name, version, namespace, defaults)` |
| Want both? Common prefs + character overrides | A toggle in your settings; store both tables, switch via accessor |

Examples from the corpus:

- **LoreBooks** (account-wide, UI is global): `LoreBooks.db = ZO_SavedVars:NewAccountWide("LBooks_SavedVariables", internal.SAVEDVARIABLES_VERSION, nil, LoreBooks.defaults)`.
- **SkyShards** (per-character, because skyshard discovery is character-bound): `ZO_SavedVars:NewCharacterIdSettings("SkyS_SavedVariables", 4, nil, defaults)`.
- **DolgubonsLazyWritCreator** offers both, gated by `useCharacterSettings`:
  ```lua
  function WritCreater:GetSettings()
    if self.savedVars.useCharacterSettings then return self.savedVars
    else return self.savedVarsAccountWide.accountWideProfile end
  end
  ```

`namespace` (3rd arg) lets you nest multiple settings groups under one declared SavedVariable. Pass `nil` for a flat top-level table.

## 2. Defaults: shape matters more than you think

The `defaults` table you pass at construction is **shallow-merged on each load**. Missing keys get filled in. **This means you can add new fields safely between releases** — old user data gains the new keys with their defaults. Renaming or restructuring keys is what hurts.

```lua
LoreBooks.defaults = {
  compassMaxDistance = 0.04,
  pinTexture = { type = ..., size = 26, level = 40 },  -- nested table OK
  filters = { [PINS_UNKNOWN] = true, [PINS_COLLECTED] = false },
}
```

**Don't put large data in defaults.** Defaults should describe configuration. Bulk data (collected items, harvest nodes, price corpus) is populated at runtime, not seeded from defaults.

## 3. The version field is a trap

The 2nd arg to `NewAccountWide` / `NewCharacterIdSettings` is a version number. **Bumping it wipes the user's data and replaces it with defaults** — there is no automatic migration. ESO simply discards data it considers older.

Treat version bumps as destructive. Bump only when you cannot keep the old shape readable. When you do bump, ship migration code that runs *before* you call the constructor (read raw `_G[savedVarName]`, transform, then construct).

**TamrielTradeCentre** uses an explicit `ActualVersion` integer inside the data and a dedicated upgrader (`SavedVarUpgrader.lua`) — separate from the ZO version field. Pattern: keep the ZO version field stable, version your data internally, run migrations on load. This avoids accidental wipes.

## 4. When to bypass ZO_SavedVars entirely

For huge or oddly-shaped data, build your own structure on the bare global table:

```lua
-- HarvestMap pattern: declare via manifest, manage manually
Harvest_SavedVars = Harvest_SavedVars or {}
Harvest_SavedVars.global = Harvest_SavedVars.global or defaultGlobalSettings
Harvest_SavedVars.account = Harvest_SavedVars.account or {}
Harvest_SavedVars.account[GetDisplayName()] = ...
```

Pair this with the manifest directive `## DisableSavedVariablesAutoSaving: 1` on submodule manifests so ESO doesn't race your manual writes.

## 5. Partition huge datasets across multiple addons

ESO loads each `## SavedVariables` declaration as one Lua file. A 50 MB SavedVariables file is slow to load and risks corruption. **HarvestMap** splits node data across sibling addons — `HarvestMapAD` (Aldmeri zones), `HarvestMapDC`, `HarvestMapEP`, `HarvestMapDLC`, `HarvestMapNF` — each declaring its own SavedVariable. The main addon stays small; data addons can be loaded selectively.

This pattern is overkill for most addons. Reach for it only when one zone's data exceeds tens of MB.

## 6. Size management techniques

When data grows, in order of difficulty:

- **Cap with pruning.** CombatMetrics keeps the last 25 fights in memory and 50 on disk; on overflow it removes the oldest non-boss fight first. TamrielTradeCentre caps `MaxAutoRecordStoreEntryCount = 20000`.
- **Abbreviate dict keys.** TTC's price tables use single-letter keys: `["A"]` (avg), `["X"]` (max), `["N"]` (min), `["EC"]` (entry count). Multiplied across millions of records this halves the file.
- **Pack scalars.** HarvestMap packs each node's `(x, y, z, day)` into 8 bytes via `string.byte`/`string.char`, decoded on load (`(x1 * 256 + x2) * 0.2`). Roughly 4× smaller than `string.format("%.1f", ...)`.
- **Lazy-load by partition.** HarvestMap loads only the current zone's node cache; other zones are evicted via LRU when memory pressure hits.

## 7. External-tool sync via SavedVariables (TTC pattern)

Want a desktop app to sync addon data? The contract is: **the desktop tool reads/writes the `.lua` SavedVariables file directly** while the game is closed. Your addon publishes data into well-known top-level tables; the tool parses the Lua syntax.

For this to work:
- Use stable, explicit table names (`PriceTable`, `Guilds`, `Entries`).
- Include a version integer the tool can read (`ActualVersion = 11`).
- Include timestamps so the tool knows what's fresh.
- Don't use Lua features that are hard to parse — stick to simple nested tables of scalars.
- Document your file format somewhere your users (and your tool) can find it.

## 8. What NOT to persist

- **Function references.** ZOS will silently drop them, then your addon crashes when it tries to call a nil.
- **UserData (control objects).** Same problem — they don't survive serialization.
- **Cyclic references.** Will hang or error during save.
- **Transient state** — what scene is open, what bag the user just looked at. Recompute on load.
- **Cached game data.** Recompute from `GetItemInfo`, `GetCollectibleInfo`, etc. on load. Caching across sessions means stale data after patches.

## 9. Read order on init

The reliable boot sequence:

```lua
EVENT_MANAGER:RegisterForEvent(ADDON_NAME, EVENT_ADD_ON_LOADED, function(_, name)
  if name ~= ADDON_NAME then return end
  EVENT_MANAGER:UnregisterForEvent(ADDON_NAME, EVENT_ADD_ON_LOADED)
  -- 1. (optional) read raw _G[savedVarName] and migrate
  -- 2. construct ZO_SavedVars
  -- 3. wire up settings UI, register slash commands, hook events
end)
```

`EVENT_ADD_ON_LOADED` is the only event guaranteed to fire after your SavedVariable global is populated. Don't touch saved data in any earlier hook.
