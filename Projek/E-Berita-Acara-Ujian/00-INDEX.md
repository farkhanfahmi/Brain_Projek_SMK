---
tags:
  - project
  - index
  - smk
  - berita-acara
created: 2026-06-04
updated: 2026-06-04
---

# E-Berita Acara Ujian — Index

> Sistem manajemen presensi dan berita acara ujian digital untuk SMK Kartanegara Wates. Menggantikan form kertas dengan sistem real-time berbasis QR code.

**Status Proyek:** 🟢 Aktif — v1.0 Produksi
**Lokasi Kode:** `/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/`

---

## Navigasi Cepat

| File | Deskripsi |
|------|-----------|
| [[01-Overview]] | Latar belakang, visi, scope, timeline |
| [[02-Tech-Stack]] | Stack lengkap & arsitektur sistem |
| [[03-User-Roles]] | Aktor, hak akses, matriks izin |
| [[04-Database-Schema]] | Semua tabel, kolom, relasi |
| [[05-API-Endpoints]] | Semua route API |
| [[06-Features]] | Fitur per modul |
| [[07-User-Flows]] | Alur kerja per aktor |
| [[08-UI-UX-Guidelines]] | Design system & komponen |
| [[09-Conventions]] | Coding standards & conventions |
| [[10-Environment]] | Setup dev/prod & env vars |
| [[11-Decisions]] | ADR — keputusan arsitektur |
| [[12-Status]] | Sprint aktif & progress |
| [[13-Backlog]] | Roadmap & technical debt |
| [[14-Debug-Log]] | Log sesi per tanggal |

---

## Status Ringkas

- **Backend:** Laravel 11, berjalan di produksi
- **Frontend Pengawas/Panitia:** React 19, mobile-first
- **Frontend Admin:** React 19, full CRUD management
- **Frontend TV:** React 18, wall-display real-time
- **Database:** MySQL (produksi), SQLite (dev)
- **Auth:** Sanctum token (pengawas) + localStorage session (panitia)

---

## Aktor Sistem

1. **Admin** — Manajemen data master via `frontend-admin`
2. **Pengawas** — Scan peserta + submit Berita Acara via `frontend`
3. **Panitia** — Monitor lintas ruang + catatan pelanggaran via `frontend`
4. **TV Display** — Barcode reader presensi masuk/pulang via `frontend-tv`
