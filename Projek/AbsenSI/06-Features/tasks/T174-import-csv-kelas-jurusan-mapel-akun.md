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
- [x] Admin bisa import CSV untuk Kelas (dengan lookup Jurusan+Kampus, kolom `ruangan` diterima tapi DIABAIKAN karena T169 belum dikerjakan).
- [x] Admin bisa import CSV untuk Jurusan.
- [x] Admin bisa import CSV untuk Mapel.
- [x] Admin bisa import CSV untuk User/Akun — role `super_admin` DITOLAK EKSPLISIT dengan pesan keamanan, password DARI KOLOM CSV (keputusan user) tapi DI-HASH sebelum simpan (tidak pernah mentah), `mustChangePassword` HARDCODE `true` untuk semua akun hasil import.
- [x] Semua endpoint baru guard `super_admin` — verified 403 untuk `card_admin`, log ringkasan tercatat via `ActivityLogService.record()` (pola sama 3 import existing, BUKAN `@LogActivity` decorator per-baris karena bulk).
- [x] Error per baris ditampilkan jelas ke admin (nama field yang salah, alasan spesifik) — konsisten pola 3 import existing, verified live tiap skenario.
- [x] Build + type-check hijau, jest untuk 4 method baru pass (25 test baru).

## Validasi Claudian
- [x] Konfirmasi keputusan role mana yang BOLEH dibuat via import User — **DIKONFIRMASI via AskUserQuestion 2026-08-14**: semua role KECUALI `super_admin` (guru, guru_piket, kepsek, admin_jurnal, card_admin, pembina_ekstra).
- [x] Konfirmasi password hasil import User — **DIKONFIRMASI via AskUserQuestion 2026-08-14**: dari kolom CSV apa adanya (BUKAN auto-generate), di-hash dengan bcrypt SALT_ROUNDS 10 (SAMA persis `UsersService.create()`), `mustChangePassword` HARDCODE `true` (bukan `ForcePasswordChangeConfigService.shouldForceFor()` yang bisa di-toggle off — mitigasi wajib tidak boleh dimatikan untuk jalur CSV).
- [x] `@LogActivity` — pola bulk import TIDAK pakai decorator itu (konsisten `importStudents`/`importTeachers`/`importCards` existing), dipakai `logImportSummary()` manual via `ActivityLogService.record()`, 1 entry ringkasan per operasi — verified tercatat live untuk keempat endpoint baru.
- [x] T169 belum dikerjakan saat eksekusi (dikonfirmasi `Kelas.ruangan` tidak ada di schema) — kolom `ruangan` di CSV Kelas diterima via `KelasRow` interface tapi TIDAK dipakai/divalidasi, verified live tidak error.

## Status Eksekusi (2026-08-14)

**Selesai.**

**Backend (`apps/api/src/import/`)**:
- `import.service.ts` — 4 method baru (`importKelas`, `importJurusan`, `importMapel`, `importUsers`), REPLIKASI PERSIS pola 3 method existing (validasi wajib isi → duplikat dalam file → duplikat/lookup DB → create → kumpulkan error). `IMPORTABLE_ROLES` const eksplisit exclude `super_admin`. `importUsers()` replikasi manual constraint `UsersService.ensureLinksValid()` (guru/guru_piket wajib teacherId via lookup NIY, guru_piket wajib kampusId via lookup nama) — TIDAK reuse `UsersService` langsung (private method, tidak diekspor modul, konsisten pola 3 method lama yang juga tidak panggil service lain kecuali `CardsService.create()` publik).
- `import.controller.ts` — 4 endpoint baru (`POST /import/kelas`, `/jurusan`, `/mapel`, `/users`), guard `@Roles(super_admin)` level controller (TIDAK diperluas seperti `/cards` yang juga terima `card_admin`), `logImportSummary()` dipanggil di semuanya.
- `import.service.spec.ts` (baru) — 25 test: lookup jurusan/kampus, validasi tingkat enum, kolom ruangan diabaikan aman, dedup file+DB untuk Jurusan/Mapel (termasuk `kode` unique), DAN untuk User — role super_admin ditolak eksplisit, password di-hash (bukan mentah), guru/guru_piket wajib link valid, password <8 karakter ditolak, username duplikat.

**Frontend**:
- `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx` — 2 `ImportDialog` terpisah (Jurusan di `JurusanCard`, Kelas di `KelasCard`), `router.refresh()` + `useEffect` sync state lokal dari props Server Component setelah refetch.
- `apps/web/src/app/(admin-jurnal)/admin-jurnal/mapel/mapel-view.tsx` — prop baru `canImport` (default `false`, backward-compatible), `ImportDialog` HANYA tampil kalau `true` — mencegah `admin_jurnal` (yang pakai komponen SAMA via route lain) melihat tombol yang akan gagal 403 (endpoint `/import/mapel` guard `super_admin` saja).
- `apps/web/src/app/(admin)/mapel/page.tsx` — pass `canImport` (route super_admin).
- `apps/web/src/app/(admin)/akun/akun-view.tsx` — `ImportDialog` untuk User/Akun (halaman ini 100% super_admin, tidak perlu conditional).

**Verifikasi live** (dev DB port 3307, API dev port 3101, production tidak disentuh, akun `adminSU`+`adminKartu` password di-override sementara lalu DIKEMBALIKAN):
1. `POST /import/jurusan` — 1 baris baru berhasil, 1 duplikat (DKV existing) ditolak dengan pesan jelas.
2. `POST /import/kelas` — lookup jurusan+kampus berhasil; jurusan tidak ditemukan DITOLAK; tingkat "XIII" (invalid) DITOLAK; kolom `ruangan` diterima TANPA error (diabaikan).
3. `POST /import/mapel` — 1 baris berhasil, duplikat DALAM FILE (nama sama 2x) terdeteksi.
4. `POST /import/users` — skenario lengkap dalam 1 file: `super_admin` DITOLAK dengan pesan keamanan eksplisit; guru tanpa `niy` DITOLAK; password <8 karakter DITOLAK; `admin_jurnal` (tanpa link) BERHASIL; `guru` dengan NIY valid BERHASIL (teacherId ter-resolve); `guru_piket` dengan NIY+kampus valid BERHASIL (teacherId+kampusId ter-resolve). SELECT DB konfirmasi: `must_change_password=1` untuk SEMUA 3 akun sukses, `password_hash` berformat bcrypt (`$2b$10$...`, bukan mentah). Username duplikat (lintas request) DITOLAK.
5. `activity_log` — 5 entri `import.jurusan`/`import.kelas`(x2)/`import.mapel`/`import.users` tercatat dengan ringkasan `{filename, totalRows, successCount, failedCount}`.
6. Role `card_admin` → `POST /import/kelas` dan `/import/users` → 403 Forbidden, dikonfirmasi guard `super_admin`-only bekerja di endpoint baru.
7. Semua data uji (users, kelas, jurusan, mapel, activity_log dengan penanda `t174`) dibersihkan, password hash 2 akun test dikembalikan PERSIS, dikonfirmasi via SELECT.
8. `tsc --noEmit` bersih `apps/api` dan `apps/web`, `jest` — 25 suite / 327 test lulus 100%.
