# Component Specs — EzMart Dashboard Style

All values below assume Tailwind CSS utility classes; exact px/rem given so Claude Code does not guess.

## Global Card (base unit for every panel)

```
background: var(--color-bg-surface);      /* #FFFFFF */
border-radius: 24px;                        /* rounded-3xl */
padding: 24px;                              /* p-6 */
box-shadow: 0 4px 24px rgba(23,20,18,0.05); /* soft, diffuse, no hard edge */
border: none;                               /* no visible border — shadow + bg contrast separates it from the beige page */
```
- Gap between cards in a grid: **24px** (`gap-6`).
- Card corner radius is **consistent everywhere** — 24px on large cards, 16px (`rounded-2xl`) on small nested elements (icon chips, pills, table row hover state).
- The one exception: the "hero" KPI card (Total Sales) uses `--color-primary-soft` (#FCEEDD) as its background instead of white, same radius/padding/shadow.

## Sidebar

- Width: `240px`, fixed, full height, background `--color-bg-surface` (white), sits on the page's beige background with its own card-like separation (subtle right shadow or the page bg shows around it with page padding).
- Logo block: icon mark (colored orange rounded-square glyph) + wordmark, bold, `text-h2` size, top-left, ~32px padding.
- Nav item (default): flex row, icon (20px, `--color-text-secondary` stroke) + label (`text-body-medium`, `--color-text-secondary`), 12px vertical padding, 16px horizontal padding, full-width, `border-radius: 16px`, no background.
- Nav item (active): background `--color-primary` (#F5841F), label + icon color `--color-white`, same radius (16px), bold weight (600) label. This is the ONLY solid-orange-fill nav treatment — do not add hover-orange to inactive items, only a very light neutral hover bg (`--color-bg-surface-subtle`).
- Nav groups: primary group (Dashboard, Orders, Products, Customers, Reports, Discounts) then a visual gap (~24px) then secondary group (Integrations, Help, Settings) — optionally separated by a `1px` `--color-border-subtle` divider.

## Top Bar

- Height: ~72px, background transparent/matches page area (sits directly above the card grid, not itself a card).
- Left: page title, `text-h1`, bold, dark.
- Center-right: search input — pill-shaped (`border-radius: 9999px`), background `--color-bg-surface`, no visible border, height 44px, padding-left 20px, placeholder icon (search glyph) right-aligned inside the field at 16px from the right edge, placeholder text `text-body`, `--color-text-secondary`.
- Right: icon buttons (chat bubble, bell) — circular, 40x40px, white background, centered icon 20px `--color-text-secondary`; the notification bell carries a small red dot badge (8px circle, `--color-danger-text` bg, positioned top-right of the icon, no border).
- Far right: user block — circular avatar (40px, `border-radius: 9999px`) + stacked text (name `text-h3` bold dark, role `text-label` gray below it) + chevron-down icon, all clickable as one unit, no visible container/border.

## Buttons

### Primary (filled orange pill) — e.g. "This Week"
```
background: var(--color-primary);
color: var(--color-white);
border-radius: 9999px;         /* fully pill-shaped */
padding: 10px 20px;
font-size: 14px; font-weight: 600;
box-shadow: none;
```
Hover: background → `--color-primary-hover`. Active/pressed: scale 0.98 + same darker bg, per `scale-feedback` motion rule.

### Secondary / dropdown filter (e.g. "Last 8 Days ▾")
```
background: var(--color-bg-surface);
color: var(--color-text-primary);
border-radius: 9999px;
padding: 10px 16px;
font-size: 14px; font-weight: 600;
border: 1px solid var(--color-border-subtle);
```
Includes a trailing chevron-down icon (16px).

### Icon-only button (top bar chat/bell, card overflow "⋯")
```
width/height: 40px (top bar) or 32px (in-card overflow);
border-radius: 9999px;
background: var(--color-bg-surface) or transparent;
icon size: 18-20px, color var(--color-text-secondary);
```
Minimum tap target 44×44px even if visual circle is smaller (use padding, not a shrunk hit area).

## Icon Chip (KPI card icon, e.g. dollar-sign square on Total Sales)
```
width/height: 44px;
border-radius: 14px;             /* rounded-2xl, NOT a circle — a rounded square */
background: var(--color-primary);
icon: 20px, stroke color var(--color-white);
```
Placed top-right inside the KPI card, opposite the label.

## KPI Card Layout
```
[label text, gray, top-left]                [icon chip, top-right]
[value, 32px bold dark]
[delta badge, colored]   [caption "vs last week", gray, small]
```
Delta badge: no pill background in the KPI row variant — just colored text (`+3.34%` green / `-2.89%` red) at `text-caption` size, bold, directly followed by the gray caption on the next line or inline (matches reference: value+delta same visual row, caption stacked below-right).

## Badge / Delta Pill (used elsewhere, e.g. table rows, funnel deltas)
```
display: inline-flex; align-items: center;
padding: 2px 8px;
border-radius: 9999px;
font-size: 12px; font-weight: 600;
background: var(--color-success-bg) | var(--color-danger-bg);
color: var(--color-success-text) | var(--color-danger-text);
```
Always includes explicit `+`/`-` sign, never color-only.

## Progress / Target Pills (bottom of "Monthly Target" card)
Two side-by-side rounded rectangles ("Target $600,000" / "Revenue $510,000"):
```
background: var(--color-primary-soft);   /* #FCEEDD */
border-radius: 16px;
padding: 12px 16px;
label: text-label gray on top, value: text-h3 bold dark below;
```

## List Row (Active User country breakdown)
```
row: flex justify-between, 8px vertical padding
label (country name): text-body, dark
value (percentage): text-body-medium, dark, right-aligned
below each row: horizontal progress bar, height 6px, border-radius 9999px,
  track background var(--color-primary-soft), fill background var(--color-primary),
  fill width = percentage value
```

## Table / Funnel Column (Conversion Rate card)
Vertical bar chart with 5 columns (Product Views → Abandoned Carts):
- Each column: label (`text-label`, gray) + big number (`text-h2` bold dark) + small green/red delta badge above a bar.
- Bars: width ~48-64px, `border-radius: 12px 12px 0 0` (rounded top only, flat bottom sitting on baseline), height proportional to value, fill color from the monochrome orange ramp (tallest = `--color-primary-soft-2`, shortest/last = `--color-primary` solid to draw attention to drop-off — match reference: last bar is the most saturated orange to flag "Abandoned Carts" as the attention point).

## Dropdown / Select (e.g. "Last 8 Days ⌄")
Same visual spec as Secondary Button above; opens a white rounded-2xl (16px) menu, 8px padding, soft shadow matching card shadow, list items 40px tall with 12px horizontal padding and `--color-bg-surface-subtle` hover state.
