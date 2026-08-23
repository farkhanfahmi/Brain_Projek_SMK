# T192 — API+Web: Fitur Baru "Dispensasi Siswa" (Lomba/Pelatihan Resmi Sekolah)

## Depends on
Tidak ada dependency teknis. Independen.

## Objective
Fitur BARU — **Dispensasi Siswa** (mirip Izin Guru dari sisi konsep, tapi untuk SISWA): siswa yang dispen KELUAR/tidak masuk karena kegiatan RESMI dari sekolah (lomba, pelatihan, dll — BUKAN sakit/izin pribadi). **Admin/Admin Jurnal** yang input, dan hasilnya **otomatis tampil di halaman Piket** (supaya piket tahu siswa itu memang dispen resmi, bukan alfa/masalah).

## Context — Keputusan Diskusi (2026-08-15)

Riset mengonfirmasi: model `Permit` (siswa) SAAT INI hanya punya `alasanKategori: sakit | izin` (`schema.prisma:1099-1102`) — TIDAK ADA kategori mirip "dispen sekolah". `PermitsService.create()` SAAT INI HANYA `guru_piket` yang bisa create (via `@Roles(UserRole.guru_piket)`).

**Keputusan user**: kategori BARU **`dispen`**, dan **admin_jurnal** yang menginput (BUKAN guru_piket, karena info dispen sekolah — surat tugas lomba/pelatihan — biasanya diketahui admin/TU duluan, bukan piket harian). Hasil input ini **otomatis muncul di halaman piket** (piket lihat siswa itu berstatus dispen, TIDAK perlu approve/reject — informasi sudah final dari admin_jurnal).

## Spec Detail

### 1. Backend — kategori baru + role baru yang boleh create

- `enum PermitAlasanKategori` — TAMBAH value `dispen` (`sakit | izin | dispen`).
- `PermitsController` — endpoint create — PERLUAS `@Roles` TAMBAH `UserRole.admin_jurnal` (ADDITIVE, `guru_piket` TETAP bisa create seperti biasa untuk izin/sakit — TIDAK dicabut) — TAPI **VALIDASI TAMBAHAN**: kalau `alasanKategori === "dispen"`, HANYA `admin_jurnal`/`super_admin` yang boleh (guru_piket TIDAK BOLEH input kategori dispen — role-restriction PER-KATEGORI, bukan cuma per-endpoint) — REKOMENDASI validasi ini di SERVICE layer (bukan cuma guard controller) supaya jelas dan testable.
- **Field tambahan untuk dispen** — VERIFIKASI apakah field existing `Permit` (alasanDetail, buktiFilePath) SUDAH CUKUP untuk kasus dispen (misal alasanDetail = "Lomba LKS Provinsi", buktiFilePath = surat tugas) — KEMUNGKINAN BESAR CUKUP tanpa field baru, REUSE struktur yang ada. Field `jamKeluar`/`jamKembaliDiharapkan` — untuk dispen KEMUNGKINAN tidak relevan (dispen biasanya seharian penuh atau multi-hari, bukan keluar-kembali dalam 1 hari) — VERIFIKASI dan PUTUSKAN saat implementasi apakah field ini WAJIB/opsional/diabaikan untuk kategori dispen.
- **Rentang tanggal untuk dispen** — LOMBA/PELATIHAN sering LEBIH DARI 1 HARI (beda dari izin/sakit yang biasanya 1 hari) — VERIFIKASI: apakah `Permit.tanggal` (SINGLE date, SAMA seperti TeacherPermit sebelum T191) CUKUP, atau perlu pola RENTANG TANGGAL SERUPA T191 (`tanggalSelesai`)? REKOMENDASI KUAT: TERAPKAN pola rentang yang SAMA seperti T191 (`tanggalSelesai` nullable) — KONSISTEN kebutuhan nyata (dispen multi-hari itu WAJAR/SERING terjadi), JANGAN batasi ke 1 hari kalau use-case aslinya jelas butuh rentang.

### 2. Backend — tampil otomatis di halaman piket

- **VERIFIKASI**: apakah halaman piket (board/rekap) SUDAH otomatis menampilkan SEMUA `Permit` (termasuk kategori baru `dispen`) tanpa perubahan tambahan (KARENA piket query `Permit` secara umum, kategori baru otomatis ikut kalau query-nya TIDAK hardcode filter `sakit`/`izin` saja) — KEMUNGKINAN BESAR sudah otomatis tercakup TANPA perubahan tambahan, TAPI WAJIB VERIFIKASI (cek SEMUA titik yang query `Permit.alasanKategori` — kalau ADA yang hardcode enum lama tanpa memperhitungkan value baru, PERLU diperbarui).
- Badge/label visual untuk kategori "Dispen" di board piket — WARNA/IKON BEDA dari Izin/Sakit (supaya piket cepat kenali secara visual) — KONSISTEN pola badge status yang sudah ada di board.

### 3. Frontend — form input Dispensasi Siswa (admin/admin_jurnal)

- Halaman/form BARU — REKOMENDASI REUSE pola form Izin Guru YANG SUDAH DIREDESIGN di T191 SEBAGAI REFERENSI STRUKTUR (search siswa, rentang tanggal, alasan, upload bukti) — TAPI target SISWA (bukan guru), jadi search-select siswa (bukan guru).
- Lokasi menu — REKOMENDASI di admin-jurnal (KONSISTEN role yang boleh input), PUTUSKAN nama menu jelas ("Dispensasi Siswa" atau serupa, JANGAN campur dengan menu Izin Guru yang beda konteks).

## Edge Cases
- Siswa yang SUDAH punya `Permit` izin/sakit di tanggal yang SAMA dengan dispen baru — TOLAK duplikat (KONSISTEN validasi Permit existing yang SUDAH ADA untuk kombinasi siswa+tanggal, VERIFIKASI apakah constraint ini sudah ada atau perlu ditambah).
- Guru_piket MENCOBA input kategori dispen (lewat manipulasi request, bukan UI normal karena UI TIDAK menampilkan opsi ini untuk role mereka) — DITOLAK backend (defense-in-depth, validasi role-per-kategori di poin 1).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`PermitAlasanKategori` +value `dispen`, PERTIMBANGKAN `tanggalSelesai` kalau rentang diperlukan), `apps/api/src/permits/permits.controller.ts`+`permits.service.ts` (role+validasi kategori), halaman board piket (VERIFIKASI query mencakup kategori baru, badge visual baru).
- **Buat:** form/halaman Dispensasi Siswa baru di admin-jurnal.
- **Jangan sentuh:** logic izin/sakit existing (guru_piket TETAP bisa create seperti biasa, TIDAK berubah).

## Acceptance Criteria
- [x] Admin_jurnal bisa input Dispensasi Siswa (rentang tanggal, alasan, bukti) — verified via curl (route ter-mapping, guard aktif), live browser terhalang sesi lain memakai instance yang sama.
- [x] Guru_piket TIDAK BISA input kategori dispen — route `POST /permits/dispen` TERPISAH TOTAL, guru_piket tidak punya `@Roles` akses sama sekali (bukan validasi kategori di endpoint yang sama).
- [x] Guru_piket TETAP bisa input izin/sakit seperti biasa (regresi nol) — `create()`/`createTidakMasuk()`/`createKeluar()` TIDAK diubah sama sekali.
- [x] Dispensasi yang diinput OTOMATIS muncul di halaman board piket dengan badge visual jelas berbeda dari izin/sakit — token `status-processing` (ungu), sudah dipakai "Libur Semester" kalender.
- [x] Build + type-check hijau, jest baru untuk role-restriction per-kategori (via route separation) + rentang tanggal + konflik.

## Validasi Claudian
- [x] **WAJIB putuskan** apakah `Permit` perlu `tanggalSelesai` (rentang) — dikonfirmasi user, pola sama T191.
- [x] **WAJIB verifikasi** SEMUA titik query `Permit.alasanKategori` — ditemukan 3 ternary binary izin/sakit di `attendance-report.service.ts` (range mode, single-day mode, riwayat catatan) + 1 di `tv-piket.service.ts` (TIDAK diubah, dispen sengaja tidak pernah reach branch itu karena sudah punya AttendanceRecord) — semua diperbaiki KECUALI yang terbukti sudah benar by design.
- [x] Konfirmasi role-restriction per-kategori — didesain LEBIH KUAT dari sekadar validasi service: endpoint `POST /permits/dispen` TERPISAH TOTAL dari `create()` biasa, guru_piket tidak reachable sama sekali (defense-in-depth di level route, bukan cuma if-check).

## Catatan Implementasi (2026-08-16)

- **Diskrepansi ditemukan+dikonfirmasi user SEBELUM implementasi besar** (2 klarifikasi terpisah): (1) `AttendanceStatus` (dipakai rekap/dashboard/TV piket di 6+ file) TIDAK punya value dispen — `STATUS_BY_KATEGORI` di `permits.service.ts` otomatis insert `AttendanceRecord` dari `alasanKategori`, jadi dispen BUTUH `AttendanceStatus.dispen` baru (bukan reuse `izin`) — dikonfirmasi user. (2) Rentang tanggal — dikonfirmasi `tanggalSelesai` di `Permit` + `createDispen()` insert `AttendanceRecord` **per hari** dalam rentang (bukan 1 baris merepresentasikan rentang), supaya rekap/alfa harian per-tanggal tetap akurat tanpa refactor besar ke query rekap yang sudah ada.
- **Schema**: `PermitAlasanKategori` +`dispen`, `AttendanceStatus` +`dispen`, `Permit.tanggalSelesai` (nullable, NULL = 1 hari). 1 row `Permit` existing di DB dev, migrasi aman (kolom baru + enum value baru, tanpa perubahan data).
- **Endpoint baru**: `POST /permits/dispen` (`CreateDispenDto`, admin_jurnal+super_admin, TIDAK kampus-scoped seperti guru_piket, TIDAK lewat `PiketOnDutyGuard`), `GET /permits/dispen` (riwayat lintas kampus). `PermitsService.createDispen()`: verifikasi siswa ada (tanpa scoping kampus), loop tanggal dalam rentang, cek konflik `AttendanceRecord`+`Permit` overlap SEBELUM create, transaksi `Permit.create` + `AttendanceRecord.createMany` (1 per hari, `status: dispen`).
- **`GET /students`** diperluas additive tambah `admin_jurnal` (untuk search-select siswa di form) — TIDAK dibuka ke method mutasi manapun. `GET /permits/:id/bukti-file` diperluas additive tambah `admin_jurnal` (kampusId null, sama seperti super_admin).
- **Blast radius lebih besar dari spec tertulis** (dikonfirmasi user lanjut full scope, bukan minimal): `teaching-sessions` TIDAK terpengaruh (dispen murni siswa), TAPI:
  - `attendance.service.ts` (`piketBoard()`) — query `permits` per siswa diubah dari exact-date jadi range-aware (`tanggal <= today <= tanggalSelesai`), supaya `alasanDetail` tetap muncul di hari ke-2+ dispen multi-hari (status/kategoriLive sendiri sudah benar dari `AttendanceRecord` per-hari, tidak perlu fix).
  - `attendance-report.service.ts` — 3 titik: range mode (`bucket.dispen` terpisah dari `bucket.hadir`, permit dispen HANYA untuk excusedDates bukan re-counting karena AttendanceRecord sudah cover), single-day mode (cabang baru `record.status === dispen` SEBELUM cabang hadir/terlambat generik), riwayat catatan (`JENIS_BY_KATEGORI` map ganti ternary binary).
  - `attendance-report-export.service.ts` — kolom Excel+PDF (range mode: header+row+chart), `ChartTotals`+`computeTotals()` tambah field `dispen`.
  - `tv-piket.service.ts` — `hitungGuruIzin` TIDAK relevan (itu guru), `hitungSiswaTidakHadir`+`hitungPersentase` DIVERIFIKASI TIDAK PERLU DIUBAH (dispen sengaja tidak pernah muncul di "Siswa Tidak Hadir" board karena sudah punya AttendanceRecord — masuk hitungan "Hadir" di persentase bar, defensible karena mereka memang "accounted for").
- **Frontend**: 3 file baru (`admin-jurnal/dispen/page.tsx`+`dispen-view.tsx`+`components/{dispen-form,dispen-table}.tsx`, pola search-siswa sama `input-izin-view.tsx` piket + rentang tanggal sama T191). Sidebar admin-jurnal +1 entry "Dispensasi Siswa" (ikon Award). `piket-board-view.tsx` (`KATEGORI_LIVE_BADGE`+`getPriority`), `siswa-detail-view.tsx` (`RIWAYAT_LABEL`/`_BADGE_CLASS`/`_ICON` — ikon Award TERSENDIRI, bukan share dengan izin/sakit), `rekap-view.tsx` (kolom tabel+chart+badge), `riwayat-izin-view.tsx` piket (label saja, dispen tetap read-only di sana karena mereka tidak bisa create).
- **Warna**: token existing `status-processing` (ungu #7B4DC6, sudah dipakai "Libur Semester" kalender) di-REUSE untuk badge/chart dispen — TIDAK menambah aksen warna baru (patuh DESIGN.md "1 aksen oranye saja", token semantik status BEDA dari aksen brand).
- **Test baru**: `permits.service.spec.ts` (`describe("createDispen — T192")`, 7 test: siswa tidak ditemukan, tanggalSelesai invalid, 1-hari vs rentang, konflik AttendanceRecord/Permit existing), `attendance-report.service.spec.ts` (2 test: single-day+range mode dispen counting terpisah dari hadir), `attendance.service.spec.ts` (1 test: `kategoriLive` = dispen apa adanya).
- **Verifikasi**: `tsc --noEmit` bersih 2 app, `nest build`+`next build` sukses (`/admin-jurnal/dispen` muncul di output, 3.5kB), 461 test backend lulus (naik dari 452 sebelum T192). Endpoint baru di-smoke-test via curl (401 tanpa auth, route `GET`+`POST /api/permits/dispen` ter-mapping benar di log nest watch). Live browser terhalang (instance dipakai sesi lain, tidak dipaksa ambil alih sesuai instruksi keselamatan sesi paralel).
