---
tags: [absensi, database, migrasi]
status: uji-coba-selesai-cutover-pending
updated: 2026-08-31
---

# Feature — Migrasi Data dari Database Lama (Laravel/Spatie)

← Index (00-INDEX AbsenSI.md)

> AbsenSI adalah rebuild dari aplikasi absensi lama (Laravel + Spatie Permission, MySQL). File dump lama: `/media/anunnaki/DataNvme/sql_absensi_smk.sql` (65MB, ~359rb baris, 22 tabel). Dokumen ini membandingkan struktur lama vs skema Prisma baru, dan mencatat keputusan migrasi yang sudah dan belum diambil.
>
> **[2026-08-31] STATUS TERKINI — PENTING, beda dari kesan dokumen di bawah:** ETL migrasi (T062) **SUDAH DIEKSEKUSI SUKSES 2026-07-23** — 348.391 `attendance_records`, 4.090 siswa, 159 guru, 76 kelas berhasil masuk ke AbsenSI. **TAPI ini BUKAN cutover final** — sistem lama (Laravel, server `10.10.10.100`) masih berjalan live saat migrasi dijalankan, jadi data ini untuk keperluan **uji coba** dulu. User akan berikan dump database TERBARU dari sistem lama untuk sinkronisasi ulang setelah AbsenSI dinyatakan siap — itu baru cutover resmi. **Jangan asumsikan data siswa/guru/attendance saat ini final** sampai cutover benar-benar terjadi. Script ETL: `apps/api/scripts/legacy-migration/`, hasil id-mapping tersimpan sebagai `id-mapping-*.json` untuk referensi delta-sync nanti.

---

## 🗄️ Struktur Database Lama (Referensi)

### Tabel Inti (relevan untuk migrasi)
| Tabel | Isi | PK | FK Keluar |
|---|---|---|---|
| `siswas` | Biodata siswa lengkap | `id` varchar(255) (UUID) | `id_jenjang`→jenjangs, `id_jurusan`→jurusans, `id_kelas`→kelas |
| `pegawais` | Biodata guru+karyawan (1 tabel gabungan) | `id` varchar(255) | — |
| `jenjangs` | **Tingkat kelas** (X/XI/XII) — BUKAN jenjang sekolah SMA/SMK seperti nama menyesatkan | `id` int | — |
| `jurusans` | Jurusan (LAKES, DKV, TJKT, TKR, TM, TPFL, TSM, TO, GAMEDEV) | `id` int | — |
| `kelas` | **Angka rombel** (1-10) — bukan nama kelas lengkap | `id` int | — |
| `smartcards` | Kartu RFID pegawai | `id` varchar | `id_pegawai`→pegawais (index saja, bukan FK constraint) |
| `smartcard_siswas` | Kartu RFID siswa | `id` varchar | `id_siswa`→siswas (FK constraint asli) |
| `presensi_pegawais` | Tap absensi pegawai | `id` varchar | `id_smartcard` (index saja, **BUKAN FK constraint** — celah data lama) |
| `presensi_siswas` | Tap absensi siswa | `id` varchar | `id_smartcard`→smartcard_siswas (FK constraint asli) |
| `users` | Akun login (role sebagai string bebas + Spatie RBAC penuh) | `id` bigint | `id_pegawai`→pegawais |
| `presensi_*_v` | 3 VIEW SQL gabungan untuk laporan (tidak perlu dimigrasikan — cukup query Prisma on-demand) | — | — |

### Nama Kelas Siswa = Kombinasi 3 Tabel (Temuan Penting)
Skema lama **tidak punya 1 kolom "nama kelas lengkap"** — nama kelas siswa (misal "X TJKT 1") adalah gabungan runtime dari `jenjangs.nama_jenjang` (X) + `jurusans.jurusan` (TJKT) + `kelas.kelas` (1, angka rombel). Skema baru AbsenSI (`Kelas.nama`) mengasumsikan **1 string tunggal** per kelas (misal "XI-TKJ-1") — jadi migrasi harus **menggabungkan 3 tabel lama jadi 1 baris `kelas` baru per kombinasi unik** (jenjang × jurusan × angka rombel yang benar-benar dipakai siswa aktif), bukan migrasi 1:1 antar tabel.

### Data Aktual (dicek langsung dari dump, bukan asumsi)
- `pegawais`: **114 Guru, 44 Karyawan** (28% non-guru — signifikan, tidak bisa diabaikan)
- `pegawais.status_pernikahan`: 97 Menikah, 58 Belum Menikah, 4 Pernah Menikah
- `siswas`: **2.488 Belum Lulus, 1.603 Lulus, 5 Mengundurkan Diri** (total ±4.096 baris)
- `pegawais.status_pekerjaan`: 157 Aktif, 1 Cuti, 1 Nonaktif

---

## 🔍 Perbandingan dengan Skema Prisma Baru

### ✅ Sudah Sesuai / Lebih Baik di Skema Baru
- `cards` (unified siswa+guru dual-FK) menggantikan `smartcards`+`smartcard_siswas` — lebih rapi, sesuai ADR-010
- `attendance_records` (unified) menggantikan `presensi_pegawais`+`presensi_siswas` — lebih rapi
- `users.role` sebagai enum sederhana menggantikan Spatie RBAC penuh (`roles`, `permissions`, `model_has_*`) — keputusan sadar ADR-008, bukan celah, sesuai skala kebutuhan (5-6 role tetap, bukan RBAC dinamis)
- Konsep **kampus** (multi-lokasi) tidak ada di data lama sama sekali — ini fitur BARU murni, tidak ada yang perlu dimigrasikan untuk kampus (asumsi: semua data lama masuk ke 1 kampus default saat migrasi)

### ❌ Field yang HILANG di Skema Baru (Keputusan: 2026-07-22 — akan DITAMBAHKAN)
**Ke `Teacher` model** (keputusan user: tambahkan, jangan hilang):
- `gelar_depan`, `gelar_belakang` (gelar akademik, char(255) di lama — kemungkinan typo lama harusnya varchar, cek saat migrasi)
- `no_wa` (nomor WhatsApp — beda dari `no_hp` yang sudah ada di skema baru? perlu klarifikasi apakah field sama atau field tambahan)
- `status_pernikahan` (enum: Menikah/Belum Menikah/Pernah Menikah)
- `status_kepegawaian` (enum: **Guru/Karyawan** — KRITIS, 28% data adalah Karyawan non-guru, tabel `Teacher` tetap dipakai sebagai wadah generik pegawai sesuai keputusan user "sesuaikan dengan data lama")
- `agama` sudah ada di `Student` tapi **tidak ada di `Teacher`** skema baru — data lama punya `agama` di `pegawais` juga
- `tempat_lahir`, `tanggal_lahir`, `jenis_kelamin` — sudah ada di `Student`, **tidak ada di `Teacher`** skema baru padahal data lama `pegawais` punya semua ini
- `alamat` — sudah ada di `Student`, tidak ada di `Teacher`

**Field yang HILANG tanpa keputusan tegas (perlu didiskusikan lagi):**
- `presensi_pegawais.webcam_datang`/`webcam_pulang` (path foto capture saat tap) — tidak ada padanan di `attendance_records`. Perlu cek apakah file fisiknya masih ada untuk dipertimbangkan migrasi, atau dianggap tidak penting
- `siswas.status` granular (Belum Lulus/Lulus/**Mengundurkan Diri**) vs `PersonStatus` baru cuma `aktif`/`nonaktif` — informasi "kenapa keluar" (lulus normal vs mengundurkan diri) akan hilang kalau tidak ditambahkan
- `siswas.tahun_lulus` — tidak ada padanan di `Student` baru

### ⚠️ Perbedaan Struktural yang Butuh Strategi Migrasi (bukan sekadar tambah kolom)
1. **Nama kelas = kombinasi 3 tabel lama → 1 kolom `Kelas.nama` baru** (lihat temuan di atas) — perlu proses ETL yang generate kombinasi unik, bukan migrasi tabel 1:1
2. **PK lama adalah `varchar` (UUID/string acak), PK baru `Int autoincrement`** — wajib bikin mapping table sementara (id lama → id baru) selama proses migrasi, tidak bisa insert langsung dengan id yang sama
3. **`presensi_pegawais` tidak punya FK constraint asli ke smartcard** (cuma index, beda dari `presensi_siswas` yang benar ada FK) — kemungkinan bug/inkonsistensi di aplikasi lama, migrasi harus validasi manual data yang mungkin yatim (orphaned) di tabel ini
4. **Tidak ada konsep "kampus" di data lama** — semua data lama diasumsikan masuk ke 1 kampus (default) kecuali ditentukan lain
5. **`role` di `users` lama adalah string bebas** (bukan enum FK ke `roles` Spatie — meski tabel `roles`/`model_has_roles` ada, kolom `users.role` sendiri tetap varchar terpisah, kemungkinan legacy dari sebelum Spatie dipasang) — perlu cek nilai-nilai unik yang benar-benar dipakai sebelum dipetakan ke `UserRole` enum baru

---

## ✅ Keputusan Final (2026-07-22)
- [x] Field biodata guru/pegawai yang hilang → **ditambahkan ke `Teacher`** (gelar depan/belakang, status_pernikahan, status_kepegawaian, agama, tempat_lahir, tanggal_lahir, jenis_kelamin, alamat)
- [x] Karyawan non-guru → **tetap masuk tabel `Teacher`** (wadah generik pegawai, bukan role login — field `status_kepegawaian` yang membedakan Guru vs Karyawan)
- [x] `no_wa` vs `no_hp` → **digabung jadi 1 field** `no_hp` yang sudah ada di `Teacher` — TIDAK buat field `no_wa` terpisah, data lama `no_wa` dipetakan langsung ke `no_hp` saat migrasi
- [x] Foto webcam tap (`webcam_datang`/`webcam_pulang`) → **diabaikan**, tidak dimigrasikan — tidak ada kolom baru untuk ini di `attendance_records`
- [x] Status siswa granular (Lulus vs Mengundurkan Diri) → **kolom baru terpisah** `alasanNonaktif` (enum: `lulus`/`mengundurkan_diri`/`lainnya`, nullable) di `Student` — TIDAK mengubah/expand enum `PersonStatus` yang sudah dipakai luas
- [x] `tahun_lulus` → **field baru** `tahunLulus` (Int, nullable) ditambahkan ke `Student`
- [x] Kampus default untuk semua data lama → **"Kampus 1"** (sesuai kampus yang sudah ada di seed Fase 1, cek dulu apakah sudah ada baris `Kampus` dengan nama ini sebelum insert baru)

## ✅ Keputusan Final Tambahan (2026-07-22, ronde 2)
- [x] Data wali murid (nama_ayah/ibu, no_hp_ayah/ibu, no_hp_siswa, rtrw) → **DIMIGRASIKAN SEMUA**. `rtRw`/`namaAyah`/`namaIbu` sudah ada di `Student` (T028), sisanya ditambahkan lewat T063
- [x] `smartcards.id`/`smartcard_siswas.id` (varchar) → **dikonfirmasi ini adalah UID fisik kartu RFID** (bukan ID internal) — mapping langsung ke `cards.uid`, kartu fisik lama tetap kompatibel tanpa registrasi ulang

## ❓ Masih Perlu Dicek Saat Eksekusi (bukan keputusan desain, tapi verifikasi data — lihat T062)
- [ ] Nilai unik `users.role` lama (string bebas) — perlu di-dump dulu untuk dipetakan manual ke `UserRole` enum baru (kemungkinan tidak 1:1 dengan role baru yang lebih granular)
- [ ] Strategi mapping ID lama (varchar UUID) → ID baru (Int autoincrement) — WAJIB tabel translasi sementara selama proses ETL, disimpan permanen sebagai audit trail
- [ ] Data `presensi_pegawais` yang mungkin yatim (orphaned, karena tidak ada FK constraint asli) — perlu query validasi sebelum migrasi, catat berapa banyak baris bermasalah
- [ ] `pegawais.status_pekerjaan = 'Cuti'` (1 baris ditemukan) — perlu dipetakan ke `aktif` atau `nonaktif` di `PersonStatus` baru, tanyakan user saat eksekusi kalau belum diputuskan

---

## 🔗 Lihat Juga
- 04-Database-Schema (04-Database-Schema.md) — skema baru lengkap
- `apps/api/prisma/schema.prisma` — sumber kebenaran teknis terkini
- File dump lama: `/media/anunnaki/DataNvme/sql_absensi_smk.sql`
