# T059 — Komponen Generik: Activity Feed

## Depends on
Tidak ada dependency schema/API — murni komponen presentasional. Tapi pemakaian NYATA-nya menunggu TV Piket (backlog Fase 2 prioritas #3, lihat `tv-piket.md`) atau Dashboard Kepsek (Fase 3) — task ini membuat komponennya lebih dulu supaya siap dipakai begitu fitur itu mulai dikerjakan.

## Objective
Buat komponen reusable "Activity Feed" di `packages/ui` sesuai spec `03-components.md` bagian "Activity Feed" — list kronologis event dengan icon chip berwarna + teks 2-baris, untuk dipakai di TV Piket ("guru izin terbaru", "siswa tap terbaru") dan Dashboard Kepsek ke depan.

## Context
- **App:** `packages/ui`
- **Ref:** `Projek/AbsenSI/06-Features/design-system/03-components.md` — bagian "Activity Feed" (ditambahkan 2026-07-22 dari audit gambar referensi EzMart asli)

## Spec Detail

### Komponen baru: `packages/ui/src/components/activity-feed/`

**`ActivityFeedCard`** — wrapper card container:
- Props: `title` (string), `maxHeight` (opsional, default sesuatu yang masuk akal misal `320px`) — card TIDAK BOLEH tumbuh unbounded kalau list panjang, harus scroll di dalam card
- Styling: Global Card spec (`radius-xl` 24px, padding 24px, `shadow-card`)

**`ActivityFeedItem`** — 1 baris event:
- Props: `iconChipColor: 'primary-soft' | 'success' | 'danger' | 'neutral'` (semantic type event — TIDAK bebas hex, harus salah satu token ini), `icon: ReactNode` (dari `lucide-react`), `title: string` (baris 1, deskripsi event), `timestamp: string` (baris 2, `text-label` abu-abu)
- Layout: icon chip 32px (`radius-md` 14px) kiri, 2-line text block kanan
- Spacing antar item: `space-3` (12px), TANPA hairline divider (beda dari Data Table — ini list bebas, bukan tabular grid, sesuai catatan eksplisit di `03-components.md`)

### Contoh pemakaian yang diantisipasi (untuk referensi implementasi, BUKAN scope task ini)
- TV Piket: "Budi Santoso tap masuk — 07:15" (icon chip `success`), "Ahmad S. izin sakit — Kelas XI-RPL-1 kosong tanpa tugas" (icon chip `danger`)
- Dashboard Kepsek (Fase 3): log aktivitas across sekolah

## JANGAN
- ❌ JANGAN implementasikan TV Piket atau Dashboard Kepsek di task ini — itu task terpisah (belum dibreakdown, backlog Fase 2 #3 dan Fase 3). Task ini HANYA komponen presentasional generik
- ❌ JANGAN buat `iconChipColor` bebas hex — harus salah satu dari token semantic yang sudah ada (`primary-soft`/`success`/`danger`/neutral pakai `surface-subtle`)
- ❌ JANGAN buat card ini tumbuh tanpa batas tinggi — WAJIB scrollable dengan max-height, sesuai catatan di `03-components.md` ("never let the card grow unbounded")

## Files
- **Buat:** `packages/ui/src/components/activity-feed/activity-feed-card.tsx`
- **Buat:** `packages/ui/src/components/activity-feed/activity-feed-item.tsx`

## Acceptance Criteria
- [ ] Komponen render dengan data dummy (misal 10 item) — item ke-6 dst tidak terlihat sampai di-scroll di dalam card, card sendiri tidak melebihi `maxHeight`
- [ ] Icon chip warna cocok dengan salah satu dari 4 varian token yang diizinkan, tidak ada warna lain
- [ ] Tidak ada hairline divider antar item (beda pattern dari Data Table T058)
- [ ] Komponen bisa diimpor dari `packages/ui` tanpa error type

## Handoff
Dipakai oleh task TV Piket (belum dibreakdown, lihat `Projek/AbsenSI/06-Features/tv-piket.md`) dan Dashboard Kepsek (Fase 3) — saat task itu mulai dibreakdown, referensikan komponen ini sebagai basis "daftar siswa tidak hadir" / "status guru real-time" alih-alih membuat list custom baru.
