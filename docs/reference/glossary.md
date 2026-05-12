# Glossary

Cameroon-specific terms + project shorthand. The product is bilingual-friendly but day-to-day vocabulary defaults to French / Camfranglais.

## Actors

| Term | Meaning |
|---|---|
| **Consumer** / **Consommateur** | The customer ordering food |
| **Vendeur** | Person/business selling food. Three sub-types: |
| **Cuisinier(e) informel** | Home cook, no formal establishment; sells from kitchen or street stand. Often "mamans du quartier" |
| **Maquis** | Semi-formal eatery; fixed location, often outdoor, popular for grilled/local food |
| **Restaurant** | Formal establishment with full kitchen, often air-conditioned indoor seating |
| **Livreur** | Delivery rider. Usually moto, sometimes bicycle or on foot for short trips |

## Food + commerce

| Term | Meaning |
|---|---|
| **Mboué** | Cassava-based dish (proper noun, used in brand identity — Mboué Green is a brand color) |
| **Chop** | Camfranglais for "to eat"; "chopper" = to eat. Brand name comes from this |
| **MoMo** | Mobile Money. Either **MTN MoMo** or **Orange Money** — the two telco money networks in Cameroon |
| **Appoint** | Exact change. "Préparez l'appoint" = have exact change ready (cash livreur deliveries) |
| **FCFA** | Franc CFA (Communauté Financière Africaine). Cameroon's currency. 1 EUR ≈ 656 FCFA. **No decimals in display**, no thousands separators (e.g. `5000 FCFA`, not `5,000 FCFA`) |

## Geography

| Term | Meaning |
|---|---|
| **Quartier** | Neighborhood. Douala is divided into ~50 quartiers |
| **Makepe** | Pilot quartier — north Douala, mixed residential/commercial |
| **Bonamoussadi** | Pilot quartier — adjacent to Makepe, more residential |
| **Logpom** | Pilot quartier — south of Makepe |
| **Bonabéri** | West Douala, across the Wouri river — Phase 2 launch zone |
| **Akwa** | Central Douala — Phase 2 launch zone |
| **Bonapriso** | Central-south, affluent residential — Phase 2 |
| **Landmark** | Named point of reference (rond-point, pharmacy, school) used in the SmartAddress system. Couriers find addresses by landmark, not street name |

## Tech / project shorthand

| Term | Meaning |
|---|---|
| **POC-N** | Proof-of-concept, numbered (POC-1 Campay, POC-2 Twilio OTP, POC-3 Wakelock GPS, POC-4 Voice proxy). All validated 2026-04-13 |
| **TD-N** | Tech Debt item, numbered. Tracked in `_bmad-output/implementation-artifacts/sprint-status.yaml` |
| **TERRAIN-N** | Pre-sprint physical-world task (e.g. landmark collection, vendor recruitment). Tracked in same file |
| **ADR-NNNN** | Architecture Decision Record. Lives in [`decisions/`](../decisions/index.md) |
| **Story 1.1, 2.5, etc.** | Backlog story, in Epic.Story notation. Authoritative list in `sprint-status.yaml` and per-epic Markdown files in `_bmad-output/planning-artifacts/epics/` |
| **Soft launch** | Private launch with the first ~17 waitlist signups; July 2026. No public marketing |
| **Lancement** | Public launch after soft launch validates 50+ orders without major incidents |

## Things we intentionally don't translate

- **"Chop Now"** — brand. No French translation.
- **"Mboué Green"** — brand color (success/confirmed states only).
- **"Chop Orange"** — brand color (legacy; being phased out by new identity work).
