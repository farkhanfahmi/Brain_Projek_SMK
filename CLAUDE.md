# CLAUDE.md — AbsenSI Project Context

> Baca file ini di awal setiap sesi sebelum mengerjakan apapun.
> Vault Obsidian ini adalah **sumber kebenaran desain** untuk proyek AbsenSI.
> Kode dieksekusi di repo terpisah; vault ini adalah referensi spec, ADR, dan status task.

---

## Quick Start per Sesi

1. Baca `Projek/AbsenSI/STATUS.md` — satu-satunya tempat untuk tahu task apa yang aktif/belum dikerjakan
2. Baca spec yang direferensikan di task tersebut (kalau ada)
3. Cek `Projek/AbsenSI/11-Decisions.md` kalau ada keputusan arsitektur yang relevan
4. Baru mulai coding

**Jangan skip langkah 2 dan 3.** Setiap keputusan teknis punya alasan yang sudah didokumentasikan.

---

## Orientasi Cepat

| Apa yang kamu cari | Di mana |
|---|---|
| **Status & task aktif (cek ini duluan)** | `Projek/AbsenSI/STATUS.md` |
| Gambaran umum proyek | `Projek/AbsenSI/01-Overview.md` |
| Tech stack & arsitektur | `Projek/AbsenSI/02-Tech-Stack.md` |
| Keputusan arsitektur (ADR) | `Projek/AbsenSI/11-Decisions.md` |
| Skema database lengkap | `Projek/AbsenSI/04-Database-Schema.md` |
| Spec fitur spesifik | `Projek/AbsenSI/06-Features/` |
| Role & permission user | `Projek/AbsenSI/03-User-Roles.md` |
| API endpoints | `Projek/AbsenSI/05-API-Endpoints.md` |
| Konvensi kode & git | `Projek/AbsenSI/09-Conventions.md` |
| Environment, topologi dev/production, backup | `Projek/AbsenSI/10-Environment.md` |
| Riwayat task lama (jarang perlu dibaca) | `Projek/AbsenSI/_archive/` |

---

## Tech Stack Ringkas

```
Backend  : NestJS + Prisma ORM + MySQL 8
Frontend : Next.js 14 (App Router)
UI       : shadcn/ui + Tailwind CSS (komponen di packages/ui)
Realtime : Socket.IO
Queue    : BullMQ + Redis
Auth     : JWT (access 15m, refresh 7d) + Redis blacklist
Kiosk    : Next.js (HID keyboard emulation, tidak perlu driver native)
Monorepo : Turborepo
```

**Struktur repo:**
```
apps/
  api/      ← NestJS — semua backend logic
  web/      ← Next.js — admin dashboard, dashboard TV (/tv), portal guru, dashboard piket (/piket)
  kiosk/    ← Next.js — kiosk gerbang (fullscreen tap UI)
packages/
  types/    ← shared TypeScript interfaces/types
  config/   ← shared tsconfig, tailwind config, eslint
  ui/       ← shadcn/ui components
```

**Dev/Production dipisah fisik total** (2 folder, 2 database Docker, port berbeda) sejak T105 — detail lengkap di `Projek/AbsenSI/10-Environment.md`.

---

## Auth Architecture

| Actor | Mekanisme |
|---|---|
| User (admin, guru, piket, kepsek, dst) | JWT Bearer — `POST /auth/login` → access + refresh token |
| Kiosk (unattended) | Static device token → `KioskGuard` |
| TV display | JWT refresh token 30 hari, sliding renewal |

**Guard hierarchy di NestJS:**
- `JwtAuthGuard` — validasi JWT + cek Redis blacklist
- `RolesGuard` — cek role dari JWT payload vs `@Roles()` decorator
- `KioskGuard` — validasi static device token (terpisah dari JWT flow)

---

## Database Rules

- Engine: **MySQL 8**, ORM: **Prisma**
- Relasi ke "siswa ATAU guru" selalu pakai **dual-FK nullable** (`student_id` + `teacher_id`, tepat 1 yang terisi) — jangan polymorphic
- `tap_events` dan `activity_log` adalah **insert-only** — tidak boleh ada endpoint UPDATE/DELETE untuk keduanya
- **Hari wajib** = Senin–Jumat, dalam range `academic_years` aktif, tidak masuk `school_holidays`
- **Sabtu** = opsional (hadir dicatat, tidak hadir BUKAN alfa)
- **Alfa** bukan kolom di DB — kondisi "tidak ada data" yang dihitung saat query rekap

---

## Scope & Boundaries Kritis

### Yang HANYA boleh dilakukan `guru_piket`
- Membuat `permits` (izin, sakit)
- Mengubah status kehadiran siswa
- Lock/unlock siswa

### Yang TIDAK boleh dilakukan `super_admin`
- Mengubah status kehadiran siswa (bukan domain admin)

### Yang SELALU dilog
- Semua tap RFID → `tap_events` (termasuk yang ditolak)
- Semua aksi user yang login → `activity_log`

---

## Hal yang Sering Bikin Salah

1. **Tap ke-3+** → tetap update `waktu_pulang` (bukan diabaikan)
2. **Debounce** → tap kartu yang sama < 30 detik = `rejected_duplicate` (dicatat di `tap_events`, tidak buat record)
3. **`scanned_at`** di `tap_events` → selalu **timestamp server**, bukan dari request body kiosk
4. **Status kehadiran** (izin/sakit/alfa) → hanya `guru_piket` yang bisa set; `super_admin` tidak punya akses
5. **`client_uuid`** → untuk idempotency offline sync; server tolak duplikat dengan return 200 OK (bukan 409)
6. **`tap_events`** dan **`activity_log`** → insert-only, tidak ada endpoint modifikasi

---

## Cetak Struk Izin

Route handler internal `apps/web` (bukan lagi server PHP eksternal) — detail di ADR-018 revisi, `Projek/AbsenSI/11-Decisions.md`.

---

## Catatan Penting

- **Bukan lagi rencana tim 3-developer.** Vault ini sempat dirancang untuk skenario 3 developer paralel (lihat `Projek/AbsenSI/_archive/_claudian/`) — eksekusi aktual sejak awal berjalan sebagai sesi tunggal Claude Code. Jangan ikuti dokumen itu sebagai workflow yang berlaku sekarang.
- **Tidak ada wikilink `[[...]]` di vault ini** — semua referensi antar file pakai path teks biasa. Lihat `Projek/AbsenSI/_archive/_REPAIR-LOG.md` untuk alasan (wikilink terbukti rapuh 2x).
- Workflow umum Claude Code (bukan spesifik AbsenSI) ada di `_WORKFLOW-CLAUDE-CODE.md` di root vault ini — baca itu untuk paham prinsip pengelolaan memory/vault yang berlaku lintas proyek.
