# Moduł ocen i opinii produktowych – wymagania

## MUST HAVE (podstawowe)

* **Ocena gwiazdkowa** (np. 1–5) z wyliczaniem średniej i liczby ocen (globalnie i per-wariant).
* **Opinie tekstowe** z tytułem, treścią i datą publikacji.
* **Weryfikacja zakupu** (znacznik „Potwierdzony zakup”).
* **Moderacja**: kolejka do akceptacji/odrzucenia, zgłaszanie nadużyć, czarna lista słów.
* **Zasady publikacji**: minimalna/dopuszczalna długość, zakaz linków, RODO/zgoda na publikację.
* **Filtrowanie i sortowanie** (najwyżej oceniane, najnowsze, z/bez zdjęć, tylko potwierdzone).
* **Agregacja po atrybutach** (rozmiar, kolor) i **paginacja**.
* **Przydatność opinii** („Czy ta opinia była pomocna?” – głosy +1/–1, z limitem na użytkownika/IP).
* **Dodawanie opinii z poziomu konta i strony produktu** (formularz dostępny mobile/desktop).
* **API/SDK** do odczytu/zapisu (REST/GraphQL) + webhooks (np. „nowa opinia oczekuje”).
* **SEO**: dane strukturalne Schema.org (Product/Review/AggregateRating), przyjazne URL-e.
* **Międzynarodowość**: język opinii, strefy czasowe, formaty liczb.
* **Bezpieczeństwo/Anty-spam**: rate limiting, reCAPTCHA/hCaptcha, weryfikacja e-mail.
* **Panel admina**: lista opinii, filtry, masowe akcje, historia moderacji, eksport (CSV).
* **Zgodność prawna**: informacja o weryfikacji pochodzenia opinii, polityka publikacji, anonimizacja danych.

## NICE TO HAVE (warto mieć)

* **Multimedia w opinii**: zdjęcia/wideo, kompresja/miniatury, moderacja obrazów.
* **Pytania i odpowiedzi (Q&A)** przy produkcie (od klientów i od sprzedawcy).
* **Tagi treści** (np. „zgodność z opisem”, „jakość”, „rozmiar zawyżony/zaniżony”).
* **Wersje językowe** z automatycznym wykrywaniem języka i przełącznikiem.
* **Powiadomienia**: e-mail/SMS/web push o odpowiedzi na opinię czy statusie moderacji.
* **Zachęty/oprogramowanie lojalnościowe**: punkty za recenzje (z zabezpieczeniami anty-fraud).
* **Ankiety po zakupie** (post-purchase survey) z automatyczną prośbą o opinię (X dni po dostawie).
* **Wyróżnienia**: odznaki autora („Ekspert”, „Top reviewer”), pinowanie opinii.
* **Porównanie nastroju** (pozytywne/neutralne/negatywne) i wykres trendu ocen.
* **Widgety**: karuzele opinii, „testimonial bar”, widżet oceny sklepu (site-wide).
* **Import/eksport** z marketplace’ów i systemów zewnętrznych (CSV/JSON, np. Google, Allegro).
* **Zasady per kategorię/markę** (np. wymagana długość, wymóg zdjęcia).
* **Widok „o autorze”** z historią jego opinii (z poszanowaniem prywatności).
* **A/B testy** rozmieszczenia, długości teaserów, kolejności sortowania.
* **Integracja z CRM/Helpdesk** (eskalacja złych opinii do wsparcia).
* **Panel marki/dostawcy** (jeśli marketplace) do odpowiadania na opinie o własnych produktach.

## EXCLUSIVE (wyróżniki/zaawansowane)

* **Analiza NLP/AI**: automatyczne podsumowania opinii („Co klienci chwalą/krytykują”), ekstrakcja cech produktu i aspektowe oceny (np. „jakość 4.7, rozmiar 3.8”).
* **Rekomendacje oparte na opiniach**: „Klienci o profilu podobnym do Ciebie ocenili…”.
* **Detekcja fałszywych opinii** (sygnatury behawioralne, sieci powiązań, anomalia czasu/IP/urządzenia).
* **Personalizacja listy opinii** (najpierw te istotne dla danego użytkownika/regionu/rozmiaru).
* **Generowanie odpowiedzi szkicowych** dla moderatora (asystent AI zgodny z tonem marki).
* **Mapy nastrojów w czasie** z alertami (nagły spadek oceny po nowej partii produktu).
* **Zgodność z wytycznymi platform reklamowych** (Google Seller Ratings, Product Ratings feed).
* **Moderacja obrazów AI** (wykrywanie treści nieodpowiednich, OCR do wyłapywania danych wrażliwych).
* **Tryb B2B**: opinie techniczne, pola niestandardowe (MTBF, kompatybilność), waga recenzenta.
* **Gwarancja przejrzystości**: dziennik audytowy publiczny (hash łańcuchowy) zmian/statusów opinii.
* **Asystent rozmiaru** trenowany na treści opinii (fashion/obuwie – rozkład „true-to-size”).
* **Eksport do „voice of customer hub”** (CDP/warehouse: BigQuery/Snowflake) z eventami.

---

### Uwagi implementacyjne (skrót)

* **Model danych**: `Review`, `Rating`, `Media`, `Vote`, `Flag`, `PurchaseVerification`, `Reply`, `AspectScore`.
* **Uprawnienia**: klient, moderator, marka/dostawca, admin; log audytu każdej zmiany.
* **Wydajność**: cache’owana średnia ocena per produkt/wariant, indeksy po `productId`, `verified`, `createdAt`.
* **Dostępność/UX**: WCAG 2.1 AA (klawiatura, ARIA, kontrast), lazy-load mediów, skeletony.
* **Mierniki (KPI)**: udział produktów z ≥1 opinią, średnia ocena, odsetek zweryfikowanych opinii, CTR „Helpful”, czas do odpowiedzi na negatywne.

Chcesz, żebym przerobił to na listę wymagań użytkowych (user stories) albo checklistę do RFP?
