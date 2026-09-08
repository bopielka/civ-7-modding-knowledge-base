# 06 — Cookbook: a new civilization

A recipe reconstructed from a **working mod**, `szczupakabra-poland`
(Workshop ID `3768377608`, 53 files) ✅ — the best available reference model.

**Copy this mod for analysis before you start your own:**
```
C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660\3768377608
```

## File structure

```
my-civ\
├── my-civ.modinfo
├── Core\
│   ├── shared.sql              ← the civ definition, traits, traditions, syncretism (all ages)
│   ├── age-antiquity.sql       ← Antiquity age content
│   ├── age-exploration.sql
│   ├── age-modern.sql
│   ├── effects.xml             ← modifiers (GameEffects)
│   ├── unlocks.sql
│   └── narrative-events-*.sql
├── Shell\
│   ├── config.sql              ← visibility on the game setup screen
│   └── gameplay-preview.xml    ← the bonus preview in the menu
├── Art\
│   ├── ArtDefines.sql          ← mapping ID → PNG files
│   ├── VisualRemaps.xml        ← new types → existing 3D models
│   └── *.png
└── Text\
    ├── ModuleText.xml          ← the mod's name in the menu (LocalizedText)
    ├── MyCivText.xml           ← en_us texts
    └── MyCivText-pl_PL.xml     ← translation
```

## Step 1 — a `.modinfo` split by age and scope

The key structure: **6 action groups** (game/always, game×3 ages, remaps, shell).

```xml
<ActionCriteria>
    <Criteria id="always"><AlwaysMet /></Criteria>
    <Criteria id="aq"><AgeInUse>AGE_ANTIQUITY</AgeInUse></Criteria>
    <Criteria id="ex"><AgeInUse>AGE_EXPLORATION</AgeInUse></Criteria>
    <Criteria id="mo"><AgeInUse>AGE_MODERN</AgeInUse></Criteria>
</ActionCriteria>
```

⚠️ **Remember:** the `ImportFiles` with the artwork and the `UpdateText` have to be **repeated**
in the `shell` group, otherwise the civilization will not show up correctly in the game setup menu.

## Step 2 — `Types` (nothing works without this)

```sql
INSERT INTO Types(Type,Kind) VALUES
('CIVILIZATION_MINE','KIND_CIVILIZATION'),
('TRAIT_MINE','KIND_TRAIT'),
('TRAIT_MINE_UNIQUE','KIND_TRAIT'),
('TRADITION_MINE_I','KIND_TRADITION');
```

## Step 3 — the civilization and its traits

```sql
INSERT INTO Civilizations(CivilizationType,Adjective,AITargetCityPercentage,ApexAge,
    CapitalName,Description,FullName,Name,RandomCityNameDepth,
    StartingCivilizationLevelType,UniqueCultureProgressionTree)
VALUES('CIVILIZATION_MINE','LOC_CIVILIZATION_MINE_ADJECTIVE',50,'AGE_MODERN',
    'LOC_CITY_NAME_MINE_1','LOC_CIVILIZATION_MINE_DESCRIPTION',
    'LOC_CIVILIZATION_MINE_FULLNAME','LOC_CIVILIZATION_MINE_NAME',10,
    'CIVILIZATION_LEVEL_FULL_CIV','TREE_CIVICS_MO_MINE');

INSERT INTO Traits(TraitType,Description,InternalOnly,Name) VALUES
('TRAIT_MINE','LOC_CIVILIZATION_MINE_DESCRIPTION',1,'LOC_CIVILIZATION_MINE_NAME');

INSERT INTO CivilizationTraits(CivilizationType,TraitType) VALUES
('CIVILIZATION_MINE','TRAIT_MODERN_CIV'),
('CIVILIZATION_MINE','TRAIT_ATTRIBUTE_EXPANSIONIST'),
('CIVILIZATION_MINE','TRAIT_MINE');
```

## Step 4 — city names, appearance, favored wonder

```sql
INSERT INTO CityNames(CivilizationType,CityName) VALUES
('CIVILIZATION_MINE','LOC_CITY_NAME_MINE_1'), ... ;   -- at least a dozen or so

INSERT INTO VisArt_CivilizationBuildingCultures(CivilizationType,BuildingCulture) VALUES
('CIVILIZATION_MINE','BUILDING_CULTURE_NEU'),('CIVILIZATION_MINE','ANT_STONE');
INSERT INTO VisArt_CivilizationUnitCultures(CivilizationType,UnitCulture)
VALUES('CIVILIZATION_MINE','Euro');

INSERT INTO CivilizationFavoredWonders(CivilizationType,FavoredWonderType,FavoredWonderName)
VALUES('CIVILIZATION_MINE','WONDER_MY_WONDER','LOC_WONDER_MY_WONDER_NAME');
```

## Step 5 — hooking into the age system (syncretism)

```sql
-- unlocking via other civilizations / leaders
INSERT INTO CivilizationSyncretismUnlocks(CivilizationType,UnlockCivilizationType)
VALUES('CIVILIZATION_ROME','CIVILIZATION_MINE');

INSERT INTO LeaderSyncretismUnlocks(LeaderType,UnlockCivilizationType)
SELECT LeaderType,'CIVILIZATION_MINE' FROM Leaders
WHERE LeaderType IN('LEADER_CHARLEMAGNE','LEADER_CATHERINE');

INSERT INTO LeaderCivPriorities(Civilization,Leader,Priority)
SELECT 'CIVILIZATION_MINE',LeaderType,3 FROM Leaders WHERE LeaderType IN(...);

-- ⚠️ REQUIRED, otherwise the card in the UI will be empty
INSERT INTO LegacyCivilizations(CivilizationType,Name,FullName,Adjective,Age)
VALUES('CIVILIZATION_MINE','LOC_..._NAME','LOC_..._FULLNAME','LOC_..._ADJECTIVE','AGE_MODERN');
INSERT INTO LegacyCivilizationTraits(CivilizationType,TraitType)
VALUES('CIVILIZATION_MINE','TRAIT_MINE');
```

## Step 6 — icons and artwork

```sql
INSERT OR IGNORE INTO Icons(ID,Context) VALUES('CIVILIZATION_MINE','DEFAULT');
INSERT INTO IconDefinitions(ID,Path) VALUES
('ICON_CIVILIZATION_MINE','fs://game/my-civ/Art/civ_mine.png'),
('CIVILIZATION_MINE','fs://game/my-civ/Art/civ_mine.png'),
('civ_sym_mine','fs://game/my-civ/Art/civ_mine.png');

INSERT INTO IconDefinitions(ID,Context,Path,IconSize) VALUES
('CIVILIZATION_MINE','BACKGROUND','fs://game/my-civ/Art/bg_1080.png',1080),
('CIVILIZATION_MINE','BACKGROUND','fs://game/my-civ/Art/bg_720.png',720);
```
Hooked up via the `<UpdateIcons><Item>Art/ArtDefines.sql</Item></UpdateIcons>` action.

⚠️ In `ImportFiles` the Poland mod lists every file **twice** — with and without the
extension (`Art/civ_poland.png` and `Art/civ_poland`). I do not know whether this is necessary
or excess caution ❓ — see [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md).

## Step 7 — 3D models without making your own (VisualRemaps) ✅

The cleverest trick in this mod: new units/buildings **borrow the models** of existing ones.

```xml
<Database><VisualRemaps>
  <Row><ID>REMAP_MY_HUSSAR</ID><DisplayName>LOC_UNIT_MY_HUSSAR_NAME</DisplayName>
       <Kind>UNIT</Kind><From>UNIT_MY_HUSSAR</From><To>UNIT_HUSSAR</To></Row>
  <Row><ID>REMAP_MY_BUILDING</ID><DisplayName>LOC_BUILDING_MY_NAME</DisplayName>
       <Kind>BUILDING</Kind><From>BUILDING_MINE</From><To>BUILDING_BANK</To></Row>
</VisualRemaps></Database>
```
`Kind`: `UNIT`, `BUILDING`, `CONSTRUCTIBLE`. Thanks to this you **do not need to be able to model in 3D**
to build a complete civilization — 2D icons are enough.

## Step 8 — texts

```xml
<!-- en_us -->
<Database><EnglishText>
    <Row Tag="LOC_CIVILIZATION_MINE_NAME"><Text>Mine</Text></Row>
</EnglishText></Database>

<!-- pl_PL -->
<Database><LocalizedText>
    <Row Tag="LOC_CIVILIZATION_MINE_NAME" Language="pl_PL"><Text>Moja</Text></Row>
</LocalizedText></Database>
```

## An alternative file layout (a community pattern)

The Poland mod groups files by **layer** (`Core/`, `Shell/`, `Art/`, `Text/`).
Organizing by **content type** is also popular — e.g. the Scythia mod by the same author
who created `civ7-modding-tools`:

```
civmods-izica-civilization-scythia/
├── imports/            external dependencies
├── civilizations/
├── traditions/
├── units/
├── constructibles/
├── progression-trees/
└── izica-civilization-scythia.modinfo
```
Both work. For a single civilization, splitting by type is often more readable.

⚠️ Instead of writing the XML by hand you can generate it from TypeScript —
see [20-typescript-tooling.md](20-typescript-tooling.md). Worth doing only once you
already understand the target format.

## Recommended order of work

1. `.modinfo` + `Types` + `Civilizations` + minimal texts → **check that the civ appears in the menu**
2. Traits + traditions → check in game
3. Modifiers (`effects.xml`) → check whether the bonuses work
4. Progression tree, narrative
5. Artwork and icons last (the most tedious, the least critical)

Iterate in small steps — with 53 files at once you will not find what broke.
