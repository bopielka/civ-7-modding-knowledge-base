# 07 — Cookbook: a new leader

A recipe based on the **official Firaxis DLC** `DLC\ada-lovelace\modules` ✅ — it is
an ordinary mod with exactly the same structure as a user mod, so it makes for
a reference model of the highest quality.

> None of the 49 Workshop mods added a leader — which is why the DLC is the model.
>
> ❗ **Do not trust the leader guide from the community documentation.** It lists the
> tables `LeaderCivilizations`, `Agendas`, `HistoricalAgendas`, `RandomAgendas`,
> which **do not exist in Civ VII** (those are Civ VI names), and its XML example sets
> a non-existent `Description` column on the `Leaders` row. Verification details:
> [22-source-evaluation.md](22-source-evaluation.md). Stick to the DLC model below.
> The other DLCs with leaders: `napoleon`, `genghis-khan`, `bolivar`, `ashoka-himiko-alt`,
> `friedrich-xerxes-alt`, `lakshmibai`, `edward-teach`, `sayyida-al-hurra`,
> `trung-nhi`, `yi-sun-sin`, `toyotomi-hideyoshi`, `shawnee-tecumseh`.

## Files in a leader module (the ada-lovelace DLC) ✅

```
modules/
├── ada-lovelace.modinfo
├── config/config.xml, metaprogression.xml, unlockableRewards.xml
├── data/
│   ├── leaders.xml                  ← THE CORE: the leader's definition
│   ├── leaders-gameeffects.xml      ← ability modifiers
│   ├── civilizations-shared.xml, civilizations-legacy.xml
│   ├── loading-info.xml, movies.xml, playercolors.xml
│   ├── mementos.xml + mementos-gameeffects.xml
│   ├── metaprogression.xml + -gameeffects.xml
│   ├── narrative-stories*.xml
│   ├── unlocks.xml, unlocks-syncretism.xml
│   └── icons/leader-icons.xml, icons/card-icons.xml
└── (one level up) ada-lovelace.dep  ← the 3D asset package
```

## `leaders.xml` — the complete model ✅

```xml
<?xml version="1.0" encoding="utf-8"?>
<Database>
    <Kinds>
        <InsertOrIgnore Kind="KIND_TRAIT"/>      <!-- safe when it already exists -->
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

### Key observations ✅

- **`InheritFrom="LEADER_DEFAULT"`** — the standard pattern; you do not have to fill in
  all the `Leaders` columns, you inherit the defaults. Firaxis supplies only
  `LeaderType`, `Name`, `IsMajorLeader`, `InheritFrom`.
- Booleans in XML are written as `"true"` (in SQL it would be `1`).
- A leader has **three kinds of traits**: their own ability (`TRAIT_LEADER_*_ABILITY`),
  attributes (`TRAIT_LEADER_ATTRIBUTE_CULTURAL`, `..._SCIENTIFIC`) and victory
  preferences per age (`TRAIT_AQ_SCIENCE_VICTORY`, `TRAIT_EX_*`, `TRAIT_MO_*`).
- Quotes: `TypeQuotes` — separately for leader selection and for victory
  (with `QuoteAudio` pointing at a sound event).

## AI attitude ✅

```xml
<AiListTypes>
    <Row ListType="Ada Lovelace Yield Biases"/>
    <Row ListType="Ada Lovelace Diplomacy Biases"/>
</AiListTypes>
<AiLists>
    <!-- NOTE: the column is called LeaderType, but it holds a TRAIT, not a LEADER -->
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
⚠️ **A naming trap:** `AiLists.LeaderType` holds a **trait type**, not a leader.
Systems: `YieldBiases`, `PseudoYieldBiases`, `DiplomaticGroupBiases`.

## The leader's 3D model — a barrier ❗ ✅ (confirmed)

In the DLC's `.modinfo`:
```xml
<UpdateArt>
    <Item>ada-lovelace-shell</Item>
    <Item>ada-lovelace</Item>
</UpdateArt>
```
These are **not file paths**, but the names of **asset packages** declared in
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

**Practical conclusion:** a full 3D leader model requires a built asset package with GUIDs
and material-library dependencies — that is a product of the Firaxis art pipeline.
The community **has no** public tool for creating a `.dep`.

❓ Open: whether a leader can be made without a model of your own — e.g. via `VisualRemaps`
(it worked in the Poland mod for `UNIT`/`BUILDING`/`CONSTRUCTIBLE`, but `LEADER` did not
show up) or by pointing at another leader's existing art package.
**That is the first thing to test if you are going to make a leader.**

A realistic strategy to start with: a "re-skin" leader — new data, traits and modifiers,
but visually reusing an existing leader.

## The leader's remaining pieces

| File/table | Role |
|---|---|
| `loading-info.xml` → `LoadingInfo_Leaders` | the loading screen |
| `playercolors.xml` (`UpdateColors`) | player colors |
| `icons/leader-icons.xml` (`UpdateIcons`) | icon/portrait |
| `movies.xml` | movies (intro/victory) |
| `mementos.xml` + `-gameeffects.xml` | the leader's mementos |
| `metaprogression.xml` | unlocks in the player profile |
| `unlocks-syncretism.xml` | which civilizations the leader unlocks |
| `config/config.xml` | visibility in the selection menu (scope `shell`) |
