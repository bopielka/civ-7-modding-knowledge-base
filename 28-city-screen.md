# 28. The City Screen — a map for mods

**Established: 2026-08-26.** Everything below is ✅ read in the game's files and in the
`bz-city-hall` 2.7.3 mod. The basis for the "Better City UI by Najane" mod.

This is everything the player sees after clicking a settlement: the production list, the city details panel,
the growth/specialists panel, the modes for placing buildings and buying tiles, and the map
overlays that accompany them.

## ⚠️ MOST IMPORTANT: this is the **old** framework, not `ui-next`

The opposite of the Commerce screen ([26](26-commerce-screen.md)). All four panels are
`Controls.define` in the old `ui/` framework, so **`Controls.decorate` DOES WORK here**.
That single difference determines the architecture of every mod for this screen.

The exception: **the production list's rows**. `<production-chooser-item>` and
`<production-chooser-unique-quarter-item>` are Solid components registered via
`defineLegacyComponent(...)` and embedded in the old DOM as custom elements. So the screen is
a **hybrid**: an old panel with Solid children.

✅ **How to take over such a row:** call `defineLegacyComponent` with the same name **later** —
the render function lands in a plain `Map` with no priority check, so the last write wins,
not the highest priority. The full explanation and the traps (among them that you have to declare
**every** `data-*` attribute you read) are in [25](25-ui-next-solidjs.md), the "CORRECTION 2026-08-26" section.
It works even when another mod has replaced the whole game file through `ImportFiles`.

## Where the panels sit ✅

`base-standard/ui/root-game.html`, the `top-center` slot:

```html
<fxs-hslot class="flex-auto panel-production-slot">
    <panel-production-chooser></panel-production-chooser>
    <panel-city-capture-chooser></panel-city-capture-chooser>
</fxs-hslot>
<fxs-vslot class="flex self-end items-start">
    <div class="panel-city-details-slot"></div>
</fxs-vslot>
```

- `.panel-production-slot` and `.panel-city-details-slot` are **siblings**, present in the document from
  startup. `panel-production-chooser` reaches for them via
  `MustGetElement(".panel-city-details-slot", document)`.
- ⚠️ `panel-city-details` is **not in the HTML** — the production chooser creates it on demand and it is
  the one that dispatches `city-details-closed` on close.
- The common ancestor of both panels (useful for gamepad focus handling) is
  `.panel-production-slot.parentNode`.

## The four panels ✅

| Element | File | Lines | Class |
|---|---|---|---|
| `panel-production-chooser` | `ui/production-chooser/panel-production-chooser.js` | 1441 | `ProductionChooserScreen extends Panel` |
| `panel-city-details` | `ui/city-details/panel-city-details.js` | 1474 | `PanelCityDetails extends Panel` |
| `panel-place-population` | `ui/place-population/panel-place-population.js` | — | `PlacePopulationPanel` |
| `last-production-section` | `ui/production-chooser/last-production-section.js` | — | — |

### `panel-production-chooser`

The owner of the header with the settlement's name, the previous/next city arrows, the
**Production / Purchase** tab bar, the category accordions, the **Show hidden** checkbox and the button to convert
into a city.

- `isPurchase` is a **getter/setter pair on the prototype** — which is why it can be patched (the pattern is
  below).
- `productionCategorySlots` — one slot per `ProductionPanelCategory`
  (`buildings`, `units`, `projects`, `wonders`).
- `doOrConfirmConstruction(category, type, cb)` — **a single funnel** for every build order and
  every purchase.
- The item list is not built here: the panel calls `GetProductionItems(...)` from
  `production-chooser-helpers.js` and writes the result onto the `<production-chooser-item>` elements
  as `data-*` attributes. **That wall of attributes is the seam** — the Solid component renders only
  what the panel gives it in an attribute.

### `panel-city-details`

Three tabs, with ids from the `cityDetailTabID` enum:

| id | Method | Contains |
|---|---|---|
| `city-details-tab-growth` | `renderGrowthSlot()` | food, population, growth |
| `city-details-tab-buildings` | `renderBuildingSlot()` | buildings, improvements, wonders |
| `city-details-tab-yields` | `renderYieldsSlot()` | the yield breakdown |

✅ **The set of tabs is an attribute**: `tabBar.setAttribute("tab-items", JSON.stringify(...))`.
Adding a tab of your own does not require replacing the class — see the "adding a tab" pattern below.

Prototype methods worth knowing (all replaceable): `renderBuildingSlot`,
`renderYieldsSlot`, `addConstructibleData`, `addDistrictData`, `addImprovementEntry`,
`addProductionTooltip`, `addWarehouseBreakdownTooltip`, `onCollapseAllSection`,
`onCollapseImprovementSection`, `update`, `render`.

The data: `model-city-details.js` → `CityDetailsModel`, published as `g_CityDetails`
(`engine.createJSModel`), with changes announced by the window event `update-city-details`.

### `panel-place-population`

The panel for placing a specialist and for expanding the city. The `PlacePopulationModel` model provides
`hoveredPlotIndex`, `hoveredPlotWorkerIndex`, `hoveredPlotWorkerPlacementInfo` plus per-yield
`CurrentYields`/`NextYields` and `CurrentMaintenance`/`NextMaintenance`.

⚠️ **A specialist's maintenance is only in `*Maintenance`, never in `*Yields`** (already described in
[14](14-quirks-and-gotchas.md)).

⚠️ The panel has several containers — `improvementMinimizedContainer`, `improvementMaximizedContainer`,
`specialistMinimizedContainer`, `specialistMaximizedContainer`, plus `subsystemFrame` and
`placeSpecialistFrame` — and **the game hides some of them depending on the interface mode**. DOM injected
into the wrong one is simply invisible. City Hall writes into all four containers.

## Interface modes and managers ✅

| Object | File | What it holds |
|---|---|---|
| `INTERFACEMODE_PLACE_BUILDING` | `ui/interface-modes/interface-mode-place-building.js` | `decorate(overlay)` draws the hint colors |
| `INTERFACEMODE_ACQUIRE_TILE` | `ui/interface-modes/interface-mode-acquire-tile.js` | the tile/specialist purchase view |
| `BuildingPlacementManager` | `ui/building-placement/building-placement-manager.js` | `urbanPlots`, `developedPlots`, `expandablePlots`, `uniqueQuarterPlots`, `allPlacementData`, `selectPlacementData()` |
| `PlotWorkersManager` | `ui/plot-workers/plot-workers-manager.js` | `workablePlots`, `blockedPlots`, `cityWorkerCap`, `update()` |
| `CityDecorationSupport` | `ui/interface-modes/support-city-decoration.js` | the overlay group, dimming outside the city, slot icons |

Both modes are fetched by name:
`InterfaceMode.getInterfaceModeHandler("INTERFACEMODE_PLACE_BUILDING")` — it returns the **registered
singleton**, so assigning to `.decorate` patches the live object. The managers are singletons too,
patched through `Object.getPrototypeOf(Manager)`.

## Lens layers ✅

| Layer | Class |
|---|---|
| `fxs-building-placement-layer` | `WorkerYieldsLensLayer` |
| `fxs-worker-yields-layer` | `WorkerYieldsLensLayer` |
| `fxs-yields-layer` | `YieldsLensLayer` |
| `fxs-city-borders-layer` | `CityBordersLayer` |
| `fxs-city-growth-improvements-layer` | `CityGrowthImprovementsLensLayer` |

⚠️ **The first two are TWO SEPARATE INSTANCES OF THE SAME CLASS.** A patch on the class hits both;
a patch on `LensManager.layers.get(name)` hits one. City Hall patches them separately, because one
draws at map zoom and the other inside the building-placement view — and each gets its own sprite
scale.

⚠️ **Layers register LATER than mods' scripts** (known from [14](14-quirks-and-gotchas.md)).
✅ The cleanest workaround, proven in City Hall: **import the game's module** before you reach for the
layer —

```js
import '/base-standard/ui/lenses/layer/worker-yields-layer.js';   // forces the order
const WYLL = LensManager.layers.get("fxs-worker-yields-layer");
```

The `import` guarantees the module ran before the code below. That is more reliable than `LoadOrder`
and simpler than retrying on `InterfaceModeChangedEventName`.

Your own layer: `LensManager.registerLensLayer(name, instance)`, and then adding yourself to
a lens: `LensManager.lenses.get(lens)?.activeLayers.add(name)`. A layer needs
`initLayer()`, `applyLayer()`, `removeLayer()`.

## Patching patterns — proven in City Hall ✅

### A decorator + a prototype patch with a **one-time lock**

⚠️ This is the most important pattern on this screen and the easiest to miss.
`Controls.decorate` calls the factory **for every instance**, and the city panels are created and destroyed on
every opening of a settlement. A prototype patch without a lock wraps the prototype once per opening and the chain
of wrappers grows without end — it only shows up after a dozen or so minutes of play.

```js
class myDecorator {
    static c = null;                       // the lock
    constructor(component) {
        this.component = component;
        component.myMod = this;            // the bridge from the prototype to the decorator
        this.patchPrototype(Object.getPrototypeOf(component));
    }
    patchPrototype(proto) {
        if (myDecorator.c) return;         // ← without this the chain grows
        const c = myDecorator.c = { proto };
        c.update = proto.update;
        proto.update = function(...args) {
            const r = c.update.apply(this, args);
            this.myMod.afterUpdate(...args);
            return r;
        }
    }
}
Controls.decorate("panel-city-details", (v) => new myDecorator(v));
```

### Patching an accessor property (getter/setter)

A naive `proto.x = ...` wipes the setter. Correctly:

```js
const descriptor = Object.getOwnPropertyDescriptor(proto, "isPurchase");
Object.defineProperty(proto, "isPurchase", {
    ...descriptor,
    set(value) {
        remember(value);
        descriptor.set.apply(this, [value]);   // ALWAYS call the original
    },
});
```

### Adding a tab without replacing the panel

```js
const tabs = JSON.parse(component.tabBar.getAttribute("tab-items"));
tabs.unshift(MY_TAB);                              // { id, icon:{default,hover,focus,pressed}, iconClass, headerText }
component.tabBar.setAttribute("tab-items", JSON.stringify(tabs));
component.slotGroup.appendChild(slot);             // <fxs-vslot id="<the same id>">
```

### `UpdateGate` as a debouncer for somebody else's leak

✅ The "Collapse all" button in `panel-city-details` **leaks listeners** and fires the handler several
times per click. City Hall replaces `onCollapseAllSection` with an `UpdateGate`, which runs
at most once per frame. The pattern works on any leaky game handler.

### Finding anchors by `data-l10n-id`, not by CSS class ✅

The classes on these panels are generated from the game's SCSS and change between patches; mods
rewrite them. Localization ids are stable:

```js
view.querySelector('[data-l10n-id="LOC_BUILDING_PLACEMENT_RESULTS"]')?.parentElement;
```

### A style toggled with a class on `<body>` ✅

Instead of rewriting CSS when an option changes: `document.body.classList.toggle("my-class", flag)`
with all the rules written as `.my-class .whatever { … }`. The cheapest possible option mechanism.

## `<ActionCriteria>` + `<ModInUse>` — switching off the risky half of a mod ✅

The most portable idea from City Hall. The mod's dangerous part gets **an action group of its own**,
switched off when a competitor is installed; the safe part always runs.

```xml
<ActionCriteria>
    <Criteria id="production-ok">
        <ModInUse inverse="1">compact-production</ModInUse>
        <ModInUse inverse="1">drongos-compact-production</ModInUse>
    </Criteria>
</ActionCriteria>
<ActionGroup id="..." scope="game" criteria="production-ok"> … </ActionGroup>
```

⚠️ `<ModInUse>` compares **the mod's id** from its own `.modinfo` — not the folder name and not the
display name.

## ⚠️ Replacing a game file (`ImportFiles`) — what two mods cannot do at once

A file placed at **the same relative path** as a `base-standard` file and listed in
`<ImportFiles>` replaces the game's copy for everyone (already described in
[05](05-ui-javascript.md) and [14](14-quirks-and-gotchas.md) #10). A new fact from practice:

✅ **City Hall replaces five files**, and this is a list no second mod may step into:

```
ui/production-chooser/panel-production-chooser.js
ui/production-chooser/production-chooser-unique-quarter.js
ui/interface-modes/support-city-decoration.js
ui-next/components/production-chooser-item.js
ui-next/components/production-chooser-unique-quarter-item.js
```

⚠️ **When two mods replace one file, one copy wins and the other's changes disappear — with no
error in the logs.** Nothing reports it.

✅ A detail that shows what it is done for: `panel-production-chooser.js` in City Hall is
**a literal copy of the game's 1441-line file with a 16-line diff** — redirecting two imports
to its own helper and adding five `data-*` attributes. All the rest of the changes live in normal
mod files. So: replacing a file is sometimes needed purely to **redirect an import**
or to **widen the wall of attributes** — never to write logic in it.

⚠️ A replacement also freezes the file at the game version it was copied from. A Firaxis patch will not reach
the player until the mod's author copies the file again.

## Small things that would cost time ✅

- **Overlay colors are linear, not sRGB.** Pass them through `Color.convertToLinear(0xRRGGBB)`;
  City Hall's frame color is the same components divided by 4.
- **Sprites support built-in BLPs only.** The check from City Hall:
  `const url = UI.getIconURL(t); const blp = UI.getIconBLP(t);` and the icon is usable only when
  `url == "blp:"+blp || url == "fs://game/"+blp` — otherwise you need a fallback (City Hall draws
  `BUILDING_OPEN` and the first letter of the name). This applies to icons added by other mods.
- **A bug in the game:** `renderYieldsSlot` in `panel-city-details` closes an `<fxs-scrollable>` with a
  `</div>` tag. City Hall replaces that method solely for this.
- **`district.MaxConstructibles`** has to be reduced by 1 for every building with the `FULL_TILE` tag,
  with walls skipped (`ExistingDistrictOnly`) — and **raised to the actual count**, because the AI sometimes
  exceeds the limit.
- **The building under construction is in three different places** and you have to check all three:
  ```js
  city.BuildQueue.getQueuedPositionOfType(hash)                            // in the queue
  Game.CityOperations.canStart(city.id, CityOperationTypes.BUILD, …)       // being built
  city.Constructibles.hasConstructible(hash, false)                        // finished
  ```
  The game's own `findExistingUniqueBuilding` function checks only the third — hence its
  blindness to the building you are currently placing.
- **`Districts.getFreeConstructible(loc, playerID)`** returns the improvement the tile *would
  build* — not the one standing on it. City Hall groups improvements by it in the
  warehouse table.
- **A custom religion name** requires walking `Players.getEverAlive()` and finding the founder:
  `founder.Religion?.getReligionType() == id → founder.Religion.getReligionName()`.
  `GameInfo.Religions.lookup(id).Name` only gives the default name.
- **`getGlobalParamNumber("APPEAL_FOR_HAPPINESS_TILE_YIELD")`**
  (`/core/ui/utilities/utilities-data.js`) — the tile appeal threshold. An example that
  thresholds from the game's rules can be read instead of hardcoded.
- ⚠️ **The opposite example:** `modelTownFocus` in City Hall has each town focus's bonuses
  **hardcoded** (+25 fortification, +2 production per mine…), not read from the modifier tables.
  After a game rebalance that panel quietly lies.

## ✅ A city's yield breakdown — the game computes the whole tree for us

**Established 2026-08-26.** There is no need to derive anything from the modifier tables: `city.Yields` has a full
source-tree API.

```js
city.Yields.getNetYield(YieldTypes.YIELD_FOOD)          // the net number
city.Yields.getYields()                                  // all of them, in GameInfo.Yields order
city.Yields.getYieldsForNode(yieldIndex, path, true)     // → { base: { value, steps }, modifier, tooltip }
city.Yields.getYieldSummaryForNode(yieldIndex, path)     // → { value, base, modifier }
```

`path` is an array from the `CityYieldNodes` enum. All 19 nodes the game itself uses:

```
INCOME
  BUILDING_YIELDS            IMPROVEMENT_YIELDS          YIELD_FROM_RESOURCES
  YIELD_FROM_TRADE           YIELD_FROM_DEMAND_FOR_TRADE YIELD_FROM_PROJECTS
  YIELD_FROM_GREAT_WORKS     YIELD_FROM_CONNECTED_TOWNS  YIELD_FROM_SURPLUS_HAPPINESS
  PRODUCTION_CONVERSION_PROJECT  CITY_EFFECTS_YIELDS     ACTIVE_CIV_TRADITIONS
  INCOME_MODIFIERS
MINUS_DEDUCTIONS
  DEDUCTIONS
    BUILDING_MAINTENANCE     HAPPINESS_UPKEEP            MAINTENANCE_FROM_WORKERS
```

The reference implementation to copy from: `base-standard/ui/city-details/model-city-details.js`,
the methods `addIncomeHierarchy`, `addDeductionsHierarchy`, `addYieldsToHierarchy`, `addYieldSteps`.
They already handle two things that are easy to miss:

- **the "Other" row** — `incomeNode.base.value - childTotal`, i.e. the income the named nodes
  do not explain (`LOC_GLOBAL_YIELDS_OTHER`);
- **percentage modifiers** — `adjustment = base.value * modifier.value / 100`, labeled
  `LOC_YIELD_BONUS_NAME` / `LOC_YIELD_PENALTY_NAME`, broken into items from
  `INCOME_MODIFIERS` → `base.steps` where `GameValueDisplayTypes.PERCENTAGE`.

### ✅ A READY API — without walking the nodes by hand (2026-08-26, from `f1rstdan-cool-ui`)

The whole tree in one call:

```js
import CityYields from '/base-standard/ui/utilities/utilities-city-yields.js';
const yields = CityYields.getCityYieldDetails(city.id);
// → [{ label, value /* a string to display */, valueNum, valueType, type,
//      showIcon, isNegative, isModifier, childData: [ … the same … ] }]
```

`CityYieldsEngine` (221 lines, `base-standard/ui/utilities/utilities-city-yields.js`) handles
the recursion over `base.steps` / `modifier.steps` itself, plus the `LOC_ATTR_SOURCES` grouping, the
`LOC_ATTR_ADD_PERCENTAGE_OF_SOURCES` / `LOC_ATTR_MULTIPLIED_BY_SOURCES` /
`LOC_ATTR_MODIFIERS` labels, formatting of percentages and multipliers, and the collapsing of redundant nodes.
The fallback: `CityDetails.yields` from `model-city-details.js` (the shape `{name, value, children}`).

### ❗ CORRECTION 2026-08-26: `<yield-bar>` is DEAD CODE

An earlier version of this file pointed at `<yield-bar>`
(`base-standard/ui/yield-bar/yield-bar.js` + `model-yield-bar.js`, the `g_YieldBar` model) as the city's
yield bar. **Nothing in the game creates it** — the file loads, the element is defined
and that is that. It is the same pattern as [14](14-quirks-and-gotchas.md) #33: Firaxis leaves
the old implementation on disk and keeps loading it.

✅ The live element is created by the production chooser itself:

```js
cityYieldBar = document.createElement("yield-bar-base");     // panel-production-chooser.js:137
updateCityYieldBar() {
    …
    this.cityYieldBar.setAttribute("data-yield-bar", JSON.stringify(data));  // [{type, value, style}]
}
```

- you reach it as `panel.cityYieldBar` from a decorator on `panel-production-chooser`;
- **the seam is the `data-yield-bar` attribute (JSON)** — adding, removing or changing an entry does not
  require any DOM work. You can also **add entries that are not yields at all** — that is how
  `f1rstdan-cool-ui` puts population and "connection network" there, with its own icons from `UpdateIcons`;
- the structure after rendering (`yield-bar-base.js`): `outerContainer > container`
  (i.e. `lastElementChild`), the value's text in `.text-sm`, an entry's style
  `NONE | GAIN | LOSS` = `0 | 1 | 2`;
- ⚠️ **it renders ASYNCHRONOUSLY** — code that walks the children right after setting the attribute
  will find nothing. The workaround proven in Cool UI: compare
  `root.children.length >= expected`, and if not — a `MutationObserver` on `{childList: true}`
  **with a 500 ms `setTimeout` safety net** that disconnects the observer and does what it can.
  Without the safety net the observer leaks.

### ✅ Tooltips: `TooltipManager.registerType`

```js
import TooltipManager from '/core/ui/tooltips/tooltip-manager.js';
TooltipManager.registerType('my-tooltip', new MyTooltipType());
```

A tooltip type is a plain object with five members:

| Member | Contract |
|---|---|
| `getHTML()` | returns the `<fxs-tooltip>` root, built **once** in the constructor |
| `reset()` | clears only the content, never removes nodes |
| `isUpdateNeeded(target)` | `true` when the target changed; this is also where the actual target is resolved |
| `update()` | fills the content from `this.target` |
| `isBlank()` | `return !this.target` — decides whether the tooltip shows at all |

An element opts in with `data-tooltip-style="my-tooltip"`. Hang the view model **directly on the
DOM node** (`el.yieldData = …`) instead of looking it up again in `update()`.

⚠️ **`isUpdateNeeded` runs on every pointer move.** It should be a reference comparison
and a cache read. Cool UI keeps a comment there begging the next reader not to
break it — because broken, it causes visible stutter.

⚠️ Pass everything that is to be displayed through `Locale.stylize(…)` — that is what renders
`[icon:YIELD_FOOD]`. The inverse is `Locale.plainText(str)`, which **strips the icons** out of a string
so that it can be parsed as a number.

## ✅ Purchasing: the price without switching tabs, and buying something under construction

```js
city.Gold.getBuildingPurchaseCost(YieldTypes.YIELD_GOLD, constructibleType)
city.Gold.getUnitPurchaseCost(YieldTypes.YIELD_GOLD, unitType)
Game.CityCommands.canStartQuery(city.id, CityCommandTypes.PURCHASE, CityQueryType.Unit)
Game.CityCommands.canStart(city.id, CityCommandTypes.PURCHASE, { ConstructibleType: hash }, false)
// → result.Success, result.Cost, result.InsufficientFunds, result.InProgress, result.Plots
```

✅ **Buying a building that is already under construction works in the game itself** — `Construct` from
`production-chooser-helpers.js` does it like this:

```js
if (result.InProgress && result.Plots) {
    const loc = GameplayMap.getLocationFromIndex(result.Plots[0]);
    args.X = loc.x; args.Y = loc.y;      // buy it where it stands
}
Game.CityCommands.sendRequest(city.id, CityCommandTypes.PURCHASE, args);
```

⚠️ **Projects cannot be purchased.** The block is in two places:
`typeInfo.Kind != "KIND_PROJECT"` in `Construct` and `!project.CanPurchase && isPurchase`
in `getProjectItems`.

✅ **Three further purchase rules** (2026-08-26, read in `f1rstdan-cool-ui` 1.9.6):

| Rule | |
|---|---|
| **Towns (`city.isTown`) do not purchase at all** | the game gives them a separate purchase path |
| **Projects never** | as above |
| ⚠️ **Wonders cannot be purchased — UNLESS** the player has fully unlocked the Mughal civic `NODE_CIVIC_MO_MUGHAL_GARDENS_OF_PARADISE` | a real game rule with a single exception |

```js
Game.ProgressionTrees.getNodeState(playerId, nodeType) >= ProgressionTreeNodeState.NODE_STATE_FULLY_UNLOCKED
```

❗ **A signature change in game version 1.1.1:**
`city.Production.getConstructibleProductionCost(type, FeatureTypes.NO_FEATURE, false)` — **three**
arguments. `bz-city-hall` still calls it with one. A comment in Cool UI notes that breakage.

⚠️ A control **inside** an activatable row (the purchase button in a production row) has to call
`event.stopPropagation()` **and** `preventDefault()`, otherwise the row will also add the item to
the queue. Cool UI shipped that bug and fixed it in 1.9.5.

## ✅ Constructible tags — ready-made keys for sorting and filtering

`ConstructibleHasTagType(type, TAG)` from `/base-standard/ui/utilities/utilities-tags.js`;
`getConstructibleTagsFromType(type)` gives all of them. Counts taken from the base game's data:

| Tag | Count | Means |
|---|---|---|
| `AGELESS` | 109 | ageless (warehouses etc.) |
| `UNIQUE_IMPROVEMENT` | 33 | a unique improvement |
| `UNIQUE` | 32 | a unique building |
| `CITY_STATE_UNIQUE_IMPROVEMENT` | 19 | a unique improvement from a city-state |
| `WAREHOUSE`, `FORTIFICATION`, `FULL_TILE`, `URBANCENTER`, `PERSISTENT`, `BRIDGE`, `WATER`… | — | see `constructibleTagNames` and `constructibleTagsToExclude` in `utilities-tags.js` |

✅ **Syncretism needs no separate handling.** The tags sit on the constructible, not on how it was
unlocked — a unique granted by syncretism carries `UNIQUE` exactly like a civilization's own
unique.

The adjacency bonus, ready for sorting:

```js
BuildingPlacementManager.canGetAdjacencyBonuses(constructible.ConstructibleType)
BuildingPlacementManager.getHighestAdjacencyBonus(constructible.$hash)   // the value
BuildingPlacementManager.getNumberOfWarehouseBonuses(constructible.$hash)
```

## Data objects ✅

| Call | Gives |
|---|---|
| `UI.Player.getHeadSelectedCity()` | the selected settlement's `ComponentID` |
| `city.Growth` | `growthType`, `currentFood`, `getNextGrowthFoodThreshold()`, `turnsUntilGrowth`, `projectType` |
| `city.population` / `.urbanPopulation` / `.ruralPopulation` | population |
| `city.Workers` | `getNumWorkers(false)`, `GetAllPlacementInfo()`, `getCityWorkerCap()` |
| `city.Districts.getIds()` → `Districts.get(id)` | `isQuarter`, `isUrbanCore`, `type`, `location` |
| `city.Constructibles.getIds()` → `Constructibles.getByComponentID(id)` | what stands there |
| `city.BuildQueue` | `getQueue()`, `getQueuedPositionOfType`, `getPercentComplete(hash)` (0-100), `getTurnsLeft(TYPE_STRING)`, `currentProductionTypeHash`, `currentTurnsLeft`, `isEmpty` |
| `city.Production.getConstructibleProductionCost(hash)` | the cost |
| `city.getConnectedCities()` / `city.getPurchasedPlots()` | connections / the settlement's tiles |
| `city.Religion` | `majorityReligion`, `urbanReligion`, `ruralReligion` |
| `city.Happiness.hasUnrest` | unrest; ❓ nothing in the game's UI breaks happiness down further |
| `Game.CityCommands.canStart(id, CityCommandTypes.CHANGE_GROWTH_MODE, {Type: GrowthTypes.PROJECT}, false)` | `.Projects` = the allowed town focuses |

Events: `CityGrowthModeChanged`, `CityPopulationChanged`, `CitySelectionChanged`,
`ConstructibleAddedToMap`, `ConstructibleRemovedFromMap`, `ConstructibleChanged`,
`PlotWorkersUpdated`, plus the window events `update-city-details` and `city-details-closed`.

⚠️ Engine events fire **for every player in the match** — a handler without an owner filter
runs thousands of times during an AI turn.

## Mods already sitting here

| Mod | Workshop ID | What it does on this screen |
|---|---|---|
| `bz-city-hall` | 3507102289 | everything above; the full analysis is in `mod-projects/better-city-ui/documentation/03-city-hall-analysis.md` |
| `f1rstdan-cool-ui` | 3510572267 | ✅ analyzed (1.9.6): yield tooltips through `TooltipManager`, a quick-purchase button in the row, extra entries in the yield bar. ⚠️ Its **compact production row layout does not work** since the migration to `ui-next` — the author says so in his changelog. The full analysis is in `mod-projects/better-city-ui/documentation/04-f1rstdan-cool-ui-analysis.md` |
| `EnhancedTownFocusInfo` | 3548476215 | ❓ town focus — overlaps with City Hall's Overview tab |
| `najane-common-specialists-yields` | (the user's mod) | ⚠️ patches `PlotWorkersManager`, `fxs-worker-yields-layer` and `panel-place-population` — the same objects as City Hall, with a different formula for the baseline |

⚠️ **City Hall and the specialists mod compute the "baseline" differently**: City Hall takes `Math.min` over
the components, the user's mod takes the value closest to zero preserving its sign, and treats a missing yield
as an explicit `0`. Where a yield occurs with both signs, they give different numbers.
❓ Whether they visibly conflict in game is **untested**, even though both are installed.

## Population and connected settlements next to the yields

They are not yields — there is no `CityYieldNodes` tree behind them. You build them by hand from the settlement's API:

| Data | Call |
|---|---|
| total / rural / urban / pending population | `city.population`, `.ruralPopulation`, `.urbanPopulation`, `.pendingPopulation` |
| specialists + the per-tile cap | `city.Workers.getNumWorkers(false)`, `city.Workers.getCityWorkerCap()` |
| turns to a new citizen | `city.Growth.turnsUntilGrowth` |
| the food threshold and stockpile | `city.Growth.getNextGrowthFoodThreshold().value`, `city.Growth.currentFood` |
| connected settlements | `city.getConnectedCities()` → `Cities.get(id)` → `.isTown`, `.isCapital`, `.population` |
| whether it is in the trade network | `city.Trade.isInTradeNetwork()` |

⚠️ **Rural population = `ruralPopulation - pendingPopulation`.** Pending citizens sit
in `ruralPopulation`; without subtracting them the rows do not add up to `city.population`.

⚠️ **Food from a specialized town is SPLIT** between all the cities it is
connected to. This settlement receives `townFood / number_of_cities_connected_to_the_TOWN` — the divisor
is on the town's side, not the side of the city reading the tooltip.

⚠️ A specialized town connected to a city **does not grow** (the food goes out), but
`turnsUntilGrowth` keeps counting down. The game shows that counter without comment — it is the only number on
that panel that genuinely misleads.

### Icons without depending on somebody else's mod

The game has its own: **`CITY_CITIZENS`** (population) and **`CITY_SETTLEMENT`** (a settlement), both in
`base-standard/data/icons/city-icons.xml`, default context. Useful in rows:
`CITY_RURAL`, `CITY_URBAN`, `CITY_SPECIAL_BASE`, `CITY_CENTERPIN`, `YIELD_CITIES`, `YIELD_TOWNS`.

⚠️ Cool UI keeps its PNGs under `fs://game/f1rstdan-cool-ui/textures/…`. That path resolves
**only when Cool UI is installed** — using it turns it into a silent dependency.

### Ready-made localization tags (present in all of the game's languages)

`LOC_UI_CITY_INTERACT_CURENT_POPULATION_HEADER`, `LOC_UI_CITY_STATUS_RURAL_POPULATION`,
`LOC_UI_CITY_STATUS_URBAN_POPULATION`, `LOC_UI_SPECIALISTS_SUBTITLE`,
`LOC_UI_ACQUIRE_TILE_ADD_POPULATION_MAX_PER_TILE`, `LOC_UI_CITY_DETAILS_NEW_CITIZEN_IN_TURNS`,
`LOC_UI_CITY_DETAILS_FOOD_NEEDED_TO_GROW`, `LOC_UI_CITY_STATUS_CURRENT_FOOD_STOCKPILE`,
`LOC_UI_CITY_DETAILS_FOOD_PER_TURN`, `LOC_UI_CITY_DETAILS_GROWTH_TAB` ("Citizen growth"),
`LOC_PEDIA_CONCEPTS_PAGE_CONNECTED_1_TITLE`, `LOC_UI_SETTLEMENT_TAB_BAR_CITIES`,
`LOC_UI_SETTLEMENT_TAB_BAR_TOWNS`, `LOC_GLOBAL_YIELDS_SUMMARY_TOTAL_INCOME`.

⚠️ `LOC_UI_CITY_DETAILS_GROWTH_TITLE` **does not exist** — there are `…_GROWTH_TAB` and `…_GROWTH_BREAKDOWN`.

## Highlighting tiles when placing a building (`INTERFACEMODE_PLACE_BUILDING`)

This is drawn by `PBIM.decorate(overlay)` — the handler obtained with
`InterfaceMode.getInterfaceModeHandler('INTERFACEMODE_PLACE_BUILDING')`. Overriding that one
method is enough; you do not have to touch `selectPlacementData`.

Available at drawing time (all on `BuildingPlacementManager`):

| Property | Contents |
|---|---|
| `urbanPlots` | existing urban districts (the "best" color) |
| `developedPlots` | tiles in a district, but not an urban one ("okay") |
| `expandablePlots` | rural / undeveloped ("good") |
| `uniqueQuarterPlots` | districts that complete a unique quarter (VFX) |
| `potentialUniqueQuarterPlots` | `{plotID, uniqueQuarterDef}` |
| `currentConstructible` | the definition of the building being placed |
| `isRepairing`, `cityID` | context |

⚠️ **The colors are AABBGGRR, not RGBA.** The base values: `0xc84db123` (best), `0xc800f2fe` (okay),
`0xc81de5b5` (good). In the game's code they are written in decimal (`3360534819` etc.).

```js
this.plotOverlay = overlay.addPlotOverlay();
this.plotOverlay.addPlots(indexList, { fillColor: 0xc8003ce6 });
this.uniqueQuarterModelGroup.addVFXAtPlot('VFX_3dUI_Hex_Highlight_01', plot,
    { x: 0, y: 0, z: 0 }, { angle: 0, constants: { Color3: [1, 0.992, 0.62], Alpha1: 1 } });
```

⚠️ When overriding `decorate` you have to **repeat all of the base content** — `CityZoomer.zoomToCity` and
`WorldUI.pushRegionColorFilter(city.getPurchasedPlots(), {}, this.OUTER_REGION_OVERLAY_FILTER)`.
Calling the original first achieves nothing if you want to repaint the same tiles: the base color
will stay on top.

### ❗ `findExistingUniqueBuilding` sees only FINISHED buildings

The base method asks only `city.Constructibles.hasConstructible(hash, false)`. The result: when
the first half of a unique quarter is **queued or under construction**, the game stops highlighting that
district for the second half — i.e. exactly when it is needed most.

The fix is a patch on the **prototype** (`Object.getPrototypeOf(BuildingPlacementManager)`), because
`selectPlacementData` calls that method along the way. A constructible can be in three places — see the section
above in this file.

### The "this tile will ruin a unique quarter" rule

For the building being placed, unless it is a repair or `ExistingDistrictOnly`:

1. Collect the districts that already hold half of an unlocked unique quarter
   (`Players.Constructibles.get(owner).getUnlockedUniqueQuarters()` → `GameInfo.UniqueQuarters`).
2. A candidate that **is** such a district → OK only if it is the same unique quarter.
3. A candidate for a **unique building** in a new place → bad, if the other half is already somewhere
   else or the district is "spoiled".
4. A district is spoiled when it holds a building that is **ageless** or from the **current era**:

```js
Database.makeHash(def.Age ?? '') === Game.age
```

⚠️ Only `ConstructibleClass == 'BUILDING'` without `ExistingDistrictOnly` counts — walls do not occupy
a slot. A building from the **previous** era does not block, because it can be built over.

### Building icons on the tiles in placement mode

They are drawn by the `fxs-building-placement-layer` lens layer
(`LensManager.layers.get('fxs-building-placement-layer')`), in the `realizeBuildSlots(district)` method.

⚠️ The game calls it **only for candidate tiles** — from `getPlacementOptions()`, i.e.
`urbanPlots + developedPlots + expandablePlots`, and only where `Districts.getAtLocation()`
returns something. The rest of the settlement stays empty.

⚠️ The parent method's name has **a typo in the game's code**: `realizeBuidlingPlacementSprites`
("Buidling"). You have to reproduce it exactly.

Extending it to the whole settlement without duplication:

```js
let drawn = null;
layer.realizeBuildSlots = function (district) {          // records what the game drew itself
    drawn?.add(GameplayMap.getIndexFromLocation(district.location));
    return original.apply(this, arguments);
};
layer.realizeBuidlingPlacementSprites = function (...a) {
    drawn = new Set();
    try { originalSprites.apply(this, a); drawTheRest(this); } finally { drawn = null; }
};
```

⚠️ **Call through, do not replace.** City Hall replaces `realizeBuildSlots` with a richer version
(specialists, yield badges, a letter on icons that cannot be loaded as a sprite)
and loads earlier — calling the original means its drawing also reaches the
extra tiles.

⚠️ The renderer draws **one placeholder tile per free slot** (`MaxConstructibles`), so without
a "is anything standing here" filter every rural tile gets a row of empty frames. `ExistingDistrictOnly`
constructibles (walls) do not count as development — they do not occupy a slot.

The settlement's tiles: `city.Districts.getIds()` → `Districts.get(id)` → `.location`, `.type`.
