# 27 — Resources: what each one gives, and under what condition

✅ **Verified directly in the game's files** (`Base/modules/{base-standard,age-*}/data/`),
by parsing every `<Modifier>` from `*-gameeffects.xml` and linking them to resources
through both of the routes described below. The tables at the end are generated from the data, not written out
by hand.

It came out of work on the *Better Commerce Screen UI* mod, where the resource-assignment
algorithm made bad decisions because it asked the data about the wrong thing. The pattern itself is however
a property of the game, not of the mod, so it lives here rather than in the mod's documentation.

---

## 1. Where "what a resource gives" lives

None of it is in the `Resources` table. That holds only the resource's identity:
`ResourceType`, `ResourceClassType` (`BONUS` / `CITY` / `EMPIRE` / `TREASURE` / `FACTORY`),
the icon and the name.

| What | Where |
|---|---|
| How much a resource pays and in what | a `<Modifier>` in `<age>/data/resources-gameeffects.xml` |
| Which modifier belongs to which resource | `ModifierMetadatas` **or** a `ResourceType` argument |
| Conditions ("only in a city with a Port") | `<SubjectRequirements>` inside the `<Modifier>` |
| Who the modifier applies to | the `collection` attribute on the `<Modifier>` |
| What the modifier does | the `effect` attribute on the `<Modifier>` |
| The yields shown in the UI | `TypeTags` (`FOOD`, `PRODUCTION`, `GOLD`, `SCIENCE`, `CULTURE`, `HAPPINESS`) |

### ⚠️ Two linking routes, not one

Most resource modifiers are registered in `ModifierMetadatas`:

```xml
<Row ModifierId="MOD_FISH_PORT_FOOD" FieldName="ResourceType" String="RESOURCE_FISH"/>
```

**But not all of them.** Some are linked only through the modifier's own `ResourceType`
argument — in the Modern age that is the case for nickel, in Antiquity for one
of the gypsum modifiers. Code that reads `ModifierMetadatas` only will conclude that those resources give
**nothing**. You have to read both sources and merge them.

### ⚠️ `TypeTags` is not the same as actual yields

`TypeTags` says which yield icons the UI will show next to a resource. It does not say how much, or whether in
that particular settlement it will pay anything at all. Jade has the `GOLD` tag, but in a town it gives zero,
because its modifier is gated on `REQUIREMENT_CITY_HAS_BUILD_QUEUE`.

### ⚠️ `effect` and `collection` are read from `DynamicModifiers`, not from `Modifiers`

In the runtime database a `Modifiers` row **has no** `EffectType` or `CollectionType` columns
— you have to go through `DynamicModifiers` by `ModifierType`. In XML they are attributes on the
`<Modifier>` itself, which is easy to confuse with how they look after loading. Reading
`CollectionType` straight from `Modifiers` returns `undefined` every time.

---

## 2. ✅ A resource's class decides whether it is assigned at all — and it changes between ages

`ResourceClassType` has five values: `BONUS`, `CITY`, `FACTORY`, `EMPIRE`, `TREASURE`.

⚠️ **Empire and treasure resources never go into any settlement.** An empire one pays for merely
**being owned**, a treasure one turns into treasure fleets. The game's Commerce screen skips both before
it even builds the unassigned pool — `commerce-screen-model.ts`:

```ts
if (playerResource.ResourceClassType == "RESOURCECLASS_EMPIRE" ||
    playerResource.ResourceClassType == "RESOURCECLASS_TREASURE") {
    return;
}
```

Note: **no age logic whatsoever**. It is not needed, because…

### ⚠️ …it is the class itself that changes between ages

**17 of the 55 resources change class.** Each age rewrites the column with `<Update>` rows in its own
`<age>/data/resources.xml`. So reading `ResourceClassType` from the loaded database gives you the
answer for the age being played straight away — but **a list of resource names written for one age is wrong
in the other two**.

| Resource | Antiquity | Exploration | Modern |
|---|---|---|---|
| Cocoa | — | TREASURE | FACTORY |
| Cotton | BONUS | BONUS | FACTORY |
| Flax | CITY | BONUS | — |
| Furs | — | TREASURE | CITY |
| Gold | EMPIRE | TREASURE | EMPIRE |
| Gold (distant lands) | EMPIRE | TREASURE | EMPIRE |
| Horses | EMPIRE | TREASURE | BONUS |
| Ivory | EMPIRE | BONUS | BONUS |
| Kaolin | CITY | CITY | FACTORY |
| Rubies | BONUS | TREASURE | — |
| Silver | EMPIRE | TREASURE | EMPIRE |
| Silver (distant lands) | EMPIRE | TREASURE | EMPIRE |
| Spices | — | TREASURE | BONUS |
| Sugar | — | TREASURE | BONUS |
| Tea | — | TREASURE | FACTORY |
| Tin | BONUS | BONUS | FACTORY |
| Wine | EMPIRE | EMPIRE | BONUS |

Gold is the best example here: empire → treasure → empire. In none of those ages
can it be put into a city, but **ivory** and **horses** move from empire
to `BONUS` and from then on they **have** to be assigned.

⚠️ Write the filter as an **exclusion** (`EMPIRE`, `TREASURE`), not as an allowlist — exactly as
the game does. A class added by a patch or a DLC will then land in the pool instead of quietly disappearing.

---

## 3. ⚠️ The branched pattern — the most important thing in this file

The game encodes an "either–or" bonus as **two gated modifiers, the second of which is the
negation of the first**:

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

Confirmed by the in-game description: *"+8 Food in Settlements with a Port, +4 Food in any other
Settlement"*. The variants are **mutually exclusive, not additive** — exactly one applies.

### Why this is a trap

The natural question "does this resource have a conditional bonus that this settlement satisfies?" returns
**true on both sides of the branch**. A settlement without a port satisfies the `NOT PORT` condition —
i.e. it satisfies the condition of the *consolation* variant. An algorithm that turns "the condition is satisfied" into
"this resource is exceptionally good here" will put fish (4 food) into a town without a port
ahead of sugar (a flat 8 food), because sugar has no condition at all.

### ✅ The rule that settles it

> A bonus is "conditional" (read: this settlement is a good place for it) only when
> some gated modifier applies here **and** what the settlement gets from it is
> **the maximum that resource can pay for that yield anywhere**.

Fish with a port: 8 = max 8 → a good place. Fish without a port: 4 < 8 → an ordinary resource, scored
at its actual amount. Sugar: no gate → it never enters that category, and with 8
food it beats fish at 4 anyway.

### The complete list of branches (all ages)

| Age | Resource | Yield | Better variant | Worse variant |
|---|---|---|---|---|
| Antiquity | Gypsum | Production | 4 — **outside** the capital | 2 — no condition (i.e. in the capital) |
| Antiquity | Kaolin | Food | 4 — **outside** the capital | 2 — no condition |
| Antiquity | Pearls | Happiness | 6 — **outside** the capital | 3 — no condition |
| Antiquity | Tin | Production | 4 — a town | 2 — a city |
| Antiquity | Wild game | Food | 4 — a town | 2 — a city |
| Exploration | Gypsum | Production | 6 — distant lands | 3 — the homeland |
| Exploration | Kaolin | Food | 6 — distant lands | 3 — the homeland |
| Exploration | Pearls | Happiness | 6 — distant lands | 3 — the homeland |
| Exploration | Wild game | Food | 6 — a town | 3 — a city |
| Exploration | Cocoa | Happiness | 2 — a town in the homeland | 1 — a town in distant lands |
| Exploration | Rubies | Gold | 2 — a town in the homeland | 1 — a town in distant lands |
| Exploration | Spices | Culture / Diplomacy | 2 — the homeland | 1 — distant lands |
| Exploration | Sugar | Food / Happiness | 2 — the homeland | 1 — distant lands |
| Exploration | Tea | Production / Science | 2 — the homeland | 1 — distant lands |
| Modern | **Fish** | Food | **8 — with a Port** | **4 — without a Port** |
| Modern | Furs | Happiness | 8 — with a Rail Station | 4 — without |
| Modern | Pearls | Happiness | 8 — the capital (Palace) | 4 — outside the capital |
| Modern | Silk | Culture | 8 — the capital (Palace) | 4 — outside the capital |
| Modern | Tobacco | Production | 8 — with a Rail Station | 4 — without |
| Modern | Truffles | Food | 8 — with a Rail Station | 4 — without |

⚠️ Note that **the direction of the condition changes between ages**: in Antiquity pearls
are better *outside* the capital, in the Modern age *in* it. Any hand-written table of resource
names will drift away from the data at the first age its author did not check — which is why
this is read from the data.

⚠️ Gypsum, kaolin and pearls in Antiquity are encoded differently from the rest: the "worse" variant has
**no** condition at all, so outside the capital both modifiers are active at once. The in-game description
("+2 in the capital, +4 in any other city") says they are meant to be **mutually exclusive**, so when
summing you have to take **the maximum of the group**, not the sum — otherwise outside the capital you get 6
instead of 4.

---

## 4. One-sided variants — a gate with no alternative

A separate category: a modifier has a condition, but no fallback variant. Outside the condition
the resource simply gives **zero**.

| Age | Resource | Yield | Condition |
|---|---|---|---|
| all | Cowrie | Gold **or** Science | city → gold, town → science (mutually exclusive) |
| Ant. / Expl. | Silk | Culture % | cities only (build queue) |
| Ant. / Expl. | Jade | Gold % | cities only |
| Antiquity | Lapis lazuli | Production + Gold % | cities only |
| Antiquity | Incense | Science % | cities only |
| Exploration | Cloves | Gold % | cities only |
| Modern | Nickel | Science % + Gold % | cities only |
| Ant. / Expl. | Wine | Culture | only during a Celebration (Golden Age) |
| Exploration | Furs | Gold | only during a Celebration |

⚠️ `REQUIREMENT_CITY_HAS_BUILD_QUEUE` is the most common way of writing "cities only" — **a town does
not have a build queue**. 29 resource modifiers are gated this way. Code that ignores it
will assign towns +10 gold from jade, +10 culture from silk and +4 production
from lapis lazuli, none of which will ever materialize there.

⚠️ `REQUIREMENT_PLAYER_IS_IN_GOLDEN_AGE` is the only condition concerning the **player** rather than a settlement.
It cannot be satisfied by choosing a settlement, so it should not affect where a resource goes —
and when totalling the empire's yield it has to be counted separately, because outside a Celebration it pays nothing.

---

## 5. The kinds of effects resources actually have

Not every resource modifier grants a yield. The full set of shapes found in the data:

| Effect shape | What it means | Examples |
|---|---|---|
| `CITY_ADJUST_YIELD_PER_RESOURCE` | a flat yield per settlement | Fish, Pearls, Ivory |
| `CITY_ADJUST_YIELD_PER_AVAILABLE_RESOURCE_TYPE` | the same, counted differently | Gold, Silver, Wine, Furs |
| `PLAYER_ADJUST_YIELD_PER_RESOURCE_TYPE` | a yield for the player, once | — |
| `UNIT_ADJUST_COMBAT_STRENGTH_PER_RESOURCE` | combat strength, **capped at +6** | Niter, Coal, Oil, Rubber |
| `CITY_ADJUST_CONSTRUCTIBLE_PRODUCTION_PER_RESOURCE` | a % towards building something | Coal, Oil, Marble |
| `*_ADJUST_UNIT_PRODUCTION_*` | cheaper units | Truffles, Salt, Cotton, Incense |
| `CITY_ADJUST_CONSTRUCTIBLE_YIELD_PER_RESOURCE` with `Tag=WAREHOUSE` | scales with the number of warehouses | Turtles, Clay, Crabs |
| `ADJUST_PLAYER_YIELD_PER_SLOTTED_RESOURCE` | a % of a yield, factory resources | Cocoa, Tea, Kaolin |
| `CITY_ADJUST_GROWTH_PER_RESOURCE` | a % of the growth rate | Tin |
| `UNIT_ADJUST_HEAL_PER_RESOURCE` | healing units | Quinine |
| `PLOT_PLACE_RESOURCE` | not a player effect — placement on the map | Gypsum, Horses |

⚠️ **The +6 combat strength cap does not exist in the data.** Every one of those resources has "(maximum +6)"
in its description, but no argument, global parameter or table carries it — it is held by
the engine. You have to record it as a constant and know that a patch can invalidate it.

⚠️ **Two factory effects keep the number in a `Percent` argument, not `Amount`**, and one
names `ConstructibleClass` instead of `ConstructibleType`. Code written for the shapes of empire
resources will read all of them as **zero**.

⚠️ **All four suffixes scale with the number of copies** — `PER_RESOURCE`,
`PER_AVAILABLE_RESOURCE_TYPE`, `PER_RESOURCE_TYPE`, `PER_SLOTTED_RESOURCE`. The name suggests
that the `_TYPE` ones pay once for the whole empire; **measurement in game says otherwise**. What
actually differs is the **scope**, and that comes from `collection`.

---

## 6. The collections used by resources

| Collection | How many modifiers | Scope |
|---|---|---|
| `COLLECTION_ALL_CITIES` | 115 | once per settlement that the requirements let through |
| `COLLECTION_ALL_UNITS` | 10 | the army |
| `COLLECTION_ALL_PLAYERS` | 10 | once, for the player |
| `COLLECTION_ALL_CAPITAL_CITIES` | 6 | the capital only |

⚠️ Reading the requirements alone **is not enough**. Furs give +3 Happiness through
`COLLECTION_ALL_CAPITAL_CITIES` — once, in the capital — and counting that in every settlement multiplies
the result by the size of the empire.

---

## 7. Methodological notes

- **A name in the data is a hypothesis, a measurement in a running game is a fact.** Several errors in this
  area came from reading an effect's name as if it were a specification.
- **Treat what you do not understand as satisfied.** When evaluating requirements, being too eager
  costs you a somewhat inaccurate result; being too strict **cuts the resource out of consideration entirely**, which is
  much worse and much harder to notice.
- ❓ **The tables below cover `Base/modules` only.** DLC (`DLC/*/modules`) may add
  and override resources — that has not been checked.
- Factory resources: **a settlement runs only one kind at a time, but any number of copies**
  (`LOC_PEDIA_CONCEPTS_FACTORY_RESOURCES_TOOLTIP`).

---

## 8. The complete effect tables, per age

Generated from the game's files. An "Amount" with `%` is a percentage value. Rows with no resource name
belong to the resource in the row above.

### age-antiquity

| Resource | Class | Effect | Amount | Condition |
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

| Resource | Class | Effect | Amount | Condition |
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

| Resource | Class | Effect | Amount | Condition |
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
