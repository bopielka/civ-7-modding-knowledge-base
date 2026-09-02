# Knowledge base: Civilization VII modding

Notes built up over successive AI-assisted work sessions on Civ VII mods.
The sources of truth are the **installed game files** and **49 working mods from the Steam Workshop**,
not the publisher's documentation (Firaxis has essentially never released any).

## ⚠️ OVERRIDING RULE: the knowledge base is a living document

**Every time something new comes to light while working on a mod,
it has to be written down here.** The knowledge base is not an archive: its value
comes from growing along with experience.

This applies to the AI agent and to the user alike. Detailed procedure:
[24-kb-maintenance.md](24-kb-maintenance.md).

**What to record:**
- something that behaved differently than described here → **fix the entry**
- a ⚠️ or ❓ confirmed in practice → **change it to ✅** and note how it was verified
- a mistake that cost more than a few minutes → **[14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)**
- a new table / effect / API that is missing here → into the appropriate topic file
- the solution to a non-obvious problem → cookbook or quirks

**When:** immediately after establishing the fact, not "later". Later never comes.

## Marking convention

Throughout the knowledge base I use confidence markers — this matters so that future sessions
do not treat guesses as facts:

- ✅ **verified** — checked directly in the game files or in a working mod
- ⚠️ **inferred** — a logical conclusion from a schema/pattern, but not confirmed in practice
- ❓ **open question** — conflicting data or no verification; needs an in-game test

## Where the data comes from

| Source | Path |
|---|---|
| Game installation | `C:\Program Files (x86)\Steam\steamapps\common\Sid Meier's Civilization VII` |
| Community documentation ⚠️ | `civ7community.mintlify.app` — see [22](22-source-evaluation.md) on its reliability |
| TypeScript framework | `github.com/izica/civ7-modding-tools` |
| Workshop mods (49 of them) | `C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660` |
| SQL schemas | `Base\Assets\schema\` |
| Base game modules | `Base\modules\{core,base-standard,age-antiquity,age-exploration,age-modern}` |
| DLC modules | `DLC\<name>\modules\` |
| **Game logs** (debugging) | `C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Logs\` |
| Modding database | `...\Firaxis Games\Sid Meier's Civilization VII\Mods.sqlite` |
| **User mods** ❗ | `C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\` |

## Table of contents

**Foundations**
- [01-architecture.md](01-architecture.md) — installation layout, modules, loading pipeline, scope
- [02-database.md](02-database.md) — the database layer, XML vs SQL, operations, key tables
- [03-modifiers-effects.md](03-modifiers-effects.md) — the GameEffects system: modifiers, collections, requirements
- [04-ages-and-civilizations.md](04-ages-and-civilizations.md) — ages, civilizations, traditions, progression trees
- [05-ui-javascript.md](05-ui-javascript.md) — UI architecture (**the old** `ui/` framework), Controls, decorators, events
- [25-ui-next-solidjs.md](25-ui-next-solidjs.md) — ⚠️ **the second UI framework: `ui-next` on Solid.js**;
  it also covers ✅ **how to build good-looking tooltips** out of the game's components instead of bare `data-tooltip-content`;
  new screens live there and `Controls.decorate` does not touch them

**Cookbooks (step-by-step recipes)**
- [06-cookbook-new-civilization.md](06-cookbook-new-civilization.md) — a new civilization
- [07-cookbook-new-leader.md](07-cookbook-new-leader.md) — a new leader
- [08-cookbook-units-buildings-traditions.md](08-cookbook-units-buildings-traditions.md) — units, buildings, traditions
- [09-cookbook-ui-mod.md](09-cookbook-ui-mod.md) — a mod that modifies the interface

**Process**
- [24-kb-maintenance.md](24-kb-maintenance.md) — ⚠️ **how and when to update this knowledge base**

**Practice**
- [10-tools-frameworks.md](10-tools-frameworks.md) — tools and libraries
- [11-distribution-and-managers.md](11-distribution-and-managers.md) — distribution, Workshop, mod managers
- [12-assets-icons-localization.md](12-assets-icons-localization.md) — graphics, icons, assets
- [23-localization-i18n.md](23-localization-i18n.md) — **localization and i18n**: languages, plurals, gender, case inflection
- [13-references-and-community.md](13-references-and-community.md) — community and external resources
- [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md) — traps, oddities, things that catch you out
- [19-workflow-and-debugging.md](19-workflow-and-debugging.md) — working workflow and debugging

**Maps of specific screens**
- [26-commerce-screen.md](26-commerce-screen.md) — the Commerce screen (resources + trade routes),
  i.e. `screen-resource-allocation` written in `ui-next`

**Reference material**
- [27-resources.md](27-resources.md) — ⚠️ **what each resource gives and under what condition**, per age;
  the "branched bonus" pattern (fish with a port 8 / without a port 4) and why it breaks the naive
  question "is the condition met"
- [15-schema-reference.md](15-schema-reference.md) — columns of the most important tables
- [16-ui-source-reference.md](16-ui-source-reference.md) — how to read the game's UI sources (TypeScript!)
- [17-advanced-and-undocumented.md](17-advanced-and-undocumented.md) — undocumented things
- [18-reference-enumerations.md](18-reference-enumerations.md) — full lists of effects/collections/requirements

**Knowledge from the community documentation**
- [20-typescript-tooling.md](20-typescript-tooling.md) — `civ7-modding-tools`: mods in TypeScript
- [21-gameplay-mechanics.md](21-gameplay-mechanics.md) — how Civ VII works as a game (what it does NOT have!)
- [22-source-evaluation.md](22-source-evaluation.md) — ⚠️ **read before using external sources**

## If you are coming back to mod work — start here

1. **[24-kb-maintenance.md](24-kb-maintenance.md)** — the rule for updating this knowledge base
2. **[19-workflow-and-debugging.md](19-workflow-and-debugging.md)** — logs and the work loop;
   `console.log` does **not** reach `UI.log`, use `console.error`
3. **[14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)** — 31 traps; skim them before you
   start debugging anything "weird"
4. The "Better Specialists UI" mod — sources and project state:
   `../mod-projects/najane-common-specialists-yields/README.md`
   (edit there, deploy with `./deploy.sh` — do **not** edit the game folder, deploy will overwrite it)

**The four things that cost the most time in practice** — worth keeping in the back of your mind:
- the mod "loads", it is ticked in the Mods menu, and its scripts stay silent → check the
  **list of enabled mods in `Modding.log`** just before "Applying mod components".
  Not there? Either it is not enabled (#37), or it has `version="0.x"` in `.modinfo`,
  which yields `Version = 0` and a silent skip (#39)
- the mod "loads" but does nothing → check whether the patched object **already exists**
  at the moment of the patch (lens layers register later than mod scripts)
- the change is visible in the code but not in the game → check whether you are injecting DOM into
  a container that the game is currently **hiding** in that mode
- the `UpdateDatabase` action silently does nothing → a single file in the wrong scope (`game` vs `shell`)
  triggers a **rollback of the whole action**, not just that file

## Numbers worth keeping in your head

- **493** tables in the gameplay database
- **387** effect types (`EFFECT_*`), **38** collections (`COLLECTION_*`), **270** requirement types (`REQUIREMENT_*`)
- **8640** modifiers defined in the base game alone
- **~1400** UI JS files + **~1419** sourcemaps with the **full original TypeScript code**
