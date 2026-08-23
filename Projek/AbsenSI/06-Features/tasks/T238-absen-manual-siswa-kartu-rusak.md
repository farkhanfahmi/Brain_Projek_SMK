# T238 — API+Web: Absen Manual Siswa (Kartu Rusak/Belum Tercetak) dari Riwayat Catatan

## Depends on
Tidak ada dependency teknis. Independen, murni modul `attendance`/frontend `RiwayatCatatanTable`.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-21)

**Komponen `RiwayatCatatanTable`** (`apps/web/src/components/riwayat-catatan-table.tsx`, komponen SHARED dipakai halaman detail siswa admin, wali kelas, dan piket — T224d) — kolom SAAT INI: Tanggal, Jenis, Detail, Petugas — **TIDAK ADA kolom aksi/tombol apa pun per baris**.

**Endpoint manual override existing TIDAK CUKUP untuk kasus ini**:
- `manualPulang()` (`attendance.service.ts:860-867`) — HANYA update `waktuPulang` pada `AttendanceRecord` yang **SUDAH ADA** hari itu — TIDAK BISA dipakai kalau siswa belum tap sama sekali (kartu rusak = tidak ada `AttendanceRecord` hari itu).
- `konfirmasiPulangRetroaktif()`/`konfirmasiPulangRetroaktifBulk()` (`attendance.service.ts:775-853`) — scope KEMARIN saja, SAMA-SAMA butuh `AttendanceRecord` sudah ada.
- **TIDAK ADA endpoint yang bisa CREATE `AttendanceRecord` baru dari NOL** untuk siswa yang belum tap sama sekali — SEMUA endpoint manual yang ada mengasumsikan record sudah ada dari tap fisik. Task ini MEMBANGUN endpoint baru untuk kasus ini.

**`toleransiSiswaMenit`** (`ScheduleConfig`, default 10) — dibaca `ScheduleConfigService.get()`, dipakai `determineStatus()` (`attendance.service.ts:1000-1024`) untuk resolusi status hadir/terlambat — REUSE logic yang SAMA untuk task ini (waktu masuk manual harus "sebelum toleransi" supaya status = hadir, bukan terlambat).

**Jam pelajaran terakhir PER KELAS di hari itu** — TIDAK ADA method siap pakai (beda dari T237 yang per-GURU) — perlu dibangun: query `JadwalSlot` kelas itu di hari tsb, `orderBy: {jamKe: "desc"}`, resolve wall-clock via `AlokasiWaktuSlot` (pola sama seperti T237 poin 5, TAPI scope KELAS bukan GURU — VERIFIKASI SAAT IMPLEMENTASI apakah bisa share 1 helper generik "jam akhir jadwal di hari X" yang terima parameter kelasId ATAU teacherId, supaya T237 dan T238 tidak duplikasi logic resolusi serupa).

## Keputusan Dikonfirmasi User (2026-08-21)

1. **Lokasi tombol**: DI SETIAP BARIS tabel `RiwayatCatatanTable` (screenshot user tunjukkan tombol langsung di baris, PALING RELEVAN untuk baris berjenis "Alfa" — supaya admin bisa langsung perbaiki dari baris itu, TAPI VERIFIKASI SAAT IMPLEMENTASI apakah tombol INI HANYA muncul untuk baris "Alfa", atau semua jenis baris — REKOMENDASI: HANYA baris "Alfa" yang relevan diberi tombol "Hadir" ini, jenis lain seperti "izin"/"terlambat" TIDAK PERLU tombol serupa karena sudah ada datanya).
2. **Cakupan tanggal**: BISA untuk tanggal APAPUN (termasuk yang sudah lewat/alfa lama) — BUKAN cuma hari ini. Karena baris tabel Riwayat Catatan SUDAH punya tanggal spesifik per baris, tombol di baris itu OTOMATIS tahu tanggal target (tidak perlu date-picker tambahan di dialog).
3. **Waktu masuk OTOMATIS** = SEBELUM batas waktu toleransi keterlambatan (`ScheduleConfig.toleransiSiswaMenit`) — supaya status TERHITUNG "hadir" normal, BUKAN "terlambat".
4. **Waktu pulang OTOMATIS** = jam akhir pelajaran KELAS siswa itu PADA TANGGAL YANG DIPERBAIKI (bukan hari ini).
5. **Endpoint dibatasi admin** (super_admin/card_admin — role yang sudah punya akses ke halaman detail siswa dan endpoint riwayat catatan, KONSISTEN `@Roles()` existing `riwayat-catatan` endpoint).

## Spec Detail

### 1. Backend — endpoint absen manual baru (CREATE dari nol)

`POST /attendance/students/:id/absen-manual` — body `{ tanggal: string }` (format `YYYY-MM-DD`, WAJIB — task ini TIDAK terima tanggal implisit "hari ini", SELALU eksplisit dari baris riwayat yang diklik).

Logic:
1. **VALIDASI**: siswa dengan `id` ada dan `status: aktif` (siswa nonaktif TIDAK bisa diabsenkan manual, tidak masuk akal).
2. **VALIDASI**: tanggal yang diminta adalah HARI WAJIB (`resolveHariWajib` — bukan weekend/libur) — TOLAK dengan pesan jelas kalau bukan hari wajib ("Tanggal [X] bukan hari sekolah, tidak bisa diabsenkan").
3. **VALIDASI**: `AttendanceRecord` untuk siswa+tanggal itu BELUM ADA (kalau SUDAH ADA — entah karena sudah tap fisik atau sudah pernah diabsen manual sebelumnya — TOLAK dengan pesan jelas "Siswa sudah punya catatan kehadiran di tanggal ini, gunakan fitur edit/konfirmasi pulang yang sudah ada, bukan absen manual baru").
4. Resolve `jamMasuk` kelas siswa itu (`SchedulesService.resolveJamMasuk`, REUSE method existing) — `waktuMasukOtomatis = jamMasuk - beberapa menit` (SEBELUM deadline toleransi, BUKAN pas di batas — REKOMENDASI: `jamMasuk.jamMulai` PERSIS, dianggap "tepat waktu", KONSISTEN definisi "hadir" yang sudah ada di `determineStatus()` — VERIFIKASI SAAT IMPLEMENTASI logic `determineStatus()` PERSIS supaya waktu yang di-input otomatis benar-benar menghasilkan status "hadir" bukan "terlambat" kalau dilewatkan lagi ke fungsi yang sama).
5. Resolve jam akhir pelajaran KELAS siswa itu PADA TANGGAL yang diminta (method BARU, lihat Konteks) — `waktuPulangOtomatis`.
6. `AttendanceRecord.create()` — `studentId`, `tanggal`, `waktuMasuk: waktuMasukOtomatis`, `waktuPulang: waktuPulangOtomatis`, `status: "hadir"` (LANGSUNG hadir, tidak lewat `determineStatus()` lagi kalau sudah dipastikan waktu masuk sebelum deadline — VERIFIKASI SAAT IMPLEMENTASI apakah lebih aman tetap panggil `determineStatus()` untuk konsistensi single-source-of-truth logic status, daripada hardcode "hadir"), `masukVia`/`pulangVia`: ENUM BARU atau REUSE existing yang menandakan "manual oleh admin" (KONSISTEN pola `PulangVia`/`MasukVia` yang sudah ada — CEK enum ini, kemungkinan perlu tambah varian `manual` kalau belum ada).
7. `academicYearId`/`semesterId` — WAJIB TAG (aturan CLAUDE.md, resolve dari `AcademicYear.isActive`/`Semester.isActive` SAAT TANGGAL itu — VERIFIKASI kalau tanggal yang diperbaiki dari SEMESTER LAMA yang sudah tidak aktif, ambil tahun-ajaran/semester yang BENAR mencakup tanggal itu, BUKAN yang aktif sekarang).
8. **Log aktivitas** — `ActivityLogService.record()` atau `@LogActivity` (endpoint mutasi baru, WAJIB sesuai aturan CLAUDE.md), `snapshotAfter` catat data yang diinput (waktu masuk/pulang, tanggal, alasan "kartu rusak" kalau ada field catatan — VERIFIKASI SAAT IMPLEMENTASI apakah perlu field `catatan`/`alasan` opsional di body request, untuk dokumentasi kenapa diabsenkan manual — REKOMENDASI: TAMBAH field opsional `catatan?: string` di body, disimpan di suatu tempat yang bisa ditelusuri nanti, MISAL di `ActivityLog.snapshotAfter` cukup, tidak perlu kolom baru di `AttendanceRecord`).

### 2. Backend — method resolusi "jam akhir jadwal per kelas per tanggal" (BARU)

- Method baru di service yang tepat (`SchedulesService`, atau `JadwalSlotService` — VERIFIKASI SAAT IMPLEMENTASI lokasi paling pas) — terima `kelasId`+`tanggal`, resolve `hari` dari tanggal, query `JadwalSlot` kelas itu hari tsb (via `OpsiJadwal` yang AKTIF pada tanggal tersebut — bukan cuma `isActive` sekarang, tapi `isActive` PADA SAAT tanggal yang diminta kalau beda dari sekarang — VERIFIKASI SAAT IMPLEMENTASI kompleksitas ini, kemungkinan CUKUP pakai OpsiJadwal aktif SEKARANG untuk kasus umum "tanggal baru-baru ini", tapi CATAT keterbatasan kalau tanggal jauh di masa lalu dengan Opsi Jadwal yang sudah berbeda), ambil `jamKe` TERTINGGI, resolve wall-clock via `AlokasiWaktuSlot`.
- **KEMUNGKINAN SHARE dengan T237** (resolusi serupa untuk GURU) — VERIFIKASI SAAT IMPLEMENTASI apakah bisa dibuat 1 helper generik yang terima `{kelasId}` ATAU `{teacherId}`, atau tetap 2 method terpisah kalau logic query-nya cukup beda (JadwalSlot filter by kelasId langsung vs filter by teacherId via JadwalSlotGuru) — JANGAN paksa share kalau membuat kode lebih rumit dari 2 method sederhana terpisah.

### 3. Frontend — tombol "Hadir" di baris Alfa, `RiwayatCatatanTable`

- `apps/web/src/components/riwayat-catatan-table.tsx` — TAMBAH kolom aksi BARU (atau tombol inline di kolom Detail/kolom baru) — HANYA muncul untuk baris `jenis === "alfa"`.
- Klik tombol → Dialog konfirmasi (KONSISTEN pola konfirmasi aksi sensitif proyek) — tampilkan tanggal yang akan diperbaiki, preview waktu masuk/pulang OTOMATIS yang akan diisi (fetch dulu atau hitung di FE dari data yang sudah ada — VERIFIKASI SAAT IMPLEMENTASI apakah preview perlu call API terpisah atau backend langsung proses saat submit tanpa preview terpisah, REKOMENDASI: submit langsung dengan konfirmasi dialog sederhana "Absenkan [nama] sebagai Hadir pada [tanggal]? Waktu akan diisi otomatis sesuai jam masuk sekolah dan jam pelajaran terakhir." — TANPA perlu preview API terpisah, backend yang hitung).
- Setelah berhasil — **REFRESH tabel Riwayat Catatan** (baris "Alfa" itu HARUS BERUBAH jadi tidak muncul lagi, karena `riwayatCatatan()` method (T220 dkk) menghitung Alfa dari SELISIH `wajibDates` vs data hadir/izin — begitu ada `AttendanceRecord` baru, alfa untuk tanggal itu OTOMATIS hilang dari hasil berikutnya, TIDAK PERLU logic khusus tambahan di `riwayatCatatan()` itu sendiri, method itu SUDAH benar akan reflect perubahan begitu data baru ada).

## Edge Cases

- **Siswa sudah pernah diabsenkan manual untuk tanggal yang SAMA** (klik tombol 2x tidak sengaja) — endpoint TOLAK (poin 1.3 spec), pesan jelas.
- **Tanggal yang diperbaiki sudah SANGAT LAMA** (misal alfa dari bulan lalu) — TETAP DIIZINKAN (user eksplisit "bisa pilih tanggal apa saja") — TIDAK ADA batas mundur waktu di spec ini.
- **Kelas siswa TIDAK PUNYA `JadwalSlot` di hari tsb sama sekali** (misal kelas itu libur hari itu meski hari wajib secara umum, atau data jadwal belum lengkap) — method resolusi jam pulang (poin 2) HARUS punya fallback jelas (misal pesan error actionable "Tidak ada jadwal pelajaran untuk kelas ini di tanggal tsb, tidak bisa hitung jam pulang otomatis — hubungi admin jadwal" — JANGAN crash/silent null).
- **Siswa PKL saat tanggal tersebut** (`StudentPkl` aktif) — VERIFIKASI SAAT IMPLEMENTASI apakah siswa PKL BOLEH diabsenkan manual dengan cara ini (kemungkinan TIDAK relevan — siswa PKL tidak masuk hitungan alfa sama sekali karena logic existing sudah exclude PKL, jadi baris "Alfa" untuk siswa PKL SEHARUSNYA tidak pernah muncul — kalau muncul juga itu bug terpisah, DI LUAR SCOPE task ini).

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (endpoint+logic baru), `apps/api/src/attendance/attendance.controller.ts` (endpoint baru), `apps/web/src/components/riwayat-catatan-table.tsx` (tombol+dialog).
- **Buat:** method resolusi "jam akhir jadwal per kelas per tanggal" (baru, kemungkinan share sebagian dengan T237).
- **Jangan sentuh:** `manualPulang()`/`konfirmasiPulangRetroaktif()` existing (endpoint LAMA tetap ada untuk kasus BEDA — record sudah ada tapi cuma pulang yang kurang — task ini endpoint BARU untuk kasus record belum ada sama sekali).

## Acceptance Criteria
- [ ] Baris "Alfa" di Riwayat Catatan — tombol "Hadir" tampil, klik → dialog konfirmasi → submit berhasil.
- [ ] Setelah absen manual — `AttendanceRecord` baru tercipta dengan `waktuMasuk` SEBELUM deadline toleransi (status hadir, bukan terlambat), `waktuPulang` = jam akhir pelajaran kelas hari itu.
- [ ] Baris "Alfa" yang sudah diperbaiki — TIDAK MUNCUL LAGI di refresh Riwayat Catatan berikutnya (otomatis via logic `riwayatCatatan()` existing).
- [ ] Tanggal yang SUDAH ADA `AttendanceRecord` (bukan alfa murni) — endpoint TOLAK, pesan jelas.
- [ ] Siswa nonaktif — TIDAK bisa diabsenkan manual.
- [ ] Bukan hari wajib (weekend/libur) — TOLAK, pesan jelas.
- [ ] Log aktivitas tercatat untuk tiap absen manual (siapa admin, siswa mana, tanggal berapa).
- [ ] Build + type-check hijau, jest baru: happy path, record sudah ada (ditolak), siswa nonaktif (ditolak), bukan hari wajib (ditolak), tidak ada JadwalSlot kelas (pesan jelas bukan crash).

## Validasi Claudian
- [ ] Konfirmasi endpoint BARU (CREATE dari nol), BUKAN modifikasi `manualPulang()`/`konfirmasiPulangRetroaktif()` existing yang scope-nya beda (butuh record sudah ada).
- [ ] Konfirmasi `academicYearId`/`semesterId` di-tag BENAR sesuai tanggal yang diperbaiki (bukan selalu period aktif sekarang, terutama untuk tanggal lama).
- [ ] Konfirmasi status hasil SELALU "hadir" (bukan "terlambat") — waktu masuk otomatis SELALU sebelum deadline toleransi, diverifikasi lewat `determineStatus()`/logic yang sama dipakai tap normal.
- [ ] Konfirmasi tombol "Hadir" HANYA muncul untuk baris jenis "alfa" (bukan izin/sakit/terlambat/dst).
