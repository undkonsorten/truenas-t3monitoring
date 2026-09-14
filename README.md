# t3monitoring TrueNAS catalog

A TrueNAS SCALE app catalog for `ghcr.io/undkonsorten/t3monitoring` — same format
as the [official community catalog](https://github.com/truenas/apps).

## What the container is

TYPO3 13 CMS running the [t3monitoring](https://github.com/georgringer/t3monitoring)
extension (used to monitor other TYPO3 sites), in `Production` context, built on
`php:8.4-apache-bookworm`. One image, two roles — both containers below run the
exact same image, just with a different command:

| Container | Role | Healthcheck |
| --- | --- | --- |
| `t3monitoring` (web) | Apache + TYPO3, serves `/typo3/` on port 80 | `curl` against `/typo3/`; adds `X-Forwarded-Proto: https` when `force_backend_https` is on, so the check still exercises a full page render instead of following TYPO3's HTTPS redirect |
| `t3monitoring-scheduler` | same image, runs `vendor/bin/typo3 scheduler:run` in a `while true` retry loop (TYPO3 has no persistent scheduler daemon — `scheduler:run` is meant to be invoked periodically, traditionally via cron) | `pidof sh` liveness check on the loop |

Only the web container's port (80 internally) is published. Both containers share
the same MariaDB database and the same two volumes:

| Path | Purpose |
| --- | --- |
| `/var/www/html/public/fileadmin` | TYPO3's public `fileadmin` — user-managed content/uploads |
| `/var/www/html/var/log` | TYPO3's `var/log` |

### Environment variables

Set by the compose template from the TrueNAS config form (`questions.yaml`); listed
here as the container's actual contract, useful if you're running the image
directly instead of through this catalog.

| Variable | Purpose |
| --- | --- |
| `SERVER_NAME` | Public hostname TYPO3 builds its site base URL from |
| `TYPO3_BE_LOCKSSL` | `1`/`0` — forces the backend onto HTTPS (redirects plain-HTTP `/typo3/` requests); only usable behind a TLS-terminating reverse proxy |
| `TYPO3_DB_HOST` / `TYPO3_DB_PORT` / `TYPO3_DB_USER` / `TYPO3_DB_PASSWORD` / `TYPO3_DB_DBNAME` | MariaDB connection |
| `TYPO3_SYS_ENCRYPTIONKEY` | Signs sessions, hashes and links — must not be blank |
| `TYPO3_BE_ADMIN_USER` / `TYPO3_BE_ADMIN_PASSWORD` | Backend administrator created on first boot, if the database is empty |
| `TYPO3_BE_INSTALLTOOLPASSWORD` | Install Tool password; plain text is accepted and hashed once at boot |
| `T3M_DB_AUTO_SETUP` | `1` on **exactly one** container — the one that runs first-boot schema setup (the web container, in this catalog's template). Both containers wait for MariaDB; only this one may run DDL, or concurrent `database:updateschema` calls corrupt the schema |

### First boot

A fresh install hands the app an empty database. The container holding
`T3M_DB_AUTO_SETUP=1` runs, in order: `database:updateschema safe` (creates the
schema), `extension:setup` (static data), `backend:createadmin` (if admin
credentials were given), `cache:flush`. The other container just waits for the
scheduler-specific tables to exist before it starts its loop.

### Known, expected behaviour

- **The public frontend 404s** (`No site configuration found`) on a fresh
  install — there's no content page yet. The backend at `/typo3/`, which is all
  t3monitoring needs, works fine.
- **Three admin rows after setup**: the configured administrator plus two
  `_cli_` system users TYPO3's own console commands create. Not usable for
  interactive login.
