# Technical Decisions

---

## Why Spring Boot?

**Problem**: Choose a backend framework for a REST API with complex domain logic, security requirements, and database interactions.

**Options considered**: Node.js (Express/NestJS), Spring Boot, Django

**Decision**: Spring Boot 4

**Reasons**:
- Strong type system reduces runtime errors
- Spring Security provides production-grade authentication out of the box
- Spring Data JPA with Flyway makes database management reliable
- Rich ecosystem for every requirement (mail, validation, Swagger, etc.)
- Better suited for a domain with 20+ entities and complex relationships

**Trade-offs**: Higher memory footprint at startup than Node.js; slower cold start.

---

## Why PostgreSQL?

**Problem**: Choose a database for structured relational data with search requirements.

**Options considered**: MySQL, PostgreSQL, MongoDB

**Decision**: PostgreSQL 17

**Reasons**:
- Native full-text search with `tsvector`, `tsquery`, and GIN indexes
- Superior constraint support (composite primary keys, check constraints)
- `JSONB` available for future use
- Better handling of reserved words and edge cases than MySQL
- Flyway integration is seamless

**Trade-offs**: Slightly more complex initial setup than MySQL.

---

## Why JWT + UUID Refresh Tokens?

**Problem**: Design a stateless authentication system that also supports token revocation.

**Options considered**: Pure JWT (stateless), database sessions, JWT + refresh tokens

**Decision**: JWT access tokens (15 min) + UUID refresh tokens (30 days, stored in DB)

**Reasons**:
- JWT access tokens eliminate database lookups on every API request
- UUID refresh tokens stored in database enable revocation at any time (logout, security incidents)
- Token rotation on every refresh reduces the risk of stolen refresh tokens
- The 15-minute access token window limits damage if a token is intercepted

**Trade-offs**: Refresh tokens require a database table and introduce statefulness for that specific operation.

---

## Why Cloudflare R2 over AWS S3?

**Problem**: Store and serve audio files (up to several MB each) at scale.

**Options considered**: AWS S3, Cloudflare R2, VPS local storage

**Decision**: Cloudflare R2

**Reasons**:
- Zero egress fees: serving audio files from S3 at scale is expensive
- Cloudflare CDN is automatically available with a custom domain
- S3-compatible API means switching is straightforward
- 10 GB free tier covers early-stage usage

**Trade-offs**: Cloudflare R2 has fewer advanced features than S3 (no lifecycle policies at the time of implementation).

---

## Why Caddy over Nginx?

**Problem**: Need a reverse proxy with HTTPS for the VPS.

**Options considered**: Nginx + Certbot, Caddy, Traefik

**Decision**: Caddy

**Reasons**:
- Automatic TLS certificate provisioning and renewal via Let's Encrypt and zero configuration
- Simple, readable Caddyfile syntax
- Can serve static files (Flutter Web) and proxy API requests in the same config
- Docker-native

**Trade-offs**: Less widely used than Nginx; fewer tutorials available for edge cases.

---

## Why Docker Compose over Kubernetes?

**Problem**: Orchestrate multiple services (API, database, reverse proxy) on a single VPS.

**Options considered**: Docker Compose, Kubernetes, manual process management

**Decision**: Docker Compose

**Reasons**:
- Single VPS, two developers and Kubernetes adds operational complexity without benefit at this scale
- Docker Compose is sufficient for 3 containers
- Easy to understand, debug, and modify
- Upgrade path to Kubernetes exists if the project grows

**Trade-offs**: No automatic container scheduling or self-healing across nodes.

---

## Why Stateless JWT for API + Stateful Tokens for Refresh?

See [authentication.md](../security/authentication.md) for the full explanation.

---

## Why Flyway over Hibernate `ddl-auto`?

**Problem**: Manage database schema evolution safely.

**Options considered**: Hibernate `ddl-auto: update`, Flyway, Liquibase

**Decision**: Flyway with `ddl-auto: validate`

**Reasons**:
- Hibernate's `update` mode can silently drop columns or produce inconsistent results
- Flyway provides versioned, auditable, reproducible migrations
- `ddl-auto: validate` catches schema mismatches at startup, preventing silent bugs
- SQL-based migrations are readable and reviewable

**Trade-offs**: Every schema change requires writing a migration file.

---

## Why Composite Primary Keys for join tables?

**Problem**: Model many-to-many or unique user-content relationships.

**Options considered**: Surrogate ID (auto-increment), composite primary key

**Decision**: Composite primary keys for `favorite`, `rating`, `zakir_follow`, `daira_follow`, `playlist_zikr`

**Reasons**:
- The database enforces uniqueness at the constraint level, not just at the application level
- No need for a separate unique index
- Semantically correct: a user can have exactly one favorite entry per Zikr

**Trade-offs**: More complex JPA mapping (`@Embeddable`, `@EmbeddedId`, `@MapsId`, `implements Serializable`, `@EqualsAndHashCode`).

---

## Why Atomic JPQL for counters?

**Problem**: Increment play counts, like counts, and download counts safely under concurrent load.

**Options considered**: Read-modify-write (fetch entity, increment, save), atomic SQL update, Redis

**Decision**: Atomic `@Modifying` JPQL queries

**Reasons**:
- Read-modify-write causes lost updates when multiple users hit the same endpoint simultaneously
- `UPDATE ZikrStats SET total_plays = total_plays + 1` is atomic at the database level
- No additional infrastructure (Redis) needed

**Trade-offs**: Cannot use JPA dirty checking for these fields; must use explicit queries.


**Why Caffeine over Redis?**
Problem: Reduce repeated database queries for frequently accessed, slowly changing data: home feed sections, top Zakirs, new releases.

Options considered: No cache, Redis, Caffeine (in-process)

Decision: Caffeine with per-cache TTLs

Reasons:

Single-instance VPS deployment, no need for a distributed cache
In-process cache means zero network round-trips and no serialization overhead
Redis requires a separate server, adds operational complexity, and network latency
Caffeine is sufficient when all API requests are handled by one process

Configuration — per-cache TTLs instead of a single shared spec:

home:trending         5 min   (play counts update constantly)
home:trendingAll      5 min
home:newReleases      15 min  (changes when admin publishes)
home:newReleasesAll   15 min
home:collections      30 min  (changes when admin curates)
home:topZakirs        15 min  (follower counts, not real-time)
home:topConferenciers 15 min

A single spring.cache.caffeine.spec would apply the same TTL to every cache. Trending data needs a short TTL because it reflects real-time listening. Collections only change when an admin manually curates them — recomputing them every 5 minutes is wasted work. Matching the TTL to the actual rate of change reduces unnecessary database load without serving stale data where freshness matters.

Trade-offs: Cache is not shared across processes. If the API ever scales to multiple instances, Caffeine would need to be replaced with Redis to avoid cache inconsistency between nodes.
