# 26. The Commerce Screen — a map for mods

**Established: 2026-08-10.** Everything below is ✅ read in the game's files
(the 2026-07-28 build). The basis for the "Better Commerce Screen UI by Najane" mod.

This is the screen where the player **assigns resources to settlements and reviews trade routes**.
It opens through `ContextManager.push("screen-resource-allocation", …)`.

⚠️ **Written in `ui-next` / Solid.js**, not in the old framework — read
[25-ui-next-solidjs.md](25-ui-next-solidjs.md) first. `Controls.decorate` **will not work** here.

## A naming trap ⚠️

The element's name is still **`screen-resource-allocation`** (the old one), while the directory and the
component names say **`commerce`** (the new one). Both sets of files exist in the game:

| | path | status |
|---|---|---|
| old | `base-standard/ui/resource-allocation/screen-resource-allocation.js` | loaded, but **loses** |
| old model | `base-standard/ui/resource-allocation/model-resource-allocation.js` | as above |
| **new** | `base-standard/ui-next/screens/commerce/` | **this is what the player sees** ✅ |

Both are in `base-standard.modinfo` under `<UIScripts>`; the new one wins on
`Controls.define` priority (1 vs 0) — the mechanism is described in [25](25-ui-next-solidjs.md).
**Modifying the old files has no effect on the screen.**

Similarly misleading neighbors (these are **different** screens, not Commerce):
`ui/city-trade/` (trade with a specific city), `ui/trade-route-chooser/`,
`ui/lenses/layer/trade-layer.js`, `ui/interface-modes/interface-mode-resource-allocation.js`.

## The files ✅

```
base-standard/ui-next/screens/commerce/
  commerce-screen.tsx                 the skeleton: ScreenFrame + 4 tabs
  commerce-screen-model.ts            3334 lines — ALL the logic and data of every tab
  commerce-screen-base-tab-content.tsx the shared tab frame (title, description, header bar)
  commerce-screen-resources-tab.tsx   the "Resources" tab (1640 lines, drag&drop)
  commerce-screen-trade-tab.tsx       the "Trade" tab (routes)
  commerce-screen-empire-tab.tsx      the "Empire" tab
  commerce-screen-treasure-tab.tsx    the "Treasure" tab (AGE_EXPLORATION only)
  trade-route-card.tsx                a single route's card
  treasure-convoy-card.tsx            a treasure convoy's card
  treasure-convoy-progress-bar.tsx
  factory-type-display.tsx            the factory resource picker
  commerce-criteria-display.tsx       the "condition met / not met" row
  commerce-screen.css / .scss.js      styles
```

Texts: `base-standard/text/en_us/CommerceScreenText.xml` — 109 entries, all
prefixed `LOC_COMMERCE_*`. The resources tab's older keys are `LOC_UI_RESOURCE_*`.

## The screen's structure ✅

```
<screen-resource-allocation>            (a custom element, class .fullscreen)
└ CommerceScreenContext.Provider        model = createCommerceScreenModel()
  └ ScreenFrame  name="Commerce-Screen"  title = LOC_COMMERCE_SCREEN_TITLE(civilization name)
    └ Tab                               nextHotkey="nav-next" previousHotkey="nav-previous"
      ├ Tab.Item "Resources"  LOC_UI_RESOURCE_ALLOCATION_TITLE  → CommerceResourcesContainer
      ├ Tab.Item "Trade"      LOC_COMMERCE_TRADE_ROUTE_TAB      → TradeRoutesContainer
      ├ Tab.Item "Empire"     LOC_RESOURCECLASS_EMPIRE_NAME     → EmpireResourceContainer
      └ Tab.Item "Treasure"   LOC_RESOURCECLASS_TREASURE_NAME   → TreasureResourceContainer
                                        ⚠️ only when Game.age == Database.makeHash("AGE_EXPLORATION")
```

Every tab is wrapped in `CommerceScreenBaseTabContent`
(title + description + a `headerBar` for sorting/searching + a dark background + a `hud_section-line` frame).

## Override points (`ComponentRegistry.register`) ✅

These are the only places a mod can enter without replacing the whole screen:

| name | what it is | scope of changes |
|---|---|---|
| `CommerceScreen` | the whole screen | everything, but you have to recreate the rest |
| `CommerceScreenBaseTabContent` | the shared frame of **all** the tabs | adding something to every tab at once — the cheapest hook |
| `CommerceResourcesContainer` | the whole "Resources" tab | |
| `TradeRouteCard` | one route's card | |
| `TreasureConvoyCard`, `TreasureConvoyProgressBar` | treasure convoys | |
| `FactoryTypeDisplay` | the factory resource picker | |
| `CommerceCriteriaDisplay` | one row of a route's condition | |

❗ **Not registered** (i.e. they cannot be overridden by name):
`TradeRoutesContainer`, `EmpireResourceContainer`, `TreasureResourceContainer`
and all the internal components of the resources tab (`CityResourceContainer`,
`AvailableResourcesSection`, `SettlementName`, `DraggableResource`, `ResourceSlot`,
`WarningBanner`…). To change them you have to override their parent.

⚠️ **`CommerceScreenModel` from `ModelRegistry` is a dead end.** The model is
registered (`ModelRegistry.register("CommerceScreenModel", SharedInstance,
createCommerceScreenModel)`), but `commerce-screen.tsx` **calls the factory directly**
(`const model = createCommerceScreenModel()`), so overriding the registration changes
nothing. To replace the data you have to override `CommerceScreen` and pass your own model
to `CommerceScreenContext.Provider`.

Accessing the model from inside your own component:
`useCommerceScreenContext()` from `/base-standard/ui-next/screens/commerce/commerce-screen-model.js`
(it throws outside the screen's tree).

## The model — what is inside ✅

`createCommerceScreenModel()` returns a `createMutable(...)` with the fields collected in
`CommerceScreenContextModel`. The most important ones:

**Data (`model.data`, type `CommerceScreenData`):**
`resourceTabData`, `tradeRouteTabData`, `empireTabData`, `treasureTabData`, `ornatePanelData`.

**Selection/focus (Solid signals):** `selectedResource`, `prevSelectedResource`,
`focusedResource`, `selectedSettlementId`, `focusedSettlementId`, `selectedTradeRouteId`,
`selectedEmpireResource`, `selectedTreasureConvoyId`, `ghostResourceFocused`.

**Actions:** `clickAvailableResource`, `slotSelectedResource(cityID, targetResourceValue?)`,
`clickSlottedResource`, `unslotSelectedResource`, `deselectSelectedResource`,
`clearAllResources(cityID?)`, `clearFactoryResources(cityID)`, `clickCityName`,
`clickTreasureFleet`, `clickUnimprovedTreasure(location)`, `clickCloseButton`.

**Rules:** `canSelectResource`, `canDropResourceOnTarget`,
`canAssignSelectedResourceToSettlement`, `resourceIsConnectedToTradeNetwork`,
`cityIsConnectedToTradeNetwork`, `settlementHasSlottedResources`, `hasUnassignedResources`.

**Sorting/filtering:** `selectedSettlementSortType` (the `ResourceSettlementSortType` enum:
settlement type, name, free slots, all slots, warehouses, near/distant lands, rail,
factory, and each of the 7 yields), `selectedTradeRouteSorting` (the `TradeRouteSortType` enum),
`selectedResourceFilter`, `tradeRouteSearch(text)` (fuzzy, `FullTextSearch`).

**Refreshing:** `onMount` hooks up `createEngineEvent("ResourceCapChanged" |
"ResourceAssigned" | "ResourceUnassigned")` and batches them with the old `UpdateGate`
into a single recomputation.

## The "Trade" tab — how routes are built ✅

`populateTradeRoutes()` calls
`Players.get(GameContext.localPlayerID).Trade.projectPossibleTradeRoutes(
INCLUDE_FAILED + EXTENDED_STATUS)` and splits the result into **three sections**
(`CollapsibleContainer`):

| section | condition (`route.status`) | `TradeRouteAvailabiltyType` |
|---|---|---|
| active `LOC_COMMERCE_ACTIVE_TRADE_ROUTES_TITLE` | `ALREADY_EXISTS` | `Established` |
| available `LOC_COMMERCE_AVAILABLE_TRADE_ROUTES_TITLE` | `SUCCESS` | `Available` |
| unavailable `LOC_COMMERCE_UNAVAILABLE_TRADE_ROUTES_TITLE` (collapsed by default) | the rest | `Unavailable` |

Routes with the `NO_RESOURCES` status are **skipped entirely**.

One `TradeRouteData` carries: the city's name and icon (`res_capital` / `Yield_Towns` /
`Yield_Cities`), `isCityState`, `domainString` ("delivered to X by land/sea"),
`incomingResources` (sorted by resource class: City → Bonus → Empire → Treasure →
Factory, then alphabetically), `yieldElement` (ready JSX with `LOC_TRADE_LENS_YIELD_EXPORT`),
`relationshipChange` (`getPotentialRelationshipGainFromTradeRouteWith`), `leaderId`,
`cityID`, `fullText` (material for the search) and `statuses`.

`statuses` is always **three** `CommerceCriteriaStatusEntry` entries
(capacity / range / peace), each with `isNegative` and a tooltip key; they are shown
only on an unavailable route's card. ⚠️ A Firaxis comment in the code admits that for
the range criterion it is not possible from that place to check whether it is satisfied
(`appliesToCurrentCiv: true` hardcoded).

**The cards' layout** is computed by hand in `TradeRoutesContainer`: a `ResizeObserver` +
`checkForWrap()` measures the container, divides the width by `DEFAULT_CARD_WIDTH`
(`Layout.pixelsToScreenPixels(512)`) and sets each card's width in px.
Until the first measurement the cards have `opacity-0`. ⚠️ Overriding `TradeRouteCard`
with a card of your own with different dimensions will break that logic — you have to respect
`props.style.width` and the `.trade-route-card` class (`querySelectorAll` counts cards by it).

Route sorting: number of resources / leader name / relationship with the leader / settlement name /
settlement type. The search bar (`SearchBar`) is shown **only** on
`ViewExperience() === UIViewExperience.Desktop` and when a gamepad is not active.

## The "Resources" tab — structure ✅

The biggest and most complex one. Two columns of data in the model:

- `availableResourceSectionData[]` — unassigned resources, split into subsections by
  class (City / Bonus / Factory…), with an `isConnectedToTradeNetwork` flag;
- `slottedResourceSectionData[]` — settlements (`CommerceCityResourceData`): the settlement's name and icon,
  `baseYields` + `yieldDeltas` (a preview of the yield change!), `slottedResources`,
  `availableSlots`, `factoryResourceData`;
- `unslottedBonuses` — the bonus from unassigned resources
  (`getUnassignedResourceYieldBonus`).

Moving resources is full **drag & drop** (`createTypedDragAndDrop` from
`core/ui-next`), with separate gamepad handling (`GamepadTrayItemProvider`, the
`ResourceTabInteractionTypeFlag` / `ResourceTabInteractionCombo` enums bit-encoding
the "selected resource + hovered settlement" combinations).

`CommerceCityNameData` carries ready-made counters, useful for summaries of your own:
`warehouseCount`, `tradeConnectionCount`, `waterCount`, `hasRail`, `isTown`,
`townFocusName/Icon`, `settlementDistanceTypeName`.

## The "Empire" and "Treasure" tabs ✅

- **Empire**: `EmpireResourceData[]` — the resource, how many, which cities and which
  leaders it comes from (`resourceOriginData`, `tooltips` per player).
- **Treasure**: the Exploration age only. `TreasureFleetData` — the city, the list of treasure
  resources, progress (`progress`/`progressGoal`), `getTurnsUntilTreasureGenerated()`,
  `isDistantLand`, plus `statuses` in the same format as routes.

## ⭐ A working pattern from the Workshop: the **Resource+** mod ✅

`steamapps\workshop\content\1295660\3756000777\ui\commerce\resource-plus.js` (1251 lines,
id `brads-assign-all-resources`, by Brad). **The only known mod that modifies this
screen** — and proof that all the theory above works in practice. It adds: an
"Assign All / Reassign All" button, a per-settlement yield priority choice, and locks that keep
resources from being moved.

The whole mod is **a single JS file + a `.modinfo`** — no `text/`, no CSS as a file,
no dependencies. `AffectsSavedGames = 0`, `LoadOrder` 1100.

**The technique (described at greater length in [25](25-ui-next-solidjs.md)):**

1. it imports `CommerceResourcesContainer` from the game and takes `.factory` as `originalFactory`;
2. it registers its own wrapper under the same name with `overridePriority: 1100`;
3. the wrapper calls `useCommerceScreenContext()` (which works — we are in the Provider's tree),
   injects **raw DOM** (`document.createElement`) in `onMount`, and at the end returns
   `originalFactory(props)`;
4. a `<style id="brad-assign-all-style">` goes into `document.head`;
5. a `MutationObserver` on `document.body` calls `reconcileUI()` after every rebuild
   of the tree by Solid;
6. `onCleanup` removes **every** injected element individually.

❗ **`overridePriority: 1100` is already taken** on `CommerceResourcesContainer`.
Our mod has to give more if it wants to be "on the outside", and it **must delegate** to
`originalFactory` — otherwise Resource+ will disappear (exactly the same bug as the conflict
with City Hall, quirk #30).

**It makes changes in the game through official player operations**, not by poking at the model:

```js
Game.PlayerOperations.canStart(GameContext.localPlayerID,
    PlayerOperationTypes.ASSIGN_RESOURCE,
    { Location: GameplayMap.getLocationFromIndex(resourceValue), City: cityID.id },
    false).Success
```

⚠️ That is already a **mechanics change**, not just UI — if our mod is to stay purely UI,
we do not reproduce that step.

## DOM hooks in the "Resources" tab ✅

The `data-name` attributes and classes used by Resource+, confirmed in the game's sources
(`name=` on an `Activatable`/`SpatialSlot` renders as `data-name`):

| selector | what it is |
|---|---|
| `[data-name="available-resources-container"]` | the left column: unassigned resources |
| `[data-name="commerce-unassigned-resources"]` | the list inside it |
| `[data-name="slotted-resource-container"]` | the right column: settlements |
| `[data-name="commerce-screen-base-tab-content"]` | the shared content of **every** tab |
| `[data-name$="-city-resource-activatable"]` | one settlement's card |
| `[data-name^="city-resource-container-"]` | the inside of a settlement's card (with the settlement's name!) |
| `.text-secondary.w-full.mb-2` | a section (connected / disconnected) |
| `.flex.flex-row.flex-wrap.relative.w-full.justify-between` | a settlement card's header |
| `.size-19` | a single resource slot in a settlement |
| `[data-name="Commerce-Screen-Trade-Tab"]`, `…-Empire-Tab`, `…-Treasure-Tab` | the tabs' roots |
| `[data-name$="-Trade-Route-Card"]`, `.trade-route-card` | a route's card |

⚠️ Class-based selectors (`.text-secondary.w-full.mb-2`) are brittle — those are ordinary
layout classes, not identifiers. `data-name` is safer.

Resource+ does the **model → DOM mapping** by index, not by id:

```js
const sections = container.querySelectorAll('.text-secondary.w-full.mb-2');
model.data.resourceTabData.slottedResourceSectionData.forEach((section, i) => {
    const cards = sections[i].querySelectorAll('[data-name$="-city-resource-activatable"]');
    section.cityResources.forEach((settlement, j) => { /* cards[j] ↔ settlement */ });
});
```

⚠️ It assumes the DOM order = the model's order. With settlement sorting/filtering
that assumption can break — a better key is the settlement's name from
`data-name="city-resource-container-<name>"`.

## Another neighbor: "Trade Chooser Improvements" (Slothoth) ✅

Workshop 3570879406, id `resource-fixes-deadbeef`, in a directory named
`Resource-Screen-Improvements` — **misleading, this is NOT the Commerce screen.** It concerns
`ui/trade-route-chooser/` (the route picker on a trade unit), i.e. the
**old** framework, and it uses `<ImportFiles>` to replace whole game files
(`trade-route-chooser.js`, `trade-routes-model.js`) — the most invasive technique,
guaranteed to conflict with any other mod touching those files. It also adds its own
icons (`UpdateIcons` + `.dds`) and texts through `.sql`.

## Un-assigning a resource ✅

The game does it with one player operation (`commerce-screen-model.ts`, `unassignResource`):

```js
Game.PlayerOperations.sendRequest(GameContext.localPlayerID,
    PlayerOperationTypes.ASSIGN_RESOURCE, {
        Location: GameplayMap.getLocationFromIndex(resourceValue),
        City: cityID.id,
        Action: PlayerOperationParameters.Deactivate,   // ← this is what distinguishes it from assigning
    });
```

So it is **the same operation as assigning**, only with `Action: Deactivate`. Assigning is
the same object without the `Action` field. Swapping places is a separate `SWAP_RESOURCES` operation
with `{ Location, Location2 }`.

After sending, nothing has to be refreshed by hand — the model listens for `ResourceUnassigned`
and recomputes through an `UpdateGate`.

⚠️ The model's `unslotSelectedResource()` operates on the **currently selected** resource, so
to un-assign a specific one you would have to select it first. In a bulk operation that means
as many selection-signal changes as there are resources — better to call the operation directly.

**Identifying a resource's kind:** `ResourceSlotData.resourceType` is
`ResourceDefinition.ResourceType`, i.e. `"RESOURCE_CAMELS"` and so on. ⚠️ Not to be confused with
`ResourceProps.resourceType`, which holds the resource's **class** as a localization key
(`LOC_RESOURCECLASS_CITY_NAME`) — two different fields with the same name.

## Assigning a resource — both routes end in one place ✅

**2026-08-10.** The player can assign a resource in two ways, but in the code they converge
on **a single model call**:

| route | place in `commerce-screen-resources-tab.tsx` | call |
|---|---|---|
| clicking a settlement | the settlement card's `Activatable`, `onActivate` | `model.slotSelectedResource(cityID)` |
| clicking an empty slot | the "ghost" `Activatable`, `onActivate` | `model.slotSelectedResource(cityID)` |
| dragging | `<DragAndDrop onDragDrop={…}>` in `CommerceResourcesContainerComponent` | `model.slotSelectedResource(cityID)` |

That is why a mod that wants to change the behavior of **every** way of assigning
wraps one method instead of three components. The model is a `createMutable`, so the
property can simply be replaced and restored during cleanup:

```js
const original = model.slotSelectedResource;
model.slotSelectedResource = (cityID, targetResourceValue) => {
    const assigned = model.selectedResource().resourceValue;   // ⚠️ before the call!
    original.call(model, cityID, targetResourceValue);
    …
};
```

⚠️ Read the selection **before** calling the original — `handleSlotSelectedResource`
ends with `handleDeselectSelectedResource()`, so afterwards it is already empty.

⚠️ A call with a second argument (`targetResourceValue`) is a **swap of two resources**,
not an insertion into a free slot. If your logic is only about
assignment, skip such calls.

The confirming events from the engine's side: **`ResourceAssigned`** and
**`ResourceUnassigned`** (`engine.on` / `engine.off`) — the screen's model listens for them the
same way, additionally with `ResourceCapChanged`.

## Resources that provide slots themselves — `BonusResourceSlots` ✅

**2026-08-10.** Camels give a settlement **two extra resource slots**, and this is not
hardcoded but held in the data:

```xml
<!-- base-standard/data/resources.xml -->
<Row ResourceType="RESOURCE_CAMELS" … ResourceClassType="RESOURCECLASS_CITY"
     Weight="10" BonusResourceSlots="2" UnlocksCiv="true"/>
```

`BonusResourceSlots` is a **schema column** (`Base\Assets\schema\gameplay\01_GameplaySchema.sql`,
`INTEGER NOT NULL DEFAULT 0`). As of 2026-08-10 camels are the only resource with a non-zero
value in the entire game including DLC — but do not assume that permanently, read the column:

```js
GameInfo.Resources.forEach((r) => { if (r.BonusResourceSlots > 0) … });
```

⚠️ **The consequence when un-assigning:** taking such a resource out **reduces
the settlement's capacity**, so the same number of other resources has to leave with it, otherwise the settlement
would hold more than it has slots for. With one camel that is 2 extra resources, with two
camels 4. Order matters: **the companions first, then the slot-granting resource.**

⚠️ Do not pick resources that **themselves** grant slots as companions — that would reduce
capacity again and you would get a cascade.

❗ **`sendRequest` only QUEUES the operation.** A `canStart` query about the next resource
in the same tick still sees the old state. Sending the companions and the camel at once
ends like this: the companions leave, the camel **stays** (refused), and the next
click charges companions again for a removal that no longer needs them.

The correct way is **sequential, and asking instead of computing**:

1. try `canStart` on the actual resource — if it passes, send it and stop;
2. if not: release **one** companion (from the end of the list), wait for confirmation,
   go back to 1;
3. stop once the candidate pool runs out (limit it by the number of slots lost).

This way the settlement never loses more than the engine actually requires — and the requirement
does not have to be known in advance.

For confirmation of the change wait for the engine event **`ResourceUnassigned`**
(`engine.on` / `engine.off`), with a timeout in case an operation quietly
gets lost.

`GameInfo.Resources` is read iteratively (`forEach`), not with a query — the screen's model does
the same in `indexResourceTypes()`.

## Open questions ❓

- How `ScreenFrame` / `ornatePanelData` reacts to a change in a tab's content height.
- Whether a **fifth tab** of your own can be injected into `<Tab>` without overriding the whole
  `CommerceScreen` (`Tab.Item`s are `CommerceScreen`'s children, so probably not).
- Whether a `MutationObserver` on `document.body` (the Resource+ technique) costs noticeably
  on large maps — whether it can be narrowed to the tab's container.

## Hooks for changing the "Resources" tab's appearance ✅

**2026-08-10.** The header bar's structure (the `headerBar` passed to
`CommerceScreenBaseTabContent` in `commerce-screen-resources-tab.tsx`):

```
div.flex.flex-row.w-full.items-center                    ← the bar's root
├ div[style="width: 28%"]                                ← the "UNASSIGNED" column
│  ├ div  (the LOC_COMMERCE_AVAILABLE_RESOURCES_TITLE title)
│  └ div.flex.flex-row                                   ← the row of yield totals
│     └ div.ml-2.flex.flex-row × N                       ← one total: an icon + "+54"
├ div  (the LOC_COMMERCE_SETTLEMENTS_TITLE title)
└ [data-name="filter-and-sort"]                          ← ⭐ a stable handle
   ├ the FILTER label + <Dropdown class="… min-h-14">
   └ the SORT label   + <Dropdown class="… min-h-14">
```

`[data-name="filter-and-sort"]` comes from `HSlot name="filter-and-sort"` and is
**the only named element in that bar** — it is the easiest starting point for reaching the
rest structurally (`.parentElement` is the bar's root, its first child is the unassigned
column, its last child is the row of totals).

The dropdowns' height comes from the `min-h-14` class = **3.1111rem**
(`core/ui/themes/default/default.css`). For comparison `min-h-10` = 2.2222rem,
`min-h-8` = 1.7777rem.

**Working out which yield a given total belongs to:** `YieldBonus` carries only
`iconSrc` and `bonusAmount`, with no type. Do not guess from the order — build a map with the same
function the model used:

```js
import { Icon } from '/core/ui/utilities/utilities-image.js';
const byIcon = new Map();
GameInfo.Yields.forEach((y) => byIcon.set(`url(${Icon.getYieldIcon(y.YieldType)})`, y.YieldType));
```

**The instruction line above the panel** (`LOC_COMMERCE_RESOURCE_ALLOCATION_DESCRIPTION`)
is rendered by `CommerceScreenBaseTabContent` as a `div.text-base.w-full.text-center.my-4`, a
**sibling** of the content frame — you cannot reach it from inside the tab, so a
CSS selector is what is left. The scope is best given through the `screen-resource-allocation` element, i.e. the
root of the whole screen (the custom element's name from `defineLegacyComponent`).

⚠️ That component is shared by **all four tabs**. If a change is meant to apply
to only one, do not fiddle with the selector — **attach the stylesheet in the tab component's `onMount`
and remove it in `onCleanup`**. The scope then comes from the lifecycle,
not from CSS.

## Badges that look like Drongo's Top Panel ✅

The **Drongo's Top Panel** mod (Workshop 3734234006) does not draw its own badges — it adds
a background to elements the game already renders, through a stylesheet in
`ui/diplo-ribbon/css-constants.js`. The recipe (to copy when something should look "like
the top bar"):

```css
height: 1.7777777778rem;            /* = min-h-8 */
border-radius: 0.4444444444rem;
padding-right: 0.5555555556rem;
margin: 0.1666666667rem;
background-clip: padding-box;
color: #FFFFFF;
```

Background colors per yield: gold `rgba(255,235,75,.3)`, happiness `rgba(253,175,50,.3)`,
science `rgba(50,151,255,.3)`, culture `rgba(197,75,255,.3)`, production
`rgba(204,118,52,.34)`, influence/diplomacy `rgba(88,192,231,.3)`, the rest
`rgba(228,228,228,.3)`. On hover, the same color with alpha `.5`.

## Reading a settlement's current yields (including happiness) ✅

**2026-08-10.** `CommerceCityResourceData.yieldDeltas` is an array of `YieldDeltaProps`
with the fields **`yieldIconSrc`**, **`yieldTotal`** and `yieldDelta`. The yield's type is not there —
it is extracted from the icon's URL:

```js
const type = String(entry.yieldIconSrc).match(/YIELD_[A-Z_]+/)?.[0] ?? null;
const total = Number(entry.yieldTotal) || 0;
```

`yieldTotal` is the value shown on the settlement's card (e.g. `-10` happiness), so it is
the simplest source of "which settlement is unhappy". ⚠️ `yieldDelta` is
a preview of the **change** on hover, not the state — do not confuse them.

**Refreshing:** after every assignment operation the model recomputes itself
(the `ResourceAssigned` event + an `UpdateGate`), so an algorithm running in a loop should
read `yieldDeltas` **anew on every pass**. That is enough to distribute resources
evenly: you always pick the currently worst settlement, and it stops being the worst
as soon as it has had enough — with no planning ahead at all.

## Game icons useful on this screen ✅

**2026-08-10.** BLP paths straight from `base-standard/data/icons/` — they can be used as
`url(blp:…)` in a style, exactly as the game does in `commerce-screen-empire-tab.tsx`:

| what | BLP | appearance |
|---|---|---|
| resource class: city | `blp:restype_city_v2` | a blue building |
| resource class: bonus | `blp:restype_bonus_v2` | a green leaf with a plus |
| resource class: empire | `blp:restype_empire_v2` | an orange hexagon |
| resource class: treasure | `blp:restype_treasure_v3` | a golden chest |
| treasure fleet | `blp:restype_treasure_v2` | |
| resources (in general) | `blp:radial_resources` | a green leaf |
| trade routes | `blp:Action_Trade` | two arrows |

The second way, used by the **Trade Chooser Improvements** mod (Slothoth): the
`data-icon-id` + `data-icon-context` attributes on an element, or `UI.getIconCSS(id, context)`.
Contexts confirmed "live" by that mod:

```js
UI.getIconCSS('RESOURCECLASS_EMPIRE', 'RESOURCECLASS')
UI.getIconCSS('RADIAL_RESOURCES', 'DEFAULT')
UI.getIconCSS('YIELD_TRADES', 'YIELD')          // blp:Action_Trade
UI.getIconCSS('TRADE_ROUTE_LAND', 'TRADE')
UI.getIconCSS('CITY_YIELDS_HI', 'DEFAULT')
UI.getIconCSS('UNKNOWN_LEADER', 'LEADER')
```

⚠️ The same `ID` sometimes exists in several contexts with different paths (e.g. `RESOURCECLASS_*`
occurs separately for the fog of war, `Context=FOW`). Without giving a context you can get
the wrong image.

## The tab bar — replacing labels with icons ✅

`[data-name="TabList"]` contains `[data-name="TabListItem"]` **in the declaration order**
from `commerce-screen.tsx` (Resources, Routes, Empire, Treasure — the last only in the Exploration
age). There is nothing on a tab element saying which one it is — **its position is the only
identity**.

❗ **CORRECTION: `font-size: 0` on the tabs does NOT work.** The labels have the
`font-fit-shrink` class, i.e. Coherent's `coh-font-fit-mode: shrink` — the engine picks
that text's size itself and ignores the declared one. You have to **remove the text nodes**:

```js
for (const node of Array.from(item.childNodes)) {
    if (node.nodeType === 3) { label += node.nodeValue; item.removeChild(node); }
}
```

Remove **text nodes only** — the icon you add is an element and should stay.
The tab titles are static, so Solid does not recreate them, but the observer should
repeat this anyway just in case. Keep the removed text for the tooltip.

⚠️ The tab bar outlives an individual tab. If you hook in from one tab's component,
**do not clean the icons up in its `onCleanup`** — they would disappear on switching to another
tab. Attach the observer to the bar itself: when the screen closes, the element is gone
and the observer stops receiving anything, with no cleanup needed.


## The "un-assign everything" buttons — where they are ✅

| what | where | call |
|---|---|---|
| the whole empire | at the very bottom of the settlements column, a `div.self-end` inside `[data-name="slotted-resource-container"]` | `model.clearAllResources()` |
| one settlement | on the right of the settlement's card, `.fxs-image-button` | `model.clearAllResources(cityID)` |

Both are a `ReturnResourceButton`, i.e. an `ImageButton` with the artwork
`blp:resource_return_button_default.png` (hover: `..._hover.png`) and the class
**`fxs-image-button`** — that is the most convenient handle, because there is no `data-name` there.

The empire one is additionally wrapped in a `ConfirmationDialog`; the settlement one is not.
When moving either of them elsewhere, **hide the original and expose your own button**
calling the same method — moving a subtree managed by Solid would fight with
whatever recreates it.

⚠️ Repeat the hiding on every pass of the observer: the settlement's card is rebuilt
and restores the original.

A ready-made label with the settlement's name:
`Locale.compose('LOC_COMMERCE_UNASSIGN_RESOURCES', Locale.compose(city.name))`.

## Tabs and the age — there is no fifth category ✅

**2026-08-10.** `commerce-screen.tsx` has **exactly four** `Tab.Item`s, and the only
condition is the age on "Treasure":

| age | visible tabs |
|---|---|
| Antiquity | Resources, Trade Routes, Empire |
| Exploration | Resources, Trade Routes, Empire, **Treasure** |
| Modern | Resources, Trade Routes, Empire |

❗ **Factory resources do NOT have a tab of their own.** They appear as a **subsection
in the unassigned column** (an `AvailableResourceSubSection` with `type =
"RESOURCECLASS_FACTORY"`, title `LOC_RESOURCECLASS_FACTORY_NAME`) and as a dropdown
list on a settlement with a factory (`FactoryTypeDisplay`). No DLC overrides this screen.

⚠️ The consequence for mapping tab icons by index: 0/1/2 are always Resources/Routes/Empire,
and 3 exists only in the Exploration age — so the indices do not shift between ages.

## Tab labels are sometimes written in CAPITALS in the localization ⚠️

`LOC_COMMERCE_TRADE_ROUTE_TAB` is literally `TRADE ROUTES` in the data, not "Trade Routes".
This is not a CSS `text-transform` — the text itself is uppercase. If you move such
a label elsewhere (e.g. into a tooltip), you have to soften it yourself:

```js
if (text === text.toUpperCase() && text !== text.toLowerCase()) {
    const lower = Locale.toLower(text);            // ⚠️ Locale.toLower, not toLowerCase
    text = lower.charAt(0).toUpperCase() + lower.slice(1);
}
```

The `text !== text.toLowerCase()` condition is there so as not to touch scripts without letter
case (Chinese, Japanese, Korean), where both versions are the same string.

## Resources that boost unit production ✅

`*_ADJUST_UNIT_PRODUCTION_*` effects attached to a resource are qualified by **exactly
one** of three arguments — and that is the only way to tell military apart from the rest:

| argument | meaning | examples |
|---|---|---|
| `Domain` = `DOMAIN_LAND` / `DOMAIN_SEA` | **combat** units | Cotton, Hardwood, Citrus |
| `UnitClass` = `UNIT_CLASS_NON_COMBAT` | settlers etc. | Hardwood (Modern) |
| `UnitTag` = `UNIT_CLASS_RELIGIOUS` | missionaries | Incense |

Resources with such modifiers as of 2026-08-10: `RESOURCE_CITRUS`, `RESOURCE_COTTON`,
`RESOURCE_HARDWOOD`, `RESOURCE_INCENSE`, `RESOURCE_SALT`, `RESOURCE_TRUFFLES`.

⚠️ Do not hardcode that list — read `ModifierMetadatas` (`FieldName="ResourceType"`)
→ `ModifierArguments` → `Modifiers` → `DynamicModifiers.EffectType`, just as when
reading yield effects. The same resource is sometimes configured differently in different ages
(Hardwood: naval in Antiquity and Exploration, civilian in the Modern age), so key the cache
by `Game.age`.

## Adding a tab of your own to the screen ✅

**2026-08-10.** The `Tab.Item`s are **children of `CommerceScreen`**, written out directly in its JSX.
The framework offers no way to add another one from outside — there is no registry
of tabs and no API on `TabContext`. The only route is to **register your own `CommerceScreen`**
with a higher `overridePriority` and recreate the whole tree inside it.

So it is the one place in the mod where you **replace** rather than wrap. Practical conclusions:

- rewrite the game's component **line for line**, adding only your own — after a patch you can
  then diff against the original and see what changed;
- keep `styles: [screenStyle]` at registration (importing
  `commerce-screen.scss.js`), otherwise the screen loses its own stylesheet;
- write in Solid's compiled form (`createComponent`, getters on props) — the mod has
  no build, so JSX is out. **The getters are mandatory**: a plain value reads the model once
  and never refreshes;
- `model.onTabChanged` is a `switch` on `tab.name` **with no `default`**, so a new tab
  name breaks nothing — it simply does not trigger a reset.

Tab icons mapped by index then have to **depend on the age**: the fourth position is
Treasure in Exploration, but e.g. your own tab in the Modern age. A fixed array will put the treasure
chest on somebody else's tab.

## A resource audit: what the data says and what the code assumes ✅

**2026-08-10.** I scanned all **111 resource occurrences** (all resources ×
all ages) and compared the game's data with the assignment algorithm's assumptions. The audit script:
`mod-projects/better-commerce-screen-ui` → the session history; the results below are permanent.

### 1. A modifier is not always in `ModifierMetadatas` ❗

The resource ↔ modifier link is sometimes written in **two ways**:

```xml
<!-- indirectly, through the metadata table -->
<ModifierMetadatas>
    <Row ModifierId="MOD_TRUFFLES_UNIT_PRODUCTION" FieldName="ResourceType" String="RESOURCE_TRUFFLES"/>
</ModifierMetadatas>

<!-- directly, through the modifier's own argument -->
<Modifier id="MOD_NICKEL_CITY_SCIENCE" effect="EFFECT_CITY_ADJUST_YIELD_PER_RESOURCE">
    <Argument name="ResourceType">RESOURCE_NICKEL</Argument>
</Modifier>
```

⚠️ Anyone reading only `ModifierMetadatas` **will see nothing** for: **Nickel** (Modern,
2 modifiers), one **Gypsum** modifier (Antiquity) and `GOLD_DISTANT_LANDS` /
`SILVER_DISTANT_LANDS`. Nickel then came out as a resource with no yields at all.

**Read both routes** and merge them into a single index.

### 2. Modifiers have conditions and mostly they mean "cities only" ❗

The distribution of `SubjectRequirements` on resource modifiers:

| condition | occurrences |
|---|---|
| `REQUIREMENT_CITY_HAS_BUILD_QUEUE` | 29 |
| `REQUIREMENT_CITY_IS_DISTANT_LANDS` | 22 |
| `REQUIREMENT_CITY_HAS_BUILDING` | 12 |
| `REQUIREMENT_CITY_IS_TOWN` | 10 |
| `REQUIREMENT_UNIT_TAG_MATCHES` | 9 |
| `REQUIREMENT_CITY_IS_CITY` | 6 |
| `REQUIREMENT_CITY_IS_CAPITAL` | 3 |
| `REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE` | 3 |

❗ **`CITY_HAS_BUILD_QUEUE` is the game's way of writing "cities only"** — a town
has no production queue. The big bonuses are gated this way: Jade +10 gold, Silk
+10 culture, Lapis Lazuli +4 production and +10 gold, Cloves +10 gold, Incense
+10 science. An algorithm that does not check them will assign those resources to a town, where
they give **nothing**.

Other resources split straight by settlement type — and there, taking the smaller of the two
values (which is what code grouping variants by `Math.min` does) understates both:

| resource | city | town |
|---|---|---|
| Tin (Antiquity) | +2 production | +4 production |
| Wild game (Antiquity) | +2 food | +4 food |
| Wild game (Exploration) | +3 food | +6 food |
| Cowrie | +4/5/6 gold | +2/3/4 science |

**The join path** (all in `GameInfo`):
`Modifiers.SubjectRequirementSetId` → `RequirementSetRequirements` → `Requirements`
(`RequirementType`, `Inverse`) → `RequirementArguments`.
`RequirementSets.RequirementSetType` says ALL or ANY (by default `REQUIREMENTSET_TEST_ALL`).

⚠️ Treat a condition you cannot evaluate **as satisfied**. Being too eager
spoils the scoring a little; being too strict cuts the resource out of consideration entirely and nobody
notices.

### 3. Resource+'s hand-written condition table is INVERTED ❗

The Resource+ mod keeps a list of resource names per age instead of reading the conditions. That list
**contradicts the data**:

- Antiquity, gypsum/kaolin/pearls — the data: the bonus requires `CITY_IS_CAPITAL`; the code: strength 1 **when
  the settlement is NOT the capital**;
- Exploration, spices/sugar/tea — the data: `CITY_HAS_BUILD_QUEUE + CITY_IS_DISTANT_LANDS`;
  the code: `!isDistantLands && !isTown`;
- cocoa — the data: `CITY_IS_TOWN + CITY_IS_DISTANT_LANDS`; the code: `!isDistantLands && isTown`.

On top of that, **31 resources with conditions in the data are not in that table at all**.

### 4. Other findings ✅

- **Extra slots are granted only by camels**, and only in Antiquity and Exploration —
  in the Modern age no resource has `BonusResourceSlots > 0`.
- **Warehouse scaling**: clay, crabs, turtles (Antiquity and Exploration), crabs alone
  in the Modern age — recognizable by `Tag = WAREHOUSE`.
- **The unit production bonus** has 8 occurrences: Hardwood (3 ages), Salt,
  Incense, Truffles, Citrus, Cotton (Modern). ⚠️ Each of them also has **+1 base
  yield** (e.g. Salt +1 food, Cotton +1 gold) — by pushing them to the back of the queue,
  you are deliberately giving up that +1.
- `REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE` (Wine, Furs) depends on the game's state, not a settlement's —
  it cannot be sensibly evaluated when planning assignments.

## `ResourceSlotData.yieldTypes` does NOT come from the resource's yields ❗✅

**2026-08-10.** The model builds a resource's `yieldTypes` field from **`GameInfo.TypeTags`**, not
from `Resource_YieldChanges`:

```js
const yieldTagTypes = new Map([
    ['FOOD','YIELD_FOOD'], ['PRODUCTION','YIELD_PRODUCTION'], ['GOLD','YIELD_GOLD'],
    ['SCIENCE','YIELD_SCIENCE'], ['CULTURE','YIELD_CULTURE'], ['HAPPINESS','YIELD_HAPPINESS'],
]);
GameInfo.TypeTags.forEach((t) => { /* Type == ResourceType, Tag == FOOD/PRODUCTION/... */ });
```

So a resource tagged `PRODUCTION` "concerns production" regardless of whether it has a flat
production yield. ⚠️ Note that **influence (`YIELD_DIPLOMACY`) is not on that list** —
that is not an oversight, it is the state of the map in the game.

This cost a whole round: while reconstructing the model's data outside the screen I left
`yieldTypes: []`, and the planning code has a fallback

```js
if (resource.yieldTypes?.length) return resource.yieldTypes;
// otherwise Resource_YieldChanges
```

so **nothing broke — the result simply came out different**. The same algorithm gave a different
arrangement of resources with the screen closed than with it open, and the happiness rescue never
kicked in.

**The rule:** when reconstructing the model's structures outside its context, build every field **the same
way the model does**, not "whatever works". Fields with a silent fallback are the worst — they throw no
error, they just quietly change decisions.

## A settlement card's header — its structure and how to put something into it ✅

The `_tmpl$15` template in `commerce-screen-resources-tab.js`:

```
<div class="flex flex-row flex-wrap relative w-full justify-between">
    <SettlementName/>          ← "flex flex-row items-center": the crest, the name, badges
    <Show when={hasFactory}><FactoryTypeDisplay/></Show>   ← "flex flex-row items-center h-10"
</div>
```

`FactoryTypeDisplay` (`factory-type-display.js`) is the `blp:restype_factory_v2.png` cogwheel
plus a black pill with the current factory resource and a return button. It exists
**only in the Modern age** and only for settlements with a factory.

**Two consequences, both visible in the UI:**

- `justify-between` with two children pushes the cogwheel to the very edge of the card —
  far from anything added on the right-hand side.
- `flex-wrap` means that with many badges (a town with a trading post
  and a rail station) **the cogwheel wraps onto a second line**.

**Inserting your own controls — your own container, together with the cogwheel.** Two approaches
failed in practice:

- **absolute** with a fixed offset to clear the cogwheel (`right: 6.25rem`) — it does not know
  the badges' width, nor whether the cogwheel is on the same line;
- **in the flow with `margin-left: auto`**, hoping `justify-between` would pull the cogwheel
  in — the cogwheel stayed at the edge anyway.

What does work is taking the cogwheel into a container of your own, where adjacency does not depend on
anybody else's layout:

```js
const header = card.querySelector('.flex.flex-row.flex-wrap.relative.w-full.justify-between');
header.classList.add(MY_HEADER_CLASS);

let actions = header.querySelector('.my-actions');
if (!actions) { actions = makeElement('div', 'my-actions'); header.appendChild(actions); }
// the name block = the header's first child, but mark it FROM JS, not with :first-child
for (const child of header.children) {
    if (child !== actions) { child.classList.add('my-name'); break; }
}
actions.appendChild(control);
// the cogwheel: found via the inner black pill, because the outer div's classes are generic
const factory = header.querySelector('.bg-black.rounded-lg')?.parentElement;
if (factory && factory.parentElement !== actions) actions.appendChild(factory);
```

```css
.my-header { flex-wrap: nowrap; }                      /* the header never breaks       */
.my-name { flex: 0 1 auto; min-width: 0;               /* instead, the badges inside    */
           flex-wrap: wrap; overflow: hidden; }        /* the name block wrap           */
.my-header .text-xs.text-accent-1 { flex-wrap: wrap; }
.my-actions { display: flex; flex-wrap: nowrap; flex: 0 0 auto; margin-left: auto; }
.my-actions > .h-10 { flex: 0 0 auto; margin-left: 0.75rem; }   /* the cogwheel          */
```

⚠️ **The name block cannot be caught with `:first-child`.** As long as no resource is
selected, the name is wrapped in an `Activatable` — the header's first child and its classes
depend on what the player is doing. Marking it with a class from JS is the only reliable way.

⚠️ Moving the cogwheel is a reparenting of a Solid node — permissible **only because**
`hasFactory` does not change while the screen is open, and a rebuilt card goes through
the same function anyway.

⚠️ `position: relative` on the control, not `static` — the dropdown menu is its child
and positions itself relative to it. With `static` it would latch onto the header (which has `relative`)
and jump somewhere else.

## The settlement's return button vs. the factory's return button ❗✅

A settlement's card has **two** `.fxs-image-button`s with the same
`blp:resource_return_button_default.png` artwork:

| where | what it does | notes |
|---|---|---|
| in the header, inside `FactoryTypeDisplay` | `model.clearFactoryResources(cityID)` | only with a factory |
| below the header, in `_tmpl$16` (class `mr-1`) | `model.clearAllResources(cityID)` | always |

`card.querySelector('.fxs-image-button')` returns **the one in the header**, because the header comes
earlier in the DOM. So code hiding "return all of the settlement's resources" was hiding the factory
button while the real one stayed on screen — for settlements without a factory everything looked
right, so the bug only surfaced in the Modern age.

```js
for (const button of card.querySelectorAll('.fxs-image-button')) {
    if (!header.contains(button)) button.classList.add(HIDDEN_CLASS);  // everything in
}                                                                      // the header = the factory
```

## Factory resources — the game's rule ❗✅

From the pedia (`LOC_PEDIA_CONCEPTS_FACTORY_RESOURCES_TOOLTIP`, en_us):

> "Factory Resources must be assigned to a Settlement with a Factory to give empire-wide
> bonuses. **Only one type of Factory Resource can be assigned to a Settlement at a time.**"

and from the factory's description:

> "**You can assign multiple copies of the same Factory Resource to a Settlement**, so it
> pays to be efficient!"

**The consequence for an assignment algorithm:** spreading "one into each factory"
is the worst possible strategy. It occupies every factory with a different kind, and then every
surplus copy has exactly one legal place. With more kinds than
factories most of the pool becomes unassignable and in game you see "it assigned only a few
items despite 5 free factories".

The right order:

1. **add to a factory that is already producing something** (only the same kind is legal);
2. **start an empty factory with the kind that has the most copies in the pool** — the one you can
   fill it with.

Additional facts from the model (`commerce-screen-model.js`):

- `factoryResourceData.hasFactory` **does not mean "it has a factory building"**:
  `isTreasureConstructiblePrereqMet() && Game.age == AGE_MODERN && (getNumFactoryResources() == 0 || factoryResourceDefinition != null)`.
- `getFactoryResource()` returns **one** type per settlement, `getNumFactoryResources()` counts copies.
- Factory resources **occupy ordinary slots**: `availableSlots = getAssignedResourcesCap() -
  getAssignedResources().length`, and `getAssignedResources()` includes factory ones too.
- Factories can be built only in settlements on the rail network (hence the "Rail Station" badge).

## The trade routes tab — a card's structure ✅

`trade-route-card.js`, the `TradeRouteCard` component — **registered in `ComponentRegistry`**,
so it can be wrapped. ⚠️ The tab's container itself (`TradeRoutesContainer` in
`commerce-screen-trade-tab.js`) is a plain exported function — it is **not** registered,
so the only signal that this tab has mounted is a card.

The order of a card's (`CardFrame`) children:

| template | what it is | classes |
|---|---|---|
| `_tmpl$` | the header: the crest + the source city's name | `flex flex-row text-secondary uppercase text-lg mb-1 items-center`, the name in `.font-title` |
| `_tmpl$2` | "Delivered to settlement X by Sea" | `p.mt-1.mr-13.font-fit-shrink` |
| `_tmpl$3` | the incoming resources | `flex flex-row flex-wrap mt-4 mr-13` |
| `_tmpl$4` | "+24 gold to <leader>" | **`<div class=mt-2>`** |
| `_tmpl$6` | the leader's portrait + relationship change (`+10`) | `absolute top-1 right-1`, inside it `size-12 mt-2 …` |

⚠️ When hiding the yields row target **`[class="mt-2"]`** (an exact attribute match).
A plain `.mt-2` also hits the relationship badge in the corner, which has `mt-2` among many other classes.

**Take a route's data from the API, not from the text.** The model gives the card only `domainString` —
a ready, translated sentence composed from `LOC_COMMERCE_TRADE_DELIVERED_TO`. Taking it back
apart will break in every language with a different word order. The source is the same call the
model uses:

```js
const trade = Players.get(GameContext.localPlayerID)?.Trade;
const options = TradeRouteSearchOptions.INCLUDE_FAILED + TradeRouteSearchOptions.EXTENDED_STATUS;
trade.projectPossibleTradeRoutes(options).forEach((route) => {
    route.domain === DomainType.DOMAIN_LAND;   // land or sea
    Cities.get(route.targetCityId);            // the city from the card's header
    Cities.get(route.nearestCityId);           // our settlement the resources go to
});
```

⚠️ That call is **expensive** — it is what the model builds the whole tab from. Cache the result and
invalidate on `TradeRouteAddedToMap` / `TradeRouteChanged` / `LocalPlayerTurnBegin`.

**The domain icons** are in `base-standard/data/icons/trade-icons.xml`: `TRADE_ROUTE_LAND`
(`blp:city_add`), `TRADE_ROUTE_SEA` (`blp:city_searoute`), plus `_WAR`, `_OUT_OF_RANGE`,
`_ALLIANCE`. Take them via `UI.getIcon('TRADE_ROUTE_SEA')` — the route chooser uses the same pair
(`trade-routes-model.js`, `getTradeRouteStatusIcon`).

**The instruction line above the panel** (`.text-base.w-full.text-center.my-4` within
`screen-resource-allocation`) is on **every** tab. Keep the hiding rule in a
stylesheet attached to the screen, not to the resources tab — otherwise it comes back on switching to
trade routes.

### The relationship tooltip — why it is narrow ❗✅

The tooltip under a leader's portrait (`base-standard/ui-next/tooltips/relationship-tooltip.js`,
whose root has `data-name="Relationship-Tooltip"` and the classes `fxs-tooltip fxs-relationship-tooltip`)
opens so narrow that every reason for a relationship change breaks across three lines.

**Nothing constrains it from above.** `Tooltip.Frame` has `img-tooltip-border img-tooltip-bg
p-4 min-w-48` — a floor only, no `max-width`. The cause is subtler: all the
rows inside have `w-full`, and a child with a percentage width **contributes nothing to
its parent's natural width**. So the frame falls back to `min-w-72` from its contents (about 16 rem)
and at that width the text wraps.

It is enough to raise the floor — the `w-full` rows will fill whatever they are given:

```css
[data-name="Relationship-Tooltip"] { min-width: 30rem; }
```

⚠️ This UI's unit scale: `w-187` = 41.5555 rem, i.e. **1 unit = 0.2222 rem**.
Hence `min-w-72` ≈ 16 rem, and `min-w-48` ≈ 10.7 rem.

**The "+10" badge under the portrait** (the relationship change) is `_tmpl$5` from `trade-route-card.js`,
a direct child of the `.absolute.top-1.right-1` corner, recognizable by `.size-12`.

## Empire resources — how to compute the ACTUAL effect ❗✅

The card in the empire tab shows the resource's rule ("+1 gold and happiness in all
settlements"), regardless of how many settlements you have or how many copies of the resource. To compute the total,
you have to know **what a given effect scales with** — and that is a property of the effect, not the resource.
The effect names say it plainly:

| effect | scales with | example |
|---|---|---|
| `EFFECT_CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE` | **the number of settlements**, NOT the number of copies | `MOD_GOLD_SETTLEMENT_FLAT_GOLD`, Amount 1 → 12 settlements = +12 |
| `EFFECT_UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE` | **the number of copies**, with a cap | `MOD_NITER_…`, Amount 1, 10 copies → +6 (the cap) |
| `EFFECT_CITY_ADJUST_YIELD_PER_RESOURCE` | copies × settlements meeting the requirements | `MOD_IVORY_CITY_FLAT_HAPPINESS`, Amount 4 |
| `EFFECT_ADJUST_PLAYER_YIELD_PER_SLOTTED_RESOURCE` | the number of assigned copies | `MOD_COCOA_EXCESS_HAPPINESS` |

⚠️ **`PER_AVAILABLE_RESOURCE_TYPE` is not the same as `PER_RESOURCE`.** The first counts
owning the *type* (once), the second every *copy*. Confusing them gives, with six units
of gold and twelve settlements, 72 instead of 12.

⚠️ **The +6 combat strength cap does not exist in the data.** It is not in the modifier's arguments,
there is no `globalparameters.xml`, there is no table — the engine holds it, and in the data it appears only
in the text of the description ("maximum +6"). In the mod it is a constant with a comment; it is the only number
in the calculator that does not come from the game.

**Modifier requirements have to be honored** when computing the reach: "in the homeland",
"cities only" and so on narrow the number of settlements — we use the same evaluator as the assignment
scoring (`planner/effects.js`, `modifierApplies`).

**The card's data** is supplied by the model (`populateEmpireResources`): `type`, `amount`, `iconSrc`,
`title` (the `Name` key), `description` (the `Tooltip` key), `originLeaderIds` and
`tooltips[leaderId]` — ready strings of "how much from which city". The resource classes in this tab:
`RESOURCECLASS_EMPIRE` **and** `RESOURCECLASS_TREASURE`.

**Replacing the tab:** `EmpireResourceContainer` (like `TradeRoutesContainer`) is **not**
registered in `ComponentRegistry`. Since the mod replaces the whole `CommerceScreen` anyway
(see "Adding a tab of your own"), your own component simply goes into the `body:` of the
appropriate `Tab.Item`. A card background like the routes tab's is given by the game's **`card-frame-bg`** class.

### The effect's suffix = the counting rule ❗✅

In the `resources-gameeffects.xml` files these four variants occur **side by side**,
chosen modifier by modifier:

| suffix | uses | counts |
|---|---|---|
| `PER_RESOURCE` | 62 | **every copy** the empire owns |
| `PER_AVAILABLE_RESOURCE_TYPE` | 29 | **the type once**, no matter how many you have |
| `PER_RESOURCE_TYPE` | 3 | as above, at the player level |
| `PER_SLOTTED_RESOURCE` | 7 | only copies **assigned** to settlements |

If `PER_RESOURCE` also meant "once", there would be no reason for a separate
`PER_AVAILABLE_RESOURCE_TYPE` to exist. Hence:

- coal, `+10%` for stations and ports through `EFFECT_CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE`
  → with 6 units **+60%**;
- gold, `+1` in all settlements through `..._PER_AVAILABLE_RESOURCE_TYPE`
  → `+1 × the number of settlements`, **without** multiplying by the number of units.

⚠️ The `Empire="true"` argument **does not mean "once per empire"** — it says WHICH copies to count
(all belonging to the empire, not only those assigned to the building settlement). The multiplier is still
carried by the effect's suffix.

⚠️ The resources' descriptions do not give this away: they are written by hand, in the form "+10% production…",
without "for each". You cannot read the counting rule out of them — you have to go to the modifier.

### A modifier's collection narrows the reach just as the requirements do ❗✅

When computing an effect across the whole empire it is not enough to check `SubjectRequirements` — a separate
piece of information is the **collection**, i.e. who the modifier applies to at all. Four occur in the
resource files:

| collection | uses | multiplier |
|---|---|---|
| `COLLECTION_ALL_CITIES` | 115 | the number of settlements (after the requirement filter) |
| `COLLECTION_ALL_UNITS` | 10 | none — it concerns units |
| `COLLECTION_ALL_PLAYERS` | 10 | 1 — once per player |
| `COLLECTION_ALL_CAPITAL_CITIES` | 6 | 1 — the capital only |

⚠️ Furs (`MOD_FURS_FLAT_HAPPINESS`, +3 happiness) use `ALL_CAPITAL_CITIES`.
Counted like `ALL_CITIES` they give a result multiplied by the size of the empire.
The column is `GameInfo.Modifiers[].CollectionType`.

### ⚠️ `Locale.stylize` STRIPS HTML — a tooltip is text, not HTML markup ❗✅

It is tempting to put your own elements into a tooltip, because the renderer literally does:

```js
this.textElement.innerHTML = Locale.stylize(content);   // tooltip-controller.js
```

**It does not work.** `Locale.stylize` is a translator of the GAME's markers, not a pass — `<div>`
and `<span>` **disappear along with their line breaks**, and the whole list merges into one
paragraph. Verified live: "Origin:Yi Sun-sin: 72x Abalasa1x Gongju1x Komarewski…".

What does work:

| you want | use |
|---|---|
| bold | `[B]…[/B]` (the game uses it ~950 times) |
| a new line | a plain `
` **plus** `white-space: pre-wrap` on `#tooltip-root-content > div` |
| an icon | `[icon:YIELD_GOLD]`, `[icon:ECONOMIC_VP]` |
| indentation | spaces or `	` |

So a "leader card" in a tooltip is done like this: `[B]Name: total[/B]`, with indented
rows below and a blank line between blocks. Frames, backgrounds and rounded corners **are not possible** — that would
require a tooltip component of your own instead of `data-tooltip-content`.

### The tooltip's text lands in `#tooltip-root-content` ❗✅

`tooltip-manager.js` passes the controller two elements from `root-game.html`:
`tooltipRootElement: #tooltip-root` and `tooltipContentElement: #tooltip-root-content`.
The text goes into a bare `<div>` appended to **the second one**. The selector
`.tooltip__content` (tempting, because such a class does exist in the CSS) matches nothing — the
`white-space: pre-wrap` rule has to target `#tooltip-root-content > div`.

### Unit classes: LIGHT and HEAVY are NAVAL classes ❗✅

Nothing in the game displays unit class tags, so it is easy to name them wrongly. Checked
in `age-modern/data/units.xml`:

| tag | units |
|---|---|
| `UNIT_CLASS_LIGHT` | cruiser, destroyer, ironclad → **light ships** |
| `UNIT_CLASS_HEAVY` | battleship, dreadnought, frigate → **heavy ships** |
| `UNIT_CLASS_NAVAL` | everything that floats |
| `UNIT_CLASS_RANGED` / `SIEGE` / `INFANTRY` / `CAVALRY` / `AIRCRAFT` | land (RANGED also covers ships) |

⚠️ "light" and "heavy" without the word "ships" read as land units. The game's descriptions
spell it out ("light naval units"), so a mod's labels must too.

### ⚠️ Resource descriptions are sometimes INCONSISTENT with the modifiers

Checked mechanically: for every resource, the tags from `REQUIREMENT_UNIT_TAG_MATCHES` were compared
with the concepts linked in the text of `LOC_EXP_RESOURCE_*_TOOLTIP`.

**Niter in the Modern age:**

- the modifier `MOD_NITER_INFANTRY_AND_RANGED_COMBAT_STRENGTH`: `UNIT_CLASS_RANGED, UNIT_CLASS_SIEGE`
- the Polish description: "siege, **ranged and heavy naval units**"

That is niter's only modifier in that age (checked both by the `ResourceType` argument
and by `ModifierMetadatas`). The Exploration version had `NAVAL, SIEGE` — it looks as though
the description did not keep up with the effect's change between ages.

**The conclusion:** when computing effects, **the modifiers are the source of truth, not the descriptions**. A description
may list a unit class the modifier does not cover.

### ⚠️ CORRECTION: `PER_AVAILABLE_RESOURCE_TYPE` DOES scale with the number of units

An earlier entry in this file claimed that gold and silver count once per empire,
because their effect is named "per available resource TYPE". **That was wrong.**

**Measured in game:** improving one additional copy of gold raised the yield by roughly as much
as the player has settlements. Under the "once per type" interpretation nothing should have changed.

Both families — `PER_RESOURCE` and `PER_AVAILABLE_RESOURCE_TYPE` — multiply by the number
of copies owned. What distinguishes their names remains unknown; **it is not the counting of copies**.

**The general principle:** a name in the data is a hypothesis. A measurement in a running game beats it.
Before basing a calculation on a naming convention, ask for one observation from an actual game.

### ⚠️ CORRECTION 2: `PER_RESOURCE_TYPE` (the player variant) ALSO scales with the number of units

The same mistake, only in the other variant — and it survived a round longer, because the correction above
concerned only `PER_AVAILABLE_RESOURCE_TYPE`. The symptom: **Wine [2]** showed
`One: +10 culture` and `All: +10 culture`.

```xml
<Modifier id="MOD_WINE_GOLDEN_AGE_CULTURE"
          collection="COLLECTION_ALL_PLAYERS"
          effect="EFFECT_PLAYER_ADJUST_YIELD_PER_RESOURCE_TYPE">
    <SubjectRequirements><Requirement type="REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE"/></SubjectRequirements>
    <Argument name="Amount">10</Argument>
</Modifier>
```

**The final rule: all four suffixes count units.** The suffix says nothing about counting.
What differs between them is the **reach** — and the reach sits in `collection` anyway:

| effect | reach | multiplier |
|---|---|---|
| `CITY_ADJUST_YIELD_PER_RESOURCE` | the collection's settlements | `amount × settlements × units` |
| `CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE` | the collection's settlements | `amount × settlements × units` |
| `PLAYER_ADJUST_YIELD_PER_RESOURCE_TYPE` | `COLLECTION_ALL_PLAYERS` → **1** | `amount × units` |

So in code it is one branch, not three — the difference is handled by the function counting settlements, because for
the `PLAYER` collection it returns `1`.

Exactly two resources use it: **Wine** (culture, Antiquity 5 / Exploration 10) and
**Furs** (gold, Exploration) — both only during a Celebration.

### Unit classes OVERLAP — one ship belongs to four ❗✅

Checked in `age-modern/data/units.xml`, all of the battleship's tags (`UNIT_BATTLESHIP`):

```
UNIT_CLASS_SIEGE, UNIT_CLASS_NAVAL, UNIT_CLASS_HEAVY, UNIT_CLASS_RANGED,
UNIT_CLASS_COMBAT, UNIT_CLASS_ELITE_NAVAL_HEAVY, UNIT_CLASS_AUTOEXPLORE
```

That is why the same ship gets **both** the niter bonus (`RANGED, SIEGE`) **and** the oil one
(`CAVALRY, HEAVY`) — you can see it in game in the combat strength breakdown ("+6 from niter, +5 from oil").

⚠️ The consequence for display: the list of classes from a modifier **is not** a disjoint
partition of units. The label "ranged, siege" is true, but a player looking at
their battleship will not recognize that it is in that set. The game's descriptions describe the same effect
in different words ("heavy naval units") and **both agree with the mechanics** —
this is not a contradiction, just two descriptions of the same set of ships.

### How to list the unit classes covered by an effect: a CONTAINMENT test ✅

Since the classes overlap, the modifier's tag list alone is misleading. The solution that
gives a complete and truthful set:

1. collect the units having **any** of the modifier's tags → the set `covered`;
2. list every named class `C` for which **all** of `C`'s units belong to
   `covered` (containment, not intersection).

The result on the Modern age's data:

| resource | modifier's tags | listed classes |
|---|---|---|
| niter | RANGED, SIEGE | ranged, siege, **heavy ships** |
| oil | CAVALRY, HEAVY | cavalry, heavy ships |
| coal | LIGHT | light ships |
| rubber | AIRCRAFT, INFANTRY | air, infantry |

Niter gains "heavy ships", because **every** heavy ship in that age is also ranged
or siege — exactly what the game's description says. "All ships" does not make it, because
light ships are naval and are not covered; the containment test does not let you promise a class
the effect does not cover entirely.

The source: `GameInfo.TypeTags` restricted to rows whose `Type` occurs
in `GameInfo.Units` (TypeTags holds tags for everything, including resources).

**A refinement:** after the containment test you also have to remove the **naval halves** when
the whole `UNIT_CLASS_NAVAL` class is covered. Otherwise in the Exploration age niter
(`NAVAL, SIEGE`) lists "heavy ships, light ships, all ships, siege",
which is the same thing said three times.

⚠️ **Do not turn this into a general "remove a class contained in another" rule.** In the Modern age
every heavy ship happens to be ranged as well, so the general rule would delete
"heavy ships" from niter — the one entry a player looking at their battleship is
looking for. Containment between roles (heavy ⊂ ranged) is an accident of the unit roster
in that age; containment within the fleet (light, heavy ⊂ ships) is taxonomy.

The result after both steps:

| age | resource | the card shows |
|---|---|---|
| Modern | niter | heavy ships, ranged, siege |
| Modern | oil | cavalry, heavy ships |
| Exploration | niter | all ships, siege |

### ⚠️ `CollectionType` is in `DynamicModifiers`, not in `Modifiers`

The XML makes it look as though the collection were an attribute of the modifier:

```xml
<Modifier id="MOD_FURS_FLAT_HAPPINESS" collection="COLLECTION_ALL_CAPITAL_CITIES"
          effect="EFFECT_CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE">
```

In the database it decomposes differently: `collection` **and** `effect` go into
**`DynamicModifiers`**, keyed by `ModifierType`. The row in `Modifiers` has only
`ModifierId` and `ModifierType`.

```js
const byType = new Map();
GameInfo.DynamicModifiers.forEach((e) => byType.set(e.ModifierType,
    { effect: e.EffectType, collection: e.CollectionType }));
GameInfo.Modifiers.forEach((e) => { /* e.ModifierType -> byType */ });
```

⚠️ `modifierRow.CollectionType` returns `undefined` **without an error**, so code reading it
from there simply loses the reach information silently. For us the result: furs (+3 happiness
**in the capital**) were multiplied by the number of all settlements — with 12 settlements and 6 units
the card showed +216 instead of +18.

### ⚠️ A route's card: the visible panel is a CHILD, not `.trade-route-card`

`trade-route-card.js`:

```js
const [local, cardFrameProps] = splitProps(props, ["tradeRoute", "autoFocus", "class", "onFocus"]);
const content = createComponent(CardFrame, mergeProps(cardFrameProps, { … }));
return createComponent(Activatable, { class: "focusable-card-activatable trade-route-card", children: content });
```

`style` is **not** on the `splitProps` list, so it stays in `cardFrameProps` and lands on
**`CardFrame`**. Which means the `width` and `margin-right` computed by the tab land on
the inner frame — the one you see — while `.trade-route-card` is only the `Activatable` around
it and the game **never sizes it**.

**The consequence:** setting any width on `.trade-route-card` does not change the panel's
appearance. You have to target the child:

```css
.trade-route-card > * {
    width: 100% !important;
    margin-right: 0 !important;   /* 12px for every card except the last in a row */
    margin-left: 0 !important;
}
```

⚠️ That `margin-right` was the visible unevenness of the columns: the tab gives 12 px of margin
to every card **except the last in a row**, so the third column looked exactly
that much wider.


---

## The treasure convoy card — what can be changed without rewriting the component ✅

`base-standard/ui-next/screens/commerce/treasure-convoy-card.js`. The card is
`ComponentRegistry.register({ name: 'TreasureConvoyCard' })`, but **registration does not help**
when removing something from inside it: `createInstance` builds all of the content in a single expression,
so an override with a higher priority means copying ~200 lines of the game's code. Two cheaper routes:

### 1. Fields from the model — we replace the data, not the DOM ✅

The card renders `props.fleet.treasureFleetText` through `insert(_el$5, () => ...)`, so
**whatever** we put there gets drawn. The model builds it as `L10n.Stylize({ text })`,
and `L10n.Stylize` does `spread(_el$, mergeProps(other, { innerHTML }))` — i.e. **extra
props land as attributes on the element**. Hence a tooltip without touching the DOM:

```js
L10n.Stylize({
    text: `+${gold} [icon:YIELD_GOLD]   +${gdp} [icon:ECONOMIC_VP]`,
    'data-tooltip-content': Locale.compose('LOC_MY_KEY'),
});
```

The raw numbers are **not** on the fleet object (the model puts them straight into a sentence). Reading
them from the settlement:

```js
const resources = Cities.get(fleet.cityID)?.Resources;
resources.getProducedTreasureFleetGold();   // gold per convoy
resources.getProducedTreasureFleetGDP();    // GDP per convoy
```

The GDP icon's token is `[icon:ECONOMIC_VP]`.

### 2. A header from `L10n.Compose` — CSS will NOT reach it ❗✅

```js
createComponent(L10n.Compose, { text: "LOC_COMMERCE_TREASURE_RESOURCES_TITLE" })
```

`L10n.Compose` is `(props) => createMemo(() => Locale.compose(props.text ?? ''))` — it returns
a **bare text node**, with no element at all. There is no selector that would hit it, and
a `MutationObserver` on the cards is exactly the class of solution that caused the game
to hang (quirk #57).

**Cheaper and safer: blank out the localization key itself** — as long as it is used in
one place (here: exactly one occurrence in the whole game, checked with `grep` over
`Base/`). Overriding an existing key is `<Replace>`, not `<Row>`:

```xml
<!-- text/en_us/…: English sits in <EnglishText>, WITHOUT a Language attribute -->
<EnglishText>
    <Replace Tag="LOC_COMMERCE_TREASURE_RESOURCES_TITLE"><Text></Text></Replace>
</EnglishText>

<!-- text/pl_PL/…: the other languages in <LocalizedText>, WITH a Language attribute -->
<LocalizedText>
    <Replace Tag="LOC_COMMERCE_TREASURE_RESOURCES_TITLE" Language="pl_PL"><Text></Text></Replace>
</LocalizedText>
```

⚠️ **One row per language.** A language without such a row still sees the header — because the game
keeps its translations in `base-standard/l10n/<locale>_Text.xml` as `<Replace … Language=…>`,
and our English row does not cover them. When adding further locales to a mod
this row has to be duplicated along with the rest.

An empty text node in a `flex` container takes up no height: `CardFrame` inserts children
**directly** into the element with the classes (`insert(_el$, () => props.children)`), and an anonymous
flex element with no content does not render.


---

## The trade route limit — the API and a trap ✅

The limit is **per leader**, not per empire. There is no single number for "how many routes can I have":

```js
const trade = Players.get(GameContext.localPlayerID)?.Trade;

trade.countPlayerTradeRoutes();            // all of our routes, in total
trade.countPlayerTradeRoutesTo(playerId);  // routes to THAT player
trade.getTradeCapacityFromPlayer(playerId);// the limit with THAT player
```

We total it over `Players.getAlive()` with a `player.isMajor` filter, skipping ourselves and
`Players.get(local).Diplomacy.hasMet(player.id)` — a limit against an unknown player is not
a slot you can use.

⚠️ **Do not mix `countPlayerTradeRoutes()` with the sum of `...To(id)`.** The first also counts routes
outside that sum, so the header stops matching the per-leader breakdown — and the breakdown is
something the player can check by eye. Either the sum, or nothing.

⚠️ **A free slot ≠ a route that can be established.** Range and war block independently of
the limit. We take the distinction from the projection, the same one the model uses:

```js
const options = TradeRouteSearchOptions.INCLUDE_FAILED + TradeRouteSearchOptions.EXTENDED_STATUS;
trade.projectPossibleTradeRoutes(options).forEach((route) => {
    if (route.status?.includes(TradeRouteStatus.SUCCESS)) {
        startable.add(Cities.get(route.targetCityId)?.owner);   // a leader we can trade with RIGHT NOW
    }
});
```

`SUCCESS` means "all criteria satisfied, all that is left is to sign".

---

## Room for your own summary above the tabs ✅

The parent of `[data-name="TabList"]` is `position: relative`, and the tab bar itself is centered
inside it — so the left side stands empty and nothing has to be measured:

```css
position: absolute; left: 2rem; top: 0.15rem; z-index: 20;
```

The same anchoring is used by: the "Assign all" buttons (the resources tab),
the yield summary (empire), the routes summary (trade) and the "?" mark (treasure).
They **do not collide**, because only one tab is open at a time — but each of those
modules has to clean up after itself in `onCleanup`, because **the row belongs to the screen, not to the tab**
and outlives leaving it.

⚠️ If the element is added from a `MutationObserver`, it has to be **idempotent** — see
quirk #57. The pattern: an "already shown" flag **plus** a check that the element is still
in the tree (the screen can rebuild the row on its own):

```js
if (shown && document.querySelector(`.${CLASS}`)) return;
```

---

## Clicking a card in treasure cities does NOT close the screen ✅

All three clickable places on a convoy's card call only `Camera.lookAtPlot`:

| click | the model's handler | effect |
|---|---|---|
| the settlement's name/banner | `handleClickCityName` | `Camera.lookAtPlot(city.location)` |
| the resource's icon | `handleClickUnimprovedTreasure` | `Camera.lookAtPlot(location)` |
| the convoy's flag | `handleClickTreasureFleet` | `Camera.lookAtPlot(unit.location)` |

So the map moves **underneath** the still-open screen and the player does not see it until they
close it themselves. This is not a bug for a mod to fix (closing the screen for the player would change
the game's behavior) — it is something to explain in a tooltip.


---

## Factory resources — the modifiers and how they are counted ❗✅

There are eight of them in the Modern age: **coffee, citrus, cotton, cocoa, tea, kaolin, quinine,
tin** (`ResourceClassType = RESOURCECLASS_FACTORY`). Each gives a percentage, and **only when
it sits in a settlement with a factory** — unassigned it gives nothing.

| resource | effect | arguments |
|---|---|---|
| coffee | `CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_SLOTTED_RESOURCE` ×2 | `Amount=5`, `ConstructibleClass` = BUILDING / WONDER |
| citrus | `CITY_ADJUST_UNIT_PRODUCTION_PER_SLOTTED_RESOURCE` | `Percent=5`, `Domain=DOMAIN_SEA` |
| cotton | the same | `Percent=5`, `Domain=DOMAIN_LAND` |
| cocoa | `ADJUST_PLAYER_YIELD_PER_SLOTTED_RESOURCE` | `Amount=3`, `YIELD_HAPPINESS` |
| tea | the same | `Amount=3`, `YIELD_SCIENCE` |
| kaolin | the same | `Amount=3`, `YIELD_CULTURE` |
| quinine | `UNIT_ADJUST_HEAL_PER_RESOURCE` | `Amount=1`, `COLLECTION_ALL_UNITS` |
| tin | `CITY_ADJUST_GROWTH_PER_RESOURCE` | `Percent=3`, **`GlobalSlots=true`** |

### ⚠️ Three traps that make code written for empire resources read nothing but zeros here

1. **The number is sometimes in `Percent`, not `Amount`.** Citrus, cotton and tin. `Number(map.get('Amount'))`
   gives `NaN` and an earlier `return` cuts out the whole modifier.
2. **`ConstructibleClass`, not `ConstructibleType`.** Reading a building's name through
   `GameInfo.Constructibles.lookup()` will return nothing — it is a class (`BUILDING`/`WONDER`),
   not a specific building.
3. **Do NOT multiply by the number of settlements.** `PER_SLOTTED_RESOURCE` and `GlobalSlots=true` mean that
   what counts is **the total number of slotted units in the empire, once**, and the percentage applies where
   the collection says. Four coffees = **+20% in every settlement**, not +20% per settlement.
   This is **the opposite** of how empire resources aggregate — the mistake inflates the numbers by
   the size of the empire.

### ⚠️ SETTLED: the factory bonuses are GLOBAL, not only in cities on the rail network ✅

The doubt was justified — rail comes up in factory descriptions often enough that it is easy
to conclude it limits the bonus's reach too. **It does not.** Three independent proofs:

1. **The game's tutorial, `LOC_TUTORIAL_FACTORY_RESOURCES_BODY`:** *"To slot a Factory Resource
   into a Settlement, it must have a Rail Connection and a Factory. (…) **Once the Resource
   has been slotted it becomes an Empire Resource.**"* — rail is a condition for **slotting**,
   and a slotted resource behaves like an empire one, i.e. everywhere.
2. **The rail requirement sits on the BUILDING, not on the resource.** `BUILDING_FACTORY`'s description: *"Must be
   built in a Settlement connected to the Capital by Railroad."* The gate is on where
   a factory may be built — not on where the bonus applies.
3. **None of the nine factory modifiers has A SINGLE `<Requirement>`.**
   Checked with `grep` over both `resources-gameeffects*.xml` files in `age-modern`.
   The collections are `ALL_CITIES` / `ALL_PLAYERS` / `ALL_UNITS` — i.e. everything.

So: we count the slotted units across the whole empire and apply the percentage everywhere. No filtering of
settlements by the rail network.

### Reading the state

```js
// slotted: ONE kind per settlement, any number of units of that kind
city.Resources.getFactoryResource();       // the type (or none)
city.Resources.getNumFactoryResources();   // how many units in that settlement

// owned: ONE entry = ONE UNIT (the game's model counts the same way in populateEmpireResources)
Players.get(local).Resources.getResources()
    .filter((r) => GameInfo.Resources.lookup(r.uniqueResource.resource)
                       ?.ResourceClassType === 'RESOURCECLASS_FACTORY');

Game.Resources.getOriginCity(resource.value);   // where that unit came from
```

Unassigned = owned − slotted. Compute it as a difference, not with a separate read — otherwise those two
sections can show a total that does not match what the player actually has.

### GDP from slotted resources — this is NOT a modifier ✅

It is not in `resources-gameeffects.xml` and no code walking the modifiers will find it. It sits
in the **`VictoryScorings`** table:

```xml
<Row VictoryType="VICTORY_ECONOMIC_MODERN"
     TrackerType="VICTORY_TRACKER_SLOTTED_RESOURCE_CLASS"
     ScoringId="VICTORY_TRACKER_SLOTTED_FACTORY"
     Points="3" Data="RESOURCECLASS_FACTORY" RequiresActivation="true" StaticCarryover="true"/>
```

```js
GameInfo.VictoryScorings.find((r) => r.ScoringId === 'VICTORY_TRACKER_SLOTTED_FACTORY').Points;
```

⚠️ **Read it from the table, do not hardcode `3`** — that is exactly the number a balance
patch touches, and written into the code it would still look fine while already being wrong.

⚠️ **There are also other rows adding GDP for the same resources**, e.g.
`VICTORY_TRACKER_FACTORY_RESOURCES_INDUSTRIAL_PARK` (`Points="1"`) — the American unique
quarter, the Industrial Park. Detecting it requires walking a settlement's tiles, so the mod does not
count it, but it **says so in the tooltip** instead of quietly showing an understated number.

Beware of contradictory in-game texts: `AdvisorText` says "2 points of GDP", `VictoriesText` "+3
GDP". The table says 3 — and the table is the source of truth.

### Icons outside `UI.getIcon(yield)`

```
growth rate   blp:fi_growth_rate_64
healing       blp:fi_action_heal_64
GDP           blp:fi_victorypoint_economic_64
```
(from `base-standard/data/icons/text-icons.xml`, the `FONTICON` context)


---

## The description next to a number: do NOT shorten it — move it to a line of its own ❗✅

The symptom: on a card with two bonuses ("+10% production" + "+1 strength") the first one's
description ran over the second.

### What did NOT work

1. **`text-overflow: ellipsis`** — this renderer does not consider "squeezed by flex" to be
   a **definite** width, so the element keeps its full natural width and draws over
   its neighbor. You would have to set a pixel width from JS.
2. **Measuring the free space after layout** — written and thrown away, for two reasons:
   - ⚠️ **`requestAnimationFrame` after `onMount` is sometimes TOO EARLY.** The card does not have
     its dimensions yet, `getBoundingClientRect().width` comes out 0, the free space comes out
     zero and the code hides **all** the descriptions. That is exactly what happened live.
   - ⚠️ Even with a good measurement: **hiding or truncating a description is a loss of
     information, not a solution.** "+6 [sword]" without words does not say which units it applies to —
     and that is the only reason that line exists. A tooltip does not save it, because you have to know
     there is something to hover over.

### What did work

**The description leaves the row of numbers for a line of its own below** (a "legend"), keyed by that
bonus's icon:

```
One:        +10% [production]   +1 [sword]
All:        +60% [production]   +6 [sword]
[production] Rail Station, Port
[sword] light ships, heavy ships
```

Three things stop hurting at once: the description has **the card's full width**, it is written **once**
instead of in both rows, and it is allowed to **wrap** instead of being cut off. No measuring, no
hiding.

⚠️ **The icon is a key only as long as the icons DIFFER.** On the factory tab coffee,
citrus and cotton all have the production icon. So: if two bonuses on a card have the
same icon, the legend line gets the number as well.

Since the row of numbers no longer holds anything superfluous, it gets `flex-wrap: wrap` instead of
`nowrap` + `overflow: hidden` — a second line costs less than an unreadable number.

### ⚠️ The legend ONLY when there is more than one bonus

With **one** bonus on a card there is no problem — a single number and its description fit
on one line, and that is better, because **the cards in a row stay the same height**. A legend
costs an extra line, so we pay for it only where it buys something.

- the empire tab: almost every card has two bonuses → a legend
- the factory tab: almost every one has one → the description inline

---

## Cards in a row have different heights ❗✅

A `flex-wrap: wrap` row **stretches** its children (the default `align-items: stretch`), so
`.card` really does have the height of the tallest in the row. But `.card` is only a spacer —
**the visible frame is the element inside**, and that one takes the height of its own text:

```css
.card__inner { flex: 1 1 auto; }   /* without this the frame is shorter than the card */
```

The same trap as with the trade route panels and convoy cards: the element with the
"obvious" class is not the one you see.

---

## A percentage of a yield turned into absolute numbers — the double-counting trap ❗✅

"+30% science" says nothing without knowing how much science you have. The empire's per-turn yield (the one from
the top panel):

```js
Players.get(GameContext.localPlayerID)?.Stats?.getNetYield(YieldTypes[yieldType]);
```

⚠️ **`net × percent / 100` OVERSTATES the result.** The net yield **already includes** the bonus from slotted
resources, and 30% is computed from the state **before** itself, not after. It has to be backed out:

```
before = net / (1 + applied/100)
worth  = before × percent/100  =  net × percent / (100 + applied)
```

where `applied` is the **total** factory percentage for that yield from all the already
slotted units (not just from this one resource). The same denominator handles both questions:

- "how much of the current science does tea provide" → `percent` = what tea gives
- "how much would slotting the units lying around add" → `percent` = what they would add, with the denominator unchanged

This is exact if the game **multiplies** percentages, and approximate if it **adds** them to
percentages from other sources — which cannot be checked from the UI. That is why the number goes on screen
as **"≈"** and the tooltip says plainly that it is an estimate.

**It can only be computed for yields that have a single number in the panel** — science (tea),
culture (kaolin), happiness (cocoa). The remaining factory resources multiply production towards
a specific thing, or the growth rate; there is nothing to take a percentage of, and guessing would be worse
than leaving the bare percentage.

---

## Reloading the screen — it works, but it is almost always the wrong answer ❗✅

**Date: 2026-08-18.** Tested in game — and **rejected** as a solution.

`commerce-screen-model.js` builds `tradeRouteTabData` **exactly once**, when the model is created,
and **never** rebuilds it for the entire duration of one opening of the screen — neither on an event nor
from a timer. The model sits in a `createMutable`, but the `populateData` / `populateTradeRoutes` functions are
closures inside `createCommerceScreenModel` and **nothing exposes them**. So when the game's state
changes underneath the screen, there is nothing to ask for a refresh.

Closing and reopening **does work**:

```js
ContextManager.pop('screen-resource-allocation');
requestAnimationFrame(() => {
    ContextManager.push('screen-resource-allocation', { singleton: true, createMouseGuard: true });
});
```

⚠️ but **in game it looks terrible**: the whole screen goes out and comes back, the scroll position
and the selection are lost, and it opens on the first tab (you have to switch back to the right one
separately). As a reaction to clicking one button on one card it is disproportionate — the user
rejected it immediately.

### ✅ What to do instead

**Redraw your own decorations, without touching Solid.** Everything the mod drew on a card itself
(buttons, warnings, group headers, the summary) is ordinary DOM injected next to
the game's elements — it can be rebuilt at any moment, and Solid knows nothing about it. In practice:
clear your own caches and run your own decorating pass (for us `forgetTradeRoutes()` +
`scheduleDecorate()`), wired up with a `CustomEvent` on `window`.

⚠️ **Clearing the cache alone is not enough** — that only fixes the *next* pass,
and on a screen whose every card belongs to the game a `MutationObserver` may not fire for
a long time. The redraw has to be scheduled explicitly.

### What this cannot do

A card **will not jump between the "available" ↔ "unavailable" sections** — those are two different `<For>`s over
two different arrays, i.e. the one thing `reconcileArrays` will not survive being moved
from outside. The only answer to that is reloading the screen, so **it is left to the player**
(they will close and reopen it themselves) rather than done under their fingers.

⚠️ Before you conclude that a card has to jump — check which section it is really in. For us
the "no trade route slot left" warning hangs on cards in the **available** section (the per-leader limit
is already taken by a merchant en route), so no jump was needed; a redraw was enough.

### If you did have to reload

- the `push` has to wait **one frame** after the `pop`. `pop` detaches the element synchronously, which fires
  Solid's cleanups — but a mod counting live components in a `requestAnimationFrame` (to tell
  a teardown from a dip in re-rendering) will only finish its cleanup in that frame. A `push` in the
  same tick will mount the new cards before that check, the counter will not drop to zero
  and the teardown will be skipped.
- `singleton: true` will nevertheless build a **new** element — `pop` took the previous one off the stack,
  so `getTargetIndex` returns -1. That is the point: a new element = a new model = fresh data.
- A fresh screen opens on the **first tab**. The tab bar is behind a `Suspense`, so
  the right one has to be asked for only once it appears — and **with an engine event**, because
  `[data-name="TabListItem"]` is an `Activatable` (a native click does not work; see
  [25-ui-next-solidjs.md](25-ui-next-solidjs.md), "Clicking a game component FROM CODE").
  Trade is always index 1.
