# 14 — Pułapki i dziwactwa

Rzeczy, które kosztują godziny, jeśli się ich nie wie. Zbierane na bieżąco.

## 1. Dwie składnie modyfikatorów, łatwo pomylić ✅

```xml
<!-- ✅ NOWA (GameEffects) — MAŁE litery, to jest domyślna droga w Civ VII -->
<Modifier id="..." collection="COLLECTION_PLAYER_CITIES" effect="EFFECT_CITY_ADJUST_YIELD">

<!-- ⚠️ STARA (tabelowa, z Civ VI) — WIELKIE litery, w tabeli DynamicModifiers -->
<Row ModifierType="..." CollectionType="COLLECTION_OWNER" EffectType="EFFECT_..."/>
```
Szukanie `CollectionType=` znajdzie ~5 wyników; szukanie `collection=` znajdzie tysiące.
**Jeśli twoje grepy zwracają podejrzanie mało wyników — sprawdź wielkość liter.**

## 2. `Types` — bez tego wiersza nic nie zadziała ✅

Każdy nowy obiekt musi mieć wiersz w `Types` z właściwym `Kind`, **zanim** pojawi się
w tabeli docelowej. Brak wpisu = cicha porażka klucza obcego.

## 3. Cywilizacja wymaga OBU scope ✅

`shell` (menu wyboru) i `game` (rozgrywka) to osobne konteksty z osobnymi bazami.
Mod Polska **powtarza** `ImportFiles` i `UpdateText` w obu grupach. Jeśli cywilizacja
działa w grze, ale nie widać jej w menu — brakuje grupy `shell`.

## 4. Pusta karta syncretismu ✅ (udokumentowane przez autora moda)

Ekran syncretismu rozwiązuje cywilizację przez tabele **legacy**, zanim sięgnie po
`CivSelfSyncretismUnlocks`. Bez wierszy w `LegacyCivilizations`
i `LegacyCivilizationTraits` efekt zostanie przyznany, ale **karta w UI będzie pusta**.

## 5. `Controls.decorate` nie działa wstecz ✅

Komentarz Firaxis w `component-support.js`:
> nie utworzy dekoratora dla istniejących instancji komponentu

Dekorator musi być zarejestrowany **zanim** komponent powstanie → dlatego `LoadOrder`
ma znaczenie, i dlatego mody UI często ustawiają `1000`.

## 6. Patch prototypu trzeba zabezpieczyć strażnikiem ✅

Bez `if (Klasa.patched === proto) return;` przy wielu instancjach komponentu obudujesz
tę samą metodę wielokrotnie — każda instancja doda kolejną warstwę.

## 7. Zawsze `engine.off` ✅

306 wywołań `engine.on` vs 96 `engine.off` w modach — wielu autorów tego nie sprząta.
Brak wyrejestrowania w `beforeDetach`/`afterDetach` = wycieki i wielokrotne handlery.

## 8. `<EnglishText>` vs `<LocalizedText>` ✅

```xml
<EnglishText><Row Tag="LOC_X"><Text>Hello</Text></Row></EnglishText>
<LocalizedText><Row Tag="LOC_X" Language="pl_PL"><Text>Cześć</Text></Row></LocalizedText>
```
Inny tag **i** atrybut `Language` tylko w drugim. Pomylenie = brak tłumaczenia.

## 9. Dwa różne `<LocalizedText>` ✅

- W `<Actions>`: nie istnieje — teksty gry ładuje `<UpdateText><Item>`
- Na poziomie `<Mod>`: `<LocalizedText><File>` — teksty **samego modinfo**
  (nazwa moda na liście). Używa `<File>`, nie `<Item>`.

## 10. `ImportFiles` nadpisuje pliki gry po ścieżce ✅

Jeśli twój plik ma **tę samą ścieżkę względną** co plik gry
(np. `ui-next/tooltips/plot-tooltip/plot-tooltip.js`), zastąpi go.
Bywa zamierzone — ale łatwo zrobić to przez przypadek, nazywając folder jak w grze.

## 11. Pliki zasobów duplikowane bez rozszerzenia ❓

Mod Polska rejestruje każdy zasób dwa razy:
```xml
<Item>Art/civ_poland.png</Item>
<Item>Art/civ_poland</Item>
```
i faktycznie ma na dysku oba pliki. Nie ustaliłem, czy to wymóg gry, czy nadmiarowość.
**Jeśli ikony nie działają — spróbuj tego wzorca.**

## 12. Alias `fs://game/` = identyfikator moda ✅ (z wyjątkami ❓)

Zweryfikowane: `bz-map-trix`, `leugi-diploribbon-tweaks`, `detailed-map-tacks`,
`maple-leaves-more-lens` — alias zgadza się z `<Mod id=...>`.

Ale są rozbieżności:
| Mod ID | Używany alias |
|---|---|
| `szczupakabra-poland` | `codex-poland-civilization` (25×, konsekwentnie) |
| `f1rstdan-cool-ui` | `f1rstdans_cool_ui` (3×) obok `/f1rstdan-cool-ui/` (5×) |
| — | `RHI_mod`, `yield_influence_5` |

❓ Nie wiem, czy to martwe ścieżki (i te zasoby po prostu nie działają), czy alias
bierze się skądinąd (np. nazwy folderu przy dystrybucji spoza Workshop).
**Bezpieczne założenie: używaj dokładnie swojego `<Mod id>`.**

## 13. `AiLists.LeaderType` przechowuje TRAIT, nie lidera ✅

```xml
<Row ListType="Ada Lovelace Yield Biases"
     LeaderType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" System="YieldBiases"/>
```
Nazwa kolumny myli — wartość to typ cechy.

## 14. Boolean: XML vs SQL ✅

- XML: `IsMajorLeader="true"` (Firaxis) — choć `"1"` też występuje
- SQL: `1` / `0`

## 15. Dane epokowe tylko w grupie z `AgeInUse` ✅

Tabele i typy epoki istnieją tylko, gdy epoka jest aktywna. Wrzucenie danych
starożytności do grupy `always` może się nie powieść.

## 16. `UniqueCultureProgressionTree` zmienia się per epoka ✅

Mod Polska ustawia wartość w `shared.sql`, a potem **nadpisuje `UPDATE`-em**
w pliku każdej epoki. Jedna wartość nie wystarczy.

## 17. Model 3D to ślepa uliczka ✅

Wymaga paczki `.dep` z GUID-ami i zależnościami bibliotek materiałów — brak publicznych
narzędzi. **Używaj `VisualRemaps`**, żeby pożyczyć istniejący model.

## 18. `grep -r` po katalogu gry jest bardzo wolny ✅

1,3 GB. Zawężaj: `Base/modules/*/data/` to tylko 14 MB i 461 plików XML.
(Jedno takie wyszukiwanie przekroczyło u mnie 120 s i musiało lecieć w tle.)

## 19. `Package` — wielkość liter ❓

44 mody mają `<Package>Mod</Package>`, 2 mają `MOD` i działają.
Prawdopodobnie porównanie ignoruje wielkość liter, ale trzymaj się `Mod`.

## 20. Dokumentacja społeczności miesza tabele z Civ VI ❗ ✅

Przewodnik o liderach na `civ7community.mintlify.app` wymienia `LeaderCivilizations`,
`Agendas`, `HistoricalAgendas`, `RandomAgendas` — **żadna z nich nie istnieje w Civ VII**.
Zawsze sprawdzaj nazwę tabeli w schemacie, zanim jej użyjesz:
```bash
grep -c "CREATE TABLE 'NazwaTabeli'" "$G/Base/Assets/schema/gameplay/01_GameplaySchema.sql"
```
Pełna analiza: [22-source-evaluation.md](22-source-evaluation.md).

## 21. Nie projektuj wokół systemów, których nie ma ✅

Brak: Wiary (Faith), Turystyki, Lojalności, Amenities, Gubernatorów.
Zadowolenie (`YIELD_HAPPINESS`) jest **yieldem na turę**, nie pulą.
Szczegóły: [21-gameplay-mechanics.md](21-gameplay-mechanics.md).

## 22. ❗ Mody użytkownika NIE idą do `Documents\My Games` ✅

Najkosztowniejsza pułapka na starcie — mod po prostu nie pojawia się na liście
i nie ma po nim śladu w logach.

**Poprawna ścieżka:**
```
C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\
```
`Documents\My Games\Sid Meier's Civilization VII\` zawiera **wyłącznie `Saves\`**.
Ścieżka z `Documents` to konwencja z Civ VI — powtarza ją też dokumentacja
społeczności, więc łatwo się na tym przejechać.

Diagnostyka: jeśli `grep -i "nazwa-moda" Modding.log` nic nie zwraca, a w logu jest
`Discovered 0 mods.` — mod leży w złym miejscu. Gra wypisuje skanowaną ścieżkę wprost:
```
Discovering new mods...
C:/Users/najan/AppData/Local/Firaxis Games/Sid Meier's Civilization VII/Mods/
```

## 23. Panel może mieć kilka ramek — dekorujesz nie tę, którą widać ✅

`panel-place-population` buduje **trzy** niezależne ramki i przełącza je klasą `hidden`
w zależności od stanu:
```js
this.subsystemFrame.classList.toggle("hidden", state != PlacePopulationSelectionState.NONE);
this.placeImprovementFrame.classList.toggle("hidden", state != ...ADD_IMPROVEMENT);
this.placeSpecialistFrame.classList.toggle("hidden", state != ...ADD_SPECIALIST);
```
Wstrzyknięcie DOM do `subsystemFrame` jest **niewidoczne** w trybie dodawania
specjalisty — bo wtedy ta ramka jest ukryta. Kosztowało mnie to jedną iterację
z „mod się ładuje, ale nic nie widać".

**Diagnostyka:** jeśli dekorator działa (brak błędów w `UI.log`), a zmian nie widać —
sprawdź, czy komponent nie ma kilku kontenerów przełączanych `hidden`, i wstrzykuj
do tego odpowiadającego bieżącemu stanowi. Zrzut ekranu z gry mówi wprost, którą
ramkę widzisz (tu: nagłówek „DODAJ SPECJALISTĘ" = `placeSpecialistFrame`).

## 24. Yieldy i utrzymanie to w UI dwie osobne listy ✅

`WorkerPlacementInfo` niesie **cztery** tablice: `CurrentYields`/`NextYields`
oraz `CurrentMaintenance`/`NextMaintenance`. Gra rysuje je jako osobne grupy pigułek.
Koszty utrzymania specjalisty (np. −2 jedzenia, −2 zadowolenia) siedzą **wyłącznie**
w `*Maintenance`, a nie w `*Yields`.

Konwencja znaków w kodzie gry:
```js
yieldChange      = NextYields[i] - CurrentYields[i]          // dodatnie = zysk
maintenanceChange = CurrentMaintenance[i] - NextMaintenance[i] // JUŻ ujemne gdy koszt rośnie
```
Czyli oba można po prostu **zsumować**, żeby dostać zmianę netto per yield.
Jeśli liczysz „co daje specjalista" i pominiesz `*Maintenance`, zgubisz całą stronę
kosztową — a to zwykle najbardziej interesuje gracza.

## 25. Warstwa soczewki rejestruje się PÓŹNIEJ niż skrypt moda ✅

`LensManager.registerLensLayer(...)` wykonuje się dopiero, gdy gra zaimportuje plik
warstwy — a to następuje **po** wykonaniu skryptów moda, nawet przy `LoadOrder=1000`.
Patchowanie warstwy w `engine.whenReady` **cicho nic nie robi**:
```
najane-specialists: 'fxs-worker-yields-layer' never registered, tile pills not patched
```
Rozwiązanie: ponawiaj próbę patcha przy `InterfaceModeChangedEventName` (albo przy
zdarzeniach `lens-event-layer-enabled`) i zabezpiecz flagą „już spatchowane".
Wtedy najpóźniej przy wejściu w tryb, który używa warstwy, patch się zastosuje.

⚠️ Zawsze loguj przypadek „nie znalazłem obiektu do spatchowania" — inaczej mod
wygląda na działający, a po prostu nic nie robi. (Patrz [19](19-workflow-and-debugging.md):
do logu używaj `console.error`, bo `console.log` nie trafia do `UI.log`.)

## 26. `fxs-subsystem-frame` przestawia dzieci — wstawiaj DOM w `waitForLayout` ✅

Wstawienie elementu przez `insertBefore(...)` w `afterAttach()` dekoratora **nie trafia
tam, gdzie każesz** — ramka `fxs-subsystem-frame` po zbudowaniu przenosi swoje dzieci
do wewnętrznego kontenera przewijania, więc Twój element ląduje na końcu.
Objaw: sekcja miała być na górze, a jest na samym dole (bez żadnego błędu w logu).

Rozwiązanie — użyj globalnego `waitForLayout()` (z `core/ui/component-support.js`,
dostępny **bez importu**; odracza o 2 klatki animacji, `LAYOUT_FRAME_DELAY = 2`):

```js
afterAttach() {
    waitForLayout(() => {
        const anchor = this.component.jakisKontener;
        anchor?.parentElement?.insertBefore(mojaSekcja, anchor);  // dopiero teraz
    });
}
```
Tak samo robi sama gra w `panel-place-population.js` — po `buildView()` poprawia
scrollbary właśnie wewnątrz `waitForLayout`.

⚠️ Zawsze celuj w `anchor.parentElement`, a nie w samą ramkę — po przestawieniu
rodzicem kotwicy nie jest już ten element, do którego ją pierwotnie dodano.

## 27. Niespójność gry: „Utrzymanie specjalistów" znika przy zajętym polu ✅

W `model-place-population.js` linia utrzymania pokazuje się **tylko** gdy surowa
wartość jest dodatnia:
```js
if (nextValue > 0) { this.showAfterSpecialistMaintenance = true; ... }
```
Na polu, które **już ma specjalistę**, warunek nie przechodzi i opis znika — mimo że
pasek „WYNIKI" nad nim poprawnie uwzględnia koszt, bo liczy się z innego pola
(`overallChange = bonusChanges - maintenanceChanges`). Dane są, tylko nie są pokazane.

> ❌ **PRÓBA OBEJŚCIA NIEUDANA — WYCOFANA.** Opakowałem `PlacePopulation.update`
> i uzupełniałem brakującą linię z `NextMaintenance`/`CurrentMaintenance`. Efekt:
> **liczby były błędne** — pokazywały ok. dwukrotność rzeczywistego kosztu i nie zgadzały
> się z paskiem „WYNIKI" w tym samym oknie. Wycofane; lepiej nie pokazać nic niż pokazać
> nieprawdę.

❓ **Czego nie wiem:** co dokładnie znaczą `CurrentMaintenance` / `NextMaintenance`
w `WorkerPlacementInfo`. Obserwacje z gry przeczą wszystkim moim hipotezom:
- „Premia za specjalistów" pokazuje **sumy** (1 specjalista +5 kultury → 2 specjalistów +10),
  a pasek „WYNIKI" **przyrost** (+5)
- przy tej samej interpretacji dla utrzymania liczby się nie domykają — wartości
  z `*Maintenance` nie odtwarzają przyrostu widocznego w „WYNIKI"

**Zanim ktoś spróbuje ponownie:** najpierw zrzuć realne wartości obu tablic dla pola
z 0 i z 1 specjalistą (`console.error`, patrz [19](19-workflow-and-debugging.md)) i dopiero
na tych danych buduj wzór. Bez tego to zgadywanie.

**Lekcja ogólna (nadal aktualna):** zanim uznasz, że dane nie istnieją, sprawdź, czy nie
chodzi tylko o warunek wyświetlania — ten sam koszt bywa liczony w dwóch miejscach
z dwoma różnymi warunkami. Ale **nie dopisuj własnych liczb, dopóki nie rozumiesz
jednostek** — wynik gorszy niż brak.

## 28. Keybindingi tylko w `scope="shell"` — inaczej rollback CAŁEJ akcji ✅

Tabele `InputActions`, `InputActionDefaultGestures`, `InputContextConstraints` należą
do bazy **frontendu**, nie gameplay. Wpięcie pliku z nimi w grupę `scope="game"` kończy się:

```
[gameplay] ERROR: no such table: InputActions
ERROR: There were errors loading 'config/input.xml' that require a rollback.
Warning: Apply Actions - Errors when applying action '...(UpdateDatabase)'. Rollback Required.
```

❗ **Najgorsze jest to, że rollback wycofuje całą akcję `UpdateDatabase` tej grupy**,
nie tylko wadliwy plik. Jeden błędnie umieszczony plik potrafi więc wyłączyć wszystkie
pozostałe dane moda — bez widocznego objawu poza brakiem działania.

Poprawnie: `config/input.xml` **wyłącznie** w grupie `scope="shell"`. Akcja i tak
działa potem w rozgrywce, bo definicje żyją w bazie konfiguracji.

Wzorzec sprawdzony w `bz-map-trix` (`config/bz-input.xml`, tylko shell) — używa też
`<Replace>` zamiast `<Row>`, co jest idempotentne przy ponownym zastosowaniu moda.

## 29. Modyfikator jako skrót: `EventType="All"` ✅

⚠️ **Korekta** wcześniejszego wniosku z [09](09-cookbook-ui-mod.md), że system akcji
gry nie potrafi wyrazić „klawisz jest trzymany". Potrafi — trzeba tylko zadeklarować:

```xml
<Replace ActionId="moja-akcja" DeviceType="Keyboard" EventType="All"
         Name="LOC_..." Description="LOC_..." />
<Replace ActionId="moja-akcja" Index="0" GestureType="KBMouse" GestureData="KEY_CONTROL"/>
```
Wtedy `InputEngineEvent` zgłasza `InputActionStatuses.START` przy wciśnięciu
i `FINISH` przy puszczeniu. Statusy: `START`, `FINISH`, `UPDATE`, `DRAG`, `HOLD`.

✅ **Sam modyfikator jest poprawnym przypisaniem** — gra robi tak dla własnej akcji
`keyboard-camera-modifier` (`GestureData="KEY_ALT"`). Dostępne m.in. `KEY_CONTROL`,
`KEY_SHIFT`, `KEY_ALT` oraz warianty L/R. Kombinacje: `KEY_SHIFT+KEY_S`.

To rozwiązanie jest **znacznie lepsze niż nasłuch DOM**: wpis pojawia się w
Opcje → Sterowanie, gracz może go przemapować, i nie koliduje z innymi modami
reagującymi na ten sam klawisz.

Etykietę aktualnego przypisania do UI pobierzesz przez:
```js
Input.getGestureDisplayString(actionId, 0, InputDeviceType.Keyboard, InputContext.ALL)
```
⚠️ Do wyświetlenia użyj `Locale.compose(key, label)` i `textContent` — atrybut
`data-l10n-id` nie przyjmie argumentu wyliczonego w czasie działania.

## 30. Patchując metodę, którą patchuje inny mod — sprawdź, co tracisz ✅

Realny konflikt: City Hall (`bz-city-hall`) opakowuje `updateSpecialistPlot`
w poprawny sposób — woła poprzednią wersję, potem dorysowuje swoje ikony:
```js
const prev = WYLL.updateSpecialistPlot;
WYLL.updateSpecialistPlot = function(...a) { prev.apply(this, a); this.realizeBuildSlots(d); }
```
Mój mod patchował **po nich** i **nie delegował** (celowo — zastępuje rysowanie).
Skutek: ich ikony znikały, a wracały tylko w trybie „pokaż oryginał", bo tam
delegowałem. Objaw wyglądał jak konflikt o klawisz, a był o kolejność patchy.

**Diagnostyka:** jeśli funkcja innego moda działa tylko wtedy, gdy Twój mod jest
„wyłączony", prawie na pewno przerywasz łańcuch wrapperów.

**Rozwiązanie** (gdy nie możesz delegować): wywołaj ich krok samodzielnie, z wykrywaniem
funkcji, żeby brak tamtego moda był no-opem:
```js
if (typeof this.realizeBuildSlots === "function" && this.bzGridSpritePosition) {
    try { /* powtórz ich kroki */ } catch (e) { console.error(...); }
}
```
⚠️ To wiąże Cię z wewnętrznymi nazwami cudzego moda. Osłoń `try/catch`, wykrywaj
funkcje, i zapisz w komentarzu, czyj kod odtwarzasz — przy jego aktualizacji ktoś
musi wiedzieć, gdzie szukać.

✅ Efekt uboczny: wołając ich metody przez `this` (np. `getSpecialistPipOffsetsAndScale`),
dostajesz też ich ulepszenia za darmo.

## 31. Nie edytuj plików w `Program Files` ✅

Steam nadpisze zmiany przy weryfikacji plików, a zapis wymaga uprawnień administratora.
Mody z Workshop (`steamapps\workshop\content\1295660`) też są zarządzane przez Steam —
Twoje zmiany zostaną nadpisane przy aktualizacji subskrypcji.

## 32. Ekran, którego nie da się udekorować — bo nie jest w starym frameworku ✅

**Data: 2026-08-10.** Zanim napiszesz `Controls.decorate('nazwa-ekranu', …)`, sprawdź,
czy ekran nie żyje w `ui-next/` (Solid.js). Objawem będzie „dekorator się rejestruje,
nic się nie dzieje" — bo element tej nazwy jest wprawdzie zdefiniowany, ale jego treść
renderuje Solid, a nie DOM z `.html.js`.

```bash
find "…/Base/modules" -ipath "*ui-next*" -iname "*nazwa*"
```

Rozpoznanie po plikach: obok `.js` leży `.js.map`, z którego wyciąga się **`.tsx`**
(a nie `.ts`), a w kodzie widać `ComponentRegistry.register` / `createSignal`.
Pełny opis: [25-ui-next-solidjs.md](25-ui-next-solidjs.md).

## 33. Ten sam ekran istnieje w DWÓCH wersjach naraz ✅

Przy migracji do `ui-next` Firaxis **zostawia stare pliki na dysku i nadal je ładuje**.
Ekran Handlu ma komplet w `ui/resource-allocation/` (stary) i w
`ui-next/screens/commerce/` (nowy) — oba wpisane w `base-standard.modinfo`.

Wygrywa nowy, bo `defineLegacyComponent` woła `Controls.define` z `priority: 1`,
a stary `Controls.define` domyślnie ma `0`. Mod edytujący stary plik nie zmieni nic
widocznego. **Zawsze sprawdź, czy nie ma drugiej wersji ekranu**, zanim zaczniesz
czytać kod „tego oczywistego" pliku.

## 34. `overridePriority` w `ui-next` znosi problem kolejności ładowania ✅

W starym frameworku patch musiał trafić na już istniejący obiekt (quirk o warstwach
soczewek). `ComponentRegistry.register` / `ModelRegistry.register` trzymają **jedną
owiniętą fabrykę na nazwę** i tylko podmieniają w niej wskaźnik na implementację,
gdy przychodzi wyższy priorytet — więc mod może zarejestrować się przed grą albo po
niej, efekt jest ten sam.

⚠️ Wyjątek: to działa tylko dla komponentów faktycznie **zarejestrowanych** w rejestrze.
Zwykły `export const Foo` bez `register` jest nietykalny. I nadpisanie modelu w
`ModelRegistry` nic nie da, jeśli konsument woła fabrykę bezpośrednio, zamiast
`Model.get()` — dokładnie tak robi ekran Handlu.

## 35. `overridePriority` na ekranie Handlu jest już zajęte przez Resource+ ✅

**2026-08-10.** Mod **Resource+** (`brads-assign-all-resources`, Workshop 3756000777)
rejestruje `CommerceResourcesContainer` z `overridePriority: 1100`. Kto chce ruszyć
tę samą zakładkę, musi dać więcej **i delegować** do `originalFactory(props)` —
inaczej funkcje Resource+ po prostu znikną (to samo, co quirk #30 z City Hall).

Ogólna zasada: **zanim wybierzesz `overridePriority`, sprawdź, czy nazwa nie jest już
zajęta** przez zainstalowany mod:

```bash
grep -rn "overridePriority" "…/steamapps/workshop/content/1295660"
```

## 36. Nadpisując komponent `ui-next`, sprzątaj po sobie w `onCleanup` ✅

Wstrzyknięty goły DOM (przyciski, `<style>` w `document.head`, klasy dopisane do
cudzych elementów) **nie zniknie sam** — Solid usuwa tylko to, co sam wyrenderował.
Resource+ usuwa w `onCleanup` każdy element z osobna, odpina `MutationObserver`,
listenery i `requestAnimationFrame`. Ekran Handlu otwiera się i zamyka wiele razy
w partii, więc brak sprzątania = narastające duplikaty.

## 37. Mod „się ładuje", a jego skrypty nie ruszają — bo nie jest WŁĄCZONY ✅

**2026-08-10.** `Modding.log` pokazywał „Loading Mod – …", „Discovered 1 mods" i nazwę
moda, a mimo to w `UI.log` nie było ani jednej linii z jego skryptu — nawet markera
z pierwszej linii pliku wejściowego. Żadnego błędu importu też nie.

**Discovery ≠ enabled.** Gra skanuje katalog `Mods\` przy każdym starcie, ale mod trzeba
jeszcze włączyć w *Menu główne → Dodatkowa zawartość → Mody*. Nowy mod domyślnie
**nie** jest włączony.

Jak to sprawdzić bez zgadywania — `Modding.log` wypisuje przy „Applying mod components"
listę **rzeczywiście włączonych** modów wraz z ich grupami akcji:

```
[…] najane-common-specialists-yields (Better Specialists UI by Najane)
[…]  * najane-specialists-ui
[…] holistic-qol-plus (Holistic QoL+)
[…]  * holistic-qol-plus-game
[…] Applying mod components.
```

Nie ma Twojego moda na tej liście → jest wyłączony, i żadne debugowanie kodu nie pomoże.

Druga droga — `Mods.sqlite`, kolumna `Mods.Disabled` (⚠️ nie `Enabled`; `NULL` znaczy
„nie wyłączony"). Rejestrację akcji można potwierdzić tak:

```python
import sqlite3
c = sqlite3.connect('Mods.sqlite'); c.row_factory = sqlite3.Row
for r in c.execute("select ModRowId, ModId, Version, Disabled from Mods where ModId='twoj-mod'"):
    print(dict(r))
# ActionGroups(ActionGroupRowId, ModRowId, ActionGroupId, Scope, CriteriaRowId)
# Actions(ActionRowId, ActionGroupRowId, ActionType)
# ActionItems(ActionRowId, Arrangement, Item)
```

## 38. Pliki tekstowe innych języków: `<LocalizedText Language>`, nie `<EnglishText>` ✅

**2026-08-10.** `text/pl_PL/ModInfoText.xml` z blokiem `<EnglishText>` wjeżdża do bazy
jako `en_US` — **nazwa katalogu nie ma znaczenia** — i zderza się z prawdziwym
`text/en_us/…`:

```
ERROR: Database: UNIQUE constraint failed: LocalizedText.ModRowId, LocalizedText.Tag, LocalizedText.Locale
Database: While executing - 'INSERT INTO LocalizedText(...) VALUES(485,'LOC_…','en_US','…')'
```

Poprawnie:

```xml
<Database>
    <EnglishText>                                  <!-- tylko text/en_us/ -->
        <Row Tag="LOC_X"><Text>English</Text></Row>
    </EnglishText>
</Database>

<Database>
    <LocalizedText>                                <!-- każdy inny język -->
        <Row Tag="LOC_X" Language="pl_PL"><Text>Polski</Text></Row>
    </LocalizedText>
</Database>
```

Dotyczy tak samo plików z `<LocalizedText><File>` w `.modinfo`, jak i tych z `UpdateText`.

## 39. `version="0.1"` w `.modinfo` = mod cicho pomijany ❗✅

**2026-08-10, kosztowało dwie rundy debugowania.** Atrybut `version` na elemencie
`<Mod>` jest parsowany jako **liczba całkowita**. `version="0.1"` ląduje w
`Mods.sqlite` jako `Version = 0`, a gra takiego moda **nie stosuje** — przy czym:

- `Modding.log` normalnie pisze „Loading Mod – …" i „Discovered 1 mods",
- mod jest widoczny i **zaznaczony jako włączony** w menu Modów,
- w `Mods.sqlite` ma `Disabled = 0`, wszystkie `ActionGroups` / `Actions` /
  `ActionItems` zarejestrowane poprawnie,
- **ale nie pojawia się na liście włączonych modów** wypisywanej tuż przed
  „Applying mod components", a jego skrypty nigdy się nie wykonują —
  **bez jednego komunikatu o błędzie**.

Dowód liczbowy z instalacji użytkownika: 93 mody w bazie, 40 zastosowanych, wszystkie
zastosowane mają `Version >= 1`, a jedyny mod z `Version = 0` to był właśnie ten
niedziałający.

```xml
<Mod id="moj-mod" version="1" xmlns="ModInfo">     <!-- ✅ liczba całkowita >= 1 -->
    <Properties>
        <Version>0.1</Version>                      <!-- ✅ tu dowolny tekst -->
```

Uwaga: `version="1.3"` też „działa", ale zapisuje się jako `1` — część po kropce jest
po prostu obcinana. Numer wersji do pokazania graczowi trzymaj w `<Properties><Version>`.

**Szybka diagnoza dowolnego „mod się nie uruchamia":**

```python
import sqlite3
c = sqlite3.connect('Mods.sqlite')
print(list(c.execute("select ModId, Version, Disabled from Mods where Version = 0")))
```

Patrz też #37 (mod znaleziony ≠ włączony) — objawy są identyczne, przyczyny różne,
a rozstrzyga ta sama lista w `Modding.log`.

## 40. Klik z modyfikatorem nie dociera jako akcja silnika ❗✅

**2026-08-10, cztery rundy testów.** Silnik **nie wysyła akcji `mousebutton-right`
(ani zapewne innych akcji myszy), gdy trzymany jest modyfikator**. Shift+PPM generuje
wyłącznie natywne zdarzenia DOM `mousedown`/`mouseup` — i nic poza tym.

Objaw mylący: wygląda to jak „nie umiem odczytać Shifta". Straciłem dwie rundy na
podmienianie źródła stanu modyfikatora (DOM `keydown` → `Input.isShiftDown()`),
podczas gdy **oba działały poprawnie** — po prostu nigdy nie były pytane, bo zdarzenie,
w którym pytałem, nie przychodziło.

**Obsługuj kliknięcia z modyfikatorem przez DOM** (`mousedown`/`mouseup`, `button === 2`,
`event.shiftKey`), a akcję silnika przechwytuj tylko po to, żeby zdusić domyślne
„cancel". Szczegóły i kolejność zdarzeń: [25-ui-next-solidjs.md](25-ui-next-solidjs.md).

**Reguła na przyszłość:** przy trzeciej nieudanej hipotezie o wejściu **przestań
zgadywać i podłącz podsłuch** logujący każde `engine-input` (nazwa + status +
współrzędne) oraz natywne `mousedown`/`mouseup`/`keydown` z flagami modyfikatorów.
Jedna runda z danymi rozstrzygnęła to, czego trzy rundy teorii nie potrafiły.
Podsłuch musi pomijać status `InputActionStatuses.UPDATE` — powtarza się co klatkę.

## 41. `Game.PlayerOperations.sendRequest` kolejkuje — `canStart` w tym samym takcie kłamie ✅

**2026-08-10.** Operacje gracza nie wykonują się natychmiast. Po `sendRequest` stan
gry jeszcze się nie zmienił, więc `canStart` zapytane o **następną** operację w tej
samej funkcji odpowiada na podstawie starego stanu.

Objaw z praktyki (ekran Handlu, usuwanie wielbłąda dającego 2 sloty): mod wysyłał
jednym ciągiem zwolnienie 2 zasobów i zwolnienie wielbłąda. Zasoby wychodziły,
wielbłąd dostawał odmowę i **zostawał**. Kolejne kliknięcie znów zwalniało 2 zasoby —
tym razem niepotrzebnie, bo miejsce już było.

**Rozwiązanie:** rób to sekwencyjnie i po każdym kroku czekaj na potwierdzenie ze
strony silnika (zdarzenie dziedzinowe, np. `ResourceUnassigned`), z limitem czasu na
wypadek operacji, która przepadnie. Dodatkowo **pytaj `canStart` zamiast liczyć**,
ile kroków przygotowawczych potrzeba: zwalniaj po jednym i przerwij, gdy właściwa
operacja przejdzie. Wtedy nie trzeba znać reguły silnika, a skutki uboczne są minimalne.

```js
if (trySend(target)) return;                 // może nic nie trzeba
while (queue.length) {
    if (trySend(queue.shift())) await waitForEngineEvent();
    if (trySend(target)) return;
}
```

## 42. Podświetlając coś w cudzym UI, użyj efektu, który ten UI już ma ✅

Własna żółta obwódka wyglądała jak nowy rodzaj dekoracji i użytkownik od razu ją
odrzucił. Ekran Handlu ma już własny efekt najechania — `hover\:scale-125` w klasach
`DraggableResource` — więc podświetlenie „to samo, co pod kursorem" powinno być
**dokładnie tym samym powiększeniem**, a nie nową konwencją.

⚠️ Celuj w element renderowany w **każdej** gałęzi komponentu. W tym wypadku
`.framed-resource` (jest zawsze), a nie `.draggable-resource` (tylko gdy zasób jest
interaktywny).

❗ **Nie nakładaj swojego efektu na element, który gra już transformuje** — `transform`
się **mnoży**. Powielenie `scale(1.25)` gry własnym `scale(1.25)` dało na najechanym
elemencie 1.5625, czyli wyraźnie większy od reszty podświetlonej grupy. Pomiń element,
którym gra zajmuje się sama — i sprawdź to warunkiem, a nie założeniem, bo efekt
najechania bywa renderowany tylko w części gałęzi komponentu:

```js
const gameHandlesIt = slotElement.querySelector('.draggable-resource') !== null;
```

## 43. W `ui-next` `onMount` NIE gwarantuje, że treść komponentu jest już w DOM ✅

**2026-08-10.** Mod szukał w `onMount` elementu wewnątrz zakładki i **za każdym razem**
dostawał `null`, mimo że element chwilę później był na ekranie.

Przyczyna: `CommerceScreenBaseTabContent` opakowuje treść zakładki w
**`<ThrobberSuspense>`**. Solidowy Suspense renderuje najpierw zastępnik, a właściwą
treść dopiero po rozwiązaniu zasobów (wstępne ładowanie obrazów i styli przez
`ComponentRegistry`). `onMount` biegnie na zastępniku.

To odwrotność pułapki #1 ze starego frameworka („patchujesz obiekt, którego jeszcze nie
ma"): tu obiekt istnieje, ale **jego zawartość jeszcze nie**.

**Rozwiązanie — szukaj, aż znajdziesz, i dopiero wtedy zawężaj:**

```js
function tryAttach() {
    const target = document.querySelector('[data-name="…"]');
    if (!target) return false;
    observer.disconnect();                    // szeroka obserwacja już niepotrzebna
    observer = new MutationObserver(onChange);
    observer.observe(target, { childList: true });   // wąska, tania
    return true;
}
if (!tryAttach()) {
    observer = new MutationObserver(tryAttach);
    observer.observe(document.querySelector('screen-…') ?? document.body,
                     { childList: true, subtree: true });
}
```

⚠️ Obserwując po to, żeby **dokładać własne klasy**, ustaw **wyłącznie `childList`**.
Przy `attributes: true` własna zmiana klasy wywoła Twój callback i zrobi się pętla.

## 44. Ukrycie elementu gry zabiera też jego marginesy ✅

`display: none` na wstawce tekstowej usunęło nie tylko tekst, ale i jej `my-4` z obu
stron — panel przykleił się do zakładek nad nim. Oddaj odstęp sąsiadowi:

```css
.ta-wstawka { display: none; }
.ta-wstawka + div { margin-top: 0.8888888889rem; }   /* tyle, co jedno my-4 */
```

Selektor `+` działa normalnie na elemencie z `display: none` — on nadal jest w drzewie.

## 45. Silnik DOM Civ VII nie ma nowszych metod `ParentNode` ✅

**2026-08-10.** `element.replaceChildren()` rzuca
`TypeError: replaceChildren is not a function`. Skutek był podstępny: kod działał
w Node przy sprawdzaniu składni, wchodził do gry bez błędu ładowania, a wywalał się
dopiero przy pierwszym użyciu — i tylko w `UI.log`.

Czyść i dodawaj dzieci po staremu:

```js
while (element.firstChild) element.removeChild(element.firstChild);   // zamiast replaceChildren()
for (const child of children) parent.appendChild(child);              // zamiast append(a, b, c)
```

⚠️ To samo dotyczy `append()` z wieloma argumentami — jest z tej samej generacji API
co `replaceChildren` i nie ma powodu zakładać, że akurat ono jest.

**Reguła:** w tym UI trzymaj się `appendChild` / `removeChild` / `insertBefore` /
`querySelector` / `classList` / `setAttribute`. Jeśli sięgasz po coś nowszego, sprawdź
najpierw, czy któryś działający mod z Workshop tego używa — Resource+ czyści kontenery
ręczną pętlą i teraz wiadomo dlaczego.

## 46. `UI.getIcon(type, "YIELD")` zwraca nazwę o MIESZANEJ wielkości liter ❗✅

**2026-08-10.** `UI.getIcon('YIELD_HAPPINESS', 'YIELD')` daje **`blp:Yield_Happiness`**,
a nie `blp:YIELD_HAPPINESS`. Kod, który wyciąga typ yieldu z nazwy ikony wyrażeniem
regularnym `/YIELD_[A-Z_]+/`, **nigdy nie trafia** — i cicho zwraca `null`, a wszystkie
zależne od tego wartości wychodzą 0.

Kosztowało to całą rundę: mechanizm sterowany zadowoleniem osad „nie działał", bo każde
zadowolenie odczytywało się jako zero. Ten sam błąd siedzi w modzie Resource+, z którego
funkcja została przeniesiona — czyli jego tryb „Balanced" też liczy na zerach.

**Nigdy nie parsuj nazwy ikony.** Zbuduj mapę tą samą funkcją, którą zbudowano wartość:

```js
const byIcon = new Map();
GameInfo.Yields.forEach((y) => byIcon.set(`url(${UI.getIcon(y.YieldType, 'YIELD')})`, y.YieldType));
// teraz: byIcon.get(entry.yieldIconSrc)
```

To ta sama zasada, co przy plakietkach yieldów w [26-commerce-screen.md](26-commerce-screen.md):
model zapisuje `url(${UI.getIcon(...)})`, więc odwrotne odwzorowanie też ma powstać
z `UI.getIcon`, a nie z domysłu o kształcie stringa.

## 47. Ten silnik CSS nie zna `:focus-visible` ✅

**2026-08-10.** Reguła z `:focus-visible` nie jest stosowana, a do `UI.log` leci
`Unsupported CSS pseudo class selector encountered: focus-visible` przy każdym
przeliczeniu arkusza — czyli zaśmieca log przy każdym otwarciu ekranu.

Używaj `:focus`. Obsługiwane są też `:hover`, `:active` i `:not(...)`; ⚠️ złożone
selektory w `:not()` potrafią wywalić `querySelector` (patrz błąd moda holistic-qol-plus
w logach: `Invalid CSS selector (.text-accent-1:not(.klasa))`).

## 48. Diagnostyka ma logować także decyzję o NIEROBIENIU niczego ✅

Mechanizm z opcją włącz/wyłącz, który przy wyłączonej opcji milczy, jest nieodróżnialny
od zepsutego. Po zgłoszeniu „nie zadziałało" log nie zawierał **niczego** i nie dało się
stwierdzić, czy problem leży w zdarzeniu, w wykrywaniu zmiany, czy po prostu opcja jest
wyłączona.

Każde wyjście wcześniejsze powinno zostawić ślad — z nazwą wyzwalacza i powodem:

```
[mod] TradeRouteAddedToMap: auto-assign is switched off (Options -> Mods)
[mod] TradeRouteAddedToMap: nothing new (37 resources owned)
[mod] TradeRouteAddedToMap: 1 newly acquired resource(s)
```

Jedna runda testu rozstrzyga wtedy wszystko naraz: czy zdarzenie w ogóle przyszło, czy
wykrywanie zadziałało i czy opcja jest włączona.

## 49. Nie inicjalizuj stanu „co już było" na pierwszym zdarzeniu ✅

**2026-08-10.** Mechanizm reagujący na *nowe* rzeczy musi znać stan wyjściowy. Naturalnie
prosi się o to, żeby zapamiętać go przy pierwszym zdarzeniu — i to jest błąd:
**pierwsze zdarzenie to zwykle dokładnie ta rzecz, na którą użytkownik czeka.**

U nas: gracz włączył opcję „przypisuj nowe zasoby", zawarł szlak handlowy, a log powiedział
`first pass, remembering 83 resources already owned` — czyli ten właśnie szlak został
zużyty na zbudowanie listy odniesienia i nic się nie przypisało. Z zewnątrz wygląda to
identycznie jak zepsuty ficzer.

**Inicjalizuj przy starcie, niezależnie od zdarzeń.** Skrypty moda ładują się, zanim gra
umie odpowiedzieć (`Players.get(GameContext.localPlayerID)` bywa jeszcze puste), więc
pytaj w pętli z ponowieniami zamiast czekać, aż coś się wydarzy:

```js
function seedWithRetries(attemptsLeft) {
    if (trySeed() || attemptsLeft <= 0) return;
    setTimeout(() => seedWithRetries(attemptsLeft - 1), 1000);
}
```

Zostaw też ścieżkę awaryjną w obsłudze zdarzenia (gdyby ponowienia się wyczerpały), ale
**zaloguj ją jako ostrzeżenie** — to znaczy, że założenie o starcie nie wyszło.

## 50. Trwały stan moda: `UI.setOption` z LICZBĄ, nie `localStorage` ✅

**2026-08-10.** Zapis własnego stanu moda (np. wyboru per miasto) wyłącznie do
`localStorage` **nie przetrwał przeładowania gry**. Kanałem, który działa, jest ten sam,
z którego korzystają opcje modów:

```js
UI.setOption('user', 'Mod', `${MOD_ID}.cokolwiek`, liczba);
Configuration.getUser().saveCheckpoint();   // ⚠️ bez tego nic się nie utrwala
…
const stored = UI.getOption('user', 'Mod', `${MOD_ID}.cokolwiek`);   // null gdy brak
```

❗ **Wartością musi być liczba.** Każde użycie `UI.setOption` w kodzie samej gry przekazuje
liczbę (0/1, opóźnienia, indeksy). Nie licz na to, że przejdzie JSON czy string — zakoduj
stan liczbowo.

Wzorzec na wyliczenie z możliwością „nie ustawiono": trzymaj **indeks + 1**, wtedy `0`,
`null` i `undefined` znaczą to samo („nigdy nie wybrano") i nie mylą się z pierwszą
pozycją listy. Listę kodów **tylko rozszerzaj** — przestawienie kolejności po cichu
przekłamie wszystko, co już zapisane.

⚠️ **Stan związany z konkretną partią kluczuj przez `Configuration.getGame().gameSeed`.**
Identyfikatory miast (`ComponentID.id`) są unikalne tylko w obrębie jednej gry — ten sam
numer w następnej kampanii to inne miasto. Ziarno jest stałe przez całe życie partii
i różne między partiami, więc dwa zapisy z tej samej kampanii dzielą ustawienia,
a niepowiązane gry nie mieszają się ze sobą.

Do zapisu gry (`AffectsSavedGames`) **nie** trzeba przy tym sięgać — i nie należy, jeśli
mod deklaruje `0`.

## 51. Wstrzyknięty element w drzewie Solid znika — potrzebny stały stróż ❗✅

**Objaw:** `insertBefore` na kontenerze renderowanym przez Solid działa (element
faktycznie trafia do DOM, brak błędu w logu), ale gracz go nie widzi. W logu widać
komplet wpisów z `onMount`, więc nic nie rzuciło wyjątku.

**Dwie przyczyny, obie występują naraz:**

1. **Solid przerysowuje kontener** i przy zamianie potomków potrafi usunąć także
   węzły, których sam nie tworzył.
2. **Komponent montuje się wielokrotnie** w trakcie jednej wizyty na ekranie
   (`right-click unassign active` pojawia się w logu po kilka razy). Nowy `onMount`
   potrafi wykonać się **przed** `onCleanup` poprzedniego — więc świeżo wstrzyknięty
   element zostaje natychmiast usunięty przez sprzątanie starego montowania.
   Dodatkowo `stop*()` operujące na modułowych singletonach (`observer`, `element`)
   rozłącza obserwator należący już do **nowego** montowania.

**Wzorzec, który działa** (`tab-icons.js`, `factory-first.js`):

```js
function ensureThing() {
    const host = document.querySelector(HOST_SELECTOR);
    if (!host) return false;                       // jeszcze nie ma — czekaj
    if (!host.querySelector(`.${CLASS}`)) host.insertBefore(build(), anchor);
    if (observedHost !== host) {                   // stróż zostaje podłączony
        observer?.disconnect();
        observedHost = host;
        observer = new MutationObserver(() => ensureThing());
        observer.observe(host, { childList: true });
    }
    return true;
}
```

- Obserwator **nie odłącza się** po pierwszym sukcesie — pilnuje i wstawia ponownie.
- Nie ma zapętlenia: własne wstawienie budzi obserwator, ten widzi element na miejscu
  i kończy bez zmian.
- `start*()` musi być idempotentne (jest wołane przy każdym montowaniu).
- `stop*()` **nie wolno** wołać z `onCleanup` zakładki — tylko przy pełnym demontażu.
  Sprzątanie usuwa wtedy `.${CLASS}` przez `querySelectorAll`, a nie przez zapamiętaną
  referencję, bo ta może wskazywać na element poprzedniego montowania.

**Rozpoznanie:** jeśli element jest w DOM, a nie widać go na ekranie — to jest ten
problem, nie CSS. Log wstawienia (`log('… added')`) natychmiast to rozstrzyga: przy tej
usterce pojawia się raz i nigdy więcej, mimo że element zniknął.

**⚠️ Aktualizacja po teście w grze: dla paska nagłówka ekranu Commerce to NIE wystarczyło.**
Stały stróż na `[data-name="filter-and-sort"]`.parentElement też nie dał efektu —
checkbox nie pojawił się ani razu, mimo braku błędu w logu i mimo że `onMount`
wykonywał się do końca. Ten konkretny pasek jest dla wstrzykiwania nieużywalny.

Wzorzec ze stróżem pozostaje słuszny tam, gdzie jest **potwierdzony w praktyce**
(`tab-icons.js`, pasek zakładek `[data-name="TabList"]`, `settlement-controls.js`,
nagłówki kart osad). Nie jest natomiast uniwersalną odpowiedzią.

**Reguła praktyczna:** własne kontrolki umieszczaj w kontenerze, który mod sam tworzy
i którego jest właścicielem — u nas pasek `.najane-assign-bar` doklejany do wiersza
zakładek. Wtedy nie ma czego pilnować. Przełącznik „Najpierw fabryczne" trafił
ostatecznie za znak zapytania w tym pasku i moduł `factory-first.js` sprowadza się do
funkcji wytwórczej elementu — zero obserwatorów, zero selektorów gry.

Kolejność prób, od najpewniejszej: **(1)** dołóż do własnego kontenera → **(2)** wstrzyknij
do kontenera gry ze stałym stróżem → **(3)** zawijaj komponent przez `ComponentRegistry`.

## 52. `Game.PlayerOperations.canStart` w pętli to główny koszt przypisywania ❗✅

**Objaw:** „Przypisz wszystkie" przy dużej puli działa bardzo wolno na początku i
przyspiesza w miarę opróżniania puli.

**Przyczyna:** planner liczy najlepszą parę (zasób, osada) od nowa przed każdym
przypisaniem, a dla każdej pary woła `canStart` — czyli round-trip do silnika.
Przy 170 zasobach i 15 osadach to 2550 wywołań na jedno przypisanie i ~430 000 na jedno
„Przypisz wszystkie". Koszt maleje liniowo z pulą, stąd przyspieszanie.

**Trzy poprawki, w kolejności zysku:**

1. **Pamiętaj odpowiedzi silnika.** Odpowiedź zmienia się wyłącznie dla osady, która
   właśnie coś wzięła (ubył jej slot, albo przybyły dwa po wielbłądzie). Reszta tabeli
   pozostaje ważna. Cache `${resourceValue}:${cityKey}` + unieważnianie **per osada**.
   ⚠️ Kluczem musi być `resourceValue`, a nie typ zasobu: kopie leżą na różnych polach,
   a połączenie z siecią handlową jest cechą pola.
2. **Punktuj jeden egzemplarz na rodzaj.** Punktacja nie czyta niczego poza typem zasobu,
   więc ocenianie 8 kopii bawełny to 8× ta sama praca. Grupuj pulę po `resourceType`,
   oceniaj reprezentanta, a pozostałe kopie trzymaj jako zamienniki na wypadek, gdyby
   silnik odmówił akurat tej.
3. **Nie pytaj bazy w pętli.** `Game.age === Database.makeHash('AGE_MODERN')` w funkcji
   wołanej per para to setki tysięcy zapytań o hash. Epoka nie zmienia się w trakcie gry —
   policz raz i zapamiętaj.

**Zasada ogólna:** wszystko, co w gorącej pętli sięga do `Game.*`, `GameInfo.*` albo
`Database.*`, ma być policzone raz i zapamiętane; jeśli wynik zależy od stanu, unieważniaj
punktowo to, co faktycznie się zmieniło, a nie cały cache.

## 53. Odwrotny apostrof w komentarzu CSS wywala CAŁY mod ❗✅

**Objaw:** mod przestaje się ładować w całości. W `UI.log`:

```
JS Error: fs://game/<mod>/ui/settlement-controls.js:70: SyntaxError: Unexpected identifier
SOURCE ERROR - fs://game/<mod>/ui/<punkt wejścia>.js
```

**Przyczyna:** style trzymamy w literałach szablonowych. Odwrotny apostrof napisany
**wewnątrz** takiego literału — choćby w komentarzu CSS, dla zacytowania nazwy właściwości —
**zamyka string**. Reszta bloku jest wtedy parsowana jako kod.

```js
const STYLE = `
/* ustawienie `margin-left: auto` nie zadziałało */   ← literał kończy się tutaj
.foo { display: flex; }
`;
```

**❗ `node --check` tego NIE wykrywa.** Powstały śmieć bywa poprawnym JavaScriptem dla
Node'a, a silnik gry odrzuca go bez litości. Wynik: „u mnie przechodzi walidację", a w grze
nie ładuje się nic.

**W komentarzach CSS używaj cudzysłowów, nigdy odwrotnych apostrofów.** Zwykłe komentarze
JSDoc (poza literałem) mogą je mieć bez ograniczeń — liczy się tylko wnętrze literału.

**Straż w `deploy.sh`** (blokuje wdrożenie, sprawdzona na podstawionym pliku):

```bash
awk -v f="$file" '
    /^const [A-Za-z_]+ = `$/ { inside = 1; next }
    inside && /^`;$/         { inside = 0; next }
    inside && /`/            { printf "%s:%d  %s\n", f, NR, $0 }
' "$file"
```

Wymaga trzymania konwencji: blok stylu zaczyna się linią `const NAZWA = ` + apostrof i
kończy linią z apostrofem i średnikiem — obie samodzielnie.

**Druga runda, gdy silnik przestaje być wąskim gardłem:**

4. **Odpytywanie modelu co 50 ms.** Każde przypisanie czeka, aż model pokaże zasób w nowym
   miejscu — ten interwał płaci się dwa razy na zasób. Zejście do jednej klatki (16 ms) to
   przy dużej puli różnica rzędu minuty. Niżej nie ma sensu: model przebudowuje się na
   granicy klatki.
5. **Podwójne czekanie na `isSlottingAvailable`** — raz na końcu iteracji, raz na początku
   następnej. Jedno wystarczy.
6. **Ta sama para liczona po kilka razy w jednym przebiegu.** `scorePair` bywa wołane z
   czterech gałęzi tej samej decyzji, `estimatedYieldBoosts` osobno dla priorytetu,
   produkcji i zadowolenia — wszystkie dla tej samej pary i wszystkie z tym samym wynikiem.
   Memoizacja **na jeden przebieg planowania**, kluczem `typ zasobu : osada`, czyszczona na
   starcie każdego przebiegu (stan planszy zmienia się między przebiegami).

**I najważniejsze: mierz, nie zgaduj.** Dwie rundy optymalizacji w tym module oparto na
domysłach i jedna z nich była chybiona. Pętla loguje teraz rozbicie na czas planowania
(nasz) i czas czekania na grę (nie nasz) — bez tego nie wiadomo, co poprawiać.

## 54. Nie steruj masową operacją przez model Solid — mierzone 30s vs. sekundy ❗✅

Pomiar na 111 zasobach, „Przypisz wszystkie" prowadzone przez model ekranu:

```
assigned 111 resource(s) in 30663ms (1610ms planning, 16346ms waiting, 276ms each)
```

**Podział 5 / 53 / 41:** planowanie 1,6 s (5%), czekanie na model 16,3 s (53%),
a 12,7 s (41%) nie mieściło się w żadnym pomiarze. Te 41% to były trzy wywołania modelu
na każdy zasób:

```js
model.clickAvailableResource({ resourceValue, cityID: undefined });
model.slotSelectedResource(cityID);
model.deselectSelectedResource();
```

Każde z nich mutuje store `createMutable` i wywołuje **przerysowanie ekranu**. Czyli trzy
zbędne przerysowania na zasób, a czwarte (to właściwe) opłacane w pozycji „czekanie",
bo pętla czekała, aż **ekran** się odbuduje, zanim zaplanowała kolejny ruch.

**Zasada:** operacja masowa ma rozmawiać z silnikiem i z silnika czytać:

- wysyłka: `Game.PlayerOperations.sendRequest` bezpośrednio, nie przez metody modelu;
- potwierdzenie: `Cities.get(cityID).Resources.getAssignedResources()` w pętli `requestAnimationFrame`;
- kolejny plan: z API gry (u nas `headless-model.js`), nie z modelu ekranu.

Ekran i tak się odświeży — słucha zdarzeń silnika — ale nic nie czeka, aż skończy.

⚠️ **Do potwierdzenia NIE używaj zdarzenia `ResourceAssigned`.** Zdarzenia silnika lecą dla
wszystkich graczy, więc AI przypisujące coś na drugim końcu mapy zwolni oczekiwanie za
wcześnie, a następny plan powstanie na niezmienionej planszy. Pytanie wprost osady jest
odporne na cudzą turę.

**Efekt uboczny, korzystny:** obie ścieżki (przycisk na ekranie i automat przy zamkniętym
ekranie) stają się jedną funkcją, bo obie czytają to samo źródło.

**Do czego uważać przy przejściu na dane z API:** model ekranu miał policzone `yieldDeltas`
i cechy budynków; licząc je samemu, przelicza się je raz na zasób. Chodzenie po
`city.Constructibles.getIds()` w każdej iteracji to realny koszt — budynki nie powstają
w trakcie przypisywania, więc wynik da się trzymać w cache'u na czas przebiegu i
unieważniać tylko dla osady, która właśnie coś wzięła.

**Runda trzecia — po przejściu na API gry planowanie DROŻEJE, i wiadomo dlaczego.**

```
przez model ekranu:  30663ms (1610ms planowanie, 16346ms czekanie)  276ms/zasób
przez API gry:       13169ms (4739ms planowanie,  8422ms czekanie)  119ms/zasób
```

Planowanie wzrosło 3×, bo model ekranu miał dochody osad **już policzone**, a licząc je
samemu łatwo sięgnąć po najdroższą funkcję:

⚠️ **`CityYields.getCityYieldDetails(cityID)` to NIE jest odczyt liczb.** Buduje drzewko,
które pokazuje tooltip dochodów: wartości bazowe, kroki modyfikatorów, zlokalizowane
etykiety. Wołane dla wszystkich osad przed każdym zasobem, było większością czasu
planowania. Tańszy odpowiednik czyta dokładnie to, co ta funkcja czyta przed ozdabianiem:

```js
const yields = city.Yields?.getYields();          // tablica indeksowana jak GameInfo.Yields
yields?.forEach((entry, index) => {
    const definition = GameInfo.Yields[index];
    if (definition) totals.set(definition.YieldType, Number(entry.value) || 0);
});
```

Drugi grzech tej samej klasy: model ekranu trzyma dochody jako `yieldIconSrc`
(`url(${UI.getIcon(type,'YIELD')})`) plus liczba, a punktacja mapuje ikonę z powrotem na
typ dochodu. Odtwarzanie tego kształtu w danych budowanych samodzielnie oznacza
budowanie stringów tylko po to, żeby je zaraz sparsować. Lepiej podać gotową mapę i
pozwolić punktacji ją wziąć wprost.

**Potwierdzanie operacji: `setTimeout` co 4 ms, nie `requestAnimationFrame`.** Silnik
przetwarza kolejkę na własnym takcie; sprawdzanie zsynchronizowane z klatką potrafi
przegapić moment o prawie całą klatkę — 16 ms na każdym zasobie, za darmo.

**Gdzie jest podłoga:** po tych zmianach większość czasu to samo przetwarzanie operacji
przez silnik. Zejście niżej wymaga wysyłania kolejnych przypisań bez czekania na
poprzednie, a to znaczy planowanie na nieaktualnej planszy — czyli utratę wyrównywania
(zadowolenie, fabryki). To jest wybór projektowy, nie optymalizacja.

## 55. Wieloliniowy tooltip: `\n` i `[N]` nie wystarczą — potrzebny `white-space` ❗✅

**Objaw:** treść `data-tooltip-content` z podziałami linii wyświetla się jako jeden akapit.
Ani `\n`, ani firaxisowy znacznik `[N]` nie robią różnicy, przy czym `[N]` **nie pojawia się
dosłownie** — więc wygląda, jakby był przetwarzany i ignorowany.

**Przyczyna** (`core/ui/tooltips/tooltip-controller.js`, `render()`):

```js
this.textElement.innerHTML = Locale.stylize(content);   // textElement to goły <div>
```

`Locale.stylize` zamienia `[N]` na **znak nowej linii**, a nie na `<br>`. W HTML znak nowej
linii zwija się do spacji jak każda inna biała spacja. Goły `<div>` nie ma
`white-space: pre-wrap`, więc podziału nie widać.

**Rozwiązanie — jedno i drugie:**

```js
lines.join('\n')            // zwykłe znaki nowej linii w treści
```
```css
.tooltip__content > div { white-space: pre-wrap; }   /* tu ląduje tekst tooltipa */
```

⚠️ Sama treść nie wymusi podziału — musi go **dopuścić CSS**. Reguła działa globalnie na
tekstowe tooltipy, więc wpinaj ją razem z arkuszem swojego ekranu, a nie na stałe.

Tooltipy z `data-tooltip-component` idą inną ścieżką (własny element dostaje treść w
atrybucie) i tej reguły nie potrzebują.

## 56. `node --check plik.js` NIE sprawdza modułów ES — cicho przepuszcza ❗✅

**To unieważnia sposób weryfikacji używany wcześniej w tym projekcie**, łącznie z uwagą
w quirku #53, że „Node akceptuje ten śmieć". Nie akceptuje — on go w ogóle nie parsuje.

`node --check <plik>.js` traktuje plik jako **CommonJS**. Widząc `import` poddaje się i
kończy z kodem **0**, nie mówiąc ani słowa. Każdy plik UI moda jest modułem ES, więc
komplet „syntax ok" z tego polecenia nic nie znaczył.

Sprawdzone na pliku z celowo wstawionym znakiem nowej linii wewnątrz literału:

```
node --check ui/screen/tab-icons.js                 -> exit 0, cisza
node --input-type=module --check < ui/screen/tab-icons.js -> exit 1, wskazuje linię 23
```

**Poprawne polecenie** czyta ze standardowego wejścia:

```bash
node --input-type=module --check < plik.js
```

Tak sprawdzony parser łapie **oba** znane zabójcze błędy: zabłąkany odwrotny apostrof w
literale szablonowym (quirk #53) i znak nowej linii w literale `'...'` — i zgłasza
dokładnie ten sam komunikat, który potem pojawia się w `UI.log`.

⚠️ **Kontrola ma stać na drodze do gry, nie w nawykach edytującego.** Druga połowa tej
wpadki: plik został sprawdzony ręcznie, potem jeszcze raz zmieniony i wdrożony bez
ponownej kontroli. `deploy.sh` sprawdza teraz każdy plik sam i odmawia wdrożenia.

**Uwaga na narzędzia pośredniczące:** sekwencja `\n` wpisana w skrypcie przekazywanym
przez warstwę powłoki potrafi zostać zamieniona na prawdziwy znak nowej linii, zanim
dotrze do pliku — tak powstały oba zepsute pliki tego dnia. Przy generowaniu kodu z
sekwencjami ucieczki składaj je programowo (`chr(92) + 'n'`) albo używaj edytora plików
zamiast heredoca.

## 57. Przeniesienie węzła to mutacja — z `MutationObserver` daje zawieszenie gry ❗✅

**Objaw:** wejście na ekran zawiesza całą grę (nie błąd, nie pusty ekran — zwis).

**Przyczyna:** dekorator wołany z `MutationObserver` przenosił cudzy element na koniec
wiersza **przy każdym przebiegu**:

```js
row.appendChild(leader);        // bezwarunkowo
```

`appendChild` na węźle, który już tam jest, **nie jest operacją pustą** — to usunięcie
i wstawienie, czyli mutacja `childList`. Obserwator ją widzi, woła dekorator, ten znowu
przenosi. Pętla bez końca w wątku UI.

**Naprawa — dwie warstwy:**

```js
if (row.lastElementChild !== leader) {   // 1. nie ruszaj, gdy już jest na miejscu
    row.appendChild(leader);
}
```

```js
let decorating = false;                  // 2. zabezpieczenie na wypadek przeoczenia
function decorateAll() {
    if (decorating) return;
    decorating = true;
    try { /* … */ } finally { decorating = false; }
}
```

**Zasada:** każda funkcja wołana z obserwatora DOM musi być *idempotentna względem DOM* —
przy drugim wywołaniu na tym samym stanie nie może niczego zmienić. Dotyczy to także
`classList.add` i `setAttribute`, jeśli obserwator patrzy na atrybuty (przy `childList:
true` — nie dotyczy).

Ten sam wzorzec działa poprawnie w `settlement-controls.js` (flaga `injecting`) — nowy kod
po prostu go nie powtórzył.

## 58. Wycinanie kodu zakresem „od funkcji do funkcji" ❗✅

Przepisując jedną funkcję skryptem, łatwo wyciąć wszystko między nią a następnym
punktem odniesienia:

```python
start = s.index('function decorate(card) {')
end   = s.index('let decorating = false;')     # a między nimi leżały jeszcze 4 funkcje
s = s[:start] + new + s[end:]
```

Zniknęły `updateMeasuredLayout`, `scheduleRemeasure` i licznik prób. **Plik nadal się
parsował** — brakujące funkcje to błąd wykonania, nie składni — więc `node --check` i
`deploy.sh` przepuściły go bez słowa. W grze: `ReferenceError` przy pierwszym użyciu i cały
mod bez efektu.

**Zabezpieczenie:** po każdym cięciu skryptem policz, co zostało:

```bash
grep -n "^function \|^let \|^const [a-z]\|^export function" plik.js
```

albo prostszy skan: zbierz nazwy zadeklarowane w pliku plus zaimportowane i sprawdź, czy
każde wywołanie ma pokrycie. To ta sama kontrola, która przy przebudowie struktury
(quirk #56) wyłapała `settlementYieldTotal` i `modifierApplies`.

⚠️ Kontrola składni **nie zastępuje** tego sprawdzenia. Wykrywa zepsuty plik, nie zepsuty
moduł.
