# 27 — Zasoby: co który daje, i pod jakim warunkiem

✅ **Zweryfikowane bezpośrednio w plikach gry** (`Base/modules/{base-standard,age-*}/data/`),
przez sparsowanie wszystkich `<Modifier>` z `*-gameeffects.xml` i powiązanie ich z zasobami
oboma ścieżkami opisanymi niżej. Tabele na końcu są wygenerowane z danych, nie przepisane
ręcznie.

Powstało przy pracy nad modem *Better Commerce Screen UI*, gdzie algorytm przypisywania
zasobów podejmował złe decyzje, bo pytał dane o niewłaściwą rzecz. Sam wzorzec jest jednak
własnością gry, nie moda, więc jest tu, a nie w dokumentacji moda.

---

## 1. Gdzie mieszka „co daje zasób"

Nic z tego nie jest w tabeli `Resources`. Tam jest tylko tożsamość zasobu:
`ResourceType`, `ResourceClassType` (`BONUS` / `CITY` / `EMPIRE` / `TREASURE` / `FACTORY`),
ikona i nazwa.

| Co | Gdzie |
|---|---|
| Ile zasób płaci i czym | `<Modifier>` w `<age>/data/resources-gameeffects.xml` |
| Który modyfikator dotyczy którego zasobu | `ModifierMetadatas` **albo** argument `ResourceType` |
| Warunki („tylko w mieście z portem") | `<SubjectRequirements>` wewnątrz `<Modifier>` |
| Do kogo modyfikator się stosuje | atrybut `collection` na `<Modifier>` |
| Co modyfikator robi | atrybut `effect` na `<Modifier>` |
| Yieldy pokazywane w UI | `TypeTags` (`FOOD`, `PRODUCTION`, `GOLD`, `SCIENCE`, `CULTURE`, `HAPPINESS`) |

### ⚠️ Dwie ścieżki powiązania, nie jedna

Większość modyfikatorów zasobowych jest zarejestrowana w `ModifierMetadatas`:

```xml
<Row ModifierId="MOD_FISH_PORT_FOOD" FieldName="ResourceType" String="RESOURCE_FISH"/>
```

**Ale nie wszystkie.** Część jest powiązana wyłącznie własnym argumentem `ResourceType`
modyfikatora — w epoce nowożytnej tak jest z niklem, w starożytności z jednym
z modyfikatorów gipsu. Kod, który czyta tylko `ModifierMetadatas`, uzna te zasoby za
niedające **niczego**. Trzeba czytać oba źródła i je scalić.

### ⚠️ `TypeTags` to nie to samo co realne yieldy

`TypeTags` mówi, jakie ikony yieldów UI pokaże przy zasobie. Nie mówi, ile ani czy w tej
konkretnej osadzie cokolwiek zapłaci. Jadeit ma tag `GOLD`, ale w miasteczku daje zero,
bo jego modyfikator jest bramkowany na `REQUIREMENT_CITY_HAS_BUILD_QUEUE`.

### ⚠️ `effect` i `collection` czyta się z `DynamicModifiers`, nie z `Modifiers`

W bazie w czasie gry wiersz `Modifiers` **nie ma** kolumn `EffectType` ani `CollectionType`
— trzeba przejść przez `DynamicModifiers` po `ModifierType`. W XML-u są to atrybuty na samym
`<Modifier>`, co łatwo pomylić z tym, jak wyglądają po załadowaniu. Odczytanie
`CollectionType` prosto z `Modifiers` zwraca `undefined` za każdym razem.

---

## 2. ✅ Klasa zasobu decyduje, czy w ogóle się go przypisuje — i zmienia się między epokami

`ResourceClassType` ma pięć wartości: `BONUS`, `CITY`, `FACTORY`, `EMPIRE`, `TREASURE`.

⚠️ **Zasoby imperialne i skarbowe nigdy nie trafiają do żadnej osady.** Imperialny płaci za samo
**posiadanie**, skarbowy zamienia się we floty skarbów. Ekran Handlu gry pomija oba, zanim
w ogóle zbuduje pulę nieprzypisanych — `commerce-screen-model.ts`:

```ts
if (playerResource.ResourceClassType == "RESOURCECLASS_EMPIRE" ||
    playerResource.ResourceClassType == "RESOURCECLASS_TREASURE") {
    return;
}
```

Zwróć uwagę: **żadnej logiki epokowej**. Nie jest potrzebna, bo…

### ⚠️ …to sama klasa zmienia się między epokami

**17 z 55 zasobów zmienia klasę.** Każda epoka przepisuje kolumnę wierszami `<Update>` w swoim
`<age>/data/resources.xml`. Czytanie `ResourceClassType` z załadowanej bazy daje więc od razu
odpowiedź dla granej epoki — ale **lista nazw zasobów napisana pod jedną epokę jest błędna
w dwóch pozostałych**.

| Zasób | Starożytność | Eksploracja | Nowożytność |
|---|---|---|---|
| Kakao | — | TREASURE | FACTORY |
| Bawełna | BONUS | BONUS | FACTORY |
| Len | CITY | BONUS | — |
| Futra | — | TREASURE | CITY |
| Złoto | EMPIRE | TREASURE | EMPIRE |
| Złoto (odległe ziemie) | EMPIRE | TREASURE | EMPIRE |
| Konie | EMPIRE | TREASURE | BONUS |
| Kość słoniowa | EMPIRE | BONUS | BONUS |
| Kaolin | CITY | CITY | FACTORY |
| Rubiny | BONUS | TREASURE | — |
| Srebro | EMPIRE | TREASURE | EMPIRE |
| Srebro (odległe ziemie) | EMPIRE | TREASURE | EMPIRE |
| Przyprawy | — | TREASURE | BONUS |
| Cukier | — | TREASURE | BONUS |
| Herbata | — | TREASURE | FACTORY |
| Cyna | BONUS | BONUS | FACTORY |
| Wino | EMPIRE | EMPIRE | BONUS |

Złoto jest tu najlepszym przykładem: imperialne → skarbowe → imperialne. W żadnej z tych epok
nie da się go wsadzić do miasta, ale **kość słoniowa** i **konie** przechodzą z imperialnych
na `BONUS` i od tej pory już **trzeba** je przypisywać.

⚠️ Filtr pisz jako **wykluczenie** (`EMPIRE`, `TREASURE`), nie jako białą listę — tak samo, jak
robi to gra. Klasa dodana patchem albo DLC trafi wtedy do puli, zamiast po cichu zniknąć.

---

## 3. ⚠️ Wzorzec rozgałęziony — najważniejsza rzecz w tym pliku

Gra zapisuje bonus „albo–albo" jako **dwa bramkowane modyfikatory, z których drugi jest
zaprzeczeniem pierwszego**:

```xml
<Modifier id="MOD_FISH_PORT_FOOD" ...>
    <SubjectRequirements>
        <Requirement type="REQUIREMENT_CITY_HAS_BUILDING">
            <Argument name="BuildingType">BUILDING_PORT</Argument>
        </Requirement>
    </SubjectRequirements>
    <Argument name="Amount">8</Argument>
</Modifier>

<Modifier id="MOD_FISH_NON_PORT_FOOD" ...>
    <SubjectRequirements>
        <Requirement type="REQUIREMENT_CITY_HAS_BUILDING" inverse='true'>
            <Argument name="BuildingType">BUILDING_PORT</Argument>
        </Requirement>
    </SubjectRequirements>
    <Argument name="Amount">4</Argument>
</Modifier>
```

Potwierdzone opisem w grze: *„+8 Food in Settlements with a Port, +4 Food in any other
Settlement"*. Warianty są **rozłączne, nie sumujące się** — dokładnie jeden obowiązuje.

### Dlaczego to jest pułapka

Naturalne pytanie „czy ten zasób ma warunkowy bonus, który ta osada spełnia?" zwraca
**prawdę po obu stronach rozgałęzienia**. Osada bez portu spełnia warunek `NOT PORT` —
czyli spełnia warunek wariantu *pocieszenia*. Algorytm, który z „warunek spełniony" robi
„ten zasób jest tu wyjątkowo dobry", wsadzi ryby (4 żywności) do miasteczka bez portu
przed cukrem (płaskie 8 żywności), bo cukier nie ma żadnego warunku.

### ✅ Reguła, która to rozstrzyga

> Bonus jest „warunkowy" (czytaj: ta osada jest dla niego dobrym miejscem) tylko wtedy, gdy
> jakiś bramkowany modyfikator tu działa **i** to, co osada z niego dostaje, jest
> **maksimum, jakie ten zasób może zapłacić za ten yield gdziekolwiek**.

Ryby z portem: 8 = maks. 8 → dobre miejsce. Ryby bez portu: 4 < 8 → zwykły zasób, punktowany
swoją realną kwotą. Cukier: brak bramki → nigdy nie wchodzi do tej kategorii, a przy 8
żywności i tak wygrywa z rybą za 4.

### Pełna lista rozgałęzień (wszystkie epoki)

| Epoka | Zasób | Yield | Lepszy wariant | Gorszy wariant |
|---|---|---|---|---|
| Starożytność | Gips | Produkcja | 4 — **poza** stolicą | 2 — bez warunku (czyli w stolicy) |
| Starożytność | Kaolin | Żywność | 4 — **poza** stolicą | 2 — bez warunku |
| Starożytność | Perły | Zadowolenie | 6 — **poza** stolicą | 3 — bez warunku |
| Starożytność | Cyna | Produkcja | 4 — miasteczko | 2 — miasto |
| Starożytność | Dziczyzna | Żywność | 4 — miasteczko | 2 — miasto |
| Eksploracja | Gips | Produkcja | 6 — odległe ziemie | 3 — ojczyzna |
| Eksploracja | Kaolin | Żywność | 6 — odległe ziemie | 3 — ojczyzna |
| Eksploracja | Perły | Zadowolenie | 6 — odległe ziemie | 3 — ojczyzna |
| Eksploracja | Dziczyzna | Żywność | 6 — miasteczko | 3 — miasto |
| Eksploracja | Kakao | Zadowolenie | 2 — miasteczko w ojczyźnie | 1 — miasteczko w odległych ziemiach |
| Eksploracja | Rubiny | Złoto | 2 — miasteczko w ojczyźnie | 1 — miasteczko w odległych ziemiach |
| Eksploracja | Przyprawy | Kultura / Dyplomacja | 2 — ojczyzna | 1 — odległe ziemie |
| Eksploracja | Cukier | Żywność / Zadowolenie | 2 — ojczyzna | 1 — odległe ziemie |
| Eksploracja | Herbata | Produkcja / Nauka | 2 — ojczyzna | 1 — odległe ziemie |
| Nowożytność | **Ryby** | Żywność | **8 — z portem** | **4 — bez portu** |
| Nowożytność | Futra | Zadowolenie | 8 — ze stacją kolejową | 4 — bez |
| Nowożytność | Perły | Zadowolenie | 8 — stolica (pałac) | 4 — poza stolicą |
| Nowożytność | Jedwab | Kultura | 8 — stolica (pałac) | 4 — poza stolicą |
| Nowożytność | Tytoń | Produkcja | 8 — ze stacją kolejową | 4 — bez |
| Nowożytność | Trufle | Żywność | 8 — ze stacją kolejową | 4 — bez |

⚠️ Zwróć uwagę, że **kierunek warunku zmienia się między epokami**: w starożytności perły
są lepsze *poza* stolicą, w nowożytności *w* stolicy. Każda ręcznie pisana tabela nazw
zasobów rozjedzie się z danymi przy pierwszej epoce, której autor nie sprawdził — dlatego
to się czyta z danych.

⚠️ Gips, kaolin i perły w starożytności są zapisane inaczej niż reszta: wariant „gorszy" nie
ma **żadnego** warunku, więc poza stolicą oba modyfikatory są aktywne naraz. Opis w grze
(„+2 do stolicy, +4 w każdym innym mieście") mówi, że mają być **rozłączne**, więc przy
sumowaniu trzeba brać **maksimum z grupy**, a nie sumę — inaczej poza stolicą wyjdzie 6
zamiast 4.

---

## 4. Warianty jednostronne — bramka bez alternatywy

Osobna kategoria: modyfikator ma warunek, ale nie ma wariantu zapasowego. Poza warunkiem
zasób daje po prostu **zero**.

| Epoka | Zasób | Yield | Warunek |
|---|---|---|---|
| wszystkie | Kauri | Złoto **albo** Nauka | miasto → złoto, miasteczko → nauka (rozłącznie) |
| Star. / Eksp. | Jedwab | Kultura % | tylko miasto (kolejka budowy) |
| Star. / Eksp. | Jadeit | Złoto % | tylko miasto |
| Starożytność | Lapis lazuli | Produkcja + Złoto % | tylko miasto |
| Starożytność | Kadzidło | Nauka % | tylko miasto |
| Eksploracja | Goździki | Złoto % | tylko miasto |
| Nowożytność | Nikiel | Nauka % + Złoto % | tylko miasto |
| Star. / Eksp. | Wino | Kultura | tylko podczas Święta (Golden Age) |
| Eksploracja | Futra | Złoto | tylko podczas Święta |

⚠️ `REQUIREMENT_CITY_HAS_BUILD_QUEUE` to najczęstszy zapis „tylko miasta" — **miasteczko nie
ma kolejki budowy**. 29 modyfikatorów zasobowych jest tak bramkowanych. Kod, który to
zignoruje, przypisze miasteczkom +10 złota z jadeitu, +10 kultury z jedwabiu i +4 produkcji
z lapis lazuli, z których żadne tam nie powstanie.

⚠️ `REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE` to jedyny warunek dotyczący **gracza**, nie osady.
Nie da się go spełnić wyborem osady, więc nie powinien wpływać na to, gdzie zasób trafi —
a przy podliczaniu dochodu imperium trzeba go liczyć osobno, bo poza Świętem nie płaci nic.

---

## 5. Rodzaje efektów, które zasoby w ogóle mają

Nie każdy modyfikator zasobowy daje yield. Pełen zestaw kształtów spotykanych w danych:

| Kształt efektu | Co to znaczy | Przykłady |
|---|---|---|
| `CITY_ADJUST_YIELD_PER_RESOURCE` | płaski yield na osadę | Ryby, Perły, Kość słoniowa |
| `CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE` | to samo, inne liczenie | Złoto, Srebro, Wino, Futra |
| `PLAYER_ADJUST_YIELD_PER_RESOURCE_TYPE` | yield dla gracza, raz | — |
| `UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE` | siła bojowa, **limit +6** | Saletra, Węgiel, Ropa, Kauczuk |
| `CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE` | % ku budowie czegoś | Węgiel, Ropa, Marmur |
| `*_ADJUST_UNIT_PRODUCTION_*` | tańsze jednostki | Trufle, Sól, Bawełna, Kadzidło |
| `CITY_ADJUST_CONSTRUCTIBLE_YIELD_PER_RESOURCE` z `Tag=WAREHOUSE` | skaluje się liczbą magazynów | Żółwie, Glina, Kraby |
| `ADJUST_PLAYER_YIELD_PER_SLOTTED_RESOURCE` | % yieldu, zasoby fabryczne | Kakao, Herbata, Kaolin |
| `CITY_ADJUST_GROWTH_PER_RESOURCE` | % tempa wzrostu | Cyna |
| `UNIT_ADJUST_HEAL_PER_RESOURCE` | leczenie jednostek | Chinina |
| `PLOT_PLACE_RESOURCE` | nie efekt gracza — rozstawianie na mapie | Gips, Konie |

⚠️ **Limit siły bojowej +6 nie istnieje w danych.** Każdy z tych zasobów ma „(maximum +6)"
w swoim opisie, ale żaden argument, parametr globalny ani tabela go nie niesie — trzyma go
silnik. Trzeba go zapisać jako stałą i wiedzieć, że patch może ją unieważnić.

⚠️ **Dwa efekty fabryczne trzymają liczbę w argumencie `Percent`, nie `Amount`**, a jeden
nazywa `ConstructibleClass` zamiast `ConstructibleType`. Kod pisany pod kształty zasobów
imperialnych odczyta je wszystkie jako **zero**.

⚠️ **Wszystkie cztery przyrostki skalują się liczbą kopii** — `PER_RESOURCE`,
`PER_AVAILABLE_RESOURCE_TYPE`, `PER_RESOURCE_TYPE`, `PER_SLOTTED_RESOURCE`. Nazwa sugeruje,
że te z `_TYPE` płacą raz za całe imperium; **pomiar w grze mówi inaczej**. To, co się
faktycznie różni, to **zasięg**, a on wynika z `collection`.

---

## 6. Kolekcje używane przez zasoby

| Kolekcja | Ile modyfikatorów | Zasięg |
|---|---|---|
| `COLLECTION_ALL_CITIES` | 115 | raz na każdą osadę, którą przepuszczą wymagania |
| `COLLECTION_ALL_UNITS` | 10 | armia |
| `COLLECTION_ALL_PLAYERS` | 10 | raz, na gracza |
| `COLLECTION_ALL_CAPITAL_CITIES` | 6 | tylko stolica |

⚠️ Czytanie samych wymagań **nie wystarcza**. Futra dają +3 Zadowolenia przez
`COLLECTION_ALL_CAPITAL_CITIES` — raz, w stolicy — a policzenie tego w każdej osadzie mnoży
wynik przez wielkość imperium.

---

## 7. Uwagi metodologiczne

- **Nazwa w danych to hipoteza, pomiar w działającej grze to fakt.** Kilka błędów w tej
  dziedzinie wzięło się z czytania nazwy efektu jak specyfikacji.
- **Czego nie rozumiesz, uznaj za spełnione.** Przy ocenie wymagań zbyt gorliwe podejście
  kosztuje trochę niedokładny wynik; zbyt surowe **wycina zasób z rozważań w ogóle**, co jest
  znacznie gorsze i znacznie trudniejsze do zauważenia.
- ❓ **Tabele poniżej pokrywają wyłącznie `Base/modules`.** DLC (`DLC/*/modules`) mogą dodawać
  i nadpisywać zasoby — nie zostało to sprawdzone.
- Zasoby fabryczne: **osada prowadzi naraz tylko jeden rodzaj, ale dowolnie wiele kopii**
  (`LOC_PEDIA_CONCEPTS_FACTORY_RESOURCES_TOOLTIP`).

---

## 8. Pełne tabele efektów, per epoka

Wygenerowane z plików gry. „Kwota" z `%` to wartość procentowa. Wiersze bez nazwy zasobu
należą do zasobu z wiersza powyżej.

### age-antiquity

| Zasób | Klasa | Efekt | Kwota | Warunek |
|---|---|---|---|---|
| CLAY | BONUS | Production | 1 | — |
| COTTON | BONUS | Food | 2 | — |
|  |  | Production | 2 | — |
| COWRIE | BONUS | Gold | 4 | city |
|  |  | Science | 2 | town |
| CRABS | BONUS | Food | 1 | — |
| DATES | BONUS | Food | 2 | — |
|  |  | Happiness | 2 | — |
| DYES | BONUS | Happiness | 4 | — |
| FISH | BONUS | Food | 3 | — |
| FLAX | CITY | Culture | 2 | — |
|  |  | Science | 2 | — |
| GOLD | EMPIRE | Gold | 1 | — |
|  |  | Happiness | 1 | — |
| GOLD_DISTANT_LANDS | EMPIRE | PLAYER_ADJUST_PURCHASE_EFFICIENCY_PER_RESOURCE | 20% | — |
| GYPSUM | CITY | PLOT_PLACE_RESOURCE | — | — |
|  |  | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE | 10 | city (build queue) |
|  |  | Production | 2 | — |
|  |  | Production | 4 | not capital |
| HARDWOOD | EMPIRE | PLAYER_ADJUST_UNIT_PRODUCTION_PER_RESOURCE | 10% | — |
| HIDES | BONUS | Production | 3 | — |
| HORSES | EMPIRE | PLOT_PLACE_RESOURCE | — | — |
|  |  | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
| INCENSE | CITY | Science | 10% | city (build queue) |
| IRON | EMPIRE | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
| IVORY | EMPIRE | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
| JADE | CITY | Gold | 10% | city (build queue) |
| KAOLIN | CITY | Food | 2 | — |
|  |  | Food | 4 | not capital |
| LAPIS_LAZULI | CITY | CITY_ADD_RESOURCE_TO_PLOT | — | PLAYER_ELIGIBLE_CS_BONUS |
|  |  | Production | 4 | city (build queue) |
|  |  | Gold | 10% | city (build queue) |
| LIMESTONE | EMPIRE | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 5 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 5 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 5 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 5 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 5 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 5 | — |
| LLAMAS | BONUS | Happiness | 3 | — |
|  |  | Production | 1 | — |
| MANGOS | CITY | Culture | 2 | — |
|  |  | Food | 2 | — |
| MARBLE | EMPIRE | PLOT_PLACE_RESOURCE | — | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
| PEARLS | CITY | PLOT_PLACE_RESOURCE | — | — |
|  |  | Happiness | 3 | — |
|  |  | Happiness | 6 | not capital |
| RICE | EMPIRE | Food | 2 | — |
| RUBIES | BONUS | Gold | 4 | — |
| SALT | CITY | CITY_ADJUST_UNIT_PRODUCTION_PER_RESOURCE | 20% | city (build queue) |
| SILK | CITY | Culture | 10% | city (build queue) |
| SILVER | EMPIRE | PLOT_PLACE_RESOURCE | — | — |
|  |  | Food | 1 | — |
|  |  | Gold | 1 | — |
| SILVER_DISTANT_LANDS | EMPIRE | PLAYER_ADJUST_PURCHASE_EFFICIENCY_PER_RESOURCE | 20% | — |
| TIN | BONUS | Production | 2 | city |
|  |  | Production | 4 | town |
| TURTLES | BONUS | Culture | 1 | — |
| WILD_GAME | BONUS | Food | 2 | city |
|  |  | Food | 4 | town |
| WINE | EMPIRE | PLOT_PLACE_RESOURCE | — | — |
|  |  | Happiness | 2 | — |
|  |  | Culture | 5 | Celebration |
| WOOL | BONUS | Happiness | 2 | — |
|  |  | Production | 2 | — |

### age-exploration

| Zasób | Klasa | Efekt | Kwota | Warunek |
|---|---|---|---|---|
| CLAY | BONUS | Production | 1 | — |
| CLOVES | CITY | CITY_ADD_RESOURCE_TO_PLOT | — | PLAYER_ELIGIBLE_CS_BONUS |
|  |  | Food | 6 | — |
|  |  | Gold | 10% | city (build queue) |
| COCOA | TREASURE | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
|  |  | Happiness | 1 | town + distant lands |
|  |  | Happiness | 2 | town + not distant lands |
| COTTON | BONUS | Food | 3 | — |
|  |  | Production | 3 | — |
| COWRIE | BONUS | Gold | 5 | city |
|  |  | Science | 3 | town |
| CRABS | BONUS | Food | 2 | — |
| DATES | BONUS | Food | 3 | — |
|  |  | Happiness | 3 | — |
| DYES | BONUS | Happiness | 5 | — |
| FISH | BONUS | PLOT_PLACE_RESOURCE | — | — |
|  |  | Food | 5 | — |
| FLAX | BONUS | Culture | 2 | — |
|  |  | Science | 2 | — |
| FURS | TREASURE | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
|  |  | Happiness | 3 | — |
|  |  | Gold | 10 | Celebration |
| GOLD | TREASURE | PLOT_PLACE_RESOURCE | — | — |
|  |  | Gold | 1 | — |
|  |  | Happiness | 1 | — |
|  |  | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
|  |  | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
| GYPSUM | CITY | Production | 6 | distant lands |
|  |  | Production | 3 | not distant lands |
| HARDWOOD | EMPIRE | PLAYER_ADJUST_UNIT_PRODUCTION_PER_RESOURCE | 20% | — |
| HORSES | TREASURE | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
| INCENSE | CITY | CITY_ADJUST_UNIT_PRODUCTION_PER_RESOURCE | 100% | city (build queue) |
|  |  | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE | 100 | city (build queue) |
| IRON | EMPIRE | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
| IVORY | BONUS | Happiness | 3 | — |
|  |  | Production | 3 | — |
| JADE | CITY | PLOT_PLACE_RESOURCE | — | — |
|  |  | Gold | 15% | city (build queue) |
| KAOLIN | CITY | Food | 6 | distant lands |
|  |  | Food | 3 | not distant lands |
| LIMESTONE | EMPIRE | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE | 10 | city (build queue) |
| LLAMAS | BONUS | Happiness | 5 | — |
|  |  | Production | 1 | — |
| MANGOS | CITY | Culture | 3 | — |
|  |  | Food | 3 | — |
| MARBLE | EMPIRE | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
| NITER | EMPIRE | PLOT_PLACE_RESOURCE | — | — |
|  |  | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
| PEARLS | CITY | Happiness | 6 | distant lands |
|  |  | Happiness | 3 | not distant lands |
| PITCH | BONUS | Gold | 1 | — |
| RICE | EMPIRE | Food | 2 | — |
| RUBIES | TREASURE | Gold | 1 | town + distant lands |
|  |  | Gold | 2 | town + not distant lands |
| SILK | CITY | Culture | 10% | city (build queue) |
| SILVER | TREASURE | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
|  |  | Food | 1 | — |
|  |  | Gold | 1 | — |
|  |  | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
| SPICES | TREASURE | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
|  |  | Culture | 1 | city (build queue) + distant lands |
|  |  | Diplomacy | 1 | city (build queue) + distant lands |
|  |  | Culture | 2 | city (build queue) + not distant lands |
|  |  | Diplomacy | 2 | city (build queue) + not distant lands |
| SUGAR | TREASURE | PLOT_PLACE_RESOURCE | — | — |
|  |  | Food | 1 | city (build queue) + distant lands |
|  |  | Happiness | 1 | city (build queue) + distant lands |
|  |  | Food | 2 | city (build queue) + not distant lands |
|  |  | Happiness | 2 | city (build queue) + not distant lands |
| TEA | TREASURE | PLAYER_ADJUST_RESOURCE_COUNT_PER_INSTANCE | 1 | — |
|  |  | PLOT_PLACE_RESOURCE | — | — |
|  |  | Production | 1 | city (build queue) + distant lands |
|  |  | Science | 1 | city (build queue) + distant lands |
|  |  | Production | 2 | city (build queue) + not distant lands |
|  |  | Science | 2 | city (build queue) + not distant lands |
| TIN | BONUS | Gold | 3 | — |
|  |  | Production | 3 | — |
| TRUFFLES | CITY | CITY_ADJUST_UNIT_PRODUCTION_PER_RESOURCE | 20% | city (build queue) |
| TURTLES | BONUS | Culture | 2 | — |
| WHALES | BONUS | Production | 5 | — |
| WILD_GAME | BONUS | Food | 3 | city |
|  |  | Food | 6 | town |
| WINE | EMPIRE | Happiness | 3 | — |
|  |  | Culture | 10 | Celebration |

### age-modern

| Zasób | Klasa | Efekt | Kwota | Warunek |
|---|---|---|---|---|
| CITRUS | FACTORY | CITY_ADJUST_UNIT_PRODUCTION_PER_SLOTTED_RESOURCE | 5% | — |
| COAL | EMPIRE | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
|  |  | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE | 10 | city (build queue) |
|  |  | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE | 10 | city (build queue) |
| COCOA | FACTORY | Happiness | 3 | — |
| COFFEE | FACTORY | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_SLOTTED_RESOURCE | 5 | — |
|  |  | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_SLOTTED_RESOURCE | 5 | — |
| COTTON | FACTORY | CITY_ADJUST_UNIT_PRODUCTION_PER_SLOTTED_RESOURCE | 5% | — |
| COWRIE | BONUS | Gold | 6 | city |
|  |  | Science | 4 | town |
| CRABS | BONUS | Food | 3 | — |
| FISH | BONUS | Food | 4 | not Port |
|  |  | Food | 8 | Port |
| FURS | CITY | Happiness | 4 | not Rail Station |
|  |  | Happiness | 8 | Rail Station |
| GOLD | EMPIRE | Gold | 1 | — |
|  |  | Happiness | 1 | — |
| GOLD_DISTANT_LANDS | EMPIRE | PLAYER_ADJUST_PURCHASE_EFFICIENCY_PER_RESOURCE | 20% | — |
| HARDWOOD | EMPIRE | PLAYER_ADJUST_UNIT_PRODUCTION_PER_RESOURCE | 20% | — |
| HORSES | BONUS | Happiness | 8 | — |
| IVORY | BONUS | Happiness | 4 | — |
|  |  | Production | 4 | — |
| KAOLIN | FACTORY | Culture | 3% | — |
| LIMESTONE | EMPIRE | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE | 10 | city (build queue) |
| LLAMAS | BONUS | Happiness | 7 | — |
|  |  | Production | 1 | — |
| MARBLE | EMPIRE | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
|  |  | CITY_ADJUST_BIOME_WONDER_PRODUCTION_PER_RESOURCE | 10 | — |
| NICKEL | CITY | CITY_ADD_RESOURCE_TO_PLOT | — | PLAYER_ELIGIBLE_CS_BONUS |
|  |  | Gold | 10% | city (build queue) |
|  |  | Science | 10% | city (build queue) |
| NITER | EMPIRE | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
| OIL | EMPIRE | CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE | 10 | city (build queue) |
|  |  | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
| PEARLS | CITY | Happiness | 8 | Palace |
|  |  | Happiness | 4 | not Palace |
| PITCH | BONUS | Production | 1 | — |
|  |  | ADJUST_BUILDING_PRODUCTION_EFFICIENCY_PER_RESOURCE_TYPE | 10% | — |
| QUININE | FACTORY | UNIT_ADJUST_HEAL_PER_RESOURCE | 1 | — |
| RICE | EMPIRE | Food | 2 | — |
| RUBBER | EMPIRE | UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE | 1 | UNIT_TAG_MATCHES |
| SILK | CITY | Culture | 8 | Palace |
|  |  | Culture | 4 | not Palace |
| SILVER | EMPIRE | Food | 1 | — |
|  |  | Gold | 1 | — |
| SILVER_DISTANT_LANDS | EMPIRE | PLAYER_ADJUST_PURCHASE_EFFICIENCY_PER_RESOURCE | 20% | — |
| SPICES | BONUS | Food | 4 | — |
|  |  | Happiness | 4 | — |
| SUGAR | BONUS | Food | 8 | — |
| TEA | FACTORY | Science | 3% | — |
| TIN | FACTORY | CITY_ADJUST_GROWTH_PER_RESOURCE | 3% | — |
| TOBACCO | CITY | Production | 4 | not Rail Station |
|  |  | Production | 8 | Rail Station |
| TRUFFLES | CITY | Food | 4 | not Rail Station |
|  |  | Food | 8 | Rail Station |
| WHALES | BONUS | Production | 8 | — |
| WINE | BONUS | Food | 4 | — |
|  |  | Production | 4 | — |
