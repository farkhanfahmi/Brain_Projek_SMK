---
tags: [absensi, conventions, git, coding-standard]
updated: 2026-06-25
---

# 09 — Conventions

← Index (00-INDEX AbsenSI.md)

---

## Git Workflow — Aktual (dikoreksi 2026-08-04)

> **Koreksi:** bagian ini sebelumnya menulis workflow tim multi-developer (branch `feat/[modul]-*`, PR review, task ID `task-CORE-XXX`) yang **tidak pernah dipraktikkan** — proyek berjalan sebagai sesi tunggal Claude Code sejak awal (lihat `_archive/_claudian/`). Workflow aktual jauh lebih sederhana:

- 2 branch: `dev` (kerja harian) dan `main` (khusus folder production, lihat `10-Environment.md`) — TIDAK ada feature branch per task.
- Commit message: `feat: T0xx — deskripsi singkat` / `fix: T0xx — deskripsi` / `chore: deskripsi` (prefix conventional-commit-style, nomor task kalau relevan). Contoh nyata dari histori: `feat: T103 — sidebar admin dikelompokkan (accordion) + pisah Upload Foto Siswa/Guru`.
- Commit ke `dev` otomatis auto-deploy ke production via git post-commit hook (lihat `10-Environment.md`) — tidak ada proses PR/review manual.
- Task ID: `T0xx` polos (tanpa prefix modul) — didefinisikan di `06-Features/tasks/T0xx-*.md`, statusnya di-track di `STATUS.md`.

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
- Jangan commit `.env` asli — detail lengkap di 10-Environment (10-Environment.md)

## Bahasa
- Kode (variable, function, comment teknis): **Bahasa Inggris** — standar industri, lebih mudah cari referensi
- Dokumentasi Obsidian, commit message, diskusi tim: **Bahasa Indonesia** — sesuai kenyamanan tim
- UI yang dilihat user akhir (siswa/guru/admin sekolah): **Bahasa Indonesia**

