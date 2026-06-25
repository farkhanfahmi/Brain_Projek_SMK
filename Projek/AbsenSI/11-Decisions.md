---
tags: [absensi, adr, decisions]
updated: 2026-06-25
---

# 11 — Decisions (ADR)

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> Setiap keputusan arsitektur permanen dicatat di sini. Format: Konteks → Keputusan → Alasan → Konsekuensi. **Jangan ubah keputusan di sini tanpa diskusi ulang dan catat ADR baru yang men-supersede.**

---

## ADR-001: Tidak Import Data Spreadsheet Lama
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Sekolah saat ini pakai Google Form + spreadsheet untuk absensi. Ada histori data di sana.
**Keputusan:** AbsenSI mulai dari data baru (clean slate). Tidak ada migrasi data spreadsheet di fase 1.
**Alasan:** Mapping NISN/NIP ke UID kartu RFID belum ada di data lama (data lama tidak berbasis kartu). Effort import tidak proporsional dengan manfaat di fase awal.
**Konsekuensi:** Kalau nanti dibutuhkan historical data, harus dikerjakan sebagai task terpisah di fase development selanjutnya — bukan blocker fase 1.

---

## ADR-002: Stack TypeScript Full (NestJS + Next.js), Bukan Laravel
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Tim biasa pakai Laravel/MySQL di proyek lain (DasiPelajar, SIMA-Sarpras). AbsenSI direncanakan jadi proyek perintis ekosistem aplikasi sekolah yang lebih besar. Tim ingin belajar ekosistem JS, dibantu Claude Code untuk eksekusi.
**Keputusan:** Backend NestJS + Prisma + PostgreSQL, frontend Next.js, dalam satu monorepo Turborepo.
**Alasan:**
- Shared TypeScript types lintas layer (backend ↔ frontend ↔ client) — manfaat nyata untuk visi ekosistem banyak aplikasi terhubung
- NestJS dipilih (bukan Express polos) karena strukturnya paling dekat dengan konsep Laravel (module, DI, decorator) — transisi developer Laravel paling mulus
- PostgreSQL dipilih atas MySQL karena kebutuhan query rekap analitik fleksibel (filter multi-dimensi) di masa depan
**Konsekuensi:**
- **Risiko diakui secara sadar:** 2 dari 3 developer belum familiar paradigma backend Node/Nest. Mitigasi: pembagian modul disesuaikan kekuatan masing-masing (lihat [[30.Projects/AbsenSI/_claudian/team|team.md]]), modul Core/paling kritis dipegang developer paling fullstack.
- Tim akan memelihara 2 stack berbeda (Laravel di proyek lama, TS di AbsenSI dst). Ini diterima sebagai trade-off sadar demi belajar.
- Ini bukan keputusan "JS lebih baik untuk realtime/API" — itu klaim yang ditolak (realtime/API bisa dicapai stack apa pun). Alasan sebenarnya adalah shared-types dan kesiapan tim belajar ekosistem baru untuk proyek-proyek berikutnya.

---

## ADR-003: Modular Monolith, Bukan Microservices
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Visi jangka panjang adalah ekosistem banyak aplikasi sekolah dengan data terpusat. Ada godaan untuk langsung desain microservices.
**Keputusan:** AbsenSI dibangun sebagai **modular monolith** — satu aplikasi NestJS dengan batas modul tegas (Core, Attendance, Card, Schedule, Notification-stub), bukan service terpisah-terpisah.
**Alasan:** Skala 1 sekolah (2.500 siswa, 100+ guru) tidak butuh kompleksitas distributed system. Microservices di skala ini menambah overhead operasional (service discovery, network latency, deployment complexity) tanpa manfaat nyata. Modul Core dirancang punya batas jelas agar **bisa** dipecah jadi service terpisah nanti — saat aplikasi sekolah ke-2/ke-3 benar-benar butuh akses data siswa/guru terpusat.
**Konsekuensi:** Tim harus disiplin menjaga batas modul (Attendance tidak boleh query langsung tabel Core di luar service layer yang disediakan) agar pemecahan ke microservice nanti tidak butuh rewrite besar.

---

## ADR-004: Reader RFID = USB-HID Keyboard Emulation
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Ada dua tipe USB RFID reader: HID keyboard-emulation vs SDK/serial proprietary. Tipe ini menentukan seluruh desain client mini-PC.
**Keputusan:** Hardware yang dipakai dikonfirmasi tipe **HID keyboard emulation** (tap kartu = device "mengetik" UID + Enter).
**Alasan:** Sudah dicek langsung oleh tim di hardware fisik.
**Konsekuensi:** Client mini-PC **tidak perlu** library serial/driver native (`serialport`, `node-hid`, SDK vendor). Cukup halaman web kiosk (`apps/kiosk`) dengan input field auto-focus menangkap keystroke. Ini menyederhanakan scope `apps/kiosk` secara signifikan — jadi web app biasa, bukan software hardware-adjacent.

---

## ADR-005: Fase 1 Hanya Gerbang — Terlambat Dihitung dari Jam Masuk Sekolah, Bukan Per Sesi Mengajar
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Definisi final "guru terlambat" adalah tap kelas melebihi jadwal mengajar (fase 2). Tapi fase 1 hanya punya reader di gerbang, belum ada reader kelas.
**Keputusan:** Fase 1: status terlambat (siswa & guru) dihitung dari **jam tap gerbang vs jam masuk sekolah** (atau jam mengajar pertama guru hari itu, untuk kasus guru). Logika "terlambat per sesi mengajar" baru aktif di fase 2 saat reader kelas terpasang.
**Alasan:** Data yang tersedia di fase 1 cuma tap gerbang — tidak cukup untuk menyimpulkan kehadiran per sesi mengajar spesifik.
**Konsekuensi:** Modul **Schedule** harus sudah punya struktur jadwal mengajar guru dari fase 1 (untuk hitung "jam mengajar pertama"), meskipun reader kelas belum ada. Desain skema DB harus mendukung kedua fase tanpa rebuild — lihat [[30.Projects/AbsenSI/04-Database-Schema|04-Database-Schema]].

---

## ADR-006: Event-Driven untuk Notifikasi Masa Depan
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Notifikasi WA ke orang tua direncanakan fase 3, tapi jalur arsitekturnya harus disiapkan dari fase 1 agar tidak perlu refactor core logic nanti.
**Keputusan:** Setiap tap kartu yang berhasil di-record memicu event (`attendance.recorded`) yang di-dispatch ke BullMQ queue. Modul Notification (fase 1: stub/cuma log) subscribe ke event ini.
**Alasan:** Decoupling — controller attendance tidak perlu tahu/peduli ada listener WA atau tidak. Menambah fitur notifikasi nanti = tambah listener baru, bukan ubah modul Attendance.
**Konsekuensi:** Butuh Redis + BullMQ terpasang dari fase 1 meskipun belum ada consumer nyata selain logging.

---

## ADR-008: Role Generik (`super_admin`, `card_admin`, `guru`), Bukan Diikat ke Identitas Developer
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Saat diskusi role, "Admin Pusat" awalnya didefinisikan sebagai "3 orang developer" — ini mengikat role sistem ke identitas spesifik orang, bukan ke fungsi.
**Keputusan:** Role disimpan generik di database: `super_admin` (full akses semua fitur), `card_admin` (CRUD kartu saja), `guru` (read-only riwayat sendiri). Saat ini 3 akun `super_admin` kebetulan dipegang oleh 3 developer, tapi ini bukan aturan permanen yang mengikat role ke orang tersebut.
**Alasan:** Kalau role diikat ke identitas, sistem rapuh terhadap perubahan personel (developer resign/pindah) dan tidak fleksibel kalau nanti staff non-developer (Kepala TU, dst.) butuh akses setingkat itu.
**Konsekuensi:** Skema tabel `users`/`accounts` butuh kolom `role` (enum atau FK ke tabel roles), permission dicek di level API (backend), **bukan** cuma disembunyikan di UI frontend — supaya Admin Pengelola Kartu tidak bisa akses endpoint di luar modul kartu meski tahu URL-nya.

---

## ADR-009: Import Data Master Naik ke Scope Fase 1, Desain Hybrid CSV + Tap-to-Assign
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Rollout awal butuh daftarkan 2.500 siswa + 100 guru + kartu RFID-nya. Input manual satu-satu tidak praktis. UID kartu untuk rollout awal sudah diketahui (dari vendor), tapi ada siswa baru yang kartunya belum terekam.
**Keputusan:** Fitur import dinaikkan dari backlog fase 3 ke scope fase 1, dengan urutan wajib: (1) Kelas & Jurusan diinput manual dulu sebagai master data, (2) import CSV data siswa & guru, (3) pemetaan kartu RFID dengan 2 mode — Mode A bulk CSV (UID sudah diketahui) dan Mode B tap-to-assign (untuk kartu yang belum terekam).
**Alasan:** CSV murni tidak cukup karena tidak semua UID diketahui di muka. Kelas/Jurusan tidak di-auto-create dari CSV untuk mencegah duplikat akibat inkonsistensi penulisan di file sumber.
**Konsekuensi:** Modul Card & modul import butuh UI tambahan untuk flow tap-to-assign (real-time capture UID), bukan cuma form upload file. Estimasi effort fase 1 bertambah, tapi krusial untuk rollout awal yang realistis.

---

## ADR-007: Monorepo Turborepo, Bukan Polyrepo
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** 3 developer perlu kelola backend, dashboard, dan kiosk app.
**Keputusan:** Satu repo GitHub, dikelola dengan Turborepo (`apps/api`, `apps/web`, `apps/kiosk`, `packages/types`, `packages/config`).
**Alasan:** Tim kecil (3 orang), perlu shared types package yang mudah diakses semua app tanpa publish package terpisah. Monorepo minim overhead koordinasi untuk skala ini.
**Konsekuensi:** Semua developer perlu nyaman kerja di satu repo besar — branch & PR harus disiplin scoped ke modul masing-masing (lihat [[30.Projects/AbsenSI/09-Conventions|09-Conventions]]).
