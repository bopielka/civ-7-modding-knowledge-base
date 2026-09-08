# 20 — `civ7-modding-tools`: mods in TypeScript

> ⚠️ **A correction to an earlier finding.** In [10-tools-frameworks.md](10-tools-frameworks.md)
> I initially wrote that no SDK for Civ VII exists. That was **wrong** — it came
> from analyzing local files only. The community tool exists and is active.

Source: the community documentation `civ7community.mintlify.app` + the GitHub repository.

## What it is ✅

| | |
|---|---|
| **npm package** | `civ7-modding-tools` |
| **Repository** | `github.com/izica/civ7-modding-tools` |
| **Author** | `izica` — ⚠️ the same one whose `izica-advanced-yield-bar` mod you have installed (Workshop `3512790304`) |
| **Created** | March 2025 |
| **Status** | active development, ~171 commits on `main` |

The author's stated motivation: hand-editing XML while creating a civilization of their own
was too tedious. The tool **generates a mod's XML files** from declarative TypeScript
code — it does not replace the mod system, only the file-writing layer.

## What it gives you compared with hand-written XML

| Aspect | By hand (XML/SQL) | civ7-modding-tools |
|---|---|---|
| Errors | only found when the game starts | **type checking at compile time** |
| IDE support | none | autocompletion, hints |
| Identifiers | typed by hand, typos | constants: `TRAIT.*`, `EFFECT.*`, `COLLECTION.*` |
| Work loop | edit XML → launch → debug | code → build → generated XML |

The biggest real benefit: **constants instead of strings**. Instead of remembering
`EFFECT_UNIT_ADJUST_COMBAT_STRENGTH` you write `EFFECT.UNIT_ADJUST_COMBAT_STRENGTH`
and a typo will not compile.

## Installation

```bash
# required: Node.js 14+ (you have v20.19.5 ✅)
pnpm init
pnpm add civ7-modding-tools typescript ts-node
```
or from the repository:
```bash
git clone https://github.com/izica/civ7-modding-tools
cd civ7-modding-tools
pnpm install
pnpm run build
```

## Project structure

```
my-civ7-mod/
├── src/            the mod's code
├── assets/         icons and assets
├── build.ts        the build script
├── package.json
└── tsconfig.json   (target ES2020, CommonJS modules)
```

Building: `pnpm ts-node build.ts` → generates a ready mod in the output directory.

## A minimal example

```typescript
import { Mod } from 'civ7-modding-tools';

const mod = new Mod({ id: 'test-mod', version: '1' });
mod.build('./dist');
```

## A full example — a civilization with a unique ability

Following the community documentation (the Dacia example):

```typescript
import {
    ACTION_GROUP_BUNDLE, CivilizationBuilder, ImportFileBuilder, Mod,
    ModifierBuilder, TAG_TRAIT, TRAIT, COLLECTION, EFFECT, REQUIREMENT, UNIT_CLASS
} from "civ7-modding-tools";

const mod = new Mod({ id: 'my-civ-mod', version: '1.0' });

const civIcon = new ImportFileBuilder({
    actionGroupBundle: ACTION_GROUP_BUNDLE.AGE_ANTIQUITY,
    content: './assets/civ-icon.png',
    name: 'civ_sym_dacia'
});

const dacia = new CivilizationBuilder({
    actionGroupBundle: ACTION_GROUP_BUNDLE.AGE_ANTIQUITY,
    civilization: {
        domain: 'AntiquityAgeCivilizations',
        civilizationType: 'CIVILIZATION_DACIA'
    },
    civilizationTraits: [
        TRAIT.ANTIQUITY_CIV, TRAIT.ATTRIBUTE_MILITARISTIC, TRAIT.ATTRIBUTE_CULTURAL
    ],
    civilizationTags: [TAG_TRAIT.CULTURAL, TAG_TRAIT.MILITARY],
    icon: { path: `fs://game/${mod.id}/${civIcon.name}` },
    localizations: [{
        name: 'Dacia',
        fullName: 'Kingdom of Dacia',
        adjective: 'Dacian',
        description: '...',
        cityNames: ['Sarmizegetusa', 'Argidava', 'Buridava']
    }],
});

const uprisings = new ModifierBuilder({
    modifier: {
        collection: COLLECTION.PLAYER_UNITS,
        effect: EFFECT.UNIT_ADJUST_COMBAT_STRENGTH,
        permanent: true,
        requirements: [{
            type: REQUIREMENT.UNIT_TAG_MATCHES,
            arguments: [{ name: 'Tag', value: UNIT_CLASS.MELEE }]
        }],
        arguments: [{ name: 'Amount', value: 5 }]
    },
    localizations: [{ description: '+5 Combat Strength for Melee units.' }]
});

dacia.bind([uprisings]);       // attaching the ability to the civilization
mod.add([dacia, civIcon]);
mod.build('./dist');
```

Note how this maps onto the knowledge in [03-modifiers-effects.md](03-modifiers-effects.md):
`collection` + `effect` + `requirements` + `arguments` is **exactly the same structure**
as `<Modifier>` in `<GameEffects>`. The tool does not invent a new model — it wraps the existing one.

## The tool's architecture

**Builders** (extending `BaseBuilder`): `CivilizationBuilder`, `UnitBuilder`,
`ConstructibleBuilder`, `ModifierBuilder`, `ImportFileBuilder`.
Their jobs: create nodes, bind entities (`bind`), return files to include.

**Nodes** (extending `BaseNode`, with a `toXmlElement()` method):
`DatabaseNode` (a whole XML file), `TypeNode`, `UnitNode`, `CivilizationNode`,
`CivilizationTraitNode`.

**Files**: `XmlFile` and `ImportFile` — each with `path`, `content`, `actionGroups`,
`actionGroupActions` (i.e. the tool generates the `.modinfo` itself).

**Constants**: `UNIT_CLASS`, `CONSTRUCTIBLE_TYPE_TAG`, `ACTION_GROUP`, `EFFECT`, `TRAIT`,
`COLLECTION`, `REQUIREMENT`, `TAG_TRAIT`, `ACTION_GROUP_BUNDLE`.

⚠️ `ACTION_GROUP_BUNDLE` is a concept of the tool, not of the game — it packages action groups
(e.g. "everything for the Antiquity age") instead of writing them by hand in `.modinfo`.

## Coverage (as declared by the author)

✅ done: modinfo, localization, units, civilizations, constructibles,
city names, civics, traditions, game effects
🚧 in progress: Great People nodes
📋 planned: AI nodes, unit abilities, wonders

⚠️ Which means **the tool does not cover everything**. For unsupported elements:
- the low-level node API (`UnitNode`, `DatabaseNode`, `XmlFile`)
- your own builders extending the base class
- or simply adding raw XML/SQL alongside

## Should you use it?

**For:** typing, autocompletion, fewer typos, good for a large mod (a civilization).

**Against:** an extra layer of abstraction; when something does not work, you debug
the **generated XML**, so you still have to understand the format from files
[02](02-database.md)–[04](04-ages-and-civilizations.md). It does not cover UI mods
(those are written in JS anyway — see [09](09-cookbook-ui-mod.md)).

**Recommendation:** make your first, small mod by hand, to understand the format.
For a large mod with a civilization — consider this tool.
For UI mods — not useful.
