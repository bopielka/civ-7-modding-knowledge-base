# 20 — `civ7-modding-tools`: mody w TypeScript

> ⚠️ **Korekta wcześniejszego ustalenia.** W [10-tools-frameworks.md](10-tools-frameworks.md)
> napisałem początkowo, że nie istnieje SDK do Civ VII. To było **błędne** — wynikało
> z analizy wyłącznie plików lokalnych. Społecznościowe narzędzie istnieje i jest aktywne.

Źródło: dokumentacja społeczności `civ7community.mintlify.app` + repozytorium GitHub.

## Czym to jest ✅

| | |
|---|---|
| **Pakiet npm** | `civ7-modding-tools` |
| **Repozytorium** | `github.com/izica/civ7-modding-tools` |
| **Autor** | `izica` — ⚠️ ten sam, którego mod `izica-advanced-yield-bar` masz zainstalowany (Workshop `3512790304`) |
| **Powstało** | marzec 2025 |
| **Status** | aktywny rozwój, ~171 commitów na `main` |

Deklarowana motywacja autora: ręczna edycja XML przy tworzeniu własnej cywilizacji
była zbyt uciążliwa. Narzędzie **generuje pliki XML moda** z deklaratywnego kodu
TypeScript — nie zastępuje systemu modów, tylko warstwę pisania plików.

## Co daje w porównaniu z ręcznym XML

| Aspekt | Ręcznie (XML/SQL) | civ7-modding-tools |
|---|---|---|
| Błędy | wykrywane dopiero przy starcie gry | **kontrola typów w czasie kompilacji** |
| Wsparcie IDE | brak | autouzupełnianie, podpowiedzi |
| Identyfikatory | wpisywane ręcznie, literówki | stałe: `TRAIT.*`, `EFFECT.*`, `COLLECTION.*` |
| Pętla pracy | edytuj XML → uruchom → debuguj | koduj → build → wygenerowany XML |

Największa realna korzyść: **stałe zamiast stringów**. Zamiast pamiętać
`EFFECT_UNIT_ADJUST_COMBAT_STRENGTH` piszesz `EFFECT.UNIT_ADJUST_COMBAT_STRENGTH`
i literówka nie skompiluje się.

## Instalacja

```bash
# wymagane: Node.js 14+ (u Ciebie jest v20.19.5 ✅)
pnpm init
pnpm add civ7-modding-tools typescript ts-node
```
albo z repozytorium:
```bash
git clone https://github.com/izica/civ7-modding-tools
cd civ7-modding-tools
pnpm install
pnpm run build
```

## Struktura projektu

```
moj-civ7-mod/
├── src/            kod moda
├── assets/         ikony i zasoby
├── build.ts        skrypt budujący
├── package.json
└── tsconfig.json   (target ES2020, moduły CommonJS)
```

Budowanie: `pnpm ts-node build.ts` → generuje gotowego moda w katalogu wyjściowym.

## Minimalny przykład

```typescript
import { Mod } from 'civ7-modding-tools';

const mod = new Mod({ id: 'test-mod', version: '1' });
mod.build('./dist');
```

## Pełny przykład — cywilizacja z unikalną zdolnością

Za dokumentacją społeczności (przykład Dacia):

```typescript
import {
    ACTION_GROUP_BUNDLE, CivilizationBuilder, ImportFileBuilder, Mod,
    ModifierBuilder, TAG_TRAIT, TRAIT, COLLECTION, EFFECT, REQUIREMENT, UNIT_CLASS
} from "civ7-modding-tools";

const mod = new Mod({ id: 'my-civ-mod', version: '1.0' });

const civIcon = new ImportFileBuilder({
    actionGroupBundle: ACTION_GROUP_BUNDLE.AGE_ANTIQUITY,
    content: './assets/civ-icon.png',
    name: 'civ_sym_dacia'
});

const dacia = new CivilizationBuilder({
    actionGroupBundle: ACTION_GROUP_BUNDLE.AGE_ANTIQUITY,
    civilization: {
        domain: 'AntiquityAgeCivilizations',
        civilizationType: 'CIVILIZATION_DACIA'
    },
    civilizationTraits: [
        TRAIT.ANTIQUITY_CIV, TRAIT.ATTRIBUTE_MILITARISTIC, TRAIT.ATTRIBUTE_CULTURAL
    ],
    civilizationTags: [TAG_TRAIT.CULTURAL, TAG_TRAIT.MILITARY],
    icon: { path: `fs://game/${mod.id}/${civIcon.name}` },
    localizations: [{
        name: 'Dacia',
        fullName: 'Kingdom of Dacia',
        adjective: 'Dacian',
        description: '...',
        cityNames: ['Sarmizegetusa', 'Argidava', 'Buridava']
    }],
});

const uprisings = new ModifierBuilder({
    modifier: {
        collection: COLLECTION.PLAYER_UNITS,
        effect: EFFECT.UNIT_ADJUST_COMBAT_STRENGTH,
        permanent: true,
        requirements: [{
            type: REQUIREMENT.UNIT_TAG_MATCHES,
            arguments: [{ name: 'Tag', value: UNIT_CLASS.MELEE }]
        }],
        arguments: [{ name: 'Amount', value: 5 }]
    },
    localizations: [{ description: '+5 Combat Strength for Melee units.' }]
});

dacia.bind([uprisings]);       // podpięcie zdolności do cywilizacji
mod.add([dacia, civIcon]);
mod.build('./dist');
```

Zwróć uwagę, jak to mapuje się na wiedzę z [03-modifiers-effects.md](03-modifiers-effects.md):
`collection` + `effect` + `requirements` + `arguments` to **dokładnie ta sama struktura**
co `<Modifier>` w `<GameEffects>`. Narzędzie nie wymyśla nowego modelu — opakowuje istniejący.

## Architektura narzędzia

**Buildery** (dziedziczą po `BaseBuilder`): `CivilizationBuilder`, `UnitBuilder`,
`ConstructibleBuilder`, `ModifierBuilder`, `ImportFileBuilder`.
Zadania: tworzą węzły, wiążą encje (`bind`), zwracają pliki do dołączenia.

**Węzły** (dziedziczą po `BaseNode`, metoda `toXmlElement()`):
`DatabaseNode` (cały plik XML), `TypeNode`, `UnitNode`, `CivilizationNode`,
`CivilizationTraitNode`.

**Pliki**: `XmlFile` i `ImportFile` — każdy z `path`, `content`, `actionGroups`,
`actionGroupActions` (czyli narzędzie samo generuje `.modinfo`).

**Stałe**: `UNIT_CLASS`, `CONSTRUCTIBLE_TYPE_TAG`, `ACTION_GROUP`, `EFFECT`, `TRAIT`,
`COLLECTION`, `REQUIREMENT`, `TAG_TRAIT`, `ACTION_GROUP_BUNDLE`.

⚠️ `ACTION_GROUP_BUNDLE` to koncept narzędzia, nie gry — pakietuje grupy akcji
(np. „wszystko dla epoki starożytnej") zamiast pisać je ręcznie w `.modinfo`.

## Zakres wsparcia (deklarowany przez autora)

✅ gotowe: modinfo, lokalizacja, jednostki, cywilizacje, konstrukcje (constructibles),
nazwy miast, civics, tradycje, efekty gry
🚧 w toku: węzły Wielkich Ludzi
📋 planowane: węzły AI, zdolności jednostek, cuda

⚠️ To znaczy, że **narzędzie nie pokrywa wszystkiego**. Dla nieobsługiwanych elementów:
- niskopoziomowe API węzłów (`UnitNode`, `DatabaseNode`, `XmlFile`)
- własne buildery dziedziczące po klasie bazowej
- albo po prostu dopisanie surowego XML/SQL obok

## Czy używać?

**Za:** typowanie, autouzupełnianie, mniej literówek, dobre przy dużym modzie (cywilizacja).

**Przeciw:** dodatkowa warstwa abstrakcji; gdy coś nie działa, debugujesz
**wygenerowany XML**, więc i tak musisz rozumieć format z plików
[02](02-database.md)–[04](04-ages-and-civilizations.md). Nie pokrywa modów UI
(te i tak pisze się w JS — patrz [09](09-cookbook-ui-mod.md)).

**Rekomendacja:** zrób pierwszy, mały mod ręcznie, żeby zrozumieć format.
Do dużego moda z cywilizacją — rozważ to narzędzie.
Do modów UI — nieprzydatne.
