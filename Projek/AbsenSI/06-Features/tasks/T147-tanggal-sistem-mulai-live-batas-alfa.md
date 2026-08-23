# T147 — Schema+API+Web: Tanggal "Sistem Mulai Live" (Batas Global Perhitungan Alfa)

## Depends on
**WAJIB T144 (fix bug alfa=0) selesai dulu** — task ini memperluas `resolveHariWajib()`/`report()` yang diperbaiki T144, menambah 1 lapis pengecualian baru di atas fondasi yang sama. TIDAK depends on T145/T146 (independen dari jalur jam-masuk-sekolah).

## Objective
Tambah 1 tanggal GLOBAL "Sistem Mulai Live" (`liveSince`, 1 nilai untuk seluruh sekolah, bukan per siswa) — perhitungan Alfa (rekap DAN live papan piket setelah T146) **TIDAK PERNAH** menghitung Alfa untuk hari-hari SEBELUM tanggal ini, apa pun kondisi data siswa/kartu saat itu. Ini memungkinkan admin mengaktifkan `AcademicYear` dengan `tanggalMulai` mundur jauh (misal ke awal tahun ajaran) TANPA salah menghukum siswa sebagai Alfa untuk periode sebelum AbsenSI benar-benar dipakai sekolah untuk mencatat kehadiran real.

## Context — Kenapa Task Ini Ada (Diskusi 2026-08-08)

User berencana mengaktifkan `AcademicYear` dengan `tanggalMulai` mundur ke tanggal go-live sistem (bukan cuma tahun ajaran berjalan saat ini) — supaya rekap Alfa historis mulai terhitung benar sejak awal. Saat didiskusikan, ditemukan masalah data:

**`Card.issuedAt` untuk 44 dari 46 kartu siswa di database SAAT INI adalah hasil MIGRASI dari sistem lama** (dikonfirmasi lewat cluster timestamp: 44 baris ber-`issuedAt` mengelompok dalam jendela 6 detik, `2026-07-30T12:37:02–08Z` — pola khas bulk-insert migrasi, BUKAN issuance satu-per-satu), sehingga nilainya BUKAN tanggal siswa itu SEBENARNYA menerima kartu fisiknya, melainkan "waktu script migrasi dijalankan". `report()` (T113) SUDAH benar mengecualikan hari sebelum `Card.issuedAt` dari hitungan Alfa (kategori terpisah "Belum Memiliki Kartu") — TAPI karena `issuedAt` kartu migrasi itu sendiri TIDAK AKURAT, logic yang secara individual sudah benar ini akan menghasilkan kesimpulan SALAH kalau `AcademicYear.tanggalMulai` diset mundur melewati tanggal migrasi (30 Juli 2026) — siswa lama yang sebenarnya sudah punya kartu jauh sebelum itu akan salah dikategorikan "Belum Memiliki Kartu" untuk periode yang seharusnya dihitung normal.

**Keputusan final** (dikonfirmasi user, bukan backfill per-kartu individual — ditolak eksplisit "kalau 1 tanggal, berarti harus ada toggle untuk mengaktifkan..."): daripada mencoba merekonstruksi tanggal kartu asli per siswa (data itu TIDAK ADA di sistem lama, tidak bisa direkonstruksi akurat), buat **1 penanda GLOBAL "Sistem Mulai Live"** — SEBELUM tanggal ini, TIDAK ADA siswa yang dihitung Alfa sama sekali, TERLEPAS dari `Card.issuedAt` individual mereka. Ini konseptual BEDA dari `AcademicYear.tanggalMulai` (yang menentukan rentang "hari wajib" untuk tahun ajaran) — `liveSince` adalah batas TAMBAHAN yang berlaku LINTAS semua tahun ajaran (kalau nanti ada tahun ajaran lebih lama dari `liveSince`, tetap `liveSince` yang jadi batas bawah efektif untuk Alfa).

**TIDAK PERLU backfill `Card.issuedAt` kartu migrasi** — task ini MENGGANTIKAN kebutuhan itu sepenuhnya (lebih sederhana, 1 nilai global vs 44+ nilai individual yang tidak bisa dipastikan akurat).

## Spec Detail

### 1. Schema (Prisma) — 1 field baru, pola singleton config

Tambah ke model config yang SUDAH ADA kalau relevan (cek `ScheduleConfig` atau serupa yang sudah 1-baris-global) ATAU buat model singleton baru KECIL kalau tidak ada yang cocok secara semantik (`liveSince` bukan tentang jadwal, jadi kemungkinan besar BUTUH model baru terpisah, bukan menumpang ke `ScheduleConfig`):

```prisma
model SystemLiveConfig {
  id          Int      @id @default(1)
  liveSince   DateTime @map("live_since")
  updatedById Int      @map("updated_by")
  updatedAt   DateTime @updatedAt @map("updated_at")

  updatedBy User @relation(fields: [updatedById], references: [id])
  @@map("system_live_config")
}
```
- Pola singleton PERSIS sama seperti `AttendanceLockConfig`/`EkstraRegistrationConfig` (`SINGLETON_ID = 1`, enforce di service layer bukan DB constraint).
- **TIDAK ADA default value untuk `liveSince`** — admin WAJIB mengisi ini secara eksplisit sebelum sistem bisa mengaktifkan `AcademicYear` mundur (lihat poin 3, validasi silang). Kalau baris config belum pernah dibuat sama sekali (`findFirst()` return null) — PERLAKUKAN sebagai "belum di-set", behavior fallback dijelaskan di poin 2.
- Migration baru.

### 2. Backend — perluas `resolveHariWajib()` (T144) dengan lapis pengecualian baru

- Di `AttendanceReportService.resolveHariWajib()` (sudah diperbaiki T144 dengan sinyal `adaTahunAjaranAktif`) — TAMBAH parameter/logic baru: setelah `wajibDates` dihitung SEPERTI SEKARANG (dari `AcademicYear.isActive` + exclude `SchoolHoliday`), **FILTER TAMBAHAN**: buang semua tanggal di `wajibDates` yang **LEBIH AWAL** dari `SystemLiveConfig.liveSince` (kalau config belum di-set/`null`, JANGAN filter apa pun — fallback ke behavior T144 apa adanya, supaya sistem yang belum sempat set `liveSince` tidak tiba-tiba kehilangan SEMUA kemampuan hitung Alfa; ini murni pengecualian TAMBAHAN, bukan pengganti).
- **Method ini DIPAKAI oleh SEMUA 4 caller yang sudah diperbaiki T144** (`report()`, `riwayatCatatan()`, `TvPiketService.hitungPersentase()`, `TvPiketService.hitungSiswaTidakHadir()`) — filter `liveSince` otomatis berlaku ke SEMUANYA tanpa perlu ubah caller-nya (KARENA sudah terpusat di `resolveHariWajib()`, tinggal tambah 1 filter di titik itu).
- **`T146` (Alfa Live papan piket, kalau sudah dikerjakan saat T147 dimulai)** — pastikan `resolveHariWajib()` yang dipanggil `piketBoard()` di T146 JUGA otomatis kena filter ini (karena reuse method yang sama) — TIDAK perlu perubahan tambahan di T146 kalau memang reuse method terpusat sesuai spec T146.

### 3. Backend — validasi silang saat admin mengaktifkan `AcademicYear`

- **TIDAK WAJIB blocking** (jangan menolak aktivasi tahun ajaran kalau `liveSince` belum di-set — itu 2 fitur independen, `AcademicYear` sudah ada sebelum fitur ini), TAPI **REKOMENDASI kuat**: kalau admin mengaktifkan `AcademicYear` dengan `tanggalMulai` LEBIH AWAL dari `liveSince` (atau `liveSince` belum di-set sama sekali) — response `activate()` (`AcademicYearsService`) sertakan **peringatan** (bukan error, sama pola non-blocking seperti `peringatan` di T144) mengingatkan admin: "Tanggal mulai tahun ajaran ini lebih awal dari Tanggal Sistem Mulai Live (atau belum di-set) — Alfa untuk periode sebelum itu tidak akan dihitung." — supaya admin sadar konsekuensinya SEBELUM bingung kenapa rekap sebagian periode kosong Alfa.

### 4. Backend — endpoint config baru
- `GET /system-live-config` — accessible semua role yang butuh (minimal admin), return `{ liveSince: string | null, updatedAt, updatedBy }`.
- `PATCH /system-live-config` — `@Roles(UserRole.super_admin)` SAJA, `@LogActivity` wajib (perubahan tanggal ini berdampak besar ke SELURUH rekap, audit trail penting).
- DTO: `{ liveSince: string }` (ISO date, wajib).

### 5. Frontend — halaman setting
- Tambahkan ke halaman pengaturan yang sudah relevan (kemungkinan dekat `pengaturan-absensi-view.tsx` atau halaman Tahun Ajaran/Semester — PUTUSKAN lokasi paling sesuai saat implementasi, cek struktur menu admin existing) — 1 date picker "Tanggal Sistem Mulai Live" dengan penjelasan singkat: "Alfa tidak akan dihitung untuk tanggal sebelum ini, berlaku untuk seluruh rekap." Kalau belum pernah di-set, tampilkan state jelas ("Belum diatur — Alfa dihitung tanpa batas bawah tambahan").
- Kalau ada halaman kelola Tahun Ajaran (`academic-years-section.tsx` atau serupa, disebut riset sebelumnya) — TAMBAHKAN peringatan visual (banner kecil) di halaman itu KETIKA admin akan mengaktifkan tahun ajaran yang `tanggalMulai`-nya lebih awal dari `liveSince` (reuse `peringatan` dari poin 3 di response backend).

## Edge Cases
- `liveSince` di-set KE MASA DEPAN (lebih besar dari hari ini, kesalahan input admin) — TIDAK PERLU validasi blocking khusus (biarkan admin bertanggung jawab, ini setting konfigurasi bukan celah keamanan), TAPI PERTIMBANGKAN tampilkan peringatan halus di UI kalau nilai yang diisi tampak tidak wajar (di masa depan) — putuskan saat implementasi apakah worth ditambahkan, tidak WAJIB.
- `liveSince` diubah admin SETELAH beberapa rekap sudah pernah dibuka/di-export sebelumnya — TIDAK PERLU invalidasi cache/dokumen lama (rekap yang sudah di-download/print sebelumnya TETAP seperti apa adanya saat itu, wajar untuk laporan historis) — rekap BARU yang dibuka setelah perubahan otomatis pakai nilai `liveSince` TERBARU.
- Rentang tanggal rekap YANG DIMINTA user SELURUHNYA sebelum `liveSince` (misal rekap Januari 2026 tapi `liveSince` di-set Maret 2026) — `wajibDates` untuk rentang itu jadi KOSONG TOTAL (semua tanggal di rentang itu difilter habis) — hasil rekap: Hadir/Izin/Sakit tetap terhitung apa adanya (TIDAK terpengaruh filter ini, filter HANYA mempengaruhi Alfa), Alfa = 0 untuk SEMUA siswa di rentang itu — INI PERILAKU YANG BENAR (bukan bug baru), TAPI pastikan `peringatan` (T144) TIDAK ikut muncul untuk kasus ini (beda dari kondisi "tidak ada AcademicYear aktif" yang memang harus dapat peringatan) — 2 kondisi ini harus dibedakan sinyalnya di response, JANGAN sampai tercampur jadi 1 pesan generik yang membingungkan admin.

## Files
- **Buat:** migration Prisma baru (model `SystemLiveConfig`), modul `apps/api/src/system-live-config/` (controller+service+dto+module, pola sama `attendance-lock-config`).
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (`resolveHariWajib()`, tambah filter `liveSince`), `apps/api/src/calendar/academic-years/academic-years.service.ts` (`activate()`, tambah peringatan non-blocking), halaman admin setting yang relevan (lokasi diputuskan saat implementasi).
- **Jangan sentuh:** `Card.issuedAt` (TIDAK di-backfill, task ini menggantikan kebutuhan itu), logic "Belum Memiliki Kartu" di `report()` (T113, TIDAK diubah — filter `liveSince` ini TAMBAHAN, bukan pengganti logic kartu yang sudah benar).

## Acceptance Criteria
- [x] Model `SystemLiveConfig` (singleton, `id=1`), field `liveSince` wajib diisi eksplisit oleh admin (tidak ada default).
- [x] `resolveHariWajib()` mengecualikan SEMUA tanggal sebelum `liveSince` dari `wajibDates` — kalau `liveSince` belum di-set, TIDAK ADA filter tambahan (fallback ke behavior T144 apa adanya) — **diverifikasi LIVE**.
- [x] Filter ini otomatis berlaku ke SEMUA 4 caller `resolveHariWajib()` (report, riwayatCatatan, TvPiket×2) TANPA perlu ubah kode di masing-masing caller (terpusat di 1 titik — dikonfirmasi lewat DI NestJS, `TvPiketService` inject `AttendanceReportService` sebagai instance utuh, otomatis dapat constructor baru tanpa perubahan kode apa pun di `tv-piket.service.ts`).
- [x] Admin bisa set/ubah `liveSince` via halaman setting baru (di `pengaturan-absensi-view.tsx`), `@Roles(super_admin)`, activity log tercatat MANUAL (pola sama config singleton lain, bukan `@LogActivity` decorator).
- [x] Mengaktifkan `AcademicYear` dengan `tanggalMulai` lebih awal dari `liveSince` (atau `liveSince` belum di-set) → tampil peringatan non-blocking (TIDAK menolak aktivasi) — **diverifikasi LIVE**.
- [x] Rekap untuk rentang SELURUHNYA sebelum `liveSince` → Hadir/Izin/Sakit tetap benar, Alfa=0 untuk semua siswa di rentang itu (perilaku BENAR, bukan bug), TANPA memicu `peringatan` T144 — **diverifikasi via unit test eksplisit** (`report — T147 filter liveSince`, skenario 3).
- [x] Build + type-check `apps/api` dan `apps/web` hijau. Test suite existing lulus 100% (231→238, 7 test baru).

## Validasi Claudian
- [x] **JANGAN** backfill `Card.issuedAt` — dikonfirmasi TIDAK disentuh sama sekali.
- [x] **JANGAN** membuat `liveSince` blocking — dikonfirmasi `activate()` TETAP mengaktifkan tahun ajaran, `peringatan` murni field tambahan di response, tidak pernah throw.
- [x] Sinyal `peringatan` T144 vs kondisi liveSince TIDAK tercampur — dikonfirmasi lewat code read: `adaTahunAjaranAktif` (basis `peringatan` T144 di `reportFlexible()`) dihitung dari `academicYear.count({isActive:true})` GLOBAL, TIDAK PERNAH tersentuh oleh loop filter `liveSince` (2 mekanisme independen di method yang sama) — DAN diverifikasi test eksplisit (skenario liveSince SETELAH seluruh rentang rekap, `adaTahunAjaranAktif` tetap true karena tahun ajaran aktif).
- [x] Verifikasi T144 sudah selesai — dikonfirmasi sebelum mulai (STATUS.md), `resolveHariWajib()` versi T144 (dengan sinyal `adaTahunAjaranAktif`) dipakai sebagai basis langsung.

## Status Eksekusi (2026-08-09)
- **Schema**: migration `20260809021806_t147_system_live_config`, model `SystemLiveConfig` singleton (`id=1`, TIDAK ada default value untuk `liveSince` — kolom `DateTime @db.Date` non-nullable di level kolom TAPI baris itu sendiri opsional ada/tidak, `findFirst()` return `null` kalau belum pernah dibuat, konsisten pola `AttendanceLockConfig`). Diterapkan bersih ke dev DB.
- **Backend — modul baru** `system-live-config/` (controller+service+dto+module, pola PERSIS `attendance-lock-config/`). `get()` return `{liveSince: string|null, updatedAt, updatedById}` (null-safe kalau baris belum ada). `getLiveSinceDate()` — method terpisah KHUSUS dipakai internal (`AttendanceReportService`, `AcademicYearsService`), return `Date|null` mentah (bukan string) supaya bisa dibandingkan langsung tanpa parsing ulang. `update()` upsert by `SINGLETON_ID=1`, activity log manual (`activityLogService.record()`, BUKAN `@LogActivity` — pola sama T143/T144/T146, endpoint singleton tidak punya `id` route param yang cocok dengan snapshot-fetch otomatis decorator).
- **`resolveHariWajib()` (T144) diperluas**: tambah query paralel ke-4 (`systemLiveConfig.getLiveSinceDate()`) di `Promise.all` yang sudah ada, 1 baris `if (liveSince && cursor < liveSince) continue;` di dalam loop hari yang sudah ada — filter TAMBAHAN murni, urutan cek TIDAK mengubah logic weekday/holiday yang sudah ada. `adaTahunAjaranAktif` (basis peringatan T144) TIDAK tersentuh — computed sebelum loop filter ini jalan, 2 sinyal independen SENGAJA dijaga terpisah (lihat Validasi Claudian).
- **`AcademicYearsService.activate()` diperluas**: fetch `liveSince` paralel dengan transaction activate (Promise.all, tidak menunggu berurutan), bandingkan `academicYear.tanggalMulai < liveSince` (atau `liveSince` null) → set `peringatan` di response. Response shape berubah dari `AcademicYear` polos jadi `AcademicYear & {peringatan}` — REGRESI DICEK: satu-satunya frontend consumer (`academic-years-section.tsx`) sudah diupdate untuk baca field baru ini, tidak ada consumer lain.
- **Frontend**: `pengaturan-absensi-view.tsx` — section baru `SystemLiveConfigSection` (date picker + tombol Simpan + pesan status jelas "Belum diatur" vs "Saat ini: [tanggal]"), peringatan halus (non-blocking) kalau tanggal yang diisi di masa depan. `page.tsx` fetch 2 config paralel. `academic-years-section.tsx` — `handleActivate()` sekarang baca `peringatan` dari response, tampilkan banner amber (`status-shipped` token, konsisten T144) di atas daftar tahun ajaran kalau ada.
- **Verifikasi**: `tsc --noEmit` bersih `apps/api`+`apps/web`. `jest` 238/238 pass (7 test baru: 3 `resolveHariWajib` filter liveSince — null/di-tengah-rentang/setelah-seluruh-rentang — di `attendance-report.service.spec.ts`, 4 `activate()` peringatan — not-found/belum-set/lebih-awal/sama-atau-setelah — di `academic-years.service.spec.ts`). **Live end-to-end** via script `NestFactory.createApplicationContext` (bypass HTTP, tidak ada JWT admin tersedia sesi ini) terhadap dev DB nyata — 6 skenario BERURUTAN CONFIRMED: (1) `get()` null sebelum di-set; (2) `resolveHariWajib` 5 hari wajib TANPA filter; (3) `SystemLiveConfigService.update()` tersimpan; (4) `resolveHariWajib` SEKARANG 3 hari (2 dikecualikan tepat sesuai `liveSince`); (5) `activate()` tahun ajaran ber-`tanggalMulai` sebelum `liveSince` → `peringatan` non-null CONFIRMED; (6) `get()` return nilai tersimpan. Semua data test dibuat+dihapus bersih, dev DB kembali 0 baris `system_live_config`+`academic_years`.
