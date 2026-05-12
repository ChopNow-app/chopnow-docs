# Operator / non-dev onboarding

You don't need to read code to be effective on this project. The most valuable contributions from non-engineers right now: vendor recruitment, terrain landmarks, ops monitoring, user support.

## Access you need

1. **GitHub account** with access to the `ChopNow-app` org. Ask the founder; they invite you.
2. **DigitalOcean console** (`cloud.digitalocean.com/projects`) — read-only at first, full access if you're on-call. Used to see if staging is up, check logs, see the bill.
3. **Hetzner console** (when prod launches) — same pattern as DO.
4. **Twilio console** (`console.twilio.com`) — to monitor OTP delivery, sandbox membership, the prod WhatsApp Business sender.
5. **Cloudflare** (when domain is enregistered) — DNS for `tchopnow.app`.
6. **Supabase** — read-only access to the waitlist table on the landing page.
7. **Vercel** — read-only on the landing + frontend deploys.

## What to read first

| Doc | Why | Where |
|---|---|---|
| [Overview](overview.md) | What the product does, who the actors are | here |
| Business model | Margins, charges, break-even — the financial reality | `_bmad-output/planning-artifacts/business-model.md` |
| Roadmap | When each phase ships, KPIs per phase | `_bmad-output/planning-artifacts/roadmap-poc-to-scale.md` |
| Terrain tasks | TERRAIN-1, TERRAIN-2, TERRAIN-3 — physical-world work | `_bmad-output/implementation-artifacts/todo.md` |
| [Glossary](../reference/glossary.md) | Cameroon-specific terms (cuisinier, maquis, mboué, etc.) | here |

## Tools you'll use daily

- **WhatsApp** — primary channel for vendor + livreur outreach, support, OTP testing
- **GitHub Issues** — every story has an issue, comment progress there
- **Excel / Google Sheets** — landmark data collection (TERRAIN-1), vendor onboarding sheet (TERRAIN-3)
- **Telegram or WhatsApp group** — internal team chat (link from the founder)

## Routine tasks

### Adding a vendor to the system

1. Founder or operator does the in-person pitch.
2. Vendor agrees → collect name, type (informal / semi-formal / restaurant), quartier, point of reference, MoMo number, WhatsApp number.
3. Photo session — 5–10 plats with prices.
4. Admin creates the vendor record in the `/admin` PWA (once Story 2.10 ships — Sprint 1 W3).

### Re-joining the Twilio sandbox (today)

Until prod WhatsApp Business is approved (todo.md B2), all OTP testing goes through the **sandbox**. Sandbox membership expires every **72h**. To re-join: from the WhatsApp number you want to receive OTPs on, send `join flag-excitement` to `+14155238886`. See [runbook](../runbooks/twilio-sandbox-rejoin.md).

### Checking if staging is up

```
curl http://157.230.125.224:3001/health
# → {"status":"ok","uptime":...,"timestamp":"..."}
```

Or via the DO console: `Projects → ChopNow → chopnow-staging → Graphs`.

If `uptime` is < 60s, a deploy just ran. If the curl times out, the droplet is down — page the on-call dev.

## Where to NOT touch (without a developer)

- The DO API token (it's in GitHub secrets — never paste in chat or share)
- The droplet's `/opt/chopnow/staging/.env` — overwritten by every deploy
- The chopnow-api repo's `develop` / `staging` / `main` branches — protected by rulesets, all changes through PRs

## Where to read up after this

- [Architecture overview](../architecture/overview.md) — to understand the moving parts
- [Decisions](../decisions/index.md) — why we picked Twilio, Hetzner, PWA, etc.
