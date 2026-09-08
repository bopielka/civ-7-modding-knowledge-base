# 23 — Localization and i18n

Civ VII has a **fully-fledged i18n system** with plurals, grammatical gender
and case inflection — not just string substitution. For Polish this matters,
because Polish has the most complex handling of any language in the game.

Everything below is ✅ — verified in `Base\Assets\schema\localization\`
and in `Base\modules\base-standard\l10n\pl_PL_Text.xml` (43,722 lines, 13,562 entries).

## Supported languages ✅

12 languages, all with full audio except `pt_BR`:

| Locale | Language | `PluralRule` |
|---|---|---|
| `en_US` | English | 2 |
| `de_DE` | German | 2 |
| `it_IT` | Italian | 2 |
| `es_ES` | Spanish | 2 |
| `fr_FR` | French | 3 |
| `pt_BR` | Portuguese (Brazil) | 3 |
| `ru_RU` | Russian | 8 |
| **`pl_PL`** | **Polish** | **10** ← the most complex rule |
| `ja_JP` | Japanese | 1 |
| `ko_KR` | Korean | 1 |
| `zh_Hans_CN` | Simplified Chinese | 1 |
| `zh_Hant_HK` | Traditional Chinese | 1 |

⚠️ **Watch the letter case:** the schema uses `en_US`, but the mod and game folders
are `text/en_us/` (lowercase). In `Language=` attributes stick to the schema's form
(`pl_PL`, `en_US`).

❗ **There is no Ukrainian, nor any other language outside those twelve.**
You can add files for, say, `uk_UA` to a mod — the rows will insert without an error, because
`LocalizedText` has no foreign key to `Languages` — but **they will never be displayed**,
because the game cannot be switched to that language. The translations then lie "dormant".
❓ In theory a mod could add a row to the `Languages` table
(`Locale`, `Name`, `Collator`, `PluralRule`), but I have not checked whether the entry alone
is enough — there are also fonts and the language list in the options to consider.

## Language fallback ✅

```sql
INSERT INTO "LanguagePriorities" VALUES('pl_PL','pl_PL',100);
INSERT INTO "LanguagePriorities" VALUES('pl_PL','en_US',50);
```
Every language has itself at priority 100 and English at 50.
**The practical conclusion: a missing translation automatically falls back to English.**
You do not have to translate everything at once — a mod will not break for lack of translations.

## The data model ✅

```sql
CREATE TABLE LocalizedText(
    'Language' TEXT NOT NULL,
    'Tag'      TEXT NOT NULL,
    'Text'     TEXT,
    'Gender'   TEXT,
    'Plurality' TEXT,
    PRIMARY KEY (Language, Tag));
```

⚠️ **`EnglishText` is not a separate table — it is a view:**
```sql
CREATE VIEW EnglishText AS
    SELECT Tag, Text, Gender, Plurality FROM LocalizedText WHERE Language = 'en_US';
CREATE TRIGGER AddEnglishText INSTEAD OF INSERT ON EnglishText ...
```
That is why there are two syntaxes — `<EnglishText>` is shorthand for `Language='en_US'`.
Both end up in the same table.

## Three writing syntaxes ✅

```xml
<!-- 1. English (through the view) -->
<Database><EnglishText>
    <Row Tag="LOC_MY_TEXT"><Text>Gold</Text></Row>
</EnglishText></Database>

<!-- 2. another language, a new entry -->
<Database><LocalizedText>
    <Row Tag="LOC_MY_TEXT" Language="pl_PL"><Text>Złoto</Text></Row>
</LocalizedText></Database>

<!-- 3. overriding an existing entry (this is what the game itself does in l10n/) -->
<Database><LocalizedText>
    <Replace Tag="LOC_YIELD_GOLD_NAME" Language="pl_PL">
        <Text>Złoto|Złota|Złotu|Złoto|Złotem|Złocie|złoto|złota|złotu|złoto|złotem|złocie</Text>
        <Gender>neuter</Gender>
        <Plurality>1</Plurality>
    </Replace>
</LocalizedText></Database>
```
⚠️ Use `<Replace>` when you are **overriding** an existing tag (the primary key is
`Language`+`Tag`, so a plain `<Row>` would blow up on the conflict).

## Folder convention ✅

The base game splits this differently from most mods:

```
Base/modules/base-standard/
├── text/en_us/*.xml        ← English (the source), many topic files
└── l10n/
    ├── pl_PL_Text.xml      ← one big file per language
    ├── de_DE_Text.xml
    └── ...
```

Mods more often do `text/<locale>/InGameText.xml`. Both conventions work —
what counts is what you put in `<UpdateText>`.

## Hooking it up in `.modinfo`

```xml
<UpdateText>
    <Item>text/en_us/InGameText.xml</Item>
    <Item locale="pl_PL">text/pl_PL/InGameText.xml</Item>
    <Item locale="de_DE">text/de_DE/InGameText.xml</Item>
</UpdateText>
```

And separately, for the mod's own name in the mod list:
```xml
<LocalizedText>
    <File>text/en_us/ModInfoText.xml</File>
    <File>text/pl_PL/ModInfoText.xml</File>
</LocalizedText>
```

## The template language — syntax inside texts ✅

### Parameters
```
{1_Amount}          a positional parameter
{Amount}            a named parameter
{LOC_OTHER_TAG}     a nested reference to another text
```

### Number formatting
```
{1_value: number +#;-#}      a forced sign (+5 / -5)
```

### Plurals
```
{1_value} {1_value: plural 1?tura; 2?tury; other?tur;}
```
→ "1 tura", "2 tury", "5 tur"

A real example from the game:
```
Atrybut ({2_type[6]}) zwiększony o +{1_Value} {1_Value: plural 1?punkt; 2?punkty; other?punktów;}
```

### Grammatical gender
```
{1_UnitName: gender masculine?Twój; feminine?Twoja; other?Twoje;}
```

### Nesting (plural + gender)
This is how the game resolves full Polish inflection:
```
{1_Name: plural 1?{1_Name: gender masculine?będzie stopniowo niszczony;
                              feminine?będzie stopniowo niszczona;
                              other?będzie stopniowo niszczone;};
         2?{1_Name: gender masculine?będą stopniowo niszczeni;
                    other?będą stopniowo niszczone;};}
```

### Formatting markers
```
[n]        a new line
[b]...[/b] bold
```

## Case inflection — `[n]` ✅ (the most important thing for Polish)

A text's value can contain **variants separated by `|`**, and a reference
`{1_Name[n]}` picks the **n-th variant**.

For yield names the convention is **6 cases × 2 letter cases = 12 variants**:

```
Złoto | Złota | Złotu | Złoto | Złotem | Złocie | złoto | złota | złotu | złoto | złotem | złocie
  1       2       3       4       5        6        7       8       9      10      11      12
```

| Index | Case | Capitalization |
|---|---|---|
| 1 | nominative (who? what?) | capital |
| 2 | genitive (of whom? of what?) | capital |
| 3 | dative (to whom? to what?) | capital |
| 4 | accusative (whom? what?) | capital |
| 5 | instrumental (with whom? with what?) | capital |
| 6 | locative (about whom? about what?) | capital |
| 7–12 | the same | lowercase |

(The vocative is omitted — unnecessary for the names of things.)

An example of use: `{2_YieldName[8]}` → "złota" (genitive, lowercase).

### How many entries actually need inflection

Out of 13,562 Polish entries only **908 (6.7%)** have variants:

| Variants | Entries | Use |
|---|---|---|
| 1 (no inflection) | 12,388 | ordinary texts — **the majority** |
| 12 | 550 | 6 cases × 2 letter cases |
| 6 | 249 | 6 cases, one letter case |
| 11 | 108 | adjectives with `Gender`/`Plurality` arrays |

**The practical conclusion:** inflect only those entries that **other texts refer to
via `[n]`** — that is, names inserted into sentences (yields, units, cities,
civilizations). Leave ordinary descriptions as a single variant.

## The parallel `Gender` and `Plurality` arrays ✅

When there are many variants, `<Gender>` and `<Plurality>` describe **each variant separately**,
in the same order:

```xml
<Replace Tag="LOC_ARMYNAME_PREFIX_1ST" Language="pl_PL">
  <Text>Pierwszy|Pierwsza|Pierwsi|Pierwsze|Pierwsz|pierwszy|...</Text>
  <Gender>masculine|feminine|masculine|feminine|neuter|masculine|...</Gender>
  <Plurality>1|1|2|2|1|1|...</Plurality>
</Replace>
```
When there is a single variant, a single value is given:
```xml
<Text>Złoto|Złota|...</Text>
<Gender>neuter</Gender>
<Plurality>1</Plurality>
```
⚠️ Here `Gender`/`Plurality` describe the **whole entry**, while the `|` in `<Text>` are cases —
two different mechanisms in one entry. Easy to confuse.

## A practical strategy for a mod

1. **Write in English as the source** (`text/en_us/`) — the fallback will work anyway
2. **Add `pl_PL`** for texts visible to the player
3. **No inflection** as long as the text is not inserted into other sentences —
   building and tradition descriptions and mod names do not need variants
4. **With inflection** only for names inserted via `{...[n]}` — if your text
   is going to be quoted by other texts in the game
5. **Test in Polish** — switch the game's language and check `Logs\Localization.log`

## Debugging

```bash
L="/c/Users/najan/AppData/Local/Firaxis Games/Sid Meier's Civilization VII/Logs"
cat "$L/Localization.log"
```
The symptom "I see `LOC_MY_TAG` instead of text" = there is no entry for the current language
**and** none for English (because the fallback would have caught it).

## Looking up the game's translations

The best terminology dictionary — the game's Polish files:
```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"
grep -A3 'Tag="LOC_YIELD_HAPPINESS_NAME"' "$G/Base/modules/base-standard/l10n/pl_PL_Text.xml"
```
It is worth using, so that your mod uses **the same terminology as the game**.


---

## A full set of 12 languages in a UI mod — how to do it consistently with the game ✅

### 1. First a glossary FROM THE GAME, only then a translation

Do not translate "empire resources" by ear — the game has its own word for it in every language
and the mod should use the same one, otherwise it reads as a foreign element. Extract from
`Base/modules/base-standard/l10n/<locale>_Text.xml`:

```python
re.search(r'<Replace Tag="%s" Language="[^"]*">\s*<Text>(.*?)</Text>' % key, s, re.S)
```

Keys worth extracting to begin with:

| key | what it gives |
|---|---|
| `LOC_RESOURCECLASS_EMPIRE_NAME` / `_TREASURE_` / `_FACTORY_` / `_CITY_` / `_BONUS_` | resource classes |
| `LOC_COMMERCE_EMPIRE_RESOURCE_TITLE`, `LOC_COMMERCE_TREASURE_RESOURCES_TITLE` | full section names |
| `LOC_COMMERCE_ACTIVE_/AVAILABLE_/UNAVAILABLE_TRADE_ROUTES_TITLE` | "trade routes" + variants |
| `LOC_COMMERCE_EMPIRE_RESOURCES_ORIGIN_TITLE` | "Origin" |
| `LOC_YIELD_*_NAME` | yield names |

⚠️ **`LOC_YIELD_*_NAME` is sometimes a list of inflections separated by `|`** (de, ru: `Gold|Gold|Gold|Goldes|Gold`).
Take the first member for the glossary, but in code **do not assemble those names by hand** — pass
the key and let `Locale.compose` pick the case.

⚠️ A class name is sometimes **short** (`Fabrik`, `Usine`) rather than "factory resource". If you use
`LOC_RESOURCECLASS_FACTORY_NAME` as a tab title, in some languages you will get just
"Factory" — and that is correct, because that is what the game calls it.

### 2. Generate the files with a script, not by hand

77 keys × 11 languages = 847 rows. By hand a few keys always fall out. The pattern:
a `{tag: text}` dictionary per language → a generator that takes **the key order and the complete
key set from `en_us`** and aborts when something is missing:

```python
missing = [t for t in order if t not in strings]
if missing: sys.exit(f'{locale}: missing {len(missing)}')
```

### 3. A test after generating

⚠️ **When checking "does the file use `<EnglishText>`", CUT THE COMMENTS FIRST** —
otherwise the test hits your own header comment ("never `<EnglishText>`") and reports an error
in every correct file. It cost one false-alarm round.

The test walks all the files and checks: that the XML parses, that there is no `<EnglishText>`,
that `Language=` is on **every** `<Row>`, that the `Language` attribute matches the directory name, that the key set
is complete relative to `en_us`, that there are no duplicates, and that there are `<Replace>` rows overriding base
keys.

### 4. `.modinfo` — three places, not one

```xml
<UpdateText>   <!-- in EVERY ActionGroup, including the one in scope="shell" -->
    <Item>text/en_us/InGameText.xml</Item>
    <Item locale="de_DE">text/de_DE/InGameText.xml</Item>
    ...
</UpdateText>
...
<LocalizedText>   <!-- the mod's name and description -->
    <File>text/en_us/ModInfoText.xml</File>
    <File>text/de_DE/ModInfoText.xml</File>
    ...
</LocalizedText>
```

Omitting the block in `scope="shell"` = the mod's options in the main menu are untranslated,
while in game they are translated. A symptom that is easy to miss.

### 5. Languages the game does not have

Civ VII has 12 localizations and **no Ukrainian one**. If somebody wants Ukrainian, the only
way out is to put it under another — e.g. `ru_RU`. That works, but it is a decision, not a mistake,
so it **must be documented in the file itself and in the `.modinfo`**, otherwise the next person
(or the next session) will "fix" it back to Russian.


---

## Capitalization of game terms — the convention DIFFERS in every language ❗✅

You cannot copy capitalization from English. Civ VII writes game terms in
**Title Case** in English (`Settlement`, `Happiness`, `Trade Route`, `Factory Resource`,
`Naval Units`, `Growth Rate`), but every language has its own rule. Checked by counting
occurrences in `pl_PL_Text.xml`:

| term | how the game writes it in Polish |
|---|---|
| Święto (celebration) | **Święto** — always capitalized (5/5) |
| Zadowolenie (happiness) | **Zadowolenie** — capitalized (22 occurrences) |
| Fabryka (factory) | **Fabryka/Fabryki** — capitalized |
| Miasteczko (town) | **Miasteczko** — capitalized |
| szlak handlowy (trade route) | **lowercase** — 49/49, never capitalized |
| buildings, wonders, resources, units, production | **lowercase** (a 3:1 to 20:1 majority) |

So in Polish it is **the proper names of mechanics** that get a capital letter, not common nouns —
unlike English, where almost everything that is a game term is capitalized.

### How to check a new term

```bash
grep -o "Święto[a-ząćęłńóśźż]*" pl_PL_Text.xml | sort | uniq -c | sort -rn | head
grep -o "święto[a-ząćęłńóśźż]*" pl_PL_Text.xml | wc -l
```

The form that is clearly more common wins. At a 1:1 result it is not a game term —
leave it lowercase.

⚠️ **This also catches the wrong word, not just the wrong case.** The mod had "podczas obchodów",
while the game says **Święto**. Fixing only the capitalization would have left the wrong term in place.

### The rule for a mod

German (all nouns capitalized) and CJK (no letter case) take care of themselves.
The Romance languages use sentence case in the game's files — leave them. Realistically what needs review is
English and Polish, and Ukrainian follows Polish.

## ✅ An alternative to XML: ONE `.sql` file for all languages

**Established 2026-08-26** on `f1rstdan-cool-ui` 1.9.6. `<UpdateText>` accepts a `.sql` just like
an `.xml`:

```xml
<UpdateText><Item>text/localization.sql</Item></UpdateText>
```

```sql
INSERT OR REPLACE INTO LocalizedText (Tag, Language, Text) VALUES
('LOC_MY_MOD_NAME', 'en_US', 'My Mod'),
('LOC_MY_MOD_NAME', 'pl_PL', 'Mój mod'),
('LOC_MY_MOD_NAME', 'zh_Hans_CN', '我的模组');
```

Cool UI fits **eleven languages into 151 lines of a single file** this way, instead of
eleven XML files and eleven `<Item locale="...">` entries in every action group.

⚠️ **The `Language` column's value is `en_US`, not the folder name `en_us`.** The case
matters.

✅ **`INSERT OR REPLACE` avoids the error**
`UNIQUE constraint failed: LocalizedText.ModRowId, Tag, Locale`, which on the XML route
appears when the same tag is defined twice.

⚠️ An apostrophe in the text is doubled SQL-style (`d''ensemble`), not escaped with a backslash.

**When to use which:** XML when translations come in from separate people per language (an easier pull request on
one file); SQL when the strings are few and one person maintains them.
