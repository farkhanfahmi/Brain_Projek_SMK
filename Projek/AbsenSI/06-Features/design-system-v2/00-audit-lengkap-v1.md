---
tags: [absensi, design-system-v2, audit]
created: 2026-08-28
---

# Audit Lengkap Design System v1 -- Pemicu Revamp Total

Rangkuman semua temuan audit sebelum keputusan revamp total. Lihat 00-INDEX.md untuk konteks proyek revamp v2.

## Ringkasan Umum

Design system v1 (dibangun 2026-08-28 di Figma dari kode `packages/ui`) terbukti punya banyak kekurangan struktural saat dievaluasi sebagai "produk mandiri" -- bukan sekadar katalog komponen yang meniru kode.

## Temuan Kritis

1. Token Spacing tidak ada sebagai variable terpisah -- nilai padding/gap ditulis manual, tidak terikat token.
2. Tidak ada Grid/Layout/Breakpoint system -- padahal masalah awal proyek ini adalah responsivitas.
3. Interactive State Matrix nyaris kosong -- 14 dari 23 komponen cuma 1 varian, tidak ada hover/focus/pressed/disabled.
4. Component Properties nyaris tidak dipakai -- instance Figma tidak bisa diedit tanpa masuk ke komponen asli.

## Temuan Mayor

5. Tidak ada elevation/z-index formal (layering dropdown vs modal vs toast vs tooltip).
6. Tidak ada token motion/animasi (durasi, easing).
7. Ikon cuma placeholder geometris, bukan sistem ikon Lucide nyata.
8. Checkbox asli dan Radio Button tidak ada. Input cuma 3 state, tidak ada Error/Success.
9. Search Input -- pola krusial (tabel wajib search) tidak jadi komponen sendiri.
10. Toast/Snackbar, Alert/Banner, Empty State, Confirmation Dialog (beda dari Dialog generik) semua hilang.
11. Sidebar/Navigation, App Header, Breadcrumb, Page Template level halaman tidak ada.
12. Data visualization (Chart.js dipakai di modul Rekap) sama sekali tidak tercakup token warna/style.
13. DataTable cuma 1 state -- tidak ada loading/empty/mobile-card-view.

## Temuan Minor

14. Inkonsistensi naming axis varian (State vs Variant vs Shape tanpa aturan jelas).
15. Tidak ada dokumentasi aksesibilitas per komponen (rasio kontras, touch target minimum).
16. Tidak ada Do's and Don'ts / usage guidelines per komponen.
17. Tidak ada Density/Compact mode untuk tabel data-heavy.
18. Governance minim -- changelog kosong, tidak ada aturan kontribusi.

## Audit Warna (Detail)

Dihitung rasio kontras WCAG 2.1 aktual dari nilai `globals.css`:

- `text/tertiary` di atas `bg/page`: rasio 2.45:1 -- GAGAL AA (bahkan untuk teks besar butuh 3:1).
- `text/tertiary` di atas `bg/surface`: rasio 3.04:1 -- gagal untuk teks normal, lolos teks besar saja.
- `primary/default` (teks link oranye) di atas `bg/page`: rasio 3.67:1 -- gagal AA teks normal.
- Token `status-shipped` dan `status-processing` -- penamaan warisan template e-commerce, tidak sesuai domain absensi sekolah.
- Tidak ada color ramp 50-900 sistematis, cuma titik-titik diskrit (default/hover/soft-2/tint).
- Tidak ada dark mode sama sekali.
- Fallback font `Inter` di `tailwind.config.ts` adalah bug -- font itu tidak pernah dimuat browser (font asli Plus Jakarta Sans via next/font).

## Audit Typography (Detail)

- Type scale (32/24/18/16/14/13/12px) tidak mengikuti rasio matematis standar, sehingga sulit menambah ukuran baru secara konsisten.
- Font weight 800 dimuat tapi tidak pernah dipakai di 8 text style manapun.
- Tidak ada letter-spacing bervariasi (semua 0%), padahal display/heading besar idealnya sedikit negatif.
- Tidak ada varian tabular-nums untuk angka di tabel (jam, tanggal, nominal).
- Tidak ada responsive typography (scaling ukuran font antar breakpoint).

## Bukti Kuantitatif dari Audit Kode (Fase 0)

- 27 kemunculan warna hex hardcode di 11 file, 11 warna unik -- semuanya di luar sistem token.
- `kampus-map.tsx` masih pakai `#F5841F`, yaitu primary color versi SANGAT lama sebelum revisi WCAG 2026-07-23.
- `rekap-view.tsx` dan `rekap-guru-view.tsx` hardcode warna Material Design generik untuk chart status kehadiran.
- Komponen Table primitif (`packages/ui/table.tsx`) sebenarnya SUDAH punya `overflow-auto` di wrapper -- tapi 8 dari 9 halaman modul tidak memanfaatkannya (tidak pakai primitif ini secara konsisten).

## Keputusan yang Diambil dari Audit Ini

Product owner memutuskan REVAMP TOTAL (bukan tambal sulam v1), dengan batasan:
- Warna inti Terracotta `#C1452C` dan Cream `#FFFEE0` dikunci exact.
- Warna sekunder/netral/semantik boleh diubah bebas.
- Fokus data-dense UI (tabel, dashboard, chart).
- Rollout incremental per modul, bukan big-bang.
- Wajib ada escape hatch resmi untuk komponen baru di masa depan.
