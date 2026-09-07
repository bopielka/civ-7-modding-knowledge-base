# 12 — Grafika, ikony, teksty i tłumaczenia

## System ikon ✅

Dwie tabele (baza ikon, schemat `Base\Assets\schema\icons\IconManager.sql`):

```sql
-- rejestracja kontekstu ikony
INSERT OR IGNORE INTO Icons(ID,Context) VALUES('CIVILIZATION_MOJA','DEFAULT');

-- właściwe mapowanie ID → plik
INSERT INTO IconDefinitions(ID,Path) VALUES
('ICON_CIVILIZATION_MOJA','fs://game/moj-mod/Art/civ_moja.png'),
('CIVILIZATION_MOJA','fs://game/moj-mod/Art/civ_moja.png');

-- warianty kontekstowe i rozmiarowe
INSERT INTO IconDefinitions(ID,Context,Path,IconSize) VALUES
('CIVILIZATION_MOJA','BACKGROUND','fs://game/moj-mod/Art/bg_1080.png',1080),
('CIVILIZATION_MOJA','BACKGROUND','fs://game/moj-mod/Art/bg_720.png',720),
('CIVILIZATION_MOJA','BACKGROUND_VERT','fs://game/moj-mod/Art/bg-card.png',0),
('UNIT_MOJ_HUSARZ','PORTRAIT_MASK','blp:unitflag_hussar',0);
```

Podpięcie w `.modinfo`:
```xml
<UpdateIcons><Item>Art/ArtDefines.sql</Item></UpdateIcons>
```

### Konteksty ikon ✅ (zaobserwowane)
`DEFAULT`, `BACKGROUND`, `BACKGROUND_VERT`, `PORTRAIT_MASK`

### Dwa schematy ścieżek ✅

| Prefiks | Znaczenie |
|---|---|
| `fs://game/<mod-id>/<ścieżka>` | plik z twojego moda (zarejestrowany przez `ImportFiles`) |
| `blp:<nazwa>` | **wbudowany zasób gry** — np. `blp:unitflag_hussar`, `blp:ntf_choosenarrative_blk` |

⚠️ `blp:` pozwala **użyć istniejącej grafiki gry bez dołączania własnej** — bardzo przydatne.
Żeby znaleźć dostępne nazwy: przeszukaj pliki gry pod kątem `blp:`.

### Gra nie ma uniwersalnej ikony „+" ✅ (2026-09-06)

Szukając plusa do własnego przycisku: w całym `Base/modules` są **tylko dwa** glify plusa —
`fs://game/rel_add_belief_plus.png` (panel wyboru wierzeń, czysty plus na przezroczystym tle,
nadaje się wszędzie) oraz `fs://game/shell_memento-plus.png` / `shell_memento-maj-plus.png`
(slot memento, plus wpisany w ramkę — poza shellem wygląda obco).

Odpowiednik po drugiej stronie: przycisk „usuń" na karcie kolejki budowy to nie kontrolka, tylko
`div` z tłem `fs://game/city_queue_trash.png` (klasa `build-queue__close-button`) zawieszony przez
`absolute -right-2 -top-2`. Własny przycisk w rogu karty robi się tak samo.

⚠️ Wpisany z klawiatury `+` obok narysowanego kosza czyta się jak inny rodzaj kontrolki — jeśli
sąsiaduje z ikoną gry, też musi być ikoną.

## `ImportFiles` — rejestracja zasobów ✅

```xml
<ImportFiles>
    <Item>Art/civ_moja.png</Item>
    <Item>icons/moja-ikona.png</Item>
    <Item>ui/moj-styl.css</Item>
</ImportFiles>
```
Dopiero po zarejestrowaniu pliku działa adres `fs://game/<mod-id>/...`.

⚠️ **Dziwactwo**: mod Polska wymienia każdy zasób **dwa razy** — raz z rozszerzeniem,
raz bez (`Art/civ_poland.png` oraz `Art/civ_poland`), i faktycznie ma na dysku oba
warianty (plik z rozszerzeniem i bez). ❓ Nie ustaliłem, czy plik bez rozszerzenia
to wymóg gry, czy artefakt narzędzia autora. Jeśli ikony nie działają — sprawdź to.

## Teksty i tłumaczenia

> 📖 **Pełny opis systemu i18n przeniesiony do osobnego pliku:
> [23-localization-i18n.md](23-localization-i18n.md)** — obsługiwane języki, fallback,
> liczba mnoga, rodzaj gramatyczny, odmiana przez przypadki (`{1_Nazwa[n]}`),
> konwencje folderów. Poniżej tylko minimum.

## Teksty — dwa formaty ✅

### Angielski (bazowy)
```xml
<?xml version="1.0" encoding="utf-8"?>
<Database>
    <EnglishText>
        <Row Tag="LOC_CIVILIZATION_MOJA_NAME">
            <Text>My Civilization</Text>
        </Row>
    </EnglishText>
</Database>
```

### Pozostałe języki
```xml
<Database>
    <LocalizedText>
        <Row Tag="LOC_CIVILIZATION_MOJA_NAME" Language="pl_PL">
            <Text>Moja cywilizacja</Text>
        </Row>
    </LocalizedText>
</Database>
```
⚠️ **Różnica jest istotna**: `<EnglishText>` bez atrybutu języka, `<LocalizedText>`
z `Language="xx_XX"` w każdym wierszu.

## Kody języków ✅

12 obsługiwanych: `en_US`, `de_DE`, `it_IT`, `es_ES`, `fr_FR`, `pt_BR`, `ru_RU`,
`pl_PL`, `ja_JP`, `ko_KR`, `zh_Hans_CN`, `zh_Hant_HK`.
Pełna tabela z regułami liczby mnogiej: [23-localization-i18n.md](23-localization-i18n.md).

## Podpięcie tekstów ✅

Dwa niezależne mechanizmy:

```xml
<!-- 1. teksty w grze -->
<UpdateText>
    <Item>text/en_us/InGameText.xml</Item>
    <Item locale="pl_PL">text/pl_PL/InGameText.xml</Item>
</UpdateText>

<!-- 2. nazwa/opis SAMEGO MODA na liście modów -->
<LocalizedText>
    <File>text/en_us/ModInfoText.xml</File>
    <File>text/pl_PL/ModInfoText.xml</File>
</LocalizedText>
```

⚠️ `<LocalizedText>` na poziomie `<Mod>` (nie w `Actions`) obsługuje `<File>`, nie `<Item>`.
To osobna tabela `LocalizedText` w bazie moddingu — dlatego `<Name>` w `<Properties>`
może być kluczem `LOC_*`.

Atrybut `locale=` na `<Item>` w `UpdateText` ✅ — tak robi `bz-map-trix` dla 8 języków.

## Konwencja kluczy `LOC_*`

```
LOC_<TYP>_<NAZWA>_<POLE>
LOC_CIVILIZATION_POLAND_NAME
LOC_CIVILIZATION_POLAND_FULLNAME
LOC_CIVILIZATION_POLAND_ADJECTIVE
LOC_CIVILIZATION_POLAND_DESCRIPTION
LOC_UNIT_POLAND_HUSSAR_NAME
LOC_TRADITION_POLAND_HETMAN_I_DESCRIPTION
LOC_MOD_<NAZWA>_NAME          ← dla samego moda
LOC_MODULE_<NAZWA>_NAME       ← konwencja modułów gry
```

W kodzie UI używaj `data-l10n-id`:
```js
element.setAttribute('data-l10n-id', 'LOC_MOJ_UI_ETYKIETA');
```
Gra podstawi tłumaczenie automatycznie — nie wpisuj tekstu na sztywno.

## Kolory gracza ✅

```xml
<UpdateColors><Item>data/playercolors.xml</Item></UpdateColors>
```
Osobna baza (`Base\Assets\schema\colors\ColorManager.sql`). Używane przez DLC liderów.

## Modele 3D ⛔

Patrz [07-cookbook-new-leader.md](07-cookbook-new-leader.md) — wymagają paczek `.dep`
z GUID-ami, brak publicznych narzędzi.
**Realna droga dla moddera: `VisualRemaps`** — nowy typ pożycza model istniejącego
obiektu (patrz [06](06-cookbook-new-civilization.md), krok 7).

## `UpdateIcons` — własne ikony pod własnym ID ✅

**Ustalone 2026-08-26** na `f1rstdan-cool-ui` 1.9.6.

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

Zarejestrowane `ID` działa **wszędzie tam, gdzie gra przyjmuje identyfikator ikony** —
`<fxs-icon data-icon-id="...">`, `UI.getIconCSS(...)`, `UI.getIconBLP(...)`.

✅ Sprytne zastosowanie z tego moda: ikona pod własnym ID pozwala **wstawić do paska dochodów
miasta wpis, który nie jest żadnym yieldem** — `yield-bar-base` bierze `type` z JSON-a i podaje
go jako `data-icon-id`, więc `DAN_CITY_POPULATION` renderuje się obok `YIELD_FOOD` bez żadnej
zmiany w komponencie. Patrz [28-city-screen.md](28-city-screen.md).

⚠️ Wskazuje się `Path` przez `fs://game/<nazwa-pliku>`, a sam plik trzeba osobno zarejestrować
w `<ImportFiles>` — i **powtórzyć oba w każdej grupie akcji**, która tych ikon używa
(patrz [14](14-quirks-and-gotchas.md) #1).

⚠️ Sprite'y na mapie (`WorldUI`) to **inna sprawa** — tam działają wyłącznie wbudowane BLP-y,
patrz [28-city-screen.md](28-city-screen.md).
