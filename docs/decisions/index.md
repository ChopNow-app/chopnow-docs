# Architecture Decision Records (ADRs)

One Markdown file per major call. Each ADR captures **context** (what we knew at the time), **decision** (what we picked), **alternatives considered** (what we ruled out), and **consequences** (what this costs us in flexibility / time / money).

ADRs are append-only. If a decision is later overturned, the new ADR cites and supersedes the old one (`Status: Superseded by ADR-NNNN`); the old one stays in place for historical context.

## Active ADRs

| ID | Title | Status | Date |
|---|---|---|---|
| [0001](0001-pwa-not-react-native.md) | PWA, not React Native | Accepted | 2026-04-08 |
| [0002](0002-twilio-not-africas-talking.md) | Twilio for comms, not Africa's Talking SMS direct | Accepted | 2026-04-13 |
| [0003](0003-hetzner-not-aws.md) | Hetzner for prod hosting, not AWS | Accepted | 2026-05-10 |
| [0004](0004-arm64-prod-amd64-staging.md) | arm64 production, amd64 staging | Accepted | 2026-05-10 |

## Writing a new ADR

1. Copy [`template.md`](template.md) to `NNNN-short-title.md` (next available NNNN).
2. Fill in every section. If a section is genuinely N/A, write "N/A" plus a one-liner explaining why.
3. Add an entry above and a `nav:` line in `mkdocs.yml`.
4. PR. Review focus: is the alternatives section honest?
