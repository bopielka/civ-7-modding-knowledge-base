# 17 — Rzeczy zaawansowane i nieudokumentowane

Rzeczy, których nie znajdziesz w żadnym poradniku, bo wynikają z czytania plików gry.

## 1. Własne właściwości w `.modinfo` ✅

Mod `bz-map-trix` deklaruje niestandardowe tagi:
```xml
<Properties>
    <bzIcon>blp:ntf_choosenarrative_blk</bzIcon>
    <bzIconGlow>#e5d2ac</bzIconGlow>
    <bzIconScale>...</bzIconScale>
    <bzIconCrop>...</bzIconCrop>
    <Teaser>...</Teaser>
    <LastUpdated>...</LastUpdated>
</Properties>
```
Działa, bo schemat moddingu przechowuje właściwości jako **pary nazwa/wartość**
(`ModProperties(ModRowId, Name, Value)`) — parser nie waliduje listy dozwolonych nazw ✅.

Konsekwencja: **mod może czytać własne metadane innych modów** i budować na tym
ekosystem (bz-map-trix ma własny system ikon dla rodziny modów `bz-`).

## 2. `ModProperties` jako kanał komunikacji między modami ⚠️

Skoro właściwości trafiają do bazy moddingu, mod A może odpytać o właściwości moda B.
Nie zweryfikowałem API dostępu z poziomu JS, ale mechanizm istnieje w schemacie.

## 3. Mod może tworzyć własne tabele ✅

```sql
CREATE TABLE IF NOT EXISTS CivsWithoutBackgrounds(
    CivilizationType TEXT PRIMARY KEY, ArtPath TEXT NOT NULL);
INSERT OR REPLACE INTO CivsWithoutBackgrounds VALUES('CIVILIZATION_POLAND','fs://...');
```
Mod Polska tak robi — prawdopodobnie po to, by inny mod (`custom-civ-art-fixes`?)
mógł to odczytać. To de facto **protokół międzymodowy**.

## 4. `ActionGroupRelationships` ⚠️

Schemat moddingu ma tabelę wiążącą grupy akcji różnych modów:
```sql
ActionGroupRelationships(ActionGroupRowId, OtherModId, OtherActionGroupId, Relationship)
```
Sugeruje to możliwość deklarowania relacji na poziomie **pojedynczej grupy akcji**,
a nie całego moda. Nie znalazłem składni XML, która to wypełnia — ❓ do zbadania.

## 5. `ModCompatibilityWhitelist` ✅ (schemat)

```sql
ModCompatibilityWhitelist(ModRowId, GameVersion)
```
Gra śledzi mody, które mają pomijać ostrzeżenia o niezgodności z wersją.
❓ Nie wiem, czy da się to ustawić z `.modinfo`.

## 6. `EpicMods` ✅ (schemat)

Osobna tabela na mody z wersji Epic Games Store — istotne tylko, jeśli mod ma działać
na obu platformach.

## 7. `Migrations` ✅ (schemat)

```sql
Migrations(SQL, MinVersion, MaxVersion, SortIndex)
```
System migracji bazy przy aktualizacji gry — mechanizm Firaxis, ale pokazuje, że
baza moddingu jest wersjonowana (`PRAGMA user_version(5)`).

## 8. Kryteria konfiguracyjne ✅ (w grze bazowej)

Poza `AgeInUse`/`ModInUse` istnieją:
```xml
<ConfigurationValueMatches>
    <ConfigurationId>...</ConfigurationId>
    <Group>...</Group>
    <Value>...</Value>
</ConfigurationValueMatches>
<ConfigurationValueContains>...</ConfigurationValueContains>
<RuleSetInUse>...</RuleSetInUse>
<GameModeInUse>...</GameModeInUse>
<AgeAtOrBefore>AGE_MODERN</AgeAtOrBefore>
<ModIsEnabled>...</ModIsEnabled>
```
Żaden z 49 modów Workshop ich nie użył — pole do wykorzystania, np. mod aktywny tylko
przy konkretnym ustawieniu rozgrywki.

## 9. Kryterium odwrotne ⚠️

Schemat: `Criterion(CriterionRowId, CriteriaRowId, CriterionType, Inverse)`.
Kolumna `Inverse` sugeruje możliwość negacji kryterium (np. „gdy mod NIE jest obecny").
❓ Nie znalazłem atrybutu XML, który to ustawia — prawdopodobnie `inverse="true"`
na elemencie kryterium. **Warto przetestować.**

## 10. `blp:` — wbudowane zasoby gry ✅

```sql
('UNIT_POLAND_HUSSAR','PORTRAIT_MASK','blp:unitflag_hussar',0)
```
```xml
<bzIcon>blp:ntf_choosenarrative_blk</bzIcon>
```
Pozwala użyć grafiki gry bez dołączania własnej. Nazwy trzeba wyłuskać z plików gry:
```bash
grep -rho "blp:[a-z0-9_]*" "$G/Base/modules/"*/data/ | sort -u | head -50
```

## 11. `InsertOrIgnore` na `Kinds` ✅ — wzorzec Firaxis

```xml
<Kinds>
    <InsertOrIgnore Kind="KIND_TRAIT"/>
    <InsertOrIgnore Kind="KIND_VICTORY"/>
</Kinds>
```
Deklaruje potrzebne `Kind` bez wysypania się, jeśli już istnieją. **Kopiuj ten wzorzec**
— czyni moda odpornym na kolejność ładowania.

## 12. `InheritFrom` w `Leaders` ✅

`InheritFrom="LEADER_DEFAULT"` pozwala zdefiniować lidera **czterema kolumnami**
zamiast jedenastoma. Klucz obcy wskazuje na `Leaders` — czyli można też dziedziczyć
po dowolnym istniejącym liderze.
⚠️ Sprawdź, czy `Civilizations` ma analogiczny mechanizm — nie ma kolumny `InheritFrom`,
więc raczej nie.

## 13. `Modifiers` — flagi kontroli kumulacji ✅

```xml
<Modifier id="..." collection="..." effect="..."
          permanent="true" run-once="true" new-only="true"
          owner-stack-limit="1" subject-stack-limit="1">
```
`owner-stack-limit` / `subject-stack-limit` zapobiegają wielokrotnemu nakładaniu się
tego samego bonusu — rzadko używane przez modderów, a rozwiązują realne bugi balansu.

## 14. `Requirements` ma kolumny AI ✅

```
AiWeighting, BehaviorTree, Impact, Likeliness, ProgressWeight, Persistent, Triggered, Reverse
```
Poza logiką warunku, wymaganie niesie **wskazówki dla AI**, jak ważny jest dany warunek.
Modderzy zwykle to pomijają — a to wpływa na zachowanie komputera.

## 15. Warstwa `ui-next` jest w trakcie migracji ⚠️

Proporcje (core: 373 stare vs 186 nowe; base-standard: 601 vs 132) sugerują, że Firaxis
stopniowo przepisuje UI na Solid.js. **Ryzyko dla modów**: komponent, który dziś
dekorujesz w `ui/`, może w kolejnym patchu przenieść się do `ui-next/`.
Mod `bz-map-trix` asekuruje się, mając pliki w obu drzewach.

## Lista rzeczy do przetestowania

- [ ] `inverse="true"` na kryterium
- [ ] `VisualRemaps` z `Kind=LEADER`
- [ ] czy plik zasobu bez rozszerzenia w `ImportFiles` jest wymagany
- [ ] czy alias `fs://game/` może być inny niż `<Mod id>`
- [ ] `ActionGroupRelationships` — składnia XML
- [ ] odczyt `ModProperties` innego moda z poziomu JS
- [ ] czy powrót do menu głównego przeładowuje mody bez restartu gry
      (sugerują to wpisy `Reason: Main Menu Reset` w `Modding.log`)
- [ ] `ACTION_GROUP_BUNDLE` z `civ7-modding-tools` — jaki `.modinfo` faktycznie generuje

## Strony dokumentacji społeczności jeszcze nieprzejrzane

Z `civ7community.mintlify.app` przejrzałem: documentation-guide, modding-architecture,
mod-patterns, typescript-overview, typescript-technical, environment-setup,
howto/creating-civilizations, howto/advanced-techniques, general-creating-leaders,
reference/modding-reference, reference/gameplay-mechanics, ages/age-gameplay-mechanics.

Zostały (warte zajrzenia przy konkretnym zadaniu):
- `guides/getting-started`, `guides/general-modifying-content`
- `guides/database-schemas`, `guides/base-standard-module`
- `guides/ages/age-modules`, `guides/ages/age-architecture`
- `guides/general-creating-civilizations`
- `guides/typescript/howto/`: creating-units, creating-buildings, leaders-and-ages,
  unique-quarters, traditions, progression-trees, modifiers-and-effects, assets-and-icons
- `guides/examples/dacia-*` (4 strony — pełny przykład implementacji cywilizacji)
- `reference/file-paths-reference`, `reference/modding-guide-civs-leaders`

⚠️ Przy każdej z nich pamiętaj o zasadzie z [22-source-evaluation.md](22-source-evaluation.md):
identyfikatory weryfikuj w schemacie gry.


---

## `WorldUI` — rysowanie po MAPIE 3D (nie po DOM) ✅

Odkryte przy diagnozie moda Holistic QoL+. To osobny świat od DOM-u i osobne źródło
„artefaktów na ekranie", bo nic z tego nie jest elementem HTML.

```js
const group = WorldUI.createOverlayGroup(NAZWA, OVERLAY_PRIORITY.PLOT_HIGHLIGHT, {x:1,y:1,z:1});
const plots   = group.addPlotOverlay();      // wypełnienie heksów
const borders = group.addBorderOverlay({ style: 'MovementRange', primaryColor, secondaryColor });
const marks   = group.addLandmarkOverlay();  // znaczniki na konstrukcjach

plots.addPlots([plotIndex], { fillColor });
borders.setPlotGroups(plotIndexes, 1);
borders.setGroupStyle(1, { style: 'CombatBorder', primaryColor, secondaryColor });

// billboardy z tekstem/ikoną nad heksem
const sprites = WorldUI.createSpriteGrid(NAZWA, SpriteMode.Billboard);

group.setVisible(true);
plots.clear(); borders.clear(); sprites.clear();
```

Style obramowań widziane w kodzie: `MovementRange`, `CombatBorder`.
Priorytety: `OVERLAY_PRIORITY.PLOT_HIGHLIGHT`, `OVERLAY_PRIORITY.UNIT_COMBAT`.

⚠️ **Nic tego nie sprząta za ciebie.** Overlay przeżywa zmianę ekranu, tury i trybu
interfejsu — dopóki ktoś nie zawoła `clear()`. Typowy błąd: odświeżanie podpięte pod wąską
listę zdarzeń (`UnitSelectionChanged`, `UnitMoved`, `interface-mode-changed`), a stan
zmienia się zdarzeniem spoza listy → heksy albo etykiety zostają na mapie.

⚠️ **Kolejność `ensure` i sprawdzenia „czy włączone".** W Holistic QoL+
`ensureCityLandmarkMarkersOverlay()` leci PRZED `if (!enabled) return` i robi
`setVisible(true)`. Efekt: wyłączenie funkcji w opcjach nie usuwa grupy overlayów, tylko
zostawia pustą i widoczną. Sprawdzenie ma być pierwsze.

## Animowany `box-shadow` / `filter` w CSS — kandydat na smużenie ❗

Ten sam mod animuje `box-shadow` w nieskończonej pętli (`animation: … 1.8s infinite`) oraz
`filter: brightness()`. Poświata wychodzi **poza pudełko elementu**, a ten renderer nie
zawsze unieważnia region rysowania na tyle szeroko — stąd smugi i duchy po elemencie.

Przy diagnozie artefaktów u gracza: to jest pierwsza rzecz do wyłączenia, bo jest tania
w sprawdzeniu (jeden przełącznik „ogranicz animacje", jeśli mod go ma) i nie wymaga
wyłączania całej funkcji.


---

## ⚠️ Powiadomienia rysuje ich DWÓCH, z dwóch różnych źródeł ❗✅

To najważniejsza rzecz do zapamiętania, bo pierwsze podejście „ukryj powiadomienie" nie
zrobiło nic widocznego.

| kto rysuje | skąd bierze | które powiadomienia |
|---|---|---|
| `panel-notification-train` (pasek) | `NotificationModel` | **NIE** blokujące turę (`isSoftNotification`) |
| `panel-action` — **ikonka w pierścieniu** | `Game.Notifications.getIdsForPlayer` → `getNotificationInfo` | blokujące turę, **oprócz** aktualnego blockera |
| `panel-action` — **GŁÓWNY PRZYCISK** | `Game.Notifications.findEndTurnBlocking` | **aktualny blocker końca tury** |

⚠️ **Trzeci wiersz to pułapka, która kosztowała kilka rund.** Ten sam typ powiadomienia
wędruje między drugim a trzecim torem: dopóki blokuje coś innego, jest ikonką w pierścieniu;
gdy gracz zrobi wszystko inne, **awansuje na główny przycisk** — a tam id bierze się
z `findEndTurnBlocking`, więc **żaden filtr na `getNotificationInfo` go nie dosięgnie**.
Objaw: „ukryłem, a i tak się pokazuje, ale dopiero gdy nie mam nic innego do zrobienia".

Sprawdzenie, w którym torze się jest:

```js
const type = Game.Notifications.getEndTurnBlockingType(playerID);
const id   = Game.Notifications.findEndTurnBlocking(playerID, type);
const isThisOne = id && Game.Notifications.getType(id) === Game.getHash('NOTIFICATION_...');
```

Sprawdzenie, do której grupy należy typ: `Game.Notifications.getBlocksTurnAdvancement(id)`.

`NOTIFICATION_ASSIGN_NEW_RESOURCES` **blokuje**, więc rysuje go pierścień, a pasek go
ignoruje. Ukrycie go w `NotificationModel` (przez handler) usuwa go z listy, w której
i tak nigdy nie był — z ekranu nie znika nic.

### ⚠️ Kasowanie blockera NIE działa — silnik go odtwarza

`Game.Notifications.dismiss(id)` wykonuje się i jest przyjęte, ale powiadomienie wraca
w ciągu sekundy: jego wiersz w `notification.xml` ma **`AutoNotify="True"`**, więc silnik
podnosi je ponownie, dopóki trwa warunek. Dla `NOTIFICATION_ASSIGN_NEW_RESOURCES`
warunkiem jest „masz nieprzypisane zasoby" — czyli nie do usunięcia z poziomu UI.

### Jak naprawdę wyciszyć blockera końca tury ✅

Blokada jest **w UI, nie w silniku** — `canEndTurn()` to metoda panelu czytająca
`getEndTurnBlockingType`, a zakończenie tury to `GameContext.sendTurnComplete()`. Da się
więc podmienić odpowiedź na czas jednego wywołania:

```js
function withoutOurBlocker(body) {
    const real = Game.Notifications.getEndTurnBlockingType;
    try {
        Game.Notifications.getEndTurnBlockingType = () => EndTurnBlockingTypes.NONE;
        return body();
    } finally {
        Game.Notifications.getEndTurnBlockingType = real;
    }
}
```

⚠️ **Trzeba owinąć DWIE metody `PanelAction`, nie jedną.** Panel zadaje to samo pytanie
dwa razy, w dwóch celach:

| metoda | za co odpowiada |
|---|---|
| `refreshActionButton` | jak przycisk **wygląda** |
| `tryEndTurn` | co kliknięcie **robi** — `canEndTurn` czyta blocker ponownie i przy trafieniu woła `activateBlockingNotification()` zamiast zakończyć turę |

Owinięcie samego pierwszego daje przycisk z napisem „Zakończ turę", który po kliknięciu
otwiera ekran blockera. Objaw mylący, bo wygląda jak błąd rysowania.

⚠️ `Game.Notifications` to obiekt silnika — sprawdź po podstawieniu, czy naprawdę zwraca
podmienioną wartość, i wycofaj się, jeśli nie.

Nie rysować blockera to najgorsze z możliwych wyjść: blokada zostaje (jest po stronie
silnika), a znika jej wyjaśnienie — gracz klika „Zakończ turę" i nic się nie dzieje.

Właściwe narzędzie to `Game.Notifications.dismiss(id)`. To **akcja dostępna graczowi** —
gra woła dokładnie to samo, gdy odrzucisz powiadomienie ręcznie (`panel-notification-train`)
— więc zdejmuje blokadę, zamiast ją maskować.

```js
const ids = Game.Notifications.getIdsForPlayer(playerID)
    .filter((id) => Game.Notifications.getType(id) === Game.getHash(TYP));
ids.forEach((id) => Game.Notifications.dismiss(id));
```

⚠️ Kasować **tylko** wtedy, gdy z powiadomieniem naprawdę nie da się nic zrobić. Skasowanie
takiego, na które gracz mógłby zareagować, zabiera mu informację.

### Jak ukryć ikonę z pierścienia

`panel-action` to **stary framework** (`Controls.define`), a jego klasa jest eksportowana,
więc da się owinąć prototyp. Punkt zaczepienia to `getNotificationInfo`: `refreshActionButton`
mapuje przez nią wszystkie id i **odrzuca `null`**.

```js
import { PanelAction } from '/base-standard/ui/action/panel-action.js';

const hidden = Game.getHash('NOTIFICATION_ASSIGN_NEW_RESOURCES');
const original = PanelAction.prototype.getNotificationInfo;
PanelAction.prototype.getNotificationInfo = function (id) {
    const info = original.call(this, id);
    return info?.type === hidden && niePotrzebne() ? null : info;
};
```

Panel odświeża się sam na `NotificationAdded`, `NotificationUpdated`, `LocalPlayerTurnBegin`
i zdarzeniach jednostek — nie trzeba dokładać własnego nasłuchu, żeby ikona wróciła.

⚠️ Ukrycie ikony **nie zdejmuje blokady tury** — o tym decyduje silnik, nie to, czy coś
jest narysowane.

## Ukrywanie powiadomienia z PASKA (centrum powiadomień) ✅

Powiadomienia mają **jeden handler na typ**, a mod może go podmienić — `registerHandler`
zwyczajnie nadpisuje wpis w mapie:

```js
import { NotificationHandlers } from '/base-standard/ui/notification-train/notification-handlers.js';
import { NotificationModel }    from '/base-standard/ui/notification-train/model-notification-train.js';

NotificationModel.manager.registerHandler('NOTIFICATION_ASSIGN_NEW_RESOURCES',
    new (class extends NotificationHandlers.AssignNewResources {
        add(id) {
            if (nieChcemyPokazywac()) return false;   // nie rejestruje = nie widać
            return super.add(id);
        }
    })());
```

**Dlaczego `add`:** `onNotificationAdded` woła `handler.add(id)`, a domyślne `add` dopiero
robi `NotificationModel.manager.add(...)` — i to ono wrzuca rzecz na ekran. Handler, który
tego nie zrobi, zostawia powiadomienie **niezarejestrowane**: dalej jest w
`Game.Notifications`, ale UI go nie rysuje.

⚠️ **To nie jest odrzucenie powiadomienia.** Nic nie jest dismissed ani skasowane — stan
gry zostaje nietknięty (ważne dla modów UI-only), a powiadomienie może się później pokazać
bez potrzeby ponownego wywołania przez grę.

**Żeby wróciło**, gdy warunek przestanie obowiązywać, wołamy `NotificationModel.manager.rebuild()`
— przechodzi po wszystkich żywych powiadomieniach i podaje je handlerom jeszcze raz
(`rebuild` → `handler.add(id)` w pętli).

⚠️ Dziedziczymy po **konkretnym** handlerze gry (`NotificationHandlers.AssignNewResources`),
nie po `DefaultHandler` — inaczej gubimy to, co się dzieje po kliknięciu powiadomienia
(tutaj: otwarcie ekranu handlu).

Definicje typów: `base-standard/data/notification.xml`. Warto zajrzeć w `SeverityType`
i `ExpiresEndOfTurn` — `HIGH` + `False` znaczy „będzie wisiało w rogu w kółko".
