# Repositories

All ChopNow code + infra + docs live under the [`ChopNow-app`](https://github.com/ChopNow-app) GitHub org.

| Repo | What | Branches | Hosting | Pages |
|---|---|---|---|---|
| [chopnow-api](https://github.com/ChopNow-app/chopnow-api) | Backend (NestJS 11, TypeScript, Prisma, Postgres + PostGIS, Redis) | gitflow: `feature/*` → `develop` → `staging` → `main` | Hetzner CAX11 (prod, planned) · DO Basic (staging) | — |
| [chopnow-app](https://github.com/ChopNow-app/chopnow-app) | Frontend PWA (Next.js 16.2.4, Tailwind, Mapbox) — covers consumer, livreur, vendeur, admin | tbd | Vercel | — |
| [chopnow-infra](https://github.com/ChopNow-app/chopnow-infra) | Terraform for DO staging, Hetzner prod (planned), AWS (placeholder) | trunk-based: `main` | self-hosted state | — |
| [chopnow-docs](https://github.com/ChopNow-app/chopnow-docs) | This site — MkDocs Material | trunk-based: `main` | GitHub Pages | https://chopnow-app.github.io/chopnow-docs/ |
| [chopnow-landingpage](https://github.com/ChopNow-app/chopnow-landingpage) | Waitlist landing (Next.js 14 + Supabase) | trunk-based: `main` | Vercel | https://chopnow.app (planned) |

## chopnow-api — branch protection (rulesets)

| Rule | develop | staging | main |
|---|---|---|---|
| Block deletion | ✓ | ✓ | ✓ |
| Block non-fast-forward push (force-push) | ✓ | ✓ | ✓ |
| Require PR | ✓ | ✓ | ✓ |
| Required approving reviews | 1 | 1 | 1 |
| Required status check | `lint-test-build` | `lint-test-build` | `lint-test-build` |
| Allowed merge methods | squash, rebase | squash, rebase | squash, rebase |

Repo-level: `allow_merge_commit: false`, `allow_rebase_merge: true`, `allow_squash_merge: true`.

**Solo dev** = can't self-approve own PR → `gh pr merge --admin` flag is the operating mode. Documented in [onboarding-dev.md](../getting-started/onboarding-dev.md).

## CI tiers (chopnow-api)

| Trigger | Tier 1 (lint/types/unit/audit/secrets/SAST) | Tier 2 (integration + e2e + build) | Tier 2.5 (docker build amd64) | Tier 3 (docker build arm64 + Trivy) |
|---|---|---|---|---|
| Push to feature branch | ✓ | — | — | — |
| PR → develop | ✓ | ✓ | — | — |
| PR → staging | ✓ | ✓ | ✓ | — |
| PR → main | ✓ | ✓ | ✓ | ✓ |
| Push to staging (post-merge) | ✓ | — | — | — |
| Push to main (post-merge) | ✓ | — | — | — |

Push to develop is deliberately excluded — it only ever gets PR-merged commits already validated.

## Cross-repo issue conventions

Each user story has **two GitHub issues** — one on `chopnow-api` (backend) and one on `chopnow-app` (frontend), both labelled `epic:N-<name>`. Backend PR's body uses `Closes #N` on the api repo; frontend PR's body uses `Closes #N` on the app repo. When only one half is delivered (e.g. Story 1.1 backend done, frontend pending), close the matching half with a comment linking the PR; leave the other open.

## Repo CODEOWNERS

`chopnow-api/.github/CODEOWNERS` reads `* @AndreLiar`. Will grow as the team grows.
