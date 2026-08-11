# 19 — Workflow pracy i debugowanie

## Logi gry ✅ (znalezione i zweryfikowane)

```
C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Logs\
```

**To jest najważniejsze narzędzie debugowania.** 32 pliki logów, najistotniejsze:

| Log | Do czego |
|---|---|
| **`Modding.log`** | ładowanie modów, wykryte subskrypcje, lista aktywnych modów, rekonfiguracja |
| **`Database.log`** | walidacja bazy, **błędy XML/SQL**, klucze obce |
| **`UI.log`** | błędy JS, brakujące zasoby graficzne, błędy konwersji argumentów |
| `Startup.log`, `General.log` | ogólny przebieg startu |
| `Localization.log` | problemy z tekstami |
| `ArtDef.log`, `Renderer.log`, `VFXSystem.log` | grafika |

### Format wpisów ✅

```
[2026-08-08 21:53:47]	Subscription Service - Detected 49 subscriptions
[2026-08-08 21:53:47]	bz-map-trix (Map Trix)                     ← id (nazwa wyświetlana)
[2026-08-08 21:53:47]	Successfully reconfigured game.
```

```
[2026-08-08 21:53:47]	[frontend]: Validating Foreign Key Constraints...
[2026-08-08 21:53:47]	[frontend]: Passed Validation.
[2026-08-08 21:53:47]	[localization]: Database XML root elements must start with
                                        either <Database> or <GameEffects>.
```

⚠️ **Ta linia to autorytatywna reguła:** root pliku XML musi być `<Database>` **albo**
`<GameEffects>` — nic innego nie przejdzie. (Powyższy wpis to realny błąd w plikach gry
bazowej — nawet Firaxis go ma.)

```
[2026-08-08 21:54:05]	Failed to open file - lp_circ_alexander_256
[2026-08-08 21:54:05]	ResourceRequestJob | Failed loading resource: blp:lp_circ_alexander_256
[2026-08-08 21:54:17]	Argument conversion failed: Wrong type - expected String, got Null
                        while converting argument 0 for getIconBLP
```
Tak wyglądają błędy brakujących ikon i błędy JS — dokładnie to, co zobaczysz,
gdy twoje `IconDefinitions` będą niepoprawne.

## Baza moddingu ✅

```
...\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods.sqlite
```
To fizyczna baza ze schematu `schema-modding-10.sql` — możesz ją otworzyć
w DB Browser for SQLite i zobaczyć, **jak gra faktycznie zinterpretowała twój `.modinfo`**:
tabele `Mods`, `ModProperties`, `ActionGroups`, `Actions`, `ActionItems`, `Criteria`.

Nieocenione, gdy mod „się nie ładuje" i nie wiadomo dlaczego.

Obok: `LocalStorage.sqlite`, `HallofFame.sqlite`.

## Inne lokalizacje ✅

| Ścieżka | Zawartość |
|---|---|
| `AppData\Local\Firaxis Games\...\Mods\` | (u mnie pusty) — ⚠️ możliwa alternatywna lokalizacja modów |
| `AppData\Local\Firaxis Games\...\ModUserData\` | dane zapisywane przez mody (u mnie pusty) |
| `AppData\Local\Firaxis Games\...\dumps\`, `packagedDumps\` | zrzuty po crashach |
| `AppData\Local\Firaxis Games\...\AppOptions.txt`, `UserOptions.txt` | ustawienia (tekstowe) |

## Źródła osobno, wdrożenie skryptem ✅ (zalecane)

Trzymanie źródeł w folderze gry blokuje repozytorium: `.git`, README i skrypty
trafiałyby do folderu gracza. Rozdziel to:

```
Documents\Civ7Modding\mod-projects\<mod>\   ← źródło prawdy, repozytorium
        ↓  deploy.sh
%LOCALAPPDATA%\Firaxis Games\...\Mods\<mod>\  ← wynik, traktowany jak build
```

Wzorzec skryptu (działający przykład: `mod-projects/najane-common-specialists-yields/deploy.sh`):

- **kopiuj listę dozwolonych, nie wykluczaj** — kopiuj tylko `.modinfo`, `ui/`, `text/`.
  Wtedy README, `deploy.sh` i `.git/` nie trafią do gry **z konstrukcji**, a nie dzięki
  liście wykluczeń, która z czasem się rozjedzie.
- **wyczyść cel przed kopiowaniem** — inaczej plik usunięty w repo zostaje w grze
  jako duch. Cel to build, nie magazyn.
- **zabezpiecz ścieżki** — sprawdź, czy katalog docelowy kończy się na id moda, zanim
  wywołasz `rm -rf`. Literówka w zmiennej nie może skasować czegoś innego.
- **weryfikuj po wdrożeniu** — sprawdź, czy każdy `<Item>`/`<File>` z `.modinfo`
  faktycznie istnieje w celu. Wyłapuje literówki w ścieżkach, zanim zrobi to gra.
- **dodaj `--dry`** — pokazuje, co zostanie skopiowane, bez zmian.

## Pętla pracy

```
1. edytuj pliki moda w Documents\Civ7Modding\mod-projects\moj-mod\ i uruchom ./deploy.sh
   (albo, bez repozytorium, wprost w AppData\Local\Firaxis Games\...\Mods\moj-mod\)
2. uruchom grę (albo wróć do menu głównego — patrz niżej)
3. sprawdź Modding.log  → czy mod się w ogóle załadował
4. sprawdź Database.log → czy dane przeszły walidację
5. sprawdź UI.log       → czy JS/ikony działają
6. testuj w grze
```

⚠️ W `Modding.log` widać wpisy `Reason: Main Menu Reset` i `Reason: Script Reset`
— sugeruje to, że **powrót do menu głównego przeładowuje mody**, bez restartu gry.
❓ Nie potwierdziłem tego eksperymentalnie, ale warto spróbować — oszczędza mnóstwo czasu.

## Zasady iteracji

1. **Zacznij od najmniejszej zmiany, która da widoczny efekt.**
   Np. `UPDATE Units SET BaseMoves=5 WHERE UnitType='UNIT_SCOUT'` — natychmiast widać.
2. **Dodawaj po jednym pliku.** Przy 53 plikach naraz nie znajdziesz, co zepsuło.
3. **Najpierw dane, potem grafika.** Ikony są najbardziej upierdliwe i najmniej krytyczne.
4. **Trzymaj wersję działającą.** Kopiuj folder przed większą zmianą.
5. **Zapisz, czego się nauczyłeś.** ⬅ krok, o którym najłatwiej zapomnieć

## Krok 7 pętli: aktualizacja bazy wiedzy ⚠️ obowiązkowy

Każde ustalenie, które nie wynikało wprost z bazy wiedzy, **trafia z powrotem do bazy**
— zanim skończysz sesję. Dotyczy to zarówno agenta AI, jak i Ciebie.

Najczęstsze przypadki:
- coś zadziałało inaczej, niż opisano → **popraw wpis**
- ⚠️ lub ❓ potwierdzone w praktyce → **zmień na ✅**, dopisz jak sprawdzono
- błąd, który kosztował więcej niż kilka minut → **[14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)**

Pełna procedura, mapa „co gdzie zapisać" i wzorce korekt:
**[24-kb-maintenance.md](24-kb-maintenance.md)**.

Uzasadnienie: wiedza, która zostanie tylko w oknie czatu, przepada przy następnej
sesji — a to jest dokładnie ten problem, dla którego ta baza powstała.

## Typowe objawy → przyczyny

| Objaw | Gdzie szukać | Prawdopodobna przyczyna |
|---|---|---|
| **Moda nie ma w `Modding.log` W OGÓLE** | `Modding.log` → `Discovered 0 mods.` | **mod w złym folderze** — musi być w `AppData\Local\Firaxis Games\...\Mods\`, nie w `Documents` |
| Moda nie ma na liście, ale jest w logu | `Modding.log` | błąd składni `.modinfo`, zły `<Package>`, `ShowInBrowser=0` |
| Mod na liście, ale nic nie robi | `Modding.log` + `Mods.sqlite` | kryterium niespełnione, zły `scope` |
| Błąd walidacji bazy | `Database.log` | brak wiersza w `Types`, złamany klucz obcy |
| XML zignorowany | `Database.log` | root nie jest `<Database>`/`<GameEffects>` |
| Widać `LOC_...` | `Localization.log` | brak `UpdateText` albo literówka w tagu |
| Brak ikony | `UI.log` (`Failed to open file`) | zła ścieżka `fs://game/...`, brak `ImportFiles` |
| Panel UI nie reaguje | `UI.log` | dekorator zarejestrowany za późno → podnieś `LoadOrder` |
| Gra się wysypuje | `dumps\` | zwykle błąd danych; sprawdź `Database.log` tuż przed |

## ⚠️ `console.log` NIE trafia do `UI.log` ✅

Sprawdzone empirycznie: wywołania `console.log()` z moda **nie zostawiają śladu**
w `Logs\UI.log` — zrzut diagnostyczny wypisany przez `console.log` nie pojawił się
tam wcale, mimo że kod na pewno się wykonał.

Do logowania diagnostycznego z moda używaj **`console.error()`** — te wpisy trafiają
do `UI.log` (widać je jako `JS Error`). Brzydkie, ale działa:

```js
console.error(`moj-mod: plot=${i} deltas=[${...}]`);
```
❓ Nie sprawdziłem `console.warn` ani `console.info` — możliwe, że któryś też przechodzi.

Wniosek praktyczny: planując „dodam log i sprawdzę w pliku", od razu pisz `console.error`,
inaczej stracisz cały cykl uruchamiania gry na nic.

## Szybkie komendy

```bash
L="/c/Users/najan/AppData/Local/Firaxis Games/Sid Meier's Civilization VII/Logs"

# czy mój mod się załadował
grep -i "moj-mod" "$L/Modding.log"

# błędy bazy z ostatniego uruchomienia
grep -iv "Passed Validation" "$L/Database.log" | tail -30

# błędy UI
grep -i "error\|failed\|warn" "$L/UI.log" | tail -30

# lista załadowanych modów
grep -E "^\[.*\]\t[a-z0-9-]+ \(" "$L/Modding.log" | tail -60
```

## Przeszukiwanie plików gry (najważniejsza umiejętność)

```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"

# jak Firaxis zrobił coś podobnego — TYLKO w data/ (14 MB, szybkie)
grep -rn "EFFECT_CITY_ADJUST_YIELD" "$G/Base/modules/"*/data/ | head

# kolumny dowolnej tabeli
S="$G/Base/Assets/schema/gameplay/01_GameplaySchema.sql"
awk -v w="CREATE TABLE 'Traditions' (" 'index($0,w)==1{f=1} f{print} f&&/^\);/{exit}' "$S"

# komponenty UI do dekoracji
grep -rn "Controls.define" "$G/Base/modules/base-standard/ui/" | head -40
```
⚠️ Nie rób `grep -r` po całym `$G` — 1,3 GB, zajmie minuty.

## Narzędzia pomocnicze w tym repo

- `..\tools\extract_ts.py` — wyciąga oryginalny TypeScript z sourcemap gry
  (patrz [16-ui-source-reference.md](16-ui-source-reference.md))
