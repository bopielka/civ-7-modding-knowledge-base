# 01 — Game architecture and the mod system

## ❗ Where the game looks for user mods ✅ (empirically confirmed)

```
C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\
```

> ⚠️ **CORRECTION (2026-08-08).** Earlier in this knowledge base (and in the community
> documentation) the path was given as `Documents\My Games\Sid Meier's Civilization VII\Mods\`.
> **That is wrong** — it is the Civ VI convention. A mod placed there is **not detected at all**.
>
> Evidence from `Logs\Modding.log`:
> ```
> Discovering new mods...
> C:/Users/najan/AppData/Local/Firaxis Games/Sid Meier's Civilization VII/Mods/
> Discovered 0 mods.
> ```
> The game prints **exactly one** scan path and it is the one under `AppData\Local`.
> `Documents\My Games\...` contains only `Saves\`.

How to verify this yourself after a game update:
```bash
grep -A2 "Discovering new mods" "$LOCALAPPDATA/Firaxis Games/Sid Meier's Civilization VII/Logs/Modding.log"
```

## Installation layout

```
Sid Meier's Civilization VII\
├── Base\
│   ├── Assets\schema\        ← SQL schemas (definitions of all databases)
│   └── modules\
│       ├── core\             (453 MB) UI framework, fonts, icons, vendor
│       ├── base-standard\    (280 MB) universal rules, gameplay UI
│       ├── age-antiquity\    (186 MB)
│       ├── age-exploration\  (200 MB)
│       └── age-modern\       (171 MB)
└── DLC\<name>\modules\       36 DLC directories
```

✅ Every module has the same structure as a mod: a `.modinfo` file + subfolders
(`data`, `text`, `l10n`, `ui`, `ui-next`, `config`, `scripts`, `maps`, `movies`).
**Which means the game's modules are the best modding documentation that exists.**

## The game's databases

`Base\Assets\schema\` holds several **separate** databases — this matters, because a mod has
to land in the right one:

| Schema | What for |
|---|---|
| `gameplay/01_GameplaySchema.sql` | gameplay — 493 tables (units, buildings, civilizations, modifiers) |
| `frontend/schema-frontend-*.sql` | main menu, game setup, keybindings, hall of fame |
| `modding/schema-modding-10.sql` | the mod system itself (mods, actions, criteria) |
| `localization/schema-loc-*.sql` | texts and languages |
| `icons/IconManager.sql` | icons |
| `colors/ColorManager.sql` | colors |

## Anatomy of a `.modinfo` file

```xml
<?xml version="1.0" encoding="utf-8"?>
<Mod id="my-mod" version="1" xmlns="ModInfo">
    <Properties>
        <Name>LOC_MOD_MY_NAME</Name>           <!-- can be a LOC_ key -->
        <Description>LOC_MOD_MY_DESC</Description>
        <Authors>Your name</Authors>
        <Package>Mod</Package>
        <AffectsSavedGames>0</AffectsSavedGames>
        <ShowInBrowser>1</ShowInBrowser>
    </Properties>
    <Dependencies>   <!-- hard dependencies: without them the mod will not load -->
        <Mod id="base-standard" title="LOC_MODULE_BASE_STANDARD_NAME" />
    </Dependencies>
    <References>     <!-- soft: affects ordering, does not require presence -->
        <Mod id="other-mod" title="LOC_OTHER_NAME" />
    </References>
    <ActionCriteria>
        <Criteria id="always"><AlwaysMet/></Criteria>
    </ActionCriteria>
    <ActionGroups>
        <ActionGroup id="my-mod-game" scope="game" criteria="always">
            <Properties><LoadOrder>1000</LoadOrder></Properties>
            <Actions>
                <UpdateDatabase><Item>data/mine.xml</Item></UpdateDatabase>
            </Actions>
        </ActionGroup>
    </ActionGroups>
    <LocalizedText>   <!-- texts of the modinfo itself (the mod's name in the menu) -->
        <File>text/en_us/ModInfoText.xml</File>
    </LocalizedText>
</Mod>
```

## Scope — where an action applies ✅

Verified across 49 mods: exactly **two** values are in use.

| Scope | When active | Typical use |
|---|---|---|
| `game` | during gameplay | 66 occurrences — rules, in-game UI, data |
| `shell` | main menu / game setup screen | 23 occurrences — configuration, civilization preview, options |

⚠️ A mod that adds a civilization needs **both**: `shell` so the civilization shows up
on the selection screen, `game` so it actually works in play. You can see this plainly
in the Poland mod — the same `ImportFiles`/`UpdateText` repeated in two groups.

## Action types ✅

Counted in the base game + DLC files (first number) and in Workshop mods (second):

| Action | Game/DLC | Mods | What for |
|---|---|---|---|
| `UpdateDatabase` | 330 | 18 | loads XML **or SQL** into the gameplay/shell database |
| `UpdateText` | 120 | 52 | texts and translations |
| `UpdateArt` | 113 | 0 | art definitions (the game uses it, mods hardly ever) |
| `UpdateIcons` | 89 | 14 | icon definitions |
| `UpdateColors` | 33 | 0 | color palettes |
| `UIScripts` | 9 | 63 | **adds** JS scripts to the UI — the main tool of mods |
| `ImportFiles` | 5 | 27 | drops files into the virtual FS; **overrides game files by path** |
| `ReplaceUIScript` | 0 | 3 | replaces a specific UI script |
| `UpdateVisualRemaps` | 2 | 2 | maps new types onto existing 3D models |
| `UIShortcuts` | 4 | 0 | keyboard shortcuts |
| `UIAudioRules` | 2 | 0 | audio rules |
| `LoadOrder` | 2 | 59 | (a group property, not an action) load order |

## Criteria (`ActionCriteria`) ✅

Used in mods: `AlwaysMet` (46), `AgeInUse` (23), `ModInUse` (6).
Also available in the base game: `AgeAtOrBefore`, `ModIsEnabled`, `RuleSetInUse`,
`GameModeInUse`, `ConfigurationValueMatches`, `ConfigurationValueContains`.

```xml
<Criteria id="antiquity-only"><AgeInUse>AGE_ANTIQUITY</AgeInUse></Criteria>
<Criteria id="several" any="true">   <!-- any="true" = OR instead of AND -->
    <AgeInUse>AGE_MODERN</AgeInUse>
    <AgeInUse>AGE_EXPLORATION</AgeInUse>
</Criteria>
```
The `any="true"` attribute corresponds to the `Criteria.Any` column in the modding schema ✅.

## LoadOrder — load order ✅

A property of an `ActionGroup`, not of a mod. Higher number = loaded later = **wins**
on conflict. Real values from the Workshop: from `1` to `130000`, with clear
clusters at `1000` (12 mods), `10000`, `9999`, `100`.

In practice:
- **by default set nothing** — mods without `LoadOrder` load in the base order
- `1000` is the de facto community convention for an "ordinary" UI mod
- very high values (99999+) are mods that deliberately want to override everything else
- ⚠️ for UI decorators the order can be critical — `Controls.decorate` **will not apply
  to already-created component instances** (see [05-ui-javascript.md](05-ui-javascript.md))

## The loading pipeline (mental model) ⚠️

Reconstructed from the modding schema — not from documentation:

1. The game scans mod folders → the `ScannedFiles` table
2. It parses `.modinfo` → `Mods`, `ModProperties`, `ModRelationships`
3. It evaluates criteria (`Criteria`, `Criterion`) for the current context (age, scope)
4. For satisfied criteria it queues `ActionGroups` sorted by `LoadOrder`
5. It executes the group's `Actions` — each one adds rows to the appropriate database
   or registers files in the virtual file system

An important consequence: **mods do not overwrite game files on disk** — everything happens
in the database layer and the virtual FS at startup.
