# 15 — Referencja schematu (kolumny tabel)

Wygenerowane z `Base\Assets\schema\gameplay\01_GameplaySchema.sql` (493 tabele).
Poniżej **kolumny** najważniejszych z nich dla modowania. Wszystko ✅.

Pełny schemat (z typami, kluczami obcymi, wartościami domyślnymi) czytaj wprost
z pliku schematu — jest czytelny i ma komentarze.

## Jak sprawdzić dowolną inną tabelę

```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"
S="$G/Base/Assets/schema/gameplay/01_GameplaySchema.sql"
T=Traditions   # <- nazwa tabeli
awk -v w="CREATE TABLE '$T' (" 'index($0,w)==1{f=1} f{print} f&&/^\);/{exit}' "$S"
```

---

### `Types`

`Type, Hash, Kind`

### `Kinds`

`Kind, Hash`

### `Civilizations`

`CivilizationType, Adjective, AITargetCityPercentage, ApexAge, CapitalName, Description, FullName, Name, RandomCityNameDepth, StartingCivilizationLevelType, UniqueCultureProgressionTree`

### `CivilizationTraits`

`CivilizationType, TraitType`

### `CivilizationInfo`

`CivilizationType, Header, Caption, SortIndex`

### `CivilizationFavoredWonders`

`CivilizationType, FavoredWonderName, FavoredWonderType`

### `CityNames`

`ID, CityName, CivilizationType, ContinentType, LeaderType, SortIndex`

### `LegacyCivilizations`

`CivilizationType, Adjective, Age, FullName, Name`

### `LegacyCivilizationTraits`

`CivilizationType, TraitType`

### `CivilizationSyncretismUnlocks`

`CivilizationType, UnlockCivilizationType`

### `CivSelfSyncretismUnlocks`

`Age, CivilizationType, UnlockType, Kind, ModifierID`

### `LeaderSyncretismUnlocks`

`LeaderType, UnlockCivilizationType`

### `Leaders`

`LeaderType, AITargetCityPercentage, BasePersonaType, DesiredNumAlliances, DiscountRate, InheritFrom, IsBarbarianLeader, IsIndependentLeader, IsMajorLeader, Name, OperationList`

### `LeaderTraits`

`LeaderType, TraitType`

### `LeaderInfo`

`Header, LeaderType, Caption, SortIndex`

### `LeaderCivPriorities`

`Civilization, Leader, Priority`

### `Traits`

`TraitType, Description, InternalOnly, Name`

### `TraitModifiers`

`ModifierId, TraitType`

### `Units`

`UnitType, AirSlots, AllowBarbarians, AllowEmbarkedDefenseModifiers, AllowTeleportToOtherPlayerCapitals, AntiAirCombat, BaseMoves, BaseSightRange, BuildCharges, CanBeDamaged, CanCapture, CanEarnExperience, CanPurchase, CanRetreatWhenCaptured, CanTargetAir, CanTargetLand, CanTrain, CanTriggerDiscovery, ConstructibleType, CoreClass, CostProgressionModel, Description, Domain, EnabledByReligion, EvangelizeBelief, ExtractsArtifacts, FormationClass, FoundCity, FoundReligion, IgnoreMoves, InitialLevel, LaunchInquisition, Maintenance, MaintenancePercent, MakeTradeRoute, ManualDelete, MustPurchase, Name, NumRandomChoices, Packable, PrereqPopulation, PromotionClass, PseudoYieldType, PurchaseYield, ReligionEvictPercent, ReligiousHealCharges, ReligiousStrength, RequiresInquisition, SpreadCharges, Spy, Stackable, StaticConstructibleUnit, StrategicResource, TeamVisibility, Teleport, Tier, TrackReligion, TraitType, UnitMovementClass, VictoryType, VictoryUnit, WMDCapable, ZoneOfControl`

### `Unit_Stats`

`UnitType, Bombard, Combat, Range, RangedCombat, WMDType`

### `Unit_Costs`

`UnitType, YieldType, Cost`

### `Unit_Abilities`

`UnitAbilityType, UnitType`

### `UnitReplaces`

`CivUniqueUnitType, ReplacesUnitType`

### `UnitUpgrades`

`Unit, UpgradeUnit`

### `UnitPromotions`

`UnitPromotionType, Commendation, Description, Name`

### `UnitPromotionModifiers`

`ModifierId, UnitPromotionType`

### `UnitAbilities`

`UnitAbilityType, AbilityData, AbilityValue, CommandType, DamageAmount, Description, Inactive, KeywordAbilityDuration, KeywordAbilityType, KeywordAbilityValue, Name, OperationType, Permanent, ShareWithChildren, ShowFloatTextWhenEarned`

### `UnitAbilityModifiers`

`ModifierId, UnitAbilityType`

### `Constructibles`

`ConstructibleType, AdjacentDistrict, AdjacentLake, AdjacentRiver, AdjacentTerrain, Age, Archaeology, CanBeHidden, ConstructibleClass, Cost, CostProgressionModel, Defense, Description, Discovery, DistrictDefense, ExistingDistrictOnly, ImmuneDamage, InRailNetwork, IslandSettlement, MilitaryDomain, Name, NoFeature, NoRiver, Population, ProductionBoostOverRoute, Repairable, RequiresAppealPlacement, RequiresDistantLands, RequiresHomeland, RequiresUnlock, RiverPlacement, Tooltip, VictoryItem`

### `Buildings`

`ConstructibleType, AllowsHolyCity, ArchaeologyResearch, BuildQueue, Capital, CapitalForbidden, CitizenSlots, CityCenterPriority, DefenseModifier, GrantFortification, Housing, MaxPlayerInstances, Movable, MultiplePerCity, MustPurchase, OuterDefenseHitPoints, OuterDefenseStrength, Purchasable, PurchaseYield, Town, TraitType, Workable`

### `Improvements`

`ConstructibleType, AdjacentLandTiles, AdjacentSeaResource, AirSlots, BarbarianCamp, BuildInLine, BuildOnFrontier, CanBuildOnNonDistrict, CanBuildOutsideTerritory, CityBuildable, ConstructibleBaseYieldRequired, DefenseModifier, DiscoveryType, DispersalGold, Domain, GrantFortification, Icon, IgnoreNaturalYields, ImprovementOnRemove, MinimumPopulation, MustBeAppealing, OnePerSettlement, RemoveOnEntry, ResourceTier, SameAdjacentValid, TownBuildable, TraitType, UnitBuildable, WeaponSlots, Workable`

### `Wonders`

`ConstructibleType, AdjacentCapital, AdjacentConstructible, AdjacentResource, AdjacentToLand, AdjacentToMountain, BuildOnFrontier, MaxPerPlayer, MaxWorldInstances, MustBeLake, MustNotBeLake, RequiredConstructibleInSettlement, RequiredConstructibleInSettlementCount`

### `Districts`

`DistrictType, AirSlots, AutoPlace, AutoRemove, CanAttack, CaptureRemovesBuildings, CaptureRemovesCityDefenses, CaptureRemovesDistrict, CitizenSlots, CityStrengthModifier, Description, DistrictClass, FreeEmbark, HitPoints, Maintenance, MaxConstructibles, MilitaryDomain, Name, NatureYields, OnePerCity, OverwritePreviousAge, ResourceBlocks, Roads, TravelTime, UrbanCoreType, Water, Workable`

### `Constructible_YieldChanges`

`ConstructibleType, YieldType, YieldChange`

### `Constructible_Adjacencies`

`ConstructibleType, YieldChangeId, Name, RequiresActivation`

### `Constructible_Maintenances`

`ConstructibleType, YieldType, Amount`

### `ConstructibleModifiers`

`ConstructibleType, ModifierId`

### `UniqueQuarters`

`UniqueQuarterType, Description, Name, Tooltip, TraitType`

### `UniqueQuarterModifiers`

`ModifierID, UniqueQuarterType`

### `Traditions`

`TraditionType, AgeType, AllowInitializeAdvancedStart, CultureSlotType, Description, IgnoreInitializeUnlock, IsCrisis, Name, ObsoletesTraditionType, TraitType`

### `TraditionModifiers`

`ModifierId, TraditionType`

### `ProgressionTrees`

`ProgressionTreeType, AgeType, CivInjectedName, CostProgressionModel, IconString, MultipleUnlockName, Name, PrereqFormat, RevealRequirementSetId, SystemType`

### `ProgressionTreeNodes`

`ProgressionTreeNodeType, CanBoost, CanSteal, CivInjectedIcon, CivInjectedName, Cost, Description, IconString, Name, ProgressionTree, Repeatable, RepeatableCostProgressionModel, StartingUnlockDepth, UILayoutColumn, UILayoutRow`

### `ProgressionTreePrereqs`

`Node, PrereqNode`

### `ProgressionTreeNodeUnlocks`

`ProgressionTreeNodeType, TargetType, AIIgnoreUnlockValue, Hidden, IconString, NotTraitType, RequiredGovernmentType, RequiredTraitType, TargetKind, UnlockDepth`

### `ProgressionTreeNodeTraits`

`ProgressionTreeNodeType, RequiredTraitType`

### `Modifiers`

`ModifierId, ModifierType, NewOnly, OwnerRequirementSetId, OwnerStackLimit, Permanent, RunOnce, SubjectRequirementSetId, SubjectStackLimit`

### `DynamicModifiers`

`ModifierType, CollectionType, EffectType`

### `ModifierArguments`

`ModifierId, Name, Extra, SecondExtra, Type, Value`

### `Requirements`

`RequirementId, AiWeighting, BehaviorTree, Impact, Inverse, Likeliness, Persistent, ProgressWeight, RequirementType, Reverse, Triggered`

### `RequirementArguments`

`Name, RequirementId, Extra, SecondExtra, Type, Value`

### `RequirementSets`

`RequirementSetId, RequirementSetType`

### `RequirementSetRequirements`

`RequirementId, RequirementSetId`

### `GameEffects`

`Type, ContextInterfaces, Description, GameCapabilities, SubjectInterfaces, SupportsRemove`

### `GameEffectArguments`

`Name, Type, ArgumentType, DatabaseKind, DefaultValue, Description, MaxValue, MinValue, Required`

### `Adjacency_YieldChanges`

`ID, AdjacentBiome, AdjacentBreathtakingAppeal, AdjacentCharmingAppeal, AdjacentConstructible, AdjacentConstructibleClass, AdjacentConstructibleTag, AdjacentDistrict, AdjacentFeature, AdjacentFeatureClass, AdjacentLake, AdjacentNaturalWonder, AdjacentNavigableRiver, AdjacentQuarter, AdjacentResource, AdjacentResourceClass, AdjacentRiver, AdjacentSeaResource, AdjacentSpecificResource, AdjacentTerrain, AdjacentUniqueQuarter, AdjacentUniqueQuarterType, Age, ProjectMaxYield, Self, TilesRequired, YieldChange, YieldType`

### `Warehouse_YieldChanges`

`ID, Age, BiomeInCity, ConstructibleInCity, DistrictInCity, FeatureClassInCity, FeatureInCity, LakeInCity, MinorRiverInCity, NaturalWonderInCity, NavigableRiverInCity, Overbuilt, ResourceInCity, RouteInCity, TerrainInCity, TerrainTagInCity, YieldChange, YieldType`

### `Yields`

`YieldType, DefaultValue, IconString, Name, OccupiedCityChange`

### `Ages`

`AgeType, Active, AgeTechBackgroundTexture, AgeTechBackgroundTextureOffsetX, ChronologyIndex, Description, EmbarkedUnitStrength, GenerateDiscoveries, GreatPersonBaseCost, HumanPlayersPrimaryHemisphere, MainCultureProgressionTreeType, MainTechProgressionTreeType, Name, NoVictoriesSecondaryHemisphere, NumDefenders, SettlementCountOnTransition, StartingTraditionSlots, TechTreeLayoutMethod, TradeSystemParameterSet`

### `NarrativeStories`

`NarrativeStoryType, Activation, ActivationRequirementSetId, Age, AllowDuplicates, Completion, Cost, CostYield, Crisis, Description, EndTurn, FirstOnly, ForceChoice, ForeignOnly, Hidden, Imperative, IsQuest, Multiplayer, Name, Percentage, PullQuote, Queue, ReducedScaling, RequirementSetId, ResourceReq, ShowProgress, StartEveryone, StoryTitle, Timeout, UIActivation, VariableLinks`

### `NarrativeRewards`

`NarrativeRewardType, ModifierID`

### `NarrativeStory_Rewards`

`NarrativeRewardType, NarrativeStoryType, Activation, BonusEligible, Temporary`

### `VisArt_CivilizationBuildingCultures`

`BuildingCulture, CivilizationType`

### `VisArt_CivilizationUnitCultures`

`CivilizationType, UnitCulture`

### `LoadingInfo_Civilizations`

`CivilizationType, AgeTypeOverride, Audio, BackgroundImageHigh, BackgroundImageLow, CivilizationNameTextOverride, CivilizationText, ForegroundImage, LeaderTypeOverride, MidgroundImage, Subtitle, Tip`

### `LoadingInfo_Leaders`

`LeaderType, AgeTypeOverride, Audio, CivilizationTypeOverride, LeaderImage, LeaderNameTextOverride, LeaderText`

### `TypeQuotes`

`Type, Quote, QuoteAudio, QuoteAuthor`

### `AiLists`

`LeaderType, ListType, MaxDifficulty, MinDifficulty, System`

### `AiListTypes`

`ListType`

### `AiFavoredItems`

`Favored, Item, ListType, StringVal, Value, MaxDifficulty, MinDifficulty, TooltipString`

### `Governments`

`GovernmentType, CelebrationName, Description, Name`

### `GovernmentModifiers`

`GovernmentType, ModifierId`

### `Beliefs`

`BeliefType, AISelectionBin, BeliefClassType, Description, Name, Shareable`

### `BeliefModifiers`

`BeliefType, ModifierID`

### `GreatWorks`

`GreatWorkType, AgeType, Audio, Description, Generic, GreatPersonIndividualType, GreatWorkObjectType, GreatWorkSourceType, Image, Name, Quote`

### `Resources`

`ResourceType, AdjacentToLand, AssignCoastal, AssignInland, BonusResourceSlots, Clumped, HemisphereUnique, IsPendingGenerationUpdate, LakeEligible, MinimumPerHemisphere, Name, NoRiver, RequiresRiver, ResourceClassType, Staple, Tooltip, Tradeable, UnlocksCiv, Weight`

### `Terrains`

`TerrainType, Appeal, DefenseModifier, Hills, Impassable, InfluenceCost, Mountain, MovementCost, Name, SightModifier, SightThroughModifier, Water`

### `Features`

`FeatureType, AddsFreshWater, AllowSettlement, AntiquityPriority, Appeal, AvoidWhenPathfinding, DefenseModifier, Description, FeatureClassType, Impassable, MaximumElevation, MaxLatitude, MinimumElevation, MinLatitude, MovementChange, Name, NoLake, PlacementClass, PlacementDensity, PreventUnpack, Removable, SightThroughModifier, Tooltip`

### `Biomes`

`BiomeType, Description, MaxLatitude, Name`

