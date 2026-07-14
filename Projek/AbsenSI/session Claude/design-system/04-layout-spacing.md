# Layout, Spacing, Radius & Shadow Scale — EzMart Dashboard Style

## Page Shell

```
┌─────────────┬──────────────────────────────────────────────┐
│             │  Top bar (title, search, icons, profile)      │
│  Sidebar    ├──────────────────────────────────────────────┤
│  240px      │  Content area (beige bg, padded 24-32px)      │
│  fixed      │  ┌────────────┐┌────────────┐┌────────────┐   │
│             │  │ KPI card   ││ KPI card   ││ KPI card   │┌─┐│
│             │  └────────────┘└────────────┘└────────────┘│T││
│             │  ┌───────────────────────┐┌───────────────┐│o││
│             │  │ Revenue Analytics     ││ Monthly Target ││p││
│             │  │ (line chart)          ││ (radial gauge) ││ ││
│             │  └───────────────────────┘└───────────────┘└─┘│
│             │  ┌────────────┐┌───────────────────────┐┌────┐│
│             │  │ Active User││ Conversion Rate        ││Traf││
│             │  └────────────┘└───────────────────────┘└────┘│
└─────────────┴──────────────────────────────────────────────┘
```

- Overall grid: CSS grid, 12-column, `gap-6` (24px) both axes.
- KPI row: 3 equal KPI cards + 1 tall card ("Top Categories") spanning the full height of KPI row + analytics row combined (i.e., it is visually a 2-row-tall card on the right rail).
- Analytics row: Revenue Analytics chart card takes ~2/3 width, Monthly Target gauge takes ~1/3 width.
- Bottom row: Active User (narrow), Conversion Rate (wide, center), Traffic Sources (narrow) — roughly 25% / 50% / 25% width split, all equal height.
- Content max-width: none required for an internal dashboard (fluid to viewport), but constrain to `max-w-[1600px] mx-auto` on ultra-wide monitors.
- Content area outer padding: 32px top/right/bottom, with sidebar handling its own left edge.

## Spacing Scale (4px base unit)

| Token | px | Usage |
|---|---|---|
| `space-1` | 4px | icon-to-label gaps, tightest spacing |
| `space-2` | 8px | badge internal padding, list-row gaps |
| `space-3` | 12px | button vertical padding, small card gaps |
| `space-4` | 16px | nav item padding, button horizontal padding |
| `space-6` | 24px | **card internal padding**, **grid gutter** (the two most-used values in this style) |
| `space-8` | 32px | page-level outer padding, section separation |
| `space-12` | 48px | rare, large vertical rhythm between major page sections |

This is a **dashboard/dense product** — use the high-density end of the spacing scale (16-64px range), not the spacious marketing-page range (24-96px).

## Border Radius Scale

| Token | px | Usage |
|---|---|---|
| `radius-sm` | 10px | small inline elements (chart tooltips) |
| `radius-md` | 14px | icon chips (rounded squares) |
| `radius-lg` | 16px | nav items, dropdown menus, progress-pill containers, list-item bars |
| `radius-xl` | 24px | **all primary cards** — the defining radius of this style |
| `radius-full` | 9999px | buttons, search input, avatar, badges, progress-bar tracks/fills |

Never use anything below 10px; never use 0/sharp corners anywhere in this style.

## Shadow Scale

| Token | Value | Usage |
|---|---|---|
| `shadow-card` | `0 4px 24px rgba(23,20,18,0.05)` | default resting card |
| `shadow-card-hover` | `0 8px 32px rgba(23,20,18,0.08)` | hover state on interactive cards (e.g. clickable list rows) |
| `shadow-popover` | `0 12px 40px rgba(23,20,18,0.12)` | dropdown menus, tooltips (needs to sit visually above cards) |

Shadows are always warm-toned (using the near-black-brown `#171412` as the shadow color at low opacity), never cool gray/blue shadows — this keeps shadows feeling consistent with the warm beige/orange palette.

## Responsive Behavior

- **Desktop (≥1280px):** full grid as described above, sidebar always visible/fixed.
- **Tablet (768-1279px):** sidebar collapses to icon-only rail (64px) or becomes an overlay drawer triggered by a hamburger in the top bar; KPI row wraps to 2-up; bottom row stacks to 1-up or 2-up.
- **Mobile (<768px):** sidebar becomes a slide-in drawer (off-canvas, triggered by hamburger); all card grids collapse to a single column; top bar search collapses to an icon that expands to full-width input on tap; KPI cards stack vertically full-width.
- Maintain `space-6` (24px) as the minimum gutter down to tablet size; reduce to `space-4` (16px) only at mobile widths to preserve edge-to-edge breathing room without wasting screen space.
- No horizontal scroll at any breakpoint — charts and tables must reflow (e.g., funnel bars can shrink in width, segmented traffic bar stays full-width and just gets shorter).
