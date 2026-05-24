# Mobile-first design principles

The rule, in one line:

> **Write the mobile layout as your base CSS. Add `md:` / `lg:` / `xl:`
> modifiers to ENHANCE for larger screens. Never the reverse.**

## Why this is the rule (not just a preference)

ChopNow's market is Cameroon. The pilot zone (Bonamoussadi) is
**~90 % mobile**. Real users are on Tecno / Itel / Infinix phones with:

- 320–414 px viewport widths
- 3G / patchy 4G networks (a slow CSS evaluation hurts here)
- Thumbs as the primary input (not pixel-precise mouse pointers)
- WhatsApp + browser open simultaneously (limited RAM)

A desktop-first codebase forces every mobile user (i.e. the majority)
into a "fallback" path that is always playing catch-up. Mobile-first
inverts that: the desktop user (the minority, mostly admin staff +
the founder) gets the enhancements, but the base is fast and works.

## How this looks in code

Tailwind's mobile-first breakpoints are the mechanism:

| Modifier | Min width | Use for |
|---|---|---|
| (none) | 0 px | Mobile base — **every component starts here** |
| `sm:` | 640 px | Big phone / small tablet portrait |
| `md:` | 768 px | Tablet portrait, small laptop |
| `lg:` | 1024 px | Standard laptop |
| `xl:` | 1280 px | Large laptop / external monitor |
| `2xl:` | 1536 px | Deliberately unused (see "Tier 4 not a target" below) |

### Good — mobile-first

```tsx
// Base: 1-column grid, mobile padding. md/lg/xl ADD wider columns + more
// generous padding. A mobile user gets the simplest path; a desktop user
// gets the enhancement.
<ul className="grid grid-cols-1 gap-4 px-5 sm:grid-cols-2 md:grid-cols-3 md:px-8 lg:grid-cols-3 lg:gap-5 lg:px-12 xl:grid-cols-4 xl:gap-6">
```

### Bad — desktop-first (forbidden)

```tsx
// Base: assumes 1280px container with 4 cols. max-md:hidden patches mobile
// after the fact. Every mobile-only behavior is an exception.
<ul className="grid grid-cols-4 gap-6 px-12 max-lg:grid-cols-3 max-md:grid-cols-2 max-sm:grid-cols-1 max-sm:px-5">
```

The rule: **never use `max-sm:` / `max-md:` / `max-lg:` / `max-xl:`
modifiers in new code**. If you find yourself reaching for one, the
component was probably authored desktop-first; rewrite it.

## Touch targets — 44 × 44 px floor

Apple HIG + WCAG 2.5.5 both require interactive elements to be at
least 44 × 44 px on touch surfaces. ChopNow's `Button` primitive's
`size="default"` is `h-11` (44 px) and `size="icon"` is `h-11 w-11`
(44 × 44 px) — both meet the floor.

When wrapping a non-Button interactive element (a `<Link>`, a custom
chip, a tappable card), check the rendered height includes padding to
44 px minimum.

Reference: [chopnow-app PR #181](https://github.com/ChopNow-app/chopnow-app/pull/181)
shipped the 40 → 44 px icon-button bump.

## iOS form-input no-zoom

`app/globals.css` sets `input, textarea, select { font-size: 16px }`.
Anything smaller triggers iOS Safari's auto-zoom-on-focus, which is
jarring. Don't override this to a smaller font on inputs — find
another way to fit the layout.

## The one deliberate exception — admin dense-data tables

`features/admin/components/AdminFinanceDashboard.tsx` has tables with
`min-w-[640px]`. Mobile users see horizontal scroll within the wrapper.
**This is intentional**: dense financial tables can't sensibly
collapse columns into cards without losing the at-a-glance density
admin staff need. Stripe Dashboard, Linear, GitHub Actions UI all use
the same horizontal-scroll fallback on narrow viewports.

Admin surfaces are pragmatic about this. Consumer / livreur / vendeur
surfaces never do this.

Reference: shipped in [chopnow-app PR #181](https://github.com/ChopNow-app/chopnow-app/pull/181)
with the `-mx-5 overflow-x-auto px-5 md:mx-0 md:px-0` wrapper.

## Tier 4 (≥1536 px) is not a target

You'll notice the codebase has **zero `2xl:` modifiers**. That's
deliberate. ChopNow's market doesn't include 4K-monitor consumers, and
the admin surface caps comfortably at `xl:max-w-7xl` (1280 px) with
centered whitespace on wider screens — same pattern Stripe, GitHub,
Linear use.

If a partner asks for a "spread-to-edges 4K layout" in the future,
that's a real product decision to revisit. Until then, don't add
`2xl:` modifiers as polish.

## Sanity check before opening a PR

In your terminal, in the chopnow-app repo:

```bash
# Should return 0 lines — any output means desktop-first authoring crept in
git diff develop... -- '*.tsx' '*.css' | grep -E '(\+|^)\+.*max-(sm|md|lg|xl):'
```

If it returns lines: you authored desktop-first. Rewrite the affected
classes so mobile is the base and larger screens add modifiers.

## When in doubt

Open the page at 320 × 568 px (iPhone SE) in Chrome DevTools BEFORE
1440 × 900 px (laptop). The mobile experience is the source of truth.
If it doesn't work at 320, it doesn't work — no amount of
`lg:`-modifier polish saves it.

## Related

- [chopnow-app PR #181](https://github.com/ChopNow-app/chopnow-app/pull/181) — 44 px icon buttons + admin table wrap + ListSkeleton
- [chopnow-app PR #185](https://github.com/ChopNow-app/chopnow-app/pull/185) — catalogue 3-col at md + global `prefers-reduced-motion`
- [chopnow-app PR #174](https://github.com/ChopNow-app/chopnow-app/pull/174) — vendor detail page with mobile-first hero + 2-col items grid on md+
- [chopnow-app #186](https://github.com/ChopNow-app/chopnow-app/issues/186) — landscape phone orientation backlog
- Brand color tokens used by these layouts live in `brand/` at the
  ChopNow workspace root (rouge / noir / blanc — Sugar Art Palier 4)
