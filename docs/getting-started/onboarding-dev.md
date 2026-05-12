# Developer onboarding

Goal: from zero to a running backend + tests passing in **under 30 minutes**.

## Prereqs

| | Version | Why |
|---|---|---|
| Node | 22.x | NestJS 11 baseline |
| Docker + Compose | recent | Postgres + Redis locally; testcontainers for integration tests |
| Git + GitHub CLI (`gh`) | any recent | `gh pr create`, `gh secret set` etc. used a lot |
| Access to `ChopNow-app` org | — | Ask the founder |

## Clone the repos

```bash
mkdir -p ~/Desktop/Business/ChopNow && cd $_
git clone https://github.com/ChopNow-app/chopnow-api.git
git clone https://github.com/ChopNow-app/chopnow-app.git           # frontend PWA
git clone https://github.com/ChopNow-app/chopnow-infra.git         # Terraform
git clone https://github.com/ChopNow-app/chopnow-docs.git          # this site
git clone https://github.com/ChopNow-app/chopnow-landingpage.git   # waitlist landing
```

The `_bmad-output/` planning directory is NOT in Git (TD-5 in sprint-status.yaml). Ask the founder for a copy — it has the PRD, architecture v3, business model, and the sprint backlog.

## Backend: chopnow-api

```bash
cd chopnow-api
npm ci
cp .env.example .env
# Generate JWT secrets:
./scripts/generate-secrets.sh
# Start Postgres + Redis (docker-compose.yml in the repo)
docker compose up -d
# Apply migrations
npx prisma migrate dev
npx prisma generate
# Run
npm run start:dev   # localhost:3001/api/docs for Swagger
```

Health check: `curl http://localhost:3001/health` → `{"status":"ok",...}`.

## Tests (you should run these before pushing)

```bash
npm test                # unit, ~5s
npm run lint            # ESLint, must be clean
npx tsc --noEmit        # type check, must be clean
npm run test:integration # testcontainers (Docker needs to be running), ~10s
npm run test:e2e        # full AppModule + testcontainers, ~12s
```

The pre-push Husky hook runs `typecheck && npm test` automatically. The pre-commit hook runs lint-staged (ESLint + Prettier on staged files).

## Branching workflow

```
feature/* → PR → develop      (CI Tier 1+2, squash merge)
develop  → PR → staging       (squash merge → auto-deploy to DigitalOcean)
staging  → PR → main          (squash merge → auto-deploy to Hetzner — pending)
```

- **Never push directly to `develop`, `staging`, or `main`.** All three are protected by rulesets (review + CI required).
- **Never use `--delete-branch` when the PR head is a trunk** (develop, staging, main). The head branch gets deleted from remote.
- **Use `--squash` for trunk-to-trunk merges** (develop→staging, staging→main). The repo blocks `merge` commits, and `--rebase` causes SHA divergence over time.
- **Conventional Commits:** `feat(scope):`, `fix(scope):`, `chore(scope):`, `test(scope):`, etc. No `Co-Authored-By` trailer per founder preference.

See [ADR-0004](../decisions/0004-arm64-prod-amd64-staging.md) and the [deploy runbook](../runbooks/deploy-staging.md) for what happens after merge.

## Sprint rhythm

1. Pick the next story from `_bmad-output/implementation-artifacts/sprint-status.yaml` (`sprints[0].progress`).
2. Create a feature branch off `develop`.
3. First commit of the branch updates `sprint-status.yaml` `status: in_progress` for that story.
4. Code, test, self-review (`git diff --cached`), commit.
5. Pre-push gate: `npm test && npm run lint && npx tsc --noEmit && npm run test:integration && npm run test:e2e`.
6. Open PR to develop, link the GitHub issue (`Closes ChopNow-app/chopnow-api#N`).
7. CI green → squash merge → close the matching issue (auto-closed by `Closes`).

## Standards (non-negotiable)

- Every HTTP endpoint MUST have an integration test (Nest testing module + supertest, optional testcontainers DB).
- Every controller route MUST have an E2E case in `test/auth.e2e-spec.ts`-style if it changes user-facing behavior.
- No secrets in code or `.env.example`.
- DTOs use `class-validator`; user-supplied URLs go through `shared/http/safeFetch` (SSRF-safe).
- No mocked tests for DB transactions / unique constraints — use testcontainers (`test/users.integration.spec.ts` as template).

## Where to go next

- [Architecture overview](../architecture/overview.md) — the high-level diagram and module map
- [Deploy runbook](../runbooks/deploy-staging.md) — what CI does on push to staging
- [Repositories reference](../reference/repositories.md) — full repo map with branches + rulesets
