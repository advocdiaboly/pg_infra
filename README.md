# PostgreSQL Docker Setup

Local PostgreSQL for development, based on the `pgvector/pgvector` Docker image.
This project provides a reusable PostgreSQL service and shared Docker network;
application-specific users, databases, and extensions are managed by consuming
projects.

The default container name is `postgres`, and PostgreSQL listens on host port
`5432`.

## Start

```sh
docker compose up -d
```

Check status:

```sh
docker compose ps
```

## Stop

```sh
docker compose down
```

This stops and removes the container but keeps the named Docker volume with the
database data.

## Connect

Administrative connection from the host:

```sh
psql "postgresql://postgres:postgres_password@localhost:5432/postgres"
```

Using `psql` inside the container:

```sh
docker compose exec postgres psql -U postgres -d postgres
```

From another Compose project, attach that project to the shared `pg-infra`
network and connect to host `postgres` on port `5432`.

## Configuration

Edit `.env` for local settings. `.env.example` documents the available variables.

This setup uses `pgvector/pgvector:pg18` by default. PostgreSQL 18+ images store
data under `/var/lib/postgresql`, so the named volume is mounted there.

Available variables:

| Variable | Default | Description |
| --- | --- | --- |
| `POSTGRES_VERSION` | `pg18` | `pgvector/pgvector` image tag. |
| `POSTGRES_CONTAINER_NAME` | `postgres` | Docker container name. |
| `POSTGRES_PORT` | `5432` | Host port mapped to container port `5432`. |
| `POSTGRES_PASSWORD` | `postgres_password` | Password for the default PostgreSQL admin role. |
| `POSTGRES_NETWORK_NAME` | `pg-infra` | Shared Docker network name for dependent Compose projects. |
| `PRICE_GRABBER_MCP_DB_USER` | `price_grabber_mcp_ro` | Local PostgreSQL login for the private read-only MCP. |
| `PRICE_GRABBER_MCP_DB_PASSWORD` | none | Required local password for that MCP login. |

For a pet project this setup intentionally keeps credentials simple. Do not reuse
these defaults for production.

## Private PostgreSQL MCP

`compose.yaml` defines `postgres-mcp`, a private MCP Toolbox for Databases
service used by Hermes to inspect the `price_grabber` database. It is not
published to a host port; consumers on `pg-infra` reach it as
`http://postgres-mcp:5000/mcp`.

The service starts with the Toolbox `postgres` prebuilt toolset. It includes
`execute_sql`, so its PostgreSQL role must remain least-privilege and
read-only; Docker networking is not a substitute for database authorization.
The configured host/origin allow-list is an additional browser-facing guard,
not an authorization mechanism.

Start or update only the MCP service:

```sh
docker compose up -d postgres-mcp
```

Stop it without changing database data:

```sh
docker compose stop postgres-mcp
```

## Initialization Scripts

`initdb/` is reserved for one-time server-level initialization files. It is
currently intentionally empty except for `.gitkeep`.

Application-specific setup, such as creating an app user, app database, or
installing app-required extensions, should live in the consuming project.

The official entrypoint runs files in `initdb/` only when the database volume is
empty. After the first successful startup, use each application migration or
bootstrap flow for schema and role changes.

To reset all local PostgreSQL data and re-run initialization scripts:

```sh
docker compose down -v
docker compose up -d
```

## Data Volume

Database files live in the named Docker volume `pg_infra_pgdata`.

List volumes:

```sh
docker volume ls
```

Inspect this project's volume:

```sh
docker volume inspect pg_infra_pgdata
```

Remove all local database data:

```sh
docker compose down -v
```

Use `down -v` carefully. It deletes the PostgreSQL data volume for this Compose
project.

## Logs

Follow PostgreSQL logs:

```sh
docker compose logs -f postgres
```

Show recent logs:

```sh
docker compose logs --tail=100 postgres
```

## Healthcheck

The service uses `pg_isready` against the default `postgres` database. A healthy
container should appear like this:

```txt
postgres   pgvector/pgvector:pg18   Up ... (healthy)
```

## Troubleshooting

If port `5432` is already in use, change `POSTGRES_PORT` in `.env`, for example:

```env
POSTGRES_PORT=5433
```

If changes in `initdb/` do not run, the database volume already exists. Init
scripts run only on first initialization. Reset with `docker compose down -v`
only when deleting local data is acceptable.

If authentication fails after editing `.env`, remember that `POSTGRES_PASSWORD`
only affects first initialization. Existing volumes keep the original roles and
passwords unless changed explicitly with SQL.

## Upgrade Notes

Pin the image tag in `.env` instead of using `latest`.

Patch-level upgrades within the same major version are usually straightforward:

```env
POSTGRES_VERSION=pg18
```

Major upgrades, such as PostgreSQL 18 to 19, require a planned database upgrade
or dump/restore. Do not change the major version on an existing data volume and
expect PostgreSQL to start automatically.
