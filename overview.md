# Architecture Overview

## System Design

Zikrou follows a client-server architecture with a centralized REST API backend, mobile and web clients, and a CDN layer for media delivery.

---

## Components

### Clients

| Client | Technology | Platform |
|---|---|---|
| Mobile app | Flutter | Android + iOS |
| Progressive Web App | Flutter Web | Browser |
| Admin panel | HTML/CSS/JS | Browser |

### Backend

A single **Spring Boot 4** application exposing a REST API. It handles:
- Authentication and authorization
- Business logic
- Database access via Spring Data JPA
- Email sending via JavaMailSender

### Database

**PostgreSQL 17** running in a Docker container on the same VPS as the API. It stores:
- Users, roles, and refresh tokens
- Content: Zakirs, Dairas, Albums, Zikrs, AudioFiles
- User data: favorites, history, playlists, follows
- Community: comments, ratings, reports
- Configuration: AppConfig, UserSettings

### Object Storage

**Cloudflare R2** stores all audio files and images. Files are uploaded directly from the client using presigned URLs, bypassing the API server. The API only stores the file URL in the database.

### CDN

**Cloudflare CDN** serves all media files from R2. With proper `Cache-Control` headers, ~90% of media requests are served from Cloudflare edge nodes without touching the origin server.

### Reverse Proxy

**Caddy** runs as a Docker container on the VPS. It:
- Terminates HTTPS (automatic TLS via Let's Encrypt)
- Routes `api.zikrou.com` to the Spring Boot container
- Serves static files for `app.zikrou.com` (Flutter Web PWA)

---

## Domain Structure

```
zikrou.com          → Marketing site (external)
api.zikrou.com      → REST API (Spring Boot on Hetzner)
app.zikrou.com      → Flutter Web PWA (static files on Hetzner)
cdn.zikrou.com      → Audio + images (Cloudflare R2)
```

---

## Request Flow

### API Request (authenticated)

```
Flutter App
    │
    │ HTTPS POST api.zikrou.com/api/v1/auth/login
    ▼
Cloudflare DNS resolves → VPS IP
    │
    ▼
Caddy (port 443)
    │ reverse_proxy localhost:8080
    ▼
Spring Boot API
    │
    ├── JwtFilter validates token
    ├── SecurityConfig checks role
    ├── Controller → Service → Repository
    └── Response JSON
```

### Media Request (audio stream)

```
Flutter App
    │
    │ GET cdn.zikrou.com/zikrs/5/uuid.mp3
    ▼
Cloudflare CDN
    │
    ├── Cache HIT  → served from edge (0ms latency)
    └── Cache MISS → fetched from R2, cached
```

---

## Infrastructure

| Component | Provider | Specs |
|---|---|---|
| VPS | Hetzner | CX33, 4 vCPU, 8 GB RAM, 80 GB SSD |
| Object storage | Cloudflare R2 | Pay-per-use, zero egress |
| CDN + DNS | Cloudflare | Free plan |
| Email | Mailtrap | Transactional SMTP |
| Domain | Cloudflare | zikrou.com |

---

## Docker Compose Structure

```yaml
services:
  caddy:     # Reverse proxy + HTTPS
  api:       # Spring Boot REST API
  postgres:  # PostgreSQL 17
```

All three containers run on the same VPS and communicate via Docker internal network. Only Caddy exposes ports 80 and 443 to the internet.
