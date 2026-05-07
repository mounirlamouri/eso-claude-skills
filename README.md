# eso-claude-skills

A Claude Code plugin that bundles skills for writing **Elder Scrolls Online (ESO) addons**. Each skill captures patterns and dos/don'ts mined from real, popular addons — not API reference material, which goes stale every patch.

## What's inside

A single plugin, `eso-addon-dev`, containing 8 skills:

| Skill | Loads when you're… |
|---|---|
| `core` | Working in any ESO addon directory (manifest, project layout, init lifecycle, localization). |
| `ui` | Building settings panels (LibAddonMenu), custom XML, tooltips, scene/fragment integration. |
| `savedvars` | Designing or migrating persisted data with `ZO_SavedVars` — including manual SavedVariables for huge datasets. |
| `events-and-hooks` | Registering events, choosing `ZO_PreHook` vs `SecurePostHook`, avoiding taint. |
| `inventory` | Iterating bags, identifying items reliably across patches, handling cross-bag/cross-character data. |
| `async` | Writing non-blocking work — chunking, frame-budget discipline, choosing among `LibAsync` / `RegisterForUpdate` / `zo_callLater`. |
| `libraries` | Picking dependencies — decision guide for the common `Lib*` packages. |
| `api-lookup` | Looking up exact function signatures, event constants, scenes, fragments — points Claude at a local clone of `github.com/esoui/esoui`. |

## Install

Once this repo is published on GitHub, in any Claude Code session:

```
/plugin marketplace add mounirlamouri/eso-claude-skills
/plugin install eso-addon-dev@eso-claude-skills
```

Update later:

```
/plugin update eso-addon-dev@eso-claude-skills
```

### Local install (for iteration)

If you're hacking on the skills:

```
/plugin marketplace add /absolute/path/to/eso-claude-skills
/plugin install eso-addon-dev@eso-claude-skills
```

## How the skills work

Skills auto-load when their `description` matches what you're doing. Working on a `.txt` manifest pulls in `core`. Asking about settings panels pulls in `ui`. You can also invoke them explicitly: `/eso-addon-dev:savedvars`.

The `api-lookup` skill expects you to have the official ESO UI source cloned locally for fast, offline grep:

```
git clone https://github.com/esoui/esoui.git
```

Point Claude at the clone path the first time you use it; the skill remembers nothing on its own but tells Claude how to use the clone.

## Design notes

- **Patterns over API mirrors.** Skills teach idiom and judgment ("when to reach for which library", "what trips people up with SavedVariables migration"). They do *not* enumerate API functions, event constants, or enum values — that information lives in `github.com/esoui/esoui` and is refreshed every patch.
- **GitHub is the only authoritative source skills consult.** The community wiki (`wiki.esoui.com`) is a useful human reference but blocks programmatic fetch (HTTP 403). The forum API dumps are mirrored in the source repo. Skills are instructed never to fetch them.

## Further reading (for humans)

These are useful if you're a human reading docs, but the skills don't and shouldn't touch them:

- [ESOUI Wiki main page](https://wiki.esoui.com/Main_Page) — community-maintained reference (API, Events, Constants, tutorials).
- [ESOUI Wiki: API](https://wiki.esoui.com/API), [Events](https://wiki.esoui.com/Events), [Globals](https://wiki.esoui.com/Globals), [Constant_Values](https://wiki.esoui.com/Constant_Values).
- [APIVersion history](https://wiki.esoui.com/APIVersion) — what's in each patch.
- [ESOUI forums: Tutorials & Other Helpful Info](https://www.esoui.com/forums/forumdisplay.php?f=172) — Dan Batson's per-patch API dumps live here.
- [github.com/esoui/esoui](https://github.com/esoui/esoui) — the official UI Lua source mirror, refreshed per patch. **This is what `api-lookup` skill points at.**

## Source addons

The skills cite real code from these addons (installed locally during authoring):

- **LoreBooks** — small/medium addon shape, LibMapPins, LibAddonMenu.
- **SkyShards** — paired collection-tracker idiom; per-character SavedVariables; compass pin integration.
- **HarvestMap** — manual SavedVariables, multi-addon zone partitioning, byte-packed serialization.
- **DolgubonsLazyWritCreator** — event-driven queue, LibLazyCrafting, cross-bag iteration.
- **CombatMetrics** — LibCombat abstraction, adaptive chunking, fight-history caps.
- **TamrielTradeCentre** — external-tool sync via SavedVariables, item-ID keying, tooltip extension.

## License

MIT.
