# 12 — Graphics, icons, texts and translations

## The icon system ✅

Two tables (the icon database, schema `Base\Assets\schema\icons\IconManager.sql`):

```sql
-- registering an icon context
INSERT OR IGNORE INTO Icons(ID,Context) VALUES('CIVILIZATION_MINE','DEFAULT');

-- the actual ID → file mapping
INSERT INTO IconDefinitions(ID,Path) VALUES
('ICON_CIVILIZATION_MINE','fs://game/my-mod/Art/civ_mine.png'),
('CIVILIZATION_MINE','fs://game/my-mod/Art/civ_mine.png');

-- context and size variants
INSERT INTO IconDefinitions(ID,Context,Path,IconSize) VALUES
('CIVILIZATION_MINE','BACKGROUND','fs://game/my-mod/Art/bg_1080.png',1080),
('CIVILIZATION_MINE','BACKGROUND','fs://game/my-mod/Art/bg_720.png',720),
('CIVILIZATION_MINE','BACKGROUND_VERT','fs://game/my-mod/Art/bg-card.png',0),
('UNIT_MY_HUSSAR','PORTRAIT_MASK','blp:unitflag_hussar',0);
```

Hooking it up in `.modinfo`:
```xml
<UpdateIcons><Item>Art/ArtDefines.sql</Item></UpdateIcons>
```

### Icon contexts ✅ (observed)
`DEFAULT`, `BACKGROUND`, `BACKGROUND_VERT`, `PORTRAIT_MASK`

### Two path schemes ✅

| Prefix | Meaning |
|---|---|
| `fs://game/<mod-id>/<path>` | a file from your mod (registered via `ImportFiles`) |
| `blp:<name>` | a **built-in game asset** — e.g. `blp:unitflag_hussar`, `blp:ntf_choosenarrative_blk` |

⚠️ `blp:` lets you **use the game's existing artwork without shipping your own** — very useful.
To find the available names: search the game's files for `blp:`.

### The game has no universal "+" icon ✅ (2026-09-06)

Looking for a plus sign for a button of my own: across the whole of `Base/modules` there are
**only two** plus glyphs — `fs://game/rel_add_belief_plus.png` (the belief-picking panel, a clean
plus on a transparent background, usable anywhere) and `fs://game/shell_memento-plus.png` /
`shell_memento-maj-plus.png` (the memento slot, a plus inscribed in a frame — outside the shell
it looks foreign).

The counterpart on the other side: the "remove" button on a build-queue card is not a control at
all, just a `div` with the background `fs://game/city_queue_trash.png` (class
`build-queue__close-button`) suspended by `absolute -right-2 -top-2`. A button of your own in the
corner of a card is done the same way.

⚠️ A `+` typed from the keyboard next to a drawn trash can reads as a different kind of control —
if it sits beside a game icon, it has to be an icon too.

## `ImportFiles` — registering assets ✅

```xml
<ImportFiles>
    <Item>Art/civ_mine.png</Item>
    <Item>icons/my-icon.png</Item>
    <Item>ui/my-style.css</Item>
</ImportFiles>
```
The `fs://game/<mod-id>/...` address only works once the file has been registered.

⚠️ **An oddity**: the Poland mod lists every asset **twice** — once with the extension,
once without (`Art/civ_poland.png` and `Art/civ_poland`), and it does indeed have both
variants on disk (a file with and a file without the extension). ❓ I have not established whether
the extensionless file is a requirement of the game or an artifact of the author's tooling. If icons
do not work — check this.

## Texts and translations

> 📖 **The full description of the i18n system has moved to a separate file:
> [23-localization-i18n.md](23-localization-i18n.md)** — supported languages, fallback,
> plurals, grammatical gender, case inflection (`{1_Name[n]}`),
> folder conventions. Only the minimum is below.

## Texts — two formats ✅

### English (the base)
```xml
<?xml version="1.0" encoding="utf-8"?>
<Database>
    <EnglishText>
        <Row Tag="LOC_CIVILIZATION_MINE_NAME">
            <Text>My Civilization</Text>
        </Row>
    </EnglishText>
</Database>
```

### The other languages
```xml
<Database>
    <LocalizedText>
        <Row Tag="LOC_CIVILIZATION_MINE_NAME" Language="pl_PL">
            <Text>Moja cywilizacja</Text>
        </Row>
    </LocalizedText>
</Database>
```
⚠️ **The difference matters**: `<EnglishText>` has no language attribute, `<LocalizedText>`
has `Language="xx_XX"` on every row.

## Language codes ✅

12 are supported: `en_US`, `de_DE`, `it_IT`, `es_ES`, `fr_FR`, `pt_BR`, `ru_RU`,
`pl_PL`, `ja_JP`, `ko_KR`, `zh_Hans_CN`, `zh_Hant_HK`.
The full table with plural rules: [23-localization-i18n.md](23-localization-i18n.md).

## Hooking texts up ✅

Two independent mechanisms:

```xml
<!-- 1. in-game texts -->
<UpdateText>
    <Item>text/en_us/InGameText.xml</Item>
    <Item locale="pl_PL">text/pl_PL/InGameText.xml</Item>
</UpdateText>

<!-- 2. the name/description OF THE MOD ITSELF in the mod list -->
<LocalizedText>
    <File>text/en_us/ModInfoText.xml</File>
    <File>text/pl_PL/ModInfoText.xml</File>
</LocalizedText>
```

⚠️ `<LocalizedText>` at the `<Mod>` level (not inside `Actions`) takes `<File>`, not `<Item>`.
It is a separate `LocalizedText` table in the modding database — which is why `<Name>` in
`<Properties>` can be a `LOC_*` key.

The `locale=` attribute on `<Item>` in `UpdateText` ✅ — that is what `bz-map-trix` does for 8 languages.

## The `LOC_*` key convention

```
LOC_<TYPE>_<NAME>_<FIELD>
LOC_CIVILIZATION_POLAND_NAME
LOC_CIVILIZATION_POLAND_FULLNAME
LOC_CIVILIZATION_POLAND_ADJECTIVE
LOC_CIVILIZATION_POLAND_DESCRIPTION
LOC_UNIT_POLAND_HUSSAR_NAME
LOC_TRADITION_POLAND_HETMAN_I_DESCRIPTION
LOC_MOD_<NAME>_NAME           ← for the mod itself
LOC_MODULE_<NAME>_NAME        ← the game modules' convention
```

In UI code use `data-l10n-id`:
```js
element.setAttribute('data-l10n-id', 'LOC_MY_UI_LABEL');
```
The game substitutes the translation automatically — do not hardcode the text.

## Player colors ✅

```xml
<UpdateColors><Item>data/playercolors.xml</Item></UpdateColors>
```
A separate database (`Base\Assets\schema\colors\ColorManager.sql`). Used by leader DLC.

## 3D models ⛔

See [07-cookbook-new-leader.md](07-cookbook-new-leader.md) — they require `.dep` packages
with GUIDs, and there are no public tools.
**The realistic route for a modder: `VisualRemaps`** — a new type borrows the model of an existing
object (see [06](06-cookbook-new-civilization.md), step 7).

## `UpdateIcons` — your own icons under your own ID ✅

**Established 2026-08-26** on `f1rstdan-cool-ui` 1.9.6.

```xml
<ImportFiles><Item>textures/dan_city_population.png</Item></ImportFiles>
<UpdateIcons><Item>icons/icons.xml</Item></UpdateIcons>
```

```xml
<Database>
    <IconDefinitions>
        <Row>
            <ID>DAN_CITY_POPULATION</ID>
            <Path>fs://game/dan_city_population.png</Path>
            <Context>DEFAULT</Context>
        </Row>
    </IconDefinitions>
</Database>
```

A registered `ID` works **everywhere the game accepts an icon identifier** —
`<fxs-icon data-icon-id="...">`, `UI.getIconCSS(...)`, `UI.getIconBLP(...)`.

✅ A clever use from that mod: an icon under your own ID lets you **put an entry into the city's
yield bar that is not a yield at all** — `yield-bar-base` takes `type` from the JSON and passes
it as `data-icon-id`, so `DAN_CITY_POPULATION` renders next to `YIELD_FOOD` without any
change to the component. See [28-city-screen.md](28-city-screen.md).

⚠️ `Path` is given as `fs://game/<file-name>`, and the file itself has to be registered separately
in `<ImportFiles>` — and **both have to be repeated in every action group** that uses those icons
(see [14](14-quirks-and-gotchas.md) #1).

⚠️ Sprites on the map (`WorldUI`) are **a different matter** — only built-in BLPs work there;
see [28-city-screen.md](28-city-screen.md).
