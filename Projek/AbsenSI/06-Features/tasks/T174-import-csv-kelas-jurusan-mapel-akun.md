# T174 — API+Web: Import CSV untuk Kelas, Jurusan, Mapel, User/Akun

## Depends on
Tidak ada dependency teknis. Independen — bisa dikerjakan kapan saja, TIDAK bergantung rangkaian Jurnal Guru (T168-T173) maupun Jam Pelajaran (T158-T160).

## Objective
Tambah fitur Import CSV untuk 4 modul master data yang saat ini HANYA punya form "Tambah" manual satu-per-satu: **Kelas**, **Jurusan**, **Mapel**, **User/Akun** — prioritas eksplisit dari user karena volume input besar saat setup tahun ajaran baru.

## Context — Audit Menyeluruh (2026-08-13)

Audit sistematis SEMUA modul master data di codebase menemukan: dari ~15 modul dengan form "Tambah Data", hanya **3** (Siswa, Guru, Kartu) yang punya Import CSV, via infrastruktur `apps/api/src/import/import.controller.ts` + `import.service.ts` (guard `super_admin`, pola konsisten: `parseCsv()` via `csv-parse/sync` → loop baris → validasi wajib isi → cek duplikat dalam file (`Set`) → cek duplikat di DB → `prisma.create()` → kumpulkan `ImportRowError[]` → return `ImportReport {totalRows, successCount, failedCount, errors}`). Frontend: komponen reusable `apps/web/src/components/import-dialog.tsx` (`ImportDialog`, props: `endpoint`, `columns`, `example`), dipakai di `siswa-view.tsx`/`guru-view.tsx`/`kartu-view.tsx`.

User mengecualikan modul yang BUKAN "tambah banyak record baru" secara eksplisit dari scope ini — **Toleransi Keterlambatan** (config singleton, 1 nilai global) dan **Wali Kelas** (assignment guru↔kelas existing, bukan create record baru) TIDAK masuk task ini.

**Modul lain yang TIDAK masuk task ini** (volume kecil, form manual sudah wajar): Kampus, Ekstrakurikuler, Hari Libur, Jadwal Mengajar (sudah punya task tersendiri T160, depends T158+T159), Jadwal Blok Minggu, Tahun Ajaran, Semester. Jam Pelajaran — modul BARU (T158, belum ada sama sekali) TIDAK relevan untuk import CSV karena strukturnya kompleks (banyak baris per hari dalam 1 opsi), bukan record flat sederhana — TIDAK masuk scope task ini.

## Spec Detail

### 1. Backend — extend `ImportService` existing (`apps/api/src/import/import.service.ts`)

Tambah 4 method baru, REPLIKASI PERSIS pola yang sudah ada (`importStudents`/`importTeachers`/`importCards`) — validasi wajib isi → duplikat dalam file → duplikat/lookup di DB → create → kumpulkan error, JANGAN buat pola baru berbeda:

- **`importKelas(buffer)`** — kolom CSV: `nama, jurusan, tingkat, kampus, ruangan` (kolom `ruangan` OPSIONAL, hanya relevan KALAU T169 sudah dikerjakan lebih dulu dan field `Kelas.ruangan` sudah ada — kalau T169 BELUM dikerjakan saat T174 dieksekusi, kolom `ruangan` DIABAIKAN/tidak divalidasi, TIDAK error). `jurusan` dan `kampus` di-lookup by nama (harus SUDAH ADA di DB, mirip pola `kelasNama`+`jurusanNama` di `importStudents` — TOLAK dengan pesan jelas kalau tidak ditemukan, JANGAN auto-create jurusan/kampus baru dari sini). `tingkat` harus salah satu nilai enum `Tingkat` (X/XI/XII) valid.
- **`importJurusan(buffer)`** — kolom CSV: `nama` (unik, cek constraint DB Jurusan — VERIFIKASI dulu apakah `Jurusan.nama` punya `@unique`, kalau tidak, cek duplikat manual seperti pola NISN/NIY).
- **`importMapel(buffer)`** — kolom CSV: `nama` (dan kolom lain yang WAJIB di model `Mapel` — VERIFIKASI schema `Mapel` dulu sebelum finalize kolom, kemungkinan cuma `nama` tapi CEK field lain seperti kode mapel kalau ada).
- **`importUsers(buffer)`** — kolom CSV: `username, nama, role, password` (role harus salah satu nilai enum `UserRole` valid — VERIFIKASI dulu apakah SEMUA role masuk akal di-bulk-import, atau HANYA sebagian yang wajar, misal `guru_piket`/`admin_jurnal`/`kepsek` — TANYAKAN ke user kalau ragu role mana yang boleh dibuat lewat CSV, JANGAN izinkan `super_admin` dibuat lewat CSV tanpa konfirmasi eksplisit — risiko keamanan kalau file CSV bocor bisa dipakai bikin akun super_admin baru). Password — HARUS di-hash sebelum simpan (REUSE cara hash password yang SUDAH dipakai endpoint create-user manual, JANGAN implementasi hash baru). PERTIMBANGKAN: password di-generate random + wajib ganti saat login pertama (`ForcePasswordChangeConfig` — cek apakah pola ini SUDAH ADA di sistem dan bisa di-reuse untuk akun hasil import), DARIPADA password mentah tersimpan apa adanya di file CSV yang mungkin di-share admin lewat channel tidak aman — REKOMENDASI KUAT opsi ini, TAPI klarifikasi ke user kalau menambah kompleksitas signifikan di luar scope awal.

### 2. Backend — endpoint baru di `import.controller.ts`

- `POST /import/kelas`, `POST /import/jurusan`, `POST /import/mapel`, `POST /import/users` — guard SAMA `@Roles(UserRole.super_admin)` (User/Akun JANGAN diperluas ke role lain, ini paling sensitif). `@LogActivity` di semua endpoint baru (konsisten aturan proyek).

### 3. Frontend — reuse `ImportDialog` di 4 halaman

- `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx` — tambah `ImportDialog` untuk Kelas DAN untuk Jurusan (2 dialog terpisah kalau halaman ini menangani keduanya dalam 1 tampilan, atau digabung kalau lebih masuk akal — VERIFIKASI struktur halaman existing dulu sebelum putuskan).
- `apps/web/src/app/(admin)/mapel/*` (path final tergantung apakah T157 sudah menduplikasi ke `(admin)/mapel/` — kalau sudah, tambah di situ; kalau belum, tambah di `(admin-jurnal)/admin-jurnal/mapel/`).
- `apps/web/src/app/(admin)/akun/akun-view.tsx` — tambah `ImportDialog` untuk User/Akun.
- Semua REUSE komponen `ImportDialog` existing (props `endpoint`+`columns`+`example`), TIDAK membuat komponen dialog baru dari nol.

## Edge Cases
- Import Kelas yang referensi Jurusan/Kampus BELUM ADA — TOLAK per baris dengan pesan jelas menyebut nama yang tidak ditemukan (KONSISTEN pola `importStudents` untuk kelas/jurusan yang tidak match).
- Import User dengan role yang TIDAK ADA di enum `UserRole` atau role yang DIPUTUSKAN tidak boleh via CSV (kemungkinan `super_admin`) — TOLAK dengan pesan jelas.
- Import User dengan `username` yang sudah ada — TOLAK (duplikat), KONSISTEN pola NISN/NIY.
- File CSV kosong/tanpa header yang sesuai — `parseCsv()` existing SUDAH menangani ini secara umum (`columns: true` akan gagal parse dengan jelas kalau header tidak cocok) — TIDAK PERLU penanganan tambahan khusus di luar yang sudah ada.

## Files
- **Modifikasi:** `apps/api/src/import/import.service.ts` (4 method baru), `apps/api/src/import/import.controller.ts` (4 endpoint baru), `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx`, halaman Mapel (lokasi tergantung status T157), `apps/web/src/app/(admin)/akun/akun-view.tsx`.
- **Jangan sentuh:** `importStudents`/`importTeachers`/`importCards` existing (REUSE pola, TIDAK diubah perilakunya), komponen `ImportDialog` (reuse apa adanya, TIDAK direfaktor kecuali benar-benar perlu prop baru yang backward-compatible).

## Acceptance Criteria
- [ ] Admin bisa import CSV untuk Kelas (dengan lookup Jurusan+Kampus, opsional ruangan kalau T169 sudah ada).
- [ ] Admin bisa import CSV untuk Jurusan.
- [ ] Admin bisa import CSV untuk Mapel.
- [ ] Admin bisa import CSV untuk User/Akun — role sensitif (super_admin) DITOLAK atau butuh konfirmasi eksplisit tambahan, password TIDAK tersimpan mentah tanpa hash.
- [ ] Semua endpoint baru guard `super_admin`, `@LogActivity` terpasang — verified tercatat.
- [ ] Error per baris ditampilkan jelas ke admin (nama field yang salah, alasan spesifik) — konsisten pola 3 import existing.
- [ ] Build + type-check hijau, jest untuk 4 method baru pass.

## Validasi Claudian
- [ ] Konfirmasi keputusan role mana yang BOLEH dibuat via import User (super_admin dikecualikan atau tidak) — klarifikasi ke user kalau ragu, JANGAN putuskan sepihak untuk hal sesensitif ini.
- [ ] Konfirmasi password hasil import User di-hash dengan cara YANG SAMA seperti create-user manual (reuse, bukan implementasi baru).
- [ ] Konfirmasi `@LogActivity` terpasang di 4 endpoint baru (cek [[feedback_wajib_log_activity]]).
- [ ] Kalau T169 belum dikerjakan saat task ini dieksekusi — konfirmasi kolom `ruangan` di CSV Kelas diabaikan dengan aman (tidak error), bukan diasumsikan sudah ada.
