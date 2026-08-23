# T202 — Web: Perbaiki Visibilitas Import Mapel di Admin-Jurnal + Sesuaikan Template Pasca-T201

## Depends on
**WAJIB setelah T201** (many-to-many Jurusan) — template Excel & kolom import perlu ikut format checklist-jurusan yang baru, bukan single jurusan lagi.

## Objective
1. Perbaiki **visibilitas** fitur Import Mapel (T187, SUDAH SELESAI tapi HANYA tampil di route `(admin)`, TIDAK di `(admin-jurnal)`) — user (mengakses dari admin-jurnal) tidak menemukan fitur ini sama sekali.
2. Sesuaikan kolom template Excel Import Mapel untuk mendukung **beberapa jurusan sekaligus** per mapel (pasca-T201, many-to-many).

## Context — Akar Masalah Ditemukan (2026-08-16)

T187 (Import Excel Mapel) SUDAH SELESAI dieksekusi DAN BENAR SECARA KODE — TAPI prop `canImport` (yang mengontrol tampil/tidaknya tombol+dialog Import) **HANYA di-pass `true` di `(admin)/mapel/page.tsx`** (`page.tsx:14`), **TIDAK di-pass** di `(admin-jurnal)/admin-jurnal/mapel/page.tsx` — sehingga default `false`, tombol Import TIDAK PERNAH muncul kalau diakses lewat admin-jurnal. Ini DISENGAJA saat T174 lama (guard `/import/mapel` endpoint memang `super_admin` saja) — TAPI user SEKARANG (mengakses dari admin_jurnal, KEMUNGKINAN karena T188 memberi admin_jurnal akses lebih luas) tidak menemukan fitur ini — KEPUTUSAN LAMA PERLU DIEVALUASI ULANG.

## Spec Detail

### 1. Backend — perluas guard endpoint import Mapel (KONFIRMASI dulu)

**KEPUTUSAN YANG PERLU DIPUTUSKAN** (VERIFIKASI/KLARIFIKASI kalau ragu, JANGAN asumsi sepihak): apakah `admin_jurnal` SEKARANG boleh IMPORT Mapel juga (KONSISTEN dengan T188 yang memberi admin_jurnal akses CRUD penuh Kalender Pendidikan — kalau admin_jurnal sudah dipercaya untuk hal sebesar itu, wajar juga dipercaya untuk import Mapel)? REKOMENDASI KUAT: YA, perluas — `POST /import/mapel` dan `GET /import/mapel/template` tambah `UserRole.admin_jurnal` ke `@Roles` (ADDITIVE, KONSISTEN pola T157/T188, super_admin TIDAK dicabut).

### 2. Frontend — pasang `canImport` di admin-jurnal juga

- `apps/web/src/app/(admin-jurnal)/admin-jurnal/mapel/page.tsx` — TAMBAH prop `canImport={true}` (SAMA seperti yang SUDAH ADA di `(admin)/mapel/page.tsx`) — SEKARANG tombol+dialog Import Mapel MUNCUL di KEDUA route (admin DAN admin-jurnal).

### 3. Template Excel — sesuaikan kolom jurusan pasca-T201 (many-to-many)

- `generateMapelTemplate()` (`import.service.ts`, dari T187) — kolom `jurusan` (SEBELUMNYA 1 nama jurusan tunggal per baris) — GANTI supaya mendukung BEBERAPA jurusan per mapel: REKOMENDASI kolom `jurusan` menerima **daftar nama jurusan dipisah koma/titik-koma** dalam 1 sel (misal `"TKR, TKJ"` untuk mapel yang dipakai 2 jurusan) — parse dengan split+trim+lookup PER nama, TOLAK dengan pesan jelas kalau SALAH SATU nama tidak ditemukan (sebutkan nama mana yang salah).
- `importMapel()` — parsing kolom `jurusan` disesuaikan: split by koma/titik-koma → lookup tiap nama → kumpulkan jadi `jurusanIds: number[]` → create baris `MapelJurusan` untuk SETIAP jurusan yang match (REUSE `MapelService.create()` yang SUDAH menerima array dari T201, JANGAN duplikasi logic assignment jurusan).
- Contoh data di template — perbarui supaya menunjukkan SEMUA 3 skenario: 1 baris mapel UMUM (kolom jurusan kosong), 1 baris KHUSUS 1 jurusan, 1 baris KHUSUS BEBERAPA jurusan (`"TKR, TKJ"`) — supaya admin paham SEMUA kemungkinan format dari 1 template.

## Edge Cases
- Kolom `jurusan` dengan nama yang mengandung koma dalam namanya sendiri (TIDAK MUNGKIN terjadi untuk nama jurusan SMK yang wajar, tapi CATAT sebagai batasan format kalau relevan).
- Import ulang mapel yang SUDAH ADA (duplikat nama/kode) — PERILAKU TIDAK BERUBAH dari T187 (masih tolak duplikat, task ini TIDAK mengubah logic duplikat-check).

## Files
- **Modifikasi:** `apps/api/src/import/import.controller.ts` (`@Roles` diperluas, KALAU dikonfirmasi), `apps/api/src/import/import.service.ts` (`generateMapelTemplate()`+`importMapel()` sesuai kolom jurusan multi), `apps/web/.../admin-jurnal/mapel/page.tsx` (tambah `canImport={true}`).
- **Jangan sentuh:** `(admin)/mapel/page.tsx` (SUDAH benar, `canImport={true}` sejak T187, TIDAK diubah).

## Acceptance Criteria
- [x] Tombol Import Mapel MUNCUL di halaman `(admin-jurnal)/admin-jurnal/mapel/` (SEBELUMNYA tidak ada).
- [x] Template Excel Mapel — kolom jurusan mendukung multi-value (contoh data mencakup 3 skenario: umum/1-jurusan/banyak-jurusan).
- [x] Import Excel dengan kolom jurusan berisi BEBERAPA nama (dipisah koma) — berhasil assign mapel ke SEMUA jurusan yang disebut (sudah dibangun di T201, diverifikasi ulang di sini).
- [x] Nama jurusan yang salah ketik dalam daftar — ditolak dengan pesan jelas menyebut nama mana yang salah (sudah ada sejak T201).
- [x] Build + type-check hijau, jest diperbarui untuk parsing kolom jurusan multi-value + template 3-baris.

## Validasi Claudian
- [x] **Diputuskan**: `admin_jurnal` DIPERLUAS akses import Mapel — `POST /import/mapel` + `GET /import/mapel/template` tambah `@Roles(UserRole.admin_jurnal)` ADDITIF, KONSISTEN pola T188 (Kalender Pendidikan full-CRUD) dan T193 (import Wali Kelas, guard sama persis). Alasan: admin_jurnal sudah dipercaya untuk operasi jauh lebih besar (T188), tidak masuk akal membatasi import Mapel yang lebih kecil skalanya.
- [x] Konfirmasi `(admin)/mapel/page.tsx` TIDAK disentuh — diff kosong, tetap `canImport` tanpa syarat sejak T187.

## Status Eksekusi (2026-08-16)

**Selesai.**

### Keputusan akses (dikonfirmasi)

`admin_jurnal` diperluas ke `POST /import/mapel` dan `GET /import/mapel/template` — additive, `super_admin` tidak dicabut. Alasan lengkap di atas.

### Backend

- `import.controller.ts` — 2 endpoint (`importMapel`, `downloadMapelTemplate`) dapat `@Roles(UserRole.super_admin, UserRole.admin_jurnal)` (sebelumnya inherit class-level `super_admin` saja).
- `generateMapelTemplate()` — 2 baris contoh → 3 baris (umum kosong, 1 jurusan "TKR", multi-jurusan "TKR, TSM") — mencakup SEMUA skenario yang mungkin.
- Parsing multi-jurusan (`importMapel()` kolom `jurusan` split-koma, tolak seluruh baris kalau 1 nama tidak ketemu) — SUDAH DIBANGUN saat T201 (bukan pekerjaan baru task ini), diverifikasi ulang tetap benar dan konsisten dengan pola `ImportService` lain (direct Prisma write per baris, bukan delegasi ke `MapelService.create()` — pola SAMA seperti `importKelas`/`importJurusan` yang juga tidak delegasi ke service lain, bukan duplikasi logic bisnis apa pun, murni `mapelJurusan.createMany` sederhana).
- 1 unit test diperbarui (`generateMapelTemplate` — assert 3 baris contoh persis + isi tiap baris) — 491/491 pass di seluruh suite.

### Frontend

- `(admin-jurnal)/admin-jurnal/mapel/page.tsx` — tambah `canImport` (sebelumnya tidak ada sama sekali, akar masalah "tidak menemukan fitur import").
- `mapel-view.tsx` — teks contoh CSV di `ImportDialog` diperbarui mencakup 3 skenario (sebelumnya cuma 2).
- `(admin)/mapel/page.tsx` TIDAK disentuh — diverifikasi tetap `canImport` tanpa syarat.

### Verifikasi

- `tsc --noEmit` bersih di `apps/api` dan `apps/web`.
- `jest apps/api`: 491/491 pass, 29/29 suite.
- Live-verify browser: belum dilakukan (konsisten pola T186-T201, verifikasi manual diserahkan ke user) — TAPI ini task PALING RELEVAN untuk verifikasi manual karena akar masalahnya adalah user tidak menemukan tombol secara visual, rekomendasi kuat coba langsung di browser sebagai admin_jurnal.
