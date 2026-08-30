# T263 — Dev: Duplikasi Data Kelas X GAME DEV untuk Screenshot Halaman "Hari Ini" Wali Kelas

## Depends on
Tidak ada dependency teknis ke task lain. Murni operasi data (seed script) di **database DEV** (`absensi-mysql-1`, port 3307) — **JANGAN SENTUH DATABASE PRODUCTION SAMA SEKALI**.

**Prasyarat WAJIB dicek sebelum jalankan seed**: dev sekarang punya migration baru `20260828144050_t257_teacher_wifi_access` (dari commit `8f726a5`, T257 fitur Akses Wifi Guru — **penomoran T257 itu MILIK fitur wifi, TIDAK ADA hubungannya dengan task ini**, kebetulan nomor task ini awalnya salah dipakai sama sebelum diperbaiki jadi T263) yang menurut catatan STATUS.md **belum pernah di-apply** ke DB dev manapun. Jalankan `pnpm --filter @absensi/api exec prisma migrate deploy` (atau `migrate dev` kalau masih ada migration lain yang juga pending) DULU sebelum seed script T263 ini, supaya Prisma Client tidak stale.

## Konteks — Kenapa Task Ini Ada

User butuh screenshot halaman **"Hari Ini" Wali Kelas** (dashboard monitoring harian, T226/T241) untuk kelas **X GAME DEV**, tapi tanggal saat ini (30 Agustus 2026 ke atas) bukan hari aktif belajar — jadi halaman itu kosong/tidak representatif untuk keperluan dokumentasi.

**Opsi mengubah jam server PRODUCTION ditolak** (dibahas eksplisit dengan user 2026-08-30) — risiko nyata: data tap asli yang masuk selama jam dipalsukan akan tercatat dengan `tanggal` salah permanen (kiosk kemungkinan tetap nyala 24/7 dan tap terus terkirim), scheduler/cron (BullMQ, lock guru T237, backup harian) jalan independen dari ada-tidaknya user browsing dan bisa terpicu keliru, serta sesi JWT aktif bisa rusak. **Solusi yang disepakati: duplikasi data ke server DEV** (database terpisah total dari production, aman disentuh) — build kondisi kelas X GAME DEV di dev supaya PERSIS seperti kondisi kelas itu di production pada **26 Agustus 2026**, lalu screenshot dari situ.

## Poin PALING PENTING — Tanggal Baris Data HARUS "Hari Ini" Dev, BUKAN Literal 26 Agustus

Halaman "Hari Ini" Wali Kelas (`hari-ini-tab.tsx` → `GET /journal/kelas-wali-status-hari-ini` → `AttendanceService.resolveStatusHarianSiswa()`) **SELALU** query berdasarkan `CURRENT_DATE()` server saat request terjadi (lihat `[[T241-dashboard-hari-ini-tampilkan-jam-pulang]]` — "hitung `today` fresh tiap request, filter `WHERE tanggal = today`"), **BUKAN** parameter tanggal yang bisa dipilih bebas dari UI.

**Konsekuensi WAJIB DIPATUHI**: baris `attendance_records` yang diduplikasi ke dev **HARUS** diberi `tanggal` = **tanggal hari ini SAAT SCRIPT INI DIJALANKAN** (`CURDATE()` di sisi MySQL, atau `new Date()` di sisi Node — bukan string `'2026-08-26'` yang di-hardcode), supaya saat user membuka halaman "Hari Ini" di dev pada hari itu juga, datanya muncul. Kalau user screenshot beda hari dari saat script dijalankan, script harus dijalankan ULANG dulu di hari screenshot itu (rerun aman — lihat bagian "Idempotency" di bawah).

**Jam** (`waktu_masuk`/`waktu_pulang`) DUPLIKASI PERSIS dari referensi production di bawah (jam-menit-detik sama), hanya **tanggalnya** yang diganti ke hari ini dev.

## Data Referensi dari Production (2026-08-26) — SUDAH DIVERIFIKASI LANGSUNG ke DB, JANGAN QUERY ULANG

### Kelas
- **Kelas**: `X GAME DEV`, id production = **48**, `jurusan_id` = **3** (TKJ), `tingkat` = **X**, `kampus_id` = **2** (Kampus 2).
- Dev **BELUM PUNYA** kelas ini sama sekali (dicek 2026-08-30, `SELECT * FROM kelas WHERE nama LIKE '%GAME%'` kosong) — **buat baru**, ID akan auto-increment beda dari 48, itu wajar/tidak masalah.
- Mapping ke dev: `kampus_id=2` di dev SUDAH ADA dengan nama sama persis ("Kampus 2") — pakai langsung. `jurusan_id` di dev: **TKJ = id 3 juga di dev** (kebetulan sama, TAPI VERIFIKASI ULANG SAAT EKSEKUSI, jangan asumsi ID selalu identik dev vs prod — cross-check `SELECT id FROM jurusan WHERE nama='TKJ'` di dev).

### Wali Kelas (akun guru)
- Username **`081358390505`**, role `guru`, nama guru **"Mohamad Farkhan Fahmi Zuhri"**.
- Dev **BELUM PUNYA** teacher dengan nama ini maupun user dengan username ini (dicek 2026-08-30, kosong) — buat baru: 1 baris `teachers` + 1 baris `users` (role=`guru`, `teacher_id` mengarah ke teacher baru, `kelas_id_wali` mengarah ke kelas X GAME DEV baru).
- **Password**: pakai default proyek `"12345678"` (konsisten `[[T232-generate-akun-guru-massal]]`/`[[T239-reset-password-ke-default-bukan-random]]`) — hash dengan bcrypt SAMA SEPERTI cara `users.service.ts` men-generate akun guru biasa, JANGAN taruh plaintext.
- `must_change_password`: `false` (supaya user bisa langsung login tanpa halangan untuk screenshot).

### 24 Siswa Kelas X GAME DEV (nama + NISN persis dari production, semua `status='aktif'` KECUALI baris bertanda)

| Nama | NISN | Status |
|---|---|---|
| ABDUL QOHHAR BAHA'UDIN | 0113366749 | aktif |
| ARJUNA SOEN TJIU LAYANA FAUSTA JANET | 0115447101 | aktif |
| ARVIN OWEN OTTA PANDORA | 0103326881 | aktif |
| BAGUS SABAR UDIN | 0099177003 | aktif |
| DIAN ANUGERAH | 0114358957 | aktif |
| DIKYA GARIS AVENSYA | 0101056143 | aktif |
| FAKHRI LENDRA RAHMAT HIDAYAT | 0115901283 | aktif |
| FARHAN NAZZA ABIL SYAHRIZA | 3103094295 | aktif |
| FATHIR TITO AURIL MAULANA | 0108585862 | aktif |
| FAWAZ BADRUS SHOLEH HASBULLAH | 0101124322 | aktif |
| FERDINAND SALIM | 0091828252 | aktif |
| HERJUNO SATRIO LINUWEH | 0101547901 | aktif |
| KRISHNANDA REHAN RAMADHAN | 0108916975 | aktif |
| MARISA EMILIA PUTRI | 0133527816 | aktif |
| MUHAMMAD ANWAR ULUM | 3107628557 | aktif |
| **NABILA ICA FITRIANI** | 0102009079 | **nonaktif** |
| NOVAL EKA FEBRIAN | 3100365478 | aktif |
| RAFI SAPUTRA | 0105449619 | aktif |
| RAHMAT RIJAL PAMUNGKAS | 0092549407 | aktif |
| RESTU RADITYA SANTOSO | 0102270958 | aktif |
| RIZAL NANDA PRATAMA | 0106279629 | aktif |
| VINGKI TIO OYIFANY | 0114782903 | aktif |
| WILDAN MURI FITRIANSYAH | 0114819911 | aktif |
| WILLIAM STONER | 0107021827 | aktif |

Dicek 2026-08-30: **tidak ada satupun NISN di atas yang sudah dipakai** di dev — aman insert baru tanpa konflik unique constraint. **PENTING**: NABILA ICA FITRIANI berstatus **nonaktif** di production — buat SENGAJA nonaktif juga di dev, supaya screenshot juga membuktikan fix `[[T220-fix-siswa-nonaktif-tetap-terekap]]`/`[[T227-migrasi-retroaktif-siswa-nonaktif-lama]]` bekerja benar (siswa nonaktif tidak boleh muncul di rekap "Hari Ini").

### Data Kehadiran Referensi (production, tanggal asli 2026-08-26 — GANTI tanggal ke hari ini dev, JAM-MENIT-DETIK DIPERTAHANKAN PERSIS)

Semua waktu di bawah sudah dikonversi dari UTC (nilai mentah DB) — kolom "Masuk"/"Pulang" adalah **WIB**. Saat insert ke dev, **simpan sebagai UTC lagi** (kurangi 7 jam) supaya konsisten dengan cara backend menyimpan (`waktu_masuk`/`waktu_pulang` = `DATETIME` UTC, dikonversi ke WIB di layer tampilan) — **VERIFIKASI SAAT IMPLEMENTASI** dengan cek 1 baris attendance_records existing lain di dev untuk konfirmasi asumsi ini benar sebelum insert massal.

| Siswa (NISN) | Masuk (WIB) | Pulang (WIB) | Status | pulang_via |
|---|---|---|---|---|
| 0113366749 | 06:27:50 | — (NULL) | hadir | NULL |
| 0115447101 | 06:37:48 | — (NULL) | hadir | NULL |
| 0103326881 | 06:41:22 | — (NULL) | hadir | NULL |
| 0099177003 | 06:38:31 | 16:10:43 | hadir | tap |
| 0114358957 | 06:37:20 | — (NULL) | hadir | NULL |
| 0101056143 | 06:31:09 | — (NULL) | hadir | NULL |
| 0115901283 | 06:55:53 | — (NULL) | hadir | NULL |
| 3103094295 | 07:12:01 | 16:10:58 | **terlambat** | tap |
| 0108585862 | 06:53:39 | — (NULL) | hadir | NULL |
| 0101124322 | 06:37:55 | 16:13:22 | hadir | tap |
| 0091828252 | 06:27:39 | 16:14:55 | hadir | tap |
| 0101547901 | 06:27:58 | 16:20:22 | hadir | tap |
| 0108916975 | — (tidak tap sama sekali) | — | **ALFA** (tidak ada baris) | — |
| 0133527816 | 06:29:03 | — (NULL) | hadir | NULL |
| 3107628557 | 06:35:56 | 16:13:21 | hadir | tap |
| 0102009079 (Nabila, nonaktif) | — (tidak ada baris — SENGAJA, jangan buat) | — | — | — |
| 3100365478 | 06:37:20 | — (NULL) | hadir | NULL |
| 0105449619 | 06:35:59 | — (NULL) | hadir | NULL |
| 0092549407 | 06:43:30 | — (NULL) | hadir | NULL |
| 0102270958 | 06:47:16 | 16:10:40 | hadir | tap |
| 0106279629 | 06:47:14 | 16:10:45 | hadir | tap |
| 0114782903 | 06:53:56 | 16:13:20 | hadir | tap |
| 0114819911 | 06:28:03 | 16:19:42 | hadir | tap |
| 0107021827 | 06:35:58 | 16:10:47 | hadir | tap |

**Ringkasan kategori** (untuk verifikasi visual setelah screenshot — harus cocok dengan 5 kartu kategori di halaman "Hari Ini"):
- **Hadir** (sudah tap masuk, sebagian juga sudah tap pulang, sebagian belum): 22 siswa aktif (semua kecuali Krishnanda yang alfa dan Farhan yang terlambat).
- **Terlambat**: 1 siswa (Farhan Nazza Abil Syahriza).
- **Alfa**: 1 siswa (Krishnanda Rehan Ramadhan — tidak ada baris `attendance_records` sama sekali, TIDAK PERLU insert apa pun untuknya, alfa dihitung otomatis saat query rekap sesuai aturan proyek "Alfa bukan kolom DB").
- **Izin/Sakit/Dispen**: 0 (dicek tabel `permits` production untuk tanggal 2026-08-26 kelas ini — kosong, tidak ada).
- **Nonaktif** (tidak ikut dihitung sama sekali): 1 siswa (Nabila).

`client_uuid`: generate string acak unik per baris (`nanoid()` atau setara) saat insert — kolom ini `UNIQUE`, JANGAN dikosongkan/duplikat, tapi nilai persisnya tidak perlu sama dengan production (bukan data yang divalidasi visual).

`academic_year_id`/`semester_id` di baris baru: **JANGAN hardcode `2`/`1`** (itu ID production) — pakai ID **academic_year & semester yang SEDANG AKTIF di DEV** (`SELECT id FROM academic_years WHERE is_active=1`, `SELECT id FROM semesters WHERE is_active=1`), dicek 2026-08-30 dev aktif di `academic_year_id=20` ("TEST 2026/2027") + `semester_id=14` ("genap") — **VERIFIKASI ULANG SAAT EKSEKUSI**, bisa saja sudah berubah kalau ada aktivasi tahun ajaran baru di antara sekarang dan saat task ini dikerjakan.

## Spec Detail — Langkah Implementasi

1. **Buat script seed terpisah** (pola ikuti `apps/api/scripts/seed-jurnal-uji-coba.ts` yang sudah ada di repo untuk seed data uji coba serupa) — JANGAN taruh di `prisma/seed.ts` utama (itu untuk seed dasar sistem, bukan data uji coba sekali pakai). Nama file rekomendasi: `apps/api/scripts/seed-hari-ini-x-gamedev.ts`.
2. Script membuat, DALAM 1 TRANSAKSI Prisma (`$transaction`) supaya idempotent-safe:
   - 1 baris `kelas` (nama, jurusan_id, tingkat, kampus_id sesuai referensi di atas).
   - 24 baris `students` (nama+NISN persis dari tabel di atas, `kelas_id` mengarah ke kelas baru, status sesuai — 23 aktif + 1 nonaktif untuk Nabila).
   - 1 baris `teachers` (nama "Mohamad Farkhan Fahmi Zuhri", `niy` generate unik acak yang jelas format dummy, misal prefix `DEV-` supaya gampang dibedakan dari data asli — VERIFIKASI SAAT IMPLEMENTASI kalau `niy` punya constraint format tertentu).
   - 1 baris `users` (username `081358390505`, role `guru`, `teacher_id` ke teacher baru, `kelas_id_wali` ke kelas baru, password di-hash bcrypt dari `"12345678"`, `must_change_password=false`).
   - 22 baris `attendance_records` (23 siswa aktif MINUS Krishnanda yang alfa = 22 baris tap, sesuai tabel referensi) — `tanggal` = `CURDATE()`/hari ini dev, jam-menit-detik dari tabel referensi (dikonversi ke UTC), `academic_year_id`/`semester_id` dari yang aktif di dev saat itu, `client_uuid` random unik per baris, `pulang_via`/`status` sesuai tabel.
3. **Idempotency WAJIB** — script harus AMAN dijalankan berkali-kali (misal user minta screenshot ulang di hari lain). REKOMENDASI: cek dulu apakah kelas "X GAME DEV" dengan NISN-NISN ini sudah ada (`findFirst` by NISN pertama di tabel) — kalau kelas+siswa+user SUDAH ADA, SKIP pembuatan ulang kelas/siswa/user (idempotent), tapi tetap **hapus baris `attendance_records` LAMA milik siswa-siswa ini untuk SEMUA tanggal** lalu insert ulang dengan `tanggal` = hari ini yang baru — supaya "Hari Ini" selalu menunjuk ke hari eksekusi terbaru, bukan menumpuk data basi dari run sebelumnya.
4. **JANGAN** membuat migration Prisma baru — task ini murni DML (insert data), tidak ada perubahan skema.

## Cara Menjalankan (dicatat untuk user, bukan bagian otomatis)
```bash
pnpm --filter @absensi/api exec ts-node scripts/seed-hari-ini-x-gamedev.ts
```
Dijalankan di folder **dev** (`AbsenSI`, BUKAN `AbsenSI-production`), terhubung ke `absensi-mysql-1` (port 3307) sesuai `.env` dev yang sudah ada — **PASTIKAN SAAT IMPLEMENTASI** `.env` yang aktif memang menunjuk ke DB dev, bukan production (riwayat insiden: pernah ada 1 `.env` dipakai bersama dev+production, lihat `[[feedback_insiden_database_wipe_2026-07-30]]`) — cross-check `DATABASE_URL` di `.env` mengandung port `3307` sebelum run.

## Edge Cases
- **Script dijalankan lewat tengah malam** (misal mulai jam 23:58, selesai jam 00:01 hari berikutnya) — `CURDATE()` di awal transaksi vs saat commit bisa beda tanggal secara teori. RISIKO SANGAT KECIL (transaksi ini cepat, hitungan detik) — TIDAK PERLU penanganan khusus, cukup dicatat sebagai known edge case.
- **User menjalankan ulang script di hari BERBEDA dari hari screenshot pertama** — data lama (tanggal kemarin) TETAP ada di `attendance_records` kalau langkah idempotency poin 3 di atas tidak menghapusnya — pastikan implementasi BENAR-BENAR menghapus baris attendance lama milik siswa-siswa dummy ini sebelum insert ulang, supaya tidak ada 2 hari sekaligus muncul salah tempat di rekap lain (mis. halaman Rekap Kehadiran kelas ini jadi punya history palsu menumpuk tiap kali script di-rerun).
- **Kelas "X GAME DEV" MUNGKIN sudah dibuat manual oleh user via UI sebelum script sempat jalan** (kalau user tidak sabar menunggu task ini) — script HARUS cek dulu by nama+kampus_id sebelum insert kelas baru, jangan sampai duplikat kelas dengan nama sama.

## Files
- **Baru:** `apps/api/scripts/seed-hari-ini-x-gamedev.ts`.
- **Jangan sentuh:** `prisma/seed.ts` (seed dasar sistem), APAPUN di `AbsenSI-production/` (folder production, task ini 100% scope dev), skema Prisma (tidak ada migration baru).

## Acceptance Criteria
- [ ] Kelas "X GAME DEV" muncul di dev dengan 24 siswa (nama+NISN persis referensi), 1 di antaranya (Nabila) berstatus nonaktif.
- [ ] Login sebagai `081358390505` / password `12345678` berhasil, masuk sebagai wali kelas kelas X GAME DEV (bukan role lain, bukan wali kelas lain).
- [ ] Halaman "Hari Ini" Wali Kelas menampilkan: 22 siswa di kartu Hadir (10 di antaranya sudah ada jam pulang, sisanya "Belum Pulang" sesuai `[[T241-dashboard-hari-ini-tampilkan-jam-pulang]]`), 1 siswa di kartu Terlambat (Farhan), 1 siswa di kartu Alfa (Krishnanda), 0 di kartu Izin/Sakit/Dispen.
- [ ] Nabila Ica Fitriani (nonaktif) **TIDAK MUNCUL** di kartu manapun.
- [ ] Script aman dijalankan berkali-kali tanpa duplikat kelas/siswa/user, dan data attendance selalu ter-refresh ke tanggal hari eksekusi terbaru.
- [ ] Tidak ada perubahan apa pun ke database production — dikonfirmasi eksplisit sebelum & sesudah eksekusi (`SELECT COUNT(*) FROM kelas` di production harus TETAP SAMA sebelum/sesudah task ini).

## Validasi Claudian
- [ ] Konfirmasi `tanggal` baris `attendance_records` yang di-generate = tanggal REAL saat script dijalankan (`CURDATE()`), BUKAN string hardcode `'2026-08-26'` — ini poin paling gampang salah kalau developer/executor lain tidak baca konteks task dengan teliti.
- [ ] Konfirmasi konversi UTC↔WIB benar (jam yang tampil di UI setelah insert harus PERSIS sama dengan tabel referensi WIB di atas, bukan geser 7 jam gara-gara lupa konversi atau konversi dobel).
- [ ] Konfirmasi tidak ada 1 baris pun tersentuh di `absensi-mysql-prod` — jalankan `SELECT COUNT(*) FROM attendance_records WHERE tanggal='2026-08-26'` di PRODUCTION sebelum & sesudah task, angkanya harus identik.
