---
tags: [absensi, api]
updated: 2026-06-25
---

# 05 — API Endpoints

← Index (00-INDEX AbsenSI.md)

> Draft kasar — detail lengkap menyusul setelah skema database & spec fitur final. Semua endpoint didokumentasikan via OpenAPI/Swagger auto-generate dari NestJS (lihat 02-Tech-Stack (02-Tech-Stack.md)) — file ini jadi ringkasan navigasi, bukan duplikasi penuh.

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

## ✅ Open Questions — Resolved (2026-07-03)

- [x] **Auth strategy endpoint admin/user** → **JWT + Redis blacklist untuk revoke paksa.** JWT stateless (server tidak perlu session state per request), tapi kalau akun dinonaktifkan atau logout paksa oleh admin, token masuk Redis blacklist dan langsung tidak berlaku. Token access: umur pendek (15 menit). Token refresh: umur panjang (7 hari untuk user biasa, 30 hari sliding untuk TV display `kepsek`). Implementasi di NestJS: `@nestjs/jwt` + `passport-jwt` + Redis via `ioredis`.
- [x] **Auth kiosk (unattended device)** → **Static device token per kiosk (Opsi A).** String token panjang (256-bit random) disimpan di `.env` kiosk, dikirim sebagai `Authorization: Bearer <device-token>` di setiap request ke `/attendance/tap`. Server validasi token ini di guard khusus `KioskGuard`, terpisah dari `JwtAuthGuard` user biasa. Token kiosk tidak expire — revoke dilakukan manual lewat rotasi env variable kalau kiosk dicuri/hilang. Tidak ada login flow untuk kiosk.

