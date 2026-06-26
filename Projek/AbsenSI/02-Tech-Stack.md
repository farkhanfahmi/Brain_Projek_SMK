---
tags: [absensi, tech-stack, architecture]
updated: 2026-06-25
---

# 02 — Tech Stack & Arsitektur

← [[Projek/AbsenSI/00-INDEX|Index]]

> Keputusan stack ini **berbeda** dari proyek lain di vault ini (DasiPelajar dkk pakai Laravel). Ini keputusan sadar untuk belajar ekosistem TypeScript sebagai blueprint aplikasi sekolah berikutnya. Risiko & alasan didiskusikan penuh — lihat [[Projek/AbsenSI/11-Decisions|ADR-002]].
>
> **Update 2026-06-26:** Database engine diganti dari PostgreSQL ke **MySQL** — lihat [[Projek/AbsenSI/11-Decisions|ADR-011]] yang men-supersede sebagian ADR-002. Bagian lain ADR-002 (NestJS, Prisma, arsitektur modular monolith) tetap berlaku tanpa berubah.

---

## 🖥️ Backend

| Teknologi | Kegunaan | Alasan |
|---|---|---|
| **Node.js + TypeScript** | Runtime | Type-safety end-to-end, shared types antar layer |
| **NestJS** | Framework backend | Struktur modular (module, DI, decorator) — paling mendekati Laravel secara konsep, transisi developer Laravel paling mulus dibanding Express polos |
| **Prisma** | ORM | Type-safe query, schema-as-code, migration jelas |
| **MySQL** | Database utama | Tim sudah familiar dari proyek sebelumnya (DasiPelajar, SIMA-Sarpras) — mengurangi 1 kurva belajar baru di tengah belajar NestJS/TypeScript. Volume data AbsenSI (±500rb baris/tahun) tidak butuh keunggulan analitik PostgreSQL; index komposit yang tepat sudah cukup. Lihat ADR-011. |
| **Redis** | Cache + Queue broker | Dipakai BullMQ untuk job queue, juga cache rekap TV |
| **BullMQ** | Job queue | Event-driven: `attendance.recorded` di-dispatch ke queue, listener notifikasi (fase 3) tinggal subscribe tanpa ubah core |

## 🎨 Frontend

| Teknologi | Kegunaan |
|---|---|
| **Next.js (React + TypeScript)** | Dashboard admin + halaman kiosk mini-PC |
| **TailwindCSS** | Styling |
| **Socket.IO client** | Konsumsi event realtime dari backend (dashboard TV, live monitor) |

## 🗺️ Arsitektur — Modular Monolith (BUKAN microservices)

> **Keputusan eksplisit:** Tidak microservices dari awal. Skala 1 sekolah tidak butuh kompleksitas distributed system (service discovery, distributed transaction, network latency antar service). Modular monolith memberi batas modul yang jelas SEKARANG, dan bisa dipecah jadi service terpisah NANTI kalau memang dibutuhkan (misal saat aplikasi sekolah ke-2/ke-3 butuh akses data siswa/guru). Lihat ADR-003.

```
┌──────────────────────────────────────────────────────────┐
│                    MONOREPO (Turborepo)                   │
│                                                            │
│  apps/api/        ← NestJS — modul: Core, Attendance,    │
│                      Card, Schedule, Notification(stub)   │
│  apps/web/         ← Next.js — Dashboard admin            │
│  apps/kiosk/        ← Next.js — Halaman tap gerbang +      │
│                      TV display (kiosk mode)               │
│                                                            │
│  packages/types/   ← Shared TS types/interface            │
│  packages/config/   ← Shared eslint/tsconfig               │
└──────────────────────────────────────────────────────────┘
                          │
                ┌─────────┴──────────┐
                │   PostgreSQL +     │
                │   Redis (queue/    │
                │   cache)           │
                └────────────────────┘
```

### Batas Modul di dalam `apps/api`

| Modul | Tanggung Jawab | Catatan |
|---|---|---|
| **Core** | Master data: siswa, guru, kelas, jurusan, jadwal, auth | Calon kandidat dipecah jadi service terpisah pertama kali ekosistem berkembang |
| **Attendance** | Logic tap, status hadir/telat, sesi gerbang/kelas | Konsumsi data Core via internal call (bukan HTTP, masih 1 proses) |
| **Card** | CRUD kartu RFID, mapping UID ↔ orang, riwayat ganti kartu | |
| **Schedule** | Jadwal masuk sekolah, jadwal mengajar, jadwal khusus (ujian) | Sumber kebenaran untuk hitung "terlambat" |
| **Notification** (stub fase 1) | Listener BullMQ untuk event attendance | Fase 1: cuma log, belum kirim WA. Fase 3: isi listener WA di sini tanpa ubah modul lain |

## 🔌 Client Mini-PC (Gerbang)

- **Reader: USB-HID keyboard emulation** (dikonfirmasi) → tap kartu = reader "mengetik" UID + Enter
- **Tidak perlu library serial/driver khusus** — implikasi besar: client di mini-PC cukup berupa **halaman web kiosk** (`apps/kiosk`) yang auto-focus ke input field tersembunyi, menangkap keystroke, lalu POST ke API
- **Offline-first:** halaman kiosk pakai **IndexedDB/localStorage buffer** — kalau API tidak terjangkau, simpan tap lokal dulu dengan `client_uuid` unik, retry sync berkala. Idempoten di sisi server (cek `client_uuid` sebelum insert) agar replay tidak duplikat.

## 📡 Realtime

- **Socket.IO** (server di NestJS, client di Next.js)
- Dashboard TV subscribe ke channel `attendance:today` — push instan setiap ada tap baru
- **Bukan polling** (beda dari pola DasiPelajar yang polling Alpine.js 5 detik) — TV nyala 8 jam/hari, polling terus-terusan boros request

## 📦 Repo & Tooling

| Item | Pilihan |
|---|---|
| Monorepo tool | **Turborepo** |
| Package manager | pnpm (disarankan — lebih efisien untuk monorepo dibanding npm/yarn) |
| Linting | ESLint + Prettier, config shared via `packages/config` |
| API docs | OpenAPI/Swagger auto-generate dari decorator NestJS — kontrak resmi API untuk konsumsi app lain di masa depan |
| CI | GitHub Actions (setup saat repo GitHub dibuat) |

## ⚠️ Catatan Skala (jangan over-engineer)

Estimasi peak load realistis: **2.500 siswa tap di 1 gerbang saat jam masuk ±5-10 request/detik.** Bottleneck nyata ada di antrian fisik di depan reader, bukan database. Jangan habiskan waktu optimasi premature untuk throughput — fokus ke **reliability offline-sync** dan **kebenaran data** terlebih dulu.

