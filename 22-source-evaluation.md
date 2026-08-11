# 22 — Wiarygodność źródeł i jak je weryfikować

Ten plik powstał, bo przy integrowaniu dokumentacji społeczności trafiłem na
**konkretne, sprawdzalne błędy**. Bez tej ostrożności baza wiedzy szybko zapełni się
rzeczami, które nie działają.

## Hierarchia wiarygodności

| Poziom | Źródło | Dlaczego |
|---|---|---|
| 1 (najwyższy) | **Schematy SQL gry** (`Base\Assets\schema\`) | maszyna czyta dokładnie to |
| 1 | **Logi gry** (`Logs\Database.log` itd.) | mówią, co gra faktycznie zrobiła |
| 2 | **Pliki gry i DLC** (`Base\modules\`, `DLC\`) | działający kod Firaxis |
| 3 | **Działające mody z Workshop** | działają, ale mogą mieć złe praktyki |
| 4 | **Dokumentacja społeczności** | pomocna, ale bywa błędna — patrz niżej |
| 5 | Pamięć modela / analogie z Civ VI | zawsze weryfikuj |

**Zasada:** twierdzenie z poziomu 4 sprawdź na poziomie 1 lub 2, zanim wpiszesz je
do bazy jako ✅.

## Konkretne błędy znalezione w dokumentacji społeczności ❗

Źródło: `civ7community.mintlify.app/community/guides/general-creating-leaders`

Przewodnik tworzenia lidera wymienia tabele, które **nie istnieją w Civ VII**:

| Tabela wg dokumentacji | Rzeczywistość ✅ |
|---|---|
| `LeaderCivilizations` | **nie istnieje** (0 trafień w schemacie) |
| `Agendas` | **nie istnieje** |
| `HistoricalAgendas` | **nie istnieje** |
| `RandomAgendas` | **nie istnieje** |

To są nazwy tabel z **Civilization VI**. Jedyne tabele z „Agenda" w Civ VII to
`DiplomacyAgendaAmountTypes`, `DiplomacyAgendaAwardToTypes`,
`DiplomacyAgendaWeightingTypes` — czyli zupełnie inny system.

Wszystkie istniejące tabele z „Leader" ✅:
`Leaders`, `LeaderCivPriorities`, `LeaderInfo`, `LeaderSyncretismUnlocks`,
`LeaderTraits`, `LegacyLeaderCivPriorities`, `LoadingInfo_Leaders`,
`Resource_RequiredLeaders`

Dodatkowo przykładowy XML w tym przewodniku ustawia atrybut `Description` na wierszu
`Leaders` — a tabela `Leaders` **nie ma kolumny `Description`** ✅
(kolumny: `LeaderType`, `AITargetCityPercentage`, `BasePersonaType`,
`DesiredNumAlliances`, `DiscountRate`, `InheritFrom`, `IsBarbarianLeader`,
`IsIndependentLeader`, `IsMajorLeader`, `Name`, `OperationList`).

**Wniosek:** ten konkretny przykład nie zadziałałby. Prawidłowy wzorzec —
z oficjalnego DLC — jest w [07-cookbook-new-leader.md](07-cookbook-new-leader.md).

## Prawdopodobna przyczyna

Dokumentacja sprawia wrażenie częściowo generowanej przez AI na bazie wiedzy
o Civ VI (charakterystyczne: pewny ton, poprawna struktura, błędne szczegóły).
To nie znaczy, że jest bezwartościowa — **opisy mechanik i wskazówki projektowe
są przydatne**. Ale każdy identyfikator tabeli/kolumny/typu trzeba sprawdzić.

## Jak weryfikować w 10 sekund

```bash
G="/c/Program Files (x86)/Steam/steamapps/common/Sid Meier's Civilization VII"
S="$G/Base/Assets/schema/gameplay/01_GameplaySchema.sql"

# czy tabela istnieje?
grep -c "CREATE TABLE 'LeaderCivilizations'" "$S"     # 0 = nie istnieje

# jakie tabele pasują do tematu?
grep -o "CREATE TABLE '[A-Za-z_]*Leader[A-Za-z_]*'" "$S" | sed "s/CREATE TABLE '//;s/'//"

# jakie kolumny ma tabela?
awk -v w="CREATE TABLE 'Leaders' (" 'index($0,w)==1{f=1} f{print} f&&/^\);/{exit}' "$S"

# czy typ/efekt istnieje w danych gry?
grep -rl "EFFECT_UNIT_ADJUST_COMBAT_STRENGTH" "$G/Base/modules/"*/data/
```

## Sygnały ostrzegawcze w źródłach

- 🚩 nazwy tabel/kolumn znane z Civ VI (`Amenities`, `Loyalty`, `Agendas`, `Eras`)
- 🚩 przykład XML/SQL bez wskazania, z którego pliku gry pochodzi
- 🚩 „prawdopodobnie", „zazwyczaj", „powinno" przy szczegółach technicznych
- 🚩 brak wzmianki o `scope="shell"` przy cywilizacjach (częsty realny wymóg)
- 🚩 stara składnia modyfikatorów (`ModifierType`/`CollectionType` zamiast
  `collection=`/`effect=`) — patrz [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)

## Co w dokumentacji społeczności jest wartościowe ✅

Mimo błędów w szczegółach bazodanowych, te obszary są przydatne i trudne do wywnioskowania
z samych plików:

- **`civ7-modding-tools`** — realne narzędzie, zweryfikowane repozytorium
  ([20-typescript-tooling.md](20-typescript-tooling.md))
- **Mechaniki rozgrywki** — czego w grze nie ma, jak działają epoki, kryzysy,
  niezależne siły ([21-gameplay-mechanics.md](21-gameplay-mechanics.md))
- **Wzorce organizacji moda** — struktura katalogów, konwencje nazewnicze
- **Wskazówki projektowe** — co ma sens w tej grze, a co nie

## Zasada dla przyszłych sesji

> Wiedzę **projektową i koncepcyjną** można brać z dokumentacji społeczności.
> Każdy **identyfikator** (tabela, kolumna, typ, efekt) sprawdzaj w schemacie
> lub plikach gry, zanim oznaczysz go ✅.
