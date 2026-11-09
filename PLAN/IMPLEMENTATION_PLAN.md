# AI MASTER PRO — Plan wdrożenia dostępu do plików, GitHub i Android Studio

**Status:** plan techniczny — bez zmian w kodzie produkcyjnym  
**Repozytorium:** AI_MASTER_PRO_CLEAN  
**Gałąź:** plan/agent-file-github-bridge  
**Data:** 2026-11-09

## Cel

Doprowadzić AI MASTER PRO do stanu, w którym MASTER może bezpiecznie:
1. czytać strukturę i zawartość plików projektu;
2. zlecać agentom zadania na rzeczywistych plikach, z kontrolą zakresu i śladem audytowym;
3. odczytywać repozytorium GitHub, gałęzie, commity, pliki, PR i statusy CI;
4. proponować i wykonywać zmiany wyłącznie w kontrolowanej gałęzi roboczej;
5. współpracować z agentem działającym w Android Studio przez Work Bridge, otrzymywać wyniki, logi i wyniki testów;
6. przedstawiać użytkownikowi różnicę kodu i dowody testów przed zatwierdzeniem/łączeniem zmian.

## Zasady niepodlegające negocjacji

- Najpierw ustalić i zachować rzeczywistą bazę projektu. Nie nadpisywać ani nie usuwać poprzednich paczek.
- Nie zmieniać wersji Gradle, AGP, Kotlin ani JDK bez wykazanego powodu.
- Nie oznaczać buildów/testów jako zaliczonych bez logu i identyfikatora commita.
- Domyślnie audyt i odczyt są tylko do odczytu. Zapis wymaga jawnego zakresu zadania.
- MASTER nie scala automatycznie zmian do main. Zmiany trafiają na osobną gałąź i do przeglądu.
- Klucze API, tokeny, local.properties, keystore, hasła i dane prywatne nie trafiają do promptów, raportów ani repozytorium.
- Każde zadanie ma ID, bazowy commit, listę dozwolonych plików, status, wynik i zapis audytowy.
- MASTER ma rozróżniać: brak dostępu, brak pliku, błąd narzędzia, timeout i rzeczywisty wynik negatywny.

## Docelowa architektura

`MASTER` — planowanie, delegowanie, kontrola budżetu, weryfikacja i odpowiedź końcowa.

`Agent Tool API` — ograniczone narzędzia: listowanie katalogów, odczyt pliku, wyszukiwanie symboli/tekstu, metadane pliku, zapis poprawki/diff. Każde wywołanie ma walidację ścieżki i uprawnień.

`Workspace Access Layer` — wspólna abstrakcja dostępu do źródeł:
- lokalny workspace aplikacji (pliki jawnie udostępnione);
- repozytorium GitHub przez osobny adapter;
- lokalny projekt Android Studio przez Work Bridge.

`Work Bridge` — lokalny, uwierzytelniony proces/agent po stronie komputera. Udostępnia wyłącznie zatwierdzony katalog projektu, wykonuje dozwolone polecenia Gradle/Git i odsyła logi oraz artefakty. Sama aplikacja Android nie uzyskuje automatycznie dostępu do dysku komputera.

`Task Queue + Event Log` — zadania, heartbeat, checkpointy, wyniki, błędy i ślad komunikacji agentów. Stan musi przetrwać restart aplikacji.

`GitHub Adapter` — odczyt repozytorium, gałęzi, commitów, plików, PR i CI; zapis do osobnej gałęzi; tworzenie PR. Token przechowywany bezpiecznie, z minimalnym zakresem uprawnień.

## Etapy wdrożenia i kryteria odbioru

### Etap 0 — odzyskanie i potwierdzenie bazy
- Pozyskać dokładny aktualny projekt z komputera, a nie zakładać, że stary ZIP odpowiada lokalnemu stanowi.
- Zapisać: wersję aplikacji, branch/commit, status Git, Gradle Wrapper, AGP, Kotlin, JDK, SDK i strukturę katalogów.
- Uruchomić testy jednostkowe i `:app:assembleDebug` w tym samym środowisku, zachować pełne logi i SHA-256 APK.
- Zarchiwizować bazę bez modyfikacji.
**Odbiór:** odtwarzalna baza źródłowa oraz build/APK potwierdzone logiem. Jeśli build nie przechodzi, najpierw naprawić wyłącznie blokadę bazową.

### Etap 1 — prawdziwy dostęp agentów do plików
- Dodać File Access API: `list`, `read`, `search`, `stat`.
- Ograniczyć dostęp do zatwierdzonego katalogu projektu; blokować traversal, ścieżki absolutne poza workspace, sekrety i niebezpieczne pliki.
- Dodać limity rozmiaru, paginację, kodowanie, timeouty i jawne komunikaty o błędach.
- Wstrzykiwać agentom wyniki narzędzi, a nie ogólny opis architektury udający dostęp do kodu.
**Odbiór:** agent potrafi wskazać plik, odczytać jego rzeczywistą treść i podać ścieżkę/zakres; próby wyjścia poza workspace są blokowane i logowane.

### Etap 2 — GitHub dla MASTER
- Dodać połączenie GitHub i wybór repozytorium/gałęzi.
- Obsłużyć odczyt drzewa, plików, commitów, diffów, PR i statusów CI.
- Zapisywać tylko na dedykowanej gałęzi zadania; tworzyć PR z opisem, zakresem i wynikami testów.
- Token przechowywać w Android Keystore; nie logować go ani nie umieszczać w kontekście modeli.
**Odbiór:** MASTER odczytuje wskazany plik i historię repo, potrafi utworzyć zmianę w gałęzi roboczej, a main pozostaje nietknięty.

### Etap 3 — Work Bridge do Android Studio
- Uruchomić lokalnego Bridge na komputerze użytkownika, związany z jednym zatwierdzonym katalogiem projektu.
- Uwierzytelnienie, parowanie urządzeń, rotacja/revokacja tokenu, ograniczenia sieciowe i allowlista poleceń.
- Operacje: status Git, odczyt plików, wyszukiwanie, diff, testy, build, odczyt logów i pobranie APK.
- Kolejka zadań i wyniki odporne na restart; agent Android Studio/Bridge raportuje do MASTER bez ręcznego kopiowania.
**Odbiór:** MASTER zleca zadanie, Bridge wykonuje je w lokalnym projekcie, a MASTER odbiera log, commit, diff i artefakty.

### Etap 4 — bezpieczny cykl zmian i testów
- Plan → analiza plików → mały diff → peer review → testy → build → raport.
- Każda poprawka ograniczona do jawnego zakresu plików; wykrywanie konfliktów i zmian użytkownika.
- Brak automatycznego merge do main; wymagane zatwierdzenie użytkownika.
- Raport dowodowy: commit, lista plików, testy z kodem wyjścia, build, APK i SHA-256.
**Odbiór:** pełny cykl od zadania MASTER do sprawdzonego artefaktu, z możliwością cofnięcia zmian.

### Etap 5 — autonomia i nadzór
- MASTER dobiera agentów do rodzaju pracy, nie uruchamia zespołu do prostych rozmów.
- Wspólny kontekst z rzeczywistych plików i wyników narzędzi; deduplikacja zadań; checkpointy.
- Budżet tokenów, limity równoległości, retry/cooldown i fallback z czytelnym powodem.
- Diagnostyka dostępna MASTER: narzędzia, uprawnienia, błędy, kolejka, stan Bridge, testy i historia.
**Odbiór:** MASTER potrafi sam zaplanować i wykonać ograniczone zadanie, wykryć brak dostępu, wznowić po restarcie i przedstawić dowody.

## Kolejność priorytetów

1. Etap 0 — potwierdzona baza i build.
2. Etap 1 — czytanie plików przez agentów.
3. Etap 2 — GitHub dla MASTER.
4. Etap 3 — Work Bridge do lokalnego Android Studio.
5. Etap 4 — bezpieczny zapis, testy i PR.
6. Etap 5 — autonomia.

Etapy 1 i 2 mogą być rozwijane równolegle po zabezpieczeniu bazy, ale integracja z lokalnym Android Studio wymaga Etapu 3.

## Aktualny stan potwierdzony 2026-11-09

- Repozytorium `AI_MASTER_PRO_CLEAN` jest publiczne.
- Gałąź `main` zawiera obecnie wyłącznie `.gitattributes`; nie ma w niej projektu Android.
- Jedyny widoczny branch to `main`; commit początkowy: `1d547bfa26db99bc09ed874afa7ffd6ba22a1edf`.
- Nie ma w tym repo potwierdzonej bazy kodu do poprawienia.
- Wcześniejsze archiwa źródłowe nie stanowią potwierdzonego działającego APK. Nie wolno nazywać ich działającą bazą bez build logu.
- Połączenie GitHub dostępne dla MASTER nie oznacza dostępu do lokalnego dysku ani uruchomionego Android Studio.

## Najbliższy krok

Dostarczyć dokładny aktualny katalog projektu z Android Studio (albo jego ZIP bez `.gradle`, katalogów `build`, `local.properties`, keystore i sekretów) oraz raport `git status`, branch/commit i wynik ostatniego builda. Następnie wykonać Etap 0 i dopiero na potwierdzonej bazie rozpocząć Etap 1.
