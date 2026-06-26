---
tags: [absensi, api]
updated: 2026-06-25
---

# 05 — API Endpoints

← [[Projek/AbsenSI/00-INDEX|Index]]

> Draft kasar — detail lengkap menyusul setelah skema database & spec fitur final. Semua endpoint didokumentasikan via OpenAPI/Swagger auto-generate dari NestJS (lihat [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack]]) — file ini jadi ringkasan navigasi, bukan duplikasi penuh.

---

## Rancangan Awal

| Method | Endpoint | Modul | Keterangan |
|---|---|---|---|
| POST | `/attendance/tap` | Attendance | Endpoint utama dari kiosk — terima UID + timestamp + client_uuid |
| GET | `/attendance/today` | Attendance | Rekap hari ini, dikonsumsi dashboard TV (initial load sebelum WebSocket take over) |
| GET | `/attendance/report` | Attendance | Rekap fleksibel dengan query filter (kelas, jurusan, tanggal) |
| GET/POST | `/cards` | Card | CRUD kartu |
| PATCH | `/cards/:id/revoke` | Card | Nonaktifkan kartu |
| GET/POST | `/schedules` | Schedule | CRUD jadwal |
| WS | `attendance:today` channel | Attendance (Socket.IO) | Push realtime ke dashboard TV |

## ❓ Open Questions
- [ ] Auth strategy untuk endpoint admin (JWT? session?) — belum diputuskan
- [ ] Auth untuk kiosk (device-level token, supaya kiosk tidak butuh login user) — perlu didesain karena kiosk jalan unattended

