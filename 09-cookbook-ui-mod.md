# 09 — Cookbook: an interface mod

The most popular category of Civ VII mods. Below is a complete, minimal UI mod
based on patterns from working Workshop mods ✅.

## Minimal structure

```
my-ui-mod\
├── my-ui-mod.modinfo
├── ui\
│   └── my-panel.js
└── text\
    └── en_us\InGameText.xml
```

## `.modinfo`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Mod id="my-ui-mod" version="1" xmlns="ModInfo">
    <Properties>
        <Name>LOC_MOD_MY_UI_NAME</Name>
        <Description>LOC_MOD_MY_UI_DESC</Description>
        <Authors>Your name</Authors>
        <Package>Mod</Package>
        <AffectsSavedGames>0</AffectsSavedGames>
    </Properties>
    <Dependencies>
        <Mod id="base-standard" title="LOC_MODULE_BASE_STANDARD_NAME" />
    </Dependencies>
    <ActionCriteria>
        <Criteria id="always"><AlwaysMet/></Criteria>
    </ActionCriteria>
    <ActionGroups>
        <ActionGroup id="my-ui-mod-game" scope="game" criteria="always">
            <Properties><LoadOrder>1000</LoadOrder></Properties>
            <Actions>
                <UIScripts>
                    <Item>ui/my-panel.js</Item>
                </UIScripts>
                <UpdateText>
                    <Item>text/en_us/InGameText.xml</Item>
                </UpdateText>
            </Actions>
        </ActionGroup>
    </ActionGroups>
</Mod>
```

⚠️ `AffectsSavedGames=0` — UI mods do **not** affect saves. All 47 Workshop mods
that declare this property have `0`. Set `1` only if you change gameplay
data that affects game state.

## Pattern 1 — decorating an existing component (recommended) ✅

You do not replace game files → maximum compatibility with other mods.

```js
// ui/my-panel.js
class MyDecorator {
    constructor(component) {
        this.component = component;
        component.myDecorator = this;
        this.onUpdateListener = this.onUpdate.bind(this);
    }
    beforeAttach() { }
    afterAttach() {
        // the component is already in the DOM — add elements here
        const info = document.createElement('div');
        info.classList.add('font-body-sm', 'text-accent-2');
        info.setAttribute('data-l10n-id', 'LOC_MY_UI_LABEL');
        this.component.Root.appendChild(info);

        engine.on('SomeGameEvent', this.onUpdateListener);
    }
    beforeDetach() {
        engine.off('SomeGameEvent', this.onUpdateListener);   // ALWAYS clean up
    }
    afterDetach() { }

    onUpdate(data) { /* ... */ }
}

Controls.decorate('panel-component-name', (component) => new MyDecorator(component));
```

**How to find the name of the component to decorate:**
```bash
grep -rn "Controls.define" "Base/modules/base-standard/ui/" | head -40
```

## Pattern 2 — your own component ✅

```js
class MyPanel extends Component {
    onInitialize() {
        this.Root.classList.add('flex', 'flex-col');
        this.Root.style.backgroundImage = 'url(fs://game/my-ui-mod/icons/background.png)';
    }
    onAttach() { }
    onDetach() { }
    onAttributeChanged(name, oldValue, newValue) { }
}
Controls.define('my-panel', { createInstance: MyPanel });
```

## Pattern 3 — a prototype patch (when there is no other way) ✅

```js
// run ONLY ONCE — a guard on a static field
class MyPatch {
    static patched;
    static apply(component) {
        const proto = Object.getPrototypeOf(component);
        if (MyPatch.patched === proto) return;
        MyPatch.patched = proto;

        const original = proto.update;
        proto.update = function(...args) {
            const result = original.apply(this, args);   // keep the original behavior
            /* your extra logic */
            return result;
        };
    }
}
```

## Reading game data ✅

`GameInfo` is the most used API in mods (676 occurrences) — it is direct
access to the database from [02-database.md](02-database.md):

```js
const info = GameInfo.Constructibles.lookup(item.type);
if (info?.ConstructibleClass === 'WONDER') { /* ... */ }

for (const row of GameInfo.Units) { console.log(row.UnitType, row.BaseMoves); }
```

Current game state:
```js
const player = Players.get(GameContext.localPlayerID);
const cities = player?.Cities?.getCities() ?? [];
const plot   = GameplayMap.getPlotIndexFromXY(x, y);
```

## Your own map layer (lens) ✅

```js
import LensManager from '/core/ui/lenses/lens-manager.js';

class MyLayer {
    initLayer() { }
    applyLayer() { this.visible = true;  /* drawing */ }
    removeLayer() { this.visible = false; }
}
LensManager.registerLensLayer('my-layer', new MyLayer());
```

## CSS styles ✅

```xml
<ImportFiles><Item>ui/my-style.css</Item></ImportFiles>
```
```js
Controls.loadStyle('fs://game/my-ui-mod/ui/my-style.css');
```
The game uses utility-style classes (`flex`, `flex-col`, `size-24`, `bg-cover`,
`font-body-sm`, `text-accent-2`) ⚠️ — it is safest to copy classes from the game's existing
`.css`/`.js` files rather than guess.

## Texts and translations

```xml
<!-- text/en_us/InGameText.xml -->
<Database><EnglishText>
    <Row Tag="LOC_MY_UI_LABEL"><Text>My label</Text></Row>
</EnglishText></Database>
```
In the DOM use `data-l10n-id="LOC_MY_UI_LABEL"` instead of hardcoding the text —
the game will substitute the translation itself.

## Mod options in the main menu ✅

The game has no official API for mod options, but the community has settled on a working pattern
(source: `bz-map-trix`, verified against `core/ui/options/`).

### 1. Your own "Mods" tab

The `Mods` category does not exist in the game — mods **add** it. Use `??=` and the identifier
`"mods"` so that you **share a single tab** with other mods instead of creating a duplicate:

```js
import '/core/ui/options/screen-options.js';   // must load first
import { CategoryType, Options, OptionType } from '/core/ui/options/model-options.js';
import { CategoryData } from '/core/ui/options/options-helpers.js';

CategoryType["Mods"] = "mods";
CategoryData[CategoryType.Mods] ??= {
    title: "LOC_UI_CONTENT_MGR_SUBTITLE",
    description: "LOC_UI_CONTENT_MGR_SUBTITLE_DESCRIPTION",
};
```

### 2. Registering an option

```js
Options.addInitCallback(() => {
    Options.addOption({
        category: CategoryType.Mods,
        group: "my_mod",
        type: OptionType.Checkbox,
        id: "my-mod-something",
        initListener:   (info)         => info.currentValue = MyOptions.something,
        updateListener: (_info, value) => MyOptions.something = value,
        label: "LOC_OPTIONS_MY_MOD_SOMETHING",
        description: "LOC_OPTIONS_MY_MOD_SOMETHING_DESCRIPTION",
    });
});
```
⚠️ **The group name requires a text of its own.** `group: "najane_mods"` makes the UI look for
the key `LOC_OPTIONS_GROUP_NAJANE_MODS` (group name → uppercase). Without that row
in `UpdateText` you will see the raw key in the options instead of a heading.

`OptionType` ✅: `Editor`, `Checkbox`, `Dropdown`, `Slider`, `Stepper`, `Switch`.
For `Dropdown` there is also `dropdownItems: [{label, value}, ...]`, and `initListener`
sets `info.selectedItemIndex`.

⚠️ `addInitCallback` **throws** if the options have already been initialized —
register at script load time, not later.

### 3. Saving the value ✅

```js
UI.setOption("user", "Mod", `${MOD_ID}.${optionID}`, value);
Configuration.getUser().saveCheckpoint();      // without this it will not survive a restart
const value = UI.getOption("user", "Mod", `${MOD_ID}.${optionID}`);
```
The values are **numbers** (`Number(true)` → 1), not booleans.
The community mirrors the save in `localStorage` (key `modSettings`) as a safeguard.

### 4. ⚠️ Register options in BOTH scopes

The options screen exists both in the main menu and in game:
```xml
<ActionGroup id="my-mod-game" scope="game" criteria="always">
    <Actions><UIScripts><Item>ui/options/my-options.js</Item>…</UIScripts></Actions>
</ActionGroup>
<ActionGroup id="my-mod-shell" scope="shell" criteria="always">
    <Actions>
        <UpdateText><Item>text/en_us/InGameText.xml</Item></UpdateText>
        <UIScripts><Item>ui/options/my-options.js</Item></UIScripts>
    </Actions>
</ActionGroup>
```
Without the `shell` group the options **will not be in the main menu**. Remember the `UpdateText`
in `shell` too — otherwise you will see `LOC_*` keys instead of labels.

## Your own remappable key in the options ✅ (the complete recipe)

Verified in practice. **Four** pieces are needed for it to work — missing any one of them
gives silence without an error, so it is easy to get stuck.

### 1. Declaring the action — `config/input.xml`

```xml
<Database>
    <InputActions>
        <Replace ActionId="my-mod-action" DeviceType="Keyboard"
                 Name="LOC_INPUT_MY_MOD_ACTION" Description="LOC_INPUT_MY_MOD_ACTION_HELP"
                 EventType="All" />
    </InputActions>
    <InputActionDefaultGestures>
        <Replace ActionId="my-mod-action" Index="0"
                 GestureType="KBMouse" GestureData="KEY_TAB" />
    </InputActionDefaultGestures>
    <InputContextConstraints>
        <Replace ActionId="my-mod-action" ContextId="World" />
        <Replace ActionId="my-mod-action" ContextId="Unit" />
    </InputContextConstraints>
</Database>
```
- `EventType="All"` → you get `START`/`FINISH`, i.e. **hold detection**
- ❗ hook it up **only** in `scope="shell"` (see [14](14-quirks-and-gotchas.md) #28)

### 2. Showing the action in the remapping screen ❗

The screen **does not list registered actions** — it iterates over a hardcoded array
`KEYS_TO_ADD` in `core/ui/options/editors/editor-keyboard-mapping.js`. Without this step
the action works but is **invisible in the settings**:

```js
class MyEditorPatch {
    static patched = null;
    constructor(component) { this.component = component; component.myPatch = this;
        this.patchPrototype(Object.getPrototypeOf(component)); }
    patchPrototype(proto) {
        if (MyEditorPatch.patched) return;          // shared prototype - once only
        const original = proto.addActionsForContext;
        MyEditorPatch.patched = { proto, original };
        proto.addActionsForContext = function (...args) {
            const r = original.apply(this, args);
            return this.myPatch?.afterAddActionsForContext(...args) ?? r;
        };
    }
    beforeAttach() {} afterAttach() {} beforeDetach() {} afterDetach() {}
    afterAddActionsForContext(ctx) {
        const id = Input.getActionIdByName("my-mod-action");
        if (!id || this.component.mappingDataMap.has(id)) return;
        this.component.actionContainer.appendChild(this.component.createActionEntry(id, ctx));
    }
}
Controls.decorate('editor-keyboard-mapping', (c) => new MyEditorPatch(c));
```
⚠️ Hook this file up in **both** scopes (`shell` and `game`) — the options screen is in both.

### 3. Hold detection

```js
import { InputEngineEventName } from '/core/ui/input/input-support.js';

let held = false;
window.addEventListener(InputEngineEventName, (e) => {
    if (e.detail?.name !== "my-mod-action") return;
    if (e.detail.status === InputActionStatuses.START) held = true;
    else if (e.detail.status === InputActionStatuses.FINISH) held = false;
});
// no FINISH on a mode change / focus loss - reset
window.addEventListener(InterfaceModeChangedEventName, () => held = false);
window.addEventListener("blur", () => held = false);
```

### 4. Displaying the current binding in the UI

```js
const actionId = Input.getActionIdByName("my-mod-action");       // ❗ a NUMERIC id
const deviceType = Input.getActionDeviceType(actionId);
const locKey = Input.getGestureDisplayString(actionId, 0, deviceType, InputContext.ALL);
const label = Locale.compose(locKey);                            // ❗ returns a KEY, not text
```
⚠️ Two traps at once: passing the **name** instead of the id returns nothing without an error,
and the result is a localization key (`LOC_OPTIONS_KEY_TAB`) that still has to be composed.
To insert it use `Locale.compose(hintKey, label)` + `textContent` — `data-l10n-id`
will not accept an argument computed at runtime.

### ⚠️ Do not use a bare modifier as the default

`KEY_SHIFT`/`KEY_CONTROL` **work** as a default value from XML, but the engine's
gesture recorder (`Input.beginRecordingGestures`) treats a modifier as the start of a
combination. The consequences:
- `KEY_CONTROL` was displayed on the screen as **"unbound"**
- after remapping away from `KEY_SHIFT` **you cannot get back to it** — only a full
  `Input.restoreDefault()`, which resets the bindings of **all** mods

Use an ordinary key (`KEY_TAB` was free both in the base game and in all 49 mods).

## Modifier keys (Shift/Ctrl/Alt) ❗

> ⚠️ **CORRECTION.** I previously wrote here that `window.addEventListener("keydown"/"keyup")` was enough.
> **In practice that did not work** — holding Shift produced no reaction at all.

**Why:** the game **does not route raw keyboard state to the DOM**. Its own input system
(`InputEngineEvent`) carries only **named actions**, and only discrete presses:
```js
detail: { name, status, x, y, isTouch, isMouse }   // no information about modifiers whatsoever
```
The `InputActions` / `Input.bind` system will not help either — it handles "an action key was pressed",
not "a modifier is being held right now".

**What does work:** **mouse** events carry `shiftKey` (confirmed: the `shift-que` mod reads
`event.shiftKey` from `click`). So sample the state from **all** available events:

```js
const SAMPLED = ["keydown","keyup","mousemove","mousedown","mouseup","mouseover","click","wheel"];
let shiftHeld = false;
function update(e) {
    if (!!e.shiftKey === shiftHeld) return;
    shiftHeld = !!e.shiftKey;
    /* redraw */
}
for (const t of SAMPLED) document.addEventListener(t, update, true);  // capture!
window.addEventListener("blur", () => { shiftHeld = false; /* redraw */ });
```

- **`document` + the capture phase (`true`)** — so that you see the event before
  anything stops propagation
- **read `event.shiftKey`**, not `event.key === "Shift"` — the state is also corrected
  on key combinations
- **reset on focus loss and on an interface mode change** — a missing `keyup`
  would leave the view stuck in the Shift state
- in practice `mousemove` saves you: the user is moving the cursor anyway, so the state is fresh
  even if keyboard events never arrived at all

⚠️ With a solution like this, **log the first few state changes** (`console.error`, because
`console.log` does not reach `UI.log`) — otherwise you will not be able to tell "it does not work" from
"it works, but the redraw is broken".

## Performance ✅ (lessons from a real mod)

### 1. Do not compute aggregate data inside a function called per element

The classic trap: a function that draws **one** tile calls a function that scans
**all** tiles. That is quadratic complexity, and it multiplies engine calls along the way:

```js
// WRONG - for each of n tiles it scans n tiles
layer.updateSpecialistPlot = function (info) {
    const baseline = computeBaseline();   // iterates over all plots!
    ...
};
```
With 30 plots that is ~900 `Districts.getAtLocation()` calls per redraw.

**The fix:** a cache in the module + explicit invalidation on the events that
actually change the inputs:
```js
let cachedBaseline = null;
export function invalidateCaches() { cachedBaseline = null; /* + per-plot maps */ }
window.addEventListener(PlotWorkersUpdatedEventName, invalidateCaches);
window.addEventListener(InterfaceModeChangedEventName, invalidateCaches);
```
⚠️ Register the invalidation **in the data module**, not in the consumers — then every
consumer gets fresh data automatically and it is impossible to forget.
⚠️ The data module is imported earlier than the UI components, so its listener
runs **before** the drawing listeners. That is the required order.

### 2. Redraw only what changed

A full layer redraw is expensive — `realizeGrowthPlots()` walks the city's whole
growth domain, and when there is none, it **scans the whole map** (`width × height`
with a `getRevealedState` on every plot).

Hooking such a redraw to a hover event (which fires on every mouse move) is the worst
possible case. Since a hover changes the appearance of **two** plots, redraw two:
```js
function redrawPlot(plotIndex) {
    const info = PlotWorkersManager.allWorkerPlots.find((p) => p.PlotIndex === plotIndex);
    if (!info) return;
    layer.yieldVisualizer.clearPlot(GameplayMap.getLocationFromIndex(plotIndex));
    layer.updateSpecialistPlot(info);
}
```
⚠️ `YieldChangeVisualizer.clearPlot()` takes a **location**, while `SpriteGrid.clearPlot()`
in another layer takes a **plot index**. Easy to confuse.

### 3. Check whether a redraw is needed at all

The cheapest work is the work not done. If an option means a given event changes
nothing on screen — return immediately:
```js
if (NajaneOptions.alwaysShowNegatives || isOriginalDisplayActive()) return;
```

### 4. Turn diagnostics off before release ❗

`console.error` (the only channel that reaches `UI.log` — see [19](19-workflow-and-debugging.md))
in a released mod litters the player's log with entries that look like errors. Keep a flag:
```js
export const DIAGNOSTICS = false;   // enable only for the duration of an investigation
```
⚠️ This actually shipped in version 1.0 on the Workshop — easy to overlook.

## Common mistakes

| Symptom | Cause |
|---|---|
| The decorator does nothing | the component was created **before** registration → raise `LoadOrder` |
| Leaks / duplicated handlers | no `engine.off` in `beforeDetach` |
| A method wrapped several times | no "patch once" guard on a prototype patch |
| You see `LOC_...` instead of text | a missing row in `UpdateText` or a typo in the tag |
| The mod breaks after a game patch | `ImportFiles`/`ReplaceUIScript` used instead of a decorator |

## Mod options: category, group and where the headings come from ✅

**2026-08-10.** The options screen has two levels of nesting and **both titles come from
somewhere other than the mod's code**:

```js
Options.addOption({
    category: CategoryType.Mods,   // the tab
    group: 'najane_commerce',      // the section heading inside the tab
    …
});
```

**The tab** — `CategoryData[CategoryType.Mods]` with a title and description. The "Mods" category does
not exist in the base game; you add it with `CategoryType["Mods"] = "mods"` and
`CategoryData[...] ??= {…}`. The `??=` matters: several community mods do the same thing
and thanks to it they **share a single tab** instead of multiplying separate ones.

**The section heading** — computed from the group identifier by
`GetGroupLocKey` in `core/ui/options/options-helpers.js`:

```js
const suffix = group.toUpperCase();
return `LOC_OPTIONS_GROUP_${suffix}`;
```

So `group: 'najane_commerce'` requires the key **`LOC_OPTIONS_GROUP_NAJANE_COMMERCE`**
in the mod's text files. Without it you will see the raw key.

❗ **Do not use someone else's group identifier.** Sharing a tab is fine, sharing a section is not:
a group is named after the title of its owner, so an option dropped into `najane_mods`
(the group of the "Common Specialists Yields" mod) looks like it belongs to that mod. Every mod
should have its own group named after itself.

## A UI mod file layout that holds up as it grows ✅

After `better-commerce-screen-ui` grew to ~5500 lines across 26 files sitting flat
in `ui/`, a rebuild produced layers that depend **in one direction only**:

```
support/  ← knows nothing about the game or the mod (logging, element creation, CSS injection)
engine/   ← talking to the game: PlayerOperations, waiting for events, the age, input modifiers
model/    ← reading the screen's data (and the same shapes reconstructed without the screen)
planner/  ← decisions: what goes where
screen/   ← the DOM the mod adds to the screen
options/  ← imports nothing of ours; a leaf
```

The rule: a module may import from its own directory or **from the left**, never from the right.

**What this is for, concretely:** automatic assignment works while the screen is closed.
When the planner imported the "factory first" checkbox in order to read the setting's value,
loading the assignment engine dragged the whole button bar in with it. Separating
the **setting** (`planner/factory-first-setting.js`) from the **control** (`screen/factory-first.js`)
removed the dependency. The same treatment applied to `isFactoryAge()` — the function lived next to
the factory tab while three modules from two layers asked for it; it moved to
`engine/age.js`.

**Signs that it is time for such a split:**

- one file exceeds ~1000 lines and contains `//#region` (those regions are ready-made cut lines);
- the same function exists in 2–3 copies — for us `canStart` for `ASSIGN_RESOURCE` was in
  three modules, and one of them declared in its header that it was "the only place";
- the same name means two different things — `allAvailableResources` returned, in one
  module, resources from the model and, in another, only the rendered ones, in DOM order;
- a low layer imports a high one (planner → screen).

**Verification after the move, without launching the game** — three scripts, each of which caught a real bug:

1. `node --check` on every file (syntax);
2. for every `import { a, b } from './x.js'` check that `x.js` really exports `a` and `b`;
3. list the identifiers used in a file that are neither defined nor imported in it — this
   catches functions and constants left on the other side of the cut (for us
   `settlementYieldTotal` and `modifierApplies`). Watch out for false positives: `rgba(`,
   `url(`, `scale(` from CSS inside template literals, and callback parameter names.

Plus an import graph with cycle detection and a check on the direction of the layers.
