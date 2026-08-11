# 23 — Lokalizacja i i18n

Civ VII ma **pełnoprawny system i18n** z liczbą mnogą, rodzajem gramatycznym
i odmianą przez przypadki — nie tylko podmianę stringów. Dla polskiego to istotne,
bo polski ma tu najbardziej złożoną obsługę ze wszystkich języków w grze.

Wszystko poniżej ✅ — zweryfikowane w `Base\Assets\schema\localization\`
i w `Base\modules\base-standard\l10n\pl_PL_Text.xml` (43 722 linie, 13 562 wpisy).

## Obsługiwane języki ✅

12 języków, wszystkie z pełnym audio poza `pt_BR`:

| Locale | Język | `PluralRule` |
|---|---|---|
| `en_US` | angielski | 2 |
| `de_DE` | niemiecki | 2 |
| `it_IT` | włoski | 2 |
| `es_ES` | hiszpański | 2 |
| `fr_FR` | francuski | 3 |
| `pt_BR` | portugalski (Brazylia) | 3 |
| `ru_RU` | rosyjski | 8 |
| **`pl_PL`** | **polski** | **10** ← najbardziej złożona reguła |
| `ja_JP` | japoński | 1 |
| `ko_KR` | koreański | 1 |
| `zh_Hans_CN` | chiński uproszczony | 1 |
| `zh_Hant_HK` | chiński tradycyjny | 1 |

⚠️ **Uwaga na wielkość liter:** schemat używa `en_US`, ale foldery modów i gry
to `text/en_us/` (małe). W atrybutach `Language=` trzymaj się formy ze schematu
(`pl_PL`, `en_US`).

❗ **Nie ma ukraińskiego ani żadnego innego języka spoza tej dwunastki.**
Można dołożyć do moda pliki np. dla `uk_UA` — wiersze wstawią się bez błędu, bo
`LocalizedText` nie ma klucza obcego do `Languages` — ale **nigdy się nie wyświetlą**,
bo gry nie da się przełączyć na ten język. Tłumaczenia leżą wtedy „uśpione".
❓ Teoretycznie mod mógłby dopisać wiersz do tabeli `Languages`
(`Locale`, `Name`, `Collator`, `PluralRule`), ale nie sprawdzałem, czy sam wpis
wystarcza — dochodzi kwestia czcionek i listy języków w opcjach.

## Fallback językowy ✅

```sql
INSERT INTO "LanguagePriorities" VALUES('pl_PL','pl_PL',100);
INSERT INTO "LanguagePriorities" VALUES('pl_PL','en_US',50);
```
Każdy język ma siebie z priorytetem 100 i angielski z 50.
**Praktyczny wniosek: brakujące tłumaczenie automatycznie spada do angielskiego.**
Nie musisz tłumaczyć wszystkiego naraz — mod nie zepsuje się od braku tłumaczeń.

## Model danych ✅

```sql
CREATE TABLE LocalizedText(
    'Language' TEXT NOT NULL,
    'Tag'      TEXT NOT NULL,
    'Text'     TEXT,
    'Gender'   TEXT,
    'Plurality' TEXT,
    PRIMARY KEY (Language, Tag));
```

⚠️ **`EnglishText` to nie jest osobna tabela — to widok:**
```sql
CREATE VIEW EnglishText AS
    SELECT Tag, Text, Gender, Plurality FROM LocalizedText WHERE Language = 'en_US';
CREATE TRIGGER AddEnglishText INSTEAD OF INSERT ON EnglishText ...
```
Dlatego istnieją dwie składnie — `<EnglishText>` to skrót na `Language='en_US'`.
Obie trafiają do tej samej tabeli.

## Trzy składnie zapisu ✅

```xml
<!-- 1. angielski (przez widok) -->
<Database><EnglishText>
    <Row Tag="LOC_MOJ_TEKST"><Text>Gold</Text></Row>
</EnglishText></Database>

<!-- 2. inny język, nowy wpis -->
<Database><LocalizedText>
    <Row Tag="LOC_MOJ_TEKST" Language="pl_PL"><Text>Złoto</Text></Row>
</LocalizedText></Database>

<!-- 3. nadpisanie istniejącego wpisu (tak robi sama gra w l10n/) -->
<Database><LocalizedText>
    <Replace Tag="LOC_YIELD_GOLD_NAME" Language="pl_PL">
        <Text>Złoto|Złota|Złotu|Złoto|Złotem|Złocie|złoto|złota|złotu|złoto|złotem|złocie</Text>
        <Gender>neuter</Gender>
        <Plurality>1</Plurality>
    </Replace>
</LocalizedText></Database>
```
⚠️ `<Replace>` używaj, gdy **nadpisujesz** istniejący tag (klucz główny to
`Language`+`Tag`, więc zwykły `<Row>` by się wysypał na konflikcie).

## Konwencja folderów ✅

Gra bazowa rozdziela to inaczej niż większość modów:

```
Base/modules/base-standard/
├── text/en_us/*.xml        ← angielski (źródłowy), wiele plików tematycznych
└── l10n/
    ├── pl_PL_Text.xml      ← jeden duży plik na język
    ├── de_DE_Text.xml
    └── ...
```

Mody częściej robią `text/<locale>/InGameText.xml`. Obie konwencje działają —
liczy się to, co wpiszesz w `<UpdateText>`.

## Podpięcie w `.modinfo`

```xml
<UpdateText>
    <Item>text/en_us/InGameText.xml</Item>
    <Item locale="pl_PL">text/pl_PL/InGameText.xml</Item>
    <Item locale="de_DE">text/de_DE/InGameText.xml</Item>
</UpdateText>
```

I osobno, dla nazwy samego moda na liście modów:
```xml
<LocalizedText>
    <File>text/en_us/ModInfoText.xml</File>
    <File>text/pl_PL/ModInfoText.xml</File>
</LocalizedText>
```

## Język szablonów — składnia w tekstach ✅

### Parametry
```
{1_Amount}          parametr pozycyjny
{Amount}            parametr nazwany
{LOC_INNY_TAG}      zagnieżdżone odwołanie do innego tekstu
```

### Formatowanie liczb
```
{1_value: number +#;-#}      wymuszony znak (+5 / -5)
```

### Liczba mnoga
```
{1_value} {1_value: plural 1?tura; 2?tury; other?tur;}
```
→ „1 tura", „2 tury", „5 tur"

Realny przykład z gry:
```
Atrybut ({2_type[6]}) zwiększony o +{1_Value} {1_Value: plural 1?punkt; 2?punkty; other?punktów;}
```

### Rodzaj gramatyczny
```
{1_UnitName: gender masculine?Twój; feminine?Twoja; other?Twoje;}
```

### Zagnieżdżanie (liczba mnoga + rodzaj)
Tak gra rozwiązuje pełną polską odmianę:
```
{1_Name: plural 1?{1_Name: gender masculine?będzie stopniowo niszczony;
                              feminine?będzie stopniowo niszczona;
                              other?będzie stopniowo niszczone;};
         2?{1_Name: gender masculine?będą stopniowo niszczeni;
                    other?będą stopniowo niszczone;};}
```

### Znaczniki formatowania
```
[n]        nowa linia
[b]...[/b] pogrubienie
```

## Odmiana przez przypadki — `[n]` ✅ (najważniejsze dla polskiego)

Wartość tekstu może zawierać **warianty rozdzielone `|`**, a odwołanie
`{1_Nazwa[n]}` wybiera **n-ty wariant**.

Dla nazw yieldów konwencja to **6 przypadków × 2 wielkości liter = 12 wariantów**:

```
Złoto | Złota | Złotu | Złoto | Złotem | Złocie | złoto | złota | złotu | złoto | złotem | złocie
  1       2       3       4       5        6        7       8       9      10      11      12
```

| Indeks | Przypadek | Wielkość |
|---|---|---|
| 1 | mianownik (kto? co?) | wielka |
| 2 | dopełniacz (kogo? czego?) | wielka |
| 3 | celownik (komu? czemu?) | wielka |
| 4 | biernik (kogo? co?) | wielka |
| 5 | narzędnik (kim? czym?) | wielka |
| 6 | miejscownik (o kim? o czym?) | wielka |
| 7–12 | to samo | mała |

(Wołacz pominięty — dla nazw rzeczy niepotrzebny.)

Przykład użycia: `{2_YieldName[8]}` → „złota" (dopełniacz, mała litera).

### Ile wpisów faktycznie wymaga odmiany

Z 13 562 polskich wpisów tylko **908 (6,7%)** ma warianty:

| Wariantów | Wpisów | Zastosowanie |
|---|---|---|
| 1 (brak odmiany) | 12 388 | zwykłe teksty — **większość** |
| 12 | 550 | 6 przypadków × 2 wielkości liter |
| 6 | 249 | 6 przypadków, jedna wielkość |
| 11 | 108 | przymiotniki z tablicami `Gender`/`Plurality` |

**Wniosek praktyczny:** odmieniaj tylko te hasła, do których **inne teksty odwołują się
przez `[n]`** — czyli nazwy, które wstawiane są w zdania (yieldy, jednostki, miasta,
cywilizacje). Zwykłe opisy zostawiaj jako jeden wariant.

## Równoległe tablice `Gender` i `Plurality` ✅

Gdy wariantów jest wiele, `<Gender>` i `<Plurality>` opisują **każdy wariant z osobna**,
w tej samej kolejności:

```xml
<Replace Tag="LOC_ARMYNAME_PREFIX_1ST" Language="pl_PL">
  <Text>Pierwszy|Pierwsza|Pierwsi|Pierwsze|Pierwsz|pierwszy|...</Text>
  <Gender>masculine|feminine|masculine|feminine|neuter|masculine|...</Gender>
  <Plurality>1|1|2|2|1|1|...</Plurality>
</Replace>
```
Gdy wariant jest jeden, podaje się jedną wartość:
```xml
<Text>Złoto|Złota|...</Text>
<Gender>neuter</Gender>
<Plurality>1</Plurality>
```
⚠️ Tu `Gender`/`Plurality` opisują **całe hasło**, a `|` w `<Text>` to przypadki —
dwa różne mechanizmy w jednym wpisie. Łatwo pomylić.

## Praktyczna strategia dla moda

1. **Pisz po angielsku jako źródło** (`text/en_us/`) — fallback i tak zadziała
2. **Dodaj `pl_PL`** dla tekstów widocznych dla gracza
3. **Bez odmiany**, dopóki tekst nie jest wstawiany w inne zdania —
   opisy budynków, tradycji, nazwy modów nie potrzebują wariantów
4. **Z odmianą** tylko dla nazw wstawianych przez `{...[n]}` — jeśli twój tekst
   ma być cytowany przez inne teksty gry
5. **Testuj po polsku** — przełącz język gry i sprawdź `Logs\Localization.log`

## Debugowanie

```bash
L="/c/Users/najan/AppData/Local/Firaxis Games/Sid Meier's Civilization VII/Logs"
cat "$L/Localization.log"
```
Objaw „widzę `LOC_MOJ_TAG` zamiast tekstu" = brak wpisu dla bieżącego języka
**i** dla angielskiego (bo fallback by go złapał).

## Podglądanie tłumaczeń gry

Najlepszy słownik terminologii — polskie pliki gry:
```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"
grep -A3 'Tag="LOC_YIELD_HAPPINESS_NAME"' "$G/Base/modules/base-standard/l10n/pl_PL_Text.xml"
```
Warto z tego korzystać, żeby twój mod używał **tej samej terminologii co gra**.
