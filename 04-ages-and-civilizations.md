# 04 — Ages, civilizations, progression trees

## The age system ✅

Civ VII splits the game into three ages, each a **separate module** of the game:

| Age | `AgeType` | Module |
|---|---|---|
| Antiquity | `AGE_ANTIQUITY` | `Base\modules\age-antiquity` |
| Exploration | `AGE_EXPLORATION` | `Base\modules\age-exploration` |
| Modern | `AGE_MODERN` | `Base\modules\age-modern` |

This has direct consequences for a mod: **age-specific data must live in an
`ActionGroup` with the `AgeInUse` criterion**, because the age's tables only exist then.

```xml
<Criteria id="aq"><AgeInUse>AGE_ANTIQUITY</AgeInUse></Criteria>
...
<ActionGroup id="my-mod-aq" scope="game" criteria="aq">
    <Actions><UpdateDatabase><Item>Core/age-antiquity.sql</Item></UpdateDatabase></Actions>
</ActionGroup>
```

The `AgeAtOrBefore` criterion allows "this age and earlier" (the base game uses it).

## A civilization — the `Civilizations` table ✅

Columns (complete, from the schema):
`CivilizationType`, `Adjective`, `AITargetCityPercentage`, `ApexAge`, `CapitalName`,
`Description`, `FullName`, `Name`, `RandomCityNameDepth`, `StartingCivilizationLevelType`,
`UniqueCultureProgressionTree`

```sql
INSERT INTO Civilizations(CivilizationType,Adjective,AITargetCityPercentage,ApexAge,
    CapitalName,Description,FullName,Name,RandomCityNameDepth,
    StartingCivilizationLevelType,UniqueCultureProgressionTree)
VALUES('CIVILIZATION_POLAND','LOC_CIVILIZATION_POLAND_ADJECTIVE',50,'AGE_MODERN',
    'LOC_CITY_NAME_POLAND_1','LOC_CIVILIZATION_POLAND_DESCRIPTION',
    'LOC_CIVILIZATION_POLAND_FULLNAME','LOC_CIVILIZATION_POLAND_NAME',10,
    'CIVILIZATION_LEVEL_FULL_CIV','TREE_CIVICS_MO_POLAND');
```

- `ApexAge` — the age in which the civilization is "full"/strongest
- `StartingCivilizationLevelType` — `CIVILIZATION_LEVEL_FULL_CIV` for a normal civilization
- `UniqueCultureProgressionTree` — ⚠️ **changes per age**; the Poland mod sets
  a base value and then `UPDATE`s it in that age's file

## Traits (`Traits`) ✅

Traits are the connector between a civilization and its bonuses.

```sql
INSERT INTO Traits(TraitType,Description,InternalOnly,Name) VALUES
  ('TRAIT_POLAND','LOC_..._DESCRIPTION',1,'LOC_..._NAME');

INSERT INTO CivilizationTraits(CivilizationType,TraitType) VALUES
  ('CIVILIZATION_POLAND','TRAIT_POLAND'),
  ('CIVILIZATION_POLAND','TRAIT_MODERN_CIV'),               -- age
  ('CIVILIZATION_POLAND','TRAIT_ATTRIBUTE_EXPANSIONIST'),   -- attribute
  ('CIVILIZATION_POLAND','TRAIT_ATTRIBUTE_MILITARISTIC');
```

Kinds of traits found in the game:
- `TRAIT_<CIV>` — the civilization's own trait (the carrier of its modifiers)
- `TRAIT_ANTIQUITY_CIV` / `TRAIT_EXPLORATION_CIV` / `TRAIT_MODERN_CIV` — assignment to an age
- `TRAIT_ATTRIBUTE_*` — attributes (EXPANSIONIST, MILITARISTIC, SCIENTIFIC, ECONOMIC,
  CULTURAL, DIPLOMATIC/POLITICAL), also the `_WIDE`, `_TOT_AQ`, `_TOT_EX` variants
- `TRAIT_ANACHRONISTIC_CIV` — a civilization playable outside its own age

## Age transitions and syncretism ✅

Civ VII lets you change civilization at an age transition. For a new civilization to be
available, you have to hook into several tables:

```sql
-- which civilizations unlock mine
INSERT INTO CivilizationSyncretismUnlocks(CivilizationType,UnlockCivilizationType)
VALUES('CIVILIZATION_ROME','CIVILIZATION_POLAND');

-- which leaders unlock it
INSERT INTO LeaderSyncretismUnlocks(LeaderType,UnlockCivilizationType)
SELECT LeaderType,'CIVILIZATION_POLAND' FROM Leaders WHERE LeaderType IN(...);

-- AI preferences (how eagerly a leader will pick this civilization)
INSERT INTO LeaderCivPriorities(Civilization,Leader,Priority)
SELECT 'CIVILIZATION_POLAND',LeaderType,3 FROM Leaders WHERE ...;
```

⚠️ **A trap documented in a comment by the Poland mod's author** — paraphrasing:
the syncretism screen resolves a civilization through the *legacy* tables **before** it
reaches for `CivSelfSyncretismUnlocks`. Without both rows below the effect will be granted,
but the card in the UI will be empty:

```sql
INSERT INTO LegacyCivilizations(CivilizationType,Name,FullName,Adjective,Age)
VALUES('CIVILIZATION_POLAND','LOC_..._NAME','LOC_..._FULLNAME','LOC_..._ADJECTIVE','AGE_MODERN');
INSERT INTO LegacyCivilizationTraits(CivilizationType,TraitType)
VALUES('CIVILIZATION_POLAND','TRAIT_POLAND');
```

## Traditions (`Traditions`) ✅

Columns: `TraditionType`, `AgeType`, `AllowInitializeAdvancedStart`, `CultureSlotType`,
`Description`, `IgnoreInitializeUnlock`, `IsCrisis`, `Name`, `ObsoletesTraditionType`,
`TraitType`

```sql
INSERT INTO Traditions(TraditionType,Name,Description,TraitType,AgeType,CultureSlotType,
    ObsoletesTraditionType,IgnoreInitializeUnlock,AllowInitializeAdvancedStart)
VALUES('TRADITION_POLAND_HETMAN_II','LOC_..._NAME','LOC_..._DESCRIPTION','TRAIT_POLAND',
    'AGE_MODERN','TRADITION_CULTURE_SLOT','TRADITION_POLAND_HETMAN_I',0,0);
```
`ObsoletesTraditionType` — the new tradition replaces the older version from the previous age.

Attaching an effect: `INSERT INTO TraditionModifiers(TraditionType,ModifierId) VALUES(...)`.

## Progression trees (`ProgressionTrees`) ✅

`ProgressionTrees` columns: `ProgressionTreeType`, `AgeType`, `CivInjectedName`,
`CostProgressionModel`, `IconString`, `MultipleUnlockName`, `Name`, `PrereqFormat`,
`RevealRequirementSetId`, `SystemType`

Adding your own node to an **existing** tree (the Poland mod's approach — less
invasive than a tree of your own):

```sql
INSERT INTO Types(Type,Kind) VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','KIND_TREE_NODE');

INSERT INTO ProgressionTreeNodes(ProgressionTreeNodeType,ProgressionTree,Cost,Name,IconString,CanSteal)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','TREE_CIVICS_AQ_TEST_OF_TIME',150,
       'LOC_NODE_CIVIC_AQ_POLAND_ORIGINS_NAME','cult_poland',0);

-- a node visible only to my civilization
INSERT INTO ProgressionTreeNodeTraits(ProgressionTreeNodeType,RequiredTraitType)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','TRAIT_POLAND');

-- its place in the graph (what comes after it)
INSERT INTO ProgressionTreePrereqs(Node,PrereqNode)
VALUES('NODE_CIVIC_AQ_FOUNDATION','NODE_CIVIC_AQ_POLAND_ORIGINS');

-- what it unlocks
INSERT INTO ProgressionTreeNodeUnlocks(ProgressionTreeNodeType,TargetKind,TargetType,UnlockDepth)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','KIND_TRADITION','TRADITION_POLAND_HETMAN_I',1);

-- the quote on the node's card (cosmetic)
INSERT INTO TypeQuotes(Type,Quote,QuoteAuthor)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','LOC_..._QUOTE','LOC_..._QUOTE_AUTHOR');
```

## City names and appearance ✅

```sql
INSERT INTO CityNames(CivilizationType,CityName) VALUES
  ('CIVILIZATION_POLAND','LOC_CITY_NAME_POLAND_1'), ... ;   -- the Poland mod has 29

-- architecture and unit style (uses the game's existing art sets)
INSERT INTO VisArt_CivilizationBuildingCultures(CivilizationType,BuildingCulture) VALUES
  ('CIVILIZATION_POLAND','BUILDING_CULTURE_NEU'),
  ('CIVILIZATION_POLAND','BUILDING_CULTURE_EEU_ANT'),
  ('CIVILIZATION_POLAND','ANT_STONE'),('CIVILIZATION_POLAND','EXP_STONE');
INSERT INTO VisArt_CivilizationUnitCultures(CivilizationType,UnitCulture)
VALUES('CIVILIZATION_POLAND','Euro');
```

## Narrative tied to a civilization ✅

```sql
INSERT INTO Types(Type,Kind) VALUES('POLAND_WAR_GOLD_STORY_AQ','KIND_NARRATIVE_STORY');
INSERT INTO NarrativeStories(NarrativeStoryType,Name,Description,Completion,Age,
    Activation,RequirementSetId,AllowDuplicates,Hidden,StartEveryone)
VALUES('POLAND_WAR_GOLD_STORY_AQ','LOC_..._NAME','LOC_..._DESC','LOC_..._DESC',
    'AGE_ANTIQUITY','AUTO','REQSET_POLAND_WAR_IN_AQ',1,1,1);
INSERT INTO NarrativeRewards(NarrativeRewardType,ModifierID)
VALUES('POLAND_WAR_GOLD_REWARD_AQ','POLAND_WAR_GOLD_AQ');
INSERT INTO NarrativeStory_Rewards(NarrativeStoryType,NarrativeRewardType,Activation)
VALUES('POLAND_WAR_GOLD_STORY_AQ','POLAND_WAR_GOLD_REWARD_AQ','COMPLETE');
```
`Hidden=1` + `StartEveryone=1` + `Activation='AUTO'` = a silent background mechanic
that fires once the requirements are met (here: declaring war).
