# 07 — Cookbook: nowy lider

Przepis oparty na **oficjalnym DLC Firaxis** `DLC\ada-lovelace\modules` ✅ — to jest
zwykły mod o dokładnie tej samej strukturze co mod użytkownika, więc stanowi
wzorzec referencyjny najwyższej jakości.

> Żaden z 49 modów Workshop nie dodawał lidera — dlatego wzorcem jest DLC.
>
> ❗ **Nie ufaj przewodnikowi o liderach z dokumentacji społeczności.** Wymienia on
> tabele `LeaderCivilizations`, `Agendas`, `HistoricalAgendas`, `RandomAgendas`,
> które **nie istnieją w Civ VII** (to nazwy z Civ VI), a jego przykład XML ustawia
> na wierszu `Leaders` nieistniejącą kolumnę `Description`. Szczegóły weryfikacji:
> [22-source-evaluation.md](22-source-evaluation.md). Trzymaj się wzorca z DLC poniżej.
> Pozostałe DLC z liderami: `napoleon`, `genghis-khan`, `bolivar`, `ashoka-himiko-alt`,
> `friedrich-xerxes-alt`, `lakshmibai`, `edward-teach`, `sayyida-al-hurra`,
> `trung-nhi`, `yi-sun-sin`, `toyotomi-hideyoshi`, `shawnee-tecumseh`.

## Pliki w module lidera (DLC ada-lovelace) ✅

```
modules/
├── ada-lovelace.modinfo
├── config/config.xml, metaprogression.xml, unlockableRewards.xml
├── data/
│   ├── leaders.xml                  ← RDZEŃ: definicja lidera
│   ├── leaders-gameeffects.xml      ← modyfikatory zdolności
│   ├── civilizations-shared.xml, civilizations-legacy.xml
│   ├── loading-info.xml, movies.xml, playercolors.xml
│   ├── mementos.xml + mementos-gameeffects.xml
│   ├── metaprogression.xml + -gameeffects.xml
│   ├── narrative-stories*.xml
│   ├── unlocks.xml, unlocks-syncretism.xml
│   └── icons/leader-icons.xml, icons/card-icons.xml
└── (poziom wyżej) ada-lovelace.dep  ← paczka zasobów 3D
```

## `leaders.xml` — kompletny wzorzec ✅

```xml
<?xml version="1.0" encoding="utf-8"?>
<Database>
    <Kinds>
        <InsertOrIgnore Kind="KIND_TRAIT"/>      <!-- bezpieczne, gdy już istnieje -->
        <InsertOrIgnore Kind="KIND_VICTORY"/>
    </Kinds>
    <Types>
        <Row Type="LEADER_ADA_LOVELACE" Kind="KIND_LEADER"/>
        <Row Type="TRAIT_LEADER_ADA_LOVELACE_ABILITY" Kind="KIND_TRAIT"/>
        <Row Type="VICTORY_LEADER_ADA_LOVELACE" Kind="KIND_VICTORY"/>
    </Types>

    <Leaders>
        <Row LeaderType="LEADER_ADA_LOVELACE" Name="LOC_LEADER_ADA_LOVELACE_NAME"
             IsMajorLeader="true" InheritFrom="LEADER_DEFAULT" />
    </Leaders>

    <TypeQuotes>
        <Row Type="LEADER_ADA_LOVELACE" Quote="LOC_MAIN_CHAR_SELECT_LEADER_ADA_LOVELACE_ANY"/>
        <Row Type="VICTORY_LEADER_ADA_LOVELACE" Quote="LOC_VICTORY_LEADER_ADA_LOVELACE"
             QuoteAudio="play_victory_ada"/>
    </TypeQuotes>

    <Traits>
        <Row TraitType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" InternalOnly="true"
             Name="LOC_..._NAME" Description="LOC_..._DESCRIPTION"/>
    </Traits>

    <LeaderTraits>
        <Row LeaderType="LEADER_ADA_LOVELACE" TraitType="TRAIT_LEADER_ADA_LOVELACE_ABILITY"/>
        <Row LeaderType="LEADER_ADA_LOVELACE" TraitType="TRAIT_LEADER_ATTRIBUTE_CULTURAL"/>
        <Row LeaderType="LEADER_ADA_LOVELACE" TraitType="TRAIT_LEADER_ATTRIBUTE_SCIENTIFIC"/>
        <Row LeaderType="LEADER_ADA_LOVELACE" TraitType="TRAIT_AQ_SCIENCE_VICTORY"/>
        <Row LeaderType="LEADER_ADA_LOVELACE" TraitType="TRAIT_EX_SCIENCE_VICTORY"/>
        <Row LeaderType="LEADER_ADA_LOVELACE" TraitType="TRAIT_MO_CULTURE_VICTORY"/>
    </LeaderTraits>

    <TraitModifiers>
        <Row TraitType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" ModifierId="ADA_LOVELACE_MOD_TECH_MASTERY"/>
        <Row TraitType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" ModifierId="ADA_LOVELACE_MOD_CIVIC_MASTERY"/>
    </TraitModifiers>
</Database>
```

### Kluczowe obserwacje ✅

- **`InheritFrom="LEADER_DEFAULT"`** — standardowy wzorzec; nie trzeba wypełniać
  wszystkich kolumn `Leaders`, dziedziczysz domyślne. Firaxis podaje tylko
  `LeaderType`, `Name`, `IsMajorLeader`, `InheritFrom`.
- Boolean w XML zapisywany jako `"true"` (w SQL byłoby `1`).
- Lider ma **trzy rodzaje cech**: własną zdolność (`TRAIT_LEADER_*_ABILITY`),
  atrybuty (`TRAIT_LEADER_ATTRIBUTE_CULTURAL`, `..._SCIENTIFIC`) i preferencje
  zwycięstwa per epoka (`TRAIT_AQ_SCIENCE_VICTORY`, `TRAIT_EX_*`, `TRAIT_MO_*`).
- Cytaty: `TypeQuotes` — osobno przy wyborze lidera i przy zwycięstwie
  (z `QuoteAudio` wskazującym zdarzenie dźwiękowe).

## Nastawienie AI ✅

```xml
<AiListTypes>
    <Row ListType="Ada Lovelace Yield Biases"/>
    <Row ListType="Ada Lovelace Diplomacy Biases"/>
</AiListTypes>
<AiLists>
    <!-- UWAGA: kolumna nazywa się LeaderType, ale trzyma TRAIT, nie LEADER -->
    <Row ListType="Ada Lovelace Yield Biases"
         LeaderType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" System="YieldBiases"/>
    <Row ListType="Ada Lovelace Pseudoyield Biases"
         LeaderType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" System="PseudoYieldBiases"/>
    <Row ListType="Ada Lovelace Diplomacy Biases"
         LeaderType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" System="DiplomaticGroupBiases"/>
</AiLists>
<AiFavoredItems>
    <Row ListType="Ada Lovelace Yield Biases" Item="YIELD_SCIENCE" Value="50"/>
    <Row ListType="Ada Lovelace Yield Biases" Item="YIELD_CULTURE" Value="25"/>
    <Row ListType="Ada Lovelace Pseudoyield Biases" Item="PSEUDOYIELD_TECH_MASTERY" Value="50"/>
    <Row ListType="Ada Lovelace Diplomacy Biases" Item="DIPLOMACY_ACTION_BECOME_SUZERAIN" Value="50"/>
</AiFavoredItems>
```
⚠️ **Pułapka nazewnicza:** `AiLists.LeaderType` przechowuje **typ cechy**, nie lidera.
Systemy: `YieldBiases`, `PseudoYieldBiases`, `DiplomaticGroupBiases`.

## Model 3D lidera — bariera ❗ ✅ (potwierdzone)

W `.modinfo` DLC:
```xml
<UpdateArt>
    <Item>ada-lovelace-shell</Item>
    <Item>ada-lovelace</Item>
</UpdateArt>
```
To **nie są ścieżki plików**, tylko nazwy **paczek zasobów** zadeklarowanych w
`DLC\ada-lovelace\ada-lovelace.dep`:

```xml
<AssetObjects..GameDependencyData>
    <ID><name text="ada-lovelace"/><id text="580e04ba-6bfd-4ffd-b5e6-9bb68e70d371"/></ID>
    <RequiredGameArtIDs>
        <Element><name text="Civ7"/><id text="F5D94984-..."/></Element>
    </RequiredGameArtIDs>
    <LibraryDependencies>...</LibraryDependencies>
</AssetObjects..GameDependencyData>
```

**Wniosek praktyczny:** pełny model 3D lidera wymaga zbudowanej paczki zasobów z GUID-ami
i zależnościami bibliotek materiałów — to produkt pipeline'u artystycznego Firaxis.
Społeczność **nie ma** publicznego narzędzia do tworzenia `.dep`.

❓ Otwarte: czy da się zrobić lidera bez własnego modelu — np. przez `VisualRemaps`
(w modzie Polska działało dla `UNIT`/`BUILDING`/`CONSTRUCTIBLE`, ale `LEADER` się nie
pojawił) albo przez wskazanie istniejącej paczki artu innego lidera.
**To jest pierwsza rzecz do przetestowania, jeśli będziesz robić lidera.**

Realistyczna strategia na start: lider „re-skin" — nowe dane, cechy i modyfikatory,
ale wizualnie korzystający z istniejącego lidera.

## Pozostałe elementy lidera

| Plik/tabela | Rola |
|---|---|
| `loading-info.xml` → `LoadingInfo_Leaders` | ekran ładowania |
| `playercolors.xml` (`UpdateColors`) | kolory gracza |
| `icons/leader-icons.xml` (`UpdateIcons`) | ikona/portret |
| `movies.xml` | filmy (intro/zwycięstwo) |
| `mementos.xml` + `-gameeffects.xml` | pamiątki lidera |
| `metaprogression.xml` | odblokowania w profilu gracza |
| `unlocks-syncretism.xml` | które cywilizacje lider odblokowuje |
| `config/config.xml` | widoczność w menu wyboru (scope `shell`) |
