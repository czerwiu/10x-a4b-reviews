# Aplikacja - a4b-reviews (MVP)

## Główny problem
Zbieranie i przechowywanie punktowych ocen oraz tekstowych opinii klientów o produktach zamówionych w sklepach internetowych 
założonych na platformie e-commerce funkcjonującej w modelu SaaS. Aplikacja multitenantowa, obsługująca wiele sklepów internetowych (każdy sklep to osobny tenant). 

## Najmniejszy zestaw funkcjonalności
- Możliwość dodania oceny formie punktowej (1-5) i opinii tekstowej do zamówionych produktów przez klientów sklepów internetowych za pomocą REST API
- Przechowywanie ocen i opinii w bazie danych
- Możliwość pobrania ocen i opinii dla konkretnego produktu lub wariantu produktu za pomocą REST API
- Możliwość pobrania średniej oceny i liczby opinii dla konkretnego produktu lub wariantu produktu za pomocą REST API
- Możliwość moderacji opinii (zatwierdzanie, odrzucanie) przez administratorów sklepów za pomocą REST API
- Wykorzystanie AI do analizy sentymentu opinii i zgodności pod kątem reguł moderacji (np. wykrywanie wulgaryzmów) i odpowiednie oznaczanie podejrzanych opinii
- Autoryzacja dostępu do API za pomocą Basic Auth z wykorzystaniem statycznego
- Interfejs webowy/dashboard prezentujący statystyki ocen i opinii dla administratora systemu (wykorzystanie systemu) dostępny po zalogowaniu przez administratora systemu

## Co NIE wchodzi w zakres MVP
- Import opinii z zewnętrznych źródeł
- Interfejs webowy do zarządzania opiniami dla administratorów sklepów i administratora systemu
- Dodawanie zdjęć i filmów do opinii
- Integracja z zewnętrznymi systemami opinii (np. Trusted Shops, Google Reviews)

## Kryteria sukcesu
Wydajny, skalowalny, bezpieczny, spełniający dobre praktyki i zalecenia związane z bezpieczeństwem danych i RODO system do zbierania i przechowywania opinii klientów, 
który umożliwia łatwe dodawanie i pobieranie opinii za pomocą REST API dla wielu sklepów uruchomionych w platformie e-commerce SaaS. 