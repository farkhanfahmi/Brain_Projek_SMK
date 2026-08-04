# T096 — Schema+API+UI: Absensi Ekstrakurikuler oleh Pembina

## Depends on
Tidak ada — menambah model baru (`EkstraSesi`, `EkstraAbsen`) dan field baru di `Ekstrakurikuler`/`User`, tidak mengubah `EkstraPendaftaran`/pendaftaran publik yang sudah live.

## Context
- **App:** `apps/api` + `apps/web`
- **Ref:** Feature — Ekstrakurikuler (06-Features/ekstrakurikuler.md) (diperbarui 2026-07-30, baca dulu bagian "Yang Sudah Ada"). Diskusi desain lengkap dengan user 2026-07-30, semua keputusan di bawah sudah dikonfirmasi eksplisit — JANGAN tanya ulang hal yang sudah diputuskan di sini, hanya klarifikasi kalau menemukan kondisi yang benar-benar belum tercakup.
- **Pola arsitektur yang ditiru** (sudah teruji di produksi, ikuti strukturnya): `Schedule` → `TeachingSession` → `JournalEntry` + `ClassAttendanceMark` (lihat `apps/api/prisma/schema.prisma` baris ±327-395). Ekstrakurikuler analog tapi TANPA `Schedule` (ekstra tidak instrumentasi jadwal terstruktur) — sesi dibuat manual oleh pembina, bukan auto-generate harian.

## Keputusan Final (dikonfirmasi user, 2026-07-30)

1. **Mekanisme absen: manual oleh pembina** — bukan tap kartu. Pembina buka sesi, tandai status tiap siswa satu-satu.
2. **Sesi terjadwal, dibuat manual** — pembina yang membuat `EkstraSesi` (pilih tanggal + catatan opsional) kapan pun dia mau, TIDAK ada auto-generate dari jadwal (ekstra tidak punya `Schedule` terstruktur seperti jam mengajar guru).
3. **Pembina — dua jalur akun berbeda:**
   - **Guru existing** yang jadi pembina → tetap pakai akun `guru` yang sudah ada, TIDAK dapat akun baru. Menu "Ekstrakurikuler" MUNCUL TAMBAHAN di sidebar guru — pola identik Wali Kelas (`kelasIdWali` di `apps/web/src/app/(guru)/guru-sidebar.tsx` baris 14, 25, 28: menu cuma render kalau flag true).
   - **Pembina eksternal** (bukan guru sekolah) → role BARU `pembina_ekstra` di enum `UserRole`, akun dibuatkan admin lewat halaman Manajemen Akun (existing, `apps/web/src/app/(admin)/akun/akun-view.tsx`), dashboard TERPISAH dengan sidebar sendiri (pola sama seperti `PiketSidebar`/`GuruSidebar` — scope sempit, tidak akses modul lain).
4. **1 ekstrakurikuler = tepat 1 pembina** (bukan many-to-many) — `Ekstrakurikuler.pembinaId` cukup FK tunggal nullable ke `User`, BUKAN ke `Teacher` (supaya bisa merujuk baik akun guru maupun akun `pembina_ekstra` eksternal — akun `User` sudah punya `teacherId` nullable, pola yang sama dipakai di sini: `pembinaId` merujuk `User`, bukan `Teacher`).
5. **Peserta sesi = snapshot dari `EkstraPendaftaran` SAAT sesi dibuat** — bukan query dinamis. Begitu `EkstraSesi` dibuat, sistem langsung insert baris `EkstraAbsen` untuk SEMUA siswa yang punya `EkstraPendaftaran.ekstrakurikulerId` cocok pada saat itu, dengan `status: null` (lihat poin 6). Sesi yang SUDAH dibuat TIDAK berubah kalau ada siswa baru daftar/batal setelahnya — hanya sesi BARU yang ikut daftar peserta terkini.
6. **Status kehadiran: Hadir / Izin / Sakit / Alfa, default KOSONG** (bukan salah satu dari 4 — `EkstraAbsen.status` nullable, artinya "belum ditandai"). UI per baris siswa: 3 tombol warna — **Hijau = Hadir**, **Merah = Alfa**, **Kuning = Izin/Sakit** (satu tombol gabungan, klik membuka sub-pilihan Izin vs Sakit + wajib upload bukti).
7. **Izin/Sakit WAJIB upload bukti foto** (dari galeri atau kamera langsung, `<input type="file" accept="image/*" capture>`) — pola backend SAMA PERSIS dengan `TeacherPermit.buktiFilePath` (lihat `apps/api/src/teacher-permits/teacher-permits.controller.ts` — `FileInterceptor("bukti_file")`, `ParseFilePipeBuilder`, simpan path relatif, endpoint `GET .../bukti-file` untuk baca kembali). Hadir dan Alfa TIDAK butuh bukti.
8. **Super Admin TIDAK BISA ubah data absensi ekstra** — domain eksklusif pembina, pola sama ADR-019 (guru_piket eksklusif untuk status kehadiran siswa gerbang). Admin cuma CRUD `Ekstrakurikuler` (sudah ada) dan assign `pembinaId` (baru).

## Spec Detail — Skema Baru

```prisma
enum UserRole {
  super_admin
  card_admin
  guru
  kepsek
  guru_piket
  admin_jurnal
  pembina_ekstra // BARU — akun pembina eksternal (bukan guru), scope sempit ke ekstra miliknya
}

model Ekstrakurikuler {
  id         Int    @id @default(autoincrement())
  nama       String @unique
  urutan     Int    @default(0)
  pembinaId  Int?   @unique @map("pembina_id") // BARU — 1 ekstra = tepat 1 pembina, FK ke User (bukan Teacher)

  pembina     User?               @relation(fields: [pembinaId], references: [id])
  pendaftaran EkstraPendaftaran[]
  sesi        EkstraSesi[]
}

enum EkstraAbsenStatus {
  hadir
  izin
  sakit
  alfa
}

model EkstraSesi {
  id                Int      @id @default(autoincrement())
  ekstrakurikulerId Int      @map("ekstrakurikuler_id")
  tanggal           DateTime @db.Date
  catatan           String?  @db.Text // materi/kegiatan, opsional
  createdById       Int      @map("created_by")
  createdAt         DateTime @default(now()) @map("created_at")

  ekstrakurikuler Ekstrakurikuler @relation(fields: [ekstrakurikulerId], references: [id])
  createdBy       User            @relation(fields: [createdById], references: [id])
  absen           EkstraAbsen[]

  @@index([ekstrakurikulerId, tanggal])
  @@map("ekstra_sesi")
}

model EkstraAbsen {
  id            Int                @id @default(autoincrement())
  sesiId        Int                @map("sesi_id")
  studentId     Int                @map("student_id")
  status        EkstraAbsenStatus? // NULL = belum ditandai (default saat sesi dibuat)
  buktiFilePath String?            @map("bukti_file_path") // WAJIB terisi kalau status izin/sakit, NULL untuk hadir/alfa/belum ditandai
  markedById    Int?               @map("marked_by") // NULL sampai pembina benar-benar menandai
  markedAt      DateTime?          @map("marked_at")

  sesi    EkstraSesi @relation(fields: [sesiId], references: [id])
  student Student    @relation(fields: [studentId], references: [id])
  markedBy User?     @relation(fields: [markedById], references: [id])

  @@unique([sesiId, studentId])
  @@map("ekstra_absen")
}
```

- Tambahkan relasi balik yang perlu di `User` (`ekstrakurikulerDibina Ekstrakurikuler?`, `ekstraSesiCreated EkstraSesi[]`, `ekstraAbsenMarked EkstraAbsen[]`) dan di `Student` (`ekstraAbsen EkstraAbsen[]`) sesuai kebutuhan Prisma relation.
- **Cek dulu** apakah nama migrasi/kolom bentrok dengan sesuatu yang sudah ada sebelum `prisma migrate dev` (terutama pastikan `Ekstrakurikuler` belum ada model lain yang pakai nama field sama).

## Business Rules

1. Pembina hanya bisa buat sesi & tandai absen untuk ekstra yang `pembinaId`-nya = akun dia sendiri — ditegakkan di GUARD BACKEND (bukan cuma disembunyikan di UI), pola sama `PiketOnDutyGuard`. Buat guard baru `EkstraPembinaGuard` yang cek `ekstrakurikuler.pembinaId === user.sub` untuk semua endpoint mutasi (`POST sesi`, `PATCH absen`).
2. Endpoint create sesi HARUS insert semua baris `EkstraAbsen` (status null) untuk peserta terdaftar SAAT ITU dalam satu transaksi (`prisma.$transaction`) — jangan sesi kebuat tapi baris absen menyusul terpisah (race condition risk).
3. Update status ke `izin`/`sakit` TANPA `buktiFilePath` harus ditolak (400) — validasi di service, bukan cuma di DTO (kalau pakai multipart, file bisa jadi optional di level HTTP tapi wajib di level logic sesuai status).
4. Guru dengan akun `guru` yang JUGA pembina ekstra — flag `isPembinaEkstra` dikirim ke frontend via endpoint "current user"/"me" (cek pola `isWaliKelas` yang sudah ada, kemungkinan di `apps/api/src/auth/...` atau endpoint sejenis `GET /users/me`), dipakai `GuruSidebar` untuk render menu tambahan.
5. Role `pembina_ekstra` — buat sidebar & layout baru (`apps/web/src/app/(pembina-ekstra)/...`, pola folder route group sama seperti `(piket)`), scope guard backend: semua endpoint ekstra-absensi menerima baik role `guru` (kalau dia pembina) MAUPUN role `pembina_ekstra`, ditentukan lewat `Ekstrakurikuler.pembinaId`, BUKAN lewat role check semata.

## Edge Cases

- Siswa yang sudah dicatat di sesi lama lalu MEMBATALKAN pendaftaran (`DELETE .../pendaftaran`, fitur existing) — baris `EkstraAbsen` di sesi LAMA tetap ada apa adanya (histori tidak diubah), tapi siswa itu TIDAK muncul lagi di sesi BARU yang dibuat setelah pembatalan.
- Siswa pindah ekstra (batal lalu daftar ekstra lain) — sama seperti di atas, histori sesi lama di ekstra A tetap utuh, sesi baru di ekstra A tidak lagi menyertakan siswa itu, dan siswa itu baru muncul di sesi BARU ekstra B (bukan retroaktif ke sesi lama ekstra B yang sudah ada sebelum dia daftar).
- Ekstra TANPA pembina (`pembinaId` null, ekstra existing sebelum fitur ini) — TIDAK BISA dipakai untuk membuat sesi (validasi 400 "Ekstrakurikuler ini belum punya pembina"), admin harus assign pembina dulu.
- Sesi dengan 0 peserta terdaftar (ekstra baru, belum ada yang daftar) — sesi tetap bisa dibuat, tabel absen kosong, tampilkan pesan "Belum ada siswa terdaftar di ekstra ini".
- Ganti status dari `izin` (dengan bukti) ke `hadir` — `buktiFilePath` lama boleh dibiarkan (tidak perlu dihapus file-nya, insert-only-ish, konsisten dengan filosofi audit trail di seluruh sistem) tapi tidak lagi ditampilkan sebagai bukti aktif karena status sudah bukan izin/sakit.

## Files

- **Buat (backend):**
  - Migration Prisma untuk model/enum baru di atas
  - `apps/api/src/ekstra-absensi/` — module baru: `ekstra-absensi.module.ts`, `ekstra-absensi.service.ts`, `ekstra-absensi.controller.ts`, DTO (`create-sesi.dto.ts`, `mark-absen.dto.ts`)
  - `apps/api/src/ekstra-absensi/guards/ekstra-pembina.guard.ts`
- **Modifikasi (backend):**
  - `apps/api/prisma/schema.prisma` — model/enum baru + relasi balik
  - `apps/api/src/core/jurusan/...` tidak relevan; cek `apps/api/src/ekstra-publik/ekstra-kurikuler.controller.ts` — endpoint admin assign `pembinaId` (PATCH ekstra existing, tambah field)
  - `apps/api/src/app.module.ts` — import module baru
  - Endpoint "current user"/`GET /users/me` (cari file yang return `isWaliKelas` sekarang) — tambah `isPembinaEkstra` + `ekstrakurikulerDibinaId` kalau relevan
- **Buat (frontend):**
  - `apps/web/src/app/(guru)/guru/ekstrakurikuler/` — halaman untuk guru yang jadi pembina
  - `apps/web/src/app/(pembina-ekstra)/` — route group BARU lengkap (layout, sidebar) untuk role eksternal, pola sama `(piket)/`
  - Komponen form tandai absen (3 tombol warna + sub-dialog upload bukti izin/sakit)
- **Modifikasi (frontend):**
  - `apps/web/src/app/(guru)/guru-sidebar.tsx` — tambah menu "Ekstrakurikuler" kondisional (pola `WALI_KELAS_NAV`)
  - `apps/web/src/app/(admin)/ekstra-kurikuler/ekstra-kurikuler-view.tsx` — tambah kolom/aksi assign pembina per ekstra
  - `apps/web/src/lib/core-types.ts` — tipe baru (`EkstraSesi`, `EkstraAbsen`, dst)
  - `apps/web/src/middleware.ts` — role `pembina_ekstra` perlu redirect/route rule baru (pola sama `guru_piket`/`guru`)
- **Jangan sentuh:** `apps/api/src/ekstra-publik/ekstra-publik.service.ts` (pendaftaran publik, sudah live, di luar scope task ini), `EkstraPendaftaran` model (tidak berubah).
- **⚠️ Kalau butuh ubah `packages/types` (shared):** WAJIB stop dan minta konfirmasi user dulu.

## Acceptance Criteria

- [ ] Admin bisa assign `pembinaId` ke ekstrakurikuler existing (baik memilih akun guru maupun akun `pembina_ekstra`)
- [ ] Guru yang di-assign sebagai pembina melihat menu "Ekstrakurikuler" baru di sidebar guru-nya (guru lain yang bukan pembina TIDAK melihat menu ini)
- [ ] Akun `pembina_ekstra` baru bisa login, langsung diarahkan ke dashboard ekstra miliknya, tidak bisa akses modul lain
- [ ] Pembina bisa buat sesi baru (pilih tanggal), sistem otomatis snapshot semua peserta terdaftar saat itu ke baris absen dengan status kosong
- [ ] Tabel sesi menampilkan 3 tombol warna per siswa (Hijau/Merah/Kuning), klik Kuning membuka sub-pilihan Izin/Sakit + wajib upload foto sebelum tersimpan
- [ ] Submit Izin/Sakit tanpa file ditolak dengan pesan jelas
- [ ] Super admin TIDAK bisa mengakses/mengubah endpoint mutasi absensi ekstra (403)
- [ ] Sesi yang dibuat SEBELUM siswa baru daftar tidak ikut menampilkan siswa itu; sesi yang dibuat SESUDAHNYA menampilkan siswa itu
- [ ] Ekstra tanpa pembina tidak bisa dipakai membuat sesi (pesan error jelas ke admin/pembina)
- [ ] Build + type-check hijau di `apps/api` dan `apps/web`

## Validasi Claudian
- [ ] Semua keputusan besar (mekanisme absen, akun pembina, jumlah pembina per ekstra, sumber peserta, status default, wajib bukti) SUDAH dikonfirmasi user 2026-07-30 — tidak perlu tanya ulang
- [ ] Scope tidak digabung dengan hal lain (rekap kehadiran ekstra, nilai ekskul — itu di luar scope T096, jangan tergoda menambah)
- [ ] Baca ulang `apps/api/prisma/schema.prisma` model `TeachingSession`/`ClassAttendanceMark`/`JournalEntry` SEBELUM menulis migration — pastikan pola FK/index konsisten
- [ ] Baca ulang `apps/api/src/teacher-permits/` SEBELUM implementasi upload bukti — reuse pola `FileInterceptor`/`ParseFilePipeBuilder`, jangan reinvent
- [ ] Cek dulu bug `isActive` di `apps/web/src/app/(guru)/guru-sidebar.tsx` baris 53 (sama seperti T094 (S.md) di piket) SEBELUM menambah menu baru — kalau belum diperbaiki, menu Dashboard/Jurnal guru akan salah highlight begitu menu "Ekstrakurikuler" ditambahkan, pertimbangkan fix sekalian karena keduanya menyentuh file yang sama
