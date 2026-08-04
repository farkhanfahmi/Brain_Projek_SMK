# T083 — UI: Kalender Tampilan 12 Bulan Sekaligus (Grid 2x6)

## Depends on
Tidak ada — murni perubahan tampilan di `kalender-view.tsx`, logic hari libur/tahun ajaran tidak berubah.

## Context
- **App:** `apps/web`
- **File:** `apps/web/src/app/(admin)/kalender/kalender-view.tsx`, `apps/web/src/app/(admin)/kalender/calendar-utils.ts`
- **Ref:** Diminta user 2026-07-24 — kalender sekarang render 1 bulan per waktu dengan navigasi Previous/Next (`kalender-view.tsx:24,34-35`). User ingin 12 bulan (1 tahun penuh) ditampilkan sekaligus dalam grid diperkecil.

## Spec Detail

### Layout
- Grid **2 baris x 6 kolom** (12 mini-kalender bulan sekaligus) — sesuai instruksi user eksplisit "layout 2X6"
- Tiap sel grid = 1 mini-kalender bulan: nama bulan + grid tanggal kecil (angka saja, tanpa perlu label hari lengkap di tiap sel — cukup 1 baris header hari [S S R K J S M] disingkat 1-2 huruf karena ukurannya kecil)
- Hari libur (dari `school_holidays`) tetap ditandai visual (warna beda, pola sama seperti `HOLIDAY_COLOR` yang sudah ada di `calendar-utils.ts`) tapi dalam skala lebih kecil — titik warna kecil atau background sel, BUKAN badge teks penuh (tidak muat di ukuran mini)
- Klik tanggal manapun tetap membuka dialog tambah/edit hari libur (`dialogState`, fungsi existing) — behavior klik tidak berubah, cuma ukuran visual yang mengecil

### Navigasi
- Ganti navigasi Previous/Next BULAN jadi Previous/Next **TAHUN** (karena sekarang 1 layar = 1 tahun penuh) — 1 tombol mundur/maju 1 tahun, tombol "Hari Ini" tetap ada (scroll/highlight ke bulan berjalan dalam grid 12 bulan yang tampil)
- **Pertimbangkan tetap sediakan opsi kembali ke tampilan 1-bulan** (misal toggle kecil "Tampilan: Tahun / Bulan") KALAU grid 12 bulan ternyata terlalu padat untuk device kecil (laptop) — TIDAK WAJIB, tapi kalau ternyata UX buruk di layar sempit, pertimbangkan opsi ini sebelum menganggap task selesai. Uji dulu di ukuran layar admin (bukan mobile, ini bukan halaman diakses dari HP) sebelum memutuskan perlu tidaknya toggle ini.

### Responsive
- Grid 2x6 mungkin perlu breakpoint lebih kecil di layar sempit (misal 2x6 di desktop lebar, turun ke 3x4 atau 4x3 di layar lebih sempit) — pakai Tailwind responsive grid classes, JANGAN hardcode 6 kolom mutlak kalau merusak layout di resolusi umum

## JANGAN
- ❌ JANGAN ubah logic `buildMonthGrid`, `findHolidayForDate`, atau util tanggal lain di `calendar-utils.ts` — HANYA render/layout yang berubah
- ❌ JANGAN hilangkan kemampuan klik tanggal untuk tambah/edit hari libur — fungsi ini WAJIB tetap ada meski ukuran sel mengecil

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kalender/kalender-view.tsx` — ganti render single-month grid jadi loop 12 bulan dalam grid 2x6, navigasi jadi per-tahun
- **Kemungkinan modifikasi kecil:** `apps/web/src/app/(admin)/kalender/calendar-utils.ts` — HANYA kalau perlu helper baru (misal generate array 12 bulan dari 1 tahun), bukan mengubah logic yang ada

## Acceptance Criteria
- [ ] Halaman `/kalender` menampilkan 12 bulan sekaligus dalam grid 2 baris x 6 kolom (di layar desktop lebar)
- [ ] Tombol navigasi ganti ke geser tahun (bukan bulan)
- [ ] Klik tanggal manapun di bulan manapun tetap membuka dialog tambah/edit hari libur
- [ ] Hari libur tetap tertandai visual di semua 12 mini-kalender
- [ ] Layout tidak pecah/overflow di lebar layar admin standar (≥1280px)
