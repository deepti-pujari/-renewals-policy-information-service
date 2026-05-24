# 📋 Renewals Policy Information Service

> High-concurrency REST service delivering comprehensive policy data for **50,000+ active policies** — optimized for speed with PostgreSQL views, strategic indexing, and Hibernate/JPA.

![Java](https://img.shields.io/badge/Java-17-ED8B00?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Hibernate](https://img.shields.io/badge/Hibernate-59666C?style=flat-square&logo=hibernate&logoColor=white)
![Swagger](https://img.shields.io/badge/Swagger-OpenAPI_3.0-85EA2D?style=flat-square&logo=swagger&logoColor=black)

---

## 📊 Impact at a Glance

| Metric | Result |
|---|---|
| Active policies served | **50,000+** |
| Concurrent requests supported | **500+** |
| Query latency improvement | **30–40%** |
| Downstream consumers fed | **4 services** |
| Optimization techniques | PostgreSQL views + composite indexing |

---

## 🏗️ Architecture Overview

```
                         ┌──────────────────────────────────────────┐
  Renewal Service  ──►   │    Renewals Policy Information Service    │
  Payment Service  ──►   │                                           │
  Audit Service    ──►   │   ┌──────────────┐   ┌───────────────┐   │
  Notification Svc ──►   │   │  REST Layer  │──►│  Policy Info  │   │
                         │   │  (OpenAPI)   │   │    Service    │   │
                         │   └──────────────┘   └──────┬────────┘   │
                         │                             │            │
                         │                   ┌─────────▼──────────┐ │
                         │                   │   Repository Layer  │ │
                         │                   │   (Hibernate/JPA)  │ │
                         │                   └─────────┬──────────┘ │
                         └─────────────────────────────┼────────────┘
                                                        │
                              ┌─────────────────────────▼──────────────┐
                              │              PostgreSQL                  │
                              │                                          │
                              │  ┌──────────────┐  ┌─────────────────┐  │
                              │  │  Base Tables │  │  Optimized      │  │
                              │  │  policies    │  │  Views          │  │
                              │  │  holders     │  │  vw_policy_info │  │
                              │  │  premiums    │  │  vw_renewal_due │  │
                              │  └──────────────┘  └─────────────────┘  │
                              │                                          │
                              │  Composite Indexes on:                   │
                              │    policy_id, status, renewal_date       │
                              └──────────────────────────────────────────┘

  Downstream consumers fed by this service:
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   Renewal    │  │   Payment    │  │ Notification │  │   Audit      │
  │   Service   │  │   Service   │  │   Service   │  │   Service   │
  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

---

## ✨ Key Features

- **High-concurrency read layer** — sustains 500+ concurrent requests without degradation via connection pooling and efficient JPA query design
- **PostgreSQL view optimization** — database views pre-join and pre-aggregate policy data, eliminating repeated expensive joins at query time
- **Composite indexing strategy** — targeted indexes on `policy_id`, `status`, and `renewal_date` reduce full-table scans and cut latency by 30–40%
- **Cross-service data consistency** — single source of truth for policy information consumed by 4 downstream services
- **Hibernate/JPA with projection DTOs** — avoids fetching unused columns; only the fields each consumer needs are returned
- **OpenAPI 3.0 documented** — all endpoints fully described with request/response schemas via Swagger UI

---

## 🗂️ Project Structure

```
renewals-policy-information-service/
├── src/
│   └── main/
│       └── java/com/insurance/policyinfo/
│           ├── controller/
│           │   └── PolicyInfoController.java
│           ├── service/
│           │   └── PolicyInfoService.java
│           ├── repository/
│           │   ├── PolicyInfoRepository.java
│           │   └── PolicyViewRepository.java
│           ├── model/
│           │   ├── entity/
│           │   │   └── Policy.java
│           │   └── dto/
│           │       ├── PolicyInfoDTO.java
│           │       └── PolicySummaryDTO.java
│           └── config/
│               └── DatabaseConfig.java
├── src/main/resources/
│   ├── db/
│   │   ├── views/
│   │   │   └── vw_policy_info.sql       # Optimized PostgreSQL view
│   │   └── indexes/
│   │       └── policy_indexes.sql       # Composite index definitions
│   └── application.properties.example
└── README.md
```

---

## 🔑 Core Components

### Policy Info Controller (stub)

```java
@RestController
@RequestMapping("/api/v1/policies")
@Tag(name = "Policy Information API", description = "Retrieve policy data for renewal workflows")
public class PolicyInfoController {

    private final PolicyInfoService policyInfoService;

    @GetMapping("/{policyId}")
    @Operation(summary = "Get full policy information by ID")
    public ResponseEntity<PolicyInfoDTO> getPolicyInfo(@PathVariable String policyId) {
        return ResponseEntity.ok(policyInfoService.getPolicyInfo(policyId));
    }

    @GetMapping("/batch")
    @Operation(summary = "Retrieve policy info for a batch of IDs")
    public ResponseEntity<List<PolicyInfoDTO>> getPoliciesBatch(
            @RequestParam List<String> policyIds) {
        return ResponseEntity.ok(policyInfoService.getPoliciesBatch(policyIds));
    }

    @GetMapping("/due-for-renewal")
    @Operation(summary = "List policies due for renewal within a date range")
    public ResponseEntity<Page<PolicySummaryDTO>> getPoliciesDueForRenewal(
            @RequestParam LocalDate from,
            @RequestParam LocalDate to,
            Pageable pageable) {
        return ResponseEntity.ok(policyInfoService.getDueForRenewal(from, to, pageable));
    }
}
```

### JPA Repository with Projection (stub)

```java
@Repository
public interface PolicyInfoRepository extends JpaRepository<Policy, String> {

    // Uses PostgreSQL view via @Query — avoids full table join at runtime
    @Query(value = "SELECT * FROM vw_policy_info WHERE policy_id = :policyId", nativeQuery = true)
    Optional<PolicyInfoProjection> findPolicyInfoById(@Param("policyId") String policyId);

    // Batch fetch with IN clause — efficient for downstream consumer calls
    @Query(value = "SELECT * FROM vw_policy_info WHERE policy_id IN :policyIds", nativeQuery = true)
    List<PolicyInfoProjection> findAllByPolicyIds(@Param("policyIds") List<String> policyIds);

    // Paginated renewal window query — hits composite index on (status, renewal_date)
    @Query(value = """
            SELECT * FROM vw_renewal_due
            WHERE renewal_date BETWEEN :from AND :to
            AND status = 'ACTIVE'
            ORDER BY renewal_date ASC
            """, nativeQuery = true)
    Page<PolicySummaryProjection> findDueForRenewal(
            @Param("from") LocalDate from,
            @Param("to") LocalDate to,
            Pageable pageable);
}
```

### PostgreSQL View (vw_policy_info.sql)

```sql
-- Pre-joins policy, holder, and premium tables to eliminate repeated joins at query time.
-- Reducing per-query join cost is the primary driver of the 30-40% latency improvement.

CREATE OR REPLACE VIEW vw_policy_info AS
SELECT
    p.policy_id,
    p.policy_number,
    p.status,
    p.product_type,
    p.inception_date,
    p.expiry_date,
    p.renewal_date,
    ph.holder_name,
    ph.date_of_birth,
    ph.contact_number,
    ph.email_id,
    pr.annual_premium,
    pr.gst_amount,
    pr.total_payable,
    pr.payment_frequency
FROM policies p
JOIN policy_holders ph ON p.holder_id = ph.holder_id
JOIN premiums pr       ON p.policy_id = pr.policy_id
WHERE p.is_deleted = FALSE;
```

### Composite Index Definitions (policy_indexes.sql)

```sql
-- Primary lookup index — used by single-policy fetch (most frequent query)
CREATE INDEX IF NOT EXISTS idx_policy_id
    ON policies (policy_id);

-- Renewal window query index — status + renewal_date filters hit this together
CREATE INDEX IF NOT EXISTS idx_status_renewal_date
    ON policies (status, renewal_date);

-- Downstream consumer batch fetch index
CREATE INDEX IF NOT EXISTS idx_policy_number
    ON policies (policy_number);
```

---

## 🔍 Query Optimization Strategy

The 30–40% latency reduction came from three combined changes:

```
Before optimization:
  GET /policies/{id}
  └── Query: SELECT ... FROM policies
              JOIN policy_holders ...    ← repeated at runtime
              JOIN premiums ...          ← repeated at runtime
              WHERE policy_id = ?
  Avg latency: ~180ms at 500 concurrent

After optimization:
  GET /policies/{id}
  └── Query: SELECT * FROM vw_policy_info WHERE policy_id = ?
              ↑ view pre-joins at definition time
              ↑ composite index on policy_id eliminates full scan
              ↑ projection DTO fetches only needed columns
  Avg latency: ~105ms at 500 concurrent  (↓ ~40%)
```

---

## ⚙️ Configuration

```properties
# Server
server.port=8081

# PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:5432/policy_db
spring.datasource.username=YOUR_DB_USER
spring.datasource.password=YOUR_DB_PASSWORD

# Connection pool (tuned for 500+ concurrent requests)
spring.datasource.hikari.maximum-pool-size=50
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.connection-timeout=30000

# Hibernate
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.jdbc.batch_size=25
spring.jpa.open-in-view=false
```

---

## 🚀 Running Locally

**Prerequisites:** Java 17+, PostgreSQL 14+

```bash
# 1. Clone the repo
git clone https://github.com/deepti-pujari/renewals-policy-information-service.git
cd renewals-policy-information-service

# 2. Set up the database
psql -U postgres -c "CREATE DATABASE policy_db;"
psql -U postgres -d policy_db -f src/main/resources/db/views/vw_policy_info.sql
psql -U postgres -d policy_db -f src/main/resources/db/indexes/policy_indexes.sql

# 3. Configure
cp application.properties.example src/main/resources/application.properties

# 4. Build & run
./mvnw spring-boot:run
```

API docs: `http://localhost:8081/swagger-ui.html`

---

## 📄 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/policies/{policyId}` | Full policy info by ID |
| `GET` | `/api/v1/policies/batch?policyIds=` | Batch fetch for multiple IDs |
| `GET` | `/api/v1/policies/due-for-renewal` | Paginated renewal window query |
| `GET` | `/api/v1/policies/{policyId}/summary` | Lightweight summary for downstream use |
| `GET` | `/api/v1/policies/search` | Filter by status, product type, date range |

---

## 🧪 Testing

```bash
# Unit + integration tests
./mvnw test

# With coverage
./mvnw verify
```

Tests cover service-layer logic (JUnit 5 + Mockito), repository queries against an embedded H2 instance, and concurrent request handling via Spring's MockMvc.

---

## 👩‍💻 Author

**Deepti Pujari** — Java Software Development Engineer  
[LinkedIn](https://linkedin.com/in/deepti-pujari) · [GitHub](https://github.com/deepti-pujari) · [deeptipujari02@gmail.com](mailto:deeptipujari02@gmail.com)

---

> *This repository contains sanitized code stubs and documentation. Production source code is proprietary to Star Health & Allied Insurance Co. Ltd.*
