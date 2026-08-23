# T158 — Schema+API+Web: Model "Opsi Jam Pelajaran" (Multi-Varian, Admin Aktifkan Satu)

## Depends on
Tidak ada dependency teknis wajib ke T156/T157 (bisa paralel). **T159 dan T160 (mode blok per kelas, import CSV jadwal) DEPENDS ON task ini** — kolom `jamKe`/resolusi jam pada `Schedule` akan berubah sumbernya, jadi T158 harus selesai dulu.

## Objective
Admin bisa membuat **beberapa OPSI jam pelajaran** (masing-masing berisi definisi lengkap "Jam Ke berapa → jam berapa sampai jam berapa" untuk SEMUA hari sekolah Senin-Jumat sekaligus, termasuk jam istirahat) — dan MENGAKTIFKAN salah satu opsi sebagai yang berlaku SAAT INI. Konsep ini BARU SEPENUHNYA, belum ada di sistem sama sekali (dikonfirmasi riset: field `isActive` yang ada sekarang hanya untuk `AcademicYear`/`Semester`, BUKAN untuk varian jam pelajaran).

## Context — Referensi dari File Excel Sekolah (dariDev/Sistem Jurnal Baru 2026-2027.xlsx)

User memberikan contoh nyata struktur "Alokasi Waktu" sekolah, sheet `Alokasi Waktu`:
- **Section "A. SENIN - KAMIS"**: 13 jam pelajaran (Jam Ke 1-13), tiap jam 40 menit (kecuali 3 jam terakhir 35 menit), PLUS 2 baris "Istirahat Ke-1"/"Istirahat Ke-2" yang BUKAN jam pelajaran tapi tetap bagian dari urutan waktu (jam 09:40-10:10 dan 12:50-13:40).
- **Section "B. JUMAT"**: 7 jam pelajaran, durasi BEDA (30 menit/jam, bukan 40), PLUS 1 istirahat.
- **Tidak ada section Sabtu** (konsisten aturan proyek: Sabtu bukan hari wajib berjadwal formal).

**Keputusan final dari diskusi 2026-08-11**:
1. **1 "Opsi Jam Pelajaran" = 1 paket LENGKAP mencakup SEMUA hari sekaligus** (Senin-Kamis dengan durasinya sendiri, Jumat dengan durasinya sendiri, dalam 1 opsi yang sama) — BUKAN opsi terpisah per-hari. Mengaktifkan 1 opsi = otomatis berlaku untuk semua hari sesuai definisi masing-masing di dalam opsi itu.
2. Admin bisa punya BANYAK opsi tersimpan (misal "Jam Reguler 2026/2027", "Jam Ramadhan", "Jam Ujian").

**REVISI 2026-08-13 (menggantikan sebagian keputusan poin 2 di atas)** — user eksplisit butuh opsi jam pelajaran BISA discope ke **tingkat tertentu** (X/XI/XII), BUKAN selalu seluruh sekolah:
3. Tiap Opsi Jam Pelajaran punya **cakupan tingkat** — bisa "Semua Tingkat" ATAU kombinasi tingkat spesifik (misal hanya XII). Contoh nyata kasus pakai: "Jam Reguler" aktif untuk X+XI, BERSAMAAN dengan "Jam UAS Kelas XII" aktif khusus tingkat XII — keduanya aktif di waktu yang SAMA, tidak saling menggantikan.
4. **Constraint "1 aktif" jadi PER TINGKAT, bukan global** — untuk tingkat X, hanya boleh 1 opsi aktif yang cakupannya mencakup tingkat X (baik opsi "Semua Tingkat" maupun opsi khusus X) di satu waktu; sama untuk XI dan XII independen. Ini MENGGANTIKAN model constraint global sederhana yang diasumsikan sebelumnya — lihat detail model di poin 1 Spec Detail (sudah direvisi).

## Spec Detail

### 1. Schema (Prisma) — 3 model baru (REVISI 2026-08-13: scope tingkat + constraint aktif per-tingkat)

```prisma
model JamPelajaranOption {
  id          Int      @id @default(autoincrement())
  nama        String   // misal "Jam Reguler 2026/2027", "Jam Ramadhan", "Jam UAS Kelas XII"
  semuaTingkat Boolean @default(true) @map("semua_tingkat") // true = berlaku semua tingkat, false = lihat relasi tingkatScopes
  createdById Int      @map("created_by")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  slots         JamPelajaranSlot[]
  tingkatScopes JamPelajaranOptionTingkat[] // kosong kalau semuaTingkat=true
  aktivasi      JamPelajaranAktivasi[]
  createdBy     User @relation(fields: [createdById], references: [id])

  @@map("jam_pelajaran_options")
}

// Cakupan tingkat spesifik kalau semuaTingkat=false (baris per tingkat yang dicakup opsi ini)
model JamPelajaranOptionTingkat {
  id       Int     @id @default(autoincrement())
  optionId Int     @map("option_id")
  tingkat  Tingkat // reuse enum Tingkat existing (X/XI/XII)

  option JamPelajaranOption @relation(fields: [optionId], references: [id], onDelete: Cascade)

  @@unique([optionId, tingkat])
  @@map("jam_pelajaran_option_tingkat")
}

// Status AKTIF per tingkat — INI yang menggantikan JamPelajaranOption.isActive lama (constraint global salah, harus per-tingkat)
model JamPelajaranAktivasi {
  id       Int     @id @default(autoincrement())
  tingkat  Tingkat @unique // 1 baris per tingkat (X, XI, XII) — tingkat itu SELALU py tepat 1 opsi aktif atau tidak ada sama sekali
  optionId Int     @map("option_id")

  option JamPelajaranOption @relation(fields: [optionId], references: [id])

  @@map("jam_pelajaran_aktivasi")
}

model JamPelajaranSlot {
  id           Int      @id @default(autoincrement())
  optionId     Int      @map("option_id")
  hari         Int      // 1=Minggu..7=Sabtu (DAYOFWEEK MySQL, KONSISTEN dgn Schedule.hari existing)
  jamKe        Int?     @map("jam_ke")      // NULL kalau baris ini istirahat, bukan jam pelajaran
  jamMulai     String   @map("jam_mulai")   // HH:mm
  jamSelesai   String   @map("jam_selesai") // HH:mm
  keterangan   String?  // misal "Istirahat Ke-1", null untuk jam pelajaran biasa
  urutan       Int      // urutan tampil dalam 1 hari (1,2,3... termasuk baris istirahat), BUKAN sama dengan jamKe

  option JamPelajaranOption @relation(fields: [optionId], references: [id], onDelete: Cascade)

  @@unique([optionId, hari, urutan])
  @@index([optionId, hari])
  @@map("jam_pelajaran_slots")
}
```
- **`jamKe` NULLABLE** — baris istirahat TIDAK punya nomor jam pelajaran (konsisten sheet Excel: baris istirahat kosong di kolom "Jam Ke"), tapi TETAP perlu baris tersendiri karena dia mengisi RENTANG WAKTU (09:40-10:10) yang harus "dilewati" saat resolusi jam-ke-ke-waktu (lihat poin 3).
- **`urutan`** terpisah dari `jamKe` — supaya baris istirahat bisa disisipkan di posisi yang benar dalam urutan waktu tanpa perlu jamKe pura-pura (misal istirahat ke-1 muncul SETELAH jam ke-4, SEBELUM jam ke-5 — `urutan` merepresentasikan posisi kronologis, `jamKe` merepresentasikan nomor pelajaran resminya kalau ada).
- Migration baru.

### 2. Backend — service CRUD + resolusi

- Modul baru `apps/api/src/jam-pelajaran/` — service `JamPelajaranService`:
  - `findAll()` — list semua opsi (ringkas, tanpa slot detail), termasuk cakupan tingkat (`semuaTingkat` + `tingkatScopes`) dan status aktif per tingkat (join ke `JamPelajaranAktivasi`).
  - `findOne(id)` — detail 1 opsi LENGKAP dengan semua slot, grouped by hari untuk kemudahan render UI.
  - `create(dto, actorId)` — buat opsi BARU beserta SEMUA slot-nya sekaligus + cakupan tingkat (`semuaTingkat: true` ATAU array `tingkat[]` kalau false) — transaksi `$transaction` (opsi + createMany slot + createMany tingkatScopes kalau relevan).
  - `update(id, dto, actorId)` — update opsi (nama, cakupan tingkat) + REPLACE SELURUH slot (hapus semua slot lama, insert slot baru — lebih sederhana dan aman daripada diff-patch parsial). **KALAU opsi ini SEDANG AKTIF di salah satu tingkat DAN cakupan tingkatnya diubah sehingga tidak lagi mencakup tingkat itu** — TOLAK dengan pesan jelas (harus non-aktifkan dulu dari tingkat itu sebelum ubah cakupan), JANGAN diam-diam menghapus aktivasi.
  - `activate(id, tingkat, actorId)` — **REVISI: sekarang terima parameter `tingkat` eksplisit** (bukan aktivasi global) — transaksi: upsert `JamPelajaranAktivasi` untuk `tingkat` itu ke `optionId` yang dipilih (menggantikan aktivasi tingkat itu sebelumnya kalau ada, TIDAK menyentuh aktivasi tingkat LAIN). VALIDASI: opsi yang mau diaktifkan HARUS mencakup `tingkat` itu (`semuaTingkat: true` ATAU ada baris `JamPelajaranOptionTingkat` untuk tingkat itu) — TOLAK kalau tidak mencakup, pesan jelas.
  - `deactivate(tingkat, actorId)` — hapus baris `JamPelajaranAktivasi` untuk tingkat itu (tingkat itu jadi tidak punya opsi aktif sama sekali — state valid, lihat Edge Cases fallback aman).
  - `delete(id)` — TOLAK kalau opsi ini SEDANG AKTIF di TINGKAT MANA PUN (`JamPelajaranAktivasi` refer ke opsi ini, cek SEMUA baris bukan cuma 1) — harus non-aktifkan dari SEMUA tingkat dulu sebelum bisa dihapus.
  - `getActiveForTingkat(tingkat)` — helper terpusat return opsi yang SEDANG AKTIF untuk 1 tingkat tertentu beserta slot-nya (query `JamPelajaranAktivasi` by tingkat → join opsi+slot), DIPAKAI oleh method resolusi jam-ke-ke-waktu (poin 3) — SATU sumber kebenaran per tingkat, jangan query `JamPelajaranAktivasi` di banyak tempat berbeda dengan cara berbeda.
- `@LogActivity` di endpoint create/update/activate/delete (audit trail, aturan permanen proyek — WAJIB, jangan lupa).
- Guard: `@Roles(UserRole.super_admin)` untuk mutasi, GET boleh diakses role lebih luas (konsisten pola config lain).

### 3. KEPUTUSAN ARSITEKTUR PENTING — bagaimana `Schedule.jamMulai`/`jamSelesai` berhubungan dengan `JamPelajaranSlot`

Ini keputusan desain KRITIKAL yang HARUS diputuskan SEBELUM implementasi — PILIH SALAH SATU (REKOMENDASI eksplisit diberikan, tapi implementer WAJIB paham trade-off):

- **Opsi A (REKOMENDASI)** — `Schedule` (untuk `type: jam_mengajar`) TIDAK LAGI simpan `jamMulai`/`jamSelesai` sebagai STRING WAKTU LANGSUNG, melainkan simpan **`jamKeAwal`+`jamKeAkhir`** (angka jam-ke, BUKAN jam:menit) — dan `jamMulai`/`jamSelesai` (waktu sungguhan) DIHITUNG/DI-RESOLVE saat dibutuhkan (saat generate `TeachingSession`, saat tampil di UI) dengan LOOKUP ke `JamPelajaranSlot` yang SEDANG AKTIF **UNTUK TINGKAT KELAS JADWAL ITU** (`getActiveForTingkat(schedule.kelas.tingkat)`, REVISI 2026-08-13 — bukan lagi 1 opsi aktif global, resolusi WAJIB tahu tingkat kelasnya dulu), dicocokkan `hari`+`jamKe`. **Konsekuensi**: kalau admin GANTI opsi jam pelajaran yang aktif UNTUK TINGKAT TERTENTU, hanya jadwal kelas-kelas di tingkat itu yang ikut berubah jam sungguhannya (tingkat lain tidak terpengaruh) — ini MASUK AKAL secara bisnis (misal "Jam UAS Kelas XII" aktif cuma untuk tingkat XII, tingkat X/XI tetap pakai jam reguler berjalan seperti biasa).
- **Opsi B** — `Schedule` TETAP simpan `jamMulai`/`jamSelesai` sebagai STRING WAKTU LANGSUNG (SEPERTI SEKARANG, TIDAK BERUBAH), dan `JamPelajaranOption`/`Slot` HANYA dipakai sebagai ALAT BANTU INPUT (dropdown "pilih Jam Ke X" saat admin bikin/edit Schedule manual ATAU saat import CSV, yang di-resolve SEKALI SAJA saat itu jadi `jamMulai`/`jamSelesai` konkret, lalu DISIMPAN sebagai snapshot). **Konsekuensi**: ganti opsi jam pelajaran aktif TIDAK mempengaruhi jadwal yang SUDAH ADA (perlu re-generate/re-import manual kalau mau ikut opsi baru) — LEBIH SEDERHANA implementasinya, TAPI kurang "hidup"/otomatis.

**REKOMENDASI KUAT: Opsi A** — ini yang membuat fitur "admin tinggal aktifkan opsi jam pelajaran" benar-benar BERGUNA (kalau Opsi B yang dipilih, "aktifkan opsi" jadi kurang berarti karena jadwal existing tidak ikut berubah). TAPI **INI PERUBAHAN BESAR ke model `Schedule`** (field `jamMulai`/`jamSelesai` string TETAP DIPERTAHANKAN untuk `type: jam_sekolah` yang TIDAK terkait konsep ini sama sekali — HANYA `type: jam_mengajar` yang berubah cara resolve jamnya) — KLARIFIKASI KE USER kalau ada keraguan scope perubahan ini sebelum implementasi, JANGAN putuskan sepihak tanpa baca ulang SEMUA pemakai `Schedule.jamMulai`/`jamSelesai` existing (dashboard guru, rekap, dst — banyak sekali consumer field ini, PERLU AUDIT MENYELURUH dulu sebelum ubah makna field).

### 4. Frontend — halaman admin baru "Jam Pelajaran"

- Path BARU `apps/web/src/app/(admin)/jam-pelajaran/` — daftar SEMUA opsi jam pelajaran + kolom cakupan tingkat (badge "Semua Tingkat" atau daftar tingkat spesifik) + kolom status aktif PER TINGKAT (misal 3 badge kecil X/XI/XII, badge terisi kalau opsi ini yang aktif untuk tingkat itu) + tombol "Buat Opsi Baru"/"Edit".
- **Aksi "Aktifkan" REVISI**: karena aktivasi sekarang per-tingkat, tombol Aktifkan pada 1 opsi perlu tanya/pilih **untuk tingkat mana** diaktifkan (kalau opsi itu `semuaTingkat: true`, tampilkan pilihan "Aktifkan untuk: [checklist X, XI, XII]"; kalau opsi scope-nya sudah spesifik, otomatis hanya tawarkan tingkat yang dicakup). Alternatif UI yang lebih simpel: tampilkan MATRIX 3 kolom (X/XI/XII) × baris opsi, tiap sel jadi tombol radio "aktifkan opsi ini untuk tingkat ini" — PUTUSKAN salah satu pendekatan saat implementasi, REKOMENDASI matrix karena lebih jelas menunjukkan constraint "1 aktif per tingkat" secara visual.
- Form create/edit opsi TAMBAH section cakupan tingkat: toggle "Berlaku untuk Semua Tingkat" (default ON) — kalau di-OFF, muncul checklist tingkat (X/XI/XII, bisa pilih lebih dari 1).
- **Form create/edit opsi** — UI KOMPLEKS (banyak baris per hari) — REKOMENDASI: tabel/grid per hari (Senin-Kamis 1 tabel, Jumat 1 tabel terpisah — KARENA biasanya durasi beda, TAPI biarkan admin BEBAS input per-hari secara independen, JANGAN asumsi hardcode "Senin-Kamis harus sama", sistem sebaiknya generic: admin bisa isi hari APAPUN dengan jam-ke berapa pun, bukan dibatasi cuma pola Excel contoh) — tiap baris: Jam Ke (angka, atau kosong+centang "ini istirahat"), Jam Mulai, Jam Selesai, Keterangan (opsional, misal nama istirahat).
- Tombol "Tambah Baris"/"Hapus Baris" per hari, tombol "Salin dari Hari Lain" (opsional, mempercepat input kalau beberapa hari punya pola sama — PERTIMBANGKAN, TIDAK WAJIB v1).

## Edge Cases
- Opsi jam pelajaran BARU dibuat tapi BELUM diaktifkan untuk tingkat manapun — jadwal existing TETAP pakai opsi LAMA yang masih aktif di tingkat masing-masing (regresi nol sampai admin eksplisit aktifkan yang baru).
- TIDAK ADA opsi jam pelajaran aktif untuk SATU tingkat tertentu (kondisi awal sistem baru, ATAU admin sengaja `deactivate()` tanpa gantikan) — resolusi jam-ke-ke-waktu (Opsi A) HARUS punya fallback AMAN untuk tingkat itu (return null/tidak bisa resolve, JANGAN crash) — konsisten filosofi T144/T147 (kalau syarat data tidak terpenuhi, gagal AMAN bukan salah diam-diam). Tingkat LAIN yang punya aktivasi tetap resolve normal — kegagalan 1 tingkat TIDAK BOLEH mempengaruhi tingkat lain.
- Hari yang TIDAK PUNYA slot sama sekali di opsi aktif (misal Sabtu, atau lupa diisi) — resolusi jam-ke-ke-waktu untuk hari itu return null/tidak ditemukan, TIDAK crash.
- Opsi `semuaTingkat: true` yang SEDANG AKTIF untuk X dan XI (tapi TIDAK untuk XII, karena XII pakai opsi lain) — DIUBAH jadi `semuaTingkat: false` dengan cakupan cuma XII — TOLAK di `update()` (lihat poin 2) karena akan menghapus cakupan tingkat X/XI yang SEDANG dipakai aktivasi existing.

## Files
- **Buat:** migration Prisma baru (3 model: `JamPelajaranOption`, `JamPelajaranOptionTingkat`, `JamPelajaranAktivasi`), modul `apps/api/src/jam-pelajaran/` (controller+service+dto), halaman admin baru `apps/web/src/app/(admin)/jam-pelajaran/`.
- **Modifikasi (TERGANTUNG Opsi A/B poin 3):** `apps/api/prisma/schema.prisma` (`Schedule`, KALAU Opsi A dipilih — ganti `jamMulai`/`jamSelesai` jadi `jamKeAwal`/`jamKeAkhir` KHUSUS `type: jam_mengajar`), SEMUA consumer `Schedule.jamMulai`/`jamSelesai` untuk jadwal mengajar (AUDIT MENYELURUH dulu, daftar semua titik pemakaian sebelum ubah).
- **Jangan sentuh:** `Schedule` untuk `type: jam_sekolah` (T145, jam masuk sekolah 3-lapis — DOMAIN BERBEDA, TIDAK terkait konsep opsi jam pelajaran ini SAMA SEKALI, field `jamMulai`/`jamSelesai` di situ TETAP string biasa apa adanya).

## Acceptance Criteria
- [x] Admin bisa membuat opsi jam pelajaran baru dengan slot lengkap per hari (termasuk baris istirahat) + cakupan tingkat (Semua/spesifik).
- [x] Admin bisa mengaktifkan 1 opsi UNTUK 1 TINGKAT TERTENTU tanpa mempengaruhi aktivasi tingkat lain (constraint `JamPelajaranAktivasi.tingkat @unique` per tingkat, `activate()` hanya upsert baris tingkat yang diminta).
- [x] Opsi yang cakupannya TIDAK mencakup tingkat tertentu DITOLAK saat coba diaktifkan untuk tingkat itu, pesan jelas (`JamPelajaranService.activate()`).
- [x] **Opsi A dipilih** (dikonfirmasi user 2026-08-14): jadwal mengajar existing OTOMATIS mengikuti jam sungguhan dari opsi yang AKTIF UNTUK TINGKAT KELAS itu — `Schedule.jamKeAwal`/`jamKeAkhir` (angka) menggantikan `jamMulai`/`jamSelesai` string utk `type: jam_mengajar`, di-resolve on-the-fly lewat `JamPelajaranService.resolveJamSchedule()` di semua consumer (lihat Files di bawah).
- [x] Tidak ada opsi aktif untuk 1 tingkat = sistem tidak crash untuk tingkat itu (fallback null, ditangani per-consumer — guru tetap "hadir" bukan error, sesi tidak di-auto-close, dst), tingkat lain tetap normal.
- [x] `@LogActivity`/`activityLog.record()` manual terpasang di semua endpoint mutasi (create/update/activate/deactivate/delete) — manual karena endpoint di-key oleh tingkat, bukan id baris tunggal (pola sama T145 jam-masuk).
- [x] Build + type-check `apps/api` dan `apps/web` hijau (`tsc --noEmit` bersih kedua app, `nest build` + `next build` sukses).

## Validasi Claudian
- [x] **Opsi A vs B diklarifikasi eksplisit ke user sebelum coding** (2026-08-14) — user konfirmasi ingin "yang diaktifkan langsung dipakai aplikasi", jadi Opsi A. Untuk cara resolve (helper terpusat vs Prisma middleware), user minta rekomendasi terbaik → dipilih helper terpusat (`JamPelajaranService.resolveJamSchedule()`) dengan cache in-memory per-tingkat (`getActiveForTingkat()`), bukan Prisma middleware implisit (risiko N+1, sulit didebug).
- [x] **Audit menyeluruh consumer `Schedule.jamMulai`/`jamSelesai` (type jam_mengajar)** dilakukan via subagent Explore SEBELUM ubah skema — titik yang diubah: `teaching-sessions.service.ts` (getMyToday, getSesiUntukTanggal, startSession, autoCloseDueSessions, flagPermitsNeedingFollowUp + 2 helper privat), `schedule-resolver.service.ts` (orderBy + tipe return `ScheduleWithKelasTingkat`), `attendance.service.ts` (buildAcceptedResponse kartu personal kiosk guru T089, determineStatus T175 patokan telat guru — deadline = MINIMUM di antara semua jadwal yang berhasil di-resolve, bukan cuma elemen pertama), `tv-piket.service.ts` (hitungGuruBelumMulai), `core/schedules/schedules.service.ts` (create/update/copyFromSemester/ensureNoBentrok — bentrok sekarang resolve jam per-kandidat lewat tingkat kelas masing-masing, karena 1 guru bisa mengajar di kelas tingkat berbeda dalam 1 hari).
- [x] Resolusi SELALU tahu tingkat kelas dulu sebelum lookup opsi aktif — tidak ada asumsi 1 opsi global di titik manapun; consumer yang tidak tahu tingkat (mis. tidak ada kelas) di-skip dengan fallback aman.
- [x] **JANGAN sentuh `Schedule` type: jam_sekolah** — dipatuhi: `schedules.service.ts` bagian `resolveJamMasuk`/`getJamMasukDefault`/`getJamMasukTingkat`/`updateJamMasukDefault/Tingkat/Kelas`/`deleteJamMasukKelasOverride`/`upsertJamMasukHariList` TIDAK disentuh sama sekali, `(admin)/jadwal/jadwal-view.tsx` (Jam Masuk Sekolah T145) TIDAK disentuh.
- [x] Constraint "1 aktif PER TINGKAT" dijaga (`JamPelajaranAktivasi.tingkat @unique`), tidak direfaktor jadi global.
- [x] **Ditemukan saat implementasi**: task ini dikerjakan PARALEL dengan sesi lain yang mengerjakan T159 (`Kelas.modeJadwal`) di working tree yang sama — dikoordinasikan dengan user, T159 selesai duluan, T158 dibangun DI ATAS fondasi T159 (`schedules.service.ts` sudah pakai `kelas.modeJadwal` utk resolusi minggu A/B, T158 menambah resolusi jam tanpa mengubah logic minggu itu).
- [x] Test spec disesuaikan: `schedules.service.spec.ts` (33 test), `teaching-sessions.service.spec.ts` (36 test, +1 baru "jadwal tidak bisa di-resolve"), `attendance.service.spec.ts` (19 test, +1 baru "fallback hadir tidak crash") — semua constructor tambah mock `JamPelajaranService`, fixture `jamMulai`/`jamSelesai` diganti `jamKeAwal`/`jamKeAkhir` + `kelas.tingkat`. Full suite 337/337 lulus (naik dari 249 baseline sebelum sesi ini, sebagian kenaikan dari task T159 paralel).
- [x] Frontend: halaman baru `(admin)/jam-pelajaran/` (list + Sheet form grid per-hari dgn Tabs, matrix badge aktivasi per tingkat X/XI/XII — field slot dibuat cocok 1:1 dgn kolom sheet Excel referensi sekolah "Data Alokasi" utk kemudahan import T160 nanti). `JadwalFormModal` (admin-jurnal, create/edit Schedule jam_mengajar) diubah dari input `type="time"` jadi dropdown "Jam Ke Awal/Akhir" bersumber dari opsi aktif utk tingkat kelas terpilih. Menu sidebar baru "Jam Pelajaran" di grup Jurnal Mengajar.
- [ ] **BELUM diverifikasi live via browser/curl** — dev API (`apps/api` port 3101) sedang dijalankan sesi lain (`nest start --watch`) dan tidak listen saat sesi ini selesai (kemungkinan reload/crash pasca perubahan schema bersama); sesi ini SENGAJA tidak me-restart proses itu (bukan milik sesi ini, hindari gangguan kerja paralel sesuai instruksi user). Verifikasi mengandalkan `tsc --noEmit` + `nest build`/`next build` + jest unit test (semua hijau) — **user perlu cek live manual setelah dev server sesi lain stabil kembali**.

## Follow-up WAJIB Setelah T158 Selesai — Isi `waktuPulang` Otomatis di T180 (2026-08-14)

**Catatan silang dari task terpisah [[T180-bulk-sudah-pulang-tidak-absen-pulang]]**: tombol bulk "Sudah Pulang Semua" di section piket "Tidak Absen Pulang" SEMENTARA mengisi `waktuPulang` dengan `null` ("tidak diketahui") karena SAAT T180 ditulis, sistem belum bisa tahu "jam terakhir pelajaran hari itu" untuk seorang siswa (butuh resolusi jam pelajaran per kelas per hari, yaitu FITUR INI).

**SETELAH T158 selesai** (dan idealnya T159/T160 juga, supaya resolusi jadwal per kelas lengkap) — SEBAGAI TASK TERPISAH (bukan otomatis bagian dari T158, TAPI WAJIB DIKERJAKAN SEGERA SETELAHNYA, jangan lupa/hilang): perbarui endpoint `POST /attendance/konfirmasi-pulang-retroaktif-bulk` (dari T180) supaya `waktuPulang` masing-masing siswa di-resolve OTOMATIS dari **jam selesai jadwal mengajar TERAKHIR hari itu** untuk kelas siswa tersebut (query `Schedule`/`ScheduleResolverService` yang sesuai, ambil `jamSelesai` dari slot paling akhir hari itu, KONSISTEN cara resolusi jam yang dibangun task ini). Kalau kelas siswa itu TIDAK PUNYA jadwal hari itu (misal libur/tidak ada jadwal), TETAP fallback ke `null` seperti perilaku T180 lama (JANGAN paksa isi angka yang tidak berdasar).
