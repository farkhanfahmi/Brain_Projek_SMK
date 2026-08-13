# T159 — Schema+API+Web: Toggle Mode Jadwal (Normal/Blok) Per Kelas/Jurusan

## Depends on
Tidak ada dependency teknis wajib ke T158. Independen (T158 soal RESOLUSI JAM, task ini soal MODE JADWAL — 2 konsep berbeda yang KEBETULAN sama-sama terkait domain jadwal, tapi tidak saling bergantung secara data).

## Objective
Admin bisa menentukan, **per kelas/jurusan**, apakah kelas itu memakai **Mode Normal** (jadwal sama setiap minggu) atau **Mode Blok** (jadwal bisa berbeda Minggu A vs Minggu B, mengikuti kalender blok yang SUDAH ADA di sistem).

## Context — Infrastruktur SUDAH ADA, Task Ini Mengisi 1 Gap Spesifik

Riset 2026-08-11 mengonfirmasi SEBAGIAN BESAR infrastruktur mode blok SUDAH ADA dan BERFUNGSI:

- **`BlockWeekRange`** (`schema.prisma:840-854`) — kalender "tanggal X sampai Y = Minggu A/B", SATU untuk SELURUH SEKOLAH per semester (dikonfirmasi user: TIDAK perlu kalender terpisah per kelas — 1 kalender blok global sudah cukup, TIDAK diubah task ini).
- **`Semester.mode`** (`ScheduleMode`: `blok | normal`) — SUDAH ADA di level SEMESTER.
- **`Schedule.minggu`** (`MingguTag`: `A | B | setiap_minggu`) — SUDAH ADA PER-BARIS JADWAL, dipakai `ScheduleResolverService.getJadwalUntukTanggal()` untuk filter jadwal mana yang berlaku di tanggal tertentu (kalau semester mode blok, cocokkan `minggu` jadwal dengan `mingguAktif` hasil lookup `BlockWeekRange`).
- **Admin-jurnal sudah bisa kelola** kalender blok (`(admin-jurnal)/admin-jurnal/jadwal-blok/`) DAN set `minggu` per jadwal saat create/edit `Schedule` (`(admin-jurnal)/admin-jurnal/jadwal/`).

**GAP YANG SEBENARNYA** (dikonfirmasi diskusi 2026-08-11): field `Semester.mode` ada di level SEMESTER (SATU mode untuk SEMUA kelas dalam semester itu) — TAPI user secara eksplisit ingin mode ini **PER KELAS/JURUSAN**, BUKAN 1 mode global untuk seluruh semester (verbatim: "per kelas/jurusan, admin set manual satu-satu... karena kebutuhannya memang beda, TKR pakai blok karena praktik, yang lain mungkin tidak"). Jadi 1 semester BISA punya campuran: sebagian kelas mode Normal, sebagian mode Blok — SEMUANYA tetap mengikuti KALENDER BLOK GLOBAL YANG SAMA (`BlockWeekRange`) KAPAN saja kelas mode-Blok itu perlu tahu "sekarang Minggu A atau B".

## Spec Detail

### 1. Schema (Prisma) — tambah field ke `Kelas`

```prisma
// Tambah ke model Kelas yang sudah ada:
modeJadwal ScheduleMode @default(normal) @map("mode_jadwal")
```
- REUSE enum `ScheduleMode` (`blok | normal`) yang SUDAH ADA (dipakai `Semester.mode`) — JANGAN buat enum baru terpisah.
- Default `normal` — kelas yang belum di-set eksplisit tetap berperilaku SEPERTI SEKARANG (regresi nol untuk kelas yang tidak butuh mode blok).
- Migration baru.

### 2. Backend — perbarui logic resolusi jadwal untuk baca dari `Kelas`, bukan `Semester`

- `ScheduleResolverService.getJadwalUntukTanggal()` (`apps/api/src/schedule-resolver/schedule-resolver.service.ts:124-151`) — SAAT INI kemungkinan besar cek `semester.mode` untuk MEMUTUSKAN apakah perlu filter `minggu` sama sekali. GANTI logic ini: untuk SETIAP `Schedule` yang di-resolve, filter berdasarkan **`schedule.kelas.modeJadwal`** (BUKAN lagi `semester.mode` global) — kalau kelas itu `modeJadwal: blok`, filter `schedule.minggu` terhadap `mingguAktif` (SAMA seperti sekarang); kalau `modeJadwal: normal`, JANGAN filter minggu sama sekali (semua jadwal kelas itu berlaku terlepas dari Minggu A/B — meskipun `schedule.minggu` mungkin terisi `setiap_minggu` secara konvensi, method resolusi TIDAK BOLEH bergantung pada nilai ini untuk kelas mode Normal, karena mode-nya sendiri sudah menentukan).
- **VERIFIKASI DAN BACA ULANG method ini SECARA LENGKAP sebelum ubah** — PASTIKAN paham PERSIS bagaimana `semester.mode` dipakai sekarang sebelum diganti sumbernya, supaya tidak ada logic tersembunyi yang terlewat.
- **`SchedulesService.create()`/`update()`** — validasi field `minggu` (WAJIB diisi kalau mode blok, `resolveMinggu`) SAAT INI kemungkinan mengecek `semester.mode` — GANTI jadi cek `kelas.modeJadwal` (dari `dto.kelasId` yang di-assign ke jadwal itu) — KONSISTEN dengan sumber kebenaran baru.
- `Semester.mode` field — TIDAK DIHAPUS dari schema (breaking change tidak perlu), TAPI SETELAH task ini, field ini KEMUNGKINAN JADI TIDAK TERPAKAI LAGI sebagai sumber logic (kalau semua resolusi pindah ke `Kelas.modeJadwal`) — EVALUASI saat implementasi apakah field `Semester.mode` masih perlu dipertahankan sebagai DEFAULT untuk kelas baru (opsional, TIDAK WAJIB), atau benar-benar jadi vestigial. KALAU ragu, JANGAN hapus field itu (aman dibiarkan tidak terpakai daripada breaking migration), CUKUP laporkan sebagai temuan ke user.

### 3. Frontend — toggle mode di halaman Kelas & Jurusan

- `(admin)/kelas/kelas-jurusan-view.tsx` (halaman existing, kelola data Kelas) — TAMBAH kolom/field **"Mode Jadwal"** (dropdown `Normal`/`Blok`) di form create/edit Kelas — REUSE komponen form yang sudah ada, tambah 1 field baru.
- Tabel daftar Kelas — TAMBAH kolom "Mode Jadwal" (badge Normal/Blok) — KONSISTEN aturan tabel permanen proyek (kolom baru WAJIB sortable, sesuai memory `feedback_tabel_wajib_search_sort_kolom_no` — halaman ini SUDAH termasuk scope T141 sebelumnya, pastikan kolom baru TIDAK jadi pengecualian dari aturan sortable yang sudah diterapkan di sana).
- **Halaman Jadwal Mengajar** (baik versi admin-jurnal existing MAUPUN versi baru T157 kalau sudah dikerjakan) — form create/edit `Schedule` — field `minggu` (dropdown A/B/setiap_minggu) HANYA MUNCUL/AKTIF kalau kelas yang dipilih di form itu `modeJadwal: blok` (disabled/hidden untuk kelas mode Normal, supaya admin tidak bingung mengisi field yang tidak relevan) — VERIFIKASI logic ini konsisten dengan validasi backend poin 2.

## Edge Cases
- Kelas mode `blok` TAPI TIDAK ADA `BlockWeekRange` yang mencakup tanggal tertentu (lubang kalender, kondisi yang SUDAH ditangani `ScheduleResolverService.getMingguAktif()` existing — return null, log warning) — task ini TIDAK mengubah penanganan kasus ini, HANYA mengubah SUMBER keputusan "apakah perlu resolusi minggu sama sekali" (dari semester ke kelas) — behavior lubang kalender TETAP SAMA seperti sekarang.
- Kelas yang SUDAH PUNYA jadwal (`Schedule` dengan `minggu` terisi A/B) LALU mode-nya diubah dari Blok ke Normal — `Schedule.minggu` yang sudah tersimpan TIDAK otomatis dihapus/diubah (data lama tetap ada), TAPI method resolusi (poin 2) SEKARANG mengabaikan field itu untuk kelas mode Normal — EVALUASI apakah ini menyisakan data "mati" yang membingungkan admin kalau dilihat manual (field `minggu` masih ada isinya padahal tidak dipakai lagi) — TIDAK WAJIB dibersihkan otomatis, TAPI PERTIMBANGKAN tampilkan indikator di UI ("field ini tidak berlaku karena kelas mode Normal") kalau terlihat di suatu tampilan.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`Kelas` + field baru), `apps/api/src/schedule-resolver/schedule-resolver.service.ts` (sumber keputusan mode), `apps/api/src/core/schedules/schedules.service.ts` (validasi `minggu` berdasar kelas bukan semester), `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx` (field+kolom baru), halaman form Jadwal Mengajar (conditional field `minggu`).
- **Jangan sentuh:** `BlockWeekRange` (kalender global, TIDAK berubah), `Semester.mode` (TIDAK dihapus, dievaluasi tapi dibiarkan kalau ragu).

## Acceptance Criteria
- [ ] Kelas punya field `modeJadwal` (Normal/Blok), default Normal, bisa diedit lewat halaman Kelas & Jurusan.
- [ ] Resolusi jadwal (`getJadwalUntukTanggal`) memutuskan perlu-tidaknya filter Minggu A/B berdasarkan `Kelas.modeJadwal`, BUKAN lagi `Semester.mode`.
- [ ] 1 semester BISA punya campuran kelas Normal dan Blok sekaligus, masing-masing di-resolve benar sesuai mode kelasnya.
- [ ] Form Jadwal Mengajar menampilkan/menyembunyikan field `minggu` sesuai mode kelas yang dipilih.
- [ ] Kelas mode Normal yang jadwalnya PUNYA nilai `minggu` lama (dari sebelum diubah mode) TIDAK menyebabkan jadwal itu hilang/salah tampil — field itu diabaikan, bukan menyebabkan error.
- [ ] Regresi nol untuk kelas yang TIDAK diubah (default Normal, behavior sama seperti sebelum task ini kalau semester juga mode normal).
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [ ] **BACA ULANG SECARA LENGKAP** `ScheduleResolverService.getJadwalUntukTanggal()` SEBELUM mengubah sumber keputusan mode — pastikan semua cabang logic existing dipahami, tidak ada yang terlewat saat migrasi dari `semester.mode` ke `kelas.modeJadwal`.
- [ ] **JANGAN** hapus `Semester.mode` dari schema — biarkan ada meski mungkin tidak lagi jadi sumber logic utama, laporkan sebagai temuan kalau field itu jadi vestigial.
- [ ] **JANGAN** ubah struktur `BlockWeekRange` jadi per-kelas — dikonfirmasi user TETAP 1 kalender blok global untuk seluruh sekolah.
- [ ] Pastikan perubahan ini KOMPATIBEL dengan T157 (kalau halaman Jadwal Mengajar sudah diduplikasi ke admin) — form conditional field `minggu` perlu diterapkan di KEDUA lokasi (admin-jurnal asli DAN admin baru) kalau keduanya sudah ada saat task ini dikerjakan.
