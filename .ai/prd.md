# Dokument wymagań produktu (PRD) - a4b-reviews

## 1. Przegląd produktu

a4b-reviews to mikroserwis do zarządzania opiniami o produktach przeznaczony dla wielotenantowej platformy e-commerce działającej w modelu SaaS. Aplikacja umożliwia klientom sklepów internetowych dodawanie ocen punktowych oraz tekstowych opinii o zakupionych produktach, a administratorom sklepów moderowanie tych opinii.

System został zaprojektowany jako wewnętrzny mikroserwis obsługujący wiele niezależnych sklepów internetowych (tenantów), gdzie każdy sklep posiada własną przestrzeń danych i konfigurację. Aplikacja udostępnia REST API dla systemów e-commerce oraz dashboard administratora systemu do monitorowania wykorzystania usługi.

Kluczowe charakterystyki:
- Architektura multi-tenant z izolacją danych per sklep
- REST API chronione uwierzytelnianiem Basic Auth
- Identyfikacja tenanta przez nagłówek HTTP X-Account
- Trzy tryby moderacji: automatyczny, ręczny oraz wspomagany AI
- Integracja z OpenAI API do automatycznej analizy treści opinii
- Dashboard administratora systemu z kluczowymi metrykami i statystykami
- Obsługa opinii dla produktów oraz ich wariantów
- Mechanizm soft delete zgodny z wymaganiami RODO

## 2. Problem użytkownika

Platformy e-commerce działające w modelu SaaS obsługują wielu niezależnych sprzedawców, z których każdy potrzebuje systemu do zbierania i zarządzania opiniami klientów o swoich produktach. Opinie są kluczowym elementem budowania zaufania i zwiększania konwersji w e-commerce.

Główne wyzwania:
- Brak scentralizowanego systemu do zbierania opinii dla wszystkich sklepów w platformie
- Konieczność moderacji opinii w celu eliminacji treści niedozwolonych (wulgaryzmy, mowa nienawiści, dane osobowe)
- Czasochłonność ręcznej moderacji przy dużej liczbie opinii
- Potrzeba szybkiego prezentowania opinii na stronach produktów
- Różne potrzeby moderacyjne różnych sklepów (od pełnej automatyzacji po pełną kontrolę ręczną)
- Brak narzędzi do monitorowania wykorzystania systemu opinii przez administratora platformy
- Konieczność zapewnienia bezpieczeństwa danych i zgodności z RODO

Opinie wpływają bezpośrednio na decyzje zakupowe klientów. Badania pokazują, że większość kupujących online czyta opinie przed dokonaniem zakupu, a pozytywne recenzje znacząco zwiększają konwersję. Jednocześnie negatywne lub nieodpowiednie treści mogą zaszkodzić wizerunkowi sklepu.

Różni sprzedawcy mają różne podejścia do moderacji - niektórzy preferują pełną kontrolę nad publikowanymi treściami, inni wolą automatyzację. System musi być na tyle elastyczny, aby wspierać oba podejścia oraz rozwiązania hybrydowe.

## 3. Wymagania funkcjonalne

### 3.1 Zarządzanie opiniami

FR-001: System umożliwia dodawanie opinii o produktach przez klientów sklepów za pomocą REST API
- Endpoint: POST /reviews
- Wymagane pola: userId (identyfikator użytkownika), productId (identyfikator produktu), rating (ocena 1-5), reviewText (treść opinii), orderId (identyfikator zamówienia)
- Opcjonalne pola: variantId (identyfikator wariantu produktu), author (nazwa autora), metadata (JSONB), media (JSONB)
- System automatycznie przypisuje: id opinii, timestamp utworzenia, status (zgodnie z trybem moderacji tenanta), language (ISO 639-1)
- Nagłówek X-Account identyfikuje tenanta

FR-002: System przechowuje opinie w bazie danych z pełną izolacją między tenantami
- Model danych opinii zawiera: id, userId, author, orderId, productId, variantId, rating, reviewText, status, createdAt, updatedAt, deletedAt, metadata, media, language, classificationScore, classificationReason
- Każda opinia jest powiązana z konkretnym tenantem
- Implementacja soft delete przez pole deletedAt
- Pole metadata jako JSONB na przyszłe zastosowania
- Pole media jako JSONB na przyszłe zastosowania

FR-003: System umożliwia pobieranie opinii dla produktu za pomocą REST API
- Endpoint: GET /products/{id}/reviews
- Zwraca wszystkie zatwierdzone opinie dla danego produktu (status APPROVED)
- Nie zawiera opinii usuniętych (deletedAt != null)
- Wspiera filtrowanie po ocenie: ?rating=5
- Wspiera sortowanie: ?sort=date_asc, ?sort=date_desc, ?sort=rating_asc, ?sort=rating_desc
- Brak paginacji (zwraca pełną listę)

FR-004: System umożliwia pobieranie opinii dla wariantu produktu za pomocą REST API
- Endpoint: GET /variants/{id}/reviews
- Zwraca wszystkie zatwierdzone opinie dla danego wariantu (status APPROVED)
- Opinia dodana do wariantu ma zapisane zarówno variantId jak i productId
- Wspiera te same opcje filtrowania i sortowania co endpoint produktowy
- Brak paginacji (zwraca pełną listę)

FR-005: System udostępnia zagregowane podsumowanie opinii dla produktu
- Endpoint: GET /products/{id}/reviews/summary
- Zwraca: średnią ocenę, całkowitą liczbę opinii, mapę z liczbą poszczególnych ocen (np. {"5": 10, "4": 5, "3": 2, "2": 1, "1": 0})
- Uwzględnia tylko zatwierdzone opinie (status APPROVED)
- Nie uwzględnia opinii usuniętych

FR-006: System udostępnia zagregowane podsumowanie opinii dla wariantu produktu
- Endpoint: GET /variants/{id}/reviews/summary
- Zwraca te same dane co dla produktu: średnią ocenę, liczbę opinii, mapę ocen
- Uwzględnia tylko zatwierdzone opinie przypisane do danego wariantu

### 3.2 Moderacja opinii

FR-007: System wspiera trzy tryby moderacji konfigurowane per tenant
- ALLOW_ALL: Wszystkie opinie są automatycznie zatwierdzane (status APPROVED)
- MODERATION_MANUAL: Wszystkie opinie trafiają do ręcznej moderacji (status PENDING)
- MODERATION_AI: Opinie są analizowane przez AI, bezpieczne otrzymują status APPROVED, podejrzane status VERIFICATION

FR-008: System integruje się z OpenAI API do analizy treści opinii (tryb MODERATION_AI)
- Analiza wykrywa: wulgaryzmy, mowę nienawiści, dane osobowe, treści erotyczne
- Wynik analizy zapisywany w polach classificationScore i classificationReason
- W przypadku awarii OpenAI API, opinie domyślnie trafiają do statusu VERIFICATION (fail-safe)
- System kontynuuje działanie nawet przy niedostępności OpenAI API

FR-009: System udostępnia kolejkę opinii oczekujących na moderację
- Endpoint: GET /reviews/queue
- Zwraca opinie ze statusem PENDING lub VERIFICATION
- Wspiera paginację (mechanizm kursora)
- Filtrowane według tenanta (nagłówek X-Account)

FR-010: System umożliwia zmianę statusu opinii przez moderatora
- Endpoint: PATCH /reviews/{id}/status
- Możliwe przejścia statusów: PENDING -> APPROVED/REJECTED, VERIFICATION -> APPROVED/REJECTED
- Tylko dla opinii należących do tenanta określonego w nagłówku X-Account

FR-011: System implementuje mechanizm soft delete dla opinii
- Usuwanie opinii ustawia pole deletedAt na aktualny timestamp
- Usunięte opinie nie są zwracane przez publiczne endpointy
- Usunięte opinie są zachowane w bazie danych dla celów audytowych
- Zgodność z wymaganiami RODO

### 3.3 Uwierzytelnianie i autoryzacja

FR-012: System wymaga uwierzytelniania Basic Auth dla wszystkich endpointów API
- Statyczny klucz (secret) per środowisko
- Klucz przechowywany w usłudze typu vault
- Brak dostępu do API bez poprawnego uwierzytelnienia

FR-013: System identyfikuje tenant na podstawie nagłówka HTTP X-Account
- Każde żądanie API musi zawierać nagłówek X-Account z unikalnym kluczem tekstowym tenanta (np. "tomex")
- Brak dostępu do danych bez poprawnego nagłówka X-Account
- Pełna izolacja danych między tenantami

FR-014: Autoryzacja operacji leży po stronie centralnego systemu e-commerce
- System a4b-reviews nie weryfikuje uprawnień użytkownika do dodania opinii
- Weryfikacja, czy użytkownik ma prawo wystawić opinię (np. czy kupił produkt), odbywa się w systemie e-commerce przed wywołaniem API

### 3.4 Dashboard administratora systemu

FR-015: System udostępnia dashboard administratora z kluczowymi statystykami
- Interfejs webowy read-only
- Liczba tenantów w systemie
- Tabelaryczne podsumowanie opinii per tenant (nazwa tenanta, tryb moderacji, liczba opinii w poszczególnych statusach, średnia ocena)
- Lista 50 ostatnich opinii z całego systemu z paginacją kursorem
- TOP 10 tenantów według liczby opinii i według średniej oceny
- Widget skuteczności moderacji AI

FR-016: Widget skuteczności moderacji AI prezentuje kluczowe metryki
- Procent opinii automatycznie zatwierdzonych vs. wysłanych do weryfikacji
- Spośród opinii zweryfikowanych: procent zaakceptowanych vs. odrzuconych przez moderatorów
- Pomaga ocenić jakość klasyfikacji AI
- Dane agregowane dla wszystkich tenantów używających trybu MODERATION_AI

FR-017: Dashboard nie zawiera funkcji drążenia danych (drill-down) ani eksportu w MVP
- Brak możliwości kliknięcia w tenant i zobaczenia szczegółów
- Brak eksportu danych do CSV/Excel
- Ograniczenie zakresu do kluczowych metryk dla MVP

## 4. Granice produktu

### 4.1 Co NIE wchodzi w zakres MVP

OOS-001: Import opinii z zewnętrznych źródeł
- System nie wspiera importu istniejących opinii z innych platform
- Brak integracji z bazami opinii spoza systemu
- Może być dodane w przyszłych wersjach

OOS-002: Interfejs webowy do zarządzania opiniami dla administratorów sklepów
- Administratorzy sklepów korzystają wyłącznie z REST API
- Brak dedykowanego panelu administracyjnego dla moderatorów sklepów
- System e-commerce może zbudować własny interfejs korzystając z API

OOS-003: Dodawanie zdjęć i filmów do opinii
- MVP ogranicza się do ocen punktowych i tekstu
- Pole media w JSONB przygotowane na przyszłość, ale funkcjonalność nie jest implementowana
- Może być dodane w przyszłych wersjach

OOS-004: Integracja z zewnętrznymi systemami opinii
- Brak integracji z Trusted Shops, Google Reviews, itp.
- System działa jako samodzielne rozwiązanie
- Może być dodane w przyszłych wersjach

OOS-005: Odpowiedzi sprzedawców na opinie
- Brak możliwości dodawania komentarzy/odpowiedzi od sprzedawców
- Może być dodane w przyszłych wersjach

OOS-006: Reakcje użytkowników na opinie (helpful/not helpful)
- Brak mechanizmu głosowania na przydatność opinii
- Może być dodane w przyszłych wersjach

OOS-007: Zaawansowane analizy i raporty
- Dashboard administratora systemu ogranicza się do kluczowych metryk
- Brak szczegółowych raportów, wykresów trendów, eksportu danych
- Może być dodane w przyszłych wersjach

OOS-008: Powiadomienia email/SMS o nowych opiniach
- System nie wysyła powiadomień
- System e-commerce może zaimplementować powiadomienia korzystając z API
- Może być dodane w przyszłych wersjach

### 4.2 Kwestie odroczone (wymagają doprecyzowania przed implementacją)

DEF-001: Strategia obsługi błędów API
- Konkretne kody odpowiedzi HTTP
- Formaty komunikatów o błędach
- Obsługa błędów walidacji

DEF-002: Szczegółowa walidacja pól
- Maksymalna długość tekstu opinii
- Dozwolone formaty ID
- Walidacja oceny (1-5)

DEF-003: Ochrona przed spamem
- Ograniczenie "jeden użytkownik - jedna opinia na produkt"
- Rate limiting
- Mechanizmy antyspamowe

DEF-004: Wydajność dashboardu
- Czy statystyki liczone w czasie rzeczywistym czy z cache
- Częstotliwość odświeżania danych
- Strategia cache'owania

DEF-005: Uwierzytelnianie do dashboardu administratora
- Sposób zabezpieczenia dostępu do interfejsu webowego
- Mechanizm inny niż secret dla API
- Zarządzanie kontami administratorów systemu

DEF-006: Logowanie audytowe
- Zakres logowania zmian (kto, kiedy, co zmienił)
- Przechowywanie historii zmian statusów opinii
- Retencja logów

DEF-007: Szczegóły implementacji AI
- Dokładna treść i struktura promptu dla OpenAI API
- Dokładne typy danych dla classificationScore i classificationReason
- Strategia zarządzania kosztami i limitami OpenAI API
- Obsługa limitów rate limit OpenAI

## 5. Historyjki użytkowników

### 5.1 Klient sklepu

US-001: Dodawanie opinii o zakupionym produkcie
Jako klient sklepu, chcę dodać ocenę punktową i opinię tekstową do produktu, który zakupiłem, aby podzielić się moimi doświadczeniami z innymi potencjalnymi kupującymi.

Kryteria akceptacji:
- System przyjmuje żądanie POST /reviews z wymaganymi polami: userId, productId, rating (1-5), reviewText, orderId
- System opcjonalnie przyjmuje pole author (wyświetlana nazwa autora)
- System automatycznie przypisuje timestamp utworzenia opinii
- System zwraca status 201 Created wraz z ID utworzonej opinii
- System przypisuje odpowiedni status początkowy zgodnie z trybem moderacji tenanta
- Opinia jest powiązana z tenantem identyfikowanym przez nagłówek X-Account
- W przypadku braku wymaganych pól system zwraca błąd walidacji

US-002: Dodawanie opinii o konkretnym wariancie produktu
Jako klient sklepu, chcę dodać opinię o konkretnym wariancie produktu (np. rozmiar, kolor), który zakupiłem, aby moja recenzja dotyczyła dokładnie tego wariantu.

Kryteria akceptacji:
- System przyjmuje żądanie POST /reviews z opcjonalnym polem variantId
- Gdy variantId jest podany, system zapisuje zarówno variantId jak i productId
- Opinia jest widoczna przy pobieraniu opinii dla tego konkretnego wariantu
- Opinia jest również widoczna przy pobieraniu opinii dla całego produktu
- System waliduje, że variantId należy do podanego productId (walidacja po stronie wywołującego)

US-003: Pobieranie opinii o produkcie przed zakupem
Jako potencjalny klient sklepu, chcę zobaczyć wszystkie opinie o produkcie, aby podjąć świadomą decyzję zakupową.

Kryteria akceptacji:
- System udostępnia endpoint GET /products/{id}/reviews
- System zwraca tylko zatwierdzone opinie (status APPROVED)
- System nie zwraca opinii usuniętych (deletedAt != null)
- Opinie są zwracane w formacie JSON z pełnymi danymi (rating, reviewText, author, createdAt, etc.)
- System zwraca pełną listę bez paginacji
- Opinie należą tylko do tenanta określonego w nagłówku X-Account

US-004: Filtrowanie opinii według oceny
Jako klient sklepu, chcę zobaczyć tylko opinie z określoną oceną (np. tylko 5-gwiazdkowe), aby szybciej znaleźć najbardziej pozytywne lub negatywne recenzje.

Kryteria akceptacji:
- System wspiera parametr zapytania ?rating={1-5} dla endpointu GET /products/{id}/reviews
- System zwraca tylko opinie z podaną oceną
- Gdy parametr rating nie jest podany, zwracane są wszystkie opinie
- System waliduje, że rating jest liczbą od 1 do 5

US-005: Sortowanie opinii
Jako klient sklepu, chcę móc posortować opinie według daty lub oceny, aby najpierw zobaczyć najnowsze lub najbardziej/najmniej pozytywne recenzje.

Kryteria akceptacji:
- System wspiera parametr zapytania ?sort={date_asc|date_desc|rating_asc|rating_desc}
- date_asc: najstarsze opinie pierwsze
- date_desc: najnowsze opinie pierwsze
- rating_asc: najniższe oceny pierwsze
- rating_desc: najwyższe oceny pierwsze
- Domyślne sortowanie gdy parametr nie jest podany: date_desc
- System waliduje wartość parametru sort

US-006: Sprawdzenie podsumowania opinii o produkcie
Jako klient sklepu, chcę szybko zobaczyć średnią ocenę produktu i rozkład ocen, aby ocenić ogólny odbiór produktu bez czytania wszystkich opinii.

Kryteria akceptacji:
- System udostępnia endpoint GET /products/{id}/reviews/summary
- System zwraca średnią ocenę (np. 4.5)
- System zwraca całkowitą liczbę opinii
- System zwraca mapę z liczbą poszczególnych ocen w formacie {"5": 10, "4": 5, "3": 2, "2": 1, "1": 0}
- Uwzględniane są tylko zatwierdzone opinie (status APPROVED)
- Nie uwzględniane są opinie usunięte

US-007: Pobieranie opinii o konkretnym wariancie produktu
Jako klient sklepu, chcę zobaczyć opinie o wariancie produktu, aby wiedzieć, czy wariant spełnia oczekiwania.

Kryteria akceptacji:
- System udostępnia endpoint GET /variants/{id}/reviews
- System zwraca tylko opinie przypisane do tego wariantu (variantId)
- Endpoint wspiera te same opcje filtrowania i sortowania co endpoint produktowy
- System zwraca podsumowanie dla wariantu przez GET /variants/{id}/reviews/summary

### 5.2 Administrator sklepu (moderator)

US-008: Przeglądanie kolejki opinii oczekujących na moderację
Jako administrator sklepu, chcę mieć dostęp do listy opinii oczekujących na moją decyzję, aby móc je sprawnie moderować.

Kryteria akceptacji:
- System udostępnia endpoint GET /reviews/queue
- System zwraca opinie ze statusem PENDING lub VERIFICATION
- Opinie są filtrowane według tenanta (nagłówek X-Account)
- System wspiera paginację kursorem dla dużej liczby opinii
- Każda opinia zawiera pełne dane potrzebne do podjęcia decyzji: rating, reviewText, author, userId, productId, createdAt
- Dla opinii ze statusem VERIFICATION: widoczny jest classificationScore i classificationReason z analizy AI

US-009: Zatwierdzanie opinii
Jako administrator sklepu, chcę zatwierdzić opinię, aby została opublikowana i była widoczna dla klientów.

Kryteria akceptacji:
- System udostępnia endpoint PATCH /reviews/{id}/status
- System przyjmuje żądanie ze statusem APPROVED
- Możliwe przejście ze statusu PENDING lub VERIFICATION na APPROVED
- Po zatwierdzeniu opinia jest widoczna w endpointach publicznych (GET /products/{id}/reviews)
- System weryfikuje, że opinia należy do tenanta określonego w nagłówku X-Account
- System zwraca błąd, gdy próbujemy zatwierdzić opinię innego tenanta

US-010: Odrzucanie opinii
Jako administrator sklepu, chcę odrzucić opinię zawierającą nieodpowiednie treści, aby chronić wizerunek sklepu i zapewnić wysoką jakość opinii.

Kryteria akceptacji:
- System udostępnia endpoint PATCH /reviews/{id}/status
- System przyjmuje żądanie ze statusem REJECTED
- Możliwe przejście ze statusu PENDING lub VERIFICATION na REJECTED
- Odrzucona opinia nie jest widoczna w endpointach publicznych
- System weryfikuje, że opinia należy do tenanta określonego w nagłówku X-Account
- Odrzucone opinie są zachowane w bazie danych (nie są usuwane fizycznie)

US-011: Ponowna weryfikacja opinii oflagowanej przez AI
Jako administrator sklepu, chcę przejrzeć opinie oflagowane przez system AI jako potencjalnie problematyczne, aby podjąć ostateczną decyzję o ich publikacji.

Kryteria akceptacji:
- Opinie ze statusem VERIFICATION są widoczne w GET /reviews/queue
- Administrator widzi classificationScore i classificationReason wskazujące powód oflagowania
- Administrator może zmienić status na APPROVED lub REJECTED
- System wspiera zarówno zatwierdzanie jak i odrzucanie opinii oflagowanych przez AI

### 5.3 System e-commerce (integracja API)

US-012: Pobieranie opinii do wyświetlenia na karcie produktu
Jako system e-commerce, chcę pobrać wszystkie zatwierdzone opinie dla danego produktu, aby wyświetlić je na karcie produktu w sklepie internetowym.

Kryteria akceptacji:
- System udostępnia endpoint GET /products/{id}/reviews
- Endpoint zwraca kompletne dane opinii w formacie JSON
- Zwracane są tylko opinie zatwierdzone (status APPROVED)
- Nie są zwracane opinie usunięte
- Endpoint wspiera filtrowanie i sortowanie
- System e-commerce otrzymuje pełną listę bez konieczności obsługi paginacji

US-013: Wyświetlanie podsumowania ocen produktu
Jako system e-commerce, chcę pobrać zagregowane dane o opiniach produktu, aby wyświetlić średnią ocenę i rozkład ocen na karcie produktu.

Kryteria akceptacji:
- System udostępnia endpoint GET /products/{id}/reviews/summary
- Endpoint zwraca dane w przewidywalnym formacie JSON
- Dane obejmują: średnią ocenę, liczbę opinii, rozkład ocen
- Dane są aktualne (uwzględniają najnowsze zatwierdzone opinie)
- Endpoint ma niski czas odpowiedzi (wymóg wydajnościowy)

US-014: Zbieranie opinii od klienta po zakupie
Jako system e-commerce, chcę umożliwić klientowi dodanie opinii po zakupie produktu, integrując się z API opinii.

Kryteria akceptacji:
- System e-commerce weryfikuje, czy użytkownik zakupił produkt (na podstawie orderId)
- System e-commerce wywołuje POST /reviews z wymaganymi danymi
- System e-commerce przekazuje prawidłowy nagłówek X-Account identyfikujący sklep
- System e-commerce obsługuje odpowiedzi sukcesu (201) i błędów (4xx, 5xx)
- Opinia jest dodana z odpowiednim statusem początkowym zgodnym z trybem moderacji sklepu

US-015: Budowanie interfejsu moderacji dla administratorów sklepu
Jako system e-commerce, chcę zbudować interfejs moderacji opinii dla administratorów sklepów, korzystając z API opinii.

Kryteria akceptacji:
- System e-commerce pobiera kolejkę opinii przez GET /reviews/queue
- System e-commerce wyświetla opinie oczekujące na moderację
- System e-commerce umożliwia zatwierdzenie opinii przez PATCH /reviews/{id}/status (APPROVED)
- System e-commerce umożliwia odrzucenie opinii przez PATCH /reviews/{id}/status (REJECTED)
- System e-commerce obsługuje paginację kursorem dla dużej liczby opinii

### 5.4 Administrator systemu

US-016: Monitorowanie ogólnego wykorzystania systemu
Jako administrator systemu, chcę widzieć zagregowane statystyki dotyczące liczby opinii we wszystkich sklepach, aby monitorować ogólną kondycję i wykorzystanie usługi.

Kryteria akceptacji:
- Dashboard administratora wyświetla liczbę tenantów w systemie
- Dashboard wyświetla tabelaryczne podsumowanie opinii per tenant (nazwa tenanta, liczba opinii, tryb moderacji)
- Dashboard wyświetla TOP 10 tenantów według liczby opinii i według średniej oceny
- Dane są aktualne i odzwierciedlają bieżący stan systemu
- Interfejs jest czytelny i responsywny

US-017: Przeglądanie ostatnich opinii w całym systemie
Jako administrator systemu, chcę widzieć listę ostatnich opinii dodanych do systemu we wszystkich sklepach, aby monitorować aktywność i szybko identyfikować potencjalne problemy.

Kryteria akceptacji:
- Dashboard wyświetla 50 ostatnich opinii z całego systemu
- Lista zawiera podstawowe informacje: tenant, productId, rating, fragment tekstu, status, createdAt
- Dashboard wspiera paginację kursorem dla przeglądania starszych opinii
- Opinie są sortowane od najnowszych do najstarszych
- Interfejs umożliwia szybką identyfikację tenanta dla każdej opinii

US-018: Ocena skuteczności moderacji AI
Jako administrator systemu, chcę widzieć statystyki skuteczności moderacji AI, aby ocenić, czy system działa poprawnie i czy AI właściwie klasyfikuje opinie.

Kryteria akceptacji:
- Dashboard wyświetla widget skuteczności moderacji AI
- Widget pokazuje procent opinii automatycznie zatwierdzonych (status APPROVED) vs. wysłanych do weryfikacji (status VERIFICATION)
- Widget pokazuje procent opinii zaakceptowanych vs. odrzuconych spośród tych zweryfikowanych przez moderatorów
- Dane dotyczą wszystkich tenantów używających trybu MODERATION_AI
- Widget pomaga zidentyfikować, czy AI jest zbyt restrykcyjne (za dużo false positives) lub zbyt liberalne (za dużo false negatives po weryfikacji)

US-019: Logowanie do dashboardu administratora
Jako administrator systemu, chcę bezpiecznie zalogować się do dashboardu, aby tylko autoryzowane osoby miały dostęp do statystyk systemu.

Kryteria akceptacji:
- Dashboard wymaga uwierzytelnienia
- Mechanizm uwierzytelniania jest bezpieczny i oddzielony od Basic Auth używanego dla API
- Po zalogowaniu administrator ma dostęp do wszystkich funkcji dashboardu
- Sesja wygasa po określonym czasie nieaktywności
- System loguje próby dostępu do dashboardu dla celów audytowych

### 5.5 System AI (automatyczna moderacja)

US-020: Automatyczna analiza nowych opinii w trybie MODERATION_AI
Jako system, chcę automatycznie analizować nowe opinie pod kątem niedozwolonych treści, aby odciążyć moderatorów i przyspieszyć publikację bezpiecznych opinii.

Kryteria akceptacji:
- Gdy tenant używa trybu MODERATION_AI, każda nowa opinia jest analizowana przez OpenAI API
- System wykrywa: wulgaryzmy, mowę nienawiści, dane osobowe, treści erotyczne, seksualne
- Bezpieczne opinie automatycznie otrzymują status APPROVED
- Podejrzane opinie otrzymują status VERIFICATION
- Wynik analizy jest zapisywany w polach classificationScore i classificationReason
- Proces analizy nie blokuje odpowiedzi API (może być asynchroniczny)

US-021: Automatyczne zatwierdzanie bezpiecznych opinii
Jako system, chcę automatycznie zatwierdzać opinie ocenione jako bezpieczne przez AI, aby były natychmiast widoczne dla klientów bez konieczności ręcznej moderacji.

Kryteria akceptacji:
- Opinie ocenione przez AI jako bezpieczne automatycznie otrzymują status APPROVED
- Są natychmiast widoczne w endpointach publicznych (GET /products/{id}/reviews)
- Są uwzględniane w podsumowaniach (GET /products/{id}/reviews/summary)
- classificationScore i classificationReason są zapisane dla celów audytowych
- Proces jest automatyczny i nie wymaga interwencji człowieka

US-022: Kierowanie podejrzanych opinii do ręcznej weryfikacji
Jako system, chcę automatycznie kierować opinie ocenione jako podejrzane do kolejki moderacji, aby moderator mógł podjąć ostateczną decyzję.

Kryteria akceptacji:
- Opinie ocenione przez AI jako podejrzane otrzymują status VERIFICATION
- Są widoczne w kolejce moderacji (GET /reviews/queue)
- classificationScore i classificationReason są dostępne dla moderatora
- Opinie nie są widoczne publicznie do czasu ręcznej weryfikacji
- Moderator może je zatwierdzić lub odrzucić

US-023: Fail-safe przy awarii OpenAI API
Jako system, chcę bezpiecznie obsłużyć sytuację, gdy OpenAI API jest niedostępne, aby system kontynuował działanie bez przerwy w usłudze.

Kryteria akceptacji:
- Gdy OpenAI API nie odpowiada lub zwraca błąd, system kontynuuje działanie
- Opinie, które nie mogły być przeanalizowane, domyślnie otrzymują status VERIFICATION
- System loguje błąd komunikacji z OpenAI API
- Opinie są kierowane do ręcznej moderacji jako zabezpieczenie
- Dodawanie opinii nie jest blokowane przez problemy z OpenAI API

US-024: Automatyczne zatwierdzanie wszystkich opinii w trybie ALLOW_ALL
Jako system, chcę automatycznie zatwierdzać wszystkie opinie dla tenantów w trybie ALLOW_ALL, aby opinie były natychmiast widoczne bez moderacji.

Kryteria akceptacji:
- Gdy tenant używa trybu ALLOW_ALL, wszystkie nowe opinie automatycznie otrzymują status APPROVED
- Nie jest wykonywana analiza AI
- Opinie są natychmiast widoczne publicznie
- Proces dodawania opinii jest szybki (brak opóźnienia na analizę AI)

US-025: Ręczna moderacja wszystkich opinii w trybie MODERATION_MANUAL
Jako system, chcę kierować wszystkie opinie do ręcznej moderacji dla tenantów w trybie MODERATION_MANUAL, aby moderator miał pełną kontrolę nad publikowanymi treściami.

Kryteria akceptacji:
- Gdy tenant używa trybu MODERATION_MANUAL, wszystkie nowe opinie otrzymują status PENDING
- Nie jest wykonywana analiza AI
- Opinie trafiają do kolejki moderacji (GET /reviews/queue)
- Opinie nie są widoczne publicznie do czasu zatwierdzenia przez moderatora

### 5.6 Bezpieczeństwo i uwierzytelnianie

US-026: Uwierzytelnianie żądań API przez Basic Auth
Jako system, chcę wymagać uwierzytelnienia Basic Auth dla wszystkich żądań API, aby chronić dostęp do danych przed nieautoryzowanymi użytkownikami.

Kryteria akceptacji:
- Wszystkie endpointy API wymagają poprawnego nagłówka Authorization z Basic Auth
- Statyczny klucz (secret) jest konfigurowany per środowisko
- System zwraca błąd 401 Unauthorized dla żądań bez poprawnego uwierzytelnienia
- Klucz jest przechowywany w usłudze typu vault (nie w kodzie ani w repozytorium)
- Różne środowiska (dev, staging, production) używają różnych kluczy

US-027: Identyfikacja tenanta przez nagłówek X-Account
Jako system, chcę identyfikować tenant na podstawie nagłówka HTTP X-Account, aby zapewnić izolację danych między sklepami.

Kryteria akceptacji:
- Każde żądanie API musi zawierać nagłówek X-Account z unikalnym kluczem tekstowym tenanta
- System zwraca błąd 400 Bad Request dla żądań bez nagłówka X-Account
- System zwraca błąd 404 Not Found dla nieistniejącego tenanta
- Wszystkie operacje (dodawanie, pobieranie, moderacja) są izolowane per tenant
- Tenant nie ma dostępu do danych innych tenantów

US-028: Izolacja danych między tenantami
Jako system, chcę zapewnić pełną izolację danych między tenantami, aby żaden sklep nie miał dostępu do opinii innych sklepów.

Kryteria akceptacji:
- Opinie są zawsze filtrowane według tenanta
- GET /products/{id}/reviews zwraca tylko opinie należące do tenanta z nagłówka X-Account
- GET /reviews/queue zwraca tylko opinie należące do tenanta
- PATCH /reviews/{id}/status działa tylko dla opinii należących do tenanta
- Próba dostępu do opinii innego tenanta zwraca błąd 404 Not Found (nie 403, aby nie ujawniać istnienia danych)

US-029: Audyt i zgodność z RODO przez soft delete
Jako system, chcę implementować soft delete dla opinii, aby zachować dane dla celów audytowych przy jednoczesnej zgodności z RODO.

Kryteria akceptacji:
- Usunięcie opinii ustawia pole deletedAt na aktualny timestamp
- Usunięte opinie nie są zwracane przez publiczne endpointy API
- Usunięte opinie są zachowane w bazie danych
- System wspiera faktyczne usunięcie danych (hard delete) w odpowiedzi na żądanie RODO
- Istnieje procedura dla faktycznego usunięcia danych użytkownika zgodnie z RODO

US-030: Bezpieczne przechowywanie poufnych danych
Jako system, chcę bezpiecznie przechowywać klucze API i inne poufne dane, aby minimalizować ryzyko wycieku danych.

Kryteria akceptacji:
- Klucz Basic Auth jest przechowywany w usłudze typu vault (HashiCorp Vault)
- Klucz OpenAI API jest przechowywany w vault
- Klucze nie są przechowywane w kodzie ani w repozytorium
- Klucze są ładowane w czasie uruchomienia aplikacji
- Dostęp do vault jest ograniczony i logowany

## 6. Metryki sukcesu

### 6.1 Metryki funkcjonalne

MS-F-001: Działające REST API
- Wszystkie zdefiniowane endpointy API są dostępne i działają zgodnie ze specyfikacją
- Testy integracyjne potwierdzają poprawność działania każdego endpointu
- API zwraca odpowiednie kody statusu HTTP dla scenariuszy sukcesu i błędów
- Dokumentacja API jest dostępna (np. OpenAPI/Swagger)

MS-F-002: Poprawna moderacja AI
- System AI poprawnie klasyfikuje opinie jako bezpieczne lub podejrzane
- Opinie bezpieczne otrzymują status APPROVED
- Opinie podejrzane otrzymują status VERIFICATION
- classificationScore i classificationReason są poprawnie zapisywane
- Fail-safe działa poprawnie przy awarii OpenAI API

MS-F-003: Działający dashboard administratora
- Dashboard prezentuje poprawne dane zagregowane
- Wszystkie widgety i statystyki działają zgodnie ze specyfikacją
- Dashboard jest dostępny i responsywny
- Dane na dashboardzie są aktualne

MS-F-004: Izolacja danych między tenantami
- Testy potwierdzają, że tenant nie ma dostępu do danych innych tenantów
- Wszystkie operacje są poprawnie filtrowane według X-Account
- Brak wycieków danych między tenantami w testach penetracyjnych

### 6.2 Metryki niefunkcjonalne

MS-NF-001: Bezpieczeństwo
- API jest chronione Basic Auth
- Wszystkie poufne dane są przechowywane w vault
- Brak krytycznych podatności w skanowaniu bezpieczeństwa
- Zgodność z OWASP Top 10
- Komunikacja przez HTTPS

MS-NF-002: Skalowalność
- System obsługuje co najmniej 500 tenantów
- System obsługuje co najmniej 10,000 opinii na tenanta
- System obsługuje co najmniej 100 żądań API na sekundę
- Architektura umożliwia skalowanie horyzontalne (dodawanie instancji)
- Baza danych jest odpowiednio zindeksowana

MS-NF-003: Wydajność
- Czas odpowiedzi GET /products/{id}/reviews: < 200ms dla 1000 opinii
- Czas odpowiedzi GET /products/{id}/reviews/summary: < 100ms
- Czas odpowiedzi POST /reviews: < 300ms (bez analizy AI)
- Czas odpowiedzi POST /reviews: < 2000ms (z analizą AI)
- Dashboard ładuje się w < 2s

MS-NF-004: Dostępność
- SLA 99.5% uptime dla API
- System działa nawet przy awarii OpenAI API (fail-safe)
- Monitoring i alerty są skonfigurowane
- Procedury odtwarzania po awarii są zdefiniowane

MS-NF-005: Zgodność z RODO
- Implementacja soft delete
- Możliwość faktycznego usunięcia danych użytkownika
- Minimalizacja przechowywania danych osobowych
- Opcjonalne pole author (anonimizacja możliwa)
- Logowanie operacji dla celów audytowych

### 6.3 Metryki biznesowe

MS-B-001: Skuteczność moderacji AI
- Co najmniej 70% opinii w trybie MODERATION_AI jest automatycznie zatwierdzanych (status APPROVED)
- Maksymalnie 30% opinii trafia do ręcznej weryfikacji (status VERIFICATION)
- Spośród opinii zweryfikowanych ręcznie: co najmniej 60% jest akceptowanych (co potwierdza, że AI nie jest zbyt restrykcyjne)
- Widoczne na dashboardzie administratora w widgecie skuteczności AI

MS-B-002: Redukcja obciążenia moderatorów
- Tryb MODERATION_AI redukuje liczbę opinii do ręcznej moderacji o co najmniej 70% w porównaniu z trybem MODERATION_MANUAL
- Czas poświęcony na moderację przez administratorów sklepów jest znacząco niższy

MS-B-003: Czas publikacji opinii
- Opinie bezpieczne w trybie MODERATION_AI są publikowane natychmiast (< 2s od dodania)
- Opinie w trybie ALLOW_ALL są publikowane natychmiast
- Średni czas publikacji opinii w trybie MODERATION_AI: < 1 godzina (uwzględniając ręczną weryfikację)

MS-B-004: Adopcja systemu
- Co najmniej 80% tenantów korzysta z systemu opinii w ciągu 3 miesięcy od wdrożenia
- Średnio co najmniej 50 opinii na tenanta w ciągu pierwszego miesiąca
- Co najmniej 50% tenantów wybiera tryb MODERATION_AI

MS-B-005: Jakość opinii
- Mniej niż 5% zatwierdzonych opinii jest raportowanych jako problematyczne przez użytkowników końcowych
- Mniej niż 1% opinii jest usuwanych post-factum przez administratorów sklepów
- Wysoki poziom satysfakcji administratorów sklepów z jakości moderacji AI (pomiar przez ankiety)

### 6.4 Metryki techniczne (monitorowanie)

MS-T-001: Metryki API
- Liczba żądań API per endpoint per tenant
- Średni czas odpowiedzi per endpoint
- Liczba błędów (4xx, 5xx) per endpoint
- Rate limiting i throttling (jeśli zaimplementowane)

MS-T-002: Metryki moderacji
- Liczba opinii per status (PENDING, VERIFICATION, APPROVED, REJECTED)
- Liczba opinii przeanalizowanych przez AI per tenant
- Średni czas analizy AI
- Liczba błędów komunikacji z OpenAI API

MS-T-003: Metryki bazy danych
- Rozmiar bazy danych
- Czas wykonania kluczowych zapytań
- Wykorzystanie indeksów
- Liczba połączeń do bazy danych

MS-T-004: Metryki biznesowe (dashboardowe)
- Liczba tenantów w systemie
- Liczba opinii per tenant
- TOP 10 tenantów według aktywności
- Skuteczność moderacji AI (automatyczne zatwierdzenia vs. weryfikacje)

MS-T-005: Metryki bezpieczeństwa
- Liczba nieudanych prób uwierzytelnienia
- Próby dostępu do danych innych tenantów
- Anomalie w ruchu API
- Skany bezpieczeństwa i znalezione podatności
