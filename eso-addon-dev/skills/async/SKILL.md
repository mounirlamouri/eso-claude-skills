---
description: Writing non-blocking work in ESO addons — chunking large operations, scheduling deferred or periodic updates, choosing between LibAsync / EVENT_MANAGER:RegisterForUpdate / zo_callLater, and preventing UI freezes during inventory scans, combat-log processing, or large data deserialization. Use when an operation might exceed a frame's budget.
---

# ESO async — keeping the UI smooth

ESO is single-threaded. Anything you do in a callback runs on the same thread that draws the UI. A 50 ms operation = a visible hitch. A 500 ms operation = users complain. The fix is to **break work into chunks that yield to the frame**.

## 1. The frame budget

At 60 fps, you have ~16 ms per frame; the engine itself uses most of that. CombatMetrics targets **10 ms per chunk** as its async budget — a good number to copy:

```lua
local desiredtime = 0.010   -- 10 ms target per chunk
```

If your work routinely exceeds this, chunk it.

## 2. Three tools, one decision

| Tool | When |
|---|---|
| `EVENT_MANAGER:RegisterForUpdate(name, ms, fn)` | Periodic work at a known interval. Throttled UI refreshes, keepalive ticks. Easy to unregister. |
| `zo_callLater(fn, ms)` | One-shot deferred call. "Run this once in 500 ms." Cannot easily cancel. |
| **LibAsync** | Long jobs that should run "as fast as possible without freezing." Adaptive sizing, queue management, cancellation, error handling. |

**Decision shorthand:**

- "Refresh this UI element every 50 ms" → `RegisterForUpdate`.
- "Wait 1 second after user input, then act" → `zo_callLater`.
- "Process 50,000 records over the next few seconds without dropping frames" → LibAsync, or hand-rolled chunked `RegisterForUpdate` (CombatMetrics-style).

## 3. The chunked-work pattern (without LibAsync)

When you want the simplest possible non-blocking loop, the CombatMetrics-style pattern is:

```lua
local function processChunk(state)
  local startTime = GetGameTimeSeconds()
  local i_end = math.min(state.i + state.chunksize, #state.data)
  for i = state.i + 1, i_end do
    process(state.data[i])
  end
  state.i = i_end

  if i_end >= #state.data then
    state.onDone()
    return
  end

  -- schedule next chunk one frame later
  EVENT_MANAGER:RegisterForUpdate("MyAddon_Chunk", 20, function()
    EVENT_MANAGER:UnregisterForUpdate("MyAddon_Chunk")
    processChunk(state)
  end)

  -- adaptive sizing: aim for desiredtime per chunk
  local elapsed = GetGameTimeSeconds() - startTime
  state.chunksize = math.min(
    math.ceil(0.010 / math.max(elapsed, 0.001) * state.chunksize / 20) * 20,
    20000)
end
```

Why this works:
- Yielding via `RegisterForUpdate` lets the engine paint a frame between chunks.
- The chunk size *adapts*: if a chunk took longer than 10 ms, next chunk is smaller; faster, next is bigger.
- Naming the update lets you cancel it if the user changes their mind.

## 4. Use LibAsync when chunk discipline gets complex

If you have multiple long jobs that need to interleave, error handling, cancellation, or progress tracking, hand-rolled chunking gets fragile. **LibAsync** (a common dependency in heavier addons) gives you:

```lua
local task = LibAsync:Create("MyJob")
task:For(1, #items):Do(function(i)
  process(items[i])
end):Then(function() onDone() end)
```

It manages chunk sizing, exposes `:Cancel()`, lets you compose tasks, and surfaces errors to a single handler. Pull it in when you have more than one async pipeline.

## 5. Lazy loading instead of async

Sometimes the right answer isn't "process faster" but "don't process at all unless needed." HarvestMap has data for every zone in the game, but only loads the **current zone's** node cache on `EVENT_PLAYER_ACTIVATED`, evicting other zones via LRU when memory pressure hits. The startup cost is bounded by zone size, not total data.

If you have data partitioned by some natural axis (zone, character, day), prefer lazy loading over chunked initialization.

## 6. Common ways async goes wrong

- **Synchronous deserialization at load.** Reading a SavedVariable into runtime structures *blocks the loading screen*. If the deserialization is large (thousands of entries), users see "stuck on load." Defer the heavy decode behind a `zo_callLater(fn, 1)` so the world activates first; show a "loading…" indicator if the data is needed for the UI.
- **Forgetting to unregister `RegisterForUpdate`.** It runs *forever* until you unregister. If you used it for a one-shot chunk loop, unregister inside the callback the moment you no longer need ticks.
- **Stacking multiple updates with the same name.** Same name = the second registration replaces the first. Looks like a bug ("my old chunk loop just stopped"). Use distinct names per concurrent job.
- **Calling secure functions from chunks.** A chunk callback is non-secure context; calling `RequestUseItem` and friends from inside one will taint. See the `events-and-hooks` skill.
- **Missing the chunk on a cancellation.** If the user toggles a feature off mid-chunk, your next scheduled tick still runs. Always check a "should I still be running?" flag at the top of each chunk.

## 7. Quick recipes

**Throttled UI refresh (40 ms cadence):**
```lua
EVENT_MANAGER:RegisterForUpdate("MyAddon_Refresh", 40, refreshUi)
-- later:
EVENT_MANAGER:UnregisterForUpdate("MyAddon_Refresh")
```

**One-shot debounce (don't react to events fired in rapid succession):**
```lua
local pending
local function onChange()
  if pending then return end
  pending = true
  zo_callLater(function() pending = nil; doWork() end, 200)
end
```

**Cap fight history without freezing on save:**
CombatMetrics keeps the last 25 fights in memory and 50 on disk; pruning runs after each fight, not in a single pass at logout. Spread the cost across the user's actual play.

## 8. The smallest rule

If a single callback does work that's O(player-data-size), assume it's too slow eventually. Chunk it before users complain — once they do, you're already losing them.
