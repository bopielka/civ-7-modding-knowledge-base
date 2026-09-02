# 02 — The database layer

All of Civ VII's gameplay is in practice a **SQLite database** built at startup from
XML/SQL files. Modding data = adding/changing rows in that database.

## XML or SQL? ✅

`UpdateDatabase` accepts **both** formats — confirmed: the base game uses XML,
the Poland mod uses almost exclusively `.sql`.

| | XML | SQL |
|---|---|---|
| Readability | better for simple rows | better for bulk inserts |
| Conditions | none | full `WHERE`, `SELECT`, `EXISTS` |
| Tolerance of missing dependencies | poor | `INSERT OR IGNORE ... WHERE EXISTS(...)` |
| Used by the base game | ✅ mostly | rarely (`ages-post-process.sql`) |

**Recommendation:** simple data → XML; conditional logic and compatibility with other
mods → SQL.

## The XML format

```xml
<?xml version="1.0" encoding="utf-8"?>
<Database>
    <Types>
        <Row Type="UNIT_MY_UNIT" Kind="KIND_UNIT"/>
    </Types>
    <Units>
        <Row UnitType="UNIT_MY_UNIT" BaseMoves="2" Name="LOC_UNIT_MY_NAME"/>
    </Units>
</Database>
```
The tag name = the table name, `<Row>` = a row, attributes = columns.

### Operations available in XML ✅

Counted in the base game's data:

| Operation | Count | Meaning |
|---|---|---|
| `<Row>` | 65184 | plain INSERT |
| `<InsertOrIgnore>` | 869 | INSERT, skip on key conflict |
| `<Update>` + `<Where>` + `<Set>` | 81 | conditional UPDATE |
| `<Replace>` | 33 | INSERT OR REPLACE |
| `<Delete>` | 21 | conditional DELETE |

```xml
<Ages>
    <Update>
        <Where AgeType="AGE_ANTIQUITY"/>
        <Set Active="1"/>
    </Update>
</Ages>

<Types>
    <Delete Type="UNIT_CARRIER_COMMANDER"/>
</Types>
```

## The SQL format

Plain SQLite. Examples from the Poland mod (✅ working code):

```sql
-- a plain insert
INSERT INTO Civilizations(CivilizationType, Name, ApexAge, ...)
VALUES ('CIVILIZATION_POLAND', 'LOC_CIVILIZATION_POLAND_NAME', 'AGE_MODERN', ...);

-- changing an existing row
UPDATE Civilizations SET UniqueCultureProgressionTree='TREE_CIVICS_AQ_TEST_OF_TIME'
WHERE CivilizationType='CIVILIZATION_POLAND';

-- safe against a missing mod/DLC — will not blow up if Bulgaria is absent
INSERT OR IGNORE INTO CivilizationSyncretismUnlocks(CivilizationType, UnlockCivilizationType)
SELECT 'CIVILIZATION_BULGARIA','CIVILIZATION_POLAND'
WHERE EXISTS(SELECT 1 FROM Civilizations WHERE CivilizationType='CIVILIZATION_BULGARIA');

-- a query-driven insert (for every matching leader)
INSERT INTO LeaderSyncretismUnlocks(LeaderType, UnlockCivilizationType)
SELECT LeaderType,'CIVILIZATION_POLAND' FROM Leaders
WHERE LeaderType IN('LEADER_CHARLEMAGNE','LEADER_ASHOKA','LEADER_CATHERINE');
```

⚠️ The Poland mod even creates its own table:
`CREATE TABLE IF NOT EXISTS CivsWithoutBackgrounds(...)` — so you **can add your own
tables** if another mod/UI expects them.

## The foundation: the `Types` and `Kinds` tables ✅

**Every** new object in the game must first be registered in `Types`:

```sql
INSERT INTO Types(Type,Kind) VALUES
  ('CIVILIZATION_POLAND','KIND_CIVILIZATION'),
  ('TRAIT_POLAND','KIND_TRAIT'),
  ('TRADITION_POLAND_HETMAN_I','KIND_TRADITION');
```

Without a row in `Types`, foreign keys from other tables will not resolve. Known `Kind` values:
`KIND_CIVILIZATION`, `KIND_LEADER`, `KIND_TRAIT`, `KIND_TRADITION`, `KIND_UNIT`,
`KIND_CONSTRUCTIBLE`, `KIND_MODIFIER`, `KIND_TREE_NODE`, `KIND_NARRATIVE_STORY`,
`KIND_CULTURE_SLOT`.

## Map of the most important tables (out of 493)

**Civilizations and leaders**
`Civilizations`, `CivilizationTraits`, `CivilizationInfo`, `CivilizationLevels`,
`Leaders`, `LeaderTraits`, `LeaderInfo`, `LeaderCivPriorities`, `Traits`, `TraitModifiers`,
`LegacyCivilizations`, `CivilizationSyncretismUnlocks`, `LeaderSyncretismUnlocks`

**Units**
`Units`, `Unit_Stats`, `Unit_Costs`, `Unit_Abilities`, `UnitPromotions`, `UnitUpgrades`,
`UnitReplaces`, `UnitCommands`, `UnitOperations`, `UnitNames`

**Constructibles**
`Constructibles` (the parent), `Buildings`, `Improvements`, `Wonders`, `Districts`,
`Constructible_YieldChanges`, `Constructible_Adjacencies`, `Constructible_Maintenances`,
`UniqueQuarters`

**Progression and culture**
`ProgressionTrees`, `ProgressionTreeNodes`, `ProgressionTreePrereqs`,
`ProgressionTreeNodeUnlocks`, `ProgressionTreeNodeTraits`, `Traditions`,
`TraditionModifiers`, `Governments`, `Ideologies`

**Ages and legacies**
`Ages`, `AgeProgressions`, `AgeTransition*`, `Legacies`, `LegacyPaths`, `AgeCrises`

**Modifiers** (see [03](03-modifiers-effects.md))
`Modifiers`, `DynamicModifiers`, `ModifierArguments`, `Requirements`,
`RequirementArguments`, `RequirementSets`, `GameEffects`, `GameEffectArguments`

**Narrative**
`NarrativeStories`, `NarrativeRewards`, `NarrativeStory_Rewards`, `NarrativeStory_Activations`

**Yields and adjacencies**
`Yields`, `Adjacency_YieldChanges`, `Warehouse_YieldChanges`, `Constructible_WarehouseYields`

**Map**
`Terrains`, `Features`, `Biomes`, `Resources`, `Maps`, `StartBias*`, `Continents`

## The old vs the new modifier system ❓

**Two ways** of defining modifiers **coexist** in the database:

1. **The old one (table-based, inherited from Civ VI):** `Modifiers` + `DynamicModifiers` +
   `ModifierArguments`. Only ~27 types in the base game's data.
2. **The new one (`<GameEffects>` XML):** 8640 modifiers — this is the default route in Civ VII.

I have not established whether the old system is deprecated or simply used for other purposes.
**For mods use the new one** — see [03-modifiers-effects.md](03-modifiers-effects.md).
