# 16 — The game's UI sources: the full TypeScript

**This is the most valuable find of the whole analysis.** Firaxis shipped the game with sourcemaps
containing the **complete original TypeScript code** of the interface.

## Facts ✅

- **1419** `.js.map` files in `Base\modules\`
- in a sample of 400 files, **394 contained `sourcesContent`** (~98.5%)
- the JavaScript **is not minified** — readable names, comments, `console.error`
- no `.d.ts` files — but the sourcemaps replace them (the types are in the TS code)

An example sourcemap header:
```json
{"version":3,"file":"framework.js",
 "sources":["../../../modules/core/ui/framework.ts"],
 "sourcesContent":["/**\n * @file framework.ts\n * @copyright 2021-2024, Firaxis Games\n ..."]}
```

## The tool ✅ (tested)

`C:\Users\najan\Documents\Civ7Modding\tools\extract_ts.py`

```bash
# a single file
python "C:\Users\najan\Documents\Civ7Modding\tools\extract_ts.py" ^
  "C:\Program Files (x86)\Steam\steamapps\common\Sid Meier's Civilization VII\Base\modules\core\ui\framework.js.map" ^
  "C:\Users\najan\Documents\Civ7Modding\ts-sources"

# a whole directory recursively (1419 maps -> the complete UI sources)
python "C:\Users\najan\Documents\Civ7Modding\tools\extract_ts.py" ^
  "C:\Program Files (x86)\Steam\steamapps\common\Sid Meier's Civilization VII\Base\modules" ^
  "C:\Users\najan\Documents\Civ7Modding\ts-sources"
```

⚠️ Python in this environment is natively Windows — pass **Windows paths**
(`C:\...`), not MSYS ones (`/c/...`), or you will get a `FileNotFoundError`.

The output is written in a `modules/core/ui/framework.ts` structure and so on.

## What the recovered code looks like ✅

```ts
/**
 * @file framework.ts
 * @copyright 2021-2024, Firaxis Games
 * @description A central storage location for singleton 'manager' instances
 *              and entry point into the UI framework.
 */
import type ContextManager from "#core/ui/context-manager/context-manager.js";
import type DialogManager from "#core/ui/dialog-box/manager-dialog-box.js";

let contextManager: typeof ContextManager | null = null;

const Framework = {
    get ContextManager(): typeof ContextManager {
        throw new Error("ContextManager must be set prior to using.");
    },
};

export function setContextManager(value: typeof ContextManager) { ... }
```

Full type annotations, JSDoc comments with descriptions, `#core/...` import aliases.

## What this is for as a modder

1. **Instead of API documentation** — you see method signatures and parameter types
2. **Names of components to decorate** — `Controls.define(...)` in the sources
3. **Event names** for `engine.on(...)`
4. **Understanding the contracts** — e.g. a component's lifecycle, the order of `onInitialize`/`onAttach`
5. **Copying patterns** — how Firaxis solved a similar problem

## Map of the UI sources ✅

| Location | JS files | Contents |
|---|---|---|
| `core/ui` | 373 | the framework, base components, dialogs, input, lenses, options |
| `core/ui-next` | 186 | the new components (Solid.js) |
| `core/vendor` | — | Solid.js, libraries |
| `base-standard/ui` | 601 | gameplay screens: cities, diplomacy, trees, units |
| `base-standard/ui-next` | 132 | new versions of screens (including plot-tooltip) |

Key files to start with:
- `core/ui/framework.js` — the framework's entry point
- `core/ui/component-support.js` — **`Controls.define` / `Controls.decorate`** (the contract is here)
- `core/ui/panel-support.js` — panels
- `base-standard/ui/app.js` — mounting the Solid.js application

## The two systems — how to tell them apart ✅

```js
// CLASSIC (ui/) — the Firaxis framework
class MyComponent extends Component {
    onInitialize() { this.Root... }
}
Controls.define('my-component', { createInstance: MyComponent });

// NEW (ui-next/) — Solid.js
import { render } from '../../core/vendor/solid-js/web/dist/web.js';
import { createComponent, Show } from '../../core/vendor/solid-js/dist/solid.js';
function App() { return [createComponent(PlotTooltip, {})]; }
render(App, document.getElementById('solidjs-root'));
```

The rendering engine: **Coherent Labs cohtml** (`core/ui/cohtml.js`) — HTML/CSS/JS,
but not a browser; parts of the DOM API behave differently.

## Suggested workflow

1. Extract everything once into `Civ7Modding\ts-sources\`
2. Open that directory in VS Code
3. Search it (`Ctrl+Shift+F`) instead of the game's files — faster and more readable
4. ⚠️ This is code **owned by Firaxis** — use it to learn and to write your own mod,
   do not redistribute the extracted sources
