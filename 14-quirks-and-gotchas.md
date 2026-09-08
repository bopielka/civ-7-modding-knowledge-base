# 14 — Traps and oddities

Things that cost hours if you do not know them. Collected as they come up.

## 1. Two modifier syntaxes, easy to mix up ✅

```xml
<!-- ✅ NEW (GameEffects) — lowercase, this is the default route in Civ VII -->
<Modifier id="..." collection="COLLECTION_PLAYER_CITIES" effect="EFFECT_CITY_ADJUST_YIELD">

<!-- ⚠️ OLD (table-based, from Civ VI) — uppercase, in the DynamicModifiers table -->
<Row ModifierType="..." CollectionType="COLLECTION_OWNER" EffectType="EFFECT_..."/>
```
Searching for `CollectionType=` finds ~5 results; searching for `collection=` finds thousands.
**If your greps return suspiciously few results — check the letter case.**

## 2. `Types` — nothing works without that row ✅

Every new object must have a row in `Types` with the right `Kind`, **before** it appears
in the target table. A missing entry = a silent foreign-key failure.

## 3. A civilization requires BOTH scopes ✅

`shell` (the selection menu) and `game` (gameplay) are separate contexts with separate databases.
The Poland mod **repeats** `ImportFiles` and `UpdateText` in both groups. If a civilization
works in game but is not visible in the menu — the `shell` group is missing.

## 4. An empty syncretism card ✅ (documented by the mod's author)

The syncretism screen resolves a civilization through the **legacy** tables before it reaches for
`CivSelfSyncretismUnlocks`. Without rows in `LegacyCivilizations`
and `LegacyCivilizationTraits` the effect will be granted, but **the card in the UI will be empty**.

## 5. `Controls.decorate` does not work retroactively ✅

The Firaxis comment in `component-support.js`:
> it will not create a decorator for existing instances of the component

The decorator has to be registered **before** the component is created → which is why `LoadOrder`
matters, and why UI mods often set it to `1000`.

## 6. A prototype patch needs a guard ✅

Without `if (Class.patched === proto) return;`, with several instances of a component you will wrap
the same method several times — each instance adds another layer.

## 7. Always `engine.off` ✅

306 `engine.on` calls vs 96 `engine.off` in mods — many authors do not clean up.
Failing to unregister in `beforeDetach`/`afterDetach` = leaks and duplicated handlers.

## 8. `<EnglishText>` vs `<LocalizedText>` ✅

```xml
<EnglishText><Row Tag="LOC_X"><Text>Hello</Text></Row></EnglishText>
<LocalizedText><Row Tag="LOC_X" Language="pl_PL"><Text>Cześć</Text></Row></LocalizedText>
```
A different tag **and** the `Language` attribute only on the second one. Mixing them up = no translation.

## 9. Two different `<LocalizedText>` ✅

- Inside `<Actions>`: does not exist — the game's texts are loaded by `<UpdateText><Item>`
- At the `<Mod>` level: `<LocalizedText><File>` — the texts **of the modinfo itself**
  (the mod's name in the list). It uses `<File>`, not `<Item>`.

## 10. `ImportFiles` overrides game files by path ✅

If your file has **the same relative path** as a game file
(e.g. `ui-next/tooltips/plot-tooltip/plot-tooltip.js`), it replaces it.
Sometimes that is intended — but it is easy to do by accident by naming a folder the way the game does.

## 11. Asset files duplicated without an extension ❓

The Poland mod registers every asset twice:
```xml
<Item>Art/civ_poland.png</Item>
<Item>Art/civ_poland</Item>
```
and it does have both files on disk. I have not established whether this is a game requirement or redundancy.
**If your icons do not work — try this pattern.**

## 12. The `fs://game/` alias = the mod identifier ✅ (with exceptions ❓)

Verified: `bz-map-trix`, `leugi-diploribbon-tweaks`, `detailed-map-tacks`,
`maple-leaves-more-lens` — the alias matches `<Mod id=...>`.

But there are discrepancies:
| Mod ID | Alias used |
|---|---|
| `szczupakabra-poland` | `codex-poland-civilization` (25×, consistently) |
| `f1rstdan-cool-ui` | `f1rstdans_cool_ui` (3×) alongside `/f1rstdan-cool-ui/` (5×) |
| — | `RHI_mod`, `yield_influence_5` |

❓ I do not know whether these are dead paths (and those assets simply do not work), or whether the alias
comes from somewhere else (e.g. the folder name in non-Workshop distribution).
**The safe assumption: use exactly your own `<Mod id>`.**

## 13. `AiLists.LeaderType` holds a TRAIT, not a leader ✅

```xml
<Row ListType="Ada Lovelace Yield Biases"
     LeaderType="TRAIT_LEADER_ADA_LOVELACE_ABILITY" System="YieldBiases"/>
```
The column name misleads — the value is a trait type.

## 14. Booleans: XML vs SQL ✅

- XML: `IsMajorLeader="true"` (Firaxis) — though `"1"` also occurs
- SQL: `1` / `0`

## 15. Age data only in a group with `AgeInUse` ✅

An age's tables and types exist only while that age is active. Putting Antiquity data
into an `always` group may fail.

## 16. `UniqueCultureProgressionTree` changes per age ✅

The Poland mod sets the value in `shared.sql` and then **overrides it with an `UPDATE`**
in each age's file. One value is not enough.

## 17. The 3D model is a dead end ✅

It requires a `.dep` package with GUIDs and material-library dependencies — there are no public
tools. **Use `VisualRemaps`** to borrow an existing model.

## 18. `grep -r` over the game directory is very slow ✅

1.3 GB. Narrow it down: `Base/modules/*/data/` is only 14 MB and 461 XML files.
(One such search took over 120 s here and had to run in the background.)

## 19. `Package` — letter case ❓

44 mods have `<Package>Mod</Package>`, 2 have `MOD` and work.
The comparison is probably case-insensitive, but stick to `Mod`.

## 20. The community documentation mixes in Civ VI tables ❗ ✅

The leader guide on `civ7community.mintlify.app` lists `LeaderCivilizations`,
`Agendas`, `HistoricalAgendas`, `RandomAgendas` — **none of which exist in Civ VII**.
Always check a table name in the schema before using it:
```bash
grep -c "CREATE TABLE 'TableName'" "$G/Base/Assets/schema/gameplay/01_GameplaySchema.sql"
```
Full analysis: [22-source-evaluation.md](22-source-evaluation.md).

## 21. Do not design around systems that do not exist ✅

Missing: Faith, Tourism, Loyalty, Amenities, Governors.
Happiness (`YIELD_HAPPINESS`) is a **per-turn yield**, not a pool.
Details: [21-gameplay-mechanics.md](21-gameplay-mechanics.md).

## 22. ❗ User mods do NOT go into `Documents\My Games` ✅

The most expensive trap at the start — the mod simply does not appear in the list
and leaves no trace in the logs.

**The correct path:**
```
C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\
```
`Documents\My Games\Sid Meier's Civilization VII\` contains **only `Saves\`**.
The `Documents` path is a Civ VI convention — the community documentation repeats it too,
so it is easy to be caught out.

Diagnosis: if `grep -i "mod-name" Modding.log` returns nothing and the log says
`Discovered 0 mods.` — the mod is in the wrong place. The game prints the scanned path plainly:
```
Discovering new mods...
C:/Users/najan/AppData/Local/Firaxis Games/Sid Meier's Civilization VII/Mods/
```

## 23. A panel can have several frames — you decorate the one you cannot see ✅

`panel-place-population` builds **three** independent frames and switches between them with the
`hidden` class depending on state:
```js
this.subsystemFrame.classList.toggle("hidden", state != PlacePopulationSelectionState.NONE);
this.placeImprovementFrame.classList.toggle("hidden", state != ...ADD_IMPROVEMENT);
this.placeSpecialistFrame.classList.toggle("hidden", state != ...ADD_SPECIALIST);
```
Injecting DOM into `subsystemFrame` is **invisible** in the add-specialist mode —
because that frame is hidden then. This cost me one iteration
of "the mod loads, but nothing is visible".

**Diagnosis:** if the decorator works (no errors in `UI.log`) but you see no changes —
check whether the component has several containers toggled with `hidden`, and inject
into the one matching the current state. A screenshot from the game tells you plainly which
frame you are looking at (here: the "ADD SPECIALIST" heading = `placeSpecialistFrame`).

## 24. Yields and maintenance are two separate lists in the UI ✅

`WorkerPlacementInfo` carries **four** arrays: `CurrentYields`/`NextYields`
and `CurrentMaintenance`/`NextMaintenance`. The game draws them as separate groups of pills.
A specialist's maintenance costs (e.g. −2 food, −2 happiness) live **only**
in `*Maintenance`, not in `*Yields`.

The sign convention in the game's code:
```js
yieldChange      = NextYields[i] - CurrentYields[i]           // positive = a gain
maintenanceChange = CurrentMaintenance[i] - NextMaintenance[i] // ALREADY negative when the cost rises
```
So the two can simply be **summed** to get the net change per yield.
If you compute "what a specialist gives" and skip `*Maintenance`, you lose the whole cost
side — and that is usually what interests the player most.

## 25. A lens layer registers LATER than the mod's script ✅

`LensManager.registerLensLayer(...)` only runs once the game imports the layer's file —
and that happens **after** the mod's scripts run, even with `LoadOrder=1000`.
Patching a layer inside `engine.whenReady` **silently does nothing**:
```
najane-specialists: 'fxs-worker-yields-layer' never registered, tile pills not patched
```
The fix: retry the patch on `InterfaceModeChangedEventName` (or on the
`lens-event-layer-enabled` events) and guard it with an "already patched" flag.
Then the patch applies at the latest when you enter a mode that uses the layer.

⚠️ Always log the "did not find the object to patch" case — otherwise the mod
looks like it works while simply doing nothing. (See [19](19-workflow-and-debugging.md):
use `console.error` for logging, because `console.log` does not reach `UI.log`.)

## 26. `fxs-subsystem-frame` rearranges its children — insert DOM inside `waitForLayout` ✅

Inserting an element with `insertBefore(...)` in a decorator's `afterAttach()` **does not land
where you tell it to** — once built, the `fxs-subsystem-frame` moves its children
into an internal scroll container, so your element ends up at the end.
Symptom: the section was supposed to be at the top and it is at the very bottom (with no error in the log).

The fix — use the global `waitForLayout()` (from `core/ui/component-support.js`,
available **without an import**; it defers by 2 animation frames, `LAYOUT_FRAME_DELAY = 2`):

```js
afterAttach() {
    waitForLayout(() => {
        const anchor = this.component.someContainer;
        anchor?.parentElement?.insertBefore(mySection, anchor);  // only now
    });
}
```
The game itself does the same in `panel-place-population.js` — after `buildView()` it fixes
the scrollbars inside exactly this `waitForLayout`.

⚠️ Always target `anchor.parentElement`, not the frame itself — after the rearrangement
the anchor's parent is no longer the element it was originally added to.

## 27. A game inconsistency: "Specialist maintenance" disappears on an occupied plot ✅

In `model-place-population.js` the maintenance line is shown **only** when the raw
value is positive:
```js
if (nextValue > 0) { this.showAfterSpecialistMaintenance = true; ... }
```
On a plot that **already has a specialist**, the condition fails and the description disappears — even though
the "RESULTS" bar above it correctly accounts for the cost, because it is computed from a different field
(`overallChange = bonusChanges - maintenanceChanges`). The data is there, it is just not shown.

> ❌ **AN ATTEMPTED WORKAROUND FAILED — REVERTED.** I wrapped `PlacePopulation.update`
> and filled in the missing line from `NextMaintenance`/`CurrentMaintenance`. The result:
> **the numbers were wrong** — they showed about twice the real cost and did not match
> the "RESULTS" bar in the same window. Reverted; better to show nothing than to show
> an untruth.

❓ **What I do not know:** what `CurrentMaintenance` / `NextMaintenance` in
`WorkerPlacementInfo` mean exactly. Observations in game contradict all of my hypotheses:
- "Specialist bonus" shows **totals** (1 specialist +5 culture → 2 specialists +10),
  while the "RESULTS" bar shows the **increment** (+5)
- under the same interpretation for maintenance the numbers do not add up — the values
  from `*Maintenance` do not reproduce the increment visible in "RESULTS"

**Before anyone tries again:** first dump the real values of both arrays for a plot
with 0 and with 1 specialist (`console.error`, see [19](19-workflow-and-debugging.md)) and only
then build a formula on that data. Without it, it is guesswork.

**The general lesson (still valid):** before you conclude the data does not exist, check whether
it is just a display condition — the same cost is sometimes computed in two places
with two different conditions. But **do not add numbers of your own until you understand
the units** — the result is worse than nothing.

## 28. Keybindings only in `scope="shell"` — otherwise the WHOLE action rolls back ✅

The tables `InputActions`, `InputActionDefaultGestures`, `InputContextConstraints` belong
to the **frontend** database, not gameplay. Hooking a file containing them into a `scope="game"` group ends with:

```
[gameplay] ERROR: no such table: InputActions
ERROR: There were errors loading 'config/input.xml' that require a rollback.
Warning: Apply Actions - Errors when applying action '...(UpdateDatabase)'. Rollback Required.
```

❗ **The worst part is that the rollback reverts the group's whole `UpdateDatabase` action**,
not just the offending file. So a single misplaced file can disable all of the mod's
remaining data — with no visible symptom other than nothing working.

Correctly: `config/input.xml` **only** in a `scope="shell"` group. The action still
works in gameplay afterwards, because the definitions live in the configuration database.

The pattern is proven in `bz-map-trix` (`config/bz-input.xml`, shell only) — it also uses
`<Replace>` instead of `<Row>`, which is idempotent when the mod is reapplied.

## 29. A modifier as a shortcut: `EventType="All"` ✅

⚠️ **A correction** to an earlier conclusion in [09](09-cookbook-ui-mod.md) that the game's action
system cannot express "a key is being held". It can — you just have to declare it:

```xml
<Replace ActionId="my-action" DeviceType="Keyboard" EventType="All"
         Name="LOC_..." Description="LOC_..." />
<Replace ActionId="my-action" Index="0" GestureType="KBMouse" GestureData="KEY_CONTROL"/>
```
Then `InputEngineEvent` reports `InputActionStatuses.START` on press
and `FINISH` on release. The statuses: `START`, `FINISH`, `UPDATE`, `DRAG`, `HOLD`.

✅ **A bare modifier is a valid binding** — the game does this for its own
`keyboard-camera-modifier` action (`GestureData="KEY_ALT"`). Available are, among others, `KEY_CONTROL`,
`KEY_SHIFT`, `KEY_ALT` and the L/R variants. Combinations: `KEY_SHIFT+KEY_S`.

This solution is **much better than listening on the DOM**: the entry appears in
Options → Controls, the player can remap it, and it does not clash with other mods
reacting to the same key.

You fetch the label of the current binding for the UI with:
```js
Input.getGestureDisplayString(actionId, 0, InputDeviceType.Keyboard, InputContext.ALL)
```
⚠️ To display it use `Locale.compose(key, label)` and `textContent` — the
`data-l10n-id` attribute will not accept an argument computed at runtime.

## 30. When patching a method that another mod patches — check what you are losing ✅

A real conflict: City Hall (`bz-city-hall`) wraps `updateSpecialistPlot`
correctly — it calls the previous version, then draws its own icons:
```js
const prev = WYLL.updateSpecialistPlot;
WYLL.updateSpecialistPlot = function(...a) { prev.apply(this, a); this.realizeBuildSlots(d); }
```
My mod patched **after** them and **did not delegate** (deliberately — it replaces the drawing).
The result: their icons disappeared, and came back only in "show the original" mode, because there
I did delegate. The symptom looked like a key conflict, but it was about patch order.

**Diagnosis:** if another mod's feature only works while your mod is
"off", you are almost certainly breaking the wrapper chain.

**The fix** (when you cannot delegate): perform their step yourself, with feature
detection, so that the absence of that mod is a no-op:
```js
if (typeof this.realizeBuildSlots === "function" && this.bzGridSpritePosition) {
    try { /* repeat their steps */ } catch (e) { console.error(...); }
}
```
⚠️ This ties you to the internal names of somebody else's mod. Wrap it in `try/catch`, do feature
detection, and note in a comment whose code you are reproducing — when they update it, someone
has to know where to look.

✅ A side benefit: by calling their methods through `this` (e.g. `getSpecialistPipOffsetsAndScale`),
you get their improvements for free too.

## 31. Do not edit files in `Program Files` ✅

Steam will overwrite the changes when it verifies files, and writing requires administrator rights.
Workshop mods (`steamapps\workshop\content\1295660`) are managed by Steam too —
your changes will be overwritten when the subscription updates.

## 32. A screen that cannot be decorated — because it is not in the old framework ✅

**Date: 2026-08-10.** Before you write `Controls.decorate('screen-name', …)`, check
whether the screen lives in `ui-next/` (Solid.js). The symptom will be "the decorator registers,
nothing happens" — because an element of that name is indeed defined, but its content
is rendered by Solid, not by DOM from a `.html.js`.

```bash
find "…/Base/modules" -ipath "*ui-next*" -iname "*name*"
```

Recognizing it from the files: next to the `.js` there is a `.js.map` from which a **`.tsx`**
is extracted (not a `.ts`), and the code contains `ComponentRegistry.register` / `createSignal`.
Full description: [25-ui-next-solidjs.md](25-ui-next-solidjs.md).

## 33. The same screen exists in TWO versions at once ✅

During the migration to `ui-next`, Firaxis **leaves the old files on disk and still loads them**.
The Commerce screen has a full set in `ui/resource-allocation/` (old) and in
`ui-next/screens/commerce/` (new) — both listed in `base-standard.modinfo`.

The new one wins, because `defineLegacyComponent` calls `Controls.define` with `priority: 1`,
while the old `Controls.define` defaults to `0`. A mod editing the old file will change nothing
visible. **Always check whether there is a second version of the screen** before you start
reading the code of "the obvious" file.

## 34. `overridePriority` in `ui-next` removes the load-order problem ✅

In the old framework a patch had to land on an already existing object (the lens-layer
quirk). `ComponentRegistry.register` / `ModelRegistry.register` keep **one
wrapped factory per name** and only swap the pointer to the implementation inside it
when a higher priority arrives — so a mod can register before the game or after
it, the effect is the same.

⚠️ Exception: this only works for components actually **registered** in the registry.
A plain `export const Foo` without `register` is untouchable. And overriding a model in
`ModelRegistry` achieves nothing if the consumer calls the factory directly instead of
`Model.get()` — which is exactly what the Commerce screen does.

## 35. `overridePriority` on the Commerce screen is already taken by Resource+ ✅

**2026-08-10.** The **Resource+** mod (`brads-assign-all-resources`, Workshop 3756000777)
registers `CommerceResourcesContainer` with `overridePriority: 1100`. Anyone who wants to touch
that same tab has to give a higher number **and delegate** to `originalFactory(props)` —
otherwise Resource+'s features simply disappear (the same as quirk #30 with City Hall).

The general rule: **before you pick an `overridePriority`, check whether the name is already
taken** by an installed mod:

```bash
grep -rn "overridePriority" "…/steamapps/workshop/content/1295660"
```

## 36. When overriding a `ui-next` component, clean up after yourself in `onCleanup` ✅

Injected raw DOM (buttons, a `<style>` in `document.head`, classes added to
other people's elements) **will not disappear on its own** — Solid removes only what it rendered
itself. Resource+ removes every element individually in `onCleanup`, detaches its `MutationObserver`,
listeners and `requestAnimationFrame`. The Commerce screen is opened and closed many times
in a match, so no cleanup = accumulating duplicates.

## 37. The mod "loads" but its scripts never run — because it is not ENABLED ✅

**2026-08-10.** `Modding.log` showed "Loading Mod – …", "Discovered 1 mods" and the mod's
name, and yet `UI.log` had not a single line from its script — not even the marker
from the first line of the entry file. No import error either.

**Discovery ≠ enabled.** The game scans the `Mods\` directory on every start, but the mod still
has to be enabled in *Main menu → Additional Content → Mods*. A new mod is **not**
enabled by default.

How to check without guessing — at "Applying mod components" `Modding.log` prints
the list of **actually enabled** mods together with their action groups:

```
[…] najane-common-specialists-yields (Better Specialists UI by Najane)
[…]  * najane-specialists-ui
[…] holistic-qol-plus (Holistic QoL+)
[…]  * holistic-qol-plus-game
[…] Applying mod components.
```

Your mod is not on that list → it is disabled, and no amount of code debugging will help.

The other route — `Mods.sqlite`, the `Mods.Disabled` column (⚠️ not `Enabled`; `NULL` means
"not disabled"). Action registration can be confirmed like this:

```python
import sqlite3
c = sqlite3.connect('Mods.sqlite'); c.row_factory = sqlite3.Row
for r in c.execute("select ModRowId, ModId, Version, Disabled from Mods where ModId='your-mod'"):
    print(dict(r))
# ActionGroups(ActionGroupRowId, ModRowId, ActionGroupId, Scope, CriteriaRowId)
# Actions(ActionRowId, ActionGroupRowId, ActionType)
# ActionItems(ActionRowId, Arrangement, Item)
```

## 38. Text files for other languages: `<LocalizedText Language>`, not `<EnglishText>` ✅

**2026-08-10.** A `text/pl_PL/ModInfoText.xml` with an `<EnglishText>` block goes into the database
as `en_US` — **the directory name means nothing** — and collides with the real
`text/en_us/…`:

```
ERROR: Database: UNIQUE constraint failed: LocalizedText.ModRowId, LocalizedText.Tag, LocalizedText.Locale
Database: While executing - 'INSERT INTO LocalizedText(...) VALUES(485,'LOC_…','en_US','…')'
```

Correctly:

```xml
<Database>
    <EnglishText>                                  <!-- text/en_us/ only -->
        <Row Tag="LOC_X"><Text>English</Text></Row>
    </EnglishText>
</Database>

<Database>
    <LocalizedText>                                <!-- every other language -->
        <Row Tag="LOC_X" Language="pl_PL"><Text>Polski</Text></Row>
    </LocalizedText>
</Database>
```

This applies equally to files listed in `<LocalizedText><File>` in the `.modinfo` and to those in `UpdateText`.

## 39. `version="0.1"` in `.modinfo` = the mod is silently skipped ❗✅

**2026-08-10, cost two rounds of debugging.** The `version` attribute on the
`<Mod>` element is parsed as an **integer**. `version="0.1"` lands in
`Mods.sqlite` as `Version = 0`, and the game **does not apply** such a mod — where:

- `Modding.log` normally writes "Loading Mod – …" and "Discovered 1 mods",
- the mod is visible and **ticked as enabled** in the Mods menu,
- in `Mods.sqlite` it has `Disabled = 0`, with all `ActionGroups` / `Actions` /
  `ActionItems` registered correctly,
- **but it does not appear on the list of enabled mods** printed just before
  "Applying mod components", and its scripts never execute —
  **without a single error message**.

Numerical evidence from the user's installation: 93 mods in the database, 40 applied, all
applied ones have `Version >= 1`, and the only mod with `Version = 0` was exactly the
one that did not work.

```xml
<Mod id="my-mod" version="1" xmlns="ModInfo">      <!-- ✅ an integer >= 1 -->
    <Properties>
        <Version>0.1</Version>                      <!-- ✅ any text here -->
```

Note: `version="1.3"` also "works", but is stored as `1` — the part after the dot is simply
truncated. Keep the version number you want to show the player in `<Properties><Version>`.

**A quick diagnosis for any "the mod does not start":**

```python
import sqlite3
c = sqlite3.connect('Mods.sqlite')
print(list(c.execute("select ModId, Version, Disabled from Mods where Version = 0")))
```

See also #37 (discovered ≠ enabled) — the symptoms are identical, the causes different,
and the same list in `Modding.log` settles it.

## 40. A modifier-click does not arrive as an engine action ❗✅

**2026-08-10, four rounds of testing.** The engine **does not send the `mousebutton-right`
action (nor, presumably, other mouse actions) while a modifier is held**. Shift+RMB generates
only native DOM `mousedown`/`mouseup` events — and nothing else.

A misleading symptom: it looks like "I cannot read Shift". I lost two rounds
swapping the source of the modifier state (DOM `keydown` → `Input.isShiftDown()`),
while **both worked correctly** — they were simply never asked, because the event
in which I asked never arrived.

**Handle modifier-clicks through the DOM** (`mousedown`/`mouseup`, `button === 2`,
`event.shiftKey`), and intercept the engine action only in order to suppress the default
"cancel". Details and event order: [25-ui-next-solidjs.md](25-ui-next-solidjs.md).

**A rule for the future:** on the third failed hypothesis about input, **stop
guessing and attach a listener** that logs every `engine-input` (name + status +
coordinates) plus native `mousedown`/`mouseup`/`keydown` with the modifier flags.
One round with data settled what three rounds of theory could not.
The listener must skip the `InputActionStatuses.UPDATE` status — it repeats every frame.

## 41. `Game.PlayerOperations.sendRequest` queues — `canStart` in the same tick lies ✅

**2026-08-10.** Player operations do not execute immediately. After `sendRequest` the game
state has not changed yet, so `canStart` asked about the **next** operation in the
same function answers based on the old state.

A symptom from practice (the Commerce screen, removing a camel that provides 2 slots): the mod sent,
in one go, the release of 2 resources and the release of the camel. The resources went out,
the camel was refused and **stayed**. The next click released 2 resources again —
this time needlessly, because the room was already there.

**The fix:** do it sequentially and after each step wait for confirmation from
the engine (a domain event, e.g. `ResourceUnassigned`), with a timeout in case
an operation gets lost. Additionally, **ask `canStart` instead of computing**
how many preparatory steps are needed: release one at a time and stop once the actual
operation succeeds. Then you do not have to know the engine's rule, and the side effects are minimal.

```js
if (trySend(target)) return;                 // maybe nothing is needed
while (queue.length) {
    if (trySend(queue.shift())) await waitForEngineEvent();
    if (trySend(target)) return;
}
```

## 42. When highlighting something in someone else's UI, use an effect that UI already has ✅

A yellow outline of my own looked like a new kind of decoration and the user rejected it
immediately. The Commerce screen already has its own hover effect — `hover\:scale-125` in the
`DraggableResource` classes — so "the same as what is under the cursor" highlighting should be
**exactly that same enlargement**, not a new convention.

⚠️ Target an element rendered in **every** branch of the component. In this case
`.framed-resource` (always present), not `.draggable-resource` (only when the resource is
interactive).

❗ **Do not stack your effect on an element the game already transforms** — `transform`
**multiplies**. Duplicating the game's `scale(1.25)` with my own `scale(1.25)` gave 1.5625 on the
hovered element, i.e. visibly bigger than the rest of the highlighted group. Skip the element
the game handles itself — and check it with a condition, not an assumption, because the hover
effect is sometimes rendered in only some branches of the component:

```js
const gameHandlesIt = slotElement.querySelector('.draggable-resource') !== null;
```

## 43. In `ui-next`, `onMount` does NOT guarantee the component's content is in the DOM ✅

**2026-08-10.** The mod looked in `onMount` for an element inside a tab and **every time**
got `null`, even though the element was on screen a moment later.

The cause: `CommerceScreenBaseTabContent` wraps the tab's content in
**`<ThrobberSuspense>`**. Solid's Suspense renders a placeholder first and the actual
content only after resources resolve (preloading of images and styles by
`ComponentRegistry`). `onMount` runs on the placeholder.

This is the inverse of trap #1 from the old framework ("you patch an object that does not
exist yet"): here the object exists, but **its content does not**.

**The fix — search until you find it, and only then narrow down:**

```js
function tryAttach() {
    const target = document.querySelector('[data-name="…"]');
    if (!target) return false;
    observer.disconnect();                    // the broad observation is no longer needed
    observer = new MutationObserver(onChange);
    observer.observe(target, { childList: true });   // narrow, cheap
    return true;
}
if (!tryAttach()) {
    observer = new MutationObserver(tryAttach);
    observer.observe(document.querySelector('screen-…') ?? document.body,
                     { childList: true, subtree: true });
}
```

⚠️ When you observe in order to **add classes of your own**, set **`childList` only**.
With `attributes: true` your own class change fires your callback and you get a loop.

## 44. Hiding a game element also takes away its margins ✅

`display: none` on a text insert removed not just the text but also its `my-4` on both
sides — the panel stuck to the tabs above it. Give the spacing back to the neighbor:

```css
.that-insert { display: none; }
.that-insert + div { margin-top: 0.8888888889rem; }   /* the same as one my-4 */
```

The `+` selector works normally on an element with `display: none` — it is still in the tree.

## 45. Civ VII's DOM engine lacks the newer `ParentNode` methods ✅

**2026-08-10.** `element.replaceChildren()` throws
`TypeError: replaceChildren is not a function`. The consequence was insidious: the code worked
in Node during a syntax check, entered the game without a load error, and only blew up
on first use — and only in `UI.log`.

Clear and add children the old way:

```js
while (element.firstChild) element.removeChild(element.firstChild);   // instead of replaceChildren()
for (const child of children) parent.appendChild(child);              // instead of append(a, b, c)
```

⚠️ The same goes for `append()` with multiple arguments — it is from the same API generation
as `replaceChildren` and there is no reason to assume that this one happens to be there.

**The rule:** in this UI stick to `appendChild` / `removeChild` / `insertBefore` /
`querySelector` / `classList` / `setAttribute`. If you reach for something newer, first check
whether some working Workshop mod uses it — Resource+ clears containers with a
manual loop, and now we know why.

## 46. `UI.getIcon(type, "YIELD")` returns a MIXED-CASE name ❗✅

**2026-08-10.** `UI.getIcon('YIELD_HAPPINESS', 'YIELD')` gives **`blp:Yield_Happiness`**,
not `blp:YIELD_HAPPINESS`. Code that extracts the yield type from an icon name with the
regular expression `/YIELD_[A-Z_]+/` **never matches** — and quietly returns `null`, so every
value depending on it comes out 0.

This cost a whole round: a mechanism driven by settlement happiness "did not work", because every
happiness value read as zero. The same bug sits in the Resource+ mod, from which the
function was taken over — so its "Balanced" mode is also computing on zeros.

**Never parse an icon name.** Build a map with the same function that built the value:

```js
const byIcon = new Map();
GameInfo.Yields.forEach((y) => byIcon.set(`url(${UI.getIcon(y.YieldType, 'YIELD')})`, y.YieldType));
// now: byIcon.get(entry.yieldIconSrc)
```

This is the same principle as with the yield badges in [26-commerce-screen.md](26-commerce-screen.md):
the model writes `url(${UI.getIcon(...)})`, so the reverse mapping must also be built
from `UI.getIcon`, not from a guess about the shape of the string.

## 47. This CSS engine does not know `:focus-visible` ✅

**2026-08-10.** A rule with `:focus-visible` is not applied, and `UI.log` gets
`Unsupported CSS pseudo class selector encountered: focus-visible` on every
recalculation of the stylesheet — i.e. it litters the log every time the screen opens.

Use `:focus`. `:hover`, `:active` and `:not(...)` are also supported; ⚠️ complex
selectors inside `:not()` can break `querySelector` (see the holistic-qol-plus mod's error
in the logs: `Invalid CSS selector (.text-accent-1:not(.class))`).

## 48. Diagnostics must also log the decision to DO NOTHING ✅

A mechanism with an on/off option that stays silent when the option is off is indistinguishable
from a broken one. After a report of "it did not work" the log contained **nothing** and there was no way
to tell whether the problem was in the event, in change detection, or simply that the option was
off.

Every early return should leave a trace — with the trigger's name and the reason:

```
[mod] TradeRouteAddedToMap: auto-assign is switched off (Options -> Mods)
[mod] TradeRouteAddedToMap: nothing new (37 resources owned)
[mod] TradeRouteAddedToMap: 1 newly acquired resource(s)
```

One test round then settles everything at once: whether the event arrived at all, whether
detection worked, and whether the option is on.

## 49. Do not initialize the "what was already there" state on the first event ✅

**2026-08-10.** A mechanism reacting to *new* things has to know the baseline state. It is natural
to want to record it on the first event — and that is a mistake:
**the first event is usually exactly the thing the user is waiting for.**

For us: the player turned on the "assign new resources" option, established a trade route, and the log said
`first pass, remembering 83 resources already owned` — i.e. that very route was
consumed building the reference list and nothing got assigned. From the outside it looks
exactly like a broken feature.

**Initialize at startup, independently of events.** A mod's scripts load before the game
can answer (`Players.get(GameContext.localPlayerID)` may still be empty), so
ask in a retry loop instead of waiting for something to happen:

```js
function seedWithRetries(attemptsLeft) {
    if (trySeed() || attemptsLeft <= 0) return;
    setTimeout(() => seedWithRetries(attemptsLeft - 1), 1000);
}
```

Leave an emergency path in the event handler too (in case the retries run out), but
**log it as a warning** — it means the startup assumption did not hold.

## 50. A mod's persistent state: `UI.setOption` with a NUMBER, not `localStorage` ✅ ❗ **SEE THE CORRECTION AT THE END OF THIS ENTRY**

**2026-08-10.** Saving a mod's own state (e.g. a per-city choice) to
`localStorage` alone **did not survive a game reload**. The channel that works is the same one
the mod options use:

```js
UI.setOption('user', 'Mod', `${MOD_ID}.whatever`, number);
Configuration.getUser().saveCheckpoint();   // ⚠️ without this nothing persists
…
const stored = UI.getOption('user', 'Mod', `${MOD_ID}.whatever`);   // null when absent
```

❗ **The value must be a number.** Every use of `UI.setOption` in the game's own code passes
a number (0/1, delays, indices). Do not count on JSON or a string getting through — encode the
state numerically.

A pattern for an enumeration with a "not set" option: store the **index + 1**, then `0`,
`null` and `undefined` all mean the same thing ("never chosen") and cannot be confused with the first
item of the list. **Only ever extend** the list of codes — reordering it will silently
corrupt everything already saved.

⚠️ **Key state tied to a specific match by `Configuration.getGame().gameSeed`.**
City identifiers (`ComponentID.id`) are unique only within one game — the same
number in the next campaign is a different city. The seed is constant for the life of a match
and differs between matches, so two saves from the same campaign share settings,
while unrelated games do not mix.

You do **not** need to touch the game save (`AffectsSavedGames`) for this — and you should not, if the
mod declares `0`.

### ADDENDUM 2026-08-26: there is a second channel — `Catalog`, and it accepts STRINGS ✅

The above still holds for **mod options**. But for *structured, per-match,
per-city* state there is a better tool that we did not know about before:
`Catalog` / `SerialObject` from `/core/ui/utilities/utility-serialize.js` — see
[17-advanced-and-undocumented.md](17-advanced-and-undocumented.md).

| | `UI.setOption` | `Catalog` |
|---|---|---|
| value type | ❗ **number only** | ✅ **string or number** |
| scope | the user (global) — a match has to be keyed by `gameSeed` | ✅ the player in this match, by nature |
| survives a game restart | ✅ | ✅ |
| write | immediate + `saveCheckpoint()` | ⚠️ **queued**, committed by an event |
| touches the game save | no | ⚠️ **yes** (`player.Tutorial.setProperty`) |

⚠️ That last one is a real trade-off, not a formality: `Catalog` with `player` writes **dynamic
player properties into the save**. The mod itself still does not change the rules and `AffectsSavedGames = 0` is
honest, but "does not change the mechanics" and "writes nothing into the save" are **two different
statements** and only the first is then true. A save loaded without the mod simply carries unread
properties. ❓ Not tested in practice.

The choice: **options → `UI.setOption`; per-city/per-match state → `Catalog`.**

### ❗❗ CORRECTION 2026-08-27 — measured on disk, it comes out the OTHER WAY ROUND

The above was written on 2026-08-10 based on the observation that "the setting did not survive
a reload". Inspecting the user's files shows something different. **I am not removing the original
entry — below is what can be verified.**

**Where things really live** ✅:

| Channel | On disk |
|---|---|
| `localStorage` | `%LOCALAPPDATA%\Firaxis Games\Sid Meier's Civilization VII\LocalStorage.sqlite`, table `Values(id, key, value)`, `id = 'fs://game'` |
| `UI.setOption('user', 'Mod', …)` | ❗ **nowhere to be found** |

1. ✅ **`localStorage` DOES SURVIVE a game restart.** `LocalStorage.sqlite` exists and holds entries
   from previous sessions — among them `najane-commerce-merchant-orders` with a key based on the match seed.
2. ❗ **`UI.setOption('user','Mod',…)` leaves no trace on disk.** `UserOptions.txt` has the sections
   `[Accessibility]`, `[Gameplay]`, `[Interface]`… and **no `[Mod]` section**. Searching
   all `.txt`, `.json` and `.sqlite` files in the user folder for a specific mod key
   hits **only `LocalStorage.sqlite`** — even though the code called `UI.setOption`
   **and** `Configuration.getUser().saveCheckpoint()`.
3. ❗ **`UI.getOption` returned null on every read in the 2026-08-27 session.** `UI.log`, the
   `bz-map-trix` mod (which uses the shared `ModOptionsSingleton` from City Hall):
   ```
   LOAD bz-map-trix.commanders=undefined (stored)
   LOAD bz-map-trix.commanders=2 (default)
   ```
   That code logs `(saved)` when `UI.getOption` returns something, and `(stored)` when it falls back to
   `localStorage`. **`(saved)` — zero occurrences. `(stored)` — three, all `undefined`.**

⚠️ **A caveat:** those three reads are in the **shell** scope, at game start, and the log had 30 lines.
That is strong circumstantial evidence, not proof. ❓ **To be settled by a test in game** — the procedure is in
`mod-projects/better-city-ui/documentation/07-feature-specs.md`, the section on feature 4.

**The practical conclusion for today:** write through **both** channels, the way
`better-commerce-screen-ui` does — but do not assume `UI.setOption` persists anything, and read
`localStorage` as an equal, not a "backup". And read #62 below before you consider
`localStorage` safe.

## 51. An injected element in a Solid tree disappears — you need a permanent watchdog ❗✅

**Symptom:** `insertBefore` on a container rendered by Solid works (the element
really does get into the DOM, no error in the log), but the player does not see it. The log shows
a full set of `onMount` entries, so nothing threw.

**Two causes, both present at once:**

1. **Solid re-renders the container** and, when swapping children, can remove
   nodes it did not create itself.
2. **The component mounts several times** during a single visit to the screen
   (`right-click unassign active` appears in the log several times). A new `onMount`
   can run **before** the previous one's `onCleanup` — so a freshly injected
   element is immediately removed by the old mount's cleanup.
   On top of that, `stop*()` operating on module-level singletons (`observer`, `element`)
   disconnects an observer that already belongs to the **new** mount.

**The pattern that works** (`tab-icons.js`, `factory-first.js`):

```js
function ensureThing() {
    const host = document.querySelector(HOST_SELECTOR);
    if (!host) return false;                       // not there yet — wait
    if (!host.querySelector(`.${CLASS}`)) host.insertBefore(build(), anchor);
    if (observedHost !== host) {                   // the watchdog stays attached
        observer?.disconnect();
        observedHost = host;
        observer = new MutationObserver(() => ensureThing());
        observer.observe(host, { childList: true });
    }
    return true;
}
```

- The observer **does not detach** after the first success — it stands guard and reinserts.
- There is no loop: your own insertion wakes the observer, which sees the element in place
  and finishes without changes.
- `start*()` has to be idempotent (it is called on every mount).
- `stop*()` **must not** be called from the tab's `onCleanup` — only on a full teardown.
  Cleanup then removes `.${CLASS}` via `querySelectorAll`, not via a remembered
  reference, because that may point at the previous mount's element.

**Recognizing it:** if the element is in the DOM but not visible on screen — this is the
problem, not CSS. The insertion log (`log('… added')`) settles it immediately: with this
fault it appears once and never again, even though the element vanished.

**⚠️ Update after an in-game test: for the Commerce screen's header bar this was NOT enough.**
A permanent watchdog on `[data-name="filter-and-sort"]`.parentElement had no effect either —
the checkbox never appeared once, despite no error in the log and despite `onMount`
running to completion. That particular bar is unusable for injection.

The watchdog pattern remains correct where it is **confirmed in practice**
(`tab-icons.js`, the tab bar `[data-name="TabList"]`, `settlement-controls.js`,
settlement card headers). It is not, however, a universal answer.

**A practical rule:** put your own controls in a container the mod creates and
owns — for us the `.najane-assign-bar` bar appended to the tab row. Then there is nothing to guard.
The "Factory first" toggle ended up behind a question mark in that bar, and the
`factory-first.js` module comes down to an element factory
function — zero observers, zero game selectors.

The order to try, from the most reliable: **(1)** append to your own container → **(2)** inject
into a game container with a permanent watchdog → **(3)** wrap the component via `ComponentRegistry`.

## 52. `Game.PlayerOperations.canStart` in a loop is the main cost of assigning ❗✅

**Symptom:** "Assign all" with a large pool runs very slowly at first and
speeds up as the pool empties.

**Cause:** the planner recomputes the best (resource, settlement) pair from scratch before every
assignment, and for every pair it calls `canStart` — i.e. a round-trip to the engine.
With 170 resources and 15 settlements that is 2550 calls per assignment and ~430,000 per one
"Assign all". The cost falls linearly with the pool, hence the speedup.

**Three fixes, in order of payoff:**

1. **Remember the engine's answers.** The answer changes only for the settlement that
   just took something (it lost a slot, or gained two after a camel). The rest of the table
   stays valid. Cache `${resourceValue}:${cityKey}` + invalidation **per settlement**.
   ⚠️ The key must be `resourceValue`, not the resource type: copies sit on different plots,
   and the connection to the trade network is a property of the plot.
2. **Score one instance per kind.** Scoring reads nothing but the resource type,
   so evaluating 8 copies of cotton is 8× the same work. Group the pool by `resourceType`,
   score a representative, and keep the remaining copies as substitutes in case
   the engine refuses that particular one.
3. **Do not query the database in a loop.** `Game.age === Database.makeHash('AGE_MODERN')` in a function
   called per pair means hundreds of thousands of hash queries. The age does not change during a game —
   compute it once and remember it.

**The general principle:** anything in a hot loop that reaches into `Game.*`, `GameInfo.*` or
`Database.*` should be computed once and remembered; if the result depends on state, invalidate
precisely what actually changed, not the whole cache.

## 53. A backtick in a CSS comment brings down the WHOLE mod ❗✅

**Symptom:** the mod stops loading entirely. In `UI.log`:

```
JS Error: fs://game/<mod>/ui/settlement-controls.js:70: SyntaxError: Unexpected identifier
SOURCE ERROR - fs://game/<mod>/ui/<entry point>.js
```

**Cause:** we keep styles in template literals. A backtick written
**inside** such a literal — even in a CSS comment, to quote a property name —
**closes the string**. The rest of the block is then parsed as code.

```js
const STYLE = `
/* setting `margin-left: auto` did not work */   ← the literal ends here
.foo { display: flex; }
`;
```

**❗ `node --check` does NOT catch this.** The resulting garbage is sometimes valid JavaScript for
Node, while the game engine rejects it mercilessly. The result: "it passes validation for me", and in game
nothing loads.

**In CSS comments use quotes, never backticks.** Ordinary JSDoc comments
(outside a literal) may contain them freely — only the inside of a literal matters.

**A guard in `deploy.sh`** (blocks deployment, tested on a planted file):

```bash
awk -v f="$file" '
    /^const [A-Za-z_]+ = `$/ { inside = 1; next }
    inside && /^`;$/         { inside = 0; next }
    inside && /`/            { printf "%s:%d  %s\n", f, NR, $0 }
' "$file"
```

It requires sticking to a convention: a style block starts with a line `const NAME = ` + a backtick and
ends with a line containing a backtick and a semicolon — both on their own.

**Round two, once the engine stops being the bottleneck:**

4. **Polling the model every 50 ms.** Every assignment waits until the model shows the resource in its new
   place — that interval is paid twice per resource. Dropping to one frame (16 ms) is,
   with a large pool, a difference of about a minute. Below that there is no point: the model rebuilds on
   a frame boundary.
5. **Waiting twice on `isSlottingAvailable`** — once at the end of an iteration, once at the start of
   the next. One is enough.
6. **The same pair computed several times in one pass.** `scorePair` is sometimes called from
   four branches of the same decision, `estimatedYieldBoosts` separately for priority,
   production and happiness — all for the same pair and all with the same result.
   Memoize **for a single planning pass**, keyed by `resource type : settlement`, cleared at
   the start of every pass (the board state changes between passes).

**And most importantly: measure, do not guess.** Two rounds of optimization in this module were based on
guesses and one of them missed. The loop now logs a breakdown into planning time
(ours) and time waiting for the game (not ours) — without that you do not know what to fix.

## 54. Do not drive a bulk operation through the Solid model — measured 30s vs. seconds ❗✅

Measured on 111 resources, "Assign all" driven through the screen's model:

```
assigned 111 resource(s) in 30663ms (1610ms planning, 16346ms waiting, 276ms each)
```

**A 5 / 53 / 41 split:** planning 1.6 s (5%), waiting for the model 16.3 s (53%),
and 12.7 s (41%) that fitted into no measurement at all. Those 41% were three model calls
per resource:

```js
model.clickAvailableResource({ resourceValue, cityID: undefined });
model.slotSelectedResource(cityID);
model.deselectSelectedResource();
```

Each of them mutates a `createMutable` store and triggers a **screen redraw**. So three
needless redraws per resource, and a fourth (the real one) paid for under "waiting",
because the loop waited for the **screen** to rebuild before it planned the next move.

**The principle:** a bulk operation should talk to the engine and read from the engine:

- sending: `Game.PlayerOperations.sendRequest` directly, not through the model's methods;
- confirmation: `Cities.get(cityID).Resources.getAssignedResources()` in a `requestAnimationFrame` loop;
- the next plan: from the game's API (for us `headless-model.js`), not from the screen's model.

The screen will refresh anyway — it listens to engine events — but nothing waits for it to finish.

⚠️ **Do NOT use the `ResourceAssigned` event for confirmation.** Engine events fire for
all players, so an AI assigning something on the other side of the map will release the wait
too early, and the next plan will be built on an unchanged board. Asking the settlement directly is
immune to somebody else's turn.

**A welcome side effect:** both paths (the on-screen button and the automation with the screen
closed) become one function, because both read the same source.

**What to watch out for when moving to API data:** the screen's model had `yieldDeltas`
and building properties precomputed; computing them yourself means recomputing them once per resource.
Walking `city.Constructibles.getIds()` on every iteration is a real cost — buildings do not appear
during assignment, so the result can be kept in a cache for the duration of the pass and
invalidated only for the settlement that just took something.

**Round three — after moving to the game's API, planning gets MORE expensive, and it is clear why.**

```
through the screen's model:  30663ms (1610ms planning, 16346ms waiting)  276ms/resource
through the game's API:      13169ms (4739ms planning,  8422ms waiting)  119ms/resource
```

Planning went up 3×, because the screen's model had settlement yields **already computed**, and when
computing them yourself it is easy to reach for the most expensive function:

⚠️ **`CityYields.getCityYieldDetails(cityID)` is NOT a read of numbers.** It builds the tree
that the yields tooltip shows: base values, modifier steps, localized
labels. Called for every settlement before every resource, it was the bulk of the planning
time. The cheaper equivalent reads exactly what that function reads before it decorates:

```js
const yields = city.Yields?.getYields();          // an array indexed like GameInfo.Yields
yields?.forEach((entry, index) => {
    const definition = GameInfo.Yields[index];
    if (definition) totals.set(definition.YieldType, Number(entry.value) || 0);
});
```

The second sin of the same class: the screen's model keeps yields as `yieldIconSrc`
(`url(${UI.getIcon(type,'YIELD')})`) plus a number, and the scoring maps the icon back to
a yield type. Reproducing that shape in data you build yourself means
building strings only to parse them immediately afterwards. Better to supply a ready-made map and
let the scoring take it directly.

**Confirming operations: `setTimeout` every 4 ms, not `requestAnimationFrame`.** The engine
processes its queue on its own tick; a check synchronized to the frame can miss
the moment by almost a whole frame — 16 ms per resource, for free.

**Where the floor is:** after these changes most of the time is the engine processing the
operations itself. Going lower requires sending further assignments without waiting for the
previous ones, and that means planning on a stale board — i.e. losing the balancing
(happiness, factories). That is a design choice, not an optimization.

## 55. A multi-line tooltip: `\n` and `[N]` are not enough — you need `white-space` ❗✅

**Symptom:** `data-tooltip-content` with line breaks displays as a single paragraph.
Neither `\n` nor the Firaxis `[N]` marker makes a difference, and `[N]` **does not show up
literally** — so it looks as if it were processed and ignored.

**The cause** (`core/ui/tooltips/tooltip-controller.js`, `render()`):

```js
this.textElement.innerHTML = Locale.stylize(content);   // textElement is a bare <div>
```

`Locale.stylize` turns `[N]` into a **newline character**, not into a `<br>`. In HTML a newline
collapses into a space like any other whitespace. A bare `<div>` has no
`white-space: pre-wrap`, so the break is not visible.

**The fix — both halves:**

```js
lines.join('\n')            // plain newline characters in the content
```
```css
.tooltip__content > div { white-space: pre-wrap; }   /* this is where the tooltip's text lands */
```

⚠️ The content alone will not force a break — the CSS has to **allow** it. The rule applies globally to
text tooltips, so ship it with your screen's stylesheet rather than permanently.

#### ⚠️⚠️ And the reverse problem: `[N]` comes back as a `<p>`, so it cannot be un-broken with a regex for `<br>` ✅ (2026-09-08)

Measured from `UI.log`, not inferred. `Locale.stylize` on a string containing `[N]` returns each
segment wrapped in **its own paragraph**:

```html
<p cohinline>+2<fxs-font-icon data-icon-id="YIELD_PRODUCTION" data-icon-context="icon"></fxs-font-icon></p>
```

So a mod that wants two figures **side by side** — one label pill rather than a pill twice as tall
— cannot get there with `white-space: nowrap` (two blocks stack regardless) nor by stripping
`<br>` and newlines (there are none). Join the paragraphs instead:

```js
markup.replace(/<\/p>\s*<p\b[^>]*>/gi, ' ')
```

⚠️ `cohinline` is Gameface's own attribute and does **not** make the element behave inline here —
the two paragraphs stacked with it present. Do not read it as a promise.

⚠️ This is not a contradiction of the tooltip note above: that path hands `stylize` a string the
controller then drops into a bare `<div>`. Check what you actually get with a `warn` before
writing a regex against it — a `<br>`-only regex silently changed nothing for a whole round here.

Tooltips with `data-tooltip-component` take a different path (a custom element receives the content in an
attribute) and do not need this rule.

## 56. `node --check file.js` does NOT check ES modules — it silently passes ❗✅

**This invalidates the verification method used earlier in this project**, including the note
in quirk #53 that "Node accepts that garbage". It does not — it does not parse it at all.

`node --check <file>.js` treats the file as **CommonJS**. On seeing `import` it gives up and
exits with code **0**, without saying a word. Every UI file of a mod is an ES module, so
a full "syntax ok" from that command meant nothing.

Tested on a file with a deliberately inserted newline inside a literal:

```
node --check ui/screen/tab-icons.js                 -> exit 0, silence
node --input-type=module --check < ui/screen/tab-icons.js -> exit 1, points at line 23
```

**The correct command** reads from standard input:

```bash
node --input-type=module --check < file.js
```

A parser invoked this way catches **both** known killer errors: a stray backtick in
a template literal (quirk #53) and a newline inside a `'...'` literal — and it reports
exactly the same message that later appears in `UI.log`.

⚠️ **The check has to stand on the road to the game, not in the editor's habits.** The second half of this
mishap: the file was checked by hand, then changed again and deployed without
a re-check. `deploy.sh` now checks every file itself and refuses to deploy.

**Beware of intermediate tooling:** a `\n` sequence typed into a script passed through
a shell layer can be turned into a real newline before it
reaches the file — that is how both broken files that day came about. When generating code with
escape sequences, assemble them programmatically (`chr(92) + 'n'`) or use a file editor
instead of a heredoc.

## 57. Moving a node is a mutation — with a `MutationObserver` it hangs the game ❗✅

**Symptom:** entering the screen hangs the whole game (not an error, not a blank screen — a freeze).

**Cause:** a decorator called from a `MutationObserver` moved someone else's element to the end
of a row **on every pass**:

```js
row.appendChild(leader);        // unconditionally
```

`appendChild` on a node that is already there **is not a no-op** — it is a removal
and an insertion, i.e. a `childList` mutation. The observer sees it, calls the decorator, which again
moves it. An infinite loop on the UI thread.

**The fix — two layers:**

```js
if (row.lastElementChild !== leader) {   // 1. do not touch it when it is already in place
    row.appendChild(leader);
}
```

```js
let decorating = false;                  // 2. a safeguard in case something is missed
function decorateAll() {
    if (decorating) return;
    decorating = true;
    try { /* … */ } finally { decorating = false; }
}
```

**The principle:** every function called from a DOM observer must be *idempotent with respect to the DOM* —
on a second call against the same state it must change nothing. This applies to
`classList.add` and `setAttribute` too, if the observer watches attributes (with `childList:
true` — it does not).

The same pattern works correctly in `settlement-controls.js` (the `injecting` flag) — the new code
simply did not repeat it.

## 58. Cutting code out by a "from function to function" range ❗✅

When rewriting one function with a script, it is easy to cut out everything between it and the next
reference point:

```python
start = s.index('function decorate(card) {')
end   = s.index('let decorating = false;')     # and 4 more functions lay in between
s = s[:start] + new + s[end:]
```

`updateMeasuredLayout`, `scheduleRemeasure` and the attempt counter disappeared. **The file still
parsed** — missing functions are a runtime error, not a syntax one — so `node --check` and
`deploy.sh` passed it without a word. In game: a `ReferenceError` on first use and the whole
mod having no effect.

**The safeguard:** after every scripted cut, count what is left:

```bash
grep -n "^function \|^let \|^const [a-z]\|^export function" file.js
```

or a simpler scan: collect the names declared in the file plus the imported ones and check that
every call is covered. This is the same check that, during the structural rebuild
(quirk #56), caught `settlementYieldTotal` and `modifierApplies`.

⚠️ A syntax check **does not replace** this one. It detects a broken file, not a broken
module.

## 59. `ui-next` tooltip autolock works ONLY for nested ones ✅

**2026-08-28.** The game has a complete mechanism for locking a tooltip automatically when the
cursor is held still — a progress bar filling at the bottom of the frame, a sound, taking over the
input context — and hooks it up to **one** case. In
`core/ui-next/components/tooltip-model.js`, `tryStartAutoLock` has exactly one call
site: inside `triggerTooltip`, in the `shouldNest` branch.

```js
if (!shouldNest) { setActive([name]); }
else { … ; tryStartAutoLock(name); }
```

And `shouldNest` requires the parent to **already be locked**:

```js
shouldNest = isRaisingSiblingTooltip
    || (currentTopList.includes(name) && (isLocked(currentTop) || IsTouchActive()));
```

❗ **The conclusion: a top-level tooltip never locks itself.** It is locked only by the
`keyboard-inspect-tooltip` / `toggle-tooltip` input action, handled in
`TooltipContentComponent.onEngineInput` → `tooltipModel.lock()`. Autolock only applies to
a child opened inside a locked parent.

To add this behavior to a tooltip of your own (Better City UI does it in
`ui/screen/tooltip-autolock.js`), do not touch the model — it is a singleton shared by
the whole game, and its `tryStartAutoLock` is private. Instead, in a component embedded
inside your own `<Tooltip>`:

```js
const ctx = useContext(TooltipContext);          // exported from components/tooltip.js
const model = TooltipModel.get();
// after Configuration.getUser().tooltipAutolock has elapsed:
if (top === ctx.name && !model.isLocked(ctx.name) && ctx.childTooltipList().length > 0) {
    model.lock();                                 // it plays the sound and takes over input itself
}
```

⚠️ **`lock()` refuses a tooltip with no children.** It checks
`childTooltipTable()[name]().length > 0` and returns `false` — there is no point locking something you
cannot enter. The autolock timeout ends the same way. Do not start a progress bar that
will never reach the end.

⚠️ **Gate on the PLAYER's settings, not your own.** `model.isAutolockAvailable()` already includes
`Configuration.getUser().tooltipAutolockEnabled`, `UI.isMouseAvailable()` and the rejection of
touch; `Configuration.getUser().tooltipAutolock` is the delay slider, and a value `<= 0`
means "lock immediately".

⚠️ **The progress bar is a game element and carries no `data-l10n-id` at all.**
`Tooltip.Frame` appends `Tooltip.InspectHint` as its last child, and the bar is the last
child of that hint — i.e. `frame.lastElementChild.lastElementChild`. It renders
only when `tooltipCount() > 0`. The animation is switched on by the `tooltip-autolock-progress` class from
`core/ui/tooltips/tooltip-manager.css` plus `style.animationDuration`. Protect yourself with
a class check (`bg-secondary`), so that after a game patch you do not animate some random element.

⚠️ **`@keyframes tooltip-autolock-frame` is EMPTY** in the shipped CSS — the
`tooltip-autolock-frame` class the game adds to the frame for the last 300 ms does nothing
visible. There is nothing to reproduce.

⚠️ **Do not look for the frame via a `ref` on `Tooltip.Frame`.** Solid's `spread` formally
supports a `ref` that is a function, but through `ComponentRegistry` that reference silently does not
arrive — the lock worked, and the bar never appeared once with nothing reporting it.
Render your own hidden element in the frame's `children` and walk up via `parentElement`: an element
drawn inside cannot be wrong about where it is.

⚠️ **The bar is sometimes absent even though everything is correct.** `Tooltip.InspectHint` renders
it only once the tooltip HAS nested children, and those mount a frame after the frame itself.
One retry inside `requestAnimationFrame` is needed, minus the time already spent —
a bar starting from zero after the countdown began promises more time than is left.

## 60. `SpriteGrid.addSprite` accepts `alpha` — contrary to what most of the code suggests ✅

**2026-08-28.** Almost every `addSprite` call in the game looks like this:

```js
this.yieldVisualizer.addSprite(district.location, iconURL, { x, y: 24, z: 0 }, { scale: 0.9 });
```

— and it is easy to conclude from that that the parameters are only `offset` and `scale`. **That is not true.**
`base-standard/ui/lenses/layer/worker-yields-layer.js` dims a blocked specialist pip:

```js
{ scale: offsetAndScale.scale, alpha: info.IsBlocked ? SPECIALIST_PIP_BLOCKED_ALPHA : 1 }
// SPECIALIST_PIP_BLOCKED_ALPHA = 0.5
```

⚠️ **There is, however, no tint and no rotation** — in the whole game's codebase and in all installed mods
a sprite never gets a color. Color is accepted by `addText` (`fill`, `stroke`) and by
`YieldChangeVisualizer.addYieldChange(data, location, offset, color)`, where the color is ARGB
(`0xff52ff46` for a recommended tile, `0xffffffff` for a normal one). A sprite — no.

**The practical consequence:** a "shadow" under an icon cannot be a darkened copy of that icon. All you can
do is a backing plate from an existing asset, dimmed with `alpha`. And `BUILDING_EMPTY` — the only
natural backing plate under building icons — is a **ring, not a filled circle**: drawn
noticeably larger, offset and opaque, it reads as a second outline around the icon, not
as depth beneath it.

⚠️ Reference values from the game (`building-placement-layer.js`, `realizeBuildSlots`): a building icon
`scale: 0.9`, an empty slot `scale: 0.8`, `buildSlotSpritePadding = 16`, position `{ x, y: 24, z: 0 }`.

⚠️ **The sourcemaps are on disk and contain the original TypeScript.** Every `*.js.map` in
`Base/modules/` has a `sourcesContent` with the full, commented `.ts` source — an order of
magnitude better to read than the compiled output:

```bash
python3 -c "import json,sys; print(json.load(open(sys.argv[1]))['sourcesContent'][0])" file.js.map
```

## 61. A decorator MUST have all four lifecycle hooks ❗✅

**2026-08-28.** `Controls.decorate` takes an object, and `component-support.js` calls
**all four** methods on it unconditionally:

```js
d.beforeAttach();   // component-support.js, doAttach()
d.afterAttach();
d.beforeDetach();
d.afterDetach();
```

A decorator with only `afterAttach` throws:

```
TypeError: d.beforeAttach is not a function
    at c.doAttach (fs://game/core/ui/component-support.js:289:11)
    at connectedCallback (fs://game/core/ui/component-support.js:332:14)
```

❗ **The exception is thrown while the panel is being attached and aborts its ENTIRE initialization** — not just
your decoration. In Better City UI this took out the whole production list: empty sections, the
vanilla Production/Purchase tabs came back, and no message from the mod in `UI.log` other than that one
`JS Error`. It looks like a data error, and it is a contract error.

⚠️ Empty methods are enough and **cannot be omitted**:

```js
class MyDecorator {
    constructor(component) { this.component = component; component.myMod = this; }
    beforeAttach() { }
    afterAttach() { /* … */ }
    beforeDetach() { }
    afterDetach() { }
}
```

⚠️ A `try`/`catch` around your own `installX()` will **not** catch this — the installation
(`Controls.decorate`) succeeds, and the exception only appears on the first
attachment of the element, inside the game's code. Search for `JS Error:` in `UI.log`, not for the mod's prefix.

## 62. `localStorage` in a mod: it does not survive a reload, and City Hall wipes the whole store ❗✅

**2026-08-28.** Two independent reasons why `localStorage` should **never be the source of truth** for a
mod's state:

1. on its own it did not survive a reload — verified with settlement priorities in Better Commerce
   Screen UI;
2. several mods clear the store **entirely**, wiping out every other mod's entries along the way.

### ⚠️ CORRECTION, 2026-09-03: the trigger is YOUR OWN key, and the fix is `modSettings`

The wipe is not arbitrary and it is not only City Hall. The published pattern is:

```js
save(modID, optionID, value) {
    UI.setOption("user", "Mod", `${modID}.${optionID}`, value);
    Configuration.getUser().saveCheckpoint();
    if (localStorage.length > 1) {                  // <-- ANY second top-level key
        console.warn(`ModOptions: erasing storage (${localStorage.length} items)`);
        localStorage.clear();                       // <-- takes modSettings with it
    }
    const options = JSON.parse(localStorage.getItem("modSettings") || "{}");
    options[modID] ??= {};
    options[modID][optionID] = value;
    localStorage.setItem("modSettings", JSON.stringify(options));
}
```

✅ Read verbatim in `bz-city-hall`'s `ui/options/mod-options.js` and in Leugi's
`core/settings.js`; reported in the wild for Memento Editor, More Diplo Ribbon, Policy Yields
Preview, Enhanced Town Focus Info and Advanced Options Menu Tweaks too.

⚠️ **So a private top-level key is not "keeping out of the way" — it is the trigger.** The
condition is `length > 1`, so ONE key of your own is enough: the next time any of those mods saves
anything, it erases `modSettings` and every mod that cooperates loses its settings. The mod that
gets blamed is not the one that called `clear()`.

⚠️ **The convention is one shared key with a namespace per mod**, and every write must
read-merge-write:

```js
const shared = JSON.parse(localStorage.getItem('modSettings') || '{}');
shared[MOD_ID] = { ...(shared[MOD_ID] ?? {}), ...mine };
localStorage.setItem('modSettings', JSON.stringify(shared));
```

Never `setItem('modSettings', mine)` — replacing the object destroys the other mods the same way
`clear()` does, just more quietly.

⚠️ **Migrating off an old private key: DELETING it is the half that matters.** Carrying the values
across is a courtesy; a leftover key goes on making `length > 1` for ever.

Found the expensive way: `better-city-ui` shipped three private keys and
`better-commerce-screen-ui` two, all five now folded into `modSettings`.

⚠️ The durable channel is `UI.setOption('user', 'Mod', key, A NUMBER)` + `saveCheckpoint()` — see
quirk 50. Keep `localStorage` as a mirror that is only read when the options channel
does not answer.

⚠️ **The options channel accepts NUMBERS only.** A consequence that is easy to forget: keys
**cannot be enumerated** (there is no "give me all my entries"). Every query has to be about
a key you already hold. In practice that is enough, because you ask about what the screen is
currently showing.

⚠️ `Catalog` / `SerialObject` accepts strings, but **writes into the game save** — for a mod with
`AffectsSavedGames = 0` that is not an option.

## 63. `getCityYieldDetails()` prunes whole branches of the yield tree ❗✅

**2026-08-28.** `CityYields.getCityYieldDetails()` has its own `removeBlankChildren`: if
any step in a branch has no label, **the whole branch disappears**. For Food and Influence the result is
empty — not because nothing produces them, but because somewhere along the way there is an unnamed step.

⚠️ The second source, `CityDetails.yields`, **is not populated** — measured in game: length 0 with
a settlement selected. Relying on it means depending on whether another model has managed to refresh.

⚠️ The approach that works: walk the `CityYieldNodes` paths yourself, the way
`model-city-details.js` does (`addYieldSteps` / `addYieldsToHierarchy`), including the "all
sub-steps have the parent's id" branch. No pruning, no other model, no shared state.

## 64. Searching `GameInfo.Modifiers` easily becomes quadratic ❗✅

**2026-08-28.** The natural shape of code that reads modifiers is quadratic and looks
innocent:

```js
for (const modifier of GameInfo.Modifiers) {                       // thousands of rows
    const dynamic = GameInfo.DynamicModifiers                      // ...times thousands
        .find((row) => row.ModifierType === modifier.ModifierType);
    const args = GameInfo.ModifierArguments                        // ...and again
        .filter((row) => row.ModifierId === modifier.ModifierId);
}
```

The same goes for `RequirementSetRequirements` → `Requirements`. Every installed mod adds
rows to both sides of the multiplication.

⚠️ The fix is always the same: **one linear pass per table**, the result into a `Map`, built
once. In Better City UI that is `ui/engine/modifier-index.js`
(`ModifierId → effect`, `ModifierId → arguments`, `RequirementSetId → requirement types`,
`ConstructibleType → ModifierId[]`).

⚠️ **`Modifiers` does NOT have an `EffectType` column.** The effect sits in `DynamicModifiers`, mapped by
`ModifierType`. Reading `modifier.EffectType` returns `undefined` and **every** lookup comes out empty
silently — no exception, no entry in `UI.log`, simply zero results.

⚠️ Anything built from `GameInfo` has to be keyed by the **age** (`Game.age`): moving to
the next era swaps the tables underneath every index.

---

## 59. `sendRequest` QUEUES — `canStart` right after it lies ❗✅

**Date: 2026-08-18.** Symptom: you click a button, the mod redraws the UI, and **everything looks
identical** — the same numbers, the same active button. It looks like "the redraw does not
work", and the redraw works perfectly; it is simply a faithful repainting of the **old state**.

```js
Game.PlayerOperations.sendRequest(playerId, opType, args);   // ← only QUEUES
Game.PlayerOperations.canStart(playerId, opType, args, false); // ← still Success: true
```

`sendRequest` **does not execute** the operation — it puts it into the game core's queue. For a frame or
two, `canStart`, costs, limits and everything else read from the engine describe the state **before**
the request. A redraw inside `requestAnimationFrame` falls inside that window.

### ✅ Two passes, not one

1. **Immediately** — repaint what you know on your own side (e.g. "the proposal is in flight,
   so the button should go dark"). That is true right away and gives the player a reaction to the click.
2. **After `GameCoreEventPlaybackComplete`** — only then does the engine know the new state. This is
   **the game's own pattern**: on `DiplomacyEventEnded` / `DiplomacyQueueChanged`,
   `panel-diplomacy-actions.js` **does not refresh**, it only sets a `needsRefresh` flag, and calls the actual
   `checkRefesh()` from `GameCoreEventPlaybackComplete`.

⚠️ That event fires **very often** and about everything — listen with a flag armed on
the click, so that outside that window it costs one `if`.

⚠️ Remembering the answer on your own side **is not guessing the rules**, as long as you reproduce the
answer the engine will give in a moment anyway, and use **its own** reason key (for us
`LOC_DIPLOMACY_ACTION_FAILURE_DUPLICATE_PROJECT`, through the same substitution table). Otherwise the
player will see two different sentences about one situation.

⚠️ You have to know **when to forget it**. For us a diplomatic action has `BaseDuration="0"`,
i.e. it resolves at the end of the turn in which it was submitted — so the memory is cleared by
`LocalPlayerTurnBegin`. A refusal also frees the action and also happens on a turn boundary.

## 60. `Controls.decorate` calls the factory FOR EVERY INSTANCE — a prototype patch needs a lock ❗✅

**Established 2026-08-26** while analyzing `bz-city-hall` (the city screen, see [28](28-city-screen.md)).

The natural place for a prototype patch is the decorator's constructor — and that is a trap. The city
panels (`panel-city-details`, `panel-production-chooser`) **are created and destroyed on every
opening of a settlement**, and `Controls.decorate` calls its factory once per instance. Without a lock the
prototype is wrapped anew on every opening and the chain of wrappers grows without end.

The symptom is nasty, because it **does not appear straight away**: after a dozen or so minutes of play the same method
runs dozens of times per call and the game starts to choke. Nothing in the logs.

✅ The pattern from City Hall — a static field as the lock, plus a bridge from the prototype to the
decorator instance:

```js
class myDecorator {
    static c = null;
    constructor(component) {
        this.component = component;
        component.myMod = this;             // the bridge: the prototype method reaches the decorator
        this.patchPrototype(Object.getPrototypeOf(component));
    }
    patchPrototype(proto) {
        if (myDecorator.c) return;          // ← without this the chain grows
        const c = myDecorator.c = { proto };
        c.update = proto.update;
        proto.update = function(...args) {
            const r = c.update.apply(this, args);
            this.myMod.afterUpdate(...args);
            return r;
        }
    }
}
```

⚠️ Name the bridge uniquely (`myMod`, not `component.mod`) — City Hall occupies `bzCityHall`.

## 61. Two mods replacing the same game file: one loses WITHOUT A TRACE ❗✅

**Established 2026-08-26.** An extension of [#10](#10-importfiles-overrides-game-files-by-path-).

`ImportFiles` with a file at a game file's path replaces it for all consumers. When
**two mods do it at once**, one copy wins and the other's changes simply do not exist —
**no error in `Modding.log`, `Database.log` or `UI.log`**. The mod loads, it is on the list of
enabled mods, its scripts run and do nothing.

✅ This is why the authors of mature mods put the risky part into a **separate action group** with
a `<Criteria>` that switches it off when a competitor is detected:

```xml
<Criteria id="production-ok">
    <ModInUse inverse="1">compact-production</ModInUse>
    <ModInUse inverse="1">drongos-compact-production</ModInUse>
</Criteria>
```

⚠️ `<ModInUse>` compares **the mod's id from its own `.modinfo`** — not the folder name, not the
display name.

⚠️ A replacement freezes the file at the game version it was copied from — a Firaxis patch will not reach
the player until the author copies the file again. `bz-city-hall` carries a **1441-line copy**
of `panel-production-chooser.js` with a 16-line diff this way.

**The conclusion for your own mods:** if a replacement is only meant to reorder an import or
add `data-*` attributes — avoid it anyway, and if you must, wrap it in its own action
group with a criterion. For everything else a decorator or a prototype patch is enough.

## 62. City Hall WIPES the whole `localStorage` — for every mod ❗✅

**Established 2026-08-27** by reading `bz-city-hall/ui/options/mod-options.js`. That same file
is shared by the author's mods (`bz-city-hall`, `bz-map-trix`, `bz-…`).

```js
save(modID, optionID, value) {
    UI.setOption("user", "Mod", optionName, value);
    Configuration.getUser().saveCheckpoint();
    if (localStorage.length > 1) {
        console.warn(`ModOptions: erasing storage (${localStorage.length} items)`);
        localStorage.clear();                                    // ❗ EVERYTHING, every mod's
    }
    const storage = localStorage.getItem("modSettings") || "{}";  // → "{}" after clear()
    const options = JSON.parse(storage);
    options[modID] ??= {};
    options[modID][optionID] = value;
    localStorage.setItem("modSettings", JSON.stringify(options)); // only this one value
}
```

**Every option change in City Hall**, when `localStorage` holds more than one key,
**wipes every mod's data** — and **City Hall's own remaining options too**, because after `clear()`
it rebuilds `modSettings` from an empty object.

`localStorage` in Civ VII is **shared by all mods** (a single `fs://game` origin
in `LocalStorage.sqlite`), so one mod's `clear()` reaches every other one.

⚠️ **This most likely explains the observation in #50** ("localStorage did not survive a reload"):
it did not survive because someone cleared it — not because it is not durable.

⚠️ **An observation without an explanation:** on this machine the row under the key
`najane-commerce-merchant-orders` contains, besides its own data, the settings of
`repair-shop-plus`, `drongos-cheat-panel`, `f1rstdan-cool-ui`, `better-commerce-screen-ui`
and `najane-common-specialists-yields` — in an **older snapshot** (turn 55) than the one in `modSettings`
(turn 97). So the contents of one key ended up under another. ❓ The mechanism is not established:
either the `localStorage` shim served a value from the wrong key, or two writes are racing.
**Do not waste time debugging your own mod before you have ruled this out.**

**What to do about it in your own mod:**
- keep your data under **your own key**, never in `modSettings` — that is the one City Hall
  rebuilds from scratch;
- ⚠️ expect `clear()` to take it anyway. Design so that losing the data
  means "the setting went back to its default", not a corrupted state;
- **never call `localStorage.clear()`** in your own code.

## 63. `getCityYieldDetails` WIPES a whole yield tree because of one unlabeled step ❗✅

**Established 2026-08-27** on my own mod: the yield-breakdown tooltip worked for gold
and production, while for **food and influence it showed only the title**, without a single row.

The cause is in `base-standard/ui/utilities/utilities-city-yields.js`, in the function
`removeBlankChildren`, whose own Firaxis comment reads:

> 'Blank' nodes are nodes without a label or icon to convey what exactly they mean.
> To provide concise information, nodes with blank children have **_all_** their children removed.
> To prevent situations where this is too deep a cut, these nodes should be correctly labeled in GameCore.

```js
removeBlankChildren(root) {
    for (const data of root.childData) {
        if (!data.label && !data.showIcon) { root.childData = []; }   // <- the WHOLE list
    }
    ...
}
```

❗ **One unlabeled step anywhere under a yield wipes its entire tree.** Not just that
step — all of its siblings along with it. The game itself admits this is "too deep a cut".

✅ **The workaround: a second path.** The game builds the same tree twice, with different code:

| Source | Prunes? | Shape |
|---|---|---|
| `CityYields.getCityYieldDetails(cityID)` | ❗ yes, `removeBlankChildren` | `{label, value, valueNum, valueType, type, isNegative, isModifier, childData[]}` |
| `CityDetails.yields` (`ui/city-details/model-city-details.js`) | ✅ no | `{name, value, icon, iconContext, children[]}` |

`CityDetails` walks the `CityYieldNodes` paths and has no such cleanup — it is what
feeds the Yields tab in the city details. Take `CityYields` as the basis, and when it returns
an empty tree for a yield that is not zero, swap the children for those from `CityDetails`.

⚠️ After such a swap you **have to drop the unlabeled nodes yourself** (and lift their children
one level up) — because they are exactly the reason the first path deleted them.
Otherwise you get empty rows: the window grows, there is no text.

⚠️ And drop the duplicates. The engine describes one building with three nodes — the building itself,
a "Base" with the same value, and an `x 1.0` multiplier with the same name — i.e. three rows for
one piece of information. A multiplier equal to 1 changes nothing by definition, and an only child carrying the
parent's number **is** the parent.

⚠️ Resources in supply arrive as **one node per resource, all with the same description** —
without merging siblings with identical labels the reader sees "Resources in supply +7"
directly above "Resources in supply +4" and has no way to guess what is going on.


## 64. The building you are currently constructing is NOT on the production list ❗✅

`GetConstructibleItemData` (`production-chooser-helpers.js`) returns `null` when the operation's result
carries `InQueue` and `hideIfUnavailable` is set — and it is set in **every normal
view** (`hideIfUnavailable: !showIfAvailable`, where `showIfAvailable` requires `viewHidden`).

```js
// lines 223 and 295 in production-chooser-helpers.js
if (operationResult.Success || insufficientFunds || !hideIfUnavailable || …) { … }
if (!hideIfUnavailable || insufficientFunds && possibleLocations.length > 0) { … }
```

Consequences for a mod:

- **You cannot "move" the row of a building under construction** into your own section — there is nothing
  to move. `panel.itemElementMap` does not contain that type. Your own block has to **create**
  the `production-chooser-item` elements itself and set their attributes itself.
- A "pinned at the top" tile in sorting (`getQueuedPositionOfType(hash) !== -1`) will **almost
  never work for buildings in the queue**. It only hits repairs, which have a separate path
  and stay on the list.

⚠️ There is no API returning the **remaining** production. All the game has is a percentage:

```js
const full = city.Production.getConstructibleProductionCost(type, FeatureTypes.NO_FEATURE, false);
const percent = city.BuildQueue.getPercentComplete(queueNode.type);   // 0-100, by node HASH
const remaining = Math.ceil(full * (1 - percent / 100));
```

⚠️ Two different keys in the same object: `getPercentComplete` takes the **hash** (`node.type`),
while `getTurnsLeft` takes the **type string** (`node.constructibleType` → `def.ConstructibleType`).
Confusing them throws no error, it just returns 0 / -1.

### Purchasing a building under construction

`Construct(city, item, true)` reads **only** `type` and `interfaceMode` from `item`. For
a constructible in the queue the engine answers `InProgress` and supplies `result.Plots[0]`, so the purchase
**finishes the existing construction** instead of asking for a new tile. So this is enough:

```js
Construct(city, { type, interfaceMode: 'INTERFACEMODE_PLACE_BUILDING' }, true);
```

### Your own section in the accordion

`panel.productionAccordion` (`fxs-vslot`) holds each category's `section.root`. Your own block
is inserted with `accordion.insertBefore(mine, panel.productionCategorySlots['buildings'].root)`.
To make it look native, repeat the section's classes: `production-category mb-2 ml-4` on the root and
`relative flex items-center h-10 mb-2 hud_sidepanel_list-bg` on the header.

⚠️ `ProductionKind` is an **engine global**, not an import — `city-banners.js` uses it without
any `import`. The same goes for `FeatureTypes`, `YieldTypes`, `OrderTypes`.

## 65. The right button is `engine-input`, not `contextmenu` — and it must be FINISH ❗✅

The game does not send `contextmenu`. Every button goes through its own input layer and arrives
as an `engine-input` event:

```js
inputEvent.detail.name    // 'mousebutton-right' | 'shell-action-1' (gamepad) | 'mousebutton-left' | 'accept' | …
inputEvent.detail.status  // InputActionStatuses.START | .FINISH
```

⚠️ **`FINISH`, not `START`.** A reaction to the press fires **a second time** on release.
With a toggle (hide/show) that gives two toggles and looks as if nothing happened.

⚠️ In `ui-next` components **`Activatable` intercepts** the left button, `accept`, `touch-tap`
and `keyboard-enter`, and **passes everything else** to the `on:engine-input` prop:

```js
// core/ui-next/components/activatable.js
props["on:engine-input"]?.(inputEvent);
```

`ChooserItem` does `mergeProps(props, {…})` and does not define that key, so the prop passes
through it unchanged. This is the **only** route to the right button in a row based on `ChooserItem`
— a plain `addEventListener('engine-input')` on the host is not enough, because `Activatable` calls
`stopPropagation()` on what it handles itself.

The pattern (our `production-item.js`):

```js
createComponent(ChooserItem, {
    …,
    'on:engine-input': (event) => {
        const d = event?.detail;
        if (d?.name !== 'mousebutton-right' && d?.name !== 'shell-action-1') return;
        if (d.status !== InputActionStatuses.FINISH) return;
        event.stopPropagation();
        event.preventDefault();
        // …
    },
});
```

`InputActionStatuses` is an engine global (no import), just like `ProductionKind`.

## 66. A set of strings per city WITHOUT writing to the save — one option per pair ❗✅

`UI.setOption('user', 'Mod', key, value)` takes a **number**. A set of types (strings) will not
fit into one value, and `Catalog` / `SerialObject` from `utility-serialize.js` takes
strings but **writes into the save file** — so it is out with `AffectsSavedGames = 0`.

The solution: **one option per (settlement, type) pair**:

```js
`${MOD_ID}.hidden.${gameSeed}.${cityId}.${TYPE}` = 1 (hidden) | 2 (restored)
```

⚠️ **Keys cannot be enumerated** through this channel — `UI.getOption` only answers about a specific
key. You can live with that when the set of questions is known from another source (for us: the types
the production list is showing anyway). For design purposes: if you have to **enumerate** the contents,
this channel will not work.

⚠️ **"Restored" must have its own code (2), not the absence of an entry.** There is no reliable "unset", and a missing
value must keep meaning "never chosen" — otherwise a restore is indistinguishable from the
initial state as soon as you restart.

⚠️ The match key is `Configuration.getGame().gameSeed`, because the numeric part of a settlement's `ComponentID` is
unique **only within one game**.

`localStorage` stays as a mirror, never as the source of truth — see [#62] (City Hall wipes the whole
`localStorage`) and the experience from Better Commerce Screen UI, where `localStorage` alone did not survive
a reload.

## 67. A `ui-next` tooltip on an OLD-framework element — `Tooltip.Trigger` does not wrap ❗✅

The key property that makes it possible to join the two systems
(`core/ui-next/components/tooltip.js`, `TooltipTriggerInternal`):

```js
if (resolvedChildren instanceof HTMLElement) {
    element.addEventListener('mouseover', props.onShowTooltip);
    // …
    props.setRoot(element);       // THIS element is the anchor
    setNeedsWrapper(false);       // nothing wraps it
}
```

When the child of `Tooltip.Trigger` is a **single `HTMLElement`**, the component merely **attaches
listeners to it** and uses it as the anchor. The element keeps its identity, classes and all the handlers
the game put on it — only its parent changes (Solid inserts it where the component
renders).

The bridge pattern (our `queue-tooltip.js` + `queue-decorator.js`):

1. `defineLegacyComponent('my-tip', { attrs: { 'data-anchor': null, … } }, …)`.
2. The element **cannot be passed in an attribute** → a `Map(id → element)` registry, with the id going into
   `data-anchor`. Register **before** inserting the host into the DOM — the component reads the anchor on its
   first render, and a `Trigger` without an element creates a wrapper of its own.
3. `parent.replaceChild(host, card)` — Solid will pull the card inside itself.
4. `onCleanup(() => anchors.delete(id))`, because the old framework rebuilds elements through
   `Databind.for` on every model update.

### ❗ CORRECTION — this pattern MUST NOT be used on nodes with `data-bind-for`

The bridge above works only when the element passed as `Tooltip.Trigger`'s child
**belongs to you**. On cards generated by `data-bind-for`, moving the node
**breaks the binding**: the tooltip shows, but the panel stops refreshing — after a reorder the
icons and positions stay stale. The engine holds references to the nodes it
generated itself, and after the swap it can no longer rearrange them.

The safe pattern (our `queue-decorator.js`): **one** tooltip for the whole panel, attached to
the panel's root (outside the bound subtree), `position: fixed`, `pointer-events: none`,
positioned from `card.getBoundingClientRect()`. `Tooltip.Trigger` gets **no**
element child, so it builds its own wrapper — and it is that wrapper which is stretched and fed
**synthetic** events from a listener on the card:

```js
trigger.dispatchEvent(new MouseEvent('mouseover'));
trigger.dispatchEvent(new MouseEvent('mouseleave'));
```

The card then keeps all of its own events — including the hover that reveals its buttons.

⚠️ **`MutationObserver` does NOT see nodes with `data-bind-for`.** The Gameface engine expands them natively,
bypassing the JS DOM API — no records are produced. The decoration has to be written off
the **model's callback**:

```js
const previous = Model._OnUpdate;          // the field behind the `updateCallback` setter
Model.updateCallback = (...a) => { previous?.(...a); scheduleRefresh(); };
```

⚠️ The model has **one** callback field — overwriting it without calling the previous one kills updates.

⚠️ The callback fires when the model is pushed; the engine expands the bindings **a few frames later**, and not
always the same number. Do several passes (for us 0/120/400 ms), each idempotent (see [#57]).

## 68. There are TWO old tooltip systems — `data-tooltip-content` is handled by the other one ❗✅

❗ **CORRECTION.** `data-tooltip-style="none"` on an ancestor silences `tooltip-manager.js`, but
**does not touch** `core/ui/tooltips/tooltip-controller.js` — and that is what draws the small bubble with just the name.
The controller looks only for content and **does not read** `data-tooltip-style` at all:

```js
recursiveGetTooltipContent(target) {
    const content = finalTarget.getAttribute('data-tooltip-content');
    if (content) { … return true; }
    return this.recursiveGetTooltipContent(target.parentElement);
}
```

The only way to switch it off is to **remove `data-tooltip-content`** from all the ancestors
of the element under the cursor.

⚠️ If the attribute is bound through `Databind.attribute`, it comes back on every model update —
the removal has to be repeated in the same cycle in which you renew the rest of the decoration.

### The previous note (still true for `tooltip-manager.js`)

`data-tooltip-content` is sometimes bound through `Databind.attribute` and is **rewritten on every
model update** — removing the attribute achieves nothing, it will come back.

`TooltipManager`, on the other hand, picks the type via **`RecursiveGetAttribute(target, 'data-tooltip-style')`**,
i.e. it walks UP from the element under the cursor, and `"none"` means "show nothing":

```js
const ttTypeName = RecursiveGetAttribute(targetElement, 'data-tooltip-style') ?? 'none';
if (ttTypeName == 'none') { this.hideTooltips(); return; }
```

So a single attribute on a panel's root silences the old tooltip across the whole subtree — without fighting
the binding.

## 69. The build queue: an arbitrary reorder exists, the game does not use it ❗✅

`model-build-queue.js` has only `moveItemUp` / `moveItemDown` (`Swap`) and `moveItemLast`. But
`moveItemLast` itself shows that the engine knows a "move to position" operation:

```js
const args = {
    InsertMode: CityOperationsParametersValues.MoveTo,
    QueueSourceLocation: from,
    QueueDestinationLocation: to,
};
if (Game.CityOperations.canStart(cityID, CityOperationTypes.BUILD, args, false).Success) {
    Game.CityOperations.sendRequest(cityID, CityOperationTypes.BUILD, args);
}
```

That is a complete primitive for drag-and-drop — you do not have to assemble a reorder out of several `Swap`s.

⚠️ Always `canStart` before `sendRequest`. The queue refuses some moves, and sending anyway
is a **silent no-op** that looks like a failed drag.

⚠️ `BuildQueue.cityID` (the model) is a ready source for the settlement's `ComponentID` — the queue panel does not
hold the city itself.

⚠️ A movement threshold when starting a drag is mandatory. Without it every click on a card counts
as a drag and the card's own `action-activate` stops working.

---

## 70. `style.cssText` on a plot-icon element removes the game's centering ❗✅

`plot-icons-root.js` → `createIcon()` sets an **inline** style on the icon:

```js
plotIcon.style.transform = `translateX(-50%) translateY(-50%)`;
```

That is the only thing centering the icon on the hex — the world anchor (`WorldAnchors.RegisterFixedWorldAnchor`)
positions the `<plot-icons>` PARENT, and the child has to pull itself back by half its own size.

⚠️ Assigning `element.style.cssText = '...'` on THAT element **replaces the whole declaration block**,
so it wipes that `transform`. The icon stops being centered and sits with its top-left corner on
the anchor — i.e. it **shifts right and down by half its width/height**.

⚠️ **The symptom looks like a zoom bug, not a positioning one.** The shift is constant in SCREEN
PIXELS (the DOM does not scale with the camera), while the hex shrinks as you zoom out — so up close it looks
like a small imperfection, and at a large zoom-out the icon lands several tiles to the side. Reported as
"the icons drift apart when zooming".

On your own elements `cssText` is fine and much cheaper than `setProperty` per
property (one crossing into the engine instead of N). The exception is the **icon's root**, because it is the only
element shared with the game:

```js
// ✅ the root: setProperty, adds to the existing declarations
for (const [name, value] of ROOT_ENTRIES) this.Root.style.setProperty(name, value);
// ✅ elements we created: cssText
child.style.cssText = CHILD_CSS;
```

The same principle applies to every element on which the game keeps an inline style — including everything
with `data-bind-style-*` (bindings write inline too).
