---
tags: [absensi, database, schema]
updated: 2026-06-25
---

# 04 — Database Schema

← [[Projek/AbsenSI/00-INDEX|Index]]

> **Update 2026-06-26:** Keputusan struktur tabel inti & engine database sudah final lewat ADR-010 s/d ADR-014 (lihat [[Projek/AbsenSI/11-Decisions|11-Decisions]]). Skema di bawah merefleksikan keputusan itu. Masih ada Open Questions di level detail kolom (lihat bagian bawah), tapi kerangka dasarnya tidak lagi berubah.

---

## Entitas Inti

### `students`
- `id`, `nisn` (unique), `nama`, `kelas_id` (FK), `jurusan_id` (FK), `status` (aktif/nonaktif), `tanggal_lahir` (opsional)

### `teachers`
- `id`, `nip` (unique), `nama`, `status` (aktif/nonaktif)

> **Catatan desain (ADR-010):** `persons` tunggal resmi dipecah jadi `students` + `teachers` karena field keduanya cukup berbeda (siswa punya kelas/jurusan, guru punya jadwal mengajar). Semua tabel di bawah yang perlu relasi ke "siswa ATAU guru" memakai **pola dual-FK nullable** (`student_id` + `teacher_id`, tepat 1 yang terisi) — bukan polymorphic generik — supaya foreign key constraint asli MySQL tetap berlaku dan integritas data tidak 100% bergantung pada logic aplikasi.

### `cards`
- `id`, `uid` (unique), `student_id` (FK, nullable), `teacher_id` (FK, nullable), `status` (active/inactive), `issued_at`, `revoked_at`
- Constraint aplikasi: tepat satu dari `student_id`/`teacher_id` yang terisi, sisanya null.

### `schedules` (jadwal masuk sekolah + jadwal mengajar guru — generic untuk fase 1 & 2)
- `id`, `type` (jam_sekolah / jam_mengajar / jadwal_khusus), `teacher_id` (nullable, FK ke guru jika type=jam_mengajar), `kelas_id` (nullable), `mapel_id` (nullable, fase 2), `hari`, `jam_mulai`, `jam_selesai`, `threshold_terlambat_menit`, `tanggal_berlaku_mulai`, `tanggal_berlaku_selesai` (untuk dukung jadwal khusus ujian)

### `attendance_sessions` (generic — gerbang ATAU kelas, lihat catatan di absensi-kelas-mapel.md)
- `id`, `location_type` (gerbang/kelas — fase 1 cuma gerbang), `kelas_id` (nullable), `mapel_id` (nullable, fase 2)

### `attendance_records`
- `id`, `student_id` (FK, nullable), `teacher_id` (FK, nullable), `session_id` (FK, nullable di fase 1 kalau session tidak relevan), `tanggal`, `waktu_masuk`, `waktu_pulang` (nullable), `status` (hadir/terlambat/tidak_hadir/bolos — bolos baru relevan fase 2), `client_uuid` (unique, untuk idempotency offline-sync), `kiosk_id` (device asal tap)
- Constraint aplikasi: tepat satu dari `student_id`/`teacher_id` yang terisi, sama seperti `cards`.

## ✅ Open Questions yang Sudah Resolved
- [x] **`persons` 1 tabel gabungan atau terpisah?** → Resolved: `students` + `teachers` terpisah, relasi pakai dual-FK nullable. Lihat ADR-010.
- [x] **Strategi partitioning tabel `attendance_records`?** → Resolved: **tidak perlu**. Estimasi ±500rb baris/tahun terlalu kecil untuk butuh partitioning (bahkan 10 tahun = ±5 juta baris, masih nyaman untuk MySQL dengan index komposit yang tepat). Partitioning di skala ini dianggap premature optimization.
- [x] **Engine database** → Resolved: **MySQL** (bukan PostgreSQL seperti rencana awal di ADR-002). Lihat ADR-011.

## ❓ Open Questions yang Masih Terbuka
- [ ] Index komposit final untuk filter rekap (kelas, jurusan, tanggal, status) — desain detail kolom menyusul setelah volume data lebih jelas, tapi prinsipnya sudah disetujui (lihat catatan performa di [[Projek/AbsenSI/06-Features/dashboard-tv|dashboard-tv.md]])
- [ ] Query gabungan siswa+guru (misal laporan kehadiran semua orang dalam 1 tabel hasil) — perlu `UNION` atau view gabungan karena `students`/`teachers` terpisah, desain detail menyusul saat modul rekap dikerjakan

## Entitas Baru — Dashboard Piket (Fase 1b)

> Ditambahkan 2026-06-26 mengikuti ADR-015 s/d ADR-018. Lihat [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]] untuk spek fitur lengkap.

### `kampus`
- `id`, `nama` (misal "Kampus 1", "Kampus 2")

### Perubahan ke tabel yang sudah ada
- `kelas` tambah kolom **`kampus_id`** (FK ke `kampus`) — siswa mewarisi kampus lewat relasi ke kelasnya, tidak ada `kampus_id` duplikat di `students`.
- `students` tambah kolom untuk mekanisme lock (ADR-017): `locked_at` (nullable), `locked_reason` (nullable), `locked_by` (FK ke `users`, nullable), `unlocked_at` (nullable), `unlocked_by` (FK ke `users`, nullable), `unlock_note` (nullable).
- `attendance_records` tambah kolom **`pulang_via`** (`tap` | `izin_piket`) — supaya laporan bisa bedakan pulang normal via tap vs pulang lebih awal dengan izin sah dari piket.

### `users` (akun login — definisi formal pertama kali, sebelumnya cuma dibahas konsep role di [[Projek/AbsenSI/03-User-Roles|03-User-Roles]])
- `id`, `username`, `password_hash`, `role` (`super_admin` / `card_admin` / `guru` / `kepsek` / `guru_piket`), `teacher_id` (FK ke `teachers`, nullable — terisi untuk role `guru`/`guru_piket`/`kepsek` yang merupakan akun seorang guru), `kampus_id` (FK ke `kampus`, nullable — **hanya** terisi untuk role `guru_piket`, jadi scope akses dashboard-nya), `status` (aktif/nonaktif)

### `permits`
- `id`, `student_id` (FK), `jenis` (`tidak_masuk` | `keluar`), `alasan_kategori` (`sakit` | `izin`), `alasan_detail` (teks), `tanggal`, `jam_keluar` (nullable, hanya `jenis=keluar`), `akan_kembali` (boolean, hanya relevan `jenis=keluar`), `jam_kembali_diharapkan` (nullable), `status_kembali` (`belum` / `sudah` / `tidak_relevan`), `kembali_dikonfirmasi_at` (nullable), `kembali_dikonfirmasi_by` (FK ke `users`, nullable), `approved_by` (FK ke `users`), `kode_verifikasi` (unique, nullable — hanya digenerate untuk `jenis=keluar` yang dicetak), `surat_printed_at` (nullable), `created_at`

**Constraint & logic aplikasi penting:**
- Setiap `permits` baru otomatis update `attendance_records` hari itu sesuai `jenis` (lihat ADR-016 untuk aturan precedence vs tap).
- `status_kembali='belum'` + `jam_kembali_diharapkan` sudah lewat → muncul di daftar "Belum Kembali" di Dashboard Piket (bukan otomatis apa pun selain tampil di daftar tinjauan, lihat ADR-017).

## 🏗️ Catatan Infrastruktur (lihat [[Projek/AbsenSI/10-Environment|10-Environment]] untuk detail lengkap)
Database AbsenSI hidup sebagai 1 schema/database MySQL di server fisik bersama (bukan VM terpisah, ADR-012), disinkronkan berkala ke data warehouse pusat (ADR-013) untuk laporan lintas-aplikasi ekosistem sekolah. Modul Core (siswa/guru/jadwal) tetap di dalam AbsenSI untuk saat ini, belum diekstrak jadi servis terpisah (ADR-014).

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]]
- [[Projek/AbsenSI/06-Features/absensi-kelas-mapel|absensi-kelas-mapel.md]]
- [[Projek/AbsenSI/11-Decisions|11-Decisions]] — ADR-010 s/d ADR-014

