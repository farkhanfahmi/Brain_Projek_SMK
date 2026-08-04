# T102 — Schema+API+UI: Dashboard Pembina Ekstra Lengkap (Daftar Peserta, Kelompok/Sesi Paralel, Jadwal Hari, Auto-Generate Presensi)

## Depends on
T096 (06-Features/tasks/T096-absensi-ekstrakurikuler-pembina.md) — SUDAH DIEKSEKUSI (model `EkstraSesi`/`EkstraAbsen`, guard `EkstraPembinaGuard`, halaman dasar `apps/web/src/app/(guru)/guru/ekstrakurikuler/`). **T102 ini AMANDEMEN BESAR terhadap desain T096** — mengubah `EkstraSesi` dari "dibuat manual per tanggal, 1 sesi = seluruh peserta ekstra" menjadi "auto-generate dari jadwal hari, opsional per-kelompok". Baca T096 dulu SEPENUHNYA untuk paham baseline sebelum baca task ini.

## Objective
3 submenu baru di dashboard pembina ekstra (guru yang jadi pembina ATAU akun `pembina_ekstra` eksternal): **Daftar Peserta** (lihat semua siswa terdaftar, filter+search), **Presensi** (buat/lihat/edit presensi per pertemuan, auto-generate dari jadwal, opsional per-kelompok), **Setting Ekstrakurikuler** (atur hari+jam ekstra, kelola kelompok/sesi paralel + plotting siswa ke kelompok).

## Context
- **App:** `apps/api` + `apps/web`
- Diskusi lengkap 2026-07-30, LANJUTAN dari diskusi T096 (sesi sebelumnya). Beberapa keputusan T096 DIPERTAHANKAN (status Hadir/Izin/Sakit/Alfa, upload bukti untuk Izin/Sakit, pola arsitektur meniru `TeachingSession`/`ClassAttendanceMark`), sebagian **DIUBAH** (lihat di bawah).
- **Kode existing yang jadi acuan pola**: `apps/api/src/teaching-sessions/teaching-sessions.service.ts` method `generateForDate()` (auto-generate idempotent via `upsert`, cek `SchoolHoliday`, skip hari Minggu) — **TIRU POLA INI PERSIS** untuk auto-generate `EkstraSesi`, jangan reinvent. `apps/api/src/teaching-sessions/generate-sessions.processor.ts` — pola BullMQ cron untuk trigger harian, tiru untuk ekstra.

## Perubahan dari Desain T096 (BACA DULU sebelum eksekusi)

| Aspek | T096 (lama) | T102 (baru) |
|---|---|---|
| Pembuatan `EkstraSesi` | Manual, pembina klik "Buat Sesi" kapan saja, isi tanggal bebas | **Auto-generate** dari jadwal hari ekstra (field baru), pembina TIDAK isi tanggal manual — sistem yang buat begitu hari itu tiba, pembina tinggal buka & isi presensi |
| Peserta per sesi | Semua peserta `EkstraPendaftaran` ekstra itu, snapshot saat sesi dibuat | Kalau ekstra TIDAK punya kelompok: sama seperti lama (semua peserta). Kalau PUNYA kelompok: peserta = siswa yang sudah di-assign ke kelompok itu SAJA |
| Kelompok/paralel | Tidak ada konsep ini sama sekali | BARU — `EkstraKelompok` (opsional per ekstra), beda JAM di HARI YANG SAMA (tidak pernah bentrok jam), siswa di-assign manual semi-permanen ke 1 kelompok |
| Jadwal | Tidak ada field jadwal sama sekali di `Ekstrakurikuler` | BARU — field `hari` (dan `jamMulai`/`jamSelesai` kalau TIDAK ada kelompok; kalau ADA kelompok, jam ikut per-kelompok bukan di level ekstra) |

## Keputusan Final (dikonfirmasi user 2026-07-30, sesi lanjutan)

1. **Kelompok OPSIONAL per ekstra** — ekstra tanpa kelompok tetap seperti desain T096 lama (1 sesi = semua peserta). Ekstra DENGAN kelompok, sesi jadi per-kelompok.
2. **Kelompok ditentukan pembina di awal** (master data semi-permanen, bukan dadakan tiap pertemuan) — field: nama kelompok + jam (hari SAMA dengan ekstra, cuma jamnya beda dari kelompok lain, ekstra selalu 1x seminggu jadi cukup "hari" tanpa tanggal spesifik).
3. **Kelompok TIDAK PERNAH bentrok jam** (dikonfirmasi eksplisit — Sesi A dan B beda jam, tidak ada 2 kelompok jalan bersamaan) — 1 pembina yang sama menangani semua kelompok secara BERURUTAN di hari yang sama.
4. **`EkstraSesi` di-AUTO-GENERATE** tiap hari (job cron, pola SAMA seperti `TeachingSessionsService.generateForDate()`) — cek tiap `Ekstrakurikuler` yang punya jadwal (`hari` cocok hari ini), buat `EkstraSesi` untuk hari itu (kalau ada kelompok: 1 `EkstraSesi` PER kelompok; kalau tidak: 1 `EkstraSesi` untuk seluruh ekstra) — **idempotent** (upsert, tidak boleh dobel kalau job jalan 2x).
5. **Field jam di form presensi**: tetap TAMPIL (bukan disembunyikan) tapi **otomatis terisi** dari jadwal (read-only atau minimal default terisi, bukan input kosong yang pembina isi manual) — dikonfirmasi eksplisit "tetep tampil tapi terisi otomatis".
6. **Alur halaman Presensi**:
   - Pembina buka menu Presensi → sistem cek: ada `EkstraSesi` untuk HARI INI yang belum dibuka/diisi? Kalau BELUM ada sesi hari ini sama sekali (edge case: auto-generate belum jalan, atau hari ini bukan hari jadwal tapi pembina ingin buat manual sebagai pengecualian) → tombol "Buat Presensi" muncul (isi tanggal + jam otomatis-terisi + materi opsional) — **cek dengan user apakah tombol manual ini SELALU ada sebagai fallback, atau HANYA muncul kalau auto-generate gagal/tidak ada jadwal cocok hari itu**.
   - Kalau ADA kelompok untuk ekstra ini → dropdown pilih kelompok saat membuat/membuka presensi.
   - Daftar presensi yang sudah dibuat (riwayat) ditampilkan sebagai list — klik salah satu masuk ke halaman detail presensi (tabel semua siswa di sesi itu).
   - Halaman detail presensi: tombol **"Hadir Semua"** (set semua baris jadi Hadir di STATE lokal, belum ke server), pembina bisa ubah manual baris tertentu jadi Alfa/Izin/Sakit (state lokal), lalu **1 tombol "Save"** mengirim SEMUA perubahan sekaligus ke server (dikonfirmasi: bukan auto-save per aksi).
   - Presensi HARI SEBELUMNYA bisa dibuka lagi dari daftar riwayat dan di-edit ulang (ubah status, klik Save lagi) — tidak ada penguncian setelah tersimpan pertama kali.
7. **Menu Setting Ekstrakurikuler**:
   - Input **hari** ekstra (dan jam, kalau TIDAK ada kelompok) — 1x seminggu, field `hari` (enum/int 1-7 pola sama `Schedule.hari`), bukan tanggal spesifik.
   - Kelola kelompok: buat kelompok baru (nama + jam), lihat daftar kelompok yang sudah ada.
   - **Plotting siswa ke kelompok** — halaman per-kelompok dengan **split view 1 halaman** (dikonfirmasi user, REKOMENDASI final): 2 daftar sekaligus terlihat — "Belum Berkelompok" (siswa terdaftar ekstra ini yang belum masuk kelompok manapun, tombol **+** untuk assign ke kelompok yang sedang dibuka) dan "Anggota Kelompok Ini" (siswa yang sudah di-assign, tombol **X** untuk keluarkan dari kelompok) — kedua daftar update REAL-TIME setelah aksi (tanpa reload), TIAP KLIK langsung API call (bukan batch-save, beda dari alur Presensi di atas — assign/unassign kelompok adalah aksi ringan low-risk, cocok direct-save).
8. **Menu "Daftar Peserta"** — tabel semua siswa terdaftar (`EkstraPendaftaran`) di ekstra ini, filter Jurusan (dropdown) → Kelas (dropdown, TER-FILTER oleh Jurusan terpilih, pola SAMA PERSIS seperti urutan filter Search→Jurusan→Kelas (feedback_filter_search_jurusan_kelas_order.md) yang sudah jadi konvensi di codebase — lihat `direktori-siswa-view.tsx` T093 sebagai referensi implementasi), dan search nama.

## Spec Detail — Skema Baru/Diubah

```prisma
model Ekstrakurikuler {
  id        Int    @id @default(autoincrement())
  nama      String @unique
  urutan    Int    @default(0)
  pembinaId Int?   @unique @map("pembina_id")

  // BARU — jadwal ekstra, 1x seminggu. jamMulai/jamSelesai HANYA relevan kalau TIDAK
  // pakai kelompok (kalau pakai kelompok, jam ikut EkstraKelompok masing-masing).
  hari       Int?    @map("hari") // 1=Minggu..7=Sabtu, pola sama Schedule.hari. Null = belum diset (ekstra lama sebelum T102)
  jamMulai   String? @map("jam_mulai") // HH:mm, null kalau pakai kelompok
  jamSelesai String? @map("jam_selesai")

  pembina     User?               @relation("EkstrakurikulerPembina", fields: [pembinaId], references: [id])
  pendaftaran EkstraPendaftaran[]
  sesi        EkstraSesi[]
  kelompok    EkstraKelompok[]    // BARU, kosong = ekstra ini tidak pakai kelompok

  @@map("ekstrakurikuler")
}

// BARU — kelompok/sesi paralel opsional, beda jam di hari yang SAMA dengan Ekstrakurikuler.hari.
model EkstraKelompok {
  id                Int    @id @default(autoincrement())
  ekstrakurikulerId Int    @map("ekstrakurikuler_id")
  nama              String // "Kelompok A", bebas diisi pembina
  jamMulai          String @map("jam_mulai")
  jamSelesai        String @map("jam_selesai")

  ekstrakurikuler Ekstrakurikuler       @relation(fields: [ekstrakurikulerId], references: [id])
  anggota         EkstraKelompokAnggota[]
  sesi            EkstraSesi[]

  @@index([ekstrakurikulerId])
  @@map("ekstra_kelompok")
}

// BARU — assignment siswa ke kelompok, semi-permanen (TIDAK snapshot per sesi, ini
// master data "siswa X ada di kelompok mana", dipakai ULANG tiap kali sesi baru
// digenerate untuk kelompok itu).
model EkstraKelompokAnggota {
  id         Int @id @default(autoincrement())
  kelompokId Int @map("kelompok_id")
  studentId  Int @map("student_id")

  kelompok EkstraKelompok @relation(fields: [kelompokId], references: [id])
  student  Student        @relation(fields: [studentId], references: [id])

  @@unique([studentId]) // 1 siswa cuma boleh di 1 kelompok per ekstra pada satu waktu -- cek apakah perlu @@unique([kelompokId, studentId]) TAPI juga constraint "1 siswa 1 kelompok per ekstra" perlu enforce di level SERVICE (unique studentId GLOBAL salah kalau siswa ikut >1 ekstra dgn kelompok masing2 -- PERBAIKI jadi unique gabungan yang benar, lihat catatan di bawah)
  @@map("ekstra_kelompok_anggota")
}

model EkstraSesi {
  id                Int      @id @default(autoincrement())
  ekstrakurikulerId Int      @map("ekstrakurikuler_id")
  kelompokId        Int?     @map("kelompok_id") // BARU — null kalau ekstra tidak pakai kelompok
  tanggal           DateTime @db.Date
  jamMulai          String?  @map("jam_mulai") // BARU — snapshot jam SAAT sesi dibuat (dari ekstra atau kelompok), field presensi "tetap tampil, terisi otomatis"
  jamSelesai        String?  @map("jam_selesai")
  catatan           String?  @db.Text
  createdById       Int      @map("created_by")
  createdAt         DateTime @default(now()) @map("created_at")

  ekstrakurikuler Ekstrakurikuler @relation(fields: [ekstrakurikulerId], references: [id])
  kelompok        EkstraKelompok? @relation(fields: [kelompokId], references: [id])
  createdBy       User            @relation("EkstraSesiCreatedBy", fields: [createdById], references: [id])
  absen           EkstraAbsen[]

  @@unique([ekstrakurikulerId, kelompokId, tanggal]) // idempotency key untuk auto-generate upsert
  @@index([ekstrakurikulerId, tanggal])
  @@map("ekstra_sesi")
}

// EkstraAbsen TIDAK BERUBAH dari T096.
```

**⚠️ CATATAN PENTING soal `EkstraKelompokAnggota.@@unique`**: draft di atas `@@unique([studentId])` SALAH kalau constraint yang dimaksud adalah "1 siswa 1 kelompok DALAM 1 ekstra" (karena siswa cuma boleh daftar 1 ekstra total lewat `EkstraPendaftaran.studentId @unique`, jadi sebenarnya secara TRANSITIF `unique(studentId)` di sini SAH — siswa yang sama tidak mungkin perlu masuk kelompok di ekstra LAIN karena dia cuma terdaftar di 1 ekstra). **Verifikasi ulang logika ini saat implementasi** — kalau ternyata `EkstraPendaftaran` suatu saat berubah jadi multi-ekstra per siswa, constraint ini perlu direvisi jadi composite key.

## Business Rules

1. Auto-generate `EkstraSesi` — job cron harian (BullMQ, tiru pola `generate-sessions.processor.ts`), method baru `EkstraAbsensiService.generateForDate(tanggal)`: untuk tiap `Ekstrakurikuler` dengan `hari` cocok hari ini — kalau punya `kelompok`, generate 1 `EkstraSesi` PER kelompok (peserta = `EkstraKelompokAnggota` kelompok itu); kalau tidak, generate 1 `EkstraSesi` untuk ekstra (peserta = semua `EkstraPendaftaran` aktif). Skip kalau `SchoolHoliday` hari itu (tiru cek existing). `upsert` by `(ekstrakurikulerId, kelompokId, tanggal)` — idempotent.
2. Endpoint manual trigger "Buat Presensi" TETAP ada sebagai fallback (untuk kasus auto-generate belum sempat jalan, atau ekstra baru diset jadwalnya hari ini juga) — **klarifikasi ke user** kapan tombol ini SEHARUSNYA muncul (selalu vs kondisional) sebelum implementasi UI final.
3. Snapshot `EkstraAbsen` (status null) dibuat SEKALIGUS saat `EkstraSesi` di-generate (transaksi sama), BUKAN belakangan — konsisten dengan T096.
4. Assign/unassign siswa ke `EkstraKelompok` (submenu Setting) adalah aksi LANGSUNG-SIMPAN (setiap klik +/X langsung API call), BEDA dari halaman Presensi (state lokal dulu, 1 tombol Save di akhir) — dua pola berbeda untuk dua konteks berbeda, JANGAN disamakan.
5. Guard `EkstraPembinaGuard` (T096, sudah ada) tetap dipakai untuk semua endpoint baru — pembina hanya bisa kelola ekstra miliknya sendiri.
6. Ekstra yang BELUM di-set jadwalnya (`hari` null, kondisi transisi dari data lama T096) — TIDAK ikut ter-generate otomatis, dan menu Presensi untuk ekstra itu HANYA punya opsi manual (fallback poin 2) sampai jadwalnya diisi lewat Setting.

## Edge Cases

- Siswa dipindah dari Kelompok A ke Kelompok B — histori `EkstraAbsen` di sesi-sesi LAMA (saat masih di Kelompok A) TIDAK berubah (sesuai prinsip yang sudah disepakati sebelumnya untuk kasus serupa: histori tidak retroaktif). Sesi BARU yang di-generate setelah pindah akan pakai kelompok baru.
- Ekstra baru dibuat kelompoknya SETELAH beberapa sesi tanpa-kelompok sudah berjalan — sesi LAMA (tanpa kelompok, `kelompokId: null`) tetap ada apa adanya, sesi BARU setelah kelompok dibuat mulai pakai `kelompokId`.
- Siswa terdaftar ekstra tapi BELUM di-assign ke kelompok manapun (ekstra ini pakai kelompok) — TIDAK ikut ter-snapshot ke `EkstraAbsen` manapun sampai di-assign (dia baru "aktif" absen setelah masuk kelompok) — pastikan halaman "Daftar Peserta" tetap menampilkan siswa ini (peserta ekstra secara keseluruhan), TAPI beri indikator visual "belum ada kelompok" supaya pembina sadar perlu plotting.
- Hari yang sama dipakai 2 kelompok berbeda TAPI jam mereka ternyata tumpang tindih (human error input pembina, misal Kelompok A 15:00-16:00, Kelompok B 15:30-16:30) — dikonfirmasi user "tidak akan bentrok jam", TAPI **pertimbangkan validasi sederhana** (warning, bukan hard block) saat pembina input jam kelompok baru yang overlap dengan kelompok lain di ekstra yang sama, supaya kesalahan input ketahuan lebih awal.

## Files

- **Modifikasi:** `apps/api/prisma/schema.prisma` (model/field baru di atas, migration baru).
- **Modifikasi:** `apps/api/src/ekstra-absensi/ekstra-absensi.service.ts` (method `generateForDate`, restrukturisasi `createSesi` manual jadi fallback, method assign/unassign kelompok, method list peserta dengan filter jurusan/kelas/search), `ekstra-absensi.controller.ts` (endpoint baru), DTO baru (`create-kelompok.dto.ts`, `assign-kelompok.dto.ts`, `set-jadwal-ekstra.dto.ts`, dll).
- **Buat:** job cron baru (pola `generate-sessions.processor.ts`) untuk auto-generate `EkstraSesi` harian.
- **Modifikasi:** `apps/web/src/app/(guru)/guru/ekstrakurikuler/` — restrukturisasi jadi 3 submenu (Daftar Peserta, Presensi, Setting) — kemungkinan perlu sub-routing baru (`/guru/ekstrakurikuler/peserta`, `/guru/ekstrakurikuler/presensi`, `/guru/ekstrakurikuler/setting`) dengan tab/nav lokal di dalam halaman ekstrakurikuler itu sendiri (BUKAN menambah item baru ke sidebar guru utama — ini sub-navigasi DALAM 1 menu "Ekstrakurikuler").
- **Buat (untuk akun `pembina_ekstra` eksternal, T096)**: cek apakah dashboard `(pembina-ekstra)/` route group sudah dibuat T096 — kalau sudah, terapkan 3 submenu yang sama di sana; kalau belum, ini bagian dari T096 yang mungkin belum lengkap dieksekusi, verifikasi dulu.
- **Jangan sentuh:** `EkstraPendaftaran`, pendaftaran publik (`ekstra-publik.service.ts`) — di luar scope, tidak berubah oleh T102.

## Acceptance Criteria

- [x] Admin/pembina bisa set jadwal (hari, jam) untuk ekstrakurikuler lewat menu Setting.
- [x] Pembina bisa buat kelompok (nama+jam) dan lihat daftar kelompok yang ada.
- [x] Halaman plotting kelompok menampilkan split view (belum berkelompok + anggota kelompok), assign/unassign langsung tersimpan per klik.
- [x] Job auto-generate berjalan (bisa di-trigger manual untuk testing), membuat `EkstraSesi` sesuai jadwal — 1 per ekstra (tanpa kelompok) atau 1 per kelompok (dengan kelompok) — idempotent (jalan 2x tidak bikin dobel).
- [x] Halaman Presensi menampilkan field tanggal+jam yang otomatis terisi, materi opsional, dropdown kelompok muncul kalau ekstra ini pakai kelompok.
- [x] Tombol "Hadir Semua" mengisi state lokal semua baris jadi Hadir, bisa diedit manual sebelum 1x klik Save mengirim semua sekaligus.
- [x] Presensi hari sebelumnya bisa dibuka & diedit ulang dari daftar riwayat.
- [x] Halaman Daftar Peserta punya filter Jurusan→Kelas (kelas ter-filter oleh jurusan) + search nama.
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] Ini AMANDEMEN besar T096 yang sudah live — pastikan migrasi Prisma tidak merusak data `EkstraSesi`/`EkstraAbsen` existing (kalau sudah ada data uji coba, field baru harus nullable/punya default supaya baris lama tetap valid).
- [x] Klarifikasi ke user SEBELUM implementasi: kapan tombol "Buat Presensi manual" muncul (selalu vs cuma fallback) — jangan asumsikan sendiri.
- [x] Verifikasi ulang constraint `EkstraKelompokAnggota` (catatan di atas) sebelum menulis migration — pastikan asumsi "1 siswa 1 ekstra total" masih berlaku saat task ini dieksekusi (cek `EkstraPendaftaran.studentId @unique` belum berubah).
- [x] Tiru pola `TeachingSessionsService.generateForDate()` dan `generate-sessions.processor.ts` SEPERSIS mungkin untuk auto-generate — jangan desain mekanisme cron baru yang berbeda tanpa alasan kuat.
- [x] Filter Jurusan→Kelas di Daftar Peserta WAJIB ikuti urutan Search→Jurusan→Kelas (feedback_filter_search_jurusan_kelas_order.md) yang sudah jadi konvensi (lihat memory project, dan implementasi T093 di `direktori-siswa-view.tsx`).

## Implementasi & Verifikasi (2026-07-31)

**Status: SUDAH DIEKSEKUSI**

Klarifikasi user sebelum implementasi:
1. Tombol "Buat Presensi" manual — **SELALU tampil** (bukan cuma fallback), pembina boleh buat sesi tambahan kapan pun meski auto-generate sudah jalan hari itu.
2. Gap ditemukan saat riset: route group `(pembina-ekstra)/` untuk akun eksternal (bukan guru) **belum pernah dibuat** meski T096 tercatat "SUDAH DIEKSEKUSI" di vault — user konfirmasi tutup gap ini sebagai bagian T102, bukan task terpisah.
3. Konfirmasi eksplisit sebelum jalankan `prisma migrate dev` (via workaround `migrate diff` + `db execute`, non-interaktif) terhadap DB dev live.

Yang dikerjakan:
- Schema: `Ekstrakurikuler.hari/jamMulai/jamSelesai`, model baru `EkstraKelompok` + `EkstraKelompokAnggota`, `EkstraSesi.kelompokId/jamMulai/jamSelesai` + unique `(ekstrakurikulerId, kelompokId, tanggal)`. Migration diterapkan & diverifikasi live di MySQL dev.
- Backend: `EkstraAbsensiService.generateForDate()` (auto-generate harian, tiru `TeachingSessionsService.generateForDate()` persis) + BullMQ cron processor (`10 0 * * *`), CRUD kelompok, assign/unassign anggota (direct-save), `listPeserta` dengan filter Jurusan→Kelas→search, `setJadwal`. Endpoint `generate-today` untuk trigger manual (role `admin_jurnal`/`super_admin`).
- Frontend: folder baru `apps/web/src/features/ekstrakurikuler/` (komponen shared lintas route-group — pola baru, sebelumnya cuma `components/shell/` yang begini), dipakai baik oleh `(guru)/guru/ekstrakurikuler/` (restrukturisasi jadi 3 submenu) maupun route group baru `(pembina-ekstra)/` (menutup gap T096).
- Gap tambahan yang ditemukan & ditutup saat implementasi: `GET /kelas` dan `GET /jurusan` sebelumnya tidak bisa diakses role `guru`/`pembina_ekstra` (diperlukan untuk filter Daftar Peserta); `UserRole` type di 2 tempat frontend (`core-types.ts` dan `current-user.ts`) belum punya `pembina_ekstra` sama sekali.

Verifikasi:
- `pnpm --filter @absensi/api build` + `pnpm --filter @absensi/web build` — 6/6 sukses, semua route baru (`/ekstrakurikuler`, `/ekstrakurikuler/peserta`, `/ekstrakurikuler/setting`, `/ekstrakurikuler/sesi/[sesiId]`, plus `/guru/ekstrakurikuler/*`) muncul di manifest.
- `pnpm --filter @absensi/api test` — 183/183 test lulus, tanpa regresi.
- Live verification via Playwright browser: login sebagai akun `pembina_ekstra` seed → redirect otomatis ke `/ekstrakurikuler` (traffic-cop `(admin)/layout.tsx` berfungsi) → set jadwal hari, buat kelompok, plotting siswa (split-view assign/unassign), buat presensi, tandai Hadir Semua + Save batch — semua alur berhasil end-to-end dengan data seed manual (DB dev sebelumnya kosong total, 0 rows semua tabel).

**2 bug ditemukan user saat uji manual, sudah diperbaiki:**
1. Field tanggal di dialog "Buat Presensi" masih bisa diedit bebas (harusnya read-only/otomatis-terisi sama seperti field jam, sesuai poin 5 keputusan final di atas) — fix: `presensi-view.tsx` field tanggal jadi `disabled`, terisi tanggal hari ini.
2. Error `Cannot read properties of undefined (reading 'nisn')` saat klik Save di halaman detail presensi — root cause: `EkstraAbsensiService.markAbsen()` (PATCH `/ekstra-absensi/absen/:id`) tidak `include: { student: ... }` di response, sehingga hasil PATCH yang dipakai untuk replace state lokal kehilangan relasi `student` dan crash saat render ulang. Fix: tambah `include` yang sama seperti `findSesiDetail()`.

Kedua fix di-build ulang, server di-restart, dan diverifikasi ulang live via Playwright (Save berhasil tanpa error, NISN/nama tetap tampil benar) + jest re-run (183/183 tetap lulus).
