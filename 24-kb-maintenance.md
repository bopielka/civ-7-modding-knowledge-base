# 24 — Utrzymanie bazy wiedzy

Ta baza ma sens tylko wtedy, gdy rośnie razem z realnym doświadczeniem z modowania.
Ten plik opisuje, **kiedy i jak ją aktualizować**.

## Zasada

> **Każde nowe ustalenie z pracy nad modem trafia do bazy wiedzy — od razu.**

Dotyczy agenta AI i użytkownika tak samo. Wiedza, która zostanie tylko w oknie czatu,
przepada przy następnej sesji — a to jest dokładnie ten problem, dla którego ta baza
powstała.

## Instrukcja dla agenta AI

Pracując nad modem do Civ VII w tym projekcie:

1. **Na starcie sesji** przeczytaj [00-README.md](00-README.md) i pliki dotyczące
   bieżącego zadania. Nie wyprowadzaj ponownie rzeczy, które już tu są.
2. **W trakcie pracy** notuj każde ustalenie, które nie wynika wprost z bazy.
3. **Przed końcem sesji** (lub od razu po ważnym odkryciu) zapisz je w odpowiednim pliku.
4. **Nie pytaj o pozwolenie** na aktualizację bazy wiedzy — to zadanie stałe.
   Pytaj tylko wtedy, gdy trzeba by przebudować strukturę albo usunąć duży fragment.
5. **Powiedz użytkownikowi, co zaktualizowałeś** — krótko, na koniec odpowiedzi.

## Co kwalifikuje się do zapisania

✅ **Zapisuj:**
- coś zadziałało **inaczej niż opisano** w bazie → popraw wpis (nie dopisuj obok)
- ⚠️ / ❓ **potwierdzone lub obalone** w praktyce → zmień znacznik i dopisz, jak sprawdzono
- błąd, którego rozwiązanie zajęło **więcej niż kilka minut** → [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)
- nowa tabela, kolumna, efekt, wymaganie, API → właściwy plik tematyczny + ewentualnie
  [15-schema-reference.md](15-schema-reference.md) / [18-reference-enumerations.md](18-reference-enumerations.md)
- działający fragment kodu rozwiązujący nieoczywisty problem → odpowiedni cookbook
- nowe zewnętrzne źródło → [13-references-and-community.md](13-references-and-community.md)
  (po ocenie wiarygodności wg [22-source-evaluation.md](22-source-evaluation.md))

❌ **Nie zapisuj:**
- rzeczy już opisanych (najpierw sprawdź, potem pisz)
- szczegółów jednorazowych, specyficznych dla jednego moda i nieprzenośnych
- spekulacji bez oznaczenia ⚠️/❓
- treści z zewnętrznych źródeł **bez weryfikacji** w plikach gry

## Gdzie co trafia

| Rodzaj ustalenia | Plik |
|---|---|
| Struktura moda, akcje, kryteria, scope, LoadOrder | [01-architecture.md](01-architecture.md) |
| Tabele, XML/SQL, operacje bazodanowe | [02-database.md](02-database.md) |
| Modyfikatory, efekty, wymagania | [03-modifiers-effects.md](03-modifiers-effects.md) |
| Epoki, cywilizacje, tradycje, drzewa | [04-ages-and-civilizations.md](04-ages-and-civilizations.md) |
| UI, JS, komponenty, zdarzenia | [05-ui-javascript.md](05-ui-javascript.md) |
| Konkretny przepis krok po kroku | cookbooki [06](06-cookbook-new-civilization.md)–[09](09-cookbook-ui-mod.md) |
| Ikony, grafika, zasoby | [12-assets-icons-localization.md](12-assets-icons-localization.md) |
| Teksty, tłumaczenia, odmiana | [23-localization-i18n.md](23-localization-i18n.md) |
| **Pułapka / niespodzianka** | [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md) |
| Debugowanie, logi, workflow | [19-workflow-and-debugging.md](19-workflow-and-debugging.md) |
| Rzecz nieudokumentowana / eksperyment | [17-advanced-and-undocumented.md](17-advanced-and-undocumented.md) |

Jeśli temat nie pasuje nigdzie — **załóż nowy ponumerowany plik** i dopisz go
do spisu treści w [00-README.md](00-README.md).

## Jak aktualizować poprawnie

**Zmiana statusu weryfikacji** — dopisz, skąd wiadomo:
```markdown
- ⚠️ ~~Nie wiadomo, czy powrót do menu przeładowuje mody.~~
+ ✅ Powrót do menu głównego przeładowuje mody bez restartu gry
+   (sprawdzone 2026-08-10: zmiana w data/units.xml widoczna po powrocie do menu).
```

**Korekta błędu** — nie usuwaj po cichu, zaznacz korektę:
```markdown
> ⚠️ **KOREKTA.** Wcześniej napisano tu, że X. To było błędne — w rzeczywistości Y.
```
Widać wtedy, że temat był badany, i nie wraca się do tego samego złego wniosku.
Wzorzec: sekcja „KOREKTA" w [10-tools-frameworks.md](10-tools-frameworks.md).

**Nowa pułapka** — dopisz na końcu listy w `14`, kolejny numer, zawsze ze znacznikiem
i konkretem (objaw → przyczyna → rozwiązanie).

## Regeneracja plików generowanych

Dwa pliki powstają skryptem z plików gry — po aktualizacji gry warto je odświeżyć:

- [15-schema-reference.md](15-schema-reference.md) — kolumny tabel
- [18-reference-enumerations.md](18-reference-enumerations.md) — efekty, kolekcje, wymagania

Skrypty regenerujące są **wewnątrz tych plików**, na końcu.

## Przegląd okresowy

Co jakiś czas (np. po ukończeniu moda albo po dużym patchu gry):

- [ ] czy któreś ⚠️/❓ da się już rozstrzygnąć?
- [ ] czy lista otwartych pytań w [17](17-advanced-and-undocumented.md) jest aktualna?
- [ ] czy po patchu gry zmieniły się schematy? (regeneruj `15` i `18`)
- [ ] czy lista modów w [11](11-distribution-and-managers.md) zgadza się z subskrypcjami?
- [ ] czy któryś plik urósł tak, że warto go rozbić? (wzorzec: i18n wydzielone
      z `12` do `23`)
