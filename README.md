# Movu Launcher

Docker Compose orchestrator for the Movu platform — a movie/series review app built as a NATS-based microservices architecture. This repository wires together five independent services (each a git submodule) plus their databases and message broker into one runnable stack.

## Architecture

```
                                   ┌──────────────────┐
Browser/Client ──(HTTP /api)──>   │  client-gateway   │
                                   └─────────┬─────────┘
                                             │ NATS
              ┌──────────────┬───────────────┼───────────────┬──────────────┐
              v              v               v               v              
       auth-service   content-service  review-service   user-service
              │              │               │               │
          auth-db       content-db      review-db        user-db
        (PostgreSQL)    (PostgreSQL)    (PostgreSQL)     (PostgreSQL)

content-service <──(NATS event: content.ratingStatsChanged)── review-service
auth-service    ──(NATS: users.create)────────────────────>   user-service
```

- **[client-gateway](client-gateway)** — the only HTTP entry point; validates and routes requests to the internal services.
- **[auth-service](auth-service)** — registration, login, JWT issuance, password reset.
- **[content-service](content-service)** — movie/series/genre catalog, TMDB sync.
- **[review-service](review-service)** — reviews and ratings, weekly top-rated ranking.
- **[user-service](user-service)** — user profiles, favorites, wishlist.

All inter-service communication runs over a shared [NATS](https://nats.io/) server. Each service (except the gateway) owns a dedicated PostgreSQL database and speaks NATS only — no service reaches another's database directly.

See each service's own README for its message patterns/endpoints, environment variables, and standalone run instructions.

## Requirements

- Docker & Docker Compose
- Git (with submodule support)
- A [TMDB API read access token](https://www.themoviedb.org/settings/api), for content sync

## Getting started

Clone with submodules:

```bash
git clone --recurse-submodules <repo-url>
cd movu-launcher
```

If already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

Configure environment variables:

```bash
cp .env.template .env
```

Fill in `.env` — at minimum: `DB_PASSWORD`, `DB_USERNAME`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `CLIENT_GATEWAY_PORT`, `NATS_SERVERS`, `CORS_ORIGINS`, and `TMDB_API_KEY`. See the [Environment variables](#environment-variables) section below for the full list.

Start the full stack:

```bash
docker compose up
```

The gateway will be reachable at `http://localhost:${CLIENT_GATEWAY_PORT}/api`. Each service hot-reloads on source changes (`npm run start:dev`, bind-mounted `src/`).

To rebuild after a dependency change:

```bash
docker compose up --build
```

To stop and remove containers:

```bash
docker compose down
```

To also wipe database volumes:

```bash
docker compose down -v
```

## Services and ports

| Service | Exposed port | Database | Database port (host) |
|---|---|---|---|
| `nats-server` | `8222` (monitoring) | — | — |
| `client-gateway` | `${CLIENT_GATEWAY_PORT}` | — | — |
| `auth-service` | internal only | `auth-db` | `5441` |
| `content-service` | internal only | `content-db` | `5442` |
| `review-service` | internal only | `review-db` | `5443` |
| `user-service` | internal only | `user-db` | `5440` |

Only `client-gateway` and `nats-server`'s monitoring port are published to the host; the microservices are reachable exclusively over the internal NATS network. Database ports are published for local inspection/tooling (e.g. a DB client).

## Environment variables

Defined once at the repo root (`.env`) and injected into each service by `docker-compose.yml`. See `.env.template` for the authoritative list.

| Variable | Used by | Description |
|---|---|---|
| `CLIENT_GATEWAY_PORT` | client-gateway | Host port the gateway is published on |
| `NATS_SERVERS` | all services | NATS server URL(s) |
| `CORS_ORIGINS` | client-gateway | Comma-separated allowed browser origins |
| `DB_PORT` | all databases | PostgreSQL port inside each container |
| `DB_USERNAME` / `DB_PASSWORD` | all databases | Shared PostgreSQL credentials |
| `USER_DB_NAME` | user-service | user-db database name |
| `AUTH_DB_NAME` | auth-service | auth-db database name |
| `JWT_SECRET` / `JWT_EXPIRES_IN` | auth-service | JWT signing secret and expiration |
| `CONTENT_DB_NAME` | content-service | content-db database name |
| `ENVIRONMENT` | content-service | Runtime environment name |
| `TMDB_API_KEY` | content-service | TMDB v4 read access token |
| `MOVIES_MAX_TOTAL_FETCH_PAGES` | content-service | Max TMDB pages fetched per sync |
| `REVIEW_DB_NAME` | review-service | review-db database name |

## API testing

A Postman collection covering the gateway's HTTP API is included at [`postman/movu-gateway.postman_collection.json`](postman/movu-gateway.postman_collection.json).

## Working with submodules

Each service directory is an independent git repository. To pull the latest changes for all of them:

```bash
git submodule update --remote --merge
```

After updating a submodule, commit the new commit reference from the launcher repo:

```bash
git add <service-directory>
git commit -m "Update <service> submodule commit"
```
