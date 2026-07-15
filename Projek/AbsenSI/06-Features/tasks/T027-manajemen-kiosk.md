# T027 — Manajemen Kiosk: Auth Berlapis (Token + IP Whitelist)

## Status
🆕 Task baru — refactor dari desain lama (ADR-008) ke ADR-021

## Depends on
T002 (schema), T003 (KioskGuard sudah ada — harus direfaktor), T012 (tap API pakai KioskGuard)

## Objective
Ganti sistem auth kiosk dari static env token ke token per-kiosk berbasis database, ditambah validasi IP whitelist. Bangun UI manajemen kiosk di admin dashboard.

## Context
- **ADR:** ADR-021 (supersedes bagian kiosk di ADR-008)
- **Apps:** `apps/api` (guard + migration) + `apps/web` (admin UI) + `apps/kiosk` (baca token dari URL)
- **Breaking change:** `KIOSK_DEVICE_TOKEN` di `.env` tidak dipakai lagi setelah task ini selesai

---

## Spec Detail

### 1. Migrasi Database — Tabel `kiosks`

Tambahkan ke `schema.prisma`:

```prisma
model Kiosk {
  id           String    @id @default(cuid())
  nama         String
  kampusId     String
  kampus       Kampus    @relation(fields: [kampusId], references: [id])
  deviceToken  String    @unique @db.VarChar(100)
  allowedIp    String    @db.VarChar(45)   // IPv4 atau IPv6
  isActive     Boolean   @default(true)
  lastSeenAt   DateTime?
  lastSeenIp   String?   @db.VarChar(45)
  createdBy    String
  createdByUser User     @relation(fields: [createdBy], references: [id])
  createdAt    DateTime  @default(now())

  tapEvents    TapEvent[]

  @@map("kiosks")
}
```

Update relasi di `TapEvent`:
```prisma
kioskId   String?
kiosk     Kiosk?  @relation(fields: [kioskId], references: [id])
```

Jalankan: `npx prisma migrate dev --name add_kiosks_table`

---

### 2. Refaktor `KioskGuard`

**File:** `apps/api/src/auth/guards/kiosk.guard.ts`

Logic baru:
```typescript
// 1. Extract token dari Authorization header
const token = extractBearerToken(request);
if (!token) throw new UnauthorizedException();

// 2. Cari kiosk di DB berdasarkan token
const kiosk = await this.prisma.kiosk.findUnique({
  where: { deviceToken: token, isActive: true }
});
if (!kiosk) throw new UnauthorizedException('Token kiosk tidak valid');

// 3. Validasi IP
const clientIp = extractClientIp(request); // handle X-Forwarded-For jika ada
if (clientIp !== kiosk.allowedIp) {
  // Log sebagai suspicious tap
  this.logger.warn(`IP mismatch: kiosk ${kiosk.id}, expected ${kiosk.allowedIp}, got ${clientIp}`);
  throw new ForbiddenException('IP tidak terdaftar untuk kiosk ini');
}

// 4. Update last_seen (fire-and-forget, tidak await — tidak boleh slow down tap)
this.prisma.kiosk.update({
  where: { id: kiosk.id },
  data: { lastSeenAt: new Date(), lastSeenIp: clientIp }
}).catch(() => {}); // silent fail — jangan crash tap karena update metadata

// 5. Attach ke request untuk dipakai AttendanceService
request.kiosk = { id: kiosk.id, kampusId: kiosk.kampusId };
```

`KioskGuard` perlu inject `PrismaService`. Karena ini guard, inject lewat constructor (bukan `@InjectRepository`) — `PrismaService` sudah global dari T002/T003.

---

### 3. Update `AttendanceService.tap()`

`kiosk_id` di `tap_events` sekarang diambil dari `request.kiosk.id` (hasil guard), **bukan** dari request body:

```typescript
// Sebelum (T012 — body):
kioskId: body.kiosk_id

// Sesudah (T027 — dari guard):
kioskId: req.kiosk.id
```

Hapus field `kiosk_id` dari `TapDto` — kiosk tidak perlu lagi self-report identitasnya.

---

### 4. Update `apps/kiosk` — Baca Token dari URL

**File:** `apps/kiosk/src/lib/kiosk-init.ts` (buat baru)

```typescript
export function initKioskToken(): string | null {
  // Prioritas 1: URL param (saat buka URL pertama kali / saat homepage terbuka)
  const urlParams = new URLSearchParams(window.location.search);
  const tokenFromUrl = urlParams.get('device');
  if (tokenFromUrl) {
    localStorage.setItem('kiosk_device_token', tokenFromUrl);
    return tokenFromUrl;
  }
  // Prioritas 2: localStorage (saat browser buka tanpa URL param — misalnya refresh)
  return localStorage.getItem('kiosk_device_token');
}
```

Panggil `initKioskToken()` di root layout kiosk saat mount. Token disimpan ke context/state, dikirim ke Route Handler proxy `/api/tap` sebagai header dari client.

**File:** `apps/kiosk/src/app/api/tap/route.ts`

Token tidak lagi dari `process.env.KIOSK_DEVICE_TOKEN` — token diterima dari client request header `X-Kiosk-Token`, lalu proxy server-side sertakan sebagai `Authorization: Bearer <token>` ke NestJS. Token tidak pernah di-hardcode di env kiosk.

Tampilkan layar error `"Kiosk belum dikonfigurasi. Hubungi admin."` kalau localStorage kosong dan tidak ada URL param `device`.

---

### 5. Admin UI — Halaman Manajemen Kiosk

**Route:** `/kiosk` di admin dashboard (`apps/web`)

**Tabel kiosk:**
| Nama | Kampus | IP | Status | Terakhir Aktif | Aksi |
|---|---|---|---|---|---|
| Gerbang Utama | Kampus 1 | 10.10.10.51 | 🟢 Online | 2 menit lalu | [URL] [Edit] [Nonaktifkan] |
| Gerbang Selatan | Kampus 2 | 10.10.10.52 | 🔴 Offline | 3 jam lalu | [URL] [Edit] [Nonaktifkan] |

Status online/offline = `last_seen_at` dalam 5 menit terakhir → online; lebih dari 5 menit → offline.

**Tombol [+ Tambah Kiosk]** → modal form:
- Nama kiosk (text, required)
- Kampus (dropdown, required)
- IP Address (text, required, validasi format IPv4)
- Submit → server generate `deviceToken` (nanoid(32) atau crypto random) → simpan ke DB

**Tombol [URL]** → modal tampilkan:
```
URL Kiosk:
http://10.10.10.50/kiosk?device=TOKEN_DISINI

[Copy URL]  [Tampilkan QR Code]
```
QR code generate di browser menggunakan library `qrcode` (client-side, tidak perlu backend).

**Tombol [Edit]** → modal update nama / IP address (token tidak bisa diubah, hanya nonaktifkan + buat baru).

**Tombol [Nonaktifkan]** → konfirmasi dialog → `PATCH /kiosks/:id/deactivate` → `isActive: false` → kiosk tidak bisa tap lagi (guard tolak token nonaktif).

---

### 6. API Endpoints Baru

Semua akses: `super_admin`

```
GET    /kiosks              — list semua kiosk + status online/offline
POST   /kiosks              — buat kiosk baru, generate token
GET    /kiosks/:id          — detail + URL lengkap
PATCH  /kiosks/:id          — update nama, IP
PATCH  /kiosks/:id/deactivate — nonaktifkan
PATCH  /kiosks/:id/rotate-token — generate token baru (URL lama tidak berlaku)
```

`rotate-token` penting untuk skenario: browser kiosk di-clear, atau device rusak dan diganti — operator minta token baru dari admin, set di homepage device baru.

---

### 7. Hapus dari `.env`

```diff
- KIOSK_DEVICE_TOKEN=your-static-token-here
```

Update `10-Environment.md` dan `CLAUDE.md` untuk mencerminkan perubahan ini.

---

## JANGAN
- ❌ JANGAN expose `deviceToken` di response GET `/kiosks` (list) — hanya tampilkan lewat endpoint `/kiosks/:id` atau modal URL yang butuh aksi eksplisit dari admin
- ❌ JANGAN `await` update `lastSeenAt` di dalam guard — ini akan memperlambat setiap tap. Fire-and-forget dengan `.catch(() => {})`
- ❌ JANGAN simpan token di `NEXT_PUBLIC_*` env — token harus lewat Route Handler server-side, tidak boleh terekspos ke browser JS kiosk
- ❌ JANGAN biarkan kiosk self-report `kiosk_id` di request body — `kiosk_id` harus datang dari guard berdasarkan token yang tervalidasi
- ❌ JANGAN lupa update `TapDto` — hapus field `kiosk_id` dari DTO setelah guard yang menyuplai nilainya
- ❌ JANGAN hardcode URL base di QR code — baca dari `NEXT_PUBLIC_KIOSK_URL` atau derive dari `window.location.origin`

## Files

**Buat:**
- `apps/api/src/kiosks/kiosks.module.ts`
- `apps/api/src/kiosks/kiosks.service.ts`
- `apps/api/src/kiosks/kiosks.controller.ts`
- `apps/api/src/kiosks/dto/create-kiosk.dto.ts`
- `apps/kiosk/src/lib/kiosk-init.ts`
- `apps/web/app/(admin)/kiosk/page.tsx`
- `apps/web/app/(admin)/kiosk/components/KioskTable.tsx`
- `apps/web/app/(admin)/kiosk/components/KioskUrlModal.tsx`

**Modifikasi:**
- `apps/api/prisma/schema.prisma` — tambah model `Kiosk`, update `TapEvent`
- `apps/api/src/auth/guards/kiosk.guard.ts` — refaktor total
- `apps/api/src/attendance/dto/tap.dto.ts` — hapus `kiosk_id`
- `apps/api/src/attendance/attendance.service.ts` — ambil `kiosk_id` dari `req.kiosk.id`
- `apps/kiosk/src/app/api/tap/route.ts` — token dari client header, bukan env
- `apps/kiosk/src/app/page.tsx` — panggil `initKioskToken()`, tampilkan error kalau tidak ada token
- `Projek/AbsenSI/10-Environment.md` — hapus `KIOSK_DEVICE_TOKEN`
- `CLAUDE.md` — update seksi Auth Architecture (kiosk)

## Acceptance Criteria
- [ ] Admin buat kiosk baru di dashboard → URL + QR code tampil
- [ ] Set URL tersebut sebagai homepage browser kiosk → token terbaca dari URL → tersimpan ke localStorage
- [ ] Tap kartu di kiosk → `KioskGuard` validasi token (dari DB) + IP → sukses → `tap_events.kiosk_id` terisi dari DB, bukan body
- [ ] Tap dari IP yang tidak terdaftar (meski token benar) → 403 Forbidden
- [ ] Token nonaktif (`isActive: false`) → 401 Unauthorized
- [ ] Admin klik [Nonaktifkan] → kiosk tidak bisa tap lagi
- [ ] Admin klik [Rotate Token] → URL lama tidak berlaku, URL baru bisa dipakai
- [ ] `last_seen_at` terupdate setelah setiap tap sukses
- [ ] Status online/offline di tabel admin akurat (threshold 5 menit)
- [ ] `KIOSK_DEVICE_TOKEN` tidak ada lagi di `.env` — aplikasi tetap jalan tanpa error
- [ ] Layar kiosk tampilkan error "Kiosk belum dikonfigurasi" kalau tidak ada token
