# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

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

- **`update.sh`** — unattended updates to the newest tagged release,
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
  mounted, so `/config` — the HA database, users, automations, dashboards,
  and integrations — lived inside the container and was wiped on every
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

[Unreleased]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
