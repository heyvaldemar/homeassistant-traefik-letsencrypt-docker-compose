# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.6.0] - 2026-09-04

### Fixed

- **A backup interrupted halfway no longer looks like a good one.** The loop
  already renamed a failed dump to `.failed` so nothing would restore from it,
  but that rename only runs if the shell lives long enough to reach it. Stop
  the container mid-dump and it does not: the truncated file keeps the name a
  finished backup would have, and it is the newest one, which is exactly what
  the restore script and the end-to-end test pick. Every backup is now written
  to `<name>.partial` and renamed only after the dump succeeds, so the real
  name never exists unless the file behind it is complete. Verified by killing
  a dump in flight: before, the restore path selected a file that failed
  `gzip -t`; after, it finds nothing to select.

## [1.5.0] - 2026-09-03

### Added

- **Per-image version overrides.** Every pin in the `x-images` block is
  now `${<PREFIX>_IMAGE_TAG:-repo:${<PREFIX>_IMAGE_VERSION:-tag@sha256:digest}}`.
  Set `<PREFIX>_IMAGE_VERSION` in `.env` to run a different version of one
  image while every other pin stays as tested (Compose pulls that tag
  without a digest), or `<PREFIX>_IMAGE_TAG` to replace the whole
  reference as before. A deployment that sets neither is unchanged. The
  freshness job, the Trivy matrix and the fleet digest automation resolve
  the nested default before reading a pin. Needs Docker Compose v2.5 or
  newer (2022): v2.0 to v2.4 leave the inner `${...}` unexpanded and
  `docker compose up` fails with an invalid reference instead of
  deploying something unexpected.

### Changed

- `homeassistant/home-assistant` 2026.8.3 to 2026.9.0.

## [1.4.0] - 2026-09-02

### Security

- **Container hardening.** Every service runs with
  `security_opt: no-new-privileges:true` (no privilege escalation via
  setuid binaries even if a process escapes its initial capability
  set). Infrastructure containers (the reverse proxy, databases,
  caches, backups) drop every Linux capability and add back only what
  their entrypoints need (bind :80/:443, chown a data directory, drop to
  the service user). Application containers keep the default capability
  set: upstream images assume it, and a wrong guess there is a boot loop
  in production, not a hardening win. CI boots the stack under these
  settings on every push.

### Added

- **`tests/e2e-backup-restore.sh`**: scenarios against the live stack,
  run by CI on every push: the required-variable guard fires, a backup
  set is produced, the archive is readable, the database copy passes `PRAGMA integrity_check`, a cycle that cannot
 write its archive is reported as `FAILED`, **restore 
  replaces the data** (the application is stopped, the baseline database copy is put back, and a row inserted after the baseline is gone), and pruning removes only old files.

## [1.3.1] - 2026-09-02

### Fixed

- A database file that does not exist yet (the application creates it on
  first start) is skipped with a note instead of being reported as a
  failed backup; the first cycle after a fresh install no longer logs
  `FAILED`.

## [1.3.0] - 2026-09-02

### Added

- **A `backups` service** for the configuration directory (integrations, automations, secrets) and the recorder database: on a loop it takes a consistent copy of each SQLite database (`home-assistant_v2.db`) through Python's `sqlite3` backup API - no application stop - and a `tar.gz` of the rest of the data directory (live database files excluded), logs `OK` or `FAILED` per artefact (a failed archive is kept as `.failed`), and prunes only its own files. Schedule knobs (`HOMEASSISTANT_BACKUP_INIT_SLEEP`, `HOMEASSISTANT_BACKUP_INTERVAL`, `HOMEASSISTANT_BACKUP_PRUNE_DAYS`, path and names) have defaults listed in `.env.example`.
- **`homeassistant-restore-data.sh`**: interactive restore of a backup set: stops homeassistant, unpacks the data archive, restores each database copy, starts homeassistant.
- CI waits for the first backup cycle and proves the archives are readable and the database copy passes `PRAGMA integrity_check`.

## [1.2.0] - 2026-09-02

### Added

- **Resource limits on every service, as `.env`-overridable defaults.**
  Each service now carries memory and CPU limits plus reservations
  (`<SERVICE>_MEMORY_LIMIT`, `_CPU_LIMIT`, `_MEMORY_RESERVATION`,
  `_CPU_RESERVATION`, defaults listed in `.env.example`). Set any of
  them in `.env` and the override survives every `git pull`. The
  defaults are what CI boots the stack under, so they are known to be
  enough for a fresh install; raise a limit if a service is OOM-killed
  under your real load (`docker inspect` shows `OOMKilled=true`).

## [1.1.0] - 2026-09-02

### Added

- **`update.sh`**: unattended updates to the newest tagged release,
  and nothing else: a tag is cut only after CI has booted the pinned
  images and passed the smoke tests, so "update to the latest tag" means
  "update to a combination a machine has already run". It refuses to
  cross a major version on its own (`--allow-major` after reading the
  notes), refuses a checkout with local modifications, and supports
  `--dry-run`. Put it on a cron timer for hands-off minor/patch updates.

## [1.0.0] - 2026-09-01

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose)
v1.2.0.

### Fixed

- **Home Assistant state was ephemeral**: only `configuration.yaml` was
  mounted, so `/config`: the HA database, users, automations, dashboards,
  and integrations, lived inside the container and was wiped on every
  recreate or image update. `/config` is now a named volume
  (`homeassistant-data`); the tracked `configuration.yaml` stays
  bind-mounted over it for the reverse-proxy settings.

### Changed

- **Home Assistant 2023.9 (three years old) → 2026.8.3**, **Traefik 3.2 →
  3.7** (3.2's Docker client cannot talk to Docker Engine 29), both pinned
  by `tag@sha256:digest` in the compose `x-images` block.

### Added

- **Deployment Verification workflow**: actionlint; Trivy scans of both
  pinned images; weekly `check-pin-freshness` (digest drift + Home
  Assistant and Traefik release lag); and a deploy-and-test job that
  boots the stack and requires the HA UI to answer through Traefik with
  the trusted-proxy configuration loaded.
- `.env.example`; `.env` untracked and gitignored (it held no secrets in
  this template, but the fleet convention applies).

[Unreleased]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.6.0...HEAD
[1.6.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.5.0...v1.6.0
[1.5.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.4.0...v1.5.0
[1.4.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.3.1...v1.4.0
[1.3.1]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.3.0...v1.3.1
[1.3.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
