---
tags: [absensi, index, rfid, sekolah]
status: in-progress
updated: 2026-08-04
---

# AbsenSI — Index Proyek

> Sistem Absensi RFID untuk Siswa & Guru SMK. Dieksekusi dengan bantuan AI coding agent
> (awalnya Claude Code, sejak 2026-08-25 juga Hermes Agent — aturan di vault ini berlaku
> untuk agent mana pun).

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
| Brief visual (warna, tipografi, komponen) | `06-Features/design-system/MASTER.md` — **WAJIB dibaca sebelum kerja UI apapun** |
| Spek fitur per modul | `06-Features/*.md` (sebagian besar draft pra-coding, detail implementasi asli ada di kode) |
| Detail task individual | `06-Features/tasks/T0xx-*.md` — dibaca sesuai kebutuhan, jangan baca semua sekaligus |
| Riwayat task lama (arsip, jarang perlu dibaca) | `_archive/` |

---

## Skala & Konteks

- Target: ±2.500 siswa, 100+ guru/karyawan, 1 sekolah (bukan SaaS multi-tenant)
- Repo kode: `/home/anunnaki/Documents/APP SMK/AbsenSI` (dev) + `/home/anunnaki/Documents/APP SMK/AbsenSI-production` (production) — lihat `10-Environment.md`
- Vault ini adalah sumber kebenaran DESAIN, kode aktual adalah sumber kebenaran IMPLEMENTASI. Kalau ada konflik antara vault dan kode, vault menang untuk KEPUTUSAN yang belum diimplementasikan; kode menang untuk DETAIL implementasi yang sudah jalan.

---

## Catatan Penting

- **Bukan lagi rencana tim 3-developer.** Vault ini sempat dirancang untuk skenario 3 developer paralel (lihat `_archive/_claudian/`) — tapi eksekusi aktual sejak awal berjalan sebagai sesi tunggal Claude Code. Dokumen `_claudian/*` diarsipkan, jangan diikuti sebagai workflow yang berlaku sekarang.
- **Tidak ada wikilink `[[...]]` di vault ini** (kecuali heading-link `[[#judul]]` dalam 1 file yang sama) — semua referensi antar file pakai path teks biasa (`lihat 11-Decisions.md`). Diputuskan 2026-08-04 setelah wikilink terbukti rapuh 2x (lihat `_archive/_REPAIR-LOG.md`).
- Proyek ini masih dirancang di vault pribadi. Kalau nanti perlu kolaborasi tim, pertimbangkan migrasi ke vault/repo terpisah — jangan buat keputusan yang terlalu vault-spesifik sebelum itu terjadi.
