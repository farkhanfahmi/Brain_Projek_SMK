# T215 — Schema+API+Web: Hapus Model & UI Lama (Schedule jam_mengajar, BlockWeekRange, JamPelajaranOption dkk, Kelas.modeJadwal, Semester.mode)

## Depends on
**WAJIB PALING TERAKHIR — SETELAH T203-T214 SEMUA SELESAI DAN TERVERIFIKASI LIVE oleh user** (bukan cuma test otomatis pass — user sudah COBA LANGSUNG alur penuh: buat Opsi Jadwal, isi Alokasi Waktu, input jadwal mode Normal DAN Blok, generate TeachingSession, isi Jurnal, dst — dan MENGONFIRMASI EKSPLISIT semuanya bekerja). **JANGAN EKSEKUSI TASK INI TANPA KONFIRMASI EKSPLISIT USER** bahwa sistem baru sudah siap menggantikan yang lama sepenuhnya — ini task DESTRUKTIF (hapus kode+data lama), beda dari semua task T203-T214 sebelumnya yang murni ADDITIF.

## Objective
Hapus TOTAL: model Prisma lama (`Kelas.modeJadwal`, `Semester.mode`, `BlockWeekRange`, `JamPelajaranOption`+`JamPelajaranOptionTingkat`+`JamPelajaranAktivasi`+`JamPelajaranSlot`, `Schedule` KHUSUS type `jam_mengajar` — type `jam_sekolah`/`jadwal_khusus` TETAP ADA), service/controller lama terkait, dan halaman UI lama (`(admin)/jadwal-mengajar/`, `(admin-jurnal)/admin-jurnal/jadwal-blok/`, `(admin)/jam-pelajaran/` versi lama Sheet, dkk) — SETELAH arsitektur baru (T203-T214) terbukti berfungsi penuh.

## ⚠️ PERINGATAN KERAS — Baca Sebelum Mulai

1. **INI TASK DESTRUKTIF** — SEMUA task T203-T214 sebelumnya SENGAJA ditulis ADDITIF (tidak menyentuh/menghapus apa pun yang lama) SUPAYA sistem lama dan baru bisa jalan PARALEL selama masa transisi. Task ini yang MENGAKHIRI masa transisi itu.
2. **VERIFIKASI ULANG data production SEKALI LAGI** sebelum drop kolom/tabel — walau diasumsikan `Schedule` type jam_mengajar sudah tidak dipakai lagi (semua sudah pindah ke `JadwalSlot` di T209), CEK LANGSUNG row count TERKINI sebelum eksekusi, JANGAN percaya asumsi dari task-task sebelumnya begitu saja (kondisi bisa berubah kalau ada aktivitas lain di antara waktu penulisan dan eksekusi task ini).
3. **BACKUP WAJIB** sebelum migration destruktif ini — KONSISTEN protokol proyek (memory insiden database wipe 2026-07-30) — backup manual + verifikasi row count SEBELUM dan SESUDAH.

## Spec Detail

### 1. Schema — DROP model/field lama

```prisma
// HAPUS dari Kelas: modeJadwal
// HAPUS dari Semester: mode
// HAPUS TOTAL model: BlockWeekRange, JamPelajaranOption, JamPelajaranOptionTingkat,
//                     JamPelajaranAktivasi, JamPelajaranSlot
// HAPUS dari Schedule: field jamKeAwal/jamKeAkhir/minggu KHUSUS relevan jam_mengajar
//   (VERIFIKASI: apakah Schedule type jam_mengajar SELURUHNYA dihapus barisnya dari DB,
//   atau field-field itu dibiarkan NULL selamanya karena type jam_sekolah/jadwal_khusus
//   TIDAK PERNAH mengisinya — PUTUSKAN mana yang lebih aman/bersih saat implementasi,
//   REKOMENDASI: DELETE FROM schedules WHERE type='jam_mengajar' dulu (baris LAMA, sudah
//   tidak dipakai/redundan dengan JadwalSlot), BARU pertimbangkan drop kolom jamKeAwal/
//   jamKeAkhir/minggu KALAU memang sudah tidak relevan sama sekali untuk 2 type lain)
```

### 2. Backend — hapus module lama

- Hapus TOTAL: `apps/api/src/jam-pelajaran/` (JamPelajaranService lama, DIGANTIKAN `AlokasiWaktuService`+`OpsiJadwalService` T206), `apps/api/src/block-week-ranges/` (DIGANTIKAN Date Generator T208).
- `SchedulesService`/`SchedulesController` — HAPUS method/endpoint KHUSUS terkait `jam_mengajar` (`resolveMinggu()` versi lama, `copyFromSemester()` KALAU cuma untuk jam_mengajar — VERIFIKASI) — method untuk `jam_sekolah`/`jadwal_khusus` (jam masuk sekolah 3-lapis, T145) **TETAP ADA, TIDAK DIHAPUS**.
- `ScheduleResolverService` — HAPUS method KHUSUS jam_mengajar (`getJadwalHariIni`, `getJadwalUntukTanggal`, `filterByKelasModeJadwal` — SEMUA DIGANTIKAN logic T209) — method `getMingguAktif`/`getMingguAktifSaatIni` (KALAU masih relevan untuk domain lain, VERIFIKASI) boleh tetap ada atau dihapus tergantung masih dipakai atau tidak.
- `ImportService.importSchedules()` LAMA (T160) — HAPUS, DIGANTIKAN `T213`.

### 3. Frontend — hapus halaman lama

- Hapus TOTAL: `(admin)/jam-pelajaran/` (Sheet-based lama), `(admin)/jadwal-mengajar/`, `(admin-jurnal)/admin-jurnal/jadwal-mengajar/`, `(admin-jurnal)/admin-jurnal/jadwal-blok/`, `(admin-jurnal)/admin-jurnal/jadwal/` (SEMUA sudah digantikan `(admin)/jadwal-pelajaran/`+`(admin)/alokasi-waktu/` dari T210-T212).
- **REDIRECT** path lama ke path baru (KONSISTEN pola T189/T190 — `redirect()` server-side, BUKAN hard 404) — supaya bookmark lama admin tidak mati mendadak.
- `kelas-jurusan-view.tsx` — HAPUS kolom/badge "Mode Jadwal" (field `Kelas.modeJadwal` sudah tidak ada).
- `semesters-section.tsx` — HAPUS field/badge "Mode" (field `Semester.mode` sudah tidak ada).

### 4. Backfill data lama KE arsitektur baru (KALAU relevan)

- **VERIFIKASI SAAT EKSEKUSI**: apakah SELAMA masa transisi (T203-T214 dikerjakan), ADA admin yang SUDAH input data pakai sistem LAMA (Schedule/BlockWeekRange/JamPelajaranOption) yang PERLU dipindahkan manual ke sistem BARU SEBELUM dihapus? KALAU ADA — INI HARUS DISELESAIKAN (data dipindah manual ATAU re-input via sistem baru) SEBELUM task ini dieksekusi, BUKAN diasumsikan otomatis oleh task ini (task ini MURNI PEMBERSIHAN, TIDAK ADA logic migrasi data otomatis di dalamnya).

## Edge Cases
- Kalau VERIFIKASI ULANG (poin Peringatan #2) menemukan Schedule jam_mengajar/BlockWeekRange/JamPelajaranOption MASIH ADA data terpakai NYATA (bukan 0) — **STOP TOTAL, JANGAN LANJUTKAN task ini**, laporkan ke user, diskusikan strategi migrasi data KHUSUS sebelum lanjut.

## Files
- **Hapus:** SEMUA file yang disebut poin 2-3 di atas.
- **Modifikasi:** `apps/api/prisma/schema.prisma` (drop field/model), `kelas-jurusan-view.tsx`, `semesters-section.tsx`.
- **Buat:** migration Prisma DESTRUKTIF (drop tables/columns) — WAJIB backup manual sebelum dijalankan ke production.

## Acceptance Criteria
- [ ] SEMUA model/field lama terhapus dari schema, migration berhasil TANPA kehilangan data yang MASIH RELEVAN (verified row count sebelum/sesudah).
- [ ] SEMUA halaman/endpoint lama terhapus atau redirect, tidak ada 404 mendadak untuk bookmark lama.
- [ ] Sistem BARU (T203-T214) berfungsi 100% TANPA bergantung sama sekali ke kode/data lama yang dihapus di sini — verified END-TO-END sekali lagi SETELAH pembersihan (regresi nol).
- [ ] Build + type-check hijau, SEMUA jest terkait modul lama dihapus/diganti jest modul baru, full suite pass.

## Validasi Claudian
- [ ] **WAJIB konfirmasi eksplisit dari user** SEBELUM mulai task ini — JANGAN eksekusi otomatis hanya karena T203-T214 tercatat "Selesai" di STATUS.md, user HARUS benar-benar sudah pakai sistem baru dan puas.
- [ ] **WAJIB backup manual + verifikasi row count** sebelum migration destruktif dijalankan — KONSISTEN protokol keamanan proyek pasca-insiden database wipe.
- [ ] Verifikasi ULANG (bukan asumsi dari task-task sebelumnya) tidak ada data nyata yang hilang dari penghapusan ini.
