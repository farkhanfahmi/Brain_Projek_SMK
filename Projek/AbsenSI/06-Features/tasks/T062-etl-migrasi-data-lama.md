# T062 — ETL Script: Migrasi Data dari Database Lama ke AbsenSI

## Depends on
T061 (schema tambahan biodata guru), T063 (schema data wali murid), T002 (schema dasar Fase 1) — semua schema tambahan HARUS selesai sebelum ETL dimulai

> **Task ini TIDAK BOLEH dieksekusi langsung ke database produksi/dev tanpa dry-run dulu.** Baca seluruh spec, jalankan dulu dengan mode `--dry-run` (lihat di bawah) dan review outputnya bersama user sebelum insert nyata.

## Objective
Buat script ETL (Extract-Transform-Load) yang membaca dump SQL lama (`/media/anunnaki/DataNvme/sql_absensi_smk.sql`), transformasi struktur data (kombinasi jenjang+jurusan+kelas jadi 1 nama kelas, mapping ID varchar→Int, dll), dan load ke database AbsenSI baru lewat Prisma Client — bukan raw SQL import, supaya semua validasi/relasi Prisma tetap terjaga.

## Context
- **App:** `apps/api` (script terpisah, bukan bagian dari NestJS app runtime)
- **Ref:** `Projek/AbsenSI/06-Features/migrasi-database-lama.md` — baca SELURUH dokumen sebelum mulai, terutama bagian "Perbandingan dengan Skema Prisma Baru" dan "Keputusan Final"
- **Sumber data:** `/media/anunnaki/DataNvme/sql_absensi_smk.sql` (dump lama, 65MB — JANGAN load seluruh file ke memory sekaligus kalau bisa dihindari, proses per-tabel/streaming kalau volume besar)

## Spec Detail

### Langkah 0 — Setup Database Sumber Sementara
Import dump lama ke database MySQL SEMENTARA terpisah (misal `absensi_lama_readonly`) — JANGAN pernah tulis ke database ini, murni untuk dibaca script ETL via query biasa (lebih mudah daripada parsing SQL dump manual di Node.js).
```bash
mysql -u root -p -e "CREATE DATABASE absensi_lama_readonly"
mysql -u root -p absensi_lama_readonly < /media/anunnaki/DataNvme/sql_absensi_smk.sql
```

### Langkah 1 — Cek Data Sebelum Transform (Wajib, Laporkan ke User)
Jalankan query berikut dan LAPORKAN hasilnya ke user SEBELUM lanjut ke transform (beberapa keputusan ETL bergantung ke hasil ini):
```sql
-- Nilai unik role di users lama (untuk mapping ke UserRole enum baru)
SELECT DISTINCT role FROM absensi_lama_readonly.users;

-- Cek presensi_pegawais yang smartcard-nya tidak valid (data yatim, karena FK constraint tidak ada di tabel lama)
SELECT COUNT(*) FROM absensi_lama_readonly.presensi_pegawais pp
LEFT JOIN absensi_lama_readonly.smartcards s ON pp.id_smartcard = s.id
WHERE s.id IS NULL;

-- Kombinasi jenjang+jurusan+kelas yang benar-benar dipakai siswa (untuk generate Kelas baru)
SELECT DISTINCT id_jenjang, id_jurusan, id_kelas, COUNT(*) as jumlah_siswa
FROM absensi_lama_readonly.siswas
GROUP BY id_jenjang, id_jurusan, id_kelas;
```

### Langkah 2 — Transform: Generate `Kelas` dari Kombinasi 3 Tabel
- Untuk tiap kombinasi unik `(id_jenjang, id_jurusan, id_kelas)` yang benar-benar dipakai siswa (dari query Langkah 1) → buat 1 baris `Kelas` baru
- Nama kelas: format `{jenjangs.nama_jenjang}-{jurusans.jurusan}-{kelas.kelas}` (misal "X-TJKT-1") — **konfirmasi format penamaan persis ke user sebelum insert massal**, ini keputusan kosmetik yang mudah salah tafsir
- Semua `Kelas` baru di-assign `kampusId` = ID "Kampus 1" (dari T061)
- `jurusanId` — mapping `jurusans.id` lama ke `Jurusan.id` baru (buat `Jurusan` baru dulu untuk tiap `jurusans` lama yang belum ada di skema baru)
- **Field `jenjangs` (tingkat X/XI/XII) TIDAK punya tabel padanan di skema baru** — informasi tingkat jadi bagian dari NAMA kelas saja (string), tidak ada kolom `tingkat` terpisah di `Kelas` Prisma. Ini keputusan implisit dari skema existing (bukan keputusan baru task ini) — kalau ternyata dibutuhkan kolom tingkat terpisah nanti, itu perubahan skema di luar scope task ini

### Langkah 3 — Transform: `pegawais` → `Teacher`
Mapping field:
```
pegawais.niy               → teachers.niy
pegawais.nama_pegawai       → teachers.nama
pegawais.gelar_depan        → teachers.gelarDepan
pegawais.gelar_belakang     → teachers.gelarBelakang
pegawais.no_wa              → teachers.noHp        // digabung sesuai keputusan T061
pegawais.tempat_lahir       → teachers.tempatLahir
pegawais.tanggal_lahir      → teachers.tanggalLahir
pegawais.jenis_kelamin      → teachers.jenisKelamin (map "Laki-laki"→laki_laki, "Perempuan"→perempuan)
pegawais.agama              → teachers.agama (map "Islam"→islam, "Kristen Katolik"→katolik, "Kristen Protestan"→kristen, "Hindu"→hindu, "Budha"→buddha)
pegawais.alamat             → teachers.alamat
pegawais.status_pernikahan  → teachers.statusPernikahan (map "Menikah"→menikah, "Belum Menikah"→belum_menikah, "Pernah Menikah"→pernah_menikah)
pegawais.status_kepegawaian → teachers.statusKepegawaian (map "Guru"→guru, "Karyawan"→karyawan)
pegawais.status_pekerjaan   → teachers.status (map "Aktif"→aktif, "Nonaktif"→nonaktif, "Cuti"→??? — TANYAKAN ke user, PersonStatus cuma punya aktif/nonaktif, "Cuti" perlu dipetakan ke salah satu, rekomendasi: aktif dengan catatan, tapi konfirmasi dulu)
pegawais.photo              → teachers.foto (path relatif — cek dulu apakah file fisiknya perlu disalin juga ke direktori foto baru, atau cukup path-nya saja kalau strukturnya sama)
```
- **WAJIB simpan mapping `pegawais.id` (varchar UUID lama) → `teachers.id` (Int baru)** di tabel/file terpisah selama proses (misal in-memory `Map` kalau volume kecil, atau tabel sementara `migration_id_mapping` kalau mau disimpan permanen sebagai audit trail sesuai rekomendasi di `migrasi-database-lama.md`)

### Langkah 4 — Transform: `siswas` → `Student`
Mapping field serupa Langkah 3, tambahan:
```
siswas.nisn                → students.nisn
siswas.nama                → students.nama
siswas.status               → students.status (map "Belum Lulus"→aktif, "Lulus"→nonaktif, "Mengundurkan Diri"→nonaktif)
siswas.status               → students.alasanNonaktif (map "Lulus"→lulus, "Mengundurkan Diri"→mengundurkan_diri, "Belum Lulus"→null)
siswas.tahun_lulus          → students.tahunLulus (null kalau 0 — cek data lama, tahun_lulus=0 kemungkinan berarti "belum lulus", JANGAN masukkan sebagai tahun 0 literal)
(id_jenjang, id_jurusan, id_kelas) → students.kelasId (lookup ke Kelas baru hasil Langkah 2)
```
- **Data wali murid** (`nama_ayah`→`namaAyah`, `nama_ibu`→`namaIbu`, `rtrw`→`rtRw` — sudah ada di skema dari T028; `no_hp_ayah`→`noHpAyah`, `no_hp_ibu`→`noHpIbu`, `no_hp_siswa`→`noHpSiswa` — baru ditambahkan T063) — **keputusan final: SEMUA dimigrasikan**, tidak ada yang diabaikan

### Langkah 5 — Transform: Kartu (`smartcards`+`smartcard_siswas` → `cards`)
- `smartcards` (pegawai) → `cards` dengan `teacherId` terisi (lookup dari mapping Langkah 3), `studentId` null
- `smartcard_siswas` → `cards` dengan `studentId` terisi (lookup dari mapping Langkah 4), `teacherId` null
- `uid` — **dikonfirmasi user (2026-07-22): `smartcards.id`/`smartcard_siswas.id` ADALAH UID fisik kartu RFID** (bisa dibaca langsung dari kartu saat tap) — mapping langsung `smartcards.id`/`smartcard_siswas.id` → `cards.uid`, kartu fisik lama tetap bisa dipakai tanpa registrasi ulang
- `status` → semua kartu hasil migrasi di-set `active` (asumsi kartu lama masih dipakai fisik, kecuali user bilang lain)

### Langkah 6 — Transform: Presensi (`presensi_pegawais`+`presensi_siswas` → `attendance_records`)
- **Ini volume data PALING BESAR** (kemungkinan ratusan ribu baris) — proses secara batch/chunked (misal 1000 baris per batch), JANGAN load semua ke memory sekaligus
- `datang`/`pulang` → `waktuMasuk`/`waktuPulang`
- `pulangVia` → semua di-set `tap` (asumsi wajar karena data lama tidak punya info cara pulang lain — sesuai catatan di `migrasi-database-lama.md`)
- `status` → hitung berdasarkan ada/tidaknya `waktu_masuk` dan bandingkan dengan jadwal (KOMPLEKS — kemungkinan besar cukup diisi `hadir` untuk semua yang punya `waktu_masuk`, karena data lama tidak punya informasi "terlambat" secara eksplisit; JANGAN coba hitung ulang "terlambat" dari histori kecuali user minta, cukup default `hadir`)
- **Baris `presensi_pegawais` yang yatim** (smartcard tidak valid, dari query Langkah 1) → SKIP, catat jumlahnya di laporan akhir, JANGAN paksa insert dengan `cardId` null yang menyesatkan
- `kioskId` → **tidak ada padanan** di data lama (data lama tidak punya konsep kiosk/device) → set `null`

### Langkah 7 — Mode Eksekusi
Script HARUS mendukung 2 mode:
- **`--dry-run`** (default) — jalankan semua transform, tapi TIDAK insert ke database tujuan, cukup print ringkasan: berapa baris per tabel yang akan di-generate, berapa yang di-skip (dan alasannya), berapa error/data tidak valid ditemukan
- **`--execute`** — baru benar-benar insert, HANYA dijalankan setelah user review hasil `--dry-run` dan approve

## JANGAN
- ❌ JANGAN jalankan `--execute` tanpa `--dry-run` dulu dan review bersama user — volume data besar (ribuan siswa, ratusan ribu presensi), kesalahan mapping yang tidak ketahuan di awal akan sangat mahal diperbaiki setelah insert
- ❌ JANGAN import lewat raw SQL (`mysql ... < dump.sql` langsung ke database AbsenSI baru) — struktur tabel sudah beda total (PK varchar vs Int, nama kolom beda, kombinasi tabel beda), WAJIB lewat Prisma Client supaya semua constraint/relasi tervalidasi
- ❌ JANGAN coba hitung ulang status "terlambat" dari data historis lama — data lama tidak menyimpan info itu secara eksplisit, jangan reka-reka logic baru untuk mengisi kekosongan itu
- ❌ JANGAN import kartu hasil migrasi dengan `uid` yang di-generate/di-hash ulang — pakai persis nilai `smartcards.id`/`smartcard_siswas.id` apa adanya sebagai `cards.uid`, karena itu memang UID fisik yang dibaca reader RFID

## Files
- **Buat:** `apps/api/scripts/migrate-legacy-data.ts` (atau struktur folder terpisah `apps/api/scripts/legacy-migration/` kalau script-nya besar, dipecah per langkah)
- **Buat:** `apps/api/scripts/legacy-migration/README.md` — dokumentasi cara jalan script ini, urutan langkah, cara rollback kalau ada masalah

## Acceptance Criteria
- [ ] `--dry-run` menghasilkan laporan lengkap: jumlah baris per tabel tujuan, jumlah yang di-skip beserta alasan, tidak ada error tak tertangani
- [ ] Laporan dry-run mencakup: distribusi role lama yang belum terpetakan (kalau ada nilai role yang tidak dikenali), jumlah presensi_pegawais yatim yang di-skip, daftar kombinasi jenjang+jurusan+kelas yang akan jadi Kelas baru
- [ ] `--execute` HANYA jalan setelah ada persetujuan eksplisit tertulis dari user (bukan otomatis lanjut dari dry-run)
- [ ] Setelah `--execute`, jumlah baris di tabel tujuan (Student, Teacher, Card, AttendanceRecord) cocok dengan angka yang dilaporkan dry-run (tidak ada silent difference)
- [ ] Tabel/file mapping ID lama→baru tersimpan dan bisa diquery ulang (untuk audit "siapa siswa lama dengan UUID X jadi siapa di sistem baru")

## Handoff
Setelah migrasi berhasil, JANGAN hapus dump SQL lama atau database `absensi_lama_readonly` — simpan sebagai arsip/rollback sampai user eksplisit konfirmasi migrasi sudah stabil dan tidak dibutuhkan lagi.
