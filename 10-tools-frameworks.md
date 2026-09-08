# 10 — Tools and libraries

## What ships with the game (vendor) ✅

`Base\modules\core\vendor\` contains the libraries used by the game's own UI:

- **Solid.js** — `core/vendor/solid-js/` — the reactive framework of the new UI (`ui-next`)
  - `solid-js/web/dist/web.js` (render), `solid-js/dist/solid.js` (core)
- **cohtml** — `core/ui/cohtml.js` — the bridge to the Coherent Labs engine (the UI renderer)

⚠️ For mods this means: if you are adding to `ui-next`, you can use Solid.js by
importing it from `/core/vendor/solid-js/...` — you do not have to bundle your own copy.

## What is NOT there ✅

- ❌ no `.d.ts` files **in the game's files** — zero ready-made type definitions
- ❌ no **official** SDK/ModBuddy from Firaxis (unlike Civ V)
- ❌ no public tool for building `.dep` asset packages
  → **3D models are practically out of the community's reach** (see [07](07-cookbook-new-leader.md))

## ⚠️ CORRECTION: an unofficial SDK does exist after all

I originally wrote here that "there is no SDK". That was wrong — it came from analyzing
local files only, without checking community resources.

**`civ7-modding-tools`** (npm, `github.com/izica/civ7-modding-tools`) is an actively
developed TypeScript framework that generates a mod's files from typed code.
Full description: **[20-typescript-tooling.md](20-typescript-tooling.md)**.

That does not change the fact that the game ships no `.d.ts` — the tool has its own types and constants.

## What you do have — sourcemaps ✅ (the most important tool)

**~1419 `.js.map` files, practically all of which contain the full original
TypeScript code** in the `sourcesContent` field.

```json
{"version":3,"file":"framework.js",
 "sources":["../../../modules/core/ui/framework.ts"],
 "sourcesContent":["/**\n * @file framework.ts\n * @copyright 2021-2024, Firaxis Games\n ..."]}
```

This substitutes for documentation and type definitions. How to use it — see
[16-ui-source-reference.md](16-ui-source-reference.md).

On top of that, **the game's JavaScript is not minified** — readable variable names,
comments, `console.error` with descriptions. You can read it directly.

## The tools that are enough to work with

| Need | Tool |
|---|---|
| Editing XML/SQL/JS | any editor (VS Code) |
| Searching the game's files | `ripgrep` / grep — the most important skill |
| Inspecting the SQLite database | DB Browser for SQLite — open `Mods.sqlite` (see [19](19-workflow-and-debugging.md)) |
| A large mod with a civilization | `civ7-modding-tools` ([20](20-typescript-tooling.md)) — optional |
| TS sources of the game's interface | `..\tools\extract_ts.py` ([16](16-ui-source-reference.md)) |
| 2D graphics (icons) | any PNG editor with alpha |
| 3D models | ⛔ no realistic path — use `VisualRemaps` |

The environment installed on your machine ✅: Node.js v20.19.5, Python 3.14.0 —
both are sufficient (`civ7-modding-tools` requires Node 14+).

## Patterns the community uses (observed) ✅

From an analysis of 49 mods — no build system of any kind. Mods are **raw files**
dropped into a folder:

- no `package.json`, no bundlers, no transpilation
- code written directly in JS (ES modules), not in TS
- files and classes prefixed with the author's initials (`bz-`, `leugi-`, `drongos-`)
  — a simple convention for avoiding name collisions
- CSS as separate files loaded via `ImportFiles` + `Controls.loadStyle`

## A library of patterns worth studying

The best reference mods out of the 49 installed (by complexity and code quality):

| Mod | Workshop ID | What it teaches |
|---|---|---|
| `bz-map-trix` | 3507072814 | advanced UI, map layers, decorators, prototype patching |
| `szczupakabra-poland` | 3768377608 | a full civilization in SQL, VisualRemaps |
| `f1rstdan-cool-ui` | 3510572267 | ✅ **reinstalled 2026-08-26, version 1.9.6, analyzed.** Teaches: `TooltipManager.registerType`, `CityYields.getCityYieldDetails`, a custom button in the production row, `UpdateIcons`, localization in a single `.sql`, a four-layer architecture (DAL/DPL/ULL/URL). ⚠️ It also comes with a warning: its compact row layout has been **broken** since the migration to `ui-next`. Full analysis in `mod-projects/better-city-ui/documentation/04-f1rstdan-cool-ui-analysis.md` |
| `bz-city-hall` | 3507102289 | ✅ **the model for a city-screen mod**: decorators, prototype patches, adding a tab via an attribute, a custom lens layer, `<ActionCriteria>` that switches off a risky action group. Full analysis in [28](28-city-screen.md) and in `mod-projects/better-city-ui/documentation/03-city-hall-analysis.md` |
| `leugi-diploribbon-tweaks` | 3537808797 | `ReplaceUIScript`, lots of artwork |
| `stachs-elegant-policies-and-traditions` | 3730149478 | rewriting the policies screen |
| `drongos-cheat-panel` | 3734207916 | a developer/debug tool |
| `maple-leaves-more-lens` | 3526524592 | map lenses |

Base path: `C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660\<ID>`

The full ID → name mapping is in [11-distribution-and-managers.md](11-distribution-and-managers.md).
