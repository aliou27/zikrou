# Deployment

## Infrastructure

| Component | Provider | Details |
|---|---|---|
| VPS | Hetzner | CX33 — 4 vCPU, 8 GB RAM, 80 GB SSD, Nuremberg |
| OS | Ubuntu 26.04 | |
| Object storage | Cloudflare R2 | Audio files, images |
| CDN + DNS | Cloudflare | Free plan |
| Email | Mailtrap | Transactional SMTP |
| Domain | Cloudflare | zikrou.com |

---

## Server Layout

```
/opt/zikrfy-api/
├── repo/                   # Git repository (backend)
│   ├── deploy/
│   │   ├── docker-compose.yml
│   │   ├── Caddyfile
│   │   └── deploy.sh
│   └── ...
├── .env                    # Environment variables (not in Git)
└── data/
    └── postgres/           # PostgreSQL data volume

/opt/zikrfy-web/
└── app.zikrou.com/         # Flutter Web PWA (static files)
```

---

## Docker Compose

Three containers run on the VPS:

```
caddy      → Reverse proxy, port 80/443 exposed
api        → Spring Boot REST API, port 8080 (internal only)
postgres   → PostgreSQL 17, port 5432 (internal only)
```

Only Caddy is reachable from the internet. The API and database communicate via Docker's internal network.

---

## Domain Routing (Caddy)

```
api.zikrou.com   → reverse_proxy to Spring Boot container
app.zikrou.com   → static file server (Flutter Web build)
```

TLS certificates are provisioned automatically by Caddy via Let's Encrypt.

---

## Cloudflare R2

Audio files and images are stored in Cloudflare R2. Upload flow:

```
1. Client requests a presigned upload URL from the API
2. Client uploads the file directly to R2 (bypasses the API server)
3. Client notifies the API with the final file URL
4. API stores the URL in PostgreSQL
```

Files are served via `cdn.zikrou.com`, which points to the R2 bucket via Cloudflare CDN. `Cache-Control: public, max-age=31536000, immutable` is set on audio files.

---

## Deployment Process

When a new version of the backend is ready:

```bash
# On developer machine
git push origin main

# On VPS
cd /opt/zikrfy-api/repo
git pull origin main
bash deploy/deploy.sh
```

The deploy script:
1. Builds the new Docker image from the JAR
2. Pulls the updated image
3. Restarts the `api` container
4. Caddy continues serving during the restart

For the Flutter Web PWA:

```bash
# Build locally
flutter build web --release

# Transfer to VPS
rsync -avz --delete build/web/ root@<vps>:/opt/zikrfy-web/app.zikrou.com/
```

---

## Database Migrations

Flyway runs automatically at application startup. Migrations are applied in version order and never re-run. If a migration fails, the application fails to start, preventing an inconsistent database state.

---

## Environment Variables

All sensitive configuration is provided via environment variables at runtime. The `.env` file on the VPS is never committed to version control.

Categories of variables:
- Database connection (URL, username, password)
- JWT configuration (secret, expiration)
- SMTP credentials
- Cloudflare R2 credentials (access key, secret key, account ID, bucket name)
- CORS allowed origins

---

## Monitoring

| Tool | Purpose |
|---|---|
| UptimeRobot | Uptime monitoring, SMS alert on downtime |
| Spring Boot Actuator | `/actuator/health` endpoint |
| Sentry | Runtime error tracking |
| `@Slf4j` logs | Application-level logging |

---

## Backup

[PostgreSQL backup strategy — to be documented]
