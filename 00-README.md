# Baza wiedzy: modowanie Civilization VII

Notatki budowane przez kolejne sesje pracy z AI nad modami do Civ VII.
Źródłem prawdy są **pliki zainstalowanej gry** i **49 działających modów ze Steam Workshop**,
a nie dokumentacja producenta (Firaxis praktycznie jej nie wydał).

## ⚠️ ZASADA NADRZĘDNA: baza wiedzy jest żywa

**Za każdym razem, gdy podczas pracy nad modem wyjdzie na jaw coś nowego —
trzeba to tu dopisać.** Baza wiedzy nie jest dokumentem archiwalnym: jej wartość
bierze się z tego, że rośnie razem z doświadczeniem.

Dotyczy to zarówno agenta AI, jak i użytkownika. Szczegółowa procedura:
[24-kb-maintenance.md](24-kb-maintenance.md).

**Co zapisywać:**
- rzecz, która zadziałała inaczej, niż opisano w bazie → **popraw wpis**
- ⚠️ lub ❓ potwierdzone w praktyce → **zmień na ✅** i dopisz jak zweryfikowano
- błąd, który kosztował więcej niż kilka minut → **[14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)**
- nowa tabela / efekt / API, którego tu nie ma → do właściwego pliku tematycznego
- rozwiązanie nieoczywistego problemu → cookbook albo quirks

**Kiedy:** od razu po ustaleniu faktu, nie „później". Później nie ma.

## Konwencja oznaczeń

W całej bazie wiedzy stosuję znaczniki wiarygodności — to ważne, żeby przyszłe sesje
nie traktowały domysłów jak faktów:

- ✅ **zweryfikowane** — sprawdzone bezpośrednio w plikach gry lub w działającym modzie
- ⚠️ **wywnioskowane** — logiczny wniosek ze schematu/wzorca, ale nie potwierdzony w działaniu
- ❓ **otwarte pytanie** — sprzeczne dane albo brak weryfikacji; wymaga testu w grze

## Skąd pochodzą dane

| Źródło | Ścieżka |
|---|---|
| Instalacja gry | `C:\Program Files (x86)\Steam\steamapps\common\Sid Meier's Civilization VII` |
| Dokumentacja społeczności ⚠️ | `civ7community.mintlify.app` — patrz [22](22-source-evaluation.md) o jej wiarygodności |
| Framework TypeScript | `github.com/izica/civ7-modding-tools` |
| Mody z Workshop (49 szt.) | `C:\Program Files (x86)\Steam\steamapps\workshop\content\1295660` |
| Schematy SQL | `Base\Assets\schema\` |
| Moduły gry bazowej | `Base\modules\{core,base-standard,age-antiquity,age-exploration,age-modern}` |
| Moduły DLC | `DLC\<nazwa>\modules\` |
| **Logi gry** (debugowanie) | `C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Logs\` |
| Baza moddingu | `...\Firaxis Games\Sid Meier's Civilization VII\Mods.sqlite` |
| **Mody użytkownika** ❗ | `C:\Users\najan\AppData\Local\Firaxis Games\Sid Meier's Civilization VII\Mods\` |

## Spis treści

**Fundamenty**
- [01-architecture.md](01-architecture.md) — układ instalacji, moduły, pipeline ładowania, scope
- [02-database.md](02-database.md) — warstwa bazodanowa, XML vs SQL, operacje, kluczowe tabele
- [03-modifiers-effects.md](03-modifiers-effects.md) — system GameEffects: modyfikatory, kolekcje, wymagania
- [04-ages-and-civilizations.md](04-ages-and-civilizations.md) — epoki, cywilizacje, tradycje, drzewa rozwoju
- [05-ui-javascript.md](05-ui-javascript.md) — architektura UI (**stary** framework `ui/`), Controls, dekoratory, zdarzenia
- [25-ui-next-solidjs.md](25-ui-next-solidjs.md) — ⚠️ **drugi framework UI: `ui-next` na Solid.js**;
  zawiera też ✅ **jak budować ładne tooltipy** z komponentów gry zamiast gołego `data-tooltip-content`;
  nowe ekrany są tam i `Controls.decorate` ich nie dotyka

**Cookbooki (przepisy krok po kroku)**
- [06-cookbook-new-civilization.md](06-cookbook-new-civilization.md) — nowa cywilizacja
- [07-cookbook-new-leader.md](07-cookbook-new-leader.md) — nowy lider
- [08-cookbook-units-buildings-traditions.md](08-cookbook-units-buildings-traditions.md) — jednostki, budynki, tradycje
- [09-cookbook-ui-mod.md](09-cookbook-ui-mod.md) — mod modyfikujący interfejs

**Proces**
- [24-kb-maintenance.md](24-kb-maintenance.md) — ⚠️ **jak i kiedy aktualizować tę bazę wiedzy**

**Praktyka**
- [10-tools-frameworks.md](10-tools-frameworks.md) — narzędzia i biblioteki
- [11-distribution-and-managers.md](11-distribution-and-managers.md) — dystrybucja, Workshop, menedżery modów
- [12-assets-icons-localization.md](12-assets-icons-localization.md) — grafika, ikony, zasoby
- [23-localization-i18n.md](23-localization-i18n.md) — **lokalizacja i i18n**: języki, liczba mnoga, rodzaj, odmiana przez przypadki
- [13-references-and-community.md](13-references-and-community.md) — społeczność i zasoby zewnętrzne
- [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md) — pułapki, dziwactwa, rzeczy które zaskakują
- [19-workflow-and-debugging.md](19-workflow-and-debugging.md) — workflow pracy i debugowanie

**Mapy konkretnych ekranów**
- [26-commerce-screen.md](26-commerce-screen.md) — ekran Handlu (zasoby + szlaki handlowe),
  czyli `screen-resource-allocation` napisany w `ui-next`

**Materiał referencyjny**
- [27-resources.md](27-resources.md) — ⚠️ **co daje który zasób i pod jakim warunkiem**, per epoka;
  wzorzec „bonus rozgałęziony" (ryby z portem 8 / bez portu 4) i dlaczego łamie naiwne
  pytanie „czy warunek jest spełniony"
- [15-schema-reference.md](15-schema-reference.md) — kolumny najważniejszych tabel
- [16-ui-source-reference.md](16-ui-source-reference.md) — jak czytać źródła UI gry (TypeScript!)
- [17-advanced-and-undocumented.md](17-advanced-and-undocumented.md) — rzeczy nieudokumentowane
- [18-reference-enumerations.md](18-reference-enumerations.md) — pełne listy efektów/kolekcji/wymagań

**Wiedza z dokumentacji społeczności**
- [20-typescript-tooling.md](20-typescript-tooling.md) — `civ7-modding-tools`: mody w TypeScript
- [21-gameplay-mechanics.md](21-gameplay-mechanics.md) — jak Civ VII działa jako gra (czego NIE ma!)
- [22-source-evaluation.md](22-source-evaluation.md) — ⚠️ **przeczytaj przed korzystaniem z zewnętrznych źródeł**

## Jeśli wracasz do pracy nad modem — start tutaj

1. **[24-kb-maintenance.md](24-kb-maintenance.md)** — zasada aktualizowania tej bazy
2. **[19-workflow-and-debugging.md](19-workflow-and-debugging.md)** — logi i pętla pracy;
   `console.log` **nie** trafia do `UI.log`, używaj `console.error`
3. **[14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)** — 31 pułapek; przejrzyj, zanim
   zaczniesz debugować cokolwiek „dziwnego"
4. Mod „Better Specialists UI" — źródła i stan projektu:
   `../mod-projects/najane-common-specialists-yields/README.md`
   (edytuj tam, wdrażaj `./deploy.sh` — **nie** edytuj folderu gry, deploy go nadpisze)

**Cztery rzeczy, które kosztowały najwięcej czasu w praktyce** — warto mieć z tyłu głowy:
- mod „się ładuje", jest zaznaczony w menu Modów, a jego skrypty milczą → sprawdź
  **listę włączonych modów w `Modding.log`** tuż przed „Applying mod components".
  Nie ma go tam? Albo nie jest włączony (#37), albo ma `version="0.x"` w `.modinfo`,
  co daje `Version = 0` i ciche pominięcie (#39)
- mod „się ładuje", ale nic nie robi → sprawdź, czy patchowany obiekt **już istnieje**
  w momencie patcha (warstwy soczewek rejestrują się później niż skrypty moda)
- zmiana widoczna w kodzie, ale nie w grze → sprawdź, czy nie wstrzykujesz DOM do
  kontenera, który gra właśnie **ukrywa** w tym trybie
- akcja `UpdateDatabase` cicho nie działa → jeden plik w złym zakresie (`game` vs `shell`)
  wywołuje **rollback całej akcji**, nie tylko tego pliku

## Liczby, które warto mieć w głowie

- **493** tabele w bazie gameplay
- **387** typów efektów (`EFFECT_*`), **38** kolekcji (`COLLECTION_*`), **270** typów wymagań (`REQUIREMENT_*`)
- **8640** modyfikatorów zdefiniowanych w samej grze bazowej
- **~1400** plików JS UI + **~1419** sourcemap z **pełnym oryginalnym kodem TypeScript**
