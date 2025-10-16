# Dokument wymagań produktu (PRD) - a4b-reviews

## 1. Przegląd produktu
A4b-reviews to multitenantowy (wiele sklepów jako tenanty) system backendowy do zbierania, przechowywania, moderacji i udostępniania opinii (ocena 1–5 oraz tekst do 3000 znaków) o produktach i ich wariantach w ramach platformy e-commerce SaaS. Interfejsem operacyjnym dla sklepów (dodawanie, pobieranie, moderacja, usuwanie) jest REST API. Dla administratora platformy (centralny nadzór) dostępny jest dashboard webowy prezentujący skonsolidowane statystyki jakości i wolumenów opinii. System zapewnia spójność danych (jedna opinia na produkt na użytkownika), miękkie usuwanie (soft delete) oraz konfigurację trybu moderacji per sklep.

Cele nadrzędne: szybkość wdrożenia MVP, skalowalność (obsługa rosnących wolumenów opinii wielu sklepów), integralność danych, bezpieczeństwo i zgodność (RODO: możliwość usunięcia danych; ograniczone przechowywanie danych identyfikujących użytkownika), klarowna analityka dla administratora platformy.

Zasoby: REST API (core), moduł agregacji statystyk, warstwa autoryzacji / uwierzytelniania (tenant + role), dashboard centralny (tylko dla administratora platformy) – minimalny UI.

## 2. Problem użytkownika
Brak w platformie e-commerce wbudowanego, spójnego i skalowalnego mechanizmu opinii utrudnia:
1. Sklepom: pozyskiwanie wiarygodnych opinii po zakupie, zarządzanie jakością (moderacja), analiza ocen produktów.
2. Administratorowi platformy: globalny wgląd w jakość oferty i aktywność sklepów (które sklepy generują opinie, trendy jakościowe). 
3. Użytkownikom końcowym (kupującym): dostęp do rzetelnego zbioru ocen poprawiającego zaufanie i konwersję (w dalszym etapie – poza MVP). 

Potrzeba: centralny, zunifikowany system API opinii eliminujący duplikaty, zapewniający możliwość moderacji i szybkie zapytania o agregaty (średnia ocena, rozkład ocen) oraz globalny panel operacyjno-analityczny.

## 3. Wymagania funkcjonalne
Poniżej zestaw funkcjonalny MVP (bez implementacji niewchodzących w zakres). Numeracja FR dla śledzenia z US.

- **FR-01** Dodanie opinii do produktu lub wariantu (rating 1–5, treść ≤3000 znaków, brak HTML/URL, opcjonalny author wyświetlany; wymagana identyfikacja użytkownika wystawiającego – userId). Duplikaty (ten sam userId + productId + variantId?) zabronione przy aktywnej opinii (nieusuniętej). 
- **FR-02** Walidacje: rating wyłącznie całkowity 1..5; treść bez znaczników HTML i URL; limit długości; jeśli author pusty: prezentacja jako Anonimowy. 
- **FR-03** Przechowywanie metadanych: timestamps (createdAt, updatedAt, moderatedAt), status, tenantId (sklep), language (opcjonalne pole informacyjne), productId, variantId (opcjonalnie null), userId (identyfikator użytkownika sklepu), usunięty (softDelete flag). 
- **FR-04** Konfiguracja moderacji per sklep: tryb automatyczny (nowa opinia od razu status APPROVED) lub tryb moderowany (nowa opinia status PENDING do decyzji). 
- **FR-05** Moderacja: przejścia statusów PENDING -> APPROVED lub PENDING -> REJECTED, oraz REJECTED -> APPROVED (przywrócenie). Zapis historii zmian statusu (audit trail minimalny). 
- **FR-06** Usunięcie opinii: soft delete (flag); usunięta opinia niewidoczna w listach i nie wliczana do agregatów; możliwość późniejszego ponownego dodania nowej opinii przez użytkownika (duplikat po usunięciu starej jest dopuszczalny). 
- **FR-07** Pobieranie opinii per produkt/wariant: opcje filtrów (rating), sortowanie po dacie (createdAt) lub rating (rosnąco/malejąco). Brak stronicowania w MVP (wszystkie opinie spełniające warunki). Widoczne tylko APPROVED (w auto trybie od razu) – decyzja: nieudostępniać PENDING/REJECTED publicznie przez API sklepu. 
- **FR-08** Agregaty: średnia ocena (średnia ważona zwykła z APPROVED), liczba opinii, mapowanie countów 1..5 (brak = 0). 
- **FR-09** API do uzyskania listy opinii surowych (dla właściciela sklepu) może opcjonalnie zawierać PENDING/REJECTED (autoryzowany kontekst sklepu) – jeśli uwzględnimy to w MVP. (Założenie: tak, aby umożliwić moderację). 
- **FR-10** Dashboard centralny (admin platformy): tabela sklepów (shopId | count_ALL | count_PENDING | count_APPROVED | count_REJECTED | averageRating) dla wybranego okna (7 lub 30 dni). 
- **FR-11** Dashboard: zestaw Top 10 sklepów wg całkowitej liczby opinii (ALL) w wybranym oknie. 
- **FR-12** Dashboard: lista ostatnich opinii globalnie, sort malejąco po createdAt, porcje po 50 (paginacja stronicowana offset/limit lub cursor). 
- **FR-13** Konfiguracja okna czasowego w dashboardzie (parametr). Domyślne 30 (lub 7 – do potwierdzenia; przyjmujemy 30). 
- **FR-14** Autoryzacja i bezpieczeństwo: restrykcja dostępu do endpointów moderacji i konfiguracji do roli sklepu; dashboard tylko rola ADMIN_PLATFORM; multi-tenant isolation (tenantId w tokenie / kontekście). 
- **FR-15** Audyt zmian statusu opinii (log minimalny: reviewId, oldStatus, newStatus, changedBy, timestamp). 
- **FR-16** Odporność na duplikaty: mechanizm unikalnego indeksu (tenantId + productId + variantId? + userId) filtrujący tylko nieusunięte wpisy. 
- **FR-17** Obsługa błędów walidacyjnych z jednoznacznymi kodami (np. REVIEW_ALREADY_EXISTS, INVALID_RATING, CONTENT_TOO_LONG, ILLEGAL_CONTENT). 
- **FR-18** API agregatów per produkt/wariant (GET) bez listy treści, tylko statystyki (średnia, counts, total). 
- **FR-19** Reprezentacja języka opinii (langCode ISO) – przechowywana, niewykorzystywana w filtrach w MVP. 
- **FR-20** Przywrócenie REJECTED -> APPROVED nie resetuje createdAt; aktualizuje moderatedAt. 
- **FR-21** Soft delete nie usuwa rekordu audytu; treść może zostać zanonimizowana przy żądaniu RODO (opcjonalny stretch; w MVP: usunięcie = ukrycie, przyjmujemy że treść nie jest zwracana). 


## 4. Granice produktu
Zakres wchodzi w MVP:
1. Core REST API: dodanie, walidacja, moderacja, usuwanie (soft), pobieranie opinii, pobieranie agregatów. 
2. Konfiguracja moderacji na poziomie sklepu (switch). 
3. Dashboard centralny (admin platformy) z tabelą, Top 10, ostatnimi opiniami (paginacja 50). 
4. Mechanizmy unikania duplikatów i wykluczania usuniętych opinii z agregatów. 
5. Uwierzytelnianie / autoryzacja ról (min. ROLE_PLATFORM_ADMIN, ROLE_SHOP, ewentualnie ROLE_SHOP_MODERATOR) i izolacja tenantów. 
6. Przechowywanie języka opinii jako atrybutu informacyjnego.

Poza zakresem MVP (out-of-scope):
1. Import opinii z zewnętrznych źródeł. 
2. UI (frontend) dla sklepów do zarządzania opiniami (sklepy używają tylko API). 
3. Multimedia (zdjęcia, wideo) w opiniach. 
4. Integracje z zewnętrznymi systemami recenzji (Trusted Shops, Google Reviews). 
5. Analiza AI / sentyment, streszczenia. 
6. Zaawansowane filtry (język), drill-down po kategoriach, eksporty, caching dedykowany. 
7. Twarde usunięcie (hard delete) – brak w MVP; pełna anonimizacja może być później. 
8. Stronicowanie list opinii per produkt (celowo pominięte w MVP). 
9. Zaawansowane raportowanie poza opisany dashboard. 

Założenia i ograniczenia techniczne (MVP):
- Brak wymogu real-time push; dane dashboardu mogą być obliczane on-demand. 
- Brak mechanizmów rate limiting (opcjonalne później). 
- Minimalne logowanie audytowe tylko dla zmian statusu i usuwania. 

## 5. Historyjki użytkowników
Format: ID, Tytuł, Opis, Kryteria akceptacji. Role: PLATFORM_ADMIN, SHOP_CUSTOMER (klient sklepu - kupujący), SHOP_MODERATOR (właściciel/manager sklepu), SHOP_VIEWER (klient sklepu przeglądający opinie – często to system sklepu wywołuje API w jego imieniu), SYSTEM (zadania systemowe/techniczne). Wszystkie kryteria są testowalne.

ID: US-001
Tytuł: Dodanie opinii (standard) 
Opis: Jako SHOP_CUSTOMER chcę dodać opinię (rating 1–5, treść, opcjonalny author, productId/variantId, userId) aby wzbogacić dane o produkt. 
Kryteria akceptacji:
- Gdy rating w zakresie 1..5 (int), treść ≤3000 znaków, brak HTML, żądanie POST zwraca 201 z identyfikatorem opinii i statusem zależnym od konfiguracji sklepu. 
- Jeśli sklep w trybie automatycznym: status = APPROVED. 
- Jeśli sklep w trybie moderowanym: status = PENDING. 
- Jeśli brak author: pole prezentowane w odpowiedziach jako Anonimowy (lub null -> mapowane). 
- Próba dodania drugiej aktywnej opinii tego samego userId dla tego samego productId (i variantId jeśli dotyczy) powoduje błąd 409 REVIEW_ALREADY_EXISTS. 

ID: US-002
Tytuł: Walidacja ratingu poza zakresem 
Opis: Jako SHOP_CUSTOMER otrzymuję błąd przy próbie dodania opinii z ratingiem spoza 1..5. 
Kryteria akceptacji:
- Rating = 0 lub 6 lub wartość niecałkowita powoduje 400 INVALID_RATING; opinia nie jest tworzona. 

ID: US-003
Tytuł: Walidacja treści (limit długości) 
Opis: Jako SHOP_CUSTOMER nie mogę zapisać zbyt długiej treści >3000 znaków. 
Kryteria akceptacji:
- Treść mająca 3001 znaków powoduje 400 CONTENT_TOO_LONG. 
- Treść 3000 znaków przechodzi. 

ID: US-004
Tytuł: Walidacja treści (HTML) 
Opis: Jako SHOP_CUSTOMER nie mogę dodać treści zawierającej tagi HTML 
Kryteria akceptacji:
- Treść zawierająca < script > lub < b > skutkuje 400 ILLEGAL_CONTENT. 
- Treść czysto tekstowa przechodzi. 

ID: US-005
Tytuł: Dodanie opinii anonimowej 
Opis: Jako SHOP_CUSTOMER mogę pominąć pole author, by opinia wyświetlała się jako Anonimowy. 
Kryteria akceptacji:
- Brak author w payload skutkuje odpowiedzią z polem author = Anonimowy (lub logiczny mapping przy prezentacji). 

ID: US-006
Tytuł: Konfiguracja trybu moderacji sklepu 
Opis: Jako SHOP_MODERATOR (admin sklepu) mogę ustawić tryb moderowany lub automatyczny. 
Kryteria akceptacji:
- Żądanie zmiany trybu zapisuje konfigurację dla tenantId; kolejne nowe opinie pojawiają się z odpowiednim statusem. 
- Istniejące opinie nie zmieniają statusu retroaktywnie. 

ID: US-007
Tytuł: Lista opinii do moderacji 
Opis: Jako SHOP_MODERATOR chcę pobrać listę opinii w statusie PENDING aby je ocenić. 
Kryteria akceptacji:
- GET endpoint (autoryzowany) zwraca tylko opinie PENDING danego tenantId. 
- Opinie są sortowane domyślnie malejąco po createdAt (konfigurowalne parametrem). 

ID: US-008
Tytuł: Zatwierdzenie opinii 
Opis: Jako SHOP_MODERATOR chcę zmienić status PENDING na APPROVED. 
Kryteria akceptacji:
- PATCH/POST moderacji z poprawnym ID w statusie PENDING ustawia APPROVED, zapisuje moderatedAt, audit log. 
- Próba zatwierdzenia już APPROVED zwraca 400 INVALID_STATUS_TRANSITION. 

ID: US-009
Tytuł: Odrzucenie opinii 
Opis: Jako SHOP_MODERATOR chcę odrzucić opinię PENDING. 
Kryteria akceptacji:
- Akcja zmienia status na REJECTED, zapis audit log. 
- Próba odrzucenia spoza PENDING (np. APPROVED) -> 400 INVALID_STATUS_TRANSITION. 

ID: US-010
Tytuł: Przywrócenie odrzuconej opinii 
Opis: Jako SHOP_MODERATOR chcę przywrócić opinię REJECTED do APPROVED. 
Kryteria akceptacji:
- Przejście REJECTED -> APPROVED dozwolone, moderatedAt aktualizowane. 
- Inne przejścia z REJECTED odrzucone. 

ID: US-011
Tytuł: Usunięcie opinii (soft delete) 
Opis: Jako SHOP_CUSTOMER chcę miękko usunąć własną opinię. 
Kryteria akceptacji:
- DELETE ustawia flagę deleted=true, opinia nie pojawia się w listach zwykłych ani agregatach. 
- Ponowne dodanie nowej opinii przez tego samego userId na ten sam produkt po usunięciu jest dozwolone. 

ID: US-012
Tytuł: Pobranie opinii publicznych per produkt 
Opis: Jako system sklepu (SHOP_VIEWER przez UI sklepu) chcę uzyskać listę zatwierdzonych opinii produktu/wariantu. 
Kryteria akceptacji:
- GET zwraca tylko APPROVED (w auto trybie wszystkie będą APPROVED). 
- Parametry sort: date|rating; order asc|desc. 
- Filtrowanie rating= X zwraca tylko opinie o konkretnym ratingu. 

ID: US-013
Tytuł: Pobranie wszystkich opinii (w tym PENDING/REJECTED) dla moderacji 
Opis: Jako SHOP_MODERATOR chcę pobrać pełny zestaw opinii z możliwością filtrowania po statusie. 
Kryteria akceptacji:
- GET (autoryzowany) param status=[PENDING|APPROVED|REJECTED|ALL]. 
- Brak status = ALL. 

ID: US-014
Tytuł: Agregaty ocen per produkt 
Opis: Jako system sklepu chcę uzyskać średnią i rozkład ocen, by wyświetlić rating produktu. 
Kryteria akceptacji:
- GET zwraca averageRating (float z precyzją 2), totalApprovedCount, counts map (1..5). 
- Odrzucone i niezatwierdzone (PENDING, REJECTED) nie wpływają na wyniki. 

ID: US-015
Tytuł: Dashboard – tabela sklepów 
Opis: Jako PLATFORM_ADMIN przeglądam listę sklepów z liczbą opinii w statusach i średnią ocen w oknie czasowym. 
Kryteria akceptacji:
- Domyślne okno 30 dni; parametr window=7|30. 
- Dane obejmują count_ALL, count_PENDING, count_APPROVED, count_REJECTED, averageApprovedRating. 
- Sortowanie po count_ALL lub averageApprovedRating (param sortBy, order). 

ID: US-016
Tytuł: Dashboard – Top 10 sklepów 
Opis: Jako PLATFORM_ADMIN chcę zobaczyć Top 10 sklepów wg liczby opinii w wybranym oknie. 
Kryteria akceptacji:
- GET zwraca max 10 rekordów posortowanych malejąco po count_ALL. 
- W przypadku remisu – wtórne sortowanie po averageApprovedRating desc. 

ID: US-017
Tytuł: Dashboard – ostatnie opinie globalnie 
Opis: Jako PLATFORM_ADMIN chcę przeglądać najnowsze opinie wszystkich sklepów porcjami po 50. 
Kryteria akceptacji:
- GET zwraca listę posortowaną malejąco po createdAt. 
- Parametry paginacji: limit (max 50, domyślnie 50) i cursor/offset. 

ID: US-018
Tytuł: Autoryzacja ról i izolacja tenantów 
Opis: Jako SYSTEM zapewniam, że użytkownik sklepu nie odczyta ani nie zmodyfikuje opinii innego sklepu. 
Kryteria akceptacji:
- Próba dostępu do zasobu innego tenantId -> 403 FORBIDDEN. 
- Dashboard dostępny tylko dla PLATFORM_ADMIN (SHOP_CUSTOMER otrzymuje 403). 

ID: US-019
Tytuł: Próba moderacji w trybie automatycznym 
Opis: Jako SHOP_MODERATOR w sklepie z trybem automatycznym nie muszę i nie mogę moderować (brak PENDING). 
Kryteria akceptacji:
- Próba wywołania endpointu moderacji na opinii APPROVED w sklepie auto -> 400 INVALID_OPERATION. 
- Nowo dodane opinie w sklepie auto nie pojawiają się jako PENDING. 

ID: US-020
Tytuł: Unikalność opinii (duplikat) 
Opis: Jako SYSTEM chcę blokować duplikat aktywnej opinii tego samego userId na ten sam produkt/wariant. 
Kryteria akceptacji:
- Druga próba POST przed usunięciem poprzedniej -> 409 REVIEW_ALREADY_EXISTS. 
- Po soft delete poprzedniej POST jest możliwy (201). 

ID: US-021
Tytuł: Przywrócenie odrzuconej opinii – audyt 
Opis: Jako PLATFORM_ADMIN chcę móc audytować zmianę REJECTED -> APPROVED. 
Kryteria akceptacji:
- Audit log zawiera rekord z reviewId, oldStatus=REJECTED, newStatus=APPROVED, changedBy. 
- Dostęp do logu ograniczony do PLATFORM_ADMIN. 

ID: US-022
Tytuł: Zgodność RODO – usunięcie treści 
Opis: Jako SHOP_CUSTOMER chcę usunąć swoją opinię, by nie pojawiała się w wynikach. 
Kryteria akceptacji:
- Po soft delete opinia nie jest zwracana w listach publicznych ani moderatora (domyślne filtry exclude deleted). 
- Agregaty nie uwzględniają tej opinii. 

ID: US-023
Tytuł: Pobranie opinii z filtrem rating 
Opis: Jako SHOP_VIEWER chcę zobaczyć tylko opinie o określonej ocenie. 
Kryteria akceptacji:
- GET z param rating=4 zwraca wyłącznie APPROVED rating==4. 
- Brak param rating zwraca wszystkie APPROVED. 

ID: US-024
Tytuł: Sortowanie opinii po ocenie 
Opis: Jako SHOP_VIEWER mogę sortować od najwyższej lub najniższej oceny. 
Kryteria akceptacji:
- sort=rating&order=desc pokazuje od 5 do 1. 
- sort=rating&order=asc odwrotnie. 
- Brak param sort = default sort date desc. 

ID: US-025
Tytuł: Obsługa wariantów produktu 
Opis: Jako SHOP_CUSTOMER dodając opinię mogę wskazać variantId (opcjonalne). 
Kryteria akceptacji:
- Jeśli variantId nie podane agregaty produktu obejmują obie opinie (wariantowe i bez variantId) – (Założenie: w MVP agregaty per produkt wliczają wszystkie jego warianty). 
- Możliwe rozszerzenie później (poza MVP) dla agregatów per wariant; w MVP endpoint może przyjąć variantId i wtedy ogranicza się do wariantu. 

ID: US-026
Tytuł: Pobranie agregatów dla wariantu 
Opis: Jako SHOP_CUSTOMER chcę uzyskać średnią i counts dla konkretnego variantId. 
Kryteria akceptacji:
- GET z param variantId zwraca statystyki tylko dla opinii z tym variantId. 
- Brak variantId -> agregaty dla całego produktu. 

ID: US-027
Tytuł: Ograniczenie długości odpowiedzi – brak stronicowania 
Opis: Jako PLATFORM_ADMIN jestem świadom ryzyka dużych odpowiedzi; MVP dopuszcza brak paginacji per produkt. 
Kryteria akceptacji:
- Endpoint listy opinii per produkt zwraca wszystkie pasujące (może to być duże – zaakceptowane w MVP). 

ID: US-028
Tytuł: Błąd przy nieuprawnionym dostępie do dashboardu 
Opis: Jako nie-admin próbując wejść na dashboard otrzymuję jasny komunikat o braku uprawnień. 
Kryteria akceptacji:
- Żądanie z rolą SHOP_CUSTOMER -> 403 { code: FORBIDDEN, message }. 

ID: US-029
Tytuł: Stabilność agregatów przy braku opinii 
Opis: Jako system chcę zwrócić zero wartości gdy brak opinii. 
Kryteria akceptacji:
- counts map zawiera klucze 1..5 z wartościami 0. 
- averageRating = null lub 0 (Założenie: null – konieczne ustalenie; przyjmujemy null). 

ID: US-030
Tytuł: Próba moderacji nieistniejącej opinii 
Opis: Jako SHOP_MODERATOR dostaję błąd gdy ID nie istnieje. 
Kryteria akceptacji:
- PATCH moderacji dla nieistniejącego ID -> 404 REVIEW_NOT_FOUND. 

ID: US-031
Tytuł: Paginacja ostatnich opinii globalnie 
Opis: Jako PLATFORM_ADMIN mogę pobrać kolejną stronę ostatnich opinii. 
Kryteria akceptacji:
- Pierwsze wywołanie zwraca 50 i token/offset. 
- Drugie z tokenem zwraca kolejne <=50 aż do wyczerpania. 

ID: US-032
Tytuł: Bezpieczne uwierzytelnienie API 
Opis: Jako SYSTEM wymagam ważnego tokena dla endpointów niepublicznych. 
Kryteria akceptacji:
- Brak / nieważny token -> 401 UNAUTHORIZED. 
- Token bez właściwej roli -> 403 FORBIDDEN. 

ID: US-033
Tytuł: Log audytu statusów 
Opis: Jako PLATFORM_ADMIN mogę pobrać historię zmian statusów danej opinii. 
Kryteria akceptacji:
- GET audit/reviews/{id} zwraca chronologiczną listę zmian. 
- Brak uprawnień -> 403. 

ID: US-034
Tytuł: Zmiana konfiguracji moderacji – wpływ na nowe opinie 
Opis: Jako SHOP_MODERATOR zmieniając tryb z PENDING na AUTO powoduję, że nowe opinie są APPROVED od razu. 
Kryteria akceptacji:
- Opinia utworzona przed zmianą zachowuje status PENDING do moderacji. 

ID: US-035
Tytuł: Próba przywrócenia opinii nie-REJECTED 
Opis: Jako SHOP_MODERATOR dostaję błąd przy próbie REJECTED->APPROVED jeśli opinia nie jest REJECTED. 
Kryteria akceptacji:
- PATCH restore na APPROVED -> 400 INVALID_STATUS_TRANSITION. 

ID: US-036
Tytuł: Anonimizacja po soft delete (widoczność) 
Opis: Jako PLATFORM_ADMIN nie widzę usuniętych opinii w dashboardzie (licznik ALL ich nie uwzględnia). 
Kryteria akceptacji:
- count_ALL w dashboardzie nie zawiera deleted. 

ID: US-037
Tytuł: Brak odmiennej walidacji języka 
Opis: Jako SHOP_CUSTOMER podając language system akceptuje dowolny kod ISO (niefiltrowane później). 
Kryteria akceptacji:
- tylko poprawny kod iso języka (np. en, pl, de) jest akceptowany; brak lub niepoprawny -> 400 INVALID_LANGUAGE_CODE.
- dopuszczalne puste (null) language.

ID: US-038
Tytuł: Obsługa równoczesnych prób duplikatu 
Opis: Jako SYSTEM zapewniam atomiczność – przy wyścigu dwóch identycznych POST jedna zakończy się 201, druga 409. 
Kryteria akceptacji:
- Test równoległy: dokładnie jedna akceptowana. 

ID: US-039
Tytuł: Próba usunięcia już usuniętej opinii 
Opis: Jako SHOP_CUSTOMER otrzymuję właściwy błąd informacyjny. 
Kryteria akceptacji:
- DELETE na deleted=true -> 400 ALREADY_DELETED. 

ID: US-040
Tytuł: Próba moderacji usuniętej opinii 
Opis: Jako SHOP_MODERATOR nie mogę moderować soft deleted opinii. 
Kryteria akceptacji:
- PATCH moderacji -> 400 INVALID_OPERATION gdy deleted=true. 

ID: US-041
Tytuł: Spójny format błędów 
Opis: Jako integrator chcę otrzymywać spójne JSON błędów. 
Kryteria akceptacji:
- Struktura: { code, message, details? }. 
- Wszystkie walidacje i autoryzacje stosują ten format. 

ID: US-042
Tytuł: Agregaty przy dużej liczbie opinii 
Opis: Jako PLATFORM_ADMIN chcę by agregaty działały szybko. 
Kryteria akceptacji:
- Dla produktu o 10k aprob. opinii odpowiedź agregatów < 500 ms w środowisku referencyjnym (założenie). 

ID: US-043
Tytuł: Wsparcie variantId null 
Opis: Jako SHOP_CUSTOMER mogę dodać opinię bez variantId. 
Kryteria akceptacji:
- Brak variantId = null zapisane; agregaty produktu obejmują tę opinię. 

ID: US-044
Tytuł: Filtr status w zapytaniu admin do opinii 
Opis: Jako PLATFORM_ADMIN mogę filtrować opinię globalnie (opcjonalnie) – (Założenie: nie w MVP; ODRZUCONE). 
Kryteria akceptacji:
- Odrzucone dla MVP; brak endpointu globalnego listującego wszystkie opinie (poza ostatnimi). 

ID: US-045
Tytuł: Ustalony format dat 
Opis: Jako integrator otrzymuję daty w ISO 8601 UTC. 
Kryteria akceptacji:
- Wszystkie pola czasowe w odpowiedziach mają sufiks Z. 

ID: US-046
Tytuł: Brak stronicowania list per produkt – akceptacja ryzyka 
Opis: Jako PLATFORM_ADMIN akceptuję zwiększony transfer dla produktów z wieloma opiniami w MVP. 
Kryteria akceptacji:
- Nie implementuje się limit/offset w tym endpoint w MVP. 

ID: US-047
Tytuł: Próba zmiany statusu z APPROVED na PENDING 
Opis: Jako SHOP_MODERATOR nie mogę cofnąć do PENDING. 
Kryteria akceptacji:
- PATCH do PENDING -> 400 INVALID_STATUS_TRANSITION. 

ID: US-048
Tytuł: Próba odrzucenia REJECTED 
Opis: Jako SHOP_MODERATOR nie mogę jeszcze raz odrzucić REJECTED. 
Kryteria akceptacji:
- REJECTED -> REJECTED request -> 400 INVALID_STATUS_TRANSITION. 

ID: US-049
Tytuł: Podstawowa dostępność API 
Opis: Jako integrator chcę by podstawowe endpointy działały (healthcheck). 
Kryteria akceptacji:
- GET /health -> 200 { status: UP }. 

ID: US-050
Tytuł: Rozdzielenie roli moderatora (opcjonalnie) 
Opis: Jako SHOP_CUSTOMER mogę delegować moderację do dedykowanego SHOP_MODERATOR. 
Kryteria akceptacji:
- Użytkownik z rolą SHOP_MODERATOR może moderować; SHOP_CUSTOMER bez uprawnień moderacji -> 403. 

ID: US-051
Tytuł: Brak opinii przy zapytaniu listy 
Opis: Jako SHOP_VIEWER widzę pustą listę ([], 200) zamiast błędu. 
Kryteria akceptacji:
- Produkt bez opinii -> []. 

ID: US-052
Tytuł: Średnia przy braku opinii 
Opis: Jako SHOP_VIEWER widzę null dla średniej. 
Kryteria akceptacji:
- averageRating = null, totalApprovedCount=0. 

ID: US-053
Tytuł: Format mapy ocen pełny 
Opis: Jako integrator zawsze otrzymuję pełen zakres 1..5. 
Kryteria akceptacji:
- counts = {1: x1, 2: x2, 3: x3, 4: x4, 5: x5}. 

ID: US-054
Tytuł: Próba wstrzyknięcia HTML w author 
Opis: Jako SYSTEM waliduję, że author nie zawiera HTML. 
Kryteria akceptacji:
- author z <b> -> 400 ILLEGAL_CONTENT. 

ID: US-055
Tytuł: Filtrowanie opinii po dacie (rozszerzalność) 
Opis: Jako integrator mogę opcjonalnie podać przedział dat (Założenie: w MVP nie implementujemy – ODRZUCONE). 
Kryteria akceptacji:
- Brak parametrów dateFrom/dateTo w MVP. 

ID: US-056
Tytuł: Próba aktualizacji opinii (brak w MVP) 
Opis: Jako SHOP_CUSTOMER nie mogę edytować istniejącej opinii. 
Kryteria akceptacji:
- PUT/PATCH treści -> 405 METHOD_NOT_ALLOWED (lub 404 – Założenie: 405). 

ID: US-057
Tytuł: Wymuszenie JSON 
Opis: Jako SYSTEM odrzucam nie-JSON payloady. 
Kryteria akceptacji:
- Content-Type != application/json -> 415 UNSUPPORTED_MEDIA_TYPE. 

ID: US-058
Tytuł: Obsługa wielojęzyczna (pole language) 
Opis: Jako SHOP_CUSTOMER mogę zapisać language=pl. 
Kryteria akceptacji:
- Odczyt opinii zawiera pole language. 

ID: US-059
Tytuł: Attempt moderacji opinii w trybie auto (brak PENDING) 
Opis: (Duplikat logiki z US-019) Upewniamy się, że testy pokrywają brak PENDING. 
Kryteria akceptacji:
- Po przełączeniu na auto brak zwracanych PENDING. 

ID: US-060
Tytuł: Spójność średniej po zmianie statusu 
Opis: Jako SYSTEM aktualizuję średnią po APPROVED opinii natychmiast. 
Kryteria akceptacji:
- Po zatwierdzeniu nowej opinii agregat odzwierciedla ją w kolejnym GET. 

## 6. Metryki sukcesu
Metryki biznesowe i produktowe:
1. Pokrycie funkcjonalne: 100% implementacji historii oznaczonych jako in-scope (US-001..US-060 z wyjątkiem oznaczonych ODRZUCONE/poza zakresem). 
2. Jakość danych: <0.1% błędów duplikatów (sprawdzone w logach); brak niespójnych agregatów (średnia = wyliczona z countów). 
3. Wydajność API: 
   - Dodanie opinii P95 < 300 ms przy obciążeniu referencyjnym (np. 50 req/s). 
   - Pobranie agregatów produktu P95 < 200 ms. 
4. Stabilność: uptime API > 99% w okresie pilotażu. 
5. Dashboard adoption: min. 80% aktywnych tygodniowo (PLATFORM_ADMIN loguje się ≥1 raz/tydzień w okresie pilotażowym). 
6. Średnia satysfakcja integratorów (ankieta) ≥4/5 (po 1 miesiącu). 
7. Szybkość moderacji: 90% opinii PENDING rozstrzygnięte w <72h (metryka do obserwacji). 
8. Bezpieczeństwo: 0 krytycznych incydentów naruszenia izolacji tenantów. 
9. Skala: System poprawnie obsługuje ≥50 sklepów pilotażowych oraz >=100k opinii skumulowanych bez degradacji P95 > zdefiniowanych progów. 
10. RODO: 100% próśb o usunięcie opinii zrealizowanych < 7 dni (soft delete). 

Checklist wewnętrzna (weryfikacja PRD):
- Wszystkie historyjki mają testowalne kryteria: TAK. 
- Kryteria są jasne i konkretne: TAK (jednoznaczne kody błędów, stany). 
- Zestaw historii pokrywa pełen zakres MVP: TAK (dodawanie, walidacje, moderacja, usuwanie, pobieranie, agregaty, dashboard, bezpieczeństwo). 
- Wymagania dot. uwierzytelniania/autoryzacji uwzględnione: TAK (US-018, US-032, plus powiązane). 
- Historie out-of-scope oznaczone (US-044, US-055) aby uniknąć nieporozumień. 

Koniec dokumentu.
