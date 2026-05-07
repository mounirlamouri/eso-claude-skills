---
description: Registering ESO game events with EVENT_MANAGER, hooking ZOS functions safely with ZO_PreHook or SecurePostHook, avoiding taint ("Attempt to access private function"), filtering high-volume events, and choosing between OnUpdate and named RegisterForUpdate. Use when wiring event handlers, intercepting ZOS code, or debugging event-related crashes.
---

# ESO events and hooks

ESO addons are event-driven. The two operations you'll do constantly are **subscribing to game events** and **wrapping ZOS functions**. Both have sharp edges that the docs don't warn you about.

## 1. Subscribe to events

`EVENT_MANAGER:RegisterForEvent(uniqueName, EVENT_*, callback)` is the only way in.

```lua
EVENT_MANAGER:RegisterForEvent(ADDON_NAME, EVENT_PLAYER_ACTIVATED, function(_, ...) ... end)
```

The `uniqueName` is a string namespace. **Reusing it across registrations replaces, not appends.** Use your addon name as a prefix to avoid colliding with other addons.

### EVENT_ADD_ON_LOADED is special

It fires once for every addon, including yours. The right shape is:

```lua
local function OnLoad(eventCode, name)
  if name ~= ADDON_NAME then return end
  EVENT_MANAGER:UnregisterForEvent(ADDON_NAME, EVENT_ADD_ON_LOADED)  -- critical
  -- init here
end
EVENT_MANAGER:RegisterForEvent(ADDON_NAME, EVENT_ADD_ON_LOADED, OnLoad)
```

Unregister immediately after your name fires. Otherwise you keep getting called for every other addon's load and waste cycles. LoreBooks does this at `[PC/LoreBooks.lua:2102]`.

### When to defer past EVENT_ADD_ON_LOADED

`EVENT_ADD_ON_LOADED` fires before the world is ready — you can read SavedVariables but not query player state, inventory, or anything zone-aware. For init that needs the player in the world, use `EVENT_PLAYER_ACTIVATED` (fires every zone load including the first).

## 2. Filter high-volume events

Many events (combat, ability, inventory) fire dozens or hundreds of times per second. **Filter at registration**, not in your callback:

```lua
EVENT_MANAGER:RegisterForEvent(ADDON_NAME, EVENT_COMBAT_EVENT, callback)
EVENT_MANAGER:AddFilterForEvent(ADDON_NAME, EVENT_COMBAT_EVENT,
  REGISTER_FILTER_IS_ERROR, false,
  REGISTER_FILTER_SOURCE_COMBAT_UNIT_TYPE, COMBAT_UNIT_TYPE_PLAYER)
```

`AddFilterForEvent` runs in the engine before your callback is even queued. A single filter can cut event volume by 100×. `REGISTER_FILTER_*` constants vary per event — grep the live game source for what's available (see `api-lookup` skill).

### Better still: use a library that pre-filters for you

For combat, **don't register raw events at all** — use **LibCombat**. It registers the heavy events itself, fans them through a clean callback API, and lets you ignore most of the variant-handling. CombatMetrics, the canonical combat addon, does exactly this:

```lua
LC:RegisterCallbackType(LIBCOMBAT_EVENT_DAMAGE_OUT, AddToLog, ADDON_NAME)
```

The general principle: if a popular library covers an event domain, use it. You'll get filter discipline for free.

## 3. Cleanup discipline

Whatever you register, you should be able to unregister:

```lua
EVENT_MANAGER:UnregisterForEvent(ADDON_NAME, EVENT_COMBAT_EVENT)
```

Common reasons to unregister mid-lifetime: user disabled a feature in settings, you finished a one-shot init, the player left a zone where the event mattered. Track what you've registered so toggles work.

## 4. Hooks — modifying ZOS behavior

ESO ships its UI as Lua, so you can intercept any non-secure function. Two helpers:

| Helper | When |
|---|---|
| `ZO_PreHook(obj, "method", fn)` | Run *before* the original. Return `true` from your fn to **suppress** the original. |
| `SecurePostHook(obj, "method", fn)` | Run *after* the original. Cannot suppress. **Use this when the function is "secure"** (handles loot, inventory, abilities). |

### The taint problem

ESO marks certain functions as **secure** to prevent addons from automating combat/loot in unfair ways. If you call a secure function from a context that has been "tainted" — meaning a non-secure function (your hook) ran first — you'll get:

> Attempt to access private function 'X' from insecure code

This is the single most common mystery error in ESO addon development.

**Rules of thumb to avoid taint:**

1. Use **`SecurePostHook`** for any function that touches inventory, abilities, action bar, group invites, or trading. It runs after the secure code, so taint can't propagate.
2. Use **`ZO_PreHook`** only for non-secure UI code (menus, tooltips, scenes, your own controls).
3. Never call `RequestUseItem`, `CallSecureProtected`, ability or trade functions from inside a `ZO_PreHook`.
4. Don't share state via globals between your hook and a callback that may run in a secure context.

DolgubonsLazyWritCreator combines both — `SecurePostHook("CallSecureProtected", ...)` for the protected-call wrapper, and `ZO_PreHook(_G, "ZO_InventorySlot_DiscoverSlotActionsFromActionList", ...)` for the *non-secure* inventory menu UI. This is the right pattern: secure post for protected code, pre-hook for the menu rendering.

## 5. RegisterForUpdate vs OnUpdate

For periodic work (UI animation, throttled refresh), avoid hand-rolled `OnUpdate` handlers — they fire every frame and are hard to manage. Use:

```lua
EVENT_MANAGER:RegisterForUpdate("UniqueName", intervalMs, callback)
```

Named, throttled, easy to unregister. CombatMetrics uses this for its 20–40 ms throttled UI updates: `em:RegisterForUpdate("CMX_Report_Cursor_Control", 40, updatePlotCursor)`.

For one-shot deferral after the current frame, `zo_callLater(fn, delayMs)` is the simplest tool. Use it for "do this in 500 ms after the user interacts" — not for sustained polling.

For chunked work that should yield to the frame, see the `async` skill.

## 6. Patterns to internalise

- **Always namespace your `uniqueName`** with the addon name. `EVENT_MANAGER:RegisterForEvent("MyAddon_Combat", EVENT_COMBAT_EVENT, ...)` — not `"combat"`.
- **One callback per concern.** Don't register one fat callback for ten events; you'll suffer in the debugger.
- **Filter early, decode late.** `AddFilterForEvent` first, then in the callback only do the cheap dispatch. Heavy work goes through the `async` skill's chunking patterns.
- **Don't register events from inside an event callback.** It mostly works, but you'll get re-entrancy surprises. Defer with `zo_callLater(fn, 0)` if you must.
- **Hook on init, not on first use.** Hooks should be installed once during your `EVENT_ADD_ON_LOADED` handler, not lazily — otherwise your hook misses early invocations.
