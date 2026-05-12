# Architecture overview

One backend, one frontend, two managed providers, two cloud hosts. Optimised for solo-dev maintenance and per-order unit economics.

## High-level diagram

```mermaid
flowchart TB
  subgraph Consumer["Consumer device (Tecno / Itel / Infinix)"]
    PWA[Next.js 14 PWA<br/>Service Worker · Web Push]
  end

  subgraph Vercel["Vercel (free tier)"]
    Front[chopnow-app build]
    Landing[chopnow-landingpage]
  end

  subgraph DO["DigitalOcean Basic 2 GB · Frankfurt (staging)"]
    StA[NestJS API]
    StPG[(PostgreSQL + PostGIS)]
    StR[(Redis)]
  end

  subgraph Hetz["Hetzner CAX11 · Falkenstein (prod — planned)"]
    PrA[NestJS API]
    PrPG[(PostgreSQL + PostGIS)]
    PrR[(Redis)]
    Nx[nginx · TLS · reverse proxy]
  end

  subgraph Managed["Managed services"]
    TW[Twilio<br/>WhatsApp · SMS · Voice]
    CP[Campay<br/>MTN MoMo · Orange]
    R2[Cloudflare R2<br/>media]
    SB[Supabase<br/>waitlist only]
    CF[Cloudflare<br/>DNS · CDN · WAF]
  end

  PWA --> CF
  CF --> Front
  CF --> Nx
  Front -.staging only.-> StA
  Front --> Nx
  Nx --> PrA
  PrA --> PrPG
  PrA --> PrR
  PrA --> TW
  PrA --> CP
  PWA --> R2
  Landing --> SB
```

## Module map (chopnow-api)

Modules align with Epics in the PRD. Each domain module owns its database tables; cross-module reactions go through domain events, never direct service calls.

```mermaid
flowchart LR
  Auth[AuthModule<br/>Epic 1] --> Users[UsersModule<br/>Epic 1]
  Auth --> Twilio[TwilioModule<br/>infra]
  Catalogue[CatalogueModule<br/>Epic 2] --> Users
  Orders[OrdersModule<br/>Epic 3] --> Catalogue
  Orders --> Payments[PaymentsModule<br/>Epic 3+]
  Payments --> Campay[CampayModule<br/>infra]
  Dispatch[DispatchModule<br/>Epic 4] --> Orders
  Dispatch --> Twilio
  Finance[FinanceModule<br/>Epic 7] --> Orders
  Admin[AdminModule<br/>Epic 6] -.observes.-> Auth
  Admin -.observes.-> Catalogue
  Admin -.observes.-> Orders
  Admin -.observes.-> Dispatch
  Admin -.observes.-> Finance

  classDef pending fill:#f5f5f5,stroke:#aaa,color:#888
  class Catalogue,Orders,Payments,Campay,Dispatch,Finance,Admin pending
```

Today (post Story 1.1):

- ✅ AuthModule, UsersModule, TwilioModule, HealthModule
- ⏳ Everything else lands per Sprint 1–3 in `sprint-status.yaml`

## Request → response anatomy (signup example)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as PWA
  participant N as Nest API
  participant T as Twilio
  participant D as Postgres
  participant R as Redis

  U->>W: enter phone (e.g. 670000000)
  W->>N: POST /api/auth/request-otp
  N->>N: normalisePhone → +237670000000
  N->>D: INSERT OtpLog (status=PENDING)
  N->>T: messages.create (WhatsApp + statusCallback)
  T-->>N: { sid }
  N->>D: UPDATE OtpLog (status=SENT, providerMessageId=sid)
  N-->>W: 200 { ok, expiresInSeconds: 300 }

  Note over T,N: Async (later)
  T->>N: POST /api/twilio/status (delivered/failed)
  N->>D: UPDATE OtpLog by providerMessageId

  U->>W: enter the 6-digit code
  W->>N: POST /api/auth/verify-otp
  N->>D: find OtpLog (PENDING/SENT/DELIVERED, not expired)
  N->>N: argon2 verify
  N->>D: UPSERT User
  N->>D: UPDATE OtpLog (status=VERIFIED)
  N-->>W: 200 { accessToken, refreshToken }
```

## Why this shape

| Concern | Decision | See |
|---|---|---|
| Single VPS until 2 000 cmd/jour | One Hetzner box, Postgres + Redis colocated. Predictable cost, simple ops. | [ADR-0003](../decisions/0003-hetzner-not-aws.md) |
| PWA, not native | Wakelock API validated on Android Tecno/Itel. No app-store friction. | [ADR-0001](../decisions/0001-pwa-not-react-native.md) |
| Twilio for comms | Unified WhatsApp + SMS + Voice SDK. Skips 15-day Meta verification. | [ADR-0002](../decisions/0002-twilio-not-africas-talking.md) |
| ARM in prod, x86 in staging | Hetzner CAX11 (Ampere ARM) is 20% cheaper. DO basic is x86. | [ADR-0004](../decisions/0004-arm64-prod-amd64-staging.md) |
| Cloudflare R2 for media | Zero egress cost. S3-compatible API. | architecture.md |
| Vercel for frontend | Free tier, atomic deploys, instant rollback. | architecture.md |
