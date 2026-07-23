# T058 — Komponen Generik: Data Table dengan Status Badge Workflow

## Depends on
T057 (token warna status-shipped/status-processing harus sudah ada)

## Objective
Buat pattern/wrapper reusable "Data Table" di `packages/ui` yang menstandarkan struktur tabel data (header sortable, row hover, status badge workflow) sesuai spec `03-components.md` bagian "Data Table" — dipakai ulang oleh Rekap Kehadiran, Riwayat Izin Guru (T051), dan tabel manapun ke depan yang butuh pola serupa.

## Context
- **App:** `packages/ui` (komponen shared), dikonsumsi `apps/web`
- **Ref:** `Projek/AbsenSI/06-Features/design-system/03-components.md` — bagian "Data Table" (ditambahkan 2026-07-22 dari audit gambar referensi EzMart asli, baca detail lengkap sebelum mulai)
- **Komponen existing yang jadi basis:** `packages/ui/src/components/ui/table.tsx` (shadcn base), `packages/ui/src/components/ui/badge.tsx` — task ini EXTEND komponen itu, bukan bikin dari nol

## Spec Detail

### Komponen baru: `packages/ui/src/components/data-table/`

**`DataTableCard`** — wrapper card container:
- Props: `title` (string), `headerAction` (ReactNode opsional — search input/dropdown filter di kanan header, sesuai spec "search-in-card" dari gambar referensi)
- Styling: Global Card spec (`radius-xl` 24px, padding 24px, `shadow-card`)

**`DataTableHeader`** — header kolom dengan sort indicator:
- Props: array kolom `{ label: string, sortable?: boolean }`
- Render: `text-label` (13px, `text-ink-secondary`), border bottom `1px solid border`, ikon 2 chevron kecil kalau `sortable: true` (pakai `lucide-react` `ChevronsUpDown` atau serupa, ukuran 12px, warna `text-ink-tertiary`)

**`DataTableRow`** — baris data:
- `text-body` (14px), padding vertikal 16px, hairline border bawah (bukan antar-card shadow — ini SATU-SATUNYA tempat di sistem desain ini hairline border jadi pemisah utama, sesuai catatan di `03-components.md`)

**`DataTableEntityCell`** — sel untuk entitas dengan thumbnail (misal nama siswa+foto, produk+ikon):
- Props: `thumbnail` (url/ReactNode), `label` (string)
- Thumbnail: rounded-square 32px (`radius-md` 14px)

**`StatusBadge`** — komponen badge status workflow, generalisasi dari token T057:
- Props: `variant: 'success' | 'danger' | 'shipped' | 'processing' | 'neutral'`, `label: string`
- Styling per varian pakai token yang sesuai (`bg-success-bg text-success-text`, `bg-status-shipped-bg text-status-shipped-text`, dst — `neutral` pakai `bg-surface-subtle text-ink-secondary`)
- **Ini KOMPONEN GENERIK** — dipakai untuk status workflow apapun ke depan (bukan cuma order-style), tapi HANYA untuk konteks status/kategori tabel, bukan delta KPI (yang sudah ada pattern sendiri dari komponen KPI existing)

### Migrasi konsumen existing (opsional tapi direkomendasikan)
- **`izin-table.tsx`** (T051, sudah ada) — pertimbangkan migrasi ke `DataTableCard`/`DataTableRow`/`StatusBadge` generik ini, supaya tidak ada 2 implementasi tabel yang mirip tapi terpisah. Kalau migrasi terlalu berisiko mengubah behavior yang sudah jalan, boleh ditunda — evaluasi saat implementasi, tidak wajib di scope task ini kalau ada risiko regresi

## JANGAN
- ❌ JANGAN buat ulang primitif `Table`/`Badge` dari nol — extend/compose dari yang sudah ada di `packages/ui/src/components/ui/table.tsx` dan `badge.tsx`
- ❌ JANGAN buat `StatusBadge` varian selain 5 yang disebut (`success`/`danger`/`shipped`/`processing`/`neutral`) tanpa instruksi eksplisit — sesuai batasan token T057, jangan buka jalan warna baru sembarangan
- ❌ JANGAN paksa migrasi `izin-table.tsx` kalau berisiko regresi — evaluasi dulu, laporkan keputusan di PR/commit message kalau ditunda

## Files
- **Buat:** `packages/ui/src/components/data-table/data-table-card.tsx`
- **Buat:** `packages/ui/src/components/data-table/data-table-header.tsx`
- **Buat:** `packages/ui/src/components/data-table/data-table-row.tsx`
- **Buat:** `packages/ui/src/components/data-table/entity-cell.tsx`
- **Buat:** `packages/ui/src/components/data-table/status-badge.tsx`
- **Modifikasi (opsional):** `apps/web/src/app/(admin-jurnal)/admin-jurnal/izin/components/izin-table.tsx` — migrasi kalau aman

## Acceptance Criteria
- [ ] `StatusBadge` dengan varian `shipped`/`processing` render warna yang cocok dengan token T057 (verifikasi visual via Playwright screenshot)
- [ ] `DataTableRow` pakai hairline border sebagai pemisah, TIDAK ada shadow/border tebal
- [ ] Komponen bisa diimpor dari `packages/ui` dan dipakai di `apps/web` tanpa error type
- [ ] Tidak ada duplikasi definisi warna badge — semua rujuk token Tailwind (`bg-success-bg`, dst), tidak ada hex hardcode di komponen ini

## Handoff
Komponen ini jadi basis untuk Rekap Kehadiran (existing Fase 1, kalau nanti direstyle) dan tabel manapun di task-task Fase 2 lanjutan. Task berikutnya yang butuh tabel data baru harus rujuk komponen ini dulu sebelum bikin implementasi tabel baru dari nol.
