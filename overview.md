# Backend Overview

## Stack

| Component | Technology |
|---|---|
| Framework | Spring Boot 4 |
| Language | Java |
| Build tool | Gradle |
| ORM | Spring Data JPA + Hibernate |
| Migrations | Flyway |
| API docs | SpringDoc OpenAPI (Swagger UI) |
| Email | Spring Boot Starter Mail |
| Storage client | AWS SDK v2 (Cloudflare R2 compatible) |
| Security | Spring Security 6 |

---

## Package Structure

The backend is organized by feature domain:

```
com.zikrFy.api/
├── Album/
├── AudioFile/
├── Community/        # Comments, Ratings, Reports, Suggestions
├── Country/
├── Daira/
├── Domain/           # Base entities, enums
├── DTOs/             # Request/Response objects
├── Exception/        # Global error handling
├── Genre/
├── Home/
├── Notification/
├── PlayList/
├── Refresh_token/
├── Search/
├── Security/         # JWT, filters, config
├── Subscription/
├── Tag/
├── User/             # Auth, profile, settings
├── Zakir/
└── Zikr/
```

---

## API Design

The API follows REST conventions:

- Base path: `/api/v1/`
- JSON request and response bodies
- Standard HTTP status codes
- Global error handling via `@RestControllerAdvice`
- Pagination on list endpoints (`page`, `size` query parameters)
- Filtering and sorting on Zikr endpoints

### Response format

All responses are wrapped in a standard envelope:

```json
{
  "success": true,
  "message": "OK",
  "data": { ... }
}
```

Error responses:

```json
{
  "success": false,
  "message": "Zikr not found with id: 42",
  "data": null
}
```

---

## Key Design Decisions

### Pagination

All list endpoints that can return large datasets use Spring Data `Pageable`. The response includes:

```json
{
  "content": [...],
  "page": 0,
  "size": 20,
  "totalElements": 847,
  "totalPages": 43,
  "last": false
}
```

### Filtering

The Zikr endpoint supports dynamic filtering via JPA `Specification`:

```
GET /api/v1/zikr?genreId=1&isPremium=false&sortBy=playCount&sortDir=desc&page=0&size=20
```

### Full-text search

Search uses PostgreSQL `tsvector` with a GIN index, with `plainto_tsquery` for query parsing. Title matches are weighted higher than description matches (weight A vs B).

### Error handling

A global `@RestControllerAdvice` catches:
- `ResourceNotFoundException` → 404
- `DataIntegrityViolationException` → 400 (with human-readable constraint messages)
- `MethodArgumentNotValidException` → 400 (validation errors per field)
- Generic `Exception` → 500

### Stats atomicity

Play counts, like counts, and download counts use atomic `@Modifying` JPQL queries to prevent lost updates under concurrent load:

```sql
UPDATE ZikrStats SET total_plays = total_plays + 1 WHERE zikr_id = :id
```

---

### Application Cache

Frequently accessed, slowly changing endpoints are cached in memory using **Caffeine** with per-cache TTLs:

| Cache | TTL | Reason |
|---|---|---|
| `home:trending` | 5 min | Play counts update constantly |
| `home:newReleases` | 15 min | Changes when admin publishes |
| `home:collections` | 30 min | Changes when admin curates |
| `home:topZakirs` | 15 min | Follower counts, not real-time |

Each cache has its own TTL instead of a single shared spec, because different data changes at different rates. Trending content needs short TTLs. Curated collections need longer ones.

---



Transactional emails are sent via **Mailtrap SMTP** using Spring's `JavaMailSender`:

- Email verification on registration
- Password reset with a 6-digit `SecureRandom` code (15-minute expiry)
- Welcome email

---

## Configuration

The application uses environment variables for all sensitive configuration. No secrets are hardcoded.

Key configuration namespaces:
- `spring.datasource.*` — database connection
- `spring.mail.*` — SMTP configuration
- `spring.jwt.*` — JWT secret and expiration
- `r2.*` — Cloudflare R2 credentials
