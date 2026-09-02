# 13 — The community and external resources

> ⚠️ This file contains the most **unverified** material — I did not check
> these sources online during the analysis. Treat it as hints on where to look,
> not as confirmed facts.

## The main places

- **The community documentation** — `civ7community.mintlify.app` ✅ (checked)
  A large site with guides: architecture, creating civilizations/leaders,
  TypeScript tooling, gameplay mechanics, an implementation example (Dacia).
  ⚠️ **It contains errors in the database details** — see
  [22-source-evaluation.md](22-source-evaluation.md). Excellent for conceptual
  and design knowledge, poor as a source of table names.
  The most valuable pages:
  `/community/guides/typescript/*` (tooling), `/community/reference/gameplay-mechanics`,
  `/community/guides/mod-patterns`, `/community/guides/examples/*` (Dacia).
- **`civ7-modding-tools`** — `github.com/izica/civ7-modding-tools` ✅
  A TypeScript framework for generating mods ([20](20-typescript-tooling.md)).
- **CivFanatics** — `forums.civfanatics.com`, the Civ VII resources section.
  ✅ Confirmed indirectly: Workshop mods give a `<URL>` there, e.g.
  `https://forums.civfanatics.com/resources/map-trix.31950/`.
  Which means authors really do publish and discuss there.
- **Steam Workshop** — the main distribution channel (AppID `1295660`).
- **A Civ VII modders' Discord** — ⚠️ it exists according to general knowledge about the Civ
  community, but I have no verified invite.

## Authors whose mods are worth following ✅

Out of the 49 installed mods, a few active and competent authors stand out:

| Author / prefix | Mods | Specialty |
|---|---|---|
| `beezany` (`bz-`) | map-trix, city-hall, flag-corps, friends, ready-or-not, a-la-mods, clean-slate | 7 mods — the most advanced UI code |
| `leugi` | diploribbon-tweaks, diploicon-tweaks, happiness_stage_icons | diplomacy graphics and UI |
| `drongos` | cheat-panel, top-panel, relationship-preview | tools and panels |
| `orions` | bonus-icons-plus, clearer-agendas, victory-meter | iconography |
| `stachs`/`stachus` | elegant-policies-and-traditions, elegant-great-works | rewriting screens |
| `szczupakabra` | poland | a full civilization |
| `nasuellia` | non-sticky-selection, unit-flags | UX |

**Practical advice:** instead of hunting for tutorials, read these mods' code locally.
You have them all on disk (see [11](11-distribution-and-managers.md)).

## Why the official documentation will not help ✅

Firaxis has not released an SDK or modding documentation for Civ VII (as of the time of
this analysis). There is no:
- ModBuddy-style tool (as in Civ V)
- `.d.ts` type definitions for the UI API
- description of the database tables

**But the game itself is the documentation:**
1. `Base\Assets\schema\` — full schemas with comments
2. `Base\modules\` + `DLC\` — hundreds of working examples from Firaxis
3. sourcemaps with the TypeScript code ([16](16-ui-source-reference.md))

That is better than most official documentation — you just have to know how to search.

## Skill #1: searching the game's files

```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"

# where a given effect/type is defined
grep -rl "EFFECT_CITY_ADJUST_YIELD" "$G/Base/modules/"*/data/

# how Firaxis built a similar mechanic
grep -rn "REQUIREMENT_PLAYER_HAS_CIVILIZATION_OR_LEADER_TRAIT" "$G/Base/modules/age-antiquity/data/" | head

# which UI components can be decorated
grep -rn "Controls.define" "$G/Base/modules/base-standard/ui/" | head -40
```

⚠️ Do **not** `grep -r` the whole game directory — it is 1.3 GB and takes minutes.
Narrow it down to `*/data/` (14 MB) or `*/ui/`.

## Open questions to research online

- ❓ Is there a public tool for building `.dep` packages (3D models)?
- ❓ Is there an equivalent of FireTuner / Live Tuner from Civ VI?
  (the Civ VI SDK included one — whether the Civ VII SDK does too is to be checked after installing it)
- ~~What the Workshop publishing process looks like~~ → ✅ **established**: a separate Mod SDK
  from the Tools section in Steam, see [11-distribution-and-managers.md](11-distribution-and-managers.md)
- ~~Where the game writes its logs~~ → ✅ **found**, see [19](19-workflow-and-debugging.md):
  `AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Logs\`
