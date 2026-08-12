# 21 — Mechaniki rozgrywki (wiedza projektowa)

Ten plik to **wiedza o tym, jak Civ VII działa jako gra** — potrzebna, żeby projektować
sensowne mody, a nie tylko poprawnie wypełniać tabele.

Źródło: dokumentacja społeczności `civ7community.mintlify.app`.
⚠️ Opisy mechanik przyjmuję na wiarę; identyfikatory tabel weryfikowałem względem
schematu gry i zaznaczam rozbieżności.

## Czego w Civ VII NIE MA ❗

Najważniejsza rzecz przy projektowaniu moda — nie planuj wokół systemów z Civ VI,
których tu nie ma:

| Nie istnieje | Zastąpione przez |
|---|---|
| **Wiara (Faith)** | — (brak waluty religijnej) |
| **Turystyka** | — (brak presji kulturowej) |
| **Amenities / Zadowolenie z Civ VI** | ujednolicone **Happiness** |
| **Lojalność (Loyalty)** | system **kryzysów** przy przejściu epok |
| **Gubernatorzy** | — |
| **Punkty Wielkich Ludzi (w formie z Civ VI)** | inny system |
| **Ery** | **Epoki (Ages)** z pełnym resetem cywilizacji |

⚠️ Uwaga: schemat gry **ma** tabele `GreatPersonIndividuals`, `GreatPersonClasses`,
`Religions`, `Beliefs` ✅ — więc wielkie postaci i religia istnieją, tylko w innej
formie niż w Civ VI. Twierdzenie dokumentacji o „braku Great Person Points" traktuj
ostrożnie ❓.

## Yieldy ✅

`YIELD_FOOD`, `YIELD_PRODUCTION`, `YIELD_GOLD`, `YIELD_SCIENCE`, `YIELD_CULTURE`,
`YIELD_INFLUENCE`, `YIELD_HAPPINESS`

- **Influence** — waluta dyplomatyczna (typowo niska wartość, rzędu kilkunastu/turę)
- **Happiness** — zadowolenie jako *yield na turę*, nie pula; konsumowane przez
  specjalistów, generowane przez rozrywkę i celebracje; wpływa na wzrost i stabilność

## Epoki i ścieżki dziedzictwa

Trzy epoki: **Antiquity** (3000 p.n.e.–500 n.e.), **Exploration** (500–1700),
**Modern** (1700–współczesność).

Cztery **ścieżki dziedzictwa (Legacy Paths)** w każdej epoce:
Scientific, Militaristic, Cultural, Economic.
- kamienie milowe → Legacy Points + postęp epoki
- ukończenie całej ścieżki → **Golden Age Legacy**
- niewypełnienie → **Dark Age Legacy** w następnej epoce

Odpowiadające tabele ✅ (zweryfikowane w schemacie): `Legacies`, `LegacyPaths`,
`LegacyPathClasses`, `LegacySets`, `AgeProgressionMilestones`,
`AgeProgressionDarkAgeRewardInfos`, `GoldenAges`, `GoldenAgeModifiers`.

## Co przechodzi przez granicę epok

1. **Lider** — pozostaje ten sam (zmienia się cywilizacja!)
2. **Cuda i unikalne dzielnice** — jako obiekty **Ageless**
3. **Dowódcy (Commanders)** — zachowują poziom i doświadczenie
4. **Tradycje** — odblokowane pozostają dostępne
5. **Kontrola surowców strategicznych**
6. **Relacje dyplomatyczne** — mimo zmiany tożsamości cywilizacji

⚠️ To jest **centralna mechanika Civ VII** i główna różnica wobec poprzednich części:
gracz zmienia cywilizację między epokami, zachowując lidera i dorobek.
Stąd cała maszyneria `CivilizationSyncretismUnlocks` / `LegacyCivilizations`
opisana w [04-ages-and-civilizations.md](04-ages-and-civilizations.md).

## Kryzysy

Przy przejściach epok wyzwalają się **wydarzenia kryzysowe** — odległe osady mogą się
odrywać, pojawia się niestabilność. Karzą nadmierną ekspansję.
Tabele ✅: `AgeCrises`, `AgeCrisisEvents`, `AgeCrisisStages`.
Kolumna `Traditions.IsCrisis` ✅ — istnieją tradycje kryzysowe.

## Osady: Towns vs Cities

- **Town** — mniejsza osada, rozwijana przez „focus"
- **City** — pełne miasto
- **Urban tiles** zastąpiły wyspecjalizowane dystrykty z Civ VI:
  jedno pole miejskie mieści **dwa budynki dowolnego typu**

## Unikalne dzielnice (Unique Quarters) ✅

**Quarter** = pole z dwoma budynkami. Niektóre cywilizacje mają unikalne dzielnice
powstające, gdy konkretna **para** budynków trafi na to samo pole
(przykład: Forum Romanum = Świątynia Jowisza + Bazylika).

Tabele ✅: `UniqueQuarters`, `UniqueQuarterModifiers`.

## Przyległości i yieldy warunkowe

- bonusy przyległości wiążą się z **konkretnymi budynkami**, nie z typem dystryktu
  → tabele `Adjacency_YieldChanges`, `Constructible_Adjacencies` ✅
- budynki mogą dawać yieldy warunkowo (np. impuls produkcji po ukończeniu technologii)
- **Ageless** — unikalne ulepszenia oznaczone jako trwałe przetrwają zmianę epoki
- unikalne budynki są budowalne **tylko w swojej epoce**, ale zostają na mapie później

## Civics i tradycje

Każda cywilizacja ma **małe własne drzewko civics** (typowo 3–4 węzły w 1–2 poziomach).
Ukończenie odblokowuje **Tradycje** — polityki wnoszone do rządu w kolejnych epokach.
To jest nośnik „tożsamości kulturowej" gracza przez całą partię.

Praktyka moddingowa: patrz [04](04-ages-and-civilizations.md) — dodawanie węzła
do **istniejącego** drzewa jest mniej inwazyjne niż tworzenie własnego.

## Niezależne siły (dawne miasta-państwa)

System wysłanników **zniesiony**. Zamiast tego:
- **Befriend Independent Power** — inwestujesz Influence, startuje odliczanie
- **Suzerain** — wygrywa gracz z największą inwestycją w momencie końca odliczania
- akcje: **Levy** (najem jednostek za koszt produkcji), **Bolster**, **Incite** (podjudzenie),
  **Incorporate** (aneksja jako Town, przez projekt za Influence)
- brak pasywnych bonusów zależnych od typu miasta-państwa

Tabele ✅: `Independents`, `IndependentTribeTypes`, `CityStateTypes`,
`CityStateBonuses`, `CityStateBonusModifiers`, `DiplomacyActions`.

## Dowódcy i oblężenia

Dowódcy to osobna kategoria jednostek — mogą stacjonować w dystrykcie miasta
dla obrony, zbierają doświadczenie i przechodzą między epokami.
Miasta mają HP i siłę murów, modyfikowalne przez cechy i budynki
(`Buildings.OuterDefenseStrength`, `OuterDefenseHitPoints`, `DefenseModifier` ✅).

## Konsekwencje dla projektowania modów

1. **Cywilizacja jest związana z jedną epoką** — projektuj pod nią, a przejście
   rozwiąż przez syncretism i tradycje
2. **Nie projektuj wokół Wiary/Turystyki/Lojalności** — nie istnieją
3. **Happiness to yield**, więc bonusy do niego działają jak do produkcji czy nauki
4. **Bonusy przyległości podpinaj do budynków**, nie do dystryktów
5. **Rozważ Ageless** dla unikalnej infrastruktury, która ma przetrwać epokę
6. Unikalna dzielnica to ciekawy, tani projektowo pomysł — para budynków + modyfikator


---

## Nie ma zdarzenia „gracz zdobył zasób" ❗✅

Sprawdzone po nazwach zdarzeń w `Base/modules/`. Wszystko, co istnieje z „Resource"
w nazwie:

```
ResourceAddedToMap      zasób pojawił się NA MAPIE (nie w twoich rękach)
ResourceRemovedFromMap
ResourceAssigned        przypisanie — leci dla KAŻDEGO gracza
ResourceUnassigned
ResourceCapChanged
```

Żadne z nich nie znaczy „lokalny gracz właśnie zdobył zasób". Trzeba nasłuchiwać tanich
zdarzeń pośrednich (`ConstructibleBuildCompleted`, `TradeRouteAddedToMap`,
`TradeRouteChanged`, `ResourceCapChanged`, `LocalPlayerTurnBegin`) i za każdym razem
porównywać zbiór `player.Resources.getResources()` z poprzednim.

### ⚠️ Zdarzenie leci ZANIM zasób jest w rękach gracza

`ConstructibleBuildCompleted` mówi „ulepszenie skończone", a zasób pojawia się
w `getResources()` chwilę później. Pojedyncze sprawdzenie z debounce 400 ms **nie
łapie go**. Objaw u gracza: ulepszył pole, wszedł od razu na ekran, nic się nie stało —
bo następne zdarzenie to dopiero `LocalPlayerTurnBegin`, czyli następna tura.

Lekarstwo: po sprawdzeniu, które nic nie znalazło, dorzucić kilka opóźnionych powtórek
(600 / 1500 / 3000 ms). **Powtórka nie może zbroić kolejnych powtórek** — trzy zrobiłyby
się dziewięć, a dziewięć dwadzieścia siedem.

### ⚠️ Nie aktualizuj „co już znam" przed wykonaniem pracy

```js
known = current;              // ŹLE, jeśli praca poniżej może się nie udać
if (fresh.size === 0) return;
await placeResources(...);
```

Pass, który nic nie przypisał, i tak połknął przybycie zasobu — następne zdarzenie
zobaczy, że nic nowego nie ma, i nic nie zrobi. Jedno źle wstrzelone zdarzenie kosztuje
gracza całą funkcję do następnego zdobycia zasobu. Aktualizować **po** i tylko gdy coś
faktycznie wylądowało.
