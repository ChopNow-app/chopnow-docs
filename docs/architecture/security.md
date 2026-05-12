# Security

How we keep secrets out of source, vulnerabilities out of `develop`, and attackers off the runtime.

## CI gates (chopnow-api)

Every push to a feature branch and every PR runs Tier 1:

- `eslint --fix` (zero errors)
- `tsc --noEmit` (zero errors)
- `npm test` (unit, 100% green)
- `npm audit --audit-level=high` (blocks on any high+ advisory)
- `gitleaks detect` on git history (blocks on any secret pattern)
- `semgrep scan --config=p/typescript --config=p/security-audit` (zero findings on `src/`)

Every PR additionally runs Tier 2:

- `npm run test:integration` (testcontainers PostGIS, Nest+supertest)
- `npm run test:e2e` (full AppModule + ephemeral Postgres, real Twilio mocked at provider boundary)
- `nest build` (catches type drift in the SWC pipeline)

PRs targeting `staging` or `main` add Tier 2.5:

- Native `docker build` (proves the Dockerfile builds — caught a regression in the first staging deploy)

PRs targeting `main` add Tier 3:

- QEMU-emulated `linux/arm64` build (production target — Hetzner CAX11 Ampere)
- `aquasecurity/trivy-action` scan of the built image at HIGH,CRITICAL severity, `--exit-code 1`

See [`chopnow-api/.github/workflows/ci.yml`](https://github.com/ChopNow-app/chopnow-api/blob/develop/.github/workflows/ci.yml) for the wiring.

## Dependabot

`.github/dependabot.yml` runs weekly on Monday 06:00 Africa/Douala:

- npm version updates grouped by `dependency-type` (production vs development) — ~2 PRs/week instead of 30
- `github-actions` ecosystem tracked separately
- Security updates bypass the schedule and arrive daily (GitHub default)

Combined with the Tier 1 `npm audit` gate, the loop is closed: Dependabot opens auto-fix PRs, the audit gate prevents PRs from regressing the baseline.

## Runtime hardening (production)

| Layer | Control |
|---|---|
| **Network** | Cloudflare WAF + DDoS in front. Direct VPS IP blocked on 80/443 via firewall rules pinned to Cloudflare IP ranges (FR-160). |
| **TLS** | Let's Encrypt certs via nginx (planned). HSTS, modern cipher suite only. |
| **SSH** | Password auth disabled. Root by key only. `fail2ban` jail on sshd. Dedicated `deploy` user (uid 1000) with restricted sudo (NOPASSWD only on `/usr/bin/systemctl`). |
| **Docker** | API container runs as non-root user `nestjs` (uid 999). `--read-only` filesystem (planned). |
| **Database** | Postgres bound to `localhost:5432` only. Strong random password generated per env (32-char openssl rand). |
| **Redis** | Bound to `localhost:6379` only. No password yet (compose network is isolated). |
| **Helmet** | Strict CSP, HSTS, X-Frame-Options, etc. Set in `chopnow-api/src/main.ts`. |
| **Body size** | `express.json({ limit: '1mb' })` — payload bomb mitigation. |
| **Throttler** | Global 100 req/min/IP via `@nestjs/throttler`. Per-endpoint stricter limits planned for `/auth/*` (3 OTP/30min/phone). |

## Secrets management

- **Never in code.** `.env.example` ships placeholders only; `.env` is gitignored.
- **GitHub environments scope secrets.** `staging` env holds `STAGING_HOST`, `STAGING_USER`, `STAGING_SSH_KEY`, `STAGING_DB_PASSWORD`, `STAGING_JWT_ACCESS_SECRET`, `STAGING_JWT_REFRESH_SECRET`. `production` env will mirror.
- **Operator personal keys never go in CI.** Dedicated `chopnow-staging-deploy` keypair generated at provision time; private half lives only in GitHub `staging` env, public half in `/home/deploy/.ssh/authorized_keys` on the droplet.
- **Rotation cadence** — quarterly hygiene; immediately on exposure (paste in chat, accidental commit, contributor offboarding). See [rotate-do-token](../runbooks/rotate-do-token.md).
- **Terraform tokens** passed via `TF_VAR_*` env var at apply time — never written to `terraform.tfvars` (gitignored anyway).

## Auth flow

- **JWT HS256.** Access token 24h, refresh token 30d.
- **OTP via Twilio WhatsApp + SMS fallback.** 6-digit code, argon2id-hashed. TTL 5 min. Max 3 verify attempts before lockout.
- **Refresh rotation** (Story 1.2 — landing next). Each refresh emits new pair + revokes the old. Re-use of revoked → revoke entire family (forces re-OTP). Modeled after the OAuth refresh-token-rotation pattern.
- **Global guards** (`JwtAuthGuard`, `RolesGuard`) registered as `APP_GUARD`. Every route is JWT-protected by default; opt out with `@Public()`.

## Audit log (planned)

Every state-changing admin action lands in an `AuditLog` table (Sprint 3, Epic 6). Append-only, immutable, replicated to off-site backup. Covers:

- vendor suspension / unsuspension
- commission rate changes
- order cancellations / refunds
- manual MoMo settlement adjustments

## Known gaps

| Gap | Tracking | Plan |
|---|---|---|
| nginx + TLS not yet deployed on staging (api on 127.0.0.1:3001 internal only) | TD post-Sprint-1 | Lands with prod cutover |
| `_bmad-output/` planning dir not versioned | TD-5 | Decide: sub-dir in chopnow-api, dedicated repo, or stay local |
| Twilio sandbox membership 72h expiry (dev only — prod has approved WhatsApp Business sender) | todo.md B2 | Submit Meta verification |
| Postgres backups via Hetzner snapshots only; no `pg_dump` cron yet | post-Sprint-1 | Add to bootstrap.sh |
