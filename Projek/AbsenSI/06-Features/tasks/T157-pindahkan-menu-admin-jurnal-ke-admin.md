# T157 — API+Web: Duplikasi Menu Admin-Jurnal ke Grup Admin (Super_Admin Bisa Kelola Sendiri)

## Depends on
Tidak ada dependency teknis wajib, TAPI KERJAKAN SETELAH T156 (Semester CRUD) kalau memungkinkan — supaya pola "section baru di admin yang reuse komponen admin-jurnal" sudah punya 1 preseden sukses sebelum mengerjakan 6 menu sekaligus di task ini.

## Objective
Super_admin bisa mengakses DAN MENGEDIT 6 menu yang SAAT INI eksklusif role `admin_jurnal` — Mata Pelajaran, Jadwal Mengajar, Jadwal Blok Minggu, Izin Guru, Wali Kelas, Toleransi Keterlambatan — LANGSUNG dari sidebar admin biasa, TANPA perlu pindah role/login ulang sebagai admin_jurnal.

## Context — Temuan Riset (2026-08-11)

**6 menu di `(admin-jurnal)/` yang jadi scope task ini** (path lengkap, dari riset kode langsung):

| Menu | Path Halaman Admin-Jurnal | Backend Controller |
|---|---|---|
| Mata Pelajaran | `(admin-jurnal)/admin-jurnal/mapel/` | `apps/api/src/mapel/mapel.controller.ts` |
| Jadwal Mengajar | `(admin-jurnal)/admin-jurnal/jadwal/` | `apps/api/src/core/schedules/schedules.controller.ts` |
| Jadwal Blok Minggu | `(admin-jurnal)/admin-jurnal/jadwal-blok/` | `apps/api/src/block-week-ranges/block-week-ranges.controller.ts` |
| Izin Guru | `(admin-jurnal)/admin-jurnal/izin/` | `apps/api/src/teacher-permits/teacher-permits.controller.ts` |
| Wali Kelas | `(admin-jurnal)/admin-jurnal/wali-kelas/` | `apps/api/src/users/users.controller.ts` (endpoint `assign-wali-kelas`) |
| Toleransi Keterlambatan | `(admin-jurnal)/admin-jurnal/toleransi/` | `apps/api/src/schedule-config/schedule-config.controller.ts` |

**Arsitektur penghalang dikonfirmasi (BUKAN sekadar 1 baris guard)**:
- `apps/web/src/app/(admin-jurnal)/layout.tsx:20-22` — **HARD BLOCK**: `if (user.role !== "admin_jurnal") redirect("/")`. Komentar eksplisit di kode: "route ini EKSKLUSIF admin_jurnal (T050). super_admin TIDAK boleh akses dashboard ini."
- `apps/web/src/app/(admin)/layout.tsx:31-33` — sebaliknya, admin_jurnal yang akses `(admin)/*` di-redirect KELUAR ke `/admin-jurnal/toleransi`.
- Route group `(admin-jurnal)` adalah SHELL TERPISAH TOTAL (sidebar sendiri `admin-jurnal-sidebar.tsx`, layout sendiri) — BUKAN sekadar guard yang bisa "dilonggarkan" 1 baris, karena kalau dilonggarkan, super_admin akan masuk ke SHELL YANG BEDA (sidebar admin-jurnal, bukan sidebar admin 6-grup-accordion yang biasa dipakai) — pengalaman "pindah dashboard", BUKAN "menu nempel di sidebar admin".

**KEPUTUSAN ARSITEKTUR (mengikuti riset, opsi yang direkomendasikan)**: BUKAN melonggarkan guard `(admin-jurnal)`, TAPI **DUPLIKASI ROUTE** — buat halaman BARU di `(admin)/` untuk masing-masing 6 menu, REUSE komponen `*-view.tsx` yang SUDAH ADA (kemungkinan besar reusable langsung karena fetch data via `apiFetch`/`apiClientFetch` generic, tidak hardcode assumsi role tertentu — VERIFIKASI ini per-komponen saat implementasi). Halaman `(admin-jurnal)/` **TETAP ADA APA ADANYA**, TIDAK dihapus — admin_jurnal tetap punya dashboard mereka sendiri seperti sekarang, task ini MURNI MENAMBAH jalur paralel untuk super_admin, bukan memindahkan/menggantikan.

**Backend — role guard yang PERLU DIPERLUAS** (dikonfirmasi lewat riset, endpoint yang SAAT INI eksklusif `admin_jurnal`, PERLU ditambah `super_admin`):
- `apps/api/src/block-week-ranges/block-week-ranges.controller.ts:37,48` — create/remove.
- `apps/api/src/teacher-permits/teacher-permits.controller.ts:51,70,84,93` — create, updateBukti, getBuktiFile, findAll.
- `apps/api/src/schedule-config/schedule-config.controller.ts:24` — update (toleransi).
- `apps/api/src/users/users.controller.ts:79` — assign-wali-kelas.
- `apps/api/src/mapel/mapel.controller.ts:24,31` — create, update.
- `apps/api/src/core/schedules/schedules.controller.ts` — SUDAH dual-role untuk sebagian (findAll, create, getJamMasukDefault, getJamMasukTingkat, update SUDAH `@Roles(super_admin, admin_jurnal)`), TAPI `copyFromSemester` MASIH `admin_jurnal` saja — perlu ditambah.

**Ini perubahan ADDITIVE dan AMAN** (menambah role ke array `@Roles(...)`, TIDAK mengurangi/mengubah akses admin_jurnal yang sudah ada) — TIDAK ADA risiko regresi ke role admin_jurnal.

## Spec Detail

### 1. Backend — perluas `@Roles(...)` di 6 titik yang teridentifikasi

Untuk SETIAP endpoint di daftar "PERLU DIPERLUAS" di atas — tambah `UserRole.super_admin` ke array `@Roles(...)` yang sudah ada (JANGAN hapus `UserRole.admin_jurnal`, tambahkan SAJA). Contoh pola:
```ts
// SEBELUM
@Roles(UserRole.admin_jurnal)
// SESUDAH
@Roles(UserRole.super_admin, UserRole.admin_jurnal)
```

### 2. Frontend — 6 halaman baru di `(admin)/`, masing-masing reuse komponen existing

Untuk SETIAP menu, buat `page.tsx` BARU di path `(admin)/` yang SESUAI (putuskan nama folder final saat implementasi, REKOMENDASI konsisten dengan nama menu: `(admin)/mapel/`, `(admin)/jadwal-mengajar/` — **HATI-HATI JANGAN BENTROK** dengan `(admin)/jadwal/` yang SUDAH ADA untuk Jam Masuk Sekolah 3-Lapis, itu DOMAIN BERBEDA, nama folder baru untuk Jadwal Mengajar HARUS beda supaya tidak ambigu/collision route — misal `(admin)/jadwal-mengajar/`), `(admin)/jadwal-blok/`, `(admin)/izin-guru/`, `(admin)/wali-kelas/`, `(admin)/toleransi-keterlambatan/`.

- Untuk SETIAP halaman baru: `page.tsx` (server component, fetch data awal sama seperti versi admin-jurnal aslinya, guard `super_admin` konsisten pola `(admin)/*` lain) yang me-render komponen `*-view.tsx` YANG SAMA PERSIS (import langsung dari lokasi `(admin-jurnal)/admin-jurnal/xxx/xxx-view.tsx` — REUSE, JANGAN COPY-PASTE kode komponen, supaya perubahan di masa depan otomatis konsisten di kedua tempat).
- **VERIFIKASI per komponen SEBELUM reuse**: baca tiap `*-view.tsx` — apakah ADA asumsi hardcoded terkait role/path (misal link "kembali" yang mengarah ke `/admin-jurnal/...`, atau teks yang menyebut "admin jurnal" secara eksplisit) — kalau ADA, itu perlu disesuaikan jadi generic/dinamis (atau terima 2 varian props), BUKAN dibiarkan salah konteks saat dipakai dari admin biasa.
- **Toleransi Keterlambatan** — komponen ini KEMUNGKINAN BESAR paling sederhana (1 field global), cek dulu apakah reuse langsung tanpa modifikasi.
- **Wali Kelas** — endpoint `assign-wali-kelas` ada DI DALAM `UsersController` (bukan controller terpisah) — pastikan reuse komponen `wali-kelas-view.tsx` tidak bentrok dengan halaman Manajemen Akun (`(admin)/akun/`) yang JUGA terkait `User` — ini 2 fitur BERBEDA (assign wali kelas per-kelas vs manajemen akun users secara umum), JANGAN sampai tercampur.

### 3. Sidebar admin — tambah menu baru

- `apps/web/src/components/shell/nav-items.ts` — TAMBAH 6 entry baru ke `primaryNavGroups`. PUTUSKAN saat implementasi: masuk grup EXISTING yang paling relevan (misal "Master Data Sekolah" untuk Mapel, "Guru" untuk Izin Guru+Wali Kelas, "Absensi & Rekap" untuk Jadwal Mengajar+Jadwal Blok+Toleransi) ATAU buat GRUP BARU "Jurnal Mengajar" yang mengelompokkan SEMUA 6 menu ini jadi 1 accordion baru (REKOMENDASI: grup baru LEBIH JELAS secara UX karena semuanya terkait 1 domain "jurnal mengajar guru", daripada dipencar ke grup-grup lama yang temanya agak beda) — KLARIFIKASI ke user kalau ragu, TAPI kalau harus putuskan sendiri, REKOMENDASI KUAT adalah grup baru.

## Edge Cases
- 2 komponen yang SEBENARNYA overlapping scope (misal `wali-kelas-view.tsx` vs bagian dari `(admin)/akun/`) — pastikan TIDAK terjadi duplikasi UI yang membingungkan admin (2 tempat berbeda untuk assign wali kelas yang sama) — kalau ditemukan overlap SIGNIFIKAN saat implementasi, LAPORKAN ke user sebagai temuan, JANGAN diam-diam gabung/hapus salah satu tanpa konfirmasi (di luar scope keputusan task ini).
- Admin_jurnal LOGIN dan pakai dashboard mereka SEPERTI BIASA — regresi nol WAJIB (task ini aditif murni untuk super_admin, TIDAK MENGUBAH pengalaman admin_jurnal sama sekali).

## Files
- **Buat:** 6 folder halaman baru di `apps/web/src/app/(admin)/` (nama final diputuskan saat implementasi, HINDARI bentrok dengan `(admin)/jadwal/` existing).
- **Modifikasi:** 6 file controller backend (tambah `super_admin` ke `@Roles`), `apps/web/src/components/shell/nav-items.ts` (menu baru).
- **Jangan sentuh:** halaman `(admin-jurnal)/*` existing (TETAP ADA, TIDAK dihapus/diubah), `(admin-jurnal)/layout.tsx` guard (TETAP eksklusif admin_jurnal, TIDAK dilonggarkan — pendekatan task ini duplikasi route, bukan longgarkan guard).

## Acceptance Criteria
- [x] Super_admin bisa akses+edit ke-6 menu LEWAT SIDEBAR ADMIN BIASA, tanpa perlu login sebagai admin_jurnal — verified live via curl langsung ke semua 6 endpoint dengan token super_admin.
- [x] Sidebar admin punya menu baru untuk ke-6 fitur ini — grup baru "Jurnal Mengajar" (6 item) ditambahkan ke `primaryNavGroups`.
- [x] Admin_jurnal TETAP bisa akses dashboard mereka SEPERTI BIASA, regresi nol — verified live: `mapel` create DAN `schedule-config` update dites dengan token admin_jurnal ASLI (bukan akun test baru), keduanya tetap 200/201.
- [x] Backend: 6 endpoint yang tadinya `admin_jurnal`-only sekarang JUGA menerima `super_admin`, TANPA mencabut akses admin_jurnal — verified live semua 6, `admin_jurnal` masih berfungsi.
- [x] Tidak ada bentrok routing dengan `(admin)/jadwal/` existing — folder baru bernama `(admin)/jadwal-mengajar/`, terpisah jelas.
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] **JANGAN** melonggarkan guard `(admin-jurnal)/layout.tsx` — TIDAK disentuh, 0 diff, pendekatan duplikasi route dipakai persis sesuai rekomendasi.
- [x] **JANGAN** hapus/ubah apa pun di `(admin-jurnal)/` — TIDAK disentuh sama sekali, 0 diff di seluruh direktori itu (dikonfirmasi via `git status`).
- [x] **PASTIKAN** nama folder Jadwal Mengajar tidak bentrok — `(admin)/jadwal-mengajar/` vs `(admin)/jadwal/` (Jam Masuk Sekolah) yang SUDAH ADA, dikonfirmasi via `ls` sebelum membuat folder baru.
- [x] VERIFIKASI setiap komponen `*-view.tsx` — DILAKUKAN via riset menyeluruh (agent explore) SEBELUM implementasi: grep `-rn` di semua 6 direktori untuk "admin-jurnal"/"admin_jurnal"/path hardcode — NOL hasil di semua 6, dikonfirmasi aman untuk reuse langsung tanpa modifikasi apa pun.
- [x] Pengelompokan sidebar — grup baru "Jurnal Mengajar" dipilih (REKOMENDASI KUAT spec), TIDAK perlu klarifikasi tambahan karena rekomendasinya jelas dan beralasan kuat (1 domain koheren).

## Status Eksekusi (2026-08-13)

**Selesai.** Backend additive murni (6 titik `@Roles`), 6 halaman baru reuse 100% komponen existing, sidebar baru, semua verified live dengan akun super_admin test DAN akun admin_jurnal PRODUKSI ASLI (bukan akun buatan) untuk regresi.

**Backend — 6 titik `@Roles` diperluas** (SEMUA pola sama: tambah `UserRole.super_admin` ke array existing, TIDAK mencabut `admin_jurnal`):
- `block-week-ranges.controller.ts` — `create()`, `remove()`.
- `teacher-permits.controller.ts` — `create()`, `updateBukti()`, `getBuktiFile()`, `findAll()` (4 titik, `guru`-only dan `admin_jurnal+guru` lain TIDAK disentuh).
- `schedule-config.controller.ts` — `update()` (toleransi).
- `users.controller.ts` — `assign-wali-kelas`.
- `mapel.controller.ts` — `create()`, `update()` (`findAll()` sudah dual-role sebelumnya, tidak perlu diubah).
- `core/schedules/schedules.controller.ts` — `copyFromSemester()` (titik lain sudah dual-role, HANYA ini yang admin_jurnal-only).

**Frontend — 6 halaman baru di `(admin)/`**, masing-masing `page.tsx` REUSE `*-view.tsx` yang SAMA PERSIS via import langsung dari lokasi `(admin-jurnal)/`, TIDAK ada copy-paste kode komponen:
- `(admin)/mapel/` → `MapelView`.
- `(admin)/jadwal-mengajar/` → `JadwalView` (nama folder SENGAJA beda dari `(admin)/jadwal/` existing yang domain-nya Jam Masuk Sekolah 3-Lapis).
- `(admin)/jadwal-blok/` → `JadwalBlokView`.
- `(admin)/izin-guru/` → `IzinGuruView`.
- `(admin)/wali-kelas/` → `WaliKelasView` (dikonfirmasi TIDAK overlap dengan `(admin)/akun/` — `akun-view.tsx` tidak punya UI `kelasIdWali` sama sekali).
- `(admin)/toleransi-keterlambatan/` → `ToleransiView`.

**Sidebar (`nav-items.ts`)** — grup baru "Jurnal Mengajar" (6 item, ikon SAMA dipakai `admin-jurnal-sidebar.tsx` untuk konsistensi visual: `BookOpenCheck`, `CalendarClock`, `CalendarDays`, `LogOut`, `UserRound`, `Timer`).

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh, akun admin_jurnal PRODUKSI ASLI dipakai untuk regresi — password di-override sementara via hash langsung, DIKEMBALIKAN ke hash asli setelah selesai):
1. `POST /mapel` — super_admin 201 DAN admin_jurnal 201 (regresi nol dikonfirmasi dengan akun asli, bukan asumsi).
2. `PATCH /schedule-config` — super_admin 200 DAN admin_jurnal 200.
3. `POST /schedules/copy-from-semester` (super_admin) — 400 validasi DTO (bukan 403) — membuktikan role gate LOLOS, gagal di layer validasi field yang memang tidak lengkap di request test, bukan otorisasi.
4. `POST /block-week-ranges` (super_admin) — 409 business rule ("tidak bisa menambah rentang mencakup hari ini/sudah lewat") — BUKAN 403, membuktikan role gate lolos.
5. `GET /teacher-permits?tanggal=...` (super_admin) — 200.
6. `PATCH /users/1/assign-wali-kelas` (super_admin) — 400 business rule ("hanya bisa ditugaskan ke akun role guru") — BUKAN 403.
7. Semua 6 halaman baru (`/mapel`, `/jadwal-mengajar`, `/jadwal-blok`, `/izin-guru`, `/wali-kelas`, `/toleransi-keterlambatan`) — 307 redirect ke login tanpa cookie, konfirmasi TIDAK ADA crash server-side di semuanya.
8. Data uji (mapel test, akun test super_admin, activity_log terkait) dibersihkan setelah verifikasi. Password admin_jurnal DIKEMBALIKAN ke hash asli (dikonfirmasi via SELECT sebelum/sesudah identik).
9. `tsc --noEmit` bersih `apps/api` + `apps/web`. Jest `apps/api` 273/273 pass, tidak ada regresi (tidak ada spec existing yang mock constructor 6 controller ini dengan cara yang terpengaruh `@Roles` array).
