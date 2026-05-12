# Environments

## dev (local)

| | |
|---|---|
| **API URL** | `http://localhost:3001` |
| **Swagger** | `http://localhost:3001/api/docs` |
| **Postgres** | `localhost:5433` (mapped from container 5432) |
| **Redis** | `localhost:6379` |
| **Frontend (when running locally)** | `http://localhost:3000` |
| **Twilio mode** | Sandbox; opted-in test numbers only |
| **Campay mode** | Sandbox (`demo.campay.net/api`) |

Run via `npm run start:dev` in chopnow-api + `npm run dev` in chopnow-app.

## staging (DigitalOcean Basic)

| | |
|---|---|
| **API URL** | `http://157.230.125.224:3001` (bound to 127.0.0.1, accessible only via SSH tunnel for now) |
| **Public access** | None yet — nginx + TLS pending |
| **Region** | Frankfurt (`fra1`) |
| **Spec** | 1 vCPU / 2 GB / 50 GB SSD, x86_64, backups enabled |
| **Cost** | ~$14.40/month (covered by $200 DO new-account credit until 2026-06-26) |
| **Lifetime** | Tear down at end of June 2026 once Hetzner prod is up |
| **Database** | Postgres + PostGIS inside compose, volume `chopnow_staging_pgdata` |
| **Redis** | Inside compose, no persistence |
| **Twilio mode** | Sandbox (same as dev) |

Secrets live in chopnow-api's `staging` GitHub environment:

- `STAGING_HOST` — droplet IPv4
- `STAGING_USER` — `deploy`
- `STAGING_SSH_KEY` — dedicated deploy private key (ed25519)
- `STAGING_DB_PASSWORD` — random 32-char
- `STAGING_JWT_ACCESS_SECRET` — random 64-char base64
- `STAGING_JWT_REFRESH_SECRET` — random 64-char base64

Twilio / Campay / R2 / VAPID / Resend creds are empty for staging (OTP runs in dev-log mode in the API logs).

## production (Hetzner CAX11) — planned

| | |
|---|---|
| **API URL** | `https://api.chopnow.app` (planned; behind Cloudflare) |
| **Region** | Falkenstein (`fsn1`) |
| **Spec** | 2 vCPU Ampere ARM / 4 GB / 40 GB SSD, arm64, backups enabled |
| **Cost** | ~€4.55/month (~3 100 FCFA) |
| **Provision date** | End of June 2026 |
| **Lifetime** | Indefinite (upgrade in place to CX32 / CX42 as traffic grows) |
| **Frontend** | `https://chopnow.app` on Vercel |
| **Twilio mode** | Production WhatsApp Business sender (pending Meta approval — `todo.md` B2) |
| **Campay mode** | Live (pending Go Live submission — `todo.md` B1) |

Secrets will live in chopnow-api's `production` GitHub environment, mirroring staging's shape with `PROD_*` names.

## Domain ownership

- **chopnow.app** — to be registered (`todo.md` B0). Cloudflare for DNS.
- **tchopnow.com** — already registered (Cloudflare, expires Dec 2026). Currently parked.
- **tchopnow.cm** — to investigate via Camtel registry.

## Third-party service accounts

| Service | Where | Notes |
|---|---|---|
| GitHub org | https://github.com/ChopNow-app | Repos + Issues + Actions + Pages |
| DigitalOcean | https://cloud.digitalocean.com/projects | Project "ChopNow" — staging only |
| Hetzner Cloud | https://console.hetzner.cloud | Project "ChopNow" — prod (pending) |
| Twilio Console | https://console.twilio.com | Sandbox + Voice numbers + statuses |
| Campay | https://demo.campay.net | Sandbox + Go Live form |
| Cloudflare | https://dash.cloudflare.com | DNS + WAF (once domain registered) |
| Cloudflare R2 | https://dash.cloudflare.com/?to=/:account/r2 | Media bucket (pending) |
| Vercel | https://vercel.com/chopnow-app | Frontend + landing |
| Supabase | https://supabase.com | Waitlist landing only |
| Sentry | https://sentry.io | Error monitoring (planned) |
| Mapbox | https://account.mapbox.com | Map tiles token (planned) |
