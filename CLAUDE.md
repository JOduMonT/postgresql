# CLAUDE.md — postgresql

Shared PostgreSQL instance definition. Read `README.md` first for the standalone-vs-Coolify
basics — this file is the "don't repeat past mistakes" layer.

## This is shared infrastructure, not an app

Deployed once per tenant, reused by every app in that tenant that needs a relational
database. It must **never** get a public domain or a published host port — it's a database,
not a web service. When creating the Coolify application, explicitly suppress the
auto-assigned domain: `PATCH /applications/<uuid>` with
`{"docker_compose_domains": [{"name":"postgresql","domain":""}]}` — Coolify assigns a random
public `<uuid>.<your-domain>` hostname by default if you don't.

## Apps get a database, never their own Postgres container

An app that needs Postgres connects to this instance by its Docker Compose service name
(`postgresql`, reachable on the shared `coolify` network) and gets its own dedicated
database + user inside it:

```sql
CREATE DATABASE <app>;
CREATE USER <app>_user WITH PASSWORD '<generated>';
GRANT ALL PRIVILEGES ON DATABASE <app> TO <app>_user;
\c <app>
GRANT ALL ON SCHEMA public TO <app>_user;
```

**The last `GRANT` is not optional on Postgres 15+.** `public` is no longer writable by
`PUBLIC` by default. Skip it and the new app connects fine, then its first migration fails
with `permission denied for schema public` — which looks like a crash-looping container, not
an obvious permission problem, unless you already know to check this.

## Major-version upgrades are not in-place — and 18+ changed the mount point

Swapping the image tag on a running instance does not upgrade it; Postgres's on-disk format
isn't compatible across major versions, and the container will simply refuse to start against
existing data from an older version (a safe failure, not corruption — but a failure).

**Also, as of the 18.x images specifically:** the expected volume mount point changed from
`/var/lib/postgresql/data` to `/var/lib/postgresql` (the image now manages a
version-specific subdirectory under it, supporting `pg_upgrade --link`-style in-place
upgrades in the future). Mount at the old path and the container crash-loops on start — even
against a completely empty volume. The error message actually explains this if you read the
log; it's not corrupted output.

**The migration path that worked** (16→18, done live on real data with zero loss):
1. `pg_dumpall -U postgres` from the running old-version container, saved somewhere durable.
2. Verify the dump restores cleanly into a *throwaway* container of the target version first
   — spin one up with a scratch volume, pipe the dump in, check for `ERROR` lines (some,
   like `role "postgres" already exists`, are expected noise from restoring onto a
   freshly-initialized cluster; real problems look different), and spot-check table/row
   counts against the source.
3. Repoint the compose file at the new image, the corrected mount path, and a **new,
   differently-named volume** (e.g. `postgresql-data-v18`, not a reused `postgresql-data`) —
   this keeps the old volume as an untouched rollback point rather than risking it.
4. Deploy. The new instance comes up empty (confirm with `\l` before proceeding).
5. Restore the same dump into the now-live instance.
6. Verify: table list, role list, schema grants (`\dn+` should show your app's user with
   `C` on `public`), and row counts, all matching the pre-migration numbers.
7. Watch the dependent app's own logs/health endpoint for a few minutes — a brief DNS
   resolution hiccup during the container swap (a transient `EAI_AGAIN` on the `postgresql`
   hostname, self-resolving in well under a minute) is normal and not data loss; anything
   that doesn't clear up on its own is not normal.

Full worked example with real numbers: this repo's sibling `specs/postgresql.md` in the
fleet Hub, if you have access to it — otherwise the steps above are the complete recipe.

## Security hardening

Both compose files carry a hardening block per the
[OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html):
healthcheck (`pg_isready`), log rotation, dropped capabilities, `no-new-privileges`,
resource limits, and a read-only root filesystem. Validated against real containers — both a
fresh init and a restart against existing data — before landing on the live instance; a real
`CREATE TABLE`/`INSERT`/`SELECT` round-trip succeeded in both scenarios.

**`cap_drop: ALL` alone breaks Postgres's own entrypoint.** It needs five specific
capabilities back — `CHOWN`, `FOWNER`, `DAC_OVERRIDE`, `SETUID`, `SETGID` — to fix the data
directory's ownership and drop privileges from root to the `postgres` user on startup.
Without them: `chmod: /var/run/postgresql: Operation not permitted`, and the container
exits immediately.

**`read_only: true` needs two `tmpfs` mounts.** `/var/lib/postgresql` (the real data) is
already a named volume, so it stays writable regardless. Postgres also needs `/tmp` and
`/var/run/postgresql` (its default Unix socket) writable — nothing else.

## Known gap: no scheduled backups

This runs as a Coolify `dockercompose` application, not a Coolify-managed database resource
— Coolify's built-in scheduled-backup feature doesn't cover it. A manual `pg_dumpall`
snapshot from the last major-version migration is not an ongoing backup mechanism. If you're
picking up backup automation work, this is where it's needed.
