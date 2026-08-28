# Security Overview

## Summary

| Area               | Implementation                                   |
| ------------------ | ------------------------------------------------ |
| Transport          | HTTPS (TLS via Caddy + Let's Encrypt)            |
| Authentication     | JWT (HS512) + UUID refresh tokens                |
| Password storage   | BCrypt                                           |
| Authorization      | Spring Security RBAC                             |
| Input validation   | `@Valid` + Bean Validation                       |
| SQL injection      | JPA parameterized queries                        |
| Error handling     | Global handler (no internal detail leakage)      |
| Secrets management | Environment variables only                       |
| Token revocation   | Refresh tokens stored in DB, revocable on logout |
| Random generation  | `java.security.SecureRandom` for all codes       |

---

## HTTPS

All traffic is served over HTTPS. Caddy automatically provisions and renews TLS certificates from Let's Encrypt. HTTP is redirected to HTTPS.

---

## CORS

CORS is configured to allow only known origins:
- The Flutter mobile apps
- `https://app.zikrou.com` (PWA)
- The admin panel

---

## Known Limitations

- No Redis-based distributed session store (single-instance VPS)
- No audit trail for user data access (planned via AuditLog entity)
- Push notifications not yet implemented

---

## Secrets Management

All secrets are stored as environment variables and loaded at startup. No secrets are committed to version control. The `.env` file is excluded via `.gitignore`.

Variables include:
- Database credentials
- JWT signing secret
- SMTP credentials
- Cloudflare R2 credentials
