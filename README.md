# Home Assistant + Traefik + Let's Encrypt on Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This repository deploys **Home Assistant** (container edition) behind **Traefik** with automatic **Let's Encrypt TLS**. The bundled `configuration.yaml` pre-configures the reverse-proxy trust settings HA needs to work behind Traefik, and `/config` persists in a named volume.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-home-assistant-using-docker-compose/](https://www.heyvaldemar.com/install-home-assistant-using-docker-compose/).

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose
cd homeassistant-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create homeassistant-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env

# 4. Deploy
docker compose -f homeassistant-traefik-letsencrypt-docker-compose.yml -p homeassistant up -d
```

Within a minute `https://${HOMEASSISTANT_HOSTNAME}` serves the onboarding wizard with a fresh Let's Encrypt certificate. Create the owner account right away.

### What success looks like

```bash
docker compose -f homeassistant-traefik-letsencrypt-docker-compose.yml -p homeassistant ps
curl -fskL -o /dev/null -w "%{http_code}\n" "https://${HOMEASSISTANT_HOSTNAME}/"
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **\"400 Bad Request\" from HA.** The reverse-proxy trust settings in `configuration.yaml` aren't loaded. Make sure the file is mounted (it is, unless you removed the bind) and restart the container after editing it.
- **Networks not found.** Step 2 was skipped.

## A note on hardware integrations

Container HA behind a reverse proxy is great for dashboards and network-based integrations. Integrations that need host hardware (Zigbee/Z-Wave sticks, Bluetooth) require extra device mappings. Add a `devices:` section for your stick (e.g. `/dev/ttyUSB0`). Add-ons from the HA OS ecosystem are not available in the container edition; use separate containers instead.

## Supply chain trust

Two images ([`traefik`](https://hub.docker.com/_/traefik) and [`homeassistant/home-assistant`](https://hub.docker.com/r/homeassistant/home-assistant)) pinned to `tag@sha256:<digest>` as interpolation defaults in the compose `x-images` block. `git pull` alone delivers the tested combination; an `*_IMAGE_TAG` variable in `.env` overrides deliberately.

The daily `check-pin-freshness` CI job re-resolves each pin against its registry and compares the pinned versions against the latest upstream releases. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Complete onboarding immediately after deploy**: the wizard is open until someone claims it.
- [ ] **Back up the `homeassistant-data` volume**: it holds the HA database, users, and automations. HA's built-in backups (Settings → System → Backups) land inside the same volume; copy them off-host.
- [ ] **Regenerate the Traefik dashboard hash**: never ship the placeholder.
- [ ] **Update deliberately**: HA releases monthly; read the breaking-changes section before bumping the pin.

## Unattended updates

Releases are the update channel: a tag is cut only after CI has built the pinned images, booted the full stack, and passed the smoke tests. `update.sh` moves a deployment to the newest tag and nothing else:

```bash
./update.sh --dry-run   # show what would be applied
./update.sh             # update within the current major and redeploy
```

Put it on a timer for hands-off minor/patch updates:

```bash
# crontab -e
17 5 * * *  /opt/homeassistant-traefik-letsencrypt-docker-compose/update.sh >> /var/log/homeassistant-update.log 2>&1
```

The script refuses to cross a MAJOR template version on its own: majors are breaking by definition and their release notes exist to be read. After reading them, `./update.sh ‑‑allow-major` performs the jump. It also refuses to touch a checkout with local modifications: your customization belongs in `.env`, which updates never overwrite.

This is deliberately a host-side script and not a container in the stack: an in-stack updater needs the Docker socket (root on the host) and turns "someone pushed to a repo" into "someone deployed to your machine" with no operator in the loop. A cron job under your own user updates only to tagged, CI-verified states and leaves the trust boundary where it was.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults: the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> ‑‑format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Backups

The `backups` container runs on a loop: an initial delay (`HOMEASSISTANT_BACKUP_INIT_SLEEP`, default 30m), then every `HOMEASSISTANT_BACKUP_INTERVAL` (default 24h) it takes a consistent copy of each SQLite database (`home-assistant_v2.db`) through Python's `sqlite3` backup API - no application stop - and a `tar.gz` of the rest of the data directory (live database files excluded), into the `homeassistant-backups` volume; files older than `HOMEASSISTANT_BACKUP_PRUNE_DAYS` (default 7) are pruned. Each artefact logs `... backup OK: <file> (<bytes> bytes)` or `FAILED` (kept as `<file>.failed`). Grep the log for `FAILED` from your monitoring.

**Verify backups are running:**

```bash
docker compose -p homeassistant logs backups | tail -5
docker compose -p homeassistant exec backups ls -la /srv/homeassistant/backups/
```

**Restore** a backup set with the interactive script (`chmod +x homeassistant-restore-data.sh` once): it stops homeassistant, unpacks the data archive over the data directory, restores each database from its consistent copy, and starts homeassistant again.

```bash
./homeassistant-restore-data.sh
```

**Off-host replication.** Backups live in a named volume on the same host. Bind-mount `HOMEASSISTANT_BACKUPS_PATH` to a directory covered by your off-host backup solution (restic, rclone, Borg, S3 sync).

## Container hardening

Every service runs with `security_opt: no-new-privileges:true`, so a process cannot gain privileges through setuid binaries even if it escapes its initial capability set. Infrastructure containers (the reverse proxy, databases, caches, backups) run with `cap_drop: [ALL]` and add back only what their entrypoints need: `NET_BIND_SERVICE` for Traefik to bind :80/:443, `CHOWN`/`SETUID`/`SETGID` (and friends) for database images to own their data directory and drop to their service user. Application containers keep the default capability set on purpose: upstream images assume it, and a wrong guess there is a boot loop in production rather than a hardening win. CI boots the stack under exactly these settings on every push, so what ships is what was tested.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC: actionlint, Trivy scans of both pinned images, the weekly freshness check, and a deploy-and-test job that boots the stack and requires the HA UI to answer through Traefik.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the smoke test. The scenario that matters most is the restore roundtrip: the application is stopped, the baseline database copy is put back, and a row inserted after the baseline is gone. The tests stop the application briefly and write into its data directory. Run them on a staging copy with short intervals in `.env` (`HOMEASSISTANT_BACKUP_INIT_SLEEP=15s`, `HOMEASSISTANT_BACKUP_INTERVAL=60s`), never on production.

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

## Security Notes

- The bundled `configuration.yaml` enables `ip_ban_enabled` with a 5-attempt threshold and trusts the Docker network ranges as proxies, required for correct client IPs behind Traefik.
- Exposing HA to the internet is a real decision: it controls your home. Consider IP-allowlisting at your firewall or a VPN if you don't need public access.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** · Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
