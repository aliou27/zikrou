# Authentication

## Overview

Zikrou uses a stateless JWT-based authentication system with stateful refresh tokens. The implementation is built on **Spring Security 6**.

---

## Registration & Login Flow

```
1. User submits email + password
        │
        ▼
2. AuthService validates input
   ├── Check email uniqueness
   └── Check username uniqueness
        │
        ▼
3. Password hashed with BCrypt
        │
        ▼
4. User saved to database
        │
        ▼
5. JwtService generates access token (JWT, 15 min)
        │
        ▼
6. RefreshTokenService creates UUID refresh token (30 days)
   └── Saved to database with expiry
        │
        ▼
7. Response: { accessToken, refreshToken, userId, email, role }
```

---

## Token Architecture

| Token | Type | Duration | Storage | Revocable |
|---|---|---|---|---|
| Access token | JWT (HS512) | 15 minutes | Client memory | No |
| Refresh token | UUID v4 | 30 days | Database | Yes |

### Why this approach?

A JWT access token is **stateless**: the server does not need to query the database to validate it. This makes API requests fast.

A UUID refresh token is **stateful**: it is stored in the database and can be revoked at any time (logout, password change, suspicious activity). This solves the main limitation of pure JWT auth.

---

## Request Authentication Flow

```
Client sends: Authorization: Bearer eyJhbGci...
        │
        ▼
JwtFilter (OncePerRequestFilter)
        │
        ├── Extract token from header
        ├── Parse and verify JWT signature
        ├── Extract email from claims
        ├── Load UserDetails from database
        └── Set Authentication in SecurityContextHolder
        │
        ▼
SecurityConfig
        ├── Public routes → pass through
        ├── Authenticated routes → require valid token
        └── Admin routes → require ROLE_ADMIN
        │
        ▼
Controller receives authenticated request
```

---

## Token Refresh

When the access token expires:

```
Client sends: POST /auth/refresh { refreshToken: "uuid" }
        │
        ▼
RefreshTokenService
        ├── Find token in database
        ├── Check not revoked
        ├── Check not expired
        └── Valid → generate new access token + new refresh token
            (old refresh token revoked — rotation)
```

---

## Password Security

- Passwords are hashed with **BCrypt** before storage
- Plain-text passwords are never logged or stored
- Password reset uses a 6-digit code generated with `SecureRandom`
- Reset codes expire after 15 minutes

---

## Role-Based Access Control

Two roles are defined: `USER` and `ADMIN`.

Authorization is enforced at two levels:

**1. Route level** (SecurityConfig):
```
Public:        /auth/**, GET /zikr/**, GET /zakir/**, ...
Authenticated: /favorites/**, /history/**, /playlists/**, ...
Admin only:    /admin/**
```

**2. Method level** (`@PreAuthorize`):
```
POST/PUT/DELETE on content endpoints → @PreAuthorize("hasRole('ADMIN')")
```

---

## Email Verification

After registration:
1. A verification token (UUID) is generated and saved with a 24-hour expiry
2. A verification email is sent via Mailtrap SMTP
3. The user clicks the link → token is validated → account is marked as verified

---

## Security Practices

- JWT signed with HS512 (minimum 512-bit key)
- All sensitive configuration via environment variables: no hardcoded secrets
- Input validation with `@Valid` on all request DTOs
- Global exception handler prevents internal details from leaking in error responses
- HTTPS enforced via Caddy (automatic TLS)
- CORS configured per environment
- SQL injection prevented via JPA parameterized queries
- Refresh token rotation on every use
