# 09 — Cookbook: mod interfejsu

Najpopularniejsza kategoria modów Civ VII. Poniżej kompletny, minimalny mod UI
oparty na wzorcach z działających modów Workshop ✅.

## Minimalna struktura

```
moj-ui-mod\
├── moj-ui-mod.modinfo
├── ui\
│   └── moj-panel.js
└── text\
    └── en_us\InGameText.xml
```

## `.modinfo`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Mod id="moj-ui-mod" version="1" xmlns="ModInfo">
    <Properties>
        <Name>LOC_MOD_MOJ_UI_NAME</Name>
        <Description>LOC_MOD_MOJ_UI_DESC</Description>
        <Authors>Twoje imię</Authors>
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
        <ActionGroup id="moj-ui-mod-game" scope="game" criteria="always">
            <Properties><LoadOrder>1000</LoadOrder></Properties>
            <Actions>
                <UIScripts>
                    <Item>ui/moj-panel.js</Item>
                </UIScripts>
                <UpdateText>
                    <Item>text/en_us/InGameText.xml</Item>
                </UpdateText>
            </Actions>
        </ActionGroup>
    </ActionGroups>
</Mod>
```

⚠️ `AffectsSavedGames=0` — mody UI **nie** wpływają na zapisy. Wszystkie 47 modów
Workshop, które deklarują tę właściwość, mają `0`. Ustaw `1` tylko, jeśli zmieniasz
dane rozgrywki wpływające na stan gry.

## Wzorzec 1 — dekorator istniejącego komponentu (zalecany) ✅

Nie podmieniasz plików gry → maksymalna kompatybilność z innymi modami.

```js
// ui/moj-panel.js
class MojDekorator {
    constructor(component) {
        this.component = component;
        component.mojDekorator = this;
        this.onUpdateListener = this.onUpdate.bind(this);
    }
    beforeAttach() { }
    afterAttach() {
        // komponent jest już w DOM — tu dokładaj elementy
        const info = document.createElement('div');
        info.classList.add('font-body-sm', 'text-accent-2');
        info.setAttribute('data-l10n-id', 'LOC_MOJ_UI_ETYKIETA');
        this.component.Root.appendChild(info);

        engine.on('SomeGameEvent', this.onUpdateListener);
    }
    beforeDetach() {
        engine.off('SomeGameEvent', this.onUpdateListener);   // ZAWSZE sprzątaj
    }
    afterDetach() { }

    onUpdate(data) { /* ... */ }
}

Controls.decorate('panel-nazwa-komponentu', (component) => new MojDekorator(component));
```

**Jak znaleźć nazwę komponentu do dekoracji:**
```bash
grep -rn "Controls.define" "Base/modules/base-standard/ui/" | head -40
```

## Wzorzec 2 — własny komponent ✅

```js
class MojPanel extends Component {
    onInitialize() {
        this.Root.classList.add('flex', 'flex-col');
        this.Root.style.backgroundImage = 'url(fs://game/moj-ui-mod/icons/tlo.png)';
    }
    onAttach() { }
    onDetach() { }
    onAttributeChanged(name, oldValue, newValue) { }
}
Controls.define('moj-panel', { createInstance: MojPanel });
```

## Wzorzec 3 — patch prototypu (gdy nie ma innej drogi) ✅

```js
// wykonaj TYLKO RAZ — strażnik na polu statycznym
class MojPatch {
    static patched;
    static apply(component) {
        const proto = Object.getPrototypeOf(component);
        if (MojPatch.patched === proto) return;
        MojPatch.patched = proto;

        const oryginal = proto.update;
        proto.update = function(...args) {
            const wynik = oryginal.apply(this, args);   // zachowaj oryginalne zachowanie
            /* twoja dodatkowa logika */
            return wynik;
        };
    }
}
```

## Odczyt danych gry ✅

`GameInfo` to najczęściej używane API w modach (676 wystąpień) — to bezpośredni
dostęp do bazy z [02-database.md](02-database.md):

```js
const info = GameInfo.Constructibles.lookup(item.type);
if (info?.ConstructibleClass === 'WONDER') { /* ... */ }

for (const row of GameInfo.Units) { console.log(row.UnitType, row.BaseMoves); }
```

Stan bieżącej gry:
```js
const player = Players.get(GameContext.localPlayerID);
const cities = player?.Cities?.getCities() ?? [];
const plot   = GameplayMap.getPlotIndexFromXY(x, y);
```

## Własna warstwa mapy (soczewka) ✅

```js
import LensManager from '/core/ui/lenses/lens-manager.js';

class MojaWarstwa {
    initLayer() { }
    applyLayer() { this.visible = true;  /* rysowanie */ }
    removeLayer() { this.visible = false; }
}
LensManager.registerLensLayer('moja-warstwa', new MojaWarstwa());
```

## Style CSS ✅

```xml
<ImportFiles><Item>ui/moj-styl.css</Item></ImportFiles>
```
```js
Controls.loadStyle('fs://game/moj-ui-mod/ui/moj-styl.css');
```
Gra używa klas w stylu utility (`flex`, `flex-col`, `size-24`, `bg-cover`,
`font-body-sm`, `text-accent-2`) ⚠️ — najbezpieczniej podpatrzeć klasy w istniejących
plikach `.css`/`.js` gry niż zgadywać.

## Teksty i tłumaczenia

```xml
<!-- text/en_us/InGameText.xml -->
<Database><EnglishText>
    <Row Tag="LOC_MOJ_UI_ETYKIETA"><Text>My label</Text></Row>
</EnglishText></Database>
```
W DOM używaj `data-l10n-id="LOC_MOJ_UI_ETYKIETA"` zamiast wpisywać tekst na sztywno —
gra sama podstawi tłumaczenie.

## Opcje moda w menu głównym ✅

Gra nie ma oficjalnego API dla opcji modów, ale społeczność ustaliła działający wzorzec
(źródło: `bz-map-trix`, zweryfikowane w `core/ui/options/`).

### 1. Własna zakładka „Mods"

Kategoria `Mods` nie istnieje w grze — mody ją **dodają**. Używaj `??=` i identyfikatora
`"mods"`, żeby **współdzielić jedną zakładkę** z innymi modami zamiast tworzyć duplikat:

```js
import '/core/ui/options/screen-options.js';   // musi załadować się pierwsze
import { CategoryType, Options, OptionType } from '/core/ui/options/model-options.js';
import { CategoryData } from '/core/ui/options/options-helpers.js';

CategoryType["Mods"] = "mods";
CategoryData[CategoryType.Mods] ??= {
    title: "LOC_UI_CONTENT_MGR_SUBTITLE",
    description: "LOC_UI_CONTENT_MGR_SUBTITLE_DESCRIPTION",
};
```

### 2. Rejestracja opcji

```js
Options.addInitCallback(() => {
    Options.addOption({
        category: CategoryType.Mods,
        group: "moj_mod",
        type: OptionType.Checkbox,
        id: "moj-mod-cos",
        initListener:   (info)         => info.currentValue = MojeOpcje.cos,
        updateListener: (_info, value) => MojeOpcje.cos = value,
        label: "LOC_OPTIONS_MOJ_MOD_COS",
        description: "LOC_OPTIONS_MOJ_MOD_COS_DESCRIPTION",
    });
});
```
⚠️ **Nazwa grupy wymaga własnego tekstu.** `group: "najane_mods"` powoduje, że UI szuka
klucza `LOC_OPTIONS_GROUP_NAJANE_MODS` (nazwa grupy → wielkie litery). Bez tego wiersza
w `UpdateText` zobaczysz w opcjach surowy klucz zamiast nagłówka.

`OptionType` ✅: `Editor`, `Checkbox`, `Dropdown`, `Slider`, `Stepper`, `Switch`.
Dla `Dropdown` dochodzi `dropdownItems: [{label, value}, ...]`, a `initListener`
ustawia `info.selectedItemIndex`.

⚠️ `addInitCallback` **rzuca wyjątkiem**, jeśli opcje zostały już zainicjalizowane —
rejestruj przy ładowaniu skryptu, nie później.

### 3. Zapis wartości ✅

```js
UI.setOption("user", "Mod", `${MOD_ID}.${optionID}`, value);
Configuration.getUser().saveCheckpoint();      // bez tego nie przetrwa restartu
const value = UI.getOption("user", "Mod", `${MOD_ID}.${optionID}`);
```
Wartości to **liczby** (`Number(true)` → 1), nie boolean.
Społeczność dubluje zapis w `localStorage` (klucz `modSettings`) jako zabezpieczenie.

### 4. ⚠️ Rejestruj opcje w OBU zakresach

Ekran opcji istnieje i w menu głównym, i w grze:
```xml
<ActionGroup id="moj-mod-game" scope="game" criteria="always">
    <Actions><UIScripts><Item>ui/options/moje-opcje.js</Item>…</UIScripts></Actions>
</ActionGroup>
<ActionGroup id="moj-mod-shell" scope="shell" criteria="always">
    <Actions>
        <UpdateText><Item>text/en_us/InGameText.xml</Item></UpdateText>
        <UIScripts><Item>ui/options/moje-opcje.js</Item></UIScripts>
    </Actions>
</ActionGroup>
```
Bez grupy `shell` opcji **nie będzie w menu głównym**. Pamiętaj też o `UpdateText`
w `shell` — inaczej zamiast etykiet zobaczysz klucze `LOC_*`.

## Własny klawisz przemapowywalny w opcjach ✅ (kompletny przepis)

Zweryfikowane w praktyce. Do zadziałania potrzebne są **cztery** elementy — brak
któregokolwiek daje ciszę bez błędu, więc łatwo utknąć.

### 1. Deklaracja akcji — `config/input.xml`

```xml
<Database>
    <InputActions>
        <Replace ActionId="moj-mod-akcja" DeviceType="Keyboard"
                 Name="LOC_INPUT_MOJ_MOD_AKCJA" Description="LOC_INPUT_MOJ_MOD_AKCJA_HELP"
                 EventType="All" />
    </InputActions>
    <InputActionDefaultGestures>
        <Replace ActionId="moj-mod-akcja" Index="0"
                 GestureType="KBMouse" GestureData="KEY_TAB" />
    </InputActionDefaultGestures>
    <InputContextConstraints>
        <Replace ActionId="moj-mod-akcja" ContextId="World" />
        <Replace ActionId="moj-mod-akcja" ContextId="Unit" />
    </InputContextConstraints>
</Database>
```
- `EventType="All"` → dostajesz `START`/`FINISH`, czyli **wykrywanie trzymania**
- ❗ podpinaj **tylko** w `scope="shell"` (patrz [14](14-quirks-and-gotchas.md) #28)

### 2. Pokazanie akcji w ekranie remapowania ❗

Ekran **nie wypisuje zarejestrowanych akcji** — iteruje po zaszytej tablicy
`KEYS_TO_ADD` w `core/ui/options/editors/editor-keyboard-mapping.js`. Bez tego kroku
akcja działa, ale jest **niewidoczna w ustawieniach**:

```js
class MojEditorPatch {
    static patched = null;
    constructor(component) { this.component = component; component.mojPatch = this;
        this.patchPrototype(Object.getPrototypeOf(component)); }
    patchPrototype(proto) {
        if (MojEditorPatch.patched) return;         // prototyp współdzielony - raz
        const original = proto.addActionsForContext;
        MojEditorPatch.patched = { proto, original };
        proto.addActionsForContext = function (...args) {
            const r = original.apply(this, args);
            return this.mojPatch?.afterAddActionsForContext(...args) ?? r;
        };
    }
    beforeAttach() {} afterAttach() {} beforeDetach() {} afterDetach() {}
    afterAddActionsForContext(ctx) {
        const id = Input.getActionIdByName("moj-mod-akcja");
        if (!id || this.component.mappingDataMap.has(id)) return;
        this.component.actionContainer.appendChild(this.component.createActionEntry(id, ctx));
    }
}
Controls.decorate('editor-keyboard-mapping', (c) => new MojEditorPatch(c));
```
⚠️ Ten plik podepnij w **obu** zakresach (`shell` i `game`) — ekran opcji jest w obu.

### 3. Wykrywanie trzymania

```js
import { InputEngineEventName } from '/core/ui/input/input-support.js';

let held = false;
window.addEventListener(InputEngineEventName, (e) => {
    if (e.detail?.name !== "moj-mod-akcja") return;
    if (e.detail.status === InputActionStatuses.START) held = true;
    else if (e.detail.status === InputActionStatuses.FINISH) held = false;
});
// brak FINISH przy zmianie trybu / utracie fokusu - resetuj
window.addEventListener(InterfaceModeChangedEventName, () => held = false);
window.addEventListener("blur", () => held = false);
```

### 4. Wyświetlenie aktualnego przypisania w UI

```js
const actionId = Input.getActionIdByName("moj-mod-akcja");       // ❗ NUMERYCZNE id
const deviceType = Input.getActionDeviceType(actionId);
const locKey = Input.getGestureDisplayString(actionId, 0, deviceType, InputContext.ALL);
const label = Locale.compose(locKey);                            // ❗ zwraca KLUCZ, nie tekst
```
⚠️ Dwie pułapki naraz: przekazanie **nazwy** zamiast id zwraca pustkę bez błędu,
a wynik to klucz lokalizacji (`LOC_OPTIONS_KEY_TAB`), który trzeba jeszcze złożyć.
Do wstawienia użyj `Locale.compose(hintKey, label)` + `textContent` — `data-l10n-id`
nie przyjmie argumentu wyliczonego w czasie działania.

### ⚠️ Nie dawaj gołego modyfikatora jako domyślnego

`KEY_SHIFT`/`KEY_CONTROL` **działają** jako wartość domyślna z XML, ale silnikowy
rejestrator gestów (`Input.beginRecordingGestures`) traktuje modyfikator jako początek
kombinacji. Skutki:
- `KEY_CONTROL` wyświetlał się w ekranie jako **„nieprzypisany"**
- po przemapowaniu z `KEY_SHIFT` **nie da się do niego wrócić** — tylko pełny
  `Input.restoreDefault()`, który resetuje przypisania **wszystkich** modów

Dawaj zwykły klawisz (`KEY_TAB` był wolny i w grze bazowej, i we wszystkich 49 modach).

## Klawisze modyfikujące (Shift/Ctrl/Alt) ❗

> ⚠️ **KOREKTA.** Napisałem tu wcześniej, że wystarczy `window.addEventListener("keydown"/"keyup")`.
> **W praktyce to nie zadziałało** — przytrzymanie Shift nie wywoływało żadnej reakcji.

**Dlaczego:** gra **nie kieruje surowego stanu klawiatury do DOM**. Własny system wejścia
(`InputEngineEvent`) przenosi wyłącznie **nazwane akcje** i tylko dyskretne naciśnięcia:
```js
detail: { name, status, x, y, isTouch, isMouse }   // brak jakiejkolwiek informacji o modyfikatorach
```
System `InputActions` / `Input.bind` też nie pomoże — obsługuje „naciśnięto klawisz akcji",
a nie „modyfikator jest właśnie trzymany".

**Co działa:** zdarzenia **myszy** niosą `shiftKey` (potwierdzone: mod `shift-que` czyta
`event.shiftKey` z `click`). Dlatego próbkuj stan ze **wszystkich** dostępnych zdarzeń:

```js
const SAMPLED = ["keydown","keyup","mousemove","mousedown","mouseup","mouseover","click","wheel"];
let shiftHeld = false;
function update(e) {
    if (!!e.shiftKey === shiftHeld) return;
    shiftHeld = !!e.shiftKey;
    /* przerysuj */
}
for (const t of SAMPLED) document.addEventListener(t, update, true);  // capture!
window.addEventListener("blur", () => { shiftHeld = false; /* przerysuj */ });
```

- **`document` + faza przechwytywania (`true`)** — żeby zobaczyć zdarzenie, zanim
  cokolwiek zatrzyma propagację
- **czytaj `event.shiftKey`**, nie `event.key === "Shift"` — stan poprawia się też
  przy kombinacjach klawiszy
- **resetuj przy utracie fokusu i zmianie trybu interfejsu** — brakujący `keyup`
  zostawiłby widok zablokowany w stanie Shift
- w praktyce ratuje `mousemove`: użytkownik i tak wodzi kursorem, więc stan jest świeży
  nawet gdyby zdarzenia klawiatury nie docierały wcale

⚠️ Przy takim rozwiązaniu **loguj kilka pierwszych zmian stanu** (`console.error`, bo
`console.log` nie trafia do `UI.log`) — inaczej nie odróżnisz „nie działa" od
„działa, ale przerysowanie jest zepsute".

## Wydajność ✅ (wnioski z realnego moda)

### 1. Nie licz danych zbiorczych w funkcji wołanej per element

Klasyczna pułapka: funkcja rysująca **jeden** kafelek woła funkcję, która przegląda
**wszystkie** kafelki. To złożoność kwadratowa, a przy okazji mnoży wywołania silnika:

```js
// ŹLE - dla każdego z n kafelków przegląda n kafelków
layer.updateSpecialistPlot = function (info) {
    const baseline = computeBaseline();   // iteruje po wszystkich polach!
    ...
};
```
Przy 30 polach to ~900 wywołań `Districts.getAtLocation()` na jedno przerysowanie.

**Rozwiązanie:** cache w module + jawne unieważnianie na zdarzeniach, które
faktycznie zmieniają dane wejściowe:
```js
let cachedBaseline = null;
export function invalidateCaches() { cachedBaseline = null; /* + mapy per-plot */ }
window.addEventListener(PlotWorkersUpdatedEventName, invalidateCaches);
window.addEventListener(InterfaceModeChangedEventName, invalidateCaches);
```
⚠️ Rejestruj unieważnianie **w module danych**, nie u konsumentów — wtedy każdy
konsument dostaje świeże dane automatycznie i nie da się o tym zapomnieć.
⚠️ Moduł danych jest importowany wcześniej niż komponenty UI, więc jego listener
wykonuje się **przed** listenerami rysującymi. To jest wymagana kolejność.

### 2. Przerysowuj tylko to, co się zmieniło

Pełne przerysowanie warstwy jest drogie — `realizeGrowthPlots()` przechodzi całą
domenę wzrostu miasta, a gdy jej nie ma, **skanuje całą mapę** (`width × height`
z `getRevealedState` na każde pole).

Podpięcie takiego przerysowania pod zdarzenie najechania kursorem (które sypie się
przy każdym ruchu myszy) to najgorszy możliwy przypadek. Skoro najechanie zmienia
wygląd **dwóch** pól, przerysuj dwa:
```js
function redrawPlot(plotIndex) {
    const info = PlotWorkersManager.allWorkerPlots.find((p) => p.PlotIndex === plotIndex);
    if (!info) return;
    layer.yieldVisualizer.clearPlot(GameplayMap.getLocationFromIndex(plotIndex));
    layer.updateSpecialistPlot(info);
}
```
⚠️ `YieldChangeVisualizer.clearPlot()` przyjmuje **lokalizację**, a `SpriteGrid.clearPlot()`
w innej warstwie przyjmuje **indeks pola**. Łatwo pomylić.

### 3. Sprawdź, czy przerysowanie jest w ogóle potrzebne

Najtańsza praca to ta niewykonana. Jeśli opcja sprawia, że dane zdarzenie nie zmienia
niczego na ekranie — wyjdź od razu:
```js
if (NajaneOptions.alwaysShowNegatives || isOriginalDisplayActive()) return;
```

### 4. Wyłącz diagnostykę przed wydaniem ❗

`console.error` (jedyny kanał docierający do `UI.log` — patrz [19](19-workflow-and-debugging.md))
w wydanym modzie zaśmieca log gracza wpisami wyglądającymi jak błędy. Trzymaj flagę:
```js
export const DIAGNOSTICS = false;   // włączaj tylko na czas śledztwa
```
⚠️ To realnie wyszło w wersji 1.0 na Workshop — łatwo przeoczyć.

## Typowe błędy

| Objaw | Przyczyna |
|---|---|
| Dekorator nie działa | komponent powstał **przed** rejestracją → podnieś `LoadOrder` |
| Wycieki / podwójne handlery | brak `engine.off` w `beforeDetach` |
| Metoda obudowana wielokrotnie | brak strażnika „patchuj raz" przy patchu prototypu |
| Widać `LOC_...` zamiast tekstu | brak wiersza w `UpdateText` albo literówka w tagu |
| Mod psuje się po patchu gry | użyto `ImportFiles`/`ReplaceUIScript` zamiast dekoratora |

## Opcje moda: kategoria, grupa i skąd biorą się nagłówki ✅

**2026-08-10.** Ekran opcji ma dwa poziomy zagnieżdżenia i **oba tytuły biorą się skądinąd
niż z kodu moda**:

```js
Options.addOption({
    category: CategoryType.Mods,   // zakładka
    group: 'najane_commerce',      // nagłówek sekcji wewnątrz zakładki
    …
});
```

**Zakładka** — `CategoryData[CategoryType.Mods]` z tytułem i opisem. Kategoria „Mods" nie
istnieje w grze bazowej; dodaje się ją przez `CategoryType["Mods"] = "mods"` i
`CategoryData[...] ??= {…}`. `??=` jest istotne: kilka modów społeczności robi to samo
i dzięki temu **dzielą jedną zakładkę** zamiast mnożyć osobne.

**Nagłówek sekcji** — wyliczany z identyfikatora grupy przez
`GetGroupLocKey` w `core/ui/options/options-helpers.js`:

```js
const suffix = group.toUpperCase();
return `LOC_OPTIONS_GROUP_${suffix}`;
```

Czyli `group: 'najane_commerce'` wymaga klucza **`LOC_OPTIONS_GROUP_NAJANE_COMMERCE`**
w plikach tekstowych moda. Bez niego zobaczysz surowy klucz.

❗ **Nie używaj cudzego identyfikatora grupy.** Zakładkę dzielić wypada, sekcji nie:
grupa jest nazwana tytułem swojego właściciela, więc opcja wrzucona do `najane_mods`
(grupa moda „Common Specialists Yields") wygląda na należącą do tamtego moda. Każdy mod
powinien mieć własną grupę nazwaną swoją nazwą.

## Układ plików moda UI, który się broni przy wzroście ✅

Po tym, jak `better-commerce-screen-ui` urósł do ~5500 linii w 26 plikach leżących płasko
w `ui/`, przebudowa dała warstwy zależne **tylko w jedną stronę**:

```
support/  ← nie wie nic o grze ani o modzie (log, tworzenie elementów, wstrzykiwanie CSS)
engine/   ← rozmowa z grą: PlayerOperations, czekanie na zdarzenia, epoka, modyfikatory wejścia
model/    ← czytanie danych ekranu (i te same kształty odtworzone bez ekranu)
planner/  ← decyzje: co gdzie trafia
screen/   ← DOM, który mod dokłada do ekranu
options/  ← nie importuje niczego naszego; liść
```

Reguła: moduł może importować z własnego katalogu albo **z lewej strony**, nigdy z prawej.

**Po co to, konkretnie:** automatyczne przypisywanie działa przy zamkniętym ekranie.
Kiedy planner importował checkbox „najpierw fabryczne", żeby spytać o wartość ustawienia,
załadowanie silnika przypisań ciągnęło za sobą cały pasek przycisków. Rozdzielenie
**ustawienia** (`planner/factory-first-setting.js`) od **kontrolki** (`screen/factory-first.js`)
usunęło zależność. Ten sam zabieg dotyczył `isFactoryAge()` — funkcja mieszkała przy
zakładce fabrycznej, a pytały o nią trzy moduły z dwóch warstw; przeniesiona do
`engine/age.js`.

**Sygnały, że pora na taki podział:**

- jeden plik przekracza ~1000 linii i ma w sobie `//#region` (te regiony to gotowe cięcia);
- ta sama funkcja istnieje w 2–3 kopiach — u nas `canStart` do `ASSIGN_RESOURCE` był w
  trzech modułach, a jeden z nich w nagłówku deklarował, że jest „jedynym miejscem";
- ta sama nazwa oznacza dwie różne rzeczy — `allAvailableResources` zwracało w jednym
  module zasoby z modelu, a w drugim tylko te wyrenderowane, w kolejności DOM;
- warstwa niska importuje wysoką (planner → screen).

**Weryfikacja po przenosinach, bez uruchamiania gry** — trzy skrypty, każdy złapał realny błąd:

1. `node --check` na każdym pliku (składnia);
2. dla każdego `import { a, b } from './x.js'` sprawdź, czy `x.js` naprawdę eksportuje `a` i `b`;
3. wypisz identyfikatory używane w pliku, a niezdefiniowane i nieimportowane w nim — to
   wyłapuje funkcje i stałe, które zostały po drugiej stronie cięcia (u nas
   `settlementYieldTotal` i `modifierApplies`). Uwaga na fałszywe trafienia: `rgba(`,
   `url(`, `scale(` z CSS w literałach szablonowych oraz nazwy parametrów-callbacków.

Do tego graf importów z wykrywaniem cykli i sprawdzeniem kierunku warstw.
