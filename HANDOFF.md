# Plagiarism Detection — Setup & Handoff Guide

Packaged 2026-08-04 from the running production deployment.

**No secrets are included in this archive.** `backend/.env.docker` was deliberately
left out — you must create it from the template in Step 2 below.

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| Docker Engine 20.10+ | https://docs.docker.com/engine/install/ |
| Docker Compose v2 | `docker compose version` — note the space, not `docker-compose` |
| RAM | 4 GB minimum, 8 GB recommended (the API image installs OCR + ffmpeg) |
| Disk | ~5 GB for images and volumes |

No Python or Node install is needed on the host — everything builds inside Docker.

---

## 2. Create the environment file

```bash
cd plagiarism-detection
cp backend/.env.docker.example backend/.env.docker
```

Then edit `backend/.env.docker`. The shipped `.example` file is missing three keys
that the running deployment uses — add them:

```ini
# ---- add these three, they are absent from .env.docker.example ----
AI_LOCAL_ENGINE=ollama
MOODLE_SERVICE_EMAIL=moodle-service@moodle.local
MOODLE_API_KEY=                 # leave EMPTY to disable the Moodle integration
```

Values you MUST change before running:

| Key | What to do |
|---|---|
| `SECRET_KEY` | Generate a fresh one: `python3 -c "import secrets; print(secrets.token_urlsafe(48))"`. This signs JWTs — never reuse the production value. |
| `MOODLE_API_KEY` | Leave empty unless you are wiring up Moodle. Empty disables that integration entirely. |

Values that are fine as-is for local development:

- `DATABASE_URL`, `REDIS_URL`, `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND` — these
  use Docker service names (`db`, `redis`) and resolve automatically inside the network.
- `S3_*` and `STORAGE_TYPE=s3` — points at the bundled MinIO container, not real AWS.
- `OPENAI_API_KEY` / `TOGETHER_API_KEY` — only used when
  `USE_EXTERNAL_AI_DETECTION=true`. Leave blank and leave that flag `false`.

---

## 3. Point the AI engine somewhere reachable

`docker-compose.yml` hardcodes an Ollama server on a private AWS address:

```yaml
- OLLAMA_BASE_URL=http://172.31.15.64:11434
```

**That IP will not resolve on your machine.** Pick one:

**Option A — run Ollama locally (recommended)**
```bash
# install from https://ollama.com, then:
ollama pull llama3.1:8b
ollama serve
```
Then change both occurrences (services `api` and `celery-worker`) to:
```yaml
- OLLAMA_BASE_URL=http://host.docker.internal:11434
```
`host.docker.internal` already works — `extra_hosts` is configured in the compose file.

**Option B — skip AI detection.** Leave the URL as-is. Upload and plagiarism
comparison work fine; only the "AI Check" feature will error.

---

## 4. Build and start

```bash
docker compose up -d --build
```

First build takes 5–15 minutes (the API image compiles OCR and ML dependencies).

Watch it come up:
```bash
docker compose logs -f api
```

Wait for the seeding lines, then `Application startup complete`.

---

## 5. Verify

```bash
docker compose ps          # all 6 services should be Up
curl http://localhost:18000/docs   # FastAPI Swagger UI
```

Open the app: **http://localhost:8080**

---

## 6. Log in

The database schema is created automatically on first start
(`Base.metadata.create_all`), and `backend/app/core/database_seed.py` seeds
these accounts:

| Email | Password | Role |
|---|---|---|
| `admin@example.com` | `AdminPass123!` | admin |
| `user1@example.com` | `UserPass123!` | user |
| `user2@example.com` | `UserPass123!` | user |
| `moderator@example.com` | `ModPass123!` | moderator |

Override the admin credentials by setting `ADMIN_EMAIL` / `ADMIN_PASSWORD` in
`backend/.env.docker` *before* the first start. **Change these before any
public deployment** — they are hardcoded defaults in a public repo.

There is no migration step. Seeding is idempotent: existing users are skipped,
so restarts are safe.

---

## Architecture

Six containers, defined in `docker-compose.yml`:

| Service | Image / Build | Host port | Purpose |
|---|---|---|---|
| `frontend` | nginx + built React | **8080** | React SPA; proxies `/api/` to `api` |
| `api` | `./backend` | 18000 (localhost) | FastAPI + gunicorn, 4 uvicorn workers |
| `celery-worker` | `./backend` | — | Async document processing |
| `celery-beat` | `./backend` | — | Scheduled tasks |
| `db` | `ankane/pgvector:v0.5.1` | 15432 (localhost) | PostgreSQL 15 + pgvector embeddings |
| `redis` | `redis:7-alpine` | 16379 (localhost) | Celery broker + result backend |
| `minio` | `minio/minio` | 9000 / 9001 (localhost) | S3-compatible document storage |

**Stack:** FastAPI · SQLAlchemy (async) · fastapi-users (JWT auth) · Celery ·
PostgreSQL + pgvector · MinIO · React + TypeScript + Vite + Tailwind.

The frontend uses **relative** API URLs (nginx proxies `/api/` to `api:8000`),
so no rebuild is needed when the hostname changes — it is portable as-is.

Backend code is bind-mounted (`./backend:/app`), so Python edits need only
`docker compose restart api`, not a rebuild. Frontend changes **do** require
`docker compose up -d --build frontend`.

MinIO console: http://localhost:9001 (`minioadmin` / `minioadmin`).

---

## Common operations

```bash
docker compose logs -f api celery-worker    # tail logs
docker compose restart api                  # after backend edits
docker compose up -d --build frontend       # after frontend edits
docker compose down                         # stop, keep data
docker compose down -v                      # stop and DELETE all data
```

Database shell:
```bash
docker compose exec db psql -U user -d plagiarism_db
```

---

## Troubleshooting

**API restarts in a loop** — usually a missing `backend/.env.docker`.
Check with `docker compose logs api`.

**`port is already allocated`** — 8080 or 15432 is in use. Change the left-hand
side of the mapping in `docker-compose.yml` (e.g. `"8081:80"`).

**AI Check returns an error** — expected unless Step 3 Option A is done.

**Uploads fail with 413** — files over 25 MB. `MAX_UPLOAD_SIZE_MB` in
`backend/app/core/config.py` and `client_max_body_size` in `frontend/nginx.conf`
must be raised together.

**Frontend loads but API calls 502** — the `api` container is still starting.
First boot installs and seeds; give it a minute.

---

## Notes for whoever picks this up

- **Self-registration is open.** `POST /api/v1/auth/register` needs no
  authentication and has no email verification or domain allowlist, so anyone
  who can reach the app can create an account. Role escalation *is* blocked
  (`app/api/auth.py` forces `role="user"` on that path), but if you don't want
  open signup, remove the register router from `app/main.py`.
- **No Alembic.** Schema comes from `Base.metadata.create_all`, which creates
  missing tables but never alters existing ones. Model changes on an existing
  database will need manual SQL or an Alembic setup.
- `uploads/` and `*.bak-*` were excluded from this archive (gitignored).
- `.env.docker.example` is missing the three keys listed in Step 2 — worth
  fixing in the repo.
