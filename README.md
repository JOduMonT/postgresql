# postgresql

Shared PostgreSQL 16 instance. Standalone-usable with plain Docker Compose, or deployed as
shared tenant infrastructure on Coolify.

## Standalone

```bash
cp .env.example .env
# edit .env and set a real POSTGRES_PASSWORD
docker compose up -d
```

## On Coolify (as shared tenant infra)

Deployed with `docker_compose_location` set to `/docker-compose.coolify.yaml` (not the
default `docker-compose.yaml`) — that variant joins the `coolify` external Docker network so
other apps in the same tenant can reach it by the service name `postgresql`, and gets no
public domain/FQDN (it's a database, not a web app).

Convention: apps needing Postgres get their own database *inside* this instance
(`CREATE DATABASE <app>;`), not their own Postgres container.

## Why two compose files

`docker-compose.yaml` must run standalone with zero pre-existing infrastructure — it can't
require an external network that only exists on the Coolify host. `docker-compose.coolify.yaml`
is the same stack with that one Coolify-specific network attached. See the fleet Hub's Phase 0
design spec for the full rationale.
