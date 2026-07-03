# T002 — Prisma Schema Lengkap

## Depends on
T001 (monorepo harus sudah ada)

## Objective
Definisikan semua tabel database Fase 1 dalam satu file `schema.prisma` yang valid, jalankan migrasi awal, dan buat seed data minimal untuk development.

## Context
- **App:** `apps/api`
- **DB:** MySQL 8
- **ORM:** Prisma
- **ADR:** ADR-010 (dual-FK nullable), ADR-011 (MySQL), ADR-016 (permits), ADR-020 (tap_events + activity_log)
- **Ref wajib dibaca:** `Projek/AbsenSI/04-Database-Schema.md` — baca SELURUH file sebelum mulai

## Spec Detail

### Install:
```
pnpm add prisma @prisma/client --filter api
npx prisma init --datasource-provider mysql
```

### Tabel yang HARUS ada di schema (lengkap):

**Core:**
- `kampus`: id, nama
- `jurusan`: id, nama
- `kelas`: id, nama, kampus_id (FK kampus), jurusan_id (FK jurusan)
- `students`: id, nisn (unique), nama, kelas_id (FK), status (aktif/nonaktif), tanggal_lahir (optional), locked_at, locked_reason, locked_by (FK users nullable), unlocked_at, unlocked_by (FK users nullable), unlock_note, created_at
- `teachers`: id, nip (unique), nama, status (aktif/nonaktif), created_at

**Auth:**
- `users`: id, username (unique), password_hash, role (enum: super_admin/card_admin/guru/kepsek/guru_piket), teacher_id (FK teachers nullable), kampus_id (FK kampus nullable — hanya guru_piket), status (aktif/nonaktif), created_at

**Kartu:**
- `cards`: id, uid (unique), student_id (FK nullable), teacher_id (FK nullable), status (active/inactive), issued_at, revoked_at (nullable)

**Jadwal:**
- `schedules`: id, type (jam_sekolah/jam_mengajar/jadwal_khusus), teacher_id (FK nullable), kelas_id (FK nullable), hari, jam_mulai, jam_selesai, threshold_terlambat_menit, tanggal_berlaku_mulai, tanggal_berlaku_selesai (nullable)

**Kalender:**
- `academic_years`: id, nama, tanggal_mulai, tanggal_selesai, is_active (Boolean default false), created_by (FK users), created_at
- `school_holidays`: id, academic_year_id (FK nullable), tanggal_mulai, tanggal_selesai, jenis (enum: libur_nasional/libur_semester/libur_sekolah/libur_mendadak), keterangan, created_by (FK users), created_at, updated_by (FK users nullable), updated_at (nullable)

**Absensi:**
- `attendance_sessions`: id, location_type (gerbang/kelas), kelas_id (FK nullable), mapel_id (nullable — fase 2)
- `attendance_records`: id, student_id (FK nullable), teacher_id (FK nullable), session_id (FK nullable), tanggal, waktu_masuk, waktu_pulang (nullable), status (hadir/terlambat/tidak_hadir), pulang_via (enum nullable: tap/piket_izin/tap_izin_pulang), client_uuid (unique), kiosk_id, created_at

**Log (insert-only):**
- `tap_events`: id, uid, card_id (FK nullable), kiosk_id, scanned_at, result (enum: accepted/rejected_inactive/rejected_locked/rejected_unknown/rejected_duplicate), attendance_record_id (FK nullable)
- `activity_log`: id, actor_id (FK users), action, target_type, target_id, snapshot_before (Json nullable), snapshot_after (Json nullable), ip_address (nullable), created_at

**Permits:**
- `permits`: id, student_id (FK), jenis (enum: tidak_masuk/keluar), alasan_kategori (enum: sakit/izin), alasan_detail, tanggal, jam_keluar (nullable), jam_kembali_diharapkan (nullable), status_kembali (enum: belum/sudah/pulang default belum), kembali_dikonfirmasi_at (nullable), kembali_dikonfirmasi_by (FK users nullable), approved_by (FK users), kode_verifikasi (unique nullable), surat_printed_at (nullable), created_at

### Seed data (`prisma/seed.ts`):
```typescript
// Minimal untuk development:
// 2 kampus: "Kampus 1", "Kampus 2"
// 3 jurusan: "RPL", "TKJ", "MM"  
// 4 kelas: "X-RPL-1" (kampus 1), "XI-TKJ-1" (kampus 1), "X-MM-1" (kampus 2), "XI-RPL-1" (kampus 2)
// 1 academic_year aktif: "2025/2026", mulai 1 Juli 2025, selesai 30 Juni 2026
// 1 user super_admin: username "admin", password "admin123" (hash bcrypt)
// 1 user guru_piket kampus 1: username "piket1", password "piket123"
```

## JANGAN
- ❌ JANGAN buat 2 tabel terpisah untuk `tidak_masuk` dan `keluar` — keduanya dalam 1 tabel `permits` dengan kolom `jenis`
- ❌ JANGAN gunakan polymorphic (`owner_type` + `owner_id`) — wajib dual-FK nullable sesuai ADR-010
- ❌ JANGAN buat kolom `akan_kembali` di `permits` — sudah dihapus dari desain (diganti logika `status_kembali`)
- ❌ JANGAN tambahkan tabel yang belum ada di spec (misal `notifications`, `mapel`) — itu Fase 2/3
- ❌ JANGAN gunakan `@updatedAt` di `tap_events` dan `activity_log` — tabel ini insert-only, tidak ada update
- ❌ JANGAN buat relasi cascade delete yang bisa hapus `tap_events` atau `activity_log` secara tidak sengaja
- ❌ JANGAN skip `client_uuid` unique constraint di `attendance_records` — ini krusial untuk idempotency offline

## Files
- **Buat:** `apps/api/prisma/schema.prisma`
- **Buat:** `apps/api/prisma/seed.ts`
- **Buat:** `apps/api/prisma/migrations/` (auto-generated dari `prisma migrate dev`)
- **Modifikasi:** `apps/api/package.json` — tambah script `"db:seed": "ts-node prisma/seed.ts"`

## Acceptance Criteria
- [ ] `prisma migrate dev --name init` berjalan tanpa error
- [ ] `prisma db seed` berjalan, seed data masuk ke DB
- [ ] `prisma studio` bisa dibuka, semua tabel terlihat dengan kolom yang benar
- [ ] Tidak ada tabel yang hilang dari daftar di atas
- [ ] Enum `pulang_via` punya 3 nilai: `tap`, `piket_izin`, `tap_izin_pulang`
- [ ] Enum `status_kembali` punya 3 nilai: `belum`, `sudah`, `pulang`
- [ ] `cards` punya constraint: tepat 1 dari `student_id`/`teacher_id` terisi (implementasi via check constraint atau validasi di service — Prisma tidak support check constraint native, catat ini sebagai catatan di kode)

## Handoff ke T003
T003 akan setup auth module. Pastikan tabel `users` sudah ada dengan kolom `role`, `password_hash`, `status`. Seed user `admin` dan `piket1` harus sudah ada untuk testing auth.
