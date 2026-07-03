# T003 — Auth Module (JWT + Redis + KioskGuard)

## Depends on
T002 (tabel `users` harus sudah ada di DB)

## Objective
Bangun sistem autentikasi lengkap: JWT untuk user login, Redis blacklist untuk logout paksa, dan static device token untuk kiosk yang berjalan unattended.

## Context
- **App:** `apps/api`
- **Module:** `AuthModule`
- **ADR:** ADR-008 (role generik), keputusan auth di `05-API-Endpoints.md`
- **Redis:** untuk blacklist token yang sudah logout

## Spec Detail

### Install:
```
pnpm add @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt ioredis --filter api
pnpm add -D @types/passport-jwt @types/bcrypt --filter api
```

### Endpoint yang dibuat:

**POST `/auth/login`**
- Body: `{ username: string, password: string }`
- Validasi: cek `users` table, bandingkan password dengan `bcrypt.compare`
- Cek `users.status === 'aktif'` — kalau nonaktif, return 401
- Response sukses:
```json
{
  "access_token": "...",   // JWT, expire 15 menit
  "refresh_token": "...",  // JWT, expire 7 hari (30 hari untuk role kepsek)
  "user": { "id", "username", "role", "kampus_id" }
}
```
- Response gagal: 401 `{ "message": "Username atau password salah" }`

**POST `/auth/refresh`**
- Header: `Authorization: Bearer <refresh_token>`
- Cek token valid + tidak ada di Redis blacklist
- Return access_token baru (15 menit)
- **Sliding renewal untuk `kepsek`:** setiap refresh, buat refresh_token baru juga (reset 30 hari)

**POST `/auth/logout`**
- Header: `Authorization: Bearer <access_token>`
- Masukkan `jti` (JWT ID) ke Redis blacklist dengan TTL = sisa waktu expire token
- Return 200 OK

### JWT Payload:
```typescript
interface JwtPayload {
  sub: string;      // users.id
  username: string;
  role: Role;       // enum dari packages/types
  kampus_id: string | null;  // hanya terisi untuk guru_piket
  jti: string;      // unique token ID untuk blacklist
}
```

### Guards yang dibuat:

**`JwtAuthGuard`** (extends `AuthGuard('jwt')`):
- Validasi JWT signature
- Cek `jti` di Redis blacklist → kalau ada, throw 401
- Attach `req.user` dari payload

**`RolesGuard`** (implementasi `CanActivate`):
- Baca metadata dari decorator `@Roles(...roles)`
- Bandingkan dengan `req.user.role`
- Gunakan bersama `JwtAuthGuard`

**`KioskGuard`** (implementasi `CanActivate`):
- Baca `Authorization: Bearer <token>` dari header
- Bandingkan dengan `process.env.KIOSK_DEVICE_TOKEN`
- Tidak melibatkan JWT sama sekali — guard terpisah
- Attach `req.kioskId` dari header `X-Kiosk-ID` (opsional, untuk logging)

### Decorator yang dibuat:
```typescript
@Roles('super_admin', 'card_admin')  // bisa kombinasi role
@Public()  // bypass JwtAuthGuard untuk endpoint publik (login, refresh)
```

### Redis setup:
```typescript
// RedisModule — singleton connection
// Key format blacklist: `blacklist:${jti}`
// TTL = sisa expire token dalam detik
```

## JANGAN
- ❌ JANGAN gunakan session-based auth (express-session) — keputusan sudah JWT
- ❌ JANGAN simpan JWT di cookie — gunakan Authorization header
- ❌ JANGAN buat role `tv_display` baru — TV display menggunakan role `kepsek` dengan long-lived refresh token
- ❌ JANGAN skip Redis blacklist — ini yang membuat logout paksa bekerja
- ❌ JANGAN buat endpoint `GET /auth/me` sekarang — itu bisa ditambahkan nanti kalau dibutuhkan, bukan scope task ini
- ❌ JANGAN hash password di controller — hanya di AuthService
- ❌ JANGAN gunakan `JwtAuthGuard` untuk endpoint kiosk (`/attendance/tap`) — endpoint ini pakai `KioskGuard` yang berbeda

## Files
- **Buat:** `apps/api/src/auth/auth.module.ts`
- **Buat:** `apps/api/src/auth/auth.service.ts`
- **Buat:** `apps/api/src/auth/auth.controller.ts`
- **Buat:** `apps/api/src/auth/strategies/jwt.strategy.ts`
- **Buat:** `apps/api/src/auth/guards/jwt-auth.guard.ts`
- **Buat:** `apps/api/src/auth/guards/roles.guard.ts`
- **Buat:** `apps/api/src/auth/guards/kiosk.guard.ts`
- **Buat:** `apps/api/src/auth/decorators/roles.decorator.ts`
- **Buat:** `apps/api/src/auth/decorators/public.decorator.ts`
- **Buat:** `apps/api/src/redis/redis.module.ts`
- **Modifikasi:** `apps/api/src/app.module.ts` — import AuthModule, RedisModule

## Acceptance Criteria
- [ ] `POST /auth/login` dengan kredensial seed (`admin`/`admin123`) return access_token + refresh_token
- [ ] `POST /auth/login` dengan password salah return 401
- [ ] `POST /auth/refresh` dengan refresh_token valid return access_token baru
- [ ] `POST /auth/logout` + panggil ulang endpoint protected → 401 (token di blacklist)
- [ ] Endpoint yang ditandai `@Public()` bisa diakses tanpa token
- [ ] `KioskGuard` return 401 kalau header token tidak cocok dengan env
- [ ] `RolesGuard` return 403 kalau role tidak sesuai
- [ ] User `piket1` login berhasil, `req.user.kampus_id` terisi sesuai data seed

## Handoff ke T004
T004 dan semua task berikutnya akan menggunakan `JwtAuthGuard` + `RolesGuard`. Pastikan kedua guard bisa di-apply via decorator tanpa setup tambahan.
