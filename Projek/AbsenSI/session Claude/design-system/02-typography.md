# Typography — EzMart Dashboard Style

## Font Family

The reference uses a **geometric/humanist grotesque sans-serif** with slightly rounded terminals and confident, chunky bold weights on numerals — consistent with fonts like **"Plus Jakarta Sans"**, **"General Sans"**, or **"Inter"** at higher weights. Numerals are tabular-looking and read cleanly at large sizes.

**Recommendation for implementation:** use **Plus Jakarta Sans** (Google Fonts, free, closely matches the rounded-geometric feel of the reference) as the single font family for the entire UI — headings, body, and numerals. Do not pair a separate serif/display font; this style is single-family.

```ts
// app/layout.tsx
import { Plus_Jakarta_Sans } from 'next/font/google'

const jakarta = Plus_Jakarta_Sans({
  subsets: ['latin'],
  weight: ['400', '500', '600', '700', '800'],
  variable: '--font-sans',
  display: 'swap',
})
```

```css
/* globals.css */
body {
  font-family: var(--font-sans), system-ui, sans-serif;
}
```

If Plus Jakarta Sans is unavailable, fall back to **Inter** — never fall back to a default serif or a narrow condensed font; the roundness and width of the letterforms are part of the style.

## Type Scale

| Token | Size / Line-height | Weight | Usage |
|---|---|---|---|
| `text-display` | 32px / 1.2 | 700 (Bold) | KPI hero numbers ("$983,410", "85%", "$3,400,000") |
| `text-h1` | 24px / 1.3 | 700 (Bold) | Page title ("Dashboard") |
| `text-h2` | 18px / 1.4 | 700 (Bold) | Card titles ("Revenue Analytics", "Monthly Target", "Top Categories") |
| `text-h3` | 16px / 1.4 | 600 (Semibold) | Sub-section labels, user name in top bar ("Marcus George") |
| `text-body` | 14px / 1.5 | 400 (Regular) | Default body text, table cells, list values |
| `text-body-medium` | 14px / 1.5 | 500 (Medium) | Sidebar nav labels, emphasized inline text |
| `text-label` | 13px / 1.4 | 400 (Regular) | Card eyebrow labels ("Total Sales", "Total Orders"), muted captions ("vs last week", "Admin" role tag) |
| `text-caption` | 12px / 1.4 | 500 (Medium) | Delta badges ("+3.34%"), axis labels, chart legend labels |

## Hierarchy Rules

1. **KPI cards always follow this exact order/weight pattern:** small gray label (`text-label`, `--color-text-secondary`) → large bold dark value (`text-display`, `--color-text-primary`) → small colored delta (`text-caption`, green/red) with gray context caption ("vs last week") beside or beneath it at `text-label` weight/color.
2. **Card titles are bold and dark**, always paired with a lightweight action on the same row (a "See All" link, a dropdown like "Last 8 Days ▾", or a "⋯" overflow icon) styled at `text-body`/`text-label` weight in `--color-text-secondary` (or in the orange pill-button case, white-on-orange — see `03-components.md`).
3. **Never drop body text below 13px.** The reference maintains strong legibility; the smallest text used (badge percentages, legend items) is still ~12–13px, never 10–11px.
4. **Numerals should look tabular/aligned** in KPI values and table columns — use `font-variant-numeric: tabular-nums` so values don't jitter in width across cards.
5. **Letter-spacing:** default/normal tracking throughout. Do not tighten tracking on headings — the rounded font shape depends on natural spacing to read as "friendly."
