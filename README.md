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

This mirrors `deploy/docker-compose.yml` from the main repo (web + scheduler +
mariadb, same env vars, same reverse-proxy-trusts-X-Forwarded-Proto setup) but
expressed in TrueNAS's templating format instead of plain compose.

Since TrueNAS 24.10 dropped custom catalogs (see "Installing this on TrueNAS"
below), the templating format no longer buys UI integration. What it still buys:
a single parameterised definition of the stack that you render per environment,
with `ix_lib`'s dependency helpers handling the MariaDB wiring, healthchecks and
the permission-fixing init container for you.

## Prerequisite: publish the image

Unlike `deploy/docker-compose.yml` (which can `build:` locally), this template only
references a pre-built image (`ix_values.yaml` → `images.image`). Build and push it
first:

```bash
docker build -t ghcr.io/undkonsorten/t3monitoring:13.4.33 -f deploy/Dockerfile .
docker push ghcr.io/undkonsorten/t3monitoring:13.4.33
```

Adjust the registry/tag if you host it elsewhere.

**GHCR packages are private by default**, and a private package is indistinguishable
from a missing one when you check without credentials — anonymous pulls and the
package web page both come back 404/`DENIED`. Don't read that as "the push failed";
check with `docker manifest inspect <image>` while logged in.

TrueNAS therefore can't pull it until you do one of:

* **Give TrueNAS credentials** (keeps the package private) — **Apps → Configuration
  → Sign-in to a Docker registry → Add Registry**, choose **Other Registry**, URI
  `https://ghcr.io`, username = your GitHub username, password = a PAT with
  `read:packages`.
* **Make the package public** — GitHub → the package → Package settings → Change
  visibility. No credentials needed anywhere after that.

## Publishing an update

Whenever you rebuild the image:

1. Push the new image tag.
2. Update `ix_values.yaml` → `images.image.tag`.
3. Bump `app.yaml` → `app_version` (upstream version) **and** `version` (catalog
   revision). TrueNAS isn't watching these — nothing syncs this repo — but keeping
   them accurate is what makes this repo a trustworthy record of what's deployed.
4. Commit and push this repo, then bump the submodule pointer in the parent.
5. Re-render with your production values file (see "Installing this on TrueNAS")
   and paste the result into the app's **Edit** YAML in TrueNAS.
6. If the schema changed, run `database:updateschema` + `cache:flush` in the web
   container afterwards.

There is no "Refresh Catalog" step and no update badge — that only exists for apps
that come from a catalog TrueNAS syncs, which this isn't.

## Installing this on TrueNAS

> **You cannot register this as a catalog.** TrueNAS removed custom/third-party
> catalog support when apps moved from Kubernetes/Helm to Docker in 24.10 "Electric
> Eel". The old **Apps → Manage Catalogs → Add Catalog** screen is gone: the current
> API ([`catalog.update`](https://api.truenas.com/v25.10/api_methods_catalog.update.html))
> is a singleton that accepts only `preferred_trains` — there is no `catalog.create`
> and no repository/branch field. On 24.10+ the only ways to run a non-official app
> are the Custom App wizard and **Install via YAML**
> ([docs](https://apps.truenas.com/managing-apps/installing-custom-apps/)).
>
> This catalog is written in the *new* `ix-dev/` Docker format (correct for 24.10+),
> while custom catalogs only ever worked on ≤24.04 — so there is no TrueNAS version
> that can consume it as a catalog. What it's still good for: being the single
> maintained definition of the container topology, which you **render to plain
> compose** and paste into Install via YAML.

So instead of registering it, render it:

1. **Write a production values file.** Start from
   `templates/test_values/basic-values.yaml`, replace the dummy secrets with real
   ones, and point the storage at real dataset paths. Keep it out of git — it holds
   passwords and the encryption key.

   Set the paths via the `ix_volumes:` mapping and leave the storage `type` as
   `ix_volume`:

   ```yaml
   ix_volumes:
     fileadmin: /mnt/<pool>/apps/t3monitoring/fileadmin
     var_log:   /mnt/<pool>/apps/t3monitoring/log
     db_data:   /mnt/<pool>/apps/t3monitoring/db

   storage:
     fileadmin: {type: ix_volume, ix_volume_config: {dataset_name: fileadmin, create_host_path: true}}
     var_log:   {type: ix_volume, ix_volume_config: {dataset_name: var_log,   create_host_path: true}}
     db_data:   {type: ix_volume, ix_volume_config: {dataset_name: db_data,   create_host_path: true}}
     additional_storage: []
   ```

   **Do not switch these to `type: host_path`.** Both render to the same bind
   mounts, but `host_path` drops the `permissions` container from the output
   entirely — and that container is the only thing that chowns `fileadmin` and
   `var_log` to `33:33` (www-data) before the app starts. With `host_path` you get
   three services instead of four and have to fix ownership by hand.

2. **Render it to plain compose:**

   ```bash
   git clone https://github.com/truenas/apps.git /tmp/truenas-apps
   cp -r ix-dev/community/t3monitoring /tmp/truenas-apps/ix-dev/community/
   cp prod-values.yaml /tmp/truenas-apps/ix-dev/community/t3monitoring/templates/test_values/
   cd /tmp/truenas-apps
   python3 .github/scripts/ci.py --app t3monitoring --train community \
           --test-file prod-values.yaml --render-only=true
   ```

   Output: `ix-dev/community/t3monitoring/templates/rendered/docker-compose.yaml`.
   The render happens inside `ghcr.io/truenas/apps_validation:latest`, so you only
   need `docker`, `jq`, `openssl` and `pyyaml` locally — no Jinja/pydantic setup.

3. **Sanity-check the output** before pasting: four services (`mariadb`,
   `permissions`, `t3monitoring`, `t3monitoring-scheduler`), all mounts are bind
   mounts to your real dataset paths, `volumes: {}` is empty, and
   `docker compose -f <rendered> config -q` passes. The top-level `x-portals` /
   `x-notes` / `x-action-required` keys are TrueNAS metadata — compose ignores `x-`
   extensions, so leave them or strip them, either works.

4. **Paste it in:** TrueNAS → **Apps → Discover → ⋮ → Install via YAML**,
   Application Name `t3monitoring`. If it fails, the UI error is generic — read
   `/var/log/app_lifecycle.log` on the host for the actual Docker error (common
   cause: the published port is already taken).

What you give up versus a real catalog app: the config form (edit the YAML instead)
and the "Update available" badge (see "Publishing an update" below).

## Importing a database fixture/dump

Unlike `deploy/docker-compose.yml` (which mounts a dump into `mariadb`'s
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
   the password with the `db_root_password` from the values file you rendered with
   (also visible in the app's YAML: TrueNAS UI → the app → Edit).

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

`deploy/docker-compose.yml` (plain compose) and this catalog are two independent
ways to run the same image — pick one, not both. **Both now end at the same place:
Install via YAML.** They differ in where the YAML comes from:

* `deploy/` — you hand-edit the compose file (swap `build:` for `image:`, inline the
  `.env` values, point the volumes at datasets) and paste it. Fewer moving parts,
  more hand-editing per environment. See `../deploy/README-TrueNAS.md`.
* this catalog — you keep a values file per environment and *render* the compose
  from a parameterised template. More structure up front; per-environment changes
  are a values edit and a re-render instead of hand-surgery on YAML.
