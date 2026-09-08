# 17 — Advanced and undocumented things

Things you will not find in any guide, because they come from reading the game's files.

## 1. Custom properties in `.modinfo` ✅

The `bz-map-trix` mod declares non-standard tags:
```xml
<Properties>
    <bzIcon>blp:ntf_choosenarrative_blk</bzIcon>
    <bzIconGlow>#e5d2ac</bzIconGlow>
    <bzIconScale>...</bzIconScale>
    <bzIconCrop>...</bzIconCrop>
    <Teaser>...</Teaser>
    <LastUpdated>...</LastUpdated>
</Properties>
```
It works because the modding schema stores properties as **name/value pairs**
(`ModProperties(ModRowId, Name, Value)`) — the parser does not validate against a list of allowed names ✅.

The consequence: **a mod can read other mods' own metadata** and build an
ecosystem on it (bz-map-trix has its own icon system for the `bz-` family of mods).

## 2. `ModProperties` as a communication channel between mods ⚠️

Since properties end up in the modding database, mod A can query mod B's properties.
I have not verified the access API from JS, but the mechanism exists in the schema.

## 3. A mod can create its own tables ✅

```sql
CREATE TABLE IF NOT EXISTS CivsWithoutBackgrounds(
    CivilizationType TEXT PRIMARY KEY, ArtPath TEXT NOT NULL);
INSERT OR REPLACE INTO CivsWithoutBackgrounds VALUES('CIVILIZATION_POLAND','fs://...');
```
The Poland mod does this — probably so that another mod (`custom-civ-art-fixes`?)
can read it. That is de facto an **inter-mod protocol**.

## 4. `ActionGroupRelationships` ⚠️

The modding schema has a table linking the action groups of different mods:
```sql
ActionGroupRelationships(ActionGroupRowId, OtherModId, OtherActionGroupId, Relationship)
```
This suggests it is possible to declare relationships at the level of a **single action group**,
rather than the whole mod. I have not found the XML syntax that populates it — ❓ to be investigated.

## 5. `ModCompatibilityWhitelist` ✅ (schema)

```sql
ModCompatibilityWhitelist(ModRowId, GameVersion)
```
The game tracks mods for which version-incompatibility warnings should be skipped.
❓ I do not know whether this can be set from `.modinfo`.

## 6. `EpicMods` ✅ (schema)

A separate table for mods from the Epic Games Store version — relevant only if a mod is meant to work
on both platforms.

## 7. `Migrations` ✅ (schema)

```sql
Migrations(SQL, MinVersion, MaxVersion, SortIndex)
```
A system for migrating the database when the game updates — a Firaxis mechanism, but it shows that
the modding database is versioned (`PRAGMA user_version(5)`).

## 8. Configuration criteria ✅ (in the base game)

Besides `AgeInUse`/`ModInUse` there are:
```xml
<ConfigurationValueMatches>
    <ConfigurationId>...</ConfigurationId>
    <Group>...</Group>
    <Value>...</Value>
</ConfigurationValueMatches>
<ConfigurationValueContains>...</ConfigurationValueContains>
<RuleSetInUse>...</RuleSetInUse>
<GameModeInUse>...</GameModeInUse>
<AgeAtOrBefore>AGE_MODERN</AgeAtOrBefore>
<ModIsEnabled>...</ModIsEnabled>
```
None of the 49 Workshop mods used them — an open field, e.g. a mod active only
with a particular game setting.

## 9. An inverted criterion ⚠️

The schema: `Criterion(CriterionRowId, CriteriaRowId, CriterionType, Inverse)`.
The `Inverse` column suggests it is possible to negate a criterion (e.g. "when a mod is NOT present").
❓ I have not found the XML attribute that sets it — probably `inverse="true"`
on the criterion element. **Worth testing.**

## 10. `blp:` — the game's built-in assets ✅

```sql
('UNIT_POLAND_HUSSAR','PORTRAIT_MASK','blp:unitflag_hussar',0)
```
```xml
<bzIcon>blp:ntf_choosenarrative_blk</bzIcon>
```
Lets you use the game's artwork without shipping your own. The names have to be dug out of the game's files:
```bash
grep -rho "blp:[a-z0-9_]*" "$G/Base/modules/"*/data/ | sort -u | head -50
```

## 11. `InsertOrIgnore` on `Kinds` ✅ — the Firaxis pattern

```xml
<Kinds>
    <InsertOrIgnore Kind="KIND_TRAIT"/>
    <InsertOrIgnore Kind="KIND_VICTORY"/>
</Kinds>
```
Declares the `Kind`s you need without blowing up if they already exist. **Copy this pattern**
— it makes a mod immune to load order.

## 12. `InheritFrom` in `Leaders` ✅

`InheritFrom="LEADER_DEFAULT"` lets you define a leader with **four columns**
instead of eleven. The foreign key points at `Leaders` — so you can also inherit
from any existing leader.
⚠️ Check whether `Civilizations` has an analogous mechanism — it has no `InheritFrom` column,
so probably not.

## 13. `Modifiers` — stacking control flags ✅

```xml
<Modifier id="..." collection="..." effect="..."
          permanent="true" run-once="true" new-only="true"
          owner-stack-limit="1" subject-stack-limit="1">
```
`owner-stack-limit` / `subject-stack-limit` prevent the same bonus from stacking
repeatedly — rarely used by modders, and they solve real balance bugs.

## 14. `Requirements` has AI columns ✅

```
AiWeighting, BehaviorTree, Impact, Likeliness, ProgressWeight, Persistent, Triggered, Reverse
```
Beyond the condition's logic, a requirement carries **hints for the AI** about how important that condition is.
Modders usually skip this — and it affects the computer's behavior.

## 15. The `ui-next` layer is mid-migration ⚠️

The proportions (core: 373 old vs 186 new; base-standard: 601 vs 132) suggest that Firaxis
is gradually rewriting the UI in Solid.js. **A risk for mods**: a component you decorate
today in `ui/` may move to `ui-next/` in the next patch.
The `bz-map-trix` mod hedges by having files in both trees.

## Things to test

- [ ] `inverse="true"` on a criterion
- [ ] `VisualRemaps` with `Kind=LEADER`
- [ ] whether the extensionless asset file in `ImportFiles` is required
- [ ] whether the `fs://game/` alias can differ from `<Mod id>`
- [ ] `ActionGroupRelationships` — the XML syntax
- [ ] reading another mod's `ModProperties` from JS
- [ ] whether returning to the main menu reloads mods without restarting the game
      (the `Reason: Main Menu Reset` entries in `Modding.log` suggest so)
- [ ] `ACTION_GROUP_BUNDLE` from `civ7-modding-tools` — what `.modinfo` it actually generates

## Community documentation pages not yet reviewed

From `civ7community.mintlify.app` I have reviewed: documentation-guide, modding-architecture,
mod-patterns, typescript-overview, typescript-technical, environment-setup,
howto/creating-civilizations, howto/advanced-techniques, general-creating-leaders,
reference/modding-reference, reference/gameplay-mechanics, ages/age-gameplay-mechanics.

Left over (worth a look for a specific task):
- `guides/getting-started`, `guides/general-modifying-content`
- `guides/database-schemas`, `guides/base-standard-module`
- `guides/ages/age-modules`, `guides/ages/age-architecture`
- `guides/general-creating-civilizations`
- `guides/typescript/howto/`: creating-units, creating-buildings, leaders-and-ages,
  unique-quarters, traditions, progression-trees, modifiers-and-effects, assets-and-icons
- `guides/examples/dacia-*` (4 pages — a full example of implementing a civilization)
- `reference/file-paths-reference`, `reference/modding-guide-civs-leaders`

⚠️ With every one of them remember the rule from [22-source-evaluation.md](22-source-evaluation.md):
verify identifiers against the game's schema.


---

## `WorldUI` — drawing on the 3D MAP (not on the DOM) ✅

Discovered while diagnosing the Holistic QoL+ mod. It is a world apart from the DOM and a separate source
of "artifacts on the screen", because none of it is an HTML element.

```js
const group = WorldUI.createOverlayGroup(NAME, OVERLAY_PRIORITY.PLOT_HIGHLIGHT, {x:1,y:1,z:1});
const plots   = group.addPlotOverlay();      // hex fills
const borders = group.addBorderOverlay({ style: 'MovementRange', primaryColor, secondaryColor });
const marks   = group.addLandmarkOverlay();  // markers on constructions

plots.addPlots([plotIndex], { fillColor });
borders.setPlotGroups(plotIndexes, 1);
borders.setGroupStyle(1, { style: 'CombatBorder', primaryColor, secondaryColor });

// billboards with text/an icon above a hex
const sprites = WorldUI.createSpriteGrid(NAME, SpriteMode.Billboard);

group.setVisible(true);
plots.clear(); borders.clear(); sprites.clear();
```

Border styles seen in the code: `MovementRange`, `CombatBorder`.
Priorities: `OVERLAY_PRIORITY.PLOT_HIGHLIGHT`, `OVERLAY_PRIORITY.UNIT_COMBAT`.

⚠️ **Nothing cleans this up for you.** An overlay survives a change of screen, of turn and of interface
mode — until someone calls `clear()`. The typical mistake: a refresh hooked to a narrow
list of events (`UnitSelectionChanged`, `UnitMoved`, `interface-mode-changed`) while the state
changes on an event outside that list → hexes or labels stay on the map.

⚠️ **The order of `ensure` and the "is it enabled" check.** In Holistic QoL+
`ensureCityLandmarkMarkersOverlay()` runs BEFORE `if (!enabled) return` and calls
`setVisible(true)`. The effect: turning the feature off in the options does not remove the overlay group, it just
leaves it empty and visible. The check has to come first.

## An animated `box-shadow` / `filter` in CSS — a candidate for smearing ❗

The same mod animates `box-shadow` in an infinite loop (`animation: … 1.8s infinite`) and
`filter: brightness()`. The glow extends **beyond the element's box**, and this renderer does
not always invalidate the drawing region widely enough — hence the streaks and ghosts left behind the element.

When diagnosing artifacts on a player's machine: this is the first thing to switch off, because it is cheap
to check (one "limit animations" toggle, if the mod has one) and does not require
disabling the whole feature.

---

## ⚠️ Notifications are drawn by TWO of them, from two different sources ❗✅

This is the most important thing to remember, because the first attempt at "hide the notification" did
nothing visible.

| who draws it | where it gets it from | which notifications |
|---|---|---|
| `panel-notification-train` (the bar) | `NotificationModel` | those that do **NOT** block the turn (`isSoftNotification`) |
| `panel-action` — **the icon in the ring** | `Game.Notifications.getIdsForPlayer` → `getNotificationInfo` | turn-blocking ones, **except** the current blocker |
| `panel-action` — **THE MAIN BUTTON** | `Game.Notifications.findEndTurnBlocking` | **the current end-of-turn blocker** |

⚠️ **The third row is the trap that cost several rounds.** The same notification type
travels between the second and third track: while it blocks something else, it is an icon in the ring;
once the player has done everything else, it **is promoted to the main button** — and there the id comes
from `findEndTurnBlocking`, so **no filter on `getNotificationInfo` will reach it**.
Symptom: "I hid it and it still shows up, but only when I have nothing else to do".

Checking which track you are in:

```js
const type = Game.Notifications.getEndTurnBlockingType(playerID);
const id   = Game.Notifications.findEndTurnBlocking(playerID, type);
const isThisOne = id && Game.Notifications.getType(id) === Game.getHash('NOTIFICATION_...');
```

Checking which group a type belongs to: `Game.Notifications.getBlocksTurnAdvancement(id)`.

`NOTIFICATION_ASSIGN_NEW_RESOURCES` **blocks**, so the ring draws it and the bar
ignores it. Hiding it in `NotificationModel` (via a handler) removes it from a list it
was never in anyway — nothing disappears from the screen.

### ⚠️ Dismissing the blocker does NOT work — the engine recreates it

`Game.Notifications.dismiss(id)` executes and is accepted, but the notification comes back
within a second: its row in `notification.xml` has **`AutoNotify="True"`**, so the engine
raises it again for as long as the condition holds. For `NOTIFICATION_ASSIGN_NEW_RESOURCES`
the condition is "you have unassigned resources" — so it cannot be removed from the UI.

### How to really silence the end-of-turn blocker ✅

The block is **in the UI, not in the engine** — `canEndTurn()` is a panel method reading
`getEndTurnBlockingType`, and ending the turn is `GameContext.sendTurnComplete()`. So the answer can
be swapped for the duration of a single call:

```js
function withoutOurBlocker(body) {
    const real = Game.Notifications.getEndTurnBlockingType;
    try {
        Game.Notifications.getEndTurnBlockingType = () => EndTurnBlockingTypes.NONE;
        return body();
    } finally {
        Game.Notifications.getEndTurnBlockingType = real;
    }
}
```

⚠️ **You have to wrap TWO `PanelAction` methods, not one.** The panel asks the same question
twice, for two purposes:

| method | what it is responsible for |
|---|---|
| `refreshActionButton` | how the button **looks** |
| `tryEndTurn` | what the click **does** — `canEndTurn` reads the blocker again and, on a hit, calls `activateBlockingNotification()` instead of ending the turn |

Wrapping only the first gives you a button labeled "End Turn" which, when clicked,
opens the blocker's screen. A misleading symptom, because it looks like a drawing bug.

⚠️ `Game.Notifications` is an engine object — after substituting, check that it really returns
the substituted value, and back off if it does not.

Not drawing the blocker is the worst possible outcome: the block remains (it is on the engine's
side) and its explanation disappears — the player clicks "End Turn" and nothing happens.

The right tool is `Game.Notifications.dismiss(id)`. That is **an action available to the player** —
the game calls exactly the same thing when you dismiss a notification by hand (`panel-notification-train`)
— so it lifts the block instead of masking it.

```js
const ids = Game.Notifications.getIdsForPlayer(playerID)
    .filter((id) => Game.Notifications.getType(id) === Game.getHash(TYPE));
ids.forEach((id) => Game.Notifications.dismiss(id));
```

⚠️ Dismiss **only** when there really is nothing to be done about the notification. Dismissing
one the player could have acted on takes information away from them.

### How to hide the icon from the ring

`panel-action` is the **old framework** (`Controls.define`), and its class is exported,
so its prototype can be wrapped. The hook is `getNotificationInfo`: `refreshActionButton`
maps all ids through it and **discards `null`**.

```js
import { PanelAction } from '/base-standard/ui/action/panel-action.js';

const hidden = Game.getHash('NOTIFICATION_ASSIGN_NEW_RESOURCES');
const original = PanelAction.prototype.getNotificationInfo;
PanelAction.prototype.getNotificationInfo = function (id) {
    const info = original.call(this, id);
    return info?.type === hidden && notNeeded() ? null : info;
};
```

The panel refreshes itself on `NotificationAdded`, `NotificationUpdated`, `LocalPlayerTurnBegin`
and unit events — you do not have to add a listener of your own for the icon to come back.

⚠️ Hiding the icon **does not lift the turn block** — that is decided by the engine, not by whether
something is drawn.

## Hiding a notification from the BAR (the notification center) ✅

Notifications have **one handler per type**, and a mod can replace it — `registerHandler`
simply overwrites the entry in the map:

```js
import { NotificationHandlers } from '/base-standard/ui/notification-train/notification-handlers.js';
import { NotificationModel }    from '/base-standard/ui/notification-train/model-notification-train.js';

NotificationModel.manager.registerHandler('NOTIFICATION_ASSIGN_NEW_RESOURCES',
    new (class extends NotificationHandlers.AssignNewResources {
        add(id) {
            if (weDoNotWantToShowIt()) return false;   // not registered = not visible
            return super.add(id);
        }
    })());
```

**Why `add`:** `onNotificationAdded` calls `handler.add(id)`, and the default `add` is what
then calls `NotificationModel.manager.add(...)` — and that is what puts the thing on screen. A handler that
does not do that leaves the notification **unregistered**: it is still in
`Game.Notifications`, but the UI does not draw it.

⚠️ **This is not dismissing the notification.** Nothing is dismissed or deleted — the game's
state stays untouched (important for UI-only mods), and the notification can show up later
without the game having to raise it again.

**To bring it back** once the condition no longer holds, call `NotificationModel.manager.rebuild()`
— it walks all live notifications and offers them to the handlers again
(`rebuild` → `handler.add(id)` in a loop).

⚠️ We inherit from the game's **specific** handler (`NotificationHandlers.AssignNewResources`),
not from `DefaultHandler` — otherwise we lose what happens when the notification is clicked
(here: opening the commerce screen).

Type definitions: `base-standard/data/notification.xml`. It is worth looking at `SeverityType`
and `ExpiresEndOfTurn` — `HIGH` + `False` means "it will hang in the corner forever".

## `Catalog` / `SerialObject` — a mod's persistent state that accepts STRINGS ✅

**Established 2026-08-26** while specifying the "Better City UI" mod. It complements
[14-quirks #50](14-quirks-and-gotchas.md), which only describes `UI.setOption`.

`/core/ui/utilities/utility-serialize.js` exports `Catalog`, `SerialObject`,
`CatalogItemCommittedEvent` and `CatalogItemCommittedEventName`. It is **the game's own mechanism**,
used by it in five places: `core/ui/lenses/lens-manager.js`,
`base-standard/ui/quest-tracker/quest-tracker.js`, `base-standard/ui/advice/advice-manager.js`,
`base-standard/ui-next/screens/legacies/triumph-tracking-manager.js`,
`core/ui-next/screens/unlocks/civ-unlock-tracking-manager.js`.

```js
import { Catalog } from '/core/ui/utilities/utility-serialize.js';

const catalog = new Catalog({
    name: "MyMod",
    version: 1,
    player: Players.get(GameContext.localPlayerID),   // null => the "world" store
});
const obj = catalog.getObject("whatever");
obj.write("key", "a value, multi-word too");          // ✅ a string gets through
const v = obj.read("key");                            // undefined when absent
obj.getKeys();                                        // a Set of this object's keys
catalog.getObjectIds();                               // a Set of object names
catalog.dumpToLog();                                  // debug: the whole thing as an ASCII tree
```

**How it works underneath** (relevant for assessing the risk):

```js
internalWrite = (player, hash, value) =>
    player ? player.Tutorial.setProperty(hash, value) : GameTutorial.setProperty(hash, value);
hash = Database.makeHash("_" + scope + "_" + id + "_" + key);
```

So: **dynamic player properties**. The key list is itself kept as a string under
the `KEYS` key (`Array.join(",")`) — which is also proof that strings work.

| Property | |
|---|---|
| value type | ✅ string or number |
| scope | the player in this match (or the world, with `player: null`) |
| survives a restart | ✅ |
| versioning | ✅ `version` in the constructor, `fileVersion` vs `runningVersion`, `justCreated` |
| enumeration | ✅ `getKeys()` / `getObjectIds()` |

⚠️ **The write is QUEUED** when there is a `player`. A read in the same frame will return the old
value. The commit arrives as `PlayerDynamicPropertyChanged` and surfaces as the window event
`catalog-item-committed`. Do not build a "write then immediately read" loop.

⚠️ **This writes into the game save.** For a mod with `AffectsSavedGames = 0` that is still honest (it does not change
the rules), but "does not change the mechanics" and "writes nothing into the save" are two different statements.
❓ It has not been tested how a save loaded without the mod behaves — in theory it simply carries
unread properties.

**When to use what:**

| Need | Tool |
|---|---|
| a mod option (checkbox, pick from a list) | `UI.setOption('user','Mod',…)` + `saveCheckpoint()` |
| structured state per match / per city | ✅ `Catalog` with `player` |
| state shared across all matches, non-numeric | `Catalog` with `player: null` |
| anything | ❗ **not `localStorage`** — it does not survive a game reload |
