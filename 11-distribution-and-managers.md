# 11 — Distribution, the Workshop, managing mods

## Three mod locations ✅

| Kind | Path | Notes |
|---|---|---|
| **User mods** | `C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\` | this is where you create your own; you may have to create the folder |
| **Steam Workshop** | `C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660\` | managed by Steam, **do not edit** |
| **The game and DLC** | `...\common\Sid Meier's Civilization VII\{Base,DLC}\` | read-only |

> ⚠️ **CORRECTION (2026-08-08).** This used to say
> `Documents\My Games\Sid Meier's Civilization VII\Mods\` — **wrongly**; that is the Civ VI
> convention, repeated by the community documentation as well. A mod placed there is not
> detected (`Discovered 0 mods.` in `Modding.log`). Verified empirically
> with the first mod of my own. `Documents\My Games\...` contains only `Saves\`.

`1295660` = the AppID of Civilization VII (from `steamapps\appmanifest_1295660.acf`).

## Enabling mods

The game's main menu → **Additional Content** → the list of detected mods.
This is controlled by the `Mods.Disabled` column in the modding database:
`0 = Automatic`, `1 = ExplicitEnable`, `-1 = ExplicitDisable` ✅ (from the schema).

## Properties that affect distribution ✅

```xml
<Properties>
    <ShowInBrowser>1</ShowInBrowser>       <!-- visible in the mod list -->
    <AffectsSavedGames>0</AffectsSavedGames> <!-- whether it breaks save compatibility -->
    <EnabledByDefault>1</EnabledByDefault>  <!-- used by Firaxis DLC -->
    <Package>Mod</Package>
    <URL>https://forums.civfanatics.com/resources/...</URL>
</Properties>
```

⚠️ `<Package>` — observed values: `Mod` (44 mods), `MOD` (2 — most likely a typo,
but it works, so the comparison is probably case-insensitive ❓).
The base game uses `BaseGame`, DLC use e.g. `Carlisle`.

## Dependencies vs references ✅

```xml
<Dependencies>   <!-- hard: the mod requires this to work -->
    <Mod id="base-standard" title="LOC_MODULE_BASE_STANDARD_NAME" />
</Dependencies>
<References>     <!-- soft: affects ordering/compatibility, does not require presence -->
    <Mod id="detailed-map-tacks" title="LOC_MOD_DETAILED_MAP_TACKS_NAME" />
</References>
```
`bz-map-trix` declares 5 `References` to other UI mods — a way of stating
"I know about you, position yourself relative to me", without a hard requirement.

The `ModInUse` criterion lets you enable an action only when another mod is present (6 uses):
```xml
<Criteria id="when-other-mod"><ModInUse>other-mod-id</ModInUse></Criteria>
```

## Compatibility between mods — in practice

In order of risk, from the safest:
1. `UpdateDatabase` adding **new** rows — practically conflict-free
2. `UIScripts` + `Controls.decorate` — several mods can decorate the same component
3. `UpdateDatabase` with `UPDATE`/`DELETE` on existing rows — the last one wins
4. `ReplaceUIScript` — only one mod can win
5. `ImportFiles` overriding a game file — as above, plus it breaks on patches

## The full list of the 49 installed mods ✅

Workshop folders are named by item ID, not by name — hence this table.

| Workshop ID | Mod ID |
|---|---|
| 3507072814 | bz-map-trix |
| 3507102289 | bz-city-hall |
| 3507103281 | bz-flag-corps |
| 3507297712 | detailed-map-tacks |
| 3507454171 | bz-ready-or-not |
| 3510572267 | f1rstdan-cool-ui |
| 3512790304 | izica-advanced-yield-bar |
| 3515801789 | lf-policies-yields-preview |
| 3526524592 | maple-leaves-more-lens |
| 3535775470 | nasuellia-non-sticky-selection |
| 3537808797 | leugi-diploribbon-tweaks |
| 3548476215 | EnhancedTownFocusInfoMod |
| 3556179864 | jnr-tree-sorter |
| 3570879406 | resource-fixes-deadbeef |
| 3610524341 | orions-bonus-icons-plus |
| 3616394832 | leugi-diploicon-tweaks |
| 3625755403 | bz-friends |
| 3656078784 | orions-clearer-agendas |
| 3660393011 | slothoth-better-archeology-lens |
| 3665596939 | detailed-wonder-cinematic-continued |
| 3666000265 | efs-custom-map-search |
| 3691399583 | more_hotkeys |
| 3726413243 | tile-labeling-mod |
| 3730149478 | stachs-elegant-policies-and-traditions |
| 3730601410 | civ7-screenshot-mod |
| 3734207916 | drongos-cheat-panel |
| 3734234006 | drongos-top-panel |
| 3735407674 | repair-shop-plus |
| 3735898897 | custom-civ-art-fixes |
| 3736711944 | drongos-relationship-preview |
| 3737687964 | q_mf |
| 3737760151 | shift-que |
| 3739082020 | scapehs-better-loading-screen |
| 3740174543 | holistic-qol-plus |
| 3741204633 | nasuellia-unit-flags |
| 3741296933 | bz-a-la-mods |
| 3746235254 | attribute-screen-colors |
| 3746539500 | markmoo-tech-tree-civilization-background |
| 3751330962 | ty-ends-movement-highlights |
| 3756000777 | brads-assign-all-resources |
| 3757013000 | orions-victory-meter |
| 3758712393 | leugi_happiness_stage_icons |
| 3759409080 | rewind_map_history |
| 3764225449 | stachus-elegant-great-works |
| 3768377608 | szczupakabra-poland |
| 3770924739 | tech-civic-progress |
| 3772620134 | bz-clean-slate |
| 3773536869 | leader-xp-tracker |
| 3773763645 | AutoMissionary |

Refreshing the list after changing subscriptions:
```bash
W="/c/Program Files (x86)/Steam/steamapps/workshop/content/1295660"
for d in "$W"/*/; do
  mi=$(find "$d" -maxdepth 1 -iname "*.modinfo" | head -1)
  [ -n "$mi" ] && printf "%s\t%s\n" "$(basename "$d")" \
    "$(grep -o '<Mod id="[^"]*"' "$mi" | head -1 | sed 's/<Mod id="//;s/"//')"
done
```

## Publishing your own mod — the Mod SDK ✅

**The game does NOT have a built-in uploader.** ✅ Checked in the files: `core/ui/shell/mods-content/mods-content.js`
only **displays** Workshop content (`case "SteamWorkshopContent"`), it does not publish.

Publishing is done with a **separate tool in Steam**:

- Released in the **June 2025 update (Update 1.2.2)**, together with Steam Workshop support
- It can create, debug, search and **upload mods to the Steam Workshop**
- You download it from the **"Tools" section of the Steam library** — not from the store like a normal game
- Visible only to owners of the game

⚠️ Tools **do not show up** in the Steam library list by default — you have to enable
the "Tools" filter (Library → content type filter) or search by name.

❓ I was not able to confirm the tool's exact name or AppID from the available sources
(the official 2K post announces the SDK but does not name it; the "The SDK is now available"
thread on CivFanatics is about **Civ VI**, not VII — easy to confuse when searching).
Check the Steam library under the Tools filter.

⚠️ For comparison: the **Civ VI** SDK was called "Sid Meier's Civilization VI Development
Tools" and contained ModBuddy, FireTuner, art tools and the Workshop Uploader.
Do not assume the Civ VII SDK's contents are identical — that was the first version of the tools
and Firaxis announced further development.

### Before you publish — a checklist
- [ ] `<Name>` and `<Description>` describe what the mod actually does
- [ ] `ShowInBrowser=1`, a correct `<Package>Mod</Package>`
- [ ] `AffectsSavedGames` matches reality
- [ ] `<Authors>`, `<Version>`, optionally `<URL>`
- [ ] translations (see [23-localization-i18n.md](23-localization-i18n.md)) — the English fallback works,
      so a complete set is not required
- [ ] a thumbnail/preview image (a Workshop requirement, not a mod one)
- [ ] a test on a clean installation: does the mod work without your other mods

### An alternative channel
**CivFanatics** — many mods put a `<URL>` to it in their `.modinfo` (e.g. `bz-map-trix`
links to `forums.civfanatics.com/resources/...`). It also works for Epic players,
who have no Workshop.
