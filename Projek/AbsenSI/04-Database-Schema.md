---
tags: [absensi, database, schema]
updated: 2026-07-21
---

# 04 — Database Schema

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> **Update 2026-06-26:** Keputusan struktur tabel inti & engine database sudah final lewat ADR-010 s/d ADR-014 (lihat [[Projek/AbsenSI/11-Decisions|11-Decisions]]). Skema di bawah merefleksikan keputusan itu. Masih ada Open Questions di level detail kolom (lihat bagian bawah), tapi kerangka dasarnya tidak lagi berubah.

---

## Entitas Inti

### `students`
- `id`, `nisn` (unique), `nama`, `kelas_id` (FK), `jurusan_id` (FK), `status` (aktif/nonaktif), `tanggal_lahir` (opsional)
- **T028 (2026-07-17):** tambahan biodata lengkap, semua opsional — `tempat_lahir`, `jenis_kelamin` (enum: `laki_laki`/`perempuan`), `agama` (enum: `islam`/`kristen`/`katolik`/`hindu`/`buddha`/`konghucu`), `alamat`, `rt_rw`, `nama_ayah`, `nama_ibu`, `foto` (path relatif ke file di disk, lihat ADR-023 — **bukan** BLOB di database)

### `teachers`
- `id`, `niy` (unique — **rename dari `nip`, T028 2026-07-17**, lihat catatan migrasi di bawah), `nama`, `status` (aktif/nonaktif)
- **T028:** tambahan `no_hp` (opsional), `foto` (path relatif, sama seperti `students.foto`, ADR-023)

> **Catatan migrasi (T028):** Rename `nip`→`niy` **wajib** pakai `RENAME COLUMN` manual di migration SQL, bukan drop+add — Prisma default generate drop+add yang menghapus seluruh data NIP existing. Field ini dulunya "NIP" (istilah PNS), diganti terminologi jadi "NIY" (Nomor Induk Yayasan, lebih umum untuk sekolah swasta) tapi tetap unique identifier tunggal per guru — bukan field tambahan di samping NIP.

> **Catatan desain (ADR-010):** `persons` tunggal resmi dipecah jadi `students` + `teachers` karena field keduanya cukup berbeda (siswa punya kelas/jurusan, guru punya jadwal mengajar). Semua tabel di bawah yang perlu relasi ke "siswa ATAU guru" memakai **pola dual-FK nullable** (`student_id` + `teacher_id`, tepat 1 yang terisi) — bukan polymorphic generik — supaya foreign key constraint asli MySQL tetap berlaku dan integritas data tidak 100% bergantung pada logic aplikasi.

### `cards`
- `id`, `uid` (unique), `student_id` (FK, nullable), `teacher_id` (FK, nullable), `status` (active/inactive), `issued_at`, `revoked_at`
- Constraint aplikasi: tepat satu dari `student_id`/`teacher_id` yang terisi, sisanya null.

### `schedules` (jadwal masuk sekolah + jadwal mengajar guru — generic untuk fase 1 & 2)
- `id`, `type` (jam_sekolah / jam_mengajar / jadwal_khusus), `teacher_id` (nullable, FK ke guru jika type=jam_mengajar), `kelas_id` (nullable), `mapel_id` (nullable, fase 2), `hari`, `jam_mulai`, `jam_selesai`, `threshold_terlambat_menit`, `tanggal_berlaku_mulai`, `tanggal_berlaku_selesai` (untuk dukung jadwal khusus ujian)

### `attendance_sessions` (generic — gerbang ATAU kelas, lihat catatan di absensi-kelas-mapel.md)
- `id`, `location_type` (gerbang/kelas — fase 1 cuma gerbang), `kelas_id` (nullable), `mapel_id` (nullable, fase 2)

### `attendance_records`
- `id`, `student_id` (FK, nullable), `teacher_id` (FK, nullable), `session_id` (FK, nullable di fase 1 kalau session tidak relevan), `tanggal`, `waktu_masuk`, `waktu_pulang` (nullable), `status` (hadir/terlambat/tidak_hadir/bolos — bolos baru relevan fase 2), `pulang_via` (nullable, enum: `tap` / `piket_izin` / `tap_izin_pulang` — lihat penjelasan di bagian Entitas Baru), `client_uuid` (unique, untuk idempotency offline-sync), `kiosk_id` (device asal tap)
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
- `students` tambah kolom **`late_strike_reset_at`** (nullable — T037, ADR-025) — timestamp reset counter keterlambatan untuk mekanisme lock otomatis 2x terlambat. Diisi `now()` oleh `unlock()` kalau lock yang dibuka adalah lock otomatis (dideteksi lewat `locked_by IS NULL`, karena lock manual selalu diisi piket yang login). `null` berarti hitung dari seluruh riwayat terlambat sejak awal.
- `attendance_records` tambah kolom **`pulang_via`** (nullable, enum: `tap` / `piket_izin` / `tap_izin_pulang`) — makna masing-masing nilai:
  - `tap` = tap keluar normal di gerbang, tanpa aksi piket khusus
  - `piket_izin` = piket catat keluar tanpa tap (izin keluar tidak kembali, atau siswa gagal tap dan piket manual input)
  - `tap_izin_pulang` = siswa tap di gerbang, lalu piket konfirmasi tap itu sebagai izin pulang awal resmi (tap ada, tapi konteksnya diubah piket)
  - `null` = siswa belum pulang / belum ada data pulang hari itu

### `users` (akun login — definisi formal pertama kali, sebelumnya cuma dibahas konsep role di [[Projek/AbsenSI/03-User-Roles|03-User-Roles]])
- `id`, `username`, `password_hash`, `role` (`super_admin` / `card_admin` / `guru` / `kepsek` / `guru_piket`), `teacher_id` (FK ke `teachers`, nullable — terisi untuk role `guru`/`guru_piket`/`kepsek` yang merupakan akun seorang guru), `kampus_id` (FK ke `kampus`, nullable — **hanya** terisi untuk role `guru_piket`, jadi scope akses dashboard-nya), `status` (aktif/nonaktif)

### `permits`
- `id`, `student_id` (FK), `jenis` (`tidak_masuk` | `keluar`), `alasan_kategori` (`sakit` | `izin`), `alasan_detail` (teks), `tanggal`, `jam_keluar` (nullable, hanya `jenis=keluar`), `jam_kembali_diharapkan` (nullable — kalau siswa diperkirakan kembali hari itu), `status_kembali` (`belum` / `sudah` / `pulang`) , `kembali_dikonfirmasi_at` (nullable), `kembali_dikonfirmasi_by` (FK ke `users`, nullable), `approved_by` (FK ke `users`), `kode_verifikasi` (unique, nullable — hanya digenerate untuk `jenis=keluar` yang dicetak), `surat_printed_at` (nullable), `created_at`

> **Catatan desain (2026-07-03):** Field `akan_kembali` (boolean) **dihapus** dari desain sebelumnya. Semua izin keluar (`jenis=keluar`) dianggap berpotensi kembali — piket tetap memantau via `status_kembali`. Kalau siswa tidak kembali sampai akhir hari, piket cukup ubah `status_kembali` jadi `pulang` (siswa dianggap pulang lebih awal sah, bukan kasus alarming). Nilai `tidak_relevan` dihapus dan diganti dengan enum yang lebih eksplisit: `belum` (belum kembali / belum dikonfirmasi), `sudah` (sudah kembali, piket klik konfirmasi), `pulang` (tidak kembali, dianggap pulang lebih awal).

**Constraint & logic aplikasi penting:**
- Setiap `permits` baru otomatis update `attendance_records` hari itu sesuai `jenis`:
  - `tidak_masuk` → status record jadi `izin`/`sakit`, tidak ada `waktu_pulang`
  - `keluar` → `attendance_records.pulang_via` diset `piket_izin`, `waktu_pulang` = `jam_keluar` (baru diset kalau siswa ternyata tidak kembali / `status_kembali` diubah jadi `pulang`)
- `status_kembali='belum'` + `jam_kembali_diharapkan` sudah lewat → muncul di daftar "Belum Kembali" di Dashboard Piket (sinyal visual untuk piket, bukan trigger otomatis apa pun — lihat ADR-017).
- Kewenangan membuat dan mengubah `permits` hanya role `guru_piket` — ditegakkan di level API guard (lihat ADR-019).

### `tap_events` (immutable — raw log setiap tap RFID)
- `id`, `uid` (UID kartu yang di-scan — raw string, bukan FK, supaya tap UID tidak dikenal pun tercatat), `card_id` (FK ke `cards`, nullable — null kalau UID tidak ditemukan), `kiosk_id` (FK ke `kiosks`, nullable — **T027: sekarang proper FK bertipe integer, bukan lagi string self-reported dari kiosk**), `scanned_at` (**timestamp server**, bukan client — clock kiosk tidak bisa dijadikan sumber kebenaran waktu), `result` (enum: `accepted` / `rejected_inactive` / `rejected_locked` / `rejected_unknown` / `rejected_duplicate` / `rejected_wrong_kiosk_type` — nilai terakhir ditambahkan T028, ADR-022), `attendance_record_id` (FK ke `attendance_records`, nullable — terisi kalau tap berhasil membuat/update record)

> **Insert-only.** Tidak ada endpoint DELETE atau UPDATE dari aplikasi. Ini adalah log forensik bukti tap fisik — berguna untuk kasus "siswa klaim sudah tap tapi tidak tercatat." Volume estimasi: ±5.000 baris/hari (2.500 siswa × 2 tap) = ±1,8 juta/tahun. Masuk scope ETL ke data warehouse (ADR-013). Lihat ADR-020.

### `activity_log` (immutable — audit trail aksi pengguna yang login)
- `id`, `actor_id` (FK ke `users`), `action` (string: misal `permit.create`, `permit.confirm_kembali`, `permit.set_pulang`, `student.lock`, `student.unlock`, `attendance.set_izin_pulang`), `target_type` (string: `permit` / `attendance_record` / `student` / `card` / ...), `target_id` (ID record yang diubah), `snapshot_before` (JSON, nullable — state sebelum diubah), `snapshot_after` (JSON, nullable — state sesudah diubah), `ip_address` (nullable), `created_at`

> **Insert-only.** Tidak ada endpoint DELETE atau UPDATE dari aplikasi. Setiap aksi yang mengubah data oleh pengguna yang login harus menghasilkan 1 baris di tabel ini. `snapshot_before`/`snapshot_after` disimpan sebagai JSON supaya perubahan bisa dibaca langsung dari log tanpa harus rekonstruksi dari histori. Lihat ADR-020.

## Entitas Kalender Pendidikan (Fase 1)

> Ditambahkan 2026-07-03. Dibutuhkan sebagai fondasi perhitungan alfa di modul Rekap. Lihat [[Projek/AbsenSI/06-Features/kalender-pendidikan|kalender-pendidikan.md]] untuk spek fitur lengkap.

### `academic_years`
- `id`, `nama` (misal "2025/2026"), `tanggal_mulai` (date), `tanggal_selesai` (date), `is_active` (boolean — hanya 1 yang true sekaligus, ditegakkan di level aplikasi), `created_by` (FK ke `users`), `created_at`

### `school_holidays`
- `id`, `academic_year_id` (FK ke `academic_years`, nullable — null untuk libur yang tidak terikat tahun ajaran spesifik), `tanggal_mulai` (date), `tanggal_selesai` (date — sama dengan `tanggal_mulai` untuk hari tunggal), `jenis` (enum: `libur_nasional` / `libur_semester` / `libur_sekolah` / `libur_mendadak`), `keterangan` (teks, misal "Hari Raya Idul Fitri"), `created_by` (FK ke `users`), `created_at`, `updated_by` (FK ke `users`, nullable), `updated_at`

**Tiga kategori hari — penting untuk rekap:**

| Hari | Kategori | Tap diterima? | Absen = Alfa? |
|---|---|---|---|
| Senin–Jumat (hari sekolah aktif) | **Wajib** | ✅ | ✅ Ya |
| Sabtu | **Opsional** | ✅ | ❌ Tidak |
| Minggu | **Libur** | ✅ (kiosk tidak dimatikan) | ❌ Tidak |
| Hari dalam `school_holidays` | **Libur** | ✅ | ❌ Tidak |

> **Sabtu adalah hari opsional** — siswa dan guru tetap bisa tap dan kehadirannya tercatat normal di `attendance_records`, tapi jika tidak hadir maka **tidak dihitung alfa**. Ini beda dari hari libur (yang memang tidak ada ekspektasi kehadiran sama sekali) dan beda dari hari wajib (yang jika tidak hadir = alfa).

**Logic "hari wajib" untuk query alfa:**
```sql
-- Hari wajib = hari yang jika siswa tidak hadir, dihitung alfa
hari_wajib = tanggal ∈ range academic_years (is_active = true)
             AND DAYOFWEEK(tanggal) BETWEEN 2 AND 6  -- Senin (2) s/d Jumat (6) saja
             AND tanggal TIDAK overlap dengan school_holidays
-- Sabtu (DAYOFWEEK = 7) dan Minggu (DAYOFWEEK = 1) dikecualikan dari hari wajib
```

**Catatan implementasi:** Tidak perlu field tambahan di skema — aturan "Senin–Jumat wajib, Sabtu opsional" dikode sebagai konstanta di service layer Rekap, bukan data yang bisa dikonfigurasi admin. Kalau suatu saat jadwal berubah (misal ada program Sabtu wajib), baru dipertimbangkan sebagai perubahan skema.

## Entitas Kiosk (T027 ADR-021, T028 ADR-022)

> Ditambahkan 2026-07-16 (T027) — token per-kiosk dari database + IP whitelist, menggantikan static env token. Tipe kiosk ditambahkan 2026-07-17 (T028).

### `kiosks`
- `id` (Int autoincrement — **bukan** `String cuid()` seperti draf awal spec, disesuaikan supaya konsisten dengan seluruh model lain di schema ini), `nama`, `kampus_id` (FK ke `kampus`), `device_token` (unique, random 256-bit via `nanoid(32)`), `allowed_ip` (IPv4, divalidasi di level DTO), `tipe` (enum: `siswa` / `guru` — **T028, ADR-022**, wajib diisi admin saat registrasi, TIDAK dideteksi otomatis dari kartu), `is_active` (boolean), `last_seen_at` (nullable, update fire-and-forget setiap tap sukses), `last_seen_ip` (nullable), `created_by` (FK ke `users`), `created_at`

**Auth berlapis (ADR-021):** token dari URL (`?device=TOKEN`, admin generate + tampilkan QR code) **DAN** IP harus cocok dengan `allowed_ip` — kombinasi keduanya, bukan salah satu saja. `KioskGuard` query `kiosks` berdasarkan token dari header `Authorization: Bearer`, lalu validasi IP request terhadap `allowed_ip`.

**Kartu salah tipe kiosk (ADR-022):** kartu siswa di-tap di kiosk `tipe=guru` (atau sebaliknya) ditolak dengan `tap_events.result=rejected_wrong_kiosk_type` — tetap tercatat (insert-only, forensik), tapi tidak membuat/ubah `attendance_records`.

**Status online/offline** (dipakai UI admin `/kiosk`, belum dikerjakan): `last_seen_at` dalam 5 menit terakhir → online, lebih dari itu → offline. Bukan kolom tersendiri, dihitung saat query seperti halnya alfa di modul Rekap.

## Entitas Jadwal Piket (T032, ADR-024)

> Ditambahkan 2026-07-21 (T032) — jadwal hari bertugas per akun `guru_piket`, dasar untuk enforcement read-only dashboard piket di hari non-tugas.

### `piket_schedules`
- `id`, `hari` (Int, **basis independen 1=Senin..6=Sabtu** — **BUKAN** basis DAYOFWEEK yang dipakai `schedules.hari`, sengaja beda karena grid admin cuma 6 kolom Senin-Sabtu tanpa Minggu), `user_id` (FK ke `users`, harus role `guru_piket`), `created_by` (FK ke `users`), `created_at`
- Constraint unique `(hari, user_id)` — satu guru piket tidak bisa di-assign dobel ke hari yang sama.

**Enforcement (bukan cuma UI):** `PiketOnDutyGuard` di backend cek `piket_schedules` sebelum semua endpoint tulis modul piket (lock/unlock siswa, buat/update permit, konfirmasi kembali, dst) — guru piket yang login di luar hari jadwalnya tetap bisa lihat data, tapi request tulis ditolak 403 walau di-bypass lewat API langsung, bukan cuma disembunyikan di UI.

---

## 🏗️ Catatan Infrastruktur (lihat [[Projek/AbsenSI/10-Environment|10-Environment]] untuk detail lengkap)
Database AbsenSI hidup sebagai 1 schema/database MySQL di server fisik bersama (bukan VM terpisah, ADR-012), disinkronkan berkala ke data warehouse pusat (ADR-013) untuk laporan lintas-aplikasi ekosistem sekolah. Modul Core (siswa/guru/jadwal) tetap di dalam AbsenSI untuk saat ini, belum diekstrak jadi servis terpisah (ADR-014).

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]]
- [[Projek/AbsenSI/06-Features/absensi-kelas-mapel|absensi-kelas-mapel.md]]
- [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]]
- [[Projek/AbsenSI/11-Decisions|11-Decisions]] — ADR-010 s/d ADR-025

