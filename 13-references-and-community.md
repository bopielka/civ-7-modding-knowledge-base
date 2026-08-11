# 13 — Społeczność i zasoby zewnętrzne

> ⚠️ Ten plik zawiera najwięcej treści **niezweryfikowanych** — nie sprawdzałem
> tych źródeł online w trakcie analizy. Traktuj jako wskazówki, gdzie szukać,
> a nie jako potwierdzone fakty.

## Główne miejsca

- **Dokumentacja społeczności** — `civ7community.mintlify.app` ✅ (sprawdzona)
  Duży serwis z przewodnikami: architektura, tworzenie cywilizacji/liderów,
  narzędzia TypeScript, mechaniki rozgrywki, przykład implementacji (Dacia).
  ⚠️ **Zawiera błędy w szczegółach bazodanowych** — patrz
  [22-source-evaluation.md](22-source-evaluation.md). Świetna do wiedzy
  koncepcyjnej i projektowej, słaba jako źródło nazw tabel.
  Najcenniejsze strony:
  `/community/guides/typescript/*` (narzędzia), `/community/reference/gameplay-mechanics`,
  `/community/guides/mod-patterns`, `/community/guides/examples/*` (Dacia).
- **`civ7-modding-tools`** — `github.com/izica/civ7-modding-tools` ✅
  Framework TypeScript do generowania modów ([20](20-typescript-tooling.md)).
- **CivFanatics** — `forums.civfanatics.com`, sekcja zasobów Civ VII.
  ✅ Potwierdzone pośrednio: mody z Workshop podają tam `<URL>`, np.
  `https://forums.civfanatics.com/resources/map-trix.31950/`.
  To znaczy, że autorzy realnie tam publikują i dyskutują.
- **Steam Workshop** — główny kanał dystrybucji (AppID `1295660`).
- **Discord modderów Civ VII** — ⚠️ istnieje wg powszechnej wiedzy o społeczności
  Civ, ale nie mam zweryfikowanego zaproszenia.

## Autorzy, których mody warto śledzić ✅

Z 49 zainstalowanych modów wyłaniają się aktywni i kompetentni autorzy:

| Autor / prefiks | Mody | Specjalizacja |
|---|---|---|
| `beezany` (`bz-`) | map-trix, city-hall, flag-corps, friends, ready-or-not, a-la-mods, clean-slate | 7 modów — najbardziej zaawansowany kod UI |
| `leugi` | diploribbon-tweaks, diploicon-tweaks, happiness_stage_icons | grafika i UI dyplomacji |
| `drongos` | cheat-panel, top-panel, relationship-preview | narzędzia i panele |
| `orions` | bonus-icons-plus, clearer-agendas, victory-meter | ikonografia |
| `stachs`/`stachus` | elegant-policies-and-traditions, elegant-great-works | przepisywanie ekranów |
| `szczupakabra` | poland | pełna cywilizacja |
| `nasuellia` | non-sticky-selection, unit-flags | UX |

**Praktyczna rada:** zamiast szukać tutoriali, czytaj kod tych modów lokalnie.
Masz je wszystkie na dysku (patrz [11](11-distribution-and-managers.md)).

## Dlaczego dokumentacja oficjalna nie pomoże ✅

Firaxis nie wydał SDK ani dokumentacji modowania dla Civ VII (stan na moment analizy).
Nie ma:
- narzędzia typu ModBuddy (jak w Civ V)
- definicji typów `.d.ts` dla API UI
- opisu tabel bazy danych

**Za to gra sama jest dokumentacją:**
1. `Base\Assets\schema\` — pełne schematy z komentarzami
2. `Base\modules\` + `DLC\` — setki działających przykładów od Firaxis
3. sourcemapy z kodem TypeScript ([16](16-ui-source-reference.md))

To jest lepsze niż większość oficjalnych dokumentacji — tylko trzeba umieć szukać.

## Umiejętność nr 1: przeszukiwanie plików gry

```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"

# gdzie zdefiniowano dany efekt/typ
grep -rl "EFFECT_CITY_ADJUST_YIELD" "$G/Base/modules/"*/data/

# jak Firaxis zrobił podobną mechanikę
grep -rn "REQUIREMENT_PLAYER_HAS_CIVILIZATION_OR_LEADER_TRAIT" "$G/Base/modules/age-antiquity/data/" | head

# jakie komponenty UI można dekorować
grep -rn "Controls.define" "$G/Base/modules/base-standard/ui/" | head -40
```

⚠️ **Nie** rób `grep -r` po całym katalogu gry — to 1,3 GB i zajmuje minuty.
Zawężaj do `*/data/` (14 MB) albo `*/ui/`.

## Otwarte pytania do zbadania online

- ❓ Czy istnieje publiczne narzędzie do budowy paczek `.dep` (modele 3D)?
- ❓ Czy jest odpowiednik FireTuner / Live Tuner z Civ VI?
  (SDK do Civ VI go zawierał — czy SDK do Civ VII też, do sprawdzenia po instalacji)
- ~~Jak wygląda proces publikacji na Workshop~~ → ✅ **ustalone**: osobny Mod SDK
  z sekcji Tools w Steam, patrz [11-distribution-and-managers.md](11-distribution-and-managers.md)
- ~~Gdzie gra zapisuje logi~~ → ✅ **znalezione**, patrz [19](19-workflow-and-debugging.md):
  `AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Logs\`
