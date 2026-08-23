# T112 — API+Web: Tambah Kejadian Lock/Unlock ke "Riwayat Catatan" Profil Siswa

## Depends on
Tidak ada dependency teknis. Modifikasi `riwayatCatatan()` yang sudah ada + perluas `ActivityLog` query (filter `targetId` baru), bukan fitur baru dari nol.

## Objective
Section "Riwayat Catatan" di halaman profil siswa (`(admin)/siswa/[id]/`, sudah bisa diakses piket) menampilkan JUGA **semua** kejadian siswa dikunci/dibuka-kunci — bukan cuma yang terakhir — lengkap dengan nama petugas yang melakukannya. Saat ini kejadian lock/unlock TIDAK muncul sama sekali di riwayat itu.

## Context
- **App:** `apps/api` (perluas `ActivityLog` query dengan filter `targetId` baru + gabungkan ke `riwayatCatatan()`) + `apps/web` (tampilkan tipe entry baru di tabel yang sudah ada)
- **Riset 2026-08-05 (Explore agent, baca kode + verifikasi behavior aktual)** — temuan penting, BUKAN asumsi:
  - Halaman profil siswa `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` **SUDAH ADA** dan **SUDAH bisa diakses piket** (`page.tsx:7-15,32-33` — `isPiket` dari `getCurrentUser().role === "guru_piket"`, dapat akses read-only `readOnly=true`, cukup untuk kebutuhan melihat riwayat). **TIDAK PERLU halaman baru.**
  - Section `RiwayatCatatanSection` (`siswa-detail-view.tsx:611-691`) SUDAH menampilkan tabel Tanggal/Jenis/Detail/Petugas dari `GET /attendance/students/:id/riwayat-catatan`.
  - Backend `riwayatCatatan()` (`apps/api/src/attendance/attendance-report.service.ts:174-258`) SAAT INI cuma mengumpulkan 4 jenis entry: `terlambat`, `izin`, `sakit`, `alfa`, `pkl` — **TIDAK PERNAH query kejadian lock/unlock sama sekali**. Ini gap yang dikonfirmasi user perlu ditambahkan.
  - Untuk izin/sakit, kolom "Petugas" SUDAH benar terisi (`Permit.approvedById` wajib diisi, join ke `User.username`, baris ±247) — pola LABEL+PETUGAS ini di-REUSE untuk entry lock/unlock baru.
  - Terlambat/alfa/pkl SENGAJA tidak punya kolom petugas (masuk akal, bukan aksi manusia) — **TIDAK diubah oleh task ini**, tetap tampil "-" seperti sekarang.

## Keputusan Final (dikonfirmasi user 2026-08-05, setelah pertimbangan trade-off)

**Riwayat lock/unlock harus LENGKAP (semua kejadian dari awal siswa terdaftar, bukan cuma status terakhir)** — sumber data: `ActivityLog` (`action: "student.lock"` / `"student.unlock"`, sudah tercatat via `@LogActivity` di `students.controller.ts:141,153`), BUKAN kolom scalar `Student.lockedById`/`unlockedById` (yang overwrite tiap siklus, cuma nunjukin status terakhir).

**Kenapa `ActivityLog` dipilih dibanding bikin tabel baru (`StudentLockHistory`) atau kolom existing:**
- Skala sekolah ini (~2.500 siswa) membuat kejadian lock/unlock JARANG terjadi (puluhan-ratusan/tahun ajaran, bukan per-hari) — kekhawatiran performa query gabungan tidak relevan di skala ini, SELAMA index database yang tepat dipasang (lihat Spec Detail).
- `ActivityLog` SUDAH insert-only dan SUDAH mencatat kejadian ini lengkap dengan actor+waktu+snapshot — tabel baru terpisah akan berarti menulis ke 2 tempat sekaligus tiap lock/unlock terjadi (duplikasi sumber kebenaran, risiko drift kalau salah satu titik penulisan lupa di-update di masa depan, misal ada code path baru untuk lock yang lupa nulis ke tabel baru itu).
- Kerja tambahan MINIMAL: cuma butuh 1 field filter baru (`targetId`) di DTO yang sudah ada + 1 index database — bukan model/migration besar.

## Spec Detail

### Backend
- `apps/api/src/activity-log/dto/list-activity-log.dto.ts` — tambah field opsional `targetId?: number` (`@IsOptional() @Type(() => Number) @IsInt()`), dipakai untuk filter `WHERE targetId = :targetId` (digabung dengan `targetType` yang sudah ada supaya spesifik, misal `targetType: "student"` + `targetId: studentId`).
- `apps/api/prisma/schema.prisma` — tambah **index database** di `ActivityLog` pada kombinasi `(targetType, targetId)` (`@@index([targetType, targetId])`) — ini WAJIB dipasang bersamaan, supaya query "riwayat kejadian untuk 1 siswa tertentu" tetap cepat walau tabel `ActivityLog` terus tumbuh besar lintas semua modul lain. Migration baru untuk index ini.
- `apps/api/src/attendance/attendance-report.service.ts` — `riwayatCatatan()`:
  - Tambah query `ActivityLogService` (atau langsung Prisma kalau lebih sesuai pola existing di file ini) untuk `action IN ("student.lock", "student.unlock")`, `targetType: "student"`, `targetId: studentId`, urutkan `createdAt`.
  - Untuk tiap entry, ambil `actorId` → resolve nama petugas via join `User.username` (actor bisa `null` untuk lock OTOMATIS sistem 2x-terlambat — cek bagaimana `@LogActivity` mencatat `actorId` untuk aksi yang dipicu job/scheduler tanpa user login, kemungkinan `actorId` null atau butuh convention khusus, verifikasi saat implementasi).
  - Tambah 2 entry type baru ke `RiwayatCatatanEntry` union (baris ±6-16): `"terkunci"` dan `"dibuka"`, field `namaPetugas` (nullable untuk kasus sistem otomatis) + `alasan` (dari `snapshotAfter`/`snapshotBefore` JSON kalau `lockedReason` ada di situ — cek struktur snapshot aktual saat implementasi).
  - Gabungkan ke array hasil, urutkan kronologis bersama entry lain yang sudah ada (`sort by tanggal desc`, cek logic sorting existing baris ±255-258).

### Frontend
- `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` — `RiwayatCatatanSection`:
  - Tambah handling render untuk `entry.jenis === "terkunci"` dan `"dibuka"` di kolom Jenis (label ramah: "Dikunci"/"Dibuka") dan Detail (alasan lock kalau ada, fallback "-").
  - Kolom Petugas: tampilkan `entry.namaPetugas` untuk 2 jenis baru ini (perluas kondisi baris ±681-683 yang saat ini hanya `izin`/`sakit`) — untuk lock otomatis sistem (`namaPetugas` null), tampilkan label "Sistem (otomatis)", BUKAN "-" (supaya tidak disalahartikan sebagai data tidak lengkap).

## Edge Cases
- Siswa yang belum pernah dikunci sama sekali → tidak ada entry terkunci/dibuka muncul (bukan baris kosong/error).
- Siswa yang SEDANG terkunci (belum pernah di-unlock) → entry "terkunci" muncul, TIDAK ada entry "dibuka" pasangan (karena belum terjadi) — pastikan logic tidak mengasumsikan selalu berpasangan.
- Siswa yang dikunci-dibuka-dikunci berkali-kali → SEMUA siklus muncul sebagai entry terpisah, terurut kronologis (ini alasan utama kenapa `ActivityLog` dipilih dibanding kolom `Student`).
- Lock otomatis sistem (2x-terlambat) → label "Sistem (otomatis)", bukan kosong/error.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (index baru di `ActivityLog`), `apps/api/src/activity-log/dto/list-activity-log.dto.ts` (`targetId` filter), `apps/api/src/attendance/attendance-report.service.ts` (`riwayatCatatan()`), `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` (`RiwayatCatatanSection`).
- **Buat:** migration Prisma baru untuk index.
- **Jangan sentuh:** `Student` model schema (tidak perlu field baru), endpoint lain yang memakai `riwayatCatatan()` atau `GET /activity-log` kalau ada pemanggil lain — cek dulu grep pemanggil sebelum ubah shape response/DTO, supaya tidak breaking (terutama `GET /activity-log` yang sudah dipakai halaman admin `/log` — pastikan field `targetId` baru ini OPSIONAL, tidak mengubah perilaku default kalau tidak diisi).

## Acceptance Criteria
- [x] Riwayat Catatan siswa menampilkan SEMUA siklus lock/unlock dari awal (bukan cuma terakhir), tiap entry dengan tanggal+petugas yang benar. **Diverifikasi live**: lock manual→unlock manual→lock otomatis berturut-turut untuk siswa uji, ketiga entry muncul terpisah dan kronologis di `riwayat-catatan`.
- [x] Lock otomatis sistem (2x-terlambat) menampilkan label "Sistem (otomatis)", bukan kosong. **Diverifikasi live**: `namaPetugas: "sistem"` + `otomatis: true` dari API, frontend render "Sistem (otomatis)" (bukan username mentah).
- [x] Terlambat/izin/sakit/alfa/pkl entry TIDAK berubah perilakunya (regresi nol). Logic entry lama tidak disentuh sama sekali, hanya ditambah cabang baru.
- [x] Index database di `ActivityLog(targetType, targetId)` — **SUDAH ADA sebelum task ini** (ditemukan saat implementasi, bukan asumsi spec awal yang minta migration baru), diverifikasi `EXPLAIN` query plan: `type: ref`, `key: activity_log_target_type_target_id_idx` (bukan `ALL`/full-scan).
- [x] Halaman admin `/log` tetap berfungsi normal. **Diverifikasi live**: `GET /activity-log` (tanpa filter `targetId`) tetap 200, `auth.login` tetap terlihat di sana (tidak difilter, beda dari `/activity-log/me` piket).
- [x] Build + type-check `apps/api` dan `apps/web` hijau. `tsc --noEmit` bersih kedua app, `nest build`+`next build` sukses, jest 183/183 (1 spec file perlu diperbaiki — lihat Status Eksekusi).

## Validasi Claudian
- [x] **Verifikasi `actorId` lock otomatis MENEMUKAN GAP NYATA** (bukan cuma "pastikan tidak crash") — lock otomatis (`applyLateStrikeLock()`) TERNYATA sama sekali tidak menulis ke `ActivityLog` (beda code path dari lock manual, tidak lewat `@LogActivity`), dan `actorId` di schema NOT NULL (tidak bisa `null` untuk "sistem"). Dikonfirmasi ke user, solusi: akun placeholder "sistem" (`status: nonaktif`, tidak bisa login, password hash acak) — lihat "Celah Ditemukan" di bawah.
- [x] Grep pemanggil lain `riwayatCatatan()`/`RiwayatCatatanEntry`/`GET /activity-log`/`ListActivityLogDto` dilakukan — hanya 1 caller masing-masing (`attendance.controller.ts`, `siswa-detail-view.tsx`), tidak ada breaking change di tempat lain. `targetId` di DTO SENGAJA opsional, tidak mengubah default `/log` admin.

## Celah Desain Ditemukan+Diperbaiki (2026-08-06, saat implementasi, bukan bagian spec awal)
Spec awal MENGASUMSIKAN lock otomatis sudah tercatat di `ActivityLog` dengan `actorId` nullable — KEDUANYA salah setelah dicek ke kode: (1) `applyLateStrikeLock()` di `attendance.service.ts` dipicu internal dari `tap()`, BUKAN lewat HTTP endpoint dengan `@LogActivity` interceptor, jadi tidak pernah tercatat ke `ActivityLog` sama sekali; (2) `ActivityLog.actorId` adalah FK NOT NULL ke `User`, tidak bisa diisi `null` untuk "sistem".

**Dikonfirmasi user 2026-08-06**: buat akun placeholder `"sistem"` (migration `20260806161900_t112_akun_sistem`, `role: super_admin`, `status: nonaktif` — otomatis DITOLAK oleh `AuthService` di SEMUA jalur login karena cek `status !== aktif` sebelum verifikasi password, password hash acak tidak pernah dibagikan). `applyLateStrikeLock()` sekarang manual memanggil `ActivityLogService.record()` dengan `actorId` akun ini setiap kali lock otomatis terjadi — riwayat lock otomatis sekarang LENGKAP semua siklus, bukan cuma status terakhir.

## Status Eksekusi — SELESAI (2026-08-06)
**Migration baru**: `20260806161900_t112_akun_sistem` (insert 1 baris `users`, BUKAN perubahan schema — index `(targetType, targetId)` sudah ada sebelumnya).
**Backend**: `ActivityLogService.getSystemActorId()` (resolve+cache id akun sistem via username, bukan hardcode angka). `AttendanceService` inject `ActivityLogService` (butuh `AttendanceModule` import `ActivityLogModule` baru), `applyLateStrikeLock()` panggil `activityLog.record()` manual. `AttendanceReportService` inject `ActivityLogService`, `riwayatCatatan()` query `ActivityLog` (`action IN (student.lock, student.unlock)`, `targetType: student`, `targetId: studentId`), map jadi 2 entry jenis baru (`terkunci`/`dibuka`) dengan `otomatis` flag (`actorId === systemActorId`, dibandingkan by id bukan username string). `ListActivityLogDto` tambah `targetId?: number` opsional, `ActivityLogService.findAll()` terima parameter kedua `excludeActions` (dipakai `/activity-log/me` piket, TIDAK dipakai `/log` admin).
**Frontend**: `RiwayatCatatanEntry` union diperluas (duplikat di `core-types.ts`, tidak ada shared-types package), `RIWAYAT_LABEL`/`RIWAYAT_BADGE_CLASS`/`RIWAYAT_ICON` (Lock/LockOpen icon) diperluas, kolom Detail/Petugas di `RiwayatCatatanSection` handle jenis baru.
**Bug ditemukan+diperbaiki saat build**: `attendance-report.service.spec.ts` (test `report()`, bukan `riwayatCatatan()`) perlu update instantiasi service (constructor sekarang 2 parameter) — mock `ActivityLogService` kosong cukup karena tidak dipakai di test itu.
Diverifikasi live end-to-end: lock manual→unlock manual→lock otomatis (simulasi 2x terlambat via insert data + replika logic `applyLateStrikeLock` standalone, karena tap sungguhan butuh precondition jadwal `jam_sekolah` yang tidak lengkap di data dev), index `EXPLAIN` query plan `ref` (bukan full-scan), admin `/log` regresi nol. Semua data test dibersihkan dari DB setelahnya.
