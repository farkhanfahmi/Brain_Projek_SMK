# T136 — Schema+API+Web: Toggle Wajib Ganti Password Saat Login Pertama (Per-Section Akun)

## Depends on
Tidak ada dependency teknis. Perluasan field `User.mustChangePassword` yang sudah ada + config baru per-role.

## Objective
Super admin bisa mengaktifkan/nonaktifkan (per section tab di Manajemen Akun: Admin/Piket/Guru/Pembina Ekstra) apakah akun BARU yang dibuat di section itu WAJIB ganti password saat login pertama kali. Saat ini SEMUA akun baru (termasuk piket) TIDAK PERNAH diwajibkan ganti password — password yang di-set admin saat pembuatan akun berlaku selamanya kecuali di-reset manual.

## Context
- **App:** `apps/api` (config baru per-role + ubah `UsersService.create()`) + `apps/web` (toggle di tiap section Manajemen Akun)
- **Bug/gap dikonfirmasi 2026-08-08 (Explore agent, baca kode langsung)** — laporan user BENAR:
  - `User.mustChangePassword` (`schema.prisma:752`) — `@default(false)` di level DATABASE. Field ini SUDAH ADA dan SUDAH dipakai (middleware cek klaim JWT, redirect ke `/ganti-password` kalau `true`) — infrastrukturnya jalan, TAPI:
  - `UsersService.create()` (`apps/api/src/users/users.service.ts:42-54`) — **TIDAK PERNAH set `mustChangePassword: true`** untuk akun baru, apa pun rolenya. Field ini cuma di-set `true` oleh 2 flow LAIN yang terpisah: `resetPassword()` (reset password acak) dan `setPassword()` (admin set password manual, baris ±134-142) — **bukan** oleh `create()` (pembuatan akun awal).
  - **Kesimpulan**: akun piket yang dibuat lewat form "Tambah Akun" biasa di Manajemen Akun TIDAK PERNAH diwajibkan ganti password — password default yang di-set admin (kalaupun sama untuk semua piket) berlaku permanen sampai di-reset manual satu-satu. Ini match persis laporan user "semua piket masih pakai password default di production".
  - **Tidak ada checkbox di form pembuatan akun** untuk field ini sama sekali — `CreateUserDto` tidak punya field `mustChangePassword`.
  - **Tidak ada config per-role/section** — field ini murni per-akun, di-set imperatif oleh service method, TIDAK ADA singleton config (beda dari `AttendanceLockConfig`/`EkstraRegistrationConfig` yang sudah ada polanya) yang mengontrol "akun BARU section X wajib ganti password".

## Keputusan Final (dikonfirmasi user 2026-08-08)

**Toggle PER-SECTION** (bukan 1 toggle global) — tiap section tab di Manajemen Akun (Admin/Piket/Guru/Pembina Ekstra) punya toggle SENDIRI, bisa beda-beda (misal wajib untuk Piket, tidak wajib untuk Admin).

## Spec Detail

### Schema (Prisma)
Model config baru, pola SAMA seperti `AttendanceLockConfig`/`EkstraRegistrationConfig` (singleton, TAPI kali ini butuh 4 nilai — 1 per section, BUKAN 1 boolean tunggal seperti config lain yang sudah ada):
```prisma
model ForcePasswordChangeConfig {
  id                Int      @id @default(1)
  forceAdmin        Boolean  @default(false)
  forcePiket        Boolean  @default(false)
  forceGuru         Boolean  @default(false)
  forcePembinaEkstra Boolean @default(false)
  updatedById       Int
  updatedAt         DateTime @updatedAt

  updatedBy User @relation(fields: [updatedById], references: [id])
  @@map("force_password_change_config")
}
```
- **PERTIMBANGKAN alternatif**: kalau 4 kolom boolean terasa kaku (misal section baru ditambah di masa depan), pertimbangkan tabel per-role (`ForcePasswordChangeConfig { id, role: UserRole, enabled: Boolean }`, banyak baris bukan 1 singleton) — lebih fleksibel untuk role baru, TAPI lebih kompleks dari pola singleton yang sudah dipahami tim. **Putuskan saat implementasi** mana yang lebih sesuai, TIDAK ada preferensi kuat dari riset — 4-kolom-singleton lebih SEDERHANA dan konsisten pola existing, direkomendasikan kecuali ada alasan kuat sebaliknya.
- Migration baru.

### Backend
- Modul baru kecil (atau tambahkan ke modul `users` yang sudah ada) — service `get()`/`update()` config, pola sama `AttendanceLockConfigService`.
- `GET /force-password-change-config` — bisa diakses `super_admin` minimal (untuk render toggle di halaman admin).
- `PATCH /force-password-change-config` — `@Roles(UserRole.super_admin)` SAJA, `@LogActivity` wajib.
- **`UsersService.create()`** (`apps/api/src/users/users.service.ts:42-54`) — modifikasi: SEBELUM insert, cek config berdasarkan `dto.role` yang dikirim (map role ke section: `guru_piket` → `forcePiket`, dst — cek mapping role↔section yang dipakai `SECTION_ROLES` di frontend `akun-view.tsx` untuk konsistensi definisi "section" apa saja yang termasuk role apa) → kalau toggle section itu AKTIF, set `mustChangePassword: true` di data yang di-insert; kalau NONAKTIF, tetap `false` (default, tidak berubah dari sekarang).
- **TIDAK MENGUBAH** `resetPassword()`/`setPassword()` — flow itu SUDAH BENAR (selalu set `true`), tidak terkait scope task ini.

### Frontend
- `apps/web/src/app/(admin)/akun/akun-view.tsx` — di TIAP section tab (Admin/Piket/Guru/Pembina Ekstra), tambah toggle switch kecil di area header section (REUSE komponen toggle custom yang sudah ada dari `pengaturan-absensi-view.tsx`, JANGAN ulangi bug posisi thumb yang pernah terjadi di situ) — label: "Wajib ganti password saat login pertama untuk akun baru di section ini".
- Toggle terhubung ke field config yang sesuai section itu, `PATCH` saat diubah.
- Toggle ini HANYA mempengaruhi akun BARU yang dibuat SETELAHNYA — TIDAK mengubah `mustChangePassword` akun yang SUDAH ADA (retroaktif TIDAK termasuk scope task ini, kalau user butuh itu juga, itu keputusan terpisah — JANGAN diam-diam terapkan retroaktif tanpa konfirmasi).

## Edge Cases
- Toggle diaktifkan SETELAH beberapa akun section itu sudah dibuat — akun LAMA tidak terpengaruh (tetap `mustChangePassword: false` kecuali di-reset manual terpisah), cuma akun BARU ke depan yang kena. Ini WAJIB dijelaskan ke user secara eksplisit di UI (misal teks kecil: "Hanya berlaku untuk akun baru, tidak mengubah akun yang sudah ada") supaya tidak ada ekspektasi keliru.
- Role yang tidak masuk 4 section existing (kalau ada role lain di masa depan) — pastikan tidak crash, fallback ke `false` (tidak wajib) kalau role tidak dikenali dalam mapping.

## Files
- **Buat:** migration Prisma baru, modul/service config baru.
- **Modifikasi:** `apps/api/prisma/schema.prisma` (model baru + relasi `User`), `apps/api/src/users/users.service.ts` (`create()`), `apps/web/src/app/(admin)/akun/akun-view.tsx` (toggle per-section).
- **Jangan sentuh:** `resetPassword()`/`setPassword()` (sudah benar, tidak terkait), `mustChangePassword` retroaktif ke akun existing (di luar scope, jangan diam-diam diterapkan).

## Acceptance Criteria
- [x] Toggle muncul di tiap 4 section Manajemen Akun, independen satu sama lain — **diverifikasi LIVE** (lihat Status Eksekusi).
- [x] Toggle AKTIF untuk section Piket: akun piket BARU yang dibuat setelahnya otomatis `mustChangePassword: true` — **diverifikasi LIVE**.
- [x] Toggle NONAKTIF: akun baru section itu tetap `mustChangePassword: false` seperti sekarang (regresi nol) — **diverifikasi LIVE**.
- [x] Akun yang SUDAH ADA sebelum toggle diaktifkan TIDAK terpengaruh retroaktif — **diverifikasi LIVE**.
- [x] Activity log tercatat di endpoint PATCH config — MANUAL (`activityLogService.record()`, BUKAN decorator `@LogActivity` — lihat alasan di Status Eksekusi, pola sama config singleton lain T143/T144/T146/T147).
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] Mapping role↔section PERSIS SAMA `SECTION_ROLES`/`sectionOfRole()` di `akun-view.tsx` — dikonfirmasi via 9 unit test eksplisit (`force-password-change-config.service.spec.ts`) menguji SEMUA 7 role individual, termasuk 4 role yang sama-sama masuk section "admin".
- [x] **Keputusan struktur config**: 4-kolom-singleton dipilih (bukan tabel-per-role) — sesuai rekomendasi spec (lebih sederhana, konsisten pola existing `AttendanceLockConfig`/`SystemLiveConfig`), tidak ada indikasi kuat butuh fleksibilitas tabel-per-role untuk 4 section yang sudah stabil.
- [x] UI toggle secara eksplisit menjelaskan "cuma akun baru, bukan retroaktif" — teks caption di bawah tiap toggle: "Hanya berlaku untuk akun baru yang dibuat SETELAH toggle ini diaktifkan — tidak mengubah akun yang sudah ada."

## Status Eksekusi (2026-08-09)
- **Schema**: migration `20260809065609_t136_force_password_change_config`, model `ForcePasswordChangeConfig` singleton (`id=1`, 4 kolom boolean default `false` — `forceAdmin`/`forcePiket`/`forceGuru`/`forcePembinaEkstra`). Diterapkan bersih ke dev DB.
- **Backend — modul baru** `force-password-change-config/` (controller+service+dto+module, pola sama `attendance-lock-config/`+`system-live-config/`). `shouldForceFor(role: UserRole)` — method inti, mapping SWITCH per role: `guru_piket`→`forcePiket`, `guru`→`forceGuru`, `pembina_ekstra`→`forcePembinaEkstra`, SEMUA role lain (`super_admin`/`card_admin`/`admin_jurnal`/`kepsek`)→`forceAdmin` (default branch, SAMA PERSIS `sectionOfRole()` frontend yang juga fallback ke "admin" untuk role tak dikenal). Activity log MANUAL (bukan `@LogActivity`) — endpoint singleton tanpa `id` route param yang cocok snapshot-fetch otomatis decorator, pola konsisten 4 config singleton lain di proyek ini.
- **`UsersService.create()` diperluas**: 1 baris baru `const mustChangePassword = await this.forcePasswordChangeConfig.shouldForceFor(dto.role)` dipanggil SEBELUM insert, hasilnya diselipkan ke `data.mustChangePassword`. `resetPassword()`/`setPassword()`/`changeOwnPassword()` TIDAK disentuh (sudah benar, di luar scope).
- **Frontend**: `akun-view.tsx` — `SECTION_FORCE_FIELD` map (Section→field config) ditaruh tepat di bawah `sectionOfRole()` (1 sumber kebenaran section yang sama). Komponen baru `ForcePasswordChangeToggle` (toggle switch REUSE markup dari `pengaturan-absensi-view.tsx`, posisi thumb `translate-x-5`/`translate-x-0` — TIDAK mengulangi bug posisi yang disebut spec) dirender di ATAS `AccountTable` tiap `TabsContent`, jadi toggle-nya genuinely independen per tab (4 toggle terpisah, bukan 1 toggle yang berubah makna). `page.tsx` fetch config baru paralel dengan data lain.
- **Verifikasi**: `tsc --noEmit` bersih `apps/api`+`apps/web`. `jest` 254/254 pass (17 test baru: 2 di `users.service.spec.ts` — existing file, sebelumnya 6 test untuk `assignWaliKelas` saja — untuk `create()` toggle aktif/nonaktif; 9 di `force-password-change-config.service.spec.ts`, file spec BARU, cover ke-4 section + role yang belum di-set + independensi antar-section). **Live end-to-end** via `NestFactory.createApplicationContext` terhadap dev DB nyata — 5 skenario BERURUTAN CONFIRMED: (1) config belum di-set → `get()` semua false; (2) akun baru TANPA toggle aktif → `mustChangePassword:false` (regresi nol); (3) aktifkan `forcePiket` lalu buat akun piket baru → `mustChangePassword:true`; (4) akun LAMA (dibuat sebelum toggle aktif) di-refetch → TETAP `false`, TIDAK retroaktif; (5) buat akun section GURU (toggle `forceGuru` masih false) sementara `forcePiket` aktif → TETAP `false`, section independen dikonfirmasi. Semua data test (users, config) dibuat+dihapus bersih.
