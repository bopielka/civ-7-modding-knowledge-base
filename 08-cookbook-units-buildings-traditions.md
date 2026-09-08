# 08 — Cookbook: units, buildings, improvements, traditions

The best starting point for a beginner modder — a small, testable change.

## The model: everything buildable is a `Constructible` ✅

```
Constructibles  (the parent table — shared fields)
    ├── Buildings      (ConstructibleClass = BUILDING)
    ├── Improvements   (ConstructibleClass = IMPROVEMENT)
    ├── Wonders        (ConstructibleClass = WONDER)
    └── Districts
```
Every building/improvement/wonder has a **row in `Constructibles`** plus a row
in the detail table. The joining key: `ConstructibleType`.

## The simplest possible mod — changing an existing value

The best first test of "does my mod load at all":

```xml
<?xml version="1.0" encoding="utf-8"?>
<Database>
    <Units>
        <Update>
            <Where UnitType="UNIT_SCOUT"/>
            <Set BaseMoves="5"/>
        </Update>
    </Units>
</Database>
```
A scout with 5 movement is immediately visible in the game — you know right away whether it works.

## A unit ✅

`Units` columns (a selection out of 64):
`UnitType`, `Name`, `Description`, `BaseMoves`, `BaseSightRange`, `Domain`,
`CoreClass`, `FormationClass`, `PromotionClass`, `UnitMovementClass`, `Maintenance`,
`Tier`, `TraitType`, `CanTrain`, `CanPurchase`, `BuildCharges`, `FoundCity`,
`ZoneOfControl`, `Stackable`, `ConstructibleType`, `CostProgressionModel`

`Unit_Stats` columns: `UnitType`, `Combat`, `RangedCombat`, `Bombard`, `Range`, `WMDType`

```sql
INSERT INTO Types(Type,Kind) VALUES('UNIT_MY_HUSSAR','KIND_UNIT');

INSERT INTO Units(UnitType,Name,Description,BaseMoves,BaseSightRange,Domain,
    CoreClass,FormationClass,PromotionClass,UnitMovementClass,Maintenance,Tier,TraitType)
VALUES('UNIT_MY_HUSSAR','LOC_UNIT_MY_HUSSAR_NAME','LOC_UNIT_MY_HUSSAR_DESCRIPTION',
    4,2,'DOMAIN_LAND','CORE_CLASS_MILITARY','FORMATION_CLASS_LAND_COMBAT',
    'PROMOTION_CLASS_CAVALRY','UNIT_MOVEMENT_CLASS_FOOT',2,3,'TRAIT_MINE');

INSERT INTO Unit_Stats(UnitType,Combat) VALUES('UNIT_MY_HUSSAR',36);

INSERT INTO Unit_Costs(UnitType,YieldType,Cost)
VALUES('UNIT_MY_HUSSAR','YIELD_PRODUCTION',110);
```

⚠️ `TraitType='TRAIT_MINE'` makes the unit **unique to your civilization**.

A unique unit usually **replaces** a standard one:
```sql
INSERT INTO UnitReplaces(CivUniqueUnitType,ReplacesUnitType)
VALUES('UNIT_MY_HUSSAR','UNIT_CAVALRY');
```
✅ `UnitReplaces` has exactly two columns: `CivUniqueUnitType`, `ReplacesUnitType`.
The unit upgrade path: `UnitUpgrades(Unit, UpgradeUnit)`.

**3D model:** use `VisualRemaps` (see [06](06-cookbook-new-civilization.md), step 7) —
it is the only realistic route without an art pipeline.

## A building ✅

`Constructibles` columns (a selection out of 34): `ConstructibleType`, `Name`, `Description`,
`ConstructibleClass`, `Age`, `Cost`, `Population`, `Defense`, `Tooltip`,
`AdjacentRiver`, `AdjacentTerrain`, `AdjacentDistrict`, `RequiresUnlock`,
`RequiresHomeland`, `RequiresDistantLands`, `Repairable`

`Buildings` columns (22): `ConstructibleType`, `TraitType`, `Housing`, `CitizenSlots`,
`Capital`, `CapitalForbidden`, `Town`, `Workable`, `Purchasable`, `MustPurchase`,
`MaxPlayerInstances`, `MultiplePerCity`, `DefenseModifier`, `GrantFortification`,
`OuterDefenseStrength`, `OuterDefenseHitPoints`, `BuildQueue`, `CityCenterPriority`,
`AllowsHolyCity`, `ArchaeologyResearch`, `Movable`, `PurchaseYield`

```sql
INSERT INTO Types(Type,Kind) VALUES('BUILDING_MY_CLOTH_HALL','KIND_CONSTRUCTIBLE');

INSERT INTO Constructibles(ConstructibleType,Name,Description,ConstructibleClass,
    Age,Cost,Population,Repairable)
VALUES('BUILDING_MY_CLOTH_HALL','LOC_BUILDING_MY_CLOTH_HALL_NAME',
    'LOC_BUILDING_MY_CLOTH_HALL_DESCRIPTION','BUILDING','AGE_EXPLORATION',180,1,1);

INSERT INTO Buildings(ConstructibleType,TraitType,Housing,CitizenSlots,Purchasable)
VALUES('BUILDING_MY_CLOTH_HALL','TRAIT_MINE',2,1,1);

-- the building's yields
INSERT INTO Constructible_YieldChanges(ConstructibleType,YieldType,YieldChange)
VALUES('BUILDING_MY_CLOTH_HALL','YIELD_GOLD',4),
      ('BUILDING_MY_CLOTH_HALL','YIELD_CULTURE',2);
```

### Adjacency bonuses
```sql
INSERT INTO Constructible_Adjacencies(ConstructibleType,YieldChangeId)
VALUES('BUILDING_MY_CLOTH_HALL','MY_CLOTH_HALL_ADJ_RIVER');

INSERT INTO Adjacency_YieldChanges(ID,YieldType,YieldChange,TilesRequired,AdjacentRiver)
VALUES('MY_CLOTH_HALL_ADJ_RIVER','YIELD_GOLD',1,1,1);
```

## A wonder (`Wonders`) ✅

Columns: `ConstructibleType`, `MaxPerPlayer`, `MaxWorldInstances`, `AdjacentCapital`,
`AdjacentConstructible`, `AdjacentResource`, `AdjacentToLand`, `AdjacentToMountain`,
`MustBeLake`, `MustNotBeLake`, `BuildOnFrontier`, `RequiredConstructibleInSettlement`,
`RequiredConstructibleInSettlementCount`

```sql
INSERT INTO Constructibles(ConstructibleType,Name,ConstructibleClass,Age,Cost)
VALUES('WONDER_MY_CASTLE','LOC_WONDER_MY_CASTLE_NAME','WONDER','AGE_EXPLORATION',600);
INSERT INTO Wonders(ConstructibleType,MaxWorldInstances,AdjacentToMountain)
VALUES('WONDER_MY_CASTLE',1,1);
```

## An improvement (`Improvements`) ✅

Columns (a selection out of 31): `ConstructibleType`, `TraitType`, `UnitBuildable`,
`CityBuildable`, `TownBuildable`, `CanBuildOutsideTerritory`, `CanBuildOnNonDistrict`,
`OnePerSettlement`, `ResourceTier`, `Domain`, `DefenseModifier`, `BarbarianCamp`,
`Workable`, `MinimumPopulation`, `Icon`

## A tradition ✅

```sql
INSERT INTO Types(Type,Kind) VALUES('TRADITION_MINE_I','KIND_TRADITION');

INSERT INTO Traditions(TraditionType,Name,Description,TraitType,AgeType,
    CultureSlotType,ObsoletesTraditionType,IgnoreInitializeUnlock,AllowInitializeAdvancedStart)
VALUES('TRADITION_MINE_I','LOC_TRADITION_MINE_I_NAME','LOC_TRADITION_MINE_I_DESCRIPTION',
    'TRAIT_MINE','AGE_ANTIQUITY','TRADITION_CULTURE_SLOT',NULL,0,0);

INSERT INTO TraditionModifiers(TraditionType,ModifierId)
VALUES('TRADITION_MINE_I','MOD_MY_TRADITION_BONUS');
```
A tradition has to be **unlocked** by a progression tree node
(`ProgressionTreeNodeUnlocks`, see [04](04-ages-and-civilizations.md)).

## Checklist for every new object

1. ☐ a row in `Types` with the right `Kind`
2. ☐ a row in the main table (`Units` / `Constructibles`)
3. ☐ a row in the detail table (`Unit_Stats` / `Buildings`)
4. ☐ cost (`Unit_Costs` / the `Cost` column)
5. ☐ `LOC_*` texts (name + description) — without them the UI shows the raw key
6. ☐ an icon (`IconDefinitions`)
7. ☐ a 3D model (`VisualRemaps`)
8. ☐ an unlock (progression tree / `RequiresUnlock`)
9. ☐ assignment to an age (`Age`) and a civilization (`TraitType`)
