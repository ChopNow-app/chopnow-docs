# Runbooks

Step-by-step procedures for ops tasks. Every runbook is written to be executable by someone who hasn't seen the system before — assume nothing.

| Runbook | When to use |
|---|---|
| [Deploy to staging](deploy-staging.md) | Push to `staging` branch on chopnow-api, watch the deploy. Also covers manual override if CI is unavailable. |
| [Rollback](rollback.md) | A deploy went bad. Get back to the previous good image. |
| [Re-join Twilio sandbox](twilio-sandbox-rejoin.md) | OTPs stopped delivering to your test number. Sandbox membership expires every 72h. |
| [Rotate DO API token](rotate-do-token.md) | Token suspected leaked, or quarterly hygiene rotation. |

## Adding a new runbook

1. Copy a small existing runbook as template.
2. Structure: **When to use** · **Steps** · **Verification** · **What to do if it fails**.
3. Every command literal. No "you'll figure it out". If a step depends on a secret, name the secret and where it lives (e.g. `staging` GitHub env, `STAGING_HOST`).
4. Add to the table above and to `mkdocs.yml`.
