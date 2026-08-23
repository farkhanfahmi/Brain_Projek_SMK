# T206 — API: Service CRUD Opsi Jadwal + Alokasi Waktu (Backend Inti)

> ⚠️ **KOREKSI 2026-08-17 (saat eksekusi T209)**: spesifikasi asli di bawah ini menyebut
> `JadwalSlot` mendukung "team-teaching" dengan `teacherIds: number[]` (banyak guru per
> slot). Ini SALAH — user mengonfirmasi ulang: `MapelGuru` many-to-many HANYA
> pengelompokan/filter dropdown "guru relevan utk mapel ini" saat admin input jadwal,
> BUKAN team-teaching sungguhan. **1 kelas + 1 mapel + 1 jam SELALU 1 guru.** Kode SUDAH
> DIPERBAIKI saat eksekusi T209 (`teacherIds[]` → `teacherId` tunggal, `JadwalSlotGuru`
> tambah `@@unique([jadwalSlotId])`) — bagian "Spec Detail" di bawah TIDAK diedit ulang
> (dibiarkan sebagai arsip keputusan lama), rujuk task T209 utk detail perbaikan.

## Depends on
**WAJIB SETELAH T203, T204** (butuh semua model schema baru sudah ada). Bagian dari rangkaian T203-T215.

## Objective
Bangun service+controller backend untuk: (1) CRUD Alokasi Waktu (Senin-Sabtu, menggantikan `JamPelajaranService` lama), (2) CRUD Opsi Jadwal (mode permanen, toggle aktif independen dari Semester/Tahun Ajaran induk), (3) CRUD `JadwalSlot` dengan validasi lengkap (constraint mode-normal yang tidak tercakup DB, resolusi jam wall-clock, guru harus terdaftar di `MapelGuru`).

## Spec Detail

### 1. Modul `AlokasiWaktuService` (menggantikan `JamPelajaranService` konsep lama, TAPI JamPelajaranService LAMA TIDAK dihapus sampai T215)

- `findAll()` — list Alokasi Waktu (ringkas, tanpa slot detail).
- `findOne(id)` — detail lengkap dengan semua slot, grouped by hari.
- `create(dto, actorId)` — buat Alokasi Waktu + slot Senin-Sabtu (Sabtu OPSIONAL, array slot kosong untuk hari itu valid) dalam 1 `$transaction`.
- `update(id, dto, actorId)` — REPLACE SELURUH slot (hapus semua lama, insert baru — KONSISTEN pola `JamPelajaranService.update()` lama). **VALIDASI TAMBAHAN WAJIB**: kalau Alokasi Waktu ini SEDANG DIRUJUK oleh `OpsiJadwal` manapun yang punya `JadwalSlot` (ada data jadwal terisi menggunakan jam-ke dari alokasi ini) — TOLAK update yang MENGHAPUS/MENGUBAH `jamKe` yang MASIH DIPAKAI `JadwalSlot` manapun (pesan error JELAS: "Jam Ke X hari Y masih dipakai N jadwal, hapus/pindahkan jadwal itu dulu sebelum ubah alokasi waktu ini" — REFERENSI pola pesan error actionable, KONSISTEN aturan CLAUDE.md "pesan error sesuai kondisi").
- `delete(id)` — TOLAK kalau `AlokasiWaktu` ini DIRUJUK `OpsiJadwal` manapun (`opsiJadwal.length > 0`), pesan jelas sebutkan nama Opsi Jadwal yang merujuknya.

### 2. Modul `OpsiJadwalService`

- `findAll(semesterId?)` — list Opsi Jadwal, opsional filter semester.
- `findOne(id)` — detail + cakupan tingkat + status aktif.
- `create(dto, actorId)` — buat Opsi Jadwal BARU: `nama`, `semesterId`, `alokasiWaktuId`, `mode` (PERMANEN, tidak bisa diubah update() setelah ini), `tingkatScopes` (opsional, kosong = semua tingkat). `isActive` default `false` — TIDAK PERNAH otomatis aktif saat create (admin WAJIB aktifkan eksplisit via `activate()` terpisah).
- `update(id, dto, actorId)` — HANYA boleh ubah `nama`, `tingkatScopes` — **TOLAK KERAS kalau `dto` mengandung field `mode`** (pesan jelas: "Mode Opsi Jadwal tidak bisa diubah setelah dibuat — buat Opsi Jadwal baru dengan mode yang benar").
- `activate(id, actorId)` — set `isActive: true` UNTUK OPSI INI — **TIDAK OTOMATIS menonaktifkan Opsi Jadwal lain** (BEDA dari pola `JamPelajaranAktivasi` lama yang "1 aktif per tingkat" — di sini bisa BANYAK Opsi Jadwal aktif bersamaan asal tidak bentrok cakupan tingkat+kelas, VALIDASI di poin Edge Cases) — VALIDASI: **TOLAK aktivasi kalau ADA Opsi Jadwal LAIN yang JUGA aktif dengan cakupan tingkat OVERLAP DAN semester SAMA** (2 Opsi Jadwal aktif bersamaan untuk tingkat yang sama dalam semester yang sama = ambigu, kelas mana yang dipakai tidak jelas) — pesan error sebutkan nama Opsi Jadwal yang bentrok cakupan.
- `deactivate(id, actorId)` — set `isActive: false`. Data `JadwalSlot` TIDAK dihapus (tetap arsip, sesuai keputusan diskusi).
- `delete(id)` — TOLAK kalau `isActive: true` (harus nonaktifkan dulu) ATAU ada `JadwalSlot` terkait (harus kosongkan dulu) — pesan jelas.

### 3. Modul `JadwalSlotService` — validasi paling kompleks

- `create(dto, actorId)` — input: `opsiJadwalId`, `kelasId`, `mapelId`, `hari`, `jamKe`, `minggu?`, `teacherIds: number[]` (minimal 1, team-teaching).
  1. **Validasi `minggu`**: kalau `OpsiJadwal.mode === blok`, `minggu` WAJIB terisi (`BadRequestException` jelas kalau kosong). Kalau `mode === normal`, `minggu` HARUS `null` (TOLAK kalau diisi — pesan jelas "Opsi Jadwal ini mode Normal, field Minggu tidak berlaku").
  2. **Validasi `jamKe` valid**: HARUS ADA baris `AlokasiWaktuSlot` dengan `hari` dan `jamKe` yang SAMA di `AlokasiWaktu` milik `OpsiJadwal` ini (`jamKe` bukan NULL — bukan baris istirahat) — TOLAK dengan pesan jelas kalau tidak ketemu ("Jam Ke X tidak terdaftar di Alokasi Waktu opsi ini untuk hari Y").
  3. **Validasi duplikat kelas-hari-jamKe untuk mode NORMAL** (celah constraint DB yang dicatat di T204) — QUERY EKSPLISIT cek `JadwalSlot` lain dengan `opsiJadwalId+kelasId+hari+jamKe` SAMA dan `minggu IS NULL` — TOLAK kalau ketemu (constraint DB tidak menangkap kasus NULL, WAJIB dicek manual di service).
  4. **Validasi guru terdaftar di Mapel** (`MapelGuru`) — SETIAP `teacherId` di `teacherIds` HARUS ADA baris `MapelGuru` untuk `mapelId` ini — TOLAK dengan pesan jelas SEBUTKAN nama guru mana yang belum terdaftar ("Guru [Nama] belum terdaftar sebagai pengampu mapel [Nama Mapel] — daftarkan dulu di menu Mata Pelajaran").
  5. **Validasi bentrok guru** — untuk SETIAP `teacherId`, cek TIDAK ADA `JadwalSlot` LAIN (opsiJadwalId BOLEH BEDA — guru yang sama bisa punya jadwal di Opsi Jadwal lain yang JUGA aktif bersamaan untuk semester sama) dengan `hari+jamKe` SAMA (dan `minggu` cocok — REPLIKASI logic `isEverySelf`/`isEveryOther` dari `ensureNoBentrok()` LAMA, T182 — `null` mode normal diperlakukan SAMA seperti "setiap minggu") YANG SUDAH terisi guru itu — TOLAK dengan pesan jelas sebutkan DI KELAS MANA guru itu sudah mengajar jam itu.
  6. Resolve jam wall-clock via `AlokasiWaktuSlot` (untuk response/display, BUKAN disimpan di `JadwalSlot` — tetap resolve dinamis KONSISTEN pola T158).
  7. Create `JadwalSlot` + `JadwalSlotGuru[]` dalam 1 `$transaction`.
- `update(id, dto, actorId)` — SAMA validasi seperti create (kecuali dirinya sendiri dikecualikan dari cek bentrok/duplikat, `excludeId`).
- `delete(id)` — cascade hapus `JadwalSlotGuru` (via `onDelete: Cascade` schema), TIDAK ADA proteksi tambahan (hapus jadwal 1 slot itu aman, tidak ada data lain bergantung sampai T209).

### 4. Endpoint BARU untuk cek bentrok REAL-TIME (T212 dropdown Guru)

- `GET /jadwal-slot/cek-ketersediaan-guru?opsiJadwalId=&hari=&jamKe=&minggu=&excludeSlotId=` — return `{teacherId, tersedia: boolean, bentrokDi?: {kelasNama: string}}[]` UNTUK SEMUA guru yang terdaftar di Mapel yang RELEVAN (terima juga `mapelId` sebagai query param untuk filter guru dari `MapelGuru`) — dipanggil frontend SETIAP kali dropdown Guru dibuka di form input jadwal (T212), REUSE logic bentrok yang SAMA dengan poin 3.5 di atas (EXTRACT jadi method privat shared, JANGAN duplikasi logic).

## Edge Cases
- 2 Opsi Jadwal AKTIF bersamaan untuk semester sama TAPI cakupan tingkat BERBEDA (Opsi A untuk tingkat X, Opsi B untuk tingkat XII) — DIIZINKAN, tidak overlap secara cakupan.
- Guru yang mengajar LINTAS TINGKAT (kelas X dan kelas XII) di Opsi Jadwal BERBEDA yang keduanya aktif — bentrok jam TETAP dicek LINTAS Opsi Jadwal (guru yang sama tidak bisa di 2 tempat sekaligus terlepas Opsi Jadwal mana, KONSISTEN filosofi guru fisik 1 orang).
- Opsi Jadwal mode Blok yang BELUM ada `OpsiJadwalMingguGenerate` sama sekali (belum di-generate, T208) — `create()` JadwalSlot TETAP BISA jalan (minggu tetap harus diisi manual A/B), TAPI resolusi "tanggal X ini minggu apa" (dipakai T209 generate TeachingSession) akan gagal sampai admin generate — INI BUKAN blocker untuk create JadwalSlot itu sendiri, MURNI soal kapan resolusinya dipakai nanti.

## Files
- **Buat:** `apps/api/src/alokasi-waktu/` (controller+service+dto), `apps/api/src/opsi-jadwal/` (controller+service+dto), `apps/api/src/jadwal-slot/` (controller+service+dto, termasuk endpoint cek-ketersediaan-guru).
- **Jangan sentuh:** `JamPelajaranService`/`SchedulesService` LAMA (TIDAK dihapus/diubah sampai T215, boleh tetap berjalan paralel).

## Acceptance Criteria
- [x] CRUD Alokasi Waktu berfungsi, Sabtu opsional, tolak update yang menghapus jamKe terpakai.
- [x] CRUD Opsi Jadwal — mode permanen setelah create (update dengan field mode ditolak), aktivasi per-tingkat tidak overlap.
- [x] CRUD JadwalSlot — SEMUA 5 validasi (minggu, jamKe valid, duplikat mode-normal, guru terdaftar Mapel, bentrok guru) berfungsi dengan pesan error jelas dan actionable.
- [x] Endpoint cek-ketersediaan-guru real-time berfungsi, REUSE logic bentrok yang sama dengan create().
- [x] `@LogActivity` terpasang di semua endpoint mutasi — DIGANTI manual `activityLog.record()` (bukan decorator) karena snapshot butuh nested relations (slots/tingkatScopes/guru), sama pola T172/T193/T196.
- [x] Build + type-check hijau, jest lengkap untuk SEMUA skenario validasi di atas (termasuk reproduksi celah constraint DB mode-normal sebelum fix, kalau relevan).

## Validasi Claudian
- [x] **WAJIB test eksplisit** celah constraint DB mode-normal (2 JadwalSlot kelas-hari-jamKe sama dengan minggu NULL keduanya) — DIKONFIRMASI: test `jadwal-slot.service.spec.ts` "REPRODUKSI CELAH" membuktikan `findFirst` query eksplisit menangkap kasus yang `@@unique` DB lewatkan.
- [x] Konfirmasi endpoint cek-ketersediaan-guru REUSE logic yang SAMA dengan validasi create() (extract method shared), bukan implementasi duplikat yang berisiko tidak sinkron — DIKONFIRMASI: `cekKetersediaanGuru()` memanggil `ensureNoBentrok()` privat yang SAMA persis dipakai `create()`/`update()`, dibungkus try/catch per kandidat guru (bukan reimplementasi logic overlap).
- [x] Konfirmasi pesan error SEMUA validasi actionable (sebutkan APA yang salah dan APA yang harus dilakukan admin) — KONSISTEN aturan CLAUDE.md.

## Status Eksekusi

**Selesai 2026-08-17 05:15**

3 modul NestJS baru: `apps/api/src/alokasi-waktu/`, `apps/api/src/opsi-jadwal/`, `apps/api/src/jadwal-slot/` (masing-masing controller+service+dto+module, didaftarkan di `app.module.ts`). Semua route dijaga `@Roles(super_admin, admin_jurnal)` konsisten T188/T193/T202.

Detail teknis per keputusan desain:
- `AlokasiWaktuService.update()` — sebelum REPLACE-penuh slot, cek dulu `ensureNoJadwalSlotOrphaned()`: kumpulkan semua `(hari,jamKe)` yang masih dipakai `JadwalSlot` (via `OpsiJadwal` yang merujuk Alokasi Waktu ini), tolak kalau slot baru tidak lagi punya baris untuk kombinasi itu.
- `OpsiJadwalService` — `UpdateOpsiJadwalDto` SENGAJA tidak `extends PartialType(CreateOpsiJadwalDto)`, field `mode`/`semesterId`/`alokasiWaktuId` tidak ada sama sekali di DTO update — kombinasi dengan `forbidNonWhitelisted: true` global (`main.ts`) membuat body yang menyertakan `mode` otomatis ditolak 400 oleh ValidationPipe, tanpa perlu if-check manual di service.
- `JadwalSlotService.ensureNoBentrok()` — REPLIKASI persis semantik `isEverySelf`/`isEveryOther` dari `SchedulesService.ensureNoBentrok()` (T182), tapi query kandidat lintas SEMUA `OpsiJadwal` yang `isActive: true` (bukan cuma opsiJadwalId yang sama) sesuai Edge Cases spec (guru fisik 1 orang, bentrok lintas Opsi Jadwal aktif manapun).
- Test: 33 baru (AlokasiWaktu 8, OpsiJadwal 10, JadwalSlot 15) + 506 existing lulus. 1 file di luar scope (`mapel.service.spec.ts`) gagal karena sedang diedit live oleh sesi paralel T207 saat verifikasi — dikonfirmasi via `git status` bukan menyentuh direktori T206, bukan regresi dari task ini.
