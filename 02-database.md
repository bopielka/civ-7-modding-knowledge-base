# 02 — Warstwa bazodanowa

Cała rozgrywka Civ VII to w praktyce **baza SQLite** zbudowana przy starcie z plików
XML/SQL. Modowanie danych = dokładanie/zmiana wierszy w tej bazie.

## XML czy SQL? ✅

`UpdateDatabase` przyjmuje **oba** formaty — potwierdzone: gra bazowa używa XML,
mod Polska używa niemal wyłącznie `.sql`.

| | XML | SQL |
|---|---|---|
| Czytelność | lepsza przy prostych wierszach | lepsza przy masowych wstawkach |
| Warunki | brak | pełne `WHERE`, `SELECT`, `EXISTS` |
| Odporność na brak zależności | słaba | `INSERT OR IGNORE ... WHERE EXISTS(...)` |
| Używa gra bazowa | ✅ głównie | rzadko (`ages-post-process.sql`) |

**Rekomendacja:** proste dane → XML; logika warunkowa i kompatybilność z innymi
modami → SQL.

## Format XML

```xml
<?xml version="1.0" encoding="utf-8"?>
<Database>
    <Types>
        <Row Type="UNIT_MOJA_JEDNOSTKA" Kind="KIND_UNIT"/>
    </Types>
    <Units>
        <Row UnitType="UNIT_MOJA_JEDNOSTKA" BaseMoves="2" Name="LOC_UNIT_MOJA_NAME"/>
    </Units>
</Database>
```
Nazwa taga = nazwa tabeli, `<Row>` = wiersz, atrybuty = kolumny.

### Operacje dostępne w XML ✅

Zliczone w danych gry bazowej:

| Operacja | Liczba | Znaczenie |
|---|---|---|
| `<Row>` | 65184 | zwykły INSERT |
| `<InsertOrIgnore>` | 869 | INSERT, pomiń jeśli konflikt klucza |
| `<Update>` + `<Where>` + `<Set>` | 81 | UPDATE warunkowy |
| `<Replace>` | 33 | INSERT OR REPLACE |
| `<Delete>` | 21 | DELETE warunkowy |

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

## Format SQL

Zwykły SQLite. Przykłady z moda Polska (✅ działający kod):

```sql
-- zwykła wstawka
INSERT INTO Civilizations(CivilizationType, Name, ApexAge, ...)
VALUES ('CIVILIZATION_POLAND', 'LOC_CIVILIZATION_POLAND_NAME', 'AGE_MODERN', ...);

-- zmiana istniejącego wiersza
UPDATE Civilizations SET UniqueCultureProgressionTree='TREE_CIVICS_AQ_TEST_OF_TIME'
WHERE CivilizationType='CIVILIZATION_POLAND';

-- bezpieczne wobec braku innego moda/DLC — nie wysypie się, jeśli Bułgarii nie ma
INSERT OR IGNORE INTO CivilizationSyncretismUnlocks(CivilizationType, UnlockCivilizationType)
SELECT 'CIVILIZATION_BULGARIA','CIVILIZATION_POLAND'
WHERE EXISTS(SELECT 1 FROM Civilizations WHERE CivilizationType='CIVILIZATION_BULGARIA');

-- wstawka sterowana zapytaniem (dla wszystkich pasujących liderów)
INSERT INTO LeaderSyncretismUnlocks(LeaderType, UnlockCivilizationType)
SELECT LeaderType,'CIVILIZATION_POLAND' FROM Leaders
WHERE LeaderType IN('LEADER_CHARLEMAGNE','LEADER_ASHOKA','LEADER_CATHERINE');
```

⚠️ Mod Polska tworzy nawet własną tabelę:
`CREATE TABLE IF NOT EXISTS CivsWithoutBackgrounds(...)` — czyli **można dokładać własne
tabele**, jeśli inny mod/UI ich oczekuje.

## Fundament: tabela `Types` i `Kinds` ✅

**Każdy** nowy obiekt w grze musi najpierw zostać zarejestrowany w `Types`:

```sql
INSERT INTO Types(Type,Kind) VALUES
  ('CIVILIZATION_POLAND','KIND_CIVILIZATION'),
  ('TRAIT_POLAND','KIND_TRAIT'),
  ('TRADITION_POLAND_HETMAN_I','KIND_TRADITION');
```

Bez wiersza w `Types` klucze obce z innych tabel się nie powiążą. Znane `Kind`:
`KIND_CIVILIZATION`, `KIND_LEADER`, `KIND_TRAIT`, `KIND_TRADITION`, `KIND_UNIT`,
`KIND_CONSTRUCTIBLE`, `KIND_MODIFIER`, `KIND_TREE_NODE`, `KIND_NARRATIVE_STORY`,
`KIND_CULTURE_SLOT`.

## Mapa najważniejszych tabel (z 493)

**Cywilizacje i liderzy**
`Civilizations`, `CivilizationTraits`, `CivilizationInfo`, `CivilizationLevels`,
`Leaders`, `LeaderTraits`, `LeaderInfo`, `LeaderCivPriorities`, `Traits`, `TraitModifiers`,
`LegacyCivilizations`, `CivilizationSyncretismUnlocks`, `LeaderSyncretismUnlocks`

**Jednostki**
`Units`, `Unit_Stats`, `Unit_Costs`, `Unit_Abilities`, `UnitPromotions`, `UnitUpgrades`,
`UnitReplaces`, `UnitCommands`, `UnitOperations`, `UnitNames`

**Budowle**
`Constructibles` (nadrzędna), `Buildings`, `Improvements`, `Wonders`, `Districts`,
`Constructible_YieldChanges`, `Constructible_Adjacencies`, `Constructible_Maintenances`,
`UniqueQuarters`

**Postęp i kultura**
`ProgressionTrees`, `ProgressionTreeNodes`, `ProgressionTreePrereqs`,
`ProgressionTreeNodeUnlocks`, `ProgressionTreeNodeTraits`, `Traditions`,
`TraditionModifiers`, `Governments`, `Ideologies`

**Epoki i legacy**
`Ages`, `AgeProgressions`, `AgeTransition*`, `Legacies`, `LegacyPaths`, `AgeCrises`

**Modyfikatory** (patrz [03](03-modifiers-effects.md))
`Modifiers`, `DynamicModifiers`, `ModifierArguments`, `Requirements`,
`RequirementArguments`, `RequirementSets`, `GameEffects`, `GameEffectArguments`

**Narracja**
`NarrativeStories`, `NarrativeRewards`, `NarrativeStory_Rewards`, `NarrativeStory_Activations`

**Yieldy i przyległości**
`Yields`, `Adjacency_YieldChanges`, `Warehouse_YieldChanges`, `Constructible_WarehouseYields`

**Mapa**
`Terrains`, `Features`, `Biomes`, `Resources`, `Maps`, `StartBias*`, `Continents`

## Stary vs nowy system modyfikatorów ❓

W bazie **współistnieją dwa sposoby** definiowania modyfikatorów:

1. **Stary (tabelowy, dziedzictwo Civ VI):** `Modifiers` + `DynamicModifiers` +
   `ModifierArguments`. W danych gry bazowej tylko ~27 typów.
2. **Nowy (`<GameEffects>` XML):** 8640 modyfikatorów — to jest domyślna droga w Civ VII.

Nie ustaliłem, czy stary system jest deprecated czy po prostu używany do innych celów.
**Do modów używaj nowego** — patrz [03-modifiers-effects.md](03-modifiers-effects.md).
