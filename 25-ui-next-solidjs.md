# 25. `ui-next` — the second UI framework (Solid.js)

**Established: 2026-08-10.** Discovered while working on the "Better Commerce Screen UI" mod.

## ⚠️ MOST IMPORTANT: Civ VII has TWO UI frameworks, not one

Everything described in [05-ui-javascript.md](05-ui-javascript.md) and
[09-cookbook-ui-mod.md](09-cookbook-ui-mod.md) (`Controls.define`, `Controls.decorate`,
`Component`, `data-bind-*`, `.html.js`) applies to the **old** framework in the `ui/` directories.

Alongside it there is a **new** framework in the `ui-next/` directories — ✅ verified in the files:

| | old `ui/` | new `ui-next/` |
|---|---|---|
| Foundation | a custom `Component` + custom elements | **Solid.js** (`core/vendor/solid-js`) |
| View | an HTML string in `*.html.js` + `data-bind-*` | JSX compiled to `createComponent`/`template` |
| Reactivity | `data-bind-value`, manual `update*()` | signals: `createSignal`, `createMemo`, `createEffect` |
| Registration | `Controls.define('name', …)` | `ComponentRegistry.register({name, createInstance})` |
| Model | a singleton class + `engine.on` | `ModelRegistry.register(...)` / a factory + `createContext` |
| Modding | `Controls.decorate` | **`overridePriority`** (see below) |
| CSS classes | the same utilities (`flex`, `mt-4`, `text-secondary`) | the same |

Firaxis writes new screens in `ui-next` already. As of 2026-08-10 these include:
`screens/commerce` (the Commerce screen), `screens/victories`, `screens/load-screen`,
`screens/age-transition`, `screens/choosers/tech-chooser`, `.../culture-chooser`.

**The practical conclusion:** before you start writing an old-style decorator, check whether
the screen has already moved to `ui-next/`. If it has — the old file in `ui/`
is still on disk and is still loaded, but it is **not what the player sees**
(see "Which screen wins" below).

## Where things live ✅

```
Base/modules/core/vendor/solid-js/dist/solid.js         the Solid core
Base/modules/core/vendor/solid-js/web/dist/web.js       the DOM renderer
Base/modules/core/vendor/solid-js/store/dist/store.js   createMutable / createStore
Base/modules/core/ui-next/services/component-registry.js
Base/modules/core/ui-next/services/model-registry.js
Base/modules/core/ui-next/components/fxs-solid-component.js   the old↔new bridge
Base/modules/core/ui-next/components/                   Tab, Dropdown, ScrollArea,
                                                        SearchBar, CollapsibleContainer,
                                                        CardFrame, Activatable, Icon, L10n…
Base/modules/base-standard/ui-next/components/          ScreenFrame, FramedResource,
                                                        YieldDelta, LeaderWithRibbon…
Base/modules/base-standard/ui-next/screens/<screen>/
Base/modules/core/ui-next/sandbox/                      Firaxis's own examples (!)
Base/modules/core/ui-next/reference/                    references: audio, drag&drop
```

`sandbox/` holds ready, simple Firaxis examples (`hello-component.js`,
`simple-binding.js`, `list-example.js`, `effect-model.js`) — the shortest route to
understanding the conventions.

## The override mechanism — designed FOR MODS ✅

This is the most important discovery. `component-registry.ts` says so plainly in a comment:
*"Components can be overriden (for example, by mods) by setting a higher priority"*.

```js
// the game registers:
ComponentRegistry.register({ name: "TradeRouteCard", createInstance: TradeRouteCardComponent });
// the mod registers under THE SAME name with a higher priority:
ComponentRegistry.register({ name: "TradeRouteCard", createInstance: MyCard, overridePriority: 1 });
```

How it works inside (read in the source):
- the registry keeps **one wrapped factory object per name**; `register()` always returns
  that same object and only swaps its `.factory` field if the new priority is higher;
- game code that imported `TradeRouteCard` holds a reference to that wrapped
  object → **after the override it automatically renders the mod's version**, with no re-import.

✅ **Load order does not matter.** A mod can register before the game or after
it — the higher `overridePriority` wins. This entirely removes the "the patch ran
before the object existed" problem that ate a lot of time on the specialists mod
(see [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)).

Analogously `ModelRegistry.register(name, lifecycle, factory, priority)`, where
`ModelLifecycle` is `Singleton` / `SharedInstance` / `PerInstance`.

⚠️ **But:** overriding a model in `ModelRegistry` only works if the consumer
actually calls `Model.get()`. The Commerce screen **calls the `createCommerceScreenModel()`
factory directly**, so registering `"CommerceScreenModel"` with a higher priority will **not**
affect it — see [26-commerce-screen.md](26-commerce-screen.md).

**You can only override a component the game registered.** A plain `export const Foo:
Component = …` without `ComponentRegistry.register` is untouchable — you then have to replace
its parent.

## The old ↔ new bridge: `defineLegacyComponent` ✅

```js
defineLegacyComponent("screen-resource-allocation", { classNames: ["fullscreen"] },
                      () => createComponent(CommerceScreen, {}));
```

It creates a classic custom element with the given name, whose `onAttach` renders a Solid
tree inside (`render()` from `solid-js/web`). Thanks to this, `ContextManager.push("screen-…")`,
the `<template>` in `root-game.html`, tutorials and all the old infrastructure work unchanged.

### Which screen wins when an old and a new version both exist ✅

`defineLegacyComponent` calls `Controls.define(name, { priority: ranking })`, where
`ranking = Modding.getModRankByURL(calleeURLOrPriority)` or **`1` by default**.
The old `Controls.define` without a priority gives `0`. That is why, with two definitions of
`screen-resource-allocation` (the old one in `ui/resource-allocation/` and the new one in
`ui-next/screens/commerce/`), **the `ui-next` one wins**.

The same mechanism is the route to a mod taking over a screen entirely:

```js
defineLegacyComponent("screen-resource-allocation",
    { classNames: ["fullscreen"], calleeURLOrPriority: import.meta.url },
    () => createComponent(MyWholeScreen, {}));
```

⚠️ Not verified in practice; `Modding.getModRankByURL` should give a mod a rank > 1.
Overriding via `ComponentRegistry` is less invasive and should be preferred.

### ❗✅ CORRECTION 2026-08-26: the render function is a plain `Map.set` — THE LAST ONE WINS

Priority **does not decide what gets drawn**. The full body of the function, read in
`core/ui-next/components/fxs-solid-component.js`:

```js
function defineLegacyComponent(name, options, renderFunction) {
  const ranking = typeof options.calleeURLOrPriority == "number" ? options.calleeURLOrPriority
    : options.calleeURLOrPriority ? Modding.getModRankByURL(options.calleeURLOrPriority) : 1;
  registeredLegacySolidComponents.set(name.toUpperCase(), { renderFunction, ...options });  // ← a plain Map.set
  Controls.define(name, { createInstance: FxsSolidComponent, priority: ranking >= 0 ? ranking : 0, … });
}
```

`registeredLegacySolidComponents` is a plain `Map` with no priority check whatsoever.
`priority` governs `Controls.define` only — and **every** caller passes the same
`FxsSolidComponent` class, so who wins that race does not matter.

**The practical conclusion:** to take over an element defined by `defineLegacyComponent`
(e.g. `production-chooser-item`), it is enough to call `defineLegacyComponent` with the same name
**later** — i.e. to have a higher `LoadOrder`. This works even when another mod has replaced
a whole game file through `ImportFiles`, because what counts is execution order, not whose file was
executed.

⚠️ This is **only a replacement**, not a wrap: the map is private to the module, and the game's
component files export nothing. You have to write the whole renderer.

⚠️ Symmetrically: a mod that loads after you will take it from you the same way.

⚠️ `options.attrs` has to list **every** `data-*` attribute you read. `FxsSolidComponent`
copies only the declared ones into the reactive store, and `Controls.define` observes only those.
A missing attribute simply never updates — without an error.

### ✅ `Controls.decorate` DOES WORK on an element from `defineLegacyComponent` — but it does not give you the internals

Established 2026-08-26 on the working `f1rstdan-cool-ui` mod: an element defined by
`defineLegacyComponent` is still an ordinary custom element from `Controls.define` (the
`FxsSolidComponent` class), so a decorator will attach and will get `Root` plus the four lifecycle hooks.

❗ **What it will NOT get: the internals as named properties.** In the old framework a component
exposed `this.container`, `this.iconElement`, `this.itemNameElement`… After the rewrite to Solid
those fields are gone — there is only the rendered DOM under `Root`.

**Hence two routes and a real choice between them:**

| Route | Gives you | Costs |
|---|---|---|
| **A** — re-registration via `defineLegacyComponent` | full control over the component | replacement only; you have to write the whole renderer |
| **B** — `Controls.decorate` + operating on the rendered DOM | additivity, coexistence with others | fights Solid's reactivity over everything Solid redraws |

⚠️ **A warning from practice:** `f1rstdan-cool-ui` built its compact production row layout
on route B, on named internal elements. When the game moved `production-chooser-item` to
`ui-next`, **the feature stopped working and has been broken for two releases** — the author says so
plainly in his `Changelog.md`. What **did** survive in the same mod is the quick-purchase
button, because it only reaches for `Root.firstElementChild`.

**The rule:** route B for purely additive things (injecting a button of your own), route A for
changing the layout.

## ✅ Importing from a mod WORKS — confirmed by a working Workshop mod

**2026-08-10, a CORRECTION to an earlier ⚠️:** the paths `/core/vendor/…` and `/core/ui-next/…`
**work** from within a mod. This is confirmed by the **Resource+** mod (`brads-assign-all-resources`,
Workshop 3756000777, by Brad), which begins exactly like this:

```js
import { onMount, onCleanup } from '/core/vendor/solid-js/dist/solid.js';
import { ComponentRegistry } from '/core/ui-next/services/component-registry.js';
import { useCommerceScreenContext } from '/base-standard/ui-next/screens/commerce/commerce-screen-model.js';
import { CommerceResourcesContainer } from '/base-standard/ui-next/screens/commerce/commerce-screen-resources-tab.js';
```

## ⭐ The pattern: wrapping the original (`Controls.decorate` for `ui-next`) ✅

An override with `overridePriority` **replaces** the component. To merely *extend* it,
you have to catch the original and call it at the end. The pattern from Resource+:

```js
// 1. importing the component FORCES the game's module to run before this line (ESM order),
//    so the game's registration has definitely already happened
import { CommerceResourcesContainer } from '/base-standard/ui-next/screens/commerce/commerce-screen-resources-tab.js';

// 2. the wrapped factory's `.factory` is the game's implementation
const originalFactory = CommerceResourcesContainer.factory;

function MyWrapper(props) {
    const model = useCommerceScreenContext();   // works: we are inside the Provider's tree
    onMount(() => { /* inject your own — idempotently, see below */ });
    onCleanup(() => { /* see the ⚠️ below: cleaning up here is NOT always allowed */ });
    return originalFactory(props);              // 3. the original renders normally
}

ComponentRegistry.register({
    name: 'CommerceResourcesContainer',
    overridePriority: 1100,
    createInstance: MyWrapper,
});
```

⚠️ **Here order DOES matter** — unlike with a pure override.
`Component.factory` has to already point at the game's implementation at the moment of the read.
That is guaranteed by **importing the game's module** (an imported module runs before the importer),
not by `<LoadOrder>`. Resource+ sets `LoadOrder` 1100 anyway — it does no harm.

⚠️ If two mods do the same thing, the higher `overridePriority` wins, and the
`originalFactory` chain behaves correctly **only if** both read `.factory`
at import time. Resource+ occupies `CommerceResourcesContainer` with priority **1100** —
on a collision you have to give more (and then it is our mod calling their wrapper, not the other way round).

## Two routes to the content: JSX by hand, or raw DOM

### Route A (like Resource+): DOM injection ✅ simpler

Resource+ **does not build JSX at all**. It returns the original and adds its own elements with
`document.createElement` / `querySelector` in `onMount`, drops its styles as a
`<style id="…">` into `document.head`, and handles synchronization with Solid's reactive tree
with a `MutationObserver` on `document.body`:

```js
observer = new MutationObserver(reconcileUI);
observer.observe(document.body, { childList: true, subtree: true });
```

Advantages: no compilation, full control. Drawbacks: dependence on the game's CSS classes and
`data-name` attributes (brittle across patches) and the cost of a `MutationObserver` on the whole `body`.

⚠️ **Cleaning up in `onCleanup` is a trap.** The container mounts several times during
a single visit to the screen, and a new mount's `onMount` can run **before** the previous one's
`onCleanup` — the cleanup then removes the just-injected element,
and a `stop*()` operating on a module singleton disconnects an observer that already belongs to
the new mount. On top of that, Solid can remove foreign nodes when it re-renders the container.

That is why the observer **stays permanently attached** and reinserts the element whenever it
disappears, and `stop*()` is called only on a full teardown of the mod. The full pattern:
**[quirk #51](14-quirks-and-gotchas.md)**.

### Route B: hand-written compiled JSX ⚠️ untested

A mod loaded through `<UIScripts>` is a plain ES module — nobody will compile JSX in it.
But the game's compiled code is plain JS, which can be written by hand:

```js
import { createComponent, createMemo, Show, For } from '/core/vendor/solid-js/dist/solid.js';
import { template, insert, effect, setAttribute } from '/core/vendor/solid-js/web/dist/web.js';
```

- `<Foo a={1}>bar</Foo>` → `createComponent(Foo, { a: 1, get children() { return "bar" } })`
- static HTML → `const _tmpl = template('<div class="flex"></div>')`, then `_tmpl()`
- a dynamic child → `insert(parent, () => expression)`
- a reactive prop has to be a **getter** (`get children() {…}`), otherwise it loses reactivity

The exports are available: `solid.js` gives, among others, `createComponent, createSignal, createMemo,
createEffect, createContext, useContext, batch, untrack, splitProps, mergeProps, onMount,
onCleanup, For, Show, Switch, Index, lazy`; `web.js` gives `template, insert, render, spread,
setAttribute, classList, style, effect, delegateEvents`.

## Things that surprise you

- **The CSS classes are Tailwind-style utilities**, but it is a Firaxis implementation of its own;
  escapes in class names are written as `-top-0\.5`, `hover\:scale-125`, `w-1\/4`.
- Pixel sizes are computed with `Layout.pixelsToScreenPixels(512)` — do not hardcode px.
- Texts: `<L10n.Compose text="LOC_…" />`, not `data-l10n-id`.
- BLP images: `url(blp:name)` in a style + a declaration in `images: [...]` at component
  registration (preload), or `useImageCache().registerImages(Symbol, [...])`.
- `createEngineEvent("EventName")` from `#core/ui-next/utilities/game-core-utilities.js`
  turns an engine event into a Solid signal — no need for a manual `engine.off`.
- The old `UpdateGate` from `core/ui/utilities/utilities-update-gate.js` is still used
  inside the new models to batch several events into one recomputation.
- The import aliases in the TypeScript sources (`#core/…`, `#base/…`) are only a build
  convention — in the compiled `.js` they are **relative paths** (`../../../../core/…`).

## Where to get the original sources

The `.js.map` files in `ui-next/` have full `sourcesContent` with **`.tsx`** files:

```bash
python tools/extract_ts.py "…/base-standard/ui-next/screens/commerce" /target/directory
```

⚠️ `extract_ts.py` blows up on maps for `*.scss.js` (the source name ends in
`?url`, which is an illegal file name on Windows). It prints `ERROR …` and carries on —
the remaining files extract correctly, so it is harmless.

## Mouse and keyboard input in `ui-next` ✅

**Date: 2026-08-10**, established while working on the first feature of the Commerce mod.

### The shape of the `engine-input` event

`core/ui/input/input-support.js` — `detail` has **exactly** this much:

```js
{ name, status, x, y, isTouch, isMouse }
```

❗ **There is no `shiftKey`/`ctrlKey`/`altKey`, and no `target`, in `detail`.** Modifiers have to be
taken from the DOM's `keydown`/`keyup` (those work — the game's own
`core/ui/external/js-spatial-navigation/spatial_navigation.js` reads
`evt.shiftKey` from them), or you have to declare an input action of your own.

✅ `detail.x` / `detail.y` are **DOM coordinates** — the game itself does
`document.elementsFromPoint(event.detail.x, event.detail.y)` with them
(`core/ui-next/components/drag-and-drop.js`). That is the most reliable way of establishing
what the player clicked on, because with gamepad input `event.target` is sometimes the *focused*
element rather than the one under the cursor.

### `mousebutton-right` ✅

- it is an **ordinary input action** with `EventType="All"` → you get `START` and `FINISH`
  (`InputActionStatuses`), not a single event;
- it has **no row in `InputContextConstraints`** → it works in every context,
  including `Shell` (i.e. on fullscreen screens);
- default gestures: `MOUSE_R` and — interestingly — `KEY_CONTROL+MOUSE_R` as a second
  index of the same action. So `MODIFIER+MOUSE` combinations are legal in `GestureData`.

### RMB closes the screen by default — you have to intercept it ⚠️

`InputEngineEvent.isCancelInput()` returns true for `cancel`, `keyboard-escape`
**and `mousebutton-right`**. On a `FINISH` of such an event, `core/ui-next/components/panel.js`
calls `ContextManager.pop(...)` — i.e. it closes the screen.

The panel listens through Solid's `"on:engine-input"`, i.e. with a **native, non-delegated**
listener on its own element (the bubbling phase). That is why a mod that wants to give RMB
a meaning of its own has to listen **in the capture phase on `window`**:

```js
window.addEventListener(InputEngineEventName, handler, true);   // ← true = capture
...
event.preventDefault();
event.stopPropagation();
event.stopImmediatePropagation();
```

Capture from `window` runs downwards **before** reaching the panel's element, so
stopping propagation there really does prevent the screen from closing. Stop only
the clicks you actually handle — the rest of RMB should keep closing the screen.

### Clicking a game component FROM CODE: `Activatable` does NOT react to a native click ❗✅

**Date: 2026-08-18.** The inverse of the `bindActivatable` trap (mod → an injected div sees
only native events). The other way round it is just as sealed: `Activatable`
(`core/ui-next/components/activatable.js`) listens **exclusively** through Solid's
`"on:engine-input"` and reacts to `mousebutton-left` / `touch-tap` / `keyboard-enter` /
a gamepad action button. `element.click()` and `dispatchEvent(new MouseEvent('click'))` **do
nothing**. You have to dispatch an engine event:

```js
import { InputEngineEvent } from '/core/ui/input/input-support.js';

element.dispatchEvent(new InputEngineEvent(
    'mousebutton-left',
    InputActionStatuses.FINISH,   // ⚠️ FINISH, not START
    0, 0,                          // x, y — not read by Activatable
    false,                         // isTouch
    true,                          // isMouse
));
```

⚠️ **`FINISH` only.** On `START` `Activatable` plays the press sound itself, and it calls `activate()`
only on `FINISH`. Sending both = an audible press for no reason.

⚠️ `InputActionStatuses` and `InputEngineEventName` are engine global enums — they are not
imported, just like `GameContext` or `Locale`.

This applies to everything built on `Activatable` — i.e. practically every clickable
`ui-next` element. Most useful with **tabs**: the `[data-name="TabListItem"]` element
is an `Activatable`, so that is exactly how you switch a tab from a mod (see
[26-commerce-screen.md](26-commerce-screen.md), "Reloading the screen").

### An order-proof priority pattern ✅

Instead of hardcoding `overridePriority` (and risking landing under somebody else's
wrapper or wiping it out):

```js
const originalFactory   = SomeComponent.factory;
const overridePriority  = (SomeComponent.overridePriority ?? 0) + 100;
```

We load before another mod → we have a low priority, it wraps us.
We load after it → we have a higher one, we wrap it. **In both cases both wrappers
work**, as long as each delegates to its own `originalFactory`.

### Modifier key state: `Input.isShiftDown()` ✅

**2026-08-10, A CORRECTION.** I wrote earlier here that modifiers have to be taken from the DOM's
`keydown`/`keyup`. **That does not work** — checked in game: listeners on `keydown`, `keyup`
and `mousedown` (capture phase, on `window`) **never once** reported Shift being pressed,
even though the player was holding it. This interface does not pass modifier state through
DOM keyboard events.

Instead the engine answers directly:

```js
Input.isShiftDown()   // → boolean, can be asked at any moment
```

The game itself uses it the same way — `core/ui/tooltips/tooltip-manager.js` and
`tooltip-controller.js` shorten the bubble's delay when Shift is held:

```js
const tooltipDelay = Input.isShiftDown() ? 1 : Configuration.getUser().tooltipDelay;
```

The advantages over a custom input action in `config/input.xml`: no configuration files,
no action group in the `shell` scope, no state to synchronize and none of the trap
with remapping a bare modifier (see the comment in the specialists mod's
`config/input.xml`). The drawback: the key is hardcoded, it cannot be rebound.

⚠️ Watch the order in which you search: there is no complete list of the `Input.*` API in any
documentation — the fastest way to extract it is from the game's code:

```bash
grep -rhoE "Input\.[a-zA-Z]+\(" --include=*.js core/ base-standard/ | sort -u
```

The same approach works for `Game.*`, `Players.*` and the rest of the engine's global objects.

#### ❓ A CORRECTION to the above: `Input.isShiftDown()` failed too

**2026-08-10, the same session.** Swapping the DOM route for `Input.isShiftDown()` **did not
fix the problem** — the function exists, it is called (confirmed by a log), but with
Shift held and RMB clicked on the Commerce screen it returns `false`.

The state of things for now: **I do not know a working way of reading modifier state
in the `Shell` context.** `Input.isShiftDown()` is the only key-state API in the whole
game's code (`grep -rhoE "Input\.(is|get)[A-Z][a-zA-Z]*\("`), and the game uses it only
in tooltips, which live mostly in the world context.

**A working hypothesis (to be confirmed):** the engine **does not fire the bare
`mousebutton-right` action while a modifier is held**, so with Shift+RMB nothing reaches
us and the question about Shift is never asked at the right moment. The circumstantial evidence is strong —
in `core/config/Input.xml` the game itself has to declare a **second gesture** for the same action:

```xml
<Row ActionId="mousebutton-right" Index="0" GestureType="KBMouse" GestureData="MOUSE_R"/>
<Row ActionId="mousebutton-right" Index="1" GestureType="KBMouse" GestureData="KEY_CONTROL+MOUSE_R"/>
```

If `MOUSE_R` also caught Ctrl+RMB, that second row would be pointless.

**The workaround we are testing:** a custom action with a `KEY_SHIFT+MOUSE_R` gesture
in `config/input.xml` (`shell` scope, as in [09-cookbook-ui-mod.md](09-cookbook-ui-mod.md)),
and listening for its name alongside `mousebutton-right`:

```xml
<InputActions>
    <Replace ActionId="my-mod-action" DeviceType="Keyboard"
             Name="LOC_…" Description="LOC_…_HELP" EventType="All" />
</InputActions>
<InputActionDefaultGestures>
    <Replace ActionId="my-mod-action" Index="0"
             GestureType="KBMouse" GestureData="KEY_SHIFT+MOUSE_R" />
</InputActionDefaultGestures>
```

Without rows in `InputContextConstraints` — `mousebutton-right` has none either and that is exactly
why it works on fullscreen screens (the `Shell` context).

⚠️ If it turns out that the engine does send **both** actions on Shift+RMB, the duplicate has to be
filtered out — for us that is done by a ~400 ms window after a bulk operation.

**A general conclusion for the future:** with input puzzles do not guess at the next API —
attach, for one test round, a listener that logs **every** `engine-input`
(name + status + `isShiftDown()` + coordinates) plus native DOM
`mousedown`/`contextmenu`/`keydown` events with their modifier flags. One round with data
is cheaper than three rounds of guessing.

#### ❌ CORRECTION #2: the "a modifier blocks the action" hypothesis is FALSE

**2026-08-10, another round.** Shift+RMB **does fire** the ordinary `mousebutton-right` action —
you can see it in the log (the mod performed a single resource un-assignment on it). The previous conclusion
from an empty log window was a misreading: there were simply fewer clicks there.

So the second `KEY_CONTROL+MOUSE_R` gesture on `mousebutton-right` in `core/config/Input.xml`
**does not prove** that bare `MOUSE_R` fails to catch the combination. The reason for its existence remains
unknown.

The state of knowledge after this round:

| what | result |
|---|---|
| DOM `keydown`/`keyup`/`mousedown` → `shiftKey` | ❌ never reports Shift as held |
| `Input.isShiftDown()` in the `Shell` context | ❌ returns `false` despite Shift being held |
| Shift+RMB fires `mousebutton-right` | ✅ yes |
| a custom action with a `KEY_SHIFT+MOUSE_R` gesture, without `InputContextConstraints` | ❌ did not fire |

**The current hypothesis:** a mod's action requires **explicit
`InputContextConstraints` rows**. The evidence: the specialists mod declares
`ContextId="World"` and `ContextId="Unit"` for its key and works; the base game's actions with
no rows at all (`mousebutton-right`) also work, so the "no rows = everywhere" rule
apparently does **not** cover actions added by mods.

To be tested: `<Replace ActionId="…" ContextId="Shell" />` (+ `Dual`).
The contexts to choose from — `core/config/Input.xml`, the `InputContexts` table:
**`Shell`, `World`, `Unit`, `Dual`**.

⚠️ To check at runtime whether the action registered at all (this is not visible
in `Modding.log` — action groups with the `shell` scope are not printed there):

```js
const actionId = Input.getActionIdByName('my-action');   // null/undefined = not there
```

⚠️ An input listener MUST filter out `InputActionStatuses.UPDATE` — `mousebutton-left`
in that status repeats every frame and eats the whole line budget before it reaches
the click under investigation.

#### ✅ THE SOLUTION: the engine does NOT send `mousebutton-right` while a modifier is held

**2026-08-10, established with a listener — the end of guessing.** This entry invalidates CORRECTION #2
and restores the original hypothesis. Raw data from `UI.log`:

```
plain RMB:
  dom mousedown button=2 shiftKey=false isShiftDown=false
  engine-input name=mousebutton-right status=START  isMouse=true
  dom mouseup   button=2 shiftKey=false
  engine-input name=mousebutton-right status=FINISH

Shift + RMB:
  dom mousedown button=2 shiftKey=true  isShiftDown=true
  dom mouseup   button=2 shiftKey=true  isShiftDown=true
  (no engine-input at all!)
```

**Neither `event.shiftKey` nor `Input.isShiftDown()` was ever broken** — both
return `true` at the right moment. What was broken was the event source: the code listened for
the `mousebutton-right` action, which with a modifier held **does not fire at all**.
That is why the question "is Shift held" was only ever asked on clicks without
Shift and always got `false`.

This also explains the second `KEY_CONTROL+MOUSE_R` gesture in the game's `core/config/Import.xml`:
without it Ctrl+RMB would be nothing to the engine.

**The practical conclusion — handle modifier-clicks with native DOM events:**

```js
window.addEventListener('mousedown', onDown, true);   // capture
window.addEventListener('mouseup',   onUp,   true);
// event.button === 2, event.shiftKey / ctrlKey / altKey — all available
```

The DOM's mouse events fire **in both cases** and carry the correct modifier state.
The order within a single click: `dom mousedown` → `engine START` →
`dom mouseup` → `engine FINISH`; do the work on `mouseup`.

⚠️ **You still have to handle the engine action** — but only in order to **suppress** it:
a plain RMB is `isCancelInput()` and the panel closes the whole screen on it. It arrives AFTER
the DOM `mouseup`, so a timestamp set when handling the click plus suppression of the
action within a ~400 ms window is enough.

⚠️ Suppress `mousedown` too if the click lands on your element — otherwise the screen's own
handlers will manage to select or start dragging the element under the cursor.

❗ **This applies to ALL mouse buttons, not just the right one.** `Activatable`
from `core/ui-next/components/activatable.js` — i.e. everything clickable in this framework —
fires `onActivate` precisely from an engine action:

```js
if (inputEvent.detail.name == "mousebutton-left" || … ) props.onActivate?.();
```

The result: **with a modifier held the whole screen stops reacting to clicks.**
For us it showed up as Shift+drag working while Shift+click did not —
because dragging runs on DOM events and clicking on an engine action.

If your mod gives a modifier a meaning, you have to **reproduce ordinary clicking**
for as long as it is held: on the DOM `mouseup` call the same model method that
that `Activatable`'s `onActivate` would have called.

⚠️ Distinguish a click from a drag — otherwise dropping after a drag will
perform the action a second time. It is enough to remember the `mousedown` position and reject a `mouseup`
that moved more than a few pixels.

❓ **Unresolved, but no longer needed:** a custom action with a `KEY_SHIFT+MOUSE_R` gesture
declared in `config/input.xml` (`shell` scope, `<Replace>`, `EventType="All"`,
with and without `InputContextConstraints` rows) **did not register** —
`Input.getActionIdByName()` returned `null`. The reason is unknown; the DOM route is
simpler anyway (no configuration files, no `shell` scope).

## Truncating text with an ellipsis ✅

This engine supports `text-overflow: ellipsis` — the game's theme even has a ready-made
`.truncate` class (`core/ui/themes/default/default.css`):

```css
.truncate { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
```

⚠️ Inside a flexbox `text-overflow` alone is not enough: a flex item does not by default
shrink below the width of its content. You need **`min-width: 0`** on the
truncated element **and on every flex ancestor of it**, otherwise instead of
an ellipsis the row will stretch or wrap.

An addendum: `coh-font-fit-mode` accepts `fit`, `shrink` and `none` — that is a different mechanism
(scaling the font to the box), not truncation.


---

## ⚠️ CSS grid: the game does NOT USE IT EVEN ONCE — do not build a layout on it ✅

```bash
grep -rho "display:\s*grid\|grid-template-columns" --include=*.css --include=*.js Base/
# zero hits in the whole game
```

Nothing says this renderer implements grid. Since Firaxis used it nowhere —
and in many places grid would be the obvious choice — we assume it is not there. We do everything
with flexbox.

### Aligning columns of numbers without grid: a ROW OF COLUMNS, not a column of rows

The problem: two rows of numbers, several items in each, with the plus signs to line up under one another.

```
One:        +18 🪙  +18 🌿
All:        +144 🪙 +144 🌿      ← the second item starts where the first one ended
```

Laid out **by rows** this cannot be aligned: the second item's width depends on
how much space the first took, so `+18` above `+144` throws everything after it off. Neither
`justify-content` nor `text-align` will touch it.

**The fix: invert the nesting.** Instead of two rows of N items — **N+1 columns
of 2 cells**:

```
[labels]     [item 1]      [item 2]
 One:         +18 🪙        +18 🌿
 All:         +144 🪙       +144 🌿
```

```css
.figures      { display: flex; flex-direction: row; align-self: center; }
.figures-col  { display: flex; flex-direction: column; flex: 0 0 auto; }
.figures-col + .figures-col { margin-left: 0.9rem; }
```

Each column is as wide as its widest cell, both cells start at its
left edge — **the alignment falls out of the construction**, without measuring and without guessed
widths. Nothing needs tuning when a bonus grows to four digits.

`align-self: center` on the container also gives you "a centered block with the text inside aligned
left" — in a column flex container that shrinks the element to its content and centers it.

⚠️ Classes that modify appearance ("this row is dimmed", "this one is bold")
move from the row onto **every cell** — the row stops existing as an element.

---

## ✅ Building good-looking tooltips — the game's building blocks instead of a bare box

`data-tooltip-content` gives you a **raw rectangle with text**. What the game shows for resources
(a class header with an icon, the resource in a frame, a name, a description) is a **Solid component**, not an attribute —
`base-standard/ui-next/tooltips/resource-tooltip.jsx`. It wraps its trigger in a `<Tooltip.Trigger>`,
so **you cannot ask for it from the DOM**: you have to hand it an element and put whatever it
returns in that element's place.

### ⭐ The shortcut that saves hours

**Find a game screen that draws what you want and read which components it reaches for.**
The whole set below came out of a single file — `commerce-screen-empire-tab.jsx` — because that is where the game
draws exactly the data (resources + leader faces) that I needed.

### The building blocks and their props ✅

| Component | Import | Props |
|---|---|---|
| `Tooltip` + `.Trigger` `.Content` `.Frame` `.Text` | `#core/ui-next/components/tooltip.jsx` | `showFiligrees` — `false` gives a simple corner style |
| `CardFrame` | `#core/ui-next/components/card-frame.jsx` | adds the `card-frame-bg` class |
| `PortraitIcon` | `#core/ui-next/components/portrait-icon.jsx` | `{ playerId, size }` — a leader's face in their colors |
| `FramedResource` | `#base/ui-next/components/framed-resource.jsx` | `{ size, ...resourceProps }` |
| `Icon` | `#core/ui-next/components/icon.jsx` | `{ name, isUrl }` — `name` is `url(blp:…)` when `isUrl` |
| `Divider.Vertical` / `.Horizontal` | `#core/ui-next/components/divider.jsx` | `{ margin }`, `{ useGradient }` |
| `L10n.Compose` | `#core/ui-next/components/l10n.jsx` | `{ text, args }` — localizes a key |
| `L10n.Stylize` | the same | localizes **and** interprets `[B]`, `[N]`, `[STYLE:…]` |
| `OrnateCard`, `FiligreeTitle.Plain`, `Activatable`, `ScrollArea` | `#core/ui-next/components/…` | the rest of the "ornate" set |

### ⚠️ Traps

1. **`resourceType` in a resource's props is a LOCALIZATION KEY**, not a resource type. The tooltip composes it
   through `LOC_RESOURCECLASS_TOOLTIP_NAME`. It is built as `LOC_${ResourceClassType}_NAME`.
2. **Icons are `url(blp:…)` strings**, not paths: `url(blp:${UI.getIconBLP(type)})`.
3. **`getResourcePropsFromDefinition` is not exported** — you have to copy it from
   `commerce-screen-model.ts`.
4. **A component has to be created under a Solid owner** (e.g. in a component's `onMount`). Called outside
   one it leaks its reactive scope.
5. **Without a build step there is no JSX** — write it in compiled form: `createComponent(Comp, {...,
   get children() { return … } })`. The getter matters: a plain value will be read once.
6. ⚠️⚠️ **Create a nested component INSIDE the parent's `children` getter, never earlier into a
   variable.** This is not cosmetic — it is exactly what JSX does:

   ```js
   // WRONG — the Frame is created OUTSIDE the Tooltip's context
   const frame = createComponent(Tooltip.Frame, {...});
   createComponent(Tooltip.Content, { get children() { return frame; } });

   // RIGHT — this is how <Tooltip.Content><Tooltip.Frame/></Tooltip.Content> compiles
   createComponent(Tooltip.Content, {
       get children() { return createComponent(Tooltip.Frame, {...}); },
   });
   ```

   The symptom of the first version: **the tooltip is drawn twice** and two copies follow the cursor —
   the frame mounts on its own *and* through `Content`. Nothing logs an error, so without this
   knowledge you look for the cause in CSS or in the old `data-tooltip-content`.
7. ⚠️⚠️ **When mixing with raw DOM use `insert()`, NOT `appendChild()`.** A Solid component does
   not return a node — it returns a **reactive getter** (or an array, or text). `appendChild` blows up
   on that:

   ```
   TypeError: Arguments[0] expect type : Node
   ```

   ```js
   import { insert } from '/core/vendor/solid-js/web/dist/web.js';

   const box = document.createElement('div');
   insert(box, createComponent(L10n.Stylize, { text: '…' }));   // right
   box.appendChild(createComponent(L10n.Stylize, { text: '…' })); // will throw
   ```

   `insert` is exactly what JSX uses underneath for `{expressions}` — it handles nodes,
   arrays, functions and primitives, and hooks up reactivity. A plain `element.appendChild(otherElement)`
   for raw DOM is still fine.
8. ⚠️ **Always leave a fallback text tooltip.** You are reaching for a component the game did not write for
   external use; a patch that moves it will leave the element **with no tooltip at all** — worse
   than the bare box you started from.

A working example (a class header + a resource card + a card for a leader with their face and settlements):
`better-commerce-screen-ui/ui/screen/resource-tooltip.js`.


## ⚠️ The game's DOM is Coherent, not a browser — the convenience methods are missing ✅

Civ VII's UI engine is **Coherent GT**, not Chromium. It looks like the DOM and mostly behaves like
the DOM, but **it lacks the `ChildNode`/`ParentNode` interface methods** that one reaches for
by reflex:

```js
node.replaceWith(other);   // TypeError: replaceWith is not a function
node.remove();             // ⚠️ this one does work — the mod has used it for a long time
node.isConnected           // do not rely on it; check node.parentNode
```

Instead, the classics of years past:

```js
node.parentNode.replaceChild(other, node);
node.parentNode.insertBefore(other, node);
```

The symptom is insidious, because **the code loads and nothing complains** — the exception only fires on
the call, so the function simply "does nothing". For me a GDP counter did not refresh on
every assignment and it looked like a bug in the event listeners; only a `warn` in the `catch` block
revealed `replaceWith is not a function`.

**The conclusion:** wrap such operations in `try/catch` with a `warn` straight away, not only once something does not
work — otherwise you will look for the cause somewhere else entirely.

## ❗✅ THERE ARE TWO TOOLTIP SYSTEMS — and only the new one can nest and lock

**Established 2026-08-27.** This is exactly the same trap as with screens (#33): the old and the new
system exist in parallel, both work, and the choice between them decides what is possible.

| | old | new |
|---|---|---|
| File | `core/ui/tooltips/tooltip-manager.js` | `core/ui-next/components/tooltip.js` (1116 lines) |
| Opting in | the `data-tooltip-style="name"` attribute | the Solid `Tooltip.Trigger` component |
| Registration | `TooltipManager.registerType(name, instance)` | `ComponentRegistry.register` |
| Nesting | ❗ **NONE** | ✅ `NestedTooltipContext` |
| Locking / interaction | ❗ **NONE** | ✅ autolock + `pointer-events-auto` |

### Why the old one CANNOT be interactive ✅

`cursorTooltipCheck()` runs **every frame** and reads `Cursor.target`:

```js
targetElement = Cursor.target instanceof HTMLElement ? Cursor.target : void 0;
const ttTypeName = RecursiveGetAttribute(targetElement, "data-tooltip-style") ?? "none";
if (ttTypeName == "none") { this.hideTooltips(); return; }
```

The moment the cursor leaves the element towards the bubble, the style resolves to `"none"`
and the bubble disappears. You cannot enter it with the mouse. `isToggledOn`, which looks like a lock,
concerns **touch only** (`ActionHandler.deviceType != InputDeviceType.Touch`).

### The new system — the API and the mechanism ✅

```js
import { Tooltip } from '/core/ui-next/components/tooltip.js';
Tooltip.Trigger   Tooltip.Content   Tooltip.Frame   Tooltip.Text   Tooltip.InspectHint
```

Each of them is used over 100 times in the base game. The lock:

```js
"pointer-events-auto": tooltipModel.isLocked(ctx.name) || IsTouchActive(),
"pointer-events-none": !tooltipModel.isLocked(ctx.name) && !IsTouchActive()
```

✅ **Autolock is a PLAYER setting in the game's options**: `Configuration.getUser().tooltipAutolockEnabled`
plus the time slider `Configuration.getUser().tooltipAutolock` (see `core/ui/options/options.js`).
So "the bubble locks after a couple of seconds of hovering" is a native feature, not something to write.
Once locked, the bubble gets `pointer-events-auto` and you can enter it, click and expand.

`tooltip-model.js` provides `lock()`, `unlock()`, `unlockAll()`, `isLocked(name)`, and `Escape`
(`inputEvent.isCancelInput()`) unlocks.

### The old -> new bridge for a tooltip ⚠️

`core/ui-next/components/tooltip-compat.js` defines `<fxs-tip>` via `defineLegacyComponent`
and renders a `Tooltip.Text` inside — proof that an old-framework element **can** host
a `ui-next` tooltip. ⚠️ But it is a bridge for keywords in text, not a general adapter:
the nested variant requires a Solid owner (`findOwnerFromElement`), and your own element has to be
built yourself.

**The practical conclusion:** a tooltip meant only to be read — the old `TooltipManager`, one
registration and an attribute. A tooltip that should lock, be clickable or contain another tooltip
— **must** be `ui-next` / Solid.

### ❗ A trap: the lock CANNOT be enabled until the tooltip has CHILDREN

**Established 2026-08-27 on my own mod.** A `ui-next` tooltip built correctly, and yet
it could not be locked with TAB — instead of "INSPECT" (`LOC_INSPECT_TOOLTIP`) the frame showed
only "HIDE (HOLD)".

The condition in `TooltipInspectHintComponent`:

```js
when: tooltipCount() > 0 && (!isLocked() || isTopLevelActiveAndLocked())
```

where `tooltipCount()` is `ctx.childTooltipList().length` — **the number of CHILD tooltips**.

❗ **A tooltip without a single nested tooltip offers no inspection**, because there is nothing to
enter — and therefore it cannot be locked or entered with the mouse. This is not a configuration
error: the game deliberately does not offer a lock where there is no deeper level.

**The conclusion:** if you want a tooltip to be freezable, it **has to contain at least one
`Tooltip` nested inside its `Tooltip.Content`**. Rendering all the content "flat" inside
a single frame takes that possibility away.

The lock is triggered by the `keyboard-inspect-tooltip` / `toggle-tooltip` action (TAB) on `FINISH`, on
a short press; holding it beyond `HIDE_TOOLTIPS_HOLD_THRESHOLD_MS` hides tooltips
instead of locking. `Escape` (`inputEvent.isCancelInput()`) unlocks.
