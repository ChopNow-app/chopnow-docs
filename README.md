# chopnow-docs

Technical documentation for the ChopNow platform — architecture, decisions, runbooks, onboarding.

**Published at:** https://chopnow-app.github.io/chopnow-docs/

## What goes here

- **Onboarding** — how a new developer or operator gets productive
- **Architecture** — stack, infrastructure, data model, security
- **Decisions (ADRs)** — one Markdown file per major call, with context + rationale + consequences
- **Runbooks** — step-by-step procedures for ops tasks (deploy, rollback, token rotation, etc.)
- **Reference** — repo map, environment URLs, glossary

**Out of scope:** API reference (lives in chopnow-api as OpenAPI/Swagger), product PRDs (live in `_bmad-output/planning-artifacts/`).

## Contributing

Every page is plain Markdown under `docs/`. Edit, commit, push — GitHub Actions builds and publishes automatically.

### Local preview

```bash
pip install "mkdocs-material>=9.5"
mkdocs serve
# open http://localhost:8000
```

### Adding an ADR

1. Copy `docs/decisions/template.md` to `docs/decisions/<NNNN>-<short-title>.md` (increment NNNN)
2. Fill in Context, Decision, Consequences
3. Add a nav entry in `mkdocs.yml`
4. PR with the ADR + any docs it supersedes marked `Status: Superseded by ADR-NNNN`

### Adding a runbook

Same shape, under `docs/runbooks/`. Each runbook should be runnable by someone who hasn't seen the system before — assume nothing.

## Why MkDocs Material

Chose Markdown-only authoring over MDX/React frameworks (Docusaurus, Nextra) so non-engineers — ops, founders, partners — can contribute. The Material theme gives us search, dark mode, mermaid diagrams, code highlighting, and admonitions out of the box; we don't tune the theme.

If we ever grow into versioned product docs (per-release branches with their own published docs), `mike` plugs into MkDocs cleanly without a framework swap.
