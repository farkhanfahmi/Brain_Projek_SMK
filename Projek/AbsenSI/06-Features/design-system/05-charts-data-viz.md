# Charts & Data Visualization — EzMart Dashboard Style

> Recommended library for Next.js/React: **Recharts** (matches this style's needs well — line, radial bar, donut/pie, bar — with easy custom styling) or **visx** if more control is needed. Avoid heavy chart kits with default multi-hue palettes (e.g. Chart.js defaults) unless you override every series color per the ramp below.

## Global Chart Rules

- **Monochrome orange ramp only.** Every chart in this dashboard uses shades of the single primary orange, from darkest/most-saturated (`--color-primary` #F5841F) for the most important series/segment, down to lightest tint (`--color-primary-tint` #FDE9CE) for the least important. Never introduce blue/green/purple into a chart's data-series colors (green/red are reserved strictly for delta indicators outside the chart canvas).
- **No heavy gridlines.** Where present (line chart), gridlines are horizontal-only, very light (`--color-border-subtle` #EDE7DC), 1px, no vertical gridlines.
- **No chart border/frame.** The chart sits directly on the white card background; axis lines themselves are omitted or reduced to a faint baseline only.
- **Tooltips:** small white rounded rectangle (`radius-md` 14px, `shadow-popover`), appears on hover/tap over a data point, shows the exact label + value (e.g. "Revenue $14,521"), positioned above the point with a small connecting indicator dot on the line/bar itself (solid orange dot with white ring, per reference).
- **Legends:** simple dot + label pairs, top-left or top-right of the chart card, `text-caption` size, using the same orange-ramp colors as swatches.
- **Axis labels:** `text-caption` (12px), `--color-text-secondary`, never rotated/truncated — pick date/label formats short enough to fit horizontally (e.g. "12 Aug" not full ISO dates).

## Revenue Analytics — Line Chart

- Two series: **Revenue** (solid line, `--color-primary`, ~2.5px stroke) and **Order** (dashed line, `--color-primary-soft-2`, ~2px stroke, dash pattern `4 4`).
- Optional soft area-fill under the Revenue line using `--color-primary-soft` at ~30% opacity, fading to transparent — used only behind the currently-hovered/highlighted data point region (reference shows a soft vertical highlight band behind the active tooltip point), not a filled area under the whole line.
- Legend top-left: colored dot + "Revenue" / dashed dot + "Order".
- Filter control top-right: the pill dropdown ("Last 8 Days ⌄") per `03-components.md` secondary button spec.
- X-axis: date labels, short format ("12 Aug"..."19 Aug"). Y-axis: value labels with "K" suffix for thousands ("4K", "8K", "12K", "16K"), positioned left, light gray.

## Monthly Target — Radial/Gauge Chart

- Semi-circular (or ~270°) radial progress gauge, thick stroke (~14-16px), track color `--color-primary-soft`, progress-fill color `--color-primary`, rounded line-caps on both ends of the arc.
- Centered inside the arc: big bold percentage value (`text-display`, dark) + small delta caption below it ("+8.02% from last month", green text).
- Below the gauge: a short encouragement sentence in dark bold text with an inline celebratory emoji is acceptable here ONLY as copy, not as a UI icon ("Great Progress! 🎉") followed by a smaller gray descriptive sentence.
- Bottom of card: two side-by-side stat pills (Target / Revenue) per the Progress Pill spec in `03-components.md`.

## Top Categories — Donut Chart

- Multi-segment donut, thick ring (~24-28% of radius as stroke width), each segment a different shade from the orange ramp ordered by value descending (largest segment = darkest `--color-primary`, down to lightest `--color-primary-tint` for the smallest).
- Center label: small gray caption ("Total Sales") + bold dark large value ("$3,400,000") stacked, centered in the donut hole.
- Legend below the donut: list rows of `● label ......... $value`, small colored square/dot matching each segment's ramp color, label left-aligned, value right-aligned, `text-body` size.
- "See All" link top-right of card header, `text-label`, `--color-primary` colored text (the one allowed case of orange-colored text-as-link outside of buttons).

## Conversion Rate — Column/Funnel Chart

- 5 vertical columns representing a funnel (Product Views → Add to Cart → Proceed to Checkout → Completed Purchases → Abandoned Carts), decreasing in value/height left to right.
- Each column: small green/red delta badge above the bar top, bar itself colored from the orange ramp — NOTE the reference intentionally makes the **last** bar ("Abandoned Carts") the most saturated/darkest orange even though it's visually the "worst" outcome, to draw attention to it as an area needing action; the preceding bars use progressively lighter tints. Replicate this "attention via saturation" pattern rather than a strictly monotonic value-to-color mapping.
- Bars have flat baselines (sitting on an invisible axis line) and rounded top corners only (`radius-md`, 12px top-left/top-right, 0 bottom).

## Active User — Horizontal Progress List

- Simple ranked list (country name + percentage), each row followed by a thin horizontal progress bar (see `03-components.md` List Row spec): track `--color-primary-soft`, fill `--color-primary`, height 6px, `radius-full`.
- Header row: big bold total number ("2,758") + label ("Users") + green delta badge ("+8.02% from last month") top-right.

## Traffic Sources — Segmented Bar + Legend

- Single full-width horizontal bar, `radius-full`, divided into proportional segments (one per traffic source), each segment a different shade of the orange ramp ordered by size (Direct Traffic = darkest/largest segment down to Email Campaigns = lightest/smallest).
- Legend below as a vertical list: `● label` left, `percentage%` right, `text-body`/`text-caption` size, dot color matches the corresponding segment.

## Accessibility Notes for All Charts

- Because this style relies on a single-hue ramp, segments/series must ALSO differ by direct labeling (values shown via legend or on-chart text) — never rely on shade differences alone to convey which segment is which, since low-saturation distinctions can be hard to perceive. Always pair each colored element with a text label or legend entry (per `color-not-only` / `pattern-texture` rules).
- Ensure the darkest ramp color (`--color-primary` on white) and lightest ramp color (`--color-primary-tint` on white) both remain distinguishable — if contrast between adjacent ramp steps is too low for a given chart, add a thin 1px white stroke between donut/bar segments to visually separate them regardless of color contrast.
- Provide an accessible data table or `aria-label` summary for each chart's key insight (e.g., donut chart's `aria-label="Top categories by sales: Electronics $1,200,000 (largest)..."`) per `screen-reader-summary`.
