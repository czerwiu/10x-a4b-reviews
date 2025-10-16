## Podsumowanie uzgodnień dla PRD (MVP: a4b-reviews)

**Architektura i dostęp**

* Mikroserwis wewnętrzny, konsumowany wyłącznie przez platformę e-commerce (SaaS).
* Multitenant; identyfikacja tenanta: nagłówek `X-Account` (np. „tomex”, „handel-b2b”).
* Autentykacja: Basic Auth, jeden secret per środowisko, trzymany w Vault.
* Globalne endpointy dla dashboardu (zbiorcze, między-tenantowe) **bez** `X-Account`, nadal chronione Basic Auth.

**Model danych (rdzeń)**

* Review: `id`, `tenant`, `productId (Long)`, `variantId (opcjonalny/NULL)`, `userId` (wymagany), `author` (opcjonalny; gdy puste → „Anonimowy”), `rating` (1–5, całkowite), `content` (≤3000, bez HTML/URL), `status` (`PENDING`/`APPROVED`/`REJECTED`), `language` (ISO 639-1, `NULL` w MVP), `metadata JSONB`, `media JSONB` (na przyszłość), `createdAt`, `updatedAt`, `deletedAt (soft delete)`.
* Unikalność: **jedna opinia per produkt per użytkownik** — indeks warunkowy
  `UNIQUE(tenant, productId, userId) WHERE deletedAt IS NULL`.
* `variantId` przechowywany, ale nie wpływa na unikalność (opinie liczone „per produkt”).

**Walidacje i formaty**

* `author` ≤ 255 znaków; `content` ≤ 3000 znaków; zakaz HTML/URL → 400 `INVALID_CONTENT`.
* Czas: UTC, ISO 8601.
* Język (`language`) ignorowany funkcjonalnie w MVP (pole tylko w modelu).

**Moderacja i usuwanie**

* Tryb per tenant (w JSON konfiguracyjnym):
  • bez moderacji → tworzone od razu `APPROVED`
  • z moderacją → `PENDING → APPROVED/REJECTED`.
* Dozwolona zmiana `REJECTED → APPROVED` (bez przechowywania powodu).
* Usuwanie: soft delete (`deletedAt`); elementy usunięte nie wchodzą do agregatów.

**API – zakres i zachowanie**

* Tworzenie opinii: `POST /v1/reviews`
  • ignoruje status z payloadu; ustala serwer wg konfiguracji tenanta.
  • brak idempotencji; duplikat → 409 `REVIEW_ALREADY_EXISTS`.
* Odczyty **per produkt/variant** (w kontekście jednego tenanta via `X-Account`), **bez stronicowania**:
  • lista opinii z filtrem po ocenie (`rating`) i sortem (`date` lub `rating` ASC/DESC).
  • mapa ocen (1..5 → liczność; wartości 0 dla braków).
  • średnia ocena — wyłącznie z `APPROVED`.
* Globalne (dla dashboardu, między-tenantowe), **bez stronicowania**:
  • zestawienie: `tenant | #ALL | #PENDING | #APPROVED | #REJECTED | avgRating` w konfigurowalnym oknie czasu (domyślnie 7/30 dni) + sortowanie (po liczbie opinii / średniej).
  • top-10 tenantów po liczbie opinii (całościowo).
  • „ostatnie opinie” — jedyny wyjątek: kursor po `id`, od najnowszych, porcje po 50.
* Zmiana statusu / usuwanie (w kontekście tenanta przez `X-Account`), bez ról i kont per-tenant.

**Agregaty i wydajność**

* Agregaty liczone „on-the-fly” (bez cache); uwzględniają tylko `APPROVED` i pomijają soft-deleted.
* Brak metryk technicznych/Prometheus w MVP.

**Błędy i kontrakt**

* Kody: 401 (auth), 404 (not found), 409 (duplicate).
* Body błędu: `{ code, message }` (np. `INVALID_CONTENT`, `REVIEW_ALREADY_EXISTS`).

**Paginacja i sortowanie**

* Brak paginacji poza „ostatnimi opiniami” (kursor po `id`, 50 na stronę, malejąco od najnowszych).
* Dla list per produkt: tylko sort + filtry, bez stronicowania.

**Konfiguracja per tenant**

* Przechowywana jako JSON (elastyczna na przyszłe opcje); endpoint do odczytu/zapisu ustawień.
