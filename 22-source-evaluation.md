# 22 — Source reliability and how to verify it

This file exists because, while integrating the community documentation, I ran into
**specific, checkable errors**. Without that caution the knowledge base quickly fills up with
things that do not work.

## The reliability hierarchy

| Level | Source | Why |
|---|---|---|
| 1 (highest) | **The game's SQL schemas** (`Base\Assets\schema\`) | the machine reads exactly this |
| 1 | **The game's logs** (`Logs\Database.log` etc.) | they say what the game actually did |
| 2 | **Game and DLC files** (`Base\modules\`, `DLC\`) | working Firaxis code |
| 3 | **Working Workshop mods** | they work, but may have bad practices |
| 4 | **The community documentation** | helpful, but sometimes wrong — see below |
| 5 | The model's memory / analogies from Civ VI | always verify |

**The rule:** check a level-4 claim at level 1 or 2 before you write it
into the base as ✅.

## Specific errors found in the community documentation ❗

Source: `civ7community.mintlify.app/community/guides/general-creating-leaders`

The leader-creation guide lists tables that **do not exist in Civ VII**:

| Table per the documentation | Reality ✅ |
|---|---|
| `LeaderCivilizations` | **does not exist** (0 hits in the schema) |
| `Agendas` | **does not exist** |
| `HistoricalAgendas` | **does not exist** |
| `RandomAgendas` | **does not exist** |

These are table names from **Civilization VI**. The only tables with "Agenda" in Civ VII are
`DiplomacyAgendaAmountTypes`, `DiplomacyAgendaAwardToTypes`,
`DiplomacyAgendaWeightingTypes` — i.e. an entirely different system.

All existing tables with "Leader" ✅:
`Leaders`, `LeaderCivPriorities`, `LeaderInfo`, `LeaderSyncretismUnlocks`,
`LeaderTraits`, `LegacyLeaderCivPriorities`, `LoadingInfo_Leaders`,
`Resource_RequiredLeaders`

On top of that, the example XML in that guide sets a `Description` attribute on a
`Leaders` row — and the `Leaders` table **has no `Description` column** ✅
(the columns: `LeaderType`, `AITargetCityPercentage`, `BasePersonaType`,
`DesiredNumAlliances`, `DiscountRate`, `InheritFrom`, `IsBarbarianLeader`,
`IsIndependentLeader`, `IsMajorLeader`, `Name`, `OperationList`).

**Conclusion:** that particular example would not work. The correct pattern —
from the official DLC — is in [07-cookbook-new-leader.md](07-cookbook-new-leader.md).

## The likely cause

The documentation gives the impression of being partly AI-generated on the basis of knowledge
about Civ VI (the tell-tale signs: a confident tone, correct structure, wrong details).
That does not make it worthless — **the mechanics descriptions and design hints
are useful**. But every table/column/type identifier has to be checked.

## How to verify in 10 seconds

```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"
S="$G/Base/Assets/schema/gameplay/01_GameplaySchema.sql"

# does the table exist?
grep -c "CREATE TABLE 'LeaderCivilizations'" "$S"     # 0 = it does not exist

# which tables match the topic?
grep -o "CREATE TABLE '[A-Za-z_]*Leader[A-Za-z_]*'" "$S" | sed "s/CREATE TABLE '//;s/'//"

# what columns does the table have?
awk -v w="CREATE TABLE 'Leaders' (" 'index($0,w)==1{f=1} f{print} f&&/^\);/{exit}' "$S"

# does the type/effect exist in the game's data?
grep -rl "EFFECT_UNIT_ADJUST_COMBAT_STRENGTH" "$G/Base/modules/"*/data/
```

## Warning signs in sources

- 🚩 table/column names familiar from Civ VI (`Amenities`, `Loyalty`, `Agendas`, `Eras`)
- 🚩 an XML/SQL example without a pointer to which game file it came from
- 🚩 "probably", "usually", "should" next to technical details
- 🚩 no mention of `scope="shell"` for civilizations (a common real requirement)
- 🚩 the old modifier syntax (`ModifierType`/`CollectionType` instead of
  `collection=`/`effect=`) — see [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)

## What is valuable in the community documentation ✅

Despite the errors in database details, these areas are useful and hard to infer
from the files alone:

- **`civ7-modding-tools`** — a real tool, a verified repository
  ([20-typescript-tooling.md](20-typescript-tooling.md))
- **Gameplay mechanics** — what the game does not have, how ages, crises and
  independent powers work ([21-gameplay-mechanics.md](21-gameplay-mechanics.md))
- **Mod organization patterns** — directory structure, naming conventions
- **Design hints** — what makes sense in this game and what does not

## The rule for future sessions

> **Design and conceptual** knowledge can be taken from the community documentation.
> Check every **identifier** (table, column, type, effect) against the schema
> or the game's files before you mark it ✅.
