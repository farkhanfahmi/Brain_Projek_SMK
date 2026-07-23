# T052 — Schema `kelas_id_wali` + UI Admin Jurnal: Assign Wali Kelas

## Depends on
T038 (schema dasar), T047 (RBAC admin_jurnal harus sudah ada), T050 (layout dashboard admin_jurnal harus sudah ada)

## Objective
Tambah kolom `kelas_id_wali` ke `users`, dan buat menu baru di Dashboard Admin Jurnal untuk assign 1 guru sebagai wali kelas 1 kelas.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `users`, `kelas`, `teachers`
- **Role:** `admin_jurnal` (yang assign), `guru` (yang jadi wali kelas)
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "🏫 Wali Kelas (Final — Fase 2, siap eksekusi)", baca lengkap sebelum mulai

## Spec Detail

### 1. Schema
Tambah kolom ke `users` (extend, bukan tabel baru — pola identik `guru_piket.kampus_id` dari Fase 1):
```
kelas_id_wali   Int?   // FK ke kelas, nullable
```
- **Constraint "1 kelas cuma 1 wali kelas aktif"**: ditegakkan di SERVICE LAYER (bukan DB constraint) — sebelum assign, cek dulu apakah `kelas_id_wali` itu sudah dipakai user lain, kalau ya tolak dengan pesan jelas ATAU tawarkan opsi "pindahkan dari guru X ke guru Y" (putuskan salah satu UX saat implementasi, yang penting tidak ada 2 user aktif dengan `kelas_id_wali` sama tanpa sepengetahuan admin)

### 2. API: extend `users` module (sudah ada dari T020, JANGAN buat module baru)

**PATCH `/users/:id/assign-wali-kelas`** — role `admin_jurnal` (tambahan akses baru — sebelumnya endpoint users cuma `super_admin`, task ini extend guard untuk endpoint SPESIFIK ini saja)
```json
{ "kelas_id": 15 }
```
- Body `{ "kelas_id": null }` untuk **melepas** status wali kelas (bukan endpoint DELETE terpisah)
- Validasi: `user.role` harus `guru` (bukan `guru_piket`/`kepsek`/dll — meski akun-akun itu juga `teacher_id`-linked, wali kelas HANYA untuk role dasar `guru`, konsisten dengan "Wali Kelas bukan role baru, extend akun guru")
- Validasi: kalau `kelas_id` sudah dipakai user lain sebagai wali kelas aktif → 409, response sertakan nama guru yang sedang menjabat
- Log ke `activity_log`, action `user.assign_wali_kelas`

**GET `/users?role=guru&kelas_id_wali=`** — extend filter existing `GET /users` (dari T020) untuk bisa filter siapa wali kelas kelas tertentu, atau list semua yang `kelas_id_wali IS NOT NULL`

### 3. UI: `/admin-jurnal/wali-kelas` (menu baru di sidebar admin_jurnal, extend T050)
- Tabel: Kelas | Wali Kelas (nama guru, atau "— Belum ditugaskan —") | Aksi
- List SEMUA kelas (dari `GET /kelas`, existing Fase 1), join manual di FE/BE ke `users.kelas_id_wali` untuk tampilkan siapa walinya
- Aksi per baris: **[Assign/Ubah]** → modal pilih guru dari dropdown (list guru dengan `role: guru`, autocomplete by nama) → simpan → `PATCH /users/:id/assign-wali-kelas`
- Aksi **[Lepas]** untuk kelas yang sudah ada walinya → konfirmasi dialog → kirim `kelas_id: null`

## JANGAN
- ❌ JANGAN buat tabel baru untuk relasi wali kelas — extend kolom `users.kelas_id_wali`, sesuai keputusan pola `guru_piket`-like
- ❌ JANGAN izinkan assign wali kelas ke akun dengan role selain `guru` (misal `guru_piket`, `kepsek`, `admin_jurnal`) — validasi ini WAJIB di backend, bukan cuma dropdown FE yang difilter
- ❌ JANGAN buat menu ini bisa diakses `super_admin` — tetap domain `admin_jurnal` sesuai batasan role (kelola jadwal, dan wali kelas adalah bagian dari itu)
- ❌ JANGAN buat sistem co-wali (banyak-ke-banyak) — 1 kelas 1 wali kelas, sesuai keputusan eksplisit

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` — tambah `kelas_id_wali` ke `users`
- **Buat:** migration Prisma
- **Modifikasi:** `apps/api/src/users/users.service.ts` — tambah method `assignWaliKelas`
- **Modifikasi:** `apps/api/src/users/users.controller.ts` — tambah endpoint, guard `@Roles('admin_jurnal')` khusus endpoint ini
- **Buat:** `apps/web/app/(admin-jurnal)/wali-kelas/page.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/wali-kelas/components/assign-modal.tsx`
- **Modifikasi:** `apps/web/app/(admin-jurnal)/layout.tsx` — tambah menu sidebar "Wali Kelas"

## Acceptance Criteria
- [ ] Assign guru A ke kelas X → `users.kelas_id_wali` guru A terisi `X`
- [ ] Assign guru B ke kelas X yang sudah punya wali (guru A) → 409, pesan sertakan nama guru A
- [ ] Assign kelas ke akun dengan role `guru_piket` → 403/400 ditolak
- [ ] Lepas status wali kelas → `kelas_id_wali` jadi `null`, tabel tampilkan "Belum ditugaskan"
- [ ] Role selain `admin_jurnal` akses endpoint/halaman ini → 403/redirect
- [ ] `user.assign_wali_kelas` tercatat di `activity_log`

## Handoff ke T053
T053 (menu Wali Kelas di dashboard guru) akan cek `req.user.kelas_id_wali` dari JWT — pastikan JWT payload guru di-refresh untuk include field ini (cek `apps/api/src/auth/` — mungkin perlu tambah field ke JWT payload generation kalau belum otomatis ikut dari `users` table).
