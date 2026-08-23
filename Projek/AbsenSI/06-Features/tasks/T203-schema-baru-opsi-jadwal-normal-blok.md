# T203 — Schema: Fondasi Baru "Opsi Jadwal" (Menggantikan Kelas.modeJadwal + Semester.mode + BlockWeekRange)

## Depends on
Tidak ada dependency teknis. **INI TASK PALING DASAR dari rangkaian T203-T215 (Perombakan Total Modul Jadwal)** — WAJIB dikerjakan PERTAMA, SEMUA task lain di rangkaian ini bergantung pada schema yang dibuat di sini.

## ⚠️ KONTEKS WAJIB DIBACA SEBELUM MULAI — Baca Semua Task Rangkaian Ini Dulu

Task ini bagian dari **perombakan arsitektur total** modul jadwal, hasil diskusi mendalam 2026-08-16 (BUKAN task independen — baca SEMUA file `T203-*.md` sampai `T215-*.md` DULU sebelum mulai implementasi apa pun, supaya paham gambaran besar sebelum kerjakan potongan pertama). Rangkaian ini MEMBONGKAR modul jadwal yang sudah live (`Schedule`, `Kelas.modeJadwal`, `Semester.mode`, `BlockWeekRange`, hasil T158/T159/T182/T189) dan menggantinya dengan arsitektur baru.

**FAKTA PENTING yang MENDASARI keputusan ini boleh dilakukan SEKARANG (bukan nanti)**: dikonfirmasi user 2026-08-16 — **modul jadwal (Schedule, TeachingSession, JournalEntry, ClassAttendanceMark, GradeAssessment) MASIH UJI COBA, BELUM LIVE** dipakai guru sungguhan (data production 2026-08-16: `teaching_sessions=0`, `journal_entries=0`, `class_attendance_marks=0`, `grade_assessments=0`). **Data yang SUDAH LIVE dan TIDAK BOLEH TERGANGGU**: siswa, guru, tap kehadiran gerbang (`AttendanceRecord`/`tap_events`), izin (`Permit`/`TeacherPermit`), ekstrakurikuler. Task ini BOLEH mengubah struktur `Schedule` secara radikal TANPA khawatir migrasi data jadwal lama yang rumit — TAPI WAJIB tetap hati-hati karena `AttendanceRecord`/tap gerbang siswa-guru SUDAH terhubung tidak langsung ke beberapa hal ini (lihat Edge Cases).

## Objective

Bangun schema Prisma BARU untuk hierarki: **Tahun Ajaran → Semester → Opsi Jadwal (dengan mode Normal/Blok permanen sejak dibuat) → Alokasi Waktu terpisah dari Opsi Jadwal → data Schedule per Opsi**. Semester TIDAK LAGI py field `mode` (dibuang total). `Kelas.modeJadwal` DIBUANG TOTAL. `BlockWeekRange` (rentang tunggal) DIGANTI konsep "titik mulai siklus 7-harian" yang di-generate jadi tanggal individual.

## Keputusan Final Dikonfirmasi User (2026-08-16, JANGAN diubah tanpa konfirmasi ulang)

1. **`Semester.mode` DIBUANG TOTAL** — semester murni periode waktu (tanggal mulai-selesai, aktif/tidak), tidak lagi menyimpan mode Normal/Blok.
2. **`Kelas.modeJadwal` DIBUANG TOTAL** — mode bukan lagi atribut Kelas.
3. **Mode (Normal/Blok) jadi atribut "Opsi Jadwal" baru**, **PERMANEN sejak dibuat** — TIDAK BISA diubah setelah Opsi Jadwal dibuat (kalau admin salah pilih mode, harus buat Opsi Jadwal baru, opsi lama boleh dihapus/dinonaktifkan).
4. **Opsi Jadwal punya toggle Aktif/Nonaktif SENDIRI**, independen dari status aktif Tahun Ajaran/Semester induknya — kalau Opsi Jadwal "aktif" tapi Semester/Tahun Ajaran induk TIDAK aktif, status tersimpan APA ADANYA (tidak dipaksa nonaktif), TAPI tidak berpengaruh nyata ke guru/siswa (lihat T206 untuk logic ini, dan T211 untuk Warning Banner terkait).
5. **Alokasi Waktu (dulu `JamPelajaranOption`) SEKARANG entitas TERPISAH dari Opsi Jadwal** — 1 Opsi Jadwal MERUJUK ke 1 Alokasi Waktu (bukan mengandung slot langsung) — supaya 1 Alokasi Waktu bisa dipakai ulang oleh beberapa Opsi Jadwal berbeda tanpa duplikasi data.
6. **Alokasi Waktu mendukung Senin-SABTU** (bukan cuma Senin-Jumat) — Sabtu opsional, kosong = memang tidak ada jadwal pelajaran hari itu untuk Alokasi Waktu itu.
7. **`BlockWeekRange` (rentang tanggal tunggal per baris) DIBUANG**, DIGANTI **`OpsiJadwalMingguGenerate`** — 1 baris = 1 TANGGAL INDIVIDUAL (bukan rentang) yang sudah dikonfirmasi BUKAN hari libur, ditandai minggu A atau B. Dihasilkan dari 2 titik mulai (tanggal mulai Minggu A, tanggal mulai Minggu B — selisih PERSIS 7 hari) yang di-generate otomatis skip hari libur (`SchoolHoliday`) DAN skip hari yang tidak ada di Alokasi Waktu terkait (lihat T208 untuk logic generate lengkap, task ini HANYA bangun modelnya).
8. **Data hasil "Opsi Jadwal LAMA yang dinonaktifkan" TIDAK DIHAPUS** — tetap tersimpan sebagai arsip (lihat T210 untuk logic penggantian Opsi aktif).

## Spec Detail

### 1. Model Prisma baru

```prisma
// Menggantikan JamPelajaranOption LAMA (T158) — sekarang murni "Alokasi Waktu", TIDAK
// menyimpan mode/cakupan tingkat (itu urusan Opsi Jadwal, bukan Alokasi Waktu).
model AlokasiWaktu {
  id          Int      @id @default(autoincrement())
  nama        String   // misal "Alokasi Reguler 2026/2027", "Alokasi Ramadhan"
  createdById Int      @map("created_by")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  slots      AlokasiWaktuSlot[]
  opsiJadwal OpsiJadwal[]        // 1 Alokasi Waktu bisa dipakai banyak Opsi Jadwal
  createdBy  User               @relation(fields: [createdById], references: [id])

  @@map("alokasi_waktu")
}

model AlokasiWaktuSlot {
  id             Int      @id @default(autoincrement())
  alokasiWaktuId Int      @map("alokasi_waktu_id")
  hari           Int      // 1=Minggu..7=Sabtu (DAYOFWEEK MySQL, KONSISTEN Schedule.hari lama).
                           // Sabtu (7) OPSIONAL — kosong utk 1 Alokasi Waktu berarti alokasi
                           // itu memang tidak ada jadwal Sabtu, BUKAN error/wajib diisi.
  jamKe          Int?     @map("jam_ke")      // NULL = baris istirahat
  jamMulai       String   @map("jam_mulai")   // HH:mm
  jamSelesai     String   @map("jam_selesai") // HH:mm
  keterangan     String?  // misal "Istirahat Ke-1"
  urutan         Int      // urutan tampil dalam 1 hari (termasuk baris istirahat)

  alokasiWaktu AlokasiWaktu @relation(fields: [alokasiWaktuId], references: [id], onDelete: Cascade)

  @@unique([alokasiWaktuId, hari, urutan])
  @@index([alokasiWaktuId, hari])
  @@map("alokasi_waktu_slots")
}

enum ModeJadwal {
  normal
  blok
}

// Pengganti "toggle mode" T159 lama — sekarang 1 baris = 1 Opsi Jadwal utuh dengan mode
// PERMANEN, terikat ke Semester tertentu, dan cakupan tingkat (reuse pola T158
// JamPelajaranOptionTingkat/JamPelajaranAktivasi, TAPI sekarang scope-nya OPSI JADWAL bukan
// Alokasi Waktu — 1 Opsi Jadwal py cakupan tingkat SENDIRI, TERPISAH dari cakupan tingkat
// Alokasi Waktu yang dirujuknya, KARENA 1 Alokasi Waktu bisa dipakai lintas Opsi Jadwal).
model OpsiJadwal {
  id             Int        @id @default(autoincrement())
  semesterId     Int        @map("semester_id")
  alokasiWaktuId Int        @map("alokasi_waktu_id")
  nama           String     // misal "Jadwal Reguler Ganjil 2026", "Jadwal Blok Praktik TKR"
  mode           ModeJadwal // PERMANEN sejak create, TIDAK BISA diubah via update()
  isActive       Boolean    @default(false) @map("is_active")
  createdById    Int        @map("created_by")
  createdAt      DateTime   @default(now())
  updatedAt      DateTime   @updatedAt

  semester       Semester                @relation(fields: [semesterId], references: [id])
  alokasiWaktu   AlokasiWaktu            @relation(fields: [alokasiWaktuId], references: [id])
  tingkatScopes  OpsiJadwalTingkat[]     // kosong = semua tingkat (KONSISTEN pola T158 lama)
  mingguGenerate OpsiJadwalMingguGenerate[] // HANYA terisi kalau mode=blok (lihat T208)
  jadwalSlots    JadwalSlot[]            // data Schedule baru, lihat T204
  createdBy      User                    @relation(fields: [createdById], references: [id])

  @@index([semesterId])
  @@map("opsi_jadwal")
}

model OpsiJadwalTingkat {
  id           Int    @id @default(autoincrement())
  opsiJadwalId Int    @map("opsi_jadwal_id")
  tingkat      Tingkat

  opsiJadwal OpsiJadwal @relation(fields: [opsiJadwalId], references: [id], onDelete: Cascade)

  @@unique([opsiJadwalId, tingkat])
  @@map("opsi_jadwal_tingkat")
}

enum MingguAB {
  A
  B
}

// Menggantikan BlockWeekRange (rentang) — 1 baris = 1 TANGGAL INDIVIDUAL hasil generate
// (T208), sudah dikonfirmasi BUKAN hari libur. HANYA relevan utk OpsiJadwal.mode=blok.
model OpsiJadwalMingguGenerate {
  id           Int      @id @default(autoincrement())
  opsiJadwalId Int      @map("opsi_jadwal_id")
  tanggal      DateTime @db.Date
  minggu       MingguAB

  opsiJadwal OpsiJadwal @relation(fields: [opsiJadwalId], references: [id], onDelete: Cascade)

  @@unique([opsiJadwalId, tanggal])
  @@index([opsiJadwalId, minggu])
  @@map("opsi_jadwal_minggu_generate")
}
```

### 2. Migration Prisma

- Migration BARU (bukan alter dari model lama) — CREATE seluruh model di atas.
- **Tidak perlu backfill data dari `JamPelajaranOption`/`Schedule`/`BlockWeekRange` lama** (dikonfirmasi user: data jadwal masih uji coba, kosong di production) — TAPI WAJIB VERIFIKASI ULANG row count `teaching_sessions`/`journal_entries`/`class_attendance_marks`/`grade_assessments` di production TEPAT SEBELUM migration dijalankan (bukan asumsi dari temuan 2026-08-16, kondisi bisa berubah kalau ada task lain jalan duluan) — KALAU ternyata SUDAH ada data nyata di sana saat migration ini dieksekusi, STOP, LAPORKAN ke user, JANGAN lanjutkan migration destruktif tanpa konfirmasi ulang.

### 3. Model LAMA yang akan DIHAPUS (task TERPISAH, JANGAN hapus di task ini)

**CATAT DI SINI, JANGAN EKSEKUSI DI TASK INI** — `JamPelajaranOption`/`JamPelajaranOptionTingkat`/`JamPelajaranAktivasi`/`JamPelajaranSlot`/`BlockWeekRange`/`Kelas.modeJadwal`/`Semester.mode` akan DIHAPUS di **T215** (task pembersihan, PALING TERAKHIR di rangkaian ini, SETELAH semua konsumen lama dipastikan sudah pindah ke schema baru). Task ini HANYA membuat model BARU, TIDAK menyentuh/menghapus model lama sama sekali — supaya sistem LAMA tetap berfungsi paralel selagi rangkaian task ini dikerjakan bertahap (hindari downtime total di tengah pengerjaan multi-task ini).

## Edge Cases
- `Schedule` LAMA (model existing, akan diganti `JadwalSlot` baru di T204) TETAP ADA sampai T215 — JANGAN sentuh/drop di task ini.
- `TeachingSession.scheduleId` (FK ke `Schedule` LAMA) — SAAT INI 0 baris di production (dikonfirmasi), TAPI struktur constraint-nya PERLU DIRENCANAKAN untuk T204 (nanti `TeachingSession` akan refer ke `JadwalSlot` BARU, bukan `Schedule` LAMA) — CATAT sebagai dependency ke T204, JANGAN ubah `TeachingSession` di task ini.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (tambah 7 model baru + 2 enum baru di atas).
- **Buat:** migration Prisma baru.
- **Jangan sentuh:** SEMUA model lama (`Schedule`, `Kelas.modeJadwal`, `Semester.mode`, `BlockWeekRange`, `JamPelajaranOption` dkk, `TeachingSession`) — task ini MURNI ADDITIF, tidak ada DROP/ALTER ke struktur existing.

## Acceptance Criteria
- [x] 7 model baru + 2 enum baru berhasil dibuat via migration, tidak ada error.
- [x] Model lama (Schedule, BlockWeekRange, JamPelajaranOption dkk, Kelas.modeJadwal, Semester.mode) TIDAK TERSENTUH — `git diff` = 106 insertions, **0 deletions**, verified via `git diff | grep "^-"` kosong total.
- [x] Row count production tabel jadwal diverifikasi ULANG tepat sebelum migration (lihat di bawah) — semua 0, konsisten temuan 2026-08-16.
- [x] `prisma migrate status` bersih, tidak ada drift.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] **Baca SEMUA file T203-T215** sebelum mulai — ringkasan pemahaman rangkaian di bawah.
- [x] **Verifikasi ulang row count production** — dijalankan LANGSUNG ke `absensi-mysql-prod` (port 3309, dikonfirmasi via `docker port`, BUKAN dev port 3307) tepat sebelum migration dev dijalankan: `teaching_sessions=0, journal_entries=0, class_attendance_marks=0, grade_assessments=0` — SAMA seperti temuan 2026-08-16, aman lanjut.
- [x] Konfirmasi model lama 100% tidak tersentuh — diverifikasi 2 arah: `git diff` (0 baris hapus) DAN query `information_schema.tables` (semua tabel lama `schedules`/`block_week_ranges`/`jam_pelajaran_options`/`kelas`/`semesters` masih ada di dev DB pasca-migration).

## Status Eksekusi (2026-08-16)

**Selesai.**

### Ringkasan pemahaman rangkaian T203-T215 (dicatat sesuai permintaan)

Rangkaian ini membongkar total modul jadwal lama (`Schedule` type jam_mengajar, `Kelas.modeJadwal`, `Semester.mode`, `BlockWeekRange`, `JamPelajaranOption` dkk — hasil T158/T159/T182/T189) dan menggantinya dengan hierarki baru: **Tahun Ajaran → Semester → Opsi Jadwal (mode Normal/Blok permanen) → Alokasi Waktu (terpisah, reusable lintas Opsi) → JadwalSlot (assignment Kelas+Mapel+Guru-team-teaching per jam)**. Urutan dependency:

- **T203** (task ini) — fondasi schema murni additif: `AlokasiWaktu`/`AlokasiWaktuSlot`/`OpsiJadwal`/`OpsiJadwalTingkat`/`OpsiJadwalMingguGenerate` + enum `ModeJadwal`/`MingguAB`.
- **T204** — schema `JadwalSlot`+`JadwalSlotGuru` (many-to-many team-teaching) + `MapelGuru` (guru wajib terdaftar per mapel sebelum bisa dipilih jadwal). Catatan penting: constraint unique DB `[opsiJadwalId,kelasId,hari,jamKe,minggu]` TIDAK cukup untuk mode normal (`minggu` NULL diperlakukan MySQL sebagai selalu unik) — WAJIB validasi service-layer tambahan di T206.
- **T205** — independen, tambah `Kelas.lantai` + klarifikasi terpisah soal "perbaikan menu Jurusan" (WAJIB tanya user dulu, jangan asumsi).
- **T206** — backend inti: `AlokasiWaktuService`/`OpsiJadwalService`/`JadwalSlotService` dengan 5+ validasi kompleks (minggu sesuai mode, jamKe valid, duplikat mode-normal, guru terdaftar MapelGuru, bentrok guru lintas-OpsiJadwal) + endpoint cek-ketersediaan-guru real-time.
- **T207** — form Mapel (T201) diperluas assignment Guru Pengampu many-to-many, kemungkinan pindah Dialog→Sheet.
- **T208** — Date Generator Minggu A/B: 2 titik mulai berselisih 7 hari → generate tanggal individual skip libur+hari kosong.
- **T209** — `TeachingSession.scheduleId` → `jadwalSlotId`, `generateForDate()` sumber pindah dari `Schedule` ke `JadwalSlot`+`OpsiJadwal` aktif. Perlu klarifikasi user soal team-teaching di level `TeachingSession.teacherId`.
- **T210** — UI menu utama "Jadwal Pelajaran" (hierarki dropdown Tahun Ajaran→Semester→list Opsi Jadwal, Warning Banner kalau aktif tapi induk nonaktif) + halaman Alokasi Waktu Full Page (bukan Sheet lagi).
- **T211** — sub-menu read-only "Tampilan Jadwal" (Per Kelas/Per Guru, HANYA jadwal yang 3-syarat aktif: Opsi+Semester+TahunAjaran) + edit inline.
- **T212** — Workspace input paling kompleks: checklist hari persisten, tab per hari, dropdown Guru real-time cek bentrok, save per-hari dengan auto-advance tab, khusus Blok ada tab Minggu A/B + Date Generator UI.
- **T213** — import CSV/Excel format baru untuk `JadwalSlot` (REUSE validasi T206 persis, best-effort per baris) — pelajaran dari bug T160 lama: JANGAN ada efek samping tersembunyi.
- **T214** — audit+konsolidasi SEMUA pesan error lintas T206-T213 mengikuti format `[apa salah]+[apa yang harus dilakukan]`.
- **T215** — PALING TERAKHIR, DESTRUKTIF, WAJIB konfirmasi eksplisit user dulu — hapus total model+kode+UI lama setelah T203-T214 terverifikasi live.

**Prinsip kunci seluruh rangkaian**: T203-T214 SENGAJA additif (sistem lama & baru paralel selama transisi), hanya T215 yang destruktif dan butuh persetujuan eksplisit terpisah.

### Implementasi

- 7 model + 2 enum ditambahkan PERSIS sesuai spec (`AlokasiWaktu`, `AlokasiWaktuSlot`, `ModeJadwal`, `OpsiJadwal`, `OpsiJadwalTingkat`, `MingguAB`, `OpsiJadwalMingguGenerate`), diletakkan setelah `SchoolHoliday` sebelum section Users.
- Back-relation TAMBAHAN (murni additif, bukan model baru tapi field relasi baru) di `Semester.opsiJadwal` dan `User.alokasiWaktuCreated`/`User.opsiJadwalCreated` — WAJIB secara teknis Prisma (relasi 2 arah), TIDAK mengubah field/behavior existing apa pun di kedua model itu.
- Migration `20260816171005_t203_opsi_jadwal_foundation` — pure `CREATE TABLE`+`ADD CONSTRAINT`, 0 `DROP`/`ALTER MODIFY`.

### Verifikasi

- `git diff apps/api/prisma/schema.prisma`: 106 insertions, 0 deletions.
- Row count production (`absensi-mysql-prod`, port 3309): semua 4 tabel jadwal = 0, dicek LANGSUNG (bukan diasumsikan) tepat sebelum migration dev dijalankan.
- `prisma migrate status`: bersih, tidak ada drift.
- `tsc --noEmit` bersih, `jest apps/api`: 494/494 pass (29/29 suite) — zero regresi dari penambahan model baru.
- Tabel lama (`schedules`, `block_week_ranges`, `jam_pelajaran_options`, `kelas`, `semesters`) dikonfirmasi masih ada di dev DB pasca-migration via query `information_schema.tables`.
