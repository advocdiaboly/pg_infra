# Repository Guidelines For Agents

This repository provides reusable local PostgreSQL infrastructure using Docker
Compose and the `pgvector/pgvector` image. Application-specific users,
databases, schemas, and extensions belong in the consuming project, not here.

## Current Shape

- Compose file: `compose.yaml`
- Local environment file: `.env`
- Example environment file: `.env.example`
- Server-level init scripts directory: `initdb/`
- Documentation: `README.md`
- Default container name: `postgres`
- Default image: `pgvector/pgvector:pg18`
- Shared Docker network: `pg-infra`
- Data volume: `pg_infra_pgdata`

## Operating Rules

- Keep this project generic. Do not add application-specific users, databases, or
  extension scripts here.
- Prefer the `pgvector/pgvector` Docker image so consumers can use pgvector when
  they need it.
- Keep image tags pinned. Do not switch to unqualified `postgres` or `latest`.
- Preserve the named volume unless the user explicitly asks to reset data.
- Do not run `docker compose down -v` unless data deletion is requested or
  clearly required and approved.
- For PostgreSQL 18 and newer, keep the data volume mounted at
  `/var/lib/postgresql`.
- Keep `initdb/` for server-level first-boot initialization only.
- Treat `.env` as local configuration. Keep `.env.example` in sync when
  variables are added, renamed, or removed.

## Useful Commands

Start:

```sh
docker compose up -d
```

Stop without deleting data:

```sh
docker compose down
```

Status:

```sh
docker compose ps
```

Admin connection:

```sh
docker compose exec postgres psql -U postgres -d postgres
```

Logs:

```sh
docker compose logs -f postgres
```

Render config:

```sh
docker compose config
```

## Validation Checklist

After changing Compose or env files:

1. Run `docker compose config`.
2. If Docker daemon access is available, run `docker compose up -d`.
3. Verify `docker compose ps` shows `postgres` as healthy.
4. Verify an admin SQL connection:

```sh
docker compose exec -T postgres psql -U postgres -d postgres -c 'select current_database(), current_user;'
```

## Consumer Setup

Consuming projects should:

- attach their services to the external `pg-infra` Docker network;
- connect to PostgreSQL at `postgres:5432`;
- create their own app role and app database;
- install their own required extensions;
- run their own schema migrations.

Keep consumer-specific bootstrap logic in the consumer repository.

## Destructive Actions

These commands delete local database state and should not be run casually:

```sh
docker compose down -v
docker volume rm pg_infra_pgdata
```

Ask the user before running destructive commands unless they explicitly requested
a reset.
