# 06 — Cookbook: nowa cywilizacja

Przepis zrekonstruowany z **działającego moda** `szczupakabra-poland`
(Workshop ID `3768377608`, 53 pliki) ✅ — najlepszy dostępny wzorzec referencyjny.

**Skopiuj ten mod do analizy, zanim zaczniesz swój:**
```
C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660\3768377608
```

## Struktura plików

```
moj-civ\
├── moj-civ.modinfo
├── Core\
│   ├── shared.sql              ← definicja civ, cechy, tradycje, syncretism (wszystkie epoki)
│   ├── age-antiquity.sql       ← zawartość epoki starożytnej
│   ├── age-exploration.sql
│   ├── age-modern.sql
│   ├── effects.xml             ← modyfikatory (GameEffects)
│   ├── unlocks.sql
│   └── narrative-events-*.sql
├── Shell\
│   ├── config.sql              ← widoczność na ekranie wyboru gry
│   └── gameplay-preview.xml    ← podgląd bonusów w menu
├── Art\
│   ├── ArtDefines.sql          ← mapowanie ID → pliki PNG
│   ├── VisualRemaps.xml        ← nowe typy → istniejące modele 3D
│   └── *.png
└── Text\
    ├── ModuleText.xml          ← nazwa moda w menu (LocalizedText)
    ├── MojCivText.xml          ← teksty en_us
    └── MojCivText-pl_PL.xml    ← tłumaczenie
```

## Krok 1 — `.modinfo` z podziałem na epoki i scope

Kluczowa struktura: **6 grup akcji** (game/always, game×3 epoki, remapy, shell).

```xml
<ActionCriteria>
    <Criteria id="always"><AlwaysMet /></Criteria>
    <Criteria id="aq"><AgeInUse>AGE_ANTIQUITY</AgeInUse></Criteria>
    <Criteria id="ex"><AgeInUse>AGE_EXPLORATION</AgeInUse></Criteria>
    <Criteria id="mo"><AgeInUse>AGE_MODERN</AgeInUse></Criteria>
</ActionCriteria>
```

⚠️ **Pamiętaj:** `ImportFiles` z grafiką i `UpdateText` muszą być **powtórzone**
w grupie `shell`, inaczej cywilizacja nie pokaże się poprawnie w menu wyboru gry.

## Krok 2 — `Types` (bez tego nic nie zadziała)

```sql
INSERT INTO Types(Type,Kind) VALUES
('CIVILIZATION_MOJA','KIND_CIVILIZATION'),
('TRAIT_MOJA','KIND_TRAIT'),
('TRAIT_MOJA_UNIKAT','KIND_TRAIT'),
('TRADITION_MOJA_I','KIND_TRADITION');
```

## Krok 3 — cywilizacja i cechy

```sql
INSERT INTO Civilizations(CivilizationType,Adjective,AITargetCityPercentage,ApexAge,
    CapitalName,Description,FullName,Name,RandomCityNameDepth,
    StartingCivilizationLevelType,UniqueCultureProgressionTree)
VALUES('CIVILIZATION_MOJA','LOC_CIVILIZATION_MOJA_ADJECTIVE',50,'AGE_MODERN',
    'LOC_CITY_NAME_MOJA_1','LOC_CIVILIZATION_MOJA_DESCRIPTION',
    'LOC_CIVILIZATION_MOJA_FULLNAME','LOC_CIVILIZATION_MOJA_NAME',10,
    'CIVILIZATION_LEVEL_FULL_CIV','TREE_CIVICS_MO_MOJA');

INSERT INTO Traits(TraitType,Description,InternalOnly,Name) VALUES
('TRAIT_MOJA','LOC_CIVILIZATION_MOJA_DESCRIPTION',1,'LOC_CIVILIZATION_MOJA_NAME');

INSERT INTO CivilizationTraits(CivilizationType,TraitType) VALUES
('CIVILIZATION_MOJA','TRAIT_MODERN_CIV'),
('CIVILIZATION_MOJA','TRAIT_ATTRIBUTE_EXPANSIONIST'),
('CIVILIZATION_MOJA','TRAIT_MOJA');
```

## Krok 4 — nazwy miast, wygląd, ulubiony cud

```sql
INSERT INTO CityNames(CivilizationType,CityName) VALUES
('CIVILIZATION_MOJA','LOC_CITY_NAME_MOJA_1'), ... ;   -- min. kilkanaście

INSERT INTO VisArt_CivilizationBuildingCultures(CivilizationType,BuildingCulture) VALUES
('CIVILIZATION_MOJA','BUILDING_CULTURE_NEU'),('CIVILIZATION_MOJA','ANT_STONE');
INSERT INTO VisArt_CivilizationUnitCultures(CivilizationType,UnitCulture)
VALUES('CIVILIZATION_MOJA','Euro');

INSERT INTO CivilizationFavoredWonders(CivilizationType,FavoredWonderType,FavoredWonderName)
VALUES('CIVILIZATION_MOJA','WONDER_MOJ_CUD','LOC_WONDER_MOJ_CUD_NAME');
```

## Krok 5 — wpięcie w system epok (syncretism)

```sql
-- odblokowanie przez inne cywilizacje / liderów
INSERT INTO CivilizationSyncretismUnlocks(CivilizationType,UnlockCivilizationType)
VALUES('CIVILIZATION_ROME','CIVILIZATION_MOJA');

INSERT INTO LeaderSyncretismUnlocks(LeaderType,UnlockCivilizationType)
SELECT LeaderType,'CIVILIZATION_MOJA' FROM Leaders
WHERE LeaderType IN('LEADER_CHARLEMAGNE','LEADER_CATHERINE');

INSERT INTO LeaderCivPriorities(Civilization,Leader,Priority)
SELECT 'CIVILIZATION_MOJA',LeaderType,3 FROM Leaders WHERE LeaderType IN(...);

-- ⚠️ WYMAGANE, inaczej karta w UI będzie pusta
INSERT INTO LegacyCivilizations(CivilizationType,Name,FullName,Adjective,Age)
VALUES('CIVILIZATION_MOJA','LOC_..._NAME','LOC_..._FULLNAME','LOC_..._ADJECTIVE','AGE_MODERN');
INSERT INTO LegacyCivilizationTraits(CivilizationType,TraitType)
VALUES('CIVILIZATION_MOJA','TRAIT_MOJA');
```

## Krok 6 — ikony i grafika

```sql
INSERT OR IGNORE INTO Icons(ID,Context) VALUES('CIVILIZATION_MOJA','DEFAULT');
INSERT INTO IconDefinitions(ID,Path) VALUES
('ICON_CIVILIZATION_MOJA','fs://game/moj-civ/Art/civ_moja.png'),
('CIVILIZATION_MOJA','fs://game/moj-civ/Art/civ_moja.png'),
('civ_sym_moja','fs://game/moj-civ/Art/civ_moja.png');

INSERT INTO IconDefinitions(ID,Context,Path,IconSize) VALUES
('CIVILIZATION_MOJA','BACKGROUND','fs://game/moj-civ/Art/bg_1080.png',1080),
('CIVILIZATION_MOJA','BACKGROUND','fs://game/moj-civ/Art/bg_720.png',720);
```
Podpięcie przez akcję `<UpdateIcons><Item>Art/ArtDefines.sql</Item></UpdateIcons>`.

⚠️ W `ImportFiles` mod Polska wymienia każdy plik **dwa razy** — z rozszerzeniem
i bez (`Art/civ_poland.png` oraz `Art/civ_poland`). Nie wiem, czy to konieczne,
czy nadmiarowa ostrożność ❓ — patrz [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md).

## Krok 7 — modele 3D bez tworzenia własnych (VisualRemaps) ✅

Najsprytniejszy trik z tego moda: nowe jednostki/budynki **pożyczają modele** istniejących.

```xml
<Database><VisualRemaps>
  <Row><ID>REMAP_MOJ_HUSARZ</ID><DisplayName>LOC_UNIT_MOJ_HUSARZ_NAME</DisplayName>
       <Kind>UNIT</Kind><From>UNIT_MOJ_HUSARZ</From><To>UNIT_HUSSAR</To></Row>
  <Row><ID>REMAP_MOJ_BUDYNEK</ID><DisplayName>LOC_BUILDING_MOJ_NAME</DisplayName>
       <Kind>BUILDING</Kind><From>BUILDING_MOJ</From><To>BUILDING_BANK</To></Row>
</VisualRemaps></Database>
```
`Kind`: `UNIT`, `BUILDING`, `CONSTRUCTIBLE`. Dzięki temu **nie musisz umieć modelować 3D**,
żeby zrobić pełną cywilizację — wystarczą ikony 2D.

## Krok 8 — teksty

```xml
<!-- en_us -->
<Database><EnglishText>
    <Row Tag="LOC_CIVILIZATION_MOJA_NAME"><Text>Moja</Text></Row>
</EnglishText></Database>

<!-- pl_PL -->
<Database><LocalizedText>
    <Row Tag="LOC_CIVILIZATION_MOJA_NAME" Language="pl_PL"><Text>Moja</Text></Row>
</LocalizedText></Database>
```

## Alternatywna organizacja plików (wzorzec społeczności)

Mod Polska grupuje pliki wg **warstwy** (`Core/`, `Shell/`, `Art/`, `Text/`).
Popularna jest też organizacja wg **typu treści** — np. mod Scythia tego samego autora,
który stworzył `civ7-modding-tools`:

```
civmods-izica-civilization-scythia/
├── imports/            zależności zewnętrzne
├── civilizations/
├── traditions/
├── units/
├── constructibles/
├── progression-trees/
└── izica-civilization-scythia.modinfo
```
Obie działają. Przy jednej cywilizacji podział wg typu bywa czytelniejszy.

⚠️ Zamiast pisać XML ręcznie, możesz wygenerować go z TypeScriptu —
patrz [20-typescript-tooling.md](20-typescript-tooling.md). Warto dopiero, gdy
rozumiesz już format docelowy.

## Kolejność pracy (rekomendacja)

1. `.modinfo` + `Types` + `Civilizations` + minimalne teksty → **sprawdź, czy civ pojawia się w menu**
2. Cechy + tradycje → sprawdź w grze
3. Modyfikatory (`effects.xml`) → sprawdź czy bonusy działają
4. Drzewo rozwoju, narracja
5. Grafika i ikony na końcu (najbardziej upierdliwe, najmniej krytyczne)

Iteruj małymi krokami — przy 53 plikach naraz nie znajdziesz, co się zepsuło.
