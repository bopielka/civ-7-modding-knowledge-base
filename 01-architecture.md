# 01 — Architektura gry i systemu modów

## ❗ Gdzie gra szuka modów użytkownika ✅ (potwierdzone empirycznie)

```
C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\
```

> ⚠️ **KOREKTA (2026-08-08).** Wcześniej w tej bazie (i w dokumentacji społeczności)
> podawano `Documents\My Games\Sid Meier's Civilization VII\Mods\`. **To jest błędne** —
> to konwencja z Civ VI. Mod umieszczony tam **nie zostaje w ogóle wykryty**.
>
> Dowód z `Logs\Modding.log`:
> ```
> Discovering new mods...
> C:/Users/najan/AppData/Local/Firaxis Games/Sid Meier's Civilization VII/Mods/
> Discovered 0 mods.
> ```
> Gra wypisuje **dokładnie jedną** ścieżkę skanowania i jest to ścieżka w `AppData\Local`.
> W `Documents\My Games\...` są wyłącznie `Saves\`.

Jak zweryfikować samodzielnie po aktualizacji gry:
```bash
grep -A2 "Discovering new mods" "$LOCALAPPDATA/Firaxis Games/Sid Meier's Civilization VII/Logs/Modding.log"
```

## Układ instalacji

```
Sid Meier's Civilization VII\
├── Base\
│   ├── Assets\schema\        ← schematy SQL (definicje wszystkich baz)
│   └── modules\
│       ├── core\             (453 MB) framework UI, fonty, ikony, vendor
│       ├── base-standard\    (280 MB) reguły uniwersalne, UI rozgrywki
│       ├── age-antiquity\    (186 MB)
│       ├── age-exploration\  (200 MB)
│       └── age-modern\       (171 MB)
└── DLC\<nazwa>\modules\      36 katalogów DLC
```

✅ Każdy moduł ma tę samą budowę co mod: plik `.modinfo` + podfoldery
(`data`, `text`, `l10n`, `ui`, `ui-next`, `config`, `scripts`, `maps`, `movies`).
**To znaczy, że moduły gry są najlepszą dokumentacją modowania, jaka istnieje.**

## Bazy danych gry

W `Base\Assets\schema\` jest kilka **oddzielnych** baz — to ważne, bo mod musi
trafić do właściwej:

| Schemat | Do czego |
|---|---|
| `gameplay/01_GameplaySchema.sql` | rozgrywka — 493 tabele (jednostki, budynki, cywilizacje, modyfikatory) |
| `frontend/schema-frontend-*.sql` | menu główne, ustawienia gry, keybindingi, hall of fame |
| `modding/schema-modding-10.sql` | sam system modów (mody, akcje, kryteria) |
| `localization/schema-loc-*.sql` | teksty i języki |
| `icons/IconManager.sql` | ikony |
| `colors/ColorManager.sql` | kolory |

## Anatomia pliku `.modinfo`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Mod id="moj-mod" version="1" xmlns="ModInfo">
    <Properties>
        <Name>LOC_MOD_MOJ_NAME</Name>          <!-- może być klucz LOC_ -->
        <Description>LOC_MOD_MOJ_DESC</Description>
        <Authors>Twoje imię</Authors>
        <Package>Mod</Package>
        <AffectsSavedGames>0</AffectsSavedGames>
        <ShowInBrowser>1</ShowInBrowser>
    </Properties>
    <Dependencies>   <!-- twarde zależności: bez nich mod się nie załaduje -->
        <Mod id="base-standard" title="LOC_MODULE_BASE_STANDARD_NAME" />
    </Dependencies>
    <References>     <!-- miękkie: wpływa na kolejność, nie wymaga obecności -->
        <Mod id="inny-mod" title="LOC_INNY_NAME" />
    </References>
    <ActionCriteria>
        <Criteria id="always"><AlwaysMet/></Criteria>
    </ActionCriteria>
    <ActionGroups>
        <ActionGroup id="moj-mod-game" scope="game" criteria="always">
            <Properties><LoadOrder>1000</LoadOrder></Properties>
            <Actions>
                <UpdateDatabase><Item>data/moje.xml</Item></UpdateDatabase>
            </Actions>
        </ActionGroup>
    </ActionGroups>
    <LocalizedText>   <!-- teksty samego modinfo (nazwa moda w menu) -->
        <File>text/en_us/ModInfoText.xml</File>
    </LocalizedText>
</Mod>
```

## Scope — gdzie akcja działa ✅

Zweryfikowane na 49 modach: używane są dokładnie **dwie** wartości.

| Scope | Kiedy aktywny | Typowe użycie |
|---|---|---|
| `game` | w trakcie rozgrywki | 66 wystąpień — reguły, UI gry, dane |
| `shell` | menu główne / ekran wyboru gry | 23 wystąpienia — konfiguracja, podgląd cywilizacji, opcje |

⚠️ Mod dodający cywilizację potrzebuje **obu**: `shell` żeby cywilizacja pojawiła się
na ekranie wyboru, `game` żeby faktycznie działała w rozgrywce. Widać to wprost
w modzie Polska — te same `ImportFiles`/`UpdateText` powtórzone w dwóch grupach.

## Typy akcji ✅

Zliczone w plikach gry bazowej + DLC (pierwsza liczba) i w modach Workshop (druga):

| Akcja | Gra/DLC | Mody | Do czego |
|---|---|---|---|
| `UpdateDatabase` | 330 | 18 | ładuje XML **lub SQL** do bazy gameplay/shell |
| `UpdateText` | 120 | 52 | teksty i tłumaczenia |
| `UpdateArt` | 113 | 0 | definicje artu (gra używa, mody prawie nie) |
| `UpdateIcons` | 89 | 14 | definicje ikon |
| `UpdateColors` | 33 | 0 | palety kolorów |
| `UIScripts` | 9 | 63 | **dodaje** skrypty JS do UI — główne narzędzie modów |
| `ImportFiles` | 5 | 27 | wrzuca pliki do wirtualnego FS; **nadpisuje pliki gry po ścieżce** |
| `ReplaceUIScript` | 0 | 3 | podmienia konkretny skrypt UI |
| `UpdateVisualRemaps` | 2 | 2 | mapuje nowe typy na istniejące modele 3D |
| `UIShortcuts` | 4 | 0 | skróty klawiszowe |
| `UIAudioRules` | 2 | 0 | reguły audio |
| `LoadOrder` | 2 | 59 | (właściwość grupy, nie akcja) kolejność ładowania |

## Kryteria (`ActionCriteria`) ✅

Używane w modach: `AlwaysMet` (46), `AgeInUse` (23), `ModInUse` (6).
Dostępne też w grze bazowej: `AgeAtOrBefore`, `ModIsEnabled`, `RuleSetInUse`,
`GameModeInUse`, `ConfigurationValueMatches`, `ConfigurationValueContains`.

```xml
<Criteria id="antiquity-only"><AgeInUse>AGE_ANTIQUITY</AgeInUse></Criteria>
<Criteria id="wiele" any="true">   <!-- any="true" = OR zamiast AND -->
    <AgeInUse>AGE_MODERN</AgeInUse>
    <AgeInUse>AGE_EXPLORATION</AgeInUse>
</Criteria>
```
Atrybut `any="true"` odpowiada kolumnie `Criteria.Any` w schemacie moddingu ✅.

## LoadOrder — kolejność ładowania ✅

Właściwość `ActionGroup`, nie moda. Wyższa liczba = ładowane później = **wygrywa**
przy konflikcie. Realne wartości z Workshop: od `1` do `130000`, z wyraźnymi
skupiskami na `1000` (12 modów), `10000`, `9999`, `100`.

Praktyka:
- **domyślnie nic nie ustawiaj** — mody bez `LoadOrder` ładują się w kolejności bazowej
- `1000` to de facto konwencja społeczności dla „zwykłego" moda UI
- bardzo wysokie wartości (99999+) to mody, które celowo chcą nadpisać wszystko inne
- ⚠️ dla dekoratorów UI kolejność bywa krytyczna — `Controls.decorate` **nie zadziała
  na już utworzone instancje komponentów** (patrz [05-ui-javascript.md](05-ui-javascript.md))

## Pipeline ładowania (model mentalny) ⚠️

Zrekonstruowany ze schematu moddingu — nie z dokumentacji:

1. Gra skanuje foldery modów → tabela `ScannedFiles`
2. Parsuje `.modinfo` → `Mods`, `ModProperties`, `ModRelationships`
3. Wylicza kryteria (`Criteria`, `Criterion`) dla bieżącego kontekstu (epoka, scope)
4. Dla spełnionych kryteriów kolejkuje `ActionGroups` posortowane po `LoadOrder`
5. Wykonuje `Actions` grupy — każda dokłada wiersze do odpowiedniej bazy
   albo rejestruje pliki w wirtualnym systemie plików

Wynika z tego ważna rzecz: **mody nie nadpisują plików gry na dysku** — wszystko dzieje
się w warstwie bazy danych i wirtualnego FS przy starcie.
