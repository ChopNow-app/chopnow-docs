# ADR-0003 — Hetzner for production hosting, not AWS

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-05-10 |
| **Authors** | Founder + cost-modelling session |

## Context

The architecture v3 baseline assumed a single VPS holding the API, Postgres, and Redis until ~2 000 cmd/jour (FR NF-060 in PRD). Decision needed: which provider? AWS, GCP, DigitalOcean, Hetzner were the realistic shortlist.

Constraints:

- **Pre-launch, pre-revenue.** Every monthly recurring cost is real money out of the founder's pocket
- **Solo dev.** Time spent on infra is time NOT spent on features. Managed services are tempting, but their abstractions need learning + their billing is volatile
- **Africa-Europe RTT matters.** Servers in Frankfurt or Falkenstein give Douala ~200–300ms; US-East gives ~400ms+; Asia is worse
- **AWS Free Tier** offers $200 credit for 6 months. Tempting but small

## Decision

**Production runs on Hetzner CAX11 (Ampere ARM) in Falkenstein, single VPS, with snapshot backups.** ~€4.55/mois total.

The architecture stays mono-VPS — Postgres + Redis + nginx + API on one box — until traffic forces a vertical bump (CX22 → CX32 → CX42 → CX52, all one-click upgrades). We re-evaluate splitting services when we cross ~2 000 cmd/jour.

DigitalOcean is used for **staging only**, scoped to a 7-week window (until 2026-06-26), funded by their $200 new-account credit. After that we tear DO down.

AWS stays unused for the platform. We may use individual always-free AWS services (Lambda, SES, SNS) for one-off needs without committing the core to AWS.

## Alternatives considered

| Option | Dealbreaker |
|---|---|
| **AWS EC2 + RDS + ElastiCache** | Post-credit pricing ~$45/month minimum (t3.small + db.t3.micro + cache.t3.micro) vs Hetzner's ~€4. Egress charged at $0.09/GB after 1 GB/month vs Hetzner's 20 TB included. Lock-in via managed services. Setup complexity 5–10× Hetzner for one-VPS workload. |
| **DigitalOcean for prod** | $12–24/month vs Hetzner's €4 — 3–5× more for equivalent specs. DO basic CPU is shared. Reasonable if we wanted one provider for staging + prod but the cost gap is too much for the lifetime of the project. |
| **GCP Compute** | Same pattern as AWS — credit covers 90 days, then pricey. Egress hostile. No clear win vs Hetzner. |
| **Render / Railway / Fly** | Convenient but ~$40/month for our footprint. Premium for the dev UX, not the runtime. Re-evaluate once we have multiple services. |
| **Bare metal at OVH (Strasbourg)** | Cheap but minimum commit, ops overhead (no managed snapshots), too much for a solo dev. |

## Consequences

**Positive**

- **Cost: ~3% of total platform charges fixes.** Infra never becomes the limiting factor at our scale. See [Infrastructure](../architecture/infrastructure.md) cost breakdown.
- **Simple ops.** SSH + Docker Compose. No VPC, no IAM, no security groups, no NAT Gateway surprise bills.
- **Predictable.** Fixed monthly cost, not pay-per-request. Easy to budget.
- **20 TB egress included.** We could stream video and not exceed it.
- **One-click upgrade** between CX22 → CX32 → CX42 keeps the same architecture as we grow. No re-platforming.

**Negative**

- **No managed Postgres.** We run + back up + scale Postgres ourselves. Hetzner snapshots cover disaster recovery; `pg_dump` cron is a manual addition.
- **No native auto-scaling.** Not a problem at our scale, but if a viral spike hit we'd be vertical-scaling, not horizontal.
- **No global CDN by default.** Cloudflare in front mitigates this (free tier covers our volume).
- **Less ecosystem.** No SQS, no Kinesis, no SageMaker. We don't use those; revisit if we ever need them.

**Reversal cost** — medium-large. The app is portable (NestJS + Postgres + Redis = standard tooling), so we could move to AWS in ~1 week if forced. The pain would be re-doing the deploy pipeline and migrating Postgres. Not a one-way door.

## Follow-ups

- Hetzner provisioning lands end of June 2026 — `chopnow-infra/hetzner/production/` Terraform (placeholder today)
- Cloudflare in front: lock VPS firewall to Cloudflare IP ranges (FR-160)
- Re-evaluate at: 1 000 cmd/jour milestone, or any infra incident that costs > 4h
- Linked: [Infrastructure docs](../architecture/infrastructure.md), [ADR-0004 arm64](0004-arm64-prod-amd64-staging.md)
