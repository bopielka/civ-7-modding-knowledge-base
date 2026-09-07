# 03 — Modyfikatory, efekty, wymagania

To jest serce mechaniki Civ VII. Prawie każda unikalna zdolność (cywilizacji, lidera,
budynku, tradycji) to **modyfikator**.

## Model mentalny

```
Modifier = KTO (collection) + CO ROBI (effect) + POD JAKIM WARUNKIEM (requirements) + ILE (arguments)
```

- **collection** — zbiór obiektów, na które efekt działa (miasta gracza, jednostki, …)
- **effect** — konkretna operacja (dodaj yield, zwiększ siłę, …)
- **requirements** — filtr; osobno dla *właściciela* i dla *podmiotu*
- **arguments** — parametry liczbowe/typowe efektu

## Składnia `<GameEffects>` ✅

**To jest współczesny sposób w Civ VII** (8640 użyć w grze bazowej). Uwaga: atrybuty
są **małymi literami** — inaczej niż w tabelach bazy.

```xml
<?xml version="1.0" encoding="utf-8"?>
<GameEffects xmlns="GameEffects">
    <Modifier id="MOD_CIV_WONDER_PRODUCTION_AKSUM"
              collection="COLLECTION_PLAYER_CITIES"
              effect="EFFECT_CITY_ADJUST_FAVORED_WONDER_PRODUCTION">
        <SubjectRequirements>
            <Requirement type="REQUIREMENT_PLAYER_HAS_CIVILIZATION_OR_LEADER_TRAIT">
                <Argument name="TraitType">TRAIT_AKSUM</Argument>
            </Requirement>
        </SubjectRequirements>
        <Argument name="Percent">30</Argument>
        <String context="Preview">LOC_MOD_CIV_WONDER_PRODUCTION_AKSUM_DESCRIPTION</String>
    </Modifier>
</GameEffects>
```

### Atrybuty `<Modifier>` ✅ (pełna lista z danych gry)

| Atrybut | Znaczenie |
|---|---|
| `id` | unikalny identyfikator (wymagany) |
| `collection` | `COLLECTION_*` — na kim działa |
| `effect` | `EFFECT_*` — co robi |
| `permanent` | efekt zostaje po ustaniu warunku |
| `run-once` | wykonaj tylko raz |
| `new-only` | dotyczy tylko nowo tworzonych obiektów |
| `owner-stack-limit` | limit kumulacji po właścicielu |
| `subject-stack-limit` | limit kumulacji po podmiocie |

### Elementy wewnętrzne

| Element | Do czego |
|---|---|
| `<Argument name="...">wartość</Argument>` | parametr efektu |
| `<SubjectRequirements>` | warunki na obiekcie docelowym (1689 użyć) |
| `<OwnerRequirements>` | warunki na właścicielu modyfikatora (1373 użycia) |
| `<String context="Preview">` | tekst opisu w UI (klucz `LOC_*`) |

## Kolekcje — pełna lista (38) ✅

**Gracz / globalne**
`COLLECTION_OWNER`, `COLLECTION_ALL_PLAYERS`, `COLLECTION_MAJOR_PLAYERS`,
`COLLECTION_INDEPENDENT_PLAYERS`

**Miasta**
`COLLECTION_PLAYER_CITIES`, `COLLECTION_ALL_CITIES`, `COLLECTION_OWNER_CITY`,
`COLLECTION_PLAYER_CAPITAL_CITY`, `COLLECTION_ALL_CAPITAL_CITIES`,
`COLLECTION_CITIES_FOLLOWING_OWNER_RELIGION`, `COLLECTION_PLAYER_INFECTED_CITIES`,
`COLLECTION_CITY_TRAINED_UNITS`, `COLLECTION_TRADE_ROUTE_TARGET_CITY`

**Dzielnice / budowle**
`COLLECTION_CITY_DISTRICTS`, `COLLECTION_PLAYER_DISTRICTS`, `COLLECTION_ALL_DISTRICTS`,
`COLLECTION_PLAYER_CAPITAL_CITY_DISTRICTS`, `COLLECTION_PLAYER_CONSTRUCTIBLES`

**Jednostki i walka**
`COLLECTION_PLAYER_UNITS`, `COLLECTION_ALL_UNITS`, `COLLECTION_PLAYER_COMBAT`,
`COLLECTION_UNIT_COMBAT`, `COLLECTION_OWNER_COMMANDER_HIGHEST_LEVEL`,
`COLLECTION_UNIT_NEAREST_OWNER_CITY`, `COLLECTION_UNIT_OCCUPIED_CITY`,
`COLLECTION_UNIT_OCCUPIED_DISTRICT`

**Pola / yieldy**
`COLLECTION_ALL_PLOT_YIELDS`, `COLLECTION_CITY_PLOT_YIELDS`,
`COLLECTION_PLAYER_PLOT_YIELDS`, `COLLECTION_SINGLE_PLOT_YIELDS`

**Handel / narracja**
`COLLECTION_PLAYER_TRADE_ROUTES`, `COLLECTION_NARRATIVE_STORY`,
`COLLECTION_ANY_CITY_AT_STORY`, `COLLECTION_OWNER_CITY_NEAREST_STORY`,
`COLLECTION_OWNER_UNIT_NEAREST_STORY`, `COLLECTION_OWNER_COMMANDER_NEAREST_STORY`,
`COLLECTION_INDEPENDENT_NEAREST_STORY`, `COLLECTION_HOMELANDS_INDEPENDENT_NEAREST_STORY`

## Efekty (387) ✅

Pełna lista w [18-reference-enumerations.md](18-reference-enumerations.md).
Konwencja nazewnicza czytelna i przewidywalna:

```
EFFECT_<CEL>_<CZASOWNIK>_<PRZEDMIOT>
EFFECT_CITY_ADJUST_YIELD              — zmień yield miasta
EFFECT_PLAYER_GRANT_YIELD             — przyznaj graczowi yield
EFFECT_ADJUST_UNIT_STRENGTH_MODIFIER  — zmień siłę jednostki
EFFECT_CITY_GRANT_UNIT                — daj miastu jednostkę
EFFECT_GRANT_WALLS                    — przyznaj mury
```

Czasowniki: `ADJUST` (modyfikuj wartość), `GRANT` (przyznaj), `ATTACH`, `PLACE`, `CHANGE`.

**Jak znaleźć właściwy efekt:** wyszukaj w plikach gry mechanikę podobną do tej, którą
chcesz zrobić:
```bash
grep -rl "EFFECT_CITY_ADJUST_YIELD" "Base/modules/age-antiquity/data/"
```

## Wymagania (270 typów) ✅

```xml
<SubjectRequirements>
    <Requirement type="REQUIREMENT_PLAYER_HAS_CIVILIZATION_OR_LEADER_TRAIT">
        <Argument name="TraitType">TRAIT_POLAND</Argument>
    </Requirement>
</SubjectRequirements>
```

Najczęściej używany w praktyce: `REQUIREMENT_PLAYER_HAS_CIVILIZATION_OR_LEADER_TRAIT`
— to jest standardowy sposób „ten bonus dotyczy tylko mojej cywilizacji".

### Zestawy wymagań (wersja tabelowa) ✅

Gdy wymagania definiujesz w SQL, a nie inline w XML:

```sql
INSERT INTO RequirementSets(RequirementSetId,RequirementSetType)
VALUES('REQSET_POLAND_WAR_IN_AQ','REQUIREMENTSET_TEST_ALL');

INSERT INTO Requirements(RequirementId,RequirementType,ProgressWeight)
VALUES('REQ_POLAND_CIV_WAR_AQ','REQUIREMENT_PLAYER_CIVILIZATION_TYPE_MATCHES',1);

INSERT INTO RequirementArguments(RequirementId,Name,Value)
VALUES('REQ_POLAND_CIV_WAR_AQ','CivilizationType','CIVILIZATION_POLAND');

INSERT INTO RequirementSetRequirements(RequirementSetId,RequirementId)
VALUES('REQSET_POLAND_WAR_IN_AQ','REQ_POLAND_CIV_WAR_AQ');
```

Tylko dwa typy zestawów ✅: `REQUIREMENTSET_TEST_ALL` (AND), `REQUIREMENTSET_TEST_ANY` (OR).

## Podpinanie modyfikatora do obiektu ✅

Sam modyfikator nic nie robi — trzeba go **przypiąć**. Tabele wiążące:

| Tabela | Podpina do |
|---|---|
| `TraitModifiers` | cechy (czyli pośrednio: cywilizacji/lidera) |
| `TraditionModifiers` | tradycji |
| `ConstructibleModifiers` | budynku/ulepszenia |
| `UnitAbilityModifiers` | zdolności jednostki |
| `UnitPromotionModifiers` | awansu |
| `GovernmentModifiers` | ustroju |
| `BeliefModifiers` | wierzenia |
| `NarrativeRewards` (kolumna `ModifierID`) | nagrody narracyjnej |
| `GreatPersonIndividualActionModifiers` | akcji wielkiej postaci |
| `UniqueQuarterModifiers` | unikalnej dzielnicy |
| `MementoModifiers` | pamiątki |
| `LegacyModifiers`, `GoldenAgeModifiers`, `CityStateBonusModifiers` | j.w. |

```sql
-- „cecha Polski daje ten modyfikator"
INSERT INTO TraitModifiers(TraitType,ModifierId)
VALUES('TRAIT_POLAND_GOLDEN_LIBERTY','POLAND_ACTIVATE_MILITARY_FARM_AQ');
```

## Alternatywa bez modyfikatorów: tabele yieldów ✅

Do prostych bonusów yieldów **nie potrzebujesz modyfikatora** — są dedykowane tabele:

```sql
-- +1 żywności i +1 zadowolenia z każdej farmy w mieście (epoka starożytna)
INSERT INTO Warehouse_YieldChanges(ID,Age,YieldType,YieldChange,ConstructibleInCity) VALUES
('POLAND_GRANARY_I_FARM_FOOD','AGE_ANTIQUITY','YIELD_FOOD',1,'IMPROVEMENT_FARM'),
('POLAND_GRANARY_I_FARM_HAPPY','AGE_ANTIQUITY','YIELD_HAPPINESS',1,'IMPROVEMENT_FARM');

-- bonus za sąsiedztwo
INSERT INTO Adjacency_YieldChanges(ID,YieldType,YieldChange,TilesRequired,AdjacentConstructible)
VALUES('POLAND_MILITARY_FARM_AQ','YIELD_HAPPINESS',1,1,'IMPROVEMENT_FARM');
```
To jest prostsze i mniej podatne na błędy — używaj, gdy wystarcza.

## Skala tabel — ile to naprawdę wierszy ✅ (zmierzone 2026-08-25)

Policzone bezpośrednio w plikach gry (`Base/modules/**/*.xml`, wszystkie ery + `core` +
`base-standard`), przez zliczenie elementów w blokach `<GameEffects>`:

| element | sztuk w plikach |
|---|---|
| `<Modifier>` | 12 093 |
| `<Argument>` | 39 385 |
| `<Requirement>` | 15 147 |

Formy tabelowej (`<Modifiers><Row/>`) jest w porównaniu z tym śladowo mało: 86 wierszy
`Modifiers`, 230 `ModifierArguments`, 376 `Requirements`, 2 357 `TypeTags` w całej grze.
Czyli **prawie wszystkie modyfikatory gry są pisane składnią `<GameEffects>`** i dopiero
przy ładowaniu trafiają do tabel `Modifiers` / `ModifierArguments` / `Requirements`.

⚠️ **Konsekwencja dla modów UI:** w runtime `GameInfo` widzi `core` + `base-standard` +
**tylko bieżącą erę**, więc realnie to rząd kilku tysięcy modyfikatorów i kilkunastu tysięcy
argumentów — ale to nadal znaczy, że *każde* pytanie typu „który modyfikator dotyczy tego
zasobu" zadane przez skan `GameInfo.Modifiers` to pełny przemiał kilku tysięcy wierszy.
Indeksuj raz i **ogranicz indeks do tego, o co faktycznie pytasz** (np. tylko modyfikatory
przypięte do zasobów): trzymanie mapy dla wszystkich modyfikatorów to tysiące obiektów
`Map` żyjących przez całą sesję. Przykład takiego indeksu:
`mod-projects/better-commerce-screen-ui/ui/planner/effects.js`.

⚠️ **Ile to kosztuje pamięci** (zmierzone w Node na syntetycznych tabelach tej wielkości,
2026-08-25): pełny indeks `Modifiers` + `DynamicModifiers` + graf `Requirements` w mapach JS to
**~6,8 MB żywej sterty** trzymanej przez całą sesję. Indeks ograniczony do modyfikatorów zasobów
kosztuje w praktyce zero. Silnik gry to nie Node, więc traktuj to jako rząd wielkości, nie wyrocznię.

## ⛔ Czego z modyfikatora NIE da się odczytać: jego bieżącego wkładu w dochód

Sprawdzone 2026-09-03, przy próbie rozpisania wiersza „Inne" w dochodach osady na umiejętności
przywódcy. **Nie da się** — i warto wiedzieć dlaczego, bo pytanie wraca.

Przykład wzorcowy, umiejętność Tecumseha (`DLC/shawnee-tecumseh/modules/data/leaders-gameeffects.xml`):

```xml
<Modifier id="TECUMSEH_MOD_SUZERAIN_PRODUCTION"
          collection="COLLECTION_PLAYER_CITIES" effect="EFFECT_CITY_ADJUST_YIELD_PER_SUZERAIN">
    <SubjectRequirements><Requirement type="REQUIREMENT_CITY_IS_CITY"/></SubjectRequirements>
    <Argument name="YieldType">YIELD_PRODUCTION</Argument>
    <Argument name="Amount" type="ScaleByGameAge" extra="100">1</Argument>
    <Argument name="Tooltip">LOC_TRAIT_LEADER_TECUMSEH_ABILITY_NAME</Argument>
</Modifier>
```

W grze ten modyfikator dawał **+12** produkcji. W danych stoi **1**. Różnica to dwie osobne
mechaniki silnika, których żadna tabela nie wystawia:

- `type="ScaleByGameAge"` — mnożnik z numeru ery (era 2 → ×2);
- `_PER_SUZERAIN` w nazwie efektu — mnożnik z liczby miast-państw, których gracz jest suzerenem (×6).

Policzenie tego to **przepisanie reguł gry po naszej stronie**, osobno dla każdego wariantu efektu.
W plikach gry jest **12 093** definicji `<Modifier>`, więc nie jest to jednorazowy koszt.

⚠️ **Nie ma API, które by to obeszło.** Cała gra używa w runtime dokładnie dwóch metod
`GameEffects`: `getModifierDefinitionTextKey` i `getModifierDefinitionArgumentString` — obie
opisują **definicję**, żadna nie odpowiada „ile ten modyfikator daje TERAZ w TEJ osadzie".

⚠️ **Ani węzła dochodów.** Wszystkie 19 węzłów `CityYieldNodes` (patrz `28-city-screen.md`) nie
zawiera żadnego dla cech przywódcy ani cywilizacji. Dlatego „Inne" jest **resztą arytmetyczną**
(`suma węzła INCOME − suma nazwanych dzieci`), a nie pozycją, którą silnik czymkolwiek podpisał:
nie ma pod nią żadnych `steps` do rozwinięcia.

✅ **Co JEST czytelne maszynowo: nazwa.** 1 043 z 12 093 modyfikatorów niesie
`<Argument name="Tooltip">` z kluczem lokalizacyjnym własnej zdolności. Ścieżka
`Players → leaderType → GameInfo.LeaderTraits → GameInfo.TraitModifiers → GameInfo.Modifiers`
pozwala więc **nazwać kandydatów** do „Innego" bez wpisywania czegokolwiek na sztywno — ale nie
przypisać im kwot. Nazwy bez liczb są uczciwe; liczby byłyby zgadywaniem.

⚠️ Odsyłacz do przestrogi, która już tu jest: `modelTownFocus` w City Hall ma premie wpisane na
sztywno i po rebalansie gry ten panel po cichu kłamie. Statyczna mapa premii przywódców to ten
sam wzorzec, tylko o dwa rzędy wielkości większy.
