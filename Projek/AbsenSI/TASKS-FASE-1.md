---
tags: [absensi, tasks, fase-1]
updated: 2026-07-03
---

# AbsenSI — Task Breakdown Fase 1

← [[Projek/AbsenSI/00-INDEX|Index]]

> **Eksekusi solo.** Urutan task dirancang untuk menjaga konsistensi dengan rancangan — setiap blok membangun fondasi untuk blok berikutnya. Jangan loncat blok kecuali dependensinya sudah selesai.
>
> **Cara pakai:** Buka file ini di Obsidian, centang task saat selesai. Sebelum mulai setiap task, baca spec yang direferensikan. Gunakan [[Projek/AbsenSI/11-Decisions|11-Decisions]] sebagai referensi ADR saat ada keputusan arsitektur yang perlu dikonfirmasi.

---

## 📊 Progress

| Blok | Task | Selesai |
|---|---|---|
| 0 — Foundation | T001–T003 | 2/3 |
| 1 — Master Data | T004–T006 | 2/3 |
| 2 — Kartu RFID | T007–T009 | 0/3 |
| 3 — Kalender | T010 | 0/1 |
| 4 — Absensi Gerbang | T011–T016 | 0/6 |
| 5 — Realtime & Dashboard | T017–T019 | 0/3 |
| 6 — Akun Guru | T020–T021 | 0/2 |
| 7 — Dashboard Piket (Fase 1b) | T022–T026 | 0/5 |
| **Total** | | **4/26** |

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
- Dibangun sesuai brief design system baru (vault: `session Claude/design-system/*.md`) — palet beige/orange, radius 24px kartu, Plus Jakarta Sans, lucide-react. Lihat memory Claude Code "design_system_absensi" untuk ringkasan.
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

### T006 — Import CSV: Siswa & Guru
- [ ] API `POST /import/students` — upload CSV, validasi, partial commit, return laporan
  - Validasi: `nisn` unik, `kelas`+`jurusan` harus sudah ada di DB
  - Baris invalid tidak gagalkan seluruh import — laporkan per baris
- [ ] API `POST /import/teachers` — sama strukturnya
- [ ] Admin UI:
  - Halaman Import: tab Siswa / tab Guru
  - Upload file CSV → progress indicator → tampilkan laporan hasil (X berhasil, Y gagal + detail baris gagal)
  - Download laporan gagal sebagai CSV (untuk perbaikan & re-upload)
- [ ] Pastikan urutan dependency dijaga di UI: kelas/jurusan harus ada sebelum import siswa bisa berhasil (tampilkan warning kalau belum ada kelas)

**Ref:** [[Projek/AbsenSI/06-Features/import-data-master|import-data-master.md]]

---

## 💳 Blok 2 — Kartu RFID
> Dependency: T005 (students & teachers harus ada sebagai target mapping)

### T007 — Card Module: CRUD Kartu
- [ ] API endpoints:
  - `GET /cards` (filter: status aktif/nonaktif, linked ke siswa/guru)
  - `POST /cards` — registrasi kartu baru (UID + student_id atau teacher_id)
  - `PATCH /cards/:id/revoke` — nonaktifkan kartu (riwayat tap tetap)
  - `POST /cards/:id/replace` — ganti kartu (kartu lama otomatis `inactive`, kartu baru jadi `active`)
- [ ] Validasi: 1 UID hanya boleh aktif ke 1 orang; UID yang pernah dipakai tidak boleh didaftar ulang ke orang lain
- [ ] Admin UI:
  - Halaman Kartu: tabel semua kartu + filter status + link ke pemilik
  - Form registrasi kartu baru (input UID manual atau scan di PC admin)
  - Tombol revoke + replace per baris

**Ref:** [[Projek/AbsenSI/06-Features/manajemen-kartu|manajemen-kartu.md]], [[Projek/AbsenSI/11-Decisions|ADR-010]]

---

### T008 — Card Import: Mode A (Bulk CSV)
- [ ] API `POST /import/cards` (akses: `super_admin` + `card_admin`)
  - Kolom CSV: `nisn_atau_nip`, `uid`
  - Validasi: `nisn`/`nip` harus ada di DB, `uid` unik & belum pernah dipakai
  - Partial commit + laporan hasil seperti T006
- [ ] Admin UI: tab ketiga di halaman Import — "Pemetaan Kartu (CSV)"

**Ref:** [[Projek/AbsenSI/06-Features/import-data-master|import-data-master.md]], [[Projek/AbsenSI/11-Decisions|ADR-009]]

---

### T009 — Card Import: Mode B (Tap-to-Assign)
- [ ] API `GET /cards/unassigned-persons` — daftar siswa+guru yang belum punya kartu aktif
- [ ] API `POST /cards/tap-assign` — terima `uid` + `person_id` + `person_type`, buat kartu baru aktif
- [ ] Admin UI: halaman Tap-to-Assign
  - Daftar siswa/guru belum punya kartu (dengan search/filter)
  - Pilih orang dari daftar → muncul input tersembunyi yang auto-focus → instruksi "Tempelkan kartu ke reader"
  - Saat UID masuk (keystroke dari reader HID) → langsung POST tap-assign → feedback "✅ Kartu dipasangkan ke [nama]" → otomatis ke orang berikutnya
  - Desain untuk alur cepat berurutan (hari pembagian kartu)

**Ref:** [[Projek/AbsenSI/06-Features/import-data-master|import-data-master.md]], [[Projek/AbsenSI/11-Decisions|ADR-004]]

---

## 📅 Blok 3 — Kalender Pendidikan
> Dependency: T003 (auth), T004 (kampus/kelas sudah ada)

### T010 — Kalender Pendidikan
- [ ] API endpoints (akses: `super_admin`):
  - `GET/POST /academic-years`
  - `PATCH /academic-years/:id/activate` — set aktif (nonaktifkan yang lain otomatis)
  - `GET/POST /school-holidays` (filter by academic_year_id)
  - `DELETE /school-holidays/:id`
  - `PATCH /school-holidays/:id`
- [ ] Admin UI:
  - Halaman Kalender: tampilan grid bulanan (highlight hari aktif vs libur vs Sabtu)
  - Form input libur blok (range tanggal + jenis + keterangan)
  - Quick action "Tandai Libur Mendadak" dari klik tanggal di kalender
  - Edit/hapus entri libur dari klik tanggal yang sudah ditandai libur
  - Manajemen tahun ajaran: buat baru + tombol "Aktifkan"

**Ref:** [[Projek/AbsenSI/06-Features/kalender-pendidikan|kalender-pendidikan.md]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] (tabel `academic_years`, `school_holidays`)

---

## 🚪 Blok 4 — Absensi Gerbang
> Dependency: T002–T007, T010

### T011 — Kiosk App: UI Tap
- [ ] `apps/kiosk` — Next.js app, layout fullscreen (tidak ada navbar, tidak ada scroll)
- [ ] Halaman tap:
  - Input tersembunyi auto-focus menangkap keystroke dari reader HID
  - Saat Enter diterima: kirim ke API, tampilkan feedback 3 detik:
    - ✅ Hadir: nama, foto (opsional), jam tap, status (masuk/pulang/terlambat)
    - ⚠️ Terlambat: sama + highlight merah
    - ❌ Ditolak: pesan sesuai alasan (tidak terdaftar / kartu nonaktif / siswa terkunci "Hubungi Guru Piket")
  - Setelah 3 detik: kembali ke layar idle (jam digital besar, logo sekolah)
- [ ] Auth kiosk: kirim `KIOSK_DEVICE_TOKEN` dari env sebagai Bearer di setiap request

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/08-UI-UX-Guidelines|08-UI-UX-Guidelines]], [[Projek/AbsenSI/11-Decisions|ADR-004]]

---

### T012 — Attendance API: Core Tap Logic
- [ ] `POST /attendance/tap` (dilindungi `KioskGuard`):
  - Cari kartu by UID → validasi aktif/nonaktif/siswa terkunci
  - **Debounce:** cek `tap_events` — kalau kartu yang sama sudah tap < 30 detik yang lalu → `rejected_duplicate`
  - **Tap-1:** buat `attendance_records` baru (status `hadir`/`terlambat` berdasarkan perbandingan `scanned_at` vs jadwal)
  - **Tap-2+:** update `waktu_pulang` ke `scanned_at` yang terbaru, `pulang_via: tap`
  - Threshold terlambat **siswa**: dari `schedules` (jam masuk sekolah hari itu)
  - Threshold terlambat **guru**: jam mengajar pertama hari itu − threshold global dari `schedules`
  - Dispatch event `attendance.recorded` ke BullMQ (ADR-006)
- [ ] Response: data untuk kiosk UI (nama, status, waktu, pesan)

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/11-Decisions|ADR-005, ADR-006]]

---

### T013 — Tap Events Logging
- [ ] Setiap request ke `POST /attendance/tap` — **selalu** buat 1 baris `tap_events`, apapun hasilnya
- [ ] `result` enum sesuai skema: `accepted`, `rejected_inactive`, `rejected_locked`, `rejected_unknown`, `rejected_duplicate`
- [ ] `scanned_at` = timestamp server (bukan dari request body)
- [ ] `attendance_record_id` diisi kalau tap `accepted`
- [ ] Pastikan tidak ada endpoint DELETE/UPDATE untuk `tap_events` di seluruh codebase

**Ref:** [[Projek/AbsenSI/11-Decisions|ADR-020]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]]

---

### T014 — Offline Buffer Kiosk
- [ ] Di `apps/kiosk`: setup IndexedDB (gunakan `idb` library — wrapper IndexedDB yang ringan)
- [ ] Flow:
  1. Tap masuk → generate `client_uuid` (nanoid)
  2. Coba POST ke API (timeout 3 detik)
  3. Sukses → done
  4. Gagal → simpan ke IndexedDB: `{ uid, client_uuid, timestamp: Date.now(), synced: false }`
  5. Background job setiap **5 detik**: ambil semua `synced: false` → retry POST → kalau sukses set `synced: true`
- [ ] Server: jika `client_uuid` sudah ada di DB → return 200 OK (idempotent, tidak buat record duplikat)
- [ ] Kiosk: tampilkan indikator "⚠️ Offline — tap tersimpan lokal" saat API tidak terjangkau

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/11-Decisions|ADR-006]]

---

### T015 — Activity Log Middleware
- [ ] Buat `ActivityLogInterceptor` NestJS (global interceptor, hanya untuk request yang punya `req.user`)
- [ ] Aksi yang wajib dilog (minimal Fase 1):
  - Semua POST/PATCH/DELETE di modul: Cards, Students (lock/unlock), Permits, SchoolHolidays, Users
  - `action` string format: `{resource}.{verb}` misal `card.create`, `student.lock`, `permit.create`
- [ ] Simpan `snapshot_before` dan `snapshot_after` sebagai JSON
- [ ] Pastikan tidak ada endpoint DELETE/UPDATE untuk `activity_log`
- [ ] `GET /activity-log` (akses: `super_admin`) untuk audit dari admin dashboard

**Ref:** [[Projek/AbsenSI/11-Decisions|ADR-020]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]]

---

### T016 — End-of-Day Job: Deteksi Tidak Tap Pulang
- [ ] Setup BullMQ scheduled job: jalan setiap hari pukul **18:00** (atau jam setelah jam pulang sekolah — sesuaikan dengan jadwal)
- [ ] Job logic:
  - Ambil semua `attendance_records` hari ini dengan `waktu_pulang = null` AND `student_id IS NOT NULL`
  - Tandai record-record ini sebagai "perlu klarifikasi" — bisa dengan flag kolom `needs_clarification: boolean` atau cukup query dinamis di dashboard piket (tidak perlu kolom baru kalau rekap dari query saja sudah cukup)
- [ ] Data ini akan dikonsumsi Dashboard Piket di T025 (antrian klarifikasi)

**Ref:** [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]], [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]]

---

## 📡 Blok 5 — Realtime & Dashboard
> Dependency: T012 (tap API harus dispatch event)

### T017 — Socket.IO Setup
- [ ] Install `@nestjs/websockets`, `socket.io` di `apps/api`
- [ ] `AttendanceGateway`:
  - Channel `attendance:today` — kirim payload agregat (jumlah hadir/terlambat/belum hadir) setiap `attendance.recorded` event dari BullMQ
  - Channel `attendance:kampus:{id}` — kirim update per-siswa (untuk Dashboard Piket T023)
- [ ] `apps/web` + `apps/kiosk`: setup Socket.IO client (`socket.io-client`)
- [ ] Auth WebSocket: kirim JWT di handshake header (bukan query param — hindari JWT di URL log)

**Ref:** [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack]], [[Projek/AbsenSI/11-Decisions|ADR-006]]

---

### T018 — Dashboard TV (`/tv`)
- [ ] Route `/tv` di `apps/web` dengan layout tersendiri (fullscreen, no navbar)
- [ ] Initial load: `GET /attendance/today` → tampilkan agregat
- [ ] Socket.IO subscribe ke `attendance:today` → update angka realtime tanpa reload
- [ ] Tampilan: angka besar (Hadir / Terlambat / Belum Hadir), jam real-time, tanggal
- [ ] Auth: akun `kepsek`, refresh token 30 hari sliding renewal
- [ ] Proteksi route: redirect ke login kalau token expired

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-tv|dashboard-tv.md]]

---

### T019 — Admin Dashboard: Rekap Kehadiran Siswa
- [ ] API `GET /attendance/report` dengan query params: `from`, `to`, `kelas_id?`, `jurusan_id?`
  - Query logic: untuk setiap siswa dalam filter → hitung count hadir, terlambat, izin, sakit, alfa
  - **Alfa** = hari wajib (Senin–Jumat, bukan libur di `school_holidays`, dalam `academic_years` aktif) tanpa record di `attendance_records` DAN tanpa `permits`
  - Index yang dipakai: `(tanggal, student_id)` di `attendance_records`, `(tanggal, student_id)` di `permits`
- [ ] Admin UI:
  - Halaman Rekap: filter bar (date range picker, dropdown kelas, dropdown jurusan) + tombol Tampilkan
  - Tabel hasil: Nama | Kelas | Jurusan | Hadir | Terlambat | Izin | Sakit | Alfa
  - Loading state saat query berjalan

**Ref:** [[Projek/AbsenSI/06-Features/rekap-kehadiran|rekap-kehadiran.md]], [[Projek/AbsenSI/06-Features/kalender-pendidikan|kalender-pendidikan.md]]

---

## 👤 Blok 6 — Akun Guru
> Dependency: T003 (auth), T005 (teachers harus ada)

### T020 — Users Module: CRUD Akun
- [ ] API endpoints (akses: `super_admin`):
  - `GET /users` — daftar semua akun
  - `POST /users` — buat akun baru (role, username, password, link ke teacher_id atau kampus_id)
  - `PATCH /users/:id` — edit akun (nonaktifkan, ganti role)
  - `POST /users/:id/reset-password` — generate password baru, return ke admin untuk disampaikan manual
- [ ] Admin UI: halaman Manajemen Akun — tabel + form create/edit + tombol reset password

**Ref:** [[Projek/AbsenSI/03-User-Roles|03-User-Roles]], [[Projek/AbsenSI/11-Decisions|ADR-008]]

---

### T021 — Guru Portal: Riwayat Kehadiran Diri Sendiri
- [ ] API `GET /attendance/my-history` (akses: role `guru`) — ambil `attendance_records` berdasarkan `teacher_id` dari JWT payload, dengan filter tanggal opsional
- [ ] Guru UI (route `/riwayat` di `apps/web`):
  - Setelah login sebagai guru: tampilkan tabel riwayat kehadiran (tanggal, waktu masuk, waktu pulang, status)
  - Filter tanggal (date range picker)
  - Read-only — tidak ada tombol edit apa pun

**Ref:** [[Projek/AbsenSI/06-Features/akun-guru|akun-guru.md]]

---

## 🏫 Blok 7 — Dashboard Piket (Fase 1b)
> Dependency: semua Blok 0–6 harus stabil. Mulai setelah inti Fase 1 berjalan dan diuji.

### T022 — Permits Module: API
- [ ] API endpoints (akses: `guru_piket` — ditegakkan di `RolesGuard`):
  - `POST /permits` — buat permit baru (`tidak_masuk` atau `keluar`)
    - Saat create: otomatis update `attendance_records` hari itu sesuai jenis
    - Generate `kode_verifikasi` unik (nanoid 8 char uppercase) untuk `jenis=keluar`
  - `PATCH /permits/:id/confirm-kembali` — tandai `status_kembali: sudah`
  - `PATCH /permits/:id/set-pulang` — tandai `status_kembali: pulang`, update `attendance_records`
  - `GET /permits` — daftar permits (filter: tanggal, kampus_id dari token piket)
- [ ] `POST /attendance/manual-pulang` (akses: `guru_piket`) — input pulang manual tanpa tap (fallback Sub-alur B)
- [ ] `POST /attendance/confirm-izin-pulang/:record_id` (akses: `guru_piket`) — ubah `pulang_via` tap → `tap_izin_pulang`
- [ ] Semua aksi piket dicatat ke `activity_log`

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], [[Projek/AbsenSI/11-Decisions|ADR-016, ADR-019]]

---

### T023 — Dashboard Piket: Halaman Utama Realtime
- [ ] Route `/piket` di `apps/web` (scope per `kampus_id` dari JWT piket)
- [ ] Tabel semua siswa kampus ini, dengan kolom: nama | kelas | status tap hari ini | jam masuk | jam pulang | action
- [ ] Subscribe Socket.IO `attendance:kampus:{id}` → update baris realtime saat ada tap baru
- [ ] Tombol **[Izin]** dan **[Sakit]** di setiap baris → modal kecil (isi keterangan → submit → buat `permits(tidak_masuk)`)
- [ ] Badge visual untuk siswa yang: sudah hadir / belum hadir / sudah izin/sakit / terkunci

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]]

---

### T024 — Dashboard Piket: Perizinan Keluar + Print
- [ ] Menu terpisah `/piket/izin-keluar` di sidebar piket
- [ ] Form Sub-alur A (izin keluar sementara):
  - Cari siswa (autocomplete), alasan kategori, keterangan, jam keluar, jam kembali (opsional)
  - Submit → POST `/permits` → generate `kode_verifikasi`
  - Sistem konstruksi URL ke `http://10.10.10.100:8800/print.php?petugas=...&kode=...` lalu `window.open()` di tab baru
- [ ] Form Sub-alur B (konfirmasi izin pulang setelah tap):
  - Cari siswa yang sudah tap pulang hari ini → tombol "Tandai Izin Pulang" → `PATCH /attendance/confirm-izin-pulang/:id`
- [ ] **Sebelum task ini: edit `C:\ProjekSMK\print.php`** — tambah blok tampilkan `$kode` di template HTML surat

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], [[Projek/AbsenSI/11-Decisions|ADR-018]]

---

### T025 — Dashboard Piket: Monitoring & Antrian Klarifikasi
- [ ] Section "Belum Kembali" di `/piket`:
  - Query: `permits` jenis `keluar`, `status_kembali: belum`, `jam_kembali_diharapkan < now()`
  - Tombol per baris: **[Sudah Kembali]** → `confirm-kembali` | **[Dianggap Pulang]** → `set-pulang`
- [ ] Section "Tidak Tap Pulang Kemarin":
  - Query: `attendance_records` tanggal kemarin, `waktu_pulang = null`, siswa kampus ini
  - Tombol per baris: **[Konfirmasi Pulang]** (input jam) | **[Tandai Izin Keluar Tidak Kembali]** (buat permit retroaktif)

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], T016

---

### T026 — Dashboard Piket: Lock/Unlock Siswa
- [ ] API (akses: `guru_piket`):
  - `POST /students/:id/lock` — isi `locked_reason`, set `locked_at`, `locked_by`
  - `POST /students/:id/unlock` — isi `unlock_note`, set `unlocked_at`, `unlocked_by`
- [ ] Section "Perlu Ditinjau" di `/piket`:
  - Siswa dengan `permits(keluar)` lewat jam kembali + `status_kembali: belum` dari hari-hari sebelumnya (bukan hanya kemarin)
  - Tombol **[Kunci Siswa]** → form isi alasan → POST lock
- [ ] Section "Siswa Terkunci":
  - Daftar siswa kampus ini yang `locked_at IS NOT NULL`
  - Tombol **[Buka Kunci]** → form isi catatan → POST unlock
- [ ] Di T012 (tap logic): tambahkan cek `students.locked_at` — kalau ada → `rejected_locked` di tap_events, response untuk kiosk tampilkan "Hubungi Guru Piket"

**Ref:** [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]], [[Projek/AbsenSI/11-Decisions|ADR-017]]

---

## 📌 Catatan Penting

1. **Sebelum mulai coding setiap task:** baca spec yang direferensikan di bagian `Ref`. Jangan asumsikan — spec sudah lengkap dan ada alasan untuk setiap keputusan.
2. **ADR adalah hukum:** kalau ada konflik antara instinct coding dan ADR, ADR menang. Kalau ADR dirasa perlu diubah, buka dulu diskusi dan buat ADR baru.
3. **Setelah Blok 4 selesai:** uji end-to-end dengan hardware reader RFID fisik sebelum lanjut ke Blok 5. Data tap palsu (input manual dari keyboard biasa) tidak cukup untuk validasi debounce dan timing.
4. **print.php:** edit file di `C:\ProjekSMK\print.php` harus dilakukan **sebelum** T024 di-deploy — ini bukan task coding NestJS/Next.js, tapi blocker untuk fitur cetak surat.
