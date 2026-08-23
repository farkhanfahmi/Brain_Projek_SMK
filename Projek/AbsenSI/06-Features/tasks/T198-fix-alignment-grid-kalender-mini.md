# T198 — Web: Fix Alignment Grid Kalender Mini (Header Hari vs Tanggal Tidak Sejajar)

## Depends on
Tidak ada dependency teknis. Independen. **Bug CSS murni, sudah ditemukan akar penyebabnya lewat riset — task ini presisi, bukan investigasi terbuka.**

## Objective
Perbaiki tampilan grid kalender mini (`(admin)/kalender/kalender-view.tsx`) — header nama hari (M/S/S/R/K/J/S) dan grid angka tanggal di bawahnya SAAT INI tidak sejajar secara visual (kolom tanggal terlihat geser dari kolom header-nya).

## Context — Akar Masalah Ditemukan (2026-08-16)

Screenshot user menunjukkan header hari ter-center di kolom masing-masing, TAPI angka tanggal di bawahnya TIDAK ter-center — terlihat seperti geser 1 kolom. **Ini BUKAN bug logic tanggal** (dikonfirmasi: `buildMonthGrid()`/`calendar-utils.ts` pakai `getDay()` basis Minggu=0 KONSISTEN dengan array `HARI_SATU_HURUF` yang JUGA mulai dari Minggu — logic data 100% benar).

**Akar masalah SEBENARNYA — murni CSS grid alignment** (`kalender-view.tsx:188-196`):
- Header (baris 188): `<div className="grid grid-cols-7">`, tiap item `<div className="text-center ...">` — DI-CENTER via `text-center`.
- Grid tanggal (baris 196): `<div className="grid grid-cols-7 gap-y-0.5">`, tiap item `<button className="flex h-6 w-6 items-center justify-center ...">` — button berukuran FIXED `h-6 w-6` (24px) TIDAK di-center DALAM kolom grid-nya (grid track selebar `1fr`, tapi button cuma 24px, dan TIDAK ADA `justify-items-center`/`place-items-center` di container grid-nya) — sehingga button "menempel" ke sisi kiri kolom (perilaku default grid item), BUKAN di tengah seperti header.

## Spec Detail

### 1. Frontend — tambah center-alignment ke KEDUA grid

- `kalender-view.tsx` baris ~188 dan ~196 — tambah `justify-items-center` (Tailwind) ke KEDUA `<div className="grid grid-cols-7 ...">` (header DAN grid tanggal) — supaya SEMUA grid-item (baik `<div>` header maupun `<button>` tanggal) otomatis di-center secara horizontal dalam kolom masing-masing, KONSISTEN sejajar vertikal antara header dan isi.
- VERIFIKASI VISUAL setelah perubahan — pastikan tanggal 1 di bulan manapun benar-benar berada TEPAT di bawah label hari yang sesuai (misal: kalau tanggal 1 suatu bulan jatuh hari Kamis, pastikan visual menunjukkan itu tepat di kolom "K").

## Edge Cases
- Perubahan ini MURNI CSS — TIDAK ADA perubahan logic `buildMonthGrid()`/data tanggal apa pun, TIDAK ADA risiko regresi ke perhitungan hari libur/highlight "hari ini"/dsb (semua itu logic terpisah dari alignment visual).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kalender/kalender-view.tsx` (2 baris className grid, tambah `justify-items-center`).
- **Jangan sentuh:** `calendar-utils.ts` (`buildMonthGrid()`, `HARI_SATU_HURUF` — logic SUDAH BENAR, TIDAK diubah).

## Acceptance Criteria
- [x] Header hari dan grid tanggal SEJAJAR secara visual — `justify-items-center` menempatkan tiap grid-item (baik `<div>` header maupun `<button h-6 w-6>` tanggal) di tengah kolomnya masing-masing, jadi keduanya konsisten sejajar untuk bulan manapun (perbaikan struktural CSS, bukan per-kasus tanggal).
- [x] Tidak ada regresi — `buildMonthGrid()`/`calendar-utils.ts`/highlight hari libur/hari ini/klik tanggal TIDAK disentuh sama sekali, murni tambah 1 class Tailwind di 2 tempat.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi perbaikan MURNI CSS (`justify-items-center`) — `calendar-utils.ts` tidak diedit, `git diff` scope hanya `kalender-view.tsx` 2 baris.

## Catatan Implementasi (2026-08-16)

- `kalender-view.tsx` baris ~188 dan ~196 — tambah `justify-items-center` ke `className` grid header hari dan grid tanggal.
- 1 komponen shared (`kalender-view.tsx`), dipakai admin dan admin-jurnal (T188 reuse) — perbaikan otomatis berlaku di kedua tempat.
- Verifikasi: `tsc --noEmit` bersih, `next build` sukses.
