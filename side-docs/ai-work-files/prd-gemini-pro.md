# Dokument wymagań produktu (PRD) - a4b-reviews

## 1. Przegląd produktu
Aplikacja "a4b-reviews" to wyspecjalizowany mikroserwis przeznaczony do zarządzania opiniami o produktach w ramach wielotenantowej platformy e-commerce. Jego głównym zadaniem jest zbieranie, przechowywanie i moderowanie ocen punktowych oraz opinii tekstowych od klientów. System został zaprojektowany z myślą o obsłudze wielu sklepów (tenantów) w modelu SaaS, zapewniając izolację danych i konfiguracji.

Kluczowe funkcjonalności obejmują:
- REST API do operacji CRUD (Create, Read, Update, Delete) na opiniach.
- Wielopoziomowy system moderacji opinii (automatyczny, manualny, wspomagany przez AI).
- Identyfikacja tenantów za pomocą dedykowanego nagłówka HTTP.
- Uwierzytelnianie dostępu do API za pomocą statycznego klucza.
- Dedykowany, analityczny dashboard dla administratora systemu do monitorowania ogólnej kondycji usługi.

## 2. Problem użytkownika
Platformy e-commerce działające w modelu SaaS potrzebują scentralizowanego, ale jednocześnie elastycznego rozwiązania do obsługi opinii o produktach, które jest skalowalne i łatwe w integracji. Brak takiego systemu prowadzi do niespójnych danych, problemów z moderacją treści i utrudnia analizę satysfakcji klientów na poziomie całej platformy.

Główne problemy do rozwiązania:
- Klienci sklepów nie mają ustandaryzowanego sposobu na dzielenie się opiniami o zakupionych produktach.
- Administratorzy sklepów potrzebują narzędzia do efektywnej moderacji opinii, aby zapewnić ich jakość i zgodność z regulaminem, co jest trudne bez dedykowanego systemu.
- Administratorzy platformy e-commerce nie mają wglądu w zagregowane dane dotyczące opinii, co uniemożliwia monitorowanie wykorzystania usługi i identyfikację trendów.

## 3. Wymagania funkcjonalne

### 3.1. Zarządzanie Opiniami (REST API)
- `POST /reviews`: Umożliwia dodanie nowej opinii (ocena 1-5, tekst) z powiązaniem do produktu, opcjonalnie wariantu, użytkownika i zamówienia.
- `GET /products/{id}/reviews`: Umożliwia pobranie listy zatwierdzonych opinii dla danego produktu.
- `GET /variants/{id}/reviews`: Umożliwia pobranie listy zatwierdzonych opinii dla danego wariantu produktu.
- `GET /products/{id}/reviews/summary`: Umożliwia pobranie zagregowanych danych dla produktu (średnia ocena, całkowita liczba opinii, rozkład ocen).
- `GET /variants/{id}/reviews/summary`: Umożliwia pobranie zagregowanych danych dla wariantu produktu.
- `GET /reviews/queue`: Umożliwia pobranie listy opinii oczekujących na moderację (status `PENDING` lub `VERIFICATION`) z paginacją kursorową.
- `PATCH /reviews/{id}/status`: Umożliwia moderację opinii poprzez zmianę jej statusu.
- `DELETE /reviews/{id}`: Umożliwia miękkie usunięcie opinii (ustawienie flagi `deletedAt`).

### 3.2. Logika Moderacji
System musi wspierać trzy tryby moderacji konfigurowane na poziomie tenanta:
- `ALLOW_ALL`: Wszystkie nowe opinie są automatycznie zatwierdzane (status `APPROVED`).
- `MODERATION_MANUAL`: Wszystkie nowe opinie trafiają do kolejki moderacyjnej ze statusem `PENDING`.
- `MODERATION_AI`: Opinie są analizowane przez zewnętrzne API (OpenAI). Treści bezpieczne otrzymują status `APPROVED`, a podejrzane trafiają do kolejki moderacyjnej ze statusem `VERIFICATION`. W przypadku awarii API, opinie domyślnie trafiają do `VERIFICATION`.

### 3.3. Cykl Życia Opinii (Statusy)
- `PENDING`: Oczekuje na ręczną moderację.
- `VERIFICATION`: Oczekuje na ręczną weryfikację po analizie AI.
- `APPROVED`: Zatwierdzona i widoczna publicznie.
- `REJECTED`: Odrzucona i niewidoczna.

### 3.4. Uwierzytelnianie i Autoryzacja
- Dostęp do API jest chroniony przez mechanizm Basic Auth z jednym, statycznym kluczem (`secret`) na środowisko.
- Identyfikacja tenanta odbywa się na podstawie wartości przekazanej w nagłówku HTTP `X-Account`.
- Autoryzacja do wykonania operacji (np. prawo klienta do dodania opinii) leży po stronie systemu zlecającego (centralnego systemu e-commerce).

### 3.5. Dashboard Administratora Systemu
- Dostępny tylko do odczytu interfejs webowy dla administratora systemu.
- Prezentuje kluczowe statystyki: ogólna liczba tenantów, tabelaryczne podsumowanie opinii per tenant, lista 50 ostatnich opinii w systemie (z paginacją), TOP 10 tenantów wg. liczby opinii.
- Zawiera widget analityczny skuteczności AI, pokazujący stosunek opinii zatwierdzonych automatycznie do tych, które trafiły do ręcznej weryfikacji, oraz wynik tej weryfikacji.

## 4. Granice produktu

### 4.1. Funkcjonalności w zakresie MVP
- API do zarządzania opiniami.
- Mechanizmy moderacji (automatyczna, manualna, AI).
- Uwierzytelnianie Basic Auth i obsługa multitenant.
- Miękkie usuwanie opinii.
- Dashboard analityczny dla administratora systemu.

### 4.2. Funkcjonalności poza zakresem MVP
- Import opinii z zewnętrznych źródeł.
- Interfejs webowy do zarządzania opiniami dla administratorów sklepów.
- Dodawanie zdjęć i filmów do opinii.
- Integracja z zewnętrznymi systemami opinii (np. Trusted Shops, Google Reviews).
- Szczegółowe, mierzalne wskaźniki sukcesu (KPIs) oraz formalna analiza ryzyka.
- Funkcje drążenia danych (drill-down) i eksportu na dashboardzie.

## 5. Historyjki użytkowników

### US-001
- Tytuł: Dodawanie opinii do produktu
- Opis: Jako klient sklepu, chcę dodać ocenę punktową (1-5) i opinię tekstową do produktu, który zakupiłem, aby podzielić się moimi doświadczeniami.
- Kryteria akceptacji:
  - System musi przyjąć żądanie `POST /reviews` z poprawnymi danymi (rating, content, productId, userId, orderId).
  - Ocena musi być liczbą całkowitą w zakresie 1-5.
  - Treść opinii nie może być pusta.
  - Po pomyślnym dodaniu opinii, system zwraca kod odpowiedzi 201 Created.
  - Status nowej opinii jest ustawiany zgodnie z trybem moderacji skonfigurowanym dla danego tenanta.

### US-002
- Tytuł: Wyświetlanie opinii na stronie produktu
- Opis: Jako system e-commerce, chcę pobrać wszystkie zatwierdzone opinie dla danego produktu, aby wyświetlić je na karcie produktu.
- Kryteria akceptacji:
  - System musi obsłużyć żądanie `GET /products/{id}/reviews`.
  - Odpowiedź musi zawierać wyłącznie opinie ze statusem `APPROVED`.
  - Odpowiedź musi zawierać listę opinii posortowaną domyślnie od najnowszej.
  - System musi umożliwiać sortowanie po dacie (`date_asc`, `date_desc`) i ocenie (`rating_asc`, `rating_desc`).
  - System musi umożliwiać filtrowanie wyników po ocenie (np. `?rating=5`).

### US-003
- Tytuł: Wyświetlanie podsumowania opinii dla produktu
- Opis: Jako system e-commerce, chcę pobrać zagregowane dane o opiniach dla produktu, aby wyświetlić gwiazdki i ogólną liczbę ocen.
- Kryteria akceptacji:
  - System musi obsłużyć żądanie `GET /products/{id}/reviews/summary`.
  - Odpowiedź musi zawierać: średnią ocenę, całkowitą liczbę opinii oraz mapę z liczbą poszczególnych ocen (np. `{"5": 10, "4": 5, ...}`).
  - Obliczenia muszą uwzględniać wyłącznie opinie ze statusem `APPROVED`.

### US-004
- Tytuł: Dostęp do kolejki moderacyjnej
- Opis: Jako administrator sklepu, chcę mieć dostęp do listy opinii oczekujących na moją decyzję, aby móc je sprawnie moderować.
- Kryteria akceptacji:
  - System musi obsłużyć żądanie `GET /reviews/queue`.
  - Odpowiedź musi zawierać opinie ze statusem `PENDING` lub `VERIFICATION`.
  - Wyniki muszą być paginowane z użyciem mechanizmu kursora.
  - Opinie w odpowiedzi muszą należeć wyłącznie do tenanta, w imieniu którego wykonano żądanie.

### US-005
- Tytuł: Moderowanie opinii
- Opis: Jako administrator sklepu, chcę moderować (zatwierdzać/odrzucać) opinie, które wymagają mojej uwagi, aby zapewnić ich jakość i zgodność z regulaminem.
- Kryteria akceptacji:
  - System musi obsłużyć żądanie `PATCH /reviews/{id}/status` z nowym statusem (`APPROVED` lub `REJECTED`).
  - Zmiana statusu musi być możliwa tylko dla opinii ze statusem `PENDING` lub `VERIFICATION`.
  - Po pomyślnej zmianie statusu, system zwraca kod 200 OK.
  - Próba zmiany statusu opinii nienależącej do tenanta musi zwrócić błąd 404 Not Found.

### US-006
- Tytuł: Automatyczna analiza opinii przez AI
- Opis: Jako system, chcę automatycznie analizować nowe opinie pod kątem niedozwolonych treści, aby odciążyć moderatorów i przyspieszyć publikację bezpiecznych opinii.
- Kryteria akceptacji:
  - Po dodaniu nowej opinii w sklepie z trybem `MODERATION_AI`, system musi wysłać jej treść do analizy przez API OpenAI.
  - Jeśli analiza wskaże treść jako bezpieczną, opinia automatycznie otrzymuje status `APPROVED`.
  - Jeśli analiza wskaże treść jako podejrzaną, opinia otrzymuje status `VERIFICATION` i trafia do kolejki moderacyjnej.
  - W przypadku błędu komunikacji z API OpenAI, opinia domyślnie otrzymuje status `VERIFICATION`.
  - Wynik analizy (score, reason) jest zapisywany w dedykowanych polach opinii.

### US-007
- Tytuł: Monitorowanie kondycji usługi
- Opis: Jako administrator systemu, chcę widzieć zagregowane statystyki dotyczące liczby opinii we wszystkich sklepach, aby monitorować ogólną kondycję i wykorzystanie usługi.
- Kryteria akceptacji:
  - Dashboard musi wyświetlać aktualną liczbę tenantów w systemie.
  - Dashboard musi prezentować listę 50 ostatnich opinii dodanych w całym systemie, z możliwością paginacji.
  - Dashboard musi wyświetlać ranking TOP 10 tenantów z największą liczbą opinii.
  - Wszystkie dane na dashboardzie są dostępne tylko do odczytu.

### US-008
- Tytuł: Bezpieczny dostęp do API
- Opis: Jako administrator systemu, chcę mieć pewność, że dostęp do API jest chroniony i możliwy tylko dla autoryzowanych klientów.
- Kryteria akceptacji:
  - Każde żądanie do API musi zawierać poprawny nagłówek `Authorization` z danymi Basic Auth.
  - Żądania bez lub z niepoprawnym uwierzytelnieniem muszą być odrzucane z kodem 401 Unauthorized.
  - Każde żądanie musi zawierać nagłówek `X-Account` identyfikujący tenanta.
  - Żądania bez nagłówka `X-Account` muszą być odrzucane z kodem 400 Bad Request.

### US-009
- Tytuł: Obsługa opinii dla wariantów produktów
- Opis: Jako system e-commerce, chcę pobrać opinie przypisane do konkretnego wariantu produktu.
- Kryteria akceptacji:
  - System musi obsłużyć żądanie `GET /variants/{id}/reviews`.
  - Odpowiedź musi zawierać listę opinii ze statusem `APPROVED`, które mają podany `variantId`.
  - System musi również obsłużyć żądanie `GET /variants/{id}/reviews/summary` i zwrócić poprawne, zagregowane dane dla wariantu.

## 6. Metryki sukcesu

### 6.1. Metryki Funkcjonalne
- System poprawnie realizuje wszystkie operacje zdefiniowane w API.
- Moderacja AI prawidłowo klasyfikuje opinie, co jest weryfikowalne na dashboardzie administratora.
- Dashboard administratora prezentuje poprawne, zagregowane dane w czasie zbliżonym do rzeczywistego.

### 6.2. Metryki Niefunkcjonalne
- Czas odpowiedzi dla kluczowych endpointów (`GET /products/{id}/reviews`, `POST /reviews`) nie przekracza 200ms pod standardowym obciążeniem.
- System jest zgodny z RODO poprzez mechanizm "soft delete" oraz nieprzechowywanie danych osobowych poza `userId`.
- Dostęp do danych jest skutecznie izolowany pomiędzy tenantami.

### 6.3. Metryki Biznesowe
- Wysoki wskaźnik automatycznego zatwierdzania opinii w trybie `MODERATION_AI` (powyżej 80%).
- Niski odsetek fałszywie negatywnych (opinie bezpieczne wysłane do weryfikacji) i fałszywie pozytywnych (opinie niebezpieczne zatwierdzone automatycznie) w moderacji AI. Skuteczność AI jest mierzona i widoczna na dedykowanym widgecie na dashboardzie.
- Stabilne i rosnące wykorzystanie API przez systemy e-commerce, monitorowane przez liczbę dodawanych opinii.

