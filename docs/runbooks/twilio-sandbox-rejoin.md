# Re-join the Twilio sandbox

## When to use

You sent yourself an OTP for testing and **didn't receive it on WhatsApp**. Twilio API returns success, but the message never arrives. Symptoms:

- Twilio Console → Logs → recent message shows `error_code: 63015`, `status: failed`
- Or: WhatsApp from `+14155238886` responds with *"⚠️ Your number is not connected to a Sandbox. You need to connect it first by sending join…"*

This is because the **Twilio Sandbox membership expires every 72 hours**. Until our production WhatsApp Business sender is approved (`todo.md` B2), all test OTPs go through the sandbox.

## Steps

1. On your phone, open WhatsApp.
2. Send a message to **`+1 415 523 8886`** (Twilio's sandbox number).
3. The message body must be exactly: **`join flag-excitement`** (the keyword for our sandbox).
4. Wait for the confirmation reply: *"✅ Twilio Sandbox: You are all set! The sandbox can now send/receive messages…"*

## Verification

```bash
# From the chopnow-api repo, against staging (or local)
curl -X POST http://localhost:3001/api/auth/request-otp \
  -H 'Content-Type: application/json' \
  -d '{"phone":"+237XXXXXXXXX"}'
# → {"ok":true,"expiresInSeconds":300}
```

Check WhatsApp. The OTP should arrive within 30s.

If running against staging:
```bash
curl -X POST http://157.230.125.224:3001/api/auth/request-otp \
  -H 'Content-Type: application/json' \
  -d '{"phone":"+237XXXXXXXXX"}'
```

Then check the OtpLog table on staging:
```bash
ssh deploy@157.230.125.224 'docker compose -f /opt/chopnow/staging/docker-compose.yml exec -T db psql -U chopnow -c "SELECT phone, status, \"failedReason\" FROM otp_logs ORDER BY \"createdAt\" DESC LIMIT 1;"'
```

`status = SENT` (or `DELIVERED` if the webhook has fired) means you're good.

## What to do if it fails

| Symptom | Cause | Fix |
|---|---|---|
| WhatsApp says "Sandbox: You are all set!" but OTP still doesn't arrive | OTP API call failed before reaching Twilio | Check `chopnow-api` logs for the OtpLog row's `failedReason` |
| WhatsApp didn't respond to `join flag-excitement` | Wrong keyword or wrong sandbox number | The keyword is **per Twilio account**. Ours is `flag-excitement` (in `pocs/poc-2-whatsapp-otp/.env` for reference). If we ever change accounts, update this runbook. |
| Multiple devices need to receive OTPs | Each device must individually `join flag-excitement` | One number = one device's Twilio Console-managed opt-in. Keep a small list of opted-in test numbers. |

## Why this exists (= why we don't just use prod WhatsApp Business in dev)

`todo.md` B2 — we haven't submitted the Meta business verification yet. The sandbox is the only way to send WhatsApp messages from Twilio without verification. Production rollout has the proper `whatsapp:+1XXX` sender that doesn't require opt-in.
