# 05 — UI and JavaScript

The largest category of Civ VII mods. Across the 49 Workshop mods there are **63 uses of `UIScripts`**
vs 18 of `UpdateDatabase` — so the community mostly mods the interface.

## Two UI systems side by side ✅

| | `ui/` (classic) | `ui-next/` (new) |
|---|---|---|
| Technology | Firaxis's own framework (`Component`, `Controls`) | **Solid.js** |
| Files in `core` | 373 JS | 186 JS |
| Files in `base-standard` | 601 JS | 132 JS |
| Entry point | components registered via `Controls.define` | `app.js` mounts into `#solidjs-root` |

The rendering engine is **Coherent Labs (cohtml)** — `core/ui/cohtml.js`.
In practice: HTML + CSS + JS, but not a browser; parts of the DOM API behave differently.

Mods touch both worlds — `bz-map-trix` has files in both `ui/` and `ui-next/`.

## Three ways of modifying the UI

### 1. `UIScripts` — adding a script (most common, safest) ✅

```xml
<UIScripts>
    <Item>ui/tooltips/bz-plot-tooltip.js</Item>
</UIScripts>
```
The script is **added**, it replaces nothing. Your code decides for itself what to hook into.

### 2. `ImportFiles` — overriding a game file with the same path ✅

```xml
<ImportFiles>
    <Item>ui-next/tooltips/plot-tooltip/plot-tooltip.js</Item>
</ImportFiles>
```
✅ Verified: `base-standard/ui-next/tooltips/plot-tooltip/plot-tooltip.js` exists
in the game under an **identical relative path** — the mod shadows it.
Effective, but it **breaks compatibility** with every other mod that touches the file
and breaks with every game patch.

### 3. `ReplaceUIScript` — an explicit swap ✅

```xml
<ReplaceUIScript>
    <Item>ui/policies/policies-and-traditions.js</Item>
</ReplaceUIScript>
```
Used rarely (3 mods: `leugi-diploribbon-tweaks`, `leugi-diploicon-tweaks`,
`stachs-elegant-policies-and-traditions`) — when a component has to be rewritten from scratch.

**Order of preference:** `UIScripts` + a decorator → `ReplaceUIScript` → `ImportFiles`.

## `Controls.decorate` — the official extension point ✅

45 uses across mods. The definition from `core/ui/component-support.js`:

```js
decorate(name, provider)   // registers a decorator for the component with the given name
```

The Firaxis comment in the code says it plainly:
> it will not create a decorator for **existing instances** of the component

⚠️ Hence the importance of `LoadOrder` — the decorator has to be registered before the
component is created.

### The decorator contract ✅

```js
class MyDecorator {
    constructor(component) {
        this.component = component;
        component.myDecorator = this;   // mutual reference
    }
    beforeAttach() { }
    afterAttach()  { }   // this is usually where DOM is added
    beforeDetach() { }
    afterDetach()  { }   // this is where listeners are cleaned up
}
Controls.decorate('component-name', (component) => new MyDecorator(component));
```

## Registering your own component ✅

```js
class bzPlotIconWonders extends Component {
    onInitialize() {
        const wonderType = this.Root.getAttribute("wonder");
        this.Root.style.backgroundImage = `url(fs://game/my-mod/icons/icon.png)`;
        this.Root.classList.add("size-24", "bg-cover", "bg-center");
    }
}
Controls.define("bz-plot-icon-wonders", { createInstance: bzPlotIconWonders });
```
`this.Root` is the component's DOM element. The CSS classes look **Tailwind-like**
(`size-24`, `bg-cover`, `bg-no-repeat`, `bg-center`) ⚠️ — a game convention; I have not checked
whether it is full Tailwind.

## Patching LensManager layers (a cleaner technique than prototype patching) ✅

`LensManager` keeps registered layers in a **public field** `layers` (a Map),
defined in `core/ui/lenses/lens-manager.js`:
```js
class LensManagerSingleton {
  layers = new Map();   // public — reachable from outside
  registerLensLayer(layerType, layer) { this.layers.set(layerType, layer); ... }
}
```
Layers (e.g. `fxs-worker-yields-layer`, `fxs-yields-layer`) are **single instances**
registered once when the file is imported (`LensManager.registerLensLayer("name", new Class())`)
— not components created via `Controls.define`, so `Controls.decorate` does not work here.

Instead of overriding the whole game file (`ImportFiles`/`ReplaceUIScript`), you can **replace
a method directly on the already-registered instance**:

```js
import LensManager from '/core/ui/lenses/lens-manager.js';

engine.whenReady.then(() => {
    const layer = LensManager.layers.get('fxs-worker-yields-layer');
    if (!layer) { console.error('layer not registered yet'); return; }
    if (layer.__myPatch) return;          // guard — patch only once
    layer.__myPatch = true;

    layer.methodToReplace = function (arg) {   // a plain function, not an arrow — `this` = the instance
        // you can call the instance's original helper methods:
        this.otherHelperMethod(...);
        // ...your own logic...
    };
});
```
⚠️ **Advantages over `ImportFiles`:** it does not override a game file (it survives patches and
other mods touching the same file); and over a prototype patch: you do not have to recreate the
"call the original" machinery by hand — `this.otherGameMethod(...)` is enough, because the
instance's remaining methods are untouched.

⚠️ **Timing:** the layer is registered when the game file is imported (`scope="game"`,
base module). If your script has a higher `LoadOrder`, the layer should already exist —
even so, check `if (!layer)` and consider `setTimeout(fn, 0)` as a defensive retry.

The same technique (a public field holding instances) may apply to the game's other singleton
managers — it is worth checking for `class XManagerSingleton { field = new Map()/Array() }`
before reaching for a prototype patch.

## Patching prototypes (last resort) ✅

When a component has no extension point, mods replace a method on the prototype
while keeping the original:

```js
class bzPlotIconsRoot {
    static c_prototype;
    constructor(component) {
        this.component = component;
        component.bzComponent = this;
        this.patchPrototypes(component);
    }
    patchPrototypes(component) {
        const proto = Object.getPrototypeOf(component);
        if (bzPlotIconsRoot.c_prototype == proto) return;   // patch only once!
        bzPlotIconsRoot.c_prototype = proto;
        bzPlotIconsRoot.c_onRemoveIcon = proto.onRemoveIcon;   // keep the original
        proto.onRemoveIcon = function(event) {
            return this.bzComponent.onRemoveIcon(event);
        };
    }
}
```
The "patch only once" pattern (a guard on a static field) matters — without it you will wrap
the method several times when there are several instances.

## Paths and imports ✅

```js
import LensManager from '/core/ui/lenses/lens-manager.js';
import PlotIconsManager from '/core/ui/plot-icons/plot-icons-manager.js';
import '/bz-map-trix/ui/mini-map/bz-panel-mini-map.js';   // the mod's own file
```

The rule: **the path starts with `/` + the module/mod identifier**.
Verified uses: `/core/` (361), `/base-standard/` (91), `/<mod-id>/` (39).

Assets (images, CSS) are addressed via `fs://game/<mod-id>/…`:
```js
this.Root.style.backgroundImage = `url(fs://game/bz-map-trix/icons/icon.png)`;
```

## The most frequently used global APIs ✅

Counted across all 317 JS files of the mods:

| API | Uses | What for |
|---|---|---|
| `GameInfo` | 676 | **database access** — `GameInfo.Constructibles.lookup(type)` |
| `GameplayMap` | 549 | queries about map tiles |
| `engine` | 477 | events and the data model |
| `UI` | 383 | general interface functions |
| `Game` | 320 | game state |
| `Players` | 316 | players |
| `ComponentID` | 192 | identifiers of game objects |
| `InterfaceMode` | 174 | interaction modes |
| `Units` / `Cities` / `Districts` | 163/105/57 | game objects |
| `LensManager` | 148 | map layers/lenses |
| `Controls` | 108 | registering and decorating components |
| `WorldUI` | 86 | drawing in the 3D world |
| `NavTray`, `FocusManager`, `ContextManager` | 60/44/37 | navigation and focus (gamepad/keyboard) |
| `Audio`, `Configuration`, `Input`, `Camera` | 55/48/29/14 | the rest |

## Events — `engine` ✅

```js
engine.on('EventName', handler);    // 306 uses
engine.off('EventName', handler);   //  96 — ALWAYS clean up in afterDetach
engine.whenReady.then(() => { ... });    //  38 — wait for the engine to be ready
engine.trigger('EventName', data);  //  20
```
`engine.createJSModel` / `engine.updateWholeModel` / `engine.synchronizeModels` —
data binding to the view (rarer, mostly in larger mods).

## Map layers (lenses) ✅

```js
LensManager.registerLensLayer("bz-wonder-layer", new bzWonderLensLayer());
LensManager.toggleLayer("bz-wonder-layer");
```
A popular category of mods (`maple-leaves-more-lens`, `slothoth-better-archeology-lens`).

## UI mod performance — engine events ✅

Verified in practice on the `better-commerce-screen-ui` mod (2026-08-23). This is the
most common cause of "the game runs slowly when the mod is enabled".

**1. Engine events are NOT only about you.** `UnitMoved`, `UnitMoveComplete`,
`UnitMovementPointsChanged`, `ConstructibleChanged`, `ResourceAssigned` and most of the rest are
raised for **every player in the match**. Late in the game an AI turn raises thousands of them. The
game itself filters them immediately — `panel-action.ts` opens `onUnitMoved` with the line
`if (data.unit.owner !== GameContext.localPlayerID) { return; }`, and
`panel-production-chooser.ts` for `ConstructibleAddedToMap` asks about the owner of the **tile**
(`GameplayMap.getOwningCityFromXY`). A mod has to do the same, before any work.

Where the owner sits in the payload (from observing the game's sources):

| Field | Events |
|---|---|
| `unit` (ComponentID) | all `Unit*` |
| `constructible` (ComponentID) | `ConstructibleBuildCompleted` |
| `cityID` / `city` | city events |
| `player` (a plain id) | `ResourceAssigned`, `ResourceUnassigned` |
| only `location` | `ConstructibleAddedToMap` — you have to ask the tile |

⚠️ A payload **without** an owner should never be dropped — a missing trigger looks
exactly like a function that does nothing, and that is a worse bug than the cost.

**2. Every `engine.on` is a separate crossing from the engine into JS.** Several modules of one mod
usually want the same names. In this mod there were **53 subscriptions across 27 distinct names** —
`LocalPlayerTurnBegin` six times, `ResourceUnassigned` and `ResourceCapChanged` four times each. One
event in the game entered the mod six times and the same question "is this mine?" was asked six
times. The fix: **one subscription per name**, a list of listeners behind it, the owner computed
**once** per event (lazily — a name whose listeners are all unfiltered never asks at all).

**3. `engine.off` requires THE SAME function reference** that was registered. An inline arrow
passed to `engine.on` and forgotten is a leak for the whole session — subscriptions outlive the
screen, and `startX()` called on every entry into the tab adds more.

**4. Measure before you start guessing.** A single counter in such a shared dispatcher (how many
events of each name and how many ms handling them took, printed once per turn) measures the whole
mod and answers the question reading the code will not: which names really arrive in the
thousands in **this** match.

**5. Other traps of the same class** (all verified in this mod):
- `GameInfo.X.lookup(...)` is a **database call**. The answer is a column in a static
  table and does not change during the game — memoize it by the key the engine gives you.
- `GameInfo.Modifiers` / `GameInfo.DynamicModifiers` have thousands of rows; a `.find()` over them is
  a full scan. Join them into an index once.
- `Database.makeHash(...)` and `Game.getHash(...)` are **lookups, not constants** — do not call them in
  a loop or in a function fired from `refreshActionButton`.
- `Locale.compose(...)` is a call into the game; a settlement's name does not change while the screen is open.
- `Game.PlayerOperations.canStart(...)` is the most expensive of these calls — filter out pairs that
  the data rules out anyway (a CITY-class resource will not go into a Town) before asking the engine.
- `Units.getPathTo(...)` is a **full pathfinding search (A\*)**, not a cheap answer. ⚠️ And
  the most expensive case is the FAILING one: a search that reaches the target stops at the target,
  while one that does not reach it has to exhaust the entire area reachable by the unit before it
  answers "impossible". Never run it in a loop over a list of tiles (e.g. over all of a settlement's
  plots — a developed city has 30-50). Sort by distance from the unit, probe the nearest few and stop
  once you have enough hits. The same goes for
  `Game.UnitOperations.canStart(MOVE_TO, ...)`, which pathfinds internally — the loop
  "getPathTo over everything, then canStart over everything" is a double bill.
- **Count timeouts in milliseconds, not in frames.** `requestAnimationFrame` × N as a time
  limit stretches exactly when the game slows down, and during a turn transition the frame loop may
  not run at all — then the limit never fires. `setTimeout` is not tied to
  the frame loop.
- A `MutationObserver` on `document.body` with `subtree: true` sees the **whole HUD**. Target your
  own screen's element and batch the run into a single frame (`requestAnimationFrame`) — this also
  sidesteps the `reconcileArrays`/`insertBefore` crash when writing to the DOM from a Solid microtask.
