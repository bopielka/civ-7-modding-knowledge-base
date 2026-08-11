# 04 — Epoki, cywilizacje, drzewa rozwoju

## System epok ✅

Civ VII dzieli grę na trzy epoki, każda jako **osobny moduł** gry:

| Epoka | `AgeType` | Moduł |
|---|---|---|
| Starożytność | `AGE_ANTIQUITY` | `Base\modules\age-antiquity` |
| Eksploracja | `AGE_EXPLORATION` | `Base\modules\age-exploration` |
| Nowoczesność | `AGE_MODERN` | `Base\modules\age-modern` |

To ma bezpośrednie konsekwencje dla moda: **dane specyficzne dla epoki muszą być
w `ActionGroup` z kryterium `AgeInUse`**, bo tabele epoki istnieją tylko wtedy.

```xml
<Criteria id="aq"><AgeInUse>AGE_ANTIQUITY</AgeInUse></Criteria>
...
<ActionGroup id="moj-mod-aq" scope="game" criteria="aq">
    <Actions><UpdateDatabase><Item>Core/age-antiquity.sql</Item></UpdateDatabase></Actions>
</ActionGroup>
```

Kryterium `AgeAtOrBefore` pozwala na „ta epoka i wcześniejsze" (używa go gra bazowa).

## Cywilizacja — tabela `Civilizations` ✅

Kolumny (pełne, ze schematu):
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

- `ApexAge` — epoka, w której cywilizacja jest „pełna"/najsilniejsza
- `StartingCivilizationLevelType` — `CIVILIZATION_LEVEL_FULL_CIV` dla normalnej cywilizacji
- `UniqueCultureProgressionTree` — ⚠️ **zmienia się per epoka**; mod Polska ustawia
  wartość bazową, a potem `UPDATE`-uje ją w pliku danej epoki

## Cechy (`Traits`) ✅

Cechy to spinacz między cywilizacją a jej bonusami.

```sql
INSERT INTO Traits(TraitType,Description,InternalOnly,Name) VALUES
  ('TRAIT_POLAND','LOC_..._DESCRIPTION',1,'LOC_..._NAME');

INSERT INTO CivilizationTraits(CivilizationType,TraitType) VALUES
  ('CIVILIZATION_POLAND','TRAIT_POLAND'),
  ('CIVILIZATION_POLAND','TRAIT_MODERN_CIV'),               -- epoka
  ('CIVILIZATION_POLAND','TRAIT_ATTRIBUTE_EXPANSIONIST'),   -- atrybut
  ('CIVILIZATION_POLAND','TRAIT_ATTRIBUTE_MILITARISTIC');
```

Rodzaje cech spotykane w grze:
- `TRAIT_<CIV>` — własna cecha cywilizacji (nośnik jej modyfikatorów)
- `TRAIT_ANTIQUITY_CIV` / `TRAIT_EXPLORATION_CIV` / `TRAIT_MODERN_CIV` — przypisanie do epoki
- `TRAIT_ATTRIBUTE_*` — atrybuty (EXPANSIONIST, MILITARISTIC, SCIENTIFIC, ECONOMIC,
  CULTURAL, DIPLOMATIC/POLITICAL), także warianty `_WIDE`, `_TOT_AQ`, `_TOT_EX`
- `TRAIT_ANACHRONISTIC_CIV` — cywilizacja grywalna poza swoją epoką

## Przejścia między epokami i syncretism ✅

Civ VII pozwala zmienić cywilizację przy przejściu epoki. Żeby nowa cywilizacja była
dostępna, trzeba wpiąć się w kilka tabel:

```sql
-- które cywilizacje odblokowują moją
INSERT INTO CivilizationSyncretismUnlocks(CivilizationType,UnlockCivilizationType)
VALUES('CIVILIZATION_ROME','CIVILIZATION_POLAND');

-- którzy liderzy ją odblokowują
INSERT INTO LeaderSyncretismUnlocks(LeaderType,UnlockCivilizationType)
SELECT LeaderType,'CIVILIZATION_POLAND' FROM Leaders WHERE LeaderType IN(...);

-- preferencje AI (jak chętnie lider wybierze tę cywilizację)
INSERT INTO LeaderCivPriorities(Civilization,Leader,Priority)
SELECT 'CIVILIZATION_POLAND',LeaderType,3 FROM Leaders WHERE ...;
```

⚠️ **Pułapka udokumentowana w komentarzu autora moda Polska** — cytuję sens:
ekran syncretismu rozwiązuje cywilizację przez tabele *legacy* **zanim** sięgnie po
`CivSelfSyncretismUnlocks`. Bez obu wierszy poniżej efekt zostanie przyznany, ale
karta w UI będzie pusta:

```sql
INSERT INTO LegacyCivilizations(CivilizationType,Name,FullName,Adjective,Age)
VALUES('CIVILIZATION_POLAND','LOC_..._NAME','LOC_..._FULLNAME','LOC_..._ADJECTIVE','AGE_MODERN');
INSERT INTO LegacyCivilizationTraits(CivilizationType,TraitType)
VALUES('CIVILIZATION_POLAND','TRAIT_POLAND');
```

## Tradycje (`Traditions`) ✅

Kolumny: `TraditionType`, `AgeType`, `AllowInitializeAdvancedStart`, `CultureSlotType`,
`Description`, `IgnoreInitializeUnlock`, `IsCrisis`, `Name`, `ObsoletesTraditionType`,
`TraitType`

```sql
INSERT INTO Traditions(TraditionType,Name,Description,TraitType,AgeType,CultureSlotType,
    ObsoletesTraditionType,IgnoreInitializeUnlock,AllowInitializeAdvancedStart)
VALUES('TRADITION_POLAND_HETMAN_II','LOC_..._NAME','LOC_..._DESCRIPTION','TRAIT_POLAND',
    'AGE_MODERN','TRADITION_CULTURE_SLOT','TRADITION_POLAND_HETMAN_I',0,0);
```
`ObsoletesTraditionType` — nowa tradycja zastępuje starszą wersję z poprzedniej epoki.

Podpięcie efektu: `INSERT INTO TraditionModifiers(TraditionType,ModifierId) VALUES(...)`.

## Drzewa rozwoju (`ProgressionTrees`) ✅

Kolumny `ProgressionTrees`: `ProgressionTreeType`, `AgeType`, `CivInjectedName`,
`CostProgressionModel`, `IconString`, `MultipleUnlockName`, `Name`, `PrereqFormat`,
`RevealRequirementSetId`, `SystemType`

Dodanie własnego węzła do **istniejącego** drzewa (podejście moda Polska — mniej
inwazyjne niż własne drzewo):

```sql
INSERT INTO Types(Type,Kind) VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','KIND_TREE_NODE');

INSERT INTO ProgressionTreeNodes(ProgressionTreeNodeType,ProgressionTree,Cost,Name,IconString,CanSteal)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','TREE_CIVICS_AQ_TEST_OF_TIME',150,
       'LOC_NODE_CIVIC_AQ_POLAND_ORIGINS_NAME','cult_poland',0);

-- węzeł widoczny tylko dla mojej cywilizacji
INSERT INTO ProgressionTreeNodeTraits(ProgressionTreeNodeType,RequiredTraitType)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','TRAIT_POLAND');

-- miejsce w grafie (co jest po nim)
INSERT INTO ProgressionTreePrereqs(Node,PrereqNode)
VALUES('NODE_CIVIC_AQ_FOUNDATION','NODE_CIVIC_AQ_POLAND_ORIGINS');

-- co odblokowuje
INSERT INTO ProgressionTreeNodeUnlocks(ProgressionTreeNodeType,TargetKind,TargetType,UnlockDepth)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','KIND_TRADITION','TRADITION_POLAND_HETMAN_I',1);

-- cytat na karcie węzła (kosmetyka)
INSERT INTO TypeQuotes(Type,Quote,QuoteAuthor)
VALUES('NODE_CIVIC_AQ_POLAND_ORIGINS','LOC_..._QUOTE','LOC_..._QUOTE_AUTHOR');
```

## Nazwy miast i wygląd ✅

```sql
INSERT INTO CityNames(CivilizationType,CityName) VALUES
  ('CIVILIZATION_POLAND','LOC_CITY_NAME_POLAND_1'), ... ;   -- mod Polska ma 29

-- styl architektury i jednostek (używa istniejących zestawów artu gry)
INSERT INTO VisArt_CivilizationBuildingCultures(CivilizationType,BuildingCulture) VALUES
  ('CIVILIZATION_POLAND','BUILDING_CULTURE_NEU'),
  ('CIVILIZATION_POLAND','BUILDING_CULTURE_EEU_ANT'),
  ('CIVILIZATION_POLAND','ANT_STONE'),('CIVILIZATION_POLAND','EXP_STONE');
INSERT INTO VisArt_CivilizationUnitCultures(CivilizationType,UnitCulture)
VALUES('CIVILIZATION_POLAND','Euro');
```

## Narracja przypisana do cywilizacji ✅

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
`Hidden=1` + `StartEveryone=1` + `Activation='AUTO'` = cicha mechanika w tle,
która wyzwala się po spełnieniu wymagań (tu: wypowiedzenie wojny).
