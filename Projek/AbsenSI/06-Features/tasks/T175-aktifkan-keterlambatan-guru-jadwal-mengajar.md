# T175 — API: Aktifkan Logic Keterlambatan Guru Berdasarkan Jadwal Mengajar

## Depends on
Tidak ada dependency teknis (jadwal mengajar `Schedule`/`TeachingSession` SUDAH ADA). **WAJIB dikerjakan SEBELUM T176** (Rekap Kehadiran Guru) — tanpa task ini, kolom "Terlambat" di rekap guru akan selalu 0 untuk semua guru, membuat fitur itu tampak rusak sejak awal.

## Objective
Guru yang tap gerbang SETELAH jam mulai jadwal mengajar pertamanya hari itu (+ toleransi) → status `terlambat` (bukan selalu `hadir` seperti sekarang). Guru tanpa jadwal mengajar hari itu TETAP `hadir` (tidak ada patokan jam untuk dibandingkan).

## Context — Temuan Riset (2026-08-14)

`determineStatus()` di `apps/api/src/attendance/attendance.service.ts:769-792` punya percabangan eksplisit:
- Kalau `card.studentId` truthy → resolve jadwal via `schedulesService.resolveJamMasuk(kelasId, dayOfWeek)`, bandingkan `scannedAt` vs deadline (jam masuk + toleransi) → `terlambat`/`hadir`.
- Kalau kartu GURU (`card.studentId` falsy) → **langsung `return AttendanceStatus.hadir` tanpa pengecekan apa pun** (baris 791). Komentar developer eksplisit (baris 787-790): "jadwal mengajar guru belum diinput user, jadi SEMENTARA guru tidak pernah dianggap terlambat... JANGAN pakai jam_sekolah sebagai patokan pengganti."

Prasyarat komentar itu ("jadwal mengajar guru belum diinput") **SUDAH TERPENUHI** — `Schedule`/`TeachingSession` sudah berfungsi penuh (dipakai jurnal mengajar, dst). User eksplisit konfirmasi (2026-08-14): aktifkan logic ini sekarang.

**PERINGATAN EKSPLISIT dari komentar lama**: "JANGAN pakai jam_sekolah sebagai patokan pengganti" — artinya JANGAN reuse `Schedule type: jam_sekolah` (jam masuk sekolah 3-lapis siswa, T145) sebagai jam patokan guru. Patokan HARUS jadwal MENGAJAR guru itu sendiri (`Schedule type: jam_mengajar` milik `teacherId` itu, ATAU `TeachingSession` miliknya hari itu).

## Spec Detail

### 1. Backend — logic baru di `determineStatus()`

- Ganti cabang `else` (guru) di `attendance.service.ts:786-791` — SEBELUM langsung `return hadir`, cari jadwal mengajar PERTAMA milik `teacherId` itu di hari (`dayOfWeek`) tap ini terjadi:
  - Query `Schedule` (type `jam_mengajar`) milik `teacherId` = kartu ini, `hari` = hari tap ini, `orderBy: jamMulai asc`, ambil yang PERTAMA (kalau ada beberapa jadwal hari itu, jam paling pagi jadi patokan telat).
  - **VERIFIKASI dulu**: apakah resolusi jam mulai jadwal mengajar guru butuh lookup ke `JamPelajaranOption` aktif (T158, KALAU sudah dikerjakan duluan dan Opsi A dipilih — `Schedule` simpan `jamKeAwal` bukan `jamMulai` string) ATAU masih `Schedule.jamMulai` string langsung (kalau T158 belum/Opsi B) — SESUAIKAN dengan kondisi kode SAAT task ini dieksekusi, JANGAN asumsi salah satu tanpa cek dulu.
  - Kalau TIDAK ADA jadwal mengajar guru itu di hari tersebut → `return AttendanceStatus.hadir` (PERILAKU LAMA dipertahankan untuk kasus ini — guru tanpa jadwal hari itu tidak punya patokan telat, WAJAR dianggap hadir biasa, BUKAN alfa/masalah).
  - Kalau ADA jadwal → bandingkan `scannedAt` vs (jam mulai jadwal + toleransi). **Toleransi guru dari mana?** — VERIFIKASI apakah ada toleransi keterlambatan KHUSUS guru (beda dari toleransi siswa `(admin-jurnal)/admin-jurnal/toleransi/`) — KEMUNGKINAN BESAR belum ada, REKOMENDASI reuse config toleransi yang sama (1 nilai global untuk semua, siswa maupun guru) KECUALI ditemukan sudah ada pemisahan eksplisit di kode. TANYAKAN ke user kalau ragu, JANGAN putuskan sepihak nilai toleransi guru berbeda dari siswa tanpa konfirmasi.
  - Lewat deadline → `terlambat`; sebelum/pas → `hadir`.

### 2. Lock 2x-terlambat (T153/T154) — TETAP TIDAK BERLAKU untuk guru

- `applyLateStrikeLock` (dipicu di `tap()` sekitar baris 251) SAAT INI eksplisit dibatasi `status === terlambat && card.studentId` — task ini **TIDAK MENGUBAH** batasan ini. Guru yang sekarang BISA berstatus `terlambat` TETAP TIDAK memicu lock otomatis (fitur lock itu murni siswa, di luar scope task ini, TIDAK ADA instruksi user untuk memperluasnya ke guru).

### 3. Method lain yang punya logic serupa — VERIFIKASI konsistensi

- Riset menyebut ada method lain (`myHistory` sekitar baris 424) dengan komentar "PERSIS SAMA dengan determineStatus()" — BACA method itu, PASTIKAN kalau memang duplikasi logic untuk guru, method itu JUGA diperbarui konsisten (supaya riwayat guru sendiri `/guru/riwayat` dan rekap admin tidak menunjukkan status berbeda untuk kejadian tap yang sama).

## Edge Cases
- Guru dengan BEBERAPA jadwal mengajar di hari yang sama (misal 3 kelas berbeda) — patokan telat pakai jadwal PALING PAGI (jam pertama hari itu), BUKAN rata-rata/jadwal terakhir.
- Guru piket yang punya jadwal piket (`PiketSchedule`, modul terpisah) TAPI TIDAK ADA jadwal mengajar (`Schedule` type jam_mengajar) hari itu — DI LUAR SCOPE task ini (task ini murni soal jadwal MENGAJAR), guru piket murni tanpa jadwal mengajar tetap `hadir` tanpa patokan telat (PERILAKU SAMA seperti guru lain tanpa jadwal).
- Perubahan ini RETROAKTIF secara logic (kode baru berlaku untuk tap BARU sejak deploy) — data `AttendanceRecord` LAMA (guru yang sudah tercatat `hadir` padahal sebenarnya telat menurut logic baru) TIDAK di-backfill/diubah — task ini HANYA mengubah perilaku KE DEPAN, TIDAK mengoreksi histori.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`determineStatus()`, dan `myHistory`/method serupa kalau terkonfirmasi duplikasi logic).
- **Jangan sentuh:** logic keterlambatan siswa (cabang `if(studentId)`, TIDAK diubah), `applyLateStrikeLock` (batasan `card.studentId` TETAP, TIDAK diperluas ke guru).

## Acceptance Criteria
- [x] Guru dengan jadwal mengajar hari itu, tap SETELAH jam mulai+toleransi → status `terlambat`.
- [x] Guru dengan jadwal mengajar hari itu, tap SEBELUM/PAS jam mulai → status `hadir`.
- [x] Guru TANPA jadwal mengajar hari itu → tetap `hadir` (perilaku lama dipertahankan untuk kasus ini).
- [x] Guru dengan beberapa jadwal hari itu → patokan jadwal PALING PAGI.
- [x] Lock 2x-terlambat TIDAK terpicu untuk guru (verified, batasan `studentId` masih ada — bahkan `Teacher` model TIDAK PUNYA kolom `lockedAt` sama sekali, secara struktural mustahil).
- [x] Method lain dengan logic duplikat (`myHistory`) konsisten dengan `determineStatus()`.
- [x] Build + type-check hijau, jest existing tetap pass, jest baru untuk skenario guru terlambat/tidak ditambahkan.

## Validasi Claudian
- [x] Konfirmasi sumber toleransi keterlambatan guru — DITEMUKAN: siswa TIDAK PAKAI `schedule_config.toleransiTerlambatMenit` sama sekali di `determineStatus()` (field itu cuma dipakai display, deadline siswa = `jamMulai` MENTAH tanpa toleransi tambahan). Guru dibuat KONSISTEN dengan pola siswa yang sudah ada: deadline = jam mulai jadwal mengajar MENTAH, tanpa toleransi tambahan — bukan menambah sumber toleransi baru yang malah tidak konsisten.
- [x] Konfirmasi resolusi jam mulai jadwal mengajar — DICEK schema.prisma: `Schedule.jamMulai` masih `String` (`HH:mm`) langsung, TIDAK ADA `jamKeAwal`/`JamPelajaranOption`. T158 BELUM dikerjakan (dikonfirmasi user saat eksekusi task ini, boleh lanjut tanpa T158).
- [x] Konfirmasi perubahan ini TIDAK retroaktif — implementasi murni logic baru di `determineStatus()`, tidak ada backfill/migrasi data.

## Status Eksekusi (2026-08-14)

**Selesai.**

- Ditemukan `ScheduleResolverService.getJadwalHariIni(teacherId, tanggal)` (`apps/api/src/schedule-resolver/schedule-resolver.service.ts`) SUDAH JADI 1 sumber kebenaran resolusi jadwal mengajar guru per tanggal (resolve semester aktif untuk tanggal itu, filter hari + minggu blok A/B kalau relevan, `orderBy jamMulai asc`) — dipakai `TeachingSessionsService`. REUSE langsung, tidak duplikasi logic resolusi jadwal.
- `apps/api/src/attendance/attendance.module.ts` — import `ScheduleResolverModule`.
- `apps/api/src/attendance/attendance.service.ts` — inject `ScheduleResolverService`; cabang guru di `determineStatus()` diganti dari `return hadir` statis menjadi: `getJadwalHariIni(teacherId, scannedAt)` → kosong → `hadir`; ada → ambil elemen pertama (paling pagi, sudah ter-sort) → `combineDateAndTime` vs `scannedAt` → `terlambat`/`hadir`, PERSIS pola siswa.
- `myHistory()` dicek — HANYA baca `record.status` yang sudah tersimpan (tidak re-derive status), TIDAK ADA duplikasi logic, TIDAK diubah. `piketBoard()` dicek — khusus siswa (`prisma.student.findMany`), tidak ada cabang guru, TIDAK diubah.
- `apps/api/src/attendance/attendance.service.spec.ts` — 5 test baru (`determineStatus` diakses bracket-notation, private): guru tanpa jadwal → hadir; guru terlambat; guru tepat waktu; guru multi-jadwal → patokan paling pagi; logic siswa tidak terpengaruh (`getJadwalHariIni` tidak dipanggil untuk cabang siswa).

**Verifikasi live** (dev DB port 3307, API dev port 3101, production tidak disentuh, kartu+jadwal test dibuat untuk 2 guru existing):
1. Guru dengan jadwal 07:00, tap 10:00 lokal → `status: terlambat`.
2. Guru sama, tap 06:30 lokal (sebelum jadwal) → `status: hadir`.
3. Guru tanpa jadwal mengajar sama sekali, tap 17:00 lokal → tetap `status: hadir` (tidak ada patokan).
4. Guru dengan 2 jadwal (09:00 dan 14:00), tap 12:30 lokal (lewat jadwal pertama, belum jadwal kedua) → `status: terlambat` — konfirmasi patokan jadwal PALING PAGI, bukan terakhir.
5. Semua tap di atas TIDAK memicu lock (`Teacher` model dikonfirmasi tidak punya kolom `lockedAt` di schema).
6. Data uji (2 kartu test, attendance_records, tap_events, 3 schedules jam_mengajar test, 1 semester test, `kiosks.allowed_ip` override) dibersihkan, dikonfirmasi via re-query.
7. `tsc --noEmit` bersih, `jest` — 22 suite / 282 test lulus 100%.
