# 11 — Dystrybucja, Workshop, zarządzanie modami

## Trzy lokalizacje modów ✅

| Rodzaj | Ścieżka | Uwagi |
|---|---|---|
| **Mody użytkownika** | `C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\` | tutaj tworzysz własne; folder może trzeba założyć |
| **Steam Workshop** | `C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660\` | zarządzane przez Steam, **nie edytować** |
| **Gra i DLC** | `...\common\Sid Meier's Civilization VII\{Base,DLC}\` | tylko do odczytu |

> ⚠️ **KOREKTA (2026-08-08).** Wcześniej podawano tu
> `Documents\My Games\Sid Meier's Civilization VII\Mods\` — **błędnie**, to konwencja
> z Civ VI powtarzana też przez dokumentację społeczności. Mod umieszczony tam nie
> zostaje wykryty (`Discovered 0 mods.` w `Modding.log`). Zweryfikowane empirycznie
> przy pierwszym własnym modzie. W `Documents\My Games\...` są tylko `Saves\`.

`1295660` = AppID Civilization VII (z `steamapps\appmanifest_1295660.acf`).

## Włączanie modów

Menu główne gry → **Additional Content** → lista wykrytych modów.
Kontroluje to kolumna `Mods.Disabled` w bazie moddingu:
`0 = Automatic`, `1 = ExplicitEnable`, `-1 = ExplicitDisable` ✅ (ze schematu).

## Właściwości wpływające na dystrybucję ✅

```xml
<Properties>
    <ShowInBrowser>1</ShowInBrowser>       <!-- widoczny na liście modów -->
    <AffectsSavedGames>0</AffectsSavedGames> <!-- czy psuje kompatybilność zapisów -->
    <EnabledByDefault>1</EnabledByDefault>  <!-- używane przez DLC Firaxis -->
    <Package>Mod</Package>
    <URL>https://forums.civfanatics.com/resources/...</URL>
</Properties>
```

⚠️ `<Package>` — obserwowane wartości: `Mod` (44 mody), `MOD` (2 — najpewniej literówka,
ale działa, więc porównanie prawdopodobnie ignoruje wielkość liter ❓).
Gra bazowa używa `BaseGame`, DLC np. `Carlisle`.

## Zależności vs referencje ✅

```xml
<Dependencies>   <!-- twarde: mod wymaga tego do działania -->
    <Mod id="base-standard" title="LOC_MODULE_BASE_STANDARD_NAME" />
</Dependencies>
<References>     <!-- miękkie: wpływa na kolejność/kompatybilność, nie wymaga obecności -->
    <Mod id="detailed-map-tacks" title="LOC_MOD_DETAILED_MAP_TACKS_NAME" />
</References>
```
`bz-map-trix` deklaruje 5 `References` do innych modów UI — to sposób na
zadeklarowanie „wiem o tobie, ustaw się względem mnie", bez twardego wymogu.

Kryterium `ModInUse` pozwala włączyć akcję tylko, gdy inny mod jest obecny (6 użyć):
```xml
<Criteria id="gdy-inny-mod"><ModInUse>inny-mod-id</ModInUse></Criteria>
```

## Kompatybilność między modami — praktyka

Kolejność ryzyka, od najbezpieczniejszego:
1. `UpdateDatabase` dokładający **nowe** wiersze — praktycznie bezkonfliktowy
2. `UIScripts` + `Controls.decorate` — kilka modów może dekorować ten sam komponent
3. `UpdateDatabase` z `UPDATE`/`DELETE` na istniejących wierszach — wygrywa ostatni
4. `ReplaceUIScript` — tylko jeden mod może wygrać
5. `ImportFiles` nadpisujący plik gry — jak wyżej, plus psuje się przy patchach

## Pełna lista 49 zainstalowanych modów ✅

Foldery Workshop nazwane są ID przedmiotu, nie nazwą — stąd ta tabela.

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

Odświeżenie listy po zmianie subskrypcji:
```bash
W="/c/Program Files (x86)/Steam/steamapps/workshop/content/1295660"
for d in "$W"/*/; do
  mi=$(find "$d" -maxdepth 1 -iname "*.modinfo" | head -1)
  [ -n "$mi" ] && printf "%s\t%s\n" "$(basename "$d")" \
    "$(grep -o '<Mod id="[^"]*"' "$mi" | head -1 | sed 's/<Mod id="//;s/"//')"
done
```

## Publikacja własnego moda — Mod SDK ✅

**Gra NIE ma wbudowanego uploadera.** ✅ Sprawdzone w plikach: `core/ui/shell/mods-content/mods-content.js`
jedynie **wyświetla** treści z Workshop (`case "SteamWorkshopContent"`), nie publikuje.

Do publikacji służy **osobne narzędzie w Steam**:

- Wydane w **aktualizacji z czerwca 2025 (Update 1.2.2)**, razem z obsługą Steam Workshop
- Potrafi tworzyć, debugować, wyszukiwać i **wysyłać mody na Steam Workshop**
- Pobiera się z **sekcji „Narzędzia" (Tools) w bibliotece Steam** — nie ze sklepu jak zwykłą grę
- Widoczne tylko dla posiadaczy gry

⚠️ Narzędzia domyślnie **nie pokazują się** na liście biblioteki Steam — trzeba włączyć
filtr „Narzędzia"/„Tools" (Biblioteka → filtr typu treści) albo wyszukać po nazwie.

❓ Nie udało mi się potwierdzić dokładnej nazwy ani AppID narzędzia z dostępnych źródeł
(oficjalny wpis 2K zapowiada SDK, ale go nie nazywa; wątek „The SDK is now available"
na CivFanatics dotyczy **Civ VI**, nie VII — łatwo się pomylić przy wyszukiwaniu).
Sprawdź w bibliotece Steam pod filtrem Tools.

⚠️ Dla porównania: SDK do **Civ VI** nazywało się „Sid Meier's Civilization VI Development
Tools" i zawierało ModBuddy, FireTuner, narzędzia artystyczne oraz Workshop Uploader.
Nie zakładaj, że skład SDK do Civ VII jest identyczny — to była pierwsza wersja narzędzi
i Firaxis zapowiadał rozbudowę.

### Zanim opublikujesz — checklist
- [ ] `<Name>` i `<Description>` opisują, co mod faktycznie robi
- [ ] `ShowInBrowser=1`, poprawny `<Package>Mod</Package>`
- [ ] `AffectsSavedGames` zgodne z prawdą
- [ ] `<Authors>`, `<Version>`, opcjonalnie `<URL>`
- [ ] tłumaczenia (patrz [23-localization-i18n.md](23-localization-i18n.md)) — fallback na angielski działa,
      więc komplet nie jest wymagany
- [ ] miniatura/obrazek podglądu (wymóg Workshop, nie moda)
- [ ] test na czystej instalacji: czy mod działa bez Twoich innych modów

### Kanał alternatywny
**CivFanatics** — wiele modów podaje tam `<URL>` w `.modinfo` (np. `bz-map-trix`
linkuje do `forums.civfanatics.com/resources/...`). Działa też dla graczy z Epic,
którzy nie mają Workshop.
