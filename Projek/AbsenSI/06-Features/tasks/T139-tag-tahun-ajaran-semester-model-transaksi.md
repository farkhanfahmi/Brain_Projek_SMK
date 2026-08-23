# T139 — Schema+API: Tag `academicYearId`+`semesterId` Eksplisit di Semua Model Transaksi (Fondasi)

## Depends on
Tidak ada dependency teknis ke task lain. **Task ini adalah FONDASI untuk T140** (T140 baru bisa dikerjakan setelah T139 selesai — filter rekap by tahun ajaran/semester butuh kolom yang ditambahkan task ini). JANGAN kerjakan T140 sebelum T139 selesai dan diverifikasi.

## Objective
Setiap baris baru di 15 model transaksi (daftar lengkap di bawah) menyimpan `academicYearId`+`semesterId` SECARA EKSPLISIT saat INSERT, diambil dari `AcademicYear`/`Semester` yang `isActive: true` PADA SAAT baris itu dibuat — bukan diinfer belakangan dari perbandingan tanggal seperti sekarang (`resolveDateRange()`). Nilai ini permanen menempel ke baris itu selamanya, tidak berubah lagi walau nanti tahun/semester aktif berganti.

## Context — Kenapa Task Ini Ada (Latar Belakang Diskusi 2026-08-08)

User mengajukan usulan arsitektur "1 tahun ajaran + 1 semester aktif sebagai patokan global, semua menu ikut" untuk memudahkan pengelolaan data jangka panjang. Setelah didiskusikan, disepakati:

1. **Konsep "aktif" yang SUDAH ADA (`AcademicYear.isActive`/`Semester.isActive`) TETAP dipakai HANYA untuk operasional live** (kiosk tap, papan piket, hari-wajib/alfa) — TIDAK diubah maknanya, TIDAK menjadi "time machine" yang bisa mem-browse mundur seluruh aplikasi. Ini keputusan FINAL, jangan diinterpretasikan ulang saat implementasi.
2. **Task ini murni soal PENANDAAN DATA** (data tagging) — memastikan setiap kejadian yang direkam tahu persis "ini terjadi di tahun ajaran/semester apa", disimpan permanen di baris itu sendiri. Ini FONDASI untuk fitur filter/browsing history (T140) dan untuk query performa (index langsung, bukan date-range scan).
3. **Riset dikonfirmasi (2026-08-08, baca kode langsung)**: `kenaikanMassal()` (T073, `apps/api/src/core/kelas/kelas.service.ts`) **BELUM PERNAH dijalankan di production** (dikonfirmasi user) — jadi task ini murni PENCEGAHAN KE DEPAN, TIDAK PERLU backfill/perbaikan data lama yang sudah salah (karena belum ada yang salah). Kolom baru cukup NULLABLE untuk data lama, TIDAK WAJIB diisi retroaktif.
4. **`AttendanceRecord` snapshot `kelasId` per-kejadian** (isu terpisah, TERKAIT tapi BUKAN scope task ini) — TIDAK termasuk di task ini. Kalau nanti dibutuhkan (misal `kenaikanMassal` akhirnya dipakai), itu task terpisah lagi, JANGAN digabung di sini.

## 15 Model Target (SEMUA WAJIB, riset lengkap 2026-08-08)

Format: Model | tabel (`@@map`) | field tanggal kejadian utama | file:baris CREATE | method CREATE

| # | Model | Tabel | Field tanggal kejadian | File create | Method | Baris |
|---|---|---|---|---|---|---|
| 1 | `AttendanceRecord` | `attendance_records` | `tanggal` | `attendance/attendance.service.ts` | `tap()` | 164 |
| 1b | `AttendanceRecord` (jalur ke-2) | `attendance_records` | `tanggal` | `permits/permits.service.ts` | `createTidakMasuk()` (tx) | 191 |
| 1c | `AttendanceRecord` (jalur ke-3) | `attendance_records` | `tanggal` | `permits/permits.service.ts` | `tandaiIzinTidakKembali()` (tx) | 351 |
| 2 | `Permit` | `permits` | `tanggal` | `permits/permits.service.ts` | `createTidakMasuk()` (tx) | 179 |
| 2b | `Permit` (jalur ke-2) | `permits` | `tanggal` | `permits/permits.service.ts` | `createKeluar()` | 214 |
| 2c | `Permit` (jalur ke-3) | `permits` | `tanggal` | `permits/permits.service.ts` | `tandaiIzinTidakKembali()` (tx) | 425 |
| 3 | `EkstraAbsen` | `ekstra_absen` | (via `sesi.tanggal`) | `ekstra-absensi/ekstra-absensi.service.ts` | `createSesi()` (tx, createMany) | 171 |
| 3b | `EkstraAbsen` (jalur ke-2) | `ekstra_absen` | (via `sesi.tanggal`) | `ekstra-absensi/ekstra-absensi.service.ts` | `generateSesiIdempotent()` (createMany) | 270 |
| 3c | `EkstraAbsen` (jalur ke-3) | `ekstra_absen` | (via `sesi.tanggal`) | `ekstra-absensi/ekstra-absensi.service.ts` | `updateSesi()` (createMany) | 553 |
| 4 | `EkstraSesi` | `ekstra_sesi` | `tanggal` | `ekstra-absensi/ekstra-absensi.service.ts` | `createSesi()` (tx) | 148 |
| 4b | `EkstraSesi` (jalur ke-2) | `ekstra_sesi` | `tanggal` | `ekstra-absensi/ekstra-absensi.service.ts` | `generateSesiIdempotent()` | 248 |
| 5 | `PiketJournalEntry` | `piket_journal_entries` | `tanggal` | `piket-journal/piket-journal.service.ts` | `submit()` (upsert) | 105 |
| 5b | `PiketJournalEntry` (jalur ke-2) | `piket_journal_entries` | `tanggal` | `piket-journal/piket-journal.service.ts` | `submit()` (create, isi utang) | 120 |
| 6 | `TapEvent` | `tap_events` | `scannedAt` | `attendance/attendance.service.ts` | `logTapEvent()` (privat) | 658 |
| 7 | `TeacherPermit` | `teacher_permits` | `tanggal` | `teacher-permits/teacher-permits.service.ts` | `create()` | 68 |
| 8 | `ActivityLog` | `activity_log` | `createdAt` (audit=kejadian) | `activity-log/activity-log.service.ts` | `record()` | 51 |
| 9 | `StudentPkl` | `student_pkl` | `tanggalMulai` | `core/students/students.service.ts` | `pklMulaiForStudentIds()` (privat) | 237 |
| 10 | `TeachingSession` | `teaching_sessions` | `tanggal` | `teaching-sessions/teaching-sessions.service.ts` | `generateForDate()` (upsert) | 103 |
| 11 | `JournalEntry` | `journal_entries` | (via `session.tanggal`) | `journal/journal.service.ts` | `updateJournal()` (upsert) | 123 |
| 12 | `ClassAttendanceMark` | `class_attendance_marks` | (via `session.tanggal`) | `journal/journal.service.ts` | `updateAttendance()` (upsert) | 190 |
| 13 | `LateEntrySlip` | `late_entry_slips` | `tanggal` | `late-entry-slips/late-entry-slips.service.ts` | (create) | ~46 |

**TIDAK termasuk** (dikonfirmasi riset, sudah punya FK atau bukan model kejadian personal):
- `SchoolHoliday` — sudah punya `academicYearId` (nullable).
- `BlockWeekRange` — sudah punya `semesterId` (wajib).
- `Schedule`, `Semester` — sudah punya FK terkait.
- `EkstraKelompokAnggota`, `EkstraPendaftaran`, `Card`, `PiketSchedule` — bukan kejadian berulang per-tanggal (state/master data), TIDAK di-tag.

## Spec Detail

### 1. Schema (Prisma) — migration TUNGGAL menambah kolom ke 13 model di atas
Untuk SETIAP model di daftar 13 (bukan 15 baris tabel — beberapa model punya banyak titik CREATE tapi cuma 1 kolom ditambah per model):

```prisma
academicYearId Int?
semesterId     Int?

academicYear AcademicYear? @relation(fields: [academicYearId], references: [id])
semester     Semester?     @relation(fields: [semesterId], references: [id])

@@index([academicYearId])
@@index([semesterId])
```

- **NULLABLE** (bukan wajib) — data lama (sebelum migration ini jalan) TIDAK diisi retroaktif, TETAP `null`. Ini SENGAJA, bukan kurang lengkap — dikonfirmasi user tidak perlu backfill karena `kenaikanMassal` belum pernah dipakai jadi tidak ada data yang "rusak" untuk diperbaiki, dan risiko salah tebak tahun ajaran data lama (terutama sebelum sistem AcademicYear/Semester ada) lebih besar dari manfaatnya.
- Nama relasi Prisma: pastikan tidak bentrok dengan relasi existing di `AcademicYear`/`Semester` (Prisma butuh nama relasi unik kalau ada multiple relasi ke model yang sama dari 1 model — cek dengan `pnpm --filter @absensi/api exec prisma validate` sebelum migrate).

### 2. Helper terpusat — SATU sumber kebenaran untuk resolve "tahun ajaran/semester aktif saat ini"
Buat 1 method/service kecil BARU, contoh `AcademicPeriodService.getActive()` di modul yang sudah relevan (`apps/api/src/academic-years/` atau `apps/api/src/semesters/` — cek modul mana yang sudah ada, JANGAN bikin modul baru kalau bisa nempel ke yang ada) — return `{ academicYearId: number | null, semesterId: number | null }` hasil query `AcademicYear.findFirst({where:{isActive:true}})` + `Semester.findFirst({where:{isActive:true}})`.

- **WAJIB dipakai oleh SEMUA 13 titik CREATE di atas** — JANGAN duplikasi query `isActive` di 13 tempat berbeda (risiko tidak sinkron kalau logic "aktif" berubah nanti).
- **Kalau tidak ada AcademicYear/Semester yang `isActive: true` saat insert terjadi** (edge case: admin lupa set aktif, atau baru instalasi) — set `academicYearId`/`semesterId` tetap `null`, JANGAN throw error/gagalkan transaksi utama (misal tap kartu siswa TIDAK BOLEH gagal cuma karena tidak ada tahun ajaran aktif — ini data operasional kritis, prioritaskan availability). Log warning via `ActivityLog` atau `console.warn` cukup (putuskan saat implementasi, tidak kritis).

### 3. Modifikasi 13 titik CREATE (path lengkap sudah di tabel atas)
Setiap titik create/upsert/createMany di 13 lokasi itu — tambahkan pemanggilan helper `getActive()` SEBELUM insert, sisipkan hasilnya (`academicYearId`, `semesterId`) ke data yang di-insert.

- Untuk titik yang ada di dalam **transaksi Prisma** (`tx.xxx.create(...)`, ditandai "(tx)" di tabel) — panggil `getActive()` SEBELUM `$transaction` dimulai (bukan di dalam callback transaksi, supaya tidak nested-query yang tidak perlu) DAN gunakan connection yang sama kalau helper butuh query DB (atau terima sedikit inefisiensi 1 query ekstra di luar transaksi — ini bukan data yang berubah-ubah cepat, staleness beberapa milidetik tidak masalah).
- Untuk `createMany` (EkstraAbsen, 3 titik) — SEMUA row dalam 1 batch createMany pakai NILAI YANG SAMA (`academicYearId`/`semesterId` hasil 1x panggilan `getActive()`, bukan per-row).
- `JournalEntry`/`ClassAttendanceMark` — field tanggal kejadian TIDAK LANGSUNG ada di model ini (didapat via `session.tanggal`) — TAPI tetap tag `academicYearId`/`semesterId` pakai `getActive()` yang SAMA (bukan coba infer dari `session`, karena `TeachingSession` sendiri juga di-tag task ini, jadi konsisten).
- `ActivityLog` — karena ini audit table generic (banyak pemanggil), pastikan `record()` (baris 51) menerima/menghitung `academicYearId`/`semesterId` di SATU titik itu saja (bukan di setiap pemanggil `record()` di luar), supaya semua caller otomatis dapat tanpa perlu diubah satu-satu.

### 4. TIDAK ada perubahan ke endpoint/DTO publik
Task ini murni internal (kolom baru + logic insert) — TIDAK ada field baru yang perlu dikirim dari frontend/kiosk, TIDAK ada perubahan response API existing. Filter/query pakai kolom baru ini adalah scope T140, BUKAN task ini.

## Edge Cases
- Insert yang terjadi TEPAT saat admin sedang mengganti `isActive` (race condition kecil, misal admin ganti semester aktif di detik yang sama ada tap kartu masuk) — terima hasil apa adanya (salah satu semester, tidak masalah dibulatkan ke salah satunya), TIDAK PERLU locking khusus untuk kasus ini (dampak minor, cuma soal tag data bukan soal keamanan/uang).
- Kalau ada LEBIH DARI 1 `AcademicYear`/`Semester` dengan `isActive: true` secara bersamaan (harusnya tidak mungkin kalau ada constraint, tapi VERIFIKASI dulu apakah ada `@@unique`/validasi service yang menjamin cuma 1 aktif — kalau TIDAK ada constraint itu, `getActive()` pakai `findFirst()` yang ambil salah satu secara deterministik, TIDAK throw error, tapi LAPORKAN temuan ini ke user sebagai potential bug terpisah kalau memang tidak ada constraint-nya).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (13 model + relasi baru), migration baru.
- **Buat:** helper `getActive()` (di modul academic-years/semesters existing).
- **Modifikasi:** 13 file service pada tabel titik-CREATE di atas (path lengkap tersedia di tabel).
- **Jangan sentuh:** endpoint/DTO publik, frontend (`apps/web`, `apps/kiosk`) — task ini backend-only, murni penanda data.

## Acceptance Criteria
- [x] Migration jalan bersih, 13 model dapat kolom `academicYearId`+`semesterId` nullable+index.
- [x] Helper `getActive()` dibuat, dipakai KONSISTEN oleh SEMUA 13 titik create (bukan duplikasi logic).
- [x] Data BARU yang di-insert setelah task ini (tap kartu, izin, ekstra, jurnal piket, dst) otomatis punya `academicYearId`+`semesterId` terisi sesuai yang `isActive` saat itu (test manual: cek row baru di DB setelah tap kartu/submit izin).
- [x] Data LAMA (sebelum migration) tetap `null`, TIDAK ada backfill dijalankan (sesuai keputusan, verifikasi row lama tidak berubah).
- [x] Insert tetap SUKSES walau tidak ada AcademicYear/Semester aktif (tidak throw error, `null` tersimpan) — test dengan skenario semua `isActive=false` sementara.
- [x] `ActivityLog.record()` otomatis tag tanpa perlu ubah pemanggil-pemanggilnya satu-satu.
- [x] Build + type-check `apps/api` hijau. Test suite existing tetap lulus 100% (regresi nol — task ini TIDAK mengubah behavior apa pun yang terlihat user, murni tambahan data).

## Validasi Claudian
- [x] **JANGAN** membuat field ini WAJIB (non-nullable) — breaking change ke data lama tidak diinginkan, sudah diputuskan nullable.
- [x] **JANGAN** membuat task ini juga mengerjakan filter UI/rekap (itu T140, dependency terpisah, jangan digabung walau terasa "sekalian saja").
- [x] **JANGAN** menyentuh `kelasId` snapshot di `AttendanceRecord` — itu isu terpisah yang SENGAJA di luar scope (baca Context di atas), kalau tergoda "sekalian benerin", TAHAN, itu keputusan sadar bukan kelupaan.
- [x] Verifikasi ke user KALAU ternyata ditemukan lebih dari 1 `isActive:true` bersamaan tanpa constraint DB — laporkan sebagai temuan terpisah, jangan diam-diam "difix" sebagai bagian task ini (di luar scope, keputusan fix constraint itu milik user).
- [x] Pastikan nama relasi Prisma baru tidak bentrok dengan relasi existing (`prisma validate` sebelum migrate) — beberapa model (Schedule, dll) sudah punya relasi lain ke AcademicYear/Semester.

## Status Eksekusi (2026-08-08)

**Selesai, diverifikasi live.**

### Schema & Migration
- Migration `20260807210830_t139_tag_academic_period_transaksi` — 13 tabel dapat `academic_year_id`+`semester_id` (nullable, index, FK `ON DELETE SET NULL`), murni additive. `prisma validate` bersih — tidak ada bentrok nama relasi.
- Ditemukan 1 titik CREATE tambahan di luar 15 baris tabel spec: `PermitsService.setPulang()` (`attendanceRecord.create` untuk sub-alur "Dianggap Pulang") — ikut ditag karena genuine insert point untuk model target, konsisten niat task (bukan scope creep, model sudah masuk daftar 13).

### Helper Terpusat
- `AcademicPeriodService.getActive()` (modul baru `academic-period/`, bukan nempel ke `semesters`/`calendar` existing karena keduanya tidak sama-sama owner `AcademicYear`+`Semester` — modul kecil standalone di-import ke 8 modul consumer: attendance, permits, ekstra-absensi, piket-journal, teacher-permits, teaching-sessions, journal, late-entry-slips, activity-log, core (students)).
- Query paralel `Promise.all` ke `AcademicYear.findFirst({isActive:true})` + `Semester.findFirst({isActive:true})`, tidak pernah throw — `null` kalau tidak ada yang aktif, `Logger.warn` saja.
- Konfirmasi constraint "1 aktif" (spec item validasi): `AcademicYearsService.activate()` dan `SemestersService.activate()` SAMA-SAMA pakai pola `$transaction(updateMany false -> update true)` di service layer, konsisten pola `ScheduleConfig`/`AttendanceLockConfig` — TIDAK ditemukan celah untuk 2 aktif bersamaan lewat jalur normal, tidak perlu dilaporkan sebagai bug terpisah.

### 13 Titik CREATE
Semua dipanggil `getActive()` SEKALI sebelum `$transaction`/loop (bukan di dalam callback), hasil di-thread eksplisit ke setiap titik insert dalam 1 request/operasi — termasuk `AttendanceService.tap()` yang punya 6 titik `logTapEvent()` berbeda (semua pakai `period` yang sama, bukan re-fetch per cabang) dan `createMany` (EkstraAbsen) yang pakai nilai sama untuk semua baris batch.

### Verifikasi Live (dev)
- **Tanpa AcademicYear/Semester aktif** (state asli dev DB, kosong total): tap kartu via kiosk siswa → `result: accepted`, `AttendanceRecord`+`TapEvent` tersimpan dengan `academic_year_id`/`semester_id` = `NULL`, TIDAK ada error.
- **Dengan academic year+semester aktif** (dibuat sementara untuk tes, dihapus setelahnya): tap kartu accepted → tag terisi benar (`academicYearId=3, semesterId=1`); tap ditolak (`rejected_locked`) → `TapEvent` (insert-only) tetap tertag benar meski `AttendanceRecord` tidak dibuat.
- `ActivityLog.record()` — trigger via `PATCH /ekstra-registration-config` → baris baru otomatis ter-tag tanpa perlu ubah caller sama sekali.
- Data uji (2 `AttendanceRecord`, 2 `TapEvent`, 1 `AcademicYear`, 1 `Semester` sementara) dibersihkan, kiosk `allowed_ip` direstore ke `10.10.10.103`.

### Test Suite
- `tsc --noEmit` bersih.
- Jest: 5 spec file existing (`teaching-sessions`, `journal`, `teacher-permits`, `piket-journal`, `late-entry-slips`) perlu update instantiation constructor (`academicPeriodStub` ditambah) + 1 assertion `piket-journal.service.spec.ts` disesuaikan expect data baru. Total 203 test lulus 100%, regresi nol.
