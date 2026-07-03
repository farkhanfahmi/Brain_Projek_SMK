# T020 — Users Module: Manajemen Akun

## Depends on
T003 (auth), T005 (teachers harus ada untuk link teacher_id), T004 (kampus harus ada untuk guru_piket)

## Objective
Buat API dan UI untuk super_admin membuat, mengelola, dan reset password akun pengguna sistem.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `users`
- **Role akses:** `super_admin`
- **ADR:** ADR-008 (role generik)

## Spec Detail

### API Endpoints:

**GET `/users`** — list semua akun
- Include: nama (dari teacher terkait kalau ada), role, status, kampus (untuk guru_piket)
- Filter: `role?`, `status?`

**POST `/users`** — buat akun baru
```json
{
  "username": "piket2",
  "password": "password-awal",
  "role": "guru_piket",
  "teacher_id": "xxx",   // optional — untuk guru/kepsek/guru_piket yang merupakan guru
  "kampus_id": "xxx"     // required kalau role = guru_piket
}
```
- Validasi: `username` unik, kalau role `guru_piket` maka `kampus_id` wajib

**PATCH `/users/:id`** — update akun
- Bisa update: `username`, `role`, `status` (aktif/nonaktif), `kampus_id`
- Tidak bisa update: password via endpoint ini (ada endpoint khusus)

**POST `/users/:id/reset-password`** — reset password
- Body: `{ new_password: string }`
- Hash password baru dengan bcrypt, update ke DB
- Response: 200 OK — password baru dikomunikasikan admin ke user secara manual (tidak ada email/WA)
- Log ke `activity_log` dengan action `user.reset_password` (tanpa menyimpan password di snapshot)

**PATCH `/users/:id/deactivate`** — nonaktifkan akun
- Set `status: nonaktif`
- Jika user sedang punya active JWT: token tersebut tetap valid sampai expire (15 menit), karena check `status` dilakukan saat login bukan saat setiap request. **Ini acceptable trade-off** — dicatat sebagai known limitation.

### Admin UI (`/admin/akun`):

Tabel: Username | Role | Nama (guru terkait) | Kampus | Status | Aksi

Aksi per baris:
- **[Edit]** → modal form update username/role/status/kampus
- **[Reset Password]** → modal: input password baru + konfirmasi → submit
- **[Nonaktifkan]** → konfirmasi dialog

Form create akun baru (tombol di atas tabel):
- Username, password awal, role (dropdown), teacher (autocomplete — untuk role guru/kepsek/guru_piket), kampus (dropdown — wajib untuk guru_piket)

## JANGAN
- ❌ JANGAN buat self-registration — akun hanya dibuat oleh super_admin
- ❌ JANGAN kirim password via email/WA otomatis — admin sampaikan manual (keputusan Fase 1)
- ❌ JANGAN simpan password plaintext di log atau snapshot activity_log
- ❌ JANGAN buat role `super_admin` bisa diassign via UI ke semua orang sembarangan — pertimbangkan hanya bisa create `super_admin` kalau sudah ada minimal 1 `super_admin` yang login (chicken-and-egg dijaga via seed data)
- ❌ JANGAN buat endpoint DELETE user — hanya deactivate

## Files
- **Buat:** `apps/api/src/users/users.module.ts`
- **Buat:** `apps/api/src/users/users.service.ts`
- **Buat:** `apps/api/src/users/users.controller.ts`
- **Buat:** `apps/web/app/(admin)/akun/page.tsx`

## Acceptance Criteria
- [ ] Buat akun `guru_piket` tanpa `kampus_id` → error "kampus_id wajib untuk guru_piket"
- [ ] Reset password → bisa login dengan password baru
- [ ] Nonaktifkan akun → login dengan akun itu return 401
- [ ] `user.create` dan `user.reset_password` tercatat di `activity_log`
- [ ] `activity_log` untuk `reset_password` tidak mengandung password di snapshot
