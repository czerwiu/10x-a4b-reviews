<conversation_summary>
<decisions>
1.  **Uwierzytelnianie i Autoryzacja**: Uwierzytelnianie przez Basic Auth. Dostęp do API będzie chroniony przez jeden `secret` na środowisko, przechowywany w usłudze typu vault. Aplikacja będzie działać w sieci wewnętrznej. Autoryzacja operacji (np. prawo do dodania opinii) leży po stronie centralnego systemu e-commerce.
2.  **Identyfikacja Tenantów (Sklepów)**: Tenanci będą identyfikowani przez unikalny klucz tekstowy (np. "tomex") przekazywany w nagłówku HTTP `X-Account`. Zarządzanie tenantami odbywa się w systemie centralnym.
3.  **Model Danych Opinii**: Opinia, oprócz pól związanych z oceną i treścią opinii będzie zawierać pola: `id użytkownika` (obowiązkowe), `autor` (tekst, opcjonalnie), `id zamówienia`, `id produktu`, `id wariantu` (opcjonalnie), `status`, `data utworzenia`, `metadata` (JSONB, na przyszłe zastosowania), `media` (JSONB, na przyszłe zastosania) oraz `language` (ISO 639-1, na początek nie będzie wykorzystywane), `classification_score` (liczbowy, opcjonalnie - wynik analizy AI) oraz `classification_reason` (tekst, opcjonalnie - powód klasyfikacji AI).
4.  **Moderacja Opinii**: System będzie wspierał trzy tryby na poziomie tenanta: 
* ALLOW_ALL - automatyczne zatwierdzanie (`APPROVED`) 
* MODERATION_AI - moderacja wspomagana przez AI (treści bezpieczne od razu `APPROVED`, podejrzane `VERIFICATION` do ręcznej weryfikacji) 
* MODERATION_MANUAL - moderacja ręczna (`PENDING` -> `APPROVED` / `REJECTED`).
5. Usuwanie opinii będzie realizowane jako "soft delete" (ustawienie pola `deletedAt`).
6.  **Dashboard Administratora Systemu**: Dashboard w MVP będzie zawierał: liczbę tenantów, tabelaryczne podsumowanie opinii per tenant, listę 50 ostatnich opinii z całego systemu (z paginacją kursorem), TOP 10 tenantów wg liczby opinii oraz widget skuteczności moderacji AI (procent opinii auto-zatwierdzonych vs. wysłanych do weryfikacji; w ramach zweryfikowanych - procent zaakceptowanych vs. odrzuconych). Bez funkcji drążenia danych (drill-down) i eksportu.
7.  **Obsługa Wariantów Produktów**: Opinia dodana do wariantu będzie miała zapisane zarówno `id wariantu`, jak i `id produktu`. API umożliwi pobieranie opinii dla konkretnego wariantu.
8.  **Paginacja i Filtrowanie**: Paginacja nie będzie implementowana dla endpointów pobierających opinie dla produktu/wariantu (zwracana będzie pełna lista). Dostępne będzie sortowanie i filtrowanie. Jedynym wyjątkiem z paginacją (kursorem) jest lista wszystkich opinii na dashboardzie administratora.
9.  **Pominięte w MVP**: Szczegółowe, mierzalne wskaźniki sukcesu (KPIs) oraz formalna analiza ryzyka zostają pominięte w zakresie MVP.
</decisions>

<matched_recommendations>
1.  **Zdefiniowanie szczegółowych mechanizmów bezpieczeństwa**: Decyzja o użyciu `secret` i powiązaniu opinii z zamówieniem jest zgodna z tą rekomendacją, upraszczając ją do realiów systemu wewnętrznego.
2.  **Zdefiniowanie pełnego modelu danych dla opinii**: Użytkownik precyzyjnie określił pola i ich właściwości, co realizuje tę rekomendację.
3.  **Stosowanie "miękkiego usuwania" (soft delete)**: Decyzja o użyciu pola `deletedAt` jest bezpośrednim wdrożeniem tej rekomendacji, co ułatwi audyt i ewentualne przywracanie danych.
4.  **Zdefiniowanie logiki obsługi wariantów**: Potwierdzono, że opinie będą mogły być powiązane z wariantami, a API pozwoli na ich odpytywanie, co daje elastyczność w prezentacji danych.
5.  **Implementacja mechanizmu paginacji**: Rekomendacja została częściowo wdrożona – paginacja kursorem pojawi się w kluczowym dla wydajności miejscu (dashboard administratora), co jest dobrym kompromisem dla MVP.
6.  **Ustalenie logiki moderacji i stanów opinii**: Zdefiniowano tryby moderacji per tenant oraz cykl życia opinii (`PENDING`, `VERIFICATION`, `APPROVED`, `REJECTED`), co wprowadza porządek w procesie.
7.  **Wykorzystanie usług zewnętrznych i plan awaryjny**: Zdecydowano o użyciu zewnętrznego API (OpenAI) do analizy treści i zdefiniowano zachowanie systemu w przypadku jego awarii (fail-safe do ręcznej weryfikacji), co jest dobrą praktyką projektową.
</matched_recommendations>

<prd_planning_summary>
### Podsumowanie Planowania PRD dla Aplikacji "a4b-reviews" (MVP)

#### 1. Główne Wymagania Funkcjonalne

Aplikacja będzie mikroserwisem do zarządzania opiniami o produktach dla wielotenantowej platformy e-commerce.

*   **API do zarządzania opiniami**:
    *   `POST /reviews`: Dodawanie nowej opinii (ocena 1-5, tekst) z powiązaniem do produktu, wariantu, użytkownika i zamówienia.
    *   `GET /products/{id}/reviews`: Pobieranie listy opinii dla produktu. Umożliwia filtrowanie po ocenie (np. `?rating=5`) oraz sortowanie po dacie lub ocenie (np. `?sort=date_asc` lub `?sort=rating_desc`).
    *   `GET /variants/{id}/reviews`: Pobieranie listy opinii dla wariantu, z takimi samymi opcjami filtrowania i sortowania jak dla produktu.
    *   `GET /products/{id}/reviews/summary`: Pobieranie zagregowanych danych o opiniach dla produktu: średnia ocena, całkowita liczba opinii oraz mapa z liczbą poszczególnych ocen (np. `{"5": 10, "4": 5, ...}`).
    *   `GET /variants/{id}/reviews/summary`: Pobieranie zagregowanych danych o opiniach dla wariantu (średnia ocena, liczba opinii, mapa ocen).
    *   `GET /reviews/queue`: Pobieranie listy opinii oczekujących na moderację (status `PENDING` lub `VERIFICATION`), z paginacją.
    *   `PATCH /reviews/{id}/status`: Moderacja opinii (zmiana statusu na `APPROVED`/`REJECTED`/`VERIFICATION`).
*   **Logika Moderacji**: System wspiera trzy tryby pracy per tenant:
    *   `ALLOW_ALL`: Wszystkie opinie są automatycznie zatwierdzane (`APPROVED`).
    *   `MODERATION_MANUAL`: Wszystkie opinie trafiają do ręcznej moderacji ze statusem `PENDING`.
    *   `MODERATION_AI`: Opinie są analizowane przez OpenAI API pod kątem niedozwolonych treści (wulgaryzmy, mowa nienawiści, dane osobowe, treści erotyczne). Bezpieczne opinie otrzymują status `APPROVED`, a podejrzane `VERIFICATION`. W przypadku awarii API, opinie domyślnie trafiają do `VERIFICATION`.
*   **Uwierzytelnianie**: Basic Auth ze statycznym kluczem (`secret`) per środowisko. Identyfikacja tenanta na podstawie nagłówka `X-Account`.
*   **Dashboard Administratora Systemu**: Interfejs webowy (read-only) dla administratora systemu, prezentujący kluczowe statystyki wykorzystania systemu, w tym widget analityczny skuteczności AI, pokazujący stosunek opinii zatwierdzonych automatycznie do tych, które trafiły do ręcznej weryfikacji, oraz wynik tej weryfikacji (procent zaakceptowanych i odrzuconych).

#### 2. Kluczowe Historie Użytkownika

*   **Klient sklepu**: "Jako klient sklepu, chcę dodać ocenę punktową i opinię tekstową do produktu, który zakupiłem, aby podzielić się moimi doświadczeniami."
*   **Administrator sklepu**: "Jako administrator sklepu, chcę mieć dostęp do listy opinii oczekujących na moją decyzję, aby móc je sprawnie moderować."
*   **Administrator sklepu**: "Jako administrator sklepu, chcę moderować (zatwierdzać/odrzucać) opinie, które wymagają mojej uwagi, aby zapewnić ich jakość i zgodność z regulaminem."
*   **System (AI)**: "Jako system, chcę automatycznie analizować nowe opinie pod kątem niedozwolonych treści, aby odciążyć moderatorów i przyspieszyć publikację bezpiecznych opinii."
*   **System e-commerce (użytkownik API)**: "Jako system e-commerce, chcę pobrać wszystkie zatwierdzone opinie dla danego produktu, aby wyświetlić je na karcie produktu."
*   **Administrator systemu**: "Jako administrator systemu, chcę widzieć zagregowane statystyki dotyczące liczby opinii we wszystkich sklepach, aby monitorować ogólną kondycję i wykorzystanie usługi."

#### 3. Kryteria Sukcesu

*   **Funkcjonalne**: System umożliwia dodawanie, pobieranie i moderowanie opinii zgodnie ze zdefiniowanymi ścieżkami przez REST API. Moderacja AI poprawnie klasyfikuje opinie. Dashboard administratora prezentuje poprawne, zagregowane dane.
*   **Niefunkcjonalne**: System jest bezpieczny (dostęp chroniony), skalowalny (obsługa wielu tenantów) i wydajny (szybkie odpowiedzi API, zwłaszcza dla kluczowych endpointów). Zgodność z RODO poprzez mechanizm "soft delete" i anonimizację autora.
*   **Biznesowe**: Niska liczba opinii wymagających ręcznej moderacji w sklepach korzystających z trybu `MODERATION_AI`. Skuteczność AI (mierzona widgetem na dashboardzie) pokazuje, że większość opinii kierowanych do weryfikacji jest słusznie oflagowana.

</prd_planning_summary>

<unresolved_issues>
Poniższe kwestie zostały uznane za zbyt niskopoziomowe na tym etapie, jednak będą wymagały doprecyzowania przed rozpoczęciem implementacji:

1.  **Strategia obsługi błędów API**: Konkretne kody odpowiedzi HTTP i formaty komunikatów o błędach (np. dla braku tenanta, błędnej walidacji).
2.  **Szczegółowa walidacja pól**: Ograniczenia dla danych wejściowych (np. maksymalna długość tekstu opinii, dozwolone formaty ID).
3.  **Ochrona przed spamem**: Potencjalna potrzeba wprowadzenia ograniczenia typu "jeden użytkownik - jedna opinia na produkt".
4.  **Wydajność dashboardu**: Kwestia, czy statystyki na dashboardzie administratora mają być liczone w czasie rzeczywistym, czy z wykorzystaniem cache/okresowego odświeżania.
5.  **Uwierzytelnianie do dashboardu**: Sposób zabezpieczenia dostępu do interfejsu administratora systemu (inny niż `secret` dla API).
6.  **Logowanie audytowe**: Potrzeba i zakres logowania zmian w systemie (np. kto i kiedy zmienił status opinii).
7.  **Szczegóły implementacji AI**:
    *   Dokładna treść i struktura promptu wysyłanego do OpenAI API.
    *   Sposób przechowywania wyników analizy AI: Zdecydowano o użyciu dedykowanych kolumn `classification_score` i `classification_reason`. Należy zdefiniować dokładne typy danych i ewentualne wartości.
    *   Strategia zarządzania kosztami i limitami użycia OpenAI API.
8.  **Szczegóły filtrowania i sortowania**: Dokładna lista pól, po których będzie można filtrować i sortować opinie na liście produktowej, została wstępnie zdefiniowana (filtrowanie po ocenie, sortowanie po dacie/ocenie).
</unresolved_issues>

</conversation_summary>
