---
tags: [absensi, backlog, roadmap]
updated: 2026-06-25
---

# 13 — Backlog & Roadmap Fase

← [[30.Projects/AbsenSI/00-INDEX|Index]]

---

## 🟢 Fase 1 — Absensi Gerbang (sedang dirancang)
Lihat [[30.Projects/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]] untuk Open Questions yang harus dijawab sebelum mulai coding.

## 🟡 Fase 2 — Absensi Kelas & Mapel (planning)
Lihat [[30.Projects/AbsenSI/06-Features/absensi-kelas-mapel|absensi-kelas-mapel.md]].
**Blocker untuk mulai fase ini:** keputusan soal hard-block vs soft-block gerbang-dulu (risiko fisik gerbang tanpa penghalang).

## 🔵 Fase 3 — Ide Masa Depan (belum direncanakan detail)
- Notifikasi WA/SMS ke orang tua (jalur sudah disiapkan, lihat ADR-006 + notifikasi-ortu.md)
- Import data historis dari spreadsheet lama (kalau ternyata dibutuhkan — ADR-001 saat ini menolaknya)
- Integrasi ke sistem akademik/nilai
- Bulk import kartu via CSV untuk rollout awal 2.500 kartu (kemungkinan ini harus naik prioritas ke fase 1 — lihat Open Question di manajemen-kartu.md)
- AbsenSI jadi modul dari "OS sekolah" yang lebih besar — kemungkinan modul Core (siswa/guru/jadwal) dipecah jadi service terpisah yang dikonsumsi app lain

## Technical Debt / Risiko yang Diketahui
- Gerbang tanpa penghalang fisik — risiko false-negative di fase 2 (lihat absensi-kelas-mapel.md)
- Belum ada keputusan soal threshold terlambat guru per-guru vs global
- Belum ada keputusan retensi data buffer offline kiosk saat mati listrik total
