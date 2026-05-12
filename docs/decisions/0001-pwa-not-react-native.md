# ADR-0001 — PWA, not React Native

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-04-08 |
| **Authors** | Founder + architecture review v3 |

## Context

Early architecture (v1, v2) assumed React Native for the rider app to handle GPS background tracking, the load-bearing technical requirement. React Native means:

- Separate iOS + Android builds, app-store reviews (7+ days each for the first submission, 24–48h after)
- Install funnel drop-off (~50% loss from "interested" to "installed app")
- A second codebase to maintain alongside the consumer/vendor PWA
- Tecno/Itel/Infinix devices in Douala have inconsistent Play Store experiences

POC-3 (2026-04-13) tested the **Wakelock API** on a real Tecno + a real Itel: GPS held active for >10 minutes on Chrome Android with the screen off. That eliminated the only hard reason to go native.

## Decision

**Consumer, livreur, vendeur, and admin all share the same Next.js 14 PWA.** No native apps.

GPS background tracking for the livreur uses the Wakelock API + a 15-second position heartbeat to the API. The PWA is installable from Chrome's "Add to Home Screen" prompt (Story 1.10).

## Alternatives considered

| Option | Dealbreaker |
|---|---|
| **React Native for rider only** | Two codebases, two stores, two build pipelines. The PWA already needed routing per actor; doubling that is solo-dev hostile. Native auth (deep-link OTP, biometrics) isn't worth the maintenance cost at this scale. |
| **Capacitor / Tauri wrapper around the PWA** | Ships the same JS bundle inside a native shell. Adds app-store friction (still a review) without buying us much beyond Push (which Web Push covers) and Biometrics (post-MVP). |
| **Pure web with no service worker** | Loses offline cart, loses Web Push, loses install prompt. The PWA features are the cheapest way to feel "app-like" without being one. |

## Consequences

**Positive**

- Single codebase across four surfaces
- Iteration speed: deploy = `vercel --prod`; no review cycle
- No app-store gatekeeping; works on every Android browser
- Web Push API + VAPID covers notifications without Firebase

**Negative**

- iOS install UX is worse than Android (Safari "Add to Home Screen" is a manual gesture, not a prompt). Acceptable for an Android-dominant market.
- Background sync on iOS is limited; we mitigate by keeping the livreur app visually awake during deliveries via Wakelock.
- Some native APIs (Bluetooth payment readers, NFC) are gated. None are MVP requirements.

**Reversal cost** — medium. If we later need native, we'd port the rider screens to React Native (~2 weeks). Consumer/vendor/admin stay PWA.

## Follow-ups

- `epic-5-whatsapp-bot` was retired in the same revision (PWA covers the same UX without bot constraints)
- Story 1.10 — PWA install prompt (in Sprint 1)
- Story 1.11 — Web Push VAPID subscription (in Sprint 1)
- Story 1.12 — GPS permissions for livreur (in Sprint 1)
