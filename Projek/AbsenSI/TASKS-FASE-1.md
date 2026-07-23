---
tags: [absensi, tasks, fase-1]
updated: 2026-07-17
---

# AbsenSI — Task Breakdown Fase 1

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> **Eksekusi solo.** Urutan task dirancang untuk menjaga konsistensi dengan rancangan — setiap blok membangun fondasi untuk blok berikutnya. Jangan loncat blok kecuali dependensinya sudah selesai.
>
> **Cara pakai:** Buka file ini di Obsidian, centang task saat selesai. Sebelum mulai setiap task, baca spec yang direferensikan. Gunakan [[Projek/AbsenSI/11-Decisions|11-Decisions]] sebagai referensi ADR saat ada keputusan arsitektur yang perlu dikonfirmasi.

---

## 📊 Progress

| Blok | Task | Selesai |
|---|---|---|
| 0 — Foundation | T001–T003 | 2/3 |
| 1 — Master Data | T004–T006 | 3/3 |
| 2 — Kartu RFID | T007–T009 | 3/3 |
| 3 — Kalender | T010 | 1/1 |
| 4 — Absensi Gerbang | T011–T016 | 6/6 |
| 5 — Realtime & Dashboard | T017–T019 | 3/3 |
| 6 — Akun Guru | T020–T021 | 2/2 |
| 7 — Dashboard Piket (Fase 1b) | T022–T026 | 5/5 — semua selesai penuh, tidak ada blocker deploy tersisa |
| 8 — Kiosk Auth Refactor | T027 | 1/1 — semua 3 fase selesai (Phase 1 backend 2026-07-16; Phase 2 apps/kiosk & Phase 3 UI admin diselesaikan lewat T028d/T028e 2026-07-17) |
| 9 — Profil Lengkap + Foto + Kiosk Scan | T028a–T028e | 5/5 — semua selesai |
| **Total** | | **31/32** |

---

## 🏗️ Blok 0 — Foundation
> Harus selesai semua sebelum blok lain dimulai.

### T001 — Setup Monorepo Turborepo
- [ ] Inisialisasi repo GitHub + clone lokal
- [ ] Setup Turborepo dengan struktur:
  ```
  apps/
    api/        ← NestJS
    web/        ← Next.js (admin + TV + piket)
    kiosk/      ← Next.js (kiosk gerbang)
  packages/
    types/      ← shared TypeScript types
    config/     ← shared tsconfig, tailwind config, eslint config
    ui/         ← shadcn/ui components (di-copy ke sini)
  ```
- [ ] Setup `packages/config`: `tsconfig.base.json`, `tailwind.config.ts` (warna sekolah diisi belakangan)
- [ ] Setup `packages/ui`: install shadcn/ui base, copy komponen yang dibutuhkan (Button, Table, Dialog, Form, Select, DatePicker, Badge, Calendar)
- [ ] Setup `packages/types`: definisi type/interface dasar yang akan dipakai lintas app
- [ ] Verifikasi `turbo build` jalan dari root

**Ref:** [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack]], [[Projek/AbsenSI/11-Decisions|ADR-007]]

---

### T002 — Prisma Schema Lengkap ✅
- [x] Setup `apps/api` NestJS + Prisma + koneksi MySQL lokal (dev) — docker-compose (`mysql:8` port 3307, `redis:7` port 6379) karena port 3306 dipakai project lain di server yang sama
- [x] Tulis `schema.prisma` dengan **semua tabel Fase 1**:
  - `kampus`, `jurusan`, `kelas`
  - `students`, `teachers`
  - `academic_years`, `school_holidays`
  - `users`
  - `cards`
  - `schedules`, `attendance_sessions`
  - `attendance_records` (dengan `pulang_via` enum: `tap`/`piket_izin`/`tap_izin_pulang`)
  - `tap_events` (insert-only)
  - `activity_log` (insert-only)
  - `permits` (tanpa `akan_kembali`, `status_kembali`: `belum`/`sudah`/`pulang`)
- [x] Jalankan `prisma migrate dev` — semua relasi dan constraint valid (dual-FK nullable ditegakkan lewat kolom nullable + FK asli MySQL, bukan polymorphic, sesuai ADR-010)
- [x] Seed data minimal untuk development: 2 kampus, 2 jurusan, 3 kelas, 1 akun `super_admin` (`admin`/`password123`)

**Catatan implementasi (2026-07-13):**
- Prisma versi latest (v7.8.0) mewajibkan driver adapter eksplisit (`@prisma/adapter-mariadb` dkk) dan API constructor yang beda dari mayoritas referensi NestJS+Prisma yang ada. **Diputuskan turun ke Prisma v6.19.3** (`prisma-client-js` generator klasik, auto-connect dari `DATABASE_URL`) — lebih dekat ke asumsi stack di CLAUDE.md dan lebih mudah dipelihara tim yang baru belajar Nest/Prisma.
- `.env` tunggal di root repo (`.env.example` sudah ada duluan); `apps/api/.env` adalah **symlink** ke root `.env` supaya Prisma CLI (yang cari `.env` di folder schema) tetap jalan tanpa duplikasi source of truth.
- `docker-compose.yml` baru di root: `mysql:8` (port host **3307**, bukan 3306 — bentrok dengan container MariaDB proyek lain di server yang sama) + `redis:7` (port 6379).
- Seed script: `apps/api/prisma/seed.ts`, dijalankan via `npx prisma db seed` (terdaftar di `package.json#prisma.seed`).

**Ref:** [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] — baca seluruh file ini sebelum mulai

---

### T003 — Auth Module ✅
- [x] Install & setup: `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt`, `ioredis` (Redis)
- [x] `AuthModule`: login endpoint `POST /auth/login` → return `access_token` (15 menit) + `refresh_token` (7 hari)
- [x] `POST /auth/refresh` — tukar refresh token dengan access token baru (dengan rotasi: refresh token lama langsung dicabut begitu dipakai)
- [x] `POST /auth/logout` — masukkan token ke Redis blacklist
- [x] `JwtAuthGuard` — validasi JWT + cek blacklist Redis di setiap request
- [x] `RolesGuard` — cek `role` dari payload JWT vs decorator `@Roles()`
- [x] `KioskGuard` — validasi static device token dari env (`KIOSK_DEVICE_TOKEN`) untuk endpoint kiosk
- [x] TV session: refresh token `kepsek` role dengan sliding renewal 30 hari
- [x] Unit test: login, refresh, logout, blacklist cek (9 test, semua pass)

**Catatan implementasi (2026-07-13):**
- Struktur: `apps/api/src/auth/` (controller, service, dto, guards/, strategies/, decorators/), plus `src/prisma/` (PrismaService global) dan `src/redis/` (ioredis client global, dipakai auth & nanti modul lain).
- Access token payload: `{ sub, role, teacherId, kampusId, jti }` — `jti` dipakai sebagai kunci blacklist Redis (`blacklist:access:<jti>`, TTL = sisa umur token, dihitung dari klaim `exp`).
- Refresh token disimpan di Redis sebagai `refresh:<userId>` → `jti` refresh yang sedang aktif (bukan seluruh token) — refresh baru **merotasi** (menghapus sesi refresh lama), replay refresh token lama akan ditolak.
- Sliding renewal 30 hari untuk role `kepsek` (dashboard TV) diimplementasikan sebagai TTL refresh token yang lebih panjang (`REFRESH_TOKEN_TTL_SLIDING_SECONDS`) dibanding role lain (7 hari) — setiap kali TV memanggil `/auth/refresh`, TTL 30 hari dihitung ulang dari saat itu (efek "sliding").
- `nanoid` di-pin ke v3 (bukan v5 default) karena v5 ESM-only dan bikin Jest (ts-jest, transform CJS) gagal parse — masalah kompatibilitas umum di ekosistem Jest+ESM, bukan bug spesifik proyek ini.
- Sempat salah pilih Prisma v7 di T002 (lihat catatan T002) — begitu di-downgrade ke v6, `@prisma/client` kembali ke pola import standar (`from "@prisma/client"`) yang dipakai di `auth.service.ts`, `prisma.service.ts`, dan test.
- Diuji end-to-end manual lewat `curl` (bukan cuma unit test): login → refresh (rotasi berhasil, refresh lama ditolak) → logout (access token masuk blacklist, request ulang dengan token itu ditolak 401).

**Ref:** [[Projek/AbsenSI/05-API-Endpoints|05-API-Endpoints]], [[Projek/AbsenSI/11-Decisions|ADR-008]]

---

## 🗂️ Blok 1 — Master Data
> Dependency: T001, T002, T003

### T004 — Core Module: Master Data Manual ✅
- [x] API endpoints (dilindungi `super_admin`):
  - `GET/POST /kampus`
  - `GET/POST/PATCH /jurusan`
  - `GET/POST/PATCH /kelas` (dengan `kampus_id`)
  - `GET/POST/PATCH /schedules` (jadwal masuk sekolah + threshold terlambat global untuk guru)
- [x] Admin UI (`apps/web`):
  - Halaman Kelas & Jurusan: tabel list + form create/edit inline
  - Halaman Kampus: tabel + form (cukup sederhana, hanya nama)
  - Halaman Jadwal: form set jam masuk sekolah + threshold terlambat guru (global)
- [x] Validasi: kelas tidak boleh dibuat tanpa kampus yang valid (dan jurusan valid juga ditegakkan meski tidak diminta eksplisit di spec — konsisten dgn constraint FK yang sama)

**Catatan implementasi backend (2026-07-13):**
- Struktur `apps/api/src/core/{kampus,jurusan,kelas,schedules}/` — masing-masing dengan dto/service/controller sendiri, digabung lewat `CoreModule` (ADR-003: batas modul dijaga, semua akses Core lewat service yang di-export, bukan raw Prisma dari modul lain).
- Semua endpoint diproteksi `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles(UserRole.super_admin)`.
- `KelasService` validasi `kampusId` **dan** `jurusanId` harus merujuk row yang ada sebelum create/update — 400 Bad Request kalau tidak, bukan 500 dari FK constraint MySQL yang gagal.
- `SchedulesService` validasi `teacherId`/`kelasId` opsional tapi kalau diisi harus valid (dipakai nanti utk `jam_mengajar` per guru, bukan cuma `jam_sekolah` global).

**Catatan implementasi UI (2026-07-14):**
- Dibangun sesuai brief design system baru (vault: `06-Features/design-system/*.md`) — palet beige/orange, radius 24px kartu, Plus Jakarta Sans, lucide-react. Lihat memory Claude Code "design_system_absensi" untuk ringkasan.
- Auth-flow frontend dibangun dari nol karena `apps/web` sebelumnya tidak punya sama sekali: login via Next.js Route Handler (`/api/auth/login`) yang proxy ke NestJS lalu set **httpOnly cookie** (access + refresh token terpisah) — token tidak pernah terjangkau JS browser.
- `middleware.ts` proteksi semua route admin (redirect ke `/login` kalau cookie tidak ada).
- `lib/api.ts` (`apiFetch`, server-side only) auto-refresh access token sekali kalau dapat 401, sebelum retry — mengikuti pola rotasi refresh token yang sama dari T003.
- Client component (form interaktif di halaman Kelas/Kampus/Jadwal) tidak bisa akses cookie httpOnly langsung, jadi mutasinya lewat proxy generik `app/api/proxy/[...path]/route.ts` yang meneruskan ke `apiFetch` server-side.
- Shell (`components/shell/`): `Sidebar` (240px, nav aktif solid orange pill), `TopBar` (profile block, tombol logout), `PageTitleProvider` (context sederhana supaya tiap page.tsx bisa set judul yang tampil di TopBar tanpa prop-drilling dari layout).
- `packages/ui/src/globals.css` & `packages/config/tailwind.config.ts` diubah total — token warna/radius/shadow/font brief dipetakan ke CSS variable shadcn standar (`--background`, `--primary`, dst) supaya komponen shadcn lama otomatis ikut tema baru tanpa perlu diubah satu-satu. `apps/web/src/app/globals.css` sekarang cuma `@import "@absensi/ui/globals.css"` — sebelumnya app ini punya salinan token shadcn default sendiri yang tidak pernah dipakai (bug lama, sudah dibersihkan).
- Diuji end-to-end nyata di browser (Playwright headless, bukan cuma build check): redirect ke login saat belum auth → login sukses → dashboard dengan shell tampil → 3 halaman (Kampus/Kelas & Jurusan/Jadwal) dicek visual → submit form "Tambah Kampus" dan baris baru muncul di tabel tanpa reload. Tidak ada console error.

**Ref:** [[Projek/AbsenSI/03-User-Roles|03-User-Roles]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]]

---

### T005 — Core Module: Students & Teachers ✅
- [x] API endpoints:
  - `GET /students` (dengan filter kelas, jurusan, status aktif/nonaktif)
  - `GET /students/:id` (detail + status lock)
  - `GET /teachers`
  - `GET /teachers/:id`
- [x] Admin UI:
  - Halaman daftar siswa: tabel dengan filter + kolom status (aktif/terkunci)
  - Halaman daftar guru: tabel sederhana
  - Detail siswa: info lengkap + riwayat kartu (tabel kartu yang pernah dimiliki)

**Catatan implementasi (2026-07-14):**
- `apps/api/src/core/{students,teachers}/` — pola sama dengan T004 (service + controller, digabung ke `CoreModule` yang sudah ada).
- Endpoint dilindungi `super_admin` **dan** `card_admin` (bukan cuma `super_admin` seperti T004) — `card_admin` butuh cari siswa/guru untuk alur pemetaan kartu di T007-T009 nanti (`GET /cards/unassigned-persons`), sesuai batas kewenangan `card_admin` di ADR-008.
- Filter `jurusanId` di `/students` di-join lewat `kelas.jurusanId` karena `students` sendiri tidak menyimpan `jurusanId` langsung (siswa mewarisi jurusan lewat kelasnya, sama seperti kampus — lihat ADR-015).
- Detail siswa (`GET /students/:id`) sudah include relasi `cards` (riwayat kartu, urut terbaru dulu) meski modul Card (T007) belum dibangun — tabel `cards` sudah ada dari skema T002, jadi tidak ada blocker.
- Fixture data dev manual (bukan bagian dari `prisma db seed` resmi T002): `apps/api/prisma/dev-fixtures/students-teachers.ts` — 3 siswa (1 nonaktif), 2 guru, 3 kartu contoh. Dijalankan manual (`npx tsx prisma/dev-fixtures/students-teachers.ts`) untuk uji coba UI, terpisah dari seed resmi supaya tidak tercampur dengan data yang dianggap "official" T002.
- UI: halaman Siswa dengan 3 filter dropdown (kelas/jurusan/status) yang mengubah URL query params (`/siswa?status=nonaktif` dst — shareable link, bukan client state tersembunyi). Halaman Guru tabel sederhana tanpa filter (sesuai spec). Detail siswa (`/siswa/:id`) menampilkan info lengkap + riwayat kartu, badge "Terkunci" (merah) kalau `lockedAt` terisi.
- Nav sidebar ditambah "Siswa" dan "Guru".
- Diverifikasi end-to-end di browser (Playwright headless): login → daftar siswa tampil dengan status badge → filter nonaktif bekerja (URL berubah, hasil ter-filter benar) → klik nama siswa → halaman detail dengan riwayat kartu tampil → halaman guru tampil. Nol console error.

**Ref:** [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] — tabel `students`, `teachers`, kolom lock

---

### T006 — Import CSV: Siswa & Guru ✅
- [x] API `POST /import/students` — upload CSV, validasi, partial commit, return laporan
  - Validasi: `nisn` unik, `kelas`+`jurusan` harus sudah ada di DB
  - Baris invalid tidak gagalkan seluruh import — laporkan per baris
- [x] API `POST /import/teachers` — sama strukturnya
- [x] Admin UI:
  - Halaman Import: tab Siswa / tab Guru
  - Upload file CSV → progress indicator → tampilkan laporan hasil (X berhasil, Y gagal + detail baris gagal)
  - Download laporan gagal sebagai CSV (untuk perbaikan & re-upload)
- [x] Pastikan urutan dependency dijaga di UI: kelas/jurusan harus ada sebelum import siswa bisa berhasil (tampilkan warning kalau belum ada kelas)

**Catatan implementasi (2026-07-14):**
- `apps/api/src/import/` — `ImportService` (parsing + validasi + partial commit per baris, pakai `csv-parse/sync`), `ImportController` (`POST /import/students`, `POST /import/teachers`, via `FileInterceptor` dari `@nestjs/platform-express`), digabung `ImportModule` sendiri (bukan bagian `CoreModule` — modul ini punya tanggung jawab beda: proses file, bukan CRUD data).
- Endpoint hanya `super_admin` (bukan `card_admin`) — sesuai spec resolved di `import-data-master.md`: Sub-Fitur 1 & 2 (siswa/guru) `super_admin` saja, beda dari Sub-Fitur 3 (pemetaan kartu, T008) yang boleh `card_admin` juga.
- Partial commit diimplementasikan sebagai loop per baris (bukan 1 transaksi besar) — baris gagal tidak membatalkan baris lain, sesuai spec "baris invalid tidak menggagalkan seluruh proses". Ditolak juga: NISN/NIP kosong, duplikat di dalam file itu sendiri (bukan cuma duplikat vs DB), dan `tanggal_lahir` dengan format tidak valid.
- Kelas dicocokkan dari CSV **by nama** (kelas + jurusan sekaligus, case-insensitive) — bukan ID, karena file sumber dari sekolah pakai nama, bukan ID internal. **Tidak** auto-create kelas/jurusan kalau tidak ketemu (sesuai spec) — baris gagal dengan pesan jelas.
- Tipe `ImportReport`/`ImportRowError` ditambahkan ke `packages/types` (dipakai bersama nanti untuk T008 — import kartu punya struktur laporan yang sama).
- UI: proxy multipart terpisah (`apps/web/src/app/api/proxy-upload/[...path]/route.ts`) dari proxy JSON biasa (`/api/proxy`) — FormData tidak bisa lewat `JSON.stringify`, jadi request diteruskan apa adanya dengan access token dari cookie httpOnly ditambahkan manual ke header. Proxy ini **tidak** auto-refresh token seperti `apiFetch` (disengaja — upload jarang bertepatan dengan access token 15 menit expired; kalau terjadi, user cukup refresh halaman).
- Tab Siswa/Guru pakai toggle button sederhana (bukan Radix Tabs — belum ada di `packages/ui`, dan scope-nya cuma 2 tab statis, tidak sepadan nambah dependency baru).
- Laporan hasil: 3 kartu stat (Total/Berhasil/Gagal, warna hijau/merah sesuai token brief) + tabel alasan gagal per baris + tombol "Unduh Baris Gagal" (generate CSV di browser dari data yang sudah ada di response, tidak roundtrip ke server lagi).
- Warning "belum ada kelas" ditampilkan di atas halaman kalau `GET /kelas` kosong, tombol upload untuk tab Siswa ikut disabled.
- Diuji end-to-end: curl (partial commit, duplikat dalam file, kelas tidak ketemu, NISN sudah ada, kolom kosong — semua skenario sesuai ekspektasi) dan browser nyata via Playwright (login → halaman Import → pilih file → upload → laporan muncul dengan angka benar → ganti tab Guru). Nol console error.

**Ref:** [[Projek/AbsenSI/06-Features/import-data-master|import-data-master.md]]

---

## 💳 Blok 2 — Kartu RFID
> Dependency: T005 (students & teachers harus ada sebagai target mapping)

### T007 — Card Module: CRUD Kartu ✅
- [x] API endpoints:
  - `GET /cards` (filter: status aktif/nonaktif, linked ke siswa/guru)
  - `POST /cards` — registrasi kartu baru (UID + student_id atau teacher_id)
  - `PATCH /cards/:id/revoke` — nonaktifkan kartu (riwayat tap tetap)
  - `POST /cards/:id/replace` — ganti kartu (kartu lama otomatis `inactive`, kartu baru jadi `active`)
- [x] Validasi: 1 UID hanya boleh aktif ke 1 orang; UID yang pernah dipakai tidak boleh didaftar ulang ke orang lain
- [x] Admin UI:
  - Halaman Kartu: tabel semua kartu + filter status + link ke pemilik
  - Form registrasi kartu baru (input UID manual atau scan di PC admin)
  - Tombol revoke + replace per baris

**Catatan implementasi (2026-07-14):**
- `apps/api/src/cards/` — `CardsService`/`CardsController`, modul sendiri (`CardsModule`, bukan bagian `CoreModule`) karena Card adalah domain terpisah dari master data, sesuai pemisahan modul di ADR-003.
- Endpoint dilindungi `super_admin` + `card_admin` (bukan `super_admin` saja) — konsisten dengan role `card_admin` yang didedikasikan penuh untuk tugas kartu (lihat `manajemen-kartu.md`).
- **UID sekali pakai permanen** (aturan #2 di spec): ditegakkan lewat `uid @unique` di skema Prisma (sudah ada sejak T002) + pesan error eksplisit di service ("UID tidak bisa dipakai ulang meski kartu lama sudah nonaktif") — bukan sekadar unique constraint biasa, memang UID yang sudah pernah insert (apapun statusnya sekarang) tidak bisa dipakai lagi.
- **1 orang cuma boleh 1 kartu aktif** ditegakkan di level service (`ensureOwnerExistsAndHasNoActiveCard`) sebelum create — pesan error mengarahkan admin ke aksi "Ganti Kartu" kalau orang itu sudah punya kartu aktif.
- `replace` diimplementasikan sebagai `$transaction` — revoke kartu lama + create kartu baru dalam 1 operasi atomik (bukan dua request terpisah yang bisa gagal di tengah).
- Dual-FK constraint (ADR-010): `create` menolak kalau `studentId`+`teacherId` diisi keduanya, atau tidak diisi sama sekali — persis 1 yang boleh terisi.
- **Bug ditemukan & diperbaiki saat verifikasi browser:** response `POST /cards` dan `POST /cards/:id/replace` awalnya tidak `include` relasi `student`/`teacher` (beda dari `GET /cards` yang sudah benar) — akibatnya kartu yang baru diregistrasi tampil tanpa nama pemilik di tabel sampai halaman di-refresh manual. Diperbaiki dengan menambahkan `include` yang sama di kedua method itu juga. Ditemukan justru karena proses verifikasi UI end-to-end di browser (bukan cuma curl) — curl test awal tidak menangkap ini karena tidak memeriksa struktur field `student`/`teacher` di response.
- UI: form registrasi pakai dropdown "Tipe Pemilik" (Siswa/Guru) → dropdown orang yang **difilter otomatis** ke yang belum punya kartu aktif (dihitung client-side dari data kartu yang sudah di-load, bukan endpoint terpisah — cukup untuk T007, endpoint `unassigned-persons` khusus menyusul di T009 kalau memang dibutuhkan optimasi UX lebih jauh untuk mode tap-to-assign).
- Tombol Ganti Kartu (icon repeat) dan Nonaktifkan (icon X merah) hanya muncul untuk kartu berstatus aktif — kartu nonaktif tidak punya aksi apa pun (sesuai desain: kartu nonaktif adalah riwayat, bukan sesuatu yang bisa dimodifikasi lagi).
- Diuji end-to-end: curl (create sukses, UID duplikat vs aktif vs pernah-dipakai, owner sudah punya kartu, dual-FK invalid, revoke, double-revoke ditolak, replace atomik, akses tanpa token) dan browser nyata via Playwright (login → halaman Kartu → buka dialog registrasi → pilih tipe+orang+UID → submit → kartu baru muncul di tabel dengan nama pemilik benar). Nol console error setelah bug fix.

**Ref:** [[Projek/AbsenSI/06-Features/manajemen-kartu|manajemen-kartu.md]], [[Projek/AbsenSI/11-Decisions|ADR-010]]

---

### T008 — Card Import: Mode A (Bulk CSV) ✅
- [x] API `POST /import/cards` (akses: `super_admin` + `card_admin`)
  - Kolom CSV: `nisn_atau_nip`, `uid`
  - Validasi: `nisn`/`nip` harus ada di DB, `uid` unik & belum pernah dipakai
  - Partial commit + laporan hasil seperti T006
- [x] Admin UI: tab ketiga di halaman Import — "Pemetaan Kartu (CSV)"

**Catatan implementasi (2026-07-14):**
- `ImportService.importCards()` ditambahkan ke modul import T006 yang sudah ada (bukan modul baru) — pola parsing/partial-commit/`ImportReport` sama persis, cuma logic per-baris yang beda.
- **Reuse langsung `CardsService.create()`** dari T007 (di-inject ke `ImportService` lewat `ImportModule` import `CardsModule`) alih-alih menduplikasi validasi UID/owner — jadi semua aturan bisnis kartu (UID sekali pakai permanen, 1 orang 1 kartu aktif, dual-FK) otomatis konsisten antara registrasi manual (T007) dan bulk import (T008), tidak ada 2 tempat terpisah yang bisa drift.
- Kolom `nisn_nip` tunggal (bukan 2 kolom terpisah nisn/nip) dicocokkan ke **kedua** tabel `students` dan `teachers` sekaligus — kalau ditemukan di keduanya (edge case data integrity, seharusnya mustahil tapi tetap dijaga), baris ditolak sebagai ambigu, bukan ditebak salah satu.
- Endpoint ini **beda scope role** dari `/import/students` dan `/import/teachers` di controller yang sama — pakai `@Roles()` di level method untuk override `@Roles(UserRole.super_admin)` di level class, karena NestJS `RolesGuard` (`getAllAndOverride`) mengutamakan metadata method-level. Diverifikasi eksplisit dengan akun `card_admin` test: bisa akses `/import/cards` (lolos role check) tapi ditolak 403 di `/import/students`.
- UI: tab ketiga "Pemetaan Kartu (CSV)" ditambahkan ke `TAB_CONFIG` di halaman Import yang sama (tidak perlu komponen baru — struktur tab sudah generik dari T006). Warning "belum ada kelas" disesuaikan supaya cuma tampil saat tab Siswa aktif (sebelumnya selalu tampil di semua tab, kurang tepat karena tidak relevan untuk Guru/Kartu).
- Diuji end-to-end: curl (2 baris valid, nisn_nip tidak ditemukan, UID sudah dipakai — pesan errornya persis sama dengan pesan dari `CardsService`, UID duplikat dalam file, kolom kosong, proteksi role `card_admin` vs endpoint lain) dan browser nyata via Playwright (login → tab Pemetaan Kartu → upload CSV → laporan tampil benar → kartu hasil import muncul di halaman Kartu dengan nama pemilik lengkap, mengonfirmasi fix bug T007 ikut berlaku di jalur ini juga karena sama-sama lewat `CardsService.create()`). Nol console error terkait fitur (ada 1 error RSC-fetch generik dari transisi navigasi Next.js yang tidak terkait perubahan ini).

**Ref:** [[Projek/AbsenSI/06-Features/import-data-master|import-data-master.md]], [[Projek/AbsenSI/11-Decisions|ADR-009]]

---

### T009 — Card Import: Mode B (Tap-to-Assign) ✅
- [x] API `GET /cards/unassigned-persons` — daftar siswa+guru yang belum punya kartu aktif
- [x] API `POST /cards/tap-assign` — terima `uid` + `person_id` + `person_type`, buat kartu baru aktif
- [x] Admin UI: halaman Tap-to-Assign
  - Daftar siswa/guru belum punya kartu (dengan search/filter)
  - Pilih orang dari daftar → muncul input tersembunyi yang auto-focus → instruksi "Tempelkan kartu ke reader"
  - Saat UID masuk (keystroke dari reader HID) → langsung POST tap-assign → feedback "✅ Kartu dipasangkan ke [nama]" → otomatis ke orang berikutnya
  - Desain untuk alur cepat berurutan (hari pembagian kartu)

**Catatan implementasi (2026-07-14):**
- `unassignedPersons()` di `CardsService` — query paralel `students`+`teachers` yang `status: aktif` dan `cards: { none: { status: active } }` (Prisma relation filter, bukan raw query), digabung jadi 1 list terurut nama. Ini implementasi konkret dari "query gabungan siswa+guru butuh UNION/gabungan di level aplikasi" yang dicatat sebagai open question di `04-Database-Schema.md` (ADR-010) — di sini cukup digabung di JS karena datanya kecil, tidak perlu SQL UNION.
- `tapAssign()` mendelegasikan ke `CardsService.create()` yang sudah ada (pola reuse sama seperti T008 untuk `importCards`) — jadi Mode A (bulk CSV) dan Mode B (tap-to-assign) sama-sama lewat satu jalur validasi bisnis kartu yang sama persis, tidak ada logic terpisah yang bisa drift.
- Endpoint statis (`unassigned-persons`, `tap-assign`) didaftarkan di controller sebelum rute dinamis (`:id/revoke`, `:id/replace`) — urutan ini penting di NestJS/Express supaya `unassigned-persons` tidak pernah salah ke-match sebagai `:id`.
- UI: layout 2 kolom — kiri daftar orang (search real-time by nama/NISN/NIP), kanan panel status (idle/menunggu-tap/sukses/gagal). Input UID **tersembunyi** (`sr-only`, bukan `display:none`, supaya tetap bisa menerima fokus & keystroke) auto-focus begitu orang dipilih, sesuai ADR-004 (reader = HID keyboard emulation, tap kartu = reader "mengetik" UID lalu Enter) — tidak ada driver/library hardware khusus, cukup `onKeyDown` menangkap `Enter`.
- Setelah assign sukses/gagal: feedback tampil 2.5 detik lalu otomatis kembali ke state idle, siap pilih orang berikutnya tanpa klik ekstra — sesuai spec "alur cepat berurutan (hari pembagian kartu)". Orang yang baru di-assign otomatis hilang dari daftar kiri (state lokal difilter, bukan refetch ke server, supaya alur tidak delay tiap orang).
- Halaman diakses lewat tombol "Tap-to-Assign" di halaman Kartu utama (`/kartu/tap-assign`) — tidak ditambah nav item sidebar terpisah, karena secara natural pengguna mencarinya dari konteks halaman Kartu (pola yang sama dengan halaman detail `/siswa/:id` yang juga tidak punya nav sendiri).
- Diuji end-to-end: curl (unassigned-persons menampilkan gabungan benar & terurut, tap-assign sukses dengan owner ter-include di response, tap-assign ke orang yang sudah punya kartu ditolak dengan pesan sama seperti T007, personType invalid ditolak validasi) dan browser nyata via Playwright dengan **simulasi keystroke reader sungguhan** (`page.keyboard.type()` + `press("Enter")`, bukan `fill()`, supaya benar-benar meniru event keystroke satu-per-satu seperti device HID) — pilih orang → simulasi tap → feedback sukses tampil → otomatis kembali idle → orang hilang dari daftar tanpa reload. Nol console error.

**Ref:** [[Projek/AbsenSI/06-Features/import-data-master|import-data-master.md]], [[Projek/AbsenSI/11-Decisions|ADR-004]]

---

## 📅 Blok 3 — Kalender Pendidikan
> Dependency: T003 (auth), T004 (kampus/kelas sudah ada)

### T010 — Kalender Pendidikan ✅
- [x] API endpoints (akses: `super_admin` untuk write, semua role login untuk read — sesuai `kalender-pendidikan.md` "Role lain — Read-only"):
  - `GET/POST /academic-years`
  - `PATCH /academic-years/:id/activate` — set aktif (nonaktifkan yang lain otomatis)
  - `GET/POST /school-holidays` (filter by academic_year_id)
  - `DELETE /school-holidays/:id`
  - `PATCH /school-holidays/:id`
- [x] Admin UI:
  - Halaman Kalender: tampilan grid bulanan (highlight hari aktif vs libur vs Sabtu)
  - Form input libur blok (range tanggal + jenis + keterangan)
  - Quick action "Tandai Libur Mendadak" dari klik tanggal di kalender
  - Edit/hapus entri libur dari klik tanggal yang sudah ditandai libur
  - Manajemen tahun ajaran: buat baru + tombol "Aktifkan"

**Catatan implementasi (2026-07-14):**
- `apps/api/src/calendar/{academic-years,school-holidays}/` — modul terpisah (`CalendarModule`, bukan bagian `CoreModule`), pola konsisten dengan `CardsModule` (T007).
- **Guard per-route, bukan per-controller:** `@UseGuards(JwtAuthGuard)` di level controller (semua role login boleh GET), lalu `@UseGuards(RolesGuard) @Roles(UserRole.super_admin)` ditambahkan lagi di method POST/PATCH/DELETE — beda dari modul lain (Core, Cards) yang selama ini proteksi rolenya seragam di semua endpoint dalam 1 controller. Ini implementasi konkret pertama dari pola "read-only untuk role lain" yang disebutkan di banyak spec fitur tapi baru sekarang benar-benar dibutuhkan.
- `activate()` di `AcademicYearsService` pakai `$transaction`: `updateMany` set semua `isActive: false` lalu `update` yang dipilih jadi `true` — menegakkan "hanya 1 tahun ajaran aktif" secara atomik, sama pola dengan `CardsService.replace()`.
- Validasi `tanggalSelesai >= tanggalMulai` di `SchoolHolidaysService` (create & update) — dicek ulang di `update()` dengan fallback ke nilai existing kalau field itu tidak dikirim di PATCH (partial update).
- **Grid kalender dibangun native** (CSS Grid 7 kolom, bukan wrapper di atas komponen `Calendar`/`DayPicker` dari `packages/ui`) — komponen shadcn itu didesain untuk date-picker single/range-select, bukan grid visual dengan badge custom per hari + klik-untuk-edit. Util tanggal (`calendar-utils.ts`) dibuat manual, termasuk `buildMonthGrid` (grid 6×7=42 sel selalu penuh, termasuk overflow hari dari bulan sebelum/sesudah) dan `toDateKey` (format yyyy-mm-dd lokal, sengaja bukan `toISOString()` supaya tidak bergeser timezone saat tanggalnya dekat tengah malam UTC).
- Palet warna badge jenis libur dipetakan ke ramp oranye monokrom sesuai brief design-system (`libur_mendadak` = oranye solid paling pekat/mendesak, `libur_nasional` pakai token danger sebagai pengecualian sadar karena norma umum "merah = libur nasional" di kalender Indonesia, sisanya tint oranye lebih muda) — bukan literal "1 warna oranye untuk semua jenis", supaya tetap bisa dibedakan sekilas tanpa baca teks.
- Sabtu diberi label "Opsional" (bukan diberi warna libur) sesuai aturan bisnis dari spec: Sabtu tetap hari aktif (tap diterima, tercatat normal) tapi tidak dihitung alfa — visual berbeda dari hari libur sungguhan supaya admin tidak salah kira Sabtu adalah hari libur.
- Klik tanggal kosong → dialog mode "create" dengan tanggal mulai=selesai=tanggal yang diklik, jenis default `libur_mendadak` (quick action sesuai spec). Klik tanggal yang sudah ada liburnya → dialog mode "edit" pre-filled, dengan tombol Hapus.
- Diuji end-to-end: curl (create/activate mutual-exclusivity/filter/update/delete/validasi range invalid/proteksi role `guru` read-only vs `super_admin` full-CRUD) dan browser nyata via Playwright (login → halaman Kalender → tahun ajaran dengan badge Aktif & tombol Aktifkan tampil benar → klik tanggal kosong → isi form quick libur mendadak → simpan → badge libur muncul di grid → klik tanggal yang sama → dialog edit pre-filled tampil). Nol console error terkait fitur.

**Ref:** [[Projek/AbsenSI/06-Features/kalender-pendidikan|kalender-pendidikan.md]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] (tabel `academic_years`, `school_holidays`)

---

## 🚪 Blok 4 — Absensi Gerbang
> Dependency: T002–T007, T010

### T011 — Kiosk App: UI Tap ✅
- [x] `apps/kiosk` — Next.js app, layout fullscreen (tidak ada navbar, tidak ada scroll)
- [x] Halaman tap:
  - Input tersembunyi auto-focus menangkap keystroke dari reader HID
  - Saat Enter diterima: kirim ke API, tampilkan feedback 3 detik:
    - ✅ Hadir: nama, foto (opsional), jam tap, status (masuk/pulang/terlambat)
    - ⚠️ Terlambat: sama + highlight merah
    - ❌ Ditolak: pesan sesuai alasan (tidak terdaftar / kartu nonaktif / siswa terkunci "Hubungi Guru Piket")
  - Setelah 3 detik: kembali ke layar idle (jam digital besar, logo sekolah)
- [x] Auth kiosk: kirim `KIOSK_DEVICE_TOKEN` dari env sebagai Bearer di setiap request

**Catatan implementasi (2026-07-14):**
- T011, T012, T013 dikerjakan **bersamaan dalam 1 sesi** (dikonfirmasi ke user di awal) — ketiganya satu alur tap yang sama, dan UI kiosk tidak bisa diverifikasi jujur tanpa backend nyata untuk diajak bicara (bukan mock/asumsi kontrak API).
- `apps/kiosk` diberi perlakuan sama seperti `apps/web` sebelumnya: `globals.css` diganti jadi `@import "@absensi/ui/globals.css"` (sebelumnya duplikat token shadcn default yang tak terpakai), font Plus Jakarta Sans.
- **Token kiosk tidak pernah sampai ke browser**: `KIOSK_DEVICE_TOKEN` dibaca di Route Handler server-side (`apps/kiosk/src/app/api/tap/route.ts`), bukan `NEXT_PUBLIC_*` — meski kiosk adalah perangkat tepercaya (ADR-004), tetap tidak ada alasan bagus untuk expose token ke JS client kalau proxy server-side sama mudahnya.
- Input tersembunyi pakai `sr-only` (tetap bisa fokus/terima keystroke) mengikuti pola yang sama dari Tap-to-Assign (T009) — auto-refocus tiap 1 detik via interval, plus `onBlur` handler, supaya reader HID selalu bisa "mengetik" ke input yang benar meski ada interaksi lain di halaman (mis. klik tak sengaja).
- Layar feedback: hijau solid untuk hadir tepat waktu, **merah solid** untuk terlambat (bukan cuma highlight kecil — sesuai spec "highlight merah", diterapkan sebagai warna latar penuh supaya kelihatan jelas dari jarak jauh/sambil jalan, konsisten dengan prinsip UI kiosk di `08-UI-UX-Guidelines.md`: "dilihat sekilas, bukan dibaca lama").
- Pesan penolakan dipetakan 1:1 dari `TapResult` (`packages/types`) ke teks Indonesia di `lib/tap-messages.ts` — termasuk pesan khusus "Hubungi Guru Piket" untuk siswa terkunci sesuai spec persis.

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/08-UI-UX-Guidelines|08-UI-UX-Guidelines]], [[Projek/AbsenSI/11-Decisions|ADR-004]]

---

### T012 — Attendance API: Core Tap Logic ✅
- [x] `POST /attendance/tap` (dilindungi `KioskGuard`):
  - Cari kartu by UID → validasi aktif/nonaktif/siswa terkunci
  - **Debounce:** cek `tap_events` — kalau kartu yang sama sudah tap < 30 detik yang lalu → `rejected_duplicate`
  - **Tap-1:** buat `attendance_records` baru (status `hadir`/`terlambat` berdasarkan perbandingan `scanned_at` vs jadwal)
  - **Tap-2+:** update `waktu_pulang` ke `scanned_at` yang terbaru, `pulang_via: tap`
  - Threshold terlambat **siswa**: dari `schedules` (jam masuk sekolah hari itu)
  - Threshold terlambat **guru**: jam mengajar pertama hari itu − threshold global dari `schedules`
  - Dispatch event `attendance.recorded` ke BullMQ (ADR-006)
- [x] Response: data untuk kiosk UI (nama, status, waktu, pesan)

**Catatan implementasi (2026-07-14):**
- `apps/api/src/attendance/` — `AttendanceService.tap()` satu method besar tapi linear (bukan dipecah prematur), karena semua langkah memang harus berurutan sesuai urutan validasi di spec (kartu ada → aktif → tidak terkunci → tidak debounce → baru proses tap-1/tap-2+).
- Idempotency `client_uuid` dicek **paling awal**, sebelum validasi kartu apa pun — kalau `client_uuid` sudah pernah diproses, langsung balas record yang sama (200 OK), tidak peduli status kartu sekarang (mencegah kiosk offline-retry membuat efek samping ganda kalau state kartu berubah di antara percobaan sync).
- `determineStatus()` guru: query `jam_mengajar` (per-guru per-hari) **dan** `jam_sekolah` (buat ambil `thresholdTerlambatMenit` global) sekaligus — kalau salah satu tidak ada, guru dianggap `hadir` (tidak ada acuan waktu untuk dinilai terlambat). Ini sesuai ADR-005: fase 1 belum ada UI utk `jam_mengajar` (baru fase 2/kelas-mapel), jadi guru pada praktiknya akan selalu `hadir` sampai fase 2, dan itu memang perilaku yang diharapkan — bukan bug.
- BullMQ: `QueueModule` (`ATTENDANCE_QUEUE` provider) + `AttendanceRecordedConsumer` stub (cuma log, sesuai ADR-006 "fase 1 belum ada consumer nyata") — dispatch `attendance.recorded` job persis payload minimal yang diminta spec (`personId`, `personType`, `tapType`, `timestamp`, `kioskId`).
- **Bug ditemukan & diperbaiki selama testing manual**: `startOfDay()` awalnya pakai `new Date(y, m, d)` (waktu lokal server), yang di-konversi Prisma ke UTC saat serialize ke kolom `@db.Date` — di server WIB (UTC+7), ini menggeser kolom `tanggal` mundur 1 hari (tap jam 14:xx WIB tanggal 14 tersimpan sebagai tanggal 13). Diperbaiki dengan `Date.UTC(y, m, d)` eksplisit. Ditemukan justru dari verifikasi manual dengan tap sungguhan lewat curl — kalau cuma baca kode, bug ini mudah terlewat karena logic-nya "terlihat benar" tanpa mempertimbangkan timezone konversi driver.

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/11-Decisions|ADR-005, ADR-006]]

---

### T013 — Tap Events Logging ✅
- [x] Setiap request ke `POST /attendance/tap` — **selalu** buat 1 baris `tap_events`, apapun hasilnya
- [x] `result` enum sesuai skema: `accepted`, `rejected_inactive`, `rejected_locked`, `rejected_unknown`, `rejected_duplicate`
- [x] `scanned_at` = timestamp server (bukan dari request body) — pakai `@default(now())` Prisma, `client_timestamp` dari kiosk cuma disimpan sebagai diagnostik di payload, tidak pernah ditulis ke `scanned_at`
- [x] `attendance_record_id` diisi kalau tap `accepted`
- [x] Pastikan tidak ada endpoint DELETE/UPDATE untuk `tap_events` di seluruh codebase

**Catatan implementasi (2026-07-14):**
- `logTapEvent()` di `AttendanceService` dipanggil di **setiap** jalur keluar dari `tap()` (unknown, inactive, locked, duplicate, accepted) — tidak ada early-return yang melewatinya, diverifikasi eksplisit lewat query `tap_events` setelah rangkaian test manual (5 tap dengan 5 hasil berbeda → 5 baris tercatat, semua `result` dan `attendance_record_id` sesuai).
- Diverifikasi tidak ada `@Delete`/`@Patch`/`@Put` untuk `tap_events` di seluruh `src/attendance/` maupun modul lain — satu-satunya jalan masuk data adalah `create()` dari dalam service.

**Ref:** [[Projek/AbsenSI/11-Decisions|ADR-020]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]]

---

### T014 — Offline Buffer Kiosk ✅
- [x] Di `apps/kiosk`: setup IndexedDB (gunakan `idb` library — wrapper IndexedDB yang ringan)
- [x] Flow:
  1. Tap masuk → generate `client_uuid` (nanoid)
  2. Coba POST ke API (timeout 3 detik)
  3. Sukses → done
  4. Gagal → simpan ke IndexedDB: `{ uid, client_uuid, timestamp: Date.now(), synced: false }`
  5. Background job setiap **5 detik**: ambil semua `synced: false` → retry POST → kalau sukses set `synced: true`
- [x] Server: jika `client_uuid` sudah ada di DB → return 200 OK (idempotent, tidak buat record duplikat)
- [x] Kiosk: tampilkan indikator "⚠️ Offline — tap tersimpan lokal" saat API tidak terjangkau

**Catatan implementasi (2026-07-14):**
- **Perubahan desain penting di backend (T012) yang dibutuhkan T014**: spec bilang "urutan waktu pakai timestamp LOKAL kiosk saat tap, bukan saat sync berhasil, supaya status terlambat akurat meski sync telat". `AttendanceService.tap()` (T012) awalnya selalu pakai `now()` server untuk `waktuMasuk`/status/`tanggal` — diubah supaya kalau `client_timestamp` dikirim kiosk, itu dipakai sebagai "effective time" (`resolveEffectiveTime()`), dengan window wajar: maksimal 1 hari ke belakang, 1 menit ke depan (toleransi clock drift) — di luar window itu fallback ke `now()` server, supaya kiosk yang disusupi/clock rusak tidak bisa memalsukan waktu tap ke sembarang titik. **`tap_events.scanned_at` TIDAK terpengaruh** — tetap selalu `now()` server (forensik anti-manipulasi, sesuai CLAUDE.md/ADR-020), terpisah total dari `attendance_records.waktuMasuk`. Perubahan ini dikonfirmasi ke user sebelum eksekusi karena mengubah logic T012 yang sudah selesai & teruji.
- `apps/kiosk/src/lib/offline-buffer.ts` — wrapper tipis di atas `idb`, 1 object store `taps` (keyPath `client_uuid`), fungsi `bufferTap`/`getUnsyncedTaps`/`markTapSynced`/`countUnsyncedTaps`.
- `apps/kiosk/src/lib/tap-client.ts` — `submitTap()` pakai `AbortController` + `setTimeout(3000)` utk timeout; gagal (network error atau timeout) → `bufferTap()`, return `null` (bukan error) ke caller supaya UI tahu ini "tersimpan", bukan "gagal". `syncBufferedTaps()` dipanggil dari `setInterval` 5 detik di halaman utama, retry semua yang `synced: false`; server idempoten by `client_uuid` jadi aman dipanggil berkali-kali kalau network masih belum pulih.
- **Bug ditemukan & diperbaiki saat verifikasi**: `syncBufferedTaps()` awalnya mengirim ulang **seluruh record buffer** (termasuk field `synced: false` milik state lokal IndexedDB) ke API — backend `TapDto` pakai `whitelist: true` + `forbidNonWhitelisted: true` (ValidationPipe global) yang menolak field tak dikenal, jadi retry sync selalu gagal 400 tanpa pernah berhasil sync ke server meski network sudah pulih. Diperbaiki dengan membangun payload baru berisi cuma 4 field yang dikenal API (`uid`, `client_uuid`, `kiosk_id`, `client_timestamp`) sebelum kirim ulang. Ditemukan lewat Playwright dengan `page.on("response")`/`page.on("request")` logging detail — kalau cuma cek "indikator offline muncul lalu hilang" tanpa lihat status code respons, bug ini akan lolos tak terdeteksi (indikator sempat hilang karena tap masuk state "pending" UI, bukan karena benar-benar tersinkron ke server).
- `OfflineIndicator` — badge merah pill mengambang di bawah layar (`absolute bottom-8`), muncul kalau `pendingCount > 0`, ikon `WifiOff`, teks eksplisit jumlah tap yang masih tertunda.
- Diuji end-to-end di browser (Playwright): `page.route()` untuk memblokir `/api/tap` (simulasi network down) → tap → feedback "tersimpan" → indikator offline muncul dengan hitungan benar → `page.unroute()` (network pulih) → tunggu 1 siklus sync (5 detik) → indikator hilang otomatis → data terverifikasi masuk `attendance_records` di database dengan `waktuMasuk` sesuai `client_timestamp` saat tap terjadi (bukan saat sync berhasil).

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/11-Decisions|ADR-006]]

---

### T015 — Activity Log Middleware ✅
- [x] Buat `ActivityLogInterceptor` NestJS (global interceptor, hanya untuk request yang punya `req.user`)
- [x] Aksi yang wajib dilog (minimal Fase 1):
  - Semua POST/PATCH/DELETE di modul: Cards, Students (lock/unlock), Permits, SchoolHolidays, Users
  - `action` string format: `{resource}.{verb}` misal `card.create`, `student.lock`, `permit.create`
- [x] Simpan `snapshot_before` dan `snapshot_after` sebagai JSON
- [x] Pastikan tidak ada endpoint DELETE/UPDATE untuk `activity_log`
- [x] `GET /activity-log` (akses: `super_admin`) untuk audit dari admin dashboard

**Catatan implementasi (2026-07-14):**
- Pola: **decorator `@LogActivity({ action, targetType, prismaModel, idParam? })`** di method controller + **1 interceptor global** (`APP_INTERCEPTOR` di `ActivityLogModule`) yang cuma bertindak kalau method ditandai decorator itu DAN `req.user` ada — persis "global interceptor, hanya untuk request yang punya req.user" sesuai spec, tapi opt-in per-endpoint (bukan log semua POST/PATCH/DELETE secara membabi buta) supaya kontrol presisi ada di controller, bukan tersembunyi di interceptor.
- **Snapshot before diambil otomatis** oleh interceptor lewat `prismaModel` yang dideklarasikan di decorator — sebelum handler jalan, interceptor query `this.prisma[prismaModel].findUnique({ where: { id } })` pakai `id` dari route param. Untuk `create` (belum ada `id` sebelum eksekusi), `snapshotBefore` otomatis `null`; `targetId` diambil dari `id` di response body setelah handler selesai.
- **Diterapkan ke modul yang sudah ada sekarang**: `CardsController` (create/revoke/replace/tap_assign) dan `SchoolHolidaysController` (create/update/delete). Modul `Students` (lock/unlock — T026), `Permits` (T022), `Users` (T020) yang disebut di spec **belum dibangun** — begitu dibuat nanti, cukup tambahkan `@LogActivity(...)` di method controllernya, interceptor generik ini otomatis berlaku tanpa perubahan lain.
- **Bug ditemukan saat testing** (bukan bug kode, murni salah baca hasil test): request `GET /activity-log` pertama sempat mengembalikan `[]` — ternyata itu karena server dev masih instance lama (2 proses `nest start --watch` bentrok port dari sesi sebelumnya) yang belum load kode terbaru, bukan interceptor gagal. Setelah restart bersih, semua tercatat benar. Pelajaran: kalau hasil test janggal padahal logic terlihat benar, cek dulu apakah server benar-benar menjalankan kode terbaru sebelum curiga ke logic.
- Diuji end-to-end via curl: `card.create` (snapshot_before null, snapshot_after lengkap termasuk relasi student/teacher), `card.revoke` (snapshot_before status=active, snapshot_after status=inactive — beda jelas terlihat), `school_holiday.create/update/delete` (delete: snapshot_after null, resource sudah tidak ada), proteksi role (`card_admin` ditolak 403 di `GET /activity-log`, cuma `super_admin` boleh), tanpa token ditolak 401. Response `GET /activity-log` include info `actor` (username, role) lewat relasi, memudahkan audit tanpa join manual di UI nanti.
- Tidak ada task UI eksplisit di checklist T015 (beda dari task lain yang selalu punya bagian "Admin UI" kalau memang butuh halaman) — jadi T015 selesai di backend saja, sesuai spec apa adanya. Halaman audit log di admin dashboard bisa jadi task terpisah nanti kalau dibutuhkan.

**Ref:** [[Projek/AbsenSI/11-Decisions|ADR-020]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]]

---

### T016 — End-of-Day Job: Deteksi Tidak Tap Pulang ✅
- [x] Setup BullMQ scheduled job: jalan setiap hari pukul **18:00** (atau jam setelah jam pulang sekolah — sesuaikan dengan jadwal)
- [x] Job logic:
  - Ambil semua `attendance_records` hari ini dengan `waktu_pulang = null` AND `student_id IS NOT NULL`
  - Tandai record-record ini sebagai "perlu klarifikasi" — bisa dengan flag kolom `needs_clarification: boolean` atau cukup query dinamis di dashboard piket (tidak perlu kolom baru kalau rekap dari query saja sudah cukup)
- [x] Data ini akan dikonsumsi Dashboard Piket di T025 (antrian klarifikasi)

**Catatan implementasi (2026-07-14):**
- `apps/api/src/attendance/end-of-day.service.ts` (`EndOfDayService.detectMissingCheckouts()`) — **tidak menulis flag apa pun ke DB**, sesuai opsi yang dipilih spec ("cukup query dinamis, tidak perlu kolom baru"). Job ini query + log sebagai bukti kerja terjadwal & titik ekstensi mudah kalau nanti dibutuhkan notifikasi/ringkasan otomatis (WA ke piket, dst — fase 3). Dashboard Piket (T025) nanti query ulang secara live saat dibuka, tidak bergantung sama sekali ke hasil job ini — job dan dashboard sama-sama baca dari `attendance_records`, sumber kebenaran tunggal.
- `apps/api/src/attendance/end-of-day.scheduler.ts` — pola BullMQ **repeatable job** pakai `queue.upsertJobScheduler()` (API BullMQ v5.16+, menggantikan `add(..., {repeat})` yang deprecated) dengan cron pattern `0 18 * * *` (setiap hari jam 18:00 WIB, sesuai timezone server — lihat `10-Environment.md`). Worker terpisah listen ke queue yang sama, konsisten dengan pola raw BullMQ (bukan `@nestjs/bullmq` decorator) yang sudah dipakai `AttendanceRecordedConsumer` di T012.
- Filter query: `tanggal = hari ini` (pakai `startOfDay()` dengan `Date.UTC()` eksplisit, pola yang sama dengan fix bug timezone di T012 — supaya konsisten dan tidak mengulang bug yang sama), `waktuPulang: null`, `studentId: { not: null }` (guru sengaja dikecualikan — spec cuma minta siswa, guru tidak masuk alur "klarifikasi ke piket").
- Diuji manual dengan trigger job langsung ke queue (`queue.add()` sekali-jalan, bukan menunggu jam 18:00 sungguhan) — skenario: (1) siswa tap masuk tanpa tap pulang → terdeteksi "1 siswa belum tap pulang"; (2) guru tap masuk tanpa tap pulang di hari yang sama → **tidak** ikut terhitung (filter `studentId` bekerja); (3) siswa yang sama lalu tap pulang → re-trigger job → hasil berubah jadi "0 siswa belum tap pulang", membuktikan query benar-benar dinamis dan real-time terhadap state `attendance_records`, bukan snapshot yang basi.

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]]

---

## 📡 Blok 5 — Realtime & Dashboard
> Dependency: T012 (tap API harus dispatch event)

### T017 — Socket.IO Setup
- [x] Install `@nestjs/websockets`, `socket.io` di `apps/api`
- [x] `AttendanceGateway`:
  - Channel `attendance:today` — kirim payload agregat (jumlah hadir/terlambat/belum hadir) setiap `attendance.recorded` event dari BullMQ
  - Channel `attendance:kampus:{id}` — kirim update per-siswa (untuk Dashboard Piket T023)
- [x] `apps/web` + `apps/kiosk`: setup Socket.IO client (`socket.io-client`)
- [x] Auth WebSocket: kirim JWT di handshake header (bukan query param — hindari JWT di URL log)

**Ref:** [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack]], [[Projek/AbsenSI/11-Decisions|ADR-006]]

**Catatan implementasi (2026-07-14):**
- `apps/api/src/realtime/attendance.gateway.ts` (`AttendanceGateway`) + `realtime.module.ts` (baru). Auth dilakukan manual di `handleConnection` (bukan lewat guard Passport HTTP-bound) — verifikasi JWT dari `socket.handshake.auth.token` (bukan query param URL, sesuai spec), plus cek blacklist Redis via `AuthService.isAccessTokenBlacklisted()` yang sudah ada dari modul Auth. Socket yang gagal auth langsung `disconnect(true)` setelah emit event `error`.
- Kiosk device (ADR-004, token statis `KIOSK_DEVICE_TOKEN`) juga difasilitasi di `handleConnection` lewat `handshake.auth.deviceToken` sebagai jalur auth terpisah dari JWT — tapi **client Socket.IO di `apps/kiosk` sengaja TIDAK dibuat di task ini**. Alasan: kiosk device token selama ini konsisten tidak pernah diekspos ke browser JS kiosk (selalu lewat Route Handler proxy server-side seperti `/api/tap`); mengirim token itu ke browser untuk keperluan WS handshake akan melanggar pola tersebut. Kiosk juga sudah dapat feedback tap synchronous dari response REST, jadi tidak ada kebutuhan mendesak untuk data realtime tambahan di layar kiosk. Jalur auth kiosk di gateway dibiarkan siap dipakai kalau nanti memang dibutuhkan (mis. status lock siswa realtime di layar kiosk).
- Channel `attendance:kampus:{id}` **hanya untuk siswa**, bukan guru — skema `Teacher` tidak punya `kampusId` langsung (guru mengajar lintas kampus lewat `schedules`), jadi payload event `kampusId` di-resolve dari `card.student.kelas.kampusId` (null untuk tap guru). `AttendanceService.tap()` diubah untuk include `kelas: { select: { kampusId: true } }` di query card, dan payload BullMQ `attendance.recorded` ditambah field `kampusId` dan `status`.
- `AttendanceRecordedConsumer` (BullMQ worker attendance.recorded) sekarang inject `AttendanceGateway` dan panggil `broadcastAttendanceRecorded()` di setiap job — broadcast ke room `attendance:kampus:{id}` (kalau `kampusId` tidak null) dan broadcast global `attendance:today` (re-query agregat langsung dari DB, bukan counter in-memory — cukup murah di skala ~2500 siswa Fase 1, dan menghindari counter drift).
- Endpoint baru `GET /attendance/today` (guard `JwtAuthGuard` saja, semua role login boleh akses) — dipakai untuk hidrasi awal sebelum data pertama datang lewat socket. Delegasi ke `AttendanceGateway.computeTodayAggregate()` (method yang sama dipakai gateway) supaya tidak ada duplikasi logic hitung agregat.
- `apps/web`: `socket.io-client` diinstall, helper `lib/use-attendance-socket.ts` (`useAttendanceSocket()` hook) dibuat. Karena access token httpOnly tidak bisa dibaca client JS (pola yang sudah dipakai sejak T004), dibuat Route Handler baru `GET /api/ws-token` yang baca cookie server-side (refresh dulu kalau access token sudah expired, reuse logic yang sama dengan `apiFetch`) dan kirim token mentah sekali ke browser untuk dipakai di socket handshake — level trust yang sama dengan data lain yang sudah lewat proxy ke browser. Hook juga hydrate state awal lewat `GET /attendance/today` (via `apiClientFetch`/proxy) supaya angka tidak kosong sebelum event pertama masuk.
- Widget preview kecil (bukan desain final — itu tugas T018/T019) ditambahkan sementara ke `(admin)/page.tsx` untuk membuktikan wiring end-to-end: badge status koneksi + 3 angka (Hadir/Terlambat/Belum Hadir).
- **Verifikasi end-to-end via Playwright** (browser asli, bukan curl): login admin sungguhan di browser → dashboard hydrate angka awal dari `GET /attendance/today` lewat proxy → fire tap sungguhan lewat `POST /attendance/tap` (kiosk token) di request terpisah saat browser masih terbuka → angka di dashboard berubah live (0 Hadir → 1 Terlambat → 4 Belum Hadir) TANPA reload, murni dari event Socket.IO yang di-push. Screenshot dashboard sebelum & sesudah tap dicocokkan dengan style design system (beige bg, card putih rounded, sidebar oranye) — konsisten.
- **Bug ditemukan+diperbaiki selama testing:** (1) `useAttendanceSocket` awalnya cuma listen event push, tidak hydrate dari REST saat mount — jadi counter tampil "-" sampai event pertama datang. Diperbaiki dengan menambahkan `apiClientFetch("/attendance/today")` di awal effect. (2) Setelah menjalankan `pnpm --filter @absensi/web build` (production build) sementara dev server `next dev` masih jalan di proses lama, cache `.next` jadi campur dan route handler proxy gagal resolve module (`MODULE_NOT_FOUND` di `app/api/proxy/[...path]/route.js`) — root cause bukan bug kode, tapi kolusi build artifact dev vs prod di direktori `.next` yang sama. Fix: hapus `.next`, restart dev server bersih. Pelajaran: jangan jalankan `build` dan `dev` bersamaan di app yang sama tanpa membersihkan `.next` di antaranya.
- Ditemukan juga (bukan bug, tapi catatan operasional): proses `nest start --watch` dari sesi-sesi sebelumnya menumpuk tanpa pernah di-kill (11+ proses zombie sejak kemarin), menyebabkan `EADDRINUSE` palsu yang awalnya terlihat seperti error startup baru. Selalu `ps aux | grep nest` sebelum menyalakan dev server baru di sesi ini.

---

### T018 — Dashboard TV (`/tv`)
- [x] Route `/tv` di `apps/web` dengan layout tersendiri (fullscreen, no navbar)
- [x] Initial load: `GET /attendance/today` → tampilkan agregat
- [x] Socket.IO subscribe ke `attendance:today` → update angka realtime tanpa reload
- [x] Tampilan: angka besar (Hadir / Terlambat / Belum Hadir), jam real-time, tanggal
- [x] Auth: akun `kepsek`, refresh token 30 hari sliding renewal
- [x] Proteksi route: redirect ke login kalau token expired

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-tv|dashboard-tv.md]]

**Catatan implementasi (2026-07-14):**
- `apps/web/src/app/tv/layout.tsx` (baru) — layout terpisah dari `(admin)/layout.tsx`, cuma `bg-page` full height, tanpa `Sidebar`/`TopBarWithTitle`. Proteksi auth pakai pola yang sama persis dengan `(admin)/layout.tsx`: `getCurrentUser()` (baca cookie httpOnly server-side) → `redirect("/login")` kalau tidak ada sesi. Middleware global (`middleware.ts`) sudah otomatis mencakup `/tv` juga (matcher default menangkap semua path selain `_next` & `/api/auth`), jadi request tanpa cookie sama sekali langsung ke-redirect sebelum layout sempat render.
- `apps/web/src/app/tv/page.tsx` (baru) — reuse `useAttendanceSocket()` hook dari T017 apa adanya (hydrasi awal + subscribe `attendance:today`), tidak ada channel `attendance:kampus:{id}` di sini (itu untuk Dashboard Piket T023). Jam & tanggal live lewat `setInterval` 1 detik client-side (`toLocaleTimeString`/`toLocaleDateString` locale `id-ID`), memakai `useState(null)` awal + set di `useEffect` supaya tidak ada mismatch hydration SSR/CSR (server tidak tahu jam client).
- Angka ditampilkan pakai ukuran arbitrary Tailwind (`text-[130px]` untuk digit, `text-[88px]` untuk jam) karena token `fontSize` di `packages/config/tailwind.config.ts` cuma sampai `display` (32px) — jauh terlalu kecil untuk dilihat dari jarak jauh di TV. Warna tetap pakai token semantik yang sudah ada (`text-success-text`, `text-primary`, `text-ink-secondary`) supaya konsisten dengan design system, bukan warna hardcoded baru.
- **Sliding renewal 30 hari untuk `kepsek`:** `AuthService.issueTokenPair()` sudah menerbitkan refresh token 30 hari (bukan 7 hari) khusus role `kepsek` sejak T004 (`REFRESH_TOKEN_TTL_SLIDING_SECONDS`), dan `AuthService.refresh()` sudah rotasi (refresh token lama dicabut, terbit baru dengan TTL penuh lagi) — infrastrukturnya sudah ada, task ini tinggal memastikan sesuatu di halaman TV benar-benar memicu siklus refresh itu secara berkala. Ditambahkan `setInterval` 5 menit di `/tv` yang memanggil ulang `GET /attendance/today` lewat proxy (`apiClientFetch`) — proxy Route Handler (`lib/api.ts` → `apiFetch`) sudah auto-refresh kalau access token 401, jadi pemanggilan berkala ini otomatis memperbarui access + refresh token cookie di background tanpa TV pernah perlu re-login manual selama dia menyala terus.
- Ditambahkan akun seed baru **`kepsek` / `password123`** (role `kepsek`) di `apps/api/prisma/seed.ts` untuk keperluan testing T018 — DB dev yang sudah pernah di-seed sebelumnya di-insert manual (bukan re-run seed penuh, supaya tidak bentrok unique constraint kampus/jurusan yang sudah ada).
- Widget preview kecil yang sempat ditambahkan ke dashboard admin (`(admin)/page.tsx`) saat T017 untuk pembuktian wiring dihapus lagi di task ini — sudah digantikan oleh implementasi `/tv` yang sesungguhnya, dashboard admin balik ke placeholder text (menunggu T019).
- **Verifikasi end-to-end via Playwright** (browser asli): (1) navigasi ke `/tv` tanpa sesi → dikonfirmasi redirect ke `/login?next=%2Ftv`; (2) login sebagai `kepsek` → dikonfirmasi kembali otomatis ke `/tv` (bukan ke `/` seperti akun admin biasa) berkat query param `next`; (3) dikonfirmasi tidak ada elemen sidebar (`Kampus`, dst) di halaman — fullscreen murni; (4) hydrasi awal dari `GET /attendance/today` tampil benar sebelum ada tap; (5) fire tap sungguhan (`POST /attendance/tap`) dari dalam skrip yang sama saat browser masih terbuka → angka Terlambat & Belum Hadir berubah live dalam ~4 detik tanpa reload, sinkron dengan hasil response tap. Screenshot sebelum & sesudah tap menunjukkan angka besar kontras tinggi (hijau/oranye/abu-abu) di atas background beige, konsisten dengan design system.
- Tidak ada bug baru ditemukan selama implementasi task ini (infrastruktur T017 — gateway, hook, proxy — terpakai ulang tanpa modifikasi berarti). Satu kesalahan proses (bukan bug kode) saat testing awal: skrip Playwright pertama menembak tap dari proses `curl` terpisah yang timing-nya tidak sinkron dengan window tunggu skrip, sehingga event terlewat sebelum screenshot diambil — diperbaiki dengan memanggil `curl` dari dalam skrip Playwright yang sama (`execSync`) supaya urutan event terjamin.

---

### T019 — Admin Dashboard: Rekap Kehadiran Siswa
- [x] API `GET /attendance/report` dengan query params: `from`, `to`, `kelas_id?`, `jurusan_id?`
  - Query logic: untuk setiap siswa dalam filter → hitung count hadir, terlambat, izin, sakit, alfa
  - **Alfa** = hari wajib (Senin–Jumat, bukan libur di `school_holidays`, dalam `academic_years` aktif) tanpa record di `attendance_records` DAN tanpa `permits`
  - Index yang dipakai: `(tanggal, student_id)` di `attendance_records`, `(tanggal, student_id)` di `permits`
- [x] Admin UI:
  - Halaman Rekap: filter bar (date range picker, dropdown kelas, dropdown jurusan) + tombol Tampilkan
  - Tabel hasil: Nama | Kelas | Jurusan | Hadir | Terlambat | Izin | Sakit | Alfa
  - Loading state saat query berjalan

**Ref:** [[Projek/AbsenSI/06-Features/rekap-kehadiran|rekap-kehadiran.md]], [[Projek/AbsenSI/06-Features/kalender-pendidikan|kalender-pendidikan.md]]

**Catatan implementasi (2026-07-14):**
- `apps/api/src/attendance/attendance-report.service.ts` (`AttendanceReportService`, baru) + `dto/report-query.dto.ts` (`from`, `to` wajib `IsDateString`, `kelasId`/`jurusanId` opsional). Endpoint `GET /attendance/report` di `AttendanceController`, digerbang `JwtAuthGuard + RolesGuard` + `@Roles(super_admin)` — sesuai spec Fase 1 (rekap-kehadiran.md: hanya Admin Pusat, `kepsek` baru Fase 2). Diverifikasi 403 untuk akun `kepsek`.
- **Perhitungan "hari wajib"** (dasar kolom Total Hari Aktif & Alfa) di-generate di kode aplikasi (loop tanggal harian di TypeScript), BUKAN recursive CTE SQL — untuk setiap `academic_years` aktif yang overlap `[from,to]`, iterasi tanggal, exclude Sabtu(6)/Minggu(0) `getUTCDay()`, exclude tanggal yang masuk range manapun di `school_holidays`. Dipilih pendekatan ini karena rentang rekap khas (per bulan) cuma puluhan hari — jauh di bawah kebutuhan optimasi SQL murni yang disebut di spec (spec menyebut volume 50rb-75rb baris/bulan untuk `attendance_records`, bukan untuk perhitungan kalender itu sendiri).
- **Alfa per siswa** dihitung sebagai `hari_wajib − (hari_ada_di_attendance_records ∪ hari_ada_di_permits)` — union tanggal (bukan penjumlahan count) supaya siswa yang kebetulan taps DAN punya permit di hari sama (data kotor/edge case) tidak dihitung 2x atau menghasilkan alfa negatif.
- **Hadir vs Terlambat**: keduanya dihitung terpisah dari `attendance_records.status` (spec eksplisit: kedua kolom terpisah untuk lihat kedisiplinan, meski "masuk" formal = hadir+terlambat). Kolom `hadir` di response HANYA menghitung status `hadir` murni (bukan hadir+terlambat digabung) — sesuai definisi kolom terpisah di tabel spec.
- **Izin/Sakit**: dihitung dari `permits.alasan_kategori`, tidak peduli `jenis` (`tidak_masuk` vs `keluar`) — kedua jenis izin dianggap "ada keterangan" untuk hari itu, konsisten dengan definisi alfa (tidak ada attendance DAN tidak ada permit). Modul Permits (API CRUD-nya) belum dibangun (baru T022), tapi tabel `permits` sudah ada di schema sejak migrasi awal — service ini baca langsung lewat Prisma (bukan bypass modul, karena belum ada modul Permits untuk di-bypass).
- **UI**: `apps/web/src/app/(admin)/rekap/` (`page.tsx` server component fetch `kelas`+`jurusan` via `apiFetch` untuk dropdown, `rekap-view.tsx` client component). Filter pakai komponen `DatePicker` dari `packages/ui` (belum pernah dipakai di app manapun sebelumnya — dipasang pertama kali di sini), `Select` dengan pola sentinel `__all__` yang sudah dipakai di halaman Siswa. Nav item baru "Rekap Kehadiran" ditambahkan ke `nav-items.ts` (ikon `BarChart3`). `date-fns` ditambahkan sebagai dependency eksplisit `apps/web` (sebelumnya cuma ada di `packages/ui`, dipakai transitif lewat hoisting pnpm — sekarang eksplisit karena `rekap-view.tsx` import `format` langsung).
- **Verifikasi API menyeluruh via curl** dengan data fixture nyata (insert langsung ke `attendance_records`/`permits`, bukan lewat tap API karena `client_timestamp` dibatasi maksimal 1 hari ke belakang — tidak bisa dipakai backfill data historis untuk testing rentang minggu): skenario 1 siswa dengan hadir+terlambat+izin+sakit+alfa (masing-masing 1 hari dalam seminggu, total 5 hari wajib) menghasilkan angka yang tepat sesuai desain; siswa tanpa data sama sekali menunjukkan alfa = total hari wajib; filter kelas & jurusan mempersempit hasil dengan benar; rentang yang overlap libur semester (16 Des) mengecualikan hari libur dari hari wajib dengan tepat; rentang Sabtu-Minggu murni menghasilkan `totalHariAktif: 0`; validasi `from > to` menghasilkan 400; akses non-`super_admin` menghasilkan 403.
- **Verifikasi UI end-to-end via Playwright** (browser asli): login admin → buka `/rekap` → set tanggal via `DatePicker` popup kalender (klik tanggal 13 & 19 Juli) → filter kelas via `Select` → klik "Tampilkan" → loading state (skeleton rows, tombol "Memuat...") sempat tertangkap di screenshot → hasil tabel cocok persis dengan hasil curl. Filter kelas "XI TKJ 1" dikonfirmasi mempersempit tabel ke 2 siswa yang benar. Tidak ada bug ditemukan di lapisan UI.
- Tidak ada bug logic baru ditemukan. Satu gangguan operasional (bukan bug kode) saat testing awal API: proses `dist/main` sisa dari sesi `pnpm build` sebelumnya (dari jam 16:47) masih menempati port 3001 dan menyajikan kode lama tanpa route `/attendance/report`, menyebabkan 404 palsu meski log startup proses baru menunjukkan route sudah ter-mapping dengan benar (proses baru langsung crash `EADDRINUSE` setelah listen gagal). Root cause ketahuan dari membaca log lengkap (bukan cuma cek "server up"), bukan dari curl semata — pelajaran yang sama persis dengan insiden T015, dicatat lagi di sini karena berulang: **selalu cek `ps aux | grep -E "nest|next"` untuk proses ganda SEBELUM curiga ke kode, terutama kalau baru saja menjalankan `build` dan `dev` bergantian di direktori yang sama.**

---

## 👤 Blok 6 — Akun Guru
> Dependency: T003 (auth), T005 (teachers harus ada)

### T020 — Users Module: CRUD Akun
- [x] API endpoints (akses: `super_admin`):
  - `GET /users` — daftar semua akun
  - `POST /users` — buat akun baru (role, username, password, link ke teacher_id atau kampus_id)
  - `PATCH /users/:id` — edit akun (nonaktifkan, ganti role)
  - `POST /users/:id/reset-password` — generate password baru, return ke admin untuk disampaikan manual
- [x] Admin UI: halaman Manajemen Akun — tabel + form create/edit + tombol reset password

**Ref:** [[Projek/AbsenSI/03-User-Roles|03-User-Roles]], [[Projek/AbsenSI/11-Decisions|ADR-008]]

**Catatan implementasi (2026-07-14):**
- `apps/api/src/users/` (`UsersModule` baru): `users.service.ts`, `users.controller.ts`, `dto/create-user.dto.ts`, `dto/update-user.dto.ts`. Semua endpoint digerbang `JwtAuthGuard + RolesGuard` + `@Roles(super_admin)` di level class controller — konsisten ADR-008 (permission ditegakkan di backend, bukan cuma UI).
- **Bug keamanan ditemukan+diperbaiki SEBELUM sempat jadi masalah nyata** (bukan dari testing, tapi dari review desain sebelum coding): `ActivityLogInterceptor` (dibangun T015) melakukan `findUnique` mentah tanpa `select` untuk `snapshot_before`, dan menyimpan seluruh response body sebagai `snapshot_after` — kalau dipakai apa adanya di modul Users, ini akan menulis `password_hash` (dan password baru hasil reset) ke tabel `activity_log` yang insert-only (ADR-020, tidak bisa dihapus/dikoreksi kalau kebobolan). Diperbaiki secara general (bukan workaround khusus modul Users): `LogActivityOptions` dapat field baru `sensitiveFields?: string[]`, `ActivityLogInterceptor` redact field tersebut dari `snapshot_before` maupun `snapshot_after` sebelum insert. Modul lain (`cards`, `school-holidays`, dst.) tidak terpengaruh karena field ini opsional. Endpoint Users pakai `sensitiveFields: ["passwordHash"]` (create/update) dan `["passwordHash", "newPassword"]` (reset-password, karena password baru dalam bentuk plaintext ada sebentar di response body).
- **`UsersService`** query `select` eksplisit (`USER_SELECT` const) di semua response — `passwordHash` tidak pernah keluar dari service, terlepas dari redaction di interceptor (defense in depth).
- **Validasi keterkaitan role**: role `guru` **wajib** `teacherId` (dicek `ensureLinksValid`, 400 kalau tidak ada) — mencegah akun guru "mengambang" tanpa data Teacher terkait, yang akan bikin fitur T021 (riwayat kehadiran guru, baca `attendance_records` by `teacherId` dari JWT) rusak diam-diam. `teacherId`/`kampusId` (kalau diisi) divalidasi benar-benar ada di tabel terkait sebelum create/update. Role lain (`super_admin`, `card_admin`, `kepsek`) tidak mewajibkan keterkaitan apa pun (`teacherId`/`kampusId` tetap opsional untuk role-role ini) — beda dari pola "exactly one" dual-FK di `Card` (ADR-010), karena di sini banyak kombinasi role valid tanpa keterkaitan sama sekali.
- **Generate password reset**: `nanoid`'s `customAlphabet`, 10 karakter, alfabet tanpa karakter yang mudah tertukar (`0/O`, `1/l/I` dihilangkan) — karena spec eksplisit "return ke admin untuk disampaikan manual" (dibacakan lisan/diketik ulang oleh admin ke pemilik akun, bukan dikirim otomatis).
- **UI**: `apps/web/src/app/(admin)/akun/` (`page.tsx` fetch `users`+`teachers`+`kampus` server-side, `akun-view.tsx` client). Nav item "Manajemen Akun" masuk ke `secondaryNav` (sebelumnya kosong, baru dipakai pertama kali di sini — dipisah visual dari nav Core data via divider yang sudah ada di `Sidebar`). Form create/edit sama (`UserForm` dengan `mode` prop) mengikuti pola dialog di halaman Kartu — field `teacherId`/`kampusId` muncul kondisional cuma saat role `guru`/`guru_piket` dipilih. Dialog terpisah untuk menampilkan hasil reset password (sekali tampil, form-nya sendiri bukan alert browser supaya bisa di-styling & di-screenshot admin kalau perlu).
- **Verifikasi API menyeluruh via curl**: list, create (`card_admin` tanpa keterkaitan, `guru` dengan `teacherId` valid), create gagal (guru tanpa teacherId → 400, teacherId tidak ada → 400, username duplikat → 409, password < 8 karakter → 400), patch nonaktifkan (dikonfirmasi akun nonaktif gagal login lewat `AuthService.login` yang sudah ada), reset-password (dikonfirmasi password lama gagal login, password baru hasil reset berhasil login), role guard (`kepsek` akses `GET /users` → 403), dan yang paling penting: query langsung ke `activity_log` mengonfirmasi **tidak ada `password_hash` atau `newPassword` yang tersimpan** di `snapshot_before`/`snapshot_after` manapun.
- **Verifikasi UI end-to-end via Playwright** (browser asli): buat akun baru role guru lewat form (isi username/password, pilih role, pilih guru terkait dari dropdown) → submit → baris baru muncul di tabel tanpa reload; edit akun tersebut → ubah status ke Nonaktif → tabel update; klik Reset pada akun yang sama → dialog konfirmasi browser (`confirm()`) → dialog hasil menampilkan password baru dengan instruksi yang jelas. Semua langkah dikonfirmasi lewat screenshot, konsisten dengan design system (beige bg, card putih, badge status hijau/abu-abu, tombol pill oranye).
- Tidak ada bug fungsional baru ditemukan selain isu keamanan interceptor di atas (yang dicegah sebelum sempat termanifestasi sebagai data bocor, bukan hasil temuan reaktif dari incident).

---

### T021 — Guru Portal: Riwayat Kehadiran Diri Sendiri
- [x] API `GET /attendance/my-history` (akses: role `guru`) — ambil `attendance_records` berdasarkan `teacher_id` dari JWT payload, dengan filter tanggal opsional
- [x] Guru UI (route `/riwayat` di `apps/web`):
  - Setelah login sebagai guru: tampilkan tabel riwayat kehadiran (tanggal, waktu masuk, waktu pulang, status)
  - Filter tanggal (date range picker)
  - Read-only — tidak ada tombol edit apa pun

**Ref:** [[Projek/AbsenSI/06-Features/akun-guru|akun-guru.md]]

**Catatan implementasi (2026-07-14):**
- `AttendanceService.myHistory(teacherId, query)` (baru, di `attendance.service.ts` yang sudah ada) + `dto/my-history-query.dto.ts` (`from`/`to` opsional, `IsDateString`). Endpoint `GET /attendance/my-history` di `AttendanceController`, digerbang `JwtAuthGuard + RolesGuard` + `@Roles(guru)`.
- **Keputusan keamanan inti**: `teacherId` untuk query **hanya** diambil dari `@CurrentUser()` (payload JWT, `user.teacherId`) via decorator yang sudah ada dari modul Auth — **tidak pernah** dari query param atau body. `MyHistoryQueryDto` sengaja tidak punya field `teacherId` sama sekali, jadi kalau seseorang coba inject `?teacherId=X` di URL, `ValidationPipe` global (`forbidNonWhitelisted: true`) langsung menolak dengan 400 sebelum request sampai ke service — dicoba manual saat testing dan terkonfirmasi. Ini mencegah satu guru melihat riwayat guru lain hanya dengan mengubah parameter URL, sesuai spec "tidak ada akses ke data guru lain".
- Kalau akun ber-role `guru` ternyata tidak tertaut ke `teacherId` manapun (harusnya tidak mungkin terjadi karena `UsersService.create()` di T020 sudah mewajibkan `teacherId` untuk role `guru` — tapi tetap dijaga di endpoint ini juga, defense in depth), endpoint melempar 400 alih-alih query dengan `teacherId: null` yang bisa berperilaku tidak terduga di Prisma.
- **UI — keputusan arsitektur**: dibuatkan route group terpisah `apps/web/src/app/(guru)/` dengan layout sendiri (`(guru)/layout.tsx`), BUKAN menambahkan halaman `/riwayat` ke shell admin `(admin)/layout.tsx` yang sudah ada. Alasan: shell admin sebelumnya menampilkan sidebar penuh (Kampus, Siswa, Kartu, dst.) untuk siapa pun yang login tanpa filter role — backend sudah menolak akses guru ke endpoint-endpoint itu (403), tapi UI belum menyesuaikan sebelum task ini. Route group terpisah meniru pola yang sudah dipakai `/tv` (T018): layout sendiri, tanpa `Sidebar` sama sekali (bukan cuma nav item disembunyikan) — supaya guru tidak dapat *satu pun* jalur navigasi visual ke data siswa/guru lain, konsisten dengan spec akun-guru.md ("tidak ada akses ke data siswa, tidak ada akses ke data guru lain").
- **Redirect dua arah** ditambahkan sebagai penjaga silang: `(admin)/layout.tsx` sekarang redirect ke `/riwayat` kalau `user.role === "guru"` (mencegah guru "nyasar" ke shell admin lewat URL manapun di grup itu, termasuk `/` root); `(guru)/layout.tsx` redirect ke `/login` kalau tidak ada sesi, dan ke `/` kalau role BUKAN `guru` (mencegah role lain mengakses `/riwayat` meski tahu URL-nya — walau secara data tidak berbahaya karena API tetap 403 untuk non-guru, tapi UX-nya jadi konsisten "role hanya melihat shell yang relevan untuknya").
- `TopBar` (komponen yang sudah ada dari T004, generik: title/userName/roleLabel) dipakai ulang apa adanya di shell guru — tidak perlu komponen baru. `Sidebar` sengaja TIDAK dipakai ulang (dan tidak di-generalize untuk terima props nav items berbeda) karena kompleksitas conditional-nav tidak sepadan untuk 1 kasus penggunaan; shell guru cukup header statis tanpa navigasi sama sekali (cuma 1 halaman yang bisa diakses).
- **Verifikasi API menyeluruh via curl**: guru melihat riwayat sendiri (3 record, urut terbaru dulu) dengan data cocok; filter `from` mempersempit hasil dengan benar; admin (`super_admin`) akses endpoint ini → 403 (endpoint khusus role `guru`); guru akses `/users` (admin-only) → 403; **dua akun guru berbeda dikonfirmasi TIDAK saling melihat data** (`guru_suherman` dengan `teacherId=2` mendapat array kosong, bukan data milik `guru_ratna`); percobaan injeksi `?teacherId=1` via query param ditolak 400 oleh whitelist validation.
- **Verifikasi UI end-to-end via Playwright** (browser asli): login sebagai `guru_ratna` → landing di `/` otomatis ter-redirect ke `/riwayat` tanpa guru sempat melihat shell admin sama sekali → dikonfirmasi tidak ada elemen sidebar (`Kampus`, dst.) di halaman → tabel riwayat tampil benar (3 baris, badge Hadir/Terlambat, format tanggal Indonesia lengkap) → percobaan navigasi manual ke `/siswa` dan `/akun` keduanya langsung ter-redirect balik ke `/riwayat`. Sanity check terpisah: login `admin` biasa dikonfirmasi TIDAK terpengaruh (tetap landing di `/` dengan sidebar penuh) — tidak ada regresi di alur admin.
- Tidak ada bug ditemukan. Blok 6 (Akun Guru) sekarang selesai penuh (2/2) — ini juga menandai seluruh **Blok 0–6 (fondasi inti Fase 1) selesai**; sisa scope Fase 1 murni Blok 7 (Dashboard Piket, Fase 1b).

---

## 🏫 Blok 7 — Dashboard Piket (Fase 1b)
> Dependency: semua Blok 0–6 harus stabil. Mulai setelah inti Fase 1 berjalan dan diuji.

### T022 — Permits Module: API
- [x] API endpoints (akses: `guru_piket` — ditegakkan di `RolesGuard`):
  - `POST /permits` — buat permit baru (`tidak_masuk` atau `keluar`)
    - Saat create: otomatis update `attendance_records` hari itu sesuai jenis
    - Generate `kode_verifikasi` unik (nanoid 8 char uppercase) untuk `jenis=keluar`
  - `PATCH /permits/:id/confirm-kembali` — tandai `status_kembali: sudah`
  - `PATCH /permits/:id/set-pulang` — tandai `status_kembali: pulang`, update `attendance_records`
  - `GET /permits` — daftar permits (filter: tanggal, kampus_id dari token piket)
- [x] `POST /attendance/manual-pulang` (akses: `guru_piket`) — input pulang manual tanpa tap (fallback Sub-alur B)
- [x] `POST /attendance/confirm-izin-pulang/:record_id` (akses: `guru_piket`) — ubah `pulang_via` tap → `tap_izin_pulang`
- [x] Semua aksi piket dicatat ke `activity_log`

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], [[Projek/AbsenSI/11-Decisions|ADR-016, ADR-019]]

**Catatan implementasi (2026-07-14):**
- **Task ini API-only** sesuai breakdown task (tidak ada item UI di T022) — UI Dashboard Piket menyusul di T023/T024. Modul baru `apps/api/src/permits/` (`PermitsModule`, `PermitsService`, `PermitsController`) + penambahan 2 method di `AttendanceService`/`AttendanceController` yang sudah ada (`manualPulang`, `confirmIzinPulang`).
- **Perbaikan gap dari T020 (ditemukan sebelum sempat jadi bug produksi)**: `UsersService.ensureLinksValid()` sebelumnya hanya mewajibkan `teacherId` untuk role `guru`, TIDAK mewajibkan `kampusId` untuk `guru_piket` — padahal seluruh logic scoping T022 bergantung mutlak pada `kampusId` ada di JWT payload piket (ADR-015). Kalau tidak diperbaiki, admin bisa membuat akun `guru_piket` tanpa kampus dan endpoint Permits akan error/berperilaku tidak terduga saat akun itu login. Ditambahkan validasi simetris dengan yang sudah ada untuk `guru`/`teacherId`.
- **Keputusan desain (dikonfirmasi ke user sebelum coding)**: `POST /permits` jenis `tidak_masuk` (Izin/Sakit) **menolak dengan 409** kalau `attendance_records` hari itu untuk siswa tersebut sudah ada (siswa sudah tap masuk). Spec dashboard-piket.md tidak eksplisit menyebutkan skenario ini — dipilih "tolak, jangan overwrite diam-diam" konsisten dengan semangat ADR-016 (tap dan permits adalah 2 jalur terpisah yang tidak saling menimpa tanpa jejak jelas). Efek samping yang berguna: aturan yang sama otomatis mencegah 2 permit `tidak_masuk` dibuat untuk siswa+hari yang sama (permit pertama sudah membuat `attendance_records`, jadi percobaan kedua kena 409 juga).
- **Kampus-scoping ditegakkan berlapis di setiap endpoint** (bukan cuma 1 tempat terpusat): `PermitsService` (`ensureStudentInKampus`, `ensurePermitInKampus`) dan `AttendanceService` (`findTodayRecordInKampus`, cek langsung di `confirmIzinPulang`) masing-masing query `student.kelas.kampusId` dan bandingkan dengan `kampusId` dari JWT — kalau siswa/permit/record bukan milik kampus akun yang login, 403 `ForbiddenException`. `kampusId` sendiri diambil murni dari `@CurrentUser()` (JWT), sama sekali tidak pernah dari body/query request — pola yang sama persis dengan `teacherId` di T021.
- **`createTidakMasuk` dan `setPulang` pakai `$transaction`** (permit + attendance_records ditulis atomik) — mencegah state permit "sukses" tapi attendance_records gagal ter-update (atau sebaliknya), yang akan menghasilkan data tidak konsisten antara 2 tabel yang seharusnya selalu sinkron per ADR-016.
- **`createKeluar` sengaja TIDAK pakai transaction** — cuma insert 1 row (`permits`), tidak menyentuh `attendance_records` sama sekali saat dibuat (siswa belum tentu kembali/tidak kembali di titik ini, jadi belum ada yang perlu disinkronkan ke attendance).
- **`confirmKembali`** (Sub-alur A, siswa lapor sudah kembali) dikonfirmasi via testing **tidak membuat/mengubah** `attendance_records` sama sekali — sesuai tabel di spec ("siswa dianggap hadir normal", tidak ada efek ke data kehadiran). Ini beda dari `setPulang` yang memang mengubah `attendance_records`.
- **Bug timezone dihindari proaktif** (bukan ditemukan lewat testing — pelajaran dari insiden T012/T016 yang sudah terjadi 2x sebelumnya di task lain): draft awal `PermitsService` sempat pakai `new Date().toISOString().slice(0,10)` untuk tanggal default permit (kalau `tanggal` tidak diisi di request) — pola ini salah untuk server WIB (UTC+7) karena `toISOString()` mengambil tanggal UTC, bukan tanggal lokal, dan bisa mundur 1 hari di jam-jam awal pagi WIB. Diperbaiki sebelum sempat di-commit, memakai ulang helper `startOfDay()` yang sama persis (local `getFullYear/getMonth/getDate` → `Date.UTC()`) yang sudah dipakai di `AttendanceService` dan `EndOfDayService`.
- **Verifikasi API menyeluruh via curl** dengan 2 akun `guru_piket` di 2 kampus berbeda (dibuat lewat `POST /users` dari T020) dan data siswa lintas kampus asli: (1) create permit tanpa `kampusId` di akun piket → 400 saat pembuatan akun; (2) `tidak_masuk` (sakit) untuk siswa sendiri → sukses, `attendance_records` terisi status `sakit`; (3) percobaan permit kedua untuk siswa+hari yang sama → 409; (4) percobaan permit untuk siswa **kampus lain** → 403; (5) `super_admin` akses `/permits` sama sekali → 403 (ADR-019); (6) `GET /permits` dikonfirmasi ter-scope sempurna — piket kampus lain melihat array kosong; (7) alur `keluar` lengkap: create (kode_verifikasi 8 karakter ter-generate) → `confirm-kembali` (tidak ubah attendance_records, dikonfirmasi via query DB langsung) → percobaan confirm-kembali kedua kali → 409; (8) alur `keluar` kedua: create → `set-pulang` (Dianggap Pulang) → `attendance_records` terisi `waktu_pulang=jam_keluar`, `pulang_via=piket_izin` sesuai tabel spec; (9) tap masuk+pulang normal via kiosk API sungguhan → `confirm-izin-pulang` → `pulang_via` berubah `tap`→`tap_izin_pulang` tanpa mengubah `waktu_pulang`; (10) `manual-pulang` ditolak 403 untuk siswa kampus lain, sukses untuk siswa kampus sendiri dengan `pulang_via: piket_izin`; (11) `keluar` tanpa `jamKeluar` → 400; (12) query `activity_log` langsung mengonfirmasi ke-7 jenis aksi piket (`permit.create` x3, `permit.confirm_kembali`, `permit.set_pulang`, `attendance.manual_pulang`, `attendance.confirm_izin_pulang`) semuanya tercatat dengan `actor_id` yang benar sesuai akun piket yang melakukan aksi.
- Data uji (akun piket, kartu, permit, attendance_records, activity_log terkait) sudah dibersihkan seluruhnya setelah verifikasi — database kembali ke state bersih.

---

### T023 — Dashboard Piket: Halaman Utama Realtime
- [x] Route `/piket` di `apps/web` (scope per `kampus_id` dari JWT piket)
- [x] Tabel semua siswa kampus ini, dengan kolom: nama | kelas | status tap hari ini | jam masuk | jam pulang | action
- [x] Subscribe Socket.IO `attendance:kampus:{id}` → update baris realtime saat ada tap baru
- [x] Tombol **[Izin]** dan **[Sakit]** di setiap baris → modal kecil (isi keterangan → submit → buat `permits(tidak_masuk)`)
- [x] Badge visual untuk siswa yang: sudah hadir / belum hadir / sudah izin/sakit / terkunci

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]]

**Catatan implementasi (2026-07-14):**
- **Backend baru**: `GET /attendance/piket-board` di `AttendanceController`/`AttendanceService` (bukan modul baru — ditaruh di Attendance karena ini fundamentalnya read-model status kehadiran, sejalan dengan `GET /attendance/today` dari T017/T018). Query semua `students` aktif di `kampusId` dari JWT piket, LEFT JOIN `attendanceRecords` hari ini (Prisma `include` dengan `where: { tanggal: today }, take: 1` per siswa, bukan query terpisah per baris — 1 round-trip DB untuk seluruh papan). Response gabung status tap + status lock siswa dalam 1 row per siswa, siap pakai tabel tanpa transformasi tambahan di frontend.
- **UI — route group baru** `apps/web/src/app/(piket)/` mengikuti pola persis `(guru)` dari T021: layout sendiri tanpa `Sidebar`, redirect ke `/login` kalau tidak ada sesi, redirect ke `/` kalau role bukan `guru_piket`. `(admin)/layout.tsx` ditambah 1 baris redirect (`guru_piket` → `/piket`) melengkapi rantai redirect 3-role yang sudah ada (guru → /riwayat, guru_piket → /piket, sisanya tetap di shell admin).
- **`useAttendanceSocket` hook (dari T017) di-generalize**, bukan diduplikasi jadi hook baru: sebelumnya cuma menerima `kampusId` opsional dan selalu subscribe+fetch agregat global `attendance:today`. Ditambah overload — bisa dipanggil dengan object options (`{ kampusId, skipTodayAggregate, onKampusUpdate }`) sambil tetap backward-compatible dengan pemanggilan lama `useAttendanceSocket()` (dashboard TV) dan `useAttendanceSocket(kampusId)` (bentuk number, kalau ada). Halaman piket pakai `skipTodayAggregate: true` (papan piket tidak butuh angka agregat sekolah) + `onKampusUpdate` callback untuk patch baris tabel spesifik dari event `attendance:kampus:update` yang sudah dipancarkan gateway sejak T017 — tidak ada perubahan di `AttendanceGateway` sama sekali, murni pemanfaatan ulang infrastruktur yang sudah ada.
- **Bug ditemukan+diperbaiki lewat Playwright** (bukan dari curl — baru kelihatan setelah interaksi realtime sungguhan di browser): tombol Izin/Sakit awalnya disembunyikan berdasarkan `!row.attendanceRecordId`, tapi patch realtime dari `onKampusUpdate` cuma meng-update field `status`/`waktuMasuk`/`waktuPulang`, TIDAK `attendanceRecordId` (event dari gateway memang tidak membawa field itu). Akibatnya siswa yang baru saja tap tetap menampilkan tombol Izin/Sakit sampai halaman di-refresh manual — piket bisa klik dan kena 409 dari backend (aman secara data karena backend sudah menolak, tapi UX membingungkan). Diperbaiki dengan mengganti kondisi visibility ke `row.status === null` (field yang memang selalu ikut ter-update oleh kedua sumber patch: event realtime dan submit form izin) — lebih tepat juga secara semantik karena persis mencerminkan kondisi 409 yang dicek backend (`PermitsService.createTidakMasuk`).
- **Modal Izin/Sakit** pakai `Input` (bukan `Textarea` — package `@absensi/ui` belum punya komponen itu, dan spec cuma minta "keterangan singkat" jadi single-line cukup) untuk field `alasanDetail` opsional. Submit langsung `POST /permits` dengan `jenis: tidak_masuk` dan `alasanKategori` sudah terisi dari tombol yang diklik (Izin atau Sakit), sesuai spec "kategori sudah terisi dari tombol yang diklik".
- **Verifikasi API via curl** (melengkapi yang sudah dilakukan T022, fokus ke endpoint baru): `piketBoard` menampilkan siswa aktif kampus yang benar dengan status null/hadir/terlambat/izin/sakit/terkunci sesuai data asli; scoping kampus dikonfirmasi ketat (piket kampus lain melihat daftar siswa yang sama sekali berbeda, bukan array kosong — karena memang ada siswa aktif di kampus itu); siswa `nonaktif` dikonfirmasi tidak muncul di papan; `super_admin` akses endpoint → 403.
- **Verifikasi UI end-to-end via Playwright** (browser asli, skenario realtime penuh): login piket → redirect ke `/piket` → tidak ada sidebar admin → tabel awal menampilkan "Belum Hadir" untuk semua siswa → tap kartu sungguhan lewat `POST /attendance/tap` dari proses terpisah (disinkronkan lewat `execSync` di script Playwright yang sama, pelajaran dari T018) saat browser masih terbuka → baris siswa yang tap berubah live ke "Terlambat" dengan jam masuk terisi, TANPA reload, dan tombol Izin/Sakit hilang otomatis dari baris itu (setelah fix bug di atas) → submit form Sakit untuk siswa lain lewat modal sungguhan → baris berubah ke badge "Sakit" live. Sanity check tambahan: dikonfirmasi `/tv` (T018) dan halaman lain yang memakai `useAttendanceSocket` tidak mengalami regresi setelah generalisasi hook.
- **Insiden operasional berulang** (pola identik dengan yang sudah dicatat di T019 dan T020, terjadi lagi di sini): menjalankan `pnpm --filter @absensi/web build` di direktori yang sama saat `next dev` sedang berjalan mengotori `.next` dan menyebabkan 404 pada static chunk (`page.js`, `main-app.js`, dst.), membuat halaman `/tv` sempat gagal total saat sanity check meski kodenya benar. Diperbaiki dengan pola yang sudah baku: hapus `.next`, restart `dev` bersih. Tidak dicatat sebagai bug kode.
- Data uji (akun piket, tap, permit, activity_log terkait) sudah dibersihkan seluruhnya setelah verifikasi.

---

### T024 — Dashboard Piket: Perizinan Keluar + Print
- [x] Menu terpisah `/piket/izin-keluar` di sidebar piket
- [x] Form Sub-alur A (izin keluar sementara):
  - Cari siswa (autocomplete), alasan kategori, keterangan, jam keluar, jam kembali (opsional)
  - Submit → POST `/permits` → generate `kode_verifikasi`
  - Sistem konstruksi URL ke route cetak internal lalu `window.open()` di tab baru
- [x] Form Sub-alur B (konfirmasi izin pulang setelah tap):
  - Cari siswa yang sudah tap pulang hari ini → tombol "Tandai Izin Pulang" → `PATCH /attendance/confirm-izin-pulang/:id`
- [x] ~~Edit `C:\ProjekSMK\print.php` di server 10.10.10.100~~ — **TIDAK LAGI RELEVAN**, lihat revisi arsitektur 2026-07-15 di bawah

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], [[Projek/AbsenSI/11-Decisions|ADR-018 (direvisi 2026-07-15)]]

**Revisi arsitektur (2026-07-15) — print.php eksternal DIHAPUS, diganti Route Handler internal:**
- User (pemilik `print.php` asli) memberikan source code PHP yang sebenarnya dipakai sekolah dan meminta itu disinkronkan ke repo ini — **bukan sekadar edit file di server luar**, tapi mengganti pendekatan ADR-018 sepenuhnya: hentikan ketergantungan pada server PHP terpisah (`10.10.10.100:8800`), port template itu jadi Next.js Route Handler di `apps/web/src/app/print/struk-izin/route.ts`.
- **ADR-018 revisi**: alasan asli ADR-018 (reuse print.php yang sudah terbukti jalan dengan hardware) tetap dihormati — HTML/CSS/struktur struk 58mm untuk printer thermal Blueprint ECO 58D di-port 1:1 dari source PHP asli, bukan didesain ulang. Yang berubah cuma *tempat* logic itu berjalan: dari PHP+Apache di server terpisah, jadi Route Handler Next.js yang sudah co-located dengan `apps/web` — konsisten dengan alasan ADR-012 (hindari server terpisah tanpa manfaat operasional nyata). Tidak ada lagi dependency ke jaringan sekolah/server fisik `10.10.10.100` sama sekali.
- **Field baru `kode` (kode verifikasi)** — satu-satunya perubahan fungsional dari template PHP asli — ditambahkan sebagai baris `<div class="kode-verifikasi">` di bagian footer struk, sesuai kebutuhan T024 asli (antisipasi pemalsuan surat, ADR-018).
- Semua value yang di-interpolasi ke HTML di-escape (`escapeHtml()`) — PHP asli pakai `echo` mentah tanpa escaping (aman waktu itu karena input dari form internal terpercaya), tapi di Route Handler ini nilai datang dari query string URL yang bisa saja diketik manual — escaping ditambahkan sebagai pencegahan reflected-XSS berbiaya rendah, tidak mengubah tampilan untuk input normal.
- `logo.png` sekolah (diberikan user) dipindah ke `apps/web/public/logo.png` — direferensikan sebagai `/logo.png` (Next.js static asset convention), dengan `onerror` fallback yang sama seperti PHP asli (sembunyikan img kalau gagal load, bukan broken-image icon).
- `NEXT_PUBLIC_PRINT_SERVER_URL` dan `PRINT_SERVER_URL` dihapus dari `.env`/`.env.example` — tidak ada lagi env var untuk print server eksternal, URL cetak sekarang relatif (`/print/struk-izin?...`, same-origin dengan `apps/web`).
- `print.php` di root repo (yang user attach ke sesi ini) dihapus setelah isinya di-port — sudah tidak relevan lagi sebagai artifact terpisah.
- **Verifikasi menyeluruh**: curl langsung ke route (dengan cookie sesi) mengonfirmasi semua 8 parameter (termasuk `kode`) tampil benar di HTML mentah; verifikasi end-to-end via Playwright — submit form Sub-alur A sungguhan → permit tersimpan → popup terbuka ke `/print/struk-izin` (bukan lagi ke `10.10.10.100`) → screenshot mengonfirmasi layout struk 58mm, logo termuat, semua field terisi dari data permit yang baru dibuat, kode verifikasi tampil di footer.
- **T024 sekarang benar-benar 100% selesai** — tidak ada lagi item deploy eksternal yang menunggu, tidak ada lagi dependency ke server fisik sekolah untuk fitur cetak.

**Catatan implementasi (2026-07-14, SEBAGIAN SUDAH DIGANTIKAN — lihat revisi arsitektur 2026-07-15 di atas):**
- ~~⚠️ **PENTING — belum di-deploy**: `print.php` di server fisik `10.10.10.100:8800` **belum diedit**...~~ **Superseded** — user menyediakan source `print.php` asli tanggal 15 Juli dan meminta itu di-port jadi Route Handler internal, menghapus kebutuhan server eksternal sama sekali. Catatan asli di bawah ini dipertahankan untuk jejak sejarah keputusan, bukan lagi mencerminkan state sekarang.
- **Perubahan infrastruktur kecil**: `AccessTokenPayload` (JWT) ditambah field `username` (sebelumnya cuma `sub`, `role`, `teacherId`, `kampusId`, `jti`) — dibutuhkan untuk mengisi parameter `petugas` di URL print.php dengan nama akun yang bermakna, bukan `Piket #18` yang tidak informatif untuk dokumen fisik yang dibaca orang tua/siswa. Field ini tidak sensitif (username, bukan password) jadi aman ditambahkan ke token. `AuthService.issueTokenPair()` dan `apps/web/lib/current-user.ts` (decoder client-side) diupdate sejalan. Test suite (`auth.service.spec.ts`) tidak perlu berubah karena `baseUser` mock sudah punya field `username` dari awal (mengikuti schema Prisma `User`).
- **Env var baru**: `NEXT_PUBLIC_PRINT_SERVER_URL` (sebelumnya `PRINT_SERVER_URL` tanpa prefix, belum dipakai di kode manapun) — wajib `NEXT_PUBLIC_` karena `window.open()` construction terjadi di browser (client component), bukan server-side.
- **UI**: route baru `apps/web/src/app/(piket)/piket/izin-keluar/`. Ditambahkan `PiketNav` — tab bar horizontal ringkas (bukan Sidebar penuh gaya admin) di `(piket)/layout.tsx`, dipakai bersama oleh `/piket` dan `/piket/izin-keluar` — pertama kalinya shell piket (yang sejak T023 sengaja tanpa navigasi sama sekali) punya lebih dari 1 halaman, jadi baru sekarang navigasi antar-halaman dibutuhkan. Tetap jauh lebih ringkas dari `Sidebar` admin (cuma 2 tab, tanpa ikon, height minimal) — konsisten dengan prinsip "guru_piket cuma dapat navigasi seperlunya" dari T023.
- **Autocomplete siswa**: reuse data `piketBoard` yang sudah di-fetch di server component (bukan endpoint baru) — filter nama client-side saat mengetik. Cukup untuk skala kampus (puluhan-ratusan siswa), tidak perlu endpoint search/debounce server-side terpisah.
- **Sub-alur B**: reuse data `piketBoard` yang sama, filter `waktuPulang !== null` untuk daftar kandidat. Field `pulangVia` di-patch optimis di state lokal setelah konfirmasi sukses (tidak re-fetch board), konsisten dengan pola optimistic update yang sudah dipakai di halaman `/piket` (T023).
- **Kombinasi tanggal+jam untuk `jamKeluar`/`jamKembaliDiharapkan`**: input `<input type="time">` HTML native (pola yang sama dipakai di halaman Jadwal, T006) hanya menghasilkan string `HH:mm`, digabung dengan tanggal hari ini di client (`combineWithToday()`) sebelum dikirim sebagai ISO datetime ke backend (`CreatePermitDto` mengharap `IsDateString()` lengkap).
- **Verifikasi UI end-to-end via Playwright** (browser asli, termasuk skenario yang tidak bisa diuji lewat curl): login piket → klik tab "Perizinan Keluar" di nav → Sub-alur B menampilkan Ahmad Fauzi (sudah tap pulang) dengan status "Tap Normal" dan tombol aktif → isi form Sub-alur A untuk Siti Nurhaliza (autocomplete, kategori, keterangan, jam keluar/kembali) → submit → **popup window ditangkap oleh Playwright** (`context.waitForEvent("page")`, tidak menunggu load karena server print.php sungguhan tidak reachable dari environment ini) dan URL popup dikonfirmasi berisi ke-8 parameter yang benar dengan encoding URL yang tepat (`petugas=piket_kampus1&tgl=14%2F7%2F2026&nama=Siti+Nurhaliza&kls=XI+TKJ+1&alasan=izin&ket=Berobat+ke+klinik&jamkembali=11%3A00&kode=RR5BE6WS`) → kembali ke Sub-alur B, klik "Tandai Izin Pulang" untuk Ahmad Fauzi → badge berubah ke "Izin Pulang" live, tombol hilang, dan dikonfirmasi juga langsung di database (`pulang_via` berubah `tap`→`tap_izin_pulang`).
- Tidak ada bug baru ditemukan. Data uji (akun piket, tap, permit, activity_log terkait) sudah dibersihkan seluruhnya setelah verifikasi.

---

### T025 — Dashboard Piket: Monitoring & Antrian Klarifikasi
- [x] Section "Belum Kembali" di `/piket`:
  - Query: `permits` jenis `keluar`, `status_kembali: belum`, `jam_kembali_diharapkan < now()`
  - Tombol per baris: **[Sudah Kembali]** → `confirm-kembali` | **[Dianggap Pulang]** → `set-pulang`
- [x] Section "Tidak Tap Pulang Kemarin":
  - Query: `attendance_records` tanggal kemarin, `waktu_pulang = null`, siswa kampus ini
  - Tombol per baris: **[Konfirmasi Pulang]** (input jam) | **[Tandai Izin Keluar Tidak Kembali]** (buat permit retroaktif)

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], T016

**Catatan implementasi (2026-07-15):**
- **"Belum Kembali"**: endpoint baru `GET /permits/belum-kembali` di `PermitsService` — query `jenis=keluar, statusKembali=belum, jamKembaliDiharapkan < now()`, di-scope kampus. Aksi "Sudah Kembali"/"Dianggap Pulang" REUSE `confirmKembali`/`setPulang` yang sudah dibangun penuh di T022 — tidak ada logic baru untuk dua aksi ini, murni endpoint listing baru + UI.
- **"Tidak Tap Pulang Kemarin"**: endpoint baru `GET /attendance/tidak-tap-pulang-kemarin` di `AttendanceService` — query kampus-scoped live (BUKAN bergantung pada hasil `EndOfDayService.detectMissingCheckouts()` dari T016, sesuai catatan eksplisit di komentar kode T016 sendiri: "Dashboard Piket (T025) query ulang secara live saat dibuka, tidak bergantung pada hasil job ini" — job T016 tetap jalan untuk logging/observability, dua hal ini sengaja tidak disatukan).
- **2 aksi resolusi baru** untuk antrian ini:
  - `POST /attendance/:record_id/konfirmasi-pulang-retroaktif` — isi jam pulang perkiraan, ATAU kosongkan untuk "tidak diketahui". Kasus "tidak diketahui" secara desain tetap menyimpan `waktu_pulang: null`, tapi `pulang_via` diisi `piket_izin` — pola ini dipakai sebagai penanda "record sudah diklarifikasi" (beda dari record yang belum pernah ditinjau sama sekali, yang juga punya `waktu_pulang: null` tapi `pulang_via: null`). Tidak perlu kolom/flag baru di skema.
  - `POST /permits/tandai-izin-tidak-kembali` — bikin `permits(jenis: keluar, status_kembali: pulang)` retroaktif untuk tanggal record yang diklarifikasi, plus update `attendance_records` yang sama.
- **Keputusan desain (dikonfirmasi ke user sebelum coding)**: field `jam_keluar` untuk permit retroaktif "Tandai Izin Keluar Tidak Kembali" **wajib diisi** di skema tapi piket sama sekali tidak tahu jam pastinya (itulah kenapa jadi kasus klarifikasi, bukan izin keluar resmi). Dipakai estimasi dari `jam_selesai` jadwal `jam_sekolah` pada hari itu (bukan input manual dari piket) — form tetap sederhana tanpa memaksa piket mengarang angka yang mereka sendiri tidak tahu.
- **Bug ditemukan+diperbaiki sebelum sempat ke production** (ditemukan lewat verifikasi manual nilai tersimpan di database, bukan dari gejala UI): draft awal helper `combineDateAndTime` di `PermitsService` memakai `Date.UTC(y, m, d, hours, minutes)` untuk menggabung tanggal record (yang sudah UTC-midnight-normalized) dengan jam `HH:mm` dari jadwal — ini SALAH karena `HH:mm` di jadwal dimaksud sebagai jam dinding WIB, bukan UTC. Untuk server WIB (UTC+7), ini menghasilkan pergeseran 7 jam (jam_selesai "15:00" tersimpan sebagai 22:00 WIB, bukan 15:00 WIB). Root cause: helper `combineDateAndTime` yang SUDAH ADA di `AttendanceService` (dipakai sejak T012) memakai pola berbeda — `setHours()` pada Date asli (bukan `Date.UTC()`) — karena input aslinya adalah timestamp nyata (`scannedAt`), bukan tanggal ter-normalisasi. Draft awal keliru menyalin pola yang salah konteks. Diperbaiki dengan membangun ulang Date LOKAL dari komponen UTC tanggal record, baru `setHours()` — dikonfirmasi benar lewat pengecekan manual: `jamSelesai="15:00"` pada hari Selasa tersimpan sebagai `2026-07-14T08:00:00.000Z` (=15:00 WIB), bukan `22:00Z`. Ini bug timezone kelas yang sama dengan insiden T012/T016 — kali ini tertangkap sebelum verifikasi selesai, bukan lewat regresi berikutnya.
- **UI**: dua section baru (`BelumKembaliSection`, `TidakTapPulangSection`) ditambahkan ke `piket-board-view.tsx` (T023) di bawah tabel utama — state di-patch optimis (filter item yang baru diresolusi dari array lokal) tanpa re-fetch penuh, konsisten dengan pola yang sudah dipakai board utama. Dialog klarifikasi (`TidakTapPulangForm`) menggabungkan kedua opsi resolusi (form isi jam + tombol aksi terpisah) dalam 1 dialog supaya piket bisa lihat & pilih tanpa berpindah context.
- **Verifikasi API menyeluruh via curl**: data uji nyata untuk kedua skenario (permit overdue, attendance record kemarin tanpa pulang) — `belum-kembali` & `tidak-tap-pulang-kemarin` menampilkan data yang tepat dan ter-scope kampus; `konfirmasi-pulang-retroaktif` dengan jam maupun tanpa jam (kasus "tidak diketahui") keduanya diverifikasi hasil di DB; `tandai-izin-tidak-kembali` diverifikasi menghasilkan permit + update attendance_records yang benar (termasuk setelah fix bug timezone); role guard (`super_admin` → 403) dan idempotency (record yang sudah py `waktu_pulang` → 403 kalau dicoba lagi) dikonfirmasi.
- **Verifikasi UI end-to-end via Playwright** (browser asli): login piket → dashboard menampilkan ketiga section (papan utama, Belum Kembali, Tidak Tap Pulang Kemarin) dengan data yang cocok dan format tanggal/jam yang mudah dibaca ("15 Jul, 07.39") → klik "Sudah Kembali" → baris hilang live dari section Belum Kembali → klik "Klarifikasi" pada baris Tidak Tap Pulang → dialog menampilkan kedua opsi resolusi → klik "Tandai Izin Keluar Tidak Kembali" → baris hilang live, dikonfirmasi juga di database (waktu_pulang & pulang_via terisi benar, permit retroaktif tercipta dengan kode verifikasi).
- Data uji (akun piket, permit, attendance_records, activity_log terkait) sudah dibersihkan seluruhnya setelah verifikasi.

---

### T026 — Dashboard Piket: Lock/Unlock Siswa
- [x] API (akses: `guru_piket`):
  - `POST /students/:id/lock` — isi `locked_reason`, set `locked_at`, `locked_by`
  - `POST /students/:id/unlock` — isi `unlock_note`, set `unlocked_at`, `unlocked_by`
- [x] Section "Perlu Ditinjau" di `/piket`:
  - Siswa dengan `permits(keluar)` lewat jam kembali + `status_kembali: belum` dari hari-hari sebelumnya (bukan hanya kemarin)
  - Tombol **[Kunci Siswa]** → form isi alasan → POST lock
- [x] Section "Siswa Terkunci":
  - Daftar siswa kampus ini yang `locked_at IS NOT NULL`
  - Tombol **[Buka Kunci]** → form isi catatan → POST unlock
- [x] Di T012 (tap logic): tambahkan cek `students.locked_at` — kalau ada → `rejected_locked` di tap_events, response untuk kiosk tampilkan "Hubungi Guru Piket" *(sudah dikerjakan sejak T012, dikonfirmasi ulang di task ini — bukan pekerjaan baru)*

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], [[Projek/AbsenSI/11-Decisions|ADR-017]]

**Catatan implementasi (2026-07-15):**
- Item terakhir ("cek `locked_at` di tap logic") ternyata **sudah dikerjakan sejak T012** — `AttendanceService.tap()` sudah punya `if (card.student?.lockedAt) { ... return { result: rejected_locked, message: "Hubungi Guru Piket" } }`. Dikonfirmasi ulang lewat curl (tap kartu siswa terkunci → `rejected_locked` + pesan yang tepat + tercatat di `tap_events`), bukan pekerjaan baru — murni verifikasi regresi.
- **Lock/unlock ditaruh di `StudentsController`** (bukan modul Permits atau modul baru) karena secara REST resource memang milik `/students/:id` — tapi digerbang dengan **override role di level method**: class-level `@Roles(super_admin, card_admin)` (dipakai `GET /students`, `GET /students/:id` untuk admin), method-level `@Roles(guru_piket)` khusus di `lock`/`unlock`/`terkunci`. Pola override ini sudah dipakai sebelumnya di Calendar module (T010) — `RolesGuard` pakai `getAllAndOverride` dari `Reflector`, jadi metadata handler menang atas metadata class. Diverifikasi eksplisit lewat curl: `super_admin` masih bisa `GET /students` normal (200, tidak ada regresi ke fitur admin yang sudah ada) tapi ditolak di `POST /students/:id/lock` (403) — dua route berbeda kewenangan di controller yang sama, keduanya benar.
- **"Perlu Ditinjau" vs "Belum Kembali" (T025) — beda scope yang disengaja**: "Belum Kembali" (T025) menampilkan SEMUA permit `keluar` yang `jamKembaliDiharapkan` sudah lewat, termasuk yang overdue hari ini juga (kejadian biasa, siswa mungkin baru terlambat sedikit). "Perlu Ditinjau" (T026) HANYA permit dengan `tanggal < hari ini` — sinyal lebih serius (sudah lewat 1 hari lebih, bukan cuma telat beberapa jam). Keduanya BOLEH overlap (permit lama muncul di kedua daftar) — bukan bug, memang dua lensa berbeda atas data yang sama: satu untuk monitoring harian, satu untuk pertimbangan lock.
- **`lockedReason` tidak dihapus saat unlock** — sengaja dipertahankan sebagai riwayat (kenapa siswa ini pernah dikunci), sementara `lockedAt` di-null-kan (status aktif: tidak terkunci lagi). Field `unlockedAt`/`unlockedById`/`unlockNote` terisi sebagai catatan penutupan kasus. Kalau siswa dikunci lagi di masa depan, `lockedReason` yang lama ditimpa dengan alasan baru — riwayat lock/unlock lengkap idealnya dari `activity_log` (`student.lock`/`student.unlock` sudah tercatat di sana lewat `@LogActivity`), bukan dari kolom `students` yang cuma menyimpan state lock saat ini.
- **Duplicate-lock & duplicate-unlock dicegah eksplisit** (409 `ConflictException`) — lock siswa yang sudah terkunci, atau unlock siswa yang tidak sedang terkunci, keduanya ditolak alih-alih diam-diam menimpa state. Konsisten dengan ADR-017 (keputusan manual, harus disengaja).
- **Verifikasi API menyeluruh via curl**: lock berhasil dengan data lengkap (`lockedBy` ter-include); tap kartu siswa terkunci → `rejected_locked` dikonfirmasi nyata (bukan cuma baca kode); `GET /students/terkunci` menampilkan siswa yang tepat; lock kedua kali → 409; unlock berhasil, field terisi benar, riwayat `lockedReason` dipertahankan; tap setelah unlock → `accepted` normal lagi; cross-kampus lock (piket kampus lain) → 403; `super_admin` di endpoint lock → 403 (override method-level dikonfirmasi bekerja) tapi tetap 200 di `GET /students` biasa (tidak ada regresi).
- **Verifikasi UI end-to-end via Playwright** (browser asli, skenario penuh 5 section sekaligus aktif — papan utama, Belum Kembali, Tidak Tap Pulang Kemarin, Perlu Ditinjau, Siswa Terkunci): login piket dengan data seed campuran (1 siswa sudah terkunci dari sebelumnya, 1 permit overdue 2 hari) → semua 5 section tampil benar dan konsisten dengan design system → klik "Kunci Siswa" di Perlu Ditinjau → dialog isi alasan → submit → LIVE tanpa reload: baris di papan utama berubah badge "Terkunci", section Perlu Ditinjau kosong, section Siswa Terkunci bertambah 1 → klik "Buka Kunci" → dialog isi catatan → submit → LIVE: baris papan utama kembali "Belum Hadir" dengan tombol Izin/Sakit muncul lagi, section Siswa Terkunci berkurang 1.
- Data uji (akun piket, permit, lock state) sudah dibersihkan seluruhnya setelah verifikasi. **Ini menandai T022–T026 (Blok 7 — Dashboard Piket, Fase 1b) SELESAI SEMUA** — *(update 2026-07-15: item prasyarat deploy print.php eksternal yang disebut di sini sudah tidak berlaku lagi, lihat revisi arsitektur di catatan T024)*.

---

## 🔐 Blok 8 — Kiosk Auth Refactor
> Dependency: T002 (schema), T003 (KioskGuard), T011–T012 (kiosk app + tap API)
> Task ini breaking change terhadap T003 (KioskGuard) dan T011 (cara kiosk kirim token). Jalankan migration + update sekaligus dalam 1 sesi agar tidak ada periode di mana sistem setengah-lama setengah-baru.

### T027 — Manajemen Kiosk: Auth Berlapis (Token DB + IP Whitelist)

**Phase 1 (backend) selesai & terverifikasi 2026-07-16.** Phase 2 (apps/kiosk) & Phase 3 (UI admin) menyusul — kemungkinan digabung dengan T028 karena keduanya menyentuh form Tambah Kiosk yang sama.

- [x] Tambah tabel `kiosks` ke `schema.prisma` + jalankan migration — pakai `Int @id @default(autoincrement())`, BUKAN `String cuid()` seperti draf awal spec, supaya konsisten dengan seluruh model lain di schema ini.
- [x] Refaktor `KioskGuard`: query token dari DB `kiosks`, validasi `allowed_ip`, update `last_seen_at` fire-and-forget
- [x] Update `AttendanceService.tap()`: ambil `kiosk_id` dari `req.kiosk.id` (hasil guard), bukan dari request body — dihapus dari `TapDto`
- [ ] Update `apps/kiosk`: baca token dari URL param `?device=TOKEN` → simpan ke `localStorage` → kirim ke Route Handler proxy → sertakan di `Authorization: Bearer`
- [ ] Tampilkan layar error di kiosk kalau tidak ada token (localStorage kosong + tidak ada URL param)
- [ ] Admin UI di `/kiosk`: tabel kiosk + status online/offline + form tambah kiosk + modal URL+QR code — **form tambah kiosk digabung dengan field Tipe Kiosk dari T028 (ADR-022)**
- [x] API endpoints: `GET/POST /kiosks`, `GET/PATCH /kiosks/:id`, `PATCH /kiosks/:id/deactivate`, `PATCH /kiosks/:id/rotate-token`
- [ ] Hapus `KIOSK_DEVICE_TOKEN` dari `.env`, update `10-Environment.md` dan `CLAUDE.md` — ditunda sampai Phase 2 (`apps/kiosk`) selesai, supaya kiosk dev server tidak mendadak berhenti berfungsi di tengah refactor
- [x] Verifikasi regresi: tap end-to-end via curl (create kiosk → tap sukses dengan token+IP benar → `tap_events.kiosk_id` terisi FK yang benar → token salah/kiosk nonaktif/IP salah semua ditolak dengan status code yang benar → rotate-token invalidate token lama)

**Catatan implementasi:** Ditemukan & diperbaiki bug normalisasi IP saat verifikasi — Node melaporkan loopback sebagai `::1` (bukan `127.0.0.1` literal) dan IPv4 lewat dual-stack socket sebagai `::ffff:x.x.x.x` — `KioskGuard.extractClientIp()` sekarang menormalisasi keduanya ke bentuk IPv4 murni sebelum dibandingkan dengan `allowedIp`.

**Ref:** [[Projek/AbsenSI/06-Features/tasks/T027-manajemen-kiosk|T027-manajemen-kiosk.md]], [[Projek/AbsenSI/11-Decisions|ADR-021]]

---

## 🖼️ Blok 9 — Profil Lengkap + Foto + Kiosk Scan Terpisah

> Dependency: T005 (Students & Teachers), T011 (Kiosk UI dasar), T027 (tabel `kiosks` harus ada dulu sebelum tambah kolom `tipe`)
> Didiskusikan lewat AskUserQuestion sebelum eksekusi (2026-07-17) — semua keputusan desain besar sudah dikonfirmasi user, dicatat di ADR-022 & ADR-023.

> **Semua keputusan desain final 2026-07-17** (auth foto, realtime tabel guru, textarea, form 1-dialog, batas ukuran foto) — lihat tabel "Keputusan Final" di file task. Dipecah jadi 5 sub-task linear, tiap sub-task diverifikasi (build+test+manual) sebelum lanjut ke berikutnya.

### ✅ T028a — Migrasi Schema

**Selesai 2026-07-17.**

- [x] Enum baru: `JenisKelamin`, `Agama`, `KioskTipe`
- [x] `Student` +8 kolom biodata (semua nullable): `tempatLahir`, `jenisKelamin`, `agama`, `alamat` (`@db.Text`), `rtRw`, `namaAyah`, `namaIbu`, `foto`
- [x] `Teacher.nip` → `niy` (RENAME COLUMN manual di SQL migration, bukan drop+add) + tambah `noHp`, `foto`
- [x] `Kiosk.tipe` (non-nullable — tabel `kiosks` kosong saat migrasi, tanpa backfill)
- [x] `TapResult` + `rejected_wrong_kiosk_type`

**Catatan implementasi:**
- Migration ditulis manual (bukan hasil auto `prisma migrate dev`) — dicek dulu lewat `prisma migrate diff --script` untuk lihat apa yang Prisma akan generate secara default: benar seperti dugaan, Prisma mau `DROP COLUMN nip` + `ADD COLUMN niy VARCHAR(191) NOT NULL` (2 operasi terpisah, akan menghapus 5 baris data NIP existing tanpa peringatan). Migration file `20260717004344_add_profil_lengkap_foto_kiosk_tipe/migration.sql` dibuat manual dengan `ALTER TABLE teachers RENAME COLUMN nip TO niy;` sebagai gantinya, plus `RENAME INDEX teachers_nip_key TO teachers_niy_key` supaya nama index tetap konsisten.
- Diterapkan lewat `prisma migrate deploy` (bukan `migrate dev`, supaya migration file yang sudah ditulis manual tidak ditimpa ulang oleh Prisma).
- **Verifikasi data selamat:** `SELECT niy, nama FROM teachers` setelah migrasi — semua 5 baris NIP lama (termasuk "198703152011012002 — Ratna Dewi, S.Pd" dst.) utuh di kolom `niy` yang baru, tidak ada yang NULL.
- `npx prisma generate` sukses, client TypeScript baru menghasilkan tipe `JenisKelamin`, `Agama`, `KioskTipe` dengan benar.
- **Build `pnpm --filter @absensi/api build` sengaja masih merah (11 error) setelah T028a** — semua error adalah referensi `nip`/`tipe` yang belum diupdate di kode (bukan bug migrasi): `cards.service.ts`, `teachers.service.ts`, `import.service.ts` masih pakai `nip`; `kiosks.service.ts` belum kirim `tipe` saat create. Ini persis scope T028b, bukan regresi — deliberately left for the next sub-task per rencana pemecahan.
- Jest suite (`pnpm --filter @absensi/api test`) tetap hijau — test yang ada tidak menyentuh flow teacher/kiosk creation, jadi tidak kena efek `nip`/`tipe` yang belum diupdate.

### ✅ T028b — Backend Core

**Selesai 2026-07-17.**

- [x] `KioskGuard` attach `tipe` ke `request.kiosk`; `AttendanceService.tap()` validasi tipe kiosk vs tipe kartu (ADR-022) — kartu salah tipe ditolak, tetap tercatat `tap_events` dengan `result=rejected_wrong_kiosk_type`, TIDAK bikin `attendance_record`
- [x] DTO Siswa (`create`+`update`, 7 field baru optional, `foto` TIDAK lewat DTO ini) — `PATCH /students/:id` **belum ada sebelumnya**, dibuat baru (`UpdateStudentDto`, `StudentsService.update()`, controller endpoint)
- [x] DTO Guru (`create`+`update`, `nip`→`niy`, +`noHp`) — `PATCH /teachers/:id` **belum ada sebelumnya**, dibuat baru sama seperti siswa
- [x] Cari-ganti SEMUA referensi `nip`→`niy` di `apps/api` dan `apps/web` — `grep -rn "\bnip\b"` nol hasil di kedua app setelah selesai

**Catatan implementasi:**
- File yang disentuh untuk cari-ganti `nip`→`niy`: `cards.service.ts` (5 titik — select/query/interface), `import.service.ts` (`TeacherRow` interface, `importTeachers()`, dan 2 pesan error CSV cards yang menyebut "NIP guru"→"NIY guru" untuk konsistensi terminologi — kolom `nisn_nip` di CSV cards **sengaja dibiarkan** karena itu format gabungan lookup, bukan identifier guru itu sendiri), `users.service.ts` (`USER_SELECT`), `apps/web/src/lib/core-types.ts` (`Teacher.nip`→`niy`, plus tambah `noHp`/`foto`/tipe enum baru `JenisKelamin`/`Agama`), `apps/web/.../guru-view.tsx` (kolom tabel, form, `#nip-guru`→`#niy-guru`), `apps/web/.../import-view.tsx` (contoh CSV teacher).
- `KioskTipe` juga sempat ketinggalan dari `CreateKioskDto`/`KiosksService.create()` (ditemukan saat build check di akhir T028a) — sekalian diperbaiki di sini karena satu rumpun dengan validasi tipe kiosk.
- **Verifikasi end-to-end manual (curl):** buat kiosk tanpa `tipe` → ditolak validasi 400; buat kiosk `tipe=siswa` dan `tipe=guru`; tap kartu siswa di kiosk `tipe=guru` → `rejected_wrong_kiosk_type`, `tap_events.kiosk_id` tercatat benar, TIDAK ada `attendance_record` baru; tap kartu siswa yang sama di kiosk `tipe=siswa` → lolos validasi tipe, lanjut ke pemeriksaan berikutnya (debounce, karena tap sebelumnya masih dalam window 30 detik — bukti validasi tipe kiosk berjalan sebagai lapisan terpisah dari debounce). `PATCH /students/:id` dan `PATCH /teachers/:id` diuji langsung dengan field baru, tersimpan dengan benar, lalu direvert lagi ke NULL supaya data tidak tercemar.
- `pnpm turbo run build` (full monorepo) + `pnpm --filter @absensi/api test` (Jest, 9 test) + `tsc --noEmit` di `apps/web` — semuanya hijau.

### ✅ T028c — Backend Foto (modul `photos`)

**Selesai 2026-07-17.**

- [x] Setup folder `storage/photos/{students,teachers,unmatched}/`, masuk `.gitignore` (dengan `.gitkeep` per folder supaya struktur tetap ter-track)
- [x] `POST /photos/upload-bulk` — auto-match nama file (tanpa ekstensi) = NISN/NIY, batas 1MB/file (Multer `limits.fileSize` + validasi service-level), JPEG/PNG saja, laporan `{totalFiles, matchedCount, unmatchedCount, unmatched}`
- [x] `GET /photos/students/:filename`, `GET /photos/teachers/:filename` — guard ganda `PhotoAccessGuard` (token kiosk ATAU JWT admin), bukan public
- [x] `PATCH /photos/assign` — assign manual file unmatched ke siswa/guru, pindah file + update kolom `foto` + hapus dari folder unmatched
- [x] **`DELETE /photos/students/:id`, `DELETE /photos/teachers/:id`** (ditambahkan 2026-07-17 setelah user melaporkan "fitur hapus foto belum ada" pasca-T028e) — hapus file dari disk + null-kan kolom `foto`, guard sama seperti create/upload (`super_admin`/`card_admin`). UI: tombol hapus (ikon sampah, overlay hover) di halaman detail siswa (`/siswa/:id`) dan section baru "Foto Tersimpan" di menu `/foto` (daftar semua siswa+guru yang sudah punya foto, dengan thumbnail + tombol hapus per baris — sebelumnya menu ini cuma bisa upload, tidak ada cara lihat/kelola foto yang sudah ada). Diverifikasi lewat API langsung (400 kalau belum punya foto, 404 kalau id tidak ada) dan Playwright UI di kedua lokasi — foto berganti ke fallback avatar / hilang dari daftar tanpa reload.

**Catatan implementasi:**
- `apps/api/src/photos/photo-access.guard.ts` (baru) — guard custom (bukan kombinasi `@UseGuards` dua guard sekaligus, karena butuh logic "coba A, kalau gagal coba B" bukan "wajib A DAN B"): cek dulu apakah token cocok kiosk aktif di tabel `kiosks`, kalau tidak baru coba `jwtService.verifyAsync()` + cek blacklist (pola sama persis dengan `AttendanceGateway`'s Socket.IO auth di T017).
- `apps/api/src/photos/photos.service.ts` — pakai `node:fs/promises` murni (bukan library upload pihak ketiga) karena kasusnya simpel: simpan buffer dari Multer ke path yang sudah ditentukan (`{id}.{ext}`), tidak perlu abstraksi storage-provider (konsisten ADR-023, tidak akan pindah ke cloud storage dalam waktu dekat). Upload ulang foto (siswa ganti foto) otomatis hapus file lama dengan ekstensi berbeda dulu, supaya tidak ada file yatim.
- `@absensi/types` — tambah `PhotoUploadReport`/`PhotoUnmatchedEntry`, dan **tambah `TapResult.REJECTED_WRONG_KIOSK_TYPE`** yang sebelumnya luput dari T028b (enum ini didefinisikan terpisah dari `TapResult` Prisma di `apps/api`, ternyata ada juga salinan di `@absensi/types` yang dipakai `apps/kiosk` — build gagal begitu ditambahkan karena `apps/kiosk/src/lib/tap-messages.ts` pakai `Record<Exclude<TapResult, ACCEPTED>, string>` yang exhaustive, jadi sekalian ditambahkan pesan `"Kartu ini bukan untuk gerbang ini"` di sana supaya build tidak merah — bukan scope creep ke T028e, cuma menjaga tipe tetap valid, styling/UX pesan ini tetap job T028e).
- **Verifikasi end-to-end manual:** upload 3 file (JPEG asli via Pillow) — 1 nama sesuai NISN siswa, 1 sesuai NIY guru, 1 nama sembarang → hasil 2 matched + 1 unmatched persis sesuai ekspektasi, dicek juga langsung ke DB (`students.foto`/`teachers.foto` terisi) dan filesystem (`storage/photos/{students,teachers,unmatched}/`). Serve foto diuji dengan JWT admin (200), token kiosk aktif (200), tanpa token (401), token sampah (401). Assign manual diuji: file unmatched dipindah ke siswa lain, kolom `foto` update, file hilang dari folder unmatched. Upload file >1MB → ditolak 413 di layer Multer (sebelum sempat masuk service). Upload file `text/plain` menyamar `.jpg` → ditolak service-level dengan pesan jelas, masuk laporan unmatched (bukan crash). Semua data test (kiosk sementara, foto, kolom `foto` di 2 siswa + 1 guru) dibersihkan lagi di akhir.
- `pnpm turbo run build` (full monorepo, termasuk `apps/kiosk` yang sempat merah karena TapResult enum) + `pnpm --filter @absensi/api test` (Jest, 9 test) — semuanya hijau.

### ✅ T028d — Frontend Admin

**Selesai 2026-07-17.**

- [x] Form Siswa (1 dialog, tidak di-split): tambah 7 field baru, `alamat` pakai `<textarea>` native (bukan komponen baru di `packages/ui`)
- [x] Form Guru: NIY (bukan NIP, sudah benar sejak T028b) + No HP
- [x] Halaman detail siswa: tampilkan foto (fallback avatar kalau kosong) + biodata lengkap
- [x] Menu baru `/foto`: upload bulk + laporan + assign manual untuk unmatched
- [x] Form Tambah Kiosk: field Tipe Kiosk (Select, wajib) — sekalian bangun seluruh UI `/kiosk` dari nol (T027 Phase 3 belum pernah dikerjakan) termasuk tabel, status online/offline, modal URL+QR code, rotate token, nonaktifkan

**Catatan implementasi:**
- **Proxy foto binary (masalah baru yang ditemukan):** `apiFetch`/`apiClientFetch`/proxy generik yang sudah ada semua asumsikan respons JSON (selalu `JSON.parse` body) — tidak bisa dipakai untuk `<img src>` yang perlu byte gambar mentah dengan `Content-Type` yang benar. Dibuat route baru `apps/web/src/app/api/photo-proxy/[...path]/route.ts` yang sisipkan access token dari cookie httpOnly (sama seperti proxy biasa) tapi teruskan `response.arrayBuffer()` apa adanya, bukan JSON.
- **Upload foto tidak butuh route baru** — sempat mau bikin `photo-upload/route.ts` sendiri, tapi ternyata `apps/web/src/app/api/proxy-upload/[...path]/route.ts` (dipakai fitur Import CSV) sudah generik dan pas dipakai ulang untuk `/api/proxy-upload/photos/upload-bulk` tanpa modifikasi — dihapus lagi supaya tidak duplikasi logic.
- **`NEXT_PUBLIC_KIOSK_URL` (env var baru)** — dibutuhkan untuk generate URL+QR code kiosk yang benar (harus arahkan ke `apps/kiosk`, port 3002, bukan port admin 3000). Ditambahkan ke `.env` dan `.env.example` dengan komentar jelas ganti ke IP LAN saat deploy.
- **`qrcode.react` (dependency baru)** — dipasang di `apps/web` untuk render QR code di modal URL Kiosk, sesuai spec asli T027 yang memang meminta QR code client-side tanpa backend tambahan.
- `apps/web/src/lib/core-types.ts` — tambah interface `Kiosk` + type `KioskTipe` (belum ada sebelumnya, T027 cuma sempat bikin backend).
- UI `/kiosk` dari nol: tabel (Nama/Kampus/Tipe/IP/Status online-offline/Terakhir Aktif/Aksi), dialog Tambah Kiosk (dengan Select Tipe wajib), modal URL+QR (fetch `GET /kiosks/:id` on-demand untuk ambil `deviceToken` — sengaja tidak di-preload di list, sesuai catatan keamanan T027 "JANGAN expose deviceToken di list"), tombol Rotate Token, dialog konfirmasi Nonaktifkan.
- **Verifikasi Playwright:** create siswa dengan semua 7 field baru → halaman detail (navigasi langsung ke `/siswa/:id`, bukan via klik nama karena ada race-condition timing di test pertama yang screenshot sebelum navigasi selesai — dikonfirmasi ulang dengan navigasi langsung) menampilkan semua field dengan benar termasuk fallback avatar untuk siswa tanpa foto. Guru dengan NIY+No HP tersimpan. Halaman `/foto` termuat dengan instruksi jelas. Form Kiosk punya field Tipe wajib, create sukses dengan badge Tipe tampil di tabel, status Offline benar (belum pernah tap), modal URL+QR menampilkan device token dan IP LAN dari env yang benar.
- Data test dibersihkan: kiosk test dihapus dari DB (sudah ada endpoint delete lewat SQL manual karena API cuma expose deactivate). Siswa/guru test **dibiarkan** — konsisten dengan keterbatasan yang sama seperti P005 (belum ada endpoint DELETE untuk keduanya, di luar scope T028d).
- `./scripts/build.sh` (full monorepo, termasuk `apps/kiosk`) dan `./scripts/test.sh @absensi/api` (Jest, 9 test) — semuanya hijau, dijalankan lewat script operasional yang baru dibuat sesi ini.

### ✅ T028e — Kiosk App

**Selesai 2026-07-17. Sub-task terakhir T028 — semua 5/5 selesai.**

- [x] Deteksi tipe kiosk dari data kiosk sendiri (localStorage bareng token), bukan dari kartu
- [x] Varian siswa: foto+nama+jam (regresi T011 aman)
- [x] Varian guru: foto+nama+jam + 2 tabel "5 terbaru datang/pulang" — update **realtime via Socket.IO**, BUKAN polling
- [x] Kartu salah tipe kiosk → pesan jelas di layar, bukan crash

**Catatan implementasi — scope meluas dari rencana awal (dikonfirmasi user sebelum eksekusi):**

Saat survei kode ditemukan **T027 Phase 2 (alur token kiosk) ternyata belum pernah dikerjakan sama sekali** — `apps/kiosk` masih pakai `KIOSK_DEVICE_TOKEN` statis dari env dan `NEXT_PUBLIC_KIOSK_ID` hardcoded, bukan token per-device dari URL seperti ADR-021. T028e tidak bisa jalan tanpa ini (kiosk perlu tahu tipenya dari token), jadi dikerjakan sekalian di sini, bukan ditunda.

Juga ditemukan 2 bug tersembunyi yang jadi blocker realtime guru:
1. `AttendanceGateway.broadcastAttendanceRecorded()` cuma broadcast kalau `kampusId !== null` — tap GURU selalu kirim `kampusId: null` (guru tidak terikat kelas). Diperbaiki: untuk **absen gerbang** (fokus fase ini, per konfirmasi user — kiosk per-kelas menyusul fase 2), `kampusId` tap guru diambil dari **kampus milik kiosk** yang dipakai tap, bukan data guru.
2. `AttendanceGateway.handleConnection()` masih validasi kiosk pakai `KIOSK_DEVICE_TOKEN` env statis (sisa sebelum T027) — diperbaiki ke query tabel `kiosks` seperti `KioskGuard`.

**Backend:**
- `TapResponse`/`TapResultPayload` (`@absensi/types` dan `apps/api`) — tambah field `foto`; `buildAcceptedResponse()` include foto siswa/guru dari record.
- `KioskRecentEntry`/`KioskRecentPayload` (`@absensi/types`, baru) — bentuk payload tabel "5 terbaru".
- `AttendanceGateway.computeKioskRecent(kampusId)` — query 5 `attendance_records` terbaru (`teacherId` tidak null, `kiosk.kampusId` cocok) untuk masuk & pulang; dipanggil REST (`GET /attendance/kiosk-recent`, guard `KioskGuard`) untuk initial load, dan otomatis lewat `broadcastAttendanceRecorded()` setelah tap guru sukses (emit ke channel `attendance:kiosk:{kampusId}`, kiosk guru auto-join channel ini saat connect kalau `tipe=guru`).
- `KiosksController`/`KiosksService` — endpoint baru `GET /kiosks/me` (guard `KioskGuard`, bukan JWT admin) untuk kiosk tahu tipenya sendiri (`findMinimal()`, sengaja TIDAK include `deviceToken`).
- `AttendanceService.tap()` — signature tambah parameter `kioskKampusId`, dipakai untuk `kampusId` job queue kalau tap dari guru (siswa tetap dari `kelas.kampusId`).

**`apps/kiosk` (rombak signifikan):**
- `lib/kiosk-init.ts` (baru) — baca `?device=TOKEN` dari URL, simpan `localStorage`, fallback baca localStorage kalau tidak ada URL param (persis ADR-021).
- `app/api/tap/route.ts` — token dari header `X-Kiosk-Token` (dikirim client), bukan `process.env.KIOSK_DEVICE_TOKEN` — token tetap tidak pernah lewat `NEXT_PUBLIC_*`.
- `app/api/kiosk-info/route.ts`, `app/api/kiosk-recent/route.ts` (baru) — proxy server-side ke `GET /kiosks/me` dan `GET /attendance/kiosk-recent`, pola sama seperti `/api/tap`.
- `app/api/photo-proxy/[...path]/route.ts` (baru) — proxy foto untuk `<img src>` kiosk; beda dari route lain, token diterima lewat **query string** karena `<img>` tidak bisa kirim header custom (bukan exposure baru — token yang sama sudah dipegang penuh localStorage kiosk).
- `lib/tap-client.ts`, `lib/offline-buffer.ts` — `kiosk_id` dihapus dari payload (server sudah tahu dari token, sejak T028b), token dikirim per-call sebagai parameter bukan konstanta modul.
- `lib/use-kiosk-recent.ts` (baru) — hook Socket.IO, auth `{ deviceToken }` di handshake (beda dari `useAttendanceSocket` versi `apps/web` yang pakai `{ token }` JWT), initial fetch REST + subscribe event `attendance:kiosk:update`.
- `components/kiosk-avatar.tsx`, `components/kiosk-recent-table.tsx` (baru) — avatar dengan fallback ikon generik (`onError` sembunyikan `<img>` kalau gagal load), tabel dengan varian `light` (dashboard beige) dan gelap (sisa dari draf awal, masih didukung tapi tidak dipakai lagi).
- `components/not-configured-screen.tsx` (baru) — layar error kalau localStorage kosong DAN tidak ada `?device=` sama sekali.

**Bug desain yang ditemukan & diperbaiki SAAT verifikasi (bukan sebelumnya):** draf pertama menaruh 2 tabel "5 terbaru" di dalam `FeedbackScreen` — cuma muncul 3 detik setelah tap LOKAL kiosk itu sendiri, lalu hilang balik ke idle kosong. Diverifikasi pakai 2 browser context terpisah (satu "observer" idle, satu "tapper") — observer TIDAK menerima update sama sekali karena tabel tidak pernah dirender saat idle. Diperbaiki dengan bikin `components/guru-dashboard.tsx` (baru) — layar default kiosk guru yang SELALU menampilkan kedua tabel secara persisten (bukan cuma sekilas), `FeedbackScreen` disederhanakan jadi overlay sesaat (foto+nama+jam saja) yang numpuk di ATAS dashboard, bukan menggantikannya. Setelah fix, tes ulang dengan 2 context terpisah membuktikan observer menerima update realtime tanpa reload sama sekali.

**Operasional:** `KIOSK_DEVICE_TOKEN` akhirnya dihapus dari `.env`/`.env.example` (mandat T027 yang sempat ditunda sampai `apps/kiosk` tidak lagi menggunakannya) — dicek dengan `grep` di seluruh `apps/`, cuma tersisa di komentar historis dan `dist/` (regenerated).

**Verifikasi end-to-end (Playwright, 2 kiosk device + real browser):**
- Kiosk tanpa token → layar "belum dikonfigurasi".
- Kiosk siswa: idle screen tampil, token persist setelah reload tanpa `?device=`.
- Tap kartu siswa di kiosk siswa → diterima.
- Tap kartu guru di kiosk siswa → ditolak "Kartu ini bukan untuk gerbang ini" (ADR-022).
- Tap kartu guru di kiosk guru → diterima, foto fallback + nama + jam tampil, kedua tabel muncul.
- **2 browser context terpisah** (observer idle vs tapper aktif) — tap di kiosk B ter-refleksi di kiosk A tanpa reload/interaksi apa pun, membuktikan broadcast Socket.IO nyata (bukan cuma initial REST fetch).
- Semua kiosk/kartu/guru test dibersihkan; siswa/guru test dibiarkan (belum ada endpoint DELETE, konsisten P005/T028d).
- `./scripts/build.sh` (full monorepo) + `./scripts/test.sh @absensi/api` (Jest, 9 test) — hijau.

**Ref:** [[Projek/AbsenSI/06-Features/tasks/T028-profil-lengkap-foto-kiosk-scan|T028-profil-lengkap-foto-kiosk-scan.md]], [[Projek/AbsenSI/11-Decisions|ADR-022]], [[Projek/AbsenSI/11-Decisions|ADR-023]]

---

## 📌 Catatan Penting

1. **Sebelum mulai coding setiap task:** baca spec yang direferensikan di bagian `Ref`. Jangan asumsikan — spec sudah lengkap dan ada alasan untuk setiap keputusan.
2. **ADR adalah hukum:** kalau ada konflik antara instinct coding dan ADR, ADR menang. Kalau ADR dirasa perlu diubah, buka dulu diskusi dan buat ADR baru.
3. **Setelah Blok 4 selesai:** uji end-to-end dengan hardware reader RFID fisik sebelum lanjut ke Blok 5. Data tap palsu (input manual dari keyboard biasa) tidak cukup untuk validasi debounce dan timing.
lanjut 
