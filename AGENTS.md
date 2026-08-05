# AGENTS.md

Guidance for coding agents (and humans) working in this TrueNAS catalog repository.

## What this is

A private TrueNAS SCALE app catalog for the **t3monitoring** app — same format as
the [official community catalog](https://github.com/truenas/apps), hosted separately
because the app depends on a privately-built image and isn't meant to be publicly
listed. This repo is a **git submodule** of `t3monitoring-dev` at
`truenas-catalog/`; the parent repo pins a commit, edits here are a separate
commit/push, and the parent then bumps the pointer.

## Current target

TYPO3 **13.4** / PHP **8.4**, image `ghcr.io/undkonsorten/t3monitoring:13.4.33`,
MariaDB **10.11**. t3monitoring extension `^3.1`.

## Layout

```
ix-dev/community/t3monitoring/
├── app.yaml                 # metadata: version (catalog rev), app_version (upstream), icon...
├── ix_values.yaml           # fixed defaults: image repo/tag, mariadb tag, container names, run uid/gid
├── questions.yaml           # the config form TrueNAS renders in the UI
├── README.md                # shown in the TrueNAS app details panel
└── templates/
    ├── docker-compose.yaml  # Jinja2 template — the actual container topology (web + scheduler + mariadb)
    └── test_values/basic-values.yaml  # values used for local render/deploy testing
README.md                    # this catalog's usage docs (registering, publishing, importing fixtures)
```

## The catalog mirrors `deploy/` from the parent repo

`templates/docker-compose.yaml` expresses the same stack as
`../deploy/docker-compose.yaml` (plain compose): `web` + `scheduler` (same image)
+ `mariadb`, same env vars, same reverse-proxy-trusts-`X-Forwarded-Proto` setup.
Differences: TrueNAS templating format (config form, auto storage datasets,
permission fixing, update tracking) vs. raw paste-YAML. Pick one, not both.

## Updating on a new image build

1. Build + push the image (from the parent repo):
   `docker build -t ghcr.io/undkonsorten/t3monitoring:<tag> -f deploy/Dockerfile . && docker push ...`
2. Bump **`ix_values.yaml`** → `images.image.tag` (and `mariadb_image.tag` if MariaDB moved).
3. Bump **`app.yaml`** → `app_version` (upstream TYPO3/t3monitoring version) **and**
   `version` (catalog revision — **this** is the field TrueNAS watches for "Update
   available"; increment it on every change, even metadata-only).
4. Commit + push **this** repo.
5. In the parent `t3monitoring-dev` repo: `git add truenas-catalog` and commit the
   new pinned commit (the submodule pointer moved).
6. TrueNAS: Apps → Discover → Refresh Catalog (or wait for periodic sync).

## TYPO3 13 specifics carried in the template

- **Single `public/` docroot** (no `private/` split — `helhum/typo3-secure-web` is
  gone). Storage mounts `public/fileadmin`, not `private/fileadmin`.
- **Config lives under `config/system/`** (`settings.php` + `additional.php`), baked
  into the image — the catalog only injects runtime env vars (`TYPO3_DB_*`,
  `TYPO3_SYS_ENCRYPTIONKEY`, `TYPO3_BE_INSTALLTOOLPASSWORD`, `TYPO3_MAIL_*`).
- **CLI binary is `vendor/bin/typo3`** (typo3-console v8 merged into core; no
  `typo3cms`). Scheduler loops `vendor/bin/typo3 scheduler:run` every 60s.
- **`TYPO3_SYS_ENCRYPTIONKEY` must be non-empty** in the config form — compose always
  sets the env var, so blank overwrites the baked key and TYPO3 refuses to boot.
- **Post-import upgrade**: the `deploy/seed/dump.sql.gz` is a pre-upgrade 11.5
  fixture. After importing it into a fresh install, run once (in the `web`
  container) or the app 500s on missing tables:
  `vendor/bin/typo3 database:updateschema` then each pending `upgrade:run <wizard>`
  (191 field changes + 2 wizards on the 11→13 jump). See root `README.md`.

## Local validation

Validate against the official `truenas/apps` repo before pushing (render + optional
deploy):

```bash
git clone https://github.com/truenas/apps.git /tmp/truenas-apps
cp -r ix-dev/community/t3monitoring /tmp/truenas-apps/ix-dev/community/
cd /tmp/truenas-apps
python3 .github/scripts/ci.py --app t3monitoring --train community --test-file basic-values.yaml --render-only=true
python3 .github/scripts/ci.py --app t3monitoring --train community --test-file basic-values.yaml --wait=true
```

## Gotchas

- **This is a submodule**: a plain `git clone` of the parent leaves this dir empty —
  `git clone --recurse-submodules` or `git submodule update --init --recursive`.
  Edits here don't show in the parent until you `git add truenas-catalog` there.
- **`version` is what TrueNAS tracks** for updates, not `app_version` — bump it on
  every published change.
- **`deps.mariadb()` has no init-script hook**, so DB fixture import is always
  manual (see root `README.md` → "Importing a database fixture/dump").
- **MariaDB charset stays `utf8`/`utf8_unicode_ci`** on purpose — the fixture DB
  predates utf8mb4; `deps.mariadb()` only sets `--port`, so the template re-sets
  the full command with the classic charset.
