# ADR-0004 — arm64 production, amd64 staging

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-05-10 |
| **Authors** | Founder + first staging deploy session |

## Context

Production hosting on Hetzner CAX11 (per [ADR-0003](0003-hetzner-not-aws.md)). The CAX line uses **Ampere ARM** (arm64) CPUs — ~20% cheaper than the equivalent CX line (Intel/AMD x86_64).

Staging on DigitalOcean Basic Regular droplets, which are **x86_64** (shared Intel/AMD CPU); the DO ARM options are CPU-Optimized tier, ~3× more expensive, defeats the purpose of using DO's $200 credit.

So we have two deploy targets on two different CPU architectures. Decision needed: how does the build pipeline handle this?

## Decision

**Build per-architecture images per deploy target:**

- **Staging deploys (DO)** build `linux/amd64` natively on `ubuntu-latest` GitHub runners (~30s)
- **Production deploys (Hetzner)** build `linux/arm64` via QEMU emulation under `docker buildx` (~80–100s)
- **Tier 3 CI** (PR → main) runs the arm64 build + Trivy scan, so the production-target arch is exercised before merge

No multi-arch manifest list yet — single-arch image per target, scp + `docker load`. Migrate to a registry (ghcr.io) with multi-arch manifests if/when we have a second prod target.

## Alternatives considered

| Option | Dealbreaker |
|---|---|
| **Use Hetzner CX23 (Intel) for prod too** | Costs ~€4.60/mois vs CAX11's ~€3.80/mois. Saves ~€10/year. Not worth giving up the ARM cost advantage. |
| **Use DO ARM droplets for staging** | DO ARM is CPU-Optimized tier, $42/mois minimum vs $12 for Basic AMD. Burns the credit faster. Defeats the "free staging" goal. |
| **Build multi-arch manifest for every deploy** | Doubles CI build time (~150s instead of 80s) for marginal benefit until we have multiple prod targets. Easy upgrade later. |
| **Compile to amd64 only and run via QEMU on arm64** | ~30% perf penalty at runtime. Hetzner CAX11 is small enough that we'd feel it. Always use native binaries on the target arch. |

## Consequences

**Positive**

- **Cost.** Hetzner CAX11 cheapest in its class. Save ~3 000 FCFA/an vs Intel equivalent.
- **Architecture diversity** catches portability bugs in CI (Tier 2.5 amd64 + Tier 3 arm64 = both surfaces exercised before main lands).
- **Future-proof.** ARM is increasingly the cheap path (AWS Graviton, GCP Tau T2A, Hetzner CAX). We're already there.

**Negative**

- **CI build time** for Tier 3 is QEMU-emulated, ~2× native amd64. Acceptable at PR-to-main cadence (low frequency).
- **Two image variants** to manage; the deploy job needs to pick the right one per target. Mitigated by tagging convention (`staging-<sha>` is always amd64, future prod tags will be arm64).
- **Local dev on Mac (Apple Silicon)** is arm64-native — `docker build` produces arm64 by default. Devs may accidentally build arm64 and try to push to staging. CI catches this (it builds amd64 from scratch).
- **Some native deps** sometimes lag on arm64 (older bcrypt, prebuilt sharp variants). We use `argon2` and sharp 0.33+; both have arm64 prebuilds. Re-test before adding any new native dep.

**Reversal cost** — small. Drop the `--platform` flag, drop the QEMU step. ~5 lines of YAML.

## Follow-ups

- When `chopnow-infra/hetzner/production/` lands (late June 2026), the prod deploy job mirrors `deploy-staging` but with `linux/arm64` build
- Move to registry (ghcr.io) with multi-arch manifest when we add a second prod region or want to share the image with the frontend (e.g. SSR)
- Verify `prisma` engines are arm64-compatible on each major Prisma version bump (Prisma 6.x is fine; revisit at 7.x)
