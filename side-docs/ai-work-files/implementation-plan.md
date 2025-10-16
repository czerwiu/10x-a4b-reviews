# Implementation Plan - a4b-reviews MVP

## Przegląd

**Cel:** Dostarczenie MVP systemu opinii dla wielotenantowej platformy e-commerce
**Zespół:** 2-3 doświadczonych Java/Spring Boot developers
**Timeline:** 3-4 tygodnie
**Tech Stack:** Java 25 LTS, Spring Boot 3.5.6, PostgreSQL 15+, React 18, Redis, Kubernetes

---

## Faza 0: Setup projektu (Dzień 1 - 0.5 dnia)

### 0.1 Inicjalizacja projektu backend

**Estymacja:** 2h
**Odpowiedzialny:** Backend Lead

**Zadania:**
- [ ] Wygenerować projekt Spring Boot 3.5.6 (Spring Initializr)
  - Dependencies: Web, Data JPA, PostgreSQL Driver, Flyway, Security, Actuator, Validation, Redis, OpenAPI
- [ ] Skonfigurować Maven multi-module (opcjonalnie: api, core, persistence)
- [ ] Setup Git repository structure
- [ ] Konfiguracja `application.yml` (profiles: local, dev, prod)
- [ ] Dockerfile + `.dockerignore`
- [ ] README.md z instrukcjami uruchomienia

**Deliverables:**
```
a4b-reviews/
├── pom.xml
├── src/main/
│   ├── java/com/a4b/reviews/
│   │   └── ReviewsApplication.java
│   └── resources/
│       ├── application.yml
│       ├── application-local.yml
│       └── application-prod.yml
├── Dockerfile
└── README.md
```

### 0.2 Inicjalizacja projektu frontend

**Estymacja:** 1h
**Odpowiedzialny:** Frontend Developer

**Zadania:**
- [ ] Setup Vite + React 18 + TypeScript
- [ ] Dodać Ant Design 5.x + Recharts
- [ ] Konfiguracja ESLint + Prettier
- [ ] Setup React Query (opcjonalnie)
- [ ] Dockerfile dla frontend
- [ ] Proxy configuration dla API

**Deliverables:**
```
a4b-reviews-ui/
├── package.json
├── vite.config.ts
├── src/
│   ├── App.tsx
│   ├── main.tsx
│   └── vite-env.d.ts
├── Dockerfile
└── README.md
```

### 0.3 Infrastructure setup

**Estymacja:** 1h
**Odpowiedzialny:** Backend Lead / DevOps

**Zadania:**
- [ ] PostgreSQL w K8s (StatefulSet) lub managed service
- [ ] Redis w K8s (StatefulSet) lub managed service
- [ ] Vault secrets dla aplikacji (Basic Auth, DB credentials, OpenAI key)
- [ ] ConfigMaps dla non-sensitive config

---

## Faza 1: Fundament backendu (Tydzień 1 - 5 dni)

### 1.1 Model danych i persistence layer

**Estymacja:** 1.5 dnia
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] **Entities:**
  - `Account` (tenant)
  - `Review` (opinia)
  - Enums: `ReviewStatus`, `ModerationMode`

- [ ] **Flyway migrations:**
  - `V1__create_accounts_table.sql`
  - `V2__create_reviews_table.sql`
  - `V3__create_indexes.sql`

- [ ] **JPA Repositories:**
  - `AccountRepository`
  - `ReviewRepository` z custom queries

**Przykład Entity:**
```java
@Entity
@Table(name = "reviews", indexes = {
    @Index(name = "idx_reviews_account_product", columnList = "account_id,product_id"),
    @Index(name = "idx_reviews_account_status", columnList = "account_id,status"),
    @Index(name = "idx_reviews_product_approved", columnList = "product_id,status,deleted_at")
})
public class Review {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "account_id", nullable = false)
    private Account account;

    @Column(nullable = false)
    private String userId;

    private String author;

    @Column(nullable = false)
    private String orderId;

    @Column(nullable = false)
    private String productId;

    private String variantId;

    @Column(nullable = false)
    @Min(1) @Max(5)
    private Integer rating;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String reviewText;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ReviewStatus status;

    @Column(length = 10)
    private String language;

    private Double classificationScore;

    @Column(columnDefinition = "TEXT")
    private String classificationReason;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadata;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> media;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    private LocalDateTime deletedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

**Flyway migration V2:**
```sql
CREATE TABLE reviews (
    id BIGSERIAL PRIMARY KEY,
    account_id BIGINT NOT NULL REFERENCES accounts(id),
    user_id VARCHAR(255) NOT NULL,
    author VARCHAR(255),
    order_id VARCHAR(255) NOT NULL,
    product_id VARCHAR(255) NOT NULL,
    variant_id VARCHAR(255),
    rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
    review_text TEXT NOT NULL,
    status VARCHAR(20) NOT NULL,
    language VARCHAR(10),
    classification_score DOUBLE PRECISION,
    classification_reason TEXT,
    metadata JSONB,
    media JSONB,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    deleted_at TIMESTAMP
);

CREATE INDEX idx_reviews_account_product ON reviews(account_id, product_id);
CREATE INDEX idx_reviews_account_status ON reviews(account_id, status);
CREATE INDEX idx_reviews_product_approved ON reviews(product_id, status, deleted_at)
    WHERE status = 'APPROVED' AND deleted_at IS NULL;
CREATE INDEX idx_reviews_queue ON reviews(account_id, status, created_at)
    WHERE status IN ('PENDING', 'VERIFICATION');
CREATE INDEX idx_reviews_created_desc ON reviews(created_at DESC);
```

### 1.2 Multi-tenancy infrastructure

**Estymacja:** 1 dzień
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] **TenantContext** - ThreadLocal przechowujący current tenant
- [ ] **TenantInterceptor** - extract X-Account header
- [ ] **TenantFilter** - WebFilter dla każdego requesta
- [ ] **Exception handling** - brak X-Account = 400, nieistniejący tenant = 404
- [ ] **Testy jednostkowe** dla tenant isolation

**Implementacja TenantContext:**
```java
public class TenantContext {
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>();

    public static void setCurrentTenant(String accountId) {
        currentTenant.set(accountId);
    }

    public static String getCurrentTenant() {
        return currentTenant.get();
    }

    public static void clear() {
        currentTenant.remove();
    }
}

@Component
public class TenantFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String accountId = httpRequest.getHeader("X-Account");

        if (accountId == null || accountId.isBlank()) {
            ((HttpServletResponse) response).sendError(400, "Missing X-Account header");
            return;
        }

        try {
            TenantContext.setCurrentTenant(accountId);
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

### 1.3 Security configuration

**Estymacja:** 0.5 dnia
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] Spring Security z Basic Auth
- [ ] Secret z Vault (username/password)
- [ ] HTTPS/TLS configuration (Ingress)
- [ ] CORS configuration dla dashboard
- [ ] Security filter chain

**Implementacja:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Value("${api.auth.username}")
    private String username;

    @Value("${api.auth.password}")
    private String password;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // Internal API
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.builder()
            .username(username)
            .password("{noop}" + password) // Secret z Vault
            .roles("API_CLIENT")
            .build();
        return new InMemoryUserDetailsManager(user);
    }
}
```

### 1.4 REST API - Core endpoints (CRUD)

**Estymacja:** 2 dni
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] **POST /reviews** - dodawanie opinii
- [ ] **GET /products/{id}/reviews** - pobieranie opinii produktu
- [ ] **GET /variants/{id}/reviews** - pobieranie opinii wariantu
- [ ] **GET /products/{id}/reviews/summary** - podsumowanie produktu
- [ ] **GET /variants/{id}/reviews/summary** - podsumowanie wariantu
- [ ] **DTOs** (Request/Response)
- [ ] **Validation** (Bean Validation)
- [ ] **Exception handling** (@ControllerAdvice)
- [ ] **OpenAPI documentation** (SpringDoc)

**Przykład Controller:**
```java
@RestController
@RequestMapping("/api/v1")
@Validated
public class ReviewController {

    private final ReviewService reviewService;

    @PostMapping("/reviews")
    public ResponseEntity<ReviewResponse> createReview(
            @Valid @RequestBody CreateReviewRequest request) {
        ReviewResponse response = reviewService.createReview(request);
        return ResponseEntity.status(201).body(response);
    }

    @GetMapping("/products/{productId}/reviews")
    public ResponseEntity<List<ReviewResponse>> getProductReviews(
            @PathVariable String productId,
            @RequestParam(required = false) Integer rating,
            @RequestParam(defaultValue = "date_desc") String sort) {
        List<ReviewResponse> reviews = reviewService.getProductReviews(productId, rating, sort);
        return ResponseEntity.ok(reviews);
    }

    @GetMapping("/products/{productId}/reviews/summary")
    @Cacheable(value = "product-summary", key = "#productId")
    public ResponseEntity<ReviewSummary> getProductSummary(@PathVariable String productId) {
        ReviewSummary summary = reviewService.getProductSummary(productId);
        return ResponseEntity.ok(summary);
    }
}
```

**Service layer:**
```java
@Service
@Transactional
public class ReviewService {

    private final ReviewRepository reviewRepository;
    private final AccountRepository accountRepository;
    private final ModerationService moderationService;

    public ReviewResponse createReview(CreateReviewRequest request) {
        String accountId = TenantContext.getCurrentTenant();
        Account account = accountRepository.findByAccountKey(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));

        Review review = new Review();
        // ... mapping
        review.setAccount(account);
        review.setStatus(determineInitialStatus(account.getModerationMode()));
        review.setLanguage(detectLanguage(request.getReviewText()));

        Review saved = reviewRepository.save(review);

        // Async moderation if needed
        if (account.getModerationMode() == ModerationMode.MODERATION_AI) {
            moderationService.analyzeReviewAsync(saved.getId());
        }

        return mapToResponse(saved);
    }

    private ReviewStatus determineInitialStatus(ModerationMode mode) {
        return switch (mode) {
            case ALLOW_ALL -> ReviewStatus.APPROVED;
            case MODERATION_MANUAL -> ReviewStatus.PENDING;
            case MODERATION_AI -> ReviewStatus.PENDING; // Will be updated after AI analysis
        };
    }
}
```

---

## Faza 2: Moderacja i AI (Tydzień 2 - 5 dni)

### 2.1 OpenAI Integration

**Estymacja:** 1.5 dnia
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] Spring AI lub OpenAI Java SDK setup
- [ ] OpenAI API key z Vault
- [ ] Prompt engineering dla moderacji
- [ ] Async processing (CompletableFuture lub @Async)
- [ ] Fail-safe handling (circuit breaker - Resilience4j opcjonalnie)
- [ ] Retry logic dla transient errors
- [ ] Timeout configuration

**Implementacja ModerationService:**
```java
@Service
public class ModerationService {

    private final OpenAiClient openAiClient;
    private final ReviewRepository reviewRepository;

    @Async
    public void analyzeReviewAsync(Long reviewId) {
        Review review = reviewRepository.findById(reviewId)
            .orElseThrow(() -> new ReviewNotFoundException(reviewId));

        try {
            ModerationResult result = analyzeReview(review.getReviewText());

            if (result.isSafe()) {
                review.setStatus(ReviewStatus.APPROVED);
            } else {
                review.setStatus(ReviewStatus.VERIFICATION);
            }

            review.setClassificationScore(result.getScore());
            review.setClassificationReason(result.getReason());

            reviewRepository.save(review);

        } catch (Exception e) {
            // Fail-safe: send to manual verification
            log.error("AI moderation failed for review {}: {}", reviewId, e.getMessage());
            review.setStatus(ReviewStatus.VERIFICATION);
            review.setClassificationReason("AI analysis failed: " + e.getMessage());
            reviewRepository.save(review);
        }
    }

    private ModerationResult analyzeReview(String reviewText) {
        String prompt = """
            Analyze the following product review for inappropriate content.
            Check for: profanity, hate speech, personal data (emails, phones), sexual content.

            Review: "%s"

            Respond in JSON format:
            {
              "safe": true/false,
              "score": 0.0-1.0 (confidence),
              "reason": "explanation if not safe"
            }
            """.formatted(reviewText);

        // OpenAI API call with timeout
        String response = openAiClient.complete(prompt, Duration.ofSeconds(10));
        return parseResponse(response);
    }
}
```

### 2.2 Moderation queue endpoints

**Estymacja:** 1 dzień
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] **GET /reviews/queue** - kolejka moderacji (PENDING + VERIFICATION)
- [ ] **PATCH /reviews/{id}/status** - zmiana statusu (approve/reject)
- [ ] Cursor-based pagination
- [ ] Tenant isolation verification
- [ ] Validation status transitions

**Implementacja:**
```java
@RestController
@RequestMapping("/api/v1/reviews")
public class ModerationController {

    private final ModerationService moderationService;

    @GetMapping("/queue")
    public ResponseEntity<PagedResponse<ReviewResponse>> getModerationQueue(
            @RequestParam(required = false) String cursor,
            @RequestParam(defaultValue = "50") int limit) {
        PagedResponse<ReviewResponse> queue = moderationService.getModerationQueue(cursor, limit);
        return ResponseEntity.ok(queue);
    }

    @PatchMapping("/{id}/status")
    public ResponseEntity<ReviewResponse> updateReviewStatus(
            @PathVariable Long id,
            @Valid @RequestBody UpdateStatusRequest request) {
        ReviewResponse updated = moderationService.updateStatus(id, request.getStatus());
        return ResponseEntity.ok(updated);
    }
}
```

**Cursor pagination:**
```java
public PagedResponse<ReviewResponse> getModerationQueue(String cursor, int limit) {
    String accountId = TenantContext.getCurrentTenant();

    LocalDateTime cursorDate = cursor != null
        ? LocalDateTime.parse(cursor)
        : LocalDateTime.now();

    List<Review> reviews = reviewRepository.findModerationQueue(
        accountId,
        cursorDate,
        PageRequest.of(0, limit + 1)
    );

    boolean hasMore = reviews.size() > limit;
    List<Review> pageReviews = hasMore ? reviews.subList(0, limit) : reviews;

    String nextCursor = hasMore
        ? reviews.get(limit).getCreatedAt().toString()
        : null;

    return new PagedResponse<>(
        pageReviews.stream().map(this::mapToResponse).toList(),
        nextCursor,
        hasMore
    );
}
```

### 2.3 Redis caching

**Estymacja:** 1 dzień
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] Redis configuration (Spring Data Redis)
- [ ] Cache dla summary endpoints
- [ ] Cache dla dashboard stats
- [ ] Cache eviction strategy
- [ ] TTL configuration

**Configuration:**
```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            );
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();

        // Product summary - 10 min TTL
        cacheConfigurations.put("product-summary",
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(10)));

        // Dashboard stats - 5 min TTL
        cacheConfigurations.put("dashboard-stats",
            RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(5)));

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(cacheConfiguration())
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }
}
```

**Cache eviction:**
```java
@Service
public class ReviewService {

    @Transactional
    @CacheEvict(value = "product-summary", key = "#review.productId")
    public void updateReviewStatus(Review review, ReviewStatus newStatus) {
        review.setStatus(newStatus);
        reviewRepository.save(review);
    }
}
```

### 2.4 Rate limiting

**Estymacja:** 0.5 dnia
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] Bucket4j + Redis integration
- [ ] Rate limiting per tenant
- [ ] 429 Too Many Requests handling
- [ ] Rate limit headers (X-RateLimit-*)

**Implementacja:**
```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    private final ProxyManager<String> buckets;

    public RateLimitInterceptor(RedissonClient redisson) {
        this.buckets = Bucket4j.extension(RedissonBasedProxyManager.class)
            .proxyManagerForRedisson(redisson)
            .build();
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String accountId = TenantContext.getCurrentTenant();

        Bucket bucket = buckets.builder().build(
            accountId,
            () -> Bucket4j.builder()
                .addLimit(Bandwidth.simple(100, Duration.ofMinutes(1)))
                .build()
        );

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);

        if (probe.isConsumed()) {
            response.addHeader("X-RateLimit-Remaining", String.valueOf(probe.getRemainingTokens()));
            return true;
        } else {
            response.setStatus(429);
            response.addHeader("X-RateLimit-Retry-After", String.valueOf(probe.getNanosToWaitForRefill() / 1_000_000_000));
            return false;
        }
    }
}
```

### 2.5 Testy integracyjne backendu

**Estymacja:** 1 dzień
**Odpowiedzialny:** Wszyscy backend devs

**Zadania:**
- [ ] Setup Testcontainers (PostgreSQL + Redis)
- [ ] Integration tests dla każdego endpointa
- [ ] Testy multi-tenancy isolation
- [ ] Testy moderacji AI (mock OpenAI)
- [ ] Testy rate limiting
- [ ] Testy cache'owania

**Przykład:**
```java
@SpringBootTest
@Testcontainers
class ReviewControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7");

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateReviewForTenant() {
        CreateReviewRequest request = new CreateReviewRequest(/* ... */);

        ResponseEntity<ReviewResponse> response = restTemplate
            .withBasicAuth("api-client", "secret")
            .exchange(
                "/api/v1/reviews",
                HttpMethod.POST,
                new HttpEntity<>(request, headers("X-Account", "tenant1")),
                ReviewResponse.class
            );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getStatus()).isEqualTo("APPROVED");
    }

    @Test
    void shouldIsolateTenantData() {
        // Create review for tenant1
        createReview("tenant1", "product-123");

        // Try to fetch as tenant2
        ResponseEntity<List> response = restTemplate
            .withBasicAuth("api-client", "secret")
            .exchange(
                "/api/v1/products/product-123/reviews",
                HttpMethod.GET,
                new HttpEntity<>(headers("X-Account", "tenant2")),
                List.class
            );

        assertThat(response.getBody()).isEmpty();
    }
}
```

---

## Faza 3: Dashboard administratora (Tydzień 3 - 5 dni)

### 3.1 Backend - Dashboard API

**Estymacja:** 1.5 dnia
**Odpowiedzialny:** Backend Developer

**Zadania:**
- [ ] **GET /admin/stats** - ogólne statystyki systemu
- [ ] **GET /admin/tenants** - lista tenantów z metrykami
- [ ] **GET /admin/reviews/recent** - ostatnie 50 opinii
- [ ] **GET /admin/ai-effectiveness** - skuteczność AI
- [ ] Cache dla dashboard data (5 min TTL)
- [ ] Separate security dla admin endpoints (opcjonalnie)

**Implementacja:**
```java
@RestController
@RequestMapping("/api/v1/admin")
public class AdminDashboardController {

    private final DashboardService dashboardService;

    @GetMapping("/stats")
    @Cacheable(value = "dashboard-stats", unless = "#result == null")
    public ResponseEntity<DashboardStats> getStats() {
        return ResponseEntity.ok(dashboardService.getStats());
    }

    @GetMapping("/tenants")
    @Cacheable(value = "dashboard-tenants")
    public ResponseEntity<List<TenantStats>> getTenantStats() {
        return ResponseEntity.ok(dashboardService.getTenantStats());
    }

    @GetMapping("/reviews/recent")
    public ResponseEntity<PagedResponse<ReviewResponse>> getRecentReviews(
            @RequestParam(required = false) String cursor) {
        return ResponseEntity.ok(dashboardService.getRecentReviews(cursor, 50));
    }

    @GetMapping("/ai-effectiveness")
    @Cacheable(value = "ai-effectiveness")
    public ResponseEntity<AiEffectivenessStats> getAiEffectiveness() {
        return ResponseEntity.ok(dashboardService.getAiEffectiveness());
    }
}
```

**Przykład DTO:**
```java
public record DashboardStats(
    int totalTenants,
    long totalReviews,
    Map<ReviewStatus, Long> reviewsByStatus,
    double averageRating
) {}

public record TenantStats(
    String accountKey,
    String name,
    ModerationMode moderationMode,
    long totalReviews,
    long pendingReviews,
    long verificationReviews,
    long approvedReviews,
    long rejectedReviews,
    double averageRating
) {}

public record AiEffectivenessStats(
    long totalAiAnalyzed,
    long autoApproved,
    long sentToVerification,
    double autoApprovalRate,
    long verifiedAccepted,
    long verifiedRejected,
    double verifiedAcceptanceRate
) {}
```

### 3.2 Frontend - Setup i routing

**Estymacja:** 0.5 dnia
**Odpowiedzialny:** Frontend Developer

**Zadania:**
- [ ] React Router setup
- [ ] Layout z Ant Design (Header, Sider, Content)
- [ ] API client (axios + interceptors)
- [ ] React Query setup
- [ ] Environment config (.env)

**Struktura:**
```
src/
├── api/
│   ├── client.ts
│   └── dashboard.api.ts
├── components/
│   ├── Layout/
│   │   ├── DashboardLayout.tsx
│   │   └── Header.tsx
│   └── Stats/
│       ├── StatCard.tsx
│       └── AiEffectivenessWidget.tsx
├── pages/
│   ├── DashboardPage.tsx
│   ├── TenantsPage.tsx
│   └── RecentReviewsPage.tsx
├── types/
│   └── dashboard.types.ts
├── App.tsx
└── main.tsx
```

**API Client:**
```typescript
// api/client.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '/api/v1',
  auth: {
    username: import.meta.env.VITE_API_USERNAME,
    password: import.meta.env.VITE_API_PASSWORD,
  },
});

export default apiClient;

// api/dashboard.api.ts
import apiClient from './client';
import type { DashboardStats, TenantStats, AiEffectivenessStats } from '../types/dashboard.types';

export const dashboardApi = {
  getStats: () => apiClient.get<DashboardStats>('/admin/stats'),
  getTenants: () => apiClient.get<TenantStats[]>('/admin/tenants'),
  getAiEffectiveness: () => apiClient.get<AiEffectivenessStats>('/admin/ai-effectiveness'),
  getRecentReviews: (cursor?: string) =>
    apiClient.get('/admin/reviews/recent', { params: { cursor } }),
};
```

### 3.3 Frontend - Dashboard components

**Estymacja:** 2 dni
**Odpowiedzialny:** Frontend Developer

**Zadania:**
- [ ] Overview page (statystyki główne)
- [ ] Tenants table (lista tenantów)
- [ ] Recent reviews table (ostatnie opinie)
- [ ] AI effectiveness widget (pie chart)
- [ ] TOP 10 tenants (listy)
- [ ] Loading states
- [ ] Error handling

**Przykład DashboardPage:**
```typescript
import { useQuery } from '@tanstack/react-query';
import { Card, Col, Row, Statistic } from 'antd';
import { dashboardApi } from '../api/dashboard.api';

export const DashboardPage = () => {
  const { data: stats, isLoading } = useQuery({
    queryKey: ['dashboard-stats'],
    queryFn: () => dashboardApi.getStats().then(res => res.data),
    refetchInterval: 300000, // 5 min
  });

  if (isLoading) return <Spin />;

  return (
    <div className="dashboard">
      <h1>System Overview</h1>

      <Row gutter={16}>
        <Col span={6}>
          <Card>
            <Statistic
              title="Total Tenants"
              value={stats.totalTenants}
            />
          </Card>
        </Col>
        <Col span={6}>
          <Card>
            <Statistic
              title="Total Reviews"
              value={stats.totalReviews}
            />
          </Card>
        </Col>
        <Col span={6}>
          <Card>
            <Statistic
              title="Average Rating"
              value={stats.averageRating}
              precision={2}
              suffix="/ 5"
            />
          </Card>
        </Col>
        <Col span={6}>
          <Card>
            <Statistic
              title="Pending Reviews"
              value={stats.reviewsByStatus.PENDING || 0}
            />
          </Card>
        </Col>
      </Row>

      <Row gutter={16} style={{ marginTop: 16 }}>
        <Col span={12}>
          <TenantsTable />
        </Col>
        <Col span={12}>
          <AiEffectivenessWidget />
        </Col>
      </Row>
    </div>
  );
};
```

**AI Effectiveness Widget:**
```typescript
import { useQuery } from '@tanstack/react-query';
import { Card } from 'antd';
import { PieChart, Pie, Cell, ResponsiveContainer, Legend } from 'recharts';
import { dashboardApi } from '../api/dashboard.api';

export const AiEffectivenessWidget = () => {
  const { data, isLoading } = useQuery({
    queryKey: ['ai-effectiveness'],
    queryFn: () => dashboardApi.getAiEffectiveness().then(res => res.data),
  });

  if (isLoading) return <Spin />;

  const chartData = [
    { name: 'Auto Approved', value: data.autoApproved, color: '#52c41a' },
    { name: 'Sent to Verification', value: data.sentToVerification, color: '#faad14' },
  ];

  return (
    <Card title="AI Moderation Effectiveness">
      <ResponsiveContainer width="100%" height={300}>
        <PieChart>
          <Pie
            data={chartData}
            dataKey="value"
            nameKey="name"
            cx="50%"
            cy="50%"
            outerRadius={80}
            label
          >
            {chartData.map((entry, index) => (
              <Cell key={`cell-${index}`} fill={entry.color} />
            ))}
          </Pie>
          <Legend />
        </PieChart>
      </ResponsiveContainer>

      <div style={{ marginTop: 16 }}>
        <p>Auto-approval rate: <strong>{data.autoApprovalRate.toFixed(1)}%</strong></p>
        <p>Verified acceptance rate: <strong>{data.verifiedAcceptanceRate.toFixed(1)}%</strong></p>
      </div>
    </Card>
  );
};
```

### 3.4 Frontend - Styling i responsive design

**Estymacja:** 1 dzień
**Odpowiedzialny:** Frontend Developer

**Zadania:**
- [ ] Custom theme dla Ant Design
- [ ] Responsive layout (mobile support)
- [ ] Dark mode (opcjonalnie)
- [ ] Loading skeletons
- [ ] Error boundaries

---

## Faza 4: Deployment i DevOps (Tydzień 3-4 - 2-3 dni)

### 4.1 Kubernetes manifests

**Estymacja:** 1 dzień
**Odpowiedzialny:** Backend Lead / DevOps

**Zadania:**
- [ ] **Deployment** dla backend API
- [ ] **Deployment** dla dashboard UI
- [ ] **Service** definitions
- [ ] **Ingress** configuration (Nginx)
- [ ] **ConfigMap** dla non-sensitive config
- [ ] **HorizontalPodAutoscaler** dla API
- [ ] **NetworkPolicy** dla izolacji

**Backend Deployment:**
```yaml
# k8s/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: a4b-reviews-api
  namespace: a4b-reviews
spec:
  replicas: 3
  selector:
    matchLabels:
      app: a4b-reviews-api
  template:
    metadata:
      labels:
        app: a4b-reviews-api
    spec:
      containers:
      - name: api
        image: registry.example.com/a4b-reviews-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.url
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: SPRING_REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: redis.host
        - name: API_AUTH_USERNAME
          valueFrom:
            secretKeyRef:
              name: api-credentials
              key: username
        - name: API_AUTH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: api-credentials
              key: password
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: openai-secret
              key: api-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: a4b-reviews-api
  namespace: a4b-reviews
spec:
  selector:
    app: a4b-reviews-api
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: a4b-reviews-api-hpa
  namespace: a4b-reviews
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: a4b-reviews-api
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

**Ingress:**
```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: a4b-reviews-ingress
  namespace: a4b-reviews
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - reviews-api.internal.example.com
    - reviews-dashboard.internal.example.com
    secretName: a4b-reviews-tls
  rules:
  - host: reviews-api.internal.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: a4b-reviews-api
            port:
              number: 80
  - host: reviews-dashboard.internal.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: a4b-reviews-ui
            port:
              number: 80
```

### 4.2 CI/CD Pipeline

**Estymacja:** 1 dzień
**Odpowiedzialny:** DevOps / Backend Lead

**Zadania:**
- [ ] Dockerfile optimization (multi-stage build)
- [ ] CI pipeline (GitHub Actions / GitLab CI / Jenkins)
- [ ] Automated tests w pipeline
- [ ] Docker image build + push
- [ ] Deployment do K8s (kubectl apply / Helm)
- [ ] Smoke tests post-deployment

**Przykład GitHub Actions:**
```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-java@v3
      with:
        java-version: '25'
        distribution: 'temurin'
    - name: Run tests
      run: mvn clean verify

  build-backend:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Build Docker image
      run: |
        docker build -t registry.example.com/a4b-reviews-api:${{ github.sha }} .
        docker push registry.example.com/a4b-reviews-api:${{ github.sha }}

  deploy:
    needs: build-backend
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to K8s
      run: |
        kubectl set image deployment/a4b-reviews-api \
          api=registry.example.com/a4b-reviews-api:${{ github.sha }} \
          -n a4b-reviews
        kubectl rollout status deployment/a4b-reviews-api -n a4b-reviews
```

### 4.3 Monitoring i observability

**Estymacja:** 0.5 dnia
**Odpowiedzialny:** Backend Lead

**Zadania:**
- [ ] Prometheus metrics endpoints (Spring Actuator)
- [ ] Grafana dashboards (JVM, API, business metrics)
- [ ] Alerting rules (Prometheus Alertmanager)
- [ ] Structured logging (Logback JSON)
- [ ] Log aggregation (jeśli masz ELK/Loki)

**Prometheus ServiceMonitor:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: a4b-reviews-api
  namespace: a4b-reviews
spec:
  selector:
    matchLabels:
      app: a4b-reviews-api
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
```

**Przykład Grafana dashboard queries:**
- Request rate: `rate(http_server_requests_seconds_count[5m])`
- Error rate: `rate(http_server_requests_seconds_count{status=~"5.."}[5m])`
- P95 latency: `histogram_quantile(0.95, http_server_requests_seconds_bucket)`
- Reviews created: `rate(reviews_created_total[5m])`
- AI moderation rate: `rate(ai_moderation_total[5m])`

---

## Faza 5: Testing i stabilizacja (Tydzień 4 - 2-3 dni)

### 5.1 E2E testing

**Estymacja:** 1 dzień
**Odpowiedzialny:** Cały zespół

**Zadania:**
- [ ] E2E scenarios (Postman/REST Assured)
- [ ] Multi-tenant isolation tests
- [ ] Performance tests (JMeter/Gatling)
- [ ] Load testing (100 RPS)
- [ ] Chaos engineering (opcjonalnie - kill pods)

**Przykład E2E test:**
```java
@Test
void fullReviewLifecycle() {
    // 1. Create tenant
    Account tenant = createTenant("test-shop", ModerationMode.MODERATION_AI);

    // 2. Add review
    ReviewResponse review = createReview(tenant.getAccountKey(), "product-1", 5, "Great product!");
    assertThat(review.getStatus()).isEqualTo("PENDING");

    // 3. Wait for AI analysis (async)
    await().atMost(5, SECONDS).until(() ->
        getReview(review.getId()).getStatus().equals("APPROVED")
    );

    // 4. Fetch public reviews
    List<ReviewResponse> publicReviews = getProductReviews(tenant.getAccountKey(), "product-1");
    assertThat(publicReviews).hasSize(1);

    // 5. Check summary
    ReviewSummary summary = getProductSummary(tenant.getAccountKey(), "product-1");
    assertThat(summary.getAverageRating()).isEqualTo(5.0);
    assertThat(summary.getTotalReviews()).isEqualTo(1);
}
```

### 5.2 Bug fixing i polish

**Estymacja:** 1-2 dni
**Odpowiedzialny:** Cały zespół

**Zadania:**
- [ ] Fix bugs znalezione w testach
- [ ] Code review całego projektu
- [ ] Documentation update
- [ ] API documentation review (OpenAPI)
- [ ] Performance optimization (jeśli needed)

### 5.3 Documentation

**Estymacja:** 0.5 dnia
**Odpowiedzialny:** Backend Lead

**Zadania:**
- [ ] API documentation (OpenAPI/Swagger)
- [ ] Deployment guide
- [ ] Operations runbook
- [ ] Architecture diagram
- [ ] README update

---

## Tracking i management

### Daily standup (15 min)
- Co zrobiłem wczoraj?
- Co planuję dziś?
- Czy są blockery?

### Weekly review (30 min)
- Przegląd postępów vs plan
- Adjustment timeline jeśli needed
- Demo funkcjonalności

### Definition of Done dla każdej funkcjonalności:
- [ ] Kod napisany i działa
- [ ] Testy jednostkowe written
- [ ] Testy integracyjne written (gdzie applicable)
- [ ] Code review done i approved
- [ ] Documentation updated
- [ ] Deployed to dev environment
- [ ] Smoke tested

---

## Ryzyka i mitigation

| Ryzyko | Prawdopodobieństwo | Impact | Mitigation |
|--------|-------------------|--------|------------|
| OpenAI API quota/rate limit | Medium | High | Implement circuit breaker, fail-safe to manual moderation |
| Performance issues z multi-tenancy | Low | High | Early load testing, proper indexing, Redis cache |
| Delay w AI analysis | Medium | Medium | Async processing, don't block API response |
| Database bottleneck | Low | High | Connection pooling, query optimization, monitoring |
| Frontend delays | Low | Medium | Start frontend równolegle z backend API |
| Integration issues K8s | Low | Medium | Early deployment tests, use existing patterns |

---

## Post-MVP enhancements (backlog)

### P1 (następny sprint):
- [ ] Hard delete endpoint dla GDPR compliance
- [ ] Bulk operations dla admin (mass approve/reject)
- [ ] Advanced filtering dla moderation queue
- [ ] Email notifications dla moderatorów (opcjonalnie)

### P2 (Q2):
- [ ] Media support (images) w opiniach
- [ ] Vendor responses do opinii
- [ ] Export danych (CSV)
- [ ] Advanced analytics w dashboardzie

### P3 (Q3):
- [ ] Internationalization (i18n)
- [ ] Multi-language support dla AI moderation
- [ ] GraphQL API (opcjonalnie)
- [ ] Mobile app dla moderatorów

---

## Success metrics

### Technical metrics:
- [ ] API uptime > 99.5%
- [ ] P95 response time < 200ms (GET endpoints)
- [ ] Zero critical security vulnerabilities
- [ ] Test coverage > 80%

### Business metrics:
- [ ] AI auto-approval rate > 70%
- [ ] Average moderation time < 1h
- [ ] Zero data leaks between tenants
- [ ] Dashboard load time < 2s

### Team metrics:
- [ ] MVP delivered in 4 weeks
- [ ] < 20 critical bugs in first month
- [ ] Zero P0 incidents
- [ ] Team velocity stable

---

## Final checklist przed produkcją

### Security:
- [ ] All secrets w Vault
- [ ] TLS/HTTPS enabled
- [ ] Rate limiting configured
- [ ] Input validation comprehensive
- [ ] CORS properly configured
- [ ] Security audit passed

### Performance:
- [ ] Load testing passed (100 RPS)
- [ ] Database properly indexed
- [ ] Redis cache working
- [ ] HPA configured and tested
- [ ] Connection pools optimized

### Reliability:
- [ ] Health checks working
- [ ] Liveness/readiness probes configured
- [ ] Monitoring dashboards created
- [ ] Alerts configured
- [ ] Runbook documented
- [ ] Backup strategy defined

### Compliance:
- [ ] GDPR soft delete implemented
- [ ] Audit logging enabled
- [ ] Data retention policy defined
- [ ] Privacy policy reviewed

---

**Timeline summary:**
- Week 1: Backend foundation + CRUD API
- Week 2: Moderation + AI integration
- Week 3: Dashboard + caching
- Week 4: Deployment + testing + stabilization

**Total: 3-4 tygodnie dla zespołu 2-3 Java/Spring Boot developers**

**Ready to start? 🚀**
