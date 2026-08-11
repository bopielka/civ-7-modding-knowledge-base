# 10 — Narzędzia i biblioteki

## Co jest w grze (vendor) ✅

`Base\modules\core\vendor\` zawiera biblioteki, których używa samo UI gry:

- **Solid.js** — `core/vendor/solid-js/` — reaktywny framework nowego UI (`ui-next`)
  - `solid-js/web/dist/web.js` (render), `solid-js/dist/solid.js` (rdzeń)
- **cohtml** — `core/ui/cohtml.js` — most do silnika Coherent Labs (renderer UI)

⚠️ Dla modów oznacza to: jeśli dopisujesz do `ui-next`, możesz korzystać z Solid.js
importując go z `/core/vendor/solid-js/...` — nie musisz dołączać własnej kopii.

## Czego NIE ma ✅

- ❌ brak plików `.d.ts` **w plikach gry** — zero gotowych definicji typów
- ❌ brak **oficjalnego** SDK/ModBuddy od Firaxis (w przeciwieństwie do Civ V)
- ❌ brak publicznego narzędzia do budowania paczek zasobów `.dep`
  → **modele 3D są praktycznie poza zasięgiem społeczności** (patrz [07](07-cookbook-new-leader.md))

## ⚠️ KOREKTA: nieoficjalne SDK jednak istnieje

Pierwotnie napisałem tu, że „nie ma SDK". To było błędne — wynikało z analizy
wyłącznie plików lokalnych, bez sprawdzenia zasobów społeczności.

**`civ7-modding-tools`** (npm, `github.com/izica/civ7-modding-tools`) to aktywnie
rozwijany framework TypeScript, który generuje pliki moda z typowanego kodu.
Pełny opis: **[20-typescript-tooling.md](20-typescript-tooling.md)**.

Nie zmienia to faktu, że gra nie dostarcza `.d.ts` — narzędzie ma własne typy i stałe.

## Co masz mimo to — sourcemapy ✅ (najważniejsze narzędzie)

**~1419 plików `.js.map`, z czego praktycznie wszystkie zawierają pełny oryginalny
kod TypeScript** w polu `sourcesContent`.

```json
{"version":3,"file":"framework.js",
 "sources":["../../../modules/core/ui/framework.ts"],
 "sourcesContent":["/**\n * @file framework.ts\n * @copyright 2021-2024, Firaxis Games\n ..."]}
```

To zastępuje dokumentację i definicje typów. Jak z tego korzystać — patrz
[16-ui-source-reference.md](16-ui-source-reference.md).

Dodatkowo **JavaScript gry nie jest zminifikowany** — czytelne nazwy zmiennych,
komentarze, `console.error` z opisami. Można czytać wprost.

## Narzędzia, które wystarczą do pracy

| Potrzeba | Narzędzie |
|---|---|
| Edycja XML/SQL/JS | dowolny edytor (VS Code) |
| Przeszukiwanie plików gry | `ripgrep` / grep — najważniejsza umiejętność |
| Podgląd bazy SQLite | DB Browser for SQLite — otwórz `Mods.sqlite` (patrz [19](19-workflow-and-debugging.md)) |
| Duży mod z cywilizacją | `civ7-modding-tools` ([20](20-typescript-tooling.md)) — opcjonalnie |
| Źródła TS interfejsu gry | `..\tools\extract_ts.py` ([16](16-ui-source-reference.md)) |
| Grafika 2D (ikony) | dowolny edytor PNG z alfą |
| Modele 3D | ⛔ brak realnej ścieżki — używaj `VisualRemaps` |

Zainstalowane u Ciebie środowisko ✅: Node.js v20.19.5, Python 3.14.0 —
oba wystarczają (`civ7-modding-tools` wymaga Node 14+).

## Wzorce, których używa społeczność (zaobserwowane) ✅

Z analizy 49 modów — brak jakiegokolwiek build-systemu. Mody to **surowe pliki**
wrzucone do folderu:

- brak `package.json`, brak bundlerów, brak transpilacji
- kod pisany bezpośrednio w JS (ES modules), nie w TS
- prefiksowanie plików i klas inicjałami autora (`bz-`, `leugi-`, `drongos-`)
  — prosta konwencja unikania kolizji nazw
- CSS jako osobne pliki ładowane przez `ImportFiles` + `Controls.loadStyle`

## Biblioteka wzorców do podpatrzenia

Najlepsze mody referencyjne z zainstalowanych 49 (wg złożoności i jakości kodu):

| Mod | Workshop ID | Czego uczy |
|---|---|---|
| `bz-map-trix` | 3507072814 | zaawansowany UI, warstwy map, dekoratory, patch prototypów |
| `szczupakabra-poland` | 3768377608 | pełna cywilizacja w SQL, VisualRemaps |
| `f1rstdan-cool-ui` | 3510572267 | rozbudowany mod UI z własnymi importami |
| `leugi-diploribbon-tweaks` | 3537808797 | `ReplaceUIScript`, dużo grafiki |
| `stachs-elegant-policies-and-traditions` | 3730149478 | przepisanie ekranu polityk |
| `drongos-cheat-panel` | 3734207916 | narzędzie deweloperskie/debug |
| `maple-leaves-more-lens` | 3526524592 | soczewki mapy |

Ścieżka bazowa: `C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660\<ID>`

Pełne mapowanie ID → nazwa w [11-distribution-and-managers.md](11-distribution-and-managers.md).
