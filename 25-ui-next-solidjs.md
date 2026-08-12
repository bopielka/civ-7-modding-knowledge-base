# 25. `ui-next` — drugi framework UI (Solid.js)

**Data ustalenia: 2026-08-10.** Odkryte przy pracy nad modem „Better Commerce Screen UI".

## ⚠️ NAJWAŻNIEJSZE: Civ VII ma DWA frameworki UI, nie jeden

Wszystko, co opisuje [05-ui-javascript.md](05-ui-javascript.md) i
[09-cookbook-ui-mod.md](09-cookbook-ui-mod.md) (`Controls.define`, `Controls.decorate`,
`Component`, `data-bind-*`, `.html.js`), dotyczy **starego** frameworka w katalogach `ui/`.

Równolegle istnieje **nowy** framework w katalogach `ui-next/` — ✅ zweryfikowane w plikach:

| | stary `ui/` | nowy `ui-next/` |
|---|---|---|
| Podstawa | własny `Component` + custom elements | **Solid.js** (`core/vendor/solid-js`) |
| Widok | string HTML w `*.html.js` + `data-bind-*` | JSX kompilowany do `createComponent`/`template` |
| Reaktywność | `data-bind-value`, ręczne `update*()` | sygnały: `createSignal`, `createMemo`, `createEffect` |
| Rejestracja | `Controls.define('nazwa', …)` | `ComponentRegistry.register({name, createInstance})` |
| Model | klasa singleton + `engine.on` | `ModelRegistry.register(...)` / fabryka + `createContext` |
| Modowanie | `Controls.decorate` | **`overridePriority`** (patrz niżej) |
| Klasy CSS | te same utility (`flex`, `mt-4`, `text-secondary`) | te same |

Nowe ekrany Firaxis pisze już w `ui-next`. Na 2026-08-10 są tam m.in.:
`screens/commerce` (ekran Handlu), `screens/victories`, `screens/load-screen`,
`screens/age-transition`, `screens/choosers/tech-chooser`, `.../culture-chooser`.

**Praktyczny wniosek:** zanim zaczniesz pisać dekorator w starym stylu, sprawdź, czy
ekran nie został już przeniesiony do `ui-next/`. Jeśli został — stary plik w `ui/`
nadal leży na dysku i nadal jest ładowany, ale **nie jest tym, co widzi gracz**
(patrz „Który ekran wygrywa" niżej).

## Gdzie co leży ✅

```
Base/modules/core/vendor/solid-js/dist/solid.js         rdzeń Solid
Base/modules/core/vendor/solid-js/web/dist/web.js       renderer DOM
Base/modules/core/vendor/solid-js/store/dist/store.js   createMutable / createStore
Base/modules/core/ui-next/services/component-registry.js
Base/modules/core/ui-next/services/model-registry.js
Base/modules/core/ui-next/components/fxs-solid-component.js   most stary↔nowy
Base/modules/core/ui-next/components/                   Tab, Dropdown, ScrollArea,
                                                        SearchBar, CollapsibleContainer,
                                                        CardFrame, Activatable, Icon, L10n…
Base/modules/base-standard/ui-next/components/          ScreenFrame, FramedResource,
                                                        YieldDelta, LeaderWithRibbon…
Base/modules/base-standard/ui-next/screens/<ekran>/
Base/modules/core/ui-next/sandbox/                      przykłady Firaxis (!)
Base/modules/core/ui-next/reference/                    referencje: audio, drag&drop
```

W `sandbox/` leżą gotowe, proste przykłady Firaxis (`hello-component.js`,
`simple-binding.js`, `list-example.js`, `effect-model.js`) — to najkrótsza droga do
zrozumienia konwencji.

## Mechanizm nadpisywania — zaprojektowany POD MODY ✅

To najważniejsze odkrycie. `component-registry.ts` mówi wprost w komentarzu:
*„Components can be overriden (for example, by mods) by setting a higher priority"*.

```js
// gra rejestruje:
ComponentRegistry.register({ name: "TradeRouteCard", createInstance: TradeRouteCardComponent });
// mod rejestruje pod TĄ SAMĄ nazwą z wyższym priorytetem:
ComponentRegistry.register({ name: "TradeRouteCard", createInstance: MyCard, overridePriority: 1 });
```

Jak to działa w środku (przeczytane w źródle):
- rejestr trzyma **jeden owinięty obiekt-fabrykę na nazwę**; `register()` zwraca zawsze
  ten sam obiekt i tylko podmienia w nim pole `.factory`, jeśli nowy priorytet jest wyższy;
- kod gry, który zaimportował `TradeRouteCard`, trzyma referencję do tego owiniętego
  obiektu → **po nadpisaniu automatycznie renderuje wersję moda**, bez reimportu.

✅ **Kolejność ładowania nie ma znaczenia.** Mod może zarejestrować się przed grą albo po
niej — wygrywa wyższy `overridePriority`. To całkowicie usuwa problem „patch wykonał się,
zanim obiekt istniał", który zjadł dużo czasu przy modzie specjalistów
(patrz [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)).

Analogicznie `ModelRegistry.register(name, lifecycle, factory, priority)`, gdzie
`ModelLifecycle` to `Singleton` / `SharedInstance` / `PerInstance`.

⚠️ **Ale:** nadpisanie modelu w `ModelRegistry` działa tylko wtedy, gdy konsument
faktycznie woła `Model.get()`. Ekran Handlu **woła fabrykę `createCommerceScreenModel()`
bezpośrednio**, więc rejestracja `"CommerceScreenModel"` z wyższym priorytetem go **nie**
dotknie — patrz [26-commerce-screen.md](26-commerce-screen.md).

**Nadpisać można tylko komponent, który gra zarejestrowała.** Zwykły `export const Foo:
Component = …` bez `ComponentRegistry.register` jest nietykalny — trzeba wtedy podmienić
jego rodzica.

## Most stary ↔ nowy: `defineLegacyComponent` ✅

```js
defineLegacyComponent("screen-resource-allocation", { classNames: ["fullscreen"] },
                      () => createComponent(CommerceScreen, {}));
```

Tworzy klasyczny custom element o podanej nazwie, którego `onAttach` renderuje w środku
drzewo Solid (`render()` z `solid-js/web`). Dzięki temu `ContextManager.push("screen-…")`,
`<template>` w `root-game.html`, tutoriale i cała stara infrastruktura działają bez zmian.

### Który ekran wygrywa, gdy istnieje stara i nowa wersja ✅

`defineLegacyComponent` woła `Controls.define(name, { priority: ranking })`, gdzie
`ranking = Modding.getModRankByURL(calleeURLOrPriority)` albo **domyślnie `1`**.
Stary `Controls.define` bez priorytetu daje `0`. Dlatego przy dwóch definicjach
`screen-resource-allocation` (starej w `ui/resource-allocation/` i nowej w
`ui-next/screens/commerce/`) **wygrywa ta z `ui-next`**.

Ten sam mechanizm jest drogą do całkowitego przejęcia ekranu przez mod:

```js
defineLegacyComponent("screen-resource-allocation",
    { classNames: ["fullscreen"], calleeURLOrPriority: import.meta.url },
    () => createComponent(MyWholeScreen, {}));
```

⚠️ Niesprawdzone w praktyce; `Modding.getModRankByURL` powinno dać modowi rangę > 1.
Nadpisanie przez `ComponentRegistry` jest mniej inwazyjne i należy je preferować.

## ✅ Import z moda DZIAŁA — potwierdzone działającym modem z Workshop

**2026-08-10, KOREKTA wcześniejszego ⚠️:** ścieżki `/core/vendor/…` i `/core/ui-next/…`
z poziomu moda **działają**. Potwierdza to mod **Resource+** (`brads-assign-all-resources`,
Workshop 3756000777, autor Brad), który zaczyna się dokładnie tak:

```js
import { onMount, onCleanup } from '/core/vendor/solid-js/dist/solid.js';
import { ComponentRegistry } from '/core/ui-next/services/component-registry.js';
import { useCommerceScreenContext } from '/base-standard/ui-next/screens/commerce/commerce-screen-model.js';
import { CommerceResourcesContainer } from '/base-standard/ui-next/screens/commerce/commerce-screen-resources-tab.js';
```

## ⭐ Wzorzec: opakowanie oryginału (`Controls.decorate` dla `ui-next`) ✅

Nadpisanie z `overridePriority` **zastępuje** komponent. Żeby go tylko *rozszerzyć*,
trzeba złapać oryginał i zawołać go na końcu. Wzorzec z Resource+:

```js
// 1. import komponentu WYMUSZA wykonanie modułu gry przed tą linią (kolejność ESM),
//    więc rejestracja gry na pewno już się odbyła
import { CommerceResourcesContainer } from '/base-standard/ui-next/screens/commerce/commerce-screen-resources-tab.js';

// 2. `.factory` owiniętej fabryki to implementacja gry
const originalFactory = CommerceResourcesContainer.factory;

function MyWrapper(props) {
    const model = useCommerceScreenContext();   // działa: jesteśmy w drzewie Providera
    onMount(() => { /* wstrzyknij swoje — idempotentnie, patrz niżej */ });
    onCleanup(() => { /* patrz ⚠️ pod spodem: NIE zawsze wolno tu sprzątać */ });
    return originalFactory(props);              // 3. oryginał renderuje się normalnie
}

ComponentRegistry.register({
    name: 'CommerceResourcesContainer',
    overridePriority: 1100,
    createInstance: MyWrapper,
});
```

⚠️ **Tu kolejność JEDNAK ma znaczenie** — inaczej niż przy czystym nadpisaniu.
`Component.factory` musi już wskazywać na implementację gry w chwili odczytu.
Gwarantuje to **import modułu gry** (moduł importowany wykonuje się przed importującym),
a nie `<LoadOrder>`. Resource+ i tak ustawia `LoadOrder` 1100 — nie zaszkodzi.

⚠️ Jeśli dwa mody zrobią to samo, wygrywa wyższy `overridePriority`, a łańcuch
`originalFactory` zachowa się poprawnie **tylko wtedy**, gdy oba czytają `.factory`
w momencie importu. Resource+ zajmuje `CommerceResourcesContainer` z priorytetem **1100** —
przy kolizji trzeba dać więcej (i wtedy to nasz mod woła ich wrapper, a nie odwrotnie).

## Dwie drogi do treści: JSX ręcznie albo goły DOM

### Droga A (jak Resource+): wstrzykiwanie DOM ✅ prostsze

Resource+ **w ogóle nie buduje JSX**. Zwraca oryginał, a swoje elementy dokłada
`document.createElement` / `querySelector` w `onMount`, style wrzuca jako
`<style id="…">` do `document.head`, a synchronizację z reaktywnym drzewem Solid
załatwia `MutationObserver` na `document.body`:

```js
observer = new MutationObserver(reconcileUI);
observer.observe(document.body, { childList: true, subtree: true });
```

Zalety: zero kompilacji, pełna kontrola. Wady: zależność od klas CSS i atrybutów
`data-name` gry (kruche przy patchach) oraz koszt `MutationObserver` na całym `body`.

⚠️ **Sprzątanie w `onCleanup` jest pułapką.** Kontener montuje się kilka razy w trakcie
jednej wizyty na ekranie, a `onMount` nowego montowania potrafi wykonać się **przed**
`onCleanup` poprzedniego — sprzątanie usuwa wtedy dopiero co wstrzyknięty element,
a `stop*()` operujące na modułowym singletonie rozłącza obserwator już należący do
nowego montowania. Do tego Solid przy przerysowaniu kontenera potrafi usunąć obce węzły.

Dlatego obserwator **zostaje podłączony na stałe** i wstawia element z powrotem, ilekroć
zniknie, a `stop*()` woła się wyłącznie przy pełnym demontażu moda. Pełny wzorzec:
**[quirk #51](14-quirks-and-gotchas.md)**.

### Droga B: ręcznie pisany skompilowany JSX ⚠️ nieprzetestowane

Mod ładowany przez `<UIScripts>` to zwykły moduł ES — nikt nie skompiluje w nim JSX.
Ale skompilowany kod gry to czysty JS, który da się pisać ręcznie:

```js
import { createComponent, createMemo, Show, For } from '/core/vendor/solid-js/dist/solid.js';
import { template, insert, effect, setAttribute } from '/core/vendor/solid-js/web/dist/web.js';
```

- `<Foo a={1}>bar</Foo>` → `createComponent(Foo, { a: 1, get children() { return "bar" } })`
- statyczny HTML → `const _tmpl = template('<div class="flex"></div>')`, potem `_tmpl()`
- dziecko dynamiczne → `insert(rodzic, () => wyrażenie)`
- prop reaktywny musi być **getterem** (`get children() {…}`), inaczej traci reaktywność

Eksporty są dostępne: `solid.js` daje m.in. `createComponent, createSignal, createMemo,
createEffect, createContext, useContext, batch, untrack, splitProps, mergeProps, onMount,
onCleanup, For, Show, Switch, Index, lazy`; `web.js` daje `template, insert, render, spread,
setAttribute, classList, style, effect, delegateEvents`.

## Rzeczy, które zaskakują

- **Klasy CSS są utility w stylu Tailwinda**, ale to własna implementacja Firaxis;
  ucieczki w nazwach klas piszą jako `-top-0\.5`, `hover\:scale-125`, `w-1\/4`.
- Rozmiary w px liczy się przez `Layout.pixelsToScreenPixels(512)` — nie wpisuj px na sztywno.
- Teksty: `<L10n.Compose text="LOC_…" />`, nie `data-l10n-id`.
- Obrazy BLP: `url(blp:nazwa)` w stylu + deklaracja w `images: [...]` przy rejestracji
  komponentu (preload), albo `useImageCache().registerImages(Symbol, [...])`.
- `createEngineEvent("NazwaZdarzenia")` z `#core/ui-next/utilities/game-core-utilities.js`
  zamienia zdarzenie silnika na sygnał Solid — nie ma potrzeby ręcznego `engine.off`.
- Stary `UpdateGate` z `core/ui/utilities/utilities-update-gate.js` jest nadal używany
  wewnątrz nowych modeli do zbierania wielu zdarzeń w jedno przeliczenie.
- Aliasy importów w źródłach TypeScript (`#core/…`, `#base/…`) to tylko konwencja
  buildu — w skompilowanym `.js` są **ścieżkami względnymi** (`../../../../core/…`).

## Skąd brać oryginalne źródła

`.js.map` w `ui-next/` mają pełne `sourcesContent` z plikami **`.tsx`**:

```bash
python tools/extract_ts.py "…/base-standard/ui-next/screens/commerce" /katalog/docelowy
```

⚠️ `extract_ts.py` wywala się na mapach dla `*.scss.js` (nazwa źródła kończy się na
`?url`, co jest nielegalną nazwą pliku w Windows). Wypisuje `BLAD …` i idzie dalej —
reszta plików wyciąga się poprawnie, więc to nieszkodliwe.

## Wejście myszy i klawiatury w `ui-next` ✅

**Data: 2026-08-10**, ustalone przy pierwszym feature moda Commerce.

### Kształt zdarzenia `engine-input`

`core/ui/input/input-support.js` — `detail` ma **dokładnie** tyle:

```js
{ name, status, x, y, isTouch, isMouse }
```

❗ **Nie ma `shiftKey`/`ctrlKey`/`altKey` ani `target` w `detail`.** Modyfikatory trzeba
brać z DOM-owych `keydown`/`keyup` (te działają — własna
`core/ui/external/js-spatial-navigation/spatial_navigation.js` czyta z nich
`evt.shiftKey`), albo zadeklarować własną akcję wejścia.

✅ `detail.x` / `detail.y` to **współrzędne DOM-owe** — gra sama robi na nich
`document.elementsFromPoint(event.detail.x, event.detail.y)`
(`core/ui-next/components/drag-and-drop.js`). To najpewniejszy sposób ustalenia,
w co gracz kliknął, bo `event.target` przy wejściu z pada bywa elementem *skupionym*,
a nie tym pod kursorem.

### `mousebutton-right` ✅

- jest **zwykłą akcją wejścia** z `EventType="All"` → dostajesz `START` i `FINISH`
  (`InputActionStatuses`), a nie jedno zdarzenie;
- **nie ma żadnego wiersza w `InputContextConstraints`** → działa w każdym kontekście,
  także `Shell` (czyli na ekranach pełnoekranowych);
- gesty domyślne: `MOUSE_R` oraz — co ciekawe — `KEY_CONTROL+MOUSE_R` jako drugi
  indeks tej samej akcji. Kombinacje `MODYFIKATOR+MYSZ` są więc legalne w `GestureData`.

### PPM domyślnie zamyka ekran — trzeba to przechwycić ⚠️

`InputEngineEvent.isCancelInput()` zwraca true dla `cancel`, `keyboard-escape`
**i `mousebutton-right`**. `core/ui-next/components/panel.js` na `FINISH` takiego
zdarzenia woła `ContextManager.pop(...)` — czyli zamyka ekran.

Panel słucha przez Solidowe `"on:engine-input"`, czyli **natywnym, niedelegowanym**
listenerem na swoim elemencie (faza bąbelkowania). Dlatego mod, który chce nadać PPM
własne znaczenie, musi słuchać **w fazie przechwytywania na `window`**:

```js
window.addEventListener(InputEngineEventName, handler, true);   // ← true = capture
...
event.preventDefault();
event.stopPropagation();
event.stopImmediatePropagation();
```

Capture od `window` biegnie w dół **przed** dotarciem do elementu panelu, więc
zatrzymanie propagacji tam faktycznie zapobiega zamknięciu ekranu. Zatrzymuj tylko
te kliknięcia, które naprawdę obsługujesz — reszta PPM ma dalej zamykać ekran.

### Wzorzec priorytetu odpornego na kolejność ✅

Zamiast wpisywać `overridePriority` na sztywno (i ryzykować, że wyląduje pod cudzym
wrapperem albo go skasuje):

```js
const originalFactory   = SomeComponent.factory;
const overridePriority  = (SomeComponent.overridePriority ?? 0) + 100;
```

Ładujemy się przed innym modem → mamy niski priorytet, on owija nas.
Ładujemy się po nim → mamy wyższy, owijamy jego. **W obu przypadkach oba wrappery
działają**, o ile każdy delegue do swojego `originalFactory`.

### Stan klawiszy modyfikujących: `Input.isShiftDown()` ✅

**2026-08-10, KOREKTA.** Wcześniej napisałem tu, że modyfikatory trzeba brać z DOM-owych
`keydown`/`keyup`. **To nie działa** — sprawdzone w grze: nasłuch na `keydown`, `keyup`
i `mousedown` (faza capture, na `window`) **ani razu** nie zgłosił wciśniętego Shifta,
mimo że gracz go trzymał. Ten interfejs nie przepuszcza stanu modyfikatorów przez
zdarzenia klawiatury DOM.

Zamiast tego silnik odpowiada wprost:

```js
Input.isShiftDown()   // → boolean, można pytać w dowolnym momencie
```

Sama gra używa tego tak samo — `core/ui/tooltips/tooltip-manager.js` i
`tooltip-controller.js` skracają opóźnienie dymka, gdy Shift jest wciśnięty:

```js
const tooltipDelay = Input.isShiftDown() ? 1 : Configuration.getUser().tooltipDelay;
```

Zalety wobec własnej akcji wejścia w `config/input.xml`: zero plików konfiguracyjnych,
zero grupy akcji w zakresie `shell`, żadnego stanu do synchronizowania i żadnej pułapki
z przemapowywaniem gołego modyfikatora (patrz komentarz w `config/input.xml` moda
o specjalistach). Wada: klawisz jest na sztywno, nie da się go przypisać na nowo.

⚠️ Uwaga na kolejność szukania: pełnej listy API `Input.*` nie ma w żadnej
dokumentacji — najszybciej wyciąga się ją z kodu gry:

```bash
grep -rhoE "Input\.[a-zA-Z]+\(" --include=*.js core/ base-standard/ | sort -u
```

To samo podejście działa dla `Game.*`, `Players.*` i reszty globalnych obiektów silnika.

#### ❓ KOREKTA do powyższego: `Input.isShiftDown()` też zawiodło

**2026-08-10, ta sama sesja.** Podmiana toru DOM na `Input.isShiftDown()` **nie
naprawiła problemu** — funkcja istnieje, jest wołana (potwierdzone logiem), ale przy
wciśniętym Shifcie i kliknięciu PPM na ekranie Handlu zwraca `false`.

Stan na teraz: **nie znam działającego sposobu na odczytanie stanu modyfikatora
w kontekście `Shell`.** `Input.isShiftDown()` to jedyne API stanu klawiszy w całym
kodzie gry (`grep -rhoE "Input\.(is|get)[A-Z][a-zA-Z]*\("`), a gra używa go wyłącznie
w tooltipach, które żyją głównie w kontekście świata.

**Hipoteza robocza (do potwierdzenia):** silnik **nie wyzwala gołej akcji
`mousebutton-right`, gdy trzymany jest modyfikator**, więc przy Shift+PPM nie dociera
do nas nic i pytanie o Shift nigdy nie pada w dobrym momencie. Poszlaka jest mocna —
w `core/config/Input.xml` sama gra musi zadeklarować **drugi gest** dla tej samej akcji:

```xml
<Row ActionId="mousebutton-right" Index="0" GestureType="KBMouse" GestureData="MOUSE_R"/>
<Row ActionId="mousebutton-right" Index="1" GestureType="KBMouse" GestureData="KEY_CONTROL+MOUSE_R"/>
```

Gdyby `MOUSE_R` łapało też Ctrl+PPM, ten drugi wiersz byłby zbędny.

**Sposób obejścia, który testujemy:** własna akcja z gestem `KEY_SHIFT+MOUSE_R`
w `config/input.xml` (zakres `shell`, jak w [09-cookbook-ui-mod.md](09-cookbook-ui-mod.md)),
i nasłuch na jej nazwę obok `mousebutton-right`:

```xml
<InputActions>
    <Replace ActionId="moj-mod-akcja" DeviceType="Keyboard"
             Name="LOC_…" Description="LOC_…_HELP" EventType="All" />
</InputActions>
<InputActionDefaultGestures>
    <Replace ActionId="moj-mod-akcja" Index="0"
             GestureType="KBMouse" GestureData="KEY_SHIFT+MOUSE_R" />
</InputActionDefaultGestures>
```

Bez wierszy w `InputContextConstraints` — `mousebutton-right` też ich nie ma i właśnie
dlatego działa na ekranach pełnoekranowych (kontekst `Shell`).

⚠️ Jeżeli okaże się, że silnik jednak wysyła **obie** akcje przy Shift+PPM, trzeba
odsiać duplikat — u nas robi to okno czasowe ~400 ms po operacji masowej.

**Wniosek ogólny na przyszłość:** przy zagadkach z wejściem nie zgaduj kolejnego API —
podłącz na jedną rundę testu podsłuch, który loguje **każde** `engine-input`
(nazwa + status + `isShiftDown()` + współrzędne) oraz natywne zdarzenia DOM
`mousedown`/`contextmenu`/`keydown` z ich flagami modyfikatorów. Jedna runda z danymi
jest tańsza niż trzy rundy zgadywania.

#### ❌ KOREKTA #2: hipoteza „modyfikator blokuje akcję" jest FAŁSZYWA

**2026-08-10, kolejna runda.** Shift+PPM **wyzwala** zwykłą akcję `mousebutton-right` —
widać to w logu (mod wykonał na niej pojedyncze cofnięcie zasobu). Poprzedni wniosek
z pustego okna logu był błędem interpretacji: tam po prostu było mniej kliknięć.

Więc drugi gest `KEY_CONTROL+MOUSE_R` przy `mousebutton-right` w `core/config/Input.xml`
**nie dowodzi**, że gołe `MOUSE_R` nie łapie kombinacji. Powód jego istnienia pozostaje
nieznany.

Stan wiedzy po tej rundzie:

| co | wynik |
|---|---|
| DOM `keydown`/`keyup`/`mousedown` → `shiftKey` | ❌ nigdy nie zgłasza wciśniętego Shifta |
| `Input.isShiftDown()` w kontekście `Shell` | ❌ zwraca `false` mimo trzymanego Shifta |
| Shift+PPM wyzwala `mousebutton-right` | ✅ tak |
| własna akcja z gestem `KEY_SHIFT+MOUSE_R`, bez `InputContextConstraints` | ❌ nie wyzwoliła się |

**Aktualna hipoteza:** mod-owa akcja wymaga **jawnych wierszy
`InputContextConstraints`**. Poszlaka: mod o specjalistach deklaruje dla swojego
klawisza `ContextId="World"` i `ContextId="Unit"` i działa; akcje bazowej gry bez
żadnych wierszy (`mousebutton-right`) też działają, więc reguła „brak wierszy = wszędzie"
najwyraźniej **nie** obejmuje akcji dodanych przez mody.

Do przetestowania: `<Replace ActionId="…" ContextId="Shell" />` (+ `Dual`).
Konteksty do wyboru — `core/config/Input.xml`, tabela `InputContexts`:
**`Shell`, `World`, `Unit`, `Dual`**.

⚠️ Sprawdzenie w czasie działania, czy akcja w ogóle się zarejestrowała (tego nie widać
w `Modding.log` — grupy akcji o zakresie `shell` nie są tam wypisywane):

```js
const actionId = Input.getActionIdByName('moja-akcja');   // null/undefined = nie ma
```

⚠️ Podsłuch wejścia MUSI odsiewać `InputActionStatuses.UPDATE` — `mousebutton-left`
w tym statusie powtarza się co klatkę i zjada cały limit linii, zanim dojdzie do
badanego kliknięcia.

#### ✅ ROZWIĄZANIE: silnik NIE wysyła `mousebutton-right`, gdy trzymany jest modyfikator

**2026-08-10, ustalone podsłuchem — koniec zgadywania.** Ten wpis unieważnia KOREKTĘ #2
i przywraca pierwotną hipotezę. Surowe dane z `UI.log`:

```
zwykły PPM:
  dom mousedown button=2 shiftKey=false isShiftDown=false
  engine-input name=mousebutton-right status=START  isMouse=true
  dom mouseup   button=2 shiftKey=false
  engine-input name=mousebutton-right status=FINISH

Shift + PPM:
  dom mousedown button=2 shiftKey=true  isShiftDown=true
  dom mouseup   button=2 shiftKey=true  isShiftDown=true
  (żadnego engine-input!)
```

**Ani `event.shiftKey`, ani `Input.isShiftDown()` nigdy nie były zepsute** — oba
zwracają `true` we właściwym momencie. Zepsute było źródło zdarzeń: kod nasłuchiwał
akcji `mousebutton-right`, która przy wciśniętym modyfikatorze **w ogóle nie leci**.
Dlatego pytanie „czy Shift jest wciśnięty" padało wyłącznie przy kliknięciach bez
Shifta i zawsze dostawało `false`.

To wyjaśnia też drugi gest `KEY_CONTROL+MOUSE_R` w `core/config/Import.xml` gry:
bez niego Ctrl+PPM byłby dla silnika niczym.

**Wniosek praktyczny — klik z modyfikatorem obsługuj natywnymi zdarzeniami DOM:**

```js
window.addEventListener('mousedown', onDown, true);   // capture
window.addEventListener('mouseup',   onUp,   true);
// event.button === 2, event.shiftKey / ctrlKey / altKey — wszystko dostępne
```

Zdarzenia DOM myszy lecą **w obu przypadkach** i niosą poprawny stan modyfikatorów.
Kolejność w obrębie jednego kliknięcia: `dom mousedown` → `engine START` →
`dom mouseup` → `engine FINISH`; robotę wykonuj na `mouseup`.

⚠️ **Nadal trzeba obsłużyć akcję silnika** — ale wyłącznie po to, żeby ją **zdusić**:
zwykły PPM jest `isCancelInput()` i panel zamyka na nim cały ekran. Przychodzi PO
DOM-owym `mouseup`, więc wystarczy znacznik czasu ustawiony przy obsłudze kliknięcia
i tłumienie akcji w oknie ~400 ms.

⚠️ Zduś też `mousedown`, jeśli kliknięcie trafia w Twój element — inaczej własne
handlery ekranu zdążą zaznaczyć albo zacząć przeciągać element pod kursorem.

❗ **To dotyczy WSZYSTKICH przycisków myszy, nie tylko prawego.** `Activatable`
z `core/ui-next/components/activatable.js` — czyli wszystko, co w tym frameworku jest
klikalne — wyzwala `onActivate` właśnie z akcji silnika:

```js
if (inputEvent.detail.name == "mousebutton-left" || … ) props.onActivate?.();
```

Skutek: **z wciśniętym modyfikatorem cały ekran przestaje reagować na kliknięcia.**
U nas objawiło się to tak, że Shift+przeciągnięcie działało, a Shift+kliknięcie nie —
bo przeciąganie jedzie na zdarzeniach DOM, a klikanie na akcji silnika.

Jeśli Twój mod nadaje modyfikatorowi znaczenie, musisz **odtworzyć zwykłe klikanie**
na czas jego trzymania: na DOM-owym `mouseup` wywołaj tę samą metodę modelu, którą
wywołałby `onActivate` danego `Activatable`.

⚠️ Odróżnij kliknięcie od przeciągnięcia — inaczej upuszczenie po przeciągnięciu
wykona akcję drugi raz. Wystarczy zapamiętać pozycję `mousedown` i odrzucić `mouseup`
przesunięty o więcej niż kilka pikseli.

❓ **Nierozwiązane, ale już niepotrzebne:** własna akcja z gestem `KEY_SHIFT+MOUSE_R`
zadeklarowana w `config/input.xml` (zakres `shell`, `<Replace>`, `EventType="All"`,
z wierszami `InputContextConstraints` i bez nich) **nie zarejestrowała się** —
`Input.getActionIdByName()` zwracało `null`. Powód nieznany; droga przez DOM jest
i tak prostsza (zero plików konfiguracyjnych, zero zakresu `shell`).

## Skracanie tekstu wielokropkiem ✅

Ten silnik obsługuje `text-overflow: ellipsis` — motyw gry ma nawet gotową klasę
`.truncate` (`core/ui/themes/default/default.css`):

```css
.truncate { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
```

⚠️ Wewnątrz flexboxa samo `text-overflow` nie wystarczy: element flex domyślnie nie
kurczy się poniżej szerokości swojej treści. Potrzebne jest **`min-width: 0`** na
skracanym elemencie **i na każdym jego przodku będącym elementem flex**, inaczej zamiast
wielokropka wiersz się rozepnie albo zawinie.

Uzupełnienie: `coh-font-fit-mode` przyjmuje `fit`, `shrink` i `none` — to inny mechanizm
(skalowanie czcionki do pudełka), nie skracanie.


---

## ⚠️ CSS grid: gra go NIE UŻYWA ANI RAZU — nie stawiaj na nim layoutu ✅

```bash
grep -rho "display:\s*grid\|grid-template-columns" --include=*.css --include=*.js Base/
# zero trafień w całej grze
```

Nic nie mówi, że ten renderer implementuje grid. Skoro Firaxis nie użył go nigdzie —
a w wielu miejscach grid byłby oczywistym wyborem — zakładamy, że go nie ma. Wszystko
robimy flexboxem.

### Wyrównanie kolumn liczb bez grida: rząd KOLUMN, nie kolumna rzędów

Problem: dwa wiersze liczb, w każdym kilka pozycji, mają mieć plusy jeden pod drugim.

```
Jeden:      +18 🪙  +18 🌿
Wszystkie:  +144 🪙 +144 🌿      ← druga pozycja startuje tam, gdzie skończyła pierwsza
```

Ułożone **wierszami** nie da się tego wyrównać: szerokość drugiej pozycji zależy od tego,
ile miejsca zajęła pierwsza, więc `+18` nad `+144` rozjeżdża wszystko dalej. Ani
`justify-content`, ani `text-align` tego nie ruszą.

**Rozwiązanie: odwrócić zagnieżdżenie.** Zamiast dwóch wierszy po N pozycji — **N+1 kolumn
po 2 komórki**:

```
[etykiety]   [pozycja 1]   [pozycja 2]
 Jeden:       +18 🪙        +18 🌿
 Wszystkie:   +144 🪙       +144 🌿
```

```css
.figures      { display: flex; flex-direction: row; align-self: center; }
.figures-col  { display: flex; flex-direction: column; flex: 0 0 auto; }
.figures-col + .figures-col { margin-left: 0.9rem; }
```

Każda kolumna jest szeroka na swoją najszerszą komórkę, obie komórki startują przy jej
lewej krawędzi — **wyrównanie wychodzi z konstrukcji**, bez mierzenia i bez zgadywanych
szerokości. Nie trzeba tego stroić, gdy premia urośnie do czterech cyfr.

`align-self: center` na kontenerze daje przy okazji „blok wyśrodkowany, tekst w środku do
lewej" — w kolumnowym kontenerze flex to zwęża element do zawartości i centruje.

⚠️ Klasy modyfikujące wygląd („ten wiersz jest przygaszony", „ten jest pogrubiony")
przenoszą się z wiersza na **każdą komórkę** — wiersz przestaje istnieć jako element.
