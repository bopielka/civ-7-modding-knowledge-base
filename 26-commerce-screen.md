# 26. Ekran Handlu (Commerce Screen) — mapa dla modów

**Data ustalenia: 2026-08-10.** Wszystko poniżej ✅ przeczytane w plikach gry
(wersja z 2026-07-28). Baza dla moda „Better Commerce Screen UI by Najane".

To ekran, na którym gracz **przydziela zasoby do osad i przegląda szlaki handlowe**.
Otwiera się przez `ContextManager.push("screen-resource-allocation", …)`.

⚠️ **Napisany w `ui-next` / Solid.js**, nie w starym frameworku — przeczytaj najpierw
[25-ui-next-solidjs.md](25-ui-next-solidjs.md). `Controls.decorate` tu **nie zadziała**.

## Pułapka nazewnicza ⚠️

Nazwa elementu to nadal **`screen-resource-allocation`** (stara), a katalog i nazwy
komponentów mówią **`commerce`** (nowa). W grze istnieją oba komplety plików:

| | ścieżka | status |
|---|---|---|
| stary | `base-standard/ui/resource-allocation/screen-resource-allocation.js` | ładowany, ale **przegrywa** |
| stary model | `base-standard/ui/resource-allocation/model-resource-allocation.js` | jw. |
| **nowy** | `base-standard/ui-next/screens/commerce/` | **to widzi gracz** ✅ |

Oba są w `base-standard.modinfo` w `<UIScripts>`; nowy wygrywa priorytetem
`Controls.define` (1 vs 0) — mechanizm opisany w [25](25-ui-next-solidjs.md).
**Modyfikowanie starych plików nie da żadnego efektu na ekranie.**

Podobnie mylące sąsiedztwo (to **inne** ekrany, nie Commerce):
`ui/city-trade/` (handel z konkretnym miastem), `ui/trade-route-chooser/`,
`ui/lenses/layer/trade-layer.js`, `ui/interface-modes/interface-mode-resource-allocation.js`.

## Pliki ✅

```
base-standard/ui-next/screens/commerce/
  commerce-screen.tsx                 szkielet: ScreenFrame + 4 zakładki
  commerce-screen-model.ts            3334 linie — CAŁA logika i dane wszystkich zakładek
  commerce-screen-base-tab-content.tsx wspólna ramka zakładki (tytuł, opis, pasek nagłówka)
  commerce-screen-resources-tab.tsx   zakładka „Zasoby" (1640 linii, drag&drop)
  commerce-screen-trade-tab.tsx       zakładka „Handel" (szlaki)
  commerce-screen-empire-tab.tsx      zakładka „Imperium"
  commerce-screen-treasure-tab.tsx    zakładka „Skarby" (tylko AGE_EXPLORATION)
  trade-route-card.tsx                kafelek pojedynczego szlaku
  treasure-convoy-card.tsx            kafelek konwoju skarbów
  treasure-convoy-progress-bar.tsx
  factory-type-display.tsx            wybór zasobu fabrycznego
  commerce-criteria-display.tsx       wiersz „warunek spełniony / niespełniony"
  commerce-screen.css / .scss.js      style
```

Teksty: `base-standard/text/en_us/CommerceScreenText.xml` — 109 wpisów, wszystkie
z prefiksem `LOC_COMMERCE_*`. Starsze klucze zakładki zasobów to `LOC_UI_RESOURCE_*`.

## Struktura ekranu ✅

```
<screen-resource-allocation>            (custom element, klasa .fullscreen)
└ CommerceScreenContext.Provider        model = createCommerceScreenModel()
  └ ScreenFrame  name="Commerce-Screen"  title = LOC_COMMERCE_SCREEN_TITLE(nazwa cywilizacji)
    └ Tab                               nextHotkey="nav-next" previousHotkey="nav-previous"
      ├ Tab.Item "Resources"  LOC_UI_RESOURCE_ALLOCATION_TITLE  → CommerceResourcesContainer
      ├ Tab.Item "Trade"      LOC_COMMERCE_TRADE_ROUTE_TAB      → TradeRoutesContainer
      ├ Tab.Item "Empire"     LOC_RESOURCECLASS_EMPIRE_NAME     → EmpireResourceContainer
      └ Tab.Item "Treasure"   LOC_RESOURCECLASS_TREASURE_NAME   → TreasureResourceContainer
                                        ⚠️ tylko gdy Game.age == Database.makeHash("AGE_EXPLORATION")
```

Każda zakładka opakowana jest w `CommerceScreenBaseTabContent`
(tytuł + opis + `headerBar` na sortowanie/wyszukiwanie + ciemne tło + ramka `hud_section-line`).

## Punkty nadpisania (`ComponentRegistry.register`) ✅

To jedyne miejsca, w które mod może wejść bez podmiany całego ekranu:

| nazwa | co to jest | zasięg zmian |
|---|---|---|
| `CommerceScreen` | cały ekran | wszystko, ale trzeba odtworzyć resztę |
| `CommerceScreenBaseTabContent` | wspólna ramka **wszystkich** zakładek | dopisanie czegoś do każdej zakładki naraz — najtańszy haczyk |
| `CommerceResourcesContainer` | cała zakładka „Zasoby" | |
| `TradeRouteCard` | kafelek jednego szlaku | |
| `TreasureConvoyCard`, `TreasureConvoyProgressBar` | konwoje skarbów | |
| `FactoryTypeDisplay` | wybór zasobu fabrycznego | |
| `CommerceCriteriaDisplay` | jeden wiersz warunku szlaku | |

❗ **Nie są zarejestrowane** (czyli nie da się ich nadpisać po nazwie):
`TradeRoutesContainer`, `EmpireResourceContainer`, `TreasureResourceContainer`
oraz wszystkie komponenty wewnętrzne zakładki zasobów (`CityResourceContainer`,
`AvailableResourcesSection`, `SettlementName`, `DraggableResource`, `ResourceSlot`,
`WarningBanner`…). Żeby je zmienić, trzeba nadpisać ich rodzica.

⚠️ **`CommerceScreenModel` z `ModelRegistry` jest ślepą uliczką.** Model jest
zarejestrowany (`ModelRegistry.register("CommerceScreenModel", SharedInstance,
createCommerceScreenModel)`), ale `commerce-screen.tsx` **woła fabrykę bezpośrednio**
(`const model = createCommerceScreenModel()`), więc nadpisanie rejestracji niczego nie
zmieni. Żeby podmienić dane, trzeba nadpisać `CommerceScreen` i podać własny model
do `CommerceScreenContext.Provider`.

Dostęp do modelu z wnętrza własnego komponentu:
`useCommerceScreenContext()` z `/base-standard/ui-next/screens/commerce/commerce-screen-model.js`
(rzuca wyjątkiem poza drzewem ekranu).

## Model — co jest w środku ✅

`createCommerceScreenModel()` zwraca `createMutable(...)` z polami zebranymi w
`CommerceScreenContextModel`. Najważniejsze:

**Dane (`model.data`, typ `CommerceScreenData`):**
`resourceTabData`, `tradeRouteTabData`, `empireTabData`, `treasureTabData`, `ornatePanelData`.

**Wybór/fokus (sygnały Solid):** `selectedResource`, `prevSelectedResource`,
`focusedResource`, `selectedSettlementId`, `focusedSettlementId`, `selectedTradeRouteId`,
`selectedEmpireResource`, `selectedTreasureConvoyId`, `ghostResourceFocused`.

**Akcje:** `clickAvailableResource`, `slotSelectedResource(cityID, targetResourceValue?)`,
`clickSlottedResource`, `unslotSelectedResource`, `deselectSelectedResource`,
`clearAllResources(cityID?)`, `clearFactoryResources(cityID)`, `clickCityName`,
`clickTreasureFleet`, `clickUnimprovedTreasure(location)`, `clickCloseButton`.

**Reguły:** `canSelectResource`, `canDropResourceOnTarget`,
`canAssignSelectedResourceToSettlement`, `resourceIsConnectedToTradeNetwork`,
`cityIsConnectedToTradeNetwork`, `settlementHasSlottedResources`, `hasUnassignedResources`.

**Sortowanie/filtrowanie:** `selectedSettlementSortType` (enum `ResourceSettlementSortType`:
typ osady, nazwa, wolne sloty, wszystkie sloty, magazyny, ląd bliski/daleki, kolej,
fabryka, oraz każdy z 7 yieldów), `selectedTradeRouteSorting` (enum `TradeRouteSortType`),
`selectedResourceFilter`, `tradeRouteSearch(text)` (rozmyte, `FullTextSearch`).

**Odświeżanie:** `onMount` podpina `createEngineEvent("ResourceCapChanged" |
"ResourceAssigned" | "ResourceUnassigned")` i zbiera je starym `UpdateGate`
w jedno przeliczenie.

## Zakładka „Handel" — jak powstają szlaki ✅

`populateTradeRoutes()` woła
`Players.get(GameContext.localPlayerID).Trade.projectPossibleTradeRoutes(
INCLUDE_FAILED + EXTENDED_STATUS)` i rozdziela wynik na **trzy sekcje**
(`CollapsibleContainer`):

| sekcja | warunek (`route.status`) | `TradeRouteAvailabiltyType` |
|---|---|---|
| aktywne `LOC_COMMERCE_ACTIVE_TRADE_ROUTES_TITLE` | `ALREADY_EXISTS` | `Established` |
| dostępne `LOC_COMMERCE_AVAILABLE_TRADE_ROUTES_TITLE` | `SUCCESS` | `Available` |
| niedostępne `LOC_COMMERCE_UNAVAILABLE_TRADE_ROUTES_TITLE` (domyślnie zwinięta) | reszta | `Unavailable` |

Trasy ze statusem `NO_RESOURCES` są **pomijane w całości**.

Jeden `TradeRouteData` niesie: nazwę i ikonę miasta (`res_capital` / `Yield_Towns` /
`Yield_Cities`), `isCityState`, `domainString` („dostarczane do X drogą lądową/morską”),
`incomingResources` (sortowane wg klasy zasobu: City → Bonus → Empire → Treasure →
Factory, potem alfabetycznie), `yieldElement` (gotowy JSX z `LOC_TRADE_LENS_YIELD_EXPORT`),
`relationshipChange` (`getPotentialRelationshipGainFromTradeRouteWith`), `leaderId`,
`cityID`, `fullText` (materiał dla wyszukiwarki) oraz `statuses`.

`statuses` to zawsze **trzy** wpisy `CommerceCriteriaStatusEntry`
(pojemność / zasięg / pokój), każdy z `isNegative` i kluczem tooltipa; pokazywane są
tylko w kafelku niedostępnego szlaku. ⚠️ Komentarz Firaxis w kodzie przyznaje, że dla
kryterium zasięgu nie da się z tego miejsca sprawdzić, czy jest spełnione
(`appliesToCurrentCiv: true` na sztywno).

**Układ kafelków** liczony jest ręcznie w `TradeRoutesContainer`: `ResizeObserver` +
`checkForWrap()` mierzy kontener, dzieli szerokość przez `DEFAULT_CARD_WIDTH`
(`Layout.pixelsToScreenPixels(512)`) i ustawia szerokość każdej karty w px.
Do czasu pierwszego pomiaru karty mają `opacity-0`. ⚠️ Nadpisanie `TradeRouteCard`
własną kartą o innych rozmiarach rozjedzie tę logikę — trzeba respektować
`props.style.width` i klasę `.trade-route-card` (po niej `querySelectorAll` liczy karty).

Sortowanie szlaków: liczba zasobów / nazwa lidera / relacja z liderem / nazwa osady /
typ osady. Pasek wyszukiwania (`SearchBar`) pokazuje się **tylko** na
`ViewExperience() === UIViewExperience.Desktop` i przy nieaktywnym padzie.

## Zakładka „Zasoby" — struktura ✅

Największa i najbardziej złożona. Dwie kolumny danych w modelu:

- `availableResourceSectionData[]` — nieprzydzielone zasoby, dzielone na podsekcje wg
  klasy (City / Bonus / Factory…), z flagą `isConnectedToTradeNetwork`;
- `slottedResourceSectionData[]` — osady (`CommerceCityResourceData`): nazwa i ikona
  osady, `baseYields` + `yieldDeltas` (podgląd zmiany yieldów!), `slottedResources`,
  `availableSlots`, `factoryResourceData`;
- `unslottedBonuses` — premia z zasobów nieprzydzielonych
  (`getUnassignedResourceYieldBonus`).

Przenoszenie zasobów to pełny **drag & drop** (`createTypedDragAndDrop` z
`core/ui-next`), z osobną obsługą pada (`GamepadTrayItemProvider`, enumy
`ResourceTabInteractionTypeFlag` / `ResourceTabInteractionCombo` kodujące bitowo
kombinacje „zaznaczony zasób + najechana osada").

`CommerceCityNameData` niesie gotowe liczniki, przydatne do własnych podsumowań:
`warehouseCount`, `tradeConnectionCount`, `waterCount`, `hasRail`, `isTown`,
`townFocusName/Icon`, `settlementDistanceTypeName`.

## Zakładka „Imperium" i „Skarby" ✅

- **Imperium**: `EmpireResourceData[]` — zasób, ile sztuk, z jakich miast i od jakich
  liderów pochodzi (`resourceOriginData`, `tooltips` per gracz).
- **Skarby**: tylko w epoce Odkryć. `TreasureFleetData` — miasto, lista zasobów
  skarbowych, postęp (`progress`/`progressGoal`), `getTurnsUntilTreasureGenerated()`,
  `isDistantLand`, plus `statuses` w tym samym formacie co szlaki.

## ⭐ Działający wzorzec z Workshop: mod **Resource+** ✅

`steamapps\workshop\content\1295660\3756000777\ui\commerce\resource-plus.js` (1251 linii,
id `brads-assign-all-resources`, autor Brad). **Jedyny znany mod, który modyfikuje ten
ekran** — i dowód, że cała powyższa teoria działa w praktyce. Robi: przycisk
„Assign All / Reassign All", per-osada wybór priorytetu yieldu, kłódki blokujące
zasoby przed przenoszeniem.

Cały mod to **jeden plik JS + `.modinfo`** — bez `text/`, bez CSS-a jako pliku,
bez zależności. `AffectsSavedGames = 0`, `LoadOrder` 1100.

**Technika (opisana szerzej w [25](25-ui-next-solidjs.md)):**

1. importuje `CommerceResourcesContainer` z gry, bierze `.factory` jako `originalFactory`;
2. rejestruje własny wrapper pod tą samą nazwą z `overridePriority: 1100`;
3. wrapper woła `useCommerceScreenContext()` (działa — jesteśmy w drzewie Providera),
   w `onMount` wstrzykuje **goły DOM** (`document.createElement`), a na końcu zwraca
   `originalFactory(props)`;
4. `<style id="brad-assign-all-style">` ląduje w `document.head`;
5. `MutationObserver` na `document.body` wywołuje `reconcileUI()` po każdej przebudowie
   drzewa przez Solid;
6. `onCleanup` usuwa **każdy** wstrzyknięty element z osobna.

❗ **`overridePriority: 1100` jest już zajęte** na `CommerceResourcesContainer`.
Nasz mod musi dać więcej, jeśli chce być „na zewnątrz", i **musi delegować** do
`originalFactory` — inaczej Resource+ zniknie (dokładnie ten sam błąd, co konflikt
z City Hall, quirk #30).

**Zmiany w grze robi przez oficjalne operacje gracza**, nie przez grzebanie w modelu:

```js
Game.PlayerOperations.canStart(GameContext.localPlayerID,
    PlayerOperationTypes.ASSIGN_RESOURCE,
    { Location: GameplayMap.getLocationFromIndex(resourceValue), City: cityID.id },
    false).Success
```

⚠️ To już **zmiana mechaniki**, nie tylko UI — jeśli nasz mod ma zostać czysto UI-owy,
tego kroku nie powielamy.

## Haczyki DOM w zakładce „Zasoby" ✅

Atrybuty `data-name` i klasy używane przez Resource+, potwierdzone w źródłach gry
(`name=` na `Activatable`/`SpatialSlot` renderuje się jako `data-name`):

| selektor | co to |
|---|---|
| `[data-name="available-resources-container"]` | lewa kolumna: zasoby nieprzydzielone |
| `[data-name="commerce-unassigned-resources"]` | lista wewnątrz niej |
| `[data-name="slotted-resource-container"]` | prawa kolumna: osady |
| `[data-name="commerce-screen-base-tab-content"]` | wspólna treść **każdej** zakładki |
| `[data-name$="-city-resource-activatable"]` | kafelek jednej osady |
| `[data-name^="city-resource-container-"]` | wnętrze kafelka osady (z nazwą osady!) |
| `.text-secondary.w-full.mb-2` | sekcja (połączone / rozłączone) |
| `.flex.flex-row.flex-wrap.relative.w-full.justify-between` | nagłówek kafelka osady |
| `.size-19` | pojedynczy slot na zasób w osadzie |
| `[data-name="Commerce-Screen-Trade-Tab"]`, `…-Empire-Tab`, `…-Treasure-Tab` | korzenie zakładek |
| `[data-name$="-Trade-Route-Card"]`, `.trade-route-card` | kafelek szlaku |

⚠️ Klasowe selektory (`.text-secondary.w-full.mb-2`) są kruche — to zwykłe klasy
układu, nie identyfikatory. `data-name` jest bezpieczniejsze.

**Mapowanie model → DOM** Resource+ robi po indeksie, nie po id:

```js
const sections = container.querySelectorAll('.text-secondary.w-full.mb-2');
model.data.resourceTabData.slottedResourceSectionData.forEach((section, i) => {
    const cards = sections[i].querySelectorAll('[data-name$="-city-resource-activatable"]');
    section.cityResources.forEach((settlement, j) => { /* cards[j] ↔ settlement */ });
});
```

⚠️ Zakłada, że kolejność w DOM = kolejność w modelu. Przy sortowaniu/filtrowaniu
osad to założenie może pęknąć — lepszym kluczem jest nazwa osady z
`data-name="city-resource-container-<nazwa>"`.

## Inny sąsiad: „Trade Chooser Improvements" (Slothoth) ✅

Workshop 3570879406, id `resource-fixes-deadbeef`, katalog nazwany
`Resource-Screen-Improvements` — **mylące, to NIE jest ekran Handlu.** Dotyczy
`ui/trade-route-chooser/` (panel wyboru szlaku przy jednostce handlowej), czyli
**starego** frameworka, i używa `<ImportFiles>` do podmiany całych plików gry
(`trade-route-chooser.js`, `trade-routes-model.js`) — najbardziej inwazyjnej techniki,
gwarantującej konflikt z każdym innym modem ruszającym te pliki. Dorzuca też własne
ikony (`UpdateIcons` + `.dds`) i teksty przez `.sql`.

## Cofanie przypisania zasobu ✅

Gra robi to jedną operacją gracza (`commerce-screen-model.ts`, `unassignResource`):

```js
Game.PlayerOperations.sendRequest(GameContext.localPlayerID,
    PlayerOperationTypes.ASSIGN_RESOURCE, {
        Location: GameplayMap.getLocationFromIndex(resourceValue),
        City: cityID.id,
        Action: PlayerOperationParameters.Deactivate,   // ← to odróżnia od przypisania
    });
```

Czyli **ta sama operacja co przypisanie**, tylko z `Action: Deactivate`. Przypisanie to
ten sam obiekt bez pola `Action`. Zamiana miejscami to osobna operacja `SWAP_RESOURCES`
z `{ Location, Location2 }`.

Po wysłaniu nic nie trzeba odświeżać ręcznie — model nasłuchuje `ResourceUnassigned`
i przelicza się przez `UpdateGate`.

⚠️ Modelowe `unslotSelectedResource()` działa na **aktualnie zaznaczonym** zasobie, więc
do cofania konkretnego trzeba by go najpierw zaznaczyć. Przy operacji masowej to znaczy
tyle zmian sygnału zaznaczenia, ile zasobów — lepiej wołać operację wprost.

**Identyfikacja rodzaju zasobu:** `ResourceSlotData.resourceType` to
`ResourceDefinition.ResourceType`, czyli `"RESOURCE_CAMELS"` itp. ⚠️ Nie mylić z
`ResourceProps.resourceType`, które trzyma **klasę** zasobu jako klucz lokalizacji
(`LOC_RESOURCECLASS_CITY_NAME`) — to dwa różne pola o tej samej nazwie.

## Przypisywanie zasobu — obie drogi kończą się w jednym miejscu ✅

**2026-08-10.** Gracz może przypisać zasób na dwa sposoby, ale w kodzie schodzą się one
do **jednego wywołania modelu**:

| droga | miejsce w `commerce-screen-resources-tab.tsx` | wywołanie |
|---|---|---|
| kliknięcie w osadę | `Activatable` karty osady, `onActivate` | `model.slotSelectedResource(cityID)` |
| kliknięcie w pusty slot | „ghost" `Activatable`, `onActivate` | `model.slotSelectedResource(cityID)` |
| przeciągnięcie | `<DragAndDrop onDragDrop={…}>` w `CommerceResourcesContainerComponent` | `model.slotSelectedResource(cityID)` |

Dlatego mod, który chce zmienić zachowanie **przy każdym** sposobie przypisania,
opakowuje jedną metodę zamiast trzech komponentów. Model to `createMutable`, więc
własność da się po prostu podmienić i przywrócić przy sprzątaniu:

```js
const original = model.slotSelectedResource;
model.slotSelectedResource = (cityID, targetResourceValue) => {
    const assigned = model.selectedResource().resourceValue;   // ⚠️ przed wywołaniem!
    original.call(model, cityID, targetResourceValue);
    …
};
```

⚠️ Zaznaczenie czytaj **przed** wywołaniem oryginału — `handleSlotSelectedResource`
kończy się `handleDeselectSelectedResource()`, więc potem jest już puste.

⚠️ Wywołanie z drugim argumentem (`targetResourceValue`) to **zamiana dwóch zasobów
miejscami**, a nie wstawienie w wolny slot. Jeśli Twoja logika dotyczy tylko
przypisywania, pomijaj takie wywołania.

Zdarzenia potwierdzające ze strony silnika: **`ResourceAssigned`** i
**`ResourceUnassigned`** (`engine.on` / `engine.off`) — model ekranu nasłuchuje ich tak
samo, dodatkowo z `ResourceCapChanged`.

## Zasoby, które same dają sloty — `BonusResourceSlots` ✅

**2026-08-10.** Wielbłądy dają osadzie **dwa dodatkowe sloty na zasoby**, i nie jest to
zaszyte w kodzie, tylko w danych:

```xml
<!-- base-standard/data/resources.xml -->
<Row ResourceType="RESOURCE_CAMELS" … ResourceClassType="RESOURCECLASS_CITY"
     Weight="10" BonusResourceSlots="2" UnlocksCiv="true"/>
```

`BonusResourceSlots` to **kolumna schematu** (`Base\Assets\schema\gameplay\01_GameplaySchema.sql`,
`INTEGER NOT NULL DEFAULT 0`). Na 2026-08-10 wielbłądy są jedynym zasobem z wartością
niezerową w całej grze wraz z DLC — ale nie zakładaj tego na stałe, tylko czytaj kolumnę:

```js
GameInfo.Resources.forEach((r) => { if (r.BonusResourceSlots > 0) … });
```

⚠️ **Konsekwencja przy cofaniu przypisania:** wyjęcie takiego zasobu **zmniejsza
pojemność osady**, więc tyle samo innych zasobów musi wyjść razem z nim, inaczej osada
trzymałaby więcej, niż ma slotów. Przy wielbłądzie to 2 dodatkowe zasoby, przy dwóch
wielbłądach 4. Kolejność ma znaczenie: **najpierw towarzysze, potem zasób dający sloty.**

⚠️ Na towarzyszy nie wybieraj zasobów, które **same** dają sloty — to zmniejszyłoby
pojemność jeszcze raz i zrobiła się kaskada.

❗ **`sendRequest` tylko KOLEJKUJE operację.** Zapytanie `canStart` o kolejny zasób
w tym samym takcie widzi jeszcze stary stan. Wysłanie towarzyszy i wielbłąda naraz
kończy się tak: towarzysze wychodzą, wielbłąd **zostaje** (odmowa), a następne
kliknięcie znów nalicza towarzyszy za usunięcie, które ich już nie potrzebuje.

Poprawnie jest **sekwencyjnie i pytając zamiast licząc**:

1. spróbuj `canStart` na właściwym zasobie — jeśli przechodzi, wyślij i koniec;
2. jeśli nie: zwolnij **jednego** towarzysza (od końca listy), poczekaj na potwierdzenie,
   wróć do 1;
3. przerwij po wyczerpaniu puli kandydatów (ogranicz ją liczbą traconych slotów).

Dzięki temu osada nigdy nie traci więcej, niż silnik faktycznie wymaga — a wymagania
nie trzeba znać z góry.

Na potwierdzenie zmiany czekaj na zdarzenie silnika **`ResourceUnassigned`**
(`engine.on` / `engine.off`), z zabezpieczeniem czasowym na wypadek operacji, która
cicho przepadnie.

`GameInfo.Resources` czyta się iteracyjnie (`forEach`), nie zapytaniem — tak samo robi
model ekranu w `indexResourceTypes()`.

## Otwarte pytania ❓

- Jak `ScreenFrame` / `ornatePanelData` reaguje na zmianę wysokości treści zakładki.
- Czy da się wstrzyknąć własną **piątą zakładkę** do `<Tab>` bez nadpisywania całego
  `CommerceScreen` (`Tab.Item` to dzieci `CommerceScreen`, więc raczej nie).
- Czy `MutationObserver` na `document.body` (technika Resource+) zauważalnie kosztuje
  na dużych mapach — czy da się zawęzić do kontenera zakładki.

## Zaczepy do zmiany wyglądu zakładki „Zasoby" ✅

**2026-08-10.** Struktura paska nagłówka (`headerBar` przekazany do
`CommerceScreenBaseTabContent` w `commerce-screen-resources-tab.tsx`):

```
div.flex.flex-row.w-full.items-center                    ← korzeń paska
├ div[style="width: 28%"]                                ← kolumna „NIEPRZYDZIELONE"
│  ├ div  (tytuł LOC_COMMERCE_AVAILABLE_RESOURCES_TITLE)
│  └ div.flex.flex-row                                   ← wiersz sum yieldów
│     └ div.ml-2.flex.flex-row × N                       ← jedna suma: ikona + „+54"
├ div  (tytuł LOC_COMMERCE_SETTLEMENTS_TITLE)
└ [data-name="filter-and-sort"]                          ← ⭐ stabilny uchwyt
   ├ etykieta FILTRUJ + <Dropdown class="… min-h-14">
   └ etykieta SORTUJ  + <Dropdown class="… min-h-14">
```

`[data-name="filter-and-sort"]` pochodzi z `HSlot name="filter-and-sort"` i jest
**jedynym nazwanym elementem w tym pasku** — od niego najłatwiej dojść strukturalnie do
reszty (`.parentElement` to korzeń paska, jego pierwsze dziecko to kolumna
nieprzydzielonych, jej ostatnie dziecko to wiersz sum).

Wysokość rozwijanych list bierze się z klasy `min-h-14` = **3.1111rem**
(`core/ui/themes/default/default.css`). Dla porównania `min-h-10` = 2.2222rem,
`min-h-8` = 1.7777rem.

**Rozpoznanie, do którego yieldu należy dana suma:** `YieldBonus` niesie tylko
`iconSrc` i `bonusAmount`, bez typu. Nie zgaduj z kolejności — zbuduj mapę tą samą
funkcją, której użył model:

```js
import { Icon } from '/core/ui/utilities/utilities-image.js';
const byIcon = new Map();
GameInfo.Yields.forEach((y) => byIcon.set(`url(${Icon.getYieldIcon(y.YieldType)})`, y.YieldType));
```

**Linia z instrukcją nad panelem** (`LOC_COMMERCE_RESOURCE_ALLOCATION_DESCRIPTION`)
renderuje `CommerceScreenBaseTabContent` jako `div.text-base.w-full.text-center.my-4`,
**rodzeństwo** ramki z treścią — nie da się do niej dojść z wnętrza zakładki, zostaje
selektor CSS. Zakres najlepiej dać przez element `screen-resource-allocation`, czyli
korzeń całego ekranu (nazwa custom elementu z `defineLegacyComponent`).

⚠️ Ten komponent jest wspólny dla **wszystkich czterech zakładek**. Jeśli zmiana ma
dotyczyć tylko jednej, nie kombinuj z selektorem — **dołączaj arkusz w `onMount`
komponentu tej zakładki i usuwaj w `onCleanup`**. Zakres wychodzi wtedy z cyklu życia,
a nie z CSS-a.

## Wygląd plakietek jak w Drongo's Top Panel ✅

Mod **Drongo's Top Panel** (Workshop 3734234006) nie rysuje własnych plakietek — dokłada
tło do elementów, które gra już renderuje, przez arkusz w
`ui/diplo-ribbon/css-constants.js`. Przepis (do skopiowania, gdy coś ma wyglądać „jak
na górnym pasku"):

```css
height: 1.7777777778rem;            /* = min-h-8 */
border-radius: 0.4444444444rem;
padding-right: 0.5555555556rem;
margin: 0.1666666667rem;
background-clip: padding-box;
color: #FFFFFF;
```

Barwy tła per yield: złoto `rgba(255,235,75,.3)`, zadowolenie `rgba(253,175,50,.3)`,
nauka `rgba(50,151,255,.3)`, kultura `rgba(197,75,255,.3)`, produkcja
`rgba(204,118,52,.34)`, wpływ/dyplomacja `rgba(88,192,231,.3)`, reszta
`rgba(228,228,228,.3)`. Przy najechaniu ten sam kolor z alfą `.5`.

## Odczyt bieżących yieldów osady (w tym zadowolenia) ✅

**2026-08-10.** `CommerceCityResourceData.yieldDeltas` to tablica `YieldDeltaProps`
z polami **`yieldIconSrc`**, **`yieldTotal`** i `yieldDelta`. Typu yieldu tam nie ma —
wyciąga się go z URL-a ikony:

```js
const type = String(entry.yieldIconSrc).match(/YIELD_[A-Z_]+/)?.[0] ?? null;
const total = Number(entry.yieldTotal) || 0;
```

`yieldTotal` to wartość pokazywana na karcie osady (np. `-10` zadowolenia), więc to
najprostsze źródło informacji „która osada jest nieszczęśliwa". ⚠️ `yieldDelta` to
podgląd **zmiany** przy najechaniu, nie stan — nie myl ich.

**Odświeżanie:** po każdej operacji przypisania model przelicza się sam
(zdarzenie `ResourceAssigned` + `UpdateGate`), więc algorytm działający w pętli powinien
czytać `yieldDeltas` **na nowo w każdym przebiegu**. To wystarcza, żeby rozdzielać zasoby
równomiernie: wybierasz zawsze aktualnie najgorszą osadę, a ona przestaje być najgorsza,
gdy tylko dostanie dość — bez żadnego planowania z góry.

## Ikony gry przydatne na tym ekranie ✅

**2026-08-10.** Ścieżki BLP wprost z `base-standard/data/icons/` — można ich użyć jako
`url(blp:…)` w stylu, tak jak robi to sama gra w `commerce-screen-empire-tab.tsx`:

| co | BLP | wygląd |
|---|---|---|
| klasa zasobu: miejski | `blp:restype_city_v2` | niebieski budynek |
| klasa zasobu: dodatkowy | `blp:restype_bonus_v2` | zielony listek z plusem |
| klasa zasobu: imperium | `blp:restype_empire_v2` | pomarańczowy heksagon |
| klasa zasobu: skarb | `blp:restype_treasure_v3` | złota skrzynka |
| flota skarbów | `blp:restype_treasure_v2` | |
| zasoby (ogólnie) | `blp:radial_resources` | zielony listek |
| szlaki handlowe | `blp:Action_Trade` | dwie strzałki |

Drugi sposób, którego używa mod **Trade Chooser Improvements** (Slothoth): atrybuty
`data-icon-id` + `data-icon-context` na elemencie, albo `UI.getIconCSS(id, context)`.
Konteksty potwierdzone przez tamten mod „na żywo":

```js
UI.getIconCSS('RESOURCECLASS_EMPIRE', 'RESOURCECLASS')
UI.getIconCSS('RADIAL_RESOURCES', 'DEFAULT')
UI.getIconCSS('YIELD_TRADES', 'YIELD')          // blp:Action_Trade
UI.getIconCSS('TRADE_ROUTE_LAND', 'TRADE')
UI.getIconCSS('CITY_YIELDS_HI', 'DEFAULT')
UI.getIconCSS('UNKNOWN_LEADER', 'LEADER')
```

⚠️ Ten sam `ID` bywa w kilku kontekstach z różnymi ścieżkami (np. `RESOURCECLASS_*`
występuje osobno dla mgły wojny, `Context=FOW`). Bez podania kontekstu można dostać
nie tę grafikę.

## Pasek zakładek — podmiana napisów na ikony ✅

`[data-name="TabList"]` zawiera `[data-name="TabListItem"]` **w kolejności deklaracji**
z `commerce-screen.tsx` (Zasoby, Szlaki, Imperium, Skarb — ten ostatni tylko w epoce
Odkryć). Na elemencie zakładki nie ma nic, co mówi, którą jest — **pozycja jest jedyną
tożsamością**.

❗ **KOREKTA: `font-size: 0` na zakładkach NIE działa.** Etykiety mają klasę
`font-fit-shrink`, czyli koherentowe `coh-font-fit-mode: shrink` — silnik sam dobiera
rozmiar tego tekstu i ignoruje zadeklarowany. Trzeba **usunąć węzły tekstowe**:

```js
for (const node of Array.from(item.childNodes)) {
    if (node.nodeType === 3) { label += node.nodeValue; item.removeChild(node); }
}
```

Usuwaj **wyłącznie węzły tekstowe** — ikona, którą dokładasz, jest elementem i ma zostać.
Tytuły zakładek są statyczne, więc Solid ich nie odtwarza, ale obserwator i tak powinien
to powtarzać na wszelki wypadek. Zdjęty tekst zachowaj na tooltip.

⚠️ Pasek zakładek żyje dłużej niż pojedyncza zakładka. Jeśli podpinasz się z komponentu
jednej z nich, **nie sprzątaj ikon w jej `onCleanup`** — zniknęłyby przy przejściu na inną
zakładkę. Podepnij obserwator do samego paska: gdy ekran się zamyka, element przepada
i obserwator przestaje cokolwiek dostawać, bez potrzeby sprzątania.


## Przyciski „cofnij przydzielenia" — gdzie są ✅

| co | gdzie | wywołanie |
|---|---|---|
| całe imperium | na samym dole kolumny osad, `div.self-end` wewnątrz `[data-name="slotted-resource-container"]` | `model.clearAllResources()` |
| jedna osada | po prawej stronie karty osady, `.fxs-image-button` | `model.clearAllResources(cityID)` |

Oba to `ReturnResourceButton`, czyli `ImageButton` z grafiką
`blp:resource_return_button_default.png` (najechanie: `..._hover.png`) i klasą
**`fxs-image-button`** — to najwygodniejszy uchwyt, bo `data-name` tam nie ma.

Ten dla imperium jest dodatkowo owinięty w `ConfirmationDialog`; ten dla osady nie.
Przenosząc którykolwiek gdzie indziej **ukryj oryginał i wystaw własny przycisk**
wołający tę samą metodę — przenoszenie poddrzewa zarządzanego przez Solida biłoby się
z tym, co je odtwarza.

⚠️ Ukrywanie powtarzaj przy każdym przebiegu obserwatora: karta osady jest przebudowywana
i przywraca oryginał.

Gotowa etykieta z nazwą osady:
`Locale.compose('LOC_COMMERCE_UNASSIGN_RESOURCES', Locale.compose(city.name))`.

## Zakładki a epoka — nie ma piątej kategorii ✅

**2026-08-10.** W `commerce-screen.tsx` są **dokładnie cztery** `Tab.Item`, a jedynym
warunkiem jest epoka przy „Skarbie":

| epoka | widoczne zakładki |
|---|---|
| Antyk | Zasoby, Szlaki handlowe, Imperium |
| Odkrycia | Zasoby, Szlaki handlowe, Imperium, **Skarb** |
| Nowożytność | Zasoby, Szlaki handlowe, Imperium |

❗ **Zasoby fabryczne NIE mają własnej zakładki.** Pojawiają się jako **podsekcja
w kolumnie nieprzydzielonych** (`AvailableResourceSubSection` o `type =
"RESOURCECLASS_FACTORY"`, tytuł `LOC_RESOURCECLASS_FACTORY_NAME`) oraz jako rozwijana
lista przy osadzie z fabryką (`FactoryTypeDisplay`). Żaden DLC nie nadpisuje tego ekranu.

⚠️ Wniosek dla mapowania ikon zakładek po indeksie: 0/1/2 to zawsze Zasoby/Szlaki/Imperium,
a 3 istnieje wyłącznie w epoce Odkryć — więc indeksy nie przesuwają się między epokami.

## Etykiety zakładek bywają zapisane WERSALIKAMI w lokalizacji ⚠️

`LOC_COMMERCE_TRADE_ROUTE_TAB` to w danych dosłownie `TRADE ROUTES`, nie „Trade Routes".
To nie `text-transform` w CSS — sam tekst jest wielkimi literami. Jeśli przenosisz taką
etykietę gdzie indziej (np. do tooltipa), trzeba ją złagodzić samemu:

```js
if (text === text.toUpperCase() && text !== text.toLowerCase()) {
    const lower = Locale.toLower(text);            // ⚠️ Locale.toLower, nie toLowerCase
    text = lower.charAt(0).toUpperCase() + lower.slice(1);
}
```

Warunek `text !== text.toLowerCase()` jest po to, żeby nie ruszać pism bez wielkości
liter (chiński, japoński, koreański), gdzie obie wersje to ten sam string.

## Zasoby zwiększające produkcję jednostek ✅

Efekty `*_ADJUST_UNIT_PRODUCTION_*` przypięte do zasobu są kwalifikowane **dokładnie
jednym** z trzech argumentów — i to jedyny sposób, żeby odróżnić wojsko od reszty:

| argument | znaczenie | przykłady |
|---|---|---|
| `Domain` = `DOMAIN_LAND` / `DOMAIN_SEA` | jednostki **bojowe** | Bawełna, Twarde drewno, Cytrusy |
| `UnitClass` = `UNIT_CLASS_NON_COMBAT` | osadnicy itp. | Twarde drewno (nowożytność) |
| `UnitTag` = `UNIT_CLASS_RELIGIOUS` | misjonarze | Kadzidło |

Zasoby z takimi modyfikatorami na 2026-08-10: `RESOURCE_CITRUS`, `RESOURCE_COTTON`,
`RESOURCE_HARDWOOD`, `RESOURCE_INCENSE`, `RESOURCE_SALT`, `RESOURCE_TRUFFLES`.

⚠️ Nie zaszywaj tej listy w kodzie — czytaj `ModifierMetadatas` (`FieldName="ResourceType"`)
→ `ModifierArguments` → `Modifiers` → `DynamicModifiers.EffectType`, tak samo jak przy
odczycie efektów yieldowych. Ten sam zasób bywa w różnych epokach różnie skonfigurowany
(Twarde drewno: morskie w antyku i odkryciach, cywilne w nowożytności), więc cache
kluczuj po `Game.age`.

## Dodanie własnej zakładki do ekranu ✅

**2026-08-10.** `Tab.Item` są **dziećmi `CommerceScreen`**, wypisanymi wprost w jego JSX.
Framework nie daje żadnego sposobu, żeby dorzucić kolejną z zewnątrz — nie ma rejestru
zakładek ani API na `TabContext`. Jedyna droga to **zarejestrować własny `CommerceScreen`**
z wyższym `overridePriority` i odtworzyć w nim całe drzewo.

Czyli jedyne miejsce w modzie, gdzie się **zastępuje**, a nie owija. Praktyczne wnioski:

- przepisz komponent gry **linia w linię**, dokładając tylko swoje — po patchu można
  wtedy zrobić diff z oryginałem i zobaczyć, co doszło;
- zachowaj `styles: [screenStyle]` przy rejestracji (import
  `commerce-screen.scss.js`), inaczej ekran straci własny arkusz;
- pisz w postaci skompilowanej Solida (`createComponent`, gettery na propsach) — mod nie
  ma buildu, więc JSX odpada. **Gettery są konieczne**: zwykła wartość odczyta model raz
  i nigdy się nie odświeży;
- `model.onTabChanged` to `switch` po `tab.name` **bez `default`**, więc nowa nazwa
  zakładki niczego nie psuje — po prostu nie wywołuje resetu.

Ikony zakładek mapowane po indeksie muszą wtedy **zależeć od epoki**: czwarte miejsce to
Skarb w Odkryciach, a np. własna zakładka w Nowożytności. Sztywna tablica wstawi skrzynkę
skarbów na cudzej zakładce.

## Audyt zasobów: co mówią dane, a co zakłada kod ✅

**2026-08-10.** Przeskanowałem wszystkie **111 wystąpień zasobów** (wszystkie zasoby ×
wszystkie epoki) i porównałem dane gry z założeniami algorytmu przypisań. Skrypt audytu:
`mod-projects/better-commerce-screen-ui` → historia sesji; wyniki poniżej są trwałe.

### 1. Modyfikator nie zawsze jest w `ModifierMetadatas` ❗

Powiązanie zasób ↔ modyfikator bywa zapisane na **dwa sposoby**:

```xml
<!-- pośrednio, tabelą metadanych -->
<ModifierMetadatas>
    <Row ModifierId="MOD_TRUFFLES_UNIT_PRODUCTION" FieldName="ResourceType" String="RESOURCE_TRUFFLES"/>
</ModifierMetadatas>

<!-- bezpośrednio, argumentem samego modyfikatora -->
<Modifier id="MOD_NICKEL_CITY_SCIENCE" effect="EFFECT_CITY_ADJUST_YIELD_PER_RESOURCE">
    <Argument name="ResourceType">RESOURCE_NICKEL</Argument>
</Modifier>
```

⚠️ Kto czyta tylko `ModifierMetadatas`, **nie zobaczy nic** dla: **Niklu** (nowożytność,
2 modyfikatory), jednego modyfikatora **Gipsu** (antyk) oraz `GOLD_DISTANT_LANDS` /
`SILVER_DISTANT_LANDS`. Nikiel wychodził wtedy jako zasób bez żadnych dochodów.

**Czytaj obie drogi** i scal w jeden indeks.

### 2. Modyfikatory mają warunki i przeważnie znaczą „tylko miasta" ❗

Rozkład `SubjectRequirements` przy modyfikatorach zasobów:

| warunek | wystąpień |
|---|---|
| `REQUIREMENT_CITY_HAS_BUILD_QUEUE` | 29 |
| `REQUIREMENT_CITY_IS_DISTANT_LANDS` | 22 |
| `REQUIREMENT_CITY_HAS_BUILDING` | 12 |
| `REQUIREMENT_CITY_IS_TOWN` | 10 |
| `REQUIREMENT_UNIT_TAG_MATCHES` | 9 |
| `REQUIREMENT_CITY_IS_CITY` | 6 |
| `REQUIREMENT_CITY_IS_CAPITAL` | 3 |
| `REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE` | 3 |

❗ **`CITY_HAS_BUILD_QUEUE` to gra-owy sposób na napisanie „tylko miasta"** — miasteczko
nie ma kolejki produkcji. Tak bramkowane są duże bonusy: Jade +10 złota, Jedwab
+10 kultury, Lapis Lazuli +4 produkcji i +10 złota, Goździki +10 złota, Kadzidło
+10 nauki. Algorytm, który ich nie sprawdza, przypisze te zasoby do miasteczka, gdzie
nie dadzą **niczego**.

Inne zasoby dzielą się wprost po typie osady — i wtedy wzięcie mniejszej z dwóch
wartości (co robi kod grupujący warianty przez `Math.min`) zaniża obie:

| zasób | miasto | miasteczko |
|---|---|---|
| Cyna (antyk) | +2 produkcji | +4 produkcji |
| Dziczyzna (antyk) | +2 żywności | +4 żywności |
| Dziczyzna (odkrycia) | +3 żywności | +6 żywności |
| Kauri | +4/5/6 złota | +2/3/4 nauki |

**Ścieżka złączenia** (wszystko w `GameInfo`):
`Modifiers.SubjectRequirementSetId` → `RequirementSetRequirements` → `Requirements`
(`RequirementType`, `Inverse`) → `RequirementArguments`.
`RequirementSets.RequirementSetType` mówi ALL czy ANY (domyślnie `REQUIREMENTSET_TEST_ALL`).

⚠️ Warunku, którego nie umiesz ocenić, **uznawaj za spełniony**. Zbyt gorliwe liczenie
psuje trochę punktację; zbyt surowe wycina zasób z rozważań całkowicie i nikt tego nie
zauważy.

### 3. Ręczna tabela warunków w Resource+ jest ODWRÓCONA ❗

Mod Resource+ trzyma listę nazw zasobów per epoka zamiast czytać warunki. Ta lista
**przeczy danym**:

- antyk, gips/kaolin/perły — dane: bonus wymaga `CITY_IS_CAPITAL`; kod: siła 1, **gdy
  osada NIE jest stolicą**;
- odkrycia, przyprawy/cukier/herbata — dane: `CITY_HAS_BUILD_QUEUE + CITY_IS_DISTANT_LANDS`;
  kod: `!isDistantLands && !isTown`;
- kakao — dane: `CITY_IS_TOWN + CITY_IS_DISTANT_LANDS`; kod: `!isDistantLands && isTown`.

Do tego **31 zasobów z warunkami w danych w ogóle nie ma w tej tabeli**.

### 4. Pozostałe ustalenia ✅

- **Sloty dodatkowe daje wyłącznie wielbłąd**, i to tylko w antyku i epoce odkryć —
  w nowożytności `BonusResourceSlots > 0` nie ma żadnego zasobu.
- **Skalowanie magazynami**: glina, kraby, żółwie (antyk i odkrycia), same kraby
  w nowożytności — rozpoznawalne po `Tag = WAREHOUSE`.
- **Bonus do produkcji jednostek** ma 8 wystąpień: Twarde drewno (3 epoki), Sól,
  Kadzidło, Trufle, Cytrusy, Bawełna (nowożytność). ⚠️ Każdy z nich ma też **+1 dochodu
  bazowego** (np. Sól +1 żywności, Bawełna +1 złota) — spychając je na koniec kolejki,
  świadomie rezygnujesz z tego +1.
- `REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE` (Wino, Futra) zależy od stanu gry, nie osady —
  nie da się go sensownie ocenić przy planowaniu przypisań.

## `ResourceSlotData.yieldTypes` NIE pochodzi z dochodów zasobu ❗✅

**2026-08-10.** Pole `yieldTypes` przy zasobie model buduje z **`GameInfo.TypeTags`**, a nie
z `Resource_YieldChanges`:

```js
const yieldTagTypes = new Map([
    ['FOOD','YIELD_FOOD'], ['PRODUCTION','YIELD_PRODUCTION'], ['GOLD','YIELD_GOLD'],
    ['SCIENCE','YIELD_SCIENCE'], ['CULTURE','YIELD_CULTURE'], ['HAPPINESS','YIELD_HAPPINESS'],
]);
GameInfo.TypeTags.forEach((t) => { /* Type == ResourceType, Tag == FOOD/PRODUCTION/... */ });
```

Czyli zasób otagowany `PRODUCTION` „dotyczy produkcji" niezależnie od tego, czy ma płaski
dochód z produkcji. ⚠️ Zwróć uwagę, że **wpływu (`YIELD_DIPLOMACY`) na tej liście nie ma** —
to nie przeoczenie, tylko stan mapy w grze.

Kosztowało to całą rundę: przy odtwarzaniu danych modelu poza ekranem zostawiłem
`yieldTypes: []`, a kod planujący ma fallback

```js
if (resource.yieldTypes?.length) return resource.yieldTypes;
// w przeciwnym razie Resource_YieldChanges
```

więc **nic się nie zepsuło — po prostu wyszedł inny wynik**. Ten sam algorytm dawał inny
układ zasobów przy zamkniętym ekranie niż przy otwartym, a ratowanie zadowolenia się nie
uruchamiało.

**Reguła:** odtwarzając struktury modelu poza jego kontekstem, każde pole buduj **tą samą
drogą co model**, a nie „byle było". Pola z cichym fallbackiem są najgorsze — nie rzucają
błędu, tylko po cichu zmieniają decyzje.

## Nagłówek karty osady — struktura i jak w nim coś umieścić ✅

Szablon `_tmpl$15` w `commerce-screen-resources-tab.js`:

```
<div class="flex flex-row flex-wrap relative w-full justify-between">
    <SettlementName/>          ← "flex flex-row items-center": herb, nazwa, plakietki
    <Show when={hasFactory}><FactoryTypeDisplay/></Show>   ← "flex flex-row items-center h-10"
</div>
```

`FactoryTypeDisplay` (`factory-type-display.js`) to zębatka `blp:restype_factory_v2.png`
plus czarna pigułka z aktualnym zasobem fabrycznym i przyciskiem zwrotu. Istnieje
**tylko w epoce nowożytnej** i tylko dla osad z fabryką.

**Dwie konsekwencje, obie widoczne w UI:**

- `justify-between` przy dwóch dzieciach odrzuca zębatkę do samej krawędzi karty —
  daleko od czegokolwiek, co się dołoży po prawej stronie.
- `flex-wrap` sprawia, że przy dużej liczbie plakietek (miasteczko z placówką handlową
  i stacją kolejową) **zębatka zawija się do drugiej linii**.

**Wstawianie własnych kontrolek — własny kontener, razem z zębatką.** Dwa podejścia
odpadły w praktyce:

- **absolutne** ze sztywnym odsunięciem pod zębatkę (`right: 6.25rem`) — nie zna
  szerokości plakietek ani tego, czy zębatka jest w tej samej linii;
- **w przepływie z `margin-left: auto`**, w nadziei że `justify-between` przyciągnie
  zębatkę — zębatka i tak została przy krawędzi.

Działa dopiero zabranie zębatki do własnego kontenera, gdzie sąsiedztwo nie zależy od
niczyjego układu:

```js
const header = card.querySelector('.flex.flex-row.flex-wrap.relative.w-full.justify-between');
header.classList.add(MY_HEADER_CLASS);

let actions = header.querySelector('.my-actions');
if (!actions) { actions = makeElement('div', 'my-actions'); header.appendChild(actions); }
// blok nazwy = pierwsze dziecko nagłówka, ale oznacz je Z JS, nie przez :first-child
for (const child of header.children) {
    if (child !== actions) { child.classList.add('my-name'); break; }
}
actions.appendChild(control);
// zębatka: po wewnętrznej czarnej pigułce, bo klasy zewnętrznego diva są generyczne
const factory = header.querySelector('.bg-black.rounded-lg')?.parentElement;
if (factory && factory.parentElement !== actions) actions.appendChild(factory);
```

```css
.my-header { flex-wrap: nowrap; }                      /* nagłówek nie łamie się nigdy */
.my-name { flex: 0 1 auto; min-width: 0;               /* zamiast tego zawijają się    */
           flex-wrap: wrap; overflow: hidden; }        /* plakietki wewnątrz nazwy     */
.my-header .text-xs.text-accent-1 { flex-wrap: wrap; }
.my-actions { display: flex; flex-wrap: nowrap; flex: 0 0 auto; margin-left: auto; }
.my-actions > .h-10 { flex: 0 0 auto; margin-left: 0.75rem; }   /* zębatka             */
```

⚠️ **Bloku nazwy nie da się złapać przez `:first-child`.** Dopóki żaden zasób nie jest
zaznaczony, nazwa jest opakowana w `Activatable` — pierwsze dziecko nagłówka i jego klasy
zależą od tego, co robi gracz. Oznaczenie klasą z JS jest jedynym pewnym sposobem.

⚠️ Przeniesienie zębatki to reparenting węzła Solid — dopuszczalne **tylko dlatego**, że
`hasFactory` nie zmienia się przy otwartym ekranie, a przebudowana karta i tak przechodzi
przez tę samą funkcję.

⚠️ `position: relative` na kontrolce, a nie `static` — rozwijane menu jest jej dzieckiem
i pozycjonuje się względem niej. Przy `static` zaczepiłoby się o nagłówek (ma `relative`)
i przeskoczyło w inne miejsce.

## Przycisk zwrotu osady vs. przycisk zwrotu fabryki ❗✅

W karcie osady są **dwa** `.fxs-image-button` z tą samą grafiką
`blp:resource_return_button_default.png`:

| gdzie | co robi | uwagi |
|---|---|---|
| w nagłówku, wewnątrz `FactoryTypeDisplay` | `model.clearFactoryResources(cityID)` | tylko przy fabryce |
| pod nagłówkiem, w `_tmpl$16` (klasa `mr-1`) | `model.clearAllResources(cityID)` | zawsze |

`card.querySelector('.fxs-image-button')` zwraca **ten z nagłówka**, bo nagłówek jest
wcześniej w DOM. Kod ukrywający „zwrot wszystkich zasobów osady" chował więc przycisk
fabryki, a właściwy zostawał na ekranie — przy osadach bez fabryki wszystko wyglądało
poprawnie, więc błąd ujawnił się dopiero w epoce nowożytnej.

```js
for (const button of card.querySelectorAll('.fxs-image-button')) {
    if (!header.contains(button)) button.classList.add(HIDDEN_CLASS);  // wszystko w
}                                                                      // nagłówku = fabryka
```

## Zasoby fabryczne — reguła gry ❗✅

Z pedii (`LOC_PEDIA_CONCEPTS_FACTORY_RESOURCES_TOOLTIP`, en_us):

> "Factory Resources must be assigned to a Settlement with a Factory to give empire-wide
> bonuses. **Only one type of Factory Resource can be assigned to a Settlement at a time.**"

i z opisu fabryki:

> "**You can assign multiple copies of the same Factory Resource to a Settlement**, so it
> pays to be efficient!"

**Konsekwencja dla algorytmu przypisywania:** rozkładanie „po jednym do każdej fabryki"
jest najgorszą możliwą strategią. Zajmuje każdą fabrykę innym rodzajem, a wtedy każda
nadmiarowa kopia ma dokładnie jedno legalne miejsce. Przy większej liczbie rodzajów niż
fabryk większość puli staje się nieprzypisywalna i w grze widać „przypisało tylko kilka
sztuk mimo 5 wolnych fabryk".

Właściwa kolejność:

1. **dokładaj do fabryki, która już coś produkuje** (tylko ten sam rodzaj jest legalny);
2. **pustą fabrykę zaczynaj od rodzaju z największą liczbą kopii w puli** — tego, którym da
   się ją zapełnić.

Dodatkowe fakty z modelu (`commerce-screen-model.js`):

- `factoryResourceData.hasFactory` **nie znaczy „ma budynek fabryki"**:
  `isTreasureConstructiblePrereqMet() && Game.age == AGE_MODERN && (getNumFactoryResources() == 0 || factoryResourceDefinition != null)`.
- `getFactoryResource()` zwraca **jeden** typ na osadę, `getNumFactoryResources()` liczy kopie.
- Zasoby fabryczne **zajmują zwykłe sloty**: `availableSlots = getAssignedResourcesCap() -
  getAssignedResources().length`, a `getAssignedResources()` zawiera także fabryczne.
- Fabryki można stawiać tylko w osadach w sieci kolejowej (stąd plakietka „Stacja kolejowa").

## Zakładka szlaków handlowych — struktura karty ✅

`trade-route-card.js`, komponent `TradeRouteCard` — **zarejestrowany w `ComponentRegistry`**,
więc da się go owinąć. ⚠️ Sam kontener zakładki (`TradeRoutesContainer` w
`commerce-screen-trade-tab.js`) to zwykła eksportowana funkcja — **nie jest** zarejestrowany,
więc jedynym sygnałem montowania tej zakładki jest karta.

Kolejność dzieci karty (`CardFrame`):

| szablon | co to jest | klasy |
|---|---|---|
| `_tmpl$` | nagłówek: herb + nazwa miasta źródłowego | `flex flex-row text-secondary uppercase text-lg mb-1 items-center`, nazwa w `.font-title` |
| `_tmpl$2` | „Dostarczono do osiedla X przez Morskie" | `p.mt-1.mr-13.font-fit-shrink` |
| `_tmpl$3` | zasoby przychodzące | `flex flex-row flex-wrap mt-4 mr-13` |
| `_tmpl$4` | „+24 do złota do <przywódca>" | **`<div class=mt-2>`** |
| `_tmpl$6` | portret przywódcy + zmiana relacji (`+10`) | `absolute top-1 right-1`, w środku `size-12 mt-2 …` |

⚠️ Ukrywając wiersz dochodów celuj **`[class="mt-2"]`** (dokładne dopasowanie atrybutu).
Zwykłe `.mt-2` trafia też w plakietkę relacji w rogu, która ma `mt-2` wśród wielu innych klas.

**Dane szlaku bierz z API, nie z tekstu.** Model podaje karcie tylko `domainString` —
gotowe, przetłumaczone zdanie złożone z `LOC_COMMERCE_TRADE_DELIVERED_TO`. Rozbieranie go
z powrotem pęknie w każdym języku o innym szyku. Źródłem jest to samo wywołanie, którego
używa model:

```js
const trade = Players.get(GameContext.localPlayerID)?.Trade;
const options = TradeRouteSearchOptions.INCLUDE_FAILED + TradeRouteSearchOptions.EXTENDED_STATUS;
trade.projectPossibleTradeRoutes(options).forEach((route) => {
    route.domain === DomainType.DOMAIN_LAND;   // lądowy czy morski
    Cities.get(route.targetCityId);            // miasto z nagłówka karty
    Cities.get(route.nearestCityId);           // nasza osada, do której trafiają zasoby
});
```

⚠️ To wywołanie jest **kosztowne** — to nim model buduje całą zakładkę. Cache'uj wynik i
unieważniaj na `TradeRouteAddedToMap` / `TradeRouteChanged` / `LocalPlayerTurnBegin`.

**Ikony domeny** są w `base-standard/data/icons/trade-icons.xml`: `TRADE_ROUTE_LAND`
(`blp:city_add`), `TRADE_ROUTE_SEA` (`blp:city_searoute`), plus `_WAR`, `_OUT_OF_RANGE`,
`_ALLIANCE`. Bierz przez `UI.getIcon('TRADE_ROUTE_SEA')` — tej samej pary używa kreator
szlaków (`trade-routes-model.js`, `getTradeRouteStatusIcon`).

**Linia instrukcji nad panelem** (`.text-base.w-full.text-center.my-4` w obrębie
`screen-resource-allocation`) jest na **każdej** zakładce. Regułę ukrywającą trzymaj w
arkuszu przypiętym do ekranu, nie do zakładki zasobów — inaczej wraca po przejściu na
szlaki handlowe.

### Podpowiedź stosunków — dlaczego jest wąska ❗✅

Podpowiedź spod portretu przywódcy (`base-standard/ui-next/tooltips/relationship-tooltip.js`,
korzeń ma `data-name="Relationship-Tooltip"` oraz klasy `fxs-tooltip fxs-relationship-tooltip`)
otwiera się tak wąska, że każdy powód zmiany stosunków łamie się na trzy linijki.

**Nic jej nie ogranicza od góry.** `Tooltip.Frame` ma `img-tooltip-border img-tooltip-bg
p-4 min-w-48` — sam dół, żadnego `max-width`. Przyczyna jest subtelniejsza: wszystkie
wiersze w środku mają `w-full`, a dziecko o szerokości procentowej **nie wnosi nic do
szerokości naturalnej rodzica**. Ramka spada więc do `min-w-72` z zawartości (ok. 16 rem)
i przy tej szerokości tekst się zawija.

Wystarczy podnieść podłogę — wiersze `w-full` same wypełnią to, co dostaną:

```css
[data-name="Relationship-Tooltip"] { min-width: 30rem; }
```

⚠️ Skala jednostek tego UI: `w-187` = 41,5555 rem, czyli **1 jednostka = 0,2222 rem**.
Stąd `min-w-72` ≈ 16 rem, a `min-w-48` ≈ 10,7 rem.

**Plakietka „+10" pod portretem** (zmiana stosunków) to `_tmpl$5` z `trade-route-card.js`,
bezpośrednie dziecko rogu `.absolute.top-1.right-1`, rozpoznawalna po `.size-12`.

## Zasoby imperialne — jak policzyć FAKTYCZNY efekt ❗✅

Karta w zakładce imperium pokazuje regułę zasobu („+1 do złota i zadowolenia we wszystkich
osiedlach"), niezależnie od tego, ile masz osiedli i ile kopii zasobu. Żeby policzyć sumę,
trzeba wiedzieć, **z czym skaluje się dany efekt** — a to jest cecha efektu, nie zasobu.
Nazwy efektów mówią to wprost:

| efekt | skaluje się z | przykład |
|---|---|---|
| `EFFECT_CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE` | **liczbą osiedli**, NIE liczbą kopii | `MOD_GOLD_SETTLEMENT_FLAT_GOLD`, Amount 1 → 12 osiedli = +12 |
| `EFFECT_UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE` | **liczbą kopii**, z limitem | `MOD_NITER_…`, Amount 1, 10 kopii → +6 (limit) |
| `EFFECT_CITY_ADJUST_YIELD_PER_RESOURCE` | kopiami × osiedlami spełniającymi wymagania | `MOD_IVORY_CITY_FLAT_HAPPINESS`, Amount 4 |
| `EFFECT_ADJUST_PLAYER_YIELD_PER_SLOTTED_RESOURCE` | liczbą przypisanych kopii | `MOD_COCOA_EXCESS_HAPPINESS` |

⚠️ **`PER_AVAILABLE_RESOURCE_TYPE` to nie to samo co `PER_RESOURCE`.** Pierwsze liczy
posiadanie *typu* (raz), drugie każdą *kopię*. Pomylenie ich daje przy sześciu sztukach
złota i dwunastu osiedlach 72 zamiast 12.

⚠️ **Limit +6 dla siły bojowej nie istnieje w danych.** Nie ma go w argumentach modyfikatora,
nie ma pliku `globalparameters.xml`, nie ma tabeli — trzyma go silnik, a w danych jest tylko
w treści opisu („maksymalnie +6"). W modzie jest to stała z komentarzem; to jedyna liczba
w kalkulatorze, która nie pochodzi z gry.

**Wymagania modyfikatorów trzeba honorować** przy liczeniu zasięgu: „w ojczyźnie",
„tylko miasta" itd. zawężają liczbę osiedli — używamy tego samego ewaluatora, co punktacja
przypisań (`planner/effects.js`, `modifierApplies`).

**Dane karty** dostarcza model (`populateEmpireResources`): `type`, `amount`, `iconSrc`,
`title` (klucz `Name`), `description` (klucz `Tooltip`), `originLeaderIds` oraz
`tooltips[leaderId]` — gotowe stringi „ile z którego miasta". Klasy zasobów w tej zakładce:
`RESOURCECLASS_EMPIRE` **i** `RESOURCECLASS_TREASURE`.

**Podmiana zakładki:** `EmpireResourceContainer` (jak `TradeRoutesContainer`) **nie jest**
zarejestrowany w `ComponentRegistry`. Ponieważ mod i tak zastępuje cały `CommerceScreen`
(patrz „Dodanie własnej zakładki"), własny komponent wstawia się po prostu w `body:`
odpowiedniego `Tab.Item`. Tło karty jak w zakładce szlaków daje klasa gry **`card-frame-bg`**.

### Sufiks efektu = reguła zliczania ❗✅

W plikach `resources-gameeffects.xml` te cztery warianty występują **obok siebie**,
wybierane modyfikator po modyfikatorze:

| sufiks | użyć | liczy |
|---|---|---|
| `PER_RESOURCE` | 62 | **każdą kopię** posiadaną przez imperium |
| `PER_AVAILABLE_RESOURCE_TYPE` | 29 | **typ raz**, niezależnie ile masz |
| `PER_RESOURCE_TYPE` | 3 | jw., na poziomie gracza |
| `PER_SLOTTED_RESOURCE` | 7 | tylko kopie **przypisane** do osad |

Gdyby `PER_RESOURCE` też znaczyło „raz", nie byłoby powodu, żeby istniał osobny
`PER_AVAILABLE_RESOURCE_TYPE`. Stąd:

- węgiel, `+10%` na stacje i porty przez `EFFECT_CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE`
  → przy 6 sztukach **+60%**;
- złoto, `+1` we wszystkich osiedlach przez `..._PER_AVAILABLE_RESOURCE_TYPE`
  → `+1 × liczba osiedli`, **bez** mnożenia przez liczbę sztuk.

⚠️ Argument `Empire="true"` **nie znaczy „raz na imperium"** — mówi, KTÓRE kopie liczyć
(wszystkie należące do imperium, a nie tylko przypisane do budującej osady). Mnożnik i tak
niesie sufiks efektu.

⚠️ Opisy zasobów tego nie zdradzają: są pisane ręcznie, w formie „+10% do produkcji…",
bez „za każdy". Nie da się z nich wyczytać reguły zliczania — trzeba iść do modyfikatora.

### Kolekcja modyfikatora zawęża zasięg tak samo jak wymagania ❗✅

Licząc efekt na całe imperium nie wystarczy sprawdzić `SubjectRequirements` — osobną
informacją jest **kolekcja**, czyli do kogo modyfikator w ogóle się stosuje. W plikach
zasobów występują cztery:

| kolekcja | użyć | mnożnik |
|---|---|---|
| `COLLECTION_ALL_CITIES` | 115 | liczba osad (po filtrze wymagań) |
| `COLLECTION_ALL_UNITS` | 10 | brak — dotyczy jednostek |
| `COLLECTION_ALL_PLAYERS` | 10 | 1 — raz na gracza |
| `COLLECTION_ALL_CAPITAL_CITIES` | 6 | 1 — tylko stolica |

⚠️ Futra (`MOD_FURS_FLAT_HAPPINESS`, +3 zadowolenia) używają `ALL_CAPITAL_CITIES`.
Policzone jak `ALL_CITIES` dają wynik przemnożony przez wielkość imperium.
Kolumna to `GameInfo.Modifiers[].CollectionType`.

### Tekst tooltipa ląduje w `#tooltip-root-content` ❗✅

`tooltip-manager.js` przekazuje kontrolerowi dwa elementy z `root-game.html`:
`tooltipRootElement: #tooltip-root` oraz `tooltipContentElement: #tooltip-root-content`.
Tekst trafia do gołego `<div>` doklejanego do **tego drugiego**. Selektor
`.tooltip__content` (kuszący, bo taka klasa istnieje w CSS) nie trafia w nic — reguła
`white-space: pre-wrap` musi celować w `#tooltip-root-content > div`.

### Klasy jednostek: LIGHT i HEAVY to klasy MORSKIE ❗✅

Tagów klas jednostek nic w grze nie wyświetla, więc łatwo je źle nazwać. Sprawdzone
w `age-modern/data/units.xml`:

| tag | jednostki |
|---|---|
| `UNIT_CLASS_LIGHT` | krążownik, niszczyciel, okręt pancerny → **lekkie okręty** |
| `UNIT_CLASS_HEAVY` | pancernik, drednot, fregata → **ciężkie okręty** |
| `UNIT_CLASS_NAVAL` | wszystko pływające |
| `UNIT_CLASS_RANGED` / `SIEGE` / `INFANTRY` / `CAVALRY` / `AIRCRAFT` | lądowe (RANGED obejmuje też okręty) |

⚠️ „lekkie" i „ciężkie" bez słowa „okręty" czyta się jako jednostki lądowe. Opisy gry
piszą to wprost („lekkich jednostek pływających"), więc etykiety moda też muszą.

### ⚠️ Opisy zasobów bywają NIEZGODNE z modyfikatorami

Sprawdzone maszynowo: dla każdego zasobu porównano tagi z `REQUIREMENT_UNIT_TAG_MATCHES`
z pojęciami linkowanymi w tekście `LOC_EXP_RESOURCE_*_TOOLTIP`.

**Saletra w epoce nowożytnej:**

- modyfikator `MOD_NITER_INFANTRY_AND_RANGED_COMBAT_STRENGTH`: `UNIT_CLASS_RANGED, UNIT_CLASS_SIEGE`
- polski opis: „jednostek oblężniczych, **dystansowych i ciężkich jednostek pływających**"

To jedyny modyfikator saletry w tej epoce (sprawdzone i po argumencie `ResourceType`,
i po `ModifierMetadatas`). Wersja z eksploracji miała `NAVAL, SIEGE` — wygląda na to, że
opis nie nadążył za zmianą efektu między epokami.

**Wniosek:** przy liczeniu efektów **źródłem prawdy są modyfikatory, nie opisy**. Opis
może wymieniać klasę jednostek, której modyfikator nie obejmuje.

### ⚠️ KOREKTA: `PER_AVAILABLE_RESOURCE_TYPE` JEDNAK skaluje się z liczbą sztuk

Wcześniejszy wpis w tym pliku twierdził, że złoto i srebro liczą się raz na imperium,
bo ich efekt nazywa się „per available resource TYPE". **To było błędne.**

**Pomiar w grze:** ulepszenie jednej dodatkowej kopii złota podniosło dochód o ok. tyle,
ile gracz ma osad. Przy interpretacji „raz na typ" nie powinno zmienić się nic.

Obie rodziny — `PER_RESOURCE` i `PER_AVAILABLE_RESOURCE_TYPE` — mnożą się przez liczbę
posiadanych kopii. Co rozróżnia ich nazwy, pozostaje nieznane; **nie jest to liczenie kopii**.

**Zasada ogólna:** nazwa w danych to hipoteza. Pomiar w działającej grze ją bije.
Zanim oprzesz kalkulację na konwencji nazewniczej, poproś o jedną obserwację z rozgrywki.

### ⚠️ KOREKTA 2: `PER_RESOURCE_TYPE` (wariant gracza) TEŻ skaluje się z liczbą sztuk

Ta sama pomyłka, tylko w drugim wariancie — i przeżyła o rundę dłużej, bo korekta wyżej
dotyczyła tylko `PER_AVAILABLE_RESOURCE_TYPE`. Objaw: **Wino [2]** pokazywało
`Jeden: +10 kultury` i `Wszystkie: +10 kultury`.

```xml
<Modifier id="MOD_WINE_GOLDEN_AGE_CULTURE"
          collection="COLLECTION_ALL_PLAYERS"
          effect="EFFECT_PLAYER_ADJUST_YIELD_PER_RESOURCE_TYPE">
    <SubjectRequirements><Requirement type="REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE"/></SubjectRequirements>
    <Argument name="Amount">10</Argument>
</Modifier>
```

**Reguła końcowa: wszystkie cztery sufiksy liczą sztuki.** Sufiks nie mówi nic o zliczaniu.
To, co je różni, to **zasięg** — a zasięg i tak siedzi w `collection`:

| efekt | zasięg | mnożnik |
|---|---|---|
| `CITY_ADJUST_YIELD_PER_RESOURCE` | osady z kolekcji | `kwota × osady × sztuki` |
| `CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE` | osady z kolekcji | `kwota × osady × sztuki` |
| `PLAYER_ADJUST_YIELD_PER_RESOURCE_TYPE` | `COLLECTION_ALL_PLAYERS` → **1** | `kwota × sztuki` |

Czyli w kodzie to jedna gałąź, nie trzy — różnicę załatwia funkcja licząca osady, bo dla
kolekcji `PLAYER` zwraca `1`.

Używają go dokładnie dwa zasoby: **Wino** (kultura, starożytność 5 / eksploracja 10) i
**Futra** (złoto, eksploracja) — oba tylko podczas Święta.

### Klasy jednostek NAKŁADAJĄ SIĘ — jeden okręt należy do czterech ❗✅

Sprawdzone w `age-modern/data/units.xml`, wszystkie tagi pancernika (`UNIT_BATTLESHIP`):

```
UNIT_CLASS_SIEGE, UNIT_CLASS_NAVAL, UNIT_CLASS_HEAVY, UNIT_CLASS_RANGED,
UNIT_CLASS_COMBAT, UNIT_CLASS_ELITE_NAVAL_HEAVY, UNIT_CLASS_AUTOEXPLORE
```

Dlatego ten sam okręt dostaje **i** premię z saletry (`RANGED, SIEGE`), **i** z ropy
(`CAVALRY, HEAVY`) — widać to w grze w rozbiciu siły bojowej („+6 do saletry, +5 do ropy").

⚠️ Konsekwencja dla wyświetlania: lista klas z modyfikatora **nie jest** rozłącznym
podziałem jednostek. Napis „dystansowe, oblężnicze" jest prawdziwy, ale gracz patrzący na
swój pancernik nie rozpozna, że jest w tym zbiorze. Opisy gry mówią o tym samym efekcie
innymi słowami („ciężkich jednostek pływających”) i **oba są zgodne z mechaniką** —
to nie jest sprzeczność, tylko dwa opisy tego samego zbioru okrętów.

### Jak wypisać klasy jednostek objęte efektem: test ZAWIERANIA ✅

Skoro klasy się nakładają, sama lista tagów z modyfikatora jest myląca. Rozwiązanie, które
daje pełny i prawdziwy zbiór:

1. zbierz jednostki mające **którykolwiek** z tagów modyfikatora → zbiór `covered`;
2. wypisz każdą nazwaną klasę `C`, dla której **wszystkie** jednostki z `C` należą do
   `covered` (zawieranie, nie przecięcie).

Wynik na danych epoki nowożytnej:

| zasób | tagi modyfikatora | wypisane klasy |
|---|---|---|
| saletra | RANGED, SIEGE | dystansowe, oblężnicze, **ciężkie okręty** |
| ropa | CAVALRY, HEAVY | kawaleria, ciężkie okręty |
| węgiel | LIGHT | lekkie okręty |
| kauczuk | AIRCRAFT, INFANTRY | lotnicze, piechota |

Saletra zyskuje „ciężkie okręty", bo **każdy** ciężki okręt w tej epoce jest też dystansowy
albo oblężniczy — dokładnie to, o czym mówi opis gry. „Wszystkie okręty" nie dochodzą, bo
lekkie okręty są morskie i nie są objęte; test zawierania nie pozwala obiecać klasy,
której efekt nie pokrywa w całości.

Źródło: `GameInfo.TypeTags` ograniczone do wierszy, których `Type` występuje
w `GameInfo.Units` (TypeTags trzyma tagi wszystkiego, także zasobów).

**Doprecyzowanie:** po teście zawierania trzeba jeszcze usunąć **połówki morskie**, gdy
pokryta jest cała klasa `UNIT_CLASS_NAVAL`. Inaczej w epoce eksploracji saletra
(`NAVAL, SIEGE`) wypisuje „ciężkie okręty, lekkie okręty, wszystkie okręty, oblężnicze",
co jest tym samym powiedzianym trzy razy.

⚠️ **Nie rób z tego ogólnej reguły „usuń klasę zawartą w innej".** W epoce nowożytnej
każdy ciężki okręt jest przypadkiem także dystansowy, więc reguła ogólna skasowałaby
„ciężkie okręty" z saletry — czyli tę jedną pozycję, której szuka gracz patrzący na swój
pancernik. Zawieranie między rolami (ciężkie ⊂ dystansowe) to przypadek składu jednostek
w danej epoce; zawieranie w obrębie floty (lekkie, ciężkie ⊂ okręty) to taksonomia.

Wynik po obu krokach:

| epoka | zasób | karta pokazuje |
|---|---|---|
| nowożytna | saletra | ciężkie okręty, dystansowe, oblężnicze |
| nowożytna | ropa | kawaleria, ciężkie okręty |
| eksploracji | saletra | wszystkie okręty, oblężnicze |

### ⚠️ `CollectionType` jest w `DynamicModifiers`, nie w `Modifiers`

Zapis w XML wygląda, jakby kolekcja była atrybutem modyfikatora:

```xml
<Modifier id="MOD_FURS_FLAT_HAPPINESS" collection="COLLECTION_ALL_CAPITAL_CITIES"
          effect="EFFECT_CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE">
```

W bazie rozkłada się to inaczej: `collection` **i** `effect` trafiają do
**`DynamicModifiers`**, kluczowane przez `ModifierType`. Wiersz w `Modifiers` ma tylko
`ModifierId` i `ModifierType`.

```js
const byType = new Map();
GameInfo.DynamicModifiers.forEach((e) => byType.set(e.ModifierType,
    { effect: e.EffectType, collection: e.CollectionType }));
GameInfo.Modifiers.forEach((e) => { /* e.ModifierType -> byType */ });
```

⚠️ `modifierRow.CollectionType` zwraca `undefined` **bez błędu**, więc kod czytający je
stamtąd po prostu cicho traci informację o zasięgu. U nas skutek: futra (+3 zadowolenia
**w stolicy**) były mnożone przez liczbę wszystkich osad — przy 12 osadach i 6 sztukach
karta pokazywała +216 zamiast +18.

### ⚠️ Karta szlaku: widoczny panel to DZIECKO, nie `.trade-route-card`

`trade-route-card.js`:

```js
const [local, cardFrameProps] = splitProps(props, ["tradeRoute", "autoFocus", "class", "onFocus"]);
const content = createComponent(CardFrame, mergeProps(cardFrameProps, { … }));
return createComponent(Activatable, { class: "focusable-card-activatable trade-route-card", children: content });
```

`style` **nie jest** na liście `splitProps`, więc zostaje w `cardFrameProps` i trafia na
**`CardFrame`**. To znaczy, że wyliczane przez zakładkę `width` i `margin-right` lądują na
wewnętrznej ramce — tej, którą widać — a `.trade-route-card` to tylko `Activatable` wokół
niej i gra **nigdy jej nie wymiarowuje**.

**Konsekwencja:** wszelkie nadawanie szerokości `.trade-route-card` nie zmienia wyglądu
panelu. Trzeba przypiąć dziecko:

```css
.trade-route-card > * {
    width: 100% !important;
    margin-right: 0 !important;   /* 12px dla każdej karty poza ostatnią w rzędzie */
    margin-left: 0 !important;
}
```

⚠️ To `margin-right` było widoczną nierównością kolumn: zakładka daje 12 px marginesu
każdej karcie **oprócz ostatniej w rzędzie**, więc trzecia kolumna wyglądała dokładnie
o tyle szerzej.


---

## Karta konwoju skarbowego — co da się zmienić bez przepisywania komponentu ✅

`base-standard/ui-next/screens/commerce/treasure-convoy-card.js`. Karta jest
`ComponentRegistry.register({ name: 'TreasureConvoyCard' })`, ale **rejestracja nie pomaga**
przy usuwaniu czegoś ze środka: `createInstance` buduje całą zawartość w jednym wyrażeniu,
więc nadpisanie z wyższym priorytetem to skopiowanie ~200 linii kodu gry. Dwie tańsze drogi:

### 1. Pola z modelu — podmieniamy dane, nie DOM ✅

Karta renderuje `props.fleet.treasureFleetText` przez `insert(_el$5, () => ...)`, więc
**cokolwiek** tam włożymy, to się narysuje. Model buduje to jako `L10n.Stylize({ text })`,
a `L10n.Stylize` robi `spread(_el$, mergeProps(other, { innerHTML }))` — czyli **dodatkowe
propsy lądują jako atrybuty na elemencie**. Stąd tooltip bez dotykania DOM:

```js
L10n.Stylize({
    text: `+${gold} [icon:YIELD_GOLD]   +${gdp} [icon:ECONOMIC_VP]`,
    'data-tooltip-content': Locale.compose('LOC_MOJ_KLUCZ'),
});
```

Surowe liczby **nie są** na obiekcie floty (model wkłada je od razu w zdanie). Odczyt
z osady:

```js
const resources = Cities.get(fleet.cityID)?.Resources;
resources.getProducedTreasureFleetGold();   // złoto za konwój
resources.getProducedTreasureFleetGDP();    // PKB za konwój
```

Token ikony PKB to `[icon:ECONOMIC_VP]`.

### 2. Nagłówek z `L10n.Compose` — CSS go NIE dosięgnie ❗✅

```js
createComponent(L10n.Compose, { text: "LOC_COMMERCE_TREASURE_RESOURCES_TITLE" })
```

`L10n.Compose` to `(props) => createMemo(() => Locale.compose(props.text ?? ''))` — zwraca
**goły węzeł tekstowy**, bez żadnego elementu. Nie ma selektora, który by go trafił, a
`MutationObserver` na kartach to dokładnie ta klasa rozwiązań, która wywołała zawieszenie
gry (quirk #57).

**Tańsze i bezpieczniejsze: wyzerować sam klucz lokalizacyjny** — o ile jest używany w
jednym miejscu (tu: dokładnie jedno wystąpienie w całej grze, sprawdzone `grep`iem po
`Base/`). Nadpisanie istniejącego klucza to `<Replace>`, nie `<Row>`:

```xml
<!-- text/en_us/…: angielski siedzi w <EnglishText>, BEZ atrybutu Language -->
<EnglishText>
    <Replace Tag="LOC_COMMERCE_TREASURE_RESOURCES_TITLE"><Text></Text></Replace>
</EnglishText>

<!-- text/pl_PL/…: reszta języków w <LocalizedText>, Z atrybutem Language -->
<LocalizedText>
    <Replace Tag="LOC_COMMERCE_TREASURE_RESOURCES_TITLE" Language="pl_PL"><Text></Text></Replace>
</LocalizedText>
```

⚠️ **Jeden wiersz na język.** Język bez takiego wiersza dalej widzi nagłówek — bo gra
trzyma tłumaczenia w `base-standard/l10n/<locale>_Text.xml` jako `<Replace … Language=…>`,
a nasz angielski wiersz ich nie przykrywa. Przy dodawaniu kolejnych lokalizacji do moda
trzeba ten wiersz powielić razem z resztą.

Pusty węzeł tekstowy w kontenerze `flex` nie zajmuje wysokości: `CardFrame` wstawia dzieci
**bezpośrednio** do elementu z klasami (`insert(_el$, () => props.children)`), a anonimowy
element flex bez treści się nie renderuje.


---

## Limit szlaków handlowych — API i pułapka ✅

Limit jest **per lider**, nie na imperium. Nie ma jednej liczby „ile szlaków mogę mieć":

```js
const trade = Players.get(GameContext.localPlayerID)?.Trade;

trade.countPlayerTradeRoutes();            // wszystkie nasze szlaki, łącznie
trade.countPlayerTradeRoutesTo(playerId);  // szlaki do TEGO gracza
trade.getTradeCapacityFromPlayer(playerId);// limit z TYM graczem
```

Sumę robimy po `Players.getAlive()` z filtrem `player.isMajor`, pominięciem siebie i
`Players.get(local).Diplomacy.hasMet(player.id)` — limit wobec nieznanego gracza to nie
jest miejsce, którego da się użyć.

⚠️ **Nie mieszać `countPlayerTradeRoutes()` z sumą `...To(id)`.** Pierwsze liczy też szlaki
poza tą sumą, więc nagłówek przestaje się zgadzać z rozpiską po liderach — a to rozpiskę
gracz może sprawdzić wzrokiem. Albo suma, albo nic.

⚠️ **Wolny limit ≠ szlak, który da się zawrzeć.** Zasięg i wojna blokują niezależnie od
limitu. Rozróżnienie bierzemy z projekcji, tej samej, której używa model:

```js
const options = TradeRouteSearchOptions.INCLUDE_FAILED + TradeRouteSearchOptions.EXTENDED_STATUS;
trade.projectPossibleTradeRoutes(options).forEach((route) => {
    if (route.status?.includes(TradeRouteStatus.SUCCESS)) {
        startable.add(Cities.get(route.targetCityId)?.owner);   // lider, z którym można OD RAZU
    }
});
```

`SUCCESS` znaczy „wszystkie kryteria spełnione, zostaje podpisać".

---

## Miejsce na własne podsumowanie nad zakładkami ✅

Rodzic `[data-name="TabList"]` jest `position: relative`, a sam pasek zakładek jest w nim
wyśrodkowany — więc lewa strona stoi pusta i nie trzeba nic mierzyć:

```css
position: absolute; left: 2rem; top: 0.15rem; z-index: 20;
```

Tego samego kotwiczenia używają: przyciski „Przypisz wszystkie" (zakładka zasobów),
podsumowanie dochodów (imperialne), podsumowanie szlaków (handel) i znak „?" (skarbowe).
**Nie kolidują**, bo w danej chwili otwarta jest tylko jedna zakładka — ale każdy z tych
modułów musi sprzątać po sobie w `onCleanup`, bo **rząd należy do ekranu, nie do zakładki**
i przeżywa wyjście z niej.

⚠️ Jeśli dokładanie elementu leci z `MutationObserver`, musi być **idempotentne** — patrz
quirk #57. Wzorzec: flaga „już pokazane" **plus** sprawdzenie, czy element wciąż jest
w drzewie (ekran potrafi przebudować rząd sam z siebie):

```js
if (shown && document.querySelector(`.${CLASS}`)) return;
```

---

## Kliknięcie karty w miastach skarbowych NIE zamyka ekranu ✅

Wszystkie trzy klikalne miejsca na karcie konwoju wołają wyłącznie `Camera.lookAtPlot`:

| kliknięcie | handler modelu | efekt |
|---|---|---|
| nazwa/baner osady | `handleClickCityName` | `Camera.lookAtPlot(city.location)` |
| ikona zasobu | `handleClickUnimprovedTreasure` | `Camera.lookAtPlot(location)` |
| flaga konwoju | `handleClickTreasureFleet` | `Camera.lookAtPlot(unit.location)` |

Czyli mapa jedzie **pod** wciąż otwartym ekranem i gracz tego nie widzi, dopóki sam go nie
zamknie. To nie jest błąd do naprawienia z moda (zamykanie ekranu za gracza zmieniłoby
zachowanie gry) — to rzecz do wyjaśnienia w tooltipie.


---

## Zasoby fabryczne — modyfikatory i sposób liczenia ❗✅

W nowożytności jest ich osiem: **kawa, cytrusy, bawełna, kakao, herbata, kaolin, chinina,
cyna** (`ResourceClassType = RESOURCECLASS_FACTORY`). Każdy daje procent, i **tylko wtedy,
gdy siedzi w osadzie z fabryką** — nieprzypisany nie daje nic.

| zasób | efekt | argumenty |
|---|---|---|
| kawa | `CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_SLOTTED_RESOURCE` ×2 | `Amount=5`, `ConstructibleClass` = BUILDING / WONDER |
| cytrusy | `CITY_ADJUST_UNIT_PRODUCTION_PER_SLOTTED_RESOURCE` | `Percent=5`, `Domain=DOMAIN_SEA` |
| bawełna | to samo | `Percent=5`, `Domain=DOMAIN_LAND` |
| kakao | `ADJUST_PLAYER_YIELD_PER_SLOTTED_RESOURCE` | `Amount=3`, `YIELD_HAPPINESS` |
| herbata | to samo | `Amount=3`, `YIELD_SCIENCE` |
| kaolin | to samo | `Amount=3`, `YIELD_CULTURE` |
| chinina | `UNIT_ADJUST_HEAL_PER_RESOURCE` | `Amount=1`, `COLLECTION_ALL_UNITS` |
| cyna | `CITY_ADJUST_GROWTH_PER_RESOURCE` | `Percent=3`, **`GlobalSlots=true`** |

### ⚠️ Trzy pułapki, przez które kod od zasobów imperialnych czyta tu same zera

1. **Liczba bywa w `Percent`, nie w `Amount`.** Cytrusy, bawełna i cyna. `Number(map.get('Amount'))`
   daje `NaN` i wcześniejszy `return` wycina cały modyfikator.
2. **`ConstructibleClass`, nie `ConstructibleType`.** Odczyt nazwy budynku przez
   `GameInfo.Constructibles.lookup()` nic nie zwróci — to klasa (`BUILDING`/`WONDER`),
   nie konkretny budynek.
3. **NIE mnożymy przez liczbę osad.** `PER_SLOTTED_RESOURCE` i `GlobalSlots=true` znaczą, że
   liczy się **całkowita liczba wpiętych sztuk w imperium, raz**, a procent stosuje się tam,
   gdzie mówi kolekcja. Cztery kawy = **+20% w każdej osadzie**, nie +20% na osadę.
   To jest **odwrotnie** niż agregują się zasoby imperialne — pomyłka zawyża liczby o
   wielkość imperium.

### ⚠️ ROZSTRZYGNIĘTE: bonusy fabryczne są GLOBALNE, nie tylko w miastach na kolei ✅

Wątpliwość była uzasadniona — kolej pojawia się w opisach fabryk na tyle często, że łatwo
uznać, iż ogranicza też zasięg premii. **Nie ogranicza.** Trzy niezależne dowody:

1. **Tutorial gry, `LOC_TUTORIAL_FACTORY_RESOURCES_BODY`:** *„To slot a Factory Resource
   into a Settlement, it must have a Rail Connection and a Factory. (…) **Once the Resource
   has been slotted it becomes an Empire Resource.**"* — kolej jest warunkiem **wpięcia**,
   a wpięty zasób działa jak imperialny, czyli wszędzie.
2. **Wymaganie kolei siedzi na BUDYNKU, nie na zasobie.** Opis `BUILDING_FACTORY`: *„Must be
   built in a Settlement connected to the Capital by Railroad."* Bramka jest na tym, gdzie
   wolno postawić fabrykę — nie na tym, gdzie działa premia.
3. **Żaden z dziewięciu modyfikatorów fabrycznych nie ma ANI JEDNEGO `<Requirement>`.**
   Sprawdzone `grep`iem po obu plikach `resources-gameeffects*.xml` w `age-modern`.
   Kolekcje to `ALL_CITIES` / `ALL_PLAYERS` / `ALL_UNITS` — czyli wszystko.

Czyli: liczymy wpięte sztuki w całym imperium i stosujemy procent wszędzie. Bez filtrowania
osad po sieci kolejowej.

### Odczyt stanu

```js
// wpięte: JEDEN rodzaj na osadę, dowolnie wiele sztuk tego rodzaju
city.Resources.getFactoryResource();       // typ (albo brak)
city.Resources.getNumFactoryResources();   // ile sztuk w tej osadzie

// posiadane: JEDEN wpis = JEDNA SZTUKA (tak samo liczy model gry w populateEmpireResources)
Players.get(local).Resources.getResources()
    .filter((r) => GameInfo.Resources.lookup(r.uniqueResource.resource)
                       ?.ResourceClassType === 'RESOURCECLASS_FACTORY');

Game.Resources.getOriginCity(resource.value);   // skąd pochodzi ta sztuka
```

Nieprzypisane = posiadane − wpięte. Liczyć różnicą, nie osobnym odczytem — inaczej te dwie
sekcje potrafią pokazać sumę niezgodną z tym, co gracz faktycznie ma.

### PKB z wpiętych zasobów — to NIE jest modyfikator ✅

Nie ma go w `resources-gameeffects.xml` i nie znajdzie go żaden kod chodzący po
modyfikatorach. Siedzi w tabeli **`VictoryScorings`**:

```xml
<Row VictoryType="VICTORY_ECONOMIC_MODERN"
     TrackerType="VICTORY_TRACKER_SLOTTED_RESOURCE_CLASS"
     ScoringId="VICTORY_TRACKER_SLOTTED_FACTORY"
     Points="3" Data="RESOURCECLASS_FACTORY" RequiresActivation="true" StaticCarryover="true"/>
```

```js
GameInfo.VictoryScorings.find((r) => r.ScoringId === 'VICTORY_TRACKER_SLOTTED_FACTORY').Points;
```

⚠️ **Czytać z tabeli, nie wpisywać `3` na sztywno** — to dokładnie ta liczba, którą rusza
patch balansowy, a zapisana w kodzie wyglądałaby dalej dobrze będąc już błędną.

⚠️ **Są też inne wiersze doliczające PKB za te same zasoby**, np.
`VICTORY_TRACKER_FACTORY_RESOURCES_INDUSTRIAL_PARK` (`Points="1"`) — amerykańska unikalna
dzielnica Park Przemysłowy. Wykrycie jej wymaga chodzenia po polach osady, więc mod jej nie
liczy, ale **mówi o tym w tooltipie** zamiast po cichu pokazywać zaniżoną liczbę.

Uwaga na sprzeczne teksty gry: `AdvisorText` mówi „2 points of GDP", `VictoriesText` „+3
GDP". Tabela mówi 3 — i to ona jest źródłem prawdy.

### Ikony spoza `UI.getIcon(yield)`

```
tempo wzrostu   blp:fi_growth_rate_64
leczenie        blp:fi_action_heal_64
PKB             blp:fi_victorypoint_economic_64
```
(z `base-standard/data/icons/text-icons.xml`, kontekst `FONTICON`)


---

## Opis przy liczbie: NIE skracać go — przenieść do własnej linii ❗✅

Objaw: w karcie z dwiema premiami („+10% do produkcji" + „+1 do siły") opis pierwszej
nachodził na drugą.

### Co NIE zadziałało

1. **`text-overflow: ellipsis`** — ten renderer nie uznaje „ściśnięty przez flexa" za
   **określoną** szerokość, więc element trzyma pełną naturalną szerokość i rysuje po
   sąsiedzie. Trzeba by nadać szerokość w pikselach z JS.
2. **Pomiar wolnego miejsca po layoucie** — napisany i wyrzucony, z dwóch powodów:
   - ⚠️ **`requestAnimationFrame` po `onMount` bywa ZA WCZEŚNIE.** Karta nie ma jeszcze
     swoich wymiarów, `getBoundingClientRect().width` wychodzi 0, wolne miejsce wychodzi
     zero i kod chowa **wszystkie** opisy. Dokładnie to stało się na żywo.
   - ⚠️ Nawet gdyby pomiar był dobry: **schowanie albo przycięcie opisu to utrata
     informacji, nie rozwiązanie.** „+6 [miecz]" bez słów nie mówi, do jakich jednostek —
     a to jedyne, po co ta linia istnieje. Tooltip nie ratuje, bo trzeba wiedzieć, że
     jest się czego najechać.

### Co zadziałało

**Opis wychodzi z linii liczb do własnej linii pod spodem** („legenda"), kluczowanej ikoną
tej premii:

```
Jeden:      +10% [produkcja]   +1 [miecz]
Wszystkie:  +60% [produkcja]   +6 [miecz]
[produkcja] Stacja kolejowa, Port
[miecz] lekkie okręty, ciężkie okręty
```

Trzy rzeczy naraz przestają boleć: opis ma **całą szerokość karty**, jest napisany **raz**
zamiast w obu wierszach, i wolno mu **zawijać** zamiast się ucinać. Zero pomiarów, zero
chowania.

⚠️ **Ikona jest kluczem tylko dopóki ikony się RÓŻNIĄ.** Na zakładce fabrycznej kawa,
cytrusy i bawełna mają wszystkie ikonę produkcji. Więc: jeśli w karcie dwie premie mają tę
samą ikonę, do linii legendy dochodzi też liczba.

Skoro w linii liczb nie ma już nic zbędnego, dostaje `flex-wrap: wrap` zamiast
`nowrap` + `overflow: hidden` — druga linijka kosztuje mniej niż nieczytelna liczba.

### ⚠️ Legenda TYLKO gdy premii jest więcej niż jedna

Przy **jednej** premii w karcie problemu nie ma — jedna liczba i jej opis mieszczą się
w linii, a to jest lepsze, bo **karty w rzędzie zostają tej samej wysokości**. Legenda
kosztuje dodatkową linijkę, więc płacimy za nią tylko tam, gdzie coś kupuje.

- zakładka imperialna: prawie każda karta ma dwie premie → legenda
- zakładka fabryczna: prawie każda ma jedną → opis w linii

---

## Karty w rzędzie mają różną wysokość ❗✅

Rząd `flex-wrap: wrap` **rozciąga** swoje dzieci (domyślne `align-items: stretch`), więc
`.card` faktycznie ma wysokość najwyższej w rzędzie. Ale `.card` to tylko dystansownik —
**widoczna ramka to element w środku**, a ten sam z siebie dostaje wysokość swojego tekstu:

```css
.card__inner { flex: 1 1 auto; }   /* bez tego ramka jest niższa niż karta */
```

Ta sama pułapka co przy panelach szlaków handlowych i kartach konwojów: element z
„oczywistą" klasą nie jest tym, co widać.

---

## Procent dochodu na liczby bezwzględne — pułapka podwójnego liczenia ❗✅

„+30% nauki" nic nie mówi bez wiedzy, ile się ma nauki. Dochód imperium na turę (ten z
górnego panelu):

```js
Players.get(GameContext.localPlayerID)?.Stats?.getNetYield(YieldTypes[yieldType]);
```

⚠️ **`net × procent / 100` ZAWYŻA wynik.** Dochód netto **już zawiera** premię od wpiętych
zasobów, a 30% liczy się od stanu **przed** sobą, nie po. Trzeba ją wycofać:

```
przed  = net / (1 + zastosowane/100)
warto  = przed × procent/100  =  net × procent / (100 + zastosowane)
```

gdzie `zastosowane` to **łączny** procent fabryczny dla tego dochodu ze wszystkich już
wpiętych sztuk (nie tylko z tego jednego zasobu). Ten sam mianownik obsługuje oba pytania:

- „ile z obecnej nauki daje herbata" → `procent` = to, co daje herbata
- „ile dołoży wpięcie leżących sztuk" → `procent` = to, co dołożą, a mianownik bez zmian

To jest dokładne, jeśli gra **mnoży** procenty, i przybliżone, jeśli **dodaje** je do
procentów z innych źródeł — czego z UI nie da się sprawdzić. Dlatego liczba idzie na ekran
jako **„≈"** i tooltip mówi wprost, że to szacunek.

**Da się policzyć tylko dla dochodów, które mają jedną liczbę w panelu** — nauka (herbata),
kultura (kaolin), zadowolenie (kakao). Reszta zasobów fabrycznych mnoży produkcję w stronę
konkretnej rzeczy albo tempo wzrostu; nie ma czego wziąć procent, a zgadywanie byłoby gorsze
niż zostawienie samego procentu.
