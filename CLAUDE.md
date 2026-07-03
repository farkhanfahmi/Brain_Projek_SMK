# CLAUDE.md — AbsenSI Project Context

> Baca file ini di awal setiap sesi sebelum mengerjakan apapun.
> Vault Obsidian ini adalah **sumber kebenaran desain** untuk proyek AbsenSI.
> Kode dieksekusi di repo terpisah; vault ini adalah referensi spec, ADR, dan task.

---

## 🗺️ Orientasi Cepat

| Apa yang kamu cari | Di mana |
|---|---|
| Gambaran umum proyek | `Projek/AbsenSI/01-Overview.md` |
| Tech stack & arsitektur | `Projek/AbsenSI/02-Tech-Stack.md` |
| Keputusan arsitektur (ADR) | `Projek/AbsenSI/11-Decisions.md` |
| Skema database lengkap | `Projek/AbsenSI/04-Database-Schema.md` |
| Task yang harus dikerjakan | `Projek/AbsenSI/TASKS-FASE-1.md` |
| Spec fitur spesifik | `Projek/AbsenSI/06-Features/` |
| Role & permission user | `Projek/AbsenSI/03-User-Roles.md` |
| API endpoints | `Projek/AbsenSI/05-API-Endpoints.md` |
| Konvensi kode & git | `Projek/AbsenSI/09-Conventions.md` |
| Environment & infra | `Projek/AbsenSI/10-Environment.md` |

---

## ⚡ Quick Start per Sesi

1. Buka `Projek/AbsenSI/TASKS-FASE-1.md` — lihat task mana yang sedang dikerjakan
2. Baca spec yang direferensikan di task tersebut (bagian `Ref:`)
3. Cek `Projek/AbsenSI/11-Decisions.md` kalau ada keputusan arsitektur yang relevan
4. Baru mulai coding

**Jangan skip langkah 2 dan 3.** Setiap keputusan teknis punya alasan yang sudah didokumentasikan.

---

## 🏗️ Tech Stack Ringkas

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

---

## 🔐 Auth Architecture

| Actor | Mekanisme |
|---|---|
| User (admin, guru, piket, kepsek) | JWT Bearer — `POST /auth/login` → access + refresh token |
| Kiosk (unattended) | Static device token di `.env` (`KIOSK_DEVICE_TOKEN`) → `KioskGuard` |
| TV display (kepsek) | JWT refresh token 30 hari, sliding renewal |

**Guard hierarchy di NestJS:**
- `JwtAuthGuard` — validasi JWT + cek Redis blacklist
- `RolesGuard` — cek role dari JWT payload vs `@Roles()` decorator
- `KioskGuard` — validasi static device token (terpisah dari JWT flow)

---

## 🗄️ Database Rules

- Engine: **MySQL 8**, ORM: **Prisma**
- Relasi ke "siswa ATAU guru" selalu pakai **dual-FK nullable** (`student_id` + `teacher_id`, tepat 1 yang terisi) — jangan polymorphic
- `tap_events` dan `activity_log` adalah **insert-only** — tidak boleh ada endpoint UPDATE/DELETE untuk keduanya
- `school_holidays` + `academic_years` digunakan untuk hitung hari wajib (alfa = tidak hadir di hari wajib)
- **Hari wajib** = Senin–Jumat (`DAYOFWEEK` 2–6), dalam range `academic_years` aktif, tidak masuk `school_holidays`
- **Sabtu** = opsional (hadir dicatat, tidak hadir BUKAN alfa)

---

## 🎯 Scope & Boundaries Kritis

### Yang HANYA boleh dilakukan `guru_piket`
- Membuat `permits` (status kehadiran: izin, sakit)
- Mengubah status kehadiran siswa
- Lock/unlock siswa

### Yang TIDAK boleh dilakukan `super_admin`
- Mengubah status kehadiran siswa (bukan domain admin)

### Yang SELALU dilog
- Semua tap RFID → `tap_events` (termasuk yang ditolak)
- Semua aksi user yang login → `activity_log`

---

## 📋 Spec Fitur Fase 1 (semua sudah Final ✅)

| Fitur | File Spec | Status |
|---|---|---|
| Absensi Gerbang | `06-Features/absensi-gerbang.md` | ✅ Final |
| Manajemen Kartu | `06-Features/manajemen-kartu.md` | ✅ Final |
| Import Data Master | `06-Features/import-data-master.md` | ✅ Final |
| Kalender Pendidikan | `06-Features/kalender-pendidikan.md` | ✅ Final |
| Rekap Kehadiran | `06-Features/rekap-kehadiran.md` | ✅ Final |
| Dashboard TV | `06-Features/dashboard-tv.md` | ✅ Final |
| Akun Guru | `06-Features/akun-guru.md` | ✅ Final |
| Dashboard Piket | `06-Features/dashboard-piket.md` | ✅ Final (Fase 1b) |

---

## ⚠️ Hal yang Sering Bikin Salah

1. **Tap ke-3+** → tetap update `waktu_pulang` (bukan diabaikan)
2. **Debounce** → tap kartu yang sama < 30 detik = `rejected_duplicate` (dicatat di `tap_events`, tidak buat record)
3. **`scanned_at`** di `tap_events` → selalu **timestamp server**, bukan dari request body kiosk
4. **Status kehadiran** (izin/sakit/alfa) → hanya `guru_piket` yang bisa set; `super_admin` tidak punya akses
5. **Alfa** → bukan kolom di DB, ini kondisi "tidak ada data" yang dihitung saat query rekap
6. **`client_uuid`** → untuk idempotency offline sync; server tolak duplikat dengan return 200 OK (bukan 409)
7. **Sabtu** → hadir dicatat normal di `attendance_records`, tapi tidak masuk perhitungan alfa
8. **`tap_events`** dan **`activity_log`** → insert-only, tidak ada endpoint modifikasi

---

## 🖨️ Integrasi print.php

- Server: `http://10.10.10.100:8800/print.php`
- Script fisik: `C:\ProjekSMK\print.php`
- Parameter: `petugas`, `tgl`, `nama`, `kls`, `alasan`, `ket`, `jamkembali`, `kode` (parameter baru)
- Flow: sistem konstruksi URL → `window.open(url)` di tab baru → petugas klik Print manual di browser
- **Edit `print.php`** untuk tambah `kode` harus dilakukan sebelum T024 di-deploy

---

## 🔗 Dokumen Penting Lainnya

- Semua ADR final: `Projek/AbsenSI/11-Decisions.md` (ADR-001 s/d ADR-020)
- Backlog & roadmap: `Projek/AbsenSI/13-Backlog.md`
- Debug log: `Projek/AbsenSI/14-Debug-Log.md` (isi saat ada masalah saat development)
- Environment: `Projek/AbsenSI/10-Environment.md`
