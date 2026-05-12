# Rollback (staging or prod)

## When to use

A deploy went bad: health check fails, error rate spikes, a regression is reported. You want to get back to the previous good image **fast** (target RTO: ~3 min).

## Step 1 — identify the last good SHA

Two ways:

```bash
# Via Git: find the PR before the bad one
git log --oneline origin/staging | head -5

# Via Docker on the droplet: see what images are still loaded
ssh deploy@157.230.125.224 'docker images chopnow-api --format "{{.Tag}}\t{{.CreatedAt}}"' | head -5
```

The droplet keeps the last ~3 images on disk (we don't `docker image prune` after every deploy). Pick the most recent one with a known-good tag.

## Step 2 — re-deploy that SHA via workflow_dispatch

If the previous image is still on the droplet, just retag and recreate:

```bash
ssh deploy@157.230.125.224 'bash -s' <<EOF
set -euo pipefail
cd /opt/chopnow/staging
# Pick the prior tag
PRIOR=staging-<prior-sha-from-git>
# Update IMAGE_TAG in .env
sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=${PRIOR#staging-}/" .env
docker compose up -d api --force-recreate --remove-orphans
docker compose ps
EOF
```

If the image is no longer on the droplet (pruned), re-trigger the workflow on the prior commit via `gh workflow run ci.yml --ref <prior-sha>` — CI will rebuild from source.

## Step 3 — verify

```bash
ssh deploy@157.230.125.224 'curl -s http://localhost:3001/health'
# uptime should reset to < 60s
```

Then check application-level: hit a couple of routes that were broken in the bad deploy, confirm they work.

## Step 4 — post-incident

1. **Don't revert blindly.** The bad commit is on `staging` (and possibly `develop`). Leave it there unless it's actively serving traffic — debugging is easier with the bad code visible.
2. Open a follow-up issue / PR with the fix.
3. Write a 3-bullet postmortem in the PR: what broke, why CI didn't catch it, what gate we add next.

## What to do if rollback itself fails

| Symptom | Cause | Fix |
|---|---|---|
| Compose can't find prior image | Pruned | Re-build via `gh workflow run ci.yml --ref <sha>` |
| Migrations applied during bad deploy aren't backwards-compatible | Real DB regression | This is the hard case — restore from Hetzner snapshot. RTO ~30 min. Document the regression class so the next migration is reversible. |
| Health check passes but app behaves wrong | Stale Redis / cached state | `docker compose restart redis` clears Redis; full `down && up -d` clears everything |
