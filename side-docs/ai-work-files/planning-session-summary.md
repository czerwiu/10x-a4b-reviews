# Podsumowanie biznesowe (MVP: a4b-reviews)

## Główny problem
W systemie e-commerce SaaS brakuje natywnego, prostego w użyciu i skalowalnego mechanizmu do zbierania, przechowywania i przeglądu opinii klientów o produktach zakupionych w sklepach internetowych działających na platformie. 

## Główne założenia

* System wspiera **wiele sklepów (multitenancy)**.
* Interfejsem systemu w zakresie dodawania, pobierania i moderacji opinii jest **REST API** (brak UI dla sklepów) 
* W zakresie informacji zbiorczych i statystyk dostępny jest centralny dashboard dla administratora systemu (UI).
* **Jedna opinia na produkt na użytkownika** – duplikaty nie są akceptowane.
* **Moderacja konfigurowalna per sklep**:

    * tryb bez moderacji (opinie od razu widoczne),
    * tryb z moderacją (oczekujące → zatwierdzone/odrzucone) z możliwością późniejszego przywrócenia odrzuconej opinii do zatwierdzonej.
* **Usuwanie opinii** nie powoduje trwałego zniknięcia z historii biznesowej (miękkie usunięcie); opinie usunięte **nie wpływają** na statystyki.
* **Język opinii** może być przechowywany informacyjnie; w MVP **nie wpływa** na filtrowanie ani prezentację.

## Wymagania funkcjonalne (biznes)

* **Dodawanie opinii** do produktów/ew. wariantów:

    * ocena w skali **1–5** (wyłącznie wartości całkowite),
    * treść do **3000 znaków**, bez treści HTML/URL,
    * autor **opcjonalny** (gdy pusty: „Anonimowy”),
    * użytkownik wystawiający opinię **wymagany**.
* **Moderacja opinii** zgodnie z preferencjami sklepu: automatyczna publikacja lub przepływ „oczekująca → zatwierdzona/odrzucona” z możliwością zmiany „odrzucona → zatwierdzona”.
* **Usuwanie opinii** (na potrzeby zgodności/wniosków użytkownika) – nie wpływa na możliwość ponownego wystawienia opinii w przyszłości.
* **Przegląd opinii per produkt/wariant** (bez stronicowania):

    * filtrowanie po ocenie,
    * sortowanie po dacie lub ocenie (rosnąco/malejąco),
    * **mapa ocen 1..5** z pełnym zakresem (wartości 0, jeśli brak danego poziomu).
* **Agregaty jakości**:

    * **średnia ocena** i **liczebności** liczone wyłącznie z **zatwierdzonych** opinii.
* **Dashboard centralny (widok globalny)**:

    * tabela: **sklep | liczba ALL | PENDING | APPROVED | REJECTED | średnia ocen**,
    * konfigurowalne **okno czasowe** (domyślnie 7 lub 30 dni),
    * **sortowanie** po liczbie opinii i/lub średniej ocenie,
    * **Top 10** sklepów pod względem liczby opinii (całościowo),
    * **Ostatnie opinie** ze wszystkich sklepów – lista od najnowszych; przeglądane porcjami po **50**.

## Kluczowe historie użytkownika

* **Administrator platformy (biznes/dataset owner)**:

    1. Przegląda zbiorczą tabelę sklepów z liczbami opinii w statusach i średnią ocen (okno 7/30 dni), sortuje wg priorytetu analiz.
    2. Ogląda **Top 10** sklepów (wolumen opinii) w ujęciu całościowym.
    3. Przegląda **najnowsze opinie** globalnie (porcjami po 50).
* **Sklep (właściciel/manager oferty)**:

    1. Wystawia/pozyskuje opinie od klientów po zakupie.
    2. (Gdy aktywna moderacja) Zatwierdza lub odrzuca nowe opinie; w razie potrzeby **przywraca** odrzuconą do zatwierdzonej.
    3. Usuwa opinię na żądanie (np. zgodność/RODO).
    4. Analizuje oceny i treści opinii **per produkt/wariant** (filtry, sort).

## Kryteria sukcesu (biznes) i pomiar

* **Kompletność funkcji**: dodawanie, moderacja (wg ustawień), usuwanie, przegląd i agregaty zgodnie z powyższymi zasadami.
* **Jakość danych**: brak duplikatów (jedna opinia na produkt na użytkownika); brak wpływu usuniętych opinii na statystyki.
* **Użyteczność dashboardu**:

    * czytelna, aktualna tabela sklepów z możliwością sortowania,
    * dostępne okna czasowe 7/30 dni,
    * poprawne **Top 10**,
    * wygodny podgląd ostatnich opinii (od najnowszych, po 50).
* **Adopcja**: uruchomienie na co najmniej kilku sklepach pilotażowych; regularne wykorzystanie dashboardu do przeglądu jakości i wolumenów opinii.

## Zakres MVP vs. później

* **W MVP**: wszystkie funkcje opisane powyżej; brak importu zewnętrznych opinii; brak multimediów; brak analizy AI/sentymentu; brak eksportów i drill-downów; brak cache’owania.
* **Po MVP (potencjalnie)**: rozszerzenia filtrów (np. po języku), drill-downy, eksporty, multimedia, integracje zewnętrzne, analityka AI.

## Kwestie do dalszej obserwacji (biznes)

1. **Brak stronicowania** w widokach per produkt może utrudnić pracę przy bardzo dużych wolumenach opinii – do weryfikacji po pierwszych wdrożeniach.
2. **Zakres globalnego dashboardu** jest szeroki; ewentualna rozbudowa o dodatkowe przekroje (produkty/kategorie) pozostaje poza MVP, ale warto zebrać feedback od użytkowników biznesowych.
