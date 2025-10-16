# Analiza Stacku Technologicznego - a4b-reviews

## Spis treści
1. [Specyfikacja Stacku](#specyfikacja-stacku)
2. [Analiza Dopasowania do Wymagań](#analiza-dopasowania-do-wymagań)
3. [Mocne Strony](#mocne-strony)
4. [Słabe Strony i Ryzyka](#słabe-strony-i-ryzyka)
5. [Alternatywy](#alternatywy)
6. [Architektura Techniczna](#architektura-techniczna)
7. [Rekomendacje Implementacyjne](#rekomendacje-implementacyjne)
8. [Werdykt Finalny](#werdykt-finalny)

---

## Specyfikacja Stacku

### Backend Stack

| Kategoria | Technologia | Wersja | Uzasadnienie |
|-----------|-------------|--------|--------------|
| Język | Java | 25 (LTS) | Najnowszy LTS, Virtual Threads, Pattern Matching, wsparcie do ~2033 |
| Framework | Spring Boot | 3.5.6+ | Oficjalne wsparcie Java 25, ekosystem, Spring Data JPA, Spring Security |
| Build Tool | Maven | 3.6.3+ | Oficjalnie wspierany przez Spring Boot, znany przez zespół |
| ORM | Spring Data JPA | (w Spring Boot) | Abstrakcja nad Hibernate, repository pattern |
| Persistence | Hibernate | (z JPA) | Dojrzały, wsparcie dla JSONB (PostgreSQL) |
| Security | Spring Security | (w Spring Boot) | Basic Auth, integracja z Spring Boot |
| Validation | Bean Validation | (Hibernate Validator) | Deklaratywna walidacja, JSR-380 |
| API Docs | SpringDoc OpenAPI | 2.x | Automatyczna generacja Swagger UI z annotacji |
| HTTP Client | Spring WebClient | (w Spring Boot) | Reaktywny klient dla OpenAI API calls |
| JSON | Jackson | (w Spring Boot) | Domyślny w Spring Boot, wsparcie dla JSON/JSONB |

### Database Stack

| Kategoria | Technologia | Wersja | Uzasadnienie |
|-----------|-------------|--------|--------------|
| RDBMS | PostgreSQL | 15+ | JSONB support, multi-tenant, ACID, dojrzałość |
| Migracje | Flyway | Latest | Wersjonowanie schematu, rollback capability |
| Connection Pool | HikariCP | (w Spring Boot) | Domyślny w Spring Boot, najszybszy pool |

### Frontend Stack

| Kategoria | Technologia | Wersja | Uzasadnienie |
|-----------|-------------|--------|--------------|
| Framework | React | 18+ | Komponenty, hooks, duże wsparcie społeczności |
| Język | TypeScript | 5.x | Type safety dla API calls, lepsze DX |
| Build Tool | Vite | 5.x | Szybszy niż CRA, HMR, nowoczesny |
| UI Library | Ant Design | 5.x | Gotowe komponenty dla dashboardów (Table, Statistic, Card) |
| Charts | Recharts | 2.x | React-native, deklaratywne wykresy kołowe |
| HTTP Client | Axios / Fetch API | - | Do komunikacji z backend API |
| State Management | React Context / Zustand | - | Prosty stan dla dashboardu (filtry, paginacja) |
| Routing | React Router | 6.x | Jeśli dashboard będzie miał podstrony |

### Infrastructure & DevOps

| Kategoria | Technologia | Uzasadnienie |
|-----------|-------------|--------------|
| Orchestration | Kubernetes | Skalowanie, self-healing, deklaratywna konfiguracja |
| Cloud Provider | DigitalOcean | Managed Kubernetes (DOKS), prostota, koszty |
| Containerization | Docker | Standard dla Kubernetes deployments |
| Secrets Management | HashiCorp Vault | Bezpieczne przechowywanie secrets (Basic Auth, OpenAI API key) |
| Reverse Proxy | Nginx / Ingress | Routing, TLS termination, load balancing |

### External Services

| Kategoria | Technologia | Uzasadnienie |
|-----------|-------------|--------------|
| AI Moderation | OpenAI API | GPT-4 lub moderation endpoint do analizy treści |

### Monitoring & Observability (Rekomendowane)

| Kategoria | Technologia | Uzasadnienie |
|-----------|-------------|--------------|
| Metrics | Prometheus + Grafana | Open-source, Kubernetes-native |
| Logging | ELK Stack / Loki | Centralizacja logów, analiza |
| APM | Spring Boot Actuator | Health checks, metrics endpoint |
| Tracing | Spring Cloud Sleuth + Zipkin (opcjonalnie) | Distributed tracing dla debugowania |

---

## Analiza Dopasowania do Wymagań

### Wymagania Funkcjonalne vs Stack

| Wymaganie PRD | Technologia | Dopasowanie | Komentarz |
|---------------|-------------|-------------|-----------|
| FR-001: REST API do dodawania opinii | Spring Web + Bean Validation | ⭐⭐⭐⭐⭐ | `@RestController`, `@Valid`, `@PostMapping` - naturalne dla Spring |
| FR-002: Multi-tenant data isolation | PostgreSQL + JPA | ⭐⭐⭐⭐⭐ | Discriminator column w entity, `@Where`, `@Filter` |
| FR-003-006: Pobieranie i filtrowanie opinii | Spring Data JPA | ⭐⭐⭐⭐⭐ | Query methods, Specifications, native queries |
| FR-007-011: Moderacja i statusy | JPA Enums + State Machine | ⭐⭐⭐⭐⭐ | `@Enumerated`, walidacja przejść stanów |
| FR-008: Integracja OpenAI API | WebClient | ⭐⭐⭐⭐⭐ | Async calls, reactive, timeout handling |
| FR-012-014: Basic Auth + X-Account | Spring Security | ⭐⭐⭐⭐⭐ | `BasicAuthenticationFilter`, custom `@TenantId` resolver |
| FR-015-017: Dashboard administratora | React + Ant Design + Recharts | ⭐⭐⭐⭐⭐ | Table z sortowaniem OOTB, Statistic, PieChart |
| FR-011: Soft delete | JPA + @Where | ⭐⭐⭐⭐⭐ | `deletedAt` field, `@Where(clause = "deleted_at IS NULL")` |

### Wymagania Niefunkcjonalne vs Stack

| Wymaganie | Stack Decision | Dopasowanie | Komentarz |
|-----------|----------------|-------------|-----------|
| NF-001: Bezpieczeństwo | Spring Security + Vault + HTTPS | ⭐⭐⭐⭐⭐ | Industry standard, OWASP compliant |
| NF-002: Skalowalność (100 tenantów, 10k opinii) | Kubernetes + PostgreSQL | ⭐⭐⭐⭐⭐ | Horizontal scaling, DB indexing |
| NF-003: Wydajność (< 200ms response) | Java 25 Virtual Threads + indexes | ⭐⭐⭐⭐⭐ | Virtual Threads idealne dla I/O-heavy |
| NF-004: Dostępność (99.5% SLA) | Kubernetes + DigitalOcean DOKS | ⭐⭐⭐⭐ | K8s self-healing, DOKS managed control plane |
| NF-005: RODO compliance | Soft delete + audit logs | ⭐⭐⭐⭐⭐ | `deletedAt`, Spring Data Envers (opcja) |

### Wymagania Zespołowe vs Stack

| Wymaganie | Stack Decision | Dopasowanie | Komentarz |
|-----------|----------------|-------------|-----------|
| Doświadczenie w Java + Spring Boot | Java 25 + Spring Boot 3.5.6 | ⭐⭐⭐⭐⭐ | Zespół pracuje w tej technologii |
| Doświadczenie w React | React 18 + TypeScript | ⭐⭐⭐⭐ | Część zespołu zna React |
| Brak doświadczenia w Tailwind | Ant Design (CSS-in-JS) | ⭐⭐⭐⭐⭐ | Zero learning curve dla Tailwind |
| Szybki time-to-market | Ant Design + Spring Boot Starter | ⭐⭐⭐⭐⭐ | Komponenty OOTB, auto-configuration |

---

## Mocne Strony

### 1. Backend: Java 25 + Spring Boot 3.5.6

MOCNE:
- ✅ Zespół ma duże doświadczenie - zero learning curve
- ✅ Virtual Threads (z Java 21) - idealne dla I/O-heavy operations (DB queries, OpenAI API calls)
- ✅ Spring Boot ecosystem - wszystko "just works" (Security, Data JPA, Validation, Actuator)
- ✅ Spring Data JPA - repository pattern, query methods, specifications dla złożonych filtrów
- ✅ Spring Security - Basic Auth out-of-the-box, łatwa konfiguracja
- ✅ Dojrzałość - battle-tested w enterprise, ogromna społeczność
- ✅ LTS do ~2033 - długie wsparcie
- ✅ Pattern Matching, Sealed Classes - czystszy kod dla state machine (statusy opinii)

PRZYKŁAD użycia Virtual Threads dla wydajności:
```java
// application.properties
spring.threads.virtual.enabled=true

// Każde żądanie HTTP będzie obsługiwane przez Virtual Thread
// Tysiące współbieżnych połączeń bez overhead'u thread pool'a
```

### 2. Database: PostgreSQL 15+

MOCNE:
- ✅ JSONB native support - idealne dla pól `metadata`, `media`, `classificationReason`
- ✅ ACID compliance - spójność danych przy moderacji (zmiana statusów)
- ✅ Partial indexes - wydajność dla `WHERE deleted_at IS NULL`
- ✅ JSON operators - filtrowanie i agregacje na JSONB bez ORM gymnastics
- ✅ Multi-tenant support - tenant_id jako discriminator, row-level security (opcja)
- ✅ Full-text search - przyszłość: wyszukiwanie w treści opinii
- ✅ Mature tooling - PgAdmin, psql, monitoring extensions
- ✅ DigitalOcean Managed PostgreSQL - automatyczne backupy, HA, scaling

PRZYKŁAD indeksowania dla wydajności:
```sql
-- Partial index dla aktywnych opinii
CREATE INDEX idx_reviews_product_approved
ON reviews (product_id, created_at DESC)
WHERE status = 'APPROVED' AND deleted_at IS NULL;

-- JSONB GIN index dla metadata
CREATE INDEX idx_reviews_metadata ON reviews USING GIN (metadata);
```

### 3. Frontend: React 18 + Ant Design 5 + Recharts

MOCNE:
- ✅ Ant Design Table z sortowaniem OOTB - zero pracy dla tabel z sortowaniem
- ✅ Ant Design Statistic - gotowe portlety informacyjne
- ✅ Ant Design RangePicker - filtrowanie dat (tydzień, miesiąc, rok) out-of-the-box
- ✅ Recharts - deklaratywne wykresy kołowe, React-native API
- ✅ TypeScript - type safety dla API contracts, mniej błędów runtime
- ✅ Vite - szybki dev server, HMR, nowoczesny build
- ✅ Komponent ecosystem - setki gotowych rozwiązań dla React
- ✅ Część zespołu zna React - mniejsza krzywa uczenia niż full-stack nowa technologia

PRZYKŁAD prostoty Ant Design:
```tsx
<Table
  dataSource={reviews}
  columns={[
    { title: 'Tenant', dataIndex: 'tenant', sorter: (a, b) => a.tenant.localeCompare(b.tenant) },
    { title: 'Rating', dataIndex: 'rating', sorter: (a, b) => a.rating - b.rating }
  ]}
/>
// Sortowanie działa od razu, bez state management
```

### 4. Infrastructure: Kubernetes + DigitalOcean

MOCNE:
- ✅ Kubernetes - horizontal scaling (dodawanie pod'ów przy obciążeniu)
- ✅ Self-healing - automatyczny restart failed containers
- ✅ Deklaratywna konfiguracja - Infrastructure as Code (YAML manifests)
- ✅ DigitalOcean DOKS - managed control plane, prostszy niż AWS EKS
- ✅ DigitalOcean Managed PostgreSQL - automatyczne backupy, HA, monitoring
- ✅ Prostota vs AWS - mniej skomplikowane IAM/VPC, niższe koszty dla MVP
- ✅ Load Balancer + Ingress - automatyczny routing, TLS termination

### 5. Secrets Management: HashiCorp Vault

MOCNE:
- ✅ Industry standard dla secrets management
- ✅ Centralizacja secrets (Basic Auth secret, OpenAI API key, DB credentials)
- ✅ Audit logs - kto i kiedy pobrał secret
- ✅ Dynamic secrets - możliwość rotacji bez redeployment
- ✅ Kubernetes integration - Vault Agent Injector dla automatycznego inject secrets do pod'ów

### 6. API Documentation: SpringDoc OpenAPI

MOCNE:
- ✅ Automatyczna generacja dokumentacji z annotacji
- ✅ Swagger UI out-of-the-box - interaktywna dokumentacja
- ✅ OpenAPI 3.0 spec - standard dla REST API
- ✅ Zero maintenance - dokumentacja aktualizuje się z kodem

---

## Słabe Strony i Ryzyka

### 1. Brak Cache Layer

SŁABE:
- ⚠️ Dashboard metrics (liczba tenantów, TOP 10, widget AI) - każde odświeżenie = query do DB
- ⚠️ GET /products/{id}/reviews/summary - agregacje bez cache mogą być wolne dla produktów z tysiącami opinii
- ⚠️ Bez cache OpenAI API responses - każda podobna opinia = nowe wywołanie API (koszty)

RYZYKO:
- Średnie dla MVP, Wysokie dla scale

MITIGACJA:
```
MVP: Zaakceptować (proste query optimization + PostgreSQL indexes wystarczą)
Post-MVP: Dodać Redis dla:
  - Cache summary endpoints (TTL 5-15 min)
  - Cache dashboard metrics (TTL 1-5 min)
  - Cache OpenAI responses dla identycznych tekstów (content hash jako klucz)
```

DECYZJA:
✅ Akceptowalne dla MVP - dodanie Redis post-MVP jest trywialne (Spring Cache + @Cacheable)

### 2. Brak Message Broker / Queue

SŁABE:
- ⚠️ OpenAI API call w trybie MODERATION_AI - synchroniczny w request path
- ⚠️ POST /reviews może trwać 1-2s zamiast < 300ms (czekanie na OpenAI response)
- ⚠️ Brak retry logic - jeśli OpenAI timeout, fail-safe do VERIFICATION (OK, ale można lepiej)
- ⚠️ Brak rate limiting control - burst of reviews = burst of OpenAI calls (koszty)

RYZYKO:
- Średnie dla MVP, Wysokie dla scale

MITIGACJA:
```
MVP: Zaakceptować + implementować:
  - WebClient timeout (2s max)
  - Fail-safe do VERIFICATION przy błędzie
  - Prosty rate limiter dla OpenAI calls (np. Bucket4j)

Post-MVP: Dodać RabbitMQ/Redis Queue:
  - POST /reviews zwraca 201 natychmiast (status PENDING lub PROCESSING)
  - Worker async analizuje opinię przez OpenAI
  - Worker aktualizuje status (APPROVED lub VERIFICATION)
  - Retry logic w queue (3 próby z exponential backoff)
```

DECYZJA:
✅ Akceptowalne dla MVP - WebClient z timeout + fail-safe wystarczy
⚠️ Zaplanować refactor post-MVP na async processing

### 3. Java Memory Footprint

SŁABE:
- ⚠️ Java aplikacje zużywają więcej RAM niż Go/Rust/Node.js
- ⚠️ Kubernetes pod może potrzebować 512MB-1GB RAM na instancję
- ⚠️ Wyższe koszty infrastructure dla tej samej liczby requestów

RYZYKO:
- Niskie dla MVP (100 tenantów, 1M opinii rocznie = mało)

MITIGACJA:
```
- Java 25 ma lepszy GC (ZGC, Generational ZGC)
- Virtual Threads redukują memory pressure (vs thread-per-request)
- Kubernetes HPA (Horizontal Pod Autoscaler) - skalowanie tylko gdy potrzeba
- DigitalOcean droplets są tańsze niż AWS EC2
```

DECYZJA:
✅ Akceptowalne - doświadczenie zespołu > oszczędność 100-200 USD/miesiąc

### 4. Ant Design Bundle Size

SŁABE:
- ⚠️ Ant Design full bundle ~600KB (gzipped)
- ⚠️ Dashboard będzie ładował się wolniej niż z Tailwind (~200KB)
- ⚠️ Dla dashboardu wewnętrznego to NIE jest problem, ale warto wiedzieć

RYZYKO:
- Niskie (dashboard = internal tool, nie public-facing)

MITIGACJA:
```
- Tree shaking w Vite (importuj tylko używane komponenty)
- Code splitting dla różnych części dashboardu (jeśli będzie rósł)
- Lazy loading dla charts (Recharts ~100KB dodatkowe)
```

DECYZJA:
✅ Akceptowalne - dashboard to internal tool, nie SEO/performance-critical

### 5. Vendor Lock-in: DigitalOcean

SŁABE:
- ⚠️ DOKS (managed Kubernetes) - przeniesienie na AWS EKS/GCP GKE wymaga reconfiguration
- ⚠️ DigitalOcean Managed PostgreSQL - backup/restore, migration

RYZYKO:
- Niskie (Kubernetes manifests są przenośne, PostgreSQL dump/restore jest standardem)

MITIGACJA:
```
- Używać standardowych Kubernetes manifests (nie DigitalOcean-specific features)
- Trzymać Terraform/IaC dla infrastructure (łatwiejsza migracja)
- Regular backups PostgreSQL (pg_dump) przechowywane niezależnie od DO
```

DECYZJA:
✅ Akceptowalne - benefit (prostota DO) > ryzyko (teoretyczna migracja w przyszłości)

### 6. OpenAI API Dependency

SŁABE:
- ⚠️ External dependency - vendor outage = system degradation
- ⚠️ Koszty - każda opinia w trybie MODERATION_AI = API call ($$$)
- ⚠️ Rate limits - burst of reviews może przekroczyć OpenAI rate limit

RYZYKO:
- Średnie (fail-safe do VERIFICATION działa, ale UX gorszy)

MITIGACJA:
```
MVP:
  - Fail-safe do VERIFICATION (już zaplanowane w PRD)
  - OpenAI timeout 2s max
  - Circuit breaker pattern (po N failures, przestań próbować na X sekund)
  - Monitoring OpenAI API status

Post-MVP:
  - Cache OpenAI responses (content hash)
  - Implementacja własnego ML model (open-source, self-hosted) jako fallback
  - Rate limiting per tenant (nie pozwól jednemu tenantowi wyczerpać limitu)
```

DECYZJA:
✅ Akceptowalne dla MVP - fail-safe + monitoring wystarczą
⚠️ Zaplanować optymalizację kosztów post-MVP (cache, własny model)

### 7. Brak Experience z TypeScript w Zespole

SŁABE:
- ⚠️ TypeScript dla części zespołu może być nowy
- ⚠️ Krzywa uczenia dla type definitions, generics, utility types

RYZYKO:
- Niskie (TypeScript dla dashboardu to ~80% JavaScript + typy dla API)

MITIGACJA:
```
- Wygenerować TypeScript types z OpenAPI spec (openapi-generator)
- Używać prostych typów (interfaces dla API responses)
- Code review + pair programming dla nauki
```

DECYZJA:
✅ Akceptowalne - korzyści (type safety) > koszt (learning curve)
💡 Można rozważyć JavaScript zamiast TypeScript TYLKO jeśli zespół ma < 2 tygodni na frontend

---

## Alternatywy

### Backend Alternatives

| Alternatywa | Pros | Cons | Werdykt |
|-------------|------|------|---------|
| Go + Gin/Echo | ✅ Mniejszy footprint (50-100MB RAM)<br>✅ Szybka kompilacja<br>✅ Goroutines dla concurrency | ❌ Zespół nie zna Go<br>❌ Mniej batteries-included niż Spring<br>❌ Słabszy ORM (GORM) | ❌ Nie dla tego projektu (learning curve) |
| Node.js + NestJS | ✅ TypeScript full-stack<br>✅ Szybki prototyping<br>✅ Spring-like architecture | ❌ Zespół nie zna Node.js/NestJS<br>❌ Słabszy ORM niż Hibernate<br>❌ Single-threaded (clustering potrzebny) | ❌ Nie dla tego projektu (learning curve) |
| Python + FastAPI | ✅ Prosty syntax<br>✅ Świetny dla ML/AI (ale tu nie potrzeba)<br>✅ Auto OpenAPI docs | ❌ Zespół nie zna Python<br>❌ Wolniejszy niż Java/Go<br>❌ GIL (Global Interpreter Lock) | ❌ Nie dla tego projektu (learning curve) |
| Kotlin + Spring Boot | ✅ JVM (jak Java)<br>✅ Modern syntax<br>✅ Null safety | ⚠️ Zespół zna Javę, nie Kotlin<br>⚠️ Mniejsza społeczność niż Java | ⚠️ Możliwe, ale po co? (Java 25 jest świetna) |

WERDYKT: ✅ Java 25 + Spring Boot - zero learning curve, najlepszy fit

### Database Alternatives

| Alternatywa | Pros | Cons | Werdykt |
|-------------|------|------|---------|
| MySQL 8 | ✅ JSON support<br>✅ Popularność<br>✅ DigitalOcean managed | ❌ Słabszy JSON support niż PostgreSQL<br>❌ Brak partial indexes<br>❌ Słabszy dla agregacji | ❌ PostgreSQL lepszy dla tego projektu |
| MongoDB | ✅ Native JSON<br>✅ Flexible schema<br>✅ Horizontal scaling | ❌ Brak ACID transactions (starsze wersje)<br>❌ Over-engineering dla relacyjnych danych<br>❌ Zespół zna SQL, nie MongoDB | ❌ RDBMS lepszy dla opinii (relacje: tenant, product, user) |
| CockroachDB | ✅ PostgreSQL compatible<br>✅ Distributed<br>✅ Global scale | ❌ Over-engineering dla MVP (100 tenantów)<br>❌ Wyższe koszty<br>❌ Większa złożoność | ❌ Za dużo dla MVP, rozważyć dla scale |

WERDYKT: ✅ PostgreSQL 15+ - najlepszy balance feature/complexity

### Frontend Alternatives

| Alternatywa | Pros | Cons | Werdykt |
|-------------|------|------|---------|
| Vue 3 + Element Plus | ✅ Element Plus ma gotowe komponenty<br>✅ Prostsza krzywa uczenia niż React | ❌ Zespół zna React, nie Vue<br>❌ Mniejsza społeczność niż React | ❌ React + Ant lepszy (zespół zna React) |
| Angular + Material | ✅ Full framework<br>✅ TypeScript native<br>✅ Material Design | ❌ Zespół nie zna Angular<br>❌ Ciężki framework dla prostego dashboardu<br>❌ Stromość krzywej uczenia | ❌ Over-engineering dla dashboardu |
| Svelte + SvelteKit | ✅ Najmniejszy bundle<br>✅ Prosty syntax<br>✅ Reactive | ❌ Zespół nie zna Svelte<br>❌ Mniejszy ekosystem komponentów<br>❌ Mniej gotowców dla dashboardów | ❌ Nie dla tego projektu (learning curve) |
| Server-side (Thymeleaf/JSP) | ✅ Zero JavaScript framework<br>✅ Proste | ❌ Brak interaktywności (sortowanie, filtry)<br>❌ Full page reload dla każdej akcji<br>❌ Gorsze UX | ❌ Nie dla dashboardu z sortowaniem/filtrami |

WERDYKT: ✅ React 18 + Ant Design - zespół zna React, Ant ma gotowe komponenty

### Infrastructure Alternatives

| Alternatywa | Pros | Cons | Werdykt |
|-------------|------|------|---------|
| AWS EKS + RDS | ✅ Największy cloud provider<br>✅ Więcej managed services<br>✅ Global reach | ❌ Bardziej skomplikowany (IAM, VPC)<br>❌ Droższy niż DigitalOcean<br>❌ Over-engineering dla MVP | ⚠️ Rozważyć dla scale, nie dla MVP |
| Google Cloud GKE + CloudSQL | ✅ Dobry Kubernetes (GKE inventor)<br>✅ Dobry PostgreSQL managed | ❌ Droższy niż DO<br>❌ Bardziej skomplikowany niż DO | ⚠️ Możliwe, ale DO prostsze dla MVP |
| Heroku | ✅ Najprostsze deployment<br>✅ Zero DevOps | ❌ Vendor lock-in<br>❌ Droższy per unit compute<br>❌ Brak Kubernetes (mniejsza kontrola) | ❌ Nie dla projektu (potrzebujesz K8s experience) |
| Bare Metal VPS (Hetzner, OVH) | ✅ Najtańsze<br>✅ Pełna kontrola | ❌ Musisz zarządzać wszystkim (OS, K8s, DB)<br>❌ Brak managed services<br>❌ Więcej DevOps work | ❌ Za dużo pracy dla MVP |

WERDYKT: ✅ DigitalOcean DOKS - najlepszy balance prostota/koszty/kontrola

---

## Architektura Techniczna

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Internet / Clients                            │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                │ HTTPS
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              DigitalOcean Load Balancer (TLS)                    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (DOKS)                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Nginx Ingress Controller                 │ │
│  └────────┬──────────────────────────────────┬────────────────┘ │
│           │                                   │                  │
│           │ /api/*                           │ /                │
│           ▼                                   ▼                  │
│  ┌─────────────────┐                ┌──────────────────┐        │
│  │  Backend Pods   │                │  Frontend Pods   │        │
│  │  (Spring Boot)  │                │  (Nginx + React) │        │
│  │                 │                │                  │        │
│  │  ┌──────────┐   │                │  ┌────────────┐  │        │
│  │  │  Pod 1   │   │                │  │   Pod 1    │  │        │
│  │  └──────────┘   │                │  └────────────┘  │        │
│  │  ┌──────────┐   │                │  ┌────────────┐  │        │
│  │  │  Pod 2   │   │                │  │   Pod 2    │  │        │
│  │  └──────────┘   │                │  └────────────┘  │        │
│  │  ┌──────────┐   │                └──────────────────┘        │
│  │  │  Pod N   │   │                                             │
│  │  └──────────┘   │                                             │
│  └────────┬─────────┘                                            │
│           │                                                      │
└───────────┼──────────────────────────────────────────────────────┘
            │
            │ JDBC
            ▼
┌─────────────────────────────────────────────────────────────────┐
│       DigitalOcean Managed PostgreSQL (Primary + Standby)        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     HashiCorp Vault                              │
│  (Secrets: Basic Auth, OpenAI API Key, DB Credentials)          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       OpenAI API                                 │
│                  (External Service)                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                 Monitoring & Observability                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Prometheus  │  │   Grafana    │  │  ELK/Loki    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

```
CLIENT REQUEST FLOW (POST /reviews):
==================================

1. Client (e-commerce system)
   └─> HTTPS POST /api/reviews
       Headers: Authorization: Basic <secret>, X-Account: tomex
       Body: { userId, productId, rating, reviewText, orderId }

2. DigitalOcean Load Balancer
   └─> TLS termination
   └─> Forward to Nginx Ingress

3. Nginx Ingress
   └─> Route /api/* to Backend Service

4. Spring Boot Backend (Pod)
   └─> Spring Security: Validate Basic Auth (secret from Vault)
   └─> Custom Filter: Extract X-Account → resolve Tenant
   └─> Controller: @PostMapping("/reviews")
       └─> Validation: @Valid ReviewRequest
       └─> Service Layer:
           ├─> Check tenant moderation mode (ALLOW_ALL/MODERATION_AI/MODERATION_MANUAL)
           ├─> If MODERATION_AI:
           │   └─> WebClient call to OpenAI API (async, 2s timeout)
           │       ├─> Success: classificationScore, classificationReason
           │       │   └─> status = APPROVED (safe) or VERIFICATION (suspicious)
           │       └─> Failure/Timeout: status = VERIFICATION (fail-safe)
           ├─> If ALLOW_ALL: status = APPROVED
           ├─> If MODERATION_MANUAL: status = PENDING
           └─> Repository: reviewRepository.save(review)
               └─> PostgreSQL INSERT with tenant_id, status, etc.

5. Response to Client
   └─> 201 Created, Location: /api/reviews/{id}
       Body: { id, status, createdAt }

DATABASE QUERY FLOW (GET /products/{id}/reviews):
===============================================

1. Client → Load Balancer → Ingress → Backend Pod
2. Spring Boot:
   └─> Security: Validate Basic Auth + X-Account
   └─> Controller: @GetMapping("/products/{id}/reviews")
       └─> Params: rating (optional), sort (optional)
       └─> Service:
           └─> Repository: Custom query with Specifications
               SELECT * FROM reviews
               WHERE tenant_id = :tenantId
                 AND product_id = :productId
                 AND status = 'APPROVED'
                 AND deleted_at IS NULL
                 AND (:rating IS NULL OR rating = :rating)
               ORDER BY created_at DESC (or rating, depending on sort param)
   └─> Response: List<ReviewDTO>

DASHBOARD FLOW (Frontend → Backend):
===================================

1. Browser → Load Balancer → Ingress → Frontend Pod (Nginx serving React)
2. React App loads → Fetches data:
   └─> Axios GET /api/admin/stats
       └─> Backend: Aggregation queries
           ├─> Count tenants
           ├─> Count reviews per tenant
           ├─> TOP 10 tenants by review count
           ├─> Last 50 reviews (cursor pagination)
           └─> AI widget metrics:
               SELECT
                 COUNT(*) FILTER (WHERE status = 'APPROVED' AND classification_score IS NOT NULL) as auto_approved,
                 COUNT(*) FILTER (WHERE status = 'VERIFICATION') as to_verification,
                 COUNT(*) FILTER (WHERE status = 'APPROVED' AND previous_status = 'VERIFICATION') as verified_approved,
                 COUNT(*) FILTER (WHERE status = 'REJECTED') as verified_rejected
               FROM reviews
               WHERE created_at > :startDate
   └─> Ant Design Table, Statistic, Recharts PieChart render data
```

### Database Schema (PostgreSQL)

```sql
-- Tenants table
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_key VARCHAR(100) UNIQUE NOT NULL, -- 'tomex', 'shop123', etc.
    moderation_mode VARCHAR(20) NOT NULL, -- 'ALLOW_ALL', 'MODERATION_AI', 'MODERATION_MANUAL'
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Reviews table
CREATE TABLE reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),

    -- User & order info
    user_id VARCHAR(255) NOT NULL,
    author VARCHAR(255), -- Optional display name
    order_id VARCHAR(255) NOT NULL,

    -- Product info
    product_id VARCHAR(255) NOT NULL,
    variant_id VARCHAR(255), -- Optional

    -- Review content
    rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
    review_text TEXT NOT NULL,
    language VARCHAR(10), -- ISO 639-1, e.g., 'en', 'pl'

    -- Status & moderation
    status VARCHAR(20) NOT NULL, -- 'PENDING', 'VERIFICATION', 'APPROVED', 'REJECTED'
    classification_score DECIMAL(5,4), -- 0.0000 to 1.0000 from OpenAI
    classification_reason TEXT, -- Why flagged by AI

    -- Metadata
    metadata JSONB, -- Future use
    media JSONB, -- Future use for photos/videos

    -- Timestamps
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP, -- Soft delete

    -- Indexes
    INDEX idx_tenant_product (tenant_id, product_id),
    INDEX idx_tenant_variant (tenant_id, variant_id),
    INDEX idx_tenant_status (tenant_id, status),

    -- Partial index for active approved reviews (most common query)
    INDEX idx_active_approved (product_id, created_at DESC)
        WHERE status = 'APPROVED' AND deleted_at IS NULL,

    -- JSONB indexes
    INDEX idx_metadata_gin USING GIN (metadata),
    INDEX idx_media_gin USING GIN (media)
);

-- Flyway migration versioning
CREATE TABLE flyway_schema_history (
    installed_rank INT NOT NULL,
    version VARCHAR(50),
    description VARCHAR(200) NOT NULL,
    type VARCHAR(20) NOT NULL,
    script VARCHAR(1000) NOT NULL,
    checksum INT,
    installed_by VARCHAR(100) NOT NULL,
    installed_on TIMESTAMP NOT NULL DEFAULT NOW(),
    execution_time INT NOT NULL,
    success BOOLEAN NOT NULL,
    PRIMARY KEY (installed_rank)
);
```

---

## Rekomendacje Implementacyjne

### 1. Backend Implementation Best Practices

ARCHITEKTURA:
```
src/main/java/com/a4b/reviews/
├── config/
│   ├── SecurityConfig.java          // Spring Security: Basic Auth
│   ├── WebConfig.java               // CORS, tenant resolver
│   ├── OpenApiConfig.java           // Swagger configuration
│   └── OpenAiConfig.java            // WebClient for OpenAI API
├── controller/
│   ├── ReviewController.java        // REST endpoints
│   └── AdminController.java         // Dashboard stats endpoints
├── service/
│   ├── ReviewService.java           // Business logic
│   ├── ModerationService.java       // AI moderation logic
│   └── TenantService.java           // Tenant resolution
├── repository/
│   ├── ReviewRepository.java        // Spring Data JPA
│   └── TenantRepository.java
├── domain/
│   ├── Review.java                  // JPA Entity
│   ├── Tenant.java
│   └── ReviewStatus.java            // Enum: PENDING, VERIFICATION, APPROVED, REJECTED
├── dto/
│   ├── ReviewRequest.java           // API request DTO
│   ├── ReviewResponse.java          // API response DTO
│   └── ReviewSummaryResponse.java
├── security/
│   ├── TenantContext.java           // ThreadLocal for current tenant
│   └── TenantInterceptor.java       // Extract X-Account header
└── exception/
    ├── TenantNotFoundException.java
    └── GlobalExceptionHandler.java  // @ControllerAdvice
```

KLUCZOWE IMPLEMENTACJE:

1. Multi-tenant isolation:
```java
@Component
public class TenantInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String accountKey = request.getHeader("X-Account");
        if (accountKey == null) {
            throw new BadRequestException("Missing X-Account header");
        }
        Tenant tenant = tenantService.findByAccountKey(accountKey)
            .orElseThrow(() -> new TenantNotFoundException(accountKey));
        TenantContext.setCurrentTenant(tenant);
        return true;
    }
}

// In repository queries:
@Query("SELECT r FROM Review r WHERE r.tenant = :#{T(TenantContext).getCurrentTenant()} AND r.status = 'APPROVED'")
List<Review> findApprovedReviews();
```

2. Virtual Threads configuration:
```properties
# application.properties
spring.threads.virtual.enabled=true
```

3. OpenAI integration with fail-safe:
```java
@Service
public class ModerationService {
    private final WebClient openAiClient;

    public ModerationResult moderateContent(String text) {
        try {
            return openAiClient.post()
                .uri("/v1/moderations")
                .bodyValue(Map.of("input", text))
                .retrieve()
                .bodyToMono(ModerationResult.class)
                .timeout(Duration.ofSeconds(2))
                .block();
        } catch (Exception e) {
            log.error("OpenAI API error, fail-safe to VERIFICATION", e);
            return ModerationResult.failSafe(); // status = VERIFICATION
        }
    }
}
```

4. Soft delete:
```java
@Entity
@Where(clause = "deleted_at IS NULL") // Hibernate filter
public class Review {
    @Id
    private UUID id;

    private LocalDateTime deletedAt;

    @PreRemove
    public void softDelete() {
        this.deletedAt = LocalDateTime.now();
    }
}
```

### 2. Frontend Implementation Best Practices

STRUKTURA:
```
src/
├── components/
│   ├── ReviewsTable.tsx         // Ant Design Table dla opinii
│   ├── StatsCard.tsx            // Ant Design Statistic
│   ├── AiEffectivenessWidget.tsx // Recharts PieChart
│   └── DateRangeFilter.tsx      // Ant Design RangePicker
├── pages/
│   ├── DashboardPage.tsx        // Main dashboard
│   └── LoginPage.tsx            // Auth (jeśli dashboard wymaga osobnego logowania)
├── services/
│   ├── api.ts                   // Axios instance with interceptors
│   └── reviewsApi.ts            // API calls
├── types/
│   └── api.types.ts             // TypeScript types (wygenerowane z OpenAPI)
├── hooks/
│   └── useReviews.ts            // Custom hook dla fetching data
└── utils/
    └── formatters.ts            // Date formatting, etc.
```

KLUCZOWE IMPLEMENTACJE:

1. API client z Basic Auth:
```typescript
// services/api.ts
import axios from 'axios';

const api = axios.create({
  baseURL: '/api',
  headers: {
    'Authorization': `Basic ${btoa('username:' + import.meta.env.VITE_API_SECRET)}`,
    'X-Account': 'admin' // Or from context for multi-tenant dashboard
  }
});

export default api;
```

2. Ant Design Table z sortowaniem:
```typescript
// components/ReviewsTable.tsx
const columns: ColumnsType<Review> = [
  {
    title: 'Tenant',
    dataIndex: ['tenant', 'accountKey'],
    sorter: (a, b) => a.tenant.accountKey.localeCompare(b.tenant.accountKey)
  },
  {
    title: 'Rating',
    dataIndex: 'rating',
    sorter: (a, b) => a.rating - b.rating,
    render: (rating) => <Rate disabled value={rating} />
  },
  {
    title: 'Created',
    dataIndex: 'createdAt',
    sorter: (a, b) => new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime(),
    render: (date) => dayjs(date).format('YYYY-MM-DD HH:mm')
  }
];

<Table dataSource={reviews} columns={columns} pagination={{ pageSize: 50 }} />
```

3. Recharts dla widget AI:
```typescript
// components/AiEffectivenessWidget.tsx
const data = [
  { name: 'Auto-approved', value: stats.autoApproved, fill: '#52c41a' },
  { name: 'To verification', value: stats.toVerification, fill: '#faad14' }
];

<PieChart width={400} height={400}>
  <Pie data={data} dataKey="value" nameKey="name" label />
  <Tooltip />
</PieChart>
```

### 3. DevOps & Deployment

KUBERNETES MANIFESTS:

1. Backend Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: reviews-backend
  template:
    metadata:
      labels:
        app: reviews-backend
    spec:
      containers:
      - name: backend
        image: registry.digitalocean.com/your-registry/reviews-backend:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: url
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: API_SECRET
          valueFrom:
            secretKeyRef:
              name: vault-secret
              key: api-basic-auth-secret
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: vault-secret
              key: openai-api-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: reviews-backend
spec:
  selector:
    app: reviews-backend
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

2. Frontend Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: reviews-frontend
  template:
    metadata:
      labels:
        app: reviews-frontend
    spec:
      containers:
      - name: frontend
        image: registry.digitalocean.com/your-registry/reviews-frontend:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: reviews-frontend
spec:
  selector:
    app: reviews-frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

3. Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: reviews-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - reviews.yourdomain.com
    secretName: reviews-tls
  rules:
  - host: reviews.yourdomain.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: reviews-backend
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: reviews-frontend
            port:
              number: 80
```

4. HorizontalPodAutoscaler:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: reviews-backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: reviews-backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

CI/CD PIPELINE (przykład GitHub Actions):

```yaml
name: Deploy to DigitalOcean

on:
  push:
    branches: [ main ]

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-java@v3
      with:
        java-version: '25'
        distribution: 'temurin'
    - name: Build with Maven
      run: mvn clean package -DskipTests
    - name: Build Docker image
      run: docker build -t registry.digitalocean.com/your-registry/reviews-backend:${{ github.sha }} .
    - name: Push to DO Registry
      run: |
        echo ${{ secrets.DO_REGISTRY_TOKEN }} | docker login registry.digitalocean.com -u ${{ secrets.DO_REGISTRY_TOKEN }} --password-stdin
        docker push registry.digitalocean.com/your-registry/reviews-backend:${{ github.sha }}

  build-frontend:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: '20'
    - name: Install dependencies
      run: npm ci
    - name: Build
      run: npm run build
    - name: Build Docker image
      run: docker build -t registry.digitalocean.com/your-registry/reviews-frontend:${{ github.sha }} ./frontend
    - name: Push to DO Registry
      run: |
        echo ${{ secrets.DO_REGISTRY_TOKEN }} | docker login registry.digitalocean.com -u ${{ secrets.DO_REGISTRY_TOKEN }} --password-stdin
        docker push registry.digitalocean.com/your-registry/reviews-frontend:${{ github.sha }}

  deploy:
    needs: [build-backend, build-frontend]
    runs-on: ubuntu-latest
    steps:
    - uses: digitalocean/action-doctl@v2
      with:
        token: ${{ secrets.DO_API_TOKEN }}
    - name: Update Kubernetes
      run: |
        doctl kubernetes cluster kubeconfig save your-cluster-name
        kubectl set image deployment/reviews-backend backend=registry.digitalocean.com/your-registry/reviews-backend:${{ github.sha }}
        kubectl set image deployment/reviews-frontend frontend=registry.digitalocean.com/your-registry/reviews-frontend:${{ github.sha }}
        kubectl rollout status deployment/reviews-backend
        kubectl rollout status deployment/reviews-frontend
```

### 4. Monitoring & Observability Setup

PROMETHEUS + GRAFANA:

1. Spring Boot Actuator configuration:
```properties
# application.properties
management.endpoints.web.exposure.include=health,info,prometheus,metrics
management.metrics.export.prometheus.enabled=true
management.endpoint.health.show-details=always
```

2. Kluczowe metryki do monitorowania:
- **Backend**:
  - Request rate per endpoint (req/s)
  - Response time p50, p95, p99 per endpoint
  - Error rate 4xx, 5xx per endpoint
  - JVM memory usage (heap, non-heap)
  - Thread count (Virtual Threads pool)
  - Database connection pool metrics (active, idle, waiting)
  - OpenAI API call duration, success rate, error rate

- **Database**:
  - Connection count
  - Query duration per table
  - Slow queries (> 1s)
  - Disk usage

- **Business metrics**:
  - Reviews created per minute per tenant
  - Moderation mode distribution (ALLOW_ALL vs AI vs MANUAL)
  - AI classification: auto-approved vs to-verification ratio
  - Average time to moderate (PENDING/VERIFICATION -> APPROVED/REJECTED)

3. Alerts:
- Backend error rate > 5% for 5 minutes
- Backend response time p95 > 1s for 5 minutes
- OpenAI API error rate > 10% for 5 minutes
- Database connection pool exhausted
- Kubernetes pod restart > 3 times in 10 minutes
- Disk usage > 80%

---

## Werdykt Finalny

### Ocena Ogólna: ⭐⭐⭐⭐⭐ (5/5)

Stack technologiczny jest DOSKONALE dopasowany do projektu a4b-reviews.

### Kluczowe Atuty:

✅ ZERO LEARNING CURVE dla zespołu (Java, Spring Boot)
✅ NAJSZYBSZY TIME-TO-MARKET (Ant Design komponenty OOTB)
✅ SKALOWALNOŚĆ (Kubernetes, PostgreSQL, Virtual Threads)
✅ BEZPIECZEŃSTWO (Spring Security, Vault, HTTPS)
✅ MAINTAINABILITY (Spring Boot ecosystem, Java LTS, TypeScript)
✅ COST-EFFECTIVE dla MVP (DigitalOcean vs AWS)

### Zaakceptowane Trade-offs:

⚠️ Brak cache - akceptowalne dla MVP, łatwo dodać Redis post-MVP
⚠️ Brak message queue - akceptowalne dla MVP, WebClient + fail-safe wystarczy
⚠️ Większy memory footprint (Java) - akceptowalne, doświadczenie zespołu > oszczędność $100/m
⚠️ Ant Design bundle size - akceptowalne, dashboard to internal tool

### Ryzyka Zidentyfikowane i Zmitigowane:

✅ OpenAI API dependency → Fail-safe do VERIFICATION + monitoring
✅ Vendor lock-in (DigitalOcean) → Standardowe K8s manifests, łatwa migracja
✅ Wydajność bez cache → PostgreSQL indexes + Virtual Threads wystarczą dla MVP
✅ TypeScript learning curve → Prosty TypeScript + wygenerowane types z OpenAPI

### Rekomendacja Ostateczna:

🚀 ZATWIERDZAM stack bez zastrzeżeń. Projekt jest gotowy do implementacji.

### Następne Kroki (Post-Approval):

1. Setup projektu:
   - Inicjalizacja Spring Boot 3.5.6 + Java 25 (Spring Initializr)
   - Inicjalizacja React + Vite + TypeScript + Ant Design
   - Setup PostgreSQL na DigitalOcean
   - Setup HashiCorp Vault

2. Implementacja MVP (priorytet wg PRD):
   - Backend: REST API (POST /reviews, GET /products/{id}/reviews, etc.)
   - Backend: Multi-tenant isolation + Basic Auth
   - Backend: OpenAI integration z fail-safe
   - Database: Migracje Flyway (schema + indexes)
   - Frontend: Dashboard z tabelkami, statystykami, widget AI
   - DevOps: Dockerize + Kubernetes manifests

3. Post-MVP (roadmap):
   - Redis cache dla summary endpoints + dashboard metrics
   - Message queue (RabbitMQ/Redis) dla async OpenAI processing
   - Optymalizacja kosztów OpenAI (cache responses)
   - Advanced monitoring dashboards (Grafana)
   - Load testing + performance tuning

---

## Appendix: Dodatkowe Narzędzia i Biblioteki

### Backend Libraries (Maven dependencies)

```xml
<dependencies>
    <!-- Spring Boot Starters -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId> <!-- For WebClient -->
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>

    <!-- API Documentation -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.3.0</version>
    </dependency>

    <!-- Vault Integration -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>

    <!-- Utilities -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.5.5.Final</version>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Frontend Libraries (package.json)

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "antd": "^5.12.0",
    "recharts": "^2.10.0",
    "axios": "^1.6.2",
    "dayjs": "^1.11.10",
    "zustand": "^4.4.7"
  },
  "devDependencies": {
    "@types/react": "^18.2.45",
    "@types/react-dom": "^18.2.18",
    "@vitejs/plugin-react": "^4.2.1",
    "typescript": "^5.3.3",
    "vite": "^5.0.8",
    "eslint": "^8.55.0",
    "@typescript-eslint/eslint-plugin": "^6.15.0",
    "@typescript-eslint/parser": "^6.15.0"
  }
}
```

### Estimowane Koszty (DigitalOcean, MVP)

| Zasób | Konfiguracja | Koszt/miesiąc (USD) |
|-------|--------------|---------------------|
| DOKS Cluster | 3x Standard Droplet (2 vCPU, 4GB RAM) | ~90 |
| Managed PostgreSQL | Basic Plan (2 vCPU, 4GB RAM, 25GB storage) | ~60 |
| Load Balancer | 1x Load Balancer | ~12 |
| Container Registry | Basic | ~5 |
| Vault (self-hosted na K8s) | Shared resources | ~0 |
| **TOTAL** | | **~167 USD/miesiąc** |

Koszty zewnętrzne:
- OpenAI API: $0.002-0.02 per request (zależy od modelu), ~$50-200/miesiąc dla 10k opinii/miesiąc

---

Data utworzenia: 2025-10-15
Wersja: 1.0
Status: APPROVED ✅
