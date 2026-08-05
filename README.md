# t3monitoring TrueNAS catalog

A private TrueNAS SCALE app catalog for the `t3monitoring-dev` instance — same
format as the [official community catalog](https://github.com/truenas/apps), just
hosted separately since this app depends on a privately-built image and isn't meant
to be publicly listed.

## What's here

```
ix-dev/community/t3monitoring/
├── app.yaml                          # Metadata: version, app_version, icon, maintainers...
├── ix_values.yaml                    # Fixed defaults: image repo/tag, container names
├── questions.yaml                    # The config form TrueNAS renders in the UI
├── README.md                         # Shown in the TrueNAS app details panel
└── templates/
    ├── docker-compose.yaml           # Jinja2 template — the actual container topology
    └── test_values/basic-values.yaml # Values used for local render/deploy testing
```

This mirrors `deploy/docker-compose.yaml` from the main repo (web + scheduler +
mariadb, same env vars, same reverse-proxy-trusts-X-Forwarded-Proto setup) but
expressed in TrueNAS's templating format instead of plain compose, which is what
gets you: a real config form in the Apps UI, TrueNAS-managed storage datasets with
automatic permission fixing, and — the actual point of doing this instead of just
using "Install via YAML" — proper update tracking.

## Prerequisite: publish the image

Unlike `deploy/docker-compose.yaml` (which can `build:` locally), this template only
references a pre-built image (`ix_values.yaml` → `images.image`). Build and push it
first:

```bash
docker build -t ghcr.io/undkonsorten/t3monitoring:13.4.33 -f deploy/Dockerfile .
docker push ghcr.io/undkonsorten/t3monitoring:13.4.33
```

Adjust the registry/tag to wherever you actually host it — `ghcr.io/undkonsorten/...`
in `ix_values.yaml` is a placeholder. If it's a private registry, TrueNAS's Docker
daemon needs credentials for it configured on the host (`docker login` on the
TrueNAS box, or the equivalent registry-credentials setup in the Apps UI) — plain
`ghcr.io` packages default to private unless you make them public.

## Publishing an update

Whenever you rebuild the image:

1. Push the new image tag.
2. Update `ix_values.yaml` → `images.image.tag`.
3. Bump `app.yaml` → `app_version` (upstream version) **and** `version` (catalog
   revision — this is the field TrueNAS actually watches to show "Update available").
4. Commit and push to whatever git repo/branch this catalog lives in.
5. In TrueNAS: Apps → Discover → Refresh Catalog (or wait for the periodic sync).

## Registering this catalog in TrueNAS

This directory needs to be the root of its own git repo (or you push this subtree to
one) before TrueNAS can add it:

```bash
# from inside truenas-catalog/
git init && git add -A && git commit -m "Initial t3monitoring catalog"
git remote add origin git@your-git-host:you/t3monitoring-catalog.git
git push -u origin main
```

Then in TrueNAS: **Apps → Manage Catalogs → Add Catalog** → point **Repository** at
that git URL, **Branch** at `main`, **Preferred Trains** at `community` (the only
train this repo defines). Wait ~1-2 minutes for the initial sync, then find
"t3monitoring" under **Apps → Discover**.

## Importing a database fixture/dump

Unlike `deploy/docker-compose.yaml` (which mounts a dump into `mariadb`'s
`/docker-entrypoint-initdb.d` for auto-import on first boot), the catalog's
`mariadb` container is created through `ix_lib`'s `deps.mariadb()` helper, which
doesn't expose that hook. So here, importing a fixture is always a manual,
one-time step you run yourself after the app is installed — never automatic.

**⚠️ This overwrites the schema/data it touches.** Only do this right after first
install, before real data accumulates, or deliberately knowing it'll clobber
what's there.

1. **Get the dump onto the TrueNAS box**, if it isn't already:
   ```bash
   scp deploy/seed/dump.sql.gz root@your-truenas:/tmp/
   ```

2. **Find the mariadb container's actual name.** TrueNAS's Docker apps run as a
   compose project named `ix-<app-name>`, so it's typically
   `ix-t3monitoring-mariadb-1` — confirm rather than assume:
   ```bash
   ssh root@your-truenas
   docker ps --filter name=mariadb --format '{{.Names}}'
   ```

3. **Import it**, piping the gzipped dump straight into the container:
   ```bash
   zcat /tmp/dump.sql.gz | docker exec -i <mariadb-container-name> \
     mariadb -uroot -p'<db_root_password from the app's config>' t3monitoring
   ```
   Replace `t3monitoring` if you changed `consts.db_name` in `ix_values.yaml`, and
   the password with whatever you set for `db_root_password` in the app's config
   form (TrueNAS UI → the app → Edit).

   Prefer the UI instead? Apps → Installed → t3monitoring → **Shell** → select the
   `mariadb` container — but the shell doesn't accept piped stdin from your local
   machine, so you'd need `docker cp /tmp/dump.sql.gz <container>:/tmp/` first, then
   run `zcat /tmp/dump.sql.gz | mariadb -uroot -p'...' t3monitoring` inside the shell.

4. **Restart `t3monitoring` and `t3monitoring-scheduler`** so nothing's holding a
   stale DB-schema assumption from before the import:
   ```bash
   docker restart ix-t3monitoring-t3monitoring-1 ix-t3monitoring-t3monitoring-scheduler-1
   ```

5. **Run the upgrade wizards** — the shipped `deploy/seed/dump.sql.gz` is a
   pre-upgrade TYPO3 11.5 fixture, so on a fresh v13 install the schema is stale
   and the app 500s on missing tables. Run once inside the `t3monitoring` (web)
   container after the import:
   ```bash
   docker exec -i ix-t3monitoring-t3monitoring-1 vendor/bin/typo3 database:updateschema
   docker exec -i ix-t3monitoring-t3monitoring-1 vendor/bin/typo3 upgrade:run databaseRowsUpdateWizard
   docker exec -i ix-t3monitoring-t3monitoring-1 vendor/bin/typo3 upgrade:run sysLogSerialization
   docker exec -i ix-t3monitoring-t3monitoring-1 vendor/bin/typo3 cache:flush
   ```
   (191 field changes + 2 wizards on the 11→13 jump.) If you import a dump that
   was *already* on v13, skip this step.

6. Log into `/typo3/` with an account from the dump to confirm it actually landed.

## Local validation (before pushing)

The official `truenas/apps` repo ships a real render/deploy CLI — this catalog was
built and validated against it directly rather than guessing at the template syntax:

```bash
git clone https://github.com/truenas/apps.git /tmp/truenas-apps
cp -r ix-dev/community/t3monitoring /tmp/truenas-apps/ix-dev/community/
cd /tmp/truenas-apps
# Render only — checks the Jinja template actually produces valid compose:
python3 .github/scripts/ci.py --app t3monitoring --train community --test-file basic-values.yaml --render-only=true
# Full deploy against real Docker, waits for containers to report healthy:
python3 .github/scripts/ci.py --app t3monitoring --train community --test-file basic-values.yaml --wait=true
```

## Relationship to `deploy/`

`deploy/docker-compose.yaml` (plain compose, "Install via YAML") and this catalog
are two independent ways to run the same image — pick one, not both. The catalog is
worth the extra structure if you want update badges and a proper config form; the
plain compose file is worth it for "I just want it running now, I'll manage updates
by hand." See `../deploy/README-TrueNAS.md` for the plain-compose path.
