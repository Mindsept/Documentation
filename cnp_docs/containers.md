# Containers and Local Network

This document explains the Docker Compose topology for Cloud Native Platform.

The local setup is development-friendly. In production, the same backend/PostgreSQL/worker containers are used, but public TLS is handled by host Nginx on the VPS.

## Services

The Compose stack defines four services:

```txt
postgres
backend
worker
nginx
```

## Container Topology

```mermaid
flowchart LR
    browser[Browser or Frontend]
    github[GitHub]
    ngrok[ngrok tunnel]
    nginx[Nginx container<br/>port 80]
    backend[FastAPI backend<br/>port 8000]
    worker[Worker<br/>job runner]
    postgres[(PostgreSQL<br/>port 5432)]
    volume[(postgres_data<br/>Docker volume)]

    browser -->|http://localhost:8080| nginx
    browser -->|http://localhost:8000| backend
    github -->|webhooks and callback| ngrok
    ngrok -->|forwards to localhost:8080| nginx
    nginx -->|proxy /api /docs /openapi.json| backend
    backend -->|SQLAlchemy async| postgres
    worker -->|polls jobs table| postgres
    worker -->|GitHub API for jobs| github
    backend -->|GitHub API| github
    postgres --> volume
```

## Docker Compose Network

Docker Compose creates a default private network for the project.

Inside that network:

```txt
backend -> postgres:5432
worker  -> postgres:5432
nginx   -> backend:8000
```

That is why `DATABASE_URL` must use:

```env
DATABASE_URL="postgresql+asyncpg://cnp:cnp_password@postgres:5432/cnp"
```

The hostname `postgres` is the Compose service name, not a host machine DNS name.

## Exposed Host Ports

```txt
localhost:8080 -> nginx:80
localhost:8000 -> backend:8000
localhost:5432 -> postgres:5432, unless POSTGRES_PORT is changed
```

If local port `5432` is already occupied:

```env
POSTGRES_PORT=15432
```

Then PostgreSQL is available on:

```txt
localhost:15432
```

The internal container address remains:

```txt
postgres:5432
```

## Service Dependencies

```mermaid
flowchart TD
    postgres[postgres healthcheck<br/>pg_isready]
    backend[backend<br/>alembic upgrade head<br/>uvicorn]
    worker[worker<br/>python -m app.worker.runner]
    nginx[nginx<br/>reverse proxy]

    postgres -->|healthy| backend
    backend -->|healthy| worker
    backend -->|healthy| nginx
```

The backend waits for PostgreSQL to be healthy, runs Alembic migrations, and then starts Uvicorn.

The worker waits for the backend to be healthy to avoid racing migrations.

Nginx waits for the backend healthcheck.

## Request Routing

```mermaid
sequenceDiagram
    participant Client
    participant Nginx
    participant Backend
    participant PostgreSQL

    Client->>Nginx: GET /api/v1/health
    Nginx->>Backend: proxy to backend:8000/api/v1/health
    Backend-->>Nginx: 200 JSON
    Nginx-->>Client: 200 JSON

    Client->>Nginx: POST /api/v1/auth/login
    Nginx->>Backend: proxy request
    Backend->>PostgreSQL: read user and refresh token state
    PostgreSQL-->>Backend: result
    Backend-->>Nginx: access token and refresh token
    Nginx-->>Client: 200 JSON
```

## Nginx Responsibilities

The Compose Nginx service is a local reverse proxy in front of the backend.

It proxies:

```txt
/api/          -> backend:8000/api/
/docs          -> backend:8000/docs
/openapi.json  -> backend:8000/openapi.json
```

It also applies basic local-development safeguards:

- request size limit
- rate limiting on API routes
- forwarding headers

Production uses host Nginx instead:

```txt
Internet
-> host Nginx on OVH VPS, HTTPS and rate limiting
-> backend container on 127.0.0.1:8000
```

The validated production Nginx configuration is documented in [Production Deployment](production-deployment.md).

## Backend Container

The backend image is built from:

```txt
apps/backend/Dockerfile
```

Startup command:

```bash
alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000
```

With `docker-compose.override.yml`, development mode mounts:

```txt
./apps/backend:/app
```

and starts Uvicorn with `--reload`.

## Worker Container

The worker runs:

```bash
python -m app.worker.runner
```

It polls the `jobs` table in PostgreSQL and executes queued operations.

The worker uses the same backend codebase, environment variables, and database as the API server.

## PostgreSQL Container

The PostgreSQL service:

- uses the official `postgres:16-alpine` image
- persists data in the `postgres_data` Docker volume
- runs SQL init files from `docker/postgres/init`
- uses `pg_isready` for healthchecks

Data is preserved when running:

```bash
docker compose down
```

Data is deleted when running:

```bash
docker compose down -v
```

## ngrok in Local Development

ngrok is only required when GitHub must call your local backend:

- GitHub App setup callback
- GitHub App webhooks

Recommended local forwarding:

```txt
ngrok public URL -> http://localhost:8080
```

GitHub App local URLs:

```txt
Setup URL:   https://your-ngrok-domain.ngrok-free.dev/api/v1/github/setup-callback
Webhook URL: https://your-ngrok-domain.ngrok-free.dev/api/v1/webhooks/github
```

ngrok is not required for normal local API calls from the frontend to `localhost`.

## Common Commands

Start:

```bash
docker compose up -d --build
```

Stop without deleting data:

```bash
docker compose down
```

Reset database and volumes:

```bash
docker compose down -v
docker compose up -d --build
docker compose exec backend python -m app.db.seed
```

Check service state:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs backend --tail=100
docker compose logs worker --tail=100
docker compose logs nginx --tail=100
```

Production commands use the base Compose file explicitly to avoid development overrides:

```bash
docker compose -f docker-compose.yml up -d --build
docker compose -f docker-compose.yml logs backend --tail=150
docker compose -f docker-compose.yml down
```
