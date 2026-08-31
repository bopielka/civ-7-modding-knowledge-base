# 28. Ekran miasta (City Screen) — mapa dla modów

**Data ustalenia: 2026-08-26.** Wszystko poniżej ✅ przeczytane w plikach gry oraz w modzie
`bz-city-hall` 2.7.3. Baza dla moda „Better City UI by Najane".

To wszystko, co widzi gracz po kliknięciu w osadę: lista produkcji, panel szczegółów miasta,
panel wzrostu/specjalistów, tryby stawiania budynków i kupowania kafelków, oraz nakładki na
mapie, które im towarzyszą.

## ⚠️ NAJWAŻNIEJSZE: to **stary** framework, nie `ui-next`

Odwrotnie niż ekran Handlu ([26](26-commerce-screen.md)). Wszystkie cztery panele to
`Controls.define` w starym frameworku `ui/`, więc **`Controls.decorate` tutaj DZIAŁA**.
To jest ta jedna różnica, która decyduje o architekturze każdego moda na ten ekran.

Wyjątek: **wiersze listy produkcji**. `<production-chooser-item>` i
`<production-chooser-unique-quarter-item>` to komponenty Solid zarejestrowane przez
`defineLegacyComponent(...)` i osadzone w starym DOM jako custom elementy. Ekran jest więc
**hybrydą**: stary panel z dziećmi w Solid.

✅ **Jak przejąć taki wiersz:** zawołać `defineLegacyComponent` z tą samą nazwą **później** —
funkcja renderująca ląduje w zwykłej `Map` bez sprawdzania priorytetu, więc wygrywa ostatni
zapis, a nie najwyższy priorytet. Pełne wyjaśnienie i pułapki (m.in. że trzeba zadeklarować
**każdy** czytany atrybut `data-*`) w [25](25-ui-next-solidjs.md), sekcja „KOREKTA 2026-08-26".
Działa nawet wtedy, gdy inny mod podmienił cały plik gry przez `ImportFiles`.

## Gdzie panele siedzą ✅

`base-standard/ui/root-game.html`, slot `top-center`:

```html
<fxs-hslot class="flex-auto panel-production-slot">
    <panel-production-chooser></panel-production-chooser>
    <panel-city-capture-chooser></panel-city-capture-chooser>
</fxs-hslot>
<fxs-vslot class="flex self-end items-start">
    <div class="panel-city-details-slot"></div>
</fxs-vslot>
```

- `.panel-production-slot` i `.panel-city-details-slot` to **rodzeństwo**, obecne w dokumencie od
  startu. `panel-production-chooser` sięga po nie przez
  `MustGetElement(".panel-city-details-slot", document)`.
- ⚠️ `panel-city-details` **nie jest w HTML** — tworzy go chooser produkcji na żądanie i to on
  wysyła `city-details-closed` przy zamknięciu.
- Wspólny przodek obu paneli (przydatny przy obsłudze fokusu pada) to
  `.panel-production-slot.parentNode`.

## Cztery panele ✅

| Element | Plik | Linie | Klasa |
|---|---|---|---|
| `panel-production-chooser` | `ui/production-chooser/panel-production-chooser.js` | 1441 | `ProductionChooserScreen extends Panel` |
| `panel-city-details` | `ui/city-details/panel-city-details.js` | 1474 | `PanelCityDetails extends Panel` |
| `panel-place-population` | `ui/place-population/panel-place-population.js` | — | `PlacePopulationPanel` |
| `last-production-section` | `ui/production-chooser/last-production-section.js` | — | — |

### `panel-production-chooser`

Właściciel nagłówka z nazwą osady, strzałek poprzednie/następne miasto, paska zakładek
**Produkcja / Zakup**, akordeonów kategorii, checkboxa **Pokaż ukryte** i przycisku zamiany w
miasto.

- `isPurchase` to **para getter/setter na prototypie** — dlatego da się ją opatchować (wzorzec
  niżej).
- `productionCategorySlots` — po jednym slocie na `ProductionPanelCategory`
  (`buildings`, `units`, `projects`, `wonders`).
- `doOrConfirmConstruction(category, type, cb)` — **jeden lejek** dla każdego rozkazu budowy i
  każdego zakupu.
- Lista pozycji nie powstaje tutaj: panel woła `GetProductionItems(...)` z
  `production-chooser-helpers.js`, a wynik zapisuje na elementach `<production-chooser-item>`
  jako atrybuty `data-*`. **Ta ściana atrybutów jest szwem** — komponent Solid renderuje wyłącznie
  to, co panel mu w atrybucie poda.

### `panel-city-details`

Trzy zakładki, id z enuma `cityDetailTabID`:

| id | Metoda | Zawiera |
|---|---|---|
| `city-details-tab-growth` | `renderGrowthSlot()` | żywność, populacja, wzrost |
| `city-details-tab-buildings` | `renderBuildingSlot()` | budynki, ulepszenia, cuda |
| `city-details-tab-yields` | `renderYieldsSlot()` | rozbicie dochodów |

✅ **Zestaw zakładek to atrybut**: `tabBar.setAttribute("tab-items", JSON.stringify(...))`.
Dodanie własnej zakładki nie wymaga podmiany klasy — patrz wzorzec „dodanie zakładki" niżej.

Metody prototypu, które warto znać (wszystkie podmienialne): `renderBuildingSlot`,
`renderYieldsSlot`, `addConstructibleData`, `addDistrictData`, `addImprovementEntry`,
`addProductionTooltip`, `addWarehouseBreakdownTooltip`, `onCollapseAllSection`,
`onCollapseImprovementSection`, `update`, `render`.

Dane: `model-city-details.js` → `CityDetailsModel`, publikowany jako `g_CityDetails`
(`engine.createJSModel`), zmiany ogłaszane zdarzeniem okna `update-city-details`.

### `panel-place-population`

Panel przy stawianiu specjalisty i przy rozbudowie miasta. Model `PlacePopulationModel` daje
`hoveredPlotIndex`, `hoveredPlotWorkerIndex`, `hoveredPlotWorkerPlacementInfo` oraz per-yield
`CurrentYields`/`NextYields` i `CurrentMaintenance`/`NextMaintenance`.

⚠️ **Utrzymanie specjalisty jest wyłącznie w `*Maintenance`, nigdy w `*Yields`** (już opisane w
[14](14-quirks-and-gotchas.md)).

⚠️ Panel ma kilka kontenerów — `improvementMinimizedContainer`, `improvementMaximizedContainer`,
`specialistMinimizedContainer`, `specialistMaximizedContainer`, plus `subsystemFrame` i
`placeSpecialistFrame` — a **gra chowa część z nich zależnie od trybu interfejsu**. DOM wstrzyknięty
w niewłaściwy jest po prostu niewidoczny. City Hall pisze do wszystkich czterech kontenerów.

## Tryby interfejsu i menedżery ✅

| Obiekt | Plik | Co trzyma |
|---|---|---|
| `INTERFACEMODE_PLACE_BUILDING` | `ui/interface-modes/interface-mode-place-building.js` | `decorate(overlay)` rysuje kolory podpowiedzi |
| `INTERFACEMODE_ACQUIRE_TILE` | `ui/interface-modes/interface-mode-acquire-tile.js` | widok kupna kafelka / specjalisty |
| `BuildingPlacementManager` | `ui/building-placement/building-placement-manager.js` | `urbanPlots`, `developedPlots`, `expandablePlots`, `uniqueQuarterPlots`, `allPlacementData`, `selectPlacementData()` |
| `PlotWorkersManager` | `ui/plot-workers/plot-workers-manager.js` | `workablePlots`, `blockedPlots`, `cityWorkerCap`, `update()` |
| `CityDecorationSupport` | `ui/interface-modes/support-city-decoration.js` | grupa nakładek, przyciemnienie spoza miasta, ikony slotów |

Oba tryby pobiera się po nazwie:
`InterfaceMode.getInterfaceModeHandler("INTERFACEMODE_PLACE_BUILDING")` — zwraca **zarejestrowany
singleton**, więc przypisanie do `.decorate` patchuje żywy obiekt. Menedżery też są singletonami,
patchowanymi przez `Object.getPrototypeOf(Menedżer)`.

## Warstwy soczewek ✅

| Warstwa | Klasa |
|---|---|
| `fxs-building-placement-layer` | `WorkerYieldsLensLayer` |
| `fxs-worker-yields-layer` | `WorkerYieldsLensLayer` |
| `fxs-yields-layer` | `YieldsLensLayer` |
| `fxs-city-borders-layer` | `CityBordersLayer` |
| `fxs-city-growth-improvements-layer` | `CityGrowthImprovementsLensLayer` |

⚠️ **Pierwsze dwie to DWIE OSOBNE INSTANCJE TEJ SAMEJ KLASY.** Patch na klasie trafia w obie;
patch na `LensManager.layers.get(nazwa)` trafia w jedną. City Hall patchuje je osobno, bo jedna
rysuje w zoomie mapy, a druga wewnątrz widoku stawiania budynku — i każda dostaje własną skalę
sprite'ów.

⚠️ **Warstwy rejestrują się PÓŹNIEJ niż skrypty modów** (znane z [14](14-quirks-and-gotchas.md)).
✅ Najczystsze obejście, sprawdzone w City Hall: **zaimportować moduł gry**, zanim sięgniesz po
warstwę —

```js
import '/base-standard/ui/lenses/layer/worker-yields-layer.js';   // wymusza kolejność
const WYLL = LensManager.layers.get("fxs-worker-yields-layer");
```

`import` gwarantuje, że moduł wykonał się przed kodem poniżej. To pewniejsze niż `LoadOrder`
i prostsze niż ponawianie na `InterfaceModeChangedEventName`.

Własna warstwa: `LensManager.registerLensLayer(nazwa, instancja)`, a potem dopisanie się do
soczewki: `LensManager.lenses.get(soczewka)?.activeLayers.add(nazwa)`. Warstwa potrzebuje
`initLayer()`, `applyLayer()`, `removeLayer()`.

## Wzorce patchowania — sprawdzone w City Hall ✅

### Dekorator + patch prototypu z **jednorazową blokadą**

⚠️ To jest najważniejszy wzorzec na tym ekranie i najłatwiejszy do przeoczenia.
`Controls.decorate` woła fabrykę **na każdą instancję**, a panele miasta powstają i giną przy
każdym otwarciu osady. Patch prototypu bez blokady owija prototyp raz na otwarcie i łańcuch
opakowań rośnie bez końca — objawia się dopiero po kilkunastu minutach gry.

```js
class mojDekorator {
    static c = null;                       // blokada
    constructor(component) {
        this.component = component;
        component.mojMod = this;           // most z prototypu do dekoratora
        this.patchPrototype(Object.getPrototypeOf(component));
    }
    patchPrototype(proto) {
        if (mojDekorator.c) return;        // ← bez tego łańcuch rośnie
        const c = mojDekorator.c = { proto };
        c.update = proto.update;
        proto.update = function(...args) {
            const r = c.update.apply(this, args);
            this.mojMod.afterUpdate(...args);
            return r;
        }
    }
}
Controls.decorate("panel-city-details", (v) => new mojDekorator(v));
```

### Patch właściwości akcesorowej (getter/setter)

Naiwne `proto.x = ...` kasuje setter. Poprawnie:

```js
const opis = Object.getOwnPropertyDescriptor(proto, "isPurchase");
Object.defineProperty(proto, "isPurchase", {
    ...opis,
    set(value) {
        zapamiętaj(value);
        opis.set.apply(this, [value]);   // ZAWSZE wołaj oryginał
    },
});
```

### Dodanie zakładki bez podmiany panelu

```js
const tabs = JSON.parse(component.tabBar.getAttribute("tab-items"));
tabs.unshift(MOJA_ZAKŁADKA);                       // { id, icon:{default,hover,focus,pressed}, iconClass, headerText }
component.tabBar.setAttribute("tab-items", JSON.stringify(tabs));
component.slotGroup.appendChild(slot);             // <fxs-vslot id="<to samo id>">
```

### `UpdateGate` jako debouncer cudzego wycieku

✅ Przycisk „Zwiń wszystko" w `panel-city-details` **wycieka listenery** i odpala handler kilka
razy na kliknięcie. City Hall podmienia `onCollapseAllSection` na `UpdateGate`, który wykona się
najwyżej raz na klatkę. Wzorzec działa na dowolny nieszczelny handler gry.

### Szukanie kotwic po `data-l10n-id`, nie po klasie CSS ✅

Klasy na tych panelach są generowane z SCSS gry i zmieniają się między patchami; mody je
przepisują. Id lokalizacyjne są stabilne:

```js
view.querySelector('[data-l10n-id="LOC_BUILDING_PLACEMENT_RESULTS"]')?.parentElement;
```

### Styl przełączany klasą na `<body>` ✅

Zamiast przepisywać CSS przy zmianie opcji: `document.body.classList.toggle("moja-klasa", flaga)`
i wszystkie reguły pisane jako `.moja-klasa .cokolwiek { … }`. Najtańszy możliwy mechanizm opcji.

## `<ActionCriteria>` + `<ModInUse>` — gaszenie ryzykownej połowy moda ✅

Najbardziej przenośny pomysł z City Hall. Niebezpieczna część moda dostaje **własną grupę akcji**,
zagaszoną, gdy zainstalowany jest konkurent; bezpieczna działa zawsze.

```xml
<ActionCriteria>
    <Criteria id="production-ok">
        <ModInUse inverse="1">compact-production</ModInUse>
        <ModInUse inverse="1">drongos-compact-production</ModInUse>
    </Criteria>
</ActionCriteria>
<ActionGroup id="..." scope="game" criteria="production-ok"> … </ActionGroup>
```

⚠️ `<ModInUse>` porównuje **id moda** z jego własnego `.modinfo` — nie nazwę folderu i nie nazwę
wyświetlaną.

## ⚠️ Podmiana pliku gry (`ImportFiles`) — czego dwa mody nie mogą zrobić naraz

Plik położony pod **tą samą ścieżką względną** co plik `base-standard` i wpisany w
`<ImportFiles>` zastępuje kopię gry dla wszystkich (już opisane w
[05](05-ui-javascript.md) i [14](14-quirks-and-gotchas.md) #10). Nowy fakt z praktyki:

✅ **City Hall podmienia pięć plików**, i to jest lista, w którą nie wolno wejść drugiemu modowi:

```
ui/production-chooser/panel-production-chooser.js
ui/production-chooser/production-chooser-unique-quarter.js
ui/interface-modes/support-city-decoration.js
ui-next/components/production-chooser-item.js
ui-next/components/production-chooser-unique-quarter-item.js
```

⚠️ **Gdy dwa mody podmieniają jeden plik, wygrywa jedna kopia, a zmiany drugiej znikają — bez
żadnego błędu w logach.** Nic tego nie zgłasza.

✅ Ciekawostka pokazująca, po co się to robi: `panel-production-chooser.js` w City Hall to
**dosłowna kopia 1441-liniowego pliku gry z 16-liniowym diffem** — przekierowanie dwóch importów
na własny helper i dopisanie pięciu atrybutów `data-*`. Cała reszta zmian mieszka w normalnych
plikach moda. Czyli: podmiana pliku bywa potrzebna wyłącznie po to, żeby **przestawić import**
albo **poszerzyć ścianę atrybutów** — nigdy po to, żeby napisać w nim logikę.

⚠️ Podmiana zamraża też plik na wersji gry, z której go skopiowano. Patch Firaxis nie dotrze do
gracza, dopóki autor moda nie skopiuje pliku ponownie.

## Drobiazgi, które kosztowałyby czas ✅

- **Kolory nakładek są liniowe, nie sRGB.** Przepuść przez `Color.convertToLinear(0xRRGGBB)`;
  kolor ramki w City Hall to te same składowe podzielone przez 4.
- **Sprite'y obsługują tylko wbudowane BLP-y.** Sprawdzenie z City Hall:
  `const url = UI.getIconURL(t); const blp = UI.getIconBLP(t);` i ikona jest użyteczna tylko gdy
  `url == "blp:"+blp || url == "fs://game/"+blp` — inaczej trzeba fallbacku (City Hall rysuje
  `BUILDING_OPEN` i pierwszą literę nazwy). Dotyczy ikon dodanych przez inne mody.
- **Bug w grze:** `renderYieldsSlot` w `panel-city-details` zamyka `<fxs-scrollable>` tagiem
  `</div>`. City Hall podmienia tę metodę wyłącznie po to.
- **`district.MaxConstructibles`** trzeba zmniejszyć o 1 za każdy budynek z tagiem `FULL_TILE`,
  pominąć mury (`ExistingDistrictOnly`) — i **podnieść do faktycznej liczby**, bo AI czasem
  przekracza limit.
- **Konstruowany budynek jest w trzech różnych miejscach** i trzeba sprawdzić wszystkie trzy:
  ```js
  city.BuildQueue.getQueuedPositionOfType(hash)                            // w kolejce
  Game.CityOperations.canStart(city.id, CityOperationTypes.BUILD, …)       // w budowie
  city.Constructibles.hasConstructible(hash, false)                        // gotowy
  ```
  Własna funkcja gry `findExistingUniqueBuilding` sprawdza tylko trzecie — stąd bierze się jej
  ślepota na budynek, który właśnie stawiasz.
- **`Districts.getFreeConstructible(loc, playerID)`** zwraca ulepszenie, które kafelek *by
  zbudował* — a nie to, które na nim stoi. City Hall grupuje po nim ulepszenia w tabeli
  magazynów.
- **Własna nazwa religii** wymaga przejścia `Players.getEverAlive()` i znalezienia założyciela:
  `founder.Religion?.getReligionType() == id → founder.Religion.getReligionName()`.
  `GameInfo.Religions.lookup(id).Name` daje tylko nazwę domyślną.
- **`getGlobalParamNumber("APPEAL_FOR_HAPPINESS_TILE_YIELD")`**
  (`/core/ui/utilities/utilities-data.js`) — próg atrakcyjności kafelka. Przykład na to, że
  progi z zasad gry da się czytać zamiast wpisywać na sztywno.
- ⚠️ **Odwrotny przykład:** `modelTownFocus` w City Hall ma **wpisane na sztywno** premie każdego
  focusu miasteczka (+25 fortyfikacji, +2 produkcji za kopalnię…), nieczytane z tabel modyfikatorów.
  Po rebalansie gry ten panel po cichu kłamie.

## ✅ Rozbicie dochodów miasta — gra liczy całe drzewo za nas

**Ustalone 2026-08-26.** Nie trzeba nic wyliczać z tabel modyfikatorów: `city.Yields` ma pełne
API drzewa źródeł.

```js
city.Yields.getNetYield(YieldTypes.YIELD_FOOD)          // liczba netto
city.Yields.getYields()                                  // wszystkie, w kolejności GameInfo.Yields
city.Yields.getYieldsForNode(yieldIndex, path, true)     // → { base: { value, steps }, modifier, tooltip }
city.Yields.getYieldSummaryForNode(yieldIndex, path)     // → { value, base, modifier }
```

`path` to tablica z enuma `CityYieldNodes`. Wszystkie 19 węzłów, których używa sama gra:

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

Wzorcowa implementacja do przepisania: `base-standard/ui/city-details/model-city-details.js`,
metody `addIncomeHierarchy`, `addDeductionsHierarchy`, `addYieldsToHierarchy`, `addYieldSteps`.
Obsługują już dwie rzeczy, których łatwo nie zauważyć:

- **wiersz „Inne"** — `incomeNode.base.value - childTotal`, czyli dochód, którego nazwane węzły
  nie tłumaczą (`LOC_GLOBAL_YIELDS_OTHER`);
- **modyfikatory procentowe** — `adjustment = base.value * modifier.value / 100`, podpisane
  `LOC_YIELD_BONUS_NAME` / `LOC_YIELD_PENALTY_NAME`, rozbite na pozycje z
  `INCOME_MODIFIERS` → `base.steps` przy `GameValueDisplayTypes.PERCENTAGE`.

### ✅ GOTOWE API — bez chodzenia po węzłach ręcznie (2026-08-26, z `f1rstdan-cool-ui`)

Całe drzewo w jednym wywołaniu:

```js
import CityYields from '/base-standard/ui/utilities/utilities-city-yields.js';
const yields = CityYields.getCityYieldDetails(city.id);
// → [{ label, value /* string do wyświetlenia */, valueNum, valueType, type,
//      showIcon, isNegative, isModifier, childData: [ … to samo … ] }]
```

`CityYieldsEngine` (221 linii, `base-standard/ui/utilities/utilities-city-yields.js`) sam
obsługuje rekurencję po `base.steps` / `modifier.steps`, grupowanie `LOC_ATTR_SOURCES`,
etykiety `LOC_ATTR_ADD_PERCENTAGE_OF_SOURCES` / `LOC_ATTR_MULTIPLIED_BY_SOURCES` /
`LOC_ATTR_MODIFIERS`, formatowanie procentów i mnożników oraz zwijanie nadmiarowych węzłów.
Zapas: `CityDetails.yields` z `model-city-details.js` (kształt `{name, value, children}`).

### ❗ KOREKTA 2026-08-26: `<yield-bar>` to MARTWY KOD

Wcześniejsza wersja tego pliku wskazywała `<yield-bar>`
(`base-standard/ui/yield-bar/yield-bar.js` + `model-yield-bar.js`, model `g_YieldBar`) jako pasek
dochodów miasta. **Nic w grze go nie tworzy** — plik się ładuje, element jest zdefiniowany
i na tym koniec. To ten sam wzorzec co [14](14-quirks-and-gotchas.md) #33: Firaxis zostawia
starą implementację na dysku i dalej ją ładuje.

✅ Żywy element robi sam chooser produkcji:

```js
cityYieldBar = document.createElement("yield-bar-base");     // panel-production-chooser.js:137
updateCityYieldBar() {
    …
    this.cityYieldBar.setAttribute("data-yield-bar", JSON.stringify(data));  // [{type, value, style}]
}
```

- sięga się po niego jako `panel.cityYieldBar` z dekoratora na `panel-production-chooser`;
- **szwem jest atrybut `data-yield-bar` (JSON)** — dodanie, usunięcie albo zmiana wpisu nie
  wymaga żadnej pracy w DOM. Można też **dopisać wpisy, które nie są żadnym yieldem** — tak
  `f1rstdan-cool-ui` wsadza tam populację i „sieć połączeń”, z własnymi ikonami z `UpdateIcons`;
- struktura po wyrenderowaniu (`yield-bar-base.js`): `outerContainer > container`
  (czyli `lastElementChild`), tekst wartości w `.text-sm`, styl wpisu
  `NONE | GAIN | LOSS` = `0 | 1 | 2`;
- ⚠️ **renderuje się ASYNCHRONICZNIE** — kod, który zaraz po ustawieniu atrybutu chodzi po
  dzieciach, nic nie znajdzie. Obejście sprawdzone w Cool UI: porównaj
  `root.children.length >= oczekiwane`, a jeśli nie — `MutationObserver` na `{childList: true}`
  **z bezpiecznikiem `setTimeout` 500 ms**, który rozłącza obserwator i robi co się da.
  Bez bezpiecznika obserwator wycieka.

### ✅ Tooltipy: `TooltipManager.registerType`

```js
import TooltipManager from '/core/ui/tooltips/tooltip-manager.js';
TooltipManager.registerType('moj-tooltip', new MojTooltipType());
```

Typ tooltipa to zwykły obiekt z pięcioma składowymi:

| Składowa | Kontrakt |
|---|---|
| `getHTML()` | zwraca korzeń `<fxs-tooltip>`, zbudowany **raz** w konstruktorze |
| `reset()` | czyści tylko treść, nigdy nie usuwa węzłów |
| `isUpdateNeeded(target)` | `true`, gdy cel się zmienił; tu też rozwiązuje się właściwy cel |
| `update()` | wypełnia treść z `this.target` |
| `isBlank()` | `return !this.target` — decyduje, czy tooltip w ogóle się pokazuje |

Element zgłasza się przez `data-tooltip-style="moj-tooltip"`. Model widoku wieszaj **wprost na
węźle DOM** (`el.yieldData = …`), zamiast szukać go ponownie w `update()`.

⚠️ **`isUpdateNeeded` wykonuje się przy każdym ruchu wskaźnika.** Ma być porównaniem referencji
i odczytem z cache. Cool UI trzyma tam komentarz błagający następnego czytelnika, żeby tego nie
zepsuć — bo zepsute daje zacięcia obrazu.

⚠️ Wszystko, co ma się wyświetlić, przepuszczaj przez `Locale.stylize(…)` — to on renderuje
`[icon:YIELD_FOOD]`. Odwrotność to `Locale.plainText(str)`, który **wycina ikony** ze stringa,
żeby dawało się go sparsować jako liczbę.

## ✅ Zakup: cena bez przełączania zakładki, i zakup rzeczy w budowie

```js
city.Gold.getBuildingPurchaseCost(YieldTypes.YIELD_GOLD, constructibleType)
city.Gold.getUnitPurchaseCost(YieldTypes.YIELD_GOLD, unitType)
Game.CityCommands.canStartQuery(city.id, CityCommandTypes.PURCHASE, CityQueryType.Unit)
Game.CityCommands.canStart(city.id, CityCommandTypes.PURCHASE, { ConstructibleType: hash }, false)
// → result.Success, result.Cost, result.InsufficientFunds, result.InProgress, result.Plots
```

✅ **Zakup budynku, który już się buduje, działa w samej grze** — `Construct` z
`production-chooser-helpers.js` robi to tak:

```js
if (result.InProgress && result.Plots) {
    const loc = GameplayMap.getLocationFromIndex(result.Plots[0]);
    args.X = loc.x; args.Y = loc.y;      // dokup go tam, gdzie stoi
}
Game.CityCommands.sendRequest(city.id, CityCommandTypes.PURCHASE, args);
```

⚠️ **Projektów nie da się kupić.** Blokada jest w dwóch miejscach:
`typeInfo.Kind != "KIND_PROJECT"` w `Construct` oraz `!project.CanPurchase && isPurchase`
w `getProjectItems`.

✅ **Trzy dalsze reguły zakupu** (2026-08-26, wyczytane w `f1rstdan-cool-ui` 1.9.6):

| Reguła | |
|---|---|
| **Miasteczka (`city.isTown`) nie kupują w ogóle** | gra daje im osobną ścieżkę zakupu |
| **Projekty nigdy** | jw. |
| ⚠️ **Cudów nie da się kupić — CHYBA ŻE** gracz ma w pełni odblokowany mogolski civic `NODE_CIVIC_MO_MUGHAL_GARDENS_OF_PARADISE` | prawdziwa reguła gry z jednym wyjątkiem |

```js
Game.ProgressionTrees.getNodeState(playerId, nodeType) >= ProgressionTreeNodeState.NODE_STATE_FULLY_UNLOCKED
```

❗ **Zmiana sygnatury w wersji gry 1.1.1:**
`city.Production.getConstructibleProductionCost(type, FeatureTypes.NO_FEATURE, false)` — **trzy**
argumenty. `bz-city-hall` woła to nadal z jednym. Komentarz w Cool UI notuje tę awarię.

⚠️ Kontrolka **wewnątrz** aktywowalnego wiersza (przycisk kupna w wierszu produkcji) musi robić
`event.stopPropagation()` **i** `preventDefault()`, inaczej wiersz również doda pozycję do
kolejki. Cool UI wypuściło ten błąd i naprawiło w 1.9.5.

## ✅ Tagi konstruktów — gotowe klucze do sortowania i filtrowania

`ConstructibleHasTagType(type, TAG)` z `/base-standard/ui/utilities/utilities-tags.js`;
`getConstructibleTagsFromType(type)` daje wszystkie. Liczności policzone w danych gry bazowej:

| Tag | Sztuk | Znaczy |
|---|---|---|
| `AGELESS` | 109 | ponadczasowe (magazyny itd.) |
| `UNIQUE_IMPROVEMENT` | 33 | unikalne ulepszenie |
| `UNIQUE` | 32 | unikalny budynek |
| `CITY_STATE_UNIQUE_IMPROVEMENT` | 19 | unikalne ulepszenie od miasta-państwa |
| `WAREHOUSE`, `FORTIFICATION`, `FULL_TILE`, `URBANCENTER`, `PERSISTENT`, `BRIDGE`, `WATER`… | — | patrz `constructibleTagNames` i `constructibleTagsToExclude` w `utilities-tags.js` |

✅ **Synkretyzm nie wymaga osobnej obsługi.** Tagi siedzą na konstrukcie, nie na tym, jak został
odblokowany — unikat przyznany przez synkretyzm ma `UNIQUE` dokładnie tak samo jak własny
unikat cywilizacji.

Premia za sąsiedztwo, gotowa do sortowania:

```js
BuildingPlacementManager.canGetAdjacencyBonuses(constructible.ConstructibleType)
BuildingPlacementManager.getHighestAdjacencyBonus(constructible.$hash)   // wartość
BuildingPlacementManager.getNumberOfWarehouseBonuses(constructible.$hash)
```

## Obiekty danych ✅

| Wywołanie | Daje |
|---|---|
| `UI.Player.getHeadSelectedCity()` | `ComponentID` zaznaczonej osady |
| `city.Growth` | `growthType`, `currentFood`, `getNextGrowthFoodThreshold()`, `turnsUntilGrowth`, `projectType` |
| `city.population` / `.urbanPopulation` / `.ruralPopulation` | populacja |
| `city.Workers` | `getNumWorkers(false)`, `GetAllPlacementInfo()`, `getCityWorkerCap()` |
| `city.Districts.getIds()` → `Districts.get(id)` | `isQuarter`, `isUrbanCore`, `type`, `location` |
| `city.Constructibles.getIds()` → `Constructibles.getByComponentID(id)` | co stoi |
| `city.BuildQueue` | `getQueue()`, `getQueuedPositionOfType`, `getPercentComplete(hash)` (0-100), `getTurnsLeft(TYPE_STRING)`, `currentProductionTypeHash`, `currentTurnsLeft`, `isEmpty` |
| `city.Production.getConstructibleProductionCost(hash)` | koszt |
| `city.getConnectedCities()` / `city.getPurchasedPlots()` | połączenia / kafelki osady |
| `city.Religion` | `majorityReligion`, `urbanReligion`, `ruralReligion` |
| `city.Happiness.hasUnrest` | niepokoje; ❓ nic w UI gry nie rozbija szczęścia szczegółowiej |
| `Game.CityCommands.canStart(id, CityCommandTypes.CHANGE_GROWTH_MODE, {Type: GrowthTypes.PROJECT}, false)` | `.Projects` = dozwolone focusy miasteczka |

Zdarzenia: `CityGrowthModeChanged`, `CityPopulationChanged`, `CitySelectionChanged`,
`ConstructibleAddedToMap`, `ConstructibleRemovedFromMap`, `ConstructibleChanged`,
`PlotWorkersUpdated`, plus okienne `update-city-details` i `city-details-closed`.

⚠️ Zdarzenia silnika lecą **dla każdego gracza w partii** — handler bez filtra po właścicielu
wykonuje się tysiące razy na turę AI.

## Mody, które już tu siedzą

| Mod | Workshop ID | Co robi na tym ekranie |
|---|---|---|
| `bz-city-hall` | 3507102289 | wszystko powyżej; pełna analiza w `mod-projects/better-city-ui/documentation/03-city-hall-analysis.md` |
| `f1rstdan-cool-ui` | 3510572267 | ✅ przeanalizowany (1.9.6): tooltipy dochodów przez `TooltipManager`, przycisk szybkiego zakupu w wierszu, dodatkowe wpisy w pasku dochodów. ⚠️ Jego **kompaktowy układ wiersza produkcji nie działa** od migracji do `ui-next` — autor pisze o tym w swoim changelogu. Pełna analiza w `mod-projects/better-city-ui/documentation/04-f1rstdan-cool-ui-analysis.md` |
| `EnhancedTownFocusInfo` | 3548476215 | ❓ focus miasteczka — pokrywa się z zakładką Overview City Hall |
| `najane-common-specialists-yields` | (mod użytkownika) | ⚠️ patchuje `PlotWorkersManager`, `fxs-worker-yields-layer` i `panel-place-population` — te same obiekty co City Hall, innym wzorem na bazę |

⚠️ **City Hall i mod specjalistów liczą „bazę" inaczej**: City Hall bierze `Math.min` po
składowych, mod użytkownika — wartość najbliższą zeru z zachowaniem znaku, i traktuje brak yieldu
jako jawne `0`. Tam, gdzie yield występuje z obydwoma znakami, dają różne liczby.
❓ Czy kolidują widocznie w grze — **nietestowane**, mimo że oba są zainstalowane.

## Populacja i połączone osiedla obok dochodów

Nie są dochodami — nie ma za nimi drzewa `CityYieldNodes`. Buduje się je ręcznie z API osady:

| Dane | Wywołanie |
|---|---|
| populacja łącznie / wiejska / miejska / oczekująca | `city.population`, `.ruralPopulation`, `.urbanPopulation`, `.pendingPopulation` |
| specjaliści + limit na pole | `city.Workers.getNumWorkers(false)`, `city.Workers.getCityWorkerCap()` |
| tury do nowego obywatela | `city.Growth.turnsUntilGrowth` |
| próg i zapas żywności | `city.Growth.getNextGrowthFoodThreshold().value`, `city.Growth.currentFood` |
| połączone osady | `city.getConnectedCities()` → `Cities.get(id)` → `.isTown`, `.isCapital`, `.population` |
| czy w sieci handlowej | `city.Trade.isInTradeNetwork()` |

⚠️ **Wiejska populacja = `ruralPopulation - pendingPopulation`.** Oczekujący obywatele siedzą
w `ruralPopulation`; bez odjęcia suma wierszy nie zgadza się z `city.population`.

⚠️ **Żywność ze specjalizowanego miasteczka DZIELI SIĘ** między wszystkie miasta, do których jest
podłączone. Do tej osady trafia `townFood / liczba_miast_podłączonych_do_MIASTECZKA` — dzielnik
jest po stronie miasteczka, nie po stronie miasta, które czyta tooltip.

⚠️ Specjalizowane miasteczko podłączone do miasta **nie rośnie** (żywność wychodzi), ale
`turnsUntilGrowth` dalej odlicza. Gra pokazuje ten licznik bez komentarza — to jedyna liczba na
tym panelu, która realnie wprowadza w błąd.

### Ikony bez zależności od cudzego moda

Gra ma własne: **`CITY_CITIZENS`** (populacja) i **`CITY_SETTLEMENT`** (osada), obie w
`base-standard/data/icons/city-icons.xml`, kontekst domyślny. Przydatne w wierszach:
`CITY_RURAL`, `CITY_URBAN`, `CITY_SPECIAL_BASE`, `CITY_CENTERPIN`, `YIELD_CITIES`, `YIELD_TOWNS`.

⚠️ Cool UI trzyma swoje PNG pod `fs://game/f1rstdan-cool-ui/textures/…`. Ta ścieżka rozwiązuje się
**tylko przy zainstalowanym Cool UI** — użycie jej robi z niego cichą zależność.

### Gotowe tagi lokalizacyjne (są we wszystkich językach gry)

`LOC_UI_CITY_INTERACT_CURENT_POPULATION_HEADER`, `LOC_UI_CITY_STATUS_RURAL_POPULATION`,
`LOC_UI_CITY_STATUS_URBAN_POPULATION`, `LOC_UI_SPECIALISTS_SUBTITLE`,
`LOC_UI_ACQUIRE_TILE_ADD_POPULATION_MAX_PER_TILE`, `LOC_UI_CITY_DETAILS_NEW_CITIZEN_IN_TURNS`,
`LOC_UI_CITY_DETAILS_FOOD_NEEDED_TO_GROW`, `LOC_UI_CITY_STATUS_CURRENT_FOOD_STOCKPILE`,
`LOC_UI_CITY_DETAILS_FOOD_PER_TURN`, `LOC_UI_CITY_DETAILS_GROWTH_TAB` („Przyrost obywateli"),
`LOC_PEDIA_CONCEPTS_PAGE_CONNECTED_1_TITLE`, `LOC_UI_SETTLEMENT_TAB_BAR_CITIES`,
`LOC_UI_SETTLEMENT_TAB_BAR_TOWNS`, `LOC_GLOBAL_YIELDS_SUMMARY_TOTAL_INCOME`.

⚠️ `LOC_UI_CITY_DETAILS_GROWTH_TITLE` **nie istnieje** — jest `…_GROWTH_TAB` i `…_GROWTH_BREAKDOWN`.

## Podświetlanie pól przy stawianiu budowli (`INTERFACEMODE_PLACE_BUILDING`)

Rysuje to `PBIM.decorate(overlay)` — handler pobierany przez
`InterfaceMode.getInterfaceModeHandler('INTERFACEMODE_PLACE_BUILDING')`. Nadpisanie tej jednej
metody wystarcza; nie trzeba ruszać `selectPlacementData`.

Do dyspozycji w momencie rysowania (wszystko na `BuildingPlacementManager`):

| Właściwość | Zawartość |
|---|---|
| `urbanPlots` | istniejące dzielnice miejskie (kolor „best") |
| `developedPlots` | kafelki w dzielnicy, ale nie miejskiej („okay") |
| `expandablePlots` | wieś / niezagospodarowane („good") |
| `uniqueQuarterPlots` | dzielnice domykające unikalną dzielnicę (VFX) |
| `potentialUniqueQuarterPlots` | `{plotID, uniqueQuarterDef}` |
| `currentConstructible` | definicja stawianej budowli |
| `isRepairing`, `cityID` | kontekst |

⚠️ **Kolory to AABBGGRR, nie RGBA.** Wartości bazowe: `0xc84db123` (best), `0xc800f2fe` (okay),
`0xc81de5b5` (good). W kodzie gry zapisane dziesiętnie (`3360534819` itd.).

```js
this.plotOverlay = overlay.addPlotOverlay();
this.plotOverlay.addPlots(listaIndeksów, { fillColor: 0xc8003ce6 });
this.uniqueQuarterModelGroup.addVFXAtPlot('VFX_3dUI_Hex_Highlight_01', plot,
    { x: 0, y: 0, z: 0 }, { angle: 0, constants: { Color3: [1, 0.992, 0.62], Alpha1: 1 } });
```

⚠️ Nadpisując `decorate`, trzeba **powtórzyć całą bazową treść** — `CityZoomer.zoomToCity` oraz
`WorldUI.pushRegionColorFilter(city.getPurchasedPlots(), {}, this.OUTER_REGION_OVERLAY_FILTER)`.
Wywołanie oryginału najpierw nic nie da, jeśli chcesz przemalować te same pola: bazowy kolor
zostanie na wierzchu.

### ❗ `findExistingUniqueBuilding` widzi tylko GOTOWE budynki

Bazowa metoda pyta wyłącznie `city.Constructibles.hasConstructible(hash, false)`. Skutek: gdy
pierwsza połowa unikalnej dzielnicy jest **w kolejce albo w budowie**, gra przestaje podświetlać tę
dzielnicę pod drugą połowę — czyli dokładnie wtedy, kiedy jest to najbardziej potrzebne.

Poprawka to patch na **prototypie** (`Object.getPrototypeOf(BuildingPlacementManager)`), bo
`selectPlacementData` woła tę metodę po drodze. Konstrukt bywa w trzech miejscach — patrz sekcja
wyżej w tym pliku.

### Reguła „to pole zepsuje unikalną dzielnicę"

Dla stawianej budowli, o ile nie jest to naprawa ani `ExistingDistrictOnly`:

1. Zbierz dzielnice, w których stoi już połowa odblokowanej unikalnej dzielnicy
   (`Players.Constructibles.get(owner).getUnlockedUniqueQuarters()` → `GameInfo.UniqueQuarters`).
2. Kandydat, który **jest** taką dzielnicą → OK tylko wtedy, gdy to ta sama unikalna dzielnica.
3. Kandydat pod **unikalny budynek** w nowym miejscu → źle, jeśli druga połowa jest już gdzie
   indziej albo dzielnica jest „zepsuta".
4. Dzielnica jest zepsuta, gdy stoi w niej budynek **ponadczasowy** albo z **bieżącej ery**:

```js
Database.makeHash(def.Age ?? '') === Game.age
```

⚠️ Liczą się tylko `ConstructibleClass == 'BUILDING'` bez `ExistingDistrictOnly` — mury nie zajmują
slotu. Budynek z **poprzedniej** ery nie blokuje, bo da się go nadbudować.

### Ikony budynków na kafelkach w trybie stawiania

Rysuje je warstwa soczewki `fxs-building-placement-layer`
(`LensManager.layers.get('fxs-building-placement-layer')`), metodą `realizeBuildSlots(district)`.

⚠️ Gra woła ją **tylko dla kafelków kandydujących** — z `getPlacementOptions()`, czyli
`urbanPlots + developedPlots + expandablePlots`, i tylko tam, gdzie `Districts.getAtLocation()`
coś zwraca. Reszta osady zostaje pusta.

⚠️ Nazwa metody nadrzędnej ma **literówkę w kodzie gry**: `realizeBuidlingPlacementSprites`
(„Buidling"). Trzeba ją odwzorować dokładnie.

Rozszerzenie na całą osadę bez dublowania:

```js
let drawn = null;
layer.realizeBuildSlots = function (district) {          // zapisuje, co gra sama narysowała
    drawn?.add(GameplayMap.getIndexFromLocation(district.location));
    return original.apply(this, arguments);
};
layer.realizeBuidlingPlacementSprites = function (...a) {
    drawn = new Set();
    try { originalSprites.apply(this, a); dorysujResztę(this); } finally { drawn = null; }
};
```

⚠️ **Wołaj przez, nie podmieniaj.** City Hall zastępuje `realizeBuildSlots` bogatszą wersją
(specjaliści, plakietki dochodów, litera na ikonach, których nie da się załadować jako sprite)
i ładuje się wcześniej — wywołanie oryginału sprawia, że to jego rysowanie trafia też na
dorysowane kafelki.

⚠️ Renderer rysuje **jedno pole-zaślepkę na każdy wolny slot** (`MaxConstructibles`), więc bez
filtra „czy cokolwiek tu stoi" każdy kafelek wiejski dostanie rządek pustych ramek. Konstrukty
`ExistingDistrictOnly` (mury) nie liczą się jako zabudowa — nie zajmują slotu.

Kafelki osady: `city.Districts.getIds()` → `Districts.get(id)` → `.location`, `.type`.
