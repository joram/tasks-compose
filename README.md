# VeilStream Task Tracker

A full-stack task tracking application with web and mobile clients.

## Architecture

```
task-tracker/
├── backend-api/        # RESTful API (Go + Gin)
├── web/                # Web app (TypeScript + HTML/CSS + Vite, no React)
├── mobile/
│   └── flutter/        # Flutter mobile app
└── docker-compose.yml
```

## Stack

| Layer       | Technology                          |
|-------------|-------------------------------------|
| Backend     | Go, Gin                             |
| Auth        | JWT (HS256), single-user            |
| Database    | PostgreSQL (GORM)                   |
| Web         | TypeScript, HTML/CSS, Vite         |
| Mobile      | Flutter (Dart)                      |

---

## Backend API (`/backend-api`)

### Auth

- Only `john@veilstream.com` is permitted to authenticate.
- `POST /auth/login` accepts `{ email, password }` and returns a signed JWT.
- All other routes require `Authorization: Bearer <token>`.
- Tokens are verified on every request; expired or invalid tokens return `401`.

### Endpoints

| Method | Path             | Description              |
|--------|------------------|--------------------------|
| POST   | /auth/login      | Authenticate and get JWT |
| GET    | /tasks           | List all tasks           |
| POST   | /tasks           | Create a task            |
| GET    | /tasks/:id       | Get a task               |
| PATCH  | /tasks/:id       | Update a task            |
| DELETE | /tasks/:id       | Delete a task            |

### Database

With **docker compose**, a **PostgreSQL 16** service runs by default (`postgres` on port 5432, data in volume `postgres_data`). The backend connects using that unless you set **`DATABASE_DSN`** (e.g. Supabase), which always wins.

If `DATABASE_DSN` is empty, the API builds a URL from **`DATABASE_HOST`** (default `postgres`), **`POSTGRES_USER`**, **`POSTGRES_PASSWORD`**, **`POSTGRES_DB`**, and **`POSTGRES_PORT`** (defaults match the compose service).

### Environment Variables

Create `backend-api/.env` or use root `.env` for docker-compose:

```env
# Optional: remote DB (overrides bundled Postgres URL)
# DATABASE_DSN=postgresql://postgres.PROJECT_REF:YOUR_PASSWORD@aws-0-REGION.pooler.supabase.com:6543/postgres
JWT_SECRET=your-jwt-secret
PORT=3000
```

### Running Locally

```bash
cd backend-api
npm install
npm run dev
```

### Running with Docker

```bash
docker-compose up backend
```

---

## Web App (`/web`)

Plain TypeScript + HTML + CSS, bundled with Vite. Desktop-first UX (sidebar nav, task list, detail, archive, settings). No React.

### Running Locally

```bash
cd web
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173). API calls to `/api/*` are proxied to the backend (default `http://localhost:3000`). The frontend uses the **`VITE_API_URL`** env var (baked in at build time) for the API base URL; set it for production builds.

---

## Mobile — Flutter (`/mobile/flutter`)

### Prerequisites

- Flutter SDK >= 3.x
- Xcode (iOS) or Android Studio (Android)

### Environment

API base URL is configured in `lib/config.dart`:

```dart
const String apiBaseUrl = 'http://localhost:3000';
```

Change this to your backend's address for device/emulator testing.

### Running

```bash
cd mobile/flutter
flutter pub get
flutter run
```

### Release APK and in-place updates

From the repo root, `make apk-docker` builds a signed APK (see `mobile/flutter/README.md`). Release builds use a single keystore (`release.keystore`) so installing a newer APK over an existing install works (no "App not installed"). If `android/app` is root-owned after a Docker build, run `make apk-fix-owner` then `make apk-use-release-keystore` once, then build again.

---

## Mobile — React Native (`/mobile/react-native`)

Built with React Native and TypeScript.

### Prerequisites

- Node.js >= 18
- React Native CLI or Expo CLI
- Xcode (iOS) or Android Studio (Android)

### Environment Variables

Create `mobile/react-native/.env`:

```env
API_URL=http://localhost:3000
```

### Running

```bash
cd mobile/react-native
npm install
npx react-native run-android   # Android
npx react-native run-ios       # iOS
```

---

## Database Schema

```sql
CREATE TABLE tasks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       TEXT NOT NULL,
    description TEXT,
    status      TEXT NOT NULL DEFAULT 'pending',  -- pending | in_progress | done
    due_date    TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Docker Compose

`docker-compose.yml` orchestrates the backend and web app locally.

```bash
# Start everything
docker-compose up

# Start a specific service
docker-compose up backend
docker-compose up web
```

Services:

| Service  | Port |
|----------|------|
| backend  | 3000 |
| web      | 5173 |

---

## Kubernetes

The stack is deployed to namespace **env-195**. Use `kubectl` with `-n env-195` (or set `K8S_NS`).

```bash
# Status (pods, services, deployments)
make k8s-status
# or: make k8s-all

# Pods only
make k8s-pods

# Stream logs (first backend or web pod)
make k8s-logs-backend
make k8s-logs-web

# Different namespace
make k8s-pods K8S_NS=other-ns
```

Direct `kubectl` examples:

```bash
kubectl -n env-195 get all
kubectl -n env-195 get pods
kubectl -n env-195 logs -f deployment/backend
kubectl -n env-195 describe pod -l app=web
```

### Backend crash (CrashLoopBackOff)

If the backend pod is in **CrashLoopBackOff**, check logs:

```bash
kubectl -n env-195 logs deployment/backend --tail=50
kubectl -n env-195 logs deployment/backend --previous
```

**Common cause:** database connection fails because `DATABASE_DSN` (and/or `JWT_SECRET`, `ADMIN_PASSWORD`) are not set correctly in the pod. The deployment must inject secrets via `valueFrom.secretKeyRef`, not as literal env values. For example, the backend container env should be:

```yaml
env:
  - name: DATABASE_DSN
    valueFrom:
      secretKeyRef:
        name: <your-secret-name>
        key: DATABASE_DSN
  # same for JWT_SECRET, ADMIN_PASSWORD
```

---

## Development Notes

- The API enforces a single-user policy: only `john@veilstream.com` can log in. Any other email returns `403 Forbidden`.
- JWTs should be stored securely: `HttpOnly` cookies on web, secure storage on mobile.
- All timestamps are UTC.
- `.env` files are gitignored — never commit secrets.
