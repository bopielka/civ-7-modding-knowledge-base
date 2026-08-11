# 05 — UI i JavaScript

Najliczniejsza kategoria modów do Civ VII. Z 49 modów Workshop **63 użycia `UIScripts`**
vs 18 `UpdateDatabase` — czyli społeczność modduje głównie interfejs.

## Dwa systemy UI obok siebie ✅

| | `ui/` (klasyczny) | `ui-next/` (nowy) |
|---|---|---|
| Technologia | własny framework Firaxis (`Component`, `Controls`) | **Solid.js** |
| Pliki w `core` | 373 JS | 186 JS |
| Pliki w `base-standard` | 601 JS | 132 JS |
| Punkt wejścia | komponenty rejestrowane przez `Controls.define` | `app.js` montuje do `#solidjs-root` |

Silnik renderujący to **Coherent Labs (cohtml)** — `core/ui/cohtml.js`.
W praktyce: HTML + CSS + JS, ale nie przeglądarka; część API DOM działa inaczej.

Mody dotykają obu światów — `bz-map-trix` ma pliki i w `ui/`, i w `ui-next/`.

## Trzy sposoby modyfikacji UI

### 1. `UIScripts` — dodanie skryptu (najczęstsze, najbezpieczniejsze) ✅

```xml
<UIScripts>
    <Item>ui/tooltips/bz-plot-tooltip.js</Item>
</UIScripts>
```
Skrypt jest **dokładany**, nie zastępuje niczego. Twój kod sam decyduje, co podpiąć.

### 2. `ImportFiles` — nadpisanie pliku gry tą samą ścieżką ✅

```xml
<ImportFiles>
    <Item>ui-next/tooltips/plot-tooltip/plot-tooltip.js</Item>
</ImportFiles>
```
✅ Zweryfikowane: `base-standard/ui-next/tooltips/plot-tooltip/plot-tooltip.js` istnieje
w grze pod **identyczną ścieżką względną** — mod ją przesłania.
Skuteczne, ale **łamie kompatybilność** z każdym innym modem ruszającym ten plik
i psuje się przy każdym patchu gry.

### 3. `ReplaceUIScript` — jawna podmiana ✅

```xml
<ReplaceUIScript>
    <Item>ui/policies/policies-and-traditions.js</Item>
</ReplaceUIScript>
```
Używane rzadko (3 mody: `leugi-diploribbon-tweaks`, `leugi-diploicon-tweaks`,
`stachs-elegant-policies-and-traditions`) — gdy trzeba przepisać komponent od zera.

**Hierarchia preferencji:** `UIScripts` + dekorator → `ReplaceUIScript` → `ImportFiles`.

## `Controls.decorate` — oficjalny punkt rozszerzeń ✅

45 użyć w modach. Definicja z `core/ui/component-support.js`:

```js
decorate(name, provider)   // rejestruje dekorator dla komponentu o danej nazwie
```

Komentarz Firaxis w kodzie mówi wprost:
> nie utworzy dekoratora dla **istniejących instancji** komponentu

⚠️ Stąd znaczenie `LoadOrder` — dekorator musi być zarejestrowany, zanim komponent
zostanie utworzony.

### Kontrakt dekoratora ✅

```js
class MojDekorator {
    constructor(component) {
        this.component = component;
        component.mojDekorator = this;   // wzajemna referencja
    }
    beforeAttach() { }
    afterAttach()  { }   // tu zwykle dokłada się DOM
    beforeDetach() { }
    afterDetach()  { }   // tu sprząta się listenery
}
Controls.decorate('nazwa-komponentu', (component) => new MojDekorator(component));
```

## Rejestracja własnego komponentu ✅

```js
class bzPlotIconWonders extends Component {
    onInitialize() {
        const wonderType = this.Root.getAttribute("wonder");
        this.Root.style.backgroundImage = `url(fs://game/moj-mod/icons/ikona.png)`;
        this.Root.classList.add("size-24", "bg-cover", "bg-center");
    }
}
Controls.define("bz-plot-icon-wonders", { createInstance: bzPlotIconWonders });
```
`this.Root` to element DOM komponentu. Klasy CSS wyglądają na **Tailwind-podobne**
(`size-24`, `bg-cover`, `bg-no-repeat`, `bg-center`) ⚠️ — konwencja gry, nie sprawdzałem
czy to pełny Tailwind.

## Patchowanie warstw LensManager (technika czystsza niż patch prototypu) ✅

`LensManager` trzyma zarejestrowane warstwy w **publicznym polu** `layers` (Map),
zdefiniowanym w `core/ui/lenses/lens-manager.js`:
```js
class LensManagerSingleton {
  layers = new Map();   // publiczne — dostępne z zewnątrz
  registerLensLayer(layerType, layer) { this.layers.set(layerType, layer); ... }
}
```
Warstwy (np. `fxs-worker-yields-layer`, `fxs-yields-layer`) to **pojedyncze instancje**
zarejestrowane raz przy imporcie pliku (`LensManager.registerLensLayer("nazwa", new Klasa())`)
— nie komponenty przez `Controls.define`, więc `Controls.decorate` tu nie działa.

Zamiast nadpisywać cały plik gry (`ImportFiles`/`ReplaceUIScript`), można **podmienić
metodę bezpośrednio na już zarejestrowanej instancji**:

```js
import LensManager from '/core/ui/lenses/lens-manager.js';

engine.whenReady.then(() => {
    const layer = LensManager.layers.get('fxs-worker-yields-layer');
    if (!layer) { console.error('warstwa jeszcze niezarejestrowana'); return; }
    if (layer.__mojPatch) return;         // strażnik — patchuj tylko raz
    layer.__mojPatch = true;

    layer.metodaDoPodmiany = function (arg) {   // zwykła funkcja, nie arrow — `this` = instancja
        // można wywoływać oryginalne metody pomocnicze instancji:
        this.innaMetodaPomocnicza(...);
        // ...własna logika...
    };
});
```
⚠️ **Zalety nad `ImportFiles`:** nie nadpisuje pliku gry (przetrwa patche i inne mody
dotykające tego samego pliku), a nad prototype-patch: nie trzeba ręcznie odtwarzać
mechanizmu „wywołaj oryginał" — wystarczy `this.innaMetodaGry(...)`, bo pozostałe
metody instancji są nietknięte.

⚠️ **Timing:** rejestracja warstwy następuje przy imporcie pliku gry (`scope="game"`,
moduł bazowy). Jeśli Twój skrypt ma wyższy `LoadOrder`, warstwa powinna już istnieć —
mimo to sprawdzaj `if (!layer)` i rozważ `setTimeout(fn, 0)` jako defensywny retry.

Ta sama technika (publiczne pole z instancjami) może dotyczyć innych menedżerów-singletonów
gry — warto sprawdzić `class XManagerSingleton { pole = new Map()/Array() }` przed
sięgnięciem po prototype-patch.

## Patchowanie prototypów (ostateczność) ✅

Gdy komponent nie ma punktu rozszerzeń, mody podmieniają metodę na prototypie
z zachowaniem oryginału:

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
        if (bzPlotIconsRoot.c_prototype == proto) return;   // patchuj tylko raz!
        bzPlotIconsRoot.c_prototype = proto;
        bzPlotIconsRoot.c_onRemoveIcon = proto.onRemoveIcon;   // zachowaj oryginał
        proto.onRemoveIcon = function(event) {
            return this.bzComponent.onRemoveIcon(event);
        };
    }
}
```
Wzorzec „patchuj tylko raz" (strażnik na statycznym polu) jest istotny — bez niego
przy wielu instancjach obudujesz metodę wielokrotnie.

## Ścieżki i importy ✅

```js
import LensManager from '/core/ui/lenses/lens-manager.js';
import PlotIconsManager from '/core/ui/plot-icons/plot-icons-manager.js';
import '/bz-map-trix/ui/mini-map/bz-panel-mini-map.js';   // własny plik moda
```

Reguła: **ścieżka zaczyna się od `/` + identyfikator modułu/moda**.
Zweryfikowane użycia: `/core/` (361), `/base-standard/` (91), `/<mod-id>/` (39).

Zasoby (obrazy, CSS) adresuje się przez `fs://game/<mod-id>/…`:
```js
this.Root.style.backgroundImage = `url(fs://game/bz-map-trix/icons/ikona.png)`;
```

## Najczęściej używane globalne API ✅

Zliczone we wszystkich 317 plikach JS modów:

| API | Użyć | Do czego |
|---|---|---|
| `GameInfo` | 676 | **dostęp do bazy danych** — `GameInfo.Constructibles.lookup(type)` |
| `GameplayMap` | 549 | zapytania o pola mapy |
| `engine` | 477 | zdarzenia i model danych |
| `UI` | 383 | ogólne funkcje interfejsu |
| `Game` | 320 | stan gry |
| `Players` | 316 | gracze |
| `ComponentID` | 192 | identyfikatory obiektów gry |
| `InterfaceMode` | 174 | tryby interakcji |
| `Units` / `Cities` / `Districts` | 163/105/57 | obiekty gry |
| `LensManager` | 148 | warstwy/soczewki mapy |
| `Controls` | 108 | rejestracja i dekoracja komponentów |
| `WorldUI` | 86 | rysowanie w świecie 3D |
| `NavTray`, `FocusManager`, `ContextManager` | 60/44/37 | nawigacja i fokus (pad/klawiatura) |
| `Audio`, `Configuration`, `Input`, `Camera` | 55/48/29/14 | pozostałe |

## Zdarzenia — `engine` ✅

```js
engine.on('NazwaZdarzenia', handler);    // 306 użyć
engine.off('NazwaZdarzenia', handler);   //  96 — ZAWSZE sprzątaj w afterDetach
engine.whenReady.then(() => { ... });    //  38 — czekaj na gotowość silnika
engine.trigger('NazwaZdarzenia', dane);  //  20
```
`engine.createJSModel` / `engine.updateWholeModel` / `engine.synchronizeModels` —
wiązanie danych do widoku (rzadziej, głównie w większych modach).

## Warstwy mapy (soczewki) ✅

```js
LensManager.registerLensLayer("bz-wonder-layer", new bzWonderLensLayer());
LensManager.toggleLayer("bz-wonder-layer");
```
Popularna kategoria modów (`maple-leaves-more-lens`, `slothoth-better-archeology-lens`).
