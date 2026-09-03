---
tags: [absensi, index, rfid, sekolah]
status: in-progress
updated: 2026-08-31
---

# AbsenSI — Index Proyek

> Sistem Absensi RFID untuk Siswa & Guru SMK. Dieksekusi dengan bantuan AI coding agent
> (awalnya Claude Code, sejak 2026-08-25 juga Hermes Agent).
>
> **[2026-08-31] Pembagian peran ketat**: Hermes = mitra diskusi kritis + penyusun spec task
> (baca kode, SELECT read-only DB, start/stop server — TIDAK menulis kode/DB/`.env`, TIDAK
> memanggil Claude Code CLI). Claude Code = eksekutor kode, SELALU dipicu manual oleh user.
> Detail lengkap: `workflow-2-sesi-diskusi-eksekusi.md`.

**Mulai dari sini setiap sesi baru: baca STATUS.md dulu** — itu satu-satunya tempat
untuk tahu apa yang sudah selesai dan apa yang masih perlu dikerjakan. File ini (INDEX)
hanya peta navigasi ke dokumen lain, bukan tempat cek progres.

---

## Peta Dokumen

| Kebutuhan | File |
|---|---|
| **Status & task aktif (cek ini duluan)** | `STATUS.md` |
| Latar belakang, visi, skala target | `01-Overview.md` |
| Stack teknis lengkap | `02-Tech-Stack.md` |
| Role & permission | `03-User-Roles.md` |
| Skema database | `04-Database-Schema.md` (cross-check `apps/api/prisma/schema.prisma` untuk detail terkini) |
| API endpoints (draft, belum sinkron) | `05-API-Endpoints.md` |
| Konvensi kode & git | `09-Conventions.md` |
| Environment, topologi dev/production, backup | `10-Environment.md` |
| Keputusan arsitektur (ADR-001 s/d ADR-025+) | `11-Decisions.md` — **terpelihara aktif** |
| **Workflow Hermes ⟷ Claude Code (2 sesi, batas kewenangan, format task)** | `workflow-2-sesi-diskusi-eksekusi.md` — **WAJIB dibaca sebelum sesi diskusi/eksekusi apa pun** |
| **Akses data production (siapa bisa apa, cara Hermes cek data untuk task)** | `06-Features/akses-data-production.md` — **WAJIB dibaca Claude Code kalau task menyebut "cek data production"** |
| Template task baru (format 8-bagian) | `06-Features/tasks/_task-template.md` |
| Brief visual v1 (warna, tipografi, komponen — sebagian sudah revamp ke v2) | `06-Features/design-system/MASTER.md` |
| **Design System v2 (token, kontrak komponen, Figma, governance)** | `06-Features/design-system-v2/00-INDEX.md` — **WAJIB dibaca sebelum kerja UI apapun** |
| Spek fitur per modul (2026-08-31: dikonsolidasi — status diverifikasi ke kode, file duplikat digabung, arsip dipindah) | `06-Features/*.md` |
| Detail task individual | `06-Features/tasks/T0xx-*.md` (task lama) atau `task-MODUL-NNN-*.md` (task baru sejak 2026-08-31) — dibaca sesuai kebutuhan, jangan baca semua sekaligus |
| Riwayat task lama (arsip, jarang perlu dibaca) | `_archive/` |
| **154 task SELESAI (detail lengkap, dipisah dari STATUS.md 2026-08-31)** | `_archive/STATUS-Arsip-Selesai.md` |

---

## Skala & Konteks

- Target: ±2.500 siswa, 100+ guru/karyawan, 1 sekolah (bukan SaaS multi-tenant)
- Repo kode: **DEV** `C:\ProjekSMK\AbsenSI` (Windows, branch `dev`) + **PRODUCTION** `/home/anunnaki/Documents/APP SMK/AbsenSI-production` (Linux, branch `main`) — 2 mesin terpisah sejak 2026-08-25, lihat `10-Environment.md` untuk topologi & prosedur sinkronisasi
- Vault ini adalah sumber kebenaran DESAIN, kode aktual adalah sumber kebenaran IMPLEMENTASI. Kalau ada konflik antara vault dan kode, vault menang untuk KEPUTUSAN yang belum diimplementasikan; kode menang untuk DETAIL implementasi yang sudah jalan.

---

## Catatan Penting

- **Bukan lagi rencana tim 3-developer.** Vault ini sempat dirancang untuk skenario 3 developer paralel (lihat `_archive/_claudian/`) — tapi eksekusi aktual sejak awal berjalan sebagai sesi tunggal Claude Code. Dokumen `_claudian/*` diarsipkan, jangan diikuti sebagai workflow yang berlaku sekarang.
- **Tidak ada wikilink `[[...]]` di vault ini** (kecuali heading-link `[[#judul]]` dalam 1 file yang sama) — semua referensi antar file pakai path teks biasa (`lihat 11-Decisions.md`). Diputuskan 2026-08-04 setelah wikilink terbukti rapuh 2x (lihat `_archive/_REPAIR-LOG.md`).
- Proyek ini masih dirancang di vault pribadi. Kalau nanti perlu kolaborasi tim, pertimbangkan migrasi ke vault/repo terpisah — jangan buat keputusan yang terlalu vault-spesifik sebelum itu terjadi.
