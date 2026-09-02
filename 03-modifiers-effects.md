# 03 — Modifiers, effects, requirements

This is the heart of Civ VII's mechanics. Nearly every unique ability (of a civilization, leader,
building, tradition) is a **modifier**.

## Mental model

```
Modifier = WHO (collection) + WHAT IT DOES (effect) + UNDER WHAT CONDITION (requirements) + HOW MUCH (arguments)
```

- **collection** — the set of objects the effect applies to (the player's cities, units, …)
- **effect** — the concrete operation (add a yield, increase strength, …)
- **requirements** — a filter; separate ones for the *owner* and for the *subject*
- **arguments** — the effect's numeric/type parameters

## The `<GameEffects>` syntax ✅

**This is the modern way in Civ VII** (8640 uses in the base game). Note: the attributes
are **lowercase** — unlike in the database tables.

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

### `<Modifier>` attributes ✅ (full list from the game's data)

| Attribute | Meaning |
|---|---|
| `id` | unique identifier (required) |
| `collection` | `COLLECTION_*` — who it applies to |
| `effect` | `EFFECT_*` — what it does |
| `permanent` | the effect persists after the condition ends |
| `run-once` | execute only once |
| `new-only` | applies only to newly created objects |
| `owner-stack-limit` | stacking limit per owner |
| `subject-stack-limit` | stacking limit per subject |

### Inner elements

| Element | What for |
|---|---|
| `<Argument name="...">value</Argument>` | an effect parameter |
| `<SubjectRequirements>` | conditions on the target object (1689 uses) |
| `<OwnerRequirements>` | conditions on the modifier's owner (1373 uses) |
| `<String context="Preview">` | the description text in the UI (a `LOC_*` key) |

## Collections — the full list (38) ✅

**Player / global**
`COLLECTION_OWNER`, `COLLECTION_ALL_PLAYERS`, `COLLECTION_MAJOR_PLAYERS`,
`COLLECTION_INDEPENDENT_PLAYERS`

**Cities**
`COLLECTION_PLAYER_CITIES`, `COLLECTION_ALL_CITIES`, `COLLECTION_OWNER_CITY`,
`COLLECTION_PLAYER_CAPITAL_CITY`, `COLLECTION_ALL_CAPITAL_CITIES`,
`COLLECTION_CITIES_FOLLOWING_OWNER_RELIGION`, `COLLECTION_PLAYER_INFECTED_CITIES`,
`COLLECTION_CITY_TRAINED_UNITS`, `COLLECTION_TRADE_ROUTE_TARGET_CITY`

**Districts / constructibles**
`COLLECTION_CITY_DISTRICTS`, `COLLECTION_PLAYER_DISTRICTS`, `COLLECTION_ALL_DISTRICTS`,
`COLLECTION_PLAYER_CAPITAL_CITY_DISTRICTS`, `COLLECTION_PLAYER_CONSTRUCTIBLES`

**Units and combat**
`COLLECTION_PLAYER_UNITS`, `COLLECTION_ALL_UNITS`, `COLLECTION_PLAYER_COMBAT`,
`COLLECTION_UNIT_COMBAT`, `COLLECTION_OWNER_COMMANDER_HIGHEST_LEVEL`,
`COLLECTION_UNIT_NEAREST_OWNER_CITY`, `COLLECTION_UNIT_OCCUPIED_CITY`,
`COLLECTION_UNIT_OCCUPIED_DISTRICT`

**Plots / yields**
`COLLECTION_ALL_PLOT_YIELDS`, `COLLECTION_CITY_PLOT_YIELDS`,
`COLLECTION_PLAYER_PLOT_YIELDS`, `COLLECTION_SINGLE_PLOT_YIELDS`

**Trade / narrative**
`COLLECTION_PLAYER_TRADE_ROUTES`, `COLLECTION_NARRATIVE_STORY`,
`COLLECTION_ANY_CITY_AT_STORY`, `COLLECTION_OWNER_CITY_NEAREST_STORY`,
`COLLECTION_OWNER_UNIT_NEAREST_STORY`, `COLLECTION_OWNER_COMMANDER_NEAREST_STORY`,
`COLLECTION_INDEPENDENT_NEAREST_STORY`, `COLLECTION_HOMELANDS_INDEPENDENT_NEAREST_STORY`

## Effects (387) ✅

The full list is in [18-reference-enumerations.md](18-reference-enumerations.md).
The naming convention is readable and predictable:

```
EFFECT_<TARGET>_<VERB>_<OBJECT>
EFFECT_CITY_ADJUST_YIELD              — change a city's yield
EFFECT_PLAYER_GRANT_YIELD             — grant the player a yield
EFFECT_ADJUST_UNIT_STRENGTH_MODIFIER  — change a unit's strength
EFFECT_CITY_GRANT_UNIT                — give a city a unit
EFFECT_GRANT_WALLS                    — grant walls
```

Verbs: `ADJUST` (modify a value), `GRANT` (grant), `ATTACH`, `PLACE`, `CHANGE`.

**How to find the right effect:** search the game files for a mechanic similar to the one
you want to build:
```bash
grep -rl "EFFECT_CITY_ADJUST_YIELD" "Base/modules/age-antiquity/data/"
```

## Requirements (270 types) ✅

```xml
<SubjectRequirements>
    <Requirement type="REQUIREMENT_PLAYER_HAS_CIVILIZATION_OR_LEADER_TRAIT">
        <Argument name="TraitType">TRAIT_POLAND</Argument>
    </Requirement>
</SubjectRequirements>
```

The one most used in practice: `REQUIREMENT_PLAYER_HAS_CIVILIZATION_OR_LEADER_TRAIT`
— this is the standard way of saying "this bonus applies only to my civilization".

### Requirement sets (the table-based form) ✅

When you define requirements in SQL instead of inline in XML:

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

Only two set types exist ✅: `REQUIREMENTSET_TEST_ALL` (AND), `REQUIREMENTSET_TEST_ANY` (OR).

## Attaching a modifier to an object ✅

A modifier on its own does nothing — it has to be **attached**. The linking tables:

| Table | Attaches to |
|---|---|
| `TraitModifiers` | traits (so indirectly: a civilization/leader) |
| `TraditionModifiers` | a tradition |
| `ConstructibleModifiers` | a building/improvement |
| `UnitAbilityModifiers` | a unit ability |
| `UnitPromotionModifiers` | a promotion |
| `GovernmentModifiers` | a government |
| `BeliefModifiers` | a belief |
| `NarrativeRewards` (the `ModifierID` column) | a narrative reward |
| `GreatPersonIndividualActionModifiers` | a great person's action |
| `UniqueQuarterModifiers` | a unique quarter |
| `MementoModifiers` | a memento |
| `LegacyModifiers`, `GoldenAgeModifiers`, `CityStateBonusModifiers` | as above |

```sql
-- "Poland's trait grants this modifier"
INSERT INTO TraitModifiers(TraitType,ModifierId)
VALUES('TRAIT_POLAND_GOLDEN_LIBERTY','POLAND_ACTIVATE_MILITARY_FARM_AQ');
```

## An alternative without modifiers: the yield tables ✅

For simple yield bonuses you **do not need a modifier** — there are dedicated tables:

```sql
-- +1 food and +1 happiness from every farm in the city (Antiquity age)
INSERT INTO Warehouse_YieldChanges(ID,Age,YieldType,YieldChange,ConstructibleInCity) VALUES
('POLAND_GRANARY_I_FARM_FOOD','AGE_ANTIQUITY','YIELD_FOOD',1,'IMPROVEMENT_FARM'),
('POLAND_GRANARY_I_FARM_HAPPY','AGE_ANTIQUITY','YIELD_HAPPINESS',1,'IMPROVEMENT_FARM');

-- an adjacency bonus
INSERT INTO Adjacency_YieldChanges(ID,YieldType,YieldChange,TilesRequired,AdjacentConstructible)
VALUES('POLAND_MILITARY_FARM_AQ','YIELD_HAPPINESS',1,1,'IMPROVEMENT_FARM');
```
This is simpler and less error-prone — use it whenever it is enough.

## Table scale — how many rows this really is ✅ (measured 2026-08-25)

Counted directly in the game files (`Base/modules/**/*.xml`, all ages + `core` +
`base-standard`), by counting elements inside `<GameEffects>` blocks:

| element | count in the files |
|---|---|
| `<Modifier>` | 12,093 |
| `<Argument>` | 39,385 |
| `<Requirement>` | 15,147 |

The table form (`<Modifiers><Row/>`) is negligible by comparison: 86 `Modifiers` rows,
230 `ModifierArguments`, 376 `Requirements`, 2,357 `TypeTags` in the whole game.
So **almost all of the game's modifiers are written in `<GameEffects>` syntax** and only
end up in the `Modifiers` / `ModifierArguments` / `Requirements` tables at load time.

⚠️ **Consequence for UI mods:** at runtime `GameInfo` sees `core` + `base-standard` +
**only the current age**, so realistically it is on the order of a few thousand modifiers and
a few tens of thousands of arguments — but that still means *every* question of the kind "which
modifier applies to this resource", asked by scanning `GameInfo.Modifiers`, is a full grind
through several thousand rows. Index once and **limit the index to what you actually ask about**
(e.g. only modifiers attached to resources): keeping a map for all modifiers means thousands of
`Map` objects alive for the entire session. An example of such an index:
`mod-projects/better-commerce-screen-ui/ui/planner/effects.js`.

⚠️ **How much memory that costs** (measured in Node on synthetic tables of this size,
2026-08-25): a full index of `Modifiers` + `DynamicModifiers` + the `Requirements` graph in JS maps is
**~6.8 MB of live heap** held for the entire session. An index restricted to resource modifiers
costs practically nothing. The game engine is not Node, so treat this as an order of magnitude,
not gospel.
