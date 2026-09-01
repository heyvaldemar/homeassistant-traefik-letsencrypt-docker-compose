# Home Assistant + Traefik + Let's Encrypt — Docker Compose

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

Within a minute `https://${HOMEASSISTANT_HOSTNAME}` serves the onboarding wizard with a fresh Let's Encrypt certificate — create the owner account right away.

### What success looks like

```bash
docker compose -f homeassistant-traefik-letsencrypt-docker-compose.yml -p homeassistant ps
curl -fskL -o /dev/null -w "%{http_code}\n" "https://${HOMEASSISTANT_HOSTNAME}/"
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **\"400 Bad Request\" from HA.** The reverse-proxy trust settings in `configuration.yaml` aren't loaded — make sure the file is mounted (it is, unless you removed the bind) and restart the container after editing it.
- **Networks not found.** Step 2 was skipped.

## A note on hardware integrations

Container HA behind a reverse proxy is great for dashboards and network-based integrations. Integrations that need host hardware (Zigbee/Z-Wave sticks, Bluetooth) require extra device mappings — add a `devices:` section for your stick (e.g. `/dev/ttyUSB0`). Add-ons from the HA OS ecosystem are not available in the container edition; use separate containers instead.

## Supply chain trust

Two images — [`traefik`](https://hub.docker.com/_/traefik) and [`homeassistant/home-assistant`](https://hub.docker.com/r/homeassistant/home-assistant) — pinned to `tag@sha256:<digest>` as interpolation defaults in the compose `x-images` block. `git pull` alone delivers the tested combination; an `*_IMAGE_TAG` variable in `.env` overrides deliberately.

The weekly `check-pin-freshness` CI job re-resolves each pin against its registry and compares the pinned versions against the latest upstream releases. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Complete onboarding immediately after deploy** — the wizard is open until someone claims it.
- [ ] **Back up the `homeassistant-data` volume** — it holds the HA database, users, and automations. HA's built-in backups (Settings → System → Backups) land inside the same volume; copy them off-host.
- [ ] **Regenerate the Traefik dashboard hash** — never ship the placeholder.
- [ ] **Update deliberately** — HA releases monthly; read the breaking-changes section before bumping the pin.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/homeassistant-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every Monday at 06:00 UTC: actionlint, Trivy scans of both pinned images, the weekly freshness check, and a deploy-and-test job that boots the stack and requires the HA UI to answer through Traefik.

## Security Notes

- The bundled `configuration.yaml` enables `ip_ban_enabled` with a 5-attempt threshold and trusts the Docker network ranges as proxies — required for correct client IPs behind Traefik.
- Exposing HA to the internet is a real decision: it controls your home. Consider IP-allowlisting at your firewall or a VPN if you don't need public access.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
