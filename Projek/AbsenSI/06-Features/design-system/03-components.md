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

## Data Table (e.g. "Recent Orders")

> **Added 2026-07-22** — captured from a second reference screen not present in the original component pass. This is the most structurally complex component in the style; previous documentation missed it entirely.

```
Container: same Global Card spec (radius-xl 24px, padding 24px, shadow-card, white bg)
Header row: card title (text-h2) left, action cluster right —
  a small pill search input (see Top Bar search spec, but ~200px wide, sits INSIDE the card
  not the top bar) + a Secondary Button dropdown filter (e.g. "All Categories ⌄") beside it
```
- **Column header row:** each label paired with a small sort-indicator icon (two tiny chevrons stacked, `--color-text-tertiary`, 12px) — `text-label` weight/color, `padding: 12px 16px`, bottom border `1px solid --color-border-subtle` (the ONE place in this style a hairline border is the primary separator, because it's inside a table, not between cards).
- **Row:** `text-body` (14px) cells, `padding: 16px` vertical rhythm, no vertical column borders — rows separated by the same `1px --color-border-subtle` hairline only.
- **Product/entity cell:** small rounded-square thumbnail (32px, `radius-md` 14px, image or icon placeholder) + label text inline, left-aligned.
- **Status cell — the ONE exception to "success/danger only":** order/workflow status in a data table uses a **4-color categorical set**, distinct from the KPI-delta success/danger pair (which stays reserved for positive/negative deltas only). Use these as additional semantic tokens, all still warm/muted (never saturated primary hues), each a Badge Pill (`radius-full`, `text-caption` 12px/600):
  - `--color-status-shipped-bg` `#FDECD1` / `--color-status-shipped-text` `#B8720A` (warm amber — "in transit" states)
  - `--color-status-processing-bg` `#EDE3F7` / `--color-status-processing-text` `#7C4FC7` (muted violet — "in progress" states)
  - `--color-status-delivered-bg` = reuse `--color-success-bg`/`--color-success-text` (a completed/positive terminal state IS a delta-style positive, reuse the existing pair rather than adding a 5th hue)
  - `--color-status-pending-bg` `#FBE2E1` / `--color-status-pending-text` `#E13B3B` (reuse `--color-danger-bg`/`--color-danger-text` — pending/blocked reads as needs-attention)
  - **Rule of thumb:** only introduce a NEW categorical hue (amber, violet) when a table/list needs to distinguish 3+ non-binary workflow states that aren't simply "good" vs "bad" — this is different from a KPI card delta, a mode-A/B distinction, or any binary/positive-negative signal, which must still use the monochrome-orange-or-success/danger rule from `01-colors.md`. Do not add categorical hues to charts, KPI deltas, or A/B-style toggles — those stay strictly monochrome-orange or success/danger per the existing rule.
- Empty table state: centered icon (48px, `--color-text-tertiary`) + `text-body` gray message, vertically centered in a min-height ~240px area.

## Activity Feed (e.g. "Recent Activity")

- Same Global Card container.
- List of rows, each: small icon chip (32px, `radius-md` 14px, colored background matching the activity's semantic type — e.g. `--color-primary-soft` for a neutral/info event, `--color-success-bg` for a positive event) + 2-line text block (`text-body` bold-ish first line describing the event, `text-label` gray timestamp second line) — no dividers between rows, rely on consistent `12px` vertical spacing (`space-3`) instead of hairlines (unlike the Data Table above, this is a free-floating list, not a tabular grid).
- Scrollable within a fixed card height if the list overflows — never let the card grow unbounded.

## Hero Promo Card (sidebar, e.g. "Version X.Y is Ready")

- Sits at the bottom of the sidebar, full width, `radius-xl` (24px), solid `--color-primary` background (the ONE other place — besides the KPI hero card's soft tint — where a large surface is fully colored, this time full-strength not soft-tint).
- White bold heading (`text-h3`), 2-3 short bullet lines with small checkmark icons (white, 16px) confirming feature highlights, all white text at `text-body`/`text-label` size.
- Bottom: a white pill button (`radius-full`, white bg, `--color-primary` text — i.e. an inverted Primary Button, colors swapped so it still reads as the primary action against the solid-orange card).
- Use sparingly — this is a single fixed promotional slot, not a repeating pattern; do not replicate this treatment elsewhere in the sidebar or content area.

## Form Input Panjang (Sheet, bukan Dialog kecil)

> **Added 2026-07-22.** Ditemukan saat audit skema database: form `Tambah Guru`/`Tambah Siswa` existing pakai `Dialog` kecil (popup card sederhana, lebar tetap ~400-500px) untuk field yang sudah 11+ (siswa) dan akan bertambah ~9-13 lagi setelah field biodata lengkap ditambahkan (lihat `migrasi-database-lama.md`). **Popup Dialog kecil TIDAK LAGI memadai** untuk form sebanyak ini — pola baru berikut WAJIB dipakai untuk form dengan >6 field, menggantikan `Dialog` sebagai kontainer form data-entry panjang.

### Kontainer: Sheet (Panel Geser), bukan Dialog Tengah
- Form panjang (biodata siswa/guru, dst) dibuka sebagai **panel yang geser masuk dari sisi kanan layar** (`Sheet` — primitif BARU, belum ada di `packages/ui`, perlu ditambahkan, lihat companion task), bukan `Dialog` yang muncul di tengah layar dengan lebar tetap kecil.
- Lebar Sheet: **480-560px** pada desktop (jauh lebih lega dari Dialog ~400px), tinggi penuh viewport, konten di dalamnya **scrollable secara internal** (header dan footer/tombol aksi tetap terlihat/sticky, hanya body form yang scroll).
- Overlay/backdrop: sama seperti Dialog existing (`shadow-popover`, backdrop semi-transparan warm-tone `rgba(23,20,18,...)`, bukan hitam pekat).
- Radius: sisi yang menempel ke tepi layar (kanan) tidak dibulatkan, sisi kiri (yang menghadap konten) pakai `radius-xl` (24px) di sudut atas-bawah kiri saja.

### Struktur Internal: Section Bertahap dengan Tabs, bukan 1 Kolom Panjang Vertikal
- Field dikelompokkan jadi **section bermakna** (bukan 1 daftar vertikal panjang tanpa jeda) — contoh untuk form Siswa: "Data Pokok" (NISN, nama, kelas), "Biodata" (tempat/tanggal lahir, jenis kelamin, agama), "Kontak & Alamat" (no HP, alamat, RT/RW), "Data Wali" (nama & no HP ayah/ibu)
- Navigasi antar section pakai **Tabs horizontal** di bagian atas body Sheet (primitif BARU, belum ada di `packages/ui`, lihat companion task) — styling Tabs: pill-style (`radius-full`) mengikuti pola Secondary Button non-aktif / Primary Button untuk tab aktif, BUKAN underline-style tab generik
- Semua tab tetap dalam **1 form HTML tunggal** (submit sekali di akhir, bukan wizard multi-step terpisah) — tab murni navigasi visual untuk mengurangi kepadatan, bukan validasi bertahap yang mem-block pindah tab

### Field Layout dalam Tiap Section
- Grid 2 kolom pada desktop untuk field pendek (nama, NISN, no HP, dst) — `grid-cols-2 gap-4`, bukan 1 kolom penuh yang membuat Sheet terlalu panjang di-scroll
- Field panjang (alamat, textarea) tetap full-width (`col-span-2`)
- Tiap field: Label (`text-label`, `--color-ink-secondary`) di atas Input, sesuai pola existing yang sudah benar di `GuruForm`/`siswa-view.tsx` — TIDAK berubah, hanya konteks kontainer & pengelompokan yang berubah

### Footer Sticky
- Tombol "Batal" (Secondary Button) + "Simpan" (Primary Button) selalu terlihat di bawah Sheet (sticky, tidak ikut scroll bersama body form) — pola sama seperti `DialogFooter` existing, cuma posisinya sticky di bagian bawah Sheet bukan mengikuti alur dokumen

### Kapan TETAP Pakai Dialog Kecil (bukan Sheet)
- Form ≤6 field yang sederhana (misal form Mapel: nama+kode, form Kampus: nama saja) — Dialog kecil TETAP cocok, JANGAN dipaksa jadi Sheet kalau kesederhanaannya memang pas
- Konfirmasi aksi (hapus, aktivasi, dst) — tetap Dialog kecil sesuai pola existing
- Aturan praktis: **>6 field ATAU field yang butuh pengelompokan section bermakna → Sheet dengan Tabs. ≤6 field datar → Dialog kecil tetap dipertahankan.**

## In-Page Footer

- Appears only at the bottom of the scrollable content area (not fixed/sticky), NOT inside a card — sits directly on the beige page background like the Top Bar does.
- Left: `text-label` gray copyright line ("Copyright © 2024 EzMart") + inline links (Privacy Policy, Terms, Contact) same size/color, separated by simple spacing (no visible dividers).
- Right: row of circular icon-only social links, 32px, `--color-text-secondary` icon color, no background/border (transparent, matching Icon-only Button spec at the smaller size variant).
- Only appears on pages where content doesn't fill the viewport height naturally (e.g. dashboard with a long scroll) — do not force it to a sticky/fixed bottom bar.
