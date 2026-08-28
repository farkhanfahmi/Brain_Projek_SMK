# T247 — API: Schema — Struktur Pengurus Kelas, Jadwal Piket Kelas, Akun Siswa (Fondasi)

## Depends on
Tidak ada. **Ini task FONDASI** — T248 (UI wali kelas), T249 (backend QR), T250 (portal
siswa), T251 (scanner guru) SEMUA bergantung pada model data di task ini. Kerjakan task
ini PALING AWAL dari seri T247-T251.

## Objective
3 kebutuhan data baru: (1) struktur pengurus kelas per tahun ajaran, (2) jadwal piket
kebersihan kelas per hari, (3) kemampuan `User` (akun login) terhubung ke `Student` —
akun login siswa PERTAMA di sistem ini (sebelumnya `Student` murni data, tidak pernah login).

## Konteks Arsitektur Penting

**Ini perubahan besar secara konseptual**: seluruh sistem AbsenSI sejak awal memisahkan
`Student` (data, tap kartu RFID via kiosk, TIDAK PERNAH login) dari `User` (akun staf —
admin/guru/piket/dst, login JWT). Task ini MENJEMBATANI keduanya untuk kasus SEMPIT (Ketua
Kelas + Wakil Ketua Kelas SAJA, bukan semua siswa) — bukan mengubah arsitektur siswa secara
umum. `Student` TETAP tidak bisa login kecuali eksplisit dihubungkan lewat mekanisme di
task ini.

**Keputusan user (2026-08-25)**: kalau Ketua Kelas DAN Wakil Ketua sama-sama absen, akun
salah satu dari mereka BOLEH dipakai/dioper ke pengurus lain SECARA INFORMAL oleh siswa
sendiri (bukan fitur delegasi di aplikasi) — sistem TIDAK PERLU bangun mekanisme
reassignment otomatis. Konsekuensi: sistem tidak pernah tahu PERSIS siapa individu yang
menekan tombol (cuma tahu akun mana) — ini trade-off yang SUDAH DISADARI dan DITERIMA user,
JANGAN dianggap sebagai bug/celah yang perlu "diperbaiki" di task manapun nanti.

## Spec Detail

### 1. Model `KelasPengurus` (struktur pengurus, histori per tahun ajaran)

```prisma
enum JabatanPengurus {
  ketua
  wakil_ketua
  sekretaris
  wakil_sekretaris
  bendahara
  wakil_bendahara
}

model KelasPengurus {
  id             Int             @id @default(autoincrement())
  kelasId        Int             @map("kelas_id")
  studentId      Int             @map("student_id")
  jabatan        JabatanPengurus
  academicYearId Int             @map("academic_year_id")
  createdById    Int             @map("created_by")
  createdAt      DateTime        @default(now()) @map("created_at")

  kelas        Kelas        @relation(fields: [kelasId], references: [id])
  student      Student      @relation(fields: [studentId], references: [id])
  academicYear AcademicYear @relation(fields: [academicYearId], references: [id])
  createdBy    User         @relation(fields: [createdById], references: [id])

  @@unique([kelasId, jabatan, academicYearId])
  @@index([studentId])
  @@map("kelas_pengurus")
}
```
- **Histori per tahun ajaran** (keputusan user) — ganti pengurus tahun baru = baris BARU
  dengan `academicYearId` baru, baris lama TETAP ADA (tidak dihapus/di-update) untuk arsip.
  Dalam 1 tahun ajaran yang SAMA, mengganti 1 jabatan = UPDATE baris existing (bukan bikin
  baris baru terus tiap wali kelas ganti pikiran) — `@@unique([kelasId, jabatan, academicYearId])`
  menjamin ini.
- Jabatan opsional (wakil_ketua/wakil_sekretaris/wakil_bendahara) — boleh TIDAK ADA baris
  sama sekali untuk kombinasi kelas+jabatan+tahun itu (bukan baris dengan `studentId` null).

### 2. Model jadwal piket kelas — NAMA HARUS BEDA dari `PiketSchedule` existing

**PENTING**: `PiketSchedule` (`schema.prisma:1393-1406`) SUDAH ADA tapi itu untuk jadwal
piket STAF (`guru_piket`, per `User.id`) — konsep BEDA TOTAL dari piket kebersihan kelas
(siswa). JANGAN reuse/perluas model itu — bikin model baru dengan nama yang TIDAK
membingungkan dengan yang lama:

```prisma
model KelasPiketJadwal {
  id        Int @map("id") @id @default(autoincrement())
  kelasId   Int @map("kelas_id")
  hari      Int // 1=Senin..6=Sabtu, KONSISTEN konvensi PiketSchedule.hari existing
  studentId Int @map("student_id")

  kelas   Kelas   @relation(fields: [kelasId], references: [id])
  student Student @relation(fields: [studentId], references: [id])

  @@unique([kelasId, hari, studentId])
  @@index([kelasId, hari])
  @@map("kelas_piket_jadwal")
}
```
- 1 baris = 1 siswa di 1 hari (flat, KONSISTEN pola `PiketSchedule` yang juga flat
  hari+userId tanpa tabel "hari" terpisah) — 1 hari bisa banyak siswa (banyak baris hari
  sama, studentId beda).
- TIDAK di-tag `academicYearId`/semester — ini murni jadwal OPERASIONAL saat ini (state
  "sekarang"), BUKAN data transaksional historis (beda dari `KelasPengurus` yang sengaja
  minta histori) — wali kelas timpa langsung kalau mau ganti, tanpa arsip. VERIFIKASI ke
  user kalau ternyata histori piket kelas juga diinginkan nanti (DI LUAR SCOPE task ini
  kalau tidak diminta eksplisit).

### 3. Akun login siswa — extend `User`

```prisma
// Tambahan ke model User existing:
studentId Int? @map("student_id") // dual-FK pola SAMA seperti teacherId existing

student User_student_relation? // nama relasi VERIFIKASI SAAT IMPLEMENTASI, hindari
                                 // konflik nama dengan relasi existing di model User
```
Tambah value baru ke `UserRole` enum: `ketua_kelas` (dipakai SAMA untuk Ketua MAUPUN Wakil
Ketua — keduanya statusnya setara dari sisi akses aplikasi, TIDAK PERLU role terpisah kecuali
ditemukan kebutuhan permission berbeda saat implementasi T250).

**Constraint penting**: `User.studentId` HANYA boleh terisi kalau `User.role === ketua_kelas`
DAN siswa itu benar-benar tercatat di `KelasPengurus` sebagai `ketua`/`wakil_ketua` kelas
aktif tahun ajaran berjalan — VALIDASI INI DI SERVICE LAYER (Prisma tidak bisa enforce cross-
table constraint kompleks begini), bukan cuma di level schema.

**Provisioning akun** — REPLIKASI pola `generateAkunGuruMassal()` (T232: username dari
identifier unik, password default, `mustChangePassword: true`) — untuk siswa pakai NISN
sebagai username (siswa tidak punya NIY). Endpoint provisioning JADI BAGIAN T248 (dipicu
dari UI struktur pengurus: begitu wali kelas assign siswa ke jabatan Ketua/Wakil Ketua,
sistem OTOMATIS provision akun kalau belum ada — VERIFIKASI SAAT IMPLEMENTASI T248 alur
persis, disebut di sini supaya schema-nya sudah siap).

## Edge Cases
- **Siswa yang SAMA jadi Ketua 2 tahun ajaran berturut-turut** — `KelasPengurus` histori
  tetap benar (2 baris beda `academicYearId`), akun `User.studentId` yang SAMA cukup 1x
  provision (tidak perlu akun baru tiap tahun kalau orangnya sama — VERIFIKASI SAAT
  IMPLEMENTASI logic "sudah punya akun apa belum" saat assign ulang).
- **Siswa diganti dari Ketua ke jabatan lain di tahun yang sama** (mis. wali kelas salah
  assign, ganti lagi) — akun `ketua_kelas` yang sudah dibuat untuk siswa lama HARUS
  di-nonaktifkan (`User.status = nonaktif`), BUKAN dihapus (jejak audit) — provisioning
  ulang untuk siswa baru yang di-assign.
- **Siswa nonaktif/lulus yang kebetulan pernah jadi pengurus** — histori `KelasPengurus`
  TETAP ada (arsip), tapi akun `User`-nya (kalau ada) harus ikut `nonaktif` mengikuti
  status `Student.status` (VERIFIKASI SAAT IMPLEMENTASI — mungkin sudah otomatis kalau ada
  job/logic existing yang menonaktifkan akun siswa lulus, kalau belum ada perlu ditambah).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (3 penambahan: `KelasPengurus`,
  `KelasPiketJadwal`, extend `User`+`UserRole`).
- **Buat:** migration Prisma baru (ADDITIVE — `CREATE TABLE`+`ADD COLUMN`, TIDAK ada
  `DROP`, aman commit tanpa checklist backup-destruktif di CLAUDE.md).

## Acceptance Criteria
- [x] Migration jalan bersih tanpa error di DB dev (lihat catatan implementasi soal
      `prisma migrate dev` interaktif di bawah).
- [x] `KelasPengurus` bisa insert dengan histori multi-tahun-ajaran — diverifikasi live
      via script Prisma langsung.
- [x] `KelasPiketJadwal` bisa insert banyak siswa per hari — diverifikasi live.
- [x] `User.studentId` bisa terisi, relasi ke `Student` bekerja — diverifikasi live
      (`user.student.nama` ter-resolve benar).
- [x] Prisma Client ter-generate ulang tanpa error type — `tsc --noEmit` api+web bersih.

## Validasi Claudian
- [x] Konfirmasi TIDAK ada konflik nama model/field dengan yang sudah ada — `PiketSchedule`
      TIDAK disentuh sama sekali, `KelasPiketJadwal` nama BEDA jelas (grep dilakukan
      sebelum implementasi).
- [x] Konfirmasi migration ADDITIVE murni — `migration.sql` HANYA `ADD COLUMN`+`CREATE TABLE`
      +`AddForeignKey`, TIDAK ADA DROP sama sekali (dicek manual isi file sebelum apply).
- [x] Konfirmasi enum `UserRole` baru (`ketua_kelas`) tidak bentrok — grep
      `Object.values(UserRole)`/`UserRole[]` di seluruh api+web SEBELUM implementasi,
      TIDAK ADA kode yang meng-iterate seluruh enum tanpa whitelist eksplisit (semua
      pemakaian sudah whitelist manual per-role), jadi penambahan role baru AMAN nol dampak
      ke kode existing manapun.

## Catatan Implementasi: `prisma migrate dev` Gagal di Lingkungan Non-Interaktif (2026-08-26)

`prisma migrate dev` (dan `--create-only`) MENOLAK jalan sama sekali di shell tool ini
("environment is non-interactive") — bukan soal warning yang perlu di-approve, CLI Prisma
memang mendeteksi tidak ada TTY dan refuse total, tidak ada flag `--force`/`--yes` untuk
lewati ini. **Solusi yang dipakai** (alternatif resmi Prisma untuk kasus ini, BUKAN
workaround berbahaya):
1. `prisma migrate diff --from-migrations ./prisma/migrations --to-schema-datamodel ./prisma/schema.prisma --shadow-database-url <db_kosong_temporer> --script` — generate SQL diff murni dari schema saat ini vs migration history, TANPA butuh interaktif.
2. Buat folder migration manual (`prisma/migrations/<timestamp>_t247_...>/migration.sql`) isi SQL hasil poin 1 — PERSIS format yang `prisma migrate dev` akan hasilkan sendiri.
3. `prisma migrate deploy` (didesain non-interaktif untuk CI/CD) — apply migration itu ke DB dev.
4. Database shadow temporer dibuat+dihapus manual via `docker exec ... mysql -e "CREATE/DROP DATABASE"`.

Hasil akhir identik dengan `prisma migrate dev` normal (folder migration timestamped +
SQL additif murni), cuma jalur eksekusinya beda karena keterbatasan environment ini —
BUKAN kompromi terhadap keamanan/kualitas migration.

## Implementasi (2026-08-26)

Schema: `JabatanPengurus` enum baru (6 nilai), `KelasPengurus` model baru (histori per
tahun ajaran, `@@unique([kelasId,jabatan,academicYearId])`), `KelasPiketJadwal` model baru
(flat per-hari, TIDAK di-tag academicYear/semester sesuai spec — state operasional
"sekarang"), `User.studentId` (nullable, `@unique` — 1 siswa maksimal 1 akun) + relasi ke
`Student`, `UserRole.ketua_kelas` baru. Relasi balik ditambahkan ke `Kelas`/`Student`/
`AcademicYear`/`User` (createdBy). Frontend `UserRole` type + `USER_ROLE_LABEL` (core-types.ts)
diperluas sekalian (exhaustiveness check TypeScript `Record<UserRole,string>` akan gagal
compile kalau lupa) — belum dipakai UI manapun (T248 nanti). Constraint cross-table
(`studentId` hanya untuk `ketua_kelas` yang tercatat `KelasPengurus` aktif) SENGAJA belum
divalidasi di task ini — didelegasikan ke T248 sesuai spec (Prisma tidak bisa enforce ini
di level schema, butuh service layer yang belum ada endpoint-nya).
