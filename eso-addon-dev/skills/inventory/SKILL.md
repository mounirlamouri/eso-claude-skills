---
description: Iterating ESO bags and items, identifying items reliably across game patches, handling cross-bag and cross-character data, and writing performant inventory scans. Use when working with BAG_* constants, GetBagSize, GetItemInfo, GetItemLink, item type filtering, or any code that walks the player's inventory/bank.
---

# ESO inventory and items

Bags in ESO are flat arrays of slots, addressed by `(bagId, slotIndex)`. Walking them is straightforward; doing it efficiently and identifying items in a way that survives patches is where addons get into trouble.

## 1. The `BAG_*` constants you actually use

| Constant | What it is |
|---|---|
| `BAG_BACKPACK` | The player's carried inventory. |
| `BAG_BANK` | The shared account bank. |
| `BAG_SUBSCRIBER_BANK` | ESO Plus subscriber bank slots (separate bag from `BAG_BANK`). |
| `BAG_VIRTUAL` | The "Craft Bag" — infinite stack of crafting materials for ESO Plus. |
| `BAG_GUILDBANK` | The currently-open guild bank (only valid while interacting with one). |
| `BAG_HOUSE_BANK_ONE` … `BAG_HOUSE_BANK_TEN` | Per-house bank chests. |
| `BAG_WORN` | Equipped gear. |
| `BAG_COMPANION` | Active companion's inventory. |

**Don't iterate "all bags" in a loop over the integer range** — bag IDs are sparse, some require interaction, and new ones get added per patch. Maintain an explicit whitelist for what you scan, like DolgubonsLazyWritCreator does:

```lua
local bags = { BAG_BANK, BAG_SUBSCRIBER_BANK, BAG_BACKPACK,
               BAG_HOUSE_BANK_ONE, BAG_HOUSE_BANK_TWO, ... }
for _, bagId in ipairs(bags) do
  for slot = 1, GetBagSize(bagId) do
    -- ...
  end
end
```

`BAG_VIRTUAL` is special — for materials, search it *first* and most queries are answered without touching the regular bags.

## 2. Identifying items reliably

This is the gotcha that most new addons get wrong. **Item IDs as exposed by `GetItemId()` are not stable across patches** — ZOS occasionally re-IDs items in major updates. Equally, `GetItemLink()` returns a string that bakes in level/quality/trait, so two "logically same" items have different links.

The pattern that survives patches: key by **name + specialized item type + relevant traits**, not raw IDs.

TamrielTradeCentre maintains a per-language `ItemLookUpTable` that maps `(itemName, specializedItemType) → stable ID`:

```lua
function TTC:NameSpecializedItemTypeToItemID(itemName, specializedItemType)
  return self.ItemLookUpTable[itemName][specializedItemType]
end
```

For identifying an item from a slot:

```lua
local link = GetItemLink(bagId, slotIndex)
local itemType, specializedItemType = GetItemLinkItemType(link)
local quality = GetItemLinkQuality(link)
local trait   = GetItemLinkTraitType(link)
local level   = GetItemLinkRequiredLevel(link)
```

Combine the fields you actually care about. A weapon's identity for a price tracker is `(name, specializedItemType, quality, trait, level)` — the keys TTC nests by.

For "is this the kind of item I care about?" use **specialized item types**, which are finer-grained than `itemType`:

```lua
local _, specialized = GetItemType(bagId, slotIndex)
if specialized == SPECIALIZED_ITEMTYPE_TROPHY_SURVEY_REPORT then ... end
```

DolgubonsLazyWritCreator keys all its survey-report scans this way.

## 3. Walking a bag efficiently

```lua
local size = GetBagSize(bagId)        -- cache once
for slot = 0, size - 1 do             -- ESO uses 0-indexed slots in some APIs, 1-indexed in others — check the function
  local link = GetItemLink(bagId, slot)
  if link ~= "" then                  -- empty slots return ""
    -- decode link / GetItemInfo
  end
end
```

`GetBagSize` is cheap but allocating; cache it once at the top of the loop. `GetItemInfo(bagId, slot)` returns name/icon/stack/sellPrice/meetsUsageRequirement/equipped/quality — fewer string allocations than parsing the link, so prefer it when you only need scalars.

## 4. Cross-bag accumulation

Counting "how many of X do I have, anywhere"? The whitelist + `BAG_VIRTUAL` first pattern:

```lua
local function countItem(itemId)
  local total = GetItemTotalCount(BAG_VIRTUAL, itemId)
  for _, bagId in ipairs({ BAG_BACKPACK, BAG_BANK, BAG_SUBSCRIBER_BANK }) do
    for slot = 0, GetBagSize(bagId) - 1 do
      if GetItemId(bagId, slot) == itemId then
        local _, stack = GetItemInfo(bagId, slot)
        total = total + stack
      end
    end
  end
  return total
end
```

For materials specifically, `BAG_VIRTUAL` already aggregates everything ESO Plus subscribers have, so often you can stop there.

## 5. Don't do full scans on hot paths

A full inventory scan is O(slots) — 200+ for the backpack alone, more for stocked banks. Don't do it inside:
- High-frequency events (`EVENT_INVENTORY_SINGLE_SLOT_UPDATE`, combat events, `OnUpdate`).
- Tooltip rendering.
- Loops that run on every player action.

When you need fresh totals, **cache and invalidate on `EVENT_INVENTORY_SINGLE_SLOT_UPDATE`** rather than recomputing per query. Or, for bulk operations, defer to the chunking pattern in the `async` skill.

## 6. Houses, guild banks, companion bags

Three bags are not always available:

- **Guild banks** — only valid while the player has the guild bank UI open. Outside that, querying them errors. React to `EVENT_GUILD_BANK_OPEN` / `EVENT_GUILD_BANK_CLOSE` to gate guild-bank logic.
- **House banks** — only the currently-loaded house's banks are queryable. Travelling to a different house unloads the previous one's data.
- **Companion bag** — only when a companion is summoned and active.

Don't assume any of these are present; check `GetBagSize(bagId) > 0` and gate on the relevant open/active events.

## 7. Item links — minimal parsing

An item link string looks like `|H1:item:43009:30:50:0:0:0:...|h[Soul Gem]|h`. Avoid hand-parsing. The `GetItemLink*` family of functions decodes everything:

- `GetItemLinkName(link)`
- `GetItemLinkItemType(link)` → `itemType, specializedItemType`
- `GetItemLinkQuality(link)`
- `GetItemLinkTraitType(link)` / `GetItemLinkArmorType(link)` / `GetItemLinkWeaponType(link)`
- `GetItemLinkSetInfo(link)` for set pieces
- `GetItemLinkRequiredLevel(link)` / `GetItemLinkRequiredChampionPoints(link)`

For the full surface, grep the live source via the `api-lookup` skill — don't memorise.

## 8. Practical do/don't

- **Do** maintain an explicit bag whitelist; **don't** iterate over a numeric range of bag IDs.
- **Do** key items by `(name, specializedItemType, …)`; **don't** rely on `GetItemId()` for cross-patch persistence.
- **Do** check `BAG_VIRTUAL` first for materials.
- **Do** cache `GetBagSize()` at loop top; **don't** call it per iteration.
- **Don't** scan on every event — debounce or batch via `RegisterForUpdate` (see `events-and-hooks`).
- **Don't** assume guild/house/companion bags are open; gate on their availability events.
