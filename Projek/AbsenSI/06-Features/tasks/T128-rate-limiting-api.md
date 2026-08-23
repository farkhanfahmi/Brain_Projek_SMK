# T128 — API: Tambah Rate Limiting (Tidak Ada Sama Sekali di Seluruh API)

## Depends on
Tidak ada dependency teknis. Ini penambahan infrastruktur baru (throttler global), bukan fix ke fitur tertentu.

## Objective
Semua endpoint API punya batas laju permintaan (rate limit) yang wajar — saat ini TIDAK ADA rate limiting sama sekali di seluruh `apps/api`, endpoint apa pun (buat izin, lock siswa, login, dll) bisa dipanggil berkali-kali tanpa batas dari sisi server.

## Context
- **App:** `apps/api` — infrastruktur baru (global), bukan per-fitur.
- **Dikonfirmasi 2026-08-06** lewat audit keamanan dashboard piket (Explore agent, baca kode langsung): `grep -rn "Throttle\|ThrottlerModule\|ThrottlerGuard"` di seluruh `apps/api/src` — **0 hasil**. Package `@nestjs/throttler` **TIDAK ADA** di `package.json`. Ini bukan celah spesifik 1 fitur, ini gap infrastruktur menyeluruh — ditemukan saat audit dashboard piket tapi berlaku untuk SELURUH API, termasuk endpoint auth/login yang secara umum paling kritis untuk dibatasi (brute-force).

## Spec Detail

### Pilihan Pendekatan (untuk diputuskan/dikonfirmasi user sebelum eksekusi kalau ambigu)
- **Rekomendasi**: pasang `@nestjs/throttler` sebagai `ThrottlerModule` GLOBAL (`app.module.ts`, `ThrottlerGuard` sebagai `APP_GUARD`) dengan batas default wajar (misal 100 request/menit per IP) — melindungi SEMUA endpoint sekaligus tanpa perlu menambah decorator satu-satu di puluhan controller.
- **Endpoint yang butuh batas LEBIH KETAT dari default** (rawan abuse/brute-force):
  - `POST /auth/login` — batas jauh lebih ketat (misal 5-10 percobaan/menit per IP) untuk mencegah brute-force password.
  - Endpoint mutasi piket yang baru diaudit (permits create, lock/unlock, late-entry-slip create) — pertimbangkan batas sedang (bukan seketat login, tapi lebih ketat dari default umum) karena ini aksi manusia yang wajar jarang dilakukan puluhan kali per menit oleh 1 akun.
  - Endpoint kiosk `POST /attendance/tap` — **HATI-HATI**: endpoint ini secara LEGITIMATE menerima banyak request cepat saat sync buffer offline (T122, retry tiap 5 detik, bisa ratusan tap menumpuk sekaligus saat kiosk baru online lagi setelah lama offline) — JANGAN pasang rate limit yang terlalu ketat di endpoint ini sampai memblokir sync buffer yang sah. Pertimbangkan exclude endpoint ini dari throttle global, atau batas yang jauh lebih longgar khusus untuk endpoint ini.
- **Endpoint publik tanpa auth** (`GET /health/time` dari T124, endpoint publik ekstrakurikuler `ekstra-publik`) — pertimbangkan batas tersendiri, karena ini paling rawan diakses tanpa kontrol sama sekali (tidak ada JWT untuk identifikasi pelaku).

### Implementasi
- Install `@nestjs/throttler`.
- `app.module.ts` — daftarkan `ThrottlerModule.forRoot([{ ttl: ..., limit: ... }])` + `{ provide: APP_GUARD, useClass: ThrottlerGuard }`.
- Untuk endpoint yang butuh override (login lebih ketat, tap kiosk lebih longgar/exclude) — pakai decorator `@Throttle()` per-method atau `@SkipThrottle()` sesuai kebutuhan.
- Response saat rate limit terlampaui — pastikan pesan error jelas (429 Too Many Requests, bukan error generik yang membingungkan).

## Edge Cases
- Kiosk di belakang NAT/1 IP publik yang sama untuk banyak device (kalau infrastruktur jaringan sekolah begitu) — rate limit berbasis IP bisa salah kena ke semua kiosk sekaligus kalau limitnya terlalu ketat. Pertimbangkan throttle berbasis kombinasi IP+device-token untuk endpoint kiosk, bukan cuma IP polos, KALAU ini jadi masalah nyata saat testing.
- Sync buffer offline kiosk (T122) yang legitimate mengirim banyak tap sekaligus saat baru online lagi setelah down lama — JANGAN sampai throttle baru ini justru menghalangi proses sync yang sah, ini prioritas tertinggi untuk dites eksplisit.

## Files
- **Modifikasi:** `apps/api/package.json` (dependency baru), `apps/api/src/app.module.ts` (registrasi global), kemungkinan `apps/api/src/auth/auth.controller.ts` (override lebih ketat untuk login), `apps/api/src/attendance/attendance.controller.ts` (kemungkinan exclude/longgar untuk endpoint tap).

## Keputusan Final (dikonfirmasi user 2026-08-07)
1. **Limit global default**: 300 request/menit per IP (lebih longgar dari rekomendasi awal 100/menit — pertimbangan dashboard admin fetch banyak paralel: tabel+grafik+filter).
2. **Limit login**: 10 percobaan/menit per IP.
3. **Endpoint tap kiosk (`POST /attendance/tap`)**: dikecualikan TOTAL dari rate limit (`@SkipThrottle()`), bukan sekadar dilonggarkan — endpoint ini sudah dijaga `KioskGuard` (device token+IP whitelist), risiko abuse lebih rendah dari endpoint publik.

## Acceptance Criteria
- [x] Ada batas rate limit global yang berlaku untuk semua endpoint secara default. Diverifikasi live: header `X-RateLimit-Limit: 300` muncul di response endpoint biasa.
- [x] `POST /auth/login` punya batas lebih ketat dari default (mitigasi brute-force). Diverifikasi live: 10 request pertama lolos (400, username salah — respons normal), request ke-11/12 → 429.
- [x] Endpoint `POST /attendance/tap` TIDAK terhalang oleh rate limit saat skenario sync buffer offline mengirim banyak tap beruntun. **Test eksplisit dilakukan**: 50 request tap beruntun (client_uuid unik tiap request, uid sama) ke endpoint sungguhan — 0 dari 50 kena 429.
- [x] Percobaan melebihi batas menghasilkan response 429 yang jelas. `{"statusCode":429,"message":"ThrottlerException: Too Many Requests"}` — bukan error generik.
- [x] Build + type-check `apps/api` hijau, jest existing tetap lulus. `tsc --noEmit` bersih, jest 192/192 (tidak ada regresi — unit test tidak melalui HTTP layer jadi tidak tersentuh guard baru).

## Status Eksekusi — SELESAI (2026-08-07)
`app.module.ts` — `ThrottlerModule.forRoot([{ ttl: 60000, limit: 300 }])` + `{ provide: APP_GUARD, useClass: ThrottlerGuard }` (guard global, berlaku ke SEMUA endpoint secara default tanpa perlu decorator satu-satu). `auth.controller.ts` `login()` — `@Throttle({ default: { ttl: 60000, limit: 10 } })` override lebih ketat. `attendance.controller.ts` `tap()` — `@SkipThrottle()`, dikecualikan total.

**Verifikasi live paling kritis (simulasi sync buffer offline, T122-T125)**: karena `KioskGuard` dev juga cek IP whitelist per-kiosk (kiosk id=1 terdaftar `10.10.10.103`, bukan localhost), `allowed_ip` diubah SEMENTARA ke `127.0.0.1` di DB dev untuk memungkinkan test HTTP asli lewat curl localhost, dikirim 50 tap beruntun, dikonfirmasi 0 kena 429, lalu `allowed_ip` DIKEMBALIKAN ke `10.10.10.103` segera setelah test (tidak meninggalkan perubahan data dev permanen).

**Endpoint ping kiosk lain dicek juga** (`/health/time` via `/api/server-health` di kiosk app) — interval ping 20 detik = 3 request/menit per kiosk, jauh di bawah limit 300/menit global, TIDAK perlu exclude khusus (dibiarkan kena limit global biasa, aman).

## Validasi Claudian
- [x] Test skenario sync buffer offline WAJIB dilakukan sebelum dianggap selesai — dilakukan via HTTP asli (bukan cuma baca kode/asumsi `@SkipThrottle()` pasti benar), 50 tap beruntun 0 gagal.
- [x] Angka batas dikonfirmasi eksplisit ke user via AskUserQuestion sebelum eksekusi (global 300/menit dipilih user, lebih longgar dari rekomendasi awal 100/menit; login 10/menit dipilih user, lebih longgar dari rekomendasi awal 5/menit; tap exclude total dipilih user) — bukan asal pilih angka.
