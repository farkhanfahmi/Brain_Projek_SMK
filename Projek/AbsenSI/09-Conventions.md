---
tags: [absensi, conventions, git, coding-standard]
updated: 2026-06-25
---

# 09 — Conventions

← [[30.Projects/AbsenSI/00-INDEX|Index]]

---

## Git Workflow — Trunk-Based + Short Feature Branch

- `main` selalu deployable
- Branch fitur umur pendek (idealnya 1-3 hari kerja), naming: `feat/[modul]-[deskripsi-singkat]`
  - Contoh: `feat/kiosk-offline-buffer`, `feat/core-jadwal-mengajar`, `fix/web-rekap-filter`
- **Aturan review:**
  - PR scoped ke `apps/web` saja atau `apps/kiosk` saja → boleh self-merge setelah CI hijau
  - PR yang sentuh `apps/api` (modul Core) atau `packages/types` → **wajib 1 approval** dari developer lain sebelum merge
- Commit message: `[modul] deskripsi singkat` — contoh: `[core] add jadwal mengajar schema`, `[kiosk] implement offline buffer with IndexedDB`

## Penamaan Task & Branch
- Task ID berprefix modul: `task-CORE-XXX`, `task-WEB-XXX`, `task-KIOSK-XXX`
- 1 task = idealnya 1 branch = 1 PR

## Coding Standard (TypeScript)
- ESLint + Prettier config shared via `packages/config` — jangan override per-app tanpa diskusi
- Semua tipe data lintas-app (siswa, guru, kartu, jadwal, attendance record) didefinisikan di `packages/types` — JANGAN duplikasi interface yang sama di tiap app
- NestJS: 1 modul = 1 folder dengan controller/service/dto terpisah, jangan campur logic bisnis di controller
- Prisma: schema diubah lewat migration, jangan edit langsung struktur tabel di production

## Batas Modul (Wajib Dijaga — lihat ADR-003)
- Modul lain TIDAK BOLEH query langsung ke tabel milik modul Core — selalu lewat service layer yang Core sediakan
- Kalau butuh data lintas modul, tambahkan method baru di service Core, JANGAN bypass dengan raw query

## Environment & Secrets
- `.env.example` selalu update kalau ada env var baru
- Jangan commit `.env` asli — detail lengkap di [[30.Projects/AbsenSI/10-Environment|10-Environment]]

## Bahasa
- Kode (variable, function, comment teknis): **Bahasa Inggris** — standar industri, lebih mudah cari referensi
- Dokumentasi Obsidian, commit message, diskusi tim: **Bahasa Indonesia** — sesuai kenyamanan tim
- UI yang dilihat user akhir (siswa/guru/admin sekolah): **Bahasa Indonesia**
