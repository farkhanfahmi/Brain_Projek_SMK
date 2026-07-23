# T054 — Schema: Semester (Ganjil/Genap) + Rentang Blok A/B Eksplisit per Semester

## Depends on
T002 (schema dasar Fase 1, tabel `academic_years` harus sudah ada), T038 (schema jurnal — `schedules` sudah punya `mapel_id`/`minggu` dari task itu, task ini menambah kolom terpisah `semester_id`)

> **Task ini paling sensitif secara logic di seluruh batch semester** — berisi validasi no-gap dan no-overlap yang KRUSIAL untuk mencegah sistem penjadwalan blok jadi ambigu. Baca seluruh spec sebelum eksekusi, jangan skip bagian validasi.

## Objective
Tambahkan tabel `semesters` (2 semester per tahun ajaran, tanggal manual, 1 aktif sekaligus, mode blok/normal PER semester) dan tabel `block_week_ranges` (rentang tanggal eksplisit yang admin input untuk menandai tiap minggu sebagai `A` atau `B`), lengkap dengan validasi ketat no-gap/no-overlap yang WAJIB dicek di service layer.

## Context
- **App:** `apps/api`
- **Tables:** `academic_years` (existing), `schedules` (existing, extend)
- **Ref:** `Projek/AbsenSI/06-Features/kalender-pendidikan.md` — bagian "1b. Semester", dan `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "📅 Jadwal: Mode Blok (Minggu A/B) vs Mode Normal" (**baca ULANG bagian ini, direvisi total 2026-07-22** — mekanisme "titik acuan + hitung selisih" DIBATALKAN, diganti rentang eksplisit)

## Spec Detail

### 1. Tabel baru: `semesters`
```prisma
model Semester {
  id                Int          @id @default(autoincrement())
  academic_year_id  Int          // FK academic_years
  nama              SemesterNama // enum: ganjil / genap
  tanggal_mulai     DateTime     @db.Date
  tanggal_selesai   DateTime     @db.Date
  mode              ScheduleMode @default(normal)   // enum: blok / normal — PER SEMESTER, bukan global sekolah
  is_active         Boolean      @default(false)     // hanya 1 TRUE lintas SEMUA tahun ajaran, bukan cuma dalam 1 academic_year
  created_by        Int          // FK users
  created_at        DateTime     @default(now())

  @@unique([academic_year_id, nama])   // 1 tahun ajaran cuma punya 1 semester ganjil dan 1 semester genap
}

enum SemesterNama {
  ganjil
  genap
}

enum ScheduleMode {
  blok
  normal
}
```
- **Constraint "1 is_active lintas semua semester"**: ditegakkan di service layer (pola sama seperti `academic_years.is_active` di Fase 1) — saat set 1 semester jadi aktif, service HARUS set semua semester lain (termasuk lintas tahun ajaran berbeda) jadi `is_active: false` dalam 1 transaction

### 2. Tabel baru: `block_week_ranges` (rentang tanggal eksplisit minggu A/B — INTI perubahan 2026-07-22)
```prisma
model BlockWeekRange {
  id            Int       @id @default(autoincrement())
  semester_id   Int       // FK semesters
  tanggal_mulai DateTime  @db.Date
  tanggal_selesai DateTime @db.Date
  minggu        MingguABSaja   // enum khusus, HANYA A/B — TIDAK ada 'setiap_minggu' di level rentang tanggal (beda dari schedules.minggu yang dari T038 punya 3 nilai)
  created_by    Int       // FK users
  created_at    DateTime  @default(now())
}

enum MingguABSaja {
  A
  B
}
```
- **Kenapa enum terpisah dari `MingguTag` (T038)**: `schedules.minggu` butuh 3 nilai (`A`/`B`/`setiap_minggu` — untuk jam yang sama di kedua minggu). `block_week_ranges.minggu` cuma butuh 2 nilai karena ini menandai KALENDER (tiap minggu literal adalah A atau B, tidak ada "minggu campuran"). Jangan disatukan jadi 1 enum.

### 3. Validasi WAJIB (Service Layer — Ini Bagian Terpenting Task Ini)

Semua validasi berikut dicek di `BlockWeekRangesService`, dijalankan **setiap kali admin submit 1 pasangan rentang baru** (`POST /block-week-ranges`) — REAL-TIME, bukan ditunda ke titik lain.

> **Wajib pakai DB transaction dengan row lock untuk validasi (a)+(b)+insert** (misal `SELECT ... FOR UPDATE` di baris-baris `block_week_ranges` yang relevan, atau serialize lewat transaction isolation level yang cukup ketat di MySQL) — bukan "cek dulu di query terpisah, baru insert di query lain". Dua request submit rentang yang overlap nyaris bersamaan (dua tab browser admin, atau retry jaringan) HARUS tidak bisa keduanya lolos validasi lalu sama-sama ter-insert. Ini bukan optimisasi, ini mencegah data korup yang persis ingin dicegah validasi (a)/(b) di atas.

**a) Tidak boleh overlap DALAM semester yang sama**
- Rentang baru (`tanggal_mulai`–`tanggal_selesai`) tidak boleh beririsan dengan `block_week_ranges` manapun yang SUDAH ADA untuk `semester_id` yang sama
- Overlap check: `existing.tanggal_mulai <= baru.tanggal_selesai AND existing.tanggal_selesai >= baru.tanggal_mulai`
- Gagal → 409, response sertakan rentang yang bentrok (tanggal + minggu-nya)

**b) Tidak boleh overlap ANTAR semester (semester apapun, aktif atau tidak)**
- Rentang baru tidak boleh beririsan dengan `block_week_ranges` milik `semester_id` LAIN manapun — join ke semua `block_week_ranges` lintas semester, cek overlap yang sama seperti (a)
- Ini yang memungkinkan validasi "Semester Genap sedang disiapkan tidak boleh klaim tanggal yang masih milik Semester Ganjil yang berjalan"
- Gagal → 409, response sertakan semester + rentang yang bentrok

**c) Rentang harus berada dalam batas `semesters.tanggal_mulai`–`tanggal_selesai` semester terkait**
- `baru.tanggal_mulai >= semester.tanggal_mulai AND baru.tanggal_selesai <= semester.tanggal_selesai`
- Gagal → 400, `"Rentang tanggal di luar batas semester"`

**d) Deteksi "lubang" (gap) — dicek di endpoint TERPISAH, bukan saat submit tiap rentang**
- Endpoint `GET /block-week-ranges/coverage-check?semester_id=` — hitung apakah seluruh hari dari `tanggal_mulai` sampai `tanggal_selesai` semester itu SUDAH tercover oleh `block_week_ranges` (union semua rentang = persis rentang semester, tanpa celah)
- Response: `{ "lengkap": boolean, "lubang": [{ "tanggal_mulai": "...", "tanggal_selesai": "..." }] }` — kalau ada lubang, list rentang tanggal yang belum tercover
- Endpoint ini **read-only, tidak mem-block apapun** saat dipanggil manual — ini alat bantu admin_jurnal untuk melihat progres kelengkapan input. Enforcement HARD BLOCK yang sebenarnya (sesi tidak bisa dimulai) terjadi otomatis lewat T039 (`ScheduleResolverService.getJadwalHariIni` return `[]` untuk tanggal yang tidak ketemu di `block_week_ranges`). **Tapi endpoint ini SEKARANG JUGA jadi blocker keras di `PATCH /semesters/:id/activate`** (lihat bagian API Semester di bawah) — jadi tidak murni "informasi pasif" lagi, sudah dipakai sebagai gerbang validasi juga

**e) Sinyal proaktif "lubang mendekat" — endpoint monitor untuk dashboard admin_jurnal (2026-07-22, menutup celah: sebelumnya tidak ada yang mendorong info ini ke admin tanpa mereka buka halaman kalender duluan)**
- Endpoint `GET /block-week-ranges/upcoming-gaps?hari=7` (default 7 hari ke depan) — cek SEMUA semester yang `is_active: true` dengan `mode: blok`, hitung apakah ada tanggal dalam N hari ke depan dari hari ini yang BELUM tercover `block_week_ranges`
- Response: `{ "ada_lubang_mendekat": boolean, "detail": [{ "tanggal": "...", "hari_lagi": number }] }`
- Dipakai oleh T050 (dashboard admin_jurnal, halaman utama/landing) untuk tampilkan banner peringatan mencolok kalau `ada_lubang_mendekat: true` — supaya admin_jurnal tidak perlu ingat-ingat buka halaman Jadwal Blok Minggu secara manual untuk sadar ada masalah yang akan terjadi beberapa hari lagi

### 4. Perubahan ke `schedules` (existing table, extend — dari T038)
```prisma
semester_id   Int?   // FK semesters — WAJIB diisi untuk type=jam_mengajar, TIDAK RELEVAN (biarkan null) untuk type=jam_sekolah/jadwal_khusus
```
- Nullable di level schema, wajib secara validasi aplikasi untuk `type: jam_mengajar` (dicek di T047)

### 5. API CRUD Semester
**GET `/semesters?academic_year_id=`** — list semester, read-only untuk siapa saja yang login

**POST `/semesters`** — role `super_admin`
```json
{ "academic_year_id": 3, "nama": "ganjil", "tanggal_mulai": "2026-07-13", "tanggal_selesai": "2026-12-19", "mode": "blok" }
```
- Validasi: `academic_year_id` + `nama` unik, tanggal dalam rentang `academic_years` terkait

**PATCH `/semesters/:id`** — role `super_admin` — update tanggal/mode (HANYA kalau belum ada `schedules`/`block_week_ranges` terkait, untuk mencegah data jadi tidak konsisten — kalau sudah ada data terkait, tolak 409 dengan pesan jelas "sudah ada jadwal/rentang terkait, tidak bisa diubah")

**PATCH `/semesters/:id/activate`** — role `super_admin`
- **Validasi WAJIB sebelum aktivasi (2026-07-22, celah yang sempat lolos dari draf awal):** kalau `semester.mode === 'blok'`, panggil `coverage-check` internal dulu — **tolak 409** kalau `lengkap: false`, sertakan daftar lubang di response error. Semester dengan lubang jadwal TIDAK BOLEH diaktifkan — kalau lolos, besok semua guru terkunci total dari "Mulai Mengajar" karena `getJadwalHariIni()` return `[]`. Ini hard block, bukan sekadar warning
- Kalau `mode: normal` → tidak perlu cek coverage (tidak relevan, tidak ada `block_week_ranges` untuk mode ini)
- Set semester ini `is_active: true`, semua semester lain `is_active: false` (transaction)
- Log ke `activity_log`, action `semester.activate`

### 6. API CRUD Block Week Ranges
**GET `/block-week-ranges?semester_id=`** — list rentang untuk 1 semester, urut tanggal

**POST `/block-week-ranges`** — role `admin_jurnal` (ini beda dari semester itu sendiri yang `super_admin` — rentang blok A/B adalah bagian dari PENJADWALAN, domain `admin_jurnal`, sedangkan semester sebagai wadah kalender tetap `super_admin`)
```json
{ "semester_id": 5, "tanggal_mulai": "2026-07-13", "tanggal_selesai": "2026-07-19", "minggu": "A" }
```
- Jalankan validasi (a), (b), (c) di atas SEBELUM insert — gagal salah satu → tolak, tidak ada baris tercipta
- Log ke `activity_log`, action `block_week_range.create`

**DELETE `/block-week-ranges/:id`** — role `admin_jurnal` — untuk kasus admin salah input dan perlu hapus lalu buat ulang (bukan edit in-place, sesuai open question di spec yang belum final — DELETE+CREATE lebih aman untuk sekarang daripada edit in-place yang butuh validasi ulang kompleks)
- **Validasi WAJIB (2026-07-22, menutup celah "sesi sudah tergenerate tapi jadwalnya diubah di tengah hari"):** tolak 409 kalau `tanggal_mulai` atau `tanggal_selesai` rentang yang mau dihapus **mencakup hari ini atau tanggal yang sudah lewat** (`tanggal_selesai < hari ini` ATAU rentang mencakup hari ini) — `"Tidak bisa menghapus rentang yang mencakup hari ini atau sudah lewat. teaching_sessions untuk tanggal itu mungkin sudah digenerate/dipakai guru."`. Rentang untuk tanggal MASA DEPAN (belum tiba) tetap boleh dihapus kapan saja
- Validasi yang sama (tolak untuk tanggal hari ini/lewat) **juga berlaku untuk `POST /block-week-ranges`** kalau admin mencoba menambah rentang baru yang mencakup tanggal hari ini atau masa lalu — cegah admin menyisipkan rentang baru yang bisa mengubah resolusi jadwal untuk hari yang sesinya sudah/sedang berjalan

## JANGAN
- ❌ JANGAN buat `semesters` (siklus hidup: create/edit/activate) sebagai wewenang `admin_jurnal` — domain `super_admin`. Tapi `block_week_ranges` (isi rentang blok A/B) ADALAH wewenang `admin_jurnal` — dua tabel ini beda pemilik wewenang, jangan disamakan
- ❌ JANGAN hitung semester atau rentang blok otomatis dari tanggal (misal "tengah tahun ajaran", atau "generate otomatis tiap 7 hari") — SEMUA tanggal (semester maupun tiap rentang blok) WAJIB input manual admin, sesuai keputusan eksplisit untuk menghindari kesalahan sistem
- ❌ JANGAN skip validasi (a)/(b)/(c) — ini BUKAN nice-to-have, ini yang mencegah seluruh sistem jadwal blok jadi data yang ambigu/rusak. Kalau salah satu validasi ini terlewat, `ScheduleResolverService` (T039) bisa dapat >1 hasil untuk 1 tanggal (ambiguitas) atau 0 hasil yang seharusnya tidak 0 (bug, bukan celah asli)
- ❌ JANGAN buat validasi gap (d) sebagai BLOCKER saat submit — itu cuma read-only info endpoint, bukan yang mem-block. Blocker sebenarnya ada di T039 (return `[]`/`null` untuk tanggal yang tidak ketemu), BUKAN di endpoint create rentang ini
- ❌ JANGAN buat endpoint edit in-place untuk `block_week_ranges` — cuma create & delete, sesuai keputusan sementara (edit in-place ditandai open question, jangan diimplementasikan sampai diputuskan eksplisit)
- ❌ JANGAN buat `semester_id` wajib (non-nullable) di level schema Prisma untuk SEMUA `schedules` — nullable di schema, validasi "wajib untuk jam_mengajar" ada di service layer (T047)
- ❌ JANGAN buat lebih dari 2 semester per tahun ajaran — unique constraint `(academic_year_id, nama)` dengan `nama` cuma `ganjil`/`genap` sudah membatasi ini secara alami
- ❌ JANGAN validasi overlap (a)/(b) dengan pola "SELECT cek dulu, lalu INSERT di query terpisah" tanpa row lock/transaction — celah race condition nyata untuk 2 request nyaris bersamaan, WAJIB pakai transaction dengan lock yang memadai (lihat catatan di bagian Validasi)
- ❌ JANGAN izinkan `PATCH /semesters/:id/activate` lolos tanpa cek `coverage-check` untuk semester `mode: blok` — WAJIB tolak 409 kalau ada lubang, ini hard block bukan warning (celah kritis yang sempat lolos dari draf awal)
- ❌ JANGAN izinkan `POST`/`DELETE /block-week-ranges` untuk rentang yang mencakup tanggal hari ini atau sudah lewat — hanya untuk tanggal masa depan (celah: sesi yang sudah tergenerate/dipakai guru tidak boleh terpengaruh perubahan jadwal blok di tengah jalan)

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` — tambah model `Semester`, `BlockWeekRange`, enum `SemesterNama`, `ScheduleMode`, `MingguABSaja`, kolom `schedules.semester_id`
- **Buat:** `apps/api/prisma/migrations/`
- **Buat:** `apps/api/src/semesters/semesters.module.ts`, `semesters.service.ts`, `semesters.controller.ts`
- **Buat:** `apps/api/src/block-week-ranges/block-week-ranges.module.ts`, `block-week-ranges.service.ts` (berisi SEMUA logic validasi a/b/c/d), `block-week-ranges.controller.ts`
- **Buat:** `apps/api/src/block-week-ranges/block-week-ranges.service.spec.ts` — unit test WAJIB untuk tiap validasi:
  - Submit rentang yang overlap dengan rentang lain di semester sama → 409
  - Submit rentang yang overlap dengan semester LAIN → 409
  - Submit rentang di luar batas tanggal semester → 400
  - Submit rentang valid (tidak overlap apapun, dalam batas) → berhasil
  - `coverage-check` dengan semester yang rentangnya lengkap tanpa celah → `lengkap: true`
  - `coverage-check` dengan semester yang ada 1 hari tidak tercover → `lengkap: false`, `lubang` berisi rentang yang hilang
- **Modifikasi:** `apps/api/prisma/seed.ts` — tambah seed:
  - 2 `semesters` untuk tahun ajaran aktif existing: Ganjil (13 Jul–19 Des 2026, `mode: blok`, `is_active: true`), Genap (5 Jan–20 Jun 2027, `mode: blok`, `is_active: false`)
  - Beberapa `block_week_ranges` untuk Semester Ganjil yang **lengkap tanpa celah** (misal 4-5 pasangan rentang berselang A-B-A-B mencakup beberapa minggu pertama saja untuk keperluan testing — TIDAK perlu mencover seluruh semester di seed, cukup cakupan yang jelas untuk data uji task berikutnya)

## Acceptance Criteria
- [ ] `prisma migrate dev` berjalan tanpa error
- [ ] `POST /semesters` 2x untuk `academic_year_id` sama dengan `nama` sama → yang kedua gagal (unique constraint)
- [ ] `PATCH /semesters/:id/activate` → semester lain (termasuk dari tahun ajaran berbeda) otomatis `is_active: false`
- [ ] `POST /semesters` dari role selain `super_admin` → 403
- [ ] `POST /block-week-ranges` dari role selain `admin_jurnal` → 403
- [ ] Submit rentang blok yang overlap dengan rentang lain DALAM semester sama → 409, tidak ada baris baru tercipta
- [ ] Submit rentang blok untuk Semester Genap yang tanggalnya masih tumpang tindih dengan Semester Ganjil yang sedang berjalan → 409 (validasi antar-semester)
- [ ] Submit rentang blok untuk Semester Genap dengan tanggal yang TIDAK overlap Ganjil (meski Ganjil masih `is_active: true`) → berhasil — membuktikan "boleh disiapkan lebih awal" benar-benar berfungsi
- [ ] `GET /block-week-ranges/coverage-check` untuk semester dengan seed lengkap → `lengkap: true` untuk rentang yang di-seed (bukan untuk seluruh semester kalau seed sengaja tidak lengkap, sesuaikan assertion dengan cakupan seed aktual)
- [ ] `GET /block-week-ranges/coverage-check` untuk semester yang sengaja ada celah → `lengkap: false`, `lubang` berisi rentang yang benar
- [ ] `PATCH /semesters/:id/activate` untuk semester `mode: blok` yang punya lubang → 409, semester TIDAK jadi aktif, `is_active` tidak berubah
- [ ] `PATCH /semesters/:id/activate` untuk semester `mode: blok` yang coverage lengkap → berhasil aktif
- [ ] `PATCH /semesters/:id/activate` untuk semester `mode: normal` → berhasil tanpa perlu cek coverage sama sekali
- [ ] `DELETE /block-week-ranges/:id` untuk rentang yang mencakup HARI INI → 409, tidak terhapus
- [ ] `DELETE /block-week-ranges/:id` untuk rentang MASA DEPAN (belum tiba) → berhasil dihapus
- [ ] `POST /block-week-ranges` dengan tanggal yang mencakup hari ini/masa lalu → 409
- [ ] Uji konkurensi (2 request submit rentang overlap dikirim nyaris bersamaan, misal via `Promise.all` di test) → HANYA 1 yang berhasil, yang lain 409 — bukan keduanya lolos
- [ ] `GET /block-week-ranges/upcoming-gaps` untuk semester dengan lubang dalam 7 hari ke depan → `ada_lubang_mendekat: true`, detail tanggal sesuai

## Handoff ke T039
T039 (resolver jadwal aktif) akan konsumsi `semesters.mode`, `semesters.is_active`, dan `block_week_ranges` sebagai SATU-SATUNYA sumber untuk menentukan minggu aktif — tidak ada lagi perhitungan selisih. Pastikan struktur tabel ini bisa di-query efisien by `(semester_id, tanggal)` — pertimbangkan index komposit `@@index([semester_id, tanggal_mulai, tanggal_selesai])` di `BlockWeekRange` kalau volume data besar (meski untuk 1 sekolah dengan ±20 rentang per semester ini masih sangat kecil, index tetap best practice).

## Handoff ke T047 & T050
T047 (API jadwal mengajar admin_jurnal) WAJIB validasi `semester_id` terisi untuk `type: jam_mengajar`, dan implementasi fitur "Salin Jadwal dari Semester Sebelumnya". T050 (UI admin_jurnal) tambah dropdown semester di halaman Jadwal Mengajar, DAN halaman baru untuk kelola `block_week_ranges` (lihat T056, task UI terpisah untuk kalender visual rentang blok).
