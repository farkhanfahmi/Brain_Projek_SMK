# MASTER Design System — "EzMart" Admin Dashboard Style

> **Source of truth.** Claude Code must read this file (and the companion files listed below) before writing any UI code for this project. Do not invent colors, spacing, or component shapes that are not defined here. If a needed pattern isn't covered, extrapolate using the same visual logic (rounded, soft, orange-accented, high-contrast text) rather than defaulting to generic Tailwind/shadcn defaults.

## Companion files (read all of them)
- [`01-colors.md`](./01-colors.md) — full color token table, usage rules, states
- [`02-typography.md`](./02-typography.md) — font stack, type scale, weight rules
- [`03-components.md`](./03-components.md) — buttons, cards, sidebar, badges, inputs, tables — exact specs
- [`04-layout-spacing.md`](./04-layout-spacing.md) — grid, spacing scale, radii scale, shadow scale
- [`05-charts-data-viz.md`](./05-charts-data-viz.md) — chart-specific styling (line, donut, bar, segmented bar)

## Product Type & Reference
- **Product type:** SaaS / E-commerce admin analytics dashboard (order, sales, customer, product management).
- **Reference screenshot:** "EzMart" dashboard — sidebar nav + top bar + KPI cards + charts grid.
- **Stack:** Next.js (App Router) + React + TypeScript. Use Tailwind CSS for styling (utility classes matching the tokens below). Icons: `lucide-react` (NOT emoji, NOT filled/solid mixed with outline).

## Design Philosophy (the "feel" to replicate)
1. **Warm, soft, approachable enterprise UI** — not sterile corporate blue/gray. The overall mood comes from a warm neutral (beige/tan) canvas that makes white cards "float."
2. **One dominant accent color** (vivid orange) used sparingly but decisively: primary buttons, active nav state, icon chips, chart primary series, and small data highlights. Everything else is neutral (white, near-black text, mid-gray secondary text).
3. **Very rounded geometry.** Corner radii are large and consistent — this is a "soft/friendly" enterprise aesthetic, not brutalist or sharp-cornered. Circles/pills appear frequently (icon chips, avatar, filter buttons, donut chart).
4. **Card-based composition.** Every discrete unit of information lives inside a white rounded rectangle floating on the beige page background, with generous internal padding and a soft, low-opacity shadow (never a hard border as the primary separator — shadow + white-on-beige contrast does the separating).
5. **Numbers are the hero.** KPI values are large, bold, near-black, and given more visual weight than their labels. Supporting metadata (deltas, comparisons) is small and color-coded (green = positive, red = negative) with an icon-free, text-only badge or plain colored text.
6. **Data density is high but never cramped** — achieved through consistent internal card padding (24px) and a strict 24px gutter between cards, not by shrinking type.

## Anti-Patterns (explicitly avoid)
- Do NOT use pure white or pure gray (#F5F5F5 / #FAFAFA) as the page background — it must be the warm beige tone defined in `01-colors.md`. This is the single most identity-defining choice of this style.
- Do NOT use sharp/small corner radii (4px, 6px) anywhere — this style reads as "friendly SaaS," not "flat enterprise tool."
- Do NOT use emoji as icons. The reference uses a small celebratory emoji (🎉) only once, inline in a sentence of encouragement copy ("Great Progress! 🎉") — never as a UI control or structural icon.
- Do NOT introduce a second accent color (e.g., blue for links, purple for tags). Orange is the only brand accent. Green/red are reserved exclusively for positive/negative delta indicators.
- Do NOT use heavy/dark drop shadows or hard 1px borders as the primary card separator. Shadows are soft, diffuse, and light.
- Do NOT left-align KPI numbers with tiny label text of similar size — hierarchy must be obvious (label small+gray on top, value large+bold+dark below, delta small+colored to the side).
- Do NOT use flat, single-tone bar/donut charts — the reference always uses a monochrome orange ramp (dark orange → light peach) across chart segments/bars, never arbitrary multi-hue palettes.

## Page Archetype in Reference
The reference is a **Dashboard / Overview** page:
- Fixed left sidebar (white, ~240px) with logo, primary nav list, secondary nav list, no footer content visible.
- Top bar: page title (left), global search input (center-right), icon buttons (chat, notifications w/ red dot), user profile block with avatar + name + role + chevron (far right).
- Content area on beige background, organized in a responsive grid of cards: a 3-up KPI row + 1 "Top Categories" donut card spanning full height on the right; a 2-up analytics row (line chart + radial gauge); a 3-up bottom row (active users list, funnel/conversion metrics, traffic sources segmented bar + legend).

When building other pages (Orders, Products, Customers, etc.) for this project, reuse the same shell (sidebar + top bar) and the same card/KPI/table visual language defined here, adapting only the content.
