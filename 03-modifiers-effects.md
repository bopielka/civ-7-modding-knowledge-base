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
