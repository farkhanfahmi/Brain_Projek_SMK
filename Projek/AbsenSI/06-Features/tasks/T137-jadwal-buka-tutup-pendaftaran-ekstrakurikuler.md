# T137 — Schema+API+Web: Jadwal Buka/Tutup Pendaftaran Ekstrakurikuler (Toggle + Rentang Tanggal/Jam)

## Depends on
Depends on T134 (`EkstraRegistrationConfig`, SUDAH SELESAI 2026-08-07) — task ini MEMPERLUAS model config yang sama, bukan model baru terpisah. Baca ulang implementasi T134 dulu.

## Objective
Halaman pendaftaran publik ekstrakurikuler (`/daftar-ekstra`) bisa diatur admin: AKTIF/NONAKTIF, dan opsional dibatasi jendela waktu tertentu (tanggal+jam mulai, tanggal+jam selesai) — di luar jendela itu atau saat dinonaktifkan, halaman menampilkan status "pendaftaran ditutup" alih-alih form pendaftaran.

## Context
- **App:** `apps/api` (perluas `EkstraRegistrationConfig` + guard di endpoint submit) + `apps/web` (toggle+date-time picker di admin, gating di halaman publik)
- **Riset 2026-08-08 (Explore agent, baca kode langsung)** — konfirmasi kondisi SEBELUM task ini:
  - `EkstraRegistrationConfig` (`schema.prisma:352-361`, hasil T134) — SAAT INI cuma 1 field: `lockPindahEkstra` (boolean). **TIDAK ADA field jendela waktu, TIDAK ADA field "registration open/closed" terpisah** — field itu semantiknya SEMPIT (cuma soal larangan PINDAH ekstra untuk siswa yang SUDAH terdaftar), BUKAN "boleh daftar sama sekali atau tidak". Task ini menambah konsep BARU yang terpisah, bukan reuse `lockPindahEkstra`.
  - `/daftar-ekstra` (`apps/web/src/app/daftar-ekstra/page.tsx`) — SAAT INI halaman statis yang SELALU menampilkan form, TIDAK ADA pengecekan config apa pun di level render halaman. `EkstraPublikService.submit()` juga TANPA guard apa pun (endpoint publik, siapa saja bisa akses) — TIDAK ADA gating "pendaftaran ditutup" di titik mana pun saat ini.
  - **Pola precedent yang ADA di codebase** (`Kiosk.isActive`, `TvSession`) — flag boolean SEDERHANA tanpa expiry berbasis waktu. **TAPI user secara eksplisit minta rentang tanggal/jam** — jadi task ini BUKAN reuse pola `isActive` sederhana, ini pola BARU (date-range) yang belum ada precedent-nya di proyek ini.

## Keputusan Final (dikonfirmasi user 2026-08-08)

1. **URL TETAP** — `/daftar-ekstra` tidak berubah, TIDAK PERLU token/link unik yang di-generate ulang. Halaman yang sama, cuma isinya (form vs pesan "ditutup") tergantung status config.
2. **2 mekanisme independen**: toggle AKTIF/NONAKTIF (on/off manual) DAN rentang tanggal/jam (jendela waktu otomatis) — keduanya BISA dipakai bersamaan atau salah satu saja (misal toggle AKTIF tapi tanpa rentang tanggal = selalu buka sampai dimatikan manual; ATAU toggle AKTIF + rentang tanggal = otomatis tertutup begitu lewat tanggal selesai, tanpa perlu admin matikan manual). **Klarifikasi logic gabungan ini WAJIB saat implementasi** (lihat Spec Detail).

## Spec Detail

### Schema (Prisma)
Perluas `EkstraRegistrationConfig` (SATU model yang sama dengan T134, TAMBAH field, jangan bikin model baru terpisah):
```prisma
model EkstraRegistrationConfig {
  id                Int       @id @default(1)
  lockPindahEkstra  Boolean   @default(false)  // sudah ada (T134)
  isOpen            Boolean   @default(false)   // BARU — toggle manual on/off
  bukaMulai         DateTime? @map("buka_mulai") // BARU — opsional, null = tidak ada batas mulai
  bukaSampai        DateTime? @map("buka_sampai") // BARU — opsional, null = tidak ada batas akhir
  updatedById       Int
  updatedAt         DateTime  @updatedAt

  updatedBy User @relation(fields: [updatedById], references: [id])
  @@map("ekstra_registration_config")
}
```
- Migration baru (ALTER TABLE tambah kolom, bukan CREATE TABLE baru).

### Logic Gabungan Toggle + Jendela Waktu (WAJIB didefinisikan jelas, jangan ambigu)
Pendaftaran dianggap **BUKA** (siswa bisa submit) HANYA kalau SEMUA kondisi berikut benar:
1. `isOpen === true` (toggle manual aktif) — DAN
2. Kalau `bukaMulai` diisi (tidak null) → waktu sekarang >= `bukaMulai` — DAN
3. Kalau `bukaSampai` diisi (tidak null) → waktu sekarang <= `bukaSampai`.

Kalau `bukaMulai`/`bukaSampai` KEDUANYA null (admin tidak set rentang) → pendaftaran buka SELAMA `isOpen === true`, tanpa batas waktu otomatis (cocok kasus "buka terus sampai saya matikan manual").

### Backend
- Service `EkstraRegistrationConfigService` (dari T134, DIPERLUAS) — tambah method `isPendaftaranBuka()` yang menghitung logic gabungan di atas (bukan cuma baca field mentah — method ini yang JADI SUMBER KEBENARAN tunggal dipakai backend DAN diexpose ke frontend lewat endpoint GET, supaya logic gabungan tidak diduplikasi di 2 tempat berbeda yang berisiko tidak sinkron).
- `PATCH /ekstra-registration-config` (endpoint T134 yang sudah ada) — DTO diperluas terima `isOpen?`, `bukaMulai?`, `bukaSampai?` opsional selain `lockPindahEkstra` yang sudah ada.
- `GET /ekstra-registration-config` (endpoint T134 yang sudah ada) — response diperluas, TAMBAH field terhitung `isPendaftaranBukaSaatIni: boolean` (hasil `isPendaftaranBuka()`) supaya frontend TIDAK PERLU replikasi logic tanggal/waktu sendiri — cukup baca boolean jadi dari backend.
- **`EkstraPublikService.submit()`** — tambah pengecekan `isPendaftaranBuka()` SEBELUM proses simpan (kecuali untuk siswa yang submit ULANG dengan ekstra SAMA seperti pola T134 yang sudah ada — putuskan apakah pengecekan jendela waktu ini berlaku untuk SEMUA submit termasuk siswa lama, atau HANYA pendaftar BARU pertama kali; REKOMENDASI: berlaku untuk SEMUA submit termasuk siswa existing, karena ini soal "kapan pendaftaran dibuka" bukan soal "siapa boleh pindah" — beda scope dari `lockPindahEkstra`) — kalau tertutup, TOLAK dengan pesan jelas.

### Frontend
- Admin: halaman `ekstra-kurikuler` (lokasi toggle T134 yang sudah ada) — tambah date-time picker untuk `bukaMulai`/`bukaSampai` (opsional, bisa dikosongkan), berdampingan dengan toggle `isOpen` yang baru (BEDA dari toggle `lockPindahEkstra` T134 — 2 toggle terpisah dengan label jelas berbeda, JANGAN sampai admin bingung mana yang mana).
- Halaman publik `/daftar-ekstra` — SEKARANG jadi server component yang CEK `isPendaftaranBukaSaatIni` SEBELUM render form — kalau `false`, tampilkan halaman/pesan "Pendaftaran ekstrakurikuler belum/sudah ditutup" (bedakan pesan "belum dibuka" vs "sudah ditutup" kalau memungkinkan dari data `bukaMulai`/`bukaSampai`, supaya siswa tahu KAPAN harus kembali kalau belum dibuka) alih-alih form.

## Edge Cases
- `bukaMulai` di masa depan (belum waktunya) → halaman tampilkan "Pendaftaran belum dibuka, mulai [tanggal]" — bukan sekadar "ditutup" generik.
- `bukaSampai` sudah lewat → halaman tampilkan "Pendaftaran sudah ditutup" — beda pesan dari "belum dibuka".
- Admin ubah `isOpen`/rentang waktu SAAT ADA siswa sedang mengisi form (race condition kecil) → validasi tetap di titik SUBMIT (backend), bukan cuma render awal — siswa yang sudah buka form lalu telat submit setelah tertutup TETAP ditolak backend dengan pesan jelas, bukan silent fail.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (perluas `EkstraRegistrationConfig`), service+controller `ekstra-registration-config` (dari T134), `apps/api/src/ekstra-publik/ekstra-publik.service.ts` (`submit()`), `apps/web/src/app/(admin)/ekstra-kurikuler/ekstra-kurikuler-view.tsx` (UI toggle+date-time baru), `apps/web/src/app/daftar-ekstra/page.tsx` (gating render).
- **Jangan sentuh:** `lockPindahEkstra` (field T134 existing, TIDAK diubah maknanya), `pindahkanEkstra()` (jalur admin, di luar scope task ini juga seperti T134).

## Acceptance Criteria
- [x] Admin bisa toggle `isOpen` + set opsional `bukaMulai`/`bukaSampai` di halaman `ekstra-kurikuler`.
- [x] `isOpen: false` → halaman `/daftar-ekstra` tampilkan pesan tertutup, form tidak muncul.
- [x] `isOpen: true` tanpa rentang tanggal → pendaftaran buka terus sampai dimatikan manual.
- [x] `isOpen: true` + rentang tanggal → otomatis tertutup begitu lewat `bukaSampai`, tanpa perlu admin matikan manual.
- [x] Waktu sebelum `bukaMulai` → pesan "belum dibuka" (bukan pesan generik "ditutup").
- [x] Backend TETAP menolak submit di luar jendela waktu meski frontend somehow mengizinkan submit (defense in depth).
- [x] `lockPindahEkstra` (T134) tetap berfungsi independen, regresi nol.
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] **WAJIB** definisikan logic gabungan `isOpen` + `bukaMulai`/`bukaSampai` di SATU method backend (`isPendaftaranBuka()`), jangan duplikasi logic tanggal di frontend — frontend cukup baca hasil boolean jadi dari API.
- [x] Putuskan/konfirmasi ke user: apakah jendela waktu ini berlaku untuk SEMUA submit (termasuk siswa existing yang mau submit ulang ekstra sama) atau HANYA pendaftar baru — rekomendasi: semua submit, tapi konfirmasi kalau ambigu.

## Status Eksekusi (2026-08-09)

**Selesai.** Backend + frontend + test + verifikasi live, semua hijau.

- **Schema**: `EkstraRegistrationConfig` diperluas (`isOpen`, `bukaMulai`, `bukaSampai`) via migration `20260809073254_t137_ekstra_registration_config_jendela_waktu` (ALTER TABLE, tanpa data loss — tabel kosong di dev).
- **Backend**: `EkstraRegistrationConfigService` — `isPendaftaranBuka()` + `computeIsPendaftaranBuka()` (private, SATU implementasi dipakai `isPendaftaranBuka()` DAN `toResponse()`, tidak diduplikasi). `EkstraPublikService.submit()` digerbangi PALING AWAL sebelum query lain. Endpoint publik baru `GET /ekstra-publik/pendaftaran-status` (hanya expose `isPendaftaranBukaSaatIni`+`bukaMulai`+`bukaSampai`, TIDAK `lockPindahEkstra`).
- **Frontend admin**: `JadwalPendaftaranConfig` component baru di `ekstra-kurikuler-view.tsx` — toggle + 2 date-time picker, label eksplisit beda dari `LockPindahEkstraToggle`.
- **Frontend publik**: `/daftar-ekstra/page.tsx` jadi async server component, fetch status sebelum render, 3 pesan beda (belum dibuka / sudah ditutup / tertutup generik).
- **Test**: `ekstra-registration-config.service.spec.ts` (9 test, semua branch `computeIsPendaftaranBuka` + update nullable field) + `ekstra-publik.service.spec.ts` (2 test, gating `submit()`). Full suite: 265/265 pass, tsc bersih di `apps/api` dan `apps/web`.
- **Verifikasi live**: script sementara (`apps/api/scripts/verify-t137-temp.ts`, dihapus setelah run) terhadap DB dev port 3307 — 6/6 skenario PASS (closed/open/before-mulai/after-sampai/within-window/DB round-trip), config direset ke `isOpen: false` di akhir (state aman, tidak meninggalkan data uji).
- **Tidak disentuh** (sesuai spec): `pindahkanEkstra()`, semantik `lockPindahEkstra` T134.
