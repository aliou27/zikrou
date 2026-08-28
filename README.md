# Zikrou

![Platform](https://img.shields.io/badge/platform-Android%20%7C%20iOS%20%7C%20Web-blue)
![Backend](https://img.shields.io/badge/backend-Spring%20Boot%204-green)
![Database](https://img.shields.io/badge/database-PostgreSQL%2017-blue)
![Status](https://img.shields.io/badge/status-Production-brightgreen)
![License](https://img.shields.io/badge/license-Proprietary-red)

Islamic audio streaming application for the Taalibé community of Cheikh Ibrahima Niass and anyone who appreciates their Zikr.

---

## Overview

Zikrou is a full-stack mobile and web application that centralizes Islamic audio content: Zikrs and Conversations from communities across West Africa and beyond (Senegal, Mauritania, Nigeria, Guinea).

The application is available on **Android**, **iOS** , and as a **Progressive Web App** at [app.zikrou.com](https://app.zikrou.com)

---

## Problem

The audio content of the Taalibé community is scattered across WhatsApp groups, YouTube channels, Telegram, and local MP3 files. There is no centralized, organized platform dedicated to this content.

Zikrou solves this by providing a single, clean, and accessible streaming platform organized by Zakir, Daira, Album, and Genre designed to be usable by all generations, including older and less tech-savvy users.

---

## Features

**Content**
- Stream Zikrs by Zakir, Daira, Album, or Genre
- Full-text search powered by PostgreSQL `tsvector`
- Content from multiple countries and communities

**User**
- Account creation with email and password
- Email verification and password reset
- User profile with optional name
- Favorites, listening history, follow Zakirs and Dairas
- Playlists with ordered tracks
- Sleep timer
- Offline download 

**Discovery**
- Home feed: trending, recently added, continue listening, from followed Zakirs
- Search with history
- Recommendations by genre and community

**Platform**
- Android (Google Play: closed testing)
- iOS (available)
- Progressive Web App (app.zikrou.com)
- Admin web panel

---

## Architecture

```
┌─────────────────────────────────────────────┐
│                  Clients                    │
│  Android App │ iOS App │ PWA │ Admin Panel  │
└──────────────────────┬──────────────────────┘
                       │ HTTPS
                       ▼
┌─────────────────────────────────────────────┐
│              Cloudflare                     │
│         DNS + CDN + R2 Storage              │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│         Hetzner VPS (CX33)                  │
│  ┌─────────────────────────────────────┐    │
│  │  Caddy (Reverse Proxy + HTTPS)      │    │
│  └──────────────┬──────────────────────┘    │
│                 │                            │
│  ┌──────────────▼──────────────────────┐    │
│  │  Spring Boot 4 REST API             │    │
│  │  (Docker Container — port 8080)     │    │
│  └──────────────┬──────────────────────┘    │
│                 │                            │
│  ┌──────────────▼──────────────────────┐    │
│  │  PostgreSQL 17                      │    │
│  │  (Docker Container — port 5432)     │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘

Audio & Images → Cloudflare R2 (CDN)
Emails         → Mailtrap SMTP
```

See [docs/architecture/overview.md](docs/architecture/overview.md) for details.

---

## Technology Stack

| Category | Technology |
|---|---|
| **Mobile** | Flutter (Android + iOS) |
| **State management** | Riverpod |
| **HTTP client** | Dio |
| **Navigation** | Go Router |
| **Audio** | just_audio |
| **Local storage** | flutter_secure_storage, shared_preferences |
| **Backend** | Spring Boot 4, Java |
| **Database** | PostgreSQL 17 |
| **Migrations** | Flyway |
| **Authentication** | Spring Security + JWT (HS512) |
| **Build tool** | Gradle |
| **Reverse proxy** | Caddy |
| **Containerization** | Docker + Docker Compose |
| **VPS** | Hetzner CX33 (4 vCPU, 8GB RAM) |
| **CDN & DNS** | Cloudflare |
| **Object storage** | Cloudflare R2 |
| **Email** | Mailtrap SMTP |
| **API documentation** | SpringDoc OpenAPI (Swagger) |
| **Version control** | Git + GitHub |

---

## Security

- **Authentication**: JWT access tokens (15 min) + UUID refresh tokens stored in database (30 days, revocable)
- **Password hashing**: BCrypt
- **Authorization**: Role-based access control (USER, ADMIN) via Spring Security `@PreAuthorize`
- **HTTPS**: Automatic TLS via Caddy + Let's Encrypt
- **Input validation**: `@Valid` on all request DTOs
- **CORS**: Configured per environment
- **SQL injection**: Prevented via JPA parameterized queries
- **Token revocation**: Refresh tokens can be revoked at logout or on suspicious activity

See [docs/security/authentication.md](docs/security/authentication.md) for details.

---

## Deployment

The application runs on a single **Hetzner CX33 VPS** in Nuremberg, Germany, orchestrated with **Docker Compose**:

```
services:
  caddy      # Reverse proxy + automatic HTTPS
  api        # Spring Boot REST API
  postgres   # PostgreSQL database
```

Audio and image files are stored on **Cloudflare R2** and served via **Cloudflare CDN**, offloading ~90% of media traffic from the server.

See [docs/deployment/deployment.md](docs/deployment/deployment.md) for details.

---

## CI/CD

Deployments are triggered manually via a deploy script on the VPS after pushing to the main branch on GitHub. The process builds a new Docker image, pulls it on the server, and restarts the containers with zero-downtime.

See [docs/cicd/github-actions.md](docs/cicd/github-actions.md) for details.

---

## My Role

Zikrou is a solo project. I designed, built, and published every part of it:

- **Product design**: defined the problem, the target users, and the feature set
- **Backend**: designed and implemented the entire REST API with Spring Boot
- **Database**: designed the PostgreSQL schema (20+ tables), wrote all Flyway migrations
- **Authentication**: implemented JWT auth with refresh token rotation and email verification
- **Security**: role-based access control, input validation, CORS, HTTPS
- **Mobile**: built the Flutter app for Android and iOS
- **Infrastructure**: provisioned and configured the Hetzner VPS, Docker, Caddy
- **Storage**: integrated Cloudflare R2 for audio and image files
- **CDN**: configured Cloudflare CDN to serve media files
- **Deployment**: wrote and maintains the deploy pipeline
- **Testing**: manual and Postman collection testing across all endpoints
- **Publication**: submitted and published on Google Play (closed testing) and Apple TestFlight

  As for the Deployment https://github.com/jeanthiao helped me to set up the vps and automate it with CI/CD, Github Actions and SonaCloud

---

## Technical Challenges

### JWT key size
Spring Security's `jjwt` library requires a minimum 512-bit key for HS512. The initial configuration used a shorter key, causing a `WeakKeyException` at runtime. Fixed by generating a proper 64-byte hex secret.

### PostgreSQL reserved word
The `user` table name conflicts with a reserved PostgreSQL keyword. Fixed with `@Table(name = "\"user\"")` using escaped quotes in the JPA entity.

### Composite primary keys
Tables like `favorite`, `rating`, and `playlist_zikr` use composite primary keys. Implementing them in JPA requires `@Embeddable` + `@EmbeddedId` + `@MapsId`, with `implements Serializable` and `@EqualsAndHashCode` — all three are mandatory.

### Concurrent counter updates
The `ZikrStats` entity tracks play counts, downloads, and likes. A naive read-modify-write pattern causes lost updates under concurrent load. Fixed with atomic `@Modifying` JPQL queries: `UPDATE ZikrStats SET total_plays = total_plays + 1`.

### CDN cache invalidation
After updating the Cloudflare R2 CORS policy, the CDN served stale cached responses without the new headers. Fixed by purging the Cloudflare cache after configuration changes.

### Full-text search
Simple `LIKE` queries are slow and imprecise on large datasets. Implemented PostgreSQL full-text search using `tsvector` with a GIN index and `plainto_tsquery`, with weighted columns (title = A, description = B) for relevance ranking.

---

## Technical Decisions

See [docs/decisions/technical-decisions.md](docs/decisions/technical-decisions.md) for the full reasoning behind each major decision.

Key decisions:
- **Spring Boot over Node.js**: type safety, mature ecosystem, Spring Security
- **PostgreSQL over MySQL**: full-text search, JSONB, better constraint support
- **Stateless JWT + stateful refresh tokens**: balance between performance and revocability
- **Cloudflare R2 over AWS S3**: zero egress fees, integrated CDN
- **Caddy over Nginx**: automatic HTTPS with zero configuration
- **Docker Compose over Kubernetes**: appropriate complexity for a solo project at this scale

---

## Screenshots

*Screenshots available in [assets/screenshots/](assets/screenshots/)*

| Home Feed | Audio Player | Search |
|---|---|---|
| <img width="642" height="1389" alt="Screenshot 2026-08-27 at 11 59 09 PM" src="https://github.com/user-attachments/assets/1fae8d2e-1761-494d-92a9-9613a64ad551" />
| <img width="642" height="1389" alt="Screenshot 2026-08-27 at 11 59 52 PM" src="https://github.com/user-attachments/assets/08858c04-782d-45b7-971d-bb1acda8c4b0" />
 | <img width="642" height="1389" alt="Screenshot 2026-08-28 at 12 00 30 AM" src="https://github.com/user-attachments/assets/277cb118-becb-44c7-84de-1f8b9dfa7c6d" />
 |

---

## Demo

https://drive.google.com/file/d/1ZRBj-HYLATBcI9bIV1dwsAmtU9zUnlv3/view?usp=drive_link

---

## Production

| Platform | Status |
|---|---|
| Android (Google Play) | Closed testing |
| iOS (TestFlight) | Available |
| Web PWA | Live at [app.zikrou.com](https://app.zikrou.com) |

---

## Source Code

The backend source code is maintained in a private repository.

This repository contains technical documentation only. No source code, credentials, or proprietary implementation details are published here.

---

## License

© 2026 Aliou Ba. All rights reserved.

This repository contains documentation only. No part of this documentation may be used to reproduce, reverse-engineer, or derive the Zikrou product or its backend implementation.
