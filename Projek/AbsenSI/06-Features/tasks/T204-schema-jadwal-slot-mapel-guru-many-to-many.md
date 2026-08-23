# T204 — Schema: JadwalSlot Baru (Menggantikan Schedule) + Mapel↔Guru Many-to-Many (Team Teaching)

## Depends on
**WAJIB SETELAH T203** (butuh model `OpsiJadwal`, `AlokasiWaktuSlot` sudah ada). Bagian dari rangkaian T203-T215.

## Objective
1. Model baru `JadwalSlot` — menggantikan `Schedule` (type `jam_mengajar` saja — `jam_sekolah`/`jadwal_khusus` TIDAK terdampak rangkaian ini, TETAP pakai `Schedule` lama, lihat Edge Cases) — 1 baris = assignment Kelas+Mapel+Guru(banyak, team-teaching)+JamKe pada 1 Opsi Jadwal.
2. Model baru `MapelGuru` — many-to-many Mapel↔Teacher, WAJIB diisi sebelum guru bisa dipilih di form input jadwal (T212).

## Keputusan Final Dikonfirmasi User (2026-08-16)

1. **Team Teaching**: 1 `JadwalSlot` bisa punya BANYAK guru sekaligus (many-to-many ke Teacher, BUKAN 1 `teacherId` tunggal seperti `Schedule` lama).
2. **`Mapel↔Teacher` many-to-many WAJIB diisi dulu** — guru harus terdaftar sebagai pengampu Mapel tertentu SEBELUM bisa dipilih di dropdown Guru saat input jadwal untuk Mapel itu (form T212 filter dropdown Guru HANYA menampilkan guru yang terdaftar).
3. Ruangan/Kampus/Lantai **TIDAK diinput di form jadwal** — selalu diwariskan (join) dari `Kelas`, TIDAK ADA kolom ruangan di `JadwalSlot`.

## Spec Detail

### 1. Model Prisma baru

```prisma
// Menggantikan relasi Mapel<->Teacher yang SEBELUMNYA cuma tersirat lewat Schedule/
// TeachingSession — sekarang JADI assignment eksplisit, WAJIB diisi sebelum guru itu
// bisa dipilih di form input jadwal untuk Mapel tersebut (lihat T212).
model MapelGuru {
  id        Int @id @default(autoincrement())
  mapelId   Int @map("mapel_id")
  teacherId Int @map("teacher_id")

  mapel   Mapel   @relation(fields: [mapelId], references: [id], onDelete: Cascade)
  teacher Teacher @relation(fields: [teacherId], references: [id])

  @@unique([mapelId, teacherId])
  @@index([teacherId])
  @@map("mapel_guru")
}

// Menggantikan Schedule (KHUSUS jam_mengajar) — Schedule type jam_sekolah/jadwal_khusus
// TIDAK terdampak, TETAP pakai model Schedule lama (domain berbeda total, T145).
model JadwalSlot {
  id           Int      @id @default(autoincrement())
  opsiJadwalId Int      @map("opsi_jadwal_id")
  kelasId      Int      @map("kelas_id")
  mapelId      Int      @map("mapel_id")
  hari         Int      // 1=Minggu..7=Sabtu, KONSISTEN AlokasiWaktuSlot.hari
  jamKe        Int      @map("jam_ke") // WAJIB terisi (beda dari AlokasiWaktuSlot.jamKe yang
                                        // nullable utk baris istirahat -- JadwalSlot TIDAK
                                        // PERNAH merujuk baris istirahat, hanya jam pelajaran)
  minggu       MingguAB? // NULL utk OpsiJadwal.mode=normal, WAJIB terisi utk mode=blok
                          // (validasi service layer, KONSISTEN pola resolveMinggu() lama)
  createdById  Int      @map("created_by")
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  opsiJadwal OpsiJadwal        @relation(fields: [opsiJadwalId], references: [id])
  kelas      Kelas             @relation(fields: [kelasId], references: [id])
  mapel      Mapel             @relation(fields: [mapelId], references: [id])
  guru       JadwalSlotGuru[]  // many-to-many team-teaching
  createdBy  User              @relation(fields: [createdById], references: [id])

  @@unique([opsiJadwalId, kelasId, hari, jamKe, minggu])
  @@index([opsiJadwalId, kelasId])
  @@index([opsiJadwalId, hari])
  @@map("jadwal_slots")
}

model JadwalSlotGuru {
  id           Int @id @default(autoincrement())
  jadwalSlotId Int @map("jadwal_slot_id")
  teacherId    Int @map("teacher_id")

  jadwalSlot JadwalSlot @relation(fields: [jadwalSlotId], references: [id], onDelete: Cascade)
  teacher    Teacher    @relation(fields: [teacherId], references: [id])

  @@unique([jadwalSlotId, teacherId])
  @@index([teacherId])
  @@map("jadwal_slot_guru")
}
```

- **`jamKe` di `JadwalSlot` WAJIB integer, MERUJUK ke `jamKe` yang ADA di `AlokasiWaktuSlot`** milik `AlokasiWaktu` yang dirujuk `OpsiJadwal` terkait — VALIDASI ini di service layer (T206), BUKAN foreign key langsung (karena `jamKe` bukan PK `AlokasiWaktuSlot`, hubungannya by-value bukan by-reference — REPLIKASI pola `resolveJamSchedule()` lama yang lookup by `hari+jamKe`, JANGAN buat FK yang tidak masuk akal secara data).
- **`@@unique([opsiJadwalId, kelasId, hari, jamKe, minggu])`** — mencegah 1 kelas punya 2 mapel di jam yang SAMA hari yang SAMA dalam Opsi Jadwal yang SAMA (constraint DB level, bukan cuma service-layer check) — TAPI CATATAN: `minggu` NULL (mode normal) — MySQL unique constraint MEMPERLAKUKAN NULL sebagai "tidak sama dengan NULL manapun" (setiap kombinasi dengan NULL dianggap unik) — JADI constraint DB ini **TIDAK CUKUP** untuk mode normal, WAJIB tambahan validasi service-layer eksplisit (T206) untuk cegah duplikat kelas-hari-jamKe saat `minggu IS NULL`.

### 2. Field `MapelGuru` — endpoint & UI (lihat T207 untuk detail lengkap form Mapel)

- Task ini HANYA schema — endpoint CRUD `MapelGuru` dan UI form-nya dikerjakan di **T207** (bersamaan dengan redesign form Mapel yang sudah ada dari T201).

## Edge Cases
- `Schedule` (model lama, type jam_sekolah/jadwal_khusus) — **TIDAK TERDAMPAK SAMA SEKALI** oleh task ini, domain jam masuk sekolah 3-lapis (T145) TETAP pakai `Schedule` seperti sekarang, JANGAN campur/migrasikan ke `JadwalSlot` baru (JadwalSlot HANYA untuk jam_mengajar).
- Kelas yang PUNYA `JadwalSlot` di 2 `OpsiJadwal` BERBEDA yang KEDUANYA aktif bersamaan (skenario tidak wajar tapi TIDAK dicegah schema) — VALIDASI ini di T210 (logic aktivasi Opsi Jadwal), BUKAN di task ini.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (tambah `MapelGuru`, `JadwalSlot`, `JadwalSlotGuru`).
- **Buat:** migration Prisma baru.
- **Jangan sentuh:** `Schedule` (model lama, TIDAK didrop/diubah — domain jam_sekolah/jadwal_khusus tetap pakai ini), `TeachingSession.scheduleId` (FK ke `Schedule` lama, TIDAK diubah di task ini — akan diubah ke `jadwalSlotId` di **T209**, task terpisah, SETELAH `JadwalSlot` benar-benar bisa dipakai end-to-end).

## Acceptance Criteria
- [x] `MapelGuru`, `JadwalSlot`, `JadwalSlotGuru` berhasil dibuat via migration.
- [x] Constraint `@@unique([opsiJadwalId, kelasId, hari, jamKe, minggu])` dibuat sesuai spec (mode blok, minggu terisi, akan terbukti bekerja saat T206 mengujinya end-to-end) — DICATAT eksplisit TIDAK CUKUP untuk mode normal (lihat Validasi Claudian).
- [x] `Schedule` (model lama) TIDAK tersentuh — verified `git diff` (0 deletions) + `SHOW COLUMNS FROM schedules` pasca-migration identik.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] **Dicatat eksplisit**: constraint unique DB (`@@unique([opsiJadwalId, kelasId, hari, jamKe, minggu])`) TIDAK CUKUP untuk mode normal — `minggu` NULL diperlakukan MySQL sebagai "tidak sama dengan NULL manapun", jadi tiap baris ber-`minggu` NULL (mode normal) dianggap unik meski `kelasId+hari+jamKe` sama. **T206 WAJIB tambah validasi service-layer eksplisit** (query `findFirst` cek duplikat manual sebelum create) untuk menutup celah ini — belum dikerjakan di task ini (murni schema).
- [x] Konfirmasi `Schedule` lama (jam_sekolah/jadwal_khusus) sama sekali tidak tersentuh — 0 baris diff terhapus, kolom tabel `schedules` identik sebelum/sesudah.

## Catatan Implementasi (2026-08-16)

- Row count production diverifikasi ULANG langsung (bukan diasumsikan dari T203) tepat sebelum migration dev: `teaching_sessions=0, journal_entries=0, class_attendance_marks=0, grade_assessments=0` (konsisten temuan T203), `schedules=18` (tidak relevan ke task ini, model itu tidak disentuh).
- 3 model ditambahkan PERSIS sesuai spec, diletakkan setelah `OpsiJadwalMingguGenerate` sebelum section Users.
- Back-relation additif WAJIB secara teknis Prisma (relasi 2 arah): `Mapel.guruPengampu`+`Mapel.jadwalSlots`, `Teacher.mapelPengampu`+`Teacher.jadwalSlotGuru`, `Kelas.jadwalSlots`, `OpsiJadwal.jadwalSlots`, `User.jadwalSlotCreated` — semua murni tambah field relasi baru, TIDAK mengubah field/behavior existing model manapun.
- Migration `20260816171505_t204_jadwal_slot_mapel_guru` — pure `CREATE TABLE`+`ADD CONSTRAINT`, 0 `DROP`/`ALTER MODIFY`.

### Verifikasi

- `git diff apps/api/prisma/schema.prisma`: 176 insertions, 0 deletions.
- `prisma migrate status`: bersih, tidak ada drift (51 migrations).
- `tsc --noEmit` bersih, `jest apps/api`: 494/494 pass (29/29 suite) — zero regresi.
- `nest build` sukses.
- Tabel lama (`schedules`, `block_week_ranges`, `jam_pelajaran_options`, `kelas`, `semesters`, `mapel`, `teachers`) dikonfirmasi masih ada + struktur kolom `schedules` identik via `information_schema.tables` + `SHOW COLUMNS`.
