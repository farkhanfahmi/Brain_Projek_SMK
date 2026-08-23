# T195 — API+Web: Pisah Field Toleransi Keterlambatan Siswa vs Guru (Bukan 1 Nilai Gabungan)

## Depends on
Tidak ada dependency teknis. **Merevisi sebagian keputusan T185** (sudah selesai — T185 memisahkan Karyawan ke `KaryawanJamKerjaConfig` terpisah, TAPI Siswa dan Guru MASIH 1 field gabungan `ScheduleConfig.toleransiTerlambatMenit`).

## Objective
Pisahkan `ScheduleConfig.toleransiTerlambatMenit` (SAAT INI 1 nilai GLOBAL dipakai BERSAMA Siswa DAN Guru) jadi **2 field independen** — toleransi Siswa dan toleransi Guru bisa diatur BEDA nilai, TIDAK LAGI 1 angka yang sama untuk keduanya.

## Context — Temuan Testing User (2026-08-16)

Setelah T185 dieksekusi (memisahkan Karyawan ke config terpisah `KaryawanJamKerjaConfig`), user testing dan menemukan: **Siswa dan Guru MASIH pakai 1 field yang SAMA** (`ScheduleConfig.toleransiTerlambatMenit`, `schema.prisma:411-420`, komentar eksplisit "global, bukan per-guru/mapel"). `toleransi-view.tsx` (baris 26-59) bahkan comment eksplisit menyatakan nilai ini "SATU-SATUNYA yang dipakai... untuk SISWA... MAUPUN GURU".

User eksplisit: "bedakan toleransi guru dan murid, jangan dijadikan 1 dalam 1 field. guru dan murid punya field waktu sendiri sendiri" — KONSISTEN dengan pola yang SUDAH diterapkan untuk Karyawan di T185 (config terpisah), tinggal direplikasi untuk pasangan Siswa-Guru.

## Spec Detail

### 1. Backend — pecah field jadi 2

```prisma
// GANTI di ScheduleConfig:
// toleransiTerlambatMenit Int  →  HAPUS, GANTI:
toleransiSiswaMenit Int @default(15) @map("toleransi_siswa_menit")
toleransiGuruMenit  Int @default(15) @map("toleransi_guru_menit")
```
- Migration: rename/split kolom lama — data existing (nilai `toleransiTerlambatMenit` yang SUDAH diisi admin) di-**backfill** ke KEDUA kolom baru (sama nilainya di awal, admin bisa ubah beda-beda SETELAHNYA) — supaya toleransi yang SUDAH dikonfigurasi tidak hilang/reset ke default begitu saja.
- `ScheduleConfigService`/`Controller` — endpoint GET/PATCH diperbarui untuk 2 field ini (REKOMENDASI: tetap 1 endpoint `PATCH /schedule-config` menerima KEDUA field sekaligus dalam 1 payload, KONSISTEN pola `KaryawanJamKerjaConfig` T185 kalau applicable — VERIFIKASI pola T185 dan REPLIKASI strukturnya).
- **Titik pemakaian yang WAJIB diperbarui** — SEMUA tempat yang baca `toleransiTerlambatMenit` (SAAT INI 2 titik dikonfirmasi T185: papan TV Piket "guru belum mulai mengajar", hitung MENIT-terlambat guru di dashboard jurnal — GREP MENYELURUH untuk pastikan tidak ada titik lain yang terlewat) — SESUAIKAN masing-masing pakai `toleransiSiswaMenit` ATAU `toleransiGuruMenit` sesuai KONTEKS-nya (siswa vs guru), BUKAN salah satu dipakai untuk keduanya.
- `determineStatus()` (`attendance.service.ts`, sudah diaktifkan T185 untuk benar-benar dipakai perhitungan, bukan cuma display) — cabang SISWA pakai `toleransiSiswaMenit`, cabang GURU (T175/T185) pakai `toleransiGuruMenit`.

### 2. Frontend — 2 field terpisah di UI

- `toleransi-view.tsx` — GANTI 1 input jadi 2 card/section terpisah: "Toleransi Keterlambatan Siswa" dan "Toleransi Keterlambatan Guru" (SEJAJAR visual dengan card "Toleransi Keterlambatan Karyawan" yang SUDAH ADA dari T185 — supaya 3 card konsisten: Siswa, Guru, Karyawan, masing-masing independen).
- Duplikat `(admin)/toleransi-keterlambatan/` — REUSE komponen yang sama, otomatis ikut berubah.

## Edge Cases
- Data lama (`toleransiTerlambatMenit` SATU nilai yang sudah dikonfigurasi admin sebelumnya) — WAJIB di-backfill ke KEDUA field baru dengan nilai SAMA saat migration (BUKAN reset ke default 15 menit begitu saja, supaya konfigurasi existing tidak hilang tanpa sepengetahuan admin).
- Admin ubah toleransi Guru TAPI TIDAK ubah toleransi Siswa (atau sebaliknya) — SETELAH task ini, keduanya independen, TIDAK saling mempengaruhi.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`ScheduleConfig`, split field), `apps/api/src/schedule-config/` (service+controller+DTO), `apps/api/src/attendance/attendance.service.ts` (`determineStatus()`, pakai field sesuai konteks), titik lain yang baca toleransi (TV Piket, dashboard jurnal — grep menyeluruh), `apps/web/.../toleransi/toleransi-view.tsx` (2 card terpisah Siswa+Guru).
- **Buat:** migration Prisma dengan backfill data lama ke kedua kolom baru.

## Acceptance Criteria
- [x] `ScheduleConfig` punya field terpisah: `toleransiSiswaMenit`, `toleransiGuruMenit` — DIPERLUAS jadi 3 (+`toleransiKaryawanMenit`, dikonfirmasi user setelah riset menemukan cabang karyawan juga pakai field gabungan lama).
- [x] Admin bisa set toleransi Siswa BEDA dari Guru (dan Karyawan) — verified via 4 test discriminating (nilai jauh berbeda per aktor: 5/20/30 menit, kalau ada cabang salah pakai field lain pasti gagal).
- [x] Data lama ter-backfill ke SEMUA field baru saat migration, TIDAK reset ke default — verified query sebelum (`toleransi_terlambat_menit=10`) dan sesudah (`toleransi_siswa_menit=toleransi_guru_menit=toleransi_karyawan_menit=10`).
- [x] `determineStatus()` — cabang siswa pakai `toleransiSiswaMenit`, cabang guru pakai `toleransiGuruMenit`, cabang karyawan pakai `toleransiKaryawanMenit` — verified 4 test baru.
- [x] UI toleransi tampilkan 3 card independen (Siswa, Guru, Karyawan — card Karyawan digabung dari 2 endpoint: jam kerja `KaryawanJamKerjaConfig` T185 + menit toleransi `ScheduleConfig` baru, supaya tetap "1 card = 1 aktor").
- [x] Build + type-check hijau, 4 jest baru untuk skenario toleransi berbeda per aktor (bukan cuma siswa vs guru, tapi 3 arah).

## Validasi Claudian
- [x] **WAJIB verifikasi ULANG semua titik pemakaian** `toleransiTerlambatMenit` lama — grep menyeluruh menemukan 6 titik (bukan cuma 2 dari T185): `attendance.service.ts` (3 cabang determineStatus), `schedule-config.service.ts` (get/update/hitungTerlambatMenit), `tv-piket.service.ts` (hitungGuruBelumMulai), `toleransi-view.tsx` frontend. Semua diperbarui.
- [x] Konfirmasi migration BENAR-BENAR backfill nilai lama — migration DITULIS TANGAN (bukan `prisma migrate dev` interaktif, environment non-interactive tidak didukung), 3 kolom baru ditambah NULLABLE dulu → UPDATE backfill dari kolom lama → NOT NULL+default dikunci setelah backfill → kolom lama di-drop. Diverifikasi via `SELECT` sebelum dan sesudah (10 → 10/10/10), `prisma migrate status` konfirmasi "up to date" tanpa drift.

## Catatan Implementasi (2026-08-16)

- **Scope diperluas dari spec tertulis**: task asli cuma sebut Siswa vs Guru, tapi riset kode menemukan cabang KARYAWAN di `determineStatus()` (T185) juga pakai `toleransiTerlambatMenit` yang sama (bukan `KaryawanJamKerjaConfig` yang cuma punya jam+toggle, tanpa field menit). Dikonfirmasi user via pertanyaan pilihan: tambah field ke-3 `toleransiKaryawanMenit`, konsisten pola "3 aktor independen" yang dimulai T185.
- **Migration ditulis tangan**: `prisma migrate dev --create-only` gagal karena environment non-interactive (warning drop kolom dengan 1 row data). Migration SQL ditulis manual: ADD COLUMN 3 kolom baru NULLABLE → UPDATE backfill dari kolom lama → MODIFY NOT NULL+DEFAULT 10 → DROP COLUMN lama. Dieksekusi via `docker exec absensi-mysql-1 mysql ...` (dikonfirmasi container dev via `docker port`, BUKAN `absensi-mysql-prod`), lalu `prisma migrate resolve --applied` untuk sinkronkan metadata `_prisma_migrations`.
- **`hitungTerlambatMenit()`** (`ScheduleConfigService`) — HANYA dipanggil dari `TeachingSessionsService.startSession()` (konteks guru mulai sesi mengajar), jadi di-hardcode pakai `toleransiGuruMenit` (bukan parameter pilih aktor — method ini secara struktural cuma untuk guru).
- **UI Karyawan card**: gabung 2 API call (`PATCH /karyawan-jam-kerja-config` untuk jam+toggle, `PATCH /schedule-config` untuk menit toleransi) dalam 1 `handleSimpan()` via `Promise.all` — dipilih user daripada pecah jadi card ke-4 terpisah, supaya UX tetap 1 card per aktor meski datanya dari 2 sumber.
- **Test**: 4 test baru di `attendance.service.spec.ts` (describe "T195 toleransi independen per aktor") sengaja set nilai JAUH BERBEDA per aktor (Siswa 5 menit, Guru 20 menit, Karyawan 30 menit) — supaya kalau ada cabang yang salah pakai field aktor lain, test PASTI gagal (bukan cuma reuse angka sama seperti test T185 lama yang tidak bisa membedakan bug ini). `schedule-config.service.spec.ts` ditulis ulang total untuk 3 field + verifikasi independensi (ubah Guru tidak ikut ubah Siswa/Karyawan).
- **Verifikasi**: `tsc --noEmit` bersih 2 app, `nest build`+`next build` sukses, 481 test backend lulus (naik dari 471 sebelum T195). Endpoint `/schedule-config` di-smoke-test via curl (401 tanpa auth, route ter-mapping benar di log nest watch, tidak ada compile error setelah edit).
