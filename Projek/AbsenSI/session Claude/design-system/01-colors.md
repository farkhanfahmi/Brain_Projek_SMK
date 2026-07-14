# Color Tokens — EzMart Dashboard Style

> Use these as CSS variables / Tailwind theme extensions. Never hardcode raw hex in components — reference the semantic token name.

## Core Palette

| Token | Hex | Usage |
|---|---|---|
| `--color-bg-page` | `#EEE6D9` | App/page background (warm beige/tan). Applied to `<body>` / outer layout wrapper. |
| `--color-bg-surface` | `#FFFFFF` | All cards, sidebar, top bar, modals, dropdowns. |
| `--color-bg-surface-subtle` | `#F7F3EC` | Inner sub-panels inside a card that need slight separation without a border (rare use). |
| `--color-primary` | `#F5841F` | Brand accent — vivid warm orange. Primary buttons, active nav item bg, icon-chip bg, primary chart series, progress fill, focus rings. |
| `--color-primary-hover` | `#E3760F` | Hover/active state of primary orange elements (slightly darker/more saturated). |
| `--color-primary-soft` | `#FCEEDD` | Soft tint of primary — used for the "Total Sales" KPI card background, "Target" pill background, chart area-fill under lines. |
| `--color-primary-soft-2` | `#FBDCB8` | Mid-tint orange — used for donut chart secondary ring segment, bar-chart mid bars, traffic-source segmented bar segments. |
| `--color-primary-tint-light` | `#FDE9CE` | Lightest orange tint — donut chart lightest segment, funnel's shortest/last bar. |
| `--color-text-primary` | `#171412` | Headings, KPI numbers, primary body text. Near-black, warm-toned (not pure `#000`). |
| `--color-text-secondary` | `#8A8580` | Labels, captions, muted metadata ("vs last week", axis labels, placeholder text). Warm gray. |
| `--color-text-tertiary` | `#B7B2AB` | Disabled text, faint helper copy. |
| `--color-border-subtle` | `#EDE7DC` | Rare hairline dividers (e.g., sidebar nav separator line). Always subtle, never a dark gray border. |
| `--color-success-bg` | `#DCF5E3` | Positive delta badge background. |
| `--color-success-text` | `#1E9E4C` | Positive delta text/icon ("+3.34%", "+8.02%"). |
| `--color-danger-bg` | `#FBE2E1` | Negative delta badge background. |
| `--color-danger-text` | `#E13B3B` | Negative delta text ("-2.89%", "-5%"). |
| `--color-white` | `#FFFFFF` | Text-on-primary (buttons, active sidebar item label), card surfaces. |
| `--color-shadow` | `rgba(23, 20, 18, 0.06)` | Base card shadow color (see shadow scale in `04-layout-spacing.md`). |

## Usage Rules

1. **Page vs. card contrast is the core visual signature.** `--color-bg-page` (beige) must always be visible as gutter/margin around and between `--color-bg-surface` (white) cards. Never make the page background white.
2. **Orange is a spotlight, not a wash.** Only apply `--color-primary` (full-strength) to: the active sidebar nav pill, primary CTA buttons, small icon-chip squares (e.g., dollar-sign icon on the Total Sales card), the filled arc of the radial gauge, and the darkest ring of the donut chart. Do not tint entire cards or large surfaces with full-strength orange except the one KPI "hero" card (Total Sales in the reference), which uses `--color-primary-soft` as its full card background — this is a deliberate one-card emphasis pattern, not something to repeat on every card.
3. **Deltas are always color + text, never color alone.** Positive = green text on transparent or `--color-success-bg` pill; negative = red text on transparent or `--color-danger-bg` pill. Always prefix with `+`/`-` and pair with "vs last week" / "vs last month" gray caption underneath or beside it.
4. **Chart color ramp is monochrome-orange.** Any chart with multiple series/segments (donut, bar, segmented horizontal bar) must use a single-hue ramp from `--color-primary` (darkest/most important segment) down through `--color-primary-soft-2` to `--color-primary-tint-light` (lightest/least important). Do not introduce blue, purple, teal, etc. into charts.
5. **Dark mode (if requested later):** invert to a warm near-black page background (e.g. `#1C1815`), card surfaces at `#26211D`, keep `--color-primary` identical (it already has enough contrast on dark), flip text tokens to light warm-grays. Do not simply CSS-invert; re-derive per `color-dark-mode` rule (desaturated tonal shift, not literal inversion).

## Tailwind Config Snippet

```ts
// tailwind.config.ts (excerpt)
colors: {
  page: '#EEE6D9',
  surface: '#FFFFFF',
  'surface-subtle': '#F7F3EC',
  primary: {
    DEFAULT: '#F5841F',
    hover: '#E3760F',
    soft: '#FCEEDD',
    'soft-2': '#FBDCB8',
    tint: '#FDE9CE',
  },
  ink: {
    DEFAULT: '#171412',
    secondary: '#8A8580',
    tertiary: '#B7B2AB',
  },
  success: { bg: '#DCF5E3', text: '#1E9E4C' },
  danger:  { bg: '#FBE2E1', text: '#E13B3B' },
}
```
