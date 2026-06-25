---
tags: [absensi, index, rfid, sekolah]
status: planning
updated: 2026-06-25
---

# AbsenSI — Index Proyek

> Sistem Absensi RFID untuk Siswa & Guru SMK. Proyek perintis dari ekosistem aplikasi sekolah yang lebih besar. Dikembangkan oleh tim 3 guru-developer, dieksekusi via Claude Code, dirancang via Claudian di Obsidian.

---

## 📋 Status Singkat

| Item | Detail |
|---|---|
| Phase | **Fase 1** — Absensi Gerbang (perencanaan) |
| Tim | 3 developer (lihat [[30.Projects/AbsenSI/_claudian/team|team.md]]) |
| Stack | TypeScript full-stack — lihat [[30.Projects/AbsenSI/02-Tech-Stack|02-Tech-Stack]] |
| Repo | Monorepo Turborepo (belum dibuat — masih tahap desain di vault ini) |
| Skala target | ±2.500 siswa, 100+ guru/karyawan, 1 sekolah |

---

## 🗂️ Navigasi

### Dokumen Inti
- [[30.Projects/AbsenSI/01-Overview|01 — Overview]] — latar belakang, visi, scope, fase pengembangan
- [[30.Projects/AbsenSI/02-Tech-Stack|02 — Tech Stack]] — stack, arsitektur monorepo, alasan tiap pilihan
- [[30.Projects/AbsenSI/03-User-Roles|03 — User Roles]] — role pengguna aplikasi (bukan tim dev)
- [[30.Projects/AbsenSI/04-Database-Schema|04 — Database Schema]]
- [[30.Projects/AbsenSI/05-API-Endpoints|05 — API Endpoints]]

### Fitur
- [[30.Projects/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang]] (Fase 1 — sedang dirancang)
- [[30.Projects/AbsenSI/06-Features/absensi-kelas-mapel|Absensi Kelas & Mapel]] (Fase 2 — planning)
- [[30.Projects/AbsenSI/06-Features/manajemen-kartu|Manajemen Kartu RFID]]
- [[30.Projects/AbsenSI/06-Features/akun-guru|Akun & Riwayat Kehadiran Guru]]
- [[30.Projects/AbsenSI/06-Features/dashboard-tv|Dashboard TV Realtime]]
- [[30.Projects/AbsenSI/06-Features/notifikasi-ortu|Notifikasi Orang Tua]] (Fase 3 — jalur disiapkan, belum dibangun)

### Proses & Keputusan
- [[30.Projects/AbsenSI/07-User-Flows|07 — User Flows]]
- [[30.Projects/AbsenSI/08-UI-UX-Guidelines|08 — UI/UX Guidelines]]
- [[30.Projects/AbsenSI/09-Conventions|09 — Conventions]] — coding standard + git workflow tim
- [[30.Projects/AbsenSI/10-Environment|10 — Environment]]
- [[30.Projects/AbsenSI/11-Decisions|11 — Decisions (ADR)]]
- [[30.Projects/AbsenSI/12-Status|12 — Status]] — board per modul/developer
- [[30.Projects/AbsenSI/13-Backlog|13 — Backlog & Roadmap Fase]]
- [[30.Projects/AbsenSI/14-Debug-Log|14 — Debug Log]]
- [[30.Projects/AbsenSI/15-Deployment-Guide|15 — Deployment Guide]]

### Khusus Tim (`_claudian/`)
- [[30.Projects/AbsenSI/_claudian/team|team.md]] — pembagian modul, kontak, area tanggung jawab
- [[30.Projects/AbsenSI/_claudian/project-context|project-context.md]] — quick-attach awal sesi
- [[30.Projects/AbsenSI/_claudian/discussion-log|discussion-log.md]] — buffer keputusan sebelum masuk ADR
- [[30.Projects/AbsenSI/_claudian/workflow-multi-dev|workflow-multi-dev.md]] — workflow Claudian khusus tim 3 orang

---

## ⚠️ Catatan Penting

Proyek ini **masih dirancang di vault pribadi**. Saat siap kolaborasi, seluruh folder ini akan dipindah ke vault khusus tim + repo GitHub terpisah. Jangan buat keputusan final yang terlalu vault-spesifik sampai migrasi itu terjadi.
