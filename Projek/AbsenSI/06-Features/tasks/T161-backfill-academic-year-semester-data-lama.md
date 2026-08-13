# T161 — Backfill `academicYearId`/`semesterId` untuk Data Kehadiran Sejak 13 Juli 2026

## Depends on
Tidak ada dependency teknis. Independen, PRIORITAS TINGGI (mengganggu kegunaan fitur rekap SEKARANG — user sudah pakai workaround manual "Semua Tahun Ajaran" sementara).

## Objective
Semua baris `AttendanceRecord` (dan tabel transaksi lain yang relevan, lihat poin cakupan) dengan tanggal kejadian **sejak 13 Juli 2026** dan `academicYearId`/`semesterId` masih `NULL` — diisi (backfill) retroaktif ke Tahun Ajaran 2026/2027 yang sekarang aktif, supaya filter rekap berbasis kolom (T140) mencakup SEMUA data operasional sejak sekolah mulai pakai AbsenSI, bukan cuma data yang tercipta setelah Tahun Ajaran pertama kali diaktifkan (12 Agustus 2026).

## Context — Insiden yang Memicu Task Ini (2026-08-12)

User melaporkan: rekap kehadiran untuk tanggal 10-12 Agustus 2026 tampil KOSONG di semua siswa, padahal data tap TERBUKTI ada (dikonfirmasi query DB langsung: 1376/1368/1330 baris `AttendanceRecord` untuk 3 tanggal itu). Root cause DIKONFIRMASI: filter Tahun Ajaran di halaman Rekap (T140) AUTO-SELECT ke Tahun Ajaran yang sedang `isActive` (`rekap-view.tsx:113-118`, `defaultAcademicYear = academicYears.find(a => a.isActive)`), dan backend melakukan **EXACT MATCH** `WHERE academic_year_id = X` (`attendance-report.service.ts`, `buildPeriodFilter()`) — TIDAK ADA fallback untuk data `NULL`.

**Data produksi dikonfirmasi** (query langsung 2026-08-12):
```
academic_year_id | semester_id | jumlah | dari       | sampai
NULL             | NULL        | 10227  | 2026-07-31 | 2026-08-12
2                | NULL        | 3      | 2026-08-12 | 2026-08-12
```
Tahun Ajaran 2026/2027 (id=2) baru dibuat+diaktifkan **12 Agustus 2026 pagi** (04:14:59). SEBELUM itu, sistem sudah beroperasi (siswa tap absen normal) sejak minimal 31 Juli — TAPI SELURUH data itu (10227 baris) tersimpan `NULL` karena memang belum ada Tahun Ajaran aktif saat itu (sesuai desain T139 yang sengaja tidak memaksa tebak — "keterbatasan diterima, bukan bug", TAPI keterbatasan ini sekarang benar-benar mengganggu operasional harian, sehingga user minta diperbaiki via backfill).

**Keputusan final user (2026-08-12)**:
1. Backfill HANYA untuk data **sejak 13 Juli 2026** (tanggal mulai Tahun Ajaran 2026/2027 yang tercatat di `AcademicYear.tanggalMulai`) — BUKAN semua data historis tanpa batas.
2. **Perhitungan ALFA TIDAK BOLEH TERPENGARUH oleh backfill ini** — alfa tetap mengikuti `SystemLiveConfig.liveSince` (T147, SUDAH di-set user ke **2026-08-17**) sebagai batas bawahnya, TERPISAH TOTAL dari keberadaan tag `academicYearId`/`semesterId`. Ini KRUSIAL: backfill task ini HANYA mengisi kolom tag (supaya filter REKAP KEHADIRAN bekerja), TIDAK MENGUBAH logic/hasil perhitungan Alfa sama sekali — mekanisme T147 (`resolveHariWajib()` filter tambahan `liveSince`) sudah SEPENUHNYA independen dari kolom `academicYearId`, JANGAN sampai backfill ini disalahartikan sebagai "sekarang semua data sejak 13 Juli otomatis dihitung alfa" — TIDAK, itu tetap ditahan sampai 17 Agustus oleh mekanisme T147 yang TERPISAH.

## Spec Detail

### 1. Pola implementasi — ikuti PERSIS preseden T144 (raw SQL migration, BUKAN script terpisah)

Riset mengonfirmasi: SATU-SATUNYA preseden backfill/UPDATE massal berbasis kondisi tanggal di proyek ini adalah migration T144 (`20260808045112_t144_auto_activate_academic_year/migration.sql`) — pola raw SQL langsung di file `migration.sql` baru, dengan variabel `SET @var = (SELECT ...)`, guard eksplisit, dan komentar panjang menjelaskan kondisi edge case. **IKUTI POLA INI PERSIS**, JANGAN buat script Node/TS terpisah di luar folder migrations (tidak ada preseden untuk itu di proyek ini).

Migration baru, struktur:
```sql
-- T161 — backfill academic_year_id+semester_id untuk data SEJAK 13 Juli 2026 (tanggal
-- mulai Tahun Ajaran 2026/2027) yang tersimpan NULL karena belum ada Tahun Ajaran aktif
-- saat kejadian itu terjadi (T139 sengaja tidak backfill otomatis, tapi keterbatasan ini
-- mengganggu fitur Rekap yang filter berbasis kolom, T140). Backfill ini HANYA mengisi tag,
-- TIDAK MEMPENGARUHI perhitungan Alfa (tetap dibatasi terpisah oleh SystemLiveConfig.liveSince, T147).

SET @target_academic_year_id = (SELECT id FROM academic_years WHERE is_active = TRUE LIMIT 1);
SET @target_semester_id = (SELECT id FROM semesters WHERE is_active = TRUE LIMIT 1);

-- Kalau tidak ada Tahun Ajaran aktif SAAT migration dijalankan — TIDAK BISA backfill,
-- UPDATE di bawah otomatis no-op (WHERE ... AND @target_academic_year_id IS NOT NULL).
-- Semester BOLEH null (backfill academicYearId saja kalau semester belum aktif/ada).

UPDATE attendance_records
SET academic_year_id = @target_academic_year_id,
    semester_id = @target_semester_id
WHERE tanggal >= '2026-07-13'
  AND academic_year_id IS NULL
  AND @target_academic_year_id IS NOT NULL;
```
- **Ulangi UPDATE serupa untuk SEMUA 12 tabel yang T139 tambahkan kolomnya** (lihat "Cakupan Tabel" di bawah) — masing-masing dengan kolom tanggal kejadian yang SESUAI tabel itu (BUKAN semua pakai kolom `tanggal` — beberapa tabel pakai `created_at`/kolom lain, VERIFIKASI PERSIS nama kolom tanggal tiap tabel sebelum menulis UPDATE-nya, JANGAN asumsi seragam).
- **Tanggal batas `'2026-07-13'` HARUS diambil DINAMIS dari `AcademicYear.tanggalMulai` yang sedang aktif**, BUKAN di-hardcode sebagai string literal kalau memungkinkan secara SQL (`SET @batas_tanggal = (SELECT tanggal_mulai FROM academic_years WHERE is_active = TRUE LIMIT 1);` lalu pakai `@batas_tanggal` di WHERE clause) — supaya migration ini TETAP BENAR walau dijalankan di environment lain (dev, atau kalau production di-restore dari kondisi berbeda) tanpa perlu edit manual tanggalnya. KALAU secara teknis SQL raw sulit melakukan ini dengan bersih untuk SEMUA 12 tabel sekaligus, boleh hardcode literal `'2026-07-13'` TAPI WAJIB beri komentar jelas di migration bahwa ini spesifik untuk kondisi data production SAAT INI (bukan general-purpose), dan TIDAK relevan/tidak akan running ulang salah di environment lain (karena kondisi `academic_year_id IS NULL` sudah pasti false setelah pertama kali jalan).

### 2. Cakupan Tabel (12 tabel dari T139, VERIFIKASI kolom tanggal masing-masing sebelum implementasi)

Daftar 12 tabel yang T139 tambahkan `academic_year_id`/`semester_id` (dari migration `20260807210830_t139_tag_academic_period_transaksi`): `activity_log`, `attendance_records`, `class_attendance_marks`, `ekstra_absen`, `ekstra_sesi`, `journal_entries`, `late_entry_slips`, `permits`, `piket_journal_entries`, `student_pkl`, `tap_events`, `teacher_permits`, `teaching_sessions`.

- **PRIORITAS UTAMA (WAJIB)**: `attendance_records` (kolom `tanggal`) — ini yang LANGSUNG menyebabkan bug rekap yang dilaporkan user.
- **PERTIMBANGKAN JUGA** (untuk konsistensi menyeluruh, supaya fitur LAIN yang mungkin nanti filter by `academicYearId` di tabel-tabel ini juga tidak kena masalah sama): `permits`, `tap_events`, `ekstra_absen`, `ekstra_sesi`, `teaching_sessions`, `piket_journal_entries`, `teacher_permits`, `late_entry_slips`, `student_pkl`, `journal_entries`, `class_attendance_marks`.
- **`activity_log`** — PERTIMBANGKAN SECARA TERPISAH: tabel ini INSERT-ONLY forensik, backfill di sini MURNI kosmetik (tidak ada fitur yang filter rekap berdasarkan `academicYearId` di activity log SAAT INI, sejauh diketahui) — BOLEH dimasukkan untuk konsistensi menyeluruh TAPI TIDAK PRIORITAS TINGGI seperti `attendance_records`. Putuskan cakupan final saat implementasi berdasarkan APAKAH ada fitur LAIN yang nyata membutuhkannya sekarang (kalau ragu, LEBIH AMAN backfill semua 12 sekalian dalam 1 migration, karena marginal cost-nya kecil dan mencegah bug serupa muncul lagi di tabel lain nanti).
- **VERIFIKASI kolom tanggal PERSIS per tabel** sebelum menulis UPDATE — JANGAN asumsi semua pakai nama kolom `tanggal`. Baca `schema.prisma` untuk tiap model, konfirmasi field tanggal kejadian yang benar (misal `TeachingSession` mungkin pakai `tanggal`, `TapEvent` mungkin pakai `scannedAt`, dst — MASING-MASING WAJIB DICEK, JANGAN COPY-PASTE WHERE clause yang sama untuk semua tabel tanpa verifikasi).

### 3. VERIFIKASI EKSPLISIT — backfill ini TIDAK mempengaruhi Alfa

Setelah migration dijalankan, WAJIB verifikasi manual (bagian dari proses eksekusi task ini, bukan opsional):
1. Jalankan rekap untuk tanggal SEBELUM `liveSince` (17 Agustus 2026) — misal 1 Agustus 2026 — konfirmasi kolom Alfa TETAP 0 untuk SEMUA siswa (TIDAK BERUBAH oleh backfill ini, TETAP ditahan oleh filter `liveSince` T147 yang independen).
2. Jalankan rekap untuk tanggal SETELAH backfill (misal 10-12 Agustus 2026, yang jadi laporan awal insiden) DENGAN filter Tahun Ajaran 2026/2027 dipilih SPESIFIK (bukan "Semua Tahun Ajaran") — konfirmasi data SEKARANG MUNCUL (Hadir/Izin/Sakit terisi benar), Alfa TETAP 0 (karena masih sebelum `liveSince`).
3. Jalankan rekap untuk tanggal SETELAH `liveSince` (misal nanti setelah 17 Agustus, KALAU sudah lewat tanggal itu saat task dieksekusi) — konfirmasi Alfa MULAI terhitung normal untuk siswa yang benar-benar tidak hadir.

## Edge Cases
- Migration dijalankan LEBIH DARI SEKALI (misal re-deploy) — Prisma migration TIDAK akan menjalankan ulang migration yang sudah tercatat "applied" di `_prisma_migrations` (perilaku standar Prisma) — TIDAK PERLU idempotency tambahan di SQL-nya sendiri, TAPI tetap baik practice menambahkan `AND academic_year_id IS NULL` di WHERE (SUDAH ada di contoh poin 1) supaya AMAN kalau prosesnya somehow re-triggered manual.
- Data `attendance_records` dengan tanggal SEBELUM 13 Juli 2026 (kalau ada, misal dari testing/percobaan awal sebelum go-live resmi) — SENGAJA TIDAK di-backfill (tetap NULL selamanya) — sesuai keputusan eksplisit user (batas 13 Juli, bukan semua histori).

## Files
- **Buat:** migration Prisma baru (raw SQL, pola T144).
- **Jangan sentuh:** `resolveHariWajib()`/logic Alfa (T144/T147) — TIDAK diubah sama sekali, backfill ini murni isi kolom tag, alfa tetap dikontrol mekanisme terpisah yang sudah ada.

## Acceptance Criteria
- [ ] Migration berhasil dijalankan di dev DAN production, tanpa error.
- [ ] `SELECT COUNT(*) FROM attendance_records WHERE tanggal >= '2026-07-13' AND academic_year_id IS NULL` = 0 SETELAH migration (semua data sejak 13 Juli ter-backfill).
- [ ] Data SEBELUM 13 Juli (kalau ada) TETAP `academic_year_id IS NULL` (tidak ikut ter-backfill, sesuai batas yang diminta).
- [ ] Rekap dengan filter Tahun Ajaran SPESIFIK (bukan "Semua") untuk tanggal 10-12 Agustus 2026 SEKARANG menampilkan data benar (bukan kosong lagi).
- [ ] Kolom Alfa TIDAK BERUBAH sama sekali dibanding sebelum backfill — VERIFIKASI EKSPLISIT dengan membandingkan hasil rekap Alfa sebelum vs sesudah migration untuk rentang tanggal yang SAMA (harus identik, backfill ini TIDAK BOLEH mengubah angka Alfa).
- [ ] Row count SEMUA tabel yang di-backfill TIDAK BERUBAH (backfill murni UPDATE kolom, bukan INSERT/DELETE — verifikasi row count sebelum=sesudah).

## Validasi Claudian
- [ ] **WAJIB baca memory `feedback_insiden_database_wipe_2026-07-30`** SEBELUM eksekusi — operasi UPDATE massal ke database production perlu SANGAT hati-hati, backup dulu, verifikasi row count sebelum+sesudah, JANGAN asumsikan aman.
- [ ] **JANGAN** ubah logic `resolveHariWajib()`/perhitungan Alfa — task ini MURNI backfill kolom tag, terpisah total dari mekanisme Alfa yang sudah benar (T144/T147).
- [ ] **VERIFIKASI EKSPLISIT poin 3 di atas** SEBELUM melaporkan task selesai — perbandingan Alfa sebelum/sesudah backfill WAJIB dilakukan dan dilaporkan hasilnya, bukan diasumsikan aman.
- [ ] Cek nama kolom tanggal PERSIS per tabel (JANGAN asumsi seragam `tanggal` untuk semua 12 tabel) sebelum menulis UPDATE masing-masing.
- [ ] Backup database production SEBELUM menjalankan migration ini (manual, terpisah dari backup otomatis harian) — konsisten praktik yang sudah dilakukan untuk deploy T136-T152 sebelumnya.
