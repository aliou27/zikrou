# CI/CD

## Current Setup

Deployments are currently triggered manually via a deploy script on the VPS after pushing to GitHub. This is appropriate for a solo project at this stage.

```
Developer machine
        │
        │ git push origin main
        ▼
GitHub (private repository)
        │
        │ SSH into VPS
        ▼
VPS: git pull + bash deploy.sh
        │
        ├── Build Docker image
        ├── Restart API container
        └── Flyway runs migrations on startup
```

---

## Deploy Script

The deploy script (`deploy/deploy.sh`) handles:

1. `git pull` — fetch latest changes
2. `./gradlew build -x test` — build the JAR
3. `docker compose build api` — build Docker image
4. `docker compose up -d api` — restart the container
5. Health check on `/actuator/health`

---

## Flutter Web Deployment

The PWA is deployed separately:

```bash
flutter build web --release
rsync -avz --delete build/web/ root@<vps>:/opt/zikrfy-web/app.zikrou.com/
```

Caddy serves the updated static files immediately after the rsync completes.

---

## Planned: GitHub Actions Pipeline

The planned CI/CD pipeline with GitHub Actions:

```yaml
# Trigger: push to main
on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    steps:
      - Checkout code
      - Set up Java 21
      - Run tests (./gradlew test)
      - Build JAR (./gradlew build)
      - Build Docker image
      - Push to GitHub Container Registry
      - SSH into VPS
      - Pull new image
      - docker compose up -d
      - Health check
```

Secrets would be stored in GitHub repository secrets, never in code.

---

## Mobile Releases

### Android

```
flutter build appbundle --release
→ Upload to Google Play Console (closed testing → production)
```

### iOS

```
flutter build ios --release
→ Archive in Xcode
→ Distribute via App Store Connect (TestFlight → App Store)
```

Version numbers are managed in `pubspec.yaml`:

```yaml
version: 1.0.0+7  # name+buildNumber
```

The build number is incremented on every release. The `min_app_build` value in `app_config` can be updated via the admin API to force users to update.

---

## Environment Separation

| Environment | Database | API URL | Notes |
|---|---|---|---|
| Local | localhost:5433 | localhost:8080 | `.env` file |
| Production | Docker container | api.zikrou.com | VPS `.env` file |

There is no staging environment at this time.
