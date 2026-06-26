# Project Context — AbsenSI

**Stack:** NestJS + Prisma + PostgreSQL (apps/api) · Next.js + TailwindCSS (apps/web, apps/kiosk) · Turborepo monorepo · Redis + BullMQ
**Status:** Fase 1 — masih tahap perancangan spec fitur gerbang, belum mulai coding
**Docs lengkap:** [[Projek/AbsenSI/00-INDEX|00-INDEX]]
**Tim:** 3 developer, modular ownership — lihat [[Projek/AbsenSI/_claudian/team|team.md]]

## Keputusan Aktif
- 1 sekolah saja, bukan SaaS multi-tenant
- Modular monolith, bukan microservices (ADR-003)
- Reader RFID = USB-HID keyboard emulation, client kiosk = web app biasa (ADR-004)
- Fase 1 hanya gerbang, terlambat dihitung dari jam masuk sekolah (ADR-005)
- Tidak import data spreadsheet lama (ADR-001)
- Event-driven via BullMQ untuk siapkan jalur notifikasi fase 3 (ADR-006)
- CONTEXT.md dipisah per app, bukan satu di root (lihat workflow-multi-dev.md)
- Role generik: `super_admin` (full akses, saat ini = 3 developer), `card_admin` (CRUD kartu saja), `guru` (read-only riwayat sendiri), `kepsek` (read-only dashboard TV + rekap, akun login tersendiri) — permission dicek di backend, bukan UI (ADR-008)
- TV dashboard tetap butuh auth, BUKAN akses publik tanpa login
- Siswa TIDAK punya akun login — interaksi cuma fisik tap
- Fase 2 (planning): siklus sesi kelas dikunci tap-buka/tap-tutup oleh guru untuk cegah tap iseng siswa — banyak edge case belum dijawab, lihat Open Questions di absensi-kelas-mapel.md

## Sedang Dikerjakan
Merancang spec fitur Absensi Gerbang (06-Features/absensi-gerbang.md) dan skema database awal.

## Jangan Lupakan
- Gerbang sekolah TIDAK ada penghalang fisik — aturan "wajib tap gerbang dulu" (fase 2) punya risiko false-negative, perlu keputusan UX sebelum fase 2 mulai
- Repo & vault masih di vault pribadi — akan dipindah ke vault tim + repo GitHub terpisah saat siap kolaborasi
- Jangan over-engineer untuk throughput — beban riil rendah (±5-10 req/s peak), fokus reliability offline-sync

