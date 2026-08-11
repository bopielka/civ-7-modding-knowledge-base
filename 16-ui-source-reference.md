# 16 — Źródła UI gry: pełny TypeScript

**To jest najcenniejsze odkrycie z całej analizy.** Firaxis wysłał grę z sourcemapami
zawierającymi **kompletny oryginalny kod TypeScript** interfejsu.

## Fakty ✅

- **1419** plików `.js.map` w `Base\modules\`
- w próbce 400 plików **394 zawierały `sourcesContent`** (~98,5%)
- JavaScript **nie jest zminifikowany** — czytelne nazwy, komentarze, `console.error`
- brak plików `.d.ts` — ale sourcemapy je zastępują (typy są w kodzie TS)

Przykład nagłówka sourcemapy:
```json
{"version":3,"file":"framework.js",
 "sources":["../../../modules/core/ui/framework.ts"],
 "sourcesContent":["/**\n * @file framework.ts\n * @copyright 2021-2024, Firaxis Games\n ..."]}
```

## Narzędzie ✅ (przetestowane)

`C:\Users\najan\Documents\Civ7Modding\tools\extract_ts.py`

```bash
# pojedynczy plik
python "C:\Users\najan\Documents\Civ7Modding\tools\extract_ts.py" ^
  "C:\Program Files (x86)\Steam\steamapps\common\Sid Meier's Civilization VII\Base\modules\core\ui\framework.js.map" ^
  "C:\Users\najan\Documents\Civ7Modding\ts-sources"

# cały katalog rekurencyjnie (1419 map -> pełne źródła UI)
python "C:\Users\najan\Documents\Civ7Modding\tools\extract_ts.py" ^
  "C:\Program Files (x86)\Steam\steamapps\common\Sid Meier's Civilization VII\Base\modules" ^
  "C:\Users\najan\Documents\Civ7Modding\ts-sources"
```

⚠️ Python w tym środowisku jest natywnie windowsowy — podawaj **ścieżki Windows**
(`C:\...`), nie MSYS-owe (`/c/...`), inaczej dostaniesz `FileNotFoundError`.

Wynik zapisuje się w strukturze `modules/core/ui/framework.ts` itd.

## Jak wygląda odzyskany kod ✅

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

Pełne adnotacje typów, komentarze JSDoc z opisami, aliasy importów `#core/...`.

## Po co to modderowi

1. **Zamiast dokumentacji API** — widzisz sygnatury metod i typy parametrów
2. **Nazwy komponentów do dekoracji** — `Controls.define(...)` w źródłach
3. **Nazwy zdarzeń** do `engine.on(...)`
4. **Zrozumienie kontraktów** — np. cykl życia komponentu, kolejność `onInitialize`/`onAttach`
5. **Kopiowanie wzorców** — jak Firaxis rozwiązał podobny problem

## Mapa źródeł UI ✅

| Lokalizacja | Plików JS | Zawartość |
|---|---|---|
| `core/ui` | 373 | framework, komponenty bazowe, dialogi, input, lensy, opcje |
| `core/ui-next` | 186 | nowe komponenty (Solid.js) |
| `core/vendor` | — | Solid.js, biblioteki |
| `base-standard/ui` | 601 | ekrany rozgrywki: miasta, dyplomacja, drzewa, jednostki |
| `base-standard/ui-next` | 132 | nowe wersje ekranów (m.in. plot-tooltip) |

Kluczowe pliki na start:
- `core/ui/framework.js` — punkt wejścia frameworka
- `core/ui/component-support.js` — **`Controls.define` / `Controls.decorate`** (tu jest kontrakt)
- `core/ui/panel-support.js` — panele
- `base-standard/ui/app.js` — montowanie aplikacji Solid.js

## Dwa systemy — jak rozpoznać ✅

```js
// KLASYCZNY (ui/) — framework Firaxis
class MojKomponent extends Component {
    onInitialize() { this.Root... }
}
Controls.define('moj-komponent', { createInstance: MojKomponent });

// NOWY (ui-next/) — Solid.js
import { render } from '../../core/vendor/solid-js/web/dist/web.js';
import { createComponent, Show } from '../../core/vendor/solid-js/dist/solid.js';
function App() { return [createComponent(PlotTooltip, {})]; }
render(App, document.getElementById('solidjs-root'));
```

Silnik renderujący: **Coherent Labs cohtml** (`core/ui/cohtml.js`) — HTML/CSS/JS,
ale nie przeglądarka; część API DOM zachowuje się inaczej.

## Sugerowany workflow

1. Wyekstrahuj całość raz do `Civ7Modding\ts-sources\`
2. Otwórz ten katalog w VS Code
3. Szukaj po nim (`Ctrl+Shift+F`) zamiast po plikach gry — szybciej i czytelniej
4. ⚠️ To kod **własności Firaxis** — używaj do nauki i pisania własnego moda,
   nie redystrybuuj wyekstrahowanych źródeł
