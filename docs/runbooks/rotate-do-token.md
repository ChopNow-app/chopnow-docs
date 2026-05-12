# Rotate the DigitalOcean API token

## When to use

- **Immediately** if the token was exposed (paste in chat, accidental git commit, contributor offboarding)
- **Quarterly** as hygiene

The token gives full access to the DO account — create / destroy droplets, modify DNS, view billing. Treat it like a root password.

## Steps

### 1. Generate a new token

1. Go to https://cloud.digitalocean.com/account/api/tokens
2. Click **Generate New Token**
3. Name: `chopnow-terraform-YYYY-MM` (date-stamp helps next rotation know what's old)
4. Scope: **Custom scopes** → check **Read** + **Write** on `droplet`, `ssh_key`, `firewall`, `project`. (Avoid the lazy "Full Access" — least privilege.)
5. Expiration: 90 days
6. **Copy the token immediately** (DO won't show it again)

### 2. Replace the old token where it's used

Terraform local apply:
```bash
# On the operator's machine
export TF_VAR_do_token='dop_v1_<new-token>'
# Verify it works
cd ~/Desktop/Business/ChopNow/chopnow-infra/digitalocean/staging
terraform plan -var 'ssh_key_name=<your-key-name>'
# Should report "No changes" if state matches reality
```

### 3. Revoke the old token

1. Back to https://cloud.digitalocean.com/account/api/tokens
2. Find the old token (named `chopnow-terraform-<old-date>` or unnamed if it was the original)
3. Click the three-dot menu → **Delete**

### 4. Verify nothing broke

```bash
# Sanity-check the staging droplet is still reachable
ssh -i ~/.ssh/do_ed25519 deploy@157.230.125.224 'echo ok'

# And Terraform still sees it
cd ~/Desktop/Business/ChopNow/chopnow-infra/digitalocean/staging
TF_VAR_do_token='dop_v1_<new-token>' terraform plan -var 'ssh_key_name=<your-key-name>'
# Should still be "No changes"
```

## What to do if it fails

| Symptom | Cause | Fix |
|---|---|---|
| Terraform errors out with 401 Unauthorized | Token mistyped or scoped too narrowly | Re-generate; verify scopes include droplet/ssh_key/firewall read+write |
| `terraform plan` reports unexpected changes | The new token may not see resources created by the old one (token-scoped projects) | Confirm both tokens belong to the same DO team; check token's project access |
| SSH still works but Terraform doesn't | Token problem only | The droplet itself doesn't care about the API token; only Terraform does. Fix the token, leave the droplet alone. |

## Long-term mitigation

- **No more pasting tokens in chat** (incident 2026-05-10 — exposed in conversation log + Claude transcripts; token was deferred-rotated by founder's choice).
- **GitHub secrets for CI** — once we have a deploy-time DO API call (e.g. provisioning bumps via CI), the token goes in the repo's `staging` environment secret, not anywhere else.
- **Annual hygiene review** — even if no exposure, rotate every 90 days. Calendar reminder.
