# ChopNow Docs

Food delivery PWA for Douala, Cameroon. Soft launch July 2026.

This site collects everything someone joining the project needs to be productive — system design, stack decisions, runbooks, and onboarding guides. **Always Markdown, always edited via PR.**

## Quick links

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg .middle } **Just joined?**

    ---

    Start with the [overview](getting-started/overview.md), then [dev setup](getting-started/onboarding-dev.md) if you'll write code or [operator setup](getting-started/onboarding-non-dev.md) otherwise.

-   :material-sitemap:{ .lg .middle } **Need the technical picture?**

    ---

    [Architecture overview](architecture/overview.md) → [stack rationale](architecture/stack.md) → [infrastructure](architecture/infrastructure.md).

-   :material-decision:{ .lg .middle } **Why did we choose X?**

    ---

    See the [decision records (ADRs)](decisions/index.md). One file per major call, with context + alternatives.

-   :material-tools:{ .lg .middle } **Need to do an ops task?**

    ---

    [Runbooks](runbooks/index.md) cover deploy, rollback, token rotation, Twilio sandbox re-join, etc.

</div>

## Project at a glance

| | |
|---|---|
| **Product** | Food delivery PWA, cash + MoMo, ~50 cmd/jour at break-even (M5–M6) |
| **First market** | Douala (Makepe, Bonamoussadi, Logpom) → Yaoundé Sept 2027 |
| **Stack** | Next.js 14 PWA · NestJS API · PostgreSQL+PostGIS · Redis · Twilio · Campay |
| **Hosting (prod)** | Hetzner CAX11 Falkenstein (planned, end of June 2026) |
| **Hosting (staging)** | DigitalOcean Basic 1 vCPU / 2 GB Frankfurt (live now) |
| **Repos** | [`chopnow-api`](https://github.com/ChopNow-app/chopnow-api) · [`chopnow-app`](https://github.com/ChopNow-app/chopnow-app) · [`chopnow-infra`](https://github.com/ChopNow-app/chopnow-infra) · [`chopnow-docs`](https://github.com/ChopNow-app/chopnow-docs) · [`chopnow-landingpage`](https://github.com/ChopNow-app/chopnow-landingpage) |

See [Reference → Repositories](reference/repositories.md) for the full map.
