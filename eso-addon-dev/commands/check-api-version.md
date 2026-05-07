---
description: Find the current ESO live and PTS APIVersion for the ## APIVersion manifest line
---

The user wants to know the current ESO API version(s) to declare in their addon's `## APIVersion:` manifest line. Look it up from the official `esoui/esoui` source mirror (the only authoritative source skills consult — see the `api-lookup` skill for why).

## Lookup procedure

The `esoui/esoui` repository uses two long-lived branches that map cleanly to the two API versions an addon author cares about:

- `live` branch → current **live server** API version
- `pts` branch → current **PTS (Public Test Server)** API version

Each branch's `README.md` declares the version explicitly, in the form `version X.Y.Z (API NNNNNN)` or `Last update: X.Y.Z (API NNNNNN) on <date>`.

### Preferred path: local clone

If the user has a local clone of `esoui/esoui` (set up per the `api-lookup` skill):

1. `git -C <clone-path> fetch origin live pts` — make sure both branches are up to date.
2. `git -C <clone-path> show origin/live:README.md | head -20` — extract the live API version.
3. `git -C <clone-path> show origin/pts:README.md | head -20` — extract the PTS API version.

Parse the API version with a simple grep for `API \d{6}`.

### Fallback: raw GitHub fetch

If no clone is available:

- `https://raw.githubusercontent.com/esoui/esoui/live/README.md` — live API version.
- `https://raw.githubusercontent.com/esoui/esoui/pts/README.md` — PTS API version.

Use a WebFetch (or `curl` from Bash) and grep the output for `API \d{6}`. Both branches are public and require no authentication.

## What to report to the user

Tell the user:

1. The current **live** API version (e.g. `101049`).
2. The current **PTS** API version, if different from live (e.g. `101050`). If both branches show the same number, no transition is in progress.
3. The recommended `## APIVersion:` manifest line based on the situation:
   - **No transition** (`live` == `pts`, or the user only cares about live): `## APIVersion: <live>`.
   - **Transition in progress** (`pts` > `live`): `## APIVersion: <live> <pts>` — covers both versions during the patch window. Drop the older one once the new version lands on live.
4. The dates each version became current (visible in the READMEs), so the user knows how recent the information is.

## Important constraints

- **Do not** consult `wiki.esoui.com` (HTTP 403 to programmatic fetch) or the ESOUI forum threads — they're either unreachable or redundant with the source mirror.
- **Do not** rely on cached values from earlier in the conversation. APIVersions change with patches; always perform a fresh lookup.
- If the lookup fails (network error, branch renamed, README format changed), tell the user explicitly rather than guessing or falling back to a stale value.
