---
tags:
  - project
  - debug-log
created: 2026-06-04
updated: 2026-06-04
---

# Debug Log — E-Berita Acara Ujian

_Log sesi per tanggal: apa yang dikerjakan, bug ditemukan, keputusan teknis minor._

---

## 2026-06-04 — Inisialisasi Dokumentasi Obsidian

**Dikerjakan:**
- Claudian membaca seluruh codebase dari `/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/`
- Dibuat seluruh file dokumentasi proyek di `Brain/Projek/E-Berita-Acara-Ujian/`

**File yang Dibuat:**
- `_claudian/project-context.md`
- `_claudian/discussion-log.md`
- `00-INDEX.md` hingga `14-Debug-Log.md`
- `06-Features/_template.md`
- `06-Features/tasks/_task-template.md`

**Temuan dari Code Review Awal:**
- `ExamReportController.php` = 834 baris (God class) — kandidat refactor
- 7 file `.sql` backup di root project tidak di-gitignore
- Banyak public API endpoint tanpa auth (by design — lihat ADR-002)
- `frontend-tv` hardcode 2 kampus
- Inkonsistensi `nomor_peserta` vs `kode_peserta`
- `pengawas` bisa multi-record per NIY — query harus selalu by NIY bukan by ID

**Keputusan Teknis:**
- Semua keputusan arsitektur yang ditemukan dari kode sudah didokumentasikan di `11-Decisions.md` sebagai ADR-001 hingga ADR-007

---

## 2026-06-04 — Task-001: Setup & Fix ZipArchive (Claude Code)

**Dikerjakan:**
- Restart artisan serve → fix ZipArchive (PHP 8.4.11 dengan zip extension)
- Database diimport dari SQL backup → 26 tabel berhasil
- Storage link dibuat
- Script `start-dev.sh` dibuat di root project
- Port backend dipindah dari 8000 → **8002** (konflik DasiPelajar)
- Semua proxy di `vite.config.js` ketiga frontend diupdate ke 8002

**File yang Diubah:**
- `backend/.env` — DB_PASSWORD=casaos, APP_URL ke port 8002
- `frontend/vite.config.js` — proxy 8000 → 8002
- `frontend-admin/vite.config.js` — proxy 8000 → 8002
- `frontend-tv/vite.config.js` — proxy 8000 → 8002
- `start-dev.sh` — BARU, script startup semua service

**Keputusan Teknis:**
- Port backend: 8002 (DasiPelajar pakai 8000 dengan process manager yang auto-restart)
- MariaDB: Docker container `linuxserver/mariadb:11.4.8`, akses via `docker exec mariadb`

**Status:** ✅ Selesai — ZipArchive OK, health-check HTTP 200

---

## 2026-06-04 — Task-002: Fix Pivot Priority (Claude Code) + Temuan Masalah Lanjutan

**Dikerjakan:**
- `AssignmentService::getParticipants()`: pivot jadi PRIMARY, room+sesi jadi FALLBACK
- `ExamReportController::rekapAdmin()`: pivot jadi PRIMARY untuk filter per-tanggal
- Penghapusan `orWhereNull('sesi')` yang terlalu broad

**Temuan Kritis:**
- Pivot berisi data salah — 1 siswa masuk ke 5 jadwal berbeda (harusnya 1)
- Root cause: `importCsv()` tidak pakai `waktu_ujian` saat generate pivot → assignee semua jadwal di ruang+sesi
- Ini bug sistem, bukan kesalahan admin

---

## 2026-06-04 — Task-003: Fix Pivot Generation + Template Sesi (Claude Code)

**Dikerjakan:**
- `PesertaUjianController.importCsv()`: tambah `whereDate('mulai_ujian', waktu_ujian)` filter
- `PesertaUjianController.store()`: tambah `whereDate` filter
- `PesertaUjianController.update()`: tambah `whereDate` filter
- `downloadTemplate()`: label kolom F diperbaiki (`'Sesi (isi angka: 1, 2, atau 3)'`), contoh sesi diubah dari `'Sesi 1'` → `'1'`
- Pivot database diregenerasi ulang via SQL langsung

**Verifikasi:**
- Sebelum: Bengkel TKR setiap jadwal = 133 siswa (semua siswa ruangan)
- Sesudah: Bengkel TKR setiap jadwal = 12-14 siswa (hanya siswa terjadwal hari itu)
- Angga Setyawan (waktu_ujian=30 Mei): dari 5 jadwal → hanya 1 jadwal (30 Mei)

---

## 2026-06-04 — Task-004: Fitur "Hapus Semua" per Menu Import (Claude Code)

**Dikerjakan:**
- 5 route DELETE baru di `routes/api.php` (`/reset` per entitas)
- `resetByUjian()` method di: PesertaUjian, Pengawas, Panitia, JadwalUjian, Ruang
- Tombol "Hapus Semua" + `handleResetAll` di: Students.jsx, Proctors.jsx, Panitia.jsx, ExamSchedule.jsx, Rooms.jsx
- Dialog konfirmasi SweetAlert2 + checkbox wajib dicentang sebelum konfirmasi

**Bug self-found dan self-fixed:**
- `fetchData()` dipanggil tapi fungsinya bernama `fetchInitialData()` di 3 halaman (Proctors, Panitia, ExamSchedule)
- TypeError tersebut menimpa response 422 dari backend → pesan block tidak pernah tampil
- Fixed: ganti ke `fetchInitialData()`

---

---

## 2026-06-05 — Task-005: Fix Jadwal Pengawas + Rekap Tanggal + Template Sesi (Claude Code)

**Dikerjakan:**
- `JadwalUjianController::import()`: tambah `->where('ujian_id', $request->ujian_id)` pada lookup pengawas utama (baris 221) dan pengganti (baris 229) + pesan error diperjelas
- `JadwalUjianController::template()`: header kolom B → `'Sesi (isi angka: 1, 2, atau 3)'`, contoh → `'1'`
- `ExamReportController::rekapAdmin()` baris 501: fallback tanggal siswa absen → `($tanggal ? $tanggal : now()->toDateString())`
- `ExamSchedule.jsx` baris 104: tambah `if (editMode) return;` + `editMode` ke deps array
- `ManualAttendance.jsx` baris 256: hapus opsi "Semua Tanggal", ganti dengan disabled

**Masalah Data Ditemukan:**
- 349 jadwal (ujian_id=4: 288, ujian_id=5: 61) menyimpan `pengawas_id` dari ujian berbeda akibat bug `->first()` lama
- Keputusan: **abaikan** — data lama, tidak mempengaruhi ujian baru
- 12 pengawas unik di ujian_id=5 tidak punya record di ujian itu (semuanya dari ujian_id=4 atau 2)

---

## 2026-06-05 — Pivot Ujian_id=9 Tidak Lengkap (Claude Code)

**Gejala:** Filter ruang di Manual Kehadiran ujian_id=9 hanya tampilkan 8 dari 20 ruang. Siswa di 12 ruang lain (misal Mohammad Bintang Alfurqoon, ruang C1 sesi 1) tidak muncul di rekapitulasi.

**Root cause:** Ujian_id=9 dibuat pada 2026-06-05, setelah SQL regenerasi pivot dijalankan pada 2026-06-04. Akibatnya hanya 1.286 dari 17.268 pivot rows yang terisi — hanya peserta yang masuk via `importCsv` setelah regenerasi yang ter-cover.

**Penyelesaian:**
```sql
DELETE jp FROM jadwal_peserta jp JOIN jadwal_ujians j ON jp.jadwal_ujian_id = j.id WHERE j.ujian_id = 9;
-- Regenerasi ulang pivot untuk ujian_id=9 saja
INSERT INTO jadwal_peserta (jadwal_ujian_id, peserta_ujian_id)
SELECT j.id, p.id FROM peserta_ujians p
JOIN jadwal_ujians j ON j.ujian_id = p.ujian_id AND j.ruang_id = p.ruang_id
  AND (j.sesi = p.sesi OR p.sesi IS NULL) AND DATE(j.mulai_ujian) = DATE(p.waktu_ujian)
WHERE p.waktu_ujian IS NOT NULL AND p.ruang_id IS NOT NULL AND p.ujian_id = 9;
INSERT INTO jadwal_peserta (jadwal_ujian_id, peserta_ujian_id)
SELECT j.id, p.id FROM peserta_ujians p
JOIN jadwal_ujians j ON j.ujian_id = p.ujian_id AND j.ruang_id = p.ruang_id
  AND (j.sesi = p.sesi OR p.sesi IS NULL)
WHERE p.waktu_ujian IS NULL AND p.ruang_id IS NOT NULL AND p.ujian_id = 9;
```

**Catatan penting:** Pivot tidak otomatis terisi saat ujian baru dibuat. Admin harus pastikan semua peserta + jadwal sudah diimport SEBELUM ujian aktif berjalan.

---

---

## 2026-06-06 — Task-006/007/008: Fitur Monitoring Petugas Keliling + PDS (Claude Code)

**Task-006 Backend:**
- Migration: `panitia` + `is_pds`, `is_keliling`. `presensi_pesertas` + `scanned_by_panitia_id`, `scan_keterangan`
- `KelilingController` baru: `jadwalAktif`, `siswaBelumScan`, `simpanKeterangan`
- `PanitiaController`: `togglePds()` + `toggleKeliling()` (mutually exclusive validation)
- `ExamReportController`: PDS logic — `require_keterangan: true` saat is_pds tanpa keterangan
- `PresensiService`: simpan `scanned_by_panitia_id` + `scan_keterangan`
- `LaporanController`: `catatan_pelaksanaan_full` (computed, tidak ubah DB)
- Login response: `is_pds` + `is_keliling` otomatis dari model Panitia

**Masalah saat eksekusi task-006:**
- Migration B hang karena metadata lock (PID 4413 & 4414, 29 jam) → `KILL` + retry
- `laporan_ujians` tidak ada `jadwal_ujian_id` → adapted: lookup via `ujian_id + pengawas_id + mulai_ujian`
- `laporan_ujians.catatan_pelaksanaan` tidak ada → field sebenarnya `notes`

**Task-007 Frontend Keliling:**
- `frontend/src/Keliling.jsx` baru: 2 screen (pilih ruang + detail ruang)
- `App.jsx`: routing `is_keliling → userType='keliling' → render <Keliling>`
- Tidak ada React Router → SPA conditional rendering

**Task-008 Frontend PDS + Admin:**
- `App.jsx`: PDS modal (radio Terlambat/Device Bermasalah/Lainnya), atribusi amber di daftar siswa
- `Panitia.jsx` admin: toggle PDS + Keliling per baris
- `Reports.jsx`: pakai `catatan_pelaksanaan_full`

---

## 2026-06-06 — Task-009: Fix KelilingController (Claude Code)

**Bug yang difix:**
1. `simpanKeterangan()`: `keterangan` → `scan_keterangan` (kolom lama tidak ada di DB)
2. `simpanKeterangan()`: tambah `ruang_id => $jadwal->ruang_id` (presensi keliling butuh ruang)
3. `jadwalAktif()` + `siswaBelumScan()`: ganti `pengawas_pengganti_id ?? pengawas_id` → `whereIn` kedua ID agar status BA benar jika ada pengganti

**File:** `backend/app/Http/Controllers/KelilingController.php` — satu file, tiga method.

---

_[tambahkan entri baru di bawah sini setelah setiap sesi Claude Code]_
