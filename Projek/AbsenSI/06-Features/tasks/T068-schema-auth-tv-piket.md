# T068 — Schema & Auth: Token TV Piket (Tanpa Expiry, Revocable)

## Depends on
T038-T054 (data `teaching_sessions`/`teacher_permits` harus sudah ada — sudah terpenuhi di working tree)

## Objective
Buat mekanisme token khusus untuk TV Piket — tanpa masa berlaku (`exp`), tapi bisa dicabut manual oleh `super_admin` kapan saja. Berbeda dari JWT access/refresh token biasa yang selalu punya expiry, dan berbeda dari `kiosks.deviceToken` (yang scope-nya tap RFID, bukan dashboard read-only).

## Context
- **App:** `apps/api`
- **Ref:** `Projek/AbsenSI/06-Features/tv-piket.md` — bagian "Auth" dan "Implikasi Skema" (final, 2026-07-22)
- **Pola referensi:** `apps/api/src/auth/guards/kiosk.guard.ts` + `kiosks.deviceToken` (ADR-021) — task ini ADAPTASI pola yang sama, bukan reinvent dari nol

## Spec Detail

### Tabel baru: `tv_sessions`
```prisma
model TvSession {
  id         Int       @id @default(autoincrement())
  kampusId   Int       @map("kampus_id")
  token      String    @unique @db.VarChar(100)
  isActive   Boolean   @default(true) @map("is_active")
  revokedAt  DateTime? @map("revoked_at")
  revokedById Int?     @map("revoked_by")
  createdById Int      @map("created_by")
  createdAt  DateTime  @default(now()) @map("created_at")

  kampus     Kampus @relation(fields: [kampusId], references: [id])
  revokedBy  User?  @relation("TvSessionRevokedBy", fields: [revokedById], references: [id])
  createdBy  User   @relation("TvSessionCreatedBy", fields: [createdById], references: [id])

  @@map("tv_sessions")
}
```
- `token` di-generate pakai `nanoid(32)` atau setara — pola SAMA seperti `kiosks.deviceToken` (ADR-021), random 256-bit
- **TIDAK ADA kolom `expiresAt`** — ini yang membedakan dari JWT/refresh token biasa. Validitas token murni ditentukan oleh `isActive`/`revokedAt`, bukan waktu

### `TvPiketGuard` (baru, adaptasi `KioskGuard`)
- Validasi token dari query string (`?token=...`, dibaca sekali saat load halaman TV, disimpan di localStorage browser TV — pola SAMA seperti kiosk `?device=TOKEN` di ADR-021) ATAU header `Authorization: Bearer` (pilih salah satu, konsisten dengan cara `apps/web` biasa fetch API — kemungkinan besar header lebih konsisten karena ini route Next.js biasa bukan kiosk hardware)
- Query `tv_sessions` by `token`, cek `isActive: true` DAN `revokedAt IS NULL` — tolak 401 kalau tidak valid/sudah di-revoke
- `kampusId` dari `tv_sessions` yang cocok dipakai untuk scope semua query TV Piket (T069) — TV hanya bisa akses data kampus yang token-nya terdaftar, TIDAK BISA akses kampus lain meski tahu ID-nya

### API Admin (role `super_admin` — pengelolaan token TV adalah domain admin pusat, BUKAN admin_jurnal atau guru_piket)
**POST `/tv-sessions`** — generate token baru untuk 1 kampus
```json
{ "kampusId": 1 }
```
Response: `{ "id": 5, "token": "xxxxx...", "kampusId": 1 }` — token HANYA ditampilkan sekali saat generate (mirip kiosk device token QR), tidak pernah ditampilkan ulang di list

**GET `/tv-sessions`** — list semua token TV (tanpa menampilkan token mentahnya di list, hanya metadata: kampus, status aktif/revoked, kapan dibuat)

**POST `/tv-sessions/:id/revoke`** — cabut token
- Set `revokedAt = now()`, `revokedById = req.user.id`, `isActive = false`
- Log ke `activity_log` (WAJIB pakai `@LogActivity`, sesuai aturan T067 — jangan sampai task baru ini juga lupa)

## JANGAN
- ❌ JANGAN tambah kolom `expiresAt`/`exp` — keputusan eksplisit "tanpa masa berlaku", jangan diam-diam menambahkan expiry "untuk jaga-jaga"
- ❌ JANGAN reuse `KioskGuard`/`kiosks.deviceToken` apa adanya — scope-nya beda (kiosk untuk tap RFID hardware, TV Piket untuk dashboard read-only browser), buat guard & tabel terpisah meski polanya mirip
- ❌ JANGAN izinkan `POST /tv-sessions` atau `.../revoke` diakses role selain `super_admin` — token TV Piket adalah kontrol keamanan, bukan wewenang `guru_piket`/`admin_jurnal`
- ❌ JANGAN tampilkan token mentah di endpoint `GET /tv-sessions` (list) — hanya saat `POST` pertama kali generate, sesuai prinsip "tampilkan sekali" yang sudah dipakai pola kiosk
- ❌ JANGAN lupa `@LogActivity` di endpoint create/revoke — ini persis kasus yang diaudit T067, jangan mengulang kelalaian yang sama di task baru

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` — tambah model `TvSession`
- **Buat:** migration Prisma
- **Buat:** `apps/api/src/auth/guards/tv-piket.guard.ts`
- **Buat:** `apps/api/src/tv-sessions/tv-sessions.module.ts`, `.service.ts`, `.controller.ts`

## Acceptance Criteria
- [ ] `POST /tv-sessions` menghasilkan token, response HANYA muncul sekali saat create
- [ ] Request ke endpoint TV Piket (T069) dengan token valid → berhasil, di-scope ke `kampusId` token tsb
- [ ] Request dengan token yang sudah di-`revoke` → 401
- [ ] Token TIDAK PERNAH kedaluwarsa sendiri berdasarkan waktu — token yang dibuat kapan pun tetap valid sampai di-revoke manual
- [ ] `POST /tv-sessions` dan `.../revoke` dari role selain `super_admin` → 403
- [ ] `action: tv_session.create`/`tv_session.revoke` tercatat di `activity_log`

## Handoff ke T069
T069 (endpoint data TV Piket) memakai `TvPiketGuard` untuk proteksi, dan `kampusId` dari guard untuk scope semua query.
