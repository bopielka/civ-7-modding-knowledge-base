# 17 — Rzeczy zaawansowane i nieudokumentowane

Rzeczy, których nie znajdziesz w żadnym poradniku, bo wynikają z czytania plików gry.

## 1. Własne właściwości w `.modinfo` ✅

Mod `bz-map-trix` deklaruje niestandardowe tagi:
```xml
<Properties>
    <bzIcon>blp:ntf_choosenarrative_blk</bzIcon>
    <bzIconGlow>#e5d2ac</bzIconGlow>
    <bzIconScale>...</bzIconScale>
    <bzIconCrop>...</bzIconCrop>
    <Teaser>...</Teaser>
    <LastUpdated>...</LastUpdated>
</Properties>
```
Działa, bo schemat moddingu przechowuje właściwości jako **pary nazwa/wartość**
(`ModProperties(ModRowId, Name, Value)`) — parser nie waliduje listy dozwolonych nazw ✅.

Konsekwencja: **mod może czytać własne metadane innych modów** i budować na tym
ekosystem (bz-map-trix ma własny system ikon dla rodziny modów `bz-`).

## 2. `ModProperties` jako kanał komunikacji między modami ⚠️

Skoro właściwości trafiają do bazy moddingu, mod A może odpytać o właściwości moda B.
Nie zweryfikowałem API dostępu z poziomu JS, ale mechanizm istnieje w schemacie.

## 3. Mod może tworzyć własne tabele ✅

```sql
CREATE TABLE IF NOT EXISTS CivsWithoutBackgrounds(
    CivilizationType TEXT PRIMARY KEY, ArtPath TEXT NOT NULL);
INSERT OR REPLACE INTO CivsWithoutBackgrounds VALUES('CIVILIZATION_POLAND','fs://...');
```
Mod Polska tak robi — prawdopodobnie po to, by inny mod (`custom-civ-art-fixes`?)
mógł to odczytać. To de facto **protokół międzymodowy**.

## 4. `ActionGroupRelationships` ⚠️

Schemat moddingu ma tabelę wiążącą grupy akcji różnych modów:
```sql
ActionGroupRelationships(ActionGroupRowId, OtherModId, OtherActionGroupId, Relationship)
```
Sugeruje to możliwość deklarowania relacji na poziomie **pojedynczej grupy akcji**,
a nie całego moda. Nie znalazłem składni XML, która to wypełnia — ❓ do zbadania.

## 5. `ModCompatibilityWhitelist` ✅ (schemat)

```sql
ModCompatibilityWhitelist(ModRowId, GameVersion)
```
Gra śledzi mody, które mają pomijać ostrzeżenia o niezgodności z wersją.
❓ Nie wiem, czy da się to ustawić z `.modinfo`.

## 6. `EpicMods` ✅ (schemat)

Osobna tabela na mody z wersji Epic Games Store — istotne tylko, jeśli mod ma działać
na obu platformach.

## 7. `Migrations` ✅ (schemat)

```sql
Migrations(SQL, MinVersion, MaxVersion, SortIndex)
```
System migracji bazy przy aktualizacji gry — mechanizm Firaxis, ale pokazuje, że
baza moddingu jest wersjonowana (`PRAGMA user_version(5)`).

## 8. Kryteria konfiguracyjne ✅ (w grze bazowej)

Poza `AgeInUse`/`ModInUse` istnieją:
```xml
<ConfigurationValueMatches>
    <ConfigurationId>...</ConfigurationId>
    <Group>...</Group>
    <Value>...</Value>
</ConfigurationValueMatches>
<ConfigurationValueContains>...</ConfigurationValueContains>
<RuleSetInUse>...</RuleSetInUse>
<GameModeInUse>...</GameModeInUse>
<AgeAtOrBefore>AGE_MODERN</AgeAtOrBefore>
<ModIsEnabled>...</ModIsEnabled>
```
Żaden z 49 modów Workshop ich nie użył — pole do wykorzystania, np. mod aktywny tylko
przy konkretnym ustawieniu rozgrywki.

## 9. Kryterium odwrotne ⚠️

Schemat: `Criterion(CriterionRowId, CriteriaRowId, CriterionType, Inverse)`.
Kolumna `Inverse` sugeruje możliwość negacji kryterium (np. „gdy mod NIE jest obecny").
❓ Nie znalazłem atrybutu XML, który to ustawia — prawdopodobnie `inverse="true"`
na elemencie kryterium. **Warto przetestować.**

## 10. `blp:` — wbudowane zasoby gry ✅

```sql
('UNIT_POLAND_HUSSAR','PORTRAIT_MASK','blp:unitflag_hussar',0)
```
```xml
<bzIcon>blp:ntf_choosenarrative_blk</bzIcon>
```
Pozwala użyć grafiki gry bez dołączania własnej. Nazwy trzeba wyłuskać z plików gry:
```bash
grep -rho "blp:[a-z0-9_]*" "$G/Base/modules/"*/data/ | sort -u | head -50
```

## 11. `InsertOrIgnore` na `Kinds` ✅ — wzorzec Firaxis

```xml
<Kinds>
    <InsertOrIgnore Kind="KIND_TRAIT"/>
    <InsertOrIgnore Kind="KIND_VICTORY"/>
</Kinds>
```
Deklaruje potrzebne `Kind` bez wysypania się, jeśli już istnieją. **Kopiuj ten wzorzec**
— czyni moda odpornym na kolejność ładowania.

## 12. `InheritFrom` w `Leaders` ✅

`InheritFrom="LEADER_DEFAULT"` pozwala zdefiniować lidera **czterema kolumnami**
zamiast jedenastoma. Klucz obcy wskazuje na `Leaders` — czyli można też dziedziczyć
po dowolnym istniejącym liderze.
⚠️ Sprawdź, czy `Civilizations` ma analogiczny mechanizm — nie ma kolumny `InheritFrom`,
więc raczej nie.

## 13. `Modifiers` — flagi kontroli kumulacji ✅

```xml
<Modifier id="..." collection="..." effect="..."
          permanent="true" run-once="true" new-only="true"
          owner-stack-limit="1" subject-stack-limit="1">
```
`owner-stack-limit` / `subject-stack-limit` zapobiegają wielokrotnemu nakładaniu się
tego samego bonusu — rzadko używane przez modderów, a rozwiązują realne bugi balansu.

## 14. `Requirements` ma kolumny AI ✅

```
AiWeighting, BehaviorTree, Impact, Likeliness, ProgressWeight, Persistent, Triggered, Reverse
```
Poza logiką warunku, wymaganie niesie **wskazówki dla AI**, jak ważny jest dany warunek.
Modderzy zwykle to pomijają — a to wpływa na zachowanie komputera.

## 15. Warstwa `ui-next` jest w trakcie migracji ⚠️

Proporcje (core: 373 stare vs 186 nowe; base-standard: 601 vs 132) sugerują, że Firaxis
stopniowo przepisuje UI na Solid.js. **Ryzyko dla modów**: komponent, który dziś
dekorujesz w `ui/`, może w kolejnym patchu przenieść się do `ui-next/`.
Mod `bz-map-trix` asekuruje się, mając pliki w obu drzewach.

## Lista rzeczy do przetestowania

- [ ] `inverse="true"` na kryterium
- [ ] `VisualRemaps` z `Kind=LEADER`
- [ ] czy plik zasobu bez rozszerzenia w `ImportFiles` jest wymagany
- [ ] czy alias `fs://game/` może być inny niż `<Mod id>`
- [ ] `ActionGroupRelationships` — składnia XML
- [ ] odczyt `ModProperties` innego moda z poziomu JS
- [ ] czy powrót do menu głównego przeładowuje mody bez restartu gry
      (sugerują to wpisy `Reason: Main Menu Reset` w `Modding.log`)
- [ ] `ACTION_GROUP_BUNDLE` z `civ7-modding-tools` — jaki `.modinfo` faktycznie generuje

## Strony dokumentacji społeczności jeszcze nieprzejrzane

Z `civ7community.mintlify.app` przejrzałem: documentation-guide, modding-architecture,
mod-patterns, typescript-overview, typescript-technical, environment-setup,
howto/creating-civilizations, howto/advanced-techniques, general-creating-leaders,
reference/modding-reference, reference/gameplay-mechanics, ages/age-gameplay-mechanics.

Zostały (warte zajrzenia przy konkretnym zadaniu):
- `guides/getting-started`, `guides/general-modifying-content`
- `guides/database-schemas`, `guides/base-standard-module`
- `guides/ages/age-modules`, `guides/ages/age-architecture`
- `guides/general-creating-civilizations`
- `guides/typescript/howto/`: creating-units, creating-buildings, leaders-and-ages,
  unique-quarters, traditions, progression-trees, modifiers-and-effects, assets-and-icons
- `guides/examples/dacia-*` (4 strony — pełny przykład implementacji cywilizacji)
- `reference/file-paths-reference`, `reference/modding-guide-civs-leaders`

⚠️ Przy każdej z nich pamiętaj o zasadzie z [22-source-evaluation.md](22-source-evaluation.md):
identyfikatory weryfikuj w schemacie gry.
