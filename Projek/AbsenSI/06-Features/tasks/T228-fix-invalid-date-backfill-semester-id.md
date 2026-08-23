# T228 — Fix "Invalid Date" Kalender Pendidikan + Backfill semesterId attendance_records + Bersihkan Tahun Ajaran Duplikat

## Depends on
Tidak ada dependency teknis. **CAMPURAN**: bug frontend murni (aman, prioritas tinggi) + migrasi data production (DESTRUKTIF-UPDATE, wajib protokol backup). WAJIB ikuti CLAUDE.md "Aturan WAJIB Sebelum Commit Migration Destruktif" untuk bagian migrasi data.

## Konteks — 3 Masalah Ditemukan Bersamaan (2026-08-20)

User laporkan rekap kosong setelah aktifkan Tahun Ajaran+Semester baru di production. Investigasi (dengan 1 kekeliruan riset awal yang dikoreksi — sub-agent riset sempat salah connect ke database yang bukan production sesungguhnya, dikoreksi via `docker exec absensi-mysql-prod` langsung, dikonfirmasi row count 4099 siswa/156 guru match data production asli) menemukan 3 masalah:

### 1. Bug "Invalid Date" — Halaman Kalender Pendidikan, section Semester

`apps/web/src/app/(admin)/kalender/semesters-section.tsx`, fungsi `formatTanggal()`:
```js
function formatTanggal(value: string) {
  return new Date(`${value}T00:00:00Z`).toLocaleDateString("id-ID", {
    day: "numeric", month: "short", year: "numeric", timeZone: "UTC",
  });
}
```
Backend mengirim `tanggalMulai`/`tanggalSelesai` sebagai **ISO datetime string PENUH** (Prisma `DateTime @db.Date` diserialisasi JSON jadi `"2026-07-20T00:00:00.000Z"`, BUKAN `"2026-07-20"` murni). Fungsi ini menambahkan suffix `T00:00:00Z` ke string yang SUDAH ISO lengkap → `"2026-07-20T00:00:00.000ZT00:00:00Z"` → `Invalid Date`. **DIREPRODUKSI dan dikonfirmasi 100% inilah penyebabnya** (bukan dugaan).

Section "Tahun Ajaran" di halaman yang sama (`academic-years-section.tsx`) TIDAK kena bug ini — parsing tanggalnya `new Date(year.tanggalMulai)` langsung tanpa suffix ganda, itu sebabnya tanggal Tahun Ajaran tampil normal sementara Semester "Invalid Date".

**Fix**: hapus suffix ganda, samakan pola dengan `academic-years-section.tsx`:
```js
function formatTanggal(value: string) {
  return new Date(value).toLocaleDateString("id-ID", { day: "numeric", month: "short", year: "numeric", timeZone: "UTC" });
}
```

### 2. `attendance_records` — `semesterId` NULL massal (akar masalah "rekap 0")

**Data production TERVERIFIKASI LANGSUNG** (`docker exec absensi-mysql-prod mysql ...`, database `absensi_db`, dikonfirmasi row count sesuai skala asli):

```
Total attendance_records: 18.943 baris
academic_year_id=2, semester_id=NULL: 18.943 baris (SEMUANYA)
Rentang tanggal: 2026-07-31 s.d. 2026-08-20
```

**Tahun Ajaran & Semester aktif SEBENARNYA di production:**
```
academic_years: id=2 "2026/2027", 2026-07-13 s.d. 2027-06-30, is_active=1
semesters:      id=1 "Ganjil",    2026-07-20 s.d. 2026-12-31, is_active=1
```

`academic_year_id` SEMUA baris SUDAH benar (=2, tertag otomatis sistem) — HANYA `semester_id` yang kosong, karena Semester "Ganjil" baru dibuat/diaktifkan SETELAH baris-baris attendance ini di-insert (rentang tanggal data 31 Juli-20 Agustus 2026 masuk akal terjadi sebelum Semester "Ganjil" resmi diaktifkan sistem).

**Rentang tanggal data (31 Juli - 20 Agustus) SELURUHNYA masuk rentang Semester Ganjil (20 Juli - 31 Desember 2026)** — backfill `semesterId=1` untuk baris-baris ini SECARA FAKTUAL BENAR, bukan "dipaksakan". Ini KONSISTEN pola migrasi T161 (13 Agustus, backfill serupa persis untuk masalah yang sama saat itu) — REPLIKASI pendekatan yang sama, bukan solusi baru.

### 3. Tahun Ajaran Duplikat — 2 Baris Nama Sama "2026/2027"

```
id=1: "2026/2027", 2026-07-13 s.d. 2026-06-30 (SALAH — tanggal selesai LEBIH AWAL dari tanggal mulai, jelas typo input), is_active=0
id=2: "2026/2027", 2026-07-13 s.d. 2027-06-30 (BENAR), is_active=1
```
`id=1` adalah percobaan input yang salah (typo tahun selesai) sebelum `id=2` dibuat dengan benar. `id=1` TIDAK AKTIF dan (perlu diverifikasi) kemungkinan TIDAK punya data terhubung — kandidat untuk dihapus, TAPI **tidak ada endpoint DELETE AcademicYear di backend saat ini** (dikonfirmasi via riset kode — fitur ini belum diimplementasikan).

## Spec Detail

### A. Fix Bug Frontend (aman, tidak destruktif, prioritas tinggi)

- `apps/web/src/app/(admin)/kalender/semesters-section.tsx` — perbaiki `formatTanggal()` sesuai poin 1 di atas.
- **VERIFIKASI SAAT IMPLEMENTASI**: cek apakah ada fungsi `formatTanggal` SERUPA di file lain dengan bug yang SAMA (grep pola `` `${value}T00:00:00Z` `` atau serupa di seluruh `apps/web/src`) — kalau ada duplikasi bug di tempat lain, perbaiki sekalian, JANGAN cuma 1 file kalau ternyata pola yang sama di-copy-paste ke beberapa komponen.

### B. Migrasi Backfill `semesterId` (DESTRUKTIF — WAJIB backup dulu)

1. **Backup manual production DULU** (`bash /home/anunnaki/scripts/backup-absensi.sh`, verifikasi file muncul) — WAJIB, tanpa pengecualian.
2. **Script migrasi terpisah** (REPLIKASI pola T161 — cek `apps/api/prisma/migrations/20260813010937_t161_backfill_academic_period_data_lama/migration.sql` sebagai referensi PERSIS, task ini kemungkinan besar bisa pakai pendekatan SQL serupa, BUKAN script TypeScript terpisah kalau polanya sama):
   ```sql
   UPDATE attendance_records
   SET semester_id = 1
   WHERE academic_year_id = 2
     AND semester_id IS NULL
     AND tanggal BETWEEN '2026-07-20' AND '2026-12-31';
   ```
   **VERIFIKASI SAAT IMPLEMENTASI**: JANGAN hardcode `semester_id = 1`/`academic_year_id = 2` sebagai angka mati di migration — resolve ID Semester "Ganjil" aktif dan AcademicYear aktif secara DINAMIS dari tabel `semesters`/`academic_years` (`WHERE is_active = 1`) SAAT MIGRASI DIJALANKAN, KONSISTEN pola T161 yang pakai `@target_academic_year_id` dinamis, BUKAN id literal — supaya migrasi tetap valid kalau ID berbeda saat benar-benar dieksekusi.
3. **Cek tabel LAIN dengan masalah sama** — `permits`, `ekstra_absen`, `ekstra_sesi`, `piket_journal_entries`, `teacher_permits`, `teaching_sessions`, `tap_events`, `student_pkl`, `late_entry_slips`, `activity_log`, `class_attendance_marks`, `journal_entries` — jalankan query serupa poin B.2 riset (`COUNT WHERE academic_year_id IS NULL OR semester_id IS NULL`) untuk SEMUA tabel ini SEBELUM migrasi, laporkan angka LENGKAP (jangan asumsi cuma `attendance_records` yang bermasalah — riset sebelumnya di sesi ini SEMPAT salah connect database, jadi VERIFIKASI ULANG dari awal, jangan percaya angka riset yang sudah terbukti keliru sebelumnya).
4. **Verifikasi SEBELUM/SESUDAH row count** (wajib protokol):
   ```sql
   -- SEBELUM
   SELECT COUNT(*) FROM attendance_records WHERE semester_id IS NULL;
   -- SESUDAH — harus 0 (atau berkurang sesuai baris yang match rentang tanggal semester aktif)
   SELECT COUNT(*) FROM attendance_records WHERE semester_id IS NULL;
   -- Total baris TIDAK BOLEH BERUBAH
   SELECT COUNT(*) FROM attendance_records;
   ```
5. **Setelah backfill, verifikasi rekap TIDAK LAGI 0** — buka halaman Rekap Kehadiran Murid production dengan filter Tahun Ajaran 2026/2027 + Semester Ganjil, konfirmasi data muncul.

### C. Tahun Ajaran Duplikat (id=1, salah input) — DIHAPUS 2026-08-21 (susulan, di luar sesi asli T228)

**UPDATE 2026-08-21**: user minta hapus lewat chat terpisah setelah lihat UI Kalender masih
menampilkan 2 baris "2026/2027". Audit ULANG (lebih lengkap dari audit 2026-08-20) menemukan
**15 tabel** (bukan 13) yang FK ke `academic_years` — ketinggalan `school_holidays` dan
`semesters` sendiri di audit pertama. Hasil cek SEMUA 15 tabel terhadap `academic_year_id=1`:
**14 tabel 0 baris**, hanya `activity_log` 3 baris (log administratif `academic_year.create`
id=1 → `system_live_config.update` → `academic_year.create` id=2, semuanya dalam ~4 menit
2026-08-12 04:10-04:14, `adminSU` — riwayat pembuatan+koreksi typo itu sendiri, bukan
operasional). Backup manual dijalankan+diverifikasi (`absensi_20260821_000303.sql.gz`, 3.5M)
SEBELUM eksekusi `DELETE FROM academic_years WHERE id = 1;`. Hasil: `id=1` hilang, `id=2`
tetap utuh (aktif). 3 baris `activity_log` TETAP ADA (bukan ikut terhapus, FK
`ON DELETE SET NULL` bukan CASCADE) — hanya `academic_year_id`-nya jadi NULL, action/actor/
target_id/created_at semuanya utuh, diverifikasi query setelah DELETE.

Bagian ini semula ditulis "TIDAK DIHAPUS" (lihat versi lama di bawah, dipertahankan sebagai
riwayat keputusan) — dibatalkan setelah audit lebih lengkap mengonfirmasi tidak ada data
operasional bergantung padanya.

---

**Versi lama (2026-08-20, sudah tidak berlaku)**:

- **JANGAN hapus lewat query manual langsung** — cek dulu APAKAH ADA data (Semester/JadwalSlot/dll) yang terhubung ke `academic_years.id=1` (`SELECT COUNT(*) FROM semesters WHERE academic_year_id = 1;` dan tabel lain yang FK ke `academicYearId`) — kalau 0 di semua, aman dihapus manual dengan `DELETE FROM academic_years WHERE id = 1;` (SETELAH backup, SETELAH verifikasi tidak ada FK terhubung).
- **REKOMENDASI LEBIH BAIK**: task TERPISAH untuk membangun endpoint DELETE AcademicYear di backend (belum ada sama sekali) — supaya ke depan admin bisa hapus Tahun Ajaran keliru lewat UI sendiri, TIDAK perlu lagi query manual manual tiap kali salah input. CATAT sebagai rekomendasi task susulan, TIDAK WAJIB dikerjakan bersamaan task ini.

## Edge Cases

- **Ada baris `attendance_records` dengan tanggal DI LUAR rentang Semester Ganjil** (misal tanggal setelah 31 Desember 2026, kalau ada) — JANGAN backfill paksa ke Semester Ganjil, biarkan `semesterId` tetap NULL untuk baris itu (akan otomatis tertag benar nanti kalau admin buat+aktifkan Semester Genap dan sistem insert data baru dengan tag yang benar — INI scope migrasi SUSULAN kalau terjadi, bukan task ini).
- **Row count attendance_records SANGAT BESAR** (18.943 baris) — UPDATE ini BISA lambat/lock tabel sebentar di production yang sedang live — PERTIMBANGKAN jalankan di luar jam sibuk sekolah, ATAU pastikan `UPDATE` pakai index (`academic_year_id`+`semester_id`+`tanggal` — cek index existing sebelum eksekusi supaya tidak full table scan).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kalender/semesters-section.tsx` (fix `formatTanggal`).
- **Buat:** migration SQL/script backfill (lokasi PUTUSKAN saat implementasi — REPLIKASI pola folder T161 kalau memang jadi Prisma migration resmi, atau script terpisah `apps/api/scripts/` kalau sifatnya one-off manual seperti T161 sebenarnya juga tersimpan sebagai migration folder).

## Acceptance Criteria
- [x] Halaman Kalender Pendidikan, section Semester — tanggal tampil BENAR (bukan "Invalid Date"). Plus 4 file lain dengan bug sama juga diperbaiki (ditemukan via grep pola serupa).
- [x] Backup production terverifikasi ADA sebelum migrasi backfill dijalankan (`absensi_20260820_235406.sql.gz`, 3.5M, diverifikasi `ls -lh`).
- [x] SEMUA tabel model transaksi T139 dicek ulang untuk baris NULL (bukan cuma `attendance_records`), angka dilaporkan LENGKAP sebelum migrasi (13 tabel, lihat STATUS.md untuk angka lengkap).
- [x] Setelah backfill — `attendance_records.semesterId IS NULL` untuk rentang tanggal Semester Ganjil = 0. (Berlaku juga 8 tabel lain yang punya baris NULL.)
- [x] Total row `attendance_records` TIDAK berubah sebelum/sesudah (hanya UPDATE field, bukan insert/delete). Diverifikasi di SEMUA 13 tabel, bukan cuma attendance_records.
- [x] Rekap Kehadiran Murid (admin) dengan filter Tahun Ajaran+Semester aktif — TIDAK LAGI 0 data (diverifikasi via query production langsung: 16.456 baris hadir ter-tag academic_year_id=2+semester_id=1).
- [x] Tahun Ajaran id=1 (duplikat salah input) — **UPDATE 2026-08-21**: DIHAPUS. Audit ulang lebih lengkap (15 tabel FK, bukan 13) mengonfirmasi 0 data operasional terhubung — hanya 3 baris `activity_log` (forensik, FK `ON DELETE SET NULL`, log tetap ada cuma tag academic_year_id jadi NULL). Backup diverifikasi sebelum DELETE. Endpoint DELETE AcademicYear resmi tetap belum dibuat — masih rekomendasi task terpisah untuk kasus serupa ke depan.

## Validasi Claudian
- [x] Konfirmasi migrasi backfill resolve ID Semester aktif SECARA DINAMIS (`WHERE is_active=TRUE` saat dieksekusi), BUKAN hardcode id literal — lihat `@target_semester_id`/`@batas_mulai`/`@batas_selesai` di migration.sql.
- [x] Konfirmasi backup dijalankan dan diverifikasi SEBELUM UPDATE apa pun, bukan sesudah — urutan dieksekusi persis: backup dulu → verifikasi file ada → baru jalankan migration.sql.
- [x] Konfirmasi migrasi HANYA backfill baris yang tanggalnya benar-benar masuk rentang semester target — pakai `BETWEEN @batas_mulai AND @batas_selesai` (bukan open-ended `>=` seperti T161, karena Semester punya tanggal_selesai definitif).
- [x] Konfirmasi row count SEMUA tabel model transaksi T139 dicek ulang dari NOL — query fresh langsung ke `absensi-mysql-prod` container, dikonfirmasi row count users=36/students=4099/teachers=156 cocok skala production asli sebelum lanjut ke tabel transaksi.
