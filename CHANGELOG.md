# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

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

[Unreleased]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
