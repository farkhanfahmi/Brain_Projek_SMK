# T209 — API: Pindahkan TeachingSession dari Schedule Lama ke JadwalSlot Baru

## Depends on
**WAJIB SETELAH T206** (JadwalSlot harus bisa create/dipakai end-to-end). Bagian dari rangkaian T203-T215. **T210-T213 (UI) TIDAK BISA berfungsi penuh sampai task ini selesai** — karena generate `TeachingSession` (dasar Jurnal Guru T168-T173) butuh sumber jadwal yang benar.

## Objective
`TeachingSession.scheduleId` (FK ke `Schedule` LAMA) — GANTI jadi `jadwalSlotId` (FK ke `JadwalSlot` BARU). Semua logic generate `TeachingSession` harian (`TeachingSessionsService.generateForDate()`) dan resolusi terkait — pindah sumber dari `Schedule`+`ScheduleResolverService` LAMA ke `JadwalSlot`+resolusi BARU.

## Konteks Penting — Kenapa Task Ini AMAN Dilakukan Sekarang

Data production `teaching_sessions=0` (dikonfirmasi 2026-08-16, TAPI **WAJIB VERIFIKASI ULANG** — lihat Validasi Claudian) — modul Jurnal Guru (T168-T173, sudah live secara KODE tapi belum dipakai NYATA oleh guru) TIDAK KEHILANGAN DATA APAPUN dari perubahan struktural ini. TAPI kode yang SUDAH ADA (journal.service.ts, teaching-sessions.service.ts, dst) TETAP HARUS diperbarui supaya TIDAK RUSAK begitu ada guru pertama yang benar-benar pakai fitur ini.

## Spec Detail

### 1. Schema — ganti FK

```prisma
// TeachingSession — GANTI:
// scheduleId Int @map("schedule_id")
// GANTI JADI:
jadwalSlotId Int @map("jadwal_slot_id")
```
- Migration — DROP FK lama ke `Schedule`, ADD FK baru ke `JadwalSlot`. **VERIFIKASI ULANG row count `teaching_sessions` di production TEPAT SEBELUM migration** (sama seperti T203) — kalau ternyata SUDAH ADA data nyata, STOP dan laporkan ke user, JANGAN migrate destruktif.
- `@@unique([scheduleId, tanggal])` → `@@unique([jadwalSlotId, tanggal])`.

### 2. Backend — `TeachingSessionsService.generateForDate()` dan method terkait

- **GANTI SUMBER RESOLUSI JADWAL HARIAN**: method ini SAAT INI query `Schedule` (via `ScheduleResolverService.getJadwalUntukTanggal()`, LAMA) untuk tahu "kelas mana yang punya jadwal mengajar tanggal X" — GANTI jadi query `JadwalSlot` dari `OpsiJadwal` yang **AKTIF** untuk tanggal itu:
  1. Cari `Semester` aktif yang mencakup tanggal X.
  2. Cari SEMUA `OpsiJadwal` yang `isActive: true` DAN `semesterId` = semester itu.
  3. Untuk tiap `OpsiJadwal` aktif: kalau `mode: normal` → ambil SEMUA `JadwalSlot` dengan `hari` = hari tanggal X (minggu selalu NULL, cocok apa saja). Kalau `mode: blok` → ambil `JadwalSlot` dengan `hari` = hari tanggal X DAN `minggu` = hasil lookup `OpsiJadwalMingguGenerate` untuk tanggal X (kalau tanggal X TIDAK ADA di `OpsiJadwalMingguGenerate` untuk Opsi ini — SKIP Opsi Jadwal itu untuk tanggal ini, fallback aman KONSISTEN filosofi lama, JANGAN error/block generate untuk Opsi Jadwal lain).
  4. GABUNGKAN hasil dari SEMUA Opsi Jadwal aktif yang cakupan tingkatnya cocok dengan tingkat kelas masing-masing `JadwalSlot` (VERIFIKASI: apakah 1 kelas bisa dapat `JadwalSlot` dari 2 Opsi Jadwal AKTIF berbeda sekaligus di hari yang SAMA — SEHARUSNYA TIDAK, karena T206 sudah validasi "2 Opsi Jadwal aktif bersamaan tidak boleh overlap cakupan tingkat" — TAPI VERIFIKASI ULANG asumsi ini benar-benar ditegakkan sebelum diandalkan di sini).
- **REPLACE PEMANGGILAN** `ScheduleResolverService` di method ini — TIDAK PERLU HAPUS `ScheduleResolverService` (masih dipakai domain `jam_sekolah`/`jadwal_khusus` LAMA, T145, TIDAK TERDAMPAK rangkaian ini) — HANYA method spesifik jam_mengajar yang diganti sumbernya.

### 3. Backend — audit SEMUA consumer lain `TeachingSession.scheduleId`

- GREP MENYELURUH semua pemakaian `TeachingSession.scheduleId`/`schedule` relasi — dari riset SEBELUM task ini ditulis, titik yang PASTI terdampak: `journal.service.ts` (getKelasDiajar, getKalenderBulan, getRiwayatJurnal — SEMUA join lewat TeachingSession→Schedule untuk info kelas/mapel/guru, GANTI ke join TeachingSession→JadwalSlot), `teaching-sessions.service.ts` (getMyToday, getSesiUntukTanggal, startSession, isBerlakuPadaTanggal), `teacher-attendance-report.service.ts` (reportFlexible). **UNTUK SETIAP TITIK**, ganti join dari `schedule.kelas`/`schedule.mapel`/`schedule.teacher` (SINGULAR, 1 guru) jadi `jadwalSlot.kelas`/`jadwalSlot.mapel`/`jadwalSlot.guru[]` (PLURAL, team-teaching — PERHATIAN: tampilan UI yang SEBELUMNYA asumsi "1 guru per sesi" (misal `sesi-card.tsx`, jurnal-view, dll) PERLU DIPERBARUI untuk tampilkan BANYAK guru kalau team-teaching — TANDAI temuan ini, TAPI perbaikan UI detailnya masuk scope **T213** (UI Jurnal Guru terdampak team-teaching), task INI fokus BACKEND saja).

## Edge Cases
- `TeachingSession` yang SUDAH `startedAt`/`closedAt` terisi (guru sudah mulai mengajar) SAAT migrasi ini dijalankan (kalau ternyata ADA data, kontras temuan 0 baris) — TIDAK BOLEH kehilangan histori jurnal/presensi yang sudah terikat — INI SKENARIO YANG SEHARUSNYA TIDAK TERJADI (data 0), TAPI KALAU TERJADI, STOP dan diskusikan strategi migrasi data KHUSUS dengan user, JANGAN jalankan migration ini secara destruktif.
- Team-teaching (banyak guru 1 TeachingSession) — `TeachingSession.teacherId` (SAAT INI single field) — VERIFIKASI apakah field ini PERLU diubah jadi many-to-many JUGA, atau CUKUP tetap 1 guru "penanggung jawab utama" per sesi (misal guru PERTAMA di `JadwalSlotGuru`) sementara guru lain hanya tercatat di level `JadwalSlot` — **KLARIFIKASI KE USER SAAT IMPLEMENTASI**, ini keputusan besar yang BELUM eksplisit dibahas di diskusi (diskusi fokus ke Team Teaching di level definisi jadwal, BELUM soal bagaimana `TeachingSession`/jurnal/presensi individual menangani banyak guru per sesi).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`TeachingSession.jadwalSlotId`), `apps/api/src/teaching-sessions/teaching-sessions.service.ts`, `apps/api/src/journal/journal.service.ts`, `apps/api/src/teacher-attendance-report/teacher-attendance-report.service.ts`.
- **Buat:** migration Prisma baru.
- **Jangan sentuh:** `ScheduleResolverService` (TIDAK dihapus, masih dipakai domain jam_sekolah/jadwal_khusus T145 yang TIDAK terdampak rangkaian ini).

## Acceptance Criteria
- [x] `TeachingSession.jadwalSlotId` berhasil menggantikan `scheduleId`.
- [x] `generateForDate()` menghasilkan `TeachingSession` benar dari `JadwalSlot` + `OpsiJadwal` aktif (mode normal DAN blok, termasuk resolusi minggu A/B dari `OpsiJadwalMingguGenerate`).
- [x] SEMUA consumer lain diperbarui, tidak ada sisa referensi `schedule`/`scheduleId` untuk domain jam_mengajar yang terlewat (grep menyeluruh + tsc --noEmit sebagai bukti definitif) — **daftar lengkap**: `teaching-sessions.service.ts` (generateForDate, getMyToday, getSesiUntukTanggal, startSession, autoCloseDueSessions, flagPermitsNeedingFollowUp, izinSesiSpesifikSudahLewat, izinSeharianSudahLewat), `teaching-sessions.controller.ts` (response field), `teaching-sessions.module.ts` (ganti import module), `journal.service.ts` (getRiwayatJurnal), **`tv-piket.service.ts`** (hitungGuruBelumMulai, hitungGuruIzin — TIDAK disebut di spec awal, ditemukan lewat tsc). `teacher-attendance-report.service.ts` DIKONFIRMASI tidak consume TeachingSession sama sekali (asumsi spec salah, sudah dicek langsung).
- [x] Keputusan team-teaching di level TeachingSession DIKONFIRMASI user dan didokumentasikan.
- [x] Build + type-check hijau (`tsc --noEmit` + `nest build`), jest diperbarui.

## Validasi Claudian
- [x] **WAJIB verifikasi ULANG row count production** `teaching_sessions` tepat sebelum migration — DIKONFIRMASI via `docker exec absensi-mysql-prod mysql ... SELECT COUNT(*)` = 0 (bukan asumsi dari catatan lama; MCP mysql-absensi yang tersedia sesi ini ternyata terhubung ke DEV port 3307, bukan production 3309 — dikoreksi via AskUserQuestion ke user sebelum lanjut).
- [x] **WAJIB klarifikasi user** soal team-teaching di level `TeachingSession.teacherId` — **HASIL: TEPAT 1 guru per JadwalSlot**, `MapelGuru` many-to-many HANYA utk filter dropdown "guru relevan per mapel" saat admin input jadwal, BUKAN team-teaching sungguhan. Ini mengoreksi T206 yang sempat mengizinkan array `teacherIds[]` — diperbaiki sebagai bagian task ini dengan izin eksplisit user (lihat catatan Status Eksekusi).
- [x] Grep menyeluruh SEMUA consumer `scheduleId`/`schedule` relasi TeachingSession — daftar lengkap di atas. Cross-check via `tsc --noEmit` (yang akan gagal compile kalau ada sisa field/relasi lama) menemukan `tv-piket.service.ts` yang tidak disebut di spec awal.

## Status Eksekusi

**Selesai 2026-08-17 02:10**

Koreksi scope T206 (dengan persetujuan eksplisit user, di luar file-list asli task ini): `apps/api/src/jadwal-slot/dto/create-jadwal-slot.dto.ts` (`teacherIds: number[]` → `teacherId: number`), `jadwal-slot.service.ts` (semua method `create`/`update`/`ensureGuruTerdaftarMapel`/`ensureNoBentrok`/`cekKetersediaanGuru` disesuaikan single-teacher), `schema.prisma` (`JadwalSlotGuru.@@unique([jadwalSlotId, teacherId])` → `@@unique([jadwalSlotId])`, DB-level enforce), `jadwal-slot.service.spec.ts` (15 test disesuaikan + 3 test baru khusus verifikasi single-guru).

Migration `20260816185325_t209_teaching_session_jadwal_slot` — additive-safe murni (kedua tabel terdampak 0 baris di dev DAN production), digenerate manual via `prisma migrate diff` (non-interaktif, `migrate dev` butuh TTY) lalu ditulis sebagai file migration dan di-apply via `migrate deploy`.
