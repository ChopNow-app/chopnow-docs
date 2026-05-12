# Stack rationale

Each tech we use, why we picked it, and what we considered before settling. Authoritative source: `_bmad-output/planning-artifacts/architecture.md` (v3, 2026-04-08).

## Frontend

| Choice | Alternatives considered | Why we picked it |
|---|---|---|
| **Next.js 14 (App Router)** | Remix, vanilla React, SvelteKit | Server Components reduce client JS payload (matters on 3G Douala); routing across consumer/livreur/vendeur/admin in one app |
| **Tailwind CSS** | Plain CSS modules, styled-components | Fast iteration, tiny CSS bundle in production |
| **Mapbox GL JS** | Google Maps, Leaflet | Best PWA performance on 3G; free tier covers our volume; vector tiles cache offline |
| **Web Push API + VAPID** | Firebase Cloud Messaging | Browser-native; no Google dependency; PWAs install + push without an app store |

## Backend

| Choice | Alternatives | Why |
|---|---|---|
| **NestJS 11** | Express bare, Fastify, Hono | DI + module boundaries match the Epic structure; OpenAPI generation; mature testing story |
| **TypeScript 5** | JS | Type safety for a solo dev; refactoring confidence |
| **Prisma 6** | TypeORM, Drizzle, raw SQL | Migrations are first-class; type-safe queries; `prisma migrate deploy` for prod is unambiguous |
| **Joi** for env validation | Zod | Already used by NestJS ConfigModule by default. Project uses `class-validator` for HTTP DTOs |
| **class-validator** for DTOs | Zod | NestJS-native; decorator-based; type inference via `class-transformer` |
| **argon2id** for password / OTP hashing | bcrypt | Memory-hard; modern recommendation |
| **JWT** (HS256, 24h access / 30d refresh) | sessions | Stateless API; works across PWA + future mobile if ever |

## Data layer

| Choice | Why |
|---|---|
| **PostgreSQL 16** | One DB for the whole MVP. Mature, fast, transactional |
| **PostGIS 3.4** | Geo queries for dispatch (`ST_DWithin`, `ST_Distance`); landmarks table |
| **Redis 7** | Rate limiting (Throttler), OTP TTL, BullMQ queue backing |
| **BullMQ** | Delayed jobs (livreur payout @ 21h MoMo), dispatch retries, status webhooks |

PostgreSQL + Redis run **on the same VPS as the API** until ~2 000 cmd/jour. Cost-fixed architecture. Not a microservices project.

## External services

| Service | What for | Cost profile | ADR |
|---|---|---|---|
| **Twilio** | WhatsApp OTP, SMS fallback, Voice proxy | ~10 000 FCFA/mois MVP → 60–100k at scale | [0002](../decisions/0002-twilio-not-africas-talking.md) |
| **Campay** | MTN MoMo + Orange Money collect | Per-tx fees; webhook driven | architecture.md §3.3 |
| **Cloudflare R2** | Media (vendor photos, delivery proofs) | Free 10 GB then $0.015/GB | architecture.md §3.6 |
| **Vercel** | Frontend hosting | Free tier sufficient until ~50k MAU | architecture.md §3.8 |
| **Sentry** | Error monitoring | Free tier < 5k errors/mois | architecture.md §3.9 |
| **Resend** | Admin password-reset emails | Free tier 3k/mois | architecture.md §3.10 |

## Things we considered and rejected

| Tech | Why not |
|---|---|
| **React Native** | App-store friction (review cycles, install drop-off). Wakelock API on Android Chrome handles GPS background tracking (POC-3 validated). |
| **AWS** | Post-credit pricing 3–5× Hetzner for our profile; egress charges; managed-service lock-in. See [ADR-0003](../decisions/0003-hetzner-not-aws.md). |
| **Africa's Talking SMS** | POC-2 showed Twilio was simpler (no separate vendor for WhatsApp); kept as optional bulk-SMS optimization later. |
| **Meta Cloud API direct** for WhatsApp | 15-day Business verification gate. Twilio fronts that. |
| **Firebase** for anything | Push handled via Web Push + VAPID; no need to drag in FCM. |
| **GitBook / Confluence** for docs | This MkDocs site is Markdown-only + free on GitHub Pages. Non-engineers can PR. |
| **Microservices** | At 50–2 000 cmd/jour, monolith on one VPS is cheaper, simpler, faster. Re-evaluate at scale. |
