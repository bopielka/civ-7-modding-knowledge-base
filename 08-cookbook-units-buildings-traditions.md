# 08 — Cookbook: jednostki, budynki, ulepszenia, tradycje

Najlepszy punkt startu dla początkującego moddera — mała, testowalna zmiana.

## Model: wszystko budowalne to `Constructible` ✅

```
Constructibles  (tabela nadrzędna — wspólne pola)
    ├── Buildings      (ConstructibleClass = BUILDING)
    ├── Improvements   (ConstructibleClass = IMPROVEMENT)
    ├── Wonders        (ConstructibleClass = WONDER)
    └── Districts
```
Każdy budynek/ulepszenie/cud ma **wiersz w `Constructibles`** plus wiersz
w tabeli szczegółowej. Klucz łączący: `ConstructibleType`.

## Najprostszy możliwy mod — zmiana istniejącej wartości

Najlepszy pierwszy test „czy mój mod w ogóle się ładuje":

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
Zwiadowca z 5 ruchami jest natychmiast widoczny w grze — od razu wiesz, czy działa.

## Jednostka ✅

Kolumny `Units` (wybrane z 64):
`UnitType`, `Name`, `Description`, `BaseMoves`, `BaseSightRange`, `Domain`,
`CoreClass`, `FormationClass`, `PromotionClass`, `UnitMovementClass`, `Maintenance`,
`Tier`, `TraitType`, `CanTrain`, `CanPurchase`, `BuildCharges`, `FoundCity`,
`ZoneOfControl`, `Stackable`, `ConstructibleType`, `CostProgressionModel`

Kolumny `Unit_Stats`: `UnitType`, `Combat`, `RangedCombat`, `Bombard`, `Range`, `WMDType`

```sql
INSERT INTO Types(Type,Kind) VALUES('UNIT_MOJ_HUSARZ','KIND_UNIT');

INSERT INTO Units(UnitType,Name,Description,BaseMoves,BaseSightRange,Domain,
    CoreClass,FormationClass,PromotionClass,UnitMovementClass,Maintenance,Tier,TraitType)
VALUES('UNIT_MOJ_HUSARZ','LOC_UNIT_MOJ_HUSARZ_NAME','LOC_UNIT_MOJ_HUSARZ_DESCRIPTION',
    4,2,'DOMAIN_LAND','CORE_CLASS_MILITARY','FORMATION_CLASS_LAND_COMBAT',
    'PROMOTION_CLASS_CAVALRY','UNIT_MOVEMENT_CLASS_FOOT',2,3,'TRAIT_MOJA');

INSERT INTO Unit_Stats(UnitType,Combat) VALUES('UNIT_MOJ_HUSARZ',36);

INSERT INTO Unit_Costs(UnitType,YieldType,Cost)
VALUES('UNIT_MOJ_HUSARZ','YIELD_PRODUCTION',110);
```

⚠️ `TraitType='TRAIT_MOJA'` sprawia, że jednostka jest **unikalna dla twojej cywilizacji**.

Jednostka unikalna zwykle **zastępuje** standardową:
```sql
INSERT INTO UnitReplaces(CivUniqueUnitType,ReplacesUnitType)
VALUES('UNIT_MOJ_HUSARZ','UNIT_CAVALRY');
```
✅ `UnitReplaces` ma dokładnie dwie kolumny: `CivUniqueUnitType`, `ReplacesUnitType`.
Ścieżka ulepszania jednostek: `UnitUpgrades(Unit, UpgradeUnit)`.

**Model 3D:** użyj `VisualRemaps` (patrz [06](06-cookbook-new-civilization.md) krok 7) —
to jedyna realna droga bez pipeline'u artystycznego.

## Budynek ✅

Kolumny `Constructibles` (wybrane z 34): `ConstructibleType`, `Name`, `Description`,
`ConstructibleClass`, `Age`, `Cost`, `Population`, `Defense`, `Tooltip`,
`AdjacentRiver`, `AdjacentTerrain`, `AdjacentDistrict`, `RequiresUnlock`,
`RequiresHomeland`, `RequiresDistantLands`, `Repairable`

Kolumny `Buildings` (22): `ConstructibleType`, `TraitType`, `Housing`, `CitizenSlots`,
`Capital`, `CapitalForbidden`, `Town`, `Workable`, `Purchasable`, `MustPurchase`,
`MaxPlayerInstances`, `MultiplePerCity`, `DefenseModifier`, `GrantFortification`,
`OuterDefenseStrength`, `OuterDefenseHitPoints`, `BuildQueue`, `CityCenterPriority`,
`AllowsHolyCity`, `ArchaeologyResearch`, `Movable`, `PurchaseYield`

```sql
INSERT INTO Types(Type,Kind) VALUES('BUILDING_MOJE_SUKIENNICE','KIND_CONSTRUCTIBLE');

INSERT INTO Constructibles(ConstructibleType,Name,Description,ConstructibleClass,
    Age,Cost,Population,Repairable)
VALUES('BUILDING_MOJE_SUKIENNICE','LOC_BUILDING_MOJE_SUKIENNICE_NAME',
    'LOC_BUILDING_MOJE_SUKIENNICE_DESCRIPTION','BUILDING','AGE_EXPLORATION',180,1,1);

INSERT INTO Buildings(ConstructibleType,TraitType,Housing,CitizenSlots,Purchasable)
VALUES('BUILDING_MOJE_SUKIENNICE','TRAIT_MOJA',2,1,1);

-- yieldy budynku
INSERT INTO Constructible_YieldChanges(ConstructibleType,YieldType,YieldChange)
VALUES('BUILDING_MOJE_SUKIENNICE','YIELD_GOLD',4),
      ('BUILDING_MOJE_SUKIENNICE','YIELD_CULTURE',2);
```

### Bonusy za sąsiedztwo
```sql
INSERT INTO Constructible_Adjacencies(ConstructibleType,YieldChangeId)
VALUES('BUILDING_MOJE_SUKIENNICE','MOJE_SUKIENNICE_ADJ_RIVER');

INSERT INTO Adjacency_YieldChanges(ID,YieldType,YieldChange,TilesRequired,AdjacentRiver)
VALUES('MOJE_SUKIENNICE_ADJ_RIVER','YIELD_GOLD',1,1,1);
```

## Cud (`Wonders`) ✅

Kolumny: `ConstructibleType`, `MaxPerPlayer`, `MaxWorldInstances`, `AdjacentCapital`,
`AdjacentConstructible`, `AdjacentResource`, `AdjacentToLand`, `AdjacentToMountain`,
`MustBeLake`, `MustNotBeLake`, `BuildOnFrontier`, `RequiredConstructibleInSettlement`,
`RequiredConstructibleInSettlementCount`

```sql
INSERT INTO Constructibles(ConstructibleType,Name,ConstructibleClass,Age,Cost)
VALUES('WONDER_MOJ_ZAMEK','LOC_WONDER_MOJ_ZAMEK_NAME','WONDER','AGE_EXPLORATION',600);
INSERT INTO Wonders(ConstructibleType,MaxWorldInstances,AdjacentToMountain)
VALUES('WONDER_MOJ_ZAMEK',1,1);
```

## Ulepszenie (`Improvements`) ✅

Kolumny (wybrane z 31): `ConstructibleType`, `TraitType`, `UnitBuildable`,
`CityBuildable`, `TownBuildable`, `CanBuildOutsideTerritory`, `CanBuildOnNonDistrict`,
`OnePerSettlement`, `ResourceTier`, `Domain`, `DefenseModifier`, `BarbarianCamp`,
`Workable`, `MinimumPopulation`, `Icon`

## Tradycja ✅

```sql
INSERT INTO Types(Type,Kind) VALUES('TRADITION_MOJA_I','KIND_TRADITION');

INSERT INTO Traditions(TraditionType,Name,Description,TraitType,AgeType,
    CultureSlotType,ObsoletesTraditionType,IgnoreInitializeUnlock,AllowInitializeAdvancedStart)
VALUES('TRADITION_MOJA_I','LOC_TRADITION_MOJA_I_NAME','LOC_TRADITION_MOJA_I_DESCRIPTION',
    'TRAIT_MOJA','AGE_ANTIQUITY','TRADITION_CULTURE_SLOT',NULL,0,0);

INSERT INTO TraditionModifiers(TraditionType,ModifierId)
VALUES('TRADITION_MOJA_I','MOD_MOJA_TRADYCJA_BONUS');
```
Tradycja musi być **odblokowana** przez węzeł drzewa rozwoju
(`ProgressionTreeNodeUnlocks`, patrz [04](04-ages-and-civilizations.md)).

## Checklist dla każdego nowego obiektu

1. ☐ wiersz w `Types` z właściwym `Kind`
2. ☐ wiersz w tabeli głównej (`Units` / `Constructibles`)
3. ☐ wiersz w tabeli szczegółowej (`Unit_Stats` / `Buildings`)
4. ☐ koszt (`Unit_Costs` / kolumna `Cost`)
5. ☐ teksty `LOC_*` (nazwa + opis) — bez nich w UI zobaczysz surowy klucz
6. ☐ ikona (`IconDefinitions`)
7. ☐ model 3D (`VisualRemaps`)
8. ☐ odblokowanie (drzewo rozwoju / `RequiresUnlock`)
9. ☐ przypisanie do epoki (`Age`) i cywilizacji (`TraitType`)
