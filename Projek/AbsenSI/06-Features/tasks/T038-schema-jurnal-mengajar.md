# T038 — Prisma Schema: Jurnal Mengajar, Izin Guru, Mode Jadwal, Geofence

## Depends on
T002 (schema dasar Fase 1 harus sudah ada). Tidak depend ke task lain di batch ini — task ini paling awal, semua task T039+ depend ke task ini.

## Objective
Tambahkan seluruh tabel & kolom baru ke `schema.prisma` yang dibutuhkan fitur Dashboard Guru Jurnal (lihat `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md`), jalankan migrasi, dan update seed data minimal untuk development/testing task-task berikutnya.

## Context
- **App:** `apps/api`
- **Ref wajib dibaca:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — baca SELURUH file sebelum mulai, terutama bagian "Implikasi Skema"
- **Tabel existing yang terlibat:** `schedules`, `kampus`, `users`, `attendance_sessions`, `attendance_records`

## Spec Detail

### 1. Tabel baru: `mapel` (master mata pelajaran)
```
id            Int      @id @default(autoincrement())
nama          String
kode          String?  @unique   // kode singkat opsional, misal "MTK", "RPL-01"
created_at    DateTime @default(now())
```

### 2. Model konfigurasi sekolah (HANYA toleransi keterlambatan): `schedule_config`
> **Revisi 2026-07-22:** field `mode`/`minggu_acuan_tanggal`/`minggu_acuan_nilai` yang sebelumnya direncanakan di sini **DIPINDAH ke tabel `semesters`** (dikerjakan di T054, bukan di sini) — mode blok/normal sekarang per-semester, bukan konfigurasi sekolah tunggal. Rentang tanggal minggu A/B juga **tidak lagi dihitung dari 1 titik acuan** — admin input rentang eksplisit lewat tabel `block_week_ranges` (juga di T054). Tabel `schedule_config` di task ini HANYA menyimpan toleransi keterlambatan (T042), bukan lagi hal lain.

Single-row config (pola: service selalu `upsert` ke `id: 1`, tidak ada constraint DB untuk "1 baris saja").
```
id                          Int      @id @default(autoincrement())
toleransi_terlambat_menit   Int      @default(10)
updated_by                  Int      // FK users
updated_at                  DateTime @updatedAt
```

### 3. Enum baru
```prisma
enum MingguTag {
  A
  B
  setiap_minggu
}

enum SesiJurnalStatus {
  open
  closed
}

enum ClassAttendanceStatus {
  hadir
  tidak_ada_di_kelas
}

enum TeacherPermitStatus {
  diizinkan
}
```

### 4. Perubahan ke `schedules` (existing table)
Tambah kolom:
```
mapel_id   Int?       // FK mapel, nullable — dulu placeholder nullable, sekarang benar-benar dipakai
minggu     MingguTag? // nullable — null kalau semesters.mode (T054) = normal untuk semester terkait (diabaikan), wajib diisi kalau mode = blok
```
> **Catatan penting:** kolom `mapel_id` sudah disebut sebagai placeholder nullable di `04-Database-Schema.md` sejak Fase 1 (ADR-005) tapi belum pernah dipakai — task ini yang pertama kali benar-benar mengisinya.
> **Catatan revisi 2026-07-22:** `minggu` di sini TETAP dipakai persis seperti semula (tag `A`/`B`/`setiap_minggu` per slot jadwal) — yang berubah HANYA sumber "minggu aktif hari ini sekarang apa", dari hitungan otomatis (draf lama) jadi lookup ke `block_week_ranges` per semester (T054). Kolom ini tidak berubah strukturnya.

### 5. Perubahan ke `kampus` (existing table)
Tambah kolom:
```
lokasi_lat             Decimal? @db.Decimal(10, 7)
lokasi_lng             Decimal? @db.Decimal(10, 7)
radius_geofence_meter  Int?     // nullable, admin isi belakangan — kalau null maka gating geofence untuk kampus itu SKIP (lihat catatan di T043)
```

### 6. Perubahan ke `users` (existing table)
- Tambah nilai enum baru ke `role`: `admin_jurnal`
- **Tidak** ada kolom tambahan untuk `admin_jurnal` di `users` — role ini tidak di-scope ke kampus/kelas manapun (akses global ke seluruh domain jurnal, sesuai spec)

### 7. Tabel baru: `teaching_sessions` (sesi mengajar guru per slot per tanggal)
> Nama sengaja dibedakan dari `attendance_sessions` (yang generic gerbang/kelas dari Fase 1) karena field yang dibutuhkan cukup berbeda (lokasi GPS, status open/closed, dst) — reuse `attendance_sessions` dievaluasi saat desain tapi field-nya tidak cocok tanpa banyak kolom nullable tambahan yang membingungkan.
```
id             Int      @id @default(autoincrement())
schedule_id    Int      // FK schedules — slot jadwal tetap guru yang jadi sumber sesi ini
teacher_id     Int      // FK teachers
kelas_id       Int      // FK kelas
mapel_id       Int      // FK mapel
tanggal        DateTime @db.Date
started_at     DateTime?   // diisi saat guru klik "Mulai Mengajar", null = belum mulai
closed_at      DateTime?   // diisi otomatis oleh job saat jam selesai jadwal tercapai
status         SesiJurnalStatus @default(open)  // "open" dari saat dibuat (belum tentu started_at terisi), "closed" setelah closed_at
lokasi_lat     Decimal? @db.Decimal(10, 7)   // capture saat started_at
lokasi_lng     Decimal? @db.Decimal(10, 7)
terlambat_menit Int?    // dihitung saat started_at diisi: max(0, started_at - (jadwal_mulai + toleransi))
created_at     DateTime @default(now())

@@unique([schedule_id, tanggal])   // 1 slot jadwal cuma bikin 1 sesi per tanggal
```
> **Catatan status:** baris `teaching_sessions` dibuat oleh job harian (lihat T039) untuk SETIAP slot jadwal yang jatuh pada tanggal itu — bukan dibuat on-demand saat guru klik. Ini supaya sesi yang guru "belum mulai" tetap bisa dideteksi & ditampilkan di TV Piket nanti (Fase selanjutnya) tanpa harus insert row baru saat query read.

### 8. Tabel baru: `journal_entries`
```
id                    Int      @id @default(autoincrement())
session_id            Int      @unique   // FK teaching_sessions — 1 sesi = 1 jurnal
materi                String?  @db.Text
tujuan_pembelajaran   String?  @db.Text
tugas_penilaian       String?  @db.Text
catatan               String?  @db.Text
created_at            DateTime @default(now())
updated_at            DateTime @updatedAt
```

### 9. Tabel baru: `class_attendance_marks`
```
id           Int      @id @default(autoincrement())
session_id   Int      // FK teaching_sessions
student_id   Int      // FK students
status       ClassAttendanceStatus @default(hadir)
marked_by    Int      // FK users (guru yang koreksi)
marked_at    DateTime @default(now())

@@unique([session_id, student_id])
```
> Baris di sini HANYA dibuat untuk siswa yang guru tandai `tidak_ada_di_kelas` (koreksi manual). Siswa yang tidak punya baris di sini untuk sesi tsb dianggap default `hadir` — bukan berarti semua siswa harus punya baris. Ini keputusan desain untuk menghindari insert 30+ baris tiap sesi kalau semua siswa hadir normal.

### 10. Tabel baru: `teacher_permits`
```
id                  Int      @id @default(autoincrement())
teacher_id          Int      // FK teachers
tanggal             DateTime @db.Date
session_id          Int?     // FK teaching_sessions, nullable — null berarti izin SEHARIAN PENUH (semua slot guru itu hari ini), terisi = izin cuma slot spesifik itu
kategori            TeacherPermitKategori   // enum: sakit / izin_pribadi / tugas_dinas / pelatihan — murni informatif
status              TeacherPermitStatus @default(diizinkan)
bukti_file_path     String   // WAJIB (bukan nullable) — path relatif file bukti (surat/nota dinas/dsb), pola sama seperti students.foto (ADR-023)
bukti_updated_at    DateTime @default(now())   // update tiap kali admin replace file bukti
approved_by          Int      // FK users, harus role admin_jurnal
approved_at          DateTime @default(now())
tugas_file_path      String?  // path relatif, tugas TITIPAN untuk siswa — BEDA dari bukti_file_path (bukti = alasan izin, ini = materi pengganti)
tugas_keterangan     String?  @db.Text
submitted_at         DateTime?   // kapan guru isi form, null = belum diisi
follow_up_needed     Boolean  @default(false)   // di-set true oleh job kalau lewat jam sesi & tugas belum diisi (lihat T044)
```
> **Catatan penting:** `bukti_file_path` WAJIB diisi sejak baris pertama kali dibuat (lihat T046) — beda dari `tugas_file_path` yang nullable karena guru mungkin belum sempat isi. Field baru ini menambah enum:
```prisma
enum TeacherPermitKategori {
  sakit
  izin_pribadi
  tugas_dinas
  pelatihan
}
```

## JANGAN
- ❌ JANGAN buat tabel `wali_kelas` atau ubah struktur role untuk Wali Kelas — itu masih open question di spec, TIDAK masuk scope task ini
- ❌ JANGAN insert baris `class_attendance_marks` untuk siswa berstatus hadir — hanya untuk yang dikoreksi jadi `tidak_ada_di_kelas`
- ❌ JANGAN buat `teaching_sessions` dengan FK langsung ke `attendance_sessions` — ini tabel terpisah, tidak menggantikan `attendance_sessions` existing (yang tetap dipakai gerbang)
- ❌ JANGAN hapus atau ubah kolom `mapel_id` yang sudah ada sebagai placeholder nullable di `schedules` — task ini isi definisinya, formatnya harus tetap kompatibel dengan data lama (semua data lama punya `mapel_id = null`, itu valid)
- ❌ JANGAN buat constraint DB untuk "1 baris saja" di `schedule_config` — MySQL/Prisma tidak native support ini, cukup didisiplinkan di service layer (`upsert` ke `id: 1` selalu)
- ❌ JANGAN buat kolom `radius_geofence_meter` non-nullable — kampus lama (Fase 1) belum ada datanya, harus nullable dengan makna eksplisit "gating skip" (lihat T043)
- ❌ JANGAN buat `bukti_file_path` nullable — WAJIB diisi sejak create (lihat T046), beda dari `tugas_file_path` yang memang nullable
- ❌ JANGAN buat tabel `semesters` atau kolom `schedules.semester_id` di task ini — itu scope T054 (task terpisah, karena domainnya kalender pendidikan bukan murni jurnal). Task ini fokus ke tabel yang sudah dirinci di atas saja

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma`
- **Buat:** `apps/api/prisma/migrations/` (auto-generated dari `prisma migrate dev`)
- **Modifikasi:** `apps/api/prisma/seed.ts` — tambah seed minimal:
  - 3 `mapel`: "Matematika", "Pemrograman Web" (kode "RPL-WEB"), "Produktif TKJ" (kode "TKJ-PROD")
  - 1 `schedule_config` row: `toleransi_terlambat_menit: 10` (hanya field ini, `mode`/rentang minggu A/B di-seed di T054)
  - 1 user `admin_jurnal`: username "adminjurnal", password "jurnal123"
  - Update 2-3 `schedules` existing (dari seed T002) untuk isi `mapel_id` dan `minggu` (campur `A`, `B`, `setiap_minggu`) — nilai `minggu` ini baru benar-benar "aktif"/resolvable setelah T054 selesai (butuh `block_week_ranges`), tapi kolomnya sudah bisa diisi dari task ini

## Acceptance Criteria
- [ ] `prisma migrate dev` berjalan tanpa error, semua tabel baru muncul di `prisma studio`
- [ ] `prisma db seed` berjalan, data seed baru masuk tanpa merusak seed Fase 1 yang sudah ada
- [ ] Enum `role` di `users` berisi `admin_jurnal` sebagai nilai valid
- [ ] `schedules.mapel_id` dan `schedules.minggu` bisa diisi dan dibaca lewat Prisma Client
- [ ] `teaching_sessions` punya unique constraint `(schedule_id, tanggal)` — insert duplikat harus gagal
- [ ] `journal_entries.session_id` unique — 1 sesi tidak bisa punya 2 jurnal
- [ ] `class_attendance_marks` unique `(session_id, student_id)` — 1 siswa tidak bisa ditandai dobel di sesi yang sama
- [ ] `teacher_permits.bukti_file_path` non-nullable — insert tanpa nilai ini harus gagal di level schema/Prisma Client type

## Handoff ke T039
T039 (job generate sesi harian) butuh `teaching_sessions`, `schedules.minggu` sudah ada dan bisa di-query. **Catatan:** T039 sekarang JUGA depend ke T054 (schema semester + `block_week_ranges`) karena resolver "minggu aktif hari ini" sudah pindah mekanisme — baca ulang T039 dan T054 sebelum eksekusi, jangan asumsi dari versi lama task ini.

## Handoff ke T054
T054 (schema semester) akan menambah kolom `schedules.semester_id` — pastikan kolom `mapel_id`/`minggu` yang dibuat task ini tidak perlu diubah lagi saat T054 menambah `semester_id` (kolom terpisah, tidak saling bergantung secara struktur).
