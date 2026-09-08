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

⚠️ **`isPurchase` CANNOT BE SWITCHED OFF IN A TOWN, and the flag survives the promotion** (✅ verified
2026-09-03). The setter in `panel-production-chooser.js` has a guard:

```js
set isPurchase(value) {
    if (value === this._isPurchase || !value && this.city.isTown) return;
    this._isPurchase = value;
    this.productionPurchaseTabBar.setAttribute("selected-tab-index", value ? "1" : "0");
    this.updateItems.call("isPurchase");     // ← the setter rebuilds the list itself
}
```

So `panel.isPurchase = false` in a town is **a silent no-op** — rightly so, a town does not
produce. But `_isPurchase` is set by the `cityID` setter (`city.isTown || shouldReturnToPurchase`)
and **nothing in the game clears it when a town is promoted to a city**. `onCityGovernmentLevelChanged`
calls `updateProductionPurchaseBar`, `updateTownFocusSection` and `updateItems` — it does not touch the flag.

The result: `updateItems` rebuilds the list **still in purchase mode**, so a freshly promoted city
shows every building greyed out with "not enough gold in the treasury" — instead of the production costs
it could simply pay with hammers.

In a clean game the player sees this as the "Purchase" tab being selected and can switch manually. **A mod
that hides the tabs locks the player in that mode with no way out** — then you have to set
`isPurchase = false` from a `CityGovernmentLevelChanged` handler. That is the first moment at which
`isTown` is already false and the setter will accept the value.

✅ Setting the flag **rebuilds the list by itself** (the setter's last line), so a separate call to
`updateItems` is only for the case where the mode was already correct.

⚠️⚠️ **BUT `city.isTown` IS NOT SWITCHED OVER YET WHEN THE EVENT FIRES** (✅ verified
2026-09-03, after a failed first attempt at the fix). Setting `isPurchase = false` straight
from the `CityGovernmentLevelChanged` handler runs **into that same guard** and does nothing.

The proof is in the game's code: its own handler has `city` in hand and **even so** reads the rank from the
event payload, not from the object:

```js
onCityGovernmentLevelChanged({ cityID, governmentlevel }) {
    const city = Cities.get(cityID);
    const isTown = governmentlevel === CityGovernmentLevels.TOWN;   // NOT city.isTown
```

Nothing else explains that line. **A general conclusion, not just about this one field:** with engine
events, read the state from the event payload; the game object may still be carrying the previous
answer. If you need the object switched over (because you are calling an API that checks it itself),
**wait until it switches** — poll in a short, **bounded** loop, and the event payload tells you which
answer you are waiting for. Example: `whenSettled` in
`mod-projects/better-city-ui/ui/screen/government-change.js`.

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

⚠️ **`panel-city-details` does NOT listen to `CityGovernmentLevelChanged`** (✅ verified in the game's code,
2026-09-03). The panel registers only `InputContextChanged`, and its model — `CitySelectionChanged`,
`CityGrowthModeChanged` and `CityPopulationChanged`. The only city-screen panel that reacts to
a town's promotion to a city is `panel-production-chooser` (`onCityGovernmentLevelChanged` →
`updateProductionPurchaseBar`, `updateTownFocusSection`, `updateUpgradeToCityButton`,
`updateItems.call(...)`).

The consequence for every mod decorating the tabs: **after clicking "Convert to city" the details panel
still describes a town** — the town focus, no specialists, food routes — until the player closes and
reopens the settlement. You need your own `engine.on('CityGovernmentLevelChanged', …)`
(the owner filter first) and a call to `panel.maybeComponent?.update()`.

⚠️ Any cache of your own kept **per settlement** has to be dropped at that same moment:
`isTown` flips on the fly and changes the result of every query that reads it.

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
