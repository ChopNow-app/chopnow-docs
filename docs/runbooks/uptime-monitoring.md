# External uptime monitoring

External polling of public surfaces, so a real-world outage is
detected by us before a pilot user has to WhatsApp the founder.

!!! info "Status — 2026-05-24"
    UptimeRobot free tier. **3 monitors live** (api `/health` + api
    `/ready` + consumer PWA with keyword check), 5-minute interval,
    email alerts to the founder. The docs-site monitor is documented
    in the table below but deliberately not deployed — a docs outage
    doesn't hurt users, so it's not worth the dashboard noise.
    WhatsApp escalation deferred (see "future work" below).

## Why external monitoring

Internal probes (`/health`, `/ready`, Loki uptime queries) only fire
if the system is up enough to push them. A blanket outage — droplet
down, Cloudflare DNS broken, Caddy crashed, Vercel routing wrong —
is invisible from inside the system. External polling from a third
party catches the failure modes that matter to a user.

## What we monitor

| Monitor | URL | Interval | Pass condition | Why |
|---|---|---|---|---|
| **API liveness** | `https://api-staging.tchopnow.app/health` | 5 min | HTTP 200 | Cheapest signal — process is alive + Caddy is forwarding |
| **API readiness** | `https://api-staging.tchopnow.app/ready` | 5 min | HTTP 200 | Adds Postgres + Redis health to the above (deep probe) |
| **Consumer PWA** | `https://app.tchopnow.app` | 5 min | HTTP 200, contains `TChopNow` | Catches Vercel routing / alias breakage |
| **Docs site** *(not deployed)* | `https://chopnow-app.github.io/chopnow-docs/` | 30 min | HTTP 200 | Lowest priority — Pages outage doesn't hurt users; add later if Pages becomes load-bearing for partners |

## Provider — UptimeRobot (free tier)

| | UptimeRobot Free | Why this choice |
|---|---|---|
| Monitor count | 50 | We use 4 |
| Interval | 5 min minimum | Acceptable RTO for pilot |
| Alert channels | Email | Founder's inbox + phone push |
| WhatsApp/SMS | Paid only ($7/mo) | Email push notification on phone is operationally equivalent |
| Cost | $0 | $0 |
| Sign-up time | 2 min | — |

**Alternatives considered**: Better Stack (formerly Better Uptime) free tier
is smaller; Healthchecks.io is better for cron monitoring than HTTP uptime;
self-hosted (Uptime Kuma) adds another thing to monitor.

## First-time setup (one-off, ~30 min)

### 1. Create the UptimeRobot account
1. Go to <https://uptimerobot.com>
2. Sign up with the founder's email
3. Verify the email (UptimeRobot sends a confirmation link)

### 2. Configure the 4 monitors

For each row in the "What we monitor" table above:

1. Dashboard → **+ New Monitor**
2. **Monitor Type**: HTTPS
3. **Friendly Name**: e.g. `chopnow-api-staging-health` (matches the row label)
4. **URL**: the URL from the table
5. **Monitoring Interval**: 5 min (or 30 min for the docs monitor)
6. **Monitor Timeout**: 30 sec (default)
7. **Advanced**:
   - For `/ready` and `/health` — **HTTP Method**: GET
   - For the PWA monitor — **Keyword Type**: "Should be present" → **Keyword**: `TChopNow`
8. **Alert Contacts**: tick the founder's email (created automatically on signup)
9. **Create Monitor**

Repeat for all 4.

### 3. Configure alert contact

By default UptimeRobot uses the signup email. Recommended additions:

- **Add a phone push contact** via the UptimeRobot mobile app (free, iOS + Android) — this gives near-instant push notifications without paying for SMS/WhatsApp
- Optionally: add a second email (e.g. ops backup) so the founder isn't a SPOF on alert delivery

### 4. Set the alert threshold

- **"Send alert when": Down for 1 cycle** (5 min) — first failure pages immediately
- **"Re-notify every": 30 min** until resolved
- **"Send 'Up' notification": yes** — confirmation of recovery

These defaults match a "wake me up when the API is down for >5 min"
SLA. Tune up if you start getting false positives (Cloudflare blips,
DigitalOcean droplet maintenance windows).

## What to do when an alert fires

1. **Verify it's real** — open the URL in a browser. UptimeRobot has
   ~1% false-positive rate from regional network glitches; a quick
   eyeball confirms.
2. **Check the most-likely culprits in this order**:
   - Was a deploy in progress? (GitHub Actions tab — `CD — staging` workflow)
   - Is the droplet up? (`ssh -i ~/.ssh/do_ed25519 root@157.230.125.224 'systemctl status'`)
   - Is the API container up? (`docker ps | grep chopnow-staging-api`)
   - Is Caddy up? (`systemctl status caddy`)
   - Is Cloudflare DNS resolving? (`dig api-staging.tchopnow.app`)
3. **Mitigate first, diagnose second**:
   - Container down → `docker compose -f /opt/chopnow/staging/docker-compose.yml up -d`
   - Caddy down → `systemctl restart caddy`
   - Deploy broke the API → see [`runbooks/rollback.md`](rollback.md)
   - DB / Redis unreachable → check `docker logs chopnow-staging-db` / `redis`
4. **Post-incident**:
   - Add a one-liner to a recurring `runbooks/incidents.md` (what fired, what fixed it, how long)
   - If the same alert fires twice in a week, write a real runbook for that failure mode
   - Update this doc if the cause was something we didn't expect

## Future work

- **WhatsApp alerting** — currently email-only. WhatsApp alerts require
  UptimeRobot paid ($7/mo) OR a self-hosted bridge (email → IFTTT/Zapier
  → WhatsApp via Twilio). Defer until founder has a real "I missed an
  alert because email push was off" incident.
- **Synthetic transaction monitoring** — UptimeRobot only checks "did the
  URL return 200?" It doesn't verify "can I actually log in and place
  an order?" That's the Newman smoke test in
  [chopnow-api#296](https://github.com/ChopNow-app/chopnow-api/issues/296)
  — same trigger / different layer.
- **Hetzner production endpoints** — when the production environment
  lands, add monitors for `api.tchopnow.app/health` + `/ready`.
- **Pingdom / Better Stack** alternative when team grows — multi-region
  probes, status page, on-call rotation. Worth ~$30/mo at that point.

## Where the alerts go

Email: `kanmegnea@gmail.com` (founder primary)
UptimeRobot dashboard: <https://uptimerobot.com/dashboard>
(Pin in browser; status board for a glance during an incident)

## Related

- [Observability stack](../architecture/observability.md) — internal probes (Loki, Prometheus, Sentry) complement this external view
- [Deploy to staging](deploy-staging.md) — what got deployed when an alert fires
- [Rollback](rollback.md) — the runbook a fired alert routes into
